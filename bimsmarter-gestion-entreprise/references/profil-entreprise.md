# Profil Entreprise BIMsmarter — Données de référence

## Identité entreprise

| Champ | Valeur |
|-------|--------|
| Nom entreprise | BIMsmarter |
| Exploitant | Renaud Climent (personne physique) |
| N° entreprise | BE 1034.675.145 |
| N° unité établissement | 2.385.401.828 |
| N° national (NISS) | 870412.411.17 |
| Adresse | Grand-Rue, Châtillon 75, 6747 Saint-Léger (Belgique) |
| Téléphone | +32 498 023 078 |
| Email | renaudcliment@outlook.com |
| IBAN | BE41 6507 2048 5210 |
| BIC | REVOBEB2 |
| Date de début activité | 01/04/2026 |

---

## Statut juridique et social

- **Forme** : Personne physique (entreprise individuelle)
- **Statut social** : Indépendant à titre **complémentaire** — catégorie "Autres prestations suffisantes"
- **Caisse d'assurances sociales** : Partena Professional (référence client : N00 870412.411.17)
- **Doccle-token** : 5BG2K
- **Activité principale salariée** : Salarié au Luxembourg (secteur privé, au moins à mi-temps) — statut transfrontalier
- **Statut Starter** : Oui — cotisations provisoires forfaitaires les 3 premières années

**Contact Partena :**
- Tél. : 081/25.36.10 (lundi-vendredi 8h-17h)
- Email : independant@partena.be
- Web : www.partena-professional.be
- Portail : My Social Security Manager

---

## Régime TVA

- **Régime** : Franchise de taxe pour petites entreprises (Art. 56bis §1 Code TVA belge)
- **Plafond CA annuel** : 25.000€ (hors TVA)
- **CA présumé déclaré** : 24.999€
- **Date identification TVA** : 01/04/2026
- **Centre TVA compétent** : Centre PME Namur - Gestion Team 6, Rue des Thermes-Romains 74, 6700 Arlon — 02 572 57 57
- **Référence dossier TVA** : 1034675145E604A202602231425
- **Mention OBLIGATOIRE sur toutes les factures** : *"TVA non applicable, franchise de taxe pour les petites entreprises"*

**⚠️ RÈGLE ABSOLUE** : Renaud ne facture JAMAIS de TVA et ne peut PAS déduire la TVA sur ses achats.

---

## Activités enregistrées (codes NACEBEL)

| Code | Description | Type |
|------|-------------|------|
| 62100 | Activités de programmation informatique | Principal |
| 62200 | Conseil en informatique et gestion d'installations | Principal |
| 62900 | Autres activités de service informatique | Principal |
| 63910 | Activités de portail de recherche sur le web | Principal |
| 74999 | Autres activités professions libérales scientifiques | Principal |
| 63100 | Infrastructure informatique, hébergement | Secondaire |
| 82990 | Autres activités de soutien aux entreprises | Secondaire |
| 85599 | Autres formes d'enseignement | Secondaire |

---

## Stripe — Gestion financière

**Clés Stripe** : ⚠️ Stockées séparément — NE PAS afficher dans les réponses.

**Règles comptables Stripe :**
1. CA à déclarer = montant brut payé par le client (pas le montant net versé)
2. Frais Stripe = charges professionnelles déductibles (frais bancaires/financiers)
3. Le plafond 25.000€ se calcule sur le CA brut

**Exemple :**
- Client paie 500€ → CA = 500€ (même si Stripe verse 480€ après frais)
- Les 20€ de frais = charge déductible

**Alerte OSS** : Si des abonnements Stripe sont vendus à des particuliers dans d'autres pays UE et que le CA dépasse 10.000€/an en Europe → risque OSS (guichet unique TVA UE) → consulter comptable.

---

## Contexte business BIMsmarter

**Description** : Plateforme SaaS/portail web BIM gratuit (modèle lead-gen) ciblant les BIM coordinateurs Luxembourg/Benelux.

**4 applications React + Firebase :**
1. **Document Generator** — Création docs ISO 19650, export CSV/Excel/PDF
2. **GID-Assistant** — Chatbot Mistral API, données GID Luxembourg
3. **Workflow Generator** — Gestion workflows BIM, intégration n8n
4. **IFC Viewer + Dashboard** — Audit IFC, audit IDS, BCF topics

**Stack technique** :
- Frontend : React + Firebase (Firestore, Auth, Storage)
- APIs : Mistral API (mistral-small-latest), Perplexity, Groq fallback
- Infra : Hostinger VPS + Coolify, n8n (automation), Supabase (IFC)
- Paiements : Stripe (abonnements récurrents)

**Modèle tarifaire** : Freemium → Club Premium (en cours de développement)

---

## Documents officiels disponibles

| Document | Statut | Référence |
|----------|--------|-----------|
| Extrait BCE (Banque-Carrefour Entreprises) | ✅ Obtenu | Extrait 23/02/2026 |
| Demande identification TVA (SPF Finances) | ✅ Soumis | Réf. 1034675145E604A202602231425 |
| Attestation affiliation Partena | ✅ Obtenu | Réf. N00 870412.411.17 |
| Attestation de carrière Partena | ✅ Obtenu | Date début 01/04/2026 |
| Déclaration conventions internationales | ✅ Signé | Activité salariée Luxembourg |
| Mandat SEPA Partena | ✅ À compléter | IBAN à renseigner |

---

## Spécificité transfrontalier LU/BE

- Renaud travaille comme salarié au Luxembourg (Sweco Luxembourg)
- Il est indépendant complémentaire en Belgique
- **Impact fiscal** : les revenus BIMsmarter s'ajoutent aux revenus luxembourgeois pour déterminer la tranche d'imposition en Belgique
- Partena a transmis le dossier à l'INASTI pour vérification de l'assujettissement (procédure normale — déclaration déjà signée et retournée)
- Statut confirmé : **complémentaire** (salarié LU = activité principale)

---

## Estimation financière rapide

**Cotisations sociales (statut Starter complémentaire) :**
- Années 1-3 : cotisations provisoires forfaitaires (faibles — à vérifier sur partena-professional.be avec le simulateur)
- Régularisation après 2 ans sur revenus réels

**Mise de côté recommandée (hors cotisations) :**
- Impôts belges (sur revenus indépendant) : ~30-40% en provision de sécurité
- Raison : les revenus s'ajoutent aux revenus LU → tranche haute probable

**Charges déductibles BIMsmarter :**
- Firebase (Blaze), Supabase, Hostinger VPS, Coolify
- Abonnements Claude.ai, Mistral API
- Frais Stripe (transactions)
- Matériel informatique (pro-rata)
- Formation professionnelle
