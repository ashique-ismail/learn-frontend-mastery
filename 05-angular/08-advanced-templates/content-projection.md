# Content Projection

## Overview

Content projection (also known as transclusion) is Angular's mechanism for inserting external content into a component's template. Using `ng-content`, you can create flexible, reusable components that accept dynamic content from parent components. This pattern enables composition-based architectures and is fundamental to building component libraries.

Content projection supports single-slot, multi-slot (with selectors), conditional projection, and programmatic access to projected content via `ContentChild` and `ContentChildren` decorators. Understanding content projection is essential for creating highly reusable components.

## Core Concepts

### 1. Basic Content Projection (Single Slot)

The simplest form uses `<ng-content>` to project all content into a single slot.

**Single Slot Projection:**

```typescript
// card.component.ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-content">
        <ng-content></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .card {
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 16px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
  `]
})
export class CardComponent {}

// Usage
@Component({
  selector: 'app-parent',
  template: `
    <app-card>
      <h2>Card Title</h2>
      <p>This content is projected into the card component.</p>
      <button>Action</button>
    </app-card>
  `
})
export class ParentComponent {}
```

### 2. Multi-Slot Projection (Named Slots)

Use selectors to project content into specific slots.

**Multi-Slot Projection:**

```typescript
// panel.component.ts
@Component({
  selector: 'app-panel',
  template: `
    <div class="panel">
      <div class="panel-header">
        <ng-content select="[slot=header]"></ng-content>
      </div>
      
      <div class="panel-body">
        <ng-content select="[slot=body]"></ng-content>
      </div>
      
      <div class="panel-footer" *ngIf="hasFooter">
        <ng-content select="[slot=footer]"></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .panel {
      border: 1px solid #ccc;
      border-radius: 4px;
    }
    .panel-header {
      background: #f5f5f5;
      padding: 12px;
      border-bottom: 1px solid #ccc;
    }
    .panel-body {
      padding: 16px;
    }
    .panel-footer {
      background: #f5f5f5;
      padding: 12px;
      border-top: 1px solid #ccc;
    }
  `]
})
export class PanelComponent {
  @Input() hasFooter = true;
}

// Usage
@Component({
  selector: 'app-usage',
  template: `
    <app-panel>
      <div slot="header">
        <h3>Panel Title</h3>
        <button>Close</button>
      </div>
      
      <div slot="body">
        <p>Main panel content goes here.</p>
        <ul>
          <li>Item 1</li>
          <li>Item 2</li>
        </ul>
      </div>
      
      <div slot="footer">
        <button>Cancel</button>
        <button>Save</button>
      </div>
    </app-panel>
  `
})
export class UsageComponent {}
```

### 3. Content Selection Strategies

Different selector types for targeting projected content.

**Selector Types:**

```typescript
// selector-demo.component.ts
@Component({
  selector: 'app-selector-demo',
  template: `
    <!-- CSS class selector -->
    <div class="header-section">
      <ng-content select=".header"></ng-content>
    </div>

    <!-- Element selector -->
    <div class="title-section">
      <ng-content select="h2"></ng-content>
    </div>

    <!-- Attribute selector -->
    <div class="actions-section">
      <ng-content select="[actions]"></ng-content>
    </div>

    <!-- Component selector -->
    <div class="icon-section">
      <ng-content select="app-icon"></ng-content>
    </div>

    <!-- Multiple selectors (comma-separated) -->
    <div class="interactive-section">
      <ng-content select="button, a"></ng-content>
    </div>

    <!-- Catch-all for unprojected content -->
    <div class="default-section">
      <ng-content></ng-content>
    </div>
  `
})
export class SelectorDemoComponent {}

// Usage
@Component({
  selector: 'app-selector-usage',
  template: `
    <app-selector-demo>
      <!-- Matches .header -->
      <div class="header">Header Content</div>

      <!-- Matches h2 -->
      <h2>Title</h2>

      <!-- Matches [actions] -->
      <div actions>
        <button>Edit</button>
        <button>Delete</button>
      </div>

      <!-- Matches app-icon -->
      <app-icon name="star"></app-icon>

      <!-- Matches button, a -->
      <button>Click Me</button>
      <a href="#">Link</a>

      <!-- Default (no selector match) -->
      <p>This goes to default ng-content</p>
    </app-selector-demo>
  `
})
export class SelectorUsageComponent {}
```

### 4. ngProjectAs

Use `ngProjectAs` when content is wrapped in structural directives.

**ngProjectAs Directive:**

```typescript
// tabs.component.ts
@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs">
      <div class="tab-headers">
        <ng-content select="app-tab-header"></ng-content>
      </div>
      <div class="tab-contents">
        <ng-content select="app-tab-content"></ng-content>
      </div>
    </div>
  `
})
export class TabsComponent {}

// Without ngProjectAs (doesn't work)
@Component({
  selector: 'app-tabs-wrong',
  template: `
    <app-tabs>
      <!-- ng-container prevents projection -->
      <ng-container *ngIf="showTab1">
        <app-tab-header>Tab 1</app-tab-header>
      </ng-container>
    </app-tabs>
  `
})
export class TabsWrongComponent {
  showTab1 = true;
}

// With ngProjectAs (works correctly)
@Component({
  selector: 'app-tabs-correct',
  template: `
    <app-tabs>
      <!-- ngProjectAs tells Angular how to project this content -->
      <ng-container *ngIf="showTab1" ngProjectAs="app-tab-header">
        <app-tab-header>Tab 1</app-tab-header>
      </ng-container>

      <ng-container *ngIf="showTab1" ngProjectAs="app-tab-content">
        <app-tab-content>Content 1</app-tab-content>
      </ng-container>
    </app-tabs>
  `
})
export class TabsCorrectComponent {
  showTab1 = true;
}
```

### 5. ContentChild and ContentChildren

Access projected content programmatically from the component class.

**ContentChild/ContentChildren:**

```typescript
// alert-icon.component.ts
@Component({
  selector: 'app-alert-icon',
  template: `<span class="icon">{{ icon }}</span>`
})
export class AlertIconComponent {
  @Input() icon = '!';
}

// alert.component.ts
@Component({
  selector: 'app-alert',
  template: `
    <div class="alert" [class]="'alert-' + type">
      <div class="alert-icon">
        <ng-content select="app-alert-icon"></ng-content>
      </div>
      <div class="alert-content">
        <ng-content></ng-content>
      </div>
      <button *ngIf="dismissible" (click)="onDismiss()" class="alert-close">
        ×
      </button>
    </div>
  `
})
export class AlertComponent implements AfterContentInit {
  @Input() type: 'success' | 'warning' | 'error' | 'info' = 'info';
  @Input() dismissible = true;

  // Access single projected component
  @ContentChild(AlertIconComponent) 
  icon!: AlertIconComponent;

  // Access all projected button elements
  @ContentChildren('alertButton', { read: ElementRef })
  buttons!: QueryList<ElementRef>;

  ngAfterContentInit(): void {
    // Content is available here
    if (this.icon) {
      console.log('Alert has custom icon:', this.icon.icon);
    }

    if (this.buttons) {
      console.log('Alert has', this.buttons.length, 'action buttons');
    }
  }

  onDismiss(): void {
    console.log('Alert dismissed');
  }
}

// Usage
@Component({
  selector: 'app-alert-usage',
  template: `
    <app-alert type="warning">
      <app-alert-icon icon="⚠"></app-alert-icon>
      
      <p>This is a warning message!</p>
      
      <button #alertButton>Action 1</button>
      <button #alertButton>Action 2</button>
    </app-alert>
  `
})
export class AlertUsageComponent {}
```

### 6. Conditional Content Projection

Project content conditionally based on component state.

**Conditional Projection:**

```typescript
// expandable.component.ts
@Component({
  selector: 'app-expandable',
  template: `
    <div class="expandable">
      <div class="expandable-header" (click)="toggle()">
        <ng-content select="[slot=header]"></ng-content>
        <span class="toggle-icon">{{ expanded ? '▼' : '▶' }}</span>
      </div>

      <div class="expandable-content" *ngIf="expanded" [@expandCollapse]>
        <ng-content select="[slot=content]"></ng-content>
      </div>

      <!-- Lazy projection: only projects when expanded -->
      <ng-container *ngIf="expanded">
        <div class="expandable-details">
          <ng-content select="[slot=details]"></ng-content>
        </div>
      </ng-container>
    </div>
  `,
  animations: [
    trigger('expandCollapse', [
      transition(':enter', [
        style({ height: 0, opacity: 0 }),
        animate('300ms ease-out', style({ height: '*', opacity: 1 }))
      ]),
      transition(':leave', [
        animate('300ms ease-in', style({ height: 0, opacity: 0 }))
      ])
    ])
  ]
})
export class ExpandableComponent {
  @Input() expanded = false;

  toggle(): void {
    this.expanded = !this.expanded;
  }
}

// Usage
@Component({
  selector: 'app-expandable-usage',
  template: `
    <app-expandable [expanded]="false">
      <div slot="header">
        <h3>Click to expand</h3>
      </div>

      <div slot="content">
        <p>This content is always available when expanded.</p>
      </div>

      <div slot="details">
        <p>This content is lazily projected only when expanded.</p>
        <app-heavy-component></app-heavy-component>
      </div>
    </app-expandable>
  `
})
export class ExpandableUsageComponent {}
```

### 7. Template Outlets with Projection

Combine content projection with template outlets for maximum flexibility.

**Template Outlet Projection:**

```typescript
// flexible-card.component.ts
@Component({
  selector: 'app-flexible-card',
  template: `
    <div class="card">
      <div class="card-header">
        <!-- Use custom template or projected content -->
        <ng-container *ngIf="headerTemplate; else defaultHeader">
          <ng-container *ngTemplateOutlet="headerTemplate; context: headerContext">
          </ng-container>
        </ng-container>
        <ng-template #defaultHeader>
          <ng-content select="[slot=header]"></ng-content>
        </ng-template>
      </div>

      <div class="card-body">
        <ng-container *ngIf="bodyTemplate; else defaultBody">
          <ng-container *ngTemplateOutlet="bodyTemplate; context: bodyContext">
          </ng-container>
        </ng-container>
        <ng-template #defaultBody>
          <ng-content></ng-content>
        </ng-template>
      </div>

      <div class="card-footer" *ngIf="footerTemplate || hasFooterContent">
        <ng-container *ngIf="footerTemplate; else defaultFooter">
          <ng-container *ngTemplateOutlet="footerTemplate; context: footerContext">
          </ng-container>
        </ng-container>
        <ng-template #defaultFooter>
          <ng-content select="[slot=footer]"></ng-content>
        </ng-template>
      </div>
    </div>
  `
})
export class FlexibleCardComponent {
  @Input() headerTemplate?: TemplateRef<any>;
  @Input() bodyTemplate?: TemplateRef<any>;
  @Input() footerTemplate?: TemplateRef<any>;

  @Input() headerContext: any = {};
  @Input() bodyContext: any = {};
  @Input() footerContext: any = {};

  @ContentChild('[slot=footer]') hasFooterContent: any;
}

// Usage with projection
@Component({
  selector: 'app-card-projection',
  template: `
    <app-flexible-card>
      <div slot="header">
        <h3>Projected Header</h3>
      </div>

      <p>Projected body content</p>

      <div slot="footer">
        <button>OK</button>
      </div>
    </app-flexible-card>
  `
})
export class CardProjectionComponent {}

// Usage with templates
@Component({
  selector: 'app-card-template',
  template: `
    <app-flexible-card
      [headerTemplate]="customHeader"
      [bodyTemplate]="customBody"
      [headerContext]="{ title: 'Dynamic Title' }"
      [bodyContext]="{ user: currentUser }">
    </app-flexible-card>

    <ng-template #customHeader let-title="title">
      <h3>{{ title }}</h3>
      <button>Edit</button>
    </ng-template>

    <ng-template #customBody let-user="user">
      <div class="user-info">
        <p>Name: {{ user.name }}</p>
        <p>Email: {{ user.email }}</p>
      </div>
    </ng-template>
  `
})
export class CardTemplateComponent {
  currentUser = { name: 'John Doe', email: 'john@example.com' };
}
```

### 8. Wrapper Components with Projection

Create wrapper components that enhance projected content.

**Wrapper Components:**

```typescript
// highlight-wrapper.component.ts
@Component({
  selector: 'app-highlight-wrapper',
  template: `
    <div class="highlight-container" [class.active]="isActive">
      <div class="highlight-badge" *ngIf="badge">
        {{ badge }}
      </div>
      <div class="highlight-content">
        <ng-content></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .highlight-container {
      position: relative;
      padding: 16px;
      border: 2px solid transparent;
      border-radius: 8px;
      transition: all 0.3s;
    }
    .highlight-container.active {
      border-color: #4CAF50;
      background-color: #f1f8f4;
    }
    .highlight-badge {
      position: absolute;
      top: -10px;
      right: -10px;
      background: #4CAF50;
      color: white;
      padding: 4px 8px;
      border-radius: 12px;
      font-size: 12px;
    }
  `]
})
export class HighlightWrapperComponent {
  @Input() isActive = false;
  @Input() badge?: string;
}

// list-wrapper.component.ts
@Component({
  selector: 'app-list-wrapper',
  template: `
    <div class="list-container">
      <div class="list-header" *ngIf="title">
        <h3>{{ title }}</h3>
        <span class="list-count">{{ itemCount }} items</span>
      </div>

      <div class="list-items" #listItems>
        <ng-content></ng-content>
      </div>

      <div class="list-footer" *ngIf="showFooter">
        <button (click)="onLoadMore()" *ngIf="hasMore">
          Load More
        </button>
      </div>
    </div>
  `
})
export class ListWrapperComponent implements AfterContentInit {
  @Input() title?: string;
  @Input() showFooter = true;
  @Input() hasMore = false;
  @Output() loadMore = new EventEmitter<void>();

  @ContentChildren('listItem') listItems!: QueryList<ElementRef>;

  itemCount = 0;

  ngAfterContentInit(): void {
    this.itemCount = this.listItems.length;

    // Watch for changes in projected content
    this.listItems.changes.subscribe(() => {
      this.itemCount = this.listItems.length;
    });
  }

  onLoadMore(): void {
    this.loadMore.emit();
  }
}

// Usage
@Component({
  selector: 'app-wrapper-usage',
  template: `
    <app-highlight-wrapper [isActive]="true" badge="New">
      <h3>Featured Product</h3>
      <p>Special offer!</p>
    </app-highlight-wrapper>

    <app-list-wrapper 
      title="Product List" 
      [hasMore]="true"
      (loadMore)="loadMoreProducts()">
      <div #listItem *ngFor="let product of products">
        {{ product.name }}
      </div>
    </app-list-wrapper>
  `
})
export class WrapperUsageComponent {
  products = [
    { name: 'Product 1' },
    { name: 'Product 2' },
    { name: 'Product 3' }
  ];

  loadMoreProducts(): void {
    // Load more products
  }
}
```

### 9. Detecting Projected Content

Check if content has been projected before rendering sections.

**Content Detection:**

```typescript
// dialog.component.ts
@Component({
  selector: 'app-dialog',
  template: `
    <div class="dialog-overlay" (click)="onOverlayClick()">
      <div class="dialog" (click)="$event.stopPropagation()">
        <!-- Only show header if content is projected -->
        <div class="dialog-header" *ngIf="hasHeader">
          <ng-content select="[slot=header]"></ng-content>
          <button class="dialog-close" (click)="onClose()">×</button>
        </div>

        <!-- Body always shown -->
        <div class="dialog-body" [class.no-header]="!hasHeader">
          <ng-content></ng-content>
        </div>

        <!-- Only show footer if content is projected -->
        <div class="dialog-footer" *ngIf="hasFooter">
          <ng-content select="[slot=footer]"></ng-content>
        </div>

        <!-- Show default footer if no footer projected -->
        <div class="dialog-footer" *ngIf="!hasFooter && showDefaultFooter">
          <button (click)="onClose()">Close</button>
        </div>
      </div>
    </div>
  `
})
export class DialogComponent implements AfterContentInit {
  @Input() showDefaultFooter = true;
  @Input() closeOnOverlay = true;
  @Output() close = new EventEmitter<void>();

  @ContentChild('[slot=header]') headerContent: any;
  @ContentChild('[slot=footer]') footerContent: any;

  hasHeader = false;
  hasFooter = false;

  ngAfterContentInit(): void {
    this.hasHeader = !!this.headerContent;
    this.hasFooter = !!this.footerContent;
  }

  onClose(): void {
    this.close.emit();
  }

  onOverlayClick(): void {
    if (this.closeOnOverlay) {
      this.onClose();
    }
  }
}

// Usage - with header and footer
@Component({
  selector: 'app-full-dialog',
  template: `
    <app-dialog (close)="onDialogClose()">
      <div slot="header">
        <h2>Dialog Title</h2>
      </div>

      <p>Dialog content here</p>

      <div slot="footer">
        <button (click)="onDialogClose()">Cancel</button>
        <button (click)="onSave()">Save</button>
      </div>
    </app-dialog>
  `
})
export class FullDialogComponent {
  onDialogClose(): void {}
  onSave(): void {}
}

// Usage - minimal (body only)
@Component({
  selector: 'app-minimal-dialog',
  template: `
    <app-dialog (close)="onDialogClose()">
      <p>Simple message</p>
    </app-dialog>
  `
})
export class MinimalDialogComponent {
  onDialogClose(): void {}
}
```

### 10. Advanced Projection Patterns

Complex projection patterns for sophisticated component designs.

**Advanced Patterns:**

```typescript
// data-table.component.ts
@Component({
  selector: 'app-data-table',
  template: `
    <div class="data-table">
      <div class="table-header">
        <ng-content select="app-table-column"></ng-content>
      </div>

      <div class="table-body">
        <div class="table-row" *ngFor="let row of data; let i = index">
          <ng-container *ngFor="let column of columns">
            <div class="table-cell">
              <ng-container *ngTemplateOutlet="
                column.template;
                context: { $implicit: row, index: i, column: column }
              "></ng-container>
            </div>
          </ng-container>
        </div>
      </div>

      <div class="table-footer" *ngIf="hasFooter">
        <ng-content select="[slot=footer]"></ng-content>
      </div>
    </div>
  `
})
export class DataTableComponent implements AfterContentInit {
  @Input() data: any[] = [];
  @ContentChildren(TableColumnComponent) columns!: QueryList<TableColumnComponent>;
  @ContentChild('[slot=footer]') footerContent: any;

  hasFooter = false;

  ngAfterContentInit(): void {
    this.hasFooter = !!this.footerContent;
  }
}

@Component({
  selector: 'app-table-column',
  template: `
    <ng-template #columnTemplate let-row let-index="index" let-column="column">
      <ng-content></ng-content>
    </ng-template>
  `
})
export class TableColumnComponent {
  @Input() header = '';
  @Input() field = '';
  @ViewChild('columnTemplate', { read: TemplateRef }) template!: TemplateRef<any>;
}

// Usage
@Component({
  selector: 'app-table-usage',
  template: `
    <app-data-table [data]="users">
      <app-table-column header="Name" field="name">
        <ng-template let-user>
          <strong>{{ user.name }}</strong>
        </ng-template>
      </app-table-column>

      <app-table-column header="Email" field="email">
        <ng-template let-user>
          <a [href]="'mailto:' + user.email">{{ user.email }}</a>
        </ng-template>
      </app-table-column>

      <app-table-column header="Actions" field="actions">
        <ng-template let-user let-index="index">
          <button (click)="editUser(user, index)">Edit</button>
          <button (click)="deleteUser(user, index)">Delete</button>
        </ng-template>
      </app-table-column>

      <div slot="footer">
        <p>Total users: {{ users.length }}</p>
      </div>
    </app-data-table>
  `
})
export class TableUsageComponent {
  users = [
    { name: 'John Doe', email: 'john@example.com' },
    { name: 'Jane Smith', email: 'jane@example.com' }
  ];

  editUser(user: any, index: number): void {}
  deleteUser(user: any, index: number): void {}
}
```

## Common Mistakes

### 1. Not Using Selectors for Multi-Slot Projection

```typescript
// WRONG: No selector, content goes to first ng-content
<ng-content></ng-content>
<ng-content></ng-content>

// CORRECT: Use selectors to target specific content
<ng-content select="[slot=header]"></ng-content>
<ng-content select="[slot=body]"></ng-content>
```

### 2. Forgetting ngProjectAs with Structural Directives

```typescript
// WRONG: Wrapped in *ngIf, won't project correctly
<ng-container *ngIf="condition">
  <div slot="header">Header</div>
</ng-container>

// CORRECT: Use ngProjectAs
<ng-container *ngIf="condition" ngProjectAs="[slot=header]">
  <div>Header</div>
</ng-container>
```

### 3. Accessing Projected Content Too Early

```typescript
// WRONG: Content not available in ngOnInit
ngOnInit() {
  console.log(this.projectedContent); // undefined
}

// CORRECT: Use ngAfterContentInit
ngAfterContentInit() {
  console.log(this.projectedContent); // available
}
```

### 4. Not Handling Empty Projection

```typescript
// WRONG: Footer always shows even if empty
<div class="footer">
  <ng-content select="[slot=footer]"></ng-content>
</div>

// CORRECT: Check if content exists
<div class="footer" *ngIf="hasFooter">
  <ng-content select="[slot=footer]"></ng-content>
</div>
```

### 5. Mixing Projection and Template Outlets Incorrectly

```typescript
// WRONG: Can't use both for same slot
<ng-content></ng-content>
<ng-container *ngTemplateOutlet="template"></ng-container>

// CORRECT: Use one or the other with fallback
<ng-container *ngIf="template; else projected">
  <ng-container *ngTemplateOutlet="template"></ng-container>
</ng-container>
<ng-template #projected>
  <ng-content></ng-content>
</ng-template>
```

## Best Practices

### 1. Use Descriptive Slot Names

```typescript
// Clear, semantic slot names
<ng-content select="[slot=header]"></ng-content>
<ng-content select="[slot=body]"></ng-content>
<ng-content select="[slot=footer]"></ng-content>
```

### 2. Provide Default Content

```typescript
<ng-content select="[slot=header]"></ng-content>
<ng-template #defaultHeader>
  <h3>Default Title</h3>
</ng-template>
```

### 3. Document Projection Slots

```typescript
/**
 * Card component with three projection slots:
 * - [slot=header]: Card header content
 * - Default: Card body content
 * - [slot=footer]: Card footer content
 */
@Component({...})
export class CardComponent {}
```

### 4. Check Content Existence

```typescript
@ContentChild('[slot=footer]') footer: any;

ngAfterContentInit() {
  this.hasFooter = !!this.footer;
}
```

### 5. Combine with Template Outlets

```typescript
// Provide both projection and template options
<ng-container *ngIf="customTemplate; else defaultContent">
  <ng-container *ngTemplateOutlet="customTemplate"></ng-container>
</ng-container>
<ng-template #defaultContent>
  <ng-content></ng-content>
</ng-template>
```

## Interview Questions

**Q: What is content projection in Angular?**

A: Content projection is Angular's mechanism for inserting external content into a component's template using `<ng-content>`. It enables creating flexible, reusable components that accept dynamic content from parent components.

**Q: What's the difference between ng-content and template outlets?**

A: `ng-content` projects content defined in the parent component's template, while template outlets (`ngTemplateOutlet`) render templates dynamically. ng-content is for static composition, outlets for dynamic templates.

**Q: When do you need ngProjectAs?**

A: Use `ngProjectAs` when content is wrapped in structural directives (`*ngIf`, `*ngFor`) that would prevent correct projection. It tells Angular how to project the wrapped content.

**Q: What's the difference between ContentChild and ViewChild?**

A: `@ContentChild` queries projected content (from parent), while `@ViewChild` queries elements in the component's own template. Content is available in `ngAfterContentInit`, view in `ngAfterViewInit`.

**Q: How do you implement multi-slot projection?**

A: Use selectors in `ng-content`: `<ng-content select="[slot=header]"></ng-content>`. Parent marks content with matching attributes: `<div slot="header">Content</div>`.

## Key Takeaways

1. `ng-content` enables content projection from parent to child components
2. Use selectors for multi-slot projection: `<ng-content select="[slot=name]">`
3. `ngProjectAs` fixes projection when content is wrapped in structural directives
4. Access projected content with `@ContentChild` and `@ContentChildren`
5. Projected content is available in `ngAfterContentInit` lifecycle hook
6. Check content existence before showing optional sections
7. Combine projection with template outlets for maximum flexibility
8. Use descriptive slot names for better developer experience
9. QueryList.changes subscribes to dynamic content changes
10. Content projection enables composition-based component architecture

## Resources

- [Angular Content Projection Guide](https://angular.io/guide/content-projection)
- [ng-content API](https://angular.io/api/core/ng-content)
- [ContentChild API](https://angular.io/api/core/ContentChild)
- [ContentChildren API](https://angular.io/api/core/ContentChildren)
- [Component Interaction Guide](https://angular.io/guide/component-interaction)
