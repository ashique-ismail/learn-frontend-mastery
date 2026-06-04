# RxAngular

## What It Is

RxAngular is a set of Angular libraries focused on performance through reactive programming. It enables zone-less rendering, local state management, and efficient template binding without the boilerplate of `OnPush` + `ChangeDetectorRef`.

Main packages:
- `@rx-angular/state` — local reactive state management
- `@rx-angular/template` — reactive template directives (`rxLet`, `rxFor`, `push` pipe)
- `@rx-angular/cdk` — reactive utilities (intersection observer, viewport, etc.)

---

## The Problem It Solves

Standard Angular template with `async` pipe:
```html
<!-- Multiple subscriptions, each triggers CD separately -->
<div *ngIf="user$ | async as user">
  <div *ngFor="let item of items$ | async">
    {{ item.name }}
  </div>
  <span>{{ (loading$ | async) ? 'Loading...' : '' }}</span>
</div>
```

RxAngular approach: single subscription per directive, zone-less CD:
```html
<!-- Single subscription, batched updates, no Zone.js needed -->
<div *rxLet="user$; let user">
  <div *rxFor="let item of items$">
    {{ item.name }}
  </div>
</div>
```

---

## `rxLet` Directive

Replaces `async` pipe + `*ngIf as`:

```html
<!-- Standard approach -->
<div *ngIf="data$ | async as data; else loading">
  {{ data.title }}
</div>
<ng-template #loading>Loading...</ng-template>

<!-- rxLet: more complete template states -->
<div *rxLet="data$; let data; rxError: errorTpl; rxSuspense: loadingTpl">
  {{ data.title }}
</div>
<ng-template #loadingTpl>Loading...</ng-template>
<ng-template #errorTpl let-error>Error: {{ error.message }}</ng-template>
```

---

## `rxFor` Directive

High-performance replacement for `*ngFor` with Observables:

```html
<ul>
  <li *rxFor="
    let item of items$;
    trackBy: trackById;
    renderCallback: itemsRendered
  ">
    {{ item.name }}
  </li>
</ul>
```

```ts
trackById = (index: number, item: Item) => item.id;

// Called when all items are rendered (for analytics, testing)
itemsRendered = new Subject<Item[]>();
```

**Performance difference:**
- `*ngFor` + `async` pipe: subscription change → detect entire list → re-render all rows
- `rxFor`: subscription change → update only changed rows (uses reconciliation)

---

## `push` Pipe (Replaces `async`)

```html
<!-- async pipe triggers change detection via Zone.js -->
{{ items$ | async }}

<!-- push pipe triggers change detection directly (works without Zone.js) -->
{{ items$ | push }}
```

The `push` pipe works in zoneless apps where `async` pipe wouldn't trigger CD.

---

## `RxState` — Local Component State

```ts
import { RxState } from '@rx-angular/state';

interface ComponentState {
  items: Item[];
  filter: string;
  loading: boolean;
}

@Component({
  providers: [RxState],  // or: extends RxState<ComponentState>
  template: `
    <input (input)="setFilter($event.target.value)" />
    <div *rxFor="let item of filteredItems$">{{ item.name }}</div>
  `,
})
export class ItemListComponent implements OnInit {
  private state = inject(RxState) as RxState<ComponentState>;

  // Derived state (like computed)
  filteredItems$ = this.state.select(
    map(state => state.items.filter(item => item.name.includes(state.filter)))
  );

  constructor(private itemService: ItemService) {
    // Initialize state
    this.state.set({ items: [], filter: '', loading: false });

    // Connect Observable to state
    this.state.connect('items', this.itemService.getItems());
    this.state.connect('loading', this.itemService.getItems().pipe(
      map(() => false),
      startWith(true)
    ));
  }

  setFilter(filter: string) {
    this.state.set({ filter });
  }
}
```

---

## Zone-less Performance

```ts
// Enable zoneless in standalone app
bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection(),
    // RxAngular directives work natively with zoneless
  ]
});
```

With RxAngular + zoneless:
- **No Zone.js** in the bundle (~36 KB saved)
- Changes only trigger CD for the specific component that subscribed
- Template directives batch multiple synchronous updates into one render

---

## Comparison with Standard Angular

| | Standard + async pipe | RxAngular |
|---|---|---|
| Zone.js required | Yes | No (works zoneless) |
| CD granularity | Whole component tree | Per subscription |
| Loading/error templates | Manual `*ngIf` chains | Built-in rxLet states |
| Large list performance | Re-render all | Reconcile changed rows |
| Learning curve | Lower | Higher |

---

## Common Interview Questions

**Q: How does `rxFor` differ from `*ngFor`?**
`rxFor` accepts Observables directly without `async` pipe, uses reconciliation to update only changed items (instead of re-rendering the whole list), works without Zone.js, and provides a `renderCallback` for when rendering completes.

**Q: Can you use RxAngular with Zone.js still active?**
Yes. RxAngular works with both Zone.js and zoneless configurations. With Zone.js, it still improves performance by using local change detection instead of triggering global CD.

**Q: When is RxAngular worth adopting?**
For performance-critical applications with large reactive data flows — complex dashboards, real-time feeds, data grids. For simple apps with few reactive subscriptions, the added complexity isn't justified.
