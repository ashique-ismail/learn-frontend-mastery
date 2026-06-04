# Advanced Service Workers

## Introduction

Service Workers are the backbone of Progressive Web Apps, acting as programmable network proxies that sit between your web application and the network. While basic service worker functionality covers caching and offline support, advanced patterns enable powerful features like background sync, push notifications, periodic updates, and sophisticated caching strategies.

This guide explores advanced service worker techniques that transform web applications into truly app-like experiences, covering everything from complex caching strategies to background processing and performance optimization.

## Service Worker Lifecycle Advanced

### Update Strategies

```javascript
// sw.js - Advanced update handling
const VERSION = 'v2.0.0';
const CACHE_NAME = `app-${VERSION}`;

self.addEventListener('install', (event) => {
    console.log('[SW] Installing version:', VERSION);
    
    // Skip waiting to activate immediately
    self.skipWaiting();
    
    event.waitUntil(
        caches.open(CACHE_NAME).then((cache) => {
            return cache.addAll([
                '/',
                '/index.html',
                '/styles.css',
                '/app.js',
                '/manifest.json'
            ]);
        })
    );
});

self.addEventListener('activate', (event) => {
    console.log('[SW] Activating version:', VERSION);
    
    event.waitUntil(
        Promise.all([
            // Take control immediately
            self.clients.claim(),
            
            // Clean up old caches
            caches.keys().then((cacheNames) => {
                return Promise.all(
                    cacheNames
                        .filter((name) => name !== CACHE_NAME)
                        .map((name) => {
                            console.log('[SW] Deleting old cache:', name);
                            return caches.delete(name);
                        })
                );
            }),
            
            // Notify clients of update
            self.clients.matchAll().then((clients) => {
                clients.forEach((client) => {
                    client.postMessage({
                        type: 'SW_UPDATED',
                        version: VERSION
                    });
                });
            })
        ])
    );
});
```

### Update Notification

```javascript
// app.js - Handle service worker updates
if ('serviceWorker' in navigator) {
    let refreshing = false;
    
    navigator.serviceWorker.addEventListener('controllerchange', () => {
        if (refreshing) return;
        refreshing = true;
        window.location.reload();
    });
    
    navigator.serviceWorker.register('/sw.js').then((registration) => {
        // Check for updates periodically
        setInterval(() => {
            registration.update();
        }, 60 * 60 * 1000); // Every hour
        
        registration.addEventListener('updatefound', () => {
            const newWorker = registration.installing;
            
            newWorker.addEventListener('statechange', () => {
                if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                    // New version available
                    showUpdateNotification();
                }
            });
        });
    });
    
    // Listen for messages from service worker
    navigator.serviceWorker.addEventListener('message', (event) => {
        if (event.data.type === 'SW_UPDATED') {
            console.log('New version:', event.data.version);
            showUpdateBanner(event.data.version);
        }
    });
}

function showUpdateNotification() {
    const notification = document.createElement('div');
    notification.className = 'update-banner';
    notification.innerHTML = `
        <p>New version available!</p>
        <button onclick="updateApp()">Update Now</button>
        <button onclick="dismissUpdate()">Later</button>
    `;
    document.body.appendChild(notification);
}

function updateApp() {
    navigator.serviceWorker.getRegistration().then((registration) => {
        registration.waiting?.postMessage({ type: 'SKIP_WAITING' });
    });
}
```

## Background Sync

### Basic Background Sync

```javascript
// sw.js - Background sync
self.addEventListener('sync', (event) => {
    if (event.tag === 'sync-messages') {
        event.waitUntil(syncMessages());
    } else if (event.tag === 'sync-posts') {
        event.waitUntil(syncPosts());
    }
});

async function syncMessages() {
    try {
        // Get pending messages from IndexedDB
        const db = await openDB();
        const messages = await db.getAll('pending-messages');
        
        // Send each message
        const results = await Promise.allSettled(
            messages.map((message) => 
                fetch('/api/messages', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(message)
                })
            )
        );
        
        // Remove successfully synced messages
        await Promise.all(
            results.map((result, index) => {
                if (result.status === 'fulfilled' && result.value.ok) {
                    return db.delete('pending-messages', messages[index].id);
                }
            })
        );
        
        // Notify clients
        const clients = await self.clients.matchAll();
        clients.forEach((client) => {
            client.postMessage({
                type: 'SYNC_COMPLETE',
                tag: 'sync-messages'
            });
        });
        
        return true;
    } catch (error) {
        console.error('Sync failed:', error);
        throw error; // Retry sync
    }
}
```

```javascript
// app.js - Register background sync
async function sendMessage(message) {
    // Save to IndexedDB
    const db = await openDB();
    await db.add('pending-messages', {
        id: Date.now(),
        ...message,
        timestamp: new Date().toISOString()
    });
    
    // Try to send immediately
    if (navigator.onLine) {
        try {
            const response = await fetch('/api/messages', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(message)
            });
            
            if (response.ok) {
                await db.delete('pending-messages', message.id);
                return { success: true };
            }
        } catch (error) {
            console.log('Send failed, will sync later');
        }
    }
    
    // Register for background sync
    if ('sync' in registration) {
        try {
            await registration.sync.register('sync-messages');
            return { success: true, syncing: true };
        } catch (error) {
            console.error('Sync registration failed:', error);
        }
    }
    
    return { success: false };
}
```

### Periodic Background Sync

```javascript
// sw.js - Periodic sync
self.addEventListener('periodicsync', (event) => {
    if (event.tag === 'update-content') {
        event.waitUntil(updateContent());
    }
});

async function updateContent() {
    try {
        const response = await fetch('/api/updates');
        const data = await response.json();
        
        // Cache new content
        const cache = await caches.open('dynamic-content');
        await cache.put('/api/updates', new Response(JSON.stringify(data)));
        
        // Check for important updates
        if (data.important) {
            // Show notification
            await self.registration.showNotification('New Content Available', {
                body: data.message,
                icon: '/icon-192.png',
                badge: '/badge-72.png',
                tag: 'content-update'
            });
        }
        
        return true;
    } catch (error) {
        console.error('Periodic sync failed:', error);
    }
}
```

```javascript
// app.js - Register periodic sync
async function registerPeriodicSync() {
    if ('periodicSync' in registration) {
        try {
            await registration.periodicSync.register('update-content', {
                minInterval: 24 * 60 * 60 * 1000 // 24 hours
            });
            console.log('Periodic sync registered');
        } catch (error) {
            console.error('Periodic sync registration failed:', error);
        }
    }
}

// Check periodic sync status
async function checkPeriodicSync() {
    if ('periodicSync' in registration) {
        const tags = await registration.periodicSync.getTags();
        console.log('Registered periodic syncs:', tags);
    }
}

// Unregister periodic sync
async function unregisterPeriodicSync() {
    if ('periodicSync' in registration) {
        await registration.periodicSync.unregister('update-content');
    }
}
```

## Push Notifications

### Push Subscription

```javascript
// app.js - Subscribe to push notifications
async function subscribeToPush() {
    try {
        const registration = await navigator.serviceWorker.ready;
        
        // Request notification permission
        const permission = await Notification.requestPermission();
        
        if (permission !== 'granted') {
            console.log('Notification permission denied');
            return null;
        }
        
        // Get VAPID public key from server
        const response = await fetch('/api/vapid-public-key');
        const { publicKey } = await response.json();
        
        // Subscribe
        const subscription = await registration.pushManager.subscribe({
            userVisibleOnly: true,
            applicationServerKey: urlBase64ToUint8Array(publicKey)
        });
        
        // Send subscription to server
        await fetch('/api/push-subscription', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(subscription)
        });
        
        console.log('Push subscription successful');
        return subscription;
    } catch (error) {
        console.error('Push subscription failed:', error);
        return null;
    }
}

function urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - base64String.length % 4) % 4);
    const base64 = (base64String + padding)
        .replace(/\-/g, '+')
        .replace(/_/g, '/');
    
    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);
    
    for (let i = 0; i < rawData.length; ++i) {
        outputArray[i] = rawData.charCodeAt(i);
    }
    
    return outputArray;
}
```

### Push Event Handling

```javascript
// sw.js - Handle push events
self.addEventListener('push', (event) => {
    console.log('[SW] Push received');
    
    const data = event.data ? event.data.json() : {};
    
    const options = {
        body: data.body || 'New notification',
        icon: data.icon || '/icon-192.png',
        badge: '/badge-72.png',
        vibrate: [200, 100, 200],
        tag: data.tag || 'default',
        requireInteraction: data.requireInteraction || false,
        data: {
            url: data.url || '/',
            timestamp: Date.now()
        },
        actions: data.actions || [
            { action: 'open', title: 'Open' },
            { action: 'close', title: 'Close' }
        ]
    };
    
    event.waitUntil(
        self.registration.showNotification(data.title || 'Notification', options)
    );
});

self.addEventListener('notificationclick', (event) => {
    console.log('[SW] Notification clicked:', event.action);
    
    event.notification.close();
    
    const urlToOpen = event.notification.data.url;
    
    event.waitUntil(
        clients.matchAll({ type: 'window', includeUncontrolled: true })
            .then((clientList) => {
                // Check if already open
                for (const client of clientList) {
                    if (client.url === urlToOpen && 'focus' in client) {
                        return client.focus();
                    }
                }
                
                // Open new window
                if (clients.openWindow) {
                    return clients.openWindow(urlToOpen);
                }
            })
    );
});

self.addEventListener('notificationclose', (event) => {
    console.log('[SW] Notification closed');
    
    // Track notification dismissals
    event.waitUntil(
        fetch('/api/analytics/notification-closed', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                tag: event.notification.tag,
                timestamp: Date.now()
            })
        })
    );
});
```

### Advanced Notification Patterns

```javascript
// sw.js - Advanced notification handling
class NotificationManager {
    static async show(data) {
        const { title, options } = this.buildNotification(data);
        
        // Check existing notifications
        const existing = await self.registration.getNotifications({
            tag: options.tag
        });
        
        // Update or stack notifications
        if (existing.length > 0 && data.stack) {
            await this.stackNotifications(existing, data);
        } else {
            await self.registration.showNotification(title, options);
        }
    }
    
    static buildNotification(data) {
        return {
            title: data.title,
            options: {
                body: data.body,
                icon: data.icon || '/icon-192.png',
                badge: '/badge-72.png',
                tag: data.tag || `notification-${Date.now()}`,
                data: data.data || {},
                actions: data.actions || [],
                silent: data.silent || false,
                renotify: data.renotify || false,
                requireInteraction: data.requireInteraction || false
            }
        };
    }
    
    static async stackNotifications(existing, newData) {
        const count = existing.length + 1;
        
        // Close existing
        existing.forEach((notification) => notification.close());
        
        // Show stacked notification
        await self.registration.showNotification(
            `You have ${count} notifications`,
            {
                body: `${newData.title}\n${newData.body}`,
                icon: '/icon-192.png',
                badge: '/badge-72.png',
                tag: newData.tag,
                data: {
                    count,
                    notifications: [
                        ...existing.map((n) => n.data),
                        newData.data
                    ]
                }
            }
        );
    }
}

self.addEventListener('push', (event) => {
    const data = event.data.json();
    event.waitUntil(NotificationManager.show(data));
});
```

## Advanced Caching Strategies

### Network Priority with Timeout

```javascript
// sw.js - Network first with timeout
async function networkFirstWithTimeout(request, timeout = 3000) {
    const cache = await caches.open('dynamic');
    
    try {
        // Race between network and timeout
        const networkPromise = fetch(request);
        const timeoutPromise = new Promise((_, reject) => 
            setTimeout(() => reject(new Error('Timeout')), timeout)
        );
        
        const response = await Promise.race([networkPromise, timeoutPromise]);
        
        // Cache successful response
        if (response.ok) {
            cache.put(request, response.clone());
        }
        
        return response;
    } catch (error) {
        console.log('Network failed, trying cache:', error);
        
        const cached = await cache.match(request);
        
        if (cached) {
            return cached;
        }
        
        // Return offline page
        return caches.match('/offline.html');
    }
}
```

### Stale While Revalidate

```javascript
// sw.js - Stale while revalidate
async function staleWhileRevalidate(request) {
    const cache = await caches.open('dynamic');
    
    // Return cached response immediately
    const cached = await cache.match(request);
    
    // Fetch fresh data in background
    const fetchPromise = fetch(request)
        .then((response) => {
            if (response.ok) {
                cache.put(request, response.clone());
            }
            return response;
        })
        .catch(() => null);
    
    // Return cached or wait for network
    return cached || fetchPromise;
}
```

### Cache Strategy Router

```javascript
// sw.js - Route-based caching
class CacheRouter {
    constructor() {
        this.routes = new Map();
    }
    
    register(pattern, strategy, options = {}) {
        this.routes.set(pattern, { strategy, options });
    }
    
    async handle(request) {
        const url = new URL(request.url);
        
        // Find matching route
        for (const [pattern, config] of this.routes) {
            if (this.matches(pattern, url)) {
                return this.executeStrategy(request, config);
            }
        }
        
        // Default: network only
        return fetch(request);
    }
    
    matches(pattern, url) {
        if (pattern instanceof RegExp) {
            return pattern.test(url.pathname);
        }
        return url.pathname.startsWith(pattern);
    }
    
    async executeStrategy(request, config) {
        const { strategy, options } = config;
        
        switch (strategy) {
            case 'cache-first':
                return this.cacheFirst(request, options);
            case 'network-first':
                return this.networkFirst(request, options);
            case 'stale-while-revalidate':
                return this.staleWhileRevalidate(request, options);
            case 'network-only':
                return fetch(request);
            case 'cache-only':
                return caches.match(request);
            default:
                return fetch(request);
        }
    }
    
    async cacheFirst(request, options) {
        const cache = await caches.open(options.cacheName || 'dynamic');
        const cached = await cache.match(request);
        
        if (cached) {
            return cached;
        }
        
        const response = await fetch(request);
        if (response.ok) {
            cache.put(request, response.clone());
        }
        
        return response;
    }
    
    async networkFirst(request, options) {
        const cache = await caches.open(options.cacheName || 'dynamic');
        
        try {
            const response = await fetch(request);
            if (response.ok) {
                cache.put(request, response.clone());
            }
            return response;
        } catch (error) {
            const cached = await cache.match(request);
            if (cached) {
                return cached;
            }
            throw error;
        }
    }
    
    async staleWhileRevalidate(request, options) {
        const cache = await caches.open(options.cacheName || 'dynamic');
        const cached = await cache.match(request);
        
        const fetchPromise = fetch(request)
            .then((response) => {
                if (response.ok) {
                    cache.put(request, response.clone());
                }
                return response;
            })
            .catch(() => null);
        
        return cached || fetchPromise;
    }
}

// Usage
const router = new CacheRouter();

router.register('/api/', 'network-first', { cacheName: 'api' });
router.register('/images/', 'cache-first', { cacheName: 'images' });
router.register(/\.css$/, 'stale-while-revalidate', { cacheName: 'styles' });
router.register(/\.js$/, 'stale-while-revalidate', { cacheName: 'scripts' });

self.addEventListener('fetch', (event) => {
    event.respondWith(router.handle(event.request));
});
```

## Cache Management

### Cache Size Limits

```javascript
// sw.js - Cache size management
class CacheManager {
    constructor(cacheName, maxItems = 50, maxAge = 7 * 24 * 60 * 60 * 1000) {
        this.cacheName = cacheName;
        this.maxItems = maxItems;
        this.maxAge = maxAge;
    }
    
    async put(request, response) {
        const cache = await caches.open(this.cacheName);
        
        // Add timestamp to response
        const headers = new Headers(response.headers);
        headers.append('sw-cache-time', Date.now().toString());
        
        const modifiedResponse = new Response(response.body, {
            status: response.status,
            statusText: response.statusText,
            headers
        });
        
        await cache.put(request, modifiedResponse);
        await this.cleanup();
    }
    
    async cleanup() {
        const cache = await caches.open(this.cacheName);
        const requests = await cache.keys();
        
        // Remove old entries
        const now = Date.now();
        
        for (const request of requests) {
            const response = await cache.match(request);
            const cacheTime = response.headers.get('sw-cache-time');
            
            if (cacheTime && (now - parseInt(cacheTime)) > this.maxAge) {
                await cache.delete(request);
                console.log('[SW] Removed old cache entry:', request.url);
            }
        }
        
        // Enforce size limit
        const remainingRequests = await cache.keys();
        
        if (remainingRequests.length > this.maxItems) {
            const toRemove = remainingRequests.length - this.maxItems;
            
            for (let i = 0; i < toRemove; i++) {
                await cache.delete(remainingRequests[i]);
                console.log('[SW] Removed cache entry (size limit):', 
                    remainingRequests[i].url);
            }
        }
    }
    
    async clear() {
        await caches.delete(this.cacheName);
    }
}

// Usage
const imageCache = new CacheManager('images', 100, 30 * 24 * 60 * 60 * 1000);
const apiCache = new CacheManager('api', 50, 24 * 60 * 60 * 1000);
```

## Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| Service Workers | 40+ | 44+ | 11.1+ | 17+ |
| Background Sync | 49+ | ❌ | ❌ | 79+ |
| Periodic Background Sync | 80+ | ❌ | ❌ | 80+ |
| Push API | 42+ | 44+ | 16+ | 17+ |

## Common Mistakes

1. **Not handling update properly**
```javascript
// Wrong - old worker stays active
self.addEventListener('install', (event) => {
    // Don't call skipWaiting()
});

// Correct - activate immediately
self.addEventListener('install', (event) => {
    self.skipWaiting();
});
```

2. **Forgetting to claim clients**
```javascript
// Wrong - new worker doesn't control pages
self.addEventListener('activate', (event) => {
    // Clean up only
});

// Correct - take control
self.addEventListener('activate', (event) => {
    event.waitUntil(self.clients.claim());
});
```

3. **Not cleaning up old caches**
```javascript
// Wrong - old caches accumulate
// ...

// Correct - delete old versions
caches.keys().then(names => {
    return Promise.all(
        names
            .filter(name => name !== CURRENT_CACHE)
            .map(name => caches.delete(name))
    );
});
```

## Best Practices

1. **Version Cache Names** - Easy cleanup
2. **Use skipWaiting() Carefully** - May break active pages
3. **Implement Update UI** - Notify users of updates
4. **Clean Up Old Caches** - Prevent storage bloat
5. **Handle Failed Syncs** - Implement retry logic
6. **Test Offline** - Verify offline functionality
7. **Monitor Performance** - Track sync and cache metrics
8. **Handle Permissions** - Request notification permission properly

## Interview Questions

1. What is the service worker lifecycle?
2. How does background sync work?
3. What is periodic background sync?
4. How do you handle service worker updates?
5. What are the different caching strategies?
6. How do push notifications work?
7. What is cache eviction?
8. How do you debug service workers?

## Key Takeaways

- Service workers enable offline functionality
- Background sync ensures data eventually reaches server
- Periodic sync enables scheduled updates
- Push notifications require user permission
- Multiple caching strategies serve different needs
- Cache management prevents storage bloat
- Proper update handling improves user experience
- Testing offline scenarios is crucial

## Resources

- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [Background Sync API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API)
- [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- [Workbox](https://developers.google.com/web/tools/workbox)
- [Service Workers: an Introduction](https://developers.google.com/web/fundamentals/primers/service-workers)
