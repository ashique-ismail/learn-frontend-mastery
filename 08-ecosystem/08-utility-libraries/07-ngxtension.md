# ngxtension

## The Idea

**In plain English:** ngxtension is a toolbox of ready-made helpers for Angular apps that saves you from writing the same repetitive setup code over and over. Think of it as a collection of shortcuts that handle common tasks — like watching for URL changes or tracking a value's history — in just one line instead of ten.

**Real-world analogy:** Imagine a professional kitchen where every chef has to sharpen their own knife, prep their own cutting board, and mix their own basic sauces from scratch before they can cook anything. ngxtension is like a prep station that arrives each morning with all those basics already done — knives sharpened, boards set out, sauces ready — so chefs can jump straight to cooking.

- The prep station = ngxtension (the library you install)
- The ready-made sauces and sharpened knives = utility functions like `injectQueryParams` and `derivedAsync`
- The chefs spending less time on setup = developers writing less boilerplate code

---

## What It Is

ngxtension is a collection of Angular utility functions, directives, and services that extend Angular's capabilities — especially for working with signals, RxJS interop, and common component patterns.

```bash
npm install ngxtension
```

---

## Signal Utilities

### `computedPrevious` — Access Previous Value

```ts
import { computedPrevious } from 'ngxtension/computed-previous';

export class TrackingComponent {
  count = signal(0);
  previousCount = computedPrevious(this.count);

  increment() { this.count.update(c => c + 1); }
}
// Template: Current: {{ count() }}, Previous: {{ previousCount() }}
```

### `derivedAsync` — Async Derived Signal

```ts
import { derivedAsync } from 'ngxtension/derived-async';

export class UserComponent {
  userId = input.required<string>();

  user = derivedAsync(() =>
    inject(UserService).getUser(this.userId())
  ); // returns Signal<User | undefined>

  // Template: {{ user()?.name }}
}
```

Like `toSignal` but re-runs when any signal dependency changes.

### `signalHistory` — Track Signal Changes

```ts
import { signalHistory } from 'ngxtension/signal-history';

export class FormComponent {
  formValue = signal({ name: '', email: '' });
  history = signalHistory(this.formValue, { limit: 10 });

  undo() { this.formValue.set(this.history.previous()); }
}
```

---

## RxJS ↔ Signal Utilities

### `rxMethod` — Effect with RxJS Operators

```ts
import { rxMethod } from 'ngxtension/rx-method';

export class SearchComponent {
  private searchService = inject(SearchService);

  // Creates a signal-reactive effect with RxJS operators
  search = rxMethod<string>(
    pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(query => this.searchService.search(query)),
      tap(results => this.results.set(results))
    )
  );

  query = signal('');

  constructor() {
    // Connect signal to the RxJS method
    this.search(this.query);
  }
}
```

### `createNotifier` — Trigger Observables from Signals

```ts
import { createNotifier } from 'ngxtension/create-notifier';

const refreshTrigger = createNotifier();

// In effect or wherever:
refreshTrigger.notify(); // triggers refresh

// In service:
data$ = refreshTrigger.pipe(
  switchMap(() => this.http.get('/api/data'))
);
```

---

## Component Utilities

### `hostBinding` — Reactive Host Bindings

```ts
import { hostBinding } from 'ngxtension/host-binding';

@Component({ selector: 'app-card' })
export class CardComponent {
  disabled = input(false);

  // Reactively binds to the host element
  @HostBinding('class.disabled')
  isDisabled = hostBinding(() => this.disabled());
}
```

### `injectQueryParams` — Type-Safe Query Params

```ts
import { injectQueryParams } from 'ngxtension/inject-query-params';

@Component({})
export class SearchPage {
  // Returns a Signal<string> that updates when query param changes
  query = injectQueryParams('q');
  page = injectQueryParams('page', { transform: Number });
}
// Template: {{ query() }} — no manual ActivatedRoute subscription
```

### `injectRouteData` — Route Data as Signal

```ts
import { injectRouteData } from 'ngxtension/inject-route-data';

@Component({})
export class ProtectedPage {
  routeData = injectRouteData<{ permissions: string[] }>();
  permissions = computed(() => this.routeData()?.permissions ?? []);
}
```

---

## Directives

### `*ngxRepeat` — Repeat N Times

```ts
import { NgxRepeat } from 'ngxtension/repeat';

@Component({
  imports: [NgxRepeat],
  template: `
    @for (i of 5 | ngxRepeat; track i) {
      <skeleton-row />
    }
  `
})
class LoadingState {}
```

### `ngxConnectForm` — Connect FormGroup to Signal

```ts
import { injectForm } from 'ngxtension/form';

export class ProfileForm {
  form = injectForm({
    defaultValues: { name: '', email: '' },
  });

  // form.values is a Signal<{name: string, email: string}>
  // form.valid is a Signal<boolean>
  // Auto-cleanup on destroy
}
```

---

## `takeUntilDestroyed` (Angular built-in, ngxtension-inspired)

Angular 16+ includes `takeUntilDestroyed()` as a built-in operator, inspired by ngxtension's equivalent:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({})
export class MyComponent {
  constructor() {
    someObservable$.pipe(
      takeUntilDestroyed()
    ).subscribe(data => this.data = data);
    // Automatically unsubscribes when component is destroyed
  }
}
```

---

## Common Interview Questions

**Q: What makes ngxtension different from using Angular built-ins directly?**
ngxtension provides utilities for patterns that Angular supports but requires boilerplate for — like `derivedAsync` (async computed signals), `injectQueryParams` (type-safe query params as signals), and `rxMethod` (bridging RxJS operators to signals). It reduces common patterns from 10+ lines to 1-2.

**Q: Is ngxtension stable for production?**
Yes — maintained by the Angular community with contributions from Angular core contributors. It's widely used in production Angular apps. Check version compatibility with your Angular version before upgrading.

**Q: How does `rxMethod` differ from `toSignal`?**
`toSignal` converts an Observable into a Signal (read-only). `rxMethod` creates a reactive "method" that accepts a signal or observable as input, runs it through RxJS operators, and executes effects — it's primarily for side effects with operator chains (debouncing, switching, etc.).
