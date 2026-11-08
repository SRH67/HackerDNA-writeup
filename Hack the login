# Compte rendu : Hack The Login
Plateforme : HackerDNA ( https://hackerdna.com/labs/hack-the-login )
Catégorie : Web, Client-Side Auth
Difficulté : Facile
Tags : Web, JavaScript, Hardcoded Credentials

## Scénario
Démarrer le lab, pirater le système et trouver le flag caché pour relever ce défi et gagner des points.

Objectif : trouver un moyen de se connecter au système.

Prérequis : Aucun.

## Chaîne d'Attaque
[Ouverture du lab] → [Page de login] → [Inspection du code source] → [Analyse de script.js] → [Identifiants en dur + chemin du flag] → 🚩

## Étape 1 — Ouverture du lab
Une fois le laboratoire démarré, la page présente un simple écran de connexion :

- Champ `Username`
- Champ `Password`
- Bouton `Login`

Aucun identifiant n'est fourni dans l'énoncé.

## Étape 2 — Inspection du code source
Premier réflexe : vérifier le code source de la page.

```
CTRL + U
```

Ou :

```
view-source:[URL_du_lab]
```

Dans le HTML, un fichier JavaScript est chargé :

```html
<script src="script.js"></script>
```

Code HTML complet de la page :

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Authentication</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <div class="login-box">
            <h2>Login</h2>
            <form id="loginForm">
                <div class="input-group">
                    <label for="username">Username:</label>
                    <input type="text" id="username" name="username" required>
                </div>
                <div class="input-group">
                    <label for="password">Password:</label>
                    <input type="password" id="password" name="password" required>
                </div>
                <button type="submit">Login</button>
            </form>
            <div id="message"></div>
        </div>
    </div>
    <script src="script.js"></script>
</body>
</html>
```

## Étape 3 — Analyse de script.js
Ouverture du fichier `script.js` en l'ajoutant à l'URL du lab :

```
[URL_du_lab]/script.js
```

Contenu du fichier :

```javascript
document.getElementById('loginForm').addEventListener('submit', function(e) {
    e.preventDefault();

    const username = document.getElementById('username').value;
    const password = document.getElementById('password').value;
    const messageDiv = document.getElementById('message');

    // Simple authentication check
    if (username === "admin" && password === "SecretPassword123!") {
        messageDiv.textContent = "Authentication successful! Retrieving flag...";
        messageDiv.className = "success";

        // Read the flag from the random folder
        fetch('7d9f3b2e1c8a5f6d4e0/flag.txt')
            .then(response => response.text())
            .then(flag => {
                messageDiv.textContent = `Success! Flag: ${flag}`;
            })
            .catch(error => {
                messageDiv.textContent = "Error reading flag file";
                messageDiv.className = "";
            });
    } else {
        messageDiv.textContent = "Invalid username or password";
        messageDiv.className = "";
    }
});
```

Plusieurs informations critiques sont visibles en clair :

| Élément | Valeur |
| --- | --- |
| Username | `admin` |
| Password | `SecretPassword123!` |
| Chemin du flag | `7d9f3b2e1c8a5f6d4e0/flag.txt` |

## Étape 4 — Récupération du flag
Deux méthodes possibles :

### Méthode 1 : Connexion normale
Saisir les identifiants trouvés dans le formulaire de login :

- Username : `admin`
- Password : `SecretPassword123!`

Le JavaScript affiche alors le flag directement dans la page.

### Méthode 2 : Accès direct au flag (plus rapide)
Ajouter directement le chemin découvert à l'URL du lab, sans passer par le login :

```
[URL_du_lab]/7d9f3b2e1c8a5f6d4e0/flag.txt
```

Le flag s'affiche en clair.

🚩 Flag : FLAG{...}

## Conclusion
Ce challenge met en évidence deux erreurs classiques :

- Authentification réalisée uniquement côté client (JavaScript)
- Identifiants en dur dans le code source

Le serveur ne vérifie rien : c'est le navigateur qui décide si l'utilisateur est authentifié. Il suffit de lire le code pour devenir admin.

En cybersécurité :

> Ne jamais faire confiance au client. Toute vérification d'authentification doit être faite côté serveur.

## Leçons Apprises
- Toujours inspecter les fichiers JS chargés par une page (`script.js`, `main.js`, `app.js`).
- Les identifiants en dur dans le JavaScript sont visibles par n'importe quel visiteur.
- Un dossier "aléatoire" comme `7d9f3b2e1c8a5f6d4e0/` n'est pas une protection si son chemin est écrit en clair dans le code.
- La sécurité par obscurité ne fonctionne pas.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
