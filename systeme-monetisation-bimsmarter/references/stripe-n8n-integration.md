# Stripe & N8N — Intégration complète BIMSmarter

> Référence technique exhaustive. Lire ce fichier quand Renaud travaille sur :
> Checkout, Customer Portal, webhooks Stripe → N8N → Firebase, go-live, tests.

---

## IDs Stripe de production

| Élément | ID / Valeur |
|---|---|
| Product | `prod_U4MuXIB3GRIxzK` |
| Price mensuel (19€/mois) | `price_1T6EALHJbbn2JQrqXDTn8S9q` |
| Montant | 1900 centimes (EUR) |
| Description produit | "5 projets, 200 requêtes IA/jour, partage d'équipe" |
| Webhook URL cible (N8N) | `https://n8n.bimsmarter.eu/webhook/stripe-events` |

---

## 1. stripe-checkout.php — Créer une session de paiement

**Où** : Hostinger, même répertoire que `mistral-proxy.php`

**Pourquoi PHP et pas React** : La clé secrète Stripe (`sk_test_` / `sk_live_`) ne peut JAMAIS être côté client. Elle serait visible dans le source du navigateur.

```php
<?php
header('Access-Control-Allow-Origin: *');
header('Content-Type: application/json');

require_once 'vendor/autoload.php'; // composer require stripe/stripe-php

$secretKey = getenv('STRIPE_SECRET_KEY'); // Variable d'env Hostinger — jamais en dur
\Stripe\Stripe::setApiKey($secretKey);

$data = json_decode(file_get_contents('php://input'), true);
$priceId     = $data['priceId']     ?? '';
$firebaseUid = $data['firebaseUid'] ?? '';
$email       = $data['email']       ?? '';

if (!$priceId || !$firebaseUid) {
    http_response_code(400);
    echo json_encode(['error' => 'Missing priceId or firebaseUid']);
    exit;
}

// Vérifier le token Firebase (optionnel mais recommandé pour v1)
// En v1 acceptable : faire confiance au firebaseUid passé depuis l'app authentifiée

try {
    $session = \Stripe\Checkout\Session::create([
        'payment_method_types' => ['card'],
        'line_items' => [[
            'price'    => $priceId,
            'quantity' => 1,
        ]],
        'mode'        => 'subscription',
        'customer_email' => $email,
        'metadata'    => [
            'firebase_uid' => $firebaseUid,  // CRITIQUE : permet à N8N de retrouver l'user
        ],
        'subscription_data' => [
            'metadata' => ['firebase_uid' => $firebaseUid],
        ],
        'success_url' => 'https://app.bimsmarter.eu/upgrade-success?session_id={CHECKOUT_SESSION_ID}',
        'cancel_url'  => 'https://app.bimsmarter.eu/pricing',
    ]);
    echo json_encode(['sessionId' => $session->id, 'url' => $session->url]);
} catch (\Exception $e) {
    http_response_code(500);
    echo json_encode(['error' => $e->getMessage()]);
}
?>
```

**Appel depuis React** :
```javascript
const startCheckout = async () => {
  const user = firebase.auth().currentUser;
  const res = await fetch('https://bimsmarter.eu/stripe-checkout.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      priceId: 'price_1T6EALHJbbn2JQrqXDTn8S9q',
      firebaseUid: user.uid,
      email: user.email,
    })
  });
  const { url } = await res.json();
  window.location.href = url; // Redirection directe vers Stripe Checkout
};
```

---

## 2. stripe-portal.php — Accès au Customer Portal

**Usage** : Permettre aux users Pro de gérer leur CB, voir les factures, annuler — sans que Renaud code quoi que ce soit.

```php
<?php
header('Access-Control-Allow-Origin: *');
header('Content-Type: application/json');

require_once 'vendor/autoload.php';
\Stripe\Stripe::setApiKey(getenv('STRIPE_SECRET_KEY'));

$data = json_decode(file_get_contents('php://input'), true);
$stripeCustomerId = $data['stripeCustomerId'] ?? '';

if (!$stripeCustomerId) {
    http_response_code(400);
    echo json_encode(['error' => 'Missing stripeCustomerId']);
    exit;
}

try {
    $session = \Stripe\BillingPortal\Session::create([
        'customer'   => $stripeCustomerId,
        'return_url' => 'https://app.bimsmarter.eu/account',
    ]);
    echo json_encode(['url' => $session->url]);
} catch (\Exception $e) {
    http_response_code(500);
    echo json_encode(['error' => $e->getMessage()]);
}
?>
```

**Appel depuis React** :
```javascript
const openPortal = async () => {
  const userDoc = await db.collection('users').doc(currentUser.uid).get();
  const { stripeCustomerId } = userDoc.data();
  const res = await fetch('https://bimsmarter.eu/stripe-portal.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ stripeCustomerId })
  });
  const { url } = await res.json();
  window.location.href = url;
};
```

**Config Stripe Dashboard requise** :
- Settings → Billing → Customer portal → Activer
- Autoriser : modifier CB ✅, annuler ✅, voir factures ✅, changer de plan ❌
- Return URL : `https://app.bimsmarter.eu/account`

---

## 3. Webhook N8N — Architecture complète

### URL webhook à configurer dans Stripe
`https://n8n.bimsmarter.eu/webhook/stripe-events`

### Événements à écouter dans Stripe Dashboard
- `invoice.paid`
- `customer.subscription.deleted`
- `customer.subscription.updated`
- `payment_intent.payment_failed`

### Structure du workflow N8N

```
[Webhook Trigger] ← POST Stripe
        ↓
[Code Node] — Vérification signature Stripe
  const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
  const sig = $headers['stripe-signature'];
  const event = stripe.webhooks.constructEvent(
    $rawBody, sig, process.env.STRIPE_WEBHOOK_SECRET
  );
  return [{ json: event }];
        ↓
[Switch Node] — sur event.type
  ├── invoice.paid           → branche UPGRADE
  ├── subscription.deleted   → branche DOWNGRADE
  ├── subscription.updated   → branche SYNC
  └── default                → [Respond 200 OK]
        ↓
[Branche UPGRADE]
  1. Extraire firebase_uid depuis event.data.object.subscription_details.metadata.firebase_uid
     (ou depuis event.data.object.metadata.firebase_uid si invoice)
  2. Extraire stripeCustomerId depuis event.data.object.customer
  3. HTTP Request → Firebase REST API (avec Service Account JWT) :
     PATCH /v1/projects/bimsmarter/databases/(default)/documents/users/{uid}
     Body: { fields: {
       plan: { stringValue: "pro" },
       planSince: { stringValue: new Date().toISOString() },
       stripeCustomerId: { stringValue: customerId }
     }}
  4. [Respond 200 OK]
        ↓
[Branche DOWNGRADE]
  1. Extraire firebase_uid (même méthode)
  2. Lire projets Firestore : GET /users/{uid}/projects?ownerId=uid
     (ou query via REST)
  3. [If] count(projets actifs) <= 1 → SET plan: free directement
  4. [If] count(projets actifs) > 1 → 
     - Envoyer email "Choisissez votre projet à conserver" (SMTP/Brevo)
     - SET plan: free (l'app gestion-projet gère le modal de sélection)
  5. [Respond 200 OK]
```

### Récupérer le firebase_uid depuis un webhook Stripe

**Méthode recommandée (passer en metadata lors du Checkout)** :
```javascript
// Dans le webhook N8N — event invoice.paid
const subscriptionMetadata = event.data.object.subscription_details?.metadata;
const firebaseUid = subscriptionMetadata?.firebase_uid;
```

**Méthode fallback (query Firestore par stripeCustomerId)** :
```javascript
// Si firebase_uid absent des metadata
// HTTP Request Firestore REST :
// POST /v1/projects/bimsmarter/databases/(default)/documents:runQuery
// Body: structuredQuery avec where stripeCustomerId == customerId
```

### Service Account Firebase pour N8N
1. Firebase Console → Project Settings → Service Accounts → Generate new private key
2. Télécharger le JSON
3. Dans N8N : stocker les credentials dans les variables d'environnement N8N (pas dans le workflow)
4. Utiliser le node "Google Firebase Admin SDK" ou HTTP Request avec JWT Service Account

> ⚠️ Le Service Account a les droits admin complets sur Firestore. Ne jamais l'exposer dans un repo GitHub ou dans le code côté client.

---

## 4. Configuration Stripe Dashboard — Checklist

### Avant les tests (mode TEST)
- [ ] Vérifier que sk_test_... est actif (basculer via le toggle en haut du dashboard)
- [ ] Webhook → Add endpoint → URL N8N + sélectionner les 4 events
- [ ] Copier le Webhook Signing Secret (whsec_...) → stocker dans N8N comme variable d'env
- [ ] Customer portal → Activer + configurer return URL
- [ ] Branding → Logo BIMSmarter + couleur #1E3A5F

### Avant le go-live (mode LIVE)
- [ ] Basculer sur sk_live_...
- [ ] Recréer le webhook endpoint en mode live (les endpoints test ne fonctionnent pas en live)
- [ ] Tester un vrai paiement de 1€ sur soi-même (puis rembourser)
- [ ] Activer Smart Retries : Settings → Billing → Retry schedule
- [ ] Configurer les emails Stripe : Settings → Emails → reply-to avec email pro

---

## 5. UI React — Composants à implémenter

### Bouton "Passer Pro" (users Free uniquement)
```jsx
const UpgradeButton = () => {
  const [loading, setLoading] = useState(false);
  const handleUpgrade = async () => {
    setLoading(true);
    const user = firebase.auth().currentUser;
    const res = await fetch('https://bimsmarter.eu/stripe-checkout.php', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        priceId: 'price_1T6EALHJbbn2JQrqXDTn8S9q',
        firebaseUid: user.uid,
        email: user.email,
      })
    });
    const { url } = await res.json();
    window.location.href = url;
  };
  return (
    <button onClick={handleUpgrade} disabled={loading}>
      {loading ? 'Redirection...' : '🚀 Passer Pro — 19€/mois'}
    </button>
  );
};
```

### Page success après paiement
```jsx
// URL : /upgrade-success?session_id=cs_xxx
const UpgradeSuccess = () => {
  useEffect(() => {
    // Le webhook N8N met à jour Firestore async
    // Retry si le plan n'est pas encore pro après 3s
    const checkPlan = async () => {
      const doc = await db.collection('users').doc(currentUser.uid).get();
      if (doc.data().plan !== 'pro') {
        setTimeout(checkPlan, 3000); // Retry
      } else {
        window.userPlan = 'pro'; // Invalider le cache
      }
    };
    setTimeout(checkPlan, 1500);
  }, []);
  
  return (
    <div>
      <h1>🎉 Bienvenue Pro !</h1>
      <p>Vos nouvelles limites : 200 requêtes IA/jour, 5 projets, partage d'équipe</p>
    </div>
  );
};
```

### Modale de blocage enrichie
```jsx
// Quand quota atteint ou limite projets
const BlockModal = ({ reason, onClose }) => (
  <div className="modal">
    <h2>{reason === 'quota' ? 'Quota IA atteint' : 'Limite de projets atteinte'}</h2>
    <p>
      {reason === 'quota'
        ? 'Plan Free : 50 messages/jour. Plan Pro : 200 messages/jour.'
        : 'Plan Free : 1 projet. Plan Pro : 5 projets.'}
    </p>
    <UpgradeButton />
    <button onClick={onClose}>Fermer</button>
  </div>
);
```

---

## 6. Sécurité — Règles absolues

| Règle | Statut |
|---|---|
| Clé sk_test_/sk_live_ uniquement en variable d'env PHP/N8N, jamais dans le code | ❌ À VÉRIFIER |
| Règle Firestore : champ `plan` non modifiable côté client | ❌ À DÉPLOYER EN PROD |
| stripe-checkout.php vérifie que l'user est authentifié Firebase avant de créer la session | ❌ À IMPLÉMENTER V2 |
| stripeCustomerId jamais passé en paramètre URL modifiable | ❌ À VÉRIFIER |
| Vérification signature webhook (whsec_...) dans N8N | ❌ À IMPLÉMENTER |
| Pas de bypass DEBUG en prod (ex: if DEBUG_MODE skip plan check) | ❌ À VÉRIFIER |

**Règle Firestore renforcée pour production** (à déployer avant go-live) :
```javascript
match /users/{userId} {
  allow read: if request.auth != null;
  allow write: if request.auth != null && request.auth.uid == userId
    && !('plan' in request.resource.data.diff(resource.data).affectedKeys());
}
```

---

## 7. Tests — Cartes Stripe

| Scénario | Numéro de carte | CVV | Date |
|---|---|---|---|
| Paiement réussi | 4242 4242 4242 4242 | 123 | 12/30 |
| Paiement refusé (fonds insuffisants) | 4000 0000 0000 9995 | 123 | 12/30 |
| 3DS requis | 4000 0025 0000 3155 | 123 | 12/30 |
| Expiration en cours de subscription | 4000 0000 0000 0069 | 123 | 12/30 |

**Tests CLI Stripe (si installé localement)** :
```bash
stripe listen --forward-to https://n8n.bimsmarter.eu/webhook/stripe-events
stripe trigger invoice.paid  # Simuler un paiement
stripe trigger customer.subscription.deleted  # Simuler une annulation
```

---

## 8. Timeline go-live — 4 semaines (deadline 1er avril 2026)

| Semaine | Dates | Objectif | Livrables |
|---|---|---|---|
| Sem. 1 | 8–14 mars | Infrastructure PHP | stripe-checkout.php, stripe-portal.php, Customer Portal configuré |
| Sem. 2 | 15–21 mars | N8N Webhooks | Workflow upgrade/downgrade opérationnel, tests events Stripe |
| Sem. 3 | 22–28 mars | UI & Tests E2E | Boutons upgrade apps, page success, modal downgrade, tests scénarios |
| Sem. 4 | 29 mar–1 avr | Go-Live | CGV, sk_live_, test paiement réel, monitoring |

---

## 9. Checklist go-live finale

### CRITIQUE (bloquant)
- [ ] Mode LIVE Stripe activé (sk_live_... dans toutes les configs PHP + N8N)
- [ ] Webhook endpoint recréé en mode LIVE dans Stripe Dashboard
- [ ] Règle Firestore renforcée déployée (plan non modifiable côté client)
- [ ] Service Account Firebase configuré dans N8N (production)
- [ ] CGV publiées et linkées dans Checkout Stripe
- [ ] Test paiement réel 1€ (puis remboursement immédiat)
- [ ] Test annulation via Customer Portal réel

### IMPORTANT (fortement recommandé)
- [ ] Smart Retries activés dans Stripe
- [ ] Email "Bienvenue Pro" envoyé via N8N/Brevo
- [ ] Branding Stripe configuré (logo + couleur #1E3A5F)
- [ ] Monitoring N8N : alertes si webhook échoue
- [ ] Backup Firestore activé (Firebase Blaze plan)
