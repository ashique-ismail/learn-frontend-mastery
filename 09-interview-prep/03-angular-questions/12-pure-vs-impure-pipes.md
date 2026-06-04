# Pure vs Impure Pipes

## Pure Pipes (Default)

A pure pipe's `transform()` method is only called when Angular detects a **pure change** to the input:
- Primitive value change (`string`, `number`, `boolean`)
- Object/array **reference** change (not mutation of existing object)

```ts
@Pipe({
  name: 'capitalize',
  pure: true,  // this is the default — you don't need to write it
})
export class CapitalizePipe implements PipeTransform {
  transform(value: string): string {
    return value.charAt(0).toUpperCase() + value.slice(1);
  }
}
```

Angular caches the result and only recalculates when inputs change by reference. Like `OnPush` for pipes.

---

## Impure Pipes (`pure: false`)

An impure pipe's `transform()` is called on **every change detection cycle**, regardless of whether the input reference changed.

```ts
@Pipe({
  name: 'filter',
  pure: false,  // must opt in
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], searchTerm: string): any[] {
    if (!searchTerm) return items;
    return items.filter(item =>
      item.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }
}
```

```html
<!-- Called on every CD cycle — potentially every keystroke, scroll, etc. -->
<li *ngFor="let item of items | filter:searchTerm">
```

---

## When You NEED Impure Pipes

Impure pipes are necessary when the output can change even if the **reference** hasn't:

```ts
// Filtering a mutated array (bad practice, but common in legacy code)
this.items.push(newItem); // same array reference → pure pipe won't re-run

// Pure pipe would miss this change:
// items | filter:term  → stale results after push()
```

Also used for the built-in `async` pipe (which handles Observable subscriptions — the observable reference doesn't change, but new values arrive).

---

## Performance Implications

```
Every CD cycle (scroll, click, mouseover, any async event)
    ↓
Impure pipe.transform() runs
    ↓
Could be 100+ times per second
```

For a list of 1000 items with a filter pipe, this is `1000 * filter iterations` per CD cycle.

**Rule: Avoid impure pipes for expensive computations.**

---

## Better Alternatives to Impure Pipes

### 1. Immutable Updates (makes pure pipes work)

```ts
// Instead of mutating:
this.items.push(newItem); // doesn't trigger pure pipe

// Use immutable update:
this.items = [...this.items, newItem]; // new reference → triggers pure pipe
```

### 2. Signals (Angular 17+)

```ts
items = signal<Item[]>([]);
filteredItems = computed(() =>
  this.items().filter(item => item.name.includes(this.searchTerm()))
);
// Template: {{ filteredItems() }} — recomputes only when items or searchTerm changes
```

### 3. Derived Data in the Component

```ts
get filteredItems(): Item[] {
  return this.items.filter(item => item.name.includes(this.searchTerm));
}
// This also runs every CD cycle, but it's explicit
```

---

## Built-In Impure Pipes

Angular's built-in impure pipes:
- `async` — subscribes to Observable/Promise, emits new values
- `json` — for development debugging
- `keyvalue` — iterates object entries (object reference may not change when keys do)

---

## Common Interview Questions

**Q: Why is the `async` pipe impure?**
The Observable reference doesn't change, but the emitted value does. Angular must check for new emissions on every CD cycle. The `async` pipe handles subscriptions, unsubscriptions, and triggers change detection when new values arrive.

**Q: What's the performance cost of impure pipes?**
`transform()` runs every change detection cycle. For a simple transformation on a small dataset, negligible. For expensive computations on large arrays (sorting, filtering), it can cause janky UIs — especially with default change detection.

**Q: How do you use OnPush with an impure pipe?**
You generally can't benefit from OnPush if you're using impure pipes, since impure pipes run regardless of change detection strategy. The solution is to switch to pure pipes with immutable data or use signals/computed.

**Q: Is `| async` the most common impure pipe?**
Yes, and it's the most justified one. Its impurity is essential — it must check the Observable for new emissions on every cycle.
