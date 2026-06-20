# Index des sections à compléter

> **Utilisez cette page comme checklist** avant d'intégrer l'API SEIGMA à votre projet. Chaque 🚧 correspond à une information spécifique à votre instance que vous devez fournir.

---

## Avant de commencer — 4 prérequis

| # | Élément | Où le trouver | Chapitre |
|---|---------|---------------|----------|
| 1 | `{VOTRE_INSTANCE}` | Sous-domaine de votre SEIGMA (ex: `monentreprise` → `monentreprise.seigma.app`) | [01](01-authentification.md) |
| 2 | `{VOTRE_EMAIL}` | Compte API créé par votre administrateur SEIGMA | [01](01-authentification.md) |
| 3 | `{VOTRE_MOT_DE_PASSE}` | Associé au compte API | [01](01-authentification.md) |
| 4 | `{VOTRE_COMPANY_ID}` | Menu Administration → Entreprises dans SEIGMA, ou contactez votre administrateur | [index](index.md) |

---

## Référentiels à obtenir (IDs de base)

Ces IDs sont nécessaires pour les opérations d'écriture (création de WO, facture, reçu). Obtenez-les via les endpoints `search` :

| Référentiel | Endpoint | Utilisé pour | Chapitre |
|-------------|----------|-------------|----------|
| **CustomerId** | `POST /reference/Customer/search` | Création WO, facture, reçu | [04](04-operations-ecriture.md), [05](05-modeles-reference.md) |
| **PaymentTermId** | `POST /reference/PaymentTerm/search` | Création WO, facture | [04](04-operations-ecriture.md), [05](05-modeles-reference.md) |
| **WarehouseId** | `POST /reference/Warehouse/search` | Création WO uniquement (⚠️ pas requis pour facture) | [04](04-operations-ecriture.md), [05](05-modeles-reference.md) |
| **PaymentMethodId** | `POST /reference/PaymentMethod/search` | Création de reçu (⚠️ obligatoire) | [04](04-operations-ecriture.md) |

---

## Équipes et utilisateurs (planning)

| Élément | Description | Chapitre |
|---------|-------------|----------|
| **UserIds des équipes** | Objet `{UserId, Display}` pour chaque équipe. Utilisé par `getactivitiesfordate`. Obtenez la liste via l'interface SEIGMA ou votre administrateur. | [03](03-activities-timelogs.md), [06](06-guides-pratiques.md) |

---

## Volumes et disponibilité (dépendent de l'instance)

| Élément | Note | Chapitre |
|---------|------|----------|
| **Nombre d'enregistrements** | Les chiffres dans le catalogue des modèles (~1932 WO, ~3839 clients, etc.) sont des ordres de grandeur. Vos chiffres varient. | [05](05-modeles-reference.md) |
| **ModelCodes disponibles** | Tous les modèles listés peuvent ne pas être activés sur votre instance. Testez avec un `search` minimal (`Limit=1`) avant d'intégrer. | [02](02-reference-api.md), [05](05-modeles-reference.md) |
| **Endpoints cassés** | Certains endpoints (User/search, Invoice/search, Activity/search, SalesOrderLine) retournent 500. Ce comportement peut être corrigé dans une version plus récente de SEIGMA. Testez avant de déclarer inutilisable. | [07](07-pitfalls-limitations.md) |

---

## Checklist rapide

- [ ] J'ai `{VOTRE_INSTANCE}`, `{VOTRE_EMAIL}`, `{VOTRE_MOT_DE_PASSE}`, `{VOTRE_COMPANY_ID}`
- [ ] J'ai testé `POST /api/auth/authenticate` → 200
- [ ] J'ai obtenu un `CustomerId` via `Customer/search`
- [ ] J'ai obtenu un `PaymentTermId` via `PaymentTerm/search`
- [ ] J'ai obtenu un `WarehouseId` via `Warehouse/search`
- [ ] J'ai obtenu un `PaymentMethodId` via `PaymentMethod/search` (si j'utilise Receipt)
- [ ] J'ai les `UserId` de mes équipes (si j'utilise `getactivitiesfordate`)
- [ ] J'ai testé `GetReferences` (résolution par lot d'IDs)
- [ ] J'ai testé `POST /api/timelogs/search`
- [ ] J'ai testé `GET .../timelogs/start` et `GET .../timelogs/stop`
- [ ] J'ai vérifié quels ModelCodes sont disponibles sur mon instance
- [ ] J'ai noté les endpoints qui retournent 500 sur mon instance

---

> 💡 **Astuce** : Automatisez la résolution des IDs de référentiels dans votre code d'intégration. Ne les hardcodez jamais — utilisez les endpoints `search` au démarrage pour construire un cache local.

---

◄ [Précédent : 07 — Pièges et limitations](07-pitfalls-limitations.md) │ [Index](index.md) │ [Suivant : Accueil — Index](index.md) ►
