# CSS Position Property

## The Idea

**In plain English:** The CSS `position` property lets you control exactly where an element sits on a webpage — whether it stays in the normal flow of content, gets nudged from its usual spot, or gets pinned to a fixed place on the screen regardless of scrolling. "Normal flow" just means the default top-to-bottom, left-to-right order that elements follow before you change anything.

**Real-world analogy:** Think of arranging sticky notes on a whiteboard. By default, you place each note in sequence, one after another (normal flow). But sometimes you want to pin a note in the top-right corner so it stays visible even when you scroll the board, or you want to shift a note slightly to overlap the one next to it, or you want a note that "floats" above all the others.
- The whiteboard = the webpage (the document)
- Each sticky note = an HTML element
- Pinning a note to a fixed corner of the frame = `position: fixed` (stays put while you scroll)
- Shifting a note slightly from where it would normally sit = `position: relative`
- Placing a note precisely relative to a specific area of the board = `position: absolute`

---

## Overview

The `position` property determines how an element is positioned in the document. It's the foundation for creating overlays, sticky headers, fixed navigation, and complex layered layouts.

## Position Values

### `static` (Default)

```css
.element {
  position: static; /* Default behavior */
}
```

- **Normal document flow**
- `top`, `right`, `bottom`, `left` have **no effect**
- `z-index` has **no effect**
- Elements stack in source order

**Use case**: This is the default; you rarely set it explicitly unless overriding.

### `relative`

```css
.element {
  position: relative;
  top: 20px;    /* Moves down 20px from original position */
  left: 10px;   /* Moves right 10px from original position */
}
```

**Key behaviors**:
- Element **remains in normal flow** (space is reserved)
- Offset properties (`top`, `right`, `bottom`, `left`) move it **relative to itself**
- **Creates positioning context** for absolutely positioned children
- `z-index` works

**Visual effect**: Element is displaced but its original space is preserved (like it's still there).

```
Before:        After (top: 20px, left: 10px):
┌───┬───┐      ┌───┬───┐
│ A │ B │      │ A │ B │    ┌─────┐
├───┴───┤      ├───┴───┤    │  C  │ ← Visually offset
│   C   │      │ [gap] │    └─────┘
└───────┘      └───────┘
```

**Common use case**: Creating a positioning context for children:

```css
.parent {
  position: relative; /* Establishes positioning context */
}

.child {
  position: absolute; /* Positioned relative to .parent */
  top: 0;
  right: 0;
}
```

### `absolute`

```css
.element {
  position: absolute;
  top: 20px;
  right: 30px;
}
```

**Key behaviors**:
- **Removed from normal flow** (no space reserved)
- Positioned relative to the **nearest positioned ancestor** (anything with `position` other than `static`)
- If no positioned ancestor, positioned relative to the **initial containing block** (usually `<html>`)
- `z-index` works
- Width becomes **shrink-to-fit** (unless specified)

**Offset properties** are relative to the edges of the containing block:
- `top`: distance from top edge
- `right`: distance from right edge
- `bottom`: distance from bottom edge
- `left`: distance from left edge

#### Centering with Absolute Position

```css
.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%); /* Shift back by half its own size */
}

/* Alternative (when dimensions are known) */
.centered-alt {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  margin: auto; /* Centers if width/height are set */
  width: 200px;
  height: 100px;
}
```

#### Stretching to Fill Container

```css
.overlay {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  /* Fills entire positioned ancestor */
}

/* Shorthand (newer browsers) */
.overlay {
  position: absolute;
  inset: 0; /* top, right, bottom, left all 0 */
}
```

### `fixed`

```css
.element {
  position: fixed;
  top: 20px;
  right: 20px;
}
```

**Key behaviors**:
- **Removed from normal flow**
- Positioned relative to the **viewport** (browser window)
- **Stays in place when scrolling** (unless inside a `transform`/`filter`/`perspective` ancestor)
- `z-index` works
- Width becomes **shrink-to-fit**

**Common use cases**:
- Fixed headers/navbars
- Floating action buttons
- Chat widgets
- Cookie banners

#### Fixed Header Example

```css
.header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 60px;
  background: white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  z-index: 100;
}

body {
  padding-top: 60px; /* Prevent content from hiding under fixed header */
}
```

#### Gotcha: Transform Creates New Containing Block

```css
.parent {
  transform: translateZ(0); /* or any transform */
}

.child {
  position: fixed; /* Now fixed relative to .parent, NOT viewport! */
}
```

**Other properties that create new containing blocks for `fixed`**:
- `transform` (any value except `none`)
- `filter` (any value except `none`)
- `perspective` (any value except `none`)
- `will-change: transform | filter | perspective`
- `contain: paint`

### `sticky`

```css
.element {
  position: sticky;
  top: 20px; /* Sticks when 20px from top of scroll container */
}
```

**Key behaviors**:
- **Hybrid of `relative` and `fixed`**
- Remains in normal flow **until scroll threshold is reached**
- Then "sticks" to specified offset
- Only sticks **within its containing block** (parent element)
- Requires at least one offset value (`top`, `bottom`, `left`, or `right`)
- `z-index` works

**How it works**:
1. Element scrolls normally with the page (`relative` behavior)
2. When it reaches the specified offset (e.g., `top: 20px`), it "sticks" (`fixed` behavior)
3. When its container scrolls out of view, it scrolls away with the container

#### Sticky Header Example

```css
.section-header {
  position: sticky;
  top: 0;
  background: white;
  z-index: 10;
}
```

#### Multiple Sticky Elements

```css
.sticky-header { position: sticky; top: 0; }
.sticky-subheader { position: sticky; top: 60px; } /* Stacks below header */
```

#### Common Gotchas

1. **Parent must allow overflow**: If any ancestor has `overflow: hidden`, sticky won't work
   ```css
   .parent {
     overflow: hidden; /* ❌ Breaks sticky */
   }
   ```

2. **Height matters**: Sticky element must have room to scroll within its container
   ```css
   /* ❌ Won't work if .container height < .sticky height */
   .container { height: 100px; }
   .sticky { position: sticky; height: 150px; }
   ```

3. **Needs a threshold**: Must specify `top`, `bottom`, `left`, or `right`
   ```css
   .sticky {
     position: sticky; /* ❌ Won't stick without offset */
   }
   ```

4. **Table elements**: Sticky works on `<thead>`, `<th>`, `<tr>`, but with quirks

## Positioning Context (Containing Block)

The "containing block" is the reference point for positioning:

| Position | Containing Block |
|----------|------------------|
| `static` / `relative` | Content area of parent element |
| `absolute` | Padding box of nearest **positioned ancestor** (not `static`) |
| `fixed` | Viewport (or nearest `transform`/`filter`/etc. ancestor) |
| `sticky` | Nearest scrolling ancestor |

### Example: Nested Positioning

```css
.grandparent {
  position: relative; /* Positioning context */
}

.parent {
  /* No position set (static) */
}

.child {
  position: absolute; /* Positioned relative to .grandparent, not .parent */
  top: 0;
}
```

## Offset Properties

### `top`, `right`, `bottom`, `left`

```css
.element {
  position: absolute;
  top: 10px;    /* Distance from top of containing block */
  right: 20px;  /* Distance from right edge */
  bottom: 30px; /* Distance from bottom */
  left: 40px;   /* Distance from left */
}
```

**Values**:
- Length: `10px`, `2em`, `5rem`
- Percentage: `50%` (relative to containing block dimensions)
- `auto` (default)

### `inset` Shorthand (Modern)

```css
/* Instead of: */
.element {
  top: 10px;
  right: 20px;
  bottom: 10px;
  left: 20px;
}

/* Use: */
.element {
  inset: 10px 20px; /* vertical horizontal */
  /* or */
  inset: 10px; /* all sides */
  /* or */
  inset: 10px 20px 30px 40px; /* top right bottom left */
}
```

**Logical properties**:
```css
inset-block-start: 10px;  /* top in LTR */
inset-block-end: 10px;    /* bottom in LTR */
inset-inline-start: 20px; /* left in LTR */
inset-inline-end: 20px;   /* right in LTR */
```

## Stacking and `z-index`

`z-index` only works on **positioned elements** (`relative`, `absolute`, `fixed`, `sticky`):

```css
.element {
  position: relative;
  z-index: 10; /* Stacks above elements with lower z-index */
}
```

**Values**:
- Integer: `1`, `10`, `100`, `-1`
- `auto` (default)

### Stacking Contexts

Positioned elements with `z-index` create a **new stacking context**. Children are stacked relative to their parent, not the document:

```css
.parent {
  position: relative;
  z-index: 1;
}

.child {
  position: relative;
  z-index: 9999; /* Still behind elements with parent z-index > 1 */
}
```

**Other properties that create stacking contexts**:
- `opacity` < 1
- `transform` (any value except `none`)
- `filter` (any value except `none`)
- `mix-blend-mode` (any value except `normal`)
- `isolation: isolate`

## Practical Patterns

### Modal Overlay

```css
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 1000;
}

.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: white;
  padding: 20px;
  z-index: 1001;
}
```

### Badge on Button

```css
.button {
  position: relative;
}

.badge {
  position: absolute;
  top: -8px;
  right: -8px;
  background: red;
  color: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
}
```

### Sticky Sidebar

```css
.sidebar {
  position: sticky;
  top: 80px; /* Stick below fixed header */
  align-self: flex-start; /* Important for flex containers */
}
```

### Floating Action Button (FAB)

```css
.fab {
  position: fixed;
  bottom: 20px;
  right: 20px;
  width: 60px;
  height: 60px;
  border-radius: 50%;
  box-shadow: 0 4px 8px rgba(0,0,0,0.3);
  z-index: 100;
}
```

## Accessibility Considerations

1. **Reading order**: Positioned elements can disrupt logical reading order for screen readers
   ```css
   /* Visual order ≠ DOM order */
   ```

2. **Focus management**: Ensure focus order makes sense, especially with modals

3. **`position: fixed`**: Can obscure content; ensure proper `z-index` layering

4. **Keyboard navigation**: Fixed/absolute elements can trap focus if not handled properly

## Performance Considerations

1. **`position: fixed`**: Triggers its own layer; generally performant
2. **`position: absolute`**: Reflows can be expensive in complex layouts
3. **`position: sticky`**: Can cause jank on scroll if not optimized
   ```css
   .sticky {
     position: sticky;
     will-change: transform; /* Hint to browser for optimization */
   }
   ```

4. **Avoid positioning hundreds of elements**: Creates many layers

## Interview Questions to Master

1. What's the difference between `relative`, `absolute`, and `fixed` positioning?
2. What is a "positioned element"?
3. How does `absolute` positioning determine its containing block?
4. When does `position: fixed` NOT position relative to the viewport?
5. What are the requirements for `position: sticky` to work?
6. What creates a new stacking context?
7. How do you center an absolutely positioned element?
8. What's the difference between `top: 0` and `bottom: 0` when both are applied?

## Resources

- [MDN Position](https://developer.mozilla.org/en-US/docs/Web/CSS/position)
- [CSS Tricks: position](https://css-tricks.com/almanac/properties/p/position/)
- [Understanding Stacking Context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Positioning/Understanding_z_index/The_stacking_context)
