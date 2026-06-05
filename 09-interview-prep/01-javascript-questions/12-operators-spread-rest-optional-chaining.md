# Spread, Rest, Optional Chaining & Nullish Coalescing

## The Idea

**In plain English:** These are special symbols in JavaScript that let you unpack a collection of items, gather multiple items into one group, safely look up information without crashing, or choose a fallback value when something is missing. Think of them as handy shortcuts that make common tasks less wordy and less error-prone.

**Real-world analogy:** Imagine you are packing for a trip. You have one big suitcase (an array) already packed. Spread is like dumping all those clothes out onto the bed so you can mix them with new items — then repacking everything together. Rest is the opposite: you grab the specific outfit you need today and toss everything else back into a single pile. Optional chaining is like checking "does this drawer even exist?" before opening it, so you do not accidentally yank open a drawer that was never installed. Nullish coalescing is like having a backup restaurant in mind — you only go there if your first choice is completely closed, not just a bit slow.

- The suitcase (array with existing items) = the original array or object you want to expand with `...` (spread)
- Grabbing one outfit and tossing the rest into a pile = pulling out named variables and collecting leftovers with `...` (rest)
- Checking "does this drawer exist?" before opening it = using `?.` to safely access a property that might not exist (optional chaining)
- The backup restaurant you visit only when the first is fully closed = using `??` to fall back only when a value is `null` or `undefined` (nullish coalescing)

---

## Overview

These ES2015–ES2021 operators appear constantly in modern JavaScript. Interviewers test whether you understand the subtle differences between visually similar syntax and know the gotchas.

---

## Spread Operator (`...`)

**Expands** an iterable or object into individual elements.

### Arrays

```js
const a = [1, 2, 3];
const b = [...a, 4, 5];      // [1, 2, 3, 4, 5]
const copy = [...a];         // shallow copy
Math.max(...a);              // 3 — spreads into arguments
```

### Objects

```js
const defaults = { theme: 'light', lang: 'en' };
const user = { ...defaults, lang: 'fr', name: 'Alice' };
// { theme: 'light', lang: 'fr', name: 'Alice' }
// Later keys override earlier ones
```

**Gotcha — Shallow Copy:**

```js
const obj = { a: { b: 1 } };
const copy = { ...obj };
copy.a.b = 99;   // mutates original! nested ref is shared
```

---

## Rest Parameters (`...`)

**Collects** remaining arguments into an array. Only valid as the *last* parameter.

```js
function sum(first, ...rest) {
  return first + rest.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4); // 10

// Rest in destructuring
const [head, ...tail] = [1, 2, 3, 4];  // head=1, tail=[2,3,4]
const { a, ...remaining } = { a: 1, b: 2, c: 3 };  // remaining={b:2,c:3}
```

**Rest vs `arguments`:**

- `arguments` is array-like (no array methods), not available in arrow functions
- Rest parameters are real arrays, work everywhere

---

## Optional Chaining (`?.`)

Short-circuits to `undefined` instead of throwing when accessing a property on `null` or `undefined`.

```js
const user = null;
user?.profile?.avatar;   // undefined (no throw)
user.profile.avatar;     // TypeError!
```

### Variants

```js
// Method calls
obj?.method?.();          // calls only if method exists

// Bracket notation
arr?.[0];                 // safe array access

// With function calls
callback?.();             // calls if callback is not nullish
```

### Practical Pattern

```js
// Before
const city = user && user.address && user.address.city;
// After
const city = user?.address?.city;
```

**Gotcha:** `?.` stops evaluation at the first nullish value, returning `undefined` — not `null`, not an error. Falsy values (0, '', false) are NOT short-circuited.

---

## Nullish Coalescing (`??`)

Returns the right side only when the left side is `null` or `undefined` (not for other falsy values).

```js
const count = userCount ?? 0;  // 0 only if userCount is null/undefined
const count2 = userCount || 0; // 0 also if userCount is 0 or '' or false — BUG RISK!

0 ?? 'default';     // 0  (0 is not nullish)
'' ?? 'default';    // '' (empty string is not nullish)
null ?? 'default';  // 'default'
undefined ?? 'default'; // 'default'
```

**When to prefer `??` over `||`:** Whenever `0`, `false`, or `''` are valid values (counts, flags, empty inputs).

---

## Logical Assignment Operators (ES2021)

```js
// &&= — assign only if left is truthy
user.role &&= 'editor';       // sets role only if it already had a truthy value

// ||= — assign only if left is falsy
settings.theme ||= 'light';   // sets default only if theme is falsy

// ??= — assign only if left is null/undefined
config.timeout ??= 5000;      // sets default only if truly absent
```

These are short-circuit assignments — the right side is not evaluated if the condition isn't met.

---

## Common Interview Questions

**Q: What's the difference between `??` and `||`?**
`||` returns right side for any falsy left value (including 0, '', false). `??` only returns right side for `null` or `undefined`. Use `??` when 0 or empty string are valid values you want to keep.

**Q: Can you use optional chaining with `delete`?**
Yes: `delete obj?.prop` — does nothing if `obj` is nullish.

**Q: What does `{...a, ...b}` do when both have the same key?**
The last key wins. `{...a, ...b}` means `b`'s keys override `a`'s.

**Q: Is spread always a shallow copy?**
Yes. Spread copies one level deep. Nested objects/arrays are still shared references.

**Q: Can you combine `?.` and `??`?**
Yes, and it's idiomatic:

```js
const name = user?.profile?.displayName ?? 'Anonymous';
```
