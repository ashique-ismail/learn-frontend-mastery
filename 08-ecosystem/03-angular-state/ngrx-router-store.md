# NgRx Router Store

## What It Does

NgRx Router Store connects Angular's Router state to the NgRx store. Router navigation becomes a sequence of dispatched actions, and the current route params/data are available as selectors alongside your other store state.

---

## Setup

```bash
npm install @ngrx/router-store
```

```ts
// app.config.ts
import { provideRouterStore } from '@ngrx/router-store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideStore(),
    provideRouterStore(), // connects router to store
  ],
};
```

---

## Default Selectors

```ts
import { getSelectors } from '@ngrx/router-store';

const {
  selectCurrentRoute,         // current ActivatedRouteSnapshot
  selectFragment,             // URL fragment (#section)
  selectQueryParams,          // { q: 'search' }
  selectQueryParam,           // factory: selectQueryParam('q')
  selectRouteParams,          // { id: '123' }
  selectRouteParam,           // factory: selectRouteParam('id')
  selectRouteData,            // route.data object
  selectUrl,                  // '/users/123'
  selectTitle,                // page title
} = getSelectors();

// Usage in component
@Component({})
export class UserComponent {
  userId$ = this.store.select(selectRouteParam('id'));
  searchQuery$ = this.store.select(selectQueryParam('q'));
  currentUrl$ = this.store.select(selectUrl);

  constructor(private store: Store) {}
}
```

---

## Reacting to Navigation in Effects

```ts
import { ROUTER_NAVIGATION, RouterNavigationAction } from '@ngrx/router-store';

@Injectable()
export class UserEffects {
  // Load user data when navigating to /users/:id
  loadUserOnNavigation$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ROUTER_NAVIGATION),
      filter((action: RouterNavigationAction) =>
        action.payload.routerState.url.startsWith('/users/')
      ),
      map(action => {
        const id = action.payload.routerState.root.firstChild?.params['id'];
        return UserActions.loadUser({ id });
      })
    )
  );

  constructor(private actions$: Actions) {}
}
```

---

## Router Actions

```ts
import { ROUTER_NAVIGATION, ROUTER_NAVIGATED, ROUTER_CANCEL, ROUTER_ERROR } from '@ngrx/router-store';

// ROUTER_NAVIGATION: fired before navigation completes (guards can cancel)
// ROUTER_NAVIGATED: fired after successful navigation
// ROUTER_CANCEL: fired when navigation is cancelled (guard denied)
// ROUTER_ERROR: fired on navigation error

// Dispatch navigation from store
import { routerNavigatedAction } from '@ngrx/router-store';

// Or use the Router service to navigate (triggers ROUTER_NAVIGATION action automatically)
this.router.navigate(['/users', userId]);
// This dispatches ROUTER_NAVIGATION → ROUTER_NAVIGATED to the store
```

---

## Custom Serializer (Reduce State Size)

By default, the full ActivatedRouteSnapshot is stored — it's large. A custom serializer stores only what you need:

```ts
import { RouterStateSerializer } from '@ngrx/router-store';
import { RouterStateSnapshot } from '@angular/router';

export interface CustomRouterState {
  url: string;
  params: Record<string, string>;
  queryParams: Record<string, string>;
  data: Record<string, unknown>;
}

@Injectable()
export class CustomSerializer implements RouterStateSerializer<CustomRouterState> {
  serialize(routerState: RouterStateSnapshot): CustomRouterState {
    let route = routerState.root;
    while (route.firstChild) route = route.firstChild; // deepest active route

    return {
      url: routerState.url,
      params: route.params,
      queryParams: routerState.root.queryParams,
      data: route.data,
    };
  }
}

// Register in providers
provideRouterStore({
  serializer: CustomSerializer,
})
```

---

## Selectors with Custom State

```ts
import { createFeatureSelector, createSelector } from '@ngrx/store';

const selectRouter = createFeatureSelector<CustomRouterState>('router');

export const selectCurrentUserId = createSelector(
  selectRouter,
  router => router.params['id']
);

export const selectIsUserRoute = createSelector(
  selectRouter,
  router => router.url.startsWith('/users')
);
```

---

## Use Cases

**1. Load data based on route:**
```ts
// Effect: load user when navigating to user detail
ofType(ROUTER_NAVIGATED),
withLatestFrom(store.select(selectCurrentUserId)),
filter(([, userId]) => !!userId),
map(([, userId]) => UserActions.loadUser({ id: userId })),
```

**2. Combine route params with store:**
```ts
// Selector: get the currently-displayed user from the store
export const selectCurrentUser = createSelector(
  selectCurrentUserId,       // from router state
  selectUserEntities,        // from users feature state
  (id, entities) => id ? entities[id] : null
);
```

**3. Breadcrumbs from route data:**
```ts
export const selectBreadcrumbs = createSelector(
  selectCurrentRoute,
  route => route?.data?.['breadcrumbs'] ?? []
);
```

---

## Common Interview Questions

**Q: Why use NgRx Router Store instead of injecting ActivatedRoute?**
Uniformity: all state (including route state) is in the store, accessible via selectors, and time-travel debuggable. Testable: effects and selectors based on store actions are easier to test than code that injects ActivatedRoute. Reactive: combine route params with other store state in a single selector.

**Q: What's the performance benefit of a custom serializer?**
The default serializer stores the full ActivatedRouteSnapshot which includes component references and circular references — large in memory and in DevTools. A custom serializer stores only the data you need (url, params, queryParams), making DevTools fast and state management efficient.
