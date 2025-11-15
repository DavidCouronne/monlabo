# Backup et Restauration Immich

!!! warning "Prérequis"
    Cette documentation suppose que vous avez déjà installé Immich via Docker et configuré Duplicati pour les backups incrémentiels des photos/vidéos vers OneDrive.

## Vue d'ensemble

La stratégie de backup Immich se compose de deux parties :

1. **Fichiers photos/vidéos** : Sauvegarde incrémentielle via Duplicati (déjà configuré)
2. **Base de données PostgreSQL** : Dump quotidien sauvegardé localement puis synchronisé via Duplicati

!!! danger "Ordre de restauration critique"
    **Ne JAMAIS lancer Immich avant de restaurer la base de données !**
    
    Immich enregistre les chemins de fichiers dans la base de données et ne scanne pas le dossier library pour mettre à jour la base. Si vous lancez Immich avant de restaurer la base, vous créerez une base vierge et ne pourrez plus restaurer proprement.

## Configuration du backup de la base de données

### Script de backup

Créez le script de backup :

```bash
sudo kate /usr/local/bin/immich-backup.sh
```

Copiez le contenu suivant :

```bash title="/usr/local/bin/immich-backup.sh" linenums="1"
#!/usr/bin/env bash

# Script de backup de la base de données Immich uniquement
# Les fichiers sont gérés par Duplicati séparément

# Configuration
BACKUP_DIR="/path/to/backup/immich_db"  # À ADAPTER : dossier qui sera synchronisé par Duplicati
DB_USERNAME="postgres"  # Username PostgreSQL (généralement "postgres")
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

# Vérification Docker
if ! docker ps &> /dev/null; then
    error "Docker n'est pas accessible."
    exit 1
fi

# Vérification conteneur PostgreSQL
if ! docker ps | grep -q immich_postgres; then
    error "Le conteneur immich_postgres n'est pas lancé."
    exit 1
fi

# Création du dossier de backup
mkdir -p "${BACKUP_DIR}"

# Nom du fichier avec timestamp
DATE=$(date +%Y%m%d_%H%M%S)
DUMP_FILE="${BACKUP_DIR}/immich_db_${DATE}.sql.gz"

log "===== Backup de la base de données Immich ====="
log "Création du dump PostgreSQL..."

# Création du dump
if docker exec -t immich_postgres pg_dumpall --clean --if-exists --username="${DB_USERNAME}" | gzip > "${DUMP_FILE}"; then
    
    # Vérification que le fichier n'est pas vide
    if [ -s "${DUMP_FILE}" ]; then
        SIZE=$(du -h "${DUMP_FILE}" | cut -f1)
        log "✓ Backup réussi : ${DUMP_FILE}"
        log "  Taille : ${SIZE}"
        
        # Création d'un lien symbolique vers le dernier backup
        ln -sf "$(basename ${DUMP_FILE})" "${BACKUP_DIR}/immich_db_latest.sql.gz"
        
        # Informations de contexte
        cat > "${BACKUP_DIR}/backup_${DATE}_info.txt" << EOF
Date du backup: $(date)
Hostname: $(hostname)
Version Immich: $(docker inspect immich_server --format='{{.Config.Image}}' 2>/dev/null || echo "N/A")
DB Username: ${DB_USERNAME}
Fichier dump: $(basename ${DUMP_FILE})
EOF
        
    else
        error "Le fichier dump est vide !"
        rm -f "${DUMP_FILE}"
        exit 1
    fi
else
    error "Échec du dump de la base de données"
    exit 1
fi

# Nettoyage des anciens backups
log "Nettoyage des anciens backups (conservation des ${KEEP_BACKUPS} derniers)..."
cd "${BACKUP_DIR}"
ls -t immich_db_*.sql.gz 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f
ls -t backup_*_info.txt 2>/dev/null | tail -n +$((KEEP_BACKUPS + 1)) | xargs -r rm -f

log "✓ Nettoyage terminé"
log "===== Backup terminé avec succès ====="
log ""
log "Duplicati va synchroniser ce dossier : ${BACKUP_DIR}"
```

!!! tip "Configuration"
    Modifiez la variable `BACKUP_DIR` pour pointer vers le dossier qui sera synchronisé par Duplicati.

Rendez le script exécutable :

```bash
sudo chmod +x /usr/local/bin/immich-backup.sh
```

### Alias Fish Shell

=== "Configuration Fish"

    Éditez votre configuration Fish :
    
    ```bash
    kate ~/.config/fish/config.fish
    ```
    
    Ajoutez ces alias à la fin du fichier :
    
    ```fish title="~/.config/fish/config.fish"
    # Alias Immich
    alias backup-immich='sudo /usr/local/bin/immich-backup.sh'
    alias restore-immich='sudo /usr/local/bin/immich-restore.sh'
    ```
    
    Rechargez la configuration :
    
    ```bash
    source ~/.config/fish/config.fish
    ```

=== "Configuration Bash/Zsh"

    Éditez votre `~/.bashrc` ou `~/.zshrc` :
    
    ```bash
    # Alias Immich
    alias backup-immich='sudo /usr/local/bin/immich-backup.sh'
    alias restore-immich='sudo /usr/local/bin/immich-restore.sh'
    ```
    
    Rechargez la configuration :
    
    ```bash
    source ~/.bashrc  # ou source ~/.zshrc
    ```

### Automatisation avec systemd

=== "Service systemd"

    Créez le fichier de service :
    
    ```bash
    sudo kate /etc/systemd/system/immich-backup.service
    ```
    
    Contenu :
    
    ```ini title="/etc/systemd/system/immich-backup.service"
    [Unit]
    Description=Backup Immich Database
    After=docker.service
    Requires=docker.service
    
    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/immich-backup.sh
    User=root
    StandardOutput=journal
    StandardError=journal
    ```

=== "Timer systemd"

    Créez le timer :
    
    ```bash
    sudo kate /etc/systemd/system/immich-backup.timer
    ```
    
    Contenu :
    
    ```ini title="/etc/systemd/system/immich-backup.timer"
    [Unit]
    Description=Backup Immich Database Daily
    Requires=immich-backup.service
    
    [Timer]
    OnCalendar=daily
    OnCalendar=02:00
    Persistent=true
    
    [Install]
    WantedBy=timers.target
    ```

=== "Alternative : Cron"

    Si vous préférez utiliser cron :
    
    ```bash
    sudo crontab -e
    ```
    
    Ajoutez :
    
    ```cron
    0 2 * * * /usr/local/bin/immich-backup.sh >> /var/log/immich-backup.log 2>&1
    ```

Activez le timer systemd :

```bash
sudo systemctl daemon-reload
sudo systemctl enable immich-backup.timer
sudo systemctl start immich-backup.timer
```

Vérifiez le statut :

```bash
# Statut du timer
sudo systemctl status immich-backup.timer

# Liste des timers actifs
sudo systemctl list-timers --all | grep immich
```

### Configuration Duplicati

Créez un second job Duplicati :

!!! info "Job Duplicati - Base de données"
    - **Source** : `/path/to/backup/immich_db/`
    - **Destination** : OneDrive
    - **Planification** : Quotidienne à 03:00 (après le script à 02:00)
    - **Rétention** : Garder les 30 derniers jours

## Script de restauration

Créez le script de restauration :

```bash
sudo kate /usr/local/bin/immich-restore.sh
```

Copiez le contenu suivant :

??? note "Script de restauration complet (cliquez pour déplier)"
    ```bash title="/usr/local/bin/immich-restore.sh" linenums="1"
    #!/usr/bin/env bash
    
    # Script de restauration Immich avec Duplicati
    # ATTENTION : À exécuter UNIQUEMENT sur une installation fraîche !
    
    # ==============================
    # CONFIGURATION À ADAPTER
    # ==============================
    BACKUP_DB_PATH="/path/to/restored/immich_db"  # Où Duplicati a restauré les dumps DB
    RESTORED_FILES_PATH="/path/to/restored/immich_files"  # Où Duplicati a restauré photos/vidéos
    
    # Nouveaux emplacements (peuvent être différents de l'original)
    NEW_UPLOAD_LOCATION="/nouveau/path/immich/upload"  # Nouvelle destination des fichiers
    NEW_DB_DATA_LOCATION="/nouveau/path/immich/postgres"  # Nouvelle destination DB
    COMPOSE_FILE="/path/to/docker-compose.yml"  # Votre docker-compose.yml
    
    DB_USERNAME="postgres"
    
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
    
    # Vérifier que les backups existent
    if [ ! -d "${BACKUP_DB_PATH}" ]; then
        error "Dossier de backup DB introuvable : ${BACKUP_DB_PATH}"
        exit 1
    fi
    
    # Chercher le dernier dump (ou utiliser le lien symbolique latest)
    if [ -L "${BACKUP_DB_PATH}/immich_db_latest.sql.gz" ]; then
        DB_DUMP="${BACKUP_DB_PATH}/immich_db_latest.sql.gz"
    elif [ -f "${BACKUP_DB_PATH}/immich_db_latest.sql.gz" ]; then
        DB_DUMP="${BACKUP_DB_PATH}/immich_db_latest.sql.gz"
    else
        # Trouver le plus récent
        DB_DUMP=$(ls -t "${BACKUP_DB_PATH}"/immich_db_*.sql.gz 2>/dev/null | head -n1)
    fi
    
    if [ -z "${DB_DUMP}" ] || [ ! -f "${DB_DUMP}" ]; then
        error "Aucun dump de base de données trouvé dans ${BACKUP_DB_PATH}"
        exit 1
    fi
    
    log "✓ Dump de base de données trouvé : $(basename ${DB_DUMP})"
    
    if [ ! -d "${RESTORED_FILES_PATH}" ]; then
        error "Dossier des fichiers restaurés introuvable : ${RESTORED_FILES_PATH}"
        exit 1
    fi
    
    log "✓ Fichiers restaurés trouvés : ${RESTORED_FILES_PATH}"
    
    # ==============================
    # CONFIRMATION UTILISATEUR
    # ==============================
    echo ""
    warning "╔════════════════════════════════════════════════════════════╗"
    warning "║  ATTENTION : RESTAURATION COMPLÈTE D'IMMICH                ║"
    warning "╚════════════════════════════════════════════════════════════╝"
    echo ""
    info "Cette opération va :"
    info "  1. Supprimer toute installation Immich existante"
    info "  2. Créer une installation fraîche"
    info "  3. Restaurer la base de données depuis : $(basename ${DB_DUMP})"
    info "  4. Copier les fichiers depuis : ${RESTORED_FILES_PATH}"
    info "  5. Utiliser les nouveaux emplacements :"
    info "     - Fichiers : ${NEW_UPLOAD_LOCATION}"
    info "     - Base de données : ${NEW_DB_DATA_LOCATION}"
    echo ""
    read -p "Tapez 'OUI' en majuscules pour continuer : " confirm
    
    if [ "$confirm" != "OUI" ]; then
        log "Restauration annulée."
        exit 0
    fi
    
    # ==============================
    # ÉTAPE 1 : NETTOYAGE COMPLET
    # ==============================
    log ""
    log "===== ÉTAPE 1/6 : Nettoyage de l'installation existante ====="
    
    cd "$(dirname ${COMPOSE_FILE})"
    
    if docker compose ps | grep -q immich; then
        log "Arrêt et suppression des conteneurs..."
        docker compose down -v
        log "✓ Conteneurs arrêtés"
    fi
    
    # Suppression des données si elles existent
    if [ -d "${NEW_DB_DATA_LOCATION}" ]; then
        warning "Suppression de la base de données existante : ${NEW_DB_DATA_LOCATION}"
        rm -rf "${NEW_DB_DATA_LOCATION}"
    fi
    
    if [ -d "${NEW_UPLOAD_LOCATION}" ]; then
        warning "Suppression des fichiers existants : ${NEW_UPLOAD_LOCATION}"
        rm -rf "${NEW_UPLOAD_LOCATION}"
    fi
    
    log "✓ Nettoyage terminé"
    
    # ==============================
    # ÉTAPE 2 : RESTAURATION DES FICHIERS
    # ==============================
    log ""
    log "===== ÉTAPE 2/6 : Copie des fichiers restaurés ====="
    
    mkdir -p "${NEW_UPLOAD_LOCATION}"
    
    info "Copie en cours (cela peut prendre du temps)..."
    rsync -ah --info=progress2 "${RESTORED_FILES_PATH}/" "${NEW_UPLOAD_LOCATION}/"
    
    if [ $? -eq 0 ]; then
        log "✓ Fichiers copiés vers ${NEW_UPLOAD_LOCATION}"
    else
        error "Échec de la copie des fichiers"
        exit 1
    fi
    
    # ==============================
    # ÉTAPE 3 : CRÉATION DES CONTENEURS
    # ==============================
    log ""
    log "===== ÉTAPE 3/6 : Création des conteneurs Docker ====="
    
    info "Mise à jour des images (optionnel)..."
    docker compose pull
    
    log "Création des conteneurs (sans les démarrer)..."
    docker compose create
    
    if [ $? -eq 0 ]; then
        log "✓ Conteneurs créés"
    else
        error "Échec de la création des conteneurs"
        exit 1
    fi
    
    # ==============================
    # ÉTAPE 4 : DÉMARRAGE POSTGRESQL UNIQUEMENT
    # ==============================
    log ""
    log "===== ÉTAPE 4/6 : Démarrage de PostgreSQL ====="
    
    docker start immich_postgres
    
    log "Attente du démarrage de PostgreSQL..."
    sleep 10
    
    # Vérification que Postgres est prêt
    for i in {1..20}; do
        if docker exec immich_postgres pg_isready -U "${DB_USERNAME}" > /dev/null 2>&1; then
            log "✓ PostgreSQL est prêt (tentative $i)"
            break
        fi
        if [ $i -eq 20 ]; then
            error "PostgreSQL ne répond pas après 20 tentatives"
            exit 1
        fi
        sleep 2
    done
    
    # ==============================
    # ÉTAPE 5 : RESTAURATION DE LA BASE DE DONNÉES
    # ==============================
    log ""
    log "===== ÉTAPE 5/6 : Restauration de la base de données ====="
    
    info "Restauration en cours (cela peut prendre quelques minutes)..."
    
    gunzip --stdout "${DB_DUMP}" \
    | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
    | docker exec -i immich_postgres psql --dbname=postgres --username="${DB_USERNAME}" 2>&1 | grep -v "^$"
    
    if [ ${PIPESTATUS[0]} -eq 0 ] && [ ${PIPESTATUS[2]} -eq 0 ]; then
        log "✓ Base de données restaurée avec succès"
    else
        error "Échec de la restauration de la base de données"
        error "Vérifiez les logs ci-dessus pour plus de détails"
        exit 1
    fi
    
    # ==============================
    # ÉTAPE 6 : DÉMARRAGE COMPLET D'IMMICH
    # ==============================
    log ""
    log "===== ÉTAPE 6/6 : Démarrage de tous les services Immich ====="
    
    docker compose up -d
    
    log "Attente du démarrage complet des services (30 secondes)..."
    sleep 30
    
    # Vérification que les conteneurs sont lancés
    RUNNING=$(docker compose ps --status running | grep immich | wc -l)
    if [ $RUNNING -ge 3 ]; then
        log "✓ Services Immich démarrés ($RUNNING conteneurs actifs)"
    else
        warning "Seulement $RUNNING conteneurs actifs, vérifiez les logs"
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
    echo "  1. Accédez à l'interface web Immich"
    echo "     (généralement http://localhost:2283)"
    echo ""
    echo "  2. Connectez-vous avec vos identifiants"
    echo ""
    echo "  3. Vérifiez que vos photos et vidéos sont visibles"
    echo ""
    echo "  4. Si les miniatures sont manquantes, allez dans :"
    echo "     Administration → Jobs Status"
    echo "     Puis lancez manuellement :"
    echo "     - Thumbnail Generation Job"
    echo "     - Video Transcoding Job (si nécessaire)"
    echo ""
    info "Emplacements finaux :"
    info "  - Fichiers : ${NEW_UPLOAD_LOCATION}"
    info "  - Base de données : ${NEW_DB_DATA_LOCATION}"
    echo ""
    log "Restauration complète !"
    ```

Rendez le script exécutable :

```bash
sudo chmod +x /usr/local/bin/immich-restore.sh
```

## Procédure de restauration

!!! danger "Ordre de restauration CRITIQUE"
    Ne jamais démarrer Immich avant d'avoir restauré la base de données !

### Étapes de restauration

1. **Restaurer les fichiers avec Duplicati**
   
    Restaurez vos photos/vidéos depuis OneDrive vers leur nouvel emplacement

2. **Restaurer le dossier de backup DB avec Duplicati**
   
    Restaurez le dossier contenant les dumps de base de données

3. **Adapter le script de restauration**
   
    Éditez `/usr/local/bin/immich-restore.sh` et modifiez :
    
    - `BACKUP_DB_PATH` : où Duplicati a restauré les dumps
    - `RESTORED_FILES_PATH` : où Duplicati a restauré les photos/vidéos
    - `NEW_UPLOAD_LOCATION` : destination finale des fichiers
    - `NEW_DB_DATA_LOCATION` : destination finale de la DB
    - `COMPOSE_FILE` : chemin vers votre docker-compose.yml

4. **Lancer la restauration**
   
    ```bash
    restore-immich
    ```

5. **Vérifications post-restauration**
   
    - Connectez-vous à l'interface web
    - Vérifiez que vos photos/vidéos sont visibles
    - Si nécessaire, relancez les jobs de génération de miniatures

### Changement de disque/emplacement

!!! success "Flexibilité totale"
    Le script de restauration permet de restaurer vers un emplacement complètement différent (nouveau disque, NAS, etc.). Il suffit d'adapter les variables `NEW_UPLOAD_LOCATION` et `NEW_DB_DATA_LOCATION`.

Les chemins dans la base de données sont relatifs au conteneur Docker, donc aucun problème lors d'un changement d'emplacement physique.

## Commandes utiles

### Backup manuel

```bash
# Lancer un backup immédiatement
backup-immich

# Ou via systemd
sudo systemctl start immich-backup.service
```

### Consultation des logs

=== "Systemd"

    ```bash
    # Logs du dernier backup
    sudo journalctl -u immich-backup.service -n 50
    
    # Suivre les logs en temps réel
    sudo journalctl -u immich-backup.service -f
    
    # Logs depuis hier
    sudo journalctl -u immich-backup.service --since yesterday
    ```

=== "Cron"

    ```bash
    # Voir les logs (si configuré dans cron)
    sudo tail -f /var/log/immich-backup.log
    ```

### Gestion du timer

```bash
# Statut du timer
sudo systemctl status immich-backup.timer

# Liste de tous les timers
sudo systemctl list-timers --all

# Désactiver temporairement
sudo systemctl stop immich-backup.timer

# Réactiver
sudo systemctl start immich-backup.timer
```

### Test de restauration

```bash
# Lancer la restauration (interactive)
restore-immich
```

## Dépannage

### Le backup ne se lance pas

Vérifiez que :

1. Docker est démarré : `sudo systemctl status docker`
2. Le conteneur PostgreSQL est actif : `docker ps | grep postgres`
3. Le timer est actif : `sudo systemctl status immich-backup.timer`

### La restauration échoue

!!! warning "Erreurs communes"
    - **Base de données déjà existante** : Supprimez `DB_DATA_LOCATION` avant de restaurer
    - **Immich déjà lancé** : Arrêtez tous les conteneurs avec `docker compose down -v`
    - **Fichiers dump corrompus** : Vérifiez l'intégrité du fichier `.sql.gz`

### Vérifier l'intégrité d'un dump

```bash
# Tester si le fichier peut être décompressé
gunzip -t /path/to/backup/immich_db_latest.sql.gz

# Voir les premières lignes
gunzip -c /path/to/backup/immich_db_latest.sql.gz | head -n 20
```

## Stratégie 3-2-1

!!! info "Recommandation"
    Pour une sécurité maximale, suivez la règle 3-2-1 :
    
    - **3** copies de vos données
    - Sur **2** supports différents (disque local + OneDrive)
    - Dont **1** copie hors site (OneDrive)

Configuration actuelle :

- ✅ Copie 1 : Données originales sur le serveur
- ✅ Copie 2 : Backup local (dumps DB + Duplicati cache)
- ✅ Copie 3 : OneDrive (via Duplicati)
- ✅ 2 supports : SSD local + Cloud
- ✅ 1 hors site : OneDrive

## Références

- [Documentation officielle Immich - Backup](https://immich.app/docs/administration/backup-and-restore)
- [PostgreSQL Backup Documentation](https://www.postgresql.org/docs/current/backup.html)
- [Stratégie 3-2-1 Backblaze](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)