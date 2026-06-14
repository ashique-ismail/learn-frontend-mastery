# Full-Screen Image Lightbox

## The Idea

**In plain English:** A lightbox is a UI pattern where clicking a thumbnail opens the full-resolution image in an overlay that covers the page. Users can navigate through a gallery with arrow keys or on-screen buttons, close it with Escape, and ideally pinch-to-zoom on touch devices. When the lightbox closes, focus returns to the exact thumbnail that opened it.

**Real-world analogy:** Think of a slide projector in a darkened room.

- The **thumbnail grid** = the contact sheet sitting on the table in full light. You point at one photo and say "show me that one."
- The **overlay** = the lights going off, the room going dark. Everything else disappears from attention.
- The **projected slide** = the current full-size image filling your view.
- **Left/right arrow clickers** = navigating between slides in the carousel tray.
- **Pressing the light switch** (Escape) = lights back on, room returns to normal, your finger is still pointing at the same thumbnail you started with.
- **Preloading adjacent slides** = the projectionist silently loading the next carousel tray while you look at the current slide, so there is no wait when you click forward.
- The key insight: the lightbox is not a separate page — it is a **focus context switch** layered on top of the existing page. When it is dismissed, the page must be exactly as the user left it, and focus must be exactly where it was.

---

## Learning Objectives

- Understand why a lightbox requires compound component architecture (provider + consumer hook)
- Build the CSS foundation: full-screen overlay, image centering, transition in/out
- Implement keyboard navigation (ArrowLeft, ArrowRight, Escape) with correct focus management
- Build a manual focus trap that survives dynamic content changes
- Preload adjacent images using the `Image` constructor without rendering them to the DOM
- Integrate pinch-to-zoom as an isolated gesture layer that does not conflict with gallery navigation
- Return focus to the correct trigger element on close
- Handle all ARIA requirements for a dialog-based overlay

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Dark overlay with fade-in | ✅ with `:target` trick | — |
| Display the full-size image centered | ✅ | — |
| Know *which* thumbnail was clicked | ❌ | CSS has no click-source state |
| Navigate left/right through gallery | ❌ | CSS cannot track an integer index |
| Preload adjacent images | ❌ | Requires `new Image()` or `<link rel="preload">` in JS |
| Trap focus inside the open overlay | ❌ | Requires `focusin` / Tab key listeners in JS |
| Return focus to the trigger thumbnail on close | ❌ | Requires storing a reference to the opener element |
| Pinch-to-zoom gesture with scale/translate state | ❌ | Pointer event math, not CSS |
| Announce "Image 3 of 12" to screen readers | ❌ | Dynamic `aria-label` requires JS |
| Lock background scroll without layout shift | ❌ | `overflow:hidden` + scrollbar compensation requires JS |

**Conclusion:** CSS owns the overlay shape, the fade/scale entrance animation, image object-fit centering, and button styling. Every stateful concern — which image, which index, whether open, what the trigger was — belongs to JS.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Thumbnail grid -->
<div class="lightbox-grid" role="list">
  <button
    class="lightbox-trigger"
    aria-label="Open image: Sunset over mountains"
    data-full-src="/images/sunset-full.jpg"
  >
    <img
      class="lightbox-thumb"
      src="/images/sunset-thumb.jpg"
      alt="Sunset over mountains"
      loading="lazy"
    />
  </button>
  <!-- more thumbnails... -->
</div>

<!-- Lightbox overlay (injected/controlled by JS) -->
<div
  id="lightbox"
  class="lightbox-overlay"
  role="dialog"
  aria-modal="true"
  aria-label="Image lightbox"
  aria-hidden="true"
>
  <button class="lightbox-close" aria-label="Close lightbox">
    <span aria-hidden="true">✕</span>
  </button>

  <button class="lightbox-nav lightbox-nav--prev" aria-label="Previous image">
    <span aria-hidden="true">‹</span>
  </button>

  <figure class="lightbox-figure">
    <img class="lightbox-image" src="" alt="" />
    <figcaption class="lightbox-caption" aria-live="polite"></figcaption>
  </figure>

  <button class="lightbox-nav lightbox-nav--next" aria-label="Next image">
    <span aria-hidden="true">›</span>
  </button>

  <!-- SR counter -->
  <p class="lightbox-counter sr-only" aria-live="polite" aria-atomic="true"></p>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --lb-overlay-bg: rgba(0, 0, 0, 0.92);
  --lb-transition: 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  --lb-nav-size: 52px;
  --lb-close-size: 44px;
}

/* ─── Thumbnail grid ─── */
.lightbox-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 0.5rem;
}

.lightbox-trigger {
  display: block;
  padding: 0;
  border: none;
  background: none;
  cursor: pointer;
  border-radius: 4px;
  overflow: hidden;
  /* Visible focus ring for keyboard users */
  outline-offset: 3px;
}

.lightbox-thumb {
  display: block;
  width: 100%;
  aspect-ratio: 4 / 3;
  object-fit: cover;
  transition: transform 0.2s, opacity 0.2s;
}

.lightbox-trigger:hover .lightbox-thumb,
.lightbox-trigger:focus-visible .lightbox-thumb {
  transform: scale(1.04);
  opacity: 0.88;
}

/* ─── Overlay ─── */
.lightbox-overlay {
  position: fixed;
  inset: 0;
  background: var(--lb-overlay-bg);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9000;

  /* Hidden state */
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--lb-transition), visibility var(--lb-transition);
}

.lightbox-overlay.is-open {
  opacity: 1;
  visibility: visible;
}

/* ─── Image ─── */
.lightbox-figure {
  display: flex;
  flex-direction: column;
  align-items: center;
  max-width: min(90vw, 1200px);
  max-height: 90dvh;
  margin: 0;
}

.lightbox-image {
  display: block;
  max-width: 100%;
  max-height: calc(90dvh - 3rem); /* leave room for caption */
  object-fit: contain;
  border-radius: 2px;

  /* Entrance animation */
  transform: scale(0.92);
  transition: transform var(--lb-transition);
  /* Touch manipulation — allow pinch without triggering browser zoom */
  touch-action: none;
  user-select: none;
}

.lightbox-overlay.is-open .lightbox-image {
  transform: scale(1);
}

.lightbox-caption {
  margin: 0.5rem 0 0;
  color: rgba(255, 255, 255, 0.75);
  font-size: 0.875rem;
  text-align: center;
}

/* ─── Nav buttons ─── */
.lightbox-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: var(--lb-nav-size);
  height: var(--lb-nav-size);
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.3);
  background: rgba(0, 0, 0, 0.4);
  color: #fff;
  font-size: 1.75rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s, border-color 0.15s;
}

.lightbox-nav:hover,
.lightbox-nav:focus-visible {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.7);
}

.lightbox-nav--prev { left: 1rem; }
.lightbox-nav--next { right: 1rem; }

.lightbox-nav[disabled] {
  opacity: 0.25;
  cursor: default;
  pointer-events: none;
}

/* ─── Close button ─── */
.lightbox-close {
  position: absolute;
  top: 0.75rem;
  right: 0.75rem;
  width: var(--lb-close-size);
  height: var(--lb-close-size);
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.3);
  background: rgba(0, 0, 0, 0.4);
  color: #fff;
  font-size: 1.125rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* ─── Body scroll lock ─── */
body.lightbox-open {
  overflow: hidden;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .lightbox-overlay,
  .lightbox-image,
  .lightbox-thumb {
    transition: none;
  }
  .lightbox-overlay.is-open .lightbox-image {
    transform: scale(1);
  }
}
```

**What CSS owns:** overlay positioning and fade, image entrance scale, nav/close button appearance, thumbnail hover effect, scroll lock class, reduced-motion fallback.

**What CSS cannot own:** which image is shown, the gallery index, focus trap, pinch-to-zoom gesture math, preloading, returning focus to the trigger.

---

## React Implementation

### Types

```tsx
// lightbox.types.ts
export interface LightboxImage {
  id: string;
  src: string;           // full-resolution URL
  thumbSrc: string;
  alt: string;
  caption?: string;
}

export interface LightboxContextValue {
  open: (images: LightboxImage[], startIndex: number) => void;
  close: () => void;
}
```

### Context + Provider

```tsx
// LightboxContext.tsx
import {
  createContext, useContext, useState, useCallback,
  useEffect, useRef, ReactNode
} from 'react';
import type { LightboxContextValue, LightboxImage } from './lightbox.types';

const LightboxContext = createContext<LightboxContextValue | null>(null);

export function useLightbox(): LightboxContextValue {
  const ctx = useContext(LightboxContext);
  if (!ctx) throw new Error('useLightbox must be used inside <LightboxProvider>');
  return ctx;
}

interface ProviderProps { children: ReactNode; }

export function LightboxProvider({ children }: ProviderProps) {
  const [images, setImages]   = useState<LightboxImage[]>([]);
  const [index, setIndex]     = useState(0);
  const [isOpen, setIsOpen]   = useState(false);

  // Ref to the element that triggered open — used to restore focus on close
  const triggerRef = useRef<HTMLElement | null>(null);
  const overlayRef = useRef<HTMLDivElement>(null);

  // Preload adjacent images whenever index changes
  useEffect(() => {
    if (!isOpen) return;
    [-1, 1].forEach(offset => {
      const adj = images[index + offset];
      if (adj) {
        const img = new Image();
        img.src = adj.src;
      }
    });
  }, [index, images, isOpen]);

  // Keyboard navigation
  useEffect(() => {
    if (!isOpen) return;
    const onKey = (e: KeyboardEvent) => {
      switch (e.key) {
        case 'ArrowRight': setIndex(i => Math.min(i + 1, images.length - 1)); break;
        case 'ArrowLeft':  setIndex(i => Math.max(i - 1, 0));                 break;
        case 'Escape':     close();                                             break;
      }
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [isOpen, images.length]);

  // Focus trap
  useEffect(() => {
    if (!isOpen || !overlayRef.current) return;

    const overlay = overlayRef.current;
    const getFocusable = () =>
      Array.from(
        overlay.querySelectorAll<HTMLElement>(
          'button:not([disabled]), [href], input, [tabindex]:not([tabindex="-1"])'
        )
      );

    // Move focus into the dialog
    getFocusable()[0]?.focus();

    const trap = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      const nodes = getFocusable();
      const first = nodes[0];
      const last  = nodes[nodes.length - 1];
      if (e.shiftKey) {
        if (document.activeElement === first) { e.preventDefault(); last?.focus(); }
      } else {
        if (document.activeElement === last)  { e.preventDefault(); first?.focus(); }
      }
    };

    document.addEventListener('keydown', trap);
    return () => document.removeEventListener('keydown', trap);
  }, [isOpen, index]); // re-run when index changes so disabled nav buttons are excluded

  // Body scroll lock
  useEffect(() => {
    if (isOpen) {
      document.body.classList.add('lightbox-open');
    } else {
      document.body.classList.remove('lightbox-open');
    }
    return () => document.body.classList.remove('lightbox-open');
  }, [isOpen]);

  const open = useCallback((imgs: LightboxImage[], startIndex: number) => {
    triggerRef.current = document.activeElement as HTMLElement;
    setImages(imgs);
    setIndex(startIndex);
    setIsOpen(true);
  }, []);

  const close = useCallback(() => {
    setIsOpen(false);
    // Return focus to the trigger that opened the lightbox
    requestAnimationFrame(() => {
      triggerRef.current?.focus();
      triggerRef.current = null;
    });
  }, []);

  const current = images[index];

  return (
    <LightboxContext.Provider value={{ open, close }}>
      {children}

      {/* Overlay — always in DOM, visibility controlled by class */}
      <div
        ref={overlayRef}
        id="lightbox"
        className={`lightbox-overlay ${isOpen ? 'is-open' : ''}`}
        role="dialog"
        aria-modal="true"
        aria-label={`Image lightbox${current ? `: ${current.alt}` : ''}`}
        aria-hidden={!isOpen}
        onClick={(e) => { if (e.target === e.currentTarget) close(); }}
      >
        {current && (
          <>
            <button className="lightbox-close" onClick={close} aria-label="Close lightbox">
              <span aria-hidden="true">✕</span>
            </button>

            <button
              className="lightbox-nav lightbox-nav--prev"
              onClick={() => setIndex(i => i - 1)}
              disabled={index === 0}
              aria-label="Previous image"
            >
              <span aria-hidden="true">‹</span>
            </button>

            <figure className="lightbox-figure">
              <LightboxImageWithZoom key={current.id} image={current} />
              {current.caption && (
                <figcaption className="lightbox-caption">{current.caption}</figcaption>
              )}
            </figure>

            <button
              className="lightbox-nav lightbox-nav--next"
              onClick={() => setIndex(i => i + 1)}
              disabled={index === images.length - 1}
              aria-label="Next image"
            >
              <span aria-hidden="true">›</span>
            </button>

            {/* Screen reader position announcement */}
            <p className="sr-only" aria-live="polite" aria-atomic="true">
              Image {index + 1} of {images.length}: {current.alt}
            </p>
          </>
        )}
      </div>
    </LightboxContext.Provider>
  );
}
```

### Pinch-to-Zoom Subcomponent

```tsx
// LightboxImageWithZoom.tsx
import { useRef, useEffect, useState } from 'react';
import type { LightboxImage } from './lightbox.types';

interface Props { image: LightboxImage; }

interface ZoomState {
  scale: number;
  originX: number;
  originY: number;
  translateX: number;
  translateY: number;
}

const INITIAL: ZoomState = { scale: 1, originX: 0.5, originY: 0.5, translateX: 0, translateY: 0 };
const MAX_SCALE = 4;

export function LightboxImageWithZoom({ image }: Props) {
  const imgRef    = useRef<HTMLImageElement>(null);
  const [zoom, setZoom] = useState<ZoomState>(INITIAL);

  useEffect(() => {
    const el = imgRef.current;
    if (!el) return;

    let lastDistance = 0;
    let lastScale    = 1;

    const getDistance = (a: Touch, b: Touch) =>
      Math.hypot(b.clientX - a.clientX, b.clientY - a.clientY);

    const getMidpoint = (a: Touch, b: Touch, rect: DOMRect) => ({
      x: (((a.clientX + b.clientX) / 2) - rect.left) / rect.width,
      y: (((a.clientY + b.clientY) / 2) - rect.top)  / rect.height,
    });

    const onTouchStart = (e: TouchEvent) => {
      if (e.touches.length !== 2) return;
      lastDistance = getDistance(e.touches[0], e.touches[1]);
      lastScale    = zoom.scale;
    };

    const onTouchMove = (e: TouchEvent) => {
      if (e.touches.length !== 2) return;
      e.preventDefault(); // stop page zoom
      const dist     = getDistance(e.touches[0], e.touches[1]);
      const ratio    = dist / lastDistance;
      const newScale = Math.min(MAX_SCALE, Math.max(1, lastScale * ratio));
      const rect     = el.getBoundingClientRect();
      const { x, y } = getMidpoint(e.touches[0], e.touches[1], rect);

      setZoom({
        scale: newScale,
        originX: x,
        originY: y,
        translateX: newScale === 1 ? 0 : zoom.translateX,
        translateY: newScale === 1 ? 0 : zoom.translateY,
      });
    };

    const onTouchEnd = (e: TouchEvent) => {
      if (e.touches.length < 2) {
        // Snap back to 1× if below threshold
        setZoom(prev => prev.scale < 1.15 ? INITIAL : prev);
      }
    };

    const onDblClick = () => setZoom(INITIAL);

    el.addEventListener('touchstart',  onTouchStart, { passive: true });
    el.addEventListener('touchmove',   onTouchMove,  { passive: false }); // must be non-passive to preventDefault
    el.addEventListener('touchend',    onTouchEnd,   { passive: true });
    el.addEventListener('dblclick',    onDblClick);

    return () => {
      el.removeEventListener('touchstart',  onTouchStart);
      el.removeEventListener('touchmove',   onTouchMove);
      el.removeEventListener('touchend',    onTouchEnd);
      el.removeEventListener('dblclick',    onDblClick);
    };
  }, [zoom.scale, zoom.translateX, zoom.translateY]);

  const transform = zoom.scale === 1
    ? 'none'
    : `scale(${zoom.scale})`;

  return (
    <img
      ref={imgRef}
      className="lightbox-image"
      src={image.src}
      alt={image.alt}
      style={{
        transform,
        transformOrigin: `${zoom.originX * 100}% ${zoom.originY * 100}%`,
      }}
      draggable={false}
    />
  );
}
```

### Thumbnail Grid Consumer

```tsx
// PhotoGallery.tsx
import { useLightbox } from './LightboxContext';
import type { LightboxImage } from './lightbox.types';

interface Props { images: LightboxImage[]; }

export function PhotoGallery({ images }: Props) {
  const { open } = useLightbox();

  return (
    <div className="lightbox-grid" role="list">
      {images.map((img, i) => (
        <button
          key={img.id}
          role="listitem"
          className="lightbox-trigger"
          aria-label={`Open image: ${img.alt}`}
          onClick={() => open(images, i)}
        >
          <img
            className="lightbox-thumb"
            src={img.thumbSrc}
            alt={img.alt}
            loading="lazy"
          />
        </button>
      ))}
    </div>
  );
}
```

### App Wiring

```tsx
// App.tsx
import { LightboxProvider } from './LightboxContext';
import { PhotoGallery } from './PhotoGallery';
import { GALLERY_IMAGES } from './data';

export function App() {
  return (
    <LightboxProvider>
      <main>
        <PhotoGallery images={GALLERY_IMAGES} />
      </main>
    </LightboxProvider>
  );
}
```

---

## Angular Implementation

### Model & Service

```typescript
// lightbox.model.ts
export interface LightboxImage {
  id: string;
  src: string;
  thumbSrc: string;
  alt: string;
  caption?: string;
}
```

```typescript
// lightbox.service.ts
import { Injectable, signal, computed, effect } from '@angular/core';
import { LightboxImage } from './lightbox.model';

@Injectable({ providedIn: 'root' })
export class LightboxService {
  private _images  = signal<LightboxImage[]>([]);
  private _index   = signal(0);
  private _isOpen  = signal(false);
  private _trigger: HTMLElement | null = null;

  readonly isOpen  = this._isOpen.asReadonly();
  readonly index   = this._index.asReadonly();
  readonly images  = this._images.asReadonly();
  readonly current = computed(() => this._images()[this._index()] ?? null);
  readonly hasPrev = computed(() => this._index() > 0);
  readonly hasNext = computed(() => this._index() < this._images().length - 1);
  readonly counter = computed(() =>
    `Image ${this._index() + 1} of ${this._images().length}`
  );

  constructor() {
    // Preload adjacent images reactively
    effect(() => {
      if (!this._isOpen()) return;
      const idx  = this._index();
      const imgs = this._images();
      [-1, 1].forEach(offset => {
        const adj = imgs[idx + offset];
        if (adj) {
          const img = new Image();
          img.src = adj.src;
        }
      });
    });

    // Body scroll lock
    effect(() => {
      if (this._isOpen()) {
        document.body.classList.add('lightbox-open');
      } else {
        document.body.classList.remove('lightbox-open');
      }
    });
  }

  open(images: LightboxImage[], startIndex: number): void {
    this._trigger = document.activeElement as HTMLElement;
    this._images.set(images);
    this._index.set(startIndex);
    this._isOpen.set(true);
  }

  close(): void {
    this._isOpen.set(false);
    requestAnimationFrame(() => {
      this._trigger?.focus();
      this._trigger = null;
    });
  }

  prev(): void {
    this._index.update(i => Math.max(0, i - 1));
  }

  next(): void {
    this._index.update(i => Math.min(i + 1, this._images().length - 1));
  }
}
```

### Lightbox Overlay Component

```typescript
// lightbox-overlay.component.ts
import {
  Component, inject, ElementRef, effect, HostListener, OnDestroy
} from '@angular/core';
import { NgIf } from '@angular/common';
import { LightboxService } from './lightbox.service';

@Component({
  selector: 'app-lightbox-overlay',
  standalone: true,
  imports: [NgIf],
  template: `
    <div
      #overlay
      id="lightbox"
      class="lightbox-overlay"
      [class.is-open]="lb.isOpen()"
      role="dialog"
      aria-modal="true"
      [attr.aria-label]="lb.current() ? 'Image lightbox: ' + lb.current()!.alt : 'Image lightbox'"
      [attr.aria-hidden]="!lb.isOpen()"
      (click)="onBackdropClick($event)"
    >
      <ng-container *ngIf="lb.current() as img">
        <button class="lightbox-close" (click)="lb.close()" aria-label="Close lightbox">
          <span aria-hidden="true">✕</span>
        </button>

        <button
          class="lightbox-nav lightbox-nav--prev"
          (click)="lb.prev()"
          [disabled]="!lb.hasPrev()"
          aria-label="Previous image"
        >
          <span aria-hidden="true">‹</span>
        </button>

        <figure class="lightbox-figure">
          <img
            #lightboxImg
            class="lightbox-image"
            [src]="img.src"
            [alt]="img.alt"
            (touchstart)="onTouchStart($event)"
            (touchmove)="onTouchMove($event)"
            (touchend)="onTouchEnd($event)"
            (dblclick)="resetZoom()"
          />
          <figcaption *ngIf="img.caption" class="lightbox-caption">
            {{ img.caption }}
          </figcaption>
        </figure>

        <button
          class="lightbox-nav lightbox-nav--next"
          (click)="lb.next()"
          [disabled]="!lb.hasNext()"
          aria-label="Next image"
        >
          <span aria-hidden="true">›</span>
        </button>

        <!-- Screen reader counter -->
        <p class="sr-only" aria-live="polite" aria-atomic="true">
          {{ lb.counter() }}: {{ img.alt }}
        </p>
      </ng-container>
    </div>
  `,
})
export class LightboxOverlayComponent implements OnDestroy {
  lb = inject(LightboxService);
  private el = inject(ElementRef);

  // ── Pinch-to-zoom state ──
  private lastDistance = 0;
  private lastScale    = 1;
  private currentScale = 1;
  private originX      = 0.5;
  private originY      = 0.5;
  private readonly MAX_SCALE = 4;

  // ── Focus trap cleanup ──
  private trapCleanup?: () => void;

  constructor() {
    effect(() => {
      const open = this.lb.isOpen();
      this.trapCleanup?.();
      this.resetZoom();

      if (!open) return;

      setTimeout(() => {
        const overlay = this.el.nativeElement.querySelector('.lightbox-overlay');
        if (!overlay) return;

        const getFocusable = (): HTMLElement[] =>
          Array.from(overlay.querySelectorAll(
            'button:not([disabled]), [href], [tabindex]:not([tabindex="-1"])'
          ));

        getFocusable()[0]?.focus();

        const trap = (e: KeyboardEvent) => {
          if (e.key !== 'Tab') return;
          const nodes = getFocusable();
          const first = nodes[0];
          const last  = nodes[nodes.length - 1];
          if (e.shiftKey && document.activeElement === first) {
            e.preventDefault(); last?.focus();
          } else if (!e.shiftKey && document.activeElement === last) {
            e.preventDefault(); first?.focus();
          }
        };

        document.addEventListener('keydown', trap);
        this.trapCleanup = () => document.removeEventListener('keydown', trap);
      }, 50);
    });
  }

  @HostListener('document:keydown', ['$event'])
  onKey(e: KeyboardEvent): void {
    if (!this.lb.isOpen()) return;
    if (e.key === 'ArrowRight') this.lb.next();
    if (e.key === 'ArrowLeft')  this.lb.prev();
    if (e.key === 'Escape')     this.lb.close();
  }

  onBackdropClick(e: MouseEvent): void {
    if ((e.target as HTMLElement).classList.contains('lightbox-overlay')) {
      this.lb.close();
    }
  }

  // ── Touch / pinch-to-zoom ──
  onTouchStart(e: TouchEvent): void {
    if (e.touches.length !== 2) return;
    this.lastDistance = Math.hypot(
      e.touches[1].clientX - e.touches[0].clientX,
      e.touches[1].clientY - e.touches[0].clientY
    );
    this.lastScale = this.currentScale;
  }

  onTouchMove(e: TouchEvent): void {
    if (e.touches.length !== 2) return;
    e.preventDefault();
    const dist  = Math.hypot(
      e.touches[1].clientX - e.touches[0].clientX,
      e.touches[1].clientY - e.touches[0].clientY
    );
    const img   = (e.target as HTMLImageElement);
    const rect  = img.getBoundingClientRect();
    this.originX = (((e.touches[0].clientX + e.touches[1].clientX) / 2) - rect.left) / rect.width;
    this.originY = (((e.touches[0].clientY + e.touches[1].clientY) / 2) - rect.top)  / rect.height;
    this.currentScale = Math.min(this.MAX_SCALE, Math.max(1, this.lastScale * (dist / this.lastDistance)));
    img.style.transform       = `scale(${this.currentScale})`;
    img.style.transformOrigin = `${this.originX * 100}% ${this.originY * 100}%`;
  }

  onTouchEnd(e: TouchEvent): void {
    if (e.touches.length < 2 && this.currentScale < 1.15) {
      this.resetZoom();
    }
  }

  resetZoom(): void {
    this.currentScale = 1;
    const img = this.el.nativeElement.querySelector('.lightbox-image') as HTMLImageElement | null;
    if (img) {
      img.style.transform       = 'none';
      img.style.transformOrigin = '50% 50%';
    }
  }

  ngOnDestroy(): void {
    this.trapCleanup?.();
    document.body.classList.remove('lightbox-open');
  }
}
```

### Thumbnail Grid Component

```typescript
// photo-gallery.component.ts
import { Component, Input, inject } from '@angular/core';
import { NgFor } from '@angular/common';
import { LightboxService } from './lightbox.service';
import { LightboxImage } from './lightbox.model';

@Component({
  selector: 'app-photo-gallery',
  standalone: true,
  imports: [NgFor],
  template: `
    <div class="lightbox-grid" role="list">
      <button
        *ngFor="let img of images; let i = index; trackBy: trackById"
        role="listitem"
        class="lightbox-trigger"
        [attr.aria-label]="'Open image: ' + img.alt"
        (click)="lb.open(images, i)"
      >
        <img
          class="lightbox-thumb"
          [src]="img.thumbSrc"
          [alt]="img.alt"
          loading="lazy"
        />
      </button>
    </div>
  `,
})
export class PhotoGalleryComponent {
  @Input() images: LightboxImage[] = [];
  lb = inject(LightboxService);
  trackById(_: number, img: LightboxImage) { return img.id; }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Overlay is a `role="dialog"` with `aria-modal="true"` | Prevents virtual cursor from wandering behind the overlay |
| `aria-hidden="true"` on the overlay when closed | Screen readers skip the entire dialog when not open |
| `aria-label` on the dialog updated with current image alt | Announced on open: "Image lightbox: Sunset over mountains" |
| Position announced on navigation | `aria-live="polite" aria-atomic="true"` counter reads "Image 3 of 12: ..." |
| Focus moves into dialog on open | First focusable element (close button) receives focus |
| Focus trapped inside dialog while open | Tab/Shift+Tab cycle bounded; re-evaluated when nav buttons become disabled |
| Escape closes the dialog | `keydown` listener on document; standard dialog behavior |
| ArrowLeft / ArrowRight navigate gallery | Keyboard-accessible alternative to mouse/touch swipe |
| Focus returns to the opening thumbnail on close | Opener stored in a ref/field before open; `focus()` called after close |
| Disabled nav buttons excluded from focus cycle | `disabled` attribute prevents tabbing; `pointer-events: none` in CSS |
| Thumbnail buttons have descriptive `aria-label` | "Open image: [alt text]" — more specific than just the alt text |
| Caption in a `<figcaption>` associated with `<figure>` | Semantic grouping; caption is read after the image description |
| Minimum tap target 44×44px on nav buttons | Enforced via `width`/`height` CSS |
| Pinch-to-zoom does not trigger browser native zoom | `touch-action: none` on image; `e.preventDefault()` in touchmove |
| Reduced motion: no scale entrance animation | `@media (prefers-reduced-motion: reduce)` removes the transition |

---

## Production Pitfalls

**1. `touchmove` passive listener blocks `preventDefault()`**
Modern browsers register touch listeners as passive by default, which means calling `e.preventDefault()` to block native pinch-zoom throws a console warning and does nothing. Fix: explicitly register the touchmove listener with `{ passive: false }`. In Angular template bindings you cannot control passivity — use `ElementRef` or `Renderer2` to add the listener imperatively with the correct option.

**2. Focus trap references go stale after navigation**
When the user moves to the next image, the previous image's nav button may switch from enabled to disabled (at gallery boundaries). The focusable elements list cached on open becomes stale, causing Tab to cycle into a `[disabled]` button. Fix: re-derive the focusable list on every `keydown` Tab event rather than caching it once, or re-run the trap setup whenever the index changes.

**3. `new Image()` preload triggers CORS errors on cross-origin CDN images**
If your CDN requires `crossorigin` credentials and you preload with a bare `new Image(); img.src = url`, the browser makes a no-CORS request, the CDN rejects it, and the prefetch is wasted. Fix: set `img.crossOrigin = 'anonymous'` before assigning `src`, matching the attribute on your actual `<img>` tag.

**4. Body scroll lock causes layout shift on scrollbar removal**
Adding `overflow: hidden` to `body` removes the scrollbar, shifting the page content right by ~15px. This causes a visible jump. Fix: before locking, measure `window.innerWidth - document.documentElement.clientWidth` and apply that as `padding-right` to `body` (or to a fixed header) for the duration of the lock.

**5. Opening animation plays on every index change, not just on first open**
If the `<img>` element is keyed on `image.id` (which is correct for React reconciliation), it remounts on every navigation and the CSS entrance scale animation replays. This feels jarring when browsing quickly. Fix: separate the animation from the image mount. Either use a CSS `@keyframes` animation triggered by an `.entering` class that is removed after the transition, or animate opacity only (no scale) so the flash is less noticeable.

**6. Pinch-to-zoom state persists when navigating to the next image**
If the user zooms in on image 3 and then presses ArrowRight, image 4 opens at the same zoom level and origin. Fix: reset zoom state whenever the gallery index changes. In React, key the `LightboxImageWithZoom` component on `image.id` so it remounts with fresh state. In Angular, call `resetZoom()` inside the index change handler.

**7. `aria-live` fires on every render, not just when open**
If the live region is always in the DOM and the text updates even when the dialog is closed (because index/images are signals that update), screen readers may announce changes the user did not request. Fix: only render the live region element when `isOpen` is `true`, or gate the text content on `isOpen`.

---

## Interview Angle

**Q: "Walk me through how you would architect a lightbox that works across a large codebase where multiple pages need to open it."**

The right answer is a compound component pattern: a `LightboxProvider` (React) or singleton service (Angular) sits at the app root and owns all lightbox state. Individual gallery components call a `useLightbox` hook or inject the service to get an `open(images, index)` function. This means the overlay is rendered exactly once in the DOM, not duplicated on every page that has a gallery. State (current index, open/closed, the trigger reference) lives in one place and is not duplicated. The overlay component subscribes to that shared state. This is the same pattern used for toast notifications and global modals.

**Follow-up: "How do you handle returning focus correctly when there are multiple galleries on the same page?"**

The key is that `triggerRef` (React) or `_trigger` (Angular) is captured at the moment `open()` is called — specifically `document.activeElement` at that instant. Since each thumbnail button is a real focusable element, the currently focused one at click-time is the right element to return to. This works even with multiple galleries because we store the element reference, not an index or selector. The only edge case is programmatic opens (e.g., auto-opening on page load): in that case there is no meaningful trigger element, so you should fall back to focusing the page's main heading or skip-link target rather than failing silently.

**Follow-up: "Why use `requestAnimationFrame` when restoring focus instead of calling `focus()` directly?"**

Calling `focus()` synchronously inside the close handler runs before the React re-render (or Angular change detection) has committed the DOM changes — specifically, before `aria-hidden="true"` has been written back onto the overlay. Some screen readers will call `blur` on an element inside an `aria-hidden` container and immediately re-announce the container, creating a confusing experience. `requestAnimationFrame` defers the `focus()` call to after the paint, guaranteeing the DOM reflects the closed state before the browser processes the focus change.
