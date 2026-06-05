# Injection Decorators in Angular

## The Idea

**In plain English:** Injection decorators are special labels you put on a component's requests for tools (services) to tell Angular exactly where to look for them — for example, "only check your own toolbox" or "skip your own and ask your parent instead." A decorator is just a tag that changes how something behaves.

**Real-world analogy:** Imagine you're a new employee at a company. When you need a stapler, you can ask in different ways: "Do I have one at my own desk?" (@Self), "Ask my manager's floor but skip my own desk?" (@SkipSelf), "Stop asking once you reach the department head?" (@Host), or "It's okay if nobody has one, I'll manage without." (@Optional).

- The employee's own desk = the component's own injector
- The company floor hierarchy (desk → manager → department head → CEO) = Angular's injector tree
- The stapler = the service being requested
- The rule about where to stop searching = the resolution modifier decorator

---

## Table of Contents

- [Introduction](#introduction)
- [Resolution Modifiers Overview](#resolution-modifiers-overview)
- [@Optional Decorator](#optional-decorator)
- [@Self Decorator](#self-decorator)
- [@SkipSelf Decorator](#skipself-decorator)
- [@Host Decorator](#host-decorator)
- [Combining Decorators](#combining-decorators)
- [Resolution Modifier Hierarchy](#resolution-modifier-hierarchy)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Angular's dependency injection system provides resolution modifiers that control how the injector searches for dependencies. These decorators give you fine-grained control over where Angular looks for providers and how it handles missing dependencies.

Understanding resolution modifiers is crucial for:
- Preventing unintended dependency sharing
- Handling optional dependencies gracefully
- Creating isolated component contexts
- Building reusable component libraries
- Optimizing dependency resolution

## Resolution Modifiers Overview

Angular provides four main resolution modifier decorators:

```typescript
import { Component, Optional, Self, SkipSelf, Host, Inject } from '@angular/core';

@Component({
  selector: 'app-example',
  template: '<p>Resolution modifiers demo</p>'
})
export class ExampleComponent {
  constructor(
    @Optional() private optionalService: OptionalService,
    @Self() private selfService: SelfService,
    @SkipSelf() private parentService: ParentService,
    @Host() private hostService: HostService
  ) {}
}
```

Each modifier changes the injector's search strategy:
- **@Optional**: Makes dependency optional, returns null if not found
- **@Self**: Only looks in the component's own injector
- **@SkipSelf**: Skips component's injector, starts with parent
- **@Host**: Stops search at the host component boundary

## @Optional Decorator

### Basic Usage

`@Optional` tells Angular that a dependency is not required. If the provider isn't found, Angular injects `null` instead of throwing an error.

```typescript
import { Component, Optional } from '@angular/core';
import { LoggerService } from './logger.service';

@Component({
  selector: 'app-optional-example',
  template: '<p>Check console for logs</p>'
})
export class OptionalExampleComponent {
  constructor(@Optional() private logger: LoggerService | null) {
    // Safe to use - logger might be null
    if (this.logger) {
      this.logger.log('Logger is available');
    } else {
      console.log('Logger not available, using console');
    }
  }
  
  logMessage(message: string): void {
    this.logger?.log(message) ?? console.log(message);
  }
}
```

### Feature Toggle Pattern

```typescript
import { Injectable, Optional } from '@angular/core';

@Injectable()
export class FeatureFlagService {
  isEnabled(flag: string): boolean {
    // Feature flag logic
    return false;
  }
}

@Component({
  selector: 'app-feature',
  template: `
    <div *ngIf="isFeatureEnabled">
      <h2>New Feature</h2>
      <p>This feature is only visible when enabled</p>
    </div>
  `
})
export class FeatureComponent {
  isFeatureEnabled = false;
  
  constructor(@Optional() private featureFlags: FeatureFlagService | null) {
    // Gracefully handle missing feature flag service
    this.isFeatureEnabled = this.featureFlags?.isEnabled('new-feature') ?? false;
  }
}
```

### Configuration Pattern

```typescript
import { InjectionToken, Optional, Inject } from '@angular/core';

export interface AppConfig {
  apiUrl: string;
  enableLogging: boolean;
  timeout: number;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Default configuration
const DEFAULT_CONFIG: AppConfig = {
  apiUrl: 'https://api.example.com',
  enableLogging: false,
  timeout: 30000
};

@Injectable({ providedIn: 'root' })
export class ApiService {
  private config: AppConfig;
  
  constructor(
    @Optional() @Inject(APP_CONFIG) config: AppConfig | null
  ) {
    // Use provided config or fall back to defaults
    this.config = config ?? DEFAULT_CONFIG;
  }
  
  getApiUrl(): string {
    return this.config.apiUrl;
  }
}
```

### Optional with inject() Function

```typescript
import { inject, Optional } from '@angular/core';
import { LoggerService } from './logger.service';

@Injectable({ providedIn: 'root' })
export class DataService {
  // Using inject() with Optional flag
  private logger = inject(LoggerService, { optional: true });
  
  fetchData(): void {
    this.logger?.log('Fetching data...') ?? console.log('Fetching data...');
  }
}
```

## @Self Decorator

### Basic Usage

`@Self` restricts dependency resolution to the component's own injector. Angular won't look up the injector hierarchy.

```typescript
import { Component, Self, Injectable } from '@angular/core';

@Injectable()
export class ComponentStateService {
  private state = new Map<string, any>();
  
  setState(key: string, value: any): void {
    this.state.set(key, value);
  }
  
  getState(key: string): any {
    return this.state.get(key);
  }
}

@Component({
  selector: 'app-self-example',
  template: '<p>Component with isolated state</p>',
  providers: [ComponentStateService] // Must provide here
})
export class SelfExampleComponent {
  constructor(
    // Only uses ComponentStateService from this component's injector
    @Self() private state: ComponentStateService
  ) {
    this.state.setState('initialized', true);
  }
}
```

### Isolated Component Instances

```typescript
import { Component, Self, Injectable } from '@angular/core';

@Injectable()
export class FormStateService {
  private isDirty = false;
  private errors: string[] = [];
  
  markDirty(): void {
    this.isDirty = true;
  }
  
  addError(error: string): void {
    this.errors.push(error);
  }
  
  isValid(): boolean {
    return this.errors.length === 0;
  }
}

@Component({
  selector: 'app-form-section',
  template: `
    <div class="form-section">
      <ng-content></ng-content>
      <p *ngIf="!isValid()">Form has errors</p>
    </div>
  `,
  providers: [FormStateService]
})
export class FormSectionComponent {
  constructor(@Self() private formState: FormStateService) {}
  
  isValid(): boolean {
    return this.formState.isValid();
  }
  
  validate(): void {
    this.formState.markDirty();
    // Validation logic
  }
}
```

### Preventing Service Sharing

```typescript
import { Component, Self, Injectable } from '@angular/core';

@Injectable()
export class SelectionService {
  private selectedItems = new Set<string>();
  
  select(item: string): void {
    this.selectedItems.add(item);
  }
  
  deselect(item: string): void {
    this.selectedItems.delete(item);
  }
  
  isSelected(item: string): boolean {
    return this.selectedItems.has(item);
  }
  
  getSelection(): string[] {
    return Array.from(this.selectedItems);
  }
}

@Component({
  selector: 'app-list',
  template: `
    <div *ngFor="let item of items">
      <input type="checkbox" 
             [checked]="isSelected(item)"
             (change)="toggleItem(item)">
      {{ item }}
    </div>
  `,
  providers: [SelectionService] // Each list has its own selection
})
export class ListComponent {
  items = ['Item 1', 'Item 2', 'Item 3'];
  
  constructor(
    @Self() private selection: SelectionService
  ) {}
  
  toggleItem(item: string): void {
    if (this.selection.isSelected(item)) {
      this.selection.deselect(item);
    } else {
      this.selection.select(item);
    }
  }
  
  isSelected(item: string): boolean {
    return this.selection.isSelected(item);
  }
}
```

### @Self with inject()

```typescript
import { inject, Self } from '@angular/core';
import { ComponentStateService } from './component-state.service';

@Component({
  selector: 'app-modern-self',
  template: '<p>Modern self injection</p>',
  providers: [ComponentStateService]
})
export class ModernSelfComponent {
  private state = inject(ComponentStateService, { self: true });
  
  ngOnInit(): void {
    this.state.setState('ready', true);
  }
}
```

## @SkipSelf Decorator

### Basic Usage

`@SkipSelf` tells Angular to skip the current injector and start searching from the parent injector.

```typescript
import { Component, SkipSelf, Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class GlobalConfigService {
  getConfig(): any {
    return { theme: 'dark', lang: 'en' };
  }
}

@Component({
  selector: 'app-child',
  template: '<p>Child component</p>',
  providers: [
    { 
      provide: GlobalConfigService, 
      useValue: { getConfig: () => ({ theme: 'light' }) } 
    }
  ]
})
export class ChildComponent {
  constructor(
    // Skips local provider, gets parent's GlobalConfigService
    @SkipSelf() private config: GlobalConfigService
  ) {
    console.log(this.config.getConfig()); // Uses parent's config
  }
}
```

### Parent-Child Communication

```typescript
import { Component, SkipSelf, Optional, Injectable } from '@angular/core';

@Injectable()
export class BreadcrumbService {
  private path: string[] = [];
  
  addCrumb(label: string): void {
    this.path.push(label);
  }
  
  getPath(): string[] {
    return [...this.path];
  }
}

@Component({
  selector: 'app-root-breadcrumb',
  template: `
    <nav>{{ breadcrumb.getPath().join(' > ') }}</nav>
    <ng-content></ng-content>
  `,
  providers: [BreadcrumbService]
})
export class RootBreadcrumbComponent {
  constructor(public breadcrumb: BreadcrumbService) {}
}

@Component({
  selector: 'app-page',
  template: '<ng-content></ng-content>',
  providers: [BreadcrumbService]
})
export class PageComponent {
  constructor(
    private breadcrumb: BreadcrumbService,
    @Optional() @SkipSelf() private parentBreadcrumb: BreadcrumbService | null
  ) {
    // Inherit parent breadcrumbs
    if (this.parentBreadcrumb) {
      this.parentBreadcrumb.getPath().forEach(crumb => {
        this.breadcrumb.addCrumb(crumb);
      });
    }
    this.breadcrumb.addCrumb('Current Page');
  }
}
```

### Configuration Inheritance

```typescript
import { Component, SkipSelf, Optional, Injectable } from '@angular/core';

interface ThemeConfig {
  primaryColor: string;
  secondaryColor: string;
  fontSize: number;
}

@Injectable()
export class ThemeService {
  constructor(private config: ThemeConfig) {}
  
  getConfig(): ThemeConfig {
    return this.config;
  }
  
  updateConfig(updates: Partial<ThemeConfig>): ThemeConfig {
    return { ...this.config, ...updates };
  }
}

@Component({
  selector: 'app-themed-section',
  template: `
    <div [style.color]="theme.getConfig().primaryColor">
      <ng-content></ng-content>
    </div>
  `,
  providers: [
    {
      provide: ThemeService,
      useFactory: (parent: ThemeService | null) => {
        const parentConfig = parent?.getConfig() ?? {
          primaryColor: '#000',
          secondaryColor: '#666',
          fontSize: 16
        };
        // Override only fontSize
        return new ThemeService({ ...parentConfig, fontSize: 14 });
      },
      deps: [[new Optional(), new SkipSelf(), ThemeService]]
    }
  ]
})
export class ThemedSectionComponent {
  constructor(public theme: ThemeService) {}
}
```

### @SkipSelf with inject()

```typescript
import { inject, SkipSelf } from '@angular/core';
import { ConfigService } from './config.service';

@Component({
  selector: 'app-skip-self-modern',
  template: '<p>Modern skip self</p>'
})
export class SkipSelfModernComponent {
  private parentConfig = inject(ConfigService, { skipSelf: true });
  
  ngOnInit(): void {
    console.log('Parent config:', this.parentConfig);
  }
}
```

## @Host Decorator

### Basic Usage

`@Host` limits dependency resolution to the host component. It stops searching at the component that contains the `ng-content` or dynamic component.

```typescript
import { Component, Host, Injectable } from '@angular/core';

@Injectable()
export class FormValidationService {
  private validators: any[] = [];
  
  addValidator(validator: any): void {
    this.validators.push(validator);
  }
  
  validate(): boolean {
    return this.validators.every(v => v.isValid());
  }
}

@Component({
  selector: 'app-form',
  template: `
    <form>
      <ng-content></ng-content>
      <button (click)="submit()">Submit</button>
    </form>
  `,
  providers: [FormValidationService]
})
export class FormComponent {
  constructor(private validation: FormValidationService) {}
  
  submit(): void {
    if (this.validation.validate()) {
      console.log('Form is valid');
    }
  }
}

@Component({
  selector: 'app-input-field',
  template: '<input [formControl]="control">'
})
export class InputFieldComponent {
  control = new FormControl('');
  
  constructor(
    // Looks up to the host component (FormComponent)
    @Host() private formValidation: FormValidationService
  ) {
    this.formValidation.addValidator(this);
  }
  
  isValid(): boolean {
    return this.control.valid;
  }
}
```

### Content Projection Boundaries

```typescript
import { Component, Host, Optional, Injectable } from '@angular/core';

@Injectable()
export class AccordionService {
  private openPanelId: string | null = null;
  
  openPanel(id: string): void {
    this.openPanelId = id;
  }
  
  isPanelOpen(id: string): boolean {
    return this.openPanelId === id;
  }
}

@Component({
  selector: 'app-accordion',
  template: '<div class="accordion"><ng-content></ng-content></div>',
  providers: [AccordionService]
})
export class AccordionComponent {
  constructor(public accordion: AccordionService) {}
}

@Component({
  selector: 'app-accordion-panel',
  template: `
    <div class="panel">
      <div class="header" (click)="toggle()">{{ title }}</div>
      <div class="content" *ngIf="isOpen()">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class AccordionPanelComponent {
  @Input() title = '';
  @Input() id = '';
  
  constructor(
    // Only looks up to the accordion host, not beyond
    @Host() private accordion: AccordionService
  ) {}
  
  toggle(): void {
    this.accordion.openPanel(this.id);
  }
  
  isOpen(): boolean {
    return this.accordion.isPanelOpen(this.id);
  }
}
```

### Dynamic Component Context

```typescript
import { Component, Host, ViewContainerRef, Injectable } from '@angular/core';

@Injectable()
export class DialogContext {
  constructor(public data: any) {}
  
  close(): void {
    // Close dialog logic
  }
}

@Component({
  selector: 'app-dialog',
  template: '<div class="dialog"><ng-content></ng-content></div>',
  providers: [DialogContext]
})
export class DialogComponent {
  constructor(context: DialogContext) {
    context.data = { title: 'Dialog Title' };
  }
}

@Component({
  selector: 'app-dialog-content',
  template: `
    <h2>{{ context.data.title }}</h2>
    <button (click)="close()">Close</button>
  `
})
export class DialogContentComponent {
  constructor(
    // Gets DialogContext from host DialogComponent
    @Host() public context: DialogContext
  ) {}
  
  close(): void {
    this.context.close();
  }
}
```

### @Host with inject()

```typescript
import { inject, Host } from '@angular/core';
import { FormValidationService } from './form-validation.service';

@Component({
  selector: 'app-host-modern',
  template: '<input>'
})
export class HostModernComponent {
  private formValidation = inject(FormValidationService, { host: true });
  
  ngOnInit(): void {
    this.formValidation.addValidator(this);
  }
}
```

## Combining Decorators

### @Optional with @Self

```typescript
import { Component, Optional, Self, Injectable } from '@angular/core';

@Injectable()
export class LocalStorageService {
  save(key: string, value: any): void {
    // Local storage logic
  }
}

@Component({
  selector: 'app-combined',
  template: '<p>Combined decorators</p>'
})
export class CombinedComponent {
  constructor(
    // Only looks in own injector, doesn't throw if not found
    @Optional() @Self() private storage: LocalStorageService | null
  ) {
    if (this.storage) {
      this.storage.save('key', 'value');
    } else {
      console.log('No local storage service provided');
    }
  }
}
```

### @Optional with @SkipSelf

```typescript
import { Component, Optional, SkipSelf, Injectable } from '@angular/core';

@Injectable()
export class LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

@Component({
  selector: 'app-child-logger',
  template: '<p>Child with optional parent logger</p>',
  providers: [LoggerService]
})
export class ChildLoggerComponent {
  constructor(
    private logger: LoggerService,
    // Gets parent logger if available, null otherwise
    @Optional() @SkipSelf() private parentLogger: LoggerService | null
  ) {
    this.logger.log('Child logger message');
    this.parentLogger?.log('Parent logger message');
  }
}
```

### @Optional with @Host

```typescript
import { Component, Optional, Host, Injectable, Directive } from '@angular/core';

@Injectable()
export class TabGroupService {
  selectTab(id: string): void {
    console.log('Selected tab:', id);
  }
}

@Component({
  selector: 'app-tab-group',
  template: '<ng-content></ng-content>',
  providers: [TabGroupService]
})
export class TabGroupComponent {}

@Component({
  selector: 'app-tab',
  template: '<div (click)="select()"><ng-content></ng-content></div>'
})
export class TabComponent {
  @Input() id = '';
  
  constructor(
    // Only looks in host, optional if tab is standalone
    @Optional() @Host() private tabGroup: TabGroupService | null
  ) {}
  
  select(): void {
    if (this.tabGroup) {
      this.tabGroup.selectTab(this.id);
    } else {
      console.log('Tab is not inside a tab group');
    }
  }
}
```

### Complex Resolution Pattern

```typescript
import { Component, Optional, Self, SkipSelf, Host, Injectable } from '@angular/core';

@Injectable()
export class ConfigService {
  constructor(public config: any) {}
}

@Component({
  selector: 'app-complex',
  template: '<p>Complex resolution</p>',
  providers: [
    {
      provide: ConfigService,
      useFactory: (
        ownConfig: ConfigService | null,
        parentConfig: ConfigService | null,
        hostConfig: ConfigService | null
      ) => {
        // Merge configurations with priority
        return new ConfigService({
          ...hostConfig?.config,
          ...parentConfig?.config,
          ...ownConfig?.config
        });
      },
      deps: [
        [new Optional(), new Self(), ConfigService],
        [new Optional(), new SkipSelf(), ConfigService],
        [new Optional(), new Host(), ConfigService]
      ]
    }
  ]
})
export class ComplexComponent {
  constructor(private config: ConfigService) {
    console.log('Merged config:', this.config.config);
  }
}
```

## Resolution Modifier Hierarchy

### Search Order Visualization

```typescript
/*
Component Tree:
  AppComponent (provides ServiceA)
    └─ ParentComponent (provides ServiceB)
        └─ HostComponent (provides ServiceC)
            └─ ChildComponent (injects dependencies)

Resolution without modifiers:
ChildComponent → HostComponent → ParentComponent → AppComponent → Root

With @Self:
ChildComponent only

With @SkipSelf:
HostComponent → ParentComponent → AppComponent → Root

With @Host:
ChildComponent → HostComponent (stops here)
*/

@Injectable()
export class ServiceA {
  name = 'Service A';
}

@Injectable()
export class ServiceB {
  name = 'Service B';
}

@Injectable()
export class ServiceC {
  name = 'Service C';
}

@Component({
  selector: 'app-root',
  template: '<app-parent></app-parent>',
  providers: [ServiceA]
})
export class AppComponent {}

@Component({
  selector: 'app-parent',
  template: '<app-host></app-host>',
  providers: [ServiceB]
})
export class ParentComponent {}

@Component({
  selector: 'app-host',
  template: '<app-child></app-child>',
  providers: [ServiceC]
})
export class HostComponent {}

@Component({
  selector: 'app-child',
  template: '<p>Child</p>'
})
export class ChildComponent {
  constructor(
    serviceA: ServiceA,                // Found in AppComponent
    serviceB: ServiceB,                // Found in ParentComponent
    serviceC: ServiceC,                // Found in HostComponent
    @Self() @Optional() serviceD: any, // null (not in ChildComponent)
    @SkipSelf() serviceC2: ServiceC,   // Found in HostComponent (skipped self)
    @Host() serviceC3: ServiceC        // Found in HostComponent (stops at host)
  ) {}
}
```

### Resolution Decision Tree

```typescript
import { Injectable, Optional, Self, SkipSelf, Host } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ResolutionDemoService {
  demonstrateResolution(): void {
    /*
    Decision Tree:
    
    1. No modifier: Search from current → parent → root
    2. @Optional: Same as 1, but return null if not found
    3. @Self: Only search current injector
    4. @Self + @Optional: Search current, return null if not found
    5. @SkipSelf: Search from parent → root (skip current)
    6. @SkipSelf + @Optional: Same as 5, return null if not found
    7. @Host: Search from current → host boundary
    8. @Host + @Optional: Same as 7, return null if not found
    */
  }
}
```

## Common Mistakes

### Mistake 1: Using @Self Without Providing

```typescript
// BAD: @Self requires provider in component
@Component({
  selector: 'app-bad-self',
  template: '<p>Bad</p>'
  // Missing providers!
})
export class BadSelfComponent {
  constructor(
    @Self() private service: MyService // Will throw error!
  ) {}
}

// GOOD: Provide service in component
@Component({
  selector: 'app-good-self',
  template: '<p>Good</p>',
  providers: [MyService]
})
export class GoodSelfComponent {
  constructor(
    @Self() private service: MyService
  ) {}
}
```

### Mistake 2: Forgetting @Optional with @Self

```typescript
// BAD: Might throw if not provided
@Component({
  selector: 'app-bad-optional',
  template: '<p>Bad</p>'
})
export class BadOptionalComponent {
  constructor(
    @Self() private service: OptionalService // Error if not provided!
  ) {}
}

// GOOD: Use @Optional with @Self
@Component({
  selector: 'app-good-optional',
  template: '<p>Good</p>'
})
export class GoodOptionalComponent {
  constructor(
    @Optional() @Self() private service: OptionalService | null
  ) {
    if (this.service) {
      // Use service
    }
  }
}
```

### Mistake 3: Misunderstanding @Host Scope

```typescript
// BAD: Expecting @Host to search beyond host boundary
@Component({
  selector: 'app-container',
  template: '<app-host-demo></app-host-demo>',
  providers: [ContainerService]
})
export class ContainerComponent {}

@Component({
  selector: 'app-host-boundary',
  template: '<ng-content></ng-content>'
})
export class HostBoundaryComponent {}

@Component({
  selector: 'app-host-demo',
  template: '<p>Demo</p>'
})
export class HostDemoComponent {
  constructor(
    @Host() private service: ContainerService // Won't find it!
  ) {}
}

// GOOD: Use @SkipSelf for parent services
@Component({
  selector: 'app-host-demo-fixed',
  template: '<p>Demo</p>'
})
export class HostDemoFixedComponent {
  constructor(
    @SkipSelf() private service: ContainerService
  ) {}
}
```

### Mistake 4: Incorrect Decorator Order

```typescript
// BAD: Wrong order (decorators apply right to left)
@Component({
  selector: 'app-bad-order',
  template: '<p>Bad</p>'
})
export class BadOrderComponent {
  constructor(
    @Self() @Optional() private service: MyService // Works but inconsistent
  ) {}
}

// GOOD: Consistent order (Optional first)
@Component({
  selector: 'app-good-order',
  template: '<p>Good</p>'
})
export class GoodOrderComponent {
  constructor(
    @Optional() @Self() private service: MyService | null
  ) {}
}
```

### Mistake 5: Not Handling Null from @Optional

```typescript
// BAD: Not checking for null
@Component({
  selector: 'app-bad-null',
  template: '<p>Bad</p>'
})
export class BadNullComponent {
  constructor(
    @Optional() private service: MyService | null
  ) {
    this.service.doSomething(); // Potential null pointer!
  }
}

// GOOD: Always check for null
@Component({
  selector: 'app-good-null',
  template: '<p>Good</p>'
})
export class GoodNullComponent {
  constructor(
    @Optional() private service: MyService | null
  ) {
    this.service?.doSomething();
  }
}
```

## Best Practices

### 1. Use @Optional for Non-Critical Dependencies

```typescript
@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  constructor(
    @Optional() private errorReporter: ErrorReporter | null,
    @Optional() private performanceMonitor: PerformanceMonitor | null
  ) {}
  
  trackEvent(event: string): void {
    // Main functionality works without optional dependencies
    console.log('Event:', event);
    
    this.performanceMonitor?.recordEvent(event);
  }
  
  reportError(error: Error): void {
    console.error(error);
    this.errorReporter?.report(error);
  }
}
```

### 2. Use @Self for Component-Specific State

```typescript
@Injectable()
export class ComponentStateService {
  private state = new Map<string, any>();
  
  set(key: string, value: any): void {
    this.state.set(key, value);
  }
  
  get(key: string): any {
    return this.state.get(key);
  }
}

@Component({
  selector: 'app-stateful',
  template: '<ng-content></ng-content>',
  providers: [ComponentStateService]
})
export class StatefulComponent {
  constructor(@Self() private state: ComponentStateService) {}
}
```

### 3. Use @SkipSelf for Parent Communication

```typescript
@Injectable()
export class HierarchyService {
  private children: HierarchyService[] = [];
  
  constructor(
    @Optional() @SkipSelf() private parent: HierarchyService | null
  ) {
    this.parent?.addChild(this);
  }
  
  addChild(child: HierarchyService): void {
    this.children.push(child);
  }
  
  getAncestors(): HierarchyService[] {
    return this.parent ? [this.parent, ...this.parent.getAncestors()] : [];
  }
}
```

### 4. Use @Host for Content Projection Boundaries

```typescript
@Injectable()
export class ListContext {
  items: any[] = [];
  
  addItem(item: any): void {
    this.items.push(item);
  }
}

@Component({
  selector: 'app-list',
  template: '<div><ng-content></ng-content></div>',
  providers: [ListContext]
})
export class ListComponent {
  constructor(public context: ListContext) {}
}

@Component({
  selector: 'app-list-item',
  template: '<div>{{ data }}</div>'
})
export class ListItemComponent {
  @Input() data: any;
  
  constructor(@Host() private listContext: ListContext) {
    this.listContext.addItem(this);
  }
}
```

### 5. Combine Decorators Thoughtfully

```typescript
@Injectable()
export class ConfigurationService {
  constructor(public config: any) {}
}

@Component({
  selector: 'app-configured',
  template: '<p>Configured</p>',
  providers: [
    {
      provide: ConfigurationService,
      useFactory: (
        parentConfig: ConfigurationService | null,
        localConfig: ConfigurationService | null
      ) => {
        // Merge parent and local configs
        const merged = {
          ...(parentConfig?.config ?? {}),
          ...(localConfig?.config ?? {})
        };
        return new ConfigurationService(merged);
      },
      deps: [
        [new Optional(), new SkipSelf(), ConfigurationService],
        [new Optional(), new Self(), ConfigurationService]
      ]
    }
  ]
})
export class ConfiguredComponent {}
```

## Interview Questions

### Q1: What is the difference between @Self and @SkipSelf?

**Answer:** `@Self` restricts dependency resolution to the component's own injector, while `@SkipSelf` skips the component's injector and starts searching from the parent. `@Self` is useful for component-specific instances, while `@SkipSelf` is useful for accessing parent dependencies.

### Q2: When would you use @Optional?

**Answer:** Use `@Optional` when:
- A dependency is not critical for functionality
- You want to provide fallback behavior
- You're building a library that may not have all dependencies available
- You want to avoid throwing errors in certain configurations

### Q3: How does @Host differ from @SkipSelf?

**Answer:** `@Host` stops searching at the host component boundary (the component containing `ng-content`), while `@SkipSelf` continues up the entire injector hierarchy. `@Host` is useful for content projection scenarios where you want to limit dependency scope.

### Q4: Can you combine multiple resolution modifiers?

**Answer:** Yes, you can combine modifiers like `@Optional() @Self()` or `@Optional() @SkipSelf()`. The modifiers work together to control resolution behavior. Common combinations include using `@Optional` with other modifiers to prevent errors when dependencies aren't found.

### Q5: What happens if you use @Self without providing the service?

**Answer:** Angular will throw a `NullInjectorError` because `@Self` restricts the search to the component's own injector. If the service isn't provided there, it cannot be found. Use `@Optional() @Self()` to avoid this error.

## Key Takeaways

1. **@Optional** makes dependencies optional, returning null if not found
2. **@Self** restricts resolution to component's own injector
3. **@SkipSelf** skips component's injector and searches parent chain
4. **@Host** stops resolution at host component boundary
5. Decorators can be combined for complex resolution strategies
6. Always handle null values when using @Optional
7. Use @Self for component-specific service instances
8. Use @SkipSelf for parent-child communication
9. Use @Host for content projection boundaries
10. Resolution modifiers provide fine-grained control over DI

## Resources

- [Angular Dependency Injection Guide](https://angular.dev/guide/di)
- [Resolution Modifiers Documentation](https://angular.dev/guide/di/dependency-injection-providers#resolution-modifiers)
- [Hierarchical Injectors](https://angular.dev/guide/di/hierarchical-dependency-injection)
- [Angular DI Deep Dive](https://blog.angular.io/angular-dependency-injection-deep-dive)
- [inject() Function Guide](https://angular.dev/api/core/inject)
