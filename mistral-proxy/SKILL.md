---
name: mistral-proxy
description: Architecture et maintenance du proxy PHP BIMSmarter qui centralise les appels Mistral API pour toutes les apps. Utilise ce skill quand l'utilisateur parle de son proxy PHP, d'erreurs 429 résiduelles, de timeout N8N, de déploiement du proxy sur une nouvelle app, ou d'optimisation des coûts tokens Mistral. Aussi pertinent pour déboguer les réponses vides, les erreurs de format JSON, ou les problèmes de détection d'app. Si l'utilisateur mentionne "mistral-proxy.php", "proxy", "l'IA ne répond plus", utilise ce skill.
---

# Mistral Proxy — BIMSmarter

Tu connais l'architecture du proxy PHP BIMSmarter hébergé sur Hostinger. Plan actuel : **Mistral payant, 6 req/sec** — le rate limiting n'est plus le problème principal.

---

## Architecture

```
Apps React
    ↓ POST JSON
[app]/mistral-proxy.php   ← juste require('/home/u313130122/shared/mistral-proxy.php')
    ↓
/home/u313130122/shared/mistral-proxy.php   ← fichier UNIQUE pour toutes les apps
    ↓ 1. Appel N8N RAG webhook (si USE_N8N = true, timeout 25s)
    ↓ 2. Appel Mistral API
    ↓ Réponse JSON format OpenAI-compatible
Apps React
```

**Règle critique** : modifier le proxy = toujours éditer `/home/u313130122/shared/mistral-proxy.php`. Les fichiers PHP des apps ne contiennent que le `require`, rien d'autre. Déployer une nouvelle app = créer un `index.php` avec juste le `require`.

---

## Configuration de référence

```php
// CLÉ API (une seule suffit sur plan payant 6 req/sec)
$MISTRAL_KEY = 'ta_clé_mistral';

// MODÈLE — Small = rapport qualité/coût optimal pour BIM Luxembourg
$MISTRAL_MODEL = 'mistral-small-latest';

// N8N RAG
$N8N_URL = 'https://n8n.bimsmarter.eu/webhook/rag-apps';
$USE_N8N = true;

// Détection app via domaine
$host = $_SERVER['HTTP_HOST'];
if (strpos($host, 'gid-assistant') !== false)              $APP_ID = 'gid-assistant';
elseif (strpos($host, 'bim-document-generator') !== false) $APP_ID = 'bim-document-generator';
elseif (strpos($host, 'bim-workflow-generator') !== false) $APP_ID = 'bim-workflow-generator';
else                                                        $APP_ID = 'unknown';
```

---

## Appel depuis les apps React

```javascript
const response = await fetch('/mistral-proxy.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        messages: messages.map(m => ({ role: m.role, content: m.content })),
        model: 'mistral-small-latest'
    })
});

const data = await response.json();
// Le proxy peut retourner soit le format N8N, soit le format Mistral standard
const content = data.choices?.[0]?.message?.content 
             || data.output 
             || 'Pas de réponse';
```

---

## Diagnostic des problèmes fréquents

### Réponse vide ou null
- Vérifier `data.choices?.[0]?.message?.content` vs `data.output`
- Si RAG N8N a répondu → format différent du format Mistral standard
- Ajouter `error_log(json_encode($data))` dans le PHP pour voir ce qui sort

### 429 résiduel malgré plan payant
- Cause probable : N8N appelle Mistral **en parallèle** pendant que le proxy appelle aussi
- N8N + proxy = 2 appels simultanés sur la même clé
- Solution : vérifier les workflows N8N, ou espacer avec `usleep(200000)` avant l'appel Mistral

### Timeout N8N
- N8N configuré 25s — si Supabase RAG est lent, N8N timeout
- Le proxy bascule alors sur Mistral direct sans contexte RAG
- Symptôme : réponses génériques sans référence aux docs GID
- Vérifier les logs N8N si ce comportement est observé

### L'app ne détecte pas le bon APP_ID
- Vérifier que le domaine Hostinger contient bien le mot-clé (`gid-assistant`, etc.)
- Ajouter `error_log("APP_ID: " . $APP_ID)` pour debugger

### Permissions fichiers PHP
- Le dossier doit être writable pour les fichiers de log éventuels
- chmod 755 sur le dossier, 644 sur les fichiers PHP

---

## Coûts de référence

| Modèle | Input | Output | Recommandation |
|--------|-------|--------|----------------|
| Mistral Small | $0.10/M | $0.30/M | ✅ Principal — optimal BIM |
| Mistral Large | $2/M | $6/M | ❌ Éviter sauf besoin spécifique |
| Groq Llama 3.3 | $0.59/M | $0.79/M | Fallback si Mistral down |

---

## Checklist déploiement nouvelle app

1. Créer le dossier de l'app sur Hostinger
2. Créer `.php` avec juste :
```php
<?php
require('/home/u313130122/shared/mistral-proxy.php');
```
3. Ajouter la détection du domaine dans **shared/mistral-proxy.php** :
```php
elseif (strpos($host, 'nouvelle-app') !== false) $APP_ID = 'nouvelle-app';
```
4. Tester avec curl :
```bash
curl -X POST https://nouvelle-app.bimsmarter.eu/mistral-proxy.php \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"test"}],"model":"mistral-small-latest"}'
```
5. Vérifier que la réponse a bien `choices[0].message.content`
