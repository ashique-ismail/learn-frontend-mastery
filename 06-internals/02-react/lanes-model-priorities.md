# React Lanes Model and Priorities

## Table of Contents
- [Overview](#overview)
- [What Are Lanes](#what-are-lanes)
- [Lane Bitmask Layout](#lane-bitmask-layout)
- [Priority Levels and Mapping](#priority-levels-and-mapping)
- [How Lanes Are Assigned](#how-lanes-are-assigned)
- [Transitions and useDeferredValue](#transitions-and-usedeferredvalue)
- [Lane Masking and Batching](#lane-masking-and-batching)
- [Starvation and Expiration](#starvation-and-expiration)
- [Entanglement](#entanglement)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

The Lanes model is the internal priority system React uses to schedule, batch, and interleave units of work. Before React 18, priorities were expressed as a simple integer (the `ExpirationTime` model). That single number made it impossible to represent multiple independent streams of work running concurrently; you had to choose one winner. Lanes replace that with a **32-bit bitmask** where each bit (or small range of bits) represents a distinct priority "lane". Multiple lanes can be active simultaneously, they can be batched by bitwise OR, and the scheduler can pick whichever set of lanes has the highest urgency without abandoning other pending work.

Key capabilities unlocked by lanes:
- **Concurrent rendering** — high-priority user input can interrupt a low-priority transition render
- **Selective hydration** — each Suspense boundary gets its own lane during SSR hydration
- **`startTransition`** — transitions are assigned a separate lane batch and yield to urgent input
- **Automatic batching** — multiple setState calls in the same context merge into one lane

## What Are Lanes

A lane is a bit position in a 31-bit integer (React reserves one bit for signed/unsigned safety). Each lane, or group of bits, represents a priority bucket. Think of a highway: lane 0 is the fast lane for synchronous user interactions, lane 8 is a transition lane, lane 24 is the idle/offscreen lane.

```typescript
// Simplified from packages/react-reconciler/src/ReactFiberLane.ts

export type Lanes = number;  // bitmask — multiple lanes OR'd together
export type Lane  = number;  // single lane — exactly one bit set

// Checking if a lane is included in a set of lanes
export function includesSomeLane(a: Lanes, b: Lanes): boolean {
  return (a & b) !== NoLanes;
}

// Merging two lane sets (union)
export function mergeLanes(a: Lanes, b: Lanes): Lanes {
  return a | b;
}

// Remove a lane from a set
export function removeLanes(set: Lanes, subset: Lanes): Lanes {
  return set & ~subset;
}

// Is `a` a subset of `b`?
export function isSubsetOfLanes(set: Lanes, subset: Lanes): boolean {
  return (set & subset) === subset;
}
```

## Lane Bitmask Layout

```
Bit positions (0 = least significant):

Bit 0:        SyncHydrationLane      (0b000...001)
Bit 1:        SyncLane               (0b000...010)   ← synchronous, urgent
Bit 2:        InputContinuousHydrationLane
Bit 3:        InputContinuousLane    ← drag/scroll events
Bits 4-6:     (reserved)
Bits 7-8:     DefaultHydrationLane / DefaultLane  ← normal setState
Bits 9-14:    TransitionLanes 1-6    ← startTransition
Bits 15-20:   TransitionLanes 7-12
Bit 21:       RetryLane1             ← Suspense retry
Bits 22-23:   RetryLanes 2-3
Bit 24:       SelectiveHydrationLane
Bit 25:       IdleLane               ← requestIdleCallback equivalent
Bit 26:       OffscreenLane          ← hidden Offscreen tree
Bit 27-30:    (unused / reserved)

Lane groups:
  NonIdleLanes   = SyncLane | InputContinuousLane | DefaultLane | TransitionLanes
  InputLanes     = SyncLane | InputContinuousLane
  TransitionLanes = bits 9-20 (12 individual transition slots)
```

```typescript
// Exact constants (React source)
export const NoLanes: Lanes         = 0b0000000000000000000000000000000;
export const NoLane: Lane           = 0b0000000000000000000000000000000;
export const SyncLane: Lane         = 0b0000000000000000000000000000010;
export const InputContinuousLane    = 0b0000000000000000000000000001000;
export const DefaultLane: Lane      = 0b0000000000000000000000100000000;
export const TransitionLane1: Lane  = 0b0000000000000000000010000000000;
// ...TransitionLane2 through TransitionLane12
export const RetryLane1: Lane       = 0b0000000000100000000000000000000;
export const IdleLane: Lane         = 0b0100000000000000000000000000000;
export const OffscreenLane: Lane    = 0b1000000000000000000000000000000;
```

## Priority Levels and Mapping

React has two separate priority systems that must stay in sync: **Scheduler priorities** (from the `scheduler` package — used to schedule tasks on the CPU) and **Lane priorities** (used inside the reconciler). There is a mapping layer between them.

```
Scheduler Priority     →    React Lane
─────────────────────────────────────────────────────
ImmediatePriority      →    SyncLane
UserBlockingPriority   →    InputContinuousLane
NormalPriority         →    DefaultLane
LowPriority            →    TransitionLane (next available)
IdlePriority           →    IdleLane
```

```typescript
// Simplified mapping (ReactFiberRootScheduler.ts)
function lanesToEventPriority(lanes: Lanes): EventPriority {
  const lane = getHighestPriorityLane(lanes);
  if (!isHigherPriorityLane(DefaultLane, lane)) {
    return DefaultEventPriority;  // NormalPriority
  }
  if (!isHigherPriorityLane(InputContinuousLane, lane)) {
    return ContinuousEventPriority; // UserBlockingPriority
  }
  if (includesSomeLane(SyncLane, lane)) {
    return DiscreteEventPriority;   // ImmediatePriority
  }
  return DefaultEventPriority;
}

// Highest priority = lowest bit number (rightmost bit)
export function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes; // isolate least significant set bit
}
```

### Event Priority Assignment

React's event system stamps each event with a lane before dispatching:

```typescript
// Discrete events (click, keydown, focus) → SyncLane
// Continuous events (mousemove, scroll, drag) → InputContinuousLane
// Default (everything else, setTimeout, Promise) → DefaultLane

function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
) {
  const eventPriority = getEventPriority(domEventName);
  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:
      listenerWrapper = dispatchDiscreteEvent;  // sets SyncLane
      break;
    case ContinuousEventPriority:
      listenerWrapper = dispatchContinuousEvent; // sets InputContinuousLane
      break;
    default:
      listenerWrapper = dispatchEvent;           // sets DefaultLane
  }
  return listenerWrapper.bind(null, domEventName, eventSystemFlags, targetContainer);
}
```

## How Lanes Are Assigned

Every call to `setState` (or `useReducer` dispatch, or `startTransition`) goes through `requestUpdateLane`, which determines which lane to assign based on execution context.

```typescript
// Simplified ReactFiberWorkLoop.ts
export function requestUpdateLane(fiber: Fiber): Lane {
  // 1. If we are inside a render phase update, use the current render lane
  const mode = fiber.mode;
  if ((mode & ConcurrentMode) === NoMode) {
    // Legacy synchronous mode — always SyncLane
    return SyncLane;
  }

  // 2. If triggered by a DOM event, use the event's lane
  const isTransition = ReactCurrentBatchConfig.transition !== null;
  if (isTransition) {
    // Inside startTransition — assign the next available transition lane
    const actionScopeLane = peekEntangledActionLane();
    return actionScopeLane !== NoLane
      ? actionScopeLane
      : requestTransitionLane();
  }

  // 3. Use the currently executing update context
  const updateLane: Lane = getCurrentUpdatePriority();
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // 4. Fall back to event priority
  const eventLane: Lane = getCurrentEventPriority();
  return eventLane;
}

// Transition lanes are round-robined to spread work
let nextTransitionLane: Lane = TransitionLane1;
function requestTransitionLane(): Lane {
  const lane = nextTransitionLane;
  // Shift to next transition lane, wrap around at the end
  nextTransitionLane <<= 1;
  if ((nextTransitionLane & TransitionLanes) === 0) {
    nextTransitionLane = TransitionLane1;
  }
  return lane;
}
```

## Transitions and useDeferredValue

`startTransition` and `useDeferredValue` are the primary developer-facing APIs that interact with lanes.

### startTransition

```typescript
import { startTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    // Urgent update: keep the input responsive (SyncLane)
    setQuery(e.target.value);

    // Non-urgent update: filtering 10k items (TransitionLane)
    startTransition(() => {
      setResults(filterItems(e.target.value));
    });
  }

  // If the user types quickly, React may interrupt the transition render
  // and re-render with the latest query before finishing the old transition.
  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultList results={results} />
    </>
  );
}
```

Internally, `startTransition` sets `ReactCurrentBatchConfig.transition` to a non-null object, which causes `requestUpdateLane` to assign a TransitionLane instead of DefaultLane.

```typescript
// ReactStartTransition.ts (simplified)
export function startTransition<S>(
  setPending: (pending: boolean) => void,
  callback: () => void,
): void {
  const previousPriority = getCurrentUpdatePriority();
  setCurrentUpdatePriority(
    higherEventPriority(previousPriority, ContinuousEventPriority),
  );

  // Mark the transition scope
  const prevTransition = ReactCurrentBatchConfig.transition;
  const currentTransition: BatchConfigTransition = {};
  ReactCurrentBatchConfig.transition = currentTransition;

  setPending(true); // urgent — keeps current lane
  try {
    setPending(false); // inside transition scope — gets TransitionLane
    callback();       // setState calls here get TransitionLane
  } finally {
    setCurrentUpdatePriority(previousPriority);
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

### useDeferredValue

`useDeferredValue` defers a value to a transition lane. It creates two renders: an urgent one with the old value and a deferred one with the new value, allowing the UI to keep showing stale content while the new content is being prepared.

```typescript
function TypeaheadResults({ query }: { query: string }) {
  // deferredQuery lags behind query during transitions
  const deferredQuery = useDeferredValue(query);

  // During typing: query is new, deferredQuery is stale
  // React renders immediately with old deferredQuery (no jank)
  // Then re-renders in a transition lane with new deferredQuery
  const isStale = query !== deferredQuery;

  return (
    <div style={{ opacity: isStale ? 0.7 : 1 }}>
      <Results query={deferredQuery} />
    </div>
  );
}
```

## Lane Masking and Batching

One of the core operations in the scheduler is picking which lanes to work on in a given render. React does this by masking the root's `pendingLanes` field.

```typescript
// Root has a bitmask of all work that needs to be done
root.pendingLanes |= lane;  // schedule new work
root.pendingLanes |= lane2; // more work arrives

// When scheduling, pick the highest-priority subset
function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) {
    return NoLanes;
  }

  let nextLanes = NoLanes;

  // Non-idle work has higher priority than idle work
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    // Pick the highest priority among non-idle lanes
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~root.suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // All non-idle lanes are suspended — try pinged lanes
      const nonIdlePingedLanes = nonIdlePendingLanes & root.pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // Only idle work remains
    const unblockedLanes = pendingLanes & ~root.suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else {
      // Check pinged lanes for suspended idle work
      if (root.pingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(root.pingedLanes);
      }
    }
  }

  if (nextLanes === NoLanes) {
    return NoLanes;
  }

  // If we're already in a concurrent render, include any lanes that
  // overlap with the current work-in-progress lanes (avoid thrashing)
  if (
    wipLanes !== NoLanes &&
    wipLanes !== nextLanes &&
    (wipLanes & root.suspendedLanes) === NoLanes
  ) {
    const nextLane = getHighestPriorityLane(nextLanes);
    const wipLane  = getHighestPriorityLane(wipLanes);
    if (
      nextLane >= wipLane ||
      (nextLane === DefaultLane && (wipLane & TransitionLanes) !== 0)
    ) {
      // The next lanes are no more urgent than what we're doing, keep going
      return wipLanes;
    }
  }

  return nextLanes;
}
```

### Automatic Batching (React 18)

Before React 18, batching only happened inside React event handlers. After React 18, all setState calls within the same microtask are batched by merging their lanes:

```typescript
// All three updates end up in the same DefaultLane render
setTimeout(() => {
  setCount(c => c + 1);  // DefaultLane
  setFlag(f => !f);       // DefaultLane (same batch)
  setName('new');         // DefaultLane (same batch)
  // React 18: one re-render, not three
}, 0);

// Opt out of batching when needed
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // forces synchronous render
flushSync(() => setFlag(f => !f));     // another synchronous render
```

## Starvation and Expiration

A concern with priority-based scheduling is starvation: low-priority work might never run if high-priority work keeps arriving. React prevents this by tracking how long each lane has been pending and expiring it when a threshold is exceeded.

```typescript
// Expiration times per lane (milliseconds since React started)
const TransitionLaneExpirationMs = 5000;  // 5 seconds
const RetryLaneExpirationMs = 5000;
const DefaultLaneExpirationMs = 5000;
const InputContinuousLaneExpirationMs = 1000; // 1 second

// Stored per root
root.expirationTimes[laneIndex] = now() + getExpirationTimeForLane(lane);

// When checking if a lane is expired
function includesExpiredLane(root: FiberRoot, lanes: Lanes): boolean {
  const expiredLanes = root.expiredLanes;
  return (expiredLanes & lanes) !== NoLanes;
}

// Expired lanes are included in the sync work check:
// if any lane has expired, React treats the entire render as synchronous
// (it won't yield to the browser mid-render to avoid showing stale UI)
```

```typescript
// How expiration promotes a transition to synchronous
function markStarvedLanesAsExpired(root: FiberRoot, currentTime: number): void {
  let lanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  let lanes2 = lanes;
  while (lanes2 > 0) {
    const index = pickArbitraryLaneIndex(lanes2);
    const lane  = 1 << index;

    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      // This lane has no expiration yet — compute one if it's not suspended
      if ((lane & suspendedLanes) === NoLanes || (lane & pingedLanes) !== NoLanes) {
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      // Lane has expired — promote it
      root.expiredLanes |= lane;
    }
    lanes2 &= ~lane;
  }
}
```

## Entanglement

Entanglement is a mechanism that forces certain lanes to always be rendered together, even if they have different priorities. This prevents inconsistent UI states.

```typescript
// Example: Two stores that must stay in sync
// If you update StoreA on a transition lane and StoreB on SyncLane,
// React entangles them so they render at the same time.

// entangleLanes links lane B to lane A's render
function entangleLanes(root: FiberRoot, entangledLane: Lane): void {
  root.entangledLanes |= entangledLane;
  const entanglements = root.entanglements;
  let lanes = root.pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane  = 1 << index;
    entanglements[index] |= entangledLane;
    lanes &= ~lane;
  }
}

// When React picks nextLanes, it adds entangled lanes:
nextLanes = mergeLanes(nextLanes, root.entangledLanes & nextLanes);
```

Real-world example: `useSyncExternalStore` — when an external store update arrives on a transition lane but React detects tearing risk, it entangles the update to SyncLane so it flushes synchronously.

## Debugging Lanes

```typescript
// Human-readable lane names (React DevTools uses these)
const LaneNames: Record<number, string> = {
  [SyncLane]:              'SyncLane',
  [InputContinuousLane]:   'InputContinuousLane',
  [DefaultLane]:           'DefaultLane',
  [TransitionLane1]:       'TransitionLane1',
  // ...
  [IdleLane]:              'IdleLane',
  [OffscreenLane]:         'OffscreenLane',
};

function laneToString(lane: Lane): string {
  return LaneNames[lane] ?? `Lane(${lane.toString(2).padStart(31, '0')})`;
}

// React DevTools Profiler shows lane information per commit
// Look for "Transition" vs "Sync" labels on renders in the flamegraph
```

## Common Pitfalls

### Pitfall 1: Wrapping Urgent Updates in startTransition

```typescript
// BAD: Don't wrap the input value setter in a transition
function SearchBar() {
  const [query, setQuery] = useState('');

  return (
    <input
      value={query}
      onChange={e => {
        // This would make typing laggy — the visible input update is delayed!
        startTransition(() => setQuery(e.target.value));
      }}
    />
  );
}

// GOOD: Keep the controlled value update urgent; only defer the expensive work
function SearchBar() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  return (
    <input
      value={query}
      onChange={e => {
        setQuery(e.target.value); // urgent
        startTransition(() => setResults(search(e.target.value))); // deferred
      }}
    />
  );
}
```

### Pitfall 2: Expecting isPending to Always Be True

```typescript
function TransitionDemo() {
  const [isPending, startTransition] = useTransition();
  const [data, setData] = useState(null);

  // isPending is true only while the transition is still rendering
  // If the update finishes instantly (no async, no Suspense), isPending flips
  // back to false almost immediately — you may never see the loading state

  async function handleClick() {
    startTransition(async () => {
      const result = await fetchData(); // async action in React 19
      setData(result);
    });
  }
}
```

### Pitfall 3: Starving Transitions by Overloading Them

```typescript
// BAD: Every keystroke creates a new transition on the same lane
// With fast typing, transitions pile up and may expire together,
// causing a big synchronous flush that blocks the main thread

function BadAutocomplete({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');
  const [filtered, setFiltered] = useState(items);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
    startTransition(() => {
      // 10,000-item sort inside a transition — still expensive
      setFiltered(items.filter(i => i.includes(e.target.value)).sort());
    });
  };
  // ...
}

// BETTER: Combine with useMemo for purely derived state
function GoodAutocomplete({ items }: { items: string[] }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  // Pure derivation — no setState needed for the filtered list
  const filtered = useMemo(
    () => items.filter(i => i.includes(deferredQuery)).sort(),
    [items, deferredQuery],
  );
  // ...
}
```

### Pitfall 4: Confusing DefaultLane and TransitionLane

```typescript
// DefaultLane: normal setState calls (setTimeout, Promise.then, etc.)
// NOT the same as transitions — DefaultLane can still block the main thread
// if the component tree is large enough

setTimeout(() => {
  setLargeList(newData); // DefaultLane, not a transition
  // This WILL block if rendering 10,000 rows synchronously
}, 0);

// Wrap in startTransition to get interruptible rendering:
setTimeout(() => {
  startTransition(() => setLargeList(newData)); // TransitionLane — interruptible
}, 0);
```

## Interview Questions

### Question 1: Why did React replace ExpirationTime with Lanes?

**Answer**: The ExpirationTime model used a single integer to represent priority. This made it impossible to have two independent streams of updates at different priorities running concurrently — you had to pick one. The Lanes model uses a bitmask, so multiple priority levels can be represented simultaneously. React can check if a high-priority lane is pending (without discarding a lower-priority in-progress render), batch related lanes together, and use bit operations for efficient lane manipulation. This is foundational to concurrent features like `startTransition` and selective hydration.

### Question 2: What is the difference between SyncLane and DefaultLane?

**Answer**: `SyncLane` is for discrete user events like clicks and key presses — work that must render synchronously without ever yielding to the browser. `DefaultLane` is for everything else: `setTimeout`, `Promise.then`, network responses. In concurrent mode, DefaultLane work can still be interrupted by higher-priority SyncLane work, but DefaultLane itself is not interruptible by transitions. Both are higher priority than TransitionLanes.

### Question 3: How does startTransition use lanes internally?

**Answer**: Calling `startTransition` sets `ReactCurrentBatchConfig.transition` to a non-null object. When `setState` is called inside the callback, `requestUpdateLane` sees the non-null transition context and calls `requestTransitionLane()`, which returns the next available TransitionLane bit (1 of 12 transition lane slots, chosen round-robin). This lane has lower priority than DefaultLane, so the scheduler will always run urgent SyncLane and DefaultLane work first. If new urgent work arrives mid-transition render, React discards the in-progress transition work and restarts it after the urgent work commits.

### Question 4: What is lane entanglement and when does React use it?

**Answer**: Entanglement forces multiple lanes to be rendered together. React uses it when independently-scheduled updates share state that must stay consistent. A key use case is `useSyncExternalStore`: if an external store update arrives on a low-priority lane but React detects that the committed tree already used a different snapshot (tearing), it entangles the update to `SyncLane` to force an immediate synchronous re-render. Entanglement is also used in `useFormState`/`useActionState` where transitions within a form must complete in a predictable order.

### Question 5: How does React prevent transition starvation?

**Answer**: Each lane gets an expiration timestamp when it is first scheduled. For transition lanes, the default expiration window is 5 seconds. React calls `markStarvedLanesAsExpired` on each scheduler task. If the current time has passed a lane's expiration timestamp, that lane is added to `root.expiredLanes`. When React processes expired lanes, it treats the render as synchronous (does not yield to the browser), ensuring the transition always completes eventually even if high-priority work kept interrupting it.

## Key Takeaways

1. **Lanes are a bitmask**: 31 bits, each bit or group of bits representing a priority bucket — enables simultaneous tracking of multiple independent work streams

2. **Priority hierarchy**: SyncLane > InputContinuousLane > DefaultLane > TransitionLanes > RetryLanes > IdleLane > OffscreenLane

3. **startTransition assigns TransitionLane**: work in that lane is interruptible by anything with a lower lane index (higher priority)

4. **getHighestPriorityLane uses `lanes & -lanes`**: isolates the least significant set bit — lowest bit index = highest priority

5. **Automatic batching merges lanes**: multiple `setState` calls in the same microtask accumulate into one lane set and trigger a single render

6. **Starvation prevention via expiration**: lanes that have been pending too long are promoted to synchronous to guarantee eventual completion

7. **Entanglement ensures consistency**: lanes that share interdependent state are forced to render together

8. **Debugging**: React DevTools Profiler labels each commit with its lane type — look for "Transition" vs "Default" vs "Sync" on the timeline

## Resources

- [React RFC: Lanes](https://github.com/facebook/react/issues/18796)
- [React source: ReactFiberLane.ts](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberLane.ts)
- [React source: ReactFiberWorkLoop.ts](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.ts)
- [React 18 Working Group: Automatic Batching](https://github.com/reactwg/react-18/discussions/21)
- ["Scheduling in React" by Philipp Spiess (Smashing Magazine)](https://www.smashingmagazine.com/2020/07/react-concurrent-features/)
- [React DevTools Profiler Documentation](https://react.dev/learn/react-developer-tools)
