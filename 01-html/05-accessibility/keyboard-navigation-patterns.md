# Keyboard navigation patterns

## Overview

Keyboard navigation patterns define how users interact with UI components using only the keyboard. Following established patterns ensures consistency and usability.

## Basic keyboard interactions

### Standard keys

- **Tab** - Move to next focusable element
- **Shift + Tab** - Move to previous focusable element
- **Enter** - Activate links and buttons
- **Space** - Activate buttons, toggle checkboxes
- **Escape** - Close dialogs, menus, cancel actions
- **Arrow keys** - Navigate within components
- **Home** - First item in component
- **End** - Last item in component

## Common patterns

### Button

```html
<button onclick="handleClick()">Click Me</button>
```

**Keyboard:**
- `Enter` or `Space` - Activate

**Implementation:**
```javascript
button.addEventListener('keydown', (e) => {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();
    handleClick();
  }
});
```

### Link

```html
<a href="/page">Go to page</a>
```

**Keyboard:**
- `Enter` - Activate (navigate)

### Checkbox

```html
<input type="checkbox" id="agree">
<label for="agree">I agree</label>
```

**Keyboard:**
- `Space` - Toggle checked state

### Radio group

```html
<fieldset>
  <legend>Choose option</legend>
  <input type="radio" name="option" id="opt1">
  <label for="opt1">Option 1</label>
  <input type="radio" name="option" id="opt2">
  <label for="opt2">Option 2</label>
  <input type="radio" name="option" id="opt3">
  <label for="opt3">Option 3</label>
</fieldset>
```

**Keyboard:**
- `Tab` - Focus group (first checked or first item)
- `Arrow keys` - Move between options
- `Space` - Select focused option

### Select dropdown

```html
<label for="country">Country</label>
<select id="country">
  <option>USA</option>
  <option>Canada</option>
  <option>Mexico</option>
</select>
```

**Keyboard:**
- `Enter` or `Space` - Open dropdown
- `Arrow Up/Down` - Navigate options
- `Home/End` - First/last option
- `Enter` - Select and close
- `Escape` - Close without selecting

## Widget patterns

### Tabs

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
    Content 1
  </div>
  <div role="tabpanel" id="panel2" aria-labelledby="tab2" tabindex="0" hidden>
    Content 2
  </div>
  <div role="tabpanel" id="panel3" aria-labelledby="tab3" tabindex="0" hidden>
    Content 3
  </div>
</div>
```

**Keyboard:**
- `Tab` - Focus tab list, then into panel
- `Arrow Left/Right` - Navigate between tabs
- `Home` - First tab
- `End` - Last tab
- `Tab` from tab - Focus panel

**Implementation:**
```javascript
const tabs = document.querySelectorAll('[role="tab"]');
let currentIndex = 0;

tabs.forEach((tab, index) => {
  tab.addEventListener('keydown', (e) => {
    switch (e.key) {
      case 'ArrowRight':
        e.preventDefault();
        currentIndex = (index + 1) % tabs.length;
        selectTab(currentIndex);
        break;
        
      case 'ArrowLeft':
        e.preventDefault();
        currentIndex = (index - 1 + tabs.length) % tabs.length;
        selectTab(currentIndex);
        break;
        
      case 'Home':
        e.preventDefault();
        selectTab(0);
        break;
        
      case 'End':
        e.preventDefault();
        selectTab(tabs.length - 1);
        break;
    }
  });
});

function selectTab(index) {
  tabs.forEach((tab, i) => {
    const isSelected = i === index;
    tab.setAttribute('aria-selected', isSelected);
    tab.tabIndex = isSelected ? 0 : -1;
    
    if (isSelected) {
      tab.focus();
    }
    
    const panel = document.getElementById(tab.getAttribute('aria-controls'));
    panel.hidden = !isSelected;
  });
}
```

### Accordion

```html
<div class="accordion">
  <h3>
    <button 
      id="accordion1" 
      aria-expanded="false" 
      aria-controls="panel1">
      Section 1
    </button>
  </h3>
  <div id="panel1" role="region" aria-labelledby="accordion1" hidden>
    <p>Content 1</p>
  </div>
  
  <h3>
    <button 
      id="accordion2" 
      aria-expanded="false" 
      aria-controls="panel2">
      Section 2
    </button>
  </h3>
  <div id="panel2" role="region" aria-labelledby="accordion2" hidden>
    <p>Content 2</p>
  </div>
</div>
```

**Keyboard:**
- `Tab` - Move between accordion buttons
- `Enter` or `Space` - Toggle panel
- `Arrow Down` (optional) - Next button
- `Arrow Up` (optional) - Previous button

**Implementation:**
```javascript
const accordionButtons = document.querySelectorAll('.accordion button');

accordionButtons.forEach((button) => {
  button.addEventListener('click', () => {
    toggleAccordion(button);
  });
  
  button.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      toggleAccordion(button);
    }
  });
});

function toggleAccordion(button) {
  const expanded = button.getAttribute('aria-expanded') === 'true';
  const panel = document.getElementById(button.getAttribute('aria-controls'));
  
  button.setAttribute('aria-expanded', !expanded);
  panel.hidden = expanded;
}
```

### Menu/dropdown

```html
<nav>
  <button 
    id="menu-button"
    aria-haspopup="true" 
    aria-expanded="false"
    aria-controls="menu">
    Menu
  </button>
  
  <ul id="menu" role="menu" hidden>
    <li role="none">
      <button role="menuitem">New</button>
    </li>
    <li role="none">
      <button role="menuitem">Open</button>
    </li>
    <li role="none">
      <button role="menuitem">Save</button>
    </li>
  </ul>
</nav>
```

**Keyboard:**
- `Enter`, `Space`, or `Arrow Down` - Open menu
- `Arrow Down/Up` - Navigate items
- `Home` - First item
- `End` - Last item
- `Enter` - Select item
- `Escape` - Close menu
- `Tab` - Close menu and move to next element

**Implementation:**
```javascript
class Menu {
  constructor(button, menu) {
    this.button = button;
    this.menu = menu;
    this.items = menu.querySelectorAll('[role="menuitem"]');
    this.currentIndex = 0;
    
    this.button.addEventListener('click', () => this.toggle());
    this.button.addEventListener('keydown', (e) => this.handleButtonKey(e));
    this.menu.addEventListener('keydown', (e) => this.handleMenuKey(e));
  }
  
  open() {
    this.menu.hidden = false;
    this.button.setAttribute('aria-expanded', 'true');
    this.items[0].focus();
    this.currentIndex = 0;
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
      case 'Enter':
      case ' ':
      case 'ArrowDown':
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
        
      case 'Tab':
        this.close();
        break;
    }
  }
}
```

### Modal dialog

```html
<div 
  role="dialog" 
  aria-modal="true"
  aria-labelledby="dialog-title"
  class="dialog">
  
  <h2 id="dialog-title">Dialog Title</h2>
  <p>Dialog content</p>
  
  <button id="cancel">Cancel</button>
  <button id="confirm">Confirm</button>
</div>
```

**Keyboard:**
- `Tab` - Cycle through focusable elements (trapped)
- `Shift + Tab` - Cycle backward (trapped)
- `Escape` - Close dialog

**Implementation:**
```javascript
class Dialog {
  constructor(dialogElement) {
    this.dialog = dialogElement;
    this.focusableElements = this.dialog.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    this.firstFocusable = this.focusableElements[0];
    this.lastFocusable = this.focusableElements[this.focusableElements.length - 1];
  }
  
  open() {
    this.lastFocused = document.activeElement;
    this.dialog.style.display = 'block';
    this.firstFocusable.focus();
    this.trapFocus();
  }
  
  close() {
    this.dialog.style.display = 'none';
    this.lastFocused.focus();
    this.releaseFocus();
  }
  
  trapFocus() {
    this.handleTab = (e) => {
      if (e.key !== 'Tab') {
        if (e.key === 'Escape') {
          this.close();
        }
        return;
      }
      
      if (e.shiftKey) {
        if (document.activeElement === this.firstFocusable) {
          e.preventDefault();
          this.lastFocusable.focus();
        }
      } else {
        if (document.activeElement === this.lastFocusable) {
          e.preventDefault();
          this.firstFocusable.focus();
        }
      }
    };
    
    this.dialog.addEventListener('keydown', this.handleTab);
  }
  
  releaseFocus() {
    this.dialog.removeEventListener('keydown', this.handleTab);
  }
}
```

### Listbox

```html
<div>
  <label id="listbox-label">Choose option</label>
  <div 
    role="listbox" 
    aria-labelledby="listbox-label"
    tabindex="0">
    <div role="option" aria-selected="true">Option 1</div>
    <div role="option">Option 2</div>
    <div role="option">Option 3</div>
  </div>
</div>
```

**Keyboard:**
- `Arrow Down/Up` - Navigate options
- `Home` - First option
- `End` - Last option
- `Space` - Select option (multi-select)
- `Enter` - Select option and close (single-select)

### Slider

```html
<div 
  role="slider"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="50"
  aria-label="Volume"
  tabindex="0">
</div>
```

**Keyboard:**
- `Arrow Right/Up` - Increase value
- `Arrow Left/Down` - Decrease value
- `Home` - Minimum value
- `End` - Maximum value
- `Page Up` - Large increase
- `Page Down` - Large decrease

**Implementation:**
```javascript
class Slider {
  constructor(element) {
    this.slider = element;
    this.min = parseInt(element.getAttribute('aria-valuemin'));
    this.max = parseInt(element.getAttribute('aria-valuemax'));
    this.value = parseInt(element.getAttribute('aria-valuenow'));
    
    this.slider.addEventListener('keydown', (e) => this.handleKey(e));
  }
  
  handleKey(e) {
    const step = 1;
    const largeStep = 10;
    let newValue = this.value;
    
    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowUp':
        e.preventDefault();
        newValue = Math.min(this.value + step, this.max);
        break;
        
      case 'ArrowLeft':
      case 'ArrowDown':
        e.preventDefault();
        newValue = Math.max(this.value - step, this.min);
        break;
        
      case 'PageUp':
        e.preventDefault();
        newValue = Math.min(this.value + largeStep, this.max);
        break;
        
      case 'PageDown':
        e.preventDefault();
        newValue = Math.max(this.value - largeStep, this.min);
        break;
        
      case 'Home':
        e.preventDefault();
        newValue = this.min;
        break;
        
      case 'End':
        e.preventDefault();
        newValue = this.max;
        break;
        
      default:
        return;
    }
    
    this.setValue(newValue);
  }
  
  setValue(value) {
    this.value = value;
    this.slider.setAttribute('aria-valuenow', value);
    // Update visual representation
  }
}
```

### Combobox (autocomplete)

```html
<div class="combobox">
  <label for="combo">Search</label>
  <input 
    type="text" 
    id="combo"
    role="combobox"
    aria-expanded="false"
    aria-controls="listbox"
    aria-autocomplete="list">
  
  <ul id="listbox" role="listbox" hidden>
    <li role="option">Result 1</li>
    <li role="option">Result 2</li>
    <li role="option">Result 3</li>
  </ul>
</div>
```

**Keyboard:**
- `Type` - Filter results, show listbox
- `Arrow Down` - Open listbox and move to first item
- `Arrow Down/Up` - Navigate options
- `Enter` - Select option
- `Escape` - Close listbox

## Best practices

### 1. Follow ARIA Authoring Practices

Use established patterns from WAI-ARIA Authoring Practices Guide.

### 2. Prevent default when handling keys

```javascript
element.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowDown') {
    e.preventDefault(); // Prevent page scroll
    // Handle arrow down
  }
});
```

### 3. Manage focus explicitly

```javascript
// Focus first item when opening
menu.items[0].focus();

// Restore focus when closing
lastFocusedElement.focus();
```

### 4. Use roving tabindex for groups

```javascript
// Only one item in group is tabbable
items.forEach((item, i) => {
  item.tabIndex = i === activeIndex ? 0 : -1;
});
```

### 5. Provide visual focus indicators

```css
button:focus-visible {
  outline: 2px solid #0066CC;
  outline-offset: 2px;
}
```

### 6. Test with keyboard only

Unplug mouse and navigate entire application.

## Common mistakes

### Missing keyboard support

```html
<!-- Bad: div button without keyboard handler -->
<div onclick="handleClick()">Click me</div>

<!-- Good: proper button or keyboard handler -->
<button onclick="handleClick()">Click me</button>
```

### Wrong keys for pattern

```javascript
// Bad: using Enter for checkbox
checkbox.addEventListener('keydown', (e) => {
  if (e.key === 'Enter') { // Should be Space
    toggle();
  }
});
```

### Not preventing default

```javascript
// Bad: allows default behavior
element.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowDown') {
    nextItem(); // Page also scrolls
  }
});

// Good: prevents default
element.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowDown') {
    e.preventDefault();
    nextItem();
  }
});
```

### No focus management

```javascript
// Bad: focus not managed
function openDialog() {
  dialog.style.display = 'block';
  // Focus stays on button
}

// Good: focus moved to dialog
function openDialog() {
  dialog.style.display = 'block';
  dialog.querySelector('button').focus();
}
```

## Testing

### Manual testing

1. Unplug mouse
2. Use Tab to navigate
3. Try all keyboard patterns
4. Verify focus indicators visible
5. Test with screen reader

### Automated testing

```javascript
// Check for keyboard event listeners
const button = document.querySelector('button');
console.log(getEventListeners(button));
```

## Resources

- WAI-ARIA Authoring Practices Guide
- Keyboard shortcuts for screen readers
- MDN Web Docs - Keyboard-navigable JavaScript widgets

## Key takeaways

- All interactive elements must be keyboard accessible
- Follow established keyboard patterns for widgets
- Tab/Shift+Tab for between elements
- Arrow keys for within elements
- Always prevent default for handled keys
- Manage focus explicitly
- Use roving tabindex for groups
- Trap focus in modal dialogs
- Restore focus when closing components
- Provide visible focus indicators
- Test with keyboard only
- Follow ARIA Authoring Practices patterns
