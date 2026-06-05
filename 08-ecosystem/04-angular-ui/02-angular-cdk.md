# Angular CDK (Component Dev Kit)

## The Idea

**In plain English:** The Angular CDK is a toolkit that gives developers the working logic behind common UI behaviors — like dragging, scrolling, and pop-ups — without deciding what those things look like. Think of it as the invisible machinery that makes interactive parts of a website work, separate from the colors and styles.

**Real-world analogy:** Imagine a theater stage with a trapdoor, a spotlight, and a rotating platform. The stage crew operates all of it from backstage — they handle the mechanics so the actors (your visual components) can perform. You can dress the stage however you like; the crew's job stays the same.

- The trapdoor = the Overlay system (things pop in and out of the page at the right position)
- The spotlight = the A11y focus trap (keeps keyboard focus exactly where it should be, like a spotlight following the active actor)
- The rotating platform = the DragDrop module (moves items around on cue without caring what they look like)

---

## What It Is

The Angular CDK provides behavioral primitives for building UI components — the "headless" layer. It handles the hard parts (accessibility, positioning, drag-drop) without imposing visual styles.

```bash
npm install @angular/cdk
```

---

## Overlay — Popup/Tooltip Positioning

```ts
import { Overlay, OverlayConfig } from '@angular/cdk/overlay';
import { ComponentPortal } from '@angular/cdk/portal';

@Injectable({ providedIn: 'root' })
export class TooltipService {
  constructor(private overlay: Overlay) {}

  show(origin: ElementRef, content: string) {
    const positionStrategy = this.overlay
      .position()
      .flexibleConnectedTo(origin)
      .withPositions([
        { originX: 'center', originY: 'bottom', overlayX: 'center', overlayY: 'top' },
        { originX: 'center', originY: 'top', overlayX: 'center', overlayY: 'bottom' }, // fallback
      ]);

    const config = new OverlayConfig({
      positionStrategy,
      hasBackdrop: false,
      scrollStrategy: this.overlay.scrollStrategies.reposition(),
    });

    const overlayRef = this.overlay.create(config);
    const portal = new ComponentPortal(TooltipComponent);
    const ref = overlayRef.attach(portal);
    ref.instance.content = content;

    return overlayRef; // caller responsible for .dispose()
  }
}
```

Scroll strategies:
- `reposition()` — repositions overlay as user scrolls
- `close()` — closes overlay when scrolled out of view
- `block()` — prevents scrolling while overlay is open
- `noop()` — does nothing (default)

---

## Portal — Dynamic DOM Insertion

```ts
// DomPortal: move an existing DOM element
const portal = new DomPortal(this.elementRef);

// ComponentPortal: dynamically render a component
const portal = new ComponentPortal(DialogComponent);

// TemplatePortal: render a TemplateRef
const portal = new TemplatePortal(this.templateRef, this.viewContainerRef);
```

```html
<!-- Portal Outlet: declares where portals are inserted -->
<ng-template [cdkPortalOutlet]="activePortal"></ng-template>
```

---

## DragDrop — Sortable Lists

```ts
import { DragDropModule, CdkDragDrop, moveItemInArray, transferArrayItem } from '@angular/cdk/drag-drop';

@Component({
  imports: [DragDropModule],
  template: `
    <ul cdkDropList (cdkDropListDropped)="onDrop($event)">
      @for (item of items; track item.id) {
        <li cdkDrag>{{ item.name }}</li>
      }
    </ul>
  `
})
export class SortableListComponent {
  items = ['Alpha', 'Beta', 'Gamma'];

  onDrop(event: CdkDragDrop<string[]>) {
    moveItemInArray(this.items, event.previousIndex, event.currentIndex);
  }
}
```

**Transfer between lists:**
```html
<ul cdkDropList #todoList="cdkDropList" [cdkDropListConnectedTo]="[doneList]">
  ...
</ul>
<ul cdkDropList #doneList="cdkDropList" [cdkDropListConnectedTo]="[todoList]">
  ...
</ul>
```

---

## VirtualScrollViewport — Large Lists

```ts
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="50" style="height: 400px">
      @for (item of items; track $index) {
        <div *cdkVirtualFor="let item of items" class="item" style="height: 50px">
          {{ item.name }}
        </div>
      }
    </cdk-virtual-scroll-viewport>
  `
})
export class VirtualListComponent {
  items = Array.from({ length: 10000 }, (_, i) => ({ id: i, name: `Item ${i}` }));
}
```

Only renders visible items — handles 100,000+ items without performance issues.

---

## A11y — Accessibility Utilities

```ts
import { A11yModule, FocusTrap, FocusTrapFactory, LiveAnnouncer } from '@angular/cdk/a11y';

@Component({
  imports: [A11yModule],
})
export class ModalComponent implements AfterViewInit, OnDestroy {
  private focusTrap!: FocusTrap;

  constructor(
    private focusTrapFactory: FocusTrapFactory,
    private liveAnnouncer: LiveAnnouncer,
    private el: ElementRef
  ) {}

  ngAfterViewInit() {
    // Trap focus inside modal
    this.focusTrap = this.focusTrapFactory.create(this.el.nativeElement);
    this.focusTrap.focusInitialElement();

    // Announce to screen readers
    this.liveAnnouncer.announce('Dialog opened');
  }

  ngOnDestroy() {
    this.focusTrap.destroy();
  }
}
```

**`cdkTrapFocus` directive** (simpler):
```html
<div cdkTrapFocus cdkTrapFocusAutoCapture>
  <!-- Focus is trapped here -->
</div>
```

---

## BreakpointObserver

```ts
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';

@Component({})
export class ResponsiveComponent implements OnInit {
  isMobile = false;

  constructor(private breakpoints: BreakpointObserver) {}

  ngOnInit() {
    this.breakpoints.observe([Breakpoints.Handset]).subscribe(result => {
      this.isMobile = result.matches;
    });
  }
}
```

---

## Common Interview Questions

**Q: What's the difference between Angular CDK Overlay and a regular `position: fixed` element?**
CDK Overlay handles: positioning relative to a trigger element (with fallback positions), scroll strategy, z-index management, and cleanup. `position: fixed` alone requires manual positioning calculations and doesn't handle scroll or z-index automatically.

**Q: When would you use CDK VirtualScroll vs Angular Material's Virtual Scroll?**
They're the same — Angular Material's virtual scroll IS Angular CDK's virtual scroll with Material styling applied. If you're not using Angular Material, import directly from CDK.

**Q: What's a Portal and when would you use it?**
A Portal renders content at a different location in the DOM than where it's defined in the component tree. Use it for: modals (render at body level to escape stacking contexts), tooltips (render above everything), and dynamic component loading.
