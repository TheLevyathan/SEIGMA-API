# Chapitre 7 — Pièges et limitations

> **Dernière mise à jour** : 2026-06-20
> **Source** : PDFs officiels SEIGMA + tests de production

---

## Table des matières

### Pièges critiques (bloquants)
1. [SalesOrderLine INACCESSIBLE — HTTP 500](#1-salesorderline-inaccessible--http-500)
2. [POST /api/activity → 404 — Création standalone impossible](#2-post-apiactivity--404--création-standalone-impossible)
3. [Activity/search → 500 — Recherche d'activités impossible](#3-activitysearch--500--recherche-dactivités-impossible)
4. [POST /api/timelogs/search avec WorkOrderId → 0 résultats](#4-post-apitimelogssearch-avec-workorderid--0-résultats)

### Pièges de format
5. [DateStart dans getactivitiesfordate — ISO complet obligatoire](#5-datestart-dans-getactivitiesfordate--iso-complet-obligatoire)
6. [UserId dans getactivitiesfordate — OBJET obligatoire](#6-userid-dans-getactivitiesfordate--objet-obligatoire)
7. [UserId dans Timelog — ModelAttributeList (pas UserAssignableModel)](#7-userid-dans-timelog--modelattributelist-pas-userassignablemodel)

### Pièges de données
8. [SalesOrderId est TRANSITIF sur Receipt](#8-salesorderid-est-transitif-sur-receipt)
9. [SelectAttributes IGNORÉ sur Receipt/search et SalesInvoice/search](#9-selectattributes-ignoré-sur-receiptsearch-et-salesinvoicesearch)
10. [Prix des produits à 0.0](#10-prix-des-produits-à-00)
11. [WhereCondition par SalesOrderId sur Call/search → 500](#11-wherecondition-par-salesorderid-sur-callsearch--500)
12. [PaymentMethodId obligatoire pour création de Receipt](#12-paymentmethodid-obligatoire-pour-création-de-receipt)

### Pièges d'opérateur
13. [Operator vs OperatorCode — comportement incohérent](#13-operator-vs-operatorcode--comportement-incohérent)
14. [Contains / StartsWith / EndsWith → HTTP 500 (crash serveur)](#14-contains--startswith--endswith--http-500-crash-serveur)
15. [Recherche par numéro de WO — utiliser Number, pas Display](#15-recherche-par-numéro-de-wo--utiliser-number-pas-display)

### Pièges de performance
16. [Timeout O(n²) sur grosses requêtes](#16-timeout-on²-sur-grosses-requêtes)
17. [Rate limiting à >50 requêtes parallèles](#17-rate-limiting-à-50-requêtes-parallèles)

### Pièges de body
18. [Body canonique obligatoire pour certains search](#18-body-canonique-obligatoire-pour-certains-search)
19. [Wrapper Properties/Attributes pour l'écriture](#19-wrapper-propertiesattributes-pour-lécriture)

### Autres sections
- [Endpoints complètement cassés (500 Object reference)](#endpoints-complètement-cassés-500-object-reference)
- [Incohérences de conception](#incohérences-de-conception)
- [Résumé des workarounds](#résumé-des-workarounds)

---

## Pièges critiques (bloquants)

### 1. SalesOrderLine INACCESSIBLE — HTTP 500

| | |
|---|---|
| **Endpoints** | `POST /api/reference/SalesOrderLine/search`, `GET /api/reference/SalesOrder/{id}/lines`, `POST /api/reference/OrderItem/search` |
| **Erreur** | `"Object reference not set to an instance of an object."` |
| **Impact** | Impossible de lire le détail des lignes de commande (produits, quantités, prix unitaires) |
| **Workaround** | Utiliser `SubTotal`/`Total` du SalesOrder comme montants agrégés. Le `CurrentActivitySubject` ou `Description` donne le type de service |

### 2. POST /api/activity → 404 — Création standalone impossible

| | |
|---|---|
| **Endpoint** | `POST /api/activity` (sans referenceId) |
| **Erreur** | HTTP 404 — aucune action sur le contrôleur |
| **Impact** | Impossible de créer une activité non liée à une référence |
| **Workaround** | Créer via `POST /api/activity/{referenceId}` (liée à une référence existante) ou planifier côté application |

### 3. Activity/search → 500 — Recherche d'activités impossible

| | |
|---|---|
| **Endpoint** | `POST /api/reference/Activity/search` |
| **Erreur** | `"Object reference not set to an instance of an object."` |
| **Workaround** | Utiliser uniquement `POST /api/activity/getactivitiesfordate` (par UserId + date) |

### 4. POST /api/timelogs/search avec WorkOrderId → 0 résultats

| | |
|---|---|
| **Endpoint** | `POST /api/timelogs/search` avec `{"WorkOrderId":"..."}` |
| **Comportement** | Retourne 0 résultats même quand des timelogs existent |
| **Workaround** | **Toujours** utiliser `GET /api/reference/SalesOrder/{id}/timelogs` |

---

## Pièges de format

### 5. DateStart dans getactivitiesfordate — ISO complet obligatoire

| Format | Résultat |
|--------|----------|
| `"2026-05-26T00:00:00"` | ✅ 200 |
| `"2026-05-26"` | ❌ 500 — "Could not convert string to DateTime" |
| `"26-05-2026"` | ❌ 500 — même erreur |

### 6. UserId dans getactivitiesfordate — OBJET obligatoire

```json
// ✅ CORRECT
{"UserId": {"UserId": "...", "Display": "Équipe 1"}, "DateStart": "2026-05-26T00:00:00"}

// ❌ REJETÉ
{"UserId": "d3c672d1-...", "DateStart": "2026-05-26T00:00:00"}
// → "Error converting value to type 'UserAssignableModel'"
```

### 7. UserId dans Timelog — ModelAttributeList (pas UserAssignableModel)

```json
// ✅ Timelog (format différent de Activity !)
{"UserId": {"Display": "Nom", "ReferenceId": "...", "ModelCode": "User"}}

// ❌ Format Activity API
{"UserId": {"UserId": "...", "Display": "Nom"}}
```

---

## Pièges de données

### 8. SalesOrderId est TRANSITIF sur Receipt

`Receipt.SalesOrderId` n'est **jamais** résolu dans le search. C'est un champ transitif :
```
Receipt → SalesInvoice (FK directe, résolue ✅) → SalesOrder (FK directe, absente du search ⚠️)
```

**Conséquence** : Pour obtenir le SalesOrder lié à un Receipt, il faut :
1. GET /api/reference/Receipt/{id} → extraire `SalesInvoiceId`
2. GET /api/reference/SalesInvoice/{salesInvoiceId} → extraire `SalesOrderId`
3. GET /api/reference/SalesOrder/{salesOrderId}

**Optimisation** : chunker les getDetail par lots de 25 en parallèle (pattern `chunkedDetails`).

### 9. SelectAttributes IGNORÉ sur Receipt/search et SalesInvoice/search

`SelectAttributes` est **accepté sans erreur** mais **complètement ignoré** sur ces 2 modèles. Les 14 champs par défaut sont toujours retournés, sans filtrage ni ajout.

**Modèles où SelectAttributes fonctionne** : SalesOrder, Quotation, Call, Lead, Customer, Product, PaymentTerm, Warehouse.

### 10. Prix des produits à 0.0

Tous les 21 produits ont `Price: 0.0` dans leur définition. Le prix réel est dans `SubTotal`/`Total` du SalesOrder.

### 11. WhereCondition par SalesOrderId sur Call/search → 500

```json
// ❌ CRASH
{"WhereCondition": [{"ModelAttributeCode":"SalesOrderId","Operator":"=","Value":"..."}]}
// → "The multi-part identifier \"dbo.CallView.SalesOrder.Display_2\" could not be bound."
```

**Workaround** : Récupérer tous les Calls (95 max), filtrer côté client.

### 12. PaymentMethodId obligatoire pour création de Receipt

`POST /api/reference/Receipt` exige **obligatoirement** `PaymentMethodId` (en plus de `CustomerId` et `Amount`). Sans cela, l'API retourne `400`.

**Workaround** : Utiliser `POST /api/reference/PaymentMethod/search` pour obtenir les méthodes disponibles. Exemple : « Carte de crédit » (`52bb87e0-...`).

---

## Pièges d'opérateur

### 13. Operator vs OperatorCode — comportement incohérent

| Champ | Operator | OperatorCode |
|-------|----------|-------------|
| `Number` (SalesOrder) | `"="` ✅ | `"Equal"` ❌ (retourne HTTP 500 — crash serveur) |
| `SalesOrderStatusId` | `"="` ✅ | `"Equal"` ✅ |
| `Display` | `"Like"` ✅ | — |

**Règle** : Essayer `Operator` d'abord. Si 0 résultat, essayer `OperatorCode`.

### 14. Contains / StartsWith / EndsWith → HTTP 500 (crash serveur)

Les opérateurs `Contains`, `StartsWith` et `EndsWith` retournent **HTTP 500 Internal Server Error** sur certains champs (notamment `SalesOrderStatusId`). Le serveur crash au lieu de retourner des résultats.

**Workaround** : Utiliser exclusivement l'opérateur `"="` pour les filtres sur ces attributs problématiques.

### 15. Recherche par numéro de WO — utiliser Number, pas Display

```json
// ✅ Fiable
{"ModelAttributeCode": "Number", "Operator": "=", "Value": "713"}
// → WO-00713

// ⚠️ Moins fiable
{"ModelAttributeCode": "Display", "Operator": "Like", "Value": "WO-00713"}
```

La valeur est le numéro **SANS le préfixe** (ex: `"713"` pour WO-00713).

---

## Pièges de performance

### 16. Timeout O(n²) sur grosses requêtes

- **Limite recommandée** : pas de plafond strict observé (testé jusqu'à 1000). Pour des performances optimales, paginez par tranches de 200.
- **Au-delà** : timeout exponentiel, échecs de connexion
- **Solution** : pagination avec chunks ≤ 200

### 17. Rate limiting à >50 requêtes parallèles

- **Max safe** : 50 requêtes concurrentes
- **Au-delà** : rate limiting agressif, bannissement temporaire
- **Solution** : semaphore(50) ou chunker par lots de 25-50

---

## Pièges de body

### 18. Body canonique obligatoire pour certains search

```json
// ✅ CORRECT — body complet
{"Offset":0,"Limit":50,"IsSelector":false,"ModelListId":null,"CurrentReferenceId":null,
 "WhereCondition":[],"WhereOrCondition":[],"WhereInCondition":[],"OrderByCondition":[]}

// ⚠️ Toléré sur SalesOrder/search mais PAS sur tous les endpoints
{"offset":0,"limit":3}
// → 500 sur SalesOrderLine et endpoints non testés
```

### 19. Wrapper Properties/Attributes pour l'écriture

```json
// ✅ CORRECT — création SalesOrder
{"Properties": {"CustomerId": {"ReferenceId":"..."}, "PaymentTermId": {"ReferenceId":"..."},
  "WarehouseId": {"ReferenceId":"..."}, "ShippingStreet":"123 Rue Test"}}

// ❌ REJETÉ — champs scalaires directs
{"ShippingStreet":"123 Rue Test", "CustomerId":"guid-here"}
// → "Error converting value to type Dictionary`2"
```

---

## Endpoints complètement cassés (500 Object reference)

> ℹ️ **Note** : Les endpoints listés ci-dessous peuvent avoir été corrigés dans des versions plus récentes de SEIGMA. Testez toujours avec un appel minimal avant d'intégrer — ces observations sont basées sur une version spécifique testée en juin 2026.

| Endpoint | Status |
|----------|--------|
| `POST /api/reference/User/search` | 500 — Object reference not set |
| `POST /api/reference/Employee/search` | ✅ CORRIGÉ — Juin 2026 : retourne 200 (0 résultat pour compte non-admin, résultats visibles avec compte admin) |
| `POST /api/reference/Invoice/search` | 500 — Object reference not set |
| `POST /api/reference/Activity/search` | 500 — Object reference not set |
| `POST /api/reference/SalesOrderLine/search` | 500 — Object reference not set |
| `GET /api/reference/SalesOrder/{id}/lines` | 500 — Object reference not set |
| `POST /api/reference/Contact/search` | 500 — Object reference not set |
| `POST /api/reference/Report/search` | 500 — Object reference not set |
| `GET /api/reference/{ModelCode}/metadata` | 500 — Endpoint inexistant (pas un endpoint valide) |

⚠️ Ces endpoints existent (pas de 404) mais crashent côté serveur — probablement un bug SEIGMA ou des paramètres obligatoires non documentés.

---

## Incohérences de conception

| Problème | Détail |
|----------|--------|
| UserId format différent Activity vs Timelog | Activity = `UserAssignableModel {UserId, Display}`, Timelog = `ModelAttributeList {ReferenceId, Display, ModelCode}` |
| SelectAttributes comportement incohérent | Fonctionne sur 8 modèles, ignoré sur 2 (Receipt, SalesInvoice) |
| Operator vs OperatorCode | Comportement différent selon le champ, pas de règle cohérente |
| Body canonique vs simplifié | Le body simplifié fonctionne sur SalesOrder/search mais casse d'autres endpoints |

---

## Résumé des workarounds

| Problème | Workaround |
|----------|------------|
| SalesOrderLine inaccessible | SubTotal/Total + Description |
| Activity/search cassé | getactivitiesfordate uniquement |
| Timelog/search par WO cassé | GET /api/reference/SalesOrder/{id}/timelogs |
| SalesOrderId transitif sur Receipt | chunkedDetails (getDetail Receipt → SalesInvoice → SalesOrder) |
| SelectAttributes ignoré sur Receipt/SalesInvoice | getDetail unitaire obligatoire |
| Call/search par SalesOrderId → 500 | Fetch all + filtre client |
| PaymentMethodId obligatoire pour Receipt | POST /api/reference/PaymentMethod/search |
| Rate limiting | semaphore(50), chunks de 25-50 |

---

◄ [Précédent : 06 — Guides pratiques](06-guides-pratiques.md) │ [Index](index.md) │ [Suivant : 08 — Checklist de complétion](08-checklist-completion.md) ►
