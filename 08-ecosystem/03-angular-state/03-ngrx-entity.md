# NgRx Entity

## What It Solves

Managing collections of entities (users, products, orders) requires repetitive CRUD state patterns. NgRx Entity provides adapters that generate these operations and selectors automatically.

---

## Setup

```ts
import { createEntityAdapter, EntityAdapter, EntityState } from '@ngrx/entity';

export interface User {
  id: string;
  name: string;
  email: string;
}

// Define the state shape (includes EntityState: ids[] + entities{})
export interface UsersState extends EntityState<User> {
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

// Create adapter (optionally configure selectId and sortComparer)
export const adapter: EntityAdapter<User> = createEntityAdapter<User>({
  selectId: (user) => user.id,        // which field is the ID (default: 'id')
  sortComparer: (a, b) => a.name.localeCompare(b.name), // sorted order
});

// Initial state uses adapter.getInitialState()
export const initialState: UsersState = adapter.getInitialState({
  loading: false,
  error: null,
  selectedId: null,
});
```

---

## Reducer with CRUD Operations

```ts
import { createReducer, on } from '@ngrx/store';
import { UserActions } from './user.actions';

export const usersReducer = createReducer(
  initialState,

  on(UserActions.loadUsers, state => ({ ...state, loading: true, error: null })),

  on(UserActions.loadUsersSuccess, (state, { users }) =>
    adapter.setAll(users, { ...state, loading: false })
  ),

  on(UserActions.loadUsersFailure, (state, { error }) => ({
    ...state, loading: false, error
  })),

  // CRUD operations via adapter
  on(UserActions.addUser, (state, { user }) =>
    adapter.addOne(user, state)
  ),
  on(UserActions.updateUser, (state, { id, changes }) =>
    adapter.updateOne({ id, changes }, state)
  ),
  on(UserActions.deleteUser, (state, { id }) =>
    adapter.removeOne(id, state)
  ),
  on(UserActions.upsertUser, (state, { user }) =>
    adapter.upsertOne(user, state) // add if missing, update if exists
  ),
);
```

---

## All Adapter Methods

```ts
// Add
adapter.addOne(entity, state)
adapter.addMany(entities, state)

// Set (replace)
adapter.setOne(entity, state)
adapter.setAll(entities, state)        // replaces entire collection
adapter.setMany(entities, state)

// Update (partial)
adapter.updateOne({ id, changes }, state)
adapter.updateMany([{ id, changes }, ...], state)

// Upsert (add or update)
adapter.upsertOne(entity, state)
adapter.upsertMany(entities, state)

// Remove
adapter.removeOne(id, state)
adapter.removeMany(ids, state)
adapter.removeAll(state)

// Map (transform each entity)
adapter.map(entity => ({ ...entity, verified: true }), state)
```

---

## Selectors

```ts
// adapter.getSelectors() generates the standard selectors
const { selectIds, selectEntities, selectAll, selectTotal } = adapter.getSelectors();

// Feature selectors
import { createFeatureSelector, createSelector } from '@ngrx/store';

const selectUsersState = createFeatureSelector<UsersState>('users');

// Base entity selectors (scoped to feature state)
const {
  selectAll: selectAllUsers,
  selectEntities: selectUserEntities,
  selectIds: selectUserIds,
  selectTotal: selectUserCount,
} = adapter.getSelectors(selectUsersState);

// Custom selectors
export const selectSelectedUser = createSelector(
  selectUsersState,
  selectUserEntities,
  (state, entities) => state.selectedId ? entities[state.selectedId] : null
);

export const selectUsersLoading = createSelector(
  selectUsersState,
  state => state.loading
);

export const selectActiveUsers = createSelector(
  selectAllUsers,
  users => users.filter(u => u.active)
);
```

---

## Usage in Component

```ts
@Component({
  template: `
    @if (loading()) {
      <spinner />
    } @else {
      @for (user of users(); track user.id) {
        <user-card [user]="user" (delete)="deleteUser(user.id)" />
      }
    }
  `
})
export class UserListComponent {
  users = this.store.selectSignal(selectAllUsers);
  loading = this.store.selectSignal(selectUsersLoading);

  constructor(private store: Store) {
    store.dispatch(UserActions.loadUsers());
  }

  deleteUser(id: string) {
    this.store.dispatch(UserActions.deleteUser({ id }));
  }
}
```

---

## Custom `selectId` for Non-Standard IDs

```ts
// If your entity uses 'userId' instead of 'id'
const adapter = createEntityAdapter<User>({
  selectId: user => user.userId,
});

// If your entity uses composite keys
const adapter = createEntityAdapter<OrderItem>({
  selectId: item => `${item.orderId}-${item.productId}`,
});
```

---

## Common Interview Questions

**Q: What's the structure of EntityState?**
`{ ids: string[], entities: { [id: string]: T } }`. `ids` maintains order; `entities` provides O(1) lookup. `selectAll` combines them: `ids.map(id => entities[id])`.

**Q: Why use `setAll` vs `addMany`?**
`setAll` replaces the entire collection (use after full data reload). `addMany` adds to existing (use for pagination or incremental loading). Using `addMany` when you meant `setAll` leads to stale data remaining.

**Q: Does the adapter preserve insertion order?**
If no `sortComparer` is provided, entities appear in insertion order. With `sortComparer`, entities are re-sorted after each change. This means all selectors return sorted data automatically.
