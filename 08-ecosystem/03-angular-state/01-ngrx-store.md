# NgRx Store - Redux-Inspired State Management for Angular

## The Idea

**In plain English:** NgRx Store is a way to keep all of your app's data in one central place so that every part of the app can read from it and update it in an organized, predictable way. Think of "state" as the current snapshot of everything your app knows — who is logged in, what items are in a cart, what the search results are.

**Real-world analogy:** Imagine a busy restaurant where every order goes through a single order board in the kitchen. Waiters never walk up to a chef and quietly whisper an order — they write a ticket (action), pin it to the board, the chef reads it and updates the order board (reducer), and every station (component) watches the board for what to prepare next.

- The order ticket = an action (a formal description of what needs to happen)
- The order board = the store (the single source of truth for all current orders)
- The chef updating the board = the reducer (the rule-follower that decides how the board changes)
- Each kitchen station watching the board = a selector (a focused view of just the data a component needs)

---

## Table of Contents

- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
- [Actions](#actions)
- [Reducers](#reducers)
- [Selectors](#selectors)
- [Effects](#effects)
- [Entity Adapter](#entity-adapter)
- [Advanced Features](#advanced-features)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

NgRx Store is a state management library for Angular applications, inspired by Redux. It provides a single source of truth for application state, making data flow predictable and easier to debug.

**Key Features:**
- Single source of truth
- Immutable state
- Action-based state changes
- RxJS integration
- Time-travel debugging
- TypeScript support

## Installation & Setup

```bash
# Install NgRx packages
npm install @ngrx/store @ngrx/effects @ngrx/entity @ngrx/store-devtools

# Or use Angular CLI schematics
ng add @ngrx/store@latest
ng add @ngrx/effects@latest
ng add @ngrx/entity@latest
ng add @ngrx/store-devtools@latest
```

### Basic Setup

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { provideStoreDevtools } from '@ngrx/store-devtools';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(),
    provideEffects(),
    provideStoreDevtools({
      maxAge: 25,
      logOnly: false,
    }),
  ],
};

// Or for module-based
// app.module.ts
import { StoreModule } from '@ngrx/store';
import { EffectsModule } from '@ngrx/effects';
import { StoreDevtoolsModule } from '@ngrx/store-devtools';

@NgModule({
  imports: [
    StoreModule.forRoot({}),
    EffectsModule.forRoot([]),
    StoreDevtoolsModule.instrument({
      maxAge: 25,
      logOnly: environment.production,
    }),
  ],
})
export class AppModule {}
```

## Core Concepts

### 1. State

Define your application state:

```typescript
// state/counter.state.ts
export interface CounterState {
  count: number;
  loading: boolean;
  error: string | null;
}

export const initialCounterState: CounterState = {
  count: 0,
  loading: false,
  error: null,
};

// state/app.state.ts
export interface AppState {
  counter: CounterState;
  users: UsersState;
  products: ProductsState;
}
```

### 2. Store Structure

Organize your store:

```typescript
// store/index.ts
import { ActionReducerMap } from '@ngrx/store';
import { counterReducer } from './counter/counter.reducer';
import { usersReducer } from './users/users.reducer';

export interface AppState {
  counter: CounterState;
  users: UsersState;
}

export const reducers: ActionReducerMap<AppState> = {
  counter: counterReducer,
  users: usersReducer,
};

// Register in app config
import { reducers } from './store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(reducers),
  ],
};
```

### 3. Using Store in Components

```typescript
// counter.component.ts
import { Component } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { increment, decrement, reset } from './store/counter/counter.actions';
import { selectCount, selectLoading } from './store/counter/counter.selectors';
import { AppState } from './store';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <h1>Count: {{ count$ | async }}</h1>
      <p *ngIf="loading$ | async">Loading...</p>
      
      <button (click)="increment()">Increment</button>
      <button (click)="decrement()">Decrement</button>
      <button (click)="reset()">Reset</button>
    </div>
  `,
})
export class CounterComponent {
  count$: Observable<number>;
  loading$: Observable<boolean>;

  constructor(private store: Store<AppState>) {
    this.count$ = this.store.select(selectCount);
    this.loading$ = this.store.select(selectLoading);
  }

  increment() {
    this.store.dispatch(increment());
  }

  decrement() {
    this.store.dispatch(decrement());
  }

  reset() {
    this.store.dispatch(reset());
  }
}
```

## Actions

### 1. Creating Actions

```typescript
// counter.actions.ts
import { createAction, props } from '@ngrx/store';

// Simple actions
export const increment = createAction('[Counter] Increment');
export const decrement = createAction('[Counter] Decrement');
export const reset = createAction('[Counter] Reset');

// Actions with payload
export const incrementBy = createAction(
  '[Counter] Increment By',
  props<{ amount: number }>()
);

export const setCount = createAction(
  '[Counter] Set Count',
  props<{ count: number }>()
);

// Async actions (with success/failure)
export const loadUsers = createAction('[Users] Load Users');
export const loadUsersSuccess = createAction(
  '[Users] Load Users Success',
  props<{ users: User[] }>()
);
export const loadUsersFailure = createAction(
  '[Users] Load Users Failure',
  props<{ error: string }>()
);
```

### 2. Action Groups

Organize related actions:

```typescript
// users.actions.ts
import { createActionGroup, props, emptyProps } from '@ngrx/store';
import { User } from './user.model';

export const UsersActions = createActionGroup({
  source: 'Users',
  events: {
    'Load Users': emptyProps(),
    'Load Users Success': props<{ users: User[] }>(),
    'Load Users Failure': props<{ error: string }>(),
    'Add User': props<{ user: User }>(),
    'Update User': props<{ user: User }>(),
    'Delete User': props<{ id: string }>(),
  },
});

// Usage
this.store.dispatch(UsersActions.loadUsers());
this.store.dispatch(UsersActions.addUser({ user }));
```

### 3. Action Creators

```typescript
// custom-action.ts
import { Action } from '@ngrx/store';

export const CUSTOM_ACTION = '[Feature] Custom Action';

export class CustomAction implements Action {
  readonly type = CUSTOM_ACTION;
  constructor(public payload: { data: string }) {}
}

// Usage
this.store.dispatch(new CustomAction({ data: 'value' }));
```

## Reducers

### 1. Creating Reducers

```typescript
// counter.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { increment, decrement, reset, incrementBy } from './counter.actions';
import { CounterState, initialCounterState } from './counter.state';

export const counterReducer = createReducer(
  initialCounterState,
  
  on(increment, (state) => ({
    ...state,
    count: state.count + 1,
  })),
  
  on(decrement, (state) => ({
    ...state,
    count: state.count - 1,
  })),
  
  on(reset, (state) => ({
    ...state,
    count: 0,
  })),
  
  on(incrementBy, (state, { amount }) => ({
    ...state,
    count: state.count + amount,
  }))
);
```

### 2. Handling Multiple Actions

```typescript
// users.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { UsersActions } from './users.actions';
import { UsersState, initialUsersState } from './users.state';

export const usersReducer = createReducer(
  initialUsersState,
  
  // Loading
  on(UsersActions.loadUsers, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),
  
  // Success
  on(UsersActions.loadUsersSuccess, (state, { users }) => ({
    ...state,
    users,
    loading: false,
  })),
  
  // Failure
  on(UsersActions.loadUsersFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),
  
  // Add user
  on(UsersActions.addUser, (state, { user }) => ({
    ...state,
    users: [...state.users, user],
  })),
  
  // Update user
  on(UsersActions.updateUser, (state, { user }) => ({
    ...state,
    users: state.users.map((u) => (u.id === user.id ? user : u)),
  })),
  
  // Delete user
  on(UsersActions.deleteUser, (state, { id }) => ({
    ...state,
    users: state.users.filter((u) => u.id !== id),
  }))
);
```

### 3. Reducer Composition

```typescript
// feature.reducer.ts
import { createReducer, on } from '@ngrx/store';

const dataReducer = createReducer(
  initialDataState,
  on(loadData, (state) => ({ ...state, loading: true })),
  on(loadDataSuccess, (state, { data }) => ({ ...state, data, loading: false }))
);

const uiReducer = createReducer(
  initialUIState,
  on(toggleSidebar, (state) => ({ ...state, sidebarOpen: !state.sidebarOpen }))
);

export const featureReducer = combineReducers({
  data: dataReducer,
  ui: uiReducer,
});
```

## Selectors

### 1. Creating Selectors

```typescript
// counter.selectors.ts
import { createSelector, createFeatureSelector } from '@ngrx/store';
import { CounterState } from './counter.state';
import { AppState } from '../app.state';

// Feature selector
export const selectCounterState = createFeatureSelector<CounterState>('counter');

// Simple selectors
export const selectCount = createSelector(
  selectCounterState,
  (state) => state.count
);

export const selectLoading = createSelector(
  selectCounterState,
  (state) => state.loading
);

export const selectError = createSelector(
  selectCounterState,
  (state) => state.error
);

// Composed selectors
export const selectIsPositive = createSelector(
  selectCount,
  (count) => count > 0
);

export const selectDoubleCount = createSelector(
  selectCount,
  (count) => count * 2
);
```

### 2. Parameterized Selectors

```typescript
// users.selectors.ts
import { createSelector } from '@ngrx/store';

export const selectUsersState = createFeatureSelector<UsersState>('users');

export const selectAllUsers = createSelector(
  selectUsersState,
  (state) => state.users
);

// Selector with parameter
export const selectUserById = (id: string) =>
  createSelector(selectAllUsers, (users) => users.find((user) => user.id === id));

// Usage in component
constructor(private store: Store) {
  this.user$ = this.store.select(selectUserById('123'));
}

// Factory function
export const selectUsersByRole = createSelector(
  selectAllUsers,
  (users: User[], props: { role: string }) =>
    users.filter((user) => user.role === props.role)
);

// Usage
this.admins$ = this.store.select(selectUsersByRole, { role: 'admin' });
```

### 3. Multiple Input Selectors

```typescript
// dashboard.selectors.ts
import { createSelector } from '@ngrx/store';

export const selectDashboardData = createSelector(
  selectUsers,
  selectProducts,
  selectOrders,
  (users, products, orders) => ({
    totalUsers: users.length,
    totalProducts: products.length,
    totalOrders: orders.length,
    revenue: orders.reduce((sum, order) => sum + order.total, 0),
  })
);
```

## Effects

### 1. Creating Effects

```typescript
// users.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { of } from 'rxjs';
import { map, catchError, switchMap } from 'rxjs/operators';
import { UsersService } from './users.service';
import { UsersActions } from './users.actions';

@Injectable()
export class UsersEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UsersActions.loadUsers),
      switchMap(() =>
        this.usersService.getUsers().pipe(
          map((users) => UsersActions.loadUsersSuccess({ users })),
          catchError((error) =>
            of(UsersActions.loadUsersFailure({ error: error.message }))
          )
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private usersService: UsersService
  ) {}
}

// Register effects
export const appConfig: ApplicationConfig = {
  providers: [
    provideEffects([UsersEffects]),
  ],
};
```

### 2. Effects with Side Effects

```typescript
// notification.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { tap } from 'rxjs/operators';
import { NotificationService } from './notification.service';
import { UsersActions } from './users.actions';

@Injectable()
export class NotificationEffects {
  // Non-dispatching effect
  showSuccessNotification$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(UsersActions.loadUsersSuccess),
        tap(() => this.notificationService.show('Users loaded successfully'))
      ),
    { dispatch: false } // Don't dispatch another action
  );

  showErrorNotification$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(UsersActions.loadUsersFailure),
        tap(({ error }) => this.notificationService.error(error))
      ),
    { dispatch: false }
  );

  constructor(
    private actions$: Actions,
    private notificationService: NotificationService
  ) {}
}
```

### 3. Effect Chaining

```typescript
// order.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { Store } from '@ngrx/store';
import { map, switchMap, withLatestFrom } from 'rxjs/operators';
import { OrderService } from './order.service';
import { OrderActions } from './order.actions';
import { selectCurrentUser } from './user.selectors';

@Injectable()
export class OrderEffects {
  createOrder$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrderActions.createOrder),
      withLatestFrom(this.store.select(selectCurrentUser)),
      switchMap(([action, user]) =>
        this.orderService.create({ ...action.order, userId: user.id }).pipe(
          map((order) => OrderActions.createOrderSuccess({ order }))
        )
      )
    )
  );

  // Chain effects
  createOrderSuccess$ = createEffect(() =>
    this.actions$.pipe(
      ofType(OrderActions.createOrderSuccess),
      map(() => OrderActions.loadOrders()) // Trigger another action
    )
  );

  constructor(
    private actions$: Actions,
    private store: Store,
    private orderService: OrderService
  ) {}
}
```

### 4. Concurrent Effects

```typescript
// search.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { debounceTime, switchMap, map } from 'rxjs/operators';
import { SearchActions } from './search.actions';
import { SearchService } from './search.service';

@Injectable()
export class SearchEffects {
  search$ = createEffect(() =>
    this.actions$.pipe(
      ofType(SearchActions.search),
      debounceTime(300), // Debounce user input
      switchMap(({ query }) => // Cancel previous requests
        this.searchService.search(query).pipe(
          map((results) => SearchActions.searchSuccess({ results }))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private searchService: SearchService
  ) {}
}
```

## Entity Adapter

### 1. Setup Entity Adapter

```typescript
// users.state.ts
import { EntityState, EntityAdapter, createEntityAdapter } from '@ngrx/entity';

export interface User {
  id: string;
  name: string;
  email: string;
}

export interface UsersState extends EntityState<User> {
  loading: boolean;
  error: string | null;
  selectedUserId: string | null;
}

export const usersAdapter: EntityAdapter<User> = createEntityAdapter<User>({
  selectId: (user) => user.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

export const initialUsersState: UsersState = usersAdapter.getInitialState({
  loading: false,
  error: null,
  selectedUserId: null,
});
```

### 2. Entity Reducer

```typescript
// users.reducer.ts
import { createReducer, on } from '@ngrx/store';
import { UsersActions } from './users.actions';
import { usersAdapter, initialUsersState } from './users.state';

export const usersReducer = createReducer(
  initialUsersState,
  
  // Load all
  on(UsersActions.loadUsersSuccess, (state, { users }) =>
    usersAdapter.setAll(users, { ...state, loading: false })
  ),
  
  // Add one
  on(UsersActions.addUserSuccess, (state, { user }) =>
    usersAdapter.addOne(user, state)
  ),
  
  // Add many
  on(UsersActions.addUsersSuccess, (state, { users }) =>
    usersAdapter.addMany(users, state)
  ),
  
  // Update one
  on(UsersActions.updateUserSuccess, (state, { user }) =>
    usersAdapter.updateOne({ id: user.id, changes: user }, state)
  ),
  
  // Update many
  on(UsersActions.updateUsersSuccess, (state, { users }) =>
    usersAdapter.updateMany(
      users.map(user => ({ id: user.id, changes: user })),
      state
    )
  ),
  
  // Remove one
  on(UsersActions.deleteUserSuccess, (state, { id }) =>
    usersAdapter.removeOne(id, state)
  ),
  
  // Remove many
  on(UsersActions.deleteUsersSuccess, (state, { ids }) =>
    usersAdapter.removeMany(ids, state)
  ),
  
  // Upsert (add or update)
  on(UsersActions.upsertUser, (state, { user }) =>
    usersAdapter.upsertOne(user, state)
  ),
  
  // Clear all
  on(UsersActions.clearUsers, (state) =>
    usersAdapter.removeAll(state)
  )
);
```

### 3. Entity Selectors

```typescript
// users.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { usersAdapter, UsersState } from './users.state';

export const selectUsersState = createFeatureSelector<UsersState>('users');

// Get default selectors
const { selectAll, selectEntities, selectIds, selectTotal } =
  usersAdapter.getSelectors(selectUsersState);

export const selectAllUsers = selectAll;
export const selectUserEntities = selectEntities;
export const selectUserIds = selectIds;
export const selectUsersTotal = selectTotal;

// Custom selectors
export const selectUsersLoading = createSelector(
  selectUsersState,
  (state) => state.loading
);

export const selectSelectedUserId = createSelector(
  selectUsersState,
  (state) => state.selectedUserId
);

export const selectSelectedUser = createSelector(
  selectUserEntities,
  selectSelectedUserId,
  (entities, selectedId) => (selectedId ? entities[selectedId] : null)
);

// Filter users
export const selectActiveUsers = createSelector(
  selectAllUsers,
  (users) => users.filter((user) => user.active)
);
```

## Advanced Features

### 1. Meta-Reducers

```typescript
// meta-reducers.ts
import { ActionReducer, MetaReducer } from '@ngrx/store';
import { AppState } from './app.state';

// Logging meta-reducer
export function logger(reducer: ActionReducer<any>): ActionReducer<any> {
  return (state, action) => {
    console.log('Previous State:', state);
    console.log('Action:', action);
    const nextState = reducer(state, action);
    console.log('Next State:', nextState);
    return nextState;
  };
}

// Clear state on logout
export function clearState(reducer: ActionReducer<AppState>): ActionReducer<AppState> {
  return (state, action) => {
    if (action.type === '[Auth] Logout') {
      state = undefined;
    }
    return reducer(state, action);
  };
}

export const metaReducers: MetaReducer<AppState>[] = [
  logger,
  clearState,
];

// Register
provideStore(reducers, { metaReducers });
```

### 2. Runtime Checks

```typescript
// app.config.ts
import { provideStore } from '@ngrx/store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(reducers, {
      runtimeChecks: {
        strictStateImmutability: true,
        strictActionImmutability: true,
        strictStateSerializability: true,
        strictActionSerializability: true,
        strictActionWithinNgZone: true,
        strictActionTypeUniqueness: true,
      },
    }),
  ],
};
```

### 3. Feature States

```typescript
// users/users.module.ts
import { NgModule } from '@angular/core';
import { StoreModule } from '@ngrx/store';
import { EffectsModule } from '@ngrx/effects';
import { usersReducer } from './store/users.reducer';
import { UsersEffects } from './store/users.effects';

@NgModule({
  imports: [
    StoreModule.forFeature('users', usersReducer),
    EffectsModule.forFeature([UsersEffects]),
  ],
})
export class UsersModule {}

// Or with standalone components
export const usersProviders = [
  provideState('users', usersReducer),
  provideEffects([UsersEffects]),
];
```

### 4. Router Store

```typescript
// Install
// npm install @ngrx/router-store

// app.config.ts
import { provideRouterStore, routerReducer } from '@ngrx/router-store';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({ router: routerReducer }),
    provideRouterStore(),
  ],
};

// router.selectors.ts
import { getRouterSelectors } from '@ngrx/router-store';

export const {
  selectCurrentRoute,
  selectQueryParams,
  selectRouteParams,
  selectRouteData,
  selectUrl,
} = getRouterSelectors();

// Usage in component
this.userId$ = this.store.select(selectRouteParams).pipe(
  map(params => params['id'])
);
```

## Performance Optimization

### 1. Memoized Selectors

```typescript
// Selectors are automatically memoized
export const selectExpensiveComputation = createSelector(
  selectAllUsers,
  selectAllOrders,
  (users, orders) => {
    // This computation only runs when inputs change
    return users.map(user => ({
      ...user,
      orders: orders.filter(o => o.userId === user.id),
      totalSpent: orders
        .filter(o => o.userId === user.id)
        .reduce((sum, o) => sum + o.total, 0),
    }));
  }
);
```

### 2. OnPush Change Detection

```typescript
// user-list.component.ts
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush, // Optimize
})
export class UserListComponent {
  users$ = this.store.select(selectAllUsers);

  constructor(private store: Store) {}
}
```

### 3. Normalize State

```typescript
// Instead of nested data
interface BadState {
  users: {
    id: string;
    orders: Order[]; // Nested
  }[];
}

// Use normalized structure
interface GoodState {
  users: EntityState<User>;
  orders: EntityState<Order>;
}

// Link via IDs
interface User {
  id: string;
  orderIds: string[]; // References
}
```

## Common Mistakes

### 1. Mutating State

```typescript
// ❌ Wrong - mutates state
on(addUser, (state, { user }) => {
  state.users.push(user); // Mutation!
  return state;
});

// ✅ Correct - immutable update
on(addUser, (state, { user }) => ({
  ...state,
  users: [...state.users, user],
}));
```

### 2. Not Using Selectors

```typescript
// ❌ Wrong - selecting directly
constructor(private store: Store<AppState>) {
  this.users$ = this.store.pipe(map(state => state.users.list));
}

// ✅ Correct - use selectors
constructor(private store: Store) {
  this.users$ = this.store.select(selectAllUsers);
}
```

### 3. Side Effects in Reducers

```typescript
// ❌ Wrong - side effects in reducer
on(loadUsers, (state) => {
  http.get('/users').subscribe(users => {
    // Side effect!
  });
  return state;
});

// ✅ Correct - use effects
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() => this.http.get('/users'))
  )
);
```

### 4. Not Handling Errors

```typescript
// ❌ Wrong - no error handling
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() => this.http.get('/users'))
  )
);

// ✅ Correct - handle errors
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() =>
      this.http.get('/users').pipe(
        map(users => loadUsersSuccess({ users })),
        catchError(error => of(loadUsersFailure({ error })))
      )
    )
  )
);
```

### 5. Subscribing in Effects

```typescript
// ❌ Wrong - manual subscription
loadUsers$ = createEffect(() => {
  this.actions$.pipe(ofType(loadUsers)).subscribe(() => {
    // Don't subscribe in effects!
  });
});

// ✅ Correct - return observable
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() => this.http.get('/users'))
  )
);
```

## Best Practices

### 1. Organize by Feature

```
src/
  app/
    users/
      store/
        users.actions.ts
        users.reducer.ts
        users.selectors.ts
        users.effects.ts
        users.state.ts
        index.ts
```

### 2. Use Action Groups

```typescript
export const UsersActions = createActionGroup({
  source: 'Users',
  events: {
    'Load Users': emptyProps(),
    'Load Users Success': props<{ users: User[] }>(),
    'Load Users Failure': props<{ error: string }>(),
  },
});
```

### 3. Type-Safe Actions

```typescript
// actions.ts
export const loadUsers = createAction('[Users] Load');

// effects.ts - typed action
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(loadUsers), // Type-safe
    switchMap(() => this.service.load())
  )
);
```

### 4. Facade Pattern

```typescript
// users.facade.ts
@Injectable({ providedIn: 'root' })
export class UsersFacade {
  users$ = this.store.select(selectAllUsers);
  loading$ = this.store.select(selectUsersLoading);
  selectedUser$ = this.store.select(selectSelectedUser);

  constructor(private store: Store) {}

  loadUsers(): void {
    this.store.dispatch(UsersActions.loadUsers());
  }

  selectUser(id: string): void {
    this.store.dispatch(UsersActions.selectUser({ id }));
  }
}

// Component uses facade
constructor(public usersFacade: UsersFacade) {}
```

### 5. Use DevTools

```typescript
// Enable Redux DevTools
provideStoreDevtools({
  maxAge: 25,
  logOnly: environment.production,
  autoPause: true,
  trace: true,
  traceLimit: 75,
});
```

## When to Use

### Use NgRx Store When

1. **Large applications** - Complex state management needs
2. **Multiple data sources** - APIs, WebSockets, local storage
3. **Shared state** - Data used across many components
4. **Time-travel debugging** - Need to debug state changes
5. **Predictable state** - Want strict unidirectional data flow

### Alternatives

- **NgRx Component Store** - Local component state
- **Akita** - Less boilerplate
- **NGXS** - Simpler API
- **Elf** - Lightweight alternative
- **Services + RxJS** - Simple state management

## Interview Questions

### 1. What is NgRx and why use it?

**Answer:** NgRx is a state management library for Angular based on Redux pattern. Benefits:
- Single source of truth for application state
- Predictable state updates through pure functions
- Time-travel debugging with DevTools
- RxJS integration for reactive programming
- Better testing through pure reducers and effects
- Enforces unidirectional data flow

### 2. Explain the NgRx data flow.

**Answer:**
1. Component dispatches an **Action**
2. **Reducer** receives action and returns new state
3. **Store** updates with new state
4. **Selectors** compute derived data
5. Components subscribe to selectors via observables
6. **Effects** handle side effects and dispatch new actions

### 3. What are selectors and why are they important?

**Answer:** Selectors are pure functions that compute derived state from the store. They're important because:
- **Memoization**: Cached results, only recompute when inputs change
- **Composability**: Build complex selectors from simple ones
- **Performance**: Reduce unnecessary computations
- **Testability**: Easy to unit test
- **Decoupling**: Components don't know about state structure

### 4. What is the purpose of Effects?

**Answer:** Effects handle side effects outside of reducers:
- API calls
- Navigation
- Logging
- External service interactions
- LocalStorage operations
- WebSocket connections

Reducers must be pure functions, so all side effects go in Effects.

### 5. Explain Entity Adapter.

**Answer:** EntityAdapter provides utilities for managing collections of entities:
- Predefined CRUD operations (addOne, updateOne, removeOne, etc.)
- Sorting and selecting by ID
- Normalized state structure
- Built-in selectors (selectAll, selectEntities, etc.)
- Reduces boilerplate code
- Optimized performance

```typescript
const adapter = createEntityAdapter<User>();
adapter.addOne(user, state);
adapter.updateMany(updates, state);
```

## Key Takeaways

1. **NgRx implements Redux pattern** for Angular with RxJS integration
2. **Actions describe what happened**, reducers describe how state changes
3. **Reducers must be pure functions** - no side effects, mutations, or async operations
4. **Effects handle side effects** like API calls and external service interactions
5. **Selectors compute derived state** with automatic memoization for performance
6. **Entity Adapter simplifies CRUD operations** for normalized collections
7. **Store is single source of truth** - all state changes flow through actions/reducers
8. **DevTools enable time-travel debugging** to inspect state history
9. **Use feature modules** to lazy load state slices
10. **Meta-reducers intercept actions** for cross-cutting concerns like logging

## Resources

### Official Documentation
- [NgRx Docs](https://ngrx.io/)
- [API Reference](https://ngrx.io/api)
- [Style Guide](https://ngrx.io/guide/eslint-plugin)

### Tutorials
- [NgRx Tutorial](https://ngrx.io/guide/store)
- [Angular University NgRx](https://angular-university.io/course/ngrx-course)

### Tools
- [Redux DevTools Extension](https://github.com/reduxjs/redux-devtools)
- [NgRx Schematics](https://ngrx.io/guide/schematics)

### Community
- [GitHub Repository](https://github.com/ngrx/platform)
- [Discord Community](https://discord.com/invite/ngrx)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/ngrx)
