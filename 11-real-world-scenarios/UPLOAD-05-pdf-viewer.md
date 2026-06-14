# PDF Viewer (PDF.js)

## The Idea

**In plain English:** You want to embed a PDF inside a webpage — not just hand the browser a `<embed src="file.pdf">` and hope for the best, but build a fully controlled viewer: load the PDF from a URL, render each page to a `<canvas>`, let users flip pages, zoom in and out, and select text. You own the UI; PDF.js owns the rendering engine.

**Real-world analogy:** Think of a professional slide projector setup in a conference room.

- The **PDF file** is the film reel — it contains everything but cannot display itself.
- **PDF.js** is the projector mechanism — it knows how to read the reel and project pixels onto a surface.
- The **canvas element** is the projector screen — a blank surface that receives the projected image.
- **Pagination controls** are the "next slide / previous slide" buttons on the presenter's remote.
- **Zoom** is the physical zoom ring on the projector lens — it changes how large the image appears on the screen, but the source material hasn't changed.
- The **text layer** is a transparent acetate overlay placed on top of the screen — it looks invisible but carries the actual character positions, so you can highlight and copy text even though you're looking at a picture.
- **Progressive rendering** means you project the current slide immediately while the rest of the reel loads in the background — the audience sees something right away instead of waiting for the full reel to spool up.

The key insight: the canvas renders a pixel-perfect image of the PDF page. The text layer is a separate, absolutely-positioned DOM overlay with invisible `<span>` elements that map to the canvas pixels. These two layers work together but are rendered independently.

---

## Learning Objectives

- Load PDF.js as an ES module from a CDN or npm and understand the worker setup
- Render a PDF page to a `<canvas>` at the correct device pixel ratio
- Manage pagination state: current page number, total page count, and boundary conditions
- Apply zoom via CSS `transform: scale()` (no re-render) versus re-rendering at a new viewport scale
- Build the text layer overlay using `pdfjsLib.renderTextLayer` for selectable text
- Implement progressive rendering: render the visible page immediately, defer all others
- Avoid the common pitfalls: stale render tasks, canvas size mismatch, worker path errors

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Fetch and decode a binary PDF file | ❌ | CSS cannot make network requests or parse binary formats |
| Rasterize a PDF page to pixels | ❌ | Requires a rendering engine (PDF.js canvas API) |
| Track current page / total page count | ❌ | CSS has no integer counter that responds to user clicks |
| Cancel an in-progress render on page change | ❌ | Requires calling `renderTask.cancel()` in JS |
| Position text spans to match canvas glyphs | ❌ | Glyph positions come from PDF.js transform matrices, not the layout engine |
| Respond to zoom input and recalculate viewport | ❌ | Viewport scale is a PDF.js concept; CSS `zoom` / `transform` is a separate, approximate substitute |
| Lazy-render pages only when visible | ❌ | Requires IntersectionObserver and JS render-task management |

---

## HTML & CSS Foundation

### The Structure

```html
<!-- PDF viewer shell -->
<div class="pdf-viewer" id="pdf-viewer">

  <!-- Toolbar -->
  <div class="pdf-toolbar" role="toolbar" aria-label="PDF controls">
    <button class="pdf-btn" id="btn-prev" aria-label="Previous page" disabled>
      &#8592;
    </button>

    <span class="pdf-page-info">
      Page <span id="pdf-current-page">1</span>
      of <span id="pdf-total-pages">–</span>
    </span>

    <button class="pdf-btn" id="btn-next" aria-label="Next page">
      &#8594;
    </button>

    <label class="pdf-zoom-label" for="pdf-zoom-select">Zoom</label>
    <select class="pdf-zoom-select" id="pdf-zoom-select" aria-label="Zoom level">
      <option value="0.5">50%</option>
      <option value="0.75">75%</option>
      <option value="1" selected>100%</option>
      <option value="1.25">125%</option>
      <option value="1.5">150%</option>
      <option value="2">200%</option>
    </select>
  </div>

  <!-- Scrollable page container -->
  <div class="pdf-pages" id="pdf-pages" role="document" aria-label="PDF document">

    <!-- One .pdf-page div per rendered page -->
    <div class="pdf-page" data-page-number="1">
      <!-- Canvas layer: the rasterised page image -->
      <canvas class="pdf-canvas" aria-hidden="true"></canvas>

      <!-- Text layer: invisible spans for text selection -->
      <div class="pdf-text-layer"></div>

      <!-- Loading indicator shown while a page renders -->
      <div class="pdf-page-loading" aria-hidden="true">Rendering…</div>
    </div>

  </div>

  <!-- Global loading / error state -->
  <div class="pdf-status" id="pdf-status" role="status" aria-live="polite"></div>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --pdf-toolbar-h: 48px;
  --pdf-bg: #525659;          /* dark surround, matches Acrobat */
  --pdf-page-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
  --pdf-toolbar-bg: #3d3d3d;
  --pdf-toolbar-color: #f0f0f0;
  --pdf-page-bg: #ffffff;
}

/* ─── Viewer shell ─── */
.pdf-viewer {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: var(--pdf-bg);
  overflow: hidden;
  font-family: system-ui, sans-serif;
}

/* ─── Toolbar ─── */
.pdf-toolbar {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  height: var(--pdf-toolbar-h);
  padding: 0 1rem;
  background: var(--pdf-toolbar-bg);
  color: var(--pdf-toolbar-color);
  flex-shrink: 0;
  user-select: none;
}

.pdf-btn {
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  color: inherit;
  font-size: 1.1rem;
  cursor: pointer;
  transition: background 0.15s;
}

.pdf-btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.2);
}

.pdf-btn:disabled {
  opacity: 0.35;
  cursor: default;
}

.pdf-page-info {
  font-size: 0.875rem;
  min-width: 90px;
  text-align: center;
}

.pdf-zoom-label {
  font-size: 0.875rem;
}

.pdf-zoom-select {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 4px;
  color: inherit;
  padding: 4px 6px;
  font-size: 0.875rem;
  cursor: pointer;
}

/* ─── Scrollable page area ─── */
.pdf-pages {
  flex: 1;
  overflow-y: auto;
  overflow-x: auto;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.5rem;
}

/* ─── Individual page wrapper ─── */
/*
  position: relative is essential — the text layer is
  absolutely positioned on top of the canvas.
*/
.pdf-page {
  position: relative;
  background: var(--pdf-page-bg);
  box-shadow: var(--pdf-page-shadow);
  display: inline-block;  /* shrink-wraps to canvas width */
  line-height: 0;         /* removes phantom gap below canvas */
}

/* ─── Canvas (pixel-accurate raster) ─── */
.pdf-canvas {
  display: block;
  /* Width/height set in JS to match viewport. Never set in CSS. */
}

/* ─── Text layer ─── */
/*
  Absolutely covers the canvas. Spans inside have transform:matrix()
  set by PDF.js to match glyph positions. Font size is set to 0 on
  the container; each span sets its own via PDF.js.
*/
.pdf-text-layer {
  position: absolute;
  inset: 0;
  overflow: hidden;
  line-height: 1;
  /* PDF.js text layer CSS (textLayer.css) sets text color to transparent,
     but selection highlight uses ::selection pseudo-element */
}

.pdf-text-layer span {
  color: transparent;
  position: absolute;
  white-space: pre;
  cursor: text;
  transform-origin: 0% 0%;
}

.pdf-text-layer ::selection {
  background: rgba(0, 100, 255, 0.25);
  color: transparent;
}

/* ─── Per-page loading indicator ─── */
.pdf-page-loading {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.875rem;
  color: #666;
  background: var(--pdf-page-bg);
}

.pdf-page-loading.is-hidden {
  display: none;
}

/* ─── Global status ─── */
.pdf-status {
  position: absolute;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.75);
  color: #fff;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  font-size: 0.875rem;
  pointer-events: none;
  opacity: 0;
  transition: opacity 0.2s;
}

.pdf-status.is-visible {
  opacity: 1;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .pdf-btn,
  .pdf-status {
    transition: none;
  }
}
```

**What CSS owns:** visual chrome (toolbar, page shadows, background), text layer selection highlight colour, page loading placeholder, responsive overflow scrolling.

**What CSS cannot own:** canvas dimensions (must match PDF.js viewport pixels exactly), text span positions (set by PDF.js transform matrices), zoom scale calculation, page navigation state.

---

## React Implementation

### Types

```tsx
// pdf-viewer.types.ts
export interface PDFViewerState {
  doc: import('pdfjs-dist').PDFDocumentProxy | null;
  currentPage: number;
  totalPages: number;
  scale: number;
  isLoading: boolean;
  error: string | null;
}
```

### Worker Setup (module-level, outside any component)

```tsx
// pdfWorker.ts — import once at app entry point or in the hook
import * as pdfjsLib from 'pdfjs-dist';

// Point to the worker bundled with pdfjs-dist.
// With Vite/webpack, use the ?url import to get the resolved asset path.
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

pdfjsLib.GlobalWorkerOptions.workerSrc = workerUrl;

export { pdfjsLib };
```

### Core Hook

```tsx
// usePdfViewer.ts
import { useState, useRef, useCallback, useEffect } from 'react';
import { pdfjsLib } from './pdfWorker';
import type { PDFDocumentProxy, PDFPageProxy, RenderTask } from 'pdfjs-dist';

export function usePdfViewer(url: string) {
  const [doc, setDoc]               = useState<PDFDocumentProxy | null>(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [totalPages, setTotalPages] = useState(0);
  const [scale, setScale]           = useState(1);
  const [isLoading, setIsLoading]   = useState(true);
  const [error, setError]           = useState<string | null>(null);

  // Refs to DOM elements — populated by the component via callbacks
  const canvasRef   = useRef<HTMLCanvasElement>(null);
  const textLayerRef = useRef<HTMLDivElement>(null);

  // Track in-progress render task so we can cancel on fast navigation
  const renderTaskRef = useRef<RenderTask | null>(null);

  // ── Load the document ──────────────────────────────────────────────
  useEffect(() => {
    let cancelled = false;
    setIsLoading(true);
    setError(null);

    const loadingTask = pdfjsLib.getDocument(url);

    loadingTask.promise.then(
      (loadedDoc) => {
        if (cancelled) return;
        setDoc(loadedDoc);
        setTotalPages(loadedDoc.numPages);
        setCurrentPage(1);
        setIsLoading(false);
      },
      (err: Error) => {
        if (cancelled) return;
        setError(`Failed to load PDF: ${err.message}`);
        setIsLoading(false);
      }
    );

    return () => {
      cancelled = true;
      loadingTask.destroy();
    };
  }, [url]);

  // ── Render a page ──────────────────────────────────────────────────
  const renderPage = useCallback(
    async (pageNum: number, pageScale: number) => {
      if (!doc || !canvasRef.current || !textLayerRef.current) return;

      // Cancel any in-progress render first
      if (renderTaskRef.current) {
        await renderTaskRef.current.cancel().catch(() => {});
        renderTaskRef.current = null;
      }

      let page: PDFPageProxy;
      try {
        page = await doc.getPage(pageNum);
      } catch {
        return; // doc was destroyed (e.g. URL changed)
      }

      const deviceRatio = window.devicePixelRatio || 1;
      const viewport    = page.getViewport({ scale: pageScale * deviceRatio });

      const canvas  = canvasRef.current;
      const context = canvas.getContext('2d')!;

      // Set canvas backing store size (physical pixels)
      canvas.width  = viewport.width;
      canvas.height = viewport.height;

      // Set CSS display size (logical pixels) — this is the zoom effect
      canvas.style.width  = `${viewport.width / deviceRatio}px`;
      canvas.style.height = `${viewport.height / deviceRatio}px`;

      // Render to canvas
      const renderTask = page.render({ canvasContext: context, viewport });
      renderTaskRef.current = renderTask;

      try {
        await renderTask.promise;
      } catch (err: unknown) {
        // RenderingCancelledException is expected on rapid page changes
        if ((err as { name?: string }).name === 'RenderingCancelledException') return;
        throw err;
      }

      renderTaskRef.current = null;

      // ── Text layer ──────────────────────────────────────────────
      const textLayerEl = textLayerRef.current;
      textLayerEl.innerHTML = '';  // clear previous page's spans

      // Text layer uses the same viewport as the canvas, but at logical scale
      const textViewport = page.getViewport({ scale: pageScale });
      textLayerEl.style.width  = `${textViewport.width}px`;
      textLayerEl.style.height = `${textViewport.height}px`;

      const textContent = await page.getTextContent();

      await pdfjsLib.renderTextLayer({
        textContentSource: textContent,
        container: textLayerEl,
        viewport: textViewport,
        textDivs: [],
      }).promise;
    },
    [doc]
  );

  // Re-render whenever page or scale changes
  useEffect(() => {
    renderPage(currentPage, scale);
  }, [renderPage, currentPage, scale]);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      renderTaskRef.current?.cancel();
      doc?.destroy();
    };
  }, [doc]);

  const goToPage = useCallback(
    (page: number) => {
      setCurrentPage(Math.max(1, Math.min(page, totalPages)));
    },
    [totalPages]
  );

  return {
    canvasRef,
    textLayerRef,
    currentPage,
    totalPages,
    scale,
    isLoading,
    error,
    goToPage,
    setScale,
    canGoPrev: currentPage > 1,
    canGoNext: currentPage < totalPages,
  };
}
```

### Component

```tsx
// PdfViewer.tsx
import { useRef } from 'react';
import { usePdfViewer } from './usePdfViewer';

interface Props {
  url: string;
}

const ZOOM_OPTIONS = [0.5, 0.75, 1, 1.25, 1.5, 2] as const;

export function PdfViewer({ url }: Props) {
  const {
    canvasRef,
    textLayerRef,
    currentPage,
    totalPages,
    scale,
    isLoading,
    error,
    goToPage,
    setScale,
    canGoPrev,
    canGoNext,
  } = usePdfViewer(url);

  if (error) {
    return (
      <div className="pdf-viewer" role="alert">
        <p style={{ color: 'red', padding: '1rem' }}>{error}</p>
      </div>
    );
  }

  return (
    <div className="pdf-viewer">
      {/* Toolbar */}
      <div className="pdf-toolbar" role="toolbar" aria-label="PDF controls">
        <button
          className="pdf-btn"
          aria-label="Previous page"
          disabled={!canGoPrev || isLoading}
          onClick={() => goToPage(currentPage - 1)}
        >
          &#8592;
        </button>

        <span className="pdf-page-info" aria-live="polite" aria-atomic="true">
          Page {currentPage} of {totalPages || '–'}
        </span>

        <button
          className="pdf-btn"
          aria-label="Next page"
          disabled={!canGoNext || isLoading}
          onClick={() => goToPage(currentPage + 1)}
        >
          &#8594;
        </button>

        <label className="pdf-zoom-label" htmlFor="pdf-zoom">Zoom</label>
        <select
          id="pdf-zoom"
          className="pdf-zoom-select"
          value={scale}
          onChange={(e) => setScale(Number(e.target.value))}
          aria-label="Zoom level"
        >
          {ZOOM_OPTIONS.map((z) => (
            <option key={z} value={z}>
              {Math.round(z * 100)}%
            </option>
          ))}
        </select>
      </div>

      {/* Pages */}
      <div className="pdf-pages" role="document" aria-label="PDF document">
        {isLoading && !canvasRef.current && (
          <p style={{ color: '#ccc', marginTop: '2rem' }}>Loading PDF…</p>
        )}

        <div className="pdf-page">
          <canvas
            ref={canvasRef}
            className="pdf-canvas"
            aria-hidden="true"
          />
          <div
            ref={textLayerRef}
            className="pdf-text-layer"
            aria-label={`Page ${currentPage} text content`}
          />
          {isLoading && (
            <div className="pdf-page-loading" aria-hidden="true">
              Rendering…
            </div>
          )}
        </div>
      </div>
    </div>
  );
}
```

---

## Angular Implementation

### Worker Setup

```typescript
// pdf-worker.config.ts — call once in main.ts or app.config.ts
import * as pdfjsLib from 'pdfjs-dist';

export function configurePdfWorker(): void {
  // With Angular + Vite, copy the worker to assets and reference it here.
  // Alternatively, use a CDN URL that matches the installed pdfjs-dist version.
  pdfjsLib.GlobalWorkerOptions.workerSrc =
    `https://unpkg.com/pdfjs-dist@${pdfjsLib.version}/build/pdf.worker.min.mjs`;
}
```

### Service

```typescript
// pdf-viewer.service.ts
import { Injectable, signal, computed } from '@angular/core';
import * as pdfjsLib from 'pdfjs-dist';
import type { PDFDocumentProxy, RenderTask } from 'pdfjs-dist';

@Injectable({ providedIn: 'root' })
export class PdfViewerService {
  private _doc       = signal<PDFDocumentProxy | null>(null);
  private _page      = signal(1);
  private _scale     = signal(1);
  private _loading   = signal(false);
  private _error     = signal<string | null>(null);

  readonly currentPage = this._page.asReadonly();
  readonly scale       = this._scale.asReadonly();
  readonly isLoading   = this._loading.asReadonly();
  readonly error       = this._error.asReadonly();
  readonly totalPages  = computed(() => this._doc()?.numPages ?? 0);
  readonly canGoPrev   = computed(() => this._page() > 1);
  readonly canGoNext   = computed(() => this._page() < this.totalPages());

  private renderTask: RenderTask | null = null;
  private currentDoc: PDFDocumentProxy | null = null;

  async load(url: string): Promise<void> {
    this._loading.set(true);
    this._error.set(null);

    // Destroy previous document
    await this.currentDoc?.destroy();
    this.currentDoc = null;

    try {
      const doc = await pdfjsLib.getDocument(url).promise;
      this._doc.set(doc);
      this.currentDoc = doc;
      this._page.set(1);
    } catch (err: unknown) {
      this._error.set(`Failed to load PDF: ${(err as Error).message}`);
    } finally {
      this._loading.set(false);
    }
  }

  async renderPage(
    canvas: HTMLCanvasElement,
    textLayer: HTMLDivElement
  ): Promise<void> {
    const doc = this._doc();
    if (!doc) return;

    // Cancel any in-flight render
    if (this.renderTask) {
      await this.renderTask.cancel().catch(() => {});
      this.renderTask = null;
    }

    const page = await doc.getPage(this._page());
    const deviceRatio = window.devicePixelRatio || 1;
    const viewport = page.getViewport({ scale: this._scale() * deviceRatio });

    canvas.width  = viewport.width;
    canvas.height = viewport.height;
    canvas.style.width  = `${viewport.width / deviceRatio}px`;
    canvas.style.height = `${viewport.height / deviceRatio}px`;

    const ctx = canvas.getContext('2d')!;
    const task = page.render({ canvasContext: ctx, viewport });
    this.renderTask = task;

    try {
      await task.promise;
    } catch (err: unknown) {
      if ((err as { name?: string }).name === 'RenderingCancelledException') return;
      throw err;
    }

    this.renderTask = null;

    // Text layer
    textLayer.innerHTML = '';
    const textViewport = page.getViewport({ scale: this._scale() });
    textLayer.style.width  = `${textViewport.width}px`;
    textLayer.style.height = `${textViewport.height}px`;

    const textContent = await page.getTextContent();
    await pdfjsLib.renderTextLayer({
      textContentSource: textContent,
      container: textLayer,
      viewport: textViewport,
      textDivs: [],
    }).promise;
  }

  goToPage(page: number): void {
    const clamped = Math.max(1, Math.min(page, this.totalPages()));
    this._page.set(clamped);
  }

  setScale(scale: number): void {
    this._scale.set(scale);
  }

  destroy(): void {
    this.renderTask?.cancel();
    this.currentDoc?.destroy();
    this._doc.set(null);
  }
}
```

### Component

```typescript
// pdf-viewer.component.ts
import {
  Component, OnInit, OnDestroy, Input,
  ViewChild, ElementRef, inject, effect
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { PdfViewerService } from './pdf-viewer.service';
import { configurePdfWorker } from './pdf-worker.config';

configurePdfWorker();

const ZOOM_OPTIONS = [
  { value: 0.5,  label: '50%'  },
  { value: 0.75, label: '75%'  },
  { value: 1,    label: '100%' },
  { value: 1.25, label: '125%' },
  { value: 1.5,  label: '150%' },
  { value: 2,    label: '200%' },
];

@Component({
  selector: 'app-pdf-viewer',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <div class="pdf-viewer">

      <!-- Toolbar -->
      <div class="pdf-toolbar" role="toolbar" aria-label="PDF controls">
        <button
          class="pdf-btn"
          aria-label="Previous page"
          [disabled]="!svc.canGoPrev() || svc.isLoading()"
          (click)="svc.goToPage(svc.currentPage() - 1)"
        >&#8592;</button>

        <span
          class="pdf-page-info"
          aria-live="polite"
          aria-atomic="true"
        >
          Page {{ svc.currentPage() }} of {{ svc.totalPages() || '–' }}
        </span>

        <button
          class="pdf-btn"
          aria-label="Next page"
          [disabled]="!svc.canGoNext() || svc.isLoading()"
          (click)="svc.goToPage(svc.currentPage() + 1)"
        >&#8594;</button>

        <label class="pdf-zoom-label" for="pdf-zoom">Zoom</label>
        <select
          id="pdf-zoom"
          class="pdf-zoom-select"
          [value]="svc.scale()"
          (change)="onZoom($event)"
          aria-label="Zoom level"
        >
          <option *ngFor="let z of zoomOptions" [value]="z.value">
            {{ z.label }}
          </option>
        </select>
      </div>

      <!-- Error state -->
      <p *ngIf="svc.error()" role="alert" style="color: red; padding: 1rem">
        {{ svc.error() }}
      </p>

      <!-- Pages -->
      <div class="pdf-pages" role="document" aria-label="PDF document">
        <div class="pdf-page">
          <canvas
            #canvasEl
            class="pdf-canvas"
            aria-hidden="true"
          ></canvas>

          <div
            #textLayerEl
            class="pdf-text-layer"
            [attr.aria-label]="'Page ' + svc.currentPage() + ' text content'"
          ></div>

          <div
            *ngIf="svc.isLoading()"
            class="pdf-page-loading"
            aria-hidden="true"
          >Rendering…</div>
        </div>
      </div>

    </div>
  `,
})
export class PdfViewerComponent implements OnInit, OnDestroy {
  @Input({ required: true }) url!: string;

  @ViewChild('canvasEl',    { static: true }) canvasRef!:    ElementRef<HTMLCanvasElement>;
  @ViewChild('textLayerEl', { static: true }) textLayerRef!: ElementRef<HTMLDivElement>;

  svc = inject(PdfViewerService);
  readonly zoomOptions = ZOOM_OPTIONS;

  constructor() {
    // Re-render whenever page or scale changes
    effect(() => {
      // Read both signals to register dependencies
      this.svc.currentPage();
      this.svc.scale();
      this.svc.renderPage(
        this.canvasRef.nativeElement,
        this.textLayerRef.nativeElement
      );
    });
  }

  ngOnInit(): void {
    this.svc.load(this.url);
  }

  onZoom(event: Event): void {
    this.svc.setScale(Number((event.target as HTMLSelectElement).value));
  }

  ngOnDestroy(): void {
    this.svc.destroy();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Canvas is hidden from assistive technology | `aria-hidden="true"` on `<canvas>` — it is a visual representation only |
| Text layer carries semantic content | `aria-label` on `.pdf-text-layer` div; spans contain actual PDF text |
| Page change is announced to screen readers | `aria-live="polite"` + `aria-atomic="true"` on page info span |
| Navigation buttons have accessible labels | `aria-label="Previous page"` / `aria-label="Next page"` — not just arrow symbols |
| Disabled buttons are correctly communicated | Native `disabled` attribute (not `aria-disabled`) — removes from tab order |
| Zoom select has a label | `<label for>` association and `aria-label` fallback on the `<select>` |
| Error message is announced immediately | `role="alert"` causes immediate announcement (assertive) |
| Document container is labelled | `role="document"` + `aria-label="PDF document"` on the pages container |
| Keyboard navigation works | All controls are native focusable elements; no `tabindex` hacks needed |
| Loading state does not trap focus | Loading overlay uses `aria-hidden="true"`; focus remains on toolbar controls |

---

## Production Pitfalls

**1. Worker path is wrong and PDF.js silently falls back to a fake worker**
If `GlobalWorkerOptions.workerSrc` points to a missing URL, PDF.js uses a synchronous fake worker that runs on the main thread. The viewer appears to work in dev but freezes the UI on large PDFs. Fix: confirm the worker URL returns a 200 in DevTools Network before shipping. Always pin the worker URL to the same version as the installed `pdfjs-dist` package — a version mismatch causes a runtime error.

**2. Canvas is sized in CSS instead of via the viewport**
Setting `width` and `height` on the canvas element via CSS stretches or shrinks the backing pixel buffer. The render looks blurry. Fix: always set `canvas.width` and `canvas.height` in JS to the exact pixel values from `page.getViewport()`. Use `canvas.style.width` only to control the CSS display size for HiDPI compensation.

**3. Rapid page navigation leaves a stale render on screen**
If the user clicks "next" three times quickly, three render tasks are in flight. They resolve out of order and the final canvas shows page N-2, not page N. Fix: call `renderTask.cancel()` and `await` its promise before starting a new render. PDF.js throws `RenderingCancelledException` — catch it explicitly and return early instead of propagating.

**4. Text layer is misaligned after zoom**
The text layer viewport must be computed at `scale` (logical pixels), not `scale * devicePixelRatio`. The canvas uses the physical pixel viewport to fill the full backing buffer. If you accidentally use the physical viewport for the text layer, the spans are offset on HiDPI screens. Keep the two viewports separate: physical for canvas, logical for text layer.

**5. Memory leak from unreleased PDF document objects**
`PDFDocumentProxy` holds decoded font and image data in memory. If the component unmounts or the URL changes without calling `doc.destroy()`, the memory is never released. In React, return a cleanup function from the `useEffect` that loads the document. In Angular, call `destroy()` in `ngOnDestroy`.

**6. Progressive rendering not implemented — all pages render at mount**
Rendering every page eagerly blocks the main thread and exhausts GPU memory for large PDFs. Fix: use `IntersectionObserver` on each `.pdf-page` wrapper. Only call `page.render()` when the page enters the viewport. Cancel the render task and clear the canvas when the page leaves the viewport (with a buffer of one page above/below so scrolling feels instant).

**7. `getViewport` scale must account for CSS transform zoom**
If you apply `transform: scale(1.5)` to the page container as a quick zoom shortcut, the canvas pixels are not re-rendered — they are just stretched. This is fast but produces a blurry result above ~125%. For production zoom, recalculate the viewport at the new scale and re-render. Use CSS transform only for transient pinch-to-zoom gestures, then re-render at the snapped scale on gesture end.

---

## Interview Angle

**Q: "How does a PDF viewer in the browser actually work? What is the relationship between the canvas, the text layer, and PDF.js?"**

PDF.js decodes the PDF byte stream in a Web Worker, interprets the PDF drawing operators (arcs, glyphs, fills), and executes them against a Canvas 2D rendering context. The result is a pixel-perfect raster image — visually identical to the native viewer, but just pixels: you cannot click on text. The text layer is a second pass: PDF.js also parses the text content and position matrices from the PDF, then places transparent `<span>` elements absolutely on top of the canvas, sized and positioned using CSS `transform: matrix()` to match the glyph bounding boxes. The two layers never interact — the canvas paints, the text layer handles input. The critical detail is that the text layer must use a viewport at CSS logical pixels, while the canvas backing buffer uses physical pixels (scaled by `window.devicePixelRatio`). Getting these two scales wrong is the most common source of misaligned text selection on HiDPI displays.

**Follow-up: "How would you implement progressive rendering for a 200-page PDF?"**

Never render all 200 pages eagerly. The correct approach is to create 200 empty `.pdf-page` placeholder divs sized to the expected page dimensions (you can read the page height from `page.getViewport({ scale: 1 })` before rendering). Attach an `IntersectionObserver` to each placeholder. When a page enters the viewport, call `doc.getPage(n)` then `page.render()`. When it leaves, cancel the render task and call `canvas.getContext('2d').clearRect()` to free GPU texture memory. Keep a render queue with a concurrency limit of 2–3 so fast scrolling does not spawn hundreds of simultaneous render tasks. The user always sees the current page rendered immediately because you prioritise the visible pages at the front of the queue.
