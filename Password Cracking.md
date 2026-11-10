# Compte rendu : Password Cracking - Custom Wordlist Attack
Plateforme : HackerDNA ( https://hackerdna.com/fr/labs/password-cracking-custom-wordlist )
Auteur : SRH
Catégorie : Cryptography
Difficulté : Facile
Tags : Password Cracking, Custom Wordlist, Wordlist Generation, CeWL
XP : +200 XP

## Scénario
Build a wordlist from a target's own pages and crack their file.

Objectif : générer une wordlist personnalisée à partir des propres pages de la cible, puis l'utiliser pour cracker un fichier protégé.

Prérequis : CeWL, John the Ripper / Hashcat, zip2john / ssh2john selon le fichier.

## Chaîne d'Attaque
[Reconnaissance du site] → [Scraping avec CeWL] → [Nettoyage / Mutations] → [Extraction du hash] → [Crack avec la wordlist custom] → [Mot de passe trouvé] → 🚩

## Étape 1 — Reconnaissance
Navigation vers la cible :

```
http://[IP_CIBLE]
```

Le site contient des pages avec du vocabulaire spécifique à l'entreprise / à la cible : noms, produits, villes, slogans, etc.

C'est ce vocabulaire qui servira de base à la wordlist. Un mot de passe basé sur un mot lié à la cible a beaucoup plus de chances d'être dans une wordlist custom que dans `rockyou.txt`.

## Étape 2 — Génération de la wordlist avec CeWL
CeWL (Custom Word List generator) crawl un site web et extrait les mots.

```bash
cewl http://[IP_CIBLE] -w custom.txt -d 2 -m 5 --lowercase
```

Options :
- `-w custom.txt` : fichier de sortie
- `-d 2` : profondeur de crawl
- `-m 5` : longueur minimale des mots
- `--lowercase` : tout mettre en minuscule

Vérification :

```bash
wc -l custom.txt
head -n 20 custom.txt
```

## Étape 3 — Nettoyage et mutations
Suppression des doublons et tri :

```bash
sort -u custom.txt -o custom_clean.txt
wc -l custom_clean.txt
```

Génération de variantes (majuscules, leet, chiffres à la fin) avec les règles de John / Hashcat, ou avec un outil de mutation :

```bash
# Exemple : règles John
john --wordlist=custom_clean.txt --rules --stdout > custom_mutated.txt
```

Ou fusion avec une wordlist classique en priorité custom d'abord :

```bash
cat custom_clean.txt rockyou.txt > final.txt
sort -u final.txt -o final_clean.txt
```

## Étape 4 — Identification du fichier à cracker
Le lab fournit un fichier protégé, par exemple :
- Un ZIP : `protected.zip`
- Un PDF, une clé SSH, un fichier avec hash

Exemple pour un ZIP, extraction du hash :

```bash
zip2john protected.zip > hash.txt
cat hash.txt
```

Pour une clé SSH :

```bash
ssh2john id_rsa > hash.txt
```

## Étape 5 — Crack avec la wordlist custom
Avec John the Ripper :

```bash
john --wordlist=custom_clean.txt hash.txt
john --show hash.txt
```

Avec Hashcat (exemple ZIP / mode selon le hash) :

```bash
hashcat -m [MODE] -a 0 hash.txt custom_clean.txt --show
```

## Étape 6 — Récupération du flag
Une fois le mot de passe trouvé :

```bash
unzip protected.zip
# ou
cat flag.txt
```

Mot de passe trouvé : `[MOT_DE_PASSE]`

🚩 Flag : FLAG{...}

## Conclusion
Une wordlist générique ne suffit pas toujours. Une wordlist construite à partir du vocabulaire de la cible est beaucoup plus efficace, car les utilisateurs choisissent souvent des mots de passe liés à leur entreprise, leur ville, leur produit ou leur équipe.

## Leçons Apprises
1. Toujours scraper le site de la cible avant de cracker.
2. CeWL = outil de base pour générer une wordlist ciblée.
3. Nettoyer, dédupliquer et muter la wordlist avant le crack.
4. Mettre la wordlist custom en premier, avant rockyou.
5. Un mot de passe basé sur une info publique est un mot de passe faible.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
