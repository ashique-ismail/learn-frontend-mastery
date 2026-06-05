# Route Configuration in Angular

## The Idea

**In plain English:** Route configuration is a set of rules that tells your app which "page" (component) to show when someone visits a specific web address. Think of it like a directory that connects each URL path to a piece of your app.

**Real-world analogy:** Imagine a large shopping mall with a directory board at the entrance. When you look up "Food Court," the board points you to Level 3, Wing B. Route configuration works the same way.

- The directory board = the routes array (the list of all path-to-component rules)
- Each store listing on the board = a single route object (e.g., `{ path: 'about', component: AboutComponent }`)
- The floor/wing location = the component that gets displayed when the path is matched

---

## Overview

Angular's router enables navigation between different views in single-page applications. Route configuration defines the mapping between URLs and components, supports lazy loading for performance, handles nested routes, and provides features like route parameters, guards, and resolvers.

This guide covers route setup, RouterModule configuration, lazy loading strategies, nested routes, and advanced routing patterns.

## Table of Contents

1. [Basic Route Configuration](#basic-route-configuration)
2. [RouterModule Setup](#routermodule-setup)
3. [Lazy Loading Routes](#lazy-loading-routes)
4. [Child and Nested Routes](#child-and-nested-routes)
5. [Route Redirects](#route-redirects)
6. [Wildcard Routes](#wildcard-routes)
7. [Route Data and Title](#route-data-and-title)
8. [Multi-Module Routing](#multi-module-routing)
9. [Route Configuration Options](#route-configuration-options)
10. [Testing Routes](#testing-routes)

## Basic Route Configuration

Setting up fundamental routes.

### Simple Routes

```typescript
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';
import { ContactComponent } from './contact/contact.component';

// Define routes
const routes: Routes = [
  { path: '', component: HomeComponent },           // Default route
  { path: 'about', component: AboutComponent },     // /about
  { path: 'contact', component: ContactComponent }  // /contact
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Route with Path Parameters

```typescript
import { UserComponent } from './user/user.component';
import { UserDetailComponent } from './user-detail/user-detail.component';

const routes: Routes = [
  { path: 'users', component: UserComponent },
  { path: 'users/:id', component: UserDetailComponent },       // /users/123
  { path: 'users/:id/edit', component: UserEditComponent },    // /users/123/edit
  { path: 'posts/:postId/comments/:commentId', component: CommentDetailComponent }
];
```

### Routes with Multiple Parameters

```typescript
const routes: Routes = [
  // Multiple parameters
  { path: 'shop/:category/:productId', component: ProductComponent },
  // /shop/electronics/12345
  
  // Optional parameters via query strings
  { path: 'search', component: SearchComponent },
  // /search?q=angular&page=2
  
  // Fragment
  { path: 'docs', component: DocsComponent }
  // /docs#section-3
];
```

## RouterModule Setup

Configuring the router module.

### App-Level RouterModule (forRoot)

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes, ExtraOptions } from '@angular/router';

const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent }
];

// Router configuration options
const routerOptions: ExtraOptions = {
  // Enable hash-based routing (default is path-based)
  useHash: false,
  
  // Enable debug tracing (development only)
  enableTracing: false,
  
  // Scroll behavior
  scrollPositionRestoration: 'enabled', // or 'disabled' or 'top'
  anchorScrolling: 'enabled',
  
  // Initial navigation
  initialNavigation: 'enabledBlocking', // Wait for router before app render
  
  // URL update strategy
  urlUpdateStrategy: 'deferred', // or 'eager'
  
  // Cancel pending navigations
  canceledNavigationResolution: 'replace'
};

@NgModule({
  imports: [RouterModule.forRoot(routes, routerOptions)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Feature Module RouterModule (forChild)

```typescript
// feature-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/forms';

const routes: Routes = [
  { path: 'dashboard', component: DashboardComponent },
  { path: 'settings', component: SettingsComponent }
];

@NgModule({
  imports: [RouterModule.forChild(routes)], // Use forChild for feature modules
  exports: [RouterModule]
})
export class FeatureRoutingModule { }
```

### Standalone Components Routing

```typescript
// main.ts (Angular 14+)
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes)
  ]
});

// app.routes.ts
import { Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent }
];
```

## Lazy Loading Routes

Loading feature modules on-demand.

### Basic Lazy Loading

```typescript
const routes: Routes = [
  { path: '', component: HomeComponent },
  
  // Lazy load admin module
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  
  // Lazy load products module
  {
    path: 'products',
    loadChildren: () => import('./products/products.module').then(m => m.ProductsModule)
  }
];

// admin/admin-routing.module.ts
const adminRoutes: Routes = [
  { path: '', component: AdminDashboardComponent },      // /admin
  { path: 'users', component: AdminUsersComponent },     // /admin/users
  { path: 'settings', component: AdminSettingsComponent } // /admin/settings
];

@NgModule({
  imports: [RouterModule.forChild(adminRoutes)],
  exports: [RouterModule]
})
export class AdminRoutingModule { }
```

### Lazy Load with Standalone Components

```typescript
// Angular 14+ standalone lazy loading
const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent)
  },
  
  // Lazy load with child routes
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes').then(m => m.DASHBOARD_ROUTES)
  }
];

// dashboard/dashboard.routes.ts
export const DASHBOARD_ROUTES: Routes = [
  { path: '', component: DashboardComponent },
  { path: 'analytics', component: AnalyticsComponent }
];
```

### Preloading Strategy

```typescript
import { PreloadAllModules, PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

// Use built-in preloading
@NgModule({
  imports: [RouterModule.forRoot(routes, {
    preloadingStrategy: PreloadAllModules // Preload all lazy modules
  })],
  exports: [RouterModule]
})
export class AppRoutingModule { }

// Custom preloading strategy
export class CustomPreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Only preload routes with data.preload = true
    if (route.data?.['preload']) {
      console.log('Preloading:', route.path);
      return load();
    }
    return of(null);
  }
}

// Usage
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preload: true } // This module will be preloaded
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.module').then(m => m.ReportsModule)
    // This module won't be preloaded (no preload flag)
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes, {
    preloadingStrategy: CustomPreloadingStrategy
  })],
  providers: [CustomPreloadingStrategy],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

## Child and Nested Routes

Creating hierarchical route structures.

### Basic Child Routes

```typescript
const routes: Routes = [
  {
    path: 'products',
    component: ProductsComponent,
    children: [
      { path: '', component: ProductListComponent },        // /products
      { path: 'new', component: ProductCreateComponent },   // /products/new
      { path: ':id', component: ProductDetailComponent },   // /products/123
      { path: ':id/edit', component: ProductEditComponent } // /products/123/edit
    ]
  }
];

// products.component.ts
@Component({
  selector: 'app-products',
  template: `
    <div class="products-layout">
      <nav>
        <a routerLink="/products">List</a>
        <a routerLink="/products/new">Create New</a>
      </nav>
      
      <!-- Child routes render here -->
      <router-outlet></router-outlet>
    </div>
  `
})
export class ProductsComponent { }
```

### Multi-Level Nesting

```typescript
const routes: Routes = [
  {
    path: 'admin',
    component: AdminLayoutComponent,
    children: [
      {
        path: 'users',
        component: UsersComponent,
        children: [
          { path: '', component: UserListComponent },          // /admin/users
          { path: ':userId', component: UserDetailComponent,   // /admin/users/123
            children: [
              { path: 'profile', component: UserProfileComponent },  // /admin/users/123/profile
              { path: 'settings', component: UserSettingsComponent } // /admin/users/123/settings
            ]
          }
        ]
      }
    ]
  }
];

// admin-layout.component.ts
@Component({
  selector: 'app-admin-layout',
  template: `
    <div class="admin-layout">
      <app-admin-sidebar></app-admin-sidebar>
      <main>
        <router-outlet></router-outlet> <!-- Level 1 -->
      </main>
    </div>
  `
})
export class AdminLayoutComponent { }

// users.component.ts
@Component({
  selector: 'app-users',
  template: `
    <div class="users-section">
      <h2>Users</h2>
      <router-outlet></router-outlet> <!-- Level 2 -->
    </div>
  `
})
export class UsersComponent { }

// user-detail.component.ts
@Component({
  selector: 'app-user-detail',
  template: `
    <div class="user-detail">
      <nav>
        <a [routerLink]="['profile']">Profile</a>
        <a [routerLink]="['settings']">Settings</a>
      </nav>
      <router-outlet></router-outlet> <!-- Level 3 -->
    </div>
  `
})
export class UserDetailComponent { }
```

### Named Outlets

```typescript
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: '', component: DashboardHomeComponent },
      {
        path: 'inbox',
        component: InboxComponent,
        outlet: 'sidebar' // Named outlet
      },
      {
        path: 'help',
        component: HelpComponent,
        outlet: 'sidebar'
      }
    ]
  }
];

// dashboard.component.ts
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <main>
        <router-outlet></router-outlet> <!-- Primary outlet -->
      </main>
      <aside>
        <router-outlet name="sidebar"></router-outlet> <!-- Named outlet -->
      </aside>
    </div>
  `
})
export class DashboardComponent { }

// Navigate to named outlet
// URL: /dashboard/(sidebar:inbox)
this.router.navigate(['/dashboard', { outlets: { sidebar: ['inbox'] } }]);
```

## Route Redirects

Redirecting between routes.

### Basic Redirects

```typescript
const routes: Routes = [
  // Redirect empty path to home
  { path: '', redirectTo: '/home', pathMatch: 'full' },
  
  // Redirect old URLs to new URLs
  { path: 'old-products', redirectTo: '/products', pathMatch: 'full' },
  
  // Redirect with path segment preservation
  { path: 'old-user/:id', redirectTo: '/users/:id', pathMatch: 'full' },
  
  { path: 'home', component: HomeComponent },
  { path: 'products', component: ProductsComponent },
  { path: 'users/:id', component: UserComponent }
];
```

### PathMatch Explained

```typescript
const routes: Routes = [
  // pathMatch: 'full' - Must match entire URL
  { path: '', redirectTo: '/home', pathMatch: 'full' },
  // Only redirects if URL is exactly ""
  // Does NOT redirect for "/products" or "/anything"
  
  // pathMatch: 'prefix' - Matches if URL starts with path
  { path: '', redirectTo: '/home', pathMatch: 'prefix' },
  // Redirects for "", "/products", "/anything"
  // Usually creates infinite loop - use 'full' for empty path
  
  // Example: prefix matching for non-empty paths
  { path: 'api', redirectTo: '/external-api', pathMatch: 'prefix' },
  // Matches: /api, /api/users, /api/anything
];
```

### Conditional Redirects

```typescript
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [() => {
      const authService = inject(AuthService);
      const router = inject(Router);
      
      if (!authService.isLoggedIn()) {
        router.navigate(['/login']);
        return false;
      }
      return true;
    }]
  },
  
  {
    path: 'login',
    component: LoginComponent,
    canActivate: [() => {
      const authService = inject(AuthService);
      const router = inject(Router);
      
      // Redirect to dashboard if already logged in
      if (authService.isLoggedIn()) {
        router.navigate(['/dashboard']);
        return false;
      }
      return true;
    }]
  }
];
```

## Wildcard Routes

Handling unknown routes (404 pages).

### 404 Page

```typescript
import { PageNotFoundComponent } from './page-not-found/page-not-found.component';

const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'products', component: ProductsComponent },
  
  // Wildcard route MUST be last
  { path: '**', component: PageNotFoundComponent }
];

// page-not-found.component.ts
@Component({
  selector: 'app-page-not-found',
  template: `
    <div class="not-found">
      <h1>404 - Page Not Found</h1>
      <p>The page you're looking for doesn't exist.</p>
      <a routerLink="/">Go to Home</a>
    </div>
  `
})
export class PageNotFoundComponent { }
```

### Redirect Wildcard to Home

```typescript
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  
  // Redirect any unknown route to home
  { path: '**', redirectTo: '' }
];
```

### Feature-Specific 404

```typescript
const routes: Routes = [
  { path: '', component: HomeComponent },
  
  // Admin module with its own 404
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  
  // App-level 404
  { path: '**', component: PageNotFoundComponent }
];

// admin-routing.module.ts
const adminRoutes: Routes = [
  { path: '', component: AdminDashboardComponent },
  { path: 'users', component: UsersComponent },
  
  // Admin-specific 404
  { path: '**', component: AdminNotFoundComponent }
];
```

## Route Data and Title

Passing data to routes and setting page titles.

### Static Route Data

```typescript
const routes: Routes = [
  {
    path: 'about',
    component: AboutComponent,
    data: { 
      title: 'About Us',
      breadcrumb: 'About',
      requiresAuth: false,
      roles: ['user', 'admin']
    }
  },
  {
    path: 'admin',
    component: AdminComponent,
    data: { 
      title: 'Admin Panel',
      requiresAuth: true,
      roles: ['admin']
    }
  }
];

// Access route data in component
@Component({
  selector: 'app-about',
  template: `<h1>{{ pageTitle }}</h1>`
})
export class AboutComponent {
  pageTitle: string = '';

  constructor(private route: ActivatedRoute) {
    this.route.data.subscribe(data => {
      this.pageTitle = data['title'];
    });
  }
}
```

### Page Titles (Angular 14+)

```typescript
const routes: Routes = [
  { 
    path: '', 
    component: HomeComponent,
    title: 'Home' // Sets document title
  },
  { 
    path: 'about', 
    component: AboutComponent,
    title: 'About Us'
  },
  { 
    path: 'products/:id', 
    component: ProductDetailComponent,
    title: ProductTitleResolver // Dynamic title from resolver
  }
];

// Dynamic title resolver
@Injectable({ providedIn: 'root' })
export class ProductTitleResolver {
  constructor(private productService: ProductService) {}

  resolve(route: ActivatedRouteSnapshot): Observable<string> {
    const id = route.paramMap.get('id')!;
    return this.productService.getProduct(id).pipe(
      map(product => `${product.name} - Products`)
    );
  }
}
```

### Custom Title Strategy

```typescript
import { Injectable } from '@angular/core';
import { Title } from '@angular/platform-browser';
import { RouterStateSnapshot, TitleStrategy } from '@angular/router';

@Injectable()
export class CustomTitleStrategy extends TitleStrategy {
  constructor(private title: Title) {
    super();
  }

  override updateTitle(snapshot: RouterStateSnapshot): void {
    const title = this.buildTitle(snapshot);
    if (title) {
      this.title.setTitle(`MyApp - ${title}`);
    } else {
      this.title.setTitle('MyApp');
    }
  }
}

// Provide in app.config.ts or module
providers: [
  { provide: TitleStrategy, useClass: CustomTitleStrategy }
]
```

## Multi-Module Routing

Organizing routes across multiple modules.

### Feature Module Structure

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'auth',
    loadChildren: () => import('./auth/auth.module').then(m => m.AuthModule)
  },
  {
    path: 'products',
    loadChildren: () => import('./products/products.module').then(m => m.ProductsModule)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  }
];

// auth/auth-routing.module.ts
const authRoutes: Routes = [
  { path: 'login', component: LoginComponent },       // /auth/login
  { path: 'register', component: RegisterComponent }, // /auth/register
  { path: 'forgot-password', component: ForgotPasswordComponent }
];

// products/products-routing.module.ts
const productRoutes: Routes = [
  { path: '', component: ProductListComponent },      // /products
  { path: ':id', component: ProductDetailComponent }, // /products/123
  { path: ':id/edit', component: ProductEditComponent }
];

// admin/admin-routing.module.ts
const adminRoutes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  { path: 'dashboard', component: AdminDashboardComponent }, // /admin/dashboard
  { path: 'users', component: AdminUsersComponent },
  { path: 'settings', component: AdminSettingsComponent }
];
```

## Route Configuration Options

Advanced routing configuration.

### Router Options

```typescript
import { ExtraOptions } from '@angular/router';

const routerConfig: ExtraOptions = {
  // Use hash-based routing (#/path)
  useHash: false,
  
  // Enable router debugging
  enableTracing: false, // Set to true for debugging
  
  // Scroll position restoration
  scrollPositionRestoration: 'enabled', // 'disabled' | 'top' | 'enabled'
  
  // Scroll to anchor
  anchorScrolling: 'enabled', // 'disabled' | 'enabled'
  
  // Scroll offset
  scrollOffset: [0, 64], // [x, y] offset in pixels
  
  // How the router processes navigation
  onSameUrlNavigation: 'reload', // 'reload' | 'ignore'
  
  // URL update strategy
  urlUpdateStrategy: 'deferred', // 'deferred' | 'eager'
  
  // Resolve navigation promises
  paramsInheritanceStrategy: 'always', // 'emptyOnly' | 'always'
  
  // Malformed URI error handler
  malformedUriErrorHandler: (error: URIError, urlSerializer: UrlSerializer, url: string) => {
    return urlSerializer.parse('/404');
  },
  
  // Cancel pending navigation
  canceledNavigationResolution: 'replace' // 'replace' | 'computed'
};

@NgModule({
  imports: [RouterModule.forRoot(routes, routerConfig)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

## Testing Routes

Unit testing route configuration.

### Testing Router Configuration

```typescript
import { TestBed } from '@angular/core/testing';
import { Router } from '@angular/router';
import { Location } from '@angular/common';
import { routes } from './app-routing.module';

describe('AppRoutingModule', () => {
  let router: Router;
  let location: Location;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [RouterTestingModule.withRoutes(routes)]
    });

    router = TestBed.inject(Router);
    location = TestBed.inject(Location);
  });

  it('should navigate to home', async () => {
    await router.navigate(['']);
    expect(location.path()).toBe('/');
  });

  it('should navigate to about', async () => {
    await router.navigate(['/about']);
    expect(location.path()).toBe('/about');
  });

  it('should redirect unknown routes', async () => {
    await router.navigate(['/unknown']);
    expect(location.path()).toBe('/404'); // or wherever wildcard redirects
  });

  it('should navigate with parameters', async () => {
    await router.navigate(['/users', '123']);
    expect(location.path()).toBe('/users/123');
  });
});
```

## Common Mistakes

### 1. Wrong pathMatch for Empty Path Redirect

```typescript
// WRONG: Causes infinite redirect loop
{ path: '', redirectTo: '/home', pathMatch: 'prefix' }

// CORRECT: Use 'full' for empty path
{ path: '', redirectTo: '/home', pathMatch: 'full' }
```

### 2. Wildcard Route Not Last

```typescript
// WRONG: Wildcard catches all routes, nothing after it matches
const routes: Routes = [
  { path: '**', component: NotFoundComponent },
  { path: 'about', component: AboutComponent } // Never reached!
];

// CORRECT: Wildcard must be last
const routes: Routes = [
  { path: 'about', component: AboutComponent },
  { path: '**', component: NotFoundComponent }
];
```

### 3. Missing forChild in Feature Modules

```typescript
// WRONG: Using forRoot in feature module
@NgModule({
  imports: [RouterModule.forRoot(featureRoutes)] // DON'T DO THIS
})
export class FeatureRoutingModule { }

// CORRECT: Use forChild for feature modules
@NgModule({
  imports: [RouterModule.forChild(featureRoutes)]
})
export class FeatureRoutingModule { }
```

## Best Practices

1. **Use path-based routing** - Avoid hash-based unless necessary
2. **Organize routes hierarchically** - Mirror application structure
3. **Lazy load feature modules** - Improve initial load time
4. **Use forChild in feature modules** - Only forRoot in AppModule
5. **Put wildcard route last** - Must be final route definition
6. **Use pathMatch: 'full' for empty paths** - Avoid redirect loops
7. **Define route data for metadata** - Breadcrumbs, titles, permissions
8. **Use route guards** - Protect routes based on authentication/authorization
9. **Implement custom preloading** - Balance performance and UX
10. **Test route configuration** - Verify navigation works correctly

## Interview Questions

### Q1: What's the difference between forRoot and forChild?

**Answer:** `forRoot` configures router at application level and should only be called once in AppModule. `forChild` configures routes for feature modules and can be called multiple times.

### Q2: What is pathMatch and when do you use 'full' vs 'prefix'?

**Answer:** `pathMatch` determines how the router matches URLs. 'full' requires exact match (use for empty path redirects), 'prefix' matches if URL starts with the path (default, but causes issues with empty path).

### Q3: How does lazy loading improve performance?

**Answer:** Lazy loading splits the application into chunks that load on-demand. Only code for visited routes is downloaded, reducing initial bundle size and improving first load time.

### Q4: Why must the wildcard route be last?

**Answer:** Routes are matched in order. Since `**` matches any path, placing it first would catch all routes and prevent any subsequent routes from matching.

### Q5: What's the purpose of route data?

**Answer:** Route data passes static configuration to route components (titles, breadcrumbs, permissions, etc.). It's accessible via ActivatedRoute and useful for metadata-driven features.

## Key Takeaways

1. **forRoot for app-level, forChild for features** - Proper module configuration
2. **Lazy load feature modules** - Better performance
3. **Use pathMatch: 'full' for empty redirects** - Prevent loops
4. **Wildcard route must be last** - Catches unmatched routes
5. **Child routes create hierarchies** - Nested layouts and navigation
6. **Route data passes metadata** - Titles, breadcrumbs, permissions
7. **Custom preloading balances performance** - Load important features proactively
8. **Named outlets support multiple views** - Sidebars, modals, etc.
9. **Test route configuration** - Ensure navigation works correctly
10. **Configure router options for UX** - Scroll behavior, URL updates, etc.

## Resources

- [Angular Router Guide](https://angular.io/guide/router)
- [RouterModule API](https://angular.io/api/router/RouterModule)
- [Route Configuration](https://angular.io/api/router/Routes)
- [Lazy Loading Guide](https://angular.io/guide/lazy-loading-ngmodules)
- [Preloading Strategies](https://angular.io/guide/router#preloading)
