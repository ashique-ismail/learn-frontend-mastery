# Media Queries: Syntax, Features, Range Syntax, Breakpoints

## Learning Objectives
- Master media query syntax and features
- Use modern range syntax for cleaner queries
- Understand media types and conditions
- Choose effective breakpoints for responsive design
- Combine multiple conditions with logical operators
- Debug and test media queries effectively

---

## Media Query Syntax

### Basic Structure

```css
@media [media-type] [logical-operator] ([media-feature]: [value]) {
  /* CSS rules */
}
```

**Example:**
```css
@media screen and (min-width: 768px) {
  .container {
    max-width: 1200px;
  }
}
```

**Parts:**
- `@media` - Media query declaration
- `screen` - Media type (optional)
- `and` - Logical operator
- `(min-width: 768px)` - Media feature with value
- `{ }` - CSS rules to apply when condition is true

---

## Media Types

### Common Media Types

```css
/* All devices (default) */
@media all {
  body { font-size: 16px; }
}

/* Screen devices (desktops, tablets, phones) */
@media screen {
  body { background: white; }
}

/* Print preview and printed pages */
@media print {
  body { 
    background: white;
    color: black;
  }
  
  nav, footer {
    display: none;
  }
}

/* Speech synthesizers (screen readers) */
@media speech {
  body {
    voice-rate: medium;
  }
}
```

**Most common:** `screen` and `print`. Others (like `tty`, `tv`, `projection`) are deprecated.

---

### Omitting Media Type

When targeting all devices, you can omit the media type:

```css
/* These are equivalent */
@media all and (min-width: 768px) { }

@media (min-width: 768px) { }
```

**Best practice:** Omit `all` for brevity.

---

## Media Features

### Width and Height

#### Viewport Width
```css
/* Exact width (rarely used) */
@media (width: 768px) {
  /* Only at exactly 768px */
}

/* Minimum width (mobile-first) */
@media (min-width: 768px) {
  /* 768px and above */
}

/* Maximum width (desktop-first) */
@media (max-width: 767px) {
  /* 767px and below */
}

/* Width range (old syntax) */
@media (min-width: 768px) and (max-width: 1023px) {
  /* Between 768px and 1023px */
}
```

---

#### Viewport Height

```css
/* Minimum height */
@media (min-height: 600px) {
  .hero {
    height: 100vh;
  }
}

/* Maximum height (e.g., for mobile landscape) */
@media (max-height: 500px) {
  .nav {
    height: auto;
  }
}
```

**Use case:** Adjusting layouts for short viewports (landscape phones, small laptop screens).

---

### Orientation

```css
/* Portrait (height > width) */
@media (orientation: portrait) {
  .image {
    width: 100%;
    height: auto;
  }
}

/* Landscape (width > height) */
@media (orientation: landscape) {
  .image {
    width: 50%;
    float: left;
  }
}
```

**Common use case:** Different layouts for phone rotation.

```css
/* Mobile landscape - short and wide */
@media (max-height: 500px) and (orientation: landscape) {
  .nav {
    flex-direction: row;
    height: 50px;
  }
}
```

---

### Resolution and Pixel Density

#### Device Pixel Ratio (dppx)

```css
/* Standard resolution (1x) */
@media (resolution: 1dppx) {
  .logo {
    background-image: url('logo.png');
  }
}

/* Retina / high-DPI (2x) */
@media (min-resolution: 2dppx) {
  .logo {
    background-image: url('logo@2x.png');
  }
}

/* Extra high-DPI (3x) */
@media (min-resolution: 3dppx) {
  .logo {
    background-image: url('logo@3x.png');
  }
}
```

---

#### DPI (Dots Per Inch)

```css
/* 96dpi = 1dppx (standard) */
@media (min-resolution: 192dpi) {
  /* 2x resolution (192dpi = 2dppx) */
  .icon {
    background-image: url('icon@2x.png');
  }
}
```

**Conversion:** `1dppx = 96dpi`

---

#### Vendor Prefixes (Legacy)

```css
/* Old WebKit syntax (iOS, Android) */
@media (-webkit-min-device-pixel-ratio: 2) {
  .logo {
    background-image: url('logo@2x.png');
  }
}

/* Modern standard (preferred) */
@media (min-resolution: 2dppx) {
  .logo {
    background-image: url('logo@2x.png');
  }
}
```

**Best practice:** Use standard `dppx`, add `-webkit-` prefix for old iOS.

---

### Aspect Ratio

```css
/* Widescreen (16:9) */
@media (aspect-ratio: 16/9) {
  .video {
    width: 100%;
  }
}

/* Minimum aspect ratio */
@media (min-aspect-ratio: 16/9) {
  /* Widescreen or wider */
}

/* Maximum aspect ratio */
@media (max-aspect-ratio: 4/3) {
  /* Square-ish or narrower */
}
```

**Use case:** Optimizing video player layouts for different screen shapes.

---

### Hover Capability

```css
/* Device has hover capability (mouse) */
@media (hover: hover) {
  .button:hover {
    background: darkblue;
  }
}

/* Device cannot hover (touch) */
@media (hover: none) {
  .button {
    /* No hover state needed */
  }
}
```

**Use case:** Avoid hover states on touch devices, where hover doesn't work.

---

### Pointer Accuracy

```css
/* Fine pointer (mouse, stylus) */
@media (pointer: fine) {
  .target {
    width: 20px;
    height: 20px;
  }
}

/* Coarse pointer (touch) */
@media (pointer: coarse) {
  .target {
    width: 44px;  /* Minimum touch target size */
    height: 44px;
  }
}

/* No pointer */
@media (pointer: none) {
  /* Keyboard-only navigation */
}
```

**Best practice:** Use `pointer: coarse` to increase touch target sizes.

---

### Combining Hover and Pointer

```css
/* Desktop with mouse */
@media (hover: hover) and (pointer: fine) {
  .element:hover {
    transform: scale(1.05);
  }
}

/* Touch devices */
@media (hover: none) and (pointer: coarse) {
  .element {
    /* Larger touch targets, no hover effects */
    min-height: 44px;
  }
}
```

---

## Logical Operators

### `and` - All conditions must be true

```css
/* Both conditions must match */
@media (min-width: 768px) and (max-width: 1023px) {
  /* Tablet range only */
}

/* Multiple conditions */
@media screen and (min-width: 768px) and (orientation: landscape) {
  /* Screen devices, tablet+, landscape only */
}
```

---

### `,` (comma) - OR operator

```css
/* Either condition can match */
@media (max-width: 767px), (orientation: portrait) {
  /* Mobile OR any portrait orientation */
}

/* Multiple OR conditions */
@media screen and (max-width: 767px), 
       print,
       (orientation: portrait) {
  /* Small screens OR print OR portrait */
}
```

**Note:** Comma acts as OR - if any condition is true, styles apply.

---

### `not` - Negation

```css
/* Not a screen device */
@media not screen {
  /* Print, speech, etc. */
}

/* Not mobile */
@media not all and (max-width: 767px) {
  /* Desktop and tablet */
}
```

**Important:** `not` applies to the **entire** media query, not individual features.

---

### `only` - Hide from old browsers

```css
/* Old browsers that don't support media queries ignore this */
@media only screen and (min-width: 768px) {
  /* Modern browsers only */
}
```

**Use case:** Prevents old browsers from applying styles incorrectly. Rarely needed today.

---

## Modern Range Syntax (Level 4)

### Comparison Operators

**Old syntax (verbose):**
```css
@media (min-width: 768px) and (max-width: 1023px) {
  /* Tablet range */
}
```

**New syntax (cleaner):**
```css
@media (768px <= width <= 1023px) {
  /* Tablet range */
}

/* Or using < and > */
@media (768px <= width < 1024px) {
  /* Same range, exclusive upper bound */
}
```

---

### Supported Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `<` | Less than | `(width < 768px)` |
| `<=` | Less than or equal | `(width <= 767px)` |
| `>` | Greater than | `(width > 1023px)` |
| `>=` | Greater than or equal | `(width >= 1024px)` |
| `=` | Equal to | `(width = 768px)` |

---

### Range Syntax Examples

#### Minimum width (mobile-first)
```css
/* Old */
@media (min-width: 768px) { }

/* New */
@media (width >= 768px) { }
```

---

#### Maximum width (desktop-first)
```css
/* Old */
@media (max-width: 767px) { }

/* New */
@media (width <= 767px) { }
```

---

#### Width range
```css
/* Old */
@media (min-width: 768px) and (max-width: 1023px) { }

/* New */
@media (768px <= width <= 1023px) { }
```

---

#### Excluding a range
```css
/* Old */
@media (max-width: 767px), (min-width: 1024px) { }

/* New */
@media (width < 768px) or (width >= 1024px) { }
```

---

#### Height range
```css
/* Old */
@media (min-height: 600px) and (max-height: 900px) { }

/* New */
@media (600px <= height <= 900px) { }
```

---

### Browser Support (2026)

**Modern range syntax support:**
- Chrome 104+ (Aug 2022)
- Firefox 63+ (Oct 2018)
- Safari 16.4+ (Mar 2023)
- Edge 104+ (Aug 2022)

**Verdict:** Safe to use in 2026, but old syntax still works everywhere.

---

## Choosing Breakpoints

### Content-Based Breakpoints (Best Practice)

**Don't use device-specific breakpoints:**
```css
/* BAD - tied to specific devices */
@media (min-width: 375px) { } /* iPhone */
@media (min-width: 768px) { } /* iPad */
@media (min-width: 1024px) { } /* Desktop */
```

**Use content-based breakpoints:**
```css
/* GOOD - based on when layout breaks */
@media (min-width: 600px) {
  /* When text becomes too wide to read comfortably */
}

@media (min-width: 900px) {
  /* When there's room for sidebar */
}
```

**Principle:** Add breakpoints when your **content** needs them, not at arbitrary device widths.

---

### Common Breakpoint Systems

#### Minimal (3 breakpoints)

```css
/* Small (mobile) - default */

/* Medium (tablet) */
@media (min-width: 768px) { }

/* Large (desktop) */
@media (min-width: 1024px) { }
```

**Best for:** Simple sites, blogs, marketing pages.

---

#### Standard (4 breakpoints)

```css
/* Extra small (mobile) - default */

/* Small (large mobile / small tablet) */
@media (min-width: 576px) { }

/* Medium (tablet) */
@media (min-width: 768px) { }

/* Large (desktop) */
@media (min-width: 1024px) { }

/* Extra large (large desktop) - optional */
@media (min-width: 1440px) { }
```

**Best for:** Most websites, web applications.

---

#### Bootstrap-style (5 breakpoints)

```css
/* xs (extra small) - default */

/* sm (small) */
@media (min-width: 576px) { }

/* md (medium) */
@media (min-width: 768px) { }

/* lg (large) */
@media (min-width: 992px) { }

/* xl (extra large) */
@media (min-width: 1200px) { }

/* xxl (extra extra large) */
@media (min-width: 1400px) { }
```

**Best for:** Complex applications, design systems.

---

### Custom Breakpoints

**CSS Custom Properties:**
```css
:root {
  --breakpoint-sm: 576px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
}

/* Note: Can't use CSS variables in media queries (yet) */
@media (min-width: 768px) { } /* Must use literal value */
```

**Sass/SCSS Variables:**
```scss
$breakpoint-sm: 576px;
$breakpoint-md: 768px;
$breakpoint-lg: 1024px;

@media (min-width: $breakpoint-md) {
  .container {
    padding: 2rem;
  }
}
```

---

## Practical Media Query Patterns

### Pattern 1: Mobile-First Container

```css
/* Mobile: Full width with padding */
.container {
  width: 100%;
  padding: 0 1rem;
}

/* Tablet: More padding */
@media (min-width: 768px) {
  .container {
    padding: 0 2rem;
  }
}

/* Desktop: Constrained width, centered */
@media (min-width: 1024px) {
  .container {
    max-width: 1200px;
    margin: 0 auto;
  }
}
```

---

### Pattern 2: Responsive Typography

```css
/* Mobile: Base size */
body {
  font-size: 16px;
  line-height: 1.5;
}

h1 {
  font-size: 2rem;
}

/* Tablet: Slightly larger */
@media (min-width: 768px) {
  body {
    font-size: 17px;
    line-height: 1.6;
  }
  
  h1 {
    font-size: 2.5rem;
  }
}

/* Desktop: Maximum readability */
@media (min-width: 1024px) {
  body {
    font-size: 18px;
    line-height: 1.7;
  }
  
  h1 {
    font-size: 3rem;
  }
}
```

---

### Pattern 3: Responsive Grid

```css
/* Mobile: Single column */
.grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
}

/* Tablet: 2 columns */
@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
    gap: 1.5rem;
  }
}

/* Desktop: 3 columns */
@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
    gap: 2rem;
  }
}

/* Large desktop: 4 columns */
@media (min-width: 1440px) {
  .grid {
    grid-template-columns: repeat(4, 1fr);
  }
}
```

---

### Pattern 4: Navigation Transform

```css
/* Mobile: Hamburger menu */
.nav {
  position: fixed;
  top: 0;
  left: -250px;
  width: 250px;
  height: 100vh;
  background: #333;
  transition: left 0.3s;
}

.nav.open {
  left: 0;
}

.nav-toggle {
  display: block;
}

/* Desktop: Horizontal nav */
@media (min-width: 1024px) {
  .nav {
    position: static;
    width: auto;
    height: auto;
    background: transparent;
    display: flex;
    gap: 2rem;
  }
  
  .nav-toggle {
    display: none;
  }
}
```

---

### Pattern 5: Sidebar Layout

```css
/* Mobile: Stacked */
.layout {
  display: block;
}

.sidebar {
  width: 100%;
  margin-bottom: 2rem;
}

.main {
  width: 100%;
}

/* Desktop: Side-by-side */
@media (min-width: 1024px) {
  .layout {
    display: grid;
    grid-template-columns: 250px 1fr;
    gap: 2rem;
  }
  
  .sidebar {
    margin-bottom: 0;
  }
}
```

---

### Pattern 6: Print Styles

```css
/* Screen styles (default) */
.page {
  background: #f5f5f5;
  padding: 2rem;
}

nav, .sidebar, .comments {
  display: block;
}

/* Print styles */
@media print {
  /* Hide non-essential elements */
  nav, .sidebar, .comments, .no-print {
    display: none;
  }
  
  /* Optimize for print */
  body {
    background: white;
    color: black;
    font-size: 12pt;
  }
  
  .page {
    padding: 0;
  }
  
  /* Show link URLs */
  a[href]:after {
    content: " (" attr(href) ")";
  }
  
  /* Prevent page breaks inside elements */
  h1, h2, h3, h4, h5, h6 {
    page-break-after: avoid;
  }
  
  img {
    page-break-inside: avoid;
  }
}
```

---

## Combining Multiple Features

### Desktop with Mouse

```css
@media (min-width: 1024px) and (hover: hover) and (pointer: fine) {
  .card {
    transition: transform 0.3s;
  }
  
  .card:hover {
    transform: translateY(-5px);
  }
}
```

---

### Mobile Landscape

```css
@media (max-height: 500px) and (orientation: landscape) {
  /* Short, wide viewport */
  .hero {
    height: auto;
    min-height: 400px;
  }
  
  .nav {
    height: 50px;
  }
}
```

---

### High-DPI Tablet

```css
@media (min-width: 768px) and (min-resolution: 2dppx) {
  .logo {
    background-image: url('logo@2x.png');
    background-size: contain;
  }
}
```

---

### Touch Device with Large Screen

```css
@media (min-width: 1024px) and (pointer: coarse) {
  /* Large tablet or touch laptop */
  .button {
    min-width: 44px;
    min-height: 44px;
    padding: 0.75rem 1.5rem;
  }
}
```

---

## Debugging Media Queries

### Chrome DevTools

**Method 1: Responsive Mode**
1. Open DevTools (F12)
2. Toggle device toolbar (Cmd+Shift+M / Ctrl+Shift+M)
3. Resize viewport or choose preset devices
4. Check which media queries are active in Styles panel (shown in green)

**Method 2: Media Queries Inspector**
1. Open DevTools
2. Click "..." menu → More tools → Media queries
3. Visual bar shows all breakpoints
4. Click breakpoint to snap to that width

---

### Firefox DevTools

**Responsive Design Mode:**
1. Open DevTools (F12)
2. Click responsive design mode icon
3. View media query information in top bar
4. See which queries are currently active

---

### JavaScript Detection

```javascript
// Check if media query matches
const isDesktop = window.matchMedia('(min-width: 1024px)').matches;

if (isDesktop) {
  console.log('Desktop view');
}

// Listen for changes
const mediaQuery = window.matchMedia('(min-width: 1024px)');

mediaQuery.addEventListener('change', (e) => {
  if (e.matches) {
    console.log('Now desktop');
  } else {
    console.log('Now mobile/tablet');
  }
});
```

---

### Common Debugging Issues

#### Issue 1: Media query not applying

**Check:**
- Viewport meta tag in HTML: `<meta name="viewport" content="width=device-width, initial-scale=1">`
- Syntax errors in media query
- Specificity conflicts (later rules override)
- Typos in feature names

---

#### Issue 2: Wrong breakpoint triggering

**Check:**
- Using `max-width` vs `min-width` correctly
- Overlapping breakpoints
- Browser zoom level (affects `px` units)
- DevTools device mode vs real device

---

## Common Mistakes

### ❌ Mistake 1: No viewport meta tag

```html
<!-- BAD - no viewport meta tag -->
<head>
  <title>My Site</title>
</head>
```

**Problem:** Mobile browsers assume desktop width (typically 980px), media queries won't work.

**Fix:**
```html
<!-- GOOD -->
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My Site</title>
</head>
```

---

### ❌ Mistake 2: Using device-specific breakpoints

```css
/* BAD - tied to specific devices */
@media (min-width: 375px) { } /* iPhone */
@media (min-width: 414px) { } /* iPhone Plus */
@media (min-width: 768px) { } /* iPad */
```

**Problem:** Devices come in all sizes. These breakpoints are arbitrary.

**Fix:**
```css
/* GOOD - based on content needs */
@media (min-width: 600px) { }  /* When content needs more space */
@media (min-width: 1024px) { } /* When desktop layout fits */
```

---

### ❌ Mistake 3: Overlapping ranges

```css
/* BAD - 768px matches both rules */
@media (min-width: 768px) {
  .element { font-size: 18px; }
}

@media (min-width: 768px) and (max-width: 1024px) {
  .element { font-size: 16px; } /* Conflicts! */
}
```

**Fix:**
```css
/* GOOD - non-overlapping */
@media (min-width: 768px) {
  .element { font-size: 18px; }
}

@media (min-width: 1025px) {
  .element { font-size: 20px; }
}
```

---

### ❌ Mistake 4: Not testing real devices

**Problem:** DevTools approximates devices but doesn't catch all issues:
- Touch target sizes
- Actual pixel density
- Device-specific bugs
- Slow connections

**Fix:** Test on real devices or use browser testing services.

---

## Performance Tips

### Minimize media query duplication

**Bad:**
```css
@media (min-width: 768px) {
  .card { padding: 2rem; }
}

@media (min-width: 768px) {
  .button { font-size: 1.1rem; }
}

@media (min-width: 768px) {
  .nav { display: flex; }
}
```

**Good:**
```css
@media (min-width: 768px) {
  .card { padding: 2rem; }
  .button { font-size: 1.1rem; }
  .nav { display: flex; }
}
```

---

### Use container queries for components (modern)

**Instead of:**
```css
@media (min-width: 768px) {
  .card { display: flex; }
}
```

**Use (when supported):**
```css
@container (min-width: 400px) {
  .card { display: flex; }
}
```

**Benefit:** Component adapts to **container** width, not viewport width.

---

## Key Takeaways

✅ **Omit media type for brevity** - `@media (min-width: 768px)` is cleaner

✅ **Use modern range syntax** - `(width >= 768px)` is more readable than `(min-width: 768px)`

✅ **Choose 3-4 content-based breakpoints** - don't target specific devices

✅ **Combine features with `and`** - e.g., `(min-width: 1024px) and (hover: hover)`

✅ **Use `,` for OR logic** - `(max-width: 767px), (orientation: portrait)`

✅ **Always include viewport meta tag** - `<meta name="viewport" content="width=device-width, initial-scale=1">`

✅ **Test on real devices** - DevTools is good, but not perfect

✅ **Use `hover` and `pointer` queries** - adapt to input method, not screen size

---

## Resources

- [MDN: Using Media Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries/Using_media_queries)
- [MDN: Media Query Syntax](https://developer.mozilla.org/en-US/docs/Web/CSS/@media)
- [W3C: Media Queries Level 4](https://www.w3.org/TR/mediaqueries-4/)
- [CSS Tricks: Complete Guide to Media Queries](https://css-tricks.com/a-complete-guide-to-css-media-queries/)
- [Web.dev: Responsive Design](https://web.dev/responsive-web-design-basics/)
