# RxJS Combination Operators

## The Idea

**In plain English:** Combination operators are tools that let your program listen to multiple streams of events or data at the same time and decide how to blend their results together. A "stream" is just a sequence of values that arrive over time, like messages in a chat or readings from a sensor.

**Real-world analogy:** Imagine a TV news studio with three reporters — one covering sports, one covering weather, and one covering politics. A producer in the booth can choose different strategies: wait for all three to finish their segment before broadcasting a summary (forkJoin), broadcast a new update every time any reporter speaks (combineLatest), or only go live when the sports reporter talks and quietly note what the others are currently saying (withLatestFrom).

- The reporters = individual observable streams (sources of data)
- The producer's strategy = the combination operator chosen
- The broadcast output = the single combined observable your code subscribes to

---

## Table of Contents

- [Introduction](#introduction)
- [combineLatest](#combinelatest)
- [forkJoin](#forkjoin)
- [withLatestFrom](#withlatestfrom)
- [zip](#zip)
- [merge](#merge)
- [concat](#concat)
- [race](#race)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Combination operators are RxJS operators that work with multiple observables, combining their emissions in various ways. Understanding when to use each operator is crucial for building reactive Angular applications. These operators enable you to coordinate multiple data streams, synchronize async operations, and create complex data flows.

The key difference between combination operators lies in **when** and **how** they emit values, and **what happens** when one of the source observables completes or doesn't emit.

## combineLatest

`combineLatest` waits for all input observables to emit at least once, then emits an array (or object) containing the latest value from each input observable whenever any of them emits.

### Basic Usage

```typescript
import { combineLatest } from 'rxjs';
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-search-filter',
  template: `
    <input #searchBox placeholder="Search...">
    <select #category>
      <option value="">All</option>
      <option value="books">Books</option>
      <option value="electronics">Electronics</option>
    </select>
    <select #sort>
      <option value="price">Price</option>
      <option value="name">Name</option>
    </select>
  `
})
export class SearchFilterComponent implements OnInit {
  searchTerm$ = new Subject<string>();
  category$ = new Subject<string>();
  sortBy$ = new Subject<string>();

  ngOnInit() {
    // Combine all filter criteria
    combineLatest([
      this.searchTerm$,
      this.category$,
      this.sortBy$
    ]).subscribe(([search, category, sort]) => {
      console.log('Search:', search);
      console.log('Category:', category);
      console.log('Sort by:', sort);
      // Fetch filtered results
    });
  }
}
```

### Dictionary Syntax (Angular 14+)

```typescript
import { combineLatest } from 'rxjs';
import { map } from 'rxjs/operators';

interface FilterParams {
  search: string;
  category: string;
  priceRange: [number, number];
  inStock: boolean;
}

@Injectable()
export class ProductService {
  private searchTerm$ = new BehaviorSubject<string>('');
  private category$ = new BehaviorSubject<string>('');
  private priceRange$ = new BehaviorSubject<[number, number]>([0, 1000]);
  private inStock$ = new BehaviorSubject<boolean>(false);

  // Dictionary syntax for better readability
  filterParams$ = combineLatest({
    search: this.searchTerm$,
    category: this.category$,
    priceRange: this.priceRange$,
    inStock: this.inStock$
  });

  filteredProducts$ = this.filterParams$.pipe(
    map(params => this.filterProducts(params))
  );

  private filterProducts(params: FilterParams) {
    // Filter logic
    return this.products.filter(product => {
      const matchesSearch = product.name
        .toLowerCase()
        .includes(params.search.toLowerCase());
      const matchesCategory = !params.category || 
        product.category === params.category;
      const matchesPrice = product.price >= params.priceRange[0] && 
        product.price <= params.priceRange[1];
      const matchesStock = !params.inStock || product.inStock;
      
      return matchesSearch && matchesCategory && matchesPrice && matchesStock;
    });
  }
}
```

### Real-World Example: Form with Multiple Async Sources

```typescript
import { Component, inject } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { combineLatest, startWith, debounceTime, distinctUntilChanged } from 'rxjs';

@Component({
  selector: 'app-user-profile',
  template: `
    <form [formGroup]="form">
      <input formControlName="username">
      <span *ngIf="usernameAvailable$ | async as available">
        {{ available ? '✓ Available' : '✗ Taken' }}
      </span>
      
      <input formControlName="email">
      <select formControlName="country"></select>
      
      <button [disabled]="!(canSubmit$ | async)">
        Save Profile
      </button>
    </form>
  `
})
export class UserProfileComponent {
  private fb = inject(FormBuilder);
  private userService = inject(UserService);
  private geoService = inject(GeoService);

  form = this.fb.group({
    username: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
    country: ['', Validators.required]
  });

  // Check if all async validations pass
  canSubmit$ = combineLatest({
    formValid: this.form.statusChanges.pipe(
      startWith(this.form.status),
      map(status => status === 'VALID')
    ),
    usernameAvailable: this.form.get('username')!.valueChanges.pipe(
      startWith(''),
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(username => 
        username ? this.userService.checkUsername(username) : of(false)
      )
    ),
    emailVerified: this.form.get('email')!.valueChanges.pipe(
      startWith(''),
      debounceTime(300),
      switchMap(email => 
        email ? this.userService.verifyEmail(email) : of(false)
      )
    ),
    countryValid: this.form.get('country')!.valueChanges.pipe(
      startWith(''),
      map(country => !!country)
    )
  }).pipe(
    map(({ formValid, usernameAvailable, emailVerified, countryValid }) =>
      formValid && usernameAvailable && emailVerified && countryValid
    )
  );
}
```

### With Multiple HTTP Requests

```typescript
@Injectable()
export class DashboardService {
  constructor(private http: HttpClient) {}

  // Load all dashboard data in parallel
  loadDashboard(userId: string) {
    return combineLatest({
      user: this.http.get<User>(`/api/users/${userId}`),
      stats: this.http.get<Stats>(`/api/users/${userId}/stats`),
      notifications: this.http.get<Notification[]>(`/api/users/${userId}/notifications`),
      recentActivity: this.http.get<Activity[]>(`/api/users/${userId}/activity`)
    }).pipe(
      map(data => ({
        ...data,
        lastUpdated: new Date()
      }))
    );
  }
}

@Component({
  selector: 'app-dashboard',
  template: `
    <div *ngIf="dashboard$ | async as data">
      <app-user-profile [user]="data.user"></app-user-profile>
      <app-stats [stats]="data.stats"></app-stats>
      <app-notifications [items]="data.notifications"></app-notifications>
      <app-activity [items]="data.recentActivity"></app-activity>
    </div>
  `
})
export class DashboardComponent {
  private route = inject(ActivatedRoute);
  private dashboardService = inject(DashboardService);

  dashboard$ = this.route.params.pipe(
    map(params => params['userId']),
    switchMap(userId => this.dashboardService.loadDashboard(userId))
  );
}
```

## forkJoin

`forkJoin` waits for all input observables to complete, then emits a single value containing the last emission from each observable. If any observable never completes or errors, `forkJoin` will not emit.

### Basic Usage

```typescript
import { forkJoin } from 'rxjs';

@Injectable()
export class DataLoaderService {
  constructor(private http: HttpClient) {}

  // Load multiple resources that must all complete
  loadInitialData() {
    return forkJoin({
      config: this.http.get('/api/config'),
      user: this.http.get('/api/user'),
      permissions: this.http.get('/api/permissions')
    }).pipe(
      tap(data => console.log('All data loaded:', data)),
      catchError(error => {
        console.error('Failed to load initial data:', error);
        return throwError(() => error);
      })
    );
  }
}
```

### Real-World Example: Batch Operations

```typescript
@Injectable()
export class UserManagementService {
  constructor(private http: HttpClient) {}

  // Delete multiple users
  deleteUsers(userIds: string[]): Observable<{ success: string[], failed: string[] }> {
    const deleteRequests = userIds.map(id =>
      this.http.delete(`/api/users/${id}`).pipe(
        map(() => ({ id, success: true })),
        catchError(() => of({ id, success: false }))
      )
    );

    return forkJoin(deleteRequests).pipe(
      map(results => ({
        success: results.filter(r => r.success).map(r => r.id),
        failed: results.filter(r => !r.success).map(r => r.id)
      }))
    );
  }

  // Update multiple records
  bulkUpdate(updates: UserUpdate[]): Observable<BulkUpdateResult> {
    const updateRequests = updates.map(update =>
      this.http.put(`/api/users/${update.id}`, update.data).pipe(
        map(() => ({ id: update.id, status: 'success' as const })),
        catchError(error => of({ 
          id: update.id, 
          status: 'error' as const, 
          error: error.message 
        }))
      )
    );

    return forkJoin(updateRequests).pipe(
      map(results => ({
        total: results.length,
        successful: results.filter(r => r.status === 'success').length,
        failed: results.filter(r => r.status === 'error'),
        timestamp: new Date()
      }))
    );
  }
}
```

### Parallel File Uploads

```typescript
@Component({
  selector: 'app-file-uploader',
  template: `
    <input type="file" multiple (change)="onFilesSelected($event)">
    <button (click)="uploadAll()" [disabled]="!files.length || uploading">
      Upload All ({{ files.length }} files)
    </button>
    <div *ngIf="uploading">Uploading...</div>
    <div *ngIf="uploadResult">
      <p>{{ uploadResult.successful }} successful</p>
      <p>{{ uploadResult.failed.length }} failed</p>
    </div>
  `
})
export class FileUploaderComponent {
  private http = inject(HttpClient);
  files: File[] = [];
  uploading = false;
  uploadResult: any;

  onFilesSelected(event: Event) {
    const input = event.target as HTMLInputElement;
    this.files = Array.from(input.files || []);
  }

  uploadAll() {
    if (!this.files.length) return;

    this.uploading = true;

    const uploads = this.files.map(file => {
      const formData = new FormData();
      formData.append('file', file);

      return this.http.post('/api/upload', formData).pipe(
        map(() => ({ file: file.name, success: true })),
        catchError(error => of({ 
          file: file.name, 
          success: false, 
          error: error.message 
        }))
      );
    });

    forkJoin(uploads).subscribe({
      next: results => {
        this.uploadResult = {
          successful: results.filter(r => r.success).length,
          failed: results.filter(r => !r.success)
        };
        this.uploading = false;
      },
      error: error => {
        console.error('Upload batch failed:', error);
        this.uploading = false;
      }
    });
  }
}
```

## withLatestFrom

`withLatestFrom` combines the source observable with other observables, but only emits when the source emits. It takes the latest values from the other observables at that moment.

### Basic Usage

```typescript
import { withLatestFrom } from 'rxjs';

@Component({
  selector: 'app-search-with-filters',
  template: `
    <input (input)="search$.next($event.target.value)">
    <select (change)="category$.next($event.target.value)">
      <option value="">All</option>
      <option value="books">Books</option>
    </select>
  `
})
export class SearchComponent implements OnInit {
  search$ = new Subject<string>();
  category$ = new BehaviorSubject<string>('');
  
  ngOnInit() {
    // Only search when user types, using current category
    this.search$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      withLatestFrom(this.category$),
      switchMap(([searchTerm, category]) => 
        this.searchService.search(searchTerm, category)
      )
    ).subscribe(results => {
      console.log('Search results:', results);
    });
  }
}
```

### Real-World Example: Save with Current State

```typescript
@Injectable()
export class DocumentService {
  private document$ = new BehaviorSubject<Document>(null);
  private isDirty$ = new BehaviorSubject<boolean>(false);
  private saveClicks$ = new Subject<void>();

  constructor(private http: HttpClient) {
    // Save button clicked - grab current document and dirty state
    this.saveClicks$.pipe(
      withLatestFrom(this.document$, this.isDirty$),
      filter(([_, doc, isDirty]) => isDirty && !!doc),
      switchMap(([_, doc]) => this.http.put(`/api/documents/${doc.id}`, doc)),
      tap(() => this.isDirty$.next(false))
    ).subscribe({
      next: () => console.log('Document saved'),
      error: err => console.error('Save failed:', err)
    });
  }

  save() {
    this.saveClicks$.next();
  }

  updateDocument(doc: Document) {
    this.document$.next(doc);
    this.isDirty$.next(true);
  }
}
```

### Form Submit with User Context

```typescript
@Component({
  selector: 'app-comment-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <textarea formControlName="text"></textarea>
      <button type="submit">Post Comment</button>
    </form>
  `
})
export class CommentFormComponent {
  private authService = inject(AuthService);
  private commentService = inject(CommentService);
  private submitClicks$ = new Subject<void>();

  form = new FormBuilder().group({
    text: ['', Validators.required]
  });

  constructor() {
    // When submitting, attach current user info
    this.submitClicks$.pipe(
      withLatestFrom(
        this.authService.currentUser$,
        this.form.valueChanges.pipe(startWith(this.form.value))
      ),
      filter(([_, user, formValue]) => !!user && this.form.valid),
      switchMap(([_, user, formValue]) =>
        this.commentService.createComment({
          text: formValue.text,
          userId: user.id,
          username: user.username,
          timestamp: new Date()
        })
      )
    ).subscribe({
      next: () => {
        this.form.reset();
        console.log('Comment posted');
      }
    });
  }

  onSubmit() {
    if (this.form.valid) {
      this.submitClicks$.next();
    }
  }
}
```

## zip

`zip` waits for all input observables to emit, then combines the nth emission from each observable into a single array. It pairs emissions by index.

### Basic Usage

```typescript
import { zip, interval, of } from 'rxjs';
import { take } from 'rxjs/operators';

// Pair values by index
const letters$ = of('A', 'B', 'C', 'D');
const numbers$ = of(1, 2, 3);
const symbols$ = of('!', '@', '#', '$', '%');

zip(letters$, numbers$, symbols$).subscribe(
  ([letter, number, symbol]) => {
    console.log(`${letter}${number}${symbol}`);
    // Output: A1!, B2@, C3#
    // Stops at 3 because numbers$ only has 3 values
  }
);
```

### Real-World Example: Paginated Parallel Requests

```typescript
@Injectable()
export class DataSyncService {
  constructor(private http: HttpClient) {}

  // Fetch pages in order, pairing requests with their page numbers
  fetchPagesInOrder(baseUrl: string, totalPages: number) {
    const pageNumbers$ = from(Array.from({ length: totalPages }, (_, i) => i + 1));
    
    const requests$ = pageNumbers$.pipe(
      map(page => this.http.get(`${baseUrl}?page=${page}`))
    );

    // Zip ensures we get results in order, paired with page numbers
    return zip(pageNumbers$, requests$).pipe(
      map(([pageNum, data]) => ({ page: pageNum, data })),
      toArray() // Collect all pages
    );
  }
}
```

### Coordinating Multiple Async Operations

```typescript
interface StepResult {
  step: number;
  data: any;
  timestamp: Date;
}

@Injectable()
export class MultiStepProcessService {
  constructor(private http: HttpClient) {}

  // Execute steps that depend on each other's completion timing
  runMultiStepProcess(entityId: string): Observable<StepResult[]> {
    const step1$ = this.http.post(`/api/step1/${entityId}`, {}).pipe(
      map(data => ({ step: 1, data, timestamp: new Date() }))
    );

    const step2$ = this.http.post(`/api/step2/${entityId}`, {}).pipe(
      map(data => ({ step: 2, data, timestamp: new Date() }))
    );

    const step3$ = this.http.post(`/api/step3/${entityId}`, {}).pipe(
      map(data => ({ step: 3, data, timestamp: new Date() }))
    );

    // All steps run in parallel, but results are paired by completion order
    return zip([step1$, step2$, step3$]).pipe(
      map(results => {
        console.log('All steps completed');
        return results;
      })
    );
  }
}
```

## merge

`merge` subscribes to all input observables simultaneously and emits values from any of them as they arrive. All emissions are flattened into a single stream.

### Basic Usage

```typescript
import { merge, fromEvent } from 'rxjs';
import { mapTo } from 'rxjs/operators';

@Component({
  selector: 'app-activity-tracker',
  template: `<div>Last activity: {{ lastActivity | date:'medium' }}</div>`
})
export class ActivityTrackerComponent implements OnInit {
  lastActivity = new Date();

  ngOnInit() {
    const clicks$ = fromEvent(document, 'click').pipe(mapTo('click'));
    const keypress$ = fromEvent(document, 'keypress').pipe(mapTo('keypress'));
    const scroll$ = fromEvent(document, 'scroll').pipe(mapTo('scroll'));

    // Merge all activity streams
    merge(clicks$, keypress$, scroll$).pipe(
      debounceTime(1000)
    ).subscribe(activityType => {
      this.lastActivity = new Date();
      console.log('Activity detected:', activityType);
    });
  }
}
```

### Real-World Example: Multiple Data Sources

```typescript
@Injectable()
export class NotificationService {
  private webSocketNotifications$: Observable<Notification>;
  private pollingNotifications$: Observable<Notification>;
  private localNotifications$ = new Subject<Notification>();

  constructor(
    private wsService: WebSocketService,
    private http: HttpClient
  ) {
    this.webSocketNotifications$ = this.wsService.connect('notifications');
    
    this.pollingNotifications$ = interval(30000).pipe(
      switchMap(() => this.http.get<Notification[]>('/api/notifications')),
      switchMap(notifications => from(notifications))
    );
  }

  // Combine all notification sources
  getAllNotifications(): Observable<Notification> {
    return merge(
      this.webSocketNotifications$,
      this.pollingNotifications$,
      this.localNotifications$
    ).pipe(
      distinctUntilKeyChanged('id'),
      tap(notification => this.showToast(notification))
    );
  }

  addLocalNotification(notification: Notification) {
    this.localNotifications$.next(notification);
  }
}
```

## concat

`concat` subscribes to observables in sequence - only subscribing to the next observable after the previous one completes.

### Basic Usage

```typescript
import { concat, of } from 'rxjs';
import { delay } from 'rxjs/operators';

@Injectable()
export class SequentialTaskService {
  constructor(private http: HttpClient) {}

  // Execute tasks in strict order
  runTasksInSequence(userId: string) {
    const task1$ = this.http.post('/api/task1', { userId }).pipe(
      tap(() => console.log('Task 1 complete'))
    );

    const task2$ = this.http.post('/api/task2', { userId }).pipe(
      tap(() => console.log('Task 2 complete'))
    );

    const task3$ = this.http.post('/api/task3', { userId }).pipe(
      tap(() => console.log('Task 3 complete'))
    );

    return concat(task1$, task2$, task3$).pipe(
      toArray() // Collect all results
    );
  }
}
```

## race

`race` subscribes to all observables but only emits from the first one to emit, unsubscribing from all others.

### Basic Usage

```typescript
import { race } from 'rxjs';

@Injectable()
export class FastestDataService {
  constructor(private http: HttpClient) {}

  // Use whichever server responds first
  getFastestData(id: string) {
    const server1$ = this.http.get(`https://server1.com/api/data/${id}`);
    const server2$ = this.http.get(`https://server2.com/api/data/${id}`);
    const server3$ = this.http.get(`https://server3.com/api/data/${id}`);

    return race(server1$, server2$, server3$).pipe(
      tap(() => console.log('Got response from fastest server'))
    );
  }

  // Timeout pattern
  getDataWithTimeout(id: string, timeoutMs: number) {
    const data$ = this.http.get(`/api/data/${id}`);
    const timeout$ = timer(timeoutMs).pipe(
      switchMap(() => throwError(() => new Error('Request timeout')))
    );

    return race(data$, timeout$);
  }
}
```

## Common Mistakes

### 1. Using forkJoin with Non-Completing Observables

```typescript
// WRONG: forkJoin will never emit
const wrong$ = forkJoin({
  data: this.http.get('/api/data'), // Completes ✓
  clicks: fromEvent(button, 'click') // Never completes ✗
});

// RIGHT: Use combineLatest for ongoing streams
const right$ = combineLatest({
  data: this.http.get('/api/data'),
  clicks: fromEvent(button, 'click').pipe(take(1)) // Or limit emissions
});
```

### 2. Confusing withLatestFrom Direction

```typescript
// WRONG: category$ change won't trigger search
this.category$.pipe(
  withLatestFrom(this.search$),
  switchMap(([category, search]) => this.api.search(search, category))
);

// RIGHT: search$ is the primary stream
this.search$.pipe(
  withLatestFrom(this.category$),
  switchMap(([search, category]) => this.api.search(search, category))
);
```

### 3. Using combineLatest Without Initial Values

```typescript
// WRONG: Won't emit until ALL observables emit
combineLatest([
  this.filter$, // No initial value
  this.sort$    // No initial value
]).subscribe(/* ... */);

// RIGHT: Provide initial values
combineLatest([
  this.filter$.pipe(startWith('')),
  this.sort$.pipe(startWith('name'))
]).subscribe(/* ... */);
```

### 4. Memory Leaks with merge/combineLatest

```typescript
// WRONG: Subscriptions never cleaned up
ngOnInit() {
  merge(this.source1$, this.source2$, this.source3$)
    .subscribe(value => this.handleValue(value));
}

// RIGHT: Use takeUntilDestroyed
private destroy$ = new Subject<void>();

ngOnInit() {
  merge(this.source1$, this.source2$, this.source3$)
    .pipe(takeUntil(this.destroy$))
    .subscribe(value => this.handleValue(value));
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

## Best Practices

### 1. Choose the Right Operator

```typescript
// Use combineLatest when you need the latest from ALL sources
const filters$ = combineLatest({
  search: this.search$,
  category: this.category$,
  sort: this.sort$
});

// Use forkJoin for parallel HTTP requests that must all complete
const initialData$ = forkJoin({
  user: this.http.get('/api/user'),
  config: this.http.get('/api/config')
});

// Use withLatestFrom when one stream is primary
const saveWithContext$ = this.saveClicks$.pipe(
  withLatestFrom(this.currentUser$, this.document$)
);

// Use merge for combining event streams
const allEvents$ = merge(clicks$, touches$, keyPresses$);
```

### 2. Type Safety with Dictionary Syntax

```typescript
// Typed result object
interface DashboardData {
  user: User;
  stats: Stats;
  notifications: Notification[];
}

const dashboard$ = combineLatest({
  user: this.userService.getUser(),
  stats: this.statsService.getStats(),
  notifications: this.notificationService.getNotifications()
} as const).pipe(
  map((data): DashboardData => data)
);
```

### 3. Error Handling in Combinations

```typescript
// Handle errors in individual streams
const safeData$ = forkJoin({
  critical: this.http.get('/api/critical'),
  optional: this.http.get('/api/optional').pipe(
    catchError(() => of(null)) // Don't fail entire forkJoin
  )
});
```

## Interview Questions

### Q1: What's the difference between combineLatest and forkJoin?

**Answer:**
- **combineLatest**: Emits whenever any source emits (after all have emitted once). Continues emitting. Use for ongoing streams.
- **forkJoin**: Waits for all sources to complete, emits once with final values. Use for parallel HTTP requests.

```typescript
// combineLatest - emits multiple times
combineLatest([intervalA$, intervalB$]) // Emits: [a1,b1], [a2,b1], [a2,b2], ...

// forkJoin - emits once when all complete
forkJoin([httpRequestA$, httpRequestB$]) // Emits once: [resultA, resultB]
```

### Q2: When should you use withLatestFrom vs combineLatest?

**Answer:** Use `withLatestFrom` when you have a primary stream and only want emissions when the primary stream emits. Use `combineLatest` when all streams are equal and any emission should trigger.

```typescript
// withLatestFrom - only emits when button clicked
buttonClick$.pipe(withLatestFrom(userName$))

// combineLatest - emits when button OR userName changes
combineLatest([buttonClick$, userName$])
```

### Q3: What happens if one observable in forkJoin errors?

**Answer:** The entire `forkJoin` will error immediately and not emit any values, even from successfully completed observables.

```typescript
// If request2 fails, entire forkJoin errors
forkJoin({
  request1: this.http.get('/api/1'), // Success
  request2: this.http.get('/api/2'), // Fails - whole thing errors
  request3: this.http.get('/api/3')  // Never executes
});

// Better: Handle errors individually
forkJoin({
  request1: this.http.get('/api/1'),
  request2: this.http.get('/api/2').pipe(catchError(() => of(null))),
  request3: this.http.get('/api/3').pipe(catchError(() => of(null)))
});
```

## Key Takeaways

1. **combineLatest** - Latest from all sources, emits when any source emits
2. **forkJoin** - Wait for all to complete, emit once with final values
3. **withLatestFrom** - Primary source drives emissions, pull latest from others
4. **zip** - Pair emissions by index, wait for all sources per emission
5. **merge** - Flatten all emissions into single stream
6. **concat** - Subscribe sequentially, wait for each to complete
7. **race** - First to emit wins, unsubscribe from others
8. Choose based on **when** you need values and **how** they should combine
9. Use dictionary syntax for better type safety and readability
10. Handle errors appropriately to prevent entire combinations from failing

## Resources

- [RxJS Documentation - Combination Operators](https://rxjs.dev/guide/operators#join-creation-operators)
- [Learn RxJS - Combination Operators](https://www.learnrxjs.io/learn-rxjs/operators/combination)
- [RxJS Marbles - Interactive Diagrams](https://rxmarbles.com/)
- [Angular Blog - Combination Operators](https://blog.angular.io/rxjs-in-angular-combining-multiple-observables-8c3a97d7c11c)
