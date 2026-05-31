# Jellyfin avec Podman

!!! info "À qui s'adresse ce guide ?"
    Ce guide est écrit pour quelqu'un qui n'est **pas informaticien**. Chaque étape est expliquée simplement, et tout se fait par **copier-coller**. Une fois l'installation terminée, tu n'auras plus jamais besoin d'ouvrir un terminal.

!!! note "Qu'est-ce que Jellyfin ?"
    Jellyfin est un serveur multimédia — un peu comme Netflix, mais **chez toi**, sur ton propre ordinateur. Il te permet de regarder tes films et séries, écouter ta musique, et visionner tes photos depuis n'importe quel appareil sur ton réseau (téléphone, télévision, tablette...).

---

## Prérequis — Vérifier que Podman est installé

Dans **Konsole** :

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
mkdir -p ~/.local/share/jellyfin/config
mkdir -p ~/.local/share/jellyfin/cache
```

!!! tip "Copier-coller dans Konsole"
    Pour coller dans Konsole, utilise **Ctrl+Shift+V** (pas juste Ctrl+V).

---

## Étape 2 — Créer le fichier de configuration

```bash
kate ~/.config/containers/systemd/jellyfin.container
```

Kate va s'ouvrir avec un fichier vide. Copie-colle **tout le contenu suivant** :

```ini
[Unit]
Description=Jellyfin Media Server
After=network-online.target

[Container]
Image=docker.io/jellyfin/jellyfin:latest
AutoUpdate=registry
PublishPort=8096:8096
Volume=%h/.local/share/jellyfin/config:/config:Z
Volume=%h/.local/share/jellyfin/cache:/cache:Z
Volume=%h/Vidéos:/media/videos:ro,Z
Volume=%h/Musique:/media/music:ro,Z
Environment=JELLYFIN_PublishedServerUrl=http://localhost:8096

[Service]
Restart=always
RestartSec=10
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

!!! warning "Vérifie le nom de tes dossiers"
    Les lignes `Volume=` supposent que :

    - Tes **films et séries** sont dans `~/Vidéos`
    - Ta **musique** est dans `~/Musique`

    Si tes dossiers ont des noms différents, modifie ces lignes. Si tu n'as pas de musique à gérer dans Jellyfin (tu as Navidrome pour ça !), supprime la ligne `Volume=%h/Musique...`.

    `%h` est remplacé automatiquement par le chemin de ton dossier personnel.

Sauvegarde avec **Ctrl+S** et ferme Kate.

---

## Étape 3 — Activer et démarrer Jellyfin

Dans **Konsole**, copie-colle ces commandes une par une :

```bash
systemctl --user daemon-reload
```

```bash
systemctl --user enable --now jellyfin.service
```

```bash
loginctl enable-linger $USER
```

!!! info "Le premier démarrage prend quelques secondes"
    Jellyfin doit télécharger son image au premier lancement. Selon ta connexion, ça peut prendre une à deux minutes.

!!! success "Vérification"
    ```bash
    systemctl --user status jellyfin.service
    ```
    Tu dois voir `active (running)` en vert. **C'est la dernière fois que tu utilises le terminal pour Jellyfin.** 🎉

---

## Étape 4 — Ouvrir le pare-feu

=== "firewall-cmd (Fedora, etc.)"

    ```bash
    sudo firewall-cmd --add-port=8096/tcp --permanent
    sudo firewall-cmd --reload
    ```

=== "ufw (CachyOS, Ubuntu, Debian, etc.)"

    ```bash
    sudo ufw allow 8096/tcp
    ```

!!! note "Mot de passe invisible"
    Quand tu tapes ton mot de passe après `sudo`, rien ne s'affiche — c'est normal.

---

## Étape 5 — Configuration initiale

Ouvre **Firefox** et tape dans la barre d'adresse :

```
http://localhost:8096
```

!!! tip "Mets ça en favori !"

Jellyfin te guide à travers une configuration initiale en quelques étapes :

1. **Choisis la langue** — Français
2. **Crée un compte administrateur** — mets un nom et un mot de passe dont tu te souviendras
3. **Ajoute tes bibliothèques** — clique "Ajouter une bibliothèque multimédia" :
    - Type : **Films** → pointe vers `/media/videos`
    - Type : **Musique** → pointe vers `/media/music` (si applicable)
4. **Termine la configuration** et laisse Jellyfin scanner tes fichiers

!!! info "Le scan peut prendre du temps"
    Jellyfin va télécharger automatiquement les affiches, bandes-annonces et informations pour chaque film/série. Selon ta bibliothèque, ça peut prendre plusieurs minutes à quelques heures. Tu peux utiliser Jellyfin pendant ce temps.

**C'est tout ! Tu n'as plus besoin du terminal.** 🎬

---

## Accès depuis d'autres appareils

Pour regarder depuis ton téléphone, tablette ou télévision, trouve d'abord l'adresse IP de ton ordinateur :

```bash
ip a
```

Cherche une ligne `inet` avec une adresse du style `192.168.1.xx`.

### Depuis un navigateur

Tape dans le navigateur : `http://192.168.1.xx:8096`

### Applications recommandées

=== "Android / iOS"

    **Jellyfin Mobile** (application officielle)

    - Android : [Play Store](https://play.google.com/store/apps/details?id=org.jellyfin.mobile)
    - iOS : [App Store](https://apps.apple.com/app/jellyfin-mobile/id1480192618)

=== "Télévision (Android TV / Fire TV)"

    **Jellyfin for Android TV**

    - Google Play (Android TV) : cherche "Jellyfin"
    - Amazon Fire TV : cherche "Jellyfin"

=== "Kodi"

    Jellyfin dispose d'un plugin officiel pour Kodi : cherche "Jellyfin" dans les dépôts Kodi.

Dans chaque application, entre l'adresse du serveur : `http://192.168.1.xx:8096`

---

## Mises à jour automatiques

Comme pour Navidrome et Immich, les mises à jour se font toutes seules. Active le timer une bonne fois pour toutes :

```bash
systemctl --user enable --now podman-auto-update.timer
```

Podman vérifie chaque nuit et met à jour automatiquement. 

---

## En cas de problème

!!! question "Jellyfin ne démarre pas ?"
    ```bash
    systemctl --user status jellyfin.service
    podman logs jellyfin
    ```

!!! question "Je ne peux pas accéder depuis mon téléphone ?"
    Vérifie l'Étape 4 (pare-feu) et que le téléphone est sur le même réseau Wi-Fi.

!!! question "Les films n'apparaissent pas ou les affiches sont absentes ?"
    Dans l'interface web : **Tableau de bord → Bibliothèques → (ta bibliothèque) → Analyser**. Tu peux aussi forcer une récupération des métadonnées via **Tableau de bord → Tâches planifiées**.

!!! question "Comment redémarrer Jellyfin ?"
    ```bash
    systemctl --user restart jellyfin.service
    ```
