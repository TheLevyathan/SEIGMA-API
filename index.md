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

> 🚧 **À compléter** : Obtenez votre `{VOTRE_COMPANY_ID}` depuis l'interface SEIGMA (menu Administration → Entreprises) ou contactez votre administrateur SEIGMA.

---

## Quickstart — Premier appel en 2 minutes

```bash
# 1. Obtenir un jeton
TOKEN=$(curl -sk -X POST https://exemple.seigma.app/api/auth/authenticate \
  -H "Content-Type: application/json" \
  -d '{"username":"{VOTRE_EMAIL}","password":"{VOTRE_MOT_DE_PASSE}"}' | jq -r '.token')

# 2. Faire un appel
curl -sk -X POST https://exemple.seigma.app/api/reference/Customer/search \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "seigma-company: {VOTRE_COMPANY_ID}" \
  -H "Accept-Language: fr-CA" \
  -H "Content-Type: application/json" \
  -d '{"Offset":0,"Limit":3,"IsSelector":false,"ModelListId":null,"CurrentReferenceId":null,"WhereCondition":[],"WhereOrCondition":[],"WhereInCondition":[],"OrderByCondition":[]}'
```

Tu devrais voir 3 clients.

---

## Table des matières

| Chapitre | Fichier | Description |
|----------|---------|-------------|
| **1** | [01-authentification.md](01-authentification.md) | Obtenir et utiliser un jeton JWT, headers obligatoires |
| **2** | [02-reference-api.md](02-reference-api.md) | GET/{id} + POST/search — le cœur de l'API |
| **3** | [03-activities-timelogs.md](03-activities-timelogs.md) | Activities (planning) + Timelogs (poinçons) |
| **4** | *(planifié)* | Opérations d'écriture (POST/PUT SalesOrder, etc.) |
| **5** | [05-modeles-reference.md](05-modeles-reference.md) | Catalogue des 10+ ModelCodes, relations, pipeline VENTES |
| **6** | [06-guides-pratiques.md](06-guides-pratiques.md) | Workflows : facturation, planning, sync Supabase |
| **7** | [07-pitfalls-limitations.md](07-pitfalls-limitations.md) | Tous les pièges, bugs connus et workarounds |

---

## Ce que cette API permet de faire

- 🔍 Rechercher et lire **toutes** les entités SEIGMA : clients, soumissions, bons de travail, factures, reçus, leads, billets, produits
- 📅 Récupérer le planning quotidien des 9 équipes (getactivitiesfordate)
- ⏱️ Gérer les poinçons (timelogs) par WO : consulter, créer, modifier, supprimer
- ✍️ Créer et modifier des bons de travail (SalesOrder)
- 🔄 Synchroniser SEIGMA ↔ Supabase pour les apps {VOTRE_ENTREPRISE} (facturation, billets, pourboires, reconnaissance)

## Ce que cette API NE permet PAS de faire

- ❌ Lire les lignes de commande (SalesOrderLine) — retourne 500, cause inconnue
- ❌ Rechercher des utilisateurs, employés, factures fournisseur — endpoints cassés (500)
- ❌ Créer des activités standalone — POST /api/activity → 404
- ❌ Accéder aux journaux comptables (Invoice, Activity via search) — endpoints cassés

---

## Architecture de l'API

```
https://{VOTRE_INSTANCE}.seigma.app/api/
├── /auth/authenticate          ← JWT (7 jours)
├── /reference/{ModelCode}/{id} ← GET fiche
├── /reference/{ModelCode}/search ← POST rechercher
├── /reference/{ModelCode}/GetReferences ← POST batch resolve (fonctionne sur 16 modèles)
├── /reference/SalesOrder/{id}/timelogs ← GET poinçons
├── /reference/SalesOrder/{id}/timelogs/add ← POST ajouter poinçon
├── /reference/SalesOrder/{id}/timelogs/start ← GET démarrer chrono
├── /reference/SalesOrder/{id}/timelogs/{tid}/stop ← GET arrêter chrono
├── /activity/{id}              ← GET activité
├── /activity/getactivitiesfordate ← POST planning
└── /activity/{refId}           ← POST créer activité liée
```

---

## Ressources

- **PDFs officiels SEIGMA** : `/mnt/beelink_partage/API/` (Reference API, Activity API, Timelog API)
- **Skill Hermes** : `seigma-api` (expertise terrain, v1.13.0)
- **Code production** : `{VOTRE_PROJET}/supabase/functions/` (seigma-facturation, seigma-timelogs, seigma-clients, bl-vitres-sync)
- **Instance SEIGMA** : https://exemple.seigma.app
