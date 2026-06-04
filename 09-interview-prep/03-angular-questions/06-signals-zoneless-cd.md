# Signals and Zoneless Change Detection

## The Problem with Zone.js-Based Change Detection

Traditional Angular change detection:
1. Zone.js patches all async APIs
2. Any async operation (click, HTTP, setTimeout) triggers `ApplicationRef.tick()`
3. Angular walks the entire component tree checking for changes
4. Even unrelated components get checked (with Default CD strategy)

This is fine for small apps but inefficient at scale.

---

## How Signals Enable Fine-Grained Change Detection

Signals build a **reactive graph** of dependencies:

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1>{{ title() }}</h1>       <!-- depends on title signal -->
    <p>{{ count() }}</p>          <!-- depends on count signal -->
    <p>{{ doubleCount() }}</p>    <!-- depends on computed(count) -->
  `
})
export class DashboardComponent {
  title = signal('Dashboard');
  count = signal(0);
  doubleCount = computed(() => this.count() * 2);

  increment() {
    this.count.update(c => c + 1);
    // Angular knows ONLY components using count need to re-render
    // title is unaffected — its component view is skipped
  }
}
```

When `count` is updated, Angular knows exactly which template expressions depend on it (via the reactive graph built during rendering) and re-renders **only those specific views**.

---

## Zoneless Setup (Angular 17+)

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection(),
  ]
});
```

Remove Zone.js from `angular.json`:
```json
"polyfills": ["@angular/localize/init"]
// Removed: "zone.js"
```

---

## What Changes in Zoneless Mode

Without Zone.js, Angular doesn't automatically know about async operations. Change detection is only triggered by:

1. **Signal updates** — `signal.set()`, `signal.update()`, `signal.mutate()`
2. **Async pipe** — uses `ChangeDetectorRef.markForCheck()` internally
3. **Manual notification** — `ChangeDetectorRef.markForCheck()` or `ApplicationRef.tick()`

```ts
// Works great with signals (zoneless-native)
@Component({
  template: `{{ count() }}`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class Counter {
  count = signal(0);
  increment() { this.count.update(c => c + 1); } // triggers CD automatically
}

// Needs attention without Zone.js: setTimeout
export class TimerComponent implements OnInit {
  time = signal('');

  ngOnInit() {
    // Without Zone.js, this won't trigger CD
    // setInterval(() => { this.time.set(new Date().toTimeString()); }, 1000);

    // Solution 1: Use signal (works because signal mutation triggers CD)
    setInterval(() => {
      this.time.set(new Date().toTimeString()); // signal → triggers CD
    }, 1000);

    // Solution 2: Manual markForCheck
    // this.cdr.markForCheck();
  }
}
```

---

## Migration Checklist

1. **Audit event handlers** — wrap non-signal state updates in signal mutations or `markForCheck()`
2. **Convert BehaviorSubjects to signals** where possible
3. **Check third-party libraries** — those using `setTimeout`/Promises without signals may need wrapping
4. **Add `ChangeDetectionStrategy.OnPush`** to all components (required for zoneless to work well)
5. **Replace `async` pipe** with `toSignal()` where possible

---

## Performance Comparison

| Strategy | CD triggers | Checks per trigger | Optimal for |
|---|---|---|---|
| Default + Zone.js | Every async op | All components | Small apps |
| OnPush + Zone.js | Async + input changes | Only marked subtrees | Medium apps |
| OnPush + Signals + Zone.js | Signal mutations + async | Only signal consumers | Large apps |
| Zoneless + Signals | Signal mutations only | Only signal consumers | Most efficient |

---

## Common Interview Questions

**Q: Can you go zoneless without signals?**
Technically yes, but it's very painful. You'd need to manually call `markForCheck()` or `ApplicationRef.tick()` after every async operation. Signals provide the reactive graph that makes zoneless practical.

**Q: What's `ChangeDetectionStrategy.OnPush` and why is it required for zoneless?**
OnPush tells Angular to only check a component when its input references change, when an Observable/signal it depends on emits, or when manually marked dirty. Without OnPush, Angular would need to check all components on every tick — but in zoneless mode, there are no automatic ticks, so OnPush is essential for the signals-driven approach to work.

**Q: Is zoneless stable in Angular 18?**
The API is `provideExperimentalZonelessChangeDetection()` — still experimental as of Angular 18. Most of the ecosystem (Angular Material, etc.) works with it, but some edge cases remain. Angular 19 is expected to stabilize it.

**Q: Do signals work in Zone.js mode too?**
Yes. Signals improve change detection efficiency even with Zone.js — you get fine-grained updates within the existing Zone.js-triggered CD cycles. Zoneless is the next step beyond that.
