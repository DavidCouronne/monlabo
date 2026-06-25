# Chiffrement LUKS, déverrouillage TPM et bootstrap automatisé

Guide pour Bazzite / Aurora / Bluefin (Universal Blue) — gestion des disques chiffrés LUKS,
déverrouillage automatique via TPM2, et déploiement reproductible depuis un dépôt GitHub
avec gestion des secrets via `age`.

---

## Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────┐
│  Bootstrap (une commande)                                       │
│    └─ clone GitHub → age déchiffre les secrets                  │
│         └─ déploie configs, quadlets, units systemd             │
│              └─ enrôle les disques secondaires au TPM2          │
│                   └─ lance les services                          │
└─────────────────────────────────────────────────────────────────┘

Disque système   →  ujust setup-luks-tpm-unlock  (PCR 7+14)
Disques données  →  systemd-cryptenroll manuel   (PCR 7+14)
                    + /etc/crypttab
Fallback         →  mot de passe LUKS (toujours présent)
```

!!! warning "Avant de commencer"
    - Les disques secondaires doivent être **déjà chiffrés en LUKS2** avec des données dessus.
      On ajoute un slot TPM2, on ne reformate pas.
    - Secure Boot doit être actif et les clés UBlue enrôlées
      (`ujust enroll-secure-boot-key`) pour que les PCR soient stables.
    - Si tu changes les clés Secure Boot ou le MOKList après l'enrôlement TPM,
      il faudra ré-enrôler les disques.

---

## Partie 1 — Identifier les disques et leurs UUID

### 1.1 Lister tous les disques et partitions

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINT
```

Repère tes disques chiffrés : leur `FSTYPE` sera `crypto_LUKS`.

### 1.2 Obtenir l'UUID LUKS de chaque disque secondaire

L'UUID LUKS est celui du **conteneur chiffré** (le disque brut), pas du filesystem intérieur.

```bash
# Remplace /dev/sdX par chaque disque secondaire
sudo blkid /dev/sdX
```

La ligne ressemblera à :

```
/dev/sdb: UUID="a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS"
```

Note ces UUID — ils iront dans `/etc/crypttab`.

### 1.3 Obtenir l'UUID du filesystem intérieur (btrfs)

Le filesystem n'est accessible qu'une fois le conteneur ouvert.

```bash
# Ouvrir manuellement le conteneur (tape ton mot de passe LUKS)
sudo cryptsetup open /dev/sdX data1

# Lire l'UUID du filesystem
sudo blkid /dev/mapper/data1

# Fermer
sudo cryptsetup close data1
```

La ligne ressemblera à :

```
/dev/mapper/data1: UUID="f5e6d7c8-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="btrfs"
```

Note ces UUID — ils iront dans `/etc/fstab`.

### 1.4 Tableau récapitulatif à remplir

Complète ce tableau avant de continuer :

| Nom logique | Périphérique | UUID LUKS (crypttab) | UUID filesystem (fstab) | Point de montage |
|-------------|-------------|----------------------|------------------------|-----------------|
| `data1`     | `/dev/sdb`  | `a1b2c3d4-...`       | `f5e6d7c8-...`         | `~/data1`       |
| `data2`     | `/dev/sdc`  | `...`                | `...`                  | `~/data2`       |

---

## Partie 2 — Disque système : déverrouillage TPM2

Le disque système est géré par `ujust` qui s'appuie sur `rd.luks.uuid` dans la ligne de
commande du kernel — il n'agit **que** sur le disque système.

```bash
# Vérifier que Secure Boot est actif et les clés enrôlées
mokutil --sb-state

# Enrôler les clés UBlue si ce n'est pas fait
ujust enroll-secure-boot-key

# Configurer le déverrouillage TPM2 du disque système
ujust setup-luks-tpm-unlock
```

!!! info "Ce que fait le script"
    Il lit `rd.luks.uuid` dans `/proc/cmdline`, puis exécute
    `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14` sur ce seul disque,
    et régénère l'initramfs via `rpm-ostree initramfs --enable`.

Redémarre et vérifie que le système déverrouille automatiquement.

---

## Partie 3 — Disques secondaires : ajout d'un slot TPM2

### 3.1 Enrôler chaque disque

`systemd-cryptenroll` **ajoute** un slot TPM2 sans toucher au slot mot de passe existant.
Le mot de passe LUKS actuel est demandé pour confirmer l'opération.

```bash
# Remplace /dev/sdX par le vrai périphérique
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14 /dev/sdb
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14 /dev/sdc
```

### 3.2 Vérifier les slots

```bash
sudo cryptsetup luksDump /dev/sdb | grep -A5 "Keyslots"
```

Tu dois voir au moins deux slots : l'un de type `luks2` (mot de passe), l'autre `systemd-tpm2`.

### 3.3 Configurer `/etc/crypttab`

`/etc/crypttab` est mutable sur Fedora Atomic — c'est prévu et il survit aux mises à jour.

```bash
sudo nano /etc/crypttab
```

Ajoute une ligne par disque secondaire (utilise les UUID LUKS du tableau section 1.4) :

```
# Format : nom   UUID   options-clé   options
data1   UUID=a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx   none   tpm2-device=auto,tpm2-pcrs=7+14,nofail
data2   UUID=...                                     none   tpm2-device=auto,tpm2-pcrs=7+14,nofail
```

!!! tip "Option `nofail`"
    Si le TPM échoue (boot depuis USB, Secure Boot modifié...), le système boot quand même
    et demandera le mot de passe LUKS plutôt que de bloquer.

### 3.4 Configurer `/etc/fstab`

```bash
sudo nano /etc/fstab
```

Ajoute une ligne par disque (utilise les UUID **filesystem** du tableau section 1.4) :

```
# Disques chiffrés secondaires — montés après déverrouillage crypttab
UUID=f5e6d7c8-xxxx-xxxx-xxxx-xxxxxxxxxxxx   /home/TON_USER/data1   btrfs   defaults,compress=zstd,noatime,nofail,x-systemd.requires=dev-mapper-data1.device   0   0
UUID=...                                     /home/TON_USER/data2   btrfs   defaults,compress=zstd,noatime,nofail,x-systemd.requires=dev-mapper-data2.device   0   0
```

!!! warning "Crée les points de montage d'abord"
    ```bash
    mkdir -p ~/data1 ~/data2
    ```

### 3.5 Tester sans redémarrer

```bash
sudo systemctl daemon-reload
sudo systemctl start systemd-cryptsetup@data1.service
sudo systemctl start home-TON_USER-data1.mount
```

Vérifie que le disque est monté :

```bash
df -h ~/data1
```

---

## Partie 4 — Gestion des secrets avec `age`

### 4.1 Installer `age`

`age` est disponible via Homebrew (inclus dans UBlue), sans toucher à `rpm-ostree` :

```bash
brew install age
```

### 4.2 Générer une paire de clés

```bash
mkdir -p ~/.config/age
age-keygen -o ~/.config/age/identity.txt
# Affiche ta clé publique :
# Public key: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

!!! danger "Sauvegarde ta clé privée"
    `~/.config/age/identity.txt` contient ta clé privée.
    **Sauvegarde-la hors ligne** (Bitwarden, clé USB chiffrée...).
    Si tu la perds, tu ne pourras plus déchiffrer tes secrets.

Note ta clé publique (commence par `age1...`) — elle sera utilisée pour chiffrer.

### 4.3 Alternative : utiliser ta clé SSH existante

Si tu as déjà une clé Ed25519 :

```bash
# Extraire la clé publique age depuis ta clé SSH
age-keygen -y ~/.ssh/id_ed25519 > ~/.config/age/pubkey.txt
cat ~/.config/age/pubkey.txt
```

Pour déchiffrer plus tard, tu utiliseras `--identity ~/.ssh/id_ed25519`.

### 4.4 Chiffrer tes secrets (à faire une fois, sur la machine de référence)

```bash
# Exemple : chiffrer la config rclone
age -r age1xxxx... -o secrets/rclone.conf.age ~/.config/rclone/rclone.conf

# Exemple : générer et chiffrer une keyfile LUKS (optionnel, si tu veux une keyfile en plus du TPM)
dd if=/dev/urandom bs=512 count=1 | age -r age1xxxx... -o secrets/luks-data1.key.age

# Exemple : chiffrer le mot de passe restic
echo "mon-super-mot-de-passe" | age -r age1xxxx... -o secrets/restic-password.age
```

!!! tip "Vérification"
    ```bash
    age --decrypt -i ~/.config/age/identity.txt secrets/rclone.conf.age
    ```

---

## Partie 5 — Structure du dépôt GitHub

```
mon-dotfiles/
├── bootstrap.sh                    # script principal (voir Partie 6)
├── secrets/
│   ├── .gitkeep
│   ├── rclone.conf.age
│   ├── restic-password.age
│   └── luks-data1.key.age          # optionnel
├── config/
│   ├── systemd/                    # units utilisateur
│   │   └── user/
│   │       └── mon-service.service
│   ├── quadlets/                   # fichiers podman quadlet
│   │   ├── mon-app.container
│   │   └── mon-volume.volume
│   └── rclone/                     # config non-secrète (structure, pas credentials)
│       └── rclone.conf.template
└── docs/                           # ce guide, par exemple
    └── luks-bootstrap.md
```

!!! warning "Ne jamais commiter de secrets en clair"
    Tous les fichiers dans `secrets/` doivent être des `.age`.
    Ajoute au `.gitignore` :
    ```
    secrets/*.key
    secrets/*.conf
    secrets/*.password
    !secrets/*.age
    ```

---

## Partie 6 — Script de bootstrap

Crée `bootstrap.sh` à la racine du dépôt :

```bash
#!/usr/bin/env bash
# bootstrap.sh — déploiement initial sur install fraîche Bazzite/Aurora/Bluefin
set -euo pipefail

# ── À ADAPTER ──────────────────────────────────────────────────────────────────
REPO_URL="https://github.com/TON_USER/mon-dotfiles.git"
REPO_DIR="$HOME/.local/share/dotfiles"
AGE_IDENTITY="$HOME/.config/age/identity.txt"

# Disques secondaires : tableau associatif nom→périphérique
declare -A SECONDARY_DISKS=(
    ["data1"]="/dev/sdb"
    ["data2"]="/dev/sdc"
)

# Points de montage correspondants
declare -A MOUNT_POINTS=(
    ["data1"]="$HOME/data1"
    ["data2"]="$HOME/data2"
)

# UUID filesystem btrfs pour fstab (récupérés à la section 1.4)
declare -A FS_UUIDS=(
    ["data1"]="f5e6d7c8-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    ["data2"]="..."
)
# ── FIN DE LA SECTION À ADAPTER ────────────────────────────────────────────────

echo "╔════════════════════════════════════════════════╗"
echo "║   Bootstrap Universal Blue — $(date +%Y-%m-%d)   ║"
echo "╚════════════════════════════════════════════════╝"
echo

# ── 1. Dépendances ─────────────────────────────────────────────────────────────
echo "▶ Vérification des dépendances..."

if ! command -v age &>/dev/null; then
    echo "  Installation de age via brew..."
    brew install age
fi

if ! command -v git &>/dev/null; then
    echo "  git non trouvé. Installation..."
    brew install git
fi

# ── 2. Clé age ─────────────────────────────────────────────────────────────────
if [[ ! -f "$AGE_IDENTITY" ]]; then
    echo
    echo "▶ Clé age introuvable : $AGE_IDENTITY"
    echo "  Options :"
    echo "  [1] Coller la clé privée (depuis ton gestionnaire de mots de passe)"
    echo "  [2] Utiliser la clé SSH existante (~/.ssh/id_ed25519)"
    read -rp "  Choix [1/2] : " KEY_CHOICE

    if [[ "$KEY_CHOICE" == "2" ]]; then
        AGE_IDENTITY="$HOME/.ssh/id_ed25519"
        echo "  Utilisation de la clé SSH."
    else
        mkdir -p "$(dirname "$AGE_IDENTITY")"
        echo "  Colle ta clé privée age (termine avec Ctrl+D sur une ligne vide) :"
        cat > "$AGE_IDENTITY"
        chmod 600 "$AGE_IDENTITY"
        echo "  Clé enregistrée."
    fi
fi

# ── 3. Clone du dépôt ──────────────────────────────────────────────────────────
echo
echo "▶ Récupération du dépôt..."
if [[ -d "$REPO_DIR/.git" ]]; then
    echo "  Mise à jour du dépôt existant..."
    git -C "$REPO_DIR" pull --ff-only
else
    git clone "$REPO_URL" "$REPO_DIR"
fi
cd "$REPO_DIR"

# ── 4. Déchiffrement et déploiement des secrets ────────────────────────────────
echo
echo "▶ Déploiement des secrets..."

deploy_secret() {
    local src="$1"   # chemin relatif dans secrets/
    local dst="$2"   # chemin absolu de destination
    mkdir -p "$(dirname "$dst")"
    age --decrypt -i "$AGE_IDENTITY" -o "$dst" "secrets/$src"
    chmod 600 "$dst"
    echo "  ✓ $dst"
}

deploy_secret "rclone.conf.age"      "$HOME/.config/rclone/rclone.conf"
deploy_secret "restic-password.age"  "$HOME/.local/share/restic/password"
# deploy_secret "luks-data1.key.age" "$HOME/.secrets/luks-data1.key"  # si keyfile utilisée

# ── 5. Déploiement des configs système ────────────────────────────────────────
echo
echo "▶ Déploiement des configurations..."

# Units systemd utilisateur
mkdir -p "$HOME/.config/systemd/user"
rsync -av config/systemd/user/ "$HOME/.config/systemd/user/"

# Quadlets podman
mkdir -p "$HOME/.config/containers/systemd"
rsync -av config/quadlets/ "$HOME/.config/containers/systemd/"

# ── 6. Création des points de montage ─────────────────────────────────────────
echo
echo "▶ Création des points de montage..."
for name in "${!MOUNT_POINTS[@]}"; do
    mkdir -p "${MOUNT_POINTS[$name]}"
    echo "  ✓ ${MOUNT_POINTS[$name]}"
done

# ── 7. Enrôlement TPM2 des disques secondaires ────────────────────────────────
echo
echo "▶ Enrôlement TPM2 des disques secondaires..."
echo "  (Ton mot de passe LUKS sera demandé pour chaque disque)"

for name in "${!SECONDARY_DISKS[@]}"; do
    dev="${SECONDARY_DISKS[$name]}"
    uuid_luks=$(sudo blkid -s UUID -o value "$dev")

    echo
    echo "  Disque : $dev ($name) — UUID LUKS : $uuid_luks"

    # Vérifier si un slot TPM2 existe déjà
    if sudo cryptsetup luksDump "$dev" | grep -q "systemd-tpm2"; then
        echo "  → Slot TPM2 déjà présent, passage au suivant."
        continue
    fi

    sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14 "$dev"
    echo "  ✓ Enrôlement TPM2 terminé pour $name"

    # Ajouter à /etc/crypttab si absent
    if ! sudo grep -q "$uuid_luks" /etc/crypttab 2>/dev/null; then
        echo "$name   UUID=$uuid_luks   none   tpm2-device=auto,tpm2-pcrs=7+14,nofail" \
            | sudo tee -a /etc/crypttab > /dev/null
        echo "  ✓ Entrée ajoutée dans /etc/crypttab"
    fi

    # Ajouter à /etc/fstab si absent
    fs_uuid="${FS_UUIDS[$name]}"
    if [[ -n "$fs_uuid" ]] && ! sudo grep -q "$fs_uuid" /etc/fstab 2>/dev/null; then
        echo "UUID=$fs_uuid   ${MOUNT_POINTS[$name]}   btrfs   defaults,compress=zstd,noatime,nofail,x-systemd.requires=dev-mapper-${name}.device   0   0" \
            | sudo tee -a /etc/fstab > /dev/null
        echo "  ✓ Entrée ajoutée dans /etc/fstab"
    fi
done

# ── 8. Déverrouillage TPM2 du disque système ──────────────────────────────────
echo
read -rp "▶ Configurer le déverrouillage TPM2 du disque système ? [y/N] " SETUP_SYS
if [[ "${SETUP_SYS,,}" == "y" ]]; then
    if [[ -e /dev/tpm0 ]] || [[ -e /dev/tpmrm0 ]]; then
        ujust setup-luks-tpm-unlock
    else
        echo "  ⚠ Aucun TPM détecté, étape ignorée."
    fi
fi

# ── 9. Activation des services ────────────────────────────────────────────────
echo
echo "▶ Activation des services systemd utilisateur..."
systemctl --user daemon-reload
# Adapte selon tes services :
# systemctl --user enable --now mon-service.service

systemctl --user daemon-reload  # recharge aussi les quadlets
echo "  ✓ Services rechargés"

# ── Fin ───────────────────────────────────────────────────────────────────────
echo
echo "╔════════════════════════════════════════════════╗"
echo "║   Bootstrap terminé ! Redémarre pour tester.  ║"
echo "╚════════════════════════════════════════════════╝"
echo
echo "  Vérifie que tout est en ordre :"
echo "  • sudo cat /etc/crypttab"
echo "  • sudo cat /etc/fstab"
echo "  • systemctl --user status"
```

---

## Partie 7 — Premier lancement et vérifications

### 7.1 Lancer le bootstrap

Sur une install fraîche :

```bash
# Si git est déjà dispo (il l'est sur Bazzite)
git clone https://github.com/TON_USER/mon-dotfiles.git ~/bootstrap-tmp
cd ~/bootstrap-tmp
chmod +x bootstrap.sh
./bootstrap.sh
```

### 7.2 Vérifier `/etc/crypttab`

```bash
sudo cat /etc/crypttab
```

Chaque disque secondaire doit avoir sa ligne avec l'option `tpm2-device=auto`.

### 7.3 Vérifier `/etc/fstab`

```bash
sudo cat /etc/fstab
```

Chaque point de montage doit apparaître avec `nofail` et `x-systemd.requires`.

### 7.4 Tester le montage à chaud (sans redémarrer)

```bash
sudo systemctl daemon-reload
sudo systemctl start systemd-cryptsetup@data1.service
sudo mount ~/data1
df -h ~/data1
```

### 7.5 Redémarrer et vérifier

```bash
reboot
```

Après le boot :

```bash
# Les disques doivent être montés automatiquement
df -h ~/data1 ~/data2

# Vérifier les services cryptsetup
systemctl status systemd-cryptsetup@data1.service
systemctl status systemd-cryptsetup@data2.service
```

---

## Partie 8 — Maintenance et cas particuliers

### Re-enrôler après un changement de Secure Boot / MOKList

Si tu as ré-enrôlé des clés (`ujust enroll-secure-boot-key`) ou modifié le MOKList,
les PCR 7+14 changent et les slots TPM2 deviennent invalides.

```bash
# Retirer les anciens slots TPM2
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sdb
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sdc

# Ré-enrôler
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14 /dev/sdb
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14 /dev/sdc
```

Pour le disque système :

```bash
ujust remove-luks-tpm-unlock
ujust setup-luks-tpm-unlock
```

### Vérifier les slots d'un disque

```bash
sudo cryptsetup luksDump /dev/sdb
```

Cherche la section `Keyslots` : les slots `luks2` sont les mots de passe,
les slots `systemd-tpm2` sont les entrées TPM.

### Mettre à jour les secrets dans le dépôt

```bash
# Re-chiffrer un secret modifié
age -r age1xxxx... -o secrets/rclone.conf.age ~/.config/rclone/rclone.conf
git add secrets/rclone.conf.age
git commit -m "chore: update rclone config"
git push

# Sur une autre machine, relancer le bootstrap pour récupérer la mise à jour
cd ~/.local/share/dotfiles && git pull && ./bootstrap.sh
```

### Supprimer un slot TPM2 (sans toucher au mot de passe)

```bash
# Identifier le numéro de slot
sudo cryptsetup luksDump /dev/sdb | grep -B2 "systemd-tpm2"

# Supprimer ce slot uniquement
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sdb
```

---

## Référence rapide — Commandes fréquentes

| Besoin | Commande |
|--------|----------|
| Lister les disques et UUIDs | `lsblk -o NAME,SIZE,FSTYPE,UUID` |
| UUID d'un disque précis | `sudo blkid /dev/sdX` |
| Dump des slots LUKS | `sudo cryptsetup luksDump /dev/sdX` |
| Ajouter slot TPM2 | `sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+14 /dev/sdX` |
| Retirer slot TPM2 | `sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sdX` |
| Ouvrir un conteneur LUKS manuellement | `sudo cryptsetup open /dev/sdX nom` |
| Fermer un conteneur LUKS | `sudo cryptsetup close nom` |
| Chiffrer avec age | `age -r age1xxxx... -o secret.age fichier` |
| Déchiffrer avec age | `age --decrypt -i identity.txt -o fichier secret.age` |
| Tester fstab sans reboot | `sudo systemctl daemon-reload && sudo mount -a` |
| Disque système TPM | `ujust setup-luks-tpm-unlock` |
| Retirer TPM disque système | `ujust remove-luks-tpm-unlock` |
