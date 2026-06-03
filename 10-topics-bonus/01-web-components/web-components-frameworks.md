# Web Components Frameworks

## Introduction

While native Web Components provide a powerful foundation for building reusable components, frameworks and libraries have emerged to make the development process more ergonomic and productive. These tools provide features like reactive data binding, enhanced templating, TypeScript support, and developer-friendly APIs while still producing standards-compliant web components.

The two most popular frameworks for building web components are Lit (developed by Google) and Stencil (developed by the Ionic team). Both frameworks compile to native web components, ensuring broad compatibility and framework-agnostic usage, while providing modern development experiences.

## Lit Framework

Lit is a lightweight library for building fast, reactive web components. It builds directly on web standards and adds just what's needed: reactivity, declarative templates, and reduced boilerplate.

### Getting Started with Lit

```bash
npm install lit
```

### Basic Lit Component

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

@customElement('my-counter')
export class MyCounter extends LitElement {
  static styles = css`
    :host {
      display: block;
      font-family: Arial, sans-serif;
      padding: 16px;
    }
    
    .counter {
      text-align: center;
      border: 2px solid #667eea;
      border-radius: 8px;
      padding: 24px;
      max-width: 300px;
    }
    
    .count {
      font-size: 48px;
      font-weight: bold;
      color: #667eea;
      margin: 20px 0;
    }
    
    .buttons {
      display: flex;
      gap: 8px;
      justify-content: center;
    }
    
    button {
      padding: 10px 20px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 16px;
      transition: opacity 0.2s;
    }
    
    button:hover {
      opacity: 0.8;
    }
    
    .decrement { background: #ff6b6b; color: white; }
    .increment { background: #51cf66; color: white; }
    .reset { background: #748ffc; color: white; }
  `;
  
  @property({ type: Number })
  count = 0;
  
  @property({ type: Number })
  step = 1;
  
  render() {
    return html`
      <div class="counter">
        <div class="count">${this.count}</div>
        <div class="buttons">
          <button class="decrement" @click=${this._decrement}>-${this.step}</button>
          <button class="reset" @click=${this._reset}>Reset</button>
          <button class="increment" @click=${this._increment}>+${this.step}</button>
        </div>
      </div>
    `;
  }
  
  private _increment() {
    this.count += this.step;
    this._dispatchChange();
  }
  
  private _decrement() {
    this.count -= this.step;
    this._dispatchChange();
  }
  
  private _reset() {
    this.count = 0;
    this._dispatchChange();
  }
  
  private _dispatchChange() {
    this.dispatchEvent(new CustomEvent('count-changed', {
      detail: { count: this.count },
      bubbles: true,
      composed: true
    }));
  }
}

// Usage:
// <my-counter count="0" step="5"></my-counter>
```

### Lit Property Decorators

```typescript
import { LitElement, html } from 'lit';
import { customElement, property, state, query, queryAll } from 'lit/decorators.js';

@customElement('property-demo')
export class PropertyDemo extends LitElement {
  // Public property - reflected to attribute
  @property({ type: String })
  name = '';
  
  // Public property with custom attribute name
  @property({ type: Number, attribute: 'max-value' })
  maxValue = 100;
  
  // Public property that doesn't reflect to attribute
  @property({ type: Object, attribute: false })
  data = {};
  
  // Internal reactive state
  @state()
  private _internalCount = 0;
  
  // Query single element
  @query('#my-input')
  private _input!: HTMLInputElement;
  
  // Query all matching elements
  @queryAll('.item')
  private _items!: NodeListOf<HTMLElement>;
  
  render() {
    return html`
      <div>
        <input id="my-input" type="text" .value=${this.name}>
        <p>Count: ${this._internalCount}</p>
        <button @click=${this._increment}>Increment Internal</button>
        
        <div class="item">Item 1</div>
        <div class="item">Item 2</div>
        <div class="item">Item 3</div>
      </div>
    `;
  }
  
  private _increment() {
    this._internalCount++;
  }
  
  firstUpdated() {
    // Access queried elements after first render
    console.log('Input element:', this._input);
    console.log('Number of items:', this._items.length);
  }
}
```

### Lit Templates and Directives

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { repeat } from 'lit/directives/repeat.js';
import { classMap } from 'lit/directives/class-map.js';
import { styleMap } from 'lit/directives/style-map.js';
import { when } from 'lit/directives/when.js';
import { choose } from 'lit/directives/choose.js';

@customElement('template-demo')
export class TemplateDemo extends LitElement {
  static styles = css`
    .item {
      padding: 12px;
      margin: 8px 0;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
    
    .completed {
      opacity: 0.6;
      text-decoration: line-through;
    }
    
    .high-priority {
      border-left: 4px solid #ff6b6b;
    }
    
    .loading { color: #999; }
    .error { color: #ff6b6b; }
    .success { color: #51cf66; }
  `;
  
  @state()
  private items = [
    { id: 1, text: 'Learn Lit', completed: false, priority: 'high' },
    { id: 2, text: 'Build components', completed: false, priority: 'low' },
    { id: 3, text: 'Ship to production', completed: false, priority: 'medium' }
  ];
  
  @state()
  private status: 'loading' | 'error' | 'success' = 'success';
  
  render() {
    return html`
      <div>
        <!-- repeat directive for efficient list rendering -->
        ${repeat(
          this.items,
          (item) => item.id, // Key function
          (item) => html`
            <div 
              class=${classMap({
                item: true,
                completed: item.completed,
                'high-priority': item.priority === 'high'
              })}
              style=${styleMap({
                backgroundColor: item.completed ? '#f5f5f5' : 'white'
              })}
              @click=${() => this._toggleItem(item.id)}
            >
              ${item.text}
            </div>
          `
        )}
        
        <!-- when directive for conditional rendering -->
        ${when(
          this.items.length === 0,
          () => html`<p>No items yet</p>`,
          () => html`<p>${this.items.length} items</p>`
        )}
        
        <!-- choose directive for multiple conditions -->
        ${choose(this.status, [
          ['loading', () => html`<p class="loading">Loading...</p>`],
          ['error', () => html`<p class="error">Error loading data</p>`],
          ['success', () => html`<p class="success">Data loaded successfully</p>`]
        ])}
      </div>
    `;
  }
  
  private _toggleItem(id: number) {
    this.items = this.items.map(item =>
      item.id === id ? { ...item, completed: !item.completed } : item
    );
  }
}
```

### Lit Lifecycle Methods

```typescript
import { LitElement, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('lifecycle-demo')
export class LifecycleDemo extends LitElement {
  @property({ type: String })
  message = '';
  
  // Called before first update
  constructor() {
    super();
    console.log('constructor');
  }
  
  // Called when element is connected to DOM
  connectedCallback() {
    super.connectedCallback();
    console.log('connectedCallback');
  }
  
  // Called when element is disconnected from DOM
  disconnectedCallback() {
    super.disconnectedCallback();
    console.log('disconnectedCallback');
  }
  
  // Called before update, can prevent rendering
  shouldUpdate(changedProperties: Map<string, any>) {
    console.log('shouldUpdate', changedProperties);
    return true; // Return false to prevent update
  }
  
  // Called before render
  willUpdate(changedProperties: Map<string, any>) {
    console.log('willUpdate', changedProperties);
    // Good place to compute derived state
  }
  
  // Called after first render only
  firstUpdated(changedProperties: Map<string, any>) {
    console.log('firstUpdated', changedProperties);
    // Good place to focus inputs, start animations, etc.
  }
  
  // Called after every render
  updated(changedProperties: Map<string, any>) {
    console.log('updated', changedProperties);
  }
  
  render() {
    return html`<p>${this.message}</p>`;
  }
}
```

### Lit Component Communication

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

// Child component
@customElement('todo-item')
export class TodoItem extends LitElement {
  static styles = css`
    .item {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 12px;
      border: 1px solid #e0e0e0;
      border-radius: 4px;
      margin-bottom: 8px;
    }
    
    .text {
      flex: 1;
    }
    
    .completed {
      text-decoration: line-through;
      opacity: 0.6;
    }
  `;
  
  @property({ type: String })
  text = '';
  
  @property({ type: Boolean })
  completed = false;
  
  render() {
    return html`
      <div class="item">
        <input 
          type="checkbox" 
          .checked=${this.completed}
          @change=${this._handleToggle}
        >
        <span class=${this.completed ? 'text completed' : 'text'}>
          ${this.text}
        </span>
        <button @click=${this._handleDelete}>Delete</button>
      </div>
    `;
  }
  
  private _handleToggle() {
    this.dispatchEvent(new CustomEvent('toggle', {
      detail: { completed: !this.completed },
      bubbles: true,
      composed: true
    }));
  }
  
  private _handleDelete() {
    this.dispatchEvent(new CustomEvent('delete', {
      bubbles: true,
      composed: true
    }));
  }
}

// Parent component
@customElement('todo-list')
export class TodoList extends LitElement {
  static styles = css`
    :host {
      display: block;
      max-width: 500px;
      margin: 0 auto;
      font-family: Arial, sans-serif;
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
  `;
  
  @state()
  private todos = [
    { id: 1, text: 'Learn Lit', completed: false },
    { id: 2, text: 'Build app', completed: false }
  ];
  
  @state()
  private nextId = 3;
  
  @state()
  private inputValue = '';
  
  render() {
    return html`
      <div>
        <div class="input-container">
          <input 
            type="text" 
            .value=${this.inputValue}
            @input=${this._handleInput}
            @keypress=${this._handleKeyPress}
            placeholder="Enter a todo..."
          >
          <button @click=${this._addTodo}>Add</button>
        </div>
        
        ${this.todos.map((todo, index) => html`
          <todo-item
            .text=${todo.text}
            .completed=${todo.completed}
            @toggle=${(e: CustomEvent) => this._toggleTodo(index, e)}
            @delete=${() => this._deleteTodo(index)}
          ></todo-item>
        `)}
      </div>
    `;
  }
  
  private _handleInput(e: Event) {
    this.inputValue = (e.target as HTMLInputElement).value;
  }
  
  private _handleKeyPress(e: KeyboardEvent) {
    if (e.key === 'Enter') {
      this._addTodo();
    }
  }
  
  private _addTodo() {
    if (!this.inputValue.trim()) return;
    
    this.todos = [
      ...this.todos,
      { id: this.nextId++, text: this.inputValue, completed: false }
    ];
    this.inputValue = '';
  }
  
  private _toggleTodo(index: number, e: CustomEvent) {
    this.todos = this.todos.map((todo, i) =>
      i === index ? { ...todo, completed: e.detail.completed } : todo
    );
  }
  
  private _deleteTodo(index: number) {
    this.todos = this.todos.filter((_, i) => i !== index);
  }
}
```

## Stencil Framework

Stencil is a compiler that generates web components and builds optimized, standards-compliant web components that work in every major framework and browser.

### Getting Started with Stencil

```bash
npm init stencil
# Choose "component" starter
```

### Basic Stencil Component

```typescript
import { Component, Prop, State, Event, EventEmitter, h } from '@stencil/core';

@Component({
  tag: 'my-counter',
  styleUrl: 'my-counter.css',
  shadow: true
})
export class MyCounter {
  @Prop() count: number = 0;
  @Prop() step: number = 1;
  
  @State() private internalCount: number = 0;
  
  @Event() countChanged: EventEmitter<number>;
  
  componentWillLoad() {
    this.internalCount = this.count;
  }
  
  private increment() {
    this.internalCount += this.step;
    this.countChanged.emit(this.internalCount);
  }
  
  private decrement() {
    this.internalCount -= this.step;
    this.countChanged.emit(this.internalCount);
  }
  
  private reset() {
    this.internalCount = 0;
    this.countChanged.emit(this.internalCount);
  }
  
  render() {
    return (
      <div class="counter">
        <div class="count">{this.internalCount}</div>
        <div class="buttons">
          <button class="decrement" onClick={() => this.decrement()}>
            -{this.step}
          </button>
          <button class="reset" onClick={() => this.reset()}>
            Reset
          </button>
          <button class="increment" onClick={() => this.increment()}>
            +{this.step}
          </button>
        </div>
      </div>
    );
  }
}
```

### Stencil Decorators

```typescript
import { 
  Component, 
  Prop, 
  State, 
  Watch, 
  Event, 
  EventEmitter,
  Method,
  Element,
  h 
} from '@stencil/core';

@Component({
  tag: 'decorator-demo',
  styleUrl: 'decorator-demo.css',
  shadow: true
})
export class DecoratorDemo {
  // Reference to host element
  @Element() el: HTMLElement;
  
  // Public property
  @Prop() name: string = '';
  
  // Mutable property (can be changed internally)
  @Prop({ mutable: true }) count: number = 0;
  
  // Property with reflection
  @Prop({ reflect: true }) active: boolean = false;
  
  // Internal state
  @State() private loading: boolean = false;
  
  // Custom event
  @Event() nameChanged: EventEmitter<string>;
  
  // Watch for property changes
  @Watch('name')
  watchName(newValue: string, oldValue: string) {
    console.log(`Name changed from ${oldValue} to ${newValue}`);
    this.nameChanged.emit(newValue);
  }
  
  @Watch('count')
  validateCount(newValue: number) {
    if (newValue < 0) {
      this.count = 0;
    } else if (newValue > 100) {
      this.count = 100;
    }
  }
  
  // Public method
  @Method()
  async reset() {
    this.count = 0;
    this.name = '';
    this.active = false;
  }
  
  // Public method
  @Method()
  async getData() {
    return {
      name: this.name,
      count: this.count,
      active: this.active
    };
  }
  
  render() {
    return (
      <div>
        <p>Name: {this.name}</p>
        <p>Count: {this.count}</p>
        <p>Active: {this.active ? 'Yes' : 'No'}</p>
        {this.loading && <p>Loading...</p>}
      </div>
    );
  }
}

// Usage:
// const demo = document.querySelector('decorator-demo');
// await demo.reset();
// const data = await demo.getData();
```

### Stencil Lifecycle Methods

```typescript
import { Component, h } from '@stencil/core';

@Component({
  tag: 'lifecycle-demo',
  shadow: true
})
export class LifecycleDemo {
  // Called once before first render
  componentWillLoad() {
    console.log('componentWillLoad - fetch data here');
  }
  
  // Called once after first render
  componentDidLoad() {
    console.log('componentDidLoad - DOM is ready');
  }
  
  // Called before re-render
  componentWillRender() {
    console.log('componentWillRender');
  }
  
  // Called after re-render
  componentDidRender() {
    console.log('componentDidRender');
  }
  
  // Called before updates
  componentWillUpdate() {
    console.log('componentWillUpdate');
  }
  
  // Called after updates
  componentDidUpdate() {
    console.log('componentDidUpdate');
  }
  
  // Called when disconnected
  disconnectedCallback() {
    console.log('disconnectedCallback - cleanup here');
  }
  
  render() {
    return <div>Lifecycle Demo</div>;
  }
}
```

### Stencil Data Flow

```typescript
// Parent component
import { Component, State, h } from '@stencil/core';

@Component({
  tag: 'parent-component',
  shadow: true
})
export class ParentComponent {
  @State() private users = [
    { id: 1, name: 'John', email: 'john@example.com' },
    { id: 2, name: 'Jane', email: 'jane@example.com' }
  ];
  
  private handleUserClick(userId: number) {
    console.log('User clicked:', userId);
  }
  
  private handleUserDelete(userId: number) {
    this.users = this.users.filter(u => u.id !== userId);
  }
  
  render() {
    return (
      <div>
        <h2>Users</h2>
        {this.users.map(user => (
          <user-card
            key={user.id}
            user={user}
            onUserClick={(e) => this.handleUserClick(e.detail)}
            onUserDelete={(e) => this.handleUserDelete(e.detail)}
          />
        ))}
      </div>
    );
  }
}

// Child component
import { Component, Prop, Event, EventEmitter, h } from '@stencil/core';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  tag: 'user-card',
  styleUrl: 'user-card.css',
  shadow: true
})
export class UserCard {
  @Prop() user: User;
  
  @Event() userClick: EventEmitter<number>;
  @Event() userDelete: EventEmitter<number>;
  
  private handleClick() {
    this.userClick.emit(this.user.id);
  }
  
  private handleDelete(e: Event) {
    e.stopPropagation();
    this.userDelete.emit(this.user.id);
  }
  
  render() {
    return (
      <div class="card" onClick={() => this.handleClick()}>
        <h3>{this.user.name}</h3>
        <p>{this.user.email}</p>
        <button onClick={(e) => this.handleDelete(e)}>Delete</button>
      </div>
    );
  }
}
```

## Building Component Libraries

### Library Structure

```
my-components/
├── src/
│   └── components/
│       ├── button/
│       │   ├── button.tsx
│       │   ├── button.css
│       │   ├── button.spec.ts
│       │   └── readme.md
│       ├── card/
│       │   ├── card.tsx
│       │   ├── card.css
│       │   ├── card.spec.ts
│       │   └── readme.md
│       └── index.ts
├── stencil.config.ts
├── package.json
└── README.md
```

### Component Library Configuration (Stencil)

```typescript
// stencil.config.ts
import { Config } from '@stencil/core';

export const config: Config = {
  namespace: 'my-components',
  outputTargets: [
    {
      type: 'dist',
      esmLoaderPath: '../loader'
    },
    {
      type: 'dist-custom-elements'
    },
    {
      type: 'docs-readme'
    },
    {
      type: 'www',
      serviceWorker: null
    }
  ]
};
```

### Design System Button Component

```typescript
import { Component, Prop, h, Host } from '@stencil/core';

export type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost';
export type ButtonSize = 'small' | 'medium' | 'large';

@Component({
  tag: 'ds-button',
  styleUrl: 'button.css',
  shadow: true
})
export class Button {
  @Prop() variant: ButtonVariant = 'primary';
  @Prop() size: ButtonSize = 'medium';
  @Prop() disabled: boolean = false;
  @Prop() loading: boolean = false;
  @Prop() fullWidth: boolean = false;
  @Prop() type: 'button' | 'submit' | 'reset' = 'button';
  
  render() {
    const classes = {
      'button': true,
      [`button--${this.variant}`]: true,
      [`button--${this.size}`]: true,
      'button--disabled': this.disabled,
      'button--loading': this.loading,
      'button--full-width': this.fullWidth
    };
    
    return (
      <Host>
        <button
          class={classes}
          type={this.type}
          disabled={this.disabled || this.loading}
        >
          {this.loading && <span class="spinner"></span>}
          <slot></slot>
        </button>
      </Host>
    );
  }
}

// Usage:
// <ds-button variant="primary" size="large">Click me</ds-button>
// <ds-button variant="danger" loading>Loading...</ds-button>
```

### Theming System

```typescript
// theme-provider.ts
import { Component, Prop, h, Host } from '@stencil/core';

@Component({
  tag: 'theme-provider',
  styleUrl: 'theme-provider.css',
  shadow: true
})
export class ThemeProvider {
  @Prop() theme: 'light' | 'dark' = 'light';
  
  render() {
    return (
      <Host class={`theme-${this.theme}`}>
        <slot></slot>
      </Host>
    );
  }
}

// theme-provider.css
:host {
  display: block;
  
  /* Light theme (default) */
  --color-primary: #667eea;
  --color-secondary: #764ba2;
  --color-background: #ffffff;
  --color-surface: #f5f5f5;
  --color-text: #333333;
  --color-text-secondary: #666666;
}

:host(.theme-dark) {
  --color-primary: #8b9bff;
  --color-secondary: #a67bc4;
  --color-background: #1a1a1a;
  --color-surface: #2a2a2a;
  --color-text: #ffffff;
  --color-text-secondary: #cccccc;
}

// Usage:
// <theme-provider theme="dark">
//   <ds-button>Button</ds-button>
//   <ds-card>Card content</ds-card>
// </theme-provider>
```

## Browser Support

| Framework | Browser Support | Bundle Size | TypeScript | Testing |
|-----------|----------------|-------------|------------|---------|
| Lit | Modern browsers, IE11 with polyfills | ~5KB | ✓ | Web Test Runner |
| Stencil | Modern browsers | 0KB (lazy loaded) | ✓ | Jest |

## Common Mistakes

1. **Not Using Shadow DOM**
```typescript
// Less encapsulated
@Component({
  tag: 'my-component',
  shadow: false  // Styles can leak
})

// Better
@Component({
  tag: 'my-component',
  shadow: true  // Fully encapsulated
})
```

2. **Mutating Props Directly (Stencil)**
```typescript
// Wrong
@Prop() items: string[];

addItem(item: string) {
  this.items.push(item);  // Doesn't trigger re-render
}

// Correct
@Prop() items: string[];

addItem(item: string) {
  this.items = [...this.items, item];  // Creates new array
}
```

3. **Not Using Keys in Lists**
```typescript
// Wrong
render() {
  return html`
    ${this.items.map(item => html`<div>${item}</div>`)}
  `;
}

// Correct
render() {
  return html`
    ${repeat(this.items, item => item.id, item => html`
      <div>${item.text}</div>
    `)}
  `;
}
```

## Best Practices

1. **Use TypeScript** - Better type safety and developer experience
2. **Implement Proper Event Handling** - Use composed events
3. **Follow Naming Conventions** - Use prefixes for component namespacing
4. **Optimize Bundle Size** - Lazy load components when possible
5. **Write Tests** - Unit test components thoroughly
6. **Document Components** - Provide clear API documentation
7. **Support SSR** - Consider server-side rendering needs
8. **Version Carefully** - Follow semantic versioning

## When to Use Frameworks

Use Lit/Stencil when:
- Building component libraries
- Need TypeScript support
- Want reactive updates
- Building design systems
- Need testing infrastructure

Use vanilla Web Components when:
- Building simple components
- Want zero dependencies
- Need maximum control
- Learning Web Components

## Interview Questions

1. What are the main differences between Lit and Stencil?
2. How does Lit's reactive update system work?
3. What is the purpose of Stencil's compiler?
4. How do you handle component communication in Lit?
5. What are the benefits of using a framework over vanilla Web Components?
6. How do you test web components built with these frameworks?
7. What is the bundle size impact of using these frameworks?
8. How do you implement theming in a component library?

## Key Takeaways

- Lit and Stencil make Web Components development more productive
- Both compile to standards-compliant web components
- Frameworks provide reactivity, templating, and TypeScript support
- Component libraries need proper structure and documentation
- Testing and versioning are crucial for libraries
- Both frameworks have minimal runtime overhead
- Choose based on team needs and project requirements

## Resources

- [Lit Official Site](https://lit.dev/)
- [Stencil Official Site](https://stenciljs.com/)
- [Lit Playground](https://lit.dev/playground/)
- [Stencil Component Starter](https://github.com/ionic-team/stencil-component-starter)
- [Open WC](https://open-wc.org/)
- [Web Components Examples](https://www.webcomponents.org/examples)
