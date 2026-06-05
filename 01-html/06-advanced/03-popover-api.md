# Popover API

## The Idea

**In plain English:** The Popover API is a built-in browser feature that lets you show and hide small floating boxes of content — like a tooltip, a menu, or a notification — without writing a lot of complex code. A "popover" is just a box that pops up on top of the rest of the page when triggered.

**Real-world analogy:** Think of a fast-food restaurant where you press a button on a self-order kiosk and a small card slides out showing the item's nutritional info. When you tap anywhere else, the card disappears on its own.

- The button on the kiosk = the HTML `<button>` with the `popovertarget` attribute
- The nutritional info card = the `<div popover>` element
- The card disappearing when you tap away = the browser's built-in "light dismiss" behavior for `popover="auto"`

---

## Overview

The Popover API provides a standardized way to create popovers, tooltips, menus, and other overlay elements without JavaScript for basic cases. It handles positioning, focus management, and dismissal automatically.

## Basic popover

```html
<button popovertarget="my-popover">Show Popover</button>

<div id="my-popover" popover>
  <h3>Popover Title</h3>
  <p>This is popover content</p>
  <button popovertarget="my-popover" popovertargetaction="hide">
    Close
  </button>
</div>
```

## Popover types

### auto (default)

Automatically closes when clicking outside or opening another popover.

```html
<div id="my-popover" popover="auto">
  <!-- Content -->
</div>

<!-- Or simply -->
<div id="my-popover" popover>
  <!-- Content -->
</div>
```

**Behavior:**
- Only one auto popover visible at a time
- Closes on outside click
- Closes when Escape pressed
- Closes when another auto popover opens

### manual

Requires explicit action to close.

```html
<div id="my-popover" popover="manual">
  <!-- Content -->
  <button popovertarget="my-popover" popovertargetaction="hide">
    Close
  </button>
</div>
```

**Behavior:**
- Multiple manual popovers can be open
- Doesn't auto-close on outside click
- Must be closed explicitly
- Useful for notifications, alerts

## Triggering popovers

### popovertarget attribute

```html
<!-- Toggle (default) -->
<button popovertarget="my-popover">Toggle</button>

<!-- Show -->
<button popovertarget="my-popover" popovertargetaction="show">
  Show
</button>

<!-- Hide -->
<button popovertarget="my-popover" popovertargetaction="hide">
  Hide
</button>

<div id="my-popover" popover>
  Popover content
</div>
```

### JavaScript API

```javascript
const popover = document.getElementById('my-popover');

// Show popover
popover.showPopover();

// Hide popover
popover.hidePopover();

// Toggle popover
popover.togglePopover();

// Check if showing
console.log(popover.matches(':popover-open'));
```

## Styling popovers

### Default styling

```css
[popover] {
  border: 1px solid #ccc;
  border-radius: 8px;
  padding: 16px;
  background: white;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
}
```

### Open state

```css
[popover]:popover-open {
  /* Styles when popover is visible */
  animation: slideIn 0.2s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

### Backdrop

```css
[popover]::backdrop {
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(2px);
}
```

## Positioning

Popovers appear in the top layer but need manual positioning:

```css
[popover] {
  /* Anchor to button (CSS Anchor Positioning) */
  position: fixed;
  inset: auto;
  
  /* Position below trigger */
  top: anchor(bottom);
  left: anchor(left);
  
  /* Or use JavaScript for positioning */
}
```

### JavaScript positioning

```javascript
function positionPopover(popover, trigger) {
  const triggerRect = trigger.getBoundingClientRect();
  
  popover.style.top = `${triggerRect.bottom + 8}px`;
  popover.style.left = `${triggerRect.left}px`;
}

const trigger = document.getElementById('trigger');
const popover = document.getElementById('popover');

trigger.addEventListener('click', () => {
  popover.showPopover();
  positionPopover(popover, trigger);
});
```

## Complete examples

### Tooltip

```html
<button 
  id="help-button"
  popovertarget="help-tooltip"
  aria-describedby="help-tooltip">
  Help ⓘ
</button>

<div id="help-tooltip" popover role="tooltip">
  <p>This feature helps you manage your tasks efficiently.</p>
</div>

<style>
  #help-tooltip {
    max-width: 250px;
    padding: 8px 12px;
    font-size: 14px;
    border-radius: 4px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  }
</style>
```

### Menu

```html
<button popovertarget="file-menu">File</button>

<div id="file-menu" popover role="menu">
  <button role="menuitem">New</button>
  <button role="menuitem">Open</button>
  <button role="menuitem">Save</button>
  <hr>
  <button role="menuitem">Exit</button>
</div>

<style>
  #file-menu {
    padding: 8px 0;
    min-width: 150px;
  }
  
  #file-menu button {
    display: block;
    width: 100%;
    padding: 8px 16px;
    border: none;
    background: none;
    text-align: left;
    cursor: pointer;
  }
  
  #file-menu button:hover {
    background: #f5f5f5;
  }
  
  #file-menu hr {
    margin: 4px 0;
    border: none;
    border-top: 1px solid #eee;
  }
</style>

<script>
  const menu = document.getElementById('file-menu');
  const items = menu.querySelectorAll('[role="menuitem"]');
  
  items.forEach(item => {
    item.addEventListener('click', () => {
      console.log('Clicked:', item.textContent);
      menu.hidePopover();
    });
  });
</script>
```

### Notification

```html
<button id="notify-btn">Show Notification</button>

<div id="notification" popover="manual" role="status">
  <h4>Success!</h4>
  <p>Your changes have been saved.</p>
  <button popovertarget="notification" popovertargetaction="hide">
    Dismiss
  </button>
</div>

<style>
  #notification {
    position: fixed;
    top: 20px;
    right: 20px;
    max-width: 300px;
    padding: 16px;
    background: #4caf50;
    color: white;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  }
  
  #notification h4 {
    margin: 0 0 8px 0;
  }
  
  #notification button {
    margin-top: 8px;
    padding: 4px 12px;
    background: white;
    color: #4caf50;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
</style>

<script>
  const notifyBtn = document.getElementById('notify-btn');
  const notification = document.getElementById('notification');
  
  notifyBtn.addEventListener('click', () => {
    notification.showPopover();
    
    // Auto-hide after 5 seconds
    setTimeout(() => {
      notification.hidePopover();
    }, 5000);
  });
</script>
```

### Action sheet

```html
<button popovertarget="action-sheet">Actions</button>

<div id="action-sheet" popover>
  <h3>Choose Action</h3>
  <button data-action="edit">Edit</button>
  <button data-action="share">Share</button>
  <button data-action="delete" class="danger">Delete</button>
  <button popovertarget="action-sheet" popovertargetaction="hide">
    Cancel
  </button>
</div>

<style>
  #action-sheet {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    margin: 0;
    border-radius: 16px 16px 0 0;
    padding: 20px;
  }
  
  #action-sheet h3 {
    margin: 0 0 16px 0;
  }
  
  #action-sheet button {
    display: block;
    width: 100%;
    padding: 12px;
    margin-bottom: 8px;
    border: 1px solid #ddd;
    border-radius: 8px;
    background: white;
    cursor: pointer;
  }
  
  #action-sheet button:hover {
    background: #f5f5f5;
  }
  
  #action-sheet button.danger {
    color: red;
    border-color: red;
  }
</style>

<script>
  const actionSheet = document.getElementById('action-sheet');
  const actions = actionSheet.querySelectorAll('[data-action]');
  
  actions.forEach(button => {
    button.addEventListener('click', () => {
      const action = button.dataset.action;
      console.log('Action:', action);
      actionSheet.hidePopover();
      
      // Handle action
      if (action === 'delete') {
        deleteItem();
      }
    });
  });
</script>
```

### Nested popovers

```html
<button popovertarget="main-menu">Menu</button>

<div id="main-menu" popover>
  <button>Home</button>
  <button popovertarget="settings-menu">Settings</button>
  <button>About</button>
</div>

<div id="settings-menu" popover>
  <button>Account</button>
  <button>Privacy</button>
  <button>Notifications</button>
  <button popovertarget="settings-menu" popovertargetaction="hide">
    Back
  </button>
</div>
```

## Events

### beforetoggle event

Fires before popover state changes.

```javascript
popover.addEventListener('beforetoggle', (event) => {
  console.log('Old state:', event.oldState); // "open" or "closed"
  console.log('New state:', event.newState); // "open" or "closed"
  
  // Prevent opening
  if (shouldPreventOpen) {
    event.preventDefault();
  }
});
```

### toggle event

Fires after popover state changes.

```javascript
popover.addEventListener('toggle', (event) => {
  if (event.newState === 'open') {
    console.log('Popover opened');
  } else {
    console.log('Popover closed');
  }
});
```

## Accessibility

### ARIA attributes

```html
<button 
  popovertarget="my-popover"
  aria-expanded="false"
  aria-controls="my-popover">
  Show Popover
</button>

<div 
  id="my-popover" 
  popover
  role="dialog"
  aria-labelledby="popover-title">
  
  <h3 id="popover-title">Popover Title</h3>
  <p>Content</p>
</div>

<script>
  const button = document.querySelector('[popovertarget="my-popover"]');
  const popover = document.getElementById('my-popover');
  
  popover.addEventListener('toggle', (e) => {
    button.setAttribute('aria-expanded', e.newState === 'open');
  });
</script>
```

### Focus management

```javascript
popover.addEventListener('toggle', (e) => {
  if (e.newState === 'open') {
    // Focus first focusable element
    const firstFocusable = popover.querySelector('button, [href], input, select, textarea');
    if (firstFocusable) {
      firstFocusable.focus();
    }
  }
});
```

## Combining with Anchor Positioning

CSS Anchor Positioning (upcoming):

```html
<button id="anchor-btn" popovertarget="anchored-popover">
  Show Popover
</button>

<div id="anchored-popover" popover>
  Anchored content
</div>

<style>
  #anchor-btn {
    anchor-name: --my-anchor;
  }
  
  #anchored-popover {
    position: fixed;
    position-anchor: --my-anchor;
    
    /* Position below anchor */
    top: anchor(bottom);
    left: anchor(left);
    
    /* Or with offset */
    top: calc(anchor(bottom) + 8px);
  }
</style>
```

## Progressive enhancement

Provide fallback for browsers without popover support:

```html
<button id="trigger" popovertarget="my-popover">
  Show
</button>

<div id="my-popover" popover hidden>
  Content
</div>

<script>
  if (!HTMLElement.prototype.hasOwnProperty('popover')) {
    // Polyfill or fallback implementation
    const popover = document.getElementById('my-popover');
    const trigger = document.getElementById('trigger');
    
    trigger.addEventListener('click', () => {
      popover.hidden = !popover.hidden;
    });
  }
</script>
```

## Best practices

1. Use `popover="auto"` for most overlays
2. Use `popover="manual"` for notifications
3. Provide accessible labels and roles
4. Handle focus management
5. Position popovers appropriately
6. Test keyboard navigation
7. Consider mobile viewports
8. Provide visual feedback
9. Use semantic HTML inside popovers
10. Test with screen readers

## Browser support

As of 2025:
- Chrome 114+
- Edge 114+
- Safari 17+
- Firefox: In development

Check support:

```javascript
if ('popover' in HTMLElement.prototype) {
  // Popover API supported
} else {
  // Use fallback
}
```

## Comparison with dialog

| Feature | Popover | Dialog |
|---------|---------|--------|
| Purpose | Tooltips, menus, hints | Modal dialogs |
| Backdrop | Optional | Yes (showModal) |
| Focus trap | No | Yes (showModal) |
| Light dismiss | Yes (auto) | Optional |
| Top layer | Yes | Yes |
| Multiple open | Yes (manual) | Yes |
| Use case | Non-blocking UI | Blocking interactions |

## Key takeaways

- Popover API provides native popover functionality
- Two types: auto (light dismiss) and manual (explicit close)
- Use `popovertarget` attribute to connect trigger and popover
- Automatic focus management and light dismiss
- Appears in top layer above other content
- Style with `:popover-open` pseudo-class
- Listen to `toggle` and `beforetoggle` events
- Combine with CSS Anchor Positioning for smart placement
- Check browser support and provide fallback
- Use for tooltips, menus, notifications, action sheets
