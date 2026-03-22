# Skills BIMSmarter — Index Global

Ce fichier est la source de vérité pour tous les skills de Renaud Climent (rcliment1987).
Tous les skills sont hébergés sur GitHub : https://github.com/rcliment1987/skills-bimsmarter

---

## Règle principale

Avant toute action ou réponse, vérifie si un ou plusieurs skills s'appliquent à la situation.
Si un skill s'applique (même 1% de chance), tu DOIS :
1. Récupérer le SKILL.md correspondant via `curl` (Bash) depuis GitHub raw URL
2. Lire et suivre ses instructions AVANT de commencer à travailler

**Comment récupérer un skill :**
```bash
curl -s https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/SKILL.md
```

**Si le skill a un sous-fichier `references/` mentionné dans le SKILL.md :**
```bash
curl -s https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/references/{fichier}.md
```

> ⚠️ Toujours utiliser `curl -s` via l'outil Bash, jamais `WebFetch`. Raison : WebFetch passe le contenu par un modèle IA qui peut résumer ou tronquer les instructions. `curl -s` retourne le texte brut intégral — essentiel pour suivre un SKILL.md à la lettre.

---

## Skills BIMSmarter — Domaine Métier

Base URL : `https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/SKILL.md`

| Skill | Dossier | Quand l'activer |
|---|---|---|
| **BIMSmarter Prompt** | `bimsmarter-prompt` | Création/optimisation prompts GID-Assistant, Workflow Generator, agents n8n, stack Mistral/Groq |
| **React Debug** | `react-debug` | Bugs React : focus loss, re-render, stale state, autocomplete, performance |
| **Mistral Proxy** | `mistral-proxy` | Erreurs 429, config proxy PHP, fallback Groq, rotation de clés, déploiement nouvelle app |
| **BIMSmarter UI** | `bimsmarter-ui` | Nouveau composant/page, cohérence visuelle, design system (HSL, glass panel, dark theme) |
| **BIMSmarter n8n** | `bimsmarter-n8n` | Workflows n8n, webhooks inter-apps, RAG Supabase, format JSON agents |
| **BIMSmarter Firebase** | `bimsmarter-firebase` | Firestore queries, security rules, scalabilité IFC, Auth, structure collections |
| **Security Audit** | `bimsmarter-security-audit` | Audit sécurité, scan vulnérabilités, clés exposées. TOUJOURS avec orchestrating-swarms |
| **Monétisation** | `systeme-monetisation-bimsmarter` | Plans Free/Pro/Enterprise, Stripe, quotas IA, PlanManager. Réf: `references/stripe-n8n-integration.md` |
| **Gestion Entreprise** | `bimsmarter-gestion-entreprise` | TVA franchise, Partena, facturation, statut transfrontalier LU/BE. Réf: `references/profil-entreprise.md` |
| **Gestion Website** | `bimsmarter-gestion-website` | SEO/GEO bimsmarter.eu, Hostinger/Zyro, Schema.org, E-E-A-T. Réf: `references/geo-seo-strategie.md` |
| **SEO/GEO** | `bimsmarter-referencement-seo-geo` | Référencement moteurs génératifs (ChatGPT, Gemini, Perplexity) |
| **YouTube** | `bimsmarter-youtube` | Stratégie YouTube, titres viraux, descriptions SEO, miniatures, repurposing LinkedIn |
| **Prompt Ingénieur** | `prompt-ingenieur` | Prompt engineering LLM-agnostic, templates A-E, chain-of-thought, anti-hallucination |
| **orchestrating-swarms** | Tâches longues/itératives, multi-agents, audits sécurité |
| **bimsmarter-check-list-app** | Avant de créer une application ou un code, avant de modifier une applicaiton ou un code |
---

## Superpowers — Workflow & Méthodologie

Base URL : `https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/superpowers/skills/{skill}/SKILL.md`

| Skill | Quand l'activer |
|---|---|
| **dispatching-parallel-agents** | 2+ tâches indépendantes sans état partagé |
| **systematic-debugging** | Bug, test failure, comportement inattendu — AVANT de proposer un fix |
| **subagent-driven-development** | Exécuter des plans avec tâches indépendantes en parallèle |
| **writing-plans** | Tâche multi-étapes complexe avec specs/requirements |
| **executing-plans** | Quand tu as un plan écrit à exécuter étape par étape |
| **verification-before-completion** | AVANT de dire que c'est terminé — pas de claim sans preuve |
| **requesting-code-review** | Après une tâche complétée ou feature majeure |
| **receiving-code-review** | Quand on reçoit du feedback de code review |
| **brainstorming** | Conception d'une nouvelle feature ou architecture significative |
| **finishing-a-development-branch** | Implémentation terminée, décision merge/PR/discard |
| **using-git-worktrees** | Isolation de feature avant exécution d'un plan |
| **writing-skills** | Créer ou modifier des skills |

Agent code-reviewer : `https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/superpowers/agents/code-reviewer.md`

---

## Règles de travail

### Obligatoires (jamais déroger)
1. **Pas de fix sans investigation** → systematic-debugging, cause racine d'abord
2. **Pas de "c'est terminé" sans preuve** → verification-before-completion
3. **Jamais "Vous avez absolument raison !"** → évaluation technique, esprit critique
4. **Éditions chirurgicales uniquement** → format AVANT/APRÈS, pas de réécriture complète
5. **Pas de scope creep** → ne toucher que ce qui est demandé

### Recommandées (selon le contexte)
6. **Investigation avant fix** → systematic-debugging pour les bugs complexes
7. **Multi-agents pour les tâches lourdes** → orchestrating-swarms pour économiser le contexte
8. **Design avant implémentation** → brainstorming pour les nouvelles features significatives
9. **Review après implémentation** → requesting-code-review pour les changements critiques (sécurité, Firebase, proxy)

### Adaptations au contexte BIMSmarter
- **Stack** : React 18 (Babel standalone, single-file HTML), Firebase, PHP proxy — pas de build step, pas de framework de test
- **TDD formel non applicable** : pas de Jest/Vitest sur du Babel standalone. À la place : vérifier manuellement que le fix fonctionne, tester les cas limites, confirmer avec preuve avant de valider
- **Git worktrees optionnels** : utile pour les gros changements, pas obligatoire pour les éditions chirurgicales

---

## Priorité des skills

Quand plusieurs skills s'appliquent :
1. **Skills de processus d'abord** : systematic-debugging, brainstorming, writing-plans
2. **Skills métier ensuite** : react-debug, bimsmarter-firebase, mistral-proxy, etc.
3. **Skills de finalisation en dernier** : verification-before-completion, requesting-code-review

---

## Comment utiliser un skill

```bash
# 1. Identifier le skill applicable depuis les tableaux ci-dessus

# 2. Récupérer le SKILL.md
curl -s https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/SKILL.md

# 3. Si le skill mentionne un fichier references/, le récupérer aussi
curl -s https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/references/{fichier}.md

# 4. Suivre les instructions du skill étape par étape
```

> **Important** : Ce repo doit rester PUBLIC sur GitHub pour que les raw URLs fonctionnent. Si le repo passe en privé, tous les fetch échoueront.
