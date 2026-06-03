# System Design: Image Gallery

## Overview

Design a responsive image gallery with lazy loading, virtualization, lightbox modal, infinite scroll, and performance optimization. This covers architecture for handling thousands of images efficiently with great user experience.

## Requirements

### Functional Requirements
- Grid layout (responsive columns)
- Lazy loading images
- Lightbox/modal view
- Zoom and pan
- Keyboard navigation
- Share functionality
- Download images
- Image metadata display

### Non-Functional Requirements
- Load < 100 images initially
- Lazy load on scroll
- Support 10,000+ images
- 60fps smooth scrolling
- Responsive images (srcset)
- Accessible (WCAG 2.1)
- Works offline (cached images)
- < 200KB initial bundle

## Architecture

```typescript
// Image gallery component with lazy loading
@Component({
  selector: 'app-image-gallery',
  standalone: true,
  template: `
    <div class="gallery-grid" #grid>
      <div
        *ngFor="let image of visibleImages; trackBy: trackById"
        class="gallery-item"
        [style.aspect-ratio]="image.aspectRatio"
        (click)="openLightbox(image)">
        
        <img
          *ngIf="image.isInView"
          [src]="image.thumbnail"
          [srcset]="image.srcset"
          [sizes]="sizes"
          [alt]="image.alt"
          [loading]="'lazy'"
          class="gallery-image"
          (load)="onImageLoad(image)"
          (error)="onImageError(image)">
        
        <div *ngIf="!image.loaded" class="skeleton"></div>
        
        <div class="image-overlay">
          <span class="image-title">{{ image.title }}</span>
        </div>
      </div>
    </div>

    <!-- Lightbox -->
    <app-lightbox
      *ngIf="selectedImage"
      [image]="selectedImage"
      [images]="allImages"
      (close)="closeLightbox()"
      (next)="nextImage()"
      (previous)="previousImage()">
    </app-lightbox>
  `,
  styles: [`
    .gallery-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
      gap: 16px;
      padding: 16px;
    }

    .gallery-item {
      position: relative;
      cursor: pointer;
      overflow: hidden;
      border-radius: 8px;
      background: #f0f0f0;
    }

    .gallery-image {
      width: 100%;
      height: 100%;
      object-fit: cover;
      transition: transform 0.3s;
    }

    .gallery-item:hover .gallery-image {
      transform: scale(1.05);
    }

    .skeleton {
      width: 100%;
      height: 100%;
      background: linear-gradient(
        90deg,
        #f0f0f0 25%,
        #e0e0e0 50%,
        #f0f0f0 75%
      );
      background-size: 200% 100%;
      animation: shimmer 1.5s infinite;
    }

    @keyframes shimmer {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class ImageGalleryComponent implements OnInit, AfterViewInit {
  @ViewChild('grid') gridRef!: ElementRef;
  
  allImages: GalleryImage[] = [];
  visibleImages: GalleryImage[] = [];
  selectedImage: GalleryImage | null = null;
  
  sizes = '(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw';
  
  private observer!: IntersectionObserver;
  private page = 0;
  private pageSize = 50;

  constructor(private galleryService: GalleryService) {}

  ngOnInit() {
    this.loadImages();
  }

  ngAfterViewInit() {
    this.setupLazyLoading();
    this.setupInfiniteScroll();
  }

  private loadImages() {
    this.galleryService.getImages(this.page, this.pageSize)
      .subscribe(images => {
        this.allImages = [...this.allImages, ...images];
        this.visibleImages = [...this.visibleImages, ...images];
        this.page++;
      });
  }

  private setupLazyLoading() {
    this.observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            const img = entry.target as HTMLImageElement;
            const imageId = img.getAttribute('data-id');
            const image = this.visibleImages.find(i => i.id === imageId);
            if (image) {
              image.isInView = true;
            }
          }
        });
      },
      { rootMargin: '50px' }
    );
  }

  private setupInfiniteScroll() {
    fromEvent(window, 'scroll').pipe(
      throttleTime(200),
      map(() => window.innerHeight + window.scrollY),
      filter(scrollBottom => {
        const threshold = document.documentElement.scrollHeight - 500;
        return scrollBottom >= threshold;
      })
    ).subscribe(() => {
      this.loadImages();
    });
  }

  openLightbox(image: GalleryImage) {
    this.selectedImage = image;
  }

  closeLightbox() {
    this.selectedImage = null;
  }

  nextImage() {
    const index = this.allImages.indexOf(this.selectedImage!);
    if (index < this.allImages.length - 1) {
      this.selectedImage = this.allImages[index + 1];
    }
  }

  previousImage() {
    const index = this.allImages.indexOf(this.selectedImage!);
    if (index > 0) {
      this.selectedImage = this.allImages[index - 1];
    }
  }

  onImageLoad(image: GalleryImage) {
    image.loaded = true;
  }

  onImageError(image: GalleryImage) {
    image.error = true;
    image.src = '/assets/placeholder.jpg';
  }

  trackById(index: number, image: GalleryImage): string {
    return image.id;
  }
}

// Lightbox component
@Component({
  selector: 'app-lightbox',
  standalone: true,
  template: `
    <div class="lightbox-overlay" 
         (click)="close.emit()"
         [@fadeIn]>
      
      <div class="lightbox-container" (click)="$event.stopPropagation()">
        
        <!-- Close button -->
        <button 
          class="lightbox-close"
          (click)="close.emit()"
          aria-label="Close">
          ×
        </button>

        <!-- Navigation -->
        <button
          class="nav-button prev"
          (click)="previous.emit()"
          [disabled]="isFirst"
          aria-label="Previous image">
          ‹
        </button>

        <button
          class="nav-button next"
          (click)="next.emit()"
          [disabled]="isLast"
          aria-label="Next image">
          ›
        </button>

        <!-- Image -->
        <div class="image-container" #container>
          <img
            [src]="image.fullSize"
            [alt]="image.alt"
            class="lightbox-image"
            [style.transform]="getTransform()"
            (wheel)="onWheel($event)"
            (mousedown)="onMouseDown($event)"
            (mousemove)="onMouseMove($event)"
            (mouseup)="onMouseUp()">
        </div>

        <!-- Controls -->
        <div class="lightbox-controls">
          <button (click)="zoomIn()" aria-label="Zoom in">+</button>
          <button (click)="zoomOut()" aria-label="Zoom out">−</button>
          <button (click)="resetZoom()" aria-label="Reset zoom">⟲</button>
          <button (click)="download()" aria-label="Download">⬇</button>
          <button (click)="share()" aria-label="Share">⤴</button>
        </div>

        <!-- Metadata -->
        <div class="image-info">
          <h3>{{ image.title }}</h3>
          <p>{{ image.description }}</p>
          <span>{{ image.width }} × {{ image.height }}</span>
        </div>
      </div>
    </div>
  `,
  animations: [
    trigger('fadeIn', [
      transition(':enter', [
        style({ opacity: 0 }),
        animate('200ms', style({ opacity: 1 }))
      ]),
      transition(':leave', [
        animate('200ms', style({ opacity: 0 }))
      ])
    ])
  ]
})
export class LightboxComponent implements OnInit, OnDestroy {
  @Input() image!: GalleryImage;
  @Input() images: GalleryImage[] = [];
  @Output() close = new EventEmitter<void>();
  @Output() next = new EventEmitter<void>();
  @Output() previous = new EventEmitter<void>();

  @ViewChild('container') container!: ElementRef;

  scale = 1;
  translateX = 0;
  translateY = 0;
  
  private isDragging = false;
  private startX = 0;
  private startY = 0;

  get isFirst(): boolean {
    return this.images.indexOf(this.image) === 0;
  }

  get isLast(): boolean {
    return this.images.indexOf(this.image) === this.images.length - 1;
  }

  ngOnInit() {
    // Keyboard navigation
    fromEvent<KeyboardEvent>(document, 'keydown')
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        switch (event.key) {
          case 'Escape':
            this.close.emit();
            break;
          case 'ArrowLeft':
            if (!this.isFirst) this.previous.emit();
            break;
          case 'ArrowRight':
            if (!this.isLast) this.next.emit();
            break;
        }
      });
  }

  getTransform(): string {
    return `scale(${this.scale}) translate(${this.translateX}px, ${this.translateY}px)`;
  }

  onWheel(event: WheelEvent) {
    event.preventDefault();
    const delta = event.deltaY > 0 ? -0.1 : 0.1;
    this.scale = Math.max(0.5, Math.min(5, this.scale + delta));
  }

  zoomIn() {
    this.scale = Math.min(5, this.scale + 0.25);
  }

  zoomOut() {
    this.scale = Math.max(0.5, this.scale - 0.25);
  }

  resetZoom() {
    this.scale = 1;
    this.translateX = 0;
    this.translateY = 0;
  }

  onMouseDown(event: MouseEvent) {
    if (this.scale <= 1) return;
    this.isDragging = true;
    this.startX = event.clientX - this.translateX;
    this.startY = event.clientY - this.translateY;
  }

  onMouseMove(event: MouseEvent) {
    if (!this.isDragging) return;
    this.translateX = event.clientX - this.startX;
    this.translateY = event.clientY - this.startY;
  }

  onMouseUp() {
    this.isDragging = false;
  }

  download() {
    const link = document.createElement('a');
    link.href = this.image.fullSize;
    link.download = this.image.title;
    link.click();
  }

  async share() {
    if (navigator.share) {
      try {
        await navigator.share({
          title: this.image.title,
          text: this.image.description,
          url: this.image.shareUrl
        });
      } catch (err) {
        console.error('Share failed:', err);
      }
    }
  }

  private destroy$ = new Subject<void>();

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Image service with caching
@Injectable({ providedIn: 'root' })
export class GalleryService {
  private cache = new Map<string, GalleryImage[]>();

  constructor(private http: HttpClient) {}

  getImages(page: number, pageSize: number): Observable<GalleryImage[]> {
    const cacheKey = `${page}-${pageSize}`;
    
    if (this.cache.has(cacheKey)) {
      return of(this.cache.get(cacheKey)!);
    }

    return this.http.get<GalleryImage[]>('/api/images', {
      params: { page: page.toString(), limit: pageSize.toString() }
    }).pipe(
      map(images => images.map(img => this.processImage(img))),
      tap(images => this.cache.set(cacheKey, images))
    );
  }

  private processImage(image: any): GalleryImage {
    return {
      ...image,
      thumbnail: this.getThumbnail(image.url),
      srcset: this.generateSrcset(image.url),
      aspectRatio: image.width / image.height,
      isInView: false,
      loaded: false,
      error: false
    };
  }

  private getThumbnail(url: string): string {
    return `${url}?w=400&h=400&fit=crop`;
  }

  private generateSrcset(url: string): string {
    return [
      `${url}?w=400 400w`,
      `${url}?w=800 800w`,
      `${url}?w=1200 1200w`,
      `${url}?w=1600 1600w`
    ].join(', ');
  }
}

// Virtual scroll for large galleries
@Component({
  selector: 'app-virtual-gallery',
  template: `
    <cdk-virtual-scroll-viewport
      itemSize="250"
      class="gallery-viewport">
      
      <div class="gallery-grid">
        <div
          *cdkVirtualFor="let image of images; trackBy: trackById"
          class="gallery-item"
          (click)="openImage(image)">
          
          <img [src]="image.thumbnail" [alt]="image.alt">
        </div>
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class VirtualGalleryComponent {
  images: GalleryImage[] = [];

  trackById(index: number, image: GalleryImage): string {
    return image.id;
  }

  openImage(image: GalleryImage) {
    // Open lightbox
  }
}
```

## Performance Optimization

### Responsive Images

```typescript
// Generate srcset for responsive images
function generateResponsiveImage(url: string): ResponsiveImage {
  return {
    src: `${url}?w=800`,
    srcset: `
      ${url}?w=400 400w,
      ${url}?w=800 800w,
      ${url}?w=1200 1200w,
      ${url}?w=1600 1600w,
      ${url}?w=2000 2000w
    `,
    sizes: `
      (max-width: 400px) 100vw,
      (max-width: 800px) 50vw,
      33vw
    `
  };
}

// WebP with fallback
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg" type="image/jpeg">
  <img src="image.jpg" alt="Description">
</picture>
```

### Image Optimization

```typescript
// Progressive JPEG loading
// Service Worker for image caching
self.addEventListener('fetch', (event) => {
  if (event.request.destination === 'image') {
    event.respondWith(
      caches.match(event.request).then((response) => {
        return response || fetch(event.request).then((fetchResponse) => {
          return caches.open('images').then((cache) => {
            cache.put(event.request, fetchResponse.clone());
            return fetchResponse;
          });
        });
      })
    );
  }
});

// Preload critical images
<link rel="preload" as="image" href="hero.jpg">

// Lazy load with native browser API
<img src="image.jpg" loading="lazy" alt="Description">
```

## Key Takeaways

1. **Lazy Loading**: Use Intersection Observer API for efficient lazy loading, loading images only when they enter viewport.

2. **Responsive Images**: Implement srcset and sizes attributes to serve appropriate image sizes for different devices and screen sizes.

3. **Virtual Scrolling**: Use virtual scrolling for galleries with thousands of images to maintain smooth performance.

4. **Image Caching**: Implement multi-level caching (memory, IndexedDB, Service Worker) for offline support and faster loads.

5. **Lightbox UX**: Provide zoom, pan, keyboard navigation, and touch gestures for comprehensive image viewing experience.

6. **Progressive Loading**: Show skeleton loaders while images load, use progressive JPEGs for better perceived performance.

7. **Grid Layout**: Use CSS Grid with responsive columns (auto-fill, minmax) for flexible layouts across devices.

8. **Keyboard Navigation**: Support arrow keys, escape, and other shortcuts for accessibility and power users.

9. **Error Handling**: Show placeholder images on load errors, implement retry logic for failed loads.

10. **Performance Metrics**: Monitor LCP (Largest Contentful Paint), track image load times, and optimize critical rendering path.

## Red Flags to Avoid

- Loading all images at once without lazy loading
- Not using responsive images (srcset/sizes)
- Missing keyboard navigation in lightbox
- No error handling for failed image loads
- Not implementing proper accessibility (alt text, ARIA)
- Loading full-size images for thumbnails
- No loading states or skeletons
- Memory leaks from image preloading
- Not caching images for offline support
- Poor mobile experience (no touch gestures)

## Trade-offs

**Masonry vs Grid Layout:**
- Masonry: More visual interest, complex layout calculations
- Grid: Simpler, better performance, predictable

**Client-side vs Server-side Optimization:**
- Client: More control, slower initial load
- Server: Faster delivery, less flexibility

**Native lazy loading vs Intersection Observer:**
- Native: Simpler, less control over threshold
- Intersection Observer: More control, broader feature support

## Interview Talking Points

- Explain lazy loading implementation with Intersection Observer
- Discuss responsive image strategies (srcset, sizes, picture element)
- Describe lightbox architecture and state management
- Explain virtual scrolling benefits for large galleries
- Discuss image optimization techniques (WebP, compression)
- Describe caching strategy and offline support
- Explain performance metrics and optimization goals
- Discuss accessibility requirements for image galleries
- Describe touch gesture handling for mobile
- Explain preloading strategies for critical images
