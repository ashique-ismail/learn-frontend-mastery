# :focus-visible: Accessible Focus Styles and Keyboard Navigation

## The Idea

**In plain English:** `:focus-visible` is a CSS rule that controls when a glowing highlight ring (called a focus indicator) appears around a button or link on a webpage — it only shows the ring when you are navigating with a keyboard, not when you click with a mouse.

**Real-world analogy:** Imagine a tour guide leading two groups through a museum. For the walking tour group (keyboard users), the guide shines a flashlight on each exhibit as they stop there, so everyone knows exactly where they are. For the self-guided group (mouse users), the flashlight stays off because those visitors can already see what they clicked on.
- The flashlight beam = the focus ring (the visible outline around an element)
- The walking tour group = keyboard users who need a visible indicator to know where they are
- The self-guided group = mouse users who do not need the extra highlight

---

## Learning Objectives
- Understand the difference between `:focus` and `:focus-visible`
- Implement accessible keyboard navigation patterns
- Design effective focus indicators
- Meet WCAG 2.4.7 (Focus Visible) requirements
- Handle focus management in complex components
- Avoid common focus styling mistakes

---

## `:focus` vs `:focus-visible`

### The Problem with `:focus`

**`:focus`** applies when an element receives focus, **regardless of input method** (mouse, keyboard, touch).

```css
button:focus {
  outline: 2px solid blue;
}
```

**Problem:**
- Mouse users see outline after clicking (annoying)
- Keyboard users need outline (accessibility)
- Developers often disable outline: `outline: none` (bad for a11y)

**Visual:**
```
Mouse click:
┌──────────────┐
│ [Button]     │  ← Shows outline (not needed)
└──────────────┘

Keyboard Tab:
┌──────────────┐
│ [Button]     │  ← Shows outline (needed!)
└──────────────┘
```

---

### The Solution: `:focus-visible`

**`:focus-visible`** applies only when the browser **determines** focus should be visible (typically keyboard navigation).

```css
button:focus {
  outline: none; /* Remove default for all */
}

button:focus-visible {
  outline: 2px solid blue; /* Only show when needed */
}
```

**Behavior:**
- Mouse click: No outline (`:focus-visible` doesn't match)
- Keyboard Tab: Shows outline (`:focus-visible` matches)
- Touch: Usually no outline (`:focus-visible` doesn't match)

---

### Browser Heuristics

**When does `:focus-visible` apply?**

Browser decides based on:
1. **Input method:** Keyboard = visible, mouse/touch = not visible
2. **Element type:** Text inputs always show focus (even on click)
3. **User action:** Tab key = visible, click = not visible

**Examples:**

| Element | Action | `:focus` | `:focus-visible` |
|---------|--------|----------|------------------|
| `<button>` | Mouse click | ✓ | ✗ |
| `<button>` | Tab key | ✓ | ✓ |
| `<input type="text">` | Mouse click | ✓ | ✓ |
| `<input type="text">` | Tab key | ✓ | ✓ |
| `<a>` | Mouse click | ✓ | ✗ |
| `<a>` | Tab key | ✓ | ✓ |
| `<div tabindex="0">` | Mouse click | ✓ | ✗ |
| `<div tabindex="0">` | Tab key | ✓ | ✓ |

---

## Basic Implementation

### Recommended Pattern

```css
/* Remove default outline for all focus */
*:focus {
  outline: none;
}

/* Add accessible outline only when needed */
*:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
  border-radius: 2px;
}
```

**Why this works:**
- Removes outline for mouse users (cleaner UI)
- Keeps outline for keyboard users (accessibility)
- Consistent across all interactive elements

---

### Per-Element Styling

```css
/* Buttons */
button:focus {
  outline: none;
}

button:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}

/* Links */
a:focus {
  outline: none;
}

a:focus-visible {
  outline: 2px dashed #0066cc;
  outline-offset: 4px;
}

/* Inputs */
input:focus {
  outline: none;
  border-color: #0066cc;
}

input:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
```

---

## WCAG Requirements

### WCAG 2.4.7: Focus Visible (AA)

**Requirement:** Any keyboard-operable interface must have a **visible focus indicator**.

**Criteria:**
- Focus indicator must be visible
- Must have sufficient contrast (at least 3:1 against adjacent colors)
- Must be at least 2px thick (or 1px if high contrast)

---

### WCAG 2.4.11: Focus Appearance (AAA, WCAG 2.2)

**Enhanced criteria:**
- Minimum area of focus indicator: **2 CSS pixels** around perimeter, or **4 CSS pixels** of total area
- Contrast ratio: **At least 3:1** against adjacent colors
- No less visible than default focus indicator

---

### Accessible Focus Examples

#### Good: High Contrast Outline

```css
button:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}
```

**Passes WCAG 2.4.7 (AA):**
- ✓ Visible
- ✓ 3px thick (exceeds minimum)
- ✓ High contrast (#0066cc on white)

---

#### Good: Box Shadow

```css
button:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px #0066cc;
}
```

**Passes WCAG 2.4.7 (AA):**
- ✓ Visible
- ✓ 3px thick
- ✓ High contrast

---

#### Good: Inset Focus Ring

```css
button {
  border: 2px solid transparent;
  transition: border-color 0.2s;
}

button:focus-visible {
  border-color: #0066cc;
  outline: none;
}
```

**Passes WCAG 2.4.7 (AA):**
- ✓ Visible
- ✓ 2px thick
- ✓ High contrast

---

#### Bad: Insufficient Contrast

```css
/* BAD - low contrast */
button:focus-visible {
  outline: 2px solid #cccccc; /* Light gray on white - contrast < 3:1 */
}
```

**Fails WCAG 2.4.7:** Contrast too low.

**Fix:**
```css
/* GOOD - high contrast */
button:focus-visible {
  outline: 2px solid #666666; /* Contrast > 3:1 */
}
```

---

#### Bad: Too Thin

```css
/* BAD - too thin */
button:focus-visible {
  outline: 1px solid #0066cc;
}
```

**Passes WCAG 2.4.7 but barely:** 1px is minimum, but 2px+ is better.

**Fix:**
```css
/* GOOD - thicker */
button:focus-visible {
  outline: 2px solid #0066cc;
}
```

---

## Keyboard Navigation Patterns

### Tab Order

**Default tab order:** Document order (top to bottom, left to right).

```html
<button>First</button>
<button>Second</button>
<button>Third</button>
```

**Tab sequence:** First → Second → Third → (loops to First)

---

### Custom Tab Order with `tabindex`

**`tabindex` values:**

| Value | Behavior |
|-------|----------|
| `0` | Natural tab order (element is focusable) |
| `-1` | Not in tab order (focusable via JS only) |
| `1+` | Custom order (anti-pattern, avoid!) |

---

#### Adding to Tab Order

```html
<!-- Make non-interactive elements focusable -->
<div tabindex="0">Focusable div</div>

<!-- Custom widget -->
<div role="button" tabindex="0">Custom button</div>
```

```css
[tabindex="0"]:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
```

---

#### Removing from Tab Order

```html
<!-- Temporarily disabled -->
<button tabindex="-1">Not focusable</button>

<!-- Managed focus (e.g., in modal) -->
<div role="dialog" tabindex="-1">
  <h2>Modal Title</h2>
  <button>Close</button>
</div>
```

---

#### Avoid Positive `tabindex`

```html
<!-- BAD - breaks natural order -->
<button tabindex="3">Third</button>
<button tabindex="1">First</button>
<button tabindex="2">Second</button>
```

**Problem:** Positive `tabindex` creates confusing tab order. Avoid!

**Fix:** Use document order or JavaScript to manage focus.

---

### Skip Links

**Problem:** Keyboard users must tab through entire navigation to reach main content.

**Solution:** Skip link jumps directly to main content.

```html
<a href="#main-content" class="skip-link">Skip to main content</a>

<nav>
  <!-- Many links... -->
</nav>

<main id="main-content">
  <!-- Content -->
</main>
```

```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #0066cc;
  color: white;
  padding: 0.5rem 1rem;
  text-decoration: none;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

**Behavior:**
- Hidden by default (off-screen)
- Appears when focused (Tab key)
- Clicking jumps to `#main-content`

---

### Focus Trapping (Modals)

**Problem:** Keyboard focus can leave modal and interact with background content.

**Solution:** Trap focus inside modal.

```html
<div class="modal" role="dialog" aria-modal="true">
  <h2>Modal Title</h2>
  <button class="close">Close</button>
  <button>OK</button>
</div>
```

```javascript
// Focus trap example
const modal = document.querySelector('.modal');
const focusableElements = modal.querySelectorAll(
  'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
);
const firstElement = focusableElements[0];
const lastElement = focusableElements[focusableElements.length - 1];

modal.addEventListener('keydown', (e) => {
  if (e.key === 'Tab') {
    if (e.shiftKey) {
      // Shift+Tab - going backwards
      if (document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      }
    } else {
      // Tab - going forwards
      if (document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }
  }
});

// Focus first element when modal opens
firstElement.focus();
```

---

### Focus Restoration

**Problem:** After closing modal, focus returns to start of page.

**Solution:** Save and restore focus.

```javascript
let previouslyFocused;

function openModal() {
  // Save currently focused element
  previouslyFocused = document.activeElement;
  
  // Open modal and focus first element
  modal.classList.add('open');
  modal.querySelector('button').focus();
}

function closeModal() {
  // Close modal
  modal.classList.remove('open');
  
  // Restore focus
  previouslyFocused.focus();
}
```

---

## Styling Patterns

### Pattern 1: Outline with Offset

```css
button:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 4px;
}
```

**Visual:**
```
┌─────────────┐
│   Button    │
└─────────────┘
  ╔═══════╗      ← 4px offset
  ║       ║      ← 2px outline
```

**Best for:** Buttons, links with space around them.

---

### Pattern 2: Box Shadow Ring

```css
button:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px rgba(0, 102, 204, 0.5);
}
```

**Visual:**
```
┌─────────────┐
│   Button    │  ← Soft glow
└─────────────┘
```

**Best for:** Cards, buttons, modern UIs.

---

### Pattern 3: Border Highlight

```css
input {
  border: 2px solid #ccc;
  transition: border-color 0.2s;
}

input:focus {
  border-color: #0066cc;
  outline: none;
}

input:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
```

**Visual:**
```
┌─────────────┐
│ Input       │  ← Border changes color
└─────────────┘
  ╔═══════╗      ← Additional outline
```

**Best for:** Form inputs, text fields.

---

### Pattern 4: Inset Focus

```css
button {
  position: relative;
}

button::before {
  content: '';
  position: absolute;
  inset: 4px;
  border: 2px solid transparent;
  border-radius: 4px;
  transition: border-color 0.2s;
  pointer-events: none;
}

button:focus-visible::before {
  border-color: #0066cc;
}

button:focus {
  outline: none;
}
```

**Visual:**
```
┌─────────────┐
│ ┌─────────┐ │
│ │ Button  │ │  ← Inset border
│ └─────────┘ │
└─────────────┘
```

**Best for:** Buttons with backgrounds, filled buttons.

---

### Pattern 5: Underline (Links)

```css
a {
  text-decoration: none;
  color: #0066cc;
  position: relative;
}

a::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  bottom: -2px;
  height: 2px;
  background: transparent;
  transition: background 0.2s;
}

a:focus {
  outline: none;
}

a:focus-visible::after {
  background: #0066cc;
}

a:hover::after {
  background: #0066cc;
}
```

**Visual:**
```
Link text
─────────  ← Underline appears on focus/hover
```

**Best for:** Text links, navigation links.

---

## Dark Mode Focus Styles

```css
/* Light mode */
:root {
  --focus-color: #0066cc;
}

*:focus-visible {
  outline: 2px solid var(--focus-color);
  outline-offset: 2px;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --focus-color: #4d9fff;
  }
}
```

**Key:** Light blue in dark mode for visibility.

---

## Complex Components

### Tabs

```html
<div class="tabs" role="tablist">
  <button role="tab" aria-selected="true" tabindex="0">Tab 1</button>
  <button role="tab" aria-selected="false" tabindex="-1">Tab 2</button>
  <button role="tab" aria-selected="false" tabindex="-1">Tab 3</button>
</div>

<div role="tabpanel">Content 1</div>
```

```css
[role="tab"]:focus {
  outline: none;
}

[role="tab"]:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: -2px;
}

[role="tab"][aria-selected="true"] {
  border-bottom: 3px solid #0066cc;
  font-weight: 600;
}
```

**Keyboard navigation:**
- Tab: Focus tablist
- Arrow keys: Navigate between tabs
- Enter/Space: Activate tab

---

### Dropdown Menu

```html
<button aria-haspopup="true" aria-expanded="false">Menu</button>
<ul role="menu" hidden>
  <li role="menuitem"><a href="#">Item 1</a></li>
  <li role="menuitem"><a href="#">Item 2</a></li>
  <li role="menuitem"><a href="#">Item 3</a></li>
</ul>
```

```css
button:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}

[role="menu"] {
  list-style: none;
  padding: 0;
}

[role="menuitem"]:focus-within {
  background: #f0f0f0;
}

[role="menuitem"] a:focus {
  outline: none;
}

[role="menuitem"] a:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: -2px;
}
```

---

### Custom Checkbox

```html
<label class="checkbox">
  <input type="checkbox">
  <span class="checkbox-custom"></span>
  Accept terms
</label>
```

```css
.checkbox {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  cursor: pointer;
}

.checkbox input {
  position: absolute;
  opacity: 0;
}

.checkbox-custom {
  width: 20px;
  height: 20px;
  border: 2px solid #666;
  border-radius: 4px;
  transition: all 0.2s;
}

.checkbox input:checked + .checkbox-custom {
  background: #0066cc;
  border-color: #0066cc;
}

.checkbox input:checked + .checkbox-custom::after {
  content: '✓';
  color: white;
  display: flex;
  justify-content: center;
  align-items: center;
}

.checkbox input:focus {
  outline: none;
}

.checkbox input:focus-visible + .checkbox-custom {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
```

---

## Common Mistakes

### ❌ Mistake 1: Removing outline without replacement

```css
/* BAD - no focus indicator */
button:focus {
  outline: none;
}
```

**Problem:** Keyboard users can't see focus.

**Fix:**
```css
/* GOOD - outline removed but replaced */
button:focus {
  outline: none;
}

button:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
```

---

### ❌ Mistake 2: Only styling `:focus`, not `:focus-visible`

```css
/* BAD - shows outline on mouse click */
button:focus {
  outline: 2px solid #0066cc;
}
```

**Problem:** Outline appears on click (annoying for mouse users).

**Fix:**
```css
/* GOOD - only shows when needed */
button:focus {
  outline: none;
}

button:focus-visible {
  outline: 2px solid #0066cc;
}
```

---

### ❌ Mistake 3: Low contrast focus indicator

```css
/* BAD - light blue on white, contrast < 3:1 */
button:focus-visible {
  outline: 2px solid #cce6ff;
}
```

**Problem:** Fails WCAG 2.4.7 (contrast too low).

**Fix:**
```css
/* GOOD - dark blue on white, contrast > 3:1 */
button:focus-visible {
  outline: 2px solid #0066cc;
}
```

---

### ❌ Mistake 4: Using positive `tabindex`

```html
<!-- BAD - confusing tab order -->
<button tabindex="3">Third</button>
<button tabindex="1">First</button>
<button tabindex="2">Second</button>
```

**Fix:**
```html
<!-- GOOD - natural order -->
<button>First</button>
<button>Second</button>
<button>Third</button>
```

---

### ❌ Mistake 5: Not testing with keyboard

**Problem:** Many developers only test with mouse.

**Fix:** Test with keyboard:
1. Use Tab key to navigate
2. Use Enter/Space to activate
3. Use Arrow keys for menus/tabs
4. Verify focus is always visible

---

## Testing Focus Styles

### Manual Testing

**Keyboard:**
1. Tab through all interactive elements
2. Verify focus indicator is visible
3. Check focus order makes sense
4. Test Shift+Tab (backward navigation)

**Screen reader:**
1. Turn on VoiceOver (Mac) / NVDA (Windows)
2. Navigate with keyboard
3. Verify announcements are clear

---

### DevTools

**Chrome:**
1. Open DevTools → Elements
2. Click element
3. Toggle `:focus` state in Styles panel
4. Verify focus styles

**Firefox:**
1. Open DevTools → Inspector
2. Click element
3. Toggle `:focus` state in Rules panel

---

### Automated Testing

**axe DevTools:**
```javascript
// Check for focus indicators
axe.run({
  rules: {
    'focus-order-semantics': { enabled: true },
    'tabindex': { enabled: true }
  }
});
```

**Lighthouse:**
1. Run Lighthouse audit
2. Check "Accessibility" score
3. Review focus-related issues

---

## Browser Support (2026)

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| `:focus-visible` | 86+ | 85+ | 15.4+ | 86+ |
| `:focus-within` | 60+ | 52+ | 10.1+ | 79+ |

**Fallback for old browsers:**
```css
/* Fallback: show outline for all focus */
button:focus {
  outline: 2px solid #0066cc;
}

/* Modern: only show when needed */
button:focus:not(:focus-visible) {
  outline: none;
}

button:focus-visible {
  outline: 2px solid #0066cc;
}
```

---

## Polyfill

**For older browsers (pre-2021):**
```html
<script src="https://unpkg.com/focus-visible"></script>
```

```css
/* Polyfill adds .focus-visible class */
button:focus:not(.focus-visible) {
  outline: none;
}

button.focus-visible {
  outline: 2px solid #0066cc;
}
```

**Note:** Not needed in 2026 - all modern browsers support `:focus-visible`.

---

## Key Takeaways

✅ **Use `:focus-visible` instead of `:focus`** - shows focus only when needed

✅ **Never remove outline without replacement** - keyboard users need focus indicator

✅ **Ensure 3:1 contrast minimum** - WCAG 2.4.7 requirement

✅ **Test with keyboard** - Tab through entire interface

✅ **Use `tabindex="0"` for custom widgets** - make them keyboard-accessible

✅ **Avoid positive `tabindex` values** - breaks natural tab order

✅ **Implement focus trapping in modals** - prevent focus from escaping

✅ **Restore focus after closing modals** - return to previous element

---

## Resources

- [MDN: :focus-visible](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible)
- [MDN: :focus](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus)
- [WCAG 2.4.7: Focus Visible](https://www.w3.org/WAI/WCAG21/Understanding/focus-visible.html)
- [WebAIM: Keyboard Accessibility](https://webaim.org/articles/keyboard/)
- [Sara Soueidan: A guide to designing accessible, WCAG-compliant focus indicators](https://www.sarasoueidan.com/blog/focus-indicators/)
- [Focus Visible Polyfill](https://github.com/WICG/focus-visible)
