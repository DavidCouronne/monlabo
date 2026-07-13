Voici un tutoriel complet pour gérer vos configurations système utilisateur (**Systemd**, **Quadlets**, **Mounts**, etc.) sur **Project Bluefin** avec **Chezmoi**.

Sur une distribution immuable/atomique comme Bluefin, l'utilisation de Systemd en mode utilisateur (`--user`) et de Quadlets (les conteneurs Podman gérés par Systemd) dans le répertoire personnel (`~/.config/containers/systemd/`) est la méthode idéale. Chezmoi va vous permettre de centraliser tout cela proprement.

---

## 1. Installation de Chezmoi

Sur Bluefin, Homebrew est préinstallé et configuré par défaut pour l'utilisateur. C'est le moyen le plus simple et recommandé pour installer et maintenir Chezmoi à jour sans toucher à l'image du système.

Ouvrez votre terminal et lancez :

```bash
brew install chezmoi

```

---

## 2. Initialisation de Chezmoi

Si vous débutez de zéro, initialisez votre dépôt local (qui sera situé dans `~/.local/share/chezmoi`) :

```bash
chezmoi init

```

*Note : Si vous avez déjà un dépôt Git existant sur GitHub/GitLab, vous ferez plutôt `chezmoi init https://github.com/votre-nom/dotfiles.git`.*

---

## 3. Ajout et gestion des Services Systemd, Quadlets et Mounts

Pour que vos services s'activent correctement dès que Chezmoi les déploie, nous allons utiliser une fonctionnalité puissante : les scripts `run_onchange_`.

### Étape 3.1 : Ajouter vos fichiers à Chezmoi

Ajoutez vos fichiers de configuration actuels à la gestion de Chezmoi.

```bash
# Pour vos services Systemd classiques (User)
chezmoi add ~/.config/systemd/user/mon-service.service

# Pour vos Quadlets (Conteneurs Podman)
chezmoi add ~/.config/containers/systemd/mon-conteneur.container

# Pour vos Mounts Systemd (User)
chezmoi add ~/.config/systemd/user/data-partage.mount

```

### Étape 3.2 : Automatiser le rechargement de Systemd (Le déclencheur)

Pour éviter de devoir taper `systemctl --user daemon-reload` manuellement à chaque mise à jour, nous allons créer un script automatisé dans Chezmoi.

1. Entrez dans le répertoire source de Chezmoi :
```bash
chezmoi cd

```


2. Créez un script nommé `run_onchange_after_reload-systemd.sh.tmpl` (le préfixe `run_onchange_` indique à Chezmoi de ne l'exécuter que si les fichiers Systemd ont changé) :
```bash
nano run_onchange_after_reload-systemd.sh.tmpl

```


3. Collez-y le contenu suivant :
```bash
#!/bin/bash
# Hash des fichiers pour détecter les changements :
# {{ include (glob (joinPath .chezmoi.homeDir ".config/systemd/user/*")) | sha256sum }}
# {{ include (glob (joinPath .chezmoi.homeDir ".config/containers/systemd/*")) | sha256sum }}

echo "🔄 Changement détecté : Rechargement du manager Systemd & Quadlets..."
systemctl --user daemon-reload

```


4. Quittez l'éditeur (`Ctrl+X` puis `Y`) et tapez `exit` pour revenir à votre terminal classique.

---

## 4. Gestion des Secrets (Clés API, Mots de passe de conteneurs...)

Hors de question de pousser vos mots de passe en clair sur un dépôt Git (surtout s'il est public). La méthode moderne et robuste consiste à utiliser l'utilitaire **Age** (compatible nativement avec Chezmoi).

### Étape 4.1 : Générer une clé de chiffrement

```bash
# Créer le dossier de configuration de chezmoi s'il n'existe pas
mkdir -p ~/.config/chezmoi

# Générer une clé Age
age-keygen -o ~/.config/chezmoi/key.txt

```

Affichez votre clé publique (elle commence par `age1...`) :

```bash
age-keygen -y ~/.config/chezmoi/key.txt

```

### Étape 4.2 : Configurer Chezmoi pour utiliser Age

Créez ou modifiez le fichier `~/.config/chezmoi/chezmoi.toml` :

```toml
encryption = "age"

[age]
    identity = "~/.config/chezmoi/key.txt"
    recipient = "votre_clé_publique_age1..."

```

### Étape 4.3 : Ajouter un fichier contenant des secrets

Si votre Quadlet ou service a besoin d'un fichier d'environnement contenant des tokens (ex: `~/.config/mon-service.env`), ajoutez-le en demandant explicitement le chiffrement :

```bash
chezmoi add --encrypt ~/.config/mon-service.env

```

Dans votre répertoire Git, ce fichier sera stocké sous le nom `encrypted_dot_mon-service.env` et sera totalement illisible sans votre clé privée.

---

## 5. Sauvegarde (Backup vers Git)

Pour sauvegarder l'état actuel de vos configurations vers votre dépôt distant (GitHub, GitLab, etc.) :

```bash
# Étape 1 : Aller dans le répertoire source de chezmoi
chezmoi cd

# Étape 2 : Lier votre dépôt Git distant (à ne faire la première fois)
git remote add origin git@github.com:votre-utilisateur/votre-repo-dotfiles.git

# Étape 3 : Commit et Push
git add .
git commit -m "Ajout des services systemd, quadlets et gestion des secrets"
git push -u origin main
exit

```

---

## 6. Restauration sur une nouvelle machine Bluefin

Vous réinstallez votre PC ou configurez une nouvelle machine Bluefin ? La restauration se fait en une seule ligne de commande.

1. Installez Chezmoi via Homebrew sur la nouvelle machine :
```bash
brew install chezmoi

```


2. Remettez votre clé secrète `key.txt` dans `~/.config/chezmoi/key.txt` (via une clé USB sécurisée ou votre gestionnaire de mots de passe).
3. Lancez la restauration complète :
```bash
chezmoi init --apply https://github.com/votre-utilisateur/votre-repo-dotfiles.git

```



Chezmoi va cloner le dépôt, déchiffrer vos secrets grâce à la clé, placer les services Systemd, Mounts et Quadlets à leur place exacte, puis déclencher automatiquement le `systemctl --user daemon-reload` grâce au script créé à l'étape 3. Vos conteneurs Podman (Quadlets) et vos services seront prêts à être démarrés !