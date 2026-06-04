# HTML Templates

## Introduction

HTML Templates are a web standard that allows developers to declare fragments of HTML that are not rendered when the page loads but can be instantiated at runtime using JavaScript. The `<template>` element provides a mechanism for holding client-side content that won't be rendered until it's explicitly activated, making it perfect for creating reusable component templates.

Templates are one of the three core technologies that make up Web Components, alongside Custom Elements and Shadow DOM. They enable efficient DOM manipulation by allowing you to clone pre-parsed HTML structures rather than building them from strings or creating elements one by one.

## Template Element Fundamentals

### Basic Template Usage

```html
<template id="my-template">
  <style>
    .card {
      border: 1px solid #ddd;
      padding: 16px;
      border-radius: 8px;
    }
  </style>
  <div class="card">
    <h3 class="title">Title</h3>
    <p class="content">Content goes here</p>
  </div>
</template>

<script>
// Get the template
const template = document.getElementById('my-template');

// Clone the template content
const clone = template.content.cloneNode(true);

// Modify the clone
clone.querySelector('.title').textContent = 'My Card';
clone.querySelector('.content').textContent = 'This is my card content';

// Add to document
document.body.appendChild(clone);
</script>
```

### Template Content Property

The `content` property returns a DocumentFragment containing the template's contents.

```javascript
const template = document.getElementById('my-template');

// Access template content
console.log(template.content); // DocumentFragment

// Template content is not in the main document
console.log(template.content.parentNode); // null

// Query elements inside template
const titleElement = template.content.querySelector('.title');
console.log(titleElement);

// Clone with deep = true to include all descendants
const clone = template.content.cloneNode(true);
```

## Creating Templates with JavaScript

### Dynamic Template Creation

```javascript
function createTemplate(html) {
  const template = document.createElement('template');
  template.innerHTML = html;
  return template;
}

const userCardTemplate = createTemplate(`
  <style>
    .user-card {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 12px;
      border: 1px solid #e0e0e0;
      border-radius: 8px;
    }
    
    .avatar {
      width: 48px;
      height: 48px;
      border-radius: 50%;
      background: #667eea;
      color: white;
      display: flex;
      align-items: center;
      justify-content: center;
      font-weight: bold;
    }
    
    .info {
      flex: 1;
    }
    
    .name {
      font-weight: bold;
      margin: 0 0 4px 0;
    }
    
    .email {
      color: #666;
      font-size: 14px;
      margin: 0;
    }
  </style>
  
  <div class="user-card">
    <div class="avatar"></div>
    <div class="info">
      <p class="name"></p>
      <p class="email"></p>
    </div>
  </div>
`);

// Use the template
function createUserCard(name, email) {
  const clone = userCardTemplate.content.cloneNode(true);
  
  clone.querySelector('.avatar').textContent = name.charAt(0).toUpperCase();
  clone.querySelector('.name').textContent = name;
  clone.querySelector('.email').textContent = email;
  
  return clone;
}

// Create multiple cards
const users = [
  { name: 'John Doe', email: 'john@example.com' },
  { name: 'Jane Smith', email: 'jane@example.com' },
  { name: 'Bob Wilson', email: 'bob@example.com' }
];

const container = document.getElementById('users-container');
users.forEach(user => {
  container.appendChild(createUserCard(user.name, user.email));
});
```

### Template Factory Pattern

```javascript
class TemplateFactory {
  constructor() {
    this.templates = new Map();
  }
  
  register(name, html) {
    const template = document.createElement('template');
    template.innerHTML = html;
    this.templates.set(name, template);
  }
  
  create(name, data = {}) {
    const template = this.templates.get(name);
    if (!template) {
      throw new Error(`Template "${name}" not found`);
    }
    
    const clone = template.content.cloneNode(true);
    
    // Replace placeholders with data
    this.fillData(clone, data);
    
    return clone;
  }
  
  fillData(fragment, data) {
    // Replace text content
    fragment.querySelectorAll('[data-text]').forEach(el => {
      const key = el.dataset.text;
      if (key in data) {
        el.textContent = data[key];
      }
    });
    
    // Set attributes
    fragment.querySelectorAll('[data-attr]').forEach(el => {
      const mappings = JSON.parse(el.dataset.attr);
      Object.entries(mappings).forEach(([attr, key]) => {
        if (key in data) {
          el.setAttribute(attr, data[key]);
        }
      });
    });
  }
}

// Usage
const factory = new TemplateFactory();

factory.register('product-card', `
  <div class="product-card">
    <img data-attr='{"src":"image","alt":"name"}'>
    <h3 data-text="name"></h3>
    <p data-text="description"></p>
    <span data-text="price"></span>
  </div>
`);

const product = {
  name: 'Laptop',
  description: 'High-performance laptop',
  price: '$999',
  image: 'laptop.jpg'
};

const productCard = factory.create('product-card', product);
document.body.appendChild(productCard);
```

## Templates in Custom Elements

### Basic Custom Element with Template

```javascript
class ProductList extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    
    // Define template inside the constructor
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: Arial, sans-serif;
        }
        
        .list {
          display: grid;
          grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
          gap: 16px;
          padding: 16px;
        }
        
        .empty {
          text-align: center;
          padding: 48px;
          color: #999;
        }
      </style>
      
      <div class="list"></div>
    `;
    
    this.shadowRoot.appendChild(template.content.cloneNode(true));
    this.listElement = this.shadowRoot.querySelector('.list');
  }
  
  // Product item template
  getItemTemplate() {
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        .product {
          border: 1px solid #e0e0e0;
          border-radius: 8px;
          padding: 16px;
          cursor: pointer;
          transition: transform 0.2s, box-shadow 0.2s;
        }
        
        .product:hover {
          transform: translateY(-4px);
          box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        
        .product-name {
          font-size: 18px;
          font-weight: bold;
          margin: 0 0 8px 0;
        }
        
        .product-price {
          font-size: 24px;
          color: #667eea;
          font-weight: bold;
        }
      </style>
      
      <div class="product">
        <h3 class="product-name"></h3>
        <div class="product-price"></div>
      </div>
    `;
    return template;
  }
  
  setProducts(products) {
    this.listElement.innerHTML = '';
    
    if (!products || products.length === 0) {
      this.listElement.innerHTML = '<div class="empty">No products available</div>';
      return;
    }
    
    products.forEach(product => {
      const template = this.getItemTemplate();
      const clone = template.content.cloneNode(true);
      
      clone.querySelector('.product-name').textContent = product.name;
      clone.querySelector('.product-price').textContent = `$${product.price}`;
      
      const productElement = clone.querySelector('.product');
      productElement.addEventListener('click', () => {
        this.dispatchEvent(new CustomEvent('product-selected', {
          detail: product,
          bubbles: true,
          composed: true
        }));
      });
      
      this.listElement.appendChild(clone);
    });
  }
}

customElements.define('product-list', ProductList);

// Usage
const productList = document.querySelector('product-list');
productList.setProducts([
  { name: 'Laptop', price: 999 },
  { name: 'Mouse', price: 29 },
  { name: 'Keyboard', price: 79 }
]);
```

### Template with Nested Components

```javascript
class TableRow extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  connectedCallback() {
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        :host {
          display: table-row;
        }
        
        ::slotted(td) {
          padding: 12px;
          border-bottom: 1px solid #e0e0e0;
        }
      </style>
      
      <slot></slot>
    `;
    
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}

class DataTable extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: Arial, sans-serif;
        }
        
        table {
          width: 100%;
          border-collapse: collapse;
        }
        
        thead {
          background: #f5f5f5;
        }
        
        th {
          padding: 12px;
          text-align: left;
          font-weight: bold;
          border-bottom: 2px solid #667eea;
        }
        
        tbody ::slotted(table-row) {
          transition: background-color 0.2s;
        }
        
        tbody ::slotted(table-row:hover) {
          background-color: #f9f9f9;
        }
      </style>
      
      <table>
        <thead>
          <tr>
            <slot name="header"></slot>
          </tr>
        </thead>
        <tbody>
          <slot></slot>
        </tbody>
      </table>
    `;
    
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
  
  setData(columns, data) {
    // Clear existing content
    this.innerHTML = '';
    
    // Create header
    columns.forEach(col => {
      const th = document.createElement('th');
      th.textContent = col.label;
      th.slot = 'header';
      this.appendChild(th);
    });
    
    // Create rows
    const rowTemplate = this.getRowTemplate(columns);
    data.forEach(item => {
      const clone = rowTemplate.content.cloneNode(true);
      const row = clone.querySelector('table-row');
      
      columns.forEach(col => {
        const td = document.createElement('td');
        td.textContent = item[col.key];
        row.appendChild(td);
      });
      
      this.appendChild(clone);
    });
  }
  
  getRowTemplate(columns) {
    const template = document.createElement('template');
    template.innerHTML = '<table-row></table-row>';
    return template;
  }
}

customElements.define('table-row', TableRow);
customElements.define('data-table', DataTable);

// Usage
const table = document.querySelector('data-table');
table.setData(
  [
    { key: 'name', label: 'Name' },
    { key: 'age', label: 'Age' },
    { key: 'email', label: 'Email' }
  ],
  [
    { name: 'John Doe', age: 30, email: 'john@example.com' },
    { name: 'Jane Smith', age: 25, email: 'jane@example.com' }
  ]
);
```

## Document Fragments

Document fragments are lightweight containers that can hold DOM nodes without being part of the main document tree.

### Creating and Using Fragments

```javascript
// Create a fragment
const fragment = document.createDocumentFragment();

// Add elements to the fragment
for (let i = 0; i < 1000; i++) {
  const div = document.createElement('div');
  div.textContent = `Item ${i}`;
  div.className = 'item';
  fragment.appendChild(div);
}

// Single append - much faster than 1000 individual appends
document.getElementById('container').appendChild(fragment);
```

### Fragment Performance Benefits

```javascript
class ListRenderer {
  constructor(container) {
    this.container = container;
  }
  
  // Inefficient - causes reflow for each item
  renderSlow(items) {
    console.time('slow');
    items.forEach(item => {
      const div = document.createElement('div');
      div.textContent = item;
      this.container.appendChild(div); // Reflow for each append
    });
    console.timeEnd('slow');
  }
  
  // Efficient - single reflow
  renderFast(items) {
    console.time('fast');
    const fragment = document.createDocumentFragment();
    
    items.forEach(item => {
      const div = document.createElement('div');
      div.textContent = item;
      fragment.appendChild(div); // No reflow
    });
    
    this.container.appendChild(fragment); // Single reflow
    console.timeEnd('fast');
  }
  
  // Using template
  renderWithTemplate(items) {
    console.time('template');
    const template = document.createElement('template');
    template.innerHTML = '<div class="item"></div>';
    
    const fragment = document.createDocumentFragment();
    
    items.forEach(item => {
      const clone = template.content.cloneNode(true);
      clone.querySelector('.item').textContent = item;
      fragment.appendChild(clone);
    });
    
    this.container.appendChild(fragment);
    console.timeEnd('template');
  }
}

// Test with large dataset
const items = Array.from({ length: 1000 }, (_, i) => `Item ${i}`);
const renderer = new ListRenderer(document.getElementById('container'));

// renderSlow: ~50ms
// renderFast: ~10ms
// renderWithTemplate: ~12ms
```

## Advanced Template Patterns

### Template with Event Delegation

```javascript
class TodoList extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.todos = [];
    
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: Arial, sans-serif;
          max-width: 500px;
          margin: 0 auto;
        }
        
        .input-container {
          display: flex;
          gap: 8px;
          margin-bottom: 16px;
        }
        
        input {
          flex: 1;
          padding: 8px 12px;
          border: 1px solid #ddd;
          border-radius: 4px;
        }
        
        button {
          padding: 8px 16px;
          background: #667eea;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }
        
        .todo-list {
          list-style: none;
          padding: 0;
          margin: 0;
        }
        
        .todo-item {
          display: flex;
          align-items: center;
          gap: 8px;
          padding: 12px;
          border: 1px solid #e0e0e0;
          border-radius: 4px;
          margin-bottom: 8px;
        }
        
        .todo-item.completed {
          opacity: 0.6;
          text-decoration: line-through;
        }
        
        .todo-text {
          flex: 1;
        }
        
        .delete-btn {
          background: #ff6b6b;
          padding: 4px 8px;
          font-size: 12px;
        }
      </style>
      
      <div class="input-container">
        <input type="text" id="todo-input" placeholder="Enter a todo...">
        <button id="add-btn">Add</button>
      </div>
      
      <ul class="todo-list"></ul>
    `;
    
    this.shadowRoot.appendChild(template.content.cloneNode(true));
    
    // Event delegation on container
    this.shadowRoot.querySelector('.todo-list').addEventListener('click', (e) => {
      const todoItem = e.target.closest('.todo-item');
      if (!todoItem) return;
      
      const id = parseInt(todoItem.dataset.id, 10);
      
      if (e.target.classList.contains('delete-btn')) {
        this.deleteTodo(id);
      } else {
        this.toggleTodo(id);
      }
    });
    
    this.shadowRoot.querySelector('#add-btn').addEventListener('click', () => {
      this.addTodo();
    });
    
    this.shadowRoot.querySelector('#todo-input').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') {
        this.addTodo();
      }
    });
  }
  
  getTodoTemplate() {
    const template = document.createElement('template');
    template.innerHTML = `
      <li class="todo-item">
        <input type="checkbox" class="checkbox">
        <span class="todo-text"></span>
        <button class="delete-btn">Delete</button>
      </li>
    `;
    return template;
  }
  
  addTodo() {
    const input = this.shadowRoot.querySelector('#todo-input');
    const text = input.value.trim();
    
    if (!text) return;
    
    const todo = {
      id: Date.now(),
      text,
      completed: false
    };
    
    this.todos.push(todo);
    this.render();
    input.value = '';
  }
  
  toggleTodo(id) {
    const todo = this.todos.find(t => t.id === id);
    if (todo) {
      todo.completed = !todo.completed;
      this.render();
    }
  }
  
  deleteTodo(id) {
    this.todos = this.todos.filter(t => t.id !== id);
    this.render();
  }
  
  render() {
    const list = this.shadowRoot.querySelector('.todo-list');
    list.innerHTML = '';
    
    const fragment = document.createDocumentFragment();
    const template = this.getTodoTemplate();
    
    this.todos.forEach(todo => {
      const clone = template.content.cloneNode(true);
      const item = clone.querySelector('.todo-item');
      
      item.dataset.id = todo.id;
      if (todo.completed) {
        item.classList.add('completed');
      }
      
      clone.querySelector('.checkbox').checked = todo.completed;
      clone.querySelector('.todo-text').textContent = todo.text;
      
      fragment.appendChild(clone);
    });
    
    list.appendChild(fragment);
  }
}

customElements.define('todo-list', TodoList);
```

### Template Caching Strategy

```javascript
class TemplateCache {
  constructor() {
    this.cache = new Map();
  }
  
  get(key, factory) {
    if (!this.cache.has(key)) {
      const template = document.createElement('template');
      template.innerHTML = factory();
      this.cache.set(key, template);
    }
    return this.cache.get(key);
  }
  
  clear() {
    this.cache.clear();
  }
  
  delete(key) {
    this.cache.delete(key);
  }
}

const templateCache = new TemplateCache();

class OptimizedList extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  render(items) {
    // Get cached template
    const template = templateCache.get('list-item', () => `
      <div class="item">
        <span class="name"></span>
        <span class="value"></span>
      </div>
    `);
    
    const fragment = document.createDocumentFragment();
    
    items.forEach(item => {
      const clone = template.content.cloneNode(true);
      clone.querySelector('.name').textContent = item.name;
      clone.querySelector('.value').textContent = item.value;
      fragment.appendChild(clone);
    });
    
    this.shadowRoot.appendChild(fragment);
  }
}

customElements.define('optimized-list', OptimizedList);
```

### Conditional Templates

```javascript
class ConditionalRenderer extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this.templates = this.createTemplates();
  }
  
  createTemplates() {
    return {
      loading: this.createTemplate(`
        <div class="loading">
          <div class="spinner"></div>
          <p>Loading...</p>
        </div>
      `),
      
      error: this.createTemplate(`
        <div class="error">
          <span class="icon">⚠️</span>
          <p class="message"></p>
          <button class="retry">Retry</button>
        </div>
      `),
      
      success: this.createTemplate(`
        <div class="success">
          <h2 class="title"></h2>
          <div class="content"></div>
        </div>
      `),
      
      empty: this.createTemplate(`
        <div class="empty">
          <p>No data available</p>
        </div>
      `)
    };
  }
  
  createTemplate(html) {
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        .loading {
          text-align: center;
          padding: 48px;
        }
        
        .spinner {
          width: 40px;
          height: 40px;
          border: 4px solid #f3f3f3;
          border-top: 4px solid #667eea;
          border-radius: 50%;
          animation: spin 1s linear infinite;
          margin: 0 auto 16px;
        }
        
        @keyframes spin {
          to { transform: rotate(360deg); }
        }
        
        .error {
          text-align: center;
          padding: 48px;
          color: #d32f2f;
        }
        
        .success {
          padding: 16px;
        }
        
        .empty {
          text-align: center;
          padding: 48px;
          color: #999;
        }
      </style>
      ${html}
    `;
    return template;
  }
  
  renderState(state, data = {}) {
    const template = this.templates[state];
    if (!template) {
      console.error(`Template for state "${state}" not found`);
      return;
    }
    
    const clone = template.content.cloneNode(true);
    
    // Fill in data based on state
    if (state === 'error') {
      clone.querySelector('.message').textContent = data.message || 'An error occurred';
      clone.querySelector('.retry').addEventListener('click', data.onRetry);
    } else if (state === 'success') {
      clone.querySelector('.title').textContent = data.title;
      clone.querySelector('.content').innerHTML = data.content;
    }
    
    this.shadowRoot.innerHTML = '';
    this.shadowRoot.appendChild(clone);
  }
}

customElements.define('conditional-renderer', ConditionalRenderer);

// Usage
const renderer = document.querySelector('conditional-renderer');

// Show loading
renderer.renderState('loading');

// Simulate API call
setTimeout(() => {
  // Show success
  renderer.renderState('success', {
    title: 'Data Loaded',
    content: '<p>Here is your data...</p>'
  });
}, 2000);
```

## Browser Support

| Browser | Version | Template Element | DocumentFragment |
|---------|---------|------------------|------------------|
| Chrome | 26+ | ✓ | ✓ |
| Firefox | 22+ | ✓ | ✓ |
| Safari | 8+ | ✓ | ✓ |
| Edge | All | ✓ | ✓ |
| IE | 11* | Polyfill | ✓ |

## Common Mistakes

1. **Not Cloning Templates**
```javascript
// Wrong - modifies the template itself
const template = document.getElementById('my-template');
template.content.querySelector('.title').textContent = 'Hello';

// Correct - clone first
const clone = template.content.cloneNode(true);
clone.querySelector('.title').textContent = 'Hello';
```

2. **Shallow Cloning**
```javascript
// Wrong - shallow clone doesn't include children
const clone = template.content.cloneNode(false);

// Correct - deep clone includes all descendants
const clone = template.content.cloneNode(true);
```

3. **Forgetting to Append**
```javascript
// Wrong - clone is created but not added to document
const clone = template.content.cloneNode(true);
clone.querySelector('.title').textContent = 'Hello';

// Correct - append to document
document.body.appendChild(clone);
```

4. **Inefficient Multiple Appends**
```javascript
// Wrong - multiple reflows
items.forEach(item => {
  const clone = template.content.cloneNode(true);
  // ... modify clone
  container.appendChild(clone); // Reflow each time
});

// Correct - use fragment
const fragment = document.createDocumentFragment();
items.forEach(item => {
  const clone = template.content.cloneNode(true);
  // ... modify clone
  fragment.appendChild(clone);
});
container.appendChild(fragment); // Single reflow
```

## Best Practices

1. **Always Deep Clone Templates** - Use `cloneNode(true)`
2. **Cache Template References** - Don't query the DOM repeatedly
3. **Use Document Fragments** - For batch DOM operations
4. **Leverage Template Reusability** - Define once, use many times
5. **Keep Templates Simple** - Complex logic belongs in JavaScript
6. **Use Semantic HTML** - Templates should contain valid HTML
7. **Consider Template Caching** - For frequently used templates
8. **Separate Concerns** - Keep styling, structure, and behavior separate

## When to Use HTML Templates

Use HTML templates when:
- Creating reusable component structures
- Building dynamic UIs with repeated elements
- Optimizing DOM manipulation performance
- Working with web components
- Server-side rendering markup structures

Avoid templates when:
- Building very simple, one-off elements
- Templates are used only once
- Using a framework with its own templating system
- Need complex conditional logic (consider template engines)

## Interview Questions

1. What is the difference between a template and a regular HTML element?
2. Why should you clone template content instead of using it directly?
3. What is a DocumentFragment and why is it useful?
4. How do templates improve performance compared to string concatenation?
5. Can you query elements inside a template before cloning?
6. What happens to scripts inside template elements?
7. How do templates work with Shadow DOM?
8. What are the benefits of using templates over innerHTML?

## Key Takeaways

- Templates are inert - their content is not rendered until activated
- Always clone templates with `cloneNode(true)` for deep cloning
- Document fragments enable efficient batch DOM operations
- Templates are parsed once and can be cloned many times
- Template content is a DocumentFragment, not part of the main document
- Combining templates with custom elements creates powerful, reusable components
- Templates improve performance by avoiding repeated HTML parsing
- Template caching strategies can further optimize component rendering

## Resources

- [MDN: template element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template)
- [HTML Living Standard: template](https://html.spec.whatwg.org/multipage/scripting.html#the-template-element)
- [Using Templates and Slots](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_templates_and_slots)
- [DocumentFragment API](https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment)
- [Web Components Best Practices](https://web.dev/custom-elements-best-practices/)
