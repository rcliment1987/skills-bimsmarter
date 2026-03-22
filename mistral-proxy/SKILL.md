---
name: mistral-proxy
description: Architecture et maintenance du proxy PHP BIMSmarter à 2 niveaux — mistral-proxy-local.php (auth, quota, sécurité, system prompt par app) + shared/mistral-proxy.php (appel Mistral/Groq/Perplexity). Utilise ce skill quand l'utilisateur parle de son proxy PHP, d'erreurs 429, de timeout N8N, de Perplexity, de Groq fallback, de déploiement sur une nouvelle app, de quota IA, de rate limiting, de system prompt serveur, ou de debug des réponses vides. Si l'utilisateur mentionne "proxy", "mistral-proxy.php", "proxy local", "l'IA ne répond plus", "429", "quota atteint", utilise ce skill.
---

# Mistral Proxy — BIMSmarter (Architecture à 2 niveaux)

## Principe fondamental

L'architecture est **à 2 niveaux** :

1. **`mistral-proxy-local.php`** — déployé dans chaque sous-domaine d'app. Gère toute la sécurité, la validation, le quota, et le system prompt spécifique à l'app.
2. **`/home/u313130122/shared/mistral-proxy.php`** — proxy partagé. Gère uniquement l'appel API (Mistral, Groq fallback, Perplexity, rotation de clés).

**Règle critique** : la sécurité (auth, quota, CORS, rate limiting) est TOUJOURS dans le proxy local. Le shared proxy ne fait que relayer vers les APIs.

---

## Architecture complète

```
App React (sous-domaine)
    ↓ POST JSON + Bearer token Firebase
[app]/mistral-proxy-local.php
    ├─ 1. Headers sécurité (X-Content-Type-Options, X-Frame-Options, Cache-Control)
    ├─ 2. CORS restreint (bimsmarter.eu + www.bimsmarter.eu)
    ├─ 3. Auth Firebase (token → Identity Toolkit → uid)
    ├─ 4. Quota IA serveur (Firestore: users/{uid}/aiQuota — par jour, par plan)
    ├─ 5. Rate limiting par uid (20 req/60s via fichier lock tmp)
    ├─ 6. Validation payload (taille max, JSON valide, rôles autorisés)
    ├─ 7. System prompt injecté côté serveur (spécifique à l'app)
    ├─ 8. [Optionnel] Appel n8n prioritaire (RAG) avec fallback
    │
    └─→ require('/home/u313130122/shared/mistral-proxy.php')
         ├─ api=perplexity ?  → Perplexity API
         ├─ Mistral API (rotation clés, retry si 429)
         └─ Mistral KO → Groq fallback (5 clés)
```

---

## Couche 1 : mistral-proxy-local.php (par app)

### Responsabilités

Chaque proxy local implémente tout ou partie de ces blocs dans cet ordre :

| # | Bloc | Obligatoire | Description |
|---|------|-------------|-------------|
| 1 | Headers sécurité | ✅ | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Cache-Control: no-store` |
| 2 | CORS restreint | ✅ | Origines autorisées : `https://bimsmarter.eu`, `https://www.bimsmarter.eu` |
| 3 | Auth Firebase | ✅ | Bearer token → Identity Toolkit v1 `accounts:lookup` → extraction uid |
| 4 | Quota IA serveur | ✅ | Firestore REST API : lecture plan + compteur quotidien + incrémentation PATCH |
| 5 | Rate limiting | Recommandé | 20 req/60s par uid, fichier lock dans `sys_get_temp_dir()` |
| 6 | Validation payload | ✅ | POST uniquement, Content-Type JSON, taille max (32-50KB), JSON valide, rôles `user`/`assistant` uniquement |
| 7 | System prompt serveur | ✅ | Spécifique à l'app, injecté côté serveur (jamais côté client) |
| 8 | Appel n8n (RAG) | Selon app | Prioritaire avec fallback vers Mistral direct si n8n HS |

### Plans et quotas IA

```php
$planLimits = [
    'free'       => 20,    // 20 msg/jour
    'pro'        => 100,   // 100 msg/jour
    'enterprise' => 500,   // 500 msg/jour
    'admin'      => 9999   // illimité (bypass quota)
];
```

Le quota est stocké dans Firestore : `users/{uid}/aiQuota` → `{ date: "YYYY-MM-DD", count: N }`.
Le compteur se réinitialise automatiquement quand la date change.

### Validation du token Firebase

```php
// Endpoint utilisé pour valider le token
$verifyUrl = 'https://identitytoolkit.googleapis.com/v1/accounts:lookup?key=' . $firebaseApiKey;
// POST avec { idToken: "..." }
// Réponse attendue : users[0].localId = uid
```

La clé API Firebase (`AIzaSyBzghrSLYAjcKYskfkrc7dbSX3DKm_rIEM`) est la Web API Key **publique** — elle identifie le projet, elle ne donne aucun accès privilégié.

### System prompts par app

Chaque app a un system prompt spécifique injecté côté serveur. Le client ne peut PAS envoyer de rôle `system` — tout message avec un rôle non autorisé est rejeté.

| App | Contenu du system prompt | Particularité |
|-----|--------------------------|---------------|
| Document Generator | Nomenclature ISO 19650 Luxembourg, codes phases/émetteurs/niveaux/types | JSON caché `<!--DOCS_JSON:[...]-->` en fin de réponse |
| Workflow Generator | Expert BIM ISO 19650 + réglementation luxembourgeoise | JSON caché `<!--WORKFLOW_JSON:{...}-->` en fin de réponse |
| GID-Assistant | Données GID CRTI-B validées | Via n8n RAG (pas de system prompt local) |

**Pattern JSON caché** : certaines apps demandent à Mistral d'inclure un bloc JSON invisible (`<!--TYPE_JSON:...-->`) en fin de réponse. Le frontend le parse ensuite pour peupler l'UI (ajout de documents, création de workflows, etc.).

### Transmission au shared proxy

Deux mécanismes coexistent selon les apps :

**Mécanisme A — `$GLOBALS` + capture output** (Document Generator) :
```php
$GLOBALS['_BIMSMARTER_PROXY_DATA'] = $data;
ob_start();
require($proxyPath);
$proxyOutput = ob_get_clean();
// Injection quota dans la réponse JSON
$responseData = json_decode($proxyOutput, true);
if ($responseData !== null && isset($GLOBALS['_BIMSMARTER_QUOTA'])) {
    $responseData['quota'] = $GLOBALS['_BIMSMARTER_QUOTA'];
    echo json_encode($responseData);
} else {
    echo $proxyOutput;
}
```

**Mécanisme B — Stream wrapper override** (Workflow Generator) :
```php
// Override php://input pour que le shared proxy lise l'input modifié
class BimProxyInputStream { ... }
$modifiedInput = json_encode($data, JSON_UNESCAPED_UNICODE);
BimProxyInputStream::setData($modifiedInput);
stream_wrapper_unregister('php');
stream_wrapper_register('php', 'BimProxyInputStream');
require($proxyPath);
stream_wrapper_restore('php');
```

**Mécanisme A est préféré** — plus simple, moins fragile. Le mécanisme B (stream wrapper) est un legacy qui fonctionne mais qu'on évite pour les nouvelles apps.

### Appel n8n prioritaire (apps avec RAG)

Certaines apps (Workflow Generator, GID-Assistant) appellent n8n en priorité :

```php
$n8nPayload = json_encode([
    'chatInput'  => $lastUserMsg,
    'session_id' => $uid,
    'app_id'     => 'bim-workflow-generator'
]);
$n8nUrl = 'https://n8n.bimsmarter.eu/webhook/rag-apps';
// Timeout : 60s
// Si réponse valide (output > 10 chars) → retourner directement
// Sinon → fallback vers shared/mistral-proxy.php
```

---

## Couche 2 : shared/mistral-proxy.php

**Emplacement** : `/home/u313130122/shared/mistral-proxy.php`

**Ne PAS modifier sauf** pour ajouter/mettre à jour des clés API ou modifier la logique d'appel Mistral/Groq/Perplexity.

### Configuration actuelle

```php
// MISTRAL — 1 clé (plan payant 6 req/sec)
$MISTRAL_KEYS = ['3T3OxFwRPg20i9OfHQjbjwizuSKw57dX'];

// GROQ FALLBACK — 5 clés en rotation
$GROQ_KEYS = [5 clés configurées];

// PERPLEXITY — 1 clé
$PERPLEXITY_KEY = 'pplx-...';

// N8N RAG
$N8N_URL = 'https://n8n.bimsmarter.eu/webhook/rag-apps';
$USE_N8N = true;

// Fichiers de rotation (dans shared/)
$ROTATION_FILE      = __DIR__ . '/.mistral_rotation_index';
$GROQ_ROTATION_FILE = __DIR__ . '/.groq_rotation_index';
```

### Apps détectées (via HTTP_HOST)

| Domaine contient | APP_ID |
|-----------------|--------|
| `gid-assistant` | `gid-assistant` |
| `bim-document-generator` | `bim-document-generator` |
| `bim-workflow-generator` | `bim-workflow-generator` |
| `gestion-projet` | `gestion-projet` |
| `chat` | `chat` |
| autre | `unknown` |

L'APP_ID peut aussi être forcé via le body JSON : `"app_id": "mon-app"`.

### Logique de fallback

```
1. N8N RAG → si réponse non vide → retourner immédiatement (source: n8n-rag)
2. Mistral API → si 429 → retry sur clé suivante (max nb_clés - 1 fois, délai 1.2s)
3. Si Mistral toujours KO (429 ou 500) → Groq fallback
4. Groq → rotation sur 5 clés → retry si 429 (délai 0.5s)
```

---

## Format d'appel depuis React

```javascript
const response = await fetch('/mistral-proxy-local.php', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${idToken}` // Firebase Auth ID token
    },
    body: JSON.stringify({
        messages: [{ role: 'user', content: 'question' }],
        model: 'mistral-small-latest' // optionnel
    })
});
const data = await response.json();
const content = data.choices?.[0]?.message?.content || 'Pas de réponse';
const quota = data.quota; // { count, limit, date, plan }
```

**Important** : le system prompt n'est JAMAIS envoyé par le client. Il est injecté par le proxy local.

### Perplexity (bypass n8n)
```javascript
body: JSON.stringify({
    messages: [...],
    api: 'perplexity',
    model: 'sonar'
})
```

---

## Déploiement nouvelle app

1. **Créer le sous-domaine** sur Hostinger
2. **Créer `mistral-proxy-local.php`** dans le dossier de l'app avec les blocs suivants :
   - Headers sécurité
   - CORS restreint
   - Auth Firebase (token → uid)
   - Quota IA serveur (Firestore)
   - Rate limiting (recommandé)
   - Validation payload (POST, JSON, taille, rôles)
   - System prompt spécifique à l'app
   - Transmission au shared proxy via `$GLOBALS['_BIMSMARTER_PROXY_DATA']` + `ob_start()`/`ob_get_clean()`
3. **Ajouter la détection du domaine** dans `shared/mistral-proxy.php` :
   ```php
   elseif (strpos($host, 'nouvelle-app') !== false) $APP_ID = 'nouvelle-app';
   ```
4. **Tester** :
   ```bash
   curl -X POST https://nouvelle-app.bimsmarter.eu/mistral-proxy-local.php \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer <FIREBASE_ID_TOKEN>" \
     -d '{"messages":[{"role":"user","content":"test"}]}'
   ```
   **Note** : sans le Bearer token, la requête sera rejetée 401.

### Si l'app utilise n8n RAG

Ajouter le bloc d'appel n8n prioritaire **avant** le `require(shared)` :
- Extraire le dernier message user
- Envoyer à `https://n8n.bimsmarter.eu/webhook/rag-apps` avec `chatInput`, `session_id` (= uid), `app_id`
- Si réponse valide → retourner directement (format `{ choices: [{ message: { role, content } }], quota }`)
- Sinon → fallback vers le shared proxy

---

## Diagnostic des problèmes fréquents

### 401 Unauthorized
- Token Firebase manquant ou expiré
- Vérifier que le frontend envoie `Authorization: Bearer ${idToken}`
- Vérifier que le token est rafraîchi avant expiration (1h par défaut)

### 429 Quota IA atteint
- Le compteur quotidien dans Firestore a atteint la limite du plan
- Vérifier `users/{uid}/aiQuota` dans Firestore
- L'admin est exempt (`plan === 'admin'`)

### 429 Rate limit exceeded (20 req/min)
- Trop de requêtes en < 60 secondes pour le même uid
- Fichier lock dans `sys_get_temp_dir()` : `rl_bimwf_{md5(uid)}.json`
- Supprimer le fichier lock pour reset immédiat

### 429 Mistral (API rate limit)
- Le plan Mistral autorise 6 req/sec — le fallback Groq prend le relais automatiquement
- Si n8n appelle aussi Mistral en parallèle → consommation double
- Vérifier les workflows n8n actifs

### Réponse vide ou null côté React
- Vérifier format de lecture : `data.choices?.[0]?.message?.content`
- Pour les apps avec n8n : la réponse n8n est déjà reformatée au format `choices`
- Côté PHP : `error_log()` dans le proxy local pour tracer la réponse

### n8n timeout (réponses sans contexte RAG)
- Timeout n8n = 60s dans le proxy local
- Symptôme : réponse correcte mais sans référence aux docs spécifiques
- Le fallback Mistral direct répond sans contexte RAG
- Vérifier les logs n8n + perfs Supabase

### APP_ID = 'unknown'
- Le domaine ne matche aucun pattern dans le shared proxy
- Ajouter le domaine dans le bloc de détection
- Ou forcer via `"app_id": "mon-app"` dans le body JSON

### Quota non incrémenté
- Le PATCH Firestore peut échouer silencieusement (@ devant file_get_contents)
- Vérifier que le token Firebase a les droits d'écriture sur `users/{uid}`
- Les règles Firestore doivent permettre au user authentifié d'écrire son propre doc

---

## Coûts de référence

| Modèle | Input | Output | Rôle |
|--------|-------|--------|------|
| Mistral Small | $0.10/M | $0.30/M | Principal |
| Mistral Large | $2/M | $6/M | Éviter |
| Groq Llama 3.3 70B | $0.59/M | $0.79/M | Fallback automatique |
| Perplexity Sonar | variable | variable | Recherche web temps réel |

---

## Anti-patterns à éviter

- ❌ Mettre la sécurité (auth, quota) dans le shared proxy → chaque app a ses propres besoins
- ❌ Envoyer le system prompt depuis le client → toujours côté serveur dans le proxy local
- ❌ Utiliser le stream wrapper (`BimProxyInputStream`) pour les nouvelles apps → préférer `$GLOBALS` + `ob_start`
- ❌ Hardcoder le chemin du shared proxy → utiliser `getenv('MISTRAL_PROXY_PATH')` avec fallback
- ❌ Oublier de restaurer `php://input` si stream wrapper utilisé → `stream_wrapper_restore('php')`
