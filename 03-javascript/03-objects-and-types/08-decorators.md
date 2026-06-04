# JavaScript Decorators

## Overview

Decorators are a TC39 Stage 3 proposal (as of 2024) that adds a declarative syntax for wrapping and augmenting classes, methods, fields, and accessors. TypeScript 5.0+ supports the standard proposal. Angular has always used an older (TypeScript-experimental) decorator system; the ecosystem is gradually migrating to the standard.

---

## Syntax

Decorators are functions applied with `@` before a class declaration or class member.

```javascript
// Class decorator
@logged
class MyService { ... }

// Method decorator
class MyClass {
  @memoize
  expensiveMethod(x) { ... }

  @deprecated('Use newMethod instead')
  oldMethod() { ... }
}

// Accessor decorator
class Circle {
  @clamp(0, Infinity)
  accessor radius = 0;
}

// Field decorator
class Form {
  @required
  accessor name = '';
}
```

---

## Decorator Types (TC39 Standard)

### Class Decorator

Receives the class constructor and a context object. Can return a new class or modify the existing one.

```javascript
function singleton(Class, context) {
  let instance;
  return class extends Class {
    constructor(...args) {
      if (instance) return instance;
      super(...args);
      instance = this;
    }
  };
}

@singleton
class Database {
  constructor() {
    this.connection = createConnection();
  }
}

const db1 = new Database();
const db2 = new Database();
console.log(db1 === db2); // true
```

### Method Decorator

```javascript
function log(method, context) {
  const name = context.name;
  return function (...args) {
    console.log(`Calling ${name} with`, args);
    const result = method.apply(this, args);
    console.log(`${name} returned`, result);
    return result;
  };
}

class Calculator {
  @log
  add(a, b) {
    return a + b;
  }
}

new Calculator().add(2, 3);
// Calling add with [2, 3]
// add returned 5
```

### Accessor Decorator (`accessor` keyword)

The `accessor` keyword creates a backing private field with auto-generated getter/setter — decorators can intercept both.

```javascript
function clamp(min, max) {
  return function (target, context) {
    const { get, set } = target;
    return {
      get() { return get.call(this); },
      set(value) {
        set.call(this, Math.min(max, Math.max(min, value)));
      },
    };
  };
}

class Circle {
  @clamp(0, 1000)
  accessor radius = 0;
}

const c = new Circle();
c.radius = -10;
console.log(c.radius); // 0
c.radius = 9999;
console.log(c.radius); // 1000
```

### Field Decorator

```javascript
function required(_, context) {
  const name = context.name;
  context.addInitializer(function () {
    if (this[name] === undefined || this[name] === '') {
      throw new Error(`${name} is required`);
    }
  });
}

// Using context.addInitializer for setup logic
function bindThis(method, context) {
  const name = context.name;
  context.addInitializer(function () {
    this[name] = method.bind(this);
  });
}

class EventHandler {
  @bindThis
  handleClick(event) {
    console.log(this); // always the class instance, even as a callback
  }
}

const h = new EventHandler();
document.addEventListener('click', h.handleClick); // `this` is still h
```

---

## Context Object

Every decorator receives a context object as its second argument:

```javascript
function inspect(_, context) {
  console.log({
    kind:     context.kind,        // 'class' | 'method' | 'field' | 'accessor' | 'getter' | 'setter'
    name:     context.name,        // string or Symbol
    static:   context.static,      // boolean
    private:  context.private,     // boolean
    access:   context.access,      // { get?, set? } — direct field access
    addInitializer: context.addInitializer, // schedule code to run after class finalization
  });
}
```

---

## Practical Patterns

### Memoization

```javascript
function memoize(method, context) {
  const cache = new WeakMap();

  return function (...args) {
    if (!cache.has(this)) cache.set(this, new Map());
    const instanceCache = cache.get(this);
    const key = JSON.stringify(args);
    if (instanceCache.has(key)) return instanceCache.get(key);
    const result = method.apply(this, args);
    instanceCache.set(key, result);
    return result;
  };
}

class MathService {
  @memoize
  fibonacci(n) {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}
```

### Validation / Deprecation

```javascript
function deprecated(message) {
  return function (method, context) {
    return function (...args) {
      console.warn(`[Deprecated] ${String(context.name)}: ${message}`);
      return method.apply(this, args);
    };
  };
}

function validate(schema) {
  return function (method, context) {
    return function (data) {
      const result = schema.safeParse(data);
      if (!result.success) throw new Error(result.error.message);
      return method.call(this, result.data);
    };
  };
}
```

### Timing / Performance

```javascript
function measure(method, context) {
  return function (...args) {
    const label = `${context.name}`;
    performance.mark(`${label}-start`);
    const result = method.apply(this, args);
    performance.mark(`${label}-end`);
    performance.measure(label, `${label}-start`, `${label}-end`);
    return result;
  };
}
```

---

## TypeScript: Experimental vs Standard Decorators

Angular currently uses `experimentalDecorators: true` — this is the older TypeScript decorator spec. The standard TC39 decorators are different:

| Feature | `experimentalDecorators` | TC39 Standard (TS 5+) |
|---|---|---|
| `tsconfig` flag | `experimentalDecorators: true` | No flag needed |
| Decorator factories | Common (`@Component({...})`) | Same syntax |
| Class decorator return | Replaces class | Must return same type |
| Method descriptor | Receives PropertyDescriptor | Receives the function itself |
| `accessor` keyword | Not supported | Required for field interception |
| Parameter decorators | Supported (Angular DI) | Not in TC39 proposal |
| Metadata (`emitDecoratorMetadata`) | Available | Separate proposal |

```typescript
// TypeScript 5+ standard decorator
function Component(config: { selector: string }) {
  return function <T extends new (...args: any[]) => object>(Base: T, context: ClassDecoratorContext) {
    context.addInitializer(function (this: any) {
      this.selector = config.selector;
    });
    return Base;
  };
}

// Still uses experimental decorators (Angular-style)
// tsconfig: { "experimentalDecorators": true }
function Component(config: { selector: string }) {
  return function (target: Function) {
    Reflect.defineMetadata('selector', config.selector, target);
  };
}
```

---

## Angular Decorator Examples (Experimental)

Angular's decorators (`@Component`, `@Injectable`, `@Input`) use the old spec with `Reflect.metadata`:

```typescript
@Injectable({ providedIn: 'root' })
class UserService {
  @Inject(HTTP_CLIENT) private http: HttpClient;

  @HostListener('click', ['$event'])
  onClick(event: MouseEvent) { ... }

  @ViewChild('container')
  container!: ElementRef;
}
```

Angular 18+ is gradually introducing standard decorator alternatives via signals (`input()`, `output()`, `viewChild()`). The framework plans to deprecate experimental decorators over time.

---

## Decorator Composition

Decorators stack bottom-up (innermost first when multiple decorators on one target):

```javascript
function A(method) { return function() { console.log('A in'); const r = method.apply(this, arguments); console.log('A out'); return r; }; }
function B(method) { return function() { console.log('B in'); const r = method.apply(this, arguments); console.log('B out'); return r; }; }

class Example {
  @A
  @B  // B wraps the original; A wraps B
  greet() { console.log('hello'); }
}

new Example().greet();
// A in
// B in
// hello
// B out
// A out
```

---

## Interview Questions

**Q: What problem do decorators solve?**  
A: They separate cross-cutting concerns (logging, caching, validation, access control) from business logic, applying them declaratively. Without decorators you'd wrap methods manually or use mixins, which is verbose and error-prone.

**Q: What's the difference between the TC39 standard decorators and TypeScript's `experimentalDecorators`?**  
A: The experimental spec was written before the TC39 proposal stabilized. Key differences: experimental decorators receive `PropertyDescriptor` objects; standard ones receive the value directly. Standard decorators use `accessor` for field interception. Angular uses the experimental spec; TypeScript 5+ ships both.

**Q: Are decorators evaluated at class definition time or instance creation time?**  
A: Decorator functions run at class definition time (when the `class` statement executes), not when instances are created. The `addInitializer` callback runs at instance creation. This is why Angular's `@Component` metadata is available before any component is instantiated.

**Q: Can decorators mutate the original class?**  
A: In the TC39 standard, class decorators must return a value of the same type (or nothing). They can't return an unrelated type. This is stricter than the experimental spec, which allowed returning any object.
