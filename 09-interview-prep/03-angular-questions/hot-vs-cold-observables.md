# Hot vs Cold Observables

## The Core Distinction

**Cold Observable**: The producer is created **inside** the observable. Each subscriber gets its own independent execution — own HTTP request, own timer, own data stream.

**Hot Observable**: The producer exists **outside** the observable. All subscribers share the same execution — same WebSocket, same DOM event, same Subject.

---

## Cold Observables (Default)

```ts
import { Observable, interval } from 'rxjs';

// Cold: each subscriber creates its own timer
const cold$ = interval(1000);

cold$.subscribe(v => console.log('A:', v)); // A: 0, A: 1, A: 2...
// 2 seconds later:
cold$.subscribe(v => console.log('B:', v)); // B: 0, B: 1, B: 2...
// A and B have SEPARATE, independent sequences starting from 0
```

```ts
// HTTP calls are cold — each subscription makes a new request
const users$ = this.http.get<User[]>('/api/users');

// ❌ This makes TWO separate HTTP requests
users$.subscribe(users => this.users = users);
users$.subscribe(users => console.log(users.length));
```

---

## Hot Observables

```ts
import { Subject, fromEvent } from 'rxjs';

// Subject is hot — subscribers share the same event stream
const subject = new Subject<number>();

subject.subscribe(v => console.log('A:', v));
subject.subscribe(v => console.log('B:', v));

subject.next(1); // A: 1, B: 1 (both get it simultaneously)
subject.next(2); // A: 2, B: 2

// fromEvent is hot — the DOM event exists independently
const clicks$ = fromEvent(document, 'click');
clicks$.subscribe(e => console.log('First handler', e));
clicks$.subscribe(e => console.log('Second handler', e));
// Both share the same click event
```

---

## Converting Cold to Hot: `share` and `shareReplay`

```ts
import { share, shareReplay } from 'rxjs/operators';

// share: multicasts to all current subscribers
const users$ = this.http.get<User[]>('/api/users').pipe(
  share() // ONE request, shared among all subscribers
);

// ✅ Only ONE HTTP request made
users$.subscribe(users => this.users = users);
users$.subscribe(users => console.log(users.length));

// shareReplay: multicasts AND replays last N values to new subscribers
const config$ = this.http.get('/api/config').pipe(
  shareReplay(1) // cache last value; late subscribers get it immediately
);

config$.subscribe(config => this.config = config); // makes request
config$.subscribe(config => console.log(config));   // gets cached value, no new request
// 5 seconds later:
config$.subscribe(config => console.log(config));   // still gets cached value
```

---

## Practical Comparison

| | Cold | Hot |
|---|---|---|
| Producer location | Inside observable | Outside observable |
| Subscribers | Each gets own execution | Share same execution |
| Examples | `of()`, `interval()`, `http.get()` | `Subject`, `fromEvent()`, `BehaviorSubject` |
| Late subscriber | Gets full sequence from start | Gets values from subscription point onward |
| Use case | Independent streams per subscriber | Shared event sources |

---

## BehaviorSubject — Hot + Replays Current Value

```ts
const currentUser$ = new BehaviorSubject<User | null>(null);

// Emit new values
currentUser$.next(loggedInUser);

// Subscribers always immediately receive the current value
currentUser$.subscribe(user => updateUI(user)); // gets current value immediately
```

`BehaviorSubject` is hot (producer outside) + replays 1 value (last emitted). Ideal for reactive state.

---

## The `share` vs `shareReplay` Decision

```ts
// share: multicast, no cache — late subscriber misses past values
const events$ = source$.pipe(share());

// shareReplay(1): multicast + cache last N values — late subscriber gets them
const config$ = http.get('/config').pipe(shareReplay(1));

// shareReplay({ bufferSize: 1, refCount: true })
// refCount: true — unsubscribes from source when no subscribers remain
//           false — keeps source alive even with 0 subscribers (potential memory leak)
```

---

## Common Interview Questions

**Q: Is `http.get()` cold or hot?**
Cold — each subscription triggers a new HTTP request. Use `shareReplay(1)` if multiple parts of your app need the same data without duplicate requests.

**Q: What's the difference between `Subject`, `BehaviorSubject`, and `ReplaySubject`?**
- `Subject` — hot, no buffer. Late subscribers miss past values.
- `BehaviorSubject(initial)` — hot, buffers the **last 1** value. Late subscribers get the current value.
- `ReplaySubject(n)` — hot, buffers the last **n** values. Late subscribers get up to n past values.

**Q: Can a cold observable become hot and back to cold?**
You can make a cold observable hot via `share()`/`shareReplay()`, but the original cold observable remains cold. The `share`/`shareReplay` wrapper is hot; the underlying observable is still cold if you subscribe to it directly.
