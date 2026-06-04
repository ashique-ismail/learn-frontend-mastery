# State Persistence

## Overview

State persistence involves storing application state across browser sessions, page refreshes, and even device switches. While memory-based state disappears when users close their browser, persisted state provides continuity and improved UX. This includes strategies for localStorage, sessionStorage, IndexedDB, cookies, and handling state hydration, migrations, and synchronization across tabs. Proper persistence implementation enables offline-first apps, draft autosaving, user preferences, and seamless multi-device experiences.

## Core Concepts

### Storage Options Comparison

```
┌─────────────────────────────────────────────────────────────┐
│              Browser Storage Options                         │
└─────────────────────────────────────────────────────────────┘

localStorage
├─ Size: ~5-10MB
├─ Lifetime: Persistent (until cleared)
├─ Scope: Same origin
├─ API: Synchronous
└─ Use: Preferences, settings, small data

sessionStorage
├─ Size: ~5-10MB
├─ Lifetime: Tab session
├─ Scope: Same origin + tab
├─ API: Synchronous
└─ Use: Temporary state, wizard data

IndexedDB
├─ Size: ~50MB+ (can request more)
├─ Lifetime: Persistent
├─ Scope: Same origin
├─ API: Asynchronous
└─ Use: Large datasets, offline data

Cookies
├─ Size: ~4KB per cookie
├─ Lifetime: Configurable expiry
├─ Scope: Domain + path
├─ API: String parsing
└─ Use: Server communication, auth tokens
```

## localStorage Persistence

### Basic localStorage with React

```typescript
import React, { useState, useEffect } from 'react';

function useLocalStorage<T>(key: string, initialValue: T) {
  // Get initial value from localStorage or use initialValue
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(`Error loading ${key} from localStorage:`, error);
      return initialValue;
    }
  });

  // Update localStorage when state changes
  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(`Error saving ${key} to localStorage:`, error);
    }
  };

  return [storedValue, setValue] as const;
}

// Usage
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current theme: {theme}
    </button>
  );
}
```

### Advanced localStorage Hook with Events

```typescript
function useLocalStorageWithSync<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });

  // Listen for storage events from other tabs/windows
  useEffect(() => {
    const handleStorageChange = (e: StorageEvent) => {
      if (e.key === key && e.newValue) {
        try {
          setStoredValue(JSON.parse(e.newValue));
        } catch (error) {
          console.error('Error parsing storage event:', error);
        }
      }
    };

    window.addEventListener('storage', handleStorageChange);
    return () => window.removeEventListener('storage', handleStorageChange);
  }, [key]);

  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
      
      // Dispatch custom event for same-tab synchronization
      window.dispatchEvent(new CustomEvent(`localStorage:${key}`, {
        detail: valueToStore
      }));
    } catch (error) {
      console.error('Error saving to localStorage:', error);
    }
  };

  const remove = () => {
    try {
      window.localStorage.removeItem(key);
      setStoredValue(initialValue);
    } catch (error) {
      console.error('Error removing from localStorage:', error);
    }
  };

  return [storedValue, setValue, remove] as const;
}
```

### Redux Persist Integration

```typescript
import { createStore } from 'redux';
import { persistStore, persistReducer } from 'redux-persist';
import storage from 'redux-persist/lib/storage'; // localStorage

interface RootState {
  user: {
    profile: any;
    preferences: any;
  };
  cart: {
    items: any[];
  };
  ui: {
    sidebarOpen: boolean;
  };
}

const persistConfig = {
  key: 'root',
  storage,
  whitelist: ['user', 'cart'], // Only persist these reducers
  blacklist: ['ui'], // Don't persist UI state
};

const rootReducer = (state: RootState, action: any) => {
  // Your root reducer logic
  return state;
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = createStore(persistedReducer);
export const persistor = persistStore(store);

// In your app
import { PersistGate } from 'redux-persist/integration/react';

function App() {
  return (
    <Provider store={store}>
      <PersistGate loading={<div>Loading...</div>} persistor={persistor}>
        <YourApp />
      </PersistGate>
    </Provider>
  );
}
```

### Zustand with Persistence

```typescript
import create from 'zustand';
import { persist } from 'zustand/middleware';

interface TodoStore {
  todos: Array<{ id: string; text: string; completed: boolean }>;
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
  deleteTodo: (id: string) => void;
}

const useTodoStore = create<TodoStore>()(
  persist(
    (set) => ({
      todos: [],
      
      addTodo: (text) =>
        set((state) => ({
          todos: [
            ...state.todos,
            { id: Date.now().toString(), text, completed: false }
          ]
        })),
      
      toggleTodo: (id) =>
        set((state) => ({
          todos: state.todos.map((todo) =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo
          )
        })),
      
      deleteTodo: (id) =>
        set((state) => ({
          todos: state.todos.filter((todo) => todo.id !== id)
        }))
    }),
    {
      name: 'todo-storage', // localStorage key
      // Optional: customize storage
      storage: {
        getItem: (name) => {
          const str = localStorage.getItem(name);
          return str ? JSON.parse(str) : null;
        },
        setItem: (name, value) => {
          localStorage.setItem(name, JSON.stringify(value));
        },
        removeItem: (name) => {
          localStorage.removeItem(name);
        }
      }
    }
  )
);
```

## IndexedDB for Large Data

### IndexedDB Wrapper

```typescript
class IndexedDBStorage {
  private db: IDBDatabase | null = null;
  private dbName: string;
  private storeName: string;

  constructor(dbName: string, storeName: string) {
    this.dbName = dbName;
    this.storeName = storeName;
  }

  async init(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, 1);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        
        if (!db.objectStoreNames.contains(this.storeName)) {
          const objectStore = db.createObjectStore(this.storeName, {
            keyPath: 'id',
            autoIncrement: true
          });
          
          // Create indexes
          objectStore.createIndex('timestamp', 'timestamp', { unique: false });
        }
      };
    });
  }

  async set<T>(key: string, value: T): Promise<void> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([this.storeName], 'readwrite');
      const store = transaction.objectStore(this.storeName);
      
      const request = store.put({
        id: key,
        value,
        timestamp: Date.now()
      });

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async get<T>(key: string): Promise<T | null> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([this.storeName], 'readonly');
      const store = transaction.objectStore(this.storeName);
      const request = store.get(key);

      request.onsuccess = () => {
        const result = request.result;
        resolve(result ? result.value : null);
      };
      request.onerror = () => reject(request.error);
    });
  }

  async getAll<T>(): Promise<T[]> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([this.storeName], 'readonly');
      const store = transaction.objectStore(this.storeName);
      const request = store.getAll();

      request.onsuccess = () => {
        resolve(request.result.map(item => item.value));
      };
      request.onerror = () => reject(request.error);
    });
  }

  async delete(key: string): Promise<void> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([this.storeName], 'readwrite');
      const store = transaction.objectStore(this.storeName);
      const request = store.delete(key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async clear(): Promise<void> {
    if (!this.db) await this.init();

    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([this.storeName], 'readwrite');
      const store = transaction.objectStore(this.storeName);
      const request = store.clear();

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }
}

// Usage
const storage = new IndexedDBStorage('myApp', 'documents');

async function saveDocument(doc: any) {
  await storage.set(doc.id, doc);
}

async function loadDocument(id: string) {
  return await storage.get(id);
}
```

### React Hook for IndexedDB

```typescript
import { useState, useEffect } from 'react';

function useIndexedDB<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(initialValue);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const storage = useRef(new IndexedDBStorage('myApp', 'store'));

  useEffect(() => {
    const loadValue = async () => {
      try {
        const stored = await storage.current.get<T>(key);
        if (stored !== null) {
          setValue(stored);
        }
      } catch (err) {
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    };

    loadValue();
  }, [key]);

  const updateValue = async (newValue: T | ((prev: T) => T)) => {
    try {
      const valueToStore =
        newValue instanceof Function ? newValue(value) : newValue;
      
      setValue(valueToStore);
      await storage.current.set(key, valueToStore);
    } catch (err) {
      setError(err as Error);
    }
  };

  return { value, setValue: updateValue, loading, error };
}

// Usage
function DocumentEditor() {
  const { value: document, setValue: setDocument, loading } = useIndexedDB(
    'my-document',
    { content: '', lastModified: Date.now() }
  );

  if (loading) return <div>Loading...</div>;

  return (
    <textarea
      value={document.content}
      onChange={(e) =>
        setDocument({
          content: e.target.value,
          lastModified: Date.now()
        })
      }
    />
  );
}
```

## State Hydration

### Server State Hydration

```typescript
// Server-side rendering (Next.js example)
export async function getServerSideProps() {
  const initialState = {
    user: await fetchUser(),
    posts: await fetchPosts()
  };

  return {
    props: {
      initialState
    }
  };
}

// Client-side hydration
function MyApp({ Component, pageProps }) {
  const [store] = useState(() => {
    // Create store with server data
    const store = createStore(rootReducer, pageProps.initialState);
    return store;
  });

  return (
    <Provider store={store}>
      <Component {...pageProps} />
    </Provider>
  );
}
```

### Hydration with Version Checking

```typescript
interface PersistedState<T> {
  version: number;
  data: T;
  timestamp: number;
}

function usePersistedState<T>(
  key: string,
  initialValue: T,
  version: number
) {
  const [state, setState] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      if (!item) return initialValue;

      const persisted: PersistedState<T> = JSON.parse(item);

      // Check version compatibility
      if (persisted.version !== version) {
        console.log('Version mismatch, using initial value');
        return initialValue;
      }

      // Check if data is stale (e.g., older than 7 days)
      const age = Date.now() - persisted.timestamp;
      const maxAge = 7 * 24 * 60 * 60 * 1000; // 7 days
      
      if (age > maxAge) {
        console.log('Data is stale, using initial value');
        return initialValue;
      }

      return persisted.data;
    } catch (error) {
      console.error('Error hydrating state:', error);
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(state) : value;
      setState(valueToStore);

      const persisted: PersistedState<T> = {
        version,
        data: valueToStore,
        timestamp: Date.now()
      };

      localStorage.setItem(key, JSON.stringify(persisted));
    } catch (error) {
      console.error('Error persisting state:', error);
    }
  };

  return [state, setValue] as const;
}

// Usage
const SETTINGS_VERSION = 2;

function Settings() {
  const [settings, setSettings] = usePersistedState(
    'app-settings',
    { theme: 'light', notifications: true },
    SETTINGS_VERSION
  );

  // If settings structure changes, increment SETTINGS_VERSION
  // Old data will be ignored and initial value used
}
```

## State Migrations

### Migration System

```typescript
interface Migration<T> {
  version: number;
  migrate: (oldState: any) => T;
}

class StateMigration<T> {
  private migrations: Migration<T>[];

  constructor(migrations: Migration<T>[]) {
    this.migrations = migrations.sort((a, b) => a.version - b.version);
  }

  migrate(persistedData: any, currentVersion: number): T {
    if (!persistedData || !persistedData.version) {
      throw new Error('Invalid persisted data');
    }

    let state = persistedData.data;
    const fromVersion = persistedData.version;

    // Apply all migrations from stored version to current version
    for (const migration of this.migrations) {
      if (migration.version > fromVersion && migration.version <= currentVersion) {
        console.log(`Migrating from v${fromVersion} to v${migration.version}`);
        state = migration.migrate(state);
      }
    }

    return state;
  }
}

// Define migrations
const userStateMigrations = new StateMigration([
  {
    version: 2,
    migrate: (oldState) => ({
      ...oldState,
      // v2: Added preferences field
      preferences: { emailNotifications: true }
    })
  },
  {
    version: 3,
    migrate: (oldState) => ({
      ...oldState,
      // v3: Renamed field
      settings: oldState.preferences,
      preferences: undefined
    })
  },
  {
    version: 4,
    migrate: (oldState) => ({
      // v4: Complete restructure
      profile: {
        name: oldState.name,
        email: oldState.email
      },
      settings: oldState.settings
    })
  }
]);

// Usage
function usePersistedStateWithMigration<T>(
  key: string,
  initialValue: T,
  currentVersion: number,
  migrations: StateMigration<T>
) {
  const [state, setState] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      if (!item) return initialValue;

      const persisted = JSON.parse(item);
      
      if (persisted.version === currentVersion) {
        return persisted.data;
      }

      // Perform migration
      return migrations.migrate(persisted, currentVersion);
    } catch (error) {
      console.error('Migration failed:', error);
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(state) : value;
    setState(valueToStore);

    localStorage.setItem(key, JSON.stringify({
      version: currentVersion,
      data: valueToStore,
      timestamp: Date.now()
    }));
  };

  return [state, setValue] as const;
}
```

## Angular State Persistence

### Angular Service with localStorage

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

interface AppState {
  user: any;
  theme: string;
  preferences: any;
}

@Injectable({ providedIn: 'root' })
export class PersistentStateService {
  private readonly STORAGE_KEY = 'app-state';
  private readonly VERSION = 1;

  private stateSubject: BehaviorSubject<AppState>;
  public state$: Observable<AppState>;

  constructor() {
    const initialState = this.loadState();
    this.stateSubject = new BehaviorSubject<AppState>(initialState);
    this.state$ = this.stateSubject.asObservable();

    // Auto-save on state changes
    this.state$.subscribe(state => {
      this.saveState(state);
    });
  }

  private loadState(): AppState {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      if (!stored) return this.getDefaultState();

      const parsed = JSON.parse(stored);
      
      if (parsed.version !== this.VERSION) {
        console.log('Version mismatch, using default state');
        return this.getDefaultState();
      }

      return parsed.data;
    } catch (error) {
      console.error('Error loading state:', error);
      return this.getDefaultState();
    }
  }

  private saveState(state: AppState): void {
    try {
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify({
        version: this.VERSION,
        data: state,
        timestamp: Date.now()
      }));
    } catch (error) {
      console.error('Error saving state:', error);
    }
  }

  private getDefaultState(): AppState {
    return {
      user: null,
      theme: 'light',
      preferences: {}
    };
  }

  updateState(updates: Partial<AppState>): void {
    const currentState = this.stateSubject.value;
    this.stateSubject.next({ ...currentState, ...updates });
  }

  getCurrentState(): AppState {
    return this.stateSubject.value;
  }

  clearState(): void {
    localStorage.removeItem(this.STORAGE_KEY);
    this.stateSubject.next(this.getDefaultState());
  }
}

// Component usage
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-settings',
  template: `
    <div>
      <h2>Settings</h2>
      <label>
        <input
          type="radio"
          value="light"
          [checked]="(state$ | async)?.theme === 'light'"
          (change)="updateTheme('light')"
        />
        Light
      </label>
      <label>
        <input
          type="radio"
          value="dark"
          [checked]="(state$ | async)?.theme === 'dark'"
          (change)="updateTheme('dark')"
        />
        Dark
      </label>
    </div>
  `
})
export class SettingsComponent {
  state$ = this.stateService.state$;

  constructor(private stateService: PersistentStateService) {}

  updateTheme(theme: string): void {
    this.stateService.updateState({ theme });
  }
}
```

## Common Mistakes

### 1. Storing Sensitive Data

```typescript
// ❌ Wrong - localStorage is not secure!
localStorage.setItem('password', userPassword);
localStorage.setItem('creditCard', cardNumber);

// ✅ Correct - never store sensitive data client-side
// Use server-side sessions or encrypted tokens
```

### 2. Not Handling Storage Quota Errors

```typescript
// ❌ Wrong - no error handling
localStorage.setItem('data', hugeString);

// ✅ Correct - catch quota errors
try {
  localStorage.setItem('data', hugeString);
} catch (error) {
  if (error.name === 'QuotaExceededError') {
    // Clear old data or notify user
    handleStorageQuotaExceeded();
  }
}
```

### 3. Storing Too Much Data in localStorage

```typescript
// ❌ Wrong - localStorage for large datasets
localStorage.setItem('images', JSON.stringify(imageArray));

// ✅ Correct - use IndexedDB for large data
await indexedDB.set('images', imageArray);
```

### 4. Not Validating Persisted Data

```typescript
// ❌ Wrong - blindly trusting stored data
const data = JSON.parse(localStorage.getItem('data'));

// ✅ Correct - validate and provide fallback
const data = (() => {
  try {
    const parsed = JSON.parse(localStorage.getItem('data') || '');
    return validateSchema(parsed) ? parsed : defaultData;
  } catch {
    return defaultData;
  }
})();
```

### 5. Forgetting Cross-Tab Synchronization

```typescript
// ❌ Wrong - changes in one tab don't sync to others
localStorage.setItem('theme', 'dark');

// ✅ Correct - listen for storage events
window.addEventListener('storage', (e) => {
  if (e.key === 'theme') {
    setTheme(e.newValue);
  }
});
```

## Best Practices

### 1. Use Appropriate Storage

- **localStorage**: Small data, preferences, settings
- **sessionStorage**: Temporary, tab-specific data
- **IndexedDB**: Large datasets, offline-first apps
- **Cookies**: Server communication only

### 2. Implement Versioning

```typescript
const STORAGE_VERSION = 2;

const data = {
  version: STORAGE_VERSION,
  data: actualData,
  timestamp: Date.now()
};
```

### 3. Handle Errors Gracefully

```typescript
function safeLocalStorage() {
  try {
    return {
      getItem: (key) => localStorage.getItem(key),
      setItem: (key, value) => localStorage.setItem(key, value),
      removeItem: (key) => localStorage.removeItem(key)
    };
  } catch {
    // Fallback to memory storage
    const memoryStorage = new Map();
    return {
      getItem: (key) => memoryStorage.get(key) || null,
      setItem: (key, value) => memoryStorage.set(key, value),
      removeItem: (key) => memoryStorage.delete(key)
    };
  }
}
```

### 4. Compress Large Data

```typescript
import pako from 'pako';

function compressAndStore(key: string, data: any) {
  const json = JSON.stringify(data);
  const compressed = pako.deflate(json, { to: 'string' });
  localStorage.setItem(key, compressed);
}

function loadAndDecompress(key: string) {
  const compressed = localStorage.getItem(key);
  if (!compressed) return null;
  
  const decompressed = pako.inflate(compressed, { to: 'string' });
  return JSON.parse(decompressed);
}
```

## When to Persist State

### Persist:
- User preferences (theme, language)
- Authentication tokens (with security considerations)
- Draft content (autosave)
- Shopping cart
- Form progress
- UI state (sidebar collapsed, etc.)

### Don't Persist:
- Sensitive data (passwords, credit cards)
- Temporary UI state (modals, tooltips)
- Derived/computed values
- Large datasets better served by cache
- Time-sensitive data

## Interview Questions

### Q1: What's the difference between localStorage and sessionStorage?
**Answer**: localStorage persists indefinitely until explicitly cleared, survives browser restarts, and is shared across all tabs of the same origin. sessionStorage lasts only for the tab session, is cleared when the tab closes, and is isolated to that specific tab. Use localStorage for persistent preferences, sessionStorage for temporary wizard/form data.

### Q2: When should you use IndexedDB over localStorage?
**Answer**: Use IndexedDB for large datasets (>5MB), complex queries with indexes, offline-first apps, or when you need asynchronous access. localStorage is limited to ~5-10MB, synchronous (blocks main thread), and only supports string key-value pairs. IndexedDB can store structured data, blobs, and supports transactions.

### Q3: How do you handle storage quota errors?
**Answer**: Wrap storage operations in try-catch, check for QuotaExceededError, implement LRU eviction of old data, compress data before storing, prompt users to clear space, or fall back to memory storage. Monitor storage usage and warn users proactively before hitting limits.

### Q4: What are state migrations and why are they important?
**Answer**: Migrations transform old state formats to new ones when app structure changes. Without migrations, users lose data when you refactor state. Implement version numbers, define migration functions for each version change, and apply them sequentially. This maintains backward compatibility with persisted data.

### Q5: How do you synchronize state across browser tabs?
**Answer**: Listen to the 'storage' event which fires when localStorage changes in other tabs. For same-tab synchronization, dispatch custom events. For complex scenarios, use BroadcastChannel API or SharedWorker. IndexedDB changes require manual polling or custom sync mechanisms.

## Key Takeaways

1. Choose storage based on data size and persistence needs
2. localStorage for small persistent data, IndexedDB for large datasets
3. Always implement error handling for storage operations
4. Use versioning to handle schema changes
5. Implement migrations for backward compatibility
6. Synchronize across tabs using storage events
7. Never store sensitive data in client-side storage
8. Validate and sanitize persisted data on load
9. Compress large data to save space
10. Provide fallbacks when storage is unavailable

## Resources

- [Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API)
- [IndexedDB Guide](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Redux Persist](https://github.com/rt2zz/redux-persist)
- [Zustand Persist Middleware](https://github.com/pmndrs/zustand#persist-middleware)
- [Storage Quota Management](https://web.dev/storage-for-the-web/)
- [LocalForage (IndexedDB wrapper)](https://localforage.github.io/localForage/)
