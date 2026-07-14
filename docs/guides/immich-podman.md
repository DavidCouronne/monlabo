# Déploiement d'Immich (Podman)

Ce guide détaille le déploiement d'Immich en mode conteneurisé rootless avec Podman et des Quadlets Systemd.

---

## Prérequis

Avant de procéder à l'installation, assurez-vous que Podman est installé sur votre système.

Exécutez la commande suivante pour vérifier sa présence :

```bash
podman --version
```

Si Podman n'est pas installé, utilisez les commandes suivantes selon votre distribution :

=== "Arch / CachyOS / Manjaro"

    ```bash
    sudo pacman -S podman
    ```

=== "Fedora / Silverblue / Kinoite"

    Podman est préinstallé par défaut sur ces versions. Si nécessaire, réinstallez-le avec :
    ```bash
    rpm-ostree install podman
    ```
    *Un redémarrage du système est requis après l'installation.*

=== "Debian / Ubuntu / Mint"

    ```bash
    sudo apt install podman
    ```

---

## Étape 1 : Création des répertoires de persistance

Les données et configurations d'Immich doivent être stockées dans des répertoires persistants dans votre dossier utilisateur.

Exécutez ces commandes pour créer l'arborescence requise :

```bash
# Répertoire pour les Quadlets Systemd
mkdir -p ~/.config/containers/systemd

# Répertoires de données d'Immich
mkdir -p ~/.local/share/immich/db
mkdir -p ~/.local/share/immich/data
mkdir -p ~/.local/share/immich/model-cache
```

---

## Étape 2 : Génération du mot de passe de base de données

Immich utilise PostgreSQL et nécessite un mot de passe robuste pour authentifier les services internes.

Générez une chaîne de caractères aléatoire :

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 32
```

> [!WARNING]
> Conservez ce mot de passe dans un endroit sécurisé (tel qu'un gestionnaire de mots de passe). Il sera requis pour configurer les services à l'étape suivante.

---

## Étape 3 : Configuration des Quadlets Systemd

Les Quadlets permettent à Systemd de gérer automatiquement le cycle de vie de vos conteneurs Podman en tant que services utilisateur.

Créez et configurez les cinq fichiers requis.

### 3.1 : Définition du Pod (`immich.pod`)
Ce fichier configure le réseau commun pour l'ensemble des conteneurs d'Immich.

```bash
kate ~/.config/containers/systemd/immich.pod
```

Contenu à y insérer :

```ini title="~/.config/containers/systemd/immich.pod"
[Unit]
Description=Immich Photo Server Pod
After=network-online.target

[Pod]
PublishPort=2283:2283

[Install]
WantedBy=default.target
```

### 3.2 : Base de données (`immich-db.container`)
Ce fichier configure le moteur PostgreSQL avec l'extension vectorielle pgvecto-rs.

```bash
kate ~/.config/containers/systemd/immich-db.container
```

Contenu à y insérer (remplacez `MON_MOT_DE_PASSE` par la valeur générée à l'étape 2) :

```ini title="~/.config/containers/systemd/immich-db.container"
[Unit]
Description=Immich Database
After=immich-pod.service

[Container]
Image=docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0
Pod=immich.pod
AutoUpdate=registry
Volume=%h/.local/share/immich/db:/var/lib/postgresql/data:Z
Environment=POSTGRES_USER=immich
Environment=POSTGRES_DB=immich
Environment=POSTGRES_PASSWORD=MON_MOT_DE_PASSE
Environment=POSTGRES_INITDB_ARGS=--data-checksums

[Service]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

### 3.3 : Cache Redis (`immich-redis.container`)

```bash
kate ~/.config/containers/systemd/immich-redis.container
```

Contenu à y insérer :

```ini title="~/.config/containers/systemd/immich-redis.container"
[Unit]
Description=Immich Redis
After=immich-pod.service

[Container]
Image=docker.io/valkey/valkey:8
Pod=immich.pod
AutoUpdate=registry

[Service]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

### 3.4 : Serveur applicatif (`immich-server.container`)

```bash
kate ~/.config/containers/systemd/immich-server.container
```

Contenu à y insérer (remplacez `MON_MOT_DE_PASSE` par la même valeur qu'à la section 3.2) :

```ini title="~/.config/containers/systemd/immich-server.container"
[Unit]
Description=Immich Server
After=immich-db.service immich-redis.service

[Container]
Image=ghcr.io/immich-app/immich-server:release
Pod=immich.pod
AutoUpdate=registry
Volume=%h/Photos:/usr/src/app/upload/library:ro,Z
Volume=%h/.local/share/immich/data:/usr/src/app/upload:Z
Volume=/etc/localtime:/etc/localtime:ro
Environment=DB_HOSTNAME=localhost
Environment=DB_USERNAME=immich
Environment=DB_DATABASE_NAME=immich
Environment=DB_PASSWORD=MON_MOT_DE_PASSE
Environment=REDIS_HOSTNAME=localhost

[Service]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

### 3.5 : Apprentissage automatique / Machine Learning (`immich-ml.container`)

```bash
kate ~/.config/containers/systemd/immich-ml.container
```

Contenu à y insérer :

```ini title="~/.config/containers/systemd/immich-ml.container"
[Unit]
Description=Immich Machine Learning
After=immich-pod.service

[Container]
Image=ghcr.io/immich-app/immich-machine-learning:release
Pod=immich.pod
AutoUpdate=registry
Volume=%h/.local/share/immich/model-cache:/cache:Z

[Service]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

---

## Étape 4 : Activation et démarrage des services

Rechargez le gestionnaire Systemd de votre session et activez le démarrage des services :

```bash
# Recharger la configuration Systemd
systemctl --user daemon-reload

# Activer et démarrer les unités de service
systemctl --user enable --now immich-pod.service immich-db.service immich-redis.service immich-server.service immich-ml.service

# Permettre l'exécution des services utilisateur après la déconnexion
loginctl enable-linger $USER
```

> [!NOTE]
> Le premier démarrage peut prendre quelques minutes, le temps que Podman télécharge les images associées. Vous pouvez suivre l'état de l'initialisation avec :
> `systemctl --user status immich-server.service`

---

## Étape 5 : Configuration du pare-feu

Pour autoriser l'accès au serveur Immich (port 2283) depuis d'autres périphériques du réseau local, configurez votre pare-feu :

=== "firewall-cmd (Fedora / RHEL)"

    ```bash
    sudo firewall-cmd --add-port=2283/tcp --permanent
    sudo firewall-cmd --reload
    ```

=== "ufw (CachyOS / Ubuntu / Debian)"

    ```bash
    sudo ufw allow 2283/tcp
    ```

---

## Étape 6 : Accès initial et configuration

1. Ouvrez votre navigateur web et accédez à l'adresse suivante :
   `http://localhost:2283`
2. Créez le compte administrateur initial lors de la première connexion.
3. Importez vos médias. Le traitement (génération des miniatures, reconnaissance d'objets) s'effectuera en arrière-plan.

### Accès mobile
Pour connecter l'application mobile (iOS ou Android) :
1. Récupérez l'adresse IP de votre hôte avec la commande `ip a`.
2. Configurez l'application cliente en renseignant l'URL : `http://<IP_DE_L_HOTE>:2283`.

---

## Étape 7 : Mises à jour automatiques

Grâce aux instructions `AutoUpdate=registry` incluses dans les Quadlets, vous pouvez automatiser la mise à jour des conteneurs via le timer natif de Podman :

```bash
systemctl --user enable --now podman-auto-update.timer
```

Les conteneurs seront ainsi mis à jour automatiquement chaque nuit si une nouvelle image est disponible.

---

## Dépannage et commandes utiles

### Vérification des statuts
```bash
systemctl --user status immich-server.service
```

### Lecture des journaux applicatifs
```bash
podman logs immich-server
```

### Redémarrage des services
```bash
systemctl --user restart immich-pod.service
```