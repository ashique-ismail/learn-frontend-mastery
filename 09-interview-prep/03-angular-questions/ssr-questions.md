# Angular Server-Side Rendering (SSR) - Interview Questions

## Overview

This guide covers Angular Universal for server-side rendering, including hydration, the TransferState API, platform detection, SEO considerations, and the differences between prerendering and SSR.

## Core Concept Questions

### 1. What is Angular Universal? How does server-side rendering work in Angular?

**Expected Answer:**

Angular Universal is Angular's solution for server-side rendering (SSR), which renders Angular applications on the server and sends fully rendered HTML to the client, improving initial load time and SEO.

**Architecture Overview:**

```typescript
// Server setup with Angular Universal
// server.ts
import 'zone.js/node';
import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import express from 'express';
import { fileURLToPath } from 'node:url';
import { dirname, join, resolve } from 'node:path';
import bootstrap from './src/main.server';

export function app(): express.Express {
  const server = express();
  const serverDistFolder = dirname(fileURLToPath(import.meta.url));
  const browserDistFolder = resolve(serverDistFolder, '../browser');
  const indexHtml = join(serverDistFolder, 'index.server.html');

  const commonEngine = new CommonEngine();

  server.set('view engine', 'html');
  server.set('views', browserDistFolder);

  // Serve static files from /browser
  server.get('*.*', express.static(browserDistFolder, {
    maxAge: '1y'
  }));

  // All regular routes use the Angular engine
  server.get('*', (req, res, next) => {
    const { protocol, originalUrl, baseUrl, headers } = req;

    commonEngine
      .render({
        bootstrap,
        documentFilePath: indexHtml,
        url: `${protocol}://${headers.host}${originalUrl}`,
        publicPath: browserDistFolder,
        providers: [{ provide: APP_BASE_HREF, useValue: baseUrl }],
      })
      .then((html) => res.send(html))
      .catch((err) => next(err));
  });

  return server;
}

function run(): void {
  const port = process.env['PORT'] || 4000;
  const server = app();
  server.listen(port, () => {
    console.log(`Node Express server listening on http://localhost:${port}`);
  });
}

run();

// main.server.ts - Server bootstrap
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { config } from './app/app.config.server';

const bootstrap = () => bootstrapApplication(AppComponent, config);

export default bootstrap;

// app.config.server.ts - Server configuration
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { appConfig } from './app.config';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering()
  ]
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

**SSR Flow:**

```typescript
// 1. Client requests page
// 2. Server receives request
// 3. Angular app renders on server
// 4. Server sends fully rendered HTML
// 5. Client receives HTML (First Contentful Paint)
// 6. Angular bootstraps on client
// 7. Hydration process begins
// 8. Application becomes interactive

// Example component that works with SSR
@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    <div class="products">
      <h1>Products</h1>
      
      <!-- Rendered on server, visible immediately -->
      <div *ngFor="let product of products()">
        <h2>{{ product.name }}</h2>
        <p>{{ product.price | currency }}</p>
        <button (click)="addToCart(product)">Add to Cart</button>
      </div>
      
      <!-- Loading state only shows during client-side navigation -->
      <div *ngIf="loading()">Loading...</div>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  products = signal<Product[]>([]);
  loading = signal(true);

  constructor(
    private productService: ProductService,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  ngOnInit() {
    // This runs on both server and client
    this.productService.getProducts().subscribe(products => {
      this.products.set(products);
      this.loading.set(false);
    });
  }

  addToCart(product: Product) {
    // This only runs on client (user interaction)
    if (isPlatformBrowser(this.platformId)) {
      this.cartService.add(product);
    }
  }
}
```

**Benefits of SSR:**

1. **Improved SEO**: Search engines receive fully rendered HTML
2. **Faster First Contentful Paint**: Users see content immediately
3. **Better Social Media Sharing**: Meta tags are rendered for crawlers
4. **Performance on Slow Devices**: Less client-side JavaScript execution
5. **Progressive Enhancement**: Works even if JavaScript fails

**Challenges:**

```typescript
// Common SSR pitfalls and solutions

// PROBLEM: Window/Document not available on server
// BAD:
ngOnInit() {
  const width = window.innerWidth; // Error on server!
  document.querySelector('.element'); // Error on server!
}

// GOOD:
import { isPlatformBrowser } from '@angular/common';
import { PLATFORM_ID, Inject } from '@angular/core';

ngOnInit() {
  if (isPlatformBrowser(this.platformId)) {
    const width = window.innerWidth;
    const element = document.querySelector('.element');
  }
}

// PROBLEM: Third-party libraries that use browser APIs
// SOLUTION: Lazy load them only on browser
async loadThirdParty() {
  if (isPlatformBrowser(this.platformId)) {
    const { default: Library } = await import('browser-only-lib');
    this.library = new Library();
  }
}

// PROBLEM: Duplicate HTTP requests (server + client)
// SOLUTION: Use TransferState (covered in next question)
```

**Follow-up Questions:**
1. What's the difference between SSR and Static Site Generation (SSG)?
2. How does Angular Universal handle HTTP requests?
3. What happens during the hydration process?

**What Interviewers Look For:**
- Understanding of SSR architecture and flow
- Knowledge of benefits and trade-offs
- Awareness of browser-specific code issues
- Understanding of when SSR is beneficial

### 2. Explain the hydration process in Angular. How does it prevent content flickering?

**Expected Answer:**

Hydration is the process where Angular attaches event listeners and application state to server-rendered HTML, making it interactive without re-rendering the DOM.

**Hydration Process:**

```typescript
// Enable hydration in application config
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideClientHydration } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideClientHydration(), // Enable hydration
  ]
};

// Hydration phases:
// 1. Server renders HTML with special markers
// 2. Client receives HTML (visible immediately)
// 3. Angular bootstraps on client
// 4. Angular walks the DOM tree
// 5. Matches server-rendered DOM with client components
// 6. Attaches event listeners and state
// 7. Application becomes interactive

// Component that demonstrates hydration
@Component({
  selector: 'app-user-profile',
  standalone: true,
  template: `
    <div class="profile">
      <!-- Server-rendered, then hydrated -->
      <img [src]="user().avatar" [alt]="user().name">
      <h2>{{ user().name }}</h2>
      <p>{{ user().bio }}</p>
      
      <!-- Button becomes interactive after hydration -->
      <button (click)="followUser()">
        {{ isFollowing() ? 'Unfollow' : 'Follow' }}
      </button>
      
      <!-- Form bindings work after hydration -->
      <form (ngSubmit)="updateBio()">
        <textarea [(ngModel)]="newBio"></textarea>
        <button type="submit">Update</button>
      </form>
    </div>
  `
})
export class UserProfileComponent {
  user = signal<User>({ name: '', avatar: '', bio: '' });
  isFollowing = signal(false);
  newBio = '';

  // Method only callable after hydration
  followUser() {
    // Event listener attached during hydration
    this.isFollowing.update(following => !following);
  }

  updateBio() {
    // Form submission works after hydration
    this.userService.updateBio(this.newBio);
  }
}
```

**Preventing Flickering:**

```typescript
// WITHOUT hydration (old approach)
// 1. Server renders HTML
// 2. Client receives HTML (visible)
// 3. Angular destroys server HTML
// 4. Angular re-renders from scratch (FLICKER!)
// 5. Application interactive

// WITH hydration (new approach)
// 1. Server renders HTML
// 2. Client receives HTML (visible)
// 3. Angular reuses server HTML
// 4. Angular attaches to existing DOM (NO FLICKER!)
// 5. Application interactive

// Demonstrating the difference
@Component({
  selector: 'app-image-gallery',
  standalone: true,
  template: `
    <!-- Without hydration: images flash/reload -->
    <!-- With hydration: images stay stable -->
    <div class="gallery">
      <img *ngFor="let image of images()" 
           [src]="image.url" 
           [alt]="image.title"
           [width]="image.width"
           [height]="image.height">
    </div>
  `
})
export class ImageGalleryComponent {
  images = signal<Image[]>([]);

  constructor(private imageService: ImageService) {}

  ngOnInit() {
    this.imageService.getImages().subscribe(images => {
      this.images.set(images);
    });
  }
}

// Advanced: Handling complex hydration scenarios
@Component({
  selector: 'app-dynamic-content',
  standalone: true,
  template: `
    <div>
      <!-- Static content: hydrates perfectly -->
      <h1>{{ title }}</h1>
      
      <!-- Dynamic content: needs special handling -->
      <div [innerHTML]="sanitizedHtml"></div>
      
      <!-- Third-party components: may need ngSkipHydration -->
      <div ngSkipHydration>
        <third-party-widget></third-party-widget>
      </div>
      
      <!-- Conditionals: hydration matches server state -->
      <div *ngIf="showContent">
        Content rendered on server and hydrated
      </div>
    </div>
  `
})
export class DynamicContentComponent {
  title = 'Page Title';
  sanitizedHtml = '';
  showContent = true;

  constructor(
    private sanitizer: DomSanitizer,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  ngOnInit() {
    // Must produce same result on server and client
    const rawHtml = '<p>Safe HTML content</p>';
    this.sanitizedHtml = this.sanitizer.sanitize(
      SecurityContext.HTML,
      rawHtml
    ) || '';
  }
}

// Handling hydration errors
@Component({
  selector: 'app-hydration-debug',
  standalone: true,
  template: `
    <div>
      <!-- Hydration mismatch: different on server vs client -->
      <!-- BAD: -->
      <p>Current time: {{ Date.now() }}</p>
      
      <!-- GOOD: Use platform detection -->
      <p *ngIf="!isServer">Current time: {{ currentTime }}</p>
      
      <!-- Or use transferState to sync -->
      <p>Server time: {{ serverTime }}</p>
    </div>
  `
})
export class HydrationDebugComponent {
  isServer: boolean;
  currentTime: number = 0;
  serverTime: number = 0;

  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    private transferState: TransferState
  ) {
    this.isServer = isPlatformServer(platformId);
  }

  ngOnInit() {
    const TIME_KEY = makeStateKey<number>('serverTime');
    
    if (isPlatformServer(this.platformId)) {
      // On server: save time
      this.serverTime = Date.now();
      this.transferState.set(TIME_KEY, this.serverTime);
    } else {
      // On client: retrieve saved time
      this.serverTime = this.transferState.get(TIME_KEY, 0);
      this.currentTime = Date.now();
    }
  }
}
```

**Hydration Best Practices:**

```typescript
// 1. Ensure deterministic rendering
@Component({
  template: `
    <!-- BAD: Random values cause mismatch -->
    <div>{{ Math.random() }}</div>
    
    <!-- GOOD: Deterministic values -->
    <div>{{ staticValue }}</div>
  `
})
export class DeterministicComponent {
  staticValue = 'consistent value';
}

// 2. Handle browser-only code
@Component({
  template: `
    <div>
      <canvas #canvas></canvas>
    </div>
  `
})
export class CanvasComponent implements AfterViewInit {
  @ViewChild('canvas') canvasRef!: ElementRef<HTMLCanvasElement>;

  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

  ngAfterViewInit() {
    // Only run on browser
    if (isPlatformBrowser(this.platformId)) {
      const ctx = this.canvasRef.nativeElement.getContext('2d');
      // Draw on canvas
    }
  }
}

// 3. Use ngSkipHydration for problematic components
@Component({
  template: `
    <!-- Skip hydration for components that can't be hydrated -->
    <div ngSkipHydration>
      <google-maps-widget></google-maps-widget>
    </div>
  `
})
export class MapComponent {}

// 4. Optimize images for hydration
@Component({
  template: `
    <!-- Include dimensions to prevent layout shift -->
    <img [src]="imageUrl" 
         [width]="640" 
         [height]="480"
         [alt]="imageAlt">
  `
})
export class OptimizedImageComponent {
  imageUrl = '/assets/hero.jpg';
  imageAlt = 'Hero image';
}
```

**Follow-up Questions:**
1. What causes hydration mismatches?
2. How do you debug hydration errors?
3. When should you use ngSkipHydration?

**What Interviewers Look For:**
- Deep understanding of hydration process
- Knowledge of how to prevent mismatches
- Awareness of ngSkipHydration directive
- Understanding of performance benefits

### 3. What is the TransferState API? How does it prevent duplicate HTTP requests?

**Expected Answer:**

TransferState allows sharing data between server and client, preventing duplicate HTTP requests by transferring server-fetched data to the client.

**Implementation:**

```typescript
// Service with TransferState
import { Injectable, TransferState, makeStateKey } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

interface User {
  id: string;
  name: string;
  email: string;
}

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  getUser(id: string): Observable<User> {
    const USER_KEY = makeStateKey<User>(`user-${id}`);

    // Check if data exists in TransferState
    if (this.transferState.hasKey(USER_KEY)) {
      // Get data from TransferState and remove it
      const user = this.transferState.get(USER_KEY, null);
      this.transferState.remove(USER_KEY);
      return of(user!);
    }

    // Fetch from API
    return this.http.get<User>(`/api/users/${id}`).pipe(
      tap(user => {
        // On server: store in TransferState
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(USER_KEY, user);
        }
      })
    );
  }
}

// Advanced: Generic TransferState wrapper
@Injectable({ providedIn: 'root' })
export class TransferStateService {
  constructor(
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  get<T>(key: string, request: Observable<T>): Observable<T> {
    const stateKey = makeStateKey<T>(key);

    // Try to get from TransferState
    if (this.transferState.hasKey(stateKey)) {
      const data = this.transferState.get(stateKey, null);
      this.transferState.remove(stateKey);
      return of(data!);
    }

    // Fetch and store if on server
    return request.pipe(
      tap(data => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(stateKey, data);
        }
      })
    );
  }
}

// Usage in service
@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(
    private http: HttpClient,
    private transferStateService: TransferStateService
  ) {}

  getProducts(): Observable<Product[]> {
    return this.transferStateService.get(
      'products',
      this.http.get<Product[]>('/api/products')
    );
  }

  getProduct(id: string): Observable<Product> {
    return this.transferStateService.get(
      `product-${id}`,
      this.http.get<Product>(`/api/products/${id}`)
    );
  }
}

// Component using TransferState
@Component({
  selector: 'app-product-detail',
  standalone: true,
  template: `
    <div *ngIf="product$ | async as product">
      <h1>{{ product.name }}</h1>
      <p>{{ product.description }}</p>
      <p>{{ product.price | currency }}</p>
    </div>
  `
})
export class ProductDetailComponent implements OnInit {
  product$!: Observable<Product>;

  constructor(
    private route: ActivatedRoute,
    private productService: ProductService
  ) {}

  ngOnInit() {
    const id = this.route.snapshot.params['id'];
    // Only fetches once on server, reuses data on client
    this.product$ = this.productService.getProduct(id);
  }
}

// HTTP Interceptor with automatic TransferState
@Injectable()
export class TransferStateInterceptor implements HttpInterceptor {
  constructor(
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Only handle GET requests
    if (req.method !== 'GET') {
      return next.handle(req);
    }

    const stateKey = makeStateKey<any>(req.url);

    // Client: check TransferState
    if (isPlatformBrowser(this.platformId)) {
      if (this.transferState.hasKey(stateKey)) {
        const response = this.transferState.get(stateKey, null);
        this.transferState.remove(stateKey);
        return of(new HttpResponse({ body: response, status: 200 }));
      }
    }

    // Server: store in TransferState
    return next.handle(req).pipe(
      tap(event => {
        if (isPlatformServer(this.platformId) && event instanceof HttpResponse) {
          this.transferState.set(stateKey, event.body);
        }
      })
    );
  }
}

// Provide interceptor
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([transferStateInterceptor])
    )
  ]
};
```

**How It Works:**

```typescript
// 1. Server-side execution:
// Request: GET /products
// Server: Fetch from API -> Store in TransferState -> Render HTML

// 2. HTML includes TransferState data:
// <script id="transfer-state" type="application/json">
//   {"products": [{"id": 1, "name": "Product 1"}, ...]}
// </script>

// 3. Client-side execution:
// Angular bootstraps -> Reads TransferState -> Uses cached data -> No duplicate request!

// Timeline comparison:
// WITHOUT TransferState:
// Server: Fetch data (200ms) -> Render (50ms) -> Send HTML (250ms total)
// Client: Render (0ms) -> Fetch AGAIN (200ms) -> Render (50ms) -> Interactive (250ms)
// Total: 500ms

// WITH TransferState:
// Server: Fetch data (200ms) -> Render (50ms) -> Send HTML (250ms total)
// Client: Read TransferState (0ms) -> Render (50ms) -> Interactive (50ms)
// Total: 300ms (40% faster!)
```

**Advanced Patterns:**

```typescript
// Caching with expiration
@Injectable({ providedIn: 'root' })
export class CachedDataService {
  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  getWithExpiration<T>(
    key: string,
    request: Observable<T>,
    expirationMs: number = 60000
  ): Observable<T> {
    const stateKey = makeStateKey<{ data: T; timestamp: number }>(key);

    if (this.transferState.hasKey(stateKey)) {
      const cached = this.transferState.get(stateKey, null);
      
      if (cached) {
        const age = Date.now() - cached.timestamp;
        
        if (age < expirationMs) {
          this.transferState.remove(stateKey);
          return of(cached.data);
        }
      }
    }

    return request.pipe(
      tap(data => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(stateKey, {
            data,
            timestamp: Date.now()
          });
        }
      })
    );
  }
}

// Handling errors with TransferState
@Injectable({ providedIn: 'root' })
export class ErrorHandlingService {
  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  getWithErrorHandling<T>(key: string, request: Observable<T>): Observable<T> {
    const stateKey = makeStateKey<T>(key);
    const errorKey = makeStateKey<string>(`${key}-error`);

    // Check for error in TransferState
    if (this.transferState.hasKey(errorKey)) {
      const error = this.transferState.get(errorKey, null);
      this.transferState.remove(errorKey);
      return throwError(() => new Error(error!));
    }

    // Check for data in TransferState
    if (this.transferState.hasKey(stateKey)) {
      const data = this.transferState.get(stateKey, null);
      this.transferState.remove(stateKey);
      return of(data!);
    }

    // Fetch with error handling
    return request.pipe(
      tap(data => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(stateKey, data);
        }
      }),
      catchError(error => {
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(errorKey, error.message);
        }
        return throwError(() => error);
      })
    );
  }
}
```

**Follow-up Questions:**
1. What types of data can be stored in TransferState?
2. How large can TransferState data be?
3. Should you use TransferState for all API calls?

**What Interviewers Look For:**
- Understanding of duplicate request problem
- Knowledge of TransferState API
- Awareness of performance implications
- Understanding of when to use TransferState

### 4. How do you detect whether code is running on the server or browser? Provide examples.

**Expected Answer:**

Use PLATFORM_ID token with isPlatformBrowser/isPlatformServer guards to detect execution environment and conditionally execute platform-specific code.

**Platform Detection:**

```typescript
import { 
  PLATFORM_ID, 
  Inject, 
  isPlatformBrowser, 
  isPlatformServer 
} from '@angular/common';

// Basic platform detection
@Component({
  selector: 'app-platform-aware',
  standalone: true,
  template: `
    <div>
      <p>Running on: {{ platform }}</p>
      <p *ngIf="isBrowser">Browser-specific content</p>
      <p *ngIf="isServer">Server-specific content</p>
    </div>
  `
})
export class PlatformAwareComponent {
  isBrowser: boolean;
  isServer: boolean;
  platform: string;

  constructor(@Inject(PLATFORM_ID) private platformId: Object) {
    this.isBrowser = isPlatformBrowser(platformId);
    this.isServer = isPlatformServer(platformId);
    this.platform = this.isBrowser ? 'Browser' : 'Server';
  }
}

// Handling window/document access
@Component({
  selector: 'app-window-access',
  standalone: true,
  template: `
    <div>
      <p>Window width: {{ windowWidth }}</p>
      <p>Scroll position: {{ scrollPosition }}</p>
    </div>
  `
})
export class WindowAccessComponent implements OnInit {
  windowWidth = 0;
  scrollPosition = 0;

  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    @Inject(DOCUMENT) private document: Document
  ) {}

  ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Safe to access window
      this.windowWidth = window.innerWidth;
      
      window.addEventListener('scroll', () => {
        this.scrollPosition = window.scrollY;
      });

      // Safe to access document
      const element = this.document.querySelector('.my-element');
    }
  }
}

// Lazy loading browser-only libraries
@Component({
  selector: 'app-chart',
  standalone: true,
  template: `
    <div #chartContainer></div>
  `
})
export class ChartComponent implements AfterViewInit {
  @ViewChild('chartContainer') container!: ElementRef;

  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

  async ngAfterViewInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Dynamically import browser-only library
      const { Chart } = await import('chart.js');
      
      new Chart(this.container.nativeElement, {
        type: 'bar',
        data: {
          labels: ['Jan', 'Feb', 'Mar'],
          datasets: [{
            label: 'Sales',
            data: [12, 19, 3]
          }]
        }
      });
    }
  }
}

// Service with platform detection
@Injectable({ providedIn: 'root' })
export class StorageService {
  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

  setItem(key: string, value: string): void {
    if (isPlatformBrowser(this.platformId)) {
      localStorage.setItem(key, value);
    }
  }

  getItem(key: string): string | null {
    if (isPlatformBrowser(this.platformId)) {
      return localStorage.getItem(key);
    }
    return null;
  }
}

// Analytics service with platform detection
@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

  trackPageView(page: string): void {
    if (isPlatformBrowser(this.platformId)) {
      // Only track on browser
      if (typeof gtag !== 'undefined') {
        gtag('event', 'page_view', { page_path: page });
      }
    }
  }

  trackEvent(category: string, action: string): void {
    if (isPlatformBrowser(this.platformId)) {
      if (typeof gtag !== 'undefined') {
        gtag('event', action, { event_category: category });
      }
    }
  }
}

// Handling third-party scripts
@Component({
  selector: 'app-google-maps',
  standalone: true,
  template: `
    <div #mapContainer style="height: 400px;"></div>
  `
})
export class GoogleMapsComponent implements AfterViewInit, OnDestroy {
  @ViewChild('mapContainer') mapContainer!: ElementRef;
  private map: any;

  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

  async ngAfterViewInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Load Google Maps API
      await this.loadGoogleMapsScript();
      
      // Initialize map
      this.map = new google.maps.Map(this.mapContainer.nativeElement, {
        center: { lat: 37.7749, lng: -122.4194 },
        zoom: 12
      });
    }
  }

  private loadGoogleMapsScript(): Promise<void> {
    return new Promise((resolve) => {
      if (typeof google !== 'undefined') {
        resolve();
        return;
      }

      const script = document.createElement('script');
      script.src = `https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY`;
      script.onload = () => resolve();
      document.body.appendChild(script);
    });
  }

  ngOnDestroy() {
    if (this.map) {
      // Cleanup map instance
      google.maps.event.clearInstanceListeners(this.map);
    }
  }
}

// Custom injection token for platform-specific implementations
export const WINDOW = new InjectionToken<Window | null>('Window object', {
  factory: () => {
    const platformId = inject(PLATFORM_ID);
    return isPlatformBrowser(platformId) ? window : null;
  }
});

@Component({
  selector: 'app-window-injection',
  standalone: true
})
export class WindowInjectionComponent {
  constructor(@Inject(WINDOW) private window: Window | null) {
    if (this.window) {
      console.log('Window width:', this.window.innerWidth);
    }
  }
}
```

**Follow-up Questions:**
1. Why can't you use window directly in Angular Universal?
2. What happens if you access window on the server?
3. How do you handle conditional imports based on platform?

**What Interviewers Look For:**
- Understanding of platform differences
- Proper use of isPlatformBrowser/isPlatformServer
- Knowledge of handling browser-only APIs
- Awareness of lazy loading for platform-specific code

### 5. What's the difference between SSR and Prerendering? When would you use each?

**Expected Answer:**

SSR renders pages on-demand per request, while prerendering generates static HTML at build time. Use SSR for dynamic content, prerendering for static pages.

**Comparison:**

```typescript
// SSR Configuration (server.ts)
// Renders on every request
server.get('*', (req, res) => {
  commonEngine.render({
    bootstrap,
    documentFilePath: indexHtml,
    url: req.url, // Different for each request
    providers: [{ provide: APP_BASE_HREF, useValue: baseUrl }]
  })
  .then(html => res.send(html));
});

// Prerendering Configuration (angular.json)
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "prerender": true,
            "prerenderRoutes": [
              "/",
              "/about",
              "/contact",
              "/products",
              "/products/1",
              "/products/2"
            ]
          }
        }
      }
    }
  }
}

// Dynamic route prerendering
// prerender-routes.ts
export async function generateRoutes(): Promise<string[]> {
  const response = await fetch('https://api.example.com/products');
  const products = await response.json();
  
  return [
    '/',
    '/about',
    '/contact',
    ...products.map((p: any) => `/products/${p.id}`)
  ];
}
```

**When to Use Each:**

| Scenario | Use |
|----------|-----|
| Blog posts | Prerendering |
| Product catalog (static) | Prerendering |
| User dashboards | SSR |
| Real-time data | SSR |
| Personalized content | SSR |
| Documentation | Prerendering |
| Marketing pages | Prerendering |
| E-commerce product pages | Hybrid (prerender + ISR) |

**Hybrid Approach:**

```typescript
// Combine SSR and Prerendering
// Prerender common routes, SSR for dynamic ones

// angular.json
{
  "prerender": true,
  "prerenderRoutes": [
    "/",
    "/about",
    "/contact",
    "/products" // List page
  ]
  // Product detail pages use SSR
}

// Route configuration
export const routes: Routes = [
  { path: '', component: HomeComponent }, // Prerendered
  { path: 'about', component: AboutComponent }, // Prerendered
  { path: 'products', component: ProductsComponent }, // Prerendered
  { path: 'products/:id', component: ProductDetailComponent }, // SSR
  { path: 'dashboard', component: DashboardComponent } // SSR
];

// Incremental Static Regeneration (ISR) pattern
// Cache prerendered pages with revalidation
@Injectable({ providedIn: 'root' })
export class CacheService {
  private cache = new Map<string, { html: string; timestamp: number }>();
  private revalidateAfter = 3600000; // 1 hour

  getCachedPage(url: string): string | null {
    const cached = this.cache.get(url);
    
    if (!cached) return null;
    
    const age = Date.now() - cached.timestamp;
    if (age > this.revalidateAfter) {
      // Stale, but return it while regenerating in background
      this.regenerateInBackground(url);
      return cached.html;
    }
    
    return cached.html;
  }

  setCachedPage(url: string, html: string): void {
    this.cache.set(url, { html, timestamp: Date.now() });
  }

  private regenerateInBackground(url: string): void {
    // Trigger regeneration asynchronously
    setTimeout(() => {
      // Re-render and update cache
    }, 0);
  }
}
```

**Performance Comparison:**

```typescript
// Prerendering
// Build time: High (renders all routes)
// Request time: Instant (serve static files)
// Scalability: Excellent (CDN-friendly)
// Flexibility: Low (rebuild to update)
// Cost: Low (static hosting)

// SSR
// Build time: Low (no prerendering)
// Request time: Medium (render per request)
// Scalability: Medium (server resources)
// Flexibility: High (dynamic per request)
// Cost: Higher (server hosting)

// Example build times:
// Prerendering 1000 routes: ~10 minutes
// SSR: 1 minute (no prerendering)

// Example request times:
// Prerendered page: 10ms (CDN)
// SSR page: 100-300ms (rendering)
```

**Follow-up Questions:**
1. Can you update prerendered content without rebuilding?
2. How do you handle 404 pages with prerendering?
3. What is Incremental Static Regeneration (ISR)?

**What Interviewers Look For:**
- Understanding of trade-offs between SSR and prerendering
- Knowledge of when to use each approach
- Awareness of hybrid strategies
- Understanding of performance implications

## SEO Considerations

### 6. How does Angular Universal improve SEO? What are the key considerations?

**Expected Answer:**

Angular Universal enables search engines to crawl fully rendered HTML with proper meta tags, improving discoverability and social media sharing.

**Implementation:**

```typescript
// Meta tag service
@Injectable({ providedIn: 'root' })
export class SeoService {
  constructor(
    private meta: Meta,
    private title: Title,
    @Inject(DOCUMENT) private document: Document
  ) {}

  updateMetaTags(config: {
    title: string;
    description: string;
    image?: string;
    url?: string;
    type?: string;
  }): void {
    // Update title
    this.title.setTitle(config.title);

    // Update meta tags
    this.meta.updateTag({ name: 'description', content: config.description });

    // Open Graph tags (Facebook, LinkedIn)
    this.meta.updateTag({ property: 'og:title', content: config.title });
    this.meta.updateTag({ property: 'og:description', content: config.description });
    this.meta.updateTag({ property: 'og:type', content: config.type || 'website' });
    
    if (config.image) {
      this.meta.updateTag({ property: 'og:image', content: config.image });
    }
    
    if (config.url) {
      this.meta.updateTag({ property: 'og:url', content: config.url });
    }

    // Twitter Card tags
    this.meta.updateTag({ name: 'twitter:card', content: 'summary_large_image' });
    this.meta.updateTag({ name: 'twitter:title', content: config.title });
    this.meta.updateTag({ name: 'twitter:description', content: config.description });
    
    if (config.image) {
      this.meta.updateTag({ name: 'twitter:image', content: config.image });
    }

    // Canonical URL
    if (config.url) {
      this.updateCanonicalUrl(config.url);
    }
  }

  private updateCanonicalUrl(url: string): void {
    let link: HTMLLinkElement | null = this.document.querySelector('link[rel="canonical"]');
    
    if (!link) {
      link = this.document.createElement('link');
      link.setAttribute('rel', 'canonical');
      this.document.head.appendChild(link);
    }
    
    link.setAttribute('href', url);
  }

  addStructuredData(data: any): void {
    let script: HTMLScriptElement | null = this.document.querySelector('script[type="application/ld+json"]');
    
    if (!script) {
      script = this.document.createElement('script');
      script.type = 'application/ld+json';
      this.document.head.appendChild(script);
    }
    
    script.textContent = JSON.stringify(data);
  }
}

// Component with SEO
@Component({
  selector: 'app-blog-post',
  standalone: true,
  template: `
    <article *ngIf="post">
      <h1>{{ post.title }}</h1>
      <img [src]="post.image" [alt]="post.title">
      <div [innerHTML]="post.content"></div>
    </article>
  `
})
export class BlogPostComponent implements OnInit {
  post: BlogPost | null = null;

  constructor(
    private route: ActivatedRoute,
    private blogService: BlogService,
    private seoService: SeoService
  ) {}

  ngOnInit() {
    const id = this.route.snapshot.params['id'];
    
    this.blogService.getPost(id).subscribe(post => {
      this.post = post;
      
      // Update SEO tags
      this.seoService.updateMetaTags({
        title: post.title,
        description: post.excerpt,
        image: post.image,
        url: `https://example.com/blog/${post.slug}`,
        type: 'article'
      });

      // Add structured data
      this.seoService.addStructuredData({
        '@context': 'https://schema.org',
        '@type': 'BlogPosting',
        headline: post.title,
        image: post.image,
        datePublished: post.publishedDate,
        dateModified: post.modifiedDate,
        author: {
          '@type': 'Person',
          name: post.author
        },
        description: post.excerpt
      });
    });
  }
}
```

**Follow-up Questions:**
1. How do you verify that search engines can crawl your Angular app?
2. What are JSON-LD structured data and why are they important?
3. How do you handle dynamic sitemaps with Angular Universal?

**What Interviewers Look For:**
- Understanding of SEO fundamentals
- Knowledge of Meta and Title services
- Awareness of Open Graph and Twitter Cards
- Understanding of structured data

## Key Takeaways

1. **Angular Universal Architecture**: Renders Angular applications on the server, sending fully rendered HTML for faster First Contentful Paint and better SEO.

2. **Hydration Process**: Attaches Angular to server-rendered HTML without re-rendering, preventing content flickering and improving user experience.

3. **TransferState API**: Shares data between server and client, preventing duplicate HTTP requests and improving performance.

4. **Platform Detection**: Use isPlatformBrowser/isPlatformServer to conditionally execute platform-specific code and avoid errors.

5. **SSR vs Prerendering**: SSR renders on-demand for dynamic content, prerendering generates static HTML at build time for static pages.

6. **SEO Optimization**: Angular Universal enables proper meta tags, Open Graph tags, and structured data for search engines and social media.

7. **Browser-Only Code**: Lazy load browser-specific libraries and use platform detection to avoid server-side errors.

8. **Hydration Best Practices**: Ensure deterministic rendering, use ngSkipHydration for problematic components, and handle errors gracefully.

9. **Performance Benefits**: SSR improves initial load time, Core Web Vitals, and user experience on slow devices and networks.

10. **Hybrid Approaches**: Combine SSR and prerendering for optimal performance and flexibility, using ISR patterns where appropriate.

## Red Flags to Avoid

- Accessing window/document without platform detection
- Not using TransferState for API calls, causing duplicate requests
- Creating hydration mismatches with non-deterministic rendering
- Not handling third-party libraries that require browser APIs
- Forgetting to update meta tags for SEO
- Not testing SSR build separately from client build
- Ignoring hydration errors in console
- Using SSR when prerendering would suffice
- Not implementing proper error handling for SSR failures
- Forgetting to add structured data for better SEO

## Interview Preparation Tips

- Understand the full SSR flow from request to hydration
- Practice implementing TransferState in services
- Know how to debug hydration mismatches
- Be familiar with Meta and Title services
- Understand the difference between SSR and prerendering
- Know how to handle browser-only code safely
- Practice implementing structured data
- Understand performance metrics (FCP, LCP, TTI)
- Be ready to discuss trade-offs of SSR
- Know how to test and verify SSR implementation
