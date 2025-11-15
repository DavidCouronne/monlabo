# Backup et Restauration Jellyfin

!!! warning "Prérequis"
    Cette documentation suppose que vous avez installé Jellyfin via Docker et configuré Duplicati pour les backups incrémentiels vers OneDrive.

## Vue d'ensemble

La stratégie de backup Jellyfin se compose de deux parties :

1. **Fichiers média** : Vos films, séries, musiques (généralement déjà sur un NAS avec backup séparé)
2. **Configuration Jellyfin** : Base de données SQLite, métadonnées, images, paramètres utilisateurs

!!! info "Contenu de la configuration Jellyfin"
    - Base de données SQLite (bibliothèque, utilisateurs, vues)
    - Métadonnées et images téléchargées
    - Configuration des plugins
    - Paramètres du serveur
    - Historiques de lecture

## Architecture des données Jellyfin

Jellyfin stocke ses données dans un dossier de configuration unique (généralement `/config` dans le conteneur Docker).

### Structure du dossier config

```
/config/
├── data/                    # Base de données SQLite
│   ├── library.db          # Bibliothèque média principale
│   ├── jellyfin.db         # Configuration serveur
│   └── authentication.db   # Utilisateurs et authentification
├── metadata/               # Métadonnées téléchargées
│   ├── library/           # Posters, fanart, etc.
│   └── People/            # Photos des acteurs
├── plugins/               # Plugins installés
├── root/                  # Configuration système
└── log/                   # Logs (optionnel à sauvegarder)
```

!!! tip "Fichiers média séparés"
    Les fichiers média eux-mêmes (films, séries) sont généralement stockés séparément et doivent avoir leur propre stratégie de backup (RAID sur NAS, backup cloud, etc.).

## Configuration du backup

### Script de backup

Créez le script de backup :

```bash
sudo kate /usr/local/bin/jellyfin-backup.sh
```

Copiez le contenu suivant :

```bash title="/usr/local/bin/jellyfin-backup.sh" linenums="1"
#!/usr/bin/env bash

# Script de backup de la configuration Jellyfin
# Les fichiers média sont gérés séparément (NAS)

# Configuration
BACKUP_DIR="/path/to/backup/jellyfin"  # À ADAPTER : dossier synchronisé par Duplicati
JELLYFIN_CONFIG="/path/to/jellyfin/config"  # À ADAPTER : chemin du dossier config Jellyfin
KEEP_BACKUPS=14  # Nombre de backups à conserver

# Couleurs pour les logs
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

# Vérification que Docker est lancé
if ! docker ps &> /dev/null; then
    error "Docker n'est pas accessible."
    exit 1
fi

# Vérification que le conteneur Jellyfin existe
if ! docker ps -a | grep -q jellyfin; then
    error "Le conteneur jellyfin n'existe pas."
    exit 1
fi

# Vérification que le dossier config existe
if [ ! -d "${JELLYFIN_CONFIG}" ]; then
    error "Le dossier de configuration Jellyfin n'existe pas : ${JELLYFIN_CONFIG}"
    exit 1
fi

# Création du dossier de backup
mkdir -p "${BACKUP_DIR}"

# Nom du fichier avec timestamp
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/jellyfin_config_${DATE}.tar.gz"

log "===== Backup de la configuration Jellyfin ====="

# Option : Arrêter Jellyfin pendant le backup pour cohérence des données
read -p "Voulez-vous arrêter Jellyfin pendant le backup ? (o/N) : " stop_jellyfin
JELLYFIN_WAS_RUNNING=false

if [[ "$stop_jellyfin" =~ ^[Oo]$ ]]; then
    if docker ps | grep -q jellyfin; then
        log "Arrêt de Jellyfin..."
        docker stop jellyfin
        JELLYFIN_WAS_RUNNING=true
        sleep 5
        log "✓ Jellyfin arrêté"
    fi
else
    warning "Backup à chaud : les données peuvent être incohérentes si Jellyfin écrit pendant le backup"
fi

# Création de l'archive
log "Création de l'archive de configuration..."
if tar -czf "${BACKUP_FILE}" -C "$(dirname ${JELLYFIN_CONFIG})" "$(basename ${JELLYFIN_CONFIG})" 2>/dev/null; then
    
    # Vérification que le fichier n'est pas vide
    if [ -s "${BACKUP_FILE}" ]; then
        SIZE=$(du -h "${BACKUP_FILE}" | cut -f1)
        log "✓ Backup réussi : ${BACKUP_FILE}"
        log "  Taille : ${SIZE}"
        
        # Création d'un lien symbolique vers le dernier backup
        ln -sf "$(basename ${BACKUP_FILE})" "${BACKUP_DIR}/jellyfin_config_latest.tar.gz"
        
        # Informations de contexte
        cat > "${BACKUP_DIR}/backup_${DATE}_info.txt" << EOF
Date du backup: $(date)
Hostname: $(hostname)
Version Jellyfin: $(docker inspect jellyfin --format='{{.Config.Image}}' 2>/dev/null || echo "N/A")
Config path: ${JELLYFIN_CONFIG}
Fichier backup: $(basename ${BACKUP_FILE})
Jellyfin arrêté: ${JELLYFIN_WAS_RUNNING}
EOF
        
    else
        error "Le fichier backup est vide !"
        rm -f "${BACKUP_FILE}"
        exit 1
    fi
else
    error "Échec de la création de l'archive"
    exit 1
fi

# Redémarrer Jellyfin si on l'avait arrêté
if [ "$JELLYFIN_WAS_RUNNING" = true ]; then
    log "Redémarrage de Jellyfin..."
    docker start jellyfin
    sleep 5
    log "✓ Jellyfin redémarré"
fi

# Nettoyage des anciens backups
log "Nettoyage des anciens backups (conservation des ${KEEP_BACKUPS} derniers)..."
cd "${BACKUP_DIR}"
ls -t jellyfin_config_*.tar.gz 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f
ls -t backup_*_info.txt 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f

log "✓ Nettoyage terminé"
log "===== Backup terminé avec succès ====="
log ""
log "Duplicati va synchroniser ce dossier : ${BACKUP_DIR}"
```

!!! tip "Backup à chaud vs à froid"
    - **À froid** (Jellyfin arrêté) : Garantit la cohérence des données SQLite
    - **À chaud** (Jellyfin en cours) : Plus pratique mais risque de corruption mineure

Rendez le script exécutable :

```bash
sudo chmod +x /usr/local/bin/jellyfin-backup.sh
```

### Alias Fish Shell

=== "Configuration Fish"

    Éditez votre configuration Fish :
    
    ```bash
    kate ~/.config/fish/config.fish
    ```
    
    Ajoutez cet alias :
    
    ```fish title="~/.config/fish/config.fish"
    # Alias Jellyfin
    alias backup-jellyfin='sudo /usr/local/bin/jellyfin-backup.sh'
    alias restore-jellyfin='sudo /usr/local/bin/jellyfin-restore.sh'
    ```
    
    Rechargez la configuration :
    
    ```bash
    source ~/.config/fish/config.fish
    ```

=== "Configuration Bash/Zsh"

    Éditez votre `~/.bashrc` ou `~/.zshrc` :
    
    ```bash
    # Alias Jellyfin
    alias backup-jellyfin='sudo /usr/local/bin/jellyfin-backup.sh'
    alias restore-jellyfin='sudo /usr/local/bin/jellyfin-restore.sh'
    ```
    
    Rechargez la configuration :
    
    ```bash
    source ~/.bashrc  # ou source ~/.zshrc
    ```

### Automatisation avec systemd

=== "Service systemd"

    Créez le fichier de service :
    
    ```bash
    sudo kate /etc/systemd/system/jellyfin-backup.service
    ```
    
    Contenu :
    
    ```ini title="/etc/systemd/system/jellyfin-backup.service"
    [Unit]
    Description=Backup Jellyfin Configuration
    After=docker.service
    Requires=docker.service
    
    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/jellyfin-backup.sh
    User=root
    StandardOutput=journal
    StandardError=journal
    StandardInput=null
    # Réponse automatique "N" pour ne pas arrêter Jellyfin en mode automatique
    Environment="stop_jellyfin=N"
    ```

=== "Timer systemd"

    Créez le timer :
    
    ```bash
    sudo kate /etc/systemd/system/jellyfin-backup.timer
    ```
    
    Contenu :
    
    ```ini title="/etc/systemd/system/jellyfin-backup.timer"
    [Unit]
    Description=Backup Jellyfin Configuration Daily
    Requires=jellyfin-backup.service
    
    [Timer]
    OnCalendar=daily
    OnCalendar=03:00
    Persistent=true
    
    [Install]
    WantedBy=timers.target
    ```

=== "Alternative : Cron"

    Si vous préférez utiliser cron :
    
    ```bash
    sudo crontab -e
    ```
    
    Ajoutez (avec réponse automatique "N") :
    
    ```cron
    0 3 * * * echo "N" | /usr/local/bin/jellyfin-backup.sh >> /var/log/jellyfin-backup.log 2>&1
    ```

Activez le timer systemd :

```bash
sudo systemctl daemon-reload
sudo systemctl enable jellyfin-backup.timer
sudo systemctl start jellyfin-backup.timer
```

Vérifiez le statut :

```bash
# Statut du timer
sudo systemctl status jellyfin-backup.timer

# Liste des timers actifs
sudo systemctl list-timers --all | grep jellyfin
```

### Configuration Duplicati

Créez un job Duplicati pour Jellyfin :

!!! info "Job Duplicati - Configuration Jellyfin"
    - **Source** : `/path/to/backup/jellyfin/`
    - **Destination** : OneDrive
    - **Planification** : Quotidienne à 04:00 (après le script à 03:00)
    - **Rétention** : Garder les 30 derniers jours
    - **Chiffrement** : AES-256

## Script de restauration

Créez le script de restauration :

```bash
sudo kate /usr/local/bin/jellyfin-restore.sh
```

Copiez le contenu suivant :

??? note "Script de restauration complet (cliquez pour déplier)"
    ```bash title="/usr/local/bin/jellyfin-restore.sh" linenums="1"
    #!/usr/bin/env bash
    
    # Script de restauration Jellyfin avec Duplicati
    # ATTENTION : Cette opération écrase la configuration actuelle !
    
    # ==============================
    # CONFIGURATION À ADAPTER
    # ==============================
    BACKUP_PATH="/path/to/restored/jellyfin"  # Où Duplicati a restauré les backups
    JELLYFIN_CONFIG="/path/to/jellyfin/config"  # Destination de la config Jellyfin
    COMPOSE_FILE="/path/to/docker-compose.yml"  # Votre docker-compose.yml
    
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
    
    # ==============================
    # VÉRIFICATIONS PRÉALABLES
    # ==============================
    log "===== Vérifications préalables ====="
    
    # Vérifier que le dossier de backup existe
    if [ ! -d "${BACKUP_PATH}" ]; then
        error "Dossier de backup introuvable : ${BACKUP_PATH}"
        exit 1
    fi
    
    # Chercher le dernier backup (ou utiliser le lien symbolique latest)
    if [ -L "${BACKUP_PATH}/jellyfin_config_latest.tar.gz" ]; then
        BACKUP_FILE="${BACKUP_PATH}/jellyfin_config_latest.tar.gz"
    elif [ -f "${BACKUP_PATH}/jellyfin_config_latest.tar.gz" ]; then
        BACKUP_FILE="${BACKUP_PATH}/jellyfin_config_latest.tar.gz"
    else
        # Trouver le plus récent
        BACKUP_FILE=$(ls -t "${BACKUP_PATH}"/jellyfin_config_*.tar.gz 2>/dev/null | head -n1)
    fi
    
    if [ -z "${BACKUP_FILE}" ] || [ ! -f "${BACKUP_FILE}" ]; then
        error "Aucune archive de backup trouvée dans ${BACKUP_PATH}"
        exit 1
    fi
    
    log "✓ Archive de backup trouvée : $(basename ${BACKUP_FILE})"
    
    # ==============================
    # CONFIRMATION UTILISATEUR
    # ==============================
    echo ""
    warning "╔════════════════════════════════════════════════════════════╗"
    warning "║  ATTENTION : RESTAURATION CONFIGURATION JELLYFIN           ║"
    warning "╚════════════════════════════════════════════════════════════╝"
    echo ""
    info "Cette opération va :"
    info "  1. Arrêter Jellyfin"
    info "  2. Sauvegarder la configuration actuelle (par sécurité)"
    info "  3. Restaurer la configuration depuis : $(basename ${BACKUP_FILE})"
    info "  4. Redémarrer Jellyfin"
    echo ""
    warning "Les fichiers média ne sont PAS affectés (ils restent sur le NAS)"
    echo ""
    read -p "Tapez 'OUI' en majuscules pour continuer : " confirm
    
    if [ "$confirm" != "OUI" ]; then
        log "Restauration annulée."
        exit 0
    fi
    
    # ==============================
    # ÉTAPE 1 : ARRÊT DE JELLYFIN
    # ==============================
    log ""
    log "===== ÉTAPE 1/5 : Arrêt de Jellyfin ====="
    
    cd "$(dirname ${COMPOSE_FILE})"
    
    if docker ps | grep -q jellyfin; then
        log "Arrêt de Jellyfin..."
        docker stop jellyfin
        sleep 5
        log "✓ Jellyfin arrêté"
    else
        info "Jellyfin n'était pas démarré"
    fi
    
    # ==============================
    # ÉTAPE 2 : SAUVEGARDE DE SÉCURITÉ
    # ==============================
    log ""
    log "===== ÉTAPE 2/5 : Sauvegarde de sécurité de la config actuelle ====="
    
    if [ -d "${JELLYFIN_CONFIG}" ]; then
        SAFETY_BACKUP="${JELLYFIN_CONFIG}_backup_$(date +%Y%m%d_%H%M%S)"
        log "Copie de sécurité vers : ${SAFETY_BACKUP}"
        cp -a "${JELLYFIN_CONFIG}" "${SAFETY_BACKUP}"
        log "✓ Sauvegarde de sécurité créée"
    else
        info "Aucune configuration existante à sauvegarder"
    fi
    
    # ==============================
    # ÉTAPE 3 : SUPPRESSION CONFIG ACTUELLE
    # ==============================
    log ""
    log "===== ÉTAPE 3/5 : Suppression de la configuration actuelle ====="
    
    if [ -d "${JELLYFIN_CONFIG}" ]; then
        warning "Suppression de : ${JELLYFIN_CONFIG}"
        rm -rf "${JELLYFIN_CONFIG}"
        log "✓ Configuration actuelle supprimée"
    fi
    
    # ==============================
    # ÉTAPE 4 : EXTRACTION DU BACKUP
    # ==============================
    log ""
    log "===== ÉTAPE 4/5 : Extraction du backup ====="
    
    mkdir -p "$(dirname ${JELLYFIN_CONFIG})"
    
    info "Extraction en cours..."
    if tar -xzf "${BACKUP_FILE}" -C "$(dirname ${JELLYFIN_CONFIG})"; then
        log "✓ Backup extrait vers ${JELLYFIN_CONFIG}"
    else
        error "Échec de l'extraction du backup"
        # Restaurer la sauvegarde de sécurité si elle existe
        if [ -d "${SAFETY_BACKUP}" ]; then
            warning "Restauration de la sauvegarde de sécurité..."
            mv "${SAFETY_BACKUP}" "${JELLYFIN_CONFIG}"
        fi
        exit 1
    fi
    
    # Vérification des permissions
    log "Ajustement des permissions..."
    chown -R 1000:1000 "${JELLYFIN_CONFIG}" 2>/dev/null || true
    log "✓ Permissions ajustées"
    
    # ==============================
    # ÉTAPE 5 : REDÉMARRAGE DE JELLYFIN
    # ==============================
    log ""
    log "===== ÉTAPE 5/5 : Redémarrage de Jellyfin ====="
    
    docker start jellyfin
    
    log "Attente du démarrage de Jellyfin (30 secondes)..."
    sleep 30
    
    # Vérification que Jellyfin est lancé
    if docker ps | grep -q jellyfin; then
        log "✓ Jellyfin démarré avec succès"
    else
        error "Jellyfin ne semble pas avoir démarré correctement"
        error "Vérifiez les logs : docker logs jellyfin"
        exit 1
    fi
    
    # ==============================
    # RÉCAPITULATIF
    # ==============================
    log ""
    log "╔════════════════════════════════════════════════════════════╗"
    log "║           RESTAURATION TERMINÉE AVEC SUCCÈS                ║"
    log "╚════════════════════════════════════════════════════════════╝"
    echo ""
    info "Prochaines étapes :"
    echo ""
    echo "  1. Accédez à l'interface web Jellyfin"
    echo "     (généralement http://localhost:8096)"
    echo ""
    echo "  2. Connectez-vous avec vos identifiants"
    echo ""
    echo "  3. Vérifiez que :"
    echo "     - Vos bibliothèques sont présentes"
    echo "     - Les utilisateurs sont conservés"
    echo "     - Les historiques de lecture sont là"
    echo "     - Les plugins sont actifs"
    echo ""
    echo "  4. Si les chemins de bibliothèque ont changé :"
    echo "     Tableau de bord → Bibliothèques → Modifier les chemins"
    echo ""
    info "Sauvegarde de sécurité conservée : ${SAFETY_BACKUP}"
    info "Vous pouvez la supprimer si tout fonctionne correctement"
    echo ""
    log "Restauration complète !"
    ```

Rendez le script exécutable :

```bash
sudo chmod +x /usr/local/bin/jellyfin-restore.sh
```

## Procédure de restauration

### Étapes de restauration

1. **Restaurer le dossier de backup avec Duplicati**
   
    Restaurez le dossier contenant les archives de configuration

2. **Adapter le script de restauration**
   
    Éditez `/usr/local/bin/jellyfin-restore.sh` et modifiez :
    
    - `BACKUP_PATH` : où Duplicati a restauré les archives
    - `JELLYFIN_CONFIG` : destination de la config Jellyfin
    - `COMPOSE_FILE` : chemin vers votre docker-compose.yml

3. **Lancer la restauration**
   
    ```bash
    restore-jellyfin
    ```

4. **Vérifications post-restauration**
   
    - Connectez-vous à l'interface web
    - Vérifiez les bibliothèques
    - Testez la lecture d'un média
    - Vérifiez les comptes utilisateurs

### Cas particuliers

#### Changement de chemins média

Si vos fichiers média ont changé d'emplacement (nouveau NAS, nouvelle structure), vous devrez mettre à jour les chemins dans Jellyfin :

!!! example "Mise à jour des chemins"
    1. Connectez-vous à Jellyfin
    2. Allez dans **Tableau de bord** → **Bibliothèques**
    3. Pour chaque bibliothèque : **⋮** → **Gérer la bibliothèque**
    4. Modifiez les dossiers dans l'onglet **Dossiers**
    5. Scannez à nouveau la bibliothèque

#### Restauration partielle

Pour restaurer uniquement certains éléments :

```bash
# Extraire seulement la base de données
tar -xzf jellyfin_config_latest.tar.gz config/data/

# Extraire seulement les plugins
tar -xzf jellyfin_config_latest.tar.gz config/plugins/
```

## Commandes utiles

### Backup manuel

```bash
# Lancer un backup immédiatement
backup-jellyfin

# Ou via systemd
sudo systemctl start jellyfin-backup.service
```

### Consultation des logs

=== "Systemd"

    ```bash
    # Logs du dernier backup
    sudo journalctl -u jellyfin-backup.service -n 50
    
    # Suivre les logs en temps réel
    sudo journalctl -u jellyfin-backup.service -f
    
    # Logs depuis hier
    sudo journalctl -u jellyfin-backup.service --since yesterday
    ```

=== "Cron"

    ```bash
    # Voir les logs (si configuré dans cron)
    sudo tail -f /var/log/jellyfin-backup.log
    ```

### Gestion du timer

```bash
# Statut du timer
sudo systemctl status jellyfin-backup.timer

# Désactiver temporairement
sudo systemctl stop jellyfin-backup.timer

# Réactiver
sudo systemctl start jellyfin-backup.timer
```

### Inspection des backups

```bash
# Lister le contenu d'une archive sans extraire
tar -tzf /path/to/backup/jellyfin_config_latest.tar.gz | head -n 20

# Vérifier la taille
du -h /path/to/backup/jellyfin_config_*.tar.gz

# Vérifier l'intégrité
tar -tzf /path/to/backup/jellyfin_config_latest.tar.gz > /dev/null && echo "Archive OK"
```

## Dépannage

### Le backup ne se lance pas

Vérifiez que :

1. Docker est démarré : `sudo systemctl status docker`
2. Le conteneur Jellyfin existe : `docker ps -a | grep jellyfin`
3. Le dossier config est accessible : `ls -la ${JELLYFIN_CONFIG}`
4. Le timer est actif : `sudo systemctl status jellyfin-backup.timer`

### La restauration échoue

!!! warning "Erreurs communes"
    - **Permissions incorrectes** : Jellyfin s'exécute souvent avec l'UID/GID 1000
    - **Archive corrompue** : Vérifiez l'intégrité avec `tar -tzf`
    - **Espace disque insuffisant** : Vérifiez avec `df -h`
    - **Jellyfin toujours en cours** : Arrêtez-le manuellement avec `docker stop jellyfin`

### Jellyfin ne démarre pas après restauration

```bash
# Voir les logs de Jellyfin
docker logs jellyfin

# Vérifier les permissions
ls -la ${JELLYFIN_CONFIG}

# Corriger les permissions si nécessaire
sudo chown -R 1000:1000 ${JELLYFIN_CONFIG}

# Redémarrer
docker restart jellyfin
```

## Optimisations

### Backup sans métadonnées

Si vous voulez réduire la taille des backups, vous pouvez exclure le dossier metadata (images, posters) qui peut être retéléchargé :

```bash
# Dans le script de backup, remplacer la ligne tar par :
tar --exclude='config/metadata' -czf "${BACKUP_FILE}" -C "$(dirname ${JELLYFIN_CONFIG})" "$(basename ${JELLYFIN_CONFIG})"
```

!!! warning "Compromis"
    Vous devrez relancer le téléchargement des métadonnées après restauration

### Backup différentiel

Pour les gros volumes, utilisez rsync au lieu de tar :

```bash
# Backup incrémental avec rsync
rsync -a --delete ${JELLYFIN_CONFIG}/ ${BACKUP_DIR}/jellyfin_latest/
```

## Migration vers un nouveau serveur

Pour migrer Jellyfin vers une nouvelle machine :

1. **Préparer le nouveau serveur**
   - Installer Docker
   - Créer la structure de dossiers

2. **Restaurer la configuration**
   - Utiliser le script de restauration
   - Adapter les chemins dans docker-compose.yml

3. **Mettre à jour les chemins média**
   - Si les chemins ont changé, les mettre à jour dans Jellyfin

4. **Vérifier les plugins**
   - Certains plugins peuvent nécessiter une réinstallation

## Stratégie 3-2-1

!!! info "Application de la règle 3-2-1"
    Configuration actuelle pour Jellyfin :
    
    - ✅ Copie 1 : Configuration sur le serveur Jellyfin
    - ✅ Copie 2 : Backups locaux (archives tar.gz)
    - ✅ Copie 3 : OneDrive (via Duplicati chiffré)
    - ✅ 2 supports : SSD local + Cloud
    - ✅ 1 hors site : OneDrive

## Références

- [Documentation officielle Jellyfin](https://jellyfin.org/docs/)
- [Jellyfin Backup Best Practices](https://jellyfin.org/docs/general/administration/backup/)
- [Docker Jellyfin](https://hub.docker.com/r/jellyfin/jellyfin)