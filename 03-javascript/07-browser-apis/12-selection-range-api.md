# Selection API and Range API

## The Idea

**In plain English:** The Selection API and Range API let your code know exactly what text a user has highlighted on a webpage, and also let the code highlight or manipulate any chunk of text on its own. A "range" is just a precise description of a start point and an end point inside the page's content.

**Real-world analogy:** Imagine you are reading a physical book and you use a highlighter pen to mark a passage. You place the cap of the pen at one word (the start) and drag to another word (the end), and everything in between gets colored.

- The highlighter's starting position = the range's `startContainer` and `startOffset`
- The highlighted span of text = the `Range` object
- The act of looking at what is already highlighted = `window.getSelection()`

---

## Overview

The Selection API (`window.getSelection()`) represents what the user has selected on the page. The Range API (`document.createRange()`) represents an arbitrary contiguous portion of the document tree — a start and end position. Together they power rich-text editors, search highlighting, caret positioning, and clipboard operations.

---

## Selection API

### Getting the Selection

```javascript
const selection = window.getSelection();

// Core properties
selection.type;           // 'None' | 'Caret' | 'Range'
selection.isCollapsed;    // true = cursor/caret, no text selected
selection.rangeCount;     // number of ranges (almost always 0 or 1)
selection.anchorNode;     // node where selection started
selection.anchorOffset;   // offset within anchorNode
selection.focusNode;      // node where selection ended
selection.focusOffset;    // offset within focusNode
selection.toString();     // selected text as string

// Get the first (usually only) range
const range = selection.getRangeAt(0);
```

### Programmatic Selection

```javascript
// Select all text in an element
function selectElement(el) {
  const range = document.createRange();
  range.selectNodeContents(el);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
}

// Select specific text
function selectText(el, start, end) {
  const range = document.createRange();
  range.setStart(el.firstChild, start);
  range.setEnd(el.firstChild, end);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
}

// Clear selection
window.getSelection().removeAllRanges();
// or:
window.getSelection().empty();
```

### Reacting to Selection Changes

```javascript
document.addEventListener('selectionchange', () => {
  const sel = window.getSelection();
  if (sel.isCollapsed) return; // cursor only, no selection

  const text = sel.toString();
  console.log('Selected:', text);

  // Position a floating toolbar above the selection
  const range = sel.getRangeAt(0);
  const rect  = range.getBoundingClientRect();
  showToolbar({ x: rect.left + rect.width / 2, y: rect.top + window.scrollY - 40 });
});
```

---

## Range API

A `Range` has two boundary points: start and end. Each is a (node, offset) pair.

- For **text nodes**: offset is character index
- For **element nodes**: offset is child index

```javascript
const range = document.createRange();

// setStart / setEnd (node, offset)
range.setStart(textNode, 5);   // 5 characters into textNode
range.setEnd(textNode, 15);    // 15 characters into textNode

// Convenience methods
range.selectNode(el);          // select the element itself (including tags)
range.selectNodeContents(el);  // select all children of el
range.collapse(true);          // collapse to start (true) or end (false)

// Properties
range.startContainer;  // start node
range.startOffset;     // start offset
range.endContainer;    // end node
range.endOffset;       // end offset
range.collapsed;       // true if start === end
range.commonAncestorContainer; // deepest element containing both endpoints

// Bounding rect (for positioning UI over selected text)
range.getBoundingClientRect();   // DOMRect of the selection
range.getClientRects();          // DOMRectList — multiple rects for multi-line

// Content operations
range.cloneContents();   // returns DocumentFragment (copy)
range.extractContents(); // removes content, returns DocumentFragment
range.deleteContents();  // removes content
range.insertNode(node);  // insert node at range start
range.surroundContents(node); // wrap range contents with node
```

---

## Caret Positioning

A collapsed selection (where start === end) is the text cursor / caret.

```javascript
// Move caret to end of a contenteditable element
function moveCursorToEnd(el) {
  const range = document.createRange();
  range.selectNodeContents(el);
  range.collapse(false); // collapse to end
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
  el.focus();
}

// Move caret to specific position in a text node
function setCursorPosition(textNode, offset) {
  const range = document.createRange();
  range.setStart(textNode, offset);
  range.collapse(true);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
}

// Get cursor offset in a contenteditable element
function getCursorOffset(el) {
  const sel = window.getSelection();
  if (!sel.rangeCount) return 0;
  const range = sel.getRangeAt(0).cloneRange();
  range.selectNodeContents(el);
  range.setEnd(sel.anchorNode, sel.anchorOffset);
  return range.toString().length;
}
```

---

## Search Result Highlighting

```javascript
// Highlight all occurrences of a search term
function highlightText(container, term) {
  // Clear previous highlights
  container.querySelectorAll('mark').forEach(m => {
    m.replaceWith(...m.childNodes);
  });
  container.normalize(); // merge adjacent text nodes

  if (!term) return;

  const walker = document.createTreeWalker(
    container,
    NodeFilter.SHOW_TEXT,
    { acceptNode: (node) =>
        node.parentElement.closest('script, style, mark')
          ? NodeFilter.FILTER_REJECT
          : NodeFilter.FILTER_ACCEPT
    }
  );

  const textNodes = [];
  while (walker.nextNode()) textNodes.push(walker.currentNode);

  const regex = new RegExp(escapeRegex(term), 'gi');

  for (const node of textNodes) {
    const text = node.textContent;
    let match;
    const fragments = [];
    let lastIndex = 0;

    while ((match = regex.exec(text)) !== null) {
      if (match.index > lastIndex) {
        fragments.push(document.createTextNode(text.slice(lastIndex, match.index)));
      }
      const mark = document.createElement('mark');
      mark.textContent = match[0];
      fragments.push(mark);
      lastIndex = regex.lastIndex;
    }

    if (fragments.length > 0) {
      if (lastIndex < text.length) {
        fragments.push(document.createTextNode(text.slice(lastIndex)));
      }
      node.replaceWith(...fragments);
    }
  }
}

function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

### CSS Custom Highlight API (Modern Alternative)

The CSS Custom Highlight API is more performant — it highlights without modifying the DOM:

```javascript
// CSS: ::highlight(search-results) { background: yellow; color: black; }

const ranges = [];
const walker = document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT);

while (walker.nextNode()) {
  const node  = walker.currentNode;
  const text  = node.textContent;
  const regex = /search term/gi;
  let match;

  while ((match = regex.exec(text)) !== null) {
    const range = new StaticRange({
      startContainer: node,
      startOffset:    match.index,
      endContainer:   node,
      endOffset:      match.index + match[0].length,
    });
    ranges.push(range);
  }
}

// Apply highlight
CSS.highlights.set('search-results', new Highlight(...ranges));

// Clear
CSS.highlights.delete('search-results');
```

---

## Rich-Text Editor Patterns

```javascript
class MinimalEditor {
  constructor(el) {
    this.el = el;
    el.contentEditable = 'true';
  }

  // Apply a format to the current selection
  format(command) {
    // Save selection
    const sel = window.getSelection();
    if (!sel.rangeCount || sel.isCollapsed) return;
    const range = sel.getRangeAt(0);

    if (command === 'bold') {
      const strong = document.createElement('strong');
      range.surroundContents(strong);
    } else if (command === 'highlight') {
      const mark = document.createElement('mark');
      range.surroundContents(mark);
    }

    // Restore selection (surroundContents invalidates the range)
    sel.removeAllRanges();
    sel.addRange(range);
  }

  // Insert text at cursor
  insertAt(text) {
    const sel = window.getSelection();
    if (!sel.rangeCount) return;
    const range = sel.getRangeAt(0);
    range.deleteContents();
    range.insertNode(document.createTextNode(text));
    // Move cursor after inserted text
    range.collapse(false);
    sel.removeAllRanges();
    sel.addRange(range);
  }

  // Get full plain text
  getText() {
    return this.el.textContent;
  }

  // Get HTML
  getHTML() {
    return this.el.innerHTML;
  }
}
```

---

## TreeWalker — Efficient DOM Traversal

`TreeWalker` is the performant way to traverse text nodes for operations like search:

```javascript
// Walk all text nodes
const walker = document.createTreeWalker(
  document.body,
  NodeFilter.SHOW_TEXT,    // only text nodes
  {
    acceptNode: (node) => {
      // Skip scripts, styles, hidden elements
      const el = node.parentElement;
      if (el.closest('script, style, [hidden]')) return NodeFilter.FILTER_REJECT;
      if (!node.textContent.trim()) return NodeFilter.FILTER_SKIP;
      return NodeFilter.FILTER_ACCEPT;
    }
  }
);

const textNodes = [];
while (walker.nextNode()) {
  textNodes.push(walker.currentNode);
}
```

NodeFilter values:

- `FILTER_ACCEPT` — include this node
- `FILTER_REJECT` — exclude this node AND all descendants
- `FILTER_SKIP` — exclude this node but traverse descendants

---

## Interview Questions

**Q: How would you position a floating toolbar above selected text?**
A: Listen to `selectionchange`. When the selection is non-collapsed, call `window.getSelection().getRangeAt(0).getBoundingClientRect()` to get the selection's pixel coordinates. Position the toolbar using those coordinates. For multi-line selections, `getClientRects()` gives individual line rects — use the first rect to position the toolbar above the selection start.

**Q: How does `range.surroundContents(node)` work and when does it throw?**
A: It wraps the range's content inside `node`. It throws `HierarchyRequestError` if the range partially selects a non-text node — for example, if the selection starts inside a `<strong>` but ends outside it. In rich-text editors, you usually need to use `extractContents()` + `insertNode()` instead for robust wrapping.

**Q: What is the CSS Custom Highlight API and why is it better than DOM-based highlighting?**
A: It highlights text ranges using CSS pseudo-elements (`::highlight(name)`) without modifying the DOM. DOM-based highlighting inserts wrapper elements (e.g. `<mark>`) which disrupts the document structure, breaks existing event listeners, and triggers layout. The CSS Highlight API is a style-only overlay — zero DOM mutation, no reflow.

**Q: What's the difference between `selectNode` and `selectNodeContents`?**
A: `selectNode(el)` selects the element including its opening/closing tags — the range spans from before the element to after it. `selectNodeContents(el)` selects only the element's children — the range spans from the start to the end of the element's content. For contenteditable editing, you almost always want `selectNodeContents`.
