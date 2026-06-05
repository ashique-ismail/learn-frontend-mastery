# Event Model: Bubbling, Capturing, Delegation, and Passive Listeners

## The Idea

**In plain English:** When you interact with a webpage — like clicking a button — the browser fires an "event," which is just a signal that something happened. JavaScript lets you listen for these signals and run code in response; the "event model" is the set of rules that controls how those signals travel through the page and which listeners get notified.

**Real-world analogy:** Imagine dropping a pebble into a pond. The ripple starts at the exact spot the pebble landed, spreads outward to the edges, and you can place your hand anywhere along the way to feel it pass.

- The pebble landing = the user clicking a specific element (the event target)
- The outward ripple = the event bubbling up through parent elements toward the top of the page
- Your hand feeling the ripple = an event listener attached to any element along the path

---

## Overview

The browser event model defines how events are dispatched through the DOM and how handlers are invoked. Understanding the three-phase propagation model—capturing, target, and bubbling—along with techniques like event delegation and passive listeners, is essential for writing performant, maintainable event-driven code. These concepts are consistently tested in senior JavaScript interviews.

## Event Propagation Phases

When an event is triggered on an element, it travels through three distinct phases:

1. **Capture phase**: The event descends from `window` down to the target element.
2. **Target phase**: The event reaches the target element itself.
3. **Bubble phase**: The event ascends from the target back up to `window`.

```javascript
// Visualizing all three phases
document.addEventListener('click', () => console.log('1. window capture'), true);
document.documentElement.addEventListener('click', () => console.log('2. html capture'), true);
document.body.addEventListener('click', () => console.log('3. body capture'), true);

const btn = document.querySelector('button');
btn.addEventListener('click', () => console.log('4. button capture'), true);
btn.addEventListener('click', () => console.log('5. button bubble'));

document.body.addEventListener('click', () => console.log('6. body bubble'));
document.documentElement.addEventListener('click', () => console.log('7. html bubble'));
document.addEventListener('click', () => console.log('8. window bubble'));

// Clicking the button outputs 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
```

## addEventListener

```javascript
element.addEventListener(type, listener, options);
element.addEventListener(type, listener, useCapture); // older signature
```

### Options Object

```javascript
element.addEventListener('click', handler, {
  capture: false, // listen during capture phase (default: false = bubble phase)
  once: true,     // auto-remove after first invocation
  passive: true,  // listener will never call preventDefault() (perf hint)
  signal: controller.signal, // AbortController integration
});
```

### Removing Listeners

```javascript
// Must pass the same listener reference
const handler = (e) => console.log(e.type);
el.addEventListener('click', handler);
el.removeEventListener('click', handler);

// Arrow functions assigned to variables work; anonymous inline functions do NOT
el.addEventListener('click', (e) => console.log(e)); // cannot remove!

// Modern: use AbortController
const controller = new AbortController();
el.addEventListener('click', handler, { signal: controller.signal });
controller.abort(); // removes the listener
```

## The Event Object

```javascript
element.addEventListener('click', (event) => {
  // Identity
  event.type;           // 'click'
  event.target;         // element that was clicked (origin)
  event.currentTarget;  // element the handler is attached to
  event.eventPhase;     // 1=capture, 2=target, 3=bubble

  // Propagation control
  event.stopPropagation();       // stops further propagation
  event.stopImmediatePropagation(); // stops propagation AND other handlers on same element

  // Default action
  event.preventDefault();        // cancels default browser behavior
  event.defaultPrevented;        // true if preventDefault was called

  // Mouse specifics
  event.clientX; event.clientY;  // relative to viewport
  event.pageX; event.pageY;      // relative to document
  event.screenX; event.screenY;  // relative to screen

  // Keyboard specifics
  event.key;     // 'Enter', 'Escape', 'a', etc.
  event.code;    // 'KeyA', 'ArrowUp', physical key
  event.altKey; event.ctrlKey; event.shiftKey; event.metaKey;
});
```

### target vs. currentTarget

```javascript
document.querySelector('ul').addEventListener('click', (e) => {
  console.log(e.target);        // the <li> or <a> that was clicked
  console.log(e.currentTarget); // always the <ul> where the handler lives
});
```

This distinction is the foundation of event delegation.

## Event Bubbling

Most events bubble by default, propagating from the target element up through ancestors to `window`. Notable exceptions that do NOT bubble: `focus`, `blur`, `mouseenter`, `mouseleave`, `load`, `unload`, `scroll` (on elements).

```javascript
// Bubbling demonstration
document.querySelector('.child').addEventListener('click', () => {
  console.log('child clicked');
});

document.querySelector('.parent').addEventListener('click', () => {
  console.log('parent also fires due to bubbling');
});

// Stopping bubble
document.querySelector('.child').addEventListener('click', (e) => {
  e.stopPropagation();
  console.log('child clicked — parent handler will NOT fire');
});
```

## Event Capturing

Capture-phase listeners fire before bubble-phase listeners on ancestor elements. Useful for intercepting events before they reach their target.

```javascript
// This fires before any bubble-phase handlers lower in the tree
document.addEventListener('click', (e) => {
  console.log('capture on document — fires first');
}, true); // <-- true enables capture

// Practical use: intercept all link clicks globally
document.addEventListener('click', (e) => {
  const link = e.target.closest('a[href]');
  if (link) {
    e.preventDefault();
    router.navigate(link.href); // SPA navigation
  }
}, true);
```

## Event Delegation

Instead of attaching handlers to each child element, attach a single handler to a common ancestor and use `event.target` to identify the source.

```javascript
// Without delegation — O(n) listeners, breaks on dynamic content
document.querySelectorAll('.item').forEach(item => {
  item.addEventListener('click', handleClick);
});

// With delegation — one listener, works for dynamically added items
document.querySelector('.list').addEventListener('click', (e) => {
  const item = e.target.closest('.item');
  if (!item) return; // click was outside an .item

  handleClick(item);
});
```

### Delegation with Data Attributes

```javascript
document.querySelector('[data-actions]').addEventListener('click', (e) => {
  const btn = e.target.closest('[data-action]');
  if (!btn) return;

  const action = btn.dataset.action;
  const id = btn.dataset.id;

  switch (action) {
    case 'delete': deleteItem(id); break;
    case 'edit':   editItem(id);   break;
    case 'share':  shareItem(id);  break;
  }
});
```

### When Delegation Has Limits

```javascript
// stopPropagation inside a child breaks delegation
innerEl.addEventListener('click', (e) => {
  e.stopPropagation(); // delegation on ancestor will never see this click
});

// Events that don't bubble can't be delegated with standard bubbling
// Use capture phase instead:
document.addEventListener('focus', handler, true); // captures focus for delegation
```

## Passive Listeners

Some events—particularly `touchstart`, `touchmove`, and `wheel`—have a default action (scrolling) that the browser cannot begin until it checks whether any listener calls `preventDefault()`. Marking a listener `passive: true` tells the browser it can scroll immediately, improving scroll performance.

```javascript
// Without passive: browser waits to see if you block scrolling
window.addEventListener('touchmove', onMove);

// With passive: browser scrolls immediately, calls handler concurrently
window.addEventListener('touchmove', onMove, { passive: true });

// Attempting preventDefault inside a passive listener throws a warning
window.addEventListener('touchstart', (e) => {
  e.preventDefault(); // Warning: Unable to preventDefault inside passive event listener
}, { passive: true });
```

### Passive Defaults in Modern Browsers

Browsers now default `touchstart`, `touchmove`, `wheel`, and `mousewheel` on `window`, `document`, and `body` to `passive: true`. To override and block scrolling you must explicitly opt out:

```javascript
window.addEventListener('touchstart', (e) => {
  e.preventDefault(); // allowed now
}, { passive: false });
```

## once Option

```javascript
// Listener automatically removes itself after first invocation
button.addEventListener('click', () => {
  console.log('clicked once');
}, { once: true });

// Equivalent manual approach
function handleOnce(e) {
  console.log('clicked once');
  e.currentTarget.removeEventListener('click', handleOnce);
}
button.addEventListener('click', handleOnce);
```

## AbortController for Listener Cleanup

```javascript
class Component {
  constructor(el) {
    this.el = el;
    this.controller = new AbortController();
    const { signal } = this.controller;

    el.addEventListener('click', this.onClick, { signal });
    el.addEventListener('keydown', this.onKeydown, { signal });
    window.addEventListener('resize', this.onResize, { signal });
  }

  destroy() {
    this.controller.abort(); // removes ALL listeners registered with this signal
  }

  onClick = (e) => { /* ... */ };
  onKeydown = (e) => { /* ... */ };
  onResize = () => { /* ... */ };
}
```

## Comparison Table

| Technique | Phase | Use Case |
| --------- | ----- | -------- |
| Bubble listener (default) | Bubble | Most event handling |
| Capture listener | Capture | Intercept before target, global guards |
| Event delegation | Bubble | Many children, dynamic lists |
| `stopPropagation` | - | Isolate component from parent handlers |
| `stopImmediatePropagation` | - | Prevent other handlers on same element |
| `passive: true` | - | Touch/wheel events, scroll performance |
| `once: true` | - | One-shot interactions |
| `AbortController` | - | Bulk cleanup, component teardown |

## Best Practices

### 1. Prefer Delegation for Lists

```javascript
// Good: single handler, works for future items
listEl.addEventListener('click', (e) => {
  if (e.target.matches('.delete-btn')) deleteItem(e.target.closest('.item'));
});
```

### 2. Always Clean Up Listeners

```javascript
// Components that mount/unmount must remove listeners
class Modal {
  open() {
    this.ac = new AbortController();
    document.addEventListener('keydown', this.handleKeydown, { signal: this.ac.signal });
  }
  close() {
    this.ac.abort();
  }
}
```

### 3. Mark Scroll Handlers Passive

```javascript
document.addEventListener('scroll', updateScrollIndicator, { passive: true });
window.addEventListener('wheel', zoom, { passive: true });
```

### 4. Avoid stopPropagation in Libraries

```javascript
// Third-party code that calls stopPropagation breaks other code
// Prefer custom events or flags to signal handled state
```

## Interview Questions

### Q1: What is the difference between event.target and event.currentTarget?

**Answer:** `event.target` is the element that originally dispatched the event (where the user clicked or typed). `event.currentTarget` is the element to which the currently executing handler is attached. During bubbling, `currentTarget` changes as the event moves up the DOM, while `target` always remains the originating element. This difference is what makes event delegation work.

### Q2: How does event delegation work, and why is it preferred?

**Answer:** Event delegation attaches a single listener to a parent element instead of individual listeners on each child. Because events bubble up, the parent's handler fires for clicks on any descendant. The handler identifies the specific source using `event.target` and `closest()`. This approach uses less memory (one listener vs. hundreds), works automatically for dynamically added children, and simplifies cleanup.

### Q3: Why do passive event listeners improve scroll performance?

**Answer:** For scroll-triggering events (`touchmove`, `wheel`), the browser must wait for each listener to complete before starting a scroll to check if `preventDefault()` was called. With `passive: true`, the browser is told upfront that `preventDefault()` will not be called, so it can begin scrolling immediately on a separate compositor thread—typically reducing input latency from ~16ms to near zero.

### Q4: What is the difference between stopPropagation and stopImmediatePropagation?

**Answer:** `stopPropagation()` prevents the event from traveling to ancestor (or descendant, in capture) elements but allows other handlers on the current element to run. `stopImmediatePropagation()` additionally cancels any remaining handlers attached to the same element (in registration order). Use `stopImmediatePropagation` when you need to fully shut down event handling at one node.

### Q5: How would you implement a one-time listener without the once option?

**Answer:** Assign the handler to a named variable and call `removeEventListener` within the handler, passing the same reference. The `once: true` option introduced in modern browsers does exactly this automatically.

## Common Pitfalls

### 1. Anonymous Functions Cannot Be Removed

```javascript
el.addEventListener('click', function() { doThing(); });
el.removeEventListener('click', function() { doThing(); }); // new reference, fails silently
```

### 2. Forgetting capture Phase Matters for removeEventListener

```javascript
el.addEventListener('click', handler, true);     // capture
el.removeEventListener('click', handler);        // bubble — does NOT remove!
el.removeEventListener('click', handler, true);  // capture — correct
```

### 3. Calling stopPropagation Too Broadly

```javascript
// Breaks analytics, tooltips, dropdowns, and any ancestor-level listeners
modal.addEventListener('click', (e) => e.stopPropagation());
```

### 4. Missing Return Value from Default-Prevention Check

```javascript
form.addEventListener('submit', (e) => {
  if (!validate()) {
    e.preventDefault();
    // not returning here — code below still runs
    submitToServer(); // runs anyway
  }
});
```

### 5. Event Listener Accumulation in Frameworks

```javascript
// Re-mounting a component without cleanup adds duplicate listeners
function mount() {
  window.addEventListener('resize', onResize);
}
function unmount() {
  // forgetting this causes memory leaks and multiple handler calls
  window.removeEventListener('resize', onResize);
}
```

## Resources

- [MDN: Event reference](https://developer.mozilla.org/en-US/docs/Web/Events)
- [MDN: EventTarget.addEventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)
- [MDN: Event bubbling and capture](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events#event_bubbling_and_capture)
- [Google Developers: Passive Event Listeners](https://developers.google.com/web/updates/2016/06/passive-event-listeners)
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
