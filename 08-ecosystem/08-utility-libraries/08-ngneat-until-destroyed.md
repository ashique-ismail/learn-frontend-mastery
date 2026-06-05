# @ngneat/until-destroyed

## The Idea

**In plain English:** `@ngneat/until-destroyed` is a tool for Angular apps that automatically stops listening to data streams when a page or component is no longer visible. A "subscription" is like signing up to receive updates — if you never cancel it, it keeps running and wastes memory even after you've navigated away.

**Real-world analogy:** Imagine you hire a newspaper delivery person to drop a paper at your door every morning. If you move out of the house but never cancel the subscription, newspapers keep piling up outside an empty home forever. `@ngneat/until-destroyed` is like an automatic cancellation contract — the moment you hand over your keys (the component is destroyed), the newspaper delivery stops on its own.

- The newspaper delivery = the RxJS Observable emitting data continuously
- You living in the house = the Angular component being active on screen
- Handing over the keys (moving out) = the component being destroyed
- The automatic cancellation contract = the `untilDestroyed()` operator

---

## What It Is

`@ngneat/until-destroyed` is a utility that simplifies RxJS subscription cleanup in Angular components. It provides the `untilDestroyed()` operator that automatically unsubscribes when the component/service is destroyed.

> **Angular 16+ note:** Angular now includes `takeUntilDestroyed()` as a built-in operator (`@angular/core/rxjs-interop`). For new projects on Angular 16+, use the built-in. `@ngneat/until-destroyed` remains relevant for projects below Angular 16.

```bash
npm install @ngneat/until-destroyed
```

---

## The Problem It Solves

Forgetting to unsubscribe from Observables causes memory leaks:

```ts
// ❌ Memory leak — subscription never cleaned up
@Component({})
export class LeakyComponent implements OnInit {
  ngOnInit() {
    interval(1000).subscribe(n => this.count = n);
    // When component destroys, the interval keeps running
  }
}
```

---

## Usage: `untilDestroyed` Operator

```ts
import { UntilDestroy, untilDestroyed } from '@ngneat/until-destroyed';

@UntilDestroy()
@Component({
  template: `{{ count }}`
})
export class CounterComponent implements OnInit {
  count = 0;

  ngOnInit() {
    interval(1000).pipe(
      untilDestroyed(this)  // automatically unsubscribes on destroy
    ).subscribe(n => this.count = n);

    // Multiple subscriptions, all cleaned up automatically
    this.userService.currentUser$.pipe(
      untilDestroyed(this)
    ).subscribe(user => this.user = user);

    this.route.params.pipe(
      untilDestroyed(this),
      switchMap(params => this.dataService.load(params['id']))
    ).subscribe(data => this.data = data);
  }
}
```

The `@UntilDestroy()` class decorator patches the component's `ngOnDestroy` to trigger cleanup. `untilDestroyed(this)` creates a `takeUntil` under the hood.

---

## Without a Component: Service Usage

```ts
@UntilDestroy()
@Injectable()
export class SessionService implements OnDestroy {
  private subscription: Subscription;

  constructor(private authService: AuthService) {
    // With @UntilDestroy(), no need to manually call subscription.unsubscribe()
    this.authService.tokenExpired$.pipe(
      untilDestroyed(this)
    ).subscribe(() => this.logout());
  }

  ngOnDestroy() {
    // @UntilDestroy() calls this automatically, but you can add extra cleanup here
  }
}
```

---

## Angular 16+ Built-in Alternative

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';

@Component({})
export class ModernComponent implements OnInit {
  private destroyRef = inject(DestroyRef); // optional explicit reference

  ngOnInit() {
    interval(1000).pipe(
      takeUntilDestroyed(this.destroyRef) // or just takeUntilDestroyed() in injection context
    ).subscribe(n => this.count = n);
  }
}

// Even simpler — call directly in constructor (injection context)
@Component({})
export class EvenSimplerComponent {
  count = 0;

  constructor() {
    interval(1000).pipe(
      takeUntilDestroyed() // no argument needed in injection context
    ).subscribe(n => this.count = n);
  }
}
```

---

## Comparison of Approaches

| Approach | Angular version | Decorator needed | Setup |
|---|---|---|---|
| `@ngneat/until-destroyed` | Any | Yes (`@UntilDestroy()`) | Install package |
| Angular built-in `takeUntilDestroyed()` | 16+ | No | No setup |
| Manual `Subject` + `takeUntil` | Any | No | Boilerplate per component |
| `async` pipe | Any | No | Only in templates |

### Manual Subject Pattern (No Library)

```ts
@Component({})
export class ManualComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    interval(1000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(n => this.count = n);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

## Common Interview Questions

**Q: When is `@ngneat/until-destroyed` still needed in Angular 16+?**
For projects that must support Angular < 16, or for teams that haven't migrated yet. Also has slightly more ergonomic API for service cleanup patterns.

**Q: What's the difference between `takeUntilDestroyed()` and `async` pipe?**
`async` pipe auto-unsubscribes but only works in templates. `takeUntilDestroyed()` works anywhere in a component/service — in `ngOnInit`, constructor, or any method. Use `async` pipe for template subscriptions, `takeUntilDestroyed` for subscriptions inside class code.

**Q: What happens if you forget `@UntilDestroy()` but still use `untilDestroyed(this)`?**
It throws a runtime error: "untilDestroyed operator cannot be used inside directives or components or other objects that are not decorated with UntilDestroy decorator". The decorator is required.
