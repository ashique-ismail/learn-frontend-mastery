# API Versioning

## Overview

API versioning is the practice of managing changes to APIs over time while maintaining backward compatibility for existing clients. As applications evolve, APIs need to change without breaking existing integrations. Proper versioning strategies ensure smooth transitions and maintain trust with API consumers.

## Why API Versioning Matters

```
Without Versioning:
Client A (v1) ──┐
Client B (v1) ──┼──→ API (Breaking Change) → ❌ All Clients Break
Client C (v1) ──┘

With Versioning:
Client A (v1) ──→ API v1 (Stable) → ✓ Works
Client B (v1) ──→ API v1 (Stable) → ✓ Works
Client C (v2) ──→ API v2 (New) ───→ ✓ Works
```

### Key Principles

1. **Backward Compatibility**: Old clients continue working
2. **Clear Communication**: Deprecation notices and migration guides
3. **Graceful Transitions**: Overlap period for migration
4. **Minimal Disruption**: Smooth upgrade path for consumers

## Versioning Strategies

### 1. URL Path Versioning

**Most common approach - version in the URL path.**

```typescript
// React API client with URL versioning
class APIClient {
  private baseURL: string;
  private version: string;

  constructor(version: string = 'v1') {
    this.baseURL = process.env.REACT_APP_API_URL || 'https://api.example.com';
    this.version = version;
  }

  private buildURL(endpoint: string): string {
    return `${this.baseURL}/${this.version}${endpoint}`;
  }

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(this.buildURL(endpoint), {
      headers: {
        'Authorization': `Bearer ${this.getToken()}`,
        'Content-Type': 'application/json'
      }
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return response.json();
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(this.buildURL(endpoint), {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.getToken()}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(data)
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return response.json();
  }

  private getToken(): string {
    return localStorage.getItem('authToken') || '';
  }
}

// Usage with different versions
const apiV1 = new APIClient('v1');
const apiV2 = new APIClient('v2');

// Old client using v1
const getUserV1 = async (id: string) => {
  // GET https://api.example.com/v1/users/123
  const user = await apiV1.get(`/users/${id}`);
  return user;
};

// New client using v2
const getUserV2 = async (id: string) => {
  // GET https://api.example.com/v2/users/123
  const user = await apiV2.get(`/users/${id}`);
  return user;
};
```

**Angular Service with URL Versioning:**

```typescript
// services/api-versioned.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class ApiVersionedService {
  private baseURL: string = environment.apiUrl;

  constructor(private http: HttpClient) {}

  private buildURL(version: string, endpoint: string): string {
    return `${this.baseURL}/${version}${endpoint}`;
  }

  private getHeaders(): HttpHeaders {
    return new HttpHeaders({
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${this.getToken()}`
    });
  }

  get<T>(version: string, endpoint: string): Observable<T> {
    return this.http.get<T>(
      this.buildURL(version, endpoint),
      { headers: this.getHeaders() }
    );
  }

  post<T>(version: string, endpoint: string, data: any): Observable<T> {
    return this.http.post<T>(
      this.buildURL(version, endpoint),
      data,
      { headers: this.getHeaders() }
    );
  }

  put<T>(version: string, endpoint: string, data: any): Observable<T> {
    return this.http.put<T>(
      this.buildURL(version, endpoint),
      data,
      { headers: this.getHeaders() }
    );
  }

  delete<T>(version: string, endpoint: string): Observable<T> {
    return this.http.delete<T>(
      this.buildURL(version, endpoint),
      { headers: this.getHeaders() }
    );
  }

  private getToken(): string {
    return localStorage.getItem('authToken') || '';
  }
}

// Feature service using versioned API
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private api: ApiVersionedService) {}

  getUserV1(id: string): Observable<UserV1> {
    return this.api.get<UserV1>('v1', `/users/${id}`);
  }

  getUserV2(id: string): Observable<UserV2> {
    return this.api.get<UserV2>('v2', `/users/${id}`);
  }

  createUser(user: CreateUserRequest): Observable<User> {
    // Always use latest version for new operations
    return this.api.post<User>('v2', '/users', user);
  }
}
```

**Pros and Cons:**

```
✅ Pros:
- Very clear and explicit
- Easy to route to different implementations
- Works with all HTTP clients
- Cache-friendly (different URLs)
- Simple to understand

❌ Cons:
- URLs change between versions
- Can't version individual resources
- Verbose
- Multiple URL patterns to maintain
```

### 2. Header Versioning

**Version specified in HTTP headers.**

```typescript
// React custom hook for header-based versioning
import { useState, useCallback } from 'react';

interface APIConfig {
  version?: string;
  customHeaders?: Record<string, string>;
}

export const useAPIRequest = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const request = useCallback(async <T,>(
    url: string,
    options: RequestInit = {},
    config: APIConfig = {}
  ): Promise<T> => {
    setLoading(true);
    setError(null);

    try {
      const version = config.version || 'v2'; // Default to latest
      
      const response = await fetch(url, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          'Accept': `application/vnd.api+json`,
          'API-Version': version, // Version in header
          ...config.customHeaders,
          ...options.headers
        }
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      // Check if server sent deprecation warning
      const deprecation = response.headers.get('Deprecation');
      const sunset = response.headers.get('Sunset');
      
      if (deprecation) {
        console.warn(
          `API version ${version} is deprecated. ` +
          `Sunset date: ${sunset || 'TBD'}`
        );
      }

      const data = await response.json();
      setLoading(false);
      return data;
    } catch (err) {
      const error = err as Error;
      setError(error);
      setLoading(false);
      throw error;
    }
  }, []);

  return { request, loading, error };
};

// Usage
const UserProfile = ({ userId }: { userId: string }) => {
  const { request, loading, error } = useAPIRequest();
  const [user, setUser] = useState<any>(null);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        // Use v1 API
        const userData = await request(
          `https://api.example.com/users/${userId}`,
          { method: 'GET' },
          { version: 'v1' }
        );
        setUser(userData);
      } catch (err) {
        console.error('Failed to fetch user:', err);
      }
    };

    fetchUser();
  }, [userId, request]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
};
```

**Angular HTTP Interceptor for Header Versioning:**

```typescript
// interceptors/api-version.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpResponse
} from '@angular/common/http';
import { Observable, tap } from 'rxjs';

@Injectable()
export class ApiVersionInterceptor implements HttpInterceptor {
  private readonly DEFAULT_VERSION = 'v2';

  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    // Check if request specifies a version
    const version = request.headers.get('API-Version') || this.DEFAULT_VERSION;

    // Clone request and add version header
    const versionedRequest = request.clone({
      setHeaders: {
        'API-Version': version,
        'Accept': 'application/vnd.api+json'
      }
    });

    return next.handle(versionedRequest).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          // Check for deprecation warnings
          const deprecation = event.headers.get('Deprecation');
          const sunset = event.headers.get('Sunset');

          if (deprecation) {
            console.warn(
              `[API Deprecation] Version ${version} is deprecated.`,
              sunset ? `Sunset: ${sunset}` : ''
            );

            // Could dispatch an action to show user notification
            // this.store.dispatch(showDeprecationWarning({ version, sunset }));
          }
        }
      })
    );
  }
}

// app.module.ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';

@NgModule({
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ApiVersionInterceptor,
      multi: true
    }
  ]
})
export class AppModule {}

// Usage in service
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUserV1(id: string): Observable<User> {
    return this.http.get<User>(
      `https://api.example.com/users/${id}`,
      {
        headers: { 'API-Version': 'v1' }
      }
    );
  }

  getUserV2(id: string): Observable<User> {
    // Uses default v2 from interceptor
    return this.http.get<User>(`https://api.example.com/users/${id}`);
  }
}
```

**Pros and Cons:**

```
✅ Pros:
- Clean URLs (no version in path)
- Can version per-request
- Flexible versioning strategies
- RESTful resource URLs stay constant

❌ Cons:
- Less visible (headers hidden from URL bar)
- Harder to test manually (need to set headers)
- Cache complexity (vary header needed)
- Not supported by all HTTP clients
```

### 3. Query Parameter Versioning

**Version as query parameter.**

```typescript
// React API client with query parameter versioning
interface RequestConfig {
  version?: string;
  params?: Record<string, string>;
}

class QueryParamAPIClient {
  private baseURL: string;

  constructor() {
    this.baseURL = process.env.REACT_APP_API_URL || 'https://api.example.com';
  }

  private buildURL(endpoint: string, config: RequestConfig = {}): string {
    const url = new URL(`${this.baseURL}${endpoint}`);
    
    // Add version as query parameter
    if (config.version) {
      url.searchParams.append('version', config.version);
    }

    // Add other query parameters
    if (config.params) {
      Object.entries(config.params).forEach(([key, value]) => {
        url.searchParams.append(key, value);
      });
    }

    return url.toString();
  }

  async get<T>(endpoint: string, config?: RequestConfig): Promise<T> {
    const url = this.buildURL(endpoint, config);
    const response = await fetch(url, {
      headers: {
        'Authorization': `Bearer ${this.getToken()}`,
        'Content-Type': 'application/json'
      }
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return response.json();
  }

  async post<T>(
    endpoint: string,
    data: any,
    config?: RequestConfig
  ): Promise<T> {
    const url = this.buildURL(endpoint, config);
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.getToken()}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(data)
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return response.json();
  }

  private getToken(): string {
    return localStorage.getItem('authToken') || '';
  }
}

// Usage
const api = new QueryParamAPIClient();

// GET https://api.example.com/users/123?version=v1
const userV1 = await api.get('/users/123', { version: 'v1' });

// GET https://api.example.com/users/123?version=v2&include=profile
const userV2 = await api.get('/users/123', {
  version: 'v2',
  params: { include: 'profile' }
});

// POST https://api.example.com/users?version=v2
const newUser = await api.post('/users', userData, { version: 'v2' });
```

**Pros and Cons:**

```
✅ Pros:
- Visible in URL
- Easy to test manually
- Can combine with other query parameters
- Flexible per-request versioning

❌ Cons:
- Pollutes query string
- Can conflict with other parameters
- Not RESTful
- Caching complications
```

### 4. Content Negotiation (Accept Header)

**Version through content type negotiation.**

```typescript
// React service with content negotiation
class ContentNegotiationAPI {
  private baseURL: string;

  constructor() {
    this.baseURL = process.env.REACT_APP_API_URL || 'https://api.example.com';
  }

  private getAcceptHeader(version: string): string {
    // Custom vendor media type
    return `application/vnd.myapp.${version}+json`;
  }

  async get<T>(endpoint: string, version: string = 'v2'): Promise<T> {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      headers: {
        'Authorization': `Bearer ${this.getToken()}`,
        'Accept': this.getAcceptHeader(version),
        'Content-Type': 'application/json'
      }
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    // Check actual content type returned
    const contentType = response.headers.get('Content-Type');
    console.log('Received content type:', contentType);

    return response.json();
  }

  async post<T>(
    endpoint: string,
    data: any,
    version: string = 'v2'
  ): Promise<T> {
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.getToken()}`,
        'Accept': this.getAcceptHeader(version),
        'Content-Type': this.getAcceptHeader(version)
      },
      body: JSON.stringify(data)
    });

    if (!response.ok) {
      throw new Error(`API Error: ${response.statusText}`);
    }

    return response.json();
  }

  private getToken(): string {
    return localStorage.getItem('authToken') || '';
  }
}

// Usage
const api = new ContentNegotiationAPI();

// Request with Accept: application/vnd.myapp.v1+json
const userV1 = await api.get('/users/123', 'v1');

// Request with Accept: application/vnd.myapp.v2+json
const userV2 = await api.get('/users/123', 'v2');
```

**Angular Content Negotiation Interceptor:**

```typescript
// interceptors/content-negotiation.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class ContentNegotiationInterceptor implements HttpInterceptor {
  private readonly DEFAULT_VERSION = 'v2';
  private readonly VENDOR_PREFIX = 'application/vnd.myapp';

  intercept(
    request: HttpRequest<unknown>,
    next: HttpHandler
  ): Observable<HttpEvent<unknown>> {
    // Extract version from context or use default
    const version = request.context.get('API_VERSION') || this.DEFAULT_VERSION;

    const modifiedRequest = request.clone({
      setHeaders: {
        'Accept': `${this.VENDOR_PREFIX}.${version}+json`,
        'Content-Type': 
          request.method !== 'GET' 
            ? `${this.VENDOR_PREFIX}.${version}+json`
            : 'application/json'
      }
    });

    return next.handle(modifiedRequest);
  }
}
```

**Pros and Cons:**

```
✅ Pros:
- RESTful and HTTP spec compliant
- Clean URLs
- Standard HTTP mechanism
- Professional approach

❌ Cons:
- Complex to implement
- Harder to test manually
- Not widely understood
- Requires custom media types
```

## Version Migration Strategies

### Gradual Migration with Feature Flags

```typescript
// React version migration manager
import { useState, useEffect } from 'react';

interface VersionConfig {
  defaultVersion: string;
  enabledVersions: string[];
  featureFlags: Record<string, string>; // feature -> version
}

class VersionMigrationManager {
  private config: VersionConfig;

  constructor(config: VersionConfig) {
    this.config = config;
  }

  // Get version for specific feature
  getVersionForFeature(feature: string): string {
    return this.config.featureFlags[feature] || this.config.defaultVersion;
  }

  // Check if version is enabled
  isVersionEnabled(version: string): boolean {
    return this.config.enabledVersions.includes(version);
  }

  // Gradual rollout percentage
  shouldUseNewVersion(rolloutPercentage: number): boolean {
    const userId = this.getUserId();
    const hash = this.hashString(userId);
    return (hash % 100) < rolloutPercentage;
  }

  private getUserId(): string {
    return localStorage.getItem('userId') || 'anonymous';
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32bit integer
    }
    return Math.abs(hash);
  }
}

// Usage in component
const UserProfile = ({ userId }: { userId: string }) => {
  const [user, setUser] = useState<any>(null);

  const versionManager = new VersionMigrationManager({
    defaultVersion: 'v1',
    enabledVersions: ['v1', 'v2'],
    featureFlags: {
      'user-profile': 'v2', // Use v2 for user profile
      'user-settings': 'v1'  // Still use v1 for settings
    }
  });

  useEffect(() => {
    const fetchUser = async () => {
      // Determine version based on feature and rollout
      let version = versionManager.getVersionForFeature('user-profile');
      
      // Gradual rollout: 30% of users get v2
      if (version === 'v1' && versionManager.shouldUseNewVersion(30)) {
        version = 'v2';
      }

      const api = new APIClient(version);
      const userData = await api.get(`/users/${userId}`);
      setUser(userData);
    };

    fetchUser();
  }, [userId]);

  return user ? <div>{user.name}</div> : <div>Loading...</div>;
};
```

### Adapter Pattern for Version Compatibility

```typescript
// Version adapters to normalize different API responses
interface UserV1 {
  id: string;
  full_name: string;
  email_address: string;
  created: string;
}

interface UserV2 {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  createdAt: string;
  profile: {
    avatar: string;
    bio: string;
  };
}

// Normalized internal format
interface User {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  createdAt: Date;
  avatar?: string;
  bio?: string;
}

// Adapters
class UserV1Adapter {
  static toUser(userV1: UserV1): User {
    const [firstName, ...lastNameParts] = userV1.full_name.split(' ');
    return {
      id: userV1.id,
      firstName,
      lastName: lastNameParts.join(' '),
      email: userV1.email_address,
      createdAt: new Date(userV1.created)
    };
  }
}

class UserV2Adapter {
  static toUser(userV2: UserV2): User {
    return {
      id: userV2.id,
      firstName: userV2.firstName,
      lastName: userV2.lastName,
      email: userV2.email,
      createdAt: new Date(userV2.createdAt),
      avatar: userV2.profile?.avatar,
      bio: userV2.profile?.bio
    };
  }
}

// Service that handles both versions
class UserService {
  private apiV1 = new APIClient('v1');
  private apiV2 = new APIClient('v2');

  async getUser(id: string, version: 'v1' | 'v2' = 'v2'): Promise<User> {
    if (version === 'v1') {
      const userV1 = await this.apiV1.get<UserV1>(`/users/${id}`);
      return UserV1Adapter.toUser(userV1);
    } else {
      const userV2 = await this.apiV2.get<UserV2>(`/users/${id}`);
      return UserV2Adapter.toUser(userV2);
    }
  }

  // Automatically try v2, fallback to v1
  async getUserWithFallback(id: string): Promise<User> {
    try {
      return await this.getUser(id, 'v2');
    } catch (error) {
      console.warn('V2 API failed, falling back to V1', error);
      return await this.getUser(id, 'v1');
    }
  }
}
```

## Deprecation Strategy

### Deprecation Headers and Warnings

```typescript
// React hook to handle API deprecation warnings
import { useEffect } from 'react';

interface DeprecationInfo {
  version: string;
  deprecationDate?: string;
  sunsetDate?: string;
  migrationGuide?: string;
}

export const useDeprecationWarnings = () => {
  const [warnings, setWarnings] = useState<DeprecationInfo[]>([]);

  // Intercept fetch to check deprecation headers
  useEffect(() => {
    const originalFetch = window.fetch;

    window.fetch = async (...args) => {
      const response = await originalFetch(...args);

      // Check for deprecation headers
      const deprecation = response.headers.get('Deprecation');
      const sunset = response.headers.get('Sunset');
      const link = response.headers.get('Link');

      if (deprecation) {
        const warning: DeprecationInfo = {
          version: extractVersionFromURL(args[0] as string),
          deprecationDate: deprecation,
          sunsetDate: sunset || undefined,
          migrationGuide: link || undefined
        };

        setWarnings(prev => [...prev, warning]);

        // Log warning
        console.warn(
          `[API Deprecation] ${warning.version} is deprecated`,
          `Deprecation: ${deprecation}`,
          sunset && `Sunset: ${sunset}`,
          link && `Migration guide: ${link}`
        );

        // Could also send to analytics
        // analytics.track('api_deprecation_warning', warning);
      }

      return response;
    };

    return () => {
      window.fetch = originalFetch;
    };
  }, []);

  return { warnings };
};

const extractVersionFromURL = (url: string): string => {
  const match = url.match(/\/(v\d+)\//);
  return match ? match[1] : 'unknown';
};

// Component to display deprecation warnings
const DeprecationNotice = () => {
  const { warnings } = useDeprecationWarnings();

  if (warnings.length === 0) return null;

  return (
    <div className="deprecation-notice">
      <h3>API Version Warnings</h3>
      {warnings.map((warning, index) => (
        <div key={index} className="warning">
          <p>
            <strong>{warning.version}</strong> is deprecated
          </p>
          {warning.sunsetDate && (
            <p>Will be sunset on: {warning.sunsetDate}</p>
          )}
          {warning.migrationGuide && (
            <a href={warning.migrationGuide}>Migration Guide</a>
          )}
        </div>
      ))}
    </div>
  );
};
```

### Sunset Implementation

```typescript
// Phased deprecation timeline
enum DeprecationPhase {
  ACTIVE = 'active',           // Fully supported
  DEPRECATED = 'deprecated',    // Supported but discouraged
  LEGACY = 'legacy',           // Limited support
  SUNSET = 'sunset'            // No longer available
}

interface VersionLifecycle {
  version: string;
  phase: DeprecationPhase;
  releaseDate: Date;
  deprecationDate?: Date;
  sunsetDate?: Date;
  migrationPath?: string;
}

class VersionLifecycleManager {
  private versions: Map<string, VersionLifecycle> = new Map();

  registerVersion(lifecycle: VersionLifecycle): void {
    this.versions.set(lifecycle.version, lifecycle);
  }

  getPhase(version: string): DeprecationPhase {
    const lifecycle = this.versions.get(version);
    if (!lifecycle) return DeprecationPhase.SUNSET;

    const now = new Date();

    if (lifecycle.sunsetDate && now >= lifecycle.sunsetDate) {
      return DeprecationPhase.SUNSET;
    }

    if (lifecycle.deprecationDate && now >= lifecycle.deprecationDate) {
      const monthsUntilSunset = lifecycle.sunsetDate
        ? this.monthsBetween(now, lifecycle.sunsetDate)
        : Infinity;

      return monthsUntilSunset < 3
        ? DeprecationPhase.LEGACY
        : DeprecationPhase.DEPRECATED;
    }

    return DeprecationPhase.ACTIVE;
  }

  canUseVersion(version: string): boolean {
    const phase = this.getPhase(version);
    return phase !== DeprecationPhase.SUNSET;
  }

  getDeprecationInfo(version: string): VersionLifecycle | undefined {
    return this.versions.get(version);
  }

  private monthsBetween(date1: Date, date2: Date): number {
    return (
      (date2.getFullYear() - date1.getFullYear()) * 12 +
      (date2.getMonth() - date1.getMonth())
    );
  }
}

// Usage
const lifecycleManager = new VersionLifecycleManager();

lifecycleManager.registerVersion({
  version: 'v1',
  phase: DeprecationPhase.DEPRECATED,
  releaseDate: new Date('2022-01-01'),
  deprecationDate: new Date('2024-01-01'),
  sunsetDate: new Date('2025-01-01'),
  migrationPath: '/docs/migration/v1-to-v2'
});

lifecycleManager.registerVersion({
  version: 'v2',
  phase: DeprecationPhase.ACTIVE,
  releaseDate: new Date('2024-01-01')
});

// Check before making API call
const makeAPICall = async (version: string, endpoint: string) => {
  if (!lifecycleManager.canUseVersion(version)) {
    throw new Error(`API version ${version} is no longer available`);
  }

  const phase = lifecycleManager.getPhase(version);
  if (phase === DeprecationPhase.DEPRECATED || phase === DeprecationPhase.LEGACY) {
    const info = lifecycleManager.getDeprecationInfo(version);
    console.warn(
      `Warning: ${version} is ${phase}.`,
      info?.sunsetDate && `Sunset date: ${info.sunsetDate}`,
      info?.migrationPath && `Migrate: ${info.migrationPath}`
    );
  }

  // Proceed with API call
  const api = new APIClient(version);
  return await api.get(endpoint);
};
```

## Version Discovery and Documentation

### API Version Metadata Endpoint

```typescript
// React component to fetch and display available API versions
interface APIVersion {
  version: string;
  status: 'active' | 'deprecated' | 'sunset';
  releaseDate: string;
  deprecationDate?: string;
  sunsetDate?: string;
  documentation: string;
  changes: string[];
}

interface APIMetadata {
  currentVersion: string;
  versions: APIVersion[];
}

const APIVersionInfo = () => {
  const [metadata, setMetadata] = useState<APIMetadata | null>(null);

  useEffect(() => {
    const fetchMetadata = async () => {
      const response = await fetch('https://api.example.com/versions');
      const data = await response.json();
      setMetadata(data);
    };

    fetchMetadata();
  }, []);

  if (!metadata) return <div>Loading API information...</div>;

  return (
    <div className="api-version-info">
      <h2>API Versions</h2>
      <p>Current Version: <strong>{metadata.currentVersion}</strong></p>

      <div className="versions-list">
        {metadata.versions.map(version => (
          <div 
            key={version.version} 
            className={`version-card ${version.status}`}
          >
            <h3>{version.version}</h3>
            <span className="status-badge">{version.status}</span>

            <p>Released: {version.releaseDate}</p>
            
            {version.deprecationDate && (
              <p className="warning">
                Deprecated: {version.deprecationDate}
              </p>
            )}
            
            {version.sunsetDate && (
              <p className="error">
                Sunset: {version.sunsetDate}
              </p>
            )}

            <a href={version.documentation}>Documentation</a>

            {version.changes.length > 0 && (
              <details>
                <summary>Changes in this version</summary>
                <ul>
                  {version.changes.map((change, idx) => (
                    <li key={idx}>{change}</li>
                  ))}
                </ul>
              </details>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};
```

## Common Mistakes

### 1. Breaking Changes Without Version Bump

```typescript
// ❌ BAD: Breaking change in same version
// v1 originally returned:
interface UserV1Original {
  id: string;
  name: string;
}

// v1 later changed to (BREAKING!):
interface UserV1Breaking {
  id: number; // Changed from string to number
  fullName: string; // Changed property name
}

// ✅ GOOD: Breaking changes require new version
interface UserV1 {
  id: string;
  name: string;
}

interface UserV2 {
  id: number; // Breaking change in new version
  fullName: string;
}
```

### 2. Not Providing Migration Path

```typescript
// ❌ BAD: Just deprecate without guidance
console.warn('v1 is deprecated');

// ✅ GOOD: Clear migration path
const deprecationMessage = {
  version: 'v1',
  status: 'deprecated',
  sunset: '2025-12-31',
  message: 'Please migrate to v2',
  migrationGuide: 'https://docs.example.com/migration/v1-to-v2',
  breakingChanges: [
    'User.name changed to User.fullName',
    'User.id is now number instead of string',
    'New required field: User.email'
  ],
  codeExample: `
    // Before (v1)
    const user = await api.get('/v1/users/123');
    console.log(user.name);

    // After (v2)
    const user = await api.get('/v2/users/123');
    console.log(user.fullName);
  `
};
```

### 3. Too Many Active Versions

```typescript
// ❌ BAD: Supporting too many versions
const api = {
  v1: createAPIClient('v1'),
  v2: createAPIClient('v2'),
  v3: createAPIClient('v3'),
  v4: createAPIClient('v4'),
  v5: createAPIClient('v5'), // Too many!
};

// ✅ GOOD: Limit active versions (N-1 or N-2 pattern)
const api = {
  v4: createAPIClient('v4'), // Previous version (deprecated)
  v5: createAPIClient('v5'), // Current version (active)
};

// Sunset older versions with clear timeline
const sunsetSchedule = {
  v1: 'sunset 2023-01-01',
  v2: 'sunset 2024-01-01',
  v3: 'sunset 2024-12-31',
  v4: 'deprecated, sunset 2025-12-31',
  v5: 'active'
};
```

## Best Practices

### 1. Semantic Versioning for APIs

```
Major.Minor.Patch
v2.1.3

Major: Breaking changes (v1 → v2)
Minor: New features, backward compatible (v2.0 → v2.1)
Patch: Bug fixes, no API changes (v2.1.0 → v2.1.1)
```

### 2. Deprecation Timeline

```
┌──────────────────────────────────────────────────────────────┐
│                    Version Lifecycle                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Release        Deprecation      Sunset                       │
│    │                │              │                          │
│    v                v              v                          │
│  ┌───┐          ┌───────┐      ┌─────┐                      │
│  │ 1 │─────────→│   2   │─────→│  3  │                      │
│  └───┘  6 mo    └───────┘ 6 mo └─────┘                      │
│                                                               │
│  Phase 1: Active - Full support                               │
│  Phase 2: Deprecated - Supported but discouraged              │
│  Phase 3: Sunset - No longer available                        │
│                                                               │
│  Recommended: 6-12 months deprecation period                  │
└──────────────────────────────────────────────────────────────┘
```

### 3. Version Response Format

```typescript
// Always include version info in response
interface APIResponse<T> {
  data: T;
  meta: {
    version: string;
    timestamp: string;
    deprecation?: {
      deprecated: boolean;
      sunsetDate?: string;
      migrationGuide?: string;
    };
  };
}
```

### 4. Client Version Checking

```typescript
// Ensure client is using compatible version
const checkAPICompatibility = async () => {
  const clientVersion = '2.1.0';
  
  const response = await fetch('https://api.example.com/version');
  const serverInfo = await response.json();
  
  const clientMajor = parseInt(clientVersion.split('.')[0]);
  const serverMajor = parseInt(serverInfo.version.split('.')[0]);
  
  if (clientMajor < serverMajor) {
    console.warn('Your app version is outdated. Please update.');
  } else if (clientMajor > serverMajor) {
    console.error('Your app version is too new for this API.');
  }
};
```

## When to Version

### Breaking Changes (Require New Version)

- Removing fields or endpoints
- Changing field types
- Changing field names
- Changing response structure
- Changing authentication mechanism
- Removing or changing required fields
- Changing URL structure

### Non-Breaking Changes (Same Version)

- Adding optional fields
- Adding new endpoints
- Adding optional query parameters
- Improving error messages
- Performance improvements
- Bug fixes

## Interview Questions

### 1. What are the main API versioning strategies?
**Answer**: URL path versioning (v1/users), header versioning (API-Version header), query parameter versioning (?version=v1), and content negotiation (Accept header). URL path versioning is most common due to its simplicity and visibility, while content negotiation is most RESTful but harder to implement.

### 2. How do you handle API deprecation?
**Answer**: Use a phased approach: (1) Announce deprecation with timeline, (2) Add deprecation headers to responses, (3) Provide migration guide and code examples, (4) Maintain deprecated version for 6-12 months, (5) Send reminders as sunset approaches, (6) Finally sunset the old version. Always provide a clear migration path.

### 3. What's the difference between versioning the entire API vs. individual resources?
**Answer**: Versioning the entire API (v1/users, v1/posts) keeps all resources in sync but requires maintaining duplicate implementations. Versioning individual resources (users/v1, users/v2) provides finer control but creates complexity in managing cross-resource dependencies and client implementations.

### 4. How do you maintain backward compatibility?
**Answer**: Add new fields as optional, never remove or rename existing fields, use additive changes only, provide default values for new required fields, version breaking changes separately, and use adapters/transformers to normalize different versions internally.

### 5. What headers should you include for API versioning?
**Answer**: `API-Version` or `Accept` for specifying version, `Deprecation` header with RFC 3339 date, `Sunset` header with removal date, `Link` header pointing to migration guide, and custom `X-API-Version` header for additional metadata. These inform clients about version status and migration needs.

## Key Takeaways

1. **Version Early**: Plan versioning strategy from the start, even for v1.
2. **Communicate Clearly**: Always provide deprecation notices and migration guides.
3. **Limit Active Versions**: Support only current and previous version (N and N-1).
4. **Use Semantic Versioning**: Major for breaking changes, minor for features, patch for fixes.
5. **Defense in Depth**: Frontend checks for UX, backend enforcement for security.
6. **Graceful Deprecation**: Give clients 6-12 months to migrate before sunset.
7. **Document Everything**: Maintain clear changelog and migration documentation.
8. **Monitor Usage**: Track version usage to inform deprecation decisions.

## Resources

- [REST API Versioning Best Practices](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/)
- [Semantic Versioning](https://semver.org/)
- [RFC 8594 - The Sunset HTTP Header](https://www.rfc-editor.org/rfc/rfc8594.html)
- [Microsoft REST API Guidelines - Versioning](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
- [Stripe API Versioning](https://stripe.com/docs/api/versioning)
- [GitHub API Versioning](https://docs.github.com/en/rest/overview/api-versions)
