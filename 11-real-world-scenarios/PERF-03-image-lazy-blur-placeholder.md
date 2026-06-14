# Image Lazy Loading with Blur-Up Placeholder

## The Idea

**In plain English:** When a page has many images, loading all of them at once wastes bandwidth and delays the initial render. The blur-up technique solves two problems simultaneously: it defers the full image until it is near the viewport (`loading="lazy"`), and it fills the space immediately with a tiny, blurry stand-in so the page does not look broken while the real image loads. When the full image arrives, the blur fades away smoothly.

**Real-world analogy:** Think of developing a photograph in a darkroom.

- The **blank white paper** before any chemistry = no placeholder at all — just empty space.
- The **faint grey ghost image** that appears in the first few seconds of development = the LQIP (Low Quality Image Placeholder). It is rough, blurry, just enough for the viewer to know what is coming.
- The **fully developed print** that emerges over the next 30 seconds = the real, full-resolution image loading in.
- The **enlarger's aperture** you keep closed until you are ready = the `loading="lazy"` attribute; you do not waste chemistry on prints you are not looking at yet.
- The **aspect-ratio frame** you mount the paper in before development begins = the CSS aspect-ratio box; the frame is always the right size so nothing shifts when the print appears.

The key insight: the LQIP and the full image are not the same request — they are two separate layers that cross-fade. The user always sees *something* in the right space, at the right size, with no layout shift.

---

## Learning Objectives

- Understand what LQIP is and the two ways to deliver it (inline base64 vs. separate tiny URL)
- Prevent Cumulative Layout Shift (CLS) using an aspect-ratio box or padding-bottom hack
- Use the native `loading="lazy"` and `decoding="async"` attributes correctly
- Implement the CSS `filter: blur()` cross-fade transition
- Know when the Intersection Observer API is needed instead of native lazy loading
- Build a production-ready `LazyImage` component in React (TypeScript)
- Build the equivalent in Angular using signals and a standalone component
- Understand LQIP generation pipelines (Sharp, Plaiceholder, Thumbhash)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Reserve the image's space before it loads | ✅ with `aspect-ratio` or padding trick | — |
| Apply and animate `filter: blur()` | ✅ | — |
| Defer loading until near viewport | ❌ | CSS has no network-request control |
| Know when the full image has finished loading | ❌ | Requires the `load` event on `<img>` |
| Remove the blur only after the image is ready | ❌ | Needs JS to listen to `onload` and toggle a class |
| Generate the LQIP at build time | ❌ | Requires a Node.js pipeline (Sharp, Plaiceholder) |
| Polyfill `loading="lazy"` in older browsers | ❌ | Requires Intersection Observer in JS |
| Fade out placeholder, fade in real image simultaneously | ⚠️ | CSS can do the animation, but JS must trigger the class swap |

**Conclusion:** CSS owns the blur animation, the aspect-ratio box, and the transition timing. JS owns the load-event detection, the class swap, and the Intersection Observer fallback.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  The wrapper reserves space via aspect-ratio.
  The placeholder (LQIP) sits behind the real image.
  Once the real image fires its load event, JS adds .is-loaded
  to the wrapper, which fades out the blur.
-->
<div class="lazy-img-wrapper" style="--aspect-ratio: 16/9;">

  <!-- LQIP: tiny inline base64 image, always visible immediately -->
  <img
    class="lazy-img__placeholder"
    src="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/..."
    alt=""
    aria-hidden="true"
  />

  <!-- Real image: loaded lazily, decoded async -->
  <img
    class="lazy-img__full"
    src="/images/hero.jpg"
    srcset="/images/hero-480.jpg 480w, /images/hero-960.jpg 960w, /images/hero.jpg 1440w"
    sizes="(max-width: 600px) 100vw, 50vw"
    alt="A sunrise over mountain peaks"
    loading="lazy"
    decoding="async"
    width="1440"
    height="810"
  />

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --lqip-blur:       20px;
  --lqip-scale:      1.05;    /* slightly oversized to hide blur edges */
  --fade-duration:   0.4s;
  --fade-easing:     cubic-bezier(0.4, 0, 0.2, 1);
}

/* ─── Wrapper: reserves space, clips the scaled placeholder ─── */
.lazy-img-wrapper {
  position: relative;
  overflow: hidden;               /* clips the scaled-up placeholder */
  background: #e9e9e9;            /* fallback if LQIP also fails */
  aspect-ratio: var(--aspect-ratio, 16 / 9);

  /*
    Older browsers that don't support aspect-ratio:
    use the padding-bottom trick as a fallback.
    aspect-ratio is supported in all modern browsers (Chrome 88+, Firefox 89+, Safari 15+).
  */
}

/* ─── Placeholder: always visible, blurred, slightly scaled ─── */
.lazy-img__placeholder {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  filter: blur(var(--lqip-blur));
  transform: scale(var(--lqip-scale));  /* hides blur edge artifacts */
  transition:
    opacity var(--fade-duration) var(--fade-easing),
    filter  var(--fade-duration) var(--fade-easing);
}

/* ─── Real image: invisible until loaded ─── */
.lazy-img__full {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0;
  transition: opacity var(--fade-duration) var(--fade-easing);
}

/* ─── Loaded state: real image fades in, placeholder fades out ─── */
.lazy-img-wrapper.is-loaded .lazy-img__full {
  opacity: 1;
}

.lazy-img-wrapper.is-loaded .lazy-img__placeholder {
  opacity: 0;
}

/* ─── Reduced motion: skip the cross-fade, just swap instantly ─── */
@media (prefers-reduced-motion: reduce) {
  .lazy-img__placeholder,
  .lazy-img__full {
    transition: none;
  }
}
```

**What CSS owns:** aspect-ratio box, blur filter, scale trick to hide edge artifacts, cross-fade timing, reduced-motion fallback.

**What CSS cannot own:** detecting when the full image has loaded, adding `.is-loaded` to the wrapper, deferring the network request.

---

## React Implementation

### Types

```tsx
// lazy-image.types.ts
export interface LazyImageProps {
  src: string;
  srcSet?: string;
  sizes?: string;
  alt: string;
  lqip: string;                  // base64 data URI or tiny URL
  width: number;
  height: number;
  className?: string;
  onLoad?: () => void;
}
```

### The Hook — Load Detection

```tsx
// useLazyImage.ts
import { useState, useCallback, useRef } from 'react';

interface UseLazyImageReturn {
  isLoaded: boolean;
  imgRef: React.RefObject<HTMLImageElement>;
  handleLoad: () => void;
  handleError: () => void;
}

export function useLazyImage(): UseLazyImageReturn {
  const [isLoaded, setIsLoaded] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);

  /*
    Check synchronously: if the browser already has the image
    cached, the load event fires before React attaches the handler.
    In that case, img.complete is already true on mount.
  */
  const handleLoad = useCallback(() => {
    setIsLoaded(true);
  }, []);

  /*
    On error, still mark as "loaded" so the placeholder doesn't
    linger forever. The broken-image icon is less disruptive than
    an eternal blur.
  */
  const handleError = useCallback(() => {
    setIsLoaded(true);
  }, []);

  return { isLoaded, imgRef, handleLoad, handleError };
}
```

### The Component

```tsx
// LazyImage.tsx
import { useEffect } from 'react';
import { useLazyImage } from './useLazyImage';
import type { LazyImageProps } from './lazy-image.types';

export function LazyImage({
  src,
  srcSet,
  sizes,
  alt,
  lqip,
  width,
  height,
  className = '',
  onLoad,
}: LazyImageProps) {
  const { isLoaded, imgRef, handleLoad, handleError } = useLazyImage();

  const aspectRatio = `${width} / ${height}`;

  /*
    Handle the cached-image case: if the browser already has the
    image in cache, the `load` event fires synchronously during
    DOM construction before React can attach onLoad. We check
    img.complete after mount to cover this edge case.
  */
  useEffect(() => {
    const img = imgRef.current;
    if (img?.complete) {
      handleLoad();
    }
  }, [imgRef, handleLoad]);

  useEffect(() => {
    if (isLoaded) onLoad?.();
  }, [isLoaded, onLoad]);

  return (
    <div
      className={`lazy-img-wrapper ${isLoaded ? 'is-loaded' : ''} ${className}`}
      style={{ '--aspect-ratio': aspectRatio } as React.CSSProperties}
    >
      {/* LQIP: aria-hidden because it is purely decorative */}
      <img
        className="lazy-img__placeholder"
        src={lqip}
        alt=""
        aria-hidden="true"
      />

      {/* Full image */}
      <img
        ref={imgRef}
        className="lazy-img__full"
        src={src}
        srcSet={srcSet}
        sizes={sizes}
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
        decoding="async"
        onLoad={handleLoad}
        onError={handleError}
      />
    </div>
  );
}
```

### Usage in a Gallery

```tsx
// ImageGallery.tsx
import { LazyImage } from './LazyImage';

interface GalleryItem {
  id: string;
  src: string;
  srcSet: string;
  lqip: string;       // generated at build time with Sharp / Plaiceholder
  alt: string;
  width: number;
  height: number;
}

const items: GalleryItem[] = [
  {
    id: 'hero',
    src: '/images/hero.jpg',
    srcSet: '/images/hero-480.jpg 480w, /images/hero-960.jpg 960w, /images/hero.jpg 1440w',
    lqip: 'data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAgGBg...',
    alt: 'Sunrise over mountain peaks',
    width: 1440,
    height: 810,
  },
  // ... more items
];

export function ImageGallery() {
  return (
    <ul className="gallery" role="list">
      {items.map((item) => (
        <li key={item.id}>
          <LazyImage
            src={item.src}
            srcSet={item.srcSet}
            lqip={item.lqip}
            alt={item.alt}
            width={item.width}
            height={item.height}
            sizes="(max-width: 600px) 100vw, 50vw"
          />
        </li>
      ))}
    </ul>
  );
}
```

### Intersection Observer Fallback (Older Browsers)

```tsx
// useIntersectionLazy.ts
// Use this only when you need to support browsers that do not honour
// loading="lazy" (pre-Chromium Edge, some WebViews).
import { useState, useRef, useEffect } from 'react';

export function useIntersectionLazy(rootMargin = '200px') {
  const [shouldLoad, setShouldLoad] = useState(false);
  const sentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = sentinelRef.current;
    if (!el) return;

    // Native lazy loading is supported — no need for observer
    if ('loading' in HTMLImageElement.prototype) {
      setShouldLoad(true);
      return;
    }

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setShouldLoad(true);
          observer.disconnect();
        }
      },
      { rootMargin }
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [rootMargin]);

  return { shouldLoad, sentinelRef };
}
```

---

## Angular Implementation

### Model

```typescript
// lazy-image.model.ts
export interface LazyImageConfig {
  src: string;
  srcSet?: string;
  sizes?: string;
  alt: string;
  lqip: string;
  width: number;
  height: number;
}
```

### Standalone Component

```typescript
// lazy-image.component.ts
import {
  Component,
  Input,
  OnInit,
  AfterViewInit,
  ElementRef,
  ViewChild,
  ChangeDetectionStrategy,
  signal,
  computed,
  inject,
} from '@angular/core';
import { NgClass, NgStyle } from '@angular/common';
import type { LazyImageConfig } from './lazy-image.model';

@Component({
  selector: 'app-lazy-image',
  standalone: true,
  imports: [NgClass, NgStyle],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div
      class="lazy-img-wrapper"
      [class.is-loaded]="isLoaded()"
      [ngStyle]="wrapperStyle()"
    >
      <!-- LQIP placeholder -->
      <img
        class="lazy-img__placeholder"
        [src]="config.lqip"
        alt=""
        aria-hidden="true"
      />

      <!-- Full image -->
      <img
        #fullImg
        class="lazy-img__full"
        [src]="config.src"
        [attr.srcset]="config.srcSet ?? null"
        [attr.sizes]="config.sizes ?? null"
        [alt]="config.alt"
        [attr.width]="config.width"
        [attr.height]="config.height"
        loading="lazy"
        decoding="async"
        (load)="onLoad()"
        (error)="onError()"
      />
    </div>
  `,
})
export class LazyImageComponent implements AfterViewInit {
  @Input({ required: true }) config!: LazyImageConfig;

  @ViewChild('fullImg') fullImgEl!: ElementRef<HTMLImageElement>;

  private _isLoaded = signal(false);

  readonly isLoaded = this._isLoaded.asReadonly();

  readonly wrapperStyle = computed(() => ({
    '--aspect-ratio': `${this.config.width} / ${this.config.height}`,
  }));

  ngAfterViewInit(): void {
    /*
      Cover the cached-image case: Angular's (load) binding does
      not fire if the browser serves the image from cache before
      the event listener is attached. Check img.complete after
      the view initialises.
    */
    const img = this.fullImgEl.nativeElement;
    if (img.complete && img.naturalWidth > 0) {
      this._isLoaded.set(true);
    }
  }

  onLoad(): void {
    this._isLoaded.set(true);
  }

  /*
    On error still mark loaded so the blurry placeholder
    does not permanently obscure the broken-image indicator.
  */
  onError(): void {
    this._isLoaded.set(true);
  }
}
```

### Gallery Component Using the Image Component

```typescript
// image-gallery.component.ts
import { Component } from '@angular/core';
import { NgFor } from '@angular/common';
import { LazyImageComponent } from './lazy-image.component';
import type { LazyImageConfig } from './lazy-image.model';

@Component({
  selector: 'app-image-gallery',
  standalone: true,
  imports: [NgFor, LazyImageComponent],
  template: `
    <ul class="gallery" role="list">
      <li *ngFor="let item of images; trackBy: trackById">
        <app-lazy-image [config]="item" />
      </li>
    </ul>
  `,
})
export class ImageGalleryComponent {
  images: (LazyImageConfig & { id: string })[] = [
    {
      id: 'mountain',
      src: '/images/mountain.jpg',
      srcSet: '/images/mountain-480.jpg 480w, /images/mountain-960.jpg 960w',
      sizes: '(max-width: 600px) 100vw, 50vw',
      lqip: 'data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAgGBg...',
      alt: 'Snow-capped mountain at golden hour',
      width: 960,
      height: 640,
    },
  ];

  trackById(_: number, item: { id: string }) {
    return item.id;
  }
}
```

### LQIP Generation (Node.js Build Step)

```typescript
// scripts/generate-lqip.ts
// Run at build time with: ts-node scripts/generate-lqip.ts
import sharp from 'sharp';
import path from 'path';
import fs from 'fs/promises';

async function generateLqip(inputPath: string): Promise<string> {
  const buffer = await sharp(inputPath)
    .resize(20)           // 20px wide thumbnail
    .blur(2)              // pre-blur so upscaling looks smooth
    .jpeg({ quality: 20 })
    .toBuffer();

  return `data:image/jpeg;base64,${buffer.toString('base64')}`;
}

// Usage: called from your build pipeline per image
const lqip = await generateLqip(path.resolve('public/images/mountain.jpg'));
// Embed lqip string into your JSON data file or component metadata
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| LQIP placeholder does not pollute the accessibility tree | `alt=""` and `aria-hidden="true"` on the placeholder `<img>` |
| Real image always has a meaningful `alt` text | Required prop in both React and Angular components |
| Space is reserved before the image loads | `aspect-ratio` on the wrapper prevents layout shift (CLS = 0) |
| Explicit `width` and `height` on `<img>` | Allows the browser to compute aspect ratio even before CSS loads |
| Image is not downloaded until needed | `loading="lazy"` defers the request until the image nears the viewport |
| Decoding does not block the main thread | `decoding="async"` lets the browser decode off the main thread |
| Cross-fade does not distract users with vestibular disorders | `prefers-reduced-motion: reduce` disables all transitions |
| Error state does not leave a permanent blur overlay | `onError` / `(error)` also marks `isLoaded`, revealing the native broken-image state |
| Colour contrast of placeholder background | Neutral `#e9e9e9` fallback works against both light and dark content |

---

## Production Pitfalls

**1. `loading="lazy"` does not fire `load` for cached images**
If the image is already in the browser cache, the load event fires synchronously during HTML parsing — before your JavaScript event handler is attached. Your component will remain in the blur state forever. Fix: after mount, check `img.complete && img.naturalWidth > 0` and mark loaded immediately. Both the React and Angular implementations above include this guard.

**2. LQIP causes layout shift if the aspect-ratio box is missing**
If you skip the wrapper and set dimensions only with `width`/`height` attributes, older browsers that do not support the implicit aspect ratio from those attributes will reflow when the image loads. Fix: always use an explicit `aspect-ratio` on the wrapper. Provide `width` and `height` as a redundant fallback for the browser's built-in layout calculation.

**3. Blur edges bleed over adjacent content**
`filter: blur(20px)` on a full-size element creates a visible halo that spills outside the element's boundary. Fix: `overflow: hidden` on the wrapper clips the halo. Scale the placeholder up by ~5% (`transform: scale(1.05)`) so the blurred edges are pushed outside the clipping boundary.

**4. Large base64 LQIP strings inflate your HTML payload**
A 20px JPEG at quality 20 produces roughly 300–600 bytes of base64. Across 100 images on a single-page render that is 30–60 KB of extra HTML. Fix: cap thumbnails at 10–20px wide and quality 10–20. Alternatively use Thumbhash or Blurhash, which encode a perceptual blur in ~30 bytes and are decoded client-side by a tiny JS library.

**5. `decoding="async"` is not a silver bullet**
`decoding="async"` hints to the browser that it may decode the image off the main thread, but it does not guarantee it, and it does not defer the network request — `loading="lazy"` does that. Confusing the two leads to incorrect expectations. Always use both attributes together; they address different bottlenecks.

**6. LCP images must not be lazy-loaded**
The browser's Largest Contentful Paint (LCP) element — typically a hero image — must load as fast as possible. Applying `loading="lazy"` to it delays the LCP score and hurts Core Web Vitals. Fix: never use `loading="lazy"` on images that are in the initial viewport. Use `loading="eager"` (the default) and `fetchpriority="high"` on hero images. Only apply the blur-up pattern with lazy loading to below-the-fold images.

**7. `srcset` and `sizes` are ignored if the browser skips lazy loading**
Some browser extensions and privacy tools disable native lazy loading. In those cases the browser loads the image immediately but may still pick the wrong `srcset` candidate if `sizes` is incorrect. Fix: always provide accurate `sizes` values that reflect your actual CSS layout. Measure with Chrome DevTools — if the browser selects a larger candidate than the rendered size, your `sizes` attribute is wrong.

---

## Interview Angle

**Q: "Walk me through how you would implement a blur-up image loading pattern and why each piece is necessary."**

A strong answer addresses each layer separately. The LQIP (a 10–20px inline base64 thumbnail) is generated at build time using Sharp and embedded in the HTML so there is zero extra network request — it appears in the same response payload as the page itself. The `aspect-ratio` box on the wrapper reserves exactly the right space before any image loads, which eliminates layout shift and keeps CLS at zero. The real image uses `loading="lazy"` to defer its network request until the image is within 200–1000px of the viewport (browser-determined), and `decoding="async"` to prevent its decode from blocking the main thread. The JavaScript role is purely reactive: an `onload` event listener adds a class to the wrapper, which triggers a CSS cross-fade between the blurred placeholder (opacity 1 → 0) and the full image (opacity 0 → 1). The blur edge artifact is handled by scaling the placeholder up 5% and clipping it with `overflow: hidden` on the wrapper.

**Follow-up: "What would you do differently for the hero image versus below-the-fold images?"**

The hero image is the LCP candidate, so it must never use `loading="lazy"` — that would defer the most important paint on the page. For the hero, use `loading="eager"` (the default) and add `fetchpriority="high"` to signal to the browser's preload scanner that this is a high-priority resource. The blur-up placeholder is still valid for the hero because the LQIP renders instantly from the inline base64 and the transition will fire as soon as the full image loads, which — for the LCP image — should be fast. For all images below the fold, apply the full lazy pattern described above.
