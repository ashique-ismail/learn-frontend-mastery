# Dark Mode Implementation

## The Idea

**In plain English:** Dark mode lets a website switch its color scheme from bright (light backgrounds, dark text) to dim (dark backgrounds, light text) based on what the user prefers. It works by storing a set of named color "slots" that get swapped out when the theme changes, rather than repainting every element by hand.

**Real-world analogy:** Imagine a stage theater that uses colored gel filters on its spotlights. The lighting director has a switchboard with two preset scenes — "matinee" (bright, white lights) and "evening" (warm, dark-amber lights). Switching scenes instantly changes the mood of every spotlight at once, without the crew touching individual bulbs.

- The lighting switchboard preset = the `data-theme` attribute on `<html>`
- Each spotlight's gel filter = a CSS custom property (e.g., `--bg-primary`)
- The lighting crew's rule sheet = the CSS block that maps each token to a specific color per theme
- The audience member requesting a scene change = the user clicking the dark/light toggle

---

## Overview

Dark mode implementation involves more than a single CSS media query. A production-quality implementation must handle: CSS custom property token architecture, user preference persistence, OS-level preference detection, avoiding the "flash of wrong theme" on page load, syncing across tabs, and accessible contrast ratios in both themes.

---

## Layer 1: CSS Custom Properties Token System

Architect your colors as semantic tokens backed by custom properties. This is what you switch — not class names on individual components.

```css
/* Design tokens — primitives */
:root {
  --color-gray-0:   #ffffff;
  --color-gray-100: #f3f4f6;
  --color-gray-200: #e5e7eb;
  --color-gray-700: #374151;
  --color-gray-800: #1f2937;
  --color-gray-900: #111827;
  --color-gray-950: #030712;

  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;
}

/* Semantic tokens — light theme (default) */
:root {
  --bg-primary:    var(--color-gray-0);
  --bg-secondary:  var(--color-gray-100);
  --bg-elevated:   var(--color-gray-0);

  --text-primary:   var(--color-gray-900);
  --text-secondary: var(--color-gray-700);
  --text-disabled:  var(--color-gray-200);

  --border-default: var(--color-gray-200);

  --action-primary:       var(--color-blue-500);
  --action-primary-hover: var(--color-blue-600);
}

/* Dark theme — override semantic tokens */
:root[data-theme="dark"],
.dark {
  --bg-primary:    var(--color-gray-950);
  --bg-secondary:  var(--color-gray-900);
  --bg-elevated:   var(--color-gray-800);

  --text-primary:   var(--color-gray-0);
  --text-secondary: var(--color-gray-200);
  --text-disabled:  var(--color-gray-700);

  --border-default: var(--color-gray-700);
}

/* Components use semantic tokens — never primitives */
.card {
  background: var(--bg-elevated);
  color: var(--text-primary);
  border: 1px solid var(--border-default);
}
```

---

## Layer 2: OS Preference Detection

```css
/* Respond to OS dark mode automatically (no JS needed for CSS-only approach) */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-primary:  var(--color-gray-950);
    --text-primary: var(--color-gray-0);
    /* ... all overrides ... */
  }
}
```

---

## Layer 3: User Preference Override (Three-State Toggle)

Users may want to override the OS setting. You need three states: **system**, **light**, **dark**.

```typescript
type Theme = 'system' | 'light' | 'dark';

const STORAGE_KEY = 'color-theme';

function getEffectiveTheme(preference: Theme): 'light' | 'dark' {
  if (preference === 'system') {
    return window.matchMedia('(prefers-color-scheme: dark)').matches
      ? 'dark'
      : 'light';
  }
  return preference;
}

function applyTheme(preference: Theme): void {
  const effective = getEffectiveTheme(preference);
  document.documentElement.setAttribute('data-theme', effective);
  // Optionally set meta theme-color for browser chrome
  document.querySelector('meta[name="theme-color"]')
    ?.setAttribute('content', effective === 'dark' ? '#030712' : '#ffffff');
}

function savePreference(preference: Theme): void {
  localStorage.setItem(STORAGE_KEY, preference);
  applyTheme(preference);
}

function loadPreference(): Theme {
  return (localStorage.getItem(STORAGE_KEY) as Theme) ?? 'system';
}

// Listen for OS-level changes (when preference is 'system')
window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
  if (loadPreference() === 'system') applyTheme('system');
});
```

---

## Layer 4: Avoiding Flash of Wrong Theme (FOUT)

This is the hardest part. If your theme-applying JS runs after the browser paints, users see the wrong theme for a moment.

**The only reliable fix: run a blocking inline script in `<head>` before any CSS is parsed.**

```html
<!DOCTYPE html>
<html>
<head>
  <!-- This MUST be inline — no defer/async, no external file -->
  <script>
    (function() {
      const stored = localStorage.getItem('color-theme');
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      const effective = stored === 'dark' || (!stored && prefersDark) ? 'dark' : 'light';
      document.documentElement.setAttribute('data-theme', effective);
    })();
  </script>
  <link rel="stylesheet" href="/styles.css" />
</head>
```

The script runs synchronously before the browser calculates any styles, so the initial paint uses the correct theme.

**Next.js (App Router):**

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <head>
        <script
          dangerouslySetInnerHTML={{
            __html: `
              (function() {
                var stored = localStorage.getItem('color-theme');
                var prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
                var theme = stored || (prefersDark ? 'dark' : 'light');
                document.documentElement.setAttribute('data-theme', theme);
              })()
            `,
          }}
        />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

`suppressHydrationWarning` is needed because the server renders without knowing the user's preference, so `data-theme` will differ between server and client — React would normally warn about this hydration mismatch.

---

## Layer 5: Syncing Across Tabs

Use the `storage` event to keep all tabs in sync:

```typescript
window.addEventListener('storage', (event) => {
  if (event.key === STORAGE_KEY && event.newValue) {
    applyTheme(event.newValue as Theme);
    // Update UI toggle to reflect new state
    themeToggle.value = event.newValue;
  }
});
```

---

## React Implementation

```tsx
import { createContext, useContext, useEffect, useState } from 'react';

type Theme = 'system' | 'light' | 'dark';

interface ThemeContext {
  theme: Theme;
  effectiveTheme: 'light' | 'dark';
  setTheme: (t: Theme) => void;
}

const ThemeCtx = createContext<ThemeContext | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setThemeState] = useState<Theme>(() => {
    if (typeof window === 'undefined') return 'system';
    return (localStorage.getItem('color-theme') as Theme) ?? 'system';
  });

  const [systemDark, setSystemDark] = useState(() =>
    typeof window !== 'undefined'
      ? window.matchMedia('(prefers-color-scheme: dark)').matches
      : false
  );

  useEffect(() => {
    const mq = window.matchMedia('(prefers-color-scheme: dark)');
    const handler = (e: MediaQueryListEvent) => setSystemDark(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  const effectiveTheme: 'light' | 'dark' =
    theme === 'system' ? (systemDark ? 'dark' : 'light') : theme;

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', effectiveTheme);
  }, [effectiveTheme]);

  function setTheme(t: Theme) {
    localStorage.setItem('color-theme', t);
    setThemeState(t);
  }

  return (
    <ThemeCtx.Provider value={{ theme, effectiveTheme, setTheme }}>
      {children}
    </ThemeCtx.Provider>
  );
}

export function useTheme() {
  const ctx = useContext(ThemeCtx);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}

// Toggle component
export function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  return (
    <select value={theme} onChange={(e) => setTheme(e.target.value as Theme)}>
      <option value="system">System</option>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>
  );
}
```

---

## Angular Implementation

```typescript
import { Injectable, signal, effect, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

type Theme = 'system' | 'light' | 'dark';

@Injectable({ providedIn: 'root' })
export class ThemeService {
  private platformId = inject(PLATFORM_ID);
  private isBrowser  = isPlatformBrowser(this.platformId);

  theme = signal<Theme>(this.loadSaved());
  effectiveTheme = signal<'light' | 'dark'>('light');

  constructor() {
    if (!this.isBrowser) return;

    const mq = window.matchMedia('(prefers-color-scheme: dark)');
    mq.addEventListener('change', () => this.updateEffective());

    // Update document and effective signal whenever theme changes
    effect(() => {
      const t = this.theme();
      this.updateEffective();
      localStorage.setItem('color-theme', t);
    });
  }

  setTheme(t: Theme): void {
    this.theme.set(t);
  }

  private loadSaved(): Theme {
    if (!this.isBrowser) return 'system';
    return (localStorage.getItem('color-theme') as Theme) ?? 'system';
  }

  private updateEffective(): void {
    const pref = this.theme();
    const sysDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    const effective = pref === 'system' ? (sysDark ? 'dark' : 'light') : pref;
    this.effectiveTheme.set(effective);
    document.documentElement.setAttribute('data-theme', effective);
  }
}
```

---

## Tailwind CSS Dark Mode

```javascript
// tailwind.config.js
module.exports = {
  darkMode: 'class', // or 'media' for OS-only
  // ...
};
```

```tsx
// With class strategy — add/remove 'dark' class on <html>
// (matches our data-theme approach; use 'dark' class instead of attribute)

function applyTheme(dark: boolean) {
  document.documentElement.classList.toggle('dark', dark);
}

// In component
<div className="bg-white dark:bg-gray-950 text-gray-900 dark:text-gray-100">
  ...
</div>
```

---

## Accessibility Considerations

- **Contrast**: Both themes must meet WCAG AA (4.5:1 for normal text, 3:1 for large). Test with browser DevTools or Lighthouse.
- **Images and SVGs**: Use `@media (prefers-color-scheme: dark)` in `<picture>` or `filter: invert()` sparingly.
- **Focus indicators**: Ensure focus rings are visible in both themes (use `currentColor` or explicit variables).
- **Third-party embeds** (maps, code blocks, charts): Many have their own dark mode APIs — handle them separately.

---

## Common Pitfalls

1. **No blocking script** — Causes FOUC (flash of unstyled/wrong content) on first load.
2. **Using only `@media (prefers-color-scheme: dark)`** — Can't override OS preference; user is stuck.
3. **Hardcoded colors in components** — Use semantic tokens; never `color: #111827` in a component.
4. **Not testing intermediate states** — OS=dark + preference=light should show light. Test all 4 combinations.
5. **SSR hydration mismatch** — Server doesn't know user preference; use `suppressHydrationWarning` and a blocking script.
6. **Forgetting meta `theme-color`** — Browser chrome (address bar, status bar on mobile) stays light even when your UI is dark.

---

## Interview Questions

**Q: How do you prevent the flash of wrong theme on initial page load?**  
A: Run a synchronous inline `<script>` in `<head>` before any CSS links. It reads `localStorage` and sets `data-theme` on `<html>` before the browser's first paint. No `defer`, no external file — it must be blocking.

**Q: What's the difference between `prefers-color-scheme` media query and a class-based approach?**  
A: The media query auto-responds to OS preference but doesn't let users override it. A class/attribute-based approach (with a stored user preference) supports three states: system, light, dark. Most production apps need the class-based approach with `prefers-color-scheme` as the fallback for the 'system' state.

**Q: How do you sync the theme across multiple open tabs?**  
A: Listen to the `storage` event on `window`. It fires in all tabs except the one that triggered the `localStorage` write. Update the `data-theme` attribute and any UI state in the handler.

**Q: Why do you need `suppressHydrationWarning` in Next.js dark mode?**  
A: The server renders HTML with no knowledge of the user's theme preference, so `data-theme` is absent or wrong. The blocking script on the client sets the correct value before React hydrates, causing a mismatch. `suppressHydrationWarning` on `<html>` tells React to ignore that attribute's mismatch rather than warning or attempting to reconcile it.
