# Content Projection Implementation

## Table of Contents
- [Introduction](#introduction)
- [Content Projection Basics](#content-projection-basics)
- [ViewContainerRef and ViewRef](#viewcontainerref-and-viewref)
- [ng-content Internals](#ng-content-internals)
- [Multi-Slot Projection](#multi-slot-projection)
- [Conditional Content Projection](#conditional-content-projection)
- [Content Projection with Templates](#content-projection-with-templates)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Content projection is Angular's mechanism for inserting external content into a component's template. Understanding how content projection works internally reveals Angular's powerful composition model and how it manages view hierarchies, change detection, and component boundaries.

At its core, content projection involves:
- **Slot-based insertion**: Using `ng-content` to define insertion points
- **View hierarchy management**: Maintaining parent-child relationships across projection boundaries
- **Change detection coordination**: Ensuring projected content participates in change detection
- **Lifecycle management**: Coordinating lifecycle hooks between host and projected content

## Content Projection Basics

### How ng-content Works

When Angular compiles a component with `ng-content`, it creates a projection definition that maps content nodes to slots:

```typescript
// Component definition
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-header">
        <ng-content select="[card-header]"></ng-content>
      </div>
      <div class="card-body">
        <ng-content select="[card-body]"></ng-content>
      </div>
      <div class="card-footer">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class CardComponent {}

// Usage
@Component({
  template: `
    <app-card>
      <h2 card-header>Title</h2>
      <p card-body>Content</p>
      <button>Action</button>
    </app-card>
  `
})
export class AppComponent {}
```

### Internal Compilation Output

Angular's compiler transforms `ng-content` into projection instructions:

```typescript
// Simplified compiled output
class CardComponent {
  static ɵcmp = defineComponent({
    type: CardComponent,
    selectors: [['app-card']],
    ngContentSelectors: ['[card-header]', '[card-body]', '*'],
    decls: 8,
    vars: 0,
    template: function CardComponent_Template(rf, ctx) {
      if (rf & 1) {
        // Create projection definitions
        projectionDef(['[card-header]', '[card-body]', '*']);
        
        elementStart(0, 'div', 0); // card container
        elementStart(1, 'div', 1); // card-header
        projection(2, 0); // Project slot 0
        elementEnd();
        elementStart(3, 'div', 2); // card-body
        projection(4, 1); // Project slot 1
        elementEnd();
        elementStart(5, 'div', 3); // card-footer
        projection(6, 2); // Project default slot
        elementEnd();
        elementEnd();
      }
    }
  });
}
```

### Projection Node Distribution

Angular distributes nodes to slots during component creation:

```typescript
// Pseudo-code for projection distribution
function distributeProjectableNodes(
  componentRef: ComponentRef<any>,
  ngContentSelectors: string[]
): Node[][] {
  const slots: Node[][] = Array(ngContentSelectors.length)
    .fill(null)
    .map(() => []);
  
  const childNodes = componentRef.location.nativeElement.childNodes;
  
  for (const node of childNodes) {
    let matched = false;
    
    // Try to match against each selector
    for (let i = 0; i < ngContentSelectors.length - 1; i++) {
      if (matchesSelector(node, ngContentSelectors[i])) {
        slots[i].push(node);
        matched = true;
        break;
      }
    }
    
    // If no match, add to default slot (last one)
    if (!matched) {
      slots[ngContentSelectors.length - 1].push(node);
    }
  }
  
  return slots;
}
```

## ViewContainerRef and ViewRef

### Understanding ViewContainerRef

`ViewContainerRef` represents a container where views can be attached:

```typescript
@Component({
  selector: 'app-dynamic-container',
  template: `
    <div class="container">
      <ng-container #dynamicContainer></ng-container>
    </div>
  `
})
export class DynamicContainerComponent implements OnInit {
  @ViewChild('dynamicContainer', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  constructor(private componentFactoryResolver: ComponentFactoryResolver) {}
  
  ngOnInit() {
    // Create component dynamically
    const factory = this.componentFactoryResolver
      .resolveComponentFactory(DynamicComponent);
    
    // Insert at specific index
    const componentRef = this.container.createComponent(factory, 0);
    
    console.log('View created:', componentRef.hostView);
    console.log('Container length:', this.container.length);
  }
  
  addTemplate(template: TemplateRef<any>, context?: any) {
    // Create embedded view from template
    const viewRef = this.container.createEmbeddedView(template, context);
    
    return viewRef;
  }
  
  removeView(index: number) {
    this.container.remove(index);
  }
  
  moveView(viewRef: ViewRef, newIndex: number) {
    this.container.move(viewRef, newIndex);
  }
  
  clearAll() {
    this.container.clear();
  }
}
```

### ViewRef Hierarchy

Understanding the view hierarchy is crucial:

```typescript
@Component({
  selector: 'app-view-hierarchy',
  template: `
    <div>
      <h2>Parent View</h2>
      <ng-container #container></ng-container>
    </div>
  `
})
export class ViewHierarchyComponent implements AfterViewInit {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  ngAfterViewInit() {
    // Get the component's view
    const componentView = this.container.injector.get(ChangeDetectorRef);
    
    console.log('Component view:', componentView);
    
    // Create embedded view
    const template = this.createTemplate();
    const embeddedView = this.container.createEmbeddedView(template);
    
    console.log('Embedded view:', embeddedView);
    console.log('Parent view:', embeddedView.rootNodes[0].parentNode);
    
    // View hierarchy:
    // ComponentView
    //   └─ EmbeddedView (from template)
    //       └─ Nodes (DOM elements)
  }
  
  private createTemplate(): TemplateRef<any> {
    // This would typically come from @ViewChild or parameter
    return null as any;
  }
}
```

### View Attachment and Detachment

Views can be attached and detached from change detection:

```typescript
@Component({
  selector: 'app-detachable-view',
  template: `
    <div>
      <button (click)="toggle()">{{ attached ? 'Detach' : 'Attach' }}</button>
      <button (click)="updateData()">Update Data</button>
      <p>{{ data }}</p>
    </div>
  `
})
export class DetachableViewComponent {
  data = 0;
  attached = true;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  toggle() {
    if (this.attached) {
      // Detach from change detection
      this.cdr.detach();
      this.attached = false;
    } else {
      // Reattach to change detection
      this.cdr.reattach();
      this.attached = true;
    }
  }
  
  updateData() {
    this.data++;
    
    if (!this.attached) {
      // Manually trigger change detection when detached
      this.cdr.detectChanges();
    }
  }
}
```

## ng-content Internals

### Projection Definition Creation

Angular creates projection definitions during compilation:

```typescript
// How projectionDef works internally
interface ProjectionDef {
  selectors: CssSelector[][];
  slots: number;
}

function projectionDef(selectors?: string[]): ProjectionDef {
  const parsed = selectors?.map(selector => {
    return parseSelector(selector);
  }) || [];
  
  return {
    selectors: parsed,
    slots: parsed.length || 1
  };
}

// Simplified selector parsing
function parseSelector(selector: string): CssSelector[] {
  // Parse CSS selector into internal representation
  // [card-header] -> { attribute: 'card-header' }
  // .class -> { class: 'class' }
  // tag -> { element: 'tag' }
  
  if (selector.startsWith('[') && selector.endsWith(']')) {
    return [{ attribute: selector.slice(1, -1) }];
  }
  if (selector.startsWith('.')) {
    return [{ class: selector.slice(1) }];
  }
  return [{ element: selector }];
}
```

### Projection Instruction Execution

The projection instruction inserts projected content:

```typescript
// Simplified projection implementation
function projection(
  nodeIndex: number,
  selectorIndex: number,
  attrs?: string[]
) {
  const lView = getLView();
  const tView = getTView();
  const tProjectionNode = getOrCreateTNode(
    tView,
    nodeIndex,
    TNodeType.Projection,
    null,
    attrs || null
  );
  
  // Set which slot this projection represents
  tProjectionNode.projection = selectorIndex;
  
  // During creation, append projected nodes
  if (isCreationMode(lView)) {
    const componentView = lView[DECLARATION_COMPONENT_VIEW];
    const componentTNode = componentView[T_HOST];
    const projectedNodes = componentTNode.projection[selectorIndex];
    
    if (projectedNodes) {
      for (const node of projectedNodes) {
        appendChild(lView[RENDERER], lView[nodeIndex], node);
      }
    }
  }
}
```

### Content Queries with Projection

Content children are queried across projection boundaries:

```typescript
@Component({
  selector: 'app-tab-group',
  template: `
    <div class="tab-headers">
      <button *ngFor="let tab of tabs; let i = index"
              (click)="selectTab(i)"
              [class.active]="i === selectedIndex">
        {{ tab.title }}
      </button>
    </div>
    <div class="tab-content">
      <ng-content></ng-content>
    </div>
  `
})
export class TabGroupComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
  
  selectedIndex = 0;
  
  ngAfterContentInit() {
    // Tabs are queried from projected content
    console.log('Found tabs:', this.tabs.length);
    
    // Hide all tabs except selected
    this.tabs.forEach((tab, index) => {
      tab.active = index === this.selectedIndex;
    });
    
    // Listen to changes in projected content
    this.tabs.changes.subscribe(() => {
      console.log('Tabs changed:', this.tabs.length);
    });
  }
  
  selectTab(index: number) {
    this.tabs.forEach((tab, i) => {
      tab.active = i === index;
    });
    this.selectedIndex = index;
  }
}

@Component({
  selector: 'app-tab',
  template: `
    <div class="tab-panel" *ngIf="active">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent {
  @Input() title!: string;
  active = false;
}

// Usage
@Component({
  template: `
    <app-tab-group>
      <app-tab title="Tab 1">Content 1</app-tab>
      <app-tab title="Tab 2">Content 2</app-tab>
      <app-tab title="Tab 3">Content 3</app-tab>
    </app-tab-group>
  `
})
export class AppComponent {}
```

## Multi-Slot Projection

### Named Slots Implementation

Multiple slots enable complex content composition:

```typescript
@Component({
  selector: 'app-dialog',
  template: `
    <div class="dialog-overlay" (click)="onOverlayClick()">
      <div class="dialog-container" (click)="$event.stopPropagation()">
        <div class="dialog-header">
          <ng-content select="[dialog-title]"></ng-content>
          <button class="close-btn" (click)="close()">×</button>
        </div>
        
        <div class="dialog-body">
          <ng-content select="[dialog-content]"></ng-content>
        </div>
        
        <div class="dialog-actions">
          <ng-content select="[dialog-actions]"></ng-content>
        </div>
        
        <div class="dialog-footer">
          <ng-content></ng-content>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .dialog-overlay {
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      align-items: center;
      justify-content: center;
    }
    
    .dialog-container {
      background: white;
      border-radius: 8px;
      min-width: 400px;
      max-width: 600px;
    }
    
    .dialog-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 16px;
      border-bottom: 1px solid #e0e0e0;
    }
    
    .dialog-body {
      padding: 16px;
    }
    
    .dialog-actions {
      padding: 16px;
      display: flex;
      gap: 8px;
      justify-content: flex-end;
    }
  `]
})
export class DialogComponent {
  @Output() closeDialog = new EventEmitter<void>();
  
  close() {
    this.closeDialog.emit();
  }
  
  onOverlayClick() {
    this.close();
  }
}

// Usage with multiple slots
@Component({
  template: `
    <app-dialog *ngIf="showDialog" (closeDialog)="showDialog = false">
      <h2 dialog-title>Confirm Action</h2>
      
      <div dialog-content>
        <p>Are you sure you want to proceed?</p>
        <p>This action cannot be undone.</p>
      </div>
      
      <div dialog-actions>
        <button (click)="showDialog = false">Cancel</button>
        <button (click)="confirm()">Confirm</button>
      </div>
      
      <small>Default footer content</small>
    </app-dialog>
  `
})
export class AppComponent {
  showDialog = false;
  
  confirm() {
    console.log('Confirmed');
    this.showDialog = false;
  }
}
```

### Fallback Content

Providing default content when nothing is projected:

```typescript
@Component({
  selector: 'app-card-with-defaults',
  template: `
    <div class="card">
      <div class="card-header">
        <ng-content select="[card-header]"></ng-content>
        <ng-container *ngIf="!hasHeaderContent">
          <h3>Default Title</h3>
        </ng-container>
      </div>
      
      <div class="card-body">
        <ng-content select="[card-body]"></ng-content>
        <ng-container *ngIf="!hasBodyContent">
          <p>No content provided</p>
        </ng-container>
      </div>
    </div>
  `
})
export class CardWithDefaultsComponent implements AfterContentInit {
  @ContentChild('[card-header]') headerContent?: ElementRef;
  @ContentChild('[card-body]') bodyContent?: ElementRef;
  
  hasHeaderContent = false;
  hasBodyContent = false;
  
  ngAfterContentInit() {
    this.hasHeaderContent = !!this.headerContent;
    this.hasBodyContent = !!this.bodyContent;
  }
}
```

### Selector Specificity

Understanding how Angular matches selectors:

```typescript
@Component({
  selector: 'app-complex-projection',
  template: `
    <!-- Most specific: element + attribute + class -->
    <ng-content select="div[highlight].important"></ng-content>
    
    <!-- Element + attribute -->
    <ng-content select="div[highlight]"></ng-content>
    
    <!-- Attribute + class -->
    <ng-content select="[highlight].important"></ng-content>
    
    <!-- Single attribute -->
    <ng-content select="[highlight]"></ng-content>
    
    <!-- Single class -->
    <ng-content select=".important"></ng-content>
    
    <!-- Single element -->
    <ng-content select="div"></ng-content>
    
    <!-- Default (catches all unmatched) -->
    <ng-content></ng-content>
  `
})
export class ComplexProjectionComponent {}

// Usage
@Component({
  template: `
    <app-complex-projection>
      <div highlight class="important">Most specific match</div>
      <div highlight>Element + attribute match</div>
      <span highlight class="important">Attribute + class match</span>
      <p highlight>Attribute match</p>
      <span class="important">Class match</span>
      <div>Element match</div>
      <span>Default match</span>
    </app-complex-projection>
  `
})
export class AppComponent {}
```

## Conditional Content Projection

### ngProjectAs Directive

Project content into different slots dynamically:

```typescript
@Component({
  selector: 'app-conditional-card',
  template: `
    <div class="card">
      <ng-content select="[header]"></ng-content>
      <ng-content select="[body]"></ng-content>
      <ng-content select="[footer]"></ng-content>
    </div>
  `
})
export class ConditionalCardComponent {}

// Usage with ngProjectAs
@Component({
  template: `
    <app-conditional-card>
      <!-- Conditionally project as header or body -->
      <div *ngIf="showAsHeader; else bodyTemplate"
           ngProjectAs="[header]">
        This appears in header when showAsHeader is true
      </div>
      
      <ng-template #bodyTemplate>
        <div ngProjectAs="[body]">
          This appears in body when showAsHeader is false
        </div>
      </ng-template>
      
      <div ngProjectAs="[footer]">
        Always in footer
      </div>
    </app-conditional-card>
  `
})
export class AppComponent {
  showAsHeader = true;
}
```

### Dynamic Content Projection

Programmatically controlling projection:

```typescript
@Component({
  selector: 'app-dynamic-projection',
  template: `
    <div class="container">
      <ng-container #outlet></ng-container>
    </div>
  `
})
export class DynamicProjectionComponent implements AfterContentInit {
  @ViewChild('outlet', { read: ViewContainerRef })
  outlet!: ViewContainerRef;
  
  @ContentChildren(TemplateRef) templates!: QueryList<TemplateRef<any>>;
  
  private currentIndex = 0;
  
  ngAfterContentInit() {
    // Project first template initially
    this.projectTemplate(0);
  }
  
  projectTemplate(index: number) {
    // Clear current projection
    this.outlet.clear();
    
    // Project new template
    const template = this.templates.toArray()[index];
    if (template) {
      this.outlet.createEmbeddedView(template);
      this.currentIndex = index;
    }
  }
  
  next() {
    const nextIndex = (this.currentIndex + 1) % this.templates.length;
    this.projectTemplate(nextIndex);
  }
  
  previous() {
    const prevIndex = 
      (this.currentIndex - 1 + this.templates.length) % this.templates.length;
    this.projectTemplate(prevIndex);
  }
}

// Usage
@Component({
  template: `
    <app-dynamic-projection #projection>
      <ng-template>
        <h2>Slide 1</h2>
        <p>First slide content</p>
      </ng-template>
      
      <ng-template>
        <h2>Slide 2</h2>
        <p>Second slide content</p>
      </ng-template>
      
      <ng-template>
        <h2>Slide 3</h2>
        <p>Third slide content</p>
      </ng-template>
    </app-dynamic-projection>
    
    <button (click)="projection.previous()">Previous</button>
    <button (click)="projection.next()">Next</button>
  `
})
export class AppComponent {}
```

## Content Projection with Templates

### TemplateRef and Context

Passing templates as content with context:

```typescript
@Component({
  selector: 'app-list-renderer',
  template: `
    <div class="list">
      <div *ngFor="let item of items; let i = index; let isLast = last"
           class="list-item">
        <ng-container *ngTemplateOutlet="
          itemTemplate;
          context: {
            $implicit: item,
            index: i,
            isLast: isLast,
            select: selectItem.bind(this)
          }
        "></ng-container>
      </div>
    </div>
  `
})
export class ListRendererComponent<T> {
  @Input() items: T[] = [];
  @ContentChild(TemplateRef) itemTemplate!: TemplateRef<any>;
  @Output() itemSelected = new EventEmitter<T>();
  
  selectItem(item: T) {
    this.itemSelected.emit(item);
  }
}

// Usage
@Component({
  template: `
    <app-list-renderer [items]="users" (itemSelected)="onSelect($event)">
      <ng-template let-user let-index="index" let-select="select">
        <div class="user-card">
          <h3>{{ index + 1 }}. {{ user.name }}</h3>
          <p>{{ user.email }}</p>
          <button (click)="select(user)">Select</button>
        </div>
      </ng-template>
    </app-list-renderer>
  `
})
export class AppComponent {
  users = [
    { name: 'Alice', email: 'alice@example.com' },
    { name: 'Bob', email: 'bob@example.com' },
    { name: 'Charlie', email: 'charlie@example.com' }
  ];
  
  onSelect(user: any) {
    console.log('Selected:', user);
  }
}
```

### Multiple Template Projection

Using directives to identify different templates:

```typescript
// Directive to mark templates
@Directive({
  selector: '[appTemplateType]'
})
export class TemplateTypeDirective {
  @Input() appTemplateType!: string;
  
  constructor(public template: TemplateRef<any>) {}
}

@Component({
  selector: 'app-multi-template',
  template: `
    <div class="header">
      <ng-container *ngTemplateOutlet="headerTemplate?.template">
      </ng-container>
    </div>
    
    <div class="body">
      <ng-container *ngTemplateOutlet="bodyTemplate?.template">
      </ng-container>
    </div>
    
    <div class="footer">
      <ng-container *ngTemplateOutlet="footerTemplate?.template">
      </ng-container>
    </div>
  `
})
export class MultiTemplateComponent implements AfterContentInit {
  @ContentChildren(TemplateTypeDirective)
  templates!: QueryList<TemplateTypeDirective>;
  
  headerTemplate?: TemplateTypeDirective;
  bodyTemplate?: TemplateTypeDirective;
  footerTemplate?: TemplateTypeDirective;
  
  ngAfterContentInit() {
    this.templates.forEach(template => {
      switch (template.appTemplateType) {
        case 'header':
          this.headerTemplate = template;
          break;
        case 'body':
          this.bodyTemplate = template;
          break;
        case 'footer':
          this.footerTemplate = template;
          break;
      }
    });
  }
}

// Usage
@Component({
  template: `
    <app-multi-template>
      <ng-template appTemplateType="header">
        <h1>Custom Header</h1>
      </ng-template>
      
      <ng-template appTemplateType="body">
        <p>Custom Body Content</p>
      </ng-template>
      
      <ng-template appTemplateType="footer">
        <small>Custom Footer</small>
      </ng-template>
    </app-multi-template>
  `
})
export class AppComponent {}
```

### Template Outlet Context

Advanced context passing:

```typescript
@Component({
  selector: 'app-data-table',
  template: `
    <table>
      <thead>
        <tr>
          <th *ngFor="let column of columns">{{ column.header }}</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let row of data; let rowIndex = index">
          <td *ngFor="let column of columns">
            <ng-container *ngTemplateOutlet="
              column.cellTemplate || defaultCell;
              context: {
                $implicit: row[column.field],
                row: row,
                column: column,
                rowIndex: rowIndex,
                edit: edit.bind(this)
              }
            "></ng-container>
          </td>
        </tr>
      </tbody>
    </table>
    
    <ng-template #defaultCell let-value>
      {{ value }}
    </ng-template>
  `
})
export class DataTableComponent {
  @Input() data: any[] = [];
  @Input() columns: ColumnDef[] = [];
  @Output() editRow = new EventEmitter<any>();
  
  edit(row: any) {
    this.editRow.emit(row);
  }
}

interface ColumnDef {
  field: string;
  header: string;
  cellTemplate?: TemplateRef<any>;
}

// Usage
@Component({
  template: `
    <app-data-table [data]="products" [columns]="columns">
    </app-data-table>
  `
})
export class AppComponent implements OnInit {
  @ViewChild('priceCell', { read: TemplateRef })
  priceCellTemplate!: TemplateRef<any>;
  
  @ViewChild('actionsCell', { read: TemplateRef })
  actionsCellTemplate!: TemplateRef<any>;
  
  products = [
    { id: 1, name: 'Product 1', price: 99.99 },
    { id: 2, name: 'Product 2', price: 149.99 }
  ];
  
  columns: ColumnDef[] = [];
  
  ngOnInit() {
    this.columns = [
      { field: 'name', header: 'Name' },
      { field: 'price', header: 'Price', cellTemplate: this.priceCellTemplate },
      { field: 'actions', header: 'Actions', cellTemplate: this.actionsCellTemplate }
    ];
  }
}
```

## Common Misconceptions

### Misconception 1: "Projected content belongs to the child component"

**Reality**: Projected content remains part of the parent component's view hierarchy and change detection tree.

```typescript
@Component({
  selector: 'app-parent',
  template: `
    <p>Parent value: {{ parentValue }}</p>
    <app-child>
      <p>Projected content: {{ parentValue }}</p>
    </app-child>
  `
})
export class ParentComponent {
  parentValue = 'Hello';
  
  constructor() {
    // This will update both instances
    setTimeout(() => {
      this.parentValue = 'Updated';
    }, 1000);
  }
}

@Component({
  selector: 'app-child',
  template: `
    <div>
      <p>Child value: {{ childValue }}</p>
      <ng-content></ng-content>
    </div>
  `
})
export class ChildComponent {
  childValue = 'World';
}
```

### Misconception 2: "ng-content can be used conditionally with *ngIf"

**Reality**: `ng-content` cannot be used with structural directives. Use `ng-container` or `ng-template` instead.

```typescript
// WRONG - This doesn't work
@Component({
  template: `
    <ng-content *ngIf="show" select="[header]"></ng-content>
  `
})
export class WrongComponent {
  show = true;
}

// CORRECT - Wrap in ng-container
@Component({
  template: `
    <ng-container *ngIf="show">
      <ng-content select="[header]"></ng-content>
    </ng-container>
  `
})
export class CorrectComponent {
  show = true;
}
```

### Misconception 3: "Content projection creates a copy of the content"

**Reality**: Content projection moves the actual DOM nodes, not copies.

```typescript
@Component({
  selector: 'app-no-copies',
  template: `
    <div class="slot-1">
      <ng-content select=".item"></ng-content>
    </div>
    <div class="slot-2">
      <!-- This won't show anything - content is already projected above -->
      <ng-content select=".item"></ng-content>
    </div>
  `
})
export class NoCopiesComponent {}
```

## Performance Implications

### Change Detection Boundaries

Projected content participates in parent's change detection:

```typescript
@Component({
  selector: 'app-performance-demo',
  template: `
    <app-wrapper>
      <expensive-component [data]="data"></expensive-component>
    </app-wrapper>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PerformanceDemoComponent {
  data = { value: 0 };
  
  updateData() {
    // Must create new reference for OnPush to detect change
    this.data = { value: this.data.value + 1 };
  }
}

@Component({
  selector: 'app-wrapper',
  template: `
    <div class="wrapper">
      <ng-content></ng-content>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class WrapperComponent {
  // Even though wrapper uses OnPush,
  // projected content's change detection is controlled by parent
}
```

### Avoiding Unnecessary Projections

Optimize by projecting only when needed:

```typescript
@Component({
  selector: 'app-conditional-wrapper',
  template: `
    <div *ngIf="show" class="wrapper">
      <ng-content></ng-content>
    </div>
  `
})
export class ConditionalWrapperComponent {
  @Input() show = true;
  
  // Note: Even when show is false, projected content is still created
  // by parent. It's just not rendered in the DOM.
}

// Better approach for truly conditional content
@Component({
  template: `
    <div *ngIf="show" class="wrapper">
      <ng-template [ngTemplateOutlet]="content"></ng-template>
    </div>
  `
})
export class BetterConditionalComponent {
  @Input() show = true;
  @ContentChild(TemplateRef) content!: TemplateRef<any>;
  
  // Template is only instantiated when show is true
}
```

### Memory Considerations

ViewContainerRef cleanup:

```typescript
@Component({
  selector: 'app-view-manager',
  template: `<ng-container #container></ng-container>`
})
export class ViewManagerComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  private viewRefs: ViewRef[] = [];
  
  addView(template: TemplateRef<any>) {
    const viewRef = this.container.createEmbeddedView(template);
    this.viewRefs.push(viewRef);
  }
  
  ngOnDestroy() {
    // Clean up all views to prevent memory leaks
    this.viewRefs.forEach(viewRef => {
      viewRef.destroy();
    });
    this.viewRefs = [];
    
    // Alternative: clear entire container
    this.container.clear();
  }
}
```

## Interview Questions

### Question 1: How does ng-content work internally in Angular?

**Answer**: `ng-content` is a projection mechanism that works through several steps:

1. **Compilation**: Angular compiler transforms `ng-content` into projection instructions (`projectionDef` and `projection`)
2. **Distribution**: During component creation, child nodes are distributed to slots based on CSS selectors
3. **Rendering**: Projection instructions insert the distributed nodes at the appropriate locations
4. **Change Detection**: Projected content remains in parent's change detection tree

The key insight is that projected content is not moved between components logically—it stays part of the parent's view hierarchy even though it's rendered inside the child's template.

### Question 2: What's the difference between @ContentChild and @ViewChild?

**Answer**: 
- **@ContentChild**: Queries projected content (content between component tags)
- **@ViewChild**: Queries elements in the component's own template

```typescript
@Component({
  selector: 'app-demo',
  template: `
    <div #viewElement>View Child</div>
    <ng-content></ng-content>
  `
})
export class DemoComponent {
  @ViewChild('viewElement') viewChild!: ElementRef;
  @ContentChild('contentElement') contentChild!: ElementRef;
  
  // viewChild queries the template
  // contentChild queries projected content:
  // <app-demo><div #contentElement>Content</div></app-demo>
}
```

### Question 3: How do you create multiple projection slots?

**Answer**: Use the `select` attribute with CSS selectors:

```typescript
@Component({
  template: `
    <ng-content select="[slot=header]"></ng-content>
    <ng-content select="[slot=body]"></ng-content>
    <ng-content></ng-content> <!-- default slot -->
  `
})
export class MultiSlotComponent {}
```

Angular evaluates selectors in order and projects each node into the first matching slot. Unmatched content goes to the default slot (the one without a selector).

### Question 4: What is ViewContainerRef and when would you use it?

**Answer**: `ViewContainerRef` represents a container where views can be dynamically attached. Use cases include:

- **Dynamic component creation**: Creating components at runtime
- **Structural directives**: Implementing custom `*ngIf`-like directives
- **Portal/outlet patterns**: Implementing modal dialogs, tooltips
- **View manipulation**: Inserting, moving, removing views programmatically

It provides methods like `createComponent()`, `createEmbeddedView()`, `clear()`, `remove()`, and `move()`.

### Question 5: How does change detection work with projected content?

**Answer**: Projected content remains in the parent component's change detection tree. Key points:

1. Parent component's change detection checks projected content
2. Child component's change detection strategy doesn't affect projected content
3. Changes in parent automatically propagate to projected content
4. Content queries (`@ContentChild`) trigger `ngAfterContentInit` and `ngAfterContentChecked`

This means if a child uses `OnPush` strategy, it doesn't prevent projected content from being checked when parent is checked.

### Question 6: What's the purpose of ngProjectAs?

**Answer**: `ngProjectAs` allows you to override which slot an element is projected into, regardless of its actual selector. This is useful when:

- Using structural directives that wrap content
- Conditionally changing which slot content projects into
- Working with `ng-container` that doesn't match any selector

```typescript
<ng-container *ngIf="condition" ngProjectAs="[header]">
  <h1>Conditional Header</h1>
</ng-container>
```

Without `ngProjectAs`, the `ng-container` wouldn't match `[header]` selector.

### Question 7: How do you prevent content projection performance issues?

**Answer**: Best practices include:

1. **Use OnPush strategy**: Reduce change detection on wrapper components
2. **Lazy projection**: Use templates instead of direct content for conditional rendering
3. **Limit query frequency**: Use `@ContentChild` with `{ read: TemplateRef }` for templates
4. **Clean up views**: Destroy views created with ViewContainerRef
5. **Avoid deep nesting**: Multiple levels of projection add complexity
6. **Use trackBy**: When projecting lists with `*ngFor`

```typescript
// Good: Template only instantiated when needed
<ng-template #content>
  <expensive-component></expensive-component>
</ng-template>

// Less optimal: Component created even if not shown
<expensive-component></expensive-component>
```

### Question 8: Can you explain the ViewRef hierarchy?

**Answer**: Angular maintains a hierarchical view structure:

1. **Component View**: Root view of a component
2. **Embedded View**: View created from templates (`*ngIf`, `*ngFor`, etc.)
3. **Host View**: View that hosts a component

Each view can contain child views, forming a tree. ViewContainerRef manages this tree, allowing dynamic insertion and removal of views. Change detection traverses this tree from root to leaves. Understanding this hierarchy is crucial for:

- Implementing structural directives
- Dynamic component creation
- Optimizing change detection
- Memory management

## Key Takeaways

1. **Content projection moves DOM nodes** between parent and child component templates without changing ownership
2. **Projected content stays in parent's change detection tree**, making parent responsible for updates
3. **ViewContainerRef is the API for dynamic view manipulation**, enabling programmatic control over view hierarchy
4. **Multiple slots use CSS selectors** to distribute content, with specificity rules determining matches
5. **Content queries cross projection boundaries**, allowing child components to query projected content
6. **ngProjectAs overrides selector matching**, useful with structural directives and ng-container
7. **Templates provide lazy instantiation**, better than direct projection for conditional content
8. **View lifecycle management is critical** to prevent memory leaks with dynamic views

## Resources

### Official Documentation
- [Angular Content Projection Guide](https://angular.dev/guide/components/content-projection)
- [ViewContainerRef API](https://angular.dev/api/core/ViewContainerRef)
- [TemplateRef API](https://angular.dev/api/core/TemplateRef)

### Articles & Tutorials
- "Everything you need to know about ng-content" by Netanel Basal
- "Angular Content Projection Internals" by Max Koretskyi
- "Understanding ViewContainerRef in Angular" by Thoughtram

### Videos
- Angular Content Projection Deep Dive
- Advanced Component Composition Patterns
- Dynamic Component Creation in Angular

### Books
- "ng-book: The Complete Guide to Angular" - Content Projection chapter
- "Angular Development with TypeScript" - Component composition section

### Source Code
- [Angular Ivy Renderer](https://github.com/angular/angular/tree/main/packages/core/src/render3)
- [Projection Implementation](https://github.com/angular/angular/blob/main/packages/core/src/render3/instructions/projection.ts)
