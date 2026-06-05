# WebSockets

## The Idea

**In plain English:** A WebSocket is a permanent, two-way telephone line between your browser and a server — once the call is connected, either side can talk at any moment without needing to redial. Unlike a normal web request (where you ask a question and wait for one answer), a WebSocket connection stays open so the server can push new information to you instantly.

**Real-world analogy:** Imagine a walkie-talkie conversation between two people on a film set. Either person can press the button and speak at any time without waiting for the other to finish — the channel stays open all day.

- The walkie-talkie channel staying open all day = the persistent WebSocket connection
- Either person pressing the button and speaking = the client or server sending a message at any time
- The initial "Radio check, do you copy?" to confirm both sides are on the same frequency = the HTTP handshake that upgrades to a WebSocket connection

---

## Overview

WebSockets provide full-duplex, bidirectional communication between client and server over a single TCP connection. Unlike HTTP's request-response model, WebSockets allow both parties to send messages independently at any time. This makes them ideal for real-time applications like chat, live notifications, collaborative editing, and gaming.

## WebSocket Protocol

```
HTTP vs WebSocket Connection:

HTTP (Request-Response):
Client → Server: Request
Client ← Server: Response
[Connection closed]

WebSocket (Bidirectional):
Client ↔ Server: Handshake (HTTP Upgrade)
Client ↔ Server: Message
Client ↔ Server: Message
Client ↔ Server: Message
[Connection stays open]
```

### WebSocket Handshake

```
WebSocket Handshake Flow:
┌─────────┐                    ┌─────────┐
│ Client  │                    │ Server  │
└────┬────┘                    └────┬────┘
     │                              │
     │ GET /socket HTTP/1.1         │
     │ Upgrade: websocket           │
     │ Connection: Upgrade          │
     │ Sec-WebSocket-Key: xyz...    │
     │──────────────────────────────>│
     │                              │
     │ HTTP/1.1 101 Switching       │
     │ Upgrade: websocket           │
     │ Connection: Upgrade          │
     │ Sec-WebSocket-Accept: abc... │
     │<──────────────────────────────│
     │                              │
     │ ↔ WebSocket Messages ↔       │
     │                              │
```

## Native WebSocket API (React)

```typescript
// hooks/useWebSocket.ts
import { useEffect, useRef, useState, useCallback } from 'react';

interface UseWebSocketOptions {
  url: string;
  onMessage?: (data: any) => void;
  onOpen?: () => void;
  onClose?: () => void;
  onError?: (error: Event) => void;
  reconnect?: boolean;
  reconnectAttempts?: number;
  reconnectInterval?: number;
}

export const useWebSocket = ({
  url,
  onMessage,
  onOpen,
  onClose,
  onError,
  reconnect = true,
  reconnectAttempts = 5,
  reconnectInterval = 3000
}: UseWebSocketOptions) => {
  const [readyState, setReadyState] = useState<number>(WebSocket.CONNECTING);
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectCountRef = useRef(0);
  const reconnectTimeoutRef = useRef<NodeJS.Timeout>();

  const connect = useCallback(() => {
    try {
      const ws = new WebSocket(url);
      
      ws.onopen = () => {
        console.log('WebSocket connected');
        setReadyState(WebSocket.OPEN);
        reconnectCountRef.current = 0;
        onOpen?.();
      };

      ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          onMessage?.(data);
        } catch (error) {
          console.error('Failed to parse message:', error);
        }
      };

      ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        onError?.(error);
      };

      ws.onclose = () => {
        console.log('WebSocket closed');
        setReadyState(WebSocket.CLOSED);
        onClose?.();

        // Attempt reconnection
        if (reconnect && reconnectCountRef.current < reconnectAttempts) {
          reconnectCountRef.current++;
          console.log(`Reconnecting... Attempt ${reconnectCountRef.current}`);
          
          reconnectTimeoutRef.current = setTimeout(() => {
            connect();
          }, reconnectInterval);
        }
      };

      wsRef.current = ws;
    } catch (error) {
      console.error('Failed to create WebSocket:', error);
    }
  }, [url, onMessage, onOpen, onClose, onError, reconnect, reconnectAttempts, reconnectInterval]);

  const send = useCallback((data: any) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    } else {
      console.warn('WebSocket is not connected');
    }
  }, []);

  const close = useCallback(() => {
    if (reconnectTimeoutRef.current) {
      clearTimeout(reconnectTimeoutRef.current);
    }
    wsRef.current?.close();
  }, []);

  useEffect(() => {
    connect();

    return () => {
      close();
    };
  }, [connect, close]);

  return {
    send,
    close,
    readyState,
    isOpen: readyState === WebSocket.OPEN,
    isConnecting: readyState === WebSocket.CONNECTING,
    isClosed: readyState === WebSocket.CLOSED
  };
};
```

```typescript
// components/Chat.tsx
import React, { useState, useEffect, useRef } from 'react';
import { useWebSocket } from '../hooks/useWebSocket';

interface Message {
  id: string;
  userId: string;
  username: string;
  text: string;
  timestamp: string;
}

const Chat: React.FC = () => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputText, setInputText] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const { send, isOpen } = useWebSocket({
    url: 'ws://localhost:8080/chat',
    onMessage: (data) => {
      if (data.type === 'message') {
        setMessages(prev => [...prev, data.message]);
      } else if (data.type === 'history') {
        setMessages(data.messages);
      }
    },
    onOpen: () => {
      console.log('Connected to chat');
    },
    onClose: () => {
      console.log('Disconnected from chat');
    },
    reconnect: true,
    reconnectAttempts: 5
  });

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const sendMessage = () => {
    if (inputText.trim() && isOpen) {
      send({
        type: 'message',
        text: inputText
      });
      setInputText('');
    }
  };

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      sendMessage();
    }
  };

  return (
    <div className="chat-container">
      <div className="connection-status">
        {isOpen ? '🟢 Connected' : '🔴 Disconnected'}
      </div>
      
      <div className="messages">
        {messages.map((msg) => (
          <div key={msg.id} className="message">
            <strong>{msg.username}:</strong> {msg.text}
            <span className="timestamp">{msg.timestamp}</span>
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>

      <div className="input-container">
        <input
          type="text"
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
          onKeyPress={handleKeyPress}
          placeholder="Type a message..."
          disabled={!isOpen}
        />
        <button onClick={sendMessage} disabled={!isOpen}>
          Send
        </button>
      </div>
    </div>
  );
};

export default Chat;
```

## Socket.IO (React)

Socket.IO provides additional features like automatic reconnection, rooms, namespaces, and fallback support:

```typescript
// services/socket.service.ts
import { io, Socket } from 'socket.io-client';

class SocketService {
  private socket: Socket | null = null;

  connect(url: string, options?: any) {
    this.socket = io(url, {
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      reconnectionAttempts: 5,
      ...options
    });

    this.socket.on('connect', () => {
      console.log('Socket.IO connected:', this.socket?.id);
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Socket.IO disconnected:', reason);
    });

    this.socket.on('connect_error', (error) => {
      console.error('Socket.IO connection error:', error);
    });

    return this.socket;
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }

  emit(event: string, data: any) {
    this.socket?.emit(event, data);
  }

  on(event: string, callback: (data: any) => void) {
    this.socket?.on(event, callback);
  }

  off(event: string, callback?: (data: any) => void) {
    this.socket?.off(event, callback);
  }

  // Join a room
  joinRoom(roomId: string) {
    this.socket?.emit('join-room', roomId);
  }

  // Leave a room
  leaveRoom(roomId: string) {
    this.socket?.emit('leave-room', roomId);
  }

  getSocket() {
    return this.socket;
  }
}

export const socketService = new SocketService();
```

```typescript
// hooks/useSocketIO.ts
import { useEffect, useRef, useState } from 'react';
import { Socket } from 'socket.io-client';
import { socketService } from '../services/socket.service';

export const useSocketIO = (url: string) => {
  const [isConnected, setIsConnected] = useState(false);
  const socketRef = useRef<Socket | null>(null);

  useEffect(() => {
    socketRef.current = socketService.connect(url);

    const socket = socketRef.current;

    socket.on('connect', () => setIsConnected(true));
    socket.on('disconnect', () => setIsConnected(false));

    return () => {
      socketService.disconnect();
    };
  }, [url]);

  const emit = (event: string, data: any) => {
    socketRef.current?.emit(event, data);
  };

  const on = (event: string, callback: (data: any) => void) => {
    socketRef.current?.on(event, callback);
  };

  const off = (event: string, callback?: (data: any) => void) => {
    socketRef.current?.off(event, callback);
  };

  return {
    socket: socketRef.current,
    isConnected,
    emit,
    on,
    off
  };
};
```

```typescript
// components/RealtimeNotifications.tsx
import React, { useEffect, useState } from 'react';
import { useSocketIO } from '../hooks/useSocketIO';

interface Notification {
  id: string;
  type: 'info' | 'warning' | 'error' | 'success';
  message: string;
  timestamp: string;
}

const RealtimeNotifications: React.FC = () => {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const { isConnected, on, off } = useSocketIO('http://localhost:3001');

  useEffect(() => {
    const handleNotification = (notification: Notification) => {
      setNotifications(prev => [notification, ...prev].slice(0, 10));
      
      // Auto-remove after 5 seconds
      setTimeout(() => {
        setNotifications(prev => prev.filter(n => n.id !== notification.id));
      }, 5000);
    };

    on('notification', handleNotification);

    return () => {
      off('notification', handleNotification);
    };
  }, [on, off]);

  return (
    <div className="notifications-container">
      <div className={`status ${isConnected ? 'connected' : 'disconnected'}`}>
        {isConnected ? 'Live' : 'Connecting...'}
      </div>
      
      {notifications.map(notification => (
        <div key={notification.id} className={`notification ${notification.type}`}>
          <p>{notification.message}</p>
          <time>{new Date(notification.timestamp).toLocaleTimeString()}</time>
        </div>
      ))}
    </div>
  );
};

export default RealtimeNotifications;
```

### Socket.IO with Rooms

```typescript
// components/CollaborativeEditor.tsx
import React, { useEffect, useState, useCallback } from 'react';
import { useSocketIO } from '../hooks/useSocketIO';

interface EditorProps {
  documentId: string;
  userId: string;
}

const CollaborativeEditor: React.FC<EditorProps> = ({ documentId, userId }) => {
  const [content, setContent] = useState('');
  const [users, setUsers] = useState<string[]>([]);
  const { socket, isConnected, emit, on, off } = useSocketIO('http://localhost:3001');

  useEffect(() => {
    if (!socket || !isConnected) return;

    // Join document room
    emit('join-document', { documentId, userId });

    // Listen for content changes
    const handleContentChange = (data: { content: string; userId: string }) => {
      if (data.userId !== userId) {
        setContent(data.content);
      }
    };

    // Listen for user presence
    const handleUserJoined = (data: { userId: string }) => {
      setUsers(prev => [...prev, data.userId]);
    };

    const handleUserLeft = (data: { userId: string }) => {
      setUsers(prev => prev.filter(id => id !== data.userId));
    };

    // Listen for initial state
    const handleDocumentState = (data: { content: string; users: string[] }) => {
      setContent(data.content);
      setUsers(data.users);
    };

    on('content-change', handleContentChange);
    on('user-joined', handleUserJoined);
    on('user-left', handleUserLeft);
    on('document-state', handleDocumentState);

    return () => {
      emit('leave-document', { documentId, userId });
      off('content-change', handleContentChange);
      off('user-joined', handleUserJoined);
      off('user-left', handleUserLeft);
      off('document-state', handleDocumentState);
    };
  }, [socket, isConnected, documentId, userId, emit, on, off]);

  const handleContentChange = useCallback((newContent: string) => {
    setContent(newContent);
    emit('content-change', {
      documentId,
      userId,
      content: newContent
    });
  }, [documentId, userId, emit]);

  return (
    <div>
      <div className="editor-header">
        <span>Document: {documentId}</span>
        <span>{users.length} user(s) online</span>
        <span>{isConnected ? '🟢 Connected' : '🔴 Disconnected'}</span>
      </div>

      <div className="active-users">
        {users.map(id => (
          <span key={id} className="user-badge">
            {id}
          </span>
        ))}
      </div>

      <textarea
        value={content}
        onChange={(e) => handleContentChange(e.target.value)}
        placeholder="Start typing..."
        disabled={!isConnected}
      />
    </div>
  );
};

export default CollaborativeEditor;
```

## WebSocket (Angular)

```typescript
// services/websocket.service.ts
import { Injectable } from '@angular/core';
import { Observable, Subject, Observer, timer } from 'rxjs';
import { retryWhen, tap, delayWhen } from 'rxjs/operators';

export interface WebSocketMessage {
  type: string;
  data: any;
}

@Injectable({
  providedIn: 'root'
})
export class WebSocketService {
  private socket$: Subject<WebSocketMessage> | null = null;
  private ws: WebSocket | null = null;

  connect(url: string): Observable<WebSocketMessage> {
    if (!this.socket$ || this.socket$.closed) {
      this.socket$ = this.createWebSocket(url);
    }
    return this.socket$.asObservable();
  }

  private createWebSocket(url: string): Subject<WebSocketMessage> {
    return new Subject<WebSocketMessage>({
      next: (message: WebSocketMessage) => {
        if (this.ws?.readyState === WebSocket.OPEN) {
          this.ws.send(JSON.stringify(message));
        }
      },
      error: (error) => {
        console.error('WebSocket error:', error);
      },
      complete: () => {
        this.ws?.close();
      }
    } as Observer<WebSocketMessage>);
  }

  send(message: WebSocketMessage) {
    if (this.socket$) {
      this.socket$.next(message);
    }
  }

  close() {
    if (this.socket$) {
      this.socket$.complete();
      this.socket$ = null;
    }
  }
}
```

```typescript
// services/chat.service.ts
import { Injectable } from '@angular/core';
import { Observable, Subject } from 'rxjs';
import { WebSocketService, WebSocketMessage } from './websocket.service';

export interface ChatMessage {
  id: string;
  username: string;
  text: string;
  timestamp: string;
}

@Injectable({
  providedIn: 'root'
})
export class ChatService {
  private messagesSubject = new Subject<ChatMessage>();
  public messages$ = this.messagesSubject.asObservable();

  constructor(private wsService: WebSocketService) {}

  connect() {
    this.wsService.connect('ws://localhost:8080/chat').subscribe({
      next: (message: WebSocketMessage) => {
        if (message.type === 'message') {
          this.messagesSubject.next(message.data);
        }
      },
      error: (error) => console.error('Chat error:', error)
    });
  }

  sendMessage(text: string) {
    this.wsService.send({
      type: 'message',
      data: { text }
    });
  }

  disconnect() {
    this.wsService.close();
  }
}
```

```typescript
// components/chat.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { ChatService, ChatMessage } from '../services/chat.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-chat',
  template: `
    <div class="chat-container">
      <div class="messages">
        <div *ngFor="let msg of messages" class="message">
          <strong>{{ msg.username }}:</strong> {{ msg.text }}
          <span class="timestamp">{{ msg.timestamp }}</span>
        </div>
      </div>

      <div class="input-container">
        <input
          [(ngModel)]="inputText"
          (keypress)="onKeyPress($event)"
          placeholder="Type a message..."
        />
        <button (click)="sendMessage()">Send</button>
      </div>
    </div>
  `
})
export class ChatComponent implements OnInit, OnDestroy {
  messages: ChatMessage[] = [];
  inputText = '';
  private subscription: Subscription;

  constructor(private chatService: ChatService) {}

  ngOnInit() {
    this.chatService.connect();
    this.subscription = this.chatService.messages$.subscribe(
      message => {
        this.messages.push(message);
      }
    );
  }

  sendMessage() {
    if (this.inputText.trim()) {
      this.chatService.sendMessage(this.inputText);
      this.inputText = '';
    }
  }

  onKeyPress(event: KeyboardEvent) {
    if (event.key === 'Enter') {
      this.sendMessage();
    }
  }

  ngOnDestroy() {
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
    this.chatService.disconnect();
  }
}
```

## Socket.IO (Angular)

```typescript
// services/socket-io.service.ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { io, Socket } from 'socket.io-client';

@Injectable({
  providedIn: 'root'
})
export class SocketIOService {
  private socket: Socket;

  connect(url: string) {
    this.socket = io(url, {
      transports: ['websocket'],
      reconnection: true
    });

    this.socket.on('connect', () => {
      console.log('Socket.IO connected');
    });

    this.socket.on('disconnect', () => {
      console.log('Socket.IO disconnected');
    });
  }

  emit(event: string, data: any) {
    this.socket.emit(event, data);
  }

  on(event: string): Observable<any> {
    return new Observable(observer => {
      this.socket.on(event, (data: any) => {
        observer.next(data);
      });

      return () => {
        this.socket.off(event);
      };
    });
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
    }
  }
}
```

## Server-Side Implementation (Node.js)

### Native WebSocket Server

```typescript
// server/websocket-server.ts
import WebSocket, { WebSocketServer } from 'ws';
import { IncomingMessage } from 'http';

interface Client extends WebSocket {
  id: string;
  userId?: string;
}

class WSServer {
  private wss: WebSocketServer;
  private clients = new Map<string, Client>();

  constructor(port: number) {
    this.wss = new WebSocketServer({ port });
    this.initialize();
  }

  private initialize() {
    this.wss.on('connection', (ws: Client, req: IncomingMessage) => {
      const clientId = this.generateId();
      ws.id = clientId;
      this.clients.set(clientId, ws);

      console.log(`Client ${clientId} connected`);

      // Send welcome message
      this.sendToClient(ws, {
        type: 'welcome',
        clientId,
        message: 'Connected to WebSocket server'
      });

      // Handle messages
      ws.on('message', (data: Buffer) => {
        this.handleMessage(ws, data);
      });

      // Handle disconnection
      ws.on('close', () => {
        console.log(`Client ${clientId} disconnected`);
        this.clients.delete(clientId);
        this.broadcast({
          type: 'user-left',
          userId: ws.userId
        }, ws);
      });

      // Handle errors
      ws.on('error', (error) => {
        console.error(`WebSocket error for client ${clientId}:`, error);
      });
    });
  }

  private handleMessage(client: Client, data: Buffer) {
    try {
      const message = JSON.parse(data.toString());

      switch (message.type) {
        case 'authenticate':
          client.userId = message.userId;
          this.broadcast({
            type: 'user-joined',
            userId: message.userId
          }, client);
          break;

        case 'message':
          this.broadcast({
            type: 'message',
            userId: client.userId,
            text: message.text,
            timestamp: new Date().toISOString()
          }, client);
          break;

        case 'private-message':
          this.sendToUser(message.targetUserId, {
            type: 'private-message',
            from: client.userId,
            text: message.text
          });
          break;

        default:
          console.warn('Unknown message type:', message.type);
      }
    } catch (error) {
      console.error('Failed to handle message:', error);
    }
  }

  private sendToClient(client: Client, data: any) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(data));
    }
  }

  private sendToUser(userId: string, data: any) {
    for (const client of this.clients.values()) {
      if (client.userId === userId) {
        this.sendToClient(client, data);
        break;
      }
    }
  }

  private broadcast(data: any, excludeClient?: Client) {
    const message = JSON.stringify(data);
    
    this.clients.forEach(client => {
      if (client !== excludeClient && client.readyState === WebSocket.OPEN) {
        client.send(message);
      }
    });
  }

  private generateId(): string {
    return Math.random().toString(36).substring(2, 15);
  }
}

// Start server
const server = new WSServer(8080);
console.log('WebSocket server running on ws://localhost:8080');
```

### Socket.IO Server

```typescript
// server/socket-io-server.ts
import express from 'express';
import { createServer } from 'http';
import { Server, Socket } from 'socket.io';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: 'http://localhost:3000',
    methods: ['GET', 'POST']
  }
});

interface User {
  id: string;
  username: string;
  socketId: string;
}

const users = new Map<string, User>();
const documentRooms = new Map<string, Set<string>>();

io.on('connection', (socket: Socket) => {
  console.log('Client connected:', socket.id);

  // Handle authentication
  socket.on('authenticate', (data: { userId: string; username: string }) => {
    users.set(socket.id, {
      id: data.userId,
      username: data.username,
      socketId: socket.id
    });

    socket.emit('authenticated', {
      userId: data.userId,
      socketId: socket.id
    });
  });

  // Handle joining document room
  socket.on('join-document', (data: { documentId: string; userId: string }) => {
    const { documentId, userId } = data;
    
    socket.join(documentId);
    
    if (!documentRooms.has(documentId)) {
      documentRooms.set(documentId, new Set());
    }
    documentRooms.get(documentId)?.add(userId);

    // Notify others in the room
    socket.to(documentId).emit('user-joined', { userId });

    // Send current document state to the user
    socket.emit('document-state', {
      content: '', // Load from database
      users: Array.from(documentRooms.get(documentId) || [])
    });
  });

  // Handle content changes
  socket.on('content-change', (data: { documentId: string; userId: string; content: string }) => {
    // Broadcast to all other users in the room
    socket.to(data.documentId).emit('content-change', {
      userId: data.userId,
      content: data.content
    });
  });

  // Handle leaving document
  socket.on('leave-document', (data: { documentId: string; userId: string }) => {
    const { documentId, userId } = data;
    
    socket.leave(documentId);
    documentRooms.get(documentId)?.delete(userId);

    socket.to(documentId).emit('user-left', { userId });
  });

  // Handle chat messages
  socket.on('chat-message', (data: { room: string; message: string }) => {
    const user = users.get(socket.id);
    
    io.to(data.room).emit('chat-message', {
      userId: user?.id,
      username: user?.username,
      message: data.message,
      timestamp: new Date().toISOString()
    });
  });

  // Handle disconnection
  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
    
    const user = users.get(socket.id);
    if (user) {
      // Remove user from all rooms
      documentRooms.forEach((userSet, documentId) => {
        if (userSet.has(user.id)) {
          userSet.delete(user.id);
          io.to(documentId).emit('user-left', { userId: user.id });
        }
      });
      
      users.delete(socket.id);
    }
  });
});

const PORT = 3001;
httpServer.listen(PORT, () => {
  console.log(`Socket.IO server running on http://localhost:${PORT}`);
});
```

## Common Mistakes

### Mistake 1: Not Handling Reconnections

```typescript
// BAD: No reconnection logic
const ws = new WebSocket('ws://localhost:8080');

// GOOD: Implement reconnection
const connectWithRetry = (url: string, maxRetries = 5) => {
  let retries = 0;
  
  const connect = () => {
    const ws = new WebSocket(url);
    
    ws.onclose = () => {
      if (retries < maxRetries) {
        retries++;
        setTimeout(connect, 1000 * retries);
      }
    };
    
    return ws;
  };
  
  return connect();
};
```

### Mistake 2: Memory Leaks from Event Listeners

```typescript
// BAD: Not cleaning up listeners
useEffect(() => {
  socket.on('message', handleMessage);
  // Missing cleanup!
}, []);

// GOOD: Clean up listeners
useEffect(() => {
  socket.on('message', handleMessage);
  
  return () => {
    socket.off('message', handleMessage);
  };
}, []);
```

### Mistake 3: Not Validating Messages

```typescript
// BAD: Trusting client data
socket.on('message', (data) => {
  broadcast(data); // Dangerous!
});

// GOOD: Validate and sanitize
socket.on('message', (data) => {
  if (!isValidMessage(data)) {
    return;
  }
  
  const sanitized = sanitizeMessage(data);
  broadcast(sanitized);
});
```

## Best Practices

1. Implement automatic reconnection with exponential backoff
2. Use heartbeat/ping-pong to detect connection health
3. Validate and sanitize all incoming messages
4. Implement proper authentication and authorization
5. Use rooms/namespaces for logical message grouping
6. Handle back-pressure and message queuing
7. Implement rate limiting to prevent abuse
8. Use binary frames for large data transfers
9. Clean up listeners to prevent memory leaks
10. Consider fallback strategies for older browsers

## Key Takeaways

1. WebSockets provide full-duplex, bidirectional communication
2. Socket.IO adds features like automatic reconnection and rooms
3. Always implement reconnection logic with exponential backoff
4. Clean up event listeners to prevent memory leaks
5. Validate and sanitize all messages on the server
6. Use rooms for logical grouping of connections
7. Implement heartbeat mechanism to detect dead connections
8. Consider scalability with Redis adapter for multiple servers

## Interview Questions

1. What is the difference between WebSocket and HTTP?
2. How does the WebSocket handshake work?
3. What are the advantages of Socket.IO over native WebSockets?
4. How do you handle reconnections in WebSocket?
5. What is a heartbeat/ping-pong mechanism?
6. How do you scale WebSocket servers horizontally?
7. What are Socket.IO rooms and namespaces?
8. How do you secure WebSocket connections?
9. What are the performance implications of WebSockets?
10. When would you use WebSockets vs Server-Sent Events?

## Resources

- WebSocket API MDN Documentation
- Socket.IO Official Documentation
- ws (WebSocket library for Node.js)
- WebSocket Protocol RFC 6455
- Socket.IO Rooms and Namespaces
- WebSocket Security Best Practices
