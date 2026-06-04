# Fluid Typography: clamp(), Viewport Units, Accessibility

## Learning Objectives
- Master fluid typography with `clamp()` function
- Use viewport units (vw, vh, vmin, vmax) effectively
- Calculate optimal scaling ratios for readability
- Balance fluid typography with accessibility requirements
- Implement responsive type scales
- Avoid common fluid typography pitfalls

---

## What is Fluid Typography?

**Fluid typography** scales font sizes smoothly between a minimum and maximum value based on viewport width, without media queries.

**Traditional (stepped):**
```css
body {
  font-size: 16px;
}

@media (min-width: 768px) {
  body {
    font-size: 18px;
  }
}

@media (min-width: 1024px) {
  body {
    font-size: 20px;
  }
}
```

**Fluid (smooth):**
```css
body {
  font-size: clamp(16px, 2vw, 20px);
  /* Scales smoothly from 16px to 20px */
}
```

**Visual:**
```
Traditional:
16px ━━━━━┓
          ┃
          ┗━━━ 18px ━━━━━┓
                         ┃
                         ┗━━━ 20px ━━━

Fluid:
16px ━━━━━━━━━━━━━━━━━━━━━━━━━ 20px
     ↗ smooth scaling ↗
```

---

## `clamp()`: The Modern Way

### Syntax

```css
clamp(MIN, PREFERRED, MAX)
```

**Parts:**
- `MIN` - Minimum value (smallest size)
- `PREFERRED` - Ideal value (usually with viewport units)
- `MAX` - Maximum value (largest size)

**How it works:**
1. If `PREFERRED < MIN`, use `MIN`
2. If `PREFERRED > MAX`, use `MAX`
3. Otherwise, use `PREFERRED`

---

### Basic Example

```css
h1 {
  font-size: clamp(2rem, 5vw, 4rem);
}
```

**Breakdown:**
- **Minimum:** `2rem` (32px) on small screens
- **Preferred:** `5vw` (5% of viewport width)
- **Maximum:** `4rem` (64px) on large screens

**At different viewport widths:**
```
320px viewport: 5vw = 16px → clamps to 32px (MIN)
640px viewport: 5vw = 32px → uses 32px
800px viewport: 5vw = 40px → uses 40px
1280px viewport: 5vw = 64px → uses 64px
1920px viewport: 5vw = 96px → clamps to 64px (MAX)
```

---

### Complete Type Scale

```css
:root {
  /* Base size */
  --font-size-base: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
  
  /* Headings */
  --font-size-h1: clamp(2rem, 1.5rem + 2.5vw, 4rem);
  --font-size-h2: clamp(1.75rem, 1.25rem + 2vw, 3rem);
  --font-size-h3: clamp(1.5rem, 1rem + 1.5vw, 2.5rem);
  --font-size-h4: clamp(1.25rem, 0.875rem + 1vw, 2rem);
  --font-size-h5: clamp(1.125rem, 0.75rem + 0.75vw, 1.5rem);
  --font-size-h6: clamp(1rem, 0.75rem + 0.5vw, 1.25rem);
  
  /* Utility sizes */
  --font-size-small: clamp(0.875rem, 0.75rem + 0.25vw, 1rem);
  --font-size-large: clamp(1.25rem, 1rem + 0.75vw, 1.75rem);
}

body {
  font-size: var(--font-size-base);
}

h1 { font-size: var(--font-size-h1); }
h2 { font-size: var(--font-size-h2); }
h3 { font-size: var(--font-size-h3); }
h4 { font-size: var(--font-size-h4); }
h5 { font-size: var(--font-size-h5); }
h6 { font-size: var(--font-size-h6); }

small { font-size: var(--font-size-small); }
.lead { font-size: var(--font-size-large); }
```

---

## Viewport Units

### Unit Types

| Unit | Meaning | Example |
|------|---------|---------|
| `vw` | 1% of viewport width | `5vw` = 5% of width |
| `vh` | 1% of viewport height | `10vh` = 10% of height |
| `vmin` | 1% of smaller dimension | `5vmin` = 5% of min(width, height) |
| `vmax` | 1% of larger dimension | `5vmax` = 5% of max(width, height) |

---

### `vw` - Viewport Width (Most Common)

```css
h1 {
  font-size: 5vw;
}
```

**At different viewports:**
```
320px viewport: 5vw = 16px
768px viewport: 5vw = 38.4px
1024px viewport: 5vw = 51.2px
1920px viewport: 5vw = 96px
```

**Problem:** Unbounded - too small on mobile, too large on desktop.

**Solution:** Use `clamp()`:
```css
h1 {
  font-size: clamp(2rem, 5vw, 4rem);
}
```

---

### `vh` - Viewport Height

```css
.hero {
  height: 100vh;
  font-size: 5vh;
}
```

**Use case:** Full-screen sections, hero text that scales with height.

**Warning:** Mobile browsers have dynamic toolbars that change `vh`. Consider `dvh` (dynamic vh) if supported.

---

### `vmin` - Smaller Dimension

```css
h1 {
  font-size: clamp(2rem, 5vmin, 4rem);
}
```

**Use case:** Ensuring text scales relative to the **smaller** dimension (useful for landscape/portrait switching).

**Example:**
```
1024x768 viewport: 5vmin = 5% of 768 = 38.4px
768x1024 viewport: 5vmin = 5% of 768 = 38.4px
```

---

### `vmax` - Larger Dimension

```css
.hero-title {
  font-size: clamp(2rem, 5vmax, 6rem);
}
```

**Use case:** Text that should scale with the **larger** dimension.

---

### New Viewport Units (Modern)

| Unit | Meaning | Browser Support |
|------|---------|-----------------|
| `svh` | Small viewport height (static) | Chrome 108+, Safari 15.4+ |
| `lvh` | Large viewport height (max) | Chrome 108+, Safari 15.4+ |
| `dvh` | Dynamic viewport height | Chrome 108+, Safari 15.4+ |

**Use case:** Avoiding mobile browser toolbar issues.

```css
/* Old - jumps when toolbar hides/shows */
.hero {
  height: 100vh;
}

/* New - adapts to dynamic height */
.hero {
  height: 100dvh;
}
```

---

## Calculating Fluid Typography

### Formula

**Goal:** Font size that scales linearly between two viewports.

**Formula:**
```
font-size = MIN_SIZE + (MAX_SIZE - MIN_SIZE) × ((100vw - MIN_VIEWPORT) / (MAX_VIEWPORT - MIN_VIEWPORT))
```

**Example:**
- Min: 16px at 320px viewport
- Max: 24px at 1200px viewport

**Calculation:**
```
font-size = 16px + (24 - 16) × ((100vw - 320px) / (1200 - 320))
font-size = 16px + 8 × ((100vw - 320px) / 880)
font-size = 16px + 0.909vw - 2.909px
font-size = 13.091px + 0.909vw
```

**In CSS:**
```css
body {
  font-size: clamp(16px, 13.091px + 0.909vw, 24px);
}
```

---

### Online Calculators

**Tools to generate `clamp()` values:**
- [Modern Fluid Typography Editor](https://modern-fluid-typography.vercel.app/)
- [Utopia Fluid Type Scale Calculator](https://utopia.fyi/type/calculator/)
- [Clamp Calculator by Andy Bell](https://andy-bell.co.uk/fluid-type-scale-calculator/)

**Example output:**
```css
/* 16px @ 320px → 24px @ 1200px */
font-size: clamp(1rem, 0.8182rem + 0.9091vw, 1.5rem);
```

---

## Accessibility Considerations

### Problem: Viewport Units Ignore User Zoom

**When a user zooms in (Cmd/Ctrl +):**
- `px` values scale (good)
- `rem`/`em` values scale (good)
- `vw`/`vh` values **don't scale** (bad)

**Example:**
```css
/* BAD - doesn't respect user zoom */
body {
  font-size: 2vw;
}
```

**At 320px viewport:**
- No zoom: `2vw = 6.4px` (too small!)
- 200% zoom: `2vw = 6.4px` (still too small - didn't scale!)

---

### Solution: Use `rem` in `clamp()`

**Good:**
```css
body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
}
```

**Why it works:**
- `1rem` = 16px by default
- User zooms to 200%: `1rem` = 32px
- `clamp()` respects `rem` min/max, so zoom works

**At 320px viewport:**
- No zoom: `clamp(16px, 14.4px + 1.6px, 20px)` = 16px
- 200% zoom: `clamp(32px, 28.8px + 1.6px, 40px)` = 32px ✓

---

### Minimum Font Size

**WCAG 2.1 Success Criterion 1.4.4 (AA):** Text can be resized up to 200% without loss of content or functionality.

**Recommendation:** Never go below `1rem` (16px) for body text.

```css
/* BAD - too small */
body {
  font-size: clamp(0.75rem, 2vw, 1.5rem);
  /* Can be as small as 12px */
}

/* GOOD - respects minimum */
body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
  /* At least 16px */
}
```

---

### `em` vs `rem` in `clamp()`

**`rem`** (root em) - Relative to root `<html>` font size:
```css
body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
}
/* 1rem = 16px (or user's browser default) */
```

**`em`** - Relative to parent font size:
```css
.card {
  font-size: 1.25rem;
}

.card-title {
  font-size: clamp(1em, 0.9em + 0.5vw, 1.5em);
  /* Relative to .card (1.25rem), so 1em = 20px */
}
```

**Best practice:** Use `rem` for consistency and accessibility.

---

## Practical Examples

### Example 1: Blog Post

```css
:root {
  --font-size-body: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  --font-size-h1: clamp(2rem, 1.5rem + 2.5vw, 3.5rem);
  --font-size-h2: clamp(1.5rem, 1.25rem + 1.5vw, 2.5rem);
  --line-height-body: 1.6;
  --line-height-heading: 1.2;
}

body {
  font-size: var(--font-size-body);
  line-height: var(--line-height-body);
  max-width: 70ch; /* Optimal reading width */
  margin: 0 auto;
  padding: 0 1rem;
}

h1 {
  font-size: var(--font-size-h1);
  line-height: var(--line-height-heading);
  margin-bottom: 1rem;
}

h2 {
  font-size: var(--font-size-h2);
  line-height: var(--line-height-heading);
  margin-top: 2rem;
  margin-bottom: 1rem;
}
```

---

### Example 2: Hero Section

```css
.hero {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100dvh;
  padding: 2rem;
  text-align: center;
}

.hero-title {
  font-size: clamp(2.5rem, 2rem + 5vw, 6rem);
  line-height: 1.1;
  margin-bottom: 1rem;
}

.hero-subtitle {
  font-size: clamp(1rem, 0.9rem + 1vw, 1.5rem);
  line-height: 1.5;
  max-width: 60ch;
}
```

---

### Example 3: Card Grid

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 2rem;
  padding: 2rem;
}

.card {
  padding: 1.5rem;
  border-radius: 8px;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.card-title {
  font-size: clamp(1.25rem, 1rem + 1vw, 1.75rem);
  margin-bottom: 0.5rem;
}

.card-body {
  font-size: clamp(0.875rem, 0.8rem + 0.25vw, 1rem);
  line-height: 1.6;
}
```

---

### Example 4: Responsive Spacing

**Fluid spacing matches fluid typography:**

```css
:root {
  --space-xs: clamp(0.5rem, 0.4rem + 0.25vw, 0.75rem);
  --space-sm: clamp(0.75rem, 0.6rem + 0.5vw, 1.25rem);
  --space-md: clamp(1rem, 0.8rem + 1vw, 2rem);
  --space-lg: clamp(1.5rem, 1rem + 2vw, 3rem);
  --space-xl: clamp(2rem, 1.5rem + 2.5vw, 4rem);
}

section {
  padding: var(--space-lg) var(--space-md);
}

h1 {
  margin-bottom: var(--space-md);
}

p + p {
  margin-top: var(--space-sm);
}
```

---

## Line Height and Fluid Typography

### Adjust Line Height with Font Size

**Principle:** Larger text needs tighter line height, smaller text needs more space.

```css
/* Small text - needs more line height */
body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  line-height: 1.6;
}

/* Medium headings */
h2 {
  font-size: clamp(1.5rem, 1.25rem + 1.5vw, 2.5rem);
  line-height: 1.3;
}

/* Large headings - tighter line height */
h1 {
  font-size: clamp(2rem, 1.5rem + 2.5vw, 4rem);
  line-height: 1.1;
}
```

---

### Fluid Line Height (Advanced)

```css
h1 {
  font-size: clamp(2rem, 1.5rem + 2.5vw, 4rem);
  line-height: clamp(1.1, 1.05 + 0.25vw, 1.3);
  /* Line height scales from 1.1 to 1.3 */
}
```

**Warning:** Rarely needed - fixed line heights are usually better for readability.

---

## Combining with Media Queries

**Fluid typography doesn't replace media queries - combine them:**

```css
body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  padding: 1rem;
}

/* Mobile-specific adjustments */
@media (max-width: 767px) {
  body {
    padding: 0.5rem;
  }
  
  /* Tighter clamp for small screens */
  h1 {
    font-size: clamp(1.75rem, 1.5rem + 1.5vw, 2.5rem);
  }
}

/* Desktop-specific adjustments */
@media (min-width: 1024px) {
  body {
    padding: 2rem;
  }
  
  /* Wider clamp for large screens */
  h1 {
    font-size: clamp(2.5rem, 2rem + 2.5vw, 5rem);
  }
}
```

---

## Common Mistakes

### ❌ Mistake 1: No Min/Max Bounds

```css
/* BAD - unbounded */
body {
  font-size: 2vw;
}
```

**Problem:**
- 320px viewport: `2vw = 6.4px` (too small!)
- 1920px viewport: `2vw = 38.4px` (too large!)

**Fix:**
```css
/* GOOD - bounded */
body {
  font-size: clamp(1rem, 2vw, 1.5rem);
}
```

---

### ❌ Mistake 2: Too Aggressive Scaling

```css
/* BAD - scales too fast */
h1 {
  font-size: clamp(2rem, 10vw, 10rem);
}
```

**Problem:** At 1000px viewport, `10vw = 100px` - way too large!

**Fix:**
```css
/* GOOD - conservative scaling */
h1 {
  font-size: clamp(2rem, 1.5rem + 2.5vw, 4rem);
}
```

---

### ❌ Mistake 3: Forgetting Accessibility

```css
/* BAD - minimum too small */
body {
  font-size: clamp(0.75rem, 2vw, 1.5rem);
}
```

**Problem:** `0.75rem = 12px` - too small for many users.

**Fix:**
```css
/* GOOD - respects minimum */
body {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);
}
```

---

### ❌ Mistake 4: Only Using `vw` in Preferred Value

```css
/* BAD - hard to reason about */
h1 {
  font-size: clamp(2rem, 5vw, 4rem);
}
```

**Better - explicit calculation:**
```css
/* GOOD - shows scaling relationship */
h1 {
  font-size: clamp(2rem, 1.5rem + 2.5vw, 4rem);
  /* Starts at 1.5rem, adds 2.5vw */
}
```

---

## Testing Fluid Typography

### DevTools Responsive Mode

1. Open DevTools (F12)
2. Toggle device toolbar (Cmd+Shift+M)
3. Resize viewport smoothly
4. Watch font sizes change in real-time

---

### Check Specific Viewports

**Test key viewports:**
- 320px (small phone)
- 375px (iPhone)
- 768px (tablet portrait)
- 1024px (tablet landscape / small laptop)
- 1440px (desktop)
- 1920px (large desktop)

---

### Accessibility Testing

**Check with user zoom:**
1. Set viewport to 1024px
2. Zoom to 200% (Cmd/Ctrl +)
3. Verify text is still readable
4. Check if layout breaks

**Expected behavior:**
- Text should scale up
- Layout should adapt (may require media queries)
- No horizontal scrolling

---

## Browser Support (2026)

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| `clamp()` | 79+ | 75+ | 13.1+ | 79+ |
| `vw`, `vh`, `vmin`, `vmax` | All | All | All | All |
| `dvh`, `svh`, `lvh` | 108+ | 120+ | 15.4+ | 108+ |

**Verdict:** `clamp()` and viewport units are universally supported in 2026.

---

## Tools and Resources

### Fluid Typography Calculators

- [Utopia Fluid Type Scale](https://utopia.fyi/type/calculator/)
- [Modern Fluid Typography](https://modern-fluid-typography.vercel.app/)
- [Type Scale](https://typescale.com/)
- [Fluid Typography Calculator](https://websemantics.uk/tools/fluid-font-size-calculator/)

---

### Recommended Type Scales

**Major Third (1.25):**
```css
--font-size-h6: 1rem;
--font-size-h5: 1.25rem;
--font-size-h4: 1.563rem;
--font-size-h3: 1.953rem;
--font-size-h2: 2.441rem;
--font-size-h1: 3.052rem;
```

**Perfect Fourth (1.333):**
```css
--font-size-h6: 1rem;
--font-size-h5: 1.333rem;
--font-size-h4: 1.777rem;
--font-size-h3: 2.369rem;
--font-size-h2: 3.157rem;
--font-size-h1: 4.209rem;
```

**Golden Ratio (1.618):**
```css
--font-size-h6: 1rem;
--font-size-h5: 1.618rem;
--font-size-h4: 2.618rem;
--font-size-h3: 4.236rem;
--font-size-h2: 6.854rem;
--font-size-h1: 11.089rem;
```

---

## Key Takeaways

✅ **Use `clamp()` for fluid typography** - smooth scaling between min and max

✅ **Always set min and max bounds** - never use unbounded `vw` values

✅ **Use `rem` for accessibility** - respects user zoom settings

✅ **Keep minimum at 1rem (16px)** - for body text accessibility

✅ **Adjust line height with font size** - larger text needs tighter line height

✅ **Test at multiple viewports** - ensure readability at all sizes

✅ **Combine with media queries** - for layout changes and fine-tuning

✅ **Use online calculators** - generate precise `clamp()` values

---

## Resources

- [MDN: clamp()](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp)
- [MDN: Viewport Units](https://developer.mozilla.org/en-US/docs/Web/CSS/length#viewport-percentage_lengths)
- [Utopia: Fluid Typography](https://utopia.fyi/blog/designing-with-fluid-type-scales/)
- [CSS Tricks: Linearly Scale font-size](https://css-tricks.com/linearly-scale-font-size-with-css-clamp-based-on-the-viewport/)
- [Smashing Magazine: Fluid Typography](https://www.smashingmagazine.com/2022/01/modern-fluid-typography-css-clamp/)
- [Adrian Bece: Practical Fluid Typography](https://www.smashingmagazine.com/2022/08/fluid-sizing-multiple-media-queries/)
