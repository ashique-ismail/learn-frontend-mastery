# Implement Memoization

## The Idea

**In plain English:** Memoization is a way to make a function faster by remembering the answers it already figured out, so the next time someone asks the same question, it just looks up the saved answer instead of doing the work all over again.

**Real-world analogy:** Imagine a student who keeps a cheat-sheet while doing homework. The first time they convert 5 miles to kilometers they work it out on paper and write the answer on the sheet. The second time the same conversion comes up, they just glance at the sheet instead of recalculating.

- The cheat-sheet = the cache (where saved results are stored)
- The math conversion = the function (the expensive work being done)
- The question "5 miles?" = the function argument (the input that determines which answer to look up)

---

## Overview

Implement memoization patterns and LRU cache from scratch to optimize function performance through caching.

## Basic Memoization

```typescript
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>) => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }

    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// Usage
const fibonacci = memoize((n: number): number => {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

console.log(fibonacci(40)); // Much faster with memoization
```

## Advanced Memoization with Options

```typescript
interface MemoizeOptions {
  maxSize?: number;
  maxAge?: number;
  keyGenerator?: (...args: any[]) => string;
}

function memoize<T extends (...args: any[]) => any>(
  fn: T,
  options: MemoizeOptions = {}
): T & { cache: Map<string, any>; clear: () => void } {
  const { maxSize = Infinity, maxAge = Infinity, keyGenerator } = options;
  
  const cache = new Map<string, { value: ReturnType<T>; timestamp: number }>();

  const memoized = ((...args: Parameters<T>) => {
    const key = keyGenerator ? keyGenerator(...args) : JSON.stringify(args);

    // Check if cached and not expired
    if (cache.has(key)) {
      const cached = cache.get(key)!;
      const age = Date.now() - cached.timestamp;
      
      if (age < maxAge) {
        return cached.value;
      } else {
        cache.delete(key);
      }
    }

    // Compute new value
    const result = fn(...args);
    
    // Evict oldest if at max size
    if (cache.size >= maxSize) {
      const firstKey = cache.keys().next().value;
      cache.delete(firstKey);
    }

    cache.set(key, { value: result, timestamp: Date.now() });
    return result;
  }) as T & { cache: Map<string, any>; clear: () => void };

  memoized.cache = cache as any;
  memoized.clear = () => cache.clear();

  return memoized;
}

// Usage with options
const expensiveOperation = memoize(
  (a: number, b: number) => {
    console.log('Computing...');
    return a + b;
  },
  { maxSize: 100, maxAge: 5000 }
);
```

## LRU Cache Implementation

```typescript
class LRUCache<K, V> {
  private capacity: number;
  private cache: Map<K, V>;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) {
      return undefined;
    }

    // Move to end (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    
    return value;
  }

  put(key: K, value: V): void {
    // Remove if exists
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }

    // Add to end
    this.cache.set(key, value);

    // Evict oldest if over capacity
    if (this.cache.size > this.capacity) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }

  has(key: K): boolean {
    return this.cache.has(key);
  }

  clear(): void {
    this.cache.clear();
  }

  get size(): number {
    return this.cache.size;
  }
}

// Usage with memoization
function memoizeWithLRU<T extends (...args: any[]) => any>(
  fn: T,
  cacheSize: number = 100
): T {
  const cache = new LRUCache<string, ReturnType<T>>(cacheSize);

  return ((...args: Parameters<T>) => {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key)!;
    }

    const result = fn(...args);
    cache.put(key, result);
    return result;
  }) as T;
}
```

## React useMemo Recreation

```typescript
// Simplified React useMemo implementation
function useMemo<T>(factory: () => T, deps: any[]): T {
  const memoizedValue = useRef<{ value: T; deps: any[] }>();

  if (!memoizedValue.current || !areDepsEqual(memoizedValue.current.deps, deps)) {
    memoizedValue.current = {
      value: factory(),
      deps
    };
  }

  return memoizedValue.current.value;
}

function areDepsEqual(prevDeps: any[], nextDeps: any[]): boolean {
  if (prevDeps.length !== nextDeps.length) {
    return false;
  }

  for (let i = 0; i < prevDeps.length; i++) {
    if (!Object.is(prevDeps[i], nextDeps[i])) {
      return false;
    }
  }

  return true;
}

// Usage in component
function ExpensiveComponent({ data }: { data: number[] }) {
  const sortedData = useMemo(() => {
    console.log('Sorting...');
    return [...data].sort((a, b) => a - b);
  }, [data]);

  return <div>{sortedData.join(', ')}</div>;
}
```

## Fibonacci Optimization

```typescript
// Without memoization - O(2^n)
function fibonacciSlow(n: number): number {
  if (n <= 1) return n;
  return fibonacciSlow(n - 1) + fibonacciSlow(n - 2);
}

// With memoization - O(n)
const fibonacciFast = memoize((n: number): number => {
  if (n <= 1) return n;
  return fibonacciFast(n - 1) + fibonacciFast(n - 2);
});

// Iterative with manual caching - O(n), O(n) space
function fibonacciIterative(n: number): number {
  if (n <= 1) return n;
  
  const cache: number[] = [0, 1];
  
  for (let i = 2; i <= n; i++) {
    cache[i] = cache[i - 1] + cache[i - 2];
  }
  
  return cache[n];
}

// Optimized iterative - O(n), O(1) space
function fibonacciOptimized(n: number): number {
  if (n <= 1) return n;
  
  let prev = 0;
  let curr = 1;
  
  for (let i = 2; i <= n; i++) {
    const next = prev + curr;
    prev = curr;
    curr = next;
  }
  
  return curr;
}

// Performance comparison
console.time('Slow');
console.log(fibonacciSlow(35)); // Very slow
console.timeEnd('Slow');

console.time('Fast');
console.log(fibonacciFast(35)); // Much faster
console.timeEnd('Fast');
```

## Cache Invalidation

```typescript
interface CacheEntry<T> {
  value: T;
  timestamp: number;
  dependencies?: string[];
}

class SmartCache<K, V> {
  private cache = new Map<K, CacheEntry<V>>();
  private dependencyGraph = new Map<string, Set<K>>();

  set(key: K, value: V, dependencies?: string[]): void {
    this.cache.set(key, {
      value,
      timestamp: Date.now(),
      dependencies
    });

    // Register dependencies
    if (dependencies) {
      dependencies.forEach(dep => {
        if (!this.dependencyGraph.has(dep)) {
          this.dependencyGraph.set(dep, new Set());
        }
        this.dependencyGraph.get(dep)!.add(key);
      });
    }
  }

  get(key: K, maxAge: number = Infinity): V | undefined {
    const entry = this.cache.get(key);
    
    if (!entry) return undefined;

    const age = Date.now() - entry.timestamp;
    if (age > maxAge) {
      this.cache.delete(key);
      return undefined;
    }

    return entry.value;
  }

  invalidate(dependency: string): void {
    const affectedKeys = this.dependencyGraph.get(dependency);
    
    if (affectedKeys) {
      affectedKeys.forEach(key => this.cache.delete(key));
      this.dependencyGraph.delete(dependency);
    }
  }

  invalidateAll(): void {
    this.cache.clear();
    this.dependencyGraph.clear();
  }
}

// Usage
const cache = new SmartCache<string, any>();

// Cache with dependencies
cache.set('user:1', { name: 'John' }, ['users']);
cache.set('post:1', { title: 'Hello' }, ['users', 'posts']);

// Invalidate all user-related cache
cache.invalidate('users'); // Clears both entries
```

## Testing

```typescript
describe('Memoization', () => {
  it('should cache function results', () => {
    const fn = jest.fn((x: number) => x * 2);
    const memoized = memoize(fn);

    expect(memoized(2)).toBe(4);
    expect(memoized(2)).toBe(4);
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it('should handle different arguments', () => {
    const fn = jest.fn((x: number) => x * 2);
    const memoized = memoize(fn);

    memoized(2);
    memoized(3);
    memoized(2);

    expect(fn).toHaveBeenCalledTimes(2);
  });
});

describe('LRU Cache', () => {
  it('should evict least recently used', () => {
    const cache = new LRUCache<string, number>(2);

    cache.put('a', 1);
    cache.put('b', 2);
    cache.put('c', 3); // Should evict 'a'

    expect(cache.get('a')).toBeUndefined();
    expect(cache.get('b')).toBe(2);
    expect(cache.get('c')).toBe(3);
  });

  it('should update access order on get', () => {
    const cache = new LRUCache<string, number>(2);

    cache.put('a', 1);
    cache.put('b', 2);
    cache.get('a'); // Access 'a'
    cache.put('c', 3); // Should evict 'b', not 'a'

    expect(cache.get('a')).toBe(1);
    expect(cache.get('b')).toBeUndefined();
    expect(cache.get('c')).toBe(3);
  });
});
```

## Key Takeaways

1. **Memoization**: Cache function results based on input arguments to avoid recomputation.

2. **Key Generation**: Use JSON.stringify for simple cases, custom key generators for complex objects.

3. **LRU Cache**: Evict least recently used items when cache reaches capacity.

4. **TTL (Time To Live)**: Expire cached values after specified time to prevent stale data.

5. **Cache Size Limits**: Set maximum cache size to prevent unbounded memory growth.

6. **Invalidation**: Implement cache invalidation strategies for dependent data.

7. **Map Data Structure**: JavaScript Map maintains insertion order, perfect for LRU implementation.

8. **Performance Trade-offs**: Memoization trades memory for speed - only use for expensive operations.

9. **React useMemo**: Memoizes values between renders, only recomputes when dependencies change.

10. **Dependency Tracking**: Track cache dependencies for intelligent invalidation.

## Interview Talking Points

- Explain memoization concept and benefits
- Describe LRU cache algorithm and implementation
- Discuss cache invalidation strategies
- Compare different key generation approaches
- Explain time vs space complexity trade-offs
- Describe when memoization is beneficial
- Discuss cache size and eviction policies
- Explain TTL and staleness considerations
- Compare memoization with other optimization techniques
- Describe React.memo vs useMemo vs useCallback
