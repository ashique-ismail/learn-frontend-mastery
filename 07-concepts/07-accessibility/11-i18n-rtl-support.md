# Internationalization (i18n) and RTL Support

## The Idea

**In plain English:** Internationalization (i18n) is the process of building an app so it can be translated and adapted for people in any country — handling different languages, number formats, dates, and even text that reads right-to-left (RTL) instead of left-to-right (like Arabic or Hebrew). RTL support means your layout and icons mirror themselves automatically when a user switches to one of those languages.

**Real-world analogy:** Think of a clothing store that ships worldwide. When they design their labels, they leave blank spaces for each country's team to fill in the local language, price format, and size chart — the label template is built once, but adapted per region.

- The blank label template = the i18n setup in code (message IDs, format hooks)
- The country-specific content filled in = the translation files (l10n)
- Mirroring the store layout for countries that read right-to-left = RTL CSS support

---

## Overview

Internationalization (i18n) is the process of designing software so it can be adapted to different locales without engineering changes. Localization (l10n) is the actual adaptation for a specific market — translations, date formats, currency symbols, and layout direction. RTL (right-to-left) languages — Arabic, Hebrew, Persian, Urdu — have a combined speaker count of over 700 million people and require specific layout considerations that go far beyond text direction. This guide covers the i18n architecture, RTL layout techniques, CSS logical properties, and React i18n library choices.

## i18n vs l10n vs a11y

```text
i18n (Internationalization):
  Engineering work — making the app capable of being localized.
  Once. Supports any locale.
  Examples: message ID system, locale-aware date/number formatting,
            RTL layout support, font fallbacks

l10n (Localization):
  Per-locale content work — translations, regional formatting.
  Repeated for each new locale.
  Examples: translating strings, adapting images, currency symbols

g11n (Globalization):
  The combination of both — the full process of making a product
  work globally.
```

## Message Extraction and Translation Files

### ICU Message Format

```typescript
// ICU format supports pluralization, gender, selection, and variables
// Supported by react-i18next, @formatjs/intl, Lingui

// Simple string
"welcome": "Welcome, {name}!"

// Plural
"item_count": "{count, plural,
  =0 {No items}
  =1 {One item}
  other {# items}
}"

// Select (gender-aware)
"user_greeting": "{gender, select,
  male {He is our customer}
  female {She is our customer}
  other {They are our customer}
}"

// Nested
"notification": "You have {count, plural,
  =0 {no new messages}
  one {# new message from {sender}}
  other {# new messages}
}"
```

## react-i18next

The most widely used i18n library for React:

```bash
npm install react-i18next i18next i18next-browser-languagedetector i18next-http-backend
```

```typescript
// i18n.ts — initialization
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import Backend from 'i18next-http-backend';

i18n
  .use(Backend)             // Load translations from /public/locales/<lng>/<ns>.json
  .use(LanguageDetector)    // Auto-detect user language from browser
  .use(initReactI18next)    // Bind to React
  .init({
    fallbackLng: 'en',
    supportedLngs: ['en', 'ar', 'fr', 'de', 'ja'],
    debug: process.env.NODE_ENV === 'development',

    interpolation: {
      escapeValue: false, // React already escapes values
    },

    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json',
    },

    detection: {
      order: ['querystring', 'cookie', 'navigator', 'htmlTag'],
      caches: ['cookie'],
    },
  });

export default i18n;
```

```json
// public/locales/en/common.json
{
  "greeting": "Hello, {{name}}!",
  "items_in_cart": "{{count}} item in cart",
  "items_in_cart_plural": "{{count}} items in cart",
  "last_updated": "Last updated: {{date}}"
}
```

```json
// public/locales/ar/common.json — Arabic translation
{
  "greeting": "مرحباً، {{name}}!",
  "items_in_cart": "{{count}} عنصر في السلة",
  "items_in_cart_plural": "{{count}} عناصر في السلة",
  "last_updated": "آخر تحديث: {{date}}"
}
```

```typescript
// Component usage
import { useTranslation } from 'react-i18next';

function Header({ name }: { name: string }) {
  const { t, i18n } = useTranslation('common');

  return (
    <header>
      <h1>{t('greeting', { name })}</h1>
      <button onClick={() => i18n.changeLanguage('ar')}>
        عربي
      </button>
      <button onClick={() => i18n.changeLanguage('en')}>
        English
      </button>
    </header>
  );
}

// Plural handling (i18next uses _plural suffix by default)
function CartBadge({ count }: { count: number }) {
  const { t } = useTranslation('common');
  return <span>{t('items_in_cart', { count })}</span>;
  // Auto-selects singular/plural based on count
}
```

## Intl API — Browser-Native Formatting

```typescript
// Date formatting — locale-aware
const date = new Date('2026-01-15');

new Intl.DateTimeFormat('en-US').format(date);  // "1/15/2026"
new Intl.DateTimeFormat('de-DE').format(date);  // "15.1.2026"
new Intl.DateTimeFormat('ar-EG').format(date);  // "١٥‏/١‏/٢٠٢٦" (Arabic-Indic digits)
new Intl.DateTimeFormat('ja-JP').format(date);  // "2026/1/15"

// Number formatting
const amount = 1234567.89;
new Intl.NumberFormat('en-US').format(amount);   // "1,234,567.89"
new Intl.NumberFormat('de-DE').format(amount);   // "1.234.567,89"
new Intl.NumberFormat('ar-EG').format(amount);   // "١٬٢٣٤٬٥٦٧٫٨٩"

// Currency
new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(1000);
// "$1,000.00"
new Intl.NumberFormat('de-DE', { style: 'currency', currency: 'EUR' }).format(1000);
// "1.000,00 €"
new Intl.NumberFormat('ar-SA', { style: 'currency', currency: 'SAR' }).format(1000);
// "١٬٠٠٠٫٠٠ ر.س."

// Relative time
const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });
rtf.format(-1, 'day');    // "yesterday"
rtf.format(-3, 'hour');   // "3 hours ago"
rtf.format(2, 'day');     // "in 2 days"

const rtfAr = new Intl.RelativeTimeFormat('ar', { numeric: 'auto' });
rtfAr.format(-1, 'day');  // "أمس"
rtfAr.format(-3, 'hour'); // "قبل ٣ ساعات"
```

```typescript
// React hook for locale-aware formatting
import { useCallback } from 'react';
import { useTranslation } from 'react-i18next';

function useFormatters() {
  const { i18n } = useTranslation();
  const locale = i18n.language;

  const formatDate = useCallback(
    (date: Date, options?: Intl.DateTimeFormatOptions) =>
      new Intl.DateTimeFormat(locale, options).format(date),
    [locale]
  );

  const formatCurrency = useCallback(
    (amount: number, currency: string) =>
      new Intl.NumberFormat(locale, { style: 'currency', currency }).format(amount),
    [locale]
  );

  const formatRelativeTime = useCallback(
    (date: Date) => {
      const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
      const diff = date.getTime() - Date.now();
      const seconds = Math.round(diff / 1000);
      if (Math.abs(seconds) < 60) return rtf.format(seconds, 'second');
      const minutes = Math.round(seconds / 60);
      if (Math.abs(minutes) < 60) return rtf.format(minutes, 'minute');
      const hours = Math.round(minutes / 60);
      if (Math.abs(hours) < 24) return rtf.format(hours, 'hour');
      return rtf.format(Math.round(hours / 24), 'day');
    },
    [locale]
  );

  return { formatDate, formatCurrency, formatRelativeTime };
}
```

## RTL Layout Fundamentals

```text
LTR (English, French, German):
  ←──── reading direction ────→
  [Logo]  [Nav]  [Search]  [Avatar]   ← header flows left to right
  Content  |  Sidebar                 ← main content on left, sidebar on right
  Prev  1 2 3  Next                   ← pagination

RTL (Arabic, Hebrew, Persian):
  ←────── reading direction ──────
  [Avatar]  [Search]  [Nav]  [شعار]  ← header flows right to left
  Sidebar  |  Content                ← sidebar on left, main content on right
  التالي  ١ ٢ ٣  السابق              ← pagination mirrored
```

### Setting Document Direction

```html
<!-- Static HTML -->
<html lang="ar" dir="rtl">

<!-- Dynamic (JavaScript) -->
document.documentElement.dir = i18n.language === 'ar' || i18n.language === 'he' ? 'rtl' : 'ltr';
document.documentElement.lang = i18n.language;
```

```typescript
// React — set dir on language change
import { useEffect } from 'react';
import { useTranslation } from 'react-i18next';

const RTL_LANGUAGES = new Set(['ar', 'he', 'fa', 'ur']);

function App() {
  const { i18n } = useTranslation();

  useEffect(() => {
    const isRTL = RTL_LANGUAGES.has(i18n.language);
    document.documentElement.dir = isRTL ? 'rtl' : 'ltr';
    document.documentElement.lang = i18n.language;
  }, [i18n.language]);

  return <AppRoutes />;
}
```

## CSS Logical Properties

CSS Logical Properties are the modern solution for RTL support — they use flow-relative terms (`start`/`end`, `block`/`inline`) instead of physical terms (`left`/`right`, `top`/`bottom`):

```css
/* ❌ Physical properties — must override for RTL */
.card {
  margin-left: 16px;
  padding-right: 24px;
  border-left: 4px solid blue;
  text-align: left;
}

/* RTL override needed: */
[dir="rtl"] .card {
  margin-left: 0;
  margin-right: 16px;
  padding-right: 0;
  padding-left: 24px;
  border-left: none;
  border-right: 4px solid blue;
  text-align: right;
}

/* ✅ Logical properties — automatically RTL-aware */
.card {
  margin-inline-start: 16px;    /* left in LTR, right in RTL */
  padding-inline-end: 24px;     /* right in LTR, left in RTL */
  border-inline-start: 4px solid blue;
  text-align: start;            /* left in LTR, right in RTL */
}
/* No RTL override needed! */
```

### Logical Property Reference

```text
Physical → Logical equivalents:

SPACING:
  margin-left  → margin-inline-start
  margin-right → margin-inline-end
  padding-left → padding-inline-start
  padding-right → padding-inline-end

  (new shorthand):
  margin-inline: 16px;    /* = margin-left + margin-right */
  padding-inline: 24px;   /* = padding-left + padding-right */
  margin-block: 8px;      /* = margin-top + margin-bottom */

BORDERS:
  border-left  → border-inline-start
  border-right → border-inline-end
  border-top   → border-block-start
  border-bottom → border-block-end

SIZING:
  width  → inline-size
  height → block-size
  min-width → min-inline-size
  max-width → max-inline-size

POSITIONING:
  left  → inset-inline-start
  right → inset-inline-end
  top   → inset-block-start
  bottom → inset-block-end

TEXT:
  text-align: left  → text-align: start
  text-align: right → text-align: end

FLEXBOX (already logical):
  flex-direction: row (LTR start=left; RTL start=right — automatic!)
  justify-content: flex-start (already direction-aware)

BORDER RADIUS:
  border-top-left-radius     → border-start-start-radius
  border-top-right-radius    → border-start-end-radius
  border-bottom-left-radius  → border-end-start-radius
  border-bottom-right-radius → border-end-end-radius
```

## Bidirectional Text (BiDi)

```html
<!-- HTML dir attribute for mixed-direction content -->

<!-- Block-level: section in Arabic within English page -->
<p lang="ar" dir="rtl">هذا نص عربي</p>

<!-- Inline: Arabic name within English sentence -->
<p>
  Contact <span dir="rtl" lang="ar">محمد علي</span> for details.
</p>

<!-- CSS unicode-bidi for inline elements -->
<style>
  .rtl-inline {
    unicode-bidi: embed;   /* Treat inline text as RTL */
    direction: rtl;
  }
  .rtl-isolation {
    unicode-bidi: isolate; /* Isolate BiDi from surrounding text */
    direction: rtl;
  }
</style>
```

## Icons and Images in RTL

```css
/* Icons with directional meaning must flip in RTL */
/* Forward arrow → flip to point right in RTL context */

[dir="rtl"] .arrow-icon {
  transform: scaleX(-1);
}

/* Using CSS logical transform (CSS4) */
.arrow-icon {
  /* Flip only when layout direction is RTL */
  /* Not yet widely supported — use [dir="rtl"] override */
}

/* Logos, icons with no directional meaning: don't flip */
[dir="rtl"] .logo { /* NO transform */ }
[dir="rtl"] .heart-icon { /* NO transform */ }

/* Examples that SHOULD flip:
  ← → arrow icons
  < > chevron icons
  ⏪ ⏩ media player icons
  Checkmark positioning in checkboxes
  Progress bar fill direction

  Examples that should NOT flip:
  Clock icons (clockwise is universal)
  Currency symbols ($, €, ¥)
  Logos
  Warning/info icons (symmetric)
*/
```

## Number and Date Formatting in RTL

```typescript
// Numbers: Arabic locale uses Arabic-Indic digits by default
const number = 42;
new Intl.NumberFormat('ar').format(number);    // "٤٢" (Arabic-Indic)
new Intl.NumberFormat('ar-u-nu-latn').format(number); // "42" (Latin digits with extension)

// Some apps prefer Latin digits even in Arabic UI
// Use the -u-nu-latn Unicode extension

// Phone numbers — special handling
// Saudi: +966 50 000 0000 — right-to-left number, but digits are LTR
// Use <bdi> or unicode-bidi: isolate for phone numbers in RTL context
<bdi dir="ltr">+966 50 000 0000</bdi>
```

## Testing RTL

```typescript
// Playwright — test RTL layout
import { test, expect } from '@playwright/test';

test('layout mirrors correctly in Arabic', async ({ page }) => {
  // Set locale to Arabic
  await page.goto('/?lng=ar');

  // Verify document direction
  const dir = await page.evaluate(() => document.documentElement.dir);
  expect(dir).toBe('rtl');

  // Check that a left-aligned button is now right-aligned
  const button = page.locator('.primary-button');
  const boundingBox = await button.boundingBox();
  const viewportWidth = page.viewportSize()!.width;

  // Button should be closer to right edge in RTL
  expect(boundingBox!.x + boundingBox!.width).toBeGreaterThan(viewportWidth / 2);
});

// Visual regression for RTL
test('RTL visual appearance matches snapshot', async ({ page }) => {
  await page.goto('/?lng=ar');
  await expect(page).toHaveScreenshot('homepage-rtl.png');
});
```

## Lingui — Alternative i18n Library

```typescript
// Lingui uses tagged template literals and JSX — more natural feel
import { Trans, useLingui } from '@lingui/react';
import { msg } from '@lingui/macro';

// In components — no translation key strings
function Welcome({ name }: { name: string }) {
  const { i18n } = useLingui();

  return (
    <div>
      {/* JSX component for inline translations */}
      <Trans>Welcome, <strong>{name}</strong>!</Trans>

      {/* useLingui for JS strings */}
      <title>{i18n._(msg`My Application`)}</title>
    </div>
  );
}

// Lingui extracts these at build time via lingui extract
// Generates .po files for translators
```

## Common Mistakes

### 1. Using left/right Instead of Logical Properties

```css
/* ❌ Requires RTL overrides for every instance */
.nav-item { margin-left: 8px; }
[dir="rtl"] .nav-item { margin-left: 0; margin-right: 8px; }

/* ✅ Works in both directions automatically */
.nav-item { margin-inline-start: 8px; }
```

### 2. Hardcoding text-align: left

```css
/* ❌ Forces LTR alignment in RTL context */
.description { text-align: left; }

/* ✅ Respects reading direction */
.description { text-align: start; }
```

### 3. Not Using the Intl API for Dates and Numbers

```typescript
// ❌ Manual date formatting — doesn't adapt to locale
const formatted = `${date.getMonth() + 1}/${date.getDate()}/${date.getFullYear()}`;
// Always US format — wrong in Germany, Japan, or with Arabic digits

// ✅ Intl.DateTimeFormat
const formatted = new Intl.DateTimeFormat(locale).format(date);
```

### 4. Forgetting to Flip Directional Icons

```jsx
// ❌ Back arrow looks like "forward" in RTL
<button><ArrowLeftIcon /> Back</button>
// In Arabic RTL, this arrow points in the "forward" direction

// ✅ Flip the icon in RTL
const { i18n } = useTranslation();
const isRTL = RTL_LANGUAGES.has(i18n.language);
<button>
  <ArrowLeftIcon style={{ transform: isRTL ? 'scaleX(-1)' : 'none' }} />
  {t('back')}
</button>
```

## Interview Questions

### 1. What is the difference between i18n and l10n?

**Answer:** Internationalization (i18n) is the engineering work of building a system that *can* be localized — message extraction, locale-aware formatters, RTL layout support, font loading. Done once, it supports any future locale. Localization (l10n) is the per-locale content work — translating strings, adapting currency symbols, date formats, images, and cultural references for a specific market. i18n is a prerequisite for l10n: you can't translate a hardcoded string. For a developer, i18n means never concatenating strings, using message IDs, using Intl APIs for formatting, and building layouts that work in both LTR and RTL.

### 2. What are CSS Logical Properties and why do they matter for RTL support?

**Answer:** CSS Logical Properties replace physical directional terms (left/right) with flow-relative terms (inline-start/inline-end) that automatically adapt to the document's writing direction. `margin-inline-start` is `margin-left` in LTR and `margin-right` in RTL — no override needed. Without logical properties, every directional CSS rule needs a `[dir="rtl"]` override, which is error-prone and easily forgotten. With logical properties, writing RTL-compatible layouts requires no RTL-specific CSS at all. The `dir="rtl"` attribute on the root element handles the flip automatically.

### 3. How does the Intl API handle number and date formatting differences across locales?

**Answer:** `Intl.DateTimeFormat` and `Intl.NumberFormat` accept a locale string and formatting options and produce locale-appropriate output. German uses periods for thousands and commas for decimals; Arabic may use Arabic-Indic digits; Japanese uses a year-month-day order; Saudi Arabia formats currency differently from the US. Using these APIs means a single `format(value)` call produces the correct regional format without any custom logic. For relative time ("3 hours ago"), `Intl.RelativeTimeFormat` handles the locale-specific phrasing and pluralization rules automatically.

### 4. Which icons should flip in RTL layouts and which should not?

**Answer:** Icons with inherent directionality should flip: arrows (back/forward, left/right), chevrons, media player controls (rewind/fast-forward), progress indicators with a start point, and any icon representing a directional action. Icons without directionality should not flip: logos (they're brand assets), symmetric icons (warning triangle, heart, star), clock/time icons (clockwise direction is universal), and numeric content (phone numbers should use `dir="ltr"` even in RTL contexts because digit sequence is LTR-ordered). The test: does the icon's meaning change if mirrored? If yes, flip it in RTL.
