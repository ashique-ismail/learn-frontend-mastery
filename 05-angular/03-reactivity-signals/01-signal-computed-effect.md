# Signal, Computed, and Effect in Angular

## The Idea

**In plain English:** Signals are a way for your app to remember a value (like a counter or a name) and automatically update anything that depends on it the moment that value changes — no manual refreshing needed. Think of a signal as a special box that shouts "hey, I changed!" to anyone listening whenever you put something new inside.

**Real-world analogy:** Imagine a whiteboard in a classroom that shows the current score of a game. A scorekeeper updates the number on the whiteboard (the signal). Two students in the room have assigned jobs: one always calculates the average score so far and rewrites it on a sticky note (computed), and another student is in charge of ringing a bell whenever the score changes so people pay attention (effect).

- The whiteboard number = the signal (the raw piece of state that can be updated)
- The sticky note with the running average = the computed value (automatically recalculated whenever the whiteboard changes, read-only)
- The student ringing the bell = the effect (runs a side action in response to the change)

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding Signals](#understanding-signals)
- [signal() Function](#signal-function)
- [computed() Function](#computed-function)
- [effect() Function](#effect-function)
- [Glitch-Free Updates](#glitch-free-updates)
- [Signal vs Observable](#signal-vs-observable)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Angular Signals represent a fundamental shift in how Angular handles reactivity. Introduced in Angular 16 and becoming the default in Angular 19+, signals provide a fine-grained reactivity system that enables better performance, simpler code, and a clearer mental model for state management.

Signals are reactive primitives that notify consumers when their value changes. The three core building blocks are:

- **signal()**: Writable signal for mutable state
- **computed()**: Read-only derived state
- **effect()**: Side effects that run when signals change

## Understanding Signals

### What is a Signal?

A signal is a wrapper around a value that notifies interested consumers when that value changes.

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <button (click)="increment()">Increment</button>
    </div>
  `
})
export class CounterComponent {
  // Create a signal with initial value
  count = signal(0);
  
  increment(): void {
    // Update signal value
    this.count.set(this.count() + 1);
  }
}
```

### Signal Characteristics

```typescript
import { signal } from '@angular/core';

// 1. Signals are synchronous
const count = signal(0);
console.log(count()); // 0
count.set(5);
console.log(count()); // 5 immediately

// 2. Signals always have a value
const name = signal('Angular'); // Never undefined
console.log(name()); // 'Angular'

// 3. Signals track dependencies automatically
const firstName = signal('John');
const lastName = signal('Doe');
const fullName = computed(() => `${firstName()} ${lastName()}`);
// fullName automatically updates when firstName or lastName change

// 4. Signals are lazy
const expensive = computed(() => {
  console.log('Computing...');
  return heavyCalculation();
});
// 'Computing...' only logs when expensive() is called
```

### Why Signals?

```typescript
// Before: Zone.js triggers change detection for entire component tree
@Component({
  selector: 'app-old',
  template: `
    <div>{{ user.name }}</div>
    <div>{{ user.email }}</div>
    <div>{{ user.address }}</div>
  `
})
export class OldComponent {
  user = {
    name: 'John',
    email: 'john@example.com',
    address: '123 Main St'
  };
  
  updateName(): void {
    this.user.name = 'Jane';
    // Zone.js triggers change detection for entire component
    // All bindings are checked even if unchanged
  }
}

// After: Signals enable fine-grained reactivity
@Component({
  selector: 'app-new',
  template: `
    <div>{{ user.name() }}</div>
    <div>{{ user.email() }}</div>
    <div>{{ user.address() }}</div>
  `
})
export class NewComponent {
  user = {
    name: signal('John'),
    email: signal('john@example.com'),
    address: signal('123 Main St')
  };
  
  updateName(): void {
    this.user.name.set('Jane');
    // Only the name binding updates
    // Email and address bindings are not checked
  }
}
```

## signal() Function

### Creating Signals

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-signals',
  template: `
    <div>
      <p>Primitive: {{ count() }}</p>
      <p>Object: {{ user().name }}</p>
      <p>Array: {{ items().length }} items</p>
    </div>
  `
})
export class SignalsComponent {
  // Primitive signal
  count = signal(0);
  
  // Object signal
  user = signal({ name: 'John', age: 30 });
  
  // Array signal
  items = signal<string[]>(['apple', 'banana', 'orange']);
  
  // Nullable signal
  selectedItem = signal<string | null>(null);
  
  // Complex type signal
  data = signal<{ id: number; values: number[] }>({
    id: 1,
    values: [1, 2, 3]
  });
}
```

### Reading Signal Values

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-read',
  template: '<p>{{ message() }}</p>'
})
export class ReadComponent {
  message = signal('Hello');
  
  ngOnInit(): void {
    // Call signal as a function to read value
    console.log(this.message()); // 'Hello'
    
    // Reading establishes a dependency in reactive contexts
    effect(() => {
      console.log('Message changed:', this.message());
      // This effect re-runs whenever message changes
    });
  }
}
```

### Updating Signal Values

```typescript
import { Component, signal } from '@angular/core';

interface User {
  name: string;
  age: number;
}

@Component({
  selector: 'app-update',
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <p>User: {{ user().name }}, {{ user().age }}</p>
      <button (click)="updateExamples()">Update</button>
    </div>
  `
})
export class UpdateComponent {
  count = signal(0);
  user = signal<User>({ name: 'John', age: 30 });
  items = signal<number[]>([1, 2, 3]);
  
  updateExamples(): void {
    // 1. set() - Replace entire value
    this.count.set(10);
    
    // 2. update() - Update based on current value
    this.count.update(current => current + 1);
    
    // 3. Immutable object update
    this.user.update(current => ({
      ...current,
      age: current.age + 1
    }));
    
    // 4. Immutable array update
    this.items.update(current => [...current, 4]);
    
    // 5. Conditional update
    this.count.update(current => {
      return current < 100 ? current + 1 : current;
    });
  }
  
  // Helper methods for clarity
  incrementCount(): void {
    this.count.update(c => c + 1);
  }
  
  resetCount(): void {
    this.count.set(0);
  }
  
  updateUserName(name: string): void {
    this.user.update(u => ({ ...u, name }));
  }
  
  addItem(item: number): void {
    this.items.update(arr => [...arr, item]);
  }
}
```

### Signal with Complex State

```typescript
import { Component, signal } from '@angular/core';

interface TodoItem {
  id: number;
  text: string;
  completed: boolean;
}

@Component({
  selector: 'app-todo',
  template: `
    <div>
      <input #input (keyup.enter)="addTodo(input.value); input.value = ''">
      <ul>
        <li *ngFor="let todo of todos()">
          <input 
            type="checkbox" 
            [checked]="todo.completed"
            (change)="toggleTodo(todo.id)">
          <span [class.completed]="todo.completed">{{ todo.text }}</span>
          <button (click)="removeTodo(todo.id)">Remove</button>
        </li>
      </ul>
      <p>{{ completedCount() }} / {{ todos().length }} completed</p>
    </div>
  `
})
export class TodoComponent {
  todos = signal<TodoItem[]>([]);
  nextId = signal(1);
  
  // Computed signal for completed count
  completedCount = computed(() => 
    this.todos().filter(t => t.completed).length
  );
  
  addTodo(text: string): void {
    if (!text.trim()) return;
    
    this.todos.update(todos => [
      ...todos,
      {
        id: this.nextId(),
        text: text.trim(),
        completed: false
      }
    ]);
    
    this.nextId.update(id => id + 1);
  }
  
  toggleTodo(id: number): void {
    this.todos.update(todos =>
      todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  }
  
  removeTodo(id: number): void {
    this.todos.update(todos => todos.filter(t => t.id !== id));
  }
}
```

## computed() Function

### Basic Computed Signals

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-computed',
  template: `
    <div>
      <input type="number" [(ngModel)]="quantity" (input)="quantitySignal.set(+$any($event.target).value)">
      <input type="number" [(ngModel)]="price" (input)="priceSignal.set(+$any($event.target).value)">
      <p>Quantity: {{ quantitySignal() }}</p>
      <p>Price: ${{ priceSignal() }}</p>
      <p>Total: ${{ total() }}</p>
      <p>Tax: ${{ tax() }}</p>
      <p>Grand Total: ${{ grandTotal() }}</p>
    </div>
  `
})
export class ComputedComponent {
  quantitySignal = signal(1);
  priceSignal = signal(10);
  quantity = 1;
  price = 10;
  
  // Computed signals automatically track dependencies
  total = computed(() => this.quantitySignal() * this.priceSignal());
  
  tax = computed(() => this.total() * 0.1);
  
  grandTotal = computed(() => this.total() + this.tax());
}
```

### Computed with Complex Logic

```typescript
import { Component, signal, computed } from '@angular/core';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

@Component({
  selector: 'app-product-list',
  template: `
    <div>
      <input [(ngModel)]="searchTermValue" (input)="searchTerm.set($any($event.target).value)">
      <select [(ngModel)]="categoryValue" (change)="selectedCategory.set($any($event.target).value)">
        <option value="">All Categories</option>
        <option *ngFor="let cat of categories()" [value]="cat">{{ cat }}</option>
      </select>
      
      <div *ngFor="let product of filteredProducts()">
        {{ product.name }} - ${{ product.price }}
      </div>
      
      <p>Showing {{ filteredProducts().length }} of {{ products().length }} products</p>
      <p>Average price: ${{ averagePrice() }}</p>
    </div>
  `
})
export class ProductListComponent {
  searchTermValue = '';
  categoryValue = '';
  
  products = signal<Product[]>([
    { id: 1, name: 'Laptop', price: 999, category: 'Electronics' },
    { id: 2, name: 'Phone', price: 699, category: 'Electronics' },
    { id: 3, name: 'Desk', price: 299, category: 'Furniture' },
    { id: 4, name: 'Chair', price: 199, category: 'Furniture' }
  ]);
  
  searchTerm = signal('');
  selectedCategory = signal('');
  
  // Computed: unique categories
  categories = computed(() => {
    const cats = this.products().map(p => p.category);
    return Array.from(new Set(cats));
  });
  
  // Computed: filtered products
  filteredProducts = computed(() => {
    let filtered = this.products();
    
    const search = this.searchTerm().toLowerCase();
    if (search) {
      filtered = filtered.filter(p =>
        p.name.toLowerCase().includes(search)
      );
    }
    
    const category = this.selectedCategory();
    if (category) {
      filtered = filtered.filter(p => p.category === category);
    }
    
    return filtered;
  });
  
  // Computed: average price of filtered products
  averagePrice = computed(() => {
    const products = this.filteredProducts();
    if (products.length === 0) return 0;
    
    const sum = products.reduce((acc, p) => acc + p.price, 0);
    return (sum / products.length).toFixed(2);
  });
}
```

### Chained Computed Signals

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-chained',
  template: `
    <div>
      <p>Width: <input type="number" (input)="width.set(+$any($event.target).value)"></p>
      <p>Height: <input type="number" (input)="height.set(+$any($event.target).value)"></p>
      <p>Area: {{ area() }}</p>
      <p>Perimeter: {{ perimeter() }}</p>
      <p>Diagonal: {{ diagonal() }}</p>
      <p>Is Square: {{ isSquare() }}</p>
    </div>
  `
})
export class ChainedComponent {
  width = signal(10);
  height = signal(10);
  
  // First level computed
  area = computed(() => this.width() * this.height());
  perimeter = computed(() => 2 * (this.width() + this.height()));
  
  // Second level computed (depends on other computed)
  diagonal = computed(() => {
    const w = this.width();
    const h = this.height();
    return Math.sqrt(w * w + h * h).toFixed(2);
  });
  
  // Computed with conditional logic
  isSquare = computed(() => this.width() === this.height());
  
  // Computed with multiple dependencies
  description = computed(() => {
    const w = this.width();
    const h = this.height();
    const a = this.area();
    const square = this.isSquare();
    
    return `${square ? 'Square' : 'Rectangle'} with dimensions ${w}x${h} and area ${a}`;
  });
}
```

### Computed Performance Optimization

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-optimized',
  template: `
    <div>
      <button (click)="addItem()">Add Item</button>
      <p>Items: {{ items().length }}</p>
      <p>Expensive result: {{ expensiveComputation() }}</p>
    </div>
  `
})
export class OptimizedComponent {
  items = signal<number[]>([1, 2, 3]);
  
  // Computed signals are memoized
  // Only recalculated when dependencies change
  expensiveComputation = computed(() => {
    console.log('Computing...');
    const items = this.items();
    
    // Expensive calculation
    return items.reduce((sum, item) => {
      for (let i = 0; i < 1000000; i++) {
        sum += Math.random() * item;
      }
      return sum;
    }, 0);
  });
  
  addItem(): void {
    this.items.update(items => [...items, items.length + 1]);
    // expensiveComputation automatically recalculates
  }
  
  // This method doesn't trigger recomputation
  doUnrelatedWork(): void {
    console.log('Doing unrelated work');
    // expensiveComputation is NOT recalculated
  }
}
```

## effect() Function

### Basic Effects

```typescript
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'app-effect',
  template: `
    <div>
      <button (click)="increment()">Count: {{ count() }}</button>
    </div>
  `
})
export class EffectComponent {
  count = signal(0);
  
  constructor() {
    // Effect runs immediately and whenever count changes
    effect(() => {
      console.log('Count is now:', this.count());
    });
    
    // Effect with multiple dependencies
    effect(() => {
      const current = this.count();
      if (current > 10) {
        console.log('Count exceeded 10!');
      }
    });
  }
  
  increment(): void {
    this.count.update(c => c + 1);
    // Both effects run automatically
  }
}
```

### Effect with Cleanup

```typescript
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'app-cleanup',
  template: `
    <div>
      <input (input)="searchTerm.set($any($event.target).value)">
      <p>Searching for: {{ searchTerm() }}</p>
    </div>
  `
})
export class CleanupComponent {
  searchTerm = signal('');
  
  constructor() {
    effect((onCleanup) => {
      const term = this.searchTerm();
      
      if (!term) return;
      
      console.log('Starting search for:', term);
      
      // Simulate async operation
      const timeoutId = setTimeout(() => {
        console.log('Search completed for:', term);
      }, 1000);
      
      // Cleanup function runs before next effect execution
      onCleanup(() => {
        console.log('Cancelling search for:', term);
        clearTimeout(timeoutId);
      });
    });
  }
}
```

### Effect for Side Effects

```typescript
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'app-side-effects',
  template: `
    <div>
      <h2>{{ title() }}</h2>
      <button (click)="toggleTheme()">Toggle Theme</button>
      <button (click)="changeLanguage()">Change Language</button>
    </div>
  `
})
export class SideEffectsComponent {
  title = signal('My App');
  theme = signal<'light' | 'dark'>('light');
  language = signal('en');
  
  constructor() {
    // Update document title
    effect(() => {
      document.title = this.title();
    });
    
    // Update theme class on body
    effect(() => {
      document.body.className = `theme-${this.theme()}`;
    });
    
    // Update HTML lang attribute
    effect(() => {
      document.documentElement.lang = this.language();
    });
    
    // Local storage sync
    effect(() => {
      const preferences = {
        theme: this.theme(),
        language: this.language()
      };
      localStorage.setItem('preferences', JSON.stringify(preferences));
    });
    
    // Analytics tracking
    effect(() => {
      console.log('Page view:', {
        title: this.title(),
        theme: this.theme(),
        language: this.language()
      });
    });
  }
  
  toggleTheme(): void {
    this.theme.update(t => t === 'light' ? 'dark' : 'light');
  }
  
  changeLanguage(): void {
    this.language.update(l => l === 'en' ? 'es' : 'en');
  }
}
```

### Conditional Effects

```typescript
import { Component, signal, effect, computed } from '@angular/core';

@Component({
  selector: 'app-conditional',
  template: `
    <div>
      <input type="checkbox" [(ngModel)]="enabledValue" (change)="enabled.set(enabledValue)">
      Enable Feature
      <button (click)="increment()" [disabled]="!enabled()">
        Count: {{ count() }}
      </button>
    </div>
  `
})
export class ConditionalComponent {
  enabled = signal(true);
  count = signal(0);
  enabledValue = true;
  
  constructor() {
    // Effect only runs when enabled
    effect(() => {
      if (!this.enabled()) return;
      
      const current = this.count();
      console.log('Enabled count:', current);
      
      if (current >= 5) {
        alert('Count reached 5!');
      }
    });
  }
  
  increment(): void {
    this.count.update(c => c + 1);
  }
}
```

### Effects with Injected Services

```typescript
import { Component, signal, effect, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-service-effect',
  template: `
    <div>
      <input (input)="userId.set($any($event.target).value)">
      <p *ngIf="loading()">Loading...</p>
      <p *ngIf="user()">User: {{ user()?.name }}</p>
    </div>
  `
})
export class ServiceEffectComponent {
  private http = inject(HttpClient);
  
  userId = signal('1');
  user = signal<any>(null);
  loading = signal(false);
  
  constructor() {
    effect((onCleanup) => {
      const id = this.userId();
      if (!id) return;
      
      this.loading.set(true);
      
      const subscription = this.http.get(`/api/users/${id}`)
        .subscribe({
          next: user => {
            this.user.set(user);
            this.loading.set(false);
          },
          error: err => {
            console.error(err);
            this.loading.set(false);
          }
        });
      
      onCleanup(() => subscription.unsubscribe());
    });
  }
}
```

## Glitch-Free Updates

### Understanding Glitches

```typescript
// Problem: Without glitch-free updates
class WithoutSignals {
  firstName = 'John';
  lastName = 'Doe';
  
  // These are independent
  get fullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }
  
  get greeting(): string {
    return `Hello, ${this.fullName}`;
  }
  
  updateName(): void {
    this.firstName = 'Jane'; // Change detection runs
    // At this point, fullName = 'Jane Doe', greeting = 'Hello, John Doe'
    // GLITCH: Temporary inconsistent state!
    
    this.lastName = 'Smith'; // Change detection runs again
    // Now fullName = 'Jane Smith', greeting = 'Hello, Jane Doe'
    // GLITCH: Still inconsistent!
    
    // After both updates, finally consistent
  }
}

// Solution: Signals provide glitch-free updates
@Component({
  selector: 'app-glitch-free',
  template: `
    <div>
      <p>Full Name: {{ fullName() }}</p>
      <p>Greeting: {{ greeting() }}</p>
      <button (click)="updateName()">Update Name</button>
    </div>
  `
})
export class GlitchFreeComponent {
  firstName = signal('John');
  lastName = signal('Doe');
  
  // Computed signals guarantee consistency
  fullName = computed(() => `${this.firstName()} ${this.lastName()}`);
  greeting = computed(() => `Hello, ${this.fullName()}`);
  
  updateName(): void {
    this.firstName.set('Jane');
    this.lastName.set('Smith');
    // Angular batches updates
    // fullName and greeting computed only once with latest values
    // NO GLITCHES: Always consistent!
  }
}
```

### Diamond Dependency Problem

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-diamond',
  template: `
    <div>
      <p>Width: {{ width() }}</p>
      <p>Height: {{ height() }}</p>
      <p>Area: {{ area() }}</p>
      <p>Perimeter: {{ perimeter() }}</p>
      <p>Ratio: {{ ratio() }}</p>
    </div>
  `
})
export class DiamondComponent {
  width = signal(10);
  height = signal(5);
  
  // Two paths from width/height to ratio
  //      width    height
  //        ↓        ↓
  //      area    perimeter
  //         ↘    ↙
  //          ratio
  
  area = computed(() => {
    console.log('Computing area');
    return this.width() * this.height();
  });
  
  perimeter = computed(() => {
    console.log('Computing perimeter');
    return 2 * (this.width() + this.height());
  });
  
  ratio = computed(() => {
    console.log('Computing ratio');
    // Depends on both area and perimeter
    return this.area() / this.perimeter();
  });
  
  updateDimensions(): void {
    this.width.set(20);
    this.height.set(10);
    
    // Without glitch-free updates:
    // - area computes with width=20, height=5 (INCONSISTENT)
    // - perimeter computes with width=20, height=10
    // - ratio uses inconsistent area value
    
    // With signals (glitch-free):
    // - Updates batched
    // - area computes with width=20, height=10
    // - perimeter computes with width=20, height=10
    // - ratio computed only once with consistent values
    // Console logs appear only once per computed signal
  }
}
```

### Batched Updates

```typescript
import { Component, signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-batched',
  template: `
    <div>
      <button (click)="updateAll()">Update All</button>
      <p>First: {{ first() }}</p>
      <p>Second: {{ second() }}</p>
      <p>Combined: {{ combined() }}</p>
    </div>
  `
})
export class BatchedComponent {
  first = signal(0);
  second = signal(0);
  
  combined = computed(() => {
    console.log('Combined computed');
    return this.first() + this.second();
  });
  
  constructor() {
    effect(() => {
      console.log('Effect ran', this.combined());
    });
  }
  
  updateAll(): void {
    // These updates are batched
    this.first.set(1);
    this.second.set(2);
    this.first.set(3);
    this.second.set(4);
    
    // combined() computed only once with final values (3 + 4)
    // Effect runs only once
    // Console logs: 'Combined computed', 'Effect ran 7'
  }
}
```

## Signal vs Observable

### Comparison

```typescript
import { Component, signal, computed } from '@angular/core';
import { BehaviorSubject, combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-comparison',
  template: `
    <div>
      <h3>Signals</h3>
      <p>Count: {{ countSignal() }}</p>
      <p>Doubled: {{ doubledSignal() }}</p>
      
      <h3>Observables</h3>
      <p>Count: {{ countObservable$ | async }}</p>
      <p>Doubled: {{ doubledObservable$ | async }}</p>
      
      <button (click)="increment()">Increment</button>
    </div>
  `
})
export class ComparisonComponent {
  // Signals approach
  countSignal = signal(0);
  doubledSignal = computed(() => this.countSignal() * 2);
  
  // Observables approach
  countObservable$ = new BehaviorSubject(0);
  doubledObservable$ = this.countObservable$.pipe(
    map(count => count * 2)
  );
  
  increment(): void {
    // Signals: Synchronous, simpler syntax
    this.countSignal.update(c => c + 1);
    console.log('Signal:', this.countSignal()); // Immediate value
    
    // Observables: Asynchronous, requires subscription
    const current = this.countObservable$.value;
    this.countObservable$.next(current + 1);
    // Must subscribe to get value
    this.countObservable$.subscribe(v => console.log('Observable:', v));
  }
}

/*
Key Differences:

1. Synchronous vs Asynchronous
   - Signals: Always synchronous, immediate values
   - Observables: Asynchronous by default, lazy execution

2. Syntax
   - Signals: count() to read, count.set() to write
   - Observables: Must subscribe, use async pipe in templates

3. Memory Management
   - Signals: Automatic cleanup
   - Observables: Manual unsubscribe required (except async pipe)

4. Performance
   - Signals: Fine-grained reactivity, no zone.js
   - Observables: Zone.js triggers change detection

5. Glitch-Free
   - Signals: Guaranteed glitch-free updates
   - Observables: Potential timing issues with multiple sources

6. Use Cases
   - Signals: Synchronous state, UI state, derived values
   - Observables: Async operations, HTTP, event streams, complex composition
*/
```

### When to Use Each

```typescript
import { Component, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-when-to-use',
  template: `
    <div>
      <!-- Use signals for synchronous state -->
      <p>Count: {{ count() }}</p>
      <p>Is Even: {{ isEven() }}</p>
      
      <!-- Use observables for async operations -->
      <p>Users: {{ users$ | async | json }}</p>
    </div>
  `
})
export class WhenToUseComponent {
  private http = inject(HttpClient);
  
  // Use signals for:
  // 1. Component state
  count = signal(0);
  isEven = computed(() => this.count() % 2 === 0);
  
  // 2. Form state
  formData = signal({
    name: '',
    email: ''
  });
  
  // 3. UI state
  isMenuOpen = signal(false);
  selectedTab = signal(0);
  
  // Use observables for:
  // 1. HTTP requests
  users$: Observable<any> = this.http.get('/api/users');
  
  // 2. Event streams
  clicks$ = fromEvent(document, 'click');
  
  // 3. Complex async flows
  data$ = combineLatest([this.users$, this.settings$]).pipe(
    map(([users, settings]) => processData(users, settings))
  );
}
```

## Advanced Patterns

### Signal-based Store

```typescript
import { Injectable, signal, computed } from '@angular/core';

export interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

@Injectable({ providedIn: 'root' })
export class TodoStore {
  // Private state
  private todos = signal<Todo[]>([]);
  private filter = signal<'all' | 'active' | 'completed'>('all');
  
  // Public selectors
  readonly allTodos = this.todos.asReadonly();
  readonly currentFilter = this.filter.asReadonly();
  
  readonly filteredTodos = computed(() => {
    const todos = this.todos();
    const filter = this.filter();
    
    switch (filter) {
      case 'active':
        return todos.filter(t => !t.completed);
      case 'completed':
        return todos.filter(t => t.completed);
      default:
        return todos;
    }
  });
  
  readonly stats = computed(() => {
    const todos = this.todos();
    return {
      total: todos.length,
      active: todos.filter(t => !t.completed).length,
      completed: todos.filter(t => t.completed).length
    };
  });
  
  // Actions
  addTodo(text: string): void {
    this.todos.update(todos => [
      ...todos,
      {
        id: Date.now(),
        text,
        completed: false
      }
    ]);
  }
  
  toggleTodo(id: number): void {
    this.todos.update(todos =>
      todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  }
  
  removeTodo(id: number): void {
    this.todos.update(todos => todos.filter(t => t.id !== id));
  }
  
  setFilter(filter: 'all' | 'active' | 'completed'): void {
    this.filter.set(filter);
  }
}
```

### Derived State Pattern

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-derived-state',
  template: `
    <div>
      <input type="number" (input)="celsius.set(+$any($event.target).value)">
      <p>Celsius: {{ celsius() }}°C</p>
      <p>Fahrenheit: {{ fahrenheit() }}°F</p>
      <p>Kelvin: {{ kelvin() }}K</p>
      <p>Description: {{ description() }}</p>
    </div>
  `
})
export class DerivedStateComponent {
  // Single source of truth
  celsius = signal(0);
  
  // All derived from celsius
  fahrenheit = computed(() => this.celsius() * 9/5 + 32);
  kelvin = computed(() => this.celsius() + 273.15);
  
  description = computed(() => {
    const temp = this.celsius();
    if (temp < 0) return 'Freezing';
    if (temp < 10) return 'Cold';
    if (temp < 20) return 'Cool';
    if (temp < 30) return 'Warm';
    return 'Hot';
  });
}
```

### Async Signal Pattern

```typescript
import { Component, signal, effect, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface AsyncState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

@Component({
  selector: 'app-async-signal',
  template: `
    <div>
      <button (click)="loadUser()">Load User</button>
      <p *ngIf="userState().loading">Loading...</p>
      <p *ngIf="userState().error">Error: {{ userState().error?.message }}</p>
      <p *ngIf="userState().data">User: {{ userState().data?.name }}</p>
    </div>
  `
})
export class AsyncSignalComponent {
  private http = inject(HttpClient);
  
  userState = signal<AsyncState<any>>({
    data: null,
    loading: false,
    error: null
  });
  
  loadUser(): void {
    this.userState.set({
      data: null,
      loading: true,
      error: null
    });
    
    this.http.get('/api/user').subscribe({
      next: data => {
        this.userState.set({
          data,
          loading: false,
          error: null
        });
      },
      error: error => {
        this.userState.set({
          data: null,
          loading: false,
          error
        });
      }
    });
  }
}
```

## Common Mistakes

### Mistake 1: Forgetting to Call Signal

```typescript
// BAD: Not calling signal
@Component({
  template: '<p>{{ count }}</p>' // undefined!
})
export class BadComponent {
  count = signal(0);
}

// GOOD: Call signal as function
@Component({
  template: '<p>{{ count() }}</p>'
})
export class GoodComponent {
  count = signal(0);
}
```

### Mistake 2: Mutating Signal Values

```typescript
// BAD: Directly mutating
const items = signal([1, 2, 3]);
items().push(4); // DON'T DO THIS!

const user = signal({ name: 'John' });
user().name = 'Jane'; // DON'T DO THIS!

// GOOD: Immutable updates
items.update(arr => [...arr, 4]);
user.update(u => ({ ...u, name: 'Jane' }));
```

### Mistake 3: Side Effects in Computed

```typescript
// BAD: Side effects in computed
const count = signal(0);
const doubled = computed(() => {
  console.log('Computing'); // Side effect!
  localStorage.setItem('count', count().toString()); // Side effect!
  return count() * 2;
});

// GOOD: Use effect for side effects
const count = signal(0);
const doubled = computed(() => count() * 2);

effect(() => {
  localStorage.setItem('count', count().toString());
});
```

### Mistake 4: Not Using Computed for Derived State

```typescript
// BAD: Manually tracking dependencies
@Component({
  template: '<p>{{ fullName }}</p>'
})
export class BadComponent {
  firstName = signal('John');
  lastName = signal('Doe');
  fullName = '';
  
  ngOnInit(): void {
    effect(() => {
      this.fullName = `${this.firstName()} ${this.lastName()}`;
    });
  }
}

// GOOD: Use computed
@Component({
  template: '<p>{{ fullName() }}</p>'
})
export class GoodComponent {
  firstName = signal('John');
  lastName = signal('Doe');
  fullName = computed(() => `${this.firstName()} ${this.lastName()}`);
}
```

### Mistake 5: Effects for Everything

```typescript
// BAD: Overusing effects
const firstName = signal('');
const lastName = signal('');
const fullName = signal('');

effect(() => {
  fullName.set(`${firstName()} ${lastName()}`); // Use computed instead!
});

// GOOD: Use computed
const firstName = signal('');
const lastName = signal('');
const fullName = computed(() => `${firstName()} ${lastName()}`);
```

## Best Practices

### 1. Prefer Computed Over Effects for Derived State

```typescript
// Always use computed for derived values
const price = signal(100);
const quantity = signal(2);
const total = computed(() => price() * quantity());

// Only use effects for side effects
effect(() => {
  console.log('Total changed:', total());
});
```

### 2. Keep Signals Immutable

```typescript
// Always create new objects/arrays
const items = signal([1, 2, 3]);
items.update(arr => [...arr, 4]); // Good

const user = signal({ name: 'John', age: 30 });
user.update(u => ({ ...u, age: 31 })); // Good
```

### 3. Use Readonly for Public Signals

```typescript
@Injectable({ providedIn: 'root' })
export class DataService {
  private _data = signal<any[]>([]);
  
  // Expose readonly version
  readonly data = this._data.asReadonly();
  
  addData(item: any): void {
    this._data.update(d => [...d, item]);
  }
}
```

### 4. Cleanup Effects Properly

```typescript
effect((onCleanup) => {
  const subscription = someObservable$.subscribe();
  
  onCleanup(() => {
    subscription.unsubscribe();
  });
});
```

### 5. Organize Signals by Concern

```typescript
@Component({
  selector: 'app-organized',
  template: '...'
})
export class OrganizedComponent {
  // State signals
  private _selectedId = signal<number | null>(null);
  private _filter = signal('');
  
  // Computed signals
  readonly selectedId = this._selectedId.asReadonly();
  readonly filter = this._filter.asReadonly();
  readonly filteredItems = computed(() => {
    // ...
  });
  
  // Actions
  selectItem(id: number): void {
    this._selectedId.set(id);
  }
  
  setFilter(filter: string): void {
    this._filter.set(filter);
  }
}
```

## Interview Questions

### Q1: What is the difference between signal() and computed()?

**Answer:** `signal()` creates writable reactive state that can be updated using `set()` or `update()`. `computed()` creates read-only derived state that automatically recalculates when its dependencies change. Computed signals are memoized and only recompute when dependencies change.

### Q2: When should you use effect() vs computed()?

**Answer:** Use `computed()` for derived values that you need to read in templates or other code. Use `effect()` for side effects like logging, DOM manipulation, or external API calls. Computed is for transforming data, effects are for reacting to changes.

### Q3: What are glitch-free updates?

**Answer:** Glitch-free updates ensure that derived values are always consistent with their dependencies. Angular batches signal updates and computes derived values only once with the final values, preventing temporary inconsistent states (glitches) in the UI.

### Q4: How do signals improve performance compared to Zone.js?

**Answer:** Signals enable fine-grained reactivity, updating only the specific bindings that depend on changed signals, rather than checking the entire component tree. This eliminates the need for Zone.js change detection and significantly reduces unnecessary checks.

### Q5: Can you modify an object inside a signal directly?

**Answer:** No, you should never mutate objects or arrays inside signals directly. Always use `update()` or `set()` with immutable updates to ensure Angular detects changes and triggers updates properly.

## Key Takeaways

1. Signals provide fine-grained reactivity without Zone.js
2. signal() creates writable state, computed() creates derived state
3. effect() is for side effects, not derived values
4. Signals guarantee glitch-free updates through batching
5. Computed signals are memoized and lazy
6. Always use immutable updates with signals
7. Signals are synchronous, observables are asynchronous
8. Use computed for derived state, effect for side effects
9. Effects run immediately and on dependency changes
10. Signals enable better performance and simpler mental model

## Resources

- [Angular Signals Guide](https://angular.dev/guide/signals)
- [Signals API Documentation](https://angular.dev/api/core/signal)
- [computed() API](https://angular.dev/api/core/computed)
- [effect() API](https://angular.dev/api/core/effect)
- [Angular Signals Deep Dive](https://blog.angular.io/angular-signals-deep-dive)
