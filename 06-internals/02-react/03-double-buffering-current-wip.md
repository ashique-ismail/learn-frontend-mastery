# React Double Buffering: Current and Work-in-Progress Fiber Trees

## The Idea

**In plain English:** Double buffering is how React prepares a brand-new version of the screen behind the scenes without touching what the user is already looking at. It keeps two copies of its internal blueprint (called a "fiber tree") — one that matches what is currently shown on screen, and one that is quietly being updated — then swaps them in a single instant when the new version is ready.

**Real-world analogy:** Think of a live theatre stage with a hidden second stage behind a curtain. The crew builds and arranges the next scene's set behind the curtain while the audience watches the current scene play out on the live stage. When the scene change happens, the curtain drops for just a moment and the new set is revealed — the audience never sees a half-built stage.

- The live stage = the current fiber tree (what is on screen right now)
- The hidden stage behind the curtain = the work-in-progress fiber tree (being built during a render)
- Dropping the curtain and revealing the new set = the commit phase swap (`fiberRoot.current = finishedWork`)

---

## Table of Contents
- [Overview](#overview)
- [The Two-Tree Architecture](#the-two-tree-architecture)
- [Fiber Node Anatomy](#fiber-node-anatomy)
- [The alternate Field](#the-alternate-field)
- [Render Phase: Building the WIP Tree](#render-phase-building-the-wip-tree)
- [Commit Phase: Swapping the Trees](#commit-phase-swapping-the-trees)
- [Concurrent Mode Implications](#concurrent-mode-implications)
- [Stale Closures and Double Buffering](#stale-closures-and-double-buffering)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

React's fiber architecture maintains **two fiber trees** at all times: the **current tree** (reflecting what is currently rendered in the DOM) and the **work-in-progress (WIP) tree** (the new tree being constructed during a render). This is called **double buffering** — the same technique used in graphics rendering to prevent screen tearing.

The key insight: React never mutates the current tree mid-render. It builds a completely new tree in the WIP buffer, then atomically swaps it in during the commit phase. This ensures the DOM always reflects a consistent state and makes concurrent rendering safe.

```
BEFORE render:
  current → [App] → [Header] → [Main] → [Footer]
  WIP     → null (no pending work)

DURING render:
  current → [App] → [Header] → [Main] → [Footer]  (untouched — live DOM)
  WIP     → [App'] → [Header'] → [Main'] → [Footer'] (being built)

AFTER commit:
  current → [App'] → [Header'] → [Main'] → [Footer'] (new tree is now current)
  WIP     → [App] → [Header] → [Main] → [Footer]     (old tree, reusable)
```

## The Two-Tree Architecture

### Why Double Buffering?

```typescript
// Without double buffering: mutation mid-render is catastrophic
// Imagine React directly mutating the current tree while rendering:
function render() {
  // React is in the middle of updating Main...
  // User action triggers a read of the component tree state
  // The tree is HALF-updated — inconsistent!
  // React could interrupt itself in concurrent mode and see partial state
}

// With double buffering:
function render() {
  // React builds WIP tree independently
  // Current tree is always consistent (reflects last committed state)
  // User actions, suspense, and interruptions all read from current tree
  // Only when WIP is fully ready does React swap it in
}
```

### FiberRoot and the Current Pointer

```typescript
// Simplified FiberRoot structure (packages/react-reconciler/src/ReactFiber.ts)
interface FiberRoot {
  // The current tree's root fiber
  current: Fiber; // Points to HostRoot fiber of current tree

  // Used during render to track work in progress
  // The WIP root is the alternate of current
  finishedWork: Fiber | null; // WIP tree root when render completes

  // Container (DOM node, usually document.getElementById('root'))
  containerInfo: Container;
}

// After every commit:
// fiberRoot.current = finishedWork (WIP tree becomes current)
// The old current tree's root becomes fiberRoot.current.alternate
// = available for the next WIP tree (memory reuse)
```

## Fiber Node Anatomy

### Core Fiber Fields Relevant to Double Buffering

```typescript
// From packages/react-reconciler/src/ReactInternalTypes.ts (simplified)
interface Fiber {
  // === Identity ===
  tag: WorkTag;            // Type of fiber (FunctionComponent, ClassComponent, etc.)
  key: null | string;      // Reconciliation key
  type: any;               // Component function/class

  // === Tree Structure ===
  return: Fiber | null;    // Parent fiber
  child: Fiber | null;     // First child fiber
  sibling: Fiber | null;   // Next sibling fiber
  index: number;           // Position among siblings

  // === THE DOUBLE BUFFER POINTER ===
  alternate: Fiber | null;
  // current.alternate = WIP fiber
  // WIP.alternate = current fiber
  // This is how React pairs corresponding fibers across the two trees

  // === State ===
  pendingProps: any;       // Props at the start of this render
  memoizedProps: any;      // Props from the last completed render (on current)
  memoizedState: any;      // State/hooks linked list (on current)

  // === Effects ===
  flags: Flags;            // Bitmask: Placement, Update, Deletion, etc.
  subtreeFlags: Flags;     // Aggregated flags from subtree (React 18+)
  deletions: Fiber[] | null; // Child fibers to delete

  // === Work ===
  lanes: Lanes;            // Priority lanes for pending work
  childLanes: Lanes;       // Lanes of descendant work
}
```

### Fiber Tag Types

```typescript
// WorkTag values (what type of fiber is this?)
const WorkTag = {
  FunctionComponent: 0,
  ClassComponent: 1,
  IndeterminateComponent: 2, // Before we know if function or class
  HostRoot: 3,               // Root of the tree (fiberRoot.current)
  HostPortal: 4,             // React.createPortal
  HostComponent: 5,          // DOM element (div, span, etc.)
  HostText: 6,               // Text node
  Fragment: 7,
  Mode: 8,                   // StrictMode, ConcurrentMode
  ContextConsumer: 9,
  ContextProvider: 10,
  ForwardRef: 11,
  Profiler: 12,
  SuspenseComponent: 13,
  MemoComponent: 14,
  // ... etc
};
```

## The alternate Field

### How Pairing Works

```typescript
// The alternate pointer creates a doubly-linked relationship between
// the current fiber and its WIP counterpart.

// During a render, React creates the WIP fiber from the current fiber:
function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;

  if (workInProgress === null) {
    // No alternate exists yet — create a new fiber
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode; // Shares DOM node!

    // Establish the bidirectional link
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    // Alternate already exists (reuse from previous render cycle)
    workInProgress.pendingProps = pendingProps;
    workInProgress.type = current.type;

    // Reset effect flags (will be re-computed during this render)
    workInProgress.flags = NoFlags;
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;
  }

  // Copy over fields from current that don't change during render
  workInProgress.lanes = current.lanes;
  workInProgress.childLanes = current.childLanes;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;

  return workInProgress;
}
```

### The alternate Pointer Through a Render Cycle

```
Initial mount (no current tree yet):
  current: HostRoot fiber (created by createRoot)
  WIP:     createWorkInProgress(current) → new fiber, alternate = current

First render complete:
  Commit swaps: fiberRoot.current = WIP root
  Tree state:
    [current] App ←──alternate──→ [WIP] App (old current, now alternate)

Second render triggered:
  New WIP = createWorkInProgress(current App)
  Since current.alternate already exists, REUSE it!
  → Reset its flags, copy memoizedState from current
  → Build the new tree on top of the old alternate

Second render complete:
  Commit swaps: fiberRoot.current = new WIP root
  Tree state: same structure, roles swapped

This flip-flop continues indefinitely — only 2 trees ever exist at once.
```

### stateNode Sharing

```typescript
// IMPORTANT: Both current and WIP fibers point to the SAME DOM node
// (via stateNode). Only one set of DOM properties are written to.

// HostComponent fiber (e.g., <div>):
const currentDivFiber = {
  tag: HostComponent, // 5
  stateNode: actualDOMDiv, // The real <div> in the document
  memoizedProps: { className: 'old' },
  // ...
};

const wipDivFiber = {
  tag: HostComponent,
  stateNode: actualDOMDiv, // SAME DOM node!
  pendingProps: { className: 'new' },
  // ...
};

// During commit, React uses WIP fiber's pendingProps to update actualDOMDiv
// The stateNode is shared because we don't create a new DOM node on every update
// We just update the PROPERTIES of the existing DOM node
```

## Render Phase: Building the WIP Tree

### beginWork and completeWork

```typescript
// The render phase walks the WIP tree doing work at each fiber

// beginWork: process a fiber, create/reconcile children
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const newProps = workInProgress.pendingProps;
      // Run the function component → execute hooks
      return updateFunctionComponent(current, workInProgress, Component, newProps, renderLanes);
    }
    case HostComponent: {
      // DOM element — reconcile children
      return updateHostComponent(current, workInProgress, renderLanes);
    }
    // ...
  }
}

// completeWork: finalize a fiber after all children are done
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  switch (workInProgress.tag) {
    case HostComponent: {
      if (current !== null && workInProgress.stateNode != null) {
        // UPDATE: compare old props to new props, generate update payload
        updateHostComponent(current, workInProgress, ...);
      } else {
        // MOUNT: create new DOM node
        const instance = createInstance(workInProgress.type, workInProgress.pendingProps, ...);
        workInProgress.stateNode = instance;
        // Append child DOM nodes
        appendAllChildren(instance, workInProgress, false, false);
      }
      // Bubble flags up to parent
      bubbleProperties(workInProgress);
      return null;
    }
  }
}
```

### The Work Loop

```typescript
// Simplified render phase work loop
function workLoopConcurrent(): void {
  // Process fibers one at a time, yielding to browser between fibers
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate; // Corresponding current fiber

  let next: Fiber | null;
  next = beginWork(current, unitOfWork, subtreeRenderLanes);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // No child fiber returned — this fiber is complete
    completeUnitOfWork(unitOfWork);
  } else {
    // Move to the child fiber
    workInProgress = next;
  }
}

// completeUnitOfWork walks back up when no more children
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return; // parent

    completeWork(current, completedWork, subtreeRenderLanes);

    // Move to sibling if one exists
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      workInProgress = siblingFiber;
      return;
    }
    // Move back to parent
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

### Effect Collection During Render

```typescript
// During render, fibers are tagged with flags describing needed mutations
const Flags = {
  NoFlags: 0b000000000000000000000000000,
  Placement: 0b000000000000000000000000010,    // Insert into DOM
  Update: 0b000000000000000000000000100,       // Update DOM props/text
  Deletion: 0b000000000000000000000001000,     // Remove from DOM
  ContentReset: 0b000000000000000000000010000,
  Callback: 0b000000000000000000000100000,     // Has setState callback
  DidCapture: 0b000000000000000000001000000,
  ChildDeletion: 0b000000000000000000010000000, // Has child deletions
  // ...
};

// React 18 subtreeFlags optimization:
// Each fiber aggregates its children's flags into subtreeFlags
// The commit phase can skip entire subtrees with subtreeFlags === NoFlags
// This is much faster than walking the entire tree looking for effects
```

## Commit Phase: Swapping the Trees

### Three Sub-Phases of Commit

```typescript
// commitRoot orchestrates three passes over the effect list (or WIP tree)

function commitRoot(root: FiberRoot, finishedWork: Fiber): void {
  // === PHASE 1: Before Mutation ===
  // Call getSnapshotBeforeUpdate, schedule useEffect cleanup
  commitBeforeMutationEffects(root, finishedWork);

  // === PHASE 2: Mutation ===
  // Apply all DOM mutations (insertions, updates, deletions)
  // THIS IS WHERE THE TREE SWAP HAPPENS
  commitMutationEffects(root, finishedWork, lanes);

  // THE SWAP: current tree is now the WIP tree
  root.current = finishedWork; // ← ATOMIC SWAP

  // === PHASE 3: Layout ===
  // Run useLayoutEffect callbacks, componentDidMount/Update
  // These run AFTER DOM mutations but BEFORE browser paints
  commitLayoutEffects(finishedWork, root, lanes);

  // Schedule useEffect callbacks (run asynchronously after paint)
  scheduleCallback(NormalSchedulerPriority, () => {
    flushPassiveEffects();
  });
}
```

### Why the Swap Happens After Mutation but Before Layout

```typescript
// This ordering is deliberate:
// - DOM mutations are done, so Layout effects can read the final DOM
// - current = finishedWork happens before layout effects,
//   so useLayoutEffect callbacks see the "new" tree as current

// Example: In useLayoutEffect, you can call setState
// React processes this synchronously — the new state starts a render
// on the CURRENT tree (which is now finishedWork) as a base
// If the swap hadn't happened, React would render against the old tree!

function MyComponent() {
  const [size, setSize] = useState(0);

  useLayoutEffect(() => {
    // This runs after DOM mutation, after root.current = finishedWork
    const width = ref.current.offsetWidth;
    if (width !== size) {
      setSize(width); // Triggers synchronous re-render
      // React will start another render with root.current = finishedWork as base
    }
  }, []);

  return <div ref={ref} style={{ width: size ? `${size}px` : 'auto' }} />;
}
```

### Deletion Handling During Commit

```typescript
// Deletions are special: they're tracked on the parent fiber
// (not on the deleted fiber itself, since it may not be in the WIP tree)

// During render, when a child is removed:
function deleteChild(returnFiber: Fiber, childToDelete: Fiber): void {
  const deletions = returnFiber.deletions;
  if (deletions === null) {
    returnFiber.deletions = [childToDelete];
    returnFiber.flags |= ChildDeletion;
  } else {
    deletions.push(childToDelete);
  }
}

// During commit mutation phase:
function commitMutationEffectsOnFiber(finishedWork: Fiber): void {
  // Process deletions first
  const deletions = finishedWork.deletions;
  if (deletions !== null) {
    for (const childToDelete of deletions) {
      commitDeletionEffects(root, finishedWork, childToDelete);
      // Recursively unmount: call useEffect cleanups, componentWillUnmount,
      // detach refs, then remove the DOM node
    }
  }
  // ... process other flags
}
```

## Concurrent Mode Implications

### Interrupted Renders and the WIP Tree

```typescript
// In concurrent mode, React can interrupt and discard a WIP tree
// The current tree is NEVER affected by an abandoned render

// Scenario:
// 1. High priority update arrives (e.g., user types in input)
// 2. React is in the middle of a low-priority render (WIP tree being built)
// 3. React throws away the partial WIP tree
// 4. The current tree still shows the last committed state — no jank
// 5. React starts a new WIP tree for the high-priority update

// Discarding WIP fibers:
function resetWorkInProgress(workInProgress: Fiber, renderLanes: Lanes): void {
  // Reset the WIP fiber to re-derive its state from current
  workInProgress.flags &= StaticMask;
  workInProgress.subtreeFlags = NoFlags;
  workInProgress.deletions = null;

  const current = workInProgress.alternate;
  if (current !== null) {
    // Restore from current fiber (the abandoned render's changes are discarded)
    workInProgress.memoizedState = current.memoizedState;
    workInProgress.lanes = current.lanes;
    workInProgress.pendingProps = workInProgress.pendingProps; // keep new props
  }
}
```

### Suspense and Partial Trees

```typescript
// Suspense creates a partial render where some subtrees are dehydrated
// React keeps the current tree visible while building the WIP tree

// During render, if a component suspends:
// 1. React throws a "thenable" (promise)
// 2. React catches it at the nearest Suspense boundary
// 3. The Suspense fiber's WIP is marked with DidCapture flag
// 4. React re-renders the Suspense boundary's WIP subtree with fallback content
// 5. The WIP tree completes — showing fallback
// 6. When the promise resolves, React starts a new render
//    with the now-available data

// The current tree showed: <ExpensiveComponent /> (or nothing on first load)
// WIP shows: <Suspense fallback={<Spinner/>}> ... </Suspense>
// After data: WIP shows: <Suspense> <ExpensiveComponent data={data} /> </Suspense>
```

### Reading Consistent State During Render

```typescript
// Because the current tree is never mutated during render,
// concurrent reads from event handlers always see a consistent state

// Event handlers read from the current fiber's memoizedState
// Renders build new state on the WIP fiber's pendingProps/memoizedState

function Counter() {
  const [count, setCount] = useState(0);

  // Event handler reading count reads from current fiber's memoizedState
  const handleClick = () => {
    // Even if a render is in progress, this reads the committed count
    console.log('Current count:', count);
    setCount(c => c + 1);
  };

  // Rendering reads from WIP fiber's state
  return <button onClick={handleClick}>{count}</button>;
}
```

## Stale Closures and Double Buffering

### How Hooks Access State Across Both Trees

```typescript
// useState is stored on the fiber's memoizedState linked list
// When a component renders, hooks are read from the WIP fiber's memoizedState
// (which was copied from current.memoizedState at WIP creation time)

// Hook state linked list on a fiber:
interface Hook {
  memoizedState: any;    // The state value (or effect/ref/etc)
  baseState: any;        // State before pending updates
  baseQueue: Update<any> | null;
  queue: UpdateQueue<any> | null; // Pending updates
  next: Hook | null;     // Next hook in the list
}

// Fiber.memoizedState → Hook1 → Hook2 → Hook3 → null

// During render:
function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: Function,
  props: any
): any {
  // Point hooks dispatcher to the WIP fiber
  currentlyRenderingFiber = workInProgress;

  // Copy hooks state from current to WIP (done in createWorkInProgress)
  workInProgress.memoizedState = null; // Will be rebuilt by hook calls

  // Run the component function — hooks read/write from workInProgress
  const children = Component(props);

  currentlyRenderingFiber = null;
  return children;
}
```

### The useRef Trick to Avoid Stale Closures

```typescript
// Refs escape the double-buffering model:
// ref.current is a mutable box on the current fiber (not the WIP)
// Reading ref.current in a closure always gets the latest value

function useLatestCallback<T extends (...args: any[]) => any>(callback: T): T {
  const ref = useRef(callback);

  // useLayoutEffect runs after commit — current tree is now the new WIP
  // So ref.current is updated to the latest callback before effects run
  useLayoutEffect(() => {
    ref.current = callback;
  });

  // This stable function reference never stales — it always calls current callback
  return useCallback((...args: any[]) => {
    return ref.current(...args);
  }, []) as T;
}
```

## Common Pitfalls

### Pitfall 1: Mutating State During Render

```typescript
// NEVER mutate in a class component render or function component body
// React may call render multiple times in concurrent mode before committing

let renderCount = 0; // Module-level variable — shared across renders

function BadComponent({ data }: { data: string[] }) {
  renderCount++; // BAD: Mutating outside state during render
  // In strict mode, render is called twice on mount — this would be 2, not 1
  // In concurrent mode, render may be called multiple times before commit

  data.push('mutated'); // CATASTROPHIC: Mutating prop!
  // Since current and WIP share prop references, this affects both trees

  return <div>{data.join(', ')}</div>;
}

// GOOD: Only compute during render, never mutate
function GoodComponent({ data }: { data: string[] }) {
  const processed = useMemo(() => [...data, 'computed'], [data]);
  return <div>{processed.join(', ')}</div>;
}
```

### Pitfall 2: Relying on DOM State After Render But Before Commit

```typescript
// The DOM is NOT updated during render phase
// DOM reads during render may return stale values

function BadMeasurement() {
  const ref = useRef<HTMLDivElement>(null);

  // BAD: Reading DOM during render — DOM hasn't been updated yet!
  if (ref.current) {
    const height = ref.current.offsetHeight; // May return old/wrong value
    // The WIP tree hasn't been committed, DOM reflects current tree
  }

  return <div ref={ref}>content</div>;
}

// GOOD: Read DOM in effects (after commit) or useLayoutEffect
function GoodMeasurement() {
  const ref = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    // DOM is now committed — safe to read
    if (ref.current) {
      setHeight(ref.current.offsetHeight);
    }
  });

  return <div ref={ref} style={{ lineHeight: `${height}px` }}>content</div>;
}
```

### Pitfall 3: Reading alternate Directly

```typescript
// NEVER read fiber.alternate or manipulate fiber internals in user code
// The double-buffer structure is an implementation detail

// BAD: Accessing internals
function badHook() {
  const fiber = (React as any).__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED
    .ReactCurrentFiber.current;
  const wipAlternate = fiber?.alternate; // DO NOT DO THIS
  // The alternate relationship changes with every render
  // This will break with any React version
}

// GOOD: Use the provided APIs
function goodHook() {
  const ref = useRef(null); // React manages the ref on the fiber
  // ref.current is updated safely after commit
}
```

## Interview Questions

### Question 1: What is double buffering in React and why is it used?

**Answer**: Double buffering is a rendering technique where React maintains two fiber trees simultaneously: the **current tree** (reflecting the last committed, visible state) and the **work-in-progress (WIP) tree** (the tree being built for the next render). React never mutates the current tree during a render. Once the WIP tree is fully built and all effects are scheduled, React atomically swaps `fiberRoot.current = finishedWork` during the commit phase. This guarantees:
1. The DOM always reflects a consistent, fully-committed state
2. An interrupted render (concurrent mode) can be discarded without affecting the live UI
3. Suspense can show fallback content (from current tree) while building a new tree

### Question 2: What is the `alternate` pointer on a fiber and how does it enable the double buffer?

**Answer**: Every fiber has an `alternate` field that points to its counterpart in the other tree. If `A` is the current fiber for a component, then `A.alternate` is the WIP fiber for the same component (and vice versa). This bidirectional pairing means React can navigate between trees from any fiber. When React creates a WIP fiber, it reuses the old current fiber as the new WIP alternate (no extra allocation after the first render). After commit, the roles simply flip — what was WIP becomes current, and what was current becomes the next render's WIP candidate.

### Question 3: At what point does `fiberRoot.current` get swapped to the new tree?

**Answer**: The swap `root.current = finishedWork` happens in `commitRoot`, during the **mutation phase** — after all DOM mutations have been applied but before `useLayoutEffect` callbacks run. This ordering is intentional: layout effects run after the swap so that if they call `setState`, React will start the next render from the correct (newly-committed) current tree as a base. Running layout effects before the swap would cause incorrect behavior where the "current" tree would still point to the old state.

### Question 4: How does concurrent mode benefit from double buffering?

**Answer**: In concurrent mode, React can interrupt a WIP tree build mid-way and discard the partially-built WIP tree entirely. Because the current tree is never touched during render, the user always sees the last committed, consistent state — there's no partial or corrupted DOM. React can also use the discarded WIP fibers for the next render (they're already allocated and linked via `alternate`). The double-buffer model also enables time-slicing: React can work on the WIP tree across multiple browser frames, yielding between fibers, without ever exposing an inconsistent intermediate state.

### Question 5: Why does React Strict Mode render components twice in development?

**Answer**: In Strict Mode, React intentionally renders function components twice (the WIP tree is built, discarded, then built again from the current tree). This verifies that component render functions are pure — no side effects during render. Since double buffering means the render phase can be discarded at any time in concurrent mode, components must be idempotent. The double render exposes bugs where render-phase code has side effects (mutating external state, making network requests, etc.) that would cause incorrect behavior when React discards a WIP render in production.

## Key Takeaways

1. **Two trees, always**: React maintains `current` (committed, visible) and `work-in-progress` (being built) fiber trees — never just one

2. **alternate is the pairing mechanism**: `current.alternate === WIP` and `WIP.alternate === current` — this bidirectional link enables the flip-flop model

3. **The swap is atomic**: `fiberRoot.current = finishedWork` happens in a single assignment during the commit mutation phase — there is no intermediate state

4. **DOM nodes are shared**: Both current and WIP fiber nodes for DOM elements point to the same `stateNode` (actual DOM node) — the DOM isn't recreated on every render, only properties are updated

5. **Render phase must be pure**: React may call render multiple times before committing (concurrent mode, Strict Mode double-invoke) — any side effects in render are unsafe

6. **Layout effects see the new current tree**: The swap happens before `useLayoutEffect` runs, so layout effects that call `setState` correctly start from the newly committed state

7. **Effects are scheduled, not immediate**: `useEffect` callbacks are scheduled after the commit and run asynchronously after paint — they do NOT run during the commit phase

8. **Interrupted renders are free**: In concurrent mode, discarding a WIP tree doesn't cause a visible flicker — the current tree remains intact and painted

9. **Memory efficiency**: The double-buffer flip-flop reuses fibers — after two render cycles, no new fibers are allocated for existing components (only new/moved components need new fibers)

10. **subtreeFlags optimization**: React 18 aggregates child flags into `subtreeFlags` so the commit phase can skip entire subtrees with no pending work

## Resources

### React Source Code
- `packages/react-reconciler/src/ReactFiber.ts` — fiber creation, `createWorkInProgress`
- `packages/react-reconciler/src/ReactFiberWorkLoop.ts` — work loop, `commitRoot`
- `packages/react-reconciler/src/ReactFiberCommitWork.ts` — commit phases

### Articles
- "React Fiber Architecture" — Andrew Clark (original design doc)
- "Inside Fiber: In-depth overview" — Max Koretskyi (indepth.dev)
- "The how and why of React's double buffering" — Mark Erikson
