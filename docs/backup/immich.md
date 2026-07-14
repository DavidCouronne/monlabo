# Sauvegarde et Restauration Immich (Podman)

Cette documentation présente les procédures de sauvegarde et de restauration de l'instance Immich déployée en mode rootless avec Podman et des Quadlets Systemd.

---

## 1. Vue d'ensemble de la stratégie

La stratégie de sauvegarde d'Immich se divise en deux parties distinctes :

1. **Données média (photos et vidéos)** : Sauvegarde incrémentielle du dossier source via Duplicati vers OneDrive.
2. **Base de données PostgreSQL** : Dump quotidien automatique de la base de données, stocké localement, puis synchronisé vers OneDrive via Duplicati.

> [!CAUTION]
> **Règle critique de restauration :**
> Ne démarrez jamais l'interface d'Immich sur une nouvelle installation avant d'avoir restauré la base de données. Immich associe de manière stricte les fichiers physiques aux enregistrements en base de données. Lancer l'application à vide générerait une base de données incohérente.

---

## 2. Configuration de la sauvegarde de la base de données

Le script de sauvegarde s'exécute en mode utilisateur (rootless) et ne nécessite aucun privilège administrateur (`sudo`).

### Étape 2.1 : Création du script de sauvegarde

Créez le dossier pour les scripts personnels s'il n'existe pas :

```bash
mkdir -p ~/.local/bin
```

Créez le fichier de script `~/.local/bin/immich-backup.sh` :

```bash
kate ~/.local/bin/immich-backup.sh
```

Ajoutez-y le contenu suivant :

```bash title="~/.local/bin/immich-backup.sh" linenums="1"
#!/usr/bin/env bash

# ==============================================================================
# Script de sauvegarde de la base de données Immich (Podman Rootless)
# ==============================================================================

# Configuration
BACKUP_DIR="${HOME}/Documents/Sauvegardes/immich_db"  # Dossier cible à synchroniser par Duplicati
DB_USERNAME="immich"                                 # Utilisateur de la base de données (défini dans immich-db.container)
KEEP_BACKUPS=14                                      # Nombre de sauvegardes locales à conserver

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

# Vérification de l'accessibilité de Podman
if ! podman ps &> /dev/null; then
    error "Podman n'est pas accessible ou ne fonctionne pas."
    exit 1
fi

# Vérification que le conteneur de base de données tourne
if ! podman ps | grep -q "immich-db"; then
    error "Le conteneur de base de données 'immich-db' n'est pas actif."
    exit 1
fi

# Création du dossier de sauvegarde
mkdir -p "${BACKUP_DIR}"

# Définition du fichier de dump
DATE=$(date +%Y%m%d_%H%M%S)
DUMP_FILE="${BACKUP_DIR}/immich_db_${DATE}.sql.gz"

log "===== Début de la sauvegarde de la base de données Immich ====="
log "Génération du dump PostgreSQL..."

# Exécution du dump (pg_dumpall pour inclure les schémas et rôles)
if podman exec -t immich-db pg_dumpall --clean --if-exists --username="${DB_USERNAME}" | gzip > "${DUMP_FILE}"; then
    
    # Vérification que le fichier généré n'est pas vide
    if [ -s "${DUMP_FILE}" ]; then
        SIZE=$(du -h "${DUMP_FILE}" | cut -f1)
        log "✓ Sauvegarde réussie : ${DUMP_FILE}"
        log "  Taille : ${SIZE}"
        
        # Mise à jour du lien symbolique vers la dernière version
        ln -sf "$(basename ${DUMP_FILE})" "${BACKUP_DIR}/immich_db_latest.sql.gz"
        
        # Génération du fichier d'informations contextuelles
        cat > "${BACKUP_DIR}/backup_${DATE}_info.txt" << EOF
Date du backup: $(date)
Machine hôte: $(hostname)
Image de base de données: $(podman inspect immich-db --format='{{.ImageName}}' 2>/dev/null || echo "N/A")
Utilisateur DB: ${DB_USERNAME}
Fichier généré: $(basename ${DUMP_FILE})
EOF
        
    else
        error "Le fichier dump généré est vide !"
        rm -f "${DUMP_FILE}"
        exit 1
    fi
else
    error "Échec lors de l'exécution de pg_dumpall."
    exit 1
fi

# Nettoyage des sauvegardes obsolètes (rétention locale)
log "Nettoyage des anciennes sauvegardes (conservation des ${KEEP_BACKUPS} dernières)..."
cd "${BACKUP_DIR}" || exit 1
ls -t immich_db_*.sql.gz 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f
ls -t backup_*_info.txt 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f

log "✓ Nettoyage terminé."
log "===== Sauvegarde terminée avec succès ====="
log "Dossier prêt pour la synchronisation Duplicati : ${BACKUP_DIR}"
```

Rendez le script exécutable :

```bash
chmod +x ~/.local/bin/immich-backup.sh
```

### Étape 2.2 : Automatisation via Systemd User Timers

Pour exécuter la sauvegarde quotidiennement sans privilèges root, utilisez le gestionnaire Systemd de votre session utilisateur.

Créez le dossier des services utilisateur si nécessaire :

```bash
mkdir -p ~/.config/systemd/user
```

#### Fichier Service (`~/.config/systemd/user/immich-backup.service`) :

```ini title="~/.config/systemd/user/immich-backup.service"
[Unit]
Description=Sauvegarde de la base de données Immich (User)
After=immich-db.service

[Service]
Type=oneshot
ExecStart=%h/.local/bin/immich-backup.sh
StandardOutput=journal
StandardError=journal
```

#### Fichier Timer (`~/.config/systemd/user/immich-backup.timer`) :

```ini title="~/.config/systemd/user/immich-backup.timer"
[Unit]
Description=Déclencheur journalier de sauvegarde Immich

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

#### Activation du Timer :

Chargez la configuration Systemd et activez le déclencheur :

```bash
systemctl --user daemon-reload
systemctl --user enable --now immich-backup.timer
```

Vérifiez le statut du timer :

```bash
systemctl --user status immich-backup.timer
systemctl --user list-timers --all | grep immich
```

---

## 3. Configuration des alias de commandes

Pour simplifier les interactions, configurez des alias dans votre shell de session.

=== "Fish Shell"

    Éditez le fichier de configuration de Fish :
    ```bash
    kate ~/.config/fish/config.fish
    ```

    Ajoutez les lignes suivantes à la fin :
    ```fish
    # Raccourcis pour Immich
    alias backup-immich='~/.local/bin/immich-backup.sh'
    alias restore-immich='~/.local/bin/immich-restore.sh'
    ```

    Rechargez le fichier :
    ```bash
    source ~/.config/fish/config.fish
    ```

=== "Bash / Zsh"

    Éditez votre fichier `~/.bashrc` ou `~/.zshrc` :
    ```bash
    kate ~/.bashrc # ou ~/.zshrc
    ```

    Ajoutez-y :
    ```bash
    # Raccourcis pour Immich
    alias backup-immich="${HOME}/.local/bin/immich-backup.sh"
    alias restore-immich="${HOME}/.local/bin/immich-restore.sh"
    ```

    Rechargez :
    ```bash
    source ~/.bashrc # ou source ~/.zshrc
    ```

---

## 4. Procédure de restauration de la base de données

En cas de panne matérielle ou de migration vers une nouvelle installation, cette procédure permet de restaurer l'intégralité d'Immich.

### Étape 4.1 : Création du script de restauration

Créez le fichier de script `~/.local/bin/immich-restore.sh` :

```bash
kate ~/.local/bin/immich-restore.sh
```

Ajoutez-y le contenu du script de restauration :

```bash title="~/.local/bin/immich-restore.sh" linenums="1"
#!/usr/bin/env bash

# ==============================================================================
# Script de restauration de la base de données Immich (Podman Rootless)
# ==============================================================================

# Configuration
BACKUP_DB_DIR="${HOME}/Documents/Sauvegardes/immich_db" # Emplacement où les sauvegardes ont été restaurées
DB_USERNAME="immich"                                    # Utilisateur PostgreSQL

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
if [ ! -d "${BACKUP_DB_DIR}" ]; then
    error "Le dossier de sauvegarde est introuvable : ${BACKUP_DB_DIR}"
    exit 1
fi

# Recherche du dump le plus récent ou du lien symbolique
if [ -f "${BACKUP_DB_DIR}/immich_db_latest.sql.gz" ]; then
    DB_DUMP="${BACKUP_DB_DIR}/immich_db_latest.sql.gz"
else
    DB_DUMP=$(ls -t "${BACKUP_DB_DIR}"/immich_db_*.sql.gz 2>/dev/null | head -n1)
fi

if [ -z "${DB_DUMP}" ] || [ ! -f "${DB_DUMP}" ]; then
    error "Aucune archive de base de données valide n'a été trouvée dans ${BACKUP_DB_DIR}"
    exit 1
fi

log "✓ Archive de sauvegarde identifiée : $(basename "${DB_DUMP}")"

# Confirmation de l'utilisateur
echo ""
warning "╔════════════════════════════════════════════════════════════╗"
warning "║  ATTENTION : RESTAURATION COMPLÈTE D'IMMICH                ║"
warning "╚════════════════════════════════════════════════════════════╝"
echo ""
info "Cette action va écraser les données actuelles d'Immich."
info "Le dump de la base de données à restaurer est : $(basename "${DB_DUMP}")"
echo ""
read -p "Saisissez 'OUI' pour confirmer la restauration : " confirm

if [ "${confirm}" != "OUI" ]; then
    log "Opération annulée par l'utilisateur."
    exit 0
fi

log "===== Étape 1/4 : Arrêt de l'instance Immich ====="
# Arrêt complet du pod Immich (qui arrête automatiquement tous ses conteneurs liés)
if systemctl --user is-active --quiet immich-pod.service; then
    log "Arrêt des services systemd d'Immich..."
    systemctl --user stop immich-pod.service
    log "✓ Services arrêtés."
else
    info "Le service Immich n'est pas actif actuellement."
fi

log "===== Étape 2/4 : Démarrage du service de base de données seul ====="
# Démarrage isolé de la base de données pour permettre la restauration
systemctl --user start immich-db.service

log "Attente de l'initialisation de PostgreSQL..."
for i in {1..15}; do
    if podman exec immich-db pg_isready -U "${DB_USERNAME}" &> /dev/null; then
        log "✓ PostgreSQL est prêt à recevoir des requêtes."
        break
    fi
    if [ "$i" -eq 15 ]; then
        error "Impossible de joindre PostgreSQL après 15 tentatives."
        exit 1
    fi
    sleep 2
done

log "===== Étape 3/4 : Restauration des schémas et des données ====="
info "Restauration en cours..."

# Décompression et injection du dump dans le conteneur pgvecto-rs
gunzip --stdout "${DB_DUMP}" \
| podman exec -i immich-db psql --username="${DB_USERNAME}" --dbname=immich 2>&1 \
| grep -v "^$"

if [ ${PIPESTATUS[0]} -eq 0 ] && [ ${PIPESTATUS[1]} -eq 0 ]; then
    log "✓ Base de données restaurée avec succès."
else
    error "Une erreur est survenue lors de l'injection du dump SQL."
    exit 1
fi

log "===== Étape 4/4 : Redémarrage de l'instance complète ====="
# Redémarrage de l'ensemble des services définis par le Quadlet pod
systemctl --user restart immich-pod.service

log "Attente du démarrage des conteneurs applicatifs (20 secondes)..."
sleep 20

# Vérification du statut final
if systemctl --user is-active --quiet immich-server.service; then
    log "✓ Les services applicatifs d'Immich sont actifs."
else
    warning "Le serveur ne semble pas avoir démarré correctement. Veuillez exécuter 'journalctl --user -u immich-server.service'."
fi

echo ""
log "╔════════════════════════════════════════════════════════════╗"
log "║           RESTAURATION TERMINÉE AVEC SUCCÈS                ║"
log "╚════════════════════════════════════════════════════════════╝"
echo ""
info "Veuillez vérifier l'accessibilité de l'interface web (http://localhost:2283)."
info "Si les miniatures ne s'affichent pas immédiatement, relancez le job d'indexation depuis l'onglet Administration."
```

Rendez le script de restauration exécutable :

```bash
chmod +x ~/.local/bin/immich-restore.sh
```

---

## 5. Commandes utiles de diagnostic

### Exécuter une sauvegarde immédiate
```bash
backup-immich
```
Ou via systemd :
```bash
systemctl --user start immich-backup.service
```

### Surveiller les journaux (logs) d'exécution
```bash
journalctl --user -u immich-backup.service -f
```

### Vérifier le timer planifié
```bash
systemctl --user list-timers --all | grep immich
```