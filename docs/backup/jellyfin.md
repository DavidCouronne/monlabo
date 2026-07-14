# Sauvegarde et Restauration Jellyfin (Podman)

Cette documentation présente les procédures de sauvegarde et de restauration de l'instance Jellyfin déployée en mode rootless avec Podman et des Quadlets Systemd.

---

## 1. Vue d'ensemble de la stratégie

La stratégie de sauvegarde de Jellyfin se divise en deux parties distinctes :

1. **Fichiers média (films, séries, musiques)** : Stockés séparément sur un NAS ou un disque dédié, gérés par leurs propres routines de sauvegarde.
2. **Configuration Jellyfin** : Contient la base de données SQLite (historique de lecture, utilisateurs, configurations), les métadonnées et les configurations de plugins.

### Architecture des données Jellyfin
Les données de Jellyfin sont persistées dans le répertoire personnel de l'utilisateur :
```
~/.local/share/jellyfin/config/
├── data/                    # Bases de données SQLite
│   ├── library.db          # Base principale des fichiers média
│   ├── jellyfin.db         # Configuration globale du serveur
│   └── authentication.db   # Comptes et mots de passe des utilisateurs
├── metadata/               # Images et posters téléchargés
├── plugins/               # Plugins installés
└── root/                  # Configuration du serveur
```

---

## 2. Configuration de la sauvegarde

Le script de sauvegarde s'exécute en mode utilisateur (rootless) et ne nécessite aucun privilège administrateur (`sudo`). Il est recommandé d'arrêter le service Jellyfin pendant l'archivage pour éviter les écritures concurrentes dans la base SQLite.

### Étape 2.1 : Création du script de sauvegarde

Créez le fichier de script `~/.local/bin/jellyfin-backup.sh` :

```bash
kate ~/.local/bin/jellyfin-backup.sh
```

Ajoutez-y le contenu suivant :

```bash title="~/.local/bin/jellyfin-backup.sh" linenums="1"
#!/usr/bin/env bash

# ==============================================================================
# Script de sauvegarde de la configuration Jellyfin (Podman Rootless)
# ==============================================================================

# Configuration
BACKUP_DIR="${HOME}/Documents/Sauvegardes/jellyfin"       # Dossier cible à synchroniser par Duplicati
JELLYFIN_CONFIG="${HOME}/.local/share/jellyfin/config"     # Chemin vers le dossier config
KEEP_BACKUPS=14                                           # Nombre de sauvegardes locales à conserver

# Couleurs pour les journaux
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERREUR:${NC} $1"
}

warning() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] ATTENTION:${NC} $1"
}

# Vérification de l'accessibilité de Podman
if ! podman ps &> /dev/null; then
    error "Podman n'est pas accessible ou ne fonctionne pas."
    exit 1
fi

# Vérification du dossier de configuration
if [ ! -d "${JELLYFIN_CONFIG}" ]; then
    error "Le dossier de configuration de Jellyfin est introuvable : ${JELLYFIN_CONFIG}"
    exit 1
fi

# Création du dossier de sauvegarde
mkdir -p "${BACKUP_DIR}"

# Définition du fichier d'archive
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/jellyfin_config_${DATE}.tar.gz"

log "===== Début de la sauvegarde Jellyfin ====="

# Arrêt de Jellyfin pour assurer la cohérence de la base de données
JELLYFIN_WAS_RUNNING=false
if systemctl --user is-active --quiet jellyfin.service; then
    log "Arrêt temporaire de Jellyfin..."
    systemctl --user stop jellyfin.service
    JELLYFIN_WAS_RUNNING=true
    sleep 3
    log "✓ Jellyfin arrêté."
else
    warning "Jellyfin n'était pas en cours d'exécution. Sauvegarde à froid."
fi

# Création de l'archive tar.gz
log "Compression du répertoire de configuration..."
if tar -czf "${BACKUP_FILE}" -C "$(dirname "${JELLYFIN_CONFIG}")" "$(basename "${JELLYFIN_CONFIG}")" 2>/dev/null; then
    
    # Vérification du fichier généré
    if [ -s "${BACKUP_FILE}" ]; then
        SIZE=$(du -h "${BACKUP_FILE}" | cut -f1)
        log "✓ Sauvegarde réussie : ${BACKUP_FILE}"
        log "  Taille : ${SIZE}"
        
        # Mise à jour du lien symbolique vers la dernière sauvegarde
        ln -sf "$(basename "${BACKUP_FILE}")" "${BACKUP_DIR}/jellyfin_config_latest.tar.gz"
        
        # Fichier d'informations
        cat > "${BACKUP_DIR}/backup_${DATE}_info.txt" << EOF
Date du backup: $(date)
Machine hôte: $(hostname)
Image Jellyfin: $(podman inspect jellyfin --format='{{.ImageName}}' 2>/dev/null || echo "N/A")
Chemin d'origine: ${JELLYFIN_CONFIG}
Fichier généré: $(basename "${BACKUP_FILE}")
Arrêt requis du service: ${JELLYFIN_WAS_RUNNING}
EOF
    else
        error "L'archive générée est vide !"
        rm -f "${BACKUP_FILE}"
        exit 1
    fi
else
    error "Échec lors de la création de l'archive de configuration."
    exit 1
fi

# Relance de Jellyfin s'il était actif
if [ "${JELLYFIN_WAS_RUNNING}" = true ]; then
    log "Redémarrage de Jellyfin..."
    systemctl --user start jellyfin.service
    sleep 3
    log "✓ Jellyfin redémarré."
fi

# Nettoyage des anciennes sauvegardes
log "Nettoyage des anciennes sauvegardes (conservation des ${KEEP_BACKUPS} dernières)..."
cd "${BACKUP_DIR}" || exit 1
ls -t jellyfin_config_*.tar.gz 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f
ls -t backup_*_info.txt 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f

log "✓ Nettoyage terminé."
log "===== Sauvegarde terminée avec succès ====="
log "Dossier prêt pour la synchronisation Duplicati : ${BACKUP_DIR}"
```

Rendez le script exécutable :

```bash
chmod +x ~/.local/bin/jellyfin-backup.sh
```

### Étape 2.2 : Automatisation via Systemd User Timers

Créez le service et son timer dans votre session utilisateur pour des sauvegardes quotidiennes automatiques.

#### Fichier Service (`~/.config/systemd/user/jellyfin-backup.service`) :

```ini title="~/.config/systemd/user/jellyfin-backup.service"
[Unit]
Description=Sauvegarde de la configuration Jellyfin (User)
After=jellyfin.service

[Service]
Type=oneshot
ExecStart=%h/.local/bin/jellyfin-backup.sh
StandardOutput=journal
StandardError=journal
```

#### Fichier Timer (`~/.config/systemd/user/jellyfin-backup.timer`) :

```ini title="~/.config/systemd/user/jellyfin-backup.timer"
[Unit]
Description=Déclencheur journalier de sauvegarde Jellyfin

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

#### Activation du Timer :

```bash
systemctl --user daemon-reload
systemctl --user enable --now jellyfin-backup.timer
```

Vérifiez le statut :

```bash
systemctl --user status jellyfin-backup.timer
```

---

## 3. Configuration des alias de commandes

=== "Fish Shell"

    Éditez votre configuration de Fish :
    ```bash
    kate ~/.config/fish/config.fish
    ```

    Ajoutez ces alias :
    ```fish
    # Raccourcis pour Jellyfin
    alias backup-jellyfin='~/.local/bin/jellyfin-backup.sh'
    alias restore-jellyfin='~/.local/bin/jellyfin-restore.sh'
    ```

    Rechargez :
    ```bash
    source ~/.config/fish/config.fish
    ```

=== "Bash / Zsh"

    Éditez votre `~/.bashrc` ou `~/.zshrc` :
    ```bash
    kate ~/.bashrc # ou ~/.zshrc
    ```

    Ajoutez-y :
    ```bash
    # Raccourcis pour Jellyfin
    alias backup-jellyfin="${HOME}/.local/bin/jellyfin-backup.sh"
    alias restore-jellyfin="${HOME}/.local/bin/jellyfin-restore.sh"
    ```

    Rechargez :
    ```bash
    source ~/.bashrc # ou source ~/.zshrc
    ```

---

## 4. Procédure de restauration de la configuration

Cette procédure permet de restaurer complètement les profils utilisateurs et la bibliothèque de Jellyfin.

### Étape 4.1 : Création du script de restauration

Créez le fichier de script `~/.local/bin/jellyfin-restore.sh` :

```bash
kate ~/.local/bin/jellyfin-restore.sh
```

Ajoutez-y le contenu du script de restauration :

```bash title="~/.local/bin/jellyfin-restore.sh" linenums="1"
#!/usr/bin/env bash

# ==============================================================================
# Script de restauration de la configuration Jellyfin (Podman Rootless)
# ==============================================================================

# Configuration
BACKUP_DIR="${HOME}/Documents/Sauvegardes/jellyfin"       # Dossier cible contenant les archives
JELLYFIN_CONFIG="${HOME}/.local/share/jellyfin/config"     # Chemin vers le dossier config à écraser

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERREUR:${NC} $1"
}

warning() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] ATTENTION:${NC} $1"
}

info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

log "===== Début des vérifications préalables ====="

# Vérification du dossier de sauvegarde
if [ ! -d "${BACKUP_DIR}" ]; then
    error "Le dossier de sauvegarde est introuvable : ${BACKUP_DIR}"
    exit 1
fi

# Recherche de l'archive la plus récente ou du lien symbolique
if [ -f "${BACKUP_DIR}/jellyfin_config_latest.tar.gz" ]; then
    BACKUP_FILE="${BACKUP_DIR}/jellyfin_config_latest.tar.gz"
else
    BACKUP_FILE=$(ls -t "${BACKUP_DIR}"/jellyfin_config_*.tar.gz 2>/dev/null | head -n1)
fi

if [ -z "${BACKUP_FILE}" ] || [ ! -f "${BACKUP_FILE}" ]; then
    error "Aucune archive de sauvegarde valide n'a été trouvée dans ${BACKUP_DIR}"
    exit 1
fi

log "✓ Archive identifiée : $(basename "${BACKUP_FILE}")"

# Confirmation de l'utilisateur
echo ""
warning "╔════════════════════════════════════════════════════════════╗"
warning "║  ATTENTION : RESTAURATION DE LA CONFIGURATION JELLYFIN     ║"
warning "╚════════════════════════════════════════════════════════════╝"
echo ""
info "Cette opération va arrêter le conteneur Jellyfin et remplacer"
info "intégralement la configuration actuelle par le contenu de l'archive."
info "Les fichiers média ne seront pas affectés."
echo ""
read -p "Saisissez 'OUI' pour confirmer la restauration : " confirm

if [ "${confirm}" != "OUI" ]; then
    log "Opération annulée par l'utilisateur."
    exit 0
fi

log "===== Étape 1/4 : Arrêt du service Jellyfin ====="
if systemctl --user is-active --quiet jellyfin.service; then
    systemctl --user stop jellyfin.service
    log "✓ Service Jellyfin arrêté."
else
    info "Le service Jellyfin n'était pas actif."
fi

log "===== Étape 2/4 : Sauvegarde de sécurité de l'état actuel ====="
if [ -d "${JELLYFIN_CONFIG}" ]; then
    SAFETY_BACKUP="${JELLYFIN_CONFIG}_safety_$(date +%Y%m%d_%H%M%S)"
    log "Création d'une copie de sauvegarde temporaire dans : ${SAFETY_BACKUP}"
    cp -a "${JELLYFIN_CONFIG}" "${SAFETY_BACKUP}"
    log "✓ Copie de sécurité créée."
fi

log "===== Étape 3/4 : Remplacement de la configuration ====="
if [ -d "${JELLYFIN_CONFIG}" ]; then
    log "Suppression de l'ancienne configuration..."
    rm -rf "${JELLYFIN_CONFIG}"
fi

mkdir -p "$(dirname "${JELLYFIN_CONFIG}")"

log "Extraction de l'archive..."
if tar -xzf "${BACKUP_FILE}" -C "$(dirname "${JELLYFIN_CONFIG}")"; then
    log "✓ Fichiers extraits."
else
    error "Échec lors de l'extraction."
    # Restauration automatique de secours
    if [ -d "${SAFETY_BACKUP}" ]; then
        warning "Échec critique. Restauration de la copie de sécurité..."
        mv "${SAFETY_BACKUP}" "${JELLYFIN_CONFIG}"
    fi
    exit 1
fi

log "===== Étape 4/4 : Redémarrage de Jellyfin ====="
systemctl --user start jellyfin.service
sleep 5

if systemctl --user is-active --quiet jellyfin.service; then
    log "✓ Le service Jellyfin est démarré et actif."
else
    error "Le service n'a pas pu redémarrer. Exécutez 'journalctl --user -u jellyfin.service' pour déboguer."
    exit 1
fi

echo ""
log "╔════════════════════════════════════════════════════════════╗"
log "║           RESTAURATION TERMINÉE AVEC SUCCÈS                ║"
log "╚════════════════════════════════════════════════════════════╝"
echo ""
info "Veuillez vérifier l'accessibilité de l'interface (http://localhost:8096)."
info "La copie de sécurité temporaire a été conservée sous : ${SAFETY_BACKUP}"
```

Rendez le script de restauration exécutable :

```bash
chmod +x ~/.local/bin/jellyfin-restore.sh
```

---

## 5. Commandes utiles de diagnostic

### Exécuter une sauvegarde immédiate
```bash
backup-jellyfin
```
Ou via systemd :
```bash
systemctl --user start jellyfin-backup.service
```

### Surveiller les journaux (logs) d'exécution
```bash
journalctl --user -u jellyfin-backup.service -f
```

### Inspecter le contenu d'un backup sans l'extraire
```bash
tar -tzf ~/Documents/Sauvegardes/jellyfin/jellyfin_config_latest.tar.gz | head -n 20
```