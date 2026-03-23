---
name: bimsmarter-firebase
description: Patterns Firebase et Firestore pour les apps BIMSmarter. Utilise ce skill quand l'utilisateur parle de Firestore, Firebase Auth, règles de sécurité, requêtes lentes, structure de données, ou erreurs Firebase. Si l'utilisateur mentionne "Firestore", "Firebase", "règles sécurité", "requête lente", "collection", "document Firebase", utilise ce skill.
---

---
name: bimsmarter-firebase
description: Patterns Firebase et Firestore pour les apps BIMSmarter. Utilise ce skill quand l'utilisateur parle de Firestore, Firebase Auth, règles de sécurité, requêtes lentes, structure de données, ou erreurs Firebase. Si l'utilisateur mentionne "Firestore", "Firebase", "règles sécurité", "requête lente", "collection", "document Firebase", utilise ce skill.
---

# Firebase BIMSmarter — Firestore & Auth

Stack : Firebase (Auth + Firestore + Storage) sur les 3 apps BIMSmarter.

---

## Structure Firestore actuelle

```
/users/{uid}
    ├── profile: { email, displayName, role, createdAt }
    └── settings: { ... }

/projects/{projectId}
    ├── metadata: { name, phase, createdBy, createdAt }
    ├── members: [uid1, uid2]
    └── /documents/{docId}
            ├── type: "ISO19650" | "GID"
            ├── content: { ... }
            └── updatedAt: timestamp

/workflows/{workflowId}
    ├── createdBy: uid
    ├── steps: [...]
    └── status: "draft" | "active"
```

---

## Patterns de requêtes

### Requête de base (avec gestion d'erreur)
```javascript
import { collection, query, where, getDocs, orderBy, limit } from 'firebase/firestore';

const fetchUserProjects = async (uid) => {
  try {
    const q = query(
      collection(db, 'projects'),
      where('members', 'array-contains', uid),
      orderBy('createdAt', 'desc'),
      limit(20)
    );
    const snapshot = await getDocs(q);
    return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  } catch (error) {
    console.error('Firestore error:', error.code, error.message);
    throw error;
  }
};
```

### Listener temps réel (avec cleanup obligatoire)
```javascript
import { onSnapshot } from 'firebase/firestore';

useEffect(() => {
  const unsubscribe = onSnapshot(
    query(collection(db, 'projects'), where('createdBy', '==', uid)),
    (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setProjects(data);
    },
    (error) => console.error('Listener error:', error)
  );

  return () => unsubscribe(); // ← OBLIGATOIRE sinon memory leak
}, [uid]);
```

### Écriture sécurisée (avec merge pour ne pas écraser)
```javascript
import { doc, setDoc, serverTimestamp } from 'firebase/firestore';

// setDoc avec merge:true = update partiel (ne supprime pas les autres champs)
await setDoc(
  doc(db, 'projects', projectId),
  { status: 'active', updatedAt: serverTimestamp() },
  { merge: true }
);
```

### Batch write (plusieurs docs en une transaction)
```javascript
import { writeBatch, doc } from 'firebase/firestore';

const batch = writeBatch(db);
steps.forEach((step, i) => {
  batch.set(doc(db, 'steps', `${workflowId}_${i}`), step);
});
await batch.commit(); // max 500 docs par batch
```

---

## Règles de sécurité Firestore

### Règles de production actuelles (v2 — avec protection plan Stripe)

> ⚠️ Ces règles sont les règles **de production** déployées sur Firebase.
> Ne pas simplifier — chaque restriction est intentionnelle (voir commentaires).

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

### Points clés de sécurité

- **`plan` non modifiable côté client** : `safeUserFields()` bloque `plan`, `stripeCustomerId`, `role`. Seul N8N via Admin SDK peut les modifier (webhooks Stripe upgrade/downgrade).
- **Création compte** : forcé à `plan: 'free'` — impossible de se créer un compte Pro en manipulant la requête.
- **Membres vs propriétaire** : les invités peuvent lire un projet mais pas le modifier (seul `ownerId` a le droit `update`/`delete`).
- **Limite projets** : gérée par `PlanManager` côté app, pas dans les règles Firestore (Firestore ne peut pas compter les documents facilement).

### Déploiement
```bash
firebase deploy --only firestore:rules
```

---

## Diagnostic des problèmes fréquents

### Requête lente sur gros volume
**Root cause** : pas d'index composite + Firestore facture chaque document lu.

```javascript
// ❌ LENT : filter côté client = lit TOUS les docs
const all = await getDocs(collection(db, 'documents'));
const filtered = all.docs.filter(d => d.data().phase === 'APS');

// ✅ RAPIDE : filter côté Firestore = lit seulement les docs matchants
const q = query(
  collection(db, 'documents'),
  where('phase', '==', 'APS'),
  limit(50)
);
```

Si erreur "index required" → Firestore affiche un lien direct pour créer l'index. Toujours cliquer ce lien.

### Permission denied
```
FirebaseError: Missing or insufficient permissions
```
- Vérifier que `request.auth` n'est pas null (utilisateur connecté ?)
- Tester les règles dans l'émulateur Firestore ou Firebase Console → Règles → Playground

### Listener qui se déclenche en boucle
**Root cause** : même cause que le bug React re-render — dépendance instable dans `useEffect`.

```javascript
// ❌ uid recréé à chaque render
useEffect(() => {
  const unsub = onSnapshot(...);
  return unsub;
}, [user]); // user est un objet → référence instable

// ✅ uid est une string → référence stable
useEffect(() => {
  if (!user?.uid) return;
  const unsub = onSnapshot(...);
  return unsub;
}, [user?.uid]);
```

### Quota Firestore dépassé
- Firestore : 50k lectures/jour en gratuit
- Ajouter du cache local pour éviter les re-fetch inutiles :
```javascript
const [cache, setCache] = useState({});

const fetchWithCache = async (id) => {
  if (cache[id]) return cache[id]; // hit cache
  const data = await getDoc(doc(db, 'projects', id));
  setCache(prev => ({ ...prev, [id]: data.data() }));
  return data.data();
};
```

---

## Scalabilité sur données IFC volumineuses

Le problème : les fichiers IFC peuvent faire 50-200MB. **Ne jamais stocker le contenu IFC brut dans Firestore** (limite : 1MB par document).

### Pattern recommandé
```
Fichier IFC → Firebase Storage (stockage binaire)
Métadonnées IFC → Firestore (référence + données extraites)
```

```javascript
// Stockage fichier IFC dans Storage
import { ref, uploadBytes, getDownloadURL } from 'firebase/storage';

const uploadIFC = async (file, projectId) => {
  const storageRef = ref(storage, `ifc/${projectId}/${file.name}`);
  const snapshot = await uploadBytes(storageRef, file);
  const url = await getDownloadURL(snapshot.ref);

  // Sauvegarder seulement la référence dans Firestore
  await setDoc(doc(db, 'projects', projectId), {
    ifc: { url, filename: file.name, uploadedAt: serverTimestamp() }
  }, { merge: true });

  return url;
};
```

---

## Commandes utiles Firebase CLI

```bash
# Déployer les règles Firestore
firebase deploy --only firestore:rules

# Déployer les règles Storage
firebase deploy --only storage

# Émuler en local (évite de toucher la prod)
firebase emulators:start --only firestore,auth

# Exporter données Firestore (backup)
firebase firestore:export ./backup
```
