# Chapitre 1 — Authentification

L'endpoint `POST /api/auth/authenticate` échange un couple identifiant/mot de passe contre un jeton JWT valide 7 jours.

---

## 1. Aperçu rapide

```bash
curl -X POST https://blvitres.seigma.app/api/auth/authenticate \
  -H "Content-Type: application/json" \
  -H "Accept-Language: fr-CA" \
  -d '{"username":"web@blvitres.com","password":"votre_mot_de_passe"}'
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
| **URL de base** | `https://blvitres.seigma.app/api/`                  |
| **Endpoint**    | `POST /api/auth/authenticate`                       |
| **Content-Type**| `application/json`                                  |

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
  "sub": "web@blvitres.com",
  "name": "Web",
  "aud": "blvitres.seigma.app",
  "iss": "blvitres.seigma.app",
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
| `401`     | Identifiants invalides (courriel ou mot de passe).  |
| `400`     | Corps de requête mal formé ou champ manquant.       |
| `500`     | Erreur interne du serveur d'authentification.       |

> ⚠️ **Attention** — L'API ne renvoie **pas de message d'erreur détaillé** en cas de 401. Traitez tout 401 comme « credentials invalides » sans tenter de deviner la raison précise.

---

## 6. Utilisation du jeton

Tous les appels subséquents à l'API nécessitent **trois en-têtes obligatoires** :

| En-tête             | Valeur                              | Description                               |
|---------------------|-------------------------------------|-------------------------------------------|
| `Authorization`     | `Bearer {token}`                    | Jeton JWT obtenu via l'authentification.  |
| `seigma-company`    | `8d5c5ee5-746e-4a2b-ba80-ce73393916e5` | GUID de l'entreprise SEIGMA.        |
| `Accept-Language`   | `fr-CA`                             | Langue des réponses (fr-CA pour le français canadien). |

---

## 7. Exemples d’intégration

### 7.1 TypeScript / Deno — Edge Function Supabase

```typescript
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

const SEIGMA_BASE = "https://blvitres.seigma.app/api";
const COMPANY_ID = "8d5c5ee5-746e-4a2b-ba80-ce73393916e5";

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

BASE_URL = "https://blvitres.seigma.app/api"
COMPANY_ID = "8d5c5ee5-746e-4a2b-ba80-ce73393916e5"

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
token = authenticate("web@blvitres.com", "mon_mot_de_passe")

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

*Chapitre suivant : [02-structure-api.md](./02-structure-api.md)*
