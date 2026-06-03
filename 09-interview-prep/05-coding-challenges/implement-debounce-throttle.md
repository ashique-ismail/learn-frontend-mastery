# Implement Debounce and Throttle

## Overview

Implement debounce and throttle functions from scratch, understanding their differences, use cases, and implementation patterns.

## Debounce vs Throttle

**Debounce**: Delays function execution until after a specified time has elapsed since the last invocation.

**Throttle**: Ensures function executes at most once in a specified time period.

## Implementation

### 1. Basic Debounce

```typescript
function debounce<T extends (...args: any[]) => any>(
  func: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: number | undefined;

  return function(this: any, ...args: Parameters<T>) {
    // Clear existing timeout
    if (timeoutId !== undefined) {
      clearTimeout(timeoutId);
    }

    // Set new timeout
    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// Usage
const search = debounce((query: string) => {
  console.log('Searching for:', query);
  // API call here
}, 300);

search('a');    // Won't execute
search('an');   // Won't execute
search('ang');  // Executes after 300ms
```

### 2. Debounce with Leading Edge

```typescript
function debounce<T extends (...args: any[]) => any>(
  func: T,
  delay: number,
  options: { leading?: boolean; trailing?: boolean } = {}
): (...args: Parameters<T>) => void {
  let timeoutId: number | undefined;
  let lastArgs: Parameters<T> | undefined;
  let lastThis: any;

  const { leading = false, trailing = true } = options;

  return function(this: any, ...args: Parameters<T>) {
    lastArgs = args;
    lastThis = this;

    const shouldCallNow = leading && timeoutId === undefined;

    if (timeoutId !== undefined) {
      clearTimeout(timeoutId);
    }

    if (shouldCallNow) {
      func.apply(lastThis, lastArgs);
    }

    timeoutId = setTimeout(() => {
      if (trailing && lastArgs) {
        func.apply(lastThis, lastArgs);
      }
      timeoutId = undefined;
      lastArgs = undefined;
    }, delay);
  };
}

// Usage: Execute immediately, then wait
const handleInput = debounce((value: string) => {
  console.log('Input:', value);
}, 300, { leading: true, trailing: false });
```

### 3. Debounce with Cancel

```typescript
interface DebouncedFunction<T extends (...args: any[]) => any> {
  (...args: Parameters<T>): void;
  cancel(): void;
  flush(): void;
}

function debounce<T extends (...args: any[]) => any>(
  func: T,
  delay: number
): DebouncedFunction<T> {
  let timeoutId: number | undefined;
  let lastArgs: Parameters<T> | undefined;
  let lastThis: any;

  const debounced = function(this: any, ...args: Parameters<T>) {
    lastArgs = args;
    lastThis = this;

    if (timeoutId !== undefined) {
      clearTimeout(timeoutId);
    }

    timeoutId = setTimeout(() => {
      if (lastArgs) {
        func.apply(lastThis, lastArgs);
      }
      timeoutId = undefined;
      lastArgs = undefined;
    }, delay);
  };

  debounced.cancel = () => {
    if (timeoutId !== undefined) {
      clearTimeout(timeoutId);
      timeoutId = undefined;
      lastArgs = undefined;
    }
  };

  debounced.flush = () => {
    if (timeoutId !== undefined && lastArgs) {
      clearTimeout(timeoutId);
      func.apply(lastThis, lastArgs);
      timeoutId = undefined;
      lastArgs = undefined;
    }
  };

  return debounced;
}

// Usage
const saveData = debounce((data: any) => {
  console.log('Saving:', data);
}, 1000);

saveData({ name: 'John' });
saveData.cancel(); // Cancel pending save
saveData.flush();  // Execute immediately
```

### 4. Basic Throttle

```typescript
function throttle<T extends (...args: any[]) => any>(
  func: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;
  let lastResult: ReturnType<T>;

  return function(this: any, ...args: Parameters<T>) {
    if (!inThrottle) {
      inThrottle = true;
      lastResult = func.apply(this, args);

      setTimeout(() => {
        inThrottle = false;
      }, limit);
    }

    return lastResult;
  };
}

// Usage: Execute at most once per 1000ms
const handleScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY);
}, 1000);

window.addEventListener('scroll', handleScroll);
```

### 5. Throttle with Leading and Trailing

```typescript
function throttle<T extends (...args: any[]) => any>(
  func: T,
  limit: number,
  options: { leading?: boolean; trailing?: boolean } = {}
): (...args: Parameters<T>) => void {
  let inThrottle = false;
  let lastArgs: Parameters<T> | undefined;
  let lastThis: any;
  let result: ReturnType<T>;

  const { leading = true, trailing = true } = options;

  return function(this: any, ...args: Parameters<T>) {
    lastArgs = args;
    lastThis = this;

    if (!inThrottle) {
      if (leading) {
        result = func.apply(lastThis, lastArgs);
      }

      inThrottle = true;

      setTimeout(() => {
        if (trailing && lastArgs) {
          result = func.apply(lastThis, lastArgs);
        }
        inThrottle = false;
        lastArgs = undefined;
      }, limit);
    }

    return result;
  };
}

// Usage
const trackMouseMove = throttle((event: MouseEvent) => {
  console.log('Mouse position:', event.clientX, event.clientY);
}, 100, { leading: true, trailing: true });
```

### 6. Advanced: Throttle with Cancel and Flush

```typescript
interface ThrottledFunction<T extends (...args: any[]) => any> {
  (...args: Parameters<T>): void;
  cancel(): void;
  flush(): void;
}

function throttle<T extends (...args: any[]) => any>(
  func: T,
  limit: number
): ThrottledFunction<T> {
  let inThrottle = false;
  let lastArgs: Parameters<T> | undefined;
  let lastThis: any;
  let timeoutId: number | undefined;

  const throttled = function(this: any, ...args: Parameters<T>) {
    lastArgs = args;
    lastThis = this;

    if (!inThrottle) {
      func.apply(lastThis, lastArgs);
      lastArgs = undefined;
      inThrottle = true;

      timeoutId = setTimeout(() => {
        inThrottle = false;
        
        if (lastArgs) {
          throttled.apply(lastThis, lastArgs);
        }
      }, limit);
    }
  };

  throttled.cancel = () => {
    if (timeoutId !== undefined) {
      clearTimeout(timeoutId);
      timeoutId = undefined;
    }
    inThrottle = false;
    lastArgs = undefined;
  };

  throttled.flush = () => {
    if (lastArgs) {
      func.apply(lastThis, lastArgs);
      throttled.cancel();
    }
  };

  return throttled;
}
```

## Use Cases

### Debounce Use Cases

```typescript
// 1. Search input
const searchAPI = debounce((query: string) => {
  fetch(`/api/search?q=${query}`)
    .then(res => res.json())
    .then(data => console.log(data));
}, 300);

document.querySelector('input')?.addEventListener('input', (e) => {
  searchAPI((e.target as HTMLInputElement).value);
});

// 2. Window resize
const handleResize = debounce(() => {
  console.log('Window resized:', window.innerWidth, window.innerHeight);
}, 250);

window.addEventListener('resize', handleResize);

// 3. Auto-save
const autoSave = debounce((data: any) => {
  localStorage.setItem('draft', JSON.stringify(data));
  console.log('Draft saved');
}, 1000);

document.querySelector('textarea')?.addEventListener('input', (e) => {
  autoSave({ content: (e.target as HTMLTextAreaElement).value });
});

// 4. Form validation
const validateField = debounce(async (value: string) => {
  const isValid = await checkUsername(value);
  console.log('Validation result:', isValid);
}, 500);
```

### Throttle Use Cases

```typescript
// 1. Scroll event
const handleScroll = throttle(() => {
  const scrollPercent = (window.scrollY / document.body.scrollHeight) * 100;
  console.log('Scroll progress:', scrollPercent);
}, 100);

window.addEventListener('scroll', handleScroll);

// 2. Mouse move tracking
const trackMouse = throttle((event: MouseEvent) => {
  sendAnalytics('mouse_move', {
    x: event.clientX,
    y: event.clientY
  });
}, 200);

document.addEventListener('mousemove', trackMouse);

// 3. Button click (prevent spam)
const submitForm = throttle(() => {
  console.log('Form submitted');
  // API call
}, 2000, { leading: true, trailing: false });

document.querySelector('button')?.addEventListener('click', submitForm);

// 4. API rate limiting
const fetchData = throttle(() => {
  fetch('/api/data')
    .then(res => res.json())
    .then(data => console.log(data));
}, 1000);

// Can only call once per second
setInterval(fetchData, 100); // Will throttle to once per second
```

## Testing

```typescript
describe('Debounce', () => {
  jest.useFakeTimers();

  it('should delay function execution', () => {
    const func = jest.fn();
    const debounced = debounce(func, 100);

    debounced();
    expect(func).not.toHaveBeenCalled();

    jest.advanceTimersByTime(100);
    expect(func).toHaveBeenCalledTimes(1);
  });

  it('should reset timer on multiple calls', () => {
    const func = jest.fn();
    const debounced = debounce(func, 100);

    debounced();
    jest.advanceTimersByTime(50);
    debounced();
    jest.advanceTimersByTime(50);
    
    expect(func).not.toHaveBeenCalled();

    jest.advanceTimersByTime(50);
    expect(func).toHaveBeenCalledTimes(1);
  });

  it('should cancel pending execution', () => {
    const func = jest.fn();
    const debounced = debounce(func, 100);

    debounced();
    debounced.cancel();

    jest.advanceTimersByTime(100);
    expect(func).not.toHaveBeenCalled();
  });
});

describe('Throttle', () => {
  jest.useFakeTimers();

  it('should execute immediately', () => {
    const func = jest.fn();
    const throttled = throttle(func, 100);

    throttled();
    expect(func).toHaveBeenCalledTimes(1);
  });

  it('should throttle subsequent calls', () => {
    const func = jest.fn();
    const throttled = throttle(func, 100);

    throttled();
    throttled();
    throttled();

    expect(func).toHaveBeenCalledTimes(1);

    jest.advanceTimersByTime(100);
    expect(func).toHaveBeenCalledTimes(1);
  });

  it('should execute trailing call after cooldown', () => {
    const func = jest.fn();
    const throttled = throttle(func, 100, { trailing: true });

    throttled();
    throttled();

    jest.advanceTimersByTime(100);
    expect(func).toHaveBeenCalledTimes(2);
  });
});
```

## Key Takeaways

1. **Debounce**: Waits for pause in activity before executing (search input, auto-save).

2. **Throttle**: Executes at regular intervals during continuous activity (scroll, mouse move).

3. **Leading Edge**: Execute immediately on first call, then wait.

4. **Trailing Edge**: Execute after waiting period ends.

5. **Cancel Method**: Ability to cancel pending execution.

6. **Flush Method**: Execute pending call immediately.

7. **Context Preservation**: Maintain `this` context with `apply`.

8. **Type Safety**: Use TypeScript generics for type-safe implementations.

9. **Memory Management**: Clear timeouts and references to prevent leaks.

10. **Testing**: Use fake timers for deterministic testing.

## Interview Talking Points

- Explain difference between debounce and throttle
- Discuss real-world use cases for each
- Describe leading vs trailing edge options
- Explain how to implement cancel/flush methods
- Discuss TypeScript typing strategies
- Explain context preservation with apply/call
- Describe testing approach with fake timers
- Discuss performance implications
- Explain memory management considerations
- Compare with RxJS operators (debounceTime, throttleTime)
