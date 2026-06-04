# Components, Templates, and Decorators

## Overview

Components are the fundamental building blocks of Angular applications. The `@Component` decorator defines metadata that Angular uses to create and present components. Understanding component configuration, template options, and styling strategies is essential for building Angular applications.

## Table of Contents

1. [@Component Decorator](#component-decorator)
2. [Component Metadata](#component-metadata)
3. [Selector Types](#selector-types)
4. [Template Options](#template-options)
5. [Style Encapsulation](#style-encapsulation)
6. [Change Detection Strategies](#change-detection-strategies)
7. [Component Lifecycle](#component-lifecycle)
8. [Host Bindings](#host-bindings)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)

## @Component Decorator

The @Component decorator marks a class as an Angular component and provides configuration metadata.

### Basic Component

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: '<h1>Hello, {{ name }}!</h1>',
  styles: ['h1 { color: blue; }']
})
export class HelloComponent {
  name = 'World';
}
```

### Standalone Component

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './user-card.component.html',
  styleUrls: ['./user-card.component.css']
})
export class UserCardComponent {
  userName = 'Alice';
  userEmail = 'alice@example.com';
}
```

### Component with Full Metadata

```typescript
import { Component, ViewEncapsulation, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-product-card',
  standalone: true,
  templateUrl: './product-card.component.html',
  styleUrls: [
    './product-card.component.css',
    './product-card-theme.css'
  ],
  encapsulation: ViewEncapsulation.Emulated,
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    'class': 'product-card',
    '[class.featured]': 'isFeatured',
    '(click)': 'onClick()'
  }
})
export class ProductCardComponent {
  isFeatured = false;
  
  onClick() {
    console.log('Product card clicked');
  }
}
```

## Component Metadata

Essential properties of the @Component decorator.

### selector

```typescript
// Element selector
@Component({
  selector: 'app-button',
  template: '<button><ng-content></ng-content></button>'
})
export class ButtonComponent { }

// Usage: <app-button>Click me</app-button>

// Attribute selector
@Component({
  selector: '[app-tooltip]',
  template: '<span class="tooltip">{{ text }}</span>'
})
export class TooltipComponent {
  text = 'Tooltip text';
}

// Usage: <div app-tooltip>Hover me</div>

// Class selector
@Component({
  selector: '.app-highlight',
  template: '<div class="highlight"><ng-content></ng-content></div>'
})
export class HighlightComponent { }

// Usage: <div class="app-highlight">Highlighted text</div>
```

### template vs templateUrl

```typescript
// Inline template (small templates)
@Component({
  selector: 'app-greeting',
  template: `
    <div class="greeting">
      <h2>Hello, {{ name }}!</h2>
      <p>Welcome to our application.</p>
    </div>
  `
})
export class GreetingComponent {
  name = 'User';
}

// External template (larger templates)
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html'
})
export class UserProfileComponent {
  user = {
    name: 'Alice',
    email: 'alice@example.com',
    bio: 'Software developer'
  };
}
```

### styles vs styleUrls

```typescript
// Inline styles
@Component({
  selector: 'app-card',
  template: '<div class="card"><ng-content></ng-content></div>',
  styles: [`
    .card {
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 16px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
  `]
})
export class CardComponent { }

// External stylesheets
@Component({
  selector: 'app-dashboard',
  templateUrl: './dashboard.component.html',
  styleUrls: [
    './dashboard.component.css',
    './dashboard-responsive.css'
  ]
})
export class DashboardComponent { }

// Multiple inline styles
@Component({
  selector: 'app-complex',
  template: '<div>Complex component</div>',
  styles: [
    `
      .container {
        display: flex;
        gap: 16px;
      }
    `,
    `
      .item {
        flex: 1;
        padding: 8px;
      }
    `
  ]
})
export class ComplexComponent { }
```

## Selector Types

Different ways to define component selectors.

### Element Selector (Most Common)

```typescript
@Component({
  selector: 'app-header',
  template: `
    <header>
      <h1>{{ title }}</h1>
      <nav>
        <a href="/home">Home</a>
        <a href="/about">About</a>
      </nav>
    </header>
  `
})
export class HeaderComponent {
  title = 'My Application';
}

// Usage
// <app-header></app-header>
```

### Attribute Selector

```typescript
@Component({
  selector: '[appDropdown]',
  template: `
    <div class="dropdown" (click)="toggle()">
      <ng-content></ng-content>
      @if (isOpen) {
        <div class="dropdown-menu">
          <ng-content select="[menu]"></ng-content>
        </div>
      }
    </div>
  `,
  styles: [`
    .dropdown { position: relative; }
    .dropdown-menu {
      position: absolute;
      top: 100%;
      left: 0;
      background: white;
      border: 1px solid #ddd;
    }
  `]
})
export class DropdownComponent {
  isOpen = false;
  
  toggle() {
    this.isOpen = !this.isOpen;
  }
}

// Usage
// <div appDropdown>
//   <button>Toggle</button>
//   <div menu>Menu items</div>
// </div>
```

### Class Selector

```typescript
@Component({
  selector: '.alert-box',
  template: `
    <div class="alert" [class.alert-error]="type === 'error'">
      <strong>{{ type }}:</strong> {{ message }}
    </div>
  `
})
export class AlertBoxComponent {
  type = 'info';
  message = 'This is an alert';
}

// Usage
// <div class="alert-box"></div>
```

### Combined Selectors

```typescript
// Element with attribute
@Component({
  selector: 'button[app-primary]',
  template: '<ng-content></ng-content>',
  styles: [`
    :host {
      background-color: blue;
      color: white;
      padding: 8px 16px;
      border: none;
      border-radius: 4px;
    }
  `]
})
export class PrimaryButtonComponent { }

// Usage: <button app-primary>Click me</button>

// Multiple selectors (not recommended)
@Component({
  selector: 'app-button, button[app-button]',
  template: '<ng-content></ng-content>'
})
export class ButtonComponent { }
```

## Template Options

Different ways to define and structure component templates.

### Multi-line Template Strings

```typescript
@Component({
  selector: 'app-user-form',
  template: `
    <form (ngSubmit)="onSubmit()">
      <div class="form-group">
        <label for="name">Name:</label>
        <input
          id="name"
          type="text"
          [(ngModel)]="user.name"
          required
        />
      </div>
      
      <div class="form-group">
        <label for="email">Email:</label>
        <input
          id="email"
          type="email"
          [(ngModel)]="user.email"
          required
        />
      </div>
      
      <button type="submit">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  user = { name: '', email: '' };
  
  onSubmit() {
    console.log('User:', this.user);
  }
}
```

### External Template File

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.css']
})
export class UserListComponent {
  users = [
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' }
  ];
  
  deleteUser(id: number) {
    this.users = this.users.filter(u => u.id !== id);
  }
}

// user-list.component.html
/*
<div class="user-list">
  <h2>Users</h2>
  <div class="user-card" *ngFor="let user of users">
    <h3>{{ user.name }}</h3>
    <p>{{ user.email }}</p>
    <button (click)="deleteUser(user.id)">Delete</button>
  </div>
</div>
*/
```

### Template with Conditional Content

```typescript
@Component({
  selector: 'app-loading-wrapper',
  template: `
    @if (loading) {
      <div class="spinner">Loading...</div>
    } @else if (error) {
      <div class="error">{{ error }}</div>
    } @else {
      <ng-content></ng-content>
    }
  `
})
export class LoadingWrapperComponent {
  loading = false;
  error: string | null = null;
}
```

## Style Encapsulation

Control how component styles are scoped and applied.

### ViewEncapsulation.Emulated (Default)

```typescript
@Component({
  selector: 'app-card',
  template: `<div class="card">Card content</div>`,
  styles: [`
    .card {
      border: 1px solid blue;
      padding: 16px;
    }
  `],
  encapsulation: ViewEncapsulation.Emulated
})
export class CardComponent { }

// Generated HTML:
// <app-card _ngcontent-c0>
//   <div _ngcontent-c0 class="card">Card content</div>
// </app-card>

// Generated CSS:
// .card[_ngcontent-c0] {
//   border: 1px solid blue;
//   padding: 16px;
// }
```

### ViewEncapsulation.None (Global Styles)

```typescript
@Component({
  selector: 'app-global-styles',
  template: `<div class="global">Global styled content</div>`,
  styles: [`
    .global {
      color: red;
    }
  `],
  encapsulation: ViewEncapsulation.None
})
export class GlobalStylesComponent { }

// Generated CSS (applies globally):
// .global {
//   color: red;
// }
// This will affect ALL .global elements in the entire app!
```

### ViewEncapsulation.ShadowDom (Native Shadow DOM)

```typescript
@Component({
  selector: 'app-shadow',
  template: `<div class="shadow-content">Shadow DOM content</div>`,
  styles: [`
    .shadow-content {
      background: lightgray;
      padding: 16px;
    }
  `],
  encapsulation: ViewEncapsulation.ShadowDom
})
export class ShadowComponent { }

// Creates true Shadow DOM:
// <app-shadow>
//   #shadow-root
//     <style>.shadow-content { ... }</style>
//     <div class="shadow-content">Shadow DOM content</div>
// </app-shadow>
```

### :host and :host-context

```typescript
@Component({
  selector: 'app-themed-button',
  template: `<button><ng-content></ng-content></button>`,
  styles: [`
    /* Style the component host element */
    :host {
      display: inline-block;
      margin: 4px;
    }
    
    /* Style host when it has a class */
    :host(.primary) button {
      background: blue;
      color: white;
    }
    
    /* Style based on ancestor */
    :host-context(.dark-theme) button {
      background: #333;
      color: white;
    }
    
    /* Regular component styles */
    button {
      padding: 8px 16px;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
  `]
})
export class ThemedButtonComponent { }

// Usage:
// <app-themed-button class="primary">Primary</app-themed-button>
// <div class="dark-theme">
//   <app-themed-button>Dark button</app-themed-button>
// </div>
```

## Change Detection Strategies

Control how Angular checks for component changes.

### ChangeDetectionStrategy.Default

```typescript
@Component({
  selector: 'app-counter',
  template: `
    <div>
      <p>Count: {{ count }}</p>
      <button (click)="increment()">Increment</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.Default
})
export class CounterComponent {
  count = 0;
  
  increment() {
    this.count++;
    // Change detection runs automatically
  }
}
```

### ChangeDetectionStrategy.OnPush

```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <button (click)="edit()">Edit</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent {
  @Input() user!: User;
  
  edit() {
    console.log('Editing:', this.user);
  }
}

// Parent component
@Component({
  selector: 'app-user-list',
  template: `
    @for (user of users; track user.id) {
      <app-user-card [user]="user"></app-user-card>
    }
    <button (click)="updateUser()">Update User</button>
  `
})
export class UserListComponent {
  users: User[] = [
    { id: 1, name: 'Alice', email: 'alice@example.com' }
  ];
  
  updateUser() {
    // ❌ Won't trigger change detection in OnPush child
    this.users[0].name = 'Alice Updated';
    
    // ✅ Creates new reference, triggers change detection
    this.users = [
      { ...this.users[0], name: 'Alice Updated' }
    ];
  }
}
```

### Manual Change Detection

```typescript
import { Component, ChangeDetectorRef, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-manual-detection',
  template: `
    <div>
      <p>Value: {{ value }}</p>
      <button (click)="updateOutsideAngular()">Update</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ManualDetectionComponent {
  value = 0;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  updateOutsideAngular() {
    setTimeout(() => {
      this.value++;
      // Manually trigger change detection
      this.cdr.markForCheck();
    }, 1000);
  }
}
```

## Component Lifecycle

Components have a defined lifecycle managed by Angular.

### Lifecycle Overview

```typescript
import {
  Component,
  OnInit,
  OnChanges,
  OnDestroy,
  AfterViewInit,
  AfterContentInit,
  SimpleChanges
} from '@angular/core';

@Component({
  selector: 'app-lifecycle',
  template: `<div>Lifecycle component</div>`
})
export class LifecycleComponent implements
  OnInit,
  OnChanges,
  OnDestroy,
  AfterViewInit,
  AfterContentInit {
  
  constructor() {
    console.log('1. Constructor');
  }
  
  ngOnChanges(changes: SimpleChanges) {
    console.log('2. ngOnChanges', changes);
  }
  
  ngOnInit() {
    console.log('3. ngOnInit');
  }
  
  ngAfterContentInit() {
    console.log('4. ngAfterContentInit');
  }
  
  ngAfterViewInit() {
    console.log('5. ngAfterViewInit');
  }
  
  ngOnDestroy() {
    console.log('6. ngOnDestroy');
  }
}
```

## Host Bindings

Bind properties and events to the component's host element.

### Host Property Bindings

```typescript
@Component({
  selector: 'app-collapsible',
  template: `
    <div class="header" (click)="toggle()">
      {{ title }}
    </div>
    @if (expanded) {
      <div class="content">
        <ng-content></ng-content>
      </div>
    }
  `,
  host: {
    '[class.expanded]': 'expanded',
    '[attr.aria-expanded]': 'expanded',
    'role': 'button',
    'tabindex': '0'
  }
})
export class CollapsibleComponent {
  title = 'Click to expand';
  expanded = false;
  
  toggle() {
    this.expanded = !this.expanded;
  }
}
```

### Host Event Listeners

```typescript
@Component({
  selector: 'app-click-tracker',
  template: `<div>Clicks: {{ clickCount }}</div>`,
  host: {
    '(click)': 'onClick()',
    '(mouseenter)': 'onMouseEnter()',
    '(mouseleave)': 'onMouseLeave()'
  }
})
export class ClickTrackerComponent {
  clickCount = 0;
  
  onClick() {
    this.clickCount++;
  }
  
  onMouseEnter() {
    console.log('Mouse entered');
  }
  
  onMouseLeave() {
    console.log('Mouse left');
  }
}
```

## Common Mistakes

### 1. Forgetting standalone: true

```typescript
// ❌ WRONG: Standalone component without flag
@Component({
  selector: 'app-button',
  template: '<button>Click</button>'
})
export class ButtonComponent { }

// ✅ CORRECT
@Component({
  selector: 'app-button',
  standalone: true,
  template: '<button>Click</button>'
})
export class ButtonComponent { }
```

### 2. Using ViewEncapsulation.None Carelessly

```typescript
// ❌ WRONG: Global styles leak everywhere
@Component({
  selector: 'app-card',
  template: '<div class="card">Content</div>',
  styles: ['.card { color: red; }'],
  encapsulation: ViewEncapsulation.None
})
export class CardComponent { }

// ✅ CORRECT: Use Emulated or scope styles carefully
@Component({
  selector: 'app-card',
  template: '<div class="card">Content</div>',
  styles: ['.card { color: red; }'],
  encapsulation: ViewEncapsulation.Emulated
})
export class CardComponent { }
```

### 3. Mutating @Input Properties with OnPush

```typescript
// ❌ WRONG: Mutating input won't trigger change detection
@Component({
  selector: 'app-list',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListComponent {
  @Input() items: string[] = [];
  
  addItem(item: string) {
    this.items.push(item); // Won't trigger change detection!
  }
}

// ✅ CORRECT: Create new reference
@Component({
  selector: 'app-list',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListComponent {
  @Input() items: string[] = [];
  @Output() itemsChange = new EventEmitter<string[]>();
  
  addItem(item: string) {
    const newItems = [...this.items, item];
    this.itemsChange.emit(newItems);
  }
}
```

## Best Practices

### 1. Use OnPush for Performance

```typescript
// ✅ Optimize with OnPush
@Component({
  selector: 'app-product-card',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductCardComponent {
  @Input() product!: Product;
}
```

### 2. Keep Templates Simple

```typescript
// ❌ Complex logic in template
@Component({
  template: `
    <div>
      {{ users.filter(u => u.active).map(u => u.name.toUpperCase()).join(', ') }}
    </div>
  `
})

// ✅ Compute in component
@Component({
  template: `<div>{{ activeUserNames }}</div>`
})
export class UserListComponent {
  @Input() users: User[] = [];
  
  get activeUserNames(): string {
    return this.users
      .filter(u => u.active)
      .map(u => u.name.toUpperCase())
      .join(', ');
  }
}
```

### 3. Use External Templates for Complexity

```typescript
// ✅ External template for readability
@Component({
  selector: 'app-dashboard',
  templateUrl: './dashboard.component.html',
  styleUrls: ['./dashboard.component.css']
})
export class DashboardComponent { }
```

## Interview Questions

### Q1: What are the essential properties of @Component decorator?

**Answer:** Required: `selector` (defines how to use the component in HTML). Either `template` or `templateUrl` (inline or external template). Optional: `styles`/`styleUrls`, `encapsulation`, `changeDetection`, `standalone`, `imports`, `providers`, `host`.

### Q2: What's the difference between ViewEncapsulation modes?

**Answer:** `Emulated` (default) scopes styles using attribute selectors. `None` makes styles global, affecting the entire app. `ShadowDom` uses native Shadow DOM for true encapsulation. Emulated is recommended for most cases.

### Q3: When should you use ChangeDetectionStrategy.OnPush?

**Answer:** Use OnPush for performance optimization when: component only depends on @Input properties, inputs are immutable, or you want manual control over change detection. It reduces checks by only running when inputs change reference or events fire.

### Q4: What's the difference between template and templateUrl?

**Answer:** `template` is for inline templates (strings), best for small components. `templateUrl` references external HTML files, better for larger templates. Use template for simple components, templateUrl for complex ones.

### Q5: What are component selectors and their types?

**Answer:** Selectors define how to use components in HTML. Element selector (most common): `'app-button'` → `<app-button>`. Attribute: `'[appTooltip]'` → `<div appTooltip>`. Class: `'.app-card'` → `<div class="app-card">`. Element selectors are preferred.

## Key Takeaways

1. @Component decorator defines component metadata
2. selector, template/templateUrl are required
3. ViewEncapsulation.Emulated is default and recommended
4. OnPush improves performance but requires immutable inputs
5. Use templateUrl for complex templates
6. Host bindings attach to component element
7. :host and :host-context style the host element
8. Standalone components use imports array
9. Component lifecycle hooks manage initialization and cleanup
10. Keep templates simple, move logic to component class

## Resources

- [Angular Components Guide](https://angular.io/guide/component-overview)
- [Component Styles](https://angular.io/guide/component-styles)
- [View Encapsulation](https://angular.io/api/core/ViewEncapsulation)
- [Change Detection](https://angular.io/guide/change-detection)
- [Lifecycle Hooks](https://angular.io/guide/lifecycle-hooks)
