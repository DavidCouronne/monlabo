# Sauvegardes et restauration

!!! info "Pourquoi faire des sauvegardes ?"
    Tes fichiers de musique et tes photos sont précieux. Mais ce que tu risques vraiment de perdre, ce ne sont pas les fichiers eux-mêmes — c'est tout ce que tu as construit autour : tes **albums Immich**, tes **playlists Navidrome**, tes **compteurs d'écoute**, tes **comptes utilisateurs**. Tout ça est stocké dans une base de données. Ce guide t'explique comment la sauvegarder automatiquement et comment la restaurer si besoin.

!!! note "Ce guide suppose que tu as suivi les guides d'installation"
    - [Guide Navidrome](navidrome-podman.md) — pour la musique
    - [Guide Immich](immich-podman.md) — pour les photos

---

## Ce qu'on sauvegarde (et ce qu'on ne sauvegarde pas)

| Service | Base de données | Fichiers média |
|---|---|---|
| **Navidrome** | Playlists, comptes, historique d'écoute | Tes fichiers MP3/FLAC — **déjà dans `~/Musique`**, pas besoin de les sauvegarder séparément |
| **Immich** | Albums, tags, visages, comptes, favoris | Tes photos — **déjà dans `~/Photos`**, pas besoin de les sauvegarder séparément |

!!! warning "La base de données seule ne suffit pas"
    Si tu perds tes fichiers de musique ou de photos, la base de données ne les récupère pas. Elle contient uniquement les métadonnées. **Garde toujours une copie de tes fichiers sur un disque externe.**

---

## Navidrome — Activer les sauvegardes automatiques

Les sauvegardes de Navidrome se configurent en ajoutant trois lignes dans le fichier Quadlet.

### Modifier le fichier de configuration

Ouvre **Konsole** et lance Kate :

```bash
kate ~/.config/containers/systemd/navidrome.container
```

Trouve la section `[Container]` et ajoute ces trois lignes à la fin de cette section, juste avant `[Service]` :

```ini
Environment=ND_BACKUP_PATH=/backup
Environment=ND_BACKUP_SCHEDULE=0 2 * * *
Environment=ND_BACKUP_COUNT=7
```

Il faut aussi monter un dossier pour stocker les sauvegardes. Ajoute cette ligne avec les autres `Volume=` :

```ini
Volume=%h/Documents/Sauvegardes/navidrome:/backup:Z
```

??? note "À quoi ressemble le fichier complet après modification ?"

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
    Volume=%h/Documents/Sauvegardes/navidrome:/backup:Z
    Environment=ND_MUSICFOLDER=/music
    Environment=ND_DATAFOLDER=/data
    Environment=ND_PORT=4533
    Environment=ND_LOGLEVEL=info
    Environment=ND_BACKUP_PATH=/backup
    Environment=ND_BACKUP_SCHEDULE=0 2 * * *
    Environment=ND_BACKUP_COUNT=7

    [Service]
    Restart=always
    RestartSec=10

    [Install]
    WantedBy=default.target
    ```

Sauvegarde avec **Ctrl+S** et ferme Kate.

### Créer le dossier de sauvegarde et redémarrer

```bash
mkdir -p ~/Documents/Sauvegardes/navidrome
```

```bash
systemctl --user daemon-reload
systemctl --user restart navidrome.service
```

!!! success "C'est tout !"
    Navidrome va maintenant créer une sauvegarde automatiquement **tous les jours à 2h du matin**. Les 7 dernières sauvegardes sont conservées. Tu trouveras les fichiers dans `~/Documents/Sauvegardes/navidrome/`.

### Vérifier que ça fonctionne

Pour créer une sauvegarde immédiatement (sans attendre 2h du matin) :

```bash
podman exec navidrome navidrome backup create
```

Puis vérifie qu'un fichier est bien apparu :

```bash
ls ~/Documents/Sauvegardes/navidrome/
```

Tu dois voir un fichier `.bak` avec la date du jour.

---

## Immich — Activer les sauvegardes automatiques

Immich a un système de sauvegarde intégré directement dans son interface web — **pas besoin de toucher au terminal**.

### Activer depuis l'interface web

1. Ouvre Immich dans Firefox : **http://localhost:2283**
2. Clique sur ton avatar en haut à droite → **Administration**
3. Dans le menu de gauche : **Settings** (Paramètres)
4. Cherche la section **Backup** (Sauvegarde)
5. Vérifie que les paramètres sont bien activés :
    - **Enabled** : activé ✅
    - **Keep last N backups** : `7` (ou plus selon l'espace disque)
    - **Cron schedule** : `0 2 * * *` (tous les jours à 2h)
6. Clique **Save** pour enregistrer

!!! info "Où sont stockées les sauvegardes ?"
    Les sauvegardes Immich sont automatiquement dans `~/.local/share/immich/data/backups/`. Elles sont déjà dans ton dossier de données Immich.

### Créer une sauvegarde immédiatement

Pour ne pas attendre la nuit :

1. Va dans **Administration → Job Queues**
2. Clique sur **Create job** (en haut à droite)
3. Sélectionne **Create Database Backup**
4. Clique **Confirm**

Un nouveau fichier apparaîtra dans `~/.local/share/immich/data/backups/`.

---

## Copier les sauvegardes sur une clé USB ou un disque externe

Les sauvegardes automatiques protègent contre une erreur logicielle. Mais si ton ordinateur tombe en panne ou est volé, il faut une copie **physique** sur un support externe.

### Avec Dolphin (gestionnaire de fichiers)

Connecte ta clé USB ou ton disque externe, puis ouvre **Dolphin**.

**Pour Navidrome :**

Navigue jusqu'à `Documents/Sauvegardes/navidrome/` et copie-colle le dossier entier sur ta clé USB.

**Pour Immich :**

Navigue jusqu'à `.local/share/immich/data/backups/` — note que ce dossier est **caché** car il commence par un point. Pour l'afficher dans Dolphin : menu **Affichage → Afficher les fichiers cachés** (ou raccourci **Alt+.**).

Copie le dossier `backups/` sur ta clé USB.

!!! tip "À quelle fréquence copier sur clé USB ?"
    Une fois par mois suffit pour la plupart des usages. Prends l'habitude de le faire en même temps qu'une autre routine (ex. le premier dimanche du mois).

---

## Restaurer Navidrome

!!! danger "Lis ceci avant de restaurer"
    La restauration **remplace toutes les données actuelles** (playlists, comptes...) par celles du backup. C'est irréversible. Ne fais ça qu'en cas de vrai problème.

### Étape 1 — Arrêter Navidrome

```bash
systemctl --user stop navidrome.service
```

### Étape 2 — Lancer la restauration

Remplace `NOM_DU_FICHIER.bak` par le nom exact du fichier backup que tu veux restaurer (visible dans `~/Documents/Sauvegardes/navidrome/`) :

```bash
podman run --rm \
  -v ~/.local/share/navidrome:/data:Z \
  -v ~/Documents/Sauvegardes/navidrome:/backup:Z \
  docker.io/deluan/navidrome:latest \
  navidrome backup restore /backup/NOM_DU_FICHIER.bak
```

### Étape 3 — Redémarrer Navidrome

```bash
systemctl --user start navidrome.service
```

!!! success "Vérifie"
    Ouvre Firefox sur **http://localhost:4533** et vérifie que tes playlists et comptes sont bien revenus.

---

## Restaurer Immich

Immich permet de restaurer en quelques clics depuis l'interface web.

### Si Immich fonctionne encore

1. Ouvre **http://localhost:2283** dans Firefox
2. Va dans **Administration → Maintenance**
3. Déroule la section **Restore database backup**
4. Tu vois la liste des sauvegardes disponibles avec leur date
5. Clique **Restore** à côté de celle que tu veux
6. Confirme l'opération

!!! info "Sécurité intégrée"
    Immich crée automatiquement un point de sauvegarde juste avant de restaurer. Si quelque chose se passe mal, tu peux revenir en arrière.

### Si tu dois repartir de zéro (nouveau PC ou réinstallation)

1. Suis le [guide d'installation Immich](immich-podman.md) jusqu'à l'Étape 4 (activation des services)
2. **Avant** d'ouvrir l'interface web pour la première fois, copie tes anciens fichiers de sauvegarde :

```bash
cp /chemin/vers/ta/cle/usb/backups/* ~/.local/share/immich/data/backups/
```

3. Ouvre **http://localhost:2283** — Immich détecte automatiquement les sauvegardes disponibles et propose de les restaurer au premier lancement

---

## Résumé — Ce que tu dois faire une fois par mois

1. **Brancher** ta clé USB ou ton disque externe
2. **Copier** `~/Documents/Sauvegardes/navidrome/` sur la clé
3. **Copier** `~/.local/share/immich/data/backups/` sur la clé (penser à afficher les fichiers cachés dans Dolphin)
4. **Débrancher** la clé et la ranger en lieu sûr

Le reste — les sauvegardes quotidiennes automatiques — se fait tout seul. 🎉
