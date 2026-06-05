# Route Guards in Angular

## The Idea

**In plain English:** Route guards are like bouncers at the door of a web page — they check whether you are allowed in before letting you navigate there, and can also stop you from leaving a page if you have unsaved work.

**Real-world analogy:** Imagine a movie theatre with age-restricted screening rooms. A staff member at each door checks your ID before you enter, turns you away (or points you to a different room) if you don't qualify, and also stops you from walking out mid-film if you left your bag inside.

- The staff member at the door = the guard function
- Checking your ID = the guard's logic (is the user logged in? do they have the right role?)
- Being turned away to a different room = redirecting to `/login` or `/unauthorized`
- Being stopped from leaving mid-film = `canDeactivate` guard (preventing navigation away from unsaved forms)

---

## Table of Contents

- [Introduction](#introduction)
- [canActivate Guard](#canactivate-guard)
- [canMatch Guard](#canmatch-guard)
- [canDeactivate Guard](#candeactivate-guard)
- [canActivateChild Guard](#canactivatechild-guard)
- [Functional Guards](#functional-guards)
- [Guard Return Types](#guard-return-types)
- [Dependency Injection in Guards](#dependency-injection-in-guards)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Route guards are interfaces that control navigation in Angular applications. They allow you to run logic before activating, deactivating, or loading routes, enabling authentication, authorization, data validation, and preventing navigation away from unsaved forms.

## canActivate Guard

### Basic canActivate Implementation

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isLoggedIn()) {
    return true;
  }

  // Redirect to login page
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard]
  },
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [authGuard]
  }
];
```

### Class-Based Guard (Legacy)

```typescript
// auth-class.guard.ts
import { Injectable } from '@angular/core';
import { 
  CanActivate, 
  ActivatedRouteSnapshot, 
  RouterStateSnapshot,
  Router,
  UrlTree
} from '@angular/router';
import { Observable } from 'rxjs';
import { AuthService } from './auth.service';

@Injectable({ providedIn: 'root' })
export class AuthClassGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
    if (this.authService.isLoggedIn()) {
      return true;
    }

    return this.router.createUrlTree(['/login'], {
      queryParams: { returnUrl: state.url }
    });
  }
}

// Usage in routes
export const routes: Routes = [
  {
    path: 'protected',
    component: ProtectedComponent,
    canActivate: [AuthClassGuard]
  }
];
```

### Role-Based Authorization Guard

```typescript
// role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export function roleGuard(allowedRoles: string[]): CanActivateFn {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);

    const userRoles = authService.getUserRoles();
    const hasRole = allowedRoles.some(role => userRoles.includes(role));

    if (hasRole) {
      return true;
    }

    // Redirect to unauthorized page
    return router.createUrlTree(['/unauthorized']);
  };
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [roleGuard(['admin'])]
  },
  {
    path: 'editor',
    component: EditorComponent,
    canActivate: [roleGuard(['admin', 'editor'])]
  },
  {
    path: 'viewer',
    component: ViewerComponent,
    canActivate: [roleGuard(['admin', 'editor', 'viewer'])]
  }
];
```

### Async canActivate Guard

```typescript
// auth-async.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';
import { map, take } from 'rxjs/operators';

export const authAsyncGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  // Returns Observable<boolean | UrlTree>
  return authService.checkAuth().pipe(
    take(1),
    map(isAuthenticated => {
      if (isAuthenticated) {
        return true;
      }
      return router.createUrlTree(['/login'], {
        queryParams: { returnUrl: state.url }
      });
    })
  );
};

// auth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private http: HttpClient) {}

  checkAuth(): Observable<boolean> {
    return this.http.get<{ authenticated: boolean }>('/api/auth/check').pipe(
      map(response => response.authenticated),
      catchError(() => of(false))
    );
  }
}
```

### Permission-Based Guard with Route Data

```typescript
// permission.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { PermissionService } from './permission.service';

export const permissionGuard: CanActivateFn = (route, state) => {
  const permissionService = inject(PermissionService);
  const router = inject(Router);

  const requiredPermission = route.data['permission'];

  if (!requiredPermission) {
    console.warn('No permission specified for route');
    return true;
  }

  if (permissionService.hasPermission(requiredPermission)) {
    return true;
  }

  return router.createUrlTree(['/access-denied']);
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'delete-user',
    component: DeleteUserComponent,
    canActivate: [permissionGuard],
    data: { permission: 'users.delete' }
  },
  {
    path: 'edit-settings',
    component: EditSettingsComponent,
    canActivate: [permissionGuard],
    data: { permission: 'settings.edit' }
  }
];
```

## canMatch Guard

### Basic canMatch Implementation

```typescript
// feature-flag.guard.ts
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { FeatureFlagService } from './feature-flag.service';

export const featureFlagGuard: CanMatchFn = (route, segments) => {
  const featureFlagService = inject(FeatureFlagService);
  
  const featureFlag = route.data?.['featureFlag'];
  
  if (!featureFlag) {
    return true;
  }

  return featureFlagService.isEnabled(featureFlag);
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'new-feature',
    component: NewFeatureComponent,
    canMatch: [featureFlagGuard],
    data: { featureFlag: 'new-feature-enabled' }
  },
  // Fallback route if feature is disabled
  {
    path: 'new-feature',
    component: ComingSoonComponent
  }
];
```

### canMatch vs canActivate

```typescript
// canMatch: Decides if route definition should be used (evaluated before lazy loading)
// canActivate: Decides if route can be activated (evaluated after module is loaded)

// subscription-tier.guard.ts
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { SubscriptionService } from './subscription.service';

export function subscriptionTierGuard(requiredTier: string): CanMatchFn {
  return (route, segments) => {
    const subscriptionService = inject(SubscriptionService);
    return subscriptionService.hasMinimumTier(requiredTier);
  };
}

// app.routes.ts
export const routes: Routes = [
  // Premium route - only loads module if user has premium
  {
    path: 'premium',
    loadChildren: () => import('./premium/premium.routes'),
    canMatch: [subscriptionTierGuard('premium')]
  },
  // Fallback for non-premium users
  {
    path: 'premium',
    component: UpgradeComponent
  },
  // Basic route - loads module, then checks activation
  {
    path: 'basic',
    loadChildren: () => import('./basic/basic.routes'),
    canActivate: [authGuard]
  }
];
```

### Environment-Based Route Matching

```typescript
// environment.guard.ts
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { environment } from '../environments/environment';

export const developmentOnlyGuard: CanMatchFn = (route, segments) => {
  return !environment.production;
};

export const productionOnlyGuard: CanMatchFn = (route, segments) => {
  return environment.production;
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'debug',
    component: DebugToolsComponent,
    canMatch: [developmentOnlyGuard]
  },
  {
    path: 'analytics',
    loadChildren: () => import('./analytics/analytics.routes'),
    canMatch: [productionOnlyGuard]
  }
];
```

## canDeactivate Guard

### Basic canDeactivate Implementation

```typescript
// unsaved-changes.guard.ts
import { CanDeactivateFn } from '@angular/router';

export interface CanComponentDeactivate {
  canDeactivate: () => boolean | Promise<boolean>;
}

export const unsavedChangesGuard: CanDeactivateFn<CanComponentDeactivate> = (
  component,
  currentRoute,
  currentState,
  nextState
) => {
  return component.canDeactivate ? component.canDeactivate() : true;
};

// form.component.ts
import { Component } from '@angular/core';
import { CanComponentDeactivate } from './unsaved-changes.guard';

@Component({
  selector: 'app-form',
  standalone: true,
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="formData" (input)="hasChanges = true">
      <button type="submit">Save</button>
    </form>
  `
})
export class FormComponent implements CanComponentDeactivate {
  formData = '';
  hasChanges = false;

  canDeactivate(): boolean | Promise<boolean> {
    if (!this.hasChanges) {
      return true;
    }

    return confirm('You have unsaved changes. Do you really want to leave?');
  }

  onSubmit(): void {
    // Save logic
    this.hasChanges = false;
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'edit',
    component: FormComponent,
    canDeactivate: [unsavedChangesGuard]
  }
];
```

### Advanced canDeactivate with Observable

```typescript
// dialog-deactivate.guard.ts
import { inject } from '@angular/core';
import { CanDeactivateFn } from '@angular/router';
import { Observable } from 'rxjs';
import { DialogService } from './dialog.service';

export interface CanComponentDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

export const dialogDeactivateGuard: CanDeactivateFn<CanComponentDeactivate> = (
  component,
  currentRoute,
  currentState,
  nextState
) => {
  if (!component.canDeactivate) {
    return true;
  }

  const result = component.canDeactivate();

  if (typeof result === 'boolean') {
    return result;
  }

  return result;
};

// editor.component.ts
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { DialogService } from './dialog.service';
import { CanComponentDeactivate } from './dialog-deactivate.guard';

@Component({
  selector: 'app-editor',
  standalone: true,
  template: `<textarea [(ngModel)]="content" (input)="onContentChange()"></textarea>`
})
export class EditorComponent implements CanComponentDeactivate {
  content = '';
  private pristineContent = '';
  private isDirty = false;

  constructor(private dialogService: DialogService) {}

  onContentChange(): void {
    this.isDirty = this.content !== this.pristineContent;
  }

  canDeactivate(): Observable<boolean> | boolean {
    if (!this.isDirty) {
      return true;
    }

    return this.dialogService.confirm({
      title: 'Unsaved Changes',
      message: 'You have unsaved changes. Do you want to leave?',
      confirmText: 'Leave',
      cancelText: 'Stay'
    });
  }

  save(): void {
    // Save logic
    this.pristineContent = this.content;
    this.isDirty = false;
  }
}

// dialog.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DialogService {
  confirm(options: {
    title: string;
    message: string;
    confirmText: string;
    cancelText: string;
  }): Observable<boolean> {
    // In real app, open a modal and return its result
    const result = window.confirm(`${options.title}\n\n${options.message}`);
    return of(result);
  }
}
```

### Form Dirty State Guard

```typescript
// form-dirty.guard.ts
import { CanDeactivateFn } from '@angular/router';
import { FormGroup } from '@angular/forms';

export interface HasFormGroup {
  form: FormGroup;
}

export const formDirtyGuard: CanDeactivateFn<HasFormGroup> = (
  component,
  currentRoute,
  currentState,
  nextState
) => {
  if (!component.form || !component.form.dirty) {
    return true;
  }

  return confirm('You have unsaved form changes. Do you want to discard them?');
};

// profile-edit.component.ts
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { HasFormGroup } from './form-dirty.guard';

@Component({
  selector: 'app-profile-edit',
  standalone: true,
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="name">
      <input formControlName="email">
      <button type="submit" [disabled]="!form.valid">Save</button>
    </form>
  `
})
export class ProfileEditComponent implements OnInit, HasFormGroup {
  form!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.form = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    });
  }

  onSubmit(): void {
    if (this.form.valid) {
      // Save logic
      this.form.markAsPristine();
    }
  }
}

// app.routes.ts
export const routes: Routes = [
  {
    path: 'profile/edit',
    component: ProfileEditComponent,
    canDeactivate: [formDirtyGuard]
  }
];
```

## canActivateChild Guard

### Basic canActivateChild Implementation

```typescript
// admin.guard.ts
import { inject } from '@angular/core';
import { CanActivateChildFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const adminGuard: CanActivateChildFn = (childRoute, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAdmin()) {
    return true;
  }

  return router.createUrlTree(['/unauthorized']);
};

// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    component: AdminLayoutComponent,
    canActivateChild: [adminGuard],
    children: [
      { path: 'users', component: UsersComponent },
      { path: 'settings', component: SettingsComponent },
      { path: 'reports', component: ReportsComponent }
    ]
  }
];
```

### Combining canActivate and canActivateChild

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardLayoutComponent,
    canActivate: [authGuard], // Check auth for parent
    canActivateChild: [roleGuard(['user'])], // Check role for all children
    children: [
      { path: 'overview', component: OverviewComponent },
      { path: 'profile', component: ProfileComponent },
      {
        path: 'admin',
        component: AdminSectionComponent,
        canActivate: [roleGuard(['admin'])], // Additional check for specific child
        children: [
          { path: 'users', component: AdminUsersComponent },
          { path: 'settings', component: AdminSettingsComponent }
        ]
      }
    ]
  }
];
```

### Audit Logging Guard

```typescript
// audit-log.guard.ts
import { inject } from '@angular/core';
import { CanActivateChildFn } from '@angular/router';
import { AuditService } from './audit.service';
import { AuthService } from './auth.service';

export const auditLogGuard: CanActivateChildFn = (childRoute, state) => {
  const auditService = inject(AuditService);
  const authService = inject(AuthService);

  const user = authService.getCurrentUser();
  const timestamp = new Date();
  const path = state.url;

  auditService.log({
    user: user?.email || 'anonymous',
    action: 'route-access',
    path,
    timestamp
  });

  return true;
};

// audit.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface AuditLog {
  user: string;
  action: string;
  path: string;
  timestamp: Date;
}

@Injectable({ providedIn: 'root' })
export class AuditService {
  constructor(private http: HttpClient) {}

  log(entry: AuditLog): void {
    this.http.post('/api/audit-log', entry).subscribe();
  }
}
```

## Functional Guards

### Creating Reusable Functional Guards

```typescript
// guards/factories.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

// Guard factory for role checking
export function createRoleGuard(allowedRoles: string[]): CanActivateFn {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);

    const userRoles = authService.getUserRoles();
    const hasAccess = allowedRoles.some(role => userRoles.includes(role));

    return hasAccess || router.createUrlTree(['/unauthorized']);
  };
}

// Guard factory for permissions
export function createPermissionGuard(permission: string): CanActivateFn {
  return (route, state) => {
    const permissionService = inject(PermissionService);
    const router = inject(Router);

    return permissionService.hasPermission(permission) 
      || router.createUrlTree(['/access-denied']);
  };
}

// Guard factory for subscription tier
export function createSubscriptionGuard(minTier: number): CanActivateFn {
  return (route, state) => {
    const subscriptionService = inject(SubscriptionService);
    const router = inject(Router);

    const userTier = subscriptionService.getUserTier();
    
    return userTier >= minTier 
      || router.createUrlTree(['/upgrade'], {
        queryParams: { required: minTier }
      });
  };
}

// Usage in routes
export const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [createRoleGuard(['admin'])]
  },
  {
    path: 'delete',
    component: DeleteComponent,
    canActivate: [createPermissionGuard('content.delete')]
  },
  {
    path: 'premium',
    component: PremiumComponent,
    canActivate: [createSubscriptionGuard(2)]
  }
];
```

### Composing Multiple Guards

```typescript
// guard-composer.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router, UrlTree } from '@angular/router';
import { Observable, of, forkJoin } from 'rxjs';
import { map } from 'rxjs/operators';

type GuardResult = boolean | UrlTree | Observable<boolean | UrlTree> | Promise<boolean | UrlTree>;

export function composeGuards(...guards: CanActivateFn[]): CanActivateFn {
  return (route, state) => {
    const results = guards.map(guard => {
      const result = guard(route, state);
      
      // Convert to Observable
      if (result instanceof Observable) {
        return result;
      } else if (result instanceof Promise) {
        return from(result);
      } else {
        return of(result);
      }
    });

    return forkJoin(results).pipe(
      map(guardResults => {
        // Return false or UrlTree if any guard denies access
        for (const result of guardResults) {
          if (result !== true) {
            return result;
          }
        }
        return true;
      })
    );
  };
}

// Usage
export const strictAccessGuard = composeGuards(
  authGuard,
  roleGuard(['admin']),
  subscriptionGuard('premium')
);

export const routes: Routes = [
  {
    path: 'restricted',
    component: RestrictedComponent,
    canActivate: [strictAccessGuard]
  }
];
```

### Parameterized Functional Guards

```typescript
// parameterized-guards.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

// Age verification guard
export function minAgeGuard(minAge: number): CanActivateFn {
  return (route, state) => {
    const userService = inject(UserService);
    const router = inject(Router);

    const user = userService.getCurrentUser();
    
    if (!user) {
      return router.createUrlTree(['/login']);
    }

    const age = calculateAge(user.dateOfBirth);
    
    return age >= minAge || router.createUrlTree(['/age-restricted']);
  };
}

// Time-based access guard
export function timeWindowGuard(startHour: number, endHour: number): CanActivateFn {
  return (route, state) => {
    const currentHour = new Date().getHours();
    return currentHour >= startHour && currentHour < endHour;
  };
}

// IP whitelist guard
export function ipWhitelistGuard(allowedIps: string[]): CanActivateFn {
  return (route, state) => {
    const networkService = inject(NetworkService);
    const userIp = networkService.getClientIp();
    return allowedIps.includes(userIp);
  };
}

// Usage
export const routes: Routes = [
  {
    path: 'adult-content',
    component: AdultContentComponent,
    canActivate: [minAgeGuard(18)]
  },
  {
    path: 'business-hours',
    component: BusinessHoursComponent,
    canActivate: [timeWindowGuard(9, 17)]
  },
  {
    path: 'internal',
    component: InternalComponent,
    canActivate: [ipWhitelistGuard(['192.168.1.1', '10.0.0.1'])]
  }
];

function calculateAge(dateOfBirth: Date): number {
  const today = new Date();
  const birthDate = new Date(dateOfBirth);
  let age = today.getFullYear() - birthDate.getFullYear();
  const monthDiff = today.getMonth() - birthDate.getMonth();
  
  if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
    age--;
  }
  
  return age;
}
```

## Guard Return Types

### All Valid Guard Return Types

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, Router, UrlTree } from '@angular/router';
import { Observable, of } from 'rxjs';
import { map } from 'rxjs/operators';

// 1. Boolean - direct allow/deny
export const booleanGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.isLoggedIn();
};

// 2. UrlTree - redirect to another route
export const urlTreeGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.isLoggedIn()) {
    return true;
  }
  
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// 3. Observable<boolean> - async boolean
export const observableBooleanGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  
  return authService.checkAuth().pipe(
    map(isAuthenticated => isAuthenticated)
  );
};

// 4. Observable<UrlTree> - async redirect
export const observableUrlTreeGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  return authService.checkAuth().pipe(
    map(isAuthenticated => {
      if (isAuthenticated) {
        return true;
      }
      return router.createUrlTree(['/login']);
    })
  );
};

// 5. Promise<boolean> - promise-based async
export const promiseBooleanGuard: CanActivateFn = async (route, state) => {
  const authService = inject(AuthService);
  
  try {
    const isAuthenticated = await authService.verifyToken();
    return isAuthenticated;
  } catch {
    return false;
  }
};

// 6. Promise<UrlTree> - promise-based redirect
export const promiseUrlTreeGuard: CanActivateFn = async (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  const isAuthenticated = await authService.verifyToken();
  
  if (isAuthenticated) {
    return true;
  }
  
  return router.createUrlTree(['/login']);
};
```

## Dependency Injection in Guards

### Using inject() in Functional Guards

```typescript
// modern-guards.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';
import { Logger Service } from './logger.service';
import { NotificationService } from './notification.service';

export const modernAuthGuard: CanActivateFn = (route, state) => {
  // Inject multiple services
  const authService = inject(AuthService);
  const router = inject(Router);
  const logger = inject(LoggerService);
  const notifications = inject(NotificationService);

  logger.log(`Checking auth for: ${state.url}`);

  if (!authService.isLoggedIn()) {
    notifications.showWarning('Please log in to access this page');
    logger.log('Auth failed: User not logged in');
    
    return router.createUrlTree(['/login'], {
      queryParams: { returnUrl: state.url }
    });
  }

  logger.log('Auth successful');
  return true;
};
```

### Conditional Dependency Injection

```typescript
// conditional-injection.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

export const browserOnlyGuard: CanActivateFn = (route, state) => {
  const platformId = inject(PLATFORM_ID);
  
  if (!isPlatformBrowser(platformId)) {
    console.log('Server-side: allowing navigation');
    return true;
  }

  // Browser-specific logic
  const localStorageService = inject(LocalStorageService);
  const hasAccess = localStorageService.getItem('access-token') !== null;
  
  return hasAccess;
};
```

## Common Mistakes

### 1. Not Handling Async Properly

```typescript
// WRONG - Guard doesn't wait for async operation
export const wrongAsyncGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  
  let isAuthenticated = false;
  authService.checkAuth().subscribe(result => {
    isAuthenticated = result;
  });
  
  return isAuthenticated; // Always false!
};

// CORRECT - Return the Observable
export const correctAsyncGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  
  return authService.checkAuth();
};
```

### 2. Forgetting to Redirect

```typescript
// WRONG - Just blocks without explaining
export const wrongGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.isLoggedIn();
};

// CORRECT - Redirect to login
export const correctGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  if (authService.isLoggedIn()) {
    return true;
  }
  
  return router.createUrlTree(['/login']);
};
```

### 3. Incorrect canDeactivate Interface

```typescript
// WRONG - Component doesn't implement interface
@Component({})
export class WrongComponent {
  hasUnsavedChanges = true;
}

// Guard expects method that doesn't exist
export const guard: CanDeactivateFn<WrongComponent> = (component) => {
  return component.canDeactivate(); // Error: method doesn't exist!
};

// CORRECT - Implement interface
interface CanComponentDeactivate {
  canDeactivate: () => boolean;
}

@Component({})
export class CorrectComponent implements CanComponentDeactivate {
  hasUnsavedChanges = true;
  
  canDeactivate(): boolean {
    return !this.hasUnsavedChanges;
  }
}
```

### 4. Using Class Guards in Standalone Apps

```typescript
// WRONG - Class guards don't work well with standalone
@Injectable({ providedIn: 'root' })
export class OldStyleGuard implements CanActivate {
  canActivate() { return true; }
}

// CORRECT - Use functional guards
export const modernGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.isLoggedIn();
};
```

## Best Practices

### 1. Create Guard Composition Utilities

```typescript
// guard-utils.ts
import { CanActivateFn } from '@angular/router';

export function combineGuards(...guards: CanActivateFn[]): CanActivateFn {
  return (route, state) => {
    for (const guard of guards) {
      const result = guard(route, state);
      if (result !== true) {
        return result;
      }
    }
    return true;
  };
}

// Usage
const protectedRouteGuard = combineGuards(
  authGuard,
  roleGuard(['admin']),
  subscriptionGuard
);
```

### 2. Centralize Guard Logic

```typescript
// guards/index.ts
export { authGuard } from './auth.guard';
export { roleGuard } from './role.guard';
export { permissionGuard } from './permission.guard';
export { unsavedChangesGuard } from './unsaved-changes.guard';

// app.routes.ts
import { authGuard, roleGuard } from './guards';

export const routes: Routes = [
  {
    path: 'admin',
    canActivate: [authGuard, roleGuard(['admin'])],
    component: AdminComponent
  }
];
```

### 3. Use Type-Safe Route Data

```typescript
// route-data.types.ts
export interface RouteData {
  permission?: string;
  roles?: string[];
  title?: string;
  requiresAuth?: boolean;
}

// permission.guard.ts
export const permissionGuard: CanActivateFn = (route, state) => {
  const data = route.data as RouteData;
  const permission = data.permission;
  
  if (!permission) {
    return true;
  }
  
  const permissionService = inject(PermissionService);
  return permissionService.hasPermission(permission);
};
```

### 4. Implement Guard Logging

```typescript
// logged.guard.ts
import { CanActivateFn } from '@angular/router';

export function withLogging(guard: CanActivateFn, guardName: string): CanActivateFn {
  return (route, state) => {
    console.log(`[${guardName}] Checking route: ${state.url}`);
    
    const result = guard(route, state);
    
    if (result === true) {
      console.log(`[${guardName}] Access granted`);
    } else {
      console.log(`[${guardName}] Access denied`);
    }
    
    return result;
  };
}

// Usage
export const loggedAuthGuard = withLogging(authGuard, 'AuthGuard');
```

## Interview Questions

### Q1: What's the difference between canActivate and canMatch?

**Answer:** `canActivate` runs after the route is matched and module loaded, preventing route activation. `canMatch` runs before matching, preventing the route from being selected and lazy modules from loading. Use `canMatch` for feature flags and `canActivate` for authentication.

### Q2: When should you use canActivateChild instead of canActivate?

**Answer:** Use `canActivateChild` to protect all child routes with a single guard instead of adding the same guard to each child. It's more maintainable and efficient for protecting entire route subtrees.

### Q3: How do you handle async operations in guards?

**Answer:** Return an `Observable<boolean | UrlTree>` or `Promise<boolean | UrlTree>`. Angular's router automatically subscribes and waits for the result before proceeding with navigation.

### Q4: What should a guard return when denying access?

**Answer:** Return `false` to cancel navigation, or better, return a `UrlTree` created with `router.createUrlTree()` to redirect users to an appropriate page (login, unauthorized, etc.).

### Q5: How do you prevent navigation away from unsaved forms?

**Answer:** Use `canDeactivate` guard. The component implements an interface with a `canDeactivate()` method that returns whether navigation should proceed, often showing a confirmation dialog if there are unsaved changes.

### Q6: Can you inject services in functional guards?

**Answer:** Yes, use the `inject()` function from `@angular/core` within the guard function to inject services, router, or any other dependencies.

### Q7: How do you compose multiple guards?

**Answer:** Create a utility function that runs guards in sequence, or add multiple guards to the `canActivate` array. The router runs them in order and all must pass for navigation to proceed.

### Q8: What's the purpose of ActivatedRouteSnapshot in guards?

**Answer:** It provides access to route information like parameters, data, URL segments, and query params, allowing guards to make decisions based on the target route's configuration.

## Key Takeaways

1. **Guards control navigation** before routes activate or deactivate
2. **canActivate** protects route activation after loading
3. **canMatch** prevents route matching and lazy loading
4. **canDeactivate** prevents navigation away from routes
5. **canActivateChild** protects all child routes at once
6. **Functional guards** are modern, preferred over class-based
7. **Return UrlTree** to redirect, not just boolean false
8. **Handle async** operations by returning Observable/Promise
9. **Use inject()** for dependency injection in functional guards
10. **Compose guards** for complex authorization logic

## Resources

- [Angular Router Guards Documentation](https://angular.io/guide/router#preventing-unauthorized-access)
- [CanActivate API](https://angular.io/api/router/CanActivateFn)
- [CanDeactivate API](https://angular.io/api/router/CanDeactivateFn)
- [CanMatch API](https://angular.io/api/router/CanMatchFn)
- [Functional Guards Guide](https://angular.io/guide/router#functional-guards)
- [Router inject() Function](https://angular.io/api/core/inject)
- [Route Guards Tutorial](https://angular.io/guide/router-tutorial-toh#milestone-5-route-guards)
