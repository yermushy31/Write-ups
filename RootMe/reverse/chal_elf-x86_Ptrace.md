# Write-up CTF : Root-Me — Ptrace x86

## 1. Petit contexte

L'objectif est de decouvrir le mot de passe atendu par le binaire il y a deux mécanismes suivants:

* **Une protection anti-débogage** via l'appel système `ptrace`.
* **De l'obfuscation de code** par désalignement d'instructions (overlapping instructions).

---

## 2. Analyse Statique & Mécanismes de Defense

### A. Le contournement de l'anti-débogage (`ptrace`)

Dès le début de la fonction `main`, le programme effectue un appel à `ptrace` :

```assembly
0x08048410 <+32>:    call   0x8058a70 <ptrace>
0x08048415 <+37>:    add    $0x10,%esp
0x08048418 <+40>:    test   %eax,%eax
0x0804841a <+42>:    jns    0x8048436 <main+70>

```

**Principe** : Sous Linux, un processus ne peut être inspecté (*tracé*) que par un seul débogueur à la fois. L'appel `ptrace(PTRACE_TRACEME, ...)` permet au programme de demander à être tracé par son parent. Si un débogueur (comme GDB) est déjà attaché, cet appel échoue et retourne `-1` (stocké dans `%eax`).
* **Logique du binaire** : L'instruction `jns` (*Jump if Not Sign*) saute à la suite du programme si le résultat de `test %eax,%eax` montre que `%eax` est positif ou nul ($\geq 0$). Si `%eax` vaut `-1`, le saut n'est pas pris, le programme affiche un message d'erreur et quitte.

### B. L'obfuscation par désalignement d'instructions

Juste après la lecture de notre entrée via `fgets`, on observe ce bloc étrange :

```assembly
0x0804848e <+158>:   lea    0x8048497,%eax
0x08048494 <+164>:   inc    %eax
0x08048495 <+165>:   jmp    *%eax
0x08048497 <+167>:   mov    $0x8bea558a,%eax
0x0804849c <+172>:   inc    %ebp
0x0804849d <+173>:   hlt

```

**L'astuce** : Le programme charge l'adresse `0x8048497` dans `%eax`, l'incrémente de 1 (`0x8048498`), puis saute directement à cette adresse intermédiaire via `jmp *%eax`.

Le désassembleur de GDB affiche une instruction `hlt` (qui stopperait le programme), mais celle-ci n'est jamais exécutée ! En sautant au milieu de l'instruction `mov`, le processeur décode les octets suivants d'une toute autre manière, exécutant du code masqué qui initialise les premières vérifications.

### C. La vérification du mot de passe

Après cette feinte, le programme compare un à un les caractères saisis (stockés à partir de `-0x16(%ebp)`) avec des caractères hardcodés calculés dynamiquement à partir d'un pointeur de référence en `-0xc(%ebp)`.

```assembly
0x080484a1 <+177>:   mov    (%eax),%al
0x080484a3 <+179>:   cmp    %al,%dl
0x080484a5 <+181>:   jne    0x80484e4 <main+244>

```

Si une seule comparaison échoue (`jne`), le programme saute en `0x80484e4` pour afficher un message d'erreur (vraisemblablement "Wrong password").

---

## 3. Analyse Dynamique & Résolution avec GDB

Grâce à la session GDB fournie, nous pouvons suivre pas à pas la résolution du challenge.

### Étape 1 : Forcer le passage du Ptrace

On place un breakpoint juste après l'appel à `ptrace` (`0x08048415`). Comme attendu, `%eax` contient `-1` car GDB est attaché. On modifie manuellement le registre pour simuler un succès :

```text
Breakpoint 1, 0x08048415 in main ()
(gdb) set $eax = 0
(gdb) c

```

Le programme est dupé et continue son exécution.

### Étape 2 : Extraction du mot de passe (Brute-force des registres)

Le programme demande le mot de passe. On entre une fausse valeur : `AAAAAAAAAAAA`.

Le binaire va maintenant comparer notre saisie (dans `%dl`) avec le bon caractère (dans `%al`). Pour obtenir le flag sans se fatiguer à recoder l'algorithme, il suffit de regarder ce que contient `%al` à chaque point de comparaison, puis de modifier `%dl` pour qu'il valide le test et passe au caractère suivant.

1. **Premier caractère** (Adresse `0x080484a3`) :
* `$al` contient `valeur_ascii`, soit le caractère **`redacted`**.
* On valide : `set $dl = $al`.


2. **Deuxième caractère** (Adresse `0x080484a7`) :
* `$al` contient `valeur_ascii`, soit le caractère **`redacted`**.
* On valide : `set $dl = $al`.


3. **Troisième caractère** (Adresse `0x080484b2`) :
* `$al` contient `valeur_ascii`, soit le caractère **`redacted`**.
* On valide : `set $dl = $al`.


4. **Quatrième caractère** (Adresse `0x080484bf`) :
* `$al` contient `valeur_ascii`, soit le caractère **`redacted`**.
* On valide : `set $dl = $al`.


5. **Cinquième caractère** (Adresse `0x080484ce`) :
* `$al` contient `valeur_ascii`, soit le caractère **`redacted`**.
* On valide : `set $dl = $al`.



En envoyant l'instruction `continue`, le programme affiche le message de succès :

> **Good password !!!**

---

## 4. Conclusion & Flag

En mettant bout à bout les caractères extraits des examens de registres, on obtient le mot de passe final: ******


* **Sources**:
- https://docs.libdebug.org/latest/quality_of_life/anti_debugging/
- https://man7.org/linux/man-pages/man2/ptrace.2.html
- https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf

