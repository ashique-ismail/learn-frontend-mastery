# switchMap vs mergeMap vs concatMap vs exhaustMap in Angular

## The Operators in Angular Context

These four flattening operators are the backbone of RxJS-based Angular code. They're used in Effects, HTTP chains, form interactions, and anywhere async operations chain together.

The question each operator answers: **"What do I do when the source emits a new value while the previous inner Observable is still running?"**

---

## Visual Summary

```
Source: --A----B------C-->
```

| Operator | Behavior | Output |
|---|---|---|
| `switchMap` | Cancel A, start B | A's result dropped if B arrives before A completes |
| `mergeMap` | Run A and B concurrently | Both A and B results emitted, possibly interleaved |
| `concatMap` | Wait for A to complete, then start B | A then B, strictly ordered |
| `exhaustMap` | Ignore B until A completes | B dropped if A is still running |

---

## `switchMap` — Latest Wins (Cancel Previous)

```ts
// NgRx Effect: search — cancel old request when new query arrives
loadSearchResults$ = createEffect(() =>
  this.actions$.pipe(
    ofType(SearchActions.queryChanged),
    debounceTime(300),
    switchMap(({ query }) =>
      this.searchService.search(query).pipe(
        map(results => SearchActions.resultsLoaded({ results })),
        catchError(() => of(SearchActions.searchFailed()))
      )
    )
  )
);

// Route-based data loading: user navigates away → cancel pending request
loadUser$ = createEffect(() =>
  this.actions$.pipe(
    ofType(ROUTER_NAVIGATION),
    switchMap(action =>
      this.userService.getUser(action.payload.routerState.params['id']).pipe(
        map(user => UserActions.loaded({ user })),
      )
    )
  )
);
```

---

## `mergeMap` — All Concurrent (Parallel)

```ts
// Upload multiple files in parallel — all at once
uploadFiles$ = createEffect(() =>
  this.actions$.pipe(
    ofType(FileActions.uploadSelected),
    mergeMap(({ files }) =>
      from(files).pipe(
        mergeMap(file =>
          this.fileService.upload(file).pipe(
            map(result => FileActions.uploadSuccess({ file, result })),
            catchError(err => of(FileActions.uploadFailure({ file, error: err })))
          )
        )
      )
    )
  )
);

// Analytics events — fire and forget, order doesn't matter
logEvent$ = createEffect(
  () =>
    this.actions$.pipe(
      ofType(AnalyticsActions.trackEvent),
      mergeMap(({ event }) => this.analytics.log(event))
    ),
  { dispatch: false }
);
```

---

## `concatMap` — Sequential (Queue)

```ts
// Process a queue of operations in order
processQueue$ = createEffect(() =>
  this.actions$.pipe(
    ofType(QueueActions.addOperation),
    concatMap(({ operation }) =>
      this.operationService.execute(operation).pipe(
        map(result => QueueActions.operationComplete({ result })),
        catchError(err => of(QueueActions.operationFailed({ error: err })))
      )
    )
  )
);

// Autosave: ensure saves don't overlap, process in order received
autoSave$ = createEffect(() =>
  this.actions$.pipe(
    ofType(DocumentActions.contentChanged),
    debounceTime(1000),
    concatMap(({ content }) => this.documentService.save(content))
  )
);
```

---

## `exhaustMap` — Ignore Until Done

```ts
// Login: prevent double-submission
login$ = createEffect(() =>
  this.actions$.pipe(
    ofType(AuthActions.loginAttempt),
    exhaustMap(({ credentials }) =>
      this.authService.login(credentials).pipe(
        map(user => AuthActions.loginSuccess({ user })),
        catchError(err => of(AuthActions.loginFailure({ error: err })))
      )
    )
  )
);

// Refresh token: ignore new refresh attempts while one is in progress
refreshToken$ = createEffect(() =>
  this.actions$.pipe(
    ofType(AuthActions.tokenExpired),
    exhaustMap(() =>
      this.authService.refreshToken().pipe(
        map(token => AuthActions.tokenRefreshed({ token })),
        catchError(() => of(AuthActions.logout()))
      )
    )
  )
);
```

---

## Angular-Specific Patterns

### Form + HTTP (switchMap)

```ts
// Reactive form → live search
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.api.search(query))
).subscribe(results => this.results = results);
```

### Route Params → HTTP (switchMap)

```ts
// Load entity based on route param — switch when param changes
this.route.params.pipe(
  map(params => params['id']),
  switchMap(id => this.userService.getUser(id))
).subscribe(user => this.user = user);
```

### Button Click → HTTP (exhaustMap)

```ts
// Save button: ignore clicks while save is in progress
fromEvent(this.saveButton.nativeElement, 'click').pipe(
  exhaustMap(() => this.service.save(this.form.value))
).subscribe(() => this.toast.success('Saved!'));
```

---

## Common Interview Questions

**Q: What happens to error handling with `switchMap`?**
If the `catchError` is inside the `switchMap` (on the inner observable), the outer stream remains alive. If `catchError` is on the outer stream (after `switchMap`), the whole stream terminates on error. Always put `catchError` inside the flattening operator.

**Q: Why not always use `mergeMap`?**
`mergeMap` runs all inner observables concurrently. For search, this causes race conditions (old results overwriting new). For saves, this causes out-of-order writes. Use the operator that matches your concurrency requirement.

**Q: What's the difference between `switchMap` and `concatMap` for a route param subscription?**
`switchMap` cancels the previous load when the route changes — correct behavior (we don't want old user data). `concatMap` would queue the loads — if you navigate quickly between 5 users, all 5 requests run in order, and you see users 1-4 before finally seeing user 5. Not ideal.
