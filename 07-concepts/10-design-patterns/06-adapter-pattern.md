# Adapter Pattern

## The Idea

**In plain English:** The Adapter pattern is a way to make two pieces of code "talk" to each other even though they were designed with different rules — just like using a translator between two people who speak different languages, without changing how either person speaks.

**Real-world analogy:** Imagine you travel to a country where wall power outlets have a different shape than your phone charger plug. You buy a travel adapter — a small converter that fits the foreign outlet on one side and accepts your plug on the other. Neither the outlet nor your charger changes at all; the adapter sits in the middle and makes them compatible.

- The foreign wall outlet = the existing code (Adaptee) you cannot change
- Your phone charger plug = the interface your app expects (Target)
- The travel adapter = the Adapter class that connects the two

---

## Overview

The Adapter pattern is a structural design pattern that allows objects with incompatible interfaces to collaborate. It acts as a bridge between two incompatible interfaces by wrapping an existing class with a new interface. This pattern is particularly useful when integrating legacy code, third-party libraries, or when you need to make existing classes work with others without modifying their source code.

## Core Concepts

### Adapter Pattern Structure

```
┌──────────────┐         ┌──────────────┐
│    Client    │────────▶│    Target    │
└──────────────┘         │  «interface» │
                         ├──────────────┤
                         │ + request()  │
                         └──────┬───────┘
                                △
                                │ implements
                         ┌──────┴───────┐
                         │   Adapter    │
                         ├──────────────┤
                         │ - adaptee    │◆────┐
                         │ + request()  │     │
                         └──────────────┘     │
                                              │
                                    ┌─────────▼──────┐
                                    │    Adaptee     │
                                    ├────────────────┤
                                    │+ specificReq() │
                                    └────────────────┘
```

### Key Components

1. **Target**: Interface expected by client
2. **Client**: Collaborates with objects conforming to Target interface
3. **Adaptee**: Existing interface that needs adapting
4. **Adapter**: Wraps Adaptee and implements Target interface

## Implementation Patterns

### Basic Adapter Pattern

```typescript
// Target interface - what the client expects
interface MediaPlayer {
  play(audioType: string, fileName: string): void;
}

// Adaptee - existing incompatible interface
class AdvancedMediaPlayer {
  playVlc(fileName: string): void {
    console.log(`Playing VLC file: ${fileName}`);
  }

  playMp4(fileName: string): void {
    console.log(`Playing MP4 file: ${fileName}`);
  }
}

// Adapter - makes AdvancedMediaPlayer work with MediaPlayer interface
class MediaAdapter implements MediaPlayer {
  private advancedPlayer: AdvancedMediaPlayer;

  constructor(audioType: string) {
    this.advancedPlayer = new AdvancedMediaPlayer();
  }

  play(audioType: string, fileName: string): void {
    if (audioType === 'vlc') {
      this.advancedPlayer.playVlc(fileName);
    } else if (audioType === 'mp4') {
      this.advancedPlayer.playMp4(fileName);
    } else {
      console.log(`Invalid media type: ${audioType}`);
    }
  }
}

// Client class that uses the Target interface
class AudioPlayer implements MediaPlayer {
  private adapter: MediaAdapter | null = null;

  play(audioType: string, fileName: string): void {
    // Built-in support for mp3
    if (audioType === 'mp3') {
      console.log(`Playing MP3 file: ${fileName}`);
    }
    // Use adapter for other formats
    else if (audioType === 'vlc' || audioType === 'mp4') {
      this.adapter = new MediaAdapter(audioType);
      this.adapter.play(audioType, fileName);
    }
    else {
      console.log(`Invalid media type: ${audioType}. Format not supported.`);
    }
  }
}

// Usage
const player = new AudioPlayer();
player.play('mp3', 'song.mp3');      // Playing MP3 file: song.mp3
player.play('mp4', 'video.mp4');     // Playing MP4 file: video.mp4
player.play('vlc', 'movie.vlc');     // Playing VLC file: movie.vlc
player.play('avi', 'video.avi');     // Invalid media type: avi
```

### API Wrapper Adapter

```typescript
// Target interface - modern API
interface UserService {
  getUser(id: string): Promise<User>;
  updateUser(id: string, data: Partial<User>): Promise<User>;
  deleteUser(id: string): Promise<boolean>;
}

interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}

// Legacy API (Adaptee)
class LegacyUserAPI {
  fetchUserData(userId: number): Promise<any> {
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({
          user_id: userId,
          user_name: 'John Doe',
          user_email: 'john@example.com',
          created_timestamp: 1234567890
        });
      }, 100);
    });
  }

  modifyUser(userId: number, changes: any): Promise<any> {
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({
          success: true,
          user_id: userId,
          ...changes
        });
      }, 100);
    });
  }

  removeUser(userId: number): Promise<{ status: string }> {
    return new Promise(resolve => {
      setTimeout(() => {
        resolve({ status: 'deleted' });
      }, 100);
    });
  }
}

// Adapter - converts legacy API to modern interface
class LegacyUserServiceAdapter implements UserService {
  private legacyAPI: LegacyUserAPI;

  constructor() {
    this.legacyAPI = new LegacyUserAPI();
  }

  async getUser(id: string): Promise<User> {
    const legacyData = await this.legacyAPI.fetchUserData(parseInt(id));
    
    // Transform legacy format to modern format
    return {
      id: legacyData.user_id.toString(),
      name: legacyData.user_name,
      email: legacyData.user_email,
      createdAt: new Date(legacyData.created_timestamp * 1000)
    };
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    // Transform modern format to legacy format
    const legacyChanges: any = {};
    if (data.name) legacyChanges.user_name = data.name;
    if (data.email) legacyChanges.user_email = data.email;

    const result = await this.legacyAPI.modifyUser(parseInt(id), legacyChanges);
    
    return {
      id: result.user_id.toString(),
      name: result.user_name,
      email: result.user_email,
      createdAt: new Date()
    };
  }

  async deleteUser(id: string): Promise<boolean> {
    const result = await this.legacyAPI.removeUser(parseInt(id));
    return result.status === 'deleted';
  }
}

// Usage
async function demonstrateAdapter() {
  const userService: UserService = new LegacyUserServiceAdapter();

  const user = await userService.getUser('123');
  console.log('User:', user);

  const updated = await userService.updateUser('123', { name: 'Jane Doe' });
  console.log('Updated:', updated);

  const deleted = await userService.deleteUser('123');
  console.log('Deleted:', deleted);
}
```

### React Component Adapter

```typescript
// Target interface - your app's component interface
interface ModernButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
}

// Legacy third-party button (Adaptee)
interface LegacyButtonProps {
  text: string;
  onPress: () => void;
  type?: 'default' | 'success' | 'danger';
  buttonSize?: 'xs' | 'sm' | 'md' | 'lg';
  isDisabled?: boolean;
  className?: string;
}

const LegacyButton: React.FC<LegacyButtonProps> = ({
  text,
  onPress,
  type = 'default',
  buttonSize = 'md',
  isDisabled = false,
  className
}) => {
  return (
    <button
      onClick={onPress}
      disabled={isDisabled}
      className={`legacy-button ${type} ${buttonSize} ${className || ''}`}
    >
      {text}
    </button>
  );
};

// Adapter - makes LegacyButton work with modern interface
const ButtonAdapter: React.FC<ModernButtonProps> = ({
  label,
  onClick,
  variant = 'primary',
  size = 'medium',
  disabled = false
}) => {
  // Map modern props to legacy props
  const legacyType = variant === 'primary' ? 'success' : 'default';
  const legacySize = {
    small: 'sm',
    medium: 'md',
    large: 'lg'
  }[size] as 'sm' | 'md' | 'lg';

  return (
    <LegacyButton
      text={label}
      onPress={onClick}
      type={legacyType}
      buttonSize={legacySize}
      isDisabled={disabled}
    />
  );
};

// Usage in modern app
const App: React.FC = () => {
  return (
    <div>
      <ButtonAdapter
        label="Click Me"
        onClick={() => console.log('Clicked')}
        variant="primary"
        size="large"
      />
    </div>
  );
};
```

### Storage Adapter Pattern

```typescript
// Target interface - unified storage interface
interface StorageAdapter {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  delete(key: string): Promise<void>;
  clear(): Promise<void>;
  keys(): Promise<string[]>;
}

// LocalStorage Adapter
class LocalStorageAdapter implements StorageAdapter {
  async get(key: string): Promise<string | null> {
    return localStorage.getItem(key);
  }

  async set(key: string, value: string): Promise<void> {
    localStorage.setItem(key, value);
  }

  async delete(key: string): Promise<void> {
    localStorage.removeItem(key);
  }

  async clear(): Promise<void> {
    localStorage.clear();
  }

  async keys(): Promise<string[]> {
    return Object.keys(localStorage);
  }
}

// SessionStorage Adapter
class SessionStorageAdapter implements StorageAdapter {
  async get(key: string): Promise<string | null> {
    return sessionStorage.getItem(key);
  }

  async set(key: string, value: string): Promise<void> {
    sessionStorage.setItem(key, value);
  }

  async delete(key: string): Promise<void> {
    sessionStorage.removeItem(key);
  }

  async clear(): Promise<void> {
    sessionStorage.clear();
  }

  async keys(): Promise<string[]> {
    return Object.keys(sessionStorage);
  }
}

// IndexedDB Adapter
class IndexedDBAdapter implements StorageAdapter {
  private dbName = 'AppStorage';
  private storeName = 'KeyValueStore';

  private async getDB(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, 1);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        if (!db.objectStoreNames.contains(this.storeName)) {
          db.createObjectStore(this.storeName);
        }
      };
    });
  }

  async get(key: string): Promise<string | null> {
    const db = await this.getDB();
    return new Promise((resolve, reject) => {
      const transaction = db.transaction([this.storeName], 'readonly');
      const store = transaction.objectStore(this.storeName);
      const request = store.get(key);

      request.onsuccess = () => resolve(request.result || null);
      request.onerror = () => reject(request.error);
    });
  }

  async set(key: string, value: string): Promise<void> {
    const db = await this.getDB();
    return new Promise((resolve, reject) => {
      const transaction = db.transaction([this.storeName], 'readwrite');
      const store = transaction.objectStore(this.storeName);
      const request = store.put(value, key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async delete(key: string): Promise<void> {
    const db = await this.getDB();
    return new Promise((resolve, reject) => {
      const transaction = db.transaction([this.storeName], 'readwrite');
      const store = transaction.objectStore(this.storeName);
      const request = store.delete(key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async clear(): Promise<void> {
    const db = await this.getDB();
    return new Promise((resolve, reject) => {
      const transaction = db.transaction([this.storeName], 'readwrite');
      const store = transaction.objectStore(this.storeName);
      const request = store.clear();

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  async keys(): Promise<string[]> {
    const db = await this.getDB();
    return new Promise((resolve, reject) => {
      const transaction = db.transaction([this.storeName], 'readonly');
      const store = transaction.objectStore(this.storeName);
      const request = store.getAllKeys();

      request.onsuccess = () => resolve(request.result as string[]);
      request.onerror = () => reject(request.error);
    });
  }
}

// Storage Manager using adapters
class StorageManager {
  private adapter: StorageAdapter;

  constructor(storageType: 'local' | 'session' | 'indexed') {
    switch (storageType) {
      case 'local':
        this.adapter = new LocalStorageAdapter();
        break;
      case 'session':
        this.adapter = new SessionStorageAdapter();
        break;
      case 'indexed':
        this.adapter = new IndexedDBAdapter();
        break;
    }
  }

  async saveData(key: string, value: any): Promise<void> {
    await this.adapter.set(key, JSON.stringify(value));
  }

  async loadData<T>(key: string): Promise<T | null> {
    const value = await this.adapter.get(key);
    return value ? JSON.parse(value) : null;
  }

  async removeData(key: string): Promise<void> {
    await this.adapter.delete(key);
  }

  async clearAll(): Promise<void> {
    await this.adapter.clear();
  }

  async getAllKeys(): Promise<string[]> {
    return await this.adapter.keys();
  }
}

// Usage
async function demonstrateStorage() {
  // Use localStorage
  const localManager = new StorageManager('local');
  await localManager.saveData('user', { name: 'John', age: 30 });
  
  // Use IndexedDB
  const indexedManager = new StorageManager('indexed');
  await indexedManager.saveData('user', { name: 'Jane', age: 25 });
  
  // Same interface, different storage!
  const user1 = await localManager.loadData('user');
  const user2 = await indexedManager.loadData('user');
  
  console.log('LocalStorage user:', user1);
  console.log('IndexedDB user:', user2);
}
```

### HTTP Client Adapter

```typescript
// Target interface - unified HTTP client
interface HttpClient {
  get<T>(url: string, config?: RequestConfig): Promise<T>;
  post<T>(url: string, data: any, config?: RequestConfig): Promise<T>;
  put<T>(url: string, data: any, config?: RequestConfig): Promise<T>;
  delete<T>(url: string, config?: RequestConfig): Promise<T>;
}

interface RequestConfig {
  headers?: Record<string, string>;
  timeout?: number;
  params?: Record<string, string>;
}

// Axios Adapter
import axios, { AxiosInstance } from 'axios';

class AxiosAdapter implements HttpClient {
  private instance: AxiosInstance;

  constructor(baseURL: string = '') {
    this.instance = axios.create({ baseURL });
  }

  async get<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await this.instance.get(url, {
      headers: config?.headers,
      timeout: config?.timeout,
      params: config?.params
    });
    return response.data;
  }

  async post<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await this.instance.post(url, data, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }

  async put<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await this.instance.put(url, data, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }

  async delete<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await this.instance.delete(url, {
      headers: config?.headers,
      timeout: config?.timeout
    });
    return response.data;
  }
}

// Fetch Adapter
class FetchAdapter implements HttpClient {
  constructor(private baseURL: string = '') {}

  private buildURL(url: string, params?: Record<string, string>): string {
    const fullURL = `${this.baseURL}${url}`;
    if (!params) return fullURL;

    const searchParams = new URLSearchParams(params);
    return `${fullURL}?${searchParams.toString()}`;
  }

  async get<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.buildURL(url, config?.params), {
      method: 'GET',
      headers: config?.headers,
      signal: config?.timeout
        ? AbortSignal.timeout(config.timeout)
        : undefined
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    return await response.json();
  }

  async post<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.buildURL(url), {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...config?.headers
      },
      body: JSON.stringify(data),
      signal: config?.timeout
        ? AbortSignal.timeout(config.timeout)
        : undefined
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    return await response.json();
  }

  async put<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.buildURL(url), {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        ...config?.headers
      },
      body: JSON.stringify(data)
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    return await response.json();
  }

  async delete<T>(url: string, config?: RequestConfig): Promise<T> {
    const response = await fetch(this.buildURL(url), {
      method: 'DELETE',
      headers: config?.headers
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    return await response.json();
  }
}

// API Service using adapter
class ApiService {
  constructor(private client: HttpClient) {}

  async getUsers(): Promise<User[]> {
    return this.client.get<User[]>('/users');
  }

  async createUser(userData: Partial<User>): Promise<User> {
    return this.client.post<User>('/users', userData);
  }

  async updateUser(id: string, userData: Partial<User>): Promise<User> {
    return this.client.put<User>(`/users/${id}`, userData);
  }

  async deleteUser(id: string): Promise<void> {
    return this.client.delete<void>(`/users/${id}`);
  }
}

// Usage - easily switch HTTP clients
const axiosService = new ApiService(new AxiosAdapter('https://api.example.com'));
const fetchService = new ApiService(new FetchAdapter('https://api.example.com'));

// Same interface, different implementations!
```

### Angular HTTP Adapter

```typescript
// Angular HTTP Client Adapter
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable, firstValueFrom } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class AngularHttpAdapter implements HttpClient {
  constructor(private http: HttpClient) {}

  async get<T>(url: string, config?: RequestConfig): Promise<T> {
    let params = new HttpParams();
    if (config?.params) {
      Object.entries(config.params).forEach(([key, value]) => {
        params = params.append(key, value);
      });
    }

    const observable = this.http.get<T>(url, {
      headers: config?.headers,
      params
    });

    return firstValueFrom(observable);
  }

  async post<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const observable = this.http.post<T>(url, data, {
      headers: config?.headers
    });

    return firstValueFrom(observable);
  }

  async put<T>(url: string, data: any, config?: RequestConfig): Promise<T> {
    const observable = this.http.put<T>(url, data, {
      headers: config?.headers
    });

    return firstValueFrom(observable);
  }

  async delete<T>(url: string, config?: RequestConfig): Promise<T> {
    const observable = this.http.delete<T>(url, {
      headers: config?.headers
    });

    return firstValueFrom(observable);
  }
}
```

### Two-Way Adapter

```typescript
// Two-way adapter converts between two interfaces
interface MetricSystem {
  getDistanceInKilometers(): number;
  getWeightInKilograms(): number;
}

interface ImperialSystem {
  getDistanceInMiles(): number;
  getWeightInPounds(): number;
}

class MetricData implements MetricSystem {
  constructor(
    private distanceKm: number,
    private weightKg: number
  ) {}

  getDistanceInKilometers(): number {
    return this.distanceKm;
  }

  getWeightInKilograms(): number {
    return this.weightKg;
  }
}

class ImperialData implements ImperialSystem {
  constructor(
    private distanceMiles: number,
    private weightPounds: number
  ) {}

  getDistanceInMiles(): number {
    return this.distanceMiles;
  }

  getWeightInPounds(): number {
    return this.weightPounds;
  }
}

// Two-way adapter
class MeasurementAdapter implements MetricSystem, ImperialSystem {
  constructor(private source: MetricSystem | ImperialSystem) {}

  getDistanceInKilometers(): number {
    if ('getDistanceInKilometers' in this.source) {
      return this.source.getDistanceInKilometers();
    } else {
      // Convert miles to kilometers
      return this.source.getDistanceInMiles() * 1.60934;
    }
  }

  getWeightInKilograms(): number {
    if ('getWeightInKilograms' in this.source) {
      return this.source.getWeightInKilograms();
    } else {
      // Convert pounds to kilograms
      return this.source.getWeightInPounds() * 0.453592;
    }
  }

  getDistanceInMiles(): number {
    if ('getDistanceInMiles' in this.source) {
      return this.source.getDistanceInMiles();
    } else {
      // Convert kilometers to miles
      return this.source.getDistanceInKilometers() / 1.60934;
    }
  }

  getWeightInPounds(): number {
    if ('getWeightInPounds' in this.source) {
      return this.source.getWeightInPounds();
    } else {
      // Convert kilograms to pounds
      return this.source.getWeightInKilograms() / 0.453592;
    }
  }
}

// Usage
const metricData = new MetricData(100, 75);
const adapter = new MeasurementAdapter(metricData);

console.log('Metric:', adapter.getDistanceInKilometers(), 'km');
console.log('Imperial:', adapter.getDistanceInMiles(), 'miles');
console.log('Metric:', adapter.getWeightInKilograms(), 'kg');
console.log('Imperial:', adapter.getWeightInPounds(), 'lbs');
```

## Common Mistakes

### 1. Adapter Adding New Functionality

```typescript
// BAD - Adapter adds business logic
class BadAdapter implements TargetInterface {
  transform(data: any) {
    // Don't add new business logic here!
    const transformed = this.adaptee.process(data);
    const validated = this.validateAndEnrich(transformed);
    return validated;
  }
}

// GOOD - Adapter only translates
class GoodAdapter implements TargetInterface {
  transform(data: any) {
    // Only translate between interfaces
    return this.adaptee.process(data);
  }
}
```

### 2. Creating Unnecessary Adapters

```typescript
// BAD - Adapter when you control both interfaces
class UnnecessaryAdapter {
  // Just fix the interface instead!
}

// GOOD - Only use adapter when you can't modify the adaptee
class NecessaryAdapter {
  // Wrapping third-party or legacy code
  constructor(private legacySystem: LegacyAPI) {}
}
```

### 3. Leaking Adaptee Implementation

```typescript
// BAD - Exposing adaptee details
class BadAdapter implements Target {
  public adaptee: Adaptee; // Don't expose!
  
  method() {
    return this.adaptee.legacyMethod();
  }
}

// GOOD - Encapsulate adaptee
class GoodAdapter implements Target {
  private adaptee: Adaptee; // Private!
  
  method() {
    return this.adaptee.legacyMethod();
  }
}
```

## Best Practices

1. **Use adapters for incompatible interfaces** you can't modify
2. **Keep adapters simple** - only translate, don't add logic
3. **Encapsulate the adaptee** - make it private
4. **Document the adaptation** - explain interface differences
5. **Consider bidirectional adapters** when needed
6. **Use dependency injection** to provide adapters
7. **Test adapter thoroughly** - ensure correct translation
8. **Group related adaptations** in single adapter
9. **Version adapters** when API changes
10. **Consider adapter factory** for multiple adaptee types

## When to Use Adapter Pattern

### Good Use Cases

- Integrating legacy systems
- Working with third-party libraries
- Making incompatible interfaces work together
- Standardizing interfaces across different implementations
- Creating reusable components from existing code
- API versioning and backwards compatibility

### When to Avoid

- When you control both interfaces (fix the interface instead)
- For simple wrappers (direct delegation may be clearer)
- When it adds unnecessary complexity
- If modification of the source is feasible and better

## Interview Questions

### Q1: What's the difference between Adapter and Facade patterns?

**Answer:**
- **Adapter**: Makes one interface compatible with another (one-to-one mapping)
- **Facade**: Simplifies a complex system with a unified interface (many-to-one)

### Q2: What's the difference between Adapter and Decorator?

**Answer:**
- **Adapter**: Changes interface to make incompatible classes work together
- **Decorator**: Adds new functionality while keeping the same interface

### Q3: Should an adapter add new functionality?

**Answer:** No. An adapter should only translate between interfaces. New functionality belongs in separate classes using composition or delegation.

### Q4: When should you use class adapter vs object adapter?

**Answer:**
- **Object adapter**: Uses composition (preferred in JavaScript/TypeScript)
- **Class adapter**: Uses multiple inheritance (not available in JS/TS)

### Q5: How do you test adapters effectively?

**Answer:** Test the interface translation separately from the adaptee. Mock the adaptee and verify the adapter correctly translates calls and data.

### Q6: Can you have bidirectional adapters?

**Answer:** Yes, a bidirectional adapter implements both interfaces and can convert in both directions (see Two-Way Adapter example).

### Q7: How does Adapter pattern relate to Interface Segregation Principle?

**Answer:** Adapters help apply ISP by allowing clients to depend on minimal interfaces while adapting to richer legacy interfaces.

### Q8: What's the performance impact of using adapters?

**Answer:** Minimal - adds one extra layer of indirection. The translation overhead is usually negligible compared to the adaptee's work.

## Key Takeaways

1. Adapter pattern enables collaboration between incompatible interfaces
2. Wraps existing class to match expected interface
3. Essential for integrating legacy code and third-party libraries
4. Should only translate, not add business logic
5. Use composition over inheritance (object adapter)
6. Encapsulate the adaptee as private implementation detail
7. Common in API wrappers and interface standardization
8. Different from Facade (which simplifies) and Decorator (which extends)
9. Promotes code reuse without modifying existing code
10. Critical for maintaining backwards compatibility

## Resources

- [Refactoring Guru: Adapter Pattern](https://refactoring.guru/design-patterns/adapter)
- [Gang of Four Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- [Martin Fowler: Gateway Pattern](https://martinfowler.com/eaaCatalog/gateway.html)
- [TypeScript Design Patterns](https://www.typescriptlang.org/docs/handbook/patterns.html)
- [MDN: Adapter Pattern](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Design_Patterns)
- [Source Making: Adapter Pattern](https://sourcemaking.com/design_patterns/adapter)
