---
name: bimsmarter-ui
description: Charte graphique et composants UI BIMSmarter. Utilise ce skill quand l'utilisateur veut créer une nouvelle page, un nouveau composant React, ou mettre en cohérence une app existante avec le style BIMSmarter. Si l'utilisateur mentionne "style", "charte", "dark theme", "glass panel", "même style que les autres apps", "fond sombre", utilise ce skill immédiatement pour générer du code cohérent avec la charte.
---

# BIMSmarter UI — Charte graphique

**Règle absolue** : toute nouvelle page ou composant doit respecter cette charte. Ne jamais inventer un style différent.

---
## Fond de page principal

Fond bleu foncé : hsl(222 47% 11%) (équivalent #0f172a)
Pattern de petites croix/points cyan subtils en arrière-plan
Les croix utilisent : hsl(199 89% 48% / 0.15) (cyan transparent)
Espacement du pattern : 24px x 24px
Créer une classe CSS .bg-bimsmarter avec :

## Tokens de couleur

```css
/* Fond principal */
--bg-base: hsl(222, 47%, 11%);        /* #0f172a */
--bg-panel: hsl(222, 47%, 11%, 0.95);

/* Accent (titres, liens, icônes actives) */
--accent: hsl(199, 89%, 48%);          /* #0ea5e9 */

/* Bordures */
--border: hsl(199, 89%, 48%, 0.2);

/* Texte */
--text-primary: hsl(0, 0%, 95%);
--text-muted: hsl(0, 0%, 60%);
```

---

## Structure HTML de base (toute app)

```html
<div class="min-h-screen flex flex-col bg-[hsl(222,47%,11%)]"
     style="background-image: radial-gradient(circle at 20% 20%, hsl(199,89%,48%,0.05) 0%, transparent 50%)">

  <header class="sticky top-0 z-10 glass-panel border-b border-[hsl(199,89%,48%,0.2)] px-4 py-3">
    <a href="https://bimsmarter.eu" class="text-[hsl(199,89%,48%)] font-bold">BIMsmarter</a>
  </header>

  <main class="p-4 md:p-6 space-y-4">
    <h1 class="text-2xl font-bold text-[hsl(199,89%,48%)]">Titre de la page</h1>
    <!-- Contenu -->
  </main>

</div>
```

---

## Composants clés

### Glass Panel
```css
.glass-panel {
  background: hsl(222, 47%, 11%, 0.95);
  border: 1px solid hsl(199, 89%, 48%, 0.2);
  backdrop-filter: blur(8px);
}
```
```jsx
// En Tailwind inline
<div className="bg-[hsl(222,47%,11%,0.95)] border border-[hsl(199,89%,48%,0.2)] rounded-lg p-4">
```

### Table compacte (données BIM)
```css
table.data-dense {
  font-family: monospace;
  width: 100%;
  table-layout: fixed;
}
table.data-dense th {
  color: hsl(199, 89%, 48%);
  border-bottom: 1px solid hsl(199, 89%, 48%, 0.2);
  padding: 6px 8px;
  text-align: left;
}
table.data-dense td {
  padding: 4px 8px;
  border-bottom: 1px solid hsl(199, 89%, 48%, 0.1);
  color: hsl(0, 0%, 85%);
}
```

### Tooltip d'info
```html
<span title="BCF Topic : Issue de collaboration selon ISO 21597">🛈</span>
```

### Bouton principal
```jsx
<button className="bg-[hsl(199,89%,48%)] text-white font-medium px-4 py-2 rounded hover:bg-[hsl(199,89%,55%)] transition-colors">
  Action
</button>
```

### Bouton secondaire (outline)
```jsx
<button className="border border-[hsl(199,89%,48%,0.4)] text-[hsl(199,89%,48%)] px-4 py-2 rounded hover:border-[hsl(199,89%,48%)] transition-colors">
  Action secondaire
</button>
```

### Input / Select
```jsx
<input className="bg-[hsl(222,47%,15%)] border border-[hsl(199,89%,48%,0.2)] text-white rounded px-3 py-2 focus:outline-none focus:border-[hsl(199,89%,48%)] w-full" />
```

### Badge / Tag
```jsx
<span className="text-xs bg-[hsl(199,89%,48%,0.15)] text-[hsl(199,89%,48%)] border border-[hsl(199,89%,48%,0.3)] rounded px-2 py-0.5">
  ISO 19650
</span>
```

---

## Règles typographie

- **Titres** : `text-[hsl(199,89%,48%)]` + `font-bold`
- **Texte principal** : `text-white` ou `text-gray-100`
- **Texte secondaire/muted** : `text-gray-400`
- **Code/monospace** : `font-mono text-[hsl(199,89%,48%)]`
- Taille h1 : `text-2xl`, h2 : `text-xl`, h3 : `text-lg`
- Police principale : Roboto (Google Fonts)
Import : @import url("https://fonts.googleapis.com/css2?family=Roboto&display=swap");

---

## Anti-patterns à éviter

| ❌ À éviter | ✅ BIMSmarter |
|------------|---------------|
| Fond blanc ou gris clair | Fond `hsl(222,47%,11%)` |
| Accent rouge/vert/violet | Accent `hsl(199,89%,48%)` uniquement |
| Bordures grises standard | Bordures `hsl(199,89%,48%,0.2)` |
| Cards avec ombre portée classique | Glass panel avec bordure cyan |
| Boutons ronds plein style Material | Boutons `rounded` (pas `rounded-full`) |

---

## Pattern page complète React

```jsx
export default function MaPage() {
  return (
    <div
      className="min-h-screen flex flex-col"
      style={{
        background: 'hsl(222, 47%, 11%)',
        backgroundImage: 'radial-gradient(circle at 20% 20%, hsl(199,89%,48%,0.05) 0%, transparent 50%)'
      }}
    >
      <header className="sticky top-0 z-10 border-b px-4 py-3 flex items-center gap-3"
              style={{ background: 'hsl(222,47%,11%,0.95)', borderColor: 'hsl(199,89%,48%,0.2)' }}>
        <a href="https://bimsmarter.eu" className="font-bold" style={{ color: 'hsl(199,89%,48%)' }}>
          BIMsmarter
        </a>
        <span className="text-gray-400 text-sm">/ Nom de l'app</span>
      </header>

      <main className="p-4 md:p-6 space-y-4">
        <h1 className="text-2xl font-bold" style={{ color: 'hsl(199,89%,48%)' }}>
          Titre
        </h1>

        <div className="rounded-lg p-4"
             style={{ background: 'hsl(222,47%,11%,0.95)', border: '1px solid hsl(199,89%,48%,0.2)' }}>
          {/* Contenu */}
        </div>
      </main>
    </div>
  );
}
```
