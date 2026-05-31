---

## Jellyfin — Sauvegardes automatiques

Jellyfin intègre un système de backup directement dans son interface web depuis la version 10.11 — **pas besoin de toucher au terminal**.

### Ce que le backup Jellyfin contient

| Contenu | Inclus |
|---|---|
| Base de données (comptes, historique, playlists) | ✅ Toujours |
| Métadonnées et affiches | ✅ Optionnel |
| Sous-titres extraits | ✅ Optionnel |

### Activer les sauvegardes automatiques

1. Ouvre Jellyfin dans Firefox : **http://localhost:8096**
2. Clique sur l'icône ☰ en haut à gauche → **Tableau de bord**
3. Dans le menu de gauche : **Sauvegarde**
4. Configure les paramètres :
    - **Activer les sauvegardes automatiques** : ✅
    - **Nombre de sauvegardes à conserver** : `7`
    - **Inclure les métadonnées** : selon ton choix (attention : les métadonnées peuvent représenter plusieurs centaines de Mo)
5. Clique **Enregistrer**

!!! info "Où sont stockées les sauvegardes ?"
    Les sauvegardes Jellyfin sont automatiquement dans `~/.local/share/jellyfin/config/data/backups/`. Elles sont déjà dans ton dossier de données Jellyfin.

### Créer une sauvegarde immédiatement

Pour ne pas attendre la nuit :

1. Va dans **Tableau de bord → Sauvegarde**
2. Clique **Créer une sauvegarde maintenant**

Un nouveau fichier apparaîtra dans `~/.local/share/jellyfin/config/data/backups/`.

---

## Copier les sauvegardes Jellyfin sur clé USB

Connecte ta clé USB, ouvre **Dolphin** et navigue jusqu'à :

```
.local/share/jellyfin/config/data/backups/
```

Ce dossier est **caché** car il commence par un point. Pour l'afficher dans Dolphin : **Affichage → Afficher les fichiers cachés** (ou raccourci **Alt+.**).

Copie le dossier `backups/` sur ta clé USB.

---

## Restaurer Jellyfin

!!! danger "Lis ceci avant de restaurer"
    La restauration **remplace toutes les données actuelles**. C'est irréversible. Ne fais ça qu'en cas de vrai problème.

!!! warning "Jellyfin doit être arrêté avant la restauration"
    Si Jellyfin tourne pendant la restauration, la base de données est verrouillée et la restauration peut échouer ou corrompre les données.

### Si Jellyfin fonctionne encore

1. Ouvre **http://localhost:8096** dans Firefox
2. Va dans **Tableau de bord → Sauvegarde**
3. Tu vois la liste des sauvegardes disponibles avec leur date
4. Clique **Restaurer** à côté de celle que tu veux
5. Jellyfin s'arrête automatiquement, restaure, puis redémarre

### Si tu dois repartir de zéro (nouveau PC ou réinstallation)

1. Suis le [guide d'installation Jellyfin](jellyfin-podman.md) jusqu'à l'Étape 3 (activation des services)
2. **Avant** d'ouvrir l'interface web pour la première fois, copie tes fichiers de sauvegarde :

```bash
cp /chemin/vers/ta/cle/backups/* ~/.local/share/jellyfin/config/data/backups/
```

3. Ouvre **http://localhost:8096** et suis l'assistant de configuration
4. Une fois connecté, va dans **Tableau de bord → Sauvegarde** et restaure le fichier voulu

---

## Résumé — Tableau comparatif des sauvegardes

| Service | Backup automatique | Où sont les fichiers | Restauration |
|---|---|---|---|
| **Navidrome** | ✅ Tous les jours à 2h | `~/Documents/Sauvegardes/navidrome/` | Terminal (service arrêté) |
| **Immich** | ✅ Tous les jours à 2h | `~/.local/share/immich/data/backups/` | Interface web — clic-clic |
| **Jellyfin** | ✅ Tous les jours | `~/.local/share/jellyfin/config/data/backups/` | Interface web — clic-clic |
