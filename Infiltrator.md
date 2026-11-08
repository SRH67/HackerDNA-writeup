# Compte rendu : Infiltrator
Plateforme : HackerDNA ( https://hackerdna.com/labs/infiltrator )
Catégorie : Web & API Security
Difficulté : Boss
Tags : JWT Manipulation, SSH, Log Injection, Privilège Escalation
XP : +200 XP

## Scénario
Forger un token de connexion, s'infiltrer dans le système, et monter jusqu'à root.

Objectif : obtenir un accès initial via une faille JWT, puis escalader les privilèges jusqu'à root.

Prérequis : Nmap, JWT, SSH, Linux.

## Chaîne d'Attaque
[Scan Nmap] → [Découverte Web / API] → [Analyse JWT] → [Forge de token] → [Accès initial] → [SSH] → [Log Injection] → [Escalade vers root] → 🚩

## Étape 1 — Reconnaissance
Scan des ports et services :

```bash
nmap -sV -sC [IP_CIBLE]
```

Identification du service web / API et du mécanisme d'authentification par JWT.

## Étape 2 — Analyse du JWT
Récupération d'un token JWT valide (connexion normale / cookie / header Authorization).

Décodage du token :

```bash
echo [TOKEN] | cut -d. -f2 | base64 -d
```

Structure d'un JWT :

```
header.payload.signature
```

Le header indique l'algorithme (`alg`), le payload contient les claims (`user`, `role`, `exp`...).

## Étape 3 — Forge du token
Vulnérabilité exploitée : mauvaise vérification de la signature JWT.

Techniques possibles selon la configuration :
- `alg: none` : suppression de la signature
- Secret faible : brute-force du secret HMAC
- Confusion d'algorithme

Création d'un token forgé avec un rôle privilégié :

```json
{
  "user": "admin",
  "role": "admin"
}
```

Utilisation du token forgé pour s'authentifier sur l'application et obtenir un accès privilégié.

## Étape 4 — Accès initial / SSH
À partir de l'accès web, récupération d'identifiants ou d'un accès SSH.

Connexion SSH :

```bash
ssh [USER]@[IP_CIBLE]
```

Vérification de l'utilisateur courant :

```bash
whoami
id
```

## Étape 5 — Log Injection / Escalade
Analyse des logs et des services tournant en root.

Exploitation d'une injection dans les logs pour obtenir une exécution de code, puis escalade vers root.

Vérification :

```bash
sudo -l
cat /etc/crontab
ps aux
```

## Étape 6 — Récupération des flags
Flag utilisateur :

```bash
cat ~/user.txt
```

Flag root :

```bash
cat /root/root.txt
```

🚩 Flag User : FLAG{...}
🚩 Flag Root : FLAG{...}

## Conclusion
Ce lab illustre une chaîne complète :
- Un JWT mal vérifié permet de forger une identité admin
- L'accès web mène à un accès SSH
- Une mauvaise gestion des logs mène à root

## Leçons Apprises
- Ne jamais faire confiance au header `alg` d'un JWT.
- Ne jamais utiliser un secret faible pour signer un JWT.
- Toujours vérifier la signature côté serveur.
- Les logs ne doivent jamais être interprétables comme du code.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
