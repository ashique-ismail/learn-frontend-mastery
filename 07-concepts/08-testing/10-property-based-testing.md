# Property-Based Testing

## Overview

Example-based testing asks "does this specific input produce this specific output?" Property-based testing asks "does this function hold these invariants for ALL valid inputs?" Instead of hand-picking test cases, property-based testing generates hundreds or thousands of random inputs and verifies that defined properties hold true for each one. When a counterexample is found, the framework automatically shrinks it to the minimal failing case. This guide covers the theory, the `fast-check` library for JavaScript/TypeScript, generators, shrinking, and when property-based testing pays off.

## Example-Based vs Property-Based

```
Example-based test:
  input: "hello"
  expected: "HELLO"
  → Tests one case. Miss edge cases.

Property-based test:
  Property: toUpperCase(s).length === s.length
  Property: toUpperCase(s) === toUpperCase(toUpperCase(s)) (idempotent)
  Property: toUpperCase(s).toLowerCase() === s.toLowerCase()
  → Tests hundreds of generated strings.
  → Finds edge cases you'd never think of.
```

### What Properties Look Like

```
Common property patterns:

1. Roundtrip (encode/decode)
   decode(encode(x)) === x
   Example: JSON.parse(JSON.stringify(obj)) deep-equals obj

2. Idempotency
   f(f(x)) === f(x)
   Example: sort(sort(arr)) deep-equals sort(arr)

3. Commutativity
   f(a, b) === f(b, a)
   Example: add(a, b) === add(b, a)

4. Associativity
   f(f(a, b), c) === f(a, f(b, c))
   Example: concat(concat(a, b), c) deep-equals concat(a, concat(b, c))

5. Invariants
   result.length >= 0
   result is a valid date
   result.price >= 0

6. Oracle (known reference)
   mySort(arr) deep-equals [...arr].sort(compareFn)

7. Symmetry
   getUser(createUser(data)).id === data.id
```

## fast-check

`fast-check` is the standard property-based testing library for JavaScript and TypeScript:

```bash
npm install --save-dev fast-check
```

```typescript
import * as fc from 'fast-check';

// Basic example: addition commutativity
test('addition is commutative', () => {
  fc.assert(
    fc.property(fc.integer(), fc.integer(), (a, b) => {
      expect(a + b).toBe(b + a);
    })
  );
});
// fast-check generates 100 random (a, b) pairs and verifies the property each time

// Roundtrip: JSON serialization
test('JSON roundtrip preserves primitive values', () => {
  fc.assert(
    fc.property(
      fc.oneof(
        fc.string(),
        fc.integer(),
        fc.boolean(),
        fc.constant(null)
      ),
      (value) => {
        expect(JSON.parse(JSON.stringify(value))).toEqual(value);
      }
    )
  );
});
```

## Arbitraries — Generating Test Data

`fast-check` ships with built-in arbitraries (generators) for common types:

```typescript
// Primitives
fc.integer()                      // any integer
fc.integer({ min: 0, max: 100 }) // bounded integer
fc.float()                        // float
fc.string()                       // any string
fc.string({ minLength: 1 })      // non-empty string
fc.boolean()                      // true or false
fc.constant('fixed')             // always the same value

// Collections
fc.array(fc.integer())           // array of integers
fc.array(fc.string(), { minLength: 1, maxLength: 10 })
fc.set(fc.integer())             // Set of integers (unique)
fc.tuple(fc.string(), fc.integer()) // [string, integer] tuple

// Structured data
fc.record({
  id: fc.uuidV4(),
  name: fc.string({ minLength: 1, maxLength: 100 }),
  age: fc.integer({ min: 0, max: 120 }),
  email: fc.emailAddress(),
})

// Special types
fc.date()                        // random Date
fc.uuidV4()                      // UUID v4
fc.emailAddress()                // valid email
fc.ipV4()                        // IPv4 address
fc.url()                         // valid URL

// Recursive structures
const jsonValue: fc.Arbitrary<unknown> = fc.letrec((tie) => ({
  value: fc.oneof(
    fc.string(),
    fc.integer(),
    fc.boolean(),
    fc.constant(null),
    fc.array(tie('value')),
    fc.dictionary(fc.string({ minLength: 1 }), tie('value')),
  ),
})).value;
```

## Custom Arbitraries

```typescript
import * as fc from 'fast-check';

// Custom type: valid user
interface User {
  id: string;
  email: string;
  age: number;
  role: 'admin' | 'editor' | 'viewer';
}

const userArbitrary: fc.Arbitrary<User> = fc.record({
  id: fc.uuidV4(),
  email: fc.emailAddress(),
  age: fc.integer({ min: 18, max: 100 }),
  role: fc.oneof(
    fc.constant('admin' as const),
    fc.constant('editor' as const),
    fc.constant('viewer' as const)
  ),
});

// Custom type: sorted non-empty array
const sortedNonEmptyArray: fc.Arbitrary<number[]> = fc
  .array(fc.integer(), { minLength: 1 })
  .map((arr) => [...arr].sort((a, b) => a - b));

// Custom type: positive amount (financial)
const positiveAmount: fc.Arbitrary<{ amount: number; currency: string }> =
  fc.record({
    amount: fc.float({ min: 0.01, max: 1_000_000, noNaN: true }),
    currency: fc.oneof(
      fc.constant('USD'),
      fc.constant('EUR'),
      fc.constant('GBP')
    ),
  });
```

## Shrinking

When fast-check finds a failing input, it automatically tries to find the simplest possible counterexample by shrinking the input while the property still fails:

```
Shrinking in action:

fast-check generates: "abcdefghijklmnop123!@#" → property fails
Shrinking:
  → "abcdefghijk123!@#" still fails
  → "abc123" still fails
  → "a1" still fails
  → "1" passes
  → "!" fails ← minimal counterexample found!

You see: "!" instead of "abcdefghijklmnop123!@#"
Much easier to debug!
```

```typescript
// Property that fails: demonstrates shrinking
function parsePositiveInt(s: string): number {
  const n = parseInt(s, 10);
  if (isNaN(n)) throw new Error('Not a number');
  return n; // BUG: doesn't enforce positive
}

test('parsePositiveInt always returns positive', () => {
  expect(() => {
    fc.assert(
      fc.property(fc.string(), (s) => {
        const result = parsePositiveInt(s);
        expect(result).toBeGreaterThan(0);
      })
    );
  }).toThrow();
  // fast-check will report the minimal string that fails, e.g. "0" or "-1"
});
```

## Practical Examples

### Testing a Sort Function

```typescript
function mySort<T>(arr: T[], compare: (a: T, b: T) => number): T[] {
  return [...arr].sort(compare);
}

const numberCompare = (a: number, b: number) => a - b;

test('sort: result has same length as input', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      expect(mySort(arr, numberCompare)).toHaveLength(arr.length);
    })
  );
});

test('sort: result is sorted', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = mySort(arr, numberCompare);
      for (let i = 0; i < sorted.length - 1; i++) {
        expect(sorted[i]).toBeLessThanOrEqual(sorted[i + 1]);
      }
    })
  );
});

test('sort: result contains same elements (permutation)', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = mySort(arr, numberCompare);
      // Same elements — just reordered
      expect([...sorted].sort()).toEqual([...arr].sort());
    })
  );
});

test('sort is idempotent', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = mySort(arr, numberCompare);
      const sortedTwice = mySort(sorted, numberCompare);
      expect(sortedTwice).toEqual(sorted);
    })
  );
});
```

### Testing a Serializer/Deserializer (Roundtrip)

```typescript
interface Post {
  id: string;
  title: string;
  tags: string[];
  publishedAt: Date | null;
}

function serializePost(post: Post): string {
  return JSON.stringify({
    ...post,
    publishedAt: post.publishedAt?.toISOString() ?? null,
  });
}

function deserializePost(json: string): Post {
  const data = JSON.parse(json);
  return {
    ...data,
    publishedAt: data.publishedAt ? new Date(data.publishedAt) : null,
  };
}

const postArbitrary: fc.Arbitrary<Post> = fc.record({
  id: fc.uuidV4(),
  title: fc.string({ minLength: 1, maxLength: 200 }),
  tags: fc.array(fc.string({ minLength: 1, maxLength: 50 }), { maxLength: 10 }),
  publishedAt: fc.oneof(fc.date(), fc.constant(null)),
});

test('serialize/deserialize roundtrip preserves Post', () => {
  fc.assert(
    fc.property(postArbitrary, (post) => {
      const result = deserializePost(serializePost(post));
      expect(result.id).toBe(post.id);
      expect(result.title).toBe(post.title);
      expect(result.tags).toEqual(post.tags);
      expect(result.publishedAt?.getTime()).toBe(post.publishedAt?.getTime());
    })
  );
});
```

### Testing Business Logic: Discount Application

```typescript
interface Cart {
  items: Array<{ price: number; quantity: number }>;
  discountPercent: number; // 0-100
}

function calculateTotal(cart: Cart): number {
  const subtotal = cart.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );
  const discountFactor = 1 - cart.discountPercent / 100;
  return Math.round(subtotal * discountFactor * 100) / 100; // round to cents
}

const cartArbitrary = fc.record({
  items: fc.array(
    fc.record({
      price: fc.float({ min: 0.01, max: 10_000, noNaN: true }),
      quantity: fc.integer({ min: 1, max: 100 }),
    }),
    { minLength: 1 }
  ),
  discountPercent: fc.integer({ min: 0, max: 100 }),
});

test('total is always non-negative', () => {
  fc.assert(
    fc.property(cartArbitrary, (cart) => {
      expect(calculateTotal(cart)).toBeGreaterThanOrEqual(0);
    })
  );
});

test('100% discount results in 0 total', () => {
  fc.assert(
    fc.property(
      fc.array(
        fc.record({
          price: fc.float({ min: 0.01, max: 1000, noNaN: true }),
          quantity: fc.integer({ min: 1, max: 10 }),
        }),
        { minLength: 1 }
      ),
      (items) => {
        const total = calculateTotal({ items, discountPercent: 100 });
        expect(total).toBe(0);
      }
    )
  );
});

test('0% discount equals full price', () => {
  fc.assert(
    fc.property(cartArbitrary, (cart) => {
      const withDiscount = calculateTotal(cart);
      const withoutDiscount = calculateTotal({ ...cart, discountPercent: 0 });
      expect(withDiscount).toBeLessThanOrEqual(withoutDiscount + 0.01); // float tolerance
    })
  );
});
```

## Configuration: Number of Runs, Seed, Verbose

```typescript
// Run with more samples (default: 100)
fc.assert(
  fc.property(fc.string(), (s) => { /* ... */ }),
  { numRuns: 1000 }
);

// Reproducible — use a specific seed
fc.assert(
  fc.property(fc.integer(), (n) => { /* ... */ }),
  { seed: 42 }
);

// Verbose — log all generated values on failure
fc.assert(
  fc.property(fc.record({ name: fc.string(), age: fc.integer() }), (user) => { /* ... */ }),
  { verbose: true }
);

// When a property fails, fast-check reports the seed:
// "Property failed after 47 tests
//  Seed: 1671234567"
// Use that seed to reproduce the exact failure:
fc.assert(
  fc.property(fc.string(), (s) => { /* ... */ }),
  { seed: 1671234567, path: '47' } // path replays the exact shrunk path
);
```

## When to Use Property-Based Testing

```
USE property-based testing for:

✓ Pure functions with mathematical properties
  (sort, parse, format, encode/decode, math operations)

✓ Serialization/deserialization roundtrips
  (JSON, protobuf, custom serializers)

✓ Data transformations and pipelines
  (input → output with invariants)

✓ Business logic with many branches
  (pricing engines, discount calculators, access control)

✓ Parsers and validators
  ("all valid inputs are accepted", "all invalid inputs are rejected")

✓ State machines
  ("any sequence of valid transitions ends in a valid state")

COMPLEMENT with example-based tests for:
  - Known edge cases (null, empty, max value)
  - Specific regression fixes
  - UI behavior (properties don't describe "button turns blue on hover")
  - Integration tests (need real DB, real API)
```

## Common Mistakes

### 1. Testing Too Many Things in One Property

```typescript
// ❌ One property testing too many things — hard to debug when it fails
fc.assert(
  fc.property(fc.array(fc.integer()), (arr) => {
    const sorted = mySort(arr);
    expect(sorted.length).toBe(arr.length);    // invariant 1
    for (let i = 0; i < sorted.length - 1; i++) {
      expect(sorted[i]).toBeLessThanOrEqual(sorted[i + 1]); // invariant 2
    }
    expect(sortedToMultiset(sorted)).toEqual(toMultiset(arr)); // invariant 3
  })
);

// ✅ One test per property — clearer failure messages
test('sort preserves length', ...);
test('sort produces sorted output', ...);
test('sort is a permutation', ...);
```

### 2. Non-Deterministic Code in Properties

```typescript
// ❌ Date.now() or Math.random() inside property — non-deterministic
fc.assert(
  fc.property(fc.integer({ min: 1 }), (days) => {
    const expiry = new Date(Date.now() + days * 86400 * 1000);
    expect(isExpired(expiry)).toBe(false); // Might flake as time passes!
  })
);

// ✅ Pass time as a parameter — make it deterministic
fc.assert(
  fc.property(
    fc.date(),  // controlled reference date
    fc.date(),  // controlled expiry date
    (refDate, expiryDate) => {
      const expired = isExpiredAt(expiryDate, refDate);
      expect(expired).toBe(expiryDate < refDate);
    }
  )
);
```

### 3. Ignoring Shrinking — Making Tests Too Hard to Reproduce

```typescript
// ❌ Using external state that affects shrinking
let callCount = 0;
fc.assert(
  fc.property(fc.integer(), (n) => {
    callCount++;
    return doSomething(n, callCount); // callCount changes every shrink!
  })
);

// ✅ Properties should depend only on the generated values
fc.assert(
  fc.property(fc.integer(), fc.integer(), (n, seed) => {
    return doSomething(n, seed); // fully deterministic
  })
);
```

## Interview Questions

### 1. What is property-based testing and how does it differ from example-based testing?

**Answer:** Example-based testing verifies specific input/output pairs you've chosen manually. Property-based testing defines invariants that must hold for all valid inputs, then automatically generates hundreds of random inputs to test them. The key advantages are: discovering edge cases you wouldn't have thought of, testing a much larger input space with less code, and automatic counterexample minimization (shrinking). Example-based tests excel for specific regression cases and readable documentation-by-example. Property-based testing excels for pure functions with mathematical properties, roundtrips, and business logic with many branches.

### 2. What is shrinking and why does it matter?

**Answer:** When fast-check finds a failing input, it automatically tries to find a simpler input that still fails the property — shrinking progressively reduces the size and complexity of the counterexample. Without shrinking, a failing input might be a 200-character string with random content. After shrinking, it might be "0" or "!" — the minimal case that triggers the bug. This makes the failure immediately debuggable. fast-check's arbitraries have built-in shrinking logic: integers shrink toward 0, strings shrink toward empty, arrays shrink by removing elements. The seed and path values reported on failure let you reproduce the exact shrunk counterexample.

### 3. When would you NOT use property-based testing?

**Answer:** When there are no clear invariants — UI interaction tests ("clicking Buy shows the cart") don't have mathematical properties that generalize across random inputs. When the function is so simple that generating random inputs adds noise without finding anything useful. When the test needs real infrastructure (databases, APIs) — property tests should be fast and pure; external I/O makes them slow and non-deterministic. When specific edge cases matter more than general properties — a known regression test for "empty string crashes the parser" is clearer as an example-based test. Property-based testing is most valuable for pure, deterministic transformations.

### 4. How do you write a useful property for a function that isn't obviously mathematical?

**Answer:** Look for the invariants that must hold regardless of input: "result is never null," "result.length >= input.length," "output is a valid date," "total >= 0," "response time < 100ms" (not for property tests, but structurally similar). Use oracle patterns — compare your implementation against a known-correct reference: `mySort(arr)` should equal `[...arr].sort()`. Use roundtrip patterns — `deserialize(serialize(x))` equals `x`. Use monotonicity — "more input items never decreases output." These patterns apply to business logic too: "applying a 100% discount always yields 0 total" or "no user with role=viewer can call adminAction."
