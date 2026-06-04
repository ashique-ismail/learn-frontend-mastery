# Long Polling

## Overview

Long polling is a technique where the client sends an HTTP request to the server and keeps the connection open until the server has data to send or a timeout occurs. Once the response is sent, the client immediately sends another request, creating a near real-time communication channel. While less efficient than WebSockets or SSE, long polling works universally across all browsers and network configurations without requiring special protocol support.

## Polling vs Long Polling vs WebSockets

```
Regular Polling:
Client ────Request────> Server
Client <───Response──── Server (immediate, may be empty)
[Wait delay]
Client ────Request────> Server
Client <───Response──── Server
[Repeated frequently]

Long Polling:
Client ────Request────────────────> Server
                                    [Server holds connection]
                                    [Until data available or timeout]
Client <─────────Response───────── Server
Client ────Request────────────────> Server (immediately)

WebSockets:
Client ─────Handshake─────> Server
Client <────Connected────→ Server
       [Persistent bidirectional]
```

## Architecture Comparison

```
Network Efficiency:

Regular Polling (Most overhead):
┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
│Req││Res││Req││Res││Req││Res│
└───┘└───┘└───┘└───┘└───┘└───┘
 1s   1s   1s   1s   1s   1s

Long Polling (Better):
┌─────────┐┌─────────┐┌─────────┐
│  Req    ││  Req    ││  Req    │
│ (wait)  ││ (wait)  ││ (wait)  │
│   Res   ││   Res   ││   Res   │
└─────────┘└─────────┘└─────────┘

WebSocket (Best):
┌──────────────────────────────┐
│    Persistent Connection     │
│  Messages sent as needed     │
└──────────────────────────────┘
```

## Basic Long Polling (React)

```typescript
// hooks/useLongPolling.ts
import { useEffect, useRef, useState, useCallback } from 'react';

interface UseLongPollingOptions<T> {
  url: string;
  interval?: number;
  timeout?: number;
  onData?: (data: T) => void;
  onError?: (error: Error) => void;
  enabled?: boolean;
}

export const useLongPolling = <T>({
  url,
  interval = 0,
  timeout = 30000,
  onData,
  onError,
  enabled = true
}: UseLongPollingOptions<T>) => {
  const [data, setData] = useState<T | null>(null);
  const [isPolling, setIsPolling] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const abortControllerRef = useRef<AbortController | null>(null);
  const timeoutRef = useRef<NodeJS.Timeout>();
  const isActiveRef = useRef(true);

  const poll = useCallback(async () => {
    if (!enabled || !isActiveRef.current) return;

    setIsPolling(true);
    abortControllerRef.current = new AbortController();

    try {
      const response = await fetch(url, {
        signal: abortControllerRef.current.signal,
        headers: {
          'Content-Type': 'application/json'
        }
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const result = await response.json();
      
      if (isActiveRef.current) {
        setData(result);
        setError(null);
        onData?.(result);

        // Schedule next poll
        if (interval > 0) {
          timeoutRef.current = setTimeout(poll, interval);
        } else {
          // Immediate re-poll for true long polling
          poll();
        }
      }
    } catch (err) {
      if (err.name === 'AbortError') {
        console.log('Poll aborted');
        return;
      }

      const error = err instanceof Error ? err : new Error('Unknown error');
      
      if (isActiveRef.current) {
        setError(error);
        onError?.(error);

        // Retry after interval on error
        timeoutRef.current = setTimeout(poll, interval || 5000);
      }
    } finally {
      if (isActiveRef.current) {
        setIsPolling(false);
      }
    }
  }, [url, interval, timeout, onData, onError, enabled]);

  const stopPolling = useCallback(() => {
    isActiveRef.current = false;
    abortControllerRef.current?.abort();
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
  }, []);

  const startPolling = useCallback(() => {
    isActiveRef.current = true;
    poll();
  }, [poll]);

  useEffect(() => {
    if (enabled) {
      startPolling();
    }

    return () => {
      stopPolling();
    };
  }, [enabled, startPolling, stopPolling]);

  return {
    data,
    isPolling,
    error,
    startPolling,
    stopPolling
  };
};
```

```typescript
// components/NotificationCenter.tsx
import React, { useState } from 'react';
import { useLongPolling } from '../hooks/useLongPolling';

interface Notification {
  id: string;
  message: string;
  type: 'info' | 'warning' | 'error';
  timestamp: string;
}

interface PollResponse {
  notifications: Notification[];
  lastId: string;
}

const NotificationCenter: React.FC = () => {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [lastId, setLastId] = useState<string>('0');

  const { isPolling, error } = useLongPolling<PollResponse>({
    url: `/api/notifications/poll?lastId=${lastId}`,
    onData: (data) => {
      if (data.notifications.length > 0) {
        setNotifications(prev => [...data.notifications, ...prev]);
        setLastId(data.lastId);
      }
    },
    onError: (err) => {
      console.error('Polling error:', err);
    },
    timeout: 30000,
    enabled: true
  });

  const clearNotifications = () => {
    setNotifications([]);
  };

  return (
    <div className="notification-center">
      <div className="header">
        <h2>Notifications</h2>
        <div className="status">
          {isPolling ? '🔄 Polling...' : '✓ Connected'}
          {error && <span className="error">⚠️ Error</span>}
        </div>
      </div>

      <div className="notifications-list">
        {notifications.length === 0 ? (
          <p className="empty-state">No new notifications</p>
        ) : (
          <>
            <button onClick={clearNotifications}>Clear All</button>
            {notifications.map((notification) => (
              <div key={notification.id} className={`notification ${notification.type}`}>
                <p>{notification.message}</p>
                <time>{new Date(notification.timestamp).toLocaleString()}</time>
              </div>
            ))}
          </>
        )}
      </div>
    </div>
  );
};

export default NotificationCenter;
```

## Advanced Long Polling with Exponential Backoff (React)

```typescript
// hooks/useAdvancedLongPolling.ts
import { useEffect, useRef, useState, useCallback } from 'react';

interface AdvancedPollingOptions<T> {
  url: string;
  onData?: (data: T) => void;
  onError?: (error: Error) => void;
  maxRetries?: number;
  initialBackoff?: number;
  maxBackoff?: number;
  timeout?: number;
  enabled?: boolean;
}

export const useAdvancedLongPolling = <T>({
  url,
  onData,
  onError,
  maxRetries = 10,
  initialBackoff = 1000,
  maxBackoff = 60000,
  timeout = 30000,
  enabled = true
}: AdvancedPollingOptions<T>) => {
  const [data, setData] = useState<T | null>(null);
  const [isPolling, setIsPolling] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [retryCount, setRetryCount] = useState(0);
  
  const abortControllerRef = useRef<AbortController | null>(null);
  const timeoutRef = useRef<NodeJS.Timeout>();
  const isActiveRef = useRef(true);
  const retryCountRef = useRef(0);

  const calculateBackoff = useCallback((retries: number): number => {
    // Exponential backoff with jitter
    const exponential = Math.min(maxBackoff, initialBackoff * Math.pow(2, retries));
    const jitter = Math.random() * 1000;
    return exponential + jitter;
  }, [initialBackoff, maxBackoff]);

  const poll = useCallback(async () => {
    if (!enabled || !isActiveRef.current) return;

    if (retryCountRef.current >= maxRetries) {
      const maxRetriesError = new Error('Max retries reached');
      setError(maxRetriesError);
      onError?.(maxRetriesError);
      return;
    }

    setIsPolling(true);
    abortControllerRef.current = new AbortController();

    const timeoutId = setTimeout(() => {
      abortControllerRef.current?.abort();
    }, timeout);

    try {
      const response = await fetch(url, {
        signal: abortControllerRef.current.signal,
        headers: {
          'Content-Type': 'application/json',
          'X-Retry-Count': retryCountRef.current.toString()
        }
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const result = await response.json();
      
      if (isActiveRef.current) {
        setData(result);
        setError(null);
        setRetryCount(0);
        retryCountRef.current = 0;
        onData?.(result);

        // Immediate re-poll on success
        poll();
      }
    } catch (err) {
      if (err.name === 'AbortError') {
        // Timeout occurred, retry
        console.log('Request timeout, retrying...');
      }

      clearTimeout(timeoutId);

      const error = err instanceof Error ? err : new Error('Unknown error');
      
      if (isActiveRef.current) {
        setError(error);
        onError?.(error);
        
        retryCountRef.current++;
        setRetryCount(retryCountRef.current);

        // Exponential backoff
        const backoffTime = calculateBackoff(retryCountRef.current);
        console.log(`Retrying in ${backoffTime}ms (attempt ${retryCountRef.current}/${maxRetries})`);
        
        timeoutRef.current = setTimeout(poll, backoffTime);
      }
    } finally {
      if (isActiveRef.current) {
        setIsPolling(false);
      }
    }
  }, [url, timeout, onData, onError, enabled, maxRetries, calculateBackoff]);

  const stopPolling = useCallback(() => {
    isActiveRef.current = false;
    abortControllerRef.current?.abort();
    if (timeoutRef.current) {
      clearTimeout(timeoutRef.current);
    }
  }, []);

  const startPolling = useCallback(() => {
    isActiveRef.current = true;
    retryCountRef.current = 0;
    setRetryCount(0);
    poll();
  }, [poll]);

  useEffect(() => {
    if (enabled) {
      startPolling();
    }

    return () => {
      stopPolling();
    };
  }, [enabled, startPolling, stopPolling]);

  return {
    data,
    isPolling,
    error,
    retryCount,
    startPolling,
    stopPolling
  };
};
```

```typescript
// components/LiveFeed.tsx
import React, { useState, useCallback } from 'react';
import { useAdvancedLongPolling } from '../hooks/useAdvancedLongPolling';

interface FeedItem {
  id: string;
  title: string;
  content: string;
  author: string;
  timestamp: string;
}

interface FeedResponse {
  items: FeedItem[];
  cursor: string;
  hasMore: boolean;
}

const LiveFeed: React.FC = () => {
  const [feedItems, setFeedItems] = useState<FeedItem[]>([]);
  const [cursor, setCursor] = useState<string>('');
  const [isPaused, setIsPaused] = useState(false);

  const { isPolling, error, retryCount, stopPolling, startPolling } = 
    useAdvancedLongPolling<FeedResponse>({
      url: `/api/feed/poll?cursor=${cursor}`,
      onData: useCallback((data) => {
        if (data.items.length > 0) {
          setFeedItems(prev => [...data.items, ...prev]);
          setCursor(data.cursor);
        }
      }, []),
      onError: (err) => {
        console.error('Feed polling error:', err);
      },
      maxRetries: 10,
      initialBackoff: 1000,
      maxBackoff: 60000,
      timeout: 45000,
      enabled: !isPaused
    });

  const togglePause = () => {
    if (isPaused) {
      startPolling();
    } else {
      stopPolling();
    }
    setIsPaused(!isPaused);
  };

  return (
    <div className="live-feed">
      <div className="header">
        <h2>Live Feed</h2>
        <div className="controls">
          <button onClick={togglePause}>
            {isPaused ? '▶️ Resume' : '⏸️ Pause'}
          </button>
          <div className="status">
            {isPolling && '🔄 Polling...'}
            {error && `⚠️ Error (Retry ${retryCount})`}
            {isPaused && '⏸️ Paused'}
          </div>
        </div>
      </div>

      <div className="feed-items">
        {feedItems.map((item) => (
          <article key={item.id} className="feed-item">
            <h3>{item.title}</h3>
            <p>{item.content}</p>
            <div className="meta">
              <span className="author">{item.author}</span>
              <time>{new Date(item.timestamp).toLocaleString()}</time>
            </div>
          </article>
        ))}
        {feedItems.length === 0 && (
          <p className="empty-state">No items yet</p>
        )}
      </div>
    </div>
  );
};

export default LiveFeed;
```

## Long Polling (Angular)

```typescript
// services/long-polling.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, Subject, timer, throwError } from 'rxjs';
import { 
  switchMap, 
  retry, 
  catchError, 
  takeUntil, 
  tap,
  delayWhen 
} from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class LongPollingService {
  private stopPolling$ = new Subject<void>();

  constructor(private http: HttpClient) {}

  startPolling<T>(
    url: string,
    interval: number = 0,
    maxRetries: number = 3
  ): Observable<T> {
    return timer(0, interval).pipe(
      switchMap(() => 
        this.http.get<T>(url).pipe(
          retry(maxRetries),
          catchError(error => {
            console.error('Polling error:', error);
            return throwError(() => error);
          })
        )
      ),
      takeUntil(this.stopPolling$)
    );
  }

  startPollingWithBackoff<T>(
    url: string,
    initialDelay: number = 1000,
    maxDelay: number = 60000
  ): Observable<T> {
    let currentDelay = initialDelay;
    let retryCount = 0;

    const poll$ = new Subject<T>();

    const executePoll = () => {
      this.http.get<T>(url).subscribe({
        next: (data) => {
          // Reset delay on success
          currentDelay = initialDelay;
          retryCount = 0;
          poll$.next(data);
          
          // Immediately poll again
          setTimeout(executePoll, 0);
        },
        error: (error) => {
          console.error(`Polling error (attempt ${retryCount}):`, error);
          
          // Exponential backoff
          retryCount++;
          currentDelay = Math.min(maxDelay, currentDelay * 2);
          
          setTimeout(executePoll, currentDelay);
        }
      });
    };

    executePoll();

    return poll$.asObservable().pipe(
      takeUntil(this.stopPolling$)
    );
  }

  stopPolling() {
    this.stopPolling$.next();
  }
}
```

```typescript
// services/notification-polling.service.ts
import { Injectable } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { LongPollingService } from './long-polling.service';

export interface Notification {
  id: string;
  message: string;
  type: string;
  timestamp: string;
}

export interface NotificationResponse {
  notifications: Notification[];
  lastId: string;
}

@Injectable({
  providedIn: 'root'
})
export class NotificationPollingService {
  private notificationsSubject = new Subject<Notification[]>();
  public notifications$ = this.notificationsSubject.asObservable();
  private lastId = '0';

  constructor(private pollingService: LongPollingService) {}

  startPolling() {
    this.pollingService
      .startPollingWithBackoff<NotificationResponse>(
        `/api/notifications/poll?lastId=${this.lastId}`
      )
      .subscribe({
        next: (response) => {
          if (response.notifications.length > 0) {
            this.notificationsSubject.next(response.notifications);
            this.lastId = response.lastId;
          }
        },
        error: (error) => {
          console.error('Notification polling error:', error);
        }
      });
  }

  stopPolling() {
    this.pollingService.stopPolling();
  }
}
```

```typescript
// components/notifications.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { 
  NotificationPollingService, 
  Notification 
} from '../services/notification-polling.service';

@Component({
  selector: 'app-notifications',
  template: `
    <div class="notifications-container">
      <h2>Notifications</h2>
      <div class="notifications-list">
        <div *ngFor="let notification of notifications" 
             [class]="'notification ' + notification.type">
          <p>{{ notification.message }}</p>
          <time>{{ notification.timestamp | date:'short' }}</time>
        </div>
        <p *ngIf="notifications.length === 0" class="empty-state">
          No notifications
        </p>
      </div>
    </div>
  `
})
export class NotificationsComponent implements OnInit, OnDestroy {
  notifications: Notification[] = [];
  private subscription: Subscription;

  constructor(
    private notificationPolling: NotificationPollingService
  ) {}

  ngOnInit() {
    this.notificationPolling.startPolling();
    
    this.subscription = this.notificationPolling.notifications$
      .subscribe(newNotifications => {
        this.notifications = [...newNotifications, ...this.notifications];
      });
  }

  ngOnDestroy() {
    this.notificationPolling.stopPolling();
    
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
  }
}
```

## Server-Side Implementation

```typescript
// server/long-polling-server.ts
import express, { Request, Response } from 'express';

const app = express();
app.use(express.json());

interface PendingRequest {
  id: string;
  response: Response;
  timeout: NodeJS.Timeout;
  lastId: string;
}

const pendingRequests: PendingRequest[] = [];
const notifications: any[] = [];
let notificationId = 0;

// Long polling endpoint
app.get('/api/notifications/poll', (req: Request, res: Response) => {
  const lastId = req.query.lastId as string || '0';
  const clientId = Math.random().toString(36).substring(7);

  console.log(`Client ${clientId} polling (lastId: ${lastId})`);

  // Check if there are new notifications
  const newNotifications = notifications.filter(n => n.id > parseInt(lastId));

  if (newNotifications.length > 0) {
    // Send immediately if data is available
    res.json({
      notifications: newNotifications,
      lastId: notifications[notifications.length - 1].id.toString()
    });
    return;
  }

  // Hold the connection (long polling)
  const timeout = setTimeout(() => {
    // Send empty response after timeout
    const index = pendingRequests.findIndex(r => r.id === clientId);
    if (index !== -1) {
      pendingRequests.splice(index, 1);
    }

    res.json({
      notifications: [],
      lastId
    });
  }, 30000); // 30 second timeout

  // Store pending request
  pendingRequests.push({
    id: clientId,
    response: res,
    timeout,
    lastId
  });

  // Handle client disconnect
  req.on('close', () => {
    const index = pendingRequests.findIndex(r => r.id === clientId);
    if (index !== -1) {
      clearTimeout(pendingRequests[index].timeout);
      pendingRequests.splice(index, 1);
    }
  });
});

// Endpoint to create notification (for testing)
app.post('/api/notifications', (req: Request, res: Response) => {
  const notification = {
    id: ++notificationId,
    message: req.body.message,
    type: req.body.type || 'info',
    timestamp: new Date().toISOString()
  };

  notifications.push(notification);

  // Send to all pending clients
  pendingRequests.forEach(request => {
    if (parseInt(request.lastId) < notification.id) {
      clearTimeout(request.timeout);
      
      const relevantNotifications = notifications.filter(
        n => n.id > parseInt(request.lastId)
      );

      request.response.json({
        notifications: relevantNotifications,
        lastId: notification.id.toString()
      });
    }
  });

  // Clear pending requests
  pendingRequests.length = 0;

  res.json({ success: true, notification });
});

// Feed polling endpoint with cursor pagination
app.get('/api/feed/poll', (req: Request, res: Response) => {
  const cursor = req.query.cursor as string || '';

  // Simulate waiting for new data
  setTimeout(() => {
    const newItems = [
      {
        id: Date.now().toString(),
        title: 'New Feed Item',
        content: 'Content here...',
        author: 'User',
        timestamp: new Date().toISOString()
      }
    ];

    res.json({
      items: newItems,
      cursor: Date.now().toString(),
      hasMore: true
    });
  }, Math.random() * 10000 + 5000); // Random delay 5-15 seconds
});

const PORT = 3001;
app.listen(PORT, () => {
  console.log(`Long polling server running on http://localhost:${PORT}`);
});
```

## Common Mistakes

### Mistake 1: Not Implementing Timeout

```typescript
// BAD: No timeout, connection hangs forever
const poll = async () => {
  const response = await fetch('/api/poll');
  const data = await response.json();
  poll(); // Immediate re-poll
};

// GOOD: Implement timeout
const poll = async () => {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 30000);
  
  try {
    const response = await fetch('/api/poll', { signal: controller.signal });
    const data = await response.json();
    clearTimeout(timeout);
    poll();
  } catch (error) {
    clearTimeout(timeout);
    // Handle error and retry with backoff
  }
};
```

### Mistake 2: No Exponential Backoff on Errors

```typescript
// BAD: Immediate retry floods server
const poll = async () => {
  try {
    const response = await fetch('/api/poll');
    poll();
  } catch (error) {
    poll(); // Immediate retry!
  }
};

// GOOD: Exponential backoff
let retries = 0;
const poll = async () => {
  try {
    const response = await fetch('/api/poll');
    retries = 0;
    poll();
  } catch (error) {
    const backoff = Math.min(60000, 1000 * Math.pow(2, retries));
    retries++;
    setTimeout(poll, backoff);
  }
};
```

### Mistake 3: Memory Leaks from Not Cleaning Up

```typescript
// BAD: No cleanup
useEffect(() => {
  poll();
}, []);

// GOOD: Cleanup on unmount
useEffect(() => {
  let active = true;
  
  const poll = async () => {
    if (!active) return;
    // ... polling logic
  };
  
  poll();
  
  return () => {
    active = false;
  };
}, []);
```

## Best Practices

1. Implement timeout on both client and server
2. Use exponential backoff for error retry
3. Clean up pending requests on component unmount
4. Send empty responses on timeout to keep connection alive
5. Use cursor/lastId for incremental updates
6. Implement maximum retry limit
7. Add jitter to backoff to prevent thundering herd
8. Monitor and limit concurrent connections on server
9. Consider upgrading to WebSockets or SSE for better efficiency
10. Use long polling as fallback strategy

## Key Takeaways

1. Long polling keeps HTTP connection open until data is available
2. More efficient than regular polling, less efficient than WebSockets/SSE
3. Works universally without special protocol support
4. Requires proper timeout and retry logic
5. Can cause server resource exhaustion without proper limits
6. Good fallback strategy when WebSockets/SSE not available
7. Exponential backoff prevents server overload
8. Always clean up connections on client disconnect

## Interview Questions

1. What is the difference between polling and long polling?
2. When would you use long polling vs WebSockets?
3. How do you implement exponential backoff?
4. What are the disadvantages of long polling?
5. How do you handle timeouts in long polling?
6. What is the thundering herd problem?
7. How do you scale long polling across multiple servers?
8. What are connection limits and how do they affect long polling?
9. How do you implement cursor-based pagination with long polling?
10. What is the performance impact of long polling?

## Resources

- HTTP Long Polling MDN
- Comet (Web Application Model)
- Long Polling vs WebSockets Comparison
- Exponential Backoff Algorithm
- Server-Side Event Handling
- HTTP Connection Management
