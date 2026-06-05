# NgRx Effects

## The Idea

**In plain English:** NgRx Effects are background listeners in your app that watch for specific events (called actions) and then go do something outside the app's memory — like fetching data from a server, saving to storage, or navigating to a new page — and can report back with a new event when they're done.

**Real-world analogy:** Think of a hotel concierge desk. A guest (the app) drops a request note into a slot (dispatches an action). The concierge (the Effect) watches that slot, picks up notes matching their specialty, goes off to handle the task (makes a phone call, books a restaurant), and then slides a result note back to the front desk (dispatches a success or failure action).

- The guest dropping a note = the app dispatching an action
- The concierge watching the slot = the Effect listening to the action stream
- The concierge making a phone call = the side effect (API call, navigation, storage)
- The result note slid back = the new action dispatched after the task completes

---

## What Effects Do

Effects are where side effects live in the NgRx pattern — API calls, navigation, local storage, analytics. They listen to the action stream and optionally dispatch new actions.

```
Action dispatched → Effect listens → Side effect runs → New action dispatched (optional)
```

---

## Basic Effect

```ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { switchMap, map, catchError } from 'rxjs/operators';
import { of } from 'rxjs';
import { UserActions } from './user.actions';
import { UserService } from './user.service';

@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.loadUsers),         // filter: only react to this action
      switchMap(() =>                          // cancel previous, start new
        this.userService.getUsers().pipe(
          map(users => UserActions.loadUsersSuccess({ users })),
          catchError(error => of(UserActions.loadUsersFailure({ error })))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private userService: UserService
  ) {}
}
```

---

## Operator Choice Guide

| Operator | Behavior | Use for |
|---|---|---|
| `switchMap` | Cancels previous, uses latest | Search, navigation, GET requests where fresh data matters |
| `mergeMap` | Runs all concurrently | Independent operations that can overlap (analytics, logging) |
| `concatMap` | Queues and runs sequentially | Order-sensitive operations (optimistic updates) |
| `exhaustMap` | Ignores new until current completes | Login form, prevent double-submit |

```ts
// switchMap — search (cancel old requests when new query arrives)
search$ = createEffect(() =>
  this.actions$.pipe(
    ofType(SearchActions.query),
    debounceTime(300),
    distinctUntilChanged(({ query: a }, { query: b }) => a === b),
    switchMap(({ query }) =>
      this.searchService.search(query).pipe(
        map(results => SearchActions.resultsLoaded({ results })),
        catchError(error => of(SearchActions.searchFailed({ error })))
      )
    )
  )
);

// exhaustMap — login (ignore repeated clicks while processing)
login$ = createEffect(() =>
  this.actions$.pipe(
    ofType(AuthActions.login),
    exhaustMap(({ credentials }) =>
      this.authService.login(credentials).pipe(
        map(user => AuthActions.loginSuccess({ user })),
        catchError(error => of(AuthActions.loginFailure({ error })))
      )
    )
  )
);
```

---

## Non-Dispatching Effects

Effects don't have to dispatch an action. Use `{ dispatch: false }` for pure side effects:

```ts
// Navigation after login
navigateAfterLogin$ = createEffect(
  () =>
    this.actions$.pipe(
      ofType(AuthActions.loginSuccess),
      tap(() => this.router.navigate(['/dashboard']))
    ),
  { dispatch: false } // don't dispatch anything — just navigate
);

// Analytics tracking
trackEvent$ = createEffect(
  () =>
    this.actions$.pipe(
      ofType(CartActions.itemAdded),
      tap(({ item }) => this.analytics.track('add_to_cart', { item }))
    ),
  { dispatch: false }
);
```

---

## Error Handling Patterns

```ts
// CORRECT: catchError inside switchMap so the effect stream doesn't die
loadData$ = createEffect(() =>
  this.actions$.pipe(
    ofType(DataActions.load),
    switchMap(() =>
      this.dataService.getData().pipe(
        map(data => DataActions.loadSuccess({ data })),
        catchError(error => of(DataActions.loadFailure({ error: error.message })))
        // catchError here: catches errors from this inner observable only
        // Effect stream continues to listen for future DataActions.load
      )
    )
  )
);

// WRONG: catchError on the outer stream kills the effect
loadDataWrong$ = createEffect(() =>
  this.actions$.pipe(
    ofType(DataActions.load),
    switchMap(() => this.dataService.getData()),
    map(data => DataActions.loadSuccess({ data })),
    catchError(error => of(DataActions.loadFailure({ error })))
    // If error occurs, catchError handles it but the stream COMPLETES
    // Subsequent DataActions.load will never trigger this effect again!
  )
);
```

---

## Functional Effects (Angular 17+)

```ts
// No @Injectable class needed
export const loadUsers = createEffect(
  (actions$ = inject(Actions), userService = inject(UserService)) => {
    return actions$.pipe(
      ofType(UserActions.loadUsers),
      switchMap(() =>
        userService.getUsers().pipe(
          map(users => UserActions.loadUsersSuccess({ users })),
          catchError(error => of(UserActions.loadUsersFailure({ error })))
        )
      )
    );
  }
);

// Register in providers
bootstrapApplication(AppComponent, {
  providers: [
    provideStore(),
    provideEffects([loadUsers]),
  ]
});
```

---

## Testing Effects

```ts
it('should dispatch loadUsersSuccess on successful API call', () => {
  const users = [{ id: '1', name: 'Alice' }];
  const action = UserActions.loadUsers();
  const outcome = UserActions.loadUsersSuccess({ users });

  actions$ = hot('-a', { a: action });
  const response = cold('-b|', { b: users });
  userService.getUsers.mockReturnValue(response);

  const expected = cold('--c', { c: outcome });
  expect(effects.loadUsers$).toBeObservable(expected);
});
```

---

## Common Interview Questions

**Q: Why is `catchError` placement critical in effects?**
If `catchError` is on the outer stream (after `switchMap`), a caught error completes the outer stream. The effect stops listening for actions. Placing `catchError` inside the `switchMap` (on the inner observable) keeps the outer stream alive.

**Q: When would you use `mergeMap` vs `switchMap` in effects?**
`switchMap` is for operations where only the latest matters (search, navigation). `mergeMap` is for operations that are independent and must all complete (adding multiple items to a list, parallel API calls). `concatMap` when order matters.
