# em vs rem

## The Difference

**`rem`** — relative to the **root element** (`<html>`) font size. Always the same base.

**`em`** — relative to the **current element's** font size (or the parent's, for font-size itself).

```css
:root { font-size: 16px; } /* default browser size */

/* rem — always based on 16px root */
h1 { font-size: 2rem; }     /* 32px — always */
.card { padding: 1rem; }     /* 16px — always */

/* em — based on current element's font-size */
.card {
  font-size: 1.125em;  /* 18px (relative to parent) */
  padding: 1em;        /* 18px (relative to THIS element's 18px) */
}
```

---

## The Compounding Problem with `em`

`em` multiplies up through nested elements:

```css
body { font-size: 16px; }
.container { font-size: 1.2em; }  /* 19.2px */
.box { font-size: 1.2em; }        /* 23.04px (1.2 × 19.2) */
.text { font-size: 1.2em; }       /* 27.648px (1.2 × 23.04) */
```

Each level multiplies the parent's computed value. Deep nesting causes font size creep.

`rem` avoids this completely — always relative to `:root`.

---

## When to Use `rem`

**Font sizes** — consistent scale that users can adjust (browser zoom):
```css
body { font-size: 1rem; }    /* 16px = browser default */
h1 { font-size: 2rem; }      /* 32px */
small { font-size: 0.875rem; } /* 14px */
```

**Global spacing** — predictable, consistent:
```css
.section { margin-bottom: 2rem; }
.container { max-width: 80rem; padding: 0 1.5rem; }
```

---

## When to Use `em`

**Component-internal spacing** — when you want padding/margin to scale with the component's font size:

```css
.button {
  font-size: 1rem;
  padding: 0.75em 1.5em; /* scales if button font-size changes */
  border-radius: 0.25em;
}

.button.large {
  font-size: 1.25rem; /* only change font-size */
  /* padding, border-radius automatically scale up proportionally */
}
```

**Media queries for text-based breakpoints:**
```css
@media (min-width: 48em) { /* 768px at default size */
  /* em in media queries is based on browser default (16px), not :root */
}
```

---

## The 62.5% Trick (Anti-Pattern Now)

```css
/* Old approach: make 1rem = 10px for easy math */
:root { font-size: 62.5%; }  /* 16px × 62.5% = 10px */
h1 { font-size: 3.2rem; }    /* 32px */

/* Problems: breaks user font-size preferences, adds complexity */
```

Modern CSS is readable without this trick. Just use `rem` directly.

---

## User Accessibility: Why Relative Units Matter

```css
/* ❌ Ignores user font-size preference */
body { font-size: 16px; }

/* ✅ Respects user preference */
body { font-size: 1rem; } /* user can set 20px in browser settings */
```

Users with vision impairments increase their browser's default font size. `rem` and `em` scale with it; `px` does not.

**Browser zoom** scales everything including `px`, but **user font-size preference** (Settings → Font Size) only scales `rem`/`em`.

---

## `clamp()` Combining Units

Modern fluid typography combines `rem` for minimum/maximum with viewport units for fluid scaling:

```css
h1 {
  font-size: clamp(1.5rem, 4vw, 3rem);
  /* min: 24px, fluid: 4% viewport width, max: 48px */
}
```

---

## Common Interview Questions

**Q: What's the base for `em` in a media query?**
The browser's default font size (typically 16px), NOT the `<html>` element's font size. This is a quirk — even if you set `html { font-size: 20px }`, `em` in media queries is still 16px.

**Q: When does `em` reference the element's own font-size vs the parent's?**
For `font-size` itself, `em` is relative to the **inherited** (parent's) font-size. For all other properties (padding, margin, width, etc.), `em` is relative to the **element's own** computed font-size.

**Q: Should you ever mix px and rem?**
For borders (1px), box shadows (subtle values), and icon sizes at specific pixel targets, `px` is fine. These don't need to scale with font size. Use `rem`/`em` for text and spacing.
