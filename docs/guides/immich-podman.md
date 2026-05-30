# Immich avec Podman

!!! info "À qui s'adresse ce guide ?"
    Ce guide est écrit pour quelqu'un qui n'est **pas informaticien**. Chaque étape est expliquée simplement, et tout se fait par **copier-coller**. Une fois l'installation terminée, tu n'auras plus jamais besoin d'ouvrir un terminal.

!!! note "Qu'est-ce qu'Immich ?"
    Immich est une application pour gérer tes photos et vidéos, un peu comme Google Photos mais **chez toi**, sur ton propre ordinateur. Tes photos restent privées, et tu peux y accéder depuis ton téléphone.

---

## Prérequis — Vérifier que Podman est installé

Dans Konsole :

```bash
podman --version
```

Si tu vois un numéro de version (ex. `podman version 5.x.x`), c'est bon, passe à l'Étape 1.

Si tu obtiens une erreur, installe Podman selon ta distribution :

=== "Fedora / Silverblue / Kinoite"

    Podman est installé par défaut. Si besoin :
    ```bash
    rpm-ostree install podman
    ```
    Puis redémarre l'ordinateur.

=== "Arch / CachyOS / Manjaro"

    ```bash
    sudo pacman -S podman
    ```

=== "Debian / Ubuntu / Mint"

    ```bash
    sudo apt install podman
    ```

---

## Étape 1 — Créer les dossiers nécessaires

Ouvre **Konsole** et copie-colle ces lignes une par une :

```bash
mkdir -p ~/.config/containers/systemd
```

```bash
mkdir -p ~/.local/share/immich/db
mkdir -p ~/.local/share/immich/data
mkdir -p ~/.local/share/immich/model-cache
```

!!! tip "Copier-coller dans Konsole"
    Pour coller dans Konsole, utilise **Ctrl+Shift+V** (pas juste Ctrl+V).

---

## Étape 2 — Générer un mot de passe pour la base de données

Immich a besoin d'un mot de passe interne pour sa base de données. Génères-en un automatiquement :

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 32
```

Une suite de caractères aléatoires s'affiche, par exemple : `mK9xQz3pLwR8nVtYcDjF2sHbGu5eAi7N`

!!! warning "Note ce mot de passe !"
    Copie-le dans un endroit sûr (gestionnaire de mots de passe, bloc-notes...). Tu en auras besoin à l'étape suivante. Si tu le perds, il faudra réinstaller Immich.

---

## Étape 3 — Créer les fichiers de configuration

On va créer plusieurs fichiers. Ouvre Kate pour chacun.

### Le pod (conteneur principal)

```bash
kate ~/.config/containers/systemd/immich.pod
```

Copie-colle **tout le contenu suivant** :

```ini
[Unit]
Description=Immich Photo Server
After=network-online.target

[Pod]
PublishPort=2283:2283

[Install]
WantedBy=default.target
```

Sauvegarde avec **Ctrl+S** et ferme Kate.

---

### La base de données

```bash
kate ~/.config/containers/systemd/immich-db.container
```

Copie-colle tout le contenu suivant, **en remplaçant `MON_MOT_DE_PASSE`** par le mot de passe généré à l'étape 2 :

```ini
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

Sauvegarde et ferme Kate.

---

### Redis (cache interne)

```bash
kate ~/.config/containers/systemd/immich-redis.container
```

```ini
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

Sauvegarde et ferme Kate.

---

### Le serveur Immich

```bash
kate ~/.config/containers/systemd/immich-server.container
```

Copie-colle tout le contenu, **en remplaçant `MON_MOT_DE_PASSE`** par le même mot de passe qu'à l'étape 2 :

```ini
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

Sauvegarde et ferme Kate.

---

### Le machine learning (reconnaissance des visages et objets)

```bash
kate ~/.config/containers/systemd/immich-ml.container
```

```ini
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

Sauvegarde et ferme Kate.

---

## Étape 4 — Activer et démarrer Immich

Dans **Konsole**, copie-colle ces commandes une par une :

```bash
systemctl --user daemon-reload
```

```bash
systemctl --user enable --now immich-pod.service immich-db.service immich-redis.service immich-server.service immich-ml.service
```

```bash
loginctl enable-linger $USER
```

!!! info "Le premier démarrage est lent"
    Immich doit télécharger 4 images (serveur, base de données, redis, machine learning). Selon ta connexion, ça peut prendre **5 à 15 minutes**. C'est normal.

!!! success "Vérification"
    ```bash
    systemctl --user status immich-server.service
    ```
    Au bout de quelques minutes, tu dois voir `active (running)` en vert.

---

## Étape 5 — Ouvrir le pare-feu

=== "firewall-cmd (Fedora, etc.)"

    ```bash
    sudo firewall-cmd --add-port=2283/tcp --permanent
    sudo firewall-cmd --reload
    ```

=== "ufw (CachyOS, Ubuntu, Debian, etc.)"

    ```bash
    sudo ufw allow 2283/tcp
    ```

!!! note "Mot de passe invisible"
    Quand tu tapes ton mot de passe après `sudo`, rien ne s'affiche — c'est normal.

---

## Étape 6 — Premier accès

Ouvre **Firefox** et tape dans la barre d'adresse :

```
http://localhost:2283
```

!!! tip "Mets ça en favori !"

Au premier lancement, crée un **compte administrateur** avec le nom et mot de passe de ton choix.

Une fois connecté, tu peux commencer à importer tes photos. Immich va les scanner et créer des miniatures — selon le nombre de photos, ça peut prendre un moment.

**C'est tout ! Tu n'as plus besoin du terminal.** 📷

---

## Accès depuis ton téléphone

Trouve d'abord l'adresse IP de ton ordinateur :

```bash
ip a
```

Cherche une ligne `inet` avec une adresse du style `192.168.1.xx`.

Puis **installe l'application Immich** sur ton téléphone :

- **Android** : [Play Store — Immich](https://play.google.com/store/apps/details?id=app.alextran.immich)
- **iOS** : [App Store — Immich](https://apps.apple.com/app/immich/id1613945652)

Dans l'application, entre l'adresse du serveur : `http://192.168.1.xx:2283`

---

## Mises à jour automatiques

Comme pour Navidrome, les mises à jour se font toutes seules grâce au label `AutoUpdate=registry` dans chaque fichier de configuration. Active le timer une bonne fois pour toutes :

```bash
systemctl --user enable --now podman-auto-update.timer
```

C'est tout — Podman vérifie chaque nuit et met à jour automatiquement.

---

## En cas de problème

!!! question "Immich ne démarre pas ?"
    ```bash
    systemctl --user status immich-server.service
    podman logs immich-immich-server
    ```

!!! question "Je ne peux pas accéder depuis mon téléphone ?"
    Vérifie l'Étape 5 (pare-feu) et que le téléphone est sur le même réseau Wi-Fi.

!!! question "Les photos n'apparaissent pas ?"
    Dans l'interface web : **Administration → Jobs → Library → Run**. Le scan peut prendre plusieurs minutes selon le nombre de photos.

!!! question "Comment redémarrer Immich ?"
    ```bash
    systemctl --user restart immich-server.service
    ```