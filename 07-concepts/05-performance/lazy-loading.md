# Lazy Loading

## Overview

Lazy loading is a design pattern that defers the loading of non-critical resources until they are actually needed. This technique significantly improves initial page load performance by reducing the amount of code, images, and other assets that need to be downloaded upfront. Lazy loading is essential for modern web applications, especially those with heavy content or targeting mobile users.

## Types of Lazy Loading

```
Lazy Loading Categories
========================

┌────────────────────────────────────────────────┐
│           Lazy Loading Strategies              │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │        Image Lazy Loading                │ │
│  │  • Native loading="lazy"                 │ │
│  │  • Intersection Observer                 │ │
│  │  • Blur-up/LQIP technique                │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │      Component Lazy Loading              │ │
│  │  • React.lazy() + Suspense               │ │
│  │  • Dynamic imports                       │ │
│  │  • Conditional rendering                 │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │         Script Lazy Loading              │ │
│  │  • async/defer attributes                │ │
│  │  • Dynamic script injection              │ │
│  │  • Module imports                        │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │         Content Lazy Loading             │ │
│  │  • Infinite scroll                       │ │
│  │  • Pagination                            │ │
│  │  • Load more button                      │ │
│  └──────────────────────────────────────────┘ │
│                                                │
└────────────────────────────────────────────────┘
```

## Native Image Lazy Loading

Modern browsers support native lazy loading with the `loading` attribute.

```typescript
// React: Native lazy loading
function ImageGallery({ images }: { images: string[] }) {
  return (
    <div className="gallery">
      {images.map((src, index) => (
        <img
          key={index}
          src={src}
          alt={`Gallery image ${index + 1}`}
          loading="lazy"
          width={800}
          height={600}
          style={{ maxWidth: '100%', height: 'auto' }}
        />
      ))}
    </div>
  );
}

// Next.js Image component with built-in lazy loading
import Image from 'next/image';

function OptimizedGallery({ images }: { images: string[] }) {
  return (
    <div className="gallery">
      {images.map((src, index) => (
        <Image
          key={index}
          src={src}
          alt={`Gallery image ${index + 1}`}
          width={800}
          height={600}
          loading="lazy"
          placeholder="blur"
          blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
        />
      ))}
    </div>
  );
}

// Eager loading for above-the-fold images
function HeroSection() {
  return (
    <div className="hero">
      <img
        src="/hero-image.jpg"
        alt="Hero"
        loading="eager" // Or omit loading attribute
        fetchPriority="high"
        width={1920}
        height={1080}
      />
    </div>
  );
}

// Fallback for browsers without native lazy loading
function LazyImage({ src, alt, ...props }: any) {
  const [imageSrc, setImageSrc] = useState<string | null>(null);
  const imgRef = useRef<HTMLImageElement>(null);

  useEffect(() => {
    // Check if browser supports native lazy loading
    if ('loading' in HTMLImageElement.prototype) {
      setImageSrc(src);
      return;
    }

    // Fallback to Intersection Observer
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            setImageSrc(src);
            observer.disconnect();
          }
        });
      },
      { rootMargin: '50px' }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, [src]);

  return (
    <img
      ref={imgRef}
      src={imageSrc || 'data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7'}
      alt={alt}
      loading="lazy"
      {...props}
    />
  );
}
```

**Angular Implementation:**

```typescript
// lazy-image.directive.ts
import {
  Directive,
  ElementRef,
  Input,
  OnInit,
  OnDestroy
} from '@angular/core';

@Directive({
  selector: 'img[appLazyLoad]'
})
export class LazyLoadDirective implements OnInit, OnDestroy {
  @Input() appLazyLoad!: string;
  @Input() defaultImage = 'data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7';
  
  private observer?: IntersectionObserver;

  constructor(private el: ElementRef<HTMLImageElement>) {}

  ngOnInit(): void {
    // Check for native lazy loading support
    if ('loading' in HTMLImageElement.prototype) {
      this.loadImage();
      return;
    }

    // Fallback to Intersection Observer
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            this.loadImage();
            this.observer?.disconnect();
          }
        });
      },
      {
        rootMargin: '50px'
      }
    );

    this.observer.observe(this.el.nativeElement);
    this.el.nativeElement.src = this.defaultImage;
  }

  ngOnDestroy(): void {
    this.observer?.disconnect();
  }

  private loadImage(): void {
    const img = this.el.nativeElement;
    img.src = this.appLazyLoad;
    img.classList.add('loaded');
  }
}

// Usage in component
@Component({
  selector: 'app-gallery',
  template: `
    <div class="gallery">
      <img
        *ngFor="let image of images"
        [appLazyLoad]="image.src"
        [alt]="image.alt"
        [width]="image.width"
        [height]="image.height"
        class="gallery-image"
      />
    </div>
  `,
  styles: [`
    .gallery-image {
      max-width: 100%;
      height: auto;
      opacity: 0;
      transition: opacity 0.3s;
    }
    .gallery-image.loaded {
      opacity: 1;
    }
  `]
})
export class GalleryComponent {
  images = [
    { src: '/image1.jpg', alt: 'Image 1', width: 800, height: 600 },
    { src: '/image2.jpg', alt: 'Image 2', width: 800, height: 600 },
    // ...
  ];
}
```

## Intersection Observer API

The Intersection Observer API provides a powerful way to lazy load elements when they enter the viewport.

```typescript
// React: Custom hook for Intersection Observer
import { useEffect, useRef, useState } from 'react';

interface UseIntersectionObserverProps {
  threshold?: number;
  root?: Element | null;
  rootMargin?: string;
  freezeOnceVisible?: boolean;
}

function useIntersectionObserver({
  threshold = 0,
  root = null,
  rootMargin = '0px',
  freezeOnceVisible = false
}: UseIntersectionObserverProps = {}): [
  React.RefObject<any>,
  boolean
] {
  const [isIntersecting, setIsIntersecting] = useState(false);
  const elementRef = useRef<any>(null);

  useEffect(() => {
    const element = elementRef.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        const isElementIntersecting = entry.isIntersecting;
        setIsIntersecting(isElementIntersecting);

        if (isElementIntersecting && freezeOnceVisible) {
          observer.disconnect();
        }
      },
      { threshold, root, rootMargin }
    );

    observer.observe(element);

    return () => observer.disconnect();
  }, [threshold, root, rootMargin, freezeOnceVisible]);

  return [elementRef, isIntersecting];
}

// Usage: Lazy load images
function LazyLoadImage({ src, alt }: { src: string; alt: string }) {
  const [ref, isVisible] = useIntersectionObserver({
    threshold: 0.1,
    rootMargin: '100px',
    freezeOnceVisible: true
  });

  return (
    <div ref={ref} className="image-container">
      {isVisible ? (
        <img src={src} alt={alt} className="fade-in" />
      ) : (
        <div className="image-placeholder" />
      )}
    </div>
  );
}

// Usage: Lazy load components
function LazyLoadComponent() {
  const [ref, isVisible] = useIntersectionObserver({
    threshold: 0.5,
    freezeOnceVisible: true
  });

  return (
    <div ref={ref}>
      {isVisible ? (
        <HeavyComponent />
      ) : (
        <div style={{ height: 400 }}>Loading...</div>
      )}
    </div>
  );
}

// Advanced: Progressive image loading with blur-up
function ProgressiveImage({
  lowQualitySrc,
  highQualitySrc,
  alt
}: {
  lowQualitySrc: string;
  highQualitySrc: string;
  alt: string;
}) {
  const [currentSrc, setCurrentSrc] = useState(lowQualitySrc);
  const [isLoading, setIsLoading] = useState(true);
  const [ref, isVisible] = useIntersectionObserver({
    threshold: 0.1,
    freezeOnceVisible: true
  });

  useEffect(() => {
    if (!isVisible) return;

    const img = new Image();
    img.src = highQualitySrc;
    img.onload = () => {
      setCurrentSrc(highQualitySrc);
      setIsLoading(false);
    };
  }, [isVisible, highQualitySrc]);

  return (
    <div ref={ref} className="progressive-image">
      <img
        src={currentSrc}
        alt={alt}
        className={isLoading ? 'loading' : 'loaded'}
        style={{
          filter: isLoading ? 'blur(10px)' : 'none',
          transition: 'filter 0.3s'
        }}
      />
    </div>
  );
}

// Lazy load background images
function LazyBackgroundImage({
  lowQualitySrc,
  highQualitySrc,
  children
}: {
  lowQualitySrc: string;
  highQualitySrc: string;
  children: React.ReactNode;
}) {
  const [backgroundImage, setBackgroundImage] = useState(
    `url(${lowQualitySrc})`
  );
  const [ref, isVisible] = useIntersectionObserver({
    threshold: 0.1,
    freezeOnceVisible: true
  });

  useEffect(() => {
    if (!isVisible) return;

    const img = new Image();
    img.src = highQualitySrc;
    img.onload = () => {
      setBackgroundImage(`url(${highQualitySrc})`);
    };
  }, [isVisible, highQualitySrc]);

  return (
    <div
      ref={ref}
      style={{
        backgroundImage,
        backgroundSize: 'cover',
        backgroundPosition: 'center',
        transition: 'background-image 0.3s'
      }}
    >
      {children}
    </div>
  );
}
```

**Angular Implementation:**

```typescript
// intersection-observer.service.ts
import { Injectable, ElementRef } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class IntersectionObserverService {
  createObserver(
    element: ElementRef,
    options: IntersectionObserverInit = {}
  ): Observable<boolean> {
    return new Observable(subscriber => {
      const observer = new IntersectionObserver(
        (entries) => {
          entries.forEach(entry => {
            subscriber.next(entry.isIntersecting);
          });
        },
        {
          threshold: options.threshold || 0.1,
          rootMargin: options.rootMargin || '50px'
        }
      );

      observer.observe(element.nativeElement);

      return () => observer.disconnect();
    });
  }
}

// lazy-load-image.component.ts
import { Component, Input, ElementRef, OnInit } from '@angular/core';
import { IntersectionObserverService } from './intersection-observer.service';

@Component({
  selector: 'app-lazy-load-image',
  template: `
    <div class="image-container">
      <img
        *ngIf="isVisible"
        [src]="src"
        [alt]="alt"
        class="fade-in"
      />
      <div *ngIf="!isVisible" class="image-placeholder"></div>
    </div>
  `,
  styles: [`
    .image-container {
      position: relative;
    }
    .image-placeholder {
      background: #f0f0f0;
      width: 100%;
      padding-top: 75%; /* 4:3 aspect ratio */
    }
    .fade-in {
      animation: fadeIn 0.3s;
    }
    @keyframes fadeIn {
      from { opacity: 0; }
      to { opacity: 1; }
    }
  `]
})
export class LazyLoadImageComponent implements OnInit {
  @Input() src!: string;
  @Input() alt!: string;

  isVisible = false;

  constructor(
    private el: ElementRef,
    private intersectionObserver: IntersectionObserverService
  ) {}

  ngOnInit(): void {
    this.intersectionObserver
      .createObserver(this.el, { threshold: 0.1, rootMargin: '100px' })
      .subscribe(isVisible => {
        if (isVisible && !this.isVisible) {
          this.isVisible = true;
        }
      });
  }
}
```

## Infinite Scroll Implementation

```typescript
// React: Infinite scroll with lazy loading
import { useState, useEffect, useRef, useCallback } from 'react';

interface InfiniteScrollProps {
  loadMore: () => Promise<any[]>;
  hasMore: boolean;
}

function useInfiniteScroll({ loadMore, hasMore }: InfiniteScrollProps) {
  const [loading, setLoading] = useState(false);
  const [items, setItems] = useState<any[]>([]);
  const observerRef = useRef<IntersectionObserver | null>(null);
  const loadMoreRef = useRef<HTMLDivElement>(null);

  const handleObserver = useCallback(
    (entries: IntersectionObserverEntry[]) => {
      const target = entries[0];
      if (target.isIntersecting && hasMore && !loading) {
        setLoading(true);
        loadMore().then(newItems => {
          setItems(prev => [...prev, ...newItems]);
          setLoading(false);
        });
      }
    },
    [hasMore, loading, loadMore]
  );

  useEffect(() => {
    const option = {
      root: null,
      rootMargin: '100px',
      threshold: 0
    };

    observerRef.current = new IntersectionObserver(handleObserver, option);

    if (loadMoreRef.current) {
      observerRef.current.observe(loadMoreRef.current);
    }

    return () => {
      if (observerRef.current) {
        observerRef.current.disconnect();
      }
    };
  }, [handleObserver]);

  return { items, loading, loadMoreRef };
}

// Usage
function InfiniteScrollList() {
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = async () => {
    const response = await fetch(`/api/items?page=${page}`);
    const data = await response.json();
    
    setPage(prev => prev + 1);
    setHasMore(data.hasMore);
    
    return data.items;
  };

  const { items, loading, loadMoreRef } = useInfiniteScroll({
    loadMore,
    hasMore
  });

  return (
    <div className="infinite-scroll-container">
      <div className="items-list">
        {items.map((item, index) => (
          <ItemCard key={index} item={item} />
        ))}
      </div>

      <div ref={loadMoreRef} className="load-more-trigger">
        {loading && <LoadingSpinner />}
        {!hasMore && <div>No more items</div>}
      </div>
    </div>
  );
}

// Advanced: Virtualized infinite scroll
import { useVirtual } from 'react-virtual';

function VirtualizedInfiniteScroll() {
  const [items, setItems] = useState<any[]>([]);
  const [hasMore, setHasMore] = useState(true);
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtual({
    size: items.length,
    parentRef,
    estimateSize: useCallback(() => 100, []),
    overscan: 5
  });

  useEffect(() => {
    const lastItem = rowVirtualizer.virtualItems[
      rowVirtualizer.virtualItems.length - 1
    ];

    if (!lastItem) return;

    if (
      lastItem.index >= items.length - 1 &&
      hasMore &&
      !isLoading
    ) {
      loadMoreItems();
    }
  }, [rowVirtualizer.virtualItems, items.length, hasMore]);

  return (
    <div
      ref={parentRef}
      className="virtualized-list"
      style={{ height: '600px', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${rowVirtualizer.totalSize}px`,
          width: '100%',
          position: 'relative'
        }}
      >
        {rowVirtualizer.virtualItems.map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`
            }}
          >
            <ItemCard item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Angular Implementation:**

```typescript
// infinite-scroll.directive.ts
import {
  Directive,
  EventEmitter,
  HostListener,
  Input,
  Output
} from '@angular/core';

@Directive({
  selector: '[appInfiniteScroll]'
})
export class InfiniteScrollDirective {
  @Output() scrolled = new EventEmitter<void>();
  @Input() scrollThreshold = 200;

  @HostListener('scroll', ['$event'])
  onScroll(event: Event): void {
    const element = event.target as HTMLElement;
    const scrollPosition = element.scrollHeight - element.scrollTop;
    const clientHeight = element.clientHeight;

    if (scrollPosition <= clientHeight + this.scrollThreshold) {
      this.scrolled.emit();
    }
  }
}

// infinite-scroll-list.component.ts
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-infinite-scroll-list',
  template: `
    <div
      class="scroll-container"
      appInfiniteScroll
      (scrolled)="onScroll()"
      [scrollThreshold]="100"
    >
      <div class="items-list">
        <app-item-card
          *ngFor="let item of items; trackBy: trackByFn"
          [item]="item"
        ></app-item-card>
      </div>

      <div class="loading-indicator" *ngIf="loading">
        <app-spinner></app-spinner>
      </div>

      <div class="end-message" *ngIf="!hasMore">
        No more items
      </div>
    </div>
  `,
  styles: [`
    .scroll-container {
      height: 600px;
      overflow-y: auto;
    }
  `]
})
export class InfiniteScrollListComponent implements OnInit {
  items: any[] = [];
  loading = false;
  hasMore = true;
  page = 1;

  ngOnInit(): void {
    this.loadItems();
  }

  onScroll(): void {
    if (!this.loading && this.hasMore) {
      this.loadItems();
    }
  }

  async loadItems(): Promise<void> {
    this.loading = true;

    try {
      const response = await fetch(`/api/items?page=${this.page}`);
      const data = await response.json();

      this.items = [...this.items, ...data.items];
      this.hasMore = data.hasMore;
      this.page++;
    } catch (error) {
      console.error('Failed to load items:', error);
    } finally {
      this.loading = false;
    }
  }

  trackByFn(index: number, item: any): any {
    return item.id;
  }
}
```

## Lazy Loading Iframes and Embeds

```typescript
// React: Lazy load YouTube videos
function YouTubeEmbed({ videoId }: { videoId: string }) {
  const [shouldLoad, setShouldLoad] = useState(false);
  const [ref, isVisible] = useIntersectionObserver({
    threshold: 0.5,
    freezeOnceVisible: true
  });

  useEffect(() => {
    if (isVisible) {
      setShouldLoad(true);
    }
  }, [isVisible]);

  return (
    <div ref={ref} className="video-container">
      {shouldLoad ? (
        <iframe
          width="560"
          height="315"
          src={`https://www.youtube.com/embed/${videoId}`}
          title="YouTube video"
          frameBorder="0"
          allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
          allowFullScreen
          loading="lazy"
        />
      ) : (
        <div
          className="video-placeholder"
          onClick={() => setShouldLoad(true)}
          style={{
            backgroundImage: `url(https://img.youtube.com/vi/${videoId}/maxresdefault.jpg)`,
            cursor: 'pointer'
          }}
        >
          <div className="play-button">▶</div>
        </div>
      )}
    </div>
  );
}

// Lazy load third-party widgets
function LazyWidget({ src }: { src: string }) {
  const [loaded, setLoaded] = useState(false);
  const [ref, isVisible] = useIntersectionObserver({
    threshold: 0.1,
    freezeOnceVisible: true
  });

  useEffect(() => {
    if (isVisible && !loaded) {
      const script = document.createElement('script');
      script.src = src;
      script.async = true;
      document.body.appendChild(script);
      setLoaded(true);

      return () => {
        document.body.removeChild(script);
      };
    }
  }, [isVisible, loaded, src]);

  return (
    <div ref={ref}>
      {loaded ? (
        <div id="widget-container"></div>
      ) : (
        <div className="widget-placeholder">Loading widget...</div>
      )}
    </div>
  );
}

// Native iframe lazy loading
function LazyIframe({ src, title }: { src: string; title: string }) {
  return (
    <iframe
      src={src}
      title={title}
      loading="lazy"
      width="600"
      height="400"
      frameBorder="0"
    />
  );
}
```

## Lazy Loading Flow

```
Lazy Loading Decision Process
==============================

          User scrolls page
                 │
                 ▼
     ┌───────────────────────┐
     │ Element enters        │
     │ viewport threshold?   │
     └──────┬────────────────┘
            │
    ┌───────┴────────┐
    │                │
   No               Yes
    │                │
    │                ▼
    │    ┌──────────────────────┐
    │    │ Already loaded?      │
    │    └──────┬───────────────┘
    │           │
    │    ┌──────┴────────┐
    │    │               │
    │   Yes             No
    │    │               │
    │    │               ▼
    │    │    ┌─────────────────────┐
    │    │    │ Start loading       │
    │    │    │ resource            │
    │    │    └──────┬──────────────┘
    │    │           │
    │    │           ▼
    │    │    ┌─────────────────────┐
    │    │    │ Show loading state  │
    │    │    └──────┬──────────────┘
    │    │           │
    │    │           ▼
    │    │    ┌─────────────────────┐
    │    │    │ Resource loaded     │
    │    │    └──────┬──────────────┘
    │    │           │
    │    └───────────┴───────┐
    │                        │
    └────────────────────────┤
                             ▼
                  ┌──────────────────────┐
                  │ Display content      │
                  └──────────────────────┘
```

## Common Mistakes

1. **Lazy loading above-the-fold content**
   - Problem: Delays initial render
   - Solution: Eager load visible content, lazy load below-the-fold

2. **Not providing image dimensions**
   - Problem: Causes layout shifts (CLS)
   - Solution: Always specify width/height or aspect-ratio

3. **No loading placeholder**
   - Problem: Poor UX during loading
   - Solution: Show skeleton screens or blur-up previews

4. **Aggressive lazy loading threshold**
   - Problem: Users see loading states
   - Solution: Use appropriate rootMargin (100-200px)

5. **Not handling loading errors**
   - Problem: Failed resources break layout
   - Solution: Implement error boundaries and fallbacks

6. **Lazy loading everything**
   - Problem: Unnecessary complexity, possible waterfalls
   - Solution: Only lazy load heavy or below-the-fold content

7. **No fallback for old browsers**
   - Problem: Images don't load in older browsers
   - Solution: Use IntersectionObserver with polyfill

8. **Forgetting to disconnect observers**
   - Problem: Memory leaks
   - Solution: Clean up observers when component unmounts

## Best Practices

1. **Use native `loading="lazy"`**: Simplest solution for images/iframes
2. **Provide placeholders**: Show loading states or low-quality previews
3. **Set appropriate thresholds**: Load slightly before element enters viewport
4. **Specify dimensions**: Prevent layout shifts with explicit sizing
5. **Eager load critical content**: Don't lazy load above-the-fold
6. **Handle errors gracefully**: Provide fallback content
7. **Test on slow connections**: Verify loading experience
8. **Monitor performance**: Track lazy loading metrics
9. **Use blur-up technique**: Progressive loading improves perceived performance
10. **Combine with other optimizations**: Use with responsive images, CDN

## When to Use

**Use lazy loading for:**
- Images below the fold
- Heavy components not immediately visible
- Third-party widgets and embeds
- Long lists or galleries
- Modal content
- Analytics and tracking scripts

**Don't lazy load:**
- Above-the-fold hero images
- Critical UI components
- Small, lightweight resources
- Content needed for SEO
- Essential functionality

## Interview Questions

1. **What is lazy loading and why is it important?**
   - Defers loading of non-critical resources
   - Reduces initial page load time
   - Saves bandwidth
   - Improves Core Web Vitals (LCP, TBT)

2. **Explain the difference between native lazy loading and Intersection Observer.**
   - Native: Browser-level, simple `loading="lazy"` attribute, limited control
   - IO: JavaScript API, more control, can observe any element, requires polyfill for old browsers

3. **How does the Intersection Observer API work?**
   - Observes when element intersects with viewport or ancestor
   - Asynchronous, doesn't block main thread
   - Configurable threshold and rootMargin
   - Fires callback when intersection changes

4. **What are the trade-offs of lazy loading?**
   - Pros: Faster initial load, reduced bandwidth, better performance
   - Cons: Complexity, potential CLS if not implemented properly, requires JavaScript

5. **How would you implement infinite scroll with lazy loading?**
   - Use Intersection Observer on sentinel element
   - Load more items when sentinel visible
   - Update page/offset state
   - Handle loading and error states

6. **What's the blur-up technique and why use it?**
   - Load small, blurred thumbnail first
   - Replace with full image when loaded
   - Improves perceived performance
   - Prevents empty space during load

7. **How do you prevent Cumulative Layout Shift with lazy loading?**
   - Always specify image dimensions
   - Use aspect-ratio or padding-top hack
   - Provide placeholder with same dimensions
   - Reserve space before loading

8. **What's the optimal rootMargin for lazy loading?**
   - Depends on content type and scroll speed
   - Typically 100-200px for images
   - Larger for slow connections (300-500px)
   - Consider Network Information API for adaptive thresholds

## Key Takeaways

1. Lazy loading defers non-critical resources until needed, improving initial load time
2. Use native `loading="lazy"` attribute for images and iframes when possible
3. Intersection Observer API provides flexible, performant lazy loading
4. Always specify dimensions to prevent layout shifts (CLS)
5. Use appropriate thresholds (rootMargin) to load before element is visible
6. Don't lazy load above-the-fold or critical content
7. Provide meaningful placeholders or skeleton screens during loading
8. Combine blur-up technique for progressive image loading
9. Handle errors gracefully with fallback content
10. Test on slow networks to verify user experience

## Resources

- **Official Documentation**
  - [Native Lazy Loading](https://web.dev/browser-level-image-lazy-loading/)
  - [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
  - [Loading Attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attr-loading)

- **Tools**
  - [lazysizes](https://github.com/aFarkas/lazysizes) - Lazy loading library
  - [react-lazyload](https://github.com/twobin/react-lazyload)
  - [ng-lazyload-image](https://github.com/tjoskar/ng-lazyload-image) - Angular

- **Articles**
  - [Lazy Loading Images and Video](https://web.dev/lazy-loading-images/)
  - [Lazy Loading Best Practices](https://web.dev/lazy-loading-best-practices/)
  - [Intersection Observer v2](https://developers.google.com/web/updates/2019/02/intersectionobserver-v2)
