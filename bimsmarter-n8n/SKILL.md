---
name: bimsmarter-n8n
description: Debug et conception de workflows n8n BIMSmarter. Utilise ce skill quand l'utilisateur parle de workflows n8n, de webhooks HTTP entre apps, de RAG Supabase via n8n, de format JSON pour les agents, ou d'automatisation inter-apps. Si l'utilisateur mentionne "n8n", "workflow", "webhook", "agent n8n", "rag-apps", utilise ce skill.
---

# n8n BIMSmarter — Debug & Conception

Instance : `https://n8n.bimsmarter.eu`

---

## Architecture inter-apps (actuelle)

```
Apps React
    ↓ POST { app_id, chatInput, session_id }
shared/mistral-proxy.php
    ↓ POST (timeout 25s)
n8n webhook /rag-apps
    ├─ Supabase RAG (vector search)
    ├─ Mistral API (reformulation/réponse)
    └─ Retour JSON { output: "..." }
        ↓
mistral-proxy.php → App React
```

**Objectif futur** : webhooks directs App ↔ App (Document Generator ↔ GID-Assistant ↔ Workflow Generator)

---

## Format des webhooks

### Entrée proxy → n8n (`/webhook/rag-apps`)
```json
{
  "app_id": "gid-assistant",
  "chatInput": "Question de l'utilisateur",
  "session_id": "gid-assistant_abc123"
}
```

### Sortie n8n → proxy (format attendu)
```json
[{ "output": "Réponse de l'agent" }]
```
ou
```json
{ "output": "Réponse de l'agent" }
```

Le proxy gère les deux formats : `n8n_data[0]['output'] ?? n8n_data['output']`

### Format inter-agents (app → app via n8n)
```json
{
  "chatInput": "Question ou données",
  "app_id": "app-source",
  "session_id": "optionnel"
}
```

---

## Structure d'un workflow RAG type

```
[Webhook] → [Set app_id] → [Supabase Vector Search]
                                    ↓
                         [Code: formater contexte]
                                    ↓
                         [Mistral: réponse avec contexte]
                                    ↓
                         [Respond to Webhook: { output }]
```

### Node Supabase Vector Search — config type
```
Table : documents (ou équivalent)
Query : {{ $json.chatInput }}
Filter : metadata->>'app_id' = '{{ $json.app_id }}'
Limit : 5
```

### Node Code — formater le contexte RAG
```javascript
const docs = $input.all();
const context = docs
  .map(d => d.json.content || d.json.text)
  .filter(Boolean)
  .join('\n\n---\n\n');

return [{ json: { context, question: $('Webhook').item.json.chatInput } }];
```

### Node Mistral dans n8n — body type
```json
{
  "model": "mistral-small-latest",
  "messages": [
    { "role": "system", "content": "Tu es un assistant BIM. Contexte disponible :\n\n{{ $json.context }}" },
    { "role": "user", "content": "{{ $json.question }}" }
  ]
}
```

---

## Diagnostic des problèmes fréquents

### n8n ne reçoit pas le webhook
- Vérifier que le workflow est **actif** (toggle ON dans n8n)
- URL exacte : `https://n8n.bimsmarter.eu/webhook/rag-apps` (pas `/webhook-test/`)
- Tester avec curl :
```bash
curl -X POST https://n8n.bimsmarter.eu/webhook/rag-apps \
  -H "Content-Type: application/json" \
  -d '{"app_id":"test","chatInput":"test question","session_id":"test_001"}'
```

### n8n répond mais proxy retourne réponse générique (sans RAG)
- Le workflow a dépassé 25s → proxy a abandonné et appelé Mistral directement
- Vérifier le nœud le plus lent (souvent Supabase vector search)
- Solution court terme : augmenter le timeout dans le proxy
- Solution long terme : optimiser les index Supabase

### Format de sortie incorrect (proxy ne lit pas la réponse)
- n8n doit retourner exactement `[{"output":"..."}]` ou `{"output":"..."}`
- Dans le nœud "Respond to Webhook" :
```json
{ "output": "{{ $json.text }}" }
```
- Ne pas retourner le format Mistral natif (`choices[0].message.content`) depuis n8n → le proxy ne le lira pas

### Session_id non persistant entre messages
- Le `session_id` est `app_id + '_' + session_id()` PHP
- La session PHP expire → nouveau session_id → n8n perd la mémoire conversationnelle
- Fix : générer un UUID côté React et l'envoyer dans chaque requête

---

## Webhooks inter-apps (à implémenter)

Objectif : Document Generator → GID-Assistant pour enrichir les docs avec données GID.

### Pattern à suivre
```
[App Source] → POST /webhook/doc-to-gid → [n8n workflow]
                                              ↓
                                    [Appel GID-Assistant API]
                                              ↓
                                    [Retour données GID enrichies]
                                              ↓
                              [App Source reçoit la réponse]
```

### Format de déclenchement depuis React
```javascript
const enrichWithGID = async (projectData) => {
  const response = await fetch('https://n8n.bimsmarter.eu/webhook/doc-to-gid', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      source_app: 'bim-document-generator',
      phase: projectData.phase,        // APS, APD, PDE, EXE, EXP
      project_type: projectData.type
    })
  });
  return response.json();
};
```

---

## Bonnes pratiques n8n BIMSmarter

- **Toujours tester avec `/webhook-test/`** avant d'activer en prod
- **Timeout n8n** : configurer à 30s max côté n8n pour rester sous le timeout proxy (25s)
- **Logs** : activer "Save execution data" sur les workflows en prod pour débugger
- **Erreurs** : ajouter un nœud "Error Trigger" qui log vers un webhook de monitoring
- **app_id obligatoire** : toujours filtrer le RAG par app_id pour éviter de mélanger les données GID avec les données d'autres apps
