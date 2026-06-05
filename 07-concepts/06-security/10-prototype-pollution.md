# Prototype Pollution

## The Idea

**In plain English:** Prototype pollution is a security flaw where a hacker tricks a program into secretly adding properties to a shared "blueprint" that every object in JavaScript uses, so those fake properties suddenly appear on everything in the app. A "prototype" is just a template that objects copy their default abilities from.

**Real-world analogy:** Imagine a school where every student's ID card is printed from one master template stored in the office. A sneaky student finds a way to edit that master template and adds "Has hall pass: YES" to it. Now every new ID card printed — for every student — automatically says "Has hall pass: YES," even though none of them actually earned one.

- The master template in the office = `Object.prototype` (the shared blueprint all JavaScript objects inherit from)
- Adding "Has hall pass: YES" to the template = injecting a property like `isAdmin: true` onto `Object.prototype`
- Every student's ID card inheriting the fake entry = every object in the app now having that injected property

---

## Overview

Prototype pollution is a JavaScript vulnerability where an attacker can inject properties into `Object.prototype` — the base prototype that every object inherits from. Because all objects inherit from `Object.prototype`, a polluted prototype affects every object in the application: `{}.__proto__.isAdmin` becoming `true` means every object in the process has `isAdmin === true`. This can bypass authorization checks, cause RCE in server-side code, and corrupt application logic in ways that are extremely hard to debug.

## How JavaScript Prototypes Work

```text
Prototype chain:

const obj = {};
obj.toString(); // works — but obj doesn't define toString!

Lookup order:
1. obj.toString → not found
2. obj.__proto__.toString → found on Object.prototype

Object.prototype is the ROOT of all objects.
Polluting it affects EVERY object in the process.

obj → Object.prototype → null

new Foo() → Foo.prototype → Object.prototype → null

[] → Array.prototype → Object.prototype → null
```

## Attack Vector 1: Deep Merge

The most common vector is a "deep merge" or "deep assign" function that recursively copies properties:

```javascript
// ❌ Vulnerable deep merge
function deepMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object' && source[key] !== null) {
      if (!target[key]) target[key] = {};
      deepMerge(target[key], source[key]);
    } else {
      target[key] = source[key]; // ← vulnerable line
    }
  }
  return target;
}

// Attack payload:
const userInput = JSON.parse('{"__proto__": {"isAdmin": true}}');

const config = {};
deepMerge(config, userInput);

// Now EVERY object has isAdmin: true
console.log({}.isAdmin);           // true ← polluted!
console.log([].isAdmin);           // true ← arrays affected too!
console.log(new Date().isAdmin);   // true ← all objects affected!
```

### Why `__proto__` Is Dangerous

```javascript
// In JSON, __proto__ is just a string key — JSON.parse handles it
const payload = JSON.parse('{"__proto__": {"evil": "pwned"}}');
// payload is: { __proto__: { evil: 'pwned' } }
// payload.__proto__ === Object.prototype? NO — it's a plain property

// But when deeply merged:
target[key] = source[key];
// If key === '__proto__', this does:
target.__proto__ = source.__proto_value__;
// This modifies Object.prototype!

// Also vulnerable:
target['__proto__']['polluted'] = 'yes';
// Same as:
Object.prototype.polluted = 'yes';
```

## Attack Vector 2: Object.assign with Nested Objects

```javascript
// Object.assign is shallow — not directly vulnerable for __proto__
// But manual nested assign can be:

function assign(target, source) {
  Object.keys(source).forEach(key => {
    if (typeof source[key] === 'object') {
      if (!target[key]) target[key] = {};
      assign(target[key], source[key]); // recursive — vulnerable
    } else {
      target[key] = source[key];
    }
  });
}
```

## Attack Vector 3: Query String / URL Parsing

```javascript
// Some qs / querystring libraries historically vulnerable
// Payload: ?__proto__[admin]=true

const qs = require('qs');
const parsed = qs.parse('__proto__[admin]=true', { allowPrototypes: true });
// ← allowPrototypes: true enables this! Default is false in modern qs

// Modern qs (v6.3.3+) ignores __proto__ by default
// But custom parsers may not
```

## Attack Vector 4: Property Access via Path String

```typescript
// Dangerous: setting nested value via dot-notation string
function setByPath(obj: any, path: string, value: any) {
  const parts = path.split('.');
  let current = obj;
  for (let i = 0; i < parts.length - 1; i++) {
    current = current[parts[i]]; // ← vulnerable traversal
  }
  current[parts[parts.length - 1]] = value; // ← vulnerable assignment
}

// Attack:
setByPath({}, '__proto__.isAdmin', true);
// Equivalent to: Object.prototype.isAdmin = true
```

## Real-World Impact

```text
Prototype pollution impacts:

Authorization bypass:
  if (user.isAdmin) { /* ... */ }
  // If Object.prototype.isAdmin = true → everyone is admin

Default property injection:
  const config = {};
  if (config.debugMode) startDebugServer(); // attacker sets debugMode = true

Template injection / RCE (server-side):
  Some template engines (Pug, Handlebars) read properties from Object.prototype
  Polluting certain keys → arbitrary template injection → RCE

Express.js res.locals:
  res.locals is a plain object — inherits from Object.prototype
  Polluted properties accessible in all templates

Node.js process.env:
  Some parsers use object spread on env vars
  Prototype pollution can inject environment variables
```

## Mitigations

### 1. Object.create(null) — Null Prototype Objects

```typescript
// ❌ Regular object — inherits from Object.prototype
const config: Record<string, unknown> = {};
// config.__proto__ === Object.prototype ← can be polluted

// ✅ Null prototype object — no prototype chain
const safeConfig = Object.create(null) as Record<string, unknown>;
// safeConfig.__proto__ === undefined
// Even if __proto__ is set on it, it doesn't affect Object.prototype
// because Object.assign/spread won't set the prototype chain

// Use for dictionaries that receive user-controlled keys
const userPreferences = Object.create(null);
deepMerge(userPreferences, untrustedInput); // safer — no prototype chain to traverse
```

### 2. Safe Deep Merge with Key Denylist

```typescript
// ❌ Vulnerable
function deepMerge(target: any, source: any): any {
  for (const key of Object.keys(source)) {
    if (isObject(source[key])) {
      target[key] = deepMerge(target[key] ?? {}, source[key]);
    } else {
      target[key] = source[key]; // ← vulnerable
    }
  }
  return target;
}

// ✅ Block dangerous keys
const DANGEROUS_KEYS = new Set(['__proto__', 'constructor', 'prototype']);

function safeMerge(target: any, source: any): any {
  for (const key of Object.keys(source)) {
    if (DANGEROUS_KEYS.has(key)) continue; // ← skip dangerous keys

    if (isObject(source[key]) && isObject(target[key])) {
      safeMerge(target[key], source[key]);
    } else if (!DANGEROUS_KEYS.has(key)) {
      target[key] = source[key];
    }
  }
  return target;
}

function isObject(val: unknown): val is Record<string, unknown> {
  return val !== null && typeof val === 'object' && !Array.isArray(val);
}
```

### 3. Object.freeze(Object.prototype)

```typescript
// Freeze Object.prototype — prevent any modifications
Object.freeze(Object.prototype);

// Now pollution attempts throw in strict mode, silently fail otherwise
const payload = JSON.parse('{"__proto__": {"evil": true}}');
const config = {};
deepMerge(config, payload); // Object.prototype.evil = true → throws TypeError!

// Trade-offs:
// ✅ Prevents all prototype pollution
// ⚠️  Can break libraries that legitimately extend prototypes (rare in modern code)
// ⚠️  Must call at application startup, before any other imports if polyfills are needed
// ✅ Safe to call in most modern applications
```

```typescript
// Freeze multiple prototypes
[Object, Array, Function, String, Number, Boolean].forEach((cls) => {
  Object.freeze(cls.prototype);
});
```

### 4. Schema Validation Before Deep Merge

```typescript
import { z } from 'zod';

// Define allowed shape — anything outside schema is rejected
const ConfigSchema = z.object({
  theme: z.enum(['dark', 'light']),
  language: z.string().max(5),
  notifications: z.boolean(),
});

type Config = z.infer<typeof ConfigSchema>;

function mergeUserConfig(
  defaults: Config,
  userInput: unknown
): Config {
  // Validate and strip unknown keys before any merge
  const parsed = ConfigSchema.safeParse(userInput);
  if (!parsed.success) {
    console.warn('Invalid config input:', parsed.error.errors);
    return defaults;
  }

  // Now safe to spread — schema validation stripped __proto__ and constructor
  return { ...defaults, ...parsed.data };
}
```

### 5. Using hasOwnProperty for Safe Property Checks

```typescript
// ❌ Checking inherited properties
if ('isAdmin' in user) { /* dangerous — checks prototype chain too */ }

// ✅ Check own properties only
if (Object.prototype.hasOwnProperty.call(user, 'isAdmin')) {
  // Only true if user itself has isAdmin, not from prototype
}

// Even safer with null-prototype objects:
const dict = Object.create(null);
// No hasOwnProperty method — must use:
Object.prototype.hasOwnProperty.call(dict, 'key');
// OR
Object.hasOwn(dict, 'key'); // ES2022+
```

### 6. JSON Schema Validation

```typescript
// Validate untrusted JSON payloads against a strict schema
import Ajv from 'ajv';

const ajv = new Ajv({ allowUnionTypes: true });

const schema = {
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'number' },
  },
  additionalProperties: false, // ← key: reject unknown keys like __proto__
  required: ['name'],
};

function parseUserInput(raw: unknown): { name: string; age?: number } | null {
  const valid = ajv.validate(schema, raw);
  if (!valid) return null;
  return raw as { name: string; age?: number };
}
```

## Detecting Prototype Pollution

```typescript
// Check if Object.prototype has been polluted
function detectPrototypePollution(): string[] {
  const polluted: string[] = [];
  const baseline = Object.getOwnPropertyNames(Object.prototype);

  // Known baseline properties — any additions are suspicious
  const known = new Set([
    'constructor', 'hasOwnProperty', 'isPrototypeOf',
    'propertyIsEnumerable', 'toString', 'toLocaleString',
    'valueOf', '__defineGetter__', '__defineSetter__',
    '__lookupGetter__', '__lookupSetter__', '__proto__',
  ]);

  for (const key of baseline) {
    if (!known.has(key)) {
      polluted.push(key);
    }
  }

  return polluted;
}

// Add to monitoring
setInterval(() => {
  const polluted = detectPrototypePollution();
  if (polluted.length > 0) {
    console.error('PROTOTYPE POLLUTION DETECTED:', polluted);
    reportSecurityIncident({ type: 'prototype-pollution', keys: polluted });
  }
}, 5000);
```

## Common Mistakes

### 1. Assuming JSON.parse Is Safe

```typescript
// JSON.parse doesn't pollute directly — but passing the result to vulnerable code does
const untrusted = JSON.parse(userInput);
// untrusted is a plain object — safe so far

// ❌ Immediately merging into config without validation:
Object.assign(config, untrusted);      // shallow — safe for __proto__
deepMerge(config, untrusted);          // ❌ vulnerable if untrusted has __proto__

// ✅ Validate schema first, then merge
const validated = ConfigSchema.parse(untrusted);
Object.assign(config, validated);
```

### 2. Using for...in Without hasOwnProperty Check

```typescript
// ❌ for...in iterates prototype properties
const obj = { a: 1 };
// Attacker pollutes: Object.prototype.evil = 'injected'
for (const key in obj) {
  console.log(key); // 'a', 'evil' ← prototype pollution leaks here!
  processKey(obj[key]);
}

// ✅ Filter to own properties
for (const key in obj) {
  if (Object.hasOwn(obj, key)) {
    processKey(obj[key as keyof typeof obj]);
  }
}

// Or: use Object.keys() which only returns own enumerable properties
for (const key of Object.keys(obj)) {
  processKey(obj[key]);
}
```

### 3. Building Path Setters Without Validation

```typescript
// ❌ No key validation
function setNestedValue(obj: any, keys: string[], value: unknown): void {
  keys.reduce((current, key, index) => {
    if (index === keys.length - 1) current[key] = value;
    return current[key] ?? (current[key] = {});
  }, obj);
}

// ✅ Validate keys
function setNestedValueSafe(obj: any, keys: string[], value: unknown): void {
  const BLOCKED = new Set(['__proto__', 'constructor', 'prototype']);
  if (keys.some((k) => BLOCKED.has(k))) {
    throw new Error(`Blocked key in path: ${keys.join('.')}`);
  }
  keys.reduce((current, key, index) => {
    if (index === keys.length - 1) current[key] = value;
    return current[key] ?? (current[key] = {});
  }, obj);
}
```

## Interview Questions

### 1. What is prototype pollution and what makes it dangerous?

**Answer:** Prototype pollution modifies `Object.prototype` — the base object all JavaScript objects inherit from. Because every object shares this prototype, injecting a property (e.g., `Object.prototype.isAdmin = true`) makes it appear on every object in the process. This bypasses authorization checks (`if (user.isAdmin)` is true for everyone), corrupts default values, and in server-side contexts can cause RCE via template engines that read from `Object.prototype`. The attack is subtle because the pollution is invisible in the object itself — it only appears through prototype chain traversal.

### 2. What are the three most common attack vectors for prototype pollution?

**Answer:** (1) Deep merge functions that recursively copy properties including `__proto__` — the most common. (2) Property path setters that accept dot-notation strings like `"__proto__.isAdmin"`. (3) URL query string parsers with `allowPrototypes: true` or custom parsers that don't block `__proto__`. All three share the same root cause: code that sets `target[key] = value` without validating that `key` is not `__proto__`, `constructor`, or `prototype`.

### 3. How does Object.freeze(Object.prototype) prevent prototype pollution?

**Answer:** `Object.freeze` makes an object non-extensible and makes all existing properties non-writable and non-configurable. Calling it on `Object.prototype` prevents any code from adding or modifying properties on the prototype. A pollution attempt (`Object.prototype.evil = true`) will throw a TypeError in strict mode or silently fail in sloppy mode — the property is never set. This is the most comprehensive defense because it prevents pollution regardless of how vulnerable the merging code is. The trade-off is that it can break polyfills that add methods to `Object.prototype`, so it must be applied carefully, ideally before any polyfills run.

### 4. Why is validating JSON input not enough to prevent prototype pollution?

**Answer:** `JSON.parse` itself doesn't cause prototype pollution — the parsed result is a plain JavaScript object. The vulnerability lies in what you do with that object afterward. If you pass a parsed object containing `__proto__` as a key to an unsafe deep merge function, the merge function will write to `Object.prototype`. Schema validation with `additionalProperties: false` or Zod strips unknown keys (including `__proto__`) before the data reaches merging code, which is effective. But if the validation step is bypassed or the data is merged before validation, the pollution can still occur.

### 5. How would you safely implement a deep merge function?

**Answer:** Block dangerous keys (`__proto__`, `constructor`, `prototype`) before recursing. Check them against a denylist using `DANGEROUS_KEYS.has(key)` and skip if matched. Use `Object.keys()` rather than `for...in` so inherited properties aren't traversed. Consider using null-prototype target objects (`Object.create(null)`) for dictionaries that receive untrusted keys. Use `Object.hasOwn(source, key)` rather than `key in source` to avoid checking the prototype chain. The most robust approach is to validate the schema of input with Zod or Ajv before merging, so the shape is known and safe before any recursive property assignment.
