# Angular Testing - Interview Questions

## The Idea

**In plain English:** Testing in Angular means writing extra code that automatically checks your app's code is working correctly — like a checklist that runs itself every time you make a change, catching mistakes before users do. A "test" is just a small program that calls your real code and verifies the result matches what you expect.

**Real-world analogy:** Imagine a car factory that has a quality-control inspector at the end of the assembly line. Before any car leaves, the inspector runs through a checklist: do the brakes work, do the lights turn on, does the engine start? Each check is done in a controlled environment (not on a real road) with known conditions.

- The inspector's checklist = the test suite (the collection of automated tests)
- Each individual check (do the brakes work?) = a single test case (one `it(...)` block)
- The controlled test track = the TestBed (Angular's fake environment that simulates a real browser)

---

## Overview

This guide covers Angular testing strategies, including TestBed configuration, component testing with fixture and debugElement, service testing, HTTP testing with HttpTestingController, marble testing for observables, and async testing patterns.

## Core Concept Questions

### 1. How do you configure TestBed for component testing? What are the common configuration options?

**Expected Answer:**

TestBed is Angular's primary testing utility that creates an Angular testing module where you can configure dependencies, providers, and imports needed for testing.

**Basic TestBed Configuration:**

```typescript
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { Component, DebugElement } from '@angular/core';
import { By } from '@angular/platform-browser';

describe('UserComponent', () => {
  let component: UserComponent;
  let fixture: ComponentFixture<UserComponent>;
  let debugElement: DebugElement;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        UserComponent, // Standalone component
        CommonModule,
        HttpClientTestingModule
      ],
      providers: [
        UserService,
        { provide: AuthService, useClass: MockAuthService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(UserComponent);
    component = fixture.componentInstance;
    debugElement = fixture.debugElement;
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });
});

// Advanced configuration with custom providers
describe('ProductComponent with dependencies', () => {
  let component: ProductComponent;
  let fixture: ComponentFixture<ProductComponent>;
  let productService: jasmine.SpyObj<ProductService>;
  let router: jasmine.SpyObj<Router>;

  beforeEach(async () => {
    // Create spy objects
    const productServiceSpy = jasmine.createSpyObj('ProductService', 
      ['getProducts', 'getProduct', 'addProduct']);
    const routerSpy = jasmine.createSpyObj('Router', ['navigate']);

    await TestBed.configureTestingModule({
      imports: [
        ProductComponent,
        ReactiveFormsModule,
        RouterTestingModule
      ],
      providers: [
        { provide: ProductService, useValue: productServiceSpy },
        { provide: Router, useValue: routerSpy },
        { provide: APP_CONFIG, useValue: { apiUrl: 'http://test.api' } }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(ProductComponent);
    component = fixture.componentInstance;
    productService = TestBed.inject(ProductService) as jasmine.SpyObj<ProductService>;
    router = TestBed.inject(Router) as jasmine.SpyObj<Router>;
  });

  it('should load products on init', () => {
    const mockProducts = [
      { id: 1, name: 'Product 1' },
      { id: 2, name: 'Product 2' }
    ];
    productService.getProducts.and.returnValue(of(mockProducts));

    component.ngOnInit();

    expect(productService.getProducts).toHaveBeenCalled();
    expect(component.products()).toEqual(mockProducts);
  });
});

// Configuration with module dependencies
describe('DashboardComponent with modules', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        DashboardComponent,
        BrowserAnimationsModule, // or NoopAnimationsModule
        MaterialModule,
        ChartsModule
      ],
      providers: [
        provideMockStore({ initialState }),
        { provide: MatDialog, useClass: MockMatDialog }
      ]
    })
    .overrideComponent(DashboardComponent, {
      set: {
        providers: [
          { provide: DashboardService, useClass: MockDashboardService }
        ]
      }
    })
    .compileComponents();
  });
});

// Configuring test module with schemas
describe('Component with unknown elements', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [MyComponent],
      schemas: [
        NO_ERRORS_SCHEMA, // Ignore unknown elements and attributes
        // or CUSTOM_ELEMENTS_SCHEMA for custom elements
      ]
    }).compileComponents();
  });
});
```

**Common Configuration Options:**

```typescript
// 1. Import standalone components and modules
imports: [
  ComponentUnderTest,
  CommonModule,
  FormsModule,
  HttpClientTestingModule
]

// 2. Provide services and dependencies
providers: [
  ServiceClass,
  { provide: AbstractService, useClass: ConcreteService },
  { provide: TOKEN, useValue: value },
  { provide: FactoryService, useFactory: factoryFn, deps: [Dependency] }
]

// 3. Override providers
.overrideComponent(Component, {
  set: { providers: [...] }
})

// 4. Declare schemas for unknown elements
schemas: [NO_ERRORS_SCHEMA, CUSTOM_ELEMENTS_SCHEMA]

// 5. Configure animations
imports: [NoopAnimationsModule] // Disable animations for faster tests

// 6. Compile components
.compileComponents() // Required for async component compilation
```

**Reusable Test Configuration:**

```typescript
// test-helpers.ts
export function configureTestBed(config: {
  component: any;
  imports?: any[];
  providers?: any[];
  declarations?: any[];
}) {
  return TestBed.configureTestingModule({
    imports: [
      config.component,
      CommonModule,
      NoopAnimationsModule,
      ...(config.imports || [])
    ],
    providers: [
      ...(config.providers || [])
    ],
    declarations: config.declarations || []
  }).compileComponents();
}

// Usage
describe('MyComponent', () => {
  beforeEach(async () => {
    await configureTestBed({
      component: MyComponent,
      imports: [ReactiveFormsModule],
      providers: [{ provide: MyService, useClass: MockMyService }]
    });
  });
});
```

**Follow-up Questions:**
1. What's the difference between imports and declarations in TestBed?
2. When do you need to call compileComponents()?
3. How do you override providers for specific tests?

**What Interviewers Look For:**
- Understanding of TestBed configuration
- Knowledge of when to use mocks vs real implementations
- Proper setup of dependencies
- Understanding of async compilation

### 2. Explain the difference between fixture, debugElement, and nativeElement. How do you query DOM elements in tests?

**Expected Answer:**

Fixture is a wrapper around the component providing access to the component instance and DOM. DebugElement is an Angular abstraction over native DOM elements. NativeElement is the actual DOM element.

**Comprehensive Examples:**

```typescript
describe('DOM Querying and Manipulation', () => {
  let fixture: ComponentFixture<UserCardComponent>;
  let component: UserCardComponent;
  let debugElement: DebugElement;
  let nativeElement: HTMLElement;

  beforeEach(() => {
    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
    debugElement = fixture.debugElement;
    nativeElement = fixture.nativeElement;
  });

  it('should demonstrate fixture usage', () => {
    // Fixture provides component instance
    expect(fixture.componentInstance).toBe(component);
    
    // Trigger change detection
    fixture.detectChanges();
    
    // Check if component is stable
    fixture.whenStable().then(() => {
      // Assertions after async operations complete
    });
    
    // Destroy component
    fixture.destroy();
  });

  it('should demonstrate debugElement querying', () => {
    // Query by CSS selector
    const titleDebug = debugElement.query(By.css('h1'));
    const titleElement = titleDebug.nativeElement as HTMLElement;
    expect(titleElement.textContent).toContain('User Card');

    // Query all matching elements
    const buttons = debugElement.queryAll(By.css('button'));
    expect(buttons.length).toBe(3);

    // Query by directive
    const formDebug = debugElement.query(By.directive(FormDirective));
    
    // Query by predicate
    const inputDebug = debugElement.query((de: DebugElement) => {
      return de.nativeElement.tagName === 'INPUT';
    });

    // Access component instance through debugElement
    const childComponent = debugElement.query(By.css('app-child')).componentInstance;
  });

  it('should demonstrate nativeElement usage', () => {
    fixture.detectChanges();

    // Direct DOM access (not recommended for most cases)
    const title = nativeElement.querySelector('h1') as HTMLElement;
    expect(title.textContent).toContain('User Card');

    // Get all buttons
    const buttons = nativeElement.querySelectorAll('button');
    expect(buttons.length).toBe(3);
  });
});

// Complete example with various query strategies
@Component({
  selector: 'app-search',
  standalone: true,
  template: `
    <div class="search-container">
      <input 
        #searchInput
        type="text" 
        [(ngModel)]="searchTerm"
        placeholder="Search..."
        class="search-input"
        data-testid="search-input">
      
      <button 
        (click)="search()"
        [disabled]="!searchTerm"
        class="search-button"
        data-testid="search-button">
        Search
      </button>

      <div class="results" *ngIf="results.length > 0">
        <div 
          *ngFor="let result of results; let i = index"
          class="result-item"
          [attr.data-testid]="'result-' + i">
          {{ result.name }}
        </div>
      </div>

      <app-loading *ngIf="loading"></app-loading>
    </div>
  `
})
class SearchComponent {
  searchTerm = '';
  results: any[] = [];
  loading = false;

  search() {
    this.loading = true;
    // Search logic
  }
}

describe('SearchComponent DOM Testing', () => {
  let fixture: ComponentFixture<SearchComponent>;
  let component: SearchComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [SearchComponent, FormsModule]
    }).compileComponents();

    fixture = TestBed.createComponent(SearchComponent);
    component = fixture.componentInstance;
  });

  it('should query input by CSS class', () => {
    const input = fixture.debugElement.query(By.css('.search-input'));
    expect(input).toBeTruthy();
    expect(input.nativeElement.placeholder).toBe('Search...');
  });

  it('should query button by data-testid', () => {
    const button = fixture.debugElement.query(
      By.css('[data-testid="search-button"]')
    );
    expect(button).toBeTruthy();
  });

  it('should query all result items', () => {
    component.results = [
      { name: 'Result 1' },
      { name: 'Result 2' },
      { name: 'Result 3' }
    ];
    fixture.detectChanges();

    const results = fixture.debugElement.queryAll(By.css('.result-item'));
    expect(results.length).toBe(3);
    expect(results[0].nativeElement.textContent.trim()).toBe('Result 1');
  });

  it('should query child component', () => {
    component.loading = true;
    fixture.detectChanges();

    const loadingComponent = fixture.debugElement.query(
      By.directive(LoadingComponent)
    );
    expect(loadingComponent).toBeTruthy();
    expect(loadingComponent.componentInstance).toBeInstanceOf(LoadingComponent);
  });

  it('should simulate user input', () => {
    const input = fixture.debugElement.query(By.css('.search-input'));
    const inputElement = input.nativeElement as HTMLInputElement;

    // Set value
    inputElement.value = 'test query';
    
    // Dispatch input event
    inputElement.dispatchEvent(new Event('input'));
    fixture.detectChanges();

    expect(component.searchTerm).toBe('test query');
  });

  it('should simulate button click', () => {
    spyOn(component, 'search');
    
    component.searchTerm = 'test';
    fixture.detectChanges();

    const button = fixture.debugElement.query(By.css('.search-button'));
    button.nativeElement.click();

    expect(component.search).toHaveBeenCalled();
  });

  it('should check button disabled state', () => {
    fixture.detectChanges();
    
    const button = fixture.debugElement.query(By.css('.search-button'));
    expect(button.nativeElement.disabled).toBe(true);

    component.searchTerm = 'test';
    fixture.detectChanges();
    
    expect(button.nativeElement.disabled).toBe(false);
  });
});

// Advanced: Custom query helpers
class ComponentTestHelper<T> {
  constructor(private fixture: ComponentFixture<T>) {}

  query(selector: string): DebugElement {
    return this.fixture.debugElement.query(By.css(selector));
  }

  queryAll(selector: string): DebugElement[] {
    return this.fixture.debugElement.queryAll(By.css(selector));
  }

  queryByTestId(testId: string): DebugElement {
    return this.fixture.debugElement.query(
      By.css(`[data-testid="${testId}"]`)
    );
  }

  getText(selector: string): string {
    return this.query(selector)?.nativeElement.textContent.trim() || '';
  }

  click(selector: string): void {
    this.query(selector)?.nativeElement.click();
  }

  setInputValue(selector: string, value: string): void {
    const input = this.query(selector)?.nativeElement as HTMLInputElement;
    if (input) {
      input.value = value;
      input.dispatchEvent(new Event('input'));
      this.fixture.detectChanges();
    }
  }
}

// Usage
describe('SearchComponent with Helper', () => {
  let helper: ComponentTestHelper<SearchComponent>;

  beforeEach(() => {
    const fixture = TestBed.createComponent(SearchComponent);
    helper = new ComponentTestHelper(fixture);
  });

  it('should search when button clicked', () => {
    helper.setInputValue('.search-input', 'test query');
    helper.click('.search-button');
    
    expect(helper.getText('.results')).toContain('test query');
  });
});
```

**Key Differences:**

| Feature | fixture | debugElement | nativeElement |
|---------|---------|--------------|---------------|
| **Type** | ComponentFixture<T> | DebugElement | HTMLElement |
| **Purpose** | Component wrapper | Angular abstraction | Actual DOM |
| **Change Detection** | Controls it | No direct control | No direct control |
| **Querying** | Via debugElement | By.css, By.directive | querySelector |
| **Platform** | Browser/Server | Browser/Server | Browser-specific |
| **Recommended** | Always use | Preferred for tests | Use sparingly |

**Follow-up Questions:**
1. When would you use nativeElement instead of debugElement?
2. How do you test components with ViewChild queries?
3. What's the benefit of using By.css over querySelector?

**What Interviewers Look For:**
- Understanding of test utilities
- Proper use of debugElement for platform independence
- Knowledge of DOM querying strategies
- Understanding of change detection in tests

### 3. How do you test services with dependencies? Demonstrate with HttpClient testing.

**Expected Answer:**

Test services by providing mock dependencies through TestBed or creating spy objects with jasmine.createSpyObj. Use HttpTestingController for HTTP testing.

**Service Testing Examples:**

```typescript
// Simple service without dependencies
@Injectable({ providedIn: 'root' })
export class CalculatorService {
  add(a: number, b: number): number {
    return a + b;
  }

  divide(a: number, b: number): number {
    if (b === 0) throw new Error('Division by zero');
    return a / b;
  }
}

describe('CalculatorService', () => {
  let service: CalculatorService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(CalculatorService);
  });

  it('should add two numbers', () => {
    expect(service.add(2, 3)).toBe(5);
  });

  it('should throw error when dividing by zero', () => {
    expect(() => service.divide(10, 0)).toThrowError('Division by zero');
  });
});

// Service with dependencies
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(
    private http: HttpClient,
    private auth: AuthService
  ) {}

  getCurrentUser(): Observable<User> {
    const token = this.auth.getToken();
    return this.http.get<User>('/api/user', {
      headers: { Authorization: `Bearer ${token}` }
    });
  }

  updateUser(user: User): Observable<User> {
    return this.http.put<User>(`/api/users/${user.id}`, user);
  }
}

describe('UserService with mocks', () => {
  let service: UserService;
  let httpMock: jasmine.SpyObj<HttpClient>;
  let authMock: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    httpMock = jasmine.createSpyObj('HttpClient', ['get', 'put', 'post', 'delete']);
    authMock = jasmine.createSpyObj('AuthService', ['getToken', 'isAuthenticated']);

    TestBed.configureTestingModule({
      providers: [
        UserService,
        { provide: HttpClient, useValue: httpMock },
        { provide: AuthService, useValue: authMock }
      ]
    });

    service = TestBed.inject(UserService);
  });

  it('should get current user with auth token', () => {
    const mockUser = { id: 1, name: 'John' };
    authMock.getToken.and.returnValue('fake-token');
    httpMock.get.and.returnValue(of(mockUser));

    service.getCurrentUser().subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    expect(authMock.getToken).toHaveBeenCalled();
    expect(httpMock.get).toHaveBeenCalledWith('/api/user', {
      headers: { Authorization: 'Bearer fake-token' }
    });
  });
});

// HttpClient testing with HttpTestingController
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';

@Injectable({ providedIn: 'root' })
export class ProductService {
  private apiUrl = '/api/products';

  constructor(private http: HttpClient) {}

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>(this.apiUrl);
  }

  getProduct(id: number): Observable<Product> {
    return this.http.get<Product>(`${this.apiUrl}/${id}`);
  }

  createProduct(product: Product): Observable<Product> {
    return this.http.post<Product>(this.apiUrl, product);
  }

  updateProduct(product: Product): Observable<Product> {
    return this.http.put<Product>(`${this.apiUrl}/${product.id}`, product);
  }

  deleteProduct(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }

  searchProducts(query: string): Observable<Product[]> {
    return this.http.get<Product[]>(`${this.apiUrl}/search`, {
      params: { q: query }
    });
  }
}

describe('ProductService with HttpTestingController', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ProductService]
    });

    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Verify no outstanding requests
    httpMock.verify();
  });

  it('should get all products', () => {
    const mockProducts: Product[] = [
      { id: 1, name: 'Product 1', price: 100 },
      { id: 2, name: 'Product 2', price: 200 }
    ];

    service.getProducts().subscribe(products => {
      expect(products.length).toBe(2);
      expect(products).toEqual(mockProducts);
    });

    // Expect one request to the API
    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('GET');

    // Respond with mock data
    req.flush(mockProducts);
  });

  it('should get single product', () => {
    const mockProduct: Product = { id: 1, name: 'Product 1', price: 100 };

    service.getProduct(1).subscribe(product => {
      expect(product).toEqual(mockProduct);
    });

    const req = httpMock.expectOne('/api/products/1');
    expect(req.request.method).toBe('GET');
    req.flush(mockProduct);
  });

  it('should create product', () => {
    const newProduct: Product = { id: 0, name: 'New Product', price: 150 };
    const createdProduct: Product = { ...newProduct, id: 3 };

    service.createProduct(newProduct).subscribe(product => {
      expect(product.id).toBe(3);
      expect(product.name).toBe('New Product');
    });

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newProduct);
    req.flush(createdProduct);
  });

  it('should update product', () => {
    const updatedProduct: Product = { id: 1, name: 'Updated Product', price: 150 };

    service.updateProduct(updatedProduct).subscribe(product => {
      expect(product).toEqual(updatedProduct);
    });

    const req = httpMock.expectOne('/api/products/1');
    expect(req.request.method).toBe('PUT');
    expect(req.request.body).toEqual(updatedProduct);
    req.flush(updatedProduct);
  });

  it('should delete product', () => {
    service.deleteProduct(1).subscribe();

    const req = httpMock.expectOne('/api/products/1');
    expect(req.request.method).toBe('DELETE');
    req.flush(null);
  });

  it('should handle HTTP error', () => {
    service.getProduct(1).subscribe({
      next: () => fail('should have failed'),
      error: (error) => {
        expect(error.status).toBe(404);
        expect(error.error).toBe('Not found');
      }
    });

    const req = httpMock.expectOne('/api/products/1');
    req.flush('Not found', { status: 404, statusText: 'Not Found' });
  });

  it('should search with query parameters', () => {
    const mockResults: Product[] = [
      { id: 1, name: 'Search Result', price: 100 }
    ];

    service.searchProducts('test').subscribe(products => {
      expect(products).toEqual(mockResults);
    });

    const req = httpMock.expectOne(req => 
      req.url === '/api/products/search' && req.params.get('q') === 'test'
    );
    expect(req.request.method).toBe('GET');
    req.flush(mockResults);
  });

  it('should handle multiple simultaneous requests', () => {
    service.getProduct(1).subscribe();
    service.getProduct(2).subscribe();
    service.getProducts().subscribe();

    const requests = httpMock.match(req => req.url.includes('/api/products'));
    expect(requests.length).toBe(3);

    requests[0].flush({ id: 1, name: 'Product 1', price: 100 });
    requests[1].flush({ id: 2, name: 'Product 2', price: 200 });
    requests[2].flush([]);
  });
});

// Testing service with RxJS operators
@Injectable({ providedIn: 'root' })
export class CachedProductService {
  private cache = new Map<number, Product>();

  constructor(private http: HttpClient) {}

  getProduct(id: number): Observable<Product> {
    if (this.cache.has(id)) {
      return of(this.cache.get(id)!);
    }

    return this.http.get<Product>(`/api/products/${id}`).pipe(
      tap(product => this.cache.set(id, product)),
      catchError(error => {
        console.error('Error fetching product:', error);
        return throwError(() => error);
      })
    );
  }

  clearCache(): void {
    this.cache.clear();
  }
}

describe('CachedProductService', () => {
  let service: CachedProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [CachedProductService]
    });

    service = TestBed.inject(CachedProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should cache product after first request', () => {
    const mockProduct: Product = { id: 1, name: 'Product 1', price: 100 };

    // First request - should hit API
    service.getProduct(1).subscribe(product => {
      expect(product).toEqual(mockProduct);
    });

    const req = httpMock.expectOne('/api/products/1');
    req.flush(mockProduct);

    // Second request - should use cache
    service.getProduct(1).subscribe(product => {
      expect(product).toEqual(mockProduct);
    });

    // No new HTTP request should be made
    httpMock.expectNone('/api/products/1');
  });

  it('should clear cache', () => {
    const mockProduct: Product = { id: 1, name: 'Product 1', price: 100 };

    // First request
    service.getProduct(1).subscribe();
    httpMock.expectOne('/api/products/1').flush(mockProduct);

    // Clear cache
    service.clearCache();

    // Should make new request after cache clear
    service.getProduct(1).subscribe();
    httpMock.expectOne('/api/products/1').flush(mockProduct);
  });
});
```

**Follow-up Questions:**
1. How do you test services with multiple dependencies?
2. What's the difference between HttpClientTestingModule and mocking HttpClient?
3. How do you test error handling in services?

**What Interviewers Look For:**
- Proper use of HttpTestingController
- Understanding of when to use mocks vs TestingModule
- Knowledge of testing async operations
- Proper verification of HTTP requests

### 4. How do you test asynchronous code in Angular? Explain fakeAsync, tick, and flush.

**Expected Answer:**

Angular provides fakeAsync, tick, and flush utilities to test asynchronous code synchronously by controlling time within tests.

**Async Testing Patterns:**

```typescript
// 1. Using done callback
it('should handle async with done callback', (done) => {
  service.getData().subscribe(data => {
    expect(data).toBeDefined();
    done(); // Signal test completion
  });
});

// 2. Using async/await
it('should handle async with async/await', async () => {
  const data = await firstValueFrom(service.getData());
  expect(data).toBeDefined();
});

// 3. Using fakeAsync and tick
import { fakeAsync, tick, flush, discardPeriodicTasks } from '@angular/core/testing';

describe('Async Testing with fakeAsync', () => {
  it('should handle setTimeout with tick', fakeAsync(() => {
    let value = false;

    setTimeout(() => {
      value = true;
    }, 1000);

    expect(value).toBe(false);

    tick(1000); // Advance time by 1000ms

    expect(value).toBe(true);
  }));

  it('should handle multiple setTimeout calls', fakeAsync(() => {
    let counter = 0;

    setTimeout(() => counter++, 100);
    setTimeout(() => counter++, 200);
    setTimeout(() => counter++, 300);

    expect(counter).toBe(0);

    tick(100);
    expect(counter).toBe(1);

    tick(100);
    expect(counter).toBe(2);

    tick(100);
    expect(counter).toBe(3);
  }));

  it('should handle setInterval with discardPeriodicTasks', fakeAsync(() => {
    let counter = 0;

    const interval = setInterval(() => {
      counter++;
    }, 100);

    tick(500);
    expect(counter).toBe(5);

    clearInterval(interval);
    discardPeriodicTasks(); // Clear any pending timers
  }));

  it('should use flush to advance all pending timers', fakeAsync(() => {
    let counter = 0;

    setTimeout(() => counter++, 100);
    setTimeout(() => counter++, 500);
    setTimeout(() => counter++, 1000);

    expect(counter).toBe(0);

    flush(); // Advance time to complete all pending timers

    expect(counter).toBe(3);
  }));
});

// Testing Promises
describe('Promise Testing', () => {
  it('should test promise with fakeAsync', fakeAsync(() => {
    let value = '';

    Promise.resolve('hello').then(val => {
      value = val;
    });

    expect(value).toBe(''); // Promise not resolved yet

    tick(); // Resolve pending microtasks

    expect(value).toBe('hello');
  }));

  it('should test chained promises', fakeAsync(() => {
    let result = 0;

    Promise.resolve(10)
      .then(val => val * 2)
      .then(val => val + 5)
      .then(val => {
        result = val;
      });

    tick();

    expect(result).toBe(25);
  }));
});

// Testing Observables
describe('Observable Testing', () => {
  it('should test observable with fakeAsync', fakeAsync(() => {
    let value = '';

    of('hello').pipe(delay(1000)).subscribe(val => {
      value = val;
    });

    expect(value).toBe('');

    tick(1000);

    expect(value).toBe('hello');
  }));

  it('should test observable with multiple emissions', fakeAsync(() => {
    const values: number[] = [];

    interval(100).pipe(
      take(5)
    ).subscribe(val => {
      values.push(val);
    });

    tick(500);

    expect(values).toEqual([0, 1, 2, 3, 4]);
  }));

  it('should test debounced observable', fakeAsync(() => {
    const subject = new Subject<string>();
    const values: string[] = [];

    subject.pipe(
      debounceTime(300)
    ).subscribe(val => {
      values.push(val);
    });

    subject.next('a');
    tick(100);
    subject.next('b');
    tick(100);
    subject.next('c');
    tick(300);

    // Only 'c' emitted after debounce
    expect(values).toEqual(['c']);
  }));
});

// Testing Component Async Operations
@Component({
  selector: 'app-async-data',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="!loading && data">{{ data.name }}</div>
    <div *ngIf="error">Error: {{ error }}</div>
  `
})
export class AsyncDataComponent implements OnInit {
  data: any;
  loading = false;
  error = '';

  constructor(private service: DataService) {}

  ngOnInit() {
    this.loadData();
  }

  loadData() {
    this.loading = true;
    this.service.getData().pipe(
      delay(1000)
    ).subscribe({
      next: (data) => {
        this.data = data;
        this.loading = false;
      },
      error: (error) => {
        this.error = error.message;
        this.loading = false;
      }
    });
  }
}

describe('AsyncDataComponent', () => {
  let component: AsyncDataComponent;
  let fixture: ComponentFixture<AsyncDataComponent>;
  let service: jasmine.SpyObj<DataService>;

  beforeEach(() => {
    service = jasmine.createSpyObj('DataService', ['getData']);

    TestBed.configureTestingModule({
      imports: [AsyncDataComponent],
      providers: [
        { provide: DataService, useValue: service }
      ]
    });

    fixture = TestBed.createComponent(AsyncDataComponent);
    component = fixture.componentInstance;
  });

  it('should show loading state', fakeAsync(() => {
    service.getData.and.returnValue(of({ name: 'Test' }).pipe(delay(1000)));

    fixture.detectChanges(); // ngOnInit

    expect(component.loading).toBe(true);
    
    const compiled = fixture.nativeElement;
    expect(compiled.textContent).toContain('Loading...');

    tick(1000);
    fixture.detectChanges();

    expect(component.loading).toBe(false);
    expect(compiled.textContent).toContain('Test');
  }));

  it('should handle errors', fakeAsync(() => {
    service.getData.and.returnValue(
      throwError(() => new Error('Failed to load')).pipe(delay(1000))
    );

    fixture.detectChanges();

    tick(1000);
    fixture.detectChanges();

    expect(component.error).toBe('Failed to load');
    expect(fixture.nativeElement.textContent).toContain('Error: Failed to load');
  }));

  it('should use fixture.whenStable for async operations', async () => {
    service.getData.and.returnValue(of({ name: 'Test' }));

    fixture.detectChanges();

    await fixture.whenStable();
    fixture.detectChanges();

    expect(component.data.name).toBe('Test');
  });
});

// Testing with real timers (waitForAsync)
import { waitForAsync } from '@angular/core/testing';

describe('Testing with waitForAsync', () => {
  it('should handle real async operations', waitForAsync(() => {
    let value = '';

    setTimeout(() => {
      value = 'done';
    }, 100);

    // Test waits for async zone to stabilize
    fixture.whenStable().then(() => {
      expect(value).toBe('done');
    });
  }));
});
```

**Follow-up Questions:**
1. When should you use fakeAsync vs waitForAsync?
2. What happens if you forget to call discardPeriodicTasks?
3. Can you use fakeAsync with XHR requests?

**What Interviewers Look For:**
- Understanding of async testing utilities
- Proper use of tick, flush, and discardPeriodicTasks
- Knowledge of when to use different async testing strategies
- Understanding of Angular's zone.js integration

### 5. How do you test Observables with marble testing? Provide examples.

**Expected Answer:**

Marble testing uses ASCII diagrams to represent observable streams over time, making it easy to test complex observable scenarios with jasmine-marbles or RxJS TestScheduler.

**Marble Testing Examples:**

```typescript
import { TestScheduler } from 'rxjs/testing';
import { delay, map, catchError, debounceTime, switchMap } from 'rxjs/operators';

describe('Marble Testing', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should test simple observable', () => {
    scheduler.run(({ cold, expectObservable }) => {
      // '-' = 1 frame of time passing
      // 'a' = emission of value 'a'
      // '|' = completion
      const source = cold('--a--b--c--|');
      const expected =    '--a--b--c--|';

      expectObservable(source).toBe(expected);
    });
  });

  it('should test with values', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('--a--b--c--|', {
        a: 1,
        b: 2,
        c: 3
      });
      const expected =    '--a--b--c--|';

      expectObservable(source).toBe(expected, {
        a: 1,
        b: 2,
        c: 3
      });
    });
  });

  it('should test map operator', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('--a--b--c--|', { a: 1, b: 2, c: 3 });
      const expected =    '--a--b--c--|';
      
      const result = source.pipe(map(x => x * 2));

      expectObservable(result).toBe(expected, { a: 2, b: 4, c: 6 });
    });
  });

  it('should test debounceTime', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('a-b-c-d-e-f----|');
      const expected =    '-------d----f----|';
      
      const result = source.pipe(debounceTime(3));

      expectObservable(result).toBe(expected);
    });
  });

  it('should test error handling', () => {
    scheduler.run(({ cold, expectObservable }) => {
      // '#' = error
      const source = cold('--a--b--#', { a: 1, b: 2 }, new Error('test error'));
      const expected =    '--a--b--#';

      expectObservable(source).toBe(expected, { a: 1, b: 2 }, new Error('test error'));
    });
  });

  it('should test catchError', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('--a--#', { a: 1 }, new Error('error'));
      const expected =    '--a--(b|)';
      
      const result = source.pipe(
        catchError(() => of('error caught'))
      );

      expectObservable(result).toBe(expected, { a: 1, b: 'error caught' });
    });
  });

  it('should test switchMap', () => {
    scheduler.run(({ cold, hot, expectObservable }) => {
      const source = hot('--a---b---c---|');
      const expected =   '---1--2---3---|';

      const result = source.pipe(
        switchMap(val => cold('-x|', { x: val + '1' }))
      );

      expectObservable(result).toBe(expected, { 1: 'a1', 2: 'b1', 3: 'c1' });
    });
  });

  it('should test combineLatest', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const a = cold('--a-----c-----|', { a: 1, c: 3 });
      const b = cold('----b-----d---|', { b: 2, d: 4 });
      const expected = '----x-y-z-w---|';

      const result = combineLatest([a, b]).pipe(
        map(([x, y]) => x + y)
      );

      expectObservable(result).toBe(expected, {
        x: 3, // a + b
        y: 3, // a + b (still)
        z: 5, // c + b
        w: 7  // c + d
      });
    });
  });
});

// Testing service with marble testing
@Injectable({ providedIn: 'root' })
export class SearchService {
  search(term: string): Observable<string[]> {
    return this.http.get<string[]>(`/api/search?q=${term}`);
  }

  searchWithDebounce(terms: Observable<string>): Observable<string[]> {
    return terms.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term => this.search(term)),
      catchError(() => of([]))
    );
  }
}

describe('SearchService with Marble Testing', () => {
  let scheduler: TestScheduler;
  let service: SearchService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [SearchService]
    });

    service = TestBed.inject(SearchService);
    httpMock = TestBed.inject(HttpTestingController);

    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should debounce search terms', () => {
    scheduler.run(({ cold, expectObservable, flush }) => {
      const searchTerms = cold('a-b-c---|', {
        a: 'ang',
        b: 'angu',
        c: 'angular'
      });

      const expected = '-----------r|';
      
      const result = service.searchWithDebounce(searchTerms);

      expectObservable(result).toBe(expected, {
        r: ['angular result']
      });

      flush();

      // Verify only last term was searched
      const req = httpMock.expectOne('/api/search?q=angular');
      req.flush(['angular result']);
    });
  });

  it('should cancel previous search on new term', () => {
    scheduler.run(({ cold, hot, expectObservable, flush }) => {
      const searchTerms = hot('a----b----|', {
        a: 'angular',
        b: 'react'
      });

      const expected = '------x-y-|';
      
      const result = searchTerms.pipe(
        switchMap(term => {
          return of(term).pipe(delay(2));
        })
      );

      expectObservable(result).toBe(expected, {
        x: 'angular',
        y: 'react'
      });
    });
  });
});

// Advanced: Testing custom operators
function filterNullAndUndefined<T>() {
  return (source: Observable<T | null | undefined>): Observable<T> =>
    source.pipe(
      filter((value): value is T => value != null)
    );
}

describe('Custom Operator Testing', () => {
  let scheduler: TestScheduler;

  beforeEach(() => {
    scheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });

  it('should filter null and undefined values', () => {
    scheduler.run(({ cold, expectObservable }) => {
      const source = cold('a-b-c-d-e-|', {
        a: 1,
        b: null,
        c: 2,
        d: undefined,
        e: 3
      });
      const expected =    'a---c---e-|';

      const result = source.pipe(filterNullAndUndefined());

      expectObservable(result).toBe(expected, {
        a: 1,
        c: 2,
        e: 3
      });
    });
  });
});
```

**Marble Syntax:**

```typescript
// Time progression
'-'   // 10ms of time passing
'--'  // 20ms of time passing

// Emissions
'a'   // Emit value 'a'
'ab'  // Emit 'a' then 'b' immediately

// Completion
'|'   // Stream completes
'a|'  // Emit 'a' then complete

// Error
'#'   // Stream errors
'a#'  // Emit 'a' then error

// Subscription
'^'   // Start of subscription
'!   // End of subscription

// Grouping
'(ab)' // Emit 'a' and 'b' in same frame

// Examples
'--a--b--|'           // Normal emissions
'(abc)|'              // Simultaneous emissions
'--a--b--#'           // Error after emissions
'^---!   --a--b--|'   // Unsubscribe before completion
```

**Follow-up Questions:**
1. What's the difference between cold and hot observables in marble testing?
2. How do you test observables that never complete?
3. When should you use marble testing vs fakeAsync?

**What Interviewers Look For:**
- Understanding of marble syntax
- Knowledge of when marble testing is beneficial
- Ability to test complex observable chains
- Understanding of hot vs cold observables

## Key Takeaways

1. **TestBed Configuration**: Use TestBed to create testing modules with proper imports, providers, and dependencies. Call compileComponents() for async compilation.

2. **Fixture vs DebugElement**: Fixture wraps the component, debugElement provides Angular abstraction over DOM, nativeElement is the actual DOM element.

3. **HTTP Testing**: Use HttpTestingController to mock HTTP requests, verify expectations, and provide mock responses without real API calls.

4. **Async Testing**: Use fakeAsync with tick/flush for synchronous async testing, waitForAsync for real async operations, and done callback for manual control.

5. **Marble Testing**: Use marble diagrams to test complex observable scenarios, especially useful for testing RxJS operators and timing-dependent logic.

6. **Service Testing**: Test services in isolation using spy objects for dependencies, or use TestBed for integration testing with real dependencies.

7. **Query Strategies**: Use By.css for CSS selectors, By.directive for components/directives. Prefer debugElement over nativeElement for platform independence.

8. **Change Detection**: Call fixture.detectChanges() manually in tests to trigger change detection and update the view.

9. **Mock Dependencies**: Create spy objects with jasmine.createSpyObj or provide mock implementations through TestBed providers.

10. **Test Organization**: Use beforeEach for setup, afterEach for cleanup, and describe blocks to organize related tests logically.

## Red Flags to Avoid

- Not calling httpMock.verify() to check for outstanding requests
- Forgetting fixture.detectChanges() after component changes
- Using nativeElement when debugElement would be more appropriate
- Not cleaning up subscriptions in component tests
- Forgetting discardPeriodicTasks in fakeAsync tests with intervals
- Testing implementation details instead of behavior
- Not testing error cases and edge conditions
- Creating tests that depend on execution order
- Not using data-testid attributes for reliable querying
- Writing tests that are too coupled to component internals

## Interview Preparation Tips

- Practice writing tests for components with various complexities
- Understand the difference between unit and integration tests
- Know how to test both synchronous and asynchronous code
- Be familiar with jasmine matchers and spy methods
- Practice using HttpTestingController for API testing
- Understand marble testing syntax and when to use it
- Know how to test reactive forms and template-driven forms
- Practice testing components with router and route params
- Understand how to test components with OnPush change detection
- Be ready to discuss test coverage and testing strategies
