# Documentation API SEIGMA — {VOTRE_ENTREPRISE}

> **Version** : 1.0.0 | **Dernière mise à jour** : 2026-06-20
> **Instance** : `{VOTRE_INSTANCE}.seigma.app` | **CompanyId** : `{VOTRE_COMPANY_ID}`

---

## Comment utiliser cette documentation

Cette documentation est **neutre** et conçue pour s'adapter à n'importe quelle instance SEIGMA. Vous y trouverez des **placeholders** que vous devez remplacer par vos propres valeurs :

| Placeholder | Description | Exemple |
|---|---|---|
| `{VOTRE_INSTANCE}` | Sous-domaine de votre instance SEIGMA | `monentreprise` → `monentreprise.seigma.app` |
| `{VOTRE_EMAIL}` | Courriel du compte API SEIGMA | `api@monentreprise.com` |
| `{VOTRE_MOT_DE_PASSE}` | Mot de passe du compte API | *(votre mot de passe)* |
| `{VOTRE_COMPANY_ID}` | GUID de l'entreprise dans SEIGMA | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `{VOTRE_ENTREPRISE}` | Nom de votre entreprise | `Mon Entreprise Inc.` |
| `{VOTRE_PROJET}` | Chemin vers votre projet | `/chemin/vers/votre/projet/` |

> 🚧 **À compléter** : Obtenez votre `{VOTRE_COMPANY_ID}` depuis l'interface SEIGMA (menu Administration ou Module → Company) ou contactez votre administrateur SEIGMA.

---

## Glossaire métier SEIGMA

| Terme SEIGMA | Signification | ModelCode / Endpoint |
|---|---|---|
| **WO** (Work Order) | Bon de travail | `SalesOrder` |
| **Soumission** | Devis envoyé au client | `Quotation` |
| **Billet** | Appel de service / billet de support | `Call` |
| **Reçu / Encaissement** | Paiement enregistré | `Receipt` |
| **Poinçon / Timelog** | Entrée de temps pointée | `…/timelogs/*` |
| **ModelCode** | Identifiant du modèle dans l'URL | Ex: `SalesOrder`, `Customer` |
| **ReferenceId** | UUID unique d'une fiche | Format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **ModelAttributeCode** | Nom d'un champ dans WhereCondition | Ex: `Display`, `DateCreated` |
| **Pipeline VENTES** | Lead → Quotation → SalesOrder → SalesInvoice → Receipt | N/A |
| **Prospect / Lead** | Contact non qualifié | `Lead` |

---

## Cheat-sheet — Quel endpoint pour quoi ?

| Je veux… | Endpoint |
|---|---|
| Chercher un client par nom | `POST /api/reference/Customer/search` avec `WhereCondition: [{"ModelAttributeCode":"Display","Operator":"=","Value":"Nom"}]` |
| Lire un bon de travail | `GET /api/reference/SalesOrder/{ReferenceId}` |
| Créer une facture | `POST /api/reference/SalesInvoice` |
| Enregistrer un paiement | `POST /api/reference/Receipt` |
| Voir les activités du jour | `POST /api/activity/getactivitiesfordate` |
| Ajouter un poinçon | `POST …/timelogs/add` |
| Lister tous les statuts de WO | `POST /api/reference/SalesOrderStatus/search` |
| Trouver l'ID d'un territoire | `POST /api/reference/Territory/search` |
| Paginer (>50 résultats) | `POST …/search` avec `Offset` et `Limit` |
| Filtrer par plage de dates | `WhereCondition: [{"ModelAttributeCode":"DateCreated","Operator":">=","Value":"2026-01-01"}]` |

---

## Quickstart — Premier appel en 2 minutes

```bash
TOKEN=$(curl -sk -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/auth/authenticate" \
  -H "Content-Type: application/json" \
  -d '{"username":"{VOTRE_EMAIL}","password":"{VOTRE_MOT_DE_PASSE}"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

curl -sk "https://{VOTRE_INSTANCE}.seigma.app/api/reference/SalesOrder/search" \
  -H "Authorization: Bearer $TOKEN" \
  -H "seigma-company: {VOTRE_COMPANY_ID}" \
  -H "Content-Type: application/json" \
  -d '{"Offset":0,"Limit":1}'
```

Tu devrais voir 1 résultat.

---

## Table des matières

| Chapitre | Fichier | Description |
|----------|---------|-------------|
| **1** | [01-authentification.md](01-authentification.md) | Obtenir et utiliser un jeton JWT, headers obligatoires |
| **2** | [02-reference-api.md](02-reference-api.md) | GET/{id} + POST/search — le cœur de l'API |
| **3** | [03-activities-timelogs.md](03-activities-timelogs.md) | Activities (planning) + Timelogs (poinçons) |
| **4** | [04-operations-ecriture.md](04-operations-ecriture.md) | Créer et modifier : SalesOrder, Activity, Timelog |
| **5** | [05-modeles-reference.md](05-modeles-reference.md) | Catalogue exhaustif : 155 modules + 15 fiches détaillées |
| **6** | [06-guides-pratiques.md](06-guides-pratiques.md) | Workflows : facturation, planning, sync externe |
| **7** | [07-pitfalls-limitations.md](07-pitfalls-limitations.md) | Tous les pièges, bugs connus et workarounds |
| **8** | [08-checklist-completion.md](08-checklist-completion.md) | Checklist des 🚧 à compléter pour votre instance |

---

## Parcours de lecture

**🚀 Lecture seule (consulter des données) :** [01](01-authentification.md) → [02](02-reference-api.md) → [06](06-guides-pratiques.md)

**🔧 Intégration complète (lire + écrire) :** [01](01-authentification.md) → [02](02-reference-api.md) → [03](03-activities-timelogs.md) → [04](04-operations-ecriture.md) → [05](05-modeles-reference.md) → [06](06-guides-pratiques.md) → [07](07-pitfalls-limitations.md)

**🛠 Dépannage :** [07](07-pitfalls-limitations.md) → [08](08-checklist-completion.md)

---

## Ce que cette API permet de faire

- 🔍 Rechercher et lire **toutes** les entités SEIGMA : clients, soumissions, bons de travail, factures, reçus, leads, billets, produits
- 📅 Récupérer le planning quotidien des équipes/employés (getactivitiesfordate)
- ⏱️ Gérer les poinçons (timelogs) par WO : consulter, créer, modifier, supprimer
- ✍️ Créer et modifier des bons de travail (SalesOrder)
- 🔄 Synchroniser SEIGMA avec votre base de données (facturation, planification, suivi)

## Ce que cette API NE permet PAS de faire

- ❌ Lire les lignes de commande (SalesOrderLine) — retourne 500, cause inconnue
- ❌ Rechercher des utilisateurs, employés — endpoints cassés (500)
- ❌ Créer des activités standalone (sans module) — POST /api/activity → 404
- ❌ Accéder aux journaux comptables (Invoice, Activity via search) — endpoints cassés

---

## Architecture de l'API

```
https://{VOTRE_INSTANCE}.seigma.app/api/
├── /auth/authenticate          ← JWT (7 jours)
├── /reference/{ModelCode}/{id} ← GET fiche
├── /reference/{ModelCode}/search ← POST rechercher
├── /reference/{ModelCode}/GetReferences ← POST batch resolve
├── /reference/SalesOrder/{id}/timelogs ← GET poinçons
├── /reference/SalesOrder/{id}/timelogs/add ← POST ajouter poinçon
├── /reference/SalesOrder/{id}/timelogs/start ← GET démarrer chrono
├── /reference/SalesOrder/{id}/timelogs/{tid}/stop ← GET arrêter chrono
├── /activity/{id}              ← GET activité
├── /activity/getactivitiesfordate ← POST planning
└── /activity/{refId}           ← POST créer activité liée
```

> **GetReferences** — `POST /api/reference/{ModelCode}/GetReferences` résout des ReferenceId par lot. Body : `{"ReferenceIds":["uuid1","uuid2"]}`. Retourne les fiches complètes pour chaque ID fourni.

---

## Ressources

- **PDFs officiels SEIGMA** : Reference API, Activity API, Timelog API
- **Instance SEIGMA** : https://{VOTRE_INSTANCE}.seigma.app

---

◄ [Index](index.md) │ [Suivant : 01 — Authentification](01-authentification.md) ►
