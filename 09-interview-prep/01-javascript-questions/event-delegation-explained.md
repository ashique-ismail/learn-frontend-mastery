# Event Delegation

## What It Is

Event delegation is attaching a **single event listener to a parent element** to handle events for multiple children, relying on event bubbling.

Instead of:
```js
// BAD — N listeners for N items
document.querySelectorAll('.item').forEach(item => {
  item.addEventListener('click', handleClick);
});
```

Do this:
```js
// GOOD — 1 listener on parent
document.querySelector('.list').addEventListener('click', (e) => {
  const item = e.target.closest('.item');
  if (item) handleClick(item);
});
```

---

## How It Works: Event Bubbling

When you click on a `<button>` inside a `<li>` inside a `<ul>`:
1. **Capture phase** — event travels DOWN from window → document → html → body → ul → li → button
2. **Target phase** — fires on the button (`e.target`)
3. **Bubble phase** — event travels UP button → li → ul → body → html → document → window

Delegation works because the click *bubbles up* to the parent, where your listener sits.

---

## Key Properties

```js
document.querySelector('.list').addEventListener('click', function(e) {
  e.target          // element that was actually clicked (deepest)
  e.currentTarget   // element the listener is attached to (the list)
  e.relatedTarget   // for mouseover/mouseout: the element being left/entered

  // Stop the event from bubbling further
  e.stopPropagation();

  // Stop other listeners on the SAME element from running
  e.stopImmediatePropagation();

  // Prevent default browser action (link navigation, form submit)
  e.preventDefault();
});
```

---

## `closest()` for Nested Elements

The clicked element might be a child of your target (e.g., you click an icon inside a button):

```html
<ul class="menu">
  <li class="menu-item" data-id="1">
    <span class="icon">📄</span>
    <span class="label">Item 1</span>
  </li>
</ul>
```

```js
document.querySelector('.menu').addEventListener('click', (e) => {
  // Don't use e.target directly — it might be .icon or .label
  const item = e.target.closest('.menu-item');
  if (!item) return;  // clicked outside any item
  console.log(item.dataset.id);  // '1'
});
```

`closest()` walks up the DOM from the clicked element and returns the first ancestor matching the selector (including itself).

---

## Dynamic Elements — The Key Benefit

Delegation handles elements added **after** the listener is set up:

```js
const list = document.querySelector('#todo-list');

// This listener works for items added in the future
list.addEventListener('click', (e) => {
  if (e.target.matches('.delete-btn')) {
    e.target.closest('.todo-item').remove();
  }
});

// Add items later — no need to re-attach listeners
function addTodo(text) {
  list.insertAdjacentHTML('beforeend',
    `<li class="todo-item">${text}<button class="delete-btn">✕</button></li>`
  );
}
```

---

## Performance

| Approach | DOM nodes | Event listeners | Memory |
|----------|-----------|-----------------|--------|
| Direct binding | 1000 items | 1000 listeners | High |
| Delegation | 1000 items | 1 listener | Low |

Relevant for large lists (virtual-scroll tables, infinite feeds). For small static lists, the difference is negligible.

---

## Pitfalls

### `stopPropagation` Breaks Delegation

If any child calls `e.stopPropagation()`, the event never reaches the delegated listener:

```js
document.querySelector('.child').addEventListener('click', (e) => {
  e.stopPropagation(); // BREAKS parent delegation!
});
```

### SVG Events

SVG elements have complex hierarchies — `<use>`, shadow DOM, etc. `e.target` may be a deep SVG node. Use `closest()` defensively.

### `mouseenter` / `mouseleave` Don't Bubble

These non-bubbling events can't be delegated. Use `mouseover`/`mouseout` instead (which do bubble, but also fire for child crossings).

---

## Common Interview Questions

**Q: What's the difference between `e.target` and `e.currentTarget`?**
`e.target` is the element that originally fired the event (e.g. the button clicked). `e.currentTarget` is the element the listener is attached to (e.g. the parent list). With delegation, these are often different.

**Q: When should you NOT use event delegation?**
- Events that don't bubble (`focus`/`blur` — use `focusin`/`focusout` instead; `mouseenter`/`mouseleave`)
- When you need to stop propagation inside a nested component
- When performance of `closest()` lookup is measurably slower than direct binding for your use case (rare)

**Q: How does React handle events internally?**
React uses a single root-level delegated listener (on the root container in React 17+, previously on `document`). It implements its own synthetic event system, normalizes browser differences, and dispatches through a virtual bubbling tree.
