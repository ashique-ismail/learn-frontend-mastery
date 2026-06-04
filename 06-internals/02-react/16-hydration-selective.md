# React Hydration: Selective Hydration Internals

## Table of Contents
- [Overview](#overview)
- [What Is Hydration](#what-is-hydration)
- [Hydration Algorithm Step-by-Step](#hydration-algorithm-step-by-step)
- [Selective Hydration](#selective-hydration)
- [Hydration Mismatch Detection](#hydration-mismatch-detection)
- [Suspense and Hydration](#suspense-and-hydration)
- [Server/Client Reconciliation](#serverclient-reconciliation)
- [Streaming SSR and Out-of-Order Hydration](#streaming-ssr-and-out-of-order-hydration)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Hydration is the process of attaching React's event system and component state to HTML that was pre-rendered on the server. Instead of throwing away the server HTML and re-rendering from scratch (which would cause a flash of unstyled or blank content), React "claims" the existing DOM nodes — attaching fiber references, wiring up event delegation, and verifying the tree matches expectations.

React 18 introduced **selective hydration**: Suspense boundaries act as hydration units. Instead of blocking the entire page's hydration behind one slow component, each Suspense boundary can be hydrated independently. If a boundary's JavaScript chunk hasn't loaded yet, React skips it and hydrates the rest of the page. When the chunk arrives, that boundary is hydrated in isolation. If the user interacts with an un-hydrated section, React prioritizes hydrating that boundary first.

## What Is Hydration

```
Server renders HTML:
┌─────────────────────────────────────┐
│ <div id="root">                     │
│   <header>...</header>              │
│   <main>                            │
│     <article>...</article>          │
│     <aside>...</aside>              │  ← DOM exists in HTML, no React state
│   </main>                           │
│ </div>                              │
└─────────────────────────────────────┘
         ↓ browser parses HTML (fast)
         ↓ React JS bundle loads
         ↓ hydrateRoot() called

Hydration:
┌─────────────────────────────────────┐
│ <div id="root">                     │
│   <header> ←── fiber attached       │
│   <main>   ←── fiber attached       │
│     <article> ←── fiber attached    │
│     <aside>   ←── fiber attached    │
│   </main>                           │
│ </div>                              │
└─────────────────────────────────────┘
  Events wired. State initialized. React controls the tree.
```

```typescript
// Entry point for SSR hydration
import { hydrateRoot } from 'react-dom/client';

const root = hydrateRoot(
  document.getElementById('root')!,
  <App />,
  {
    onRecoverableError(error, errorInfo) {
      // Called when React recovers from a hydration mismatch
      console.error('Hydration error:', error, errorInfo.componentStack);
    },
  },
);
```

## Hydration Algorithm Step-by-Step

### Phase 1: Creating the Fiber Tree in Hydration Mode

```typescript
// ReactFiberReconciler.ts (simplified)
function hydrateRoot(container: Container, children: ReactNodeList): RootType {
  // Create a fiber root in ConcurrentMode + HydrationMode
  const root = createFiberRoot(container, ConcurrentRoot, true /* isDehydrated */);
  // Schedule the initial render with the existing DOM as the "hydration instance"
  updateContainer(children, root, null, null);
  return root;
}
```

```
Fiber tree construction during hydration:

React renders component tree → for each host fiber (div, span, etc.):
  1. Look for a matching DOM node in the existing server HTML
  2. If found: attach the DOM node to the fiber (fiber.stateNode = domNode)
  3. If not found: create a new DOM node (hydration mismatch)
  4. Verify node type and props (warn/recover on mismatch in dev)
  5. Recurse into children
```

### Phase 2: The tryToClaimNextHydratableInstance Walk

```typescript
// ReactFiberHydrationContext.ts (simplified)

let nextHydratableInstance: null | Instance | TextInstance = null;
let isHydrating: boolean = false;

function prepareToHydrateHostInstance(
  fiber: Fiber,
  rootContainerInstance: Container,
): boolean {
  const instance = nextHydratableInstance;
  // Match the fiber's host type against the next DOM node in document order
  if (!instance || !canHydrate(fiber, instance)) {
    // Mismatch: create a new DOM element instead
    warnForDeletedHydratableElement(rootContainerInstance, instance);
    return false;
  }

  // Claim this DOM node for the fiber
  fiber.stateNode = instance;
  // Hydrating: set the fiber reference on the DOM node for event delegation
  (instance as any).__reactFiber = fiber;
  (instance as any).__reactProps = fiber.pendingProps;

  return true;
}

function tryToClaimNextHydratableInstance(fiber: Fiber): void {
  if (!isHydrating) return;

  // Walk the DOM tree in sync with the fiber tree
  const nextInstance = nextHydratableInstance;
  if (!nextInstance) {
    throwOnHydrationMismatch(fiber);
    return;
  }

  const firstAttemptedInstance = nextInstance;
  if (!tryHydrate(fiber, nextInstance)) {
    // Could not hydrate — skip forward in the DOM
    const secondAttemptedInstance = getNextHydratableSibling(nextInstance);
    if (!secondAttemptedInstance || !tryHydrate(fiber, secondAttemptedInstance)) {
      // Two misses — give up on hydration for this subtree
      insertNonHydratedInstance(returnFiber, fiber);
      isHydrating = false;
      return;
    }
    // Successfully hydrated the second sibling — the first must be deleted
    deleteHydratableInstance(returnFiber, firstAttemptedInstance);
  }
}
```

### Phase 3: Event Delegation During Hydration

React 18 changed event delegation from attaching to `document` to attaching to the root container. This matters for hydration because events on un-hydrated subtrees are replayed.

```typescript
// Before a Suspense boundary is hydrated, its subtree has no fiber references.
// React captures events that happen on those DOM nodes (using event delegation
// at the container level), stores them, and replays them after hydration.

// Simplified event capture for un-hydrated regions
function dispatchEventForPluginEventSystem(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  nativeEvent: AnyNativeEvent,
  targetInst: null | Fiber,
  targetContainer: EventTarget,
): void {
  if (targetInst === null) {
    // Target DOM node has no fiber reference — it's un-hydrated
    // Queue this event for replay after hydration
    queuedEventsOnHydrationBoundary.push({ domEventName, nativeEvent, targetContainer });
    // Prioritize hydrating the closest Suspense boundary
    attemptHydrationAtCurrentPriority(targetContainer, nativeEvent);
    return;
  }
  // Normal event dispatch
  dispatchEventsForPlugins(domEventName, eventSystemFlags, nativeEvent, targetInst, targetContainer);
}
```

## Selective Hydration

Selective hydration works by treating each `<Suspense>` boundary as a hydration root. The boundary's content can be hydrated independently of its siblings.

### Dehydrated Fragment Markers

During SSR, React wraps each Suspense boundary's HTML in HTML comments that serve as markers:

```html
<!-- Server-rendered output for a Suspense boundary -->
<!--$-->
<section id="comments">
  <!-- comment list HTML -->
</section>
<!--/$-->

<!-- If the boundary was suspended during SSR: -->
<!--$!-->
<template id="B:0"></template>
<div>Loading...</div>
<!--/$-->
```

The `<!--$-->` / `<!--/$-->` markers tell the client-side hydration walker where each Suspense boundary starts and ends. `<!--$!-->` means the boundary was in a suspended state — it has a fallback in the HTML.

```typescript
// ReactDOMServerRenderer inserts these markers
// ReactFiberHydrationContext reads them during hydrateRoot

function isSuspenseInstancePending(instance: SuspenseInstance): boolean {
  return instance.data === SUSPENSE_PENDING_START_DATA; // '<!--$!-->'
}

function isSuspenseInstanceFallback(instance: SuspenseInstance): boolean {
  return instance.data === SUSPENSE_FALLBACK_START_DATA; // '<!--$?-->'
}
```

### The Hydration Priority Mechanism

```typescript
// When the user interacts with an un-hydrated region, React must
// prioritize hydrating that Suspense boundary before others.

// Each dehydrated Suspense boundary is a special DehydratedFragment fiber
// with a lane assigned to it. User interaction bumps that lane to SyncLane.

function attemptToDispatchEvent(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  rootContainer: Container,
  nativeEvent: AnyNativeEvent,
): null | Container | SuspenseInstance {
  // Walk up the fiber tree from the event target
  const targetInst = getClosestInstanceFromNode(nativeEvent.target);
  
  if (targetInst !== null) {
    const nearestMounted = getNearestMountedFiber(targetInst);
    if (nearestMounted === null) return rootContainer;

    const tag = nearestMounted.tag;
    if (tag === SuspenseComponent) {
      // The nearest mounted ancestor is a Suspense boundary —
      // but its children are not yet hydrated
      const instance = nearestMounted.stateNode;
      if (isSuspenseInstancePending(instance)) {
        // Schedule hydration with high (SyncLane) priority
        return instance; // signal: replay this event after hydrating
      }
    }
  }
  return null;
}
```

```
Selective hydration visualization:

Page HTML (loaded):
┌──────────────────────────────────────────┐
│ <Header/>        ← hydrated first        │
│ <!--$-->                                 │
│   <Comments/>    ← dehydrated (JS pending)│  ← user clicks here
│ <!--/$-->                                │
│ <!--$-->                                 │
│   <Sidebar/>     ← dehydrated (JS pending)│
│ <!--/$-->                                │
└──────────────────────────────────────────┘

Without selective hydration:
- Header hydrates
- Comments hydration blocks (JS not loaded)
- Sidebar waits for Comments to finish
- User click is lost

With selective hydration:
- Header hydrates
- Comments dehydrated boundary registered with SyncLane (user clicked)
- Sidebar dehydration registered with DefaultLane
- Comments JS arrives → hydrated with SyncLane (high priority)
- User click replayed
- Sidebar hydrated with DefaultLane (lower priority)
```

## Hydration Mismatch Detection

React compares server-rendered attributes against what the component renders on the client. Mismatches are handled differently in development vs production.

```typescript
// Dev: warns and potentially recovers
// Production: silently recovers (may cause visual bugs)

function diffHydratedProperties(
  domElement: Element,
  tag: string,
  rawProps: Object,
  rootContainerElement: Element | Document,
): null | Array<mixed> {
  let extraAttributeNames: Set<string> = new Set();
  // ... collect all DOM attributes into extraAttributeNames

  let updatePayload: null | Array<mixed> = null;

  for (const propKey in rawProps) {
    if (!rawProps.hasOwnProperty(propKey)) continue;
    const nextProp = rawProps[propKey];

    if (propKey === 'children') {
      // Text content — check if it matches
      if (typeof nextProp === 'string' || typeof nextProp === 'number') {
        if (domElement.textContent !== String(nextProp)) {
          warnForTextDifference(domElement.textContent, nextProp);
          // Queue an update to fix it
          (updatePayload = updatePayload || []).push('children', nextProp);
        }
      }
    } else if (propKey === 'dangerouslySetInnerHTML') {
      // innerHTML — compare directly
      const serverHTML = domElement.innerHTML;
      const clientHTML = nextProp ? nextProp.__html : undefined;
      if (serverHTML !== normalizeMarkupForTextOrAttribute(clientHTML)) {
        warnForInnerHTMLMismatch(clientHTML, serverHTML);
      }
    } else {
      // Regular attribute — check for mismatch
      const serverValue = domElement.getAttribute(propKey);
      const clientValue = getValueForProperty(domElement, propKey, nextProp, ...);
      if (serverValue !== clientValue) {
        warnForPropDifference(propKey, serverValue, clientValue);
        (updatePayload = updatePayload || []).push(propKey, nextProp);
      }
    }
  }

  // Return an update payload to fix mismatches
  return updatePayload;
}
```

### Common Mismatch Sources

```typescript
// 1. Date/time rendered during SSR
function Timestamp() {
  // BAD: new Date() is different on server vs client
  return <time>{new Date().toLocaleString()}</time>;
}

// GOOD: render on client only, or pass the server timestamp as a prop
function Timestamp({ serverTime }: { serverTime: string }) {
  const [time, setTime] = useState(serverTime); // stable initial value
  useEffect(() => {
    setTime(new Date().toLocaleString()); // update on client after hydration
  }, []);
  return <time>{time}</time>;
}

// 2. Random IDs
function Widget() {
  // BAD: generates different IDs on server and client
  const id = Math.random().toString(36);
  return <div id={id} />;
}

// GOOD: use React 18's useId()
function Widget() {
  const id = useId(); // stable across server/client
  return <div id={id} />;
}

// 3. User-agent dependent rendering
function MobileMenu() {
  // BAD: window is undefined on server
  const isMobile = window?.innerWidth < 768;
  return isMobile ? <MobileNav /> : <DesktopNav />;
}

// GOOD: suppress hydration warning for known-safe differences
function MobileMenu() {
  return (
    <nav suppressHydrationWarning>
      {/* Contents differ on server/client — we know and accept this */}
    </nav>
  );
}
```

## Suspense and Hydration

```typescript
// A Suspense boundary in a hydrating tree can still suspend.
// This happens when the component needs data that wasn't included
// in the server payload.

function UserProfile({ userId }: { userId: string }) {
  // If this throws a Promise during hydration, the Suspense boundary
  // above it captures it and shows the fallback — exactly as in CSR.
  const user = use(fetchUser(userId));
  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId="123" />
    </Suspense>
  );
}
```

When a Suspense boundary suspends during hydration:
1. React shows the fallback (if one exists from the server HTML, it's already there)
2. The boundary is marked as suspended in the fiber tree
3. When the Promise resolves, React re-attempts hydration of that boundary
4. On retry, if the component renders successfully, the fallback is swapped for the content

## Server/Client Reconciliation

```
Full reconciliation flow during hydrateRoot:

hydrateRoot(container, <App />)
    │
    ▼
renderRootConcurrent (HydrationLane)
    │
    ▼
beginWork for each fiber
    │
    ├── Host fibers (div, span, etc.)
    │     tryToClaimNextHydratableInstance
    │     compare props → queue update if mismatch
    │     attach fiber.stateNode = domNode
    │
    ├── Function/class component fibers
    │     render() → produce children
    │     no DOM node to claim
    │
    ├── SuspenseComponent fibers
    │     check for <!--$--> or <!--$!--> markers
    │     if dehydrated: create DehydratedFragment fiber
    │     register in hydration queue by lane priority
    │
    └── Text fibers
          compare textContent
          warn on mismatch
          queue update if different

completeWork phase:
    mount effects (useLayoutEffect, ref attachments)

commitRoot:
    run mutations (apply queued prop updates from mismatches)
    run layout effects
    run passive effects (useEffect)

Post-commit:
    Dispatch replayed events for hydrated boundaries
```

## Streaming SSR and Out-of-Order Hydration

React 18 supports streaming SSR via `renderToPipeableStream`. Slow components can be wrapped in Suspense; their HTML is streamed later and inserted into the DOM via `<script>` tags.

```typescript
// Server-side (Node.js)
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/static/js/main.js'],
    onShellReady() {
      // Send the initial HTML shell immediately (fast first byte)
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    onError(error) {
      console.error(error);
    },
  });
});

// React streams:
// 1. Initial HTML with Suspense fallbacks inline
// 2. As each Suspense boundary resolves:
//    <div hidden id="S:0"><!-- resolved content --></div>
//    <script>$RC("B:0","S:0")</script>  ← React's runtime swaps the fallback
```

```
Streaming timeline:

t=0ms:  Browser receives first HTML chunk (header, nav, above-fold content)
t=0ms:  Browser starts parsing, rendering above-fold content visible
t=20ms: React JS bundle starts loading
t=50ms: React JS bundle ready → hydrateRoot called
t=50ms: React hydrates everything that is already in the DOM
t=100ms: Slow component resolves on server → streamed HTML arrives
t=100ms: React hydrates the newly arrived Suspense boundary
         (using SelectiveHydrationLane)
```

## Common Pitfalls

### Pitfall 1: Suppressing All Hydration Warnings

```typescript
// BAD: suppressing warnings globally hides real bugs
function Root() {
  return <div suppressHydrationWarning>...</div>;
}

// suppressHydrationWarning only suppresses warnings for the
// element it's on, not its children. But wrapping everything
// in a suppressed container means you'll never see real mismatches.

// GOOD: suppress only where you know the difference is intentional
function LocaleAwareTime({ isoString }: { isoString: string }) {
  return (
    <time
      dateTime={isoString}
      suppressHydrationWarning // Only the text content differs (locale formatting)
    >
      {new Date(isoString).toLocaleString()}
    </time>
  );
}
```

### Pitfall 2: Conditional Rendering Based on typeof window

```typescript
// BAD: creates a guaranteed hydration mismatch
function ConditionalFeature() {
  if (typeof window === 'undefined') {
    return null; // server returns null
  }
  return <ClientOnlyWidget />; // client renders widget
}

// GOOD: use a mounted state
function ConditionalFeature() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  if (!mounted) return null; // both server and client return null initially
  return <ClientOnlyWidget />;
}

// ALSO GOOD: use next/dynamic or React.lazy with ssr:false
```

### Pitfall 3: Not Handling Hydration Errors

```typescript
// hydrateRoot onRecoverableError fires for recoverable mismatches
// but some mismatches cause complete subtree remounts in production.
// Monitor these in your error tracking.

const root = hydrateRoot(container, <App />, {
  onRecoverableError(error, errorInfo) {
    // Log to Sentry, Datadog, etc.
    reportError({
      message: error.message,
      componentStack: errorInfo.componentStack,
      digest: errorInfo.digest, // server-side error ID for correlation
    });
  },
});
```

### Pitfall 4: Mismatched Keys During Hydration

```typescript
// BAD: key depends on client-side state
function ItemList({ items }: { items: Item[] }) {
  const [version, setVersion] = useState(0);
  return (
    <ul>
      {items.map(item => (
        // key includes version — server and client diverge immediately
        <li key={`${item.id}-${version}`}>{item.name}</li>
      ))}
    </ul>
  );
}

// GOOD: key is stable and consistent between server and client
function ItemList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

## Interview Questions

### Question 1: What is the difference between hydration and a full client-side render?

**Answer**: In a full client-side render, React creates DOM nodes from scratch. In hydration, React walks the existing server-rendered DOM tree simultaneously with its virtual fiber tree, "claiming" DOM nodes by attaching fiber references to them. React verifies props match expectations and queues updates for mismatches, but does not destroy and recreate elements. This makes hydration much faster than a fresh render — the DOM is already there, and React just needs to attach its event system and state management.

### Question 2: What is selective hydration and why does it matter?

**Answer**: Selective hydration (React 18) treats each Suspense boundary as an independent hydration unit. The entire page's hydration is not blocked waiting for a single slow component's JavaScript to load. React hydrates what it can, registers dehydrated boundaries with their own lane priorities, and hydrates each boundary independently as its code arrives. Critically, if a user interacts with an un-hydrated boundary, React promotes that boundary's hydration lane to SyncLane so it is hydrated immediately, and the original event is replayed.

### Question 3: How does React detect hydration mismatches?

**Answer**: During `beginWork` for host fibers (div, span, etc.), React calls `tryToClaimNextHydratableInstance` which attempts to match the fiber against the next DOM node in document order. If the node's tag or key props differ significantly, React logs a warning (in development) and may create a new DOM node. For attribute mismatches on claimed nodes, `diffHydratedProperties` compares each prop against the DOM attribute value and queues an update payload. In production, React silently recovers from recoverable mismatches but calls `onRecoverableError`.

### Question 4: How do the HTML comment markers `<!--$-->` work in streaming SSR?

**Answer**: During `renderToPipeableStream`, every Suspense boundary wraps its content in `<!--$-->…<!--/$-->` markers. If a boundary suspended during SSR, the fallback content is emitted with `<!--$!-->` or `<!--$?-->` markers. On the client, `hydrateRoot`'s DOM walk recognizes these comment nodes and creates `DehydratedFragment` fibers for them. Each such fiber is tracked in the hydration queue with a lane. When a streamed HTML chunk arrives via `<script>$RC(...)`, React's runtime moves the resolved content into place in the DOM and schedules hydration of that boundary at `SelectiveHydrationLane`.

### Question 5: How does useId prevent hydration mismatches for generated IDs?

**Answer**: `useId` generates IDs from the component's position in the fiber tree, which is deterministic. Both the server renderer and the client renderer traverse the fiber tree in the same order, so they produce identical ID sequences. The ID format is `:r0:`, `:r1:`, etc. — a base-32 encoding of the tree traversal path. This is in contrast to `Math.random()` or a counter that resets at module load, both of which produce different values on server and client.

## Key Takeaways

1. **Hydration claims DOM, not rebuilds it**: React walks the server HTML in sync with the fiber tree, attaching fiber references to existing nodes — far cheaper than a full client render

2. **Selective hydration = Suspense as hydration boundary**: Each `<Suspense>` is a separate, independently schedulable hydration unit

3. **User interaction triggers priority hydration**: clicks on un-hydrated regions bump their Suspense boundary's lane to SyncLane; the event is stored and replayed post-hydration

4. **Comment markers enable streaming**: `<!--$-->` and `<!--$!-->` in the HTML tell the client hydration walker where each Suspense boundary is and whether it had content or a fallback

5. **Mismatch recovery is silent in production**: React recovers from prop mismatches by queuing an update, but does not throw — use `onRecoverableError` to monitor these

6. **useId solves the ID mismatch problem**: fiber-tree-position-based IDs are identical on server and client, eliminating the most common source of hydration warnings

7. **typeof window guards cause mismatches**: use `useEffect` + state to defer client-only rendering to after hydration

8. **suppressHydrationWarning is element-scoped**: it suppresses warnings only on that element's text content, not on children

## Resources

- [React 18: Selective Hydration](https://github.com/reactwg/react-18/discussions/130)
- [New Suspense SSR Architecture (Manu Mtz-Almeida RFC)](https://github.com/reactjs/rfcs/blob/main/text/0213-suspense-in-react-18.md)
- [React source: ReactFiberHydrationContext.ts](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHydrationContext.ts)
- [React source: ReactDOMServerRenderer.ts](https://github.com/facebook/react/blob/main/packages/react-dom/src/server/)
- ["Demystifying React Hydration" by Shaundai Person (React Conf 2022)](https://www.youtube.com/watch?v=0mnKFoUs3iA)
- [Next.js Hydration errors guide](https://nextjs.org/docs/messages/react-hydration-error)
