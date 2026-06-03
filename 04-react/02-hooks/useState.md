# useState: Functional Updates and Lazy Initialization

## Overview

`useState` is React's most fundamental hook for managing component state. It lets function components have state that persists across renders. This document covers not just the basics, but advanced patterns like functional updates and lazy initialization.

## Basic useState

### Declaring State

```jsx
import { useState } from 'react';

function Counter() {
  // Declare state variable
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

**Syntax:**
```jsx
const [state, setState] = useState(initialValue);
```

- `state`: Current state value
- `setState`: Function to update state
- `initialValue`: Initial state value

### Multiple State Variables

```jsx
function UserForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [age, setAge] = useState(0);
  
  return (
    <form>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Name"
      />
      <input
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="number"
        value={age}
        onChange={(e) => setAge(Number(e.target.value))}
        placeholder="Age"
      />
    </form>
  );
}
```

### State with Objects

```jsx
function UserProfile() {
  const [user, setUser] = useState({
    name: '',
    email: '',
    age: 0
  });
  
  // Update individual properties
  const updateName = (name) => {
    setUser({ ...user, name });
  };
  
  const updateEmail = (email) => {
    setUser({ ...user, email });
  };
  
  return (
    <div>
      <input
        value={user.name}
        onChange={(e) => updateName(e.target.value)}
      />
      <input
        value={user.email}
        onChange={(e) => updateEmail(e.target.value)}
      />
    </div>
  );
}
```

### State with Arrays

```jsx
function TodoList() {
  const [todos, setTodos] = useState([]);
  
  const addTodo = (text) => {
    setTodos([...todos, { id: Date.now(), text }]);
  };
  
  const removeTodo = (id) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };
  
  const toggleTodo = (id) => {
    setTodos(
      todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
          <button onClick={() => removeTodo(todo.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

## Functional Updates

Functional updates use a function instead of a value to update state based on the previous state.

### Why Functional Updates?

```jsx
// Problem: Stale closure
function Counter() {
  const [count, setCount] = useState(0);
  
  const increment = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    // All three use the same count value (0)
    // Final count: 1 (not 3!)
  };
  
  return <button onClick={increment}>Count: {count}</button>;
}

// Solution: Functional update
function Counter() {
  const [count, setCount] = useState(0);
  
  const increment = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    // Each update uses the latest value
    // Final count: 3 ✓
  };
  
  return <button onClick={increment}>Count: {count}</button>;
}
```

### Functional Update Pattern

```jsx
// Generic pattern
setState(previousState => newState);

// Examples
setCount(prev => prev + 1);
setUser(prev => ({ ...prev, name: 'Alice' }));
setTodos(prev => [...prev, newTodo]);
setItems(prev => prev.filter(item => item.id !== id));
```

### When to Use Functional Updates

**1. Multiple rapid updates**

```jsx
function RapidCounter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    // Without functional update - wrong
    setTimeout(() => setCount(count + 1), 1000);
    setTimeout(() => setCount(count + 1), 2000);
    setTimeout(() => setCount(count + 1), 3000);
    // All use stale count value
    
    // With functional update - correct
    setTimeout(() => setCount(c => c + 1), 1000);
    setTimeout(() => setCount(c => c + 1), 2000);
    setTimeout(() => setCount(c => c + 1), 3000);
    // Each uses current value
  };
  
  return <button onClick={handleClick}>Count: {count}</button>;
}
```

**2. Event handlers that use previous state**

```jsx
function ToggleButton() {
  const [isOn, setIsOn] = useState(false);
  
  const toggle = () => {
    // Always toggles correctly, even if called rapidly
    setIsOn(prev => !prev);
  };
  
  return (
    <button onClick={toggle}>
      {isOn ? 'ON' : 'OFF'}
    </button>
  );
}
```

**3. State updates in useEffect**

```jsx
function AutoIncrement() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      // Functional update avoids stale closure
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // Empty deps - interval never recreated
  
  return <div>Count: {count}</div>;
}
```

**4. Optimized event handlers**

```jsx
function OptimizedCounter() {
  const [count, setCount] = useState(0);
  
  // Can be memoized because it doesn't depend on count
  const increment = useCallback(() => {
    setCount(c => c + 1);
  }, []); // No dependencies!
  
  return (
    <div>
      <p>Count: {count}</p>
      <ExpensiveChild onIncrement={increment} />
    </div>
  );
}
```

### Complex Functional Updates

```jsx
// Array operations
function TodoList() {
  const [todos, setTodos] = useState([]);
  
  const addTodo = (text) => {
    setTodos(prev => [
      ...prev,
      { id: Date.now(), text, completed: false }
    ]);
  };
  
  const toggleTodo = (id) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  };
  
  const removeTodo = (id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };
  
  return <div>{/* Render todos */}</div>;
}

// Object operations
function UserSettings() {
  const [settings, setSettings] = useState({
    theme: 'light',
    language: 'en',
    notifications: {
      email: true,
      push: false
    }
  });
  
  const updateTheme = (theme) => {
    setSettings(prev => ({ ...prev, theme }));
  };
  
  const updateNotification = (type, value) => {
    setSettings(prev => ({
      ...prev,
      notifications: {
        ...prev.notifications,
        [type]: value
      }
    }));
  };
  
  return <div>{/* Render settings */}</div>;
}
```

## Lazy Initialization

Lazy initialization runs the initial state computation only once, not on every render.

### The Problem

```jsx
function ExpensiveComponent() {
  // This runs on EVERY render (even though result is ignored)
  const [data, setData] = useState(expensiveComputation());
  
  return <div>{data}</div>;
}

function expensiveComputation() {
  console.log('Computing...');
  let result = 0;
  for (let i = 0; i < 1000000; i++) {
    result += i;
  }
  return result;
}
```

### The Solution

```jsx
function ExpensiveComponent() {
  // Function runs ONLY on first render
  const [data, setData] = useState(() => expensiveComputation());
  
  return <div>{data}</div>;
}

// Or with arrow function inline
function ExpensiveComponent() {
  const [data, setData] = useState(() => {
    console.log('Computing...');
    let result = 0;
    for (let i = 0; i < 1000000; i++) {
      result += i;
    }
    return result;
  });
  
  return <div>{data}</div>;
}
```

### When to Use Lazy Initialization

**1. Expensive computations**

```jsx
function DataProcessor({ largeDataset }) {
  const [processed, setProcessed] = useState(() => {
    // Only runs once
    return processLargeDataset(largeDataset);
  });
  
  return <div>{processed}</div>;
}
```

**2. Reading from localStorage**

```jsx
function PersistentForm() {
  const [formData, setFormData] = useState(() => {
    // Only reads from localStorage once
    const saved = localStorage.getItem('formData');
    return saved ? JSON.parse(saved) : { name: '', email: '' };
  });
  
  // Save on change
  useEffect(() => {
    localStorage.setItem('formData', JSON.stringify(formData));
  }, [formData]);
  
  return <form>{/* Form fields */}</form>;
}
```

**3. Complex initial state**

```jsx
function GameBoard({ config }) {
  const [board, setBoard] = useState(() => {
    // Complex initialization
    const rows = config.rows;
    const cols = config.cols;
    const board = Array(rows).fill(null).map(() =>
      Array(cols).fill(null).map(() => ({
        value: 0,
        revealed: false,
        flagged: false
      }))
    );
    return board;
  });
  
  return <div>{/* Render board */}</div>;
}
```

**4. Props-based initialization**

```jsx
function EditableText({ initialText }) {
  // Only uses initialText on mount
  const [text, setText] = useState(() => initialText);
  
  return (
    <input
      value={text}
      onChange={(e) => setText(e.target.value)}
    />
  );
}
```

**5. Random values**

```jsx
function UniqueID() {
  const [id] = useState(() => {
    // Generate ID only once
    return `id-${Date.now()}-${Math.random()}`;
  });
  
  return <div id={id}>Content</div>;
}
```

### Lazy Initialization Pattern

```jsx
// Without lazy init - runs every render
useState(computeExpensiveValue(prop));

// With lazy init - runs only once
useState(() => computeExpensiveValue(prop));

// Or extract to variable for clarity
function Component({ prop }) {
  const initialState = useCallback(() => {
    return computeExpensiveValue(prop);
  }, []);
  
  const [state] = useState(initialState);
}
```

## Common Patterns

### 1. Derived State (Anti-pattern)

```jsx
// Bad: Duplicating props in state
function SearchResults({ query }) {
  const [searchQuery, setSearchQuery] = useState(query);
  const [results, setResults] = useState([]);
  
  // Problem: searchQuery doesn't update when query prop changes
  
  return <div>{/* ... */}</div>;
}

// Good: Use props directly or derive in render
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    fetchResults(query).then(setResults);
  }, [query]);
  
  return <div>{/* ... */}</div>;
}
```

### 2. Previous Value Tracking

```jsx
function Component({ value }) {
  const [prevValue, setPrevValue] = useState(value);
  const [count, setCount] = useState(0);
  
  // Update count when value changes
  if (value !== prevValue) {
    setPrevValue(value);
    setCount(c => c + 1);
  }
  
  return <div>Value changed {count} times</div>;
}

// Better: Use useEffect
function Component({ value }) {
  const [count, setCount] = useState(0);
  const prevValue = useRef(value);
  
  useEffect(() => {
    if (value !== prevValue.current) {
      setCount(c => c + 1);
      prevValue.current = value;
    }
  }, [value]);
  
  return <div>Value changed {count} times</div>;
}
```

### 3. Resetting State on Prop Change

```jsx
// Use key to reset state
function EditableUser({ userId }) {
  const [name, setName] = useState('');
  
  // Problem: name doesn't reset when userId changes
  
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}

// Solution: Use key
function Parent() {
  const [userId, setUserId] = useState(1);
  
  return (
    <EditableUser
      key={userId} // Component resets when userId changes
      userId={userId}
    />
  );
}
```

### 4. Object State Updates

```jsx
function ProfileForm() {
  const [profile, setProfile] = useState({
    name: '',
    email: '',
    bio: ''
  });
  
  // Generic update function
  const updateField = (field, value) => {
    setProfile(prev => ({ ...prev, [field]: value }));
  };
  
  // Or use a single handler
  const handleChange = (e) => {
    const { name, value } = e.target;
    setProfile(prev => ({ ...prev, [name]: value }));
  };
  
  return (
    <form>
      <input
        name="name"
        value={profile.name}
        onChange={handleChange}
      />
      <input
        name="email"
        value={profile.email}
        onChange={handleChange}
      />
      <textarea
        name="bio"
        value={profile.bio}
        onChange={handleChange}
      />
    </form>
  );
}
```

### 5. State Reset

```jsx
function Form() {
  const initialState = {
    name: '',
    email: '',
    message: ''
  };
  
  const [formData, setFormData] = useState(initialState);
  
  const handleSubmit = (e) => {
    e.preventDefault();
    submitForm(formData);
    setFormData(initialState); // Reset to initial state
  };
  
  return <form onSubmit={handleSubmit}>{/* Form fields */}</form>;
}
```

## Performance Considerations

### 1. State Updates Trigger Re-renders

```jsx
function Component() {
  const [count, setCount] = useState(0);
  
  // This triggers a re-render
  setCount(1);
  
  // This doesn't (same value)
  setCount(1);
  setCount(1);
  // React bails out if new state is the same
}
```

### 2. Batch Updates

```jsx
function Component() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  
  const handleClick = () => {
    // React 18: Automatically batched
    setCount(c => c + 1);
    setFlag(f => !f);
    // Only one re-render
  };
  
  return <button onClick={handleClick}>Update</button>;
}
```

### 3. Split State Wisely

```jsx
// Bad: Updating one field causes re-render even if others don't change
function Form() {
  const [form, setForm] = useState({
    name: '',
    email: '',
    longBio: '' // Expensive to render
  });
  
  // Changing name re-renders entire form
  const updateName = (name) => {
    setForm({ ...form, name });
  };
}

// Good: Split independent state
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [longBio, setLongBio] = useState('');
  
  // Changing name doesn't affect bio rendering
}
```

## Common Mistakes

### 1. Mutating State

```jsx
// Wrong: Mutating state directly
function TodoList() {
  const [todos, setTodos] = useState([]);
  
  const addTodo = (text) => {
    todos.push({ text }); // ❌ Mutation!
    setTodos(todos);
  };
}

// Correct: Create new array
function TodoList() {
  const [todos, setTodos] = useState([]);
  
  const addTodo = (text) => {
    setTodos([...todos, { text }]); // ✓ New array
  };
}
```

### 2. Using State Immediately After Setting

```jsx
// Wrong: State update is async
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);
    console.log(count); // Still old value!
  };
}

// Correct: Use the value you're setting
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    const newCount = count + 1;
    setCount(newCount);
    console.log(newCount); // Correct value
  };
}
```

### 3. Conditional State Declarations

```jsx
// Wrong: Number of hooks must be consistent
function Component({ shouldShowCount }) {
  if (shouldShowCount) {
    const [count, setCount] = useState(0); // ❌ Conditional hook
  }
}

// Correct: Always declare hooks
function Component({ shouldShowCount }) {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      {shouldShowCount && <div>Count: {count}</div>}
    </div>
  );
}
```

## Summary

- `useState` adds state to function components
- Returns [state, setState] pair
- State updates trigger re-renders
- Use functional updates when new state depends on previous state
- Functional updates prevent stale closure bugs
- Use lazy initialization for expensive initial computations
- Lazy initializer runs only once on mount
- State updates are asynchronous and batched
- Never mutate state directly
- Split state based on update patterns for performance
- React bails out of re-renders if new state equals old state

Understanding `useState` deeply, including functional updates and lazy initialization, is essential for writing performant, bug-free React applications.
