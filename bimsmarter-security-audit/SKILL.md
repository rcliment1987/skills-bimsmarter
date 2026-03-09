---
name: bimsmarter-security-audit
description: "ALWAYS use this skill immediately — without asking — whenever the user mentions: audit securite, security audit, verifie la securite, scan mes apps, check vulnerabilites, audit BIMSmarter, teste la securite, y a-t-il des failles, cles exposees, /security-review, /audit, securite de mon application, failles dans mon code. This skill performs a complete security audit of the BIMSmarter ecosystem (React + Firebase Firestore/Auth/Storage + Mistral API proxy PHP + n8n webhooks). Do NOT attempt to audit without this skill — it contains critical BIMSmarter-specific checks that Claude cannot perform correctly without it."
---

# Security Audit — BIMSmarter Ecosystem v2.1

Tu es un auditeur de sécurité expert React/Firebase/PHP.
Méthode : **lecture sémantique du code** (comprendre la logique) + **grep ciblé** (prouver les findings).
Chaque finding doit citer du **code réel**. Zéro faux positif.

---

## STEP 1 — Utilise orchestrating-swarms

Lance 7 agents en parallèle. Chaque agent = une surface d'attaque.

```
Agent A → Firebase Security Rules + Auth guards + race conditions
Agent B → API Keys, secrets, proxy PHP Mistral + données sensibles exposées
Agent C → XSS, injection IA, prototype pollution, validation inputs
Agent D → Logique côté client : plan/quota bypass, window globals, localStorage
Agent E → Headers HTTP sécurité (CSP, HSTS, X-Frame, SRI CDN, n8n webhooks)
Agent F → Validation serveur PHP + bonnes pratiques web générales
Agent G → Dépendances, config files, system prompt, données PII
```

---

## STEP 2 — Discovery (coordinateur)

```bash
# Structure générale
find . -name "*.html" -o -name "*.jsx" -o -name "*.tsx" -o -name "*.js" \
  | grep -v node_modules | head -30

# Fichiers critiques
find . -name "firestore.rules" -o -name "storage.rules" 2>/dev/null
find . -name "*.php" 2>/dev/null
find . -name ".env*" | grep -v node_modules 2>/dev/null
find . -name "package.json" | grep -v node_modules 2>/dev/null

# Formulaires et inputs
grep -rn "input\|form\|submit\|onChange\|handleSubmit" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -not -path "*/node_modules/*" 2>/dev/null | head -30

# Webhooks n8n
grep -rn "n8n\|webhook" --include="*.js" --include="*.jsx" --include="*.html" \
  -not -path "*/node_modules/*" 2>/dev/null | head -20
```

---

## STEP 3 — Checks par agent

### AGENT A — Firebase Auth + Guards + Race Conditions (CRITICAL)

**A1 — Firestore Security Rules**
```bash
cat firestore.rules 2>/dev/null || find . -name "firestore.rules" -exec cat {} \;
cat storage.rules 2>/dev/null || find . -name "storage.rules" -exec cat {} \;
```

Patterns CRITIQUES :
```javascript
// CRITIQUE : write sans restriction de champs → plan:'enterprise' modifiable librement
allow read, write: if request.auth != null && request.auth.uid == userId;

// CRITIQUE : tout le monde peut lire/écrire
allow read, write: if true;
```

Règle CORRECTE :
```javascript
match /users/{userId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow create: if request.auth != null && request.auth.uid == userId
                && request.resource.data.plan == 'free';
  allow update: if request.auth != null && request.auth.uid == userId
                && request.resource.data.diff(resource.data).affectedKeys()
                   .hasOnly(['displayName','lastSeen','favoriteWorkflows','aiQuota']);
  // plan, stripeCustomerId, role → écrits UNIQUEMENT via Cloud Functions Admin SDK
}
match /users/{userId}/workflows/{workflowId} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}
match /projects/{projectId} {
  allow read: if request.auth != null && request.auth.uid in resource.data.members;
  allow create: if request.auth != null;
  allow update, delete: if request.auth != null && request.auth.uid == resource.data.ownerId;
}
```

**A2 — Null guards sur fonctions Firestore**
```bash
grep -n "user\.uid\|currentUser\.uid" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Cherche fonctions async accédant à `user.uid` SANS `if(!user) return;` en tête :
`saveWorkflow()`, `deleteWorkflow()`, `createProject()`, `loadUserData()`

**A3 — Race condition quota IA**
```bash
grep -n "aiUsageCount\|quota\|runTransaction\|increment" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Pattern dangereux (non-atomique) :
```javascript
const snap = await ref.get();        // Tab A et Tab B lisent count=49 simultanément
if (snap.data().count < limit) {     // Les deux passent le check
  await ref.update({count: count+1}); // count=51 → dépassement quota
}
```
Fix : `db.runTransaction()` ou vérification côté proxy PHP.

---

### AGENT B — API Keys, Secrets, Données Sensibles Exposées (CRITICAL)

**B1 — Clés privées dans le frontend**
```bash
# Service account Firebase (VRAI danger)
grep -rn "private_key\|firebase-adminsdk\|service_account" \
  --include="*.json" -not -path "*/node_modules/*" 2>/dev/null

# Clés API privées dans le JS/HTML
grep -rn "MISTRAL_API_KEY\|ANTHROPIC_API_KEY\|sk-[A-Za-z0-9]\{20,\}" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -not -path "*/node_modules/*" 2>/dev/null

# Tokens, passwords, secrets hardcodés
grep -rn "password\s*=\|secret\s*=\|token\s*=\|Bearer\s*['\"]" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -not -path "*/node_modules/*" 2>/dev/null | grep -v "process\.env\|import\.meta\.env"

# .env commités dans Git
git ls-files | grep "\.env" 2>/dev/null
```

⚠️ NE PAS reporter : `apiKey`, `authDomain`, `projectId` Firebase dans le frontend (publics par conception).

**B2 — Données PII et métier sensibles exposées**
```bash
# Emails, noms, données utilisateur dans les logs ou commentaires
grep -rn "console\.log\|console\.info\|Logger\." \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null \
  | grep -i "email\|uid\|token\|password\|phone\|address\|nom\|prenom"

# Données sensibles en commentaires de code
grep -rn "TODO\|FIXME\|HACK\|password\|secret\|key" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null | grep -i "pass\|secret\|key\|token"
```

**B3 — Proxy PHP : authentification et secrets**
```bash
find . -name "*.php" -exec cat {} \; 2>/dev/null
```

Vérifier dans le proxy :
- Token Firebase vérifié côté PHP (`Authorization: Bearer`) ?
- CORS limité aux domaines autorisés (pas `*`) ?
- Clé Mistral en variable PHP (pas hardcodée) ?
- Rate limiting implémenté ?
- System prompt côté PHP (pas côté client) ?

Requête DANGEREUSE (sans auth) :
```javascript
fetch('/mistral-proxy.php', { headers: { 'Content-Type': 'application/json' } })
// proxy accessible par n'importe qui depuis curl
```

Fix frontend :
```javascript
const idToken = await firebase.auth().currentUser.getIdToken();
fetch('/mistral-proxy.php', {
  headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${idToken}` }
})
```

---

### AGENT C — XSS, Injection IA, Validation Inputs Côté Client (CRITICAL)

**C1 — XSS via réponses IA non assainies**
```bash
grep -n "dangerouslySetInnerHTML\|innerHTML\|cleanContent\|getCleanMessage\|\.content" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Pattern dangereux : réponses Mistral affichées sans `DOMPurify.sanitize()`.
Fix :
```javascript
import DOMPurify from 'dompurify';
const safeContent = DOMPurify.sanitize(getCleanMessage(m.content));
```

**C2 — Prototype pollution via spreads Firestore**
```bash
grep -n "\.\.\.d\.data()\|\.\.\.wf\|\.\.\.snap\b" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Fix (whitelist de champs) :
```javascript
const ALLOWED_KEYS = ['id','name','description','category','steps','tags','icon','isCustom','createdAt'];
const sanitize = (raw) => Object.fromEntries(
  Object.entries(raw).filter(([k]) => ALLOWED_KEYS.includes(k))
);
```

**C3 — Validation inputs côté client**
```bash
# Inputs sans validation
grep -n "<input\|onChange\|handleChange\|setState" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null

# Vérifier la présence de validation
grep -n "validate\|isValid\|regex\|test(\|match(\|trim()\|maxLength\|minLength\|required" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Cherche inputs sensibles SANS validation :
- Champ nom/titre de projet → pas de maxLength, pas de sanitisation
- Champ email → pas de regex validation avant envoi Firebase Auth
- Champ chat → pas de maxLength → DoS potentiel sur proxy Mistral
- Import JSON → pas de validation de schéma avant Firestore

Pattern correct pour chaque input sensible :
```javascript
// Longueur max
<input maxLength={100} ... />

// Validation avant soumission
const validate = (value) => {
  if (!value?.trim()) return 'Champ requis';
  if (value.length > 100) return 'Maximum 100 caractères';
  if (/<[^>]*>/g.test(value)) return 'Caractères HTML non autorisés';
  return null;
};
```

**C4 — Import JSON sans validation de schéma**
```bash
grep -n "importJSON\|JSON\.parse\|customWorkflows\|importedAt" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

JSON importé spreadé directement dans Firestore → XSS stocké servi à tous les membres.

---

### AGENT D — Logique Client : Plan Bypass, Globals, localStorage (CRITICAL)

**D1 — Variables globales window contournables**
```bash
grep -n "window\._\|window\[" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Pattern CRITIQUE :
```javascript
window._wfUserCache = { uid: user.uid, plan: 'free' };
// → console: window._wfUserCache.plan = 'admin' → quota illimité
```

Fix — closure privée :
```javascript
const UserPlanCache = (() => {
  let _cache = null;
  return {
    set: (uid, plan) => { _cache = Object.freeze({ uid, plan }); },
    get: () => _cache,
  };
})();
```

**D2 — Limites plan/quota côté client seulement**
```bash
grep -n "LIMITS\[plan\]\|planLimit\|aiUsageCount\|projects\.length" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Limites basées sur état React = contournables via DevTools React.
Fix : validation des limites côté Firestore Rules OU proxy PHP.

**D3 — localStorage non scopé par utilisateur**
```bash
grep -n "localStorage\." \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Clés sans suffix `_${user.uid}` → deux utilisateurs sur le même navigateur s'écrasent mutuellement.

---

### AGENT E — Headers HTTP Sécurité + SRI + Webhooks (HIGH)

**E1 — Headers HTTP de sécurité absents**
```bash
# Côté HTML (meta tags)
grep -rn "Content-Security-Policy\|X-Frame-Options\|X-Content-Type\|Strict-Transport\|Referrer-Policy\|Permissions-Policy" \
  --include="*.html" 2>/dev/null

# Côté PHP (headers envoyés)
grep -rn "header(" --include="*.php" 2>/dev/null
```

Headers REQUIS — à envoyer côté PHP (préféré) ou HTML meta :

```php
<?php
// Dans mistral-proxy.php et tout fichier PHP exposé
header("X-Frame-Options: DENY");
header("X-Content-Type-Options: nosniff");
header("Referrer-Policy: strict-origin-when-cross-origin");
header("Permissions-Policy: geolocation=(), camera=(), microphone=()");
header("Strict-Transport-Security: max-age=31536000; includeSubDomains"); // HTTPS uniquement
header("Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://unpkg.com https://www.gstatic.com https://cdnjs.cloudflare.com; connect-src 'self' https://*.googleapis.com https://*.firebaseio.com; frame-ancestors 'none';");
```

Ou dans `.htaccess` Hostinger :
```apache
Header always set X-Frame-Options "DENY"
Header always set X-Content-Type-Options "nosniff"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

**E2 — Subresource Integrity (SRI) manquant**
```bash
grep -n "<script\|<link" --include="*.html" -rn 2>/dev/null \
  | grep -v "integrity=" | grep "http"
```

Scripts CDN sans `integrity=` = CDN compromis → code malveillant exécuté.
Générer les hashes : https://www.srihash.org/

**E3 — Webhooks n8n sans authentification**
```bash
grep -rn "fetch\|axios" --include="*.js" --include="*.jsx" --include="*.html" \
  -not -path "*/node_modules/*" 2>/dev/null | grep -i "n8n\|webhook\|bimsmarter"
```

Fix :
```javascript
fetch('https://n8n.bimsmarter.eu/webhook/xxx', {
  headers: { 'X-Webhook-Secret': process.env.REACT_APP_WEBHOOK_SECRET },
  method: 'POST', body: JSON.stringify(data)
})
```

**E4 — HTTPS forcé**
```bash
grep -rn "http://" --include="*.js" --include="*.jsx" --include="*.html" \
  -not -path "*/node_modules/*" 2>/dev/null | grep -v "localhost\|127\.0\.0\|comment\|//"
```

URLs en `http://` hardcodées en production → MitM possible.

---

### AGENT F — Validation Serveur PHP + Bonnes Pratiques Web (HIGH)

**F1 — Validation des inputs côté serveur PHP**
```bash
find . -name "*.php" -exec grep -n "POST\|\$_GET\|\$_REQUEST\|json_decode\|file_get_contents" {} \; 2>/dev/null
```

Vérifier dans `mistral-proxy.php` :
- `Content-Type: application/json` vérifié avant traitement ?
- Body parsé avec gestion d'erreur (`json_decode` + vérif null) ?
- Longueur du body limitée (protection DoS) ?
- Paramètres inconnus rejetés ?
- Caractères dangereux filtrés avant log ?

Pattern DANGEREUX :
```php
$body = file_get_contents('php://input');
$data = json_decode($body, true);
$message = $data['messages']; // aucune validation, taille illimitée
```

Pattern CORRECT :
```php
// Limiter la taille de la requête
$maxSize = 50000; // 50KB max
$body = file_get_contents('php://input', false, null, 0, $maxSize);
if (strlen($body) >= $maxSize) {
    http_response_code(413);
    exit(json_encode(['error' => 'Payload too large']));
}

$data = json_decode($body, true);
if (json_last_error() !== JSON_ERROR_NONE || !isset($data['messages'])) {
    http_response_code(400);
    exit(json_encode(['error' => 'Invalid JSON']));
}

// Valider et nettoyer chaque message
$messages = array_slice($data['messages'], 0, 50); // max 50 messages
foreach ($messages as &$msg) {
    if (!in_array($msg['role'] ?? '', ['user', 'assistant'])) {
        http_response_code(400);
        exit(json_encode(['error' => 'Invalid role']));
    }
    $msg['content'] = substr(strip_tags($msg['content'] ?? ''), 0, 10000);
}
```

**F2 — Open Redirect**
```bash
grep -rn "redirect\|window\.location\|navigate(" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Pattern dangereux : `window.location = params.get('redirect')` sans validation de l'URL.

**F3 — Cookies : flags de sécurité**
```bash
grep -rn "document\.cookie\|setCookie\|Set-Cookie" \
  --include="*.js" --include="*.jsx" --include="*.html" --include="*.php" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Cookies SANS `HttpOnly; Secure; SameSite=Strict` = accessibles par XSS ou envoyés sur HTTP.

**F4 — CORS PHP ouvert à `*`**
```bash
grep -rn "Access-Control-Allow-Origin" --include="*.php" 2>/dev/null
```

```php
// DANGEREUX en production
header("Access-Control-Allow-Origin: *");

// CORRECT
$allowed = ['https://bimsmarter.eu', 'https://www.bimsmarter.eu'];
$origin = $_SERVER['HTTP_ORIGIN'] ?? '';
if (in_array($origin, $allowed)) {
    header("Access-Control-Allow-Origin: $origin");
    header("Vary: Origin");
}
```

**F5 — Clickjacking / Framing**
Vérifier que `X-Frame-Options: DENY` ou CSP `frame-ancestors 'none'` est présent (déjà couvert en E1, mais confirmer).

---

### AGENT G — Config, Dépendances, System Prompt, PII (MEDIUM)

**G1 — System prompt exposé côté client**
```bash
grep -n "SYSTEM_PROMPT\|system_prompt\|CHAT_SYSTEM_PROMPT" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

System prompt dans le JS client = visible dans le source = attaquant peut étudier et contourner les restrictions.
Fix : déplacer dans le proxy PHP.

**G2 — Dépendances vulnérables**
```bash
npm audit --json 2>/dev/null | python3 -c "
import json,sys
try:
  d=json.load(sys.stdin)
  for k,v in d.get('vulnerabilities',{}).items():
    if v.get('severity') in ['high','critical']:
      via = v.get('via',['?'])
      title = via[0] if isinstance(via[0],str) else via[0].get('title','?')
      print(f\"{v['severity'].upper()}: {k} — {title}\")
except: print('Parse error')
" 2>/dev/null || echo "npm audit non disponible"
```

**G3 — Babel Standalone en production**
```bash
grep -n "babel.min.js\|@babel/standalone" --include="*.html" -rn 2>/dev/null
```

Babel Standalone (~2.5 Mo) = compilateur JS dans le navigateur = surface d'attaque massive.
Fix long terme : migrer vers Vite.

**G4 — console.clear() en production**
```bash
grep -n "console\.clear" --include="*.js" --include="*.jsx" --include="*.html" \
  -rn 2>/dev/null
```

Masque les erreurs CSP, warnings Firebase, avertissements de sécurité.

**G5 — Données PII dans localStorage**
```bash
grep -n "localStorage\.set" \
  --include="*.js" --include="*.jsx" --include="*.html" \
  -rn -not -path "*/node_modules/*" 2>/dev/null
```

Vérifier ce qui est stocké : emails, noms, données de projet sensibles en clair = RGPD + risque XSS.

---

## STEP 4 — Règles anti-faux positifs

**NE PAS reporter :**
- `apiKey`, `authDomain`, `projectId` Firebase dans le frontend (publics par conception)
- Variables `REACT_APP_*` / `VITE_*` contenant des URLs publiques
- Vulnérabilités npm "Denial of Service" sans vecteur d'attaque réel
- `console.log` sans données sensibles
- Headers manquants si déjà configurés dans `.htaccess` ou reverse proxy

**TOUJOURS vérifier avant de reporter :**
1. Est-ce exploitable ? Décris le scénario d'attaque concret
2. Y a-t-il une autre couche de protection ?
3. Quel est l'impact réel ? (accès données, coût API, RGPD, compromission compte)
4. Confidence : HAUTE (code réel + exploitable) / MOYENNE / FAIBLE

---

## STEP 5 — Rapport Final

Génère `SECURITY_AUDIT_REPORT.md` :

```markdown
# Rapport d'Audit Sécurité — BIMSmarter [APP]
**Date :** [DATE]
**Stack :** React · Firebase · Mistral PHP Proxy · n8n
**Méthode :** bimsmarter-security-audit v2.1 (7 agents parallèles)

---

## Résumé Exécutif

| Sévérité | Nombre | Délai |
|----------|--------|-------|
| 🔴 CRITIQUE | X | < 24h |
| 🟠 ÉLEVÉ | X | < 48h |
| 🟡 MOYEN | X | < 1 sem |
| 🟢 BAS | X | Planifier |
| ✅ CONFORME | X | — |

**Score sécurité : [X]/100**
**Risque principal :** [1 phrase]

---

## Findings CRITIQUES 🔴

### [ID] — [Titre]
**Sévérité :** CRITIQUE | **Confiance :** HAUTE/MOYENNE
**Fichier :** `chemin/fichier.js:ligne`

**Problème :** [explication claire]

**Preuve — code réel :**
[EXTRAIT EXACT]

**Scénario d'attaque :**
[Comment un attaquant exploite ça, étape par étape]

**Impact :** [conséquence concrète]

**Correction :**
[CODE CORRIGÉ COMPLET]

**Effort :** ~X min/h

---

## Findings ÉLEVÉS 🟠
[même structure]

## Findings MOYENS 🟡
[structure allégée — Problème + Correction + Effort]

## Findings Bas 🟢
[liste avec lien fichier:ligne + fix en une ligne]

## Points Conformes ✅
| # | Check | Statut |
|---|-------|--------|

---

## Plan d'Action Priorisé

| # | Action | Sévérité | Effort | Fichier |
|---|--------|----------|--------|---------|
| 1 | ... | 🔴 | 30 min | ... |

---
*bimsmarter-security-audit v2.1 — [DATE]*
```

---

## Référence rapide — Checks complets BIMSmarter

| Check | Agent | Sévérité | App concernée |
|-------|-------|----------|---------------|
| Firestore rules : champ `plan` modifiable | A | 🔴 CRITIQUE | Toutes |
| Proxy Mistral sans auth token Firebase | B | 🔴 CRITIQUE | Toutes |
| Race condition quota IA (non-atomique) | A | 🔴 CRITIQUE | WF Generator, GID |
| window._wfUserCache contournable console | D | 🔴 CRITIQUE | WF Generator |
| Null guards manquants fonctions Firestore | A | 🔴 CRITIQUE | Toutes |
| XSS via réponses IA non sanitisées | C | 🔴 CRITIQUE | GID, WF Generator |
| Prototype pollution spreads Firestore | C | 🔴 CRITIQUE | Toutes |
| Inputs formulaire sans validation côté client | C | 🟠 ÉLEVÉ | Toutes |
| Validation inputs absente côté PHP | F | 🟠 ÉLEVÉ | Toutes |
| Headers HTTP sécurité absents (CSP, X-Frame, HSTS...) | E | 🟠 ÉLEVÉ | Toutes |
| Scripts CDN sans SRI | E | 🟠 ÉLEVÉ | Toutes |
| CORS PHP ouvert à `*` | F | 🟠 ÉLEVÉ | Toutes |
| Import JSON sans validation schéma | C | 🟠 ÉLEVÉ | WF Generator |
| localStorage non scopé par uid | D | 🟠 ÉLEVÉ | WF Generator |
| Emails/PII loggués en clair | B | 🟠 ÉLEVÉ | Toutes |
| Webhooks n8n sans secret header | E | 🟠 ÉLEVÉ | Toutes |
| Données sensibles PII dans localStorage | G | 🟡 MOYEN | Toutes |
| System prompt exposé côté client | G | 🟡 MOYEN | GID, WF Generator |
| Cookies sans HttpOnly/Secure/SameSite | F | 🟡 MOYEN | Toutes |
| Open redirect non validé | F | 🟡 MOYEN | Toutes |
| URLs hardcodées en http:// | E | 🟡 MOYEN | Toutes |
| console.clear() en production | G | 🟡 MOYEN | WF Generator |
| Input chat sans maxLength (DoS proxy) | C | 🟡 MOYEN | GID, WF Generator |
| Babel Standalone en production | G | 🟡 MOYEN | WF Generator |
| Clés API privées dans le code | B | 🔴 CRITIQUE | Toutes |
| .env commités dans Git | B | 🔴 CRITIQUE | Toutes |
