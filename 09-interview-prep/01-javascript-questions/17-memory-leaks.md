# Memory Leaks - JavaScript Interview Questions

## The Idea

**In plain English:** A memory leak happens when a program uses up computer memory but then forgets to give it back when it is done, kind of like borrowing books from a library and never returning them. Over time, all the memory gets used up and the app slows down or crashes.

**Real-world analogy:** Imagine a restaurant where customers sit at tables, finish eating, and leave — but the waiter never clears or frees up those tables for new customers. Eventually every table is "claimed" by a ghost customer and no one new can be seated.

- The tables = memory (space set aside to hold data)
- The ghost customers (gone but tables still marked occupied) = objects that are no longer needed but still referenced by the program
- The waiter failing to clear tables = the garbage collector being blocked from freeing memory because the code still holds a reference to the old data

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Common Memory Leak Patterns](#common-memory-leak-patterns)
- [Detection and Prevention](#detection-and-prevention)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Key Takeaways](#key-takeaways)

## Core Concepts

### What is a Memory Leak?

Memory that's no longer needed but isn't released, gradually consuming available memory until the application slows or crashes.

### JavaScript Garbage Collection

JavaScript uses automatic garbage collection (mark-and-sweep algorithm). Objects are collected when they're no longer reachable from root references.

## Common Interview Questions

### Question 1: What Causes Memory Leaks in JavaScript?

**Expected Answer:**

```javascript
// 1. Global variables (never collected)
function leak() {
  globalVar = 'Memory leak'; // Creates global
}

// Fix: Use strict mode and proper declarations
'use strict';
function noLeak() {
  const localVar = 'No leak';
}

// 2. Forgotten timers
function startTimer() {
  setInterval(() => {
    console.log('Running...');
  }, 1000);
  // Never cleared - leaks forever
}

// Fix: Clear timers
function properTimer() {
  const timerId = setInterval(() => {
    console.log('Running...');
  }, 1000);
  
  return () => clearInterval(timerId);
}

// 3. Event listeners not removed
function addListeners() {
  const button = document.getElementById('btn');
  button.addEventListener('click', handleClick);
  // Never removed
}

// Fix: Remove listeners
function properListeners() {
  const button = document.getElementById('btn');
  const handler = () => console.log('Clicked');
  button.addEventListener('click', handler);
  
  return () => button.removeEventListener('click', handler);
}

// 4. Closures holding references
function createClosure() {
  const largeData = new Array(1000000).fill('data');
  
  return function() {
    console.log(largeData[0]); // Holds entire array
  };
}

// Fix: Extract only what's needed
function optimizedClosure() {
  const largeData = new Array(1000000).fill('data');
  const firstItem = largeData[0];
  
  return function() {
    console.log(firstItem); // Only holds one item
  };
}

// 5. Detached DOM nodes
let detachedTree;

function createDOMTree() {
  const div = document.createElement('div');
  div.innerHTML = '<div><div>Content</div></div>';
  document.body.appendChild(div);
  
  detachedTree = div; // Keeps reference
  document.body.removeChild(div); // Removed from DOM but still in memory
}

// Fix: Clear references
function properDOMHandling() {
  const div = document.createElement('div');
  document.body.appendChild(div);
  document.body.removeChild(div);
  // div goes out of scope and can be collected
}
```

### Question 2: Event Listener Memory Leaks

**Expected Answer:**

```javascript
// Problem: Event listener keeps component in memory
class Component {
  constructor() {
    this.data = new Array(100000).fill('data');
    
    // Event listener prevents garbage collection
    document.addEventListener('click', this.handleClick.bind(this));
  }
  
  handleClick() {
    console.log(this.data);
  }
  
  // destroy() never called or listeners not removed
}

// Solution 1: Remove listeners
class PropertyComponent {
  constructor() {
    this.data = new Array(100000).fill('data');
    this.boundHandler = this.handleClick.bind(this);
    document.addEventListener('click', this.boundHandler);
  }
  
  handleClick() {
    console.log(this.data);
  }
  
  destroy() {
    document.removeEventListener('click', this.boundHandler);
    this.boundHandler = null;
    this.data = null;
  }
}

// Solution 2: AbortController (modern)
class ModernComponent {
  constructor() {
    this.data = new Array(100000).fill('data');
    this.controller = new AbortController();
    
    document.addEventListener('click', this.handleClick.bind(this), {
      signal: this.controller.signal
    });
  }
  
  handleClick() {
    console.log(this.data);
  }
  
  destroy() {
    this.controller.abort(); // Removes all listeners
  }
}

// Solution 3: WeakMap for private data
const privateData = new WeakMap();

class WeakMapComponent {
  constructor() {
    privateData.set(this, {
      data: new Array(100000).fill('data')
    });
  }
  
  // When component is collected, WeakMap entry is too
}
```

### Question 3: Closure Memory Leaks

**Expected Answer:**

```javascript
// Problem: Closure holds unnecessary references
function createProcessor() {
  const hugeArray = new Array(1000000).fill('data');
  const metadata = { created: Date.now() };
  
  function process() {
    console.log(metadata.created);
    // Holds reference to entire hugeArray even though not used
  }
  
  return process;
}

// Fix: Clear unused references
function optimizedProcessor() {
  const hugeArray = new Array(1000000).fill('data');
  const created = Date.now(); // Extract what's needed
  
  // hugeArray can be garbage collected
  
  function process() {
    console.log(created);
  }
  
  return process;
}

// Problem: Circular references in closures
function createCircular() {
  const obj1 = {};
  const obj2 = {};
  
  obj1.ref = obj2;
  obj2.ref = obj1; // Circular reference
  
  return function() {
    console.log(obj1, obj2); // Keeps circular structure in memory
  };
}

// Fix: Break cycles when done
function properCircular() {
  let obj1 = {};
  let obj2 = {};
  
  obj1.ref = obj2;
  obj2.ref = obj1;
  
  return {
    use() {
      console.log(obj1, obj2);
    },
    cleanup() {
      obj1.ref = null;
      obj2.ref = null;
      obj1 = null;
      obj2 = null;
    }
  };
}
```

### Question 4: Timer and Interval Leaks

**Expected Answer:**

```javascript
// Problem: Intervals never cleared
class PollingService {
  constructor() {
    this.data = [];
    
    // Interval runs forever
    setInterval(() => {
      this.data.push(Date.now());
    }, 1000);
  }
}

// Fix: Store and clear timers
class ProperPollingService {
  constructor() {
    this.data = [];
    this.intervalId = null;
  }
  
  start() {
    this.intervalId = setInterval(() => {
      this.data.push(Date.now());
    }, 1000);
  }
  
  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
  
  destroy() {
    this.stop();
    this.data = null;
  }
}

// Problem: Forgotten timeouts
function deferredAction() {
  const data = new Array(100000);
  
  setTimeout(() => {
    console.log(data.length);
  }, 3600000); // 1 hour - data held in memory
}

// Fix: Cancel if no longer needed
function cancellableAction() {
  const data = new Array(100000);
  
  const timeoutId = setTimeout(() => {
    console.log(data.length);
  }, 3600000);
  
  return () => {
    clearTimeout(timeoutId);
  };
}

// Reactive cleanup example
class Timer {
  constructor() {
    this.timeouts = new Set();
    this.intervals = new Set();
  }
  
  setTimeout(callback, delay) {
    const id = setTimeout(() => {
      this.timeouts.delete(id);
      callback();
    }, delay);
    this.timeouts.add(id);
    return id;
  }
  
  setInterval(callback, delay) {
    const id = setInterval(callback, delay);
    this.intervals.add(id);
    return id;
  }
  
  clearAll() {
    this.timeouts.forEach(id => clearTimeout(id));
    this.intervals.forEach(id => clearInterval(id));
    this.timeouts.clear();
    this.intervals.clear();
  }
}
```

## Common Memory Leak Patterns

### Pattern 1: Cache Without Limit

```javascript
// Problem: Unbounded cache
class Cache {
  constructor() {
    this.cache = {};
  }
  
  set(key, value) {
    this.cache[key] = value; // Grows forever
  }
  
  get(key) {
    return this.cache[key];
  }
}

// Fix: LRU cache with size limit
class LRUCache {
  constructor(maxSize = 100) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }
  
  set(key, value) {
    // Remove oldest if at capacity
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    // Delete and re-add to move to end
    this.cache.delete(key);
    this.cache.set(key, value);
  }
  
  get(key) {
    if (!this.cache.has(key)) return undefined;
    
    // Move to end (most recently used)
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value);
    
    return value;
  }
}

// Modern: Use WeakMap for automatic cleanup
const cache = new WeakMap();

function getCached(obj) {
  if (cache.has(obj)) {
    return cache.get(obj);
  }
  
  const value = expensiveComputation(obj);
  cache.set(obj, value);
  return value;
}
// When obj is garbage collected, cache entry is too
```

### Pattern 2: Observable/Subscription Leaks

```javascript
// Problem: Subscriptions not cleaned up
class DataService {
  constructor() {
    this.listeners = [];
  }
  
  subscribe(listener) {
    this.listeners.push(listener);
    // No unsubscribe mechanism
  }
  
  notify(data) {
    this.listeners.forEach(listener => listener(data));
  }
}

// Fix: Return unsubscribe function
class ProperDataService {
  constructor() {
    this.listeners = new Set();
  }
  
  subscribe(listener) {
    this.listeners.add(listener);
    
    // Return unsubscribe function
    return () => {
      this.listeners.delete(listener);
    };
  }
  
  notify(data) {
    this.listeners.forEach(listener => listener(data));
  }
}

// Usage
const service = new ProperDataService();
const unsubscribe = service.subscribe(data => console.log(data));

// Clean up when done
unsubscribe();
```

### Pattern 3: Detached DOM Nodes

```javascript
// Problem: Keeping references to removed nodes
const deletedNodes = [];

function removeNode(node) {
  node.parentNode.removeChild(node);
  deletedNodes.push(node); // Prevents garbage collection
}

// Fix: Don't store references to removed nodes
function properRemoveNode(node) {
  node.parentNode.removeChild(node);
  // Let it be garbage collected
}

// Problem: Event listeners on detached nodes
const buttonHandler = () => console.log('Clicked');
const button = document.getElementById('myButton');
button.addEventListener('click', buttonHandler);

// Later: Remove button but listener still holds reference
button.parentNode.removeChild(button);
// button and handler still in memory

// Fix: Remove listeners before removing from DOM
button.removeEventListener('click', buttonHandler);
button.parentNode.removeChild(button);
```

## Detection and Prevention

### Using Chrome DevTools

```javascript
// Take heap snapshots
// 1. Open DevTools > Memory
// 2. Take snapshot before action
// 3. Perform action
// 4. Take snapshot after
// 5. Compare snapshots

// Look for:
// - Growing heap size
// - Objects not being collected
// - Detached DOM nodes
// - Unexpected retainers

// Performance monitoring
class MemoryMonitor {
  constructor() {
    this.baseline = performance.memory?.usedJSHeapSize;
  }
  
  check() {
    if (!performance.memory) return;
    
    const current = performance.memory.usedJSHeapSize;
    const diff = current - this.baseline;
    
    console.log(`Memory change: ${(diff / 1024 / 1024).toFixed(2)} MB`);
  }
}
```

### Prevention Strategies

```javascript
// 1. Use WeakMap/WeakSet for objects
const objectMetadata = new WeakMap();

// 2. Clean up in lifecycle hooks (React example)
useEffect(() => {
  const subscription = subscribe();
  
  return () => {
    subscription.unsubscribe(); // Cleanup
  };
}, []);

// 3. Nullify references when done
function cleanup() {
  this.data = null;
  this.listeners = null;
  this.element = null;
}

// 4. Use AbortController for fetch/events
const controller = new AbortController();

fetch(url, { signal: controller.signal });

// Cancel when needed
controller.abort();

// 5. Limit cache sizes
class BoundedCache {
  constructor(maxSize = 1000) {
    this.maxSize = maxSize;
    this.cache = new Map();
  }
  
  set(key, value) {
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
}
```

## What Interviewers Look For

### 1. Awareness of Common Leaks
- Global variables
- Forgotten timers/intervals
- Event listeners
- Closures
- Detached DOM nodes

### 2. Detection Skills
- Using DevTools
- Heap snapshots
- Performance monitoring
- Memory profiling

### 3. Prevention Strategies
- Cleanup patterns
- WeakMap/WeakSet usage
- Proper lifecycle management
- AbortController
- Cache limits

### 4. Framework Knowledge
- React useEffect cleanup
- Angular OnDestroy
- Vue beforeDestroy
- Component lifecycle

### 5. Real-World Experience
- Has debugged memory leaks
- Knows profiling tools
- Understands performance impact
- Can prevent common patterns

## Key Takeaways

### Common Causes
1. **Global variables**: Never garbage collected
2. **Forgotten timers**: setInterval/setTimeout not cleared
3. **Event listeners**: Not removed when done
4. **Closures**: Holding unnecessary references
5. **Detached DOM**: Removed from DOM but referenced

### Prevention Best Practices
1. **Always clean up**: Remove listeners, clear timers
2. **Use WeakMap/WeakSet**: For object-keyed data
3. **Limit cache sizes**: Implement LRU or size limits
4. **Framework lifecycle**: Use cleanup hooks properly
5. **AbortController**: For cancellable operations

### Detection Methods
1. **Chrome DevTools**: Heap snapshots, comparison
2. **Performance monitoring**: Track memory usage
3. **Allocation timeline**: See what's being created
4. **Detached nodes**: Find removed but referenced DOM
5. **Profiling**: Regular memory audits

### Interview Success Tips
1. **Explain common leaks**: With code examples
2. **Show detection**: DevTools proficiency
3. **Demonstrate prevention**: Cleanup patterns
4. **Framework awareness**: Lifecycle hooks
5. **Real experience**: Leaks you've fixed
6. **Performance impact**: Why it matters

Remember: Memory leaks in JavaScript are usually about retaining references to objects that are no longer needed. The key is proper cleanup: remove event listeners, clear timers, break circular references, and use WeakMap/WeakSet for automatic garbage collection of object-keyed data.
