# Offline Strategies

## Introduction

Offline strategies are critical for creating resilient Progressive Web Apps that work regardless of network conditions. A well-designed offline strategy ensures users can continue working even when connectivity is poor or absent, with changes syncing automatically when connectivity returns.

This guide covers comprehensive offline patterns, from basic caching to sophisticated sync strategies, offline-first architectures, and conflict resolution for collaborative applications.

## Cache-First Offline Strategy

### Basic Implementation

```javascript
// sw.js - Cache first strategy
const CACHE_NAME = 'offline-v1';
const OFFLINE_URL = '/offline.html';

const PRECACHE_ASSETS = [
    '/',
    '/index.html',
    '/offline.html',
    '/css/app.css',
    '/js/app.js',
    '/images/logo.png'
];

self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then((cache) => cache.addAll(PRECACHE_ASSETS))
            .then(() => self.skipWaiting())
    );
});

self.addEventListener('fetch', (event) => {
    if (event.request.mode === 'navigate') {
        event.respondWith(
            fetch(event.request)
                .catch(() => caches.match(OFFLINE_URL))
        );
    } else {
        event.respondWith(
            caches.match(event.request)
                .then((response) => response || fetch(event.request))
        );
    }
});
```

## Offline-First Architecture

### Application State Management

```javascript
// offline-store.js
class OfflineStore {
    constructor(dbName = 'offline-db', version = 1) {
        this.dbName = dbName;
        this.version = version;
        this.db = null;
    }
    
    async init() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(this.dbName, this.version);
            
            request.onerror = () => reject(request.error);
            request.onsuccess = () => {
                this.db = request.result;
                resolve(this.db);
            };
            
            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                
                // Create stores
                if (!db.objectStoreNames.contains('data')) {
                    const store = db.createObjectStore('data', { keyPath: 'id' });
                    store.createIndex('timestamp', 'timestamp', { unique: false });
                    store.createIndex('synced', 'synced', { unique: false });
                }
                
                if (!db.objectStoreNames.contains('queue')) {
                    db.createObjectStore('queue', { keyPath: 'id', autoIncrement: true });
                }
            };
        });
    }
    
    async save(storeName, data) {
        const tx = this.db.transaction([storeName], 'readwrite');
        const store = tx.objectStore(storeName);
        
        const item = {
            ...data,
            timestamp: Date.now(),
            synced: false
        };
        
        await store.put(item);
        return item;
    }
    
    async get(storeName, id) {
        const tx = this.db.transaction([storeName], 'readonly');
        const store = tx.objectStore(storeName);
        return store.get(id);
    }
    
    async getAll(storeName) {
        const tx = this.db.transaction([storeName], 'readonly');
        const store = tx.objectStore(storeName);
        return store.getAll();
    }
    
    async getUnsynced(storeName) {
        const tx = this.db.transaction([storeName], 'readonly');
        const store = tx.objectStore(storeName);
        const index = store.index('synced');
        return index.getAll(false);
    }
    
    async delete(storeName, id) {
        const tx = this.db.transaction([storeName], 'readwrite');
        const store = tx.objectStore(storeName);
        return store.delete(id);
    }
    
    async markSynced(storeName, id) {
        const item = await this.get(storeName, id);
        if (item) {
            item.synced = true;
            await this.save(storeName, item);
        }
    }
    
    async queueOperation(operation) {
        const tx = this.db.transaction(['queue'], 'readwrite');
        const store = tx.objectStore('queue');
        
        const item = {
            ...operation,
            timestamp: Date.now(),
            retries: 0
        };
        
        await store.add(item);
        return item;
    }
    
    async getQueue() {
        const tx = this.db.transaction(['queue'], 'readonly');
        const store = tx.objectStore('queue');
        return store.getAll();
    }
    
    async clearQueue() {
        const tx = this.db.transaction(['queue'], 'readwrite');
        const store = tx.objectStore('queue');
        return store.clear();
    }
}
```

### Sync Manager

```javascript
// sync-manager.js
class SyncManager {
    constructor(store, apiEndpoint) {
        this.store = store;
        this.apiEndpoint = apiEndpoint;
        this.syncing = false;
        this.syncInterval = null;
    }
    
    start(interval = 30000) {
        this.syncInterval = setInterval(() => {
            if (navigator.onLine) {
                this.sync();
            }
        }, interval);
        
        window.addEventListener('online', () => this.sync());
    }
    
    stop() {
        if (this.syncInterval) {
            clearInterval(this.syncInterval);
            this.syncInterval = null;
        }
    }
    
    async sync() {
        if (this.syncing) {
            console.log('Sync already in progress');
            return;
        }
        
        this.syncing = true;
        
        try {
            const queue = await this.store.getQueue();
            
            for (const operation of queue) {
                try {
                    await this.processOperation(operation);
                    await this.store.delete('queue', operation.id);
                } catch (error) {
                    console.error('Operation failed:', operation, error);
                    operation.retries++;
                    
                    if (operation.retries >= 3) {
                        await this.store.delete('queue', operation.id);
                        this.handleFailedOperation(operation);
                    } else {
                        await this.store.save('queue', operation);
                    }
                }
            }
            
            console.log('Sync completed');
            this.dispatchEvent('sync-complete');
        } finally {
            this.syncing = false;
        }
    }
    
    async processOperation(operation) {
        const { type, endpoint, data, method = 'POST' } = operation;
        
        const response = await fetch(`${this.apiEndpoint}${endpoint}`, {
            method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(data)
        });
        
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        return response.json();
    }
    
    handleFailedOperation(operation) {
        console.error('Operation failed permanently:', operation);
        this.dispatchEvent('operation-failed', operation);
    }
    
    dispatchEvent(type, detail = {}) {
        window.dispatchEvent(new CustomEvent(type, { detail }));
    }
}
```

## Conflict Resolution

### Last-Write-Wins Strategy

```javascript
// conflict-resolver.js
class ConflictResolver {
    static lastWriteWins(local, remote) {
        return local.timestamp > remote.timestamp ? local : remote;
    }
    
    static firstWriteWins(local, remote) {
        return local.timestamp < remote.timestamp ? local : remote;
    }
    
    static merge(local, remote) {
        const merged = { ...remote };
        
        Object.keys(local).forEach((key) => {
            if (key === 'timestamp' || key === 'version') return;
            
            if (local[key] !== remote[key]) {
                if (Array.isArray(local[key]) && Array.isArray(remote[key])) {
                    merged[key] = [...new Set([...local[key], ...remote[key]])];
                } else if (typeof local[key] === 'object' && typeof remote[key] === 'object') {
                    merged[key] = this.merge(local[key], remote[key]);
                } else {
                    merged[key] = local.timestamp > remote.timestamp ? local[key] : remote[key];
                }
            }
        });
        
        merged.timestamp = Math.max(local.timestamp, remote.timestamp);
        return merged;
    }
    
    static async resolveConflict(local, remote, strategy = 'last-write-wins') {
        switch (strategy) {
            case 'last-write-wins':
                return this.lastWriteWins(local, remote);
            case 'first-write-wins':
                return this.firstWriteWins(local, remote);
            case 'merge':
                return this.merge(local, remote);
            case 'manual':
                return this.promptUser(local, remote);
            default:
                return this.lastWriteWins(local, remote);
        }
    }
    
    static async promptUser(local, remote) {
        return new Promise((resolve) => {
            const modal = document.createElement('div');
            modal.className = 'conflict-modal';
            modal.innerHTML = `
                <div class="modal-content">
                    <h3>Conflict Detected</h3>
                    <div class="conflict-options">
                        <div class="option">
                            <h4>Your Version</h4>
                            <pre>${JSON.stringify(local, null, 2)}</pre>
                            <button onclick="resolveLocal()">Use This</button>
                        </div>
                        <div class="option">
                            <h4>Server Version</h4>
                            <pre>${JSON.stringify(remote, null, 2)}</pre>
                            <button onclick="resolveRemote()">Use This</button>
                        </div>
                    </div>
                </div>
            `;
            
            window.resolveLocal = () => {
                document.body.removeChild(modal);
                resolve(local);
            };
            
            window.resolveRemote = () => {
                document.body.removeChild(modal);
                resolve(remote);
            };
            
            document.body.appendChild(modal);
        });
    }
}
```

## Offline UI Patterns

### Connection Status Indicator

```javascript
// connection-status.js
class ConnectionStatus {
    constructor() {
        this.indicator = null;
        this.status = navigator.onLine ? 'online' : 'offline';
        this.setupListeners();
        this.createIndicator();
    }
    
    setupListeners() {
        window.addEventListener('online', () => this.update('online'));
        window.addEventListener('offline', () => this.update('offline'));
        
        // Detect slow connection
        if ('connection' in navigator) {
            navigator.connection.addEventListener('change', () => {
                const type = navigator.connection.effectiveType;
                if (type === 'slow-2g' || type === '2g') {
                    this.update('slow');
                }
            });
        }
    }
    
    createIndicator() {
        this.indicator = document.createElement('div');
        this.indicator.className = 'connection-indicator';
        this.update(this.status);
        document.body.appendChild(this.indicator);
    }
    
    update(status) {
        this.status = status;
        
        const messages = {
            online: 'Connected',
            offline: 'Offline - Changes will sync when online',
            slow: 'Slow connection detected'
        };
        
        const classes = {
            online: 'status-online',
            offline: 'status-offline',
            slow: 'status-slow'
        };
        
        this.indicator.textContent = messages[status];
        this.indicator.className = `connection-indicator ${classes[status]}`;
        
        if (status === 'online') {
            setTimeout(() => {
                this.indicator.classList.add('hidden');
            }, 3000);
        } else {
            this.indicator.classList.remove('hidden');
        }
    }
}

// Initialize
const connectionStatus = new ConnectionStatus();
```

### Offline Queue UI

```javascript
// queue-ui.js
class QueueUI {
    constructor(store) {
        this.store = store;
        this.container = null;
        this.init();
    }
    
    init() {
        this.container = document.createElement('div');
        this.container.className = 'queue-container';
        document.body.appendChild(this.container);
        
        this.update();
        
        window.addEventListener('sync-complete', () => this.update());
        window.addEventListener('operation-queued', () => this.update());
    }
    
    async update() {
        const queue = await this.store.getQueue();
        
        if (queue.length === 0) {
            this.container.classList.add('hidden');
            return;
        }
        
        this.container.classList.remove('hidden');
        this.container.innerHTML = `
            <div class="queue-header">
                <span>${queue.length} pending operation${queue.length > 1 ? 's' : ''}</span>
                <button onclick="queueUI.retry()">Retry Now</button>
            </div>
            <ul class="queue-list">
                ${queue.map((op) => `
                    <li class="queue-item">
                        <span>${op.type}: ${op.endpoint}</span>
                        <span class="timestamp">${this.formatTime(op.timestamp)}</span>
                    </li>
                `).join('')}
            </ul>
        `;
    }
    
    formatTime(timestamp) {
        const delta = Date.now() - timestamp;
        const minutes = Math.floor(delta / 60000);
        
        if (minutes < 1) return 'Just now';
        if (minutes === 1) return '1 minute ago';
        if (minutes < 60) return `${minutes} minutes ago`;
        
        const hours = Math.floor(minutes / 60);
        if (hours === 1) return '1 hour ago';
        return `${hours} hours ago`;
    }
    
    async retry() {
        if (navigator.onLine) {
            window.dispatchEvent(new CustomEvent('force-sync'));
        } else {
            alert('Cannot sync while offline');
        }
    }
}
```

## Complete Offline Application Example

```javascript
// offline-app.js
class OfflineApp {
    constructor() {
        this.store = new OfflineStore();
        this.syncManager = null;
        this.connectionStatus = null;
        this.queueUI = null;
    }
    
    async init() {
        await this.store.init();
        
        this.syncManager = new SyncManager(this.store, '/api');
        this.syncManager.start();
        
        this.connectionStatus = new ConnectionStatus();
        this.queueUI = new QueueUI(this.store);
        
        this.setupEventListeners();
        
        console.log('Offline app initialized');
    }
    
    setupEventListeners() {
        window.addEventListener('force-sync', () => {
            this.syncManager.sync();
        });
        
        window.addEventListener('sync-complete', () => {
            console.log('Sync completed');
            this.loadData();
        });
        
        window.addEventListener('operation-failed', (event) => {
            console.error('Operation failed:', event.detail);
            this.showNotification('Failed to sync some changes');
        });
    }
    
    async createItem(data) {
        const item = await this.store.save('data', {
            id: Date.now(),
            ...data
        });
        
        await this.store.queueOperation({
            type: 'CREATE',
            endpoint: '/items',
            method: 'POST',
            data: item
        });
        
        window.dispatchEvent(new CustomEvent('operation-queued'));
        
        if (navigator.onLine) {
            this.syncManager.sync();
        }
        
        return item;
    }
    
    async updateItem(id, updates) {
        const item = await this.store.get('data', id);
        
        if (!item) {
            throw new Error('Item not found');
        }
        
        const updated = await this.store.save('data', {
            ...item,
            ...updates
        });
        
        await this.store.queueOperation({
            type: 'UPDATE',
            endpoint: `/items/${id}`,
            method: 'PUT',
            data: updated
        });
        
        window.dispatchEvent(new CustomEvent('operation-queued'));
        
        if (navigator.onLine) {
            this.syncManager.sync();
        }
        
        return updated;
    }
    
    async deleteItem(id) {
        await this.store.delete('data', id);
        
        await this.store.queueOperation({
            type: 'DELETE',
            endpoint: `/items/${id}`,
            method: 'DELETE',
            data: { id }
        });
        
        window.dispatchEvent(new CustomEvent('operation-queued'));
        
        if (navigator.onLine) {
            this.syncManager.sync();
        }
    }
    
    async loadData() {
        const items = await this.store.getAll('data');
        this.renderItems(items);
    }
    
    renderItems(items) {
        const container = document.getElementById('items');
        container.innerHTML = items.map((item) => `
            <div class="item ${item.synced ? '' : 'pending'}">
                <h3>${item.title}</h3>
                <p>${item.description}</p>
                <span class="sync-status">
                    ${item.synced ? '✓ Synced' : '⏳ Pending'}
                </span>
            </div>
        `).join('');
    }
    
    showNotification(message) {
        const notification = document.createElement('div');
        notification.className = 'notification';
        notification.textContent = message;
        document.body.appendChild(notification);
        
        setTimeout(() => {
            notification.classList.add('show');
        }, 100);
        
        setTimeout(() => {
            notification.classList.remove('show');
            setTimeout(() => notification.remove(), 300);
        }, 3000);
    }
}

// Initialize app
const app = new OfflineApp();
app.init().then(() => {
    app.loadData();
});
```

## Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| Service Workers | 40+ | 44+ | 11.1+ | 17+ |
| IndexedDB | 24+ | 16+ | 10+ | 12+ |
| Background Sync | 49+ | ❌ | ❌ | 79+ |
| Online/Offline Events | All | All | All | All |

## Common Mistakes

1. **Not handling pending changes**
2. **Forgetting conflict resolution**
3. **Poor offline UX**
4. **Not testing offline scenarios**
5. **Inadequate error handling**

## Best Practices

1. **Design Offline-First** - Assume offline by default
2. **Queue Operations** - Never lose user data
3. **Show Sync Status** - Keep users informed
4. **Handle Conflicts** - Plan for concurrent edits
5. **Test Thoroughly** - Simulate various network conditions
6. **Optimize Storage** - Clean up old data
7. **Provide Feedback** - Clear UI indicators
8. **Plan for Failures** - Graceful degradation

## Interview Questions

1. What is offline-first architecture?
2. How do you handle conflicts in offline apps?
3. What are different sync strategies?
4. How do you queue offline operations?
5. What is optimistic UI?
6. How do you test offline functionality?
7. What is IndexedDB and when to use it?
8. How do you handle partial connectivity?

## Key Takeaways

- Offline-first improves resilience and UX
- Queue operations for later sync
- Conflict resolution is essential
- Clear UI feedback is critical
- IndexedDB stores structured data
- Test various network conditions
- Plan for sync failures
- Background sync enables automatic sync

## Resources

- [Offline First](http://offlinefirst.org/)
- [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Background Sync](https://developers.google.com/web/updates/2015/12/background-sync)
- [Workbox](https://developers.google.com/web/tools/workbox)
- [Building Offline-First Apps](https://web.dev/offline-cookbook/)
