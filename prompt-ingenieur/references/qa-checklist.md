# QA Checklist & Pitfalls — Référence

Lis ce fichier avant de livrer un prompt (Phase 3) et lors de chaque itération.

---

## Checklist QA complète

Exécute TOUS les checks pertinents avant de livrer un prompt.

### Checks structurels

- [ ] Le niveau de complexité correspond à la tâche
- [ ] Le mode d'interaction (one-shot vs conversationnel) est correctement identifié
- [ ] Toutes les sections requises pour le niveau sont présentes
- [ ] Les balises XML sont cohérentes et correctement imbriquées
- [ ] Aucune section ne contredit une autre
- [ ] Les instructions utilisent des verbes d'action, une action par étape

### Checks de qualité du contenu

- [ ] Le rôle nomme des outils/méthodes/standards concrets (pas de filler "vaste expérience")
- [ ] Les tâches sont des ACTIONS TESTABLES, pas des objectifs vagues
- [ ] Les contraintes incluent des règles NEVER et ALWAYS
- [ ] Au moins un bon ET un mauvais exemple (Level 2+)
- [ ] Les exemples utilisent des termes RÉELS et VÉRIFIABLES (pas de noms de menus inventés)
- [ ] La self-verification est présente (Level 2+) avec des checks spécifiques au domaine
- [ ] La gestion des edge cases est définie
- [ ] Le format de sortie est explicite et non ambigu

### Checks anti-hallucination

- [ ] Les instructions disent explicitement quoi faire quand l'info manque
- [ ] La self-verification inclut un check de traçabilité des faits
- [ ] Les contraintes interdisent d'inventer de l'information
- [ ] Pour les prompts logiciels : les règles anti-hallucination logicielle sont incluses
- [ ] Un exemple edge_case montre comment gérer l'incertitude

### Checks langue & localisation

- [ ] Si langue UI logicielle ≠ langue conversation : rules explicites incluses
- [ ] Les chemins de menus sont dans la bonne langue (celle du logiciel)
- [ ] Les explications sont dans la langue préférée de l'utilisateur

### Checks d'efficacité tokens

- [ ] Pas d'instructions redondantes ou répétées
- [ ] Le contexte ne contient que de l'information pertinente
- [ ] Les exemples sont concis mais complets
- [ ] La progressive disclosure a été envisagée (version simple testée d'abord ?)

### Checks de production (prompts intégrés dans des apps/APIs)

- [ ] Les variables utilisent un nommage clair : {user_input}, {context_data}, {output_type}
- [ ] Le prompt gère un input vide/null (erreur gracieuse)
- [ ] L'output est parseable (JSON/structuré) si nécessaire en aval
- [ ] Testé avec au moins 3 inputs divers dont un adversarial

---

## Workflow d'optimisation

Quand tu améliores un prompt existant :

1. **MESURE** : lance le prompt sur 5-10 inputs divers. Note les échecs.
2. **DIAGNOSTIQUE** : catégorise les échecs :

| Type d'échec | Section à corriger |
|-------------|-------------------|
| Erreurs de format | output_format |
| Hallucinations | self_verification + constraints + anti-hallucination |
| Info manquante | Spécificité des task instructions |
| Outputs incohérents | Ajouter des exemples |
| Verbeux / hors-sujet | Ajouter des contraintes négatives |
| Fonctionnalités logicielles inventées | Anti-hallucination logiciel |
| Mauvaise langue pour les menus | language_rules |

3. **CORRIGE** : change UNE chose à la fois.
4. **TESTE** : relance sur les MÊMES inputs + 2 nouveaux.
5. **COMPARE** : le fix a-t-il amélioré sans casser d'autres cas ?
6. **ITÈRE** : répète jusqu'à >90% de test cases passés.

**Règle d'or** : si tu ne peux pas expliquer POURQUOI un changement devrait aider, ne le fais pas.

---

## Table des pitfalls courants

| Pitfall | Pourquoi ça échoue | Fix |
|---------|-------------------|-----|
| "Be helpful and accurate" | Trop vague, chaque LLM essaie déjà | Spécifie ce que "accurate" signifie pour CETTE tâche |
| "You have vast experience" | Filler, zéro signal | Nomme les outils, méthodes et workflows concrets |
| Mur de texte sans structure | Le LLM perd le focus | Balises XML + étapes numérotées |
| Que des bons exemples | Le LLM ne sait pas quoi éviter | Ajoute des mauvais exemples avec explications |
| Exemples avec des étapes logicielles inventées | Apprend au LLM qu'halluciner est OK | Utilise UNIQUEMENT des références logicielles réelles |
| "Do your best" | Pas de barre de qualité définie | Définis des critères de succès explicites |
| Copier-coller un template sans adapter | Mismatch de contexte | Adapte toujours au cas spécifique |
| CoT sur des tâches simples | Gaspille des tokens, peut réduire la précision | Adapte la technique à la complexité |
| Pas de self-verification | Le LLM livre un premier jet | Ajoute toujours une vérification Level 2+ |
| Ignorer le LLM cible | Un prompt GPT-4 ne marchera pas pareil sur Mistral 7B | Adapte aux capacités du modèle |
| Bourrer du contexte inutile | Dilue l'attention | Inclus UNIQUEMENT ce dont le LLM a besoin |
| Pas de contraintes négatives | Le LLM comble les trous avec des hallucinations plausibles | Dis-lui ce qu'il ne doit PAS faire |
| Mélanger les langues UI | Menus dans la mauvaise langue | Ajoute une section language_rules explicite |
| Template one-shot pour un assistant conversationnel | Rate le suivi, le contexte, le comportement adaptatif | Utilise Template E |
| Objectifs vagues comme tâches | "Provide advice" est non testable | Écris des actions testables |
| JAMAIS 5/5 en première livraison | Empêche l'itération, faux sentiment de complétude | Score honnête + points faibles spécifiques |

---

## Matrice Complexité → Sections requises

| Section | Level 1 | Level 2 | Level 3 | Level 4 |
|---------|---------|---------|---------|---------|
| Rôle & Identité | Optionnel | Oui | Oui | Oui |
| Contexte | Minimal | Oui | Oui | Oui |
| Instructions tâche | 1-2 étapes | Étapes numérotées | + CoT | + CoT + ToT |
| Contraintes (négatives) | Optionnel | Oui | + anti-hallucination | Complet |
| Anti-hallucination logiciel | Non | Si logiciel impliqué | Oui | Oui |
| Règles de langue | Non | Si contexte bilingue | Oui | Oui |
| Format de sortie | Bref | Explicite | Explicite | + schéma |
| Exemples (good) | Optionnel | 1-2 | 2-3 | 3-5 |
| Exemples (bad) | Non | 1 | 1-2 | 2-3 |
| Self-verification | Non | Basique (3 checks) | Complète (5-7) | + domain-specific |
| Edge cases | Non | Basique | Complet | Complet |
| **ultrathink** | ❌ | ❌ | ✅ Si multi-hypothèses | ✅ Toujours |
