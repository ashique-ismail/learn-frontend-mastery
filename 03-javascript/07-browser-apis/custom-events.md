# Custom Events

## Overview

Custom events allow JavaScript to implement a publish-subscribe communication pattern on top of the native browser event system. Rather than coupling components through direct method calls or shared state, components can dispatch events on DOM elements and any interested party can listen. This is a first-class browser API—`CustomEvent` and `EventTarget` are built-in—and understanding them deeply demonstrates familiarity with component architecture, decoupling, and the event-driven model that underpins the browser.

## CustomEvent

`CustomEvent` extends the built-in `Event` interface and adds a `detail` property for passing arbitrary data with the event.

### Creating and Dispatching

```javascript
// Basic CustomEvent
const event = new CustomEvent('user:login', {
  detail: {
    userId: 42,
    username: 'alice',
    timestamp: Date.now(),
  },
  bubbles: true,      // default: false
  cancelable: true,   // default: false
  composed: false,    // default: false — whether it crosses Shadow DOM boundaries
});

// Dispatch from a DOM element
document.querySelector('#app').dispatchEvent(event);

// Or from document/window for app-wide events
document.dispatchEvent(event);
```

### Listening

```javascript
document.addEventListener('user:login', (e) => {
  console.log(e.type);           // 'user:login'
  console.log(e.detail.userId);  // 42
  console.log(e.target);         // #app element
});
```

### Cancelable Events

```javascript
const event = new CustomEvent('before:save', {
  bubbles: true,
  cancelable: true,
  detail: { data: formData },
});

const allowed = element.dispatchEvent(event);
// dispatchEvent returns false if any handler called preventDefault()

if (!allowed) {
  console.log('Save was cancelled by a listener');
}

// Listener cancelling the action
document.addEventListener('before:save', (e) => {
  if (!isValid(e.detail.data)) {
    e.preventDefault(); // signals cancellation to the dispatcher
  }
});
```

## Base Event vs. CustomEvent

```javascript
// Event is appropriate when no data needs to be passed
const simpleEvent = new Event('modal:close', { bubbles: true });

// CustomEvent when you need to attach data
const richEvent = new CustomEvent('item:selected', {
  bubbles: true,
  detail: { id: 7, label: 'Widget' },
});
```

## EventTarget as a Standalone Event Bus

In modern browsers, `EventTarget` can be instantiated directly—no DOM element required. This enables a pure publish-subscribe event bus decoupled from the DOM.

```javascript
class EventBus extends EventTarget {
  emit(type, detail) {
    this.dispatchEvent(new CustomEvent(type, { detail }));
  }

  on(type, handler, options) {
    this.addEventListener(type, handler, options);
    return () => this.removeEventListener(type, handler);
  }
}

const bus = new EventBus();

const unsubscribe = bus.on('data:loaded', (e) => {
  console.log(e.detail);
});

bus.emit('data:loaded', { items: [1, 2, 3] });

unsubscribe(); // clean up
```

### Typed Event Bus Pattern

```javascript
const appBus = new EventTarget();

// Emit
function emit(type, detail) {
  appBus.dispatchEvent(new CustomEvent(type, { detail }));
}

// Listen
function on(type, handler) {
  appBus.addEventListener(type, handler);
  return () => appBus.removeEventListener(type, handler);
}

// Usage across unrelated modules
// cart.js
const offCartUpdate = on('cart:updated', (e) => renderCart(e.detail));

// product.js
emit('cart:updated', { items: cartItems, total: 49.99 });
```

## Custom Events vs. Callbacks vs. Observables

```javascript
// Callback approach — tight coupling
class Modal {
  constructor(onClose) {
    this.onClose = onClose;
  }
  close() {
    this.onClose(); // caller must pass callback at construction
  }
}

// Custom event approach — loose coupling
class Modal extends EventTarget {
  close() {
    this.dispatchEvent(new CustomEvent('close', { bubbles: true }));
  }
}
// Any consumer can listen, none need to be known at construction time
modal.addEventListener('close', handler);
```

## Composing Events Through Shadow DOM

By default, events dispatched inside a Shadow DOM root do not cross the shadow boundary. The `composed: true` option allows them to propagate to the light DOM.

```javascript
class MyButton extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    const btn = document.createElement('button');
    btn.textContent = 'Click';

    btn.addEventListener('click', () => {
      // Without composed: true, this event stays inside the shadow root
      this.dispatchEvent(new CustomEvent('action', {
        detail: { value: this.getAttribute('value') },
        bubbles: true,
        composed: true, // crosses shadow boundary
      }));
    });

    this.shadowRoot.appendChild(btn);
  }
}

customElements.define('my-button', MyButton);

// Consumer in light DOM can listen
document.querySelector('my-button').addEventListener('action', (e) => {
  console.log(e.detail.value);
});
```

## Event Naming Conventions

```javascript
// Namespace with colon to avoid collisions with native events
// Format: domain:action or domain:subject:action
'user:login'
'user:logout'
'cart:item:added'
'cart:item:removed'
'modal:open'
'modal:close'
'data:loaded'
'data:error'

// Avoid single-word names that may collide with future browser events
// 'input', 'change', 'load', 'error' are already native
```

## Component Communication Patterns

### Parent to Child (Property / Method)

```javascript
// Direct property setting or method call — fine for parent-to-child
child.update({ items: newItems });
```

### Child to Parent (CustomEvent + bubbling)

```javascript
// child emits, parent listens — no coupling to parent reference
child.dispatchEvent(new CustomEvent('value:change', {
  detail: { value: newValue },
  bubbles: true,
}));

parent.addEventListener('value:change', (e) => {
  this.handleChildChange(e.detail.value);
});
```

### Sibling / Global (EventBus or document)

```javascript
// Use a shared bus or document for cross-tree communication
document.dispatchEvent(new CustomEvent('theme:changed', {
  detail: { theme: 'dark' },
}));
```

## Practical Example: Shopping Cart

```javascript
// cart-events.js — centralize event type constants
export const CART_EVENTS = {
  ADD:    'cart:add',
  REMOVE: 'cart:remove',
  UPDATE: 'cart:update',
  CLEAR:  'cart:clear',
};

// product-card.js
import { CART_EVENTS } from './cart-events.js';

class ProductCard extends HTMLElement {
  connectedCallback() {
    this.querySelector('.add-to-cart').addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent(CART_EVENTS.ADD, {
        bubbles: true,
        composed: true,
        detail: {
          productId: this.dataset.productId,
          quantity: 1,
          price: parseFloat(this.dataset.price),
        },
      }));
    });
  }
}

// cart.js — listens anywhere in the tree
document.addEventListener(CART_EVENTS.ADD, (e) => {
  const { productId, quantity, price } = e.detail;
  cartStore.addItem(productId, quantity, price);
  renderCart();
});
```

## Comparison Table

| Mechanism | Coupling | Data Passing | Multiple Listeners | DOM-Free |
|-----------|----------|--------------|--------------------|----------|
| Direct method call | Tight | Arguments | No | Yes |
| Callback | Medium | Arguments | One per registration | Yes |
| CustomEvent (DOM) | Loose | `event.detail` | Yes | No (needs element) |
| EventTarget bus | Loose | `event.detail` | Yes | Yes |
| Observable (RxJS) | Loose | Values | Yes (multicast) | Yes |

## Best Practices

### 1. Use Namespaced Event Names

```javascript
// Good — prevents collision with native events
'cart:item:added'

// Bad — may conflict with browser events
'change', 'update', 'error'
```

### 2. Define Event Types as Constants

```javascript
// Good — typos cause compile-time errors with TypeScript, runtime catching with const
export const EVT_CART_ADD = 'cart:add';

// Bad — repeated string literals invite typos
element.dispatchEvent(new CustomEvent('cart:aadd', { ... }));
```

### 3. Always Clean Up Listeners

```javascript
const handler = (e) => console.log(e.detail);
bus.addEventListener('data:loaded', handler);
// later...
bus.removeEventListener('data:loaded', handler);
// or use { once: true } for one-shot, or AbortController for bulk cleanup
```

### 4. Keep detail Serializable

```javascript
// Good — plain data, safe for potential cross-context use
detail: { id: 42, name: 'Alice' }

// Avoid — DOM nodes or class instances in detail limit reusability
detail: { element: domNode } // ties listener to DOM
```

## Interview Questions

### Q1: What is the difference between Event and CustomEvent?

**Answer:** `Event` is the base interface for all events. `CustomEvent` extends it by adding a `detail` property, which holds arbitrary data passed from the dispatcher to listeners. Use `Event` when you only need to signal that something happened. Use `CustomEvent` when you need to pass contextual data.

### Q2: Can you use CustomEvent without a DOM element?

**Answer:** Yes. `EventTarget` can be instantiated directly (`new EventTarget()`), and any class can extend it. This makes it possible to build a pure JavaScript event bus, with no DOM dependency, using the native event API—complete with `addEventListener`, `removeEventListener`, and `dispatchEvent`.

### Q3: What does the composed option do?

**Answer:** By default, events dispatched within a Shadow DOM root do not cross the shadow boundary and are not visible in the outer document. Setting `composed: true` on a CustomEvent allows it to propagate through shadow DOM boundaries into the light DOM. This is critical when building Web Components that need to communicate with the rest of the page.

### Q4: What does dispatchEvent return, and when is that return value useful?

**Answer:** `dispatchEvent` returns `false` if the event was cancelable and at least one listener called `event.preventDefault()`, and `true` otherwise. This is useful for implementing pre-flight checks—dispatch a cancelable `before:action` event and only proceed if the result is `true` (no listener vetoed the action).

### Q5: How would you implement a component that communicates with its parent without knowing the parent's implementation?

**Answer:** The component dispatches a CustomEvent with `bubbles: true` from its own element. The parent (or any ancestor) attaches a listener to that element or any ancestor, including `document`. The child component has no reference to the parent—the DOM tree and event bubbling handle the wiring. This is the same pattern used by native inputs dispatching `change` events.

## Common Pitfalls

### 1. Forgetting bubbles: true

```javascript
// Event stays on the target — parent listeners never fire
el.dispatchEvent(new CustomEvent('selected')); // bubbles defaults to false

// Fix
el.dispatchEvent(new CustomEvent('selected', { bubbles: true }));
```

### 2. detail is Not Cloned

```javascript
// Listeners receive a reference to the same detail object
const data = { count: 0 };
el.dispatchEvent(new CustomEvent('tick', { detail: data }));

// If a listener mutates detail.count, other listeners see the mutation
// Pass a copy if immutability matters
new CustomEvent('tick', { detail: { ...data } });
```

### 3. Colliding with Native Event Names

```javascript
// 'error', 'load', 'input', 'change' are already used by browsers
// Custom events with those names cause confusing behavior
el.dispatchEvent(new CustomEvent('error', { detail: appError }));
// window onerror and other error handling may trigger unexpectedly
```

### 4. Losing the Signal from an Aborted AbortController

```javascript
const ac = new AbortController();
el.addEventListener('app:event', handler, { signal: ac.signal });
ac.abort(); // removes ALL listeners registered with this signal
// If you abort prematurely, no events will be heard
```

## Resources

- [MDN: CustomEvent](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent)
- [MDN: EventTarget](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)
- [MDN: Creating and triggering events](https://developer.mozilla.org/en-US/docs/Web/Events/Creating_and_triggering_events)
- [MDN: Event.composed](https://developer.mozilla.org/en-US/docs/Web/API/Event/composed)
