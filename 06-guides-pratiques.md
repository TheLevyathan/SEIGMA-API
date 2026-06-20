# Chapitre 6 — Guides pratiques (Workflows)

> **Version documentée** : 1.0.0 | **Sources** : PDFs officiels SEIGMA + tests de production
> **Dernière mise à jour** : 2026-06-20

---

## Table des matières

1. [Workflow 1 : Facturation — Pipeline Lead → Paiement](#workflow-1--facturation--pipeline-lead--paiement)
2. [Workflow 2 : Planning équipes — getactivitiesfordate](#workflow-2--planning-équipes--getactivitiesfordate)
3. [Workflow 3 : Synchronisation par lots et performance](#workflow-3--synchronisation-par-lots-et-performance)
4. [Bonus : Chercher un WO par numéro](#bonus--chercher-un-wo-par-numéro)
5. [Aide-mémoire des patterns](#aide-mémoire-des-patterns)

---

## Workflow 1 — Facturation : Pipeline Lead → Paiement

### 6.1.1 Vue d'ensemble

Le pipeline de facturation traverse **5 modèles SEIGMA** en cascade :

```
Lead ──→ Quotation ──→ SalesOrder ──→ SalesInvoice ──→ Receipt
(Prospect)  (Soumission)  (Bon de travail)  (Facture)     (Paiement)
```

| Étape | Modèle | ModelCode | Méthode API | Particularité |
|-------|--------|-----------|-------------|---------------|
| 1 | Lead | `Lead` | `POST /search` | ~589 enregistrements, statut `LeadStatusId` |
| 2 | Soumission | `Quotation` | `POST /search` | Liée au Lead via `LeadId`, ~1935 enregistrements |
| 3 | Bon de travail | `SalesOrder` | `POST /search` + `GET /{id}` | ~1932 WO, 87 champs en détail |
| 4 | Facture | `SalesInvoice` | `POST /search` + `GET /{id}` | ⚠️ `SalesOrderId` transitif dans le search |
| 5 | Paiement | `Receipt` | `POST /search` + `GET /{id}` | ⚠️ `SalesOrderId` transitif — nécessite getDetail |

> ⚠️ **PITFALL CRITIQUE** — `SalesOrderId` est **transitif** sur `Receipt` et `SalesInvoice`. Cela signifie qu'un **search ne résout jamais** l'objet SalesOrder lié. Vous recevez seulement l'ID brut. Pour remonter du Receipt ou de la SalesInvoice au SalesOrder, vous **devez** faire un `GET /api/reference/Receipt/{id}` ou `GET /api/reference/SalesInvoice/{id}` pour obtenir le `SalesOrderId` complet.

### 6.1.2 Étape 1 — Récupérer les SalesOrders par statut

Utilisez `WhereCondition` pour filtrer par statut. Le champ `SalesOrderStatusId` est un objet `{ReferenceId, Display}` — le search retourne son `Display`, mais vous filtrez sur la **string affichée**.

```typescript
// Récupérer les WOs "Terminé" des 30 derniers jours
async function fetchCompletedOrders(
  token: string,
  sinceDate: string // ex: "2026-05-20T00:00:00" (format ISO 8601 YYYY-MM-DD)
): Promise<Record<string, unknown>[]> {
  const pages = await Promise.all(
    Array.from({ length: 30 }, (_, i) =>
      fetch(`${SEIGMA_BASE}/api/reference/SalesOrder/search`, {
        method: "POST",
        headers: seigmaHeaders(token),
        body: JSON.stringify({
          Offset: i * 50,
          Limit: 50,
          IsSelector: false,
          ModelListId: null,
          CurrentReferenceId: null,
          WhereCondition: [
            {
              ModelAttributeCode: "DateCreated",
              Operator: ">=",
              Value: sinceDate,
            },
            {
              ModelAttributeCode: "SalesOrderStatusId",
              Operator: "=",
              Value: "Terminé",
            },
          ],
          WhereOrCondition: [],
          WhereInCondition: [],
          OrderByCondition: [
            { ModelAttributeCode: "DateCreated", IsDescending: true },
          ],
        }),
      })
        .then((r) => r.json())
        .then((d) => (d.references ?? []) as Record<string, unknown>[])
    )
  );
  return pages.flat();
}
```

> ℹ️ **Note** — Le filtre `=` sur `SalesOrderStatusId` fonctionne car le search retourne le Display sous forme de string dans ce champ. ⚠️ **ATTENTION** : `Contains`, `StartsWith` et `EndsWith` sur ce champ retournent **HTTP 500** (crash serveur) — utilisez exclusivement `=`.

### 6.1.3 Étape 2 — Obtenir le client d'un WO

Le search retourne un `CustomerId` sous deux formes possibles : soit une **string brute** (le GUID), soit un **objet** `{ReferenceId, Display}`. Utilisez toujours un helper d'extraction :

```typescript
function extractRefId(field: unknown): string | null {
  if (!field) return null;
  if (typeof field === "string") return field;
  return ((field as Record<string, unknown>).ReferenceId ?? null) as string | null;
}

function extractRefDisplay(field: unknown): string {
  if (!field) return "";
  if (typeof field === "string") return field;
  return ((field as Record<string, unknown>).Display ?? "") as string;
}

// Récupérer le détail complet d'un client
async function getCustomerDetail(
  token: string,
  customerId: string
): Promise<Record<string, unknown> | null> {
  const r = await fetch(
    `${SEIGMA_BASE}/api/reference/Customer/${customerId}`,
    { headers: seigmaHeaders(token) }
  );
  if (!r.ok) return null;
  const d = await r.json();
  return (d.reference ?? null) as Record<string, unknown> | null;
}

// Usage
const wo = { CustomerId: "12f8d783-9d84-457f-9f0d-cce68e3bc65d" };
const customerId = extractRefId(wo.CustomerId);
const customer = await getCustomerDetail(token, customerId!);
console.log(customer?.Name, customer?.Email, customer?.Phone);
```

### 6.1.4 Étape 3 — Obtenir les timelogs d'un WO

L'endpoint `GET /api/reference/SalesOrder/{id}/timelogs` retourne la liste des poinçons associés au WO :

```typescript
async function fetchWOTimelogs(
  token: string,
  woId: string
): Promise<Record<string, unknown>[]> {
  const r = await fetch(
    `${SEIGMA_BASE}/api/reference/SalesOrder/${woId}/timelogs`,
    { headers: seigmaHeaders(token) }
  );
  if (!r.ok) return [];
  const d = await r.json();
  // La réponse est wrappée : { TimeLogs: [...] }
  return (d.TimeLogs ?? []) as Record<string, unknown>[];
}

// Récupérer la date du dernier poinçon
async function getLatestTimelogDate(
  token: string,
  woId: string
): Promise<string | null> {
  const logs = await fetchWOTimelogs(token, woId);
  if (logs.length === 0) return null;

  const latest = logs.reduce(
    (max, log) => {
      const t = new Date(log.StartTime as string).getTime();
      return t > max.t
        ? { t, date: new Date(log.StartTime as string).toISOString().slice(0, 10) }
        : max;
    },
    { t: 0, date: null as string | null }
  );
  return latest.date;
}
```

### 6.1.5 Étape 4 — Obtenir la facture liée à un WO

> ⚠️ **PITFALL** — Le search sur `SalesInvoice` ne contient **pas** le `SalesOrderId` dans les résultats (champ transitif). Vous devez passer par un **getDetail sur chaque SalesInvoice** pour obtenir le `SalesOrderId` et reconstituer le lien WO → Facture. Même chose pour `Receipt`.

Voici le pattern complet qui récupère les factures et paiements liés aux WOs en **deux passes** :

```typescript
// ── PATTERN : Lien WO → Facture + Paiement via getDetail ──

// Étape A : Search parallèle Receipts + Invoices (récupère les IDs)
const [receiptsBasic, invoicesBasic] = await Promise.all([
  fetchAll(token, "Receipt", 10),
  fetchAll(token, "SalesInvoice", 10),
]);

// Étape B : getDetail massif — lots de 25 (CRITIQUE : dans le MÊME Promise.all)
async function chunkedDetails(
  token: string,
  model: string,
  ids: string[],
  batchSize = 25
): Promise<(Record<string, unknown> | null)[]> {
  const results: (Record<string, unknown> | null)[] = [];
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map((id) =>
        fetch(`${SEIGMA_BASE}/api/reference/${model}/${id}`, {
          headers: seigmaHeaders(token),
        })
          .then((r) => (r.ok ? r.json() : null))
          .then((d) => (d?.reference ?? null) as Record<string, unknown> | null)
          .catch(() => null)
      )
    );
    results.push(...batchResults);
  }
  return results;
}

// Les DEUX lots doivent être dans le MÊME Promise.all pour éviter
// la séquence receipt→invoice qui cause timeout (v51).
const receiptIds = receiptsBasic.map((r) => r.ReferenceId as string);
const invoiceIds = invoicesBasic.map((r) => r.ReferenceId as string);

const [receiptDetails, invoiceDetails] = await Promise.all([
  chunkedDetails(token, "Receipt", receiptIds),
  chunkedDetails(token, "SalesInvoice", invoiceIds),
]);

// Étape C : Construire les maps WO → Receipt / WO → Invoice
const receiptByWo = new Map<string, Record<string, unknown>>();
for (const r of receiptDetails) {
  if (!r) continue;
  const woId = extractRefId(r.SalesOrderId); // Disponible SEULEMENT dans le détail
  if (!woId) continue;
  // Garder le receipt le plus récent si plusieurs pour le même WO
  const prev = receiptByWo.get(woId);
  if (!prev || (r.DateCreated as string) > (prev.DateCreated as string)) {
    receiptByWo.set(woId, r);
  }
}

const invoiceByWo = new Map<string, Record<string, unknown>>();
for (const inv of invoiceDetails) {
  if (!inv) continue;
  const woId = extractRefId(inv.SalesOrderId); // Disponible SEULEMENT dans le détail
  if (!woId) continue;
  invoiceByWo.set(woId, inv);
}
```

> 🚨 **CRITIQUE** — Les lots `receiptDetails` et `invoiceDetails` doivent être dans le **même `Promise.all`** (pas séquentiels). Une exécution séquentielle receipt→invoice cause un **timeout** sur les grosses instances (v51).

### 6.1.6 Pattern ETL — Conversion SalesOrder → structure de données

Le pattern de migration ETL convertit un SalesOrder SEIGMA (87 champs) en une structure de données pour votre base :

```typescript
interface WorkOrderRow {
  seigma_id: string;
  number: string | null;
  display: string | null;
  customer_id: string | null;
  customer_name: string;
  description: string | null;
  subtotal: number;
  total: number;
  tax_gst: number;
  tax_pst: number;
  balance: number;
  status: string;
  status_color: string;
  shipping_street: string | null;
  shipping_city: string | null;
  shipping_postal: string | null;
  shipping_name: string | null;
  assigned_to_id: string | null;
  assigned_to_name: string;
  date_created: string | null;
  date_modified: string | null;
  // Relations liées (optionnelles)
  receipt: ReceiptRow | null;
  invoice: InvoiceRow | null;
  latest_timelog_date: string | null;
  raw_json: Record<string, unknown>;
}

function shapeWorkOrder(
  wo: Record<string, unknown>,
  receiptByWo: Map<string, Record<string, unknown>>,
  invoiceByWo: Map<string, Record<string, unknown>>,
  timelogDateByWo?: Map<string, string | null>
): WorkOrderRow {
  const invoice = invoiceByWo.get(wo.ReferenceId as string);
  const receipt = receiptByWo.get(wo.ReferenceId as string);

  return {
    seigma_id: wo.ReferenceId as string,
    number: (wo.Number as string) ?? null,
    display: (wo.Display as string) ?? null,
    customer_id: extractRefId(wo.CustomerId),
    customer_name: extractRefDisplay(wo.CustomerId),
    description: (wo.Description as string) ?? null,
    subtotal: (wo.SubTotal as number) ?? 0,
    total: (invoice?.Total as number) ?? (wo.Total as number) ?? (wo.SubTotal as number) ?? 0,
    tax_gst: (wo.TaxGST as number) ?? 0,
    tax_pst: (wo.TaxPST as number) ?? 0,
    balance: (wo.Balance as number) ?? 0,
    status: extractRefDisplay(wo.SalesOrderStatusId),
    status_color: extractRefColor(wo.SalesOrderStatusId),
    shipping_street: (wo.ShippingStreet as string) ?? null,
    shipping_city: (wo.ShippingCity as string) ?? null,
    shipping_postal: (wo.ShippingPostalCode as string) ?? null,
    shipping_name: (wo.ShippingName as string) ?? null,
    assigned_to_id: extractUserId(wo.AssignedToId),
    assigned_to_name: extractUserDisplay(wo.AssignedToId),
    date_created: (wo.DateCreated as string) ?? null,
    date_modified: (wo.DateModified as string) ?? null,
    receipt: receipt ? shapeReceipt(receipt) : null,
    invoice: invoice ? shapeInvoice(invoice) : null,
    latest_timelog_date: timelogDateByWo?.get(wo.ReferenceId as string) ?? null,
    raw_json: wo,
  };
}

// Helpers d'extraction
function extractUserId(field: unknown): string | null {
  if (!field || typeof field !== "object") return null;
  return ((field as Record<string, unknown>).UserId ?? null) as string | null;
}

function extractUserDisplay(field: unknown): string {
  if (!field || typeof field !== "object") return "";
  return ((field as Record<string, unknown>).Display ?? "") as string;
}

function extractRefColor(field: unknown): string {
  if (!field || typeof field === "string") return "#888";
  return ((field as Record<string, unknown>).Color ?? "#888") as string;
}
```

### 6.1.7 Diagramme du pipeline complet

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PIPELINE FACTURATION                              │
│                                                                          │
│  [1] SalesOrder/search (30 pages × 50, filtré DateCreated >= J-30)      │
│       │                               │                                  │
│       ▼                               ▼                                  │
│  [2a] Receipt/search (10 pages)  [2b] SalesInvoice/search (10 pages)   │
│       │                               │                                  │
│       ▼                               ▼                                  │
│  [3a] chunkedDetails(Receipt,     [3b] chunkedDetails(SalesInvoice,     │
│       ids, batch=25)                   ids, batch=25)                    │
│       │  └─ Promise.all ←─────── MÊME APPEL ──────→  │                  │
│       ▼                                               ▼                  │
│  receiptByWo: Map<woId, Receipt>    invoiceByWo: Map<woId, Invoice>     │
│       │                                               │                  │
│       └────────────── Assemblage final ──────────────┘                  │
│                              │                                           │
│                              ▼                                           │
│               WorkOrderRow[] avec receipt + invoice                      │
│                                                                          │
│  [Optionnel] fetchWOTimelogs → latestTimelogDate par WO                 │
│              (batchs de 15, fire-and-forget)                             │
└─────────────────────────────────────────────────────────────────────────┘
```



---

## Workflow 2 — Planning équipes : getactivitiesfordate

### 6.2.1 Vue d'ensemble

Le planning quotidien de {VOTRE_ENTREPRISE} repose sur des équipes avec chacune un `UserId` unique. L'endpoint `POST /api/activity/getactivitiesfordate` retourne les activités d'un utilisateur (chef d'équipe) pour une date donnée.

### 6.2.2 Configuration des équipes

```typescript
// 🚧 **À compléter** : Remplacez les UserId ci-dessous par ceux de vos propres équipes.
// Obtenez les UserId via l'interface d'administration SEIGMA ou via l'API.
const SEIGMA_TEAMS = [
  { id: "team-1", name: "Équipe 1", userId: "{un-user-id-equipe-1}" },
  { id: "team-2", name: "Équipe 2", userId: "{un-user-id-equipe-2}" },
  { id: "team-3", name: "Équipe 3", userId: "{un-user-id-equipe-3}" },
  { id: "team-4", name: "Équipe 4", userId: "{un-user-id-equipe-4}" },
  { id: "team-5", name: "Équipe 5", userId: "{un-user-id-equipe-5}" },
  { id: "team-6", name: "Équipe 6", userId: "{un-user-id-equipe-6}" },
  { id: "team-7", name: "Équipe 7", userId: "{un-user-id-equipe-7}" },
  { id: "team-8", name: "Équipe 8", userId: "{un-user-id-equipe-8}" },
  { id: "team-9", name: "Équipe 9", userId: "{un-user-id-equipe-9}" },
  // Ajoutez ou retirez des équipes selon votre configuration
];
```

### 6.2.3 Fetch parallèle des équipes

```typescript
interface TeamActivity {
  id: string;
  teamId: string;
  teamName: string;
  title: string;
  startTime: string;   // ISO 8601 complet
  endTime: string;     // ISO 8601 complet
  status: string;
  workOrder?: string;
  location?: string;
  activityType: string;
  color: string;
  assignedTo?: string;
}

async function fetchAllTeamActivities(
  token: string,
  dateStr: string // ex: "2026-06-20" (format YYYY-MM-DD)
): Promise<{
  activities: TeamActivity[];
  totalCount: number;
  teams: number;
  elapsedMs: number;
}> {
  const startTime = Date.now();

  // ⚠️ FORMAT OBLIGATOIRE : ISO 8601 COMPLET avec timestamp
  // "2026-06-20"           → ❌ ERREUR 500
  // "2026-06-20T00:00:00"  → ✅ CORRECT
  const dateStart = `${dateStr}T00:00:00`;

  // Promise.all sur les équipes/configurations — appels simultanés
  const teamPromises = SEIGMA_TEAMS.map(async (team) => {
    try {
      const r = await fetch(
        `${SEIGMA_BASE}/api/activity/getactivitiesfordate`,
        {
          method: "POST",
          headers: seigmaHeaders(token),
          body: JSON.stringify({
            UserId: {
              UserId: team.userId,
              Display: team.name,
            },
            DateStart: dateStart,
          }),
        }
      );

      if (!r.ok) {
        console.warn(`Équipe ${team.name}: HTTP ${r.status}`);
        return [] as TeamActivity[];
      }

      const data = await r.json();
      const activities = (data.activities ?? []) as Record<string, unknown>[];

      return activities.map((a) => ({
        id: a.ActivityId as string,
        teamId: team.id,
        teamName: team.name,
        title: (a.Subject as string) || "Sans titre",
        startTime: a.StartDate as string,
        endTime: a.EndDate as string,
        status: (a.Status as string) || "unknown",
        workOrder: (a.ReferenceId as { Display?: string })?.Display || undefined,
        location: (a.Location || a.ReferenceShippingAddress) as string | undefined,
        activityType: (a.ActivityTypeId as { Display?: string })?.Display || "Type inconnu",
        color: (a.ActivityTypeId as { Color?: string })?.Color || "#888",
        assignedTo: (a.AssignedToId as { Display?: string })?.Display || undefined,
      }));
    } catch (err) {
      console.error(`Équipe ${team.name}:`, err);
      return [] as TeamActivity[];
    }
  });

  const results = await Promise.all(teamPromises);
  const allActivities = results.flat();
  const elapsedMs = Date.now() - startTime;

  return {
    activities: allActivities,
    totalCount: allActivities.length,
    teams: SEIGMA_TEAMS.length,
    elapsedMs,
  };
}
```

> 🚨 **PIÈGE CRITIQUE — `UserId`** : Le champ `UserId` est un **objet** `{UserId: guid, Display: string}`, **PAS une string simple**. Envoyer `"UserId": "guid-here"` provoque une **erreur 500**.

> 🚨 **PIÈGE CRITIQUE — `DateStart`** : Le format **ISO 8601 complet avec timestamp** est obligatoire : `"2026-06-20T00:00:00"`. Envoyer `"2026-06-20"` seul provoque une **erreur 500**.

### 6.2.4 Parse des timestamps UTC → heure locale (America/Toronto)

SEIGMA stocke les dates en **UTC**. Pour afficher les activités correctement en heure locale (America/Toronto = UTC-4/-5) :

```typescript
function utcToLocal(isoString: string, timeZone = "America/Toronto"): Date {
  return new Date(
    new Date(isoString).toLocaleString("en-US", { timeZone })
  );
}

function formatLocalTime(isoString: string): string {
  const d = utcToLocal(isoString);
  return d.toLocaleTimeString("fr-CA", {
    hour: "2-digit",
    minute: "2-digit",
    timeZone: "America/Toronto",
  });
}

function formatLocalDate(isoString: string): string {
  const d = utcToLocal(isoString);
  return d.toLocaleDateString("fr-CA", { timeZone: "America/Toronto" });
}

// Exemple
const activity = {
  startTime: "2026-06-20T12:00:00Z", // UTC
};
console.log(formatLocalTime(activity.startTime)); // "08:00" (EDT)
console.log(formatLocalDate(activity.startTime)); // "2026-06-20"
```

> ℹ️ **Note timezone** — Certaines activités (ex: « Registre employés sur place ») ont un `StartDate` à 04:00 UTC (= minuit EDT), ce qui peut paraître comme un jour différent en UTC. **Ne filtrez pas par date côté client** — le cache est keyed par `(date, teamId)` et contient les bonnes activités.

### 6.2.5 Pattern de cache mémoire (TTL 15 minutes)

```typescript
interface CacheEntry<T> {
  data: T;
  timestamp: number;
}

class MemoryCache<T> {
  private store = new Map<string, CacheEntry<T>>();
  private ttlMs: number;

  constructor(ttlMinutes = 15) {
    this.ttlMs = ttlMinutes * 60 * 1000;
  }

  get(key: string): T | undefined {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (Date.now() - entry.timestamp > this.ttlMs) {
      this.store.delete(key);
      return undefined;
    }
    return entry.data;
  }

  set(key: string, data: T): void {
    this.store.set(key, { data, timestamp: Date.now() });
  }

  async getOrFetch(
    key: string,
    fetcher: () => Promise<T>
  ): Promise<{ data: T; fromCache: boolean }> {
    const cached = this.get(key);
    if (cached) return { data: cached, fromCache: true };

    const fresh = await fetcher();
    this.set(key, fresh);
    return { data: fresh, fromCache: false };
  }

  clear(): void {
    this.store.clear();
  }
}

// Usage
const activitiesCache = new MemoryCache<TeamActivity[]>(15);

async function getCachedActivities(
  token: string,
  dateStr: string
): Promise<TeamActivity[]> {
  const cacheKey = `activities:${dateStr}`;
  const { data, fromCache } = await activitiesCache.getOrFetch(
    cacheKey,
    () => fetchAllTeamActivities(token, dateStr).then((r) => r.activities)
  );
  console.log(fromCache ? "📦 Cache hit" : "🌐 Fetch SEIGMA");
  return data;
}
```

### 6.2.6 Exemple complet — Page de planning

```typescript
// Planification complète d'une journée pour les équipes/configurations
async function buildDailyPlan(
  token: string,
  dateStr: string
): Promise<{
  date: string;
  teams: {
    id: string;
    name: string;
    activityCount: number;
    activities: TeamActivity[];
  }[];
  totalActivities: number;
  cacheInfo: { fromCache: boolean; fetchedAt: string };
}> {
  const { activities, fromCache } = await (async () => {
    const cacheKey = `plan:${dateStr}`;
    const cached = activitiesCache.get(cacheKey);
    if (cached) return { activities: cached, fromCache: true };

    const fresh = await fetchAllTeamActivities(token, dateStr);
    activitiesCache.set(cacheKey, fresh.activities);
    return { activities: fresh.activities, fromCache: false };
  })();

  // Grouper par équipe
  const teams = SEIGMA_TEAMS.map((team) => ({
    id: team.id,
    name: team.name,
    activities: activities
      .filter((a) => a.teamId === team.id)
      .sort(
        (a, b) =>
          new Date(a.startTime).getTime() - new Date(b.startTime).getTime()
      ),
    activityCount: activities.filter((a) => a.teamId === team.id).length,
  }));

  return {
    date: dateStr,
    teams,
    totalActivities: activities.length,
    cacheInfo: {
      fromCache,
      fetchedAt: new Date().toISOString(),
    },
  };
}
```

---

## Workflow 3 — Synchronisation par lots et performance

### 6.3.1 Stratégie de synchronisation

La synchronisation entre SEIGMA et votre base de données nécessite trois patterns :

| Pattern | Objectif | Quand |
|---------|----------|------|
| **fetchAll paginé** | Lire toutes les pages d'un modèle | Données maîtres (clients, produits) |
| **chunkedDetails** | Obtenir le détail de N IDs sans timeout | Enrichissement (Receipt → SalesInvoice → SalesOrder) |
| **throttledFetch** | Respecter les limites de concurrence | Toute synchronisation massive |

### 6.3.2 Pattern fetchAll — pagination dynamique

```typescript
const SEIGMA_BASE = `https://{VOTRE_INSTANCE}.seigma.app/api`;

interface FetchPageResult {
  references?: Record<string, unknown>[];
  metadata?: { totalCount: number };
}

async function fetchPage(
  token: string,
  model: string,
  offset: number,
  limit = 50,
  selectAttributes?: string[]
): Promise<FetchPageResult> {
  const body: Record<string, unknown> = {
    Offset: offset,
    Limit: limit,
    IsSelector: false,
    ModelListId: null,
    CurrentReferenceId: null,
    WhereCondition: [],
    WhereOrCondition: [],
    WhereInCondition: [],
    OrderByCondition: [{ ModelAttributeCode: "DateCreated", IsDescending: true }],
  };
  if (selectAttributes && selectAttributes.length > 0) {
    body.SelectAttributes = selectAttributes;
  }

  const r = await fetch(`${SEIGMA_BASE}/api/reference/${model}/search`, {
    method: "POST",
    headers: seigmaHeaders(token),
    body: JSON.stringify(body),
  });
  return r.json() as Promise<FetchPageResult>;
}

async function fetchTotalCount(token: string, model: string): Promise<number> {
  const result = await fetchPage(token, model, 0, 1, ["ReferenceId"]);
  return result.metadata?.totalCount ?? 0;
}

async function fetchAll(
  token: string,
  model: string,
  pageSize = 50,
  selectAttributes?: string[]
): Promise<Record<string, unknown>[]> {
  const total = await fetchTotalCount(token, model);
  const pages = Math.ceil(total / pageSize);

  const pagePromises = Array.from({ length: pages }, (_, i) =>
    fetchPage(token, model, i * pageSize, pageSize, selectAttributes)
  );

  const results = await Promise.all(pagePromises);
  return results.flatMap((p) => p.references ?? []);
}

// Usage
const allCustomers = await fetchAll(token, "Customer", 50, ["ReferenceId", "Name"]);
console.log(`${allCustomers.length} clients récupérés`);
```

### 6.3.3 Pattern chunkedDetails — résoudre N IDs sans timeout

```typescript
async function getDetail(
  token: string,
  model: string,
  id: string
): Promise<Record<string, unknown> | null> {
  const r = await fetch(`${SEIGMA_BASE}/api/reference/${model}/${id}`, {
    headers: seigmaHeaders(token),
  });
  if (!r.ok) return null;
  const d = await r.json();
  return (d.reference ?? null) as Record<string, unknown> | null;
}

async function chunkedDetails<T>(
  token: string,
  model: string,
  ids: string[],
  batchSize = 25
): Promise<(T | null)[]> {
  const results: (T | null)[] = [];
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map((id) => getDetail(token, model, id) as Promise<T | null>)
    );
    results.push(...batchResults);
  }
  return results;
}

// Usage typique : enrichir des Receipts avec leur SalesOrderId transitif
const receipts = await fetchAll(token, "Receipt", 50);
const receiptIds = receipts.map((r) => r.ReferenceId as string);

// Étape 1 : getDetail sur chaque Receipt → SalesInvoiceId
const receiptDetails = await chunkedDetails<{ SalesInvoiceId?: { ReferenceId: string } }>(
  token, "Receipt", receiptIds
);

// Étape 2 : getDetail sur chaque SalesInvoice → SalesOrderId
const invoiceIds = receiptDetails
  .map((r) => r?.SalesInvoiceId?.ReferenceId)
  .filter(Boolean) as string[];
const invoiceDetails = await chunkedDetails<{ SalesOrderId?: { ReferenceId: string } }>(
  token, "SalesInvoice", invoiceIds
);
```

### 6.3.4 Pattern throttledFetch — respecter la concurrence

```typescript
function createSemaphore(max: number) {
  let running = 0;
  const queue: (() => void)[] = [];

  const next = () => {
    if (queue.length > 0 && running < max) {
      running++;
      queue.shift()!();
    }
  };

  return {
    acquire: () =>
      new Promise<void>((resolve) => {
        const task = () => {
          resolve();
          running--;
          next();
        };
        running < max ? (running++, task()) : queue.push(task);
      }),
  };
}

const semaphore = createSemaphore(50);

async function throttledFetch(url: string, init?: RequestInit): Promise<Response> {
  await semaphore.acquire();
  try {
    return await fetch(url, init);
  } catch (err) {
    // Un échec réseau n'est pas un bannissement — réessayer une fois
    await new Promise((r) => setTimeout(r, 2000));
    return fetch(url, init);
  }
}

// Usage dans fetchAll
async function fetchAllThrottled(
  token: string,
  model: string,
  pageSize = 50
): Promise<Record<string, unknown>[]> {
  const total = await fetchTotalCount(token, model);
  const pages = Math.ceil(total / pageSize);

  const pagePromises = Array.from({ length: pages }, async (_, i) => {
    const body: Record<string, unknown> = {
      Offset: i * pageSize,
      Limit: pageSize,
      IsSelector: false,
      ModelListId: null,
      CurrentReferenceId: null,
      WhereCondition: [],
      WhereOrCondition: [],
      WhereInCondition: [],
      OrderByCondition: [],
    };

    const r = await throttledFetch(`${SEIGMA_BASE}/api/reference/${model}/search`, {
      method: "POST",
      headers: seigmaHeaders(token),
      body: JSON.stringify(body),
    });
    const d = (await r.json()) as FetchPageResult;
    return d.references ?? [];
  });

  const results = await Promise.all(pagePromises);
  return results.flat();
}
```

> ⚠️ **Attention** : SEIGMA limite le nombre de requêtes simultanées. Un sémaphore à 50 est une valeur de départ prudente. Ajustez selon le comportement observé sur votre instance. En cas d'erreurs 500 soudaines sur de gros volumes, réduisez à 25.

> 🚧 **À compléter** : Les limites exactes de rate limiting varient selon votre instance SEIGMA. Testez avec un petit lot (10 pages) avant de lancer une synchronisation massive.

### 6.3.5 Cache local — pattern TTL mémoire

Pour éviter de refetch les mêmes données dans une session :

```typescript
const memoryCache = new Map<string, { data: unknown; timestamp: number }>();
const CACHE_TTL = 15 * 60 * 1000; // 15 minutes

function getCached<T>(key: string): T | null {
  const entry = memoryCache.get(key);
  if (entry && Date.now() - entry.timestamp < CACHE_TTL) {
    return entry.data as T;
  }
  memoryCache.delete(key);
  return null;
}

function setCache(key: string, data: unknown): void {
  memoryCache.set(key, { data, timestamp: Date.now() });
}

// Usage
async function getCachedActivities(token: string, teamUserId: string, date: string) {
  const cacheKey = `activities:${teamUserId}:${date}`;
  const cached = getCached<SeigmaActivity[]>(cacheKey);
  if (cached) return cached;

  const activities = await fetchActivities(token, teamUserId, date);
  setCache(cacheKey, activities);
  return activities;
}
```

> 💡 **Astuce** : Pour une persistance entre redémarrages, stockez le cache dans votre base de données avec une colonne `expires_at`. Faites un upsert avec `ON CONFLICT` plutôt qu'un insert pour éviter les doublons.

## Bonus — Chercher un WO par numéro

### 6.4.1 Pattern WhereCondition avec Number

Pour chercher un WO par son numéro (ex: `WO-00713`), utilisez `WhereCondition` avec `ModelAttributeCode: "Number"` et **l'opérateur `=`** :

```typescript
async function findWOByNumber(
  token: string,
  number: string // ex: "713", PAS "WO-00713"
): Promise<Record<string, unknown> | null> {
  // ⚠️ Utilisez le numéro BRUT, pas le Display
  const r = await fetch(`${SEIGMA_BASE}/api/reference/SalesOrder/search`, {
    method: "POST",
    headers: seigmaHeaders(token),
    body: JSON.stringify({
      Offset: 0,
      Limit: 1,
      IsSelector: false,
      ModelListId: null,
      CurrentReferenceId: null,
      WhereCondition: [
        {
          ModelAttributeCode: "Number",
          Operator: "=",
          Value: number, // "713" et NON "WO-00713"
        },
      ],
      WhereOrCondition: [],
      WhereInCondition: [],
      OrderByCondition: [],
    }),
  });

  const d = await r.json();
  const refs = (d.references ?? []) as Record<string, unknown>[];
  return refs.length > 0 ? refs[0] : null;
}

// Usage
const wo = await findWOByNumber(token, "713");
if (wo) {
  console.log(`Trouvé: ${wo.Display}`);       // "WO-00713"
  console.log(`Client: ${extractRefDisplay(wo.CustomerId)}`);
  console.log(`Total: ${wo.Total}`);
}
```

> ⚠️ **PITFALL** — Filtrez sur `Number` (le numéro séquentiel brut), **PAS** sur `Display` (le nom formaté). `"713"` est correct, `"WO-00713"` ne matchera pas.

### 6.4.2 Recherche par numéro + getDetail

Pour obtenir le détail complet du WO trouvé :

```typescript
async function getFullWOByNumber(
  token: string,
  number: string
): Promise<Record<string, unknown> | null> {
  // Étape 1 : Chercher le WO
  const woSummary = await findWOByNumber(token, number);
  if (!woSummary) return null;

  // Étape 2 : Obtenir le détail complet (87 champs)
  const woId = woSummary.ReferenceId as string;
  const r = await fetch(
    `${SEIGMA_BASE}/api/reference/SalesOrder/${woId}`,
    { headers: seigmaHeaders(token) }
  );
  if (!r.ok) return null;

  const d = await r.json();
  return (d.reference ?? null) as Record<string, unknown> | null;
}

// Usage
const fullWO = await getFullWOByNumber(token, "1808");
if (fullWO) {
  console.log("Détails complets:", {
    number: fullWO.Number,
    display: fullWO.Display,
    customer: extractRefDisplay(fullWO.CustomerId),
    shipping: `${fullWO.ShippingStreet}, ${fullWO.ShippingCity} ${fullWO.ShippingPostalCode}`,
    subtotal: fullWO.SubTotal,
    total: fullWO.Total,
    taxGST: fullWO.TaxGST,
    taxPST: fullWO.TaxPST,
    balance: fullWO.Balance,
    assignedTo: extractUserDisplay(fullWO.AssignedToId),
    created: fullWO.DateCreated,
    status: extractRefDisplay(fullWO.SalesOrderStatusId),
  });
}
```

---

## Aide-mémoire des patterns

### Tableau récapitulatif

| Pattern | Usage | Complexité | Piège principal |
|---|---|---|---|
| **fetchAll** | Récupérer N pages en parallèle | O(pages) | Pas de plafond strict, O(n²) sur très larges volumes |
| **chunkedDetails** | getDetail sur N IDs | O(N/25) lots | Toujours dans le même Promise.all |
| **cache mémoire** | Éviter les refetch | O(1) | TTL cohérent (15 min) |
| **cache persistant** | Persistance entre redémarrages | O(1) + 1 write | Upsert dans votre base |
| **WhereCondition** | Filtrer par champ | O(1) | Number ≠ Display, Operator pas OperatorCode |
| **extractRefId/Display** | Extraire d'un objet Reference | O(1) | Le champ peut être string OU objet |
| **throttledFetch** | Respecter rate limit | Sémaphore(50) | Dépassement → bannissement IP |

### Checklist de débogage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Erreur 500 sur search | `Limit` > 1000 (pas de plafond strict) | Réduire et paginer |
| Timeout sur getDetail massif | Trop d'appels séquentiels | `chunkedDetails` avec lots de 25 |
| Données manquantes après search | `SalesOrderId` transitif | getDetail sur chaque référence |
| Erreur 500 sur getactivitiesfordate | `UserId` en string simple | Utiliser `{UserId, Display}` |
| Erreur 500 sur getactivitiesfordate | `DateStart` sans timestamp | Ajouter `T00:00:00` |
| Auth 401 après 7 jours | JWT expiré | Réauthentifier proactivement (6 jours) |
| Bannissement IP | >50 requêtes parallèles | Sémaphore(50) + retry |

---

*Fin du chapitre 6 — Guides pratiques*

---

◄ [Précédent : 05 — Modèles de référence](05-modeles-reference.md) │ [Index](index.md) │ [Suivant : 07 — Pièges et limitations](07-pitfalls-limitations.md) ►
