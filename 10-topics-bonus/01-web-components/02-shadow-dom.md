# Shadow DOM

## Introduction

Shadow DOM is a web standard that provides encapsulation for JavaScript, CSS, and templating in web components. It allows developers to create isolated DOM trees that are separate from the main document DOM, preventing style conflicts and providing true component isolation.

The Shadow DOM API is one of the three main technologies that make up Web Components, alongside Custom Elements and HTML Templates. It enables developers to build components with their own scoped HTML structure, styling, and behavior without worrying about conflicts with other code on the page.

## Shadow DOM Fundamentals

### What is Shadow DOM?

Shadow DOM creates a scoped subtree inside an element called the shadow host. This subtree is called the shadow tree, and it's separate from the main document tree. The shadow tree has its own scope for CSS and JavaScript, which means:

- Styles defined inside shadow DOM don't leak out
- Styles from outside don't leak in (with exceptions)
- IDs and classes inside shadow DOM don't conflict with the main document

### Creating Shadow DOM

```javascript
// Basic shadow DOM creation
const host = document.querySelector('#my-element');
const shadowRoot = host.attachShadow({ mode: 'open' });

shadowRoot.innerHTML = `
  <style>
    p { color: blue; }
  </style>
  <p>This paragraph is inside shadow DOM</p>
`;
```

### Shadow DOM Modes

Shadow DOM can be attached in two modes:

```javascript
// Open mode - shadow root accessible via element.shadowRoot
const openShadow = element.attachShadow({ mode: 'open' });
console.log(element.shadowRoot); // Returns the shadow root

// Closed mode - shadow root not accessible externally
const closedShadow = element.attachShadow({ mode: 'closed' });
console.log(element.shadowRoot); // Returns null
```

## Complete Custom Element with Shadow DOM

```javascript
class ProductCard extends HTMLElement {
  constructor() {
    super();
    
    // Attach shadow DOM
    const shadow = this.attachShadow({ mode: 'open' });
    
    // Create styles
    const style = document.createElement('style');
    style.textContent = `
      :host {
        display: block;
        font-family: Arial, sans-serif;
      }
      
      .card {
        border: 1px solid #e0e0e0;
        border-radius: 8px;
        overflow: hidden;
        max-width: 300px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        transition: transform 0.2s, box-shadow 0.2s;
      }
      
      .card:hover {
        transform: translateY(-4px);
        box-shadow: 0 4px 12px rgba(0,0,0,0.15);
      }
      
      .image {
        width: 100%;
        height: 200px;
        object-fit: cover;
        background: #f5f5f5;
      }
      
      .content {
        padding: 16px;
      }
      
      .title {
        margin: 0 0 8px 0;
        font-size: 18px;
        font-weight: bold;
        color: #333;
      }
      
      .description {
        margin: 0 0 12px 0;
        color: #666;
        font-size: 14px;
        line-height: 1.5;
      }
      
      .price {
        font-size: 24px;
        font-weight: bold;
        color: #667eea;
        margin: 0 0 12px 0;
      }
      
      .button {
        width: 100%;
        padding: 12px;
        background: #667eea;
        color: white;
        border: none;
        border-radius: 4px;
        font-size: 16px;
        cursor: pointer;
        transition: background 0.2s;
      }
      
      .button:hover {
        background: #5568d3;
      }
      
      .button:active {
        transform: scale(0.98);
      }
    `;
    
    // Create structure
    const card = document.createElement('div');
    card.className = 'card';
    card.innerHTML = `
      <img class="image" alt="Product image">
      <div class="content">
        <h3 class="title"></h3>
        <p class="description"></p>
        <div class="price"></div>
        <button class="button">Add to Cart</button>
      </div>
    `;
    
    // Append to shadow DOM
    shadow.appendChild(style);
    shadow.appendChild(card);
  }
  
  connectedCallback() {
    this.render();
    
    // Add event listener
    const button = this.shadowRoot.querySelector('.button');
    button.addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('add-to-cart', {
        bubbles: true,
        composed: true,
        detail: {
          title: this.getAttribute('title'),
          price: this.getAttribute('price')
        }
      }));
    });
  }
  
  static get observedAttributes() {
    return ['title', 'description', 'price', 'image'];
  }
  
  attributeChangedCallback() {
    this.render();
  }
  
  render() {
    const title = this.getAttribute('title') || 'Product Name';
    const description = this.getAttribute('description') || 'Product description';
    const price = this.getAttribute('price') || '0.00';
    const image = this.getAttribute('image') || '';
    
    const titleEl = this.shadowRoot.querySelector('.title');
    const descEl = this.shadowRoot.querySelector('.description');
    const priceEl = this.shadowRoot.querySelector('.price');
    const imageEl = this.shadowRoot.querySelector('.image');
    
    if (titleEl) titleEl.textContent = title;
    if (descEl) descEl.textContent = description;
    if (priceEl) priceEl.textContent = `$${price}`;
    if (imageEl) imageEl.src = image;
  }
}

customElements.define('product-card', ProductCard);
```

## Styling Shadow DOM

### :host Selector

The `:host` selector targets the shadow host element itself.

```javascript
class StyledElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        /* Style the host element */
        :host {
          display: block;
          padding: 16px;
          background: white;
          border-radius: 8px;
        }
        
        /* Style host when it has a specific class */
        :host(.highlighted) {
          background: yellow;
        }
        
        /* Style host when it has a specific attribute */
        :host([disabled]) {
          opacity: 0.5;
          pointer-events: none;
        }
        
        /* Style host based on state */
        :host(:hover) {
          box-shadow: 0 4px 8px rgba(0,0,0,0.1);
        }
      </style>
      
      <div>Content</div>
    `;
  }
}

customElements.define('styled-element', StyledElement);
```

### :host-context Selector

Style the component based on ancestor elements.

```javascript
class ContextualElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        /* Default styles */
        p {
          color: black;
        }
        
        /* Style when inside a .dark-theme ancestor */
        :host-context(.dark-theme) p {
          color: white;
        }
        
        /* Style when inside an article */
        :host-context(article) p {
          font-family: serif;
        }
      </style>
      
      <p>This text changes based on context</p>
    `;
  }
}

customElements.define('contextual-element', ContextualElement);
```

### ::slotted Selector

Style slotted content from the light DOM.

```javascript
class SlottedElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        /* Style any slotted element */
        ::slotted(*) {
          padding: 8px;
          margin: 4px;
        }
        
        /* Style specific slotted elements */
        ::slotted(p) {
          color: blue;
        }
        
        ::slotted(.highlight) {
          background: yellow;
          font-weight: bold;
        }
        
        /* Can only target direct children, not descendants */
        ::slotted(div) {
          border: 1px solid #ddd;
        }
      </style>
      
      <div>
        <slot></slot>
      </div>
    `;
  }
}

customElements.define('slotted-element', SlottedElement);

// Usage:
// <slotted-element>
//   <p>This paragraph will be blue</p>
//   <div class="highlight">This div will be highlighted</div>
// </slotted-element>
```

### CSS Custom Properties (CSS Variables)

CSS custom properties pierce the shadow boundary, enabling theming.

```javascript
class ThemeableButton extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        :host {
          display: inline-block;
        }
        
        button {
          /* Use CSS custom properties with defaults */
          background: var(--button-bg, #667eea);
          color: var(--button-color, white);
          padding: var(--button-padding, 12px 24px);
          border: var(--button-border, none);
          border-radius: var(--button-radius, 4px);
          font-size: var(--button-font-size, 16px);
          cursor: pointer;
          transition: opacity 0.2s;
        }
        
        button:hover {
          opacity: 0.9;
        }
      </style>
      
      <button><slot>Click me</slot></button>
    `;
  }
}

customElements.define('themeable-button', ThemeableButton);

// Usage with custom properties:
// <style>
//   .custom-theme {
//     --button-bg: #ff6b6b;
//     --button-color: white;
//     --button-padding: 16px 32px;
//     --button-radius: 8px;
//   }
// </style>
// <themeable-button class="custom-theme">Custom Button</themeable-button>
```

## Slots

Slots provide a way to compose components by allowing users to place their own content inside shadow DOM.

### Basic Slots

```javascript
class BasicSlot extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        .container {
          padding: 16px;
          border: 2px solid #667eea;
          border-radius: 8px;
        }
      </style>
      
      <div class="container">
        <slot>Default content if no content provided</slot>
      </div>
    `;
  }
}

customElements.define('basic-slot', BasicSlot);

// Usage:
// <basic-slot>User provided content</basic-slot>
```

### Named Slots

```javascript
class NamedSlots extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        .dialog {
          border: 1px solid #ddd;
          border-radius: 8px;
          max-width: 500px;
          box-shadow: 0 4px 12px rgba(0,0,0,0.15);
        }
        
        .header {
          padding: 16px;
          background: #667eea;
          color: white;
          font-size: 20px;
          font-weight: bold;
        }
        
        .body {
          padding: 16px;
          min-height: 100px;
        }
        
        .footer {
          padding: 12px 16px;
          background: #f5f5f5;
          display: flex;
          justify-content: flex-end;
          gap: 8px;
        }
      </style>
      
      <div class="dialog">
        <div class="header">
          <slot name="header">Default Header</slot>
        </div>
        <div class="body">
          <slot name="body">Default body content</slot>
        </div>
        <div class="footer">
          <slot name="footer">
            <button>Cancel</button>
            <button>OK</button>
          </slot>
        </div>
      </div>
    `;
  }
}

customElements.define('named-slots', NamedSlots);

// Usage:
// <named-slots>
//   <span slot="header">Confirm Action</span>
//   <p slot="body">Are you sure you want to proceed?</p>
//   <div slot="footer">
//     <button>No</button>
//     <button>Yes</button>
//   </div>
// </named-slots>
```

### Multiple Slots with Same Name

```javascript
class MultiSlot extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        .item { margin: 8px 0; }
      </style>
      
      <div>
        <h3>Before</h3>
        <slot name="item"></slot>
        
        <h3>After</h3>
        <slot name="item"></slot>
      </div>
    `;
  }
}

customElements.define('multi-slot', MultiSlot);

// All elements with slot="item" will be distributed to the first slot
```

### Slot Change Events

```javascript
class SlotEvents extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        .count { font-weight: bold; color: #667eea; }
      </style>
      
      <div>
        <p>Items: <span class="count">0</span></p>
        <slot></slot>
      </div>
    `;
    
    // Listen for slot changes
    const slot = shadow.querySelector('slot');
    slot.addEventListener('slotchange', (e) => {
      const nodes = slot.assignedNodes({ flatten: true });
      const elements = slot.assignedElements();
      
      const count = shadow.querySelector('.count');
      count.textContent = elements.length;
      
      console.log('Slot changed:', elements);
    });
  }
}

customElements.define('slot-events', SlotEvents);
```

## Event Handling in Shadow DOM

### Event Retargeting

Events that occur in shadow DOM are retargeted when they cross the shadow boundary.

```javascript
class EventElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        button {
          padding: 12px 24px;
          background: #667eea;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }
      </style>
      
      <button id="inner-button">Click me</button>
    `;
    
    // Event listener inside shadow DOM
    shadow.querySelector('#inner-button').addEventListener('click', (e) => {
      console.log('Inside shadow DOM:');
      console.log('Target:', e.target.id); // "inner-button"
      console.log('Current target:', e.currentTarget.id); // "inner-button"
    });
  }
}

customElements.define('event-element', EventElement);

// Event listener outside shadow DOM
document.addEventListener('click', (e) => {
  if (e.target.tagName === 'EVENT-ELEMENT') {
    console.log('Outside shadow DOM:');
    console.log('Target:', e.target.tagName); // "EVENT-ELEMENT" (retargeted)
    console.log('Composed path:', e.composedPath()); // Full path through shadow boundaries
  }
});
```

### Custom Events with Composed Flag

```javascript
class ComposedEvents extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <button id="btn1">Composed Event</button>
      <button id="btn2">Non-Composed Event</button>
    `;
    
    // Composed event - crosses shadow boundary
    shadow.querySelector('#btn1').addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('composed-event', {
        bubbles: true,
        composed: true, // Crosses shadow boundary
        detail: { message: 'This event crosses shadow boundary' }
      }));
    });
    
    // Non-composed event - stops at shadow boundary
    shadow.querySelector('#btn2').addEventListener('click', () => {
      this.dispatchEvent(new CustomEvent('non-composed-event', {
        bubbles: true,
        composed: false, // Stops at shadow boundary
        detail: { message: 'This event stops at shadow boundary' }
      }));
    });
  }
}

customElements.define('composed-events', ComposedEvents);

// Listen from outside
const element = document.querySelector('composed-events');
element.addEventListener('composed-event', (e) => {
  console.log('Composed event received:', e.detail);
});

element.addEventListener('non-composed-event', (e) => {
  console.log('This will never fire'); // Event doesn't cross boundary
});
```

## Advanced Patterns

### Declarative Shadow DOM (DSD)

Server-side rendering with shadow DOM:

```html
<my-element>
  <template shadowrootmode="open">
    <style>
      p { color: blue; }
    </style>
    <p>This is rendered server-side</p>
  </template>
</my-element>

<script>
class MyElement extends HTMLElement {
  constructor() {
    super();
    
    // Shadow root already attached by browser
    if (!this.shadowRoot) {
      // Fallback for browsers without DSD support
      const shadow = this.attachShadow({ mode: 'open' });
      shadow.innerHTML = `
        <style>p { color: blue; }</style>
        <p>This is rendered client-side</p>
      `;
    }
  }
}

customElements.define('my-element', MyElement);
</script>
```

### Shadow Parts

Expose specific elements for styling from outside:

```javascript
class PartElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    shadow.innerHTML = `
      <style>
        .button {
          padding: 12px 24px;
          background: #667eea;
          color: white;
          border: none;
          border-radius: 4px;
        }
        
        .icon {
          margin-right: 8px;
        }
      </style>
      
      <button class="button" part="button">
        <span class="icon" part="icon">🎨</span>
        <span part="text">Click me</span>
      </button>
    `;
  }
}

customElements.define('part-element', PartElement);

// Style parts from outside using ::part
// <style>
//   part-element::part(button) {
//     background: red;
//   }
//   
//   part-element::part(icon) {
//     font-size: 20px;
//   }
// </style>
```

### Constructable Stylesheets

Efficiently share styles across multiple shadow roots:

```javascript
// Create a stylesheet
const sheet = new CSSStyleSheet();
sheet.replaceSync(`
  .button {
    padding: 12px 24px;
    background: #667eea;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
  
  .button:hover {
    background: #5568d3;
  }
`);

class StyledElement1 extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    // Adopt the shared stylesheet
    shadow.adoptedStyleSheets = [sheet];
    
    shadow.innerHTML = `<button class="button">Button 1</button>`;
  }
}

class StyledElement2 extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    
    // Use the same stylesheet
    shadow.adoptedStyleSheets = [sheet];
    
    shadow.innerHTML = `<button class="button">Button 2</button>`;
  }
}

customElements.define('styled-element-1', StyledElement1);
customElements.define('styled-element-2', StyledElement2);
```

### Focus Delegation

Delegate focus to elements inside shadow DOM:

```javascript
class FocusElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ 
      mode: 'open',
      delegatesFocus: true // Enable focus delegation
    });
    
    shadow.innerHTML = `
      <style>
        input {
          padding: 8px;
          border: 2px solid #ddd;
          border-radius: 4px;
          font-size: 14px;
        }
        
        input:focus {
          outline: none;
          border-color: #667eea;
        }
      </style>
      
      <input type="text" placeholder="Focus is delegated here">
    `;
  }
}

customElements.define('focus-element', FocusElement);

// When you focus the custom element, focus is automatically
// delegated to the first focusable element in shadow DOM
// document.querySelector('focus-element').focus();
```

## Browser Support

| Browser | Version | Shadow DOM | Declarative Shadow DOM |
|---------|---------|------------|------------------------|
| Chrome | 53+ | ✓ | 90+ |
| Firefox | 63+ | ✓ | 123+ |
| Safari | 10+ | ✓ | 16.4+ |
| Edge | 79+ | ✓ | 90+ |
| Opera | 40+ | ✓ | 76+ |

## Common Mistakes

1. **Forgetting to Attach Shadow Root**
```javascript
// Wrong
class MyElement extends HTMLElement {
  constructor() {
    super();
    this.innerHTML = '<p>Not in shadow DOM</p>'; // Light DOM!
  }
}

// Correct
class MyElement extends HTMLElement {
  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = '<p>In shadow DOM</p>';
  }
}
```

2. **Not Using Composed Events**
```javascript
// Wrong - event won't cross shadow boundary
this.dispatchEvent(new CustomEvent('my-event', {
  bubbles: true
  // composed: false is default
}));

// Correct
this.dispatchEvent(new CustomEvent('my-event', {
  bubbles: true,
  composed: true // Event crosses shadow boundary
}));
```

3. **Trying to Style Shadow DOM from Outside**
```javascript
// This won't work - styles don't pierce shadow DOM
// <style>
//   my-element p { color: red; }
// </style>

// Use CSS custom properties or ::part instead
// <style>
//   my-element {
//     --text-color: red;
//   }
// </style>
```

4. **Not Handling Slotted Content**
```javascript
// Wrong - trying to style slotted content from shadow DOM
shadow.innerHTML = `
  <style>
    slot div { color: blue; } /* Doesn't work */
  </style>
  <slot></slot>
`;

// Correct - use ::slotted
shadow.innerHTML = `
  <style>
    ::slotted(div) { color: blue; }
  </style>
  <slot></slot>
`;
```

## Best Practices

1. **Use Open Mode by Default** - Closed mode makes debugging harder
2. **Leverage CSS Custom Properties** - Enable theming across shadow boundaries
3. **Use ::part for Styling Hooks** - Allow controlled customization
4. **Set composed: true for Public Events** - Enable event propagation
5. **Use Constructable Stylesheets** - Share styles efficiently
6. **Implement Focus Management** - Use delegatesFocus when appropriate
7. **Document Slots and Parts** - Help users understand your component API
8. **Provide Sensible Defaults** - Make components work without configuration

## When to Use Shadow DOM

Use Shadow DOM when:
- You need style encapsulation
- Building reusable components
- Preventing style conflicts
- Creating third-party widgets
- Building design systems

Avoid Shadow DOM when:
- SEO is critical (use declarative shadow DOM)
- You need global styles to apply
- Working with legacy code that expects light DOM
- Building simple, non-reusable elements

## Interview Questions

1. What is the difference between shadow DOM and virtual DOM?
2. Explain event retargeting in shadow DOM.
3. What are the two modes of shadow DOM and when would you use each?
4. How do CSS custom properties interact with shadow boundaries?
5. What is the composed path of an event?
6. Explain the difference between light DOM and shadow DOM.
7. How do you expose styling hooks in shadow DOM?
8. What is declarative shadow DOM and why is it useful?

## Key Takeaways

- Shadow DOM provides true encapsulation for web components
- Styles are scoped by default, preventing conflicts
- Slots enable content composition
- Events can be configured to cross shadow boundaries
- CSS custom properties pierce shadow boundaries
- ::part selector provides controlled styling hooks
- Constructable stylesheets enable efficient style sharing
- Declarative shadow DOM enables server-side rendering

## Resources

- [MDN: Using Shadow DOM](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM)
- [Shadow DOM v1 Spec](https://www.w3.org/TR/shadow-dom/)
- [Declarative Shadow DOM](https://web.dev/declarative-shadow-dom/)
- [Shadow Parts Explainer](https://github.com/w3c/webcomponents/blob/gh-pages/proposals/css-shadow-parts-1.md)
- [Constructable Stylesheets](https://developers.google.com/web/updates/2019/02/constructable-stylesheets)
