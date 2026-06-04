# OnPush vs Default Change Detection

## Default Change Detection

Every component uses `ChangeDetectionStrategy.Default` unless you opt out.

On every change detection cycle (triggered by any async event — click, HTTP response, setTimeout, etc.), Angular **traverses the entire component tree** from root to leaves, checking every component for changes.

```ts
@Component({
  // ChangeDetectionStrategy.Default is implicit
  template: `{{ user.name }}`
})
export class DefaultComponent {
  @Input() user: User;
}
```

This is safe but potentially wasteful — even components with no changed inputs get checked.

---

## `ChangeDetectionStrategy.OnPush`

With OnPush, Angular **skips checking a component** unless one of these conditions is true:

1. **An `@Input()` reference changed** (not deep mutation — reference equality)
2. **An event was triggered inside the component** (click, input, etc.)
3. **An Observable emitted** and `async` pipe is used — or `markForCheck()` was called
4. **`ChangeDetectorRef.markForCheck()`** was explicitly called
5. **A signal the template reads was updated** (Angular 17+)

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `{{ user.name }}`
})
export class OnPushComponent {
  @Input() user: User;
  // ↑ component only checked when user reference changes
}
```

---

## The Mutation Trap

OnPush is broken by **mutating** objects instead of replacing references:

```ts
// ❌ OnPush component will NOT detect this change
this.user.name = 'New Name'; // same object reference → OnPush skips

// ✅ New reference → OnPush sees the change
this.user = { ...this.user, name: 'New Name' };
```

This is why OnPush encourages **immutable data patterns** — always replace, never mutate.

---

## Observable + async pipe (Most Common OnPush Pattern)

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @if (user$ | async; as user) {
      <div>{{ user.name }}</div>
    }
  `
})
export class UserCardComponent {
  @Input() userId!: string;

  user$ = this.store.select(selectUser(this.userId));
  // async pipe marks the component for check when user$ emits
}
```

The `async` pipe calls `markForCheck()` automatically when the Observable emits.

---

## `ChangeDetectorRef` — Manual Control

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ManualComponent {
  data: any;

  constructor(private cdr: ChangeDetectorRef) {}

  // Called from outside (e.g., a service callback)
  updateData(newData: any) {
    this.data = newData;
    this.cdr.markForCheck(); // tell Angular to check this component
  }

  // Detach/reattach for extreme performance control
  pauseChecking() {
    this.cdr.detach(); // stop checking entirely
  }

  resumeChecking() {
    this.cdr.reattach(); // resume
    this.cdr.detectChanges(); // trigger immediate check
  }
}
```

`markForCheck()` vs `detectChanges()`:
- `markForCheck()` — marks the component and ancestors; check happens in the next CD cycle
- `detectChanges()` — triggers a synchronous CD check on this component and its children immediately

---

## Signals — The Modern OnPush Replacement

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `{{ user().name }}`
})
export class SignalComponent {
  user = signal<User>({ name: 'Alice' });

  updateName() {
    this.user.update(u => ({ ...u, name: 'Bob' }));
    // Angular automatically knows to re-check this component
    // No markForCheck() needed — signals track consumers
  }
}
```

Signals + OnPush = fine-grained reactivity without Zone.js overhead.

---

## Performance Impact

```
Default strategy with 1000 components:
  Click event → CD traverses all 1000 components

OnPush with 1000 components:
  Click event → CD only checks components with changed inputs
  → Could be 5-10 components instead of 1000
```

OnPush typically provides 3-10x fewer component checks in large applications.

---

## When to Use Each

**Always use OnPush for:**
- Smart/container components that use a state store
- Presentational/dumb components that only have inputs
- Any performance-critical component with many instances (list items, table rows)

**Stick with Default for:**
- Components you're migrating gradually
- Components with complex mutation-heavy third-party integrations
- Legacy code where you can't control input immutability

---

## Common Interview Questions

**Q: What does OnPush actually prevent?**
It prevents Angular from calling the component's change detection logic (property checks, expression evaluation) when inputs haven't changed by reference. The component's template is not re-evaluated.

**Q: If OnPush component is skipped, are its children also skipped?**
Yes — when Angular decides to skip an OnPush component, it skips the entire subtree below it. This is where the big performance gains come from.

**Q: Can I use OnPush with mutable state (regular class properties)?**
Yes, but you must call `markForCheck()` after every mutation. This is error-prone. Better to use immutable updates or signals/observables, which trigger CD automatically.
