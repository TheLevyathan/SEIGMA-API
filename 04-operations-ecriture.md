# Chapitre 4 — Opérations d'écriture

**Table des matières**

- [4.1 Format universel d'écriture](#41-format-universel-décriture)
- [4.2 Créer un bon de travail — `POST /api/reference/SalesOrder`](#42-créer-un-bon-de-travail--post-apireferencesalesorder)
- [4.3 Modifier un bon de travail — `PUT /api/reference/SalesOrder/{id}`](#43-modifier-un-bon-de-travail--put-apireferencesalesorderid)
- [4.4 Créer une facture — `POST /api/reference/SalesInvoice`](#44-créer-une-facture--post-apireferencesalesinvoice)
- [4.5 Créer un encaissement — `POST /api/reference/Receipt`](#45-créer-un-encaissement--post-apireferencereceipt)
- [4.6 Créer une activité liée — `POST /api/activity/{referenceId}`](#46-créer-une-activité-liée--post-apiactivityreferenceid)
- [4.7 Modifier une activité — `PUT /api/activity/{activityId}`](#47-modifier-une-activité--put-apiactivityactivityid)
- [4.8 Ajouter un poinçon — `POST …/timelogs/add`](#48-ajouter-un-poinçon--post-timelogsadd)
- [4.9 Modifier un poinçon — `PUT …/timelogs`](#49-modifier-un-poinçon--put-timelogs)
- [4.10 Supprimer un poinçon — `DELETE …/timelogs/{timelogId}`](#410-supprimer-un-poinçon--delete-timelogstimelogid)
- [4.11 Démarrer un chrono — `GET …/timelogs/start`](#411-démarrer-un-chrono--get-timelogsstart)
- [4.12 Arrêter un chrono — `GET …/timelogs/{timelogId}/stop`](#412-arrêter-un-chrono--get-timelogstimelogidstop)
- [4.13 Référentiels nécessaires](#413-référentiels-nécessaires)
- [4.14 Pièges et limitations](#414-pièges-et-limitations)
- [4.15 Exemples complets](#415-exemples-complets)

---

## 4.1 Format universel d'écriture

Toute opération d'écriture dans SEIGMA suit une règle simple : **les champs scalaires sont wrappés dans `"Properties"` ou `"Attributes"`, et les IDs de référence sont des objets `{"ReferenceId":"..."}`**.

```json
// ✅ CORRECT — champs dans "Properties", IDs en objets
{
  "Properties": {
    "ShippingStreet": "123 Rue Test",
    "CustomerId": {"ReferenceId": "550e8400-e29b-41d4-a716-446655440000"}
  }
}

// ❌ REJETÉ — champs scalaires directs, IDs en string
{
  "ShippingStreet": "123 Rue Test",
  "CustomerId": "550e8400-e29b-41d4-a716-446655440000"
}
// → 500 "Error converting value to type Dictionary`2"
```

> 🚧 **Règle d'or :** si vous voyez `500 Error converting value to type Dictionary\`2`, vérifiez qu'aucun champ n'est envoyé à la racine. Tout doit être dans `"Properties"` (ou `"Attributes"` selon l'endpoint).

---

## 4.2 Créer un bon de travail — `POST /api/reference/SalesOrder`

Crée un bon de travail (SalesOrder) dans SEIGMA.

| Élément | Détail |
|---|---|
| **URL** | `POST {VOTRE_INSTANCE}/api/reference/SalesOrder` |
| **Status succès** | `201 Created` |
| **Authentification** | Bearer token (voir Chapitre 2) |

### 4.2.1 Champs obligatoires

Trois champs sont **strictement requis** sous forme d'objets `ReferenceId`. Sans eux, l'API retourne `400`.

| Champ SEIGMA | Format | Message d'erreur si absent |
|---|---|---|
| `CustomerId` | `{"ReferenceId": "{id-client}"}` | `"[Client] doit être défini"` |
| `PaymentTermId` | `{"ReferenceId": "{id-terme-paiement}"}` | `"[Termes] doit être défini"` |
| `WarehouseId` | `{"ReferenceId": "{id-entrepot}"}` | `"[Entrepôt] doit être défini"` |

### 4.2.2 Champs optionnels

| Champ | Type | Description |
|---|---|---|
| `ShippingStreet` | string | Adresse de livraison — rue |
| `ShippingCity` | string | Ville de livraison |
| `ShippingPostalCode` | string | Code postal |
| `ShippingName` | string | Nom du destinataire |
| `Description` | string | Description du bon de travail |

### 4.2.3 Corps canonique

```json
{
  "Properties": {
    "CustomerId": {"ReferenceId": "{un-id-client}"},
    "PaymentTermId": {"ReferenceId": "{un-id-terme-paiement}"},
    "WarehouseId": {"ReferenceId": "{un-id-entrepot}"},
    "ShippingStreet": "123 Rue Exemple",
    "ShippingCity": "Terrebonne",
    "ShippingPostalCode": "J6W 1A1",
    "ShippingName": "Client Nom",
    "Description": "Description du bon"
  }
}
```

### 4.2.4 Exemple cURL

```bash
curl -X POST "{VOTRE_INSTANCE}/api/reference/SalesOrder" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Properties": {
      "CustomerId": {"ReferenceId": "550e8400-e29b-41d4-a716-446655440000"},
      "PaymentTermId": {"ReferenceId": "660e8400-e29b-41d4-a716-446655440001"},
      "WarehouseId": {"ReferenceId": "770e8400-e29b-41d4-a716-446655440002"},
      "Description": "Bon créé via API"
    }
  }'
```

### 4.2.5 Réponses

**Succès (201 Created) :**

```json
{
  "successMessages": ["Sauvegarde réussie"],
  "reference": {
    "ReferenceId": "880e8400-e29b-41d4-a716-446655440003",
    "ModelCode": "SalesOrder",
    "Properties": { "…": "…" }
  }
}
```

**Erreur (400 Bad Request) — champs obligatoires manquants :**

```json
{
  "message": "Erreur durant la sauvegarde.",
  "errorMessages": [
    "[Termes] doit être défini",
    "[Client] doit être défini",
    "[Entrepôt] doit être défini"
  ]
}
```

> ⚠️ **Attention :** les noms de champs dans `errorMessages` sont en **français** (ex: « Termes » pour PaymentTermId, « Entrepôt » pour WarehouseId). Ne vous fiez pas aux noms anglais pour parser ces erreurs.

---

## 4.3 Modifier un bon de travail — `PUT /api/reference/SalesOrder/{id}`

Modifie un bon de travail existant. Comportement **PATCH-like** : seuls les champs envoyés sont modifiés.

| Élément | Détail |
|---|---|
| **URL** | `PUT {VOTRE_INSTANCE}/api/reference/SalesOrder/{referenceId}` |
| **Status succès** | `200 OK` |

### 4.3.1 Corps de requête

Même format `"Properties"` que la création. Envoyez **uniquement les champs à modifier**.

```json
{
  "Properties": {
    "ShippingCity": "Montréal",
    "Description": "Description mise à jour"
  }
}
```

### 4.3.2 Exemple cURL

```bash
curl -X PUT "{VOTRE_INSTANCE}/api/reference/SalesOrder/{referenceId}" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Properties": {
      "ShippingCity": "Montréal"
    }
  }'
```

### 4.3.3 Réponse

```json
{
  "successMessages": ["Sauvegarde réussie"],
  "reference": { "…": "…" }
}
```

> ⚠️ **Piège :** envoyer un champ avec une valeur vide (`""`) ou `null` **vide effectivement** ce champ dans SEIGMA. Pour préserver la valeur existante, ne l'incluez pas dans le corps.

---

## 4.4 Créer une facture — `POST /api/reference/SalesInvoice`

Crée une facture de vente (SalesInvoice) dans SEIGMA.

| Élément | Détail |
|---|---|
| **URL** | `POST {VOTRE_INSTANCE}/api/reference/SalesInvoice` |
| **Status succès** | `201 Created` |
| **Authentification** | Bearer token (voir Chapitre 2) |

### 4.4.1 Champs obligatoires

Deux champs sont **strictement requis** sous forme d'objets `ReferenceId`.

| Champ SEIGMA | Format | Message d'erreur si absent |
|---|---|---|
| `CustomerId` | `{"ReferenceId": "{id-client}"}` | `"[Client] doit être défini"` |
| `PaymentTermId` | `{"ReferenceId": "{id-terme-paiement}"}` | `"[Termes] doit être défini"` |

> ⚠️ **Pas de WarehouseId** : Contrairement à `SalesOrder`, `WarehouseId` **n'est PAS requis** pour créer une `SalesInvoice`.

### 4.4.2 Champs optionnels notables

| Champ | Type | Description |
|---|---|---|
| `SalesOrderId` | `ReferenceId` | Bon de travail lié (`null` si non lié) |
| `DateInvoiced` | DateTime ISO | Date de facturation |
| `Total` | decimal | Total |
| `Balance` | decimal | Solde dû |
| `Description` | string | Description |

### 4.4.3 Corps canonique

```json
{
  "Properties": {
    "CustomerId": {"ReferenceId": "{un-id-client}"},
    "PaymentTermId": {"ReferenceId": "{un-id-terme-paiement}"},
    "DateInvoiced": "2026-06-20T00:00:00",
    "Total": 1500.00,
    "Balance": 1500.00,
    "Description": "Facture créée via API"
  }
}
```

### 4.4.4 Exemple cURL

```bash
curl -X POST "{VOTRE_INSTANCE}/api/reference/SalesInvoice" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Properties": {
      "CustomerId": {"ReferenceId": "550e8400-e29b-41d4-a716-446655440000"},
      "PaymentTermId": {"ReferenceId": "660e8400-e29b-41d4-a716-446655440001"},
      "Description": "Facture créée via API"
    }
  }'
```

### 4.4.5 Réponse

**Succès (201 Created) :**

```json
{
  "successMessages": ["Sauvegarde réussie"],
  "reference": {
    "ReferenceId": "990e8400-e29b-41d4-a716-446655440004",
    "ModelCode": "SalesInvoice",
    "Display": "SI-XXXXX",
    "Properties": { "…": "…" }
  }
}
```

Le `Display` suit le format `SI-XXXXX` (numéro de facture auto-généré).

---

## 4.5 Créer un encaissement — `POST /api/reference/Receipt`

Crée un encaissement (Receipt) dans SEIGMA.

| Élément | Détail |
|---|---|
| **URL** | `POST {VOTRE_INSTANCE}/api/reference/Receipt` |
| **Status succès** | `201 Created` |
| **Authentification** | Bearer token (voir Chapitre 2) |

### 4.5.1 Champs obligatoires

Trois champs sont **strictement requis**.

| Champ SEIGMA | Format | Détail |
|---|---|---|
| `CustomerId` | `{"ReferenceId": "{id-client}"}` | Client qui paie |
| `Amount` | `decimal` | Montant encaissé |
| `PaymentMethodId` | `{"ReferenceId": "{id-methode-paiement}"}` | ⚠️ **Obligatoire** — mode de paiement |

> ⚠️ **PaymentMethodId est obligatoire !** Vous devez d'abord rechercher les méthodes de paiement disponibles via `POST /api/reference/PaymentMethod/search`. Exemple de méthode trouvée : « Carte de crédit » (`{un-id-paymentmethod}`).

### 4.5.2 Champs optionnels notables

| Champ | Type | Description |
|---|---|---|
| `SalesInvoiceId` | `ReferenceId` | Facture liée |
| `PaymentDate` | DateTime ISO | Date du paiement |
| `Reference` | string | Référence du paiement |
| `TransactionId` | string | ID de transaction |

### 4.5.3 Corps canonique

```json
{
  "Properties": {
    "CustomerId": {"ReferenceId": "{un-id-client}"},
    "Amount": 500.00,
    "PaymentMethodId": {"ReferenceId": "{un-id-paymentmethod}"},
    "PaymentDate": "2026-06-20T00:00:00",
    "Reference": "Paiement facture SI-00123"
  }
}
```

### 4.5.4 Exemple cURL

```bash
curl -X POST "{VOTRE_INSTANCE}/api/reference/Receipt" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Properties": {
      "CustomerId": {"ReferenceId": "550e8400-e29b-41d4-a716-446655440000"},
      "Amount": 500.00,
      "PaymentMethodId": {"ReferenceId": "{un-id-paymentmethod}"}
    }
  }'
```

### 4.5.5 Réponse

**Succès (201 Created) :**

```json
{
  "successMessages": ["Sauvegarde réussie"],
  "reference": {
    "ReferenceId": "aa0e8400-e29b-41d4-a716-446655440006",
    "ModelCode": "Receipt",
    "Display": "RE-XXXXX",
    "Properties": { "…": "…" }
  }
}
```

Le `Display` suit le format `RE-XXXXX` (numéro d'encaissement auto-généré).

---

## 4.6 Créer une activité liée — `POST /api/activity/{referenceId}`

Crée une activité (rendez-vous, tâche, note) attachée à une référence existante.

| Élément | Détail |
|---|---|
| **URL** | `POST {VOTRE_INSTANCE}/api/activity/{referenceId}` |
| **Status succès** | `201 Created` |

> 🚧 **Important :** `POST /api/activity` **sans** `referenceId` retourne `404`. Les activités doivent **obligatoirement** être liées à une référence existante.

### 4.6.1 Format du corps

Le corps est wrappé dans `"activity"` :

```json
{
  "activity": {
    "ActivityTypeId": { "…": "…" },
    "ActivityFollowUpId": { "…": "…" }
  }
}
```

### 4.6.2 Champs obligatoires

| Champ | Format | Détail |
|---|---|---|
| `ActivityTypeId` | `ModelAttributeList` | `{"Display": "Appel", "ReferenceId": "{id}", "ModelCode": "ActivityType"}` |
| `ActivityFollowUpId` | `ModelAttributeList` | `{"Display": "À faire", "ReferenceId": "{id}", "ModelCode": "ActivityFollowUp"}` |

### 4.6.3 Champs optionnels

| Champ | Type | Description |
|---|---|---|
| `Subject` | string | Sujet de l'activité |
| `StartDate` | DateTime ISO | Date/heure de début |
| `EndDate` | DateTime ISO | Date/heure de fin |
| `AssignedToId` | objet | `{"UserId": "{id}", "Display": "Nom"}` |
| `Notes` | string | Notes textuelles |
| `Location` | string | Lieu |
| `ActivityLabelId` | objet | Étiquette |
| `ContactMethodId` | objet | Méthode de contact |
| `IsCompleted` | bool | Activité terminée ? |
| `AllDay` | bool | Toute la journée ? |

### 4.6.4 Corps canonique

```json
{
  "activity": {
    "ActivityTypeId": {
      "Display": "Appel",
      "ReferenceId": "{activity-type-id}",
      "ModelCode": "ActivityType"
    },
    "ActivityFollowUpId": {
      "Display": "À faire",
      "ReferenceId": "{followup-id}",
      "ModelCode": "ActivityFollowUp"
    },
    "Subject": "Appel de suivi client",
    "StartDate": "2026-06-22T09:00:00",
    "Notes": "Vérifier l'état de la commande"
  }
}
```

### 4.6.5 Exemple cURL

```bash
curl -X POST "{VOTRE_INSTANCE}/api/activity/{referenceId}" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "activity": {
      "ActivityTypeId": {
        "Display": "Appel",
        "ReferenceId": "aaa-bbb-ccc",
        "ModelCode": "ActivityType"
      },
      "ActivityFollowUpId": {
        "Display": "À faire",
        "ReferenceId": "ddd-eee-fff",
        "ModelCode": "ActivityFollowUp"
      },
      "Subject": "Appel de suivi"
    }
  }'
```

### 4.6.6 Réponse

```json
{
  "activity": {
    "ActivityId": "nouvel-id-activite",
    "Subject": "Appel de suivi",
    "…": "…"
  }
}
```

---

## 4.7 Modifier une activité — `PUT /api/activity/{activityId}`

Modifie une activité existante. Seuls les champs envoyés sont mis à jour.

| Élément | Détail |
|---|---|
| **URL** | `PUT {VOTRE_INSTANCE}/api/activity/{activityId}` |
| **Status succès** | `200 OK` |

### 4.7.1 Corps de requête

Mêmes champs modifiables qu'à la création. Les champs absents sont ignorés.

```json
{
  "activity": {
    "Subject": "Sujet modifié",
    "IsCompleted": true
  }
}
```

### 4.7.2 Exemple cURL

```bash
curl -X PUT "{VOTRE_INSTANCE}/api/activity/{activityId}" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "activity": {
      "IsCompleted": true
    }
  }'
```

### 4.7.3 Réponse

```json
{
  "activity": {
    "ActivityId": "{activityId}",
    "Subject": "Sujet modifié",
    "IsCompleted": true,
    "…": "…"
  }
}
```

---

## 4.8 Ajouter un poinçon — `POST …/timelogs/add`

Ajoute une entrée de temps (poinçon) sur une référence.

| Élément | Détail |
|---|---|
| **URL** | `POST {VOTRE_INSTANCE}/api/reference/{ModelCode}/{refId}/timelogs/add` |
| **Status succès** | `200 OK` |

### 4.8.1 Format du corps

Le corps est wrappé dans `"timeLog"` :

```json
{
  "timeLog": {
    "StartTime": "…",
    "UserId": { "…": "…" }
  }
}
```

### 4.8.2 Champs obligatoires

| Champ | Format | Détail |
|---|---|---|
| `StartTime` | DateTime ISO | Date et heure de début du poinçon |
| `UserId` | objet | `{"Display": "Nom", "ReferenceId": "{id}", "ModelCode": "User"}` |

> ⚠️ **Attention :** le format du `UserId` dans les poinçons (`{Display, ReferenceId, ModelCode:"User"}`) est **différent** du `AssignedToId` des activités (`{UserId, Display}`). Ne les mélangez pas.

### 4.8.3 Champs optionnels

| Champ | Type | Description |
|---|---|---|
| `EndTime` | DateTime ISO | Date/heure de fin |
| `LogDate` | DateTime ISO | Date du poinçon |
| `Description` | string | Description textuelle |

### 4.8.4 Corps canonique

```json
{
  "timeLog": {
    "StartTime": "2026-06-20T08:00:00",
    "EndTime": "2026-06-20T12:00:00",
    "UserId": {
      "Display": "Jean Tremblay",
      "ReferenceId": "{user-guid}",
      "ModelCode": "User"
    },
    "Description": "Inspection initiale"
  }
}
```

### 4.8.5 Exemple cURL

```bash
curl -X POST "{VOTRE_INSTANCE}/api/reference/SalesOrder/{refId}/timelogs/add" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "timeLog": {
      "StartTime": "2026-06-20T08:00:00",
      "UserId": {
        "Display": "Jean Tremblay",
        "ReferenceId": "{user-guid}",
        "ModelCode": "User"
      }
    }
  }'
```

### 4.8.6 Réponse

```json
{
  "timeLog": {
    "TimeLogId": "{nouvel-id}",
    "StartTime": "2026-06-20T08:00:00",
    "EndTime": null,
    "UserId": { "…": "…" }
  }
}
```

---

## 4.9 Modifier un poinçon — `PUT …/timelogs`

Modifie une entrée de temps existante.

| Élément | Détail |
|---|---|
| **URL** | `PUT {VOTRE_INSTANCE}/api/reference/{ModelCode}/{refId}/timelogs` |
| **Status succès** | `200 OK` |

### 4.9.1 Corps de requête

Le `TimeLogId` est **obligatoire** dans le corps. Les autres champs sont facultatifs.

```json
{
  "timeLog": {
    "TimeLogId": "{timelogId}",
    "Description": "Description mise à jour",
    "EndTime": "2026-06-20T17:00:00"
  }
}
```

**Restrictions :**
- On ne peut **pas** changer de référence (`refId`).
- On ne peut **pas** modifier le `LogDate`.
- Si `StartTime` ou `UserId` sont `null` → ils sont **ignorés** (aucune modification).

### 4.9.2 Exemple cURL

```bash
curl -X PUT "{VOTRE_INSTANCE}/api/reference/SalesOrder/{refId}/timelogs" \
  -H "Authorization: Bearer {VOTRE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "timeLog": {
      "TimeLogId": "{timelogId}",
      "EndTime": "2026-06-20T17:00:00"
    }
  }'
```

### 4.9.3 Réponse

```json
{
  "timeLog": {
    "TimeLogId": "{timelogId}",
    "StartTime": "2026-06-20T08:00:00",
    "EndTime": "2026-06-20T17:00:00",
    "…": "…"
  }
}
```

---

## 4.10 Supprimer un poinçon — `DELETE …/timelogs/{timelogId}`

Supprime définitivement une entrée de temps.

| Élément | Détail |
|---|---|
| **URL** | `DELETE {VOTRE_INSTANCE}/api/reference/{ModelCode}/{refId}/timelogs/{timelogId}` |
| **Status succès** | `204 No Content` |
| **Status refusé** | `403 Forbidden` (droits insuffisants) |

### 4.10.1 Exemple cURL

```bash
curl -X DELETE "{VOTRE_INSTANCE}/api/reference/SalesOrder/{refId}/timelogs/{timelogId}" \
  -H "Authorization: Bearer {VOTRE_TOKEN}"
```

### 4.10.2 Réponse

Succès : corps vide, status `204`.

---

## 4.11 Démarrer un chrono — `GET …/timelogs/start`

Démarre un chronomètre (TimeLog sans `EndTime`) sur une référence.

| Élément | Détail |
|---|---|
| **URL** | `GET {VOTRE_INSTANCE}/api/reference/{ModelCode}/{refId}/timelogs/start` |
| **Status succès** | `200 OK` |

### 4.11.1 Comportement

- Crée un TimeLog avec l'heure courante comme `StartTime` et `EndTime = null`.
- Si un TimeLog **sans EndTime** existe déjà pour cet utilisateur sur cette référence → retourne le TimeLog existant (pas de nouveau).

### 4.11.2 Exemple cURL

```bash
curl -X GET "{VOTRE_INSTANCE}/api/reference/SalesOrder/{refId}/timelogs/start" \
  -H "Authorization: Bearer {VOTRE_TOKEN}"
```

### 4.11.3 Réponse

```json
{
  "timeLog": {
    "TimeLogId": "{timelogId}",
    "StartTime": "2026-06-20T14:30:00",
    "EndTime": null
  }
}
```

---

## 4.12 Arrêter un chrono — `GET …/timelogs/{timelogId}/stop`

Arrête un chronomètre en cours.

| Élément | Détail |
|---|---|
| **URL** | `GET {VOTRE_INSTANCE}/api/reference/{ModelCode}/{refId}/timelogs/{timelogId}/stop` |
| **Status succès** | `200 OK` |

### 4.12.1 Comportement

Définit `EndTime` à l'heure courante sur le TimeLog spécifié.

### 4.12.2 Exemple cURL

```bash
curl -X GET "{VOTRE_INSTANCE}/api/reference/SalesOrder/{refId}/timelogs/{timelogId}/stop" \
  -H "Authorization: Bearer {VOTRE_TOKEN}"
```

### 4.12.3 Réponse

```json
{
  "timeLog": {
    "TimeLogId": "{timelogId}",
    "StartTime": "2026-06-20T14:30:00",
    "EndTime": "2026-06-20T16:45:00"
  }
}
```

---

## 4.13 Référentiels nécessaires

Avant de créer un bon de travail, une facture ou un encaissement, vous devez obtenir les IDs des référentiels obligatoires.

### 4.13.1 Obtenir un `CustomerId`

```bash
curl -X GET "{VOTRE_INSTANCE}/api/search?modelCode=Customer&searchTerm={nom}" \
  -H "Authorization: Bearer ***
```

### 4.13.2 Obtenir un `PaymentTermId`

```bash
curl -X GET "{VOTRE_INSTANCE}/api/search?modelCode=PaymentTerm&searchTerm={terme}" \
  -H "Authorization: Bearer ***
```

### 4.13.3 Obtenir un `WarehouseId`

```bash
curl -X GET "{VOTRE_INSTANCE}/api/search?modelCode=Warehouse&searchTerm={entrepot}" \
  -H "Authorization: Bearer ***
```

### 4.13.4 Obtenir un `PaymentMethodId` ⚠️ requis pour Receipt

```bash
curl -X POST "{VOTRE_INSTANCE}/api/reference/PaymentMethod/search" \
  -H "Authorization: Bearer *** \
  -H "Content-Type: application/json" \
  -d '{
    "Offset": 0,
    "Limit": 50,
    "IsSelector": false,
    "ModelListId": null,
    "CurrentReferenceId": null,
    "WhereCondition": [],
    "WhereOrCondition": [],
    "WhereInCondition": [],
    "OrderByCondition": []
  }'
```

> ⚠️ **PaymentMethodId est obligatoire** pour la création de `Receipt`. Utilisez `POST /api/reference/PaymentMethod/search` pour obtenir les méthodes disponibles. Exemple : « Carte de crédit » (`{un-id-paymentmethod}`).

> 🚧 **Spécifique à l'instance :** les IDs de référentiels varient d'une instance SEIGMA à l'autre. Ne les hardcodez jamais. Utilisez toujours les endpoints `/api/search` ou `/api/reference/{ModelCode}/search` pour les résoudre dynamiquement.

---

## 4.14 Pièges et limitations

| # | Piège | Conséquence | Solution |
|---|---|---|---|
| 1 | Champs scalaires hors `Properties` | `500 Error converting value to type Dictionary\`2` | Toujours wrapper dans `"Properties"` ou `"Attributes"` |
| 2 | IDs en string au lieu d'objets `ReferenceId` | `500` ou champ ignoré | Toujours `{"ReferenceId": "guid"}` |
| 3 | `POST /api/activity` sans `referenceId` | `404 Not Found` | Toujours `/api/activity/{referenceId}` |
| 4 | PUT avec champs vides/null | Le champ est **vidé** dans SEIGMA | Omettre les champs à préserver |
| 5 | Noms d'erreur en français | `"Termes"` pour PaymentTermId, `"Entrepôt"` pour WarehouseId | Parser les `errorMessages` en français |
| 6 | `UserId` poinçon ≠ `AssignedToId` activité | Formats incompatibles | Poinçon : `{Display, ReferenceId, ModelCode:"User"}`. Activité : `{UserId, Display}` |
| 7 | `PUT …/timelogs` sans `TimeLogId` | Erreur | Toujours inclure `TimeLogId` dans le corps |
| 8 | `LogDate` non modifiable en PUT | Valeur inchangée | Pas de solution — c'est verrouillé par SEIGMA |
| 9 | `PaymentMethodId` obligatoire pour Receipt | `400` si absent | Utiliser `POST /api/reference/PaymentMethod/search` pour l'obtenir |

---

## 4.15 Exemples complets

### 4.15.1 TypeScript — Créer un bon de travail

```typescript
const BASE_URL = "{VOTRE_INSTANCE}";
const TOKEN = "{VOTRE_TOKEN}";

interface ReferenceId {
  ReferenceId: string;
}

interface SalesOrderProperties {
  CustomerId: ReferenceId;
  PaymentTermId: ReferenceId;
  WarehouseId: ReferenceId;
  ShippingStreet?: string;
  ShippingCity?: string;
  ShippingPostalCode?: string;
  ShippingName?: string;
  Description?: string;
}

interface SalesOrderRequest {
  Properties: SalesOrderProperties;
}

async function createSalesOrder(props: SalesOrderProperties): Promise<any> {
  const body: SalesOrderRequest = { Properties: props };

  const response = await fetch(`${BASE_URL}/api/reference/SalesOrder`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    const err = await response.json();
    throw new Error(`Erreur ${response.status}: ${JSON.stringify(err)}`);
  }

  return response.json();
}

// Usage
const result = await createSalesOrder({
  CustomerId: { ReferenceId: "550e8400-e29b-41d4-a716-446655440000" },
  PaymentTermId: { ReferenceId: "660e8400-e29b-41d4-a716-446655440001" },
  WarehouseId: { ReferenceId: "770e8400-e29b-41d4-a716-446655440002" },
  Description: "Bon créé via TypeScript",
});

console.log("Bon créé :", result.reference.ReferenceId);
```

### 4.15.2 Python — Créer un bon de travail

```python
import requests

BASE_URL = "{VOTRE_INSTANCE}"
TOKEN = "{VOTRE_TOKEN}"

def create_sales_order(
    customer_id: str,
    payment_term_id: str,
    warehouse_id: str,
    description: str = None,
) -> dict:
    """Crée un bon de travail dans SEIGMA."""
    body = {
        "Properties": {
            "CustomerId": {"ReferenceId": customer_id},
            "PaymentTermId": {"ReferenceId": payment_term_id},
            "WarehouseId": {"ReferenceId": warehouse_id},
        }
    }

    if description:
        body["Properties"]["Description"] = description

    response = requests.post(
        f"{BASE_URL}/api/reference/SalesOrder",
        headers={
            "Authorization": f"Bearer {TOKEN}",
            "Content-Type": "application/json",
        },
        json=body,
    )

    if response.status_code == 201:
        return response.json()

    raise Exception(
        f"Erreur {response.status_code}: {response.text}"
    )

# Usage
result = create_sales_order(
    customer_id="550e8400-e29b-41d4-a716-446655440000",
    payment_term_id="660e8400-e29b-41d4-a716-446655440001",
    warehouse_id="770e8400-e29b-41d4-a716-446655440002",
    description="Bon créé via Python",
)

print(f"Bon créé : {result['reference']['ReferenceId']}")
```

### 4.15.3 Python — Démarrer et arrêter un chrono

```python
import requests
import time

BASE_URL = "{VOTRE_INSTANCE}"
TOKEN = "{VOTRE_TOKEN}"

headers = {"Authorization": f"Bearer {TOKEN}"}

# Démarrer le chrono
start = requests.get(
    f"{BASE_URL}/api/reference/SalesOrder/{ref_id}/timelogs/start",
    headers=headers,
)
timelog = start.json()["timeLog"]
timelog_id = timelog["TimeLogId"]
print(f"Chrono démarré : {timelog_id}")

# … travail en cours …
time.sleep(5)

# Arrêter le chrono
stop = requests.get(
    f"{BASE_URL}/api/reference/SalesOrder/{ref_id}/timelogs/{timelog_id}/stop",
    headers=headers,
)
stopped = stop.json()["timeLog"]
print(f"Chrono arrêté : {stopped['StartTime']} → {stopped['EndTime']}")
```

---

> **Résumé du chapitre :** toute écriture SEIGMA passe par `"Properties"` / `"Attributes"` avec des IDs en objets `{"ReferenceId":"…"}`. Les trois endpoints principaux couvrent les bons de travail, les activités et les poinçons. Les pièges majeurs sont les formats d'ID, le français dans les messages d'erreur, et le comportement PATCH-like du PUT qui vide les champs absents.
