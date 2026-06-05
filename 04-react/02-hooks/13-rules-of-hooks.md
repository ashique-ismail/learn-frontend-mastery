# Rules of Hooks (and Why They Exist)

## The Idea

**In plain English:** Rules of Hooks are a small set of guidelines you must follow when using React's special functions called "hooks" (built-in tools that let a component remember things and react to changes). Breaking these rules causes your app to behave unpredictably because React keeps track of hooks by counting them in order, and that count must stay the same every time a component runs.

**Real-world analogy:** Imagine a chef who prepares a fixed recipe card in the same order every morning — first crack two eggs, then add flour, then pour milk. The kitchen assistant tracks each step by position (step 1, step 2, step 3). If one morning the chef skips the egg step because they feel like it, the assistant's notes are now completely wrong: what was "add flour" is now labeled "crack eggs," and everything that follows is mixed up.

- The recipe card = the list of hooks in a component
- Each numbered step = one hook call (like `useState` or `useEffect`)
- The kitchen assistant's position-based notes = React's internal linked list that maps each hook call to its saved value
- Skipping or reordering a step = calling a hook inside an `if` block or loop

---

## Overview

The **Rules of Hooks** are constraints that ensure hooks work correctly with React's internal architecture. While they may seem arbitrary, they're fundamental to how hooks are implemented.

## The Two Rules

### Rule 1: Only Call Hooks at the Top Level

**Don't call hooks inside conditions, loops, or nested functions.**

```jsx
// ✗ Bad: Conditional hook
function Component({ condition }) {
  if (condition) {
    const [state, setState] = useState(0);  // Error!
  }
}

// ✗ Bad: Hook in loop
function Component({ items }) {
  for (let item of items) {
    const [state, setState] = useState(item);  // Error!
  }
}

// ✗ Bad: Hook in nested function
function Component() {
  function handleClick() {
    const [state, setState] = useState(0);  // Error!
  }
}

// ✓ Good: Top level only
function Component({ condition }) {
  const [state, setState] = useState(0);
  
  if (condition) {
    // Use the state here
  }
}
```

### Rule 2: Only Call Hooks from React Functions

**Only call hooks from:**

- React function components
- Custom hooks

```jsx
// ✗ Bad: Regular function
function regularFunction() {
  const [state, setState] = useState(0);  // Error!
}

// ✗ Bad: Class component
class Component extends React.Component {
  method() {
    const [state, setState] = useState(0);  // Error!
  }
}

// ✓ Good: Function component
function Component() {
  const [state, setState] = useState(0);
  return <div>{state}</div>;
}

// ✓ Good: Custom hook
function useCustomHook() {
  const [state, setState] = useState(0);
  return state;
}
```

## Why These Rules Exist

### The Internal Hooks Implementation

React stores hooks in a **linked list** per component. The order matters.

```jsx
function Component() {
  const [name, setName] = useState('');     // Hook 1
  const [age, setAge] = useState(0);        // Hook 2
  const [email, setEmail] = useState('');   // Hook 3
}

// Internally, React stores:
// fiber.memoizedState = {
//   memoizedState: '',           // name
//   next: {
//     memoizedState: 0,          // age
//     next: {
//       memoizedState: '',       // email
//       next: null
//     }
//   }
// }
```

### What Happens with Conditional Hooks

```jsx
function BrokenComponent({ condition }) {
  const [name, setName] = useState('');  // Hook 1
  
  if (condition) {
    const [age, setAge] = useState(0);   // Hook 2 (sometimes)
  }
  
  const [email, setEmail] = useState('');  // Hook 2 or 3?
}

// First render (condition = true):
// Hook 1: name
// Hook 2: age
// Hook 3: email

// Second render (condition = false):
// Hook 1: name
// Hook 2: email  ← React expects 'age' but gets 'email'!
// Hook 3: ???    ← React expects 'email' but it's missing!

// Result: State corruption!
```

### Visual Explanation

```jsx
// Correct: Same order every render
Render 1:  useState → useEffect → useState → useCallback
Render 2:  useState → useEffect → useState → useCallback
Render 3:  useState → useEffect → useState → useCallback
           ✓ Matches!

// Broken: Different order
Render 1:  useState → useEffect → useState → useCallback
Render 2:  useState → useState → useCallback
           ✗ Mismatch! useEffect skipped, everything shifts
```

## Common Violations and Fixes

### Violation 1: Conditional Hook Call

```jsx
// ✗ Bad
function Component({ user }) {
  if (user) {
    const [name, setName] = useState(user.name);
  }
}

// ✓ Fix: Move condition inside
function Component({ user }) {
  const [name, setName] = useState(user?.name ?? '');
  
  if (!user) return null;
  
  return <div>{name}</div>;
}

// ✓ Alternative: Early return after hooks
function Component({ user }) {
  const [name, setName] = useState('');
  
  useEffect(() => {
    if (user) {
      setName(user.name);
    }
  }, [user]);
  
  if (!user) return null;
  
  return <div>{name}</div>;
}
```

### Violation 2: Hook in Loop

```jsx
// ✗ Bad
function Component({ items }) {
  return items.map(item => {
    const [selected, setSelected] = useState(false);  // Error!
    return <div>{item}</div>;
  });
}

// ✓ Fix: Create separate component
function Item({ item }) {
  const [selected, setSelected] = useState(false);
  return <div>{item}</div>;
}

function Component({ items }) {
  return items.map(item => <Item key={item.id} item={item} />);
}

// ✓ Alternative: Single state for all items
function Component({ items }) {
  const [selectedItems, setSelectedItems] = useState(new Set());
  
  return items.map(item => (
    <div key={item.id}>
      {item}
      {selectedItems.has(item.id) && '✓'}
    </div>
  ));
}
```

### Violation 3: Hook in Callback

```jsx
// ✗ Bad
function Component() {
  const handleClick = () => {
    const [count, setCount] = useState(0);  // Error!
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// ✓ Fix: Move hook to component level
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(c => c + 1);
  };
  
  return <button onClick={handleClick}>Click {count}</button>;
}
```

### Violation 4: Hook After Early Return

```jsx
// ✗ Bad
function Component({ data }) {
  if (!data) return null;  // Early return
  
  const [state, setState] = useState(0);  // Hook after return!
}

// ✓ Fix: Hooks before return
function Component({ data }) {
  const [state, setState] = useState(0);
  
  if (!data) return null;
  
  return <div>{state}</div>;
}
```

### Violation 5: Conditional useEffect

```jsx
// ✗ Bad
function Component({ shouldFetch }) {
  if (shouldFetch) {
    useEffect(() => {
      fetchData();
    }, []);
  }
}

// ✓ Fix: Condition inside hook
function Component({ shouldFetch }) {
  useEffect(() => {
    if (shouldFetch) {
      fetchData();
    }
  }, [shouldFetch]);
}
```

## Advanced Cases

### Case 1: Optional Features

```jsx
// Problem: Want to use hook only if feature enabled
// ✗ Bad
function Component({ enableAnalytics }) {
  if (enableAnalytics) {
    useAnalytics();  // Error!
  }
}

// ✓ Fix: Hook always called, logic inside
function useAnalytics(enabled) {
  useEffect(() => {
    if (!enabled) return;
    
    // Analytics logic
  }, [enabled]);
}

function Component({ enableAnalytics }) {
  useAnalytics(enableAnalytics);
}
```

### Case 2: Dynamic Number of Fields

```jsx
// Problem: Unknown number of form fields
// ✗ Bad
function Form({ fields }) {
  return fields.map(field => {
    const [value, setValue] = useState('');  // Error!
    return <input value={value} onChange={e => setValue(e.target.value)} />;
  });
}

// ✓ Fix 1: Component per field
function Field({ field }) {
  const [value, setValue] = useState('');
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

function Form({ fields }) {
  return fields.map(field => <Field key={field.id} field={field} />);
}

// ✓ Fix 2: Single state object
function Form({ fields }) {
  const [values, setValues] = useState({});
  
  return fields.map(field => (
    <input
      key={field.id}
      value={values[field.id] || ''}
      onChange={e => setValues(v => ({ ...v, [field.id]: e.target.value }))}
    />
  ));
}

// ✓ Fix 3: useReducer
function formReducer(state, action) {
  switch (action.type) {
    case 'update':
      return { ...state, [action.field]: action.value };
    default:
      return state;
  }
}

function Form({ fields }) {
  const [values, dispatch] = useReducer(formReducer, {});
  
  return fields.map(field => (
    <input
      key={field.id}
      value={values[field.id] || ''}
      onChange={e => dispatch({ 
        type: 'update', 
        field: field.id, 
        value: e.target.value 
      })}
    />
  ));
}
```

### Case 3: Hooks in Higher-Order Components

```jsx
// ✗ Bad: Hook in HOC function
function withData(Component) {
  const data = useFetch('/api/data');  // Error! Not in component
  return <Component data={data} />;
}

// ✓ Fix: Hook in wrapper component
function withData(Component) {
  return function WrappedComponent(props) {
    const data = useFetch('/api/data');  // ✓ In component
    return <Component {...props} data={data} />;
  };
}

// Usage
const EnhancedComponent = withData(MyComponent);
```

### Case 4: Hooks in Class Methods

```jsx
// ✗ Bad: Can't use hooks in class components
class Component extends React.Component {
  componentDidMount() {
    const [state, setState] = useState(0);  // Error!
  }
}

// ✓ Fix: Convert to function component
function Component() {
  const [state, setState] = useState(0);
  
  useEffect(() => {
    // componentDidMount logic
  }, []);
  
  return <div>{state}</div>;
}

// ✓ Alternative: Wrapper component with hook
function withHook(ClassComponent) {
  return function Wrapper(props) {
    const hookValue = useCustomHook();
    return <ClassComponent {...props} hookValue={hookValue} />;
  };
}
```

## ESLint Plugin

### Installation

```bash
npm install eslint-plugin-react-hooks --save-dev
```

### Configuration

```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

### What It Catches

```jsx
// Catches conditional hooks
if (condition) {
  const [state, setState] = useState(0);
  // Error: React Hook "useState" is called conditionally
}

// Catches hooks in loops
for (let i = 0; i < 10; i++) {
  const [state, setState] = useState(0);
  // Error: React Hook "useState" may be executed more than once
}

// Catches missing dependencies
useEffect(() => {
  console.log(count);
}, []);
// Warning: React Hook useEffect has a missing dependency: 'count'
```

## Why the Linked List?

### Alternative: Named Hooks?

```jsx
// Hypothetical API that doesn't exist
function Component() {
  const [name, setName] = useState('name', '');
  const [age, setAge] = useState('age', 0);
}

// Why React doesn't do this:
// 1. More verbose
// 2. Namespace collisions
// 3. Harder to optimize
// 4. Doesn't solve the real problem (render consistency)
```

### How Linked List Works

```jsx
let currentFiber = null;
let currentHook = null;

function useState(initialValue) {
  // First render: Create new hook
  if (!currentHook) {
    currentHook = {
      memoizedState: initialValue,
      next: null
    };
    currentFiber.memoizedState = currentHook;
  }
  
  // Subsequent renders: Read existing hook
  const value = currentHook.memoizedState;
  
  const setState = (newValue) => {
    currentHook.memoizedState = newValue;
    scheduleRerender();
  };
  
  // Move to next hook for next useState call
  currentHook = currentHook.next;
  
  return [value, setState];
}
```

## Common Misconceptions

### Misconception 1: "Hooks are magic"

```jsx
// Not magic! Just maintaining order
function Component() {
  // Call 1: React creates/reads hook 1
  const [a, setA] = useState(0);
  
  // Call 2: React creates/reads hook 2
  const [b, setB] = useState(0);
  
  // Call 3: React creates/reads hook 3
  useEffect(() => {}, []);
}
```

### Misconception 2: "Rules are arbitrary"

```jsx
// Rules exist because of implementation
// They could be different if React used different internals
// But current approach is optimal for most cases
```

### Misconception 3: "Can't have dynamic behavior"

```jsx
// You CAN have dynamic behavior, just not dynamic hooks

// ✓ Dynamic values
function Component({ items }) {
  const [selected, setSelected] = useState(new Set());
  // Can select any number of items dynamically
}

// ✓ Dynamic effects
function Component({ shouldLog }) {
  useEffect(() => {
    if (shouldLog) {  // Dynamic behavior inside hook
      console.log('Logging');
    }
  }, [shouldLog]);
}

// ✗ Dynamic hooks (number changes)
function Component({ count }) {
  for (let i = 0; i < count; i++) {
    useState(0);  // Number of hooks changes!
  }
}
```

## Edge Cases

### Edge Case 1: Hooks in Render Props

```jsx
// ✗ Bad
function Component() {
  return (
    <DataProvider render={() => {
      const [state, setState] = useState(0);  // Error!
      return <div>{state}</div>;
    }} />
  );
}

// ✓ Fix: Component wrapper
function Inner() {
  const [state, setState] = useState(0);
  return <div>{state}</div>;
}

function Component() {
  return <DataProvider render={() => <Inner />} />;
}
```

### Edge Case 2: Hooks in Array/Object Methods

```jsx
// ✗ Bad
function Component({ items }) {
  const processed = items.map(item => {
    const [value, setValue] = useState(item);  // Error!
    return value;
  });
}

// ✓ Fix: Hook outside map
function Component({ items }) {
  const [values, setValues] = useState(items);
  
  const processed = values.map(value => value * 2);
}
```

### Edge Case 3: Hooks in Async Functions

```jsx
// ✗ Bad
function Component() {
  const handleClick = async () => {
    await fetchData();
    const [state, setState] = useState(0);  // Error!
  };
}

// ✓ Fix: Hook at component level
function Component() {
  const [state, setState] = useState(0);
  
  const handleClick = async () => {
    await fetchData();
    setState(0);
  };
}
```

## Testing for Rule Violations

```jsx
// Test that component follows rules
import { renderHook } from '@testing-library/react';

test('hook follows rules', () => {
  const { result, rerender } = renderHook(
    ({ condition }) => {
      const [count, setCount] = useState(0);
      
      // Should not throw even if condition changes
      if (condition) {
        setCount(1);
      }
      
      return count;
    },
    { initialProps: { condition: true } }
  );
  
  expect(result.current).toBe(1);
  
  rerender({ condition: false });
  expect(result.current).toBe(1);  // Still works
});
```

## Migration Strategies

### From Class to Hooks

```jsx
// Class with conditional logic
class Component extends React.Component {
  componentDidMount() {
    if (this.props.shouldFetch) {
      this.fetchData();
    }
  }
}

// Function with hooks (condition inside)
function Component({ shouldFetch }) {
  useEffect(() => {
    if (shouldFetch) {
      fetchData();
    }
  }, [shouldFetch]);
}
```

## Resources

- [React Docs: Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks)
- [eslint-plugin-react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks)
- [Why Do Hooks Rely on Call Order?](https://overreacted.io/why-do-hooks-rely-on-call-order/)
- [Deep Dive: How Hooks Work](https://www.netlify.com/blog/2019/03/11/deep-dive-how-do-react-hooks-really-work/)
