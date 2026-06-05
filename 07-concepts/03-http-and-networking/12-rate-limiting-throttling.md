# Rate Limiting and Throttling

## The Idea

**In plain English:** Rate limiting and throttling are ways to control how often something (like a request to a website's data service) can happen within a set amount of time. A "request" is simply your app asking a server for information — rate limiting puts a cap on how many of those asks are allowed, and throttling spaces them out so they don't all happen at once.

**Real-world analogy:** Imagine a theme park ride that only lets 10 people board every 5 minutes, and a staff member controls a turnstile to enforce it. If a big crowd rushes in, the turnstile slows everyone down to an orderly pace — and if the 10-person limit is already hit, latecomers are told to wait until the next window opens.

- The ride's 10-person limit = the rate limit (maximum requests allowed per time window)
- The 5-minute boarding window = the time window in which requests are counted
- The turnstile slowing the crowd = throttling (spreading requests out evenly over time)

---

## Overview

Rate limiting and throttling are techniques to control the frequency of operations, typically API requests. While servers implement rate limits to protect resources, frontend applications must handle these limits gracefully and implement client-side strategies to optimize request patterns and provide better user experiences.

## Core Concepts

```
Rate Limiting vs Throttling:

Rate Limiting: Hard cap on requests per time window
┌──────────────────────────────────────────┐
│  Window: 1 minute                         │
│  Limit: 100 requests                      │
│                                           │
│  Request 1-100: ✓ Allowed                │
│  Request 101+:  ✗ Blocked (429 Error)    │
└──────────────────────────────────────────┘

Throttling: Spread requests evenly over time
┌──────────────────────────────────────────┐
│  100 requests/minute = 1 request/600ms    │
│                                           │
│  0ms: Request 1  ✓                        │
│  600ms: Request 2 ✓                       │
│  1200ms: Request 3 ✓                      │
│  (Requests queued if too fast)            │
└──────────────────────────────────────────┘
```

### Key Terms

- **Rate Limit**: Maximum number of requests allowed in a time window
- **Throttle**: Slow down request rate to acceptable level
- **Debounce**: Delay execution until activity stops
- **Backoff**: Progressively increase delay after failures
- **Queue**: Hold requests for later execution

## Client-Side Rate Limiting

### Basic Rate Limiter Implementation

```typescript
// React custom hook for rate limiting
import { useRef, useCallback } from 'react';

interface RateLimiterOptions {
  maxRequests: number;
  windowMs: number;
}

export const useRateLimiter = (options: RateLimiterOptions) => {
  const requestTimestamps = useRef<number[]>([]);

  const canMakeRequest = useCallback((): boolean => {
    const now = Date.now();
    const windowStart = now - options.windowMs;

    // Remove timestamps outside the current window
    requestTimestamps.current = requestTimestamps.current.filter(
      timestamp => timestamp > windowStart
    );

    return requestTimestamps.current.length < options.maxRequests;
  }, [options.maxRequests, options.windowMs]);

  const recordRequest = useCallback((): void => {
    requestTimestamps.current.push(Date.now());
  }, []);

  const getRemainingRequests = useCallback((): number => {
    const now = Date.now();
    const windowStart = now - options.windowMs;
    
    requestTimestamps.current = requestTimestamps.current.filter(
      timestamp => timestamp > windowStart
    );

    return options.maxRequests - requestTimestamps.current.length;
  }, [options.maxRequests, options.windowMs]);

  const getResetTime = useCallback((): number => {
    if (requestTimestamps.current.length === 0) return 0;
    
    const oldest = Math.min(...requestTimestamps.current);
    return oldest + options.windowMs;
  }, [options.windowMs]);

  const waitForAvailability = useCallback((): Promise<void> => {
    return new Promise((resolve) => {
      const checkAvailability = () => {
        if (canMakeRequest()) {
          resolve();
        } else {
          const resetTime = getResetTime();
          const waitTime = resetTime - Date.now();
          setTimeout(checkAvailability, Math.max(waitTime, 100));
        }
      };

      checkAvailability();
    });
  }, [canMakeRequest, getResetTime]);

  return {
    canMakeRequest,
    recordRequest,
    getRemainingRequests,
    getResetTime,
    waitForAvailability
  };
};

// Usage example
const SearchComponent = () => {
  const [results, setResults] = useState([]);
  const [error, setError] = useState<string | null>(null);

  // Limit to 10 requests per 10 seconds
  const rateLimiter = useRateLimiter({
    maxRequests: 10,
    windowMs: 10000
  });

  const handleSearch = async (query: string) => {
    if (!rateLimiter.canMakeRequest()) {
      const remaining = rateLimiter.getRemainingRequests();
      const resetTime = new Date(rateLimiter.getResetTime());
      
      setError(
        `Rate limit exceeded. ${remaining} requests remaining. ` +
        `Resets at ${resetTime.toLocaleTimeString()}`
      );
      return;
    }

    try {
      rateLimiter.recordRequest();
      const response = await fetch(`/api/search?q=${query}`);
      const data = await response.json();
      setResults(data);
      setError(null);
    } catch (err) {
      setError('Search failed');
    }
  };

  return (
    <div>
      <input 
        type="text" 
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      {error && <div className="error">{error}</div>}
      <div>Results: {results.length}</div>
    </div>
  );
};
```

**Angular Rate Limiter Service:**

```typescript
// services/rate-limiter.service.ts
import { Injectable } from '@angular/core';
import { Observable, throwError, timer } from 'rxjs';
import { mergeMap, retryWhen, tap } from 'rxjs/operators';

export interface RateLimitConfig {
  maxRequests: number;
  windowMs: number;
}

@Injectable({
  providedIn: 'root'
})
export class RateLimiterService {
  private requestTimestamps: Map<string, number[]> = new Map();

  canMakeRequest(key: string, config: RateLimitConfig): boolean {
    const now = Date.now();
    const windowStart = now - config.windowMs;

    let timestamps = this.requestTimestamps.get(key) || [];
    
    // Remove old timestamps
    timestamps = timestamps.filter(ts => ts > windowStart);
    this.requestTimestamps.set(key, timestamps);

    return timestamps.length < config.maxRequests;
  }

  recordRequest(key: string): void {
    const timestamps = this.requestTimestamps.get(key) || [];
    timestamps.push(Date.now());
    this.requestTimestamps.set(key, timestamps);
  }

  getRemainingRequests(key: string, config: RateLimitConfig): number {
    const now = Date.now();
    const windowStart = now - config.windowMs;

    const timestamps = (this.requestTimestamps.get(key) || [])
      .filter(ts => ts > windowStart);

    return config.maxRequests - timestamps.length;
  }

  getResetTime(key: string, config: RateLimitConfig): number {
    const timestamps = this.requestTimestamps.get(key) || [];
    if (timestamps.length === 0) return 0;

    const oldest = Math.min(...timestamps);
    return oldest + config.windowMs;
  }

  waitForAvailability(key: string, config: RateLimitConfig): Observable<void> {
    return new Observable(observer => {
      const check = () => {
        if (this.canMakeRequest(key, config)) {
          observer.next();
          observer.complete();
        } else {
          const resetTime = this.getResetTime(key, config);
          const waitTime = resetTime - Date.now();
          setTimeout(check, Math.max(waitTime, 100));
        }
      };

      check();
    });
  }
}

// Usage in component
@Component({
  selector: 'app-search',
  template: `
    <input 
      (input)="onSearch($event)"
      placeholder="Search..."
    />
    <div *ngIf="error" class="error">{{ error }}</div>
    <div>Results: {{ results.length }}</div>
  `
})
export class SearchComponent {
  results: any[] = [];
  error: string | null = null;

  private readonly rateLimitConfig: RateLimitConfig = {
    maxRequests: 10,
    windowMs: 10000
  };

  constructor(
    private rateLimiter: RateLimiterService,
    private searchService: SearchService
  ) {}

  onSearch(event: Event): void {
    const query = (event.target as HTMLInputElement).value;

    if (!this.rateLimiter.canMakeRequest('search', this.rateLimitConfig)) {
      const remaining = this.rateLimiter.getRemainingRequests(
        'search',
        this.rateLimitConfig
      );
      this.error = `Rate limit exceeded. ${remaining} requests remaining.`;
      return;
    }

    this.rateLimiter.recordRequest('search');
    
    this.searchService.search(query).subscribe({
      next: (results) => {
        this.results = results;
        this.error = null;
      },
      error: (err) => {
        this.error = 'Search failed';
      }
    });
  }
}
```

## Request Throttling

### Throttle Implementation

```typescript
// React throttle hook
import { useRef, useCallback, useEffect } from 'react';

export const useThrottle = <T extends (...args: any[]) => any>(
  callback: T,
  delay: number
): T => {
  const lastRun = useRef<number>(Date.now());
  const timeoutRef = useRef<NodeJS.Timeout | null>(null);

  useEffect(() => {
    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, []);

  return useCallback(
    (...args: Parameters<T>) => {
      const now = Date.now();
      const timeSinceLastRun = now - lastRun.current;

      if (timeSinceLastRun >= delay) {
        callback(...args);
        lastRun.current = now;
      } else {
        if (timeoutRef.current) {
          clearTimeout(timeoutRef.current);
        }

        timeoutRef.current = setTimeout(() => {
          callback(...args);
          lastRun.current = Date.now();
        }, delay - timeSinceLastRun);
      }
    },
    [callback, delay]
  ) as T;
};

// Usage
const ScrollTracker = () => {
  const [scrollPosition, setScrollPosition] = useState(0);

  const handleScroll = useThrottle(() => {
    const position = window.scrollY;
    setScrollPosition(position);
    
    // This API call happens at most once per second
    fetch('/api/track-scroll', {
      method: 'POST',
      body: JSON.stringify({ position })
    });
  }, 1000); // Throttle to once per second

  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);

  return <div>Scroll position: {scrollPosition}</div>;
};
```

**Advanced Throttle with Leading and Trailing:**

```typescript
interface ThrottleOptions {
  leading?: boolean;  // Call on the leading edge
  trailing?: boolean; // Call on the trailing edge
}

export const useAdvancedThrottle = <T extends (...args: any[]) => any>(
  callback: T,
  delay: number,
  options: ThrottleOptions = { leading: true, trailing: true }
): T => {
  const lastRun = useRef<number>(0);
  const lastArgs = useRef<Parameters<T> | null>(null);
  const timeoutRef = useRef<NodeJS.Timeout | null>(null);

  useEffect(() => {
    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, []);

  return useCallback(
    (...args: Parameters<T>) => {
      const now = Date.now();
      const timeSinceLastRun = now - lastRun.current;

      lastArgs.current = args;

      const invokeCallback = () => {
        if (lastArgs.current) {
          callback(...lastArgs.current);
          lastRun.current = Date.now();
          lastArgs.current = null;
        }
      };

      const shouldInvokeLeading = 
        options.leading && timeSinceLastRun >= delay;

      if (shouldInvokeLeading) {
        invokeCallback();
      }

      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }

      if (options.trailing) {
        timeoutRef.current = setTimeout(() => {
          if (Date.now() - lastRun.current >= delay) {
            invokeCallback();
          }
        }, delay);
      }
    },
    [callback, delay, options.leading, options.trailing]
  ) as T;
};
```

## Debouncing

### Debounce Implementation

```typescript
// React debounce hook
import { useEffect, useRef, useCallback } from 'react';

export const useDebounce = <T extends (...args: any[]) => any>(
  callback: T,
  delay: number
): T => {
  const timeoutRef = useRef<NodeJS.Timeout | null>(null);

  useEffect(() => {
    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, []);

  return useCallback(
    (...args: Parameters<T>) => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }

      timeoutRef.current = setTimeout(() => {
        callback(...args);
      }, delay);
    },
    [callback, delay]
  ) as T;
};

// Debounce with immediate option
interface DebounceOptions {
  immediate?: boolean; // Call on leading edge, then debounce
}

export const useAdvancedDebounce = <T extends (...args: any[]) => any>(
  callback: T,
  delay: number,
  options: DebounceOptions = {}
): T => {
  const timeoutRef = useRef<NodeJS.Timeout | null>(null);
  const immediateRef = useRef<boolean>(true);

  useEffect(() => {
    return () => {
      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }
    };
  }, []);

  return useCallback(
    (...args: Parameters<T>) => {
      const callNow = options.immediate && immediateRef.current;

      if (timeoutRef.current) {
        clearTimeout(timeoutRef.current);
      }

      if (callNow) {
        callback(...args);
        immediateRef.current = false;
      }

      timeoutRef.current = setTimeout(() => {
        if (!options.immediate) {
          callback(...args);
        }
        immediateRef.current = true;
      }, delay);
    },
    [callback, delay, options.immediate]
  ) as T;
};

// Usage: Search input with debounce
const SearchInput = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const performSearch = async (searchQuery: string) => {
    if (!searchQuery) {
      setResults([]);
      return;
    }

    try {
      const response = await fetch(`/api/search?q=${searchQuery}`);
      const data = await response.json();
      setResults(data);
    } catch (error) {
      console.error('Search failed:', error);
    }
  };

  // Debounce search by 500ms
  const debouncedSearch = useDebounce(performSearch, 500);

  const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value); // Will only call after user stops typing for 500ms
  };

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={handleInputChange}
        placeholder="Search..."
      />
      <ul>
        {results.map((result: any) => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
};
```

**Angular Debounce Operator:**

```typescript
// component using RxJS debounce
import { Component, OnInit } from '@angular/core';
import { FormControl } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-search',
  template: `
    <input 
      [formControl]="searchControl"
      placeholder="Search..."
    />
    <ul>
      <li *ngFor="let result of results">{{ result.title }}</li>
    </ul>
  `
})
export class SearchComponent implements OnInit {
  searchControl = new FormControl('');
  results: any[] = [];

  constructor(private searchService: SearchService) {}

  ngOnInit() {
    this.searchControl.valueChanges
      .pipe(
        debounceTime(500), // Wait 500ms after user stops typing
        distinctUntilChanged(), // Only if the value actually changed
        switchMap(query => 
          query ? this.searchService.search(query) : of([])
        )
      )
      .subscribe(results => {
        this.results = results;
      });
  }
}
```

## Exponential Backoff

### Retry with Exponential Backoff

```typescript
// React hook for exponential backoff
interface BackoffOptions {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  factor: number;
  jitter?: boolean;
}

export const useExponentialBackoff = () => {
  const executeWithBackoff = async <T,>(
    operation: () => Promise<T>,
    options: BackoffOptions = {
      maxRetries: 5,
      initialDelay: 1000,
      maxDelay: 30000,
      factor: 2,
      jitter: true
    }
  ): Promise<T> => {
    let lastError: Error;
    let delay = options.initialDelay;

    for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error as Error;

        // Don't retry on last attempt
        if (attempt === options.maxRetries) {
          break;
        }

        // Check if error is retryable
        if (!isRetryableError(error)) {
          throw error;
        }

        // Calculate delay with exponential backoff
        let waitTime = Math.min(delay, options.maxDelay);

        // Add jitter to prevent thundering herd
        if (options.jitter) {
          waitTime = waitTime * (0.5 + Math.random() * 0.5);
        }

        console.log(
          `Attempt ${attempt + 1} failed. Retrying in ${waitTime}ms...`
        );

        await new Promise(resolve => setTimeout(resolve, waitTime));

        // Increase delay for next attempt
        delay *= options.factor;
      }
    }

    throw new Error(
      `Operation failed after ${options.maxRetries} retries: ${lastError.message}`
    );
  };

  const isRetryableError = (error: any): boolean => {
    // Retry on network errors or 5xx server errors
    if (error.name === 'TypeError') return true; // Network error
    if (error.response?.status >= 500) return true; // Server error
    if (error.response?.status === 429) return true; // Rate limit
    return false;
  };

  return { executeWithBackoff };
};

// Usage
const DataFetcher = ({ userId }: { userId: string }) => {
  const [data, setData] = useState(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);
  const { executeWithBackoff } = useExponentialBackoff();

  const fetchData = async () => {
    setLoading(true);
    setError(null);

    try {
      const result = await executeWithBackoff(
        async () => {
          const response = await fetch(`/api/users/${userId}`);
          
          if (!response.ok) {
            const error: any = new Error('Request failed');
            error.response = response;
            throw error;
          }

          return response.json();
        },
        {
          maxRetries: 5,
          initialDelay: 1000,
          maxDelay: 30000,
          factor: 2,
          jitter: true
        }
      );

      setData(result);
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!data) return null;

  return <div>{JSON.stringify(data)}</div>;
};
```

**Angular Retry Strategy:**

```typescript
// services/retry.service.ts
import { Injectable } from '@angular/core';
import { Observable, throwError, timer } from 'rxjs';
import { mergeMap, retryWhen, tap } from 'rxjs/operators';

export interface RetryConfig {
  maxRetries: number;
  initialDelay: number;
  maxDelay: number;
  factor: number;
  jitter: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class RetryService {
  exponentialBackoff(config: RetryConfig) {
    return <T>(source: Observable<T>) =>
      source.pipe(
        retryWhen(errors =>
          errors.pipe(
            mergeMap((error, index) => {
              const attempt = index + 1;

              // Check if we've exceeded max retries
              if (attempt > config.maxRetries) {
                return throwError(() => error);
              }

              // Check if error is retryable
              if (!this.isRetryable(error)) {
                return throwError(() => error);
              }

              // Calculate delay
              let delay = Math.min(
                config.initialDelay * Math.pow(config.factor, index),
                config.maxDelay
              );

              // Add jitter
              if (config.jitter) {
                delay = delay * (0.5 + Math.random() * 0.5);
              }

              console.log(`Retry attempt ${attempt} after ${delay}ms`);

              return timer(delay);
            })
          )
        )
      );
  }

  private isRetryable(error: any): boolean {
    // Network errors
    if (error.status === 0) return true;
    
    // Server errors
    if (error.status >= 500) return true;
    
    // Rate limiting
    if (error.status === 429) return true;

    return false;
  }
}

// Usage in service
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(
    private http: HttpClient,
    private retry: RetryService
  ) {}

  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      this.retry.exponentialBackoff({
        maxRetries: 5,
        initialDelay: 1000,
        maxDelay: 30000,
        factor: 2,
        jitter: true
      })
    );
  }
}
```

## Request Queue

### Queue Implementation for Rate Limiting

```typescript
// React request queue with rate limiting
interface QueuedRequest<T> {
  id: string;
  execute: () => Promise<T>;
  resolve: (value: T) => void;
  reject: (error: any) => void;
  priority?: number;
}

class RequestQueue {
  private queue: QueuedRequest<any>[] = [];
  private processing: boolean = false;
  private requestsPerSecond: number;
  private delayBetweenRequests: number;

  constructor(requestsPerSecond: number = 10) {
    this.requestsPerSecond = requestsPerSecond;
    this.delayBetweenRequests = 1000 / requestsPerSecond;
  }

  enqueue<T>(
    execute: () => Promise<T>,
    priority: number = 0
  ): Promise<T> {
    return new Promise((resolve, reject) => {
      const request: QueuedRequest<T> = {
        id: Math.random().toString(36).substr(2, 9),
        execute,
        resolve,
        reject,
        priority
      };

      this.queue.push(request);
      
      // Sort by priority (higher priority first)
      this.queue.sort((a, b) => (b.priority || 0) - (a.priority || 0));

      this.processQueue();
    });
  }

  private async processQueue(): Promise<void> {
    if (this.processing || this.queue.length === 0) {
      return;
    }

    this.processing = true;

    while (this.queue.length > 0) {
      const request = this.queue.shift()!;

      try {
        const result = await request.execute();
        request.resolve(result);
      } catch (error) {
        request.reject(error);
      }

      // Wait before processing next request
      if (this.queue.length > 0) {
        await new Promise(resolve => 
          setTimeout(resolve, this.delayBetweenRequests)
        );
      }
    }

    this.processing = false;
  }

  getQueueLength(): number {
    return this.queue.length;
  }

  clear(): void {
    this.queue.forEach(request => 
      request.reject(new Error('Queue cleared'))
    );
    this.queue = [];
  }
}

// React hook for request queue
export const useRequestQueue = (requestsPerSecond: number = 10) => {
  const queueRef = useRef<RequestQueue>(new RequestQueue(requestsPerSecond));

  const enqueueRequest = useCallback(
    <T,>(execute: () => Promise<T>, priority?: number): Promise<T> => {
      return queueRef.current.enqueue(execute, priority);
    },
    []
  );

  const getQueueLength = useCallback((): number => {
    return queueRef.current.getQueueLength();
  }, []);

  const clearQueue = useCallback((): void => {
    queueRef.current.clear();
  }, []);

  return {
    enqueueRequest,
    getQueueLength,
    clearQueue
  };
};

// Usage
const BulkUploader = () => {
  const [files, setFiles] = useState<File[]>([]);
  const [uploadStatus, setUploadStatus] = useState<Record<string, string>>({});
  
  // Limit to 2 uploads per second
  const { enqueueRequest, getQueueLength } = useRequestQueue(2);

  const uploadFile = async (file: File) => {
    const formData = new FormData();
    formData.append('file', file);

    const response = await fetch('/api/upload', {
      method: 'POST',
      body: formData
    });

    if (!response.ok) {
      throw new Error('Upload failed');
    }

    return response.json();
  };

  const handleUpload = async () => {
    const uploads = files.map(file =>
      enqueueRequest(
        () => uploadFile(file),
        file.size // Larger files get higher priority
      ).then(() => {
        setUploadStatus(prev => ({
          ...prev,
          [file.name]: 'completed'
        }));
      }).catch(() => {
        setUploadStatus(prev => ({
          ...prev,
          [file.name]: 'failed'
        }));
      })
    );

    await Promise.all(uploads);
  };

  return (
    <div>
      <input
        type="file"
        multiple
        onChange={(e) => setFiles(Array.from(e.target.files || []))}
      />
      <button onClick={handleUpload}>Upload All</button>
      <div>Queue length: {getQueueLength()}</div>
      <ul>
        {files.map(file => (
          <li key={file.name}>
            {file.name}: {uploadStatus[file.name] || 'pending'}
          </li>
        ))}
      </ul>
    </div>
  );
};
```

## Handling 429 Rate Limit Responses

### Retry-After Header Handling

```typescript
// React service to handle 429 responses
class RateLimitedAPIClient {
  private baseURL: string;
  private retryQueue: Map<string, Promise<any>> = new Map();

  constructor(baseURL: string) {
    this.baseURL = baseURL;
  }

  async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${this.baseURL}${endpoint}`;

    try {
      const response = await fetch(url, options);

      if (response.status === 429) {
        return this.handleRateLimitResponse<T>(url, options, response);
      }

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return response.json();
    } catch (error) {
      throw error;
    }
  }

  private async handleRateLimitResponse<T>(
    url: string,
    options: RequestInit,
    response: Response
  ): Promise<T> {
    // Check if already retrying this request
    const existingRetry = this.retryQueue.get(url);
    if (existingRetry) {
      return existingRetry;
    }

    // Extract retry delay from headers
    const retryAfter = this.getRetryAfterDelay(response);
    const rateLimitReset = this.getRateLimitReset(response);
    
    const delay = retryAfter || rateLimitReset || 60000; // Default to 1 minute

    console.warn(
      `Rate limited. Retrying after ${delay}ms`,
      `Remaining: ${response.headers.get('X-RateLimit-Remaining')}`,
      `Limit: ${response.headers.get('X-RateLimit-Limit')}`
    );

    // Create retry promise
    const retryPromise = new Promise<T>((resolve, reject) => {
      setTimeout(async () => {
        try {
          this.retryQueue.delete(url);
          const result = await this.request<T>(url, options);
          resolve(result);
        } catch (error) {
          reject(error);
        }
      }, delay);
    });

    this.retryQueue.set(url, retryPromise);
    return retryPromise;
  }

  private getRetryAfterDelay(response: Response): number | null {
    const retryAfter = response.headers.get('Retry-After');
    if (!retryAfter) return null;

    // Retry-After can be seconds or HTTP date
    const seconds = parseInt(retryAfter, 10);
    if (!isNaN(seconds)) {
      return seconds * 1000; // Convert to milliseconds
    }

    // Parse as date
    const date = new Date(retryAfter);
    if (!isNaN(date.getTime())) {
      return Math.max(0, date.getTime() - Date.now());
    }

    return null;
  }

  private getRateLimitReset(response: Response): number | null {
    const reset = response.headers.get('X-RateLimit-Reset');
    if (!reset) return null;

    const resetTime = parseInt(reset, 10) * 1000; // Unix timestamp to ms
    return Math.max(0, resetTime - Date.now());
  }
}

// Usage
const api = new RateLimitedAPIClient('https://api.example.com');

const fetchUser = async (id: string) => {
  try {
    const user = await api.request(`/users/${id}`);
    console.log('User fetched:', user);
  } catch (error) {
    console.error('Failed to fetch user:', error);
  }
};
```

## Comparison: Throttle vs Debounce vs Rate Limit

```
User Types: a b c d e f g h i j k
Time:       0 1 2 3 4 5 6 7 8 9 10 (seconds)

Debounce (2s delay):
Execution:                      ✓   (only at end, after 2s of no input)

Throttle (every 2s):
Execution:  ✓     ✓     ✓     ✓    (regularly, every 2 seconds)

Rate Limit (3 calls per 5s window):
Execution:  ✓ ✓ ✓         ✓ ✓ ✓    (3 calls, wait, 3 more calls)
```

## Common Mistakes

### 1. Not Respecting Retry-After Header

```typescript
// ❌ BAD: Ignoring server's retry instructions
const fetchData = async () => {
  try {
    return await fetch('/api/data');
  } catch (error: any) {
    if (error.status === 429) {
      await new Promise(resolve => setTimeout(resolve, 1000)); // Fixed delay
      return fetchData(); // Retry immediately
    }
  }
};

// ✅ GOOD: Respecting Retry-After header
const fetchData = async () => {
  const response = await fetch('/api/data');
  
  if (response.status === 429) {
    const retryAfter = response.headers.get('Retry-After');
    const delay = retryAfter ? parseInt(retryAfter) * 1000 : 60000;
    
    await new Promise(resolve => setTimeout(resolve, delay));
    return fetchData();
  }
  
  return response.json();
};
```

### 2. No Exponential Backoff

```typescript
// ❌ BAD: Fixed retry delay
const retry = async (fn: () => Promise<any>, maxRetries: number) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      await new Promise(resolve => setTimeout(resolve, 1000)); // Always 1s
    }
  }
};

// ✅ GOOD: Exponential backoff with jitter
const retry = async (fn: () => Promise<any>, maxRetries: number) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      const delay = Math.min(1000 * Math.pow(2, i), 30000);
      const jitter = delay * (0.5 + Math.random() * 0.5);
      await new Promise(resolve => setTimeout(resolve, jitter));
    }
  }
};
```

### 3. Debouncing Everything

```typescript
// ❌ BAD: Debouncing critical operations
const saveDocument = useDebounce(async (content: string) => {
  await api.post('/documents', { content });
}, 2000);

// User types, closes tab → data lost!

// ✅ GOOD: Immediate save + debounced updates
const saveDocument = async (content: string, immediate: boolean = false) => {
  if (immediate) {
    await api.post('/documents', { content });
  } else {
    debouncedSave(content);
  }
};

const debouncedSave = useDebounce(saveDocument, 2000);
```

## Best Practices

### 1. Progressive Enhancement

```typescript
// Start with no rate limiting, add as needed
const apiClient = {
  // Development: No rate limiting
  dev: new APIClient(),
  
  // Production: With rate limiting
  prod: new RateLimitedAPIClient({
    maxRequests: 100,
    windowMs: 60000
  })
};

const api = process.env.NODE_ENV === 'production' ? apiClient.prod : apiClient.dev;
```

### 2. User Feedback

```typescript
// Show users rate limit status
const RateLimitIndicator = () => {
  const { getRemainingRequests, getResetTime } = useRateLimiter({
    maxRequests: 100,
    windowMs: 60000
  });

  const remaining = getRemainingRequests();
  const resetTime = new Date(getResetTime());

  if (remaining < 10) {
    return (
      <div className="rate-limit-warning">
        <p>You have {remaining} requests remaining</p>
        <p>Resets at {resetTime.toLocaleTimeString()}</p>
      </div>
    );
  }

  return null;
};
```

### 3. Graceful Degradation

```typescript
// Fallback to cached data when rate limited
const fetchWithCache = async (url: string) => {
  try {
    const response = await fetch(url);
    
    if (response.status === 429) {
      console.warn('Rate limited, using cached data');
      return getCachedData(url);
    }
    
    const data = await response.json();
    setCachedData(url, data);
    return data;
  } catch (error) {
    return getCachedData(url);
  }
};
```

## When to Use Each Technique

### Debounce
- **Use for**: Search inputs, form validation, resize events
- **Don't use for**: Critical operations, real-time updates

### Throttle
- **Use for**: Scroll events, mouse movement, analytics tracking
- **Don't use for**: User input that needs immediate feedback

### Rate Limiting
- **Use for**: API calls, bulk operations, external service calls
- **Don't use for**: UI interactions, local operations

### Exponential Backoff
- **Use for**: Retrying failed requests, handling temporary errors
- **Don't use for**: Permanent errors, user-initiated retries

## Interview Questions

### 1. What's the difference between throttling and debouncing?
**Answer**: Throttling limits function execution to once per time period (e.g., once per second), guaranteeing regular execution. Debouncing delays execution until after activity stops (e.g., wait 500ms after last keystroke). Throttle is used for regular sampling (scroll events), while debounce is used for waiting until user finishes an action (search input).

### 2. How do you implement exponential backoff?
**Answer**: Start with an initial delay (e.g., 1 second). On each retry, multiply the delay by a factor (e.g., 2). Cap at a maximum delay (e.g., 30 seconds). Add jitter (random variation) to prevent thundering herd problem. Stop after max retries. Formula: `delay = min(initialDelay * factor^attempt * (0.5 + random(0.5)), maxDelay)`.

### 3. What information do rate limit headers provide?
**Answer**: `X-RateLimit-Limit`: total allowed requests, `X-RateLimit-Remaining`: remaining requests, `X-RateLimit-Reset`: Unix timestamp when limit resets, `Retry-After`: seconds/date when to retry after 429 response. These help clients make informed decisions about request timing.

### 4. How would you handle rate limiting in a bulk upload scenario?
**Answer**: Use a request queue with controlled concurrency. Process items in batches respecting rate limits. Implement exponential backoff for 429 responses. Show progress to users. Allow pause/resume. Save state to handle browser refresh. Use priority queue for important items.

### 5. Why add jitter to exponential backoff?
**Answer**: Jitter prevents the thundering herd problem where many clients retry simultaneously after a failure. By adding random variation to retry delays, requests are spread out over time, preventing synchronized spikes that could overwhelm a recovering server.

## Key Takeaways

1. **Debounce for Completion**: Wait until user finishes an action (search, form input).
2. **Throttle for Sampling**: Regular execution of high-frequency events (scroll, resize).
3. **Rate Limit for Protection**: Control request frequency to protect APIs and resources.
4. **Exponential Backoff for Reliability**: Progressively increase delays when retrying failed requests.
5. **Respect Server Limits**: Always honor Retry-After headers and rate limit responses.
6. **Add Jitter**: Prevent thundering herd with randomized delays.
7. **Provide Feedback**: Show users rate limit status and estimated wait times.
8. **Queue Requests**: Use request queues for bulk operations with rate limits.

## Resources

- [MDN - Debouncing and Throttling](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout#debouncing)
- [Google Cloud - Exponential Backoff](https://cloud.google.com/iot/docs/how-tos/exponential-backoff)
- [RFC 6585 - 429 Too Many Requests](https://tools.ietf.org/html/rfc6585#section-4)
- [AWS - Error Retries and Exponential Backoff](https://docs.aws.amazon.com/general/latest/gr/api-retries.html)
- [Lodash Debounce/Throttle](https://lodash.com/docs/)
- [RxJS Debounce/Throttle Operators](https://rxjs.dev/api/operators/debounceTime)
