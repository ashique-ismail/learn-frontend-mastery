# Focus management

## Overview

Focus management ensures keyboard users can navigate and interact with all functionality. Proper focus order, visible indicators, and focus trapping are essential for accessibility.

## Focusable elements

### Naturally focusable

These elements are focusable by default:

```html
<!-- Links -->
<a href="/page">Link</a>

<!-- Buttons -->
<button>Button</button>

<!-- Form controls -->
<input type="text">
<select><option>Option</option></select>
<textarea></textarea>

<!-- Media with controls -->
<video controls src="video.mp4"></video>
<audio controls src="audio.mp3"></audio>

<!-- Interactive elements -->
<details><summary>Summary</summary></details>
```

### Making elements focusable

Use `tabindex` to control focus behavior:

```html
<!-- tabindex="0": Adds to natural tab order -->
<div tabindex="0" role="button" onclick="handleClick()">
  Custom button
</div>

<!-- tabindex="-1": Programmatically focusable, not in tab order -->
<div tabindex="-1" id="error-message">
  Error message
</div>

<!-- tabindex="1+" (AVOID): Override natural order -->
<div tabindex="1">Don't do this</div>
```

## Focus order

### Natural focus order

Follow DOM order for intuitive navigation:

```html
<!-- Good: logical DOM order -->
<form>
  <label for="name">Name</label>
  <input type="text" id="name">
  
  <label for="email">Email</label>
  <input type="email" id="email">
  
  <button type="submit">Submit</button>
</form>
```

### Don't use positive tabindex

```html
<!-- Bad: breaks natural order -->
<input type="text" tabindex="3">
<input type="text" tabindex="1">
<input type="text" tabindex="2">

<!-- Good: natural DOM order -->
<input type="text">
<input type="text">
<input type="text">
```

### Visual vs DOM order

Ensure visual order matches DOM order:

```html
<!-- Bad: visual order differs from DOM -->
<div style="display: flex; flex-direction: column-reverse;">
  <button>Button 1 (focuses last)</button>
  <button>Button 2 (focuses first)</button>
</div>

<!-- Good: DOM order matches visual -->
<div>
  <button>Button 1 (focuses first)</button>
  <button>Button 2 (focuses last)</button>
</div>
```

## Focus indicators

### Default focus styles

```html
<button>Button with default focus</button>
```

### Custom focus styles

```css
/* Basic custom focus */
button:focus {
  outline: 2px solid blue;
  outline-offset: 2px;
}

/* Focus-visible (keyboard only) */
button:focus-visible {
  outline: 2px solid blue;
  outline-offset: 2px;
}

/* Remove outline for mouse clicks */
button:focus:not(:focus-visible) {
  outline: none;
}

/* High contrast focus */
@media (prefers-contrast: high) {
  button:focus-visible {
    outline: 3px solid currentColor;
    outline-offset: 3px;
  }
}
```

### Never remove focus without replacement

```css
/* Bad: removes all focus indicators */
*:focus {
  outline: none;
}

/* Good: replace with custom indicator */
button:focus-visible {
  outline: 2px solid blue;
  outline-offset: 2px;
}
```

## Focus management patterns

### Skip links

```html
<body>
  <a href="#main-content" class="skip-link">
    Skip to main content
  </a>
  
  <header>
    <nav><!-- Many navigation links --></nav>
  </header>
  
  <main id="main-content" tabindex="-1">
    <h1>Main Content</h1>
  </main>
</body>
```

```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

### Focus after page load

```javascript
// Focus heading after navigation
document.querySelector('h1').focus();

// Make heading focusable
const heading = document.querySelector('h1');
heading.tabIndex = -1;
heading.focus();
```

### Focus after dynamic content

```javascript
// After loading content
async function loadContent() {
  const response = await fetch('/api/content');
  const content = await response.text();
  
  const container = document.getElementById('content');
  container.innerHTML = content;
  
  // Focus first heading
  const heading = container.querySelector('h2');
  heading.tabIndex = -1;
  heading.focus();
}
```

## Modal dialogs

### Focus trap in modal

```html
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Dialog Title</h2>
  <p>Dialog content</p>
  
  <button id="cancel-btn">Cancel</button>
  <button id="confirm-btn">Confirm</button>
</div>
```

```javascript
class ModalDialog {
  constructor(dialogElement) {
    this.dialog = dialogElement;
    this.focusableElements = this.dialog.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    this.firstFocusable = this.focusableElements[0];
    this.lastFocusable = this.focusableElements[this.focusableElements.length - 1];
  }
  
  open() {
    // Store last focused element
    this.lastFocused = document.activeElement;
    
    // Show dialog
    this.dialog.style.display = 'block';
    
    // Focus first element
    this.firstFocusable.focus();
    
    // Add trap listener
    this.dialog.addEventListener('keydown', this.trapFocus.bind(this));
  }
  
  close() {
    // Hide dialog
    this.dialog.style.display = 'none';
    
    // Remove trap listener
    this.dialog.removeEventListener('keydown', this.trapFocus.bind(this));
    
    // Restore focus
    this.lastFocused.focus();
  }
  
  trapFocus(e) {
    if (e.key !== 'Tab') return;
    
    if (e.shiftKey) {
      // Shift + Tab
      if (document.activeElement === this.firstFocusable) {
        e.preventDefault();
        this.lastFocusable.focus();
      }
    } else {
      // Tab
      if (document.activeElement === this.lastFocusable) {
        e.preventDefault();
        this.firstFocusable.focus();
      }
    }
  }
}
```

### Escape key to close

```javascript
dialog.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') {
    closeDialog();
  }
});
```

## Dropdown menus

### Focus management in dropdown

```html
<button 
  id="menu-button"
  aria-haspopup="true" 
  aria-expanded="false"
  aria-controls="menu">
  Menu
</button>

<ul id="menu" role="menu" hidden>
  <li role="none">
    <button role="menuitem">Item 1</button>
  </li>
  <li role="none">
    <button role="menuitem">Item 2</button>
  </li>
  <li role="none">
    <button role="menuitem">Item 3</button>
  </li>
</ul>
```

```javascript
class DropdownMenu {
  constructor(button, menu) {
    this.button = button;
    this.menu = menu;
    this.items = menu.querySelectorAll('[role="menuitem"]');
    this.currentIndex = 0;
    
    this.button.addEventListener('click', () => this.toggle());
    this.button.addEventListener('keydown', (e) => this.handleButtonKey(e));
    this.menu.addEventListener('keydown', (e) => this.handleMenuKey(e));
    
    this.items.forEach((item, index) => {
      item.addEventListener('click', () => this.selectItem(index));
    });
  }
  
  open() {
    this.menu.hidden = false;
    this.button.setAttribute('aria-expanded', 'true');
    this.items[0].focus();
  }
  
  close() {
    this.menu.hidden = true;
    this.button.setAttribute('aria-expanded', 'false');
    this.button.focus();
  }
  
  toggle() {
    if (this.menu.hidden) {
      this.open();
    } else {
      this.close();
    }
  }
  
  handleButtonKey(e) {
    switch (e.key) {
      case 'ArrowDown':
      case 'Enter':
      case ' ':
        e.preventDefault();
        this.open();
        break;
    }
  }
  
  handleMenuKey(e) {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        this.currentIndex = (this.currentIndex + 1) % this.items.length;
        this.items[this.currentIndex].focus();
        break;
        
      case 'ArrowUp':
        e.preventDefault();
        this.currentIndex = (this.currentIndex - 1 + this.items.length) % this.items.length;
        this.items[this.currentIndex].focus();
        break;
        
      case 'Home':
        e.preventDefault();
        this.currentIndex = 0;
        this.items[0].focus();
        break;
        
      case 'End':
        e.preventDefault();
        this.currentIndex = this.items.length - 1;
        this.items[this.currentIndex].focus();
        break;
        
      case 'Escape':
        e.preventDefault();
        this.close();
        break;
    }
  }
  
  selectItem(index) {
    console.log('Selected:', this.items[index].textContent);
    this.close();
  }
}
```

## Tabs

### Tab panel focus management

```html
<div class="tabs">
  <div role="tablist" aria-label="Content tabs">
    <button role="tab" aria-selected="true" aria-controls="panel1" id="tab1">
      Tab 1
    </button>
    <button role="tab" aria-selected="false" aria-controls="panel2" id="tab2" tabindex="-1">
      Tab 2
    </button>
    <button role="tab" aria-selected="false" aria-controls="panel3" id="tab3" tabindex="-1">
      Tab 3
    </button>
  </div>
  
  <div role="tabpanel" id="panel1" aria-labelledby="tab1" tabindex="0">
    <h2>Panel 1</h2>
    <p>Content 1</p>
  </div>
  <div role="tabpanel" id="panel2" aria-labelledby="tab2" tabindex="0" hidden>
    <h2>Panel 2</h2>
    <p>Content 2</p>
  </div>
  <div role="tabpanel" id="panel3" aria-labelledby="tab3" tabindex="0" hidden>
    <h2>Panel 3</h2>
    <p>Content 3</p>
  </div>
</div>
```

```javascript
class TabsComponent {
  constructor(tablist) {
    this.tablist = tablist;
    this.tabs = tablist.querySelectorAll('[role="tab"]');
    this.currentIndex = 0;
    
    this.tabs.forEach((tab, index) => {
      tab.addEventListener('click', () => this.selectTab(index));
      tab.addEventListener('keydown', (e) => this.handleKeydown(e, index));
    });
  }
  
  selectTab(index) {
    // Deselect all tabs
    this.tabs.forEach((tab, i) => {
      tab.setAttribute('aria-selected', 'false');
      tab.tabIndex = -1;
      
      const panel = document.getElementById(tab.getAttribute('aria-controls'));
      panel.hidden = true;
    });
    
    // Select new tab
    const selectedTab = this.tabs[index];
    selectedTab.setAttribute('aria-selected', 'true');
    selectedTab.tabIndex = 0;
    selectedTab.focus();
    
    const panel = document.getElementById(selectedTab.getAttribute('aria-controls'));
    panel.hidden = false;
    
    this.currentIndex = index;
  }
  
  handleKeydown(e, index) {
    let newIndex = index;
    
    switch (e.key) {
      case 'ArrowRight':
        e.preventDefault();
        newIndex = (index + 1) % this.tabs.length;
        break;
        
      case 'ArrowLeft':
        e.preventDefault();
        newIndex = (index - 1 + this.tabs.length) % this.tabs.length;
        break;
        
      case 'Home':
        e.preventDefault();
        newIndex = 0;
        break;
        
      case 'End':
        e.preventDefault();
        newIndex = this.tabs.length - 1;
        break;
        
      default:
        return;
    }
    
    this.selectTab(newIndex);
  }
}
```

## Roving tabindex

For widget components like toolbars:

```html
<div role="toolbar" aria-label="Text formatting">
  <button tabindex="0">Bold</button>
  <button tabindex="-1">Italic</button>
  <button tabindex="-1">Underline</button>
  <button tabindex="-1">Strike</button>
</div>
```

```javascript
class RovingTabindex {
  constructor(container) {
    this.container = container;
    this.items = container.querySelectorAll('button');
    this.currentIndex = 0;
    
    this.items.forEach((item, index) => {
      item.addEventListener('keydown', (e) => this.handleKeydown(e, index));
      item.addEventListener('focus', () => this.updateTabindex(index));
    });
  }
  
  handleKeydown(e, index) {
    let newIndex = index;
    
    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowDown':
        e.preventDefault();
        newIndex = (index + 1) % this.items.length;
        break;
        
      case 'ArrowLeft':
      case 'ArrowUp':
        e.preventDefault();
        newIndex = (index - 1 + this.items.length) % this.items.length;
        break;
        
      case 'Home':
        e.preventDefault();
        newIndex = 0;
        break;
        
      case 'End':
        e.preventDefault();
        newIndex = this.items.length - 1;
        break;
        
      default:
        return;
    }
    
    this.items[newIndex].focus();
  }
  
  updateTabindex(index) {
    this.items.forEach((item, i) => {
      item.tabIndex = i === index ? 0 : -1;
    });
    this.currentIndex = index;
  }
}
```

## Best practices

### Do's

1. Ensure all interactive elements are focusable
2. Maintain logical focus order
3. Provide visible focus indicators
4. Restore focus after closing modals
5. Trap focus in modal dialogs
6. Use `tabindex="0"` for custom controls
7. Use `tabindex="-1"` for programmatic focus
8. Test with keyboard only
9. Use skip links for long navigation
10. Follow ARIA authoring practices

### Don'ts

1. Don't use positive tabindex values
2. Don't remove focus indicators without replacement
3. Don't trap focus outside modals
4. Don't break natural tab order
5. Don't make non-interactive elements focusable
6. Don't forget to restore focus
7. Don't rely on visual order if DOM order differs
8. Don't forget escape key for dismissable components

## Testing focus

### Keyboard testing

- `Tab` - Next focusable element
- `Shift + Tab` - Previous focusable element
- `Enter` - Activate links/buttons
- `Space` - Activate buttons, checkboxes
- `Arrow keys` - Navigate within components
- `Escape` - Close dialogs/menus
- `Home/End` - First/last item

### Browser DevTools

```javascript
// Track focus changes
document.addEventListener('focusin', (e) => {
  console.log('Focused:', e.target);
});

// Get currently focused element
console.log(document.activeElement);
```

### Screen reader testing

- NVDA (Windows, free)
- JAWS (Windows, commercial)
- VoiceOver (macOS, iOS, built-in)
- TalkBack (Android, built-in)

## Key takeaways

- All interactive elements must be keyboard accessible
- Follow natural DOM order for focus sequence
- Never remove focus indicators without replacement
- Use `tabindex="0"` to add to tab order
- Use `tabindex="-1"` for programmatic focus only
- Avoid positive tabindex values
- Trap focus in modal dialogs
- Restore focus after closing components
- Provide skip links for long navigation
- Test with keyboard only
- Implement proper keyboard patterns for widgets
- Use focus-visible for better UX
