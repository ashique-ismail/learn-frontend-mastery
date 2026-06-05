# System Design: Notification System

## The Idea

**In plain English:** A notification system is software that automatically sends little alerts to users when something important happens — like when someone messages you or your order ships. It figures out who needs to know what, delivers the message instantly, and keeps track of whether you have seen it yet.

**Real-world analogy:** Think of a hospital's paging system. A doctor needs to reach a specific nurse urgently, so they call the front desk, which looks up the nurse's pager number and sends a short alert — the pager buzzes, the nurse sees the message, and the front desk logs that the page was delivered.

- The front desk = the notification server (receives events and routes them to the right recipient)
- The pager number = the device/browser token (unique address that identifies where to deliver the alert)
- The pager buzzing = the push notification arriving on your screen
- The delivery log = the database that tracks which notifications were sent and whether they were read

---

## Overview

Design a notification center with real-time alerts using push notifications, Server-Sent Events, persistence, grouping, priority levels, and dismissal patterns.

## Requirements

### Functional Requirements

- Real-time notification delivery
- Notification center/inbox
- Grouping and categorization
- Priority levels (info, warning, error)
- Mark as read/unread
- Bulk actions (mark all read, delete)
- Notification preferences
- Desktop push notifications
- Sound and vibration alerts
- Rich media support

### Non-Functional Requirements

- Delivery < 1 second
- Support 1M+ daily notifications
- 99.9% delivery rate
- Offline queue support
- Cross-device sync
- Scalable architecture
- Minimal battery impact

## Architecture

```typescript
// Notification service
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notifications$ = new BehaviorSubject<Notification[]>([]);
  private unreadCount$ = new BehaviorSubject<number>(0);
  private eventSource?: EventSource;

  constructor(
    private http: HttpClient,
    private pushService: PushNotificationService,
    private storage: NotificationStorageService
  ) {
    this.initialize();
  }

  private async initialize() {
    // Load cached notifications
    const cached = await this.storage.getNotifications();
    this.notifications$.next(cached);
    this.updateUnreadCount();

    // Setup real-time connection
    this.connectSSE();

    // Request push permission
    await this.pushService.requestPermission();
  }

  private connectSSE() {
    const token = localStorage.getItem('auth_token');
    this.eventSource = new EventSource(`/api/notifications/stream?token=${token}`);

    this.eventSource.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      this.addNotification(notification);
    };

    this.eventSource.onerror = () => {
      // Reconnect logic
      setTimeout(() => this.connectSSE(), 5000);
    };
  }

  private addNotification(notification: Notification) {
    const current = this.notifications$.value;
    this.notifications$.next([notification, ...current]);
    this.updateUnreadCount();
    this.storage.saveNotification(notification);

    // Show push notification if tab not focused
    if (document.hidden) {
      this.pushService.show(notification);
    }

    // Play sound
    this.playSound(notification.priority);
  }

  getNotifications(): Observable<Notification[]> {
    return this.notifications$.asObservable();
  }

  getUnreadCount(): Observable<number> {
    return this.unreadCount$.asObservable();
  }

  markAsRead(id: string): Observable<void> {
    return this.http.put<void>(`/api/notifications/${id}/read`, {}).pipe(
      tap(() => {
        const notifications = this.notifications$.value;
        const updated = notifications.map(n =>
          n.id === id ? { ...n, read: true } : n
        );
        this.notifications$.next(updated);
        this.updateUnreadCount();
        this.storage.updateNotification(id, { read: true });
      })
    );
  }

  markAllAsRead(): Observable<void> {
    return this.http.put<void>('/api/notifications/read-all', {}).pipe(
      tap(() => {
        const updated = this.notifications$.value.map(n => ({ ...n, read: true }));
        this.notifications$.next(updated);
        this.updateUnreadCount();
      })
    );
  }

  deleteNotification(id: string): Observable<void> {
    return this.http.delete<void>(`/api/notifications/${id}`).pipe(
      tap(() => {
        const filtered = this.notifications$.value.filter(n => n.id !== id);
        this.notifications$.next(filtered);
        this.updateUnreadCount();
        this.storage.deleteNotification(id);
      })
    );
  }

  private updateUnreadCount() {
    const unread = this.notifications$.value.filter(n => !n.read).length;
    this.unreadCount$.next(unread);
  }

  private playSound(priority: NotificationPriority) {
    if (priority === 'high' || priority === 'critical') {
      const audio = new Audio('/assets/sounds/notification.mp3');
      audio.play();
    }
  }

  disconnect() {
    this.eventSource?.close();
  }
}

// Notification center component
@Component({
  selector: 'app-notification-center',
  standalone: true,
  template: `
    <div class="notification-center">
      <div class="header">
        <h3>Notifications</h3>
        <span class="badge">{{ unreadCount$ | async }}</span>
        <div class="actions">
          <button (click)="markAllAsRead()">Mark all read</button>
          <button (click)="clearAll()">Clear all</button>
        </div>
      </div>

      <div class="filters">
        <button
          *ngFor="let filter of filters"
          (click)="selectFilter(filter)"
          [class.active]="currentFilter === filter">
          {{ filter }}
        </button>
      </div>

      <div class="notifications-list">
        <div
          *ngFor="let group of groupedNotifications$ | async"
          class="notification-group">
          
          <div class="group-header">
            <h4>{{ group.title }}</h4>
            <span class="count">{{ group.notifications.length }}</span>
          </div>

          <div
            *ngFor="let notification of group.notifications"
            class="notification-item"
            [class.unread]="!notification.read"
            [class]="notification.priority"
            (click)="handleNotification(notification)">
            
            <div class="icon" [ngSwitch]="notification.type">
              <span *ngSwitchCase="'message'">💬</span>
              <span *ngSwitchCase="'alert'">⚠️</span>
              <span *ngSwitchCase="'success'">✅</span>
              <span *ngSwitchCase="'info'">ℹ️</span>
              <span *ngSwitchDefault>🔔</span>
            </div>

            <div class="content">
              <div class="title">{{ notification.title }}</div>
              <div class="message">{{ notification.message }}</div>
              <div class="meta">
                <span class="time">{{ notification.timestamp | timeAgo }}</span>
                <span *ngIf="notification.category" class="category">
                  {{ notification.category }}
                </span>
              </div>
            </div>

            <div class="actions">
              <button
                *ngIf="!notification.read"
                (click)="markAsRead(notification.id); $event.stopPropagation()"
                aria-label="Mark as read">
                ✓
              </button>
              <button
                (click)="deleteNotification(notification.id); $event.stopPropagation()"
                aria-label="Delete">
                ×
              </button>
            </div>
          </div>
        </div>

        <div *ngIf="(notifications$ | async)?.length === 0" class="empty-state">
          <span class="icon">🔔</span>
          <p>No notifications</p>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .notification-center {
      width: 400px;
      max-height: 600px;
      background: white;
      border-radius: 8px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.15);
      display: flex;
      flex-direction: column;
    }

    .notification-item {
      display: flex;
      gap: 12px;
      padding: 16px;
      border-bottom: 1px solid #e0e0e0;
      cursor: pointer;
      transition: background 0.2s;
    }

    .notification-item:hover {
      background: #f5f5f5;
    }

    .notification-item.unread {
      background: #e3f2fd;
    }

    .notification-item.high {
      border-left: 4px solid #ff9800;
    }

    .notification-item.critical {
      border-left: 4px solid #f44336;
    }

    .badge {
      background: #f44336;
      color: white;
      padding: 2px 8px;
      border-radius: 12px;
      font-size: 12px;
      font-weight: bold;
    }
  `]
})
export class NotificationCenterComponent implements OnInit {
  notifications$ = this.notificationService.getNotifications();
  unreadCount$ = this.notificationService.getUnreadCount();
  groupedNotifications$!: Observable<NotificationGroup[]>;
  
  filters = ['All', 'Unread', 'Messages', 'Alerts'];
  currentFilter = 'All';

  constructor(
    private notificationService: NotificationService,
    private router: Router
  ) {}

  ngOnInit() {
    this.groupedNotifications$ = this.notifications$.pipe(
      map(notifications => this.groupNotifications(notifications))
    );
  }

  private groupNotifications(notifications: Notification[]): NotificationGroup[] {
    const today: Notification[] = [];
    const yesterday: Notification[] = [];
    const older: Notification[] = [];

    const now = new Date();
    const todayStart = new Date(now.getFullYear(), now.getMonth(), now.getDate());
    const yesterdayStart = new Date(todayStart.getTime() - 24 * 60 * 60 * 1000);

    notifications.forEach(notification => {
      const notificationDate = new Date(notification.timestamp);
      
      if (notificationDate >= todayStart) {
        today.push(notification);
      } else if (notificationDate >= yesterdayStart) {
        yesterday.push(notification);
      } else {
        older.push(notification);
      }
    });

    const groups: NotificationGroup[] = [];
    if (today.length > 0) {
      groups.push({ title: 'Today', notifications: today });
    }
    if (yesterday.length > 0) {
      groups.push({ title: 'Yesterday', notifications: yesterday });
    }
    if (older.length > 0) {
      groups.push({ title: 'Older', notifications: older });
    }

    return groups;
  }

  handleNotification(notification: Notification) {
    if (!notification.read) {
      this.notificationService.markAsRead(notification.id).subscribe();
    }

    if (notification.actionUrl) {
      this.router.navigate([notification.actionUrl]);
    }
  }

  markAsRead(id: string) {
    this.notificationService.markAsRead(id).subscribe();
  }

  markAllAsRead() {
    this.notificationService.markAllAsRead().subscribe();
  }

  deleteNotification(id: string) {
    this.notificationService.deleteNotification(id).subscribe();
  }

  clearAll() {
    // Clear all notifications
  }

  selectFilter(filter: string) {
    this.currentFilter = filter;
    // Filter logic
  }
}

// Push notification service
@Injectable({ providedIn: 'root' })
export class PushNotificationService {
  async requestPermission(): Promise<boolean> {
    if (!('Notification' in window)) {
      return false;
    }

    if (Notification.permission === 'granted') {
      return true;
    }

    if (Notification.permission !== 'denied') {
      const permission = await Notification.requestPermission();
      return permission === 'granted';
    }

    return false;
  }

  async show(notification: Notification) {
    if (Notification.permission !== 'granted') {
      return;
    }

    const options: NotificationOptions = {
      body: notification.message,
      icon: notification.icon || '/assets/icons/icon-192x192.png',
      badge: '/assets/icons/badge.png',
      tag: notification.id,
      renotify: true,
      vibrate: [200, 100, 200],
      data: {
        url: notification.actionUrl
      }
    };

    const browserNotification = new Notification(notification.title, options);

    browserNotification.onclick = (event) => {
      event.preventDefault();
      window.focus();
      if (notification.actionUrl) {
        window.location.href = notification.actionUrl;
      }
      browserNotification.close();
    };
  }
}

// IndexedDB storage
@Injectable({ providedIn: 'root' })
export class NotificationStorageService {
  private db!: IDBDatabase;

  constructor() {
    this.initDB();
  }

  private async initDB() {
    const request = indexedDB.open('NotificationsDB', 1);

    request.onupgradeneeded = (event: any) => {
      const db = event.target.result;
      
      if (!db.objectStoreNames.contains('notifications')) {
        const store = db.createObjectStore('notifications', { keyPath: 'id' });
        store.createIndex('timestamp', 'timestamp', { unique: false });
        store.createIndex('read', 'read', { unique: false });
      }
    };

    request.onsuccess = (event: any) => {
      this.db = event.target.result;
    };
  }

  async getNotifications(): Promise<Notification[]> {
    const transaction = this.db.transaction(['notifications'], 'readonly');
    const store = transaction.objectStore('notifications');
    const request = store.getAll();

    return new Promise((resolve) => {
      request.onsuccess = () => resolve(request.result);
    });
  }

  async saveNotification(notification: Notification): Promise<void> {
    const transaction = this.db.transaction(['notifications'], 'readwrite');
    const store = transaction.objectStore('notifications');
    store.put(notification);

    return new Promise((resolve, reject) => {
      transaction.oncomplete = () => resolve();
      transaction.onerror = () => reject(transaction.error);
    });
  }

  async updateNotification(id: string, updates: Partial<Notification>): Promise<void> {
    const transaction = this.db.transaction(['notifications'], 'readwrite');
    const store = transaction.objectStore('notifications');
    
    const getRequest = store.get(id);
    
    return new Promise((resolve, reject) => {
      getRequest.onsuccess = () => {
        const notification = getRequest.result;
        if (notification) {
          const updated = { ...notification, ...updates };
          store.put(updated);
        }
      };
      
      transaction.oncomplete = () => resolve();
      transaction.onerror = () => reject(transaction.error);
    });
  }

  async deleteNotification(id: string): Promise<void> {
    const transaction = this.db.transaction(['notifications'], 'readwrite');
    const store = transaction.objectStore('notifications');
    store.delete(id);

    return new Promise((resolve, reject) => {
      transaction.oncomplete = () => resolve();
      transaction.onerror = () => reject(transaction.error);
    });
  }
}
```

## Key Takeaways

1. **Server-Sent Events**: Use SSE for real-time notification delivery with automatic reconnection.

2. **Push Notifications**: Implement browser push notifications for alerts when tab not focused.

3. **Offline Storage**: Store notifications in IndexedDB for offline access and persistence.

4. **Grouping Strategy**: Group by time (today, yesterday, older) and category for better organization.

5. **Priority Levels**: Visual indicators (colors, icons) and different behaviors (sound, vibration) based on priority.

6. **Bulk Actions**: Support mark all read, delete all for efficient notification management.

7. **Read/Unread State**: Track and sync read state across devices using server-side persistence.

8. **Rich Content**: Support images, actions, categories in notification payload.

9. **Performance**: Use virtual scrolling for large notification lists, lazy load older notifications.

10. **Battery Optimization**: Batch updates, limit sound/vibration, efficient polling strategies.

## Red Flags to Avoid

- Not handling SSE reconnection
- Missing offline support
- No push notification permission handling
- Poor grouping/categorization
- Not cleaning up old notifications
- Missing bulk actions
- Poor mobile experience
- Not handling notification actions
- Memory leaks from event listeners
- Missing error handling

## Interview Talking Points

- SSE vs WebSocket for notifications
- Push notification best practices
- Grouping and prioritization strategies
- Cross-device synchronization
- Offline storage with IndexedDB
- Battery and performance optimization
- Notification preferences/settings
- Rate limiting and batching
- Analytics and tracking
- Security considerations
