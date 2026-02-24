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

### Pattern de base BIMSmarter
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // User peut lire/modifier uniquement son profil
    match /users/{uid} {
      allow read, write: if request.auth.uid == uid;
    }

    // Projets : accès membres uniquement
    match /projects/{projectId} {
      allow read: if request.auth.uid in resource.data.members;
      allow create: if request.auth != null;
      allow update: if request.auth.uid in resource.data.members;
      allow delete: if request.auth.uid == resource.data.createdBy;

      // Sous-collections héritent de la règle parent
      match /documents/{docId} {
        allow read, write: if request.auth.uid in get(/databases/$(database)/documents/projects/$(projectId)).data.members;
      }
    }

    // Workflows : propriétaire uniquement
    match /workflows/{workflowId} {
      allow read, write: if request.auth.uid == resource.data.createdBy;
    }
  }
}
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
