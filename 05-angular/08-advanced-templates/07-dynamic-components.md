# Dynamic Components

## The Idea

**In plain English:** Dynamic components let your app build and display brand-new pieces of the screen on the fly, while the app is already running — instead of deciding everything upfront. Think of a "component" as a reusable building block (like a card, a popup, or a form) that Angular can stamp out whenever your code asks for one.

**Real-world analogy:** Imagine a food court where stalls can be set up or taken down at any time during the day. The food court manager (your Angular app) decides which stall to open, places it in an empty spot, and tears it down when it closes.

- The food court = the Angular app running in the browser
- An empty spot in the food court = the `ViewContainerRef` (a reserved place in the page where components can be inserted)
- A stall being set up = a dynamic component being created with `createComponent()`
- The stall's menu and equipment = the component's properties and logic (`componentRef.instance`)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding Dynamic Components](#understanding-dynamic-components)
3. [ViewContainerRef.createComponent](#viewcontainerrefcreatecomponent)
4. [ComponentRef API](#componentref-api)
5. [Dynamic Component Loading](#dynamic-component-loading)
6. [Portal Pattern](#portal-pattern)
7. [Component Factories](#component-factories)
8. [Passing Data to Dynamic Components](#passing-data-to-dynamic-components)
9. [Component Communication](#component-communication)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)
13. [Key Takeaways](#key-takeaways)
14. [Resources](#resources)

## Introduction

Dynamic components are Angular components that are created and inserted into the application at runtime, rather than being declared in templates at compile time. This powerful feature enables scenarios like modal dialogs, dynamic forms, plugin systems, and content management systems where the component structure isn't known until runtime.

This comprehensive guide covers modern Angular approaches to dynamic component creation, including the ViewContainerRef API, ComponentRef management, the Portal pattern, and best practices for building flexible, maintainable dynamic component systems.

## Understanding Dynamic Components

Dynamic components are instantiated programmatically through TypeScript code rather than through template declarations. They're useful when you need to:

- Create components based on user actions
- Load components from configuration
- Build plugin architectures
- Implement modal/dialog systems
- Create dynamic forms

### Static vs Dynamic Components

```typescript
import { Component } from '@angular/core';
import { ChildComponent } from './child.component';

// Static component - declared in template
@Component({
  selector: 'app-static',
  standalone: true,
  imports: [ChildComponent],
  template: `
    <app-child></app-child>
  `
})
export class StaticComponent {}

// Dynamic component - created programmatically
@Component({
  selector: 'app-dynamic',
  standalone: true,
  template: `
    <ng-container #container></ng-container>
  `
})
export class DynamicComponent implements OnInit {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  ngOnInit() {
    this.container.createComponent(ChildComponent);
  }
}
```

## ViewContainerRef.createComponent

The `createComponent` method is the modern way to create dynamic components in Angular.

### Basic Component Creation

```typescript
import { 
  Component, 
  ViewChild, 
  ViewContainerRef, 
  OnInit 
} from '@angular/core';

@Component({
  selector: 'app-alert',
  standalone: true,
  template: `<div class="alert">{{ message }}</div>`,
  styles: [`
    .alert {
      padding: 16px;
      background: #f0f0f0;
      border-radius: 4px;
    }
  `]
})
export class AlertComponent {
  message = 'Alert message';
}

@Component({
  selector: 'app-container',
  standalone: true,
  template: `
    <button (click)="addAlert()">Add Alert</button>
    <ng-container #alertContainer></ng-container>
  `
})
export class ContainerComponent implements OnInit {
  @ViewChild('alertContainer', { read: ViewContainerRef })
  alertContainer!: ViewContainerRef;
  
  ngOnInit() {
    console.log('Container ready');
  }
  
  addAlert() {
    const componentRef = this.alertContainer.createComponent(AlertComponent);
    componentRef.instance.message = 'New alert created!';
  }
}
```

### Creating Multiple Dynamic Components

```typescript
import { Component, ViewChild, ViewContainerRef, Type } from '@angular/core';

@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ title }}</h3>
      <p>{{ content }}</p>
    </div>
  `
})
export class CardComponent {
  title = 'Card Title';
  content = 'Card content';
}

@Component({
  selector: 'app-list-item',
  standalone: true,
  template: `
    <div class="list-item">{{ text }}</div>
  `
})
export class ListItemComponent {
  text = 'List item';
}

@Component({
  selector: 'app-dashboard',
  standalone: true,
  template: `
    <button (click)="addCard()">Add Card</button>
    <button (click)="addListItem()">Add List Item</button>
    <button (click)="clear()">Clear</button>
    
    <div #container></div>
  `
})
export class DashboardComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  addCard() {
    const componentRef = this.container.createComponent(CardComponent);
    componentRef.instance.title = `Card ${this.container.length}`;
    componentRef.instance.content = 'Dynamic card content';
  }
  
  addListItem() {
    const componentRef = this.container.createComponent(ListItemComponent);
    componentRef.instance.text = `Item ${this.container.length}`;
  }
  
  clear() {
    this.container.clear();
  }
}
```

### Positioning Dynamic Components

```typescript
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-positioned',
  standalone: true,
  template: `
    <button (click)="addAtEnd()">Add at End</button>
    <button (click)="addAtStart()">Add at Start</button>
    <button (click)="addAtIndex(2)">Add at Index 2</button>
    
    <div #container></div>
  `
})
export class PositionedComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  addAtEnd() {
    // Default behavior - adds at the end
    this.container.createComponent(CardComponent);
  }
  
  addAtStart() {
    // Add at the beginning
    const componentRef = this.container.createComponent(
      CardComponent,
      { index: 0 }
    );
    componentRef.instance.title = 'First Card';
  }
  
  addAtIndex(index: number) {
    // Add at specific index
    const componentRef = this.container.createComponent(
      CardComponent,
      { index }
    );
    componentRef.instance.title = `Card at index ${index}`;
  }
}
```

## ComponentRef API

`ComponentRef` represents a reference to a dynamically created component and provides methods to interact with it.

### ComponentRef Methods

```typescript
import { 
  Component, 
  ComponentRef, 
  ViewChild, 
  ViewContainerRef,
  OnDestroy
} from '@angular/core';

@Component({
  selector: 'app-component-ref',
  standalone: true,
  template: `
    <button (click)="createComponent()">Create</button>
    <button (click)="destroyComponent()">Destroy</button>
    <button (click)="updateComponent()">Update</button>
    <button (click)="detectChanges()">Detect Changes</button>
    
    <div #container></div>
  `
})
export class ComponentRefComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  private componentRef?: ComponentRef<AlertComponent>;
  
  createComponent() {
    if (this.componentRef) {
      console.log('Component already exists');
      return;
    }
    
    // Create component
    this.componentRef = this.container.createComponent(AlertComponent);
    
    // Access component instance
    this.componentRef.instance.message = 'Component created';
    
    // Access location (ElementRef)
    const element = this.componentRef.location.nativeElement;
    console.log('Component element:', element);
    
    // Access change detector
    console.log('Change detector:', this.componentRef.changeDetectorRef);
    
    // Access host view
    console.log('Host view:', this.componentRef.hostView);
  }
  
  destroyComponent() {
    if (this.componentRef) {
      this.componentRef.destroy();
      this.componentRef = undefined;
    }
  }
  
  updateComponent() {
    if (this.componentRef) {
      this.componentRef.instance.message = 'Updated message';
      this.componentRef.changeDetectorRef.detectChanges();
    }
  }
  
  detectChanges() {
    if (this.componentRef) {
      this.componentRef.changeDetectorRef.detectChanges();
    }
  }
  
  ngOnDestroy() {
    this.destroyComponent();
  }
}
```

### Managing Component Lifecycle

```typescript
import { 
  Component, 
  ComponentRef, 
  ViewChild, 
  ViewContainerRef,
  OnDestroy
} from '@angular/core';

@Component({
  selector: 'app-lifecycle-manager',
  standalone: true,
  template: `
    <button (click)="createComponent()">Create Component</button>
    <button (click)="destroyAll()">Destroy All</button>
    <p>Active components: {{ componentRefs.length }}</p>
    
    <div #container></div>
  `
})
export class LifecycleManagerComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  componentRefs: ComponentRef<any>[] = [];
  
  createComponent() {
    const componentRef = this.container.createComponent(CardComponent);
    
    componentRef.instance.title = `Component ${this.componentRefs.length + 1}`;
    
    // Track component reference
    this.componentRefs.push(componentRef);
    
    // Listen to component destruction
    componentRef.onDestroy(() => {
      console.log('Component destroyed');
      const index = this.componentRefs.indexOf(componentRef);
      if (index > -1) {
        this.componentRefs.splice(index, 1);
      }
    });
  }
  
  destroyAll() {
    // Destroy all components
    this.componentRefs.forEach(ref => ref.destroy());
    this.componentRefs = [];
    
    // Alternative: clear the container
    // this.container.clear();
  }
  
  ngOnDestroy() {
    this.destroyAll();
  }
}
```

## Dynamic Component Loading

### Loading Components Based on Type

```typescript
import { Component, ViewChild, ViewContainerRef, Type } from '@angular/core';

interface DynamicComponent {
  data: any;
}

@Component({
  selector: 'app-text-widget',
  standalone: true,
  template: `<div class="widget text">{{ data }}</div>`
})
export class TextWidgetComponent implements DynamicComponent {
  data: any;
}

@Component({
  selector: 'app-image-widget',
  standalone: true,
  template: `<div class="widget image"><img [src]="data" /></div>`
})
export class ImageWidgetComponent implements DynamicComponent {
  data: any;
}

@Component({
  selector: 'app-chart-widget',
  standalone: true,
  template: `<div class="widget chart">Chart: {{ data }}</div>`
})
export class ChartWidgetComponent implements DynamicComponent {
  data: any;
}

@Component({
  selector: 'app-widget-loader',
  standalone: true,
  template: `
    <button (click)="loadWidget('text')">Load Text</button>
    <button (click)="loadWidget('image')">Load Image</button>
    <button (click)="loadWidget('chart')">Load Chart</button>
    
    <div #widgetContainer></div>
  `
})
export class WidgetLoaderComponent {
  @ViewChild('widgetContainer', { read: ViewContainerRef })
  widgetContainer!: ViewContainerRef;
  
  private componentMap: Record<string, Type<DynamicComponent>> = {
    text: TextWidgetComponent,
    image: ImageWidgetComponent,
    chart: ChartWidgetComponent
  };
  
  loadWidget(type: string) {
    this.widgetContainer.clear();
    
    const componentType = this.componentMap[type];
    if (!componentType) {
      console.error('Unknown widget type:', type);
      return;
    }
    
    const componentRef = this.widgetContainer.createComponent(componentType);
    
    // Set data based on type
    switch (type) {
      case 'text':
        componentRef.instance.data = 'Hello World';
        break;
      case 'image':
        componentRef.instance.data = 'https://example.com/image.jpg';
        break;
      case 'chart':
        componentRef.instance.data = [1, 2, 3, 4, 5];
        break;
    }
  }
}
```

### Loading from Configuration

```typescript
import { Component, ViewChild, ViewContainerRef, OnInit, Type } from '@angular/core';

interface WidgetConfig {
  type: string;
  data: any;
  position?: number;
}

@Component({
  selector: 'app-config-loader',
  standalone: true,
  template: `
    <h2>Dynamic Dashboard</h2>
    <div #dashboard></div>
  `
})
export class ConfigLoaderComponent implements OnInit {
  @ViewChild('dashboard', { read: ViewContainerRef })
  dashboard!: ViewContainerRef;
  
  private componentMap: Record<string, Type<any>> = {
    text: TextWidgetComponent,
    image: ImageWidgetComponent,
    chart: ChartWidgetComponent
  };
  
  private config: WidgetConfig[] = [
    { type: 'text', data: 'Welcome!' },
    { type: 'chart', data: [10, 20, 30] },
    { type: 'image', data: 'logo.png' },
    { type: 'text', data: 'Footer text' }
  ];
  
  ngOnInit() {
    this.loadFromConfig(this.config);
  }
  
  loadFromConfig(config: WidgetConfig[]) {
    this.dashboard.clear();
    
    config.forEach((widgetConfig, index) => {
      const componentType = this.componentMap[widgetConfig.type];
      
      if (!componentType) {
        console.error('Unknown component type:', widgetConfig.type);
        return;
      }
      
      const position = widgetConfig.position ?? index;
      const componentRef = this.dashboard.createComponent(
        componentType,
        { index: position }
      );
      
      componentRef.instance.data = widgetConfig.data;
    });
  }
}
```

### Lazy Loading Dynamic Components

```typescript
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-lazy-loader',
  standalone: true,
  template: `
    <button (click)="loadComponent()">Load Component</button>
    <div #container></div>
  `
})
export class LazyLoaderComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  async loadComponent() {
    // Lazy load the component
    const { HeavyComponent } = await import('./heavy.component');
    
    const componentRef = this.container.createComponent(HeavyComponent);
    componentRef.instance.data = 'Lazy loaded data';
  }
}

// heavy.component.ts
@Component({
  selector: 'app-heavy',
  standalone: true,
  template: `<div>Heavy component: {{ data }}</div>`
})
export class HeavyComponent {
  data: any;
}
```

## Portal Pattern

The Portal pattern (popularized by Angular CDK) provides a structured way to render components in different locations.

### Basic Portal Implementation

```typescript
import { 
  Component, 
  ViewChild, 
  ViewContainerRef, 
  TemplateRef,
  ComponentRef,
  Type
} from '@angular/core';

// Portal base class
export abstract class Portal<T> {
  abstract attach(host: ViewContainerRef): ComponentRef<T> | any;
}

// Component Portal
export class ComponentPortal<T> extends Portal<T> {
  constructor(public component: Type<T>) {
    super();
  }
  
  attach(host: ViewContainerRef): ComponentRef<T> {
    return host.createComponent(this.component);
  }
}

// Template Portal
export class TemplatePortal extends Portal<any> {
  constructor(public template: TemplateRef<any>) {
    super();
  }
  
  attach(host: ViewContainerRef) {
    return host.createEmbeddedView(this.template);
  }
}

// Portal Outlet
@Component({
  selector: 'app-portal-outlet',
  standalone: true,
  template: `<ng-container #outlet></ng-container>`
})
export class PortalOutletComponent {
  @ViewChild('outlet', { read: ViewContainerRef })
  outlet!: ViewContainerRef;
  
  attach(portal: Portal<any>) {
    this.outlet.clear();
    return portal.attach(this.outlet);
  }
  
  detach() {
    this.outlet.clear();
  }
}
```

### Using Portal Pattern

```typescript
import { Component, ViewChild } from '@angular/core';

@Component({
  selector: 'app-modal-content',
  standalone: true,
  template: `
    <div class="modal">
      <h2>{{ title }}</h2>
      <p>{{ message }}</p>
    </div>
  `
})
export class ModalContentComponent {
  title = 'Modal Title';
  message = 'Modal message';
}

@Component({
  selector: 'app-portal-demo',
  standalone: true,
  imports: [PortalOutletComponent],
  template: `
    <button (click)="openModal()">Open Modal</button>
    <button (click)="closeModal()">Close Modal</button>
    
    <div class="modal-container">
      <app-portal-outlet #portalOutlet></app-portal-outlet>
    </div>
  `
})
export class PortalDemoComponent {
  @ViewChild('portalOutlet') portalOutlet!: PortalOutletComponent;
  
  openModal() {
    const portal = new ComponentPortal(ModalContentComponent);
    const componentRef = this.portalOutlet.attach(portal);
    
    if (componentRef instanceof ComponentRef) {
      componentRef.instance.title = 'Dynamic Modal';
      componentRef.instance.message = 'This modal was loaded dynamically!';
    }
  }
  
  closeModal() {
    this.portalOutlet.detach();
  }
}
```

### Advanced Portal Service

```typescript
import { Injectable, ComponentRef, ViewContainerRef, Type } from '@angular/core';
import { Subject } from 'rxjs';

interface PortalConfig<T> {
  component: Type<T>;
  data?: any;
  outlet?: ViewContainerRef;
}

@Injectable({ providedIn: 'root' })
export class PortalService {
  private portals = new Map<string, ComponentRef<any>>();
  private readonly portalClosed$ = new Subject<string>();
  
  open<T>(id: string, config: PortalConfig<T>): ComponentRef<T> {
    // Close existing portal with same id
    this.close(id);
    
    const { component, data, outlet } = config;
    
    if (!outlet) {
      throw new Error('Portal outlet is required');
    }
    
    const componentRef = outlet.createComponent(component);
    
    // Set data if provided
    if (data) {
      Object.assign(componentRef.instance, data);
    }
    
    this.portals.set(id, componentRef);
    
    return componentRef;
  }
  
  close(id: string): void {
    const portal = this.portals.get(id);
    
    if (portal) {
      portal.destroy();
      this.portals.delete(id);
      this.portalClosed$.next(id);
    }
  }
  
  closeAll(): void {
    this.portals.forEach((portal, id) => {
      portal.destroy();
      this.portalClosed$.next(id);
    });
    this.portals.clear();
  }
  
  get(id: string): ComponentRef<any> | undefined {
    return this.portals.get(id);
  }
  
  onPortalClosed() {
    return this.portalClosed$.asObservable();
  }
}

// Usage
@Component({
  selector: 'app-portal-service-demo',
  standalone: true,
  template: `
    <button (click)="openModal()">Open Modal</button>
    <div #modalOutlet></div>
  `
})
export class PortalServiceDemoComponent {
  @ViewChild('modalOutlet', { read: ViewContainerRef })
  modalOutlet!: ViewContainerRef;
  
  constructor(private portalService: PortalService) {}
  
  openModal() {
    this.portalService.open('modal', {
      component: ModalContentComponent,
      outlet: this.modalOutlet,
      data: {
        title: 'Service Modal',
        message: 'Opened via service'
      }
    });
  }
}
```

## Component Factories

While ComponentFactory is deprecated in newer Angular versions, understanding the concept is still valuable.

### Modern Approach (No Factory Needed)

```typescript
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-modern',
  standalone: true,
  template: `
    <button (click)="createComponent()">Create</button>
    <div #container></div>
  `
})
export class ModernComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  createComponent() {
    // Modern Angular - no factory needed
    this.container.createComponent(AlertComponent);
  }
}
```

## Passing Data to Dynamic Components

### Using Instance Properties

```typescript
import { Component, ViewChild, ViewContainerRef, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
      <p>Role: {{ user.role }}</p>
    </div>
  `
})
export class UserCardComponent {
  @Input() user: any = {};
}

@Component({
  selector: 'app-user-list',
  standalone: true,
  template: `
    <button (click)="loadUsers()">Load Users</button>
    <div #container></div>
  `
})
export class UserListComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  users = [
    { name: 'John', email: 'john@example.com', role: 'Admin' },
    { name: 'Jane', email: 'jane@example.com', role: 'User' },
    { name: 'Bob', email: 'bob@example.com', role: 'Manager' }
  ];
  
  loadUsers() {
    this.container.clear();
    
    this.users.forEach(user => {
      const componentRef = this.container.createComponent(UserCardComponent);
      componentRef.setInput('user', user);
      // Or: componentRef.instance.user = user;
    });
  }
}
```

### Using Dependency Injection

```typescript
import { Component, Injectable, inject, InjectionToken, Injector } from '@angular/core';

export const USER_DATA = new InjectionToken<any>('USER_DATA');

@Component({
  selector: 'app-injected-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ userData.name }}</h3>
      <p>{{ userData.email }}</p>
    </div>
  `
})
export class InjectedCardComponent {
  userData = inject(USER_DATA);
}

@Component({
  selector: 'app-injector-demo',
  standalone: true,
  template: `
    <button (click)="createCard()">Create Card</button>
    <div #container></div>
  `
})
export class InjectorDemoComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  createCard() {
    const userData = { name: 'John', email: 'john@example.com' };
    
    // Create custom injector with data
    const injector = Injector.create({
      providers: [
        { provide: USER_DATA, useValue: userData }
      ]
    });
    
    // Create component with custom injector
    this.container.createComponent(InjectedCardComponent, {
      injector
    });
  }
}
```

## Component Communication

### Parent to Dynamic Component

```typescript
import { Component, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-child',
  standalone: true,
  template: `
    <div>
      <p>Message: {{ message }}</p>
      <p>Count: {{ count }}</p>
    </div>
  `
})
export class ChildComponent {
  message = '';
  count = 0;
  
  updateMessage(msg: string) {
    this.message = msg;
  }
  
  increment() {
    this.count++;
  }
}

@Component({
  selector: 'app-parent-comm',
  standalone: true,
  template: `
    <button (click)="createChild()">Create Child</button>
    <button (click)="updateChild()">Update Child</button>
    <button (click)="incrementChild()">Increment Child</button>
    
    <div #container></div>
  `
})
export class ParentCommComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  private childRef?: ComponentRef<ChildComponent>;
  
  createChild() {
    this.childRef = this.container.createComponent(ChildComponent);
    this.childRef.instance.message = 'Initial message';
  }
  
  updateChild() {
    if (this.childRef) {
      this.childRef.instance.updateMessage('Updated message');
      this.childRef.changeDetectorRef.detectChanges();
    }
  }
  
  incrementChild() {
    if (this.childRef) {
      this.childRef.instance.increment();
      this.childRef.changeDetectorRef.detectChanges();
    }
  }
}
```

### Dynamic Component to Parent

```typescript
import { Component, ViewChild, ViewContainerRef, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-child-emitter',
  standalone: true,
  template: `
    <button (click)="emitData()">Send to Parent</button>
  `
})
export class ChildEmitterComponent {
  @Output() dataEmitted = new EventEmitter<string>();
  
  emitData() {
    this.dataEmitted.emit('Data from child');
  }
}

@Component({
  selector: 'app-parent-listener',
  standalone: true,
  template: `
    <button (click)="createChild()">Create Child</button>
    <p>Received: {{ receivedData }}</p>
    
    <div #container></div>
  `
})
export class ParentListenerComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  receivedData = '';
  
  createChild() {
    const componentRef = this.container.createComponent(ChildEmitterComponent);
    
    // Subscribe to child's output
    componentRef.instance.dataEmitted.subscribe(data => {
      this.receivedData = data;
    });
  }
}
```

### Using Service for Communication

```typescript
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ComponentCommunicationService {
  private messageSource = new Subject<string>();
  message$ = this.messageSource.asObservable();
  
  sendMessage(message: string) {
    this.messageSource.next(message);
  }
}

@Component({
  selector: 'app-dynamic-sender',
  standalone: true,
  template: `
    <button (click)="send()">Send Message</button>
  `
})
export class DynamicSenderComponent {
  constructor(private commService: ComponentCommunicationService) {}
  
  send() {
    this.commService.sendMessage('Hello from dynamic component!');
  }
}

@Component({
  selector: 'app-receiver',
  standalone: true,
  template: `
    <button (click)="createSender()">Create Sender</button>
    <p>Received: {{ message }}</p>
    
    <div #container></div>
  `
})
export class ReceiverComponent implements OnInit {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  message = '';
  
  constructor(private commService: ComponentCommunicationService) {}
  
  ngOnInit() {
    this.commService.message$.subscribe(msg => {
      this.message = msg;
    });
  }
  
  createSender() {
    this.container.createComponent(DynamicSenderComponent);
  }
}
```

## Common Mistakes

### 1. Not Destroying Components

```typescript
// BAD: Memory leak
@Component({ template: '<div #container></div>' })
export class BadComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  createMany() {
    for (let i = 0; i < 100; i++) {
      this.container.createComponent(CardComponent);
    }
    // Components never destroyed!
  }
}

// GOOD: Proper cleanup
@Component({ template: '<div #container></div>' })
export class GoodComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  private componentRefs: ComponentRef<any>[] = [];
  
  createMany() {
    for (let i = 0; i < 100; i++) {
      const ref = this.container.createComponent(CardComponent);
      this.componentRefs.push(ref);
    }
  }
  
  ngOnDestroy() {
    this.componentRefs.forEach(ref => ref.destroy());
  }
}
```

### 2. Not Triggering Change Detection

```typescript
// BAD: Changes not reflected
@Component({ template: '<div #container></div>' })
export class BadComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  updateComponent(ref: ComponentRef<any>) {
    ref.instance.data = 'new data';
    // Change detection not triggered!
  }
}

// GOOD: Manually trigger change detection
@Component({ template: '<div #container></div>' })
export class GoodComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  updateComponent(ref: ComponentRef<any>) {
    ref.instance.data = 'new data';
    ref.changeDetectorRef.detectChanges();
  }
}
```

### 3. Accessing ViewChild Too Early

```typescript
// BAD: ViewChild not available in constructor
@Component({ template: '<div #container></div>' })
export class BadComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  constructor() {
    // Error: container is undefined
    this.container.createComponent(CardComponent);
  }
}

// GOOD: Use ngAfterViewInit
@Component({ template: '<div #container></div>' })
export class GoodComponent implements AfterViewInit {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  ngAfterViewInit() {
    this.container.createComponent(CardComponent);
  }
}
```

## Best Practices

### 1. Track Component References

```typescript
@Component({ template: '<div #container></div>' })
export class TrackedComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  private components = new Map<string, ComponentRef<any>>();
  
  create(id: string, component: Type<any>) {
    const ref = this.container.createComponent(component);
    this.components.set(id, ref);
    return ref;
  }
  
  destroy(id: string) {
    const ref = this.components.get(id);
    if (ref) {
      ref.destroy();
      this.components.delete(id);
    }
  }
  
  ngOnDestroy() {
    this.components.forEach(ref => ref.destroy());
    this.components.clear();
  }
}
```

### 2. Use Type-Safe Component Creation

```typescript
interface ComponentData {
  title: string;
  content: string;
}

function createTypedComponent<T extends ComponentData>(
  container: ViewContainerRef,
  component: Type<T>,
  data: ComponentData
): ComponentRef<T> {
  const componentRef = container.createComponent(component);
  componentRef.instance.title = data.title;
  componentRef.instance.content = data.content;
  return componentRef;
}
```

### 3. Provide Cleanup Methods

```typescript
@Component({ template: '<div #container></div>' })
export class CleanComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;
  
  clear() {
    this.container.clear();
  }
  
  removeAt(index: number) {
    this.container.remove(index);
  }
  
  detachAt(index: number) {
    return this.container.detach(index);
  }
}
```

### 4. Document Component Requirements

```typescript
/**
 * Creates a dynamic card component
 * @param container - ViewContainerRef to create component in
 * @param data - Card data including title and content
 * @returns ComponentRef for the created component
 * @throws Error if container is not available
 */
function createCard(
  container: ViewContainerRef,
  data: CardData
): ComponentRef<CardComponent> {
  if (!container) {
    throw new Error('Container is required');
  }
  
  const componentRef = container.createComponent(CardComponent);
  componentRef.instance.title = data.title;
  componentRef.instance.content = data.content;
  
  return componentRef;
}
```

## Interview Questions

### Q1: What is a dynamic component in Angular?

**Answer:** A dynamic component is a component created programmatically at runtime using TypeScript code rather than declared in a template at compile time. It's created using `ViewContainerRef.createComponent()`.

### Q2: What's the difference between createComponent and createEmbeddedView?

**Answer:** `createComponent()` creates a component dynamically, while `createEmbeddedView()` creates a view from a template (like ng-template). Components have their own logic and lifecycle, while embedded views are just template fragments.

### Q3: How do you pass data to a dynamic component?

**Answer:** You can set properties directly on the component instance (`componentRef.instance.property = value`), use `setInput()`, or provide data through dependency injection with a custom injector.

### Q4: When should you use dynamic components?

**Answer:** Use dynamic components for modals/dialogs, dynamic forms, plugin systems, content management systems, or any scenario where the component structure isn't known until runtime.

### Q5: How do you prevent memory leaks with dynamic components?

**Answer:** Always destroy components when they're no longer needed by calling `componentRef.destroy()` or `viewContainer.clear()`. Implement `OnDestroy` to clean up all component references.

### Q6: What is the Portal pattern?

**Answer:** The Portal pattern (from Angular CDK) provides a structured way to render components in different locations. It separates the content (portal) from where it renders (portal outlet).

### Q7: Can you create a component without a ViewContainerRef?

**Answer:** In older Angular versions, you needed ComponentFactory. In modern Angular, `ViewContainerRef.createComponent()` is the standard approach. You need a ViewContainerRef to specify where the component renders.

### Q8: How does change detection work with dynamic components?

**Answer:** Dynamic components participate in the normal change detection cycle. However, when you update properties programmatically, you may need to call `componentRef.changeDetectorRef.detectChanges()` to trigger detection immediately.

## Key Takeaways

1. **Dynamic components are created at runtime** using ViewContainerRef.createComponent()
2. **ComponentRef provides methods** to interact with and manage the component
3. **Always clean up dynamic components** to prevent memory leaks
4. **Use Injectors for complex data passing** to dynamic components
5. **Track component references** for proper lifecycle management
6. **Portal pattern provides structure** for rendering components in different locations
7. **Manually trigger change detection** when updating component properties programmatically
8. **ViewChild is available in ngAfterViewInit**, not in constructor or ngOnInit
9. **Type safety matters** when creating and managing dynamic components
10. **Consider lazy loading** for heavy dynamic components

## Resources

### Official Documentation

- [Dynamic Component Loading](https://angular.dev/guide/components/advanced#dynamic-components)
- [ViewContainerRef API](https://angular.dev/api/core/ViewContainerRef)
- [ComponentRef API](https://angular.dev/api/core/ComponentRef)

### Articles

- "Dynamic Components in Angular" - Angular University
- "Portal Pattern Deep Dive" - Angular CDK Documentation
- "Advanced Dynamic Component Patterns" - Netanel Basal

### Video Tutorials

- "Mastering Dynamic Components" - ng-conf
- "Building Plugin Systems with Angular" - Angular Connect
- "CDK Portals Explained" - Angular Denver

### Libraries

- Angular CDK (Portal/Overlay) - Official dynamic component utilities
- ng-dynamic-component - Community library for dynamic components
- ngx-dynamic-hooks - Parse and render components from strings

### Tools

- Angular DevTools - Inspect dynamic component tree
- Chrome DevTools - Profile component creation performance
