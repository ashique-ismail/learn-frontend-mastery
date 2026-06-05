# Visual Regression Testing

## The Idea

**In plain English:** Visual regression testing is a way to automatically check whether a website looks the same after code changes, by taking before-and-after screenshots and highlighting any differences — even tiny ones a human might miss.

**Real-world analogy:** Imagine you hang a framed poster on your wall, take a photo of it, then do some renovation work in the room. When you're done, you take another photo and hold it next to the first one to see if anything shifted or changed without you intending it to.

- The original photo = the baseline screenshot (the known-good reference image)
- The new photo after renovation = the new screenshot taken after a code change
- Comparing the two photos side-by-side = the pixel-by-pixel diff that flags any differences

---

## Overview

Visual regression testing catches unintended UI changes by comparing screenshots across commits. A CSS refactor that "only changes specificity" might cause a button to shift 2px — invisible in code review, obvious in a screenshot diff. This guide covers screenshot diffing approaches, the major tools (Percy, Chromatic, Playwright), component-level vs page-level testing, and how to integrate visual testing into CI without it becoming a maintenance burden.

## What Visual Regression Testing Catches

```
Changes that pass code review but break visually:

✓ CSS specificity changes that shift layout
✓ Font loading differences between environments
✓ Storybook component regressions across refactors
✓ Third-party widget UI changes after version bump
✓ Responsive breakpoint regressions on non-default viewports
✓ Animation end-state differences
✓ Color theme changes that miss some components
✓ Z-index / stacking context regressions
✓ Icon or image swap without noticing

What it does NOT catch:
✗ Functional bugs (button still looks correct but does wrong thing)
✗ Performance regressions
✗ Accessibility issues (see axe-core)
✗ Server-side data changes (flaky if screenshots include dynamic content)
```

## The Core Workflow

```
Visual regression CI pipeline:

  PR opened
     │
     ▼
  Build app (with story/page snapshots)
     │
     ▼
  Take screenshots (current branch)
     │
     ▼
  Compare against baseline (main branch)
     │
     ├─ No diff → ✅ Pass
     │
     └─ Diff detected:
           ├─ Intentional change → Accept new baseline
           └─ Unintentional → ❌ Fail → Author investigates
```

## Playwright Visual Comparisons

Playwright has built-in screenshot comparison using the `toHaveScreenshot()` assertion:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  use: {
    baseURL: 'http://localhost:3000',
  },
  projects: [
    {
      name: 'chromium-1280x720',
      use: { ...devices['Desktop Chrome'], viewport: { width: 1280, height: 720 } },
    },
    {
      name: 'chromium-375x667',
      use: { ...devices['iPhone SE'], viewport: { width: 375, height: 667 } },
    },
    {
      name: 'firefox-1280x720',
      use: { ...devices['Desktop Firefox'], viewport: { width: 1280, height: 720 } },
    },
  ],
  // Screenshot comparison threshold
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01, // allow 1% pixel difference
      threshold: 0.2,          // color difference threshold per pixel
    },
  },
});
```

```typescript
// tests/visual/homepage.spec.ts
import { test, expect } from '@playwright/test';

test('homepage matches snapshot', async ({ page }) => {
  await page.goto('/');

  // Wait for dynamic content to settle
  await page.waitForLoadState('networkidle');
  await page.waitForTimeout(500); // any animation to complete

  // Full page screenshot comparison
  await expect(page).toHaveScreenshot('homepage.png', {
    fullPage: true,
  });
});

test('navigation responsive states', async ({ page }) => {
  await page.goto('/');

  // Desktop navigation
  await expect(page.locator('nav')).toHaveScreenshot('nav-desktop.png');

  // Simulate mobile
  await page.setViewportSize({ width: 375, height: 667 });
  await expect(page.locator('nav')).toHaveScreenshot('nav-mobile.png');
});

test('button states', async ({ page }) => {
  await page.goto('/design-system/buttons');

  // Normal state
  const button = page.locator('[data-testid="primary-button"]');
  await expect(button).toHaveScreenshot('button-default.png');

  // Hover state
  await button.hover();
  await expect(button).toHaveScreenshot('button-hover.png');

  // Focus state
  await button.focus();
  await expect(button).toHaveScreenshot('button-focused.png');

  // Disabled state
  const disabledButton = page.locator('[data-testid="primary-button"][disabled]');
  await expect(disabledButton).toHaveScreenshot('button-disabled.png');
});
```

### Handling Dynamic Content

```typescript
// Mask dynamic content that changes between runs
test('dashboard with masked dynamic areas', async ({ page }) => {
  await page.goto('/dashboard');

  // Mask areas with timestamps, user data, or live values
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [
      page.locator('[data-testid="last-login"]'),    // timestamp
      page.locator('[data-testid="unread-count"]'),  // dynamic count
      page.locator('.avatar-image'),                  // user avatar
    ],
  });
});

// Or: set data-testid="snapshot-mask" on all dynamic elements
// and mask them all at once
await expect(page).toHaveScreenshot('page.png', {
  mask: page.locator('[data-snapshot-mask]').all().then(handles => handles),
});
```

### Updating Baselines

```bash
# First run: no baseline exists → creates snapshot files
npx playwright test --update-snapshots

# After intentional UI changes: update baseline
npx playwright test visual/ --update-snapshots

# Baselines stored in: tests/visual/__snapshots__/
# Commit to git — baselines are version-controlled
```

## Chromatic — Storybook Visual Testing

Chromatic is built for component-level visual testing via Storybook:

```bash
npm install --save-dev chromatic
```

```typescript
// .storybook/main.ts — configure Storybook normally
// Each story becomes a screenshot test

// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  component: Button,
  parameters: {
    chromatic: {
      // Capture both light and dark themes
      themes: [
        { name: 'light', class: 'theme-light', color: '#fff' },
        { name: 'dark', class: 'theme-dark', color: '#1a1a1a' },
      ],
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { variant: 'primary', children: 'Click me' },
};

export const AllStates: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '8px' }}>
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="primary" disabled>Disabled</Button>
    </div>
  ),
};
```

```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on: push

jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for Chromatic to compare branches
      - run: npm ci
      - name: Publish to Chromatic
        uses: chromaui/action@latest
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          exitZeroOnChanges: true  # Don't fail CI; let Chromatic UI decide
          autoAcceptChanges: false # Require human review
```

```
Chromatic workflow:
  1. Build Storybook
  2. Publish to Chromatic
  3. Screenshot each story across configured viewports
  4. Compare against accepted baseline
  5. UI shows diffs — maintainer approves or rejects each change
  6. On approval → new baseline
```

## Percy — Cross-Browser Visual Testing

Percy integrates with any test framework and supports cross-browser snapshots via Chromium, Firefox, and WebKit simultaneously:

```bash
npm install --save-dev @percy/cli @percy/playwright
```

```typescript
// tests/visual/percy.spec.ts
import { test } from '@playwright/test';
import { percySnapshot } from '@percy/playwright';

test('homepage snapshot', async ({ page }) => {
  await page.goto('/');
  await page.waitForLoadState('networkidle');

  // Percy captures and uploads to their cloud for comparison
  await percySnapshot(page, 'Homepage');
});

test('responsive product card', async ({ page }) => {
  await page.goto('/products/p1');

  await percySnapshot(page, 'Product Page - Desktop', {
    widths: [1280, 1024, 768],  // multiple widths in one snapshot
  });
});
```

```yaml
# GitHub Actions with Percy
- name: Run visual tests
  env:
    PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
  run: |
    npx percy exec -- npx playwright test tests/visual/
```

## Component-Level vs Page-Level Testing

```
Component-level (Chromatic/Storybook):
  Advantages:
    + Fast — no need to spin up full app
    + Isolated — changes in one component don't affect other tests
    + Easy to test all states (empty, loading, error, full data)
    + Runs against mocked data — no flakiness from real APIs

  Disadvantages:
    - Doesn't catch integration regressions (component A + B together)
    - Requires Storybook maintenance
    - Mocked data may not reflect real edge cases

Page-level (Playwright/Percy):
  Advantages:
    + Tests the real integration
    + Catches regressions in component composition
    + Tests responsive layouts in context
    + Tests navigation and flow visuals

  Disadvantages:
    - Slower — full app rendering
    - More flaky — real network, dynamic data
    - More maintenance — full page changes more often
    - Requires masking or seeding dynamic content
```

**Recommended approach:** Component-level for design system components (high confidence, fast), page-level for critical user flows (homepage, checkout, onboarding) with all dynamic content masked.

## CI Integration Best Practices

```yaml
# Complete visual regression workflow
name: Visual Tests

on:
  pull_request:
    branches: [main]

jobs:
  visual-regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Start application
        run: |
          npm run build
          npm run start:ci &
          npx wait-on http://localhost:3000 --timeout 30000

      - name: Run visual tests
        run: npx playwright test tests/visual/ --reporter=html

      - name: Upload screenshot diffs on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-visual-diffs
          path: |
            tests/visual/__snapshots__/
            playwright-report/
          retention-days: 7
```

## Handling Flaky Visual Tests

Visual tests are prone to flakiness from timing, font loading, and dynamic content:

```typescript
// Strategy 1: Disable animations for snapshots
await page.addStyleTag({
  content: `
    *, *::before, *::after {
      animation-duration: 0.01ms !important;
      transition-duration: 0.01ms !important;
    }
  `,
});
await expect(page).toHaveScreenshot('stable.png');

// Strategy 2: Wait for specific signals before capturing
await page.waitForSelector('[data-snapshot-ready]', { state: 'visible' });
// Add data-snapshot-ready to the DOM when all async content has loaded

// Strategy 3: Mask volatile regions
await expect(page).toHaveScreenshot('page.png', {
  mask: [page.locator('.live-data'), page.locator('[data-volatile]')],
});

// Strategy 4: Use stable test data
// Seed your database with fixed test data before visual runs
// Use a separate URL/environment specifically for visual testing
```

## Common Mistakes

### 1. Too Many Page-Level Visual Tests

```typescript
// ❌ Taking full-page screenshots of every page in the app
// → Slow CI, high maintenance, frequent unrelated failures

// ✅ Component-level for components; page-level only for critical paths
// → Keep visual tests fast and focused
```

### 2. Not Masking Dynamic Content

```typescript
// ❌ Unmasked timestamps, user data → flaky tests
await expect(page).toHaveScreenshot('profile.png');
// Date shown: "Jan 15, 2026" → next week it shows "Jan 22, 2026" → test fails

// ✅ Mask dynamic content consistently
await expect(page).toHaveScreenshot('profile.png', {
  mask: [page.locator('[data-testid="last-seen"]')],
});
```

### 3. Not Committing Baselines to Version Control

```bash
# ❌ .gitignore includes snapshot files
# → Each CI run generates new "baseline" → all tests pass trivially

# ✅ Commit snapshot files
git add tests/visual/__snapshots__/
# Baseline changes appear in PR diffs — intentional changes are visible
```

### 4. Ignoring Cross-Browser Differences

```typescript
// ❌ Only testing in Chromium
// → Firefox renders fonts slightly differently → test fails on Firefox in prod

// ✅ Test in multiple browsers; use generous thresholds or browser-specific baselines
// playwright.config.ts: multiple projects (chromium, firefox, webkit)
// Or: use Percy/Chromatic which handle cross-browser baselines for you
```

## Interview Questions

### 1. What is visual regression testing and what types of bugs does it catch?

**Answer:** Visual regression testing captures screenshots and compares them pixel-by-pixel against baseline images from a known-good state. It catches changes that pass code review and unit tests but break visually: CSS specificity conflicts that shift layout, font rendering differences across environments, component regressions from refactoring, responsive breakpoint failures, and color/theme changes that miss some components. It complements functional tests — a button can work correctly and still look broken. The key value is catching unintentional visual changes between commits, especially in large-scale refactors.

### 2. When would you choose Chromatic/Storybook over Playwright for visual testing?

**Answer:** Chromatic when testing design system components in isolation: fast, stable (no real network), covers all states (loading, error, empty, full), runs on every story automatically. Playwright when testing full-page layouts and integration of multiple components, responsive behavior in real browser context, or flows that require navigation and real data. In practice: use both — Chromatic for the component library where comprehensive coverage matters, Playwright for critical user-facing pages (homepage, checkout) where the real integration must be validated.

### 3. How do you handle flaky visual tests caused by dynamic content?

**Answer:** Three strategies: (1) Mask dynamic regions using `mask: [page.locator('.dynamic')]` — they appear as gray boxes in screenshots, eliminating the content but keeping the layout; (2) seed deterministic test data — use a separate test environment with fixed data rather than real production data; (3) wait for a specific ready signal before capturing — add `data-snapshot-ready` to the DOM when all async content has loaded and wait for it in tests. For animations, inject CSS that sets all durations to 0.01ms before capturing. The goal is deterministic snapshots where the only variable is the intentional code change you're testing.

### 4. How are visual regression baselines managed in a team?

**Answer:** Baselines are committed to version control as image files. When a PR makes intentional visual changes, the author runs `--update-snapshots` locally, reviews the diff, and commits the new baselines. Unintentional visual changes show up in CI as failures with diff artifacts uploaded. Tools like Chromatic and Percy add a layer of human review via a web UI — CI uploads screenshots, a maintainer reviews diffs and approves/rejects each change, and approved baselines become the new reference. This prevents baselines from silently drifting and creates an auditable history of every visual change.
