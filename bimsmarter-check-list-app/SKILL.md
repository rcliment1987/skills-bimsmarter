---
name: bimsmarter-check-list-app
description: Checklist complète pour créer une nouvelle application BIMSmarter conforme à l'écosystème (React 18 no-build + Firebase Auth/Firestore/AppCheck + Proxy PHP Mistral + Tailwind + Hostinger). UTILISE CE SKILL immédiatement — sans demander — dès que Renaud mentionne : "nouvelle app", "créer une app", "bootstrapper", "squelette d'app", "template d'app", "structure type", "checklist app", "nouvelle page BIMSmarter", "ajouter une app", "IFC Viewer", "gestion projet app", "comment structurer mon app", "même structure que les autres", "copier la base de Document Generator", ou toute demande liée à la création d'une nouvelle application dans l'écosystème BIMSmarter. Ce skill encode toute l'architecture de référence : auth, plans, quotas IA, proxy PHP, sécurité, Firestore, exports, UI, anti-patterns React, et procédure de déploiement.
---

# Checklist — Nouvelle App BIMSmarter

Blueprint complet pour créer une application conforme à l'écosystème BIMSmarter.
Basé sur le BIM Document Generator (score sécurité 86/100) et la roadmap Go-Live 1er avril 2026.

**Légende :**
- 🔴 **Bloquant** — L'app ne peut pas être déployée sans ça
- 🟡 **Important** — À faire avant le Go-Live
- 🟢 **Recommandé** — Peut être fait post-lancement

---

## PHASE 1 — Squelette technique

### 1.1 Structure fichiers

```
/public_html (sous-domaine.bimsmarter.eu)
├── index.html                    # App React complète (monolithe no-build)
├── mistral-proxy-local.php       # Proxy IA local (auth + quota + system prompt)
├── .htaccess                     # Sécurité Apache (copier depuis BIM Doc Generator)
├── BIM_Smarter_faviconV2.jpg     # Favicon commun
└── README.md                     # Documentation (bloqué par .htaccess)
```

- 🔴 Créer le sous-domaine sur Hostinger (DNS + certificat SSL)
- 🔴 Copier le `.htaccess` de référence (BIM Document Generator) — contient headers sécu, CSP, HSTS, blocage fichiers sensibles
- 🔴 Adapter la CSP dans `.htaccess` si besoin de domaines supplémentaires dans `connect-src`
- 🟡 Favicon BIMSmarter en place
- 🟡 `README.md` créé (bloqué par `.htaccess`)

### 1.2 Stack CDN (dans le `<head>`)

Toutes les librairies sont **pinnées en version avec SRI** (`integrity` + `crossorigin="anonymous"`), sauf Tailwind CDN (pas de SRI possible — migration Vite planifiée).

**Scripts obligatoires (copier les hash SRI exacts depuis BIM Document Generator) :**
- Tailwind CSS CDN (sans SRI — exception documentée)
- Firebase SDK 10.7.1 — 4 scripts avec SRI (app, auth, firestore, app-check)
- React 18.2.0 + ReactDOM — avec SRI
- Babel Standalone 7.23.9 — avec SRI
- Librairies spécifiques à l'app (jsPDF, SheetJS, Three.js, etc.) — avec SRI

- 🔴 Ne JAMAIS charger un CDN sans SRI (sauf Tailwind)
- 🔴 Copier les hash SRI exacts depuis BIM Document Generator

### 1.3 Initialisation Firebase

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyBzghrSLYAjcKYskfkrc7dbSX3DKm_rIEM",
  authDomain: "auth.bimsmarter.eu",
  projectId: "bimsmarter",   // ⚠️ PAS "bimsmarter-2af81"
  storageBucket: "bimsmarter.firebasestorage.app",
  messagingSenderId: "670432116296",
  appId: "1:670432116296:web:38bac2f70ec706e914e27d",
  measurementId: "G-137BHQR0ED"
};

firebase.initializeApp(firebaseConfig);
firebase.appCheck().activate('6LdeP40sAAAAADr7ORpn0oHkh_Zt7CoUBg4bKXz3', true);
const auth = firebase.auth();
const db = firebase.firestore();
```

- 🔴 Config Firebase identique sur toutes les apps (projet `bimsmarter`)
- 🔴 App Check activé avec reCAPTCHA v3
- 🔴 `authDomain` pointe sur `auth.bimsmarter.eu`

---

## PHASE 2 — Authentification & Gestion utilisateurs

### 2.1 LoginPage

Copier le composant `LoginPage` du BIM Document Generator et adapter le titre/logo.

- 🔴 Connexion email/password (`signInWithEmailAndPassword`)
- 🔴 Inscription (`createUserWithEmailAndPassword`) + envoi email vérification
- 🔴 Vérification `emailVerified` dans `onAuthStateChanged` — bloquer si non vérifié
- 🔴 Reset password avec cooldown 60s (N08)
- 🔴 Politique mot de passe : min 10 chars + majuscule + minuscule + chiffre + spécial (N07)
- 🔴 `autoComplete` sur les inputs password (L03)
- 🔴 Messages d'erreur génériques — jamais `err.message` directement (N06)
- 🔴 `switch(err.code)` pour les cas connus

**⚠️ Ne JAMAIS mettre `auth.signOut()` dans `onAuthStateChanged`** — race condition avec `createUser`.

### 2.2 Profil utilisateur Firestore

À la première connexion, créer `users/{uid}` :

```javascript
await db.collection('users').doc(user.uid).set({
  email: user.email,
  displayName: user.displayName || user.email,
  lastSeen: new Date().toISOString(),
  plan: "free",
  planSince: new Date().toISOString(),
  stripeCustomerId: "",
  aiQuota: { date: new Date().toISOString().split('T')[0], count: 0 }
});
```

- 🔴 Champs obligatoires : `email`, `displayName`, `lastSeen`, `plan`, `planSince`, `stripeCustomerId`, `aiQuota`
- 🔴 `plan` initialisé à `"free"` — JAMAIS modifiable côté client (protégé par Firestore Rules)
- 🔴 `aiQuota` initialisé à `{ date: "YYYY-MM-DD", count: 0 }`
- 🟡 Mise à jour `lastSeen` à chaque connexion
- 🟡 Listener `onSnapshot` sur `users/{uid}` pour réactivité temps réel

### 2.3 Affichage plan dans le header

- 🟡 Badge plan visible : Free (gris), Pro (ambre), Enterprise (violet), Admin (violet)
- 🟡 Couleurs cohérentes avec les autres apps

---

## PHASE 3 — Système de plans & monétisation

### 3.1 Limites par plan

```javascript
const DOC_LIMITS = { free: 100, pro: 500, enterprise: 2000, admin: 10000 };
const AI_LIMITS = { free: 20, pro: 100, enterprise: 500, admin: 9999 };
const PROJECT_LIMITS = { free: 1, pro: 5, enterprise: 999, admin: 999 };
```

- 🔴 `AI_LIMITS` identique sur toutes les apps (quota partagé dans Firestore)
- 🔴 Vérification limite AVANT ajout (côté client = UX guard)
- 🟡 Toast explicite quand limite atteinte

### 3.2 Gating des fonctionnalités

```javascript
const isFree = !userData?.plan || userData.plan === 'free';
const proGate = (fn) => () => {
  if (isFree) { showToast('Fonctionnalité réservée aux plans Pro et supérieurs'); return; }
  fn();
};
```

- 🔴 Exports avancés (CSV, Excel, JSON) gatés Pro — PDF toujours accessible
- 🔴 Boutons grisés : `opacity-40` + `cursor-not-allowed`
- 🔴 Import CSV gaté Pro

---

## PHASE 4 — Proxy IA & Chatbot

### 4.1 Proxy PHP local (`mistral-proxy-local.php`)

Copier depuis BIM Document Generator et adapter **uniquement le `$systemPrompt`**.

Architecture :
1. Reçoit `POST` avec `messages[]` + `Authorization: Bearer {idToken}`
2. Valide le token Firebase via Google Identity Toolkit
3. Lit le plan/quota dans Firestore via REST API
4. Vérifie le quota (HTTP 429 si dépassé)
5. Incrémente le quota via PATCH Firestore REST API
6. Filtre les messages — rejette tout rôle `system`
7. Injecte le system prompt côté serveur
8. Transmet au proxy partagé (`shared/mistral-proxy.php`)
9. Retourne la réponse + quota mis à jour

**⚠️ IMPORTANT :** dans le proxy PHP, utiliser `$projectId = 'bimsmarter';` (PAS `bimsmarter-2af81`).

- 🔴 Token Firebase validé côté serveur (C01)
- 🔴 Quota lu et incrémenté côté serveur (C04)
- 🔴 System prompt injecté côté serveur uniquement (C05)
- 🔴 Rôle `system` filtré côté serveur
- 🔴 Historique limité aux 20 derniers messages côté serveur
- 🔴 Contenu des messages tronqué à 10 000 chars (`substr`)
- 🔴 CORS : `$allowed_origins` inclut le sous-domaine de l'app

### 4.2 Chatbot frontend

- 🔴 Vérification quota côté client AVANT envoi (UX guard — serveur = autorité)
- 🔴 Barre de décompte quota en temps réel dans le header chatbot
- 🔴 `<textarea>` avec Shift+Entrée (multiligne), Entrée (envoyer)
- 🔴 `maxLength={2000}` sur l'input
- 🔴 Historique envoyé limité aux 10 derniers messages (M02)
- 🔴 Message d'erreur générique en cas d'échec
- 🔴 Gestion HTTP 429 → toast "Quota atteint"
- 🟡 Bouton FAB (🤖) avec animation pulse — style commun
- 🟡 Bulle promo auto-hide après 10 secondes
- 🟡 Chat redimensionnable
- 🟡 Mise à jour locale du compteur quota depuis `data.quota`

---

## PHASE 5 — Persistance Firestore

### 5.1 Pattern de sauvegarde

```javascript
useEffect(() => {
  if (loading) return;
  const saveTimeout = setTimeout(async () => {
    setSaving(true);
    try {
      await db.collection('users').doc(user.uid).set({
        /* données spécifiques à l'app */
        updatedAt: firebase.firestore.FieldValue.serverTimestamp()
      }, { merge: true });
    } catch (err) {
      console.error('[Save]', err.code || err.message);
    }
    setSaving(false);
  }, 1000);
  return () => clearTimeout(saveTimeout);
}, [/* dépendances */, user.uid, loading]);
```

- 🔴 Debounce 1000ms
- 🔴 `{ merge: true }` pour ne pas écraser les autres champs
- 🔴 `updatedAt` avec `serverTimestamp()`
- 🔴 Indicateur visuel "Sauvegarde..." / "Synchronisé"
- 🔴 `console.error` limité à `err.code || err.message`

### 5.2 Chargement initial

- 🔴 `onSnapshot` pour réactivité temps réel
- 🔴 `safeSpread()` sur TOUTES les données Firestore (M04 — anti prototype-pollution)
- 🔴 Gestion du cas `!doc.exists` (premier login → créer profil)
- 🔴 Flag `initialized` pour ne charger qu'une fois

### 5.3 Gestion de projets (collection partagée)

- 🟡 Sélecteur de projet dans le header
- 🟡 Templates de projet (Résidentiel, Tertiaire, Infrastructure, Rénovation)
- 🟡 Limite projets par plan
- 🟡 `localStorage` scopé par UID (`bimsmarter_currentProject_{uid}`)

---

## PHASE 6 — Sécurité (viser 86/100)

### 6.1 Constantes de sécurité (module-level)

```javascript
const _PROTO_BLOCKED = new Set(['__proto__', 'constructor', 'prototype']);
const safeSpread = (obj) =>
  Object.fromEntries(Object.entries(obj || {}).filter(([k]) => !_PROTO_BLOCKED.has(k)));
```

- 🔴 `safeSpread()` défini au niveau module
- 🔴 `AI_LIMITS`, `DOC_LIMITS` au niveau module (jamais recréés dans le render)

### 6.2 Validation des inputs

- 🔴 `maxLength` sur TOUS les inputs critiques (chat: 2000, projet: 100)
- 🟡 Si import CSV : validation taille (2 Mo), lignes (1000), champs sanitizés
- 🟡 Si documents IA : `sanitizeAIDoc()` + regex strictes + limite 20 docs

### 6.3 Gestion des erreurs (anti info-leak)

```javascript
// ❌ JAMAIS
catch (err) { alert('Erreur: ' + err.message); }
catch (err) { console.error(err); }

// ✅ TOUJOURS
catch (err) {
  console.error('[Context]', err.code || err.message);
  alert('Une erreur est survenue. Réessayez.');
}
```

- 🔴 Tous les `console.error` → `err.code || err.message`
- 🔴 Messages user-facing génériques
- 🔴 Pas de `console.warn(err)` avec objet complet
- 🔴 Pas de `alert('Erreur: ' + err.message)`

### 6.4 `.htaccess`

- 🔴 HTTPS (redirect 301)
- 🔴 Headers : `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, HSTS, Referrer-Policy, Permissions-Policy
- 🔴 CSP configurée
- 🔴 Blocage : `.env`, `*.log`, `firestore.rules`, `*.md`, `.gitignore`
- 🔴 Anti-hotlinking images

### 6.5 Checklist OWASP rapide

- 🔴 Pas de `dangerouslySetInnerHTML`
- 🔴 Pas de clés privées dans le code
- 🔴 Proxy PHP valide le token Firebase
- 🔴 SRI sur tous les scripts CDN (sauf Tailwind)
- 🔴 `emailVerified` vérifié dans `onAuthStateChanged`

---

## PHASE 7 — UI & Charte graphique

### 7.1 Style commun

```
Background: gradient from-slate-950 via-slate-900 to-slate-950
Glass:      rgba(30, 41, 59, 0.7) + backdrop-filter blur(12px) + border rgba(6, 182, 212, 0.2)
Cyan:       #06b6d4 (primaire)
Slate-900:  #0f172a (fonds)
Purple:     #a855f7 (accents IA)
Fonts:      Rubik (titres), Inter (corps)
```

- 🟡 Dark theme identique sur toutes les apps
- 🟡 Cross-pattern cyan en background (`opacity-5`)
- 🟡 Header : logo + titre app + badge plan + logout
- 🟡 Footer : copyright BIMSmarter + ISO 19650
- 🟡 Chatbot : même bouton FAB, même bulle promo

### 7.2 Composants réutilisables

- 🟡 `CustomSelect` (dropdown avec recherche)
- 🟡 Toast : `showToast(message)` timeout 3s, z-index `99999`
- 🟡 Modals : `bg-black/70 backdrop-blur-sm` + `bg-slate-900 rounded-2xl`

---

## PHASE 8 — Exports

### 8.1 Exports PDF (tous plans)

```javascript
// Bandeau commun
doc.setFillColor(15, 23, 42); // #0f172a
doc.rect(0, 0, width, 30, 'F');
doc.setTextColor(11, 188, 217); // Cyan
doc.text('BIMSmarter', 14, 14);
doc.setTextColor(148, 163, 184); // Gris
doc.text('Nom App - ISO 19650', 14, 23);

// Footer
doc.text(`BIMSmarter [App] | ${email} | Page ${i}/${total}`, center, bottom, { align: 'center' });
```

- 🟡 Bandeau `#0f172a` + titre cyan + sous-titre gris
- 🟡 Bloc info : nom projet, généré par, date
- 🟡 Footer paginé

### 8.2 Exports gatés (Pro+)

- 🔴 CSV/Excel/JSON → vérifier `isFree`
- 🔴 Boutons grisés (`opacity-40`) pour Free
- 🔴 Toast au clic
- 🔴 PDF toujours accessible

---

## PHASE 9 — Anti-patterns à éviter

### ❌ AP1 — Composant redéfini dans le render parent
→ Cause perte de focus sur les inputs.
```javascript
// ❌ const Child = () => <input />;  dans le Parent
// ✅ Définir Child en dehors du composant Parent
```

### ❌ AP2 — Toast JSX après le `return()`
→ Écran blanc silencieux.
```javascript
// ✅ Toast DANS le return, juste avant la dernière </div>
```

### ❌ AP3 — Deux éléments JSX dans un ternaire sans wrapper
→ Erreur Babel "Unexpected token" → écran blanc.
```javascript
// ✅ Wrapper dans <React.Fragment>
```

### ❌ AP4 — Apostrophes dans les attributs JSX
```javascript
// ❌ title="aujourd'hui"
// ✅ title="aujourd&apos;hui"
```

### ❌ Autres pièges

- Ne jamais supprimer la vérification quota client (casse l'UX)
- Toast z-index : toujours `99999` (au-dessus du chatbot à `9999`)
- `localStorage` toujours scopé par UID
- Ne jamais `auth.signOut()` dans `onAuthStateChanged`

---

## PHASE 10 — Tests pre-déploiement

### Fonctionnels
- Inscription → email vérif → connexion
- Reset password → email reçu → nouveau mdp OK
- Profil Firestore créé avec tous les champs
- Chatbot → réponse + quota incrémenté
- Quota atteint → toast + blocage
- Export PDF (Free) / CSV bloqué (Free) → toast
- Déconnexion → retour login

### Cross-apps
- Quota IA cumulé entre les apps dans Firestore
- Upgrade via bimsmarter.eu/pricing → `pro` visible partout
- Downgrade → exports re-grisés, projets readonly

### Sécurité
- Navigation privée OK
- Console : pas d'objet erreur complet
- Token expiré → proxy 401
- Body > 50KB → proxy 413
- Rôle `system` filtré par proxy

### Responsive
- Mobile : chatbot, filtres, modals
- Tablet : layout adapté
- Desktop : layout complet

---

## PHASE 11 — Déploiement

### Procédure

1. Upload `index.html`, `mistral-proxy-local.php`, `.htaccess` via FTP/File Manager Hostinger
2. Vérifier `MISTRAL_PROXY_PATH` → proxy partagé
3. Ajouter le sous-domaine dans CORS du proxy (`$allowed_origins`)
4. Tester en navigation privée
5. Vérifier les logs PHP sur Hostinger
6. Lancer un audit sécurité (`bimsmarter-security-audit` skill)

### Post-déploiement

- Ajouter l'app à la page Notion roadmap
- Mettre à jour le README
- Journal de session (`.docx`) pour traçabilité

---

## Résumé : fichiers à copier depuis BIM Document Generator

| Fichier | Adapter | Garder tel quel |
|---------|---------|-----------------|
| `.htaccess` | CSP (`connect-src`) si nouveaux domaines | Tout le reste |
| `mistral-proxy-local.php` | `$systemPrompt`, `$allowed_origins` | Auth, quota, filtrage rôles |
| `index.html` (`<head>`) | Titre, librairies spécifiques | CDN avec SRI, Firebase config |
| `index.html` (LoginPage) | Titre de l'app | Tout le reste |
| `index.html` (constantes) | `DOC_LIMITS`, templates | `AI_LIMITS`, `safeSpread` |
| `index.html` (header) | Nom app, onglets | Style, badge plan, logout |
| `index.html` (chatbot) | System prompt (côté serveur) | UI chatbot complète |
