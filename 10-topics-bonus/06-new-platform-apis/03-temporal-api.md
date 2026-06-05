# TC39 Temporal API

## The Idea

**In plain English:** The Temporal API is a new, built-in JavaScript tool for working with dates and times — things like "what day is three months from today?" or "what time is it right now in Tokyo?" It replaces the old, buggy `Date` system that JavaScript has used since 1995.

**Real-world analogy:** Imagine you are a flight dispatcher managing arrivals across multiple airports worldwide. You have a giant wall of clocks — each one showing the local time for a different city. When a pilot asks "when do I land in Tokyo if I depart New York at 2pm?", you don't just count hours blindly; you look up each city's clock, account for daylight saving time changes mid-flight, and give an exact local arrival time.

- The wall of clocks = `Temporal.ZonedDateTime` (a date and time that knows its timezone)
- Each city's local clock = a named timezone like `America/New_York` or `Asia/Tokyo`
- The raw number of hours the flight takes = `Temporal.Duration` (a length of time, with no timezone attached)

---

## Overview

The `Date` object has been broken since JavaScript's creation in 1995. It was modeled after Java's `java.util.Date` class (which Java itself later deprecated) and carries 30 years of accumulated design mistakes. The TC39 Temporal proposal is a complete replacement — a comprehensive, immutable, timezone-aware date and time API that eliminates the entire class of bugs that `Date` produces.

```text
Date's fundamental problems:
  1. Mutability — Date objects can be modified after creation
     const d = new Date(); d.setMonth(d.getMonth() + 1); // mutates in place

  2. Implicit local timezone — most methods use the host timezone silently
     new Date('2024-03-10').getDate() → may return 9 or 10 depending on timezone

  3. Month indexing — months are 0-indexed, days are 1-indexed
     new Date(2024, 0, 1) → January 1st (month 0 = January)

  4. No concept of "date without time" or "time without date"
     Date always includes both — representing "today's date" correctly requires hacks

  5. No duration arithmetic — adding 1 month is not reliably possible
     March 31 + 1 month = April 31? April 30? The language has no answer.

  6. Parsing is implementation-defined
     new Date('2024-01') → varies across browsers

  7. No timezone support — only UTC and local timezone, no named timezone operations
```

## Temporal's Type System

Temporal introduces distinct types for different temporal concepts — each representing exactly what you need, nothing more:

```text
Temporal type hierarchy:

Plain types (no timezone, no UTC offset):
  Temporal.PlainDate        — a calendar date: 2024-03-15
  Temporal.PlainTime        — a wall-clock time: 14:30:00
  Temporal.PlainDateTime    — date + time, no timezone: 2024-03-15T14:30:00
  Temporal.PlainYearMonth   — a month in a year: 2024-03
  Temporal.PlainMonthDay    — a month/day without year: --03-15 (birthdays, holidays)

Absolute / zoned types:
  Temporal.Instant          — absolute point in time (like Unix timestamp), no timezone
  Temporal.ZonedDateTime    — full date+time with timezone: 2024-03-15T14:30:00[America/New_York]

Duration:
  Temporal.Duration         — a length of time: P1Y2M3DT4H5M6S (ISO 8601)

Now:
  Temporal.Now              — factory for current time in various types
```

## Installation and Browser Support

```typescript
// As of 2025: Temporal is Stage 3 (TC39), not yet in all browsers
// Use the polyfill for production code:
// npm install @js-temporal/polyfill

import { Temporal } from '@js-temporal/polyfill';

// Native browser support (status as of mid-2025):
// Chrome: behind flag
// Firefox: in development
// Safari: in development
// Node.js: not yet native

// Feature detection:
const hasTemporal = typeof globalThis.Temporal !== 'undefined';
```

## Temporal.PlainDate — Dates Without Time

```typescript
import { Temporal } from '@js-temporal/polyfill';

// Create a date — no timezone ambiguity
const date = Temporal.PlainDate.from('2024-03-15');
console.log(date.year);   // 2024
console.log(date.month);  // 3 (1-indexed, unlike Date)
console.log(date.day);    // 15
console.log(date.dayOfWeek); // 5 (Friday, ISO: 1=Monday)

// Comparison — no implicit type coercion
const d1 = Temporal.PlainDate.from('2024-01-01');
const d2 = Temporal.PlainDate.from('2024-12-31');
Temporal.PlainDate.compare(d1, d2); // -1 (d1 is earlier)
d1.equals(d2);                      // false

// Arithmetic — returns new objects (immutable)
const nextMonth = date.add({ months: 1 });
const lastYear = date.subtract({ years: 1 });
const in90Days = date.add({ days: 90 });

// March 31 + 1 month = April 30 (handles overflow with `overflow` option)
const march31 = Temporal.PlainDate.from('2024-03-31');
const april = march31.add({ months: 1 });
console.log(april.toString()); // 2024-04-30 (constrained to last day of April)

// Override the overflow behavior
const april31Reject = march31.add({ months: 1 }, { overflow: 'reject' });
// Throws RangeError — April 31 doesn't exist

// Difference between dates
const diff = d2.since(d1, { largestUnit: 'month' });
console.log(diff.months); // 11
console.log(diff.days);   // 30 (remaining days after whole months)

// Convert to different representations
const iso = date.toString();        // '2024-03-15'
const plain = date.toPlainDateTime(Temporal.PlainTime.from('09:00:00'));
```

## Temporal.ZonedDateTime — The Full Picture

```typescript
import { Temporal } from '@js-temporal/polyfill';

// Full timezone-aware date and time
const event = Temporal.ZonedDateTime.from(
  '2024-03-10T09:00:00[America/New_York]'
);

console.log(event.timeZoneId); // 'America/New_York'
console.log(event.hour);       // 9
console.log(event.offset);     // '-05:00' (or '-04:00' after DST)

// DST handling — this is where Temporal shines
// Sunday March 10, 2024 at 2:00am clocks spring forward to 3:00am
const beforeDST = Temporal.ZonedDateTime.from('2024-03-10T01:30:00[America/New_York]');
const afterAdd = beforeDST.add({ hours: 1 });
// With Date: this is ambiguous and often wrong
// With Temporal: correctly handles the DST gap
console.log(afterAdd.toString()); // 2024-03-10T03:30:00-04:00[America/New_York]

// Convert between timezones
const londonTime = event.withTimeZone('Europe/London');
const tokyoTime = event.withTimeZone('Asia/Tokyo');

// "Same wall clock time" in another timezone
const sameLocalTime = event.with({ timeZone: 'Europe/London' });

// Schedule a meeting across timezones
function scheduleMeeting(
  dateTimeInNewYork: string,
  attendeeTimezones: string[]
): Record<string, string> {
  const base = Temporal.ZonedDateTime.from(dateTimeInNewYork);

  return Object.fromEntries(
    attendeeTimezones.map(tz => [
      tz,
      base.withTimeZone(tz).toLocaleString('en', {
        dateStyle: 'full',
        timeStyle: 'short',
        timeZone: tz,
      })
    ])
  );
}

const meeting = scheduleMeeting(
  '2024-06-15T14:00:00[America/New_York]',
  ['Europe/London', 'Asia/Tokyo', 'Australia/Sydney']
);
// {
//   'Europe/London': 'Saturday, June 15, 2024 at 7:00 PM',
//   'Asia/Tokyo': 'Sunday, June 16, 2024 at 3:00 AM',
//   'Australia/Sydney': 'Sunday, June 16, 2024 at 4:00 AM',
// }
```

## Temporal.Instant — Absolute Time

```typescript
import { Temporal } from '@js-temporal/polyfill';

// Instant = absolute moment in time (epoch-based, no timezone)
const now = Temporal.Now.instant();
console.log(now.epochMilliseconds); // 1710500000000 (Unix ms)
console.log(now.epochNanoseconds);  // BigInt — nanosecond precision

// Convert from Date (migration path)
const legacyDate = new Date();
const instant = Temporal.Instant.fromEpochMilliseconds(legacyDate.getTime());

// Convert Instant to ZonedDateTime for display
const inNY = instant.toZonedDateTimeISO('America/New_York');
const inUTC = instant.toZonedDateTimeISO('UTC');

// Duration between two instants
const start = Temporal.Instant.from('2024-01-01T00:00:00Z');
const end = Temporal.Instant.from('2024-12-31T23:59:59Z');
const diff = start.until(end);
console.log(diff.hours); // total hours (no calendar, just seconds/nanoseconds)
```

## Temporal.Duration — Time Arithmetic

```typescript
import { Temporal } from '@js-temporal/polyfill';

// Create durations
const oneMonth = Temporal.Duration.from({ months: 1 });
const twoWeeks = Temporal.Duration.from({ weeks: 2 });
const complex = Temporal.Duration.from('P1Y2M3DT4H5M6.789S'); // ISO 8601

// Duration arithmetic (requires a reference date for calendar durations)
const today = Temporal.Now.plainDateISO();
const threeMonthsLater = today.add({ months: 3 });

// Total duration in a single unit (Instant-based — no calendar ambiguity)
const eventStart = Temporal.Instant.from('2024-03-01T00:00:00Z');
const eventEnd = Temporal.Instant.from('2024-03-10T00:00:00Z');
const duration = eventStart.until(eventEnd, { largestUnit: 'day' });
console.log(duration.days); // 9

// Humanized duration (for UI)
function formatDuration(duration: Temporal.Duration): string {
  const parts: string[] = [];
  if (duration.years) parts.push(`${duration.years}y`);
  if (duration.months) parts.push(`${duration.months}mo`);
  if (duration.days) parts.push(`${duration.days}d`);
  if (duration.hours) parts.push(`${duration.hours}h`);
  if (duration.minutes) parts.push(`${duration.minutes}m`);
  return parts.join(' ') || '0m';
}
```

## Temporal.Now

```typescript
import { Temporal } from '@js-temporal/polyfill';

// All "now" operations go through Temporal.Now
const nowInstant = Temporal.Now.instant();
const nowZoned = Temporal.Now.zonedDateTimeISO(); // System timezone
const nowDate = Temporal.Now.plainDateISO();
const nowTime = Temporal.Now.plainTimeISO();
const nowInTokyo = Temporal.Now.zonedDateTimeISO('Asia/Tokyo');
const systemTz = Temporal.Now.timeZoneId(); // e.g., 'America/New_York'
```

## Migrating from date-fns / Moment / Luxon

```typescript
// ─── Date arithmetic ───────────────────────────────────────────────────────

// Moment.js (deprecated)
moment('2024-01-15').add(1, 'month').format('YYYY-MM-DD');

// date-fns
import { addMonths, format } from 'date-fns';
format(addMonths(new Date('2024-01-15'), 1), 'yyyy-MM-dd');

// Luxon
DateTime.fromISO('2024-01-15').plus({ months: 1 }).toISODate();

// Temporal
Temporal.PlainDate.from('2024-01-15').add({ months: 1 }).toString();

// ─── Timezone conversion ────────────────────────────────────────────────────

// Moment-timezone
moment.tz('2024-03-15 14:00', 'America/New_York').tz('Asia/Tokyo').format();

// Luxon
DateTime.fromISO('2024-03-15T14:00:00', { zone: 'America/New_York' })
  .setZone('Asia/Tokyo').toISO();

// Temporal
Temporal.ZonedDateTime.from('2024-03-15T14:00:00[America/New_York]')
  .withTimeZone('Asia/Tokyo').toString();

// ─── Parsing ────────────────────────────────────────────────────────────────

// Moment.js (unreliable for non-ISO strings)
moment('March 15, 2024', 'MMMM D, YYYY');

// date-fns
import { parse } from 'date-fns';
parse('March 15, 2024', 'MMMM d, yyyy', new Date());

// Temporal — ISO 8601 only (by design — locale parsing is unreliable)
Temporal.PlainDate.from('2024-03-15');
// For locale strings: use Intl.DateTimeFormat to parse then convert
```

## Comparison Table

```text
Feature                       | Date     | Moment  | date-fns | Luxon   | Temporal
──────────────────────────────|──────────|─────────|──────────|─────────|─────────
Immutable                     | No       | No      | Yes      | Yes     | Yes
Timezone support              | UTC only | Plugin  | Plugin   | Built-in| Built-in
Named timezones               | No       | Plugin  | No       | Built-in| Built-in
DST-aware arithmetic          | No       | Limited | Limited  | Yes     | Yes
"Date only" type              | No       | No      | No       | No      | Yes
"Time only" type              | No       | No      | No       | No      | Yes
Duration type                 | No       | No      | No       | Yes     | Yes
Calendar systems              | No       | Plugin  | No       | No      | Yes
Nanosecond precision          | No       | No      | No       | No      | Yes
Tree-shakeable                | N/A      | No      | Yes      | Yes     | Native
Bundle size (min+gz)          | N/A      | 16KB    | 12KB     | 23KB    | 0 (native)
Native browser support (2025) | Yes      | N/A     | N/A      | N/A     | Partial
```

## Real-World Patterns

### Recurring Event Scheduling

```typescript
import { Temporal } from '@js-temporal/polyfill';

// Generate dates for a recurring event (every 2 weeks for 6 months)
function* generateRecurringDates(
  start: Temporal.PlainDate,
  intervalDays: number,
  endDate: Temporal.PlainDate
): Generator<Temporal.PlainDate> {
  let current = start;
  while (Temporal.PlainDate.compare(current, endDate) <= 0) {
    yield current;
    current = current.add({ days: intervalDays });
  }
}

const start = Temporal.PlainDate.from('2024-04-01');
const end = start.add({ months: 6 });
const biweeklyDates = [...generateRecurringDates(start, 14, end)];
```

### Age Calculation

```typescript
function calculateAge(
  birthDate: string,
  asOf: Temporal.PlainDate = Temporal.Now.plainDateISO()
): { years: number; months: number; days: number } {
  const birth = Temporal.PlainDate.from(birthDate);
  const diff = birth.until(asOf, { largestUnit: 'year' });

  return {
    years: diff.years,
    months: diff.months,
    days: diff.days,
  };
}

const age = calculateAge('1990-07-15');
// { years: 33, months: 8, days: 12 }
```

### Business Days Calculator

```typescript
function addBusinessDays(
  start: Temporal.PlainDate,
  days: number
): Temporal.PlainDate {
  let current = start;
  let added = 0;

  while (added < days) {
    current = current.add({ days: 1 });
    const dayOfWeek = current.dayOfWeek; // 1=Monday, 7=Sunday
    if (dayOfWeek < 6) { // Mon–Fri
      added++;
    }
  }

  return current;
}

const deadline = addBusinessDays(Temporal.Now.plainDateISO(), 10);
```

### Countdowns and Timers

```typescript
function getCountdown(targetISO: string): {
  days: number;
  hours: number;
  minutes: number;
  seconds: number;
  total: Temporal.Duration;
} {
  const target = Temporal.Instant.from(targetISO);
  const now = Temporal.Now.instant();
  const diff = now.until(target, { largestUnit: 'day' });

  return {
    days: diff.days,
    hours: diff.hours,
    minutes: diff.minutes,
    seconds: diff.seconds,
    total: diff,
  };
}

const countdown = getCountdown('2024-12-31T23:59:59Z');
```

## Formatting with Intl

Temporal types integrate with `Intl.DateTimeFormat`:

```typescript
const dt = Temporal.ZonedDateTime.from('2024-03-15T14:30:00[America/New_York]');

// Convert to legacy Date for Intl formatting (until native support lands)
const legacyDate = new Date(dt.epochMilliseconds);

const formatter = new Intl.DateTimeFormat('en-US', {
  dateStyle: 'full',
  timeStyle: 'long',
  timeZone: dt.timeZoneId,
});

formatter.format(legacyDate);
// 'Friday, March 15, 2024 at 2:30:00 PM Eastern Daylight Time'

// Relative time
const now = Temporal.Now.instant();
const past = now.subtract({ hours: 3 });
const rtf = new Intl.RelativeTimeFormat('en', { style: 'long' });
rtf.format(-3, 'hour'); // '3 hours ago'
```

## Interview Questions

1. **What are the five most critical design flaws in JavaScript's `Date` object?**
   (1) Mutability — `Date` objects can be modified in place via `setMonth`, `setDate`, etc., causing bugs when objects are passed to functions. (2) 0-indexed months — `new Date(2024, 0, 1)` is January 1st, an API inconsistency with no justification. (3) Implicit local timezone — `new Date('2024-03-15').getDate()` returns different values in different timezones, making "date without time" operations unreliable. (4) No proper duration type — adding 1 month to March 31st has no reliable answer (Date silently overflows to April 30 or May 1 depending on how you do it). (5) No distinct types — there's no way to represent "just a date" or "just a time" — everything is a full timestamp, forcing workarounds like stripping time components.

2. **Explain the difference between `Temporal.PlainDate`, `Temporal.Instant`, and `Temporal.ZonedDateTime`.**
   `Temporal.PlainDate` is a date without any time or timezone — representing "March 15, 2024" as a calendar concept, like a birthday or appointment date. It has no relationship to UTC or wall-clock time. `Temporal.Instant` is an absolute point in time, equivalent to a Unix timestamp but with nanosecond precision — it has no timezone, month, or year in the calendar sense; it's just a distance from the epoch. `Temporal.ZonedDateTime` combines an Instant with a named timezone and calendar, giving you a full date-time that respects DST transitions, timezone rules, and calendar arithmetic.

3. **Why is `Temporal.PlainDate` necessary — why can't you just use a `Date` set to midnight UTC?**
   The "midnight UTC trick" is fundamentally wrong because midnight UTC is not the same moment in most timezones — it's already the previous or next day in many places. `new Date('2024-03-15')` set to midnight UTC represents March 14th at 7pm in New York. When you call `.getDate()` in New York's timezone, you get 14, not 15. `PlainDate` has no time component at all — it cannot be converted to a UTC instant without specifying a timezone and time. It is genuinely timezone-agnostic, making it the correct type for dates that represent calendar concepts (birthdays, deadlines, business dates) not moments in time.

4. **How does Temporal handle DST transitions, and why does `Date` get them wrong?**
   When you add 1 day to a `Date` by adding 86400000 milliseconds, you're adding absolute time — you'll land at the wrong wall-clock time after a DST transition (23:00 or 25:00 instead of 00:00). `Temporal.ZonedDateTime.add({ days: 1 })` uses calendar arithmetic: it adds one calendar day by looking up the target date in the timezone database, calculating the correct UTC offset for that date, and returning the correct wall-clock time. If adding hours causes a transition, Temporal produces the correct result by advancing through the DST gap — e.g., adding 1 hour to 1:30am on spring-forward day gives 3:30am, not 2:30am.

5. **What is the `overflow: 'constrain'` vs `overflow: 'reject'` distinction in Temporal date arithmetic?**
   When arithmetic produces an invalid date (e.g., adding 1 month to January 31 gives "February 31"), Temporal needs a disambiguation strategy. `overflow: 'constrain'` (the default) clamps the result to the last valid day of the month — January 31 + 1 month = February 28 or 29. `overflow: 'reject'` throws a `RangeError` — the operation is disallowed because the result is invalid. Choose based on domain semantics: for scheduling (subscription renews on the 31st, bill on the 31st), `constrain` is usually correct. For strict date arithmetic where February 31st should never be created, `reject` makes the invalid operation visible.

6. **How would you migrate a codebase from date-fns to Temporal?**
   The migration strategy: (1) Start at the data boundary — database dates come in as ISO strings, parse them with `Temporal.PlainDate.from()` or `Temporal.Instant.from()` instead of `new Date()`. (2) Replace arithmetic functions: `addDays(d, 7)` becomes `d.add({ days: 7 })`. (3) Replace comparison functions: `isBefore(a, b)` becomes `Temporal.PlainDate.compare(a, b) < 0`. (4) Replace formatting by converting to `Date` for `Intl.DateTimeFormat` until native Temporal formatting lands. (5) For timezone operations, replace `date-fns-tz` with `Temporal.ZonedDateTime`. (6) Use the `@js-temporal/polyfill` package, write all new code with Temporal, and progressively replace old date-fns code. No need to migrate everything at once.

7. **What is the TC39 stage process and what does Stage 3 mean for Temporal's production readiness?**
   TC39 stages: Stage 0 (strawperson), Stage 1 (proposal), Stage 2 (draft spec), Stage 2.7 (spec complete), Stage 3 (candidate — spec final, awaiting implementation), Stage 4 (finished — in the ECMAScript standard). Stage 3 means the specification is complete and frozen — implementations may begin. Browser vendors implement Stage 3 proposals, and the proposal advances to Stage 4 once two independent implementations exist. For production readiness: the polyfill (`@js-temporal/polyfill`) is production-quality and used in production today. Native support is landing progressively (Chrome 127+ behind flag as of 2025). Teams can use Temporal today via the polyfill with zero migration cost when native support arrives.

8. **Describe a scenario where using `Date` instead of `Temporal.ZonedDateTime` would cause a production bug.**
   Scenario: A global SaaS app stores "next renewal date" as a UTC timestamp. A customer in New Zealand signs up on July 31. Their renewal is "August 31". The server calculates: `new Date('2024-07-31').setMonth(new Date('2024-07-31').getMonth() + 1)`. In UTC this is correct. But the customer signed up at 11pm on July 31 NZT, which is July 30 UTC. The stored "July 30 UTC" renewal date, displayed in NZT, shows as "August 30 NZT" — one day early. With Temporal: store the renewal as a `PlainDate` ('2024-08-31') because it's a calendar concept not a timestamp. Display it by converting `PlainDate` to the customer's timezone at business hours. No UTC offset error, no DST confusion.
