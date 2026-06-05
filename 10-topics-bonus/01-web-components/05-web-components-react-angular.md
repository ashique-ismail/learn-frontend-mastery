# Using Web Components in React and Angular

## The Idea

**In plain English:** Web Components are reusable building blocks you can create once and plug into any website or app — even ones built with different tools like React or Angular. Think of them as universal LEGO bricks: no matter which LEGO set you are using, the brick still fits.

**Real-world analogy:** Imagine a universal USB-C charger cable that works with your laptop, your phone, and your friend's tablet — even though those devices were made by completely different companies. Each device still has to use a small adapter in its port to accept the cable properly.

- The USB-C cable = the Web Component (built once, works everywhere)
- The laptop/phone/tablet = React, Angular, or any other framework (different environments)
- The port adapter each device uses = the wrapper code (the glue each framework writes to handle properties and events correctly)

---

## Introduction

Web Components are framework-agnostic by design, meaning they can work with any JavaScript framework or library. However, each framework has its own way of handling DOM updates, events, and data binding, which can create integration challenges. Understanding how to properly use Web Components in React and Angular is essential for building modern applications that leverage the best of both worlds.

This guide covers the patterns, best practices, and common pitfalls when integrating Web Components into React and Angular applications, enabling you to create truly framework-agnostic component libraries.

## Web Components in React

React and Web Components have different philosophies for managing the DOM, which can create integration challenges. React uses a virtual DOM and synthetic events, while Web Components manipulate the DOM directly and use native browser events.

### Basic Integration

```jsx
import React, { useRef, useEffect } from 'react';

// Import your web component
import './my-counter';

function App() {
  return (
    <div>
      <h1>React + Web Components</h1>
      {/* Basic usage - attributes only */}
      <my-counter count="5" step="2"></my-counter>
    </div>
  );
}

export default App;
```

### Handling Properties vs Attributes

React treats all props as attributes, which only work with string values. For complex data types, you need to use refs.

```jsx
import React, { useRef, useEffect } from 'react';

function UserList() {
  const listRef = useRef(null);
  
  const users = [
    { id: 1, name: 'John Doe', email: 'john@example.com' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
  ];
  
  useEffect(() => {
    // Set complex property using ref
    if (listRef.current) {
      listRef.current.data = users;
    }
  }, [users]);
  
  return (
    <div>
      {/* String attributes work directly */}
      <user-list 
        ref={listRef}
        title="User List"
      ></user-list>
    </div>
  );
}

export default UserList;
```

### React Wrapper Component Pattern

Create React wrapper components for better integration:

```jsx
import React, { useRef, useEffect } from 'react';
import PropTypes from 'prop-types';

// Web component wrapper
function WebCounter({ count, step, onCountChanged }) {
  const counterRef = useRef(null);
  
  // Set properties
  useEffect(() => {
    if (counterRef.current) {
      counterRef.current.count = count;
      counterRef.current.step = step;
    }
  }, [count, step]);
  
  // Handle events
  useEffect(() => {
    const element = counterRef.current;
    if (!element) return;
    
    const handleCountChanged = (event) => {
      if (onCountChanged) {
        onCountChanged(event.detail);
      }
    };
    
    element.addEventListener('count-changed', handleCountChanged);
    
    return () => {
      element.removeEventListener('count-changed', handleCountChanged);
    };
  }, [onCountChanged]);
  
  return <my-counter ref={counterRef}></my-counter>;
}

WebCounter.propTypes = {
  count: PropTypes.number,
  step: PropTypes.number,
  onCountChanged: PropTypes.func
};

WebCounter.defaultProps = {
  count: 0,
  step: 1
};

export default WebCounter;

// Usage
function App() {
  const [count, setCount] = React.useState(0);
  
  const handleCountChanged = (detail) => {
    console.log('Count changed:', detail.count);
    setCount(detail.count);
  };
  
  return (
    <div>
      <WebCounter 
        count={count}
        step={5}
        onCountChanged={handleCountChanged}
      />
      <p>Current count in React state: {count}</p>
    </div>
  );
}
```

### Generic Wrapper Hook

Create a reusable hook for any web component:

```jsx
import { useRef, useEffect } from 'react';

/**
 * Hook to integrate web components with React
 * @param {Object} props - Properties to set on the web component
 * @param {Object} events - Event handlers for web component events
 */
function useWebComponent(props = {}, events = {}) {
  const ref = useRef(null);
  
  // Set properties
  useEffect(() => {
    if (!ref.current) return;
    
    Object.entries(props).forEach(([key, value]) => {
      if (ref.current[key] !== value) {
        ref.current[key] = value;
      }
    });
  }, [props]);
  
  // Attach event listeners
  useEffect(() => {
    if (!ref.current) return;
    
    const element = ref.current;
    const listeners = [];
    
    Object.entries(events).forEach(([event, handler]) => {
      element.addEventListener(event, handler);
      listeners.push([event, handler]);
    });
    
    return () => {
      listeners.forEach(([event, handler]) => {
        element.removeEventListener(event, handler);
      });
    };
  }, [events]);
  
  return ref;
}

// Usage
function TodoApp() {
  const [todos, setTodos] = React.useState([
    { id: 1, text: 'Learn React', completed: false },
    { id: 2, text: 'Learn Web Components', completed: false }
  ]);
  
  const handleTodoAdded = (event) => {
    const newTodo = {
      id: Date.now(),
      text: event.detail.text,
      completed: false
    };
    setTodos([...todos, newTodo]);
  };
  
  const handleTodoToggled = (event) => {
    setTodos(todos.map(todo =>
      todo.id === event.detail.id
        ? { ...todo, completed: event.detail.completed }
        : todo
    ));
  };
  
  const todoListRef = useWebComponent(
    { todos },
    {
      'todo-added': handleTodoAdded,
      'todo-toggled': handleTodoToggled
    }
  );
  
  return <todo-list ref={todoListRef}></todo-list>;
}
```

### React 19+ Support

React 19 improves Web Components support with better property handling:

```jsx
import React from 'react';

// React 19+ automatically handles properties correctly
function ModernReactWebComponent() {
  const users = [
    { id: 1, name: 'John' },
    { id: 2, name: 'Jane' }
  ];
  
  return (
    <user-list
      // Complex objects work directly in React 19+
      data={users}
      // Functions as properties
      onItemClick={(e) => console.log(e.detail)}
      // Still need to use addEventListener for custom events
    ></user-list>
  );
}
```

### TypeScript Integration in React

```tsx
import React, { useRef, useEffect } from 'react';

// Define web component interface
interface MyCounter extends HTMLElement {
  count: number;
  step: number;
  reset(): void;
}

// Declare custom element in JSX namespace
declare global {
  namespace JSX {
    interface IntrinsicElements {
      'my-counter': React.DetailedHTMLProps<
        React.HTMLAttributes<MyCounter>,
        MyCounter
      > & {
        count?: number | string;
        step?: number | string;
      };
    }
  }
}

interface CounterProps {
  initialCount?: number;
  step?: number;
  onCountChanged?: (count: number) => void;
}

const Counter: React.FC<CounterProps> = ({ 
  initialCount = 0, 
  step = 1,
  onCountChanged 
}) => {
  const counterRef = useRef<MyCounter>(null);
  
  useEffect(() => {
    const element = counterRef.current;
    if (!element) return;
    
    element.count = initialCount;
    element.step = step;
  }, [initialCount, step]);
  
  useEffect(() => {
    const element = counterRef.current;
    if (!element || !onCountChanged) return;
    
    const handler = (e: Event) => {
      const customEvent = e as CustomEvent<{ count: number }>;
      onCountChanged(customEvent.detail.count);
    };
    
    element.addEventListener('count-changed', handler);
    
    return () => {
      element.removeEventListener('count-changed', handler);
    };
  }, [onCountChanged]);
  
  const handleReset = () => {
    counterRef.current?.reset();
  };
  
  return (
    <div>
      <my-counter ref={counterRef}></my-counter>
      <button onClick={handleReset}>Reset Counter</button>
    </div>
  );
};

export default Counter;
```

## Web Components in Angular

Angular has excellent built-in support for Web Components through the CUSTOM_ELEMENTS_SCHEMA. Angular's change detection works well with web components.

### Basic Integration

```typescript
// app.module.ts
import { NgModule, CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';

// Import web components
import '../web-components/my-counter';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  schemas: [CUSTOM_ELEMENTS_SCHEMA],  // Enable custom elements
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### Using Web Components in Templates

```typescript
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <div class="container">
      <h1>Angular + Web Components</h1>
      
      <!-- String attributes work directly -->
      <my-counter 
        [attr.count]="count"
        [attr.step]="step"
        (count-changed)="handleCountChanged($event)"
      ></my-counter>
      
      <p>Current count: {{ count }}</p>
    </div>
  `,
  styles: [`
    .container {
      padding: 20px;
    }
  `]
})
export class AppComponent {
  count = 0;
  step = 5;
  
  handleCountChanged(event: CustomEvent) {
    this.count = event.detail.count;
    console.log('Count changed:', this.count);
  }
}
```

### Property Binding

For complex data types, use property binding:

```typescript
// user-list.component.ts
import { Component, ViewChild, ElementRef, AfterViewInit } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-list',
  template: `
    <div>
      <h2>User Management</h2>
      
      <!-- Reference the web component -->
      <user-list 
        #userList
        [attr.title]="title"
        (user-selected)="handleUserSelected($event)"
        (user-deleted)="handleUserDeleted($event)"
      ></user-list>
      
      <button (click)="addUser()">Add User</button>
    </div>
  `
})
export class UserListComponent implements AfterViewInit {
  @ViewChild('userList', { static: false }) userListRef!: ElementRef;
  
  title = 'User Directory';
  
  users: User[] = [
    { id: 1, name: 'John Doe', email: 'john@example.com' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
  ];
  
  ngAfterViewInit() {
    // Set complex property after view init
    const element = this.userListRef.nativeElement;
    element.data = this.users;
  }
  
  handleUserSelected(event: CustomEvent<{ userId: number }>) {
    console.log('User selected:', event.detail.userId);
  }
  
  handleUserDeleted(event: CustomEvent<{ userId: number }>) {
    this.users = this.users.filter(u => u.id !== event.detail.userId);
    this.userListRef.nativeElement.data = this.users;
  }
  
  addUser() {
    const newUser: User = {
      id: Date.now(),
      name: `User ${this.users.length + 1}`,
      email: `user${this.users.length + 1}@example.com`
    };
    this.users = [...this.users, newUser];
    this.userListRef.nativeElement.data = this.users;
  }
}
```

### Angular Directive Wrapper

Create a directive for better integration:

```typescript
// web-component.directive.ts
import {
  Directive,
  ElementRef,
  Input,
  Output,
  EventEmitter,
  OnInit,
  OnChanges,
  SimpleChanges,
  OnDestroy
} from '@angular/core';

@Directive({
  selector: '[webComponent]'
})
export class WebComponentDirective implements OnInit, OnChanges, OnDestroy {
  @Input() properties: Record<string, any> = {};
  @Input() events: Record<string, string> = {};
  @Output() webComponentEvent = new EventEmitter<CustomEvent>();
  
  private eventListeners: Array<[string, EventListener]> = [];
  
  constructor(private el: ElementRef) {}
  
  ngOnInit() {
    this.attachEventListeners();
  }
  
  ngOnChanges(changes: SimpleChanges) {
    if (changes['properties']) {
      this.updateProperties();
    }
    
    if (changes['events']) {
      this.removeEventListeners();
      this.attachEventListeners();
    }
  }
  
  ngOnDestroy() {
    this.removeEventListeners();
  }
  
  private updateProperties() {
    const element = this.el.nativeElement;
    Object.entries(this.properties).forEach(([key, value]) => {
      if (element[key] !== value) {
        element[key] = value;
      }
    });
  }
  
  private attachEventListeners() {
    const element = this.el.nativeElement;
    
    Object.entries(this.events).forEach(([event, _]) => {
      const listener = ((e: Event) => {
        this.webComponentEvent.emit(e as CustomEvent);
      }) as EventListener;
      
      element.addEventListener(event, listener);
      this.eventListeners.push([event, listener]);
    });
  }
  
  private removeEventListeners() {
    const element = this.el.nativeElement;
    
    this.eventListeners.forEach(([event, listener]) => {
      element.removeEventListener(event, listener);
    });
    
    this.eventListeners = [];
  }
}

// Usage
// <my-counter
//   webComponent
//   [properties]="{ count: count, step: step }"
//   [events]="{ 'count-changed': 'handleCountChanged' }"
//   (webComponentEvent)="handleEvent($event)"
// ></my-counter>
```

### Standalone Component (Angular 14+)

```typescript
// counter.component.ts
import { Component, ViewChild, ElementRef, AfterViewInit, CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-counter',
  standalone: true,
  imports: [CommonModule],
  schemas: [CUSTOM_ELEMENTS_SCHEMA],
  template: `
    <div class="counter-wrapper">
      <my-counter 
        #counter
        [attr.step]="step"
        (count-changed)="handleCountChanged($event)"
      ></my-counter>
      
      <div class="controls">
        <button (click)="reset()">Reset</button>
        <button (click)="changeStep()">Change Step</button>
      </div>
      
      <p>Count: {{ currentCount }}</p>
    </div>
  `,
  styles: [`
    .counter-wrapper {
      padding: 20px;
    }
    
    .controls {
      margin-top: 16px;
      display: flex;
      gap: 8px;
    }
    
    button {
      padding: 8px 16px;
      background: #667eea;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
  `]
})
export class CounterComponent implements AfterViewInit {
  @ViewChild('counter') counterRef!: ElementRef;
  
  step = 1;
  currentCount = 0;
  
  ngAfterViewInit() {
    const element = this.counterRef.nativeElement;
    element.count = this.currentCount;
  }
  
  handleCountChanged(event: CustomEvent<{ count: number }>) {
    this.currentCount = event.detail.count;
  }
  
  reset() {
    const element = this.counterRef.nativeElement;
    element.reset?.();
  }
  
  changeStep() {
    this.step = this.step === 1 ? 5 : 1;
    const element = this.counterRef.nativeElement;
    element.step = this.step;
  }
}
```

### TypeScript Definitions for Angular

```typescript
// web-components.d.ts
declare namespace JSX {
  interface IntrinsicElements {
    'my-counter': any;
    'user-list': any;
    'todo-list': any;
  }
}

// my-counter.d.ts
interface MyCounterElement extends HTMLElement {
  count: number;
  step: number;
  reset(): void;
}

declare global {
  interface HTMLElementTagNameMap {
    'my-counter': MyCounterElement;
  }
}

export { MyCounterElement };

// Usage in component
import { MyCounterElement } from './types/my-counter';

@ViewChild('counter') counterRef!: ElementRef<MyCounterElement>;

someMethod() {
  this.counterRef.nativeElement.count = 10;
  this.counterRef.nativeElement.reset();
}
```

## Forms Integration

### React Controlled Components

```jsx
import React, { useRef, useEffect, useState } from 'react';

function FormWithWebComponent() {
  const inputRef = useRef(null);
  const [value, setValue] = useState('');
  
  useEffect(() => {
    const element = inputRef.current;
    if (!element) return;
    
    // Set initial value
    element.value = value;
    
    // Listen for changes
    const handleChange = (e) => {
      setValue(e.detail.value);
    };
    
    element.addEventListener('value-changed', handleChange);
    
    return () => {
      element.removeEventListener('value-changed', handleChange);
    };
  }, []);
  
  // Update web component when React state changes
  useEffect(() => {
    if (inputRef.current) {
      inputRef.current.value = value;
    }
  }, [value]);
  
  const handleSubmit = (e) => {
    e.preventDefault();
    console.log('Submitted value:', value);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <custom-input 
        ref={inputRef}
        placeholder="Enter text..."
      ></custom-input>
      
      <button type="submit">Submit</button>
      
      <p>Current value: {value}</p>
    </form>
  );
}
```

### Angular Forms Integration

```typescript
// custom-input.component.ts
import {
  Component,
  ViewChild,
  ElementRef,
  AfterViewInit,
  forwardRef
} from '@angular/core';
import {
  ControlValueAccessor,
  NG_VALUE_ACCESSOR,
  FormControl,
  ReactiveFormsModule
} from '@angular/forms';

@Component({
  selector: 'app-custom-input-wrapper',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <custom-input 
      #input
      [attr.placeholder]="placeholder"
    ></custom-input>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => CustomInputWrapperComponent),
      multi: true
    }
  ]
})
export class CustomInputWrapperComponent implements AfterViewInit, ControlValueAccessor {
  @ViewChild('input') inputRef!: ElementRef;
  
  placeholder = '';
  
  private onChange: (value: any) => void = () => {};
  private onTouched: () => void = () => {};
  
  ngAfterViewInit() {
    const element = this.inputRef.nativeElement;
    
    element.addEventListener('value-changed', (e: Event) => {
      const customEvent = e as CustomEvent;
      this.onChange(customEvent.detail.value);
    });
    
    element.addEventListener('blur', () => {
      this.onTouched();
    });
  }
  
  writeValue(value: any): void {
    if (this.inputRef) {
      this.inputRef.nativeElement.value = value;
    }
  }
  
  registerOnChange(fn: any): void {
    this.onChange = fn;
  }
  
  registerOnTouched(fn: any): void {
    this.onTouched = fn;
  }
  
  setDisabledState(isDisabled: boolean): void {
    if (this.inputRef) {
      this.inputRef.nativeElement.disabled = isDisabled;
    }
  }
}

// Usage in form
@Component({
  selector: 'app-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <app-custom-input-wrapper 
        formControlName="name"
        placeholder="Enter name"
      ></app-custom-input-wrapper>
      
      <button type="submit">Submit</button>
      
      <p>Value: {{ form.get('name')?.value }}</p>
    </form>
  `
})
export class FormComponent {
  form = new FormGroup({
    name: new FormControl('')
  });
  
  onSubmit() {
    console.log('Form value:', this.form.value);
  }
}
```

## Build Configuration

### Webpack Configuration for Web Components

```javascript
// webpack.config.js
module.exports = {
  // ... other config
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader'
        }
      }
    ]
  },
  resolve: {
    extensions: ['.js', '.ts', '.tsx']
  },
  // Don't bundle web components with main bundle
  externals: {
    'my-web-components': 'my-web-components'
  }
};
```

### Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      // Include web components in JSX pragma
      include: /\.(jsx|tsx|js|ts)$/
    })
  ],
  optimizeDeps: {
    exclude: ['my-web-components'] // Don't pre-bundle web components
  }
});
```

## Browser Support

Both React and Angular work with Web Components in all modern browsers that support the Web Components standards.

| Framework | Version | Web Components Support |
|-----------|---------|------------------------|
| React | 16+ | Good (manual property setting) |
| React | 19+ | Excellent (automatic properties) |
| Angular | 6+ | Excellent (native support) |

## Common Mistakes

1. **Not Setting Properties in React**
```jsx
// Wrong - only sets attribute
<user-list data={users}></user-list>

// Correct - use ref to set property
const ref = useRef();
useEffect(() => {
  ref.current.data = users;
}, [users]);
<user-list ref={ref}></user-list>
```

2. **Forgetting CUSTOM_ELEMENTS_SCHEMA in Angular**
```typescript
// Wrong - Angular will throw errors
@NgModule({
  // Missing schema
})

// Correct
@NgModule({
  schemas: [CUSTOM_ELEMENTS_SCHEMA]
})
```

3. **Not Handling Custom Events Properly**
```jsx
// Wrong - React doesn't auto-bind custom events
<my-counter onCountChanged={handler}></my-counter>

// Correct - use addEventListener
useEffect(() => {
  ref.current.addEventListener('count-changed', handler);
  return () => ref.current.removeEventListener('count-changed', handler);
}, []);
```

## Best Practices

1. **Create Framework Wrappers** - Abstract web component complexity
2. **Use TypeScript Definitions** - Provide type safety
3. **Handle Memory Leaks** - Remove event listeners properly
4. **Document Integration Patterns** - Help other developers
5. **Test Across Frameworks** - Ensure compatibility
6. **Version Carefully** - Breaking changes affect all frameworks
7. **Provide Examples** - Show integration in popular frameworks

## When to Use

Use Web Components with frameworks when:
- Building framework-agnostic component libraries
- Migrating between frameworks gradually
- Sharing components across different applications
- Need style encapsulation
- Building micro-frontends

Use framework components when:
- Working within a single framework
- Need deep framework integration
- Want optimal performance for that framework
- Team is familiar with framework patterns

## Interview Questions

1. What are the challenges of using Web Components in React?
2. How does Angular's CUSTOM_ELEMENTS_SCHEMA work?
3. Why do you need to use refs for properties in React?
4. How do you handle custom events in React vs Angular?
5. What's the difference between attributes and properties?
6. How does React 19 improve Web Components support?
7. How do you integrate web components with Angular forms?
8. What are the best practices for framework-agnostic components?

## Key Takeaways

- Web Components work in all major frameworks
- React requires manual property and event handling (before v19)
- Angular has excellent native support via CUSTOM_ELEMENTS_SCHEMA
- Framework wrappers improve developer experience
- TypeScript definitions enhance type safety
- Proper event listener cleanup prevents memory leaks
- Each framework has different integration patterns
- Web Components enable true framework-agnostic development

## Resources

- [Custom Elements Everywhere](https://custom-elements-everywhere.com/)
- [React - Web Components](https://react.dev/reference/react-dom/components#custom-html-elements)
- [Angular - Using Web Components](https://angular.io/guide/elements)
- [Web Components in React](https://lit.dev/docs/frameworks/react/)
- [Web Components in Angular](https://lit.dev/docs/frameworks/angular/)
