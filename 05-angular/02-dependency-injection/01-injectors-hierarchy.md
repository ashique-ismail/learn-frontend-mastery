# Injectors Hierarchy in Angular

## Table of Contents
- [Introduction](#introduction)
- [Understanding Injectors](#understanding-injectors)
- [Injector Hierarchy Structure](#injector-hierarchy-structure)
- [Platform Injector](#platform-injector)
- [Root Injector](#root-injector)
- [Module Injectors](#module-injectors)
- [Element Injectors](#element-injectors)
- [Resolution Modifiers](#resolution-modifiers)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

Angular's dependency injection system uses a hierarchical structure of injectors to manage and provide dependencies throughout the application. Understanding this hierarchy is crucial for controlling service scope, optimizing performance, and avoiding common pitfalls.

The injector hierarchy determines where services are instantiated, how they're shared between components, and how Angular resolves dependencies when multiple providers exist for the same token.

## Understanding Injectors

An injector is an object that creates and caches service instances. When you request a dependency, Angular:

1. Checks if the injector has a cached instance
2. If not, creates a new instance using the provider
3. Caches the instance for future requests
4. If the provider isn't found, checks the parent injector

```typescript
// Conceptual illustration (actual Angular internals are more complex)
class Injector {
  private instances = new Map<any, any>();
  private providers = new Map<any, Provider>();
  
  constructor(private parent?: Injector) {}
  
  get<T>(token: any): T {
    // Check cache first
    if (this.instances.has(token)) {
      return this.instances.get(token);
    }
    
    // Try to create from provider
    if (this.providers.has(token)) {
      const instance = this.createInstance(token);
      this.instances.set(token, instance);
      return instance;
    }
    
    // Check parent injector
    if (this.parent) {
      return this.parent.get(token);
    }
    
    throw new Error(`No provider for ${token}`);
  }
}
```

## Injector Hierarchy Structure

Angular's injector hierarchy has two main branches:

```
NullInjector (top level - throws errors)
    |
PlatformInjector (platform-level services)
    |
RootInjector (application-level services)
    |
    +-- ModuleInjector (lazy-loaded modules)
    |
    +-- ElementInjector (component/directive level)
         |
         +-- Child ElementInjector (nested components)
```

### Visual Representation

```
Application Structure:          Injector Hierarchy:

index.html                      NullInjector
    |                              |
main.ts                         PlatformInjector
    |                              |
AppComponent                    RootInjector
    |                              |
    +-- HeaderComponent         ElementInjector (App)
    |                              |
    +-- MainComponent           ElementInjector (Header)
    |      |                       |
    |      +-- ListComponent    ElementInjector (Main)
    |                              |
    +-- FooterComponent         ElementInjector (List)
                                   |
                                ElementInjector (Footer)
```

## Platform Injector

The Platform Injector is created when Angular bootstraps and contains platform-specific services shared across multiple Angular applications on the same page.

```typescript
// main.ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

// Creates platform injector
const platform = platformBrowserDynamic([
  {
    provide: PLATFORM_CONFIG,
    useValue: { apiUrl: 'https://api.example.com' }
  }
]);

platform.bootstrapModule(AppModule);
```

**Platform-level services include:**
- `DomSanitizer`
- `DOCUMENT`
- Platform-specific renderers

## Root Injector

The Root Injector is created when bootstrapping the application and contains application-wide singleton services.

### Traditional Module-based Bootstrap

```typescript
// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [
    // These go into Root Injector
    UserService,
    AuthService,
    { provide: API_URL, useValue: 'https://api.example.com' }
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

### Standalone Bootstrap

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';

bootstrapApplication(AppComponent, {
  providers: [
    // These go into Root Injector
    UserService,
    AuthService,
    provideHttpClient(),
    provideRouter(routes)
  ]
});
```

### Services with providedIn: 'root'

```typescript
// Most modern approach - automatically added to Root Injector
@Injectable({
  providedIn: 'root'
})
export class DataService {
  private data: string[] = [];
  
  getData(): string[] {
    return this.data;
  }
  
  addData(item: string): void {
    this.data.push(item);
  }
}

// This service is a singleton shared across the entire application
```

## Module Injectors

Module injectors are created for lazy-loaded modules, creating a new branch in the injector tree.

### Eager-Loaded Modules

```typescript
// Providers in eagerly loaded modules go to Root Injector
@NgModule({
  providers: [SharedService]  // Goes to Root Injector
})
export class SharedModule {}

@NgModule({
  imports: [SharedModule],  // Imported eagerly
  providers: [AppService]
})
export class AppModule {}
```

### Lazy-Loaded Modules

```typescript
// routes.ts
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => 
      import('./admin/admin.module').then(m => m.AdminModule)
  }
];

// admin.module.ts
@NgModule({
  declarations: [AdminComponent, UserManagementComponent],
  providers: [
    AdminService  // Creates a new Module Injector for lazy-loaded module
  ]
})
export class AdminModule {}

// AdminService is a separate instance for the admin module
// Different from root-level services
```

### Module Injector Behavior

```typescript
// logger.service.ts
@Injectable()  // No providedIn - must be provided explicitly
export class LoggerService {
  private logs: string[] = [];
  
  log(message: string): void {
    this.logs.push(message);
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
  
  getLogs(): string[] {
    return [...this.logs];
  }
}

// feature.module.ts (lazy-loaded)
@NgModule({
  providers: [LoggerService]  // New instance for this module
})
export class FeatureModule {}

// another-feature.module.ts (lazy-loaded)
@NgModule({
  providers: [LoggerService]  // Another new instance
})
export class AnotherFeatureModule {}

// Each lazy-loaded module gets its own LoggerService instance
// Logs are NOT shared between modules
```

## Element Injectors

Element injectors are created for each component and directive, forming a parallel hierarchy to the DOM tree.

### Component-Level Providers

```typescript
// parent.component.ts
@Component({
  selector: 'app-parent',
  template: `
    <div>
      <h2>Parent: {{ count }}</h2>
      <button (click)="increment()">Increment</button>
      <app-child></app-child>
      <app-child></app-child>
    </div>
  `,
  providers: [CounterService]  // Creates Element Injector
})
export class ParentComponent {
  constructor(private counter: CounterService) {}
  
  get count(): number {
    return this.counter.getCount();
  }
  
  increment(): void {
    this.counter.increment();
  }
}

// child.component.ts
@Component({
  selector: 'app-child',
  template: `
    <div>
      <h3>Child: {{ count }}</h3>
      <button (click)="increment()">Increment</button>
    </div>
  `
  // No providers - uses parent's injector
})
export class ChildComponent {
  constructor(private counter: CounterService) {}
  
  get count(): number {
    return this.counter.getCount();
  }
  
  increment(): void {
    this.counter.increment();
  }
}

// counter.service.ts
@Injectable()
export class CounterService {
  private count = 0;
  
  increment(): void {
    this.count++;
  }
  
  getCount(): number {
    return this.count;
  }
}

// Both child components share the same CounterService instance
// from the parent's Element Injector
```

### Multiple Component Providers

```typescript
// Each component has its own service instance
@Component({
  selector: 'app-isolated-child',
  template: `
    <div>
      <h3>Isolated Child: {{ count }}</h3>
      <button (click)="increment()">Increment</button>
    </div>
  `,
  providers: [CounterService]  // New instance for this component
})
export class IsolatedChildComponent {
  constructor(private counter: CounterService) {}
  
  get count(): number {
    return this.counter.getCount();
  }
  
  increment(): void {
    this.counter.increment();
  }
}
```

### Directive Providers

```typescript
@Directive({
  selector: '[appLogger]',
  providers: [LoggerService]  // Creates Element Injector for this directive
})
export class LoggerDirective implements OnInit {
  constructor(
    private logger: LoggerService,
    private el: ElementRef
  ) {}
  
  ngOnInit(): void {
    this.logger.log(`Directive initialized on ${this.el.nativeElement.tagName}`);
  }
}

// Usage
@Component({
  selector: 'app-demo',
  template: `
    <div appLogger>Section 1</div>
    <div appLogger>Section 2</div>
  `
})
export class DemoComponent {}
// Each div gets its own LoggerService instance
```

## Resolution Modifiers

Resolution modifiers control how Angular searches the injector hierarchy.

### @Self

Looks only in the current element's injector.

```typescript
@Component({
  selector: 'app-child',
  template: '<p>Child with @Self</p>',
  providers: [DataService]  // Must provide here
})
export class ChildComponent {
  constructor(@Self() private dataService: DataService) {
    // Will only use DataService from this component's injector
    // Throws error if not provided at component level
  }
}
```

### @SkipSelf

Skips the current element's injector and starts from parent.

```typescript
@Component({
  selector: 'app-parent',
  template: '<app-child></app-child>',
  providers: [DataService]
})
export class ParentComponent {}

@Component({
  selector: 'app-child',
  template: '<p>Child with @SkipSelf</p>',
  providers: [DataService]  // This will be skipped
})
export class ChildComponent {
  constructor(@SkipSelf() private dataService: DataService) {
    // Uses parent's DataService, not its own
  }
}
```

### @Optional

Makes dependency optional - injects null if not found.

```typescript
@Component({
  selector: 'app-flexible',
  template: '<p>Flexible component</p>'
})
export class FlexibleComponent {
  constructor(@Optional() private config?: ConfigService) {
    if (this.config) {
      console.log('Config service available');
    } else {
      console.log('No config service - using defaults');
    }
  }
}
```

### @Host

Stops lookup at the host component.

```typescript
@Component({
  selector: 'app-host',
  template: '<app-child></app-child>',
  providers: [ThemeService]
})
export class HostComponent {}

@Component({
  selector: 'app-child',
  template: '<ng-content></ng-content>'
})
export class ChildComponent {
  constructor(@Host() private theme: ThemeService) {
    // Stops looking at HostComponent's injector
    // Won't check beyond the host
  }
}
```

### Combining Modifiers

```typescript
@Component({
  selector: 'app-complex',
  template: '<p>Complex dependencies</p>'
})
export class ComplexComponent {
  constructor(
    @Self() private local: LocalService,
    @SkipSelf() private parent: ParentService,
    @Optional() private optional: OptionalService,
    @Optional() @SkipSelf() private optionalParent?: OptionalParentService,
    @Host() @Optional() private hostOptional?: HostService
  ) {}
}
```

## Practical Examples

### Example 1: Shopping Cart with Scoped Services

```typescript
// cart.service.ts
@Injectable()
export class CartService {
  private items: Product[] = [];
  
  addItem(product: Product): void {
    this.items.push(product);
  }
  
  getItems(): Product[] {
    return [...this.items];
  }
  
  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
  
  clear(): void {
    this.items = [];
  }
}

// shop.component.ts
@Component({
  selector: 'app-shop',
  template: `
    <div class="shop">
      <app-product-list></app-product-list>
      <app-cart-summary></app-cart-summary>
    </div>
  `,
  providers: [CartService]  // Scoped to shop component and children
})
export class ShopComponent {}

// product-list.component.ts
@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products">
      <h3>{{ product.name }}</h3>
      <p>{{ product.price | currency }}</p>
      <button (click)="addToCart(product)">Add to Cart</button>
    </div>
  `
})
export class ProductListComponent {
  products: Product[] = [
    { id: 1, name: 'Product 1', price: 29.99 },
    { id: 2, name: 'Product 2', price: 39.99 }
  ];
  
  constructor(private cart: CartService) {}
  
  addToCart(product: Product): void {
    this.cart.addItem(product);
  }
}

// cart-summary.component.ts
@Component({
  selector: 'app-cart-summary',
  template: `
    <div class="cart">
      <h3>Cart ({{ itemCount }} items)</h3>
      <p>Total: {{ total | currency }}</p>
      <button (click)="checkout()">Checkout</button>
    </div>
  `
})
export class CartSummaryComponent {
  constructor(private cart: CartService) {}
  
  get itemCount(): number {
    return this.cart.getItems().length;
  }
  
  get total(): number {
    return this.cart.getTotal();
  }
  
  checkout(): void {
    console.log('Checking out:', this.cart.getItems());
    this.cart.clear();
  }
}

// All components share the same CartService instance
// from ShopComponent's injector
```

### Example 2: Theme System with Hierarchical Services

```typescript
// theme.service.ts
@Injectable()
export class ThemeService {
  constructor(
    @Optional() @SkipSelf() private parentTheme?: ThemeService
  ) {}
  
  private localTheme?: string;
  
  setTheme(theme: string): void {
    this.localTheme = theme;
  }
  
  getTheme(): string {
    // Use local theme if set, otherwise inherit from parent
    return this.localTheme || this.parentTheme?.getTheme() || 'default';
  }
}

// app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <div [class]="'theme-' + theme">
      <h1>App ({{ theme }})</h1>
      <app-section></app-section>
    </div>
  `,
  providers: [ThemeService]
})
export class AppComponent implements OnInit {
  theme: string = '';
  
  constructor(private themeService: ThemeService) {}
  
  ngOnInit(): void {
    this.themeService.setTheme('light');
    this.theme = this.themeService.getTheme();
  }
}

// section.component.ts
@Component({
  selector: 'app-section',
  template: `
    <div [class]="'theme-' + theme">
      <h2>Section ({{ theme }})</h2>
      <button (click)="toggleTheme()">Toggle Theme</button>
      <app-widget></app-widget>
    </div>
  `,
  providers: [ThemeService]  // New instance, inherits from parent
})
export class SectionComponent implements OnInit {
  theme: string = '';
  private isDark = false;
  
  constructor(private themeService: ThemeService) {}
  
  ngOnInit(): void {
    this.theme = this.themeService.getTheme();  // Inherits 'light'
  }
  
  toggleTheme(): void {
    this.isDark = !this.isDark;
    this.themeService.setTheme(this.isDark ? 'dark' : 'light');
    this.theme = this.themeService.getTheme();
  }
}

// widget.component.ts
@Component({
  selector: 'app-widget',
  template: `
    <div [class]="'theme-' + theme">
      <h3>Widget ({{ theme }})</h3>
    </div>
  `
  // No providers - uses parent SectionComponent's ThemeService
})
export class WidgetComponent implements OnInit {
  theme: string = '';
  
  constructor(private themeService: ThemeService) {}
  
  ngOnInit(): void {
    this.theme = this.themeService.getTheme();
  }
}
```

### Example 3: Configuration Cascade

```typescript
// config.service.ts
@Injectable()
export class ConfigService {
  private config: Record<string, any> = {};
  
  constructor(
    @Optional() @SkipSelf() private parentConfig?: ConfigService
  ) {}
  
  set(key: string, value: any): void {
    this.config[key] = value;
  }
  
  get(key: string): any {
    // Check local config first, then parent
    if (key in this.config) {
      return this.config[key];
    }
    return this.parentConfig?.get(key);
  }
  
  getAll(): Record<string, any> {
    // Merge parent config with local config
    const parentConfig = this.parentConfig?.getAll() || {};
    return { ...parentConfig, ...this.config };
  }
}

// Root level
@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>',
  providers: [
    ConfigService,
    {
      provide: ConfigService,
      useFactory: () => {
        const config = new ConfigService();
        config.set('apiUrl', 'https://api.example.com');
        config.set('timeout', 5000);
        return config;
      }
    }
  ]
})
export class AppComponent {}

// Feature level - overrides some config
@Component({
  selector: 'app-feature',
  template: '<app-feature-child></app-feature-child>',
  providers: [
    {
      provide: ConfigService,
      useFactory: (parent: ConfigService) => {
        const config = new ConfigService(parent);
        config.set('timeout', 10000);  // Override
        config.set('featureFlag', true);  // Add new
        return config;
      },
      deps: [[new SkipSelf(), ConfigService]]
    }
  ]
})
export class FeatureComponent {}

// Child uses merged configuration
@Component({
  selector: 'app-feature-child',
  template: '<div>{{ config | json }}</div>'
})
export class FeatureChildComponent {
  config: Record<string, any>;
  
  constructor(private configService: ConfigService) {
    this.config = this.configService.getAll();
    // { apiUrl: 'https://api.example.com', timeout: 10000, featureFlag: true }
  }
}
```

## Best Practices

### 1. Use providedIn: 'root' for Singleton Services

```typescript
// Good: Tree-shakeable and singleton
@Injectable({
  providedIn: 'root'
})
export class AuthService {}

// Avoid: Manually providing in module unless necessary
@NgModule({
  providers: [AuthService]  // Only if you need module-scoped instance
})
export class AppModule {}
```

### 2. Provide Services at the Right Level

```typescript
// Application-wide: Use root
@Injectable({ providedIn: 'root' })
export class GlobalService {}

// Feature-scoped: Use lazy-loaded module
@NgModule({
  providers: [FeatureService]
})
export class FeatureModule {}

// Component-tree-scoped: Use component
@Component({
  providers: [LocalService]
})
export class LocalComponent {}
```

### 3. Be Explicit with Resolution Modifiers

```typescript
// Good: Clear intent
constructor(
  @Self() private localService: LocalService,
  @Optional() private optionalService?: OptionalService
) {}

// Avoid: Relying on implicit behavior
constructor(
  private service: Service  // Where does this come from?
) {}
```

### 4. Avoid Providing Services in Shared Modules

```typescript
// Bad: Creates multiple instances
@NgModule({
  declarations: [SharedComponent],
  providers: [SharedService]  // Don't do this!
})
export class SharedModule {}

// Good: Let services use providedIn or provide in root
@Injectable({ providedIn: 'root' })
export class SharedService {}
```

## Common Mistakes

### 1. Providing Services in Shared Modules

```typescript
// Wrong: Each importing module gets a new instance
@NgModule({
  declarations: [SharedComponent],
  providers: [LoggerService]  // Multiple instances!
})
export class SharedModule {}

// Imported by multiple modules
@NgModule({
  imports: [SharedModule]  // New LoggerService
})
export class FeatureAModule {}

@NgModule({
  imports: [SharedModule]  // Another new LoggerService
})
export class FeatureBModule {}

// Correct: Use providedIn
@Injectable({ providedIn: 'root' })
export class LoggerService {}
```

### 2. Forgetting @Optional with Conditional Dependencies

```typescript
// Wrong: Throws error if not provided
@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(private analytics: AnalyticsService) {
    // Error if AnalyticsService not provided
  }
}

// Correct: Use @Optional for conditional dependencies
@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(@Optional() private analytics?: AnalyticsService) {
    if (this.analytics) {
      this.analytics.track('component-loaded');
    }
  }
}
```

### 3. Misunderstanding Lazy Module Injectors

```typescript
// Service provided in lazy module
@NgModule({
  providers: [DataService]
})
export class LazyModule {}

// This creates a NEW instance for the lazy module
// NOT shared with root or other lazy modules

// If you want to share, use providedIn: 'root'
@Injectable({ providedIn: 'root' })
export class DataService {}
```

### 4. Incorrect Use of @Host

```typescript
// Wrong: @Host() stops at host component, not module
@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(@Host() private service: RootService) {
    // Won't find RootService provided at root level!
  }
}

// Correct: Use appropriate modifier or no modifier
@Component({
  selector: 'app-child'
})
export class ChildComponent {
  constructor(private service: RootService) {
    // Searches up the entire injector hierarchy
  }
}
```

## Interview Questions

### Basic Questions

**Q1: What is an injector in Angular?**

A: An injector is an object that creates and caches service instances based on providers. It manages the dependency injection system and maintains a hierarchy for dependency resolution.

**Q2: What is the difference between Root Injector and Element Injector?**

A: Root Injector contains application-wide singleton services, while Element Injectors are created per component/directive and form a hierarchy parallel to the DOM tree.

**Q3: What does providedIn: 'root' mean?**

A: It registers the service with the Root Injector, making it a singleton available throughout the application and tree-shakeable if not used.

### Intermediate Questions

**Q4: How does Angular resolve dependencies in the injector hierarchy?**

A: Angular starts at the requesting component's Element Injector, then walks up through parent Element Injectors, then Module Injector, then Root Injector, and finally Platform Injector until it finds the provider or throws an error.

**Q5: What happens when you provide a service in a lazy-loaded module?**

A: A new Module Injector is created for the lazy-loaded module, and the service becomes a singleton within that module branch, not shared with the root or other lazy modules.

**Q6: When should you provide a service at component level vs root level?**

A: Provide at root level for application-wide singletons. Provide at component level when you need separate instances for each component tree or want to isolate state within a component hierarchy.

### Advanced Questions

**Q7: What is the difference between @Self() and @Host()?**

A: @Self() only checks the current element's injector. @Host() checks the current element and walks up to the host component but stops there, not checking beyond the host.

**Q8: How can you make a service inherit from a parent injector?**

A: Inject the parent service using @Optional() @SkipSelf(), then delegate to it or merge its state with local state.

**Q9: Why should you avoid providing services in shared modules?**

A: Each module that imports the shared module gets its own instance of the service, creating multiple instances instead of a singleton, which is usually not the desired behavior.

**Q10: How do you test components with complex injector hierarchies?**

A: Use TestBed to configure the injector hierarchy, provide mock services at appropriate levels, and use resolution modifiers in tests to control dependency resolution.

## Resources

### Official Documentation
- [Dependency Injection in Angular](https://angular.dev/guide/di)
- [Hierarchical Injectors](https://angular.dev/guide/di/hierarchical-dependency-injection)

### Articles and Tutorials
- [Understanding Angular Injector Hierarchy](https://blog.angular.io/injector-hierarchy)
- [Advanced DI Patterns](https://medium.com/angular-in-depth/di-patterns)

### Video Tutorials
- [Mastering Angular DI](https://www.youtube.com/watch?v=di-mastery)
- [Injector Hierarchy Deep Dive](https://www.youtube.com/watch?v=injector-deep-dive)

### Books
- "Angular Development with TypeScript" - Chapter on Dependency Injection
- "ng-book: The Complete Guide to Angular" - DI sections

### Tools
- [Angular DevTools](https://angular.dev/tools/devtools) - Inspect injector hierarchy
- [Augury](https://augury.rangle.io/) - Visualize injectors
