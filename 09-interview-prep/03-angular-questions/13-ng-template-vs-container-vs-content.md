# ng-template vs ng-container vs ng-content

## Overview

Three different Angular template mechanisms, often confused:

| | `ng-template` | `ng-container` | `ng-content` |
|---|---|---|---|
| Renders to DOM | No (unless instantiated) | No (grouping only) | Yes (projected content) |
| Purpose | Lazy template fragment | Structural directive host | Content projection slot |
| Usage | `*ngIf else`, `ngTemplateOutlet` | Multiple directives, no wrapper | Parent → Child content passing |

---

## `ng-template`

A **lazy template fragment** — not rendered until explicitly instantiated. Used as the `else` branch of structural directives, or referenced via `TemplateRef`.

```html
<!-- As else branch -->
<div *ngIf="isLoggedIn; else loginTemplate">
  Welcome back!
</div>
<ng-template #loginTemplate>
  <a routerLink="/login">Please log in</a>
</ng-template>

<!-- Explicit instantiation with ngTemplateOutlet -->
<ng-container *ngTemplateOutlet="loginTemplate"></ng-container>

<!-- With context variables -->
<ng-template #item let-user="user" let-index="index">
  {{ index }}. {{ user.name }}
</ng-template>
<ng-container *ngTemplateOutlet="item; context: { user: currentUser, index: 0 }">
</ng-container>
```

**TypeScript side:**
```ts
@ViewChild('item') itemTemplate!: TemplateRef<{ user: User; index: number }>;

// Pass template as input to child component
<app-table [rowTemplate]="itemTemplate" />
```

---

## `ng-container`

A **logical grouping element** that renders no actual DOM element. Solves two problems:

**1. Applying multiple structural directives** (only one per element):

```html
<!-- ❌ Can't do this -->
<div *ngIf="condition" *ngFor="let item of items">...</div>

<!-- ✅ Use ng-container for the extra directive -->
<ng-container *ngIf="condition">
  <div *ngFor="let item of items">{{ item }}</div>
</ng-container>
```

**2. Grouping without a wrapper element** (avoids polluting the DOM):

```html
<!-- ❌ Adds a <span> to the DOM -->
<span *ngFor="let item of items">{{ item }}</span>

<!-- ✅ No extra DOM element -->
<ng-container *ngFor="let item of items">{{ item }}</ng-container>

<!-- New control flow syntax (Angular 17+) — ng-container often not needed -->
@for (item of items; track item.id) {
  {{ item.name }}
}
```

---

## `ng-content`

A **content projection slot** — allows a parent component to inject HTML into a child component's template. Similar to `children` in React.

**Single-slot:**
```ts
// Card component template
@Component({
  template: `
    <div class="card">
      <ng-content></ng-content>
    </div>
  `
})
export class CardComponent {}
```

```html
<!-- Usage -->
<app-card>
  <h2>Title</h2>
  <p>Body text</p>
</app-card>
<!-- Renders: <div class="card"><h2>Title</h2><p>Body text</p></div> -->
```

**Multi-slot with `select`:**
```ts
@Component({
  template: `
    <header><ng-content select="[card-title]"></ng-content></header>
    <section><ng-content select="[card-body]"></ng-content></section>
    <footer><ng-content></ng-content></footer>
  `
})
export class CardComponent {}
```

```html
<app-card>
  <h2 card-title>My Title</h2>
  <p card-body>Body text</p>
  <button>Default slot</button>
</app-card>
```

**`ngProjectAs`** — project an element as if it matches a different selector:
```html
<ng-container ngProjectAs="[card-title]">
  <h2>Title</h2>
</ng-container>
```

---

## Common Interview Questions

**Q: Does `ng-template` add any DOM elements?**
No. `ng-template` itself is never rendered to the DOM — it's a blueprint. Its contents only appear when instantiated via `*ngIf`, `ngTemplateOutlet`, `*ngFor`, etc.

**Q: When would you use `ng-container` over a `div`?**
When you need to host a structural directive but don't want an extra element in the DOM (e.g., inside a `<table>` where `<div>` would be invalid HTML, or to group elements for `*ngIf` without adding wrapper markup).

**Q: What's the difference between content projection and template references?**
`ng-content` projects DOM that the *parent* provides — the parent controls what HTML goes into the child's slots. `ng-template` with `TemplateRef` passes a *template* as an *input* to the child — the child instantiates the template, potentially with its own context data.

**Q: Can you have multiple `<ng-content>` in one component?**
Yes, using the `select` attribute to target different content by CSS selector, element name, or attribute. One `<ng-content>` with no `select` acts as the catch-all default slot.
