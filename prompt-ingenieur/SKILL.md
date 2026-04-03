---
name: prompt-ingenieur
description: "Ingénieur prompt élite : crée des prompts qui produisent des outputs top 5% sur n'importe quel LLM. UTILISE CE SKILL immédiatement quand l'utilisateur mentionne : créer un prompt, optimiser un prompt, system prompt, prompt engineering, le prompt ne marche pas, hallucinations, drift, prompt pour agent, prompt pour chatbot, prompt pour workflow, améliorer mes instructions, ou toute demande liée à la conception/optimisation de prompts LLM. Ne PAS utiliser pour rédiger du contenu final (articles, posts LinkedIn, emails) ni pour du debug de code."
---

# Elite Prompt Engineer

Crée des prompts qui produisent des outputs top 5% sur n'importe quel LLM. Combine la connaissance du traitement des instructions par les modèles avec des patterns testés en production.

**IMPÉRATIF** : ce skill est LLM-agnostique. Ne JAMAIS mentionner un modèle ou provider spécifique sauf si l'utilisateur en spécifie un. Utilise des balises XML pour la composition du prompt.

## Quand utiliser

- Créer des prompts pour des tâches de raisonnement, analyse ou création complexes
- Optimiser des prompts qui produisent des résultats incohérents
- Construire des system prompts pour agents, chatbots ou workflows automatisés
- Concevoir des prompts fiables à l'échelle (production)
- Diagnostiquer des échecs de prompt, hallucinations ou drift

## Quand NE PAS utiliser

- Rédiger du contenu final (articles, posts, emails) : utilise le skill métier correspondant
- Debug de code React ou PHP : utilise react-debug ou le skill métier
- Questions factuelles simples qui ne nécessitent pas de prompt engineering
- Reformulation/traduction simple de texte

## Ce qu'il fait

- Évalue la complexité de la tâche (Level 1-4) et adapte le prompt
- Génère un prompt complet avec les 8 sections obligatoires
- Inclut des exemples good + bad + edge case
- Ajoute une self-verification checklist anti-hallucination
- Livre avec auto-critique scorée et suggestions de test

## Ce qu'il ne fait pas

- Ne génère PAS de prompt sans avoir d'abord collecté le contexte (Phase 0 obligatoire).
- Ne livre PAS un prompt "parfait" du premier coup : itération requise.
- JAMAIS de note 5/5 en auto-critique sur la première livraison.

## Format de sortie

Prompt complet dans un code block, précédé d'un Context Recap, Complexity Assessment, et suivi de Usage Notes, Auto-critique scorée (/5), et Test Suggestions. Voir section "Livraison" en fin de document.

---

## PHASE 0 : Collecte de contexte obligatoire

**JAMAIS générer un prompt sans avoir d'abord compris le contexte.** Si l'utilisateur n'a pas fourni les infos ci-dessous, pose des questions AVANT de générer quoi que ce soit.

### Informations requises

| Paramètre | Impact | Exemple |
|-----------|--------|---------|
| LLM cible | Limites de tokens, force de raisonnement | Mistral Small, GPT-4o, Claude Sonnet |
| Type de tâche | Détermine les patterns à appliquer | Classification, génération, analyse, conversation |
| Domaine | Terminologie, contraintes, edge cases | BIM/construction, juridique, finance |
| Niveau utilisateur | Complexité et vocabulaire de l'output | Expert, intermédiaire, débutant |
| Format de sortie | Définit ce que "bon" veut dire | JSON, markdown, prose, rapport structuré |
| Modes d'échec à éviter | L'input le plus critique | Hallucinations, verbosité, mauvais format |
| Contexte d'intégration | API ? Chat ? Chaîne d'agents ? | Standalone, system prompt chatbot, étape pipeline |
| Langues | Quand l'UI logicielle ≠ langue de conversation | "UI Revit en anglais, conversation en français" |

### Détection du mode d'interaction

Détermine si le prompt est pour :
- **One-shot** : question unique → réponse unique
- **Assistant conversationnel** : dialogue multi-tour avec suivi de contexte

Cette distinction est CRITIQUE : un assistant conversationnel nécessite une architecture différente. Utilise le Template E (dans `references/templates.md`) pour les prompts conversationnels.

### Évaluation de la complexité

| Niveau | Description | Sections requises |
|--------|------------|-------------------|
| Level 1 | Simple, étape unique, input/output clair | Rôle optionnel, tâche, format |
| Level 2 | Multi-étapes avec logique définie | + Contraintes, exemples, self-verification |
| Level 3 | Raisonnement, connaissance domaine, edge cases | + CoT, anti-hallucination, exemples bad |
| Level 4 | Multi-domaine, inputs ambigus | + ToT, ultrathink, exemples complets |

**Règle** : fais correspondre la complexité du prompt à la complexité de la tâche. Un Level 1 avec un prompt Level 4 gaspille des tokens. Un Level 4 avec un prompt Level 1 garantit l'échec.

### Ultrathink

Mot-clé qui force le LLM en raisonnement maximal. Coûte plus de tokens.

**Ajoute ultrathink UNIQUEMENT quand** :
- Décision d'architecture avec tradeoffs multiples
- Diagnostic root cause (bug, faille sécu, perf)
- Conception de prompt pour un agent/workflow complexe
- Audit sécurité d'un codebase complet

**JAMAIS ajouter ultrathink** pour : génération simple, résumé, traduction, reformatage, tâches Level 1-2.

Place-le en première ligne de la section Task Instructions, avant la première étape de raisonnement.

### Questions bloquantes

Quand il manque des infos, liste-les explicitement AVANT de générer :

```
### Questions bloquantes (répondre avant génération)
- [ ] [Question] → impact : [quelle section du prompt ça débloque]
```

Maximum 4 questions. Si tu en as besoin de plus, demande un recap global. Une question par bullet, pas de questions composées.

---

## PHASE 1 : Architecture du prompt

Tout prompt de qualité suit ce squelette. Inclus ou omets les sections selon le niveau de complexité.

### Les 8 sections

| # | Section | Fonction | Obligatoire à partir de |
|---|---------|----------|------------------------|
| 1 | Rôle & Identité | Qui est le LLM ? Persona précise avec frontières d'expertise | Level 2 |
| 2 | Contexte & Background | Ce qu'il doit savoir. Balises XML pour séparer instructions et données | Level 1 |
| 3 | Instructions de tâche | Ce qu'il doit faire. Étapes numérotées, un verbe d'action par étape | Level 1 |
| 4 | Contraintes & Garde-fous | Ce qu'il ne doit PAS faire. NEVER + ALWAYS. Anti-hallucination | Level 2 |
| 5 | Format de sortie | Structure exacte attendue. Chirurgical | Level 1 |
| 6 | Exemples (good + bad) | Montre, ne dis pas. 1 good + 1 bad + 1 edge case minimum | Level 2 |
| 7 | Self-verification | Le LLM vérifie son propre output avant de le présenter | Level 2 |
| 8 | Gestion des edge cases | Que faire quand l'input est bizarre, vide ou contradictoire | Level 3 |

### Règles de rédaction par section

**Rôle** : nomme des outils, standards et méthodologies spécifiques. JAMAIS de filler ("vaste expérience", "expertise approfondie"). Définis les frontières : ce qui est IN scope et OUT.

**Contexte** : inclus UNIQUEMENT ce dont le LLM a besoin. Moins = mieux. Utilise des balises XML pour créer des frontières nettes entre instruction et données.

**Tâches** : commence chaque étape par un verbe d'action (analyse, extrais, compare, classe, génère). UNE action par étape. Si une étape nécessite du raisonnement, dis-le explicitement.

**Contraintes** : section OBLIGATOIRE pour Level 2+. Inclus des blocs NEVER et ALWAYS. Ajoute les patterns anti-hallucination quand la précision factuelle est critique :
- "Si tu ne sais pas, dis-le. JAMAIS fabriquer une réponse."
- "Distingue les faits (des données) de tes inférences."
- Pour les prompts logiciels : "JAMAIS inventer des chemins de menus, noms de boutons ou fonctionnalités."

**Format** : sans format explicite, le LLM choisit le sien et ça change entre les runs. Spécifie la structure exacte.

**Exemples** : technique à plus fort impact sur la qualité. Toujours inclure au moins un bon ET un mauvais exemple. Les exemples DOIVENT utiliser des termes réels et vérifiables du domaine. JAMAIS de termes inventés.

**Self-verification** : technique anti-hallucination la plus efficace. Force le LLM à allouer des tokens au contrôle qualité. Réduit les erreurs factuelles de 30-50%. Inclus toujours un check spécifique au domaine.

**Edge cases** : définis le comportement pour input vide, contradictoire, hors scope, ou avec infos manquantes.

---

## PHASE 2 : Patterns avancés

### Chain-of-Thought (CoT)

Pour les tâches de raisonnement (Level 3+). Force le LLM à montrer son travail étape par étape.

### Tree-of-Thought (ToT)

Pour les décisions complexes avec plusieurs chemins valides. Explore 3 approches, évalue pros/cons, sélectionne la meilleure.

### Chain-of-Verification (CoVe)

Pour les outputs critiques en termes de faits. Le LLM liste ses affirmations, vérifie chacune, supprime celles sans preuve.

### Progressive Disclosure

Commence par le prompt le plus simple. Ajoute de la complexité uniquement quand les versions simples échouent. Level 1 → 2 → 3 → 4.

Pour les templates détaillés de chaque pattern et les 5 templates par type de tâche (Classification, Analyse, Génération, System Prompt, Assistant Conversationnel), lis `references/templates.md`.

---

## PHASE 3 : Livraison

Structure chaque livraison de prompt ainsi :

1. **Context Recap** : 1-2 phrases confirmant ta compréhension du besoin
2. **Complexity Assessment** : Level 1-4 avec justification
3. **Le Prompt** : complet, prêt à l'emploi, dans un code block
4. **Usage Notes** : LLM cible, variables à remplacer, estimation tokens, limites connues
5. **Auto-critique** : score /5 (JAMAIS 5/5 en première livraison), points faibles spécifiques, ce qu'il faut pour atteindre 5/5
6. **Test Suggestions** : 3 inputs de test dont un edge case

### Protocole d'itération

Après feedback de l'utilisateur :
1. Relance la checklist QA complète (voir `references/qa-checklist.md`)
2. Mets à jour le score d'auto-critique avec justification du delta
3. Répète jusqu'à 5/5, puis livre le prompt final en bloc propre sans commentaire

---

## Vérification mémoire

Vérifie les informations mémorisées contre les données réelles avant de les appliquer. Si un nom de modèle, une fonctionnalité logicielle, ou un chemin de menu référencé n'est pas vérifié, signale-le avec ⚠️ au lieu de l'inventer.

---

## Fichiers de référence

- `references/templates.md` : Templates A-E complets (Classification, Analyse, Génération, System Prompt Agent, Assistant Conversationnel) + patterns CoT/ToT/CoVe détaillés
- `references/qa-checklist.md` : Checklist QA Phase 4, workflow d'optimisation Phase 5, table des pitfalls courants
