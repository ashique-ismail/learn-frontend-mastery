# Virtual DOM vs Real DOM vs Fiber

## Question Categories
This document covers one of the most common React interview questions, typically asked in these forms:
- "What is the Virtual DOM?"
- "Explain the difference between Virtual DOM and Real DOM"
- "What is React Fiber and how does it relate to Virtual DOM?"
- "Why is React faster than direct DOM manipulation?"

---

## Quick Answer (30 seconds)

**Virtual DOM** is React's lightweight JavaScript representation of the UI. When state changes, React:
1. Creates a new Virtual DOM tree
2. Diffs it with the previous tree (reconciliation)
3. Calculates minimal changes
4. Updates only what changed in the Real DOM

**Fiber** is React's reconciliation engine (since React 16) that implements Virtual DOM using an interruptible, priority-based algorithm. It's the *how* behind Virtual DOM diffing.

---

## The Real DOM

### What It Is
The **Document Object Model (DOM)** is the browser's live tree representation of HTML.

```javascript
// HTML
<div id="app">
  <h1>Hello</h1>
</div>

// Real DOM (simplified)
{
  tagName: 'DIV',
  attributes: { id: 'app' },
  childNodes: [
    {
      tagName: 'H1',
      childNodes: [
        { nodeType: TEXT_NODE, nodeValue: 'Hello' }
      ]
    }
  ]
}
```

### Why It's "Slow"

**Myth:** "DOM manipulation is slow"
**Reality:** Reading/writing to DOM is fast. What's slow is:

1. **Reflow (Layout)** - Browser recalculates positions/sizes
2. **Repaint** - Browser redraws pixels
3. **Style recalculation** - Browser computes CSS

**Example of inefficient updates:**
```javascript
// ❌ Bad: Causes 3 reflows
element.style.width = '100px';   // Reflow 1
element.style.height = '100px';  // Reflow 2
element.style.margin = '10px';   // Reflow 3

// ✅ Good: Causes 1 reflow (batched)
element.style.cssText = 'width: 100px; height: 100px; margin: 10px;';
```

**The real problem:** Without a strategy, updates are **frequent**, **scattered**, and **unoptimized**.

---

## The Virtual DOM

### What It Is

A **plain JavaScript object** that mirrors the DOM structure.

```jsx
// JSX
<div className="container">
  <h1>Count: {count}</h1>
  <button onClick={increment}>+</button>
</div>

// Virtual DOM (React element)
{
  type: 'div',
  props: {
    className: 'container',
    children: [
      {
        type: 'h1',
        props: {
          children: ['Count: ', count]
        }
      },
      {
        type: 'button',
        props: {
          onClick: increment,
          children: ['+']
        }
      }
    ]
  }
}
```

**Key point:** It's just data. No browser APIs, no rendering - pure JS.

### Why It's Useful

**1. Declarative programming**
```javascript
// Imperative (manual DOM)
const h1 = document.createElement('h1');
h1.textContent = 'Count: ' + count;
container.appendChild(h1);

if (count > 0) {
  h1.style.color = 'green';
} else {
  h1.style.color = 'red';
}

// Declarative (React)
return <h1 style={{ color: count > 0 ? 'green' : 'red' }}>Count: {count}</h1>;
```

**2. Batched & optimized updates**
React collects changes and applies them in one efficient pass.

**3. Platform agnostic**
Virtual DOM can target web (ReactDOM), mobile (React Native), VR, canvas, etc.

---

## The Diffing Algorithm

### Reconciliation

When state changes, React needs to figure out **what changed**. This is called **reconciliation**.

**Steps:**
1. Render new Virtual DOM tree
2. Compare (diff) with previous tree
3. Generate list of changes (called "effects")
4. Apply minimal updates to Real DOM

### Diffing Rules

React's diffing is **O(n)** instead of O(n³) thanks to two assumptions:

**1. Different types = completely different trees**
```jsx
// Old
<div>
  <Counter />
</div>

// New
<span>  {/* Different type! */}
  <Counter />
</span>

// React destroys <div> and <Counter>, creates new <span> and <Counter>
// Counter loses state because it's unmounted
```

**2. Keys indicate stable identity**
```jsx
// ❌ Without keys
<ul>
  <li>A</li>
  <li>B</li>
</ul>

// Add "C" at the beginning
<ul>
  <li>C</li>  {/* React thinks: A changed to C */}
  <li>A</li>  {/* React thinks: B changed to A */}
  <li>B</li>  {/* React thinks: new item */}
</ul>
// Result: 3 updates

// ✅ With keys
<ul>
  <li key="a">A</li>
  <li key="b">B</li>
</ul>

<ul>
  <li key="c">C</li>  {/* React: new item */}
  <li key="a">A</li>  {/* React: same, moved */}
  <li key="b">B</li>  {/* React: same, moved */}
</ul>
// Result: 1 insertion, 2 moves (cheaper)
```

### Example: Element Type Changed

```jsx
// Before
<div>
  <Counter count={5} />
</div>

// After
<span>
  <Counter count={5} />
</span>
```

**What happens:**
1. `div.type !== span.type` → different elements
2. React unmounts `<div>` and `<Counter>` (state lost!)
3. React mounts new `<span>` and new `<Counter>` (fresh state)

**Interview tip:** This is why `count` resets to 0 even though we passed `count={5}` - the `Counter` component is completely new.

---

### Example: Props Changed

```jsx
// Before
<div className="dark" title="Hello">
  <span>Content</span>
</div>

// After
<div className="light" title="Hi">
  <span>Content</span>
</div>
```

**What happens:**
1. `div.type === div.type` → same element, update props
2. React updates `className: "dark" → "light"`
3. React updates `title: "Hello" → "Hi"`
4. React recurses into children (no changes)
5. **DOM operations:** Only 2 attribute updates

---

### Example: List Reordering

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn React' },
    { id: 2, text: 'Build app' },
    { id: 3, text: 'Deploy' }
  ]);
  
  return (
    <ul>
      {todos.map(todo => (
        <Todo key={todo.id} {...todo} />
      ))}
    </ul>
  );
}

// User reorders: moves "Deploy" to top
// New order: [3, 1, 2]
```

**With keys:**
```
Old:  <Todo key={1} />  <Todo key={2} />  <Todo key={3} />
New:  <Todo key={3} />  <Todo key={1} />  <Todo key={2} />

React:
- key=3: moved from index 2 to 0 (reuse existing instance)
- key=1: moved from index 0 to 1 (reuse existing instance)
- key=2: moved from index 1 to 2 (reuse existing instance)

Result: 0 destroys, 0 creates, just DOM moves
```

**Without keys (using index):**
```
Old:  <Todo key={0} text="Learn" />  <Todo key={1} text="Build" />  <Todo key={2} text="Deploy" />
New:  <Todo key={0} text="Deploy" />  <Todo key={1} text="Learn" />  <Todo key={2} text="Build" />

React:
- key=0: text changed "Learn" → "Deploy" (UPDATE)
- key=1: text changed "Build" → "Learn" (UPDATE)
- key=2: text changed "Deploy" → "Build" (UPDATE)

Result: 3 updates (inefficient!)
```

---

## React Fiber

### What Problem Does It Solve?

**Before Fiber (React 15 and earlier):**
- Reconciliation was **synchronous** and **recursive**
- Once started, couldn't be interrupted
- Long updates blocked the main thread → janky UI

**Example scenario:**
```jsx
function HeavyList() {
  return (
    <div>
      {Array.from({ length: 10000 }).map((_, i) => (
        <ExpensiveComponent key={i} />
      ))}
    </div>
  );
}
```
- This takes 500ms to reconcile
- During those 500ms: no scrolling, no typing, no animations
- User sees the app "freeze"

**Fiber's solution:** Make reconciliation **interruptible**.

---

### How Fiber Works

**Fiber is:**
1. A **data structure** - each component/element is a "fiber" node
2. A **reconciliation engine** - implements Virtual DOM diffing
3. An **scheduler** - prioritizes and time-slices work

**Fiber node (simplified):**
```javascript
{
  type: 'div',           // What to render
  props: { /* ... */ },  // Component props
  child: Fiber,          // First child
  sibling: Fiber,        // Next sibling
  return: Fiber,         // Parent
  alternate: Fiber,      // Previous version (for diffing)
  effectTag: 'UPDATE',   // What changed
  stateNode: DOMNode,    // Actual DOM reference
}
```

**The key difference:** Instead of a recursive call stack (can't interrupt), Fiber uses a **linked list** (can pause at any node).

---

### Render Phase vs Commit Phase

**Render Phase (Interruptible):**
- Call component functions
- Build Virtual DOM tree
- Diff with previous tree (reconciliation)
- Mark what needs to update (effect tags)
- **CAN BE PAUSED** - no DOM changes yet

```javascript
// Simplified
function workLoop(deadline) {
  while (nextUnitOfWork && deadline.timeRemaining() > 0) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
  
  if (nextUnitOfWork) {
    // More work to do, schedule next chunk
    requestIdleCallback(workLoop);
  } else {
    // Done reconciling, commit to DOM
    commitRoot();
  }
}
```

**Commit Phase (Synchronous):**
- Apply all DOM updates in one go
- Call lifecycle methods (`useLayoutEffect`, `componentDidMount`)
- **CANNOT BE PAUSED** - must be atomic

---

### Priority and Lanes

Fiber assigns **priority** to updates:

```javascript
// High priority: user input
<input onChange={e => setQuery(e.target.value)} />

// Low priority: search results
<SearchResults query={query} />
```

**What Fiber does:**
1. User types "R" → high priority update queued
2. Fiber starts rendering `<SearchResults query="R">`
3. User types "e" → another high priority update
4. Fiber **interrupts** the search results render
5. Processes input update immediately (render input with "Re")
6. **Restarts** search results render with new query "Re"

**Interview tip:** This is called **concurrent rendering** (React 18+). Without Fiber, both updates would block each other.

---

## The Full Picture: How They Fit Together

```
┌─────────────────────────────────────────────────┐
│                 React Component                 │
│  function App() {                               │
│    const [count, setCount] = useState(0);       │
│    return <div>{count}</div>;                   │
│  }                                              │
└─────────────────────┬───────────────────────────┘
                      │
                      │ JSX → createElement
                      ↓
┌─────────────────────────────────────────────────┐
│              Virtual DOM (React Element)        │
│  {                                              │
│    type: 'div',                                 │
│    props: { children: [0] }                     │
│  }                                              │
└─────────────────────┬───────────────────────────┘
                      │
                      │ Reconciliation (Fiber)
                      ↓
┌─────────────────────────────────────────────────┐
│                  Fiber Tree                     │
│  {                                              │
│    type: 'div',                                 │
│    child: TextFiber,                            │
│    effectTag: 'UPDATE',  ← Marked what changed  │
│    stateNode: <div> DOM node                    │
│  }                                              │
└─────────────────────┬───────────────────────────┘
                      │
                      │ Commit Phase
                      ↓
┌─────────────────────────────────────────────────┐
│                  Real DOM                       │
│  <div>0</div>  ← Browser renders this           │
└─────────────────────────────────────────────────┘
```

---

## Common Interview Follow-ups

### Q: "Is Virtual DOM always faster than direct DOM manipulation?"

**Short answer:** No.

**Detailed answer:**
- For **small, targeted updates**, direct DOM can be faster
- For **complex UIs with many state changes**, Virtual DOM wins because:
  - Batches updates automatically
  - Minimizes reflows/repaints
  - Developer doesn't need to manually optimize

**Example where direct DOM is faster:**
```javascript
// Single targeted update
document.getElementById('counter').textContent = count;

// vs React
function Counter() {
  const [count, setCount] = useState(0);
  return <div id="counter">{count}</div>;
}
```

The first is technically faster, but React's value is **developer productivity** + **automatic optimization at scale**.

---

### Q: "What happens when you call `setState`?"

**In React 18+ (with Fiber):**

1. **Update queued** on the component's fiber
   ```javascript
   fiber.updateQueue.push({ state: newState });
   ```

2. **Schedule work** with appropriate priority
   ```javascript
   scheduleUpdateOnFiber(fiber, lane);
   ```

3. **Render phase** (interruptible)
   - Call component function with new state
   - Build new Virtual DOM
   - Diff with current tree
   - Mark effects

4. **Commit phase** (synchronous)
   - Apply DOM mutations
   - Update refs
   - Call `useLayoutEffect` / `componentDidUpdate`
   - Schedule `useEffect` callbacks

**Interview tip:** Mention that React 18 **automatically batches** multiple `setState` calls, even in async code.

---

### Q: "Why do keys need to be unique among siblings?"

**Answer:** Keys help React identify which items changed, moved, or were removed.

**Example problem:**
```jsx
// ❌ Non-unique keys
{items.map((item, i) => (
  <div key={i % 2}>  {/* Only 2 keys: 0 and 1 */}
    {item.name}
  </div>
))}
```

With 4 items: keys are [0, 1, 0, 1]
- Items 0 and 2 have the same key
- Items 1 and 3 have the same key
- React can't distinguish them → wrong elements get updated

**Why "among siblings":**
Keys only need to be unique within the same parent, not globally.

```jsx
// ✅ OK: same keys in different parents
<div>
  {groupA.map(item => <Item key={item.id} />)}
</div>
<div>
  {groupB.map(item => <Item key={item.id} />)}  {/* Same IDs OK */}
</div>
```

---

### Q: "What's the difference between React.memo and useMemo?"

**React.memo:** Memoizes the **component** (prevents re-render if props unchanged)
```javascript
const ExpensiveComponent = React.memo(function ExpensiveComponent({ data }) {
  // Only re-renders if `data` changes
  return <div>{/* heavy render */}</div>;
});
```

**useMemo:** Memoizes a **value** (prevents re-computation)
```javascript
function Component({ items }) {
  const sortedItems = useMemo(
    () => items.sort((a, b) => a - b),  // Only re-sorts if items change
    [items]
  );
  return <div>{sortedItems.length}</div>;
}
```

**Interview tip:** Both are optimizations. Don't use them prematurely - profile first.

---

### Q: "Can you explain React 18's concurrent rendering?"

**Answer:**

**Before React 18 (legacy mode):**
- Updates were synchronous (blocking)
- Once rendering started, it had to finish
- All updates had equal priority

**React 18 (concurrent mode):**
- Updates can be **interrupted** and **prioritized**
- High-priority updates (user input) interrupt low-priority ones (data fetching)
- Uses **time slicing** - yields to browser every ~5ms

**Example:**
```javascript
import { startTransition } from 'react';

function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  function handleChange(e) {
    setQuery(e.target.value);  // High priority (urgent)
    
    startTransition(() => {
      setResults(search(e.target.value));  // Low priority (not urgent)
    });
  }
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      <Results data={results} />
    </>
  );
}
```

**What happens:**
1. User types → `setQuery` runs immediately (input stays responsive)
2. `setResults` starts rendering in the background
3. If user types again, `setResults` render is abandoned and restarted
4. UI never freezes

**Interview tip:** Concurrent rendering is **opt-in** via `startTransition` or `useDeferredValue`. Default behavior is still synchronous for backwards compatibility.

---

## Code Challenge: Implement Simple Virtual DOM

**Question:** "Can you implement a basic Virtual DOM with diffing?"

```javascript
// Create virtual element
function h(type, props, ...children) {
  return { type, props: props || {}, children };
}

// Render virtual element to real DOM
function render(vnode) {
  if (typeof vnode === 'string' || typeof vnode === 'number') {
    return document.createTextNode(vnode);
  }
  
  const el = document.createElement(vnode.type);
  
  // Set props
  Object.entries(vnode.props).forEach(([key, value]) => {
    el.setAttribute(key, value);
  });
  
  // Render children
  vnode.children
    .map(render)
    .forEach(child => el.appendChild(child));
  
  return el;
}

// Diff and patch
function diff(oldVNode, newVNode) {
  // Different types → replace
  if (!oldVNode || oldVNode.type !== newVNode.type) {
    return { type: 'REPLACE', newVNode };
  }
  
  // Text nodes → update if changed
  if (typeof newVNode === 'string') {
    if (oldVNode !== newVNode) {
      return { type: 'TEXT', text: newVNode };
    }
    return null;
  }
  
  // Props changed?
  const propPatches = [];
  Object.keys(newVNode.props).forEach(key => {
    if (oldVNode.props[key] !== newVNode.props[key]) {
      propPatches.push({ type: 'SET_PROP', key, value: newVNode.props[key] });
    }
  });
  
  // Children changed?
  const childPatches = [];
  const maxLen = Math.max(oldVNode.children.length, newVNode.children.length);
  for (let i = 0; i < maxLen; i++) {
    childPatches.push(diff(oldVNode.children[i], newVNode.children[i]));
  }
  
  return {
    type: 'UPDATE',
    propPatches,
    childPatches,
  };
}

// Apply patch to real DOM
function patch(domNode, patches) {
  if (!patches) return domNode;
  
  if (patches.type === 'REPLACE') {
    return render(patches.newVNode);
  }
  
  if (patches.type === 'TEXT') {
    domNode.textContent = patches.text;
    return domNode;
  }
  
  if (patches.type === 'UPDATE') {
    patches.propPatches.forEach(p => {
      domNode.setAttribute(p.key, p.value);
    });
    
    patches.childPatches.forEach((childPatch, i) => {
      patch(domNode.childNodes[i], childPatch);
    });
  }
  
  return domNode;
}

// Usage
const oldVDOM = h('div', { id: 'container' },
  h('h1', {}, 'Hello'),
  h('p', {}, 'World')
);

const newVDOM = h('div', { id: 'container' },
  h('h1', {}, 'Hi'),  // Text changed
  h('p', {}, 'World')
);

const patches = diff(oldVDOM, newVDOM);
const domNode = render(oldVDOM);
patch(domNode, patches);  // Only updates <h1> text
```

**Interview tip:** Mention that real React is far more sophisticated (keys, effects, Fiber scheduling, etc.), but the core concept is the same.

---

## Key Takeaways

✅ **Virtual DOM** is a lightweight JS representation of the UI

✅ **Reconciliation** is the process of diffing old and new Virtual DOM trees

✅ **Fiber** is React's reconciliation engine that makes diffing interruptible and priority-based

✅ **Keys** are critical for efficient list reconciliation - use stable, unique IDs

✅ **The "speed" benefit** comes from batched, optimized updates, not from avoiding DOM entirely

✅ **Render phase** (interruptible) builds the diff; **Commit phase** (synchronous) applies it

✅ **Concurrent rendering** (React 18+) enables priority-based updates via Fiber's scheduler

---

## Red Flags in Interviews

❌ Saying "Virtual DOM is always faster" - it's not about raw speed, it's about **developer experience** and **automatic optimization**

❌ Confusing Virtual DOM with Shadow DOM - they're unrelated (Shadow DOM is a browser API for encapsulated DOM trees)

❌ Not mentioning **keys** when discussing lists

❌ Saying "Fiber replaced Virtual DOM" - Fiber **implements** Virtual DOM

❌ Claiming React re-renders everything on every state change - it only re-renders affected subtrees

---

## Further Reading

- [React Docs: Reconciliation](https://react.dev/learn/preserving-and-resetting-state)
- [Lin Clark's talk on Fiber](https://www.youtube.com/watch?v=ZCuYPiUIONs)
- [React 18 Working Group](https://github.com/reactwg/react-18/discussions)
- [Inside Fiber: in-depth overview](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
