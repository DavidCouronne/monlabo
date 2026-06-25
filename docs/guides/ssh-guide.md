# Guide SSH

SSH (Secure Shell) est un protocole permettant de se connecter à distance à un appareil de manière sécurisée et chiffrée.

---

## Prérequis

- Un terminal (Terminal sur macOS, bash/zsh sur Linux)
- Un client SSH (intégré nativement sur macOS et Linux)
- L'adresse IP ou le nom d'hôte de l'appareil cible
- Un compte utilisateur sur l'appareil cible

Vérifier que SSH est disponible :

```bash
ssh -V
```

---

## Mettre en place une connexion SSH

### 1. Générer une paire de clés SSH

```bash
ssh-keygen -t ed25519 -C "mon-commentaire"
```

- `-t ed25519` : algorithme recommandé (plus sécurisé et performant que RSA)
- `-C` : commentaire optionnel pour identifier la clé (ex. : adresse email ou nom de machine)

Appuyer sur `Entrée` pour accepter le chemin par défaut (`~/.ssh/id_ed25519`).
Définir une passphrase ou laisser vide.

Deux fichiers sont créés :

| Fichier | Rôle |
|---|---|
| `~/.ssh/id_ed25519` | Clé privée — **ne jamais partager** |
| `~/.ssh/id_ed25519.pub` | Clé publique — à copier sur l'appareil cible |

### 2. Copier la clé publique sur l'appareil cible

=== "Méthode automatique"

    ```bash
    ssh-copy-id utilisateur@adresse-ip
    ```

=== "Méthode manuelle"

    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```

    Copier la sortie, puis sur l'appareil cible :

    ```bash
    mkdir -p ~/.ssh
    echo "COLLER_LA_CLE_ICI" >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```

### 3. Se connecter

```bash
ssh utilisateur@adresse-ip
```

Exemple :

```bash
ssh alice@192.168.1.10
```

---

## Connexion selon l'appareil cible

### Vers un Mac

#### Activer SSH sur le Mac cible

Sur le Mac cible : **Réglages Système → Général → Partage → Connexion à distance** → Activer.

```bash
# Vérifier que le service est actif
sudo systemsetup -getremotelogin
```

Se connecter :

```bash
ssh utilisateur@adresse-ip-du-mac
# ou avec le nom Bonjour
ssh utilisateur@nom-du-mac.local
```

!!! note "Nom d'utilisateur"
    Utiliser le nom de compte court visible dans **Réglages Système → Utilisateurs et groupes**.

### Vers un Linux

#### Installer et activer le serveur SSH

=== "Debian / Ubuntu"

    ```bash
    sudo apt update && sudo apt install openssh-server
    sudo systemctl enable --now ssh
    ```

=== "Fedora / RHEL"

    ```bash
    sudo dnf install openssh-server
    sudo systemctl enable --now sshd
    ```

=== "Arch Linux"

    ```bash
    sudo pacman -S openssh
    sudo systemctl enable --now sshd
    ```

Vérifier le statut :

```bash
sudo systemctl status ssh
```

Se connecter :

```bash
ssh utilisateur@adresse-ip
```

### Vers un NAS Synology

#### Activer SSH sur le NAS

1. Ouvrir **DSM** (interface web du NAS)
2. Aller dans **Panneau de configuration → Terminal et SNMP**
3. Cocher **Activer le service SSH**
4. Choisir un port (défaut : `22`, recommandé : changer pour un port > 1024)
5. Cliquer sur **Appliquer**

Se connecter :

```bash
ssh admin@adresse-ip-du-nas
# Avec un port personnalisé
ssh -p 2222 admin@adresse-ip-du-nas
```

!!! warning "Utilisateur `root` désactivé par défaut"
    Sur Synology, se connecter avec le compte `admin` ou un compte appartenant au groupe `administrators`. L'accès root est possible via `sudo -i` une fois connecté.

---

## Connexions multiples sur plusieurs appareils

### Configurer le fichier `~/.ssh/config`

Le fichier `~/.ssh/config` permet de définir des alias et des paramètres par appareil, évitant de retaper l'IP, l'utilisateur et le port à chaque fois.

Créer ou éditer le fichier :

```bash
nano ~/.ssh/config
```

Exemple de configuration avec plusieurs appareils :

```
# Mac au bureau
Host mac-bureau
    HostName 192.168.1.10
    User alice
    IdentityFile ~/.ssh/id_ed25519

# Serveur Linux de prod
Host serveur-linux
    HostName 192.168.1.20
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519

# NAS Synology
Host nas
    HostName 192.168.1.30
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

# Règle globale (s'applique à tous les hôtes)
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

Se connecter ensuite simplement par :

```bash
ssh mac-bureau
ssh serveur-linux
ssh nas
```

### Utiliser le gestionnaire de clés (ssh-agent)

Pour ne pas retaper la passphrase à chaque connexion :

```bash
# Démarrer l'agent
eval "$(ssh-agent -s)"

# Ajouter la clé
ssh-add ~/.ssh/id_ed25519
```

Sur macOS, ajouter dans `~/.ssh/config` pour une persistance au redémarrage :

```
Host *
    AddKeysToAgent yes
    UseKeychain yes
```

### Connexion en rebond (ProxyJump)

Pour accéder à une machine interne via un serveur intermédiaire (bastion) :

```bash
ssh -J utilisateur@bastion utilisateur@machine-interne
```

Ou dans `~/.ssh/config` :

```
Host machine-interne
    HostName 10.0.0.5
    User alice
    ProxyJump bastion

Host bastion
    HostName mon-serveur-public.com
    User alice
```

---

## Supprimer une connexion SSH

### Supprimer une clé publique d'un appareil distant

Sur l'appareil distant, éditer `~/.ssh/authorized_keys` et supprimer la ligne correspondant à la clé à révoquer :

```bash
nano ~/.ssh/authorized_keys
```

Chaque ligne correspond à une clé publique. Supprimer la ligne souhaitée et sauvegarder.

### Supprimer une entrée du fichier `~/.ssh/config`

Éditer `~/.ssh/config` et supprimer le bloc `Host` correspondant :

```bash
nano ~/.ssh/config
```

### Supprimer une clé de `known_hosts` (empreinte d'hôte)

Si l'empreinte d'un hôte a changé (réinstallation, changement d'IP) :

```bash
# Supprimer une entrée par nom ou IP
ssh-keygen -R adresse-ip
ssh-keygen -R nom-hôte

# Exemple
ssh-keygen -R 192.168.1.10
ssh-keygen -R mac-bureau
```

### Supprimer une paire de clés locale

```bash
rm ~/.ssh/id_ed25519
rm ~/.ssh/id_ed25519.pub
```

!!! danger
    Supprimer la clé privée localement la rend définitivement inutilisable. S'assurer d'avoir d'autres moyens d'accès avant de supprimer.

---

## Sécuriser le serveur SSH

Éditer `/etc/ssh/sshd_config` sur l'appareil cible :

```
# Désactiver l'authentification par mot de passe
PasswordAuthentication no

# Désactiver la connexion root directe
PermitRootLogin no

# Restreindre les utilisateurs autorisés
AllowUsers alice bob

# Changer le port par défaut (optionnel)
Port 2222
```

Redémarrer le service après modification :

```bash
# Linux
sudo systemctl restart ssh

# macOS
sudo launchctl stop com.openssh.sshd
sudo launchctl start com.openssh.sshd
```

---

## Référence rapide des commandes

| Commande | Description |
|---|---|
| `ssh user@host` | Connexion SSH basique |
| `ssh -p 2222 user@host` | Connexion sur un port spécifique |
| `ssh -i ~/.ssh/ma_cle user@host` | Connexion avec une clé spécifique |
| `ssh-keygen -t ed25519` | Générer une nouvelle paire de clés |
| `ssh-copy-id user@host` | Copier la clé publique sur un hôte |
| `ssh-keygen -R host` | Supprimer l'empreinte d'un hôte |
| `ssh-add ~/.ssh/id_ed25519` | Ajouter une clé à l'agent SSH |
| `ssh -J bastion user@cible` | Connexion via un hôte intermédiaire |
