# Hydration Implementation

## Table of Contents
- [Introduction](#introduction)
- [Server-Side Rendering Fundamentals](#server-side-rendering-fundamentals)
- [Non-Destructive Hydration](#non-destructive-hydration)
- [TransferState API](#transferstate-api)
- [Serialization and Deserialization](#serialization-and-deserialization)
- [Hydration Markers](#hydration-markers)
- [Event Replay](#event-replay)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Hydration is the process of attaching Angular's client-side application logic to server-rendered HTML. Angular's hydration implementation has evolved from destructive (destroying and recreating DOM) to non-destructive (reusing existing DOM), providing significant performance improvements and better user experience.

Key concepts:
- **Server-Side Rendering (SSR)**: Rendering Angular components on the server to HTML
- **Hydration**: Attaching event listeners and application state to server-rendered DOM
- **TransferState**: Sharing data between server and client to avoid duplicate requests
- **Serialization**: Converting application state to transferable format
- **Non-destructive**: Reusing server-rendered DOM instead of replacing it

## Server-Side Rendering Fundamentals

### SSR Architecture

Angular Universal provides SSR capabilities:

```typescript
// server.ts - Express server setup
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

  // All regular routes use the Angular engine
  server.get('*', (req, res, next) => {
    const { protocol, originalUrl, baseUrl, headers } = req;

    commonEngine
      .render({
        bootstrap,
        documentFilePath: indexHtml,
        url: `${protocol}://${headers.host}${originalUrl}`,
        publicPath: browserDistFolder,
        providers: [
          { provide: APP_BASE_HREF, useValue: baseUrl }
        ],
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

### Rendering Process

How Angular renders on the server:

```typescript
// Simplified server rendering process
async function renderApplication(
  bootstrap: () => Promise<ApplicationRef>,
  options: RenderOptions
): Promise<string> {
  // 1. Create platform
  const platform = await platformServer([
    {
      provide: INITIAL_CONFIG,
      useValue: { document: options.document, url: options.url }
    }
  ]);

  // 2. Bootstrap application
  const appRef = await bootstrap();

  // 3. Wait for application to stabilize
  await appRef.isStable.pipe(
    filter(stable => stable),
    first()
  ).toPromise();

  // 4. Serialize application state
  const transferState = appRef.injector.get(TransferState);
  const stateScript = transferState.toJson();

  // 5. Render to HTML
  const document = appRef.injector.get(DOCUMENT);
  let html = document.documentElement.outerHTML;

  // 6. Inject state script
  html = html.replace(
    '</body>',
    `<script id="transfer-state" type="application/json">${stateScript}</script></body>`
  );

  // 7. Cleanup
  appRef.destroy();
  platform.destroy();

  return html;
}
```

### Server-Side Component Rendering

Components render differently on server:

```typescript
@Component({
  selector: 'app-user-profile',
  template: `
    <div class="profile">
      <h2>{{ user?.name }}</h2>
      <p>{{ user?.email }}</p>
      <button (click)="edit()" [disabled]="loading">
        Edit Profile
      </button>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  user: User | null = null;
  loading = true;

  constructor(
    private userService: UserService,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  ngOnInit() {
    // This runs on both server and client
    this.loadUser();
  }

  private async loadUser() {
    if (isPlatformServer(this.platformId)) {
      // Server-specific logic
      console.log('Running on server');
      this.user = await this.userService.getUser().toPromise();
      this.loading = false;
    } else {
      // Client-specific logic
      console.log('Running on browser');
      this.userService.getUser().subscribe(user => {
        this.user = user;
        this.loading = false;
      });
    }
  }

  edit() {
    // This only runs on client (no event listeners on server)
    console.log('Edit clicked');
  }
}
```

## Non-Destructive Hydration

### Legacy vs Non-Destructive Hydration

Understanding the evolution:

```typescript
// LEGACY APPROACH (before Angular 16)
// Angular would:
// 1. Server renders HTML
// 2. Client loads and destroys entire DOM
// 3. Client re-renders from scratch
// 4. Flicker and layout shift

// Example of destructive hydration
@Component({
  selector: 'app-root',
  template: `
    <div class="content">
      <h1>{{ title }}</h1>
      <app-user-list [users]="users"></app-user-list>
    </div>
  `
})
export class AppComponent {
  title = 'My App';
  users = [/* ... */];
}

// On client bootstrap (legacy):
// 1. Angular creates new component instance
// 2. Destroys server-rendered <div class="content">...</div>
// 3. Creates new DOM elements
// 4. Inserts new DOM (VISIBLE FLICKER)
```

```typescript
// NON-DESTRUCTIVE APPROACH (Angular 16+)
// Angular:
// 1. Server renders HTML with hydration markers
// 2. Client loads and PRESERVES existing DOM
// 3. Client attaches to existing nodes
// 4. No flicker or layout shift

// Enabling non-destructive hydration
import { provideClientHydration } from '@angular/platform-browser';

// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration()
  ]
});

// Server output includes markers:
// <div class="content" ngh="0">
//   <h1 ngh="1">My App</h1>
//   <app-user-list ngh="2">...</app-user-list>
// </div>

// Client hydration process:
// 1. Find elements by hydration markers
// 2. Attach component instances to existing nodes
// 3. Set up event listeners
// 4. Initialize Angular runtime
// 5. Start change detection
```

### Hydration Markers and DOM Matching

How Angular matches server and client DOM:

```typescript
// Hydration marker system
interface HydrationInfo {
  ngh: string;           // Node hydration ID
  'ngh-c': string;       // Component hydration data
  'ngh-t': string;       // Text node hydration data
}

// Server-side: Adding markers during rendering
function addHydrationMarkers(tNode: TNode, lView: LView): void {
  const renderer = lView[RENDERER];
  const element = unwrapRNode(lView[tNode.index]);

  if (isComponentHost(tNode)) {
    // Add component hydration marker
    const ngh = generateHydrationId(tNode);
    renderer.setAttribute(element, 'ngh', ngh);

    // Store component state for hydration
    const componentRef = lView[tNode.index];
    const serializedState = serializeComponentState(componentRef);
    if (serializedState) {
      renderer.setAttribute(element, 'ngh-c', serializedState);
    }
  }

  // Add text node markers
  if (tNode.type === TNodeType.Element) {
    const textNodes = collectTextNodes(tNode, lView);
    textNodes.forEach((textNode, index) => {
      const comment = renderer.createComment(`ngh-t${index}`);
      renderer.insertBefore(
        element,
        comment,
        textNode
      );
    });
  }
}

// Client-side: Matching nodes during hydration
function hydrateComponentView(
  lView: LView,
  tView: TView,
  tNode: TNode
): void {
  const renderer = lView[RENDERER];
  const element = unwrapRNode(lView[tNode.index]);

  // Find hydration marker
  const ngh = renderer.getAttribute(element, 'ngh');
  
  if (!ngh) {
    // No marker found - fall back to client rendering
    console.warn('Hydration marker missing, re-rendering component');
    renderComponentFromScratch(lView, tView, tNode);
    return;
  }

  // Verify DOM structure matches expectations
  if (!validateDOMStructure(tNode, element)) {
    console.warn('DOM structure mismatch, re-rendering');
    renderComponentFromScratch(lView, tView, tNode);
    return;
  }

  // Attach component to existing DOM
  attachComponentToDOM(lView, tNode, element);

  // Clean up hydration markers
  renderer.removeAttribute(element, 'ngh');
  renderer.removeAttribute(element, 'ngh-c');
}
```

### Hydration Configuration

Configuring hydration behavior:

```typescript
// main.ts - Client configuration
import {
  bootstrapApplication,
  provideClientHydration,
  withEventReplay,
  withIncrementalHydration,
  withNoHttpTransferCache
} from '@angular/platform-browser';

bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(
      withEventReplay(),              // Replay events during hydration
      withIncrementalHydration(),     // Hydrate incrementally (experimental)
      withNoHttpTransferCache()       // Disable HTTP cache transfer
    )
  ]
});

// main.server.ts - Server configuration
import { bootstrapApplication } from '@angular/platform-browser';
import { provideServerRendering } from '@angular/platform-server';

const config = {
  providers: [
    provideServerRendering()
  ]
};

export default () => bootstrapApplication(AppComponent, config);
```

### Skip Hydration Directive

Opting out of hydration for specific components:

```typescript
import { Component, NgZone } from '@angular/core';

@Component({
  selector: 'app-canvas-widget',
  template: `
    <canvas #canvas ngSkipHydration></canvas>
  `
})
export class CanvasWidgetComponent implements AfterViewInit {
  @ViewChild('canvas') canvas!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit() {
    // Canvas needs to be fully client-rendered
    // Server can't render canvas content accurately
    this.initializeCanvas();
  }

  private initializeCanvas() {
    const ctx = this.canvas.nativeElement.getContext('2d');
    // Draw complex canvas content
  }
}

// Components that should skip hydration:
// - Third-party widgets (Google Maps, charts)
// - Canvas/WebGL content
// - Components with complex DOM manipulation
// - Browser-only APIs (IntersectionObserver, etc.)

@Component({
  selector: 'app-third-party-map',
  template: `
    <div ngSkipHydration>
      <div #mapContainer class="map"></div>
    </div>
  `
})
export class ThirdPartyMapComponent implements AfterViewInit {
  @ViewChild('mapContainer') mapContainer!: ElementRef;

  ngAfterViewInit() {
    if (isPlatformBrowser(this.platformId)) {
      // Initialize Google Maps
      new google.maps.Map(this.mapContainer.nativeElement, {
        center: { lat: 0, lng: 0 },
        zoom: 8
      });
    }
  }

  constructor(@Inject(PLATFORM_ID) private platformId: Object) {}
}
```

## TransferState API

### Basic Usage

Sharing data between server and client:

```typescript
import { TransferState, makeStateKey } from '@angular/platform-browser';

// Define state keys
const USER_KEY = makeStateKey<User>('user');
const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Component({
  selector: 'app-product-list',
  template: `
    <div *ngFor="let product of products">
      {{ product.name }} - {{ product.price | currency }}
    </div>
  `
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];

  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  ngOnInit() {
    // Check if data exists in transfer state
    const cachedProducts = this.transferState.get(PRODUCTS_KEY, null);

    if (cachedProducts) {
      // Use cached data from server
      this.products = cachedProducts;
      console.log('Using transferred state');
      
      // Remove from transfer state to free memory
      this.transferState.remove(PRODUCTS_KEY);
    } else {
      // Fetch data
      this.http.get<Product[]>('/api/products').subscribe(products => {
        this.products = products;

        // Store in transfer state if on server
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(PRODUCTS_KEY, products);
        }
      });
    }
  }
}
```

### HTTP Transfer Cache Interceptor

Automatic HTTP request caching:

```typescript
// Enabled by default with provideClientHydration()
// Automatically caches HTTP requests made during SSR

import { provideHttpClient, withFetch } from '@angular/common/http';
import { provideClientHydration } from '@angular/platform-browser';

// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(),
    provideHttpClient(withFetch())
  ]
});

// Now HTTP requests are automatically cached:
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  user: User | null = null;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    // This request is made on server
    // Response is cached and transferred to client
    // Client reuses cached response instead of making new request
    this.http.get<User>('/api/user/123').subscribe(user => {
      this.user = user;
    });
  }
}

// How it works internally:
class HttpTransferCacheInterceptor implements HttpInterceptor {
  constructor(
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Only cache GET requests
    if (req.method !== 'GET') {
      return next.handle(req);
    }

    const storeKey = makeStateKey<HttpResponse<any>>(req.url);

    if (isPlatformServer(this.platformId)) {
      // Server: Make request and cache response
      return next.handle(req).pipe(
        tap(event => {
          if (event instanceof HttpResponse) {
            this.transferState.set(storeKey, event);
          }
        })
      );
    } else {
      // Client: Check cache first
      const cachedResponse = this.transferState.get(storeKey, null);
      
      if (cachedResponse) {
        // Return cached response
        this.transferState.remove(storeKey);
        return of(cachedResponse);
      }

      // Make actual request if not in cache
      return next.handle(req);
    }
  }
}
```

### Custom State Transfer

Transferring complex state:

```typescript
// State service with transfer state
@Injectable({ providedIn: 'root' })
export class AppStateService {
  private stateKey = makeStateKey<AppState>('app-state');

  constructor(
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  // Save state on server
  saveState(state: AppState): void {
    if (isPlatformServer(this.platformId)) {
      this.transferState.set(this.stateKey, state);
    }
  }

  // Restore state on client
  restoreState(): AppState | null {
    if (isPlatformBrowser(this.platformId)) {
      const state = this.transferState.get(this.stateKey, null);
      
      if (state) {
        this.transferState.remove(this.stateKey);
        return state;
      }
    }
    
    return null;
  }
}

interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  preferences: UserPreferences;
  cachedData: Map<string, any>;
}

// Usage in component
@Component({
  selector: 'app-root',
  template: `...`
})
export class AppComponent implements OnInit {
  constructor(private appState: AppStateService) {}

  ngOnInit() {
    // Try to restore state from transfer
    const restoredState = this.appState.restoreState();
    
    if (restoredState) {
      console.log('Restored state from server:', restoredState);
      this.applyState(restoredState);
    } else {
      console.log('No transferred state, initializing fresh');
      this.initializeState();
    }
  }

  private applyState(state: AppState): void {
    // Apply restored state to application
  }

  private initializeState(): void {
    // Initialize fresh state
  }
}
```

## Serialization and Deserialization

### Serializable State

What can and cannot be serialized:

```typescript
// SERIALIZABLE
const serializableState = {
  primitives: {
    string: 'text',
    number: 42,
    boolean: true,
    null: null
  },
  arrays: [1, 2, 3],
  objects: { nested: { data: 'value' } },
  dates: new Date(), // Serialized as ISO string
};

// NOT SERIALIZABLE
const nonSerializableState = {
  functions: () => {},              // Lost during serialization
  domElements: document.body,       // Cannot serialize DOM
  symbols: Symbol('key'),           // Lost
  circularRefs: (() => {            // Causes error
    const obj: any = {};
    obj.self = obj;
    return obj;
  })(),
  classInstances: new CustomClass() // Lost methods
};

// Safe serialization helper
function safeSerialize(data: any): string {
  const seen = new WeakSet();
  
  return JSON.stringify(data, (key, value) => {
    // Handle circular references
    if (typeof value === 'object' && value !== null) {
      if (seen.has(value)) {
        return '[Circular]';
      }
      seen.add(value);
    }

    // Handle special types
    if (value instanceof Date) {
      return { __type: 'Date', value: value.toISOString() };
    }

    if (value instanceof Map) {
      return {
        __type: 'Map',
        value: Array.from(value.entries())
      };
    }

    if (value instanceof Set) {
      return {
        __type: 'Set',
        value: Array.from(value)
      };
    }

    // Skip non-serializable values
    if (typeof value === 'function' || typeof value === 'symbol') {
      return undefined;
    }

    return value;
  });
}

function safeDeserialize(json: string): any {
  return JSON.parse(json, (key, value) => {
    // Restore special types
    if (value && typeof value === 'object' && '__type' in value) {
      switch (value.__type) {
        case 'Date':
          return new Date(value.value);
        
        case 'Map':
          return new Map(value.value);
        
        case 'Set':
          return new Set(value.value);
      }
    }

    return value;
  });
}
```

### Custom Serialization

Implementing custom serializers:

```typescript
// Serialization provider
interface Serializer<T> {
  serialize(value: T): any;
  deserialize(data: any): T;
}

class DateSerializer implements Serializer<Date> {
  serialize(date: Date): string {
    return date.toISOString();
  }

  deserialize(iso: string): Date {
    return new Date(iso);
  }
}

class UserSerializer implements Serializer<User> {
  serialize(user: User): any {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
      registeredAt: new DateSerializer().serialize(user.registeredAt)
    };
  }

  deserialize(data: any): User {
    return new User(
      data.id,
      data.name,
      data.email,
      new DateSerializer().deserialize(data.registeredAt)
    );
  }
}

// State manager with serializers
@Injectable({ providedIn: 'root' })
export class SerializationService {
  private serializers = new Map<string, Serializer<any>>();

  constructor() {
    this.registerSerializer('Date', new DateSerializer());
    this.registerSerializer('User', new UserSerializer());
  }

  registerSerializer<T>(type: string, serializer: Serializer<T>): void {
    this.serializers.set(type, serializer);
  }

  serialize(value: any, type?: string): any {
    if (type && this.serializers.has(type)) {
      return this.serializers.get(type)!.serialize(value);
    }
    return value;
  }

  deserialize(data: any, type?: string): any {
    if (type && this.serializers.has(type)) {
      return this.serializers.get(type)!.deserialize(data);
    }
    return data;
  }
}
```

## Hydration Markers

### Marker Generation

How hydration markers are generated:

```typescript
// Server-side marker generation
let hydrationIdCounter = 0;

function generateHydrationId(tNode: TNode): string {
  // Generate unique ID for each node
  return `${hydrationIdCounter++}`;
}

function annotateHostElementForHydration(
  element: RElement,
  lView: LView,
  tNode: TNode
): void {
  const renderer = lView[RENDERER];
  const hydrationId = generateHydrationId(tNode);

  // Add main hydration marker
  renderer.setAttribute(element, 'ngh', hydrationId);

  // Add component metadata if component host
  if (isComponentHost(tNode)) {
    const componentMetadata = {
      id: hydrationId,
      type: getComponentName(tNode),
      inputs: serializeComponentInputs(lView, tNode),
      flags: getHydrationFlags(tNode)
    };

    renderer.setAttribute(
      element,
      'ngh-c',
      JSON.stringify(componentMetadata)
    );
  }

  // Add text node markers
  annotateTextNodesForHydration(element, lView, tNode);
}

function annotateTextNodesForHydration(
  element: RElement,
  lView: LView,
  tNode: TNode
): void {
  const renderer = lView[RENDERER];
  let textNodeIndex = 0;

  // Walk through child nodes
  walkTNodes(tNode, (childTNode) => {
    if (childTNode.type === TNodeType.Text) {
      const textRNode = unwrapRNode(lView[childTNode.index]);
      
      // Insert comment marker before text node
      const marker = renderer.createComment(`ngh-t${textNodeIndex++}`);
      renderer.insertBefore(
        element,
        marker,
        textRNode
      );
    }
  });
}
```

### Marker Cleanup

Cleaning up markers after hydration:

```typescript
// Client-side cleanup after hydration
function cleanupHydrationMarkers(lView: LView): void {
  const renderer = lView[RENDERER];
  
  // Remove all hydration attributes
  walkLView(lView, (element) => {
    if (isElement(element)) {
      renderer.removeAttribute(element, 'ngh');
      renderer.removeAttribute(element, 'ngh-c');
    }
  });

  // Remove text node markers
  const comments = collectCommentNodes(lView);
  comments.forEach(comment => {
    if (comment.textContent?.startsWith('ngh-t')) {
      renderer.removeChild(comment.parentNode, comment);
    }
  });
}

// Cleanup during hydration phase
function hydrateView(lView: LView, tView: TView): void {
  // Hydrate all nodes
  hydrateNodes(lView, tView);

  // Attach event listeners
  attachEventListeners(lView, tView);

  // Cleanup markers
  cleanupHydrationMarkers(lView);

  // Mark view as hydrated
  markViewAsHydrated(lView);
}
```

## Event Replay

### Capturing Events During Hydration

Events that occur before hydration completes:

```typescript
// Event replay system
class EventReplayManager {
  private capturedEvents: Array<{
    type: string;
    target: EventTarget;
    event: Event;
  }> = [];

  private isCapturing = true;

  constructor() {
    this.startCapturing();
  }

  private startCapturing(): void {
    // Capture events globally during hydration
    const eventTypes = ['click', 'input', 'change', 'submit'];

    eventTypes.forEach(type => {
      document.addEventListener(
        type,
        (event) => this.captureEvent(type, event),
        { capture: true }
      );
    });
  }

  private captureEvent(type: string, event: Event): void {
    if (!this.isCapturing) return;

    // Store event for replay
    this.capturedEvents.push({
      type,
      target: event.target!,
      event: this.cloneEvent(event)
    });

    // Prevent default behavior during hydration
    event.preventDefault();
  }

  private cloneEvent(event: Event): Event {
    // Clone event for replay
    const cloned = new (event.constructor as any)(event.type, event);
    return cloned;
  }

  stopCapturing(): void {
    this.isCapturing = false;
  }

  replayEvents(): void {
    // Replay captured events after hydration
    this.capturedEvents.forEach(({ type, target, event }) => {
      target.dispatchEvent(event);
    });

    this.capturedEvents = [];
  }
}

// Integration with hydration
function bootstrapApplicationWithReplay(
  rootComponent: Type<any>,
  options: ApplicationConfig
): Promise<ApplicationRef> {
  const replayManager = new EventReplayManager();

  return bootstrapApplication(rootComponent, options).then(appRef => {
    // Wait for hydration to complete
    return appRef.isStable.pipe(
      filter(stable => stable),
      first()
    ).toPromise().then(() => {
      // Stop capturing and replay events
      replayManager.stopCapturing();
      replayManager.replayEvents();
      
      return appRef;
    });
  });
}
```

### Event Contract

Declarative event binding for replay:

```typescript
// Event contract marks which events should be captured
@Component({
  selector: 'app-button',
  template: `
    <button jsaction="click:handleClick">
      Click Me
    </button>
  `
})
export class ButtonComponent {
  handleClick(): void {
    console.log('Button clicked');
  }
}

// Event contract system
class EventContract {
  private eventMap = new Map<Element, Map<string, Function>>();

  registerAction(element: Element, eventType: string, handler: Function): void {
    if (!this.eventMap.has(element)) {
      this.eventMap.set(element, new Map());
    }

    this.eventMap.get(element)!.set(eventType, handler);
  }

  dispatchAction(element: Element, eventType: string, event: Event): void {
    const handlers = this.eventMap.get(element);
    const handler = handlers?.get(eventType);

    if (handler) {
      handler.call(element, event);
    }
  }

  setupGlobalListeners(): void {
    // Set up global event delegation
    document.addEventListener('click', (event) => {
      this.handleGlobalEvent('click', event);
    }, true);
  }

  private handleGlobalEvent(type: string, event: Event): void {
    let target = event.target as Element | null;

    // Walk up DOM tree looking for action attribute
    while (target) {
      const action = target.getAttribute('jsaction');
      
      if (action) {
        const [eventType, methodName] = action.split(':');
        
        if (eventType === type) {
          this.dispatchAction(target, type, event);
          break;
        }
      }

      target = target.parentElement;
    }
  }
}
```

## Common Misconceptions

### Misconception 1: "Hydration makes the app load faster"

**Reality**: Hydration improves perceived performance and user experience, but doesn't necessarily reduce total load time.

```typescript
// Timeline comparison:

// WITHOUT SSR (Client-only):
// 1. Download HTML (empty shell) - 100ms
// 2. Download JS bundle - 500ms
// 3. Execute JS and render - 300ms
// 4. Interactive - 900ms total
// User sees: Nothing -> Full page (at 900ms)

// WITH SSR + Hydration:
// 1. Download HTML (full content) - 300ms
// 2. Download JS bundle - 500ms
// 3. Hydrate existing DOM - 200ms
// 4. Interactive - 1000ms total
// User sees: Full page (at 300ms) -> Interactive (at 1000ms)

// Benefit: User sees content 600ms earlier
// Trade-off: Slightly longer to interactive (100ms)
```

### Misconception 2: "All components must be hydrated"

**Reality**: You can selectively skip hydration for certain components.

```typescript
// Skip hydration for third-party content
@Component({
  selector: 'app-ads',
  template: `
    <div ngSkipHydration>
      <!-- Ad content loads client-side only -->
      <div class="ad-container"></div>
    </div>
  `
})
export class AdsComponent {}
```

### Misconception 3: "TransferState works automatically for all data"

**Reality**: You must explicitly use TransferState API or HTTP transfer cache.

```typescript
// WRONG - State not transferred
@Injectable()
export class DataService {
  private data = new BehaviorSubject<any[]>([]);

  loadData() {
    this.http.get('/api/data').subscribe(data => {
      this.data.next(data);
    });
  }
}

// CORRECT - Using TransferState
@Injectable()
export class DataService {
  private data = new BehaviorSubject<any[]>([]);
  private dataKey = makeStateKey<any[]>('service-data');

  constructor(
    private http: HttpClient,
    private transferState: TransferState,
    @Inject(PLATFORM_ID) private platformId: Object
  ) {}

  loadData() {
    const cached = this.transferState.get(this.dataKey, null);
    
    if (cached) {
      this.data.next(cached);
      this.transferState.remove(this.dataKey);
    } else {
      this.http.get('/api/data').subscribe(data => {
        this.data.next(data);
        
        if (isPlatformServer(this.platformId)) {
          this.transferState.set(this.dataKey, data);
        }
      });
    }
  }
}
```

## Performance Implications

### Hydration Performance

Optimizing hydration speed:

```typescript
// Measure hydration performance
@Injectable()
export class HydrationMetrics {
  constructor(@Inject(PLATFORM_ID) private platformId: Object) {
    if (isPlatformBrowser(this.platformId)) {
      this.measureHydration();
    }
  }

  private measureHydration() {
    const hydrationStart = performance.now();

    // Wait for hydration complete
    whenStable().then(() => {
      const hydrationEnd = performance.now();
      const duration = hydrationEnd - hydrationStart;

      console.log(`Hydration completed in ${duration}ms`);

      // Report to analytics
      this.reportMetric('hydration_duration', duration);
    });
  }

  private reportMetric(name: string, value: number) {
    // Send to analytics service
  }
}

// Optimization strategies:

// 1. Minimize transferred state
// - Transfer only essential data
// - Remove transferred state after use

// 2. Use incremental hydration (experimental)
bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(
      withIncrementalHydration()
    )
  ]
});

// 3. Lazy load non-critical components
@Component({
  selector: 'app-root',
  template: `
    <app-header></app-header>
    <router-outlet></router-outlet>
    
    <!-- Lazy load heavy components -->
    <app-footer *deferHydrate></app-footer>
  `
})
export class AppComponent {}

// 4. Skip hydration for interactive widgets
@Component({
  selector: 'app-chart',
  template: `<canvas ngSkipHydration #chart></canvas>`
})
export class ChartComponent {}
```

### Bundle Size Impact

SSR adds to bundle size:

```typescript
// Analyze bundle impact
// Before SSR:
// - main.js: 500 KB
// - Total: 500 KB

// After SSR:
// - main.js: 500 KB (client)
// - main.server.js: 150 KB (server)
// - server.js: 50 KB (Express)
// - Total client: 500 KB (same)
// - Total server: 700 KB (new)

// Optimization: Code splitting
@NgModule({
  imports: [
    RouterModule.forRoot([
      {
        path: 'admin',
        loadChildren: () => import('./admin/admin.module')
          .then(m => m.AdminModule)
      }
    ])
  ]
})
export class AppRoutingModule {}

// Results in:
// - main.js: 400 KB (reduced)
// - admin.chunk.js: 100 KB (lazy loaded)
// - Total initial: 400 KB (improvement!)
```

## Interview Questions

### Question 1: What is hydration in Angular and why is it important?

**Answer**: Hydration is the process of attaching Angular's client-side runtime to server-rendered HTML. It's important because:

1. **Performance**: Users see content faster (server-rendered HTML loads before JS)
2. **SEO**: Search engines can crawl fully-rendered content
3. **Accessibility**: Content is available even if JavaScript fails to load
4. **User Experience**: No flash of empty content on load

Angular 16+ introduced non-destructive hydration, which reuses server-rendered DOM instead of destroying and recreating it, eliminating flicker and improving performance.

### Question 2: How does TransferState work and when should you use it?

**Answer**: TransferState allows sharing data between server and client to avoid duplicate HTTP requests:

1. **Server**: Fetches data and stores in TransferState
2. **Serialization**: State is serialized to JSON and embedded in HTML
3. **Client**: Reads state from HTML instead of making new request
4. **Cleanup**: State is removed after use to free memory

Use TransferState for:
- Initial page data (user profile, product lists)
- Configuration data
- API responses needed on first render

Don't use for:
- Large datasets (bloats HTML)
- Sensitive data (visible in HTML source)
- Data that changes frequently

### Question 3: What's the difference between destructive and non-destructive hydration?

**Answer**:

**Destructive (legacy)**:
- Client destroys server-rendered DOM
- Recreates DOM from scratch
- Causes visible flicker
- Loses user interactions during hydration

**Non-destructive (modern)**:
- Client reuses server-rendered DOM
- Attaches to existing nodes
- No visual flicker
- Preserves DOM state

Non-destructive is enabled with `provideClientHydration()` in Angular 16+.

### Question 4: How do you handle events that occur during hydration?

**Answer**: Angular provides event replay functionality:

```typescript
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';

bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(
      withEventReplay()
    )
  ]
});
```

With event replay:
1. Events are captured during hydration
2. Default behavior is prevented
3. After hydration completes, events are replayed
4. User interactions aren't lost

Without event replay, clicks during hydration are ignored.

### Question 5: What are hydration markers and why are they needed?

**Answer**: Hydration markers are attributes added to server-rendered DOM to help client match elements:

- `ngh`: Node hydration ID
- `ngh-c`: Component metadata
- `ngh-t`: Text node markers

They're needed because:
1. **DOM matching**: Client must find corresponding elements
2. **Structure validation**: Ensure server and client DOM match
3. **Component attachment**: Link component instances to DOM nodes
4. **State restoration**: Restore component state accurately

Markers are automatically removed after hydration completes.

### Question 6: When should you use ngSkipHydration?

**Answer**: Use `ngSkipHydration` for:

1. **Third-party widgets**: Google Maps, charts libraries
2. **Canvas/WebGL**: Server can't render canvas accurately
3. **Browser-only APIs**: IntersectionObserver, Web Audio
4. **Dynamic DOM manipulation**: jQuery plugins
5. **Interactive games**: Real-time rendering

Example:
```typescript
@Component({
  template: `
    <div ngSkipHydration>
      <canvas #gameCanvas></canvas>
    </div>
  `
})
export class GameComponent {}
```

### Question 7: How do you optimize hydration performance?

**Answer**: Optimization strategies:

1. **Minimize transferred state**: Only transfer essential data
2. **Remove state after use**: Call `transferState.remove(key)`
3. **Use HTTP transfer cache**: Enabled by default with `provideClientHydration()`
4. **Lazy load heavy components**: Use router lazy loading
5. **Skip hydration for widgets**: Use `ngSkipHydration`
6. **Incremental hydration**: Enable with `withIncrementalHydration()` (experimental)
7. **Measure performance**: Track hydration duration

```typescript
provideClientHydration(
  withEventReplay(),
  withIncrementalHydration()
)
```

### Question 8: What happens if DOM structure doesn't match between server and client?

**Answer**: Angular detects mismatches and handles them:

1. **Detection**: During hydration, Angular validates DOM structure
2. **Warning**: Console warning about mismatch
3. **Fallback**: Falls back to client-side rendering for mismatched nodes
4. **Re-render**: Destroys and recreates mismatched subtree

Common causes:
- Conditional rendering differences
- Date/time formatting (timezone differences)
- Random content (IDs, keys)
- Browser-specific APIs used on server

Prevention:
- Use consistent rendering logic
- Format dates uniformly
- Use deterministic IDs
- Check platform before using browser APIs

## Key Takeaways

1. **Non-destructive hydration reuses DOM** instead of destroying and recreating, eliminating flicker
2. **TransferState shares data** between server and client to avoid duplicate requests
3. **HTTP transfer cache works automatically** for GET requests when hydration is enabled
4. **Event replay captures interactions** during hydration so they're not lost
5. **Hydration markers enable DOM matching** between server-rendered and client-expected structure
6. **ngSkipHydration opts out** specific components from hydration when needed
7. **Serialization limits** what can be transferred (primitives, arrays, objects only)
8. **Performance optimization** requires minimizing transferred state and lazy loading heavy components

## Resources

### Official Documentation
- [Angular SSR Guide](https://angular.dev/guide/ssr)
- [Hydration Guide](https://angular.dev/guide/hydration)
- [TransferState API](https://angular.dev/api/platform-browser/TransferState)

### Articles & Tutorials
- "Angular Universal Server-Side Rendering Deep Dive" by Angular Team
- "Understanding Hydration in Angular" by Netanel Basal
- "Optimizing Angular SSR Performance" by Minko Gechev

### Videos
- Angular SSR and Hydration Explained
- Non-Destructive Hydration in Angular 16
- Building Fast Server-Side Rendered Angular Apps

### Books
- "Server-Side Rendering with Angular and Angular Universal"
- "Pro Angular SSR" by Adam Freeman

### Source Code
- [Angular Universal](https://github.com/angular/universal)
- [Angular Hydration Implementation](https://github.com/angular/angular/tree/main/packages/platform-browser/src/hydration)
- [TransferState Source](https://github.com/angular/angular/blob/main/packages/platform-browser/src/browser/transfer_state.ts)
