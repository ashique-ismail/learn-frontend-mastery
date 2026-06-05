# Standalone Components in Angular

## The Idea

**In plain English:** A standalone component is a self-contained building block of an Angular web page that carries its own list of tools and helpers it needs, so you never have to register it in a separate "group membership" file. Think of it as a Lego piece that comes with its own instruction sheet instead of relying on a shared one kept in a box somewhere else.

**Real-world analogy:** Imagine a food truck that prepares and serves its own complete menu without depending on a central restaurant kitchen. Each food truck lists exactly the equipment and ingredients it brought along.

- The food truck = the standalone component
- The equipment list on the truck = the `imports` array inside `@Component`
- The central restaurant kitchen (that old approach needed) = the NgModule

---

## Table of Contents
- [Introduction](#introduction)
- [What Are Standalone Components](#what-are-standalone-components)
- [Creating Standalone Components](#creating-standalone-components)
- [Importing Dependencies](#importing-dependencies)
- [Standalone Directives and Pipes](#standalone-directives-and-pipes)
- [Bootstrapping Standalone Applications](#bootstrapping-standalone-applications)
- [Routing with Standalone Components](#routing-with-standalone-components)
- [Migration from NgModules](#migration-from-ngmodules)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Resources](#resources)

## Introduction

Standalone components, introduced in Angular 14 and becoming the recommended approach in Angular 15+, represent a fundamental shift in how Angular applications are structured. They eliminate the need for NgModules in many cases, simplifying component architecture and reducing boilerplate code.

The standalone API allows components, directives, and pipes to be self-contained units that explicitly declare their dependencies, making the codebase more maintainable, easier to understand, and more tree-shakeable.

## What Are Standalone Components

Traditional Angular components must be declared in an NgModule:

```typescript
// Traditional approach (old way)
@Component({
  selector: 'app-user',
  template: '<p>{{ name }}</p>'
})
export class UserComponent {
  name = 'John';
}

@NgModule({
  declarations: [UserComponent],
  imports: [CommonModule],
  exports: [UserComponent]
})
export class UserModule {}
```

Standalone components don't require NgModules:

```typescript
// Standalone approach (new way)
@Component({
  selector: 'app-user',
  standalone: true,
  imports: [CommonModule],
  template: '<p>{{ name }}</p>'
})
export class UserComponent {
  name = 'John';
}
```

Key differences:
- `standalone: true` flag enables standalone mode
- Dependencies are imported directly in the component
- No NgModule wrapper needed
- Components can be imported directly where needed

## Creating Standalone Components

### Basic Standalone Component

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-greeting',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="greeting">
      <h2>Hello, {{ name }}!</h2>
      <p *ngIf="showMessage">Welcome to Angular Standalone Components</p>
    </div>
  `,
  styles: [`
    .greeting {
      padding: 20px;
      border: 1px solid #ccc;
      border-radius: 8px;
    }
  `]
})
export class GreetingComponent {
  name: string = 'Developer';
  showMessage: boolean = true;
}
```

### Using Angular CLI

Generate standalone components with the CLI:

```bash
# Generate a standalone component
ng generate component user --standalone

# Generate with inline template and styles
ng generate component user --standalone --inline-template --inline-style

# Short form
ng g c user --standalone
```

The CLI generates:

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-user',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './user.component.html',
  styleUrls: ['./user.component.css']
})
export class UserComponent {}
```

## Importing Dependencies

### Importing Other Standalone Components

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { UserProfileComponent } from './user-profile/user-profile.component';
import { UserStatsComponent } from './user-stats/user-stats.component';

@Component({
  selector: 'app-user-dashboard',
  standalone: true,
  imports: [
    CommonModule,
    UserProfileComponent,  // Import standalone component directly
    UserStatsComponent     // Import another standalone component
  ],
  template: `
    <div class="dashboard">
      <app-user-profile [userId]="userId"></app-user-profile>
      <app-user-stats [userId]="userId"></app-user-stats>
    </div>
  `
})
export class UserDashboardComponent {
  userId: string = '123';
}
```

### Importing Angular Modules

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { HttpClientModule } from '@angular/common/http';
import { ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-user-form',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule,           // For template-driven forms
    ReactiveFormsModule,   // For reactive forms
    HttpClientModule       // For HTTP requests
  ],
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="username" name="username" placeholder="Username">
      <button type="submit">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  username: string = '';
  
  onSubmit(): void {
    console.log('Username:', this.username);
  }
}
```

### Importing Third-Party Libraries

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { MatButtonModule } from '@angular/material/button';
import { MatCardModule } from '@angular/material/card';
import { MatIconModule } from '@angular/material/icon';

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [
    CommonModule,
    MatButtonModule,
    MatCardModule,
    MatIconModule
  ],
  template: `
    <mat-card>
      <mat-card-header>
        <mat-card-title>{{ product.name }}</mat-card-title>
      </mat-card-header>
      <mat-card-content>
        <p>{{ product.description }}</p>
        <p class="price">{{ product.price | currency }}</p>
      </mat-card-content>
      <mat-card-actions>
        <button mat-raised-button color="primary">
          <mat-icon>shopping_cart</mat-icon>
          Add to Cart
        </button>
      </mat-card-actions>
    </mat-card>
  `
})
export class ProductCardComponent {
  product = {
    name: 'Angular Book',
    description: 'Learn Angular from scratch',
    price: 39.99
  };
}
```

## Standalone Directives and Pipes

### Standalone Directive

```typescript
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  @Input() appHighlight: string = 'yellow';
  @Input() defaultColor: string = 'transparent';

  constructor(private el: ElementRef) {}

  @HostListener('mouseenter') onMouseEnter(): void {
    this.highlight(this.appHighlight);
  }

  @HostListener('mouseleave') onMouseLeave(): void {
    this.highlight(this.defaultColor);
  }

  private highlight(color: string): void {
    this.el.nativeElement.style.backgroundColor = color;
  }
}

// Usage in a component
@Component({
  selector: 'app-text-display',
  standalone: true,
  imports: [HighlightDirective],
  template: `
    <p appHighlight="lightblue">Hover over me!</p>
  `
})
export class TextDisplayComponent {}
```

### Standalone Pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 50, trail: string = '...'): string {
    if (!value) return '';
    return value.length > limit 
      ? value.substring(0, limit) + trail 
      : value;
  }
}

// Usage in a component
@Component({
  selector: 'app-article-preview',
  standalone: true,
  imports: [CommonModule, TruncatePipe],
  template: `
    <div class="preview">
      <h3>{{ article.title }}</h3>
      <p>{{ article.content | truncate:100 }}</p>
    </div>
  `
})
export class ArticlePreviewComponent {
  article = {
    title: 'Understanding Standalone Components',
    content: 'Standalone components are a new way to build Angular applications...'
  };
}
```

### Custom Structural Directive

```typescript
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appRepeat]',
  standalone: true
})
export class RepeatDirective {
  @Input() set appRepeat(times: number) {
    this.viewContainer.clear();
    for (let i = 0; i < times; i++) {
      this.viewContainer.createEmbeddedView(this.templateRef, {
        $implicit: i,
        index: i
      });
    }
  }

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}
}

// Usage
@Component({
  selector: 'app-list',
  standalone: true,
  imports: [CommonModule, RepeatDirective],
  template: `
    <div *appRepeat="5; let i = index" class="item">
      Item {{ i + 1 }}
    </div>
  `
})
export class ListComponent {}
```

## Bootstrapping Standalone Applications

### Traditional Bootstrap (with AppModule)

```typescript
// main.ts (old way)
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';

platformBrowserDynamic()
  .bootstrapModule(AppModule)
  .catch(err => console.error(err));
```

### Standalone Bootstrap (without AppModule)

```typescript
// main.ts (new way)
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent)
  .catch(err => console.error(err));
```

### Bootstrap with Providers

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    provideAnimations(),
    // Custom services
    { provide: API_URL, useValue: 'https://api.example.com' }
  ]
}).catch(err => console.error(err));
```

### Root Component

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
  `
})
export class AppComponent {
  title = 'My Standalone App';
}
```

## Routing with Standalone Components

### Defining Routes

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'home',
    pathMatch: 'full'
  },
  {
    path: 'home',
    loadComponent: () => 
      import('./home/home.component').then(m => m.HomeComponent)
  },
  {
    path: 'users',
    loadComponent: () =>
      import('./users/users.component').then(m => m.UsersComponent)
  },
  {
    path: 'users/:id',
    loadComponent: () =>
      import('./user-detail/user-detail.component').then(m => m.UserDetailComponent)
  },
  {
    path: '**',
    loadComponent: () =>
      import('./not-found/not-found.component').then(m => m.NotFoundComponent)
  }
];
```

### Lazy Loading with Children

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () =>
      import('./admin/admin.component').then(m => m.AdminComponent),
    children: [
      {
        path: 'dashboard',
        loadComponent: () =>
          import('./admin/dashboard/dashboard.component').then(m => m.DashboardComponent)
      },
      {
        path: 'users',
        loadComponent: () =>
          import('./admin/users/users.component').then(m => m.AdminUsersComponent)
      }
    ]
  }
];

// admin.component.ts
@Component({
  selector: 'app-admin',
  standalone: true,
  imports: [RouterOutlet, CommonModule],
  template: `
    <div class="admin-layout">
      <nav>
        <a routerLink="dashboard">Dashboard</a>
        <a routerLink="users">Users</a>
      </nav>
      <router-outlet></router-outlet>
    </div>
  `
})
export class AdminComponent {}
```

### Route Guards with Standalone

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isLoggedIn()) {
    return true;
  }

  return router.createUrlTree(['/login']);
};

// Using the guard
export const routes: Routes = [
  {
    path: 'profile',
    loadComponent: () =>
      import('./profile/profile.component').then(m => m.ProfileComponent),
    canActivate: [authGuard]
  }
];
```

### Route Resolvers

```typescript
// user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { UserService } from './user.service';

export const userResolver: ResolveFn<any> = (route, state) => {
  const userService = inject(UserService);
  const userId = route.paramMap.get('id');
  return userService.getUser(userId!);
};

// Using the resolver
export const routes: Routes = [
  {
    path: 'users/:id',
    loadComponent: () =>
      import('./user-detail/user-detail.component').then(m => m.UserDetailComponent),
    resolve: {
      user: userResolver
    }
  }
];

// user-detail.component.ts
@Component({
  selector: 'app-user-detail',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserDetailComponent implements OnInit {
  user: any;

  constructor(private route: ActivatedRoute) {}

  ngOnInit(): void {
    this.user = this.route.snapshot.data['user'];
  }
}
```

## Migration from NgModules

### Gradual Migration Strategy

You can mix standalone and NgModule-based components during migration:

```typescript
// Standalone component
@Component({
  selector: 'app-new-feature',
  standalone: true,
  imports: [CommonModule],
  template: '<p>New standalone feature</p>'
})
export class NewFeatureComponent {}

// Import standalone component in NgModule
@NgModule({
  declarations: [OldComponent],
  imports: [
    CommonModule,
    NewFeatureComponent  // Import standalone component
  ]
})
export class OldModule {}
```

### Converting Existing Components

Before (NgModule approach):

```typescript
// user.component.ts
@Component({
  selector: 'app-user',
  templateUrl: './user.component.html'
})
export class UserComponent {}

// user.module.ts
@NgModule({
  declarations: [UserComponent],
  imports: [CommonModule, FormsModule],
  exports: [UserComponent]
})
export class UserModule {}
```

After (Standalone approach):

```typescript
// user.component.ts
@Component({
  selector: 'app-user',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './user.component.html'
})
export class UserComponent {}

// No module file needed!
```

### Automated Migration

Use Angular CLI schematics to automate migration:

```bash
# Convert a single component
ng generate @angular/core:standalone --path=src/app/user/user.component.ts

# Convert entire application
ng generate @angular/core:standalone

# Remove unnecessary NgModules after conversion
ng generate @angular/core:standalone --remove-common-module
```

### Migration Checklist

1. Convert components to standalone one at a time
2. Update imports in component decorator
3. Remove component from NgModule declarations
4. Update routing to use loadComponent
5. Convert guards and resolvers to functional
6. Update bootstrap in main.ts
7. Remove unnecessary NgModules
8. Test thoroughly after each conversion

## Best Practices

### 1. Create Barrel Exports for Related Components

```typescript
// components/index.ts
export { UserCardComponent } from './user-card/user-card.component';
export { UserListComponent } from './user-list/user-list.component';
export { UserFormComponent } from './user-form/user-form.component';

// Usage
import { UserCardComponent, UserListComponent } from './components';
```

### 2. Use Shared Utilities for Common Imports

```typescript
// shared/common-imports.ts
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { RouterModule } from '@angular/router';

export const COMMON_IMPORTS = [
  CommonModule,
  FormsModule,
  RouterModule
] as const;

// Usage
@Component({
  selector: 'app-my-component',
  standalone: true,
  imports: [...COMMON_IMPORTS, OtherComponent],
  template: '...'
})
export class MyComponent {}
```

### 3. Lazy Load Feature Components

```typescript
// Always use lazy loading for routes
{
  path: 'feature',
  loadComponent: () =>
    import('./feature/feature.component').then(m => m.FeatureComponent)
}

// Instead of eager loading
{
  path: 'feature',
  component: FeatureComponent  // Avoid this
}
```

### 4. Keep Components Focused and Small

```typescript
// Good: Small, focused component
@Component({
  selector: 'app-user-avatar',
  standalone: true,
  imports: [CommonModule],
  template: `
    <img [src]="avatarUrl" [alt]="userName">
  `
})
export class UserAvatarComponent {
  @Input() avatarUrl: string = '';
  @Input() userName: string = '';
}

// Use composition
@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule, UserAvatarComponent],
  template: `
    <div class="card">
      <app-user-avatar [avatarUrl]="user.avatar" [userName]="user.name">
      </app-user-avatar>
      <h3>{{ user.name }}</h3>
    </div>
  `
})
export class UserCardComponent {
  @Input() user: any;
}
```

### 5. Use Functional Guards and Resolvers

```typescript
// Prefer functional approach
export const canActivateAdmin = () => {
  const authService = inject(AuthService);
  return authService.isAdmin();
};

// Over class-based approach
@Injectable()
export class AdminGuard implements CanActivate {
  constructor(private authService: AuthService) {}
  
  canActivate(): boolean {
    return this.authService.isAdmin();
  }
}
```

## Common Mistakes

### 1. Forgetting to Add standalone: true

```typescript
// Wrong: Missing standalone flag
@Component({
  selector: 'app-user',
  imports: [CommonModule],  // This will be ignored!
  template: '<p>User</p>'
})
export class UserComponent {}

// Correct
@Component({
  selector: 'app-user',
  standalone: true,  // Don't forget this!
  imports: [CommonModule],
  template: '<p>User</p>'
})
export class UserComponent {}
```

### 2. Not Importing CommonModule

```typescript
// Wrong: Using *ngIf without CommonModule
@Component({
  selector: 'app-message',
  standalone: true,
  template: `<p *ngIf="show">Message</p>`  // Error!
})
export class MessageComponent {
  show = true;
}

// Correct
@Component({
  selector: 'app-message',
  standalone: true,
  imports: [CommonModule],  // Add this!
  template: `<p *ngIf="show">Message</p>`
})
export class MessageComponent {
  show = true;
}
```

### 3. Circular Dependencies

```typescript
// Wrong: Circular imports
// user.component.ts
import { PostComponent } from './post.component';

@Component({
  standalone: true,
  imports: [PostComponent]
})
export class UserComponent {}

// post.component.ts
import { UserComponent } from './user.component';

@Component({
  standalone: true,
  imports: [UserComponent]  // Circular!
})
export class PostComponent {}

// Solution: Create a shared component or use a service
```

### 4. Mixing Bootstrap Methods

```typescript
// Wrong: Can't use both methods
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

bootstrapApplication(AppComponent);  // Standalone bootstrap
platformBrowserDynamic().bootstrapModule(AppModule);  // Module bootstrap

// Correct: Choose one approach
bootstrapApplication(AppComponent);
```

## Interview Questions

### Basic Questions

**Q1: What are standalone components in Angular?**

A: Standalone components are components that don't require NgModule declarations. They use the `standalone: true` flag and directly import their dependencies, making them self-contained and more straightforward to use.

**Q2: How do you create a standalone component?**

A: Add `standalone: true` to the @Component decorator and list dependencies in the `imports` array. Use `ng generate component name --standalone` with the CLI.

**Q3: Can you use standalone components in NgModule-based applications?**

A: Yes, standalone components can be imported directly into NgModule imports arrays, allowing gradual migration.

### Intermediate Questions

**Q4: How do you bootstrap a standalone application?**

A: Use `bootstrapApplication(AppComponent, config)` in main.ts instead of `platformBrowserDynamic().bootstrapModule()`. Provide services and configuration through the config object.

**Q5: What's the difference between importing a component in a module vs standalone component?**

A: In NgModules, components go in the declarations array, while imports are for modules. In standalone components, both components and modules go in the imports array.

**Q6: How do you implement lazy loading with standalone components?**

A: Use `loadComponent` in route configuration with dynamic imports: `loadComponent: () => import('./path').then(m => m.Component)`

### Advanced Questions

**Q7: How do you handle dependency injection in standalone components?**

A: Use providers in bootstrapApplication config, route providers, or component-level providers. Use inject() function in functional guards/resolvers.

**Q8: What are the benefits of standalone components over NgModules?**

A: Simpler architecture, reduced boilerplate, better tree-shaking, clearer dependencies, easier testing, more explicit imports, and simpler mental model.

**Q9: How do you share state between standalone components?**

A: Use services with providedIn: 'root', route-level providers, or signals. Same patterns as NgModule-based apps but with more explicit dependency management.

**Q10: Can you convert an existing Angular application to use standalone components?**

A: Yes, gradually. Angular provides schematics to automate conversion. Convert components one by one, update routing, convert guards/resolvers, and finally remove unnecessary NgModules.

## Resources

### Official Documentation
- [Standalone Components Guide](https://angular.dev/guide/components/standalone)
- [Standalone Migration Guide](https://angular.dev/reference/migrations/standalone)

### Articles and Tutorials
- [Getting Started with Standalone Components](https://blog.angular.io/standalone-components)
- [Migrating to Standalone Components](https://angular.io/guide/standalone-migration)

### Video Tutorials
- [Standalone Components Deep Dive](https://www.youtube.com/watch?v=standalone-deep-dive)
- [Building Apps with Standalone Components](https://www.youtube.com/watch?v=standalone-apps)

### Tools
- [Angular CLI Schematics](https://angular.dev/cli/generate) - Generate standalone components
- [Migration Schematics](https://angular.dev/cli/migrate) - Automate migration

### Community Resources
- [Angular Blog](https://blog.angular.dev) - Latest updates and best practices
- [Angular Discord](https://discord.gg/angular) - Community support
