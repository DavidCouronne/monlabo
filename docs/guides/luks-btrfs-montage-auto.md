# Chiffrement LUKS + BTRFS — Montage automatique au boot

!!! note "À qui s'adresse ce guide ?"
    Ce guide est destiné à un utilisateur à l'aise avec le terminal Linux. Il couvre deux scénarios :
    
    - **Scénario A** — Disques vierges : partitionner, chiffrer, formater, puis configurer le montage automatique
    - **Scénario B** — Disques LUKS existants avec données : configurer uniquement le montage automatique sans toucher aux données

!!! warning "Spécificités Fedora Kinoite / Silverblue"
    Sur les systèmes immutables Fedora, quelques différences importantes :
    
    - Les points de montage persistants doivent être sous **`/var/mnt/`** (mutable), pas `/mnt/` (lecture seule)
    - Modifier `/etc/crypttab` et les units systemd nécessite **`pkexec`** (ou `sudo`) car ce sont des fichiers système
    - Après toute modification de `/etc/crypttab`, il faut **régénérer l'initramfs** avec `dracut -f`

---

## Étape 0 — Identifier les disques

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

Repère tes disques (sans mountpoint actif). Confirme :

```bash
sudo fdisk -l | grep -E "^Disk /dev"
```

=== "Scénario A — Disques vierges"

    Tu dois voir tes disques **sans** `TYPE=crypto_LUKS` ni système de fichiers existant.
    
    Exemple : `vdb`, `vdc` (VM) ou `sdb`, `sdc` (SATA) ou `nvme1n1`, `nvme2n1` (NVMe).

=== "Scénario B — Disques LUKS existants"

    Tu dois voir tes disques avec `FSTYPE=crypto_LUKS`.
    
    ```bash
    sudo blkid /dev/sdb /dev/sdc
    # Doit afficher TYPE="crypto_LUKS" pour chaque disque ou partition
    ```
    
    !!! tip "Passe directement à l'Étape 4"
        Les Étapes 1, 2 et 3 ne te concernent pas. Rends-toi directement à l'**Étape 4 — Récupérer les UUIDs**.

---

## Étape 1 — Partitionner les disques *(Scénario A uniquement)*

!!! danger "Scénario B : saute cette étape"
    Cette étape **détruit toutes les données** existantes. Ne l'exécute que sur des disques vierges.

Pour chaque disque (`/dev/vdb` et `/dev/vdc`, adapte selon tes disques) :

```bash
sudo fdisk /dev/vdb
```

Dans fdisk, tape dans l'ordre :

- `g` → crée une table GPT
- `n` → nouvelle partition (valide tout par défaut = partition unique)
- `w` → écrit et quitte

Répète avec `/dev/vdc`.

Vérifie :

```bash
lsblk
# Tu dois voir vdb1 et vdc1 (ou sdb1/sdc1, etc.)
```

---

## Étape 2 — Chiffrement LUKS *(Scénario A uniquement)*

!!! danger "Scénario B : saute cette étape"

```bash
sudo cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 --hash sha256 /dev/vdb1
```

Tape `YES` en majuscules, puis saisis ton mot de passe.

```bash
sudo cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 --hash sha256 /dev/vdc1
```

!!! info "Pourquoi LUKS2 ?"
    LUKS2 est le format moderne. Il utilise Argon2id par défaut, bien plus résistant au bruteforce que LUKS1.

---

## Étape 3 — Formater en BTRFS avec sous-volumes *(Scénario A uniquement)*

!!! danger "Scénario B : saute cette étape"

```bash
# Ouvrir temporairement les volumes
sudo cryptsetup open /dev/vdb1 data1
sudo cryptsetup open /dev/vdc1 data2

# Formater
sudo mkfs.btrfs -L "data1" /dev/mapper/data1
sudo mkfs.btrfs -L "data2" /dev/mapper/data2

# Monter temporairement
sudo mkdir -p /mnt/tmp1 /mnt/tmp2
sudo mount /dev/mapper/data1 /mnt/tmp1
sudo mount /dev/mapper/data2 /mnt/tmp2

# Créer les sous-volumes
sudo btrfs subvolume create /mnt/tmp1/@data
sudo btrfs subvolume create /mnt/tmp1/@snapshots
sudo btrfs subvolume create /mnt/tmp2/@data
sudo btrfs subvolume create /mnt/tmp2/@snapshots

# Démonter et fermer
sudo umount /mnt/tmp1 /mnt/tmp2
sudo cryptsetup close data1
sudo cryptsetup close data2
```

!!! info "Convention `@data` / `@snapshots`"
    Cette structure est celle utilisée par Snapper et Timeshift. En montant uniquement `@data` au démarrage, les snapshots restent accessibles séparément sans polluer l'arborescence principale.

---

## Étape 4 — Récupérer les UUIDs

Les UUIDs dont on a besoin sont ceux des **partitions LUKS** (pas des systèmes de fichiers à l'intérieur).

```bash
sudo blkid /dev/vdb1 /dev/vdc1
```

Sortie attendue :

```
/dev/vdb1: UUID="aaa111-..." TYPE="crypto_LUKS"
/dev/vdc1: UUID="bbb222-..." TYPE="crypto_LUKS"
```

!!! tip "Garde ce terminal ouvert"
    Tu vas réutiliser ces UUIDs dans les étapes suivantes. Ou copie-les dans un fichier texte.

---

## Étape 5 — Créer les clés de déchiffrement automatique

Ces clés permettent au système de déchiffrer les disques au boot **sans demander le mot de passe**. Ton mot de passe LUKS reste valide — la clé fichier est un slot supplémentaire.

```bash
# Crée le dossier sécurisé
sudo mkdir -p /etc/luks-keys
sudo chmod 700 /etc/luks-keys

# Génère les clés aléatoires
sudo dd if=/dev/urandom of=/etc/luks-keys/data1.key bs=4096 count=1
sudo dd if=/dev/urandom of=/etc/luks-keys/data2.key bs=4096 count=1

# Permissions strictes (lecture seule pour root uniquement)
sudo chmod 400 /etc/luks-keys/data1.key
sudo chmod 400 /etc/luks-keys/data2.key
```

Maintenant, ajoute ces clés aux slots LUKS de chaque disque. Tu vas devoir saisir ton **mot de passe LUKS existant** pour chaque disque :

```bash
sudo cryptsetup luksAddKey /dev/vdb1 /etc/luks-keys/data1.key
sudo cryptsetup luksAddKey /dev/vdc1 /etc/luks-keys/data2.key
```

Vérifie que les slots sont bien en place :

```bash
sudo cryptsetup luksDump /dev/vdb1 | grep "Key Slot"
sudo cryptsetup luksDump /dev/vdc1 | grep "Key Slot"
```

Tu dois voir au moins deux slots actifs (`ENABLED`).

---

## Étape 6 — Configurer `/etc/crypttab`

```bash
pkexec kate /etc/crypttab
```

Ajoute ces deux lignes **(remplace les UUID par les tiens)** :

```
data1   UUID=aaa111-...   /etc/luks-keys/data1.key   luks,discard
data2   UUID=bbb222-...   /etc/luks-keys/data2.key   luks,discard
```

Sauvegarde avec **Ctrl+S**.

!!! warning "Fedora Kinoite / toute distro Red Hat — étape critique"
    Après modification de `/etc/crypttab`, il faut **obligatoirement** régénérer l'initramfs pour que les changements soient pris en compte au boot :
    
    ```bash
    sudo dracut -f --regenerate-all
    ```
    
    Sans cette commande, le système ne saura pas déchiffrer les disques au démarrage.

!!! note "Debian / Ubuntu / Mint"
    Sur les distributions basées sur Debian, la commande équivalente est :
    ```bash
    sudo update-initramfs -u -k all
    ```

---

## Étape 7 — Créer les points de montage

=== "Fedora Kinoite / Silverblue"

    ```bash
    sudo mkdir -p /var/mnt/data1 /var/mnt/data2
    ```
    
    !!! info "Pourquoi `/var/mnt/` ?"
        Sur les systèmes immutables Fedora, `/mnt/` est en lecture seule. `/var/mnt/` est le répertoire mutable prévu à cet effet.

=== "Autres distributions"

    ```bash
    sudo mkdir -p /mnt/data1 /mnt/data2
    ```
    
    Adapte le chemin selon ta préférence (`/media/data1`, `/data/data1`, etc.).

---

## Étape 8 — Créer les units systemd `.mount`

Les units systemd `.mount` remplacent avantageusement `/etc/fstab` pour les volumes chiffrés : elles gèrent les dépendances avec `systemd-cryptsetup` proprement.

!!! warning "Nom du fichier = chemin de montage"
    Le nom du fichier `.mount` **doit** correspondre exactement au chemin de montage, avec les `/` remplacés par des `-`.
    
    - `/var/mnt/data1` → `var-mnt-data1.mount`
    - `/mnt/data1` → `mnt-data1.mount`

=== "Scénario A — Nouveaux disques avec sous-volumes `@data`"

    **Unit pour data1 :**
    
    ```bash
    pkexec kate /etc/systemd/system/var-mnt-data1.mount
    ```
    
    ```ini
    [Unit]
    Description=Montage BTRFS chiffré data1
    After=systemd-cryptsetup@data1.service
    Requires=systemd-cryptsetup@data1.service
    
    [Mount]
    What=/dev/mapper/data1
    Where=/var/mnt/data1
    Type=btrfs
    Options=subvol=/@data,defaults,noatime,compress=zstd:3,space_cache=v2
    
    [Install]
    WantedBy=multi-user.target
    ```
    
    **Unit pour data2 :**
    
    ```bash
    pkexec kate /etc/systemd/system/var-mnt-data2.mount
    ```
    
    ```ini
    [Unit]
    Description=Montage BTRFS chiffré data2
    After=systemd-cryptsetup@data2.service
    Requires=systemd-cryptsetup@data2.service
    
    [Mount]
    What=/dev/mapper/data2
    Where=/var/mnt/data2
    Type=btrfs
    Options=subvol=/@data,defaults,noatime,compress=zstd:3,space_cache=v2
    
    [Install]
    WantedBy=multi-user.target
    ```

=== "Scénario B — Disques existants sans sous-volumes"

    Si ton volume BTRFS n'a pas la structure `@data`/`@snapshots`, monte le volume tel quel (sans `subvol=`) :
    
    ```ini
    [Unit]
    Description=Montage BTRFS chiffré data1
    After=systemd-cryptsetup@data1.service
    Requires=systemd-cryptsetup@data1.service
    
    [Mount]
    What=/dev/mapper/data1
    Where=/var/mnt/data1
    Type=btrfs
    Options=defaults,noatime,compress=zstd:3,space_cache=v2
    
    [Install]
    WantedBy=multi-user.target
    ```

=== "Scénario B — Disques existants avec sous-volumes"

    Si ton volume a déjà des sous-volumes, vérifie leur nom d'abord :
    
    ```bash
    # Ouvrir temporairement
    sudo cryptsetup open /dev/vdb1 data1-check
    sudo mkdir -p /mnt/check
    sudo mount /dev/mapper/data1-check /mnt/check
    
    # Lister les sous-volumes
    sudo btrfs subvolume list /mnt/check
    
    # Démonter et fermer
    sudo umount /mnt/check
    sudo cryptsetup close data1-check
    ```
    
    Puis adapte `subvol=/@TON_SOUS_VOLUME` dans l'option `Options=` de l'unit.

!!! info "Options de montage BTRFS expliquées"
    - `noatime` — ne met pas à jour la date d'accès à chaque lecture (réduit les écritures inutiles)
    - `compress=zstd:3` — compression zstd niveau 3, bon compromis vitesse/ratio
    - `space_cache=v2` — cache BTRFS moderne, plus fiable que v1
    - `discard` dans crypttab — active le TRIM pour les SSD/NVMe à travers le chiffrement

---

## Étape 9 — Activer et démarrer les units

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now var-mnt-data1.mount
sudo systemctl enable --now var-mnt-data2.mount
```

Vérifie immédiatement :

```bash
systemctl status var-mnt-data1.mount var-mnt-data2.mount
```

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

Les deux volumes doivent apparaître montés sur `/var/mnt/data1` et `/var/mnt/data2`.

---

## Étape 10 — Redémarrer et vérifier

```bash
sudo systemctl reboot
```

Après redémarrage, vérifie que tout s'est monté automatiquement :

```bash
# Les volumes doivent être montés
findmnt /var/mnt/data1
findmnt /var/mnt/data2

# Vérifie les sous-volumes BTRFS (Scénario A)
sudo btrfs subvolume list /var/mnt/data1
```

---

## Bonus — Snapshots manuels

Une fois les volumes montés, créer un snapshot est trivial :

```bash
# Snapshot daté de data1
sudo btrfs subvolume snapshot /var/mnt/data1 /var/mnt/data1/../@snapshots/data1-$(date +%Y%m%d)
```

Pour automatiser les snapshots avec **Snapper** (compatible avec la structure `@data`/`@snapshots`) :

=== "Fedora Kinoite (rpm-ostree)"

    ```bash
    rpm-ostree install snapper
    # Redémarrage requis, puis :
    sudo snapper -c data1 create-config /var/mnt/data1
    ```

=== "Arch / CachyOS"

    ```bash
    sudo pacman -S snapper
    sudo snapper -c data1 create-config /var/mnt/data1
    ```

=== "Debian / Ubuntu"

    ```bash
    sudo apt install snapper
    sudo snapper -c data1 create-config /var/mnt/data1
    ```

---

## Récapitulatif de la structure finale

```
/dev/vdb1  (crypto_LUKS)
  └── /dev/mapper/data1  (BTRFS)
        ├── @data        → monté sur /var/mnt/data1
        └── @snapshots   → accessible manuellement

/dev/vdc1  (crypto_LUKS)
  └── /dev/mapper/data2  (BTRFS)
        ├── @data        → monté sur /var/mnt/data2
        └── @snapshots   → accessible manuellement
```

---

## En cas de problème

!!! question "Le système demande le mot de passe au boot malgré la clé fichier ?"
    La clé fichier n'est pas chargée dans l'initramfs. Vérifie que `dracut -f --regenerate-all` a bien été exécuté après la modification de `/etc/crypttab`.

!!! question "L'unit `.mount` échoue au démarrage ?"
    ```bash
    journalctl -b -u var-mnt-data1.mount
    ```
    Vérifie que le nom du fichier correspond exactement au chemin de montage (tirets à la place des slashes).

!!! question "Le volume BTRFS est monté mais les données ne sont pas là ?"
    Tu montes probablement le mauvais sous-volume. Vérifie avec :
    ```bash
    sudo btrfs subvolume list /var/mnt/data1
    ```
    Et adapte l'option `subvol=` dans l'unit `.mount`.

!!! question "Comment vérifier que LUKS a bien deux slots actifs ?"
    ```bash
    sudo cryptsetup luksDump /dev/vdb1
    ```
    Tu dois voir le slot 0 (mot de passe) et le slot 1 (clé fichier) tous les deux en `ENABLED`.
