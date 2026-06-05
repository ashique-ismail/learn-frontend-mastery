# React.memo

## The Idea

**In plain English:** React.memo is a way to tell React "don't bother re-drawing this part of the screen unless the information it displays has actually changed." Re-drawing (called re-rendering) is how React updates what you see, and skipping unnecessary re-draws makes your app faster.

**Real-world analogy:** Imagine a whiteboard in a classroom that shows a student's name and score. A helper's job is to erase and rewrite the board whenever the teacher calls out. With React.memo, the helper checks first — if the name and score are the same as before, they put the marker down and do nothing.

- The whiteboard = the component (the piece of UI on screen)
- The helper checking before rewriting = React.memo comparing the old and new props
- The name and score written on the board = the props (the data passed into the component)

---

## Overview

`React.memo` is a higher-order component (HOC) that memoizes a component, preventing unnecessary re-renders when props haven't changed. It performs a shallow comparison of props and only re-renders when props actually change. This guide covers when and how to use `React.memo` effectively, including its performance tradeoffs and best practices.

## Table of Contents

1. [What is React.memo](#what-is-react-memo)
2. [How React.memo Works](#how-react-memo-works)
3. [Shallow Comparison](#shallow-comparison)
4. [Custom Comparison Functions](#custom-comparison-functions)
5. [When to Use React.memo](#when-to-use-react-memo)
6. [When NOT to Use React.memo](#when-not-to-use-react-memo)
7. [Common Mistakes](#common-mistakes)
8. [Best Practices](#best-practices)
9. [Real-World Use Cases](#real-world-use-cases)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## What is React.memo

`React.memo` is a performance optimization tool that wraps a component and memoizes its output:

### Basic Usage

```typescript
import React from 'react';

// Without memo - re-renders on every parent render
function Greeting({ name }: { name: string }) {
  console.log('Greeting rendered');
  return <h1>Hello, {name}!</h1>;
}

// With memo - only re-renders when name changes
const GreetingMemo = React.memo(function Greeting({ 
  name 
}: { 
  name: string 
}) {
  console.log('Greeting rendered');
  return <h1>Hello, {name}!</h1>;
});

function App() {
  const [count, setCount] = React.useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Re-renders on every count change */}
      <Greeting name="Alice" />
      
      {/* Only renders once - name never changes */}
      <GreetingMemo name="Bob" />
    </div>
  );
}
```

### TypeScript with React.memo

```typescript
import React from 'react';

interface UserProps {
  name: string;
  age: number;
  onUpdate?: (name: string) => void;
}

// Type inference works automatically
const User = React.memo(function User({ name, age, onUpdate }: UserProps) {
  return (
    <div>
      <p>{name} is {age} years old</p>
      {onUpdate && <button onClick={() => onUpdate(name)}>Update</button>}
    </div>
  );
});

// Explicit typing
const UserExplicit: React.FC<UserProps> = React.memo(({ 
  name, 
  age, 
  onUpdate 
}) => {
  return (
    <div>
      <p>{name} is {age} years old</p>
      {onUpdate && <button onClick={() => onUpdate(name)}>Update</button>}
    </div>
  );
});

// With generic props
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

const List = React.memo(function List<T>({ 
  items, 
  renderItem 
}: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}) as <T>(props: ListProps<T>) => JSX.Element;
```

## How React.memo Works

`React.memo` caches the rendered output and reuses it when props are shallowly equal:

### Render Behavior

```typescript
import React, { useState } from 'react';

const ExpensiveComponent = React.memo(function ExpensiveComponent({ 
  value 
}: { 
  value: number 
}) {
  console.log('ExpensiveComponent rendered');
  
  // Simulate expensive computation
  let result = 0;
  for (let i = 0; i < 1000000; i++) {
    result += i;
  }
  
  return <div>Value: {value}, Result: {result}</div>;
});

function App() {
  const [count, setCount] = useState(0);
  const [value, setValue] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Increment Count: {count}
      </button>
      <button onClick={() => setValue(value + 1)}>
        Increment Value: {value}
      </button>
      
      {/* Only re-renders when value changes, not when count changes */}
      <ExpensiveComponent value={value} />
    </div>
  );
}
```

### Memoization Cache

```typescript
import React, { useState } from 'react';

const Child = React.memo(function Child({ 
  id, 
  data 
}: { 
  id: number; 
  data: string 
}) {
  console.log(`Child ${id} rendered with data: ${data}`);
  return <div>Child {id}: {data}</div>;
});

function Parent() {
  const [parentState, setParentState] = useState(0);
  
  return (
    <div>
      <button onClick={() => setParentState(parentState + 1)}>
        Parent State: {parentState}
      </button>
      
      {/* These won't re-render when parentState changes */}
      <Child id={1} data="unchanged" />
      <Child id={2} data="unchanged" />
      <Child id={3} data="unchanged" />
    </div>
  );
}
```

## Shallow Comparison

React.memo uses `Object.is` for shallow comparison of props:

### Primitive Props

```typescript
import React from 'react';

const Counter = React.memo(function Counter({ 
  count 
}: { 
  count: number 
}) {
  console.log('Counter rendered');
  return <div>Count: {count}</div>;
});

function App() {
  const [count, setCount] = React.useState(0);
  const [ignored, setIgnored] = React.useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Update Count
      </button>
      <button onClick={() => setIgnored(ignored + 1)}>
        Update Ignored: {ignored}
      </button>
      
      {/* Re-renders only when count changes */}
      <Counter count={count} />
    </div>
  );
}
```

### Object Props - The Problem

```typescript
import React, { useState } from 'react';

const UserProfile = React.memo(function UserProfile({ 
  user 
}: { 
  user: { name: string; age: number } 
}) {
  console.log('UserProfile rendered');
  return <div>{user.name} is {user.age} years old</div>;
});

function AppBad() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Re-renders every time! New object reference each render */}
      <UserProfile user={{ name: 'Alice', age: 30 }} />
    </div>
  );
}

// Solution: useMemo for object props
function AppGood() {
  const [count, setCount] = useState(0);
  
  const user = React.useMemo(
    () => ({ name: 'Alice', age: 30 }),
    [] // Never changes
  );
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Won't re-render - same object reference */}
      <UserProfile user={user} />
    </div>
  );
}
```

### Function Props - The Problem

```typescript
import React, { useState, useCallback } from 'react';

const Button = React.memo(function Button({ 
  onClick, 
  children 
}: { 
  onClick: () => void;
  children: React.ReactNode;
}) {
  console.log('Button rendered');
  return <button onClick={onClick}>{children}</button>;
});

function AppBad() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      
      {/* Re-renders every time! New function each render */}
      <Button onClick={() => setCount(count + 1)}>
        Increment
      </Button>
    </div>
  );
}

// Solution: useCallback for function props
function AppGood() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    setCount(prev => prev + 1);
  }, []); // Stable function reference
  
  return (
    <div>
      <p>Count: {count}</p>
      
      {/* Won't re-render - same function reference */}
      <Button onClick={handleClick}>
        Increment
      </Button>
    </div>
  );
}
```

### Array Props

```typescript
import React, { useState, useMemo } from 'react';

const List = React.memo(function List({ 
  items 
}: { 
  items: string[] 
}) {
  console.log('List rendered');
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
});

function AppBad() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Re-renders every time! New array reference */}
      <List items={['a', 'b', 'c']} />
    </div>
  );
}

function AppGood() {
  const [count, setCount] = useState(0);
  
  const items = useMemo(() => ['a', 'b', 'c'], []);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Won't re-render - same array reference */}
      <List items={items} />
    </div>
  );
}
```

## Custom Comparison Functions

Provide a custom comparison function for complex equality checks:

### Basic Custom Comparison

```typescript
import React from 'react';

interface User {
  id: number;
  name: string;
  metadata: { lastLogin: Date };
}

// Custom comparison - only check id and name, ignore metadata
const UserCard = React.memo(
  function UserCard({ user }: { user: User }) {
    console.log('UserCard rendered');
    return (
      <div>
        <h3>{user.name}</h3>
        <p>ID: {user.id}</p>
      </div>
    );
  },
  (prevProps, nextProps) => {
    // Return true if props are equal (skip render)
    // Return false if props are different (do render)
    return (
      prevProps.user.id === nextProps.user.id &&
      prevProps.user.name === nextProps.user.name
    );
  }
);
```

### Deep Comparison

```typescript
import React from 'react';
import isEqual from 'lodash/isEqual';

interface ComplexProps {
  config: {
    theme: string;
    settings: {
      notifications: boolean;
      autoSave: boolean;
    };
  };
}

const ComplexComponent = React.memo(
  function ComplexComponent({ config }: ComplexProps) {
    console.log('ComplexComponent rendered');
    return <div>Theme: {config.theme}</div>;
  },
  (prevProps, nextProps) => {
    // Deep equality check
    return isEqual(prevProps.config, nextProps.config);
  }
);

// Note: Deep comparisons can be expensive!
// Only use when necessary
```

### Selective Property Comparison

```typescript
import React from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
  inventory: number; // Changes frequently
  lastUpdated: Date; // Always different
}

const ProductCard = React.memo(
  function ProductCard({ product }: { product: Product }) {
    return (
      <div>
        <h3>{product.name}</h3>
        <p>Price: ${product.price}</p>
        <p>In stock: {product.inventory}</p>
      </div>
    );
  },
  (prevProps, nextProps) => {
    // Only re-render if displayed fields change
    const prev = prevProps.product;
    const next = nextProps.product;
    
    return (
      prev.id === next.id &&
      prev.name === next.name &&
      prev.price === next.price &&
      prev.inventory === next.inventory
      // Ignore lastUpdated
    );
  }
);
```

## When to Use React.memo

### Expensive Rendering

```typescript
import React from 'react';

const ExpensiveChart = React.memo(function ExpensiveChart({ 
  data 
}: { 
  data: number[] 
}) {
  console.log('Rendering expensive chart');
  
  // Expensive computation or rendering
  const processedData = data.map(value => {
    // Complex transformation
    let result = value;
    for (let i = 0; i < 10000; i++) {
      result = Math.sqrt(result + i);
    }
    return result;
  });
  
  return (
    <div>
      {processedData.map((value, index) => (
        <div key={index}>{value.toFixed(2)}</div>
      ))}
    </div>
  );
});
```

### Large Lists

```typescript
import React, { useState, useMemo } from 'react';

const ListItem = React.memo(function ListItem({ 
  item, 
  onSelect 
}: { 
  item: string;
  onSelect: (item: string) => void;
}) {
  console.log(`ListItem ${item} rendered`);
  return (
    <li onClick={() => onSelect(item)}>
      {item}
    </li>
  );
});

function LargeList() {
  const [selectedItem, setSelectedItem] = useState<string | null>(null);
  
  const items = useMemo(
    () => Array.from({ length: 1000 }, (_, i) => `Item ${i}`),
    []
  );
  
  const handleSelect = React.useCallback((item: string) => {
    setSelectedItem(item);
  }, []);
  
  return (
    <div>
      <p>Selected: {selectedItem}</p>
      <ul>
        {items.map(item => (
          <ListItem 
            key={item} 
            item={item} 
            onSelect={handleSelect}
          />
        ))}
      </ul>
    </div>
  );
}
```

### Pure Presentational Components

```typescript
import React from 'react';

interface CardProps {
  title: string;
  description: string;
  imageUrl: string;
}

const Card = React.memo(function Card({ 
  title, 
  description, 
  imageUrl 
}: CardProps) {
  return (
    <div className="card">
      <img src={imageUrl} alt={title} />
      <h3>{title}</h3>
      <p>{description}</p>
    </div>
  );
});

// Good candidate for memo:
// - Pure presentational
// - No internal state
// - Used multiple times
// - Props are stable
```

## When NOT to Use React.memo

### Simple Components

```typescript
import React from 'react';

// DON'T memo - too simple, overhead not worth it
const Label = React.memo(function Label({ text }: { text: string }) {
  return <span>{text}</span>;
});

// Just use regular component
function LabelBetter({ text }: { text: string }) {
  return <span>{text}</span>;
}
```

### Props Change Frequently

```typescript
import React, { useState, useEffect } from 'react';

// DON'T memo - timestamp changes every second
const Clock = React.memo(function Clock({ 
  timestamp 
}: { 
  timestamp: number 
}) {
  return <div>{new Date(timestamp).toLocaleTimeString()}</div>;
});

function App() {
  const [timestamp, setTimestamp] = useState(Date.now());
  
  useEffect(() => {
    const interval = setInterval(() => {
      setTimestamp(Date.now());
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);
  
  // memo provides no benefit - always re-renders
  return <Clock timestamp={timestamp} />;
}
```

### Context Consumers

```typescript
import React, { useContext, createContext, useState } from 'react';

const ThemeContext = createContext({ theme: 'light' });

// DON'T memo - re-renders when context changes anyway
const ThemedButton = React.memo(function ThemedButton() {
  const { theme } = useContext(ThemeContext);
  return <button className={theme}>Click me</button>;
});

// memo doesn't prevent context-triggered renders
```

## Common Mistakes

### Forgetting to Memoize Props

```typescript
import React, { useState } from 'react';

const Child = React.memo(function Child({ 
  data, 
  onUpdate 
}: {
  data: { value: string };
  onUpdate: () => void;
}) {
  console.log('Child rendered');
  return <div>{data.value}</div>;
});

// BAD: memo is useless here
function ParentBad() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* Both props are new references every render */}
      <Child 
        data={{ value: 'hello' }}
        onUpdate={() => console.log('update')}
      />
    </div>
  );
}

// GOOD: memoize props too
function ParentGood() {
  const [count, setCount] = useState(0);
  
  const data = React.useMemo(() => ({ value: 'hello' }), []);
  const onUpdate = React.useCallback(() => console.log('update'), []);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      <Child data={data} onUpdate={onUpdate} />
    </div>
  );
}
```

### Over-Memoization

```typescript
import React from 'react';

// BAD: Over-memoization adds overhead
function OverMemoized() {
  const [state, setState] = React.useState(0);
  
  // These memos are unnecessary
  const SimpleComponent = React.memo(() => <div>Simple</div>);
  const value = React.useMemo(() => state * 2, [state]);
  const handler = React.useCallback(() => setState(state + 1), [state]);
  
  return (
    <div>
      <SimpleComponent />
      <p>{value}</p>
      <button onClick={handler}>Click</button>
    </div>
  );
}

// GOOD: Only memoize when necessary
function Optimized() {
  const [state, setState] = React.useState(0);
  
  return (
    <div>
      <div>Simple</div>
      <p>{state * 2}</p>
      <button onClick={() => setState(state + 1)}>Click</button>
    </div>
  );
}
```

### Inline Objects/Arrays as Props

```typescript
import React from 'react';

const List = React.memo(function List({ 
  items 
}: { 
  items: string[] 
}) {
  return (
    <ul>
      {items.map(item => <li key={item}>{item}</li>)}
    </ul>
  );
});

// BAD: New array every render
function AppBad() {
  return <List items={['a', 'b', 'c']} />;
}

// GOOD: Stable reference
const ITEMS = ['a', 'b', 'c'];

function AppGood() {
  return <List items={ITEMS} />;
}
```

## Best Practices

### Measure Before Optimizing

```typescript
import React, { Profiler, useState } from 'react';

const ExpensiveComponent = React.memo(function ExpensiveComponent({ 
  value 
}: { 
  value: number 
}) {
  // Expensive rendering logic
  return <div>{value}</div>;
});

function App() {
  const [count, setCount] = useState(0);
  
  const onRender = (
    id: string,
    phase: 'mount' | 'update',
    actualDuration: number
  ) => {
    console.log(`${id} ${phase} took ${actualDuration}ms`);
  };
  
  return (
    <Profiler id="App" onRender={onRender}>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveComponent value={count} />
    </Profiler>
  );
}
```

### Combine with useMemo and useCallback

```typescript
import React, { useState, useMemo, useCallback } from 'react';

interface Item {
  id: number;
  name: string;
  value: number;
}

const ItemList = React.memo(function ItemList({
  items,
  onItemClick,
}: {
  items: Item[];
  onItemClick: (id: number) => void;
}) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onItemClick(item.id)}>
          {item.name}: {item.value}
        </li>
      ))}
    </ul>
  );
});

function App() {
  const [items, setItems] = useState<Item[]>([
    { id: 1, name: 'Item 1', value: 100 },
    { id: 2, name: 'Item 2', value: 200 },
  ]);
  const [filter, setFilter] = useState('');
  
  // Memoize filtered items
  const filteredItems = useMemo(() => {
    return items.filter(item => 
      item.name.toLowerCase().includes(filter.toLowerCase())
    );
  }, [items, filter]);
  
  // Memoize callback
  const handleItemClick = useCallback((id: number) => {
    console.log(`Clicked item ${id}`);
  }, []);
  
  return (
    <div>
      <input 
        value={filter}
        onChange={e => setFilter(e.target.value)}
        placeholder="Filter items"
      />
      <ItemList 
        items={filteredItems}
        onItemClick={handleItemClick}
      />
    </div>
  );
}
```

## Real-World Use Cases

### Data Table with Sorting

```typescript
import React, { useState, useMemo, useCallback } from 'react';

interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

const TableRow = React.memo(function TableRow({
  user,
  onEdit,
}: {
  user: User;
  onEdit: (id: number) => void;
}) {
  console.log(`TableRow ${user.id} rendered`);
  
  return (
    <tr>
      <td>{user.name}</td>
      <td>{user.email}</td>
      <td>{user.age}</td>
      <td>
        <button onClick={() => onEdit(user.id)}>Edit</button>
      </td>
    </tr>
  );
});

function DataTable({ users }: { users: User[] }) {
  const [sortKey, setSortKey] = useState<keyof User>('name');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('asc');
  
  const sortedUsers = useMemo(() => {
    return [...users].sort((a, b) => {
      const aVal = a[sortKey];
      const bVal = b[sortKey];
      
      if (aVal < bVal) return sortOrder === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortOrder === 'asc' ? 1 : -1;
      return 0;
    });
  }, [users, sortKey, sortOrder]);
  
  const handleEdit = useCallback((id: number) => {
    console.log(`Edit user ${id}`);
  }, []);
  
  return (
    <table>
      <thead>
        <tr>
          <th onClick={() => setSortKey('name')}>Name</th>
          <th onClick={() => setSortKey('email')}>Email</th>
          <th onClick={() => setSortKey('age')}>Age</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        {sortedUsers.map(user => (
          <TableRow 
            key={user.id}
            user={user}
            onEdit={handleEdit}
          />
        ))}
      </tbody>
    </table>
  );
}
```

### Form with Complex Validation

```typescript
import React, { useState, useMemo, useCallback } from 'react';

interface FormFieldProps {
  label: string;
  value: string;
  error?: string;
  onChange: (value: string) => void;
}

const FormField = React.memo(function FormField({
  label,
  value,
  error,
  onChange,
}: FormFieldProps) {
  console.log(`FormField ${label} rendered`);
  
  return (
    <div>
      <label>{label}</label>
      <input 
        value={value}
        onChange={e => onChange(e.target.value)}
      />
      {error && <span style={{ color: 'red' }}>{error}</span>}
    </div>
  );
});

function ComplexForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [age, setAge] = useState('');
  
  const nameError = useMemo(() => {
    if (name && name.length < 3) {
      return 'Name must be at least 3 characters';
    }
    return undefined;
  }, [name]);
  
  const emailError = useMemo(() => {
    if (email && !email.includes('@')) {
      return 'Invalid email';
    }
    return undefined;
  }, [email]);
  
  const ageError = useMemo(() => {
    const numAge = parseInt(age);
    if (age && (isNaN(numAge) || numAge < 0)) {
      return 'Invalid age';
    }
    return undefined;
  }, [age]);
  
  const handleNameChange = useCallback((value: string) => {
    setName(value);
  }, []);
  
  const handleEmailChange = useCallback((value: string) => {
    setEmail(value);
  }, []);
  
  const handleAgeChange = useCallback((value: string) => {
    setAge(value);
  }, []);
  
  return (
    <form>
      <FormField
        label="Name"
        value={name}
        error={nameError}
        onChange={handleNameChange}
      />
      <FormField
        label="Email"
        value={email}
        error={emailError}
        onChange={handleEmailChange}
      />
      <FormField
        label="Age"
        value={age}
        error={ageError}
        onChange={handleAgeChange}
      />
    </form>
  );
}
```

## Key Takeaways

1. **React.memo is for props**: Memoizes component based on prop comparison
2. **Shallow comparison by default**: Uses `Object.is` for each prop
3. **Custom comparisons available**: Provide second argument for complex checks
4. **Requires stable props**: Must memoize object/array/function props
5. **Use for expensive renders**: Best for computationally expensive components
6. **Don't over-optimize**: Adds overhead, measure before applying
7. **Combine with hooks**: Use with useMemo and useCallback for full benefit
8. **Not for context**: Doesn't prevent context-triggered re-renders
9. **Great for lists**: Prevents unnecessary re-renders of list items
10. **Profile first**: Use React DevTools Profiler to identify bottlenecks

## Resources

- [React.memo API Reference](https://react.dev/reference/react/memo)
- [When to useMemo and useCallback](https://kentcdodds.com/blog/usememo-and-usecallback)
- [React Performance Optimization](https://react.dev/learn/render-and-commit)
- [Profiling React Components](https://react.dev/learn/render-and-commit#profiling-components)
- [Before You memo](https://overreacted.io/before-you-memo/)
- [React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/)
