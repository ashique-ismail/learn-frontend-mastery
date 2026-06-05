# CSS Specificity Calculation

## The Idea

**In plain English:** Specificity is the browser's tiebreaker system — when two CSS rules try to style the same element differently, specificity decides which rule wins by giving each selector a score and picking the highest one.

**Real-world analogy:** Think of a school's dress-code policy. A note from the principal overrides the class teacher's instruction, which overrides the general school handbook, which overrides whatever a student prefers. The closer and more direct the authority, the higher the rank — even if ten handbook rules exist, one principal's note beats them all.

- The principal's note = an inline style (`style="..."`) — the most direct, highest authority
- The class teacher's instruction = an ID selector (`#id`) — specific to one individual
- The general school handbook = a class or attribute selector (`.class`, `[attr]`) — applies to a group
- The student's personal preference = an element selector (`p`, `div`) — the broadest, lowest authority

---

## The Scoring System

Specificity is a three-column value: **(A, B, C)**

| Selector type | Column | Value |
|---|---|---|
| Inline styles (`style="..."`) | A | 1 |
| ID selectors (`#id`) | B | 1 per ID |
| Class, attribute, pseudo-class | C | 1 per class/attr/pseudo-class |
| Element, pseudo-element | C | 1 per element |
| Universal (`*`), combinators (`>`, `+`, `~`), `:where()` | — | 0 |

Columns are compared left to right. `(1,0,0)` beats `(0,99,99)`.

---

## Examples

```css
/* (0, 0, 1) */
p { color: blue; }

/* (0, 0, 2) */
p.active { color: blue; }

/* (0, 1, 0) */
#header { color: blue; }

/* (0, 1, 1) */
#header p { color: blue; }

/* (0, 2, 1) */
.nav.active a:hover { color: blue; }
/* 2 classes + 1 pseudo-class = C:3... wait: */
/* .nav = C+1, .active = C+1, :hover = C+1, a = type+1 → (0,3,1) */

/* (1, 0, 0) — beats everything below */
<p style="color: blue">          
```

---

## `!important`

Overrides the cascade entirely. A rule with `!important` beats all non-important rules:

```css
p { color: red !important; }       /* wins over... */
#id.class p { color: blue; }      /* ...even higher specificity without !important */
```

`!important` vs `!important`: the more specific rule wins within the important layer.

**When to use:** Almost never. Exceptions: utility classes you must guarantee always apply (e.g. `.hidden { display: none !important }`), overriding third-party inline styles.

---

## `:is()`, `:not()`, `:has()` — Take Highest Specificity of Argument

```css
/* :is() takes the highest specificity of its argument list */
:is(#header, .nav) a { color: blue; }
/* Specificity: (1, 0, 1) because #header is in the argument */
/* Even if no #header element exists — specificity comes from the selector, not the match */

/* :not() same rule */
:not(#id) { color: blue; }  /* (1, 0, 0) */
```

---

## `:where()` — Zero Specificity

```css
/* :where() always has (0, 0, 0) specificity */
:where(#header, .nav) a { color: blue; }
/* Specificity: (0, 0, 1) — only the 'a' contributes */
```

Useful in design systems for base styles you want users to easily override.

---

## Cascade Layers (`@layer`)

Layers exist *below* specificity in the cascade. A rule in an unlayered stylesheet beats all `@layer` rules regardless of specificity:

```css
@layer base {
  #id { color: red; }    /* (1,0,0) in base layer */
}

/* Unlayered — wins despite lower specificity */
.class { color: blue; } /* (0,1,0) but unlayered → wins */
```

Layer order (last declared wins within same specificity):
```css
@layer base, components, utilities;
/* utilities rules beat components rules of equal specificity */
```

---

## Interview Quick-Fire

**Q: Which wins?**
```css
#app .button { color: red; }    /* (0, 1, 1) */
.header #nav a { color: blue; } /* (0, 1, 1) + element = (0,1,2)? */
```
Wait: `.header` = C+1, `#nav` = B+1, `a` = type → **(0, 1, 2)** for the second, **(0, 1, 1)** for the first. Blue wins.

**Q: How do you override an inline style without `!important`?**
You can't with regular CSS. Add a class that applies `!important`, or eliminate the inline style at the source (better approach).

**Q: Does `:nth-child(2n+1)` count as one pseudo-class?**
Yes — one pseudo-class = C+1 regardless of its argument complexity.
