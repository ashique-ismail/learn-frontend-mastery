# Automated Accessibility Testing

## Overview

Automated accessibility testing catches the subset of WCAG violations that are programmatically detectable — missing alt text, insufficient color contrast, unlabeled form inputs, broken ARIA usage. WebAIM estimates that automated tools can catch 30-40% of accessibility issues; the rest require manual testing and real-user feedback. This guide covers axe-core, jest-axe, Playwright accessibility assertions, what automated testing reliably catches, and where it falls short.

## What Automated Testing Can and Cannot Catch

```
Automated tools reliably catch (~30-40% of issues):

✓ Missing alt text on images
✓ Insufficient color contrast ratio (< 4.5:1 for normal text)
✓ Form inputs without accessible labels
✓ Missing document language (lang attribute)
✓ Duplicate IDs
✓ Invalid ARIA roles or attributes
✓ ARIA required attributes missing
✓ Buttons/links with no accessible text
✓ Empty headings
✓ Keyboard focus traps in unexpected places
✓ iframes without titles

Requires manual testing:

✗ Logical reading order (DOM order vs visual order)
✗ Meaningful alt text quality (present but wrong: "image1.jpg" vs descriptive)
✗ Focus management in modals and SPAs
✗ Keyboard navigation flow through complex widgets
✗ Screen reader announcement quality and timing
✗ Cognitive accessibility (plain language, consistent navigation)
✗ Touch target size in context
✗ Motion-sensitive content
✗ Content that requires visual interpretation
```

## axe-core

axe-core is the industry-standard accessibility engine, maintained by Deque Systems and used by every major accessibility testing tool:

```bash
npm install --save-dev axe-core
```

### axe-core Directly in Browser

```typescript
// Inject into browser for manual testing
import axe from 'axe-core';

// Analyze entire page
const results = await axe.run();
console.log('Violations:', results.violations);
console.log('Passes:', results.passes.length);
console.log('Incomplete (needs manual check):', results.incomplete.length);

// Analyze a specific element
const results = await axe.run(document.getElementById('main-content')!);

// Filter by WCAG level
const wcag2aa = await axe.run({
  runOnly: {
    type: 'tag',
    values: ['wcag2a', 'wcag2aa'],
  },
});
```

### Understanding axe Results

```typescript
// Violation structure
interface AxeViolation {
  id: string;            // rule ID: "color-contrast", "label", etc.
  impact: 'critical' | 'serious' | 'moderate' | 'minor';
  description: string;  // human-readable description
  helpUrl: string;       // link to detailed fix guidance
  nodes: Array<{
    html: string;        // the problematic HTML
    target: string[];    // CSS selector path
    failureSummary: string; // what exactly failed
  }>;
}

// Example: missing label violation
{
  id: "label",
  impact: "critical",
  description: "Ensures every form element has a label",
  helpUrl: "https://dequeuniversity.com/rules/axe/4.7/label",
  nodes: [{
    html: '<input type="email" placeholder="Email">',
    target: ['#signup-form input[type="email"]'],
    failureSummary: 'Fix any of the following: ...'
  }]
}
```

## jest-axe — Integration with Jest

```bash
npm install --save-dev jest-axe
npm install --save-dev @types/jest-axe
```

```typescript
// setupTests.ts — extend expect
import { configureAxe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

// Or configure globally in jest.config.ts
// setupFilesAfterFramework: ['./src/setupTests.ts']
```

### Testing React Components with jest-axe

```typescript
// Button.test.tsx
import { render } from '@testing-library/react';
import { axe } from 'jest-axe';

// ✅ Accessible button
describe('Button accessibility', () => {
  it('accessible with text content', async () => {
    const { container } = render(
      <button type="button">Submit Order</button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('accessible icon button with aria-label', async () => {
    const { container } = render(
      <button type="button" aria-label="Close dialog">
        <XIcon aria-hidden="true" />
      </button>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('catches missing label on icon-only button', async () => {
    const { container } = render(
      <button type="button">
        <XIcon />  {/* No text, no aria-label */}
      </button>
    );
    const results = await axe(container);
    // This SHOULD have violations — we're documenting the bad pattern
    expect(results.violations.length).toBeGreaterThan(0);
    expect(results.violations[0].id).toBe('button-name');
  });
});
```

### Testing Forms

```typescript
// Form.test.tsx
describe('ContactForm accessibility', () => {
  it('has no violations with valid markup', async () => {
    const { container } = render(<ContactForm />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('error messages are accessible', async () => {
    const { container, getByRole } = render(<ContactForm />);

    // Submit without filling out — trigger errors
    fireEvent.click(getByRole('button', { name: /submit/i }));

    // Wait for error state
    await waitFor(() => {
      expect(getByRole('alert')).toBeInTheDocument();
    });

    const results = await axe(container);
    expect(results).toHaveNoViolations();
    // Verifies: error messages are associated with fields (aria-describedby),
    // errors have role="alert" or aria-live="polite"
  });

  it('required fields communicate requirement', async () => {
    const { container } = render(<ContactForm />);
    // axe checks: required fields have aria-required or required attribute
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

### Testing Modal Dialogs

```typescript
// Modal.test.tsx
describe('Modal accessibility', () => {
  it('closed modal has no violations', async () => {
    const { container } = render(
      <Modal isOpen={false} onClose={() => {}}>
        <p>Content</p>
      </Modal>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('open modal has no violations', async () => {
    const { container } = render(
      <Modal isOpen={true} onClose={() => {}}>
        <h2 id="modal-title">Confirm Delete</h2>
        <p>Are you sure?</p>
        <button onClick={() => {}}>Cancel</button>
        <button onClick={() => {}}>Delete</button>
      </Modal>
    );
    const results = await axe(container);
    expect(results).toHaveNoViolations();
    // axe checks: dialog has role="dialog" or role="alertdialog",
    // aria-labelledby or aria-label present
  });
});
```

### Configuring axe Rules

```typescript
// Disable specific rules (document why!)
const results = await axe(container, {
  rules: {
    // color-contrast: skip in tests — colors depend on CSS variables
    // that JSDOM doesn't compute correctly
    'color-contrast': { enabled: false },
  },
});

// Target only specific WCAG levels
const results = await axe(container, {
  runOnly: {
    type: 'tag',
    values: ['wcag2a', 'wcag2aa'],
  },
});

// Run specific rules only
const results = await axe(container, {
  runOnly: {
    type: 'rule',
    values: ['label', 'button-name', 'image-alt'],
  },
});
```

## Playwright Accessibility Assertions

Playwright provides a snapshot-based accessibility API via `getByRole` and `accessibility.snapshot()`:

```typescript
// tests/accessibility/homepage.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('homepage has no critical accessibility violations', async ({ page }) => {
  await page.goto('/');

  // Run axe against the whole page
  const accessibilityScanResults = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();

  expect(accessibilityScanResults.violations).toEqual([]);
});

test('checkout flow is accessible', async ({ page }) => {
  await page.goto('/checkout');

  // Scan only the main content area (exclude third-party widgets)
  const results = await new AxeBuilder({ page })
    .include('#main-content')
    .exclude('.third-party-chat-widget')
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Keyboard Navigation Testing

```typescript
test('navigation is keyboard accessible', async ({ page }) => {
  await page.goto('/');

  // Start from the beginning of the document
  await page.keyboard.press('Tab');

  // Skip to main content link should be first focusable element
  const focusedEl = await page.locator(':focus').textContent();
  expect(focusedEl).toMatch(/skip to main|skip navigation/i);

  // Tab through main navigation
  await page.keyboard.press('Tab'); // first nav item
  await page.keyboard.press('Tab'); // second nav item

  // Verify focus is visible
  const navItem = page.locator('nav a').first();
  await navItem.focus();

  // Check focus ring is visible (computed style)
  const outline = await navItem.evaluate((el) => {
    return window.getComputedStyle(el).outline;
  });
  expect(outline).not.toBe('0px none rgb(0, 0, 0)');
});

test('modal traps focus correctly', async ({ page }) => {
  await page.goto('/products/p1');

  // Open modal
  await page.click('button:text("Add to cart")');
  await page.waitForSelector('[role="dialog"]');

  // Tab through all focusable elements in modal
  const focusableInModal = await page.locator('[role="dialog"] button, [role="dialog"] [href], [role="dialog"] input').count();

  for (let i = 0; i < focusableInModal + 2; i++) {
    await page.keyboard.press('Tab');
    // Focus should stay within the modal
    const focusedElement = await page.evaluate(() => document.activeElement?.closest('[role="dialog"]'));
    expect(focusedElement).toBeTruthy();
  }
});
```

### Accessibility Tree Snapshot

```typescript
test('heading hierarchy is correct', async ({ page }) => {
  await page.goto('/about');

  // Get accessibility tree snapshot
  const snapshot = await page.accessibility.snapshot();

  // Verify heading structure
  expect(snapshot?.children?.filter(
    (node) => node.role === 'heading'
  )).toMatchObject([
    { role: 'heading', level: 1, name: expect.stringMatching(/About Us/i) },
    { role: 'heading', level: 2, name: expect.any(String) },
  ]);
});
```

## Color Contrast Testing

```typescript
// jest-axe catches color contrast issues, but JSDOM doesn't compute CSS variables
// Use Playwright for real browser color contrast testing

test('text meets contrast requirements', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page })
    .withRules(['color-contrast'])
    .analyze();

  if (results.violations.length > 0) {
    console.log(
      results.violations
        .flatMap((v) => v.nodes)
        .map((n) => `${n.html} - ${n.failureSummary}`)
        .join('\n')
    );
  }

  expect(results.violations).toHaveLength(0);
});
```

## Integrating into CI

```yaml
# .github/workflows/accessibility.yml
name: Accessibility Tests

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  accessibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }

      - run: npm ci

      # Unit-level: jest-axe (fast, component-level)
      - name: Run jest-axe tests
        run: npm test -- --testPathPattern="*.a11y.test"

      # Integration-level: Playwright + axe (full browser)
      - name: Install Playwright
        run: npx playwright install chromium --with-deps

      - name: Start app
        run: npm run build && npm start &

      - name: Run Playwright accessibility tests
        run: npx playwright test tests/accessibility/

      - name: Upload violation report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: a11y-report
          path: playwright-report/
```

## Common Mistakes

### 1. Treating Zero Violations as "Accessible"

```typescript
// ❌ Zero axe violations ≠ accessible
// axe catches ~30-40% of issues
// A form can pass all axe rules and still be unusable with a screen reader

// ✅ Automated testing is a floor, not a ceiling
// Complement with:
//   - Manual keyboard navigation testing
//   - Screen reader testing (VoiceOver, NVDA, JAWS)
//   - User testing with people with disabilities
```

### 2. Not Testing Interactive States

```typescript
// ❌ Only testing the initial render
const { container } = render(<Form />);
const results = await axe(container);
expect(results).toHaveNoViolations();
// Doesn't test: error state, loading state, disabled state

// ✅ Test all meaningful states
const states = [
  { name: 'default', action: () => {} },
  { name: 'error', action: () => { fireEvent.submit(form); } },
  { name: 'loading', action: () => { /* trigger loading state */ } },
];

for (const { name, action } of states) {
  action();
  await waitFor(async () => {
    const results = await axe(container);
    expect(results).toHaveNoViolations(), `${name} state has violations`;
  });
}
```

### 3. Skipping color-contrast Without Documentation

```typescript
// ❌ Silently disabling rules
const results = await axe(container, {
  rules: { 'color-contrast': { enabled: false } }
});

// ✅ Document why and add a manual check reminder
// JSDOM doesn't resolve CSS custom properties used for color theming.
// Color contrast is tested separately in Playwright against a real browser.
// See: tests/accessibility/color-contrast.spec.ts
const results = await axe(container, {
  rules: { 'color-contrast': { enabled: false } }
});
```

## Interview Questions

### 1. What percentage of accessibility issues can automated tools catch?

**Answer:** Research and WebAIM studies estimate automated tools catch roughly 30-40% of WCAG violations. They reliably detect structural issues: missing alt text, absent form labels, duplicate IDs, invalid ARIA, insufficient color contrast (in real browsers), and missing document language. They cannot detect issues requiring human judgment: whether alt text is meaningful (not just present), logical reading order, focus management quality in SPAs, screen reader announcement appropriateness, or cognitive accessibility (plain language, consistent navigation). Automated testing is essential as a floor — it catches the low-hanging fruit consistently — but manual testing with screen readers and user testing are required for real accessibility.

### 2. What is axe-core and what makes it the industry standard?

**Answer:** axe-core is an open-source accessibility testing engine maintained by Deque Systems. It's the underlying engine for the axe browser extension, jest-axe, @axe-core/playwright, Lighthouse accessibility audits, and most commercial accessibility scanning tools. It's the standard because: it has the most comprehensive and actively maintained ruleset (covering WCAG 2.0, 2.1, and 2.2), it produces zero false positives by design (it only reports violations it's certain about — marking uncertain cases as "incomplete"), it's embeddable in any test environment, and it returns structured, actionable results with help URLs and CSS selectors for each violation.

### 3. What is the difference between violations, passes, and incomplete in axe results?

**Answer:** `violations` are confirmed WCAG failures — axe is certain these are issues. `passes` are rules that were checked and passed — useful for auditing. `incomplete` (also called "needs review") are cases where axe detected a potential issue but couldn't confirm it algorithmically — these require manual review. For example, axe might flag an image alt text as "incomplete" if the alt text exists but it can't determine if it's meaningful. In CI, you typically assert `violations.length === 0`. `incomplete` items should feed a manual accessibility checklist rather than fail the build, since they have a higher false positive rate.

### 4. How do you test keyboard accessibility with Playwright?

**Answer:** Use `page.keyboard.press('Tab')` to move focus through the page and assert that focus lands on the expected elements in the expected order. Verify the first Tab press reaches a "skip to main content" link. Check that focus never becomes invisible — assert that `:focus` has a visible outline via computed styles. For modals, verify focus is trapped within the dialog while open: Tab repeatedly and confirm `document.activeElement` always has the modal as an ancestor. After modal close, verify focus returns to the trigger element. Also check Escape closes the modal via `page.keyboard.press('Escape')`. These tests cover WCAG 2.4.3 (Focus Order) and 2.1.1 (Keyboard).
