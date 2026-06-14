# Accessible Accordion

## The Idea

**In plain English:** An accordion is a vertically stacked list of headings. Clicking a heading expands a panel of content below it, and clicking it again collapses it. You may allow multiple panels open at once, or enforce that only one panel is open at a time (exclusive mode). The challenge is that none of this state is visible to a screen reader unless you wire it up with the right ARIA attributes and the right keyboard contract.

**Real-world analogy:** Think of a set of physical filing cabinet drawers.

- Each **drawer label** = an accordion header button. You can read the labels without opening anything.
- **Pulling a drawer open** = expanding a panel. The contents are now visible and reachable.
- **Pushing it shut** = collapsing. The contents are hidden and skipped over.
- In a **standard cabinet** (multi-expand), you can leave several drawers open at once.
- In an **office cabinet with a safety lock** (exclusive mode), opening one drawer physically locks all the others — you can only have one open at a time to prevent the cabinet tipping over.
- A **blind colleague** navigating the cabinet = the screen reader user. Without Braille labels on each drawer AND a way to feel whether the drawer is open or closed, they are completely lost. `aria-expanded` is that tactile feedback.

---

## Learning Objectives

- Understand the WAI-ARIA Accordion authoring pattern and its exact keyboard contract
- Wire `aria-controls`, `aria-expanded`, and `aria-labelledby` correctly between headers and panels
- Build animated expand/collapse with CSS `grid-template-rows` — the only approach that transitions `height: auto` without JavaScript measurement
- Implement the compound component pattern in React so consumers control which panels are open
- Implement the same in Angular using signals and a shared service
- Know when `role="region"` is appropriate (and when it creates noise)
- Avoid the production pitfalls: broken keyboard navigation, layout shift, focus loss on collapse

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide panel content | ✅ with `display: none` | — |
| Animate open/close height | ⚠️ fixed heights only | `height: auto` cannot be transitioned; JS measurement or grid trick required |
| Track which panel is open | ❌ | CSS has no persistent toggle state without the `:checked` hack |
| Enforce exclusive mode (one open at a time) | ❌ | Requires knowing which other panels to close when one opens |
| Toggle `aria-expanded` on the button | ❌ | ARIA state requires JS DOM mutation |
| Associate button with panel via `aria-controls` | ⚠️ static only | Static ids work, but dynamic data-driven lists need JS-generated ids |
| Announce collapse/expand to screen readers | ❌ | `aria-expanded` change must be driven by JS |
| Arrow-key navigation between headers | ❌ | Keyboard event listeners cannot be expressed in CSS |

**Conclusion:** CSS handles the transition animation (the grid-rows trick). JS owns every piece of interactive state: which panel is open, ARIA attribute toggling, and keyboard event routing.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  WAI-ARIA pattern: each header is a <button> inside an <h3> (or whatever
  heading level fits the document outline). The button controls a sibling panel.

  aria-controls  : points to the panel id
  aria-expanded  : "true" when panel is open, "false" when closed
  id on button   : referenced by aria-labelledby on the panel
-->
<div class="accordion">

  <!-- Item 1 — open by default -->
  <h3 class="accordion__heading">
    <button
      class="accordion__trigger"
      id="acc-btn-1"
      aria-expanded="true"
      aria-controls="acc-panel-1"
    >
      What is your return policy?
      <span class="accordion__icon" aria-hidden="true"></span>
    </button>
  </h3>
  <div
    class="accordion__panel"
    id="acc-panel-1"
    role="region"
    aria-labelledby="acc-btn-1"
  >
    <div class="accordion__panel-inner">
      <p>You can return any item within 30 days of purchase...</p>
    </div>
  </div>

  <!-- Item 2 — closed by default -->
  <h3 class="accordion__heading">
    <button
      class="accordion__trigger"
      id="acc-btn-2"
      aria-expanded="false"
      aria-controls="acc-panel-2"
    >
      How long does shipping take?
      <span class="accordion__icon" aria-hidden="true"></span>
    </button>
  </h3>
  <div
    class="accordion__panel"
    id="acc-panel-2"
    role="region"
    aria-labelledby="acc-btn-2"
    hidden
  >
    <div class="accordion__panel-inner">
      <p>Standard shipping takes 3–5 business days...</p>
    </div>
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --acc-border: 1px solid #d1d5db;
  --acc-radius: 6px;
  --acc-header-bg: #ffffff;
  --acc-header-bg-hover: #f9fafb;
  --acc-panel-bg: #ffffff;
  --acc-transition: 0.25s ease;
  --acc-icon-size: 20px;
}

/* ─── Container ─── */
.accordion {
  border: var(--acc-border);
  border-radius: var(--acc-radius);
  overflow: hidden;     /* clips the radius on child items */
}

.accordion__heading {
  margin: 0;            /* reset browser heading margin */
}

/* ─── Trigger button ─── */
.accordion__trigger {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  min-height: 52px;     /* comfortable tap target */
  padding: 0.875rem 1.25rem;
  background: var(--acc-header-bg);
  border: none;
  border-bottom: var(--acc-border);
  font-size: 1rem;
  font-weight: 500;
  text-align: left;
  cursor: pointer;
  gap: 1rem;
  transition: background-color var(--acc-transition);
}

.accordion__trigger:hover {
  background: var(--acc-header-bg-hover);
}

.accordion__trigger:focus-visible {
  outline: 2px solid #2563eb;
  outline-offset: -2px;
}

/* ─── Chevron icon (CSS-drawn) ─── */
.accordion__icon {
  flex-shrink: 0;
  width: var(--acc-icon-size);
  height: var(--acc-icon-size);
  position: relative;
  transition: transform var(--acc-transition);
}

.accordion__icon::before,
.accordion__icon::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 8px;
  height: 2px;
  background: currentColor;
  transition: transform var(--acc-transition);
}

.accordion__icon::before {
  left: 2px;
  transform: translateY(-50%) rotate(45deg);
}

.accordion__icon::after {
  right: 2px;
  transform: translateY(-50%) rotate(-45deg);
}

/* Rotate icon when expanded */
.accordion__trigger[aria-expanded="true"] .accordion__icon::before {
  transform: translateY(-50%) rotate(-45deg);
}
.accordion__trigger[aria-expanded="true"] .accordion__icon::after {
  transform: translateY(-50%) rotate(45deg);
}

/* ─── Panel — the grid-rows trick ─── */
/*
  The key insight: `grid-template-rows` CAN be transitioned between
  `0fr` (collapsed, inner content squished to 0) and `1fr` (natural height).
  This avoids JS height measurement entirely.

  The `hidden` attribute still controls whether the panel is in the
  accessibility tree. We override its display via the selector below.
*/
.accordion__panel {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows var(--acc-transition);
  background: var(--acc-panel-bg);
  border-bottom: var(--acc-border);
}

/* Show the panel when its preceding trigger is expanded.
   JS adds/removes `data-open` on the panel element. */
.accordion__panel[data-open] {
  grid-template-rows: 1fr;
}

/* The inner wrapper is required — it's what gets squished to height 0 */
.accordion__panel-inner {
  overflow: hidden;
  padding: 0 1.25rem;
  transition: padding var(--acc-transition);
}

.accordion__panel[data-open] .accordion__panel-inner {
  padding: 1rem 1.25rem;
}

/* Override `hidden` so the grid transition works.
   We rely on grid-template-rows: 0fr instead of display:none for animation. */
.accordion__panel[hidden] {
  display: grid !important;
  visibility: hidden;          /* keeps it out of tab order without display:none */
}

/* When open, restore visibility */
.accordion__panel[data-open] {
  visibility: visible;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .accordion__trigger,
  .accordion__icon,
  .accordion__icon::before,
  .accordion__icon::after,
  .accordion__panel,
  .accordion__panel-inner {
    transition: none;
  }
}
```

**What CSS owns:** chevron rotation, panel height animation via `grid-template-rows`, focus ring, hover background, reduced-motion fallback.

**What CSS cannot own:** which panel is open, `aria-expanded` state, exclusive-mode logic, keyboard navigation between headers.

---

## React Implementation

### Types and Compound Component API

```tsx
// accordion.types.ts
export interface AccordionItem {
  id: string;
  heading: string;
  content: React.ReactNode;
  defaultOpen?: boolean;
}

// The public API surface of the compound component:
//
//   <Accordion exclusive>                  ← only one panel open at a time
//     <Accordion.Item id="q1" heading="...">
//       <p>...</p>
//     </Accordion.Item>
//   </Accordion>
```

### Context and Hook

```tsx
// AccordionContext.tsx
import { createContext, useContext } from 'react';

interface AccordionContextValue {
  openIds: Set<string>;
  toggle: (id: string) => void;
  exclusive: boolean;
}

export const AccordionContext = createContext<AccordionContextValue | null>(null);

export function useAccordion() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('useAccordion must be used inside <Accordion>');
  return ctx;
}
```

### Root Component

```tsx
// Accordion.tsx
import {
  useState, useCallback, useRef, useEffect,
  createContext, useContext, type ReactNode, type KeyboardEvent
} from 'react';
import { AccordionContext } from './AccordionContext';

interface AccordionProps {
  children: ReactNode;
  exclusive?: boolean;       // only one panel open at a time
  defaultOpenIds?: string[]; // panel ids open on first render
  className?: string;
}

export function Accordion({
  children,
  exclusive = false,
  defaultOpenIds = [],
  className,
}: AccordionProps) {
  const [openIds, setOpenIds] = useState<Set<string>>(
    () => new Set(defaultOpenIds)
  );

  const toggle = useCallback((id: string) => {
    setOpenIds(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (exclusive) next.clear();  // close all others first
        next.add(id);
      }
      return next;
    });
  }, [exclusive]);

  // ── Arrow-key navigation between headers ──────────────────────────────────
  // The WAI-ARIA spec says Down Arrow moves focus to the next header,
  // Up Arrow moves to the previous header. Home/End jump to first/last.
  // We gather all trigger buttons inside this accordion at event time
  // to avoid stale refs when items are dynamically added/removed.
  const rootRef = useRef<HTMLDivElement>(null);

  const handleKeyDown = useCallback((e: KeyboardEvent<HTMLDivElement>) => {
    const triggers = Array.from(
      rootRef.current?.querySelectorAll<HTMLButtonElement>(
        '.accordion__trigger'
      ) ?? []
    );
    const focused = document.activeElement as HTMLElement;
    const idx = triggers.indexOf(focused as HTMLButtonElement);
    if (idx === -1) return;

    let next = idx;
    if (e.key === 'ArrowDown')  { e.preventDefault(); next = (idx + 1) % triggers.length; }
    if (e.key === 'ArrowUp')    { e.preventDefault(); next = (idx - 1 + triggers.length) % triggers.length; }
    if (e.key === 'Home')       { e.preventDefault(); next = 0; }
    if (e.key === 'End')        { e.preventDefault(); next = triggers.length - 1; }

    triggers[next]?.focus();
  }, []);

  return (
    <AccordionContext.Provider value={{ openIds, toggle, exclusive }}>
      <div
        ref={rootRef}
        className={`accordion ${className ?? ''}`}
        onKeyDown={handleKeyDown}
      >
        {children}
      </div>
    </AccordionContext.Provider>
  );
}
```

### Item Component

```tsx
// AccordionItem.tsx
import { useId, useRef, useEffect, type ReactNode } from 'react';
import { useAccordion } from './AccordionContext';

interface AccordionItemProps {
  id?: string;          // caller can supply a stable id or let useId() generate one
  heading: string;
  children: ReactNode;
}

export function AccordionItem({ id: externalId, heading, children }: AccordionItemProps) {
  const generatedId = useId();
  const itemId   = externalId ?? generatedId;
  const btnId    = `acc-btn-${itemId}`;
  const panelId  = `acc-panel-${itemId}`;

  const { openIds, toggle } = useAccordion();
  const isOpen = openIds.has(itemId);

  // Keep panel element in sync with open state for the CSS grid trick
  const panelRef = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const el = panelRef.current;
    if (!el) return;
    if (isOpen) {
      el.removeAttribute('hidden');
      el.setAttribute('data-open', '');
    } else {
      el.removeAttribute('data-open');
      // Delay adding `hidden` until after the collapse animation finishes,
      // so screen readers still see the element during the transition.
      // Without this delay, `hidden` cuts off the animation immediately.
      const transitionDuration = 260; // matches --acc-transition: 0.25s
      const timer = setTimeout(() => {
        if (!openIds.has(itemId)) el.setAttribute('hidden', '');
      }, transitionDuration);
      return () => clearTimeout(timer);
    }
  }, [isOpen, itemId, openIds]);

  return (
    <>
      <h3 className="accordion__heading">
        <button
          id={btnId}
          className="accordion__trigger"
          aria-expanded={isOpen}
          aria-controls={panelId}
          onClick={() => toggle(itemId)}
        >
          {heading}
          <span className="accordion__icon" aria-hidden="true" />
        </button>
      </h3>

      <div
        ref={panelRef}
        id={panelId}
        className="accordion__panel"
        role="region"
        aria-labelledby={btnId}
        hidden={!isOpen || undefined}
      >
        <div className="accordion__panel-inner">
          {children}
        </div>
      </div>
    </>
  );
}

// Attach Item to Accordion for compound component ergonomics
Accordion.Item = AccordionItem;
```

### Usage

```tsx
// App.tsx — typical usage
import { Accordion } from './Accordion';

export function FaqSection() {
  return (
    <section aria-labelledby="faq-heading">
      <h2 id="faq-heading">Frequently Asked Questions</h2>

      <Accordion exclusive defaultOpenIds={['returns']}>
        <Accordion.Item id="returns" heading="What is your return policy?">
          <p>You can return any item within 30 days of purchase for a full refund.</p>
        </Accordion.Item>

        <Accordion.Item id="shipping" heading="How long does shipping take?">
          <p>Standard shipping takes 3–5 business days. Express shipping is 1–2 days.</p>
        </Accordion.Item>

        <Accordion.Item id="warranty" heading="Do products come with a warranty?">
          <p>All products include a 12-month manufacturer warranty.</p>
        </Accordion.Item>
      </Accordion>
    </section>
  );
}
```

---

## Angular Implementation

### Model and Service

```typescript
// accordion.model.ts
export interface AccordionItemConfig {
  id: string;
  heading: string;
  defaultOpen?: boolean;
}
```

```typescript
// accordion.service.ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable()   // NOT providedIn: 'root' — each accordion gets its own instance
export class AccordionService {
  private _openIds  = signal<Set<string>>(new Set());
  private _exclusive = signal(false);

  readonly openIds = this._openIds.asReadonly();

  configure(exclusive: boolean, defaultOpenIds: string[]) {
    this._exclusive.set(exclusive);
    this._openIds.set(new Set(defaultOpenIds));
  }

  isOpen(id: string): boolean {
    return this._openIds().has(id);
  }

  toggle(id: string): void {
    this._openIds.update(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (this._exclusive()) next.clear();
        next.add(id);
      }
      return next;
    });
  }
}
```

### Accordion Container Component

```typescript
// accordion.component.ts
import {
  Component, Input, OnInit, inject,
  HostListener, ElementRef, ContentChildren, QueryList,
  AfterContentInit
} from '@angular/core';
import { AccordionService } from './accordion.service';
import { AccordionItemComponent } from './accordion-item.component';

@Component({
  selector: 'app-accordion',
  standalone: true,
  providers: [AccordionService],   // scoped instance per accordion
  imports: [AccordionItemComponent],
  template: `<ng-content />`,
  host: { class: 'accordion' },
})
export class AccordionComponent implements OnInit, AfterContentInit {
  @Input() exclusive = false;
  @Input() defaultOpenIds: string[] = [];

  @ContentChildren(AccordionItemComponent)
  items!: QueryList<AccordionItemComponent>;

  private service = inject(AccordionService);
  private el      = inject(ElementRef);

  ngOnInit() {
    this.service.configure(this.exclusive, this.defaultOpenIds);
  }

  ngAfterContentInit() {
    // nothing required — items self-register via inject(AccordionService)
  }

  // ── Arrow-key navigation ──────────────────────────────────────────────────
  @HostListener('keydown', ['$event'])
  onKeyDown(e: KeyboardEvent) {
    const triggers: HTMLButtonElement[] = Array.from(
      this.el.nativeElement.querySelectorAll('.accordion__trigger')
    );
    const idx = triggers.indexOf(document.activeElement as HTMLButtonElement);
    if (idx === -1) return;

    let next = idx;
    if (e.key === 'ArrowDown') { e.preventDefault(); next = (idx + 1) % triggers.length; }
    if (e.key === 'ArrowUp')   { e.preventDefault(); next = (idx - 1 + triggers.length) % triggers.length; }
    if (e.key === 'Home')      { e.preventDefault(); next = 0; }
    if (e.key === 'End')       { e.preventDefault(); next = triggers.length - 1; }

    triggers[next]?.focus();
  }
}
```

### Accordion Item Component

```typescript
// accordion-item.component.ts
import {
  Component, Input, OnInit, inject,
  computed, effect, ElementRef, ViewChild
} from '@angular/core';
import { AccordionService } from './accordion.service';

@Component({
  selector: 'app-accordion-item',
  standalone: true,
  template: `
    <h3 class="accordion__heading">
      <button
        [id]="btnId"
        class="accordion__trigger"
        [attr.aria-expanded]="isOpen()"
        [attr.aria-controls]="panelId"
        (click)="toggle()"
      >
        {{ heading }}
        <span class="accordion__icon" aria-hidden="true"></span>
      </button>
    </h3>

    <div
      #panelEl
      [id]="panelId"
      class="accordion__panel"
      role="region"
      [attr.aria-labelledby]="btnId"
    >
      <div class="accordion__panel-inner">
        <ng-content />
      </div>
    </div>
  `,
})
export class AccordionItemComponent implements OnInit {
  @Input({ required: true }) id!: string;
  @Input({ required: true }) heading!: string;

  @ViewChild('panelEl') panelEl!: ElementRef<HTMLDivElement>;

  private service = inject(AccordionService);

  get btnId()   { return `acc-btn-${this.id}`; }
  get panelId() { return `acc-panel-${this.id}`; }

  isOpen = computed(() => this.service.isOpen(this.id));

  // Drive panel DOM state from the signal
  private animTimer?: ReturnType<typeof setTimeout>;

  constructor() {
    effect(() => {
      const open = this.service.isOpen(this.id);
      const el   = this.panelEl?.nativeElement;
      if (!el) return;

      clearTimeout(this.animTimer);

      if (open) {
        el.removeAttribute('hidden');
        el.setAttribute('data-open', '');
      } else {
        el.removeAttribute('data-open');
        this.animTimer = setTimeout(() => {
          if (!this.service.isOpen(this.id)) {
            el.setAttribute('hidden', '');
          }
        }, 260);
      }
    });
  }

  ngOnInit() {
    // Panel starts hidden if not in defaultOpenIds
    // effect() fires after first render and sets initial state correctly
  }

  toggle() {
    this.service.toggle(this.id);
  }
}
```

### Usage in a Template

```typescript
// faq.component.ts
import { Component } from '@angular/core';
import { AccordionComponent } from './accordion.component';
import { AccordionItemComponent } from './accordion-item.component';

@Component({
  selector: 'app-faq',
  standalone: true,
  imports: [AccordionComponent, AccordionItemComponent],
  template: `
    <section aria-labelledby="faq-heading">
      <h2 id="faq-heading">Frequently Asked Questions</h2>

      <app-accordion [exclusive]="true" [defaultOpenIds]="['returns']">
        <app-accordion-item id="returns" heading="What is your return policy?">
          <p>You can return any item within 30 days of purchase.</p>
        </app-accordion-item>

        <app-accordion-item id="shipping" heading="How long does shipping take?">
          <p>Standard shipping takes 3–5 business days.</p>
        </app-accordion-item>

        <app-accordion-item id="warranty" heading="Do products come with a warranty?">
          <p>All products include a 12-month manufacturer warranty.</p>
        </app-accordion-item>
      </app-accordion>
    </section>
  `,
})
export class FaqComponent {}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Header is a `<button>` inside a heading element | `<h3><button class="accordion__trigger">` — heading level fits document outline |
| Button has `aria-expanded` | Set to `"true"` / `"false"` via JS on every toggle |
| Button has `aria-controls` pointing to panel id | Static attribute referencing `id="acc-panel-{id}"` |
| Panel has `role="region"` | Applied when panel has meaningful landmark content; omit for simple show/hide |
| Panel has `aria-labelledby` pointing to button id | Associates the region label with its trigger |
| Icon is hidden from AT | `aria-hidden="true"` on the decorative chevron span |
| Keyboard: Enter/Space toggles the focused header | Native `<button>` handles Enter/Space natively — no extra listener needed |
| Keyboard: Down Arrow moves to next header | `keydown` listener on container, queries all `.accordion__trigger` elements |
| Keyboard: Up Arrow moves to previous header | Same listener, wraps around at boundaries |
| Keyboard: Home / End jump to first / last header | Same listener |
| Collapsed panel content is not reachable by Tab | `hidden` attribute removes panel from tab order; grid trick delays removal until animation ends |
| Animation respects `prefers-reduced-motion` | `@media (prefers-reduced-motion: reduce)` disables all transitions |
| Focus ring visible on keyboard navigation | `:focus-visible` outline on `.accordion__trigger` |
| Minimum tap target | `min-height: 52px` on trigger button |

---

## Production Pitfalls

**1. Using `display: none` during animation cuts off the transition**
If you set `display: none` the moment the user clicks close, the CSS transition never plays — the panel disappears instantly. The grid-rows trick avoids this, but you still must not add `hidden` until after the animation completes. Delay the `hidden` attribute by the transition duration (260ms in the examples above) so the collapsing animation plays out first.

**2. `role="region"` creates landmark noise when overused**
Adding `role="region"` to every panel means screen reader users navigating by landmarks will hear every accordion panel announced. The WAI-ARIA spec recommends `role="region"` only when the panel content is substantial enough to warrant its own landmark — typically more than a short paragraph. For simple FAQ accordions, omit `role="region"` entirely and rely on `aria-labelledby` alone.

**3. Arrow-key navigation breaks with dynamically added items**
If items are added or removed after mount (e.g., search filtering), a cached NodeList of triggers becomes stale. Always query `.accordion__trigger` at keydown time, not at mount time. Both implementations above do this correctly.

**4. `useId()` ids are not stable across server and client**
React's `useId()` generates stable ids within a render, but they look like `:r0:`, `:r1:`. These are fine for `aria-controls` / `aria-labelledby` pairing but will cause hydration mismatches if you render accordions inside `React.lazy()` without matching server state. Prefer explicit `id` props from the data layer (e.g., a CMS slug) wherever you control the data shape.

**5. The grid-rows trick requires a dedicated inner wrapper**
`grid-template-rows: 0fr / 1fr` squishes the direct child of the grid container to zero height. If your content is the direct child (no `.accordion__panel-inner` wrapper), it gets squished — but padding and borders still take up space, so the panel never fully collapses. Always wrap content in a separate inner element that has `overflow: hidden`.

**6. Exclusive mode and default-open conflict silently**
If you pass `exclusive={true}` and `defaultOpenIds={['a', 'b']}`, the first render shows both panels open — violating exclusive mode. Add a guard: in exclusive mode, only honour the first id in `defaultOpenIds` and warn in development if more than one is supplied.

**7. Focus is not returned after a panel collapses under the active focus**
If the user is focused on a link inside an open panel and another mechanism closes that panel (e.g., exclusive mode), the focused element disappears. The browser silently moves focus to `<body>`. Add a check: before collapsing a panel, test whether `document.activeElement` is inside it. If so, move focus to that panel's trigger button first.

---

## Interview Angle

**Q: "Walk me through how you would make an accordion accessible."**

A complete answer covers three layers. The semantic layer: the trigger must be a `<button>` (not a `<div>`) nested inside a heading element so screen reader users navigating by headings can discover every section. The ARIA layer: `aria-expanded` on the button reflects open/closed state; `aria-controls` points to the panel id; `aria-labelledby` on the panel points back to the button so the region announces its own label. The keyboard layer: Enter/Space toggle is free with a native button; Down/Up Arrow, Home, and End must be wired up manually with a `keydown` listener on the container that queries live trigger buttons at event time.

**Follow-up: "Why not just toggle `display: none` for the panel animation?"**

`display: none` is not animatable — the element is removed from layout instantly and the CSS transition has nothing to interpolate between. The conventional workaround is `max-height` with an arbitrarily large value, but this produces a non-linear animation where the easing curve is stretched over dead space. The correct approach is `grid-template-rows: 0fr` to `1fr`, which transitions the row track size and genuinely compresses the content to zero height. The only requirement is a direct child wrapper with `overflow: hidden` so the squished content does not bleed outside the zero-height row.
