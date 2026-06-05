# User Preference Media Queries: Dark Mode, Reduced Motion, High Contrast

## The Idea

**In plain English:** Websites can ask your device "what are your preferences?" and automatically adjust their look and behavior to match — for example, switching to dark colors if you prefer a dark screen, or stopping animations if movement on screen bothers you.

**Real-world analogy:** Think of a restaurant that checks your dietary preferences before you order. When you tell the host you are vegetarian, the kitchen automatically adjusts every dish it serves you — no extra asking required.
- The host asking for preferences = CSS checking `prefers-color-scheme`, `prefers-reduced-motion`, etc.
- Your dietary preference = the system setting the user has turned on (e.g., Dark Mode or Reduce Motion)
- The kitchen adjusting the dish = the browser applying the matching CSS styles automatically

---

## Learning Objectives
- Implement dark mode with `prefers-color-scheme`
- Respect motion preferences with `prefers-reduced-motion`
- Support high contrast mode with `prefers-contrast`
- Handle other user preference queries (`prefers-reduced-transparency`, `inverted-colors`)
- Build accessible, user-respecting interfaces
- Test and debug preference-based media queries

---

## `prefers-color-scheme`: Dark Mode

### Basic Dark Mode Implementation

```css
/* Light mode (default) */
:root {
  --bg-color: white;
  --text-color: black;
  --border-color: #ddd;
}

body {
  background: var(--bg-color);
  color: var(--text-color);
}

.card {
  border: 1px solid var(--border-color);
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-color: #1a1a1a;
    --text-color: #e0e0e0;
    --border-color: #444;
  }
}
```

**How it works:**
- Browser detects OS/system-level dark mode preference
- Automatically applies dark mode styles
- No JavaScript needed

---

### Comprehensive Color Scheme

```css
:root {
  /* Light mode colors */
  --color-bg-primary: #ffffff;
  --color-bg-secondary: #f5f5f5;
  --color-text-primary: #1a1a1a;
  --color-text-secondary: #666666;
  --color-border: #e0e0e0;
  --color-accent: #0066cc;
  --color-accent-hover: #0052a3;
  --color-success: #28a745;
  --color-error: #dc3545;
  --color-warning: #ffc107;
}

@media (prefers-color-scheme: dark) {
  :root {
    /* Dark mode colors */
    --color-bg-primary: #1a1a1a;
    --color-bg-secondary: #2d2d2d;
    --color-text-primary: #e0e0e0;
    --color-text-secondary: #a0a0a0;
    --color-border: #444444;
    --color-accent: #4d9fff;
    --color-accent-hover: #66b3ff;
    --color-success: #4ade80;
    --color-error: #f87171;
    --color-warning: #fbbf24;
  }
}

/* Apply colors */
body {
  background: var(--color-bg-primary);
  color: var(--color-text-primary);
}

.card {
  background: var(--color-bg-secondary);
  border: 1px solid var(--color-border);
}

.link {
  color: var(--color-accent);
}

.link:hover {
  color: var(--color-accent-hover);
}
```

---

### Dark Mode Best Practices

#### 1. Don't use pure black

```css
/* BAD - pure black is harsh, causes eye strain */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-color: #000000;
  }
}

/* GOOD - softer dark gray */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-color: #1a1a1a; /* or #121212 */
  }
}
```

---

#### 2. Reduce contrast for text

```css
/* BAD - pure white on dark is harsh */
@media (prefers-color-scheme: dark) {
  :root {
    --text-color: #ffffff;
  }
}

/* GOOD - softer white/gray */
@media (prefers-color-scheme: dark) {
  :root {
    --text-color: #e0e0e0;
    --text-secondary: #a0a0a0;
  }
}
```

---

#### 3. Invert shadows and highlights

```css
/* Light mode */
.card {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* Dark mode - lighter shadows or use borders */
@media (prefers-color-scheme: dark) {
  .card {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.5);
    /* or */
    border: 1px solid #444;
  }
}
```

---

#### 4. Adjust images for dark mode

```css
/* Light mode - normal images */
img {
  opacity: 1;
}

/* Dark mode - slightly dim bright images */
@media (prefers-color-scheme: dark) {
  img {
    opacity: 0.9;
  }
  
  /* Invert logos/icons if needed */
  .logo {
    filter: invert(1);
  }
}
```

---

### Dark Mode with `color-scheme`

**Tell the browser to use dark UI:**

```css
/* Enable browser dark mode (scrollbars, form controls) */
:root {
  color-scheme: light dark;
}

/* Or per-element */
.dark-section {
  color-scheme: dark;
}
```

**In HTML:**
```html
<meta name="color-scheme" content="light dark">
```

**Effect:**
- Browser uses dark scrollbars
- Form controls render in dark mode
- Browser UI adapts (e.g., Safari's address bar)

---

### Manual Dark Mode Toggle (with JS)

```css
/* Light mode (default) */
:root {
  --bg-color: white;
  --text-color: black;
}

/* Dark mode via class */
:root.dark-mode {
  --bg-color: #1a1a1a;
  --text-color: #e0e0e0;
}

/* Respect system preference if no manual override */
@media (prefers-color-scheme: dark) {
  :root:not(.light-mode) {
    --bg-color: #1a1a1a;
    --text-color: #e0e0e0;
  }
}
```

**JavaScript:**
```javascript
// Toggle dark mode
function toggleDarkMode() {
  const root = document.documentElement;
  const isDark = root.classList.contains('dark-mode');
  
  if (isDark) {
    root.classList.remove('dark-mode');
    root.classList.add('light-mode');
    localStorage.setItem('theme', 'light');
  } else {
    root.classList.add('dark-mode');
    root.classList.remove('light-mode');
    localStorage.setItem('theme', 'dark');
  }
}

// Initialize from localStorage
const savedTheme = localStorage.getItem('theme');
if (savedTheme) {
  document.documentElement.classList.add(`${savedTheme}-mode`);
}
```

---

## `prefers-reduced-motion`: Respecting Motion Sensitivity

### Why Reduced Motion Matters

**Some users experience:**
- Vestibular disorders (dizziness, nausea from motion)
- Epilepsy/seizures triggered by animations
- Attention/focus issues (animations are distracting)

**Respecting this preference is an accessibility requirement.**

---

### Basic Usage

```css
/* Default: Animations enabled */
.button {
  transition: transform 0.3s ease;
}

.button:hover {
  transform: scale(1.05);
}

/* Reduced motion: Disable animations */
@media (prefers-reduced-motion: reduce) {
  .button {
    transition: none;
  }
  
  .button:hover {
    transform: none;
  }
}
```

---

### Global Reduced Motion Reset

```css
/* Disable all animations for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

**Note:** Using `0.01ms` instead of `0s` prevents breaking scripts that rely on animation events.

---

### Selective Animation Disabling

**Keep essential animations, remove decorative ones:**

```css
/* Decorative animation */
.hero {
  animation: float 3s ease-in-out infinite;
}

@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20px); }
}

/* Disable decorative animations */
@media (prefers-reduced-motion: reduce) {
  .hero {
    animation: none;
  }
}

/* Essential animation (loading spinner) - keep but simplify */
.spinner {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@media (prefers-reduced-motion: reduce) {
  .spinner {
    animation: pulse 1.5s ease-in-out infinite;
  }
  
  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.5; }
  }
}
```

---

### Smooth Scrolling

```css
/* Enable smooth scrolling by default */
html {
  scroll-behavior: smooth;
}

/* Disable for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  html {
    scroll-behavior: auto;
  }
}
```

---

### Parallax Effects

```css
/* Parallax effect */
.parallax {
  background-attachment: fixed;
  background-size: cover;
}

/* Disable parallax for reduced motion */
@media (prefers-reduced-motion: reduce) {
  .parallax {
    background-attachment: scroll;
  }
}
```

---

### Transition Best Practices

**Instead of disabling all transitions:**
```css
/* BAD - removes all visual feedback */
@media (prefers-reduced-motion: reduce) {
  * {
    transition: none !important;
  }
}
```

**Keep instant visual feedback:**
```css
/* GOOD - instant transitions instead of none */
@media (prefers-reduced-motion: reduce) {
  * {
    transition-duration: 0.01ms !important;
  }
  
  /* Or keep color/opacity changes (non-motion) */
  .button {
    transition: background-color 0.2s, color 0.2s;
  }
}
```

---

## `prefers-contrast`: High Contrast Mode

### Basic High Contrast Support

```css
/* Default contrast */
.button {
  background: #0066cc;
  color: white;
  border: 1px solid #0066cc;
}

/* High contrast - stronger borders and colors */
@media (prefers-contrast: high) {
  .button {
    background: #0052a3;
    color: white;
    border: 2px solid black;
    font-weight: 600;
  }
}

/* Low contrast - softer colors */
@media (prefers-contrast: low) {
  .button {
    background: #4d9fff;
    border: 1px solid #b3d9ff;
  }
}
```

---

### Values

| Value | Meaning | User Preference |
|-------|---------|-----------------|
| `no-preference` | Default | No preference set |
| `high` | High contrast | Needs strong contrast |
| `low` | Low contrast | Prefers softer colors |
| `more` | More contrast | Generic "more" request |
| `less` | Less contrast | Generic "less" request |

**Most common:** `high` (Windows High Contrast Mode, accessibility tools)

---

### High Contrast Patterns

#### Stronger borders

```css
.card {
  border: 1px solid #e0e0e0;
}

@media (prefers-contrast: high) {
  .card {
    border: 2px solid #000000;
  }
}
```

---

#### Increased font weight

```css
body {
  font-weight: 400;
}

@media (prefers-contrast: high) {
  body {
    font-weight: 500;
  }
  
  strong, b {
    font-weight: 700;
  }
}
```

---

#### More saturated colors

```css
:root {
  --color-accent: #0066cc;
}

@media (prefers-contrast: high) {
  :root {
    --color-accent: #0052a3; /* Darker, more saturated */
  }
}
```

---

#### Remove subtle effects

```css
.card {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

@media (prefers-contrast: high) {
  .card {
    box-shadow: none;
    border: 2px solid black;
  }
}
```

---

## `prefers-reduced-transparency`: Reduced Transparency

```css
/* Default: Translucent overlay */
.modal-backdrop {
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(10px);
}

/* Reduced transparency */
@media (prefers-reduced-transparency: reduce) {
  .modal-backdrop {
    background: rgba(0, 0, 0, 0.9);
    backdrop-filter: none;
  }
}
```

**Use case:** Some users find transparency disorienting or hard to read through.

---

## `inverted-colors`: Inverted Color Mode

```css
/* Detect if OS colors are inverted */
@media (inverted-colors: inverted) {
  /* Invert images back to normal */
  img, video {
    filter: invert(1);
  }
  
  /* Skip inverting already-inverted content */
  .dark-logo {
    filter: none;
  }
}
```

**Use case:** macOS/iOS "Invert Colors" accessibility feature.

---

## `prefers-color-scheme` + `prefers-reduced-motion` Combined

```css
/* Light mode with motion */
:root {
  --bg-color: white;
  --text-color: black;
}

.card {
  transition: transform 0.3s;
}

.card:hover {
  transform: translateY(-5px);
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-color: #1a1a1a;
    --text-color: #e0e0e0;
  }
}

/* Reduced motion (light mode) */
@media (prefers-reduced-motion: reduce) {
  .card {
    transition: none;
  }
  
  .card:hover {
    transform: none;
  }
}

/* Dark mode + reduced motion */
@media (prefers-color-scheme: dark) and (prefers-reduced-motion: reduce) {
  /* Specific overrides if needed */
}
```

---

## Testing User Preferences

### Chrome DevTools

1. Open DevTools (F12)
2. Open Command Menu (Cmd+Shift+P / Ctrl+Shift+P)
3. Type "Rendering"
4. Check "Emulate CSS media feature prefers-color-scheme"
5. Select `prefers-color-scheme: dark` or `prefers-color-scheme: light`

**Also available:**
- `prefers-reduced-motion`
- `prefers-contrast`
- `prefers-reduced-transparency`

---

### Firefox DevTools

1. Open DevTools (F12)
2. Click "Settings" (gear icon)
3. Scroll to "Inspector"
4. Check "Simulate prefers-color-scheme"
5. Select "dark" or "light"

**For reduced motion:**
1. Type `about:config` in address bar
2. Search `ui.prefersReducedMotion`
3. Set to `1` (reduced motion) or `0` (no preference)

---

### Safari DevTools

1. Open Web Inspector (Cmd+Option+I)
2. Go to "Elements" tab
3. Click "Style" sidebar
4. Find media query section
5. Toggle dark mode / reduced motion

---

### System-Level Testing

**macOS:**
- Dark mode: System Preferences → General → Appearance → Dark
- Reduced motion: System Preferences → Accessibility → Display → Reduce motion

**Windows:**
- Dark mode: Settings → Personalization → Colors → Choose your mode → Dark
- High contrast: Settings → Ease of Access → High contrast
- Reduced motion: Settings → Ease of Access → Display → Show animations in Windows

**iOS:**
- Dark mode: Settings → Display & Brightness → Dark
- Reduced motion: Settings → Accessibility → Motion → Reduce Motion

**Android:**
- Dark mode: Settings → Display → Dark theme
- Reduced motion: Settings → Accessibility → Remove animations

---

## JavaScript Detection

### Detect Dark Mode

```javascript
// Check current preference
const isDarkMode = window.matchMedia('(prefers-color-scheme: dark)').matches;

if (isDarkMode) {
  console.log('User prefers dark mode');
}

// Listen for changes
const darkModeQuery = window.matchMedia('(prefers-color-scheme: dark)');

darkModeQuery.addEventListener('change', (e) => {
  if (e.matches) {
    console.log('Switched to dark mode');
  } else {
    console.log('Switched to light mode');
  }
});
```

---

### Detect Reduced Motion

```javascript
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (prefersReducedMotion) {
  // Disable animations
  document.body.classList.add('reduced-motion');
}
```

---

### Detect High Contrast

```javascript
const prefersHighContrast = window.matchMedia('(prefers-contrast: high)').matches;

if (prefersHighContrast) {
  console.log('User prefers high contrast');
}
```

---

## Practical Examples

### Example 1: Accessible Card Component

```css
/* Base styles */
.card {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 1.5rem;
  transition: transform 0.3s, box-shadow 0.3s;
}

.card:hover {
  transform: translateY(-5px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  .card {
    background: #2d2d2d;
    border-color: #444;
  }
  
  .card:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.5);
  }
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  .card {
    transition: none;
  }
  
  .card:hover {
    transform: none;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
}

/* High contrast */
@media (prefers-contrast: high) {
  .card {
    border-width: 2px;
    border-color: #000;
  }
}

/* Dark mode + high contrast */
@media (prefers-color-scheme: dark) and (prefers-contrast: high) {
  .card {
    border-color: #fff;
  }
}
```

---

### Example 2: Accessible Navigation

```css
.nav {
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s;
}

.nav-link {
  color: #0066cc;
  transition: color 0.2s;
}

.nav-link:hover {
  color: #0052a3;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  .nav {
    background: #1a1a1a;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
  }
  
  .nav-link {
    color: #4d9fff;
  }
  
  .nav-link:hover {
    color: #66b3ff;
  }
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  .nav {
    transition: none;
  }
  
  .nav-link {
    transition: none;
  }
}

/* High contrast */
@media (prefers-contrast: high) {
  .nav-link {
    font-weight: 600;
    text-decoration: underline;
  }
}
```

---

### Example 3: Loading Spinner

```css
.spinner {
  border: 4px solid rgba(0, 0, 0, 0.1);
  border-top-color: #0066cc;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Reduced motion - use pulse instead */
@media (prefers-reduced-motion: reduce) {
  .spinner {
    animation: pulse 1.5s ease-in-out infinite;
    border-top-color: #0066cc;
  }
  
  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.5; }
  }
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  .spinner {
    border-color: rgba(255, 255, 255, 0.1);
    border-top-color: #4d9fff;
  }
}
```

---

## Common Mistakes

### ❌ Mistake 1: Not testing with real user preferences

```css
/* Just testing in DevTools isn't enough */
@media (prefers-color-scheme: dark) {
  /* Might look good in DevTools... */
}
```

**Fix:** Test with system-level dark mode on real devices.

---

### ❌ Mistake 2: Removing all animations for reduced motion

```css
/* BAD - removes visual feedback */
@media (prefers-reduced-motion: reduce) {
  * {
    animation: none !important;
    transition: none !important;
  }
}
```

**Fix:** Use instant transitions instead:
```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

### ❌ Mistake 3: Forgetting `color-scheme` meta tag

```html
<!-- BAD - no color-scheme -->
<head>
  <style>
    @media (prefers-color-scheme: dark) {
      body { background: black; }
    }
  </style>
</head>
```

**Problem:** Browser UI (scrollbars, form controls) stays light mode.

**Fix:**
```html
<head>
  <meta name="color-scheme" content="light dark">
  <style>
    @media (prefers-color-scheme: dark) {
      body { background: black; }
    }
  </style>
</head>
```

---

### ❌ Mistake 4: Pure black/white in dark mode

```css
/* BAD - harsh contrast */
@media (prefers-color-scheme: dark) {
  body {
    background: #000000;
    color: #ffffff;
  }
}

/* GOOD - softer contrast */
@media (prefers-color-scheme: dark) {
  body {
    background: #1a1a1a;
    color: #e0e0e0;
  }
}
```

---

## Browser Support (2026)

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| `prefers-color-scheme` | 76+ | 67+ | 12.1+ | 79+ |
| `prefers-reduced-motion` | 74+ | 63+ | 10.1+ | 79+ |
| `prefers-contrast` | 96+ | 101+ | 14.1+ | 96+ |
| `prefers-reduced-transparency` | 118+ | ❌ | 15.4+ | 118+ |
| `inverted-colors` | 93+ | ❌ | 9.1+ | 93+ |

**Verdict:** All major preference queries are widely supported in 2026.

---

## Key Takeaways

✅ **Respect `prefers-color-scheme`** - implement dark mode with CSS variables

✅ **Honor `prefers-reduced-motion`** - disable decorative animations, keep essential ones

✅ **Support `prefers-contrast`** - strengthen borders and colors for high contrast

✅ **Use `color-scheme` meta tag** - make browser UI adapt to dark mode

✅ **Don't use pure black/white** - softer colors reduce eye strain

✅ **Test with real user preferences** - DevTools is good, but not enough

✅ **Keep visual feedback in reduced motion** - use instant transitions, not none

✅ **Combine preferences** - dark mode + reduced motion should work together

---

## Resources

- [MDN: prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)
- [MDN: prefers-contrast](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-contrast)
- [Web.dev: prefers-color-scheme](https://web.dev/prefers-color-scheme/)
- [CSS Tricks: Dark Mode](https://css-tricks.com/a-complete-guide-to-dark-mode-on-the-web/)
- [Web.dev: prefers-reduced-motion](https://web.dev/prefers-reduced-motion/)
