# TLauncher sur Aurora Linux — Résoudre les blocages SELinux

Aurora Linux utilise **SELinux** en mode enforcing, ce qui peut bloquer le lancement de TLauncher (et des applications Java en général). Ce guide présente plusieurs approches, de la plus ciblée à la plus pragmatique.

---

```bash
# Créer la toolbox Ubuntu 26.04
toolbox create --distro ubuntu --release 26.04 ubuntu-java

# Entrer dedans
toolbox enter ubuntu-java

# Installer Java
sudo apt update
sudo apt install default-jdk -y

# Vérifier
java -version
```

```bash
toolbox run --container ubuntu-java java -jar /var/home/famille/TLauncher.jar
```



## Pourquoi SELinux bloque Java ?

Java utilise un compilateur **JIT (Just-In-Time)** qui écrit du code machine directement en mémoire, puis l'exécute. SELinux interprète cette mémoire exécutable comme une menace potentielle et bloque l'opération. C'est pour cette même raison qu'une Distrobox Debian fonctionne : le contexte SELinux y est différent.

---

## Option 1 — Politique SELinux ciblée avec `audit2allow` (recommandé)

Plutôt que d'affaiblir SELinux globalement, on crée un module de politique qui autorise uniquement ce que Java a besoin de faire.

```bash
# Voir les refus SELinux récents liés à Java
ausearch -c 'java' --raw | audit2allow -M my-java-policy

# Installer le module de politique
semodule -i my-java-policy.pp
```

Si `audit2allow` n'est pas disponible sur votre système :

```bash
sudo rpm-ostree install policycoreutils-devel
# Redémarrer après l'installation
```

> **Conseil :** lancez TLauncher une première fois (même s'il échoue), puis exécutez `ausearch` pour capturer les refus. Plus les logs sont riches, plus la politique générée sera précise.

---

## Option 2 — Désactiver le JIT Java

SELinux bloque principalement Java à cause du JIT. On peut forcer Java en mode interprété, ce qui évite l'écriture de mémoire exécutable.

```bash
java -Xint -jar TLauncher.jar
```

L'option `-Xint` désactive le JIT et force l'interprétation du bytecode. Les performances seront légèrement réduites, mais l'application fonctionnera.

---

## Option 3 — Booléen SELinux `allow_execmem`

Cette option autorise les processus non confinés à utiliser de la mémoire exécutable, ce qui couvre le comportement JIT de Java.

```bash
sudo setsebool -P allow_execmem 1
```

> ⚠️ **Attention :** ce réglage s'applique à tous les processus non confinés, pas uniquement à Java. Il assouplit SELinux de façon plus large que l'Option 1.

---

## Option 4 — Intégrer la Distrobox comme application native

Puisque la Distrobox Debian fonctionne déjà, on peut l'exposer proprement comme une application native sans modifier SELinux.

### Exporter le binaire Java

```bash
distrobox-export --bin /usr/bin/java --export-path ~/.local/bin
```

### Créer un raccourci `.desktop`

```ini
# ~/.local/share/applications/tlauncher.desktop
[Desktop Entry]
Name=TLauncher
Exec=distrobox-enter --name debian -- java -jar /home/votre_user/TLauncher.jar
Icon=minecraft
Terminal=false
Type=Application
```

Remplacez `debian` par le nom de votre Distrobox et ajustez le chemin vers le `.jar`.

---

## Récapitulatif

| Option | Portée | Impact SELinux | Complexité |
|--------|--------|----------------|------------|
| `audit2allow` | Ciblée sur Java | Minimal | Moyenne |
| `-Xint` (JIT désactivé) | Instance par instance | Aucun | Faible |
| `allow_execmem` | Tous les processus non confinés | Modéré | Faible |
| Distrobox | Isolation complète | Aucun | Faible |

**Recommandation :** commencez par l'**Option 1** pour une solution propre et ciblée. Si les logs SELinux sont vides, passez à l'**Option 3**. L'**Option 4** (Distrobox) reste la solution la plus robuste si vous souhaitez éviter de toucher aux politiques SELinux.

---

## Diagnostic

Pour vérifier ce que SELinux bloque exactement :

```bash
# Afficher les derniers refus en temps réel
sudo journalctl -f | grep avc

# Chercher tous les refus liés à Java
ausearch -c 'java' --raw
```

Ces commandes vous permettront d'identifier précisément les opérations bloquées et d'affiner votre politique SELinux.