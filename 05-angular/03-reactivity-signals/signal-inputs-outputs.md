# Signal Inputs and Outputs in Angular

## Table of Contents
- [Introduction](#introduction)
- [input() Function](#input-function)
- [output() Function](#output-function)
- [model() Two-Way Binding](#model-two-way-binding)
- [Input Transforms](#input-transforms)
- [Required vs Optional](#required-vs-optional)
- [Migration from Decorators](#migration-from-decorators)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Angular 17+ introduces signal-based alternatives to traditional `@Input()` and `@Output()` decorators. The new `input()`, `output()`, and `model()` functions provide a more type-safe, reactive way to handle component communication while integrating seamlessly with Angular's signal system.

Key benefits:
- **Type safety**: Better TypeScript inference
- **Reactivity**: Automatic change detection with signals
- **Simpler syntax**: Less boilerplate than decorators
- **Better DX**: Improved developer experience with computed values
- **Two-way binding**: `model()` simplifies two-way data flow

## input() Function

### Basic Input Signals

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div class="user-card">
      <h3>{{ name() }}</h3>
      <p>Age: {{ age() }}</p>
      <p>Email: {{ email() }}</p>
    </div>
  `
})
export class UserCardComponent {
  // Signal inputs - call as functions in template
  name = input<string>(''); // Optional with default
  age = input.required<number>(); // Required input
  email = input<string>(''); // Optional with default
}

// Usage
@Component({
  template: `
    <app-user-card 
      name="John Doe" 
      [age]="30" 
      email="john@example.com" />
  `
})
export class ParentComponent {}
```

### Required vs Optional Inputs

```typescript
import { Component, input } from '@angular/core';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-product-card',
  template: `
    <div class="product-card">
      <!-- Required inputs always have values -->
      <h3>{{ product().name }}</h3>
      <p>${{ product().price }}</p>
      
      <!-- Optional inputs need null checks -->
      @if (discount()) {
        <span class="badge">{{ discount() }}% OFF</span>
      }
      
      <p class="category">{{ category() }}</p>
    </div>
  `
})
export class ProductCardComponent {
  // Required input - must be provided
  product = input.required<Product>();
  
  // Optional with default value
  category = input<string>('General');
  
  // Optional without default (nullable)
  discount = input<number | null>(null);
  
  // Optional with transformation
  showDetails = input<boolean>(false);
}

// Usage
@Component({
  template: `
    <!-- product is required -->
    <app-product-card [product]="myProduct" />
    
    <!-- With optional inputs -->
    <app-product-card 
      [product]="myProduct"
      category="Electronics"
      [discount]="15"
      [showDetails]="true" />
  `
})
export class ShopComponent {
  myProduct: Product = {
    id: 1,
    name: 'Laptop',
    price: 999
  };
}
```

### Input Aliases

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-aliased',
  template: `
    <div>
      <p>User ID: {{ userId() }}</p>
      <p>Disabled: {{ isDisabled() }}</p>
    </div>
  `
})
export class AliasedComponent {
  // Internal name: userId, external name: id
  userId = input.required<number>({ alias: 'id' });
  
  // Internal: isDisabled, external: disabled
  isDisabled = input<boolean>(false, { alias: 'disabled' });
}

// Usage
@Component({
  template: `
    <!-- Use external names in template -->
    <app-aliased [id]="123" [disabled]="true" />
  `
})
export class ParentComponent {}
```

### Computed from Inputs

```typescript
import { Component, input, computed } from '@angular/core';

interface Task {
  id: number;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

@Component({
  selector: 'app-task-item',
  template: `
    <div [class]="cssClass()">
      <input 
        type="checkbox" 
        [checked]="task().completed">
      <span [class.completed]="task().completed">
        {{ task().title }}
      </span>
      <span class="priority-badge" [class]="priorityClass()">
        {{ task().priority }}
      </span>
      @if (isHighPriority()) {
        <span class="urgent">!</span>
      }
    </div>
  `
})
export class TaskItemComponent {
  task = input.required<Task>();
  highlighted = input<boolean>(false);
  
  // Computed values from inputs
  isHighPriority = computed(() => 
    this.task().priority === 'high'
  );
  
  priorityClass = computed(() => 
    `priority-${this.task().priority}`
  );
  
  cssClass = computed(() => {
    const classes = ['task-item'];
    if (this.task().completed) classes.push('completed');
    if (this.highlighted()) classes.push('highlighted');
    if (this.isHighPriority()) classes.push('urgent');
    return classes.join(' ');
  });
}
```

### Effects with Input Signals

```typescript
import { Component, input, effect, inject } from '@angular/core';
import { LoggerService } from './logger.service';

@Component({
  selector: 'app-analytics-tracker',
  template: `
    <div>
      <p>Tracking page: {{ pageId() }}</p>
    </div>
  `
})
export class AnalyticsTrackerComponent {
  private logger = inject(LoggerService);
  
  pageId = input.required<string>();
  userId = input<number | null>(null);
  
  constructor() {
    // Track page views when pageId changes
    effect(() => {
      const page = this.pageId();
      const user = this.userId();
      
      this.logger.log('Page view', {
        pageId: page,
        userId: user,
        timestamp: new Date()
      });
    });
  }
}
```

## output() Function

### Basic Output Signals

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <p>Count: {{ count }}</p>
      <button (click)="increment()">+</button>
      <button (click)="decrement()">-</button>
      <button (click)="reset()">Reset</button>
    </div>
  `
})
export class CounterComponent {
  count = 0;
  
  // Output signals
  countChange = output<number>();
  reset$ = output<void>(); // void for events without data
  thresholdReached = output<{ value: number; threshold: number }>();
  
  increment(): void {
    this.count++;
    this.countChange.emit(this.count);
    
    if (this.count >= 10) {
      this.thresholdReached.emit({
        value: this.count,
        threshold: 10
      });
    }
  }
  
  decrement(): void {
    this.count--;
    this.countChange.emit(this.count);
  }
  
  reset(): void {
    this.count = 0;
    this.reset$.emit();
    this.countChange.emit(0);
  }
}

// Usage
@Component({
  template: `
    <app-counter 
      (countChange)="onCountChange($event)"
      (reset$)="onReset()"
      (thresholdReached)="onThreshold($event)" />
    <p>Parent count: {{ parentCount }}</p>
  `
})
export class ParentComponent {
  parentCount = 0;
  
  onCountChange(count: number): void {
    this.parentCount = count;
  }
  
  onReset(): void {
    console.log('Counter was reset');
  }
  
  onThreshold(event: { value: number; threshold: number }): void {
    console.log(`Threshold ${event.threshold} reached with value ${event.value}`);
  }
}
```

### Output Aliases

```typescript
import { Component, output } from '@angular/core';

@Component({
  selector: 'app-form-field',
  template: `
    <input 
      [value]="value"
      (input)="onInput($event)"
      (blur)="onBlur()">
  `
})
export class FormFieldComponent {
  value = '';
  
  // Internal name: valueChanged, external name: valueChange
  valueChanged = output<string>({ alias: 'valueChange' });
  
  // Internal name: blurred, external name: blur
  blurred = output<void>({ alias: 'blur' });
  
  onInput(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.value = input.value;
    this.valueChanged.emit(this.value);
  }
  
  onBlur(): void {
    this.blurred.emit();
  }
}

// Usage
@Component({
  template: `
    <!-- Use external names -->
    <app-form-field 
      (valueChange)="handleChange($event)"
      (blur)="handleBlur()" />
  `
})
export class ParentComponent {}
```

### Complex Event Payloads

```typescript
import { Component, output, input } from '@angular/core';

interface SelectionEvent {
  item: any;
  index: number;
  shiftKey: boolean;
  ctrlKey: boolean;
}

interface DragEvent {
  item: any;
  fromIndex: number;
  toIndex: number;
}

@Component({
  selector: 'app-list',
  template: `
    <div 
      *ngFor="let item of items(); let i = index"
      (click)="handleClick(item, i, $event)"
      (dragstart)="handleDragStart(item, i)"
      (drop)="handleDrop(item, i)"
      draggable="true">
      {{ item.name }}
    </div>
  `
})
export class ListComponent {
  items = input.required<any[]>();
  
  // Complex event outputs
  itemSelected = output<SelectionEvent>();
  itemsReordered = output<DragEvent>();
  selectionCleared = output<void>();
  
  private draggedItem: { item: any; index: number } | null = null;
  
  handleClick(item: any, index: number, event: MouseEvent): void {
    this.itemSelected.emit({
      item,
      index,
      shiftKey: event.shiftKey,
      ctrlKey: event.ctrlKey
    });
  }
  
  handleDragStart(item: any, index: number): void {
    this.draggedItem = { item, index };
  }
  
  handleDrop(item: any, toIndex: number): void {
    if (!this.draggedItem) return;
    
    this.itemsReordered.emit({
      item: this.draggedItem.item,
      fromIndex: this.draggedItem.index,
      toIndex
    });
    
    this.draggedItem = null;
  }
}

// Usage
@Component({
  template: `
    <app-list 
      [items]="myItems"
      (itemSelected)="onSelect($event)"
      (itemsReordered)="onReorder($event)" />
  `
})
export class ParentComponent {
  myItems = [
    { id: 1, name: 'Item 1' },
    { id: 2, name: 'Item 2' },
    { id: 3, name: 'Item 3' }
  ];
  
  onSelect(event: SelectionEvent): void {
    console.log('Selected:', event.item.name);
    if (event.ctrlKey) {
      console.log('Ctrl+Click - add to selection');
    }
  }
  
  onReorder(event: DragEvent): void {
    // Reorder array
    const items = [...this.myItems];
    const [removed] = items.splice(event.fromIndex, 1);
    items.splice(event.toIndex, 0, removed);
    this.myItems = items;
  }
}
```

## model() Two-Way Binding

### Basic Two-Way Binding

```typescript
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-custom-input',
  template: `
    <input 
      [value]="value()"
      (input)="value.set($any($event.target).value)"
      placeholder="Enter text">
  `
})
export class CustomInputComponent {
  // model() creates both input and output
  value = model<string>('');
}

// Usage - two-way binding with [(value)]
@Component({
  template: `
    <app-custom-input [(value)]="text" />
    <p>Parent value: {{ text }}</p>
  `
})
export class ParentComponent {
  text = 'Initial text';
}
```

### Required Model

```typescript
import { Component, model } from '@angular/core';

interface FormData {
  name: string;
  email: string;
}

@Component({
  selector: 'app-form',
  template: `
    <div>
      <input 
        [value]="data().name"
        (input)="updateName($any($event.target).value)">
      <input 
        [value]="data().email"
        (input)="updateEmail($any($event.target).value)">
    </div>
  `
})
export class FormComponent {
  // Required two-way binding
  data = model.required<FormData>();
  
  updateName(name: string): void {
    this.data.update(d => ({ ...d, name }));
  }
  
  updateEmail(email: string): void {
    this.data.update(d => ({ ...d, email }));
  }
}

// Usage
@Component({
  template: `
    <app-form [(data)]="formData" />
    <pre>{{ formData | json }}</pre>
  `
})
export class ParentComponent {
  formData: FormData = {
    name: '',
    email: ''
  };
}
```

### Model with Validation

```typescript
import { Component, model, computed, effect } from '@angular/core';

@Component({
  selector: 'app-validated-input',
  template: `
    <div>
      <input 
        [value]="value()"
        (input)="handleInput($any($event.target).value)"
        [class.invalid]="!isValid()">
      @if (!isValid() && value()) {
        <span class="error">{{ errorMessage() }}</span>
      }
    </div>
  `
})
export class ValidatedInputComponent {
  value = model<string>('');
  min = input<number>(0);
  max = input<number>(100);
  
  isValid = computed(() => {
    const val = this.value();
    const num = Number(val);
    return !val || (!isNaN(num) && num >= this.min() && num <= this.max());
  });
  
  errorMessage = computed(() => {
    if (this.isValid()) return '';
    const val = this.value();
    const num = Number(val);
    if (isNaN(num)) return 'Must be a number';
    if (num < this.min()) return `Must be at least ${this.min()}`;
    if (num > this.max()) return `Must be at most ${this.max()}`;
    return '';
  });
  
  constructor() {
    // Emit validation events
    effect(() => {
      console.log('Valid:', this.isValid());
    });
  }
  
  handleInput(val: string): void {
    this.value.set(val);
  }
}

// Usage
@Component({
  template: `
    <app-validated-input 
      [(value)]="age"
      [min]="0"
      [max]="120" />
    <p>Age: {{ age }}</p>
  `
})
export class ParentComponent {
  age = '25';
}
```

### Custom Two-Way Binding Logic

```typescript
import { Component, model, computed } from '@angular/core';

@Component({
  selector: 'app-temperature',
  template: `
    <div>
      <label>
        Celsius:
        <input 
          type="number"
          [value]="celsius()"
          (input)="celsius.set(+$any($event.target).value)">
      </label>
      
      <label>
        Fahrenheit:
        <input 
          type="number"
          [value]="fahrenheit()"
          (input)="updateFahrenheit(+$any($event.target).value)">
      </label>
    </div>
  `
})
export class TemperatureComponent {
  // Primary model
  celsius = model<number>(0);
  
  // Computed from model
  fahrenheit = computed(() => this.celsius() * 9/5 + 32);
  
  // Update model from computed value
  updateFahrenheit(f: number): void {
    this.celsius.set((f - 32) * 5/9);
  }
}

// Usage
@Component({
  template: `
    <app-temperature [(celsius)]="temperature" />
    <p>Temperature: {{ temperature }}°C</p>
  `
})
export class ParentComponent {
  temperature = 20;
}
```

### Model with Throttling

```typescript
import { Component, model, effect } from '@angular/core';

@Component({
  selector: 'app-throttled-input',
  template: `
    <input 
      [value]="localValue"
      (input)="handleInput($any($event.target).value)">
  `
})
export class ThrottledInputComponent {
  value = model<string>('');
  localValue = '';
  
  private timeout: any;
  
  constructor() {
    // Sync local value from model
    effect(() => {
      this.localValue = this.value();
    });
  }
  
  handleInput(val: string): void {
    this.localValue = val;
    
    // Throttle updates to model
    clearTimeout(this.timeout);
    this.timeout = setTimeout(() => {
      this.value.set(val);
    }, 300);
  }
}

// Usage
@Component({
  template: `
    <app-throttled-input [(value)]="searchTerm" />
    <p>Search term: {{ searchTerm }}</p>
  `
})
export class ParentComponent {
  searchTerm = '';
}
```

## Input Transforms

### Built-in Transforms

```typescript
import { Component, input } from '@angular/core';
import { booleanAttribute, numberAttribute } from '@angular/core';

@Component({
  selector: 'app-transformed',
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <p>Enabled: {{ enabled() }}</p>
      <p>Width: {{ width() }}</p>
    </div>
  `
})
export class TransformedComponent {
  // Transform string to number
  count = input<number, string | number>(0, {
    transform: (value: string | number) => {
      return typeof value === 'string' ? parseInt(value, 10) : value;
    }
  });
  
  // Transform to boolean
  enabled = input<boolean, boolean | string>(false, {
    transform: booleanAttribute
  });
  
  // Transform with numberAttribute
  width = input<number, string | number>(100, {
    transform: numberAttribute
  });
}

// Usage
@Component({
  template: `
    <!-- All these work -->
    <app-transformed count="42" enabled width="200" />
    <app-transformed [count]="42" [enabled]="true" [width]="200" />
  `
})
export class ParentComponent {}
```

### Custom Transform Functions

```typescript
import { Component, input } from '@angular/core';

// Custom transform functions
function toUpperCase(value: string): string {
  return value.toUpperCase();
}

function parseDate(value: string | Date): Date {
  return typeof value === 'string' ? new Date(value) : value;
}

function parseCsv(value: string): string[] {
  return value.split(',').map(s => s.trim());
}

@Component({
  selector: 'app-custom-transforms',
  template: `
    <div>
      <p>Name: {{ name() }}</p>
      <p>Date: {{ date() | date }}</p>
      <p>Tags: {{ tags().join(', ') }}</p>
    </div>
  `
})
export class CustomTransformsComponent {
  name = input<string, string>('', {
    transform: toUpperCase
  });
  
  date = input<Date, string | Date>(new Date(), {
    transform: parseDate
  });
  
  tags = input<string[], string>([], {
    transform: parseCsv
  });
}

// Usage
@Component({
  template: `
    <app-custom-transforms 
      name="john doe"
      date="2024-01-01"
      tags="angular,signals,typescript" />
  `
})
export class ParentComponent {}
```

### Validation Transforms

```typescript
import { Component, input, computed } from '@angular/core';

function validateEmail(value: string): string {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(value)) {
    throw new Error(`Invalid email: ${value}`);
  }
  return value;
}

function clamp(min: number, max: number) {
  return (value: number): number => {
    return Math.max(min, Math.min(max, value));
  };
}

@Component({
  selector: 'app-validated',
  template: `
    <div>
      <p>Email: {{ email() }}</p>
      <p>Age: {{ age() }}</p>
      <p>Percentage: {{ percentage() }}%</p>
    </div>
  `
})
export class ValidatedComponent {
  email = input<string, string>('', {
    transform: validateEmail
  });
  
  age = input<number, number>(0, {
    transform: clamp(0, 120)
  });
  
  percentage = input<number, number>(0, {
    transform: clamp(0, 100)
  });
}
```

## Migration from Decorators

### Before: Decorator-based

```typescript
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-old-style',
  template: `
    <div>
      <p>{{ name }}</p>
      <button (click)="handleClick()">Click</button>
    </div>
  `
})
export class OldStyleComponent {
  @Input() name: string = '';
  @Input() age!: number;
  @Input({ required: true }) id!: string;
  @Input({ alias: 'userName' }) user: string = '';
  
  @Output() nameChange = new EventEmitter<string>();
  @Output() clicked = new EventEmitter<void>();
  
  handleClick(): void {
    this.clicked.emit();
  }
  
  updateName(newName: string): void {
    this.name = newName;
    this.nameChange.emit(newName);
  }
}
```

### After: Signal-based

```typescript
import { Component, input, output, model } from '@angular/core';

@Component({
  selector: 'app-new-style',
  template: `
    <div>
      <p>{{ name() }}</p>
      <button (click)="handleClick()">Click</button>
    </div>
  `
})
export class NewStyleComponent {
  name = model<string>('');
  age = input<number>(0);
  id = input.required<string>();
  user = input<string>('', { alias: 'userName' });
  
  clicked = output<void>();
  
  handleClick(): void {
    this.clicked.emit();
  }
  
  updateName(newName: string): void {
    this.name.set(newName);
  }
}
```

### Migration Steps

```typescript
// Step 1: Identify inputs and outputs
// OLD:
@Input() value: string = '';
@Output() valueChange = new EventEmitter<string>();

// Step 2: Determine if two-way binding
// If Input + Output with 'Change' suffix -> use model()
value = model<string>('');

// Step 3: Convert simple inputs
// OLD:
@Input() title: string = 'Default';
// NEW:
title = input<string>('Default');

// Step 4: Convert required inputs
// OLD:
@Input({ required: true }) id!: string;
// NEW:
id = input.required<string>();

// Step 5: Convert outputs
// OLD:
@Output() save = new EventEmitter<string>();
// NEW:
save = output<string>();

// Step 6: Update template bindings
// OLD: {{ title }}
// NEW: {{ title() }}

// Step 7: Update effects/computed
// Can now use inputs in computed/effect
const upperTitle = computed(() => this.title().toUpperCase());
```

## Advanced Patterns

### Input Validation Pattern

```typescript
import { Component, input, computed, effect } from '@angular/core';

interface ValidationResult {
  valid: boolean;
  errors: string[];
}

@Component({
  selector: 'app-validated-form',
  template: `
    <div>
      <input 
        [value]="email()"
        (input)="email.set($any($event.target).value)">
      
      @if (validation().errors.length > 0) {
        <ul class="errors">
          <li *ngFor="let error of validation().errors">
            {{ error }}
          </li>
        </ul>
      }
    </div>
  `
})
export class ValidatedFormComponent {
  email = model<string>('');
  
  validation = computed<ValidationResult>(() => {
    const value = this.email();
    const errors: string[] = [];
    
    if (!value) {
      errors.push('Email is required');
    } else {
      if (!value.includes('@')) {
        errors.push('Email must contain @');
      }
      if (value.length < 5) {
        errors.push('Email must be at least 5 characters');
      }
    }
    
    return {
      valid: errors.length === 0,
      errors
    };
  });
  
  constructor() {
    effect(() => {
      console.log('Validation:', this.validation());
    });
  }
}
```

### Parent-Child Communication Pattern

```typescript
import { Component, input, output, signal } from '@angular/core';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

// Child Component
@Component({
  selector: 'app-todo-item',
  template: `
    <div [class.completed]="todo().completed">
      <input 
        type="checkbox"
        [checked]="todo().completed"
        (change)="toggleComplete()">
      <span>{{ todo().text }}</span>
      <button (click)="delete()">Delete</button>
    </div>
  `
})
export class TodoItemComponent {
  todo = input.required<Todo>();
  
  todoToggled = output<number>();
  todoDeleted = output<number>();
  
  toggleComplete(): void {
    this.todoToggled.emit(this.todo().id);
  }
  
  delete(): void {
    this.todoDeleted.emit(this.todo().id);
  }
}

// Parent Component
@Component({
  selector: 'app-todo-list',
  template: `
    <div>
      @for (todo of todos(); track todo.id) {
        <app-todo-item 
          [todo]="todo"
          (todoToggled)="handleToggle($event)"
          (todoDeleted)="handleDelete($event)" />
      }
    </div>
  `
})
export class TodoListComponent {
  todos = signal<Todo[]>([
    { id: 1, text: 'Learn Signals', completed: false },
    { id: 2, text: 'Build App', completed: false }
  ]);
  
  handleToggle(id: number): void {
    this.todos.update(todos =>
      todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  }
  
  handleDelete(id: number): void {
    this.todos.update(todos => todos.filter(t => t.id !== id));
  }
}
```

### Computed Output Pattern

```typescript
import { Component, input, output, computed, effect } from '@angular/core';

@Component({
  selector: 'app-price-calculator',
  template: `
    <div>
      <p>Quantity: {{ quantity() }}</p>
      <p>Price: ${{ price() }}</p>
      <p>Total: ${{ total() }}</p>
    </div>
  `
})
export class PriceCalculatorComponent {
  quantity = input.required<number>();
  price = input.required<number>();
  
  totalChanged = output<number>();
  
  total = computed(() => this.quantity() * this.price());
  
  constructor() {
    // Emit when computed value changes
    effect(() => {
      this.totalChanged.emit(this.total());
    });
  }
}

// Usage
@Component({
  template: `
    <app-price-calculator 
      [quantity]="qty"
      [price]="price"
      (totalChanged)="handleTotalChange($event)" />
  `
})
export class ParentComponent {
  qty = 5;
  price = 10;
  
  handleTotalChange(total: number): void {
    console.log('New total:', total);
  }
}
```

## Common Mistakes

### Mistake 1: Forgetting to Call Input Signals

```typescript
// BAD: Not calling signal
@Component({
  template: '<p>{{ name }}</p>' // undefined!
})
export class BadComponent {
  name = input<string>('');
}

// GOOD: Call as function
@Component({
  template: '<p>{{ name() }}</p>'
})
export class GoodComponent {
  name = input<string>('');
}
```

### Mistake 2: Modifying Input Signal Values

```typescript
// BAD: Trying to modify input signal
@Component({...})
export class BadComponent {
  name = input<string>('');
  
  changeName(): void {
    this.name.set('New Name'); // ERROR! Input signals are read-only
  }
}

// GOOD: Use model() for two-way binding
@Component({...})
export class GoodComponent {
  name = model<string>('');
  
  changeName(): void {
    this.name.set('New Name'); // OK!
  }
}
```

### Mistake 3: Wrong Output Event Type

```typescript
// BAD: Using EventEmitter
@Component({...})
export class BadComponent {
  clicked = output<EventEmitter<void>>(); // Wrong!
}

// GOOD: Output type is the emitted value type
@Component({...})
export class GoodComponent {
  clicked = output<void>();
  numberChanged = output<number>();
  dataLoaded = output<{ id: number; data: any }>();
}
```

### Mistake 4: Not Using Required for Mandatory Inputs

```typescript
// BAD: Optional input that should be required
@Component({...})
export class BadComponent {
  id = input<number>(); // Might be undefined!
  
  loadData(): void {
    fetch(`/api/data/${this.id()}`); // Runtime error if undefined
  }
}

// GOOD: Use required for mandatory inputs
@Component({...})
export class GoodComponent {
  id = input.required<number>(); // Compile-time check
  
  loadData(): void {
    fetch(`/api/data/${this.id()}`); // Always has value
  }
}
```

### Mistake 5: Misusing model() for One-Way Data

```typescript
// BAD: Using model() when only input is needed
@Component({...})
export class BadComponent {
  readOnlyData = model<string>(''); // Unnecessary!
}

// GOOD: Use input() for one-way data
@Component({...})
export class GoodComponent {
  readOnlyData = input<string>('');
}

// Use model() only for two-way binding
@Component({...})
export class TwoWayComponent {
  editableData = model<string>('');
}
```

## Best Practices

### 1. Use input() for One-Way Data Flow

```typescript
@Component({...})
export class DisplayComponent {
  // Read-only data from parent
  data = input.required<any>();
  config = input<Config>(defaultConfig);
}
```

### 2. Use model() for Two-Way Binding

```typescript
@Component({...})
export class EditableComponent {
  // Two-way binding with parent
  value = model<string>('');
  selection = model<any[]>([]);
}
```

### 3. Use output() for Events

```typescript
@Component({...})
export class ButtonComponent {
  // Events to parent
  clicked = output<void>();
  submitted = output<FormData>();
}
```

### 4. Leverage Computed with Inputs

```typescript
@Component({...})
export class SmartComponent {
  firstName = input<string>('');
  lastName = input<string>('');
  
  // Derive values from inputs
  fullName = computed(() => 
    `${this.firstName()} ${this.lastName()}`
  );
}
```

### 5. Use Required for Mandatory Data

```typescript
@Component({...})
export class DataComponent {
  // Enforce required inputs at compile time
  id = input.required<string>();
  data = input.required<DataType>();
  
  // Optional with defaults
  pageSize = input<number>(10);
}
```

## Interview Questions

### Q1: What is the difference between input() and model()?

**Answer:** `input()` creates a read-only signal for one-way data binding from parent to child. `model()` creates a writable signal that supports two-way binding with both input and output. Use `model()` when the child component needs to update the parent's value.

### Q2: How do signal inputs improve upon @Input decorators?

**Answer:** Signal inputs provide better type safety, automatic reactivity with computed/effect, simpler syntax for derived values, and elimination of lifecycle methods like `ngOnChanges`. They integrate seamlessly with Angular's signal system.

### Q3: Can you modify an input() signal value inside the component?

**Answer:** No, `input()` creates read-only signals. If you need to modify the value, use `model()` instead, which creates a writable signal with two-way binding support.

### Q4: How do you handle optional vs required inputs with signals?

**Answer:** Use `input.required<T>()` for mandatory inputs (compile-time enforcement) and `input<T>(defaultValue)` for optional inputs with defaults. This provides better type safety than decorator-based inputs.

### Q5: What's the benefit of using output() over EventEmitter?

**Answer:** `output()` provides better type inference, clearer syntax, and consistency with the signal-based API. It's also more tree-shakeable and integrates better with Angular's modern reactive system.

## Key Takeaways

1. input() creates read-only reactive inputs
2. output() creates type-safe event emitters
3. model() provides two-way binding with a single API
4. Use input.required() for mandatory inputs
5. Signal inputs work seamlessly with computed() and effect()
6. Inputs are called as functions in templates
7. Transform functions enable input type conversion
8. Migration from decorators is straightforward
9. Better type safety and developer experience
10. Part of Angular's modern reactive signal system

## Resources

- [Angular Signals Guide](https://angular.dev/guide/signals)
- [input() API Documentation](https://angular.dev/api/core/input)
- [output() API Documentation](https://angular.dev/api/core/output)
- [model() API Documentation](https://angular.dev/api/core/model)
- [Component Interaction with Signals](https://angular.dev/guide/components/inputs)
