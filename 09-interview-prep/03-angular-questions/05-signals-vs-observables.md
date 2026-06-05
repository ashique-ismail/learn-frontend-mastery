# Signals vs Observables — When to Use What

## The Idea

**In plain English:** Signals and Observables are two ways a web app keeps track of changing information. A Signal is like a scoreboard that always shows the current score — you can check it at any moment. An Observable is like a live commentary feed — it tells you about events as they happen over time, one after another.

**Real-world analogy:** Imagine a sports stadium. The scoreboard updates instantly whenever a goal is scored, and anyone glancing at it sees the current score right away. Meanwhile, a radio commentator broadcasts a play-by-play stream of every event as they unfold throughout the match.

- The scoreboard = a Signal (always holds the current value, readable instantly)
- The radio broadcast stream = an Observable (a sequence of events delivered over time)
- Glancing at the scoreboard = reading a Signal synchronously with `mySignal()`
- Tuning in to the radio commentary = subscribing to an Observable to receive future events

---

## Core Difference

| | Signals | Observables (RxJS) |
|---|---|---|
| Current value | Always have one (synchronous read) | No current value (async stream) |
| Multiple values | No — just the current state | Yes — stream of events over time |
| Lazy | No — eager | Yes — cold observables are lazy |
| Completion | Never "complete" | Can complete or error |
| Primary use | Reactive state | Async event streams |

---

## Signals — Synchronous Reactive State

```ts
import { signal, computed, effect } from '@angular/core';

@Component({
  template: `<p>{{ fullName() }}</p>`,
})
export class ProfileComponent {
  firstName = signal('Alice');
  lastName = signal('Smith');

  // Automatically recomputes when dependencies change
  fullName = computed(() => `${this.firstName()} ${this.lastName()}`);

  // Runs as a side effect when signals change
  constructor() {
    effect(() => {
      console.log('Name changed:', this.fullName());
    });
  }

  updateName() {
    this.firstName.set('Bob'); // triggers recompute + effect
  }
}
```

**Signal reads are synchronous** — `signal()` always has a value you can read right now.

---

## Observables — Async Event Streams

```ts
@Component({
  template: `<div *ngFor="let msg of messages$ | async">{{ msg }}</div>`,
})
export class ChatComponent implements OnInit {
  messages$: Observable<Message[]>;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.messages$ = this.http.get<Message[]>('/api/messages').pipe(
      retry(3),
      catchError(() => of([])),
    );
  }
}
```

Observables model values *over time* — they may emit zero, one, or many values, and may complete or error.

---

## Interoperability

```ts
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

@Component({})
export class SearchComponent {
  // Convert Observable to Signal (for template use without async pipe)
  searchResults = toSignal(
    this.searchQuery$.pipe(
      debounceTime(300),
      switchMap(q => this.searchService.search(q))
    ),
    { initialValue: [] }
  );

  // Convert Signal to Observable (to use RxJS operators)
  userChange$ = toObservable(this.currentUser);
  
  // React to user changes with RxJS operators
  userData$ = this.userChange$.pipe(
    switchMap(user => this.api.getUserData(user.id))
  );
}
```

---

## Decision Guide

**Use Signals when:**
- Holding and deriving UI state (form values, selected items, toggle states)
- Replacing `BehaviorSubject` for simple reactive state
- You want synchronous reads in templates
- Zoneless change detection (signals work natively without Zone.js)

```ts
// Signals — ideal for local UI state
isOpen = signal(false);
selectedTab = signal<'overview' | 'details'>('overview');
items = signal<Item[]>([]);
count = computed(() => this.items().length);
```

**Use Observables when:**
- HTTP requests (single async operation)
- WebSocket / SSE streams (continuous event source)
- Complex async workflows (debounce, switchMap, retry)
- Combining multiple async sources (combineLatest, forkJoin)
- Time-based operations (interval, timer)

```ts
// RxJS — ideal for async operations
searchResults$ = this.searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(query => this.api.search(query)),
  catchError(() => of([]))
);

// WebSocket stream
messages$ = webSocket('wss://api.example.com/chat').pipe(
  retryWhen(errors => errors.pipe(delay(1000))),
);
```

---

## Signal vs BehaviorSubject

Before signals, `BehaviorSubject` was used for reactive state:

```ts
// Old way
class UserStore {
  private user$ = new BehaviorSubject<User | null>(null);
  readonly currentUser$ = this.user$.asObservable();
  setUser(user: User) { this.user$.next(user); }
  get currentUser() { return this.user$.getValue(); } // synchronous read
}

// New way
class UserStore {
  user = signal<User | null>(null);
  // Read: this.user() — always synchronous
  // Write: this.user.set(newUser)
  // Derive: computed(() => this.user()?.name)
}
```

Signals are simpler for state; observables are still better for streams.

---

## Common Interview Questions

**Q: Can signals replace RxJS completely?**
For state management, largely yes. For async operations (HTTP, WebSocket, complex operators), no — RxJS remains the right tool. The `toSignal`/`toObservable` interop bridges them cleanly.

**Q: What's the signal equivalent of `BehaviorSubject`?**
`signal()` — it has a current value, can be read synchronously, and emits changes to dependents. The main difference: signals track fine-grained dependencies; `BehaviorSubject` requires manual subscription management.

**Q: Do signals work with `async` pipe?**
No — `async` pipe is for Observables/Promises. Signals are read directly in templates with `()` call syntax: `{{ mySignal() }}`.
