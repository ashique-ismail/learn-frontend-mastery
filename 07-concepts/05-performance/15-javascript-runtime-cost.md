# JavaScript Runtime Cost Analysis

## The Idea

**In plain English:** When a website loads JavaScript code, the browser has to do a lot of work before anything actually runs — it reads the code letter by letter, translates it into instructions the computer understands, then actually carries out those instructions. Each of those steps takes time and eats up memory, which is why JavaScript is more expensive to load than a plain image.

**Real-world analogy:** Imagine receiving a complex recipe written in a foreign language. You first have to read and translate it word by word into your own language, then figure out the cooking steps from those notes, and finally actually cook the meal — all before you can eat anything.

- The translation step = parsing (browser reads the raw text and turns it into a structured list of instructions)
- The step of planning how to cook = compiling (browser converts those instructions into a format the computer's processor can run fast)
- The actual cooking = execution (the code runs, creating data and updating the page)

---

## Overview

Every byte of JavaScript you ship has a cost that goes beyond download time. A 200KB JavaScript bundle costs more than a 200KB image of the same size. Understanding exactly what that cost is — parse, compile, execute, memory — is what lets you make principled decisions about when to add a dependency, split a bundle, or reach for a Web Worker.

---

## The Full Cost of JavaScript

```
Download → Parse → Compile → Execute → Memory retention

Image (200KB):      Download → Decode
JavaScript (200KB): Download → Parse → Compile → Execute → Heap allocation
```

A 200KB JPEG is decoded once and stored as pixels. A 200KB JavaScript bundle is parsed (read character by character into an AST), compiled (AST → bytecode, hot paths JIT'd to machine code), executed (functions called, objects created), and some of those objects remain in the heap until GC.

---

## Parse Time

V8's parser converts raw JavaScript text into an Abstract Syntax Tree (AST). This happens synchronously on the main thread.

**Approximate parse cost:** ~1ms per KB on a mid-range mobile device

```
200KB bundle → ~200ms parse time on a Moto G4 (2017 device)
              → ~50ms parse time on a MacBook M1

Parse blocks the main thread. 200ms of blocked main thread = 200ms added to TTI.
```

### Lazy Parsing

V8 uses "lazy parsing" — functions that aren't immediately called are pre-parsed (just enough to determine scope) but not fully parsed. When they're called, the full parse runs.

```javascript
// Fully parsed immediately (top-level execution)
const result = computeSomething();

// Pre-parsed (body deferred until called)
function computeSomething() {
  // ... only fully parsed when this is called
}

// IIFE: forces immediate full parse
(function() {
  // ... immediately parsed AND executed
})();
```

**Implication:** Code-splitting isn't just about download size — it also reduces parse time for the initial bundle.

---

## Compile Time

After parsing, V8 compiles the AST to bytecode (Ignition). Hot functions are additionally JIT-compiled to machine code (TurboFan). Compilation is concurrent in V8 — it happens on background threads while other work proceeds.

**Script streaming:** V8 can parse and compile scripts while they download, rather than waiting for the full download.

```html
<!-- Enables script streaming — download and parse concurrently -->
<script src="bundle.js" async></script>

<!-- Blocking — no streaming, full download before parse -->
<script src="bundle.js"></script>
```

**V8 code cache:** After the first load, V8 caches compiled bytecode so subsequent page loads skip compilation entirely. Cache is keyed on the script URL + content hash. This is why fingerprinted filenames (`bundle.3f9a.js`) are important — a new content hash invalidates the cache.

---

## Execution Time

Execution is where your application code actually runs. For the initial load, execution cost comes from:

1. Module initialization (top-level code in each module)
2. Framework bootstrap (React rendering the root, Angular bootstrapping)
3. Event listener registration
4. DOM queries and mutations during hydration

**Benchmark: Initial execution cost by framework (rough, 2024 mid-range device)**

```
Vanilla JS:           ~10ms
Preact:               ~15ms
Vue 3:                ~30ms
React 18:             ~40ms
Angular 18:           ~70ms (includes compiler runtime, DI setup)
```

These numbers shrink with code splitting — you only pay for what you bootstrap.

---

## Memory Cost

Every object created during execution lives in the V8 heap until GC collects it. Objects that escape (stored in module scope, closure, DOM, cache) live indefinitely.

**Rough heap cost per entity:**

| Entity | Approximate heap size |
|---|---|
| Empty object `{}` | ~56 bytes |
| String (per char) | ~2 bytes + 40 byte overhead |
| React fiber node | ~200 bytes |
| Angular component instance | ~300–500 bytes |
| Redux store with 1000 entities | ~500KB |
| 10,000 item virtualized list | ~2MB (with buffers) |

A 10,000-item list rendered in full DOM: ~100MB (each DOM node ~10KB). With virtualization: ~2MB. This is why virtualization matters for memory, not just render performance.

---

## The Cost of Third-Party Dependencies

### Real Costs (2024 gzipped)

| Library | Gzipped | Parse cost (Moto G4) |
|---|---|---|
| React + ReactDOM | ~45KB | ~45ms |
| Moment.js | ~72KB | ~72ms |
| Lodash (full) | ~25KB | ~25ms |
| date-fns (tree-shaken) | ~3KB (2 fns) | ~3ms |
| Chart.js | ~62KB | ~62ms |
| D3 (full) | ~80KB | ~80ms |

**The question to ask about every dependency:**
> "What is the parse + execution cost? Can I get 80% of the functionality with 20% of the cost (native APIs, smaller alternative, tree-shaking)?"

```javascript
// Moment.js: 72KB gzipped, 232KB minified
import moment from 'moment';
moment().format('MMMM Do YYYY');

// Replacement: Intl API (0KB — native)
new Intl.DateTimeFormat('en-US', { 
  month: 'long', day: 'numeric', year: 'numeric' 
}).format(new Date());

// Or date-fns (tree-shaken to ~1KB for `format` alone)
import { format } from 'date-fns';
format(new Date(), 'MMMM do yyyy');
```

---

## Measuring Runtime Cost

### Chrome DevTools

```
Performance tab → record page load → look at:
- "Parse HTML" block (HTML parsing)
- "Evaluate Script" block (each script's parse + compile + execute time)
- "Scripting" total in the pie chart (% of time in JS)
```

### `performance.measure()` for Custom Timing

```javascript
performance.mark('init:start');
initializeApp();
performance.mark('init:end');
performance.measure('app-init', 'init:start', 'init:end');

const [measure] = performance.getEntriesByName('app-init');
console.log(`App init took ${measure.duration.toFixed(2)}ms`);
```

### Resource Timing

```javascript
// Check parse + execution time per script
performance.getEntriesByType('resource')
  .filter(r => r.initiatorType === 'script')
  .map(r => ({
    name:       r.name.split('/').pop(),
    size:       (r.transferSize / 1024).toFixed(1) + 'KB',
    download:   (r.responseEnd - r.fetchStart).toFixed(0) + 'ms',
    // Note: resource timing doesn't expose parse time separately
    // Use User Timing API for that
  }));
```

---

## Practical Decision Framework

### "Should I add this dependency?"

```
1. What is its gzipped size?
2. Can I tree-shake it to only what I need?
3. What does it cost per kilobyte on a mid-range mobile device (~1ms/KB)?
4. Is there a native browser API that covers 80% of the use case?
5. Can I lazy-load it (only load when the user reaches the feature)?

Rule of thumb:
- <5KB tree-shaken: generally fine
- 5–50KB: worth a second look — can we do without or use a lighter alternative?
- >50KB: requires justification and ideally lazy loading
```

### "Should I code-split this?"

```
Split if:
- Route or feature is not needed on initial load
- The component/library is >30KB gzipped
- The feature is behind a user interaction (not visible on first render)

Don't split:
- If the async chunk loads slower than just including it (network RTT > parse savings)
- If the feature is on the critical render path
- If splitting creates too many small chunks (each HTTP request has overhead)
```

### "Should I use a Web Worker?"

```
Use a Worker if:
- The operation takes >16ms (will drop a frame)
- The operation is CPU-bound (not I/O-bound — workers don't help with fetch())
- The operation doesn't need DOM access (workers have no DOM)

Examples: JSON parsing of large payloads, image processing, encryption,
          data transformation, regex on large strings, sorting large arrays
```

---

## The "200KB of JavaScript" Interview Answer

When an interviewer asks "You're shipping 200KB of JS. What does that cost?" — here's the complete answer:

> "The download cost on a fast connection is negligible — 200KB over 10Mbps is 160ms. The real costs are:
>
> Parse: ~200ms on a mid-range mobile device, synchronous on the main thread — this directly adds to TTI.
>
> Compile: V8 does this mostly on background threads now, and bytecode caches on repeat visits, so less of a concern after the first load.
>
> Execute: depends on what the code does — but framework bootstrap, module initialization, and initial render can easily add another 100–300ms on mobile.
>
> Memory: all those objects live in the heap until GC. A 200KB bundle might allocate 10–50MB of heap during execution depending on what it creates.
>
> So the real question isn't '200KB downloaded' — it's '500ms+ of main thread work on a user's Moto G4, blocking their ability to interact with the page.' That's why I'd audit the bundle for large dependencies with smaller alternatives, add code splitting for routes not needed on initial load, and measure TTI — not just bundle size."

---

## Interview Questions

**Q: Why does a 200KB JavaScript file cost more than a 200KB image?**
A: An image is decoded once into pixels — a one-time fixed cost. JavaScript must be parsed (text to AST), compiled (AST to bytecode), and executed (code runs, objects are created and held in memory). Each step is sequential and most happen on the main thread. A 200KB image adds ~100ms decode time. A 200KB JS bundle can add 200ms parse + 150ms execute + ongoing memory pressure.

**Q: What is V8's code cache and how does it affect performance?**
A: After the first parse and compile, V8 serializes the compiled bytecode and stores it in the browser's disk cache. On subsequent loads, V8 deserializes the bytecode instead of re-parsing and re-compiling, which is ~3–5x faster. The cache is keyed on the script URL and content hash — which is why using fingerprinted filenames (e.g. `app.3f9a.js`) is important. A new deployment gets a new hash, invalidating the old cache so users get fresh code. A URL without a hash never invalidates.

**Q: When is it worth using a Web Worker for computation?**
A: When the operation takes more than ~16ms and is CPU-bound (not I/O-bound). Moving it to a Worker prevents it from blocking the main thread and dropping frames. Good candidates: parsing large JSON payloads, image processing, encryption, regex against large strings, and complex data transformations. Bad candidates: fetch requests (workers don't speed up I/O), DOM manipulation (workers have no DOM access), anything that needs to run faster than the Worker's message-passing overhead (~0.1ms per transfer).
