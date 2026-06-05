# State Synchronization

## The Idea

**In plain English:** State synchronization means making sure that when data changes in one place (like on a server or another person's screen), all other places that show that data automatically update to match. Think of it like making sure every copy of a shared document shows the same words at the same time.

**Real-world analogy:** Imagine a scoreboard at a live sports game. The scoreboard operator updates the physical board whenever a team scores, and TV broadcasters instantly show the same updated score on screen for millions of viewers at home.

- The scoreboard = the server (the source of truth for the current score)
- The TV broadcast = the client app (displaying the state to the user)
- The live feed sending score updates to broadcasters = the synchronization mechanism (polling, WebSockets, or SSE)

---

## Overview

State synchronization ensures that client-side application state remains consistent with server-side data sources. As applications grow more complex with multiple users, devices, and real-time requirements, keeping state in sync becomes critical. This involves strategies like polling, WebSockets, Server-Sent Events (SSE), and optimistic updates with conflict resolution. Proper synchronization prevents stale data, race conditions, and provides users with up-to-date information.

## Core Concepts

### Synchronization Strategies

```
┌──────────────────────────────────────────────────────────┐
│         State Synchronization Strategies                  │
└──────────────────────────────────────────────────────────┘

POLLING
Client ────GET────> Server
       <───Data────
       (every N seconds)

LONG POLLING  
Client ────GET────> Server (holds connection)
       <───Data──── (when data changes)
       ────GET────> (immediately reconnect)

WEBSOCKETS
Client <══════════> Server
       (bidirectional, persistent)

SSE (Server-Sent Events)
Client <───stream─── Server
       (unidirectional, server → client)

WEBHOOKS
Server ────POST───> Client Endpoint
       (server-initiated push)
```

## Polling

### Basic Polling with React

```typescript
import React, { useState, useEffect, useRef } from 'react';

interface Message {
  id: string;
  text: string;
  timestamp: number;
}

function PollingMessages() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [isPolling, setIsPolling] = useState(true);
  const intervalRef = useRef<number>();

  useEffect(() => {
    const fetchMessages = async () => {
      try {
        const response = await fetch('/api/messages');
        if (!response.ok) throw new Error('Fetch failed');
        
        const data = await response.json();
        setMessages(data);
        setError(null);
      } catch (err) {
        setError(err.message);
      }
    };

    if (isPolling) {
      // Initial fetch
      fetchMessages();
      
      // Poll every 5 seconds
      intervalRef.current = window.setInterval(fetchMessages, 5000);
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isPolling]);

  return (
    <div>
      <button onClick={() => setIsPolling(!isPolling)}>
        {isPolling ? 'Stop Polling' : 'Start Polling'}
      </button>
      
      {error && <div className="error">{error}</div>}
      
      <ul>
        {messages.map(msg => (
          <li key={msg.id}>{msg.text}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Smart Polling with React Query

```typescript
import { useQuery } from '@tanstack/react-query';
import { useEffect, useState } from 'react';

interface Task {
  id: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  progress: number;
}

function TaskMonitor({ taskId }: { taskId: string }) {
  const { data: task, refetch } = useQuery({
    queryKey: ['task', taskId],
    queryFn: async () => {
      const response = await fetch(`/api/tasks/${taskId}`);
      return response.json() as Promise<Task>;
    },
    // Poll every 2 seconds only while task is active
    refetchInterval: (data) => {
      if (!data) return false;
      return data.status === 'pending' || data.status === 'running'
        ? 2000
        : false;
    },
    // Stop polling when window is not focused
    refetchIntervalInBackground: false
  });

  return (
    <div>
      <h3>Task Status: {task?.status}</h3>
      {task && (
        <progress value={task.progress} max={100}>
          {task.progress}%
        </progress>
      )}
      {task?.status === 'completed' && <span>✓ Done!</span>}
      {task?.status === 'failed' && (
        <span>✗ Failed - <button onClick={() => refetch()}>Retry</button></span>
      )}
    </div>
  );
}
```

### Adaptive Polling

```typescript
class AdaptivePoller {
  private intervalId: number | null = null;
  private currentInterval: number;
  private minInterval = 1000;
  private maxInterval = 30000;
  private backoffMultiplier = 1.5;
  private lastDataChangeTime = Date.now();
  
  constructor(
    private fetchFn: () => Promise<any>,
    private onData: (data: any) => void,
    private onError: (error: Error) => void
  ) {
    this.currentInterval = this.minInterval;
  }

  start() {
    this.poll();
  }

  stop() {
    if (this.intervalId) {
      clearTimeout(this.intervalId);
      this.intervalId = null;
    }
  }

  private async poll() {
    try {
      const data = await this.fetchFn();
      this.onData(data);
      
      // If data changed, reset to min interval
      const now = Date.now();
      if (this.hasDataChanged(data)) {
        this.lastDataChangeTime = now;
        this.currentInterval = this.minInterval;
      } else {
        // If no change, gradually increase interval
        const timeSinceChange = now - this.lastDataChangeTime;
        if (timeSinceChange > 10000) {
          this.currentInterval = Math.min(
            this.currentInterval * this.backoffMultiplier,
            this.maxInterval
          );
        }
      }
    } catch (error) {
      this.onError(error as Error);
      // On error, back off
      this.currentInterval = Math.min(
        this.currentInterval * 2,
        this.maxInterval
      );
    }

    this.intervalId = window.setTimeout(
      () => this.poll(),
      this.currentInterval
    );
  }

  private hasDataChanged(newData: any): boolean {
    // Implement your comparison logic
    return true;
  }
}

function AdaptivePollingComponent() {
  const [messages, setMessages] = useState([]);
  const pollerRef = useRef<AdaptivePoller>();

  useEffect(() => {
    pollerRef.current = new AdaptivePoller(
      () => fetch('/api/messages').then(r => r.json()),
      (data) => setMessages(data),
      (error) => console.error(error)
    );

    pollerRef.current.start();

    return () => pollerRef.current?.stop();
  }, []);

  return <div>{/* Render messages */}</div>;
}
```

## WebSocket Synchronization

### Basic WebSocket with React

```typescript
import React, { useState, useEffect, useRef } from 'react';

interface Message {
  id: string;
  type: 'message' | 'update' | 'delete';
  data: any;
}

function useWebSocket(url: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [connectionState, setConnectionState] = useState<
    'connecting' | 'open' | 'closed' | 'error'
  >('connecting');
  const ws = useRef<WebSocket | null>(null);
  const reconnectTimeout = useRef<number>();
  const reconnectAttempts = useRef(0);

  useEffect(() => {
    connect();

    return () => {
      if (ws.current) {
        ws.current.close();
      }
      if (reconnectTimeout.current) {
        clearTimeout(reconnectTimeout.current);
      }
    };
  }, [url]);

  const connect = () => {
    try {
      ws.current = new WebSocket(url);

      ws.current.onopen = () => {
        console.log('WebSocket connected');
        setConnectionState('open');
        reconnectAttempts.current = 0;
      };

      ws.current.onmessage = (event) => {
        const message: Message = JSON.parse(event.data);
        setMessages(prev => [...prev, message]);
      };

      ws.current.onerror = (error) => {
        console.error('WebSocket error:', error);
        setConnectionState('error');
      };

      ws.current.onclose = () => {
        console.log('WebSocket closed');
        setConnectionState('closed');
        
        // Exponential backoff reconnection
        const delay = Math.min(
          1000 * Math.pow(2, reconnectAttempts.current),
          30000
        );
        reconnectAttempts.current++;
        
        reconnectTimeout.current = window.setTimeout(connect, delay);
      };
    } catch (error) {
      console.error('Failed to create WebSocket:', error);
      setConnectionState('error');
    }
  };

  const send = (data: any) => {
    if (ws.current && ws.current.readyState === WebSocket.OPEN) {
      ws.current.send(JSON.stringify(data));
    } else {
      console.error('WebSocket is not connected');
    }
  };

  return { messages, connectionState, send };
}

function RealtimeChat() {
  const { messages, connectionState, send } = useWebSocket(
    'ws://localhost:3000/chat'
  );
  const [input, setInput] = useState('');

  const handleSend = () => {
    if (input.trim()) {
      send({
        type: 'message',
        text: input,
        timestamp: Date.now()
      });
      setInput('');
    }
  };

  return (
    <div>
      <div className={`connection-status ${connectionState}`}>
        {connectionState === 'open' && '🟢 Connected'}
        {connectionState === 'connecting' && '🟡 Connecting...'}
        {connectionState === 'closed' && '🔴 Disconnected'}
        {connectionState === 'error' && '❌ Error'}
      </div>

      <div className="messages">
        {messages.map((msg, index) => (
          <div key={index}>{msg.data.text}</div>
        ))}
      </div>

      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        onKeyPress={(e) => e.key === 'Enter' && handleSend()}
        disabled={connectionState !== 'open'}
      />
      <button onClick={handleSend} disabled={connectionState !== 'open'}>
        Send
      </button>
    </div>
  );
}
```

### WebSocket State Sync with Redux

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { Middleware } from 'redux';

interface Document {
  id: string;
  content: string;
  version: number;
}

const documentSlice = createSlice({
  name: 'document',
  initialState: {
    current: null as Document | null,
    syncing: false
  },
  reducers: {
    setDocument: (state, action: PayloadAction<Document>) => {
      state.current = action.payload;
    },
    updateContent: (state, action: PayloadAction<string>) => {
      if (state.current) {
        state.current.content = action.payload;
        state.syncing = true;
      }
    },
    syncComplete: (state) => {
      state.syncing = false;
    },
    serverUpdate: (state, action: PayloadAction<Document>) => {
      // Only apply if server version is newer
      if (
        !state.current ||
        action.payload.version > state.current.version
      ) {
        state.current = action.payload;
      }
    }
  }
});

export const { setDocument, updateContent, syncComplete, serverUpdate } =
  documentSlice.actions;

// WebSocket Middleware
export const websocketMiddleware: Middleware = (store) => {
  let ws: WebSocket | null = null;

  return (next) => (action) => {
    if (action.type === 'WS_CONNECT') {
      ws = new WebSocket(action.payload.url);

      ws.onmessage = (event) => {
        const message = JSON.parse(event.data);
        
        switch (message.type) {
          case 'document_update':
            store.dispatch(serverUpdate(message.data));
            break;
          case 'sync_ack':
            store.dispatch(syncComplete());
            break;
        }
      };

      ws.onopen = () => {
        console.log('WebSocket connected');
      };
    }

    if (action.type === updateContent.type && ws) {
      const state = store.getState();
      ws.send(JSON.stringify({
        type: 'update',
        documentId: state.document.current?.id,
        content: action.payload,
        version: state.document.current?.version
      }));
    }

    if (action.type === 'WS_DISCONNECT' && ws) {
      ws.close();
      ws = null;
    }

    return next(action);
  };
};

export default documentSlice.reducer;
```

### Angular WebSocket Service

```typescript
import { Injectable } from '@angular/core';
import { Observable, Subject, timer } from 'rxjs';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { retry, retryWhen, tap, delayWhen } from 'rxjs/operators';

interface Message {
  type: string;
  data: any;
}

@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private socket$: WebSocketSubject<Message> | null = null;
  private messagesSubject = new Subject<Message>();
  public messages$ = this.messagesSubject.asObservable();

  private connectionStatusSubject = new Subject<string>();
  public connectionStatus$ = this.connectionStatusSubject.asObservable();

  constructor() {}

  connect(url: string): void {
    if (this.socket$) {
      return; // Already connected
    }

    this.socket$ = webSocket<Message>({
      url,
      openObserver: {
        next: () => {
          console.log('WebSocket connected');
          this.connectionStatusSubject.next('connected');
        }
      },
      closeObserver: {
        next: () => {
          console.log('WebSocket closed');
          this.connectionStatusSubject.next('disconnected');
          this.socket$ = null;
        }
      }
    });

    // Subscribe with automatic reconnection
    this.socket$
      .pipe(
        tap(message => this.messagesSubject.next(message)),
        retryWhen(errors =>
          errors.pipe(
            tap(err => console.error('WebSocket error:', err)),
            delayWhen((_, index) => {
              const delay = Math.min(1000 * Math.pow(2, index), 30000);
              console.log(`Reconnecting in ${delay}ms...`);
              return timer(delay);
            })
          )
        )
      )
      .subscribe();
  }

  send(message: Message): void {
    if (this.socket$) {
      this.socket$.next(message);
    } else {
      console.error('WebSocket is not connected');
    }
  }

  disconnect(): void {
    if (this.socket$) {
      this.socket$.complete();
      this.socket$ = null;
    }
  }
}

// Component usage
import { Component, OnInit, OnDestroy } from '@angular/core';

@Component({
  selector: 'app-realtime-data',
  template: `
    <div>
      <div class="status">
        Status: {{ connectionStatus$ | async }}
      </div>
      
      <div *ngFor="let message of messages$ | async">
        {{ message.data }}
      </div>
    </div>
  `
})
export class RealtimeDataComponent implements OnInit, OnDestroy {
  messages$ = this.wsService.messages$;
  connectionStatus$ = this.wsService.connectionStatus$;

  constructor(private wsService: WebSocketService) {}

  ngOnInit() {
    this.wsService.connect('ws://localhost:3000');
  }

  ngOnDestroy() {
    this.wsService.disconnect();
  }
}
```

## Server-Sent Events (SSE)

### Basic SSE Implementation

```typescript
import React, { useState, useEffect } from 'react';

interface Event {
  id: string;
  type: string;
  data: any;
}

function useServerSentEvents(url: string) {
  const [events, setEvents] = useState<Event[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [readyState, setReadyState] = useState<number>(EventSource.CONNECTING);

  useEffect(() => {
    const eventSource = new EventSource(url);

    eventSource.onopen = () => {
      console.log('SSE connected');
      setReadyState(EventSource.OPEN);
      setError(null);
    };

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setEvents(prev => [...prev, {
        id: event.lastEventId,
        type: 'message',
        data
      }]);
    };

    // Custom event handlers
    eventSource.addEventListener('update', (event: any) => {
      const data = JSON.parse(event.data);
      setEvents(prev => [...prev, {
        id: event.lastEventId,
        type: 'update',
        data
      }]);
    });

    eventSource.onerror = () => {
      console.error('SSE error');
      setReadyState(EventSource.CLOSED);
      setError('Connection lost. Reconnecting...');
      // EventSource automatically reconnects
    };

    return () => {
      eventSource.close();
    };
  }, [url]);

  return { events, error, readyState };
}

function LiveFeed() {
  const { events, error, readyState } = useServerSentEvents('/api/events');

  return (
    <div>
      <div className="status">
        {readyState === EventSource.CONNECTING && 'Connecting...'}
        {readyState === EventSource.OPEN && '🟢 Live'}
        {readyState === EventSource.CLOSED && '🔴 Disconnected'}
      </div>

      {error && <div className="error">{error}</div>}

      <div className="feed">
        {events.map((event, index) => (
          <div key={index} className={`event event-${event.type}`}>
            {JSON.stringify(event.data)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### SSE with State Updates

```typescript
function useSSEState<T>(url: string, initialState: T) {
  const [state, setState] = useState<T>(initialState);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const eventSource = new EventSource(url);

    eventSource.addEventListener('state', (event: any) => {
      try {
        const newState = JSON.parse(event.data);
        setState(newState);
        setError(null);
      } catch (err) {
        setError('Failed to parse state update');
      }
    });

    eventSource.addEventListener('patch', (event: any) => {
      try {
        const patch = JSON.parse(event.data);
        setState(prev => ({ ...prev, ...patch }));
        setError(null);
      } catch (err) {
        setError('Failed to apply patch');
      }
    });

    eventSource.onerror = () => {
      setError('Connection error');
    };

    return () => eventSource.close();
  }, [url]);

  return { state, error };
}

interface DashboardData {
  activeUsers: number;
  revenue: number;
  orders: number;
}

function Dashboard() {
  const { state, error } = useSSEState<DashboardData>(
    '/api/dashboard/stream',
    { activeUsers: 0, revenue: 0, orders: 0 }
  );

  return (
    <div>
      {error && <div className="error">{error}</div>}
      
      <div className="metrics">
        <div>Active Users: {state.activeUsers}</div>
        <div>Revenue: ${state.revenue}</div>
        <div>Orders: {state.orders}</div>
      </div>
    </div>
  );
}
```

## Conflict Resolution

### Operational Transform (OT)

```typescript
// Simple OT for text editing
type Operation =
  | { type: 'insert'; position: number; text: string }
  | { type: 'delete'; position: number; length: number };

function transform(op1: Operation, op2: Operation): Operation {
  if (op1.type === 'insert' && op2.type === 'insert') {
    if (op1.position < op2.position) {
      return op2;
    } else if (op1.position > op2.position) {
      return {
        ...op2,
        position: op2.position + op1.text.length
      };
    } else {
      // Same position - prioritize by timestamp or client ID
      return op2;
    }
  }

  if (op1.type === 'delete' && op2.type === 'insert') {
    if (op2.position <= op1.position) {
      return op2;
    } else if (op2.position >= op1.position + op1.length) {
      return {
        ...op2,
        position: op2.position - op1.length
      };
    } else {
      return {
        ...op2,
        position: op1.position
      };
    }
  }

  // Implement other combinations...
  return op2;
}

class CollaborativeEditor {
  private text = '';
  private pendingOperations: Operation[] = [];

  applyOperation(op: Operation): void {
    switch (op.type) {
      case 'insert':
        this.text =
          this.text.slice(0, op.position) +
          op.text +
          this.text.slice(op.position);
        break;
      case 'delete':
        this.text =
          this.text.slice(0, op.position) +
          this.text.slice(op.position + op.length);
        break;
    }
  }

  sendOperation(op: Operation): void {
    this.applyOperation(op);
    this.pendingOperations.push(op);
    // Send to server
    this.syncWithServer(op);
  }

  receiveOperation(op: Operation): void {
    // Transform against pending operations
    let transformedOp = op;
    for (const pending of this.pendingOperations) {
      transformedOp = transform(pending, transformedOp);
    }
    this.applyOperation(transformedOp);
  }

  private async syncWithServer(op: Operation): Promise<void> {
    try {
      await fetch('/api/operations', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(op)
      });
      
      // Remove from pending once confirmed
      const index = this.pendingOperations.indexOf(op);
      if (index > -1) {
        this.pendingOperations.splice(index, 1);
      }
    } catch (error) {
      console.error('Failed to sync operation:', error);
    }
  }

  getText(): string {
    return this.text;
  }
}
```

### Last-Write-Wins (LWW)

```typescript
interface TimestampedValue<T> {
  value: T;
  timestamp: number;
  clientId: string;
}

class LWWRegister<T> {
  private current: TimestampedValue<T>;

  constructor(initialValue: T, clientId: string) {
    this.current = {
      value: initialValue,
      timestamp: Date.now(),
      clientId
    };
  }

  set(value: T, clientId: string): void {
    const timestamp = Date.now();
    this.merge({
      value,
      timestamp,
      clientId
    });
  }

  merge(incoming: TimestampedValue<T>): void {
    if (incoming.timestamp > this.current.timestamp) {
      this.current = incoming;
    } else if (incoming.timestamp === this.current.timestamp) {
      // Break tie using client ID
      if (incoming.clientId > this.current.clientId) {
        this.current = incoming;
      }
    }
  }

  get(): T {
    return this.current.value;
  }

  getWithMetadata(): TimestampedValue<T> {
    return this.current;
  }
}

// Usage in state sync
function useLWWState<T>(
  initialValue: T,
  syncUrl: string,
  clientId: string
) {
  const register = useRef(new LWWRegister(initialValue, clientId));
  const [value, setValue] = useState(initialValue);

  useEffect(() => {
    const ws = new WebSocket(syncUrl);

    ws.onmessage = (event) => {
      const incoming = JSON.parse(event.data);
      register.current.merge(incoming);
      setValue(register.current.get());
    };

    return () => ws.close();
  }, [syncUrl]);

  const updateValue = (newValue: T) => {
    register.current.set(newValue, clientId);
    setValue(newValue);

    // Send to other clients
    fetch(syncUrl, {
      method: 'POST',
      body: JSON.stringify(register.current.getWithMetadata())
    });
  };

  return [value, updateValue] as const;
}
```

## Common Mistakes

### 1. Polling Too Frequently

```typescript
// ❌ Wrong - polls every 100ms, wasteful
useEffect(() => {
  const interval = setInterval(fetchData, 100);
  return () => clearInterval(interval);
}, []);

// ✅ Correct - reasonable interval
useEffect(() => {
  const interval = setInterval(fetchData, 5000);
  return () => clearInterval(interval);
}, []);
```

### 2. Not Cleaning Up WebSocket Connections

```typescript
// ❌ Wrong - WebSocket leak
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = handleMessage;
}, []);

// ✅ Correct - cleanup
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = handleMessage;
  return () => ws.close();
}, []);
```

### 3. Ignoring Connection State

```typescript
// ❌ Wrong - sends without checking state
const send = (data) => {
  ws.send(JSON.stringify(data));
};

// ✅ Correct - check connection state
const send = (data) => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify(data));
  } else {
    console.error('WebSocket not ready');
  }
};
```

### 4. Not Handling Reconnection

```typescript
// ❌ Wrong - no reconnection logic
ws.onclose = () => {
  console.log('Disconnected');
};

// ✅ Correct - exponential backoff reconnection
ws.onclose = () => {
  const delay = Math.min(1000 * Math.pow(2, attempts), 30000);
  setTimeout(reconnect, delay);
  attempts++;
};
```

### 5. Race Conditions with Multiple Sync Methods

```typescript
// ❌ Wrong - polling and WebSocket both updating
usePolling(url);
useWebSocket(wsUrl);

// ✅ Correct - choose one or coordinate
const syncMethod = useOnlineStatus() ? 'websocket' : 'polling';
if (syncMethod === 'websocket') {
  useWebSocket(wsUrl);
} else {
  usePolling(url);
}
```

## Best Practices

### 1. Choose the Right Strategy

- **Polling**: Simple, works everywhere, higher latency
- **WebSockets**: Real-time, bidirectional, complex
- **SSE**: Unidirectional, simpler than WebSockets, great for updates
- **Long Polling**: Fallback for old browsers

### 2. Implement Exponential Backoff

```typescript
const reconnect = (attempts: number) => {
  const delay = Math.min(1000 * Math.pow(2, attempts), 30000);
  setTimeout(connect, delay);
};
```

### 3. Handle Offline/Online Transitions

```typescript
useEffect(() => {
  const handleOnline = () => {
    connect();
  };
  
  const handleOffline = () => {
    disconnect();
  };
  
  window.addEventListener('online', handleOnline);
  window.addEventListener('offline', handleOffline);
  
  return () => {
    window.removeEventListener('online', handleOnline);
    window.removeEventListener('offline', handleOffline);
  };
}, []);
```

### 4. Show Connection State to Users

```typescript
<div className={`status ${connectionState}`}>
  {connectionState === 'connected' && '🟢 Live'}
  {connectionState === 'connecting' && '🟡 Connecting...'}
  {connectionState === 'disconnected' && '🔴 Offline'}
</div>
```

## When to Use Each Strategy

### Polling:
- Updates are infrequent
- Simplicity is key
- Legacy browser support needed
- Firewall/proxy restrictions

### WebSockets:
- Real-time bidirectional communication
- Chat, collaborative editing
- Gaming, live trading
- High-frequency updates

### SSE:
- Unidirectional server → client
- Live feeds, notifications
- Simpler than WebSockets
- Automatic reconnection needed

### Long Polling:
- WebSocket fallback
- Corporate firewalls
- Old browser support

## Interview Questions

### Q1: What are the trade-offs between polling and WebSockets?
**Answer**: Polling is simpler, works everywhere, but has higher latency and server load. WebSockets provide real-time bidirectional communication with lower overhead but are more complex, require WebSocket support, and can have firewall/proxy issues. Choose polling for infrequent updates, WebSockets for real-time needs.

### Q2: How do you handle WebSocket reconnection?
**Answer**: Implement exponential backoff to avoid overwhelming the server. Track reconnection attempts and cap the maximum delay. Listen to online/offline events to reconnect when network returns. Store pending messages and resend after reconnection. Show connection state to users.

### Q3: What is Operational Transform?
**Answer**: OT is an algorithm for resolving conflicts in collaborative editing. When two clients make concurrent changes, their operations are transformed relative to each other to maintain consistency. Operations are applied locally immediately, then transformed against concurrent operations when syncing.

### Q4: When should you use SSE vs WebSockets?
**Answer**: Use SSE for unidirectional server → client updates (notifications, live feeds, dashboards). It's simpler than WebSockets, has automatic reconnection, and works over HTTP. Use WebSockets for bidirectional communication (chat, collaborative tools) or very high-frequency updates.

### Q5: How do you prevent race conditions in state sync?
**Answer**: Use version numbers or timestamps to detect conflicts. Implement conflict resolution strategies (OT, CRDTs, last-write-wins). Cancel inflight requests when new ones start. Use optimistic updates with rollback. Queue operations to ensure order.

## Key Takeaways

1. Choose sync strategy based on update frequency and latency requirements
2. Polling is simple but inefficient for frequent updates
3. WebSockets enable real-time bidirectional communication
4. SSE is great for unidirectional server → client updates
5. Always implement reconnection logic with exponential backoff
6. Show connection state clearly to users
7. Handle offline/online transitions gracefully
8. Implement conflict resolution for concurrent updates
9. Use optimistic updates for better UX
10. Consider battery and bandwidth when choosing strategy

## Resources

- [WebSocket API MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [Server-Sent Events Guide](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [Operational Transform](https://operational-transformation.github.io/)
- [CRDTs Explained](https://crdt.tech/)
- [React Query Real-time Updates](https://tanstack.com/query/latest/docs/react/guides/window-focus-refetching)
- [WebSocket Best Practices](https://www.nginx.com/blog/websocket-nginx/)
