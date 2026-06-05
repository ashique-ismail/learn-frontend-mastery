# Implement Deep Clone

## The Idea

**In plain English:** Deep cloning means making a completely independent copy of something — every layer of it — so that changing the copy never affects the original. Unlike a shallow copy (which only copies the top layer), a deep clone digs all the way down through every nested piece of data.

**Real-world analogy:** Imagine photocopying a scrapbook. A shallow copy is like tearing out only the cover page and taping on the same photos that are still glued inside the original — if someone writes on those photos, both copies change. A deep clone is like photocopying every single page, cutting out every photo, and reprinting each one individually, so the two scrapbooks are completely separate forever.

- The scrapbook cover = the top-level object
- The pages and photos inside = nested objects and arrays
- Reprinting every photo independently = recursively cloning each nested value

---

## Overview

Implement deep cloning for JavaScript objects, handling circular references, special objects (Date, RegExp, Map, Set), and performance optimization.

## Basic Deep Clone

```typescript
function deepClone<T>(value: T): T {
  // Primitives and null
  if (value === null || typeof value !== 'object') {
    return value;
  }

  // Arrays
  if (Array.isArray(value)) {
    return value.map(item => deepClone(item)) as any;
  }

  // Objects
  const cloned: any = {};
  for (const key in value) {
    if (value.hasOwnProperty(key)) {
      cloned[key] = deepClone((value as any)[key]);
    }
  }

  return cloned;
}
```

## Handling Circular References

```typescript
function deepCloneWithCircular<T>(value: T, cache = new WeakMap()): T {
  // Primitives and null
  if (value === null || typeof value !== 'object') {
    return value;
  }

  // Check cache for circular references
  if (cache.has(value as any)) {
    return cache.get(value as any);
  }

  // Arrays
  if (Array.isArray(value)) {
    const cloned: any[] = [];
    cache.set(value as any, cloned as any);
    
    value.forEach((item, index) => {
      cloned[index] = deepCloneWithCircular(item, cache);
    });
    
    return cloned as any;
  }

  // Objects
  const cloned: any = {};
  cache.set(value as any, cloned);

  for (const key in value) {
    if (value.hasOwnProperty(key)) {
      cloned[key] = deepCloneWithCircular((value as any)[key], cache);
    }
  }

  return cloned;
}

// Test circular reference
const obj: any = { name: 'test' };
obj.self = obj;
const cloned = deepCloneWithCircular(obj);
console.log(cloned.self === cloned); // true
```

## Handling Special Objects

```typescript
function deepCloneFull<T>(value: T, cache = new WeakMap()): T {
  // Primitives and null
  if (value === null || typeof value !== 'object') {
    return value;
  }

  // Check cache
  if (cache.has(value as any)) {
    return cache.get(value as any);
  }

  // Date
  if (value instanceof Date) {
    return new Date(value.getTime()) as any;
  }

  // RegExp
  if (value instanceof RegExp) {
    const flags = value.flags;
    const cloned = new RegExp(value.source, flags);
    cloned.lastIndex = value.lastIndex;
    return cloned as any;
  }

  // Map
  if (value instanceof Map) {
    const cloned = new Map();
    cache.set(value as any, cloned as any);
    
    value.forEach((val, key) => {
      cloned.set(
        deepCloneFull(key, cache),
        deepCloneFull(val, cache)
      );
    });
    
    return cloned as any;
  }

  // Set
  if (value instanceof Set) {
    const cloned = new Set();
    cache.set(value as any, cloned as any);
    
    value.forEach(val => {
      cloned.add(deepCloneFull(val, cache));
    });
    
    return cloned as any;
  }

  // ArrayBuffer
  if (value instanceof ArrayBuffer) {
    const cloned = new ArrayBuffer(value.byteLength);
    new Uint8Array(cloned).set(new Uint8Array(value));
    return cloned as any;
  }

  // TypedArrays
  if (ArrayBuffer.isView(value) && !(value instanceof DataView)) {
    const TypedArrayConstructor = value.constructor as any;
    return new TypedArrayConstructor(value) as any;
  }

  // Arrays
  if (Array.isArray(value)) {
    const cloned: any[] = [];
    cache.set(value as any, cloned as any);
    
    value.forEach((item, index) => {
      cloned[index] = deepCloneFull(item, cache);
    });
    
    return cloned as any;
  }

  // Plain objects
  const cloned: any = Object.create(Object.getPrototypeOf(value));
  cache.set(value as any, cloned);

  // Copy own properties
  Object.getOwnPropertyNames(value).forEach(key => {
    const descriptor = Object.getOwnPropertyDescriptor(value, key);
    if (descriptor) {
      if (descriptor.value !== undefined) {
        descriptor.value = deepCloneFull(descriptor.value, cache);
      }
      Object.defineProperty(cloned, key, descriptor);
    }
  });

  // Copy symbols
  Object.getOwnPropertySymbols(value).forEach(symbol => {
    const descriptor = Object.getOwnPropertyDescriptor(value, symbol);
    if (descriptor) {
      if (descriptor.value !== undefined) {
        descriptor.value = deepCloneFull(descriptor.value, cache);
      }
      Object.defineProperty(cloned, symbol, descriptor);
    }
  });

  return cloned;
}
```

## Performance Optimized Version

```typescript
function deepCloneOptimized<T>(value: T): T {
  // Fast path for primitives
  const type = typeof value;
  if (value === null || (type !== 'object' && type !== 'function')) {
    return value;
  }

  const cache = new WeakMap();

  function clone(val: any): any {
    // Check cache
    if (cache.has(val)) {
      return cache.get(val);
    }

    // Handle special cases with fast checks
    const constructor = val.constructor;

    if (constructor === Date) {
      return new Date(val);
    }

    if (constructor === RegExp) {
      return new RegExp(val.source, val.flags);
    }

    if (constructor === Map) {
      const cloned = new Map();
      cache.set(val, cloned);
      val.forEach((v: any, k: any) => {
        cloned.set(clone(k), clone(v));
      });
      return cloned;
    }

    if (constructor === Set) {
      const cloned = new Set();
      cache.set(val, cloned);
      val.forEach((v: any) => {
        cloned.add(clone(v));
      });
      return cloned;
    }

    // Arrays
    if (Array.isArray(val)) {
      const cloned: any[] = [];
      cache.set(val, cloned);
      for (let i = 0; i < val.length; i++) {
        cloned[i] = clone(val[i]);
      }
      return cloned;
    }

    // Objects
    const cloned: any = {};
    cache.set(val, cloned);

    const keys = Object.keys(val);
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i];
      cloned[key] = clone(val[key]);
    }

    return cloned;
  }

  return clone(value);
}
```

## structuredClone Comparison

```typescript
// Modern browser API (Chrome 98+, Firefox 94+)
const original = {
  date: new Date(),
  map: new Map([[1, 'one']]),
  set: new Set([1, 2, 3]),
  buffer: new ArrayBuffer(8)
};

// Using structuredClone (built-in)
const cloned1 = structuredClone(original);

// Using custom implementation
const cloned2 = deepCloneFull(original);

// Performance comparison
console.time('structuredClone');
for (let i = 0; i < 10000; i++) {
  structuredClone(original);
}
console.timeEnd('structuredClone');

console.time('deepCloneFull');
for (let i = 0; i < 10000; i++) {
  deepCloneFull(original);
}
console.timeEnd('deepCloneFull');
```

## Edge Cases

```typescript
// Test suite for edge cases
describe('Deep Clone Edge Cases', () => {
  it('should handle circular references', () => {
    const obj: any = { a: 1 };
    obj.self = obj;
    
    const cloned = deepCloneFull(obj);
    expect(cloned.self).toBe(cloned);
  });

  it('should clone Date objects', () => {
    const date = new Date('2024-01-01');
    const cloned = deepCloneFull(date);
    
    expect(cloned).toEqual(date);
    expect(cloned).not.toBe(date);
  });

  it('should clone RegExp with flags', () => {
    const regex = /test/gi;
    const cloned = deepCloneFull(regex);
    
    expect(cloned.source).toBe('test');
    expect(cloned.flags).toBe('gi');
  });

  it('should clone Map', () => {
    const map = new Map([
      ['key1', 'value1'],
      [{ objKey: true }, { objValue: true }]
    ]);
    
    const cloned = deepCloneFull(map);
    expect(cloned.size).toBe(2);
    expect(cloned).not.toBe(map);
  });

  it('should clone Set', () => {
    const set = new Set([1, 2, { a: 3 }]);
    const cloned = deepCloneFull(set);
    
    expect(cloned.size).toBe(3);
    expect(cloned).not.toBe(set);
  });

  it('should clone nested arrays', () => {
    const arr = [1, [2, [3, [4]]]];
    const cloned = deepCloneFull(arr);
    
    expect(cloned).toEqual(arr);
    expect(cloned[1]).not.toBe(arr[1]);
  });

  it('should handle null and undefined', () => {
    const obj = { a: null, b: undefined };
    const cloned = deepCloneFull(obj);
    
    expect(cloned.a).toBeNull();
    expect(cloned.b).toBeUndefined();
  });

  it('should clone symbol properties', () => {
    const sym = Symbol('test');
    const obj = { [sym]: 'value' };
    
    const cloned = deepCloneFull(obj);
    expect(cloned[sym]).toBe('value');
  });

  it('should preserve prototype chain', () => {
    class MyClass {
      method() { return 'test'; }
    }
    
    const obj = new MyClass();
    const cloned = deepCloneFull(obj);
    
    expect(cloned.method()).toBe('test');
    expect(Object.getPrototypeOf(cloned)).toBe(MyClass.prototype);
  });
});
```

## React/Angular Usage

```typescript
// React immutable state update
function updateNestedState(state: any, path: string[], value: any) {
  const cloned = deepClone(state);
  let current = cloned;
  
  for (let i = 0; i < path.length - 1; i++) {
    current = current[path[i]];
  }
  
  current[path[path.length - 1]] = value;
  return cloned;
}

// Usage in React
const [state, setState] = useState({
  user: {
    profile: {
      name: 'John'
    }
  }
});

setState(prevState => 
  updateNestedState(prevState, ['user', 'profile', 'name'], 'Jane')
);

// Angular immutable update
class MyService {
  private stateSubject = new BehaviorSubject(initialState);
  state$ = this.stateSubject.asObservable();

  updateState(updater: (state: any) => any) {
    const currentState = this.stateSubject.value;
    const newState = updater(deepClone(currentState));
    this.stateSubject.next(newState);
  }
}
```

## Key Takeaways

1. **Circular References**: Use WeakMap to track cloned objects and handle circular references.

2. **Special Objects**: Handle Date, RegExp, Map, Set, TypedArrays, ArrayBuffer separately.

3. **Prototype Chain**: Preserve prototype chain with Object.create(Object.getPrototypeOf()).

4. **Symbol Properties**: Clone symbol properties using Object.getOwnPropertySymbols().

5. **Performance**: For best performance, check object type early and use optimized paths.

6. **structuredClone**: Modern browsers have built-in structuredClone() - use when available.

7. **Property Descriptors**: Use Object.defineProperty() to preserve getters/setters/configurable.

8. **TypedArrays**: Handle TypedArray views (Uint8Array, etc.) with constructor copying.

9. **WeakMap for Cache**: WeakMap automatically handles garbage collection for cache.

10. **Trade-offs**: Deep cloning is expensive - consider alternatives like immutable data structures.

## Interview Talking Points

- Explain deep clone vs shallow clone
- Describe circular reference handling with WeakMap
- Discuss special object types and their cloning
- Compare custom implementation vs structuredClone
- Explain performance optimizations
- Describe prototype chain preservation
- Discuss symbol property handling
- Explain when deep cloning is necessary
- Compare with immutable data structures (Immutable.js, Immer)
- Discuss memory and performance trade-offs
