# ng-template and ng-container

## The Idea

**In plain English:** `ng-template` and `ng-container` are two special tools in Angular that help you control what appears on a webpage without cluttering the page's structure. A template (`ng-template`) is a chunk of HTML you write but keep hidden until you choose to show it, while a container (`ng-container`) is an invisible wrapper that lets you apply rules to a group of elements without adding any extra box around them.

**Real-world analogy:** Imagine a restaurant with a menu board. The chef prepares several dish "templates" in the kitchen — recipes written on cards — but nothing is plated until a customer orders. The pass-through window is an invisible opening where finished plates appear without adding any extra counter space.

- The recipe card in the kitchen = `ng-template` (defined but not shown until needed)
- The act of the waiter calling "order up" = `ngTemplateOutlet` (the instruction that renders the template)
- The pass-through window (lets dishes through without being a table itself) = `ng-container` (groups or applies rules without adding a real DOM element)

---

## Overview

`ng-template` and `ng-container` are structural directives that provide powerful templating capabilities in Angular. They allow you to define reusable template fragments, conditional rendering without extra DOM elements, and dynamic template instantiation. Understanding these tools is essential for creating flexible, efficient Angular applications.

`ng-template` defines a template that isn't rendered by default but can be instantiated programmatically or by structural directives. `ng-container` is a logical grouping element that doesn't create a DOM element, making it perfect for applying directives without adding unnecessary markup.

## Core Concepts

### 1. ng-template Basics

`ng-template` defines a template that can be instantiated dynamically or used with structural directives.

**Basic ng-template:**

```typescript
// basic-template.component.ts
@Component({
  selector: 'app-basic-template',
  template: `
    <!-- Template definition - not rendered by default -->
    <ng-template #myTemplate>
      <div class="template-content">
        <h3>Template Content</h3>
        <p>This content is defined in ng-template</p>
      </div>
    </ng-template>

    <!-- Render template using ngTemplateOutlet -->
    <ng-container *ngTemplateOutlet="myTemplate"></ng-container>

    <!-- Render template multiple times -->
    <ng-container *ngTemplateOutlet="myTemplate"></ng-container>
    <ng-container *ngTemplateOutlet="myTemplate"></ng-container>

    <!-- Conditional template rendering -->
    <button (click)="showTemplate = !showTemplate">Toggle Template</button>
    <ng-container *ngIf="showTemplate">
      <ng-container *ngTemplateOutlet="myTemplate"></ng-container>
    </ng-container>
  `
})
export class BasicTemplateComponent {
  showTemplate = false;
}
```

**Template with Context:**

```typescript
// template-context.component.ts
@Component({
  selector: 'app-template-context',
  template: `
    <!-- Template with context variables -->
    <ng-template #userTemplate let-userName let-userAge="age" let-userEmail="email">
      <div class="user-card">
        <h3>{{ userName }}</h3>
        <p>Age: {{ userAge }}</p>
        <p>Email: {{ userEmail }}</p>
      </div>
    </ng-template>

    <!-- Render with context -->
    <ng-container *ngTemplateOutlet="
      userTemplate;
      context: {
        $implicit: 'John Doe',
        age: 30,
        email: 'john@example.com'
      }
    "></ng-container>

    <ng-container *ngTemplateOutlet="
      userTemplate;
      context: {
        $implicit: 'Jane Smith',
        age: 25,
        email: 'jane@example.com'
      }
    "></ng-container>
  `
})
export class TemplateContextComponent {}
```

### 2. ng-container Basics

`ng-container` is a logical container that doesn't create a DOM element.

**Using ng-container:**

```typescript
// ng-container.component.ts
@Component({
  selector: 'app-ng-container',
  template: `
    <!-- WRONG: Extra div wrapper -->
    <div *ngIf="showContent">
      <h3>Title</h3>
      <p>Content</p>
    </div>

    <!-- CORRECT: No extra DOM element -->
    <ng-container *ngIf="showContent">
      <h3>Title</h3>
      <p>Content</p>
    </ng-container>

    <!-- Multiple structural directives (not allowed on same element) -->
    <!-- WRONG: Can't use multiple structural directives -->
    <!-- <div *ngIf="condition" *ngFor="let item of items"></div> -->

    <!-- CORRECT: Nest with ng-container -->
    <ng-container *ngIf="condition">
      <div *ngFor="let item of items">{{ item }}</div>
    </ng-container>

    <!-- Grouping without adding DOM element -->
    <ng-container>
      <h3>Section Title</h3>
      <p>Section content paragraph 1</p>
      <p>Section content paragraph 2</p>
    </ng-container>

    <!-- Using with ngSwitch -->
    <ng-container [ngSwitch]="status">
      <p *ngSwitchCase="'loading'">Loading...</p>
      <p *ngSwitchCase="'error'">Error occurred</p>
      <p *ngSwitchCase="'success'">Success!</p>
      <p *ngSwitchDefault>Unknown status</p>
    </ng-container>
  `
})
export class NgContainerComponent {
  showContent = true;
  condition = true;
  items = ['Item 1', 'Item 2', 'Item 3'];
  status: 'loading' | 'error' | 'success' | 'idle' = 'loading';
}
```

### 3. TemplateRef

`TemplateRef` represents an embedded template that can be instantiated programmatically.

**Working with TemplateRef:**

```typescript
// template-ref.component.ts
import { Component, TemplateRef, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-template-ref',
  template: `
    <ng-template #dynamicTemplate let-message>
      <div class="alert">{{ message }}</div>
    </ng-template>

    <button (click)="showTemplate()">Show Template</button>
    <button (click)="hideTemplate()">Hide Template</button>

    <div #container></div>
  `
})
export class TemplateRefComponent {
  @ViewChild('dynamicTemplate', { read: TemplateRef })
  templateRef!: TemplateRef<any>;

  @ViewChild('container', { read: ViewContainerRef })
  containerRef!: ViewContainerRef;

  showTemplate(): void {
    // Clear existing views
    this.containerRef.clear();

    // Create embedded view from template
    this.containerRef.createEmbeddedView(this.templateRef, {
      $implicit: 'This is a dynamic message!'
    });
  }

  hideTemplate(): void {
    this.containerRef.clear();
  }
}
```

**Multiple Template Instantiation:**

```typescript
// multiple-templates.component.ts
@Component({
  selector: 'app-multiple-templates',
  template: `
    <ng-template #itemTemplate let-item let-index="index">
      <div class="item">
        <span>{{ index + 1 }}.</span>
        <strong>{{ item.name }}</strong>
        <p>{{ item.description }}</p>
      </div>
    </ng-template>

    <button (click)="renderItems()">Render Items</button>
    <button (click)="clearItems()">Clear Items</button>

    <div #itemsContainer></div>
  `
})
export class MultipleTemplatesComponent {
  @ViewChild('itemTemplate', { read: TemplateRef })
  itemTemplate!: TemplateRef<any>;

  @ViewChild('itemsContainer', { read: ViewContainerRef })
  itemsContainer!: ViewContainerRef;

  items = [
    { name: 'Item 1', description: 'Description 1' },
    { name: 'Item 2', description: 'Description 2' },
    { name: 'Item 3', description: 'Description 3' }
  ];

  renderItems(): void {
    this.itemsContainer.clear();

    this.items.forEach((item, index) => {
      this.itemsContainer.createEmbeddedView(this.itemTemplate, {
        $implicit: item,
        index
      });
    });
  }

  clearItems(): void {
    this.itemsContainer.clear();
  }
}
```

### 4. ViewContainerRef

`ViewContainerRef` is a container where views can be attached and managed.

**ViewContainerRef Operations:**

```typescript
// view-container.component.ts
import { Component, ViewChild, ViewContainerRef, TemplateRef, ComponentRef } from '@angular/core';

@Component({
  selector: 'app-dynamic-alert',
  template: `
    <div class="alert alert-{{ type }}">
      {{ message }}
      <button (click)="close()">×</button>
    </div>
  `
})
export class DynamicAlertComponent {
  type = 'info';
  message = '';
  close = () => {};
}

@Component({
  selector: 'app-view-container',
  template: `
    <ng-template #alertTemplate let-message let-type="type">
      <div class="alert alert-{{ type }}">
        {{ message }}
      </div>
    </ng-template>

    <button (click)="addTemplateView()">Add Template View</button>
    <button (click)="addComponentView()">Add Component View</button>
    <button (click)="removeView(0)">Remove First View</button>
    <button (click)="clearAllViews()">Clear All</button>
    <button (click)="moveView()">Move First to Last</button>

    <div #viewContainer></div>
  `
})
export class ViewContainerComponent {
  @ViewChild('alertTemplate', { read: TemplateRef })
  alertTemplate!: TemplateRef<any>;

  @ViewChild('viewContainer', { read: ViewContainerRef })
  viewContainer!: ViewContainerRef;

  private viewCount = 0;

  addTemplateView(): void {
    this.viewContainer.createEmbeddedView(this.alertTemplate, {
      $implicit: `Template View ${++this.viewCount}`,
      type: 'success'
    });
  }

  addComponentView(): void {
    const componentRef = this.viewContainer.createComponent(DynamicAlertComponent);
    componentRef.instance.message = `Component View ${++this.viewCount}`;
    componentRef.instance.type = 'warning';
    componentRef.instance.close = () => {
      const index = this.viewContainer.indexOf(componentRef.hostView);
      if (index !== -1) {
        this.viewContainer.remove(index);
      }
    };
  }

  removeView(index: number): void {
    if (index >= 0 && index < this.viewContainer.length) {
      this.viewContainer.remove(index);
    }
  }

  clearAllViews(): void {
    this.viewContainer.clear();
    this.viewCount = 0;
  }

  moveView(): void {
    if (this.viewContainer.length > 1) {
      const view = this.viewContainer.get(0);
      if (view) {
        this.viewContainer.move(view, this.viewContainer.length - 1);
      }
    }
  }
}
```

### 5. Dynamic Templates

Create and manage templates dynamically based on application state.

**Dynamic Template Selection:**

```typescript
// dynamic-templates.component.ts
@Component({
  selector: 'app-dynamic-templates',
  template: `
    <!-- Define multiple templates -->
    <ng-template #listView let-items>
      <ul>
        <li *ngFor="let item of items">{{ item.name }}</li>
      </ul>
    </ng-template>

    <ng-template #gridView let-items>
      <div class="grid">
        <div *ngFor="let item of items" class="grid-item">
          <h4>{{ item.name }}</h4>
          <p>{{ item.description }}</p>
        </div>
      </div>
    </ng-template>

    <ng-template #tableView let-items>
      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Description</th>
            <th>Price</th>
          </tr>
        </thead>
        <tbody>
          <tr *ngFor="let item of items">
            <td>{{ item.name }}</td>
            <td>{{ item.description }}</td>
            <td>{{ item.price | currency }}</td>
          </tr>
        </tbody>
      </table>
    </ng-template>

    <!-- View switcher -->
    <div class="view-controls">
      <button (click)="currentView = 'list'" [class.active]="currentView === 'list'">
        List View
      </button>
      <button (click)="currentView = 'grid'" [class.active]="currentView === 'grid'">
        Grid View
      </button>
      <button (click)="currentView = 'table'" [class.active]="currentView === 'table'">
        Table View
      </button>
    </div>

    <!-- Render selected template -->
    <ng-container [ngSwitch]="currentView">
      <ng-container *ngSwitchCase="'list'">
        <ng-container *ngTemplateOutlet="listView; context: { $implicit: items }">
        </ng-container>
      </ng-container>
      <ng-container *ngSwitchCase="'grid'">
        <ng-container *ngTemplateOutlet="gridView; context: { $implicit: items }">
        </ng-container>
      </ng-container>
      <ng-container *ngSwitchCase="'table'">
        <ng-container *ngTemplateOutlet="tableView; context: { $implicit: items }">
        </ng-container>
      </ng-container>
    </ng-container>
  `
})
export class DynamicTemplatesComponent {
  currentView: 'list' | 'grid' | 'table' = 'list';

  items = [
    { name: 'Product 1', description: 'Description 1', price: 29.99 },
    { name: 'Product 2', description: 'Description 2', price: 39.99 },
    { name: 'Product 3', description: 'Description 3', price: 49.99 }
  ];
}
```

### 6. Template Outlets

Use `ngTemplateOutlet` to render templates dynamically with context.

**Template Outlet Examples:**

```typescript
// template-outlet.component.ts
@Component({
  selector: 'app-template-outlet',
  template: `
    <!-- Default template -->
    <ng-template #defaultTemplate>
      <p>Default content</p>
    </ng-template>

    <!-- Custom template -->
    <ng-template #customTemplate let-title let-body="body">
      <div class="custom">
        <h3>{{ title }}</h3>
        <div [innerHTML]="body"></div>
      </div>
    </ng-template>

    <!-- Use template outlet with fallback -->
    <ng-container *ngTemplateOutlet="selectedTemplate || defaultTemplate; context: templateContext">
    </ng-container>

    <button (click)="useDefault()">Use Default</button>
    <button (click)="useCustom()">Use Custom</button>
  `
})
export class TemplateOutletComponent {
  @ViewChild('defaultTemplate', { read: TemplateRef })
  defaultTemplate!: TemplateRef<any>;

  @ViewChild('customTemplate', { read: TemplateRef })
  customTemplate!: TemplateRef<any>;

  selectedTemplate: TemplateRef<any> | null = null;

  templateContext = {
    $implicit: 'Custom Title',
    body: '<p>Custom <strong>HTML</strong> content</p>'
  };

  useDefault(): void {
    this.selectedTemplate = this.defaultTemplate;
  }

  useCustom(): void {
    this.selectedTemplate = this.customTemplate;
  }
}
```

**Reusable Card Component with Template Outlet:**

```typescript
// card.component.ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-header">
        <ng-container *ngTemplateOutlet="headerTemplate || defaultHeader; context: headerContext">
        </ng-container>
      </div>

      <div class="card-body">
        <ng-container *ngTemplateOutlet="bodyTemplate || defaultBody; context: bodyContext">
        </ng-container>
      </div>

      <div class="card-footer" *ngIf="footerTemplate">
        <ng-container *ngTemplateOutlet="footerTemplate; context: footerContext">
        </ng-container>
      </div>
    </div>

    <!-- Default templates -->
    <ng-template #defaultHeader>
      <h3>Default Header</h3>
    </ng-template>

    <ng-template #defaultBody>
      <p>Default body content</p>
    </ng-template>
  `
})
export class CardComponent {
  @Input() headerTemplate?: TemplateRef<any>;
  @Input() bodyTemplate?: TemplateRef<any>;
  @Input() footerTemplate?: TemplateRef<any>;

  @Input() headerContext: any = {};
  @Input() bodyContext: any = {};
  @Input() footerContext: any = {};
}

// Usage component
@Component({
  selector: 'app-card-usage',
  template: `
    <app-card
      [headerTemplate]="customHeader"
      [bodyTemplate]="customBody"
      [footerTemplate]="customFooter"
      [bodyContext]="{ user: currentUser }">
    </app-card>

    <ng-template #customHeader>
      <h3>User Profile</h3>
      <button>Edit</button>
    </ng-template>

    <ng-template #customBody let-user="user">
      <div class="user-info">
        <p>Name: {{ user.name }}</p>
        <p>Email: {{ user.email }}</p>
        <p>Role: {{ user.role }}</p>
      </div>
    </ng-template>

    <ng-template #customFooter>
      <button>Save</button>
      <button>Cancel</button>
    </ng-template>
  `
})
export class CardUsageComponent {
  currentUser = {
    name: 'John Doe',
    email: 'john@example.com',
    role: 'Admin'
  };
}
```

### 7. Structural Directive Hosts

Use `ng-template` as the host for custom structural directives.

**Custom Structural Directive with ng-template:**

```typescript
// unless.directive.ts
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appUnless]',
  standalone: true
})
export class UnlessDirective {
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Usage
@Component({
  selector: 'app-unless-demo',
  imports: [UnlessDirective],
  template: `
    <button (click)="condition = !condition">
      Toggle ({{ condition }})
    </button>

    <div *appUnless="condition">
      This content is shown when condition is false
    </div>
  `
})
export class UnlessDemoComponent {
  condition = false;
}
```

### 8. Template Context Typing

Type-safe template contexts for better IDE support and compile-time checking.

**Typed Template Context:**

```typescript
// typed-template.component.ts
interface UserContext {
  $implicit: User;
  index: number;
  count: number;
  first: boolean;
  last: boolean;
}

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Component({
  selector: 'app-typed-template',
  template: `
    <ng-template #userTemplate let-user let-i="index" let-count="count" let-first="first" let-last="last">
      <div class="user-item" [class.first]="first" [class.last]="last">
        <span class="index">{{ i + 1 }} / {{ count }}</span>
        <h4>{{ user.name }}</h4>
        <p>{{ user.email }}</p>
        <span class="badge">{{ user.role }}</span>
      </div>
    </ng-template>

    <ng-container *ngFor="let user of users; let i = index; let c = count; let f = first; let l = last">
      <ng-container *ngTemplateOutlet="
        userTemplate;
        context: {
          $implicit: user,
          index: i,
          count: c,
          first: f,
          last: l
        }
      "></ng-container>
    </ng-container>
  `
})
export class TypedTemplateComponent {
  users: User[] = [
    { id: 1, name: 'John Doe', email: 'john@example.com', role: 'Admin' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'User' },
    { id: 3, name: 'Bob Johnson', email: 'bob@example.com', role: 'Moderator' }
  ];
}

// Strongly typed template directive
@Directive({
  selector: '[appTypedTemplate]',
  standalone: true
})
export class TypedTemplateDirective<T> {
  @Input('appTypedTemplate') data: T[] = [];
  @Input('appTypedTemplateTemplate') template!: TemplateRef<TypedContext<T>>;

  constructor(private viewContainer: ViewContainerRef) {}

  ngOnInit(): void {
    this.render();
  }

  ngOnChanges(): void {
    this.render();
  }

  private render(): void {
    this.viewContainer.clear();

    this.data.forEach((item, index) => {
      this.viewContainer.createEmbeddedView(this.template, {
        $implicit: item,
        index,
        count: this.data.length,
        first: index === 0,
        last: index === this.data.length - 1
      });
    });
  }
}

interface TypedContext<T> {
  $implicit: T;
  index: number;
  count: number;
  first: boolean;
  last: boolean;
}
```

### 9. Lazy Template Loading

Load templates lazily for performance optimization.

**Lazy Template Loading:**

```typescript
// lazy-template.component.ts
@Component({
  selector: 'app-lazy-template',
  template: `
    <button (click)="loadTemplate()">Load Template</button>

    <div #templateContainer></div>

    <ng-template #heavyTemplate>
      <div class="heavy-content">
        <!-- Complex, resource-intensive content -->
        <app-chart *ngFor="let data of chartData" [data]="data"></app-chart>
      </div>
    </ng-template>
  `
})
export class LazyTemplateComponent {
  @ViewChild('heavyTemplate', { read: TemplateRef })
  heavyTemplate!: TemplateRef<any>;

  @ViewChild('templateContainer', { read: ViewContainerRef })
  templateContainer!: ViewContainerRef;

  private templateLoaded = false;

  chartData = Array(10).fill(null).map(() => ({
    // Chart data...
  }));

  loadTemplate(): void {
    if (this.templateLoaded) {
      return;
    }

    // Load template only when needed
    this.templateContainer.createEmbeddedView(this.heavyTemplate);
    this.templateLoaded = true;
  }
}
```

### 10. Template Composition

Compose complex templates from smaller template parts.

**Template Composition:**

```typescript
// template-composition.component.ts
@Component({
  selector: 'app-template-composition',
  template: `
    <!-- Define template parts -->
    <ng-template #headerPart let-title>
      <div class="header">
        <h2>{{ title }}</h2>
        <ng-content select="[slot=header-actions]"></ng-content>
      </div>
    </ng-template>

    <ng-template #contentPart let-items>
      <div class="content">
        <div *ngFor="let item of items" class="item">
          {{ item.name }}
        </div>
      </div>
    </ng-template>

    <ng-template #footerPart let-total>
      <div class="footer">
        <span>Total items: {{ total }}</span>
        <ng-content select="[slot=footer-actions]"></ng-content>
      </div>
    </ng-template>

    <!-- Compose full template -->
    <div class="composed-view">
      <ng-container *ngTemplateOutlet="headerPart; context: { $implicit: title }">
      </ng-container>

      <ng-container *ngTemplateOutlet="contentPart; context: { $implicit: items }">
      </ng-container>

      <ng-container *ngTemplateOutlet="footerPart; context: { $implicit: items.length }">
      </ng-container>
    </div>
  `
})
export class TemplateCompositionComponent {
  @Input() title = 'Composed View';
  @Input() items: any[] = [];
}

// Usage
@Component({
  selector: 'app-composition-usage',
  template: `
    <app-template-composition [title]="'My Products'" [items]="products">
      <button slot="header-actions" (click)="addProduct()">Add Product</button>
      <button slot="footer-actions" (click)="clearAll()">Clear All</button>
    </app-template-composition>
  `
})
export class CompositionUsageComponent {
  products = [
    { name: 'Product 1' },
    { name: 'Product 2' },
    { name: 'Product 3' }
  ];

  addProduct(): void {
    this.products.push({ name: `Product ${this.products.length + 1}` });
  }

  clearAll(): void {
    this.products = [];
  }
}
```

## Common Mistakes

### 1. Trying to Style ng-template

```typescript
// WRONG: ng-template doesn't render
<ng-template style="color: red">Content</ng-template>

// CORRECT: Style the content inside
<ng-template #tmpl>
  <div style="color: red">Content</div>
</ng-template>
```

### 2. Using ng-container When Not Needed

```typescript
// WRONG: Unnecessary ng-container
<ng-container>
  <div>Single element</div>
</ng-container>

// CORRECT: Just use the element
<div>Single element</div>
```

### 3. Forgetting $implicit in Context

```typescript
// WRONG: Missing $implicit
<ng-template #tmpl let-value="value">{{ value }}</ng-template>
<ng-container *ngTemplateOutlet="tmpl; context: { value: 'test' }"></ng-container>

// CORRECT: Use $implicit for unnamed let
<ng-template #tmpl let-value>{{ value }}</ng-template>
<ng-container *ngTemplateOutlet="tmpl; context: { $implicit: 'test' }"></ng-container>
```

### 4. Not Clearing ViewContainerRef

```typescript
// WRONG: Views accumulate
addView(): void {
  this.viewContainer.createEmbeddedView(this.template);
}

// CORRECT: Clear before adding
addView(): void {
  this.viewContainer.clear();
  this.viewContainer.createEmbeddedView(this.template);
}
```

### 5. Multiple Structural Directives on Same Element

```typescript
// WRONG: Multiple structural directives
<div *ngIf="condition" *ngFor="let item of items">{{ item }}</div>

// CORRECT: Nest with ng-container
<ng-container *ngIf="condition">
  <div *ngFor="let item of items">{{ item }}</div>
</ng-container>
```

## Best Practices

### 1. Use ng-container for Structural Directives

```typescript
// Avoid unnecessary DOM elements
<ng-container *ngIf="condition">
  <h3>Title</h3>
  <p>Content</p>
</ng-container>
```

### 2. Name Templates Descriptively

```typescript
// Clear, descriptive template names
<ng-template #userProfileTemplate>...</ng-template>
<ng-template #loadingStateTemplate>...</ng-template>
```

### 3. Type Template Contexts

```typescript
interface TemplateContext {
  $implicit: User;
  index: number;
}

template: TemplateRef<TemplateContext>;
```

### 4. Clear ViewContainerRef When Done

```typescript
ngOnDestroy(): void {
  this.viewContainer.clear();
}
```

### 5. Use TemplateRef for Reusable Templates

```typescript
@Input() customTemplate?: TemplateRef<any>;

// Render custom or default template
<ng-container *ngTemplateOutlet="customTemplate || defaultTemplate">
</ng-container>
```

## Interview Questions

**Q: What's the difference between ng-template and ng-container?**

A: `ng-template` defines a template that's not rendered by default and can be instantiated programmatically. `ng-container` is a logical grouping element that doesn't create a DOM element but can have structural directives applied.

**Q: Why can't you use multiple structural directives on the same element?**

A: Each structural directive transforms the element's template differently. Multiple directives would conflict. Solution: nest elements or use `ng-container`.

**Q: What is $implicit in template context?**

A: `$implicit` is the default property name in a template context. When you use `let-variable` without `=`, it binds to `$implicit`: `let-user` binds to `context.$implicit`.

**Q: How do you instantiate a template programmatically?**

A: Inject `ViewContainerRef` and `TemplateRef`, then call `viewContainer.createEmbeddedView(templateRef, context)`.

**Q: When should you use ng-container vs a regular div?**

A: Use `ng-container` when you need to apply directives or group elements without adding an extra DOM element. Use `div` when you need styling or semantic structure.

## Key Takeaways

1. `ng-template` defines templates that aren't rendered by default
2. `ng-container` groups elements without creating DOM nodes
3. Use `ngTemplateOutlet` to render templates dynamically
4. `TemplateRef` represents a template reference that can be instantiated
5. `ViewContainerRef` is a container where views are attached
6. Template context passes data to templates via `$implicit` and named properties
7. Can't use multiple structural directives on same element - use `ng-container` to nest
8. `createEmbeddedView()` instantiates templates programmatically
9. Clear `ViewContainerRef` to remove all views and prevent memory leaks
10. Type template contexts for better type safety and IDE support

## Resources

- [Angular ng-template Documentation](https://angular.io/api/core/ng-template)
- [ng-container API](https://angular.io/api/core/ng-container)
- [TemplateRef API](https://angular.io/api/core/TemplateRef)
- [ViewContainerRef API](https://angular.io/api/core/ViewContainerRef)
- [Structural Directives Guide](https://angular.io/guide/structural-directives)
