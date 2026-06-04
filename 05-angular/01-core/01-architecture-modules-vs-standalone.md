# Angular Architecture: Modules vs Standalone Components

## Overview

Angular offers two architectural approaches: the traditional NgModule-based architecture and the modern standalone components introduced in Angular 14+. Understanding both approaches is crucial as the ecosystem transitions to standalone while maintaining backward compatibility with existing NgModule applications.

## Table of Contents

1. [NgModule-Based Architecture](#ngmodule-based-architecture)
2. [Standalone Components](#standalone-components)
3. [Bootstrapping Applications](#bootstrapping-applications)
4. [Dependency Injection Differences](#dependency-injection-differences)
5. [Lazy Loading](#lazy-loading)
6. [Migration Strategies](#migration-strategies)
7. [When to Use Each Approach](#when-to-use-each-approach)
8. [Hybrid Applications](#hybrid-applications)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)

## NgModule-Based Architecture

NgModules organize code into cohesive blocks of functionality with explicit imports and exports.

### Basic NgModule Structure

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { FormsModule } from '@angular/forms';

import { AppComponent } from './app.component';
import { HeaderComponent } from './header/header.component';
import { FooterComponent } from './footer/footer.component';
import { HomeComponent } from './home/home.component';

@NgModule({
  declarations: [
    AppComponent,
    HeaderComponent,
    FooterComponent,
    HomeComponent
  ],
  imports: [
    BrowserModule,
    HttpClientModule,
    FormsModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### NgModule Metadata

```typescript
@NgModule({
  // Components, directives, pipes declared in this module
  declarations: [
    MyComponent,
    MyDirective,
    MyPipe
  ],
  
  // Other modules whose exported classes are needed
  imports: [
    CommonModule,
    FormsModule,
    RouterModule
  ],
  
  // Services available to the entire application
  providers: [
    MyService,
    { provide: API_URL, useValue: 'https://api.example.com' }
  ],
  
  // Components, directives, pipes available to importing modules
  exports: [
    MyComponent,
    MyDirective,
    MyPipe,
    CommonModule  // Re-export modules
  ],
  
  // Entry component for root module
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### Feature Modules

```typescript
// users/users.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Routes } from '@angular/router';

import { UserListComponent } from './user-list/user-list.component';
import { UserDetailComponent } from './user-detail/user-detail.component';
import { UserFormComponent } from './user-form/user-form.component';
import { UserService } from './user.service';

const routes: Routes = [
  { path: '', component: UserListComponent },
  { path: ':id', component: UserDetailComponent }
];

@NgModule({
  declarations: [
    UserListComponent,
    UserDetailComponent,
    UserFormComponent
  ],
  imports: [
    CommonModule,
    RouterModule.forChild(routes)
  ],
  providers: [
    UserService  // Scoped to this module
  ]
})
export class UsersModule { }
```

### Shared Modules

```typescript
// shared/shared.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule, ReactiveFormsModule } from '@angular/forms';

import { LoadingSpinnerComponent } from './loading-spinner/loading-spinner.component';
import { ErrorMessageComponent } from './error-message/error-message.component';
import { HighlightDirective } from './directives/highlight.directive';
import { TruncatePipe } from './pipes/truncate.pipe';

@NgModule({
  declarations: [
    LoadingSpinnerComponent,
    ErrorMessageComponent,
    HighlightDirective,
    TruncatePipe
  ],
  imports: [
    CommonModule,
    FormsModule,
    ReactiveFormsModule
  ],
  exports: [
    // Export everything that should be available to importing modules
    LoadingSpinnerComponent,
    ErrorMessageComponent,
    HighlightDirective,
    TruncatePipe,
    // Re-export commonly used modules
    CommonModule,
    FormsModule,
    ReactiveFormsModule
  ]
})
export class SharedModule { }
```

### Core Module Pattern

```typescript
// core/core.module.ts
import { NgModule, Optional, SkipSelf } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HTTP_INTERCEPTORS } from '@angular/common/http';

import { AuthService } from './services/auth.service';
import { LoggerService } from './services/logger.service';
import { AuthInterceptor } from './interceptors/auth.interceptor';

@NgModule({
  imports: [CommonModule],
  providers: [
    AuthService,
    LoggerService,
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    }
  ]
})
export class CoreModule {
  // Prevent reimporting CoreModule
  constructor(@Optional() @SkipSelf() parentModule?: CoreModule) {
    if (parentModule) {
      throw new Error(
        'CoreModule is already loaded. Import it in AppModule only.'
      );
    }
  }
}
```

## Standalone Components

Standalone components don't require NgModules and manage their own dependencies.

### Basic Standalone Component

```typescript
// app.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterOutlet } from '@angular/router';

import { HeaderComponent } from './header/header.component';
import { FooterComponent } from './footer/footer.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [
    CommonModule,
    RouterOutlet,
    HeaderComponent,
    FooterComponent
  ],
  template: `
    <app-header></app-header>
    <main>
      <router-outlet></router-outlet>
    </main>
    <app-footer></app-footer>
  `,
  styles: [`
    main {
      min-height: calc(100vh - 120px);
      padding: 20px;
    }
  `]
})
export class AppComponent {
  title = 'My App';
}
```

### Standalone Component with Services

```typescript
// user-list/user-list.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import { UserCardComponent } from '../user-card/user-card.component';
import { LoadingSpinnerComponent } from '../shared/loading-spinner.component';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [
    CommonModule,
    UserCardComponent,
    LoadingSpinnerComponent
  ],
  template: `
    <div class="user-list">
      <h2>Users</h2>
      
      @if (loading()) {
        <app-loading-spinner></app-loading-spinner>
      }
      
      @if (error()) {
        <div class="error">{{ error() }}</div>
      }
      
      @if (users()) {
        <div class="users-grid">
          @for (user of users(); track user.id) {
            <app-user-card [user]="user"></app-user-card>
          }
        </div>
      }
    </div>
  `,
  styles: [`
    .users-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
      gap: 20px;
    }
  `]
})
export class UserListComponent implements OnInit {
  private http = inject(HttpClient);
  
  users = signal<User[]>([]);
  loading = signal(true);
  error = signal<string | null>(null);
  
  ngOnInit() {
    this.loadUsers();
  }
  
  private loadUsers() {
    this.http.get<User[]>('/api/users').subscribe({
      next: (users) => {
        this.users.set(users);
        this.loading.set(false);
      },
      error: (err) => {
        this.error.set('Failed to load users');
        this.loading.set(false);
      }
    });
  }
}
```

### Standalone Directives and Pipes

```typescript
// directives/highlight.directive.ts
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  @Input() appHighlight = 'yellow';
  
  constructor(private el: ElementRef) {}
  
  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.appHighlight);
  }
  
  @HostListener('mouseleave') onMouseLeave() {
    this.highlight('');
  }
  
  private highlight(color: string) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}

// pipes/truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 50, ellipsis = '...'): string {
    if (!value) return '';
    return value.length > limit
      ? value.substring(0, limit) + ellipsis
      : value;
  }
}

// Usage in standalone component
import { Component } from '@angular/core';
import { HighlightDirective } from './directives/highlight.directive';
import { TruncatePipe } from './pipes/truncate.pipe';

@Component({
  selector: 'app-card',
  standalone: true,
  imports: [HighlightDirective, TruncatePipe],
  template: `
    <div class="card" appHighlight="lightblue">
      <p>{{ description | truncate:100 }}</p>
    </div>
  `
})
export class CardComponent {
  description = 'Long description text...';
}
```

## Bootstrapping Applications

Different bootstrap methods for NgModule and standalone applications.

### NgModule Bootstrap

```typescript
// main.ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';

platformBrowserDynamic()
  .bootstrapModule(AppModule)
  .catch(err => console.error(err));

// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### Standalone Bootstrap

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { importProvidersFrom } from '@angular/core';

import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    // Import providers from modules if needed
    importProvidersFrom(BrowserAnimationsModule),
    // Application-level services
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
}).catch(err => console.error(err));
```

### Standalone with Custom Configuration

```typescript
// main.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';

import { authInterceptor } from './app/interceptors/auth.interceptor';
import { routes } from './app/app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding(),  // Enable route params as inputs
    ),
    provideHttpClient(
      withInterceptors([authInterceptor])
    ),
    provideAnimations(),
    // Global services
    AuthService,
    LoggerService
  ]
};

// main.ts
bootstrapApplication(AppComponent, appConfig);
```

## Dependency Injection Differences

How services are provided differs between architectures.

### NgModule Service Providers

```typescript
// app.module.ts
@NgModule({
  providers: [
    // Application-wide singleton
    UserService,
    
    // With configuration
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    },
    
    // Value provider
    { provide: API_URL, useValue: 'https://api.example.com' },
    
    // Factory provider
    {
      provide: LoggerService,
      useFactory: (config: AppConfig) => {
        return config.production ? new ProductionLogger() : new DevLogger();
      },
      deps: [AppConfig]
    }
  ]
})
export class AppModule { }

// Feature module with scoped service
@NgModule({
  providers: [FeatureService]  // New instance for this module
})
export class FeatureModule { }
```

### Standalone Service Providers

```typescript
// Tree-shakable singleton (preferred)
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  
  getUsers() {
    return this.http.get<User[]>('/api/users');
  }
}

// Provided in component
@Component({
  selector: 'app-user-form',
  standalone: true,
  providers: [FormValidationService]  // Scoped to component
})
export class UserFormComponent { }

// Provided at bootstrap
bootstrapApplication(AppComponent, {
  providers: [
    AuthService,
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
});

// Provided in route
export const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin.component'),
    providers: [AdminService]  // Available to this route and children
  }
];
```

## Lazy Loading

Both architectures support lazy loading with different syntax.

### NgModule Lazy Loading

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.module').then(m => m.UsersModule)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canActivate: [AuthGuard]
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Standalone Lazy Loading

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component')
      .then(m => m.HomeComponent)
  },
  {
    path: 'users',
    loadComponent: () => import('./users/user-list.component')
      .then(m => m.UserListComponent)
  },
  {
    path: 'users/:id',
    loadComponent: () => import('./users/user-detail.component')
      .then(m => m.UserDetailComponent)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
    canActivate: [authGuard]
  }
];

// admin/admin.routes.ts
import { Routes } from '@angular/router';

export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./admin-dashboard.component')
      .then(m => m.AdminDashboardComponent)
  },
  {
    path: 'users',
    loadComponent: () => import('./admin-users.component')
      .then(m => m.AdminUsersComponent)
  }
];
```

## Migration Strategies

Gradually migrate from NgModules to standalone components.

### Using Angular CLI Migration

```bash
# Migrate entire application
ng generate @angular/core:standalone

# Interactive migration prompts:
# 1. Convert declarations to standalone
# 2. Remove unnecessary NgModules
# 3. Bootstrap application with standalone API
```

### Manual Migration Steps

```typescript
// Step 1: Convert component to standalone
// Before (NgModule)
@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html'
})
export class UserCardComponent { }

// After (Standalone)
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './user-card.component.html'
})
export class UserCardComponent { }

// Step 2: Update module declarations
// Before
@NgModule({
  declarations: [UserCardComponent, UserListComponent],
  imports: [CommonModule]
})
export class UsersModule { }

// After (remove migrated component)
@NgModule({
  declarations: [UserListComponent],
  imports: [CommonModule, UserCardComponent]  // Import standalone
})
export class UsersModule { }

// Step 3: Eventually remove module entirely
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, UserCardComponent]
})
export class UserListComponent { }
```

### Gradual Migration Strategy

```typescript
// Phase 1: Leaf components first
// Convert components with no dependencies
@Component({
  selector: 'app-button',
  standalone: true,
  template: `<button><ng-content></ng-content></button>`
})
export class ButtonComponent { }

// Phase 2: Parent components
// Import standalone children
@Component({
  selector: 'app-form',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, ButtonComponent]
})
export class FormComponent { }

// Phase 3: Feature modules
// Convert entire features to standalone
export const FEATURE_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./feature.component')
  }
];

// Phase 4: Remove root module
// Update main.ts to use bootstrapApplication
```

## When to Use Each Approach

### Use NgModules When

```typescript
// Large existing applications
// - High migration cost
// - Stable NgModule architecture
// - Team familiar with NgModules

// Complex module hierarchies
@NgModule({
  imports: [
    CoreModule,
    SharedModule,
    FeatureAModule,
    FeatureBModule
  ]
})
export class AppModule { }

// Libraries targeting Angular < 14
// Need compatibility with older versions
```

### Use Standalone When

```typescript
// New applications (Angular 14+)
// Simpler architecture, less boilerplate

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes), provideHttpClient()]
});

// Component libraries
// Easier to import and use

// Small to medium applications
// Reduced complexity without modules

// Lazy loading individual components
// More granular code splitting
{
  path: 'feature',
  loadComponent: () => import('./feature.component')
}
```

## Hybrid Applications

Mix NgModules and standalone components during migration.

### Importing Standalone in NgModule

```typescript
// Standalone component
@Component({
  selector: 'app-new-feature',
  standalone: true,
  imports: [CommonModule]
})
export class NewFeatureComponent { }

// Import in NgModule
@NgModule({
  declarations: [LegacyComponent],
  imports: [
    CommonModule,
    NewFeatureComponent  // Import standalone component
  ]
})
export class LegacyModule { }
```

### Using NgModule Components in Standalone

```typescript
// NgModule with exported components
@NgModule({
  declarations: [LegacyComponent],
  exports: [LegacyComponent]
})
export class LegacyModule { }

// Import module in standalone component
@Component({
  selector: 'app-new-component',
  standalone: true,
  imports: [
    CommonModule,
    LegacyModule  // Import entire module
  ]
})
export class NewComponent { }
```

## Common Mistakes

### 1. Not Marking Components as Standalone

```typescript
// ❌ WRONG: Missing standalone: true
@Component({
  selector: 'app-button',
  template: `<button>Click</button>`
})
export class ButtonComponent { }

// ✅ CORRECT: Mark as standalone
@Component({
  selector: 'app-button',
  standalone: true,
  template: `<button>Click</button>`
})
export class ButtonComponent { }
```

### 2. Forgetting to Import Dependencies

```typescript
// ❌ WRONG: Using ngIf without CommonModule
@Component({
  selector: 'app-user',
  standalone: true,
  template: `
    @if (user) {
      <div>{{ user.name }}</div>
    }
  `
})
export class UserComponent { }

// ✅ CORRECT: Import CommonModule
@Component({
  selector: 'app-user',
  standalone: true,
  imports: [CommonModule],
  template: `
    @if (user) {
      <div>{{ user.name }}</div>
    }
  `
})
export class UserComponent { }
```

### 3. Mixing Module and Standalone Bootstrap

```typescript
// ❌ WRONG: Can't use both
platformBrowserDynamic().bootstrapModule(AppModule);
bootstrapApplication(AppComponent);

// ✅ CORRECT: Choose one approach
bootstrapApplication(AppComponent, appConfig);
```

## Best Practices

### 1. Prefer Standalone for New Code

```typescript
// ✅ Default to standalone
@Component({
  selector: 'app-new-feature',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule]
})
export class NewFeatureComponent { }
```

### 2. Use Tree-Shakable Providers

```typescript
// ✅ providedIn for better tree-shaking
@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);
}
```

### 3. Organize Standalone Routes

```typescript
// ✅ Group related routes in separate files
// app.routes.ts
export const routes: Routes = [
  { path: 'users', loadChildren: () => import('./users/users.routes') },
  { path: 'admin', loadChildren: () => import('./admin/admin.routes') }
];

// users/users.routes.ts
export const USERS_ROUTES: Routes = [
  { path: '', loadComponent: () => import('./user-list.component') },
  { path: ':id', loadComponent: () => import('./user-detail.component') }
];
```

### 4. Minimize Imports Array

```typescript
// ✅ Only import what you use
@Component({
  selector: 'app-simple',
  standalone: true,
  imports: [DatePipe],  // Only DatePipe, not all of CommonModule
  template: `{{ date | date:'short' }}`
})
export class SimpleComponent {
  date = new Date();
}
```

## Interview Questions

### Q1: What's the difference between NgModules and standalone components?

**Answer:** NgModules organize code into cohesive blocks with declarations, imports, and exports arrays. Standalone components (Angular 14+) are self-contained with `standalone: true` and manage their own dependencies in the imports array. Standalone components simplify the architecture by eliminating the need for NgModules in most cases.

### Q2: How do you bootstrap a standalone application?

**Answer:** Use `bootstrapApplication(AppComponent, config)` instead of `platformBrowserDynamic().bootstrapModule()`. The config object contains providers for routing, HTTP, and other services: `bootstrapApplication(AppComponent, { providers: [provideRouter(routes), provideHttpClient()] })`.

### Q3: Can you mix NgModules and standalone components?

**Answer:** Yes, during migration you can import standalone components into NgModules and vice versa. Import standalone components directly in the module's imports array, and import entire NgModules in a standalone component's imports array to use their exported declarations.

### Q4: How does lazy loading differ between architectures?

**Answer:** NgModules use `loadChildren` with module imports: `loadChildren: () => import('./module').then(m => m.Module)`. Standalone uses `loadComponent` for individual components or `loadChildren` for route arrays: `loadComponent: () => import('./component').then(m => m.Component)`.

### Q5: What happens to providers in standalone architecture?

**Answer:** Use `providedIn: 'root'` for tree-shakable services, provide in the bootstrapApplication config for app-wide services, or provide in individual components/routes for scoped services. The providers array moves from NgModule to component metadata or bootstrap config.

### Q6: Should you migrate existing NgModule apps to standalone?

**Answer:** It depends on app size and team capacity. Small apps benefit from simpler architecture. Large apps should migrate gradually using the CLI migration tool. New features can use standalone while maintaining legacy NgModules, allowing incremental migration over time.

## Key Takeaways

1. NgModules organize code with declarations, imports, exports
2. Standalone components are self-contained with `standalone: true`
3. Standalone simplifies architecture and reduces boilerplate
4. Bootstrap standalone apps with `bootstrapApplication()`
5. Both architectures support lazy loading
6. Mix NgModules and standalone during migration
7. Use `ng generate @angular/core:standalone` for migration
8. Prefer standalone for new applications (Angular 14+)
9. Tree-shakable providers with `providedIn: 'root'`
10. Standalone enables more granular lazy loading
11. Migration should be gradual: leaf components first
12. Standalone is the future of Angular architecture

## Resources

- [Angular Standalone Components Guide](https://angular.io/guide/standalone-components)
- [Migrating to Standalone](https://angular.io/guide/standalone-migration)
- [NgModules Guide](https://angular.io/guide/ngmodules)
- [Dependency Injection in Angular](https://angular.io/guide/dependency-injection)
- [Angular Router Documentation](https://angular.io/guide/router)
