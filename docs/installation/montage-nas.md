# 📂 Montage Semi-Automatique de Partages NAS

Cette page documente la mise en place d'un système de montage manuel (non automatique au démarrage) de partages SMB/CIFS depuis un NAS.

---

## 🎯 Pourquoi un montage manuel ?

!!! warning "Pas de montage automatique au démarrage"
    Le montage automatique au démarrage peut créer plusieurs problèmes :
    
    - **Échec de démarrage** : Si le NAS est éteint, le système peut bloquer ou ralentir au boot
    - **Perturbation réseau** : Le montage peut consommer de la bande passante au moment où d'autres utilisent Netflix, le gaming, etc.
    - **Flexibilité** : On ne monte les partages que quand on en a vraiment besoin
    
    Un montage manuel via un script est beaucoup plus pratique et fiable !

---

## 📋 Prérequis

### Installation des outils nécessaires

```bash
sudo pacman -S cifs-utils smbclient
```

### Création du fichier de credentials

Pour éviter de taper le mot de passe à chaque montage, on crée un fichier sécurisé :


**Créer/Éditer le fichier avec Kate :**

```bash
kate /etc/nas-credentials
```

**Contenu du fichier :**

```ini
username=votre_utilisateur_nas
password=votre_mot_de_passe_nas
domain=WORKGROUP
```

!!! danger "Sécurité importante"
    - Le fichier `/etc/nas-credentials` contient vos identifiants en clair
    - **Permissions 600** (lecture/écriture uniquement par root) sont OBLIGATOIRES
    - Ne partagez JAMAIS ce fichier
    - Vérifiez les permissions : `ls -l /etc/nas-credentials` doit afficher `-rw-------`

### Vérification des permissions

```bash
# Vérifier que seul root peut lire le fichier
ls -l /etc/nas-credentials
# Doit afficher : -rw------- 1 root root

# Si ce n'est pas le cas, corriger :
sudo chmod 600 /etc/nas-credentials
sudo chown root:root /etc/nas-credentials
```

---

## 📝 Le Script de Montage

### Création du script

```bash
# Créer le script
kate /usr/local/bin/mount-nas.sh
```

**Contenu du script :**

```bash
#!/usr/bin/env bash

# Configuration NAS
NAS_IP="192.168.1.100"

# Liste des partages à monter (format : "//IP/partage:/point/montage")
MOUNT_POINTS=(
    "//192.168.1.100/documents:/mnt/nas/documents"
    "//192.168.1.100/photos:/mnt/nas/photos"
    "//192.168.1.100/videos:/mnt/nas/videos"
    "//192.168.1.100/music:/mnt/nas/music"
)

CREDENTIALS="/etc/nas-credentials"

# Options de montage optimisées
MOUNT_OPTIONS="credentials=$CREDENTIALS,uid=1000,gid=1000,iocharset=utf8,file_mode=0644,dir_mode=0755,cache=strict,rsize=65536,wsize=65536"

# Fonction de log
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a /var/log/nas-mount.log
}

# Vérifier si le NAS est accessible
check_nas() {
    log "Vérification de la disponibilité du NAS ($NAS_IP)..."
    if ping -c 2 -W 3 "$NAS_IP" >/dev/null 2>&1; then
        log "NAS accessible"
        return 0
    else
        log "NAS non accessible"
        return 1
    fi
}

# Fonction de montage
mount_share() {
    local share_path="$1"
    local mount_point="$2"
    
    # Créer le point de montage s'il n'existe pas
    if [ ! -d "$mount_point" ]; then
        mkdir -p "$mount_point"
        log "Point de montage créé : $mount_point"
    fi
    
    # Vérifier si déjà monté
    if mountpoint -q "$mount_point"; then
        log "$mount_point déjà monté"
        return 0
    fi
    
    # Tenter le montage
    log "Montage de $share_path vers $mount_point..."
    if mount -t cifs "$share_path" "$mount_point" -o "$MOUNT_OPTIONS"; then
        log "Montage réussi : $mount_point"
        
        # Vérifier que le montage n'est pas vide
        sleep 2
        if [ "$(ls -A "$mount_point" 2>/dev/null | wc -l)" -eq 0 ]; then
            log "Attention : $mount_point semble vide après montage"
        fi
        return 0
    else
        log "Échec du montage : $mount_point"
        return 1
    fi
}

# Fonction de démontage
unmount_share() {
    local mount_point="$1"
    
    if mountpoint -q "$mount_point"; then
        log "Démontage de $mount_point..."
        if umount "$mount_point"; then
            log "Démontage réussi : $mount_point"
        else
            log "Échec du démontage : $mount_point"
            # Force le démontage si nécessaire
            umount -f "$mount_point" 2>/dev/null
        fi
    else
        log "$mount_point n'est pas monté"
    fi
}

# Fonction principale
main() {
    case "$1" in
        mount)
            if check_nas; then
                for mount_config in "${MOUNT_POINTS[@]}"; do
                    IFS=':' read -r share_path mount_point <<< "$mount_config"
                    mount_share "$share_path" "$mount_point"
                done
            else
                log "NAS non accessible, montage annulé"
                exit 1
            fi
            ;;
        unmount)
            for mount_config in "${MOUNT_POINTS[@]}"; do
                IFS=':' read -r share_path mount_point <<< "$mount_config"
                unmount_share "$mount_point"
            done
            ;;
        status)
            echo "État des montages NAS :"
            echo "======================="
            for mount_config in "${MOUNT_POINTS[@]}"; do
                IFS=':' read -r share_path mount_point <<< "$mount_config"
                if mountpoint -q "$mount_point"; then
                    echo "✓ $mount_point : MONTÉ"
                else
                    echo "✗ $mount_point : NON MONTÉ"
                fi
            done
            ;;
        *)
            echo "Usage: $0 {mount|unmount|status}"
            echo ""
            echo "Commandes :"
            echo "  mount   - Monte tous les partages NAS"
            echo "  unmount - Démonte tous les partages NAS"  
            echo "  status  - Affiche l'état des montages"
            exit 1
            ;;
    esac
}

# Exécution
main "$@"
```

### Rendre le script exécutable

```bash
sudo chmod +x /usr/local/bin/mount-nas.sh
```

---

## 🔧 Adaptation du Script à Votre Configuration

### 1. Modifier l'adresse IP du NAS

Remplacez `192.168.1.100` par l'adresse IP de votre NAS :

```bash
NAS_IP="192.168.1.100"  # Votre IP du NAS
```

### 2. Configurer vos partages

Modifiez le tableau `MOUNT_POINTS` selon vos besoins :

```bash
MOUNT_POINTS=(
    "//192.168.1.100/MonPartage1:/mnt/nas/partage1"
    "//192.168.1.100/MonPartage2:/mnt/nas/partage2"
)
```

**Format :** `"//IP_NAS/NomDuPartage:/chemin/de/montage/local"`

### 3. Adapter l'UID et GID

Par défaut, le script utilise `uid=1000,gid=1000` (premier utilisateur créé).

Pour vérifier votre UID/GID :

```bash
id
# Résultat : uid=1000(johndoe) gid=1000(johndoe) ...
```

Si votre UID est différent, modifiez dans le script :

```bash
MOUNT_OPTIONS="credentials=$CREDENTIALS,uid=VOTRE_UID,gid=VOTRE_GID,..."
```

---

## 🚀 Utilisation

### Monter les partages

```bash
sudo mount-nas.sh mount
```

### Démonter les partages

```bash
sudo mount-nas.sh unmount
```

### Vérifier l'état

```bash
sudo mount-nas.sh status
```

**Exemple de sortie :**

```
État des montages NAS :
=======================
✓ /mnt/nas/documents : MONTÉ
✓ /mnt/nas/photos : MONTÉ
✗ /mnt/nas/videos : NON MONTÉ
```

---

## 🐟 Utilisation avec Fish Shell

Sous CachyOS, le shell par défaut est **Fish** au lieu de Bash. Le script utilise le shebang `#!/usr/bin/env bash` qui le rend compatible.

### Créer un alias Fish (optionnel)

Pour simplifier l'utilisation, créez un alias dans Fish :

```bash
kate ~/.config/fish/config.fish
```

**Ajouter ces lignes :**

```fish
# Alias pour le montage NAS
alias nas-mount='sudo mount-nas.sh mount'
alias nas-unmount='sudo mount-nas.sh unmount'
alias nas-status='sudo mount-nas.sh status'
```

**Recharger la configuration :**

```bash
source ~/.config/fish/config.fish
```

**Utilisation simplifiée :**

```bash
nas-mount    # Monte tous les partages
nas-unmount  # Démonte tous les partages
nas-status   # Affiche l'état
```

---

## 📊 Logs et Dépannage

### Consulter les logs

```bash
sudo tail -f /var/log/nas-mount.log
```

### Problèmes courants

#### Le NAS n'est pas accessible

```bash
# Vérifier la connectivité réseau
ping 192.168.1.100

# Vérifier que le NAS est bien démarré
```

#### Erreur "Permission denied"

```bash
# Vérifier les permissions du fichier credentials
ls -l /etc/nas-credentials

# Doit afficher : -rw------- 1 root root
```

#### Le montage semble vide

- Attendre quelques secondes après le montage
- Vérifier que le partage existe bien sur le NAS
- Vérifier les droits d'accès sur le NAS

#### Démontage impossible (device is busy)

```bash
# Identifier les processus utilisant le montage
sudo lsof /mnt/nas/documents

# Forcer le démontage
sudo umount -f /mnt/nas/documents
```

---

## 🎨 Intégration KDE (optionnel)

### Créer un raccourci bureau

Créez un fichier `~/Bureau/NAS-Mount.desktop` :

```ini
[Desktop Entry]
Type=Application
Name=Monter NAS
Comment=Monte les partages NAS
Icon=folder-network
Exec=konsole -e sudo mount-nas.sh mount
Terminal=false
```

### Ajouter au menu KDE

```bash
kwriteconfig5 --file ~/.local/share/applications/nas-mount.desktop \
  --group "Desktop Entry" \
  --key Type Application
```

---

## 📚 Options de Montage Expliquées

| Option | Description |
|--------|-------------|
| `credentials=$CREDENTIALS` | Fichier contenant username/password |
| `uid=1000,gid=1000` | Propriétaire des fichiers montés |
| `iocharset=utf8` | Support des caractères spéciaux (accents) |
| `file_mode=0644` | Permissions des fichiers (rw-r--r--) |
| `dir_mode=0755` | Permissions des dossiers (rwxr-xr-x) |
| `cache=strict` | Mode de cache pour de meilleures performances |
| `rsize=65536,wsize=65536` | Taille des buffers de lecture/écriture |

!!! tip "Optimisation pour le streaming"
    Les options `rsize` et `wsize` à 65536 sont optimales pour le streaming vidéo (Jellyfin, Plex, etc.)

---

## ✅ Checklist de Configuration

- [ ] Installer `cifs-utils` et `smbclient`
- [ ] Créer `/etc/nas-credentials` avec les bons identifiants
- [ ] Vérifier les permissions du fichier credentials (600)
- [ ] Créer le script `/usr/local/bin/mount-nas.sh`
- [ ] Rendre le script exécutable (`chmod +x`)
- [ ] Adapter l'IP du NAS dans le script
- [ ] Configurer les partages dans `MOUNT_POINTS`
- [ ] Tester le montage avec `sudo mount-nas.sh mount`
- [ ] (Optionnel) Créer des alias Fish pour simplifier l'usage