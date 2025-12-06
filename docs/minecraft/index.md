# Commandes Minecraft Utiles

Tes enfants adorent Minecraft et te demandent souvent des commandes ? Cette section est faite pour eux (et pour toi !) pour avoir les commandes sous la main, prêtes à être copiées-collées.

## Supprimer un sol de 9x9 blocs (ou autre)

**Scénario :** Ton fils veut enlever (supprimer/creuser) un sol de 9x9 blocs, hauteur 1 bloc dans l'immeuble (probablement pour faire un trou au centre d'un palier quartz, style atrium ou vide central). C'est hyper simple avec `/fill` + `minecraft:air` (remplace les blocs par de l'air = suppression instantanée).

**Commande unique (copie-colle dans le chat T) :**

Positionne-toi au centre du sol à enlever (ex. : sur un palier à Y=~2, ~5, etc.) et tape :

```
/fill ~-4 ~-1 ~-4 ~4 ~-1 ~4 minecraft:air
```

**Explication :**

*   `~-4 ~-1 ~-4` : Premier coin (4 blocs en arrière/gauche, 1 bloc en dessous de toi).
*   `~4 ~-1 ~4` : Coin opposé (4 blocs devant/droite, même hauteur).

**Résultat :** 9x1x9 blocs supprimés (carré parfait 9x9, 1 bloc haut), juste sous tes pieds. Le sol disparaît, tu tombes en dessous !
