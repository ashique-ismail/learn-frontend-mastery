# System Design: Twitter Feed

## The Idea

**In plain English:** Designing a Twitter feed means planning how to build a system that can collect posts (tweets) from thousands of people and instantly show the right ones to each user in a smooth, scrollable list — even when millions of people are using it at the same time.

**Real-world analogy:** Imagine a busy coffee shop bulletin board where regulars pin new notes every few seconds, and each customer only wants to see notes from their friends. A staff member watches for new notes, grabs the latest ones for you, and quietly removes old ones you've scrolled past so the board never gets overloaded:

- The bulletin board = the feed UI displayed in your browser
- The staff member watching for new notes = the WebSocket connection that listens for real-time updates
- Grabbing notes in batches as you scroll down = cursor-based pagination (loading tweets in chunks instead of all at once)
- Removing notes you've already passed = virtual scrolling (only keeping visible tweet cards in memory)

---

## Overview

Design a scalable Twitter-like feed with infinite scroll, real-time updates, pagination, caching, virtualization, and efficient state management. This document covers architecture decisions, implementation strategies, and trade-offs for building a production-ready social media feed.

## Requirements Analysis

### Functional Requirements

1. Display tweets in reverse chronological order
2. Infinite scroll with smooth loading
3. Real-time updates for new tweets
4. Like, retweet, and reply interactions
5. Media content (images, videos)
6. User profiles and avatars
7. Timestamp display (relative and absolute)
8. Pull-to-refresh

### Non-Functional Requirements

1. Load initial feed < 2 seconds
2. Smooth 60fps scrolling
3. Handle 10,000+ tweets without performance degradation
4. Work offline with cached data
5. Real-time updates within 1 second
6. Responsive across devices
7. Handle slow network gracefully

### Scale Considerations

- 100M+ daily active users
- 500M tweets per day
- 6,000 tweets per second average
- 12,000 tweets per second peak
- Each user follows 100-500 accounts
- Feed size: ~800 tweets at a time

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Client Layer                         │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Feed       │  │  Real-time   │  │   Cache      │  │
│  │ Component    │  │  WebSocket   │  │   Layer      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    State Management                     │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Redux/NgRx - Normalized Tweet Store             │  │
│  └──────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    API Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  REST API    │  │  GraphQL     │  │  WebSocket   │  │
│  │  Client      │  │  Client      │  │  Client      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Implementation Strategy

### 1. Infinite Scroll Architecture

**Pagination Approaches:**

```typescript
// Cursor-based pagination (recommended for Twitter-like feeds)
interface FeedResponse {
  tweets: Tweet[];
  nextCursor: string | null;
  hasMore: boolean;
}

@Injectable({ providedIn: 'root' })
export class FeedService {
  private readonly apiUrl = '/api/feed';

  constructor(private http: HttpClient) {}

  getFeed(cursor?: string, limit: number = 20): Observable<FeedResponse> {
    const params: any = { limit };
    if (cursor) {
      params.cursor = cursor;
    }

    return this.http.get<FeedResponse>(this.apiUrl, { params });
  }

  // Alternative: Timestamp-based pagination
  getFeedByTimestamp(
    beforeTimestamp?: number,
    limit: number = 20
  ): Observable<FeedResponse> {
    const params: any = { limit };
    if (beforeTimestamp) {
      params.before = beforeTimestamp;
    }

    return this.http.get<FeedResponse>(this.apiUrl, { params });
  }
}

// State management with NgRx
interface FeedState {
  tweets: { [id: string]: Tweet };
  tweetIds: string[];
  isLoading: boolean;
  hasMore: boolean;
  nextCursor: string | null;
  error: string | null;
}

const initialState: FeedState = {
  tweets: {},
  tweetIds: [],
  isLoading: false,
  hasMore: true,
  nextCursor: null,
  error: null
};

// Actions
export const loadFeed = createAction('[Feed] Load Feed');
export const loadFeedSuccess = createAction(
  '[Feed] Load Feed Success',
  props<{ response: FeedResponse }>()
);
export const loadFeedFailure = createAction(
  '[Feed] Load Feed Failure',
  props<{ error: string }>()
);
export const loadMore = createAction('[Feed] Load More');

// Reducer
export const feedReducer = createReducer(
  initialState,
  on(loadFeed, loadMore, (state) => ({
    ...state,
    isLoading: true,
    error: null
  })),
  on(loadFeedSuccess, (state, { response }) => {
    const newTweets = response.tweets.reduce((acc, tweet) => ({
      ...acc,
      [tweet.id]: tweet
    }), {});

    return {
      ...state,
      tweets: { ...state.tweets, ...newTweets },
      tweetIds: [...state.tweetIds, ...response.tweets.map(t => t.id)],
      isLoading: false,
      hasMore: response.hasMore,
      nextCursor: response.nextCursor,
      error: null
    };
  }),
  on(loadFeedFailure, (state, { error }) => ({
    ...state,
    isLoading: false,
    error
  }))
);

// Effects
@Injectable()
export class FeedEffects {
  loadFeed$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadFeed),
      withLatestFrom(this.store.select(selectNextCursor)),
      switchMap(([_, cursor]) =>
        this.feedService.getFeed(cursor).pipe(
          map(response => loadFeedSuccess({ response })),
          catchError(error => of(loadFeedFailure({ error: error.message })))
        )
      )
    )
  );

  loadMore$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadMore),
      withLatestFrom(
        this.store.select(selectNextCursor),
        this.store.select(selectIsLoading),
        this.store.select(selectHasMore)
      ),
      filter(([_, __, isLoading, hasMore]) => !isLoading && hasMore),
      switchMap(([_, cursor]) =>
        this.feedService.getFeed(cursor).pipe(
          map(response => loadFeedSuccess({ response })),
          catchError(error => of(loadFeedFailure({ error: error.message })))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private store: Store,
    private feedService: FeedService
  ) {}
}

// Component implementation
@Component({
  selector: 'app-feed',
  standalone: true,
  template: `
    <div class="feed-container" #container>
      <!-- Pull to refresh -->
      <div class="pull-to-refresh" 
           [class.active]="pullToRefreshActive"
           [style.transform]="'translateY(' + pullDistance + 'px)'">
        <app-loading-spinner></app-loading-spinner>
      </div>

      <!-- Tweet list -->
      <app-tweet-card
        *ngFor="let tweet of tweets$ | async; trackBy: trackByTweetId"
        [tweet]="tweet"
        (like)="onLike($event)"
        (retweet)="onRetweet($event)"
        (reply)="onReply($event)">
      </app-tweet-card>

      <!-- Loading indicator -->
      <div class="loading-more" *ngIf="isLoading$ | async">
        <app-loading-spinner></app-loading-spinner>
      </div>

      <!-- End of feed -->
      <div class="end-of-feed" *ngIf="!(hasMore$ | async) && !(isLoading$ | async)">
        You're all caught up!
      </div>

      <!-- Error state -->
      <div class="error-state" *ngIf="error$ | async as error">
        {{ error }}
        <button (click)="retry()">Retry</button>
      </div>

      <!-- Infinite scroll sentinel -->
      <div #sentinel class="scroll-sentinel"></div>
    </div>
  `,
  styles: [`
    .feed-container {
      max-width: 600px;
      margin: 0 auto;
      position: relative;
    }

    .scroll-sentinel {
      height: 1px;
      position: absolute;
      bottom: 500px; /* Trigger load before reaching bottom */
    }
  `]
})
export class FeedComponent implements OnInit, OnDestroy, AfterViewInit {
  @ViewChild('sentinel') sentinel!: ElementRef;
  @ViewChild('container') container!: ElementRef;

  tweets$ = this.store.select(selectAllTweets);
  isLoading$ = this.store.select(selectIsLoading);
  hasMore$ = this.store.select(selectHasMore);
  error$ = this.store.select(selectError);

  private observer!: IntersectionObserver;
  private destroy$ = new Subject<void>();
  
  pullToRefreshActive = false;
  pullDistance = 0;
  private startY = 0;

  constructor(
    private store: Store,
    private renderer: Renderer2
  ) {}

  ngOnInit() {
    // Load initial feed
    this.store.dispatch(loadFeed());
  }

  ngAfterViewInit() {
    // Setup infinite scroll with Intersection Observer
    this.setupInfiniteScroll();
    this.setupPullToRefresh();
  }

  private setupInfiniteScroll() {
    const options = {
      root: null,
      rootMargin: '0px',
      threshold: 0.1
    };

    this.observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          // Sentinel is visible, load more
          this.store.dispatch(loadMore());
        }
      });
    }, options);

    this.observer.observe(this.sentinel.nativeElement);
  }

  private setupPullToRefresh() {
    const container = this.container.nativeElement;

    fromEvent<TouchEvent>(container, 'touchstart')
      .pipe(takeUntil(this.destroy$))
      .subscribe(e => {
        this.startY = e.touches[0].clientY;
      });

    fromEvent<TouchEvent>(container, 'touchmove')
      .pipe(
        takeUntil(this.destroy$),
        filter(() => container.scrollTop === 0)
      )
      .subscribe(e => {
        const currentY = e.touches[0].clientY;
        const distance = currentY - this.startY;

        if (distance > 0) {
          this.pullDistance = Math.min(distance, 100);
          this.pullToRefreshActive = this.pullDistance > 60;
        }
      });

    fromEvent<TouchEvent>(container, 'touchend')
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        if (this.pullToRefreshActive) {
          this.refresh();
        }
        this.pullDistance = 0;
        this.pullToRefreshActive = false;
      });
  }

  refresh() {
    this.store.dispatch(refreshFeed());
  }

  retry() {
    this.store.dispatch(loadMore());
  }

  trackByTweetId(index: number, tweet: Tweet): string {
    return tweet.id;
  }

  onLike(tweetId: string) {
    this.store.dispatch(likeTweet({ tweetId }));
  }

  onRetweet(tweetId: string) {
    this.store.dispatch(retweetTweet({ tweetId }));
  }

  onReply(tweetId: string) {
    // Navigate to reply screen
  }

  ngOnDestroy() {
    this.observer?.disconnect();
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Pagination Strategies Comparison:**

| Strategy | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Offset/Limit** | Simple, easy to implement | Poor performance at high offsets, data duplication issues | Small datasets |
| **Cursor-based** | Consistent results, no duplication | More complex, requires indexed field | Real-time feeds |
| **Timestamp-based** | Simple, works with time-ordered data | Can miss items with same timestamp | Time-ordered feeds |
| **Keyset pagination** | Efficient, consistent | Requires unique sortable key | Large datasets |

### 2. Real-time Updates Strategy

**WebSocket vs Polling Trade-offs:**

```typescript
// WebSocket implementation (recommended for real-time)
@Injectable({ providedIn: 'root' })
export class RealtimeFeedService {
  private socket$!: WebSocketSubject<any>;
  private reconnectAttempts = 0;
  private readonly maxReconnectAttempts = 5;

  constructor() {
    this.connect();
  }

  private connect() {
    this.socket$ = new WebSocketSubject({
      url: 'wss://api.example.com/feed',
      openObserver: {
        next: () => {
          console.log('WebSocket connected');
          this.reconnectAttempts = 0;
        }
      },
      closeObserver: {
        next: () => {
          console.log('WebSocket closed');
          this.reconnect();
        }
      }
    });
  }

  private reconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      
      setTimeout(() => {
        console.log(`Reconnecting... attempt ${this.reconnectAttempts}`);
        this.connect();
      }, delay);
    }
  }

  getNewTweets(): Observable<Tweet> {
    return this.socket$.pipe(
      filter((msg: any) => msg.type === 'new_tweet'),
      map((msg: any) => msg.data),
      catchError(error => {
        console.error('WebSocket error:', error);
        return EMPTY;
      })
    );
  }

  getTweetUpdates(): Observable<TweetUpdate> {
    return this.socket$.pipe(
      filter((msg: any) => msg.type === 'tweet_update'),
      map((msg: any) => msg.data)
    );
  }

  sendAction(action: any) {
    this.socket$.next(action);
  }

  disconnect() {
    this.socket$.complete();
  }
}

// Polling fallback for unreliable connections
@Injectable({ providedIn: 'root' })
export class PollingFeedService {
  private readonly pollingInterval = 5000; // 5 seconds

  constructor(private http: HttpClient) {}

  pollNewTweets(sinceId: string): Observable<Tweet[]> {
    return timer(0, this.pollingInterval).pipe(
      switchMap(() =>
        this.http.get<Tweet[]>('/api/feed/new', {
          params: { since_id: sinceId }
        })
      ),
      filter(tweets => tweets.length > 0),
      catchError(error => {
        console.error('Polling error:', error);
        return of([]);
      })
    );
  }
}

// Adaptive strategy: WebSocket with polling fallback
@Injectable({ providedIn: 'root' })
export class AdaptiveFeedService {
  private useWebSocket = true;

  constructor(
    private realtimeService: RealtimeFeedService,
    private pollingService: PollingFeedService
  ) {}

  getNewTweets(sinceId: string): Observable<Tweet | Tweet[]> {
    if (this.useWebSocket) {
      return this.realtimeService.getNewTweets().pipe(
        timeout(10000),
        catchError(() => {
          // Fallback to polling
          this.useWebSocket = false;
          return this.pollingService.pollNewTweets(sinceId);
        })
      );
    } else {
      return this.pollingService.pollNewTweets(sinceId);
    }
  }
}

// Effects for real-time updates
@Injectable()
export class RealtimeEffects {
  newTweets$ = createEffect(() =>
    this.realtimeService.getNewTweets().pipe(
      map(tweet => addNewTweet({ tweet }))
    )
  );

  tweetUpdates$ = createEffect(() =>
    this.realtimeService.getTweetUpdates().pipe(
      map(update => updateTweet({ update }))
    )
  );

  // Show notification for new tweets
  newTweetNotification$ = createEffect(() =>
    this.actions$.pipe(
      ofType(addNewTweet),
      tap(({ tweet }) => {
        this.showNewTweetNotification(tweet);
      })
    ),
    { dispatch: false }
  );

  constructor(
    private actions$: Actions,
    private realtimeService: RealtimeFeedService
  ) {}

  private showNewTweetNotification(tweet: Tweet) {
    // Show "New tweets available" banner
  }
}
```

**Real-time Update Strategies:**

1. **Immediate Insert**: Add new tweets at the top immediately
2. **Notification Banner**: Show "New tweets available" button
3. **Auto-refresh**: Automatically refresh after X new tweets
4. **Hybrid**: Banner for first few, auto-refresh threshold

```typescript
// Notification banner approach (better UX)
@Component({
  selector: 'app-feed',
  template: `
    <!-- New tweets banner -->
    <div class="new-tweets-banner" 
         *ngIf="newTweetsCount > 0"
         (click)="loadNewTweets()">
      {{ newTweetsCount }} new tweet{{ newTweetsCount > 1 ? 's' : '' }}
    </div>

    <!-- Feed content -->
  `
})
export class FeedComponent {
  newTweetsCount = 0;

  constructor(private store: Store) {
    this.store.select(selectPendingTweetsCount)
      .subscribe(count => {
        this.newTweetsCount = count;
      });
  }

  loadNewTweets() {
    this.store.dispatch(mergePendingTweets());
    
    // Smooth scroll to top
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
}
```

### 3. Caching Strategy

**Multi-layer Caching:**

```typescript
// Memory cache (in-app)
@Injectable({ providedIn: 'root' })
export class MemoryCacheService {
  private cache = new Map<string, { data: any; timestamp: number }>();
  private readonly TTL = 5 * 60 * 1000; // 5 minutes

  set(key: string, data: any) {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }

  get(key: string): any | null {
    const cached = this.cache.get(key);
    
    if (!cached) return null;
    
    const age = Date.now() - cached.timestamp;
    if (age > this.TTL) {
      this.cache.delete(key);
      return null;
    }
    
    return cached.data;
  }

  clear() {
    this.cache.clear();
  }
}

// IndexedDB cache (persistent)
@Injectable({ providedIn: 'root' })
export class IndexedDBCacheService {
  private db!: IDBDatabase;

  constructor() {
    this.initDB();
  }

  private async initDB() {
    const request = indexedDB.open('FeedCache', 1);

    request.onupgradeneeded = (event: any) => {
      const db = event.target.result;
      
      if (!db.objectStoreNames.contains('tweets')) {
        const store = db.createObjectStore('tweets', { keyPath: 'id' });
        store.createIndex('timestamp', 'timestamp', { unique: false });
      }
    };

    request.onsuccess = (event: any) => {
      this.db = event.target.result;
    };
  }

  async cacheTweets(tweets: Tweet[]): Promise<void> {
    const transaction = this.db.transaction(['tweets'], 'readwrite');
    const store = transaction.objectStore('tweets');

    tweets.forEach(tweet => {
      store.put({
        ...tweet,
        cachedAt: Date.now()
      });
    });

    return new Promise((resolve, reject) => {
      transaction.oncomplete = () => resolve();
      transaction.onerror = () => reject(transaction.error);
    });
  }

  async getCachedTweets(limit: number = 20): Promise<Tweet[]> {
    const transaction = this.db.transaction(['tweets'], 'readonly');
    const store = transaction.objectStore('tweets');
    const index = store.index('timestamp');

    const request = index.openCursor(null, 'prev');
    const tweets: Tweet[] = [];

    return new Promise((resolve) => {
      request.onsuccess = (event: any) => {
        const cursor = event.target.result;
        if (cursor && tweets.length < limit) {
          tweets.push(cursor.value);
          cursor.continue();
        } else {
          resolve(tweets);
        }
      };
    });
  }
}

// HTTP cache with interceptor
@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  constructor(private cache: MemoryCacheService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Only cache GET requests
    if (req.method !== 'GET') {
      return next.handle(req);
    }

    // Check cache
    const cached = this.cache.get(req.url);
    if (cached) {
      return of(new HttpResponse({ body: cached, status: 200 }));
    }

    // Fetch and cache
    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.cache.set(req.url, event.body);
        }
      })
    );
  }
}
```

### 4. Virtual Scrolling for Performance

```typescript
// Using CDK Virtual Scroll
@Component({
  selector: 'app-virtualized-feed',
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport 
      [itemSize]="200" 
      class="feed-viewport"
      (scrolledIndexChange)="onScroll($event)">
      
      <app-tweet-card
        *cdkVirtualFor="let tweet of tweets; trackBy: trackByTweetId"
        [tweet]="tweet">
      </app-tweet-card>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .feed-viewport {
      height: 100vh;
      width: 100%;
    }
  `]
})
export class VirtualizedFeedComponent {
  tweets: Tweet[] = [];

  onScroll(index: number) {
    // Load more when near bottom
    if (index > this.tweets.length - 10) {
      this.loadMore();
    }
  }

  trackByTweetId(index: number, tweet: Tweet): string {
    return tweet.id;
  }

  loadMore() {
    // Load more tweets
  }
}

// Custom virtual scroll for dynamic heights
@Component({
  selector: 'app-custom-virtual-scroll',
  template: `
    <div class="viewport" #viewport (scroll)="onScroll()">
      <div class="spacer" [style.height.px]="topPadding"></div>
      
      <app-tweet-card
        *ngFor="let tweet of visibleTweets; trackBy: trackByTweetId"
        [tweet]="tweet">
      </app-tweet-card>
      
      <div class="spacer" [style.height.px]="bottomPadding"></div>
    </div>
  `
})
export class CustomVirtualScrollComponent implements OnInit, AfterViewInit {
  @ViewChild('viewport') viewport!: ElementRef;
  
  allTweets: Tweet[] = [];
  visibleTweets: Tweet[] = [];
  topPadding = 0;
  bottomPadding = 0;
  
  private readonly avgItemHeight = 200;
  private readonly buffer = 5; // Render 5 items above/below viewport

  ngAfterViewInit() {
    this.updateVisibleItems();
  }

  onScroll() {
    requestAnimationFrame(() => {
      this.updateVisibleItems();
    });
  }

  private updateVisibleItems() {
    const viewport = this.viewport.nativeElement;
    const scrollTop = viewport.scrollTop;
    const viewportHeight = viewport.clientHeight;

    const startIndex = Math.max(0, 
      Math.floor(scrollTop / this.avgItemHeight) - this.buffer
    );
    const endIndex = Math.min(this.allTweets.length,
      Math.ceil((scrollTop + viewportHeight) / this.avgItemHeight) + this.buffer
    );

    this.visibleTweets = this.allTweets.slice(startIndex, endIndex);
    this.topPadding = startIndex * this.avgItemHeight;
    this.bottomPadding = (this.allTweets.length - endIndex) * this.avgItemHeight;
  }

  trackByTweetId(index: number, tweet: Tweet): string {
    return tweet.id;
  }
}
```

## Key Takeaways

1. **Cursor-based Pagination**: Use cursor-based pagination for consistent results in real-time feeds, avoiding offset pagination issues with data duplication.

2. **Intersection Observer**: Implement infinite scroll with Intersection Observer API for better performance than scroll event listeners.

3. **WebSocket for Real-time**: Use WebSocket connections for real-time updates with polling fallback for reliability.

4. **Notification Banner**: Show "new tweets available" banner instead of auto-inserting to prevent jarring user experience.

5. **Multi-layer Caching**: Implement memory cache for immediate access, IndexedDB for persistence, and HTTP cache for API responses.

6. **Virtual Scrolling**: Use virtual scrolling for feeds with thousands of items to maintain 60fps performance.

7. **Normalized State**: Store tweets in normalized format (by ID) in state management to avoid data duplication and enable efficient updates.

8. **Optimistic Updates**: Apply likes/retweets optimistically while API request is in flight for instant user feedback.

9. **Error Boundaries**: Implement error boundaries to isolate failing tweet cards from crashing entire feed.

10. **Performance Monitoring**: Track Core Web Vitals (LCP, FID, CLS) and feed-specific metrics (time to first tweet, scroll jank).

## Red Flags to Avoid

- Using offset pagination for real-time feeds
- Auto-inserting new tweets without user control
- Not implementing proper error handling for failed loads
- Forgetting to unsubscribe from WebSocket connections
- Not using trackBy in *ngFor for tweet lists
- Loading all tweets into DOM without virtualization
- Not caching API responses
- Blocking main thread with expensive operations
- Not handling race conditions in pagination
- Forgetting to implement loading and error states

## Trade-offs and Decisions

**WebSocket vs Polling:**
- WebSocket: Lower latency, less server load, more complex
- Polling: Simpler, more reliable, higher latency

**Virtual Scroll vs Regular:**
- Virtual: Better performance, more complex, breaks some browser features
- Regular: Simpler, works with all browser features, performance issues at scale

**Optimistic UI vs Wait for Response:**
- Optimistic: Better UX, risk of inconsistency
- Wait: More reliable, slower perceived performance

**Client-side vs Server-side Rendering:**
- Client: Better interactivity, SEO challenges
- Server: Better initial load, more complex architecture

## Interview Talking Points

- Explain cursor-based pagination advantages
- Discuss real-time update strategies and trade-offs
- Describe caching layers and invalidation strategy
- Explain virtual scrolling performance benefits
- Discuss state normalization for tweet data
- Explain optimistic UI update patterns
- Describe error handling and retry strategies
- Discuss performance optimization techniques
- Explain offline support with Service Workers
- Describe monitoring and observability approach
