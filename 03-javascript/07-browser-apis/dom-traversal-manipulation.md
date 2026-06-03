# DOM Traversal and Manipulation

## Overview

The Document Object Model (DOM) is a tree-structured representation of an HTML document that JavaScript can read and modify. Every element, attribute, and text node is represented as an object. Understanding how to efficiently traverse and manipulate the DOM is foundational to browser-side JavaScript—and a frequent topic in senior-level interviews, where interviewers expect nuanced answers about performance, live vs. static collections, and reflow/repaint implications.

## The DOM Tree

The DOM is a hierarchy of `Node` objects. Understanding node types matters when traversing:

```javascript
// Node type constants
Node.ELEMENT_NODE       // 1 - <div>, <p>, etc.
Node.TEXT_NODE          // 3 - text content
Node.COMMENT_NODE       // 8 - <!-- comment -->
Node.DOCUMENT_NODE      // 9 - the document itself
Node.DOCUMENT_TYPE_NODE // 10 - <!DOCTYPE>

// Check node type
if (node.nodeType === Node.ELEMENT_NODE) {
  console.log(node.tagName); // DIV, SPAN, etc.
}
```

## Selecting Elements

### Single-Element Selectors

```javascript
// Returns the first matching element or null
const el = document.querySelector('.container > .item');

// getElementById is faster than querySelector for ID lookups
const el2 = document.getElementById('main');

// Less common but available
const el3 = document.querySelector('[data-id="42"]');
```

### Multi-Element Selectors

```javascript
// Returns a static NodeList (snapshot)
const items = document.querySelectorAll('.item');

// Returns a live HTMLCollection (updates when DOM changes)
const divs = document.getElementsByTagName('div');
const byClass = document.getElementsByClassName('active');

// Demonstrating live vs. static
const liveList = document.getElementsByClassName('item');
const staticList = document.querySelectorAll('.item');

// Add a new element
document.body.appendChild(Object.assign(document.createElement('div'), { className: 'item' }));

console.log(liveList.length);   // Increased by 1
console.log(staticList.length); // Unchanged
```

### Scoped Queries

```javascript
// querySelector works on any element, not just document
const container = document.getElementById('sidebar');
const links = container.querySelectorAll('a[href]');
// Only finds anchors inside #sidebar
```

## Traversal

### Parent, Children, Siblings

```javascript
const el = document.querySelector('.target');

// Parent
el.parentElement;       // nearest parent Element (null if none)
el.parentNode;          // nearest parent Node (may be document or DocumentFragment)

// Children
el.children;            // live HTMLCollection of child Elements only
el.childNodes;          // live NodeList of all child Nodes (incl. text, comments)
el.firstElementChild;   // first child Element
el.lastElementChild;    // last child Element
el.firstChild;          // first child Node (may be a TextNode)
el.lastChild;           // last child Node

// Siblings
el.nextElementSibling;  // next sibling Element
el.previousElementSibling;
el.nextSibling;         // next sibling Node (may be text)
el.previousSibling;

// Descendants
el.closest('.ancestor'); // walks UP the tree, returns first matching ancestor (or self)
```

### Walking the DOM Manually

```javascript
// Depth-first traversal using TreeWalker (efficient, avoids recursion)
const walker = document.createTreeWalker(
  document.body,
  NodeFilter.SHOW_ELEMENT,
  {
    acceptNode(node) {
      return node.classList.contains('skip')
        ? NodeFilter.FILTER_SKIP
        : NodeFilter.FILTER_ACCEPT;
    }
  }
);

while (walker.nextNode()) {
  console.log(walker.currentNode.tagName);
}
```

### contains and compareDocumentPosition

```javascript
const parent = document.getElementById('wrapper');
const child = document.querySelector('.item');

parent.contains(child); // true if child is a descendant (or self)

// Bitmask: 0=same, 1=disconnected, 2=before, 4=after, 8=contains, 16=contained
const pos = parent.compareDocumentPosition(child);
console.log(pos & Node.DOCUMENT_POSITION_CONTAINED_BY); // non-zero if child is inside
```

## Reading and Writing Content

### Text and HTML

```javascript
const el = document.querySelector('p');

// Read/write text content (safe from XSS)
el.textContent = '<b>Not bold</b>'; // Renders as literal string

// Read/write HTML (parse and render HTML)
el.innerHTML = '<b>Bold</b>'; // Parsed as HTML — XSS risk if user-controlled!

// outerHTML includes the element itself
console.log(el.outerHTML); // <p><b>Bold</b></p>

// insertAdjacentText / insertAdjacentHTML
el.insertAdjacentHTML('beforeend', '<span>appended</span>');
// Positions: 'beforebegin' | 'afterbegin' | 'beforeend' | 'afterend'
```

### Attributes

```javascript
const link = document.querySelector('a');

// Standard properties (fast, reflected)
link.href;
link.id;
link.className;
link.classList; // DOMTokenList

// Attribute API (works for non-standard, data-*, ARIA, etc.)
link.getAttribute('href');
link.setAttribute('href', '/new-path');
link.removeAttribute('href');
link.hasAttribute('disabled');

// Dataset (data-* attributes)
const el = document.querySelector('[data-user-id]');
el.dataset.userId; // camelCase getter
el.dataset.userId = '42'; // setter
```

### Class Manipulation

```javascript
el.classList.add('active');
el.classList.remove('active');
el.classList.toggle('active');
el.classList.toggle('active', condition); // force on/off
el.classList.replace('old', 'new');
el.classList.contains('active'); // boolean
```

## Creating and Inserting Elements

### createElement and DocumentFragment

```javascript
// Create
const div = document.createElement('div');
div.className = 'card';
div.textContent = 'Hello';

// Append (modern, accepts multiple args and strings)
parent.append(div, 'text node');
parent.prepend(div);

// Legacy (accepts only a single Node)
parent.appendChild(div);
parent.insertBefore(div, referenceNode);

// insertAdjacentElement
parent.insertAdjacentElement('afterbegin', div);

// DocumentFragment — batches DOM insertions, avoids multiple reflows
const fragment = document.createDocumentFragment();
for (let i = 0; i < 1000; i++) {
  const li = document.createElement('li');
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);
}
document.querySelector('ul').appendChild(fragment); // Single reflow
```

### Cloning

```javascript
const original = document.querySelector('.template');
const shallowClone = original.cloneNode(false); // element only, no children
const deepClone = original.cloneNode(true);     // element + all descendants

// Note: cloneNode does NOT copy event listeners added via addEventListener
```

## Removing and Replacing Elements

```javascript
const el = document.querySelector('.old');

// Modern
el.remove();

// Legacy (need parent reference)
el.parentNode.removeChild(el);

// Replace
const newEl = document.createElement('section');
el.replaceWith(newEl);
// Or legacy: el.parentNode.replaceChild(newEl, el);
```

## Reading Layout and Geometry

```javascript
const el = document.querySelector('.box');

// DOMRect — relative to viewport, fractional pixels
const rect = el.getBoundingClientRect();
console.log(rect.top, rect.left, rect.width, rect.height);

// Scroll position
window.scrollX; // alias: pageXOffset
window.scrollY;

// Element scroll
el.scrollTop;
el.scrollLeft;

// Offset dimensions (layout box, integer pixels)
el.offsetWidth;  // includes padding + border (not margin)
el.offsetHeight;
el.offsetTop;    // relative to offsetParent
el.offsetParent; // nearest positioned ancestor

// Client dimensions (visible size without scrollbars)
el.clientWidth;  // includes padding, excludes border and scrollbar
el.clientHeight;

// Scroll dimensions (full scrollable content)
el.scrollWidth;
el.scrollHeight;

// Checking visibility
const isVisible = rect.top < window.innerHeight && rect.bottom > 0;
```

### Force Synchronous Layout (Layout Thrashing)

```javascript
// BAD: read-write-read-write causes multiple layout recalculations
for (const el of elements) {
  const height = el.offsetHeight;   // forces layout
  el.style.height = height + 10 + 'px'; // invalidates layout
}

// GOOD: batch reads first, then writes
const heights = elements.map(el => el.offsetHeight); // one layout
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + 'px'; // batch writes
});
```

## Styles

```javascript
const el = document.querySelector('.box');

// Inline styles (highest specificity)
el.style.backgroundColor = '#fff';
el.style.cssText = 'color: red; font-size: 16px;';

// Computed styles (includes all applied CSS, read-only)
const computed = getComputedStyle(el);
computed.backgroundColor; // "rgb(255, 255, 255)"
computed.getPropertyValue('font-size'); // "16px"

// Custom properties (CSS variables)
const root = document.documentElement;
root.style.setProperty('--primary', '#007bff');
getComputedStyle(root).getPropertyValue('--primary'); // " #007bff"
```

## Comparison Table

| Method | Returns | Live? | Scoped? |
|--------|---------|-------|---------|
| `getElementById` | Element or null | - | document only |
| `querySelector` | Element or null | No | Any element |
| `querySelectorAll` | Static NodeList | No | Any element |
| `getElementsByTagName` | HTMLCollection | Yes | Any element |
| `getElementsByClassName` | HTMLCollection | Yes | Any element |

## Best Practices

### 1. Cache DOM References

```javascript
// Bad: queries the DOM on each loop iteration
for (let i = 0; i < 100; i++) {
  document.querySelector('#counter').textContent = i;
}

// Good: cache the reference
const counter = document.querySelector('#counter');
for (let i = 0; i < 100; i++) {
  counter.textContent = i;
}
```

### 2. Avoid innerHTML with User Data

```javascript
// XSS vulnerability
el.innerHTML = `<p>${userInput}</p>`;

// Safe alternatives
el.textContent = userInput; // plain text
const p = document.createElement('p');
p.textContent = userInput;
el.appendChild(p);
```

### 3. Use DocumentFragment for Bulk Inserts

```javascript
const frag = document.createDocumentFragment();
data.forEach(item => {
  const li = document.createElement('li');
  li.textContent = item.name;
  frag.appendChild(li);
});
list.appendChild(frag); // single DOM mutation
```

### 4. Prefer classList over className

```javascript
// className overwrites all existing classes
el.className += ' active'; // Can introduce leading space bugs

// classList API is precise
el.classList.add('active');
el.classList.toggle('active', isActive);
```

## Interview Questions

### Q1: What is the difference between children and childNodes?

**Answer:** `children` returns a live `HTMLCollection` containing only Element nodes. `childNodes` returns a live `NodeList` that includes all node types—elements, text nodes, and comment nodes. Most of the time `children` is what you want; `childNodes` is useful when you need to process text nodes or comments.

### Q2: What's the difference between innerHTML and textContent? When would you use each?

**Answer:** `textContent` reads or writes the raw text content of a node and all its descendants. Setting it treats the value as plain text, which prevents XSS. `innerHTML` reads the serialized HTML of a node's subtree and, when set, parses the string as HTML. Use `textContent` for safe, text-only updates. Use `innerHTML` only for trusted HTML strings (e.g., sanitized templates)—never with raw user input.

### Q3: What is layout thrashing and how do you avoid it?

**Answer:** Layout thrashing occurs when JavaScript alternates between reading layout properties (`offsetHeight`, `getBoundingClientRect`, etc.) and writing style properties, forcing the browser to recalculate layout repeatedly within a single frame. Avoid it by batching all reads first, then all writes. For complex scenarios, `requestAnimationFrame` can defer writes to the next paint cycle. Libraries like FastDOM formally separate read and write phases.

### Q4: Why might you use createDocumentFragment?

**Answer:** Appending individual elements to the live DOM triggers a reflow per insertion. A `DocumentFragment` is an in-memory, off-document node. You append all new children to the fragment first, then append the fragment to the DOM in a single operation—causing only one reflow and repaint.

### Q5: What does closest() do and how is it different from parentElement?

**Answer:** `closest(selector)` walks up the ancestor chain starting from the element itself and returns the first element that matches the CSS selector, or `null` if none is found. `parentElement` only returns the direct parent. `closest` is especially useful in event delegation to find the nearest ancestor matching a selector from an event target.

## Common Pitfalls

### 1. Modifying a Live Collection During Iteration

```javascript
const divs = document.getElementsByTagName('div'); // live
for (let i = 0; i < divs.length; i++) {
  document.body.removeChild(divs[i]); // modifies divs.length!
}
// Items get skipped as the collection shrinks

// Fix: convert to array first
Array.from(divs).forEach(div => document.body.removeChild(div));
```

### 2. innerHTML Resets Event Listeners

```javascript
const btn = document.querySelector('.btn');
btn.addEventListener('click', handler);

container.innerHTML = container.innerHTML; // Reparsing clears all listeners on children
```

### 3. offsetHeight Triggering Layout

```javascript
// This forces layout synchronously, potentially causing jank
requestAnimationFrame(() => {
  el.classList.add('expanded');
  const h = el.offsetHeight; // forced sync layout mid-frame
});
```

### 4. Detached DOM Nodes Causing Memory Leaks

```javascript
// element removed from DOM but kept in a closure
let detached = document.querySelector('.old');
document.body.removeChild(detached);
// detached is still in memory! Set to null when done.
detached = null;
```

## Resources

- [MDN: DOM Introduction](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction)
- [MDN: Node](https://developer.mozilla.org/en-US/docs/Web/API/Node)
- [MDN: Element](https://developer.mozilla.org/en-US/docs/Web/API/Element)
- [MDN: DocumentFragment](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment)
- [What forces layout/reflow](https://gist.github.com/paulirish/5d52fb081b3570c81e3a) — Paul Irish's exhaustive list
- [FastDOM](https://github.com/wilsonpage/fastdom) — library for batching layout reads/writes
