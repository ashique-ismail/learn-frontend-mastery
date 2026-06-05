# Symbols and Well-Known Symbols

## The Idea

**In plain English:** A Symbol is a special kind of value in JavaScript that is guaranteed to be completely unique — no two Symbols are ever equal, even if you give them the same label. They are used as secret, collision-proof keys for labelling things inside objects without accidentally clashing with other code's labels.

**Real-world analogy:** Imagine a busy hotel where every guest gets a physical room key card programmed just for them. Even if two guests both ask for "Room 101", their key cards are physically different chips — only the card made for your specific stay will open your door. A new card is cut each time, so no duplicates exist.

- The key card chip = a Symbol value (unique every time one is created)
- The room label "Room 101" written on the envelope = the Symbol's description (just a human-readable note, not the identity)
- The hotel's master key-card machine = `Symbol()` (the function that creates a new unique value)

---

## Overview

`Symbol` is a primitive type introduced in ES6 that creates guaranteed-unique values. Unlike strings or numbers, every Symbol value is distinct even if two symbols share the same description. This uniqueness makes symbols ideal for property keys that must not collide with string keys or other libraries' symbols. Beyond user-defined symbols, JavaScript's specification itself uses a set of **well-known symbols** as hooks into the language's internal algorithms — allowing user-defined classes to customize how they behave with operators, iteration, coercion, and built-in functions. Mastering symbols is essential for building frameworks, polyfills, and libraries that integrate seamlessly with native JS semantics.

## Creating Symbols

### Symbol()

Every call to `Symbol()` returns a unique, immutable primitive:

```javascript
const s1 = Symbol();
const s2 = Symbol();

s1 === s2; // false — always unique
typeof s1;  // "symbol"

// Optional description (for debugging only — does not affect identity)
const s3 = Symbol("userId");
const s4 = Symbol("userId");
s3 === s4; // false — still unique despite same description
s3.toString(); // "Symbol(userId)"
s3.description; // "userId" (ES2019)
```

### Symbols Are Not Constructable

Symbols are primitives — using `new Symbol()` throws:

```javascript
new Symbol(); // TypeError: Symbol is not a constructor
```

### Symbol as Object Property Key

Symbols can be used as property keys alongside strings:

```javascript
const ID = Symbol("id");
const user = {
  name: "Alice",
  [ID]: 42  // Computed property key using a symbol
};

user[ID];     // 42
user.name;    // "Alice"
user["id"];   // undefined — symbol key is distinct from string key
```

## Symbol Invisibility

Symbol-keyed properties are excluded from most enumeration mechanisms:

```javascript
const SECRET = Symbol("secret");
const obj = {
  visible: "hello",
  [SECRET]: "hidden"
};

Object.keys(obj);            // ["visible"]
Object.values(obj);          // ["hello"]
Object.entries(obj);         // [["visible", "hello"]]
for (const key in obj) {}    // Only "visible"
JSON.stringify(obj);         // '{"visible":"hello"}'
Object.assign({}, obj);      // { visible: "hello" } — symbol keys NOT copied!

// But not truly hidden:
Object.getOwnPropertySymbols(obj);        // [Symbol(secret)]
Reflect.ownKeys(obj);                     // ["visible", Symbol(secret)]
obj[SECRET];                              // "hidden"
```

This makes symbols ideal for metadata properties on objects shared with third-party code — the metadata is invisible to normal enumeration but accessible to code that holds a reference to the symbol.

## Symbol.for and Symbol.keyFor

### Global Symbol Registry

`Symbol.for(key)` creates or retrieves a symbol from a **global cross-realm registry**, keyed by a string:

```javascript
const s1 = Symbol.for("app.userId");
const s2 = Symbol.for("app.userId");

s1 === s2; // true — same symbol from the registry

// Compare with Symbol():
const s3 = Symbol("app.userId");
s3 === s1; // false — Symbol() always creates a new unique symbol
```

### Symbol.keyFor

Retrieves the key used to register a global symbol:

```javascript
const s = Symbol.for("my.key");
Symbol.keyFor(s);      // "my.key"

const local = Symbol("local");
Symbol.keyFor(local);  // undefined — not in the registry
```

### Use Case: Cross-Module or Cross-Realm Keys

`Symbol.for` is useful when multiple modules or browser frames need to share the same symbol without passing it as an import:

```javascript
// module-a.js
const PLUGIN_KEY = Symbol.for("mylib.plugin");
export { PLUGIN_KEY };

// module-b.js — even if module-a is not imported, same key can be obtained:
const PLUGIN_KEY = Symbol.for("mylib.plugin");
// Works as long as both use the same string key
```

## Well-Known Symbols

Well-known symbols are properties on the `Symbol` constructor. They act as extension hooks into JavaScript's internal algorithms. By defining these on a class or object, you override how the built-in language mechanisms treat that object.

### Symbol.iterator

Defines the default iterator for an object. Used by `for...of`, spread, destructuring, `Array.from`, and `Promise.all`.

```javascript
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const { end, step } = this;

    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      },
      [Symbol.iterator]() { return this; } // Iterable iterator
    };
  }
}

const r = new Range(1, 10, 2);
[...r];                 // [1, 3, 5, 7, 9]
for (const n of r) {}  // Works
const [first] = r;     // first = 1

// Check if something is iterable
function isIterable(obj) {
  return obj != null && typeof obj[Symbol.iterator] === "function";
}
```

### Symbol.asyncIterator

Defines the async iterator for `for await...of`:

```javascript
class AsyncRange {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }

  [Symbol.asyncIterator]() {
    let current = this.start;
    const { end } = this;

    return {
      async next() {
        await new Promise(r => setTimeout(r, 10)); // Simulate async work
        if (current <= end) {
          return { value: current++, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

async function run() {
  for await (const n of new AsyncRange(1, 5)) {
    console.log(n); // 1, 2, 3, 4, 5 with delays
  }
}
```

### Symbol.toPrimitive

Controls type coercion. Called when an object is coerced to a primitive, with a `hint` parameter of `"number"`, `"string"`, or `"default"`:

```javascript
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }

  [Symbol.toPrimitive](hint) {
    switch (hint) {
      case "number":  return this.amount;
      case "string":  return `${this.amount} ${this.currency}`;
      case "default": return this.amount;
    }
  }
}

const price = new Money(42.5, "USD");

+price;           // 42.5 (number hint)
`${price}`;       // "42.5 USD" (string hint)
price + 10;       // 52.5 (default hint)
price > 40;       // true (number comparison)
```

Without `Symbol.toPrimitive`, JavaScript falls back to calling `valueOf()` and `toString()`.

### Symbol.toStringTag

Customizes the `[object Tag]` output of `Object.prototype.toString.call(obj)`:

```javascript
class Collection {
  get [Symbol.toStringTag]() {
    return "Collection";
  }
}

const c = new Collection();
Object.prototype.toString.call(c);  // "[object Collection]"
c.toString();                       // "[object Collection]"

// Used for reliable type checking (unlike typeof or instanceof)
function getType(value) {
  return Object.prototype.toString.call(value).slice(8, -1);
}
getType([]);         // "Array"
getType(new Map());  // "Map"
getType(c);          // "Collection"
```

### Symbol.hasInstance

Controls the behavior of the `instanceof` operator:

```javascript
class Even {
  static [Symbol.hasInstance](num) {
    return Number.isInteger(num) && num % 2 === 0;
  }
}

2 instanceof Even;  // true
3 instanceof Even;  // false
4 instanceof Even;  // true

// Non-integer
1.5 instanceof Even; // false
```

### Symbol.species

Determines the constructor used by derived methods that create new instances. Used by `Array.prototype.map`, `filter`, `slice`, `RegExp.prototype.exec`, etc.

```javascript
class MyArray extends Array {
  static get [Symbol.species]() {
    return Array; // Derived methods return plain Array, not MyArray
  }
}

const a = new MyArray(1, 2, 3);
a instanceof MyArray;        // true
a.map(x => x).constructor;  // Array (not MyArray, due to species)
a.map(x => x) instanceof MyArray; // false
```

Without `Symbol.species`, `a.map(x => x)` would return a `MyArray`. With it, you get a plain `Array`.

### Symbol.isConcatSpreadable

Controls whether an object is spread by `Array.prototype.concat`:

```javascript
const arr = [1, 2, 3];
const arrayLike = { 0: 4, 1: 5, length: 2, [Symbol.isConcatSpreadable]: true };

[].concat(arr, arrayLike); // [1, 2, 3, 4, 5]

// Prevent array spreading
const frozen = [10, 20];
frozen[Symbol.isConcatSpreadable] = false;
[].concat(frozen); // [ [10, 20] ] — not spread, included as element
```

### Symbol.match, Symbol.replace, Symbol.search, Symbol.split

Customize how `String.prototype.match`, `replace`, `search`, and `split` interact with custom "regex-like" objects:

```javascript
class CaseInsensitiveMatcher {
  constructor(pattern) {
    this.pattern = pattern.toLowerCase();
  }

  [Symbol.match](string) {
    const lower = string.toLowerCase();
    const idx = lower.indexOf(this.pattern);
    if (idx === -1) return null;
    return [string.slice(idx, idx + this.pattern.length)];
  }

  [Symbol.search](string) {
    return string.toLowerCase().indexOf(this.pattern);
  }

  [Symbol.replace](string, replacement) {
    const re = new RegExp(this.pattern, "gi");
    return string.replace(re, replacement);
  }
}

const matcher = new CaseInsensitiveMatcher("hello");
"Say HELLO world".match(matcher);          // ["HELLO"]
"Say HELLO world".search(matcher);         // 4
"Say HELLO world".replace(matcher, "hi");  // "Say hi world"
```

### Symbol.unscopables

Prevents certain properties from being exposed inside `with` blocks. Used internally by `Array.prototype` to keep new methods from breaking legacy `with`-based code:

```javascript
Array.prototype[Symbol.unscopables];
// { flat: true, flatMap: true, at: true, copyWithin: true, ... }
```

Rarely needed in modern code, but important for library authors who add methods to built-in prototypes.

## Comparison Table

| Symbol | Affected By | Hook Purpose |
|---|---|---|
| `Symbol.iterator` | `for...of`, spread, destructuring | Custom iteration |
| `Symbol.asyncIterator` | `for await...of` | Custom async iteration |
| `Symbol.toPrimitive` | Coercion operators | Custom type conversion |
| `Symbol.toStringTag` | `Object.prototype.toString` | Custom type tag |
| `Symbol.hasInstance` | `instanceof` | Custom instance check |
| `Symbol.species` | `map`, `filter`, `slice`, etc. | Derived constructor override |
| `Symbol.isConcatSpreadable` | `Array.prototype.concat` | Spreading behavior |
| `Symbol.match` | `String.prototype.match` | Custom match |
| `Symbol.replace` | `String.prototype.replace` | Custom replace |
| `Symbol.search` | `String.prototype.search` | Custom search |
| `Symbol.split` | `String.prototype.split` | Custom split |
| `Symbol.unscopables` | `with` statement | Property visibility |

## Best Practices

### 1. Use Symbols for Non-Public Properties

```javascript
// Library code
export const INTERNAL_STATE = Symbol("internalState");

// Users of the library cannot accidentally override this
// because they don't have a reference to INTERNAL_STATE
```

### 2. Use Symbol.for for Shared Protocols

When building extensible libraries where multiple modules must coordinate on the same extension point, use `Symbol.for` with a namespaced key:

```javascript
const VALIDATE = Symbol.for("mylib.validate");
const SERIALIZE = Symbol.for("mylib.serialize");
```

### 3. Document Well-Known Symbol Overrides

Overriding well-known symbols changes fundamental behavior. Always document that a class implements `Symbol.toPrimitive`, `Symbol.iterator`, etc., and what behavior is expected.

### 4. Prefer Symbol() Over String for Private Metadata

```javascript
// Bad — string key collides with user properties
obj._internalId = 1;

// Good — guaranteed unique, invisible to enumeration
const INTERNAL_ID = Symbol("internalId");
obj[INTERNAL_ID] = 1;
```

## Interview Questions

### Q1: What is the difference between Symbol() and Symbol.for()?

**Answer:** `Symbol()` always creates a new, unique symbol. Even two calls with the same description produce different symbols. `Symbol.for(key)` looks up the global symbol registry — if a symbol with that string key exists, it returns it; otherwise it creates and registers a new one. `Symbol.for` is suitable for cross-module shared keys; `Symbol()` is suitable for truly local, unshared keys.

### Q2: Why are symbol-keyed properties not included in JSON.stringify?

**Answer:** The JSON specification only serializes string-keyed enumerable own properties. Symbol keys are not strings and are excluded by the `JSON.stringify` algorithm. This is by design — symbols are metadata keys meant for internal use, not serializable data.

### Q3: How does Symbol.iterator make an object iterable?

**Answer:** The `for...of` loop, spread operator, destructuring, and `Array.from` all check whether an object has a method at `[Symbol.iterator]`. If it does, they call it to obtain an iterator — an object with a `next()` method that returns `{ value, done }`. By implementing `[Symbol.iterator]` on a class or object, you make it compatible with all iteration consumers without modifying any prototype.

### Q4: What does Symbol.toPrimitive replace, and what is the hint parameter?

**Answer:** `Symbol.toPrimitive` replaces the old `valueOf` / `toString` coercion algorithm. When JS needs to coerce an object to a primitive, it calls `[Symbol.toPrimitive](hint)` if it exists. The hint is `"number"` when a numeric context is expected (`+obj`, comparison), `"string"` when a string context is expected (template literals), or `"default"` for ambiguous contexts (binary `+`, equality). This allows a class to return different primitive types depending on context.

### Q5: Can you use a Symbol as a WeakMap key?

**Answer:** As of ES2023 (the WeakMap symbol keys proposal), registered symbols (`Symbol.for(...)`) cannot be used as WeakMap keys, but unregistered symbols created with `Symbol()` can. This was added to support use cases like records and tuples. Before ES2023, only objects could be WeakMap keys.

## Common Pitfalls

### 1. Symbols Are Not Auto-Copied by Object.assign

```javascript
const SYM = Symbol("x");
const source = { a: 1, [SYM]: 2 };

const copy = Object.assign({}, source);
copy.a;     // 1
copy[SYM];  // undefined — symbol keys are NOT copied by Object.assign!

// To copy symbol keys, use:
const fullCopy = Object.defineProperties(
  {},
  Object.getOwnPropertyDescriptors(source)
);
fullCopy[SYM]; // 2
```

### 2. Symbol Description Is Not Its Identity

```javascript
const s1 = Symbol("name");
const s2 = Symbol("name");
s1 === s2; // false — description is for debugging only, not identity
```

### 3. Overusing Symbol.for Creates Global Namespace Collisions

`Symbol.for` is global (cross-realm). Keys like `"id"` or `"type"` can collide between libraries. Always namespace your keys: `Symbol.for("mylib.1.0.id")`.

### 4. Symbol.species Was Deprecated in Favor of Direct Override

The `Symbol.species` mechanism has been deprecated in some contexts (e.g., `ArrayBuffer`, typed arrays) in recent spec work. Prefer overriding the specific method directly when possible.

## Resources

### Official Documentation
- [MDN: Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)
- [MDN: Well-known symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#well-known_symbols)
- [ECMAScript: Symbol Objects](https://tc39.es/ecma262/#sec-symbol-objects)
- [TC39: WeakMap symbol keys proposal](https://github.com/tc39/proposal-symbols-as-weakmap-keys)

### Articles and Guides
- [Metaprogramming with Symbols (2ality)](https://2ality.com/2014/12/es6-symbols.html)
- [Well-Known Symbols (Exploring JS)](https://exploringjs.com/es6/ch_symbols.html)
- [Symbol.iterator and the Iteration Protocol (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)
