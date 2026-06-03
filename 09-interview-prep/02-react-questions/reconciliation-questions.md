# React Reconciliation Interview Questions

## Overview

Reconciliation questions test your understanding of React's rendering engine - how the Virtual DOM works, the diffing algorithm, and how React efficiently updates the real DOM. This is fundamental knowledge that separates junior from senior developers.

---

## Core Reconciliation Questions

### Q1: What is the Virtual DOM and how does it work?

**Answer:**

The Virtual DOM is a lightweight JavaScript representation of the real DOM. React uses it to optimize updates by minimizing expensive DOM manipulations.

**How It Works:**

1. **Initial Render:**
   - React creates a Virtual DOM tree
   - Converts it to real DOM
   - Displays on screen

2. **When State Changes:**
   - React creates a new Virtual DOM tree
   - Compares it with the previous tree (diffing)
   - Calculates minimal changes needed
   - Applies only those changes to real DOM (reconciliation)

**Code Example:**

```javascript
// This JSX:
function App() {
  return (
    <div className="container">
      <h1>Hello World</h1>
      <p>Welcome</p>
    </div>
  );
}

// Creates this Virtual DOM (simplified):
{
  type: 'div',
  props: {
    className: 'container',
    children: [
      {
        type: 'h1',
        props: {
          children: 'Hello World'
        }
      },
      {
        type: 'p',
        props: {
          children: 'Welcome'
        }
      }
    ]
  }
}

// React elements are just objects:
const element = React.createElement(
  'div',
  { className: 'container' },
  React.createElement('h1', null, 'Hello World'),
  React.createElement('p', null, 'Welcome')
);
```

**Virtual DOM vs Real DOM:**

```javascript
// Real DOM operations are slow:
const div = document.createElement('div'); // Expensive
div.className = 'container';
div.innerHTML = '<h1>Hello</h1>'; // Triggers reflow/repaint
document.body.appendChild(div); // More reflow/repaint

// Virtual DOM operations are fast:
const vNode = {
  type: 'div',
  props: { className: 'container' }
}; // Just a JavaScript object!

// React batches and optimizes DOM updates
```

**Benefits:**

1. **Performance:** Batch updates, minimize DOM operations
2. **Cross-platform:** Same code for web, mobile (React Native), VR
3. **Declarative:** Describe what you want, React figures out how
4. **Predictable:** Pure functions, easier to reason about

**Example Showing Efficiency:**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <h1>Static Title</h1>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// When count changes:
// ❌ React DOESN'T recreate the entire div
// ✅ React ONLY updates the <p> text node
// The <h1> and <button> are untouched!
```

**What Interviewers Look For:**
- Understanding that Virtual DOM is just JavaScript objects
- Knowledge of why it's faster than direct DOM manipulation
- Awareness of the reconciliation process
- Understanding when React actually updates the DOM

**Follow-up:** Is Virtual DOM always faster than direct DOM manipulation?

**Answer:** No! For simple, targeted updates, direct DOM manipulation can be faster. Virtual DOM shines when you have complex UIs with many updates. The real benefit is the developer experience - you write declarative code and React handles optimization.

---

### Q2: Explain React's diffing algorithm

**Answer:**

React's diffing algorithm compares two Virtual DOM trees to determine what changed. It's optimized with heuristics that reduce complexity from O(n³) to O(n).

**Three Key Heuristics:**

**1. Different Types = Replace Everything:**

```javascript
// Before:
<div>
  <Counter />
</div>

// After:
<span>
  <Counter />
</span>

// Result: React destroys the entire div and Counter
// Counter unmounts and remounts (loses state!)
// Even though Counter didn't change, its parent type changed
```

**2. Same Type = Update Props:**

```javascript
// Before:
<div className="old" title="before">
  Content
</div>

// After:
<div className="new" title="after">
  Content
</div>

// Result: React keeps the DOM node, only updates className and title
// No unmount/remount, preserves DOM node and component state
```

**3. Children with Keys = Reorder Efficiently:**

```javascript
// Without keys (INEFFICIENT):
// Before: [A, B, C]
// After:  [A, B, C, D]
// React: Update A, Update B, Update C, Insert D

// Before: [A, B, C]
// After:  [D, A, B, C]
// React: Update A→D, Update B→A, Update C→B, Insert C (WRONG!)

// With keys (EFFICIENT):
// Before: [<li key="a">A</li>, <li key="b">B</li>, <li key="c">C</li>]
// After:  [<li key="d">D</li>, <li key="a">A</li>, <li key="b">B</li>, <li key="c">C</li>]
// React: Recognizes A, B, C haven't changed, just inserts D at the start!
```

**Detailed Example:**

```javascript
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn React', done: false },
    { id: 2, text: 'Build project', done: false },
    { id: 3, text: 'Get hired', done: false }
  ]);
  
  // BAD: Using index as key
  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}>
          <input type="checkbox" checked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
  
  // When you delete item 2:
  // Before: [0:Learn, 1:Build, 2:Get hired]
  // After:  [0:Learn, 1:Get hired]
  // React thinks: Keep 0, Keep 1 (but UPDATE text), Delete 2
  // Result: Wrong item gets deleted in UI! Checkbox states mixed up!
  
  // GOOD: Using stable ID as key
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input type="checkbox" checked={todo.done} />
          {todo.text}
        </li>
      ))}
    </ul>
  );
  
  // When you delete item 2:
  // Before: [1:Learn, 2:Build, 3:Get hired]
  // After:  [1:Learn, 3:Get hired]
  // React thinks: Keep 1, Delete 2, Keep 3
  // Result: Correct item deleted! Checkbox states preserved!
}
```

**Algorithm Visualization:**

```javascript
// Step-by-step diffing process

// Tree 1 (Old):
<div>
  <h1>Title</h1>
  <p>Paragraph</p>
</div>

// Tree 2 (New):
<div>
  <h1>New Title</h1>
  <span>Span</span>
</div>

// Diffing Process:
// 1. Compare root: div === div ✓ (keep, check children)
// 2. Compare child 1: h1 === h1 ✓ (update text content)
// 3. Compare child 2: p !== span ✗ (destroy p, create span)

// DOM Operations:
// 1. textNode.nodeValue = "New Title" (update)
// 2. div.removeChild(p) (delete)
// 3. div.appendChild(span) (insert)
```

**Performance Implications:**

```javascript
// Scenario 1: List reorder without keys
function BadList({ items }) {
  return (
    <ul>
      {items.map((item, i) => (
        <li key={i}>
          <ExpensiveComponent data={item} />
        </li>
      ))}
    </ul>
  );
}
// When items reorder: ALL ExpensiveComponents remount! 😱

// Scenario 2: List reorder with keys
function GoodList({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          <ExpensiveComponent data={item} />
        </li>
      ))}
    </ul>
  );
}
// When items reorder: Components just move, no remount! 🎉
```

**What Interviewers Look For:**
- Understanding of the three heuristics
- Knowledge of why keys are important
- Awareness of type changes causing remounts
- Understanding of O(n) complexity

**Red Flags:**
- Using index as key for dynamic lists
- Not knowing that type changes cause remounts
- Thinking React always re-renders everything

---

### Q3: Why are keys important and what makes a good key?

**Answer:**

Keys help React identify which items have changed, been added, or removed. They're crucial for list performance and correctness.

**Why Keys Matter:**

**1. Identity Across Renders:**

```javascript
// Without keys, React uses position to match elements
// Before: [A, B, C]
// After:  [C, A, B]
// React: 0→C (update), 1→A (update), 2→B (update) - ALL UPDATE!

// With keys, React uses identity to match elements
// Before: [key=a:A, key=b:B, key=c:C]
// After:  [key=c:C, key=a:A, key=b:B]
// React: Just reorder! No updates needed!
```

**2. State Preservation:**

```javascript
function TodoItem({ todo }) {
  const [editing, setEditing] = useState(false);
  const [draft, setDraft] = useState(todo.text);
  
  return editing ? (
    <input value={draft} onChange={e => setDraft(e.target.value)} />
  ) : (
    <div onClick={() => setEditing(true)}>{todo.text}</div>
  );
}

function TodoList({ todos }) {
  // BAD: Index as key
  return (
    <div>
      {todos.map((todo, index) => (
        <TodoItem key={index} todo={todo} />
      ))}
    </div>
  );
  
  // Problem: If you delete item while editing another,
  // the editing state moves to wrong item!
  
  // GOOD: Stable ID as key
  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </div>
  );
  
  // Now editing state stays with correct item!
}
```

**What Makes a Good Key:**

**1. Stable (doesn't change between renders):**

```javascript
// ❌ BAD: Random key
{items.map(item => (
  <div key={Math.random()}>{item}</div>
))}
// New key every render = everything remounts!

// ✅ GOOD: Stable ID
{items.map(item => (
  <div key={item.id}>{item}</div>
))}
```

**2. Unique (among siblings):**

```javascript
// ❌ BAD: Non-unique
{items.map(item => (
  <div key={item.category}>{item.name}</div>
))}
// Multiple items with same category = duplicate keys!

// ✅ GOOD: Unique ID
{items.map(item => (
  <div key={item.id}>{item.name}</div>
))}
```

**3. Predictable (same item = same key):**

```javascript
// ❌ BAD: Composite key with volatile data
{items.map(item => (
  <div key={`${item.id}-${item.timestamp}`}>{item.name}</div>
))}
// timestamp changes = new key = remount!

// ✅ GOOD: Just the ID
{items.map(item => (
  <div key={item.id}>{item.name}</div>
))}
```

**Common Key Patterns:**

```javascript
// Pattern 1: Database ID (best)
{users.map(user => (
  <UserCard key={user.id} user={user} />
))}

// Pattern 2: Compound key (when no single unique field)
{items.map(item => (
  <Item key={`${item.categoryId}-${item.productId}`} item={item} />
))}

// Pattern 3: Generated ID for static content
const ITEMS = [
  { id: 'home', label: 'Home' },
  { id: 'about', label: 'About' },
  { id: 'contact', label: 'Contact' }
];

{ITEMS.map(item => (
  <NavItem key={item.id} {...item} />
))}

// Pattern 4: Index ONLY for static lists that never reorder
const COLORS = ['red', 'green', 'blue'];

{COLORS.map((color, index) => (
  <div key={index} style={{ background: color }} />
))}
// OK because list never changes

// Pattern 5: Content hash (last resort)
import hash from 'object-hash';

{items.map(item => (
  <div key={hash(item)}>{item.name}</div>
))}
// Only if no stable ID available
```

**When Index is Acceptable:**

```javascript
// ✅ OK: Static list, never changes
const fruits = ['Apple', 'Banana', 'Cherry'];
{fruits.map((fruit, i) => <div key={i}>{fruit}</div>)}

// ✅ OK: List only grows at the end, never reorders
function ChatMessages({ messages }) {
  return (
    <div>
      {messages.map((msg, i) => (
        <div key={i}>{msg.text}</div>
      ))}
    </div>
  );
  // If new messages only append, index is fine
}

// ❌ BAD: List can reorder, items can be removed
function TodoList({ todos }) {
  return (
    <div>
      {todos.map((todo, i) => (
        <TodoItem key={i} todo={todo} />
      ))}
    </div>
  );
  // Don't do this!
}
```

**Real-World Bug Example:**

```javascript
// Shopping cart with removable items
function ShoppingCart() {
  const [items, setItems] = useState([
    { id: 1, name: 'Laptop', quantity: 1 },
    { id: 2, name: 'Mouse', quantity: 2 },
    { id: 3, name: 'Keyboard', quantity: 1 }
  ]);
  
  const removeItem = (indexToRemove) => {
    setItems(items.filter((_, i) => i !== indexToRemove));
  };
  
  // BUG: Using index as key
  return (
    <div>
      {items.map((item, index) => (
        <div key={index}>
          {item.name} - Quantity: {item.quantity}
          <input type="number" defaultValue={item.quantity} />
          <button onClick={() => removeItem(index)}>Remove</button>
        </div>
      ))}
    </div>
  );
  
  // Bug reproduction:
  // 1. Change Laptop quantity to 5 in the input
  // 2. Remove Mouse
  // 3. Laptop still shows quantity 1, but input shows 5 (state is on wrong item!)
  
  // FIX: Use stable ID
  return (
    <div>
      {items.map((item, index) => (
        <div key={item.id}>
          {item.name} - Quantity: {item.quantity}
          <input type="number" defaultValue={item.quantity} />
          <button onClick={() => removeItem(index)}>Remove</button>
        </div>
      ))}
    </div>
  );
}
```

**What Interviewers Look For:**
- Understanding that keys are about identity, not position
- Knowledge of when index is acceptable vs dangerous
- Awareness of state preservation issues
- Ability to choose appropriate keys

**Red Flags:**
- "Keys don't really matter"
- Always using index as key
- Using random values as keys
- Not knowing keys are for siblings, not global

---

### Q4: What's the difference between React elements and components?

**Answer:**

This is a fundamental distinction that many developers confuse.

**React Element:**
- Plain JavaScript object describing what to render
- Immutable
- Cheap to create
- Created by JSX or React.createElement()
- The result of rendering a component

**React Component:**
- Function or class that returns elements
- Can have state and logic
- Reusable blueprint for elements
- Gets called/instantiated by React

**Visual Comparison:**

```javascript
// Component (blueprint):
function Button({ label }) {
  return <button>{label}</button>;
}

// Element (instance):
const buttonElement = <Button label="Click me" />;

// What the element actually is:
const buttonElement = {
  type: Button,
  props: {
    label: 'Click me'
  },
  key: null,
  ref: null
};
```

**Detailed Examples:**

```javascript
// 1. Element from JSX
const element = <div className="container">Hello</div>;

// What it becomes:
const element = {
  type: 'div',
  props: {
    className: 'container',
    children: 'Hello'
  }
};

// 2. Element from component
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}

const greetingElement = <Greeting name="Alice" />;

// What it becomes:
const greetingElement = {
  type: Greeting, // Reference to the function
  props: {
    name: 'Alice'
  }
};

// 3. Element tree
const app = (
  <div>
    <Header />
    <Content />
    <Footer />
  </div>
);

// Represents this structure:
const app = {
  type: 'div',
  props: {
    children: [
      { type: Header, props: {} },
      { type: Content, props: {} },
      { type: Footer, props: {} }
    ]
  }
};
```

**Key Differences:**

```javascript
// Component: Can be instantiated multiple times
function UserCard({ user }) {
  return <div>{user.name}</div>;
}

// Each creates a separate element:
<UserCard user={user1} />
<UserCard user={user2} />
<UserCard user={user3} />

// Elements: Immutable objects
const element = <div>Hello</div>;
element.props.children = 'Goodbye'; // ❌ Won't work, it's frozen

// Component: Can have state
function Counter() {
  const [count, setCount] = useState(0); // ✅ State
  return <div>{count}</div>;
}

// Element: Just data, no state
const element = <div>{count}</div>; // Just an object
```

**Type Property Significance:**

```javascript
// For DOM elements, type is a string:
const divElement = <div />;
divElement.type === 'div'; // true

// For components, type is the function/class:
function MyComponent() {
  return <div />;
}

const myElement = <MyComponent />;
myElement.type === MyComponent; // true

// React uses this to decide how to render:
if (typeof element.type === 'string') {
  // It's a DOM element, create real DOM node
  document.createElement(element.type);
} else if (typeof element.type === 'function') {
  // It's a component, call it to get its elements
  const childElements = element.type(element.props);
}
```

**Rendering Process:**

```javascript
// You write:
function App() {
  return (
    <div>
      <Greeting name="World" />
    </div>
  );
}

// React creates element tree:
// Step 1: Call App()
{
  type: 'div',
  props: {
    children: {
      type: Greeting, // Component reference
      props: { name: 'World' }
    }
  }
}

// Step 2: Call Greeting({ name: 'World' })
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}

// Step 3: Final element tree (all components resolved):
{
  type: 'div',
  props: {
    children: {
      type: 'h1',
      props: {
        children: ['Hello, ', 'World']
      }
    }
  }
}

// Step 4: Create real DOM from final elements
```

**Practical Implications:**

```javascript
// 1. You can store elements in variables
const header = <h1>Title</h1>;
const content = <p>Content</p>;

function Page() {
  return (
    <div>
      {header}
      {content}
    </div>
  );
}

// 2. You can pass elements as props (render props)
function Layout({ header, content }) {
  return (
    <div>
      <div className="header">{header}</div>
      <div className="content">{content}</div>
    </div>
  );
}

<Layout
  header={<h1>My App</h1>}
  content={<p>Welcome</p>}
/>

// 3. You can clone and modify elements
const originalElement = <Button color="blue">Click</Button>;
const modifiedElement = React.cloneElement(
  originalElement,
  { color: 'red' }
);

// 4. Components create new elements each render
function Parent() {
  // New element created every render
  return <Child />; // Different object each time
}

// But element types are compared:
<Child /> === <Child /> // false (different objects)
Child === Child // true (same component reference)
```

**What Interviewers Look For:**
- Clear distinction between elements and components
- Understanding that elements are just objects
- Knowledge of the rendering process
- Awareness of immutability

**Follow-up:** Can you explain React.createElement()?

```javascript
// JSX is syntactic sugar for createElement:
<div className="container">
  <h1>Hello</h1>
</div>

// Compiles to:
React.createElement(
  'div',
  { className: 'container' },
  React.createElement('h1', null, 'Hello')
);

// Returns:
{
  type: 'div',
  props: {
    className: 'container',
    children: {
      type: 'h1',
      props: {
        children: 'Hello'
      }
    }
  }
}
```

---

### Q5: How does React handle updates to nested components?

**Answer:**

React's update process follows a top-down approach, but uses optimizations to skip unnecessary work.

**Update Flow:**

```javascript
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <Header />
      <Counter count={count} />
      <Footer />
    </div>
  );
}

// When count changes:
// 1. App re-renders (state changed here)
// 2. React creates new element tree
// 3. Diffing algorithm compares trees:
//    - Header: Same type, same props → SKIP
//    - Counter: Same type, different props → UPDATE
//    - Footer: Same type, same props → SKIP
```

**Detailed Example:**

```javascript
function GrandParent() {
  const [count, setCount] = useState(0);
  console.log('GrandParent render');
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <Parent count={count} />
    </div>
  );
}

function Parent({ count }) {
  console.log('Parent render');
  return (
    <div>
      <Child count={count} />
      <Sibling />
    </div>
  );
}

function Child({ count }) {
  console.log('Child render');
  return <div>Count: {count}</div>;
}

function Sibling() {
  console.log('Sibling render');
  return <div>I am a sibling</div>;
}

// When count updates:
// Console output:
// "GrandParent render"  ← State changed here
// "Parent render"       ← Props changed (count)
// "Child render"        ← Props changed (count)
// "Sibling render"      ← Props didn't change, but still renders!
```

**Why Sibling Renders:**

React's default behavior is to re-render ALL children when a parent renders, even if their props didn't change. This is usually fine because:

1. **Virtual DOM comparison is fast:** Creating new elements is cheap
2. **Only changed DOM nodes update:** Even if component renders, DOM might not change
3. **Safety over speed:** Ensures consistency

**Optimization Techniques:**

**1. React.memo (Prevent Unnecessary Renders):**

```javascript
// Without memo: Sibling renders every time Parent renders
function Sibling() {
  console.log('Sibling render');
  return <div>Static content</div>;
}

// With memo: Sibling only renders when its props change
const Sibling = React.memo(function Sibling() {
  console.log('Sibling render');
  return <div>Static content</div>;
});

// Now when count updates:
// "GrandParent render"
// "Parent render"
// "Child render"
// Sibling NOT rendered! 🎉
```

**2. Component Composition (Structural Optimization):**

```javascript
// BAD: Everything re-renders when count changes
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <ExpensiveHeader />
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <ExpensiveFooter />
    </div>
  );
}

// GOOD: Extract stateful part
function App() {
  return (
    <div>
      <ExpensiveHeader />
      <Counter />
      <ExpensiveFooter />
    </div>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Now Header and Footer never re-render!
```

**3. Children Prop Pattern:**

```javascript
// State at top, but children passed in
function Container({ children }) {
  const [theme, setTheme] = useState('light');
  
  return (
    <div className={theme}>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
      {children}
    </div>
  );
}

// Usage:
<Container>
  <ExpensiveComponent />
</Container>

// When theme changes, ExpensiveComponent doesn't re-render!
// Because children prop is the same element object
```

**4. useMemo for Expensive Children:**

```javascript
function Parent() {
  const [count, setCount] = useState(0);
  const [filter, setFilter] = useState('');
  
  // Expensive component memoized
  const expensiveChild = useMemo(() => {
    return <ExpensiveList filter={filter} />;
  }, [filter]); // Only recreate when filter changes
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      {expensiveChild}
    </div>
  );
}

// When count changes, ExpensiveList doesn't re-render
```

**Update Propagation Example:**

```javascript
// Complex nested structure
function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  
  return (
    <Dashboard>
      <Sidebar user={user} />
      <Content>
        <Header user={user} />
        <MainPanel>
          <ArticleList />
          <UserProfile user={user} />
        </MainPanel>
      </Content>
    </Dashboard>
  );
}

// When user changes:
// 1. App renders (state changed)
// 2. Dashboard renders (child of App)
// 3. Sidebar renders (user prop changed)
// 4. Content renders (child of Dashboard)
// 5. Header renders (user prop changed)
// 6. MainPanel renders (child of Content)
// 7. ArticleList renders (no props, but parent rendered)
// 8. UserProfile renders (user prop changed)

// Optimized version:
const Dashboard = React.memo(function Dashboard({ children }) {
  return <div className="dashboard">{children}</div>;
});

const Sidebar = React.memo(function Sidebar({ user }) {
  return <aside>{user.name}</aside>;
});

const Content = React.memo(function Content({ children }) {
  return <main>{children}</main>;
});

const Header = React.memo(function Header({ user }) {
  return <header>Welcome, {user.name}</header>;
});

const MainPanel = React.memo(function MainPanel({ children }) {
  return <div className="panel">{children}</div>;
});

const ArticleList = React.memo(function ArticleList() {
  return <ul>{/* articles */}</ul>;
});

const UserProfile = React.memo(function UserProfile({ user }) {
  return <div>{user.name}'s profile</div>;
});

// Now when user changes:
// Only App, Sidebar, Header, and UserProfile render!
// Dashboard, Content, MainPanel, ArticleList skip!
```

**What Interviewers Look For:**
- Understanding of top-down rendering
- Knowledge of when components re-render
- Awareness of optimization techniques
- Understanding of React.memo and composition

**Red Flags:**
- "React only re-renders changed components" (false without optimization)
- Prematurely optimizing everything with memo
- Not understanding why components re-render

---

## Key Takeaways

**Essential Concepts:**
1. Virtual DOM is JavaScript objects representing UI
2. Diffing algorithm compares trees in O(n) time
3. Keys provide stable identity for list items
4. Elements are immutable objects; components are blueprints
5. React re-renders children by default (optimize when needed)

**Best Practices:**
- Use stable, unique keys for dynamic lists
- Understand when type changes cause remounts
- Optimize with React.memo for expensive components
- Use composition to limit re-render scope
- Don't use index as key for mutable lists

**Common Mistakes:**
- Using index as key for lists that can reorder
- Not understanding element vs component distinction
- Over-optimizing with memo everywhere
- Assuming React only re-renders changed components
- Creating new objects in props (breaks memoization)

**Interview Tips:**
- Explain Virtual DOM benefits and limitations
- Walk through diffing algorithm with examples
- Demonstrate key importance with bug scenarios
- Show when optimization is needed
- Explain reconciliation trade-offs

**Red Flags to Avoid:**
- "Virtual DOM is always faster"
- "Keys aren't important"
- "Index is fine as a key"
- Not knowing when components remount
- Unable to explain element creation
