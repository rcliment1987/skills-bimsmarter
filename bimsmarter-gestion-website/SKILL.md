---
name: bimsmarter-gestion-website
description: >
  Assistant expert pour la gestion, l'optimisation SEO/GEO et la maintenance du site bimsmarter.eu.
  UTILISE CE SKILL immédiatement — sans demander — dès que Renaud parle de :
  son site web, bimsmarter.eu, Hostinger, Zyro, l'éditeur de site, le code personnalisé du site,
  le référencement, SEO, GEO, les balises HTML du site, le chatbot du site, les JSON-LD,
  les Schema.org, les pages piliers, le contenu du site, les métadonnées, E-E-A-T,
  la visibilité IA (ChatGPT, Gemini, Perplexity), les cookies, le bandeau RGPD,
  les améliorations du site, la roadmap site, ou tout ce qui touche à la présence web de BIMsmarter.
---

# BIMsmarter — Gestion & Optimisation Site Web

## CONTEXTE TECHNIQUE

**Hébergement :** Hostinger plan Premium (9,99€/mois)
- Éditeur : Zyro (Hostinger Website Builder) — PAS WordPress
- CDN gratuit inclus
- Weekly backups
- Standard DDoS protection
- ⚠️ PAS de Node.js sur l'hébergement mutualisé
- Code personnalisé intégré via l'éditeur Zyro (section "Custom Code")

**VPS séparé :** KVM2 Hostinger + Coolify → héberge N8N uniquement

**URL principale :** https://bimsmarter.eu
**Chat IA :** https://chat.bimsmarter.eu (ouvre en popup via le widget)

---

## CODE CUSTOM ACTUELLEMENT EN PLACE

Le site intègre 3 blocs de code custom :

### 1. Firebase App Check (sécurité)
```javascript
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
// Config Firebase projet "bimsmarter"
// reCAPTCHA Enterprise : 6LdlmmAsAAAAAF6JxWMcyR5ulw0uJESsLsd6fxD7
// App Check activé avec token auto-refresh
```

### 2. Schema.org JSON-LD (Organization)
Déjà en place : `@type: Organization` avec `name`, `url`, `logo`, `description`, `address`, `knowsAbout` (ISO 19650, BIM, GID, IFC...), `founder` (Renaud Climent), `hasOfferCatalog` (4 services).

### 3. Chatbot Widget
- FAB fixe bottom-right (bouton 🤖 avec animation pulse cyan)
- Bubble d'accueil auto après 2s, ferme après 10s
- Ouvre https://chat.bimsmarter.eu en popup (420×620px)
- Style : dark theme, gradient cyan/purple

### 4. Bannière Cookies RGPD
- IDs : `bs-consent-banner`, `bs-btn-accept`, `bs-btn-deny`
- Stockage `localStorage` clé `bs_consent_v1`
- ⚠️ Le HTML de la bannière doit exister dans le DOM Zyro (le JS gère juste la logique)

---

## RÈGLES ABSOLUES ZYRO/HOSTINGER

1. **PAS de framework** — HTML/CSS/JS vanilla uniquement dans le code custom
2. **PAS de Node.js** — tout doit fonctionner côté navigateur
3. **Injection via "Custom Code"** — head ou body selon le besoin
4. **Zyro génère le HTML** — on ne peut pas modifier la structure principale, seulement ajouter du code custom
5. **Tester en production** — pas d'environnement staging pour Zyro
6. **Google Search Console** — à utiliser pour valider les changements SEO

---

## STRATÉGIE GEO/SEO — PRINCIPES CLÉS (2026)

Lire `references/geo-seo-strategie.md` pour le détail complet.

**Priorité absolue : GEO (Generative Engine Optimization)**
- 58,5% des recherches = zéro-clic (93% avec IA générative)
- 97% des citations IA viennent du top 20 Google → être visible là
- Trafic IA : taux de conversion 4-5x supérieur au SEO classique

**Les 5 piliers GEO validés :**
1. Structure H1/H2/H3 logique avec entités clés
2. Format Q&A conversationnel + TL;DR en haut de page
3. Schema.org JSON-LD exhaustif (FAQPage, Article, SoftwareApplication)
4. E-E-A-T : bio auteur, credentials, case studies chiffrés
5. Fact Density : données chiffrées toutes les 150-200 mots

**Entités clés à toujours inclure :**
`BIM`, `ISO 19650`, `Luxembourg`, `Belgique`, `Benelux`, `GID`, `IFC`, `automatisation BIM`, `Audit BIM`

**⚠️ Jamais :** données Sweco, noms clients réels, métriques internes Sweco

---

## ROADMAP D'OPTIMISATION

> Cette roadmap est évolutive. Quand un point est traité, Renaud dit "retiré le point X" et le skill est mis à jour.

### 🔴 CRITIQUE — À faire en priorité

- [ ] **H1 à corriger** : remplacer "Ne laissez plus les normes et la désorganisation tuer votre productivité" → "Automatisation BIM & IA pour l'Ingénierie au Luxembourg et en Belgique"
- [ ] **Absence de H2** : ajouter structure H2/H3 logique (Zyro permet d'éditer les titres)
- [ ] **H6 partout** : les sections "Smart Tools", "Audit Flash" sont en H6 → passer en H3
- [ ] **Meta description absente** : ajouter `<meta name="description" content="Cabinet de conseil BIM & automatisation IA au Luxembourg et Belgique. Solutions ISO 19650, générateur nommage, audits maquettes BIM. Gagnez 10-15h/semaine.">`
- [ ] **FAQPage Schema** : ajouter JSON-LD FAQPage sur chaque page de service (5 Q&A minimum par page)

### 🟠 IMPORTANT — Mois 1-2

- [ ] **Page About/Expertise** : créer page avec bio Renaud + credentials + JSON-LD Person
- [ ] **Alt text images** : audit + remplissage de tous les alt text manquants
- [ ] **Article Pilier #1** : "ISO 19650 complet au Benelux : du EIR au BEP (2026)" — voir template dans références
- [ ] **Article Pilier #2** : "DIU Numérique en Belgique 2026 : Obligations, Processus BIM"
- [ ] **Article Pilier #3** : "Audit BIM : Identifier et Corriger 80% des Erreurs d'Information"
- [ ] **Case study #1 anonymisé** : projet type avec chiffres d'impact (ROI, heures économisées)
- [ ] **robots.txt** : vérifier que OAI-SearchBot (ChatGPT) est autorisé

### 🟡 OPTIMISATION AVANCÉE — Mois 2-4

- [ ] **Article Pilier #4** : "Automatisation RFI & Clash Detection avec IA : Économiser 500h/an"
- [ ] **Article Pilier #5** : "Nommage ISO 19650 en Belgique vs Luxembourg : Différences Critiques"
- [ ] **Article Schema** : ajouter JSON-LD Article sur chaque article publié
- [ ] **Backlinks qualité** : 10-15 liens DA>40 (Palmer Consulting, BuildWise, BIM-synthèse, CRTI-B)
- [ ] **LinkedIn Pulse** : 1-2 articles/mois longs (1500+ mots), hashtags #BIM #ISO19650
- [ ] **llms.txt** : créer fichier à la racine avec instructions pour les LLMs

### 🟢 MONITORING & MESURES

- [ ] **Baseline GA4** : documenter trafic organique actuel avant optimisations
- [ ] **Google Search Console** : soumettre URLs après chaque modif, tracker AI Overviews
- [ ] **Citation IA** : tester "BIM automation Luxembourg" dans ChatGPT/Perplexity chaque mois
- [ ] **Outils tracking** : évaluer Profound ou Azoma pour monitoring citations IA

---

## WORKFLOW POUR LES MODIFICATIONS

### Modifier du code custom dans Zyro :
1. Hostinger Dashboard → Websites → bimsmarter.eu → Edit
2. Dans l'éditeur : Settings → Custom Code
3. Coller le code dans Head (pour JSON-LD, meta) ou Body (pour scripts)
4. Save + Publish
5. Valider avec Google Rich Results Test : https://search.google.com/test/rich-results

### Ajouter/modifier du contenu :
1. Éditer directement dans l'éditeur Zyro (drag & drop)
2. Pour les titres : cliquer sur le texte → changer le type (H1/H2/H3)
3. Republier après chaque modif

### Valider le Schema.org :
- https://validator.schema.org/
- https://search.google.com/test/rich-results

---

## MÉTRIQUES À SUIVRE

| Métrique | Outil | Cible 6 mois |
|---|---|---|
| Sessions organiques/mois | GA4 | +100% |
| AI Overview citations | Search Console | 5-8 |
| E-E-A-T score | Audit manuel | 4/5 |
| ChatGPT mentions | Test manuel mensuel | 3-5 |
| Backlinks qualité | Ahrefs/Semrush | +10-15 |

---

## CHECKLIST AVANT PUBLICATION CONTENU

- [ ] H1 contient entités : BIM + norme + géo (Lux/Be) + action
- [ ] H2/H3 structurés en questions sémantiques
- [ ] TL;DR (40-80 mots) en haut — réponse directe pour IA
- [ ] Minimum 5 FAQ en bas — Format Q&A pour GEO
- [ ] FAQPage Schema JSON-LD ajouté
- [ ] Author bio avec credentials visibles
- [ ] Minimum 1 chiffre d'impact (%, heures, ROI)
- [ ] Alt text sur toutes les images
- [ ] Meta description : 155 chars, mot-clé principal
- [ ] Zéro data Sweco (anonymisation totale)
- [ ] Submit URL dans Search Console après publication

---

## RÉFÉRENCE RAPIDE JSON-LD À RÉUTILISER

Pour les FAQs (copier-adapter) :
```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [{
    "@type": "Question",
    "name": "Qu'est-ce que [sujet] ?",
    "acceptedAnswer": {
      "@type": "Answer",
      "text": "Réponse directe en 2-3 phrases avec entités clés."
    }
  }]
}
```

Pour les articles :
```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Titre article",
  "author": {
    "@type": "Person",
    "name": "Renaud Climent",
    "url": "https://bimsmarter.eu/expertise"
  },
  "publisher": {
    "@type": "Organization",
    "name": "BIMsmarter",
    "logo": "https://assets.zyrosite.com/BFLf5mME5ahXDCY4/bim_smarter_logo_transp-DqfITZrcZnCwoT7i.png"
  },
  "datePublished": "YYYY-MM-DD",
  "dateModified": "YYYY-MM-DD"
}
```

---

Pour les stratégies GEO détaillées, les templates d'articles piliers, et les exemples de structures H1/H2/H3 : lire `references/geo-seo-strategie.md`
