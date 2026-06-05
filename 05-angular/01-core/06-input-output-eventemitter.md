# Input, Output, and EventEmitter

## The Idea

**In plain English:** Angular apps are built from components (think of them as self-contained building blocks, like LEGO bricks). @Input and @Output are the ways those bricks talk to each other — @Input lets a parent brick hand information down to a child brick, and @Output lets a child brick shout something back up to the parent when something happens.

**Real-world analogy:** Imagine a vending machine in a school cafeteria. A student (the parent) presses a button to select a snack and inserts coins (giving the machine information). The machine then dispenses the snack and flashes a "Thank You" message on its screen (telling the student something happened inside).

- The student pressing a button and inserting coins = @Input (parent passing data into the component)
- The machine's "Thank You" screen notification = @Output with EventEmitter (child component broadcasting an event back to the parent)
- The coins/button choice itself = the data value being passed
- The "Thank You" flash = the emitted event payload

---

## Overview

Angular components communicate through @Input for parent-to-child data flow and @Output with EventEmitter for child-to-parent event communication. Understanding these decorators is fundamental to building component hierarchies and managing data flow in Angular applications.

## Table of Contents

1. [@Input Decorator](#input-decorator)
2. [@Output Decorator](#output-decorator)
3. [EventEmitter](#eventemitter)
4. [Two-Way Binding](#two-way-binding)
5. [Input/Output Aliases](#input-output-aliases)
6. [Input Transforms](#input-transforms)
7. [Required Inputs](#required-inputs)
8. [Common Patterns](#common-patterns)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)

## @Input Decorator

@Input allows parent components to pass data to child components.

### Basic Input

```typescript
// child.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ userName }}</h3>
      <p>{{ userEmail }}</p>
    </div>
  `
})
export class UserCardComponent {
  @Input() userName = '';
  @Input() userEmail = '';
}

// parent.component.ts
@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [UserCardComponent],
  template: `
    <app-user-card
      [userName]="'Alice'"
      [userEmail]="'alice@example.com'">
    </app-user-card>
  `
})
export class ParentComponent {}
```

### Object Input

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  avatar?: string;
}

@Component({
  selector: 'app-user-profile',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="profile">
      @if (user.avatar) {
        <img [src]="user.avatar" [alt]="user.name">
      }
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserProfileComponent {
  @Input() user!: User; // ! means required, will error if not provided
}

// Usage
@Component({
  template: `
    <app-user-profile [user]="currentUser"></app-user-profile>
  `
})
export class AppComponent {
  currentUser: User = {
    id: 1,
    name: 'Alice',
    email: 'alice@example.com',
    avatar: '/assets/alice.jpg'
  };
}
```

### Multiple Inputs

```typescript
@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [CommonModule, CurrencyPipe],
  template: `
    <div class="product-card" [class.featured]="featured">
      <img [src]="imageUrl" [alt]="title">
      <h3>{{ title }}</h3>
      <p>{{ description }}</p>
      <div class="price">{{ price | currency }}</div>
      @if (featured) {
        <span class="badge">Featured</span>
      }
    </div>
  `
})
export class ProductCardComponent {
  @Input() title = '';
  @Input() description = '';
  @Input() price = 0;
  @Input() imageUrl = '';
  @Input() featured = false;
}

// Usage
@Component({
  template: `
    <app-product-card
      [title]="product.title"
      [description]="product.description"
      [price]="product.price"
      [imageUrl]="product.image"
      [featured]="true">
    </app-product-card>
  `
})
export class ProductListComponent {
  product = {
    title: 'Laptop',
    description: 'High-performance laptop',
    price: 999.99,
    image: '/assets/laptop.jpg'
  };
}
```

### Default Values

```typescript
@Component({
  selector: 'app-button',
  standalone: true,
  template: `
    <button
      [class]="'btn btn-' + variant"
      [disabled]="disabled">
      <ng-content></ng-content>
    </button>
  `,
  styles: [`
    .btn { padding: 8px 16px; border: none; border-radius: 4px; }
    .btn-primary { background: blue; color: white; }
    .btn-secondary { background: gray; color: white; }
  `]
})
export class ButtonComponent {
  @Input() variant: 'primary' | 'secondary' = 'primary'; // Default value
  @Input() disabled = false;
}
```

## @Output Decorator

@Output allows child components to emit events to parent components.

### Basic Output

```typescript
// child.component.ts
import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <div class="counter">
      <button (click)="decrement()">-</button>
      <span>{{ count }}</span>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  count = 0;
  
  @Output() countChange = new EventEmitter<number>();
  
  increment() {
    this.count++;
    this.countChange.emit(this.count);
  }
  
  decrement() {
    this.count--;
    this.countChange.emit(this.count);
  }
}

// parent.component.ts
@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [CounterComponent],
  template: `
    <div>
      <h2>Parent received: {{ receivedCount }}</h2>
      <app-counter (countChange)="onCountChange($event)"></app-counter>
    </div>
  `
})
export class ParentComponent {
  receivedCount = 0;
  
  onCountChange(newCount: number) {
    this.receivedCount = newCount;
    console.log('Count changed to:', newCount);
  }
}
```

### Output with Objects

```typescript
interface FormData {
  name: string;
  email: string;
  message: string;
}

@Component({
  selector: 'app-contact-form',
  standalone: true,
  imports: [FormsModule],
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="formData.name" name="name" placeholder="Name">
      <input [(ngModel)]="formData.email" name="email" placeholder="Email">
      <textarea [(ngModel)]="formData.message" name="message"></textarea>
      <button type="submit">Submit</button>
    </form>
  `
})
export class ContactFormComponent {
  formData: FormData = { name: '', email: '', message: '' };
  
  @Output() formSubmit = new EventEmitter<FormData>();
  
  onSubmit() {
    this.formSubmit.emit(this.formData);
  }
}

// Parent
@Component({
  template: `
    <app-contact-form (formSubmit)="handleSubmit($event)"></app-contact-form>
  `
})
export class ParentComponent {
  handleSubmit(data: FormData) {
    console.log('Form submitted:', data);
    // Send to API
  }
}
```

### Multiple Outputs

```typescript
interface SearchParams {
  query: string;
  category: string;
}

@Component({
  selector: 'app-search-box',
  standalone: true,
  imports: [FormsModule],
  template: `
    <div class="search-box">
      <input
        [(ngModel)]="query"
        (keyup.enter)="search()"
        placeholder="Search...">
      
      <select [(ngModel)]="category">
        <option value="">All Categories</option>
        <option value="books">Books</option>
        <option value="electronics">Electronics</option>
      </select>
      
      <button (click)="search()">Search</button>
      <button (click)="clear()">Clear</button>
    </div>
  `
})
export class SearchBoxComponent {
  query = '';
  category = '';
  
  @Output() searchPerformed = new EventEmitter<SearchParams>();
  @Output() searchCleared = new EventEmitter<void>();
  
  search() {
    this.searchPerformed.emit({
      query: this.query,
      category: this.category
    });
  }
  
  clear() {
    this.query = '';
    this.category = '';
    this.searchCleared.emit();
  }
}
```

## EventEmitter

EventEmitter is Angular's implementation of the observable pattern for component events.

### Generic EventEmitter

```typescript
@Component({
  selector: 'app-item-selector',
  standalone: true,
  template: `
    <div class="items">
      @for (item of items; track item.id) {
        <div
          class="item"
          (click)="selectItem(item)"
          [class.selected]="item === selectedItem">
          {{ item.name }}
        </div>
      }
    </div>
  `
})
export class ItemSelectorComponent<T extends { id: number; name: string }> {
  @Input() items: T[] = [];
  @Output() itemSelected = new EventEmitter<T>();
  
  selectedItem: T | null = null;
  
  selectItem(item: T) {
    this.selectedItem = item;
    this.itemSelected.emit(item);
  }
}
```

### EventEmitter with Custom Events

```typescript
enum NotificationType {
  Success = 'success',
  Error = 'error',
  Warning = 'warning'
}

interface NotificationEvent {
  type: NotificationType;
  message: string;
  timestamp: Date;
}

@Component({
  selector: 'app-notification',
  standalone: true,
  template: `
    <div [class]="'notification ' + notification.type">
      <p>{{ notification.message }}</p>
      <button (click)="dismiss()">×</button>
    </div>
  `
})
export class NotificationComponent {
  @Input() notification!: NotificationEvent;
  @Output() dismissed = new EventEmitter<NotificationEvent>();
  
  dismiss() {
    this.dismissed.emit(this.notification);
  }
}
```

### Async Outputs

```typescript
@Component({
  selector: 'app-file-uploader',
  standalone: true,
  template: `
    <div class="uploader">
      <input
        type="file"
        (change)="onFileSelected($event)"
        #fileInput>
      
      <button (click)="fileInput.click()">Choose File</button>
      
      @if (uploading) {
        <div class="progress">Uploading...</div>
      }
    </div>
  `
})
export class FileUploaderComponent {
  uploading = false;
  
  @Output() fileUploaded = new EventEmitter<string>(); // URL
  @Output() uploadError = new EventEmitter<string>(); // Error message
  
  async onFileSelected(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];
    
    if (!file) return;
    
    this.uploading = true;
    
    try {
      const url = await this.uploadFile(file);
      this.fileUploaded.emit(url);
    } catch (error) {
      this.uploadError.emit('Upload failed');
    } finally {
      this.uploading = false;
    }
  }
  
  private async uploadFile(file: File): Promise<string> {
    // Upload logic
    return 'https://example.com/uploaded-file.jpg';
  }
}
```

## Two-Way Binding

Combine @Input and @Output for two-way data binding.

### Banana-in-a-Box Syntax

```typescript
// [(ngModel)] pattern
@Component({
  selector: 'app-custom-input',
  standalone: true,
  template: `
    <input
      [value]="value"
      (input)="onValueChange($event)"
      type="text">
  `
})
export class CustomInputComponent {
  @Input() value = '';
  @Output() valueChange = new EventEmitter<string>();
  
  onValueChange(event: Event) {
    const input = event.target as HTMLInputElement;
    this.valueChange.emit(input.value);
  }
}

// Usage with two-way binding
@Component({
  template: `
    <app-custom-input [(value)]="name"></app-custom-input>
    <p>Name: {{ name }}</p>
  `
})
export class ParentComponent {
  name = 'Alice';
}

// Equivalent to:
// <app-custom-input
//   [value]="name"
//   (valueChange)="name = $event">
// </app-custom-input>
```

### Custom Two-Way Binding

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <div class="counter">
      <button (click)="decrement()">-</button>
      <span>{{ count }}</span>
      <button (click)="increment()">+</button>
    </div>
  `
})
export class CounterComponent {
  @Input() count = 0;
  @Output() countChange = new EventEmitter<number>();
  
  increment() {
    this.count++;
    this.countChange.emit(this.count);
  }
  
  decrement() {
    this.count--;
    this.countChange.emit(this.count);
  }
}

// Two-way binding usage
@Component({
  template: `
    <app-counter [(count)]="myCount"></app-counter>
    <p>Current count: {{ myCount }}</p>
  `
})
export class AppComponent {
  myCount = 5;
}
```

### Complex Two-Way Binding

```typescript
interface SelectedOptions {
  category: string;
  minPrice: number;
  maxPrice: number;
}

@Component({
  selector: 'app-filter-panel',
  standalone: true,
  imports: [FormsModule],
  template: `
    <div class="filters">
      <select [(ngModel)]="options.category" (ngModelChange)="emitChange()">
        <option value="">All</option>
        <option value="books">Books</option>
        <option value="electronics">Electronics</option>
      </select>
      
      <input
        type="number"
        [(ngModel)]="options.minPrice"
        (ngModelChange)="emitChange()"
        placeholder="Min Price">
      
      <input
        type="number"
        [(ngModel)]="options.maxPrice"
        (ngModelChange)="emitChange()"
        placeholder="Max Price">
    </div>
  `
})
export class FilterPanelComponent {
  @Input() options: SelectedOptions = {
    category: '',
    minPrice: 0,
    maxPrice: 1000
  };
  
  @Output() optionsChange = new EventEmitter<SelectedOptions>();
  
  emitChange() {
    this.optionsChange.emit(this.options);
  }
}
```

## Input/Output Aliases

Rename input/output properties for external use.

### Input Aliases

```typescript
@Component({
  selector: 'app-user-badge',
  standalone: true,
  template: `
    <div class="badge">
      {{ displayName }}
    </div>
  `
})
export class UserBadgeComponent {
  // Internal property name: displayName
  // External binding name: name
  @Input('name') displayName = '';
}

// Usage
@Component({
  template: `
    <app-user-badge [name]="'Alice'"></app-user-badge>
  `
})
export class AppComponent {}
```

### Output Aliases

```typescript
@Component({
  selector: 'app-confirmation-dialog',
  standalone: true,
  template: `
    <div class="dialog">
      <p>{{ message }}</p>
      <button (click)="confirm()">Yes</button>
      <button (click)="cancel()">No</button>
    </div>
  `
})
export class ConfirmationDialogComponent {
  @Input() message = 'Are you sure?';
  
  // Internal: onConfirm, External: confirmed
  @Output('confirmed') onConfirm = new EventEmitter<void>();
  
  // Internal: onCancel, External: cancelled
  @Output('cancelled') onCancel = new EventEmitter<void>();
  
  confirm() {
    this.onConfirm.emit();
  }
  
  cancel() {
    this.onCancel.emit();
  }
}

// Usage
@Component({
  template: `
    <app-confirmation-dialog
      (confirmed)="handleConfirm()"
      (cancelled)="handleCancel()">
    </app-confirmation-dialog>
  `
})
export class AppComponent {
  handleConfirm() {
    console.log('Confirmed');
  }
  
  handleCancel() {
    console.log('Cancelled');
  }
}
```

## Input Transforms

Transform input values before they're set (Angular 16+).

### Boolean Transform

```typescript
import { Component, Input, booleanAttribute } from '@angular/core';

@Component({
  selector: 'app-expandable',
  standalone: true,
  template: `
    <div [class.expanded]="expanded">
      <ng-content></ng-content>
    </div>
  `
})
export class ExpandableComponent {
  // Transforms "true", "false", "" to boolean
  @Input({ transform: booleanAttribute }) expanded = false;
}

// Usage
// <app-expandable expanded>Content</app-expandable>
// <app-expandable [expanded]="true">Content</app-expandable>
// <app-expandable>Content</app-expandable>
```

### Number Transform

```typescript
import { Component, Input, numberAttribute } from '@angular/core';

@Component({
  selector: 'app-pagination',
  standalone: true,
  template: `
    <div>
      Page {{ currentPage }} of {{ totalPages }}
    </div>
  `
})
export class PaginationComponent {
  @Input({ transform: numberAttribute }) currentPage = 1;
  @Input({ transform: numberAttribute }) totalPages = 1;
}

// Usage
// <app-pagination currentPage="5" totalPages="10"></app-pagination>
```

### Custom Transform

```typescript
function trimTransform(value: string | undefined): string {
  return value?.trim() ?? '';
}

@Component({
  selector: 'app-search',
  standalone: true,
  template: `<input [value]="query">`
})
export class SearchComponent {
  @Input({ transform: trimTransform }) query = '';
}
```

## Required Inputs

Mark inputs as required (Angular 16+).

### Required Input Syntax

```typescript
import { Component, Input } from '@angular/core';

interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-product-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ product.name }}</h3>
      <p>{{ product.price }}</p>
    </div>
  `
})
export class ProductCardComponent {
  // Required input - compiler error if not provided
  @Input({ required: true }) product!: Product;
}

// ❌ Error: Missing required input 'product'
// <app-product-card></app-product-card>

// ✅ Correct
// <app-product-card [product]="myProduct"></app-product-card>
```

## Common Patterns

### Container/Presentational Pattern

```typescript
// Presentational (dumb component)
@Component({
  selector: 'app-user-list-view',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="user-list">
      @for (user of users; track user.id) {
        <div class="user-card" (click)="onUserClick(user)">
          {{ user.name }}
        </div>
      }
    </div>
  `
})
export class UserListViewComponent {
  @Input() users: User[] = [];
  @Output() userSelected = new EventEmitter<User>();
  
  onUserClick(user: User) {
    this.userSelected.emit(user);
  }
}

// Container (smart component)
@Component({
  selector: 'app-user-list-container',
  standalone: true,
  imports: [UserListViewComponent],
  template: `
    <app-user-list-view
      [users]="users"
      (userSelected)="navigateToUser($event)">
    </app-user-list-view>
  `
})
export class UserListContainerComponent {
  users: User[] = [];
  
  constructor(private router: Router, private userService: UserService) {
    this.userService.getUsers().subscribe(users => {
      this.users = users;
    });
  }
  
  navigateToUser(user: User) {
    this.router.navigate(['/users', user.id]);
  }
}
```

## Common Mistakes

### 1. Mutating Input Objects

```typescript
// ❌ WRONG: Mutating input
@Component({
  selector: 'app-user-editor'
})
export class UserEditorComponent {
  @Input() user!: User;
  
  updateName(newName: string) {
    this.user.name = newName; // Mutates parent's object!
  }
}

// ✅ CORRECT: Emit changes
@Component({
  selector: 'app-user-editor'
})
export class UserEditorComponent {
  @Input() user!: User;
  @Output() userChange = new EventEmitter<User>();
  
  updateName(newName: string) {
    const updatedUser = { ...this.user, name: newName };
    this.userChange.emit(updatedUser);
  }
}
```

### 2. Not Typing EventEmitter

```typescript
// ❌ WRONG: Untyped EventEmitter
@Output() clicked = new EventEmitter();

// ✅ CORRECT: Typed EventEmitter
@Output() clicked = new EventEmitter<MouseEvent>();
```

### 3. Subscribing to EventEmitter

```typescript
// ❌ WRONG: Don't subscribe in component
@Component({
  selector: 'app-parent'
})
export class ParentComponent {
  @ViewChild(ChildComponent) child!: ChildComponent;
  
  ngAfterViewInit() {
    this.child.dataChanged.subscribe(data => {
      console.log(data); // Memory leak!
    });
  }
}

// ✅ CORRECT: Use template event binding
@Component({
  template: `
    <app-child (dataChanged)="handleDataChange($event)"></app-child>
  `
})
export class ParentComponent {
  handleDataChange(data: any) {
    console.log(data);
  }
}
```

## Best Practices

### 1. Use Immutable Data

```typescript
// ✅ Emit new objects
@Output() dataChange = new EventEmitter<Data>();

updateData() {
  const newData = { ...this.data, updated: true };
  this.dataChange.emit(newData);
}
```

### 2. Type Your Outputs

```typescript
// ✅ Always type EventEmitters
@Output() userSelected = new EventEmitter<User>();
@Output() formSubmitted = new EventEmitter<FormData>();
@Output() errorOccurred = new EventEmitter<Error>();
```

### 3. Use Descriptive Names

```typescript
// ✅ Clear naming
@Input() userName: string;
@Input() isActive: boolean;
@Output() userDeleted = new EventEmitter<number>();
@Output() formSubmitted = new EventEmitter<FormData>();
```

### 4. Provide Defaults

```typescript
// ✅ Default values for optional inputs
@Input() size: 'small' | 'medium' | 'large' = 'medium';
@Input() disabled = false;
@Input() items: Item[] = [];
```

## Interview Questions

### Q1: What's the difference between @Input and @Output?

**Answer:** @Input passes data from parent to child (one-way down). @Output emits events from child to parent using EventEmitter (one-way up). Together they enable parent-child communication.

### Q2: How does two-way binding work with [(ngModel)]?

**Answer:** Two-way binding combines property binding and event binding. `[(value)]="data"` expands to `[value]="data" (valueChange)="data = $event"`. The component needs an @Input property and an @Output with the same name plus "Change" suffix.

### Q3: What is EventEmitter and how is it used?

**Answer:** EventEmitter is Angular's implementation of the observable pattern for component events. It's used with @Output to emit events from child to parent: `@Output() clicked = new EventEmitter<T>()`. Call `this.clicked.emit(value)` to emit events.

### Q4: When should you use input aliases?

**Answer:** Use aliases when the internal property name should differ from the external binding name for clarity or to avoid conflicts: `@Input('externalName') internalName: string`. Useful for renaming without breaking external API.

### Q5: What are required inputs and how do you use them?

**Answer:** Required inputs (Angular 16+) enforce that a value must be provided: `@Input({ required: true }) data!: Type`. The compiler errors if the input isn't bound in the template, preventing runtime errors from missing data.

## Key Takeaways

1. @Input enables parent-to-child data flow
2. @Output with EventEmitter enables child-to-parent events
3. Two-way binding combines both: [(property)]
4. EventEmitter should always be typed
5. Use required: true for mandatory inputs
6. Transform inputs with built-in or custom functions
7. Aliases rename properties for external use
8. Don't mutate input objects, emit new ones
9. Provide default values for optional inputs
10. Use container/presentational pattern for separation

## Resources

- [Angular Input/Output Guide](https://angular.io/guide/inputs-outputs)
- [EventEmitter API](https://angular.io/api/core/EventEmitter)
- [Two-Way Binding](https://angular.io/guide/two-way-binding)
- [Input Transforms](https://angular.io/guide/inputs-outputs#input-transforms)
- [Component Interaction](https://angular.io/guide/component-interaction)
