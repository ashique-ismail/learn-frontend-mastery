# @angular/animations

## What It Is

Angular's built-in animation module provides a DSL for declarative animations, integrated with Angular's change detection and template system. Triggers fire automatically on state changes or component lifecycle events.

```bash
npm install @angular/animations
```

---

## Setup

```ts
// app.config.ts
import { provideAnimations } from '@angular/platform-browser/animations';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations(), // or provideNoopAnimations() for testing
  ]
};
```

---

## Core Concepts

### Trigger → State → Transition → Animate

```ts
import {
  trigger, state, style, transition, animate, keyframes, query, stagger
} from '@angular/animations';

@Component({
  animations: [
    trigger('expandCollapse', [
      state('collapsed', style({ height: '0', opacity: 0, overflow: 'hidden' })),
      state('expanded',  style({ height: '*', opacity: 1, overflow: 'hidden' })),
      transition('collapsed <=> expanded', [
        animate('300ms ease-in-out')
      ]),
    ]),
  ],
  template: `
    <div [@expandCollapse]="isExpanded ? 'expanded' : 'collapsed'">
      <ng-content></ng-content>
    </div>
    <button (click)="isExpanded = !isExpanded">Toggle</button>
  `,
})
export class AccordionPanelComponent {
  isExpanded = false;
}
```

`height: '*'` means the element's natural height — Angular computes it automatically.

---

## Route Transitions

```ts
export const routeAnimations = trigger('routeAnimations', [
  transition('* <=> *', [
    query(':enter, :leave', style({ position: 'fixed', width: '100%' }), { optional: true }),
    query(':enter', style({ transform: 'translateX(100%)' }), { optional: true }),
    query(':leave', animate('300ms ease-out', style({ transform: 'translateX(-100%)' })), { optional: true }),
    query(':enter', animate('300ms ease-out', style({ transform: 'translateX(0)' })), { optional: true }),
  ]),
]);

// app.component.html
<main [@routeAnimations]="getRouteAnimation(outlet)">
  <router-outlet #outlet="outlet"></router-outlet>
</main>
```

---

## Keyframes

```ts
trigger('bounce', [
  transition('* => bounced', [
    animate('0.6s', keyframes([
      style({ transform: 'translateY(0)',    offset: 0    }),
      style({ transform: 'translateY(-30px)', offset: 0.3  }),
      style({ transform: 'translateY(-15px)', offset: 0.6  }),
      style({ transform: 'translateY(0)',    offset: 1    }),
    ])),
  ]),
])
```

---

## Stagger — Animate a List

```ts
trigger('listAnimation', [
  transition('* => *', [
    query(':enter', [
      style({ opacity: 0, transform: 'translateY(-10px)' }),
      stagger(50, [
        animate('200ms ease-out', style({ opacity: 1, transform: 'translateY(0)' }))
      ])
    ], { optional: true }),
    query(':leave', [
      stagger(50, [
        animate('150ms ease-in', style({ opacity: 0, transform: 'translateY(-10px)' }))
      ])
    ], { optional: true }),
  ])
])
```

```html
<ul [@listAnimation]="items.length">
  @for (item of items; track item.id) {
    <li>{{ item.name }}</li>
  }
</ul>
```

---

## Animation Callbacks

```ts
@Component({
  template: `
    <div
      [@fadeIn]="state"
      (@fadeIn.start)="onAnimationStart($event)"
      (@fadeIn.done)="onAnimationDone($event)"
    ></div>
  `
})
```

---

## Disabling Animations

```ts
// Disable for testing
TestBed.configureTestingModule({
  providers: [provideNoopAnimations()]
})

// Disable for a component (e.g., for reduced motion)
<div [@.disabled]="prefersReducedMotion">
```

---

## Prefers Reduced Motion

```ts
@Component({
  template: `<div [@fadeIn]="state" [@.disabled]="reducedMotion">...</div>`,
})
export class Component {
  reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
}
```

---

## Comparison with GSAP in Angular

| | @angular/animations | GSAP |
|---|---|---|
| Integration | Native, template-driven | Manual (useGSAP equivalent) |
| Trigger mechanism | Angular state changes | Imperative JS |
| Complexity | Low for simple, complex for advanced | Better for complex sequences |
| Performance | CSS transforms (hardware accelerated) | Also hardware accelerated |
| Timeline support | Limited | Full timeline, scrub, pause/reverse |
| Best for | State transitions, route changes, list animations | Complex choreography, scroll-driven, canvas |

---

## Common Interview Questions

**Q: What's the difference between `transition('void => *')` and `transition(':enter')`?**
They're equivalent — `:enter` is an alias for `void => *` (element entering the DOM). Similarly `:leave` = `* => void`. The `:enter`/`:leave` aliases are more readable.

**Q: Why does `height: '*'` work for animations?**
Angular computes the element's natural height at animation start and uses it as the target value. This enables `0 → auto` height animations without hardcoding pixel values.

**Q: How do you animate elements that are inside an `@for` block?**
Use `stagger` with `query(':enter')` on the parent element, as shown in the stagger example. The `[@listAnimation]="items.length"` binding ensures the trigger fires when the array length changes.
