# Server-Sent Events (SSE)

## The Idea

**In plain English:** Server-Sent Events (SSE) is a way for a web server to continuously push updates to your browser over a single, long-lived connection — like a one-way broadcast where the server talks and the browser listens, without the browser having to keep asking "anything new?"

**Real-world analogy:** Think of a radio station broadcasting the news live. You tune in once and updates keep coming to you automatically — you don't have to call the station to ask for the next headline.

- The radio station = the server (sends updates)
- Your radio receiver = the browser/client (listens for updates)
- Tuning in = opening the EventSource connection
- The broadcast signal = the stream of event messages sent over HTTP

---

## Overview

Server-Sent Events (SSE) is a server push technology that enables a server to send automatic updates to a client via HTTP connection. Unlike WebSockets which provide bidirectional communication, SSE is unidirectional - the server pushes data to the client, but the client can only communicate back via regular HTTP requests. SSE is ideal for scenarios like live notifications, news feeds, stock tickers, and real-time monitoring dashboards.

## SSE vs WebSocket vs Long Polling

```
Communication Patterns:

SSE (Server → Client):
┌─────────┐          ┌─────────┐
│ Client  │←─────────│ Server  │
└─────────┘          └─────────┘
   ↓ HTTP Request        ↑ Events
   (for sending)         (pushed)

WebSocket (Bidirectional):
┌─────────┐          ┌─────────┐
│ Client  │←────────→│ Server  │
└─────────┘          └─────────┘

Long Polling (Request-Response):
┌─────────┐          ┌─────────┐
│ Client  │─────────→│ Server  │
│         │←─────────│  (wait) │
│         │─────────→│         │
└─────────┘          └─────────┘
```

## SSE Protocol

SSE uses a simple text-based protocol:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: message
id: 1
data: Hello World

event: notification
id: 2
data: {"type":"info","message":"New update"}

: This is a comment (heartbeat)

data: Simple message without event type
data: Multi-line
data: message

```

## Native EventSource API (React)

```typescript
// hooks/useSSE.ts
import { useEffect, useState, useCallback, useRef } from 'react';

interface UseSSEOptions {
  url: string;
  withCredentials?: boolean;
  onMessage?: (event: MessageEvent) => void;
  onError?: (error: Event) => void;
  onOpen?: () => void;
  events?: Record<string, (event: MessageEvent) => void>;
}

export const useSSE = ({
  url,
  withCredentials = false,
  onMessage,
  onError,
  onOpen,
  events = {}
}: UseSSEOptions) => {
  const [readyState, setReadyState] = useState<number>(EventSource.CONNECTING);
  const [lastEventId, setLastEventId] = useState<string>('');
  const eventSourceRef = useRef<EventSource | null>(null);

  const connect = useCallback(() => {
    // Close existing connection
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
    }

    const eventSource = new EventSource(url, { withCredentials });
    eventSourceRef.current = eventSource;

    eventSource.onopen = () => {
      console.log('SSE connection opened');
      setReadyState(EventSource.OPEN);
      onOpen?.();
    };

    eventSource.onmessage = (event) => {
      setLastEventId(event.lastEventId);
      onMessage?.(event);
    };

    eventSource.onerror = (error) => {
      console.error('SSE connection error:', error);
      setReadyState(EventSource.CLOSED);
      onError?.(error);
    };

    // Register custom event listeners
    Object.entries(events).forEach(([eventName, handler]) => {
      eventSource.addEventListener(eventName, handler);
    });

    return eventSource;
  }, [url, withCredentials, onMessage, onError, onOpen, events]);

  useEffect(() => {
    const eventSource = connect();

    return () => {
      eventSource.close();
    };
  }, [connect]);

  const close = useCallback(() => {
    if (eventSourceRef.current) {
      eventSourceRef.current.close();
      setReadyState(EventSource.CLOSED);
    }
  }, []);

  return {
    readyState,
    lastEventId,
    close,
    isOpen: readyState === EventSource.OPEN,
    isConnecting: readyState === EventSource.CONNECTING,
    isClosed: readyState === EventSource.CLOSED
  };
};
```

```typescript
// components/LiveNotifications.tsx
import React, { useState } from 'react';
import { useSSE } from '../hooks/useSSE';

interface Notification {
  id: string;
  type: 'info' | 'warning' | 'error' | 'success';
  message: string;
  timestamp: string;
}

const LiveNotifications: React.FC = () => {
  const [notifications, setNotifications] = useState<Notification[]>([]);

  const { isOpen, readyState } = useSSE({
    url: '/api/notifications/stream',
    withCredentials: true,
    onOpen: () => {
      console.log('Connected to notification stream');
    },
    onMessage: (event) => {
      try {
        const notification: Notification = JSON.parse(event.data);
        setNotifications(prev => [notification, ...prev].slice(0, 20));
      } catch (error) {
        console.error('Failed to parse notification:', error);
      }
    },
    onError: (error) => {
      console.error('Notification stream error:', error);
    },
    events: {
      // Handle custom event types
      notification: (event) => {
        const notification: Notification = JSON.parse(event.data);
        setNotifications(prev => [notification, ...prev]);
      },
      alert: (event) => {
        const alert = JSON.parse(event.data);
        // Show alert UI
        alert(alert.message);
      },
      heartbeat: () => {
        // Server heartbeat - connection is alive
      }
    }
  });

  const getStatusIcon = () => {
    switch (readyState) {
      case EventSource.CONNECTING:
        return '🟡';
      case EventSource.OPEN:
        return '🟢';
      case EventSource.CLOSED:
        return '🔴';
      default:
        return '⚪';
    }
  };

  const getNotificationIcon = (type: Notification['type']) => {
    const icons = {
      info: 'ℹ️',
      warning: '⚠️',
      error: '❌',
      success: '✅'
    };
    return icons[type];
  };

  return (
    <div className="notifications-container">
      <div className="header">
        <h2>Live Notifications</h2>
        <span className="status">
          {getStatusIcon()} {isOpen ? 'Connected' : 'Disconnected'}
        </span>
      </div>

      <div className="notifications-list">
        {notifications.length === 0 ? (
          <p className="empty-state">No notifications yet</p>
        ) : (
          notifications.map((notification) => (
            <div key={notification.id} className={`notification ${notification.type}`}>
              <span className="icon">{getNotificationIcon(notification.type)}</span>
              <div className="content">
                <p>{notification.message}</p>
                <time>{new Date(notification.timestamp).toLocaleString()}</time>
              </div>
            </div>
          ))
        )}
      </div>
    </div>
  );
};

export default LiveNotifications;
```

## Advanced SSE Usage (React)

```typescript
// components/StockTicker.tsx
import React, { useState, useEffect } from 'react';

interface Stock {
  symbol: string;
  price: number;
  change: number;
  changePercent: number;
}

const StockTicker: React.FC<{ symbols: string[] }> = ({ symbols }) => {
  const [stocks, setStocks] = useState<Map<string, Stock>>(new Map());
  const [eventSource, setEventSource] = useState<EventSource | null>(null);

  useEffect(() => {
    const url = `/api/stocks/stream?symbols=${symbols.join(',')}`;
    const es = new EventSource(url);

    es.addEventListener('stock-update', (event) => {
      const stock: Stock = JSON.parse(event.data);
      setStocks(prev => new Map(prev).set(stock.symbol, stock));
    });

    es.addEventListener('market-status', (event) => {
      const status = JSON.parse(event.data);
      console.log('Market status:', status);
    });

    es.onerror = (error) => {
      console.error('Stock stream error:', error);
      // Automatically reconnects
    };

    setEventSource(es);

    return () => {
      es.close();
    };
  }, [symbols]);

  return (
    <div className="stock-ticker">
      <h2>Live Stock Prices</h2>
      <div className="stocks-grid">
        {Array.from(stocks.values()).map((stock) => (
          <div key={stock.symbol} className="stock-card">
            <div className="symbol">{stock.symbol}</div>
            <div className="price">${stock.price.toFixed(2)}</div>
            <div className={`change ${stock.change >= 0 ? 'positive' : 'negative'}`}>
              {stock.change >= 0 ? '+' : ''}
              {stock.change.toFixed(2)} ({stock.changePercent.toFixed(2)}%)
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};

export default StockTicker;
```

```typescript
// components/LiveActivityFeed.tsx
import React, { useState, useEffect, useRef } from 'react';

interface Activity {
  id: string;
  user: string;
  action: string;
  target: string;
  timestamp: string;
}

const LiveActivityFeed: React.FC = () => {
  const [activities, setActivities] = useState<Activity[]>([]);
  const [isPaused, setIsPaused] = useState(false);
  const eventSourceRef = useRef<EventSource | null>(null);
  const queueRef = useRef<Activity[]>([]);

  useEffect(() => {
    const es = new EventSource('/api/activities/stream');

    es.onmessage = (event) => {
      const activity: Activity = JSON.parse(event.data);
      
      if (isPaused) {
        // Queue activities while paused
        queueRef.current.push(activity);
      } else {
        setActivities(prev => [activity, ...prev].slice(0, 50));
      }
    };

    es.onerror = () => {
      console.error('Activity stream error');
    };

    eventSourceRef.current = es;

    return () => {
      es.close();
    };
  }, [isPaused]);

  const togglePause = () => {
    setIsPaused(prev => {
      if (prev) {
        // Resume: add queued activities
        setActivities(prevActivities => [
          ...queueRef.current,
          ...prevActivities
        ].slice(0, 50));
        queueRef.current = [];
      }
      return !prev;
    });
  };

  return (
    <div className="activity-feed">
      <div className="controls">
        <button onClick={togglePause}>
          {isPaused ? '▶️ Resume' : '⏸️ Pause'}
        </button>
        {queueRef.current.length > 0 && (
          <span className="queue-count">
            {queueRef.current.length} pending
          </span>
        )}
      </div>

      <div className="activities">
        {activities.map((activity) => (
          <div key={activity.id} className="activity">
            <strong>{activity.user}</strong> {activity.action}{' '}
            <em>{activity.target}</em>
            <time>{new Date(activity.timestamp).toLocaleTimeString()}</time>
          </div>
        ))}
      </div>
    </div>
  );
};

export default LiveActivityFeed;
```

## SSE with Angular

```typescript
// services/sse.service.ts
import { Injectable, NgZone } from '@angular/core';
import { Observable } from 'rxjs';

export interface SSEMessage {
  id?: string;
  event?: string;
  data: any;
}

@Injectable({
  providedIn: 'root'
})
export class SSEService {
  constructor(private zone: NgZone) {}

  getEventSource(url: string, withCredentials = false): Observable<SSEMessage> {
    return new Observable(observer => {
      const eventSource = new EventSource(url, { withCredentials });

      eventSource.onopen = () => {
        this.zone.run(() => {
          console.log('SSE connection opened');
        });
      };

      eventSource.onmessage = (event) => {
        this.zone.run(() => {
          try {
            const data = JSON.parse(event.data);
            observer.next({
              id: event.lastEventId,
              data
            });
          } catch (error) {
            observer.next({
              id: event.lastEventId,
              data: event.data
            });
          }
        });
      };

      eventSource.onerror = (error) => {
        this.zone.run(() => {
          observer.error(error);
        });
      };

      // Cleanup function
      return () => {
        eventSource.close();
      };
    });
  }

  getEventSourceWithCustomEvents(
    url: string,
    events: string[] = []
  ): Observable<SSEMessage> {
    return new Observable(observer => {
      const eventSource = new EventSource(url);

      // Handle default message event
      eventSource.onmessage = (event) => {
        this.zone.run(() => {
          observer.next({
            id: event.lastEventId,
            data: JSON.parse(event.data)
          });
        });
      };

      // Handle custom events
      events.forEach(eventName => {
        eventSource.addEventListener(eventName, (event: any) => {
          this.zone.run(() => {
            observer.next({
              id: event.lastEventId,
              event: eventName,
              data: JSON.parse(event.data)
            });
          });
        });
      });

      eventSource.onerror = (error) => {
        this.zone.run(() => {
          observer.error(error);
        });
      };

      return () => {
        eventSource.close();
      };
    });
  }
}
```

```typescript
// services/notification.service.ts
import { Injectable } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { SSEService, SSEMessage } from './sse.service';

export interface Notification {
  id: string;
  type: string;
  message: string;
  timestamp: string;
}

@Injectable({
  providedIn: 'root'
})
export class NotificationService {
  private notificationsSubject = new Subject<Notification>();
  public notifications$ = this.notificationsSubject.asObservable();

  constructor(private sseService: SSEService) {}

  connect() {
    this.sseService
      .getEventSourceWithCustomEvents('/api/notifications/stream', [
        'notification',
        'alert',
        'heartbeat'
      ])
      .subscribe({
        next: (message: SSEMessage) => {
          if (message.event === 'notification') {
            this.notificationsSubject.next(message.data);
          } else if (message.event === 'alert') {
            // Handle alerts differently
            this.handleAlert(message.data);
          }
        },
        error: (error) => {
          console.error('Notification stream error:', error);
        }
      });
  }

  private handleAlert(alert: any) {
    // Show alert to user
    alert(alert.message);
  }
}
```

```typescript
// components/notifications.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { NotificationService, Notification } from '../services/notification.service';

@Component({
  selector: 'app-notifications',
  template: `
    <div class="notifications-container">
      <h2>Live Notifications</h2>
      <div class="notifications-list">
        <div *ngFor="let notification of notifications" 
             [class]="'notification ' + notification.type">
          <p>{{ notification.message }}</p>
          <time>{{ notification.timestamp | date:'short' }}</time>
        </div>
        <p *ngIf="notifications.length === 0" class="empty-state">
          No notifications yet
        </p>
      </div>
    </div>
  `,
  styles: [`
    .notifications-container {
      padding: 20px;
    }
    .notification {
      padding: 10px;
      margin: 10px 0;
      border-radius: 4px;
    }
    .notification.info { background: #e3f2fd; }
    .notification.warning { background: #fff3e0; }
    .notification.error { background: #ffebee; }
    .notification.success { background: #e8f5e9; }
  `]
})
export class NotificationsComponent implements OnInit, OnDestroy {
  notifications: Notification[] = [];
  private subscription: Subscription;

  constructor(private notificationService: NotificationService) {}

  ngOnInit() {
    this.notificationService.connect();
    
    this.subscription = this.notificationService.notifications$.subscribe(
      notification => {
        this.notifications = [notification, ...this.notifications].slice(0, 20);
      }
    );
  }

  ngOnDestroy() {
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
  }
}
```

## Server-Side Implementation

### Node.js/Express SSE Server

```typescript
// server/sse-server.ts
import express, { Request, Response } from 'express';

const app = express();

interface Client {
  id: string;
  response: Response;
  lastEventId?: string;
}

const clients: Client[] = [];

// SSE endpoint
app.get('/api/notifications/stream', (req: Request, res: Response) => {
  // Set headers for SSE
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no' // Disable nginx buffering
  });

  // Create client
  const clientId = Date.now().toString();
  const client: Client = {
    id: clientId,
    response: res,
    lastEventId: req.headers['last-event-id'] as string
  };

  clients.push(client);
  console.log(`Client ${clientId} connected. Total clients: ${clients.length}`);

  // Send initial connection message
  sendEvent(res, 'connected', { clientId, timestamp: new Date() });

  // Send heartbeat every 30 seconds
  const heartbeatInterval = setInterval(() => {
    sendComment(res, 'heartbeat');
  }, 30000);

  // Handle client disconnect
  req.on('close', () => {
    console.log(`Client ${clientId} disconnected`);
    clearInterval(heartbeatInterval);
    
    const index = clients.findIndex(c => c.id === clientId);
    if (index !== -1) {
      clients.splice(index, 1);
    }
  });
});

// Stock prices stream
app.get('/api/stocks/stream', (req: Request, res: Response) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });

  const symbols = (req.query.symbols as string)?.split(',') || [];
  
  // Send stock updates every 2 seconds
  const interval = setInterval(() => {
    symbols.forEach(symbol => {
      const stock = {
        symbol,
        price: Math.random() * 1000,
        change: (Math.random() - 0.5) * 10,
        changePercent: (Math.random() - 0.5) * 5
      };
      
      sendEvent(res, 'stock-update', stock, Date.now().toString());
    });
  }, 2000);

  req.on('close', () => {
    clearInterval(interval);
  });
});

// Helper function to send SSE event
function sendEvent(
  res: Response,
  event: string,
  data: any,
  id?: string
) {
  if (id) {
    res.write(`id: ${id}\n`);
  }
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(data)}\n\n`);
}

// Helper function to send comment (heartbeat)
function sendComment(res: Response, comment: string) {
  res.write(`: ${comment}\n\n`);
}

// Broadcast to all clients
function broadcast(event: string, data: any) {
  clients.forEach(client => {
    sendEvent(client.response, event, data, Date.now().toString());
  });
}

// Example: Broadcast notifications
setInterval(() => {
  const notification = {
    id: Date.now().toString(),
    type: ['info', 'warning', 'success'][Math.floor(Math.random() * 3)],
    message: `Random notification at ${new Date().toLocaleTimeString()}`,
    timestamp: new Date().toISOString()
  };
  
  broadcast('notification', notification);
}, 10000);

const PORT = 3001;
app.listen(PORT, () => {
  console.log(`SSE server running on http://localhost:${PORT}`);
});
```

### Advanced SSE Server with Redis

```typescript
// server/sse-redis-server.ts
import express, { Request, Response } from 'express';
import Redis from 'ioredis';

const app = express();
const redis = new Redis();
const subscriber = new Redis();

interface Client {
  id: string;
  response: Response;
  channels: Set<string>;
}

const clients = new Map<string, Client>();

// Subscribe to Redis pub/sub
subscriber.on('message', (channel, message) => {
  console.log(`Received message on channel ${channel}`);
  
  // Broadcast to all clients subscribed to this channel
  clients.forEach(client => {
    if (client.channels.has(channel)) {
      try {
        const data = JSON.parse(message);
        sendEvent(client.response, channel, data);
      } catch (error) {
        console.error('Failed to parse message:', error);
      }
    }
  });
});

// SSE endpoint with channel subscription
app.get('/api/stream/:channel', (req: Request, res: Response) => {
  const channel = req.params.channel;
  
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });

  const clientId = Date.now().toString();
  const client: Client = {
    id: clientId,
    response: res,
    channels: new Set([channel])
  };

  clients.set(clientId, client);

  // Subscribe to Redis channel
  if (Array.from(clients.values()).filter(c => c.channels.has(channel)).length === 1) {
    subscriber.subscribe(channel);
    console.log(`Subscribed to Redis channel: ${channel}`);
  }

  console.log(`Client ${clientId} connected to channel ${channel}`);

  req.on('close', () => {
    clients.delete(clientId);
    
    // Unsubscribe from Redis if no clients are listening
    if (!Array.from(clients.values()).some(c => c.channels.has(channel))) {
      subscriber.unsubscribe(channel);
      console.log(`Unsubscribed from Redis channel: ${channel}`);
    }
  });
});

// Publish endpoint (for testing)
app.post('/api/publish/:channel', express.json(), async (req, res) => {
  const channel = req.params.channel;
  const message = req.body;
  
  await redis.publish(channel, JSON.stringify(message));
  
  res.json({ success: true, channel, message });
});

function sendEvent(res: Response, event: string, data: any) {
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(data)}\n\n`);
}

const PORT = 3002;
app.listen(PORT, () => {
  console.log(`SSE Redis server running on http://localhost:${PORT}`);
});
```

## Common Mistakes

### Mistake 1: Not Setting Correct Headers

```typescript
// BAD: Missing or incorrect headers
app.get('/events', (req, res) => {
  res.send('data: Hello\n\n'); // Wrong!
});

// GOOD: Correct SSE headers
app.get('/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });
  res.write('data: Hello\n\n');
});
```

### Mistake 2: Not Handling Reconnections

```typescript
// BAD: No reconnection handling
const es = new EventSource('/events');
// If connection drops, no recovery

// GOOD: Handle reconnections with Last-Event-ID
const es = new EventSource('/events');
es.addEventListener('message', (event) => {
  localStorage.setItem('lastEventId', event.lastEventId);
});

// Server reads Last-Event-ID header for replay
app.get('/events', (req, res) => {
  const lastEventId = req.headers['last-event-id'];
  // Replay events after lastEventId
});
```

### Mistake 3: Memory Leaks from Not Closing Connections

```typescript
// BAD: Not cleaning up
useEffect(() => {
  const es = new EventSource('/events');
  // Missing cleanup!
}, []);

// GOOD: Close connection on unmount
useEffect(() => {
  const es = new EventSource('/events');
  
  return () => {
    es.close();
  };
}, []);
```

## Best Practices

1. Always set correct Content-Type header (text/event-stream)
2. Implement heartbeat mechanism to detect dead connections
3. Use Last-Event-ID for connection recovery
4. Clean up EventSource connections on component unmount
5. Handle connection errors with exponential backoff
6. Use Redis pub/sub for horizontal scaling
7. Set proper Cache-Control headers (no-cache)
8. Implement connection timeouts on the server
9. Use comments for heartbeats to keep connection alive
10. Validate and sanitize data before sending

## Key Takeaways

1. SSE is unidirectional (server to client only)
2. Built on HTTP, easier to implement than WebSockets
3. Automatic reconnection with Last-Event-ID support
4. Works with existing HTTP infrastructure (proxies, load balancers)
5. Limited to text data (JSON encoded)
6. Maximum 6 concurrent connections per browser domain
7. No binary data support (use base64 encoding if needed)
8. Better than polling for real-time updates from server

## Interview Questions

1. What is the difference between SSE and WebSockets?
2. How does SSE handle reconnection?
3. What is the Last-Event-ID header used for?
4. What are the limitations of SSE?
5. How do you scale SSE across multiple servers?
6. When would you choose SSE over WebSockets?
7. How does SSE work with HTTP/2?
8. What is the maximum number of SSE connections per domain?
9. How do you implement heartbeats in SSE?
10. How do you handle binary data with SSE?

## Resources

- Server-Sent Events MDN Documentation
- EventSource API Specification
- SSE vs WebSockets Comparison
- Redis Pub/Sub for SSE Scaling
- HTTP/2 Server Push vs SSE
- Nginx SSE Configuration
