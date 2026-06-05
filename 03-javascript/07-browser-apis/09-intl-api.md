# Intl API

## The Idea

**In plain English:** The Intl API is JavaScript's built-in toolkit for displaying text, numbers, and dates in the way people from different countries actually expect to see them. For example, the number one million looks like "1,000,000" in the US but "1.000.000" in Germany — Intl handles all of that for you automatically.

**Real-world analogy:** Imagine a post office that automatically reformats addresses before delivery — it knows that in Japan the postal code comes first, in the US the state comes last, and in the UK you write the county differently. You hand it a raw address and it figures out the correct local format based on the destination country.

- The post office = the `Intl` object (the central formatting system)
- The destination country = the locale string you pass in (e.g. `'en-US'`, `'de-DE'`)
- The raw address = the raw number, date, or text your code provides
- The correctly reformatted address = the final string `Intl` produces for the user

---

## Overview

The `Intl` object is JavaScript's built-in internationalization namespace. It provides locale-sensitive formatting for numbers, dates, times, relative time, lists, and string comparison — without requiring any third-party library. Every major browser and Node.js fully supports it.

Senior engineers are expected to reach for `Intl` before installing `moment`, `date-fns` formatters, or `numeral.js`.

---

## Intl.DateTimeFormat

Formats dates and times according to locale and calendar conventions.

```javascript
const date = new Date('2024-06-15T14:30:00Z');

// Basic locale formatting
new Intl.DateTimeFormat('en-US').format(date);  // "6/15/2024"
new Intl.DateTimeFormat('en-GB').format(date);  // "15/06/2024"
new Intl.DateTimeFormat('de-DE').format(date);  // "15.6.2024"
new Intl.DateTimeFormat('ja-JP').format(date);  // "2024/6/15"

// With options
new Intl.DateTimeFormat('en-US', {
  weekday: 'long',
  year: 'numeric',
  month: 'long',
  day: 'numeric',
}).format(date);
// "Saturday, June 15, 2024"

new Intl.DateTimeFormat('en-US', {
  dateStyle: 'full',
  timeStyle: 'short',
  timeZone: 'America/New_York',
}).format(date);
// "Saturday, June 15, 2024 at 10:30 AM"

// formatToParts — for custom rendering
new Intl.DateTimeFormat('en-US', {
  month: 'long', day: 'numeric', year: 'numeric',
}).formatToParts(date);
// [
//   { type: 'month', value: 'June' },
//   { type: 'literal', value: ' ' },
//   { type: 'day', value: '15' },
//   { type: 'literal', value: ', ' },
//   { type: 'year', value: '2024' },
// ]

// formatRange — date ranges
const formatter = new Intl.DateTimeFormat('en-US', {
  month: 'long', day: 'numeric',
});
const start = new Date('2024-06-01');
const end   = new Date('2024-06-15');
formatter.formatRange(start, end);  // "June 1 – 15"
```

**Performance tip:** Constructing `Intl.DateTimeFormat` is expensive. Create one instance and reuse it:

```javascript
// Bad — creates a new instance on every call
function formatDate(date) {
  return new Intl.DateTimeFormat('en-US').format(date);
}

// Good — reuse the instance
const dateFormatter = new Intl.DateTimeFormat('en-US', {
  dateStyle: 'medium',
});
function formatDate(date) {
  return dateFormatter.format(date);
}
```

---

## Intl.NumberFormat

Formats numbers with locale-appropriate separators, currency symbols, and units.

```javascript
// Basic number formatting
new Intl.NumberFormat('en-US').format(1234567.89);  // "1,234,567.89"
new Intl.NumberFormat('de-DE').format(1234567.89);  // "1.234.567,89"
new Intl.NumberFormat('hi-IN').format(1234567.89);  // "12,34,567.89"

// Currency
new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
}).format(9.99);
// "$9.99"

new Intl.NumberFormat('de-DE', {
  style: 'currency',
  currency: 'EUR',
}).format(9.99);
// "9,99 €"

new Intl.NumberFormat('ja-JP', {
  style: 'currency',
  currency: 'JPY',
}).format(999);
// "￥999"

// Percent
new Intl.NumberFormat('en-US', {
  style: 'percent',
  minimumFractionDigits: 1,
}).format(0.1234);
// "12.3%"

// Units
new Intl.NumberFormat('en-US', {
  style: 'unit',
  unit: 'kilometer-per-hour',
  unitDisplay: 'short',
}).format(120);
// "120 km/h"

new Intl.NumberFormat('en-US', {
  style: 'unit',
  unit: 'megabyte',
}).format(1024);
// "1,024 MB"

// Compact notation
new Intl.NumberFormat('en-US', {
  notation: 'compact',
  compactDisplay: 'short',
}).format(1_500_000);
// "1.5M"

new Intl.NumberFormat('en-US', {
  notation: 'compact',
}).format(15_000);
// "15K"

// Significant digits
new Intl.NumberFormat('en-US', {
  maximumSignificantDigits: 3,
}).format(1234.56);
// "1,230"

// formatToParts
new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })
  .formatToParts(9.99);
// [
//   { type: 'currency', value: '$' },
//   { type: 'integer', value: '9' },
//   { type: 'decimal', value: '.' },
//   { type: 'fraction', value: '99' },
// ]
```

---

## Intl.RelativeTimeFormat

Human-readable relative time: "3 days ago", "in 2 hours".

```javascript
const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });

rtf.format(-1, 'day');    // "yesterday"
rtf.format(0, 'day');     // "today"
rtf.format(1, 'day');     // "tomorrow"
rtf.format(-7, 'day');    // "7 days ago"
rtf.format(3, 'month');   // "in 3 months"
rtf.format(-2, 'year');   // "2 years ago"
rtf.format(1, 'hour');    // "in 1 hour"
rtf.format(-30, 'minute');// "30 minutes ago"

// numeric: 'always' forces numeric output even for yesterday/tomorrow
const rtf2 = new Intl.RelativeTimeFormat('en', { numeric: 'always' });
rtf2.format(-1, 'day');   // "1 day ago"

// Different locales
new Intl.RelativeTimeFormat('fr', { numeric: 'auto' }).format(-1, 'day');
// "hier"

new Intl.RelativeTimeFormat('ja', { numeric: 'auto' }).format(3, 'month');
// "3か月後"

// Practical helper
function timeAgo(date) {
  const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });
  const diffMs = date - Date.now();
  const units = [
    ['year',   365 * 24 * 60 * 60 * 1000],
    ['month',  30  * 24 * 60 * 60 * 1000],
    ['week',   7   * 24 * 60 * 60 * 1000],
    ['day',    24  * 60 * 60 * 1000],
    ['hour',   60  * 60 * 1000],
    ['minute', 60  * 1000],
    ['second', 1000],
  ];
  for (const [unit, ms] of units) {
    if (Math.abs(diffMs) >= ms) {
      return rtf.format(Math.round(diffMs / ms), unit);
    }
  }
  return 'just now';
}

timeAgo(new Date(Date.now() - 3 * 60 * 60 * 1000)); // "3 hours ago"
timeAgo(new Date(Date.now() + 2 * 24 * 60 * 60 * 1000)); // "in 2 days"
```

---

## Intl.Collator

Locale-aware string comparison and sorting. Critical because `Array.sort()` uses Unicode code points by default, which produces wrong order for non-ASCII strings.

```javascript
// Wrong: default sort is code-point order
['ä', 'z', 'a'].sort();
// ['a', 'z', 'ä'] — ä sorted after z in some locales

// Correct: locale-aware sort
const collator = new Intl.Collator('de');
['ä', 'z', 'a'].sort(collator.compare);
// ['a', 'ä', 'z'] — German alphabetical order

// Case-insensitive sort
const ci = new Intl.Collator('en', { sensitivity: 'base' });
['Banana', 'apple', 'Cherry'].sort(ci.compare);
// ['apple', 'Banana', 'Cherry']

// Numeric sort — "10" after "9"
const numeric = new Intl.Collator('en', { numeric: true });
['file10', 'file2', 'file1'].sort(numeric.compare);
// ['file1', 'file2', 'file10']

// compare() returns -1 | 0 | 1 — drop-in for sort callbacks
const words = ['café', 'cache', 'camel'];
words.sort(new Intl.Collator('fr').compare);
// Correct French alphabetical order

// sensitivity options:
// 'base'     — 'a' === 'Á' (ignore case + accents)
// 'accent'   — 'a' !== 'á' but 'a' === 'A'
// 'case'     — 'a' !== 'A' but 'a' === 'á'
// 'variant'  — 'a' !== 'á' !== 'A' (default)
```

---

## Intl.ListFormat

Formats arrays as human-readable lists with locale-appropriate conjunctions.

```javascript
const lf = new Intl.ListFormat('en', { style: 'long', type: 'conjunction' });
lf.format(['Alice', 'Bob', 'Carol']);
// "Alice, Bob, and Carol"

lf.format(['Alice', 'Bob']);
// "Alice and Bob"

new Intl.ListFormat('en', { type: 'disjunction' }).format(['cat', 'dog', 'fish']);
// "cat, dog, or fish"

new Intl.ListFormat('en', { style: 'narrow', type: 'unit' }).format(['100m', '20s']);
// "100m 20s"

new Intl.ListFormat('fr', { type: 'conjunction' }).format(['lundi', 'mercredi', 'vendredi']);
// "lundi, mercredi et vendredi"

// formatToParts
new Intl.ListFormat('en').formatToParts(['a', 'b', 'c']);
// [
//   { type: 'element', value: 'a' },
//   { type: 'literal', value: ', ' },
//   { type: 'element', value: 'b' },
//   { type: 'literal', value: ', and ' },
//   { type: 'element', value: 'c' },
// ]
```

---

## Intl.PluralRules

Selects the correct plural form for a number (critical for languages with more than 2 plural forms).

```javascript
const pr = new Intl.PluralRules('en');
pr.select(0);  // "other" → "0 items"
pr.select(1);  // "one"   → "1 item"
pr.select(2);  // "other" → "2 items"

// Arabic has 6 plural categories
const ar = new Intl.PluralRules('ar');
ar.select(0);  // "zero"
ar.select(1);  // "one"
ar.select(2);  // "two"
ar.select(6);  // "few"
ar.select(18); // "many"
ar.select(100);// "other"

// Practical use
const messages = {
  en: {
    one: '1 notification',
    other: '{n} notifications',
  },
};

function formatNotifications(n, locale = 'en') {
  const pr = new Intl.PluralRules(locale);
  const rule = pr.select(n);
  const template = messages[locale]?.[rule] ?? messages[locale].other;
  return template.replace('{n}', n);
}

formatNotifications(1);  // "1 notification"
formatNotifications(5);  // "5 notifications"
```

---

## Intl.Segmenter

Splits strings into grapheme clusters, words, or sentences — correctly handling Unicode and emoji sequences.

```javascript
// String.length is WRONG for multi-codepoint sequences
'👨‍👩‍👧‍👦'.length;      // 11 — wrong!

// Grapheme segmenter gives correct count
const segmenter = new Intl.Segmenter('en', { granularity: 'grapheme' });
[...segmenter.segment('👨‍👩‍👧‍👦')].length; // 1 — correct!

[...segmenter.segment('café')].map(s => s.segment);
// ['c', 'a', 'f', 'é'] — 4 graphemes even if 'é' is 2 code points

// Word segmentation (locale-aware — no spaces in Japanese/Chinese)
const wordSeg = new Intl.Segmenter('ja', { granularity: 'word' });
[...wordSeg.segment('日本語のテキスト')].map(s => s.segment);
// ['日本語', 'の', 'テキスト'] — correctly splits Japanese

// Sentence segmentation
const sentSeg = new Intl.Segmenter('en', { granularity: 'sentence' });
[...sentSeg.segment('Hello world. How are you? Fine.')].map(s => s.segment);
// ['Hello world. ', 'How are you? ', 'Fine.']
```

---

## Locale Detection and Negotiation

```javascript
// User's preferred locales
navigator.language;      // "en-US"
navigator.languages;     // ["en-US", "en", "de"]

// Best match from supported locales
Intl.LocaleMatcher ??
Intl.getCanonicalLocales(['EN-US', 'fr-fr']);
// ['en-US', 'fr-FR'] — normalized

// Check what Intl supports
Intl.DateTimeFormat.supportedLocalesOf(['de-AT', 'xyz-999', 'en-US']);
// ['de-AT', 'en-US'] — xyz-999 unsupported

// Practical locale resolution helper
function resolveLocale(requested, supported, fallback = 'en') {
  const matches = Intl.DateTimeFormat.supportedLocalesOf(requested);
  return matches[0] ?? fallback;
}
```

---

## React / Angular Integration Patterns

```typescript
// React — memoize formatters
import { useMemo } from 'react';

function useDateFormatter(locale: string, options?: Intl.DateTimeFormatOptions) {
  return useMemo(
    () => new Intl.DateTimeFormat(locale, options),
    [locale, JSON.stringify(options)]
  );
}

function useCurrencyFormatter(locale: string, currency: string) {
  return useMemo(
    () => new Intl.NumberFormat(locale, { style: 'currency', currency }),
    [locale, currency]
  );
}

// Angular — provide via service
import { Injectable, inject, LOCALE_ID } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class IntlService {
  private locale = inject(LOCALE_ID); // Angular's locale token

  formatCurrency(value: number, currency: string): string {
    return new Intl.NumberFormat(this.locale, {
      style: 'currency',
      currency,
    }).format(value);
  }

  formatDate(date: Date, options?: Intl.DateTimeFormatOptions): string {
    return new Intl.DateTimeFormat(this.locale, options).format(date);
  }

  timeAgo(date: Date): string {
    const rtf = new Intl.RelativeTimeFormat(this.locale, { numeric: 'auto' });
    const diffMs = date.getTime() - Date.now();
    const units: [Intl.RelativeTimeFormatUnit, number][] = [
      ['year',   365 * 24 * 3600 * 1000],
      ['month',  30  * 24 * 3600 * 1000],
      ['day',    24  * 3600 * 1000],
      ['hour',   3600 * 1000],
      ['minute', 60 * 1000],
    ];
    for (const [unit, ms] of units) {
      if (Math.abs(diffMs) >= ms) return rtf.format(Math.round(diffMs / ms), unit);
    }
    return rtf.format(0, 'second');
  }
}
```

---

## Common Pitfalls

1. **Creating formatters in a loop** — Construct once, reuse. `new Intl.DateTimeFormat()` parses locale data.
2. **Using `Date.toLocaleDateString()` directly** — It calls `Intl.DateTimeFormat` under the hood on every call, no caching.
3. **Ignoring timezone** — Always specify `timeZone` if your users are in multiple zones.
4. **`String.prototype.localeCompare` vs `Intl.Collator`** — `localeCompare` creates a new collator every call. Use `Intl.Collator.compare` for sorting large arrays.
5. **`navigator.language` in SSR** — It doesn't exist on the server. Accept a `locale` prop or read from HTTP `Accept-Language` header instead.

---

## Interview Questions

**Q: Why shouldn't you use `arr.sort()` for locale-sensitive string sorting?**  
A: `Array.prototype.sort` uses Unicode code-point order by default. For non-ASCII strings (accented characters, non-Latin scripts) this produces linguistically incorrect results. `new Intl.Collator(locale).compare` handles accent folding, case sensitivity, and numeric ordering correctly.

**Q: What's the difference between `Intl.DateTimeFormat` and `Date.toLocaleDateString()`?**  
A: `toLocaleDateString()` is a shorthand that constructs a new `Intl.DateTimeFormat` instance on every call — it's expensive. When formatting many dates, create one `Intl.DateTimeFormat` and reuse `.format()`.

**Q: How would you implement "3 hours ago" without a library?**  
A: Use `Intl.RelativeTimeFormat` with `numeric: 'auto'`. Calculate the diff in milliseconds, walk down units (year → month → day → hour → minute → second) until you find the largest that fits, then call `rtf.format(Math.round(diff / unitMs), unit)`.

**Q: How does `Intl.PluralRules` help with i18n?**  
A: English has 2 plural forms (one/other). Russian has 4, Arabic has 6. `Intl.PluralRules.select(n)` returns the correct CLDR plural category for a number in a given locale, letting you select the right message template without hard-coding language-specific rules.
