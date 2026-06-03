# Custom Elements

## Introduction

Custom Elements are a fundamental part of the Web Components standard that allows developers to create their own HTML elements with custom behavior and functionality. They provide a way to extend HTML's vocabulary and create reusable, encapsulated components that work seamlessly with the browser's native APIs.

The Custom Elements API enables you to define new HTML tags, specify their behavior, and use them just like built-in elements. This is achieved through JavaScript classes that extend `HTMLElement` or its descendants, giving you full control over element lifecycle, attributes, properties, and methods.

## Custom Elements Fundamentals

### Types of Custom Elements

There are two types of custom elements:

1. **Autonomous Custom Elements**: Standalone elements that extend `HTMLElement`
2. **Customized Built-in Elements**: Elements that extend existing HTML elements like `HTMLButtonElement`

### Naming Requirements

Custom element names must:
- Contain a hyphen (-) to distinguish them from standard HTML elements
- Start with a lowercase ASCII letter
- Not contain uppercase letters
- Not be one of the reserved names

```javascript
// Valid names
'my-element'
'custom-button'
'app-header'
'x-component'

// Invalid names
'myElement'     // No hyphen
'My-Element'    // Uppercase
'button'        // No hyphen
```

## Creating Autonomous Custom Elements

### Basic Custom Element

```javascript
// Define a custom element class
class MyElement extends HTMLElement {
  constructor() {
    // Always call super() first
    super();
    
    // Element initialization
    this.textContent = 'Hello from Custom Element!';
  }
}

// Register the custom element
customElements.define('my-element', MyElement);

// Usage in HTML
// <my-element></my-element>
```

### Custom Element with Shadow DOM

```javascript
class FancyButton extends HTMLElement {
  constructor() {
    super();
    
    // Attach shadow DOM
    const shadow = this.attachShadow({ mode: 'open' });
    
    // Create button
    const button = document.createElement('button');
    button.textContent = this.getAttribute('label') || 'Click me';
    
    // Add styles
    const style = document.createElement('style');
    style.textContent = `
      button {
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        color: white;
        border: none;
        padding: 12px 24px;
        border-radius: 8px;
        font-size: 16px;
        cursor: pointer;
        transition: transform 0.2s, box-shadow 0.2s;
      }
      
      button:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
      }
      
      button:active {
        transform: translateY(0);
      }
    `;
    
    // Append to shadow DOM
    shadow.appendChild(style);
    shadow.appendChild(button);
  }
}

customElements.define('fancy-button', FancyButton);
```

### Custom Element with Template

```javascript
class UserCard extends HTMLElement {
  constructor() {
    super();
    
    const shadow = this.attachShadow({ mode: 'open' });
    
    // Create template
    const template = document.createElement('template');
    template.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: Arial, sans-serif;
        }
        
        .card {
          border: 1px solid #ddd;
          border-radius: 8px;
          padding: 16px;
          max-width: 300px;
          box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        
        .avatar {
          width: 64px;
          height: 64px;
          border-radius: 50%;
          background: #667eea;
          display: flex;
          align-items: center;
          justify-content: center;
          color: white;
          font-size: 24px;
          font-weight: bold;
          margin-bottom: 12px;
        }
        
        .name {
          font-size: 18px;
          font-weight: bold;
          margin: 0 0 4px 0;
        }
        
        .email {
          color: #666;
          margin: 0;
        }
      </style>
      
      <div class="card">
        <div class="avatar"></div>
        <h3 class="name"></h3>
        <p class="email"></p>
      </div>
    `;
    
    // Clone template and append
    shadow.appendChild(template.content.cloneNode(true));
  }
}

customElements.define('user-card', UserCard);
```

## Lifecycle Callbacks

Custom elements have four lifecycle callbacks that allow you to respond to specific events in the element's lifecycle.

### connectedCallback

Called when the element is inserted into the DOM.

```javascript
class LifecycleElement extends HTMLElement {
  connectedCallback() {
    console.log('Element added to page');
    this.render();
    this.attachEventListeners();
  }
  
  render() {
    this.innerHTML = `
      <div>
        <p>Element is connected!</p>
        <p>Connected at: ${new Date().toLocaleTimeString()}</p>
      </div>
    `;
  }
  
  attachEventListeners() {
    // Safe to add event listeners here
    this.addEventListener('click', () => {
      console.log('Element clicked');
    });
  }
}

customElements.define('lifecycle-element', LifecycleElement);
```

### disconnectedCallback

Called when the element is removed from the DOM.

```javascript
class CleanupElement extends HTMLElement {
  constructor() {
    super();
    this.intervalId = null;
  }
  
  connectedCallback() {
    // Start interval
    this.intervalId = setInterval(() => {
      console.log('Tick...');
    }, 1000);
  }
  
  disconnectedCallback() {
    console.log('Element removed from page');
    
    // Clean up resources
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}

customElements.define('cleanup-element', CleanupElement);
```

### adoptedCallback

Called when the element is moved to a new document.

```javascript
class AdoptedElement extends HTMLElement {
  adoptedCallback() {
    console.log('Element moved to new document');
    // Update any document-specific references
  }
}

customElements.define('adopted-element', AdoptedElement);
```

### attributeChangedCallback

Called when observed attributes are added, removed, or changed.

```javascript
class AttributeElement extends HTMLElement {
  // Specify which attributes to observe
  static get observedAttributes() {
    return ['name', 'age', 'active'];
  }
  
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    console.log(`Attribute ${name} changed from ${oldValue} to ${newValue}`);
    
    if (oldValue !== newValue) {
      this.render();
    }
  }
  
  render() {
    const name = this.getAttribute('name') || 'Unknown';
    const age = this.getAttribute('age') || 'N/A';
    const active = this.hasAttribute('active');
    
    this.shadowRoot.innerHTML = `
      <style>
        .status {
          padding: 4px 8px;
          border-radius: 4px;
          display: inline-block;
          font-size: 12px;
        }
        .active { background: #d4edda; color: #155724; }
        .inactive { background: #f8d7da; color: #721c24; }
      </style>
      
      <div>
        <h3>${name}</h3>
        <p>Age: ${age}</p>
        <span class="status ${active ? 'active' : 'inactive'}">
          ${active ? 'Active' : 'Inactive'}
        </span>
      </div>
    `;
  }
  
  connectedCallback() {
    this.render();
  }
}

customElements.define('attribute-element', AttributeElement);

// Usage:
// <attribute-element name="John" age="30" active></attribute-element>

// Change attributes dynamically:
// element.setAttribute('name', 'Jane');
// element.removeAttribute('active');
```

## Properties and Attributes

### Reflecting Properties to Attributes

```javascript
class ReflectingElement extends HTMLElement {
  constructor() {
    super();
    this._value = '';
    this._disabled = false;
  }
  
  static get observedAttributes() {
    return ['value', 'disabled'];
  }
  
  // Getter and setter for 'value' property
  get value() {
    return this._value;
  }
  
  set value(val) {
    this._value = val;
    // Reflect property to attribute
    this.setAttribute('value', val);
  }
  
  // Getter and setter for 'disabled' property
  get disabled() {
    return this._disabled;
  }
  
  set disabled(val) {
    this._disabled = Boolean(val);
    // Reflect boolean property to attribute
    if (this._disabled) {
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'value') {
      this._value = newValue;
    } else if (name === 'disabled') {
      this._disabled = newValue !== null;
    }
  }
}

customElements.define('reflecting-element', ReflectingElement);

// Usage:
// element.value = 'hello';  // Sets property and attribute
// element.disabled = true;  // Sets property and attribute
```

### Complex Properties

```javascript
class DataElement extends HTMLElement {
  constructor() {
    super();
    this._data = null;
    this.attachShadow({ mode: 'open' });
  }
  
  // Property for complex data (not reflected to attribute)
  get data() {
    return this._data;
  }
  
  set data(val) {
    this._data = val;
    this.render();
  }
  
  render() {
    if (!this._data) {
      this.shadowRoot.innerHTML = '<p>No data</p>';
      return;
    }
    
    this.shadowRoot.innerHTML = `
      <style>
        ul { list-style: none; padding: 0; }
        li { padding: 8px; border-bottom: 1px solid #eee; }
      </style>
      
      <ul>
        ${this._data.map(item => `
          <li>${item.name}: ${item.value}</li>
        `).join('')}
      </ul>
    `;
  }
  
  connectedCallback() {
    this.render();
  }
}

customElements.define('data-element', DataElement);

// Usage:
// const element = document.querySelector('data-element');
// element.data = [
//   { name: 'Item 1', value: 100 },
//   { name: 'Item 2', value: 200 }
// ];
```

## Customized Built-in Elements

Extend existing HTML elements to enhance their functionality.

```javascript
class FancyButton extends HTMLButtonElement {
  constructor() {
    super();
    
    // Add custom styling
    this.style.cssText = `
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
      border: none;
      padding: 12px 24px;
      border-radius: 8px;
      font-size: 16px;
      cursor: pointer;
      transition: transform 0.2s;
    `;
    
    // Add hover effect
    this.addEventListener('mouseenter', () => {
      this.style.transform = 'translateY(-2px)';
    });
    
    this.addEventListener('mouseleave', () => {
      this.style.transform = 'translateY(0)';
    });
  }
  
  connectedCallback() {
    // Add ripple effect on click
    this.addEventListener('click', this.createRipple);
  }
  
  createRipple(event) {
    const ripple = document.createElement('span');
    const rect = this.getBoundingClientRect();
    const size = Math.max(rect.width, rect.height);
    const x = event.clientX - rect.left - size / 2;
    const y = event.clientY - rect.top - size / 2;
    
    ripple.style.cssText = `
      position: absolute;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.6);
      width: ${size}px;
      height: ${size}px;
      left: ${x}px;
      top: ${y}px;
      pointer-events: none;
      animation: ripple 0.6s ease-out;
    `;
    
    this.style.position = 'relative';
    this.style.overflow = 'hidden';
    this.appendChild(ripple);
    
    setTimeout(() => ripple.remove(), 600);
  }
}

// Register customized built-in element
customElements.define('fancy-button', FancyButton, { extends: 'button' });

// Usage:
// <button is="fancy-button">Click me</button>
```

### Extending Other Built-in Elements

```javascript
// Extending input
class EmailInput extends HTMLInputElement {
  constructor() {
    super();
    this.type = 'email';
    this.pattern = '[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}$';
  }
  
  connectedCallback() {
    this.addEventListener('blur', this.validate);
  }
  
  validate() {
    if (this.validity.valid) {
      this.style.borderColor = 'green';
    } else {
      this.style.borderColor = 'red';
    }
  }
}

customElements.define('email-input', EmailInput, { extends: 'input' });

// Usage:
// <input is="email-input" />

// Extending image
class LazyImage extends HTMLImageElement {
  constructor() {
    super();
  }
  
  connectedCallback() {
    this.observer = new IntersectionObserver(entries => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          this.src = this.dataset.src;
          this.observer.unobserve(this);
        }
      });
    });
    
    this.observer.observe(this);
  }
  
  disconnectedCallback() {
    if (this.observer) {
      this.observer.disconnect();
    }
  }
}

customElements.define('lazy-image', LazyImage, { extends: 'img' });

// Usage:
// <img is="lazy-image" data-src="image.jpg" />
```

## Advanced Patterns

### Custom Element with State Management

```javascript
class CounterElement extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this._count = 0;
  }
  
  static get observedAttributes() {
    return ['count'];
  }
  
  get count() {
    return this._count;
  }
  
  set count(val) {
    const newVal = parseInt(val, 10);
    if (!isNaN(newVal) && this._count !== newVal) {
      this._count = newVal;
      this.setAttribute('count', newVal);
      this.dispatchEvent(new CustomEvent('count-changed', {
        detail: { count: this._count },
        bubbles: true,
        composed: true
      }));
      this.render();
    }
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'count' && oldValue !== newValue) {
      this._count = parseInt(newValue, 10) || 0;
      this.render();
    }
  }
  
  connectedCallback() {
    this.render();
    this.setupEventListeners();
  }
  
  setupEventListeners() {
    const decrementBtn = this.shadowRoot.querySelector('#decrement');
    const incrementBtn = this.shadowRoot.querySelector('#increment');
    const resetBtn = this.shadowRoot.querySelector('#reset');
    
    decrementBtn.addEventListener('click', () => this.count--);
    incrementBtn.addEventListener('click', () => this.count++);
    resetBtn.addEventListener('click', () => this.count = 0);
  }
  
  render() {
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: Arial, sans-serif;
        }
        
        .counter {
          text-align: center;
          padding: 20px;
          border: 2px solid #667eea;
          border-radius: 8px;
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
          transition: background-color 0.2s;
        }
        
        #decrement { background: #f8d7da; }
        #increment { background: #d4edda; }
        #reset { background: #d1ecf1; }
        
        button:hover {
          opacity: 0.8;
        }
      </style>
      
      <div class="counter">
        <div class="count">${this._count}</div>
        <div class="buttons">
          <button id="decrement">-</button>
          <button id="reset">Reset</button>
          <button id="increment">+</button>
        </div>
      </div>
    `;
    
    this.setupEventListeners();
  }
}

customElements.define('counter-element', CounterElement);
```

### Custom Element with Slots

```javascript
class CardElement extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
        }
        
        .card {
          border: 1px solid #ddd;
          border-radius: 8px;
          overflow: hidden;
          box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        
        .header {
          background: #667eea;
          color: white;
          padding: 16px;
          font-size: 20px;
          font-weight: bold;
        }
        
        .content {
          padding: 16px;
        }
        
        .footer {
          background: #f5f5f5;
          padding: 12px 16px;
          border-top: 1px solid #ddd;
        }
        
        ::slotted([slot="header"]) {
          margin: 0;
        }
      </style>
      
      <div class="card">
        <div class="header">
          <slot name="header">Default Header</slot>
        </div>
        <div class="content">
          <slot>Default content</slot>
        </div>
        <div class="footer">
          <slot name="footer">Default footer</slot>
        </div>
      </div>
    `;
  }
}

customElements.define('card-element', CardElement);

// Usage:
// <card-element>
//   <span slot="header">My Card Title</span>
//   <p>This is the card content</p>
//   <button slot="footer">Action</button>
// </card-element>
```

### Form-Associated Custom Elements

```javascript
class CustomInput extends HTMLElement {
  static formAssociated = true;
  
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    this._internals = this.attachInternals();
    this._value = '';
  }
  
  connectedCallback() {
    this.render();
    
    const input = this.shadowRoot.querySelector('input');
    input.addEventListener('input', (e) => {
      this._value = e.target.value;
      this._internals.setFormValue(this._value);
      
      // Validate
      if (this._value.length < 3) {
        this._internals.setValidity(
          { tooShort: true },
          'Value must be at least 3 characters'
        );
      } else {
        this._internals.setValidity({});
      }
    });
  }
  
  get value() {
    return this._value;
  }
  
  set value(val) {
    this._value = val;
    this._internals.setFormValue(val);
    const input = this.shadowRoot.querySelector('input');
    if (input) {
      input.value = val;
    }
  }
  
  render() {
    this.shadowRoot.innerHTML = `
      <style>
        input {
          width: 100%;
          padding: 8px;
          border: 1px solid #ddd;
          border-radius: 4px;
          font-size: 14px;
        }
        
        input:invalid {
          border-color: red;
        }
      </style>
      
      <input type="text" value="${this._value}" />
    `;
  }
}

customElements.define('custom-input', CustomInput);

// Usage in form:
// <form>
//   <custom-input name="username"></custom-input>
//   <button type="submit">Submit</button>
// </form>
```

## Browser Support

| Browser | Version | Notes |
|---------|---------|-------|
| Chrome | 67+ | Full support |
| Firefox | 63+ | Full support |
| Safari | 10.1+ | Full support |
| Edge | 79+ | Full support |
| Opera | 54+ | Full support |
| IE | None | No support |

### Polyfills

For older browsers, use the Web Components polyfills:

```html
<script src="https://unpkg.com/@webcomponents/webcomponentsjs@2.8.0/webcomponents-loader.js"></script>
```

## Common Mistakes

1. **Not Calling super() in Constructor**
```javascript
// Wrong
class MyElement extends HTMLElement {
  constructor() {
    this.value = 'test'; // Error!
  }
}

// Correct
class MyElement extends HTMLElement {
  constructor() {
    super(); // Must call super() first
    this.value = 'test';
  }
}
```

2. **Modifying Attributes in Constructor**
```javascript
// Wrong
class MyElement extends HTMLElement {
  constructor() {
    super();
    this.setAttribute('data-initialized', 'true'); // Not recommended
  }
}

// Correct
class MyElement extends HTMLElement {
  connectedCallback() {
    this.setAttribute('data-initialized', 'true');
  }
}
```

3. **Not Observing Attributes**
```javascript
// Wrong - attributeChangedCallback won't be called
class MyElement extends HTMLElement {
  attributeChangedCallback(name, oldValue, newValue) {
    console.log('Changed'); // Never called
  }
}

// Correct
class MyElement extends HTMLElement {
  static get observedAttributes() {
    return ['name', 'value'];
  }
  
  attributeChangedCallback(name, oldValue, newValue) {
    console.log('Changed'); // Now it works
  }
}
```

4. **Memory Leaks**
```javascript
// Wrong - event listeners not cleaned up
class MyElement extends HTMLElement {
  connectedCallback() {
    document.addEventListener('scroll', this.handleScroll);
  }
}

// Correct
class MyElement extends HTMLElement {
  connectedCallback() {
    this.handleScroll = this.handleScroll.bind(this);
    document.addEventListener('scroll', this.handleScroll);
  }
  
  disconnectedCallback() {
    document.removeEventListener('scroll', this.handleScroll);
  }
}
```

## Best Practices

1. **Use Shadow DOM for Encapsulation**
2. **Implement Proper Lifecycle Management**
3. **Reflect Important Properties to Attributes**
4. **Dispatch Custom Events for Communication**
5. **Provide Default Content and Styling**
6. **Document Your Custom Elements**
7. **Handle Errors Gracefully**
8. **Use Semantic HTML in Templates**
9. **Optimize Rendering Performance**
10. **Test Across Different Browsers**

## When to Use Custom Elements

Use custom elements when you need to:
- Create reusable UI components
- Encapsulate complex functionality
- Build framework-agnostic components
- Extend existing HTML elements
- Create domain-specific markup
- Share components across projects

Avoid custom elements when:
- Simple CSS classes would suffice
- You need SSR without hydration
- You're in a framework-heavy environment where framework components are more suitable

## Interview Questions

1. What are the two types of custom elements and how do they differ?
2. Explain the lifecycle callbacks of custom elements.
3. Why must custom element names contain a hyphen?
4. What is the difference between properties and attributes?
5. How do you make a custom element work with forms?
6. What is the purpose of observedAttributes?
7. How do you upgrade an element that was parsed before registration?
8. Explain the composed flag in custom events.

## Key Takeaways

- Custom elements provide native browser support for creating reusable components
- Lifecycle callbacks enable proper resource management
- Properties and attributes serve different purposes
- Shadow DOM provides encapsulation
- Custom elements work across frameworks
- Proper cleanup in disconnectedCallback prevents memory leaks
- Form-associated custom elements integrate with native forms
- Browser support is excellent in modern browsers

## Resources

- [MDN: Using Custom Elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements)
- [HTML Living Standard: Custom Elements](https://html.spec.whatwg.org/multipage/custom-elements.html)
- [Web Components Guide](https://www.webcomponents.org/introduction)
- [Custom Elements Everywhere](https://custom-elements-everywhere.com/)
- [Open Web Components](https://open-wc.org/)
