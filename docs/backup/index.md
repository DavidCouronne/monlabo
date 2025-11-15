# Backup et Restauration

Documentation des stratégies de sauvegarde et procédures de restauration pour l'ensemble de l'infrastructure.

## Stratégie globale : 3-2-1

!!! info "Règle 3-2-1"
    - **3** copies de vos données
    - Sur **2** supports différents
    - Dont **1** copie hors site

Notre implémentation :

- ✅ **Copie 1** : Données originales sur le serveur local
- ✅ **Copie 2** : Backups locaux (dumps, archives)
- ✅ **Copie 3** : Synchronisation cloud (OneDrive via Duplicati)
- ✅ **2 supports** : SSD local + Cloud storage
- ✅ **1 hors site** : OneDrive (chiffré)

## Services sauvegardés

<div class="grid cards" markdown>

-   :material-image-multiple: **[Immich](immich.md)**

    ---

    Backup de la base de données PostgreSQL et restauration des photos/vidéos
    
    [:octicons-arrow-right-24: Consulter](immich.md)

-   :material-play-box-multiple: **[Jellyfin](jellyfin.md)**

    ---

    Backup de la configuration, base de données SQLite et métadonnées
    
    [:octicons-arrow-right-24: Consulter](jellyfin.md)

-   :material-server: **Serveurs web** *(à venir)*

    ---

    Backup des configurations Nginx, certificats SSL
    
    [:octicons-arrow-right-24: Bientôt disponible](#)

-   :material-database: **Bases de données** *(à venir)*

    ---

    Procédures génériques pour MySQL, PostgreSQL, MongoDB
    
    [:octicons-arrow-right-24: Bientôt disponible](#)

</div>

## Outils utilisés

### Duplicati

**Backups incrémentiels chiffrés** vers OneDrive

- Chiffrement AES-256
- Déduplication
- Compression
- Interface web conviviale

### Scripts personnalisés

Scripts bash pour automatiser les dumps de bases de données et autres tâches spécifiques.

### Systemd timers

Planification native sous Linux pour déclencher les backups automatiques.

## Bonnes pratiques

!!! tip "Recommandations"
    1. **Tester régulièrement les restaurations** : Un backup non testé n'est pas un backup
    2. **Surveiller les logs** : Vérifier que les backups se sont bien déroulés
    3. **Rotation des backups** : Ne pas garder indéfiniment tous les backups
    4. **Documentation à jour** : Maintenir cette documentation synchronisée avec les configurations réelles
    5. **Notifications** : Configurer des alertes en cas d'échec

## Planning des backups

| Service | Fréquence | Heure | Rétention |
|---------|-----------|-------|-----------|
| Immich DB | Quotidien | 02:00 | 14 jours local |
| Immich fichiers | Quotidien | 03:00 | 30 jours cloud |
| Configurations | Hebdomadaire | Dimanche 04:00 | 90 jours |

## Checklist de restauration

Avant toute restauration :

- [ ] Identifier la date du backup à restaurer
- [ ] Vérifier l'intégrité des fichiers de backup
- [ ] Prévoir une fenêtre de maintenance
- [ ] Documenter l'état actuel du système
- [ ] Prévenir les utilisateurs si nécessaire
- [ ] Avoir un plan B en cas d'échec