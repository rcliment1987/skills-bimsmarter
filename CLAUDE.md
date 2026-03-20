# Skills BIMSmarter — Index Global

Ce fichier est la source de vérité pour tous les skills de Renaud Climent (rcliment1987).
Tous les skills sont hébergés sur GitHub : https://github.com/rcliment1987/skills-bimsmarter

## Règle principale

Avant toute action ou réponse, vérifie si un ou plusieurs skills s'appliquent à la situation.
Si un skill s'applique (même 1% de chance), tu DOIS :
1. Récupérer le SKILL.md correspondant via WebFetch depuis GitHub (URL raw)
2. Lire et suivre ses instructions AVANT de commencer à travailler

Base URL pour récupérer un skill :
```
https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/SKILL.md
```

## Superpowers — Workflow & Méthodologie

Base URL : `https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/superpowers/skills/{skill}/SKILL.md`

| Skill | Quand l'activer |
|---|---|
| **brainstorming** | AVANT toute création : features, composants, fonctionnalités. HARD-GATE : pas d'implémentation sans design approuvé |
| **test-driven-development** | AVANT d'écrire du code (feature ou bugfix). Le test échoue en premier, toujours |
| **systematic-debugging** | Dès qu'il y a un bug, test failure, ou comportement inattendu. AVANT de proposer un fix |
| **writing-plans** | Tâche multi-étapes avec specs/requirements. AVANT de toucher au code |
| **executing-plans** | Quand tu as un plan écrit à exécuter |
| **subagent-driven-development** | Exécuter des plans avec tâches indépendantes (préféré à executing-plans) |
| **dispatching-parallel-agents** | 2+ tâches indépendantes sans état partagé |
| **requesting-code-review** | Après une tâche complétée, feature majeure, ou avant un merge |
| **receiving-code-review** | Quand on reçoit du feedback de code review |
| **finishing-a-development-branch** | Implémentation terminée, tests OK, décision d'intégration |
| **using-git-worktrees** | Isolation de feature ou avant d'exécuter un plan |
| **verification-before-completion** | AVANT de dire que c'est terminé. Pas de claim sans preuve |
| **writing-skills** | Créer ou modifier des skills |

Agent code-reviewer : `https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/superpowers/agents/code-reviewer.md`

## Skills BIMSmarter — Domaine Métier

Base URL : `https://raw.githubusercontent.com/rcliment1987/skills-bimsmarter/main/{dossier}/SKILL.md`

| Skill | Dossier | Quand l'activer |
|---|---|---|
| **BIMSmarter Prompt** | `bimsmarter-prompt` | Création/optimisation prompts GID-Assistant, Workflow Generator, agents n8n, stack Mistral/Groq |
| **React Debug** | `react-debug` | Bugs React : focus loss, re-render, stale state, autocomplete, performance |
| **Mistral Proxy** | `mistral-proxy` | Erreurs 429, config proxy PHP, fallback Groq, rotation de clés |
| **BIMSmarter UI** | `bimsmarter-ui` | Nouveau composant/page, cohérence visuelle, design system (HSL, glass panel) |
| **BIMSmarter n8n** | `bimsmarter-n8n` | Workflows n8n, webhooks inter-apps, RAG Supabase, format JSON agents |
| **BIMSmarter Firebase** | `bimsmarter-firebase` | Firestore queries, security rules, scalabilité IFC, Auth |
| **Security Audit** | `bimsmarter-security-audit` | Audit sécurité, scan vulnérabilités, clés exposées |
| **Monétisation** | `systeme-monetisation-bimsmarter` | Plans Free/Pro/Enterprise, Stripe, quotas IA, PlanManager |
| **Gestion Entreprise** | `bimsmarter-gestion-entreprise` | TVA franchise, Partena, facturation, statut transfrontalier LU/BE |
| **Gestion Website** | `bimsmarter-gestion-website` | SEO/GEO bimsmarter.eu, Hostinger/Zyro, Schema.org, E-E-A-T |
| **SEO/GEO** | `bimsmarter-referencement-seo-geo` | Référencement moteurs génératifs (ChatGPT, Gemini, Perplexity) |
| **YouTube** | `bimsmarter-youtube` | Stratégie YouTube, titres viraux, descriptions SEO, miniatures |
| **Prompt Ingénieur** | `prompt-ingenieur` | Prompt engineering LLM-agnostic, templates A-E, chain-of-thought |
| **Orchestrating Swarms** | `orchestrating-swarms` | Orchestration multi-agents, TeammateTool, parallel/pipeline/swarm |

## Règles non-négociables

1. **Pas d'implémentation sans design** → brainstorming d'abord
2. **Pas de code sans test** → test-driven-development, le test échoue d'abord
3. **Pas de fix sans investigation** → systematic-debugging, cause racine d'abord
4. **Pas de "c'est terminé" sans preuve** → verification-before-completion
5. **Jamais "Vous avez absolument raison !"** → évaluation technique uniquement

## Comment utiliser un skill

1. Identifie le(s) skill(s) applicable(s) depuis les tableaux ci-dessus
2. Utilise WebFetch pour récupérer le SKILL.md depuis la raw URL GitHub
3. Suis les instructions du skill étape par étape
4. Plusieurs skills peuvent s'appliquer simultanément — les skills de processus (brainstorming, debugging) AVANT les skills d'implémentation
