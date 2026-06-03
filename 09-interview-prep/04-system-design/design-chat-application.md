# System Design: Chat Application

## Overview

Design a real-time chat application with WebSocket communication, message persistence, presence indicators, typing indicators, read receipts, and offline support.

## Requirements

### Functional Requirements
- Real-time messaging
- User presence (online/offline/away)
- Typing indicators
- Read receipts
- Message history
- File/image sharing
- Emoji reactions
- Push notifications
- Search messages
- Multiple chat rooms

### Non-Functional Requirements
- Message delivery < 100ms
- Support 10K concurrent users per room
- 99.9% uptime
- End-to-end encryption
- Offline message queue
- Cross-device sync
- Scalable to millions of users

## Architecture

```typescript
// WebSocket service for real-time communication
@Injectable({ providedIn: 'root' })
export class ChatWebSocketService {
  private socket$!: WebSocketSubject<any>;
  private reconnectAttempts = 0;
  private readonly maxReconnectAttempts = 5;

  messages$ = new Subject<ChatMessage>();
  presence$ = new Subject<PresenceUpdate>();
  typingIndicators$ = new Subject<TypingIndicator>();
  connectionStatus$ = new BehaviorSubject<ConnectionStatus>('disconnected');

  constructor(private authService: AuthService) {
    this.connect();
  }

  private connect() {
    const token = this.authService.getToken();
    this.socket$ = new WebSocketSubject({
      url: `wss://api.example.com/chat?token=${token}`,
      openObserver: {
        next: () => {
          this.connectionStatus$.next('connected');
          this.reconnectAttempts = 0;
          this.syncOfflineMessages();
        }
      },
      closeObserver: {
        next: () => {
          this.connectionStatus$.next('disconnected');
          this.handleReconnect();
        }
      }
    });

    this.socket$.subscribe({
      next: (message) => this.handleMessage(message),
      error: (error) => this.handleError(error)
    });
  }

  private handleMessage(message: any) {
    switch (message.type) {
      case 'message':
        this.messages$.next(message.data);
        break;
      case 'presence':
        this.presence$.next(message.data);
        break;
      case 'typing':
        this.typingIndicators$.next(message.data);
        break;
    }
  }

  sendMessage(roomId: string, content: string) {
    const message: ChatMessage = {
      id: this.generateId(),
      roomId,
      content,
      senderId: this.authService.getCurrentUserId(),
      timestamp: Date.now(),
      status: 'sending'
    };

    this.messages$.next(message);

    if (this.connectionStatus$.value === 'connected') {
      this.socket$.next({ type: 'message', data: message });
    } else {
      this.queueOfflineMessage(message);
    }

    return message;
  }

  sendTypingIndicator(roomId: string, isTyping: boolean) {
    this.socket$.next({
      type: 'typing',
      data: {
        roomId,
        userId: this.authService.getCurrentUserId(),
        isTyping
      }
    });
  }

  private handleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      this.connectionStatus$.next('reconnecting');
      
      const delay = 1000 * Math.pow(2, this.reconnectAttempts);
      setTimeout(() => this.connect(), delay);
    }
  }

  private queueOfflineMessage(message: ChatMessage) {
    const queue = JSON.parse(localStorage.getItem('offlineMessages') || '[]');
    queue.push(message);
    localStorage.setItem('offlineMessages', JSON.stringify(queue));
  }

  private syncOfflineMessages() {
    const queue = JSON.parse(localStorage.getItem('offlineMessages') || '[]');
    queue.forEach((message: ChatMessage) => {
      this.socket$.next({ type: 'message', data: message });
    });
    localStorage.removeItem('offlineMessages');
  }

  private generateId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Chat component with full features
@Component({
  selector: 'app-chat',
  standalone: true,
  template: `
    <div class="chat-container">
      <div class="chat-header">
        <h2>{{ room.name }}</h2>
        <span class="online-count">{{ onlineUsers.length }} online</span>
        <span class="connection-status" [class]="connectionStatus">
          {{ connectionStatus }}
        </span>
      </div>

      <div class="messages-container" #messagesContainer>
        <div
          *ngFor="let message of messages; trackBy: trackByMessageId"
          class="message"
          [class.own]="message.senderId === currentUserId">
          
          <img [src]="getUserAvatar(message.senderId)" class="avatar">
          
          <div class="message-content">
            <div class="message-header">
              <span class="sender">{{ getUserName(message.senderId) }}</span>
              <span class="timestamp">{{ message.timestamp | date:'short' }}</span>
            </div>
            
            <div class="message-text">{{ message.content }}</div>

            <div class="message-status">
              <span *ngIf="message.status === 'sending'">Sending...</span>
              <span *ngIf="message.status === 'sent'">✓</span>
              <span *ngIf="message.status === 'read'">✓✓</span>
            </div>
          </div>
        </div>

        <div *ngIf="typingUsers.length > 0" class="typing-indicator">
          <span>{{ getTypingText() }}</span>
          <span class="dots">
            <span>.</span><span>.</span><span>.</span>
          </span>
        </div>
      </div>

      <div class="chat-input">
        <textarea
          [(ngModel)]="messageText"
          (input)="onTyping()"
          (keydown.enter)="onEnter($event)"
          placeholder="Type a message..."
          class="message-input"></textarea>
        
        <button
          (click)="sendMessage()"
          [disabled]="!messageText.trim()"
          class="send-button">
          Send
        </button>
      </div>
    </div>
  `,
  styles: [`
    .chat-container {
      display: flex;
      flex-direction: column;
      height: 100vh;
      max-width: 800px;
      margin: 0 auto;
    }

    .messages-container {
      flex: 1;
      overflow-y: auto;
      padding: 20px;
      background: #f5f5f5;
    }

    .message {
      display: flex;
      gap: 12px;
      margin-bottom: 16px;
      animation: slideIn 0.3s ease-out;
    }

    @keyframes slideIn {
      from {
        opacity: 0;
        transform: translateY(10px);
      }
      to {
        opacity: 1;
        transform: translateY(0);
      }
    }

    .typing-indicator .dots span {
      animation: blink 1.4s infinite;
    }

    @keyframes blink {
      0%, 60%, 100% { opacity: 0; }
      30% { opacity: 1; }
    }
  `]
})
export class ChatComponent implements OnInit, OnDestroy {
  @ViewChild('messagesContainer') messagesContainer!: ElementRef;

  room!: ChatRoom;
  messages: ChatMessage[] = [];
  messageText = '';
  currentUserId = '';
  onlineUsers: string[] = [];
  typingUsers: string[] = [];
  connectionStatus: ConnectionStatus = 'disconnected';
  
  private typingTimeout?: number;
  private destroy$ = new Subject<void>();

  constructor(
    private chatWs: ChatWebSocketService,
    private chatService: ChatService,
    private route: ActivatedRoute
  ) {}

  ngOnInit() {
    const roomId = this.route.snapshot.params['id'];
    this.loadMessages(roomId);
    this.subscribeToUpdates(roomId);
  }

  private subscribeToUpdates(roomId: string) {
    this.chatWs.messages$
      .pipe(
        filter(msg => msg.roomId === roomId),
        takeUntil(this.destroy$)
      )
      .subscribe(message => {
        this.messages.push(message);
        this.scrollToBottom();
      });

    this.chatWs.typingIndicators$
      .pipe(
        filter(indicator => indicator.roomId === roomId),
        takeUntil(this.destroy$)
      )
      .subscribe(indicator => {
        if (indicator.isTyping && indicator.userId !== this.currentUserId) {
          if (!this.typingUsers.includes(indicator.userId)) {
            this.typingUsers.push(indicator.userId);
          }
        } else {
          this.typingUsers = this.typingUsers.filter(id => id !== indicator.userId);
        }
      });
  }

  sendMessage() {
    if (!this.messageText.trim()) return;

    this.chatWs.sendMessage(this.room.id, this.messageText);
    this.messageText = '';
    this.chatWs.sendTypingIndicator(this.room.id, false);
  }

  onTyping() {
    this.chatWs.sendTypingIndicator(this.room.id, true);

    clearTimeout(this.typingTimeout);
    this.typingTimeout = window.setTimeout(() => {
      this.chatWs.sendTypingIndicator(this.room.id, false);
    }, 1000);
  }

  onEnter(event: KeyboardEvent) {
    if (!event.shiftKey) {
      event.preventDefault();
      this.sendMessage();
    }
  }

  getTypingText(): string {
    if (this.typingUsers.length === 1) {
      return `${this.getUserName(this.typingUsers[0])} is typing`;
    }
    return `${this.typingUsers.length} people are typing`;
  }

  getUserName(userId: string): string {
    return 'User';
  }

  getUserAvatar(userId: string): string {
    return '/assets/avatars/default.png';
  }

  private scrollToBottom() {
    setTimeout(() => {
      const container = this.messagesContainer?.nativeElement;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    });
  }

  trackByMessageId(index: number, message: ChatMessage): string {
    return message.id;
  }

  private loadMessages(roomId: string) {
    this.chatService.getMessages(roomId).subscribe(messages => {
      this.messages = messages;
      this.scrollToBottom();
    });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## Key Takeaways

1. **WebSocket for Real-time**: Use WebSocket for bidirectional communication with automatic reconnection.

2. **Optimistic Updates**: Apply message send optimistically before server confirmation.

3. **Offline Support**: Queue messages locally when offline, sync when connection restored.

4. **Typing Indicators**: Debounce typing events (1s) to reduce network traffic.

5. **Read Receipts**: Track message read status with timestamps and visual indicators.

6. **Message Persistence**: Store messages in IndexedDB for offline access.

7. **Connection Management**: Handle reconnection with exponential backoff strategy.

8. **Presence System**: Track online/offline/away status using heartbeat mechanism.

9. **Performance**: Use virtual scrolling for long message histories.

10. **Security**: Implement end-to-end encryption and sanitize message content.

## Red Flags to Avoid

- Not implementing reconnection logic
- Sending typing indicators on every keystroke
- No offline message queueing
- Not handling WebSocket errors
- Missing message delivery confirmation
- No rate limiting
- Not sanitizing user input
- Memory leaks from WebSocket connections
- Not managing scroll position
- Missing accessibility

## Interview Talking Points

- WebSocket architecture and reconnection
- Message delivery guarantees
- Offline message queue implementation
- Typing indicator debouncing
- Presence system design
- Read receipt tracking
- IndexedDB for persistence
- Scalability challenges
- Push notification integration
- Security considerations
