# 🛠️ Installation des Outils de Base

Cette page regroupe l'installation de tous les outils essentiels pour un environnement de développement et d'administration système complet sous CachyOS.

---

## 📦 Chaotic-AUR

Chaotic-AUR fournit des paquets pré-compilés de l'AUR, ce qui évite de longues compilations.

!!! info "Documentation officielle"
    La procédure peut évoluer selon les versions. Consultez toujours la documentation officielle :  
    **[https://aur.chaotic.cx/docs](https://aur.chaotic.cx/docs)**

### Installation rapide

```bash
# Télécharger et installer la clé
sudo pacman-key --recv-key 3056513887B78AEB --keyserver keyserver.ubuntu.com
sudo pacman-key --lsign-key 3056513887B78AEB

# Installer le paquet de clés et miroirs
sudo pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst'
sudo pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'
```

### Configuration de pacman

Ajouter à la fin de `/etc/pacman.conf` (avec Kate bien sûr !) :

```ini
[chaotic-aur]
Include = /etc/pacman.d/chaotic-mirrorlist
```

### Mise à jour du système

```bash
sudo pacman -Syu
```

---

## 💻 Environnement de Développement

### Visual Studio Code

Installation directe depuis Chaotic-AUR :

```bash
paru -S visual-studio-code-bin
```

!!! tip "Extensions recommandées"
    - **Python** : Extension officielle Microsoft
    - **Docker** : Pour gérer les conteneurs depuis VS Code
    - **YAML** : Pour éditer les fichiers de configuration
    - **Markdown All in One** : Pour la documentation

### UV - Gestionnaire de Paquets Python

[UV](https://github.com/astral-sh/uv) est un gestionnaire de paquets Python moderne, ultra-rapide (écrit en Rust).

```bash
paru -S uv
```

**Utilisation rapide :**

```bash
# Créer un environnement virtuel
uv venv

# Activer l'environnement
source .venv/bin/activate

# Installer des paquets
uv pip install requests pandas

# Installer depuis requirements.txt
uv pip install -r requirements.txt
```

!!! tip "Pourquoi UV ?"
    - **10-100x plus rapide** que pip
    - Compatible avec les projets pip existants
    - Gestion intelligente du cache
    - Résolution de dépendances optimisée

### GitHub Desktop

Pratique pour gérer visuellement vos dépôts Git :

```bash
paru -S github-desktop-bin
```

---

## 🐳 Conteneurisation avec Docker

### Installation de Docker

```bash
# Installation de Docker et Docker Compose
sudo pacman -S docker docker-compose

# Démarrer et activer Docker au démarrage
sudo systemctl enable --now docker.service
```

### Configuration de l'utilisateur

Pour utiliser Docker sans `sudo` :

```bash
# Ajouter votre utilisateur au groupe docker
sudo usermod -aG docker $USER

# Appliquer les changements (se reconnecter ou utiliser)
newgrp docker

# Vérifier que ça fonctionne
docker run hello-world
```

!!! warning "Redémarrage nécessaire"
    Pour que les modifications de groupe soient complètement appliquées, il est préférable de **se déconnecter et se reconnecter**, ou de **redémarrer** le système.

### Commandes Docker de base

```bash
# Lister les conteneurs actifs
docker ps

# Lister tous les conteneurs (y compris arrêtés)
docker ps -a

# Lister les images
docker images

# Nettoyer les ressources inutilisées
docker system prune -a

# Voir l'utilisation disque
docker system df
```

---

## 🦭 Alternative : Podman

Podman est une alternative à Docker sans daemon, plus sécurisée (rootless par défaut).

### Installation

```bash
# Installation de Podman et outils associés
sudo pacman -S podman podman-compose podman-docker

# Pour la compatibilité avec docker-compose
paru -S podman-compose
```

### Configuration initiale

```bash
# Initialiser Podman pour l'utilisateur courant
podman system migrate

# Tester l'installation
podman run hello-world
```

### Podman Compose

Utilisation similaire à Docker Compose :

```bash
# Démarrer les services
podman-compose up -d

# Arrêter les services
podman-compose down

# Voir les logs
podman-compose logs -f
```

### Podman Play (Kubernetes YAML)

Podman peut aussi exécuter des fichiers Kubernetes :

```bash
# Depuis un fichier pod.yaml
podman play kube pod.yaml

# Arrêter le pod
podman play kube --down pod.yaml
```

!!! tip "Docker ou Podman ?"
    - **Docker** : Écosystème plus mature, meilleure compatibilité
    - **Podman** : Plus sécurisé (rootless), sans daemon, compatible Kubernetes
    - Les deux peuvent coexister sur le même système !

---

## 🔧 Utilitaires Système Essentiels

### Filelight - Analyse de l'espace disque

Visualisation graphique de l'utilisation du disque (intégré à KDE) :

```bash
sudo pacman -S filelight
```

**Utilisation :** Lancez depuis le menu KDE ou en tapant `filelight` dans le terminal.

### Autres outils indispensables

```bash
# Analyseur d'espace disque en CLI
sudo pacman -S ncdu

# Moniteur système avancé
sudo pacman -S htop btop

# Informations système détaillées
sudo pacman -S neofetch

# Recherche de fichiers ultra-rapide
sudo pacman -S fd

# Recherche dans le contenu des fichiers
sudo pacman -S ripgrep

# Gestionnaire de fichiers en CLI
sudo pacman -S ranger

# Visualisation de l'arborescence
sudo pacman -S tree

# Analyseur de logs
sudo pacman -S lnav
```

### Outils KDE Plasma utiles

```bash
# Gestionnaire de partitions
sudo pacman -S partitionmanager

# Éditeur de configuration système
sudo pacman -S systemsettings

# Gestionnaire d'archives
sudo pacman -S ark

# Visionneuse d'images
sudo pacman -S gwenview

# Lecteur vidéo
sudo pacman -S vlc
```

---

## 🎯 Récapitulatif d'Installation Rapide

Pour installer tout d'un coup (ou presque) :

```bash
# Développement
paru -S visual-studio-code-bin uv github-desktop-bin

# Conteneurisation (choisir Docker OU Podman, ou les deux)
sudo pacman -S docker docker-compose
# OU
sudo pacman -S podman podman-compose podman-docker

# Utilitaires système
sudo pacman -S filelight ncdu htop btop neofetch fd ripgrep ranger tree lnav

# Activer Docker si installé
sudo systemctl enable --now docker.service
sudo usermod -aG docker $USER
```

!!! success "Environnement prêt !"
    Une fois ces outils installés, vous disposez d'un environnement complet pour le développement, l'administration système et la conteneurisation.

---

## 📚 Ressources

- [Documentation Chaotic-AUR](https://aur.chaotic.cx/docs)
- [Documentation Docker](https://docs.docker.com/)
- [Documentation Podman](https://docs.podman.io/)
- [UV - Gestionnaire Python](https://github.com/astral-sh/uv)
- [Arch Wiki](https://wiki.archlinux.org/)