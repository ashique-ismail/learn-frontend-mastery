# OnPush Change Detection with Signals Optimization

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Change Detection](#understanding-change-detection)
3. [OnPush Change Detection Strategy](#onpush-change-detection-strategy)
4. [Signals and Change Detection](#signals-and-change-detection)
5. [OnPush + Signals Pattern](#onpush--signals-pattern)
6. [Zone-less Applications](#zone-less-applications)
7. [Performance Optimization Patterns](#performance-optimization-patterns)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)
11. [Key Takeaways](#key-takeaways)
12. [Resources](#resources)

## Introduction

Change detection is Angular's mechanism for synchronizing the application state with the UI. While Angular's default change detection works well for most applications, OnPush strategy combined with Signals provides significant performance improvements by reducing unnecessary change detection cycles. This guide covers everything from understanding change detection to building high-performance, zone-less applications with OnPush and Signals.

## Understanding Change Detection

### How Change Detection Works

```typescript
/**
 * Change Detection Process:
 * 
 * 1. Event occurs (click, HTTP response, timer)
 * 2. Zone.js notifies Angular
 * 3. Angular runs change detection
 * 4. Checks all components (Default strategy)
 * 5. Updates DOM where needed
 * 
 * Problem with Default Strategy:
 * - Checks ALL components on EVERY event
 * - Unnecessary checks even if data didn't change
 * - Performance degrades with component tree size
 * 
 * OnPush Solution:
 * - Only checks when inputs change (reference change)
 * - Only checks when events fire from component
 * - Only checks when async pipe receives new value
 * - Only checks when explicitly marked for check
 * 
 * Signals Solution:
 * - Fine-grained reactivity
 * - Only updates what actually changed
 * - Can work without Zone.js
 * - Optimal performance
 */
```

### Default vs OnPush

```typescript
// Default Strategy
@Component({
  selector: 'app-default',
  template: `
    <h1>{{ title }}</h1>
    <p>Checked: {{ checkCount }}</p>
  `
})
export class DefaultComponent implements DoCheck {
  title = 'Default';
  checkCount = 0;
  
  ngDoCheck() {
    this.checkCount++; // Increments on EVERY change detection
    console.log('Checked:', this.checkCount);
  }
}

// OnPush Strategy
@Component({
  selector: 'app-onpush',
  template: `
    <h1>{{ title }}</h1>
    <p>Checked: {{ checkCount }}</p>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnPushComponent implements DoCheck {
  @Input() title = 'OnPush';
  checkCount = 0;
  
  ngDoCheck() {
    this.checkCount++; // Only increments when necessary
    console.log('Checked:', this.checkCount);
  }
}
```

## OnPush Change Detection Strategy

### Basic OnPush Implementation

```typescript
import { Component, Input, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <p>Age: {{ user.age }}</p>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: { name: string; email: string; age: number };
}

// Usage
@Component({
  selector: 'app-users',
  standalone: true,
  imports: [UserCardComponent],
  template: `
    <app-user-card 
      *ngFor="let user of users" 
      [user]="user">
    </app-user-card>
    
    <button (click)="updateUser()">Update User</button>
    <button (click)="updateUserWrong()">Update User (Wrong)</button>
  `
})
export class UsersComponent {
  users = [
    { name: 'John', email: 'john@example.com', age: 30 },
    { name: 'Jane', email: 'jane@example.com', age: 25 }
  ];
  
  updateUser() {
    // CORRECT: Create new reference
    this.users = [
      { ...this.users[0], age: 31 },
      this.users[1]
    ];
    // OnPush detects change because array reference changed
  }
  
  updateUserWrong() {
    // WRONG: Mutate existing object
    this.users[0].age = 31;
    // OnPush doesn't detect change because reference is same
  }
}
```

### Manual Change Detection

```typescript
import { 
  Component, 
  ChangeDetectorRef, 
  ChangeDetectionStrategy,
  OnInit 
} from '@angular/core';

@Component({
  selector: 'app-manual-cd',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1>{{ title }}</h1>
    <p>Count: {{ count }}</p>
    <p>Last updated: {{ lastUpdated }}</p>
  `
})
export class ManualCdComponent implements OnInit {
  title = 'Manual Change Detection';
  count = 0;
  lastUpdated = '';
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    // Timer won't trigger change detection automatically
    setInterval(() => {
      this.count++;
      this.lastUpdated = new Date().toLocaleTimeString();
      
      // Manually mark for check
      this.cdr.markForCheck();
    }, 1000);
  }
}
```

### Detach and Reattach

```typescript
import { 
  Component, 
  ChangeDetectorRef, 
  ChangeDetectionStrategy,
  OnInit,
  OnDestroy 
} from '@angular/core';

@Component({
  selector: 'app-detach-reattach',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1>{{ title }}</h1>
    <p>Updates: {{ updateCount }}</p>
    <button (click)="pause()">Pause</button>
    <button (click)="resume()">Resume</button>
  `
})
export class DetachReattachComponent implements OnInit, OnDestroy {
  title = 'Detach/Reattach Example';
  updateCount = 0;
  private intervalId?: number;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  ngOnInit() {
    this.intervalId = window.setInterval(() => {
      this.updateCount++;
      this.cdr.markForCheck();
    }, 1000);
  }
  
  pause() {
    // Completely stop change detection for this component
    this.cdr.detach();
  }
  
  resume() {
    // Resume change detection
    this.cdr.reattach();
    this.cdr.markForCheck(); // Update immediately
  }
  
  ngOnDestroy() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }
}
```

## Signals and Change Detection

### Basic Signals with OnPush

```typescript
import { 
  Component, 
  signal, 
  computed,
  ChangeDetectionStrategy 
} from '@angular/core';

@Component({
  selector: 'app-signals-basic',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <h2>{{ title() }}</h2>
      <p>Count: {{ count() }}</p>
      <p>Double: {{ doubleCount() }}</p>
      <p>Message: {{ message() }}</p>
      
      <button (click)="increment()">Increment</button>
      <button (click)="updateTitle()">Update Title</button>
    </div>
  `
})
export class SignalsBasicComponent {
  // Writable signals
  count = signal(0);
  title = signal('Signals Example');
  
  // Computed signal
  doubleCount = computed(() => this.count() * 2);
  message = computed(() => 
    `Count is ${this.count() % 2 === 0 ? 'even' : 'odd'}`
  );
  
  increment() {
    // Signals automatically trigger change detection
    this.count.update(c => c + 1);
  }
  
  updateTitle() {
    this.title.set('Updated Title');
  }
}
```

### Signals Automatically Trigger OnPush

```typescript
import { 
  Component, 
  signal, 
  computed,
  effect,
  ChangeDetectionStrategy 
} from '@angular/core';

@Component({
  selector: 'app-auto-trigger',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <h2>{{ user().name }}</h2>
      <p>Email: {{ user().email }}</p>
      <p>Full Info: {{ fullInfo() }}</p>
      
      <input 
        [value]="user().name" 
        (input)="updateName($event)"
      >
      
      <button (click)="updateUser()">Update User</button>
    </div>
  `
})
export class AutoTriggerComponent {
  user = signal({
    name: 'John',
    email: 'john@example.com'
  });
  
  fullInfo = computed(() => {
    const u = this.user();
    return `${u.name} (${u.email})`;
  });
  
  constructor() {
    // Effects run when signals change
    effect(() => {
      console.log('User changed:', this.user());
    });
  }
  
  updateName(event: Event) {
    const name = (event.target as HTMLInputElement).value;
    // Automatically triggers change detection
    this.user.update(u => ({ ...u, name }));
  }
  
  updateUser() {
    this.user.set({
      name: 'Jane',
      email: 'jane@example.com'
    });
  }
}
```

## OnPush + Signals Pattern

### Complete Example

```typescript
import { 
  Component, 
  signal, 
  computed,
  inject,
  ChangeDetectionStrategy 
} from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="user-list">
      <h2>Users ({{ filteredUsers().length }})</h2>
      
      <input 
        type="text" 
        placeholder="Search..." 
        [value]="searchTerm()"
        (input)="searchTerm.set($any($event.target).value)"
      >
      
      <div class="filters">
        <button 
          [class.active]="sortBy() === 'name'"
          (click)="sortBy.set('name')">
          Sort by Name
        </button>
        <button 
          [class.active]="sortBy() === 'email'"
          (click)="sortBy.set('email')">
          Sort by Email
        </button>
      </div>
      
      @if (loading()) {
        <div class="loading">Loading...</div>
      } @else if (error()) {
        <div class="error">{{ error() }}</div>
      } @else {
        <div class="users">
          @for (user of filteredUsers(); track user.id) {
            <div class="user-card">
              <h3>{{ user.name }}</h3>
              <p>{{ user.email }}</p>
            </div>
          }
        </div>
      }
    </div>
  `
})
export class UserListComponent {
  private http = inject(HttpClient);
  
  // State signals
  searchTerm = signal('');
  sortBy = signal<'name' | 'email'>('name');
  loading = signal(true);
  error = signal<string | null>(null);
  
  // Data signal from Observable
  users = toSignal(this.http.get<User[]>('/api/users'), {
    initialValue: []
  });
  
  // Computed signals (automatically update)
  filteredUsers = computed(() => {
    const search = this.searchTerm().toLowerCase();
    const sort = this.sortBy();
    
    let filtered = this.users().filter(user =>
      user.name.toLowerCase().includes(search) ||
      user.email.toLowerCase().includes(search)
    );
    
    return filtered.sort((a, b) => 
      a[sort].localeCompare(b[sort])
    );
  });
}
```

### Signal-based Form

```typescript
import { 
  Component, 
  signal, 
  computed,
  ChangeDetectionStrategy 
} from '@angular/core';

@Component({
  selector: 'app-signal-form',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <form (submit)="handleSubmit()">
      <div>
        <label>Name:</label>
        <input 
          [value]="name()" 
          (input)="name.set($any($event.target).value)"
          [class.error]="nameError()"
        >
        @if (nameError()) {
          <span class="error">{{ nameError() }}</span>
        }
      </div>
      
      <div>
        <label>Email:</label>
        <input 
          type="email"
          [value]="email()" 
          (input)="email.set($any($event.target).value)"
          [class.error]="emailError()"
        >
        @if (emailError()) {
          <span class="error">{{ emailError() }}</span>
        }
      </div>
      
      <button 
        type="submit" 
        [disabled]="!isValid()">
        Submit
      </button>
    </form>
    
    <div class="preview">
      <h3>Preview:</h3>
      <pre>{{ formData() | json }}</pre>
    </div>
  `
})
export class SignalFormComponent {
  // Form field signals
  name = signal('');
  email = signal('');
  
  // Validation computed signals
  nameError = computed(() => {
    const n = this.name();
    if (!n) return 'Name is required';
    if (n.length < 2) return 'Name must be at least 2 characters';
    return null;
  });
  
  emailError = computed(() => {
    const e = this.email();
    if (!e) return 'Email is required';
    if (!/\S+@\S+\.\S+/.test(e)) return 'Invalid email format';
    return null;
  });
  
  isValid = computed(() => 
    !this.nameError() && !this.emailError()
  );
  
  formData = computed(() => ({
    name: this.name(),
    email: this.email()
  }));
  
  handleSubmit() {
    if (this.isValid()) {
      console.log('Form submitted:', this.formData());
    }
  }
}
```

## Zone-less Applications

### Disabling Zone.js

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection()
  ]
});
```

### Zone-less Component

```typescript
import { 
  Component, 
  signal, 
  computed,
  ChangeDetectionStrategy 
} from '@angular/core';

@Component({
  selector: 'app-zoneless',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <h1>{{ title() }}</h1>
      <p>Count: {{ count() }}</p>
      <p>Is Even: {{ isEven() }}</p>
      
      <button (click)="increment()">Increment</button>
      
      <!-- This works without Zone.js! -->
      <input 
        [value]="text()" 
        (input)="text.set($any($event.target).value)"
      >
      <p>Text: {{ text() }}</p>
    </div>
  `
})
export class ZonelessComponent {
  title = signal('Zone-less App');
  count = signal(0);
  text = signal('');
  
  isEven = computed(() => this.count() % 2 === 0);
  
  increment() {
    // Signals trigger change detection automatically
    // No Zone.js needed!
    this.count.update(c => c + 1);
  }
}
```

### Mixing Observables in Zone-less

```typescript
import { 
  Component, 
  signal,
  inject,
  ChangeDetectionStrategy 
} from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-observable-zoneless',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <h2>Users</h2>
      @if (users(); as userList) {
        @for (user of userList; track user.id) {
          <div>{{ user.name }}</div>
        }
      }
      
      <p>Timer: {{ timer() }}</p>
    </div>
  `
})
export class ObservableZonelessComponent {
  private http = inject(HttpClient);
  
  // Convert Observable to Signal
  users = toSignal(
    this.http.get<any[]>('/api/users')
  );
  
  // Timer Observable as Signal
  timer = toSignal(
    interval(1000).pipe(map(n => n + 1)),
    { initialValue: 0 }
  );
}
```

## Performance Optimization Patterns

### Lazy Computed Signals

```typescript
import { Component, signal, computed, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-lazy-computed',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>
      <button (click)="addItem()">Add Item</button>
      
      @if (showStats()) {
        <div>
          <p>Count: {{ items().length }}</p>
          <p>Total: {{ total() }}</p>
          <p>Average: {{ average() }}</p>
        </div>
      }
      
      <button (click)="showStats.set(!showStats())">
        Toggle Stats
      </button>
    </div>
  `
})
export class LazyComputedComponent {
  items = signal([1, 2, 3, 4, 5]);
  showStats = signal(false);
  
  // Only computed when showStats is true and used in template
  total = computed(() => {
    console.log('Computing total');
    return this.items().reduce((sum, item) => sum + item, 0);
  });
  
  average = computed(() => {
    console.log('Computing average');
    const t = this.total();
    const len = this.items().length;
    return len > 0 ? t / len : 0;
  });
  
  addItem() {
    this.items.update(items => [...items, Math.floor(Math.random() * 100)]);
  }
}
```

### TrackBy with OnPush

```typescript
import { Component, signal, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-trackby-onpush',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button (click)="shuffleItems()">Shuffle</button>
    <button (click)="updateItem()">Update First</button>
    
    <!-- Without trackBy - re-renders all items -->
    <div>
      @for (item of items(); track $index) {
        <app-item [data]="item"></app-item>
      }
    </div>
    
    <!-- With trackBy - only re-renders changed items -->
    <div>
      @for (item of items(); track item.id) {
        <app-item [data]="item"></app-item>
      }
    </div>
  `
})
export class TrackByOnPushComponent {
  items = signal([
    { id: 1, name: 'Item 1', value: 10 },
    { id: 2, name: 'Item 2', value: 20 },
    { id: 3, name: 'Item 3', value: 30 }
  ]);
  
  shuffleItems() {
    this.items.update(items => [...items].sort(() => Math.random() - 0.5));
  }
  
  updateItem() {
    this.items.update(items => [
      { ...items[0], value: items[0].value + 1 },
      ...items.slice(1)
    ]);
  }
}
```

### Memoized Expensive Operations

```typescript
import { Component, signal, computed, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-memoized',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <input 
      [value]="searchTerm()" 
      (input)="searchTerm.set($any($event.target).value)"
    >
    
    <div>
      Results: {{ filteredResults().length }}
      @for (result of filteredResults(); track result.id) {
        <div>{{ result.name }}</div>
      }
    </div>
  `
})
export class MemoizedComponent {
  searchTerm = signal('');
  
  // Large dataset
  data = signal(
    Array.from({ length: 10000 }, (_, i) => ({
      id: i,
      name: `Item ${i}`,
      category: `Category ${i % 10}`
    }))
  );
  
  // Computed signals are automatically memoized
  filteredResults = computed(() => {
    const search = this.searchTerm().toLowerCase();
    console.log('Filtering...');
    
    if (!search) return this.data();
    
    return this.data().filter(item =>
      item.name.toLowerCase().includes(search)
    );
  });
}
```

## Common Mistakes

### 1. Mutating Objects with OnPush

```typescript
// BAD: Mutation doesn't trigger OnPush
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ user.name }}</p>`
})
export class BadComponent {
  @Input() user = { name: 'John' };
  
  updateUser() {
    this.user.name = 'Jane'; // OnPush won't detect this
  }
}

// GOOD: Create new reference
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ user().name }}</p>`
})
export class GoodComponent {
  user = signal({ name: 'John' });
  
  updateUser() {
    this.user.update(u => ({ ...u, name: 'Jane' }));
  }
}
```

### 2. Forgetting markForCheck

```typescript
// BAD: Change not detected
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ count }}</p>`
})
export class BadComponent implements OnInit {
  count = 0;
  
  ngOnInit() {
    setInterval(() => {
      this.count++; // OnPush won't detect this
    }, 1000);
  }
}

// GOOD: Use signals or markForCheck
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ count() }}</p>`
})
export class GoodComponent implements OnInit {
  count = signal(0);
  
  ngOnInit() {
    setInterval(() => {
      this.count.update(c => c + 1); // Automatically detected
    }, 1000);
  }
}
```

### 3. Mixing Patterns

```typescript
// BAD: Mixing mutable state with OnPush
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Signal: {{ signalCount() }}</p>
    <p>Regular: {{ regularCount }}</p>
  `
})
export class BadComponent {
  signalCount = signal(0);
  regularCount = 0; // Won't update reliably
  
  increment() {
    this.signalCount.update(c => c + 1); // Works
    this.regularCount++; // Might not work
  }
}

// GOOD: Use signals consistently
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count 1: {{ count1() }}</p>
    <p>Count 2: {{ count2() }}</p>
  `
})
export class GoodComponent {
  count1 = signal(0);
  count2 = signal(0);
  
  increment() {
    this.count1.update(c => c + 1);
    this.count2.update(c => c + 1);
  }
}
```

## Best Practices

### 1. Always Use OnPush with Signals

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush, // Always set this
  template: `{{ data() }}`
})
export class BestPracticeComponent {
  data = signal('value');
}
```

### 2. Use Computed for Derived State

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Total: {{ total() }}</p>
    <p>Average: {{ average() }}</p>
  `
})
export class ComputedBestPractice {
  numbers = signal([1, 2, 3, 4, 5]);
  
  total = computed(() => 
    this.numbers().reduce((sum, n) => sum + n, 0)
  );
  
  average = computed(() => 
    this.total() / this.numbers().length
  );
}
```

### 3. Use toSignal for Observables

```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ data() }}</p>`
})
export class ObservableBestPractice {
  private http = inject(HttpClient);
  
  data = toSignal(
    this.http.get('/api/data'),
    { initialValue: null }
  );
}
```

### 4. Immutable Updates

```typescript
// Always create new references
items.update(items => [...items, newItem]);
user.update(u => ({ ...u, name: 'New Name' }));
```

## Interview Questions

### Q1: What is OnPush change detection?

**Answer:** OnPush is a change detection strategy that only checks a component when its inputs change (by reference), when events occur within the component, when observables with async pipe emit, or when manually marked for check.

### Q2: How do Signals work with OnPush?

**Answer:** Signals automatically trigger OnPush change detection when their values change. The signal read in the template creates a subscription that marks the component for check when the signal updates.

### Q3: Can you use OnPush without Signals?

**Answer:** Yes, but you need to ensure immutable data patterns for inputs, use async pipe for observables, or manually call markForCheck() when needed.

### Q4: What's the performance benefit of OnPush?

**Answer:** OnPush reduces unnecessary change detection cycles by only checking components when they actually need updating, rather than checking the entire component tree on every event.

### Q5: What is a zone-less application?

**Answer:** A zone-less application doesn't use Zone.js for change detection. Instead, it relies on Signals and explicit change detection triggers, resulting in smaller bundle size and better performance.

### Q6: When should you use markForCheck()?

**Answer:** Use markForCheck() when you update component state outside of Angular's knowledge (setTimeout, non-Angular events) in an OnPush component. With Signals, you rarely need it.

### Q7: What's the difference between detectChanges() and markForCheck()?

**Answer:** detectChanges() immediately runs change detection for the component and its children. markForCheck() marks the component to be checked in the next change detection cycle.

### Q8: How do computed signals optimize performance?

**Answer:** Computed signals are memoized and only recalculate when their dependencies change. They're also lazy - they only compute when actually used in the template or another computed signal.

## Key Takeaways

1. **OnPush reduces change detection cycles** by only checking when necessary
2. **Signals automatically work with OnPush** and trigger updates efficiently
3. **Computed signals are automatically memoized** and lazily evaluated
4. **Zone-less apps with Signals** offer best performance
5. **Always use immutable data patterns** with OnPush
6. **toSignal converts Observables** to work seamlessly with Signals
7. **markForCheck() rarely needed** when using Signals
8. **Combine OnPush + Signals** for optimal performance
9. **trackBy improves list rendering** performance with OnPush
10. **Consistent Signal usage** throughout the app works best

## Resources

### Official Documentation
- [Change Detection](https://angular.dev/best-practices/runtime-performance)
- [Signals](https://angular.dev/guide/signals)
- [OnPush Strategy](https://angular.dev/api/core/ChangeDetectionStrategy)

### Articles
- "OnPush Change Detection Deep Dive" - Angular Blog
- "Signals and Performance" - Web.dev
- "Zone-less Angular" - Angular University

### Video Tutorials
- "Change Detection Explained" - ng-conf
- "Mastering Signals" - Angular Connect
- "OnPush Performance Patterns" - Angular YouTube

### Tools
- Angular DevTools - Profile change detection
- Chrome DevTools - Performance profiling
- Profiler API - Measure component render time
