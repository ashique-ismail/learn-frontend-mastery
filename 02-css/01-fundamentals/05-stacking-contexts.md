# Stacking Contexts and z-index Rules

## The Idea

**In plain English:** When web pages have overlapping elements, the browser needs rules to decide which one appears on top. A stacking context is a self-contained group of elements that gets layered as a single unit against everything else on the page — like a sealed stack of cards that moves together.

**Real-world analogy:** Imagine two binders sitting on a desk. Each binder holds a stack of papers inside it. You can reorder the papers within each binder, but a paper from the first binder can never slip between papers inside the second binder — the whole binder moves as one.
- The binder = a stacking context (a grouped layer with its own internal order)
- The papers inside a binder = child elements competing with `z-index` inside that context
- The desk with both binders on it = the root page, where the two binders (contexts) are compared against each other

---

## Learning Objectives
- Understand what creates a stacking context and why it matters
- Master z-index behavior within and across stacking contexts
- Debug common z-index layering issues
- Apply stacking context knowledge to modals, dropdowns, and overlays
- Recognize when elements create implicit stacking contexts

---

## What is a Stacking Context?

A **stacking context** is a three-dimensional conceptualization of HTML elements along an imaginary z-axis extending from the screen toward the user. Elements within a stacking context are layered according to specific rules.

**Key concept:** A stacking context is like a "layer group" in Photoshop. Elements inside can layer relative to each other, but the entire group moves as one unit relative to other contexts.

---

## The Default Stacking Order (Without z-index)

In the **normal document flow**, elements stack in this order (bottom to top):

1. **Root element background and borders** (`<html>`)
2. **Descendant non-positioned blocks**, in order of appearance in HTML
3. **Descendant positioned elements**, in order of appearance in HTML

**Example:**

```html
<div>Block 1</div>
<div>Block 2</div>
<div style="position: relative; top: -20px;">Positioned</div>
```

**Stacking (bottom to top):**
```
3. "Positioned" (positioned element)
2. "Block 2" (non-positioned)
1. "Block 1" (non-positioned)
```

The positioned element appears **on top** even without `z-index`.

---

## `z-index`: Controlling Stacking Order

`z-index` specifies the stack level of an element within its stacking context.

**Syntax:**
```css
.element {
  z-index: 10;      /* Positive integer */
  z-index: -1;      /* Negative integer */
  z-index: 0;       /* Default for positioned elements */
  z-index: auto;    /* Default, doesn't create stacking context */
}
```

**Key rules:**
1. `z-index` **only works on positioned elements** (`position: relative|absolute|fixed|sticky`) and flex/grid items
2. Higher `z-index` = on top (within the same stacking context)
3. `z-index: auto` vs `z-index: 0` have different stacking context implications (see below)

---

## What Creates a Stacking Context?

A stacking context is created by any element that meets **one or more** of these conditions:

### 1. Root element (`<html>`)
The document root always creates a stacking context.

### 2. Positioned elements with `z-index` other than `auto`
```css
.element {
  position: relative;  /* or absolute, fixed, sticky */
  z-index: 1;         /* Creates stacking context */
}

.no-context {
  position: relative;
  z-index: auto;      /* Does NOT create stacking context */
}
```

### 3. Flex or Grid items with `z-index` other than `auto`
```css
.container {
  display: flex;
}

.item {
  z-index: 1;  /* Creates stacking context (no position needed) */
}
```

### 4. Elements with `opacity` less than 1
```css
.element {
  opacity: 0.99;  /* Creates stacking context */
}
```

### 5. Elements with `transform` other than `none`
```css
.element {
  transform: translateX(0);  /* Even identity transforms create context */
}
```

### 6. Elements with `filter` other than `none`
```css
.element {
  filter: blur(5px);  /* Creates stacking context */
}
```

### 7. Elements with `perspective` other than `none`
```css
.element {
  perspective: 1000px;
}
```

### 8. Elements with `clip-path` other than `none`
```css
.element {
  clip-path: circle(50%);
}
```

### 9. Elements with `mask` / `mask-image` / `mask-border`
```css
.element {
  mask-image: url(mask.svg);
}
```

### 10. Elements with `mix-blend-mode` other than `normal`
```css
.element {
  mix-blend-mode: multiply;
}
```

### 11. Elements with `isolation: isolate`
```css
.element {
  isolation: isolate;  /* Explicitly create stacking context */
}
```

### 12. Elements with `will-change` for properties that create stacking contexts
```css
.element {
  will-change: transform;  /* Creates stacking context */
}
```

### 13. Elements with `contain: layout` or `contain: paint`
```css
.element {
  contain: layout;  /* or paint, or content, or strict */
}
```

### 14. Elements with `backdrop-filter`
```css
.element {
  backdrop-filter: blur(10px);
}
```

---

## z-index: The Stacking Context Trap

**Most common mistake:** Thinking `z-index: 9999` will always place an element on top.

### The Problem

```html
<style>
  .parent1 {
    position: relative;
    z-index: 1;  /* Creates stacking context */
  }
  
  .child1 {
    position: relative;
    z-index: 9999;  /* Only compared within .parent1 context */
  }
  
  .parent2 {
    position: relative;
    z-index: 2;  /* Creates stacking context, higher than parent1 */
  }
  
  .child2 {
    position: relative;
    z-index: 1;  /* Only compared within .parent2 context */
  }
</style>

<div class="parent1">
  <div class="child1">Child 1 (z-index: 9999)</div>
</div>

<div class="parent2">
  <div class="child2">Child 2 (z-index: 1)</div>
</div>
```

**Result:** `child2` appears **on top** of `child1`, even though `child1` has `z-index: 9999`.

**Why?** `child1` and `child2` are in **different stacking contexts**. Their `z-index` values are compared **only within their own contexts**. The parent contexts are compared (`parent1.z-index: 1` vs `parent2.z-index: 2`), and `parent2` wins, bringing all its children along.

**Visualization:**
```
Stacking context tree:
<html> (root context)
  ├─ .parent1 (z-index: 1)
  │   └─ .child1 (z-index: 9999 within parent1)
  └─ .parent2 (z-index: 2)  ← wins at this level
      └─ .child2 (z-index: 1 within parent2)
```

---

## Stacking Order Within a Context

Within a stacking context, elements are layered in this order (back to front):

1. **Background and borders of the context's root**
2. **Descendant non-positioned elements**, in HTML order
3. **Descendant positioned elements with negative `z-index`**, ordered by `z-index` (most negative = bottom)
4. **Descendant non-positioned elements in normal flow**
5. **Descendant positioned elements with `z-index: auto` or `z-index: 0`**, in HTML order
6. **Descendant positioned elements with positive `z-index`**, ordered by `z-index` (highest = top)

**Example:**

```html
<style>
  .context {
    position: relative;
    z-index: 1;  /* Creates stacking context */
  }
  
  .neg { position: absolute; z-index: -1; }
  .zero { position: absolute; z-index: 0; }
  .pos { position: absolute; z-index: 1; }
  .auto { position: absolute; z-index: auto; }
  .normal { /* no position */ }
</style>

<div class="context">
  Context background
  <div class="normal">Normal</div>
  <div class="neg">z-index: -1</div>
  <div class="auto">z-index: auto</div>
  <div class="zero">z-index: 0</div>
  <div class="pos">z-index: 1</div>
</div>
```

**Stacking (back to front):**
```
1. Context background
2. z-index: -1 (behind normal flow)
3. Normal (non-positioned)
4. z-index: auto
5. z-index: 0
6. z-index: 1
```

---

## `z-index: auto` vs `z-index: 0`

**Similarity:** Both stack at the same level.

**Difference:**
- `z-index: auto` - **Does NOT create a stacking context** (unless other properties do)
- `z-index: 0` - **Creates a stacking context** (for positioned elements)

**Example:**

```html
<style>
  .parent-auto {
    position: relative;
    z-index: auto;  /* No stacking context created */
  }
  
  .parent-zero {
    position: relative;
    z-index: 0;  /* Creates stacking context */
  }
  
  .child {
    position: absolute;
    z-index: 10;
  }
</style>

<div class="parent-auto">
  <div class="child">Child in auto</div>
</div>

<div class="parent-zero">
  <div class="child">Child in zero</div>
</div>

<div class="sibling" style="position: relative; z-index: 5;">Sibling</div>
```

**Result:**
- "Child in auto" appears **above** sibling (z-index: 10 vs 5, compared in root context)
- "Child in zero" appears **below** sibling (trapped in parent-zero context)

---

## Debugging z-index Issues

### Strategy 1: Identify Stacking Contexts

**Chrome DevTools:**
1. Inspect element
2. Elements panel → Styles → Computed
3. Look for properties that create stacking contexts:
   - `position` + `z-index` (not `auto`)
   - `opacity` < 1
   - `transform` ≠ `none`
   - etc.

**Firefox DevTools:**
- Has a **3D view** (Cmd/Ctrl + Shift + M → 3D View button) showing stacking contexts

**Manual check:**
```css
/* Temporarily add this to suspected elements */
.debug {
  outline: 2px solid red;
  background: rgba(255, 0, 0, 0.1);
}
```

---

### Strategy 2: Trace the Context Tree

**Problem:** Element won't go on top despite high `z-index`.

**Solution:** Trace up the DOM tree to find the stacking context boundary.

```html
<div class="modal" style="z-index: 9999;">
  <!-- Won't appear above .overlay if .modal's parent creates a lower context -->
</div>
```

**Fix:** Move `.modal` outside the problematic stacking context (e.g., as a direct child of `<body>`).

---

### Strategy 3: Flatten Stacking Contexts

If elements don't need isolated stacking, avoid creating contexts unnecessarily.

```css
/* Creates stacking context (traps children) */
.container {
  transform: translateZ(0);  /* Often added for GPU acceleration */
}

/* If not needed, remove it */
.container {
  /* No transform, children can stack freely in parent context */
}
```

---

## Common Use Cases

### Use Case 1: Modal Overlays

**Goal:** Modal appears above all content.

**Solution:** Place modal in root stacking context with high `z-index`.

```html
<body>
  <div class="app"><!-- main content --></div>
  <div class="modal"><!-- must be sibling, not child of .app --></div>
</body>
```

```css
.modal {
  position: fixed;
  z-index: 1000;  /* High value in root context */
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
}
```

**Why it works:** `.modal` is in the root stacking context, not trapped in `.app`.

---

### Use Case 2: Dropdown Menus

**Problem:** Dropdown appears behind sibling content.

```html
<nav>
  <div class="nav-item">
    <button>Menu</button>
    <div class="dropdown"><!-- hidden, then shown --></div>
  </div>
  <div class="nav-item">Other item</div>
</nav>
```

**Solution:**

```css
.nav-item {
  position: relative;
}

.dropdown {
  position: absolute;
  z-index: 10;  /* Higher than siblings */
  top: 100%;
  left: 0;
}
```

**Alternative:** Use `isolation: isolate` on `.nav` to create a stacking context boundary.

```css
.nav {
  isolation: isolate;  /* Contains all nav-item contexts */
}
```

---

### Use Case 3: Sticky Headers

**Problem:** Sticky header should always be on top.

```css
.header {
  position: sticky;
  top: 0;
  z-index: 100;  /* Ensure it's above content */
  background: white;
}
```

**Gotcha:** If content below creates stacking contexts with `z-index > 100`, they might appear above the header.

**Solution:** Ensure header is in a higher stacking context or use higher `z-index`.

---

### Use Case 4: Tooltips

**Goal:** Tooltip appears above all sibling content.

```css
.tooltip-container {
  position: relative;
}

.tooltip {
  position: absolute;
  z-index: 9999;  /* High value */
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
}
```

**Gotcha:** If `.tooltip-container` has `overflow: hidden`, tooltip will be clipped.

**Solution:** Use `position: fixed` and calculate position with JavaScript, or portal the tooltip to `<body>`.

---

## Organizing z-index at Scale

### Strategy 1: z-index Scale

Define a consistent scale in CSS variables:

```css
:root {
  --z-below: -1;
  --z-normal: 0;
  --z-dropdown: 100;
  --z-sticky: 200;
  --z-modal-backdrop: 900;
  --z-modal: 1000;
  --z-popover: 1100;
  --z-tooltip: 1200;
  --z-notification: 1300;
}

.modal {
  z-index: var(--z-modal);
}
```

**Benefits:**
- Centralized control
- Semantic naming
- Easy to adjust

---

### Strategy 2: Named Layers (Future)

CSS Cascade Layers (`@layer`) will eventually support layering with `z-index`-like semantics, but currently, layers don't affect stacking order directly.

---

### Strategy 3: Minimize Stacking Contexts

**Principle:** Fewer stacking contexts = simpler `z-index` management.

**Guidelines:**
- Don't add `transform: translateZ(0)` unless needed for performance
- Use `opacity: 0.99` only when necessary
- Avoid unnecessary `position: relative` + `z-index` combos

---

## Common Mistakes

### ❌ Mistake 1: z-index without positioning

```css
/* Bad - z-index has no effect */
.element {
  z-index: 10;
}

/* Good */
.element {
  position: relative;  /* or absolute, fixed, sticky */
  z-index: 10;
}
```

---

### ❌ Mistake 2: Arbitrary high z-index values

```css
/* Bad - meaningless escalation */
.modal { z-index: 9999; }
.tooltip { z-index: 99999; }
.notification { z-index: 999999; }

/* Good - use a consistent scale */
.modal { z-index: 1000; }
.tooltip { z-index: 1100; }
.notification { z-index: 1200; }
```

---

### ❌ Mistake 3: Not understanding stacking context boundaries

```css
/* Bad - child z-index won't help if parent is in lower context */
.parent {
  position: relative;
  z-index: 1;  /* Low context */
}

.child {
  position: absolute;
  z-index: 9999;  /* Trapped in parent context */
}

/* Good - move child to higher context or increase parent's z-index */
.parent {
  position: relative;
  z-index: 100;  /* Higher context */
}

.child {
  position: absolute;
  z-index: 10;  /* Relative to parent context */
}
```

---

### ❌ Mistake 4: Creating unintended stacking contexts

```css
/* Bad - unintended context from opacity */
.container {
  opacity: 0.99;  /* Creates stacking context, traps children */
}

/* Good - use transparency only when needed */
.container {
  /* No opacity if not needed */
}
```

---

## Advanced: Negative z-index

**Use case:** Place element **behind** its parent's background.

```html
<style>
  .card {
    position: relative;
    background: white;
    padding: 2rem;
  }
  
  .card::before {
    content: "";
    position: absolute;
    inset: 0;
    background: linear-gradient(135deg, pink, purple);
    z-index: -1;  /* Behind .card background */
    transform: rotate(3deg);
  }
</style>

<div class="card">
  Card content
</div>
```

**Result:** Pseudo-element appears as a rotated background behind the card.

**Why it works:** Negative `z-index` places the element behind the stacking context root's background.

---

## Browser DevTools Visualization

### Chrome: Layers Panel

1. DevTools → More tools → Layers
2. Shows compositing layers (often correlated with stacking contexts)
3. Hover to highlight on page

**Note:** Compositing layers ≠ stacking contexts, but they overlap.

---

### Firefox: 3D View

1. DevTools → 3D View
2. Shows z-index stacking in 3D
3. Click elements to inspect

**Best tool** for visualizing stacking contexts and z-index.

---

## Performance Considerations

### Stacking Contexts and Compositing

Elements that create stacking contexts **may** be promoted to their own compositing layer for GPU acceleration.

**Properties that often trigger compositing:**
- `transform: translateZ(0)` or `transform: translate3d(0, 0, 0)`
- `will-change: transform`
- `opacity` < 1
- `filter`

**Benefit:** Smoother animations (GPU-accelerated).

**Cost:** More memory usage.

**Best practice:** Let the browser decide. Only force compositing for performance bottlenecks.

```css
/* Avoid unnecessary promotion */
.element {
  transform: translateZ(0);  /* Only if needed for performance */
}
```

---

## Isolation Property

`isolation: isolate` **explicitly creates a stacking context** without affecting rendering.

**Use case:** Prevent blend modes or z-index from leaking out.

```css
.component {
  isolation: isolate;  /* Contains all internal stacking */
}

.component .child {
  z-index: 10;  /* Only compared within .component */
}
```

**Benefit:** Encapsulation. Component's internal z-index doesn't affect external elements.

---

## Key Takeaways

✅ **z-index only works on positioned elements (or flex/grid items)**

✅ **Stacking contexts isolate z-index comparisons** - children can't escape their context

✅ **Many properties create stacking contexts**, not just `position` + `z-index`

✅ **z-index: auto doesn't create a context; z-index: 0 does** (for positioned elements)

✅ **Negative z-index places elements behind the context root's background**

✅ **Use a z-index scale** for consistency (e.g., 100, 200, 300, not 1, 2, 9999)

✅ **Debug with Firefox 3D View** or Chrome Layers panel

✅ **Portals (move elements to body) solve most stacking issues** for modals/tooltips

✅ **Use `isolation: isolate` to create explicit stacking boundaries**

---

## Resources

- [MDN: Stacking Context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_context)
- [MDN: z-index](https://developer.mozilla.org/en-US/docs/Web/CSS/z-index)
- [What The Heck, z-index??](https://www.joshwcomeau.com/css/stacking-contexts/)
- [Phillip Walton: What No One Told You About Z-Index](https://philipwalton.com/articles/what-no-one-told-you-about-z-index/)
- [CSS Tricks: Z-Index And The CSS Stack](https://css-tricks.com/handling-z-index/)
