# Proxy Pattern

## The Idea

**In plain English:** A proxy is a stand-in object that sits between you and the real thing, intercepting every request you make before deciding whether to pass it along, block it, or do some extra work first. Think of it as a middleman that looks identical to the original but has secret powers to control what happens.

**Real-world analogy:** Imagine a personal assistant who answers your calls at work. Anyone who wants to reach you dials the same number and hears the same voice — they have no idea they are talking to an assistant instead of you. The assistant can screen calls, take messages, or patch callers straight through.

- The caller = the client code making requests
- The assistant = the proxy object (same interface, but in the middle)
- You (the real person) = the real subject doing the actual work
- The phone number = the shared interface that makes the swap invisible

---

## Overview

The Proxy pattern is a structural design pattern that provides a **surrogate or placeholder** for another object to control access to it. The proxy sits between the client and the real subject, intercepting operations and either forwarding them, transforming them, or blocking them entirely.

Unlike the Decorator (which *adds* behaviour) or the Facade (which *simplifies* an interface), the Proxy maintains the **exact same interface** as the subject while governing how and when the real object is accessed.

```
┌──────────────┐          ┌─────────────────┐          ┌──────────────┐
│    Client    │─────────▶│     Proxy        │─────────▶│ Real Subject │
└──────────────┘  request ├─────────────────┤  forward  └──────────────┘
                          │ - realSubject    │
                          │ + request()      │  (same interface as
                          │   pre-logic      │   Real Subject)
                          │   realSubject    │
                          │     .request()   │
                          │   post-logic     │
                          └─────────────────┘
```

### Subject Interface Contract

The key invariant: **client code cannot tell it is talking to a proxy**. Both proxy and real subject implement the same interface.

```typescript
interface Subject {
  request(data: unknown): unknown;
}

class RealSubject implements Subject {
  request(data: unknown) { /* real work */ }
}

class ProxySubject implements Subject {      // same interface
  private real = new RealSubject();
  request(data: unknown) {
    // intercept, then delegate
    return this.real.request(data);
  }
}
```

---

## Types of Proxy

### 1. Virtual Proxy — Lazy Initialization

Delays creation of an expensive object until it is first accessed. The client sees the interface immediately; the heavy resource is only allocated on first use.

```typescript
// Expensive resource: a high-resolution image loaded from disk
interface Image {
  render(): string;
  getDimensions(): { width: number; height: number };
}

class HighResImage implements Image {
  private data: string;
  private width: number;
  private height: number;

  constructor(private filename: string) {
    // Simulates expensive I/O
    console.log(`[HighResImage] Loading ${filename} from disk...`);
    this.data = `<binary data of ${filename}>`;
    this.width = 4096;
    this.height = 2160;
  }

  render(): string {
    return `Rendering ${this.filename}: ${this.data}`;
  }

  getDimensions() {
    return { width: this.width, height: this.height };
  }
}

class LazyImageProxy implements Image {
  private real: HighResImage | null = null;

  constructor(private filename: string) {
    // No I/O here — object construction is instant
    console.log(`[LazyImageProxy] Proxy created for ${filename} (no load yet)`);
  }

  private ensureLoaded(): HighResImage {
    if (!this.real) {
      this.real = new HighResImage(this.filename);
    }
    return this.real;
  }

  render(): string {
    return this.ensureLoaded().render();
  }

  getDimensions() {
    return this.ensureLoaded().getDimensions();
  }
}

// Usage
const img = new LazyImageProxy("hero-banner.png"); // instant
console.log("Proxy created, no I/O yet");
console.log(img.getDimensions()); // triggers load NOW
console.log(img.render());         // uses already-loaded instance
```

**Real-world uses:** lazy-loaded React components (`React.lazy`), database connection pools, ORM entity loading (Hibernate, Sequelize associations).

---

### 2. Protection Proxy — Access Control

Wraps an object and enforces permissions before delegating. The real subject has no knowledge of authorization rules — that responsibility lives entirely in the proxy.

```typescript
type Role = "admin" | "editor" | "viewer";

interface DocumentService {
  read(docId: string): string;
  write(docId: string, content: string): void;
  delete(docId: string): void;
}

class DocumentServiceImpl implements DocumentService {
  private docs = new Map<string, string>([
    ["doc-1", "Confidential Q3 report"],
    ["doc-2", "Public press release"],
  ]);

  read(docId: string): string {
    return this.docs.get(docId) ?? "Not found";
  }

  write(docId: string, content: string): void {
    this.docs.set(docId, content);
    console.log(`[DocService] Written to ${docId}`);
  }

  delete(docId: string): void {
    this.docs.delete(docId);
    console.log(`[DocService] Deleted ${docId}`);
  }
}

class ProtectedDocumentProxy implements DocumentService {
  private real = new DocumentServiceImpl();

  constructor(private userRole: Role) {}

  read(docId: string): string {
    // All roles can read
    return this.real.read(docId);
  }

  write(docId: string, content: string): void {
    if (this.userRole !== "admin" && this.userRole !== "editor") {
      throw new Error(`Access denied: role '${this.userRole}' cannot write`);
    }
    this.real.write(docId, content);
  }

  delete(docId: string): void {
    if (this.userRole !== "admin") {
      throw new Error(`Access denied: only admins can delete documents`);
    }
    this.real.delete(docId);
  }
}

// Viewer: read-only access
const viewerProxy = new ProtectedDocumentProxy("viewer");
console.log(viewerProxy.read("doc-1"));     // OK
viewerProxy.write("doc-1", "hacked");       // throws
viewerProxy.delete("doc-1");               // throws

// Admin: full access
const adminProxy = new ProtectedDocumentProxy("admin");
adminProxy.write("doc-1", "Updated Q3");    // OK
adminProxy.delete("doc-2");               // OK
```

---

### 3. Remote Proxy

Represents an object in a different address space (server, worker thread, iframe). The client calls local methods; the proxy handles serialization and network/IPC communication transparently.

```typescript
// Conceptual — mirrors the pattern used by gRPC stubs, web worker bridges
interface CalculatorService {
  add(a: number, b: number): Promise<number>;
  multiply(a: number, b: number): Promise<number>;
}

// This runs in the main thread but delegates work to a Web Worker
class WorkerCalculatorProxy implements CalculatorService {
  private worker: Worker;
  private pending = new Map<string, (result: number) => void>();

  constructor(workerUrl: string) {
    this.worker = new Worker(workerUrl);
    this.worker.onmessage = ({ data }) => {
      const { id, result } = data as { id: string; result: number };
      this.pending.get(id)?.(result);
      this.pending.delete(id);
    };
  }

  private call(method: string, args: number[]): Promise<number> {
    const id = crypto.randomUUID();
    return new Promise((resolve) => {
      this.pending.set(id, resolve);
      this.worker.postMessage({ id, method, args });
    });
  }

  add(a: number, b: number): Promise<number> {
    return this.call("add", [a, b]);
  }

  multiply(a: number, b: number): Promise<number> {
    return this.call("multiply", [a, b]);
  }
}

// Usage is clean — no serialization details leak to the caller
const calc = new WorkerCalculatorProxy("/calc-worker.js");
const result = await calc.add(3, 4); // 7, computed off main thread
```

**Real-world uses:** tRPC client stubs, Apollo Client network layer, Comlink (web worker RPC), Angular `HttpClient` (a remote proxy over XHR/Fetch).

---

### 4. Logging / Monitoring Proxy

Records every operation on the real subject for debugging, auditing, or performance profiling — without modifying the subject.

```typescript
interface UserRepository {
  findById(id: string): Promise<{ id: string; name: string } | null>;
  save(user: { id: string; name: string }): Promise<void>;
}

class UserRepositoryImpl implements UserRepository {
  private store = new Map<string, { id: string; name: string }>();

  async findById(id: string) {
    return this.store.get(id) ?? null;
  }

  async save(user: { id: string; name: string }) {
    this.store.set(user.id, user);
  }
}

class LoggingRepositoryProxy implements UserRepository {
  constructor(private real: UserRepository, private logger = console) {}

  async findById(id: string) {
    const start = performance.now();
    try {
      const result = await this.real.findById(id);
      const ms = (performance.now() - start).toFixed(2);
      this.logger.log(`findById(${id}) → ${result ? "HIT" : "MISS"} [${ms}ms]`);
      return result;
    } catch (err) {
      this.logger.error(`findById(${id}) → ERROR`, err);
      throw err;
    }
  }

  async save(user: { id: string; name: string }) {
    const start = performance.now();
    try {
      await this.real.save(user);
      const ms = (performance.now() - start).toFixed(2);
      this.logger.log(`save(${user.id}) → OK [${ms}ms]`);
    } catch (err) {
      this.logger.error(`save(${user.id}) → ERROR`, err);
      throw err;
    }
  }
}

const repo = new LoggingRepositoryProxy(new UserRepositoryImpl());
await repo.save({ id: "u1", name: "Alice" });
await repo.findById("u1");  // logs: findById(u1) → HIT [0.12ms]
await repo.findById("u99"); // logs: findById(u99) → MISS [0.08ms]
```

---

## JavaScript `Proxy` Object and `Reflect` API

ES2015 introduced a native `Proxy` object that intercepts fundamental language operations at the JavaScript engine level. This is the mechanism underpinning Vue 3's reactivity, Valtio, Immer's draft system, and MobX 6+.

### Syntax

```javascript
const proxy = new Proxy(target, handler);
```

- `target` — any object, function, array, or even another Proxy
- `handler` — an object whose methods are **traps** (interceptors)

### Core Traps

```
┌─────────────────────┬──────────────────────────────────────────────────────┐
│ Trap                │ Intercepted Operation                                │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ get                 │ property read: obj.x, obj[key]                       │
│ set                 │ property write: obj.x = val, obj[key] = val          │
│ has                 │ in operator: 'x' in obj                              │
│ deleteProperty      │ delete obj.x                                         │
│ apply               │ function call: fn(), fn.call(), fn.apply()           │
│ construct           │ new fn()                                             │
│ getPrototypeOf      │ Object.getPrototypeOf(obj)                           │
│ setPrototypeOf      │ Object.setPrototypeOf(obj, proto)                    │
│ defineProperty      │ Object.defineProperty(obj, key, desc)               │
│ getOwnPropertyDesc  │ Object.getOwnPropertyDescriptor(obj, key)           │
│ ownKeys             │ Object.keys(), Object.getOwnPropertyNames()          │
│ isExtensible        │ Object.isExtensible(obj)                             │
│ preventExtensions   │ Object.preventExtensions(obj)                        │
└─────────────────────┴──────────────────────────────────────────────────────┘
```

### `Reflect` API

`Reflect` mirrors every trap as a static method. Its primary job in proxy handlers is to perform the **default behaviour** after your custom logic — this is critical for maintaining invariant correctness.

```javascript
const handler = {
  get(target, prop, receiver) {
    console.log(`Reading: ${String(prop)}`);
    return Reflect.get(target, prop, receiver); // default get
  },
  set(target, prop, value, receiver) {
    console.log(`Writing: ${String(prop)} = ${String(value)}`);
    return Reflect.set(target, prop, value, receiver); // default set
  },
};
```

Why `Reflect` over direct access (`target[prop]`)?
- `receiver` is forwarded correctly for prototype chains and getters/setters
- `Reflect.set` returns a boolean (success/failure) matching the trap's required return type
- Avoids subtle bugs when the target has inherited getters that reference `this`

---

## Implementation Examples

### Validation Proxy

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

function createValidatingProxy<T extends object>(
  target: T,
  validators: Partial<Record<keyof T, (value: unknown) => string | null>>
): T {
  return new Proxy(target, {
    set(obj, prop, value, receiver) {
      const validate = validators[prop as keyof T];
      if (validate) {
        const error = validate(value);
        if (error) {
          throw new TypeError(`Validation failed for '${String(prop)}': ${error}`);
        }
      }
      return Reflect.set(obj, prop, value, receiver);
    },
  });
}

const user = createValidatingProxy<User>(
  { name: "", age: 0, email: "" },
  {
    name: (v) => (typeof v === "string" && v.length >= 2 ? null : "Must be >= 2 chars"),
    age: (v) => (typeof v === "number" && v >= 0 && v <= 150 ? null : "Must be 0–150"),
    email: (v) =>
      typeof v === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v)
        ? null
        : "Invalid email",
  }
);

user.name = "Alice";    // OK
user.age = 30;          // OK
user.email = "bad";     // TypeError: Validation failed for 'email': Invalid email
```

---

### Logging Proxy (generic, using JS Proxy)

```javascript
function createLoggingProxy(target, label = "obj") {
  return new Proxy(target, {
    get(obj, prop, receiver) {
      const value = Reflect.get(obj, prop, receiver);
      if (typeof value === "function") {
        return function (...args) {
          console.log(`[${label}] Calling ${String(prop)}(${args.join(", ")})`);
          const result = value.apply(this === proxy ? obj : this, args);
          console.log(`[${label}] ${String(prop)} returned`, result);
          return result;
        };
      }
      console.log(`[${label}] GET ${String(prop)} =`, value);
      return value;
    },
    set(obj, prop, value, receiver) {
      console.log(`[${label}] SET ${String(prop)} =`, value);
      return Reflect.set(obj, prop, value, receiver);
    },
  });
}

const store = createLoggingProxy({ count: 0 }, "store");
store.count;           // [store] GET count = 0
store.count = 5;       // [store] SET count = 5
```

---

### Memoization Proxy

```typescript
function memoize<T extends (...args: unknown[]) => unknown>(fn: T): T {
  const cache = new Map<string, unknown>();

  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);
      if (cache.has(key)) {
        console.log(`[memo] cache HIT for key: ${key}`);
        return cache.get(key);
      }
      const result = Reflect.apply(target, thisArg, args);
      cache.set(key, result);
      console.log(`[memo] cache MISS, stored key: ${key}`);
      return result;
    },
  }) as T;
}

function expensiveFibonacci(n: number): number {
  if (n <= 1) return n;
  return expensiveFibonacci(n - 1) + expensiveFibonacci(n - 2);
}

const memoFib = memoize(expensiveFibonacci);
memoFib(10); // cache MISS → computed
memoFib(10); // cache HIT  → instant
memoFib(7);  // cache MISS → computed
```

The `apply` trap intercepts `memoFib(10)` as a function call, letting us wrap any function without modifying it.

---

### Read-Only Proxy

```typescript
function deepReadonly<T extends object>(target: T): Readonly<T> {
  return new Proxy(target, {
    get(obj, prop, receiver) {
      const value = Reflect.get(obj, prop, receiver);
      // Recursively wrap nested objects
      if (value !== null && typeof value === "object") {
        return deepReadonly(value as object);
      }
      return value;
    },
    set(_obj, prop) {
      throw new TypeError(
        `Cannot assign to read-only property '${String(prop)}'`
      );
    },
    deleteProperty(_obj, prop) {
      throw new TypeError(
        `Cannot delete read-only property '${String(prop)}'`
      );
    },
  });
}

const config = deepReadonly({ db: { host: "localhost", port: 5432 } });
console.log(config.db.host); // "localhost"
config.db.host = "prod-db";  // TypeError: Cannot assign to read-only property 'host'
```

---

## Vue 3 Reactivity: How `reactive()` Uses Proxy

Vue 3's reactivity system is built entirely on `Proxy`. Understanding it is essential for senior-level Vue/React architecture interviews.

### Simplified `reactive()` Implementation

```javascript
// Vue 3 simplified reactive core (conceptual — real source in @vue/reactivity)

const targetMap = new WeakMap(); // target → depsMap
let activeEffect = null;         // currently-running effect

function track(target, key) {
  if (!activeEffect) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) targetMap.set(target, (depsMap = new Map()));
  let dep = depsMap.get(key);
  if (!dep) depsMap.set(key, (dep = new Set()));
  dep.add(activeEffect);         // subscribe current effect to this key
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const dep = depsMap.get(key);
  dep?.forEach((effect) => effect()); // re-run all subscribers
}

function reactive(target) {
  return new Proxy(target, {
    get(obj, prop, receiver) {
      track(obj, prop);                      // subscribe reader
      const value = Reflect.get(obj, prop, receiver);
      // Lazily wrap nested objects (no deep clone upfront)
      if (value !== null && typeof value === "object") {
        return reactive(value);
      }
      return value;
    },
    set(obj, prop, value, receiver) {
      const result = Reflect.set(obj, prop, value, receiver);
      trigger(obj, prop);                    // notify subscribers
      return result;
    },
  });
}

function watchEffect(fn) {
  activeEffect = fn;
  fn();              // run once to collect dependencies via track()
  activeEffect = null;
}

// Demo
const state = reactive({ count: 0, name: "Vue" });

watchEffect(() => {
  console.log(`count is: ${state.count}`); // auto-subscribes to count
});

state.count++;  // triggers re-run: "count is: 1"
state.count++;  // "count is: 2"
state.name = "Vue 3"; // no re-run — effect didn't read name
```

**Key insight:** Vue 3 tracks dependencies lazily and precisely. Only the exact properties read during an effect are tracked. Changing an unread property does nothing. This is the `get` trap doing dependency collection and the `set` trap doing notification.

### Why Vue 2 Used `Object.defineProperty` (and Why Proxy is Better)

| Limitation | Vue 2 (`defineProperty`) | Vue 3 (`Proxy`) |
|---|---|---|
| Adding new properties | Not reactive — must use `Vue.set()` | Fully reactive |
| Deleting properties | Not reactive — must use `Vue.delete()` | Fully reactive (`deleteProperty` trap) |
| Array index mutations | Partially broken (`arr[0] = x` not tracked) | Fully reactive |
| `Array.length` | Not trackable | Trackable via `set` trap |
| Performance | Deep walk at init time | Lazy — only proxied on access |
| `has` / `in` operator | Not interceptable | Interceptable |

---

## Proxy for Lazy Initialization (Deep Dive)

The virtual proxy pattern is a first-class use case for the JS `Proxy` object. Here is a production-grade version with deferred module loading:

```typescript
// Lazy singleton factory using Proxy
function createLazyProxy<T extends object>(factory: () => T): T {
  let instance: T | null = null;

  function getInstance(): T {
    if (!instance) {
      console.log("[LazyProxy] Initializing instance...");
      instance = factory();
    }
    return instance;
  }

  // We need a base object for Proxy — use an empty object but intercept all ops
  return new Proxy({} as T, {
    get(_target, prop, receiver) {
      return Reflect.get(getInstance(), prop, receiver);
    },
    set(_target, prop, value, receiver) {
      return Reflect.set(getInstance(), prop, value, receiver);
    },
    has(_target, prop) {
      return Reflect.has(getInstance(), prop);
    },
    ownKeys() {
      return Reflect.ownKeys(getInstance());
    },
    getOwnPropertyDescriptor(_target, prop) {
      return Reflect.getOwnPropertyDescriptor(getInstance(), prop);
    },
  });
}

// Usage: DB connection is not established until first property access
const db = createLazyProxy(() => {
  // Expensive: opens real TCP connection, parses schema, etc.
  console.log("[DB] Connecting to PostgreSQL...");
  return {
    query: (sql: string) => `Result of: ${sql}`,
    close: () => console.log("[DB] Connection closed"),
  };
});

console.log("App started — no DB connection yet");
// ... app setup ...
const result = db.query("SELECT 1"); // triggers connection HERE
console.log(result);
db.query("SELECT version()"); // reuses existing connection
```

---

## Security: Prototype Pollution and Proxy Defense

### The Attack

Prototype pollution occurs when attacker-controlled input sets properties on `Object.prototype`, affecting all objects:

```javascript
// Attacker controls `userInput` from a JSON body, query string, etc.
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === "object") {
      target[key] = target[key] || {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

const payload = JSON.parse('{"__proto__":{"isAdmin":true}}');
merge({}, payload);

// Every object in the process is now poisoned:
const innocent = {};
console.log(innocent.isAdmin); // true — pollution via Object.prototype
```

### Proxy as a Defence

```javascript
function createSafeObject(initial = {}) {
  const BLOCKED_KEYS = new Set(["__proto__", "constructor", "prototype"]);

  return new Proxy(Object.create(null), { // null prototype — no inherited methods
    set(target, prop, value, receiver) {
      if (BLOCKED_KEYS.has(String(prop))) {
        console.warn(`[Security] Blocked write to '${String(prop)}'`);
        return false; // signals failure — throws in strict mode
      }
      return Reflect.set(target, prop, value, receiver);
    },
    get(target, prop, receiver) {
      if (BLOCKED_KEYS.has(String(prop))) {
        console.warn(`[Security] Blocked read of '${String(prop)}'`);
        return undefined;
      }
      return Reflect.get(target, prop, receiver);
    },
    has(target, prop) {
      if (BLOCKED_KEYS.has(String(prop))) return false;
      return Reflect.has(target, prop);
    },
  });
}

const safe = createSafeObject();
safe.__proto__ = { isAdmin: true }; // blocked — returns false
safe.name = "Alice";                 // OK
console.log(safe.name);             // "Alice"
console.log({}.isAdmin);            // still undefined — pollution prevented
```

**Two-layer defence:**
1. `Object.create(null)` — the target has no prototype, so pollution via `__proto__` has nowhere to attach.
2. `set` trap — explicitly blocks the dangerous keys before they reach `Reflect.set`.

---

## Proxy vs Decorator vs Facade

These three patterns are frequently confused in interviews. The distinction is precise:

### Proxy vs Decorator

| | Proxy | Decorator |
|---|---|---|
| **Primary intent** | Control *access* to an object | Add *behaviour* to an object |
| **Interface** | Same interface as real subject | Same interface as component |
| **Knows about subject?** | Yes — usually manages lifecycle | Wraps externally, no control |
| **Subject creation** | Proxy may create/delay subject | Subject already exists |
| **Examples** | Access control, lazy load, remote stub | Logging wrapper, retry wrapper |

```javascript
// Decorator ADDS behaviour — real object exists independently
const service = new UserService();
const loggingDecorator = new LoggingUserService(service); // adds logging

// Proxy CONTROLS ACCESS — proxy may own the real object lifecycle
const proxy = new UserServiceProxy(); // may create real object lazily, or never
```

### Proxy vs Facade

| | Proxy | Facade |
|---|---|---|
| **Interface** | **Same** as real subject | **Different** (simplified) |
| **Number of subsystems** | One real subject | Many subsystems behind one interface |
| **Purpose** | Mediate access | Simplify complex subsystem |
| **Transparency** | Client should not know it's a proxy | Client knows it's a higher-level API |

```javascript
// Facade: one clean method hiding 4 subsystems
class OrderFacade {
  placeOrder(cart) {
    this.inventory.reserve(cart);
    this.payment.charge(cart.total);
    this.shipping.schedule(cart.address);
    this.notifications.sendConfirmation(cart.userId);
  }
}

// Proxy: same interface, adds access control
class OrderServiceProxy {
  placeOrder(cart) {              // same signature as OrderService
    if (!this.isAuthenticated()) throw new Error("Unauthorized");
    return this.real.placeOrder(cart);
  }
}
```

---

## Real-World Usage in the Frontend Ecosystem

### Valtio (proxy-based state management for React)

Valtio uses `Proxy` to make plain JS objects observable, similar to Vue 3 but designed for React:

```javascript
import { proxy, useSnapshot } from "valtio";

// `proxy()` wraps the object in a Proxy that tracks mutations
const state = proxy({ count: 0, user: { name: "Alice" } });

// Mutations are direct — no dispatch, no immer produce
state.count++;
state.user.name = "Bob";

// In React:
function Counter() {
  const snap = useSnapshot(state); // snapshot = readonly view, triggers re-render on change
  return <div onClick={() => state.count++}>{snap.count}</div>;
}
```

Valtio's `proxy()` sets up `set` traps that schedule React re-renders. `useSnapshot` creates a read-only snapshot via a `get` trap that records which properties the component read — similar to Vue's `track`.

---

### MobX Observable

MobX 6+ dropped `Object.defineProperty` decorators in favour of `Proxy`:

```javascript
import { makeAutoObservable } from "mobx";

class TodoStore {
  todos = [];
  filter = "all";

  constructor() {
    makeAutoObservable(this); // internally uses Proxy in MobX 6
  }

  addTodo(text) {
    this.todos.push({ text, done: false }); // Proxy set trap fires
  }

  get visibleTodos() { // computed — Proxy get trap tracks this
    return this.filter === "all"
      ? this.todos
      : this.todos.filter((t) => t.done === (this.filter === "done"));
  }
}
```

---

### Immer `produce`

Immer uses a `Proxy`-based "draft" to let you write mutating code that produces an immutable result:

```javascript
import produce from "immer";

const nextState = produce(baseState, (draft) => {
  // draft is a Proxy — all mutations are recorded, not applied
  draft.todos[0].done = true;
  draft.todos.push({ text: "New todo", done: false });
  // When the recipe returns, Immer applies mutations to a new immutable object
});
// baseState is unchanged; nextState is a new object with structural sharing
```

Immer's draft Proxy:
- `get` trap: returns child drafts for nested objects (also Proxies)
- `set` trap: records mutations in a change log
- On recipe completion: replays mutations against a cloned base using structural sharing

---

### Vue 3 `reactive()` vs `ref()`

```javascript
import { reactive, ref, watch } from "vue";

// reactive() — Proxy over the whole object
const state = reactive({ count: 0 });

// ref() — wraps primitives; internally uses reactive({ value: x })
const count = ref(0);

// Both use Proxy under the hood — same set/get trap mechanism
watch(() => state.count, (newVal) => console.log("count:", newVal));
state.count = 5; // fires watcher
```

---

## Performance Considerations

### Proxy Overhead is Real

Every property access through a proxy is slower than a direct object access because the JavaScript engine:
1. Must look up the handler trap
2. Execute your trap function (a function call)
3. Forward to `Reflect` (another indirection)

```javascript
// Benchmark: raw object vs proxy
const raw = { x: 1 };
const proxied = new Proxy(raw, { get: Reflect.get });

// On V8 (Chrome/Node), a proxied get is ~2–5x slower than a raw get
// On a tight loop with 10M iterations this becomes measurable
```

### When It Matters

| Scenario | Concern Level | Reason |
|---|---|---|
| Reactive state (Vue/MobX/Valtio) | Low | State updates are rare compared to rendering |
| Hot render path (inside `for` loop per frame) | High | Thousands of property reads per second |
| Large arrays with `Array.prototype` methods | Medium | `forEach`/`map` triggers a `get` per element |
| One-time config / lazy init | Negligible | Called once |
| Deep object trees in tight loops | High | Every level adds a trap layer |

### Mitigation Strategies

```javascript
// BAD: proxy accessed inside tight loop
function renderItems(proxyState) {
  for (let i = 0; i < 100000; i++) {
    process(proxyState.items[i]); // proxy trap per iteration
  }
}

// GOOD: snapshot once, loop over plain object
function renderItems(proxyState) {
  const items = proxyState.items; // one trap, then plain array
  for (let i = 0; i < 100000; i++) {
    process(items[i]); // no trap — items is now a plain array reference
  }
}

// Vue 3's solution: toRaw() strips the proxy
import { toRaw } from "vue";
const plainState = toRaw(reactiveState); // raw access — no reactivity overhead
```

### The `has` Trap and `in` Operator

A subtle perf trap: using `in` inside a loop triggers the `has` trap per iteration:

```javascript
const handler = {
  has(target, key) {
    console.log(`has: ${key}`); // fires on every 'in' check
    return Reflect.has(target, key);
  }
};
const p = new Proxy({a: 1}, handler);
for (const k of ["a","b","c","a","b"]) {
  if (k in p) { /* ... */ } // 5 trap calls
}
```

---

## Interview Q&A

**Q: What is the intent of the Proxy pattern, and how does it differ from Decorator?**

A: Proxy controls *access* to an object through a surrogate that exposes the **same interface** as the real subject. Decorator *adds behaviour* by wrapping an already-existing object — it can also stack (chain multiple decorators). The key distinction: a proxy often manages the lifecycle of the real subject (lazy creation, remote stub, access gating), while a decorator enhances a subject that already exists independently.

---

**Q: What are all the traps available on a JavaScript `Proxy` and when would you use `apply` vs `get`?**

A: There are 13 traps. The most used are `get`, `set`, `has`, `deleteProperty`, and `apply`. Use `get`/`set` to intercept property reads/writes (validation, logging, reactivity). Use `apply` when the target itself is a **function** — it intercepts `fn()`, `fn.call()`, and `fn.apply()`. This is how memoization proxies and function call logging work. Use `construct` to intercept `new fn()`.

---

**Q: Why must you use `Reflect` in Proxy traps instead of operating on the target directly?**

A: Two reasons. First, `Reflect` correctly forwards the `receiver` argument, which matters when the target has getters/setters that reference `this` — without `receiver`, `this` points to the wrong object. Second, `Reflect.set` returns a boolean indicating success or failure, which is the required return type of the `set` trap — returning `false` in strict mode throws a `TypeError`, which is the proper way to signal a failed assignment.

---

**Q: How does Vue 3's reactivity system use Proxy? Why was it an improvement over Vue 2's `Object.defineProperty`?**

A: Vue 3's `reactive()` wraps objects in a `Proxy`. The `get` trap calls `track()` to subscribe the currently-running effect to the accessed property; the `set` trap calls `trigger()` to re-run all subscribers. This is lazily applied (nested objects are only wrapped when accessed) and covers operations `defineProperty` cannot: adding/deleting new properties, array index mutations (`arr[0] = x`), and the `in` operator. Vue 2 required `Vue.set()` and `Vue.delete()` workarounds; Vue 3 does not.

---

**Q: How does Immer use Proxy under the hood?**

A: When you call `produce(base, recipe)`, Immer creates a "draft" — a Proxy wrapping the base object. The `set` trap records mutations in a change log without applying them to the base. The `get` trap returns child drafts for nested objects (also Proxies). When the recipe function returns, Immer replays the recorded mutations against a cloned base using structural sharing — only the changed branches are new objects. The result is an immutable state update written with mutable syntax.

---

**Q: What is prototype pollution, and how can a Proxy defend against it?**

A: Prototype pollution is an attack where an adversary sets properties on `Object.prototype` via crafted input (e.g., `{"__proto__": {"isAdmin": true}}`), causing all plain objects to inherit the malicious property. A Proxy defends by: (1) using `Object.create(null)` for the target — no prototype to pollute — and (2) adding a `set` trap that blocks writes to `__proto__`, `constructor`, and `prototype` keys explicitly. This stops deep-merge/assign functions from ever reaching the prototype chain.

---

**Q: Explain the performance cost of Proxy. When should you avoid it in hot paths?**

A: Every property access through a Proxy involves a function call (the trap), which prevents V8's inline caching (ICs) from optimizing the access as aggressively as a plain object read. Benchmarks show 2–5x slower property reads in tight loops. In practice this only matters in computationally hot paths — tight loops over large datasets, per-frame rendering calculations, or real-time signal processing. Mitigation: take a snapshot of the proxied value before entering the loop (`const items = state.items`) so the proxy trap fires once, then the loop iterates over a plain array. Vue 3 exposes `toRaw()` for exactly this purpose.

---

**Q: Can you proxy a built-in like `Map` or `Array`? Are there gotchas?**

A: Yes, but with gotchas. Built-in `Map`/`Set`/`Date` methods perform internal slot checks — they require `this` to be the actual Map/Set/Date instance, not a Proxy. A naive `get` trap that returns `Reflect.get(target, prop, receiver)` will fail because `Map.prototype.get` checks that its `this` is a real Map. The fix is to bind the method to the target:

```javascript
const mapProxy = new Proxy(new Map(), {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, target); // bind to target, not receiver
    if (typeof value === "function") {
      return value.bind(target); // ensure 'this' is the real Map
    }
    return value;
  },
});
mapProxy.set("key", "val"); // works
mapProxy.get("key");        // works — would fail without .bind(target)
```

---

**Q: What is the difference between a remote proxy and a local proxy? Give a concrete frontend example.**

A: A local proxy mediates access to an object in the same process/thread (lazy init, access control, logging). A remote proxy represents an object in a different address space — another server, web worker, or iframe — and handles all serialization/deserialization transparently. The client calls methods as if the object is local. In the frontend: Comlink turns a Web Worker into a remote proxy — you call `worker.doWork(data)` as a normal async function and Comlink posts the message, receives the response, and resolves the promise. Angular's `HttpClient` is conceptually a remote proxy over the network layer.

---

**Q: Valtio, MobX, and Vue 3 all use Proxy for state. How do they differ architecturally?**

A: All three use `Proxy` to track reads (for subscriptions) and intercept writes (for notifications), but differ in design philosophy:
- **Vue 3 `reactive()`**: tracks at the component-render-function level using an effect stack; deeply reactive by default; `ref()` wraps primitives.
- **MobX**: tracks at the observable/computed/reaction level; supports class-based models; `makeAutoObservable` auto-detects actions, observables, and computeds.
- **Valtio**: tracks at the React-hook level (`useSnapshot`); mutations to the proxy object schedule a re-render; snapshot is immutable (Immer-like); no explicit actions needed.

The shared Proxy mechanism is the same; the difference is what happens *when a trap fires* and *how the UI re-render is scheduled*.
