# Angular Routing Interview Questions

## The Idea

**In plain English:** Angular Routing is the system that decides which "page" (called a view or component) to show on screen when you type a different address in the browser, or click a link — all without actually loading a brand new webpage from the server. Think of a "component" as a self-contained chunk of content (like a homepage or a profile page), and a "route" as the rule that says "when the address bar shows /profile, show the ProfileComponent."

**Real-world analogy:** Imagine a large hospital building where every department (Emergency, Radiology, Cafeteria) is already inside the building. When you arrive, a receptionist at the front desk looks at your destination slip and physically walks you to the right department — no leaving the building, no new trip required.
- The receptionist = the Angular Router (reads the URL and decides where to send you)
- The destination slip (e.g., "Radiology") = the URL path (e.g., /radiology)
- Each hospital department = a component (the actual page/view the user sees)

---

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

Angular Router enables navigation between views, managing application state through URLs, and providing advanced features like lazy loading, route guards, and resolvers.

### Core Features

- Route configuration and matching
- Navigation and URL manipulation
- Route guards (canActivate, canDeactivate, etc.)
- Lazy loading modules
- Route parameters and query parameters
- Child routes and nested routing
- Preloading strategies

## Common Interview Questions

### Q1: Explain Angular Router configuration and the different types of routes.

**What the interviewer wants to know:**
- Understanding of route configuration
- Knowledge of routing patterns
- Ability to structure navigation

**Strong Answer:**

Angular Router maps URLs to components through route configuration. Understanding different route types is essential for building scalable applications.

**Basic Route Configuration:**

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';
import { ContactComponent } from './contact/contact.component';
import { NotFoundComponent } from './not-found/not-found.component';

const routes: Routes = [
  // Basic route
  { path: '', component: HomeComponent },
  
  // Static route
  { path: 'about', component: AboutComponent },
  { path: 'contact', component: ContactComponent },
  
  // Wildcard route (catch-all, must be last)
  { path: '**', component: NotFoundComponent }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

**Parameterized Routes:**

```typescript
// Route with parameters
const routes: Routes = [
  // Single parameter
  { path: 'user/:id', component: UserDetailComponent },
  
  // Multiple parameters
  { path: 'product/:category/:id', component: ProductDetailComponent },
  
  // Optional parameter with query params
  { path: 'search', component: SearchComponent }
];

// Accessing parameters in component
@Component({
  selector: 'app-user-detail'
})
export class UserDetailComponent implements OnInit {
  userId: string;
  
  constructor(
    private route: ActivatedRoute,
    private router: Router
  ) {}
  
  ngOnInit(): void {
    // Method 1: Snapshot (one-time read)
    this.userId = this.route.snapshot.paramMap.get('id')!;
    
    // Method 2: Observable (reactive to changes)
    this.route.paramMap.subscribe(params => {
      this.userId = params.get('id')!;
      this.loadUser(this.userId);
    });
    
    // Query parameters
    this.route.queryParams.subscribe(params => {
      const tab = params['tab'];
      const sort = params['sort'];
      console.log('Tab:', tab, 'Sort:', sort);
    });
  }
  
  loadUser(id: string): void {
    // Load user data
  }
}
```

**Child Routes:**

```typescript
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: '', redirectTo: 'overview', pathMatch: 'full' },
      { path: 'overview', component: OverviewComponent },
      { path: 'analytics', component: AnalyticsComponent },
      { path: 'reports', component: ReportsComponent }
    ]
  }
];

// Dashboard component template
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <nav>
        <a routerLink="overview" routerLinkActive="active">Overview</a>
        <a routerLink="analytics" routerLinkActive="active">Analytics</a>
        <a routerLink="reports" routerLinkActive="active">Reports</a>
      </nav>
      
      <!-- Child routes render here -->
      <router-outlet></router-outlet>
    </div>
  `
})
export class DashboardComponent {}
```

**Lazy Loaded Routes:**

```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  {
    path: 'shop',
    loadChildren: () => import('./shop/shop.module').then(m => m.ShopModule)
  }
];

// Admin module routing
@NgModule({
  imports: [RouterModule.forChild([
    { path: '', component: AdminDashboardComponent },
    { path: 'users', component: UsersComponent },
    { path: 'settings', component: SettingsComponent }
  ])]
})
export class AdminRoutingModule {}
```

**Named Outlets:**

```typescript
const routes: Routes = [
  {
    path: 'home',
    component: HomeComponent,
    children: [
      { path: 'messages', component: MessagesComponent, outlet: 'sidebar' },
      { path: 'notifications', component: NotificationsComponent, outlet: 'sidebar' }
    ]
  }
];

// Template with named outlet
@Component({
  template: `
    <div class="layout">
      <main>
        <router-outlet></router-outlet>
      </main>
      <aside>
        <router-outlet name="sidebar"></router-outlet>
      </aside>
    </div>
  `
})
export class LayoutComponent {}

// Navigate to named outlet
this.router.navigate([
  { outlets: { sidebar: ['messages'] } }
]);
```

**Route Data and Resolve:**

```typescript
const routes: Routes = [
  {
    path: 'product/:id',
    component: ProductDetailComponent,
    data: { 
      title: 'Product Details',
      breadcrumb: 'Product'
    },
    resolve: {
      product: ProductResolver
    }
  }
];

// Resolver
@Injectable({ providedIn: 'root' })
export class ProductResolver implements Resolve<Product> {
  constructor(private productService: ProductService) {}
  
  resolve(route: ActivatedRouteSnapshot): Observable<Product> {
    const id = route.paramMap.get('id')!;
    return this.productService.getProduct(id);
  }
}

// Component
@Component({})
export class ProductDetailComponent implements OnInit {
  product: Product;
  
  ngOnInit(): void {
    // Data from route config
    const title = this.route.snapshot.data['title'];
    
    // Resolved data
    this.route.data.subscribe(data => {
      this.product = data['product'];
    });
  }
}
```

**Redirects:**

```typescript
const routes: Routes = [
  // Simple redirect
  { path: '', redirectTo: '/home', pathMatch: 'full' },
  
  // Redirect old routes
  { path: 'old-about', redirectTo: '/about' },
  
  // Conditional redirect (use guard instead)
  { 
    path: 'admin',
    canActivate: [AuthGuard],
    component: AdminComponent
  }
];
```

**Navigation Methods:**

```typescript
export class NavigationExamplesComponent {
  constructor(private router: Router) {}
  
  // Method 1: router.navigate with array
  navigateToUser(userId: number): void {
    this.router.navigate(['/user', userId]);
    // Navigates to: /user/123
  }
  
  // Method 2: router.navigateByUrl with string
  navigateByUrl(): void {
    this.router.navigateByUrl('/about');
  }
  
  // Method 3: With query parameters
  navigateWithQueryParams(): void {
    this.router.navigate(['/search'], {
      queryParams: { q: 'angular', page: 1 },
      queryParamsHandling: 'merge' // preserve existing params
    });
    // Navigates to: /search?q=angular&page=1
  }
  
  // Method 4: With fragment (anchor)
  navigateWithFragment(): void {
    this.router.navigate(['/docs'], {
      fragment: 'routing'
    });
    // Navigates to: /docs#routing
  }
  
  // Method 5: Relative navigation
  navigateRelative(): void {
    this.router.navigate(['../sibling'], {
      relativeTo: this.route
    });
  }
  
  // Method 6: With state (doesn't appear in URL)
  navigateWithState(): void {
    this.router.navigate(['/result'], {
      state: { data: { id: 123, name: 'Test' } }
    });
  }
  
  // Access state in target component
  ngOnInit(): void {
    const navigation = this.router.getCurrentNavigation();
    const state = navigation?.extras.state;
    console.log(state); // { data: { id: 123, name: 'Test' } }
  }
}
```

**Follow-up Questions:**

1. **What's the difference between routerLink and href?**
   - routerLink: Angular navigation, SPA behavior, no page reload
   - href: Full page reload, loses application state
   - routerLink enables client-side routing

2. **When would you use pathMatch: 'full' vs 'prefix'?**
   - 'full': Match entire URL (used with redirects)
   - 'prefix': Match URL beginning (default)
   - Important for empty path redirects

3. **How do you programmatically check the current route?**
   ```typescript
   // Method 1: Router.url
   const currentUrl = this.router.url;
   
   // Method 2: ActivatedRoute
   this.route.url.subscribe(segments => {
     console.log(segments);
   });
   
   // Method 3: Router events
   this.router.events.pipe(
     filter(event => event instanceof NavigationEnd)
   ).subscribe((event: NavigationEnd) => {
     console.log('Current URL:', event.url);
   });
   ```

---

### Q2: Explain route guards and provide examples of when to use each type.

**What the interviewer wants to know:**
- Understanding of guard types and purposes
- Ability to implement security and navigation control
- Knowledge of guard execution order

**Strong Answer:**

Route guards control navigation to and from routes. Angular provides several guard types, each serving specific purposes in controlling user access and navigation flow.

**Guard Types Overview:**

```typescript
interface CanActivate {
  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot):
    Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree;
}

interface CanActivateChild {
  canActivateChild(childRoute: ActivatedRouteSnapshot, state: RouterStateSnapshot):
    Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree;
}

interface CanDeactivate<T> {
  canDeactivate(component: T, currentRoute: ActivatedRouteSnapshot, 
                currentState: RouterStateSnapshot, nextState?: RouterStateSnapshot):
    Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree;
}

interface CanLoad {
  canLoad(route: Route, segments: UrlSegment[]):
    Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree;
}

interface Resolve<T> {
  resolve(route: ActivatedRouteSnapshot, state: RouterStateSnapshot):
    Observable<T> | Promise<T> | T;
}
```

**1. CanActivate - Protect Routes:**

```typescript
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> {
    return this.authService.isAuthenticated$.pipe(
      map(isAuthenticated => {
        if (isAuthenticated) {
          return true;
        }
        
        // Redirect to login with return URL
        return this.router.createUrlTree(['/login'], {
          queryParams: { returnUrl: state.url }
        });
      })
    );
  }
}

// Route configuration
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [AuthGuard]
  }
];

// Login component - redirect after successful login
@Component({})
export class LoginComponent {
  login(): void {
    this.authService.login(this.credentials).subscribe(() => {
      const returnUrl = this.route.snapshot.queryParams['returnUrl'] || '/';
      this.router.navigate([returnUrl]);
    });
  }
}
```

**2. Role-Based CanActivate:**

```typescript
@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> {
    const requiredRoles = route.data['roles'] as string[];
    
    return this.authService.currentUser$.pipe(
      map(user => {
        if (!user) {
          return this.router.createUrlTree(['/login']);
        }
        
        const hasRole = requiredRoles.some(role => user.roles.includes(role));
        
        if (hasRole) {
          return true;
        }
        
        // Redirect to unauthorized page
        return this.router.createUrlTree(['/unauthorized']);
      })
    );
  }
}

// Route configuration
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard, RoleGuard],
    data: { roles: ['admin', 'superuser'] }
  }
];
```

**3. CanActivateChild - Protect Child Routes:**

```typescript
@Injectable({ providedIn: 'root' })
export class ParentAuthGuard implements CanActivateChild {
  constructor(private authService: AuthService) {}
  
  canActivateChild(
    childRoute: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean> {
    // Check authentication for all child routes
    return this.authService.isAuthenticated$;
  }
}

// Apply to parent route
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivateChild: [ParentAuthGuard], // Protects all children
    children: [
      { path: 'profile', component: ProfileComponent },
      { path: 'settings', component: SettingsComponent },
      { path: 'orders', component: OrdersComponent }
    ]
  }
];
```

**4. CanDeactivate - Unsaved Changes Warning:**

```typescript
// Component interface
export interface CanComponentDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

// Guard
@Injectable({ providedIn: 'root' })
export class UnsavedChangesGuard implements CanDeactivate<CanComponentDeactivate> {
  canDeactivate(
    component: CanComponentDeactivate,
    currentRoute: ActivatedRouteSnapshot,
    currentState: RouterStateSnapshot,
    nextState?: RouterStateSnapshot
  ): Observable<boolean> | boolean {
    return component.canDeactivate ? component.canDeactivate() : true;
  }
}

// Component implementation
@Component({})
export class FormEditorComponent implements CanComponentDeactivate {
  form: FormGroup;
  
  canDeactivate(): Observable<boolean> | boolean {
    if (this.form.dirty) {
      return confirm('You have unsaved changes. Do you really want to leave?');
    }
    return true;
  }
}

// Better implementation with dialog
@Component({})
export class BetterFormComponent implements CanComponentDeactivate {
  form: FormGroup;
  
  constructor(private dialog: MatDialog) {}
  
  canDeactivate(): Observable<boolean> {
    if (!this.form.dirty) {
      return of(true);
    }
    
    const dialogRef = this.dialog.open(ConfirmDialogComponent, {
      data: {
        title: 'Unsaved Changes',
        message: 'You have unsaved changes. Do you want to discard them?'
      }
    });
    
    return dialogRef.afterClosed();
  }
}

// Route configuration
const routes: Routes = [
  {
    path: 'edit/:id',
    component: FormEditorComponent,
    canDeactivate: [UnsavedChangesGuard]
  }
];
```

**5. CanLoad - Lazy Loading Protection:**

```typescript
@Injectable({ providedIn: 'root' })
export class CanLoadGuard implements CanLoad {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}
  
  canLoad(
    route: Route,
    segments: UrlSegment[]
  ): Observable<boolean> {
    return this.authService.isAuthenticated$.pipe(
      map(isAuthenticated => {
        if (!isAuthenticated) {
          this.router.navigate(['/login']);
          return false;
        }
        return true;
      })
    );
  }
}

// Difference from CanActivate:
// - CanLoad: Prevents module from loading (better for security)
// - CanActivate: Module loads but route activation is blocked

const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canLoad: [CanLoadGuard] // Module won't load if guard returns false
  }
];
```

**6. Functional Guards (Angular 15+):**

```typescript
// Functional CanActivate
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  return authService.isAuthenticated$.pipe(
    map(isAuth => isAuth || router.createUrlTree(['/login']))
  );
};

// Functional CanDeactivate
export const unsavedChangesGuard: CanDeactivateFn<CanComponentDeactivate> = (component) => {
  return component.canDeactivate ? component.canDeactivate() : true;
};

// Route configuration
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard]
  },
  {
    path: 'edit',
    component: EditComponent,
    canDeactivate: [unsavedChangesGuard]
  }
];
```

**Multiple Guards and Execution Order:**

```typescript
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [
      AuthGuard,      // Runs first
      RoleGuard,      // Runs second if AuthGuard passes
      FeatureFlagGuard // Runs third if RoleGuard passes
    ]
  }
];

// All guards must return true for navigation to proceed
// If any guard returns false or UrlTree, navigation is cancelled or redirected
```

**Real-World Guard Examples:**

**1. Subscription Required Guard:**

```typescript
@Injectable({ providedIn: 'root' })
export class SubscriptionGuard implements CanActivate {
  constructor(
    private subscriptionService: SubscriptionService,
    private router: Router
  ) {}
  
  canActivate(): Observable<boolean | UrlTree> {
    return this.subscriptionService.hasActiveSubscription().pipe(
      map(hasSubscription => {
        if (hasSubscription) {
          return true;
        }
        return this.router.createUrlTree(['/subscribe']);
      })
    );
  }
}
```

**2. Feature Flag Guard:**

```typescript
@Injectable({ providedIn: 'root' })
export class FeatureFlagGuard implements CanActivate {
  constructor(
    private featureService: FeatureService,
    private router: Router
  ) {}
  
  canActivate(route: ActivatedRouteSnapshot): boolean | UrlTree {
    const requiredFeature = route.data['feature'] as string;
    
    if (this.featureService.isEnabled(requiredFeature)) {
      return true;
    }
    
    return this.router.createUrlTree(['/not-available']);
  }
}

const routes: Routes = [
  {
    path: 'beta-feature',
    component: BetaFeatureComponent,
    canActivate: [FeatureFlagGuard],
    data: { feature: 'beta_features' }
  }
];
```

**3. Network Status Guard:**

```typescript
@Injectable({ providedIn: 'root' })
export class OnlineGuard implements CanActivate {
  constructor(
    private networkService: NetworkService,
    private router: Router
  ) {}
  
  canActivate(): boolean | UrlTree {
    if (this.networkService.isOnline()) {
      return true;
    }
    
    return this.router.createUrlTree(['/offline']);
  }
}
```

**Follow-up Questions:**

1. **What's the difference between CanActivate and CanLoad?**
   - CanActivate: Route can't be activated, but module still loads
   - CanLoad: Module doesn't load at all
   - CanLoad is better for security (prevents code from loading)
   - CanActivate is better for UX (module cached after first block)

2. **Can guards be asynchronous?**
   - Yes, all guards can return Observable<boolean>, Promise<boolean>, or boolean
   - Common for checking authentication status from server
   - Use RxJS operators to handle async logic

3. **How do you test route guards?**
   ```typescript
   describe('AuthGuard', () => {
     let guard: AuthGuard;
     let authService: jasmine.SpyObj<AuthService>;
     let router: jasmine.SpyObj<Router>;
     
     beforeEach(() => {
       const authSpy = jasmine.createSpyObj('AuthService', ['isAuthenticated$']);
       const routerSpy = jasmine.createSpyObj('Router', ['createUrlTree']);
       
       TestBed.configureTestingModule({
         providers: [
           AuthGuard,
           { provide: AuthService, useValue: authSpy },
           { provide: Router, useValue: routerSpy }
         ]
       });
       
       guard = TestBed.inject(AuthGuard);
       authService = TestBed.inject(AuthService) as jasmine.SpyObj<AuthService>;
       router = TestBed.inject(Router) as jasmine.SpyObj<Router>;
     });
     
     it('should allow authenticated users', (done) => {
       authService.isAuthenticated$.and.returnValue(of(true));
       
       guard.canActivate(null!, null!).subscribe(result => {
         expect(result).toBe(true);
         done();
       });
     });
     
     it('should redirect unauthenticated users', (done) => {
       authService.isAuthenticated$.and.returnValue(of(false));
       const urlTree = {} as UrlTree;
       router.createUrlTree.and.returnValue(urlTree);
       
       guard.canActivate(null!, null!).subscribe(result => {
         expect(result).toBe(urlTree);
         expect(router.createUrlTree).toHaveBeenCalledWith(['/login']);
         done();
       });
     });
   });
   ```

---

### Q3: Explain lazy loading and preloading strategies in Angular.

**What the interviewer wants to know:**
- Understanding of code splitting
- Knowledge of performance optimization
- Ability to balance initial load vs runtime performance

**Strong Answer:**

Lazy loading loads feature modules on-demand rather than at application startup, significantly reducing initial bundle size. Preloading strategies control when and how lazy modules are loaded.

**Basic Lazy Loading:**

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  {
    path: 'shop',
    loadChildren: () => import('./shop/shop.module').then(m => m.ShopModule)
  },
  {
    path: 'blog',
    loadChildren: () => import('./blog/blog.module').then(m => m.BlogModule)
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}

// Feature module
@NgModule({
  declarations: [AdminComponent, UsersComponent],
  imports: [
    CommonModule,
    RouterModule.forChild([
      { path: '', component: AdminComponent },
      { path: 'users', component: UsersComponent }
    ])
  ]
})
export class AdminModule {}
```

**Built-in Preloading Strategies:**

```typescript
// 1. NoPreloading (default) - No preloading
RouterModule.forRoot(routes, {
  preloadingStrategy: NoPreloading
})

// 2. PreloadAllModules - Preload all lazy modules
RouterModule.forRoot(routes, {
  preloadingStrategy: PreloadAllModules
})

// When to use:
// - NoPreloading: Very large app, slow network, critical initial load time
// - PreloadAllModules: Smaller app, fast network, prioritize UX
```

**Custom Preloading Strategy:**

```typescript
// Preload based on route data
@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preloadedModules: string[] = [];
  
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data && route.data['preload']) {
      // Log preloaded modules
      this.preloadedModules.push(route.path || 'unknown');
      console.log('Preloading:', route.path);
      
      return load();
    }
    
    return of(null);
  }
}

// Route configuration
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preload: true } // Will be preloaded
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.module').then(m => m.ReportsModule),
    data: { preload: false } // Won't be preloaded
  }
];

// Register strategy
RouterModule.forRoot(routes, {
  preloadingStrategy: SelectivePreloadingStrategy
})
```

**Network-Aware Preloading:**

```typescript
@Injectable({ providedIn: 'root' })
export class NetworkAwarePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Check network connection
    const connection = (navigator as any).connection;
    
    if (connection) {
      // Don't preload on slow connections
      if (connection.effectiveType === '2g' || connection.saveData) {
        return of(null);
      }
      
      // Preload on fast connections
      if (connection.effectiveType === '4g') {
        return load();
      }
    }
    
    // Default behavior
    return route.data && route.data['preload'] ? load() : of(null);
  }
}
```

**On-Demand Preloading:**

```typescript
@Injectable({ providedIn: 'root' })
export class OnDemandPreloadService {
  private preloadMap = new Map<string, () => Observable<any>>();
  
  constructor(private router: Router) {}
  
  // Store preload functions
  startPreload(routePath: string): void {
    const load = this.preloadMap.get(routePath);
    if (load) {
      load().subscribe(() => {
        console.log(`Preloaded: ${routePath}`);
      });
    }
  }
}

@Injectable({ providedIn: 'root' })
export class OnDemandPreloadStrategy implements PreloadingStrategy {
  private preloadQueue: string[] = [];
  
  constructor(private preloadService: OnDemandPreloadService) {}
  
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Store load function for later
    if (route.path) {
      this.preloadService['preloadMap'].set(route.path, load);
    }
    
    // Don't preload immediately
    return of(null);
  }
}

// Usage in component
@Component({})
export class DashboardComponent {
  constructor(private preloadService: OnDemandPreloadService) {}
  
  onUserHoverAdminLink(): void {
    // Preload admin module when user hovers over link
    this.preloadService.startPreload('admin');
  }
}
```

**Time-Based Preloading:**

```typescript
@Injectable({ providedIn: 'root' })
export class DelayedPreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const delay = route.data && route.data['preloadDelay'] || 0;
    
    if (delay === 0) {
      return of(null); // Don't preload
    }
    
    // Preload after delay
    return timer(delay).pipe(
      switchMap(() => load())
    );
  }
}

const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preloadDelay: 5000 } // Preload after 5 seconds
  }
];
```

**Priority-Based Preloading:**

```typescript
@Injectable({ providedIn: 'root' })
export class PriorityPreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const priority = route.data && route.data['priority'];
    
    if (!priority) {
      return of(null);
    }
    
    // Higher priority = shorter delay
    const delay = {
      high: 0,
      medium: 2000,
      low: 5000
    }[priority] || 0;
    
    return timer(delay).pipe(
      switchMap(() => load()),
      tap(() => console.log(`Preloaded ${route.path} (${priority} priority)`))
    );
  }
}

const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule),
    data: { priority: 'high' }
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.module').then(m => m.ReportsModule),
    data: { priority: 'medium' }
  },
  {
    path: 'archive',
    loadChildren: () => import('./archive/archive.module').then(m => m.ArchiveModule),
    data: { priority: 'low' }
  }
];
```

**Predictive Preloading:**

```typescript
@Injectable({ providedIn: 'root' })
export class PredictivePreloadingStrategy implements PreloadingStrategy {
  constructor(private analytics: AnalyticsService) {}
  
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Get user navigation patterns from analytics
    return this.analytics.getNavigationProbability(route.path || '').pipe(
      switchMap(probability => {
        // Preload if >50% chance user will navigate there
        if (probability > 0.5) {
          console.log(`Preloading ${route.path} (${probability * 100}% probability)`);
          return load();
        }
        return of(null);
      })
    );
  }
}
```

**Monitoring Preloading:**

```typescript
@Injectable({ providedIn: 'root' })
export class PreloadMonitoringService {
  private preloadedModules: Map<string, number> = new Map();
  
  constructor(private router: Router) {
    this.monitorPreloading();
  }
  
  private monitorPreloading(): void {
    this.router.events.subscribe(event => {
      if (event instanceof RouteConfigLoadStart) {
        const startTime = performance.now();
        this.preloadedModules.set(event.route.path || '', startTime);
        console.log('Started loading:', event.route.path);
      }
      
      if (event instanceof RouteConfigLoadEnd) {
        const startTime = this.preloadedModules.get(event.route.path || '');
        if (startTime) {
          const duration = performance.now() - startTime;
          console.log(`Loaded ${event.route.path} in ${duration}ms`);
        }
      }
    });
  }
  
  getPreloadStats(): Map<string, number> {
    return this.preloadedModules;
  }
}
```

**Performance Comparison:**

```typescript
// Scenario 1: No Lazy Loading
// Initial bundle: 5MB
// Time to Interactive: 8s on 3G

// Scenario 2: Lazy Loading + No Preloading
// Initial bundle: 1MB
// Time to Interactive: 2s
// Module load time: 500ms per navigation

// Scenario 3: Lazy Loading + PreloadAllModules
// Initial bundle: 1MB
// Time to Interactive: 2s
// Module load time: 0ms (already loaded)
// Total data transferred: 5MB (same as Scenario 1)

// Scenario 4: Lazy Loading + Selective Preloading
// Initial bundle: 1MB
// Time to Interactive: 2s
// Critical modules: 0ms (preloaded)
// Rare modules: 500ms (loaded on demand)
// Total data transferred: 2.5MB (50% reduction)
```

**Follow-up Questions:**

1. **What's the difference between lazy loading and preloading?**
   - Lazy loading: Load module when route is activated
   - Preloading: Load module in background before user navigates
   - Lazy loading reduces initial bundle, preloading improves navigation UX

2. **How do you measure the impact of lazy loading?**
   - Use Chrome DevTools Network tab
   - Check bundle sizes with webpack-bundle-analyzer
   - Measure Time to Interactive (TTI)
   - Monitor module load times in production

3. **Can you lazy load components without modules?**
   - Yes, with Angular 13+ standalone components
   - Use loadComponent instead of loadChildren
   - More granular code splitting

---

## What Interviewers Look For

### Strong Signals

1. **Route Configuration**: Clear understanding of route types
2. **Guard Knowledge**: Knows all guard types and when to use each
3. **Lazy Loading**: Can explain benefits and implementation
4. **Performance Awareness**: Discusses bundle size and optimization
5. **Real Experience**: Shares specific routing challenges solved
6. **Security Understanding**: Properly uses guards for protection
7. **Testing**: Can test routing and guards

## Red Flags to Avoid

1. **"Never used lazy loading"**: Limited scalability experience
2. **"Guards are just for authentication"**: Narrow understanding
3. **Confusion about guard types**: Conceptual gaps
4. **No preloading strategy knowledge**: Missing optimization skills
5. **Can't explain parameterized routes**: Basic knowledge gap
6. **No child routing experience**: Limited complex app experience
7. **Doesn't test routing**: Incomplete testing practices

## Key Takeaways

### Essential Concepts

1. **Route Configuration**
   - Static routes
   - Parameterized routes
   - Child routes
   - Lazy loaded routes
   - Redirects and wildcards

2. **Route Guards**
   - CanActivate: Protect routes
   - CanActivateChild: Protect child routes
   - CanDeactivate: Prevent navigation away
   - CanLoad: Prevent module loading
   - Resolve: Prefetch data

3. **Lazy Loading**
   - Reduces initial bundle size
   - Improves Time to Interactive
   - Requires proper module structure
   - Use with route guards for security

4. **Preloading Strategies**
   - NoPreloading: Minimal initial load
   - PreloadAllModules: Best UX after load
   - Custom strategies: Balance performance
   - Network-aware: Adapt to conditions

### Best Practices

1. Use lazy loading for feature modules
2. Implement proper route guards
3. Choose appropriate preloading strategy
4. Handle navigation errors
5. Use resolvers for required data
6. Test routing configuration
7. Monitor bundle sizes

### Interview Success Tips

1. **Start with basics**: Explain simple routes first
2. **Progress to advanced**: Show guard and lazy loading knowledge
3. **Discuss trade-offs**: Loading strategies have pros/cons
4. **Share real examples**: Routing challenges from projects
5. **Performance focus**: Always discuss optimization
6. **Security awareness**: Proper guard implementation
7. **Ask clarifying questions**: Understand app requirements

### Quick Reference

```typescript
// Basic route
{ path: 'about', component: AboutComponent }

// Parameterized route
{ path: 'user/:id', component: UserComponent }

// Child routes
{
  path: 'dashboard',
  component: DashboardComponent,
  children: [...]
}

// Lazy loading
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
}

// Guards
{
  path: 'protected',
  component: ProtectedComponent,
  canActivate: [AuthGuard],
  canDeactivate: [UnsavedChangesGuard]
}

// Navigate
this.router.navigate(['/user', userId]);
this.router.navigate(['/search'], { queryParams: { q: 'angular' } });
```

Remember: Routing is about more than navigation—it's about application architecture, performance, security, and user experience. Master these concepts to build scalable, performant Angular applications.
