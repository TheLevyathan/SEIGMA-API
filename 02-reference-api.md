# Chapitre 2 — Référence API SEIGMA

> **Version documentée** : v1.13.0 (basée sur le skill SEIGMA)
> **Dernière mise à jour** : 2026-06-20

---

## Table des matières

1. [Aperçu général](#aperçu-général)
2. [GET /api/reference/{ModelCode}/{ReferenceId}](#get-apireferencemodelcodereferenceid)
3. [POST /api/reference/{ModelCode}/search](#post-apireferencemodelcodesearch)
4. [ModelCodes connus](#modelcodes-connus)
5. [Types d'attributs](#types-dattributs)
6. [Pièges et limitations (Pitfalls)](#pièges-et-limitations-pitfalls)
7. [Endpoints cassés](#endpoints-cassés)
8. [Codes d'erreur](#codes-derreur)
9. [Exemples multi-langages](#exemples-multi-langages)

---

## Aperçu général

L'API Reference de SEIGMA expose deux endpoints principaux qui permettent de lire et rechercher des références (fiches) dans un modèle donné. Chaque modèle est identifié par un `ModelCode` (par ex. `SalesOrder`, `Customer`, `Quotation`).

| Méthode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/reference/{ModelCode}/{ReferenceId}` | Récupère une référence unique |
| `POST` | `/api/reference/{ModelCode}/search` | Recherche paginée multi-critères |

Toutes les réponses sont en JSON. L'authentification se fait via les mécanismes standards SEIGMA (cookie de session ou token selon la configuration de l'instance).

---

## GET /api/reference/{ModelCode}/{ReferenceId}

### Description

Retourne une référence unique formatée en JSON, avec **tous les attributs visibles** dans le `ReferenceEditor` de l'interface SEIGMA.

La réponse est **wrappée** dans un objet contenant la clé `reference`.

### Endpoint

```
GET /api/reference/{ModelCode}/{ReferenceId}
```

### Paramètres de route

| Paramètre | Type | Description |
|-----------|------|-------------|
| `ModelCode` | `string` | Code du modèle (ex. `SalesOrder`, `Customer`) |
| `ReferenceId` | `integer` | Identifiant unique de la référence |

### Exemple cURL

```bash
curl -X GET \
  "https://votre-instance.seigma.com/api/reference/SalesOrder/12345" \
  -H "Cookie: .ASPXAUTH=..." \
  -H "Accept: application/json"
```

### Réponse (succès — 200 OK)

```json
{
  "reference": {
    "ReferenceId": 12345,
    "ModelCode": "SalesOrder",
    "ModelName": "Bon de commande",
    "Color": "#3498db",
    "Display": "SO-2024-00123",
    "CustomerId": 5678,
    "CustomerDisplay": "ACME Corp",
    "TotalAmount": 15000.00,
    "Status": "Confirmé",
    "CreatedBy": {
      "UserId": 42,
      "Display": "Jean Dupont"
    },
    "AssignedTo": [
      {
        "UserId": 42,
        "Display": "Jean Dupont"
      },
      {
        "UserId": 99,
        "Display": "Marie Martin"
      }
    ],
    "Activities": [
      {
        "ActivityId": 1001,
        "Subject": "Appel de suivi",
        "DueDate": "2024-03-15T10:00:00"
      }
    ]
  }
}
```

### Structure de la réponse

| Champ | Type | Description |
|-------|------|-------------|
| `reference` | `object` | Objet contenant tous les attributs de la référence |

Les attributs à l'intérieur de `reference` varient selon le `ModelCode`. Voir la section [Types d'attributs](#types-dattributs) pour le détail des structures possibles.

### Codes de réponse

| Code HTTP | Signification |
|-----------|---------------|
| `200 OK` | Référence trouvée et retournée |
| `401 Unauthorized` | Authentification requise |
| `403 Forbidden` | Accès non autorisé à ce modèle |
| `404 Not Found` | Référence inexistante ou `ModelCode` invalide |
| `500 Internal Server Error` | Erreur serveur (modèle instable, voir [Endpoints cassés](#endpoints-cassés)) |

---

## POST /api/reference/{ModelCode}/search

### Description

Recherche paginée de références avec filtres combinables (AND, OR, IN) et tri personnalisable.

### Endpoint

```
POST /api/reference/{ModelCode}/search
```

### Headers

| Header | Valeur |
|--------|--------|
| `Content-Type` | `application/json` |
| `Accept` | `application/json` |

### Body canonique

```json
{
  "Offset": 0,
  "Limit": 50,
  "IsSelector": false,
  "ModelListId": null,
  "CurrentReferenceId": null,
  "WhereCondition": [],
  "WhereOrCondition": [],
  "WhereInCondition": [],
  "OrderByCondition": []
}
```

### Paramètres du body

| Paramètre | Type | Requis | Description |
|-----------|------|--------|-------------|
| `Offset` | `integer` | Oui | Décalage pour pagination (0-based) |
| `Limit` | `integer` | Oui | Nombre maximum de résultats (max conseillé : 200) |
| `IsSelector` | `boolean` | Oui | Contexte sélecteur (`false` pour recherche standard) |
| `ModelListId` | `integer` ou `null` | Oui | Filtre par liste de modèles (`null` = aucune) |
| `CurrentReferenceId` | `integer` ou `null` | Oui | Référence courante pour contexte (`null` = aucune) |
| `WhereCondition` | `array` | Oui | Conditions AND (voir ci-dessous) |
| `WhereOrCondition` | `array` | Oui | Conditions OR (voir ci-dessous) |
| `WhereInCondition` | `array` | Oui | Conditions IN pour attributs de type liste |
| `OrderByCondition` | `array` | Oui | Tri des résultats |

---

### WhereCondition — Filtres AND

Tableau de conditions combinées en **ET logique**. Chaque condition est un objet avec trois champs :

```json
{
  "ModelAttributeCode": "CustomerDisplay",
  "Operator": "=",
  "Value": "ACME Corp"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `ModelAttributeCode` | `string` | Code de l'attribut sur lequel filtrer |
| `Operator` | `string` | Opérateur de comparaison (voir tableau ci-dessous) |
| `Value` | `string` | Valeur à comparer |

#### Opérateurs supportés

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `=` | Égalité exacte | `"Value": "ACME Corp"` |
| `!=` | Différent de | `"Value": "Brouillon"` |
| `>` | Supérieur à | `"Value": "1000"` |
| `<` | Inférieur à | `"Value": "500"` |
| `>=` | Supérieur ou égal | `"Value": "2024-01-01"` |
| `<=` | Inférieur ou égal | `"Value": "2024-12-31"` |
| `Contains` | Contient (sous-chaîne) | `"Value": "ACME"` |
| `StartsWith` | Commence par | `"Value": "SO-"` |
| `EndsWith` | Termine par | `"Value": "-001"` |

> ⚠️ **Attention** : Certains modèles utilisent `OperatorCode` au lieu de `Operator`. Essayez `Operator` d'abord ; si l'API renvoie une erreur silencieuse, basculez sur `OperatorCode`. Le skill v1.13.0 recommande de toujours essayer `Operator` en premier.

### WhereOrCondition — Filtres OR

Même structure que `WhereCondition`, mais les conditions sont combinées en **OU logique**.

Cas d'usage typique : rechercher des commandes qui sont soit « En attente », soit « Confirmées ».

```json
"WhereOrCondition": [
  {
    "ModelAttributeCode": "Status",
    "Operator": "=",
    "Value": "En attente"
  },
  {
    "ModelAttributeCode": "Status",
    "Operator": "=",
    "Value": "Confirmé"
  }
]
```

### WhereInCondition — Filtres IN

Pour les attributs de **type liste** (MultiSelect, MultiUser, etc.). Permet de filtrer sur l'appartenance à un ensemble de valeurs.

```json
"WhereInCondition": [
  {
    "ModelAttributeCode": "AssignedTo",
    "Values": ["42", "99"]
  }
]
```

| Champ | Type | Description |
|-------|------|-------------|
| `ModelAttributeCode` | `string` | Code de l'attribut de type liste |
| `Values` | `array` de `string` | Valeurs à inclure |

### OrderByCondition — Tri

```json
"OrderByCondition": [
  {
    "ModelAttributeCode": "CreatedDate",
    "IsDescending": true
  }
]
```

| Champ | Type | Description |
|-------|------|-------------|
| `ModelAttributeCode` | `string` | Code de l'attribut de tri |
| `IsDescending` | `boolean` | `true` = décroissant, `false` = croissant |

### Réponse (succès — 200 OK)

```json
{
  "metadata": {
    "count": 50,
    "totalCount": 1932,
    "offset": 0,
    "limit": 50
  },
  "references": [
    {
      "ReferenceId": 12345,
      "ModelCode": "SalesOrder",
      "Display": "SO-2024-00123"
    }
  ]
}
```

### Structure de la réponse

| Champ | Type | Description |
|-------|------|-------------|
| `metadata` | `object` | Métadonnées de pagination |
| `metadata.count` | `integer` | Nombre de résultats dans cette page |
| `metadata.totalCount` | `integer` | Nombre total de résultats correspondants |
| `metadata.offset` | `integer` | Offset utilisé (miroir de la requête) |
| `metadata.limit` | `integer` | Limit utilisée (miroir de la requête) |
| `references` | `array` | Tableau des références trouvées |

> **Note** : Les attributs retournés dans `references[]` sont un **sous-ensemble** des attributs du modèle. Pour obtenir tous les champs, utilisez le GET par `ReferenceId`.

### Codes de réponse

| Code HTTP | Signification |
|-----------|---------------|
| `200 OK` | Recherche exécutée, résultats retournés |
| `400 Bad Request` | Body JSON invalide ou champ inconnu |
| `401 Unauthorized` | Authentification requise |
| `403 Forbidden` | Accès non autorisé à ce modèle |
| `500 Internal Server Error` | Erreur serveur |

---

## ModelCodes connus

Voici les `ModelCode` documentés et testés avec le skill v1.13.0. Les statistiques sont indicatives et dépendent de l'instance SEIGMA cible.

### SalesOrder — Bons de commande

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `SalesOrder` |
| **Références** | ~1932 |
| **Champs en search** | 23 |
| **Champs en détail** | 87 |
| **Statut** | ✅ Stable |

**Attributs notables** : `CustomerId`, `CustomerDisplay`, `TotalAmount`, `Status`, `CreatedDate`, `AssignedTo`

```bash
# Exemple : rechercher les 10 derniers bons de commande
curl -X POST \
  "https://votre-instance.seigma.com/api/reference/SalesOrder/search" \
  -H "Content-Type: application/json" \
  -H "Cookie: .ASPXAUTH=..." \
  -d '{
    "Offset": 0,
    "Limit": 10,
    "IsSelector": false,
    "ModelListId": null,
    "CurrentReferenceId": null,
    "WhereCondition": [],
    "WhereOrCondition": [],
    "WhereInCondition": [],
    "OrderByCondition": [
      {"ModelAttributeCode": "CreatedDate", "IsDescending": true}
    ]
  }'
```

### Customer — Clients

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Customer` |
| **Références** | ~3839 |
| **Statut** | ✅ Stable |

**Attributs notables** : `Display`, `Email`, `Phone`, `Address`, `CustomerCode`

### Quotation — Soumissions/Devis

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Quotation` |
| **Références** | ~1935 |
| **Statut** | ✅ Stable |

**Attributs notables** : `CustomerId`, `TotalAmount`, `Status`, `ValidUntil`

### Receipt — Reçus

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Receipt` |
| **Références** | ~925 |
| **Statut** | ⚠️ Partiellement stable |

> ⚠️ **PITFALL** : `SalesOrderId` est **TRANSITIF** sur `Receipt`. Cela signifie qu'il n'est **jamais résolu** lors d'un search — vous ne recevrez pas l'objet SalesOrder lié, seulement l'ID brut. Pour obtenir le détail du bon de commande associé, faites un GET supplémentaire sur `/api/reference/SalesOrder/{SalesOrderId}`.

### SalesInvoice — Factures de vente

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `SalesInvoice` |
| **Statut** | ⚠️ Partiellement stable |

> ⚠️ **PITFALL** : Les **SelectAttributes** sont **IGNORÉS** sur le search. Vous ne pouvez pas demander des champs spécifiques en recherche ; tous les résultats renvoient un sous-ensemble fixe d'attributs. Utilisez le GET par `ReferenceId` pour obtenir les champs complets.

### Lead — Prospects

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Lead` |
| **Références** | ~589 |
| **Statut** | ✅ Stable |

**Attributs notables** : `Display`, `Company`, `Email`, `Status`, `Source`

### Call — Appels / Billets

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Call` |
| **Références** | ~95 |
| **Format ID** | `TI-XXXXX` |
| **Statut** | ✅ Stable |

### Product — Produits

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Product` |
| **Références** | ~21 |
| **Statut** | ✅ Stable |

### PaymentTerm — Conditions de paiement

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `PaymentTerm` |
| **Références** | ~4 |
| **Statut** | ✅ Stable |

### Warehouse — Entrepôt

| Propriété | Valeur |
|-----------|--------|
| **ModelCode** | `Warehouse` |
| **Références** | ~1 |
| **Statut** | ✅ Stable |

---

## Types d'attributs

Les attributs retournés dans `reference` peuvent prendre différentes structures selon leur type métier.

### ModelAttributeList

Liste simple d'attributs scalaires. Présents dans toutes les références.

```json
{
  "ReferenceId": 12345,
  "ModelCode": "SalesOrder",
  "ModelName": "Bon de commande",
  "Color": "#3498db",
  "Display": "SO-2024-00123"
}
```

| Sous-attribut | Type | Description |
|---------------|------|-------------|
| `Display` | `string` | Libellé affiché dans l'interface |
| `ReferenceId` | `integer` | Identifiant unique |
| `ModelCode` | `string` | Code du modèle parent |
| `ModelName` | `string` | Nom lisible du modèle |
| `Color` | `string` | Couleur hexadécimale associée |

### UserAssignableModel

Attribut mono-utilisateur (ex. `CreatedBy`, `ResponsibleUser`).

```json
{
  "CreatedBy": {
    "UserId": 42,
    "Display": "Jean Dupont"
  }
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `UserId` | `integer` | Identifiant de l'utilisateur |
| `Display` | `string` | Nom affiché de l'utilisateur |

### MultiUserAssignableModel

Attribut multi-utilisateurs (ex. `AssignedTo`, `Watchers`). Retourné comme un tableau.

```json
{
  "AssignedTo": [
    { "UserId": 42, "Display": "Jean Dupont" },
    { "UserId": 99, "Display": "Marie Martin" }
  ]
}
```

### ActivityAttributeList

Liste des activités liées (ex. appels, tâches, rendez-vous).

```json
{
  "Activities": [
    {
      "ActivityId": 1001,
      "Subject": "Appel de suivi",
      "DueDate": "2024-03-15T10:00:00",
      "Status": "Planifié",
      "AssignedTo": { "UserId": 42, "Display": "Jean Dupont" }
    }
  ]
}
```

### MultiSelectAttributeList

Attribut à choix multiples (ex. tags, catégories).

```json
{
  "Tags": [
    { "Value": "Urgent" },
    { "Value": "International" }
  ]
}
```

### ModelModelAttributeList

Attribut de type référence vers un autre modèle (lookup/relation). Ex. `CustomerId` sur un `SalesOrder`.

```json
{
  "Customer": {
    "ReferenceId": 5678,
    "ModelCode": "Customer",
    "Display": "ACME Corp"
  }
}
```

> ⚠️ **Attention** : Certaines relations sont **transitives** et ne sont pas résolues dans le search (ex. `SalesOrderId` sur `Receipt`). Voir la section [Pièges](#pièges-et-limitations-pitfalls).

---

## Pièges et limitations (Pitfalls)

### 1. SelectAttributes ignorés sur Receipt et SalesInvoice

Sur `SalesOrder`, `Quotation` et `Call/search`, il est possible d'utiliser `SelectAttributes` pour limiter les champs retournés. **Ce mécanisme est ignoré silencieusement** sur `Receipt` et `SalesInvoice`.

- ✅ **Fonctionne** : `SalesOrder/search`, `Quotation/search`, `Call/search`
- ❌ **Ignoré** : `Receipt/search`, `SalesInvoice/search`

### 2. Operator vs OperatorCode

Le champ de condition peut s'appeler `Operator` ou `OperatorCode` selon le modèle. La convention est flottante :

- Essayez **toujours** `Operator` en premier
- Si la recherche ne produit pas de résultats ou renvoie une erreur silencieuse, basculez sur `OperatorCode`

### 3. SalesOrderId transitif sur Receipt

L'attribut `SalesOrderId` sur `Receipt` est **transitif** : il n'est jamais résolu en un objet `SalesOrder` lors du search. Vous recevez uniquement l'ID brut.

**Workaround** :

```typescript
// Étape 1 : chercher les reçus
const receipts = await searchReferences('Receipt', {
  WhereCondition: [{ ModelAttributeCode: 'TotalAmount', Operator: '>', Value: '1000' }]
});

// Étape 2 : résoudre manuellement chaque SalesOrderId
for (const receipt of receipts.references) {
  if (receipt.SalesOrderId) {
    const order = await getReference('SalesOrder', receipt.SalesOrderId);
    receipt._salesOrder = order.reference;
  }
}
```

### 4. Pagination

- `Offset` est **0-based**
- Le `Limit` maximum conseillé est **200**. Au-delà, l'API peut ralentir significativement.
- `totalCount` dans la réponse reflète le nombre total de résultats correspondant aux filtres, pas le nombre total de références du modèle.

### 5. Différence search vs GET

Les résultats de `search` contiennent un **sous-ensemble** des attributs. Pour obtenir tous les champs (87 sur SalesOrder par exemple), vous devez faire un GET par `ReferenceId` pour chaque résultat.

---

## Endpoints cassés

Les endpoints suivants sont connus pour retourner une erreur **HTTP 500** systématique. Ils sont documentés ici pour éviter des tentatives infructueuses.

| Endpoint | Statut | Symptôme |
|----------|--------|----------|
| `User/search` | 🔴 500 | Erreur interne serveur |
| `Employee/search` | 🔴 500 | Erreur interne serveur |
| `Invoice/search` | 🔴 500 | Erreur interne serveur |
| `Activity/search` | 🔴 500 | Erreur interne serveur |
| `SalesOrderLine/search` | 🔴 500 | Erreur interne serveur |

> **Note** : Ces endpoints peuvent fonctionner sur certaines versions ou configurations de SEIGMA. Testez toujours avec un appel minimal avant d'intégrer.

---

## Codes d'erreur

### Erreurs HTTP standard

| Code | Nom | Cause fréquente |
|------|-----|-----------------|
| `400` | Bad Request | Body JSON malformé, champ inexistant |
| `401` | Unauthorized | Cookie de session expiré ou manquant |
| `403` | Forbidden | L'utilisateur n'a pas accès au modèle |
| `404` | Not Found | `ModelCode` ou `ReferenceId` inexistant |
| `500` | Internal Server Error | Modèle instable ou endpoint cassé |

### Erreurs applicatives SEIGMA

Certaines erreurs 500 peuvent retourner un corps JSON avec des détails :

```json
{
  "Message": "Une erreur est survenue lors du chargement du modèle.",
  "ExceptionMessage": "Object reference not set to an instance of an object.",
  "ExceptionType": "System.NullReferenceException"
}
```

---

## Exemples multi-langages

### TypeScript / Node.js

```typescript
// Types
interface SearchRequest {
  Offset: number;
  Limit: number;
  IsSelector: boolean;
  ModelListId: number | null;
  CurrentReferenceId: number | null;
  WhereCondition: Condition[];
  WhereOrCondition: Condition[];
  WhereInCondition: InCondition[];
  OrderByCondition: OrderBy[];
}

interface Condition {
  ModelAttributeCode: string;
  Operator: string;
  Value: string;
}

interface InCondition {
  ModelAttributeCode: string;
  Values: string[];
}

interface OrderBy {
  ModelAttributeCode: string;
  IsDescending: boolean;
}

interface SearchResponse {
  metadata: {
    count: number;
    totalCount: number;
    offset: number;
    limit: number;
  };
  references: Record<string, unknown>[];
}

interface ReferenceResponse {
  reference: Record<string, unknown>;
}

// Client SEIGMA
class SeigmaClient {
  constructor(
    private baseUrl: string,
    private cookie: string
  ) {}

  private async request<T>(method: string, path: string, body?: unknown): Promise<T> {
    const response = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Cookie': `.ASPXAUTH=${this.cookie}`,
      },
      body: body ? JSON.stringify(body) : undefined,
    });

    if (!response.ok) {
      throw new Error(`SEIGMA API error: ${response.status} ${response.statusText}`);
    }

    return response.json();
  }

  /** Récupère une référence par son ID */
  async getReference(modelCode: string, referenceId: number): Promise<ReferenceResponse> {
    return this.request<ReferenceResponse>(
      'GET',
      `/api/reference/${modelCode}/${referenceId}`
    );
  }

  /** Recherche paginée de références */
  async searchReferences(
    modelCode: string,
    params: Partial<SearchRequest> = {}
  ): Promise<SearchResponse> {
    const body: SearchRequest = {
      Offset: 0,
      Limit: 50,
      IsSelector: false,
      ModelListId: null,
      CurrentReferenceId: null,
      WhereCondition: [],
      WhereOrCondition: [],
      WhereInCondition: [],
      OrderByCondition: [],
      ...params,
    };

    return this.request<SearchResponse>(
      'POST',
      `/api/reference/${modelCode}/search`,
      body
    );
  }

  /** Récupère toutes les pages d'une recherche (pagination automatique) */
  async searchAllReferences(
    modelCode: string,
    params: Partial<SearchRequest> = {},
    pageSize: number = 200
  ): Promise<Record<string, unknown>[]> {
    const allRefs: Record<string, unknown>[] = [];
    let offset = 0;

    while (true) {
      const result = await this.searchReferences(modelCode, {
        ...params,
        Offset: offset,
        Limit: pageSize,
      });

      allRefs.push(...result.references);

      if (allRefs.length >= result.metadata.totalCount) break;
      offset += pageSize;
    }

    return allRefs;
  }
}

// Utilisation
const client = new SeigmaClient('https://votre-instance.seigma.com', 'votre-cookie');

// Recherche simple
const orders = await client.searchReferences('SalesOrder', {
  WhereCondition: [
    { ModelAttributeCode: 'Status', Operator: '=', Value: 'Confirmé' }
  ],
  OrderByCondition: [
    { ModelAttributeCode: 'CreatedDate', IsDescending: true }
  ],
  Limit: 10,
});

console.log(`${orders.metadata.totalCount} commandes trouvées`);

// Détail d'une référence
const detail = await client.getReference('SalesOrder', 12345);
console.log(detail.reference.Display);
```

### Python

```python
import requests
from typing import Optional, List, Dict, Any

class SeigmaClient:
    def __init__(self, base_url: str, cookie: str):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        self.session.cookies.set('.ASPXAUTH', cookie)
        self.session.headers.update({
            'Content-Type': 'application/json',
            'Accept': 'application/json',
        })

    def get_reference(self, model_code: str, reference_id: int) -> Dict[str, Any]:
        """Récupère une référence par son ID."""
        response = self.session.get(
            f'{self.base_url}/api/reference/{model_code}/{reference_id}'
        )
        response.raise_for_status()
        return response.json()

    def search_references(
        self,
        model_code: str,
        offset: int = 0,
        limit: int = 50,
        where: Optional[List[Dict]] = None,
        where_or: Optional[List[Dict]] = None,
        where_in: Optional[List[Dict]] = None,
        order_by: Optional[List[Dict]] = None,
    ) -> Dict[str, Any]:
        """Recherche paginée de références."""
        body = {
            "Offset": offset,
            "Limit": limit,
            "IsSelector": False,
            "ModelListId": None,
            "CurrentReferenceId": None,
            "WhereCondition": where or [],
            "WhereOrCondition": where_or or [],
            "WhereInCondition": where_in or [],
            "OrderByCondition": order_by or [],
        }

        response = self.session.post(
            f'{self.base_url}/api/reference/{model_code}/search',
            json=body
        )
        response.raise_for_status()
        return response.json()

    def search_all(
        self,
        model_code: str,
        page_size: int = 200,
        **kwargs
    ) -> List[Dict[str, Any]]:
        """Récupère toutes les pages d'une recherche."""
        all_refs = []
        offset = 0

        while True:
            result = self.search_references(
                model_code, offset=offset, limit=page_size, **kwargs
            )
            all_refs.extend(result['references'])

            if len(all_refs) >= result['metadata']['totalCount']:
                break
            offset += page_size

        return all_refs

# Utilisation
client = SeigmaClient('https://votre-instance.seigma.com', 'votre-cookie')

# Recherche simple
results = client.search_references(
    'SalesOrder',
    where=[
        {'ModelAttributeCode': 'Status', 'Operator': '=', 'Value': 'Confirmé'}
    ],
    order_by=[
        {'ModelAttributeCode': 'CreatedDate', 'IsDescending': True}
    ],
    limit=10
)

print(f"{results['metadata']['totalCount']} commandes trouvées")

# Détail
detail = client.get_reference('SalesOrder', 12345)
print(detail['reference']['Display'])
```

### cURL — Exemples complets

#### Récupérer une référence

```bash
curl -X GET \
  "https://votre-instance.seigma.com/api/reference/SalesOrder/12345" \
  -H "Cookie: .ASPXAUTH=VOTRE_COOKIE" \
  -H "Accept: application/json" | jq '.reference.Display'
```

#### Recherche simple (AND)

```bash
curl -X POST \
  "https://votre-instance.seigma.com/api/reference/SalesOrder/search" \
  -H "Content-Type: application/json" \
  -H "Cookie: .ASPXAUTH=VOTRE_COOKIE" \
  -d '{
    "Offset": 0,
    "Limit": 10,
    "IsSelector": false,
    "ModelListId": null,
    "CurrentReferenceId": null,
    "WhereCondition": [
      {"ModelAttributeCode": "Status", "Operator": "=", "Value": "Confirmé"},
      {"ModelAttributeCode": "TotalAmount", "Operator": ">", "Value": "10000"}
    ],
    "WhereOrCondition": [],
    "WhereInCondition": [],
    "OrderByCondition": [
      {"ModelAttributeCode": "CreatedDate", "IsDescending": true}
    ]
  }' | jq '.metadata.totalCount'
```

#### Recherche avec OR

```bash
curl -X POST \
  "https://votre-instance.seigma.com/api/reference/Customer/search" \
  -H "Content-Type: application/json" \
  -H "Cookie: .ASPXAUTH=VOTRE_COOKIE" \
  -d '{
    "Offset": 0,
    "Limit": 50,
    "IsSelector": false,
    "ModelListId": null,
    "CurrentReferenceId": null,
    "WhereCondition": [],
    "WhereOrCondition": [
      {"ModelAttributeCode": "Display", "Operator": "Contains", "Value": "ACME"},
      {"ModelAttributeCode": "Display", "Operator": "Contains", "Value": "Global"}
    ],
    "WhereInCondition": [],
    "OrderByCondition": []
  }' | jq '.references[] | .Display'
```

#### Recherche avec Contains (sous-chaîne)

```bash
curl -X POST \
  "https://votre-instance.seigma.com/api/reference/Call/search" \
  -H "Content-Type: application/json" \
  -H "Cookie: .ASPXAUTH=VOTRE_COOKIE" \
  -d '{
    "Offset": 0,
    "Limit": 5,
    "IsSelector": false,
    "ModelListId": null,
    "CurrentReferenceId": null,
    "WhereCondition": [
      {"ModelAttributeCode": "Display", "Operator": "StartsWith", "Value": "TI-"}
    ],
    "WhereOrCondition": [],
    "WhereInCondition": [],
    "OrderByCondition": []
  }' | jq '.references[] | {id: .ReferenceId, display: .Display}'
```

---

## Résumé des bonnes pratiques

1. **Toujours paginer** : utilisez `Offset`/`Limit`, ne partez pas du principe que `totalCount` tiendra dans une seule page.
2. **Essayez `Operator` d'abord**, puis `OperatorCode` en fallback.
3. **Ne vous fiez pas à `SelectAttributes` sur `Receipt` et `SalesInvoice`** — ils sont ignorés.
4. **Résolvez manuellement les relations transitives** (`SalesOrderId` sur `Receipt`).
5. **Utilisez GET par `ReferenceId` pour obtenir tous les champs** après un search.
6. **Testez les endpoints avant intégration** — certains modèles peuvent retourner 500 selon la version de SEIGMA.
7. **Limitez le `Limit` à 200 maximum** par requête pour éviter les timeouts.

---

> **Prochain chapitre** : [03 — Authentification et sécurité](03-authentification.md)
>
> **Chapitre précédent** : [01 — Introduction à l'API SEIGMA](01-introduction.md)
