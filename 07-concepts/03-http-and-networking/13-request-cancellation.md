# Request Cancellation

## The Idea

**In plain English:** Request cancellation is the ability to tell your browser "stop waiting for that answer I asked for — I don't need it anymore." A request is a message your app sends to a server asking for data; cancellation means you can take that message back before the server finishes replying.

**Real-world analogy:** Imagine you call a pizza place and order a large pepperoni. While waiting, you change your mind and call back to cancel the order before it goes in the oven. The kitchen stops making your pizza so they don't waste ingredients or time.

- The phone call to order = the HTTP request sent to the server
- Calling back to cancel = calling `AbortController.abort()`
- The kitchen stopping work = the browser dropping the request and freeing up resources

---

## Overview

Request cancellation allows you to abort ongoing HTTP requests when they're no longer needed. This prevents wasted bandwidth, improves performance, avoids race conditions, and provides better user experience. Modern browsers support the AbortController API for request cancellation across fetch, XHR, and other async operations.

## Why Cancel Requests

```
Without Cancellation:
User searches: "React" → Request 1 sent
User searches: "React Hook" → Request 2 sent
Request 2 completes → Shows results
Request 1 completes → Overwrites with old results ❌

With Cancellation:
User searches: "React" → Request 1 sent
User searches: "React Hook" → Request 1 CANCELED, Request 2 sent
Request 2 completes → Shows correct results ✓
```

### Benefits

1. **Prevent Race Conditions**: Cancel outdated requests
2. **Save Bandwidth**: Don't download unnecessary data
3. **Improve Performance**: Free up network resources
4. **Better UX**: Faster response to user actions
5. **Memory Management**: Prevent memory leaks from unmounted components

## AbortController Basics

### Basic Usage

```typescript
// React component with request cancellation
import { useState, useEffect } from 'react';

const UserProfile = ({ userId }: { userId: string }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // Create AbortController for this request
    const abortController = new AbortController();
    const signal = abortController.signal;

    const fetchUser = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await fetch(`/api/users/${userId}`, {
          signal // Pass signal to fetch
        });

        // Check if request was aborted
        if (signal.aborted) {
          console.log('Request was aborted');
          return;
        }

        if (!response.ok) {
          throw new Error('Failed to fetch user');
        }

        const data = await response.json();
        setUser(data);
      } catch (err: any) {
        // Ignore abort errors
        if (err.name === 'AbortError') {
          console.log('Fetch aborted');
          return;
        }
        
        setError(err.message);
      } finally {
        if (!signal.aborted) {
          setLoading(false);
        }
      }
    };

    fetchUser();

    // Cleanup: Cancel request when component unmounts or userId changes
    return () => {
      abortController.abort();
      console.log('Aborting fetch for userId:', userId);
    };
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
};
```

**Angular with RxJS:**

```typescript
// service with cancellable requests
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUser(userId: string, cancelSignal: Subject<void>): Observable<User> {
    return this.http
      .get<User>(`/api/users/${userId}`)
      .pipe(takeUntil(cancelSignal));
  }
}

// component using cancellation
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';

@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">Error: {{ error }}</div>
    <div *ngIf="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit, OnDestroy {
  user: User | null = null;
  loading = false;
  error: string | null = null;
  
  private destroy$ = new Subject<void>();

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.loadUser('123');
  }

  ngOnDestroy() {
    // Cancel all pending requests
    this.destroy$.next();
    this.destroy$.complete();
  }

  loadUser(userId: string) {
    this.loading = true;
    this.error = null;

    this.userService
      .getUser(userId, this.destroy$)
      .subscribe({
        next: (user) => {
          this.user = user;
          this.loading = false;
        },
        error: (err) => {
          this.error = err.message;
          this.loading = false;
        }
      });
  }
}
```

## Custom Hooks for Cancellation

### useAbortController Hook

```typescript
// React hook for managing AbortControllers
import { useEffect, useRef } from 'react';

export const useAbortController = () => {
  const abortControllerRef = useRef<AbortController | null>(null);

  // Create new controller
  const createController = (): AbortController => {
    // Abort previous controller if exists
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    const controller = new AbortController();
    abortControllerRef.current = controller;
    return controller;
  };

  // Get current controller or create new one
  const getController = (): AbortController => {
    if (!abortControllerRef.current || abortControllerRef.current.signal.aborted) {
      return createController();
    }
    return abortControllerRef.current;
  };

  // Abort current controller
  const abort = (reason?: string) => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort(reason);
      console.log('Request aborted:', reason || 'User cancelled');
    }
  };

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      abort('Component unmounted');
    };
  }, []);

  return {
    createController,
    getController,
    abort,
    signal: abortControllerRef.current?.signal
  };
};

// Usage
const SearchComponent = () => {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const { createController } = useAbortController();

  const handleSearch = async (searchQuery: string) => {
    if (!searchQuery) {
      setResults([]);
      return;
    }

    // Create new controller (aborts previous search)
    const controller = createController();

    try {
      const response = await fetch(`/api/search?q=${searchQuery}`, {
        signal: controller.signal
      });

      const data = await response.json();
      setResults(data);
    } catch (error: any) {
      if (error.name === 'AbortError') {
        console.log('Search cancelled');
        return;
      }
      console.error('Search failed:', error);
    }
  };

  const debouncedSearch = useDebounce(handleSearch, 300);

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          debouncedSearch(e.target.value);
        }}
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

### useCancellableRequest Hook

```typescript
// Complete hook for cancellable requests
import { useState, useCallback, useRef, useEffect } from 'react';

interface RequestState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

interface RequestOptions extends RequestInit {
  skip?: boolean; // Skip automatic execution
}

export const useCancellableRequest = <T,>(
  url: string,
  options: RequestOptions = {}
) => {
  const [state, setState] = useState<RequestState<T>>({
    data: null,
    loading: false,
    error: null
  });

  const abortControllerRef = useRef<AbortController | null>(null);
  const { skip, ...fetchOptions } = options;

  const execute = useCallback(
    async (overrideUrl?: string) => {
      // Cancel previous request
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }

      // Create new AbortController
      const abortController = new AbortController();
      abortControllerRef.current = abortController;

      setState(prev => ({ ...prev, loading: true, error: null }));

      try {
        const response = await fetch(overrideUrl || url, {
          ...fetchOptions,
          signal: abortController.signal
        });

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        const data = await response.json();

        // Only update state if not aborted
        if (!abortController.signal.aborted) {
          setState({ data, loading: false, error: null });
        }

        return data;
      } catch (error: any) {
        // Ignore abort errors
        if (error.name === 'AbortError') {
          console.log('Request aborted');
          return;
        }

        if (!abortController.signal.aborted) {
          setState({ data: null, loading: false, error });
        }

        throw error;
      }
    },
    [url, fetchOptions]
  );

  const cancel = useCallback(() => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
      setState(prev => ({ ...prev, loading: false }));
    }
  }, []);

  // Auto-execute unless skip is true
  useEffect(() => {
    if (!skip) {
      execute();
    }

    // Cleanup: cancel on unmount
    return () => {
      cancel();
    };
  }, [url, skip]);

  return {
    ...state,
    execute,
    cancel,
    isAborted: abortControllerRef.current?.signal.aborted || false
  };
};

// Usage
const UserList = () => {
  const { data, loading, error, execute, cancel } = useCancellableRequest<User[]>(
    '/api/users'
  );

  const handleRefresh = () => {
    execute(); // This cancels previous request and starts new one
  };

  const handleCancel = () => {
    cancel(); // Manually cancel ongoing request
  };

  return (
    <div>
      <button onClick={handleRefresh}>Refresh</button>
      <button onClick={handleCancel} disabled={!loading}>
        Cancel
      </button>

      {loading && <div>Loading...</div>}
      {error && <div>Error: {error.message}</div>}
      {data && (
        <ul>
          {data.map(user => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

## Cancelling Multiple Requests

### Managing Multiple Controllers

```typescript
// React hook for managing multiple simultaneous requests
interface RequestTracker {
  [key: string]: AbortController;
}

export const useMultipleAbortControllers = () => {
  const controllersRef = useRef<RequestTracker>({});

  const createController = (key: string): AbortController => {
    // Abort existing controller with this key
    if (controllersRef.current[key]) {
      controllersRef.current[key].abort();
    }

    const controller = new AbortController();
    controllersRef.current[key] = controller;
    return controller;
  };

  const getController = (key: string): AbortController | undefined => {
    return controllersRef.current[key];
  };

  const abort = (key: string, reason?: string) => {
    const controller = controllersRef.current[key];
    if (controller) {
      controller.abort(reason);
      delete controllersRef.current[key];
    }
  };

  const abortAll = (reason?: string) => {
    Object.keys(controllersRef.current).forEach(key => {
      controllersRef.current[key].abort(reason);
    });
    controllersRef.current = {};
  };

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      abortAll('Component unmounted');
    };
  }, []);

  return {
    createController,
    getController,
    abort,
    abortAll
  };
};

// Usage: Fetching multiple resources
const Dashboard = () => {
  const [users, setUsers] = useState([]);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  const [loading, setLoading] = useState(false);

  const { createController, abortAll } = useMultipleAbortControllers();

  const fetchAllData = async () => {
    setLoading(true);

    try {
      // Create separate controller for each request
      const usersController = createController('users');
      const postsController = createController('posts');
      const commentsController = createController('comments');

      const [usersData, postsData, commentsData] = await Promise.all([
        fetch('/api/users', { signal: usersController.signal }).then(r => r.json()),
        fetch('/api/posts', { signal: postsController.signal }).then(r => r.json()),
        fetch('/api/comments', { signal: commentsController.signal }).then(r => r.json())
      ]);

      setUsers(usersData);
      setPosts(postsData);
      setComments(commentsData);
    } catch (error: any) {
      if (error.name === 'AbortError') {
        console.log('Requests cancelled');
        return;
      }
      console.error('Failed to fetch data:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleCancelAll = () => {
    abortAll('User cancelled');
    setLoading(false);
  };

  useEffect(() => {
    fetchAllData();

    // Cleanup: cancel all on unmount
    return () => {
      abortAll('Component unmounted');
    };
  }, []);

  return (
    <div>
      {loading && (
        <>
          <div>Loading...</div>
          <button onClick={handleCancelAll}>Cancel All</button>
        </>
      )}
      <div>Users: {users.length}</div>
      <div>Posts: {posts.length}</div>
      <div>Comments: {comments.length}</div>
    </div>
  );
};
```

## Timeout with AbortController

### Automatic Timeout

```typescript
// Create AbortSignal with timeout
const fetchWithTimeout = async (
  url: string,
  options: RequestInit = {},
  timeoutMs: number = 5000
): Promise<Response> => {
  const controller = new AbortController();
  const signal = controller.signal;

  // Set timeout
  const timeoutId = setTimeout(() => {
    controller.abort('Request timeout');
  }, timeoutMs);

  try {
    const response = await fetch(url, {
      ...options,
      signal
    });

    clearTimeout(timeoutId);
    return response;
  } catch (error: any) {
    clearTimeout(timeoutId);

    if (error.name === 'AbortError') {
      throw new Error(`Request timeout after ${timeoutMs}ms`);
    }

    throw error;
  }
};

// Modern approach with AbortSignal.timeout (Node 17.3+, modern browsers)
const fetchWithTimeoutModern = async (
  url: string,
  options: RequestInit = {},
  timeoutMs: number = 5000
): Promise<Response> => {
  try {
    return await fetch(url, {
      ...options,
      signal: AbortSignal.timeout(timeoutMs)
    });
  } catch (error: any) {
    if (error.name === 'AbortError' || error.name === 'TimeoutError') {
      throw new Error(`Request timeout after ${timeoutMs}ms`);
    }
    throw error;
  }
};

// React hook with timeout
export const useRequestWithTimeout = <T,>(
  url: string,
  timeoutMs: number = 5000,
  options: RequestInit = {}
) => {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
    timedOut: boolean;
  }>({
    data: null,
    loading: false,
    error: null,
    timedOut: false
  });

  const execute = useCallback(async () => {
    setState({ data: null, loading: true, error: null, timedOut: false });

    try {
      const response = await fetchWithTimeout(url, options, timeoutMs);
      const data = await response.json();
      setState({ data, loading: false, error: null, timedOut: false });
    } catch (error: any) {
      const isTimeout = error.message.includes('timeout');
      setState({
        data: null,
        loading: false,
        error,
        timedOut: isTimeout
      });
    }
  }, [url, timeoutMs, options]);

  useEffect(() => {
    execute();
  }, [execute]);

  return { ...state, retry: execute };
};

// Usage
const SlowAPIComponent = () => {
  const { data, loading, error, timedOut, retry } = useRequestWithTimeout<any>(
    '/api/slow-endpoint',
    5000 // 5 second timeout
  );

  if (loading) return <div>Loading...</div>;
  
  if (timedOut) {
    return (
      <div>
        <p>Request timed out after 5 seconds</p>
        <button onClick={retry}>Retry</button>
      </div>
    );
  }

  if (error) return <div>Error: {error.message}</div>;
  if (!data) return null;

  return <div>{JSON.stringify(data)}</div>;
};
```

## Combining Multiple Signals

### Merge Multiple AbortSignals

```typescript
// Utility to combine multiple abort signals
const combineSignals = (...signals: AbortSignal[]): AbortSignal => {
  const controller = new AbortController();

  // Abort combined signal if any input signal aborts
  for (const signal of signals) {
    if (signal.aborted) {
      controller.abort(signal.reason);
      break;
    }

    signal.addEventListener('abort', () => {
      controller.abort(signal.reason);
    }, { once: true });
  }

  return controller.signal;
};

// Usage: User cancellation + timeout
const fetchWithUserCancellationAndTimeout = async (
  url: string,
  userController: AbortController,
  timeoutMs: number = 5000
): Promise<Response> => {
  const timeoutController = new AbortController();
  
  const timeoutId = setTimeout(() => {
    timeoutController.abort('Timeout');
  }, timeoutMs);

  try {
    // Combine user cancellation and timeout
    const combinedSignal = combineSignals(
      userController.signal,
      timeoutController.signal
    );

    const response = await fetch(url, { signal: combinedSignal });
    clearTimeout(timeoutId);
    return response;
  } catch (error) {
    clearTimeout(timeoutId);
    throw error;
  }
};

// React component with both user and automatic cancellation
const FileUploader = ({ file }: { file: File }) => {
  const [progress, setProgress] = useState(0);
  const [status, setStatus] = useState<'idle' | 'uploading' | 'completed' | 'cancelled'>('idle');
  const userControllerRef = useRef<AbortController | null>(null);

  const uploadFile = async () => {
    userControllerRef.current = new AbortController();
    setStatus('uploading');

    try {
      const formData = new FormData();
      formData.append('file', file);

      // 30 second timeout
      const response = await fetchWithUserCancellationAndTimeout(
        '/api/upload',
        userControllerRef.current,
        30000
      );

      if (response.ok) {
        setStatus('completed');
      }
    } catch (error: any) {
      if (error.name === 'AbortError') {
        setStatus('cancelled');
      } else {
        console.error('Upload failed:', error);
      }
    }
  };

  const handleCancel = () => {
    if (userControllerRef.current) {
      userControllerRef.current.abort('User cancelled');
    }
  };

  return (
    <div>
      <div>Status: {status}</div>
      <div>Progress: {progress}%</div>
      
      {status === 'idle' && (
        <button onClick={uploadFile}>Start Upload</button>
      )}
      
      {status === 'uploading' && (
        <button onClick={handleCancel}>Cancel Upload</button>
      )}
    </div>
  );
};
```

## Request Cancellation with Axios

### Axios CancelToken

```typescript
// React hook for Axios with cancellation
import axios, { CancelTokenSource } from 'axios';
import { useState, useEffect, useRef } from 'react';

export const useAxiosRequest = <T,>(
  url: string,
  options: any = {}
) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const cancelTokenSourceRef = useRef<CancelTokenSource | null>(null);

  const execute = async () => {
    // Cancel previous request
    if (cancelTokenSourceRef.current) {
      cancelTokenSourceRef.current.cancel('Operation cancelled due to new request');
    }

    // Create new cancel token
    cancelTokenSourceRef.current = axios.CancelToken.source();

    setLoading(true);
    setError(null);

    try {
      const response = await axios({
        url,
        ...options,
        cancelToken: cancelTokenSourceRef.current.token
      });

      setData(response.data);
      setLoading(false);
    } catch (err: any) {
      if (axios.isCancel(err)) {
        console.log('Request cancelled:', err.message);
      } else {
        setError(err);
        setLoading(false);
      }
    }
  };

  const cancel = (message?: string) => {
    if (cancelTokenSourceRef.current) {
      cancelTokenSourceRef.current.cancel(message || 'Request cancelled');
    }
  };

  useEffect(() => {
    execute();

    // Cleanup: cancel on unmount
    return () => {
      cancel('Component unmounted');
    };
  }, [url]);

  return { data, loading, error, execute, cancel };
};

// Usage
const ProductList = () => {
  const { data, loading, error, cancel } = useAxiosRequest<Product[]>(
    '/api/products'
  );

  return (
    <div>
      {loading && (
        <>
          <div>Loading...</div>
          <button onClick={() => cancel()}>Cancel</button>
        </>
      )}
      {error && <div>Error: {error.message}</div>}
      {data && (
        <ul>
          {data.map(product => (
            <li key={product.id}>{product.name}</li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

## Race Condition Prevention

### Latest Request Wins Pattern

```typescript
// Ensure only the latest request updates state
export const useLatestRequest = <T,>() => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  const latestRequestIdRef = useRef<number>(0);
  const abortControllerRef = useRef<AbortController | null>(null);

  const execute = async (fetcher: () => Promise<T>) => {
    // Cancel previous request
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    // Increment request ID
    const requestId = ++latestRequestIdRef.current;
    
    // Create new abort controller
    const abortController = new AbortController();
    abortControllerRef.current = abortController;

    setLoading(true);
    setError(null);

    try {
      const result = await fetcher();

      // Only update state if this is still the latest request
      if (requestId === latestRequestIdRef.current && !abortController.signal.aborted) {
        setData(result);
        setLoading(false);
      }
    } catch (err: any) {
      if (err.name === 'AbortError') {
        console.log('Request aborted');
        return;
      }

      // Only update error if this is still the latest request
      if (requestId === latestRequestIdRef.current) {
        setError(err);
        setLoading(false);
      }
    }
  };

  return { data, loading, error, execute };
};

// Usage: Search with race condition prevention
const SearchWithRaceProtection = () => {
  const [query, setQuery] = useState('');
  const { data, loading, error, execute } = useLatestRequest<SearchResult[]>();

  const handleSearch = (searchQuery: string) => {
    if (!searchQuery) return;

    execute(async () => {
      const response = await fetch(`/api/search?q=${searchQuery}`);
      return response.json();
    });
  };

  const debouncedSearch = useDebounce(handleSearch, 300);

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          debouncedSearch(e.target.value);
        }}
        placeholder="Search..."
      />
      {loading && <div>Loading...</div>}
      {error && <div>Error: {error.message}</div>}
      {data && (
        <ul>
          {data.map(result => (
            <li key={result.id}>{result.title}</li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

## Common Mistakes

### 1. Not Handling AbortError

```typescript
// ❌ BAD: Treating abort as error
const fetchUser = async (id: string, signal: AbortSignal) => {
  try {
    const response = await fetch(`/api/users/${id}`, { signal });
    return response.json();
  } catch (error) {
    // This shows error to user even when intentionally cancelled
    showErrorToUser(error);
  }
};

// ✅ GOOD: Ignoring AbortError
const fetchUser = async (id: string, signal: AbortSignal) => {
  try {
    const response = await fetch(`/api/users/${id}`, { signal });
    return response.json();
  } catch (error: any) {
    if (error.name === 'AbortError') {
      console.log('Request cancelled');
      return; // Silently ignore
    }
    showErrorToUser(error);
  }
};
```

### 2. Forgetting Cleanup

```typescript
// ❌ BAD: No cleanup on unmount
const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      const response = await fetch(`/api/users/${userId}`);
      const data = await response.json();
      setUser(data); // Memory leak if component unmounts
    };

    fetchUser();
    // Missing cleanup!
  }, [userId]);

  return <div>{user?.name}</div>;
};

// ✅ GOOD: Proper cleanup
const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    const fetchUser = async () => {
      try {
        const response = await fetch(`/api/users/${userId}`, {
          signal: controller.signal
        });
        const data = await response.json();
        
        if (!controller.signal.aborted) {
          setUser(data);
        }
      } catch (error: any) {
        if (error.name !== 'AbortError') {
          console.error(error);
        }
      }
    };

    fetchUser();

    return () => {
      controller.abort(); // Cleanup
    };
  }, [userId]);

  return <div>{user?.name}</div>;
};
```

### 3. Reusing AbortControllers

```typescript
// ❌ BAD: Reusing same controller
const controller = new AbortController();

const fetchData1 = () => fetch('/api/data1', { signal: controller.signal });
const fetchData2 = () => fetch('/api/data2', { signal: controller.signal });

controller.abort(); // Aborts BOTH requests!

// ✅ GOOD: Separate controller for each request
const controller1 = new AbortController();
const controller2 = new AbortController();

const fetchData1 = () => fetch('/api/data1', { signal: controller1.signal });
const fetchData2 = () => fetch('/api/data2', { signal: controller2.signal });

controller1.abort(); // Only aborts first request
```

## Best Practices

### 1. Always Cancel on Unmount

```typescript
// Cancel all pending requests when component unmounts
useEffect(() => {
  const controller = new AbortController();

  fetchData(controller.signal);

  return () => {
    controller.abort('Component unmounted');
  };
}, []);
```

### 2. Provide User Feedback

```typescript
const LongRunningOperation = () => {
  const [status, setStatus] = useState<'idle' | 'running' | 'cancelled'>('idle');
  const controllerRef = useRef<AbortController | null>(null);

  const startOperation = () => {
    controllerRef.current = new AbortController();
    setStatus('running');

    performOperation(controllerRef.current.signal)
      .then(() => setStatus('idle'))
      .catch((err) => {
        if (err.name === 'AbortError') {
          setStatus('cancelled');
        }
      });
  };

  const cancel = () => {
    controllerRef.current?.abort();
  };

  return (
    <div>
      {status === 'running' && (
        <>
          <div>Operation in progress...</div>
          <button onClick={cancel}>Cancel</button>
        </>
      )}
      {status === 'cancelled' && (
        <div>Operation was cancelled</div>
      )}
    </div>
  );
};
```

### 3. Graceful Degradation

```typescript
// Fallback for browsers without AbortController
const fetchWithOptionalCancellation = (url: string, signal?: AbortSignal) => {
  if (signal && 'AbortController' in window) {
    return fetch(url, { signal });
  }
  return fetch(url);
};
```

## When to Cancel Requests

### Always Cancel

- Component unmounts before request completes
- User navigates away from page
- User triggers new search (cancel old search)
- User clicks cancel button
- Request takes too long (timeout)

### Consider Not Cancelling

- Critical operations (payments, submissions)
- Data being cached for later use
- Background prefetching
- Analytics/logging requests

## Interview Questions

### 1. What is AbortController and how does it work?
**Answer**: AbortController is a Web API that provides a way to abort one or more Web requests. It creates a signal object that can be passed to fetch and other async APIs. When abort() is called on the controller, the signal triggers an AbortError in any ongoing operations using that signal, allowing graceful cancellation.

### 2. How do you prevent race conditions in search inputs?
**Answer**: Use request cancellation with AbortController. When a new search is triggered, abort the previous request before starting a new one. Alternatively, use a request ID pattern where only the latest request updates state. Combine with debouncing to reduce number of requests.

### 3. What happens if you call setState after a component unmounts?
**Answer**: It causes a memory leak warning. The request completes but tries to update unmounted component state. Fix by cancelling requests in cleanup function or checking if component is mounted before calling setState (using AbortSignal.aborted check).

### 4. Can you reuse an AbortController?
**Answer**: No, once aborted, an AbortController cannot be reused. Its signal remains in the aborted state permanently. Create a new AbortController for each new request or set of requests you want to control independently.

### 5. How do you handle errors from cancelled requests?
**Answer**: Check if error.name === 'AbortError' and handle it separately from other errors. Typically, AbortError is expected and should be handled silently (just logging), while other errors should be shown to users or handled appropriately.

## Key Takeaways

1. **Always Cancel on Unmount**: Prevent memory leaks by cancelling requests in cleanup functions.
2. **Use AbortController**: Modern, standard way to cancel fetch requests.
3. **Ignore AbortErrors**: Don't treat intentional cancellations as errors.
4. **One Controller Per Request**: Don't reuse AbortControllers across requests.
5. **Combine with Timeouts**: Use AbortSignal for both user cancellation and timeouts.
6. **Prevent Race Conditions**: Cancel old requests when new ones start.
7. **Provide Feedback**: Show users when operations can be cancelled.
8. **Handle Gracefully**: Check if request was aborted before updating state.

## Resources

- [MDN - AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [MDN - AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
- [Fetch API - Using AbortController](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#aborting_a_fetch)
- [AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout)
- [React - Cleanup Functions](https://react.dev/learn/synchronizing-with-effects#step-3-add-cleanup-if-needed)
- [Axios Cancellation](https://axios-http.com/docs/cancellation)
