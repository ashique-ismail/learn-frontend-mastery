# Whiteboard: Live Coding Patterns

## Overview

Live coding sessions test algorithmic thinking, code quality, and communication under pressure. This guide covers the patterns interviewers reach for most often in frontend roles — data structures, DOM manipulation, async patterns, and functional utilities — with worked examples and a protocol for structuring your approach out loud.

---

## The Live Coding Protocol

**Before touching the keyboard:**

1. **Restate the problem** in your own words and confirm you have it right.
2. **Clarify constraints** — input types, edge cases, performance requirements.
3. **Name your examples** — write out 2-3 input/output pairs, including an edge case.
4. **Talk through your approach** before writing — "I'll iterate once over the array, storing counts in a Map."
5. **Write, narrate, test** — code while explaining, then trace through your examples.

Interviewers care as much about your thinking process as your solution.

---

## Pattern 1: Implement a Utility Function

### Debounce

```javascript
// "Implement debounce(fn, delay) — fn should fire only after
//  delay ms of silence. Support immediate execution as an option."

function debounce(fn, delay, immediate = false) {
  let timerId = null;

  return function (...args) {
    const callNow = immediate && timerId === null;

    clearTimeout(timerId);
    timerId = setTimeout(() => {
      timerId = null;
      if (!immediate) fn.apply(this, args);
    }, delay);

    if (callNow) fn.apply(this, args);
  };
}

// Test
const log = debounce((x) => console.log(x), 300);
log('a'); log('b'); log('c'); // only 'c' fires after 300ms
```

### Deep Clone

```javascript
// "Clone a value deeply — handle primitives, arrays, objects,
//  Dates, and circular references."

function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value)) return seen.get(value); // circular reference guard

  if (value instanceof Date) return new Date(value);
  if (Array.isArray(value)) {
    const clone = [];
    seen.set(value, clone);
    for (const item of value) clone.push(deepClone(item, seen));
    return clone;
  }

  const clone = Object.create(Object.getPrototypeOf(value));
  seen.set(value, clone);
  for (const key of Object.keys(value)) {
    clone[key] = deepClone(value[key], seen);
  }
  return clone;
}
```

### Promise.all polyfill

```javascript
// "Implement Promise.all — resolve when all settle, reject on first failure."

function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!promises.length) return resolve([]);

    const results = new Array(promises.length);
    let remaining = promises.length;

    promises.forEach((p, i) => {
      Promise.resolve(p).then((value) => {
        results[i] = value;
        if (--remaining === 0) resolve(results);
      }, reject); // first rejection short-circuits
    });
  });
}
```

---

## Pattern 2: Array / Object Manipulation

### Flatten nested array to given depth

```javascript
function flatten(arr, depth = Infinity) {
  if (depth === 0) return [...arr];
  return arr.reduce((acc, item) => {
    if (Array.isArray(item) && depth > 0) {
      acc.push(...flatten(item, depth - 1));
    } else {
      acc.push(item);
    }
    return acc;
  }, []);
}

flatten([1, [2, [3, [4]]]], 2); // [1, 2, 3, [4]]
```

### Group array of objects by key

```javascript
function groupBy(arr, keyFn) {
  return arr.reduce((map, item) => {
    const key = typeof keyFn === 'function' ? keyFn(item) : item[keyFn];
    (map[key] ??= []).push(item);
    return map;
  }, {});
}

groupBy([
  { name: 'Alice', dept: 'eng' },
  { name: 'Bob',   dept: 'design' },
  { name: 'Carol', dept: 'eng' },
], 'dept');
// { eng: [Alice, Carol], design: [Bob] }
```

### LRU Cache

```javascript
class LRUCache {
  #capacity;
  #map = new Map(); // insertion order = LRU order

  constructor(capacity) {
    this.#capacity = capacity;
  }

  get(key) {
    if (!this.#map.has(key)) return -1;
    const value = this.#map.get(key);
    this.#map.delete(key);
    this.#map.set(key, value); // move to end (most recently used)
    return value;
  }

  put(key, value) {
    this.#map.delete(key); // ensure fresh insertion position
    this.#map.set(key, value);
    if (this.#map.size > this.#capacity) {
      // Map.keys() iterates in insertion order — first key = LRU
      this.#map.delete(this.#map.keys().next().value);
    }
  }
}
```

---

## Pattern 3: DOM and Event Handling

### Event delegation

```javascript
// "Add one listener to a list and handle clicks on any <li>."

document.querySelector('#list').addEventListener('click', (e) => {
  const li = e.target.closest('li[data-id]');
  if (!li) return;
  console.log('Clicked item:', li.dataset.id);
});
```

### Drag-and-drop reorder

```javascript
class DraggableList {
  constructor(container) {
    this.container = container;
    this.dragSrc = null;
    container.addEventListener('dragstart', this.#onStart.bind(this));
    container.addEventListener('dragover',  this.#onOver.bind(this));
    container.addEventListener('drop',      this.#onDrop.bind(this));
  }

  #onStart(e) {
    this.dragSrc = e.target.closest('[draggable]');
    e.dataTransfer.effectAllowed = 'move';
  }

  #onOver(e) {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
    const target = e.target.closest('[draggable]');
    if (target && target !== this.dragSrc) {
      const rect = target.getBoundingClientRect();
      const after = e.clientY > rect.top + rect.height / 2;
      target.parentNode.insertBefore(
        this.dragSrc,
        after ? target.nextSibling : target
      );
    }
  }

  #onDrop(e) {
    e.stopPropagation();
    this.dragSrc = null;
  }
}
```

### Infinite scroll with IntersectionObserver

```javascript
function createInfiniteScroll(sentinel, onLoadMore) {
  const observer = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting) onLoadMore();
    },
    { rootMargin: '200px' }
  );
  observer.observe(sentinel);
  return () => observer.disconnect(); // cleanup
}
```

---

## Pattern 4: Async Patterns

### Retry with exponential backoff

```javascript
async function fetchWithRetry(url, { retries = 3, baseDelay = 500 } = {}) {
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return await res.json();
    } catch (err) {
      if (attempt === retries) throw err;
      const delay = baseDelay * 2 ** attempt + Math.random() * 100;
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}
```

### Rate limiter (max N concurrent)

```javascript
function createLimiter(maxConcurrent) {
  let running = 0;
  const queue = [];

  function run(task) {
    running++;
    task().finally(() => {
      running--;
      if (queue.length) run(queue.shift());
    });
  }

  return (task) =>
    new Promise((resolve, reject) => {
      const wrapped = () =>
        Promise.resolve().then(task).then(resolve).catch(reject);
      if (running < maxConcurrent) {
        run(wrapped);
      } else {
        queue.push(wrapped);
      }
    });
}

const limit = createLimiter(3);
const results = await Promise.all(urls.map((url) => limit(() => fetch(url))));
```

---

## Pattern 5: React / Angular Component Snippets

### Custom hook: useLocalStorage

```typescript
import { useState, useEffect } from 'react';

function useLocalStorage<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? (JSON.parse(stored) as T) : defaultValue;
    } catch {
      return defaultValue;
    }
  });

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch { /* quota exceeded — ignore */ }
  }, [key, value]);

  return [value, setValue] as const;
}
```

### Angular signal-based store

```typescript
import { Injectable, computed, signal } from '@angular/core';

interface State<T> {
  items: T[];
  loading: boolean;
  error: string | null;
}

@Injectable({ providedIn: 'root' })
export class ItemStore<T extends { id: string | number }> {
  private _state = signal<State<T>>({ items: [], loading: false, error: null });

  readonly items    = computed(() => this._state().items);
  readonly loading  = computed(() => this._state().loading);
  readonly error    = computed(() => this._state().error);
  readonly count    = computed(() => this._state().items.length);

  setLoading(loading: boolean): void {
    this._state.update((s) => ({ ...s, loading }));
  }

  setItems(items: T[]): void {
    this._state.update((s) => ({ ...s, items, loading: false, error: null }));
  }

  upsert(item: T): void {
    this._state.update((s) => {
      const idx = s.items.findIndex((i) => i.id === item.id);
      const items =
        idx >= 0
          ? s.items.map((i, j) => (j === idx ? item : i))
          : [...s.items, item];
      return { ...s, items };
    });
  }

  remove(id: T['id']): void {
    this._state.update((s) => ({
      ...s,
      items: s.items.filter((i) => i.id !== id),
    }));
  }
}
```

---

## Live Coding Tips

**Talk through complexity** — after your solution, state the Big-O time and space. "This is O(n) time and O(n) space because of the Map."

**Edge cases to always mention:**
- Empty input (`[]`, `''`, `null`, `undefined`)
- Single-element input
- Duplicate values
- Deeply nested structures
- Circular references (for clone/traverse)

**When stuck:**
- Solve a smaller version first ("let's ignore circular refs for now")
- Describe what you'd do even if you can't code it
- Ask if a built-in helper is in scope ("can I use `Array.flat`?")

**Mistakes are fine** — interviewers care more that you catch and fix them than that you write perfect code on the first pass.

---

## Common Gotchas

| Gotcha | What goes wrong | Fix |
|---|---|---|
| `clearTimeout` without storing the id | Timer never clears | Always store the return value |
| Mutating array inside `reduce` | Subtle aliasing bugs | Spread or concat to create new array |
| `typeof null === 'object'` | Null treated as cloneable object | Guard with `value === null` first |
| Missing `Promise.resolve(p)` wrapper in `promiseAll` | Non-Promise values crash | Always wrap with `Promise.resolve` |
| Arrow function `this` in class event handlers | `this` is undefined | Use `bind` or class method fields |
