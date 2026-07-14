# Guide Git et Authentification Moderne

Ce guide rassemble les procédures essentielles pour configurer et utiliser Git au quotidien, de la configuration initiale de l'identité aux flux de travail (workflows) courants, en passant par les méthodes d'authentification moderne (SSH, HTTPS via GitHub CLI et Git Credential Manager).

---

## 1. Configuration initiale de Git

Avant toute interaction avec un dépôt Git, il est nécessaire de déclarer votre identité locale. Ces informations seront rattachées à chacun de vos commits.

Configurez votre nom d'usage et votre adresse e-mail :

```bash
git config --global user.name "Votre Nom"
git config --global user.email "votre.email@exemple.com"
```

Vous pouvez vérifier votre configuration globale à tout moment avec :

```bash
git config --list
```

---

## 2. Méthodes d'authentification sécurisée

GitHub et GitLab n'autorisent plus l'authentification par mot de passe simple en ligne de commande pour des raisons de sécurité. Deux alternatives modernes sont disponibles.

=== "Méthode A : Clés SSH (Recommandé)"

    L'utilisation de clés SSH permet d'interagir avec les dépôts distants de manière transparente sans aucune saisie répétitive de mot de passe.

    #### Étape 1 : Générer la paire de clés SSH
    Exécutez la commande suivante dans votre terminal (utilisez l'algorithme moderne Ed25519) :
    ```bash
    ssh-keygen -t ed25519 -C "votre.email@exemple.com"
    ```
    *Appuyez sur `Entrée` pour accepter l'emplacement par défaut (`~/.ssh/id_ed25519`). Il est recommandé de définir une passphrase robuste.*

    #### Étape 2 : Copier la clé publique
    Affichez et copiez le contenu de votre clé publique :
    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```
    *Copiez l'intégralité de la chaîne affichée (commençant par `ssh-ed25519`).*

    #### Étape 3 : Lier la clé publique à votre plateforme
    * **GitHub** : Rendez-vous dans *Settings* > *SSH and GPG keys* > *New SSH key* et collez la clé.
    * **GitLab** : Rendez-vous dans *User Settings* > *SSH Keys* et collez la clé.

=== "Méthode B : HTTPS et Outils d'aide"

    Si vous préférez utiliser le protocole HTTPS, vous devez utiliser des assistants d'authentification pour éviter de saisir vos jetons d'accès (tokens) à chaque transaction.

    #### Pour GitHub : Utilisation de la CLI GitHub (`gh`)
    La CLI GitHub automatise le processus d'authentification via un navigateur web.

    1. **Installer la CLI** :
       * macOS : `brew install gh`
       * Linux (Debian/Ubuntu) : `sudo apt install gh`
       * Linux (Arch/CachyOS) : `sudo pacman -S github-cli`

    2. **Lancer l'authentification** :
       ```bash
       gh auth login
       ```
    3. **Sélectionner les options suivantes** :
       * Account : **GitHub.com**
       * Preferred protocol : **HTTPS**
       * Authenticate Git : **Yes**
       * Authentication method : **Login with a web browser**
    4. Saisissez le code à usage unique fourni par le terminal dans la page web qui s'ouvre, puis validez.

    #### Pour GitLab : Utilisation de Git Credential Manager (GCM)
    GCM stocke de manière sécurisée vos jetons d'authentification.

    1. **S'assurer que GCM est installé** (souvent inclus avec Git sur Windows/macOS, ou installable via Homebrew sur macOS : `brew install --cask git-credential-manager`).
    2. **Déclencher la connexion** :
       Lancez une commande Git classique en utilisant le lien HTTPS de votre dépôt :
       ```bash
       git clone https://gitlab.com/votre-pseudo/votre-projet.git
       ```
    3. Une fenêtre de connexion GitLab s'ouvrira automatiquement dans votre navigateur pour valider la session.

---

## 3. Gestion des dépôts

### Initialisation d'un nouveau projet local
Pour commencer à suivre les fichiers d'un dossier local existant et le lier à un dépôt distant :

```bash
# Entrer dans le dossier du projet
cd mon-projet

# Initialiser le dépôt Git local
git init

# Lier le dépôt local au dépôt distant (remplacer par votre lien SSH ou HTTPS)
git remote add origin git@github.com:pseudo/mon-projet.git
```

### Clonage d'un dépôt distant existant
Pour récupérer une copie locale d'un projet déjà hébergé en ligne :

```bash
git clone git@github.com:pseudo/mon-projet.git
```

---

## 4. Gestion des branches

Les branches permettent de travailler sur des fonctionnalités de manière isolée sans affecter le code de la branche principale (`main`).

- **Créer une branche** (sans basculer dessus) :
  ```bash
  git branch nom-de-la-branche
  ```
- **Basculer sur une branche existante** :
  ```bash
  git switch nom-de-la-branche
  # (ou avec l'ancienne syntaxe : git checkout nom-de-la-branche)
  ```
- **Créer ET basculer sur une nouvelle branche** :
  ```bash
  git switch -c nom-de-la-branche
  # (ou : git checkout -b nom-de-la-branche)
  ```
- **Lister les branches locales** (la branche active est marquée par une étoile `*`) :
  ```bash
  git branch
  ```
- **Revenir à la branche principale** :
  ```bash
  git switch main
  ```

---

## 5. Flux de travail (Workflow) quotidien

Le cycle standard de modification et de sauvegarde de vos fichiers suit un enchaînement précis.

### Cycle classique de modification

| Commande | Rôle / Description |
| :--- | :--- |
| `git status` | Affiche l'état des fichiers (modifiés, non suivis, indexés). |
| `git diff` | Visualise les lignes de code modifiées avant de les indexer. |
| `git add <fichier>` | Ajoute un fichier spécifique à la zone de préparation (index). |
| `git add .` | Ajoute l'intégralité des fichiers modifiés à la zone de préparation. |
| `git commit -m "Message"` | Enregistre l'état de l'index dans l'historique local avec un message descriptif. |
| `git push origin <branche>` | Envoie les commits de votre branche locale vers le dépôt distant. |
| `git pull` | Récupère et fusionne les dernières modifications distantes dans votre branche locale. |

---

## 6. Commandes utiles et de secours

### Annuler des modifications locales non commitées
Si vous souhaitez réinitialiser un fichier modifié à son état du dernier commit :

```bash
git checkout -- nom-du-fichier.txt
```

### Modifier le dernier commit local
Pour corriger le message ou le contenu du tout dernier commit (avant de l'avoir poussé sur le dépôt distant) :

```bash
git commit --amend -m "Nouveau message du commit"
```

### Visualiser l'historique de manière condensée
Pour afficher l'historique des commits sous forme de graphe textuel simplifié :

```bash
git log --oneline --graph --decorate
```