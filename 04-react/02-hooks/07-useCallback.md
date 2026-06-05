# useCallback Hook - Optimizing Function References

## The Idea

**In plain English:** `useCallback` is a tool in React (a popular library for building websites) that remembers a function so it does not get recreated from scratch every time your page updates. A "function" is just a set of instructions your app follows when something happens, like clicking a button.

**Real-world analogy:** Imagine a teacher who writes a set of homework instructions on the board each day. On most days the instructions are identical, yet she erases and rewrites them every single morning anyway — wasting chalk and time. A smarter approach: laminate the instruction sheet and reuse it until the assignment actually changes.

- The laminated instruction sheet = the memoized (saved) function `useCallback` returns
- Rewriting identical instructions every morning = creating a brand-new function on every render
- The teacher only reprinting the sheet when the assignment changes = `useCallback` only recreating the function when its listed dependencies change

---

## Table of Contents
- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Core API](#core-api)
- [When to Use useCallback](#when-to-use-usecallback)
- [Common Patterns](#common-patterns)
- [Performance Implications](#performance-implications)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

`useCallback` is a React Hook that returns a memoized version of a callback function. It's primarily used to prevent unnecessary re-renders by maintaining stable function references across component renders.

### The Problem useCallback Solves

```javascript
// ❌ Problem: New function on every render
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  // New function reference on every render!
  const handleClick = () => {
    console.log('Clicked');
  };
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      {/* ChildComponent re-renders even when count changes */}
      <ChildComponent onClick={handleClick} />
    </div>
  );
}

const ChildComponent = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Child Button</button>;
});

// ✅ Solution: Stable function reference
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  // Same function reference across renders
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // Empty deps = never changes
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      {/* ChildComponent only renders when handleClick changes */}
      <ChildComponent onClick={handleClick} />
    </div>
  );
}
```

### Visual Representation

```
Without useCallback:              With useCallback:
┌──────────────────┐             ┌──────────────────┐
│  Parent Renders  │             │  Parent Renders  │
└────────┬─────────┘             └────────┬─────────┘
         │                                │
         │ New function                   │ Same function
         │ created                        │ reference
         ↓                                ↓
┌──────────────────┐             ┌──────────────────┐
│ Child Re-renders │             │ Child Skips      │
│ (unnecessary)    │             │ (optimized)      │
└──────────────────┘             └──────────────────┘
```

## Basic Concepts

### Syntax

```javascript
const memoizedCallback = useCallback(
  callbackFunction,
  dependencies
);
```

**Parameters:**
- `callbackFunction`: The function you want to memoize
- `dependencies`: Array of values the function depends on

**Returns:**
- Memoized version of the callback that only changes if dependencies change

### How It Works

```javascript
function Component({ data }) {
  // Function is recreated only when 'data' changes
  const handleProcess = useCallback(() => {
    processData(data);
  }, [data]);
  
  // Equivalent to (conceptually):
  // const handleProcess = useMemo(() => {
  //   return () => processData(data);
  // }, [data]);
  
  return <ExpensiveChild onProcess={handleProcess} />;
}
```

## Core API

### Basic Usage

```javascript
import { useState, useCallback } from 'react';

function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // Memoized search handler
  const handleSearch = useCallback(async (searchTerm) => {
    const data = await fetchSearchResults(searchTerm);
    setResults(data);
  }, []); // No dependencies - function never changes

  return (
    <div>
      <input 
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      <button onClick={() => handleSearch(query)}>
        Search
      </button>
      <ResultsList results={results} />
    </div>
  );
}
```

### With Dependencies

```javascript
function TodoItem({ id, onUpdate }) {
  const [text, setText] = useState('');
  const [priority, setPriority] = useState('low');

  // Function recreated only when id or priority changes
  const handleSave = useCallback(() => {
    onUpdate(id, {
      text,
      priority,
      updatedAt: Date.now()
    });
  }, [id, priority, onUpdate]); // Dependencies

  return (
    <div>
      <input 
        value={text}
        onChange={(e) => setText(e.target.value)}
      />
      <select 
        value={priority}
        onChange={(e) => setPriority(e.target.value)}
      >
        <option value="low">Low</option>
        <option value="high">High</option>
      </select>
      <button onClick={handleSave}>Save</button>
    </div>
  );
}
```

### Event Handlers

```javascript
function Form() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });

  // Memoized field change handler
  const handleFieldChange = useCallback((field) => {
    return (e) => {
      setFormData(prev => ({
        ...prev,
        [field]: e.target.value
      }));
    };
  }, []); // setFormData is stable, so no deps needed

  const handleSubmit = useCallback(async (e) => {
    e.preventDefault();
    await submitForm(formData);
  }, [formData]);

  return (
    <form onSubmit={handleSubmit}>
      <input 
        name="name"
        value={formData.name}
        onChange={handleFieldChange('name')}
      />
      <input 
        name="email"
        value={formData.email}
        onChange={handleFieldChange('email')}
      />
      <textarea 
        name="message"
        value={formData.message}
        onChange={handleFieldChange('message')}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## When to Use useCallback

### Decision Matrix

| Scenario | Use useCallback | Don't Use |
|----------|----------------|-----------|
| Passing to memoized child | ✅ | ❌ |
| Dependency in useEffect | ✅ | ❌ |
| Dependency in useMemo | ✅ | ❌ |
| Passing to non-memoized child | ❌ | ✅ |
| Simple event handlers | ❌ | ✅ |
| Functions not passed as props | ❌ | ✅ |

### Good Use Cases

```javascript
// ✅ GOOD: Child component is memoized
const MemoizedChild = React.memo(ChildComponent);

function Parent() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  
  return <MemoizedChild onClick={handleClick} />;
}

// ✅ GOOD: Function used as effect dependency
function Component({ userId }) {
  const fetchUser = useCallback(async () => {
    const data = await fetch(`/api/users/${userId}`);
    return data.json();
  }, [userId]);

  useEffect(() => {
    fetchUser().then(setUser);
  }, [fetchUser]); // Won't cause infinite loop
}

// ✅ GOOD: Expensive computation depends on function
function Component({ data }) {
  const processData = useCallback(() => {
    return expensiveOperation(data);
  }, [data]);

  const result = useMemo(() => {
    return processData();
  }, [processData]);
}
```

### Bad Use Cases

```javascript
// ❌ BAD: Premature optimization
function Component() {
  // Unnecessary - child isn't memoized
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  
  return <RegularButton onClick={handleClick} />;
}

// ❌ BAD: Adding overhead for simple handlers
function Component() {
  const [count, setCount] = useState(0);
  
  // Just use inline - simpler and faster
  const increment = useCallback(() => {
    setCount(c => c + 1);
  }, []);
  
  return <button onClick={increment}>Count: {count}</button>;
}

// ✅ BETTER: Inline handler
function Component() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

## Common Patterns

### Callback with Parameters

```javascript
function ListComponent({ items }) {
  // ❌ WRONG: Creates new function on every render
  return items.map(item => (
    <div key={item.id}>
      <button onClick={() => handleDelete(item.id)}>
        Delete
      </button>
    </div>
  ));

  // ✅ BETTER: Extract to child component
  return items.map(item => (
    <ListItem key={item.id} item={item} onDelete={handleDelete} />
  ));
}

const ListItem = React.memo(({ item, onDelete }) => {
  const handleClick = useCallback(() => {
    onDelete(item.id);
  }, [item.id, onDelete]);

  return (
    <div>
      <button onClick={handleClick}>Delete</button>
    </div>
  );
});
```

### Factory Functions

```javascript
function Form() {
  const [values, setValues] = useState({});

  // Factory function that returns memoized handlers
  const createChangeHandler = useCallback((fieldName) => {
    return (e) => {
      setValues(prev => ({
        ...prev,
        [fieldName]: e.target.value
      }));
    };
  }, []);

  return (
    <div>
      <input onChange={createChangeHandler('firstName')} />
      <input onChange={createChangeHandler('lastName')} />
      <input onChange={createChangeHandler('email')} />
    </div>
  );
}
```

### With Custom Hooks

```javascript
function useDebounce(callback, delay) {
  const timeoutRef = useRef(null);

  return useCallback((...args) => {
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }

    timeoutRef.current = setTimeout(() => {
      callback(...args);
    }, delay);
  }, [callback, delay]);
}

// Usage
function SearchInput() {
  const [query, setQuery] = useState('');

  const handleSearch = useCallback((value) => {
    console.log('Searching:', value);
    // Actual search logic
  }, []);

  const debouncedSearch = useDebounce(handleSearch, 300);

  const handleChange = useCallback((e) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  }, [debouncedSearch]);

  return (
    <input 
      value={query}
      onChange={handleChange}
      placeholder="Search..."
    />
  );
}
```

### Combining with useRef

```javascript
function InfiniteScroll({ loadMore }) {
  const observerRef = useRef(null);
  const loadMoreRef = useRef(loadMore);

  // Keep ref up to date
  useEffect(() => {
    loadMoreRef.current = loadMore;
  }, [loadMore]);

  // Stable callback that always uses latest loadMore
  const handleIntersection = useCallback((entries) => {
    const [entry] = entries;
    if (entry.isIntersecting) {
      loadMoreRef.current();
    }
  }, []); // No dependencies!

  useEffect(() => {
    const observer = new IntersectionObserver(handleIntersection);
    if (observerRef.current) {
      observer.observe(observerRef.current);
    }
    return () => observer.disconnect();
  }, [handleIntersection]);

  return <div ref={observerRef}>Loading...</div>;
}
```

## Performance Implications

### Memory vs CPU Trade-off

```javascript
// Without useCallback: Less memory, more CPU
function Component() {
  const handleClick = () => {
    console.log('Clicked');
  };
  // New function created on every render (CPU cost)
  // Old function garbage collected (memory freed)
}

// With useCallback: More memory, less CPU
function Component() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  // Same function reference kept in memory
  // No new function creation (CPU saved)
  // But uses memory to cache
}
```

### Measuring Impact

```javascript
import { Profiler } from 'react';

function App() {
  const onRender = useCallback((
    id,
    phase,
    actualDuration,
    baseDuration,
    startTime,
    commitTime
  ) => {
    console.log({
      component: id,
      phase,
      actualDuration,
      baseDuration
    });
  }, []);

  return (
    <Profiler id="App" onRender={onRender}>
      <YourComponent />
    </Profiler>
  );
}
```

### When Optimization Matters

```javascript
// ❌ Premature optimization for simple components
function SimpleCounter() {
  const [count, setCount] = useState(0);
  
  // Overhead of useCallback > benefit
  const increment = useCallback(() => {
    setCount(c => c + 1);
  }, []);
  
  return <button onClick={increment}>{count}</button>;
}

// ✅ Worthwhile for expensive child components
function DataTable({ data }) {
  const [sortField, setSortField] = useState('name');
  
  // ExpensiveTable renders 1000s of rows
  // Preventing re-render is worth the overhead
  const handleSort = useCallback((field) => {
    setSortField(field);
  }, []);
  
  return <ExpensiveTable data={data} onSort={handleSort} />;
}
```

## Common Mistakes

### 1. Missing Dependencies

```javascript
// ❌ WRONG: Missing 'count' dependency
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    console.log(count); // Stale closure!
  }, []); // Should include 'count'
  
  return <button onClick={handleClick}>Log: {count}</button>;
}

// ✅ CORRECT: Include all dependencies
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    console.log(count);
  }, [count]); // Recreates when count changes
  
  return <button onClick={handleClick}>Log: {count}</button>;
}

// ✅ ALTERNATIVE: Use functional update
function Component() {
  const [count, setCount] = useState(0);
  
  const handleIncrement = useCallback(() => {
    setCount(c => c + 1); // No dependency on count needed
  }, []);
  
  return <button onClick={handleIncrement}>Increment</button>;
}
```

### 2. Overusing useCallback

```javascript
// ❌ BAD: Using everywhere unnecessarily
function Component() {
  const handleClick = useCallback(() => {}, []);
  const handleChange = useCallback(() => {}, []);
  const handleSubmit = useCallback(() => {}, []);
  const handleReset = useCallback(() => {}, []);
  // ... more useCallbacks
  
  return <SimpleForm />; // Not even memoized!
}

// ✅ GOOD: Only where needed
function Component() {
  const [data, setData] = useState([]);
  
  // Only this one matters - passed to memoized child
  const handleUpdate = useCallback((id, newData) => {
    setData(prev => 
      prev.map(item => item.id === id ? newData : item)
    );
  }, []);
  
  return <MemoizedList items={data} onUpdate={handleUpdate} />;
}
```

### 3. Not Memoizing Child Component

```javascript
// ❌ WRONG: useCallback without React.memo
function Parent() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  
  // Child isn't memoized, so useCallback is pointless
  return <Child onClick={handleClick} />;
}

function Child({ onClick }) {
  return <button onClick={onClick}>Click</button>;
}

// ✅ CORRECT: Memoize child component
const Child = React.memo(({ onClick }) => {
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []);
  
  return <Child onClick={handleClick} />;
}
```

### 4. Circular Dependencies

```javascript
// ❌ WRONG: Circular dependency
function Component() {
  const callbackA = useCallback(() => {
    callbackB(); // Uses callbackB
  }, [callbackB]);
  
  const callbackB = useCallback(() => {
    callbackA(); // Uses callbackA
  }, [callbackA]);
  // Infinite loop!
}

// ✅ CORRECT: Restructure logic
function Component() {
  const sharedLogic = useCallback(() => {
    // Shared code
  }, []);
  
  const callbackA = useCallback(() => {
    sharedLogic();
    // A-specific code
  }, [sharedLogic]);
  
  const callbackB = useCallback(() => {
    sharedLogic();
    // B-specific code
  }, [sharedLogic]);
}
```

## Best Practices

### 1. Profile Before Optimizing

```javascript
// Don't blindly add useCallback
// First, measure if there's a performance problem

import { Profiler } from 'react';

function App() {
  return (
    <Profiler id="MyComponent" onRender={logTiming}>
      <MyComponent />
    </Profiler>
  );
}

function logTiming(id, phase, actualDuration) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}
```

### 2. Use ESLint exhaustive-deps Rule

```javascript
// .eslintrc.js
{
  "rules": {
    "react-hooks/exhaustive-deps": "warn"
  }
}

// ESLint will warn about missing dependencies
const handleClick = useCallback(() => {
  console.log(someValue); // ESLint: add 'someValue' to deps
}, []); // ⚠️ Warning
```

### 3. Combine with Other Hooks

```javascript
function useAPI(endpoint) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);

  const fetchData = useCallback(async () => {
    setLoading(true);
    try {
      const response = await fetch(endpoint);
      const json = await response.json();
      setData(json);
    } finally {
      setLoading(false);
    }
  }, [endpoint]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  const refetch = useCallback(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, refetch };
}
```

### 4. Document Why You're Using It

```javascript
function Component({ onUpdate }) {
  // useCallback: Prevent ExpensiveList re-renders
  // ExpensiveList renders 10,000+ items and is memoized
  const handleUpdate = useCallback((id, data) => {
    onUpdate(id, data);
  }, [onUpdate]);

  return <ExpensiveList onUpdate={handleUpdate} />;
}
```

## Real-World Examples

### Debounced Search

```javascript
function SearchWithDebounce() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  const searchAPI = useCallback(async (searchTerm) => {
    if (!searchTerm.trim()) {
      setResults([]);
      return;
    }

    setLoading(true);
    try {
      const response = await fetch(
        `/api/search?q=${encodeURIComponent(searchTerm)}`
      );
      const data = await response.json();
      setResults(data);
    } catch (error) {
      console.error('Search failed:', error);
    } finally {
      setLoading(false);
    }
  }, []);

  const debouncedSearch = useDebounce(searchAPI, 300);

  const handleInputChange = useCallback((e) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  }, [debouncedSearch]);

  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={handleInputChange}
        placeholder="Search..."
      />
      {loading && <Spinner />}
      <SearchResults results={results} />
    </div>
  );
}
```

### Optimized List with Actions

```javascript
const ListItem = React.memo(({ item, onEdit, onDelete }) => {
  console.log('ListItem render:', item.id);
  
  return (
    <div>
      <span>{item.name}</span>
      <button onClick={() => onEdit(item.id)}>Edit</button>
      <button onClick={() => onDelete(item.id)}>Delete</button>
    </div>
  );
});

function ItemList({ items }) {
  const [editingId, setEditingId] = useState(null);

  const handleEdit = useCallback((id) => {
    setEditingId(id);
  }, []);

  const handleDelete = useCallback((id) => {
    // API call to delete
    deleteItem(id);
  }, []);

  return (
    <div>
      {items.map(item => (
        <ListItem
          key={item.id}
          item={item}
          onEdit={handleEdit}
          onDelete={handleDelete}
        />
      ))}
    </div>
  );
}
```

### Form with Validation

```javascript
function RegistrationForm() {
  const [formData, setFormData] = useState({
    email: '',
    password: '',
    confirmPassword: ''
  });
  const [errors, setErrors] = useState({});

  const validateField = useCallback((field, value) => {
    switch (field) {
      case 'email':
        return /\S+@\S+\.\S+/.test(value) 
          ? '' 
          : 'Invalid email';
      case 'password':
        return value.length >= 8 
          ? '' 
          : 'Password must be 8+ characters';
      case 'confirmPassword':
        return value === formData.password 
          ? '' 
          : 'Passwords do not match';
      default:
        return '';
    }
  }, [formData.password]);

  const handleFieldChange = useCallback((field) => {
    return (e) => {
      const value = e.target.value;
      setFormData(prev => ({ ...prev, [field]: value }));
      
      const error = validateField(field, value);
      setErrors(prev => ({ ...prev, [field]: error }));
    };
  }, [validateField]);

  const handleSubmit = useCallback(async (e) => {
    e.preventDefault();
    
    // Validate all fields
    const newErrors = {};
    Object.keys(formData).forEach(field => {
      const error = validateField(field, formData[field]);
      if (error) newErrors[field] = error;
    });

    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }

    // Submit form
    await registerUser(formData);
  }, [formData, validateField]);

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={formData.email}
        onChange={handleFieldChange('email')}
      />
      {errors.email && <span>{errors.email}</span>}
      
      <input
        type="password"
        value={formData.password}
        onChange={handleFieldChange('password')}
      />
      {errors.password && <span>{errors.password}</span>}
      
      <input
        type="password"
        value={formData.confirmPassword}
        onChange={handleFieldChange('confirmPassword')}
      />
      {errors.confirmPassword && <span>{errors.confirmPassword}</span>}
      
      <button type="submit">Register</button>
    </form>
  );
}
```

## Interview Questions

### Q1: What's the difference between useCallback and useMemo?

**Answer:**
- **useCallback** returns a memoized function
- **useMemo** returns a memoized value

```javascript
// useCallback - memoizes the function itself
const memoizedCallback = useCallback(
  () => doSomething(a, b),
  [a, b]
);

// useMemo - memoizes the return value
const memoizedValue = useMemo(
  () => computeExpensiveValue(a, b),
  [a, b]
);

// Equivalence
useCallback(fn, deps) === useMemo(() => fn, deps)
```

### Q2: Does useCallback improve performance in all cases?

**Answer:** No. useCallback has overhead:
- Memory to store cached function
- Comparison logic for dependencies
- Only beneficial when:
  - Passed to memoized child components
  - Used as dependency in other hooks
  - Function creation is expensive (rare)

Premature optimization with useCallback can make code slower and more complex.

### Q3: How do you handle events with parameters in useCallback?

**Answer:** Three approaches:

```javascript
// 1. Curry the function
const handleClick = useCallback((id) => {
  return () => deleteItem(id);
}, []);

// 2. Use data attributes
const handleClick = useCallback((e) => {
  const id = e.currentTarget.dataset.id;
  deleteItem(id);
}, []);

// 3. Extract to separate component
const Item = React.memo(({ id, onDelete }) => {
  const handleClick = useCallback(() => {
    onDelete(id);
  }, [id, onDelete]);
  
  return <button onClick={handleClick}>Delete</button>;
});
```

## Key Takeaways

1. **useCallback prevents function recreation** - Returns same reference across renders
2. **Only optimize when needed** - Profile first, optimize later
3. **Requires React.memo for child optimization** - Without memoization, useCallback is overhead
4. **Include all dependencies** - Use ESLint exhaustive-deps rule
5. **Consider functional updates** - Reduce dependencies with setState(prev => ...)
6. **Memory vs CPU tradeoff** - useCallback uses memory to save CPU
7. **Stable functions enable other optimizations** - Safe for useEffect/useMemo dependencies

## Resources

### Official Documentation
- [React useCallback Docs](https://react.dev/reference/react/useCallback)
- [You Might Not Need useCallback](https://react.dev/learn/you-might-not-need-an-effect)

### Articles
- [When to useMemo and useCallback](https://kentcdodds.com/blog/usememo-and-usecallback)
- [React Hooks: Understanding useCallback](https://www.robinwieruch.de/react-usecallback-hook)

### Tools
- [React DevTools Profiler](https://react.dev/learn/react-developer-tools)
- [Why Did You Render](https://github.com/welldone-software/why-did-you-render)

---

**Next Steps:** Learn about useMemo for memoizing computed values, and explore React.memo for component memoization.
