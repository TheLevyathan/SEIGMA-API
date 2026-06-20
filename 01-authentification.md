# Chapitre 1 — Authentification

---

## Table des matières

1. [Aperçu rapide](#1-aperçu-rapide)
2. [URL et méthode](#2-url-et-méthode)
3. [Corps de la requête](#3-corps-de-la-requête)
4. [Réponse (200 OK)](#4-réponse-200-ok)
5. [Codes d'erreur](#5-codes-derreur)
6. [Utilisation du jeton](#6-utilisation-du-jeton)
7. [Exemples d'intégration](#7-exemples-dintégration)
8. [Bonnes pratiques](#8-bonnes-pratiques)
9. [⚠️ Pièges connus](#9-️-pièges-connus)
10. [Résumé](#10-résumé)

---

L'endpoint `POST /api/auth/authenticate` échange un couple identifiant/mot de passe contre un jeton JWT valide 7 jours.

---

## 1. Aperçu rapide

```bash
curl -X POST https://{VOTRE_INSTANCE}.seigma.app/api/auth/authenticate \
  -H "Content-Type: application/json" \
  -H "Accept-Language: fr-CA" \
  -d '{"username":"{VOTRE_EMAIL}","password":"{VOTRE_MOT_DE_PASSE}"}'
```

**Réponse (200 OK) :**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## 2. URL et méthode

| Élément         | Valeur                                              |
|-----------------|-----------------------------------------------------|
| **URL de base** | `https://{VOTRE_INSTANCE}.seigma.app/api/`                  |
| **Endpoint**    | `POST /api/auth/authenticate`                       |
| **Content-Type**| `application/json`                                  |

> 🚧 **À compléter** : Remplacez `{VOTRE_INSTANCE}`, `{VOTRE_EMAIL}` et `{VOTRE_MOT_DE_PASSE}` par les credentials de votre propre instance SEIGMA.

Contactez votre administrateur SEIGMA si vous ne disposez pas d'un compte API dédié.

---

## 3. Corps de la requête

| Paramètre  | Type   | Obligatoire | Description                          |
|------------|--------|-------------|--------------------------------------|
| `username` | string | Oui         | Adresse courriel du compte SEIGMA.   |
| `password` | string | Oui         | Mot de passe en clair.               |

> ℹ️ **Note** — Le PDF officiel documente parfois le champ `"user"` au lieu de `"username"`. Les deux semblent acceptés par l'API, mais **privilégiez `"username"`** qui est le standard observé en production.

---

## 4. Réponse (200 OK)

| Champ   | Type   | Description                                              |
|---------|--------|----------------------------------------------------------|
| `token` | string | Jeton JWT signé HS256. Durée de vie : 604 800 secondes (7 jours). |

### 4.1 Structure décodée du JWT

```json
{
  "jti": "identifiant_unique_du_jeton",
  "sub": "{VOTRE_EMAIL}",
  "name": "Web",
  "aud": "{VOTRE_INSTANCE}.seigma.app",
  "iss": "{VOTRE_INSTANCE}.seigma.app",
  "nbf": 1718880000,
  "exp": 1719484800,
  "iat": 1718880000
}
```

| Claim  | Signification                                    |
|--------|--------------------------------------------------|
| `jti`  | Identifiant unique du jeton (JWT ID).            |
| `sub`  | Sujet = courriel de l'utilisateur authentifié.   |
| `name` | Nom affiché de l'utilisateur.                    |
| `aud`  | Audience = domaine cible.                         |
| `iss`  | Émetteur du jeton.                                |
| `nbf`  | *Not Before* — timestamp Unix.                   |
| `exp`  | Expiration — timestamp Unix.                     |
| `iat`  | *Issued At* — timestamp Unix d'émission.         |

> ℹ️ **Note** — Le JWT ne contient **ni scope ni rôle**. Une fois authentifié, le jeton donne un accès complet à toutes les ressources de l'entreprise associée.

---

## 5. Codes d’erreur

| Code HTTP | Cause probable                                      |
|-----------|-----------------------------------------------------|
| `404`     | Identifiants invalides (courriel ou mot de passe).  |
| `400`     | Corps de requête mal formé ou champ manquant.       |
| `500`     | Erreur interne du serveur d'authentification.       |

> ⚠️ **Attention** — L'API ne renvoie **pas de message d'erreur détaillé** en cas de 404. Traitez tout 404 comme « credentials invalides » sans tenter de deviner la raison précise.

---

## 6. Utilisation du jeton

Tous les appels subséquents à l'API nécessitent **trois en-têtes recommandés** (seigma-company peut être omis si le token JWT est associé à une seule compagnie) :

| En-tête             | Valeur                              | Description                               |
|---------------------|-------------------------------------|-------------------------------------------|
| `Authorization`     | `Bearer {token}`                    | Jeton JWT obtenu via l'authentification.  |
| `seigma-company`    | `{VOTRE_COMPANY_ID}` | GUID de l'entreprise SEIGMA.        |
| `Accept-Language`   | `fr-CA`                             | Langue des réponses (fr-CA pour le français canadien). |

---

## 7. Exemples d’intégration

### 7.1 TypeScript / Deno

```typescript
// (imports dépendent de votre runtime — Deno, Node.js, ou autre)

// ⚠️ Utilisez des variables d'environnement pour les valeurs sensibles
const SEIGMA_BASE = Deno.env.get("SEIGMA_BASE_URL")!; // ex: "https://instance.seigma.app/api"
const COMPANY_ID = Deno.env.get("SEIGMA_COMPANY_ID")!; // ex: "xxxxxxxx-xxxx-..."

async function authenticate(username: string, password: string): Promise<string> {
  const res = await fetch(`${SEIGMA_BASE}/auth/authenticate`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Accept-Language": "fr-CA",
    },
    body: JSON.stringify({ username, password }),
  });

  if (!res.ok) {
    throw new Error(`Authentification échouée — HTTP ${res.status}`);
  }

  const data = await res.json();
  return data.token as string;
}

// Appel API authentifié
async function fetchSeigma(endpoint: string, token: string) {
  return fetch(`${SEIGMA_BASE}${endpoint}`, {
    headers: {
      Authorization: `Bearer ${token}`,
      "seigma-company": COMPANY_ID,
      "Accept-Language": "fr-CA",
    },
  });
}
```

### 7.2 Python

```python
import requests

BASE_URL = "https://{VOTRE_INSTANCE}.seigma.app/api"
COMPANY_ID = "{VOTRE_COMPANY_ID}"

def authenticate(username: str, password: str) -> str:
    """Retourne le jeton JWT SEIGMA."""
    resp = requests.post(
        f"{BASE_URL}/auth/authenticate",
        json={"username": username, "password": password},
        headers={
            "Content-Type": "application/json",
            "Accept-Language": "fr-CA",
        },
    )
    resp.raise_for_status()
    return resp.json()["token"]

# Utilisation
token = authenticate("{VOTRE_EMAIL}", "{VOTRE_MOT_DE_PASSE}")

headers = {
    "Authorization": f"Bearer {token}",
    "seigma-company": COMPANY_ID,
    "Accept-Language": "fr-CA",
}
response = requests.get(f"{BASE_URL}/...", headers=headers)
```

---

## 8. Bonnes pratiques

| Pratique                                          | Raison                                           |
|---------------------------------------------------|--------------------------------------------------|
| Stockez le jeton en mémoire, jamais en clair sur disque. | Évite les fuites de credentials.          |
| Réauthentifiez avant l’expiration (7 jours).      | Le jeton expiré produit un 401 silencieux.       |
| Réutilisez le jeton entre les appels.             | Pas de rate limiting sur l’auth, mais inutile d’en abuser. |
| Renouvelez le jeton proactivement (ex. 6 jours).  | Marge de sécurité contre les expirations.        |

---

## 9. ⚠️ Pièges connus

### 9.1 Le champ `user` vs `username`

Le PDF officiel mentionne parfois `"user"` comme clé. Les tests terrain montrent que `"username"` fonctionne systématiquement. **Utilisez `"username"`** pour éviter toute ambiguïté.

### 9.2 Aucun scope, aucun rôle

Le jeton donne un accès **complet et non limité**. Une fois le token en circulation, toute personne peut appeler n'importe quel endpoint. Traitez-le comme un secret de niveau administrateur.

### 9.3 Pas de rate limiting sur l'auth

Aucune protection anti-brute-force n'est en place sur cet endpoint. Implémentez votre propre throttling côté client pour éviter de saturer le service ou de déclencher des blocages futurs.

### 9.4 Pas de captcha

L'absence de captcha facilite l'automatisation mais rend l'endpoint vulnérable. Soyez responsable dans votre fréquence d'appel.

### 9.5 Le 401 est muet

Vous ne saurez pas si c'est le courriel ou le mot de passe qui est erroné. Journalisez les échecs côté client mais ne présentez jamais cette ambiguïté à l'utilisateur final.

---

## 10. Résumé

1. **POST** sur `/api/auth/authenticate` avec `{"username": "...", "password": "..."}`.
2. Récupérez le `token` JWT dans la réponse.
3. Ajoutez trois en-têtes à chaque appel : `Authorization: Bearer {token}`, `seigma-company: {guid}`, `Accept-Language: fr-CA`.
4. Le jeton expire dans 7 jours — anticipez le renouvellement.

---

◄ [Précédent : Accueil — Index](index.md) │ [Index](index.md) │ [Suivant : 02 — Référence API](02-reference-api.md) ►
