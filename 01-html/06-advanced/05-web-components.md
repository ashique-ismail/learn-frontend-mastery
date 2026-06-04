# Web components

## Overview

Web Components are a suite of technologies that allow you to create reusable custom elements with encapsulated functionality and styling.

## Core technologies

### 1. Custom Elements

Define new HTML elements.

### 2. Shadow DOM

Encapsulated DOM and CSS.

### 3. HTML Templates

Reusable markup templates.

## Custom Elements

### Autonomous custom elements

New elements that don't extend existing HTML elements.

```javascript
class MyButton extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        button {
          background: blue;
          color: white;
          border: none;
          padding: 10px 20px;
          cursor: pointer;
        }
      </style>
      <button>
        <slot></slot>
      </button>
    `;
  }
}

customElements.define('my-button', MyButton);
```

```html
<my-button>Click me</my-button>
```

### Customized built-in elements

Extend existing HTML elements.

```javascript
class FancyButton extends HTMLButtonElement {
  constructor() {
    super();
    this.style.background = 'blue';
    this.style.color = 'white';
  }
  
  connectedCallback() {
    this.addEventListener('click', () => {
      console.log('Fancy button clicked!');
    });
  }
}

customElements.define('fancy-button', FancyButton, { extends: 'button' });
```

```html
<button is="fancy-button">Click me</button>
```

## Lifecycle callbacks

### connectedCallback

Called when element is inserted into DOM.

```javascript
class MyElement extends HTMLElement {
  connectedCallback() {
    console.log('Element added to page');
    this.render();
  }
}
```

### disconnectedCallback

Called when element is removed from DOM.

```javascript
class MyElement extends HTMLElement {
  disconnectedCallback() {
    console.log('Element removed from page');
    this.cleanup();
  }
}
```

### adoptedCallback

Called when element is moved to new document.

```javascript
class MyElement extends HTMLElement {
  adoptedCallback() {
    console.log('Element moved to new document');
  }
}
```

### attributeChangedCallback

Called when observed attributes change.

```javascript
class MyElement extends HTMLElement {
  static get observedAttributes() {
    return ['color', 'size'];
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`${name} changed from ${oldValue} to ${newValue}`);
    this.render();
  }
}
```

## Shadow DOM

Encapsulates component's internal structure and styles.

### Creating shadow root

```javascript
class MyCard extends HTMLElement {
  constructor() {
    super();
    // Attach shadow root
    this.attachShadow({ mode: 'open' });
  }
  
  connectedCallback() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          border: 1px solid #ccc;
          border-radius: 8px;
          padding: 16px;
        }
        
        h2 {
          margin: 0 0 8px 0;
          color: #333;
        }
      </style>
      
      <h2><slot name="title">Default Title</slot></h2>
      <div><slot></slot></div>
    `;
  }
}

customElements.define('my-card', MyCard);
```

```html
<my-card>
  <span slot="title">Card Title</span>
  <p>Card content goes here</p>
</my-card>
```

### Shadow DOM modes

#### Open mode

```javascript
// Shadow root accessible
this.attachShadow({ mode: 'open' });
console.log(element.shadowRoot); // Accessible
```

#### Closed mode

```javascript
// Shadow root not accessible
this.attachShadow({ mode: 'closed' });
console.log(element.shadowRoot); // null
```

### Styling shadow DOM

#### :host selector

```css
:host {
  display: block;
  border: 1px solid #ccc;
}

:host(.active) {
  border-color: blue;
}

:host([disabled]) {
  opacity: 0.5;
}
```

#### :host-context()

```css
:host-context(.dark-theme) {
  background: #333;
  color: white;
}
```

#### ::slotted()

```css
::slotted(h2) {
  color: blue;
}

::slotted(*) {
  font-family: sans-serif;
}
```

## Templates and slots

### Template element

```html
<template id="my-template">
  <style>
    .card {
      border: 1px solid #ccc;
      padding: 16px;
    }
  </style>
  
  <div class="card">
    <h2><slot name="title">Default Title</slot></h2>
    <div><slot></slot></div>
  </div>
</template>

<script>
class MyCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    
    const template = document.getElementById('my-template');
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}

customElements.define('my-card', MyCard);
</script>
```

### Named slots

```html
<template id="user-card">
  <div class="card">
    <div class="header">
      <slot name="avatar"></slot>
      <slot name="name"></slot>
    </div>
    <div class="body">
      <slot></slot>
    </div>
    <div class="footer">
      <slot name="actions"></slot>
    </div>
  </div>
</template>

<user-card>
  <img slot="avatar" src="avatar.jpg" alt="Avatar">
  <h3 slot="name">John Doe</h3>
  <p>User bio goes here</p>
  <button slot="actions">Follow</button>
</user-card>
```

## Complete component example

```javascript
class TodoList extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.todos = [];
  }
  
  static get observedAttributes() {
    return ['title'];
  }
  
  connectedCallback() {
    this.render();
    this.attachEventListeners();
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'title' && oldValue !== newValue) {
      this.render();
    }
  }
  
  render() {
    const title = this.getAttribute('title') || 'Todo List';
    
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: sans-serif;
        }
        
        h2 {
          margin: 0 0 16px 0;
        }
        
        input {
          padding: 8px;
          border: 1px solid #ccc;
          border-radius: 4px;
          flex: 1;
        }
        
        button {
          padding: 8px 16px;
          background: blue;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }
        
        .input-group {
          display: flex;
          gap: 8px;
          margin-bottom: 16px;
        }
        
        ul {
          list-style: none;
          padding: 0;
          margin: 0;
        }
        
        li {
          padding: 8px;
          border-bottom: 1px solid #eee;
          display: flex;
          justify-content: space-between;
          align-items: center;
        }
        
        .delete {
          background: red;
        }
      </style>
      
      <h2>${title}</h2>
      
      <div class="input-group">
        <input type="text" id="new-todo" placeholder="Add new todo">
        <button id="add-btn">Add</button>
      </div>
      
      <ul id="todo-list"></ul>
    `;
    
    this.renderTodos();
  }
  
  renderTodos() {
    const list = this.shadowRoot.getElementById('todo-list');
    list.innerHTML = this.todos.map((todo, index) => `
      <li>
        <span>${todo}</span>
        <button class="delete" data-index="${index}">Delete</button>
      </li>
    `).join('');
  }
  
  attachEventListeners() {
    const input = this.shadowRoot.getElementById('new-todo');
    const addBtn = this.shadowRoot.getElementById('add-btn');
    const list = this.shadowRoot.getElementById('todo-list');
    
    addBtn.addEventListener('click', () => this.addTodo());
    
    input.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        this.addTodo();
      }
    });
    
    list.addEventListener('click', (e) => {
      if (e.target.classList.contains('delete')) {
        const index = parseInt(e.target.dataset.index);
        this.deleteTodo(index);
      }
    });
  }
  
  addTodo() {
    const input = this.shadowRoot.getElementById('new-todo');
    const text = input.value.trim();
    
    if (text) {
      this.todos.push(text);
      input.value = '';
      this.renderTodos();
      
      // Dispatch custom event
      this.dispatchEvent(new CustomEvent('todo-added', {
        detail: { text },
        bubbles: true,
        composed: true
      }));
    }
  }
  
  deleteTodo(index) {
    const deleted = this.todos.splice(index, 1)[0];
    this.renderTodos();
    
    // Dispatch custom event
    this.dispatchEvent(new CustomEvent('todo-deleted', {
      detail: { text: deleted },
      bubbles: true,
      composed: true
    }));
  }
}

customElements.define('todo-list', TodoList);
```

```html
<todo-list title="My Tasks"></todo-list>

<script>
const todoList = document.querySelector('todo-list');

todoList.addEventListener('todo-added', (e) => {
  console.log('Todo added:', e.detail.text);
});

todoList.addEventListener('todo-deleted', (e) => {
  console.log('Todo deleted:', e.detail.text);
});
</script>
```

## Properties and attributes

### Reflecting attributes to properties

```javascript
class MyElement extends HTMLElement {
  static get observedAttributes() {
    return ['title', 'color'];
  }
  
  get title() {
    return this.getAttribute('title');
  }
  
  set title(value) {
    this.setAttribute('title', value);
  }
  
  get color() {
    return this.getAttribute('color');
  }
  
  set color(value) {
    this.setAttribute('color', value);
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    this.render();
  }
}
```

## Custom events

### Dispatching events

```javascript
class MyButton extends HTMLElement {
  connectedCallback() {
    this.addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('my-click', {
        detail: { timestamp: Date.now() },
        bubbles: true,
        composed: true // Cross shadow DOM boundary
      }));
    });
  }
}
```

```html
<my-button id="btn">Click</my-button>

<script>
document.getElementById('btn').addEventListener('my-click', (e) => {
  console.log('Clicked at:', e.detail.timestamp);
});
</script>
```

## Styling from outside

### CSS custom properties

```javascript
class MyCard extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          --card-bg: white;
          --card-border: #ccc;
          
          background: var(--card-bg);
          border: 1px solid var(--card-border);
          padding: 16px;
        }
      </style>
      
      <slot></slot>
    `;
  }
}

customElements.define('my-card', MyCard);
```

```css
my-card {
  --card-bg: lightblue;
  --card-border: blue;
}
```

### Part pseudo-element

```javascript
class MyCard extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: 'open' });
    this.shadowRoot.innerHTML = `
      <style>
        .header { padding: 8px; background: #f5f5f5; }
        .body { padding: 16px; }
      </style>
      
      <div class="header" part="header">
        <slot name="title"></slot>
      </div>
      <div class="body" part="body">
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('my-card', MyCard);
```

```css
my-card::part(header) {
  background: blue;
  color: white;
}

my-card::part(body) {
  background: lightgray;
}
```

## Best practices

### 1. Use semantic naming

```javascript
// Good: descriptive name with hyphen
customElements.define('user-profile', UserProfile);

// Bad: generic or single word
customElements.define('profile', Profile);
```

### 2. Clean up in disconnectedCallback

```javascript
class MyElement extends HTMLElement {
  connectedCallback() {
    this.interval = setInterval(() => {
      this.update();
    }, 1000);
  }
  
  disconnectedCallback() {
    clearInterval(this.interval);
  }
}
```

### 3. Use progressive enhancement

```html
<my-button>
  <button>Fallback button</button>
</my-button>
```

### 4. Handle accessibility

```javascript
class MyButton extends HTMLElement {
  connectedCallback() {
    if (!this.hasAttribute('role')) {
      this.setAttribute('role', 'button');
    }
    
    if (!this.hasAttribute('tabindex')) {
      this.setAttribute('tabindex', '0');
    }
    
    this.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        this.click();
      }
    });
  }
}
```

### 5. Compose events properly

```javascript
this.dispatchEvent(new CustomEvent('change', {
  bubbles: true,      // Bubble through ancestors
  composed: true,     // Cross shadow boundaries
  detail: { value }   // Custom data
}));
```

## Browser support

Modern browsers support web components:
- Chrome 54+
- Firefox 63+
- Safari 10.1+
- Edge 79+

### Polyfills

For older browsers:

```html
<script src="https://unpkg.com/@webcomponents/webcomponentsjs@latest/webcomponents-loader.js"></script>
```

## Key takeaways

- Web Components = Custom Elements + Shadow DOM + Templates
- Use shadow DOM for style/DOM encapsulation
- Lifecycle callbacks for component lifecycle management
- Slots for content projection
- Custom events for component communication
- CSS custom properties for external styling
- Always handle accessibility
- Clean up resources in disconnectedCallback
- Use semantic naming with hyphens
- Test cross-browser compatibility
- Consider polyfills for older browsers
