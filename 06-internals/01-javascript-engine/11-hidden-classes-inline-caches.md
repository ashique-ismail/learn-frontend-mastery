# Hidden Classes and Inline Caches

## The Idea

**In plain English:** JavaScript engines like V8 secretly track the "shape" of your objects (which properties they have and in what order) using hidden classes, then use that knowledge to remember shortcuts — called inline caches — so that looking up a property is nearly instant instead of searching through a dictionary every time.

**Real-world analogy:** Imagine a hotel front desk that handles thousands of guests. At first, the clerk looks up every guest's room in a big binder (slow). After a while, they notice that guests in suits always stay in Room 101, guests with backpacks always stay in Room 202, and so on. They write these patterns on sticky notes (the cache). Now when a suited guest walks up, the clerk instantly says "Room 101" without opening the binder.

- The guest's outfit (suit, backpack, etc.) = the hidden class (the shape of the object)
- The sticky note shortcut = the inline cache (the remembered offset for that shape)
- A guest who dresses differently every day = an object whose shape keeps changing, forcing the clerk to abandon the sticky note and go back to the binder (megamorphic / slow path)

---

## Table of Contents
- [Overview](#overview)
- [Hidden Classes (Maps/Shapes)](#hidden-classes-mapsshapes)
- [Transition Chains](#transition-chains)
- [Inline Caches](#inline-caches)
- [IC State Machine](#ic-state-machine)
- [What Breaks Optimization](#what-breaks-optimization)
- [Array Element Kinds](#array-element-kinds)
- [Debugging Optimization State](#debugging-optimization-state)
- [Writing V8-Friendly Code](#writing-v8-friendly-code)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

V8 accelerates object property access through two cooperating mechanisms: **hidden classes** (V8 calls them *Maps*; the spec community calls them *Shapes*) and **inline caches** (ICs). Together they transform what would be dictionary lookups (O(n) or O(log n)) into direct memory-offset reads (O(1)) — roughly equivalent in speed to accessing a C struct field.

Understanding these mechanisms matters at the senior level for two reasons:

1. **Performance**: code patterns that preserve shapes see 10–100× faster property access
2. **Deoptimization awareness**: shape mutations can silently downgrade optimized code to interpreted mode

```
Without hidden classes:
  obj.x  →  hash-map lookup("x")  →  result
            ~5–20 instructions

With hidden classes + inline cache:
  obj.x  →  check hidden class == HC2  →  load offset 0
            ~2 instructions (or 1 with full optimization)
```

## Hidden Classes (Maps/Shapes)

### What a Hidden Class Contains

A hidden class is an internal V8 object that describes the **structure** (not the values) of a JS object. Each hidden class stores:

- The list of property names their **memory offsets** within the object's backing store
- A **descriptor array** mapping property name → offset + attributes (writable, enumerable, configurable)
- A pointer to the **prototype** (for prototype chain lookups)
- A **transition table** pointing to sibling/child hidden classes

```
JS Object in memory:
  ┌─────────────────────────┐
  │ [Map pointer] → HC2     │  ← points to hidden class
  │ [Properties pointer]    │
  │ [Elements pointer]      │
  │ in-object property slot │  ← x value (offset 0)
  │ in-object property slot │  ← y value (offset 1)
  └─────────────────────────┘

HC2 (Hidden Class for {x, y}):
  ┌──────────────────────────────┐
  │ descriptor: x → offset 0    │
  │ descriptor: y → offset 1    │
  │ prototype: Object.prototype  │
  │ transitions: {z → HC3}      │
  └──────────────────────────────┘
```

### Initial Hidden Class Assignment

```typescript
// Every object starts with a root hidden class
const empty = {}; // HC0 (the root — no properties)

// Adding 'x' transitions to a new hidden class
const withX = { x: 1 }; // HC1 (has 'x' at offset 0)

// Adding 'y' transitions again
const withXY = { x: 1, y: 2 }; // HC2 (has 'x' at 0, 'y' at 1)

// SHARED: Same hidden class for same shape
const p1 = { x: 1, y: 2 }; // HC2
const p2 = { x: 5, y: 10 }; // HC2 — SAME hidden class, different values
// p1 and p2 share HC2, so property accesses on both use the same cache
```

### Hidden Classes Are Structural, Not Value-Based

```typescript
// HC is determined by WHICH properties exist and their ORDER
// NOT by property values

const a = { x: 1, y: 2 };
const b = { x: 999, y: -7 }; // Same HC as a — values differ, structure same

const c = { x: 1, y: 2, z: 3 }; // Different HC (HC3) — extra property

// Even same properties in different order get different HCs
const d = { y: 2, x: 1 }; // Different HC from a!
// Because d's transition chain: HC0 → (add y) → HC_y → (add x) → HC_yx
// Versus a's chain:             HC0 → (add x) → HC_x → (add y) → HC_xy
```

## Transition Chains

### The Transition Tree

Hidden classes form a tree rooted at the root HC for `{}`. Each property addition creates a child node. V8 reuses existing nodes if the same transition path is taken.

```
Transition tree:

HC0 (empty)
├── + x → HC1 (x)
│   ├── + y → HC2 (x,y)         ← { x, y } and { x: any, y: any }
│   │   └── + z → HC3 (x,y,z)
│   └── + z → HC4 (x,z)         ← { x, z } — different branch!
└── + y → HC5 (y)               ← { y } — starts with y
    └── + x → HC6 (y,x)         ← { y, x } — different from HC2!

HC2 != HC6 even though both have x and y!
```

```typescript
// Demonstrating transition reuse
function makePoint(x: number, y: number) {
  const p: any = {};
  p.x = x; // Transition: HC0 → HC1 (reuses existing if already created)
  p.y = y; // Transition: HC1 → HC2 (reuses existing)
  return p;
}

// First call creates the transition chain
const p1 = makePoint(1, 2); // Creates HC0 → HC1 → HC2

// Subsequent calls REUSE the same hidden classes
const p2 = makePoint(3, 4); // Reuses HC2 directly — no allocation!
const p3 = makePoint(5, 6); // Reuses HC2 — p1, p2, p3 share HC2
```

### Constructor Functions and Stable Shapes

```typescript
// class/constructor enforces consistent property initialization order
class Vector2D {
  x: number;
  y: number;

  constructor(x: number, y: number) {
    this.x = x; // Always first — HC transition order is stable
    this.y = y; // Always second
  }
}

// All instances share the same terminal hidden class
const v1 = new Vector2D(1, 2);  // HC_Vector
const v2 = new Vector2D(3, 4);  // HC_Vector (same!)
const v3 = new Vector2D(-1, 0); // HC_Vector (same!)

// V8 can prove these all have the same shape → full optimization
function magnitude(v: Vector2D): number {
  // Inline cached: x is at offset 0, y is at offset 1
  return Math.sqrt(v.x * v.x + v.y * v.y);
}
```

### Property Deletion Breaks Chains

```typescript
const obj = { a: 1, b: 2, c: 3 }; // HC: (a, b, c)

// delete creates a new "dictionary-mode" object
delete obj.b; // obj transitions to "slow mode" (dictionary)

// Once in dictionary mode, V8 cannot use inline caches
// obj.a, obj.c are now hashtable lookups — much slower!

// WHY: No hidden class exists for "has a and c but not b at position 1"
// The shape is now irregular → dictionary mode

// ALTERNATIVE: null out instead of delete
const obj2 = { a: 1, b: 2, c: 3 };
obj2.b = null; // HC stays the same — b still exists at offset 1
               // Value is null but shape is unchanged
```

## Inline Caches

### What Is an Inline Cache

An inline cache (IC) is a **call-site-specific** cache embedded at the location in the bytecode/machine-code where a property access occurs. The key insight: the same property access (`obj.x`) at a specific position in the code usually sees objects of the same shape repeatedly.

```
Bytecode for: return obj.x

Before any caches:
  [LoadIC slot=0] → generic lookup path

After first call with HC2 object:
  [LoadIC slot=0] → check(obj.Map == HC2) → load(obj, offset=0)
                    ^^^^^^^^^^^^^^^^^^^^^^^^^
                    cached check and load — very fast!
```

### IC Types

```typescript
// UNINITIALIZED: Never seen any call
function fresh(obj: any): unknown {
  return obj.x; // IC is uninitialized until called
}

// MONOMORPHIC: Seen exactly one shape (fastest)
function mono(obj: { x: number }): number {
  return obj.x; // Always gets HC2 → monomorphic IC → near-optimal speed
}
const a = { x: 1, y: 2 };
const b = { x: 3, y: 4 };
mono(a); mono(b); // Both HC2 → IC stays monomorphic

// POLYMORPHIC: Seen 2–4 shapes (fast, but with branching)
function poly(obj: { x: number }): number {
  return obj.x;
}
poly({ x: 1, y: 2 });       // HC2
poly({ x: 1, y: 2, z: 3 }); // HC3 — IC goes polymorphic (checks 2 shapes)
poly({ x: 1, a: 2 });        // HC4 — IC checks 3 shapes

// MEGAMORPHIC: Seen > 4 shapes (slow — falls back to generic lookup)
function mega(obj: any): unknown {
  return obj.x;
}
// After 5+ different hidden classes, IC gives up caching
```

## IC State Machine

```
IC State Transitions:

UNINITIALIZED
    │ first call with shape S1
    ▼
MONOMORPHIC (caches S1)
    │ call with shape S2 (S2 ≠ S1)
    ▼
POLYMORPHIC (caches S1, S2) ← up to 4 shapes
    │ call with 5th distinct shape
    ▼
MEGAMORPHIC (global stub — no per-site caching)
    │
    (stays megamorphic — no way back)
    ▼
GENERIC (complete fallback, like unoptimized code)
```

```typescript
// Demonstrating IC degradation
class Logger {
  log(value: any): void {
    console.log(value.name); // IC for .name
  }
}

const logger = new Logger();

// Monomorphic — all { name } objects
logger.log({ name: 'Alice' });
logger.log({ name: 'Bob' });
logger.log({ name: 'Carol' }); // IC: monomorphic (shape: {name})

// IC goes polymorphic
logger.log({ name: 'Dave', age: 30 });    // shape: {name, age}
logger.log({ name: 'Eve', role: 'admin' }); // shape: {name, role}

// IC goes megamorphic after many shapes
// Subsequent calls are significantly slower
```

### Recovering from Megamorphic ICs

```typescript
// BAD: Generic function called with many shapes
function getNameBad(entity: any): string {
  return entity.name; // Will go megamorphic quickly
}

getNameBad({ name: 'user', email: '...' });
getNameBad({ name: 'product', price: 9.99 });
getNameBad({ name: 'order', items: [] });
getNameBad({ name: 'category', slug: '...' });
getNameBad({ name: 'tag', color: 'red' }); // Megamorphic after 5th shape

// GOOD: Use a base class / interface to normalize shapes
interface Named { name: string }
class User implements Named { constructor(public name: string, public email: string) {} }
class Product implements Named { constructor(public name: string, public price: number) {} }

function getNameGood(entity: Named): string {
  return entity.name; // If entity is always User or Product, IC is polymorphic
}

// BEST: If truly needed, extract property access into shape-specific code
function getUserName(user: User): string { return user.name; }
function getProductName(product: Product): string { return product.name; }
```

## What Breaks Optimization

### Shape Mutations After Initialization

```typescript
// BAD: Adding properties after construction
function makeConfig(opts: Record<string, unknown>) {
  const config: any = { debug: false, verbose: false };
  // config starts with HC_base

  if (opts.debug) config.debug = true;           // still HC_base
  if (opts.logLevel) config.logLevel = opts.logLevel; // NEW property → transition!
  if (opts.timeout) config.timeout = opts.timeout;    // ANOTHER transition!

  return config;
  // config may have HC_base, HC_logLevel, HC_timeout, HC_both
  // depending on which options were passed — multiple shapes from one factory!
}

// GOOD: Initialize all possible properties upfront
function makeConfigGood(opts: Record<string, unknown>) {
  const config = {
    debug: false,
    verbose: false,
    logLevel: null as unknown,  // Always present, may be null
    timeout: null as unknown,   // Always present, may be null
  };
  if (opts.debug) config.debug = true;
  if (opts.logLevel) config.logLevel = opts.logLevel;
  if (opts.timeout) config.timeout = opts.timeout;
  return config; // Always the same HC regardless of input
}
```

### Conditional Property Addition

```typescript
// BAD: Shape depends on runtime condition
class Connection {
  private socket: WebSocket;
  // ERROR field only added on failure — creates two possible shapes
  error?: Error;

  connect(url: string): void {
    try {
      this.socket = new WebSocket(url);
    } catch (e) {
      this.error = e as Error; // Shape change! New property added
    }
  }
}

// GOOD: Declare all properties in constructor
class ConnectionGood {
  private socket: WebSocket | null = null;
  private error: Error | null = null; // Always present, starts null

  connect(url: string): void {
    try {
      this.socket = new WebSocket(url);
    } catch (e) {
      this.error = e as Error; // No shape change — property already exists
    }
  }
}
```

### Prototype Mutations

```typescript
class Animal {
  speak(): string { return 'generic sound'; }
}

class Dog extends Animal {
  speak(): string { return 'woof'; }
}

const d = new Dog();
d.speak(); // Monomorphic IC, fast

// BAD: Mutating prototype invalidates all ICs globally for this class
Dog.prototype.speak = function() { return 'WOOF!'; };
// This invalidates ALL call sites that had cached Dog.prototype.speak
// V8 must re-profile and re-optimize everything

// BAD: Adding properties to Object.prototype
(Object.prototype as any).customProp = 42;
// This invalidates ALL hidden classes globally! Catastrophic for performance
```

### Accessor Properties vs Data Properties

```typescript
// Data property — simple, fast, inline-cacheable
class Point {
  x: number;
  y: number;
  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }
}

// Accessor property — getter/setter — different IC path, slower
class PointAccessor {
  private _x = 0;
  get x(): number { return this._x; } // Accessor descriptor
  set x(v: number) { this._x = v; }
}

// Mixed: class fields vs getters on the same class create different shapes
class Mixed {
  name: string = 'test'; // Data property
  get displayName(): string { return this.name.toUpperCase(); } // Accessor

  // V8 treats data properties and accessors as different entries in HC
  // but getters are handled less aggressively by the optimizer
}
```

## Array Element Kinds

Arrays use a parallel system of "element kinds" that work alongside hidden classes.

### Element Kind Lattice

```
SMI_ELEMENTS           (dense, all small integers)
    │ add float
    ▼
DOUBLE_ELEMENTS        (dense, all numbers, some are floats)
    │ add non-number
    ▼
PACKED_ELEMENTS        (dense, any JS values)
    │
    │ create hole (delete/sparse)
    ▼
HOLEY_SMI_ELEMENTS     (sparse, all SMI where present)
HOLEY_DOUBLE_ELEMENTS  (sparse, all numbers)
HOLEY_ELEMENTS         (sparse, any values)

Transitions are ONE-WAY: can never go back up the lattice!
```

```typescript
// Starting as SMI (most optimized)
const arr = [1, 2, 3]; // SMI_ELEMENTS — all small integers

// Transition to DOUBLE
arr.push(1.5); // Now DOUBLE_ELEMENTS — cannot go back to SMI
console.log(arr); // [1, 2, 3, 1.5] — double array

// Transition to PACKED
arr.push('string'); // PACKED_ELEMENTS — generic slow path

// Avoid these transitions in hot code:
// BAD: starts as SMI, transitions when float is added
function sumMixed(arr: number[]): number {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += arr[i]; // Each element access is faster on SMI/DOUBLE than PACKED
  }
  return sum;
}
const mixed = [1, 2, 3];
mixed.push(1.5); // Transitions before sumMixed is called
sumMixed(mixed); // Now operating on DOUBLE, not SMI

// GOOD: Consistent element types
const integers: number[] = [1, 2, 3, 4, 5]; // SMI
const floats: number[] = [1.0, 2.0, 3.0]; // DOUBLE (all are floats)
```

### Holes and Performance

```typescript
// Hole creation degrades element kind
const holey = [1, 2, 3];
delete holey[1]; // Creates a hole → HOLEY_SMI_ELEMENTS
// Equivalent to: holey = [1, <hole>, 3]

// Sparse initialization
const sparse = new Array(100); // All holes → HOLEY_SMI_ELEMENTS
sparse[50] = 42;

// BETTER: Dense initialization
const dense = new Array(100).fill(0); // PACKED_SMI_ELEMENTS
dense[50] = 42; // Still PACKED

// Array.from is dense
const fromRange = Array.from({ length: 100 }, (_, i) => i); // PACKED_SMI
```

## Debugging Optimization State

### Node.js Native Syntax Flags

```bash
# Run with optimization intrinsics enabled
node --allow-natives-syntax script.js
```

```javascript
// %GetOptimizationStatus returns a bitmask
// Bit 1: is function optimized by TurboFan
// Bit 4: is function interpreted (Ignition)
// Bit 8: is function a builtin
// Bit 16: has function been marked for optimization
function add(a, b) { return a + b; }

// Force interpretation and profiling
%NeverOptimizeFunction(add); // Pin to interpreter (testing)
%OptimizeFunctionOnNextCall(add); // Trigger optimization on next call

for (let i = 0; i < 1000; i++) add(i, i + 1);
add(1, 2); // This call triggers TurboFan

console.log(%GetOptimizationStatus(add)); // Should include TurboFan bit

// Check for deopt
add('a', 'b'); // Type change → deoptimization
console.log(%GetOptimizationStatus(add)); // Status changed
```

### --trace-opt and --trace-deopt

```bash
# Trace optimizations
node --trace-opt script.js 2>&1 | grep "optimizing"

# Trace deoptimizations (most useful for finding problems)
node --trace-deopt script.js 2>&1

# Output format:
# [deoptimizing (DEOPT soft): begin 0x... function add (opt #12) @0, FP to SP delta: 32]
# [deoptimizing (DEOPT soft): end 0x... function add @0 => node=2, pc=0x..., caller sp=0x...]
```

### Chrome DevTools

```
1. Open DevTools → Performance tab
2. Check "Advanced paint instrumentation"
3. Record a session
4. Look for yellow "Scripting" blocks
5. Hover to see function name and duration
6. Functions in optimized code appear as "native frames" — minimal overhead
7. Deoptimizations appear as sudden increases in scripting time

Memory tab → Heap Snapshot:
- Objects with same constructor but multiple "(Shape)" entries
  indicate shape instability
```

## Writing V8-Friendly Code

### Pattern 1: Stable Object Factories

```typescript
// BAD: Factory with conditional properties
function createItem(type: string, data: any): any {
  const item: any = { type };
  if (type === 'text') {
    item.content = data.content; // Only for text
  } else if (type === 'image') {
    item.src = data.src;   // Only for image
    item.alt = data.alt;
  }
  return item; // Multiple possible shapes!
}

// GOOD: Normalize shape with null/undefined for unused fields
function createItemGood(type: 'text' | 'image', data: any): any {
  return {
    type,
    content: type === 'text' ? data.content : null,
    src: type === 'image' ? data.src : null,
    alt: type === 'image' ? data.alt : null,
    // All properties always present → one hidden class
  };
}

// BETTER: Use separate classes per type (monomorphic call sites)
class TextItem {
  readonly type = 'text';
  constructor(public content: string) {}
}

class ImageItem {
  readonly type = 'image';
  constructor(public src: string, public alt: string) {}
}
```

### Pattern 2: Constructor Property Order

```typescript
// V8 creates hidden classes based on property ASSIGNMENT ORDER
// in constructors — not declaration order in the class body

class EventEmitter {
  // GOOD: Always assign in the same order
  constructor() {
    this.listeners = new Map(); // Always first
    this.maxListeners = 10;     // Always second
    // Never conditionally skip an assignment
  }
}

// BAD: Order depends on input
class BadWidget {
  constructor(config: any) {
    if (config.id) this.id = config.id;     // Sometimes first
    this.name = config.name;                 // Sometimes first, sometimes second
    if (config.value) this.value = config.value;
    // Creates 4 possible shapes depending on which flags are set!
  }
}

// GOOD: Always assign everything, use defaults
class GoodWidget {
  constructor(config: any) {
    this.id = config.id ?? null;       // Always first
    this.name = config.name;           // Always second
    this.value = config.value ?? null; // Always third
    // Only one possible shape!
  }
}
```

### Pattern 3: Avoiding Object Spread Side Effects

```typescript
// Object spread creates a new object — new hidden class allocation
const base = { x: 1, y: 2 };

// COST: Spread creates a new object with a fresh shape
// If used in a hot loop, this generates garbage and potentially
// different shapes if properties are added from different sources
function updateCoord(coord: { x: number, y: number }, dx: number): { x: number, y: number } {
  return { ...coord, x: coord.x + dx }; // New object, same shape IF coord is always {x,y}
}

// For truly hot paths, mutate in place (if ownership is clear)
function updateCoordMutable(coord: { x: number, y: number }, dx: number): void {
  coord.x += dx; // No allocation, no GC pressure
}

// React/immutable code: spread is fine when NOT in a tight loop
// The overhead is allocation + GC, not hidden class instability
// (since spread creates consistent shapes from consistent sources)
```

### Pattern 4: TypeScript's Role

```typescript
// TypeScript interfaces do NOT guarantee hidden class stability at runtime
// They only provide compile-time type checking

interface Point { x: number; y: number; }

function scale(p: Point, factor: number): Point {
  // TypeScript says p is always {x,y} but at runtime,
  // callers can pass {x,y,z} objects that satisfy the interface
  return { x: p.x * factor, y: p.y * factor };
}

// To actually guarantee shape stability, use class instances:
class PointClass {
  constructor(public x: number, public y: number) {}
}

function scaleClass(p: PointClass, factor: number): PointClass {
  return new PointClass(p.x * factor, p.y * factor);
  // V8 KNOWS this is always PointClass shape → monomorphic IC guaranteed
}
```

## Common Pitfalls

### Pitfall 1: Object.assign Spreading Different Shapes

```typescript
// BAD: Merging objects of different shapes
const defaults = { timeout: 5000, retries: 3 };
const userOpts = { timeout: 10000 }; // Missing retries

// result has same shape as defaults (Object.assign doesn't add new props to target)
const result = Object.assign({}, defaults, userOpts); // {timeout:10000, retries:3}
// This is OK because result always has {timeout, retries}

// DANGEROUS: When source has extra props
const extendedOpts = { timeout: 10000, verbose: true }; // Extra prop!
const result2 = Object.assign({}, defaults, extendedOpts);
// result2 = {timeout:10000, retries:3, verbose:true} — DIFFERENT SHAPE from result!
```

### Pitfall 2: Array Method Chaining

```typescript
// Each method creates a new array — potential element kind demotion
const nums = [1, 2, 3, 4, 5]; // SMI_ELEMENTS

// filter returns a new packed array of the same element kind
const evens = nums.filter(n => n % 2 === 0); // [2, 4] — still SMI

// map can change element kind if callback returns different type
const asFloats = nums.map(n => n / 2); // [0.5, 1, 1.5, 2, 2.5] — DOUBLE!
// The result is DOUBLE because of 0.5 and 1.5

// flatMap is particularly risky for element kinds
const danger = nums.flatMap(n => [n, n + 0.5]); // DOUBLE due to +0.5
```

### Pitfall 3: Using arguments Object

```typescript
// arguments is an "arguments exotic object" — special shape that breaks optimization
function slow(...args: any[]): any[] {
  return Array.from(arguments); // Creating array from arguments
  // V8 cannot optimize: arguments object has special behavior
}

// GOOD: Use rest parameters instead
function fast(...args: any[]): any[] {
  return [...args]; // Rest params are regular arrays — optimizable
}

// args object also prevents optimization when passed to other functions
function alsoSlow(a: number, b: number): number {
  return Math.max.apply(null, arguments as any); // BAD: leaks arguments
}

function alsoFast(a: number, b: number): number {
  return Math.max(a, b); // Direct reference — fine
}
```

### Pitfall 4: Numeric String Keys vs Numeric Keys

```typescript
// Arrays: numeric index vs string key affects element storage
const arr: any[] = [];
arr[0] = 'a'; // ELEMENTS storage (fast path)
arr[1] = 'b'; // ELEMENTS storage

arr['2'] = 'c'; // V8 converts '2' to numeric 2 — still ELEMENTS
arr['key'] = 'd'; // String key → PROPERTIES storage (different path!)

// An array with string keys is treated as an object with extra properties
// arr.length still works but arr['key'] is a property, not an element

// For hash-map-like usage, use Map instead
const map = new Map<string, string>();
map.set('key', 'd'); // Correct abstraction for string-keyed storage
```

## Interview Questions

### Question 1: What is a hidden class and why does V8 use them?

**Answer**: A hidden class (V8 calls it a "Map" internally) is a runtime-generated metadata structure that describes the layout of a JavaScript object — which properties it has, their memory offsets, and their attributes. V8 uses hidden classes because JavaScript objects are dynamically typed: without a fixed class, every property access would require a hashtable lookup. By tracking shape implicitly through hidden classes, V8 converts property lookups into fixed-offset memory reads (like C struct field access), achieving near-native speed for property access on shape-stable objects.

### Question 2: What is the difference between monomorphic, polymorphic, and megamorphic inline caches?

**Answer**: An inline cache (IC) is a per-call-site cache that records the hidden classes seen at that property access location.

- **Monomorphic**: Only one hidden class seen — V8 generates a single conditional check and direct offset read. Fastest.
- **Polymorphic**: 2–4 hidden classes seen — V8 generates a small dispatch table (linear or binary search over 2–4 cases). Still fast but with branching overhead.
- **Megamorphic**: 5+ hidden classes seen — V8 gives up on per-site caching and routes through a global generic lookup. Much slower. Once megamorphic, the IC cannot go back — the function should be split by type instead.

### Question 3: How does deleting a property affect performance?

**Answer**: The `delete` operator removes a property from an object and can transition the object into "dictionary mode" (also called "slow mode"). This happens because no existing hidden class matches "has properties A and C but not B." V8 cannot maintain the fast path, so it converts the object's property storage into a runtime hashmap, making all subsequent property accesses on that object significantly slower. The safer alternative is setting the property to `null` or `undefined`, which preserves the hidden class since the property still exists at its offset.

### Question 4: Why does property addition order matter?

**Answer**: Hidden class transitions are recorded per-step. An object that first adds `x` then `y` follows the transition chain `HC0 → HC_x → HC_xy`. An object that first adds `y` then `x` follows `HC0 → HC_y → HC_yx`. Even though both objects end up with properties `x` and `y`, they have different terminal hidden classes. This means they cannot share inline caches — a function that accesses `.x` on both objects will go polymorphic even though the values are the same type.

### Question 5: What are array element kinds and how do they affect performance?

**Answer**: V8 tracks the types of values stored in an array through "element kinds": `SMI_ELEMENTS` (dense, small integers), `DOUBLE_ELEMENTS` (dense, any numbers), and `PACKED_ELEMENTS` (dense, any values), with holey variants for each. V8 generates specialized machine code for each kind — SMI arrays can use integer arithmetic with no boxing, DOUBLE arrays use unboxed doubles, and PACKED arrays need full type checks. Element kind transitions are one-way and irreversible: adding a float to an SMI array permanently transitions it to DOUBLE, and adding a string permanently transitions to PACKED. Hot array-processing code should use arrays of consistent element types to stay in the fastest element kind tier.

## Key Takeaways

1. **Hidden classes are structural metadata** — they describe *which* properties an object has and *where* they live in memory, not what values those properties hold

2. **Same shape = shared hidden class** — objects with the same properties added in the same order share a hidden class and can share inline caches

3. **Property order matters** — `{x, y}` and `{y, x}` have different hidden classes even though they contain the same properties

4. **ICs are per-call-site** — every `obj.x` in your source code has its own inline cache; a function called with objects of many shapes will degrade each call site independently

5. **Delete is dangerous in hot code** — it can push objects into slow dictionary mode; prefer setting to `null`

6. **Element kinds are one-way** — adding a float to an integer array permanently downgrades it; mixing types in array-heavy code is a significant performance hazard

7. **Megamorphic ICs don't recover** — once a call site sees 5+ shapes, it stays megamorphic; fix the root cause (split the function or normalize object shapes)

8. **Prototype mutations are catastrophic** — changing `Constructor.prototype` or `Object.prototype` invalidates all ICs globally

9. **TypeScript interfaces don't guarantee shapes** — callers can pass structural subtypes with extra properties; use class instances for guaranteed shape stability

10. **Constructor property assignment order** is what V8 observes — class field declarations are syntactic sugar; the hidden class is built at runtime based on assignment order in constructor code

## Resources

### Official V8 Blog Posts
- "What's up with monomorphism?" — Vyacheslav Egorov
- "V8 Internals for JavaScript Developers" — Mathias Bynens
- "Shapes and Inline Caches" — Mathias Bynens & Benedikt Meurer (v8.dev)

### Tools
- `node --allow-natives-syntax` — `%GetOptimizationStatus`, `%HaveSameMap`
- `node --trace-opt --trace-deopt` — log optimization decisions
- Chrome DevTools Memory tab → Heap Snapshot (look for shape counts)
- `d8 --print-opt-code` — print generated machine code

### Academic
- "Inline Caching Meets Quickening" — Stefan Brunthaler
- SELF language papers (originator of hidden class concept)
