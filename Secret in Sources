# Compte rendu : Secrets in Source
Plateforme : HackerDNA ( https://hackerdna.com/labs/secrets-in-source )
Catégorie : Web, Information Disclosure, Broken Access Control
Difficulté : Très Facile
Tags : Web, Source Code

## Scénario
Apprendre à voir le code source et à découvrir les secrets que les développeurs cachent à la vue de tous : un chemin de fichier divulgué dans un commentaire HTML qui livre le flag.

Une introduction pour débutants à l'information disclosure et au broken access control.

Prérequis : Aucun.

## Chaîne d'Attaque
[Ouverture du lab] → [Affichage du code source] → [Recherche dans le code] → [Découverte du commentaire] → 🚩 → [Récupération du flag]

## Étape 1 — Principe
Dans ce challenge, l'objectif est simple : comprendre que le navigateur reçoit tout le contenu HTML envoyé par le serveur, même les éléments qui ne sont pas affichés à l'utilisateur.

Tout ce qui est envoyé au client (HTML, JS, CSS) doit être considéré comme accessible et lisible par l'utilisateur.

## Étape 2 — Accès au code source
Pour résoudre ce challenge, il faut afficher le code source de la page.

Deux méthodes possibles :

### Méthode 1 : Raccourci clavier
```
CTRL + U
```

### Méthode 2 : Via le navigateur
Ajouter le préfixe :

```
view-source:
```

Devant l'URL de la page, par exemple :

```
view-source:https://hackerdna.com/labs/secrets-in-source
```

## Étape 3 — Analyse du code source
Une fois le code source affiché, utiliser la recherche :

```
CTRL + F
```

Puis chercher :

```
flag
```

## Étape 4 — Découverte du flag
Dans le code HTML, un commentaire contenant une information sensible est présent :

```html
<!-- Dev note: backup completed -->
<!-- Flag location: "chemin_du_flag/flag.txt" -->
```

Ce commentaire révèle directement l'emplacement du flag.

## Étape 5 — Récupération du flag
Il suffit alors de :

- Accéder au chemin indiqué
- Ou copier directement le flag s'il est affiché dans la page source
- Puis le soumettre dans le challenge

🚩 Flag : FLAG{...}

## Conclusion
Ce challenge met en évidence une erreur classique en sécurité web :

- Laisser des informations sensibles dans le code source HTML
- Penser que les commentaires sont invisibles pour l'utilisateur

En cybersécurité :

> Tout ce qui est envoyé au client doit être considéré comme public.

## Leçons Apprises
- Les commentaires HTML ne sont jamais privés : ils sont visibles dans le code source par n'importe quel visiteur.
- Le réflexe `CTRL + U` / `view-source:` doit être systématique lors de l'audit d'une page web.
- Un chemin de fichier divulgué dans un commentaire suffit souvent à obtenir un accès non autorisé (Broken Access Control).
- Aucun secret, chemin interne, ou note de développement ne doit rester en production.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
