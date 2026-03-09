---
name: mistral-proxy
description: Architecture et maintenance du proxy PHP BIMSmarter (shared/mistral-proxy.php v5.1) qui centralise les appels Mistral, Perplexity et Groq fallback pour toutes les apps. Utilise ce skill quand l'utilisateur parle de son proxy PHP, d'erreurs 429, de timeout N8N, de Perplexity, de Groq fallback, de déploiement sur une nouvelle app, ou de debug des réponses vides. Si l'utilisateur mentionne "proxy", "mistral-proxy.php", "l'IA ne répond plus", "429", utilise ce skill.
---

---
name: mistral-proxy
description: Architecture et maintenance du proxy PHP BIMSmarter (shared/mistral-proxy.php v5.1) qui centralise les appels Mistral, Perplexity et Groq fallback pour toutes les apps. Utilise ce skill quand l'utilisateur parle de son proxy PHP, d'erreurs 429, de timeout N8N, de Perplexity, de Groq fallback, de déploiement sur une nouvelle app, ou de debug des réponses vides. Si l'utilisateur mentionne "proxy", "mistral-proxy.php", "l'IA ne répond plus", "429", utilise ce skill.
---

# Mistral Proxy — BIMSmarter (v5.1)

Proxy PHP centralisé dans `/home/u313130122/shared/mistral-proxy.php`. Chaque app a son propre `mistral-proxy.php` minimaliste qui fait juste :

```php
<?php
require('/home/u313130122/shared/mistral-proxy.php');
```

**Règle** : toujours modifier le fichier **shared**, jamais les fichiers d'app.

---

## Architecture complète

```
Apps React
    ↓ POST JSON
[app]/mistral-proxy.php  →  require('/home/u313130122/shared/mistral-proxy.php')
                         │
                         ├─ api=perplexity ?  → Perplexity API (bypass N8N)
                         │
                         ├─ USE_N8N=true ?    → N8N RAG webhook (timeout 25s)
                         │       └─ réponse RAG trouvée → retourner directement
                         │
                         ├─ Mistral API (rotation clés, retry si 429)
                         │
                         └─ Mistral KO (429 ou 500) → Groq fallback (5 clés)
```

---

## Configuration actuelle

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

---

## Apps détectées (via HTTP_HOST)

| Domaine contient | APP_ID |
|-----------------|--------|
| `gid-assistant` | `gid-assistant` |
| `bim-document-generator` | `bim-document-generator` |
| `bim-workflow-generator` | `bim-workflow-generator` |
| `gestion-projet` | `gestion-projet` |
| `chat` | `chat` |
| autre | `unknown` |

L'APP_ID peut aussi être forcé via le body JSON : `"app_id": "mon-app"`.

---

## Formats d'appel acceptés

### Standard (React)
```javascript
fetch('/mistral-proxy.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        messages: [{ role: 'user', content: 'question' }],
        model: 'mistral-small-latest'   // optionnel, défaut = mistral-small-latest
    })
});
```

### Avec system prompt inline
```javascript
body: JSON.stringify({
    messages: [...],
    system: 'Tu es un assistant BIM...'  // injecté automatiquement en 1er message
})
```

### Perplexity (bypass N8N)
```javascript
body: JSON.stringify({
    messages: [...],
    api: 'perplexity',
    model: 'sonar'   // optionnel, défaut = sonar
})
```

### Format inter-agents (n8n → proxy)
```javascript
body: JSON.stringify({
    chatInput: 'question',   // converti automatiquement en messages[]
    app_id: 'gid-assistant'
})
```

---

## Lecture de la réponse

```javascript
const data = await response.json();

// Le proxy peut retourner 3 formats différents selon la source
const content = data.choices?.[0]?.message?.content  // Mistral ou Groq
             || data.output                           // N8N RAG (rare)
             || 'Pas de réponse';

// Savoir quelle source a répondu (debug)
const source = data.source; // 'n8n-rag', ou absent si Mistral/Groq
```

---

## Logique de fallback

```
1. N8N RAG → si réponse non vide → retourner immédiatement (source: n8n-rag)
2. Mistral API → si 429 → retry sur clé suivante (max nb_clés - 1 fois, délai 1.2s)
3. Si Mistral toujours KO (429 ou 500) → Groq fallback
4. Groq → rotation sur 5 clés → retry si 429 (délai 0.5s)
```

**Note** : avec 1 seule clé Mistral, le retry Mistral ne sert à rien. Si 429 fréquents → ajouter une 2e clé dans `$MISTRAL_KEYS`.

---

## Diagnostic des problèmes fréquents

### Réponse vide ou null côté React
- Vérifier les 3 formats de lecture (`choices`, `output`)
- Ajouter `console.log(data)` pour voir ce qui revient
- Côté PHP : `error_log(json_encode($result))` dans le proxy

### 429 Mistral malgré plan 6 req/sec
- N8N appelle aussi Mistral en parallèle → consommation double
- Vérifier les workflows N8N actifs
- Le fallback Groq prend le relais automatiquement dans ce cas

### N8N timeout (réponses génériques sans contexte RAG)
- N8N timeout = 25s → proxy bascule sur Mistral direct sans contexte
- Symptôme : réponses correctes mais sans référence aux docs GID
- Vérifier les logs N8N + perfs Supabase

### APP_ID = 'unknown'
- Le domaine ne matche aucun pattern
- Ajouter le domaine dans le bloc de détection du proxy shared
- Ou forcer via `"app_id": "mon-app"` dans le body JSON

### Fichiers de rotation corrompus
- `.mistral_rotation_index` ou `.groq_rotation_index` dans `shared/`
- Supprimer le fichier → il se recrée automatiquement au prochain appel

---

## Déploiement nouvelle app

1. Créer le dossier de l'app sur Hostinger
2. Créer `mistral-proxy.php` :
```php
<?php
require('/home/u313130122/shared/mistral-proxy.php');
```
3. Ajouter la détection du domaine dans **shared/mistral-proxy.php** :
```php
elseif (strpos($host, 'nouvelle-app') !== false) $APP_ID = 'nouvelle-app';
```
4. Tester :
```bash
curl -X POST https://nouvelle-app.bimsmarter.eu/mistral-proxy.php \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"test"}]}'
```

---

## Coûts de référence

| Modèle | Input | Output | Rôle |
|--------|-------|--------|------|
| Mistral Small | $0.10/M | $0.30/M | Principal |
| Mistral Large | $2/M | $6/M | Éviter |
| Groq Llama 3.3 70B | $0.59/M | $0.79/M | Fallback automatique |
| Perplexity Sonar | variable | variable | Recherche web temps réel |