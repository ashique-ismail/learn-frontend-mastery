# useRef Hook - Mutable References and DOM Access

## The Idea

**In plain English:** `useRef` is a way to hold onto a value (or a piece of the page) in your React component that sticks around between updates — but changing it does NOT cause the page to refresh. Think of it as a sticky note attached to your component that you can read and write at any time without disturbing anything else.

**Real-world analogy:** Imagine a stage manager at a theatre who keeps a clipboard with notes (like "the timer started at 7:42 PM"). The actors on stage (your UI) perform their scenes without caring what the clipboard says. The stage manager can write new notes on the clipboard at any moment without stopping the show or making the actors restart their scene.

- The clipboard = the ref object (`useRef(...)`)
- The note written on it = the value stored in `.current`
- Writing a new note = updating `ref.current`
- The actors continuing their scene uninterrupted = the component not re-rendering

---

## Table of Contents

- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Core API](#core-api)
- [Common Use Cases](#common-use-cases)
- [Advanced Patterns](#advanced-patterns)
- [useRef vs useState](#useref-vs-usestate)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

`useRef` is a React Hook that returns a mutable ref object whose `.current` property persists across re-renders without causing re-renders when changed. It's primarily used for accessing DOM elements and storing mutable values.

### Two Main Use Cases

```javascript
// 1. Accessing DOM elements
function TextInput() {
  const inputRef = useRef(null);
  
  const focus = () => {
    inputRef.current.focus();
  };
  
  return (
    <div>
      <input ref={inputRef} />
      <button onClick={focus}>Focus Input</button>
    </div>
  );
}

// 2. Storing mutable values that don't trigger re-renders
function Timer() {
  const intervalRef = useRef(null);
  
  const startTimer = () => {
    intervalRef.current = setInterval(() => {
      console.log('Tick');
    }, 1000);
  };
  
  const stopTimer = () => {
    clearInterval(intervalRef.current);
  };
  
  return (
    <div>
      <button onClick={startTimer}>Start</button>
      <button onClick={stopTimer}>Stop</button>
    </div>
  );
}
```

### Visual Representation

```
useState:                          useRef:
┌──────────────┐                  ┌──────────────┐
│ Change value │                  │ Change value │
└──────┬───────┘                  └──────┬───────┘
       │                                 │
       ↓                                 ↓
┌──────────────┐                  ┌──────────────┐
│  Re-render!  │                  │ No re-render │
└──────────────┘                  └──────────────┘
```

## Basic Concepts

### Syntax

```javascript
const refContainer = useRef(initialValue);
```

**Parameters:**
- `initialValue`: Initial value for `ref.current`

**Returns:**
- Object with single property: `{ current: initialValue }`

### Key Characteristics

```javascript
function Component() {
  const ref = useRef(0);
  
  // ✅ Persists across renders
  console.log(ref.current); // 0
  
  // ✅ Mutable without re-rendering
  ref.current = ref.current + 1;
  console.log(ref.current); // 1
  // Component doesn't re-render!
  
  // ✅ Same object reference every render
  const sameRef = useRef(0);
  console.log(ref === sameRef); // false (different refs)
  
  // ⚠️ Changes don't trigger re-renders
  ref.current = 100; // No re-render
  
  return <div>{ref.current}</div>;
}
```

## Core API

### DOM References

```javascript
import { useRef, useEffect } from 'react';

function AutoFocusInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    // Access DOM element
    inputRef.current.focus();
  }, []);

  return <input ref={inputRef} type="text" />;
}

// Multiple refs
function Form() {
  const nameRef = useRef(null);
  const emailRef = useRef(null);
  const passwordRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    
    const formData = {
      name: nameRef.current.value,
      email: emailRef.current.value,
      password: passwordRef.current.value
    };
    
    console.log(formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} placeholder="Name" />
      <input ref={emailRef} type="email" placeholder="Email" />
      <input ref={passwordRef} type="password" placeholder="Password" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Storing Mutable Values

```javascript
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null);

  const startTimer = () => {
    if (intervalRef.current !== null) return; // Already running
    
    intervalRef.current = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
  };

  const stopTimer = () => {
    if (intervalRef.current === null) return;
    
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  const resetTimer = () => {
    stopTimer();
    setCount(0);
  };

  useEffect(() => {
    // Cleanup on unmount
    return () => {
      if (intervalRef.current !== null) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);

  return (
    <div>
      <p>Time: {count}s</p>
      <button onClick={startTimer}>Start</button>
      <button onClick={stopTimer}>Stop</button>
      <button onClick={resetTimer}>Reset</button>
    </div>
  );
}
```

### Callback Refs

```javascript
function MeasuredComponent() {
  const [height, setHeight] = useState(0);

  // Callback ref - called when element mounts/unmounts
  const measuredRef = useCallback(node => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  }, []);

  return (
    <div>
      <div ref={measuredRef}>
        <h1>Hello!</h1>
        <p>This is some content.</p>
      </div>
      <p>Height: {height}px</p>
    </div>
  );
}

// Dynamic refs with callback
function DynamicList({ items }) {
  const itemRefs = useRef({});

  const setRef = (id) => (element) => {
    if (element) {
      itemRefs.current[id] = element;
    } else {
      delete itemRefs.current[id];
    }
  };

  const scrollToItem = (id) => {
    itemRefs.current[id]?.scrollIntoView({ behavior: 'smooth' });
  };

  return (
    <div>
      {items.map(item => (
        <div key={item.id} ref={setRef(item.id)}>
          {item.name}
          <button onClick={() => scrollToItem(item.id)}>
            Scroll here
          </button>
        </div>
      ))}
    </div>
  );
}
```

## Common Use Cases

### 1. Focus Management

```javascript
function SearchInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    // Focus on mount
    inputRef.current.focus();
  }, []);

  const handleClear = () => {
    inputRef.current.value = '';
    inputRef.current.focus();
  };

  return (
    <div>
      <input 
        ref={inputRef}
        type="search"
        placeholder="Search..."
      />
      <button onClick={handleClear}>Clear</button>
    </div>
  );
}

// Focus trap for modals
function Modal({ isOpen, children }) {
  const firstFocusRef = useRef(null);
  const lastFocusRef = useRef(null);
  const previousFocusRef = useRef(null);

  useEffect(() => {
    if (isOpen) {
      // Store previously focused element
      previousFocusRef.current = document.activeElement;
      // Focus first element in modal
      firstFocusRef.current?.focus();
      
      return () => {
        // Restore focus on close
        previousFocusRef.current?.focus();
      };
    }
  }, [isOpen]);

  const handleKeyDown = (e) => {
    if (e.key === 'Tab') {
      if (e.shiftKey) {
        if (document.activeElement === firstFocusRef.current) {
          e.preventDefault();
          lastFocusRef.current?.focus();
        }
      } else {
        if (document.activeElement === lastFocusRef.current) {
          e.preventDefault();
          firstFocusRef.current?.focus();
        }
      }
    }
  };

  if (!isOpen) return null;

  return (
    <div className="modal" onKeyDown={handleKeyDown}>
      <button ref={firstFocusRef}>First</button>
      {children}
      <button ref={lastFocusRef}>Last</button>
    </div>
  );
}
```

### 2. Previous Value Tracking

```javascript
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

// Usage
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}

// Track multiple previous values
function useHistory(value, maxLength = 10) {
  const historyRef = useRef([]);

  useEffect(() => {
    historyRef.current = [value, ...historyRef.current].slice(0, maxLength);
  }, [value, maxLength]);

  return historyRef.current;
}
```

### 3. Animation and Requesters

```javascript
function AnimatedComponent() {
  const [position, setPosition] = useState(0);
  const rafRef = useRef(null);
  const startTimeRef = useRef(null);

  const animate = (timestamp) => {
    if (!startTimeRef.current) {
      startTimeRef.current = timestamp;
    }

    const elapsed = timestamp - startTimeRef.current;
    const progress = Math.min(elapsed / 1000, 1); // 1 second animation

    setPosition(progress * 100);

    if (progress < 1) {
      rafRef.current = requestAnimationFrame(animate);
    }
  };

  const startAnimation = () => {
    startTimeRef.current = null;
    rafRef.current = requestAnimationFrame(animate);
  };

  const stopAnimation = () => {
    if (rafRef.current) {
      cancelAnimationFrame(rafRef.current);
    }
  };

  useEffect(() => {
    return () => stopAnimation();
  }, []);

  return (
    <div>
      <div 
        style={{ 
          transform: `translateX(${position}px)`,
          width: 50,
          height: 50,
          background: 'blue'
        }}
      />
      <button onClick={startAnimation}>Start</button>
      <button onClick={stopAnimation}>Stop</button>
    </div>
  );
}
```

### 4. Instance Variables

```javascript
function DataFetcher({ url }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const controllerRef = useRef(null);

  useEffect(() => {
    const fetchData = async () => {
      // Cancel previous request
      controllerRef.current?.abort();
      
      // Create new controller
      controllerRef.current = new AbortController();
      
      setLoading(true);
      try {
        const response = await fetch(url, {
          signal: controllerRef.current.signal
        });
        const json = await response.json();
        setData(json);
      } catch (error) {
        if (error.name !== 'AbortError') {
          console.error('Fetch error:', error);
        }
      } finally {
        setLoading(false);
      }
    };

    fetchData();

    return () => {
      controllerRef.current?.abort();
    };
  }, [url]);

  return (
    <div>
      {loading && <p>Loading...</p>}
      {data && <pre>{JSON.stringify(data, null, 2)}</pre>}
    </div>
  );
}
```

## Advanced Patterns

### Forwarding Refs

```javascript
import { forwardRef } from 'react';

// Allow parent to access child's ref
const CustomInput = forwardRef((props, ref) => {
  return (
    <div className="custom-input">
      <input ref={ref} {...props} />
    </div>
  );
});

// Usage
function Parent() {
  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus();
  };

  return (
    <div>
      <CustomInput ref={inputRef} />
      <button onClick={focusInput}>Focus</button>
    </div>
  );
}

// Forwarding with useImperativeHandle
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    },
    clear: () => {
      inputRef.current.value = '';
    },
    getValue: () => {
      return inputRef.current.value;
    }
  }));

  return <input ref={inputRef} {...props} />;
});
```

### Ref Arrays for Lists

```javascript
function TodoList({ todos }) {
  const itemRefs = useRef([]);

  // Ensure array matches todos length
  useEffect(() => {
    itemRefs.current = itemRefs.current.slice(0, todos.length);
  }, [todos]);

  const setRef = (index) => (element) => {
    itemRefs.current[index] = element;
  };

  const focusItem = (index) => {
    itemRefs.current[index]?.focus();
  };

  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={todo.id}>
          <input
            ref={setRef(index)}
            defaultValue={todo.text}
          />
          <button onClick={() => focusItem(index)}>
            Focus
          </button>
        </li>
      ))}
    </ul>
  );
}

// Better: Using Map for dynamic items
function DynamicList({ items }) {
  const itemRefs = useRef(new Map());

  const setRef = (id) => (element) => {
    if (element) {
      itemRefs.current.set(id, element);
    } else {
      itemRefs.current.delete(id);
    }
  };

  const scrollToItem = (id) => {
    const element = itemRefs.current.get(id);
    element?.scrollIntoView({ behavior: 'smooth', block: 'center' });
  };

  return (
    <div>
      {items.map(item => (
        <div key={item.id} ref={setRef(item.id)}>
          {item.content}
        </div>
      ))}
      
      <div className="quick-nav">
        {items.map(item => (
          <button 
            key={item.id}
            onClick={() => scrollToItem(item.id)}
          >
            Go to {item.name}
          </button>
        ))}
      </div>
    </div>
  );
}
```

### Combining useRef with Other Hooks

```javascript
function useDebounce(callback, delay) {
  const timeoutRef = useRef(null);
  const callbackRef = useRef(callback);

  // Keep callback reference updated
  useEffect(() => {
    callbackRef.current = callback;
  }, [callback]);

  return useCallback((...args) => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }

    timeoutRef.current = setTimeout(() => {
      callbackRef.current(...args);
    }, delay);
  }, [delay]);
}

// Usage
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const searchAPI = async (searchTerm) => {
    const data = await fetch(`/api/search?q=${searchTerm}`);
    setResults(await data.json());
  };

  const debouncedSearch = useDebounce(searchAPI, 300);

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </div>
  );
}
```

## useRef vs useState

### When to Use Each

```javascript
// ✅ Use useState: Value affects rendering
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}

// ✅ Use useRef: Value doesn't affect rendering
function Timer() {
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  const toggle = () => {
    if (isRunning) {
      clearInterval(intervalRef.current);
    } else {
      intervalRef.current = setInterval(() => {
        console.log('Tick');
      }, 1000);
    }
    setIsRunning(!isRunning);
  };

  return <button onClick={toggle}>{isRunning ? 'Stop' : 'Start'}</button>;
}
```

### Comparison Table

| Feature | useState | useRef |
|---------|----------|--------|
| Triggers re-render | ✅ Yes | ❌ No |
| Persists across renders | ✅ Yes | ✅ Yes |
| Mutable | ❌ No (immutable updates) | ✅ Yes |
| Used for UI | ✅ Yes | ❌ Rarely |
| Used for DOM refs | ❌ No | ✅ Yes |
| Async update | ✅ Yes | ❌ Instant |

### Common Pattern: Combining Both

```javascript
function Stopwatch() {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);
  const startTimeRef = useRef(0);

  useEffect(() => {
    if (isRunning) {
      startTimeRef.current = Date.now() - time;
      intervalRef.current = setInterval(() => {
        setTime(Date.now() - startTimeRef.current);
      }, 10);
    } else {
      clearInterval(intervalRef.current);
    }

    return () => clearInterval(intervalRef.current);
  }, [isRunning, time]);

  const handleReset = () => {
    setTime(0);
    setIsRunning(false);
  };

  return (
    <div>
      <p>Time: {(time / 1000).toFixed(2)}s</p>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleReset}>Reset</button>
    </div>
  );
}
```

## Common Mistakes

### 1. Trying to Use Ref Value in Render

```javascript
// ❌ WRONG: Ref changes don't trigger re-renders
function Component() {
  const countRef = useRef(0);
  
  const increment = () => {
    countRef.current++;
    // UI won't update!
  };
  
  return (
    <div>
      <p>Count: {countRef.current}</p>
      <button onClick={increment}>Increment</button>
    </div>
  );
}

// ✅ CORRECT: Use useState for rendering
function Component() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

### 2. Not Cleaning Up Side Effects

```javascript
// ❌ WRONG: Memory leak
function Component() {
  const intervalRef = useRef(null);
  
  const startInterval = () => {
    intervalRef.current = setInterval(() => {
      console.log('Running');
    }, 1000);
  };
  
  return <button onClick={startInterval}>Start</button>;
  // Interval continues after unmount!
}

// ✅ CORRECT: Clean up on unmount
function Component() {
  const intervalRef = useRef(null);
  
  useEffect(() => {
    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);
  
  const startInterval = () => {
    intervalRef.current = setInterval(() => {
      console.log('Running');
    }, 1000);
  };
  
  return <button onClick={startInterval}>Start</button>;
}
```

### 3. Using Ref as Dependency

```javascript
// ❌ WRONG: Ref changes don't trigger useEffect
function Component() {
  const countRef = useRef(0);
  
  useEffect(() => {
    console.log('Count changed:', countRef.current);
  }, [countRef.current]); // This won't work as expected!
}

// ✅ CORRECT: Use useState if you need effect triggers
function Component() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    console.log('Count changed:', count);
  }, [count]);
}
```

### 4. Accessing Ref During Render

```javascript
// ❌ WRONG: Accessing ref during render
function Component() {
  const ref = useRef(null);
  
  // This might be null during initial render!
  const height = ref.current?.clientHeight;
  
  return <div ref={ref}>Content</div>;
}

// ✅ CORRECT: Access in effect or event handler
function Component() {
  const ref = useRef(null);
  const [height, setHeight] = useState(0);
  
  useEffect(() => {
    if (ref.current) {
      setHeight(ref.current.clientHeight);
    }
  }, []);
  
  return <div ref={ref}>Content</div>;
}
```

## Best Practices

### 1. Initialize with Null for DOM Refs

```javascript
// ✅ GOOD: Initialize DOM refs with null
const inputRef = useRef(null);

// ❌ BAD: Don't initialize with undefined
const inputRef = useRef();
```

### 2. Use Callback Refs for Dynamic Elements

```javascript
// ✅ GOOD: Callback ref for conditional rendering
function Component({ show }) {
  const [height, setHeight] = useState(0);
  
  const measuredRef = useCallback(node => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  }, []);
  
  return show ? <div ref={measuredRef}>Content</div> : null;
}
```

### 3. Don't Overuse Refs

```javascript
// ❌ BAD: Using refs for state that affects rendering
function Component() {
  const nameRef = useRef('');
  
  const updateName = (value) => {
    nameRef.current = value;
    // Need to manually trigger re-render - awkward!
    forceUpdate();
  };
}

// ✅ GOOD: Use useState for state that affects rendering
function Component() {
  const [name, setName] = useState('');
  
  return (
    <input 
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  );
}
```

### 4. Document Why You're Using Ref

```javascript
function Component() {
  // useRef: Store interval ID to clean up on unmount
  // Doesn't trigger re-render when changed
  const intervalRef = useRef(null);
  
  // useRef: Access DOM element for focus management
  const inputRef = useRef(null);
}
```

## Real-World Examples

### Scroll to Element

```javascript
function TableOfContents({ sections }) {
  const sectionRefs = useRef(new Map());

  const setRef = (id) => (element) => {
    if (element) {
      sectionRefs.current.set(id, element);
    } else {
      sectionRefs.current.delete(id);
    }
  };

  const scrollToSection = (id) => {
    const element = sectionRefs.current.get(id);
    element?.scrollIntoView({
      behavior: 'smooth',
      block: 'start'
    });
  };

  return (
    <div>
      <nav>
        <ul>
          {sections.map(section => (
            <li key={section.id}>
              <button onClick={() => scrollToSection(section.id)}>
                {section.title}
              </button>
            </li>
          ))}
        </ul>
      </nav>

      <main>
        {sections.map(section => (
          <section key={section.id} ref={setRef(section.id)}>
            <h2>{section.title}</h2>
            <p>{section.content}</p>
          </section>
        ))}
      </main>
    </div>
  );
}
```

### Click Outside Handler

```javascript
function useClickOutside(callback) {
  const ref = useRef(null);

  useEffect(() => {
    const handleClick = (event) => {
      if (ref.current && !ref.current.contains(event.target)) {
        callback();
      }
    };

    document.addEventListener('mousedown', handleClick);
    return () => {
      document.removeEventListener('mousedown', handleClick);
    };
  }, [callback]);

  return ref;
}

// Usage
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useClickOutside(() => setIsOpen(false));

  return (
    <div ref={dropdownRef}>
      <button onClick={() => setIsOpen(!isOpen)}>
        Toggle
      </button>
      {isOpen && (
        <div className="dropdown-menu">
          <button>Option 1</button>
          <button>Option 2</button>
        </div>
      )}
    </div>
  );
}
```

### Text Selection

```javascript
function SelectableText() {
  const textRef = useRef(null);
  const [selection, setSelection] = useState('');

  const handleSelect = () => {
    const element = textRef.current;
    if (!element) return;

    const selected = window.getSelection()?.toString();
    if (selected) {
      setSelection(selected);
    }
  };

  const highlightText = () => {
    const element = textRef.current;
    if (!element) return;

    const range = document.createRange();
    range.selectNodeContents(element);
    
    const selection = window.getSelection();
    selection?.removeAllRanges();
    selection?.addRange(range);
  };

  return (
    <div>
      <p ref={textRef} onMouseUp={handleSelect}>
        This is some selectable text. Try selecting parts of it!
      </p>
      
      {selection && (
        <div>
          <strong>Selected:</strong> {selection}
        </div>
      )}
      
      <button onClick={highlightText}>
        Highlight All
      </button>
    </div>
  );
}
```

## Interview Questions

### Q1: What's the difference between useRef and useState?

**Answer:**
- **useState**: Triggers re-render when updated; used for values that affect UI
- **useRef**: Doesn't trigger re-render; used for values that don't affect UI (timers, DOM refs, previous values)

```javascript
// useState - UI updates
const [count, setCount] = useState(0);

// useRef - no UI update
const countRef = useRef(0);
```

### Q2: Why use useRef instead of a regular variable?

**Answer:** Regular variables reset on every render, but useRef persists:

```javascript
function Component() {
  let count = 0; // Resets to 0 every render
  const countRef = useRef(0); // Persists across renders
  
  const increment = () => {
    count++; // Lost on next render
    countRef.current++; // Persists
  };
}
```

### Q3: How do you access a child component's DOM node?

**Answer:** Use `forwardRef`:

```javascript
const Child = forwardRef((props, ref) => {
  return <input ref={ref} />;
});

function Parent() {
  const inputRef = useRef(null);
  
  return <Child ref={inputRef} />;
}
```

### Q4: Can you use useRef in the dependency array?

**Answer:** No, because ref.current changes don't trigger re-renders or effect re-runs. The ref object itself is stable, but its .current property can change without React knowing.

## Key Takeaways

1. **useRef persists across renders** - Without triggering re-renders
2. **Two main uses** - DOM access and mutable values
3. **Returns object with .current** - Mutable property
4. **Doesn't trigger re-renders** - Changes to .current are silent
5. **Perfect for side effects** - Timers, subscriptions, DOM manipulation
6. **Use forwardRef for components** - To expose refs to parents
7. **Clean up side effects** - Always clear timers/listeners on unmount

## Resources

### Official Documentation
- [React useRef Docs](https://react.dev/reference/react/useRef)
- [Manipulating the DOM with Refs](https://react.dev/learn/manipulating-the-dom-with-refs)
- [forwardRef](https://react.dev/reference/react/forwardRef)

### Articles
- [A Complete Guide to useRef](https://www.robinwieruch.de/react-useref-hook)
- [useRef vs useState](https://kentcdodds.com/blog/usestate-vs-useref)

### Related Hooks
- [useImperativeHandle](https://react.dev/reference/react/useImperativeHandle)

---

**Next Steps:** Learn about custom hooks for reusable logic, and explore useImperativeHandle for advanced ref patterns.
