# Garbage Collection

## The Idea

**In plain English:** Garbage collection is how a program automatically finds and throws away data it no longer needs, freeing up memory so the computer does not run out of space. Think of "memory" as limited storage space, and "data" as stuff the program put there while running.

**Real-world analogy:** Imagine a restaurant kitchen where cooks write orders on sticky notes and stick them on a board. A cleaner walks through every few minutes, checks which sticky notes are still being used by cooks actively working, and throws away the ones nobody is looking at anymore.

- The sticky notes = objects stored in memory
- The cooks actively using a note = references (variables) pointing to an object
- The cleaner checking and removing unused notes = the garbage collector reclaiming unreachable objects

---

## Learning Objectives
- Understand how JavaScript engines automatically reclaim memory
- Master the mark-and-sweep algorithm and its phases
- Learn about generational garbage collection strategies
- Comprehend how to identify and prevent memory leaks
- Understand WeakMap, WeakSet, and their garbage collection benefits

## Introduction

**Garbage Collection (GC)** is the automatic process of reclaiming memory occupied by objects that are no longer reachable

 from the program. Unlike languages like C/C++ where manual memory management is required, JavaScript engines handle memory deallocation automatically.

Modern JavaScript engines like V8 (Chrome, Node.js) use sophisticated GC algorithms optimized for performance. Understanding GC is crucial for writing memory-efficient code and avoiding memory leaks that can degrade application performance over time.

## Reachability

The fundamental concept behind garbage collection is **reachability**: objects that can be accessed from "roots" are kept in memory; unreachable objects are eligible for garbage collection.

### Roots

**Roots** are inherently reachable references that cannot be garbage collected:

1. **Global variables** (`window`, `global`)
2. **Currently executing function** and its local variables
3. **Call stack** (all functions in the execution context chain)
4. **Other currently executing functions and their scopes**

```javascript
// Global root
let globalObject = { data: 'persistent' };

function example() {
  // Local root (while function executes)
  let localObject = { data: 'temporary' };
  
  return localObject; // If returned, caller keeps reference
  // If not used by caller, becomes unreachable after function returns
}
```

### Reachability Chain

An object is reachable if there's a chain of references from a root:

```javascript
const root = {
  child: {
    grandchild: {
      value: 42
    }
  }
};

// Reachability chain: root → child → grandchild
// All objects reachable from root
```

### Unreachable Objects

When no path exists from any root, the object becomes garbage:

```javascript
let obj = { data: 'temporary' };
obj = null; // Object no longer reachable, eligible for GC

function create() {
  let local = { data: 'short-lived' };
  // 'local' becomes unreachable when function returns (if not returned/captured)
}

create();
// 'local' object is now unreachable, will be garbage collected
```

## Mark-and-Sweep Algorithm

The **mark-and-sweep** algorithm is the foundation of most JavaScript garbage collectors. It runs in two phases:

### Phase 1: Mark

The GC traverses all reachable objects starting from roots and **marks** them:

```
1. Start with roots
2. For each root:
   - Mark the object
   - Recursively mark all objects referenced by it
3. Repeat until all reachable objects are marked
```

### Phase 2: Sweep

The GC **sweeps** through memory and reclaims unmarked objects:

```
1. Scan all heap memory
2. For each object:
   - If unmarked → deallocate (garbage)
   - If marked → unmark (prepare for next cycle)
```

### Visualization

```javascript
// Setup
let obj1 = { name: 'A' };
let obj2 = { name: 'B' };
let obj3 = { name: 'C' };

obj1.ref = obj2; // A → B
obj2.ref = obj3; // B → C

// Reachability graph
// Root → obj1 → obj2 → obj3

let obj4 = { name: 'D' }; // Separate object

// Now break the chain
obj1 = null;
obj2 = null;
obj3 = null;
```

**Before GC:**
```
ROOTS               HEAP
┌────────────┐      ┌──────────┐
│ obj1: ────────────► { name:'A' }
│            │       │  ref: ───┐
│ obj2: ──┐  │       └──────────┘│
│         │  │                   │
│ obj3: ──┼──┼────►  ┌──────────▼┐
│         │  │       │ { name:'B' }
│ obj4: ──┼──┼───┐   │  ref: ───┐│
└─────────┘  │   │   └──────────┘│
             │   │                │
             │   │   ┌──────────▼─┐
             │   │   │ { name:'C' }
             │   │   │  ref: null │
             │   │   └────────────┘
             │   │
             │   │   ┌────────────┐
             └───┼──►│ { name:'D' }
                 │   └────────────┘
                 │
After obj1/2/3 set to null:
ROOTS               HEAP
┌────────────┐      ┌──────────┐
│ obj1: null │  ╳───► { name:'A' } ← Unreachable!
│ obj2: null │      │  ref: ───┐
│ obj3: null │      └──────────┘│
│ obj4: ────────┐               │
└────────────┘ │   ┌──────────▼┐
               │   │ { name:'B' } ← Unreachable!
               │   │  ref: ───┐│
               │   └──────────┘│
               │                │
               │   ┌──────────▼─┐
               │   │ { name:'C' } ← Unreachable!
               │   │  ref: null │
               │   └────────────┘
               │
               │   ┌────────────┐
               └──►│ { name:'D' } ← Reachable!
                   └────────────┘
```

**Mark Phase:**
- Start from `obj4` (root)
- Mark `{ name: 'D' }`
- A, B, C are not reachable, remain unmarked

**Sweep Phase:**
- Deallocate A, B, C (unmarked)
- Keep D (marked)

### Circular References

Modern mark-and-sweep handles circular references correctly:

```javascript
let obj1 = {};
let obj2 = {};

obj1.ref = obj2;
obj2.ref = obj1; // Circular reference

// Both reachable from roots
console.log(obj1.ref.ref === obj1); // true

// Break reachability
obj1 = null;
obj2 = null;

// Both become unreachable despite circular reference
// GC reclaims both objects
```

**Why it works:** Mark-and-sweep doesn't count references; it checks reachability from roots. Once roots no longer reference the circular structure, it becomes garbage.

## Generational Garbage Collection

Most JavaScript engines use **generational GC** based on the observation that most objects die young (short-lived).

### Generations

**Young Generation (Nursery):**
- Newly allocated objects
- Frequent, fast GC (minor GC)
- Most objects die here

**Old Generation (Tenured):**
- Objects that survived multiple young GC cycles
- Infrequent, slower GC (major GC)
- Long-lived objects

```
┌────────────────────────────────────────┐
│           HEAP                         │
├────────────────────┬───────────────────┤
│  Young Generation  │  Old Generation   │
│  (1-8 MB)          │  (Larger)         │
│                    │                   │
│  New allocations → │  Survivors →      │
│  Fast GC (Scavenge)│  Slow GC (Mark-   │
│  Runs frequently   │  Sweep-Compact)   │
└────────────────────┴───────────────────┘
```

### V8's Generational Strategy

**Minor GC (Scavenge):**
- Runs on young generation only
- Fast (1-10ms typically)
- Uses Cheney's algorithm (copying collector)
- Promotes survivors to old generation

**Major GC (Mark-Sweep-Compact):**
- Runs on entire heap
- Slower (can be 100ms+)
- Uses incremental marking to reduce pauses
- Compacts memory to reduce fragmentation

### Object Lifecycle

```javascript
// 1. Allocation (Young Generation)
let obj = { data: 'new' };

// 2. Survives first minor GC → moved to survivor space

// 3. Survives multiple minor GCs → promoted to Old Generation

// 4. Eventually becomes unreachable → garbage collected in major GC
obj = null;
```

**Promotion thresholds:**
- Typically 2-3 minor GC cycles
- Configurable in V8 with flags

### Example: Short vs Long-lived Objects

```javascript
function processData() {
  // Short-lived: dies in young generation
  const temp = { temp: 'data' };
  const processed = process(temp);
  return processed; // temp becomes unreachable
}

// Long-lived: promoted to old generation
const cache = new Map();

function getCached(key) {
  if (!cache.has(key)) {
    cache.set(key, computeExpensive(key));
  }
  return cache.get(key);
}

// cache persists across many GC cycles → old generation
```

## GC Triggers and Timing

### When GC Runs

1. **Heap memory threshold reached**
   ```javascript
   // V8 default: ~70-80% of young generation filled
   ```

2. **Idle time** (in browsers)
   ```javascript
   // Browser runs GC during idle periods between frames
   ```

3. **Explicit request** (not guaranteed)
   ```javascript
   // Only works in Node.js with --expose-gc flag
   if (global.gc) {
     global.gc(); // Hint to run GC
   }
   ```

4. **Memory pressure**
   ```javascript
   // OS signals low memory availability
   ```

### GC Pauses

Traditional GC stops all JavaScript execution (**stop-the-world**):

```
JavaScript Execution → GC Pause → JavaScript Execution
──────────────────────┤        ├──────────────────────
                      │  STOP  │
                      │  THE   │
                      │  WORLD │
                      └────────┘
```

**Impact:**
- UI freezes (missed frames)
- Request latency spikes
- Poor user experience

### Incremental and Concurrent GC

Modern engines minimize pauses using:

**Incremental Marking:**
- Break marking into small chunks
- Interleave with JavaScript execution
- Reduces individual pause times

```
JavaScript → Mark Chunk → JavaScript → Mark Chunk → JavaScript → Sweep
───────────┤            ├───────────┤            ├───────────┤
           │ short pause│           │ short pause│           │ pause
```

**Concurrent Marking:**
- Run GC on separate thread
- JavaScript continues executing
- Minimal pause for final synchronization

```
JavaScript Thread:   ───────────────────────────────
                                                    ↓
GC Thread:           ────────[Marking]──────────→ Sweep
                                                  (short pause)
```

**Idle-time GC:**
- Browser runs GC during idle periods
- Uses `requestIdleCallback` timing
- Minimizes impact on user experience

## Memory Leaks

A **memory leak** occurs when objects remain reachable (preventing GC) despite no longer being needed.

### Common Leak Patterns

#### 1. Unintended Global Variables

```javascript
function leak() {
  // Missing 'let/const' creates global variable
  leakedData = new Array(1000000);
}

leak();
// leakedData persists globally, never GC'd
```

**Fix:**
```javascript
'use strict';

function noLeak() {
  // ReferenceError in strict mode
  leakedData = new Array(1000000);
}
```

#### 2. Forgotten Timers and Callbacks

```javascript
const data = { large: new Array(1000000) };

// Timer keeps closure alive forever
setInterval(() => {
  console.log(data.large.length);
}, 1000);

// Listener keeps closure alive until removed
element.addEventListener('click', () => {
  console.log(data.large.length);
});
```

**Fix:**
```javascript
const data = { large: new Array(1000000) };

const timerId = setInterval(() => {
  console.log(data.large.length);
}, 1000);

// Clear when no longer needed
setTimeout(() => {
  clearInterval(timerId);
}, 60000);

// Remove listener
function handler() {
  console.log(data.large.length);
}
element.addEventListener('click', handler);
// Later:
element.removeEventListener('click', handler);
```

#### 3. Detached DOM Nodes

```javascript
let detachedNodes = [];

function createElement() {
  const node = document.createElement('div');
  document.body.appendChild(node);
  
  // Remove from DOM but keep reference
  document.body.removeChild(node);
  detachedNodes.push(node); // Prevents GC!
}

createElement();
// 'node' and its entire subtree remain in memory
```

**Fix:**
```javascript
// Remove reference when done
detachedNodes = [];

// Or use WeakMap if you need to store metadata
const nodeMetadata = new WeakMap();
```

#### 4. Closures Capturing Large Scope

```javascript
function outer() {
  const huge = new Array(1000000).fill('data');
  const small = 'tiny';
  
  // Closure captures entire scope
  return function inner() {
    console.log(small); // Only uses 'small'
    // But 'huge' is kept in memory too!
  };
}

const fn = outer();
// 'huge' array persists in memory
```

**Fix:**
```javascript
function outer() {
  const huge = new Array(1000000).fill('data');
  const small = 'tiny';
  
  // Extract only what's needed
  const extracted = small;
  // 'huge' can now be GC'd
  
  return function inner() {
    console.log(extracted);
  };
}
```

#### 5. Cache Without Expiration

```javascript
const cache = {};

function getData(key) {
  if (!cache[key]) {
    cache[key] = fetchData(key); // Grows forever!
  }
  return cache[key];
}

// cache never shrinks, accumulates entries
```

**Fix:**
```javascript
// Option 1: LRU cache with size limit
class LRUCache {
  constructor(maxSize) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }
  
  get(key) {
    if (!this.cache.has(key)) return undefined;
    // Move to end (most recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }
  
  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.maxSize) {
      // Remove least recently used (first entry)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
}

const cache = new LRUCache(100);

// Option 2: WeakMap (keys can be GC'd)
const cache = new WeakMap();
```

### Detecting Memory Leaks

#### Browser DevTools

**Chrome DevTools Memory Profiler:**

1. **Heap Snapshot:**
   ```javascript
   // Take snapshot before
   performAction();
   // Take snapshot after
   // Compare to find retained objects
   ```

2. **Allocation Timeline:**
   ```javascript
   // Record allocations over time
   // Identify objects not being released
   ```

3. **Allocation Sampling:**
   ```javascript
   // Low-overhead profiling
   // Find hot allocation paths
   ```

#### Programmatic Detection

```javascript
// Node.js: Track memory usage
function checkMemory() {
  const used = process.memoryUsage();
  console.log({
    rss: `${Math.round(used.rss / 1024 / 1024)} MB`,          // Total
    heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)} MB`, // Allocated
    heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)} MB`,   // Actually used
    external: `${Math.round(used.external / 1024 / 1024)} MB`    // C++ objects
  });
}

setInterval(checkMemory, 5000);
```

#### Memory Leak Indicators

- **Heap size grows continuously**
- **Sawtooth pattern flattens** (heap not decreasing after GC)
- **GC frequency increases**
- **Application slows over time**

```
Healthy Pattern:
Heap Size
  ┃   ╱╲    ╱╲    ╱╲    ╱╲
  ┃  ╱  ╲  ╱  ╲  ╱  ╲  ╱  ╲
  ┃ ╱    ╲╱    ╲╱    ╲╱    ╲
  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━→ Time
     ↑      ↑      ↑      ↑
    GC     GC     GC     GC

Memory Leak Pattern:
Heap Size
  ┃                   ╱──────
  ┃              ╱───╱
  ┃         ╱───╱
  ┃    ╱───╱
  ┃───╱
  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━→ Time
  Baseline keeps rising despite GC
```

## WeakMap and WeakSet

**WeakMap** and **WeakSet** hold "weak" references that don't prevent garbage collection.

### WeakMap

Keys must be objects; if no other references exist to the key, it can be GC'd:

```javascript
let obj = { data: 'important' };
const metadata = new WeakMap();

metadata.set(obj, { timestamp: Date.now() });

console.log(metadata.get(obj)); // { timestamp: ... }

// Remove reference
obj = null;

// 'obj' and its metadata can now be GC'd
// WeakMap entry automatically removed
```

**Use cases:**
- Storing private data
- Caching computed values
- Tracking object metadata

```javascript
// Private data with WeakMap
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    this.name = name;
    privateData.set(this, { password }); // Private!
  }
  
  checkPassword(input) {
    return privateData.get(this).password === input;
  }
}

const user = new User('Alice', 'secret123');
console.log(user.password); // undefined
console.log(user.checkPassword('secret123')); // true

// When 'user' is GC'd, private data is automatically removed
```

### WeakSet

Similar to WeakMap but for sets:

```javascript
const visited = new WeakSet();

function processNode(node) {
  if (visited.has(node)) return;
  visited.add(node);
  
  // Process node
  console.log('Processing:', node.id);
  
  // Process children
  node.children.forEach(child => processNode(child));
}

// Nodes can be GC'd even if visited
```

### WeakMap vs Map

**Map:**
```javascript
const map = new Map();
let obj = { data: 'important' };

map.set(obj, 'value');
obj = null; // Object NOT GC'd (Map holds strong reference)

console.log(map.size); // 1 (entry still exists)
```

**WeakMap:**
```javascript
const weakMap = new WeakMap();
let obj = { data: 'important' };

weakMap.set(obj, 'value');
obj = null; // Object CAN be GC'd (weak reference)

// Entry automatically removed when obj is GC'd
// Can't check size (no .size property)
```

**Limitations of WeakMap/WeakSet:**
- Keys must be objects (not primitives)
- Not enumerable (can't iterate)
- No `.size`, `.clear()`, or `.keys()` methods
- Can't inspect contents (no way to list all entries)

## Performance Optimization Strategies

### 1. Object Pooling

Reuse objects instead of creating/destroying:

```javascript
class ObjectPool {
  constructor(factory, resetFn) {
    this.factory = factory;
    this.resetFn = resetFn;
    this.pool = [];
  }
  
  acquire() {
    return this.pool.pop() || this.factory();
  }
  
  release(obj) {
    this.resetFn(obj);
    this.pool.push(obj);
  }
}

// Usage: Particle system
const particlePool = new ObjectPool(
  () => ({ x: 0, y: 0, vx: 0, vy: 0 }),
  (p) => { p.x = 0; p.y = 0; p.vx = 0; p.vy = 0; }
);

function createParticle() {
  const particle = particlePool.acquire();
  particle.x = Math.random() * 100;
  particle.y = Math.random() * 100;
  return particle;
}

function destroyParticle(particle) {
  particlePool.release(particle); // Reuse instead of GC
}
```

### 2. Avoid Creating Temporary Objects

```javascript
// Bad: Creates temporary objects
function distance(p1, p2) {
  const dx = p2.x - p1.x;
  const dy = p2.y - p1.y;
  return { distance: Math.sqrt(dx**2 + dy**2), dx, dy }; // New object!
}

// Good: Return primitive or reuse object
function distance(p1, p2) {
  const dx = p2.x - p1.x;
  const dy = p2.y - p1.y;
  return Math.sqrt(dx**2 + dy**2);
}

// Or mutate existing result object
function distance(p1, p2, result) {
  result.dx = p2.x - p1.x;
  result.dy = p2.y - p1.y;
  result.distance = Math.sqrt(result.dx**2 + result.dy**2);
  return result;
}
```

### 3. Use Typed Arrays for Numeric Data

Typed arrays are allocated outside main heap, less GC pressure:

```javascript
// Bad: Regular array (heap allocated, GC overhead)
const numbers = new Array(1000000);
for (let i = 0; i < numbers.length; i++) {
  numbers[i] = i;
}

// Good: Typed array (efficient, less GC pressure)
const numbers = new Float64Array(1000000);
for (let i = 0; i < numbers.length; i++) {
  numbers[i] = i;
}
```

### 4. Clear References When Done

```javascript
// Bad: Keeps reference
let largeData = new Array(1000000);
// ... use largeData
// largeData persists

// Good: Clear reference
let largeData = new Array(1000000);
// ... use largeData
largeData = null; // Allow GC
```

### 5. Batch DOM Operations

```javascript
// Bad: Triggers GC frequently
for (let i = 0; i < 1000; i++) {
  const div = document.createElement('div');
  div.textContent = i;
  document.body.appendChild(div); // Frequent DOM updates
}

// Good: Batch with DocumentFragment
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const div = document.createElement('div');
  div.textContent = i;
  fragment.appendChild(div);
}
document.body.appendChild(fragment); // Single DOM update
```

### 6. Avoid Large Closures in Loops

```javascript
// Bad: Creates N closures
function createHandlers(count) {
  const handlers = [];
  const largeData = new Array(1000000);
  
  for (let i = 0; i < count; i++) {
    handlers.push(() => {
      console.log(i, largeData[0]); // Each closure captures largeData
    });
  }
  
  return handlers;
}

// Good: Extract shared data
function createHandlers(count) {
  const handlers = [];
  const largeData = new Array(1000000);
  const sharedValue = largeData[0]; // Extract what's needed
  
  for (let i = 0; i < count; i++) {
    const index = i;
    handlers.push(() => {
      console.log(index, sharedValue);
    });
  }
  
  return handlers;
}
```

## Common Misconceptions

### ❌ Misconception 1: Setting to `null` immediately frees memory

```javascript
let obj = { huge: new Array(1000000) };
obj = null; // Eligible for GC, but not immediately freed
```

**✅ Reality:** Setting to `null` makes the object eligible for GC, but the GC runs on its own schedule. Memory isn't freed until the next GC cycle.

### ❌ Misconception 2: Circular references always cause memory leaks

```javascript
let obj1 = {};
let obj2 = {};
obj1.ref = obj2;
obj2.ref = obj1; // Circular reference

obj1 = null;
obj2 = null; // Both can be GC'd
```

**✅ Reality:** Modern GC (mark-and-sweep) handles circular references correctly. Old reference-counting GC had this problem, but it's solved in modern engines.

### ❌ Misconception 3: WeakMap automatically cleans up values

```javascript
const weakMap = new WeakMap();
let key = {};
let value = { huge: new Array(1000000) };

weakMap.set(key, value);
key = null; // Key can be GC'd, entry removed
value = null; // But value is only GC'd if key is collected first
```

**✅ Reality:** WeakMap allows keys to be GC'd, but values must also become unreachable. If the value references the key, they're GC'd together.

### ❌ Misconception 4: Small objects don't matter

```javascript
// "It's just a small object"
for (let i = 0; i < 1000000; i++) {
  const small = { id: i }; // 1 million small objects!
}
```

**✅ Reality:** Even small objects add up. Millions of small allocations cause GC pressure and performance issues.

### ❌ Misconception 5: GC can be manually controlled

```javascript
// Won't work in browsers
if (global.gc) {
  global.gc(); // Only available with Node.js --expose-gc flag
}
```

**✅ Reality:** GC is automatic and mostly non-controllable. You can hint or influence, but can't reliably force GC.

## Interview Questions

**Q1: Explain how the mark-and-sweep algorithm works.**

**A:**

**Mark-and-sweep** is a garbage collection algorithm that reclaims unreachable objects in two phases:

**Phase 1: Mark**
1. Start from GC roots (global variables, call stack, etc.)
2. Traverse all reachable objects recursively
3. Mark each reachable object as "alive"

**Phase 2: Sweep**
1. Scan through heap memory
2. Deallocate unmarked objects (garbage)
3. Unmark marked objects (prepare for next cycle)

**Example:**
```javascript
let obj1 = { name: 'A' };
let obj2 = { name: 'B' };
obj1.ref = obj2;

let obj3 = { name: 'C' }; // Separate object

// Reachability: Root → obj1 → obj2, Root → obj3

obj1 = null;
obj2 = null;

// Now only obj3 is reachable
// GC marks obj3, sweeps obj1 and obj2
```

**Advantages:**
- Handles circular references
- Simple conceptually

**Disadvantages:**
- Stop-the-world pauses (mitigated by incremental/concurrent GC)
- Doesn't compact memory (addressed by mark-sweep-compact variant)

**Q2: What is generational garbage collection and why is it effective?**

**A:**

**Generational GC** divides heap into generations based on object age, optimizing for the observation that "most objects die young."

**Generations:**

1. **Young Generation (Nursery):**
   - New allocations
   - Frequent, fast GC (minor GC)
   - Most objects die here (90%+)

2. **Old Generation (Tenured):**
   - Objects that survived multiple young GCs
   - Infrequent, slower GC (major GC)
   - Long-lived objects

**Why effective:**

1. **Most objects die young:** Focusing GC on young generation catches most garbage
2. **Faster collection:** Young generation is smaller, faster to scan
3. **Less copying:** Don't need to scan old generation frequently

**Example:**
```javascript
function processRequest() {
  const temp = { data: 'temporary' }; // Dies in young generation
  return processData(temp);
}

const cache = new Map(); // Lives long, promoted to old generation
```

**Trade-offs:**
- More complex implementation
- Write barriers needed (track old→young references)
- Major GC still expensive

**Q3: What are common causes of memory leaks in JavaScript?**

**A:**

**1. Unintended Global Variables:**
```javascript
function leak() {
  forgotten = 'global'; // Missing let/const
}
```

**2. Forgotten Timers/Listeners:**
```javascript
setInterval(() => {
  const data = fetchData(); // Accumulates forever
}, 1000);
// Never cleared
```

**3. Closures Capturing Large Scope:**
```javascript
function outer() {
  const huge = new Array(1000000);
  return () => console.log('hi'); // Captures entire scope
}
```

**4. Detached DOM Nodes:**
```javascript
const nodes = [];
const node = document.createElement('div');
document.body.appendChild(node);
document.body.removeChild(node);
nodes.push(node); // Keeps detached node alive
```

**5. Cache Without Expiration:**
```javascript
const cache = {};
function getData(key) {
  if (!cache[key]) {
    cache[key] = fetchData(key); // Grows forever
  }
  return cache[key];
}
```

**Prevention:**
- Use strict mode
- Clear timers/listeners
- Use WeakMap/WeakSet for caches
- Null out references when done
- Profile with DevTools

**Q4: How do WeakMap and WeakSet help prevent memory leaks?**

**A:**

**WeakMap** and **WeakSet** hold "weak" references that don't prevent garbage collection.

**Regular Map (strong reference):**
```javascript
const map = new Map();
let obj = { data: 'important' };

map.set(obj, 'value');
obj = null; // Object NOT GC'd (Map holds strong reference)
```

**WeakMap (weak reference):**
```javascript
const weakMap = new WeakMap();
let obj = { data: 'important' };

weakMap.set(obj, 'value');
obj = null; // Object CAN be GC'd
// WeakMap entry automatically removed
```

**Benefits:**

1. **Automatic cleanup:** No need to manually remove entries
2. **Prevents leaks:** Objects can be GC'd even if referenced by WeakMap
3. **Private data:** Store metadata without preventing GC

**Use cases:**

1. **Private data:**
```javascript
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    this.name = name;
    privateData.set(this, { password });
  }
}
```

2. **Caching:**
```javascript
const cache = new WeakMap();

function compute(obj) {
  if (!cache.has(obj)) {
    cache.set(obj, expensiveComputation(obj));
  }
  return cache.get(obj);
}
// Cache entries auto-removed when objects are GC'd
```

3. **Tracking objects:**
```javascript
const visited = new WeakSet();

function processNode(node) {
  if (visited.has(node)) return;
  visited.add(node);
  // Process...
}
```

**Limitations:**
- Keys must be objects
- Not enumerable
- No `.size` or `.clear()`

**Q5: What techniques can reduce garbage collection pressure?**

**A:**

**1. Object Pooling:**
Reuse objects instead of creating new ones.

```javascript
class Pool {
  constructor(factory) {
    this.factory = factory;
    this.pool = [];
  }
  
  acquire() {
    return this.pool.pop() || this.factory();
  }
  
  release(obj) {
    this.pool.push(obj);
  }
}
```

**2. Avoid Temporary Objects:**
```javascript
// Bad
function distance(p1, p2) {
  return { dx: p2.x - p1.x, dy: p2.y - p1.y }; // Temp object
}

// Good
function distance(p1, p2) {
  return Math.sqrt((p2.x - p1.x)**2 + (p2.y - p1.y)**2);
}
```

**3. Use Typed Arrays:**
```javascript
// Less GC pressure
const numbers = new Float64Array(1000000);
```

**4. Batch Operations:**
```javascript
// Bad: Create many objects in loop
for (let i = 0; i < 1000; i++) {
  process({ id: i });
}

// Good: Reuse object
const obj = { id: 0 };
for (let i = 0; i < 1000; i++) {
  obj.id = i;
  process(obj);
}
```

**5. Clear References:**
```javascript
let largeData = new Array(1000000);
// ... use data
largeData = null; // Allow GC
```

**6. Minimize Closure Scope:**
```javascript
// Bad: Closure captures large scope
function outer() {
  const huge = new Array(1000000);
  return () => console.log('hi'); // Captures 'huge'
}

// Good: Extract only needed data
function outer() {
  const huge = new Array(1000000);
  const needed = huge[0];
  return () => console.log(needed);
}
```

**7. Use WeakMap for Caches:**
```javascript
const cache = new WeakMap(); // Auto-cleanup
```

**Q6: How can you detect memory leaks in a JavaScript application?**

**A:**

**Browser DevTools:**

**1. Heap Snapshot:**
- Take snapshot before action
- Perform action
- Take snapshot after
- Compare snapshots to find retained objects

```javascript
// In Chrome DevTools:
// Memory tab → Heap Snapshot → Take Snapshot
performAction();
// Take another snapshot → Compare
```

**2. Allocation Timeline:**
- Record allocations over time
- Identify objects not being released
- Look for continual memory growth

**3. Allocation Sampling:**
- Low-overhead profiling
- Find hot allocation paths

**Programmatic (Node.js):**

```javascript
function checkMemory() {
  const used = process.memoryUsage();
  console.log({
    heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)} MB`,
    heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)} MB`,
    external: `${Math.round(used.external / 1024 / 1024)} MB`
  });
}

setInterval(checkMemory, 5000);
```

**Indicators of Leaks:**

1. **Heap size grows continuously**
2. **Sawtooth pattern flattens** (heap not decreasing after GC)
3. **GC frequency increases**
4. **Application slows over time**
5. **Unexpected object retention** in heap snapshots

**Memory Leak Pattern:**
```
Heap Size
  ┃                   ╱──────  ← Keeps growing
  ┃              ╱───╱
  ┃         ╱───╱
  ┃    ╱───╱
  ┃───╱
  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━→ Time
```

**Tools:**
- Chrome DevTools Memory Profiler
- Node.js `--inspect` + Chrome DevTools
- `heapdump` npm package
- `memwatch-next` npm package

**Q7: Explain the difference between minor and major garbage collection.**

**A:**

JavaScript engines use generational GC with two types of collection:

**Minor GC (Scavenge):**

**Target:** Young generation only
**Frequency:** High (every few seconds)
**Duration:** Fast (1-10ms)
**Algorithm:** Cheney's copying collector
**Purpose:** Collect short-lived objects

```javascript
function processRequest() {
  const temp = { data: 'temporary' }; // Dies in minor GC
  return processData(temp);
}

// temp object allocated in young generation
// Becomes unreachable after function returns
// Collected in next minor GC
```

**Major GC (Mark-Sweep-Compact):**

**Target:** Entire heap (young + old generation)
**Frequency:** Low (minutes/hours)
**Duration:** Slow (can be 100ms+)
**Algorithm:** Mark-sweep with compaction
**Purpose:** Collect long-lived objects and defragment

```javascript
const cache = new Map(); // Long-lived, promotes to old generation

function cached(key) {
  if (!cache.has(key)) {
    cache.set(key, compute(key));
  }
  return cache.get(key);
}

// cache persists, only collected if made unreachable
// Collected in major GC
```

**Comparison:**

| Aspect | Minor GC | Major GC |
|--------|----------|----------|
| Target | Young generation | Entire heap |
| Frequency | High | Low |
| Speed | Fast (1-10ms) | Slow (100ms+) |
| Algorithm | Copying | Mark-Sweep-Compact |
| Objects | Short-lived | Long-lived |
| Impact | Low pause | Higher pause |

**Optimization:**
Modern engines use incremental/concurrent marking for major GC to reduce pause times.

**Q8: What is the relationship between closures and garbage collection?**

**A:**

**Closures** keep outer scope variables alive in memory, preventing garbage collection until the closure itself becomes unreachable.

**Basic Example:**
```javascript
function outer() {
  let count = 0; // Local variable
  
  return function inner() {
    count++; // Closure captures 'count'
    return count;
  };
}

const counter = createCounter();
// 'count' cannot be GC'd while 'counter' exists
```

**Memory Lifecycle:**

1. **outer() called:** `count` allocated on stack
2. **inner() returned:** `count` moved to heap (closure captures it)
3. **outer() completes:** Stack frame removed, but `count` persists
4. **counter referenced:** `count` remains in memory
5. **counter = null:** `count` becomes unreachable, eligible for GC

**Implications:**

**1. Closures extend object lifetime:**
```javascript
function createHandler() {
  const largeData = new Array(1000000);
  
  return function handler() {
    console.log(largeData[0]);
  };
}

const fn = createHandler();
// largeData kept in memory as long as 'fn' exists
```

**2. Multiple closures share scope:**
```javascript
function createClosures() {
  const shared = { count: 0 };
  
  return {
    increment: () => shared.count++,
    decrement: () => shared.count--,
    get: () => shared.count
  };
}

const closures = createClosures();
// All three closures share same 'shared' object
// 'shared' kept alive until all closures are GC'd
```

**3. Scope is captured wholesale:**
```javascript
function outer() {
  const huge = new Array(1000000);
  const small = 'tiny';
  
  return function inner() {
    console.log(small); // Only uses 'small'
  };
}

// But entire scope (including 'huge') is captured!
```

**Optimization:**
```javascript
function outer() {
  const huge = new Array(1000000);
  const small = 'tiny';
  
  // Extract only what's needed before creating closure
  const extracted = small;
  
  return function inner() {
    console.log(extracted);
  };
  // 'huge' can now be GC'd
}
```

**Memory Leak Prevention:**
```javascript
// Bad: Event listener closure prevents GC
element.addEventListener('click', () => {
  const data = fetchLargeData();
  processData(data); // 'data' kept alive by closure
});

// Good: Clear reference or remove listener
const handler = () => {
  const data = fetchLargeData();
  processData(data);
};
element.addEventListener('click', handler);
// Later:
element.removeEventListener('click', handler);
```

## Key Takeaways

- **Garbage Collection** automatically reclaims memory from unreachable objects
- **Mark-and-sweep** algorithm traverses from roots, marks reachable objects, sweeps unreachable ones
- **Generational GC** optimizes for "most objects die young" observation
- **Minor GC** (young generation, fast) vs **Major GC** (entire heap, slower)
- Modern engines use **incremental/concurrent GC** to minimize pauses
- **Memory leaks** occur when objects remain reachable despite not being needed
- Common leaks: unintended globals, forgotten timers, closures capturing large scopes, detached DOM nodes
- **WeakMap/WeakSet** allow objects to be GC'd even if referenced
- **Object pooling**, avoiding temporary objects, and typed arrays reduce GC pressure
- Use DevTools memory profiler to detect leaks and optimize memory usage

## Resources

- [MDN - Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [V8 Blog - Trash Talk: Orinoco Garbage Collector](https://v8.dev/blog/trash-talk)
- [V8 Blog - Concurrent Marking](https://v8.dev/blog/concurrent-marking)
- [Chrome DevTools - Fix Memory Problems](https://developer.chrome.com/docs/devtools/memory-problems/)
- [JavaScript.info - Garbage Collection](https://javascript.info/garbage-collection)
- [A tour of V8: Garbage Collection](http://jayconrod.com/posts/55/a-tour-of-v8-garbage-collection)
- [ECMAScript Specification - WeakMap](https://tc39.es/ecma262/#sec-weakmap-objects)
- [MDN - WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
