# Selectors, Specificity, Cascade, and Inheritance

## Learning Objectives
- Master CSS selector types and their specificity weights
- Understand how the cascade algorithm resolves conflicting styles
- Differentiate between inherited and non-inherited properties
- Debug specificity conflicts effectively
- Use specificity strategically in scalable CSS architectures

---

## CSS Selectors: A Complete Reference

### Type Selectors (Element Selectors)
Target HTML elements by their tag name.

```css
p { color: blue; }
div { margin: 1rem; }
```

**Specificity weight:** `0-0-1`

---

### Class Selectors
Target elements by their `class` attribute.

```css
.button { padding: 0.5rem 1rem; }
.card { border: 1px solid #ccc; }
```

**Specificity weight:** `0-1-0`

**Best practice:** Use classes as your primary styling hook. They're reusable and maintain low specificity.

---

### ID Selectors
Target a unique element by its `id` attribute.

```css
#header { position: sticky; }
#main-content { max-width: 1200px; }
```

**Specificity weight:** `1-0-0`

**⚠️ Warning:** IDs have very high specificity, making them hard to override. Prefer classes for styling; reserve IDs for JavaScript hooks and fragment identifiers.

---

### Attribute Selectors

#### Exact match
```css
[type="text"] { border: 1px solid gray; }
[disabled] { opacity: 0.5; }
```

#### Substring matching
```css
/* Starts with */
[href^="https"] { color: green; }

/* Ends with */
[href$=".pdf"] { 
  background: url(pdf-icon.svg) no-repeat left center;
}

/* Contains */
[class*="btn-"] { border-radius: 4px; }

/* Space-separated list contains */
[class~="active"] { font-weight: bold; }

/* Dash-separated starts with */
[lang|="en"] { /* matches en, en-US, en-GB */ }
```

#### Case-insensitive matching
```css
[href$=".PDF" i] { /* 'i' flag makes it case-insensitive */ }
```

**Specificity weight:** `0-1-0` (same as class)

---

### Pseudo-Classes

#### State-based
```css
a:hover { color: red; }
a:focus { outline: 2px solid blue; }
a:active { color: darkred; }
input:disabled { cursor: not-allowed; }
input:checked + label { font-weight: bold; }
```

#### Structural
```css
li:first-child { margin-top: 0; }
li:last-child { margin-bottom: 0; }
li:nth-child(odd) { background: #f0f0f0; }
li:nth-child(3n) { /* every 3rd */ }
li:nth-of-type(2) { /* 2nd <li> among siblings */ }
p:only-child { /* only child of parent */ }
div:empty { display: none; }
```

#### Form-related
```css
input:valid { border-color: green; }
input:invalid { border-color: red; }
input:required { border-left: 3px solid blue; }
input:optional { opacity: 0.9; }
input:in-range { border-color: green; }
input:out-of-range { border-color: red; }
```

#### Negation
```css
button:not(.primary) { background: gray; }
li:not(:last-child) { margin-bottom: 1rem; }
```

**Specificity weight:** `0-1-0` (except `:not()` which takes the specificity of its argument)

---

### Pseudo-Elements

Target sub-parts of elements or generate content.

```css
p::first-line { font-weight: bold; }
p::first-letter { font-size: 2em; float: left; }

/* Generated content */
.external-link::after {
  content: "↗";
  margin-left: 0.25em;
}

blockquote::before {
  content: """;
  font-size: 3em;
  color: #ccc;
}

/* Selection styling */
::selection {
  background: yellow;
  color: black;
}

/* Placeholder text */
input::placeholder {
  color: #999;
  opacity: 1;
}
```

**Syntax note:** Use `::` (double colon) for pseudo-elements, `:` (single colon) for pseudo-classes. Browsers accept single colon for legacy pseudo-elements (`::before`, `::after`, `::first-line`, `::first-letter`) but double colon is the standard.

**Specificity weight:** `0-0-1`

---

### Combinators

#### Descendant combinator (space)
```css
article p { color: gray; }
/* Any <p> inside <article>, at any depth */
```

#### Child combinator (>)
```css
ul > li { list-style: none; }
/* Only direct children */
```

#### Adjacent sibling combinator (+)
```css
h2 + p { font-weight: bold; }
/* <p> immediately after <h2> */
```

#### General sibling combinator (~)
```css
h2 ~ p { margin-left: 1rem; }
/* All <p> siblings after <h2> */
```

**Specificity:** Combinators themselves have no specificity; only the selectors they connect.

---

### Modern Selectors

#### `:is()` - Forgiving selector list
```css
/* Instead of repeating */
article h1, article h2, article h3 { color: navy; }

/* Use :is() */
article :is(h1, h2, h3) { color: navy; }
```

**Specificity:** Takes the highest specificity of its arguments.

#### `:where()` - Zero-specificity selector list
```css
:where(article, section, aside) p {
  line-height: 1.6;
}
```

**Specificity:** Always `0-0-0`, making it easy to override.

#### `:has()` - Parent selector
```css
/* Card that contains an image */
.card:has(img) { display: grid; }

/* Form with invalid input */
form:has(input:invalid) { border: 2px solid red; }

/* Link containing an image */
a:has(> img) { display: block; }
```

**Specificity:** Like `:is()`, takes the specificity of its argument.

---

## Specificity: The Tie-Breaker

When multiple rules target the same element and set the same property, **specificity** determines which wins.

### Specificity Calculation

Think of specificity as a three-part number: `ID-CLASS-TYPE`

| Selector | ID | CLASS | TYPE | Specificity |
|----------|-----|-------|------|-------------|
| `*` | 0 | 0 | 0 | `0-0-0` |
| `p` | 0 | 0 | 1 | `0-0-1` |
| `.button` | 0 | 1 | 0 | `0-1-0` |
| `#header` | 1 | 0 | 0 | `1-0-0` |
| `div p` | 0 | 0 | 2 | `0-0-2` |
| `.card p` | 0 | 1 | 1 | `0-1-1` |
| `#nav .item` | 1 | 1 | 0 | `1-1-0` |
| `article > p.intro` | 0 | 1 | 2 | `0-1-2` |

**Comparison rules:**
1. Compare IDs: higher wins
2. If tied, compare CLASSes: higher wins
3. If tied, compare TYPEs: higher wins
4. If still tied, last rule in source order wins

**Examples:**

```css
/* Specificity: 0-1-0 */
.button { background: blue; }

/* Specificity: 0-0-1 - LOSES */
button { background: red; }

/* Result: background is blue */
```

```css
/* Specificity: 0-1-1 */
nav a { color: white; }

/* Specificity: 0-2-0 - WINS (more classes) */
.nav-link.active { color: yellow; }
```

---

### The `!important` Nuclear Option

```css
.button { 
  background: blue !important; 
}
```

**Effect:** Overrides all normal declarations, regardless of specificity.

**Important trumps everything** except:
- Another `!important` with higher specificity
- Another `!important` declared later (if specificity is equal)

**When to use `!important`:**
- Utility classes that must always win (e.g., `.hidden { display: none !important; }`)
- Overriding third-party styles you can't modify
- Never in application code if avoidable

**Why to avoid:**
- Breaks the natural cascade
- Creates an arms race of `!important` declarations
- Makes debugging harder

---

### Specificity Best Practices

✅ **Keep specificity low and flat**
```css
/* Good */
.card { }
.card-title { }
.card-body { }

/* Bad - unnecessarily specific */
div.card > div.card-header > h2.card-title { }
```

✅ **Use classes, not IDs, for styling**
```css
/* Good */
.header { }

/* Bad */
#header { }
```

✅ **Avoid tag + class selectors**
```css
/* Good */
.button { }

/* Avoid - adds unnecessary specificity */
button.button { }
```

✅ **Use `:where()` for zero-specificity defaults**
```css
/* Easy to override */
:where(h1, h2, h3) {
  margin-top: 0;
}

/* Later, this wins easily */
.special-heading {
  margin-top: 2rem;
}
```

---

## The Cascade: Conflict Resolution Algorithm

When multiple rules apply to an element, CSS uses this algorithm to determine which values win:

### Cascade Algorithm (in order of priority)

1. **Origin and Importance**
   - User agent `!important` (browser defaults)
   - User `!important` (user stylesheets)
   - Author `!important` (your styles)
   - CSS Animations
   - Author styles (your normal styles)
   - User styles
   - User agent styles

2. **Specificity** (covered above)

3. **Order of Appearance**
   - Last declaration wins (if origin and specificity are equal)

---

### Origin Example

```css
/* User agent (browser default) */
p { margin: 1em 0; }

/* Your stylesheet */
p { margin: 0; }  /* WINS - author styles override UA styles */
```

---

### Inline Styles

Inline styles have higher specificity than any selector (except `!important`).

```html
<p style="color: red;">  <!-- Specificity: "inline-0-0-0" -->
```

Specificity comparison:
- Inline style: `1-0-0-0`
- ID selector: `0-1-0-0`
- Class selector: `0-0-1-0`
- Element selector: `0-0-0-1`

**Avoid inline styles** except for dynamic values set by JavaScript.

---

### Layers (`@layer`) - Modern Cascade Control

Cascade layers let you explicitly control the cascade order.

```css
/* Define layer order */
@layer reset, base, components, utilities;

@layer reset {
  * { margin: 0; padding: 0; }
}

@layer base {
  p { margin: 1rem 0; }
}

@layer components {
  .card { padding: 1rem; }
}

@layer utilities {
  .mt-0 { margin-top: 0 !important; }
}
```

**Key insight:** Earlier layers have **lower priority**, even if they have higher specificity.

```css
@layer base, overrides;

@layer base {
  /* Specificity: 1-0-0 - but in earlier layer */
  #header { color: blue; }
}

@layer overrides {
  /* Specificity: 0-0-1 - but in later layer, so WINS */
  header { color: red; }
}
```

**Unlayered styles** beat all layered styles.

---

## Inheritance: Passing Values Down the Tree

Some CSS properties **inherit** from parent to child automatically.

### Inherited Properties

**Typography:**
- `color`
- `font-family`, `font-size`, `font-weight`, `font-style`
- `line-height`
- `text-align`, `text-indent`, `text-transform`
- `letter-spacing`, `word-spacing`

**Lists:**
- `list-style`, `list-style-type`, `list-style-position`

**Visibility:**
- `visibility` (but not `display`)
- `cursor`

**Example:**

```css
body {
  font-family: Arial, sans-serif;
  color: #333;
  line-height: 1.6;
}

/* All descendants inherit these values unless overridden */
p { /* inherits color, font-family, line-height */ }
```

---

### Non-Inherited Properties

Most **layout** and **box model** properties do NOT inherit:

- `margin`, `padding`, `border`
- `width`, `height`
- `display`, `position`
- `background`
- `transform`, `opacity`

```css
.parent {
  margin: 2rem;
  background: lightblue;
}

.child {
  /* Does NOT inherit margin or background */
}
```

---

### Controlling Inheritance

#### `inherit` - Force inheritance
```css
.parent { color: red; }

.child {
  /* color inherits by default, but let's say we reset it */
  color: initial;
  
  /* Now force it to inherit again */
  color: inherit;  /* red */
}

/* Useful for non-inherited properties */
.child {
  border: inherit;  /* Inherits parent's border */
}
```

#### `initial` - Reset to spec default
```css
p {
  /* Reset to CSS spec default, ignoring inheritance and user styles */
  color: initial;  /* black, per spec */
  margin: initial;  /* 1em 0, per spec */
}
```

#### `unset` - Inherit if inheritable, else initial
```css
/* If property naturally inherits, behaves like 'inherit' */
/* If not, behaves like 'initial' */
.element {
  color: unset;  /* inherits from parent (color inherits) */
  margin: unset;  /* resets to 0 (margin doesn't inherit) */
}
```

#### `revert` - Roll back to user agent or user styles
```css
h1 {
  font-size: revert;  /* Goes back to browser's default h1 size */
}
```

---

### Shorthand Reset with `all`

```css
.component {
  /* Reset ALL properties to their initial values */
  all: initial;
  
  /* Or inherit all */
  all: inherit;
  
  /* Or unset all */
  all: unset;
  
  /* Or revert all */
  all: revert;
}
```

**Use case:** Isolating a component from external styles (e.g., in a Web Component or widget).

---

## Debugging Specificity Conflicts

### Browser DevTools Strategy

1. **Inspect the element**
2. **Look at the Styles panel** - crossed-out rules are overridden
3. **Check why:**
   - Hover over selector to see specificity
   - Look for `!important`
   - Check cascade order

**Chrome DevTools trick:** Click the specificity indicator to see a breakdown.

---

### Visual Specificity Calculator

Use tools like:
- [Specificity Calculator](https://specificity.keegan.st/)
- [Polypane CSS Specificity Visualizer](https://polypane.app/css-specificity-calculator/)

---

### Common Specificity Gotchas

#### Gotcha 1: Order doesn't matter if specificity differs
```css
.button { color: blue; }  /* 0-1-0 */
button { color: red; }     /* 0-0-1 - LOSES even though it's later */
```

#### Gotcha 2: Inline styles beat everything
```html
<button class="button" style="color: green;">Click me</button>
```
```css
.button { color: blue !important; }  /* LOSES to inline */
```

#### Gotcha 3: `:not()` doesn't add specificity, its argument does
```css
/* Specificity: 0-1-0 (from .special) */
:not(.special) { color: gray; }

/* NOT 0-2-0 */
```

#### Gotcha 4: Universal selector counts as zero
```css
* { color: blue; }        /* 0-0-0 */
p { color: red; }         /* 0-0-1 - WINS */
```

---

## Real-World Strategy: Managing Specificity at Scale

### BEM Naming Convention
Keeps specificity flat (`0-1-0` for everything).

```css
/* Block */
.card { }                /* 0-1-0 */

/* Element */
.card__title { }         /* 0-1-0 */
.card__body { }          /* 0-1-0 */

/* Modifier */
.card--featured { }      /* 0-1-0 */
.card__title--large { }  /* 0-1-0 */
```

**Benefit:** Predictable, flat specificity graph. Easy to override.

---

### Utility-First (Tailwind)
Uses single classes with very low specificity.

```html
<div class="p-4 bg-blue-500 text-white rounded">
```

```css
.p-4 { padding: 1rem; }           /* 0-1-0 */
.bg-blue-500 { background: blue; } /* 0-1-0 */
```

**Important classes** (prefixed with `!`) use `!important` to always win:

```html
<div class="bg-blue-500 !bg-red-500">  <!-- red wins -->
```

---

### ITCSS (Inverted Triangle CSS)
Organizes CSS from lowest to highest specificity:

```
1. Settings (variables)
2. Tools (mixins)
3. Generic (resets, normalize)    - 0-0-1
4. Elements (bare HTML)            - 0-0-1
5. Objects (layout patterns)       - 0-1-0
6. Components                      - 0-1-1 to 0-2-0
7. Utilities (!important allowed)  - 0-1-0 + !important
```

---

## Common Mistakes and How to Fix Them

### ❌ Mistake 1: Over-qualifying selectors
```css
/* Bad - unnecessary specificity */
div.card { }
ul.nav li.nav-item { }

/* Good */
.card { }
.nav-item { }
```

### ❌ Mistake 2: Using IDs for styling
```css
/* Bad */
#sidebar { width: 300px; }

/* Good */
.sidebar { width: 300px; }
```

### ❌ Mistake 3: Important overuse
```css
/* Bad - creates specificity wars */
.button { 
  background: blue !important; 
}

.button-red {
  background: red !important;  /* Now you need !important everywhere */
}

/* Good - use specificity properly */
.button { background: blue; }
.button-red { background: red; }  /* Works because source order */
```

### ❌ Mistake 4: Fighting specificity with more specificity
```css
/* Bad progression */
.button { color: white; }
.button.button-primary { color: blue; }  /* Someone added .button twice to win */
.button.button-primary.override { color: red; }  /* Specificity arms race */

/* Good - keep it flat */
.button { color: white; }
.button-primary { color: blue; }
.button-danger { color: red; }
```

---

## Performance Considerations

### Selector Performance (least to most expensive)
1. ID: `#header`
2. Class: `.button`
3. Type: `div`
4. Adjacent sibling: `h2 + p`
5. Child: `ul > li`
6. Descendant: `div p`
7. Universal: `*`
8. Attribute: `[type="text"]`
9. Pseudo-classes: `:hover`

**Modern browsers are fast enough** that selector performance rarely matters. **Readability and maintainability** matter more.

**Exception:** `:has()` is more expensive since it queries descendants. Use sparingly in performance-critical scenarios.

---

## Key Takeaways

✅ **Specificity is a three-part weight: ID-CLASS-TYPE**

✅ **The cascade resolves conflicts by origin, specificity, then source order**

✅ **Keep specificity low and consistent for maintainable CSS**

✅ **Use classes for styling, not IDs**

✅ **Inheritance applies to typography and some visibility properties, not layout**

✅ **Use `:where()` for zero-specificity defaults**

✅ **Reserve `!important` for utilities and last resorts**

✅ **Modern tools like `@layer` give you fine-grained cascade control**

---

## Resources

- [MDN: Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity)
- [MDN: Cascade and Inheritance](https://developer.mozilla.org/en-US/docs/Web/CSS/Cascade)
- [CSS Specificity Calculator](https://specificity.keegan.st/)
- [W3C Selectors Level 4](https://www.w3.org/TR/selectors-4/)
- [Cascade Layers Explainer](https://developer.chrome.com/blog/cascade-layers/)
