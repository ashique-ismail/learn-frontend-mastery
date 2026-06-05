# Sticky vs Fixed Positioning

## The Idea

**In plain English:** When you scroll a webpage, some elements stay locked in one spot on your screen forever (fixed), while others travel with the page at first but then lock in place once they reach a certain point — and only stay locked as long as their section of the page is visible (sticky).

**Real-world analogy:** Imagine a bulletin board on a school hallway wall. A teacher tapes a "Rules" sign directly onto your glasses so it always floats in the corner of your vision no matter where you walk — that is fixed positioning. But a tour guide holds up a paddle sign as you walk through each room; the sign travels with you at first, then gets pinned to the doorframe of that room while you are inside, and disappears once you leave the room — that is sticky positioning.

- The sign glued to your glasses = a `position: fixed` element (always in the same spot on the screen)
- The tour guide's paddle pinned to a doorframe = a `position: sticky` element (sticks only while its parent section is in view)
- Each room = the parent container that limits how long the sticky element stays stuck

---

## Fixed Positioning

An element with `position: fixed` is positioned relative to the **viewport** and removed from normal flow:

```css
.header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
}
```

- Stays in the same viewport position regardless of scrolling
- Does not scroll with the page
- Always visible (while viewport condition is met)
- Takes up no space in the document flow — other content flows under it

**Use cases:** Fixed navigation bars, floating action buttons, persistent chat widgets, cookie banners.

```css
/* Fixed FAB */
.fab {
  position: fixed;
  bottom: 24px;
  right: 24px;
  z-index: 50;
}
```

---

## Sticky Positioning

An element with `position: sticky` starts in the normal flow, then "sticks" at a specified offset when the user scrolls to it:

```css
.sticky-header {
  position: sticky;
  top: 0;       /* sticks 0px from the top of its scroll container */
  z-index: 10;
}
```

**Behavior:**
1. Initially renders in its normal flow position (takes up space)
2. As user scrolls, the element sticks at `top: 0` (or whatever offset you specify)
3. Stops sticking when its **parent container** scrolls out of view

**Use cases:** Section headers in long lists, table headers while scrolling, sidebar that sticks while article content is visible.

---

## The Critical Sticky Gotcha: Parent Container

Sticky position is constrained to its **parent element**. The element sticks *within* the parent's scroll area:

```html
<section class="section"> <!-- sticky stops when this scrolls away -->
  <h2 class="sticky-title">Section A</h2>
  <div class="content">...lots of content...</div>
</section>

<section class="section">
  <h2 class="sticky-title">Section B</h2>
  <div class="content">...</div>
</section>
```

```css
.sticky-title {
  position: sticky;
  top: 0;
}
```

This creates the "stacked section headers" pattern where each title sticks until the next section pushes it away. Each header is constrained to its own `<section>`.

---

## Common Sticky Bugs

### 1. `overflow: hidden` on an ancestor breaks sticky

```css
/* ❌ This breaks sticky */
.container {
  overflow: hidden; /* or overflow: auto, or overflow: scroll */
}
.child {
  position: sticky;
  top: 0;
  /* Won't stick — container intercepts the scroll */
}
```

The sticky element sticks within its scroll container, and `overflow` hidden/auto/scroll creates a new scroll container.

**Fix:** Remove `overflow` from the ancestor, or add `overflow: clip` (doesn't create a scroll container).

### 2. Parent too short

If the parent is only as tall as the sticky element, it never sticks (nowhere to scroll within the parent).

### 3. Missing `top`/`bottom`/`left`/`right` threshold

Sticky requires specifying the threshold: `position: sticky` alone doesn't do anything visible.

---

## Feature Comparison

| | `fixed` | `sticky` |
|---|---|---|
| Relative to | Viewport | Scroll container |
| Affects document flow | No (removed) | Yes (stays in flow) |
| Scrolls with content | No | Yes (until threshold) |
| Constraint | None | Parent container |
| Common use | Always-visible navigation | Section headers, sidebars |

---

## Common Interview Questions

**Q: Does sticky work inside a flex or grid container?**
Yes, but the parent must scroll. If using `sticky` inside a grid, ensure the grid row has enough height for the sticky element to actually scroll.

**Q: How do you detect if sticky is active (CSS)?**
Use the `:stuck` pseudo-class (CSS Scroll Snap, upcoming spec). Currently, you need a `IntersectionObserver` trick: observe a 1px element above the sticky target.

**Q: Why would you choose sticky over fixed for a navigation bar?**
Sticky preserves document flow — there's no gap where the element was. Fixed removes from flow — you need padding/margin on the content below to prevent it from going under the fixed element.
