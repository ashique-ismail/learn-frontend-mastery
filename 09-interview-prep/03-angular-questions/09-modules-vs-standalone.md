# NgModules vs Standalone Components

## The Idea

**In plain English:** Angular apps are made of building blocks called components (think of each as a mini screen or widget). NgModules were the old way of grouping those components together and telling Angular what each group needed to work; Standalone Components is the newer, simpler way where each component just lists its own needs directly, no group required.

**Real-world analogy:** Imagine a school where, in the old system, every student had to belong to a homeroom class, and the homeroom teacher was responsible for handing out all the textbooks each student needed. In the new system, every student carries their own backpack and packs exactly the books they personally need.

- The homeroom class = NgModule (a group that manages shared resources)
- The homeroom teacher listing textbooks = the `imports`/`declarations` array in `@NgModule`
- A student with their own backpack = a Standalone Component (self-contained, declares its own dependencies)

---

## NgModules (Legacy, Still Supported)

NgModules were Angular's original compilation and dependency scoping unit. Every component had to belong to exactly one module.

```ts
@NgModule({
  declarations: [UserListComponent, UserCardComponent], // components owned here
  imports: [CommonModule, HttpClientModule, SharedModule], // external deps
  exports: [UserListComponent],  // what other modules can use
  providers: [UserService],      // services scoped to this module
})
export class UserModule {}
```

**Problems:**
- Boilerplate — declare, import, and export every component
- Hard to understand which modules provide which dependencies
- Lazy loading required creating entire modules for single routes

---

## Standalone Components (Angular 14+, Default in Angular 17+)

Standalone components declare their own dependencies directly:

```ts
@Component({
  standalone: true,
  selector: 'app-user-list',
  template: `
    <app-user-card *ngFor="let u of users" [user]="u" />
  `,
  imports: [CommonModule, UserCardComponent], // direct imports
})
export class UserListComponent {
  users = inject(UserService).getUsers();
}
```

No module needed. The component itself knows what it needs.

---

## `bootstrapApplication` vs `platformBrowserDynamic`

```ts
// NgModule bootstrap
platformBrowserDynamic()
  .bootstrapModule(AppModule);

// Standalone bootstrap
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
    provideAnimations(),
  ]
});
```

---

## Lazy Loading with Standalone

```ts
// NgModule lazy loading (old)
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.module').then(m => m.UsersModule)
  }
];

// Standalone lazy loading (new) — load a single component
const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users/user-list.component').then(c => c.UserListComponent)
  }
];

// Standalone lazy loading with child routes
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/routes').then(r => r.USER_ROUTES)
  }
];
```

---

## Migration Path

```bash
# Automated migration (Angular 15.2+)
ng generate @angular/core:standalone
```

Three migration steps:
1. Convert all components/directives/pipes to standalone
2. Remove `declarations` from NgModules
3. Bootstrap as standalone app

---

## When NgModules Still Make Sense

- Large codebases mid-migration — don't break everything at once
- Libraries that need to group and export many components together
- Teams that need library-scoped providers (though `provideX()` functions largely replace this)

---

## Common Interview Questions

**Q: Can standalone components and NgModule components coexist?**
Yes. Standalone components can be used inside NgModules (add to `imports`), and NgModule components can be used inside standalone components (import the NgModule that declares them).

**Q: How do you provide services in a standalone app without NgModules?**
Use `providers` in `bootstrapApplication()` for root-level services, or `providers` on the route config for route-scoped services, or `providedIn: 'root'` on the service itself.

**Q: Do standalone components improve tree-shaking?**
Yes. Because standalone components declare their own imports, the compiler can more precisely determine what's used. Unused module imports no longer pull in everything declared in that module.
