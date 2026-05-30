# Zensical — Cheatsheet

## zensical.toml — Features essentielles

```toml
[project.theme]
features = [
    "navigation.footer",    # Précédent / Suivant
    "content.code.copy",    # Bouton copier
    "content.tabs.link",    # Sync onglets même label
    "navigation.instant",   # SPA navigation
    "navigation.top",       # Retour en haut
]
```

## zensical.toml — Extensions essentielles

```toml
# Icônes Material/Octicons/FontAwesome — NE PAS utiliser "pymdownx.emoji" = {}
[project.markdown_extensions.pymdownx.emoji]
emoji_index = "zensical.extensions.emoji.twemoji"
emoji_generator = "zensical.extensions.emoji.to_svg"

# Coloration syntaxique
[project.markdown_extensions.pymdownx.highlight]
anchor_linenums = true
line_spans = "__span"
pygments_lang_class = true

# Onglets
[project.markdown_extensions.pymdownx.tabbed]
alternate_style = true

# Admonitions dépliables
[project.markdown_extensions.pymdownx.details]

# Autres
[project.markdown_extensions.admonition]
[project.markdown_extensions.attr_list]
[project.markdown_extensions.pymdownx.inlinehilite]
[project.markdown_extensions.pymdownx.superfences]
[project.markdown_extensions.pymdownx.tasklist]
custom_checkbox = true
clickable_checkbox = true
```

---

## Admonitions

```markdown
!!! note "Titre"        !!! tip              !!! warning
!!! danger              !!! info             !!! success
!!! question            !!! example          !!! quote

??? note "Dépliable — fermé par défaut"
???+ note "Dépliable — ouvert par défaut"
```

## Onglets

```markdown
=== "Label A"           (4 espaces pour le contenu)

    contenu...

=== "Label B"

    contenu...
```

## Code blocks

````markdown
```bash title="titre"
```bash linenums="1"
```python hl_lines="2 3"
``` { .bash .no-copy }
````

## Icônes

```markdown
:material-home:          :material-hammer-wrench:
:octicons-rocket-24:     :fontawesome-brands-github:
:lucide-sun:             :simple-podman:
```

## Formatage

```markdown
~~barré~~    ^^souligné^^    ==surligné==    H~2~O    x^2^
++ctrl+shift+v++             (touches clavier)
- [x] fait   - [ ] à faire  (tâches)
```

## Navigation (toml)

```toml
nav = [
  { Accueil = "index.md" },
  { Section = [
    "section/index.md",
    { "Page" = "section/page.md" },
  ]},
]
```

## Front matter (par page)

```yaml
---
hide: [navigation, toc, footer, path]
---
```

## Commandes

```bash
zensical serve    # http://localhost:8000
zensical build    # → dossier site/
```

---

**Pièges** : icônes → utiliser `zensical.extensions.emoji.*` pas `pymdownx.emoji = {}` · admonitions/onglets → **4 espaces** d'indentation · `site_url` obligatoire pour `navigation.instant`
