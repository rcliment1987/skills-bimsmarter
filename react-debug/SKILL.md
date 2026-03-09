---
name: react-debug
description: Détecte et corrige les bugs React récurrents dans les apps BIMSmarter. Utilise ce skill dès qu'un input perd le focus en cours de frappe, qu'un composant se recrée à chaque render, qu'un état async ne s'affiche pas correctement, ou que l'autocomplete se comporte bizarrement. Aussi pertinent pour tout bug React lié à useState, useEffect, re-render, ou perte de référence de composant. Si l'utilisateur dit "ça se déselectionne", "le focus se perd", "ça re-render trop", "l'état ne se met pas à jour", utilise ce skill immédiatement.
---

# React Debug — BIMSmarter

Tu es un expert React qui connaît les anti-patterns récurrents des apps BIMSmarter (Workflow Generator, GID-Assistant, Document Generator). Ton approche : ROOT CAUSE d'abord, correction ensuite. Jamais de workaround.

**RÈGLE ABSOLUE** : Explique POURQUOI le bug existe avant de donner le code de correction.

---

## Bug #1 — Focus loss sur les inputs (le plus fréquent)

### Symptôme
L'utilisateur tape dans un champ → après 1 caractère, le champ se déselectionne. Il doit recliquer pour continuer à taper.

### Root cause
Un composant est **défini à l'intérieur d'un autre composant**. À chaque frappe → `setState` → re-render du parent → le composant enfant est recréé comme une **nouvelle fonction** → React le démonte et remonte → perte de focus.

### Diagnostic rapide
Cherche ce pattern dans le code :

```javascript
// ❌ PROBLÈME : composant défini DANS un autre composant
const ParentComponent = () => {
  const [steps, setSteps] = useState([]);

  // ← ICI c'est le bug : InputField est recréé à chaque render de ParentComponent
  const InputField = ({ value, onChange }) => {
    return <input value={value} onChange={onChange} />;
  };

  return <InputField value={steps[0]} onChange={...} />;
};
```

### Correction
Sortir le composant **en dehors** du parent, au niveau module :

```javascript
// ✅ CORRECT : InputField est défini UNE SEULE FOIS, référence stable
const InputField = ({ value, onChange }) => {
  return <input value={value} onChange={onChange} />;
};

const ParentComponent = () => {
  const [steps, setSteps] = useState([]);
  return <InputField value={steps[0]} onChange={...} />;
};
```

### Cas BIMSmarter connu
`AutocompleteInput` défini dans `CustomWorkflowBuilder` → à sortir avant le composant parent.

---

## Bug #2 — Autocomplete qui se ferme ou ne filtre pas

### Symptôme
La liste de suggestions disparaît en cliquant dessus, ou le focus est perdu avant la sélection.

### Root cause
L'événement `onBlur` de l'input se déclenche **avant** le `onClick` de la suggestion. L'input perd le focus → `onBlur` cache la liste → le clic n'atteint jamais la suggestion.

### Correction
Remplacer `onClick` par `onMouseDown` sur les suggestions (se déclenche avant `onBlur`) :

```javascript
// ❌ PROBLÈME
<div onClick={() => handleSelect(opt)}>

// ✅ CORRECT
<div onMouseDown={() => handleSelect(opt)}>
```

Et pour fermer la liste avec un délai pour laisser `onMouseDown` s'exécuter :

```javascript
onBlur={() => setTimeout(() => setShowSuggestions(false), 150)}
```

---

## Bug #3 — État async qui ne se met pas à jour (stale state)

### Symptôme
Une valeur affichée semble "en retard" d'un render. Ou une fonction utilise une vieille valeur alors qu'on vient de la mettre à jour.

### Root cause
Les closures JavaScript capturent la valeur **au moment de leur création**. Si tu utilises `state` dans un `useEffect` ou un callback sans le déclarer en dépendance, tu lis une valeur obsolète.

### Diagnostic
```javascript
// ❌ PROBLÈME : chatMessages capturé à la création, jamais mis à jour
useEffect(() => {
  socket.on('message', (msg) => {
    setChatMessages([...chatMessages, msg]); // chatMessages est "stale"
  });
}, []); // ← dépendances vides = closure figée
```

### Correction
Utiliser la forme fonctionnelle du setter :

```javascript
// ✅ CORRECT : prev est toujours la valeur actuelle
useEffect(() => {
  socket.on('message', (msg) => {
    setChatMessages(prev => [...prev, msg]); // toujours à jour
  });
}, []);
```

---

## Bug #4 — Re-render excessif qui ralentit l'app

### Symptôme
L'app rame, des appels API se dupliquent, des animations sont saccadées.

### Causes fréquentes dans BIMSmarter

**Objet/array créé inline dans JSX** → nouvelle référence à chaque render :
```javascript
// ❌ Crée un nouveau tableau à chaque render → enfants re-renderent toujours
<Component options={['a', 'b', 'c']} />

// ✅ Définir hors du composant ou avec useMemo
const OPTIONS = ['a', 'b', 'c']; // hors du composant
<Component options={OPTIONS} />
```

**useEffect sans dépendances correctes** → s'exécute en boucle :
```javascript
// ❌ Se relance à chaque render si `data` change à chaque render
useEffect(() => {
  fetchData(data);
}, [data]); // data est un objet recréé à chaque render

// ✅ Utiliser une valeur primitive comme dépendance
useEffect(() => {
  fetchData(data.id);
}, [data.id]); // string/number, référence stable
```

---

## Checklist de diagnostic rapide

Quand un bug React est signalé, vérifier dans cet ordre :

1. **Composant défini dans un composant ?** → sortir au niveau module
2. **onClick sur suggestion autocomplete ?** → remplacer par onMouseDown
3. **State utilisé dans closure sans dépendance ?** → forme fonctionnelle `prev =>`
4. **Objet/array créé inline dans JSX ?** → déplacer hors du composant
5. **useEffect avec objet en dépendance ?** → utiliser une propriété primitive

---

## Format de réponse

Pour chaque bug signalé :

```
### Root cause
[1-2 phrases expliquant POURQUOI le bug existe]

### Localisation
[Où chercher dans le code — composant, ligne approximative, pattern à identifier]

### Correction
[Code minimal avec commentaires ❌/✅]

### Vérification
[Comment confirmer que c'est corrigé]
```
