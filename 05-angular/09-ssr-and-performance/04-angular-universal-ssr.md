# Angular Universal and Server-Side Rendering (SSR)

## The Idea

**In plain English:** Server-Side Rendering (SSR) means your web page is built and filled with content on a computer in the cloud (the "server") before it travels to your browser, so you see a complete page almost instantly instead of staring at a blank screen while your browser does all the work. Angular Universal is the tool that makes Angular apps capable of doing this.

**Real-world analogy:** Imagine ordering a meal at a restaurant. Normally (client-side rendering) the waiter brings you raw ingredients and a portable stove, and you have to cook the meal yourself at your table before you can eat. With SSR, the kitchen cooks the full meal first, then the waiter delivers a ready-to-eat plate straight to you.

- The kitchen cooking the meal = the server running Angular and producing full HTML
- The ready-to-eat plate delivered to your table = the complete HTML page sent to your browser
- You picking up your fork and starting to eat = the browser "hydrating" (attaching interactivity to the already-visible page)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding SSR](#understanding-ssr)
3. [Angular Universal Setup](#angular-universal-setup)
4. [Platform-Server](#platform-server)
5. [App Server Configuration](#app-server-configuration)
6. [TransferState API](#transferstate-api)
7. [SSR Lifecycle](#ssr-lifecycle)
8. [Browser vs Server Code](#browser-vs-server-code)
9. [SEO Optimization](#seo-optimization)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)
13. [Key Takeaways](#key-takeaways)
14. [Resources](#resources)

## Introduction

Angular Universal enables server-side rendering (SSR) for Angular applications, rendering pages on the server before sending them to the client. SSR provides significant benefits including improved initial load performance, better SEO, enhanced social media sharing, and improved accessibility. This comprehensive guide covers everything from basic Universal setup to advanced patterns for building production-ready SSR applications.

## Understanding SSR

### Client-Side Rendering (CSR) vs Server-Side Rendering (SSR)

```typescript
// Client-Side Rendering Flow:
// 1. Browser requests page
// 2. Server sends minimal HTML + JavaScript bundles
// 3. Browser downloads and parses JavaScript
// 4. Angular bootstraps and renders content
// 5. User sees content (can take several seconds)

// Server-Side Rendering Flow:
// 1. Browser requests page
// 2. Server runs Angular, generates full HTML
// 3. Server sends complete HTML to browser
// 4. User sees content immediately
// 5. Browser downloads JavaScript (hydration)
// 6. Angular takes over and makes page interactive
```

### Benefits of SSR

```typescript
/**
 * SSR Benefits:
 * 
 * 1. Improved Performance
 *    - Faster First Contentful Paint (FCP)
 *    - Better perceived performance
 *    - Faster Time to Interactive (TTI) on slow networks
 * 
 * 2. Better SEO
 *    - Search engines see fully rendered content
 *    - Meta tags available for crawlers
 *    - Improved search rankings
 * 
 * 3. Social Media Sharing
 *    - Open Graph tags rendered on server
 *    - Proper preview cards on Facebook/Twitter
 *    - Dynamic meta tags per route
 * 
 * 4. Accessibility
 *    - Content available without JavaScript
 *    - Better for screen readers
 *    - Works with JavaScript disabled (basic functionality)
 * 
 * 5. Reduced Client Load
 *    - Less JavaScript execution on client
 *    - Better performance on low-end devices
 *    - Improved battery life on mobile
 */
```

## Angular Universal Setup

### Installing Angular Universal

```bash
# Add Angular Universal to existing project
ng add @angular/ssr

# This automatically:
# - Installs required packages
# - Updates angular.json
# - Creates server.ts
# - Adds SSR build configuration
# - Updates app.config files
```

### Project Structure After Setup

```text
my-app/
├── src/
│   ├── app/
│   │   ├── app.config.ts          # Client config
│   │   ├── app.config.server.ts   # Server config
│   │   └── app.routes.ts
│   ├── main.ts                     # Client bootstrap
│   └── main.server.ts              # Server bootstrap
├── server.ts                       # Express server
└── angular.json                    # Updated with SSR config
```

### Basic SSR Application

```typescript
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration } from '@angular/platform-browser';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideClientHydration() // Enable hydration
  ]
};

// src/app/app.config.server.ts
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { appConfig } from './app.config';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering()
  ]
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

### Main Server Bootstrap

```typescript
// src/main.server.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { config } from './app/app.config.server';

const bootstrap = () => bootstrapApplication(AppComponent, config);

export default bootstrap;
```

### Express Server Setup

```typescript
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

  // Serve static files
  server.get('*.*', express.static(browserDistFolder, {
    maxAge: '1y'
  }));

  // SSR for all other routes
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
```

## Platform-Server

### Understanding Platform-Server

```typescript
// Platform-server provides server-specific implementations
// of browser APIs and Angular platform abstractions

import { 
  renderApplication,
  renderModule,
  ServerModule 
} from '@angular/platform-server';

/**
 * Key Platform-Server Features:
 * 
 * 1. Server Rendering Engine
 *    - Renders Angular components to HTML strings
 *    - Executes Angular change detection on server
 * 
 * 2. DOM Abstraction
 *    - Provides Domino (server-side DOM implementation)
 *    - Makes browser APIs available on server
 * 
 * 3. HTTP Interception
 *    - Can intercept and mock HTTP requests
 *    - Useful for absolute URL transformation
 * 
 * 4. Request/Response Context
 *    - Access to server request data
 *    - Can set response headers, status codes
 */
```

### Using renderApplication

```typescript
// Modern approach with standalone components
import { renderApplication } from '@angular/platform-server';
import { AppComponent } from './app/app.component';
import { config } from './app/app.config.server';

async function renderPage(url: string) {
  const html = await renderApplication(AppComponent, {
    appId: 'my-app',
    document: '<app-root></app-root>',
    url: url,
    providers: config.providers
  });

  return html;
}
```

### Server-Side HTTP Interceptors

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';

export const serverInterceptor: HttpInterceptorFn = (req, next) => {
  const platformId = inject(PLATFORM_ID);

  if (isPlatformServer(platformId)) {
    // Transform relative URLs to absolute URLs on server
    const serverUrl = 'https://api.example.com';
    
    if (!req.url.startsWith('http')) {
      const absoluteUrl = `${serverUrl}${req.url}`;
      req = req.clone({ url: absoluteUrl });
    }
  }

  return next(req);
};

// app.config.server.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const serverConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([serverInterceptor])
    )
  ]
};
```

## App Server Configuration

### Server-Specific Providers

```typescript
// app.config.server.ts
import { ApplicationConfig, mergeApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { appConfig } from './app.config';
import { serverInterceptor } from './interceptors/server.interceptor';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideHttpClient(
      withInterceptors([serverInterceptor])
    )
  ]
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

### Environment-Specific Configuration

```typescript
// environment.server.ts
export const environment = {
  production: true,
  apiUrl: 'http://localhost:3000/api', // Internal API URL
  serverSide: true
};

// environment.ts
export const environment = {
  production: false,
  apiUrl: 'https://api.example.com', // External API URL
  serverSide: false
};

// Usage in service
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { environment } from '../environments/environment';

@Injectable({ providedIn: 'root' })
export class ApiService {
  private platformId = inject(PLATFORM_ID);
  
  getApiUrl(): string {
    if (isPlatformServer(this.platformId)) {
      // Use internal URL on server
      return 'http://localhost:3000/api';
    }
    // Use external URL on client
    return environment.apiUrl;
  }
}
```

## TransferState API

TransferState allows you to transfer data from server to client, avoiding duplicate HTTP requests during hydration.

### Basic TransferState Usage

```typescript
import { Component, OnInit, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { 
  TransferState, 
  makeStateKey,
  provideClientHydration,
  withHttpTransferCacheOptions
} from '@angular/platform-browser';
import { HttpClient } from '@angular/common/http';

const USERS_KEY = makeStateKey<User[]>('users');

interface User {
  id: number;
  name: string;
}

@Component({
  selector: 'app-users',
  standalone: true,
  template: `
    <div *ngFor="let user of users">
      {{ user.name }}
    </div>
  `
})
export class UsersComponent implements OnInit {
  private http = inject(HttpClient);
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);
  
  users: User[] = [];

  ngOnInit() {
    // Check if data exists in transfer state
    const cachedUsers = this.transferState.get(USERS_KEY, null);
    
    if (cachedUsers) {
      // Use cached data (client-side)
      this.users = cachedUsers;
      this.transferState.remove(USERS_KEY); // Clean up
    } else {
      // Fetch data (server-side or client without cache)
      this.http.get<User[]>('/api/users').subscribe(users => {
        this.users = users;
        
        // Store in transfer state if on server
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(USERS_KEY, users);
        }
      });
    }
  }
}
```

### Automatic HTTP Transfer Cache

```typescript
// app.config.ts - Enable automatic HTTP caching
import { ApplicationConfig } from '@angular/core';
import { 
  provideClientHydration, 
  withHttpTransferCacheOptions 
} from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withFetch()),
    provideClientHydration(
      withHttpTransferCacheOptions({
        includePostRequests: true, // Cache POST requests too
        includeRequestsWithAuthHeaders: false, // Don't cache auth requests
        includeHeaders: ['Accept', 'Content-Type'], // Headers to include
      })
    )
  ]
};

// Now HTTP requests are automatically cached without manual TransferState
@Component({
  selector: 'app-auto-cache',
  standalone: true,
  template: `<div *ngFor="let item of items$ | async">{{ item.name }}</div>`
})
export class AutoCacheComponent {
  private http = inject(HttpClient);
  
  // This request will be cached automatically
  items$ = this.http.get<any[]>('/api/items');
}
```

### Custom TransferState Service

```typescript
import { Injectable, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { TransferState, makeStateKey } from '@angular/platform-browser';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class TransferStateService {
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  /**
   * Get or fetch data with automatic transfer state management
   */
  getOrFetch<T>(
    key: string,
    fetchFn: () => Observable<T>
  ): Observable<T> {
    const stateKey = makeStateKey<T>(key);
    const cached = this.transferState.get(stateKey, null);

    if (cached !== null) {
      // Return cached data and clean up
      this.transferState.remove(stateKey);
      return of(cached);
    }

    // Fetch data
    return fetchFn().pipe(
      tap(data => {
        // Store in transfer state if on server
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(stateKey, data);
        }
      })
    );
  }

  /**
   * Set transfer state data
   */
  set<T>(key: string, value: T): void {
    if (isPlatformServer(this.platformId)) {
      const stateKey = makeStateKey<T>(key);
      this.transferState.set(stateKey, value);
    }
  }

  /**
   * Get transfer state data
   */
  get<T>(key: string, defaultValue: T): T {
    const stateKey = makeStateKey<T>(key);
    return this.transferState.get(stateKey, defaultValue);
  }
}

// Usage
@Component({
  selector: 'app-data',
  standalone: true,
  template: `<div>{{ data$ | async | json }}</div>`
})
export class DataComponent {
  private http = inject(HttpClient);
  private transferStateService = inject(TransferStateService);

  data$ = this.transferStateService.getOrFetch(
    'app-data',
    () => this.http.get('/api/data')
  );
}
```

## SSR Lifecycle

### Component Lifecycle in SSR

```typescript
import { 
  Component, 
  OnInit, 
  AfterViewInit, 
  inject, 
  PLATFORM_ID 
} from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

@Component({
  selector: 'app-lifecycle',
  standalone: true,
  template: `<div>Lifecycle Demo</div>`
})
export class LifecycleComponent implements OnInit, AfterViewInit {
  private platformId = inject(PLATFORM_ID);

  constructor() {
    console.log('Constructor called on:', 
      isPlatformServer(this.platformId) ? 'Server' : 'Browser'
    );
  }

  ngOnInit() {
    console.log('ngOnInit called on:', 
      isPlatformServer(this.platformId) ? 'Server' : 'Browser'
    );
    
    if (isPlatformBrowser(this.platformId)) {
      // Browser-only initialization
      console.log('Initializing browser-specific features');
    }
  }

  ngAfterViewInit() {
    // This runs on both server and browser
    console.log('ngAfterViewInit called');
    
    if (isPlatformBrowser(this.platformId)) {
      // Safe to access DOM here on browser
      console.log('DOM is ready');
    }
  }
}

/**
 * Lifecycle Order in SSR:
 * 
 * Server:
 * 1. Constructor
 * 2. ngOnInit
 * 3. ngAfterViewInit
 * 4. Render to HTML string
 * 5. Send HTML to browser
 * 
 * Browser (Hydration):
 * 1. Parse HTML
 * 2. Download JavaScript
 * 3. Constructor (again!)
 * 4. ngOnInit (again!)
 * 5. Hydrate existing DOM
 * 6. ngAfterViewInit (again!)
 * 7. Application interactive
 */
```

## Browser vs Server Code

### Platform Detection

```typescript
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class PlatformService {
  private platformId = inject(PLATFORM_ID);

  get isBrowser(): boolean {
    return isPlatformBrowser(this.platformId);
  }

  get isServer(): boolean {
    return isPlatformServer(this.platformId);
  }

  executeOnBrowser(fn: () => void): void {
    if (this.isBrowser) {
      fn();
    }
  }

  executeOnServer(fn: () => void): void {
    if (this.isServer) {
      fn();
    }
  }
}
```

### Browser-Only Code

```typescript
import { Component, OnInit, inject, PLATFORM_ID, afterNextRender } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-browser-only',
  standalone: true,
  template: `<div>{{ message }}</div>`
})
export class BrowserOnlyComponent implements OnInit {
  private platformId = inject(PLATFORM_ID);
  message = '';

  constructor() {
    // Modern approach: use afterNextRender
    afterNextRender(() => {
      // This only runs in the browser
      console.log('Browser only code');
      this.message = 'Rendered in browser';
      
      // Safe to use window, document, etc.
      console.log('Window width:', window.innerWidth);
    });
  }

  ngOnInit() {
    // Classic approach: platform detection
    if (isPlatformBrowser(this.platformId)) {
      // Browser-only code
      this.initBrowserFeatures();
    } else {
      // Server fallback
      this.message = 'Server rendered';
    }
  }

  private initBrowserFeatures() {
    // localStorage
    const saved = localStorage.getItem('data');
    
    // window APIs
    console.log('Screen:', window.screen.width);
    
    // Document APIs
    console.log('Cookies:', document.cookie);
    
    // Event listeners
    window.addEventListener('resize', () => {
      console.log('Resized');
    });
  }
}
```

### Conditional Feature Loading

```typescript
import { Component, OnInit, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-conditional-feature',
  standalone: true,
  template: `<div #chart></div>`
})
export class ConditionalFeatureComponent implements OnInit {
  private platformId = inject(PLATFORM_ID);

  async ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Lazy load browser-only library
      const ChartJS = await import('chart.js');
      this.initChart(ChartJS);
    }
  }

  private initChart(ChartJS: any) {
    // Initialize chart with imported library
    console.log('Chart initialized');
  }
}
```

## SEO Optimization

### Dynamic Meta Tags

```typescript
import { Component, OnInit, inject } from '@angular/core';
import { Meta, Title } from '@angular/platform-browser';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-blog-post',
  standalone: true,
  template: `
    <article>
      <h1>{{ post.title }}</h1>
      <p>{{ post.content }}</p>
    </article>
  `
})
export class BlogPostComponent implements OnInit {
  private meta = inject(Meta);
  private title = inject(Title);
  private route = inject(ActivatedRoute);

  post = {
    title: 'My Blog Post',
    content: 'Post content...',
    description: 'Post description',
    image: 'https://example.com/image.jpg',
    author: 'John Doe'
  };

  ngOnInit() {
    // Set page title
    this.title.setTitle(this.post.title);

    // Set meta tags
    this.meta.updateTag({ 
      name: 'description', 
      content: this.post.description 
    });
    
    this.meta.updateTag({ 
      name: 'author', 
      content: this.post.author 
    });

    // Open Graph tags for social media
    this.meta.updateTag({ 
      property: 'og:title', 
      content: this.post.title 
    });
    
    this.meta.updateTag({ 
      property: 'og:description', 
      content: this.post.description 
    });
    
    this.meta.updateTag({ 
      property: 'og:image', 
      content: this.post.image 
    });
    
    this.meta.updateTag({ 
      property: 'og:type', 
      content: 'article' 
    });

    // Twitter Card tags
    this.meta.updateTag({ 
      name: 'twitter:card', 
      content: 'summary_large_image' 
    });
    
    this.meta.updateTag({ 
      name: 'twitter:title', 
      content: this.post.title 
    });
    
    this.meta.updateTag({ 
      name: 'twitter:description', 
      content: this.post.description 
    });
    
    this.meta.updateTag({ 
      name: 'twitter:image', 
      content: this.post.image 
    });
  }
}
```

### SEO Service

```typescript
import { Injectable, inject } from '@angular/core';
import { Meta, Title } from '@angular/platform-browser';

export interface SeoData {
  title: string;
  description: string;
  image?: string;
  url?: string;
  type?: string;
  author?: string;
  keywords?: string[];
}

@Injectable({ providedIn: 'root' })
export class SeoService {
  private meta = inject(Meta);
  private title = inject(Title);

  updateTags(data: SeoData) {
    // Title
    this.title.setTitle(data.title);

    // Basic meta tags
    this.meta.updateTag({ name: 'description', content: data.description });
    
    if (data.author) {
      this.meta.updateTag({ name: 'author', content: data.author });
    }
    
    if (data.keywords) {
      this.meta.updateTag({ 
        name: 'keywords', 
        content: data.keywords.join(', ') 
      });
    }

    // Open Graph
    this.meta.updateTag({ property: 'og:title', content: data.title });
    this.meta.updateTag({ property: 'og:description', content: data.description });
    
    if (data.image) {
      this.meta.updateTag({ property: 'og:image', content: data.image });
    }
    
    if (data.url) {
      this.meta.updateTag({ property: 'og:url', content: data.url });
    }
    
    this.meta.updateTag({ 
      property: 'og:type', 
      content: data.type || 'website' 
    });

    // Twitter Card
    this.meta.updateTag({ name: 'twitter:card', content: 'summary_large_image' });
    this.meta.updateTag({ name: 'twitter:title', content: data.title });
    this.meta.updateTag({ name: 'twitter:description', content: data.description });
    
    if (data.image) {
      this.meta.updateTag({ name: 'twitter:image', content: data.image });
    }
  }

  clearTags() {
    this.meta.removeTag('name="description"');
    this.meta.removeTag('name="author"');
    this.meta.removeTag('name="keywords"');
    this.meta.removeTag('property="og:title"');
    this.meta.removeTag('property="og:description"');
    this.meta.removeTag('property="og:image"');
    this.meta.removeTag('property="og:url"');
    this.meta.removeTag('property="og:type"');
    this.meta.removeTag('name="twitter:card"');
    this.meta.removeTag('name="twitter:title"');
    this.meta.removeTag('name="twitter:description"');
    this.meta.removeTag('name="twitter:image"');
  }
}

// Usage
@Component({
  selector: 'app-page',
  standalone: true,
  template: `<h1>Page</h1>`
})
export class PageComponent implements OnInit {
  private seoService = inject(SeoService);

  ngOnInit() {
    this.seoService.updateTags({
      title: 'My Page Title',
      description: 'Page description for SEO',
      image: 'https://example.com/image.jpg',
      url: 'https://example.com/page',
      type: 'article',
      keywords: ['angular', 'ssr', 'seo']
    });
  }
}
```

## Common Mistakes

### 1. Accessing Window/Document Without Checking

```typescript
// BAD: Will crash on server
@Component({ template: '' })
export class BadComponent {
  ngOnInit() {
    const width = window.innerWidth; // Error on server!
  }
}

// GOOD: Platform detection
@Component({ template: '' })
export class GoodComponent {
  private platformId = inject(PLATFORM_ID);

  ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      const width = window.innerWidth;
    }
  }
}
```

### 2. Not Using TransferState

```typescript
// BAD: Duplicate HTTP requests
@Component({ template: '' })
export class BadComponent {
  data$ = this.http.get('/api/data'); // Fetches on server AND client
}

// GOOD: Use TransferState or HTTP cache
@Component({ template: '' })
export class GoodComponent {
  data$ = this.transferStateService.getOrFetch(
    'data',
    () => this.http.get('/api/data')
  ); // Fetches once, transfers to client
}
```

### 3. Memory Leaks in Server Rendering

```typescript
// BAD: Never cleaned up on server
@Component({ template: '' })
export class BadComponent implements OnInit {
  ngOnInit() {
    interval(1000).subscribe(() => {
      console.log('Tick'); // Leaks on server!
    });
  }
}

// GOOD: Clean up subscriptions
@Component({ template: '' })
export class GoodComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => console.log('Tick'));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Best Practices

### 1. Use afterNextRender for Browser Code

```typescript
import { afterNextRender } from '@angular/core';

@Component({ template: '' })
export class BrowserCodeComponent {
  constructor() {
    afterNextRender(() => {
      // Guaranteed to run only in browser
      console.log('Browser only');
    });
  }
}
```

### 2. Optimize Server Response Time

```typescript
// server.ts - Add timeout
server.get('*', (req, res, next) => {
  const timeout = setTimeout(() => {
    res.status(504).send('Server timeout');
  }, 5000); // 5 second timeout

  commonEngine.render({
    bootstrap,
    documentFilePath: indexHtml,
    url: req.url,
    publicPath: browserDistFolder,
  })
  .then(html => {
    clearTimeout(timeout);
    res.send(html);
  })
  .catch(err => {
    clearTimeout(timeout);
    next(err);
  });
});
```

### 3. Prerender Static Pages

```bash
# angular.json - Add prerender configuration
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "prerender": {
                "routesFile": "routes.txt"
              }
            }
          }
        }
      }
    }
  }
}

# routes.txt
/
/about
/contact
/blog/post-1
/blog/post-2
```

### 4. Use Proper HTTP Caching Headers

```typescript
// server.ts
server.get('*.*', express.static(browserDistFolder, {
  maxAge: '1y', // Cache static assets for 1 year
  immutable: true
}));

server.get('*', (req, res, next) => {
  // Don't cache HTML pages
  res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate');
  
  commonEngine.render(/*...*/)
    .then(html => res.send(html));
});
```

## Interview Questions

### Q1: What is Angular Universal?

**Answer:** Angular Universal is the official server-side rendering solution for Angular. It renders Angular applications on the server, generating full HTML that's sent to the browser, improving initial load time, SEO, and social media sharing.

### Q2: What's the difference between SSR and pre-rendering?

**Answer:** SSR renders pages on-demand when requested. Pre-rendering generates static HTML files at build time for specific routes. Pre-rendering is faster but only works for static content, while SSR can handle dynamic content.

### Q3: What is hydration in Angular?

**Answer:** Hydration is the process where Angular takes over the server-rendered HTML in the browser, attaching event listeners and making the application interactive without re-rendering the DOM.

### Q4: Why use TransferState?

**Answer:** TransferState transfers data from server to client, avoiding duplicate HTTP requests during hydration. Data fetched on the server is serialized and sent to the browser, where Angular reuses it.

### Q5: How do you handle browser-only code in SSR?

**Answer:** Use platform detection (`isPlatformBrowser`) or `afterNextRender()` to conditionally execute browser-only code. Alternatively, lazy load browser-specific libraries.

### Q6: What are the performance implications of SSR?

**Answer:** SSR improves initial load and perceived performance but increases server load. Each request requires running Angular on the server, which needs proper caching and optimization strategies.

### Q7: How do you debug SSR issues?

**Answer:** Check server logs, use console.log on server side, verify platform detection, ensure no browser APIs are called on server, and test with `ng serve --ssr`.

### Q8: Can you use localStorage with SSR?

**Answer:** No, localStorage doesn't exist on the server. Wrap localStorage calls in platform detection or provide a server-safe implementation using dependency injection.

## Key Takeaways

1. **SSR improves initial load performance and SEO** by rendering on the server
2. **Angular Universal uses platform-server** to run Angular in Node.js
3. **TransferState prevents duplicate HTTP requests** during hydration
4. **Always check platform** before using browser APIs
5. **Use afterNextRender** for guaranteed browser-only code
6. **Enable HTTP transfer cache** for automatic request caching
7. **Set proper meta tags** for SEO and social media
8. **Clean up subscriptions** to avoid server memory leaks
9. **Pre-render static pages** for maximum performance
10. **Use proper caching headers** for static assets and API responses

## Resources

### Official Documentation

- [Angular SSR Guide](https://angular.dev/guide/ssr)
- [Angular Universal](https://github.com/angular/universal)
- [Platform Server API](https://angular.dev/api/platform-server)

### Articles

- "Angular Universal Deep Dive" - Angular Blog
- "SSR Best Practices" - Web.dev
- "Optimizing Angular Universal" - Angular University

### Video Tutorials

- "Server-Side Rendering with Angular" - ng-conf
- "Angular Universal Masterclass" - Angular Connect
- "SSR Performance Optimization" - Google Chrome Developers

### Tools

- Angular CLI - Built-in SSR support
- Angular DevTools - Hydration debugging
- Lighthouse - SSR performance auditing
