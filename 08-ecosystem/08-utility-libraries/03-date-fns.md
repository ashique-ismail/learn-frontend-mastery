# date-fns: Modern JavaScript Date Utility Library

## Overview

date-fns is a comprehensive, modular date manipulation library that provides over 200 functions for parsing, formatting, calculating, and manipulating dates in JavaScript. Created as a modern alternative to Moment.js, date-fns emphasizes immutability, functional programming, and tree-shakable architecture.

**Key Features:**
- Immutable and pure functions (no date mutation)
- Modular design for optimal tree shaking
- Comprehensive date manipulation (200+ functions)
- Simple, consistent API
- Timezone support via date-fns-tz
- Internationalization (i18n) for 100+ locales
- TypeScript support with full type definitions
- No prototype pollution
- Works with native Date objects
- Lightweight (only import what you need)

date-fns has become the recommended date library for modern JavaScript applications, replacing Moment.js which is now in maintenance mode.

## Installation and Setup

### Installation

```bash
# Core library
npm install date-fns

# Timezone support
npm install date-fns-tz

# Specific locale (optional, for i18n)
# Locales are included, just import them
```

### Basic Importing

```javascript
// Import specific functions (recommended)
import { format, parseISO, addDays, differenceInDays } from 'date-fns';

// Import locale for formatting
import { format } from 'date-fns';
import { es, fr, de } from 'date-fns/locale';

// Timezone functions
import { zonedTimeToUtc, utcToZonedTime, format as formatTZ } from 'date-fns-tz';

// TypeScript
import { format, parseISO } from 'date-fns';
const formatted: string = format(new Date(), 'yyyy-MM-dd');
```

## Core Concepts

### 1. Immutability

date-fns never mutates date objects:

```javascript
import { addDays, addMonths } from 'date-fns';

const originalDate = new Date(2024, 0, 1); // Jan 1, 2024
const newDate = addDays(originalDate, 7);

console.log(originalDate); // Still Jan 1, 2024
console.log(newDate); // Jan 8, 2024

// Compare to mutable alternative
const mutableDate = new Date(2024, 0, 1);
mutableDate.setDate(mutableDate.getDate() + 7); // Mutates!
```

### 2. Pure Functions

All functions are pure (same input = same output):

```javascript
import { format, addYears } from 'date-fns';

const date = new Date(2024, 0, 1);

// Pure - always returns same result
format(date, 'yyyy-MM-dd'); // '2024-01-01'
format(date, 'yyyy-MM-dd'); // '2024-01-01'

// Pure - creates new date
const future = addYears(date, 5);
console.log(date); // Original unchanged
console.log(future); // 5 years later
```

### 3. Function Composition

date-fns functions work well together:

```javascript
import { addDays, addMonths, startOfWeek, endOfMonth, format } from 'date-fns';

// Compose operations
const date = new Date(2024, 0, 15);
const result = format(
  addDays(
    startOfWeek(date),
    7
  ),
  'yyyy-MM-dd'
);

// Or with intermediate steps
const weekStart = startOfWeek(date);
const nextWeek = addDays(weekStart, 7);
const formatted = format(nextWeek, 'yyyy-MM-dd');
```

## Formatting Dates

### Basic Formatting

```javascript
import { format } from 'date-fns';

const date = new Date(2024, 0, 15, 14, 30, 0); // Jan 15, 2024, 2:30 PM

// Common formats
format(date, 'yyyy-MM-dd'); // '2024-01-15'
format(date, 'MM/dd/yyyy'); // '01/15/2024'
format(date, 'dd.MM.yyyy'); // '15.01.2024'
format(date, 'yyyy-MM-dd HH:mm:ss'); // '2024-01-15 14:30:00'

// Human-readable formats
format(date, 'MMMM do, yyyy'); // 'January 15th, 2024'
format(date, 'EEEE, MMMM do'); // 'Monday, January 15th'
format(date, 'h:mm a'); // '2:30 PM'
format(date, 'PPpp'); // 'Jan 15, 2024, 2:30:00 PM'

// ISO formats
format(date, "yyyy-MM-dd'T'HH:mm:ss.SSSxxx");
// '2024-01-15T14:30:00.000+00:00'
```

### Format Tokens Reference

```javascript
import { format } from 'date-fns';

const date = new Date(2024, 0, 15, 14, 5, 3);

// Year
format(date, 'yy'); // '24' (2-digit)
format(date, 'yyyy'); // '2024' (4-digit)

// Month
format(date, 'M'); // '1' (1-12)
format(date, 'MM'); // '01' (01-12)
format(date, 'MMM'); // 'Jan' (short)
format(date, 'MMMM'); // 'January' (full)

// Day
format(date, 'd'); // '15' (1-31)
format(date, 'dd'); // '15' (01-31)
format(date, 'do'); // '15th' (ordinal)
format(date, 'E'); // 'Mon' (short day)
format(date, 'EEEE'); // 'Monday' (full day)

// Hour
format(date, 'H'); // '14' (0-23)
format(date, 'HH'); // '14' (00-23)
format(date, 'h'); // '2' (1-12)
format(date, 'hh'); // '02' (01-12)

// Minute
format(date, 'm'); // '5' (0-59)
format(date, 'mm'); // '05' (00-59)

// Second
format(date, 's'); // '3' (0-59)
format(date, 'ss'); // '03' (00-59)

// AM/PM
format(date, 'a'); // 'PM'
format(date, 'aa'); // 'PM'
format(date, 'aaa'); // 'pm'
```

### Localized Formatting

```javascript
import { format } from 'date-fns';
import { es, fr, de, ja, zhCN } from 'date-fns/locale';

const date = new Date(2024, 0, 15);

// Spanish
format(date, 'EEEE, MMMM do, yyyy', { locale: es });
// 'lunes, enero 15º, 2024'

// French
format(date, 'EEEE, MMMM do, yyyy', { locale: fr });
// 'lundi, janvier 15e, 2024'

// German
format(date, 'EEEE, d. MMMM yyyy', { locale: de });
// 'Montag, 15. Januar 2024'

// Japanese
format(date, 'yyyy年MM月dd日', { locale: ja });
// '2024年01月15日'

// Chinese
format(date, 'yyyy年MM月dd日 EEEE', { locale: zhCN });
// '2024年01月15日 星期一'
```

## Parsing Dates

### Parsing Strings

```javascript
import { parse, parseISO } from 'date-fns';

// Parse ISO 8601 strings
parseISO('2024-01-15'); // Date object
parseISO('2024-01-15T14:30:00Z'); // With time
parseISO('2024-01-15T14:30:00.000Z'); // With milliseconds

// Parse custom formats
parse('01/15/2024', 'MM/dd/yyyy', new Date());
// Third argument is reference date for relative parsing

parse('15.01.2024', 'dd.MM.yyyy', new Date());
parse('2024-01-15 14:30', 'yyyy-MM-dd HH:mm', new Date());
parse('Jan 15, 2024', 'MMM dd, yyyy', new Date());

// With locale
import { parse } from 'date-fns';
import { es } from 'date-fns/locale';

parse('15 de enero de 2024', 'dd \'de\' MMMM \'de\' yyyy', new Date(), { 
  locale: es 
});
```

### Validation

```javascript
import { isValid, parseISO, parse } from 'date-fns';

// Check if date is valid
isValid(new Date()); // true
isValid(new Date('invalid')); // false
isValid(parseISO('2024-01-15')); // true
isValid(parseISO('not a date')); // false

// Safe parsing
function safeParse(dateString, formatString) {
  const parsed = parse(dateString, formatString, new Date());
  return isValid(parsed) ? parsed : null;
}

safeParse('01/15/2024', 'MM/dd/yyyy'); // Valid date
safeParse('99/99/9999', 'MM/dd/yyyy'); // null
```

## Date Manipulation

### Adding and Subtracting

```javascript
import { 
  addYears, addMonths, addWeeks, addDays, 
  addHours, addMinutes, addSeconds,
  subYears, subMonths, subDays
} from 'date-fns';

const date = new Date(2024, 0, 15); // Jan 15, 2024

// Adding
addYears(date, 2); // Jan 15, 2026
addMonths(date, 3); // Apr 15, 2024
addWeeks(date, 2); // Jan 29, 2024
addDays(date, 7); // Jan 22, 2024
addHours(date, 24); // Jan 16, 2024
addMinutes(date, 30); // Jan 15, 2024 12:30 AM
addSeconds(date, 60); // Jan 15, 2024 12:01 AM

// Subtracting
subYears(date, 1); // Jan 15, 2023
subMonths(date, 6); // Jul 15, 2023
subDays(date, 10); // Jan 5, 2024

// Chaining
import { addDays, addMonths } from 'date-fns';
const future = addDays(addMonths(date, 1), 15);
// Feb 29, 2024 (or 30th depending on month)
```

### Start and End of Period

```javascript
import { 
  startOfDay, endOfDay,
  startOfWeek, endOfWeek,
  startOfMonth, endOfMonth,
  startOfYear, endOfYear
} from 'date-fns';

const date = new Date(2024, 0, 15, 14, 30); // Jan 15, 2024, 2:30 PM

// Day boundaries
startOfDay(date); // Jan 15, 2024, 00:00:00
endOfDay(date); // Jan 15, 2024, 23:59:59.999

// Week boundaries (starts Sunday by default)
startOfWeek(date); // Jan 14, 2024 (Sunday)
endOfWeek(date); // Jan 20, 2024 (Saturday)

// Custom week start (Monday)
import { startOfWeek } from 'date-fns';
startOfWeek(date, { weekStartsOn: 1 }); // Jan 15, 2024 (Monday)

// Month boundaries
startOfMonth(date); // Jan 1, 2024
endOfMonth(date); // Jan 31, 2024

// Year boundaries
startOfYear(date); // Jan 1, 2024
endOfYear(date); // Dec 31, 2024
```

## Date Comparison

### Comparing Dates

```javascript
import { 
  isBefore, isAfter, isEqual,
  isSameDay, isSameWeek, isSameMonth, isSameYear,
  isWithinInterval, compareAsc, compareDesc
} from 'date-fns';

const date1 = new Date(2024, 0, 15);
const date2 = new Date(2024, 0, 20);

// Basic comparison
isBefore(date1, date2); // true
isAfter(date1, date2); // false
isEqual(date1, date2); // false

// Same period checks
isSameDay(date1, date2); // false
isSameWeek(date1, date2); // true (same week)
isSameMonth(date1, date2); // true (both in January)
isSameYear(date1, date2); // true (both in 2024)

// Interval checks
const start = new Date(2024, 0, 1);
const end = new Date(2024, 0, 31);
isWithinInterval(date1, { start, end }); // true

// Sorting
const dates = [
  new Date(2024, 2, 1),
  new Date(2024, 0, 1),
  new Date(2024, 1, 1)
];

dates.sort(compareAsc); // Ascending order
dates.sort(compareDesc); // Descending order
```

### Calculating Differences

```javascript
import { 
  differenceInYears, differenceInMonths, differenceInDays,
  differenceInHours, differenceInMinutes, differenceInSeconds,
  differenceInMilliseconds
} from 'date-fns';

const start = new Date(2024, 0, 1);
const end = new Date(2026, 6, 15);

// Various differences
differenceInYears(end, start); // 2
differenceInMonths(end, start); // 30
differenceInDays(end, start); // 926
differenceInHours(end, start); // 22224

// Precise differences
const now = new Date();
const later = new Date(now.getTime() + 5000); // 5 seconds later

differenceInSeconds(later, now); // 5
differenceInMilliseconds(later, now); // 5000

// Age calculation
import { differenceInYears } from 'date-fns';

function calculateAge(birthDate) {
  return differenceInYears(new Date(), birthDate);
}

calculateAge(new Date(1990, 5, 15)); // Current age
```

## Relative Time (Humanization)

### Distance in Words

```javascript
import { formatDistance, formatDistanceToNow, formatDistanceStrict } from 'date-fns';

const date = new Date(2024, 0, 1);
const now = new Date(2024, 0, 15);

// Relative distance
formatDistance(date, now);
// 'about 2 weeks'

formatDistance(date, now, { addSuffix: true });
// 'about 2 weeks ago'

// From now
formatDistanceToNow(date);
// '2 weeks ago' (if today is Jan 15)

formatDistanceToNow(date, { addSuffix: true });
// '2 weeks ago'

// Strict (no "about")
formatDistanceStrict(date, now);
// '14 days'

formatDistanceStrict(date, now, { unit: 'hour' });
// '336 hours'

// Localized
import { formatDistance } from 'date-fns';
import { es } from 'date-fns/locale';

formatDistance(date, now, { 
  locale: es,
  addSuffix: true
});
// 'hace alrededor de 2 semanas'
```

### Relative Formatting

```javascript
import { formatRelative } from 'date-fns';

const baseDate = new Date(2024, 0, 15); // Jan 15, 2024

// Various dates relative to base
formatRelative(new Date(2024, 0, 15), baseDate);
// 'today at 12:00 AM'

formatRelative(new Date(2024, 0, 14), baseDate);
// 'yesterday at 12:00 AM'

formatRelative(new Date(2024, 0, 16), baseDate);
// 'tomorrow at 12:00 AM'

formatRelative(new Date(2024, 0, 13), baseDate);
// 'last Saturday at 12:00 AM'

formatRelative(new Date(2024, 0, 20), baseDate);
// 'next Saturday at 12:00 AM'

// With locale
import { formatRelative } from 'date-fns';
import { es } from 'date-fns/locale';

formatRelative(new Date(2024, 0, 14), baseDate, { locale: es });
// 'ayer a las 00:00'
```

## Timezone Support

### Basic Timezone Operations

```javascript
import { zonedTimeToUtc, utcToZonedTime, format } from 'date-fns-tz';

// Convert local time to UTC
const localDate = new Date(2024, 0, 15, 14, 30);
const utcDate = zonedTimeToUtc(localDate, 'America/New_York');

// Convert UTC to specific timezone
const utcNow = new Date();
const nyTime = utcToZonedTime(utcNow, 'America/New_York');
const londonTime = utcToZonedTime(utcNow, 'Europe/London');
const tokyoTime = utcToZonedTime(utcNow, 'Asia/Tokyo');

// Format with timezone
format(nyTime, 'yyyy-MM-dd HH:mm:ss zzz', { 
  timeZone: 'America/New_York' 
});
// '2024-01-15 14:30:00 EST'

format(tokyoTime, 'yyyy-MM-dd HH:mm:ss zzz', { 
  timeZone: 'Asia/Tokyo' 
});
// '2024-01-16 04:30:00 JST'
```

### Timezone Conversions

```javascript
import { zonedTimeToUtc, utcToZonedTime, format } from 'date-fns-tz';

// Meeting scheduler example
function scheduleInTimezone(localTime, fromTz, toTz) {
  // Convert to UTC first
  const utcTime = zonedTimeToUtc(localTime, fromTz);
  
  // Then to target timezone
  const targetTime = utcToZonedTime(utcTime, toTz);
  
  return format(targetTime, 'yyyy-MM-dd HH:mm zzz', { timeZone: toTz });
}

const meeting = new Date(2024, 0, 15, 10, 0); // 10 AM local

scheduleInTimezone(meeting, 'America/New_York', 'Europe/London');
// '2024-01-15 15:00 GMT'

scheduleInTimezone(meeting, 'America/New_York', 'Asia/Tokyo');
// '2024-01-16 00:00 JST'
```

## Utility Functions

### Date Queries

```javascript
import { 
  isToday, isTomorrow, isYesterday,
  isThisWeek, isThisMonth, isThisYear,
  isWeekend, isFuture, isPast,
  isFirstDayOfMonth, isLastDayOfMonth,
  isLeapYear
} from 'date-fns';

const date = new Date();

// Relative queries
isToday(date); // true
isTomorrow(new Date(Date.now() + 86400000)); // true
isYesterday(new Date(Date.now() - 86400000)); // true

// Period queries
isThisWeek(date); // true
isThisMonth(date); // true
isThisYear(date); // true

// Properties
isWeekend(date); // true/false based on day
isFuture(new Date(2025, 0, 1)); // true
isPast(new Date(2023, 0, 1)); // true

// Month boundaries
isFirstDayOfMonth(new Date(2024, 0, 1)); // true
isLastDayOfMonth(new Date(2024, 0, 31)); // true

// Leap year
isLeapYear(new Date(2024, 0, 1)); // true
isLeapYear(new Date(2023, 0, 1)); // false
```

### Date Rounding

```javascript
import { 
  roundToNearestMinutes,
  startOfHour, endOfHour,
  startOfMinute, endOfMinute
} from 'date-fns';

const date = new Date(2024, 0, 15, 14, 37, 28);

// Round to nearest interval
roundToNearestMinutes(date); // 14:37:00 (nearest minute)
roundToNearestMinutes(date, { nearestTo: 15 });
// 14:30:00 (nearest 15 minutes)

roundToNearestMinutes(date, { nearestTo: 30 });
// 14:30:00 (nearest 30 minutes)

// Hour boundaries
startOfHour(date); // 14:00:00
endOfHour(date); // 14:59:59.999

// Minute boundaries
startOfMinute(date); // 14:37:00
endOfMinute(date); // 14:37:59.999
```

## Common Patterns

### Date Ranges

```javascript
import { 
  eachDayOfInterval, eachWeekOfInterval, eachMonthOfInterval,
  startOfMonth, endOfMonth, isWithinInterval
} from 'date-fns';

// Generate date range
const start = new Date(2024, 0, 1);
const end = new Date(2024, 0, 7);

eachDayOfInterval({ start, end });
// [Jan 1, Jan 2, ..., Jan 7]

// All weeks in a month
const monthStart = startOfMonth(new Date(2024, 0, 15));
const monthEnd = endOfMonth(new Date(2024, 0, 15));

eachWeekOfInterval({ start: monthStart, end: monthEnd });
// Array of week start dates

// All months in a year
eachMonthOfInterval({ 
  start: new Date(2024, 0, 1),
  end: new Date(2024, 11, 31)
});
// [Jan 2024, Feb 2024, ..., Dec 2024]

// Check if date is in range
const date = new Date(2024, 0, 15);
isWithinInterval(date, { start, end }); // true/false
```

### Calendar Generation

```javascript
import { 
  startOfMonth, endOfMonth, startOfWeek, endOfWeek,
  eachDayOfInterval, format, isSameMonth
} from 'date-fns';

function generateCalendar(date) {
  const monthStart = startOfMonth(date);
  const monthEnd = endOfMonth(date);
  
  const calendarStart = startOfWeek(monthStart);
  const calendarEnd = endOfWeek(monthEnd);
  
  const days = eachDayOfInterval({
    start: calendarStart,
    end: calendarEnd
  });
  
  return days.map(day => ({
    date: day,
    dayNumber: format(day, 'd'),
    isCurrentMonth: isSameMonth(day, date),
    isToday: isToday(day)
  }));
}

const calendar = generateCalendar(new Date(2024, 0, 15));
// Array of 35-42 days (full calendar grid)
```

### Age Calculation

```javascript
import { differenceInYears, differenceInMonths, differenceInDays } from 'date-fns';

function calculateDetailedAge(birthDate) {
  const now = new Date();
  const years = differenceInYears(now, birthDate);
  
  const yearsDate = addYears(birthDate, years);
  const months = differenceInMonths(now, yearsDate);
  
  const monthsDate = addMonths(yearsDate, months);
  const days = differenceInDays(now, monthsDate);
  
  return { years, months, days };
}

calculateDetailedAge(new Date(1990, 5, 15));
// { years: 33, months: 7, days: 15 }
```

## Common Mistakes

### 1. Mutating Dates

```javascript
// Wrong: Mutating date
const date = new Date(2024, 0, 1);
date.setDate(date.getDate() + 7); // Mutates!

// Correct: Use date-fns (immutable)
import { addDays } from 'date-fns';
const date = new Date(2024, 0, 1);
const newDate = addDays(date, 7); // Original unchanged
```

### 2. Incorrect Timezone Handling

```javascript
// Wrong: Ignoring timezones
const date = new Date('2024-01-15'); // Parses as UTC midnight
format(date, 'yyyy-MM-dd'); // Might show Jan 14 in some timezones!

// Correct: Use parseISO or explicit timezone
import { parseISO } from 'date-fns';
const date = parseISO('2024-01-15');

// Or use timezone functions
import { zonedTimeToUtc } from 'date-fns-tz';
const date = zonedTimeToUtc('2024-01-15', 'America/New_York');
```

### 3. Wrong Format Tokens

```javascript
// Wrong: Using incorrect tokens
format(new Date(), 'YYYY-MM-DD'); // ❌ YYYY is week-numbering year

// Correct: Use lowercase yyyy
format(new Date(), 'yyyy-MM-dd'); // ✓ 2024-01-15

// Wrong: DD is day of year, not day of month
format(new Date(), 'yyyy-MM-DD'); // ❌

// Correct: dd is day of month
format(new Date(), 'yyyy-MM-dd'); // ✓
```

### 4. Not Handling Invalid Dates

```javascript
// Wrong: No validation
const date = parseISO(userInput);
format(date, 'yyyy-MM-dd'); // Crashes if invalid!

// Correct: Validate first
import { isValid, parseISO } from 'date-fns';

const date = parseISO(userInput);
if (isValid(date)) {
  format(date, 'yyyy-MM-dd');
} else {
  console.error('Invalid date');
}
```

## Best Practices

### 1. Use Specific Imports

```javascript
// Good: Import only what you need (tree shaking)
import { format, addDays, parseISO } from 'date-fns';

// Avoid: Importing everything
import * as dateFns from 'date-fns'; // Larger bundle
```

### 2. Centralize Date Formatting

```javascript
// Create utility functions for consistent formatting
import { format } from 'date-fns';

export const formatters = {
  short: (date) => format(date, 'MM/dd/yyyy'),
  long: (date) => format(date, 'MMMM do, yyyy'),
  time: (date) => format(date, 'h:mm a'),
  full: (date) => format(date, 'PPpp')
};

// Usage
formatters.short(new Date()); // '01/15/2024'
formatters.long(new Date()); // 'January 15th, 2024'
```

### 3. Handle Timezones Explicitly

```javascript
// Always be explicit about timezones
import { zonedTimeToUtc, format } from 'date-fns-tz';

function displayUserTime(utcDate, userTimezone) {
  return format(
    utcToZonedTime(utcDate, userTimezone),
    'PPpp',
    { timeZone: userTimezone }
  );
}
```

### 4. Use TypeScript for Type Safety

```typescript
import { format, parseISO, isValid } from 'date-fns';

function formatDate(date: Date | string): string {
  const dateObj = typeof date === 'string' ? parseISO(date) : date;
  
  if (!isValid(dateObj)) {
    throw new Error('Invalid date');
  }
  
  return format(dateObj, 'yyyy-MM-dd');
}
```

## Interview Questions

### 1. Why is date-fns preferred over Moment.js?

**Answer:** date-fns is immutable (never mutates dates), modular with tree shaking support (import only what you need), uses native Date objects (no custom wrappers), has a smaller bundle size per function, follows functional programming principles, and is actively maintained. Moment.js is now in maintenance mode, has a large bundle size (even with minimal usage), uses mutable operations that can cause bugs, and wraps dates in custom objects. date-fns is the modern, recommended choice.

### 2. How does date-fns handle timezones?

**Answer:** date-fns core works with local time and doesn't handle timezones directly. The separate date-fns-tz package provides timezone support through `zonedTimeToUtc` and `utcToZonedTime` functions. It uses the IANA timezone database and relies on the browser's Intl API. For timezone-aware applications, always convert to UTC for storage/transmission, then convert to user's timezone for display. This separation keeps the core library lightweight while providing timezone support when needed.

### 3. What's the difference between parse and parseISO?

**Answer:** `parseISO` specifically parses ISO 8601 date strings (e.g., '2024-01-15T14:30:00Z') and is optimized for this common format. `parse` is more general, accepting custom format strings as templates (e.g., 'MM/dd/yyyy') but requires a reference date as the third parameter for relative values. Use parseISO for API responses and ISO strings; use parse for user input with custom formats. parseISO is simpler when working with standard formats.

### 4. How do you ensure date immutability with date-fns?

**Answer:** date-fns functions never mutate input dates; they always return new Date objects. Functions like `addDays`, `setHours`, `startOfMonth` create new dates rather than modifying originals. This prevents bugs from unexpected mutations and enables functional programming patterns. In contrast, native Date methods like `setDate()` mutate in place. Always use date-fns functions for transformations rather than native mutating methods to maintain immutability.

### 5. What are the performance implications of using date-fns?

**Answer:** date-fns has minimal performance overhead compared to native operations. The modular design allows tree shaking, so you only bundle functions you use (typically 2-5KB per function). Creating new Date objects (immutability) has negligible cost in modern JavaScript engines. For high-frequency operations (thousands per second), native Date methods may be marginally faster, but date-fns provides better developer experience and fewer bugs. Always measure actual performance in your use case.

### 6. How do you handle internationalization with date-fns?

**Answer:** Import locale objects and pass them to formatting functions via the `locale` option. date-fns includes 100+ locales in the package. Example: `format(date, 'PPP', { locale: es })` for Spanish. Locales affect month names, day names, and relative time descriptions. For dynamic locales, import all needed locales and select based on user preferences. The locale system is lightweight since you only import locales you use.

### 7. When should you use formatDistance vs formatRelative?

**Answer:** `formatDistance` shows duration between dates ('2 hours ago', '3 days ago') and is best for timestamps and activity feeds. `formatRelative` shows relative calendar positions ('yesterday', 'last Monday', 'tomorrow') and is better for calendar contexts. Use formatDistance for precise time differences; use formatRelative when calendar context matters more than exact duration. Both support localization and the addSuffix option for "ago" or "in" prefixes.

## Key Takeaways

1. **Immutable operations** prevent date mutation bugs and enable functional programming
2. **Modular design** with tree shaking keeps bundle sizes small (import only what you need)
3. **Pure functions** ensure predictable behavior and easier testing
4. **Native Date objects** avoid custom wrappers and work with existing code
5. **Comprehensive API** with 200+ functions for all date manipulation needs
6. **Timezone support** via date-fns-tz for timezone-aware applications
7. **Internationalization** with 100+ locales for global applications
8. **TypeScript support** with full type definitions for type safety
9. **Modern replacement** for Moment.js with better performance and smaller size
10. **Functional composition** enables elegant date transformation pipelines

## Resources

- **Official Documentation**: https://date-fns.org/
- **GitHub Repository**: https://github.com/date-fns/date-fns
- **Timezone Package**: https://github.com/marnusw/date-fns-tz
- **Locale List**: https://date-fns.org/docs/Locale
- **Format Tokens**: https://date-fns.org/docs/format
- **Migration Guide**: From Moment.js to date-fns
- **TypeScript Guide**: Using date-fns with TypeScript
- **Bundle Size Calculator**: Measure actual impact
- **Interactive Playground**: Test date-fns online
- **npm Package**: https://www.npmjs.com/package/date-fns
