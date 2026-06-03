# Zone.js — What It Does

## The Problem It Solves

Angular needs to know when to check for changes and re-render. Without Zone.js, you'd have to manually call `detectChanges()` after every async operation.

```ts
// Without Zone.js — manual change detection
fetchUser().then(user => {
  this.user = user;
  this.cdr.detectChanges(); // required!
});
```

Zone.js makes this automatic.

---

## How It Works: Monkey-Patching

Zone.js intercepts async browser APIs by replacing the native implementations with Zone-aware wrappers at app startup:

```js
// Zone.js internally does this (simplified):
const nativeSetTimeout = window.setTimeout;
window.setTimeout = function(fn, delay, ...args) {
  return nativeSetTimeout(() => {
    // Run in the Zone context
    Zone.current.run(fn, undefined, args);
    // Notify Angular when done
    zone.onMicrotaskEmpty.next();
  }, delay, ...args);
};
```

**APIs that Zone.js patches:**
- `setTimeout`, `setInterval`, `clearTimeout`, `clearInterval`
- `Promise.then`, `Promise.catch`
- `XMLHttpRequest` (and therefore `fetch`)
- `addEventListener`, `removeEventListener`
- `requestAnimationFrame`
- Node.js async APIs when running server-side

---

## The Angular Change Detection Trigger

```
Async operation completes (setTimeout, XHR, Promise)
    ↓
Zone.js notifies NgZone
    ↓
NgZone emits onMicrotaskEmpty
    ↓
ApplicationRef.tick() is called
    ↓
Full component tree change detection runs (top to bottom)
```

---

## `NgZone.runOutsideAngular()`

Use this for performance-sensitive code that doesn't affect the UI:

```ts
constructor(private ngZone: NgZone) {}

ngAfterViewInit() {
  this.ngZone.runOutsideAngular(() => {
    // This requestAnimationFrame won't trigger change detection
    const animate = () => {
      this.canvas.draw();
      requestAnimationFrame(animate);
    };
    requestAnimationFrame(animate);
  });
}

// Trigger change detection manually when needed
updateScore(score: number) {
  this.ngZone.run(() => {
    this.score = score; // now triggers CD
  });
}
```

---

## The Problem with Zone.js

1. **Bundle size**: Zone.js adds ~36 KB gzipped to the bundle
2. **Performance**: Every async operation triggers full tree change detection
3. **Debugging complexity**: Async stack traces can be confusing
4. **Third-party interference**: Libraries that trigger many async operations cause excessive CD cycles

---

## Zoneless Angular (Angular 17+)

```ts
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection(),
  ]
});
```

Remove Zone.js from polyfills in `angular.json`:
```json
"polyfills": ["@angular/localize/init"]  // no more zone.js
```

Without Zone.js, you must use signals or manual `markForCheck()` to trigger updates:

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `{{ count() }}`,
})
export class Counter {
  count = signal(0);  // signals notify Angular automatically

  increment() { this.count.update(c => c + 1); } // no Zone.js needed
}
```

---

## Common Interview Questions

**Q: What happens if a third-party library triggers many async operations?**
Each one triggers a Zone.js notification and potentially a full change detection cycle. This can cause performance issues. Solution: run third-party code outside Angular's zone with `runOutsideAngular()`.

**Q: Does `ChangeDetectionStrategy.OnPush` eliminate Zone.js overhead?**
Partially. OnPush skips the component's subtree unless inputs change or an Observable emits. But Zone.js still triggers the top-level tick — Angular just skips unchanged OnPush components during traversal.

**Q: Is zoneless Angular production-ready?**
As of Angular 18, experimental. The API is `provideExperimentalZonelessChangeDetection()`. It works well with signals but requires all async operations to use signals or manual notifications. Stable release expected in Angular 19/20.

**Q: How does Zone.js affect async/await?**
`async/await` is compiled to Promises under the hood, which Zone.js patches. So `await` inside Angular code automatically triggers change detection when the awaited operation completes.
