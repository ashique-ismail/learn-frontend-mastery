# Functional Guards in Angular (Angular 14+)

## The Idea

**In plain English:** A functional guard is a gatekeeper function in Angular that checks whether a visitor is allowed to enter a particular page — it runs before the page loads and can send the visitor somewhere else if they do not have permission.

**Real-world analogy:** Imagine a cinema where a staff member checks your ticket before letting you into the screening room. If your ticket is valid, you walk in; if not, you are directed back to the ticket booth.

- The staff member = the functional guard (the function that runs the check)
- Your ticket = the user's authentication or role data (the credential being checked)
- The screening room = the protected route/page (the destination being guarded)

---

## Table of Contents

- [Introduction](#introduction)
- [Functional Guards Overview](#functional-guards-overview)
- [Using inject() in Guards](#using-inject-in-guards)
- [Guard Composition](#guard-composition)
- [Advanced Patterns](#advanced-patterns)
- [Migration from Class Guards](#migration-from-class-guards)
- [Testing Functional Guards](#testing-functional-guards)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Functional guards, introduced in Angular 14+, provide a simpler, more tree-shakeable alternative to class-based guards. They leverage the `inject()` function for dependency injection and enable powerful composition patterns for route protection logic.

## Functional Guards Overview

### Basic Functional Guard Structure

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

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard]
  }
];
```

### All Guard Types as Functions

```typescript
// guards/all-types.ts
import { inject } from '@angular/core';
import {
  CanActivateFn,
  CanActivateChildFn,
  CanDeactivateFn,
  CanMatchFn,
  ResolveFn
} from '@angular/router';

// CanActivate
export const canActivateGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.isLoggedIn();
};

// CanActivateChild
export const canActivateChildGuard: CanActivateChildFn = (childRoute, state) => {
  const roleService = inject(RoleService);
  return roleService.hasAccess();
};

// CanDeactivate
export const canDeactivateGuard: CanDeactivateFn<ComponentType> = (
  component,
  currentRoute,
  currentState,
  nextState
) => {
  return component.canDeactivate ? component.canDeactivate() : true;
};

// CanMatch
export const canMatchGuard: CanMatchFn = (route, segments) => {
  const featureService = inject(FeatureService);
  return featureService.isEnabled('new-feature');
};

// Resolve
export const dataResolver: ResolveFn<DataType> = (route, state) => {
  const dataService = inject(DataService);
  return dataService.loadData(route.params['id']);
};
```

### Advantages Over Class Guards

```typescript
// OLD: Class-based guard (verbose, more boilerplate)
@Injectable({ providedIn: 'root' })
export class OldAuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean | UrlTree {
    if (this.authService.isLoggedIn()) {
      return true;
    }
    return this.router.createUrlTree(['/login']);
  }
}

// NEW: Functional guard (concise, tree-shakeable)
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isLoggedIn()
    ? true
    : router.createUrlTree(['/login']);
};

// Benefits:
// 1. Less boilerplate
// 2. Better tree-shaking
// 3. Easier composition
// 4. No class overhead
// 5. More functional style
```

## Using inject() in Guards

### Basic Dependency Injection

```typescript
// inject-examples.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';

export const multiServiceGuard: CanActivateFn = (route, state) => {
  // Inject multiple services
  const authService = inject(AuthService);
  const logger = inject(LoggerService);
  const analytics = inject(AnalyticsService);
  const router = inject(Router);

  // Log the attempt
  logger.info(`Guard checking: ${state.url}`);
  
  // Track analytics
  analytics.trackEvent('route-guard-check', {
    path: state.url,
    timestamp: Date.now()
  });

  // Check authentication
  const isAuthenticated = authService.isLoggedIn();
  
  if (!isAuthenticated) {
    logger.warn('Access denied - not authenticated');
    return router.createUrlTree(['/login']);
  }

  logger.info('Access granted');
  return true;
};
```

### Injecting Optional Dependencies

```typescript
// optional-injection.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { DOCUMENT } from '@angular/common';

export const themeGuard: CanActivateFn = (route, state) => {
  // Required dependency
  const themeService = inject(ThemeService);
  
  // Optional dependency
  const document = inject(DOCUMENT, { optional: true });
  
  // Platform-specific injection
  const platformId = inject(PLATFORM_ID);

  if (document) {
    const theme = themeService.getCurrentTheme();
    document.body.classList.add(`theme-${theme}`);
  }

  return true;
};
```

### Using Injection Tokens

```typescript
// tokens.ts
import { InjectionToken } from '@angular/core';

export interface AppConfig {
  apiUrl: string;
  features: {
    enableBeta: boolean;
    enableExperimental: boolean;
  };
}

export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// config.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { APP_CONFIG } from './tokens';

export const betaFeatureGuard: CanActivateFn = (route, state) => {
  const config = inject(APP_CONFIG);
  const router = inject(Router);

  if (config.features.enableBeta) {
    return true;
  }

  return router.createUrlTree(['/coming-soon']);
};

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { APP_CONFIG } from './tokens';

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiUrl: 'https://api.example.com',
        features: {
          enableBeta: true,
          enableExperimental: false
        }
      }
    }
  ]
};
```

### Conditional Injection Based on Environment

```typescript
// environment-aware.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

export const browserOnlyGuard: CanActivateFn = (route, state) => {
  const platformId = inject(PLATFORM_ID);

  if (isPlatformBrowser(platformId)) {
    // Browser-specific logic
    const storageService = inject(StorageService);
    return storageService.has('auth-token');
  }

  if (isPlatformServer(platformId)) {
    // Server-side always allows (will check on browser)
    return true;
  }

  return false;
};
```

## Guard Composition

### Sequential Guard Composition

```typescript
// guard-composer.ts
import { CanActivateFn } from '@angular/router';

export function composeGuards(...guards: CanActivateFn[]): CanActivateFn {
  return (route, state) => {
    for (const guard of guards) {
      const result = guard(route, state);
      
      // If any guard denies, stop and return that result
      if (result !== true) {
        return result;
      }
    }
    
    return true;
  };
}

// Usage
export const strictAuthGuard = composeGuards(
  authGuard,
  emailVerifiedGuard,
  twoFactorGuard,
  roleGuard(['admin'])
);

// app.routes.ts
export const routes: Routes = [
  {
    path: 'super-secure',
    component: SuperSecureComponent,
    canActivate: [strictAuthGuard]
  }
];
```

### Async Guard Composition

```typescript
// async-composer.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { Observable, of, forkJoin } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';

type GuardResult = boolean | UrlTree | Observable<boolean | UrlTree> | Promise<boolean | UrlTree>;

export function composeAsyncGuards(...guards: CanActivateFn[]): CanActivateFn {
  return (route, state) => {
    const results = guards.map(guard => {
      const result = guard(route, state);
      
      // Normalize to Observable
      if (result instanceof Observable) {
        return result;
      } else if (result instanceof Promise) {
        return from(result);
      } else {
        return of(result);
      }
    });

    // Wait for all guards to complete
    return forkJoin(results).pipe(
      map(guardResults => {
        // Return first non-true result or true if all pass
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

// Usage with async guards
export const asyncAuthGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.validateTokenAsync();
};

export const asyncRoleGuard: CanActivateFn = (route, state) => {
  const roleService = inject(RoleService);
  return roleService.checkRoleAsync('admin');
};

export const fullAsyncGuard = composeAsyncGuards(
  asyncAuthGuard,
  asyncRoleGuard
);
```

### Conditional Guard Composition

```typescript
// conditional-composer.ts
import { CanActivateFn } from '@angular/router';

interface GuardCondition {
  guard: CanActivateFn;
  condition: (route: ActivatedRouteSnapshot, state: RouterStateSnapshot) => boolean;
}

export function composeConditionalGuards(...conditions: GuardCondition[]): CanActivateFn {
  return (route, state) => {
    for (const { guard, condition } of conditions) {
      if (condition(route, state)) {
        const result = guard(route, state);
        if (result !== true) {
          return result;
        }
      }
    }
    return true;
  };
}

// Usage
export const conditionalGuard = composeConditionalGuards(
  {
    guard: authGuard,
    condition: (route) => !route.data['public']
  },
  {
    guard: roleGuard(['admin']),
    condition: (route) => route.data['adminOnly'] === true
  },
  {
    guard: subscriptionGuard,
    condition: (route) => route.data['requiresSubscription'] === true
  }
);
```

### Guard Pipeline with Middleware Pattern

```typescript
// guard-pipeline.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router, UrlTree } from '@angular/router';

type GuardContext = {
  route: ActivatedRouteSnapshot;
  state: RouterStateSnapshot;
  data: Map<string, any>;
};

type GuardMiddleware = (
  context: GuardContext,
  next: () => boolean | UrlTree
) => boolean | UrlTree;

export function createGuardPipeline(...middlewares: GuardMiddleware[]): CanActivateFn {
  return (route, state) => {
    const context: GuardContext = {
      route,
      state,
      data: new Map()
    };

    let index = 0;

    const next = (): boolean | UrlTree => {
      if (index >= middlewares.length) {
        return true;
      }

      const middleware = middlewares[index++];
      return middleware(context, next);
    };

    return next();
  };
}

// Middleware examples
const authMiddleware: GuardMiddleware = (context, next) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (!authService.isLoggedIn()) {
    return router.createUrlTree(['/login']);
  }

  // Store user in context for next middleware
  context.data.set('user', authService.getCurrentUser());
  return next();
};

const roleMiddleware: GuardMiddleware = (context, next) => {
  const router = inject(Router);
  const user = context.data.get('user');
  const requiredRole = context.route.data['role'];

  if (requiredRole && !user.roles.includes(requiredRole)) {
    return router.createUrlTree(['/unauthorized']);
  }

  return next();
};

const loggingMiddleware: GuardMiddleware = (context, next) => {
  const logger = inject(LoggerService);
  logger.log(`Accessing: ${context.state.url}`);
  
  const result = next();
  
  logger.log(`Result: ${result}`);
  return result;
};

// Usage
export const pipelineGuard = createGuardPipeline(
  loggingMiddleware,
  authMiddleware,
  roleMiddleware
);
```

## Advanced Patterns

### Guard Factory Pattern

```typescript
// guard-factories.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

// Factory for role-based guards
export function createRoleGuard(roles: string[]): CanActivateFn {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);

    const userRoles = authService.getUserRoles();
    const hasRequiredRole = roles.some(role => userRoles.includes(role));

    if (hasRequiredRole) {
      return true;
    }

    return router.createUrlTree(['/unauthorized'], {
      queryParams: { required: roles.join(',') }
    });
  };
}

// Factory for permission-based guards
export function createPermissionGuard(permission: string): CanActivateFn {
  return (route, state) => {
    const permissionService = inject(PermissionService);
    const router = inject(Router);

    if (permissionService.hasPermission(permission)) {
      return true;
    }

    return router.createUrlTree(['/access-denied'], {
      queryParams: { permission }
    });
  };
}

// Factory for feature flag guards
export function createFeatureFlagGuard(flag: string): CanActivateFn {
  return (route, state) => {
    const featureService = inject(FeatureFlagService);
    const router = inject(Router);

    if (featureService.isEnabled(flag)) {
      return true;
    }

    return router.createUrlTree(['/not-available']);
  };
}

// Usage
export const routes: Routes = [
  {
    path: 'admin',
    canActivate: [createRoleGuard(['admin'])],
    component: AdminComponent
  },
  {
    path: 'delete',
    canActivate: [createPermissionGuard('users.delete')],
    component: DeleteComponent
  },
  {
    path: 'beta',
    canActivate: [createFeatureFlagGuard('beta-feature')],
    component: BetaComponent
  }
];
```

### Caching Guard Results

```typescript
// cached-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { Observable, of } from 'rxjs';
import { tap, shareReplay } from 'rxjs/operators';

interface CacheEntry {
  result: boolean | UrlTree;
  timestamp: number;
}

const guardCache = new Map<string, CacheEntry>();
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

export function createCachedGuard(
  guard: CanActivateFn,
  cacheKey: (route: ActivatedRouteSnapshot) => string
): CanActivateFn {
  return (route, state) => {
    const key = cacheKey(route);
    const cached = guardCache.get(key);
    
    if (cached && Date.now() - cached.timestamp < CACHE_DURATION) {
      console.log(`Using cached result for: ${key}`);
      return cached.result;
    }

    const result = guard(route, state);

    // Handle Observable results
    if (result instanceof Observable) {
      return result.pipe(
        tap(guardResult => {
          guardCache.set(key, {
            result: guardResult,
            timestamp: Date.now()
          });
        }),
        shareReplay(1)
      );
    }

    // Handle synchronous results
    if (typeof result === 'boolean' || result instanceof UrlTree) {
      guardCache.set(key, {
        result,
        timestamp: Date.now()
      });
    }

    return result;
  };
}

// Usage
const expensiveAuthGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.performExpensiveAuthCheck();
};

export const cachedAuthGuard = createCachedGuard(
  expensiveAuthGuard,
  (route) => `auth-${route.params['id']}`
);
```

### Retry Logic in Guards

```typescript
// retry-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { Observable, of } from 'rxjs';
import { retry, catchError, map } from 'rxjs/operators';

export function createRetryGuard(
  guard: CanActivateFn,
  retryCount = 3
): CanActivateFn {
  return (route, state) => {
    const result = guard(route, state);
    const router = inject(Router);

    if (result instanceof Observable) {
      return result.pipe(
        retry(retryCount),
        catchError(error => {
          console.error('Guard failed after retries:', error);
          return of(router.createUrlTree(['/error'], {
            queryParams: { message: 'Authentication failed' }
          }));
        })
      );
    }

    return result;
  };
}

// Usage
const unstableAuthGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.checkAuthWithApi(); // Might fail
};

export const retriedAuthGuard = createRetryGuard(unstableAuthGuard, 3);
```

### Rate Limiting Guard

```typescript
// rate-limit-guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

interface RateLimitEntry {
  count: number;
  resetTime: number;
}

const rateLimits = new Map<string, RateLimitEntry>();

export function createRateLimitGuard(
  maxAttempts: number,
  windowMs: number
): CanActivateFn {
  return (route, state) => {
    const router = inject(Router);
    const key = state.url;
    const now = Date.now();

    let entry = rateLimits.get(key);

    if (!entry || now > entry.resetTime) {
      entry = {
        count: 0,
        resetTime: now + windowMs
      };
      rateLimits.set(key, entry);
    }

    entry.count++;

    if (entry.count > maxAttempts) {
      return router.createUrlTree(['/rate-limited'], {
        queryParams: { 
          retryAfter: Math.ceil((entry.resetTime - now) / 1000)
        }
      });
    }

    return true;
  };
}

// Usage: Max 5 attempts per minute
export const rateLimitedGuard = createRateLimitGuard(5, 60 * 1000);
```

## Migration from Class Guards

### Converting Class to Functional Guard

```typescript
// BEFORE: Class-based guard
@Injectable({ providedIn: 'root' })
export class OldAuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router,
    private logger: LoggerService
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean | UrlTree> {
    this.logger.log(`Checking auth for: ${state.url}`);

    return this.authService.isAuthenticated$.pipe(
      map(isAuthenticated => {
        if (isAuthenticated) {
          return true;
        }
        return this.router.createUrlTree(['/login'], {
          queryParams: { returnUrl: state.url }
        });
      })
    );
  }
}

// AFTER: Functional guard
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const logger = inject(LoggerService);

  logger.log(`Checking auth for: ${state.url}`);

  return authService.isAuthenticated$.pipe(
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
```

### Converting Complex Class Guards

```typescript
// BEFORE: Complex class guard
@Injectable({ providedIn: 'root' })
export class ComplexGuard implements CanActivate, CanActivateChild {
  private cache = new Map<string, boolean>();

  constructor(
    private authService: AuthService,
    private roleService: RoleService,
    private permissionService: PermissionService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean | UrlTree> {
    return this.checkAccess(route, state);
  }

  canActivateChild(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean | UrlTree> {
    return this.checkAccess(route, state);
  }

  private checkAccess(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean | UrlTree> {
    const cacheKey = state.url;
    
    if (this.cache.has(cacheKey)) {
      return of(this.cache.get(cacheKey)!);
    }

    return combineLatest([
      this.authService.isAuthenticated$,
      this.roleService.getUserRoles$,
      this.permissionService.getUserPermissions$
    ]).pipe(
      map(([isAuth, roles, permissions]) => {
        if (!isAuth) {
          return this.router.createUrlTree(['/login']);
        }

        const requiredRole = route.data['role'];
        const requiredPermission = route.data['permission'];

        if (requiredRole && !roles.includes(requiredRole)) {
          return this.router.createUrlTree(['/unauthorized']);
        }

        if (requiredPermission && !permissions.includes(requiredPermission)) {
          return this.router.createUrlTree(['/forbidden']);
        }

        this.cache.set(cacheKey, true);
        return true;
      })
    );
  }
}

// AFTER: Functional guards with shared logic
const cache = new Map<string, boolean>();

function checkAccess(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean | UrlTree> {
  const authService = inject(AuthService);
  const roleService = inject(RoleService);
  const permissionService = inject(PermissionService);
  const router = inject(Router);

  const cacheKey = state.url;
  
  if (cache.has(cacheKey)) {
    return of(cache.get(cacheKey)!);
  }

  return combineLatest([
    authService.isAuthenticated$,
    roleService.getUserRoles$,
    permissionService.getUserPermissions$
  ]).pipe(
    map(([isAuth, roles, permissions]) => {
      if (!isAuth) {
        return router.createUrlTree(['/login']);
      }

      const requiredRole = route.data['role'];
      const requiredPermission = route.data['permission'];

      if (requiredRole && !roles.includes(requiredRole)) {
        return router.createUrlTree(['/unauthorized']);
      }

      if (requiredPermission && !permissions.includes(requiredPermission)) {
        return router.createUrlTree(['/forbidden']);
      }

      cache.set(cacheKey, true);
      return true;
    })
  );
}

export const complexGuard: CanActivateFn = (route, state) => {
  return checkAccess(route, state);
};

export const complexChildGuard: CanActivateChildFn = (route, state) => {
  return checkAccess(route, state);
};
```

## Testing Functional Guards

### Basic Guard Testing

```typescript
// auth.guard.spec.ts
import { TestBed } from '@angular/core/testing';
import { Router, UrlTree } from '@angular/router';
import { authGuard } from './auth.guard';
import { AuthService } from './auth.service';

describe('authGuard', () => {
  let authService: jasmine.SpyObj<AuthService>;
  let router: jasmine.SpyObj<Router>;

  beforeEach(() => {
    const authServiceSpy = jasmine.createSpyObj('AuthService', ['isLoggedIn']);
    const routerSpy = jasmine.createSpyObj('Router', ['createUrlTree']);

    TestBed.configureTestingModule({
      providers: [
        { provide: AuthService, useValue: authServiceSpy },
        { provide: Router, useValue: routerSpy }
      ]
    });

    authService = TestBed.inject(AuthService) as jasmine.SpyObj<AuthService>;
    router = TestBed.inject(Router) as jasmine.SpyObj<Router>;
  });

  it('should allow access when user is logged in', () => {
    authService.isLoggedIn.and.returnValue(true);

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as any, {} as any)
    );

    expect(result).toBe(true);
  });

  it('should redirect to login when user is not logged in', () => {
    const urlTree = {} as UrlTree;
    authService.isLoggedIn.and.returnValue(false);
    router.createUrlTree.and.returnValue(urlTree);

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as any, { url: '/protected' } as any)
    );

    expect(result).toBe(urlTree);
    expect(router.createUrlTree).toHaveBeenCalledWith(
      ['/login'],
      { queryParams: { returnUrl: '/protected' } }
    );
  });
});
```

### Testing Async Guards

```typescript
// async-guard.spec.ts
import { TestBed } from '@angular/core/testing';
import { of } from 'rxjs';
import { asyncAuthGuard } from './async-auth.guard';
import { AuthService } from './auth.service';

describe('asyncAuthGuard', () => {
  let authService: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    const spy = jasmine.createSpyObj('AuthService', ['checkAuth']);

    TestBed.configureTestingModule({
      providers: [{ provide: AuthService, useValue: spy }]
    });

    authService = TestBed.inject(AuthService) as jasmine.SpyObj<AuthService>;
  });

  it('should return observable that emits true when authenticated', (done) => {
    authService.checkAuth.and.returnValue(of(true));

    const result = TestBed.runInInjectionContext(() =>
      asyncAuthGuard({} as any, {} as any)
    );

    if (result instanceof Observable) {
      result.subscribe(value => {
        expect(value).toBe(true);
        done();
      });
    }
  });
});
```

### Testing Guard Composition

```typescript
// composed-guard.spec.ts
import { TestBed } from '@angular/core/testing';
import { composeGuards } from './guard-composer';
import { CanActivateFn } from '@angular/router';

describe('composeGuards', () => {
  it('should return true when all guards pass', () => {
    const guard1: CanActivateFn = () => true;
    const guard2: CanActivateFn = () => true;
    const guard3: CanActivateFn = () => true;

    const composed = composeGuards(guard1, guard2, guard3);
    const result = composed({} as any, {} as any);

    expect(result).toBe(true);
  });

  it('should return false when any guard fails', () => {
    const guard1: CanActivateFn = () => true;
    const guard2: CanActivateFn = () => false;
    const guard3: CanActivateFn = () => true;

    const composed = composeGuards(guard1, guard2, guard3);
    const result = composed({} as any, {} as any);

    expect(result).toBe(false);
  });

  it('should stop at first failing guard', () => {
    const guard1 = jasmine.createSpy('guard1').and.returnValue(true);
    const guard2 = jasmine.createSpy('guard2').and.returnValue(false);
    const guard3 = jasmine.createSpy('guard3').and.returnValue(true);

    const composed = composeGuards(guard1, guard2, guard3);
    composed({} as any, {} as any);

    expect(guard1).toHaveBeenCalled();
    expect(guard2).toHaveBeenCalled();
    expect(guard3).not.toHaveBeenCalled();
  });
});
```

## Common Mistakes

### 1. Using inject() Outside Injection Context

```typescript
// WRONG - inject() called outside injection context
export const wrongGuard: CanActivateFn = (route, state) => {
  // This might be called outside injection context
  setTimeout(() => {
    const service = inject(SomeService); // Error!
  }, 1000);
  
  return true;
};

// CORRECT - inject() called immediately in guard function
export const correctGuard: CanActivateFn = (route, state) => {
  const service = inject(SomeService); // Correct
  
  setTimeout(() => {
    service.doSomething(); // Use injected service
  }, 1000);
  
  return true;
};
```

### 2. Not Handling All Return Types

```typescript
// WRONG - Only handles boolean
export const wrongGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.isLoggedIn(); // What about redirects?
};

// CORRECT - Returns UrlTree for redirect
export const correctGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  return authService.isLoggedIn()
    ? true
    : router.createUrlTree(['/login']);
};
```

### 3. Creating New Guard Instances

```typescript
// WRONG - Creating new function each time
export const routes: Routes = [
  {
    path: 'admin',
    canActivate: [() => inject(AuthService).isAdmin()] // New function each time!
  }
];

// CORRECT - Reusable guard function
export const adminGuard: CanActivateFn = (route, state) => {
  return inject(AuthService).isAdmin();
};

export const routes: Routes = [
  {
    path: 'admin',
    canActivate: [adminGuard]
  }
];
```

## Best Practices

### 1. Keep Guards Simple and Focused

```typescript
// Good: Single responsibility
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  return authService.isLoggedIn()
    ? true
    : router.createUrlTree(['/login']);
};

export const roleGuard = (role: string): CanActivateFn => (route, state) => {
  const roleService = inject(RoleService);
  return roleService.hasRole(role);
};

// Compose for complex logic
export const adminAccessGuard = composeGuards(
  authGuard,
  roleGuard('admin')
);
```

### 2. Use Type-Safe Guard Factories

```typescript
// guard-types.ts
export type GuardConfig = {
  roles?: string[];
  permissions?: string[];
  requireAll?: boolean;
};

export function createAccessGuard(config: GuardConfig): CanActivateFn {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (!authService.isLoggedIn()) {
      return router.createUrlTree(['/login']);
    }

    // Type-safe configuration
    if (config.roles) {
      const hasRole = config.roles.some(r => authService.hasRole(r));
      if (!hasRole) {
        return router.createUrlTree(['/unauthorized']);
      }
    }

    return true;
  };
}
```

### 3. Document Guard Behavior

```typescript
/**
 * Authentication guard that checks if user is logged in.
 * 
 * @returns true if authenticated, UrlTree to /login if not
 * 
 * @example
 * ```typescript
 * const routes: Routes = [
 *   {
 *     path: 'dashboard',
 *     canActivate: [authGuard],
 *     component: DashboardComponent
 *   }
 * ];
 * ```
 */
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  
  return authService.isLoggedIn()
    ? true
    : router.createUrlTree(['/login'], {
        queryParams: { returnUrl: state.url }
      });
};
```

### 4. Centralize Guard Exports

```typescript
// guards/index.ts
export { authGuard } from './auth.guard';
export { roleGuard } from './role.guard';
export { permissionGuard } from './permission.guard';
export { featureFlagGuard } from './feature-flag.guard';
export { composeGuards } from './guard-composer';

// Usage
import { authGuard, roleGuard, composeGuards } from './guards';
```

## Interview Questions

### Q1: What are functional guards and why were they introduced?

**Answer:** Functional guards are route guards implemented as plain functions using the `inject()` API. Introduced in Angular 14+, they provide better tree-shaking, less boilerplate, easier composition, and a more functional programming style compared to class-based guards.

### Q2: How do you inject dependencies in functional guards?

**Answer:** Use the `inject()` function from `@angular/core` within the guard function body. inject() must be called synchronously in the guard function, not in callbacks or promises.

### Q3: Can you compose multiple functional guards?

**Answer:** Yes, create a composer function that takes multiple guards and returns a new guard. The composer can run guards sequentially, in parallel, or conditionally based on your requirements.

### Q4: How do functional guards improve tree-shaking?

**Answer:** Functional guards are plain functions without class overhead. Unused guards are easier for bundlers to detect and remove, resulting in smaller bundle sizes.

### Q5: Can you convert class guards to functional guards?

**Answer:** Yes, extract the logic from the `canActivate` method into a standalone function and replace constructor injection with `inject()` calls.

### Q6: How do you test functional guards?

**Answer:** Use `TestBed.runInInjectionContext()` to provide an injection context for the guard, mock dependencies with `TestBed.configureTestingModule()`, and test the guard's return values.

### Q7: What are the limitations of inject() in guards?

**Answer:** `inject()` must be called synchronously within the injection context. It cannot be used in async callbacks, promises, or after the initial guard execution starts.

### Q8: How do you create reusable parameterized guards?

**Answer:** Create a factory function that accepts parameters and returns a `CanActivateFn`. The returned function can access parameters via closure while using `inject()` for dependencies.

## Key Takeaways

1. **Functional guards** use inject() instead of constructor injection
2. **Less boilerplate** compared to class-based guards
3. **Better tree-shaking** results in smaller bundles
4. **Easier composition** enables complex authorization logic
5. **inject() must be called synchronously** in the guard function
6. **Guards are reusable** across routes and lazy-loaded modules
7. **Type-safe factories** create parameterized guards
8. **Test with runInInjectionContext()** for proper DI setup
9. **Composition patterns** enable building complex guards from simple ones
10. **Migration is straightforward** from class to functional guards

## Resources

- [Angular Functional Guards Guide](https://angular.io/guide/router#functional-guards)
- [inject() API Documentation](https://angular.io/api/core/inject)
- [CanActivateFn API](https://angular.io/api/router/CanActivateFn)
- [Angular 14 Release Notes](https://blog.angular.io/angular-v14-is-now-available-391a6db736af)
- [Router Guards Tutorial](https://angular.io/guide/router-tutorial-toh#milestone-5-route-guards)
- [Testing Functional Guards](https://angular.io/guide/testing-components-scenarios#testing-guards)
