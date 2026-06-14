# Image Crop & Resize Before Upload

## The Idea

**In plain English:** Before sending a user's photo to your server, you let them select exactly which rectangular region they want, draw that region onto an off-screen canvas at whatever resolution you need, then export that canvas as a compressed Blob — all in the browser, without touching a server. The result is a smaller, correctly-framed image that arrives upload-ready.

**Real-world analogy:** Think of a physical photo lab.

- The **original negative** = the raw `File` object the browser's `<input>` gives you.
- **Placing the negative on the enlarger** = calling `URL.createObjectURL()` and drawing it into an `<img>` tag so you can see it.
- **The red crop guides on the easel** = the draggable selection handles you draw over the preview.
- **The enlarger lens setting** = the target max-dimension you enforce before printing.
- **Exposing only the selected area onto photographic paper** = `canvas.drawImage(source, sx, sy, sw, sh, 0, 0, dw, dh)` — the six-argument form that reads a sub-rectangle and writes it to the full canvas.
- **Developing the print** = `canvas.toBlob('image/jpeg', 0.85)` — the async call that produces the final compressed binary you hand to `FormData`.
- The **aspect-ratio lock lever** = a checkbox that constrains the crop selection so the width-to-height ratio never changes as you drag.

The key insight: every operation — reading the file, cropping, scaling, compressing — runs entirely on the GPU-accelerated canvas compositing pipeline. No server round-trip, no library needed, no binary WASM bundle required.

---

## Learning Objectives

- Understand how `canvas.drawImage(img, sx, sy, sw, sh, dx, dy, dw, dh)` maps a source sub-rectangle to a destination canvas
- Build pointer-event-driven crop handle dragging with correct coordinate transforms from screen space to image space
- Export the cropped region as a Blob using `canvas.toBlob()` and wrap it for `FormData`
- Enforce max-dimension resizing before export without distorting the image
- Implement an aspect-ratio lock that constrains drag behavior live
- Understand the production edge cases: EXIF orientation, cross-origin images, alpha channels, and memory leaks from object URLs

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Display a cropped preview without altering the file | ✅ `object-fit` / `clip-path` | Only cosmetic — the underlying data is unchanged |
| Produce a new Blob containing only the cropped pixels | ❌ | CSS cannot read or write pixel data |
| Resize an image to exact pixel dimensions before upload | ❌ | CSS `width`/`height` is display-only |
| Compress the output to a target JPEG quality | ❌ | No CSS API for encoding |
| Draggable handles that snap to the image boundary | ⚠️ | CSS `resize` exists but cannot be constrained to a parent element's bounds or expose its values programmatically |
| Read EXIF orientation and auto-rotate before display | ❌ | Requires decoding the binary JPEG header |
| Convert PNG with transparency to JPEG for smaller size | ❌ | Canvas `toBlob` with `image/jpeg` is the only browser path |

**Conclusion:** CSS handles the visible overlay and handle styling. The Canvas 2D API owns all pixel operations: read, crop, scale, encode.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- File picker -->
<label class="upload-trigger">
  Choose image
  <input type="file" accept="image/*" class="sr-only" id="image-input" />
</label>

<!-- Crop workspace — only shown after a file is chosen -->
<div class="crop-workspace" hidden>

  <!-- The source image, scaled to fit the preview area -->
  <div class="crop-preview-container">
    <img id="source-preview" class="crop-source-img" alt="Image to crop" />

    <!-- The draggable selection box sits on top of the image -->
    <div class="crop-selection" id="crop-selection">
      <!-- Eight handles: corners + edge midpoints -->
      <div class="crop-handle" data-handle="nw"></div>
      <div class="crop-handle" data-handle="n"></div>
      <div class="crop-handle" data-handle="ne"></div>
      <div class="crop-handle" data-handle="e"></div>
      <div class="crop-handle" data-handle="se"></div>
      <div class="crop-handle" data-handle="s"></div>
      <div class="crop-handle" data-handle="sw"></div>
      <div class="crop-handle" data-handle="w"></div>
      <!-- Darkened area outside the selection -->
      <div class="crop-mask crop-mask--top"    aria-hidden="true"></div>
      <div class="crop-mask crop-mask--bottom" aria-hidden="true"></div>
      <div class="crop-mask crop-mask--left"   aria-hidden="true"></div>
      <div class="crop-mask crop-mask--right"  aria-hidden="true"></div>
    </div>
  </div>

  <!-- Controls -->
  <div class="crop-controls">
    <label class="aspect-lock-label">
      <input type="checkbox" id="aspect-lock" />
      Lock aspect ratio
    </label>
    <span class="crop-dimensions" id="crop-dimensions">0 × 0 px</span>
    <button class="btn-crop" id="btn-apply-crop">Crop &amp; Upload</button>
  </div>

  <!-- Hidden canvas used for off-screen rendering -->
  <canvas id="crop-canvas" style="display:none"></canvas>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --handle-size: 10px;
  --handle-color: #ffffff;
  --selection-border: 2px solid rgba(255, 255, 255, 0.9);
  --mask-color: rgba(0, 0, 0, 0.55);
  --preview-max-width: 800px;
  --preview-max-height: 500px;
}

/* ─── Upload trigger ─── */
.upload-trigger {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0.6rem 1.4rem;
  background: #0066cc;
  color: #fff;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.95rem;
  user-select: none;
}

.upload-trigger:focus-within {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}

/* ─── Preview container ─── */
.crop-preview-container {
  position: relative;
  display: inline-block;   /* shrinks to image size */
  max-width: var(--preview-max-width);
  overflow: hidden;        /* keeps masks clipped */
  cursor: crosshair;
  touch-action: none;      /* prevents browser pan/zoom from stealing pointer events */
}

.crop-source-img {
  display: block;
  max-width: var(--preview-max-width);
  max-height: var(--preview-max-height);
  width: auto;
  height: auto;
}

/* ─── Selection box ─── */
.crop-selection {
  position: absolute;
  border: var(--selection-border);
  box-sizing: border-box;
  cursor: move;

  /* Rule-of-thirds grid overlay */
  background:
    linear-gradient(rgba(255,255,255,0.15) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255,255,255,0.15) 1px, transparent 1px);
  background-size: 33.33% 33.33%;
}

/* ─── Handles ─── */
.crop-handle {
  position: absolute;
  width: var(--handle-size);
  height: var(--handle-size);
  background: var(--handle-color);
  border: 1px solid rgba(0,0,0,0.4);
  border-radius: 1px;
  box-sizing: border-box;
}

/* Position each handle at its corner or edge midpoint */
.crop-handle[data-handle="nw"] { top: calc(var(--handle-size) / -2); left: calc(var(--handle-size) / -2); cursor: nw-resize; }
.crop-handle[data-handle="n"]  { top: calc(var(--handle-size) / -2); left: 50%; transform: translateX(-50%); cursor: n-resize; }
.crop-handle[data-handle="ne"] { top: calc(var(--handle-size) / -2); right: calc(var(--handle-size) / -2); cursor: ne-resize; }
.crop-handle[data-handle="e"]  { top: 50%; right: calc(var(--handle-size) / -2); transform: translateY(-50%); cursor: e-resize; }
.crop-handle[data-handle="se"] { bottom: calc(var(--handle-size) / -2); right: calc(var(--handle-size) / -2); cursor: se-resize; }
.crop-handle[data-handle="s"]  { bottom: calc(var(--handle-size) / -2); left: 50%; transform: translateX(-50%); cursor: s-resize; }
.crop-handle[data-handle="sw"] { bottom: calc(var(--handle-size) / -2); left: calc(var(--handle-size) / -2); cursor: sw-resize; }
.crop-handle[data-handle="w"]  { top: 50%; left: calc(var(--handle-size) / -2); transform: translateY(-50%); cursor: w-resize; }

/* ─── Masks (dim area outside selection) ─── */
.crop-mask {
  position: fixed;     /* positioned relative to .crop-selection via JS inline styles */
  background: var(--mask-color);
  pointer-events: none;
}

/* ─── Controls bar ─── */
.crop-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem 0;
  flex-wrap: wrap;
}

.aspect-lock-label {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.9rem;
  cursor: pointer;
  user-select: none;
}

.crop-dimensions {
  font-size: 0.85rem;
  color: #666;
  font-variant-numeric: tabular-nums;
  min-width: 9ch;
}

.btn-crop {
  margin-left: auto;
  padding: 0.5rem 1.2rem;
  background: #28a745;
  color: #fff;
  border: none;
  border-radius: 4px;
  font-size: 0.95rem;
  cursor: pointer;
}

.btn-crop:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* ─── Screen-reader only utility ─── */
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0,0,0,0);
  white-space: nowrap;
  border: 0;
}
```

**What CSS owns:** handle positioning and cursor shapes, the selection border and rule-of-thirds overlay, mask shading, responsive scaling of the preview image.

**What CSS cannot own:** reading pixel data, tracking pointer position relative to the image coordinate space, constraining the selection to image bounds, computing output dimensions, encoding the Blob.

---

## React Implementation

### Types

```tsx
// types.ts
export interface CropRect {
  x: number;      // pixels in the *natural* image coordinate space
  y: number;
  width: number;
  height: number;
}

export interface CropState {
  // Selection in display-space (px relative to the preview element)
  display: { x: number; y: number; width: number; height: number };
  // Selection in image-space (natural image pixels)
  natural: CropRect;
}

export type HandleId = 'nw' | 'n' | 'ne' | 'e' | 'se' | 's' | 'sw' | 'w' | 'move';
```

### The Core Hook

```tsx
// useCrop.ts
import { useState, useCallback, useRef } from 'react';
import type { CropState, HandleId } from './types';

const MIN_SIZE = 20;  // px in display space

interface UseCropOptions {
  aspectRatio: number | null;   // null = free, e.g. 16/9 for locked
}

export function useCrop(
  imgRef: React.RefObject<HTMLImageElement>,
  options: UseCropOptions
) {
  const [crop, setCrop] = useState<CropState | null>(null);
  const dragRef = useRef<{
    handle: HandleId;
    startX: number;
    startY: number;
    startCrop: CropState['display'];
  } | null>(null);

  // Convert display-space rect → natural image pixels
  const toNatural = useCallback(
    (display: CropState['display']): CropState['natural'] => {
      const img = imgRef.current;
      if (!img) return { x: 0, y: 0, width: 0, height: 0 };
      const scaleX = img.naturalWidth  / img.clientWidth;
      const scaleY = img.naturalHeight / img.clientHeight;
      return {
        x:      Math.round(display.x      * scaleX),
        y:      Math.round(display.y      * scaleY),
        width:  Math.round(display.width  * scaleX),
        height: Math.round(display.height * scaleY),
      };
    },
    [imgRef]
  );

  // Clamp a display-space rect inside the image bounds
  const clamp = useCallback(
    (d: CropState['display']): CropState['display'] => {
      const img = imgRef.current;
      if (!img) return d;
      const maxW = img.clientWidth;
      const maxH = img.clientHeight;
      const x = Math.max(0, Math.min(d.x, maxW - MIN_SIZE));
      const y = Math.max(0, Math.min(d.y, maxH - MIN_SIZE));
      const width  = Math.max(MIN_SIZE, Math.min(d.width,  maxW - x));
      const height = Math.max(MIN_SIZE, Math.min(d.height, maxH - y));
      return { x, y, width, height };
    },
    [imgRef]
  );

  // Apply aspect-ratio lock: adjust height to match ratio
  const applyAspect = useCallback(
    (d: CropState['display']): CropState['display'] => {
      if (!options.aspectRatio) return d;
      return { ...d, height: d.width / options.aspectRatio };
    },
    [options.aspectRatio]
  );

  const onPointerDown = useCallback(
    (e: React.PointerEvent, handle: HandleId) => {
      e.preventDefault();
      (e.target as HTMLElement).setPointerCapture(e.pointerId);
      dragRef.current = {
        handle,
        startX: e.clientX,
        startY: e.clientY,
        startCrop: crop?.display ?? { x: 0, y: 0, width: 100, height: 100 },
      };
    },
    [crop]
  );

  const onPointerMove = useCallback(
    (e: React.PointerEvent) => {
      if (!dragRef.current) return;
      const { handle, startX, startY, startCrop } = dragRef.current;
      const dx = e.clientX - startX;
      const dy = e.clientY - startY;
      let next = { ...startCrop };

      switch (handle) {
        case 'move':
          next.x += dx;
          next.y += dy;
          break;
        case 'se':
          next.width  = Math.max(MIN_SIZE, startCrop.width  + dx);
          next.height = Math.max(MIN_SIZE, startCrop.height + dy);
          break;
        case 'sw':
          next.x      = startCrop.x + dx;
          next.width  = Math.max(MIN_SIZE, startCrop.width  - dx);
          next.height = Math.max(MIN_SIZE, startCrop.height + dy);
          break;
        case 'ne':
          next.width  = Math.max(MIN_SIZE, startCrop.width  + dx);
          next.y      = startCrop.y + dy;
          next.height = Math.max(MIN_SIZE, startCrop.height - dy);
          break;
        case 'nw':
          next.x      = startCrop.x + dx;
          next.y      = startCrop.y + dy;
          next.width  = Math.max(MIN_SIZE, startCrop.width  - dx);
          next.height = Math.max(MIN_SIZE, startCrop.height - dy);
          break;
        case 'n':
          next.y      = startCrop.y + dy;
          next.height = Math.max(MIN_SIZE, startCrop.height - dy);
          break;
        case 's':
          next.height = Math.max(MIN_SIZE, startCrop.height + dy);
          break;
        case 'e':
          next.width  = Math.max(MIN_SIZE, startCrop.width  + dx);
          break;
        case 'w':
          next.x      = startCrop.x + dx;
          next.width  = Math.max(MIN_SIZE, startCrop.width  - dx);
          break;
      }

      const clamped = clamp(applyAspect(next));
      setCrop({ display: clamped, natural: toNatural(clamped) });
    },
    [clamp, applyAspect, toNatural]
  );

  const onPointerUp = useCallback(() => {
    dragRef.current = null;
  }, []);

  // Initialize selection to full image on first load
  const initCrop = useCallback(() => {
    const img = imgRef.current;
    if (!img) return;
    const display = { x: 0, y: 0, width: img.clientWidth, height: img.clientHeight };
    setCrop({ display, natural: toNatural(display) });
  }, [imgRef, toNatural]);

  return { crop, initCrop, onPointerDown, onPointerMove, onPointerUp };
}
```

### The Export Utility

```tsx
// cropAndExport.ts

export interface ExportOptions {
  maxWidth: number;
  maxHeight: number;
  mimeType: 'image/jpeg' | 'image/webp' | 'image/png';
  quality: number;   // 0–1, ignored for PNG
}

/**
 * Draws the selected sub-rectangle of `source` onto an off-screen canvas,
 * scales it down to fit within maxWidth × maxHeight, then exports as a Blob.
 */
export function cropAndExport(
  source: HTMLImageElement,
  natural: { x: number; y: number; width: number; height: number },
  opts: ExportOptions
): Promise<Blob> {
  return new Promise((resolve, reject) => {
    // Compute output dimensions that fit within the max box
    const aspectRatio = natural.width / natural.height;
    let outWidth  = natural.width;
    let outHeight = natural.height;

    if (outWidth > opts.maxWidth) {
      outWidth  = opts.maxWidth;
      outHeight = Math.round(opts.maxWidth / aspectRatio);
    }
    if (outHeight > opts.maxHeight) {
      outHeight = opts.maxHeight;
      outWidth  = Math.round(opts.maxHeight * aspectRatio);
    }

    const canvas = document.createElement('canvas');
    canvas.width  = outWidth;
    canvas.height = outHeight;

    const ctx = canvas.getContext('2d');
    if (!ctx) { reject(new Error('Canvas 2D context unavailable')); return; }

    // Enable image smoothing for high-quality downscaling
    ctx.imageSmoothingEnabled = true;
    ctx.imageSmoothingQuality = 'high';

    // The key call: read sub-rect from source, write to full canvas
    ctx.drawImage(
      source,
      natural.x,      // source x
      natural.y,      // source y
      natural.width,  // source width
      natural.height, // source height
      0,              // destination x
      0,              // destination y
      outWidth,       // destination width
      outHeight       // destination height
    );

    canvas.toBlob(
      (blob) => {
        if (!blob) { reject(new Error('toBlob returned null')); return; }
        resolve(blob);
      },
      opts.mimeType,
      opts.quality
    );
  });
}
```

### The Full Component

```tsx
// ImageCropUpload.tsx
import { useState, useRef, useCallback } from 'react';
import { useCrop } from './useCrop';
import { cropAndExport } from './cropAndExport';
import type { HandleId } from './types';

const HANDLES: HandleId[] = ['nw', 'n', 'ne', 'e', 'se', 's', 'sw', 'w'];
const MAX_WIDTH  = 1920;
const MAX_HEIGHT = 1080;

export function ImageCropUpload() {
  const [objectUrl, setObjectUrl]     = useState<string | null>(null);
  const [aspectLocked, setAspectLock] = useState(false);
  const [status, setStatus]           = useState<'idle' | 'uploading' | 'done' | 'error'>('idle');
  const imgRef   = useRef<HTMLImageElement>(null);
  const prevUrl  = useRef<string | null>(null);

  const aspectRatio = aspectLocked && imgRef.current
    ? imgRef.current.naturalWidth / imgRef.current.naturalHeight
    : null;

  const { crop, initCrop, onPointerDown, onPointerMove, onPointerUp } = useCrop(
    imgRef,
    { aspectRatio }
  );

  const onFileChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    // Revoke previous object URL to avoid memory leak
    if (prevUrl.current) URL.revokeObjectURL(prevUrl.current);
    const url = URL.createObjectURL(file);
    prevUrl.current = url;
    setObjectUrl(url);
  }, []);

  const onImageLoad = useCallback(() => {
    initCrop();
  }, [initCrop]);

  const handleCropAndUpload = useCallback(async () => {
    if (!crop || !imgRef.current) return;
    setStatus('uploading');

    try {
      const blob = await cropAndExport(imgRef.current, crop.natural, {
        maxWidth:  MAX_WIDTH,
        maxHeight: MAX_HEIGHT,
        mimeType:  'image/jpeg',
        quality:   0.85,
      });

      const formData = new FormData();
      formData.append('image', blob, 'cropped.jpg');

      const response = await fetch('/api/upload', { method: 'POST', body: formData });
      if (!response.ok) throw new Error(`Upload failed: ${response.status}`);
      setStatus('done');
    } catch (err) {
      console.error(err);
      setStatus('error');
    }
  }, [crop]);

  const d = crop?.display;

  return (
    <div className="crop-workspace">
      <label className="upload-trigger">
        Choose image
        <input
          type="file"
          accept="image/*"
          className="sr-only"
          onChange={onFileChange}
        />
      </label>

      {objectUrl && (
        <>
          <div
            className="crop-preview-container"
            onPointerMove={onPointerMove}
            onPointerUp={onPointerUp}
            onPointerLeave={onPointerUp}
          >
            <img
              ref={imgRef}
              src={objectUrl}
              alt="Image to crop"
              className="crop-source-img"
              onLoad={onImageLoad}
              draggable={false}
            />

            {d && (
              <div
                className="crop-selection"
                style={{ left: d.x, top: d.y, width: d.width, height: d.height }}
                onPointerDown={(e) => onPointerDown(e, 'move')}
              >
                {HANDLES.map((handle) => (
                  <div
                    key={handle}
                    className="crop-handle"
                    data-handle={handle}
                    onPointerDown={(e) => {
                      e.stopPropagation(); // don't also trigger 'move'
                      onPointerDown(e, handle);
                    }}
                  />
                ))}
              </div>
            )}
          </div>

          <div className="crop-controls">
            <label className="aspect-lock-label">
              <input
                type="checkbox"
                checked={aspectLocked}
                onChange={(e) => setAspectLock(e.target.checked)}
              />
              Lock aspect ratio
            </label>

            {crop && (
              <span className="crop-dimensions" aria-live="polite" aria-atomic="true">
                {crop.natural.width} &times; {crop.natural.height} px
              </span>
            )}

            <button
              className="btn-crop"
              onClick={handleCropAndUpload}
              disabled={!crop || status === 'uploading'}
            >
              {status === 'uploading' ? 'Uploading…' : 'Crop & Upload'}
            </button>

            {status === 'done'  && <span role="status">Upload complete.</span>}
            {status === 'error' && <span role="alert">Upload failed. Try again.</span>}
          </div>
        </>
      )}
    </div>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// image-crop.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface NaturalRect { x: number; y: number; width: number; height: number; }
export interface DisplayRect { x: number; y: number; width: number; height: number; }

@Injectable({ providedIn: 'root' })
export class ImageCropService {
  private _display = signal<DisplayRect>({ x: 0, y: 0, width: 0, height: 0 });
  private _imgSize = signal<{ w: number; h: number; nw: number; nh: number } | null>(null);

  readonly display    = this._display.asReadonly();
  readonly natural    = computed<NaturalRect | null>(() => {
    const img = this._imgSize();
    if (!img) return null;
    const d    = this._display();
    const sx   = img.nw / img.w;
    const sy   = img.nh / img.h;
    return {
      x:      Math.round(d.x      * sx),
      y:      Math.round(d.y      * sy),
      width:  Math.round(d.width  * sx),
      height: Math.round(d.height * sy),
    };
  });

  setImageSize(clientW: number, clientH: number, naturalW: number, naturalH: number) {
    this._imgSize.set({ w: clientW, h: clientH, nw: naturalW, nh: naturalH });
    this._display.set({ x: 0, y: 0, width: clientW, height: clientH });
  }

  updateDisplay(rect: DisplayRect) {
    this._display.set(rect);
  }
}
```

### Component

```typescript
// image-crop-upload.component.ts
import {
  Component, ViewChild, ElementRef,
  signal, computed, inject, OnDestroy
} from '@angular/core';
import { NgIf, NgFor } from '@angular/common';
import { ImageCropService, DisplayRect } from './image-crop.service';

type HandleId = 'nw' | 'n' | 'ne' | 'e' | 'se' | 's' | 'sw' | 'w' | 'move';

@Component({
  selector: 'app-image-crop-upload',
  standalone: true,
  imports: [NgIf, NgFor],
  template: `
    <label class="upload-trigger">
      Choose image
      <input type="file" accept="image/*" class="sr-only" (change)="onFileChange($event)" />
    </label>

    <ng-container *ngIf="objectUrl()">
      <div
        class="crop-preview-container"
        (pointermove)="onPointerMove($event)"
        (pointerup)="onPointerUp()"
        (pointerleave)="onPointerUp()"
      >
        <img
          #imgEl
          [src]="objectUrl()"
          alt="Image to crop"
          class="crop-source-img"
          (load)="onImageLoad()"
          draggable="false"
        />

        <div
          *ngIf="cropService.display() as d"
          class="crop-selection"
          [style.left.px]="d.x"
          [style.top.px]="d.y"
          [style.width.px]="d.width"
          [style.height.px]="d.height"
          (pointerdown)="startDrag($event, 'move')"
        >
          <div
            *ngFor="let handle of handles"
            class="crop-handle"
            [attr.data-handle]="handle"
            (pointerdown)="onHandleDown($event, handle)"
          ></div>
        </div>
      </div>

      <div class="crop-controls">
        <label class="aspect-lock-label">
          <input type="checkbox" [checked]="aspectLocked()" (change)="toggleAspect()" />
          Lock aspect ratio
        </label>

        <span class="crop-dimensions" aria-live="polite" aria-atomic="true">
          {{ cropService.natural()?.width }} &times; {{ cropService.natural()?.height }} px
        </span>

        <button
          class="btn-crop"
          [disabled]="status() === 'uploading'"
          (click)="cropAndUpload()"
        >
          {{ status() === 'uploading' ? 'Uploading…' : 'Crop & Upload' }}
        </button>

        <span *ngIf="status() === 'done'"  role="status">Upload complete.</span>
        <span *ngIf="status() === 'error'" role="alert">Upload failed.</span>
      </div>
    </ng-container>
  `,
})
export class ImageCropUploadComponent implements OnDestroy {
  @ViewChild('imgEl') imgElRef!: ElementRef<HTMLImageElement>;

  cropService = inject(ImageCropService);

  objectUrl   = signal<string | null>(null);
  aspectLocked = signal(false);
  status      = signal<'idle' | 'uploading' | 'done' | 'error'>('idle');
  handles: HandleId[] = ['nw', 'n', 'ne', 'e', 'se', 's', 'sw', 'w'];

  private prevUrl: string | null = null;
  private drag: {
    handle: HandleId;
    startX: number; startY: number;
    startCrop: DisplayRect;
  } | null = null;

  onFileChange(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;
    if (this.prevUrl) URL.revokeObjectURL(this.prevUrl);
    const url = URL.createObjectURL(file);
    this.prevUrl = url;
    this.objectUrl.set(url);
  }

  onImageLoad() {
    const img = this.imgElRef.nativeElement;
    this.cropService.setImageSize(
      img.clientWidth, img.clientHeight,
      img.naturalWidth, img.naturalHeight
    );
  }

  toggleAspect() {
    this.aspectLocked.update(v => !v);
  }

  onHandleDown(e: PointerEvent, handle: HandleId) {
    e.stopPropagation();
    this.startDrag(e, handle);
  }

  startDrag(e: PointerEvent, handle: HandleId) {
    e.preventDefault();
    (e.target as HTMLElement).setPointerCapture(e.pointerId);
    this.drag = {
      handle,
      startX: e.clientX,
      startY: e.clientY,
      startCrop: { ...this.cropService.display() },
    };
  }

  onPointerMove(e: PointerEvent) {
    if (!this.drag) return;
    const { handle, startX, startY, startCrop } = this.drag;
    const dx = e.clientX - startX;
    const dy = e.clientY - startY;
    const MIN = 20;
    let next: DisplayRect = { ...startCrop };

    if (handle === 'move') { next.x += dx; next.y += dy; }
    else if (handle === 'se') { next.width = Math.max(MIN, startCrop.width + dx); next.height = Math.max(MIN, startCrop.height + dy); }
    else if (handle === 'e')  { next.width = Math.max(MIN, startCrop.width + dx); }
    else if (handle === 's')  { next.height = Math.max(MIN, startCrop.height + dy); }
    else if (handle === 'n')  { next.y = startCrop.y + dy; next.height = Math.max(MIN, startCrop.height - dy); }
    else if (handle === 'w')  { next.x = startCrop.x + dx; next.width = Math.max(MIN, startCrop.width - dx); }
    else if (handle === 'nw') { next.x = startCrop.x + dx; next.y = startCrop.y + dy; next.width = Math.max(MIN, startCrop.width - dx); next.height = Math.max(MIN, startCrop.height - dy); }
    else if (handle === 'ne') { next.y = startCrop.y + dy; next.width = Math.max(MIN, startCrop.width + dx); next.height = Math.max(MIN, startCrop.height - dy); }
    else if (handle === 'sw') { next.x = startCrop.x + dx; next.width = Math.max(MIN, startCrop.width - dx); next.height = Math.max(MIN, startCrop.height + dy); }

    if (this.aspectLocked()) {
      const img = this.imgElRef.nativeElement;
      const ratio = img.naturalWidth / img.naturalHeight;
      next.height = next.width / ratio;
    }

    this.cropService.updateDisplay(next);
  }

  onPointerUp() { this.drag = null; }

  async cropAndUpload() {
    const natural = this.cropService.natural();
    if (!natural) return;
    this.status.set('uploading');

    const img = this.imgElRef.nativeElement;
    const MAX_W = 1920, MAX_H = 1080;
    const aspect = natural.width / natural.height;
    let outW = natural.width, outH = natural.height;
    if (outW > MAX_W) { outW = MAX_W; outH = Math.round(MAX_W / aspect); }
    if (outH > MAX_H) { outH = MAX_H; outW = Math.round(MAX_H * aspect); }

    const canvas = document.createElement('canvas');
    canvas.width = outW; canvas.height = outH;
    const ctx = canvas.getContext('2d')!;
    ctx.imageSmoothingEnabled = true;
    ctx.imageSmoothingQuality = 'high';
    ctx.drawImage(img, natural.x, natural.y, natural.width, natural.height, 0, 0, outW, outH);

    canvas.toBlob(async (blob) => {
      if (!blob) { this.status.set('error'); return; }
      const formData = new FormData();
      formData.append('image', blob, 'cropped.jpg');
      try {
        const res = await fetch('/api/upload', { method: 'POST', body: formData });
        this.status.set(res.ok ? 'done' : 'error');
      } catch { this.status.set('error'); }
    }, 'image/jpeg', 0.85);
  }

  ngOnDestroy() {
    if (this.prevUrl) URL.revokeObjectURL(this.prevUrl);
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| File input has an accessible label | Wrapped in `<label>` — clicking the label text activates the input |
| Hidden file input is still keyboard reachable | `.sr-only` positions it off-screen but does not set `display:none` or `visibility:hidden` |
| Crop handles have cursor hints | CSS `cursor: nw-resize` etc. per `data-handle` attribute |
| Crop dimension display updates without reading noise | `aria-live="polite"` + `aria-atomic="true"` — announces only when the user stops dragging |
| Upload state announced to screen readers | `role="status"` for success, `role="alert"` for error |
| Drag-to-crop works with pointer events, not mouse events | `pointerdown/pointermove/pointerup` — unified across mouse, pen, touch |
| `touch-action: none` on the preview container | Prevents browser scroll/zoom from stealing pointer events during crop drag |
| `setPointerCapture` on drag start | Keeps pointer events flowing to the handle even if the cursor leaves it |
| Canvas element is hidden from assistive technology | `display:none` — it is a processing surface, not content |
| Object URL revoked on component destroy | Prevents memory leak; `ngOnDestroy` / `useEffect` cleanup |

---

## Production Pitfalls

**1. EXIF orientation makes the canvas crop the wrong region**
JPEG files shot in portrait on a phone embed an EXIF orientation tag. Browsers auto-apply the rotation when rendering into an `<img>`, but `canvas.drawImage()` reads the raw bytes — so your coordinates point to the rotated image, not the displayed one. Fix: use the `exifr` library or a small EXIF parser to read the orientation value and pre-rotate the canvas context with `ctx.rotate()` before drawing.

**2. Coordinate space confusion between CSS display pixels and natural pixels**
The `img.clientWidth` is the rendered size; `img.naturalWidth` is the raw file size. A 4000 × 3000 px image displayed at 800 × 600 px has a scale factor of 5×. All drag coordinates come in display space and must be multiplied by the scale before passing to `drawImage`. A single conversion utility function (`toNatural`) isolates this — never scatter the multiplication throughout the codebase.

**3. `canvas.toBlob()` returns `null` in some environments**
`toBlob` returns `null` when the canvas is "tainted" (cross-origin image drawn without CORS headers) or when the browser runs out of memory for very large images. Always guard with `if (!blob) throw`. For cross-origin images, set `img.crossOrigin = 'anonymous'` before assigning `src`, and ensure the server sends `Access-Control-Allow-Origin`.

**4. Memory leak from unrevokedObject URLs**
`URL.createObjectURL()` keeps the file's data alive in the browser's memory until `URL.revokeObjectURL()` is explicitly called. If the user picks a new file before the previous URL is revoked, the old Blob stays in memory. Always revoke the old URL before creating a new one, and revoke on component unmount.

**5. Downscaling in a single pass produces blurry output**
`imageSmoothingQuality = 'high'` helps, but for extreme downscales (e.g. 4000 px → 200 px) a single `drawImage` pass loses sharpness because the browser's bicubic filter cannot compensate. Fix: downscale in steps, halving dimensions each time, until you reach the target size. Each intermediate step preserves more detail.

**6. Large images freeze the main thread during `toBlob`**
`toBlob` is synchronous on the compositor but yields a callback — the encoding itself happens off-thread in most browsers. However, `drawImage` with a very large source is synchronous and can take 100–300 ms for a 20 MP image. Fix: use `createImageBitmap()` first, which decodes and scales the image off the main thread, then draw the resulting `ImageBitmap` onto the canvas.

**7. Aspect-ratio lock drifts on resize handles**
When only height-affecting handles (north, south) are dragged with aspect lock on, you must adjust the opposing dimension. If you only adjust height when `width` changes (common shortcut), dragging the south handle alone will not update width, breaking the lock. Every handle that changes height must also recompute width, and vice versa.

---

## Interview Angle

**Q: "Walk me through how you'd implement a client-side image cropper that resizes before upload."**

A strong answer covers five layers: (1) get the File from `<input>` and create an object URL to display it in an `<img>` tag; (2) overlay a draggable selection box using pointer events, storing the selection in display-space pixels and converting to natural-image pixels on each update using the `clientWidth / naturalWidth` scale ratio; (3) on confirm, draw the sub-rectangle onto an off-screen canvas using the nine-argument `drawImage` overload, first scaling down the output dimensions if they exceed the upload max; (4) call `canvas.toBlob('image/jpeg', 0.85)` to produce a compressed Blob; (5) append the Blob to `FormData` and `fetch` to the upload endpoint. Name at least two production hazards: EXIF orientation and main-thread jank from synchronous `drawImage` on large sources.

**Follow-up: "How does the aspect-ratio lock actually work?"**

The lock stores the original width-to-height ratio when the checkbox is checked. On every pointer move event, after computing the new width from the drag delta, height is overwritten as `width / ratio`. The clamp step runs after the ratio correction so the locked selection cannot escape image bounds. The tricky case is handles that primarily move the top or left edge — you must decide whether to anchor the opposite edge (scale toward bottom-right) or recompute in both directions (scale toward center). Anchoring the opposite edge is the industry standard behavior (used by Figma, Sketch, and most crop UIs).
