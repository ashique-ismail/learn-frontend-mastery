# Tab Bar with Active Indicator Animation

## The Idea

**In plain English:** A tab bar shows multiple tabs across the top (or bottom) of a section — think "Overview", "Features", "Pricing", "Reviews". The currently selected tab has an indicator — usually an underline or a pill background. When you click a different tab, that indicator slides smoothly to the new position instead of simply appearing there. This "sliding underline" effect communicates continuity: the same indicator moved, rather than an old one disappearing and a new one appearing.

**Real-world analogy:** Think of a slide rule or an old-school price tag holder on a store shelf.

- The **price cards** are the tab labels — they stay in fixed positions.
- The **sliding bracket** holding the highlighted tag is the active indicator — it physically moves from one position to the next along the shelf.
- When you move the bracket to "Pricing", it slides across the shelf — it doesn't teleport or flash.
- On a screen: CSS `transform: translateX()` is the bracket sliding. The CSS custom property (`--indicator-offset`) is where the bracket needs to go. JavaScript measures each tab's position and sets that custom property.

The engineering challenge: you can't hard-code the `translateX` value because tab widths are dynamic (different label lengths). You need JavaScript to measure, then CSS to animate.

---

## Learning Objectives

- Use `getBoundingClientRect()` to measure tab positions
- Drive indicator position with a CSS custom property
- Apply `transition: transform` for smooth sliding animation
- Implement full keyboard navigation: arrow keys, Home, End, Enter/Space
- Use correct ARIA: `role="tablist"`, `role="tab"`, `role="tabpanel"`, `aria-selected`
- Handle tabs with dynamic content (data from API, added/removed tabs)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Style tabs in a row | ✅ flexbox | — |
| Underline on active tab | ✅ `:checked` or class | — |
| Slide indicator between tabs | ❌ | CSS cannot measure DOM element positions |
| Update `--indicator-offset` on click | ❌ | CSS cannot set custom properties based on interaction with another element |
| Arrow-key navigation | ❌ | Requires `keydown` listener |
| Sync indicator on window resize | ❌ | ResizeObserver required |

---

## HTML & CSS Foundation

```html
<div class="tabs-wrapper" role="region" aria-label="Product details">

  <!-- Tab list -->
  <div class="tabs" role="tablist" aria-label="Product details tabs">

    <button
      class="tab is-active"
      role="tab"
      id="tab-overview"
      aria-selected="true"
      aria-controls="panel-overview"
      tabindex="0"
    >Overview</button>

    <button
      class="tab"
      role="tab"
      id="tab-features"
      aria-selected="false"
      aria-controls="panel-features"
      tabindex="-1"
    >Features</button>

    <button
      class="tab"
      role="tab"
      id="tab-pricing"
      aria-selected="false"
      aria-controls="panel-pricing"
      tabindex="-1"
    >Pricing</button>

    <!-- Sliding indicator — positioned absolutely -->
    <span class="tab-indicator" aria-hidden="true"></span>
  </div>

  <!-- Tab panels -->
  <div
    class="tab-panel"
    id="panel-overview"
    role="tabpanel"
    aria-labelledby="tab-overview"
    tabindex="0"
  >
    <p>Overview content…</p>
  </div>

  <div
    class="tab-panel"
    id="panel-features"
    role="tabpanel"
    aria-labelledby="tab-features"
    tabindex="0"
    hidden
  >
    <p>Features content…</p>
  </div>

  <div
    class="tab-panel"
    id="panel-pricing"
    role="tabpanel"
    aria-labelledby="tab-pricing"
    tabindex="0"
    hidden
  >
    <p>Pricing content…</p>
  </div>

</div>
```

```css
/* ─── Tokens ─── */
:root {
  --tab-indicator-color: #2563eb;
  --tab-indicator-height: 3px;
  --tab-active-color: #111827;
  --tab-idle-color: #6b7280;
  --tab-transition: 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

/* ─── Tab list ─── */
.tabs {
  position: relative;
  display: flex;
  align-items: stretch;
  border-bottom: 1px solid #e5e7eb;
  gap: 0;
  overflow-x: auto;          /* horizontal scroll for many tabs on mobile */
  scrollbar-width: none;     /* hide scrollbar */
  -ms-overflow-style: none;
}

.tabs::-webkit-scrollbar { display: none; }

/* ─── Individual tab ─── */
.tab {
  flex-shrink: 0;
  padding: 0.75rem 1.25rem;
  background: none;
  border: none;
  font-size: 0.9375rem;
  font-weight: 500;
  color: var(--tab-idle-color);
  cursor: pointer;
  white-space: nowrap;
  transition: color var(--tab-transition);
  position: relative;        /* above the indicator's z-index */
  z-index: 1;
}

.tab:focus-visible {
  outline: 2px solid var(--tab-indicator-color);
  outline-offset: -2px;
  border-radius: 4px;
}

.tab.is-active,
.tab[aria-selected="true"] {
  color: var(--tab-active-color);
}

/* ─── Sliding indicator ─── */
.tab-indicator {
  position: absolute;
  bottom: 0;
  left: 0;
  height: var(--tab-indicator-height);
  background: var(--tab-indicator-color);
  border-radius: var(--tab-indicator-height) var(--tab-indicator-height) 0 0;
  /* Width and translateX are set via CSS custom properties by JS */
  width: var(--indicator-width, 0px);
  transform: translateX(var(--indicator-offset, 0px));
  transition:
    transform var(--tab-transition),
    width var(--tab-transition);
  will-change: transform;
}

/* ─── Tab panel ─── */
.tab-panel {
  padding: 1.5rem 0;
  outline: none;
}

/* ─── Reduced motion: instant switch, no slide ─── */
@media (prefers-reduced-motion: reduce) {
  .tab,
  .tab-indicator {
    transition: none;
  }
}
```

---

## React Implementation

```tsx
// useTabs.ts
import {
  useState, useRef, useCallback, useLayoutEffect, useEffect
} from 'react';

export interface TabDefinition {
  id: string;
  label: string;
}

export function useTabs(tabs: TabDefinition[], initialId?: string) {
  const [activeId, setActiveId] = useState(initialId ?? tabs[0]?.id ?? '');
  const tabListRef    = useRef<HTMLDivElement>(null);
  const indicatorRef  = useRef<HTMLSpanElement>(null);

  // Measure active tab and update CSS custom properties
  const updateIndicator = useCallback(() => {
    const list      = tabListRef.current;
    const indicator = indicatorRef.current;
    if (!list || !indicator) return;

    const activeBtn = list.querySelector<HTMLButtonElement>(
      `[data-tab-id="${activeId}"]`
    );
    if (!activeBtn) return;

    const listRect = list.getBoundingClientRect();
    const tabRect  = activeBtn.getBoundingClientRect();

    const offset = tabRect.left - listRect.left + list.scrollLeft;
    const width  = tabRect.width;

    list.style.setProperty('--indicator-offset', `${offset}px`);
    list.style.setProperty('--indicator-width',  `${width}px`);
  }, [activeId]);

  useLayoutEffect(() => {
    updateIndicator();
  }, [updateIndicator]);

  // Re-measure on resize (label font may change)
  useEffect(() => {
    const ro = new ResizeObserver(updateIndicator);
    if (tabListRef.current) ro.observe(tabListRef.current);
    return () => ro.disconnect();
  }, [updateIndicator]);

  // Keyboard navigation: arrow keys, Home, End
  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    const currentIndex = tabs.findIndex(t => t.id === activeId);
    let nextIndex = currentIndex;

    switch (e.key) {
      case 'ArrowRight': nextIndex = (currentIndex + 1) % tabs.length; break;
      case 'ArrowLeft':  nextIndex = (currentIndex - 1 + tabs.length) % tabs.length; break;
      case 'Home':       nextIndex = 0; break;
      case 'End':        nextIndex = tabs.length - 1; break;
      default: return;
    }

    e.preventDefault();
    const nextId = tabs[nextIndex].id;
    setActiveId(nextId);

    // Move DOM focus to the next tab
    const nextBtn = tabListRef.current?.querySelector<HTMLButtonElement>(
      `[data-tab-id="${nextId}"]`
    );
    nextBtn?.focus();
  }, [activeId, tabs]);

  return {
    activeId,
    setActiveId,
    tabListRef,
    indicatorRef,
    handleKeyDown,
  };
}
```

```tsx
// Tabs.tsx
import { useTabs, TabDefinition } from './useTabs';

interface TabsProps {
  tabs: TabDefinition[];
  renderPanel: (id: string) => React.ReactNode;
  label?: string;
}

export function Tabs({ tabs, renderPanel, label = 'Tabs' }: TabsProps) {
  const { activeId, setActiveId, tabListRef, indicatorRef, handleKeyDown } =
    useTabs(tabs);

  return (
    <div className="tabs-wrapper" role="region" aria-label={label}>

      {/* Tab list */}
      <div
        ref={tabListRef}
        className="tabs"
        role="tablist"
        aria-label={`${label} tabs`}
        onKeyDown={handleKeyDown}
      >
        {tabs.map(tab => (
          <button
            key={tab.id}
            data-tab-id={tab.id}
            className={`tab ${activeId === tab.id ? 'is-active' : ''}`}
            role="tab"
            id={`tab-${tab.id}`}
            aria-selected={activeId === tab.id}
            aria-controls={`panel-${tab.id}`}
            tabIndex={activeId === tab.id ? 0 : -1}
            onClick={() => setActiveId(tab.id)}
          >
            {tab.label}
          </button>
        ))}
        <span ref={indicatorRef} className="tab-indicator" aria-hidden="true" />
      </div>

      {/* Tab panels */}
      {tabs.map(tab => (
        <div
          key={tab.id}
          id={`panel-${tab.id}`}
          className="tab-panel"
          role="tabpanel"
          aria-labelledby={`tab-${tab.id}`}
          tabIndex={0}
          hidden={activeId !== tab.id}
        >
          {activeId === tab.id && renderPanel(tab.id)}
        </div>
      ))}

    </div>
  );
}
```

---

## Angular Implementation

```typescript
// tabs.model.ts
export interface TabDefinition { id: string; label: string; }
```

```typescript
// tabs.component.ts
import {
  Component, Input, signal, computed,
  ViewChild, ElementRef, AfterViewInit,
  OnDestroy, ChangeDetectionStrategy, inject, ChangeDetectorRef
} from '@angular/core';
import { NgFor } from '@angular/common';
import { TabDefinition } from './tabs.model';

@Component({
  selector: 'app-tabs',
  standalone: true,
  imports: [NgFor],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="tabs-wrapper" role="region" [attr.aria-label]="label">

      <!-- Tab list -->
      <div
        #tabList
        class="tabs"
        role="tablist"
        [attr.aria-label]="label + ' tabs'"
        (keydown)="onKeyDown($event)"
      >
        <button
          *ngFor="let tab of tabs; trackBy: trackById"
          [attr.data-tab-id]="tab.id"
          class="tab"
          [class.is-active]="activeId() === tab.id"
          role="tab"
          [id]="'tab-' + tab.id"
          [attr.aria-selected]="activeId() === tab.id"
          [attr.aria-controls]="'panel-' + tab.id"
          [tabindex]="activeId() === tab.id ? 0 : -1"
          (click)="activate(tab.id)"
        >{{ tab.label }}</button>

        <span #indicator class="tab-indicator" aria-hidden="true"></span>
      </div>

      <!-- Tab panels -->
      <div
        *ngFor="let tab of tabs; trackBy: trackById"
        [id]="'panel-' + tab.id"
        class="tab-panel"
        role="tabpanel"
        [attr.aria-labelledby]="'tab-' + tab.id"
        [attr.tabindex]="0"
        [hidden]="activeId() !== tab.id"
      >
        <ng-container *ngIf="activeId() === tab.id">
          <!-- panel content projected via ng-content or template -->
        </ng-container>
      </div>

    </div>
  `,
})
export class TabsComponent implements AfterViewInit, OnDestroy {
  @Input() tabs: TabDefinition[] = [];
  @Input() label = 'Tabs';

  @ViewChild('tabList')  tabListEl!: ElementRef<HTMLDivElement>;
  @ViewChild('indicator') indicatorEl!: ElementRef<HTMLSpanElement>;

  activeId = signal('');
  private ro?: ResizeObserver;
  private cdr = inject(ChangeDetectorRef);

  activate(id: string) {
    this.activeId.set(id);
    // Wait for DOM update before measuring
    requestAnimationFrame(() => this.updateIndicator());
  }

  updateIndicator() {
    const list = this.tabListEl.nativeElement;
    const activeBtn = list.querySelector<HTMLElement>(
      `[data-tab-id="${this.activeId()}"]`
    );
    if (!activeBtn) return;

    const listRect = list.getBoundingClientRect();
    const tabRect  = activeBtn.getBoundingClientRect();
    const offset   = tabRect.left - listRect.left + list.scrollLeft;

    list.style.setProperty('--indicator-offset', `${offset}px`);
    list.style.setProperty('--indicator-width',  `${tabRect.width}px`);
  }

  onKeyDown(e: KeyboardEvent) {
    const ids   = this.tabs.map(t => t.id);
    const curr  = ids.indexOf(this.activeId());
    let next    = curr;

    switch (e.key) {
      case 'ArrowRight': next = (curr + 1) % ids.length; break;
      case 'ArrowLeft':  next = (curr - 1 + ids.length) % ids.length; break;
      case 'Home':       next = 0; break;
      case 'End':        next = ids.length - 1; break;
      default: return;
    }
    e.preventDefault();
    this.activate(ids[next]);

    const nextBtn = this.tabListEl.nativeElement.querySelector<HTMLElement>(
      `[data-tab-id="${ids[next]}"]`
    );
    nextBtn?.focus();
  }

  trackById(_: number, tab: TabDefinition) { return tab.id; }

  ngAfterViewInit() {
    this.activeId.set(this.tabs[0]?.id ?? '');
    requestAnimationFrame(() => this.updateIndicator());

    this.ro = new ResizeObserver(() => this.updateIndicator());
    this.ro.observe(this.tabListEl.nativeElement);
  }

  ngOnDestroy() { this.ro?.disconnect(); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `role="tablist"` on the container | Required ARIA role for keyboard pattern |
| `role="tab"` on each button | Identifies interactive tab trigger |
| `role="tabpanel"` on each panel | Identifies content region |
| `aria-selected="true/false"` on each tab | Screen reader announces selection state |
| `aria-controls` links tab to panel | Programmatic association |
| `aria-labelledby` on panel points to tab | Panel is named by its tab label |
| Only active tab has `tabindex="0"` | Roving tabindex: one tab in natural tab order |
| All inactive tabs have `tabindex="-1"` | Focus managed via arrow keys within tablist |
| Arrow Left/Right navigates between tabs | ARIA authoring practice requirement |
| Home/End jump to first/last tab | ARIA authoring practice requirement |
| Panel has `tabindex="0"` | Panel itself is focusable (after Tab from tab bar) |
| `hidden` attribute on inactive panels | Removes from accessibility tree when not shown |
| Indicator has `aria-hidden="true"` | Decorative element, not announced |

---

## Production Pitfalls

**1. Indicator position is wrong on first render**
If you measure tab positions before the font has loaded or before the component has painted, `getBoundingClientRect()` returns 0 or incorrect values. Fix: run `updateIndicator` inside `requestAnimationFrame` inside `AfterViewInit`/`useLayoutEffect`, not in the constructor or `ngOnInit`.

**2. Indicator doesn't update on window resize**
Tab widths change if the font size changes or if the container is resized (responsive layout). Fix: attach a `ResizeObserver` to the tab list element, not a `window:resize` listener — it's more precise and doesn't fire for unrelated page resizes.

**3. Tabs with many items overflow on mobile**
If you have 6+ tabs, the tab bar overflows horizontally. Fix: add `overflow-x: auto; scrollbar-width: none` to the tab list, and auto-scroll the active tab into view with `activeBtn.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'center' })` on activation.

**4. Indicator slides under the tab text**
If the indicator is positioned `z-index: 0` and tab buttons are `z-index: auto`, the indicator is correct. But if any CSS creates a stacking context on the tab bar container, the order may be wrong. Fix: give tabs `position: relative; z-index: 1` and the indicator `z-index: 0` explicitly.

**5. Screen reader reads "1 of 3" only for role="tab" inside role="tablist"**
This announcement only works correctly when the ARIA roles form the exact required parent-child relationship: `tablist > tab`. If you wrap tabs in a `<div>` between the tablist and the tab buttons, the ARIA tree breaks. Fix: ensure `role="tab"` buttons are direct children of `role="tablist"`, or that any intermediary `<li>` has no conflicting role.
