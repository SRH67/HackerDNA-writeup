# Compte rendu : Reverse Engineering - License Key Crackme
Plateforme : HackerDNA
Auteur : SRH
Catégorie : Reverse Engineering (https://hackerdna.com/fr/labs/reverse-engineering-crackme)
Difficulté : Moyenne
Tags : Reverse Engineering, Binary Analysis, Crackme, Keygen, objdump, GDB, ELF, ARM64 Assembly, Static Analysis, Ghidra
XP : +400 XP

## Scénario
Analyser un binaire ELF ARM64 qui vérifie une clé de licence, comprendre son algorithme de validation, puis écrire un keygen qui génère une clé valide.

Objectif : pas de patch bourrin, mais un vrai keygen qui reproduit l'algorithme du binaire.

Prérequis : Ghidra, objdump, GDB, Python, Assembly ARM64.

## Chaîne d'Attaque
[Recon du binaire] → [Strings] → [Analyse statique Ghidra] → [Analyse dynamique GDB] → [Compréhension de l'algorithme] → [Keygen Python] → [Clé valide] → [Flag] → 🚩

## Étape 1 — Reconnaissance du binaire
```bash
ls -la
file crackme
```

Résultat attendu :

```
crackme: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV)
```

Vérifications :

```bash
checksec --file=crackme
strings crackme | head -n 100
```

Strings intéressantes à repérer :
- `Enter license key:`
- `Invalid license`
- `License accepted`
- `FLAG{...}` ou format de clé, ex : `XXXX-XXXX-XXXX-XXXX`

Test de comportement :

```bash
./crackme
Enter license key: TEST-1234
# Invalid license
```

## Étape 2 — Analyse statique avec objdump
Désassemblage ARM64 :

```bash
objdump -d crackme > disasm.txt
objdump -t crackme | head -n 100
objdump -s -j .rodata crackme
```

Recherche de la fonction de validation :

```bash
grep -n "main\|validate\|check\|license" disasm.txt
```

## Étape 3 — Analyse statique avec Ghidra
1. Nouveau projet Ghidra → Importer `crackme`
2. Auto-analyse
3. Aller dans `main`
4. Renommer les fonctions : `main`, `validate_key`, `check_license`
5. Regarder le décompilé

Exemple de logique retrouvée dans le décompilé :

```c
int validate_key(char *input) {
  // 1. Vérifie la longueur, ex : 19 chars avec tirets
  // 2. Vérifie le format XXXX-XXXX-XXXX-XXXX
  // 3. Pour chaque char, applique une transformation :
  //    ex : (char * 0x1F + index) ^ 0x42
  // 4. Compare le résultat à une valeur attendue / checksum
  // 5. Si checksum == attendu → License accepted
}
```

Points clés à noter :
- Longueur exacte attendue
- Alphabet autorisé
- Constantes magiques : multiplicateurs, XOR, additions
- Checksum / somme de contrôle finale

## Étape 4 — Analyse dynamique avec GDB
Pour confirmer l'algorithme et voir les valeurs en live :

```bash
gdb ./crackme
```

Dans GDB :

```
break *validate_key
run
info registers
x/s $x0
stepi
```

Ou avec un breakpoint sur la comparaison finale pour voir la valeur attendue vs la valeur calculée.

Objectif : valider ce que Ghidra a montré, pas deviner.

## Étape 5 — Écriture du Keygen
Une fois l'algorithme inversé, on le réimplémente en Python pour générer une clé valide.

Exemple de structure (à adapter à l'algorithme réel du binaire) :

```python
#!/usr/bin/env python3
# Keygen - License Key Crackme - SRH

def generate_key():
    # Reproduire exactement l'algorithme inversé dans Ghidra
    # Exemple : checksum, XOR, transformation par index
    key = ""
    checksum = 0
    # ... logique inversée ...
    return key

if __name__ == "__main__":
    print(generate_key())
```

Exécution :

```bash
python3 keygen.py
# Exemple : ABCD-EFGH-IJKL-MNOP
```

## Étape 6 — Validation de la clé
```bash
./crackme
Enter license key: [CLE_GENEREE]
```

Résultat :

```
License accepted!
FLAG{...}
```

🚩 Flag : FLAG{...}

## Conclusion
Ce lab n'est pas un simple patch de `JE` → `JMP`.
Il faut :
1. Comprendre le binaire ELF ARM64
2. Lire l'assembly / le décompilé Ghidra
3. Extraire l'algorithme de validation complet
4. Le réimplémenter en keygen

C'est ça qui fait la différence entre un crackme et un keygen.

## Leçons Apprises
1. Toujours commencer par `file`, `strings`, `checksec` avant Ghidra.
2. Ghidra pour la logique globale, GDB pour confirmer les valeurs.
3. Noter toutes les constantes : XOR, multiplicateurs, checksums.
4. Un keygen = algorithme inversé + réimplémenté, pas une clé hardcodée.
5. En ARM64, bien suivre les registres `x0-x7` pour les arguments et `w0` pour le retour.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
