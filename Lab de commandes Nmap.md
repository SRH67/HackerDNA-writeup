# Compte rendu : Lab de commandes Nmap 
Plateforme : HackerDNA (https://hackerdna.com/fr/labs/learn-102)
Auteur : SRH
Catégorie : Network, Reconnaissance
Difficulté : Facile
Tags : Nmap, Telnet, Weak Credentials, Privilège Escalation

## Scénario
Pratiquer les commandes Nmap : scanner les ports, trouver un service Telnet avec des identifiants faibles, et escalader vers root.

Le lab est un lab réseau, pas un lab web. Le travail se fait depuis le terminal d'attaque.

Prérequis : Nmap, Telnet.

## Chaîne d'Attaque
[Scan Nmap] → [Port 23 / Telnet détecté] → [Test identifiants faibles] → [Accès Telnet] → [Élévation de privilèges vers root] → 🚩

## Étape 1 — Reconnaissance avec Nmap
Lancement d'un scan complet avec détection de version :

```bash
nmap -sV -sC [IP_CIBLE]
```

Résultat attendu :

```
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
```

Le port `23/tcp` est ouvert et expose un service Telnet.

## Étape 2 — Analyse du risque Telnet
Telnet est un protocole legacy qui envoie tout en clair sur le réseau, y compris les noms d'utilisateur et les mots de passe.

Il a été remplacé presque partout par SSH, mais il reste présent sur :
- Systèmes legacy
- Équipements embarqués
- Équipements réseau
- Systèmes industriels

Un service Telnet exposé est une cible de grande valeur, car des identifiants par défaut ou faibles donnent souvent un accès immédiat.

## Étape 3 — Exploitation : Identifiants faibles
Connexion au service Telnet :

```bash
telnet [IP_CIBLE] 23
```

Test des identifiants faibles / par défaut classiques :

| Username | Password |
| --- | --- |
| `root` | `root` |
| `admin` | `admin` |
| `root` | `` (vide) |
| `admin` | `password` |

Dans ce lab, l'un de ces couples permet d'obtenir un accès.

Alternative avec Nmap pour tester automatiquement :

```bash
nmap -p 23 --script telnet-brute [IP_CIBLE]
```

## Étape 4 — Élévation de privilèges
Une fois connecté en tant qu'utilisateur limité, vérification des possibilités d'escalade :

```bash
whoami
id
sudo -l
```

Exploitation de la mauvaise configuration pour passer `root`, puis recherche du flag :

```bash
cat /root/flag.txt
```

Ou :

```bash
find / -name "*flag*" 2>/dev/null
```

🚩 Flag : FLAG{...}

## Conclusion
Ce lab illustre la méthodologie complète du pentest :

- Scan Nmap pour découvrir les services
- Exploitation d'un service legacy mal configuré
- Élévation de privilèges vers root

Telnet ne doit plus être utilisé. Tout mot de passe envoyé en clair peut être intercepté.

## Leçons Apprises
- Toujours lancer `nmap -sV -sC` pour obtenir versions + scripts par défaut.
- Un port 23 ouvert = alerte critique.
- Les identifiants par défaut sont la première chose à tester.
- Remplacer Telnet par SSH partout.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
