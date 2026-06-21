# SEIGMA API — Documentation REST

Documentation complète de l'API REST SEIGMA. 779 endpoints, 154 modèles.
Générée automatiquement par introspection des endpoints réels.

> Généré le 2026-06-20 — Spécification OpenAPI 3.1

## Contenu

| Fichier | Usage |
|---------|-------|
| `seigma-swagger.html` | **Swagger UI interactif** — tester l'API en direct (autonome, YAML intégré) |
| `seigma-api-reference.html` | Redoc — documentation navigable (recherche, navigation par modèle) |
| `seigma-openapi.yaml` | Spécification OpenAPI 3.1 brute — pour import dans Postman, Insomnia, etc. |
| `seigma-api-reference.pdf` | Export PDF (411 pages) — consultation hors ligne, archivage |

## Utilisation rapide

### Swagger UI (test interactif)

1. Télécharger `seigma-swagger.html` (un seul fichier, le YAML est intégré)
2. Ouvrir dans un navigateur (Chrome, Firefox, Edge)
3. Remplir la barre de connexion :
   - **Instance** : votre sous-domaine SEIGMA (ex: `maseigma` → `maseigma.seigma.app`)
   - **Email** : votre email de connexion
   - **Mot de passe** : votre mot de passe
4. Cliquer **Connexion** — le token JWT est automatiquement injecté dans toutes les requêtes
5. Déplier un endpoint → **Try it out** → **Execute**

> ⚠️ Le fichier est autonome (le YAML est intégré dans le HTML). Ne pas le re-séparer si vous l'éditez.

### Redoc (référence navigable)

1. Ouvrir `seigma-api-reference.html` dans un navigateur
2. Navigation par modèle dans le menu de gauche
3. Recherche full-text en haut

## Structure de l'API

| Opération | Endpoint | Description |
|-----------|----------|-------------|
| Authentification | `POST /api/auth/authenticate` | JWT Bearer token |
| Lecture par ID | `GET /api/reference/{Model}/{id}` | Tous les champs |
| Recherche | `POST /api/reference/{Model}/search` | Pagination, filtres, SelectAttributes |
| Création | `POST /api/reference/{Model}` | ModelAttributeList |
| Modification | `PUT /api/reference/{Model}/{id}` | Mise à jour partielle |
| Suppression | `DELETE /api/reference/{Model}/{id}` | Soft-delete |
| Activités | `/api/activity/*` | CRUD complet |
| Poinçons | `/api/timelogs/*` | CRUD + start/stop |

> 💡 Les schémas détaillés (ModelAttributeList, ModelSearch) sont consultables directement dans Swagger UI.

## Prérequis Swagger UI

- Navigateur moderne (Chrome, Firefox, Edge)
- Connexion internet au premier chargement (CDN : js-yaml, swagger-ui)
- L'instance SEIGMA doit accepter les requêtes CORS (activé par défaut)

---

Documentation à usage interne. Pour toute question, consulter le schéma OpenAPI ou le Swagger UI.
