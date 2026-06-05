# map, mergeMap, switchMap, concatMap — Differences

## The Idea

**In plain English:** RxJS map operators are tools for transforming a stream of events or data, where each incoming value triggers some work — and the key difference is what happens when new work arrives before the previous work has finished. A "stream" is just a sequence of values that arrive over time, like messages in a chat.

**Real-world analogy:** Imagine a busy diner where a cashier shouts out food orders to a single cook. When a new order comes in while the cook is still preparing the previous one, different diners have different policies:

- The cashier (source stream) = the Observable emitting new values
- Each food order (a new emission) = a new inner task triggered by the source
- The cook's policy for handling overlapping orders = the map operator you choose (switch, merge, concat, or exhaust)

---

## The Core Question: What Happens When the Source Emits While an Inner Observable Is Still Running?

All four operators transform each source emission using a function. The difference is **how they handle concurrency** — what happens when a new source value arrives before the previous inner observable has completed.

---

## `map` — Synchronous Transformation (No Inner Observable)

```ts
import { of } from 'rxjs';
import { map } from 'rxjs/operators';

of(1, 2, 3).pipe(
  map(x => x * 10)
).subscribe(console.log);
// 10, 20, 30
```

`map` is not a flattening operator — it transforms each value synchronously into another value (not an Observable). Used for simple transformations.

---

## `switchMap` — Cancel Previous, Use Latest

When a new source emission arrives, **cancel the previous inner observable** and start a new one.

```ts
import { fromEvent, interval } from 'rxjs';
import { switchMap, take } from 'rxjs/operators';

// Search: cancel previous request when user types new query
searchInput$.pipe(
  debounceTime(300),
  switchMap(query => this.http.get(`/search?q=${query}`))
).subscribe(results => showResults(results));
```

Timeline:
```
Source:   --A--------B---C----->
Inner A:  ----a1--a2--a3|
Inner B:          --b1--b2|
Inner C:              --c1--c2|

Output:   ----a1--a2---b1---c1--c2
          (A's a3 cancelled when B arrives, B cancelled when C arrives)
```

**Use for:** Search, navigation, any "latest wins" scenario where old results are stale.

---

## `mergeMap` (= `flatMap`) — Run All Concurrently

All inner observables run **in parallel**. All values from all inner observables are emitted.

```ts
// Download multiple files in parallel
fileIds$.pipe(
  mergeMap(id => this.fileService.download(id))  // all downloads concurrent
).subscribe(file => processFile(file));
```

Timeline:
```
Source:   --A--------B---C----->
Inner A:  ----a1--a2--a3|
Inner B:          --b1--b2|
Inner C:              --c1--c2|

Output:   ----a1--a2--b1-a3-c1-b2-c2
          (all run simultaneously, interleaved as they emit)
```

**Use for:** Independent parallel operations (uploading multiple files, analytics events). Order not guaranteed.

---

## `concatMap` — Queue and Run Sequentially

Each inner observable must **complete before starting the next**. They are queued in order.

```ts
// Process tasks sequentially — each must finish before next starts
tasks$.pipe(
  concatMap(task => this.taskService.process(task))  // one at a time, in order
).subscribe(result => handleResult(result));

// Save operations in order (prevents overwriting)
saveActions$.pipe(
  concatMap(data => this.api.save(data))
).subscribe();
```

Timeline:
```
Source:   --A----B---C----->
Inner A:  ----a1--a2|
Inner B:            --b1--b2|   (waited for A)
Inner C:                  --c1| (waited for B)

Output:   ----a1--a2--b1--b2--c1
          (strict order, no overlap)
```

**Use for:** Order-dependent operations, sequential saves, ensuring previous completes before next starts.

---

## `exhaustMap` — Ignore New While Busy

If an inner observable is still running, **ignore new source emissions** until it completes.

```ts
// Login: ignore repeated clicks while a login attempt is in progress
loginButton$.pipe(
  exhaustMap(() => this.authService.login(credentials))
).subscribe(user => redirect(user));
```

Timeline:
```
Source:   --A----B---C----->
Inner A:  ----a1--a2|
(B ignored while A is active)
Inner C:             --c1|   (C starts because A is done)

Output:   ----a1--a2--c1
```

**Use for:** Preventing duplicate submissions, login/logout buttons, one-at-a-time user-triggered operations.

---

## Summary Table

| Operator | New source arrives while inner running | Order | Use for |
|---|---|---|---|
| `switchMap` | **Cancel** previous, start new | Latest | Search, navigation |
| `mergeMap` | **Run concurrently** with existing | Unordered | Parallel independent ops |
| `concatMap` | **Queue** after current | FIFO | Sequential, order matters |
| `exhaustMap` | **Ignore** until current completes | First wins | Prevent duplicate calls |

---

## Common Interview Questions

**Q: What happens if you use `mergeMap` for search?**
All requests run simultaneously and complete in unpredictable order. If the user types "react" then "angular", the "react" results might arrive after "angular" results, displaying stale data. Use `switchMap` for search.

**Q: What happens if you use `switchMap` for saving user data?**
If the user triggers two saves quickly, the first save request is cancelled mid-flight. Data loss or partial saves can occur. Use `concatMap` or `exhaustMap` for save operations.

**Q: When would you use `mergeMap` with a concurrency limit?**
`mergeMap(fn, 3)` — the second argument limits concurrent subscriptions to 3. Useful for downloading many files without overwhelming the server with hundreds of parallel requests.
