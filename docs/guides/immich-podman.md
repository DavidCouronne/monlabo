# Immich avec Podman (Rootless)

!!! info "Alternative moderne"
    Ce guide présente une installation Immich avec **Podman en mode rootless**, une alternative plus sécurisée et moderne à Docker.

## Pourquoi Podman ?

**Avantages de Podman rootless** :

- ✅ **Sécurité renforcée** : Pas besoin de privilèges root
- ✅ **Architecture pod** : Gestion native des groupes de conteneurs (comme Kubernetes)
- ✅ **Pas de daemon** : Moins de ressources, plus de stabilité
- ✅ **Compatible Kubernetes** : Fichiers YAML réutilisables
- ✅ **Systemd natif** : Intégration parfaite avec le système

## Prérequis

=== "CachyOS / Arch Linux"

    ```bash
    # Installation Podman
    sudo pacman -S podman
    
    # Vérifier l'installation
    podman --version
    
    # Vérifier le mode rootless
    podman info | grep rootless
    # Devrait afficher : rootless: true
    ```

=== "Autres distributions"

    ```bash
    # Ubuntu/Debian
    sudo apt install podman
    
    # Fedora (pré-installé)
    sudo dnf install podman
    
    # Vérifier
    podman info | grep rootless
    ```

!!! warning "Ne PAS utiliser sudo"
    Toutes les commandes Podman doivent être exécutées en tant qu'utilisateur normal, **SANS sudo**. C'est le principe du mode rootless.

## Installation

### Script d'installation automatique



```bash
# Créer le dossier de scripts
mkdir -p ~/scripts
```

```bash
# Ouvrir Kate
kate ~/scripts/install-immich.sh
```

**Copier le script**


??? note "Script d'installation complet (cliquez pour voir)"
    ```bash title="install-immich.sh" linenums="1"
    #!/bin/bash
    
    # Installation Immich avec Podman Rootless
    # Compatible CachyOS et autres distributions Arch
    
    set -e
    
    GREEN='\033[0;32m'
    YELLOW='\033[1;33m'
    BLUE='\033[0;34m'
    NC='\033[0m'
    
    echo -e "${BLUE}╔════════════════════════════════════════╗${NC}"
    echo -e "${BLUE}║   Installation Immich - Podman         ║${NC}"
    echo -e "${BLUE}║   Mode Rootless                        ║${NC}"
    echo -e "${BLUE}╚════════════════════════════════════════╝${NC}"
    echo ""
    
    # Vérifier qu'on n'est pas root
    if [ "$EUID" -eq 0 ]; then 
        echo -e "${YELLOW}❌ Ne lancez PAS ce script avec sudo !${NC}"
        exit 1
    fi
    
    # Vérifier Podman
    if ! command -v podman &> /dev/null; then
        echo -e "${YELLOW}❌ Podman n'est pas installé${NC}"
        echo "Installation: sudo pacman -S podman"
        exit 1
    fi
    
    # Vérifier mode rootless
    if ! podman info | grep -q "rootless: true"; then
        echo -e "${YELLOW}❌ Podman n'est pas en mode rootless${NC}"
        exit 1
    fi
    
    echo -e "${GREEN}✓ Podman rootless détecté${NC}"
    echo -e "${GREEN}✓ Backend réseau: $(podman info --format '{{.Host.NetworkBackend}}')${NC}"
    echo ""
    
    # Créer la structure
    echo "Création de la structure..."
    mkdir -p ~/containers/immich/{db,data,model-cache}
    echo -e "${GREEN}✓ Dossiers créés${NC}"
    
    # Générer un mot de passe aléatoire
    DB_PASSWORD=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-25)
    echo ""
    echo -e "${YELLOW}Mot de passe généré: ${DB_PASSWORD}${NC}"
    echo -e "${YELLOW}Sauvegardez-le dans un gestionnaire de mots de passe !${NC}"
    echo ""
    
    # Créer le fichier YAML
    echo "Création du fichier de configuration..."
    cat > ~/containers/immich/immich-pod.yaml << 'EOF'
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: immich-config
    data:
      DB_HOSTNAME: "localhost"
      DB_USERNAME: "immich"
      DB_DATABASE_NAME: "immich"
      REDIS_HOSTNAME: "localhost"
      
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: immich-secrets
    type: Opaque
    stringData:
      DB_PASSWORD: "DB_PASSWORD_PLACEHOLDER"
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: immich
      labels:
        app: immich
    spec:
      restartPolicy: Always
      
      containers:
      
      - name: redis
        image: docker.io/valkey/valkey:8@sha256:81db6d39e1bba3b3ff32bd3a1b19a6d69690f94a3954ec131277b9a26b95b3aa
        resources:
          requests:
            memory: "128Mi"
          limits:
            memory: "256Mi"
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 10
          periodSeconds: 5
      
      - name: database
        image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
        env:
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: immich-config
              key: DB_USERNAME
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: immich-config
              key: DB_DATABASE_NAME
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: immich-secrets
              key: DB_PASSWORD
        - name: POSTGRES_INITDB_ARGS
          value: "--data-checksums"
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "1Gi"
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
        - name: shm
          mountPath: /dev/shm
      
      - name: server
        image: ghcr.io/immich-app/immich-server:release
        ports:
        - containerPort: 2283
          hostPort: 2283
        envFrom:
        - configMapRef:
            name: immich-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: immich-secrets
              key: DB_PASSWORD
        resources:
          requests:
            memory: "512Mi"
          limits:
            memory: "2Gi"
        volumeMounts:
        - name: upload-data
          mountPath: /data
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
      
      - name: machine-learning
        image: ghcr.io/immich-app/immich-machine-learning:release
        envFrom:
        - configMapRef:
            name: immich-config
        resources:
          requests:
            memory: "512Mi"
          limits:
            memory: "4Gi"
        volumeMounts:
        - name: model-cache
          mountPath: /cache
      
      volumes:
      - name: db-data
        hostPath:
          path: containers/immich/db
          type: DirectoryOrCreate
      - name: upload-data
        hostPath:
          path: containers/immich/data
          type: DirectoryOrCreate
      - name: model-cache
        hostPath:
          path: containers/immich/model-cache
          type: DirectoryOrCreate
      - name: localtime
        hostPath:
          path: /etc/localtime
          type: File
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 128Mi
    EOF
    
    # Remplacer le mot de passe dans le YAML
    sed -i "s/DB_PASSWORD_PLACEHOLDER/${DB_PASSWORD}/" ~/containers/immich/immich-pod.yaml
    
    echo -e "${GREEN}✓ Configuration créée${NC}"
    
    # Sauvegarder le mot de passe
    echo "$DB_PASSWORD" > ~/containers/immich/.db_password
    chmod 600 ~/containers/immich/.db_password
    echo -e "${GREEN}✓ Mot de passe sauvegardé dans ~/containers/immich/.db_password${NC}"
    
    # Déployer
    echo ""
    echo "Déploiement du pod Immich..."
    podman play kube ~/containers/immich/immich-pod.yaml
    
    echo ""
    echo "Attente du démarrage des services..."
    echo "(Cela peut prendre 1-2 minutes)"
    echo ""
    
    # Attendre que le serveur soit prêt
    for i in {1..60}; do
        if curl -s http://localhost:2283/api/server-info/ping > /dev/null 2>&1; then
            echo -e "${GREEN}✓ Serveur démarré !${NC}"
            break
        fi
        echo -n "."
        sleep 2
    done
    echo ""
    
    # Créer le service systemd
    echo ""
    echo "Configuration du service systemd..."
    mkdir -p ~/.config/systemd/user/
    
    cat > ~/.config/systemd/user/immich.service << 'SYSTEMD_EOF'
    [Unit]
    Description=Immich Photo Management
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/bin/podman play kube %h/containers/immich/immich-pod.yaml
    ExecStop=/usr/bin/podman play kube --down %h/containers/immich/immich-pod.yaml
    Restart=on-failure
    RestartSec=30
    
    [Install]
    WantedBy=default.target
    SYSTEMD_EOF
    
    systemctl --user daemon-reload
    systemctl --user enable immich.service
    loginctl enable-linger $USER
    
    echo -e "${GREEN}✓ Service systemd configuré et activé${NC}"
    
    # Afficher l'état
    echo ""
    echo -e "${BLUE}════════════════════════════════════════${NC}"
    echo -e "${GREEN}✓ Installation terminée !${NC}"
    echo -e "${BLUE}════════════════════════════════════════${NC}"
    echo ""
    echo "Pod Immich:"
    podman pod ps
    echo ""
    echo "Conteneurs:"
    podman ps --pod --filter pod=immich
    echo ""
    echo -e "${GREEN}Immich est accessible sur: http://localhost:2283${NC}"
    echo ""
    echo "Mot de passe DB sauvegardé dans: ~/containers/immich/.db_password"
    echo ""
    echo "Commandes utiles:"
    echo "  podman pod ps                    # État du pod"
    echo "  podman logs -f immich-server     # Logs du serveur"
    echo "  systemctl --user status immich   # État du service"
    echo "  systemctl --user restart immich  # Redémarrer"
    echo ""
    echo "Le service démarrera automatiquement au boot."
    echo ""
    ```

```bash
# Rendre exécutable
chmod +x ~/scripts/install-immich.sh
```

```bash
# Lancer l'installation
~/scripts/install-immich.sh
```

### Configuration manuelle

Si vous préférez comprendre chaque étape :



=== "Étape 1 : Structure"

    ```bash
    # Créer les dossiers
    mkdir -p ~/containers/immich/{db,data,model-cache}
    ```

=== "Étape 2 : Configuration"

    Créez le fichier `~/containers/immich/immich-pod.yaml` avec le contenu disponible dans le script ci-dessous.
    
    Modifiez la ligne `DB_PASSWORD` avec un mot de passe sécurisé.

    ??? note "`~/containers/immich/immich-pod.yaml`"
        
            ---
            apiVersion: v1
            kind: ConfigMap
            metadata:
            name: immich-config
            data:
            DB_HOSTNAME: "localhost"
            DB_USERNAME: "immich"
            DB_DATABASE_NAME: "immich"
            REDIS_HOSTNAME: "localhost"
            
            ---
            apiVersion: v1
            kind: Secret
            metadata:
            name: immich-secrets
            type: Opaque
            stringData:
            DB_PASSWORD: "DB_PASSWORD_PLACEHOLDER"
            
            ---
            apiVersion: v1
            kind: Pod
            metadata:
            name: immich
            labels:
                app: immich
            spec:
            restartPolicy: Always
            
            containers:
            
            - name: redis
                image: docker.io/valkey/valkey:8@sha256:81db6d39e1bba3b3ff32bd3a1b19a6d69690f94a3954ec131277b9a26b95b3aa
                resources:
                requests:
                    memory: "128Mi"
                limits:
                    memory: "256Mi"
                livenessProbe:
                exec:
                    command:
                    - redis-cli
                    - ping
                initialDelaySeconds: 10
                periodSeconds: 5
            
            - name: database
                image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
                env:
                - name: POSTGRES_USER
                valueFrom:
                    configMapKeyRef:
                    name: immich-config
                    key: DB_USERNAME
                - name: POSTGRES_DB
                valueFrom:
                    configMapKeyRef:
                    name: immich-config
                    key: DB_DATABASE_NAME
                - name: POSTGRES_PASSWORD
                valueFrom:
                    secretKeyRef:
                    name: immich-secrets
                    key: DB_PASSWORD
                - name: POSTGRES_INITDB_ARGS
                value: "--data-checksums"
                resources:
                requests:
                    memory: "256Mi"
                limits:
                    memory: "1Gi"
                volumeMounts:
                - name: db-data
                mountPath: /var/lib/postgresql/data
                - name: shm
                mountPath: /dev/shm
            
            - name: server
                image: ghcr.io/immich-app/immich-server:release
                ports:
                - containerPort: 2283
                hostPort: 2283
                envFrom:
                - configMapRef:
                    name: immich-config
                env:
                - name: DB_PASSWORD
                valueFrom:
                    secretKeyRef:
                    name: immich-secrets
                    key: DB_PASSWORD
                resources:
                requests:
                    memory: "512Mi"
                limits:
                    memory: "2Gi"
                volumeMounts:
                - name: upload-data
                mountPath: /data
                - name: localtime
                mountPath: /etc/localtime
                readOnly: true
            
            - name: machine-learning
                image: ghcr.io/immich-app/immich-machine-learning:release
                envFrom:
                - configMapRef:
                    name: immich-config
                resources:
                requests:
                    memory: "512Mi"
                limits:
                    memory: "4Gi"
                volumeMounts:
                - name: model-cache
                mountPath: /cache
            
            volumes:
            - name: db-data
                hostPath:
                path: containers/immich/db
                type: DirectoryOrCreate
            - name: upload-data
                hostPath:
                path: containers/immich/data
                type: DirectoryOrCreate
            - name: model-cache
                hostPath:
                path: containers/immich/model-cache
                type: DirectoryOrCreate
            - name: localtime
                hostPath:
                path: /etc/localtime
                type: File
            - name: shm
                emptyDir:
                medium: Memory
                sizeLimit: 128Mi

    

=== "Étape 3 : Déploiement"

    ```bash
    # Déployer le pod
    podman play kube ~/containers/immich/immich-pod.yaml
    
    # Vérifier l'état
    podman pod ps
    podman ps --pod
    ```

=== "Étape 4 : Service systemd"

    ```bash
    # Créer le service
    mkdir -p ~/.config/systemd/user/
    
    cat > ~/.config/systemd/user/immich.service << 'EOF'
    [Unit]
    Description=Immich Photo Management
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/bin/podman play kube %h/containers/immich/immich-pod.yaml
    ExecStop=/usr/bin/podman play kube --down %h/containers/immich/immich-pod.yaml
    Restart=on-failure
    
    [Install]
    WantedBy=default.target
    EOF
    
    # Activer
    systemctl --user daemon-reload
    systemctl --user enable immich.service
    
    # Activer le linger (démarrage même si pas connecté)
    loginctl enable-linger $USER
    ```

## Premier démarrage

1. **Accéder à l'interface** : http://localhost:2283

2. **Créer le compte administrateur**
   
   Premier utilisateur = administrateur automatiquement

3. **Configurer les paramètres**
   
   Administration → Settings → ajustez selon vos besoins

## Backup et Restauration

### Script de backup automatique

Créez le script de backup :

```bash
mkdir -p ~/scripts
kate ~/scripts/backup-immich.sh
```

Contenu du script :

??? note "Script de backup complet (cliquez pour voir)"
    ```bash title="backup-immich.sh" linenums="1"
    #!/bin/bash
    
    # Backup automatique Immich (Podman Rootless)
    
    set -e
    
    # Configuration
    BACKUP_DIR="$HOME/backups/immich"
    KEEP_BACKUPS=14
    
    # Couleurs
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
    
    # Vérifier que le pod est actif
    if ! podman pod exists immich; then
        error "Le pod Immich n'existe pas"
        exit 1
    fi
    
    if ! podman ps | grep -q immich-database; then
        error "Le conteneur database n'est pas actif"
        exit 1
    fi
    
    # Créer le dossier de backup
    mkdir -p "${BACKUP_DIR}"
    
    # Nom du fichier avec timestamp
    DATE=$(date +%Y%m%d_%H%M%S)
    DUMP_FILE="${BACKUP_DIR}/immich_db_${DATE}.sql.gz"
    
    log "===== Backup de la base de données Immich ====="
    log "Création du dump PostgreSQL..."
    
    # Dump de la base
    if podman exec immich-database pg_dump -U immich immich | gzip > "${DUMP_FILE}"; then
        
        if [ -s "${DUMP_FILE}" ]; then
            SIZE=$(du -h "${DUMP_FILE}" | cut -f1)
            log "✓ Backup réussi : ${DUMP_FILE}"
            log "  Taille : ${SIZE}"
            
            # Lien symbolique vers le dernier backup
            ln -sf "$(basename ${DUMP_FILE})" "${BACKUP_DIR}/immich_db_latest.sql.gz"
            
            # Informations de contexte
            cat > "${BACKUP_DIR}/backup_${DATE}_info.txt" << EOF
    Date du backup: $(date)
    Hostname: $(hostname)
    Version Immich Server: $(podman inspect immich-server --format='{{.Config.Image}}' 2>/dev/null || echo "N/A")
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
    ```

Rendre exécutable :

```bash
chmod +x ~/scripts/backup-immich.sh
```

### Automatisation avec systemd

=== "Service"

    ```bash
    kate ~/.config/systemd/user/immich-backup.service
    ```
    
    Contenu :
    
    ```ini title="immich-backup.service"
    [Unit]
    Description=Backup Immich Database
    After=immich.service
    Requires=immich.service
    
    [Service]
    Type=oneshot
    ExecStart=%h/scripts/backup-immich.sh
    StandardOutput=journal
    StandardError=journal
    ```

=== "Timer"

    ```bash
    kate ~/.config/systemd/user/immich-backup.timer
    ```
    
    Contenu :
    
    ```ini title="immich-backup.timer"
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

=== "Activation"

    ```bash
    # Recharger systemd
    systemctl --user daemon-reload
    
    # Activer le timer
    systemctl --user enable immich-backup.timer
    systemctl --user start immich-backup.timer
    
    # Vérifier
    systemctl --user status immich-backup.timer
    systemctl --user list-timers
    ```

### Script de restauration

Créez le script de restauration :

```bash
kate ~/scripts/restore-immich.sh
```

??? note "Script de restauration complet (cliquez pour voir)"
    ```bash title="restore-immich.sh" linenums="1"
    #!/bin/bash
    
    # Restauration Immich (Podman Rootless)
    
    set -e
    
    # Configuration
    BACKUP_DIR="$HOME/backups/immich"
    IMMICH_DIR="$HOME/containers/immich"
    
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
    
    # Trouver le dernier backup
    if [ -L "${BACKUP_DIR}/immich_db_latest.sql.gz" ]; then
        DB_DUMP="${BACKUP_DIR}/immich_db_latest.sql.gz"
    else
        DB_DUMP=$(ls -t "${BACKUP_DIR}"/immich_db_*.sql.gz 2>/dev/null | head -n1)
    fi
    
    if [ -z "${DB_DUMP}" ] || [ ! -f "${DB_DUMP}" ]; then
        error "Aucun backup trouvé dans ${BACKUP_DIR}"
        exit 1
    fi
    
    log "Backup trouvé : $(basename ${DB_DUMP})"
    
    # Confirmation
    echo ""
    warning "╔════════════════════════════════════════════════════════════╗"
    warning "║  ATTENTION : RESTAURATION COMPLÈTE D'IMMICH                ║"
    warning "╚════════════════════════════════════════════════════════════╝"
    echo ""
    info "Cette opération va :"
    info "  1. Arrêter le pod Immich"
    info "  2. Supprimer la base de données actuelle"
    info "  3. Restaurer depuis : $(basename ${DB_DUMP})"
    info "  4. Redémarrer les services"
    echo ""
    read -p "Tapez 'OUI' en majuscules pour continuer : " confirm
    
    if [ "$confirm" != "OUI" ]; then
        log "Restauration annulée."
        exit 0
    fi
    
    # Arrêter le pod
    log ""
    log "===== Arrêt du pod Immich ====="
    podman play kube --down "${IMMICH_DIR}/immich-pod.yaml" || true
    podman pod rm -f immich 2>/dev/null || true
    
    # Supprimer la base de données
    log ""
    log "===== Suppression de la base de données ====="
    rm -rf "${IMMICH_DIR}/db"/*
    
    # Recréer les dossiers
    mkdir -p "${IMMICH_DIR}/db"
    
    # Redéployer
    log ""
    log "===== Redéploiement du pod ====="
    podman play kube "${IMMICH_DIR}/immich-pod.yaml"
    
    log "Attente du démarrage de PostgreSQL (30s)..."
    sleep 30
    
    # Vérifier que PostgreSQL est prêt
    for i in {1..20}; do
        if podman exec immich-database pg_isready -U immich > /dev/null 2>&1; then
            log "✓ PostgreSQL est prêt"
            break
        fi
        if [ $i -eq 20 ]; then
            error "PostgreSQL ne répond pas"
            exit 1
        fi
        sleep 2
    done
    
    # Restaurer le backup
    log ""
    log "===== Restauration de la base de données ====="
    info "Restauration en cours (cela peut prendre quelques minutes)..."
    
    gunzip -c "${DB_DUMP}" | podman exec -i immich-database psql -U immich -d immich
    
    if [ $? -eq 0 ]; then
        log "✓ Base de données restaurée"
    else
        error "Échec de la restauration"
        exit 1
    fi
    
    # Redémarrer le pod complet
    log ""
    log "===== Redémarrage complet ====="
    podman pod restart immich
    
    log "Attente du démarrage complet (30s)..."
    sleep 30
    
    # Vérifier
    RUNNING=$(podman ps --filter pod=immich --filter status=running | grep immich | wc -l)
    if [ $RUNNING -ge 3 ]; then
        log "✓ Services démarrés ($RUNNING conteneurs actifs)"
    else
        warning "Seulement $RUNNING conteneurs actifs"
    fi
    
    log ""
    log "╔════════════════════════════════════════════════════════════╗"
    log "║           RESTAURATION TERMINÉE AVEC SUCCÈS                ║"
    log "╚════════════════════════════════════════════════════════════╝"
    echo ""
    info "Accédez à Immich : http://localhost:2283"
    info "Vérifiez vos photos et albums"
    echo ""
    ```

Rendre exécutable :

```bash
chmod +x ~/scripts/restore-immich.sh
```

### Procédure de restauration

!!! danger "Ordre CRITIQUE"
    Ne jamais démarrer Immich avant d'avoir restauré la base de données !

1. **Arrêter Immich si nécessaire**
   
    ```bash
    podman play kube --down ~/containers/immich/immich-pod.yaml
    ```

2. **Lancer la restauration**
   
    ```bash
    ~/scripts/restore-immich.sh
    ```

3. **Vérifier**
   
    Accédez à http://localhost:2283 et vérifiez vos données

## Alias Fish Shell

=== "Configuration Fish"

    Éditez `~/.config/fish/config.fish` :
    
    ```fish
    # Alias Immich Podman
    alias immich-start='systemctl --user start immich'
    alias immich-stop='systemctl --user stop immich'
    alias immich-restart='systemctl --user restart immich'
    alias immich-status='systemctl --user status immich'
    alias immich-logs='podman logs -f immich-server'
    alias immich-backup='~/scripts/backup-immich.sh'
    alias immich-restore='~/scripts/restore-immich.sh'
    alias immich-ps='podman ps --pod --filter pod=immich'
    ```
    
    Rechargez :
    
    ```bash
    source ~/.config/fish/config.fish
    ```

=== "Configuration Bash/Zsh"

    Éditez `~/.bashrc` ou `~/.zshrc` :
    
    ```bash
    # Alias Immich Podman
    alias immich-start='systemctl --user start immich'
    alias immich-stop='systemctl --user stop immich'
    alias immich-restart='systemctl --user restart immich'
    alias immich-status='systemctl --user status immich'
    alias immich-logs='podman logs -f immich-server'
    alias immich-backup='~/scripts/backup-immich.sh'
    alias immich-restore='~/scripts/restore-immich.sh'
    alias immich-ps='podman ps --pod --filter pod=immich'
    ```

## Commandes utiles

### Gestion du pod

```bash
# État du pod
podman pod ps

# Conteneurs dans le pod
podman ps --pod --filter pod=immich

# Arrêter/démarrer
systemctl --user stop immich
systemctl --user start immich

# Redémarrer
systemctl --user restart immich

# Statut du service
systemctl --user status immich
```

### Logs

```bash
# Logs du serveur
podman logs -f immich-server

# Logs de la base de données
podman logs -f immich-database

# Logs du machine learning
podman logs -f immich-machine-learning

# Logs de tous les conteneurs
podman pod logs -f immich
```

### Maintenance

```bash
# Mettre à jour les images
podman pull ghcr.io/immich-app/immich-server:release
podman pull ghcr.io/immich-app/immich-machine-learning:release

# Redéployer avec les nouvelles images
podman play kube --replace ~/containers/immich/immich-pod.yaml

# Nettoyer les anciennes images
podman image prune -a

# Voir l'utilisation disque
du -sh ~/containers/immich/*

# Statistiques en temps réel
podman stats
```

### Backup manuel

```bash
# Backup immédiat
~/scripts/backup-immich.sh

# Ou via systemd
systemctl --user start immich-backup.service

# Voir les logs du backup
journalctl --user -u immich-backup.service -n 50
```

## Gestion avancée

### Changer le port

Éditez `~/containers/immich/immich-pod.yaml` :

```yaml
# Ligne à modifier (exemple pour le port 8080)
- containerPort: 2283
  hostPort: 8080
```

Puis redéployez :

```bash
podman play kube --replace ~/containers/immich/immich-pod.yaml
```

### Accès à la base de données

```bash
# Shell PostgreSQL
podman exec -it immich-database psql -U immich -d immich

# Commandes SQL utiles
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM assets;
SELECT COUNT(*) FROM albums;
```

### Régénérer les miniatures

Via l'interface web :

1. Administration → Jobs Status
2. Thumbnail Generation Job → Run Job

Ou en ligne de commande :

```bash
podman exec immich-server immich-cli jobs run thumbnails
```

