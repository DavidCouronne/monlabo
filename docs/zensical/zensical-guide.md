# Guide Zensical — Référence personnelle

## Installation et commandes de base

```bash
# Installer Zensical
pip install zensical

# Nouveau projet
zensical new .

# Prévisualisation locale (http://localhost:8000)
zensical serve

# Build
zensical build
```

---

## Configuration zensical.toml

### Config de base complète (celle qui marche)

```toml
[project]
site_name = "Mon Site"
site_url = "https://monsite.netlify.app/"
site_description = "Description du site"
site_author = "Ton Nom"

# Dépôt Git (optionnel, affiche un lien en haut à droite)
[project.repo]
url = "https://github.com/user/repo"
name = "user/repo"
edit_uri = "edit/main/docs/"

# Thème : palettes clair/sombre
[[project.theme.palette]]
scheme = "default"
toggle.icon = "lucide/sun"
toggle.name = "Switch to dark mode"

[[project.theme.palette]]
scheme = "slate"
toggle.icon = "lucide/moon"
toggle.name = "Switch to light mode"

# Features activées
[project.theme]
features = [
    "navigation.footer",      # Boutons précédent/suivant en bas de page
    "content.code.copy",      # Bouton copier sur les blocs de code
    "content.tabs.link",      # Synchronise les onglets de même nom sur toute la page
    "navigation.instant",     # Navigation SPA (sans rechargement)
    "navigation.top",         # Bouton retour en haut
    "navigation.tracking",    # URL mise à jour avec l'ancre active
]

# Navigation explicite
nav = [
  { Accueil = "index.md" },
  { "Section 1" = [
    "section1/index.md",
    { "Page A" = "section1/page-a.md" },
    { "Page B" = "section1/page-b.md" },
  ]},
  { "Section 2" = "section2/index.md" },
]

# Extensions Markdown
[project.markdown_extensions.admonition]
[project.markdown_extensions.attr_list]
[project.markdown_extensions.md_in_html]
[project.markdown_extensions.def_list]
[project.markdown_extensions.footnotes]
[project.markdown_extensions.tables]
[project.markdown_extensions.abbr]

# Icônes Material + Octicons + FontAwesome (OBLIGATOIRE pour :material-...: etc.)
[project.markdown_extensions.pymdownx.emoji]
emoji_index = "zensical.extensions.emoji.twemoji"
emoji_generator = "zensical.extensions.emoji.to_svg"

# Coloration syntaxique
[project.markdown_extensions.pymdownx.highlight]
anchor_linenums = true
line_spans = "__span"
pygments_lang_class = true

[project.markdown_extensions.pymdownx.inlinehilite]
[project.markdown_extensions.pymdownx.snippets]
[project.markdown_extensions.pymdownx.superfences]

# Onglets (=== "label")
[project.markdown_extensions.pymdownx.tabbed]
alternate_style = true
combine_header_slug = true

# Admonitions dépliables (??)
[project.markdown_extensions.pymdownx.details]

# Formatage avancé
[project.markdown_extensions.pymdownx.critic]
[project.markdown_extensions.pymdownx.caret]
[project.markdown_extensions.pymdownx.mark]
[project.markdown_extensions.pymdownx.tilde]
[project.markdown_extensions.pymdownx.keys]
[project.markdown_extensions.pymdownx.smartsymbols]
[project.markdown_extensions.pymdownx.betterem]
smart_enable = "all"

# Listes de tâches cliquables
[project.markdown_extensions.pymdownx.tasklist]
custom_checkbox = true
clickable_checkbox = true
```

---

## Navigation

### Structure du site

Les pages sont dans `docs/`. La nav suit la structure TOML :

```toml
nav = [
  { Accueil = "index.md" },                          # Page simple
  { Installation = [                                  # Section avec sous-pages
    "installation/index.md",                          # Page index de la section
    { "Étape 1" = "installation/etape1.md" },
    { "Étape 2" = "installation/etape2.md" },
  ]},
  { "Lien externe" = "https://example.com" },        # Lien externe
]
```

!!! tip "Page index de section"
    Pour qu'une section ait sa propre page (ex. cliquer sur "Installation" ouvre une page), mets `installation/index.md` **en premier** dans la liste de la section. Activer aussi `navigation.indexes` dans les features si besoin.

### Features de navigation utiles

| Feature | Effet |
|---|---|
| `navigation.footer` | Boutons Précédent / Suivant en bas |
| `navigation.instant` | Navigation sans rechargement (SPA) |
| `navigation.tabs` | Sections de premier niveau en onglets dans le header |
| `navigation.sections` | Sections affichées comme groupes dans la sidebar |
| `navigation.expand` | Sidebar dépliée par défaut |
| `navigation.top` | Bouton retour en haut au scroll |
| `navigation.tracking` | URL mise à jour avec l'ancre active |
| `navigation.indexes` | Page liée directement à une section |
| `navigation.prune` | Réduit la taille HTML (gros sites) |

### Masquer des éléments par page (front matter)

```yaml
---
hide:
  - navigation   # cache la sidebar gauche
  - toc          # cache la table des matières
  - footer       # cache les boutons précédent/suivant
  - path         # cache le fil d'Ariane
---
```

---

## Syntaxe Markdown étendue

### Admonitions

```markdown
!!! note "Titre optionnel"
    Contenu de la note. L'indentation (4 espaces) est obligatoire.

!!! tip
    Sans titre personnalisé.

!!! warning "Attention !"
    Avertissement.

!!! danger
    Danger critique.

!!! info
    Information.

!!! success
    Succès.

!!! question
    Question / FAQ.

!!! example
    Exemple.

!!! quote
    Citation.
```

**Admonition dépliable** (nécessite `pymdownx.details`) :

```markdown
??? note "Cliquez pour voir"
    Contenu caché par défaut.

???+ note "Ouvert par défaut"
    Contenu visible par défaut, mais repliable.
```

### Onglets

```markdown
=== "Fedora / Kinoite"

    ```bash
    sudo dnf install podman
    ```

=== "Arch / CachyOS"

    ```bash
    sudo pacman -S podman
    ```

=== "Debian / Ubuntu"

    ```bash
    sudo apt install podman
    ```
```

!!! tip "Synchronisation des onglets"
    Avec `content.tabs.link` dans les features, tous les groupes d'onglets avec le **même label** se synchronisent sur toute la page et entre les pages.

### Blocs de code

````markdown
```bash
echo "Hello"
```

```python title="mon_script.py"
print("avec un titre")
```

```bash linenums="1"
# avec numéros de ligne
echo "ligne 1"
echo "ligne 2"
```

```python hl_lines="2 3"
# lignes 2 et 3 surlignées
a = 1
b = 2
c = 3
```

``` { .bash .no-copy }
# sans bouton copier
```
````

**Code inline avec coloration** (nécessite `pymdownx.inlinehilite`) :

```markdown
Utilise la commande `#!bash sudo systemctl restart navidrome`
```

### Icônes et emojis

!!! warning "Config obligatoire"
    Les icônes `:material-...:` et `:octicons-...:` nécessitent **impérativement** dans le toml :
    ```toml
    [project.markdown_extensions.pymdownx.emoji]
    emoji_index = "zensical.extensions.emoji.twemoji"
    emoji_generator = "zensical.extensions.emoji.to_svg"
    ```
    Ne pas utiliser `"pymdownx.emoji" = {}` — ça ne charge pas les icônes Material/Octicons.

```markdown
:material-home:              <!-- Material Design -->
:material-hammer-wrench:
:octicons-rocket-24:         <!-- Octicons (GitHub) -->
:fontawesome-brands-github:  <!-- FontAwesome -->
:lucide-sun:                 <!-- Lucide -->
:simple-podman:              <!-- Simple Icons -->

<!-- Avec couleur (nécessite attr_list) -->
:material-heart:{ .heart style="color: red" }
```

Chercher les icônes disponibles : [zensical.org/docs/authoring/icons-emojis](https://zensical.org/docs/authoring/icons-emojis/)

### Tableaux

```markdown
| Colonne 1 | Colonne 2 | Colonne 3 |
|---|---|---|
| Valeur | Valeur | Valeur |
| **Gras** | `code` | :material-check: |

<!-- Alignement -->
| Gauche | Centre | Droite |
|:---|:---:|---:|
| a | b | c |
```

### Listes de tâches

```markdown
- [x] Tâche terminée
- [ ] Tâche à faire
- [ ] Autre tâche
```

### Touches clavier (nécessite `pymdownx.keys`)

```markdown
++ctrl+shift+v++
++enter++
++cmd+c++
```

### Formatage avancé

```markdown
~~texte barré~~          (pymdownx.tilde)
^^texte souligné^^       (pymdownx.caret)
==texte surligné==       (pymdownx.mark)
H~2~O                    (indice)
x^2^                     (exposant)
```

### Annotations dans le code

````markdown
```bash
systemctl --user enable --now navidrome.service # (1)!
```

1. Active le démarrage automatique au boot **et** démarre immédiatement.
````

### Abréviations (nécessite `abbr`)

```markdown
Le *[HTML]: HyperText Markup Language
Le HTML est utile.
<!-- "HTML" aura un tooltip avec la définition -->
```

### Notes de bas de page

```markdown
Voici une note[^1].

[^1]: Le contenu de la note de bas de page.
```

---

## Pièges et problèmes rencontrés

### Icônes qui ne s'affichent pas

**Symptôme** : `:material-hammer-wrench:` ou `:octicons-rocket-24:` s'affichent en texte brut.

**Cause** : `"pymdownx.emoji" = {}` dans le toml ne configure pas les bons index.

**Solution** : Remplacer par :
```toml
[project.markdown_extensions.pymdownx.emoji]
emoji_index = "zensical.extensions.emoji.twemoji"
emoji_generator = "zensical.extensions.emoji.to_svg"
```

---

### Bouton "copier" absent sur les blocs de code

**Cause** : Feature non activée.

**Solution** : Ajouter dans `[project.theme]` :
```toml
features = [
    "content.code.copy",
]
```

---

### Boutons Précédent / Suivant absents

**Cause** : Feature non activée.

**Solution** :
```toml
features = [
    "navigation.footer",
]
```

---

### Onglets qui ne se synchronisent pas entre sections

**Cause** : `content.tabs.link` non activé.

**Solution** :
```toml
features = [
    "content.tabs.link",
]
```

---

### Indentation des admonitions

Les admonitions nécessitent **4 espaces** d'indentation pour le contenu, pas 2 :

```markdown
!!! note
    ✅ 4 espaces — fonctionne

!!! note
  ❌ 2 espaces — ne fonctionne pas
```

---

### Blocs de code dans les onglets

Quand un bloc de code est à l'intérieur d'un onglet (`===`), il faut **4 espaces d'indentation** pour le contenu de l'onglet :

````markdown
=== "Bash"

    ```bash
    echo "bien indenté"
    ```

=== "Python"

    ```python
    print("bien indenté")
    ```
````

---

### `site_url` obligatoire pour certaines features

`navigation.instant` et les instant previews nécessitent que `site_url` soit défini dans `[project]`, sinon le `sitemap.xml` est vide et ces features ne fonctionnent pas.

---

## Déploiement Netlify

Le fichier `netlify.toml` à la racine du projet :

```toml
[build]
  command = "pip install zensical && zensical build"
  publish = "site"

[build.environment]
  PYTHON_VERSION = "3.11"
```

Zensical génère le site dans le dossier `site/` par défaut.
