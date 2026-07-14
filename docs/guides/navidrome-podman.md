# Navidrome avec Podman

!!! info "À qui s'adresse ce guide ?"
    Ce guide est écrit pour quelqu'un qui n'est **pas informaticien**. Chaque étape est expliquée simplement, et tout se fait par **copier-coller**. Une fois l'installation terminée, tu n'auras plus jamais besoin d'ouvrir un terminal.

!!! note "Pourquoi Podman et pas une installation classique ?"
    Sur les distributions modernes (Fedora Kinoite, Silverblue, CachyOS...), la méthode recommandée est d'utiliser **Podman** — une sorte de boîte isolée qui fait tourner Navidrome sans toucher au reste du système. C'est plus propre, plus sûr, et ça fonctionne sur quasiment toutes les distributions Linux.

---

## Ce dont tu as besoin

- Une distribution Linux avec **Podman** installé
- Tes fichiers de musique quelque part sur ton ordinateur (ex. `/home/tonnom/Musique`)
- **Kate** (éditeur de texte) — cherche-le dans le menu
- **Konsole** (terminal) — cherche-le dans le menu

!!! tip "Ton nom d'utilisateur"
    Si tu ne sais pas ton nom d'utilisateur exact, ouvre Konsole et tape :
    ```bash
    echo $USER
    ```
    Note ce qui s'affiche, tu en auras besoin à l'étape 2.

---

## Prérequis — Vérifier que Podman est installé

Dans Konsole :

```bash
podman --version
```

Si tu vois un numéro de version (ex. `podman version 5.x.x`), c'est bon, passe directement à l'Étape 1.

Si tu obtiens une erreur, installe Podman selon ta distribution :


=== "Arch / CachyOS / Manjaro"

    ```bash
    sudo pacman -S podman
    ```

=== "Fedora / Silverblue / Kinoite"

    Podman est installé par défaut, cette erreur ne devrait pas arriver.  
    Si besoin :
    ```bash
    rpm-ostree install podman
    ```
    Puis redémarre l'ordinateur.

=== "Debian / Ubuntu / Mint"

    ```bash
    sudo apt install podman
    ```


## Étape 1 — Créer les dossiers nécessaires

Ouvre **Konsole** et copie-colle ces deux lignes (Entrée après chaque) :

```bash
mkdir -p ~/.config/containers/systemd
```

```bash
mkdir -p ~/.local/share/navidrome
```

!!! tip "Copier-coller dans Konsole"
    Pour coller dans Konsole, utilise **Ctrl+Shift+V** (pas juste Ctrl+V).

---

## Étape 2 — Créer le fichier de configuration

Ouvre le fichier avec Kate depuis Konsole :

```bash
kate ~/.config/containers/systemd/navidrome.container
```

Kate va s'ouvrir avec un fichier vide. Copie-colle **tout le contenu suivant** :

```ini
[Unit]
Description=Navidrome Music Server
After=network-online.target

[Container]
Image=docker.io/deluan/navidrome:latest
AutoUpdate=registry
PublishPort=4533:4533
Volume=%h/Musique:/music:ro,Z
Volume=%h/.local/share/navidrome:/data:Z
Environment=ND_MUSICFOLDER=/music
Environment=ND_DATAFOLDER=/data
Environment=ND_PORT=4533
Environment=ND_LOGLEVEL=info

[Service]
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

!!! warning "Vérifie le nom de ton dossier Musique"
    La ligne `Volume=%h/Musique:/music:ro,Z` suppose que ton dossier de musique s'appelle **Musique** (avec majuscule) dans ton dossier personnel.  
    Si ton dossier s'appelle autrement (`musique`, `Music`, `mes-musiques`...), modifie uniquement cette partie.  
    Le `%h` est remplacé automatiquement par le chemin de ton dossier personnel — ne le change pas.

Sauvegarde avec **Ctrl+S**, puis ferme Kate.

---

## Étape 3 — Activer et démarrer Navidrome

Dans **Konsole**, copie-colle ces commandes une par une :

```bash
systemctl --user daemon-reload
```

```bash
systemctl --user enable --now navidrome.service
```

```bash
loginctl enable-linger $USER
```

!!! info "À quoi servent ces commandes ?"
    - **daemon-reload** : indique à systemd de prendre en compte le nouveau fichier
    - **enable --now** : active le démarrage automatique au boot *et* démarre Navidrome immédiatement
    - **enable-linger** : fait tourner Navidrome même quand tu n'es pas connecté à ta session

!!! success "Vérification"
    Pour s'assurer que tout fonctionne :
    ```bash
    systemctl --user status navidrome.service
    ```
    Tu dois voir `active (running)` en vert. Si c'est le cas, **c'est la dernière fois que tu utilises le terminal pour Navidrome** 🎉

---

## Étape 4 — Ouvrir le pare-feu

Par défaut, Linux bloque les connexions venant d'autres appareils. On autorise le port de Navidrome.

Selon ta distribution, utilise l'onglet correspondant :

=== "ufw (CachyOS, Ubuntu, Debian, etc.)"

    ```bash
    sudo ufw allow 4533/tcp
    ```


=== "firewall-cmd (Fedora, etc.)"

    ```bash
    sudo firewall-cmd --add-port=4533/tcp --permanent
    sudo firewall-cmd --reload
    ```

=== "nftables (manuel)"

    ```bash
    sudo nft add rule inet filter input tcp dport 4533 accept
    ```
    Pour rendre la règle permanente, consulte la documentation de ta distribution.

!!! note "Mot de passe invisible"
    Quand tu tapes ton mot de passe après `sudo`, **rien ne s'affiche** — c'est tout à fait normal, tape quand même et appuie sur Entrée.

---

## Étape 5 — Premier accès

Ouvre **Firefox** et tape dans la barre d'adresse :

```
http://localhost:4533
```

!!! tip "Mets ça en favori !"
    Tu utiliseras cette adresse à chaque fois pour accéder à Navidrome depuis cet ordinateur.

Au premier lancement, Navidrome te demande de créer un **compte administrateur**. Mets un nom et un mot de passe dont tu te souviendras.

Une fois connecté, Navidrome va **scanner automatiquement** ta bibliothèque musicale. Selon le nombre de morceaux, ça peut prendre quelques minutes. Les pochettes d'albums apparaissent progressivement.

**C'est tout ! Tu n'as plus besoin du terminal.** 🎵

---

## Accès depuis ton téléphone ou d'autres appareils

Pour écouter ta musique depuis ton téléphone (sur le même réseau Wi-Fi), tu as besoin de l'adresse IP de ton ordinateur. Dans Konsole :

```bash
ip a
```

Cherche une ligne avec `inet` et une adresse du style `192.168.1.xx` — généralement sous `wlan0` (Wi-Fi) ou `enp...` (câble).

Depuis ton téléphone, ouvre le navigateur et tape :

```
http://192.168.1.xx:4533
```

*(remplace `xx` par le numéro trouvé)*

---

## Applications mobiles recommandées

### Android

| Application | Prix | Recommandation |
|---|---|---|
| **Substreamer** | Gratuit, open source | ⭐ Meilleur choix — sans pub, sans abonnement, supporte FLAC/MP3/AAC |
| **Amcfy Music** | Gratuit | Interface soignée, prometteuse |
| **Navsonic** | Gratuit | Spécialement conçu pour Navidrome, écoute hors-ligne |

### iOS

- **Substreamer** (App Store)
- **iSub** (compatible protocole Subsonic)

---

## Mises à jour automatiques

Bonne nouvelle : **les mises à jour de Navidrome se font toutes seules !**

Le fichier de configuration contient `AutoUpdate=registry`, ce qui indique à Podman de vérifier automatiquement les nouvelles versions. Il faut juste activer le timer qui s'en charge :

```bash
systemctl --user enable --now podman-auto-update.timer
```

C'est tout — Podman vérifie chaque nuit s'il y a une mise à jour et l'applique automatiquement.

!!! tip "Forcer une mise à jour immédiatement"
    ```bash
    podman auto-update
    ```

---

## En cas de problème

!!! question "Navidrome ne démarre pas ?"
    ```bash
    systemctl --user status navidrome.service
    podman logs navidrome
    ```
    Vérifie que le nom de ton dossier Musique est correct dans le fichier `.container`.

!!! question "Je ne peux pas accéder depuis mon téléphone ?"
    Vérifie que l'Étape 4 (pare-feu) a bien été faite, et que ton téléphone est sur le même réseau Wi-Fi que l'ordinateur.

!!! question "La musique n'apparaît pas ?"
    Le scan peut prendre du temps. Tu peux forcer un nouveau scan depuis l'interface web : **Menu ☰ → Bibliothèque → Analyser la bibliothèque**.

!!! question "Comment redémarrer Navidrome ?"
    ```bash
    ```