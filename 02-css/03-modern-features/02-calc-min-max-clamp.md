# CSS Math Functions: `calc()`, `min()`, `max()`, `clamp()`

## The Idea

**In plain English:** CSS math functions let you write simple math directly in your styles so the browser can figure out sizes on the fly. Instead of picking one fixed number, you give the browser a formula — like "take the full width and subtract 50 pixels" — and it calculates the right value for every screen automatically.

**Real-world analogy:** Think of adjusting a shelf to fit inside a wardrobe. You measure the wardrobe's total inside width, then subtract a few centimetres on each side so the shelf slides in cleanly. You don't know the exact number until you measure, but the rule — "total width minus the gap on each side" — always works no matter which wardrobe you try it in.

- The wardrobe's inside width = `100%` (the parent container's available space)
- The gap you subtract on each side = `50px` (a fixed offset)
- The rule you apply = `calc(100% - 50px)` (the CSS math formula)

---

## Overview

CSS math functions allow you to perform calculations directly in CSS, enabling responsive and dynamic layouts without JavaScript.

## `calc()`

Perform calculations with mixed units.

### Basic Syntax

```css
.element {
  width: calc(100% - 50px);
  padding: calc(1rem + 5px);
  margin: calc(var(--space) * 2);
}
```

### Supported Operations

- **Addition**: `+`
- **Subtraction**: `-`
- **Multiplication**: `*`
- **Division**: `/`

**Important**: `+` and `-` **must have whitespace** on both sides:

```css
/* ✅ Correct */
width: calc(100% - 50px);
width: calc(100% + 50px);

/* ❌ Incorrect */
width: calc(100%-50px);
width: calc(100% -50px); /* Ambiguous: negative number or subtraction? */
```

Multiplication and division don't require spaces but recommended for readability:

```css
/* Both valid */
width: calc(100% * 0.5);
width: calc(100%*0.5);
```

### Mixing Units

```css
.element {
  /* Percentage + fixed */
  width: calc(100% - 300px);
  
  /* Viewport + rem */
  height: calc(100vh - 5rem);
  
  /* Custom property + px */
  padding: calc(var(--spacing) + 10px);
  
  /* Nested calc */
  margin: calc(calc(100% / 3) - 20px);
}
```

### Common Use Cases

#### Fluid Layouts with Fixed Sidebars

```css
.container {
  display: flex;
}

.sidebar {
  width: 250px;
}

.main {
  width: calc(100% - 250px);
}
```

#### Centering with Known Width

```css
.centered {
  position: absolute;
  left: calc(50% - 200px); /* Center a 400px wide element */
  width: 400px;
}
```

#### Responsive Typography

```css
h1 {
  font-size: calc(1.5rem + 2vw);
}
```

#### Grid Column Widths

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, calc((100% - 40px) / 3));
  gap: 20px;
}
```

## `min()`

Returns the **smallest** value from a list.

### Syntax

```css
.element {
  width: min(500px, 100%);
  /* If container < 500px: width = 100%
     If container > 500px: width = 500px */
}
```

### Use Cases

#### Responsive Max-Width (No Media Query)

```css
.container {
  width: min(1200px, 100%);
  /* Same as: max-width: 1200px; width: 100%; */
}
```

#### Fluid Padding

```css
.section {
  padding: min(5vw, 3rem);
  /* Padding scales with viewport but never exceeds 3rem */
}
```

#### Responsive Grid

```css
.grid {
  grid-template-columns: repeat(auto-fit, min(300px, 100%));
}
```

### Multiple Values

```css
.element {
  width: min(500px, 50vw, 100%);
  /* Uses the smallest of three values */
}
```

## `max()`

Returns the **largest** value from a list.

### Syntax

```css
.element {
  width: max(300px, 50%);
  /* If 50% < 300px: width = 300px
     If 50% > 300px: width = 50% */
}
```

### Use Cases

#### Minimum Width

```css
.button {
  width: max(150px, 100%);
  /* Button is at least 150px wide, even if parent is narrower */
}
```

#### Readable Line Length

```css
.text {
  max-width: max(45ch, 300px);
  /* At least 300px, or 45 characters, whichever is larger */
}
```

#### Grid Track Sizing

```css
.grid {
  grid-template-columns: repeat(3, max(200px, 1fr));
}
```

### Multiple Values

```css
.element {
  height: max(200px, 50vh, 300px);
  /* Uses the largest of three values */
}
```

## `clamp()`

Clamps a value between a **minimum** and **maximum**.

### Syntax

```css
.element {
  font-size: clamp(1rem, 2vw, 2rem);
  /* MIN, PREFERRED, MAX */
}
```

**How it works**:
- If `PREFERRED` < `MIN`, use `MIN`
- If `PREFERRED` > `MAX`, use `MAX`
- Otherwise, use `PREFERRED`

### Equivalent to

```css
clamp(MIN, VAL, MAX)
/* = */
max(MIN, min(VAL, MAX))
```

### Use Cases

#### Fluid Typography

```css
h1 {
  font-size: clamp(2rem, 5vw, 4rem);
  /* Scales between 2rem and 4rem based on viewport */
}

p {
  font-size: clamp(1rem, 0.875rem + 0.5vw, 1.25rem);
}
```

#### Responsive Spacing

```css
.section {
  padding: clamp(1rem, 5vw, 5rem);
}

.container {
  gap: clamp(1rem, 3vw, 3rem);
}
```

#### Container Width

```css
.container {
  width: clamp(320px, 90%, 1200px);
  /* Between 320px and 1200px, defaulting to 90% */
}
```

#### Flexible Grid Columns

```css
.grid {
  grid-template-columns: repeat(auto-fit, clamp(250px, 30%, 400px));
}
```

## Nesting Math Functions

You can nest any combination:

```css
.element {
  width: clamp(
    200px,
    calc(50% - 2rem),
    max(400px, 30vw)
  );
  
  padding: max(
    1rem,
    min(5vw, 3rem)
  );
}
```

## Advanced Patterns

### Fluid Typography Scale

```css
:root {
  --min-viewport: 320px;
  --max-viewport: 1200px;
  --min-font-size: 16px;
  --max-font-size: 20px;
}

body {
  font-size: clamp(
    var(--min-font-size),
    calc(var(--min-font-size) + (var(--max-font-size) - var(--min-font-size)) * ((100vw - var(--min-viewport)) / (var(--max-viewport) - var(--min-viewport)))),
    var(--max-font-size)
  );
}
```

### Responsive Container with Padding

```css
.container {
  width: min(1200px, 100% - 2rem);
  /* Max 1200px, but always has 1rem padding on each side */
  margin-inline: auto;
}
```

### Aspect Ratio Box (Pre-aspect-ratio property)

```css
.box {
  width: 100%;
  height: 0;
  padding-bottom: calc(100% / (16 / 9)); /* 16:9 aspect ratio */
}
```

### Responsive Font Size with Linear Scaling

```css
.heading {
  font-size: clamp(1.5rem, 1rem + 2vw, 3rem);
  /* At 0px viewport: 1rem (clamped to 1.5rem)
     At 50vw (800px): 1rem + 16px = 2rem
     At 100vw: 1rem + 32px (clamped to 3rem) */
}
```

## Comparison Table

| Function | Purpose | Example Use |
|----------|---------|-------------|
| `calc()` | Mixed-unit calculations | `calc(100% - 50px)` |
| `min()` | Choose smallest value (max-width equivalent) | `min(1200px, 100%)` |
| `max()` | Choose largest value (min-width equivalent) | `max(300px, 50%)` |
| `clamp()` | Bound value between min and max | `clamp(1rem, 2vw, 3rem)` |

## Browser Support

- **`calc()`**: Excellent (IE9+)
- **`min()` / `max()`**: Modern browsers (2020+)
- **`clamp()`**: Modern browsers (2020+)

### Fallbacks

```css
.element {
  font-size: 1.5rem; /* Fallback */
  font-size: clamp(1rem, 2vw, 3rem);
}

@supports (font-size: clamp(1rem, 2vw, 3rem)) {
  .element {
    font-size: clamp(1rem, 2vw, 3rem);
  }
}
```

## Gotchas

### 1. Division Must Use `/` (Not `÷`)

```css
/* ✅ */
width: calc(100% / 3);

/* ❌ */
width: calc(100% ÷ 3);
```

### 2. Percentages in `calc()` Resolve Based on Context

```css
.parent {
  width: 500px;
}

.child {
  width: calc(50% + 20px); /* 50% of parent = 250px, + 20px = 270px */
}
```

### 3. Cannot Mix Incompatible Units

```css
/* ❌ Can't add angle and length */
transform: rotate(calc(45deg + 10px)); /* Invalid */

/* ✅ Same unit types */
transform: rotate(calc(45deg + 15deg));
```

### 4. `min()`/`max()` with Single Value

```css
/* Still valid, but pointless */
width: min(500px); /* Just use width: 500px */
```

### 5. Operator Precedence

```css
/* Multiplication before addition */
calc(100% - 20px * 2) /* = 100% - 40px */

/* Use parentheses for clarity */
calc((100% - 20px) * 2)
```

## Tools and Generators

- **[Fluid Type Scale Calculator](https://www.fluid-type-scale.com/)**
- **[Modern Fluid Typography](https://modern-fluid-typography.vercel.app/)**
- **[Utopia](https://utopia.fyi/)** - Fluid responsive design

## Interview Questions to Master

1. What's the difference between `calc(100% - 20px)` and `calc(100%-20px)`?
2. How is `clamp(A, B, C)` equivalent to nested `min()`/`max()`?
3. When would you use `min()` vs `max()`?
4. How can `clamp()` eliminate media queries for typography?
5. Can you mix different unit types in `calc()`?
6. What's the operator precedence in `calc()`?
7. How would you create a fluid spacing scale using `clamp()`?

## Resources

- [MDN calc()](https://developer.mozilla.org/en-US/docs/Web/CSS/calc)
- [MDN min()](https://developer.mozilla.org/en-US/docs/Web/CSS/min)
- [MDN max()](https://developer.mozilla.org/en-US/docs/Web/CSS/max)
- [MDN clamp()](https://developer.mozilla.org/en-US/docs/Web/CSS/clamp)
- [CSS Tricks: min(), max(), clamp()](https://css-tricks.com/min-max-and-clamp-are-css-magic/)
