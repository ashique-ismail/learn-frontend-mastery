# Input Transforms and Signal Inputs

## Overview

Angular 16+ introduces signal inputs and input transforms, providing type-safe transformations and reactive input handling. Signal inputs integrate seamlessly with Angular's signals reactivity system, offering better performance and change detection optimization.

## Table of Contents

1. [Signal Inputs Basics](#signal-inputs-basics)
2. [input() Function](#input-function)
3. [input.required()](#inputrequired)
4. [Input Transforms](#input-transforms)
5. [Built-in Transforms](#built-in-transforms)
6. [Custom Transforms](#custom-transforms)
7. [Computed from Inputs](#computed-from-inputs)
8. [Migration from @Input](#migration-from-input)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)

## Signal Inputs Basics

Signal inputs provide reactive input properties with automatic change detection.

### Basic Signal Input

```typescript
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ name() }}</h3>
      <p>{{ email() }}</p>
    </div>
  `
})
export class UserCardComponent {
  // Signal input - call as function to read value
  name = input<string>('');
  email = input<string>('');
}

// Usage
// <app-user-card [name]="'Alice'" [email]="'alice@example.com'"></app-user-card>
```

### Signal Input with Default

```typescript
@Component({
  selector: 'app-button',
  standalone: true,
  template: `
    <button [class]="'btn btn-' + variant()">
      <ng-content></ng-content>
    </button>
  `
})
export class ButtonComponent {
  // Default value provided
  variant = input<'primary' | 'secondary'>('primary');
  disabled = input(false);
}
```

### Reading Signal Inputs

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <div>
      <p>Start: {{ start() }}</p>
      <p>Current: {{ count() }}</p>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  start = input(0);
  count = signal(0);
  
  ngOnInit() {
    // Read signal input value
    this.count.set(this.start());
  }
  
  increment() {
    this.count.update(v => v + 1);
  }
}
```

## input() Function

The input() function creates optional signal inputs.

### Optional Signal Inputs

```typescript
interface User {
  id: number;
  name: string;
  avatar?: string;
}

@Component({
  selector: 'app-user-avatar',
  standalone: true,
  imports: [CommonModule],
  template: `
    @if (user(); as u) {
      <div class="avatar">
        @if (u.avatar) {
          <img [src]="u.avatar" [alt]="u.name">
        } @else {
          <div class="placeholder">{{ getInitials(u.name) }}</div>
        }
      </div>
    }
  `
})
export class UserAvatarComponent {
  // Optional input - may be undefined
  user = input<User>();
  
  getInitials(name: string): string {
    return name.split(' ').map(n => n[0]).join('').toUpperCase();
  }
}
```

### Signal Input with Alias

```typescript
@Component({
  selector: 'app-product-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ productName() }}</h3>
      <p>{{ productPrice() }}</p>
    </div>
  `
})
export class ProductCardComponent {
  // External: [name], Internal: productName()
  productName = input('', { alias: 'name' });
  
  // External: [price], Internal: productPrice()
  productPrice = input(0, { alias: 'price' });
}

// Usage
// <app-product-card [name]="'Laptop'" [price]="999"></app-product-card>
```

## input.required()

Create required signal inputs that must be provided.

### Required Signal Inputs

```typescript
interface Product {
  id: number;
  title: string;
  price: number;
}

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div class="product">
      <h2>{{ product().title }}</h2>
      <p class="price">{{ product().price | currency }}</p>
      <p class="id">ID: {{ product().id }}</p>
    </div>
  `
})
export class ProductDetailComponent {
  // Required - compiler error if not provided
  product = input.required<Product>();
}

// ❌ Error: Missing required input 'product'
// <app-product-detail></app-product-detail>

// ✅ Correct
// <app-product-detail [product]="myProduct"></app-product-detail>
```

### Required with Alias

```typescript
@Component({
  selector: 'app-user-profile',
  standalone: true,
  template: `
    <div>
      <h2>{{ userId() }}</h2>
      <p>{{ userEmail() }}</p>
    </div>
  `
})
export class UserProfileComponent {
  userId = input.required<number>({ alias: 'id' });
  userEmail = input.required<string>({ alias: 'email' });
}

// Usage
// <app-user-profile [id]="123" [email]="'user@example.com'"></app-user-profile>
```

### Required with Transform

```typescript
@Component({
  selector: 'app-pagination',
  standalone: true,
  template: `
    <div>Page {{ currentPage() }} of {{ totalPages() }}</div>
  `
})
export class PaginationComponent {
  currentPage = input.required<number>({
    transform: (value: string | number) => Number(value)
  });
  
  totalPages = input.required<number>({
    transform: (value: string | number) => Number(value)
  });
}

// Accepts both string and number
// <app-pagination currentPage="5" [totalPages]="10"></app-pagination>
```

## Input Transforms

Transform input values before they're set in the component.

### Transform Function Signature

```typescript
type TransformFn<T, U> = (value: T) => U;

@Component({
  selector: 'app-example',
  standalone: true,
  template: `<div>{{ value() }}</div>`
})
export class ExampleComponent {
  value = input('', {
    transform: (value: string) => value.toUpperCase()
  });
}
```

### String Transforms

```typescript
function trimTransform(value: string | undefined): string {
  return value?.trim() ?? '';
}

function lowercaseTransform(value: string): string {
  return value.toLowerCase();
}

function capitalizeTransform(value: string): string {
  return value.charAt(0).toUpperCase() + value.slice(1).toLowerCase();
}

@Component({
  selector: 'app-text-input',
  standalone: true,
  template: `
    <div>
      <p>Trimmed: {{ trimmedValue() }}</p>
      <p>Lowercase: {{ lowercaseValue() }}</p>
      <p>Capitalized: {{ capitalizedValue() }}</p>
    </div>
  `
})
export class TextInputComponent {
  trimmedValue = input('', { transform: trimTransform });
  lowercaseValue = input('', { transform: lowercaseTransform });
  capitalizedValue = input('', { transform: capitalizeTransform });
}
```

### Number Transforms

```typescript
function parseNumberTransform(value: string | number): number {
  return typeof value === 'number' ? value : parseFloat(value) || 0;
}

function clampTransform(min: number, max: number) {
  return (value: number): number => {
    return Math.max(min, Math.min(max, value));
  };
}

@Component({
  selector: 'app-range-slider',
  standalone: true,
  template: `
    <div>
      <input
        type="range"
        [value]="value()"
        (input)="onValueChange($event)"
        [min]="min()"
        [max]="max()">
      <span>{{ value() }}</span>
    </div>
  `
})
export class RangeSliderComponent {
  min = input(0, { transform: parseNumberTransform });
  max = input(100, { transform: parseNumberTransform });
  value = input(50, {
    transform: (v: string | number) => {
      const num = parseNumberTransform(v);
      return Math.max(this.min(), Math.min(this.max(), num));
    }
  });
  
  onValueChange(event: Event) {
    const input = event.target as HTMLInputElement;
    console.log('Value:', input.value);
  }
}
```

## Built-in Transforms

Angular provides built-in transforms for common conversions.

### booleanAttribute

```typescript
import { Component, input, booleanAttribute } from '@angular/core';

@Component({
  selector: 'app-collapsible',
  standalone: true,
  template: `
    <div [class.expanded]="expanded()">
      <div class="header" (click)="toggle()">
        {{ title() }}
      </div>
      @if (expanded()) {
        <div class="content">
          <ng-content></ng-content>
        </div>
      }
    </div>
  `
})
export class CollapsibleComponent {
  title = input('Click to expand');
  
  // Transforms "", "true", "false" to boolean
  expanded = input(false, { transform: booleanAttribute });
  
  expandedSignal = signal(false);
  
  ngOnInit() {
    this.expandedSignal.set(this.expanded());
  }
  
  toggle() {
    this.expandedSignal.update(v => !v);
  }
}

// All these work:
// <app-collapsible expanded></app-collapsible>
// <app-collapsible [expanded]="true"></app-collapsible>
// <app-collapsible expanded="true"></app-collapsible>
// <app-collapsible expanded="false"></app-collapsible>
```

### numberAttribute

```typescript
import { Component, input, numberAttribute } from '@angular/core';

@Component({
  selector: 'app-star-rating',
  standalone: true,
  template: `
    <div class="stars">
      @for (star of stars(); track $index) {
        <span [class.filled]="$index < rating()">★</span>
      }
    </div>
  `
})
export class StarRatingComponent {
  // Converts string "5" to number 5
  rating = input(0, { transform: numberAttribute });
  max = input(5, { transform: numberAttribute });
  
  stars = computed(() => {
    return Array(this.max()).fill(0);
  });
}

// Both work:
// <app-star-rating rating="4" max="5"></app-star-rating>
// <app-star-rating [rating]="4" [max]="5"></app-star-rating>
```

### Combining Transforms

```typescript
function parseAndClampNumber(min: number, max: number) {
  return (value: string | number): number => {
    const num = typeof value === 'number' ? value : parseFloat(value) || min;
    return Math.max(min, Math.min(max, num));
  };
}

@Component({
  selector: 'app-percentage-input',
  standalone: true,
  template: `
    <div>
      <input
        type="number"
        [value]="percentage()"
        min="0"
        max="100">
      <span>{{ percentage() }}%</span>
    </div>
  `
})
export class PercentageInputComponent {
  percentage = input(0, {
    transform: parseAndClampNumber(0, 100)
  });
}
```

## Custom Transforms

Create reusable transform functions for complex conversions.

### Object Transform

```typescript
interface UserInput {
  id: number | string;
  name: string;
}

interface User {
  id: number;
  name: string;
  displayName: string;
}

function userTransform(value: UserInput): User {
  return {
    id: typeof value.id === 'string' ? parseInt(value.id) : value.id,
    name: value.name,
    displayName: value.name.toUpperCase()
  };
}

@Component({
  selector: 'app-user-display',
  standalone: true,
  template: `
    <div>
      <p>ID: {{ user().id }}</p>
      <p>Name: {{ user().name }}</p>
      <p>Display: {{ user().displayName }}</p>
    </div>
  `
})
export class UserDisplayComponent {
  user = input.required<User>({ transform: userTransform });
}
```

### Array Transform

```typescript
function splitStringTransform(delimiter = ','): (value: string | string[]) => string[] {
  return (value) => {
    if (Array.isArray(value)) return value;
    return value.split(delimiter).map(s => s.trim()).filter(Boolean);
  };
}

@Component({
  selector: 'app-tag-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="tags">
      @for (tag of tags(); track tag) {
        <span class="tag">{{ tag }}</span>
      }
    </div>
  `
})
export class TagListComponent {
  tags = input<string[]>([], {
    transform: splitStringTransform(',')
  });
}

// Both work:
// <app-tag-list tags="angular,react,vue"></app-tag-list>
// <app-tag-list [tags]="['angular', 'react', 'vue']"></app-tag-list>
```

### Date Transform

```typescript
function dateTransform(value: string | Date | number): Date {
  if (value instanceof Date) return value;
  if (typeof value === 'number') return new Date(value);
  return new Date(value);
}

@Component({
  selector: 'app-date-display',
  standalone: true,
  imports: [DatePipe],
  template: `
    <div>
      <p>Date: {{ date() | date:'medium' }}</p>
      <p>Timestamp: {{ date().getTime() }}</p>
    </div>
  `
})
export class DateDisplayComponent {
  date = input(new Date(), { transform: dateTransform });
}

// All work:
// <app-date-display [date]="'2024-01-01'"></app-date-display>
// <app-date-display [date]="1704067200000"></app-date-display>
// <app-date-display [date]="myDateObject"></app-date-display>
```

## Computed from Inputs

Create computed signals derived from input signals.

### Basic Computed

```typescript
@Component({
  selector: 'app-full-name',
  standalone: true,
  template: `
    <div>
      <p>First: {{ firstName() }}</p>
      <p>Last: {{ lastName() }}</p>
      <p>Full: {{ fullName() }}</p>
    </div>
  `
})
export class FullNameComponent {
  firstName = input('');
  lastName = input('');
  
  // Computed from inputs
  fullName = computed(() => {
    return `${this.firstName()} ${this.lastName()}`.trim();
  });
}
```

### Complex Computed

```typescript
interface Product {
  price: number;
  discount: number;
}

@Component({
  selector: 'app-price-calculator',
  standalone: true,
  imports: [CurrencyPipe],
  template: `
    <div class="price-card">
      <p>Original: {{ product().price | currency }}</p>
      <p>Discount: {{ product().discount }}%</p>
      <p>Final: {{ finalPrice() | currency }}</p>
      <p>Savings: {{ savings() | currency }}</p>
    </div>
  `
})
export class PriceCalculatorComponent {
  product = input.required<Product>();
  
  finalPrice = computed(() => {
    const p = this.product();
    return p.price * (1 - p.discount / 100);
  });
  
  savings = computed(() => {
    return this.product().price - this.finalPrice();
  });
}
```

### Computed with Effect

```typescript
@Component({
  selector: 'app-search',
  standalone: true,
  imports: [FormsModule],
  template: `
    <div>
      <input [(ngModel)]="searchTerm" placeholder="Search...">
      <p>Results: {{ filteredItems().length }}</p>
      <div>
        @for (item of filteredItems(); track item.id) {
          <div>{{ item.name }}</div>
        }
      </div>
    </div>
  `
})
export class SearchComponent {
  items = input<Array<{id: number, name: string}>>([]);
  searchTerm = signal('');
  
  filteredItems = computed(() => {
    const term = this.searchTerm().toLowerCase();
    if (!term) return this.items();
    
    return this.items().filter(item =>
      item.name.toLowerCase().includes(term)
    );
  });
  
  constructor() {
    effect(() => {
      console.log('Filtered count:', this.filteredItems().length);
    });
  }
}
```

## Migration from @Input

Convert traditional @Input to signal inputs.

### Before: @Input

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ userName }}</h3>
      <p>{{ userEmail }}</p>
    </div>
  `
})
export class UserCardComponent implements OnChanges {
  @Input() userName = '';
  @Input() userEmail = '';
  
  ngOnChanges(changes: SimpleChanges) {
    if (changes['userName']) {
      console.log('Name changed:', changes['userName'].currentValue);
    }
  }
}
```

### After: Signal Input

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ userName() }}</h3>
      <p>{{ userEmail() }}</p>
    </div>
  `
})
export class UserCardComponent {
  userName = input('');
  userEmail = input('');
  
  constructor() {
    effect(() => {
      console.log('Name changed:', this.userName());
    });
  }
}
```

### Migration Steps

```typescript
// Step 1: Replace @Input with input()
// Before
@Input() value = '';

// After
value = input('');

// Step 2: Update template to call as function
// Before: {{ value }}
// After: {{ value() }}

// Step 3: Update component logic
// Before
someMethod() {
  console.log(this.value);
}

// After
someMethod() {
  console.log(this.value());
}

// Step 4: Replace ngOnChanges with effect
// Before
ngOnChanges(changes: SimpleChanges) {
  if (changes['value']) {
    this.process(changes['value'].currentValue);
  }
}

// After
constructor() {
  effect(() => {
    this.process(this.value());
  });
}
```

## Common Mistakes

### 1. Forgetting Function Call

```typescript
// ❌ WRONG: Not calling signal input
@Component({
  template: `<div>{{ name }}</div>` // Missing ()
})
export class Component {
  name = input('');
}

// ✅ CORRECT: Call as function
@Component({
  template: `<div>{{ name() }}</div>`
})
export class Component {
  name = input('');
}
```

### 2. Trying to Set Signal Input

```typescript
// ❌ WRONG: Can't set signal inputs
export class Component {
  value = input(0);
  
  increment() {
    this.value.set(this.value() + 1); // Error!
  }
}

// ✅ CORRECT: Use internal signal
export class Component {
  initialValue = input(0);
  value = signal(0);
  
  ngOnInit() {
    this.value.set(this.initialValue());
  }
  
  increment() {
    this.value.update(v => v + 1);
  }
}
```

### 3. Transform Type Mismatch

```typescript
// ❌ WRONG: Return type doesn't match
value = input<number>(0, {
  transform: (v: string) => v.toUpperCase() // Returns string!
});

// ✅ CORRECT: Consistent types
value = input<number>(0, {
  transform: (v: string | number) => Number(v)
});
```

## Best Practices

### 1. Use signal inputs for new code

```typescript
// ✅ Prefer signal inputs
name = input('');
count = input(0);
user = input.required<User>();
```

### 2. Use computed for derived values

```typescript
// ✅ Computed from inputs
firstName = input('');
lastName = input('');
fullName = computed(() => `${this.firstName()} ${this.lastName()}`);
```

### 3. Use transforms for type safety

```typescript
// ✅ Transform for flexibility
age = input(0, { transform: numberAttribute });
active = input(false, { transform: booleanAttribute });
```

### 4. Required for mandatory inputs

```typescript
// ✅ Required for critical data
userId = input.required<number>();
productData = input.required<Product>();
```

## Interview Questions

### Q1: What are signal inputs and how do they differ from @Input?

**Answer:** Signal inputs use the `input()` function and return signals that must be called as functions. They integrate with Angular's reactivity system, provide better type safety, support transforms, and don't need ngOnChanges for tracking changes. Use `effect()` instead.

### Q2: What is input.required() and when should you use it?

**Answer:** `input.required()` creates a signal input that must be provided, causing a compiler error if missing. Use for mandatory data that components can't function without: `userId = input.required<number>()`.

### Q3: How do input transforms work?

**Answer:** Transforms convert input values before setting them: `input(default, { transform: fn })`. The transform function receives the bound value and returns the transformed value. Built-in transforms include `booleanAttribute` and `numberAttribute`.

### Q4: How do you create computed values from signal inputs?

**Answer:** Use `computed()` to derive values: `fullName = computed(() => \`${this.firstName()} ${this.lastName()}\`)`. Computed signals automatically update when dependencies change.

### Q5: When should you use signal inputs vs traditional @Input?

**Answer:** Use signal inputs for new code as they're the future of Angular. Use @Input for existing code until migration. Signal inputs offer better performance, type safety, and integration with signals reactivity.

## Key Takeaways

1. Signal inputs created with `input()` and `input.required()`
2. Call signal inputs as functions: `value()`
3. Transforms convert values before setting
4. Built-in transforms: booleanAttribute, numberAttribute
5. Use computed() for derived values from inputs
6. effect() replaces ngOnChanges for tracking changes
7. Signal inputs integrate with signals reactivity
8. Required inputs enforce compile-time checks
9. Transforms provide type flexibility
10. Signal inputs are the future of Angular

## Resources

- [Angular Signals Guide](https://angular.io/guide/signals)
- [Signal Inputs](https://angular.io/guide/signal-inputs)
- [Input Transforms](https://angular.io/guide/inputs-outputs#input-transforms)
- [Computed Signals](https://angular.io/guide/signals#computed-signals)
- [Effects](https://angular.io/guide/signals#effects)
