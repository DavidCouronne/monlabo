# 🧪 Mon Labo

<div class="hero" markdown>

Bienvenue dans mon laboratoire personnel de configuration et de documentation !

**Un aide-mémoire vivant** pour centraliser mes procédures, configurations système et astuces d'automatisation. Tout est ici, prêt à être réutilisé pour une réinstallation ou un nouveau déploiement.

[:octicons-rocket-24: Commencer](#commencer){ .md-button .md-button--primary }
[:octicons-mark-github-16: Voir sur GitHub](https://github.com/DavidCouronne/monlabo){ .md-button }

</div>

---

## 🎯 Philosophie

<div class="grid cards" markdown>

-   :material-check-circle:{ .lg .middle } **Pratique avant tout**

    ---

    Chaque page documente une procédure testée et fonctionnelle sur mon environnement

-   :material-code-tags:{ .lg .middle } **Exemples génériques**

    ---

    Les configurations utilisent des valeurs d'exemple (utilisateurs fictifs, chemins standards)

-   :material-shield-check:{ .lg .middle } **Sécurité**

    ---

    Aucun secret, mot de passe ou information sensible n'est présent ici

-   :material-rocket-launch:{ .lg .middle } **Évolutif**

    ---

    La documentation s'enrichit au fil de mes expérimentations et besoins

</div>

---

## 🛠️ Mon Environnement de Référence

**Distribution**
:   [**CachyOS**](https://cachyos.org/)  
    Distribution basée sur Arch Linux avec noyau optimisé pour de meilleures performances

**Environnement de bureau**
:   **KDE Plasma 6**  
    Interface moderne et hautement personnalisable, avec support Wayland amélioré

**Shell**
:   **Fish**  
    Shell interactif moderne avec autocomplétion intelligente, remplace Bash par défaut sur CachyOS

**Gestionnaire de paquets**
:   **pacman** + **paru**  
    `paru` comme alternative à `yay` pour accéder aux paquets AUR

**Dépôts supplémentaires**
:   **Chaotic-AUR**  
    Accès aux paquets AUR pré-compilés pour des installations plus rapides

**Éditeur système**
:   **Kate**  
    Éditeur de texte KDE pour tous les fichiers de configuration (fini `nano` !)

**IDE**
:   **Visual Studio Code**  
    Pour les projets de développement et le code plus complexe

**Conteneurisation**
:   **Docker** + **Docker Compose**  
    Pour le déploiement et la gestion des services auto-hébergés

**Gestionnaire Python**
:   **UV**  
    Alternative moderne et ultra-rapide à pip/poetry pour la gestion des dépendances Python

**Solution de backup**
:   **Duplicati**  
    Backups incrémentiels chiffrés (AES-256) vers OneDrive avec déduplication

!!! tip "Pourquoi cet environnement ?"
    **KDE Plasma** reste constant même si je change de distribution. **Kate** est mon éditeur privilégié pour les fichiers de configuration (plus besoin de `nano` !), tandis que **VS Code** gère mes projets plus complexes. **Fish** offre une autocomplétion intelligente qui accélère considérablement le travail en ligne de commande.

---

## 📚 Sections du Site

### :material-hammer-wrench: Installation & Configuration

<div class="grid cards" markdown>

-   **[Outils de Base](installation/outils-base.md)**

    ---

    Installation de Chaotic-AUR, VS Code, UV, Docker et utilitaires essentiels
    
    [:octicons-arrow-right-24: Consulter](installation/outils-base.md)

-   **[Montage NAS](installation/montage-nas.md)**

    ---

    Configuration du montage automatique du NAS Synology avec Fish shell
    
    [:octicons-arrow-right-24: Consulter](installation/montage-nas.md)

-   **Sécurité & Chiffrement** :lock:

    ---

    Montage LUKS au démarrage, gestion des clés
    
    :octicons-clock-24: *À venir*

-   **Personnalisation KDE** :art:

    ---

    Thèmes, raccourcis, workflow optimisé
    
    :octicons-clock-24: *À venir*

</div>

### :material-backup-restore: Backup & Restauration

<div class="grid cards" markdown>

-   **[Backup Immich](backup/immich.md)**

    ---

    Sauvegarde PostgreSQL et restauration des photos/vidéos
    
    [:octicons-arrow-right-24: Consulter](backup/immich.md)

-   **[Backup Jellyfin](backup/jellyfin.md)**

    ---

    Sauvegarde configuration SQLite et métadonnées média
    
    [:octicons-arrow-right-24: Consulter](backup/jellyfin.md)

-   **Stratégie globale** :material-strategy:

    ---

    Règle 3-2-1, planification, monitoring
    
    [:octicons-arrow-right-24: Vue d'ensemble](backup/index.md)

</div>

!!! info "Stratégie 3-2-1 appliquée"
    Tous mes backups suivent la règle **3-2-1** : 3 copies, 2 supports différents, 1 copie hors site (OneDrive chiffré)

### :material-docker: Docker & Auto-hébergement

<div class="grid cards" markdown>

-   :material-image-multiple: **Immich**

    ---

    Serveur de gestion de photos et vidéos auto-hébergé
    
    :octicons-clock-24: *Documentation à venir*

-   :material-play-box-multiple: **Jellyfin**

    ---

    Serveur multimédia personnel pour films et séries
    
    :octicons-clock-24: *Documentation à venir*

-   :material-cloud: **Nextcloud**

    ---

    Cloud privé et synchronisation de fichiers
    
    :octicons-clock-24: *À venir*

-   :material-dots-horizontal: **Autres services**

    ---

    Au fur et à mesure de mes besoins
    
    :octicons-clock-24: *À venir*

</div>

### :material-code-braces: Automatisation & Scripts

<div class="grid cards" markdown>

-   :material-bash: **Scripts Shell**

    ---

    Tâches système, maintenance, sauvegardes automatiques
    
    :octicons-clock-24: *À venir*

-   :material-language-python: **Scripts Python**

    ---

    Traitement de données, automatisations avancées
    
    :octicons-clock-24: *À venir*

-   :material-lightbulb: **Exemples pratiques**

    ---

    Cas d'usage réels et réutilisables
    
    :octicons-clock-24: *À venir*

</div>

---

## 🚀 Commencer

=== "Nouvelle installation"

    **Vous installez une nouvelle machine ?** Suivez ce parcours :
    
    1. :material-numeric-1-circle:{ .primary } **[Installer les outils de base](installation/outils-base.md)**
       
       Chaotic-AUR, VS Code, Docker, utilitaires essentiels
    
    2. :material-numeric-2-circle:{ .primary } **[Configurer le NAS](installation/montage-nas.md)**
       
       Montage automatique et gestion avec Fish shell
    
    3. :material-numeric-3-circle:{ .primary } **[Mettre en place les backups](backup/index.md)**
       
       Configuration de Duplicati et scripts automatiques
    
    4. :material-numeric-4-circle:{ .primary } **Déployer vos services**
       
       Docker Compose pour vos applications *(à venir)*

=== "Déjà configuré"

    **Vous avez déjà votre environnement ?** Explorez selon vos besoins :
    
    - :material-backup-restore: **[Backups](backup/index.md)** - Configurer ou améliorer votre stratégie de sauvegarde
    - :material-docker: **Docker** - Déployer de nouveaux services *(à venir)*
    - :material-code-braces: **Scripts** - Automatiser vos tâches répétitives *(à venir)*

=== "Recherche rapide"

    **Vous cherchez quelque chose de précis ?**
    
    Utilisez la recherche (++ctrl+k++ ou ++cmd+k++) pour trouver rapidement :
    
    - Une commande spécifique
    - Une configuration
    - Un service Docker
    - Un script d'automatisation

---

## 💡 Astuces Rapides

!!! tip "Raccourcis utiles"
    - ++ctrl+k++ ou ++cmd+k++ : Ouvrir la recherche
    - ++ctrl+m++ : Basculer le menu latéral
    - ++ctrl+shift+f++ : Recherche dans la page
    - ++home++ : Retour en haut de page

!!! example "Commandes Fish favorites"
    ```fish
    # Alias configurés
    backup-immich     # Backup base de données Immich
    backup-jellyfin   # Backup configuration Jellyfin
    restore-immich    # Restauration complète Immich
    restore-jellyfin  # Restauration complète Jellyfin
    ```

---

## 📖 Réutiliser cette Documentation

Ce site est **public** mais reste une documentation **personnelle**. Vous êtes libre de vous en inspirer !

!!! warning "Points d'attention"
    - Les configurations sont adaptées à **CachyOS + KDE Plasma + Fish**
    - Certaines procédures nécessitent des **ajustements** pour votre environnement
    - **Testez toujours** dans un environnement non-critique avant production
    - Les chemins et utilisateurs sont **génériques** - adaptez-les à votre cas

!!! success "Réutilisation encouragée"
    N'hésitez pas à fork le [dépôt GitHub](https://github.com/DavidCouronne/monlabo) et à adapter la documentation à vos besoins !

---

## 🔄 Dernières Mises à Jour

??? info "Historique récent"
    - **Backup Jellyfin** : Script complet de backup/restauration avec systemd
    - **Backup Immich** : Stratégie de backup PostgreSQL avec Duplicati
    - **Montage NAS** : Configuration Fish shell et systemd
    - **Outils de base** : Installation Chaotic-AUR, Docker, VS Code, UV

---

<div class="center" markdown>

**Bonne exploration ! 🚀**

*Cette documentation évolue constamment. Revenez régulièrement pour découvrir de nouvelles sections.*

[:octicons-mark-github-16: Contribuer sur GitHub](https://github.com/DavidCouronne/monlabo){ .md-button }

</div>