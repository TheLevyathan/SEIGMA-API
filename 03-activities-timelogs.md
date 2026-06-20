# Chapitre 3 — Activity API & Timelog API

> **API SEIGMA** — Gestion des activités (rendez-vous, tâches planifiées) et des poinçons (suivi du temps).

---

## Table des matières

- [3.1 Activity API](#31-activity-api)
  - [3.1.1 Aperçu général](#311-aperçu-général)
  - [3.1.2 GET /api/activity/{activity_id} — Récupérer une activité](#312-get-apiactivityactivity_id--récupérer-une-activité)
  - [3.1.3 POST /api/activity/getactivitiesfordate — Activités d'une date](#313-post-apiactivitygetactivitiesfordate--activités-dune-date)
  - [3.1.4 POST /api/activity/{referenceId} — Créer une activité](#314-post-apiactivityreferenceid--créer-une-activité)
  - [3.1.5 PUT /api/activity/{activityId} — Modifier une activité](#315-put-apiactivityactivityid--modifier-une-activité)
  - [3.1.6 Pièges et erreurs fréquentes — Activity API](#316-pièges-et-erreurs-fréquentes--activity-api)
- [3.2 Timelog API](#32-timelog-api)
  - [3.2.1 Aperçu général](#321-aperçu-général)
  - [3.2.2 GET …/timelogs — Lister les poinçons](#322-get-timelogs--lister-les-poinçons)
  - [3.2.3 POST …/timelogs/add — Ajouter un poinçon](#323-post-timelogsadd--ajouter-un-poinçon)
  - [3.2.4 PUT …/timelogs — Modifier un poinçon](#324-put-timelogs--modifier-un-poinçon)
  - [3.2.5 DELETE …/timelogs/{timelogId} — Supprimer un poinçon](#325-delete-timelogstimelogid--supprimer-un-poinçon)
  - [3.2.6 GET …/timelogs/start — Démarrer un poinçon](#326-get-timelogsstart--démarrer-un-poinçon)
  - [3.2.7 GET …/timelogs/{timelogId}/stop — Arrêter un poinçon](#327-get-timelogstimelogidstop--arrêter-un-poinçon)
  - [3.2.8 Pièges et erreurs fréquentes — Timelog API](#328-pièges-et-erreurs-fréquentes--timelog-api)
- [3.3 Résumé des codes d'erreur](#33-résumé-des-codes-derreur)

---

## 3.1 Activity API

### 3.1.1 Aperçu général

L'API Activity de SEIGMA permet de gérer les **activités** (rendez-vous, tâches, interventions planifiées) liées à des entités du système (clients, contrats, équipements, etc.). Les activités sont toujours rattachées à une **référence** (une entité métier) et ne peuvent pas exister de manière autonome.

| Opération | Méthode | Endpoint | Description |
|-----------|---------|----------|-------------|
| Récupérer une activité | `GET` | `/api/activity/{activity_id}` | Obtient une activité par son ID |
| Activités d'une date | `POST` | `/api/activity/getactivitiesfordate` | **Endpoint critique** pour le planning quotidien |
| Créer une activité | `POST` | `/api/activity/{referenceId}` | Crée une activité liée à une référence |
| Modifier une activité | `PUT` | `/api/activity/{activityId}` | Met à jour une activité existante |

> ⚠️ **Note importante sur les fuseaux horaires :** Les dates sont stockées en **UTC** dans la base de données, mais l'API les affiche dans le **fuseau horaire de l'utilisateur connecté**. Ce comportement est transparent pour la plupart des opérations, mais doit être pris en compte lors de comparaisons entre les données API et la base de données.

---

### 3.1.2 GET /api/activity/{activity_id} — Récupérer une activité

Récupère une activité spécifique par son identifiant unique.

**Requête :**

```bash
curl -X GET \
  "https://{VOTRE_INSTANCE}.seigma.app/api/activity/a1b2c3d4-e5f6-7890-abcd-ef1234567890" \
  -H "Authorization: Bearer {token}"
```

**Réponse (200 OK) :**

La réponse est **wrappée** dans un objet contenant la clé `activity` :

```json
{
  "activity": {
    "ActivityId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "StartDate": "2026-05-26T09:00:00",
    "EndDate": "2026-05-26T11:00:00",
    "Subject": "Maintenance préventive - Chaudière Gaz",
    "CustomerDisplay": "Résidence Les Oliviers",
    "ActivityTypeId": {
      "Display": "Intervention",
      "ReferenceId": "type-guid-here",
      "Color": "#FF5733"
    },
    "AssignedToId": {
      "UserId": "user-guid-here",
      "Display": "Dupont Jean"
    },
    "Location": "123 Avenue des Fleurs, 75001 Paris",
    "ResourceIds": ["resource-guid-1", "resource-guid-2"],
    "Notes": "Apporter clé multifonction et détecteur CO2",
    "IsCompleted": false,
    "AllDay": false
  }
}
```

**Champs principaux de l'objet Activity :**

| Champ | Type | Description |
|-------|------|-------------|
| `ActivityId` | `GUID` | Identifiant unique de l'activité |
| `StartDate` | `DateTime` | Date/heure de début (fuseau utilisateur) |
| `EndDate` | `DateTime` | Date/heure de fin (fuseau utilisateur) |
| `Subject` | `string` | Objet / titre de l'activité |
| `CustomerDisplay` | `string` | Nom d'affichage du client lié |
| `ActivityTypeId` | `ModelAttributeList` | Type d'activité avec sa couleur |
| `AssignedToId` | `UserAssignableModel` | Utilisateur assigné (`{UserId, Display}`) |
| `Location` | `string` | Adresse ou lieu de l'intervention |
| `ResourceIds` | `GUID[]` | Ressources associées |
| `Notes` | `string` | Notes internes |
| `IsCompleted` | `boolean` | Statut d'achèvement |
| `AllDay` | `boolean` | Indique si l'activité dure toute la journée |

**Implémentations :**

```typescript
// TypeScript
interface ActivityTypeRef {
  Display: string;
  ReferenceId: string;
  Color: string; // ex: "#FF5733"
}

interface ActivityAssignee {
  UserId: string;
  Display: string;
}

interface Activity {
  ActivityId: string;
  StartDate: string;   // ISO 8601
  EndDate: string;     // ISO 8601
  Subject: string;
  CustomerDisplay: string;
  ActivityTypeId: ActivityTypeRef;
  AssignedToId: ActivityAssignee;
  Location: string;
  ResourceIds: string[];
  Notes: string;
  IsCompleted: boolean;
  AllDay: boolean;
}

async function getActivity(
  activityId: string,
  token: string
): Promise<Activity> {
  const res = await fetch(
    `https://{VOTRE_INSTANCE}.seigma.app/api/activity/${activityId}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const data = await res.json();
  return data.activity; // la réponse est wrappée
}
```

```python
# Python
import requests

def get_activity(activity_id: str, token: str) -> dict:
    url = f"https://{VOTRE_INSTANCE}.seigma.app/api/activity/{activity_id}"
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(url, headers=headers)
    resp.raise_for_status()
    return resp.json()["activity"]  # la réponse est wrappée
```

**Codes d'erreur :**

| Code | Signification |
|------|---------------|
| `200` | Succès — activité retournée |
| `401` | Non authentifié |
| `404` | Activité introuvable |

---

### 3.1.3 POST /api/activity/getactivitiesfordate — Activités d'une date

> 🔥 **Endpoint critique pour le planning quotidien.** Récupère toutes les activités d'un utilisateur pour une date donnée.

Cet endpoint est le pilier du module de planification SEIGMA. Il retourne l'ensemble des activités planifiées pour un utilisateur à une date spécifique, couvrant toutes ses équipes et références.

> 🚧 **À compléter** : Les `UserId` des équipes sont spécifiques à votre instance SEIGMA. Vous devez obtenir la liste de vos équipes et leurs UserId auprès de votre administrateur SEIGMA ou via l'interface d'administration.

**Requête :**

```bash
curl -X POST \
  "https://{VOTRE_INSTANCE}.seigma.app/api/activity/getactivitiesfordate" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "UserId": {
      "UserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "Display": "Dupont Jean"
    },
    "DateStart": "2026-05-26T00:00:00"
  }'
```

**Corps de la requête :**

| Champ | Type | Obligatoire | Description |
|-------|------|:-----------:|-------------|
| `UserId` | `UserAssignableModel` | ✅ | Objet utilisateur : `{UserId: guid, Display: string}` |
| `DateStart` | `string` (ISO 8601 complet) | ✅ | Date cible au format ISO avec timestamp |

> 🚨 **PIÈGE CRITIQUE — `DateStart` :** Le champ exige un **format ISO 8601 COMPLET avec horodatage**, par exemple `"2026-05-26T00:00:00"`. Si vous envoyez uniquement la date (`"2026-05-26"`), l'API retourne une **erreur 500**.

> 🚨 **PIÈGE CRITIQUE — `UserId` :** `UserId` est un **objet `UserAssignableModel`** contenant `{UserId, Display}`, **PAS une simple chaîne GUID**. Envoyer une string simple (`"UserId": "guid"`) provoque une **erreur 500**.

**Réponse (200 OK) :**

```json
{
  "metadata": {
    "TotalCount": 39,
    "DateStart": "2026-05-26T00:00:00",
    "UserId": {
      "UserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "Display": "Dupont Jean"
    }
  },
  "activities": [
    {
      "ActivityId": "act-001",
      "StartDate": "2026-05-26T08:00:00",
      "EndDate": "2026-05-26T09:30:00",
      "Subject": "Entretien chaudière - Bâtiment A",
      "CustomerDisplay": "Résidence Les Oliviers",
      "ActivityTypeId": {
        "Display": "Maintenance",
        "ReferenceId": "type-maintenance",
        "Color": "#3498DB"
      },
      "AssignedToId": {
        "UserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "Display": "Dupont Jean"
      },
      "Location": "12 rue de la Paix, 75002 Paris",
      "ResourceIds": ["res-vehicule-1"],
      "Notes": "Prévoir pièces détachées réf. CH-2024",
      "IsCompleted": false,
      "AllDay": false
    }
    // ... 38 autres activités
  ]
}
```

**Structure de la réponse :**

| Champ | Type | Description |
|-------|------|-------------|
| `metadata.TotalCount` | `int` | Nombre total d'activités retournées |
| `metadata.DateStart` | `string` | Date demandée (écho) |
| `metadata.UserId` | `UserAssignableModel` | Utilisateur concerné (écho) |
| `activities[]` | `Activity[]` | Tableau des activités de la journée |

> ℹ️ Dans une configuration typique, cet endpoint peut retourner les activités planifiées pour la journée. Le nombre d'activités varie selon votre volume d'opérations.

**Implémentations :**

```typescript
// TypeScript
interface ActivitiesForDateRequest {
  UserId: {
    UserId: string;  // GUID — PAS une string simple seule !
    Display: string;
  };
  DateStart: string; // ISO 8601 COMPLET avec timestamp — PAS juste "YYYY-MM-DD" !
}

interface ActivitiesForDateResponse {
  metadata: {
    TotalCount: number;
    DateStart: string;
    UserId: { UserId: string; Display: string };
  };
  activities: Activity[];
}

async function getActivitiesForDate(
  userId: string,
  displayName: string,
  date: string, // ex: "2026-05-26T00:00:00"
  token: string
): Promise<ActivitiesForDateResponse> {
  const res = await fetch(
    "https://{VOTRE_INSTANCE}.seigma.app/api/activity/getactivitiesfordate",
    {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${token}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        UserId: { UserId: userId, Display: displayName },
        DateStart: date  // FORMAT COMPLET OBLIGATOIRE
      })
    }
  );
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

```python
# Python
import requests
from datetime import datetime

def get_activities_for_date(
    user_id: str,
    display_name: str,
    date: str,  # ex: "2026-05-26T00:00:00"
    token: str
) -> dict:
    """
    ⚠️ date doit être au format ISO 8601 COMPLET avec timestamp.
    Exemple valide   : "2026-05-26T00:00:00"
    Exemple invalide : "2026-05-26" → 500
    """
    url = "https://{VOTRE_INSTANCE}.seigma.app/api/activity/getactivitiesfordate"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    payload = {
        "UserId": {
            "UserId": user_id,    # Objet, PAS une string simple
            "Display": display_name
        },
        "DateStart": date          # ISO complet obligatoire
    }
    resp = requests.post(url, headers=headers, json=payload)
    resp.raise_for_status()
    return resp.json()
```

**Codes d'erreur :**

| Code | Cause probable |
|------|----------------|
| `200` | Succès |
| `400` | Corps de requête malformé |
| `401` | Token invalide ou expiré |
| `500` | `DateStart` au mauvais format (date seule au lieu d'ISO complet) **ou** `UserId` passé en string simple au lieu d'objet |

---

### 3.1.4 POST /api/activity/{referenceId} — Créer une activité

Crée une activité liée à une référence. Le corps est wrappé dans `{"activity": {...}}`. Champs obligatoires : `ActivityTypeId` et `ActivityFollowUpId` (format `{Display, ReferenceId}`).

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets, les corps de requête, les implémentations TypeScript/Python et les codes d'erreur.

---

### 3.1.5 PUT /api/activity/{activityId} — Modifier une activité

Met à jour une activité existante. Corps partiel : seuls les champs fournis sont modifiés. Wrappé dans `{"activity": {...}}`.

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets, les corps de requête, les implémentations TypeScript/Python et les codes d'erreur.

---

### 3.1.6 Pièges et erreurs fréquentes — Activity API

#### 🚨 PIÈGE N°1 : Format de `DateStart` dans `getactivitiesfordate`

| Ce que vous envoyez | Résultat |
|---------------------|----------|
| `"2026-05-26T00:00:00"` ✅ | 200 OK — 39 activités retournées |
| `"2026-05-26"` ❌ | **500 Internal Server Error** |

**Règle :** Toujours utiliser le format ISO 8601 **complet avec horodatage**.

```python
# ✅ CORRECT
payload = {"DateStart": "2026-05-26T00:00:00"}

# ❌ INCORRECT — provoque une erreur 500
payload = {"DateStart": "2026-05-26"}
```

---

#### 🚨 PIÈGE N°2 : `UserId` est un objet, pas une string

| Ce que vous envoyez | Résultat |
|---------------------|----------|
| `"UserId": {"UserId":"guid","Display":"Nom"}` ✅ | 200 OK |
| `"UserId": "guid"` ❌ | **500 Internal Server Error** |

**Règle :** `UserId` est de type `UserAssignableModel`, toujours un objet contenant `UserId` (le GUID) et `Display` (le nom).

```typescript
// ✅ CORRECT
const payload = {
  UserId: {
    UserId: "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    Display: "Dupont Jean"
  },
  DateStart: "2026-05-26T00:00:00"
};

// ❌ INCORRECT — provoque une erreur 500
const payload = {
  UserId: "a1b2c3d4-e5f6-7890-abcd-ef1234567890",  // string simple !
  DateStart: "2026-05-26T00:00:00"
};
```

---

#### 🚨 PIÈGE N°3 : Création standalone impossible

```bash
# ❌ Ceci retourne 404 — pas de création sans référence
curl -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/activity"
```

**Règle :** Les activités SEIGMA sont toujours rattachées à une référence. Utilisez `POST /api/activity/{referenceId}`.

```bash
# ✅ Correct — création liée à une référence
curl -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/activity/ref-client-123"
```

---

#### 🚨 PIÈGE N°4 : L'endpoint de recherche est inopérant

```bash
# ❌ Retourne 500 Object reference not set to an instance of an object
curl -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/reference/Activity/search"
```

**Règle :** Ne pas utiliser l'endpoint de recherche générique pour les activités. Utiliser `getactivitiesfordate` pour le planning ou `GET /api/activity/{id}` pour une activité spécifique.

---

#### 🚨 PIÈGE N°5 : Fuseaux horaires

Les dates sont stockées en **UTC** mais affichées dans le fuseau horaire de l'utilisateur connecté. Lors de comparaisons avec les données brutes de la base de données, tenir compte du décalage horaire.

---

## 3.2 Timelog API

### 3.2.1 Aperçu général

L'API Timelog de SEIGMA gère les **poinçons** (suivi du temps passé) sur les références. Chaque poinçon est rattaché à une référence (client, contrat, équipement, bon de travail) et à un utilisateur. L'API supporte un mode **start/stop** pour chronométrer le temps en direct.

| Opération | Méthode | Endpoint | Description |
|-----------|---------|----------|-------------|
| Lister les poinçons | `GET` | `/api/reference/{ModelCode}/{refId}/timelogs` | Récupère tous les poinçons d'une référence |
| Ajouter un poinçon | `POST` | `.../timelogs/add` | Crée un nouveau poinçon |
| Modifier un poinçon | `PUT` | `.../timelogs` | Met à jour un poinçon existant |
| Supprimer un poinçon | `DELETE` | `.../timelogs/{timelogId}` | Supprime un poinçon |
| Démarrer le chrono | `GET` | `.../timelogs/start` | Démarre un poinçon (heure courante) |
| Arrêter le chrono | `GET` | `.../timelogs/{timelogId}/stop` | Arrête un poinçon en cours |

> ℹ️ **`{ModelCode}`** est le code du modèle de référence (ex : `Customer`, `Contract`, `WorkOrder`, `Equipment`). **`{refId}`** est le GUID de la référence.

---

### 3.2.2 GET …/timelogs — Lister les poinçons

Récupère la liste complète des poinçons associés à une référence.

**Requête :**

```bash
curl -X GET \
  "https://{VOTRE_INSTANCE}.seigma.app/api/reference/WorkOrder/wo-123-abc/timelogs" \
  -H "Authorization: Bearer {token}"
```

**Réponse (200 OK) :**

```json
{
  "ReferenceId": "wo-123-abc",
  "ModelCode": "WorkOrder",
  "TimeLogs": [
    {
      "TimeLogId": "tl-001",
      "StartTime": "2026-05-26T08:00:00",
      "EndTime": "2026-05-26T10:30:00",
      "LogDate": "2026-05-26T00:00:00",
      "Description": "Diagnostic panne chaudière + remplacement thermostat",
      "UserId": {
        "Display": "Dupont Jean",
        "ReferenceId": "user-guid-123",
        "ModelCode": "User"
      },
      "Duration": "02:30:00"
    },
    {
      "TimeLogId": "tl-002",
      "StartTime": "2026-05-26T13:00:00",
      "EndTime": "2026-05-26T14:00:00",
      "LogDate": "2026-05-26T00:00:00",
      "Description": "Vérification fonctionnement",
      "UserId": {
        "Display": "Dupont Jean",
        "ReferenceId": "user-guid-123",
        "ModelCode": "User"
      },
      "Duration": "01:00:00"
    }
  ]
}
```

**Structure de la réponse :**

| Champ | Type | Description |
|-------|------|-------------|
| `ReferenceId` | `GUID` | Identifiant de la référence |
| `ModelCode` | `string` | Code du modèle (ex: `WorkOrder`) |
| `TimeLogs[]` | `TimeLog[]` | Tableau des poinçons |

**Champs d'un poinçon (`TimeLog`) :**

| Champ | Type | Description |
|-------|------|-------------|
| `TimeLogId` | `GUID` | Identifiant unique du poinçon |
| `StartTime` | `DateTime` | Heure de début |
| `EndTime` | `DateTime` | Heure de fin (`null` si en cours) |
| `LogDate` | `DateTime` | Date du poinçon |
| `Description` | `string` | Description du travail effectué |
| `UserId` | `UserModel` | Utilisateur : `{Display, ReferenceId, ModelCode:"User"}` |
| `Duration` | `TimeSpan` | Durée calculée (format `HH:MM:SS`) |

**Implémentations :**

```typescript
// TypeScript
interface TimeLogUser {
  Display: string;
  ReferenceId: string;
  ModelCode: "User";
}

interface TimeLog {
  TimeLogId: string;
  StartTime: string;
  EndTime: string | null;
  LogDate: string;
  Description: string;
  UserId: TimeLogUser;
  Duration: string; // "HH:MM:SS"
}

interface TimelogsResponse {
  ReferenceId: string;
  ModelCode: string;
  TimeLogs: TimeLog[];
}

async function getTimelogs(
  modelCode: string,
  refId: string,
  token: string
): Promise<TimelogsResponse> {
  const res = await fetch(
    `https://{VOTRE_INSTANCE}.seigma.app/api/reference/${modelCode}/${refId}/timelogs`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

```python
# Python
def get_timelogs(model_code: str, ref_id: str, token: str) -> dict:
    """
    Récupère les poinçons d'une référence.
    Ex: model_code="WorkOrder", ref_id="wo-123-abc"
    """
    url = f"https://{VOTRE_INSTANCE}.seigma.app/api/reference/{model_code}/{ref_id}/timelogs"
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(url, headers=headers)
    resp.raise_for_status()
    return resp.json()
```

**Codes d'erreur :**

| Code | Signification |
|------|---------------|
| `200` | Succès |
| `401` | Non authentifié |
| `404` | Référence introuvable |

---

### 3.2.3 POST …/timelogs/add — Ajouter un poinçon

`POST …/timelogs/add` — Crée un poinçon sur une référence. Corps wrappé dans `{"timeLog": {...}}`. Champs obligatoires : `StartTime` (ISO 8601), `UserId` (format `{Display, ReferenceId, ModelCode:"User"}` — différent du modèle Activity).

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets, les corps de requête, les implémentations TypeScript/Python et les codes d'erreur.

---

### 3.2.4 PUT …/timelogs — Modifier un poinçon

`PUT …/timelogs` — Met à jour un poinçon existant. Corps wrappé dans `{"timeLog": {...}}`. Le `TimeLogId` doit être inclus dans le corps. Mise à jour partielle.

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets.

---

### 3.2.5 DELETE …/timelogs/{timelogId} — Supprimer un poinçon

`DELETE …/timelogs/{timelogId}` — Supprime définitivement un poinçon. Retourne 204 No Content.

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets.

---

### 3.2.6 GET …/timelogs/start — Démarrer un poinçon

`GET …/timelogs/start` — Démarre un chronomètre avec l'heure courante du serveur. `EndTime` = `null`. Utilisé pour chronométrer le travail en direct.

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets.

---

### 3.2.7 GET …/timelogs/{timelogId}/stop — Arrêter un poinçon

`GET …/timelogs/{timelogId}/stop` — Arrête un poinçon en cours. `EndTime` = heure serveur, `Duration` calculée automatiquement.

> → Voir [Chapitre 4 — Opérations d'écriture](04-operations-ecriture.md) pour les détails complets.

---

### 3.2.8 Pièges et erreurs fréquentes — Timelog API

#### 🚨 PIÈGE N°1 : L'endpoint `POST /api/timelogs/search` ne fonctionne pas

```bash
# ❌ Retourne 0 résultats — endpoint inopérant
curl -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/timelogs/search" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"WorkOrderId": "wo-123-abc"}'
```

**Règle :** Toujours utiliser la route **GET par référence** pour récupérer les poinçons :

```bash
# ✅ Correct — GET sur la référence
curl -X GET \
  "https://{VOTRE_INSTANCE}.seigma.app/api/reference/WorkOrder/wo-123-abc/timelogs" \
  -H "Authorization: Bearer {token}"
```

---

#### 🚨 PIÈGE N°2 : Incohérence du modèle `UserId` entre Activity API et Timelog API

SEIGMA utilise **deux modèles différents** pour représenter un utilisateur selon l'API :

| API | Modèle | Structure |
|-----|--------|-----------|
| **Activity API** | `UserAssignableModel` | `{UserId: guid, Display: string}` |
| **Timelog API** | `ModelAttributeList` (UserModel) | `{Display: string, ReferenceId: guid, ModelCode: "User"}` |

```typescript
// ✅ Activity API — UserAssignableModel
const activityUser = {
  UserId: "user-guid-123",
  Display: "Dupont Jean"
};

// ✅ Timelog API — ModelAttributeList (UserModel)
const timelogUser = {
  Display: "Dupont Jean",
  ReferenceId: "user-guid-123",  // Note : ReferenceId au lieu de UserId
  ModelCode: "User"
};

// ❌ Intervertir les deux modèles provoque des erreurs
```

**Règle :** Utiliser le modèle approprié selon l'API appelée. Ne jamais interchanger les structures.

---

#### 🚨 PIÈGE N°3 : Body wrappé obligatoire

Tous les corps de requête POST et PUT doivent être wrappés dans l'objet approprié :

| Endpoint | Wrapper |
|----------|---------|
| `POST …/activity/{refId}` | `{"activity": {...}}` |
| `PUT …/activity/{id}` | `{"activity": {...}}` |
| `POST …/timelogs/add` | `{"timeLog": {...}}` |
| `PUT …/timelogs` | `{"timeLog": {...}}` |

```bash
# ❌ INCORRECT — corps non wrappé
curl -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/reference/WorkOrder/wo-123/timelogs/add" \
  -d '{"StartTime": "...", "UserId": {...}}'

# ✅ CORRECT — corps wrappé dans "timeLog"
curl -X POST "https://{VOTRE_INSTANCE}.seigma.app/api/reference/WorkOrder/wo-123/timelogs/add" \
  -d '{"timeLog": {"StartTime": "...", "UserId": {...}}}'
```

---

## 3.3 Résumé des codes d'erreur

| Code HTTP | Signification | Fréquence |
|-----------|---------------|:---------:|
| `200` | Succès — GET, PUT | Courant |
| `201` | Créé avec succès — POST | Courant |
| `204` | Supprimé avec succès — DELETE (pas de corps) | Courant |
| `400` | Requête invalide (champs manquants, format incorrect) | Fréquent |
| `401` | Token d'authentification manquant, invalide ou expiré | Fréquent |
| `404` | Ressource introuvable (activité, référence, poinçon) | Fréquent |
| `500` | Erreur interne serveur — souvent dû à un format de champ incorrect (ex: `DateStart` sans timestamp, `UserId` en string simple, endpoint de recherche inopérant) | Fréquent en cas de piège |

---

## Références rapides

### Endpoints Activity API

```
GET    /api/activity/{activity_id}              → Récupérer une activité
POST   /api/activity/getactivitiesfordate        → Activités d'une date (planning)
POST   /api/activity/{referenceId}               → Créer une activité liée
PUT    /api/activity/{activityId}                → Modifier une activité
```

### Endpoints Timelog API

```
GET    /api/reference/{ModelCode}/{refId}/timelogs              → Lister les poinçons
POST   /api/reference/{ModelCode}/{refId}/timelogs/add          → Ajouter un poinçon
PUT    /api/reference/{ModelCode}/{refId}/timelogs              → Modifier un poinçon
DELETE /api/reference/{ModelCode}/{refId}/timelogs/{timelogId}  → Supprimer un poinçon
GET    /api/reference/{ModelCode}/{refId}/timelogs/start        → Démarrer le chrono
GET    /api/reference/{ModelCode}/{refId}/timelogs/{id}/stop    → Arrêter le chrono
```

### Modèles de données clés

| Modèle | Structure | Utilisé dans |
|--------|-----------|-------------|
| `UserAssignableModel` | `{UserId, Display}` | Activity API |
| `ModelAttributeList` | `{Display, ReferenceId}` | Activity API (types, labels, etc.) |
| `UserModel` (Timelog) | `{Display, ReferenceId, ModelCode:"User"}` | Timelog API |

---

◄ [Précédent : 02 — Référence API](02-reference-api.md) │ [Index](index.md) │ [Suivant : 04 — Opérations d'écriture](04-operations-ecriture.md) ►


