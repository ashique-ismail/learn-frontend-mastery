# CSS `:is()` and `:where()` Pseudo-classes

## The Idea

**In plain English:** `:is()` and `:where()` are CSS shortcuts that let you group multiple targets together in one rule instead of writing the same style instruction over and over for each one. Think of them as a way to say "apply this style to any of these things" — the only difference between the two is how much "weight" (called specificity) each one carries when CSS decides which rule wins a conflict.

**Real-world analogy:** Imagine a school cafeteria with a sign that says "Students in 9th grade, 10th grade, or 11th grade must show their ID." Instead of writing three separate signs — one for each grade — you group them into one sign.

- The sign itself = the CSS rule (the style you want to apply)
- "9th grade, 10th grade, or 11th grade" = the selector list inside `:is()` or `:where()`
- The grade levels = the individual HTML elements or classes being targeted
- Using `:where()` instead of `:is()` = posting the sign in pencil so anyone can easily erase and override it with a pen

---

## Overview

`:is()` and `:where()` are functional pseudo-classes that simplify selector lists and reduce repetition. They differ only in **specificity**.

## `:is()` Pseudo-class

### Basic Syntax

```css
/* Instead of repeating */
h1 span,
h2 span,
h3 span {
  color: red;
}

/* Use :is() */
:is(h1, h2, h3) span {
  color: red;
}
```

### Specificity

**`:is()` takes the specificity of its most specific argument:**

```css
/* Specificity: (0, 1, 0) — class */
:is(.foo, #bar) {
  /* Specificity is same as #bar (highest in list) */
}

/* This means: */
:is(.class, #id) { color: red; }
/* Has specificity of #id: (1, 0, 0) */
```

### Use Cases

#### Simplifying Complex Selectors

```css
/* Before */
article h1,
article h2,
article h3,
aside h1,
aside h2,
aside h3 {
  color: blue;
}

/* After */
:is(article, aside) :is(h1, h2, h3) {
  color: blue;
}
```

#### Hover States

```css
/* Before */
button:hover,
button:focus,
a:hover,
a:focus {
  outline: 2px solid blue;
}

/* After */
:is(button, a):is(:hover, :focus) {
  outline: 2px solid blue;
}
```

#### Form States

```css
:is(input, textarea, select):is(:focus, :active) {
  border-color: blue;
}

:is(input, textarea, select):is(:invalid, :out-of-range) {
  border-color: red;
}
```

#### BEM-style Modifiers

```css
.button:is(.primary, .secondary, .danger) {
  padding: 1rem 2rem;
  border-radius: 4px;
}

.button.primary { background: blue; }
.button.secondary { background: gray; }
.button.danger { background: red; }
```

## `:where()` Pseudo-class

### Basic Syntax

Same syntax as `:is()`:

```css
:where(h1, h2, h3) span {
  color: red;
}
```

### Specificity

**`:where()` always has zero specificity (0, 0, 0):**

```css
/* Specificity: (0, 0, 0) — always zero! */
:where(.foo, #bar) {
  /* Specificity is 0, regardless of arguments */
}
```

### Use Case: Defaults & Resets

Perfect for creating **easily overridable** defaults:

```css
/* Default link styles (zero specificity) */
:where(a) {
  color: blue;
  text-decoration: underline;
}

/* Easy to override with any selector */
.nav a {
  /* This wins, even though it's just a class + element */
  color: black;
  text-decoration: none;
}
```

#### Reset Styles

```css
/* Reset with zero specificity */
:where(ul, ol) {
  list-style: none;
  padding: 0;
  margin: 0;
}

/* Easy to override */
article ul {
  list-style: disc;
  padding-left: 1.5rem;
}
```

#### Design System Defaults

```css
/* Base button (zero specificity) */
:where(.button) {
  padding: 0.5rem 1rem;
  border: 1px solid currentColor;
  background: transparent;
  cursor: pointer;
}

/* Variants easily override */
.button--primary {
  background: blue;
  color: white;
}
```

## `:is()` vs `:where()` — The Key Difference

| Aspect | `:is()` | `:where()` |
|--------|---------|-----------|
| Specificity | Takes highest specificity of arguments | Always zero (0, 0, 0) |
| Use case | Simplify selectors | Defaults & resets |
| Override | Normal specificity rules | Easy to override |

### Example Comparison

```css
/* :is() — Specificity: (1, 0, 0) from #id */
:is(.class, #id) {
  color: red;
}

/* :where() — Specificity: (0, 0, 0) */
:where(.class, #id) {
  color: blue;
}

/* A single class wins over :where() */
.override {
  color: green; /* Wins! (0, 1, 0) > (0, 0, 0) */
}

/* But loses to :is() */
.override {
  color: green; /* Loses. (0, 1, 0) < (1, 0, 0) */
}
```

## Forgiving Selector Lists

Both `:is()` and `:where()` are **forgiving** — if one selector is invalid, the rest still work:

### Traditional Selector List (Not Forgiving)

```css
/* If any selector is invalid, ENTIRE rule is ignored */
.foo,
.bar,
::-webkit-unknown,
.baz {
  color: red; /* Entire rule ignored due to invalid selector */
}
```

### With `:is()` or `:where()` (Forgiving)

```css
/* Invalid selector is ignored, rest still work */
:is(.foo, .bar, ::-webkit-unknown, .baz) {
  color: red; /* Works for .foo, .bar, .baz */
}
```

**Use case**: Cross-browser vendor prefixes:

```css
:is(
  ::-webkit-input-placeholder,
  ::-moz-placeholder,
  :-ms-input-placeholder,
  ::placeholder
) {
  color: gray;
  /* Works in all browsers; invalid selectors ignored */
}
```

## Practical Examples

### Responsive Navigation

```css
/* Mobile: stack links */
@media (max-width: 768px) {
  :where(nav) :is(ul, ol) {
    flex-direction: column;
  }
}
```

### Focus Styles

```css
:is(a, button, input, textarea, select):focus-visible {
  outline: 2px solid blue;
  outline-offset: 2px;
}
```

### Dark Mode

```css
:where([data-theme="dark"]) {
  --bg: #1a1a1a;
  --text: #f5f5f5;
}

/* Easy to override in specific components */
.card {
  --bg: white;
}
```

### Utility Classes (Zero Specificity)

```css
:where(.mt-0) { margin-top: 0; }
:where(.mt-1) { margin-top: 0.25rem; }
:where(.mt-2) { margin-top: 0.5rem; }

/* Easy to override */
.custom-component {
  margin-top: 2rem; /* Wins */
}
```

### Conditional Styling

```css
/* Style sections with specific classes */
:is(.featured, .highlight, .promo) section {
  background: #f0f0f0;
  padding: 2rem;
}
```

### State Management

```css
/* Multiple states */
:is([aria-pressed="true"], [aria-current="page"], .active) {
  font-weight: bold;
  color: blue;
}
```

## Combining with Other Selectors

### With `:not()`

```css
/* All headings except h1 */
:is(h2, h3, h4, h5, h6):not(.no-margin) {
  margin-top: 1.5em;
}
```

### With Pseudo-elements

```css
:is(h1, h2, h3)::before {
  content: "→ ";
  color: blue;
}
```

### With `:has()`

```css
article:has(:is(img, video, iframe)) {
  display: grid;
  grid-template-columns: 1fr 300px;
}
```

## Nesting `:is()` and `:where()`

You can nest them:

```css
:is(
  article :where(h1, h2),
  section :where(h1, h2)
) {
  color: blue;
}
```

## Performance

Both `:is()` and `:where()` are **highly optimized** by browsers. They're generally as fast or faster than writing out individual selectors.

## Browser Support

Excellent support in modern browsers (2021+).

### Fallbacks

```css
/* Fallback */
h1, h2, h3 {
  color: blue;
}

/* Progressive enhancement */
@supports selector(:is(h1, h2)) {
  :is(h1, h2, h3) {
    color: blue;
  }
}
```

## Common Gotchas

### 1. `:is()` Specificity Can Surprise

```css
:is(.foo, #bar) {
  /* Specificity is (1,0,0) even if .foo matches */
}
```

### 2. Cannot Use Pseudo-elements

```css
/* ❌ Invalid */
:is(::before, ::after) { }

/* ✅ Use separately */
::before, ::after { }
```

### 3. Combinators Outside

```css
/* ✅ Combinator outside */
:is(article, section) > h1 { }

/* ❌ Cannot put combinator inside */
:is(article >, section >) h1 { } /* Invalid */
```

## When to Use Which?

### Use `:is()` when:
- Simplifying complex selectors
- Normal specificity behavior desired
- Grouping similar selectors

### Use `:where()` when:
- Creating defaults or resets
- You want easy overridability
- Building utility classes
- Framework/library defaults

### Use Both Together:

```css
/* Default (zero specificity) */
:where(.button) {
  padding: 0.5rem 1rem;
}

/* Variants (normal specificity) */
.button:is(.primary, .secondary) {
  border-radius: 4px;
}
```

## Interview Questions to Master

1. What's the only difference between `:is()` and `:where()`?
2. What specificity does `:where()` have?
3. How does `:is()` calculate its specificity?
4. What does "forgiving selector list" mean?
5. Can you use pseudo-elements inside `:is()` or `:where()`?
6. When would you choose `:where()` over `:is()`?
7. How do `:is()` and `:where()` handle invalid selectors?

## Resources

- [MDN :is()](https://developer.mozilla.org/en-US/docs/Web/CSS/:is)
- [MDN :where()](https://developer.mozilla.org/en-US/docs/Web/CSS/:where)
- [CSS Tricks: :is() and :where()](https://css-tricks.com/almanac/selectors/i/is/)
- [web.dev: New CSS pseudo-classes](https://web.dev/css-is-and-where/)
