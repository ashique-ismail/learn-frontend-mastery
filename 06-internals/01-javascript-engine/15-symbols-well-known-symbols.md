# Symbols and Well-Known Symbols

## The Idea

**In plain English:** A symbol is a guaranteed-unique label that JavaScript can create for you — no two symbols are ever equal, even if they look the same. Well-known symbols are special built-in labels that JavaScript itself looks for to decide how your objects behave in things like loops or type conversions.

**Real-world analogy:** Imagine a coat-check room at a concert venue. Every coat is given a numbered token. Even if two people both receive a token labeled "7", they are physically different tokens — only the exact token you were handed will retrieve your coat.

- The token = a Symbol (a unique identifier, not just a label)
- The coat-check label printed on the token = the Symbol's description (just for humans to read, does not determine uniqueness)
- The well-known coat-check slot labeled "VIP" that all venues reserve = a well-known Symbol (a pre-agreed slot JavaScript uses internally, like `Symbol.iterator`)

---

## Table of Contents
- [Overview](#overview)
- [Symbol Basics](#symbol-basics)
- [Symbol Purpose](#symbol-purpose)
- [Global Symbol Registry](#global-symbol-registry)
- [Well-Known Symbols](#well-known-symbols)
- [Symbol Use Cases](#symbol-use-cases)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Symbols are a primitive data type introduced in ES6 (ES2015). They are unique, immutable values primarily used as object property keys to avoid name collisions. Well-known symbols are built-in symbols that allow customization of language behavior.

## Symbol Basics

### Creating Symbols

```javascript
// Create unique symbol
const sym1 = Symbol();
const sym2 = Symbol();

console.log(sym1 === sym2); // false (each Symbol() call creates unique symbol)

// Symbols with descriptions (for debugging)
const sym3 = Symbol('mySymbol');
const sym4 = Symbol('mySymbol');

console.log(sym3 === sym4); // false (description doesn't affect uniqueness)
console.log(sym3.toString()); // 'Symbol(mySymbol)'
console.log(sym3.description); // 'mySymbol'
```

### Symbols as Object Keys

```javascript
const id = Symbol('id');
const user = {
  name: 'Alice',
  [id]: 123 // Symbol as computed property
};

console.log(user[id]); // 123
console.log(user.id);  // undefined (not a string property)

// Symbols don't appear in normal enumeration
console.log(Object.keys(user)); // ['name']
for (let key in user) {
  console.log(key); // Only 'name', not symbol
}

// Access symbols with specific methods
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
console.log(Reflect.ownKeys(user)); // ['name', Symbol(id)]
```

### Symbol Properties Are Hidden

```javascript
const secret = Symbol('secret');
const obj = {
  visible: 'public',
  [secret]: 'hidden'
};

// Standard methods ignore symbols
JSON.stringify(obj); // '{"visible":"public"}'
Object.assign({}, obj); // { visible: 'public' } (symbol not copied)

// Deep clone needs special handling
const clone = {
  ...obj,
  ...Object.getOwnPropertySymbols(obj).reduce((acc, sym) => {
    acc[sym] = obj[sym];
    return acc;
  }, {})
};

console.log(clone[secret]); // 'hidden'
```

## Symbol Purpose

### Avoiding Property Name Collisions

```javascript
// Problem: String keys can collide
const library1 = {
  id: 'lib1-id'
};

const library2 = {
  id: 'lib2-id'
};

const merged = { ...library1, ...library2 };
console.log(merged.id); // 'lib2-id' (collision, lib1 overwritten)

// Solution: Use symbols
const id1 = Symbol('id');
const id2 = Symbol('id');

const lib1 = {
  [id1]: 'lib1-id'
};

const lib2 = {
  [id2]: 'lib2-id'
};

const mergedSafe = { ...lib1, ...lib2 };
console.log(mergedSafe[id1]); // 'lib1-id'
console.log(mergedSafe[id2]); // 'lib2-id' (no collision)
```

### Private-Like Properties

```javascript
// Symbols provide "privacy" (not true privacy, but hidden from casual access)
const _internal = Symbol('internal');

class Component {
  constructor() {
    this[_internal] = {
      state: {},
      cache: new Map()
    };
  }
  
  setState(newState) {
    this[_internal].state = { ...this[_internal].state, ...newState };
  }
  
  getState() {
    return { ...this[_internal].state };
  }
}

const comp = new Component();
comp.setState({ count: 0 });
console.log(comp.getState()); // { count: 0 }
console.log(comp._internal); // undefined (not accessible as string)
console.log(Object.keys(comp)); // [] (symbol hidden)

// Still accessible if you have the symbol
console.log(comp[_internal]); // { state: { count: 0 }, cache: Map(0) {} }
```

### Meta-Programming

```javascript
// Symbols enable language-level customization
const obj = {
  [Symbol.iterator]: function* () {
    yield 1;
    yield 2;
    yield 3;
  }
};

for (const val of obj) {
  console.log(val); // 1, 2, 3
}

// Custom behavior for built-in operations
const collection = {
  items: [1, 2, 3, 4, 5],
  
  [Symbol.iterator]() {
    let index = 0;
    return {
      next: () => {
        if (index < this.items.length) {
          return { value: this.items[index++], done: false };
        }
        return { done: true };
      }
    };
  }
};

console.log([...collection]); // [1, 2, 3, 4, 5]
```

## Global Symbol Registry

### Symbol.for() and Symbol.keyFor()

```javascript
// Create/retrieve symbol from global registry
const globalSym1 = Symbol.for('app.id');
const globalSym2 = Symbol.for('app.id');

console.log(globalSym1 === globalSym2); // true (same symbol from registry)

// Regular symbols are not in registry
const localSym = Symbol('app.id');
console.log(localSym === globalSym1); // false

// Get key from global symbol
console.log(Symbol.keyFor(globalSym1)); // 'app.id'
console.log(Symbol.keyFor(localSym)); // undefined (not in registry)
```

### Cross-Realm Symbols

```javascript
// Global symbols work across realms (iframes, workers)
// In main page:
const mainSymbol = Symbol.for('shared.key');

// In iframe:
const iframeSymbol = Symbol.for('shared.key');

// mainSymbol === iframeSymbol (true, same global symbol)

// Use case: Shared metadata across contexts
const METADATA_KEY = Symbol.for('app.metadata');

class Component {
  static [METADATA_KEY] = {
    version: '1.0.0',
    type: 'component'
  };
}

// Accessible across different JavaScript contexts
```

### When to Use Global vs Local Symbols

```javascript
// Local symbols: Default choice, truly unique
const privateId = Symbol('id'); // Use for privacy, avoiding collisions

// Global symbols: When sharing across modules/contexts
const sharedId = Symbol.for('shared.id'); // Use for cross-module communication

// Example: Metadata system
const TYPE_SYMBOL = Symbol.for('meta.type');
const PRIVATE_DATA = Symbol('privateData');

class Entity {
  static [TYPE_SYMBOL] = 'Entity'; // Global (for external tools)
  [PRIVATE_DATA] = {}; // Local (truly private to this module)
}

// External code can check type
function getType(obj) {
  return obj.constructor[Symbol.for('meta.type')];
}

console.log(getType(new Entity())); // 'Entity'
```

## Well-Known Symbols

Well-known symbols are built-in symbols that customize language behavior.

### Symbol.iterator

```javascript
// Make objects iterable
const range = {
  from: 1,
  to: 5,
  
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    
    return {
      next() {
        if (current <= last) {
          return { value: current++, done: false };
        }
        return { done: true };
      }
    };
  }
};

console.log([...range]); // [1, 2, 3, 4, 5]
for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}

// String iterator example
const str = 'Hello';
const iterator = str[Symbol.iterator]();
console.log(iterator.next()); // { value: 'H', done: false }
console.log(iterator.next()); // { value: 'e', done: false }
```

### Symbol.toStringTag

```javascript
// Customize Object.prototype.toString() output
class MyClass {
  get [Symbol.toStringTag]() {
    return 'MyClass';
  }
}

const obj = new MyClass();
console.log(Object.prototype.toString.call(obj)); // '[object MyClass]'
console.log(obj.toString()); // '[object MyClass]'

// Compare with default
class Default {}
console.log(Object.prototype.toString.call(new Default())); // '[object Object]'

// Built-in examples
console.log(Object.prototype.toString.call([])); // '[object Array]'
console.log(Object.prototype.toString.call(new Map())); // '[object Map]'
console.log(Object.prototype.toString.call(new Set())); // '[object Set]'
```

### Symbol.toPrimitive

```javascript
// Customize type coercion (covered in coercion-type-conversion.md)
const user = {
  name: 'Alice',
  balance: 100,
  
  [Symbol.toPrimitive](hint) {
    console.log(`Hint: ${hint}`);
    
    if (hint === 'number') {
      return this.balance;
    }
    if (hint === 'string') {
      return this.name;
    }
    return this.balance; // default
  }
};

console.log(+user); // Hint: number → 100
console.log(`${user}`); // Hint: string → 'Alice'
console.log(user + 50); // Hint: default → 150
```

### Symbol.hasInstance

```javascript
// Customize instanceof behavior
class MyArray {
  static [Symbol.hasInstance](instance) {
    return Array.isArray(instance);
  }
}

console.log([] instanceof MyArray); // true
console.log([1, 2, 3] instanceof MyArray); // true
console.log({} instanceof MyArray); // false

// Practical use: Duck typing
class Drivable {
  static [Symbol.hasInstance](obj) {
    return typeof obj.drive === 'function' &&
           typeof obj.stop === 'function';
  }
}

const car = {
  drive() { console.log('Driving'); },
  stop() { console.log('Stopping'); }
};

console.log(car instanceof Drivable); // true (duck typing)
```

### Symbol.species

```javascript
// Control constructor for derived objects
class MyArray extends Array {
  static get [Symbol.species]() {
    return Array; // Return Array instead of MyArray
  }
}

const myArr = new MyArray(1, 2, 3);
const mapped = myArr.map(x => x * 2);

console.log(mapped instanceof MyArray); // false
console.log(mapped instanceof Array); // true

// Why? Array methods create new instances using constructor[Symbol.species]

// Without Symbol.species
class MyArray2 extends Array {}
const myArr2 = new MyArray2(1, 2, 3);
const mapped2 = myArr2.map(x => x * 2);

console.log(mapped2 instanceof MyArray2); // true (default behavior)
```

### Symbol.isConcatSpreadable

```javascript
// Control array spreading in Array.prototype.concat()
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];

console.log(arr1.concat(arr2)); // [1, 2, 3, 4, 5, 6] (default: spread)

// Prevent spreading
arr2[Symbol.isConcatSpreadable] = false;
console.log(arr1.concat(arr2)); // [1, 2, 3, [4, 5, 6]] (not spread)

// Make array-like objects spreadable
const arrayLike = {
  length: 2,
  0: 'a',
  1: 'b',
  [Symbol.isConcatSpreadable]: true
};

console.log([].concat(arrayLike)); // ['a', 'b']
```

### Symbol.match, Symbol.replace, Symbol.search, Symbol.split

```javascript
// Customize string methods
const customMatcher = {
  [Symbol.match](str) {
    return str.includes('custom') ? ['match found'] : null;
  },
  
  [Symbol.replace](str, replacement) {
    return str.replace('custom', replacement);
  },
  
  [Symbol.search](str) {
    return str.indexOf('custom');
  },
  
  [Symbol.split](str) {
    return str.split('custom');
  }
};

const text = 'This is a custom string';

console.log(text.match(customMatcher)); // ['match found']
console.log(text.replace(customMatcher, 'special')); // 'This is a special string'
console.log(text.search(customMatcher)); // 10
console.log(text.split(customMatcher)); // ['This is a ', ' string']

// RegExp uses these symbols internally
const regex = /test/;
'testing'.match(regex); // Calls regex[Symbol.match]
```

### Symbol.unscopables

```javascript
// Exclude properties from 'with' statement (legacy, avoid 'with')
const obj = {
  a: 1,
  b: 2,
  [Symbol.unscopables]: {
    b: true // Exclude 'b' from with scope
  }
};

with (obj) {
  console.log(a); // 1 (included)
  // console.log(b); // ReferenceError (excluded)
}

// Array example: Why some methods not in 'with' scope
console.log(Array.prototype[Symbol.unscopables]);
// { copyWithin: true, entries: true, fill: true, find: true, ... }

// This prevents with breaking when new methods added to Array.prototype
```

## Symbol Use Cases

### Enum-Like Values

```javascript
// Symbols provide unique, immutable enum values
const Color = {
  RED: Symbol('red'),
  GREEN: Symbol('green'),
  BLUE: Symbol('blue')
};

function getColorName(color) {
  switch (color) {
    case Color.RED:
      return 'Red';
    case Color.GREEN:
      return 'Green';
    case Color.BLUE:
      return 'Blue';
    default:
      return 'Unknown';
  }
}

console.log(getColorName(Color.RED)); // 'Red'

// Benefits:
// - Can't accidentally create duplicate values
// - No string collision issues
// - Type-safe (can't use wrong string)
```

### Private Class Members (Pre-# Syntax)

```javascript
// Before private fields, symbols provided privacy
const _balance = Symbol('balance');
const _transactions = Symbol('transactions');

class BankAccount {
  constructor(initialBalance) {
    this[_balance] = initialBalance;
    this[_transactions] = [];
  }
  
  deposit(amount) {
    this[_balance] += amount;
    this[_transactions].push({ type: 'deposit', amount });
  }
  
  getBalance() {
    return this[_balance];
  }
  
  getTransactions() {
    return [...this[_transactions]]; // Return copy
  }
}

const account = new BankAccount(1000);
account.deposit(500);
console.log(account.getBalance()); // 1500

// "Private" data not easily accessible
console.log(account._balance); // undefined
console.log(account[Symbol('balance')]); // undefined (different symbol)

// Still accessible with Object.getOwnPropertySymbols
const symbols = Object.getOwnPropertySymbols(account);
console.log(account[symbols[0]]); // 1500 (not truly private)
```

### Object Metadata

```javascript
// Attach metadata without polluting namespace
const CREATED_AT = Symbol('createdAt');
const AUTHOR = Symbol('author');
const VERSION = Symbol('version');

class Document {
  constructor(content, author) {
    this.content = content;
    this[CREATED_AT] = new Date();
    this[AUTHOR] = author;
    this[VERSION] = 1;
  }
  
  update(content) {
    this.content = content;
    this[VERSION]++;
  }
  
  getMetadata() {
    return {
      createdAt: this[CREATED_AT],
      author: this[AUTHOR],
      version: this[VERSION]
    };
  }
}

const doc = new Document('Hello World', 'Alice');
console.log(doc.content); // 'Hello World'
console.log(Object.keys(doc)); // ['content'] (metadata hidden)
console.log(doc.getMetadata()); // { createdAt: ..., author: 'Alice', version: 1 }
```

### Protocol Implementation

```javascript
// Implement custom protocols using symbols
const Comparable = {
  compare: Symbol('compare')
};

class Version {
  constructor(major, minor, patch) {
    this.major = major;
    this.minor = minor;
    this.patch = patch;
  }
  
  [Comparable.compare](other) {
    if (this.major !== other.major) return this.major - other.major;
    if (this.minor !== other.minor) return this.minor - other.minor;
    return this.patch - other.patch;
  }
}

function max(a, b) {
  if (a[Comparable.compare] && b[Comparable.compare]) {
    return a[Comparable.compare](b) > 0 ? a : b;
  }
  return a > b ? a : b;
}

const v1 = new Version(1, 2, 3);
const v2 = new Version(1, 3, 0);
console.log(max(v1, v2)); // Version { major: 1, minor: 3, patch: 0 }
```

### Extension Points

```javascript
// Provide hooks for customization
const VALIDATOR = Symbol('validator');
const SERIALIZER = Symbol('serializer');

class Model {
  constructor(data) {
    this.data = data;
  }
  
  validate() {
    if (this[VALIDATOR]) {
      return this[VALIDATOR](this.data);
    }
    return true; // Default: always valid
  }
  
  serialize() {
    if (this[SERIALIZER]) {
      return this[SERIALIZER](this.data);
    }
    return JSON.stringify(this.data); // Default serialization
  }
}

class User extends Model {
  [VALIDATOR](data) {
    return data.email && data.email.includes('@');
  }
  
  [SERIALIZER](data) {
    return JSON.stringify({
      email: data.email,
      // Omit sensitive data
    });
  }
}

const user = new User({ email: 'test@example.com', password: 'secret' });
console.log(user.validate()); // true
console.log(user.serialize()); // '{"email":"test@example.com"}'
```

## Common Misconceptions

### Misconception 1: "Symbols Provide True Privacy"

**Reality**: Symbols are hidden from normal enumeration but accessible via `Object.getOwnPropertySymbols()` and `Reflect.ownKeys()`.

```javascript
const secret = Symbol('secret');
const obj = { [secret]: 'hidden' };

// Not in normal enumeration
console.log(Object.keys(obj)); // []

// But accessible with specific methods
console.log(Object.getOwnPropertySymbols(obj)); // [Symbol(secret)]
console.log(obj[Object.getOwnPropertySymbols(obj)[0]]); // 'hidden'

// For true privacy, use # private fields (ES2022)
class WithPrivate {
  #secret = 'truly hidden';
}
```

### Misconception 2: "Symbol Descriptions Are Unique"

**Reality**: Descriptions are just for debugging; they don't affect uniqueness.

```javascript
const sym1 = Symbol('id');
const sym2 = Symbol('id');

console.log(sym1 === sym2); // false (different symbols)
console.log(sym1.description === sym2.description); // true (same description)

// Only Symbol.for creates/retrieves symbols by key
const global1 = Symbol.for('id');
const global2 = Symbol.for('id');
console.log(global1 === global2); // true
```

### Misconception 3: "Symbols Can Be Used as String Keys"

**Reality**: Symbols and strings are completely separate key spaces.

```javascript
const sym = Symbol('key');
const obj = {
  key: 'string key',
  [sym]: 'symbol key'
};

console.log(obj.key); // 'string key'
console.log(obj['key']); // 'string key'
console.log(obj[sym]); // 'symbol key'
console.log(obj['Symbol(key)']); // undefined (can't access symbol as string)
```

### Misconception 4: "All Symbols Are Global"

**Reality**: Only symbols created with `Symbol.for()` are in the global registry.

```javascript
// Local symbol (not global)
const local = Symbol('test');

// Global symbol
const global = Symbol.for('test');

console.log(local === global); // false
console.log(Symbol.keyFor(local)); // undefined (not global)
console.log(Symbol.keyFor(global)); // 'test' (global)
```

## Performance Implications

### Symbol Property Access

```javascript
// Symbol property access is fast (similar to string keys)
const sym = Symbol('key');
const obj = {
  stringKey: 'value',
  [sym]: 'value'
};

// Both are O(1) lookups
obj.stringKey; // Fast
obj[sym]; // Fast

// Symbols don't slow down property access
```

### Symbol Creation

```javascript
// Symbol() creation is relatively lightweight
console.time('symbol');
for (let i = 0; i < 100000; i++) {
  Symbol();
}
console.timeEnd('symbol'); // ~5-10ms

// Symbol.for() is slightly slower (registry lookup)
console.time('symbol.for');
for (let i = 0; i < 100000; i++) {
  Symbol.for(`key${i}`);
}
console.timeEnd('symbol.for'); // ~10-20ms

// Still very fast; not a performance concern in practice
```

### Memory Usage

```javascript
// Symbols have minimal memory overhead
const symbols = Array.from({ length: 10000 }, (_, i) => Symbol(`sym${i}`));

// Each symbol uses ~24-32 bytes (description + unique ID)
// Negligible compared to actual object data

// Global symbols persist forever (can't be garbage collected)
for (let i = 0; i < 10000; i++) {
  Symbol.for(`global${i}`); // Creates permanent entry in global registry
}
// Use local symbols when possible to allow GC
```

## Interview Questions

### Question 1: What are symbols and why were they introduced?

**Answer**: Symbols are a primitive type introduced in ES6 that represent unique, immutable identifiers. They were added to solve three main problems: avoiding property name collisions (libraries can add properties without conflicts), creating "hidden" properties (not enumerable by default), and enabling meta-programming through well-known symbols that customize language behavior (like Symbol.iterator).

### Question 2: What's the difference between Symbol() and Symbol.for()?

**Answer**: `Symbol()` creates a new unique symbol each time, while `Symbol.for(key)` creates/retrieves a symbol from a global registry. If a symbol with that key exists in the registry, `Symbol.for()` returns it; otherwise, it creates a new one. Use `Symbol()` for truly unique values and `Symbol.for()` when you need to share symbols across modules or realms.

### Question 3: Name at least 5 well-known symbols and their purposes

**Answer**:
1. `Symbol.iterator`: Makes objects iterable (for-of, spread)
2. `Symbol.toStringTag`: Customizes `Object.prototype.toString()` output
3. `Symbol.toPrimitive`: Controls type coercion to primitives
4. `Symbol.hasInstance`: Customizes `instanceof` behavior
5. `Symbol.species`: Controls constructor for derived objects
6. `Symbol.match/replace/search/split`: Customize string methods

### Question 4: Do symbols provide true privacy?

**Answer**: No, symbols don't provide true privacy. While they're hidden from `Object.keys()`, `for...in`, and `JSON.stringify()`, they're still accessible via `Object.getOwnPropertySymbols()` and `Reflect.ownKeys()`. For true privacy, use private fields (# syntax) introduced in ES2022, or closures in constructor functions.

### Question 5: How do you make an object iterable?

**Answer**: Implement the `Symbol.iterator` method that returns an iterator object with a `next()` method:

```javascript
const obj = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() {
        if (i < 3) {
          return { value: i++, done: false };
        }
        return { done: true };
      }
    };
  }
};

for (const val of obj) {
  console.log(val); // 0, 1, 2
}
```

## Key Takeaways

1. **Symbols are unique**: Each `Symbol()` call creates a unique identifier

2. **Symbols enable meta-programming**: Well-known symbols customize language behavior

3. **Symbols avoid collisions**: Safe for adding properties without conflicts

4. **Symbol properties are hidden**: Not in `Object.keys()`, `for...in`, `JSON.stringify()`

5. **Not true privacy**: Accessible via `Object.getOwnPropertySymbols()`

6. **Global registry**: `Symbol.for()` creates/retrieves shared symbols

7. **Well-known symbols**: Built-in symbols for customizing iteration, coercion, instanceof, etc.

8. **Performance**: Symbol property access is as fast as string keys

9. **Use cases**: Enums, metadata, protocols, extension points

10. **Choose appropriately**: Local symbols by default, global only when needed for sharing

## Resources

### Official Documentation
- [MDN: Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)
- [MDN: Well-known symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#well-known_symbols)
- [ECMAScript Specification: Symbols](https://tc39.es/ecma262/#sec-ecmascript-language-types-symbol-type)

### Articles
- "Understanding JavaScript Symbols" by Axel Rauschmayer
- "ES6 Symbols in Depth" by Pony Foo
- "JavaScript Symbols: The Most Misunderstood Feature?" by JavaScript Weekly

### Books
- "Exploring ES6" by Axel Rauschmayer
- "You Don't Know JS: ES6 & Beyond" by Kyle Simpson
- "JavaScript: The Definitive Guide" by David Flanagan

### Videos
- "JavaScript Symbols" by Fun Fun Function
- "ES6 Symbols Explained" by Traversy Media
- "Well-Known Symbols" by JavaScript conferences
