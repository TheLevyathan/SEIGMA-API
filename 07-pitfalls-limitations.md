# Chapitre 7 — Pièges et limitations

> **Dernière mise à jour** : 2026-06-20
> **Source** : skill seigma-api v1.13.0 + tests de production

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
| **Workaround** | Créer via `POST /api/activity/{referenceId}` (liée à une référence existante) ou planifier côté Supabase |

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

### 12. Operator vs OperatorCode — comportement incohérent

| Champ | Operator | OperatorCode |
|-------|----------|-------------|
| `Number` (SalesOrder) | `"="` ✅ | `"Equal"` ❌ (0 résultat) |
| `SalesOrderStatusId` | `"="` ✅ | `"Equal"` ✅ |
| `Display` | `"Like"` ✅ | — |

**Règle** : Essayer `Operator` d'abord. Si 0 résultat, essayer `OperatorCode`.

### 13. Recherche par numéro de WO — utiliser Number, pas Display

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

### 14. Timeout O(n²) sur grosses requêtes

- **Limite safe** : 200 résultats par requête
- **Au-delà** : timeout exponentiel, échecs de connexion
- **Solution** : pagination avec chunks ≤ 200

### 15. Rate limiting à >50 requêtes parallèles

- **Max safe** : 50 requêtes concurrentes
- **Au-delà** : rate limiting agressif, bannissement temporaire
- **Solution** : semaphore(50) ou chunker par lots de 25-50

---

## Pièges de body

### 16. Body canonique obligatoire pour certains search

```json
// ✅ CORRECT — body complet
{"Offset":0,"Limit":50,"IsSelector":false,"ModelListId":null,"CurrentReferenceId":null,
 "WhereCondition":[],"WhereOrCondition":[],"WhereInCondition":[],"OrderByCondition":[]}

// ⚠️ Toléré sur SalesOrder/search mais PAS sur tous les endpoints
{"offset":0,"limit":3}
// → 500 sur SalesOrderLine et endpoints non testés
```

### 17. Wrapper Properties/Attributes pour l'écriture

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
| `POST /api/reference/Employee/search` | ⚠️ 200 mais 0 résultat pour compte non-admin (corrigé Juin 2026) |
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
