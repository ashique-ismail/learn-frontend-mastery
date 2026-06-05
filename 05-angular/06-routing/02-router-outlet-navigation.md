# Router Outlet and Navigation in Angular

## The Idea

**In plain English:** A router outlet is a reserved spot on a webpage where Angular swaps in different "pages" (called components) depending on which web address (URL) you visit — without ever reloading the whole page. Navigation is how you move between those pages, either by clicking a link or triggering it with code.

**Real-world analogy:** Think of a digital photo frame in your living room that displays different photos based on which button you press on a remote. The frame stays on the wall, but the picture inside changes.

- The photo frame on the wall = the `<router-outlet>` tag (fixed placeholder in the layout)
- Each photo = a component (the actual content shown for a given route)
- The remote control buttons = `routerLink` directives and `Router.navigate()` calls (the navigation triggers)

---

## Table of Contents

- [Introduction](#introduction)
- [Router Outlet](#router-outlet)
- [RouterLink Directive](#routerlink-directive)
- [RouterLinkActive Directive](#routerlinkactive-directive)
- [Programmatic Navigation](#programmatic-navigation)
- [Multiple Router Outlets](#multiple-router-outlets)
- [Navigation Extras](#navigation-extras)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Router outlet and navigation are fundamental concepts in Angular routing. The router outlet acts as a placeholder where routed components are rendered, while navigation mechanisms allow users to move between different views in your application.

## Router Outlet

### Basic Router Outlet

The `<router-outlet>` directive marks where the router should display views:

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <header>
      <h1>My Application</h1>
      <nav>
        <!-- Navigation links here -->
      </nav>
    </header>
    
    <main>
      <router-outlet></router-outlet>
    </main>
    
    <footer>
      <p>Copyright 2024</p>
    </footer>
  `
})
export class AppComponent {}
```

### Router Outlet Events

Router outlet emits events during activation and deactivation:

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <router-outlet 
      (activate)="onActivate($event)"
      (deactivate)="onDeactivate($event)">
    </router-outlet>
  `
})
export class AppComponent {
  onActivate(component: any): void {
    console.log('Component activated:', component);
    // Scroll to top on route change
    window.scrollTo(0, 0);
  }

  onDeactivate(component: any): void {
    console.log('Component deactivated:', component);
    // Cleanup or save state
  }
}
```

### Accessing Router Outlet Programmatically

```typescript
// app.component.ts
import { Component, ViewChild } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `<router-outlet #outlet="outlet"></router-outlet>`
})
export class AppComponent {
  @ViewChild(RouterOutlet) outlet!: RouterOutlet;

  ngAfterViewInit(): void {
    console.log('Current component:', this.outlet.component);
    console.log('Is activated:', this.outlet.isActivated);
    console.log('Activated route:', this.outlet.activatedRoute);
  }
}
```

## RouterLink Directive

### Basic RouterLink Usage

```typescript
// navigation.component.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-navigation',
  standalone: true,
  imports: [RouterLink],
  template: `
    <nav>
      <!-- Simple string path -->
      <a routerLink="/home">Home</a>
      
      <!-- Array syntax for complex paths -->
      <a [routerLink]="['/users', userId]">User Profile</a>
      
      <!-- Relative path -->
      <a routerLink="../parent">Go to Parent</a>
      
      <!-- Root-relative path -->
      <a routerLink="/products">Products</a>
    </nav>
  `
})
export class NavigationComponent {
  userId = 123;
}
```

### RouterLink with Query Parameters

```typescript
// product-list.component.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div class="products">
      <a 
        [routerLink]="['/products', product.id]"
        [queryParams]="{ category: product.category, sort: 'price' }"
        [fragment]="'reviews'"
        *ngFor="let product of products">
        {{ product.name }}
      </a>
    </div>
  `
})
export class ProductListComponent {
  products = [
    { id: 1, name: 'Laptop', category: 'electronics' },
    { id: 2, name: 'Phone', category: 'electronics' }
  ];
}
```

### RouterLink with State

```typescript
// user-list.component.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div class="users">
      <a 
        [routerLink]="['/users', user.id]"
        [state]="{ userData: user, source: 'list' }">
        {{ user.name }}
      </a>
    </div>
  `
})
export class UserListComponent {
  users = [
    { id: 1, name: 'John Doe', email: 'john@example.com' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
  ];
}

// user-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-user-detail',
  standalone: true,
  template: `<div>{{ userData | json }}</div>`
})
export class UserDetailComponent implements OnInit {
  userData: any;

  constructor(private router: Router) {
    const navigation = this.router.getCurrentNavigation();
    this.userData = navigation?.extras.state?.['userData'];
  }

  ngOnInit(): void {
    console.log('Received user data:', this.userData);
  }
}
```

### Query Parameters Handling

```typescript
// search.component.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-search',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div class="filters">
      <!-- Merge with existing params -->
      <a 
        routerLink="/search"
        [queryParams]="{ page: 1 }"
        queryParamsHandling="merge">
        Page 1
      </a>
      
      <!-- Preserve existing params -->
      <a 
        routerLink="/search"
        [queryParams]="{ sort: 'date' }"
        queryParamsHandling="preserve">
        Sort by Date
      </a>
      
      <!-- Replace all params (default) -->
      <a 
        routerLink="/search"
        [queryParams]="{ q: 'angular' }">
        Search Angular
      </a>
    </div>
  `
})
export class SearchComponent {}
```

## RouterLinkActive Directive

### Basic RouterLinkActive Usage

```typescript
// navigation.component.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-navigation',
  standalone: true,
  imports: [RouterLink, RouterLinkActive],
  template: `
    <nav>
      <a 
        routerLink="/home"
        routerLinkActive="active"
        [routerLinkActiveOptions]="{ exact: true }">
        Home
      </a>
      
      <a 
        routerLink="/products"
        routerLinkActive="active">
        Products
      </a>
      
      <a 
        routerLink="/about"
        routerLinkActive="active highlight"
        #aboutLink="routerLinkActive">
        About ({{ aboutLink.isActive ? 'Current' : 'Not Current' }})
      </a>
    </nav>
  `,
  styles: [`
    .active {
      font-weight: bold;
      color: #007bff;
    }
    .highlight {
      background-color: #fff3cd;
    }
  `]
})
export class NavigationComponent {}
```

### Multiple CSS Classes

```typescript
// sidebar.component.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-sidebar',
  standalone: true,
  imports: [RouterLink, RouterLinkActive],
  template: `
    <aside class="sidebar">
      <a 
        routerLink="/dashboard"
        routerLinkActive="active selected highlighted"
        [routerLinkActiveOptions]="{ exact: true }">
        Dashboard
      </a>
      
      <a 
        routerLink="/settings"
        [routerLinkActive]="['active', 'current-page']">
        Settings
      </a>
    </aside>
  `,
  styles: [`
    .active { color: blue; }
    .selected { border-left: 3px solid blue; }
    .highlighted { background: #f0f0f0; }
    .current-page { font-weight: bold; }
  `]
})
export class SidebarComponent {}
```

### RouterLinkActive with Parent/Child Routes

```typescript
// menu.component.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-menu',
  standalone: true,
  imports: [RouterLink, RouterLinkActive],
  template: `
    <nav>
      <!-- Exact match for root -->
      <a 
        routerLink="/"
        routerLinkActive="active"
        [routerLinkActiveOptions]="{ exact: true }">
        Home
      </a>
      
      <!-- Active for /products and /products/123 -->
      <a 
        routerLink="/products"
        routerLinkActive="active">
        Products
      </a>
      
      <!-- Nested menu -->
      <div class="submenu">
        <a 
          routerLink="/products/featured"
          routerLinkActive="active">
          Featured
        </a>
        <a 
          routerLink="/products/new"
          routerLinkActive="active">
          New Arrivals
        </a>
      </div>
    </nav>
  `
})
export class MenuComponent {}
```

### Programmatic Access to RouterLinkActive

```typescript
// dynamic-menu.component.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive } from '@angular/router';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-dynamic-menu',
  standalone: true,
  imports: [CommonModule, RouterLink, RouterLinkActive],
  template: `
    <nav>
      <a 
        *ngFor="let item of menuItems"
        [routerLink]="item.path"
        routerLinkActive="active"
        #rla="routerLinkActive"
        [attr.aria-current]="rla.isActive ? 'page' : null">
        {{ item.label }}
        <span *ngIf="rla.isActive" class="indicator">→</span>
      </a>
    </nav>
  `
})
export class DynamicMenuComponent {
  menuItems = [
    { path: '/home', label: 'Home' },
    { path: '/products', label: 'Products' },
    { path: '/contact', label: 'Contact' }
  ];
}
```

## Programmatic Navigation

### Basic Router Navigation

```typescript
// navigation.service.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class NavigationService {
  constructor(private router: Router) {}

  // Simple navigation
  goToHome(): void {
    this.router.navigate(['/home']);
  }

  // Navigation with parameters
  goToUser(userId: number): void {
    this.router.navigate(['/users', userId]);
  }

  // Navigation with query parameters
  goToProducts(category: string, sort: string): void {
    this.router.navigate(['/products'], {
      queryParams: { category, sort }
    });
  }

  // Navigation with fragment
  goToSection(section: string): void {
    this.router.navigate(['/docs'], {
      fragment: section
    });
  }
}
```

### NavigateByUrl vs Navigate

```typescript
// routing-examples.service.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class RoutingExamplesService {
  constructor(private router: Router) {}

  // navigate() - uses commands array
  navigateWithCommands(): void {
    // Good for programmatic routing with variables
    this.router.navigate(['/products', 123, 'details']);
  }

  // navigateByUrl() - uses absolute URL string
  navigateWithUrl(): void {
    // Good for string URLs
    this.router.navigateByUrl('/products/123/details');
  }

  // Difference with query params
  navigateWithQueryParams(): void {
    // With navigate()
    this.router.navigate(['/search'], {
      queryParams: { q: 'angular' }
    });

    // With navigateByUrl()
    this.router.navigateByUrl('/search?q=angular');
  }

  // navigateByUrl with UrlTree
  navigateWithUrlTree(): void {
    const urlTree = this.router.createUrlTree(['/products'], {
      queryParams: { category: 'electronics' }
    });
    this.router.navigateByUrl(urlTree);
  }
}
```

### Relative Navigation

```typescript
// product-detail.component.ts
import { Component } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/core';

@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div>
      <button (click)="goToEdit()">Edit</button>
      <button (click)="goToReviews()">Reviews</button>
      <button (click)="goBackToList()">Back to List</button>
      <button (click)="goToNextProduct()">Next Product</button>
    </div>
  `
})
export class ProductDetailComponent {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  // Navigate to ./edit (relative to current route)
  goToEdit(): void {
    this.router.navigate(['edit'], { relativeTo: this.route });
  }

  // Navigate to ./reviews
  goToReviews(): void {
    this.router.navigate(['reviews'], { relativeTo: this.route });
  }

  // Navigate to parent route
  goBackToList(): void {
    this.router.navigate(['..'], { relativeTo: this.route });
  }

  // Navigate to sibling route
  goToNextProduct(): void {
    this.router.navigate(['..', '124'], { relativeTo: this.route });
  }
}
```

### Navigation with State

```typescript
// navigation-with-state.service.ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';

interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({ providedIn: 'root' })
export class NavigationWithStateService {
  constructor(private router: Router) {}

  navigateToUserWithData(user: User): void {
    this.router.navigate(['/users', user.id], {
      state: { 
        user,
        source: 'search',
        timestamp: Date.now()
      }
    });
  }

  navigateWithComplexState(): void {
    this.router.navigate(['/checkout'], {
      state: {
        cartItems: [
          { id: 1, name: 'Product 1', price: 99.99 },
          { id: 2, name: 'Product 2', price: 149.99 }
        ],
        discountCode: 'SAVE20',
        previousPage: '/cart'
      }
    });
  }
}

// Receiving component
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-user-detail',
  standalone: true,
  template: `<div>{{ user | json }}</div>`
})
export class UserDetailComponent implements OnInit {
  user: User | undefined;

  constructor(private router: Router) {
    // Access state in constructor
    const navigation = this.router.getCurrentNavigation();
    if (navigation?.extras.state) {
      this.user = navigation.extras.state['user'];
    }
  }

  ngOnInit(): void {
    // State is only available during navigation
    // It won't persist on page refresh
    if (!this.user) {
      console.warn('No state data available - possibly page refresh');
      // Load data from service or route params instead
    }
  }
}
```

## Multiple Router Outlets

### Named Router Outlets

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    component: LayoutComponent,
    children: [
      { path: 'home', component: HomeComponent },
      { path: 'sidebar', component: SidebarComponent, outlet: 'sidebar' },
      { path: 'modal', component: ModalComponent, outlet: 'modal' }
    ]
  }
];

// layout.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-layout',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <div class="layout">
      <main>
        <router-outlet></router-outlet>
      </main>
      
      <aside>
        <router-outlet name="sidebar"></router-outlet>
      </aside>
      
      <div class="modal-container">
        <router-outlet name="modal"></router-outlet>
      </div>
    </div>
  `
})
export class LayoutComponent {}
```

### Navigating to Named Outlets

```typescript
// multi-outlet-navigation.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-multi-outlet-navigation',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div>
      <!-- Open sidebar -->
      <a [routerLink]="[{ outlets: { sidebar: ['sidebar'] } }]">
        Open Sidebar
      </a>
      
      <!-- Open modal -->
      <a [routerLink]="[{ outlets: { modal: ['modal'] } }]">
        Open Modal
      </a>
      
      <!-- Close sidebar -->
      <a [routerLink]="[{ outlets: { sidebar: null } }]">
        Close Sidebar
      </a>
      
      <!-- Multiple outlets at once -->
      <button (click)="openBoth()">Open Both</button>
      <button (click)="closeAll()">Close All</button>
    </div>
  `
})
export class MultiOutletNavigationComponent {
  constructor(private router: Router) {}

  openBoth(): void {
    this.router.navigate([{
      outlets: {
        primary: ['home'],
        sidebar: ['sidebar'],
        modal: ['modal']
      }
    }]);
  }

  closeAll(): void {
    this.router.navigate([{
      outlets: {
        sidebar: null,
        modal: null
      }
    }]);
  }
}
```

### Complex Named Outlet Example

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: '', component: DashboardHomeComponent },
      { path: 'stats', component: StatsComponent, outlet: 'panel' },
      { path: 'notifications', component: NotificationsComponent, outlet: 'panel' },
      { path: 'help', component: HelpComponent, outlet: 'sidebar' }
    ]
  }
];

// dashboard.component.ts
import { Component } from '@angular/core';
import { Router, RouterOutlet, RouterLink } from '@angular/router';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <div class="dashboard">
      <nav>
        <a [routerLink]="[{ outlets: { panel: ['stats'] } }]">Stats</a>
        <a [routerLink]="[{ outlets: { panel: ['notifications'] } }]">Notifications</a>
        <a [routerLink]="[{ outlets: { sidebar: ['help'] } }]">Help</a>
        <button (click)="clearPanels()">Clear</button>
      </nav>
      
      <main>
        <router-outlet></router-outlet>
      </main>
      
      <aside class="panel">
        <router-outlet name="panel"></router-outlet>
      </aside>
      
      <aside class="sidebar">
        <router-outlet name="sidebar"></router-outlet>
      </aside>
    </div>
  `
})
export class DashboardComponent {
  constructor(private router: Router) {}

  clearPanels(): void {
    this.router.navigate([{
      outlets: { panel: null, sidebar: null }
    }]);
  }
}
```

## Navigation Extras

### Complete Navigation Options

```typescript
// navigation-options.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationExtras } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class NavigationOptionsService {
  constructor(private router: Router) {}

  demonstrateAllOptions(): void {
    const extras: NavigationExtras = {
      // Query parameters
      queryParams: { search: 'angular', page: 1 },
      
      // How to handle existing query params
      queryParamsHandling: 'merge', // 'merge' | 'preserve' | ''
      
      // URL fragment (anchor)
      fragment: 'section-2',
      
      // Preserve fragment from current route
      preserveFragment: true,
      
      // Skip location change (don't update URL)
      skipLocationChange: false,
      
      // Replace current history state instead of pushing new one
      replaceUrl: false,
      
      // State to pass (not visible in URL)
      state: { fromDashboard: true },
      
      // For relative navigation
      relativeTo: null
    };

    this.router.navigate(['/products'], extras);
  }

  // Practical examples
  navigateWithoutHistoryEntry(path: string): void {
    this.router.navigate([path], { replaceUrl: true });
  }

  navigateWithoutUrlChange(path: string): void {
    // Useful for modals or overlays
    this.router.navigate([path], { skipLocationChange: true });
  }

  mergeQueryParams(newParams: any): void {
    this.router.navigate([], {
      queryParams: newParams,
      queryParamsHandling: 'merge'
    });
  }
}
```

### Preserving Query Parameters

```typescript
// query-param-preservation.component.ts
import { Component } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/core';

@Component({
  selector: 'app-query-param-preservation',
  standalone: true,
  template: `
    <div>
      <button (click)="navigatePreserving()">Navigate (Preserve)</button>
      <button (click)="navigateMerging()">Navigate (Merge)</button>
      <button (click)="navigateReplacing()">Navigate (Replace)</button>
    </div>
  `
})
export class QueryParamPreservationComponent {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}

  // Keep all existing query params
  navigatePreserving(): void {
    this.router.navigate(['/next-page'], {
      queryParamsHandling: 'preserve'
    });
  }

  // Merge new params with existing ones
  navigateMerging(): void {
    this.router.navigate(['/next-page'], {
      queryParams: { newParam: 'value' },
      queryParamsHandling: 'merge'
    });
  }

  // Replace all params (default behavior)
  navigateReplacing(): void {
    this.router.navigate(['/next-page'], {
      queryParams: { onlyThis: 'value' }
    });
  }
}
```

## Common Mistakes

### 1. Using href Instead of routerLink

```typescript
// WRONG - Causes full page reload
@Component({
  template: `<a href="/products">Products</a>`
})
export class WrongComponent {}

// CORRECT - SPA navigation
@Component({
  imports: [RouterLink],
  template: `<a routerLink="/products">Products</a>`
})
export class CorrectComponent {}
```

### 2. Forgetting relativeTo for Relative Navigation

```typescript
// WRONG - Might not work as expected
@Component({})
export class WrongComponent {
  constructor(private router: Router) {}
  
  goToEdit(): void {
    this.router.navigate(['edit']); // Relative to root!
  }
}

// CORRECT
@Component({})
export class CorrectComponent {
  constructor(
    private router: Router,
    private route: ActivatedRoute
  ) {}
  
  goToEdit(): void {
    this.router.navigate(['edit'], { relativeTo: this.route });
  }
}
```

### 3. Missing Exact Option for Root Routes

```typescript
// WRONG - Always active for all routes
@Component({
  template: `
    <a routerLink="/" routerLinkActive="active">Home</a>
  `
})
export class WrongComponent {}

// CORRECT
@Component({
  template: `
    <a 
      routerLink="/" 
      routerLinkActive="active"
      [routerLinkActiveOptions]="{ exact: true }">
      Home
    </a>
  `
})
export class CorrectComponent {}
```

### 4. Not Handling Navigation Failures

```typescript
// WRONG - Ignores navigation failures
@Component({})
export class WrongComponent {
  constructor(private router: Router) {}
  
  navigate(): void {
    this.router.navigate(['/protected']);
  }
}

// CORRECT
@Component({})
export class CorrectComponent {
  constructor(private router: Router) {}
  
  async navigate(): Promise<void> {
    try {
      const success = await this.router.navigate(['/protected']);
      if (!success) {
        console.warn('Navigation was prevented');
      }
    } catch (error) {
      console.error('Navigation error:', error);
    }
  }
}
```

## Best Practices

### 1. Use Type-Safe Navigation

```typescript
// routes.config.ts
export const APP_ROUTES = {
  home: () => ['/home'],
  userDetail: (id: number) => ['/users', id],
  productSearch: (category: string) => ['/products', { category }]
} as const;

// navigation.service.ts
@Injectable({ providedIn: 'root' })
export class NavigationService {
  constructor(private router: Router) {}

  goToUser(id: number): Promise<boolean> {
    return this.router.navigate(APP_ROUTES.userDetail(id));
  }
}
```

### 2. Centralize Navigation Logic

```typescript
// navigation.facade.ts
@Injectable({ providedIn: 'root' })
export class NavigationFacade {
  constructor(private router: Router) {}

  navigateToHome(): Promise<boolean> {
    return this.router.navigate(['/home']);
  }

  navigateToUserProfile(userId: number): Promise<boolean> {
    return this.router.navigate(['/users', userId]);
  }

  navigateBack(): void {
    window.history.back();
  }

  async navigateWithConfirmation(
    path: string[],
    message: string
  ): Promise<boolean> {
    if (confirm(message)) {
      return this.router.navigate(path);
    }
    return false;
  }
}
```

### 3. Scroll Management

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withInMemoryScrolling } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withInMemoryScrolling({
        scrollPositionRestoration: 'enabled',
        anchorScrolling: 'enabled'
      })
    )
  ]
};

// Custom scroll behavior
@Component({})
export class AppComponent {
  constructor(private router: Router) {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationEnd) {
        window.scrollTo(0, 0);
      }
    });
  }
}
```

### 4. Loading Indicators

```typescript
// loading.service.ts
@Injectable({ providedIn: 'root' })
export class LoadingService {
  private loadingSubject = new BehaviorSubject<boolean>(false);
  loading$ = this.loadingSubject.asObservable();

  constructor(private router: Router) {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.loadingSubject.next(true);
      } else if (
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
      ) {
        this.loadingSubject.next(false);
      }
    });
  }
}
```

## Interview Questions

### Q1: What's the difference between routerLink and href?

**Answer:** `routerLink` is an Angular directive that performs SPA navigation without page reload, maintains application state, and works with Angular routing features. `href` causes a full page reload, losing all application state and requiring the app to reinitialize.

### Q2: How do you navigate programmatically in Angular?

**Answer:** Use the Router service's `navigate()` or `navigateByUrl()` methods:

```typescript
// Array syntax
this.router.navigate(['/users', userId]);

// String URL
this.router.navigateByUrl('/users/123');
```

### Q3: What is the purpose of routerLinkActive?

**Answer:** `routerLinkActive` applies CSS classes to elements when the associated route is active, useful for highlighting current navigation items. It supports exact matching and multiple CSS classes.

### Q4: How do you pass data between routes?

**Answer:** Multiple ways:

1. Route parameters: `/users/:id`
2. Query parameters: `/search?q=angular`
3. Navigation state: `{ state: { data } }`
4. Services with shared state
5. Resolvers for pre-fetching

### Q5: What are named router outlets?

**Answer:** Named outlets allow multiple router-outlet instances in a component, enabling parallel views like sidebars or modals. Navigate using: `[routerLink]="[{ outlets: { sidebar: ['content'] } }]"`

### Q6: How do you handle relative navigation?

**Answer:** Use the `relativeTo` option with ActivatedRoute:

```typescript
this.router.navigate(['../sibling'], { 
  relativeTo: this.route 
});
```

### Q7: What navigation extras are available?

**Answer:** `queryParams`, `queryParamsHandling`, `fragment`, `preserveFragment`, `skipLocationChange`, `replaceUrl`, `state`, and `relativeTo`.

### Q8: How do you prevent navigation?

**Answer:** Use route guards (canActivate, canDeactivate) or check navigation result:

```typescript
const success = await this.router.navigate(['/path']);
if (!success) { /* handle */ }
```

## Key Takeaways

1. **Router Outlet** is the placeholder for routed components
2. **RouterLink** provides declarative navigation without page reloads
3. **RouterLinkActive** adds CSS classes to active route links
4. **Programmatic navigation** uses Router service methods
5. **Named outlets** enable multiple parallel views
6. **Navigation extras** provide fine-grained control
7. **Relative navigation** requires `relativeTo` option
8. **State passing** works only during navigation
9. **Always use routerLink** over href for SPA behavior
10. **Handle navigation failures** for robust applications

## Resources

- [Angular Router Documentation](https://angular.io/guide/router)
- [RouterLink API Reference](https://angular.io/api/router/RouterLink)
- [RouterLinkActive API](https://angular.io/api/router/RouterLinkActive)
- [Router API Reference](https://angular.io/api/router/Router)
- [Navigation Extras](https://angular.io/api/router/NavigationExtras)
- [Router Events Guide](https://angular.io/guide/router#router-events)
- [Angular University Router Course](https://angular-university.io/)
