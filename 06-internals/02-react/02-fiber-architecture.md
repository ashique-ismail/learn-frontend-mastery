# React Fiber Architecture

## The Idea

**In plain English:** React Fiber is the engine inside React that figures out what changed on your screen and updates only those parts, without freezing the page. It does this by breaking the update work into tiny pieces it can pause and resume, instead of doing everything all at once.

**Real-world analogy:** Imagine a chef who must repaint the menu board while also taking orders from customers. The old approach was to close the restaurant, repaint the whole board, then reopen — customers had to wait. The Fiber approach lets the chef repaint one section at a time, putting down the brush whenever a customer walks in, taking the order, then picking the brush back up exactly where they left off.

- The menu board = the UI on screen
- Repainting one section = one unit of Fiber work
- A customer walking in = a higher-priority event (like a user typing)
- Putting down the brush and resuming = pausing and resuming the render

---

## Learning Objectives
- Understand what problem Fiber solves
- Learn the Fiber data structure and how it represents the component tree
- Grasp the dual-buffering technique (current vs work-in-progress)
- Understand how Fiber enables concurrent rendering
- Learn the render and commit phases

---

## The Problem Fiber Solves

### Before Fiber: The Stack Reconciler

React's original reconciler (the "stack reconciler") had a fundamental limitation: **once rendering started, it couldn't be interrupted**.

**The problem:**
```javascript
function DeepTree() {
  return (
    <div>
      {Array.from({ length: 10000 }).map((_, i) => (
        <ExpensiveComponent key={i} />
      ))}
    </div>
  );
}
```

When this renders:
1. React starts at the root
2. Recursively processes every component
3. The call stack grows deep
4. **The main thread is blocked** - no user input, no animations, no scrolling
5. User sees the app "freeze"

**Why it couldn't be interrupted:** JavaScript's call stack is synchronous. Once you start a recursive function, you must complete it.

```javascript
// Simplified old reconciler
function reconcile(element) {
  // Process current element
  const instance = createInstance(element);
  
  // Recursively process children (CAN'T STOP HERE!)
  element.children.forEach(child => {
    reconcile(child);  // Must finish before returning
  });
  
  return instance;
}
```

**Fiber's solution:** Replace the call stack with a **custom data structure** that can be paused and resumed.

---

## What is Fiber?

A **Fiber** is a JavaScript object that represents a **unit of work**. It's React's virtual representation of a component instance or DOM node.

Think of Fiber as:
- A **node** in a tree data structure
- A **"frame" of work** that can be paused
- A **snapshot** of a component's state at a point in time

### The Fiber Node Structure

```typescript
interface Fiber {
  // Identity
  type: any;              // Component function/class, or DOM tag ("div")
  key: string | null;
  
  // Tree structure
  return: Fiber | null;   // Parent fiber
  child: Fiber | null;    // First child
  sibling: Fiber | null;  // Next sibling
  
  // State
  memoizedState: any;     // Hooks linked list (for function components)
  memoizedProps: any;     // Props from last render
  pendingProps: any;      // New props for this render
  
  // Effects
  flags: number;          // What side-effects this fiber has (Update, Placement, Deletion)
  subtreeFlags: number;   // Flags from all descendants
  
  // Dual buffering
  alternate: Fiber | null; // Points to the other tree (current ↔ work-in-progress)
  
  // Work tracking
  lanes: number;          // Priority of this update
  
  // DOM
  stateNode: any;         // DOM node, class instance, or other
}
```

---

## Fiber Tree Structure

Fiber uses a **linked list structure** instead of an array of children. Each fiber has three pointers:

```
        Parent
          |
        child
          ↓
      [First Child] → sibling → [Second Child] → sibling → [Third Child]
          |                          |                          |
        child                      child                      child
          ↓                          ↓                          ↓
      [Grandchild]              [Grandchild]              [Grandchild]
```

**Example component tree:**
```jsx
function App() {
  return (
    <div>
      <Header />
      <Content />
      <Footer />
    </div>
  );
}
```

**Resulting Fiber tree:**
```
App (Fiber)
 |
 child
 ↓
div (Fiber) ──────┐
 |                │
 child            │ return (parent pointer)
 ↓                │
Header (Fiber) ←──┘
 |
 sibling
 ↓
Content (Fiber)
 |
 sibling
 ↓
Footer (Fiber)
```

**Why this structure?**
- **No array allocation** - just pointer updates
- **Pauseable traversal** - you can stop at any node and resume later
- **Efficient** - constant-time pointer updates

---

## Tree Traversal Algorithm

Fiber traverses the tree using a **depth-first walk** with a twist: it can pause and resume.

```javascript
function performUnitOfWork(fiber) {
  // 1. Process this fiber (call component, diff DOM, etc.)
  processCurrentFiber(fiber);
  
  // 2. Try to go to child
  if (fiber.child) {
    return fiber.child;
  }
  
  // 3. No child? Try sibling
  if (fiber.sibling) {
    return fiber.sibling;
  }
  
  // 4. No sibling? Go back up to parent and try its sibling
  let parent = fiber.return;
  while (parent) {
    if (parent.sibling) {
      return parent.sibling;
    }
    parent = parent.return;
  }
  
  // 5. Reached the root, we're done
  return null;
}

// Work loop (can be interrupted!)
function workLoop(deadline) {
  while (nextUnitOfWork && deadline.timeRemaining() > 0) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
  
  if (nextUnitOfWork) {
    // More work to do, schedule continuation
    requestIdleCallback(workLoop);
  } else {
    // All work done, commit to DOM
    commitRoot();
  }
}
```

**Key insight:** Between each `performUnitOfWork`, React checks if it needs to pause (time expired, higher priority work arrived).

---

## Dual Buffering: Current vs Work-in-Progress

Fiber uses **two trees**:

1. **Current tree** - what's currently on screen
2. **Work-in-progress (WIP) tree** - being built during render

This is called **double buffering**, a technique from computer graphics.

```
┌─────────────────┐         ┌─────────────────┐
│  Current Tree   │ ←──────→│   WIP Tree      │
│  (on screen)    │ alternate│ (being built)   │
└─────────────────┘         └─────────────────┘
```

**How it works:**

**1. Initial render:**
```javascript
// React creates the first tree
const fiber = {
  type: App,
  alternate: null,  // No alternate yet
  // ...
};
```

**2. State update triggers re-render:**
```javascript
// React clones the current tree to create WIP tree
const workInProgressFiber = {
  type: App,
  alternate: currentFiber,  // Points to current
  // ...fresh state...
};

currentFiber.alternate = workInProgressFiber;  // Current points back
```

**3. Render phase builds WIP tree:**
- React processes the WIP tree
- Marks what changed (flags: Update, Placement, Deletion)
- Can be paused/resumed/aborted

**4. Commit phase swaps trees:**
```javascript
// Commit mutations to DOM
commitWork(workInProgressRoot);

// Swap: WIP becomes current
root.current = workInProgressRoot;
```

**Why double buffering?**
- **Consistency:** Users never see partial updates
- **Interruptibility:** Can abort WIP tree if higher priority work arrives
- **Efficiency:** Reuse fibers via `alternate` pointer instead of creating new objects

---

## The Two Phases: Render vs Commit

### Render Phase (Asynchronous, Can Be Interrupted)

**What happens:**
1. Call component functions
2. Diff virtual DOM
3. Mark fibers with effects (Update, Placement, Deletion)
4. Build WIP tree

**Key characteristics:**
- **Pure** - no side effects allowed
- **Interruptible** - can pause, resume, or restart
- **Invisible** - doesn't touch the real DOM

**Code example:**
```javascript
function beginWork(fiber) {
  if (fiber.type === FunctionComponent) {
    // Call your component
    const children = fiber.type(fiber.pendingProps);
    
    // Reconcile children (diff)
    reconcileChildren(fiber, children);
  }
  // ...
}
```

**Why it can be interrupted:**
- No DOM mutations yet
- No user-visible effects
- Safe to throw away WIP tree and restart

---

### Commit Phase (Synchronous, Cannot Be Interrupted)

**What happens:**
1. **Before Mutation:** Call `getSnapshotBeforeUpdate`
2. **Mutation:** Apply DOM changes (insert, update, delete)
3. **Layout:** Call `useLayoutEffect`, `componentDidMount/Update`
4. **After Paint:** Schedule `useEffect` callbacks

**Key characteristics:**
- **Synchronous** - runs to completion
- **Single pass** - processes all effects at once
- **Visible** - user sees the changes

**Code example:**
```javascript
function commitRoot(root) {
  const finishedWork = root.finishedWork;
  
  // Phase 1: Before mutation
  commitBeforeMutationEffects(finishedWork);
  
  // Phase 2: Mutation (update DOM)
  commitMutationEffects(finishedWork);
  
  // Swap trees (WIP becomes current)
  root.current = finishedWork;
  
  // Phase 3: Layout effects
  commitLayoutEffects(finishedWork);
  
  // Phase 4: Schedule passive effects (useEffect)
  schedulePassiveEffects(finishedWork);
}
```

**Why it can't be interrupted:**
- Partially applied DOM changes would cause visual inconsistencies
- Needs to be atomic from user's perspective

---

## Effect Flags: What Changed?

Fiber uses **bitwise flags** to track what work needs to be done.

```javascript
// Simplified flags
const NoFlags = 0b000000;
const Placement = 0b000001;  // Insert into DOM
const Update = 0b000010;     // Update existing DOM node
const Deletion = 0b000100;   // Remove from DOM
const Snapshot = 0b001000;   // getSnapshotBeforeUpdate
const Passive = 0b010000;    // useEffect
const Layout = 0b100000;     // useLayoutEffect

// Example fiber after render phase
const fiber = {
  flags: Update | Layout,  // 0b100010 = needs update + has layout effect
  // ...
};
```

**During commit, React checks flags:**
```javascript
function commitMutationEffects(fiber) {
  if (fiber.flags & Placement) {
    insertDOM(fiber);
  }
  if (fiber.flags & Update) {
    updateDOM(fiber);
  }
  if (fiber.flags & Deletion) {
    removeDOM(fiber);
  }
}
```

**Optimization:** `subtreeFlags` aggregates all descendant flags, allowing React to skip entire subtrees with no effects.

```javascript
if (fiber.subtreeFlags === NoFlags && fiber.flags === NoFlags) {
  // Nothing to do in this subtree, skip it
  return;
}
```

---

## Concurrent Rendering

Fiber enables **concurrent rendering** - the ability to work on multiple versions of the UI simultaneously.

### Priority Lanes

React 18+ uses **lanes** (32-bit bitmask) to track update priorities:

```javascript
const SyncLane = 0b0000000000000000000000000000001;  // Highest priority
const InputContinuousLane = 0b0000000000000000000000000000100;
const DefaultLane = 0b0000000000000000000000000010000;
const TransitionLane = 0b0000000000000000001000000000000;  // Lower priority
const IdleLane = 0b0100000000000000000000000000000;  // Lowest priority
```

**Example: Typing while heavy render is in-progress**

```javascript
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  return (
    <>
      {/* High priority: user input */}
      <input 
        value={query} 
        onChange={e => setQuery(e.target.value)}  // SyncLane
      />
      
      {/* Low priority: search results */}
      <Results data={results} />  {/* TransitionLane */}
    </>
  );
}
```

**What Fiber does:**
1. User types → `setQuery` gets **SyncLane** (high priority)
2. Heavy `<Results>` render is in-progress on **TransitionLane** (low priority)
3. Fiber **interrupts** the results render
4. Processes input update immediately
5. Resumes results render with new query

---

### Time Slicing

Fiber breaks work into **chunks** and yields to the browser between chunks.

```javascript
function workLoopConcurrent() {
  // Work until time expires or higher priority work arrives
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function shouldYield() {
  const currentTime = performance.now();
  return (
    currentTime >= deadline ||  // Time slice expired
    hasHigherPriorityWork()     // More urgent work arrived
  );
}
```

**Typical time slice:** ~5ms

**Visual timeline:**
```
Frame 1 (16.67ms):
  [Fiber work 5ms] [Browser: paint, input, etc. 11.67ms]

Frame 2:
  [Fiber work 5ms] [Browser 11.67ms]

Frame 3:
  [Fiber work 5ms] [Browser 11.67ms]
  
Result: UI stays responsive at 60fps
```

---

## Hooks Implementation on Fiber

Hooks are stored as a **linked list** on the fiber's `memoizedState`.

```javascript
const fiber = {
  memoizedState: {  // First hook
    memoizedState: 'value',  // The actual state value
    next: {  // Second hook
      memoizedState: 42,
      next: {  // Third hook
        memoizedState: [1, 2, 3],
        next: null,
      },
    },
  },
};
```

**Example component:**
```javascript
function Counter() {
  const [count, setCount] = useState(0);      // Hook 1
  const [name, setName] = useState('React');  // Hook 2
  useEffect(() => {
    document.title = `${name}: ${count}`;
  }, [name, count]);                          // Hook 3
  
  return <div>{count}</div>;
}
```

**Fiber representation:**
```javascript
{
  type: Counter,
  memoizedState: {
    // Hook 1: useState(0)
    memoizedState: 0,
    queue: { /* update queue */ },
    next: {
      // Hook 2: useState('React')
      memoizedState: 'React',
      queue: { /* update queue */ },
      next: {
        // Hook 3: useEffect
        memoizedState: {
          create: () => { /* effect function */ },
          deps: ['React', 0],
        },
        next: null,
      },
    },
  },
}
```

**This is why the "Rules of Hooks" exist:**
- Hooks must be called in the **same order** every render
- React walks the linked list in order
- If order changes, hooks get mismatched with their state

---

## Example: Tracing a Render

**Component:**
```jsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <h1>{count}</h1>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}
```

**1. Initial mount - Fiber tree:**
```
App (Fiber)
├─ memoizedState: { memoizedState: 0, queue: {...}, next: null }
├─ child ──→ div (Fiber)
             ├─ child ──→ h1 (Fiber)
             │            ├─ stateNode: <h1> DOM node
             │            └─ child ──→ "0" (Text Fiber)
             └─ sibling ──→ button (Fiber)
                          ├─ stateNode: <button> DOM node
                          └─ child ──→ "+" (Text Fiber)
```

**2. User clicks button - `setCount(1)`:**
- Update queued on App fiber's hook
- Fiber marked with **Update** flag
- Scheduler picks up work

**3. Render phase:**
```javascript
// React calls App function
const children = App({ /* props */ });

// New props for div: { children: [<h1>1</h1>, <button>...</button>] }
// Reconcile: h1's text changed from "0" to "1"
// Mark text fiber with Update flag
```

**4. Commit phase:**
```javascript
// Mutation phase
textFiber.stateNode.nodeValue = "1";  // Update DOM

// Swap trees
root.current = workInProgressRoot;
```

**5. Screen updates** - user sees "1"

---

## Key Takeaways

✅ **Fiber is a data structure** that represents a unit of work, not a rendering technique

✅ **Linked list structure** enables pauseable traversal (child, sibling, return pointers)

✅ **Dual buffering** (current ↔ WIP) enables safe interruption and atomic commits

✅ **Render phase is async** (can pause), **commit phase is sync** (must complete)

✅ **Effect flags** (bitwise) track what changed, enabling efficient DOM updates

✅ **Lanes** manage update priorities, enabling concurrent rendering

✅ **Hooks are a linked list** on `fiber.memoizedState`, explaining the Rules of Hooks

✅ **Time slicing** yields to browser every ~5ms, keeping UI responsive

---

## Common Misconceptions

❌ **"Fiber makes React faster"**
→ Not directly. Fiber enables **responsiveness** and **priority-based rendering**, not raw speed.

❌ **"Fiber is the Virtual DOM"**
→ Fiber is the **reconciler**. Virtual DOM is the concept; Fiber is the implementation.

❌ **"Render phase is always interrupted"**
→ Only in **concurrent mode**. Sync updates (e.g., user input in React 18) run without interruption.

❌ **"Two trees mean double memory"**
→ Fibers are reused via `alternate` pointers. Memory overhead is minimal.

---

## Visualizing Fiber in DevTools

**React DevTools Profiler** shows Fiber at work:

1. Open React DevTools
2. Go to **Profiler** tab
3. Start recording
4. Trigger a render
5. View the **Flamegraph** or **Ranked** chart

**What you see:**
- Each bar is a Fiber's render time
- Hovering shows which component/fiber
- Colors indicate render duration (green = fast, yellow/red = slow)

**Concurrent rendering indicators:**
- Multiple "render" passes before commit = work was interrupted/restarted
- "Passive effects" timing = `useEffect` callbacks

---

## Resources for Deeper Learning

- [React Fiber Architecture (Lin Clark's talk)](https://www.youtube.com/watch?v=ZCuYPiUIONs) - visual explanation
- [React source code: ReactFiber.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiber.js)
- [Inside Fiber: in-depth overview](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
- [React 18 Working Group Discussions](https://github.com/reactwg/react-18/discussions)
