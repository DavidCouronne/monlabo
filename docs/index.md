# 🧪 Mon Labo

Bienvenue dans mon laboratoire personnel de configuration et de documentation !

## 👋 À propos

Ce site est avant tout un **aide-mémoire vivant** pour moi-même. Il centralise mes procédures, configurations système et astuces d'automatisation. Que ce soit pour une réinstallation complète ou la mise en place d'une nouvelle machine, tout est ici, prêt à être réutilisé.

Si vous utilisez également une distribution basée sur Arch Linux avec KDE Plasma, vous trouverez probablement des informations utiles pour votre propre setup !

!!! info "Philosophie de ce site"
    - **Pratique avant tout** : Chaque page documente une procédure testée et fonctionnelle
    - **Exemples génériques** : Les configurations utilisent des valeurs d'exemple (utilisateurs fictifs, chemins standards)
    - **Sécurité** : Aucun secret, mot de passe ou information sensible n'est présent ici
    - **Évolutif** : La documentation s'enrichit au fil de mes expérimentations

---

## 🛠️ Mon Environnement de Référence

Voici l'environnement sur lequel se basent toutes les configurations documentées :

| Composant | Choix actuel | Notes |
|-----------|--------------|-------|
| **Distribution** | CachyOS | Basée sur Arch Linux, noyau optimisé |
| **Environnement** | KDE Plasma | Toutes les configurations sont orientées KDE |
| **Gestionnaire de paquets** | `pacman` + `paru` | `paru` pour l'AUR (alternative à `yay`) |
| **Dépôts** | Chaotic-AUR activé | Accès aux paquets pré-compilés |
| **Éditeur système** | Kate | Pour tous les fichiers de configuration |
| **IDE** | Visual Studio Code | Pour les projets et le développement |
| **Conteneurisation** | Docker / Podman | Docker Compose ou Podman Compose |
| **Python** | UV | Gestionnaire de paquets moderne et rapide |
| **Scripting** | Bash + Python | Automatisation et scripts système |

!!! tip "Pourquoi cet environnement ?"
    **KDE Plasma** reste constant même si je change de distribution. Kate est mon éditeur privilégié pour les fichiers de configuration (plus besoin de `nano` !), tandis que VS Code gère mes projets plus complexes.

---

## 🗺️ Navigation du Site

### 🔧 Installation & Configuration

**Premiers pas essentiels pour mettre en place votre environnement :**

- **[Outils de Base](installation/outils-base.md)** : Installation de Chaotic-AUR, VS Code, UV, Docker/Podman et utilitaires système
- **Sécurité & Chiffrement** : Montage LUKS au démarrage, gestion des clés *(à venir)*
- **Personnalisation KDE** : Thèmes, raccourcis, workflow optimisé *(à venir)*

!!! success "Commencez ici !"
    Si c'est votre première visite ou une nouvelle installation, démarrez par **[Outils de Base](installation/outils-base.md)** pour installer tous les logiciels essentiels.

### 🐳 Docker & Auto-hébergement

Documentation de mes services conteneurisés :

- **📸 Immich** : Gestion et sauvegarde de photos *(à venir)*
- **🎬 Jellyfin** : Serveur multimédia personnel *(à venir)*
- **☁️ Nextcloud** : Cloud privé et synchronisation *(à venir)*
- **Autres services** : Au fur et à mesure de mes besoins

!!! note "Section en construction"
    Cette partie s'enrichira progressivement avec mes déploiements Docker/Podman.

### 🐍 Automatisation & Scripts

Mes scripts d'automatisation pour gagner du temps :

- **Scripts Shell** : Tâches système, maintenance, sauvegardes *(à venir)*
- **Scripts Python** : Traitement de données, automatisations avancées *(à venir)*
- **Exemples pratiques** : Cas d'usage réels et réutilisables

!!! note "À venir"
    Cette section sera alimentée au fil de mes besoins d'automatisation.

---

## 🚀 Commencer

**Nouvelle installation ?** Suivez ce parcours :

1. **[Installer les outils de base](installation/outils-base.md)** - Chaotic-AUR, VS Code, Docker, utilitaires
2. **Configurer la sécurité** - Chiffrement et montages automatiques *(à venir)*
3. **Déployer vos services** - Docker Compose pour vos applications *(à venir)*

**Déjà configuré ?** Explorez les sections [Docker](#) ou [Scripts](#) selon vos besoins du moment.

**Navigation** : Utilisez le menu latéral pour accéder rapidement aux différentes sections.

---

## 📝 Contribuer ou Réutiliser

Ce site est **public** mais reste une documentation **personnelle**. Vous êtes libre de vous en inspirer, mais gardez à l'esprit que :

- Les configurations sont adaptées à **mon environnement** (CachyOS + KDE Plasma)
- Certaines procédures peuvent nécessiter des **ajustements** pour votre setup
- **Testez toujours** dans un environnement non-critique avant de déployer en production