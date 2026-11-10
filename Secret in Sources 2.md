# Compte rendu : Secrets in Source 2
Plateforme : HackerDNA ( https://hackerdna.com/labs/secrets-in-source-2 )
Auteur : SRH
Catégorie : Web, Information Disclosure, Bypass Client-Side
Difficulté : Facile
Tags : Web, Source Code, Contournement

## Scénario
SecureVault Technologies affirme que son site web est entièrement sécurisé grâce à une protection avancée du code source. Le clic droit est désactivé, les outils de développement sont bloqués et des systèmes de détection sont mis en place pour empêcher toute intrusion.

Objectif : vérifier si la sécurité côté client est réellement efficace, et percer les secrets que le site s'efforce de dissimuler.

Prérequis : Aucun.

## Chaîne d'Attaque
[Ouverture du lab] → [Blocage clic droit / CTRL+U] → [Contournement côté client] → [Récupération du code source] → [Découverte du chemin protégé] → 🚩

## Étape 1 — Constat : Les protections côté client
Une fois le défi lancé, plusieurs blocages sont visibles :

- Le clic droit est désactivé
- La combinaison de touches `CTRL + U` est désactivée
- L'ouverture des outils de développement est bloquée / détectée

Ces protections donnent l'illusion d'un site sécurisé.

**Réflexe audit :** Toute protection appliquée uniquement côté client (JavaScript, HTML) peut être contournée. Le navigateur appartient à l'utilisateur, pas au serveur.

## Étape 2 — Contournement : Récupérer le code source autrement

### Méthode 1 : Préfixe view-source:
La méthode la plus simple consiste à passer directement par la barre d'adresse, sans utiliser le clic droit ni le raccourci :

```
view-source:https://lab.hdna.me/145-secrets-in-source-2/
```

Le code source complet de la page s'affiche alors, malgré les blocages JavaScript.

### Méthode 2 : Téléchargement avec curl / wget
Comme il s'agit d'un fichier HTML, il est possible de le télécharger directement, sans passer par le navigateur :

```bash
curl -s https://lab.hdna.me/145-secrets-in-source-2/ -o source.html
```

```bash
wget https://lab.hdna.me/145-secrets-in-source-2/ -O source.html
```

Puis lecture du fichier :

```bash
cat source.html
```

### Méthode 3 : Enregistrer la page entière
Utiliser la fonction "Enregistrer la page" du navigateur :

```
CTRL + S
```

Le fichier HTML téléchargé contient le code source complet, sans passer par les protections.

## Étape 3 — Analyse du code source
Une fois la page source obtenue, la recherche dans le code permet de trouver le chemin protégé :

```
CTRL + F
```

Puis chercher :

```
flag
```

Dans le code HTML, un commentaire ou un chemin dissimulé révèle l'emplacement du flag.

## Étape 4 — Récupération du flag
Il suffit alors d'ajouter le chemin découvert à l'URL du lab pour obtenir le flag :

```
https://lab.hdna.me/145-secrets-in-source-2/[chemin_decouvert]
```

Puis copier le flag et le soumettre dans le challenge.

🚩 Flag : FLAG{...}

## Conclusion
Ce challenge démontre une erreur classique en sécurité web :

- Désactiver le clic droit ne protège rien
- Bloquer `CTRL + U` ne protège rien
- Bloquer les DevTools ne protège rien

La sécurité côté client n'est pas une sécurité. Tout ce qui est envoyé au navigateur est déjà entre les mains de l'utilisateur.

En cybersécurité :

> Si le navigateur peut l'afficher, l'utilisateur peut le lire, le copier et le télécharger.

## Leçons Apprises
- Le JavaScript de protection ne doit jamais être considéré comme une mesure de sécurité.
- Le préfixe `view-source:` contourne la plupart des blocages d'interface.
- `curl` / `wget` permettent de récupérer une page sans subir aucune protection JavaScript.
- Un secret présent dans le code source reste un secret exposé, même avec clic droit désactivé.

Les noms des défis et toute propriété intellectuelle de tiers restent la propriété de leurs détenteurs respectifs.
