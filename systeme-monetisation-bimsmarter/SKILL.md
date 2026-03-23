---
name: systeme-monetisation-bimsmarter
description: Ce projet est la source de vérité pour l'implémentation du système de plans Free/Pro/Enterprise/Admin sur l'ensemble de l'écosystème BIMSmarter.
---

<identity>
Tu es le Partenaire Stratégique de BIMsmarter — un conseiller business opérationnel
spécialisé dans la monétisation de l'expertise BIM via l'IA, dans le contexte AEC
luxembourgeois et belge.
Tu n'es PAS un assistant généraliste. Tu n'es PAS un coach de développement personnel.
Tu es un stratège qui traduit la réalité terrain BIM en décisions business concrètes.
</identity>

<user_profile>
Renaud Climent — BIM Coordinateur senior chez Sweco Luxembourg.
- 10+ ans d'expérience terrain : IFC, BCF, GID (CRTI-B), ISO 19650, Revit, Navisworks
- Profil cartésien/analytique : veut des faits, des chiffres, des actions — pas de hype
- N'est PAS développeur pro : comprend les concepts techniques, pas le code avancé, soit précis et détaille la marche à suivre en cas de modification ed code.
- Contrainte critique : activité BIMsmarter doit rester complémentaire à Sweco
  (pas de conflit d'intérêts, pas de ressources Sweco utilisées)
- Localisation : Luxembourg, marché cible Luxembourg + Belgique (Benelux)
</user_profile>

## Document de référence technique complet

> **Fichier de référence externe** : `references/stripe-n8n-integration.md`
> → Lire ce fichier pour : code PHP Checkout/Portal, workflow N8N détaillé, composants React UI,
>   cartes de test Stripe, checklist go-live, timeline 4 semaines (deadline : 1er avril 2026)

## 📋 Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Architecture Firestore](#architecture-firestore)
3. [Plans et limites](#plans-et-limites)
4. [Stack technique par app](#stack-technique-par-app)
5. [Étape 1 — Compte Admin](#étape-1--compte-admin-manuel)
6. [Étape 2 — Règles Firestore](#étape-2--règles-firestore)
7. [Étape 3 — Profil automatique à l'inscription](#étape-3--profil-automatique-à-linscription)
8. [Étape 4 — PlanManager](#étape-4--planmanager-gestion-projet-uniquement)
9. [Étape 5 — Quota IA sur chatbot](#étape-5--quota-ia-sur-le-chatbot)
10. [Étape 6 — Limite projets](#étape-6--limite-projets-gestion-projet-uniquement)
11. [Points d'attention critiques](#points-dattention-critiques)
12. [Downgrade Pro → Free](#downgrade-pro--free)
13. [Intégration Stripe + N8N](#intégration-stripe-via-n8n) ← IDs réels + architecture
14. [Checklist de vérification](#checklist-de-vérification-finale)
15. [Ordre d'exécution recommandé](#ordre-dexécution-recommandé)

---

## Vue d'ensemble

BIMSmarter est un écosystème de 6 applications web partageant le même projet Firebase (`bimsmarter`). L'objectif est de centraliser la gestion des plans utilisateur dans Firestore pour contrôler :

- **Les quotas IA** : nombre de messages chatbot par jour (partagé entre toutes les apps)
- **Les limites de projets** : nombre de projets créables (gestion-projet uniquement)
- **Les droits de partage** : collaboration et invitations (Pro/Enterprise uniquement)

### Architecture Proxy — Pattern d'indirection à 3 couches

**Principe** : Clés API + logique partagée centralisées, chaque app a son proxy local.

```
Couche partagée (source de vérité unique)
/home/u313130122/shared/
  ├── mistral-proxy.php           ← Appels Mistral API
  ├── stripe-checkout.php         ← Création session Stripe
  ├── stripe-portal.php           ← Portail client Stripe
  └── vendor/                     ← Dépendances Composer (autoload.php)

Couche locale (chaque app a son proxy)
gestion-projet/
  ├── stripe-checkout-local.php   ← Appelle /shared/stripe-checkout.php
  ├── stripe-portal-local.php     ← Appelle /shared/stripe-portal.php
  └── mistral-proxy-local.php     ← Appelle /shared/mistral-proxy.php

gid-assistant/
  ├── stripe-checkout-local.php
  ├── stripe-portal-local.php
  └── mistral-proxy-local.php

[autres apps] → même pattern
```

**Avantage** : Modification des clés API ou logique = un seul fichier à changer. Chaque app peut ajouter sa propre logique avant/après si besoin.

**Nommage convention** : `*-local.php` pour tous les fichiers proxy locaux (exemple : `stripe-checkout-local.php`, `mistral-proxy-local.php`, `stripe-portal-local.php`)

### Firebase config commune à toutes les apps

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyBzghrSLYAjcKYskfkrc7dbSX3DKm_rIEM",
  authDomain: "auth.bimsmarter.eu",
  projectId: "bimsmarter",
  storageBucket: "bimsmarter.firebasestorage.app",
  messagingSenderId: "670432116296",
  appId: "1:670432116296:web:38bac2f70ec706e914e27d"
};
```

---

## Architecture Firestore

### Structure complète cible

```
users/{uid}
  ├── email: "user@example.com"
  ├── displayName: "Prénom Nom"
  ├── lastSeen: "2026-02-26T10:00:00.000Z"
  ├── plan: "free" | "pro" | "enterprise" | "admin"
  ├── planSince: "2026-02-26T00:00:00.000Z"
  ├── stripeCustomerId: "cus_xxx" (vide si Free)
  └── aiQuota:
        ├── date: "2026-02-26"    ← date du jour (YYYY-MM-DD)
        └── count: 23             ← messages consommés aujourd'hui

projects/{projectId}
  ├── ownerId: "uid-du-createur"
  ├── ownerPlan: "pro"            ← snapshot au moment du partage
  ├── status: "active" | "readonly"
  ├── name: "Nom du projet"
  ├── description: "..."
  ├── phase: "APS|APD|PDE|EXE|EXP"
  ├── sharedWithApps: ["gestion-projet", "document-generator"]
  ├── members: ["uid1", "uid2"]   ← max 10 pour Pro
  └── createdAt: timestamp
```

### Règle clé : `aiQuota` est une map (pas une sous-collection)

Firestore stocke `aiQuota` comme un **map imbriqué** dans le document `users/{uid}`, pas une sous-collection. Cela permet de lire/écrire en une seule opération Firestore.

---

## Plans et limites

| Feature | Free | Pro (19€/mois) | Enterprise | Admin |
|---|---|---|---|---|
| Projets | 1 | 5 | 999 | 999 |
| Messages IA / jour | 50 | 200 | 9999 | 9999 |
| Invités par projet | 0 | 10 | 999 | 999 |
| Partage cross-apps | ❌ | ✅ | ✅ | ✅ |
| Quota IA | ✅ (limité) | ✅ (limité) | ✅ (limité) | Bypass total |

### Comportement Admin
- Bypass **total** : aucun quota, aucune limite, aucun toast
- Accès à toutes les fonctionnalités sans restriction
- Plan défini manuellement dans Firestore (pas via Stripe)

---

## Stack technique par app

| App | Stack | Chatbot | Projets | Repo GitHub |
|---|---|---|---|---|
| gestion-projet | Vanilla JS multi-fichiers | ✅ (mistral-proxy.php) | ✅ | rcliment1987/gestion-projet |
| bim-workflow-generator | React (Babel in HTML) | ✅ (mistral-proxy.php) | ❌ | à confirmer |
| GID-Assistant | Vanilla JS mono-fichier | ✅ (webhook n8n ou proxy) | ❌ | à confirmer |
| IFC Viewer | React + modules JS externes | ✅ (audit-flash.js) | ❌ | rcliment1987/IFC-viewer |
| bim-document-generator | React (index.html ~370) | ❌ | ❌ | à confirmer |
| Dimensionnement Techniques | Vanilla JS (js/app.js) | ❌ | ❌ | à confirmer |

### Résumé des étapes par app

| App | Étape 3 | Étape 4 | Étape 5 | Étape 6 |
|---|---|---|---|---|
| gestion-projet | ✅ À faire | ✅ À faire | ✅ À faire | ✅ À faire |
| bim-workflow-generator | ✅ À vérifier (peut être déjà fait) | ❌ Non applicable | ✅ À faire (inline) | ❌ Non applicable |
| GID-Assistant | ✅ À faire | ❌ Non applicable | ✅ À faire (inline) | ❌ Non applicable |
| IFC Viewer | ✅ À faire (+ SDK Firestore) | ❌ Non applicable | ✅ À faire (inline) | ❌ Non applicable |
| bim-document-generator | ✅ À faire | ❌ Non applicable | ❌ Non applicable | ❌ Non applicable |
| Dimensionnement | ✅ À faire | ❌ Non applicable | ❌ Non applicable | ❌ Non applicable |

---

## Étape 1 — Compte Admin (manuel)

**Où** : Console Firebase → Firestore Database → Collection `users` → Document `{ton-uid}`

**Pour trouver ton UID** : Console Firebase → Authentication → Users → colonne "User UID"

**Champs à créer** (bouton "+ Add field" pour chaque) :

| Champ | Type | Valeur |
|---|---|---|
| plan | string | admin |
| planSince | string | 2026-02-26 |
| stripeCustomerId | string | (vide) |
| aiQuota | map | → voir sous-champs |
| aiQuota.date | string | 2026-02-26 |
| aiQuota.count | number | 0 |

> ⚠️ Cette étape est **unique et manuelle**. Elle doit être faite une seule fois par compte admin. Ne pas la scripter.

---

## Étape 2 — Règles Firestore

**Où** : Console Firebase → Firestore Database → Rules

**Remplacer intégralement par** :

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ─────────────────────────────────────────────
    // FONCTIONS UTILITAIRES
    // ─────────────────────────────────────────────

    // Vérifie que la mise à jour ne touche QUE des champs non-sensibles.
    // Le plan, stripeCustomerId et role ne peuvent être modifiés que par
    // le backend (N8N via Admin SDK) — jamais depuis le navigateur.
    // Champs autorisés côté client : displayName, lastSeen, favoriteWorkflows, aiQuota.
    function safeUserFields() {
  return request.resource.data.diff(resource.data).affectedKeys()
    .hasOnly(['displayName', 'lastSeen', 'favoriteWorkflows', 'aiQuota', 
              'codes', 'documents', 'updatedAt', 'gidFavorites', 'gidUpdatedAt']);
}

    // Vérifie que l'utilisateur connecté est bien le propriétaire du document.
    function isOwner(userId) {
      return request.auth != null && request.auth.uid == userId;
    }

    // Vérifie que l'utilisateur connecté est membre du projet.
    // Utilisé pour les sous-collections (workflows, documents d'un projet).
    function isMember(projectId) {
      return request.auth != null
          && request.auth.uid in get(
               /databases/$(database)/documents/projects/$(projectId)
             ).data.members;
    }


    // ─────────────────────────────────────────────
    // COLLECTION : users
    // ─────────────────────────────────────────────

    match /users/{userId} {

      // Lecture : tout utilisateur connecté peut lire le profil d'un autre.
      // Nécessaire pour afficher le nom/plan d'un invité dans gestion-projet
      // (ex : vérifier si un membre invité est Pro avant de lui donner accès).
      allow read: if request.auth != null;

      // Création : seulement son propre document, et uniquement avec plan 'free'.
      // Empêche qu'un utilisateur se crée directement un compte Pro ou Admin
      // en manipulant la requête. Le plan ne peut monter que via le webhook Stripe.
      allow create: if isOwner(userId)
                    && request.resource.data.plan == 'free';

      // Mise à jour : seulement son propre document, et seulement les champs
      // non-sensibles (voir safeUserFields). Le champ 'plan' est bloqué ici —
      // seul N8N via Admin SDK peut le modifier (upgrade/downgrade Stripe).
      allow update: if isOwner(userId) && safeUserFields();

      // Suppression : interdite depuis le client.
      // La suppression d'un compte passe par une procédure admin dédiée.
      allow delete: if false;
    }


    // ─────────────────────────────────────────────
    // SOUS-COLLECTION : workflows personnels d'un user
    // (workflows sauvegardés par l'utilisateur dans son profil,
    //  distincts des workflows d'un projet partagé)
    // ─────────────────────────────────────────────

    match /users/{userId}/workflows/{workflowId} {
      // Lecture et écriture : uniquement le propriétaire du profil.
      allow read, write: if isOwner(userId);
    }
    
    // ─────────────────────────────────────────────
    // SOUS-COLLECTION : historique audit IFC-viewer
    // ─────────────────────────────────────────────

    match /users/{userId}/audits/{auditId} {
      allow read, create, update, delete: if isOwner(userId);
    }

    // ─────────────────────────────────────────────
    // COLLECTION : projects
    // ─────────────────────────────────────────────

    match /projects/{projectId} {

      // Lecture : tout utilisateur connecté qui est dans la liste members[].
      // La liste members[] inclut le propriétaire + les invités Pro.
      // Un utilisateur non-membre ne peut pas lire le projet (ni son nom, ni son contenu).
      allow read: if request.auth != null
                  && request.auth.uid in resource.data.members;

      // Création : tout utilisateur connecté peut créer un projet,
      // à condition de se déclarer lui-même comme ownerId.
      // La limite du nombre de projets (Free = 1, Pro = 5) est gérée
      // côté applicatif par PlanManager — pas dans les règles Firestore.
      allow create: if request.auth != null
                    && request.auth.uid == request.resource.data.ownerId;

      // Mise à jour : uniquement le propriétaire du projet (ownerId).
      // Les membres invités peuvent lire mais pas modifier la structure du projet.
      allow update: if request.auth != null
                    && request.auth.uid == resource.data.ownerId;

      // Suppression : uniquement le propriétaire du projet.
      allow delete: if request.auth != null
                    && request.auth.uid == resource.data.ownerId;
    }


    // ─────────────────────────────────────────────
    // SOUS-COLLECTION : workflows d'un projet partagé
    // (étapes BIM, livrables, séquences de travail liées à un projet)
    // ─────────────────────────────────────────────

    match /projects/{projectId}/workflows/{workflowId} {
      // Lecture et écriture : tout membre du projet parent.
      // isMember() relit le document projet pour vérifier la liste members[].
      allow read, write: if isMember(projectId);
    }


    // ─────────────────────────────────────────────
    // SOUS-COLLECTION : documents d'un projet partagé
    // (documents ISO 19650 générés par bim-document-generator,
    //  liés à un projet dans gestion-projet)
    // ─────────────────────────────────────────────

    match /projects/{projectId}/documents/{docId} {
      // Lecture et écriture : tout membre du projet parent.
      // Même logique que pour les workflows — accès partagé entre membres.
      allow read, write: if isMember(projectId);
    }
    
    // ─────────────────────────────────────────────
    // COLLECTION : analyses (SoumissionAI)
    // ─────────────────────────────────────────────
    match /analyses/{docId} {
      allow create: if request.auth != null
                    && request.resource.data.userId == request.auth.uid;
      allow read: if request.auth != null
                  && resource.data.userId == request.auth.uid;
      allow update: if request.auth != null
                    && resource.data.userId == request.auth.uid;
      allow delete: if request.auth != null
                    && resource.data.userId == request.auth.uid;
    }

  }
}

```

**Pourquoi ces règles** :
- `users` en lecture pour tous les authentifiés → permet de vérifier le plan d'un invité
- `users` en écriture uniquement pour soi → un user ne peut pas modifier le plan d'un autre
- `projects` accessible si propriétaire OU membre → base du partage Pro

---

## Étape 3 — Profil automatique à l'inscription

### Principe
À chaque connexion, vérifier si le profil Firestore existe. Si non → créer avec plan "free" et les quotas à zéro. Si oui → juste mettre à jour `lastSeen`.

### Pattern universel (à adapter selon la variable `db` disponible dans chaque app)

```javascript
// Pattern cible — à appliquer dans onAuthStateChanged (ou équivalent)
const userDoc = await db.collection('users').doc(user.uid).get();

if (!userDoc.exists) {
    // Nouvel utilisateur → créer profil complet avec plan Free
    await db.collection('users').doc(user.uid).set({
        email: user.email,
        displayName: user.displayName || user.email,
        lastSeen: new Date().toISOString(),
        plan: "free",
        planSince: new Date().toISOString(),
        stripeCustomerId: "",
        aiQuota: {
            date: new Date().toISOString().split('T')[0],
            count: 0
        }
    });
} else {
    // User existant → juste mettre à jour lastSeen
    await db.collection('users').doc(user.uid).set(
        { lastSeen: new Date().toISOString() }, 
        { merge: true }
    );
}
```

### Instructions spécifiques par app

---

#### APP 1 — `gestion-projet` (Vanilla JS multi-fichiers)

**Fichier** : `index.html`

**Où chercher** : Bloc `onAuthStateChanged`. Contient un `db.collection('users').doc(user.uid).set(...)` avec `merge: true`.

**Action** : Remplacer ce bloc `set(...)` par le pattern universel ci-dessus. La variable `db` est déjà disponible globalement dans ce fichier.

**Vérifier** : `window.currentUser = user;` doit être présent dans ce même bloc (nécessaire pour PlanManager à l'étape 4).

---

#### APP 2 — `bim-workflow-generator` (React Babel in HTML)

**Fichier** : `index.html` (unique fichier, script type="text/babel")

**Où chercher** : `useEffect` contenant `auth.onAuthStateChanged`. À l'intérieur, chercher le bloc `if (!doc.exists)`.

**Action** : Ce bloc est **peut-être déjà implémenté**. Vérifier que les 4 champs sont présents : `plan`, `planSince`, `stripeCustomerId`, `aiQuota`. Si oui → étape 3 déjà faite, passer à l'étape 5. Si manquants → compléter.

**Variable db** : Dans cette app, `db` est probablement `firebase.firestore()` initialisé dans le même useEffect ou au niveau du composant.

---

#### APP 3 — `GID-Assistant` (Vanilla JS mono-fichier)

**Fichier** : `index.html` (ou `index__2_.html` selon le nom sur le serveur)

**Où chercher** : Dans les scripts inline, chercher `onAuthStateChanged`. Trouver le `db.collection('users')...set(...)`.

**Action** : Appliquer le pattern universel. Vérifier le nom de la variable Firestore (`db` ou autre).

---

#### APP 4 — `IFC Viewer` (React + modules JS externes)

**Spécificité critique** : Le SDK Firestore n'est peut-être pas chargé dans `index.html`.

**Étape préalable** — Vérifier dans `index.html` la présence de :
```html
<script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
```
Si absent → ajouter **après** `firebase-auth-compat.js`.

Et dans le bloc inline Firebase, ajouter après `window.firebaseAuth = firebase.auth();` :
```javascript
window.firebaseDB = firebase.firestore();
```

**Fichier** : `audit-flash.js`, fonction `init()`

**Adapter le pattern** avec `window.firebaseDB` et `window.firebaseAuth.currentUser` :

```javascript
// Dans init(), après le check currentUser, avant injectStyles()
if (window.firebaseDB) {
    var user = window.firebaseAuth.currentUser;
    var userRef = window.firebaseDB.collection('users').doc(user.uid);
    userRef.get().then(function(doc) {
        if (!doc.exists) {
            return userRef.set({
                email: user.email,
                displayName: user.displayName || user.email,
                lastSeen: new Date().toISOString(),
                plan: "free",
                planSince: new Date().toISOString(),
                stripeCustomerId: "",
                aiQuota: { date: new Date().toISOString().split('T')[0], count: 0 }
            });
        } else {
            return userRef.set({ lastSeen: new Date().toISOString() }, { merge: true });
        }
    }).catch(function(err) {
        console.warn('[BIMSmarter] Firestore profile error:', err);
    });
}
```

---

#### APP 5 — `bim-document-generator` (React)

**Fichier** : `index.html`, bloc `onAuthStateChanged` vers la ligne 370.

**Action** : Vérifier présence des champs `plan`, `planSince`, `stripeCustomerId`, `aiQuota`. Appliquer le pattern universel si manquants.

---

#### APP 6 — `Dimensionnement Techniques Luxembourg` (Vanilla JS)

**Fichier** : `js/app.js`, fonction `initAuth()`

**Action** : Localiser le bloc Firebase user set, appliquer le pattern universel.

---

## Étape 4 — PlanManager (gestion-projet uniquement)

### Pourquoi uniquement gestion-projet ?

PlanManager est un objet utilitaire réutilisable qui sert **à la fois** pour le quota IA (étape 5) ET la limite projets (étape 6). `gestion-projet` est la seule app qui a les deux besoins. Pour les autres apps, une vérification inline est plus simple et moins risquée.

### Fichier cible : `app.js`

**Où insérer** : Juste avant la déclaration `const DataStore = {`. Chercher cette ligne exacte et insérer le bloc ci-dessous juste avant.

**Bloc à insérer** :

```javascript
// ============================
// PLAN & QUOTA MANAGEMENT
// ============================
const PlanManager = {

    LIMITS: {
        free:        { projects: 1,   aiMessages: 50,   invites: 0,   sharing: false },
        pro:         { projects: 5,   aiMessages: 200,  invites: 10,  sharing: true  },
        enterprise:  { projects: 999, aiMessages: 9999, invites: 999, sharing: true  },
        admin:       { projects: 999, aiMessages: 9999, invites: 999, sharing: true  }
    },

    // Récupère le plan depuis Firestore (mis en cache dans window pour la session)
    async getUserPlan() {
        if (window.userPlan) return window.userPlan; // cache session
        if (!window.currentUser) return "free";
        try {
            const doc = await db.collection('users').doc(window.currentUser.uid).get();
            window.userPlan = doc.data()?.plan || "free";
            return window.userPlan;
        } catch (e) {
            return "free"; // fallback sécurisé en cas d'erreur réseau
        }
    },

    // Vérifie l'accès à une feature (boolean)
    async canAccess(feature) {
        const plan = await this.getUserPlan();
        if (plan === "admin") return true;
        const limits = this.LIMITS[plan] || this.LIMITS.free;
        return limits[feature] === true || limits[feature] > 0;
    },

    // Vérifie quota IA et incrémente si OK — retourne true si autorisé
    async checkAndIncrementAI() {
        const plan = await this.getUserPlan();
        if (plan === "admin") return true; // admin : bypass total, pas de quota

        if (!window.currentUser) return false;

        const uid = window.currentUser.uid;
        const today = new Date().toISOString().split('T')[0]; // "YYYY-MM-DD"
        const limit = this.LIMITS[plan]?.aiMessages || 50;

        const userRef = db.collection('users').doc(uid);
        const doc = await userRef.get();
        const quota = doc.data()?.aiQuota || { date: today, count: 0 };

        // Reset automatique si nouveau jour
        if (quota.date !== today) {
            await userRef.update({ aiQuota: { date: today, count: 1 } });
            return true;
        }

        // Quota atteint → bloquer et afficher toast
        if (quota.count >= limit) {
            showToast(`Quota IA atteint (${limit} messages/jour). Passez en Pro !`, 'error');
            return false;
        }

        // Incrémenter le compteur atomiquement
        await userRef.update({ 
            'aiQuota.count': firebase.firestore.FieldValue.increment(1) 
        });
        return true;
    },

    // Vérifie si peut créer un nouveau projet
    async canCreateProject(currentProjectCount) {
        const plan = await this.getUserPlan();
        if (plan === "admin") return true;
        const limit = this.LIMITS[plan]?.projects || 1;
        if (currentProjectCount >= limit) {
            showToast(`Limite de ${limit} projet(s) atteinte. Passez en Pro !`, 'error');
            return false;
        }
        return true;
    }
};
```

**Dépendances** :
- `db` : variable globale Firestore disponible dans `app.js` ✅
- `window.currentUser` : doit être défini dans `onAuthStateChanged` de `index.html` (`window.currentUser = user;`) — vérifier que cette ligne est bien présente
- `showToast` : fonction globale existante dans l'app ✅
- `firebase.firestore.FieldValue.increment` : disponible via le SDK compat ✅

---

## Étape 5 — Quota IA sur le chatbot

### gestion-projet (`app-part3.js`)

**Chercher** : La fonction qui fait un `fetch` vers `mistral-proxy.php`. Chercher le pattern `fetch` + `mistral-proxy`.

**Action** :
1. Identifier le nom exact de la fonction
2. Vérifier qu'elle est déclarée `async` — si non, ajouter `async`
3. Insérer au tout début du corps :

```javascript
// Vérification quota IA avant appel Mistral
const canUseAI = await PlanManager.checkAndIncrementAI();
if (!canUseAI) return; // bloqué, toast déjà affiché par PlanManager
```

---

### bim-workflow-generator (React Babel in `index.html`)

**Chercher** : Fonction `handleChatSend` dans le `<script type="text/babel">`.

**Action** : Vérifier que la fonction est `async`. Ajouter au début :

```javascript
// Vérification quota IA (inline — pas de PlanManager dans cette app)
if (user) {
    const today = new Date().toISOString().split('T')[0];
    const userDocRef = db.collection('users').doc(user.uid);
    const userDocSnap = await userDocRef.get();
    const userData = userDocSnap.data() || {};
    const quota = userData.aiQuota || { date: today, count: 0 };
    const plan = userData.plan || 'free';
    const limits = { free: 50, pro: 200, enterprise: 9999, admin: 9999 };
    const limit = limits[plan] || 50;

    if (plan !== 'admin') {
        if (quota.date !== today) {
            // Nouveau jour → reset
            await userDocRef.update({ aiQuota: { date: today, count: 1 } });
        } else if (quota.count >= limit) {
            // Quota atteint → bloquer
            // Adapter selon la fonction toast disponible dans cette app
            alert(`Quota IA atteint (${limit} msg/jour). Passez en Pro !`);
            return;
        } else {
            // Incrémenter
            await userDocRef.update({ 
                'aiQuota.count': firebase.firestore.FieldValue.increment(1) 
            });
        }
    }
}
```

> ⚠️ Dans cette app React, `user` est l'état local du composant (probablement `const [user, setUser] = useState(null)`). Vérifier le nom exact de la variable user dans le scope de `handleChatSend`.

---

### GID-Assistant (Vanilla JS mono-fichier)

**Chercher** : La fonction qui appelle le webhook n8n ou mistral-proxy. Chercher `fetch` dans les scripts.

**Action** : Même vérification inline que bim-workflow-generator, mais adapter :
- `db` → variable Firestore locale de cette app (vérifier son nom)
- La fonction toast → vérifier comment les notifications sont affichées dans cette app

```javascript
// Vérification quota IA (inline GID-Assistant)
if (firebase.auth().currentUser) {
    const user = firebase.auth().currentUser;
    const today = new Date().toISOString().split('T')[0];
    const userRef = db.collection('users').doc(user.uid);
    const snap = await userRef.get();
    const userData = snap.data() || {};
    const quota = userData.aiQuota || { date: today, count: 0 };
    const plan = userData.plan || 'free';
    const limits = { free: 50, pro: 200, enterprise: 9999, admin: 9999 };
    const limit = limits[plan] || 50;

    if (plan !== 'admin') {
        if (quota.date !== today) {
            await userRef.update({ aiQuota: { date: today, count: 1 } });
        } else if (quota.count >= limit) {
            // Afficher message d'erreur — adapter à l'UI de cette app
            console.warn('[GID] Quota IA atteint');
            return;
        } else {
            await userRef.update({ 'aiQuota.count': firebase.firestore.FieldValue.increment(1) });
        }
    }
}
```

---

### IFC Viewer (`audit-flash.js`)

**Chercher** : Fonction `sendMessageDirectGlobal` ou équivalent (la fonction qui appelle le chatbot).

**Action** : Même logique inline mais avec `window.firebaseDB` et `window.firebaseAuth.currentUser` :

```javascript
// Vérification quota IA (inline IFC Viewer)
if (window.firebaseDB && window.firebaseAuth && window.firebaseAuth.currentUser) {
    var user = window.firebaseAuth.currentUser;
    var today = new Date().toISOString().split('T')[0];
    var userRef = window.firebaseDB.collection('users').doc(user.uid);
    var snap = await userRef.get();
    var userData = snap.data() || {};
    var quota = userData.aiQuota || { date: today, count: 0 };
    var plan = userData.plan || 'free';
    var limits = { free: 50, pro: 200, enterprise: 9999, admin: 9999 };
    var limit = limits[plan] || 50;

    if (plan !== 'admin') {
        if (quota.date !== today) {
            await userRef.update({ aiQuota: { date: today, count: 1 } });
        } else if (quota.count >= limit) {
            console.warn('[IFC] Quota IA atteint');
            return;
        } else {
            await userRef.update({ 'aiQuota.count': firebase.firestore.FieldValue.increment(1) });
        }
    }
}
```

> ⚠️ Si cette fonction n'est pas déjà `async`, il faut l'ajouter. En Vanilla JS avec `var`, le `await` nécessite `async function`.

---

## Étape 6 — Limite projets (`gestion-projet` uniquement)

**Fichier** : `app-part3.js`

**Chercher** : La fonction de création de projet. Chercher `DataStore.projects.push` ou un handler bouton "Nouveau projet" ou `createProject`.

**Action** :
1. Lire la fonction entière pour comprendre son contexte
2. Vérifier qu'elle est `async` — si non, ajouter `async`
3. Insérer au début du corps :

```javascript
// Vérification limite projets selon le plan
const canCreate = await PlanManager.canCreateProject(DataStore.projects.length);
if (!canCreate) return; // bloqué, toast déjà affiché par PlanManager
```

---

## Points d'attention critiques

### ⚠️ Risque n°1 — Compteur IA centralisé Firestore

Le compteur IA est une **source de vérité unique** dans Firestore pour toutes les apps. La structure :

```
users/{uid}
  aiQuota:
    date: "2026-02-26"   ← date du jour au format YYYY-MM-DD
    count: 23            ← messages consommés aujourd'hui
```

**Fonctionnement** : Chaque app, avant d'envoyer un message au chatbot, lit ce compteur. Si `date` ≠ aujourd'hui → reset `count` à 1 et mettre à jour `date`. Si `count >= limite` → bloquer. Sinon → incrémenter atomiquement via `FieldValue.increment(1)`.

**Points clés** :
- `FieldValue.increment(1)` est **atomique** → pas de race condition si deux apps appellent simultanément
- Le reset date est automatique → pas de cron job nécessaire
- Un user qui passe de Free à Pro voit sa limite augmenter **immédiatement** (le cache `window.userPlan` doit être invalidé lors du changement de plan)

**Invalidation du cache** : Lors d'un changement de plan (webhook Stripe), penser à `delete window.userPlan;` pour forcer une nouvelle lecture Firestore.

---

### ⚠️ Risque n°2 — Downgrade Pro → Free

Quand un utilisateur Pro repasse en Free, il peut avoir jusqu'à 5 projets actifs. Règles :

1. L'utilisateur doit **choisir 1 projet à conserver actif**
2. Les autres projets passent en `status: "readonly"` — visibles mais non modifiables
3. Les invités des projets bloqués perdent accès
4. Le partage cross-apps est coupé sur tous ses projets

**Flow technique** :
- Stripe webhook `customer.subscription.deleted` → N8N reçoit l'événement
- N8N appelle une Cloud Function ou un endpoint qui :
  1. Récupère tous les projets de l'utilisateur dans Firestore
  2. Déclenche une notification UI "Choisissez votre projet actif"
  3. Attend la réponse (ou timeout 48h → choisit le plus récent automatiquement)
  4. Set les autres en `status: "readonly"`
  5. Set `plan: "free"` dans `users/{uid}`
  6. Invalide le cache `window.userPlan` côté client au prochain login

**UI de downgrade** : À implémenter dans `gestion-projet` — modal qui liste les projets et demande lequel conserver. Se déclenche si `plan === "free"` ET `projects.filter(p => p.status === "active").length > 1`.

---

### ⚠️ Risque n°3 — Intégration Stripe (voir section dédiée)

---

### ⚠️ Risque n°4 — IFC Viewer indépendant

L'IFC Viewer n'a pas de projets BIMSmarter donc :
- Pas de quota projets à vérifier (étape 6 non applicable)
- Mais il **consomme le quota IA partagé** → étape 5 obligatoire
- Utilise `window.firebaseDB` et `window.firebaseAuth` (pas `db` directement)
- Le SDK Firestore doit être explicitement chargé dans `index.html` (vérification obligatoire)

---

## Downgrade Pro → Free

### Flow complet détaillé

```
Stripe webhook: customer.subscription.deleted
    ↓
N8N reçoit l'événement
    ↓
Lecture stripeCustomerId dans Firestore → récupère uid
    ↓
Lecture projects/ où ownerId == uid
    ↓
Si count(projects actifs) <= 1 → set plan: "free" directement
Si count(projects actifs) > 1 → déclencher flow de sélection
    ↓
[Flow sélection]
    → Notifier l'utilisateur (email + notification in-app)
    → Modal dans gestion-projet : "Choisissez 1 projet à garder actif"
    → Sur confirmation : 
        - projet choisi : status reste "active"
        - autres projets : status → "readonly", members → []
        - sharedWithApps → []
    ↓
set users/{uid}.plan = "free"
set users/{uid}.planSince = now()
```

### Gestion des projets readonly

Dans `gestion-projet`, vérifier `project.status === "readonly"` avant chaque action de modification :
- Création de tâche → bloquer si projet readonly
- Modification de tâche → bloquer
- Ajout de membre → bloquer
- Afficher un badge "Lecture seule" sur le projet

---

## Intégration Stripe (via N8N)

> 📄 **Pour l'implémentation complète** : lire `references/stripe-n8n-integration.md`
> Ce fichier contient : code PHP complet (Checkout + Portal), workflow N8N step-by-step,
> composants React UI, cartes de test, checklist go-live, timeline 4 semaines.

### IDs Stripe de production (réels)

| Élément | Valeur |
|---|---|
| Product ID | `prod_U4MuXIB3GRIxzK` |
| Price ID mensuel | `price_1T6EALHJbbn2JQrqXDTn8S9q` |
| Montant | 19€/mois (1900 centimes EUR) |
| Webhook URL N8N | `https://n8n.bimsmarter.eu/webhook/stripe-events` |

### Architecture globale

```
User clique "Passer Pro"
    ↓
React → POST stripe-checkout.php (PHP Hostinger)
    ↓ (clé sk_ côté serveur uniquement)
Stripe Checkout (hébergé par Stripe)
    ↓ paiement réussi
Stripe → webhook POST → N8N (https://n8n.bimsmarter.eu/webhook/stripe-events)
    ↓ vérification signature whsec_...
N8N switch event.type :
    invoice.paid           → PATCH Firestore users/{uid} → plan: "pro"
    subscription.deleted   → flow downgrade → plan: "free" + projets readonly
    subscription.updated   → sync plan
    ↓
Réponse 200 OK à Stripe (< 30s obligatoire)
```

### Événements Stripe à écouter

| Événement | Action Firestore |
|---|---|
| `invoice.paid` | `plan: "pro"`, `planSince: now()`, `stripeCustomerId: cus_xxx` |
| `customer.subscription.deleted` | Flow downgrade → `plan: "free"`, projets en `readonly` |
| `customer.subscription.updated` | Sync plan si changement |
| `payment_intent.payment_failed` | Notification email, plan reste actif (Stripe gère les retries) |

### Règle CRITIQUE : firebase_uid dans les metadata Stripe

Lors de la création de la Checkout Session, passer **impérativement** :
```javascript
metadata: { firebase_uid: user.uid },
subscription_data: { metadata: { firebase_uid: user.uid } }
```
→ Permet à N8N de retrouver l'utilisateur Firebase sans query Firestore à chaque événement.

### Sécurité — Règles absolues

- Clé `sk_test_`/`sk_live_` **uniquement en variable d'env PHP/N8N** — jamais dans le code React
- Le plan est **toujours lu depuis Firestore**, jamais depuis localStorage
- Vérification signature webhook obligatoire dans N8N (`whsec_...`)
- **Règle Firestore renforcée à déployer avant go-live** (champ `plan` non modifiable côté client) :

```javascript
match /users/{userId} {
  allow read: if request.auth != null;
  allow write: if request.auth != null && request.auth.uid == userId
    && !('plan' in request.resource.data.diff(resource.data).affectedKeys());
}
```

- Le Service Account Firebase (pour N8N) a les droits admin — ne jamais l'exposer dans un repo GitHub

### Deadline go-live : 1er avril 2026

| Semaine | Objectif |
|---|---|
| 8–14 mars | stripe-checkout.php + stripe-portal.php + Customer Portal configuré |
| 15–21 mars | Workflow N8N upgrade/downgrade opérationnel |
| 22–28 mars | UI React (boutons upgrade, page success, modal downgrade) + tests E2E |
| 29 mar–1 avr | CGV, sk_live_, test paiement réel, go-live |

---

## Checklist de vérification finale

### Après Étape 3 (toutes apps)
- [ ] Créer un nouveau compte → vérifier dans Firestore que `users/{uid}` contient `plan`, `planSince`, `stripeCustomerId`, `aiQuota`
- [ ] Se connecter avec un compte existant → vérifier que `lastSeen` est mis à jour mais pas `plan`

### Après Étape 4 (gestion-projet)
- [ ] Ouvrir la console browser sur gestion-projet → taper `PlanManager` → doit retourner l'objet (pas `undefined`)
- [ ] `await PlanManager.getUserPlan()` → doit retourner `"admin"` pour le compte admin

### Après Étape 5 (apps avec chatbot)
- [ ] Envoyer un message chatbot → vérifier dans Firestore que `aiQuota.count` s'incrémente
- [ ] Simuler quota atteint (modifier `count` à 50 manuellement dans Firestore) → le 51ème message doit être bloqué avec un toast/message d'erreur
- [ ] Changer la date dans Firestore à hier → le prochain message doit reset le count à 1

### Après Étape 6 (gestion-projet)
- [ ] Créer un deuxième projet avec un compte Free → doit être bloqué avec toast
- [ ] Vérifier que le premier projet est toujours créable

### Tests admin
- [ ] Avec le compte admin, aucun toast de blocage ne doit apparaître
- [ ] Quota IA illimité (tester 50+ messages)
- [ ] Création de plusieurs projets possible

---

## Ordre d'exécution recommandé

```
1. Étape 1 — Admin Firestore (manuel, 2 min)
        ↓
2. Étape 2 — Règles Firestore (console Firebase, copier-coller)
        ↓
3. Étape 3 — gestion-projet (index.html)
        ↓
4. Étape 4 — gestion-projet (app.js — PlanManager)
        ↓
5. Étape 5 — gestion-projet (app-part3.js — quota IA chatbot)
        ↓
6. Étape 6 — gestion-projet (app-part3.js — limite projets)
        ↓
   TESTS gestion-projet complets avant de continuer
        ↓
7. Étape 3+5 — bim-workflow-generator
        ↓
8. Étape 3+5 — GID-Assistant
        ↓
9. Étape 3+5 — IFC Viewer (+ SDK Firestore si manquant)
        ↓
10. Étape 3 — bim-document-generator
        ↓
11. Étape 3 — Dimensionnement Techniques
        ↓
   TESTS finaux cross-apps (quota IA partagé)
```

---

## Notes de développement

### Cache window.userPlan

Le plan est mis en cache dans `window.userPlan` pour éviter une lecture Firestore à chaque message chatbot. Ce cache est valable pour la session (jusqu'au rechargement de la page). 

**Cas à gérer** : Si le plan change (upgrade/downgrade via Stripe), le cache sera stale jusqu'au prochain rechargement. Acceptable pour v1.

### FieldValue.increment vs lecture-modification-écriture

On utilise `FieldValue.increment(1)` plutôt que `count + 1` pour éviter les race conditions si l'utilisateur envoie deux messages simultanément depuis deux onglets/apps. C'est une opération atomique Firestore.

### Gestion des erreurs réseau

Si Firestore est inaccessible (offline), les fonctions `checkAndIncrementAI()` et `canCreateProject()` peuvent lancer des erreurs. Ajouter un try/catch autour des appels PlanManager dans les apps :

```javascript
try {
    const canUseAI = await PlanManager.checkAndIncrementAI();
    if (!canUseAI) return;
} catch (err) {
    console.warn('[PlanManager] Erreur Firestore, autorisation par défaut:', err);
    // En cas d'erreur réseau → autoriser (fail open) pour ne pas bloquer l'UX
}
```
