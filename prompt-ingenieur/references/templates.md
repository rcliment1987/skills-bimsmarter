# Templates de Prompts — Référence détaillée

Lis ce fichier quand tu dois appliquer un template spécifique. Sélectionne le template qui correspond au type de tâche identifié en Phase 0.

---

## Template A : Classification / Extraction

```xml
<role>You are a [domain] specialist who classifies [items] with precision.</role>

<context>[Background information]</context>

<task>
Classify the following [input type] into exactly one of these categories:
- Category A: [definition + boundary]
- Category B: [definition + boundary]
- Category C: [definition + boundary]
</task>

<constraints>
- If the input fits multiple categories, choose the MOST specific one and note the ambiguity
- If the input fits NO category, classify as "Unclassified" with explanation
- NEVER create new categories
</constraints>

<examples>
<example type="good">
<input>[Representative input]</input>
<output>[Expected output]</output>
<why_good>[Specific quality criteria met]</why_good>
</example>

<example type="bad">
<input>[Same or similar input]</input>
<output>[Common wrong output]</output>
<why_bad>[Specific failure mode]. The correct approach would be [correction].</why_bad>
</example>

<example type="edge_case">
<input>[Tricky input]</input>
<output>[Expected handling]</output>
<why_good>[How edge case is handled correctly]</why_good>
</example>
</examples>

<output_format>
Category: [chosen category]
Confidence: [HIGH/MEDIUM/LOW]
Reasoning: [one sentence]
</output_format>

<self_verification>
□ Classification matches one of the defined categories exactly
□ Confidence level is justified by the reasoning
□ Edge cases (multi-category, no-category) are handled explicitly
</self_verification>

<input_data>
[The actual input to classify]
</input_data>
```

---

## Template B : Analyse / Raisonnement

```xml
<role>You are a senior [domain] analyst.</role>

<context>[Background, objectives, constraints]</context>

<task>
Analyze the following [input] by completing these steps:

1. Identify the key elements (list them)
2. Evaluate each element against [criteria]
3. Identify relationships, patterns, or anomalies
4. Synthesize findings into actionable insights
5. Provide recommendations ranked by impact
</task>

<constraints>
- Distinguish facts (from data) from inferences (your reasoning)
- Do NOT recommend actions outside the scope of [domain]
- Flag any data quality issues that affect your analysis
- If you don't know something, say "Information not available" — NEVER fabricate
</constraints>

<examples>
[1 good analysis + 1 bad analysis showing common pitfalls]
</examples>

<output_format>
[Structured format matching the task steps]
</output_format>

<self_verification>
□ Every finding is supported by specific data points
□ Recommendations are actionable and within scope
□ Confidence levels reflect actual certainty
□ No speculation presented as fact
</self_verification>

<input_data>
[The actual data to analyze]
</input_data>
```

---

## Template C : Génération / Création

```xml
<role>You are an expert [content type] creator for [audience].</role>

<context>
Purpose: [why this content exists]
Audience: [who will read/use it]
Tone: [specific tone descriptors]
Length: [target word/paragraph count]
</context>

<task>
Create [content type] about [topic] that achieves [specific goal].
</task>

<constraints>
- Stay within [word count] words
- Do NOT use: [banned terms, clichés, tones to avoid]
- Do NOT include: [types of content to exclude]
- Match the style demonstrated in the examples
</constraints>

<style_examples>
<good_style>[Example of desired writing style]</good_style>
<bad_style>[Example of what to avoid, with explanation]</bad_style>
</style_examples>

<self_verification>
□ Tone matches [specified tone] throughout
□ Length is within [target] ± 10%
□ No banned terms or excluded content
□ [Content-specific quality check]
</self_verification>

<edge_cases>
- If topic is too broad: narrow to [default scope] and state the limitation
- If conflicting style requirements: prioritize [primary constraint]
- If insufficient context: flag gaps before generating
</edge_cases>
```

---

## Template D : System Prompt Agent / Chatbot (One-Shot)

```xml
<identity>
You are [name/role] — [one-line description of purpose].
You serve [target users] by [core function].
</identity>

<capabilities>
You CAN:
- [Capability 1 with scope]
- [Capability 2 with scope]
- [Capability 3 with scope]

You CANNOT:
- [Limitation 1 — be explicit]
- [Limitation 2 — be explicit]
</capabilities>

<behavioral_rules>
1. [Rule about tone/style]
2. [Rule about handling unknowns]
3. [Rule about handling errors]
4. [Rule about escalation/boundaries]
5. [Rule about data handling]
</behavioral_rules>

<response_format>
[Default format for responses]
</response_format>

<edge_cases>
- If user asks something outside your scope: [instruction]
- If user provides contradictory information: [instruction]
- If user seems frustrated or confused: [instruction]
- If you are uncertain about your answer: [instruction]
</edge_cases>

<examples>
[2-3 example interactions showing ideal behavior, including one tricky scenario]
</examples>
```

---

## Template E : Assistant Conversationnel (Multi-Tour)

Utilise ce template quand l'utilisateur veut un assistant quotidien, pas un exécuteur de tâche one-shot. Architecture fondamentalement différente des Templates A-D.

```xml
<identity>
You are [name/role] — a hands-on [domain] assistant who works alongside the user daily.
You combine expertise in [tool 1], [tool 2], and [tool 3] to help with real-world workflows.
</identity>

<user_profile>
The user is a [role] with [level] experience in [tools].
They work with:
- [Tool 1] version [X] — UI language: [language]
- [Tool 2] — used for [specific purpose]
Their typical tasks include: [list of 3-5 recurring tasks]
</user_profile>

<language_rules>
[INCLURE quand la langue UI logicielle ≠ langue de conversation]
- All software menu paths, command names, dialog labels MUST be in [UI language]
- All explanations and commentary MUST be in [conversation language]
- Format software references in backticks: `File > Export > IFC`
</language_rules>

<interaction_style>
- Be conversational, direct, and practical — no lectures
- Give the ACTIONABLE answer first, explain WHY after if useful
- When the user describes a problem, ask ONE diagnostic question before proposing a solution (unless obvious)
- Proactively suggest next steps when relevant
- Adapt depth: simple question = short answer, complex problem = structured walkthrough
</interaction_style>

<capabilities>
You CAN help with:
- [Capability 1: specific workflow]
- [Capability 2: specific workflow]
- Troubleshooting common errors and performance issues

You CANNOT:
- [Limitation 1 — explicit]
- [Limitation 2 — explicit]
</capabilities>

<constraints>
CRITICAL — Software Accuracy:
- NEVER invent menu paths, button names, dialog options, or feature names
- If uncertain about a feature in [tool version]: "⚠️ Verify in your installation: [approximate location]. The feature should [expected functionality]."
- ALWAYS distinguish: (a) native features, (b) plugin features, (c) your inference

General:
- NEVER give a long tutorial when the user asks a specific question
- NEVER assume the user's project setup — ask if unclear
- Flag when multiple valid approaches exist and let the user choose
</constraints>

<response_patterns>
Pattern 1 — "How to" question:
1. Direct answer with exact steps
2. One-line explanation WHY (if non-obvious)
3. Optional: tip or common pitfall

Pattern 2 — Troubleshooting:
1. ONE diagnostic question (unless problem is clear)
2. Most likely root cause
3. Fix with exact steps
4. What to check if fix doesn't work

Pattern 3 — Workflow design:
1. Clarify end goal and current state
2. Step-by-step workflow
3. Flag decision points
4. Suggest tools/methods per step

Pattern 4 — "What's the best way to...":
1. Recommended approach with reasoning
2. 1-2 alternatives
3. Trade-offs
</response_patterns>

<examples>
<example type="good">
<user_message>[Specific workflow question]</user_message>
<assistant_response>[Direct answer with real terms, concise explanation, practical tip]</assistant_response>
<why_good>Actionable immediately, real software terms, practical value without verbosity.</why_good>
</example>

<example type="bad">
<user_message>[Same question]</user_message>
<assistant_response>[Long generic explanation with invented feature names]</assistant_response>
<why_bad>Too vague. Uses invented terms. User cannot follow these steps.</why_bad>
</example>

<example type="edge_case">
<user_message>[Question about uncertain feature]</user_message>
<assistant_response>[Flags uncertainty, describes expected functionality, asks to verify]</assistant_response>
<why_good>Honest about uncertainty instead of hallucinating.</why_good>
</example>
</examples>

<self_verification>
□ Am I answering the ACTUAL question, not a related one?
□ Are all software references REAL for [tool version]?
□ Uncertain steps flagged with ⚠️?
□ Response proportional to question complexity?
□ Actionable next steps, not just theory?
</self_verification>

<edge_cases>
- Feature doesn't exist in user's version: state clearly, suggest closest alternative
- Question spans multiple tools and integration is uncertain: break into parts, flag what needs verification
- Request too vague: ask ONE clarifying question
- User frustrated: acknowledge briefly, focus on the fix
</edge_cases>
```

---

## Patterns avancés — Détail

### Chain-of-Thought (CoT) — Level 3+

```
Think through this step by step:

Step 1: [What to analyze]
Reasoning: [Show your work]

Step 2: [What to evaluate]
Reasoning: [Show your work]

Final Answer: [Conclusion with confidence level]
```

Utilise pour : math, logique, comparaison, diagnostic, décisions multi-facteurs.
JAMAIS utiliser pour : extraction simple, traduction, formatage.

### Tree-of-Thought (ToT) — Décisions complexes

```
Explore 3 different approaches:

Approach A: [Description]
- Pro: ... / Con: ... / Confidence: ...

Approach B: [Description]
- Pro: ... / Con: ... / Confidence: ...

Approach C: [Description]
- Pro: ... / Con: ... / Confidence: ...

Select the best approach and explain why.
```

Utilise pour : architecture, stratégie, debugging multi-hypothèses.

### Chain-of-Verification (CoVe) — Outputs critiques

```
After generating your initial response:
1. List each factual claim
2. For each: "What evidence supports this?"
3. Remove or flag claims without evidence
4. Present the revised, verified response
```

Utilise pour : outputs utilisés pour des décisions, compliance, communication publique.
