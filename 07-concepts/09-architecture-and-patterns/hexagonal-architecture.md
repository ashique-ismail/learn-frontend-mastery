# Hexagonal Architecture (Ports & Adapters)

## Table of Contents
1. [Introduction & Origin](#introduction--origin)
2. [The Hexagon Metaphor](#the-hexagon-metaphor)
3. [The Domain Core](#the-domain-core)
4. [Ports: Inbound and Outbound](#ports-inbound-and-outbound)
5. [Adapters: Primary and Secondary](#adapters-primary-and-secondary)
6. [Mapping to a Frontend Application](#mapping-to-a-frontend-application)
7. [Folder Structure](#folder-structure)
8. [Full TypeScript Implementation](#full-typescript-implementation)
9. [Testability: Swapping Adapters](#testability-swapping-adapters)
10. [Comparison with Clean Architecture and Layered Architecture](#comparison-with-clean-architecture-and-layered-architecture)
11. [Trade-offs](#trade-offs)
12. [When to Use](#when-to-use)
13. [Interview Q&A](#interview-qa)
14. [Key Takeaways](#key-takeaways)

---

## Introduction & Origin

Hexagonal Architecture was introduced by **Alistair Cockburn in 2005** in his article *"Hexagonal Architecture"* (also titled *"Ports and Adapters"*). The motivation was straightforward: application logic was routinely entangled with UI code, database drivers, and third-party libraries. Swapping a database or running business logic in isolation for testing required heroic effort.

Cockburn's insight: the application should be equally driveable by any external actor — a user through a GUI, an automated test harness, a CLI, a message queue consumer — and should be equally capable of talking to any external system — an HTTP API, an in-memory stub, a file system, a real database — **without changing any business logic**.

The pattern achieves this through a single structural rule:

> **The application core communicates with the outside world only through explicitly defined interfaces (Ports). All concrete connections are Adapters that plug into those ports.**

This is why it is formally named **Ports & Adapters**, with "Hexagonal" being the visual representation.

---

## The Hexagon Metaphor

The hexagon is not about six layers or six sides having specific meaning. Cockburn chose a hexagon simply because it is easy to draw multiple connection points (ports) around the perimeter, and it avoids the "top/bottom" connotation of layered diagrams.

```
                    ┌───────────────────────────────────────┐
                    │                                       │
   [Test Harness]───┤  PORT: IAuthService (inbound)         │
                    │                                       │
   [React UI]───────┤  PORT: IAuthService (inbound)         │
                    │                                       │
                    │    ┌─────────────────────────────┐    │
                    │    │                             │    │
                    │    │   APPLICATION CORE          │    │
                    │    │   (Domain Logic / Use Cases)│    │
                    │    │                             │    │
                    │    └─────────────────────────────┘    │
                    │                                       │
   [HTTP API]───────┤  PORT: IUserRepository (outbound)     │
                    │                                       │
   [localStorage]───┤  PORT: ITokenStorage (outbound)       │
                    │                                       │
                    └───────────────────────────────────────┘
```

**Left side (primary/driving side):** Actors that initiate interaction with the app. They drive the application through inbound ports.

**Right side (secondary/driven side):** Systems the app talks to. The app drives them through outbound ports. It controls them.

The core does not know whether a port is connected to a real HTTP client or an in-memory mock. It cannot know — it only sees the interface.

---

## The Domain Core

The domain core is the heart of the architecture. It has exactly two properties:

1. **It contains all business rules.** Validation, computation, domain events, state transitions — anything that would be true regardless of which UI framework or database technology you are using.

2. **It has zero dependencies on infrastructure or frameworks.** No `import axios from 'axios'`, no `import { useState } from 'react'`, no `import { Injectable } from '@angular/core'`. The core is plain TypeScript classes, interfaces, and functions.

### What belongs in the core

```typescript
// Domain Entity — plain TypeScript class, no framework imports
export class User {
  private constructor(
    public readonly id: string,
    public readonly email: Email,          // Value Object
    public readonly role: UserRole,
    private _isActive: boolean
  ) {}

  static create(id: string, email: string, role: UserRole): User {
    const emailVO = Email.create(email);   // throws on invalid email
    return new User(id, emailVO, role, true);
  }

  deactivate(): User {
    if (!this._isActive) throw new DomainError('User already inactive');
    return new User(this.id, this.email, this.role, false);
  }

  get isActive(): boolean { return this._isActive; }
}

// Value Object — enforces invariants at construction time
export class Email {
  private constructor(public readonly value: string) {}

  static create(raw: string): Email {
    if (!raw.includes('@')) throw new DomainError(`Invalid email: ${raw}`);
    return new Email(raw.toLowerCase().trim());
  }
}
```

### What does NOT belong in the core

| Forbidden | Reason |
|-----------|--------|
| `axios`, `fetch` | Infrastructure concern |
| `localStorage`, `sessionStorage` | Infrastructure concern |
| `React`, `Angular`, `Vue` | UI framework — primary adapter territory |
| `@Injectable()`, `@Component()` | Framework annotations |
| `JWT decode library` | Infrastructure detail — the core works with plain token strings at most |
| `console.log` | Acceptable in dev but logging infrastructure (Sentry, etc.) belongs outside |

---

## Ports: Inbound and Outbound

A **port** is a TypeScript interface. Nothing more. It is the contract between the core and the outside world.

### Inbound Ports (Driving Ports)

These define **what the application can do** — its use cases. External actors (UI components, test harnesses, CLI commands) call these to drive the application.

```typescript
// src/auth/domain/ports/inbound/IAuthUseCase.ts

export interface LoginInput {
  email: string;
  password: string;
}

export interface LoginOutput {
  userId: string;
  accessToken: string;
  expiresAt: Date;
}

// This is the inbound port — the UI will call this interface
export interface IAuthUseCase {
  login(input: LoginInput): Promise<LoginOutput>;
  logout(): Promise<void>;
  refreshToken(): Promise<LoginOutput>;
  getCurrentUser(): Promise<User | null>;
}
```

The React component or Angular service **holds a reference to this interface**, not to any concrete class. The component calls `authUseCase.login(...)`. It never imports an HTTP client.

### Outbound Ports (Driven Ports)

These define **what the application needs** from the outside world. The core declares these interfaces; adapters in the outer layer implement them.

```typescript
// src/auth/domain/ports/outbound/IUserRepository.ts
export interface IUserRepository {
  findByEmail(email: string): Promise<User | null>;
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// src/auth/domain/ports/outbound/ITokenStorage.ts
export interface ITokenStorage {
  saveToken(token: string, expiresAt: Date): void;
  getToken(): string | null;
  clearToken(): void;
  isTokenExpired(): boolean;
}

// src/auth/domain/ports/outbound/IPasswordHasher.ts
export interface IPasswordHasher {
  hash(plain: string): Promise<string>;
  compare(plain: string, hashed: string): Promise<boolean>;
}
```

The domain core imports these interfaces at the top level and uses them via constructor injection. The core never instantiates a concrete class.

---

## Adapters: Primary and Secondary

An **adapter** is a concrete class that implements a port interface and translates between the outside world's format and the core's format.

### Primary Adapters (Driving Adapters)

Primary adapters sit on the left side. They receive external input and translate it into calls on an inbound port.

**React component as a primary adapter:**

```typescript
// src/auth/adapters/primary/react/LoginForm.tsx
// This component IS the primary adapter. It drives the core.

import React, { useState } from 'react';
import { IAuthUseCase } from '../../../domain/ports/inbound/IAuthUseCase';

interface Props {
  authUseCase: IAuthUseCase;  // depends on the PORT, not a concrete class
}

export const LoginForm: React.FC<Props> = ({ authUseCase }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);
    try {
      await authUseCase.login({ email, password });
      // navigate to dashboard, etc.
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={e => setEmail(e.target.value)} type="email" />
      <input value={password} onChange={e => setPassword(e.target.value)} type="password" />
      {error && <p>{error}</p>}
      <button type="submit" disabled={loading}>Log in</button>
    </form>
  );
};
```

Notice: the component imports `IAuthUseCase` (the port interface), not `AuthService` or `axios`. If you swap React for Vue tomorrow, this file disappears. The domain core and all outbound adapters are completely unaffected.

**Angular component as a primary adapter:**

```typescript
// src/auth/adapters/primary/angular/login.component.ts
@Component({
  selector: 'app-login',
  templateUrl: './login.component.html'
})
export class LoginComponent {
  // IAuthUseCase injected via DI token — still depends on the interface, not concrete
  constructor(@Inject(AUTH_USE_CASE_TOKEN) private authUseCase: IAuthUseCase) {}

  async onSubmit(email: string, password: string): Promise<void> {
    await this.authUseCase.login({ email, password });
  }
}
```

### Secondary Adapters (Driven Adapters)

Secondary adapters sit on the right side. They implement outbound port interfaces and talk to real infrastructure.

**HTTP adapter implementing `IUserRepository`:**

```typescript
// src/auth/adapters/secondary/http/HttpUserRepository.ts
import axios from 'axios';
import { IUserRepository } from '../../../domain/ports/outbound/IUserRepository';
import { User } from '../../../domain/entities/User';

export class HttpUserRepository implements IUserRepository {
  constructor(private readonly baseUrl: string) {}

  async findByEmail(email: string): Promise<User | null> {
    try {
      const response = await axios.get<UserApiResponse>(
        `${this.baseUrl}/users?email=${encodeURIComponent(email)}`
      );
      return this.toDomain(response.data);
    } catch (err) {
      if (axios.isAxiosError(err) && err.response?.status === 404) return null;
      throw err;
    }
  }

  async findById(id: string): Promise<User | null> {
    try {
      const response = await axios.get<UserApiResponse>(`${this.baseUrl}/users/${id}`);
      return this.toDomain(response.data);
    } catch (err) {
      if (axios.isAxiosError(err) && err.response?.status === 404) return null;
      throw err;
    }
  }

  async save(user: User): Promise<void> {
    await axios.put(`${this.baseUrl}/users/${user.id}`, this.toApiDto(user));
  }

  private toDomain(dto: UserApiResponse): User {
    return User.create(dto.id, dto.email, dto.role as UserRole);
  }

  private toApiDto(user: User): UserApiRequest {
    return { email: user.email.value, role: user.role, isActive: user.isActive };
  }
}
```

**localStorage adapter implementing `ITokenStorage`:**

```typescript
// src/auth/adapters/secondary/storage/LocalStorageTokenStorage.ts
import { ITokenStorage } from '../../../domain/ports/outbound/ITokenStorage';

const TOKEN_KEY = 'auth_token';
const EXPIRES_KEY = 'auth_token_expires';

export class LocalStorageTokenStorage implements ITokenStorage {
  saveToken(token: string, expiresAt: Date): void {
    localStorage.setItem(TOKEN_KEY, token);
    localStorage.setItem(EXPIRES_KEY, expiresAt.toISOString());
  }

  getToken(): string | null {
    return localStorage.getItem(TOKEN_KEY);
  }

  clearToken(): void {
    localStorage.removeItem(TOKEN_KEY);
    localStorage.removeItem(EXPIRES_KEY);
  }

  isTokenExpired(): boolean {
    const expiry = localStorage.getItem(EXPIRES_KEY);
    if (!expiry) return true;
    return new Date(expiry) <= new Date();
  }
}
```

---

## Mapping to a Frontend Application

The abstraction maps cleanly onto frontend layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRIMARY ADAPTERS (driving side)                                    │
│  React / Angular / Vue components, CLI scripts, test runners        │
│  They call inbound ports → translate user gestures to use cases     │
├─────────────────────────────────────────────────────────────────────┤
│  INBOUND PORTS                                                      │
│  TypeScript interfaces: IAuthUseCase, IUserUseCase, ICartUseCase    │
├─────────────────────────────────────────────────────────────────────┤
│  APPLICATION CORE                                                   │
│  Use Case implementations: AuthService, UserService, CartService    │
│  Domain entities: User, Order, Cart, Product                        │
│  Domain services: PriceCalculator, DiscountEngine                   │
│  Value objects: Email, Money, Quantity                              │
├─────────────────────────────────────────────────────────────────────┤
│  OUTBOUND PORTS                                                     │
│  TypeScript interfaces: IUserRepository, ITokenStorage, ILogger     │
├─────────────────────────────────────────────────────────────────────┤
│  SECONDARY ADAPTERS (driven side)                                   │
│  HttpUserRepository (axios/fetch), LocalStorageTokenStorage,        │
│  SentryLogger, InMemoryUserRepository (for tests)                   │
└─────────────────────────────────────────────────────────────────────┘
```

The dependency rule enforces that:
- Primary adapters depend on inbound ports (not core classes directly)
- The core depends on outbound port interfaces (not adapter classes)
- Secondary adapters depend on outbound ports (they implement them)
- Nothing in the core imports from adapters

---

## Folder Structure

A feature-based folder structure for a React authentication feature:

```
src/
└── auth/
    ├── domain/
    │   ├── entities/
    │   │   ├── User.ts                    # Domain entity
    │   │   └── Session.ts                 # Domain entity
    │   ├── value-objects/
    │   │   ├── Email.ts                   # Value object with validation
    │   │   └── Password.ts                # Value object (never stores plaintext)
    │   ├── errors/
    │   │   ├── InvalidCredentialsError.ts # Domain-specific errors
    │   │   └── SessionExpiredError.ts
    │   └── ports/
    │       ├── inbound/
    │       │   └── IAuthUseCase.ts        # What the UI can call
    │       └── outbound/
    │           ├── IUserRepository.ts     # What the core needs from storage
    │           ├── ITokenStorage.ts       # Token persistence interface
    │           └── IPasswordHasher.ts     # Hashing interface
    │
    ├── application/
    │   └── AuthUseCase.ts                 # Implements IAuthUseCase, uses outbound ports
    │
    └── adapters/
        ├── primary/
        │   ├── react/
        │   │   ├── LoginForm.tsx          # React component (primary adapter)
        │   │   ├── useAuth.ts             # Custom hook wiring IAuthUseCase
        │   │   └── AuthContext.tsx        # Provides IAuthUseCase via React context
        │   └── angular/
        │       └── login.component.ts     # Angular component (primary adapter)
        │
        └── secondary/
            ├── http/
            │   └── HttpUserRepository.ts  # Calls REST API, implements IUserRepository
            ├── storage/
            │   ├── LocalStorageTokenStorage.ts  # Real localStorage adapter
            │   └── InMemoryTokenStorage.ts      # For tests/SSR
            ├── crypto/
            │   └── BcryptPasswordHasher.ts      # bcrypt adapter
            └── mock/
                ├── InMemoryUserRepository.ts    # Test double
                └── MockPasswordHasher.ts        # Test double
```

The key invariant: **nothing inside `domain/` imports from `adapters/`**. The `application/` layer imports from `domain/` only. The `adapters/` layer imports from `domain/` and `application/`, but those layers never import back.

---

## Full TypeScript Implementation

A complete walkthrough of the authentication feature.

### Step 1 — Domain entities and value objects

```typescript
// src/auth/domain/value-objects/Email.ts
export class Email {
  private constructor(public readonly value: string) {}

  static create(raw: string): Email {
    const trimmed = raw.trim().toLowerCase();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(trimmed)) {
      throw new Error(`Invalid email address: ${raw}`);
    }
    return new Email(trimmed);
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }

  toString(): string { return this.value; }
}
```

```typescript
// src/auth/domain/entities/User.ts
import { Email } from '../value-objects/Email';

export type UserRole = 'admin' | 'member' | 'guest';

export class User {
  private constructor(
    public readonly id: string,
    public readonly email: Email,
    public readonly role: UserRole,
    public readonly passwordHash: string,
    public readonly isActive: boolean
  ) {}

  static reconstitute(params: {
    id: string;
    email: string;
    role: UserRole;
    passwordHash: string;
    isActive: boolean;
  }): User {
    return new User(
      params.id,
      Email.create(params.email),
      params.role,
      params.passwordHash,
      params.isActive
    );
  }
}
```

### Step 2 — Outbound ports

```typescript
// src/auth/domain/ports/outbound/IUserRepository.ts
import { User } from '../../entities/User';

export interface IUserRepository {
  findByEmail(email: string): Promise<User | null>;
  findById(id: string): Promise<User | null>;
}

// src/auth/domain/ports/outbound/ITokenStorage.ts
export interface ITokenStorage {
  saveToken(token: string, expiresAt: Date): void;
  getToken(): string | null;
  clearToken(): void;
  isTokenExpired(): boolean;
}

// src/auth/domain/ports/outbound/IPasswordHasher.ts
export interface IPasswordHasher {
  compare(plain: string, hashed: string): Promise<boolean>;
}
```

### Step 3 — Inbound port

```typescript
// src/auth/domain/ports/inbound/IAuthUseCase.ts
import { User } from '../../entities/User';

export interface LoginInput { email: string; password: string; }
export interface LoginOutput { userId: string; accessToken: string; expiresAt: Date; }

export interface IAuthUseCase {
  login(input: LoginInput): Promise<LoginOutput>;
  logout(): Promise<void>;
  getCurrentUser(): Promise<User | null>;
}
```

### Step 4 — Application use case (core implementation)

```typescript
// src/auth/application/AuthUseCase.ts
import { IAuthUseCase, LoginInput, LoginOutput } from '../domain/ports/inbound/IAuthUseCase';
import { IUserRepository } from '../domain/ports/outbound/IUserRepository';
import { ITokenStorage } from '../domain/ports/outbound/ITokenStorage';
import { IPasswordHasher } from '../domain/ports/outbound/IPasswordHasher';
import { User } from '../domain/entities/User';

// Zero imports from 'react', 'axios', 'localStorage' — only domain interfaces
export class AuthUseCase implements IAuthUseCase {
  constructor(
    private readonly userRepository: IUserRepository,
    private readonly tokenStorage: ITokenStorage,
    private readonly passwordHasher: IPasswordHasher
  ) {}

  async login(input: LoginInput): Promise<LoginOutput> {
    const user = await this.userRepository.findByEmail(input.email);
    if (!user) throw new Error('Invalid credentials');
    if (!user.isActive) throw new Error('Account is deactivated');

    const isValid = await this.passwordHasher.compare(input.password, user.passwordHash);
    if (!isValid) throw new Error('Invalid credentials');

    // In a real app, call an auth server here via another outbound port (IAuthTokenService)
    const fakeToken = `token-${user.id}-${Date.now()}`;
    const expiresAt = new Date(Date.now() + 3600 * 1000);

    this.tokenStorage.saveToken(fakeToken, expiresAt);

    return { userId: user.id, accessToken: fakeToken, expiresAt };
  }

  async logout(): Promise<void> {
    this.tokenStorage.clearToken();
  }

  async getCurrentUser(): Promise<User | null> {
    const token = this.tokenStorage.getToken();
    if (!token || this.tokenStorage.isTokenExpired()) return null;

    // Parse userId from token (simplified)
    const userId = token.split('-')[1];
    return this.userRepository.findById(userId);
  }
}
```

### Step 5 — Secondary adapters (real infrastructure)

```typescript
// src/auth/adapters/secondary/http/HttpUserRepository.ts
import { IUserRepository } from '../../../domain/ports/outbound/IUserRepository';
import { User } from '../../../domain/entities/User';

export class HttpUserRepository implements IUserRepository {
  constructor(private readonly baseUrl: string) {}

  async findByEmail(email: string): Promise<User | null> {
    const res = await fetch(`${this.baseUrl}/users?email=${encodeURIComponent(email)}`);
    if (res.status === 404) return null;
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const dto = await res.json();
    return User.reconstitute(dto);
  }

  async findById(id: string): Promise<User | null> {
    const res = await fetch(`${this.baseUrl}/users/${id}`);
    if (res.status === 404) return null;
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const dto = await res.json();
    return User.reconstitute(dto);
  }
}
```

```typescript
// src/auth/adapters/secondary/storage/LocalStorageTokenStorage.ts
import { ITokenStorage } from '../../../domain/ports/outbound/ITokenStorage';

export class LocalStorageTokenStorage implements ITokenStorage {
  private static readonly TOKEN_KEY = 'auth:token';
  private static readonly EXPIRES_KEY = 'auth:expires';

  saveToken(token: string, expiresAt: Date): void {
    localStorage.setItem(LocalStorageTokenStorage.TOKEN_KEY, token);
    localStorage.setItem(LocalStorageTokenStorage.EXPIRES_KEY, expiresAt.toISOString());
  }

  getToken(): string | null {
    return localStorage.getItem(LocalStorageTokenStorage.TOKEN_KEY);
  }

  clearToken(): void {
    localStorage.removeItem(LocalStorageTokenStorage.TOKEN_KEY);
    localStorage.removeItem(LocalStorageTokenStorage.EXPIRES_KEY);
  }

  isTokenExpired(): boolean {
    const raw = localStorage.getItem(LocalStorageTokenStorage.EXPIRES_KEY);
    return !raw || new Date(raw) <= new Date();
  }
}
```

### Step 6 — Mock adapters for tests

```typescript
// src/auth/adapters/secondary/mock/InMemoryUserRepository.ts
import { IUserRepository } from '../../../domain/ports/outbound/IUserRepository';
import { User } from '../../../domain/entities/User';

export class InMemoryUserRepository implements IUserRepository {
  private store: Map<string, User> = new Map();

  seed(users: User[]): void {
    users.forEach(u => {
      this.store.set(u.id, u);
      this.store.set(u.email.value, u);  // index by email too
    });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.store.get(email) ?? null;
  }

  async findById(id: string): Promise<User | null> {
    return this.store.get(id) ?? null;
  }
}
```

```typescript
// src/auth/adapters/secondary/mock/InMemoryTokenStorage.ts
import { ITokenStorage } from '../../../domain/ports/outbound/ITokenStorage';

export class InMemoryTokenStorage implements ITokenStorage {
  private token: string | null = null;
  private expiresAt: Date | null = null;

  saveToken(token: string, expiresAt: Date): void {
    this.token = token;
    this.expiresAt = expiresAt;
  }

  getToken(): string | null { return this.token; }

  clearToken(): void {
    this.token = null;
    this.expiresAt = null;
  }

  isTokenExpired(): boolean {
    if (!this.expiresAt) return true;
    return this.expiresAt <= new Date();
  }
}
```

---

## Testability: Swapping Adapters

This is the architecture's most immediate practical benefit. Unit-testing the domain core requires zero mocking frameworks — you simply inject in-memory adapters.

```typescript
// src/auth/application/AuthUseCase.test.ts
import { AuthUseCase } from './AuthUseCase';
import { InMemoryUserRepository } from '../adapters/secondary/mock/InMemoryUserRepository';
import { InMemoryTokenStorage } from '../adapters/secondary/mock/InMemoryTokenStorage';
import { User } from '../domain/entities/User';

// A trivial mock hasher — no bcrypt, no async complexity
class AlwaysValidHasher {
  async compare(_plain: string, _hashed: string): Promise<boolean> {
    return true;
  }
}

class AlwaysInvalidHasher {
  async compare(_plain: string, _hashed: string): Promise<boolean> {
    return false;
  }
}

describe('AuthUseCase', () => {
  const activeUser = User.reconstitute({
    id: 'u1',
    email: 'alice@example.com',
    role: 'member',
    passwordHash: 'irrelevant-because-mock',
    isActive: true,
  });

  function makeUseCase(overrides?: { hasher?: any }) {
    const repo = new InMemoryUserRepository();
    repo.seed([activeUser]);
    const storage = new InMemoryTokenStorage();
    const hasher = overrides?.hasher ?? new AlwaysValidHasher();
    return { useCase: new AuthUseCase(repo, storage, hasher), storage };
  }

  it('returns a token on successful login', async () => {
    const { useCase, storage } = makeUseCase();
    const result = await useCase.login({ email: 'alice@example.com', password: 'any' });
    expect(result.userId).toBe('u1');
    expect(storage.getToken()).not.toBeNull();
  });

  it('throws on unknown email', async () => {
    const { useCase } = makeUseCase();
    await expect(
      useCase.login({ email: 'ghost@example.com', password: 'any' })
    ).rejects.toThrow('Invalid credentials');
  });

  it('throws when password is wrong', async () => {
    const { useCase } = makeUseCase({ hasher: new AlwaysInvalidHasher() });
    await expect(
      useCase.login({ email: 'alice@example.com', password: 'wrong' })
    ).rejects.toThrow('Invalid credentials');
  });

  it('clears token on logout', async () => {
    const { useCase, storage } = makeUseCase();
    await useCase.login({ email: 'alice@example.com', password: 'any' });
    await useCase.logout();
    expect(storage.getToken()).toBeNull();
  });
});
```

**No HTTP server. No browser APIs. No React test renderer. No `jest.mock()` calls on modules.** The tests run in pure Node.js. Feedback loop is milliseconds, not seconds.

To switch from the mock adapter to the real HTTP adapter for an integration test, you replace one constructor argument:

```typescript
// Integration test — hits a real (or containerized) API
const repo = new HttpUserRepository('http://localhost:3001');
const useCase = new AuthUseCase(repo, new InMemoryTokenStorage(), new BcryptPasswordHasher());
```

The `AuthUseCase` source code does not change at all.

---

## Comparison with Clean Architecture and Layered Architecture

### vs. Clean Architecture (Robert C. Martin, 2012)

Clean Architecture and Hexagonal Architecture are deeply aligned — many practitioners treat them as the same idea expressed differently. The key similarities and differences:

| Dimension | Hexagonal | Clean Architecture |
|-----------|-----------|-------------------|
| **Origin** | Cockburn, 2005 | Martin, 2012 |
| **Shape metaphor** | Hexagon (ports around the perimeter) | Concentric circles (onion rings) |
| **Core concept** | Ports & Adapters | Entities, Use Cases, Interface Adapters, Frameworks |
| **Dependency direction** | Always inward (toward core) | Always inward (The Dependency Rule) |
| **Named layers** | Domain, Application, Adapters (informal) | Entities, Use Cases, Interface Adapters, Frameworks & Drivers |
| **Inversion of control** | Outbound ports inverted via DI | Same — interface defined in inner layer, implemented in outer |
| **Primary/Secondary split** | Explicit — driving vs. driven | Implicit — "Interface Adapters" covers both |
| **Framework independence** | Explicit design goal | Explicit design goal |

**Key difference in practice:** Clean Architecture prescribes more granular layers (Entities vs. Use Cases vs. Interface Adapters). Hexagonal Architecture is less prescriptive about internal structure — it focuses almost entirely on the boundary between core and outside world. You can implement Clean Architecture *as* Hexagonal Architecture by treating "Interface Adapters" as the adapter layer.

### vs. Layered Architecture (N-Tier)

| Dimension | Hexagonal | Layered |
|-----------|-----------|---------|
| **Dependency direction** | Always toward core (no "down" concept) | Downward — presentation → service → data |
| **Symmetry** | Symmetric: left and right sides are peers | Asymmetric: presentation is always "above" data |
| **Testability** | High — every port is swappable | Medium — service layer can be tested; data layer often requires a real DB |
| **Database coupling** | Explicitly decoupled via outbound port | Often coupled — data layer is a specific tier |
| **Framework coupling** | Explicitly decoupled via primary adapters | Often coupled — UI framework lives in presentation tier with no abstraction |
| **Complexity** | Higher — more interfaces and wiring | Lower — direct dependencies, familiar pattern |
| **Common violation** | Sneaking infrastructure into core | "Smart UI" anti-pattern, service bypassing |

The critical layered-architecture smell that hexagonal eliminates: a service importing `axios` directly. In layered architecture, nothing structurally prevents `UserService.ts` from importing `axios`. In hexagonal architecture, the service is part of the core, and the core **cannot** import `axios` because `axios` is never defined behind an outbound port interface.

---

## Trade-offs

### Costs

**Interface explosion.** Every interaction with the outside world requires an interface. For a feature with five data sources and three external services, you write eight interfaces, eight adapter classes, eight mock implementations. On a small app, this is more code than the actual business logic.

**Wiring boilerplate.** Somebody has to instantiate every adapter and inject it into the core. Without a DI container (Angular has one; React does not), this wiring code lives in "composition root" files that become long and tedious to maintain.

```typescript
// src/auth/composition-root.ts — wiring everything together
import { AuthUseCase } from './application/AuthUseCase';
import { HttpUserRepository } from './adapters/secondary/http/HttpUserRepository';
import { LocalStorageTokenStorage } from './adapters/secondary/storage/LocalStorageTokenStorage';
import { BcryptPasswordHasher } from './adapters/secondary/crypto/BcryptPasswordHasher';

export function createAuthUseCase(): IAuthUseCase {
  return new AuthUseCase(
    new HttpUserRepository(process.env.REACT_APP_API_URL!),
    new LocalStorageTokenStorage(),
    new BcryptPasswordHasher()
  );
}
```

This is clean but it accumulates. A large app might have dozens of composition roots or a complex DI module.

**Learning curve.** Developers familiar with "just call axios in the component" resist the indirection initially. The value only becomes apparent when the first adapter swap (e.g., replacing localStorage with IndexedDB) happens without touching any business logic.

**Overkill for small apps.** A CRUD app with two entities and one API endpoint does not benefit from hexagonal architecture. The overhead outweighs the payoff before the app reaches a complexity threshold.

**TypeScript discipline required.** The pattern collapses if developers import concrete classes instead of interfaces. ESLint rules or import boundary tools (like `dependency-cruiser` or NX module boundaries) must enforce the architecture at the tooling level.

### Benefits

- Unit tests run in pure Node.js with no infrastructure setup
- Framework migrations (React → Vue, Angular → React) touch only primary adapters
- Backend API changes (REST → GraphQL, fetch → axios) touch only secondary adapters
- Domain logic is readable without understanding any framework API
- New team members can understand business rules without learning infrastructure
- Multiple primary adapters in parallel (web UI + mobile + CLI) share one core

---

## When to Use

**Use hexagonal architecture when:**

- The team has 4+ developers working on the same feature area
- The application is expected to live for 3+ years with ongoing feature development
- High unit test coverage (80%+) is a requirement
- Multiple delivery mechanisms exist or are planned (web app + mobile shell + public API)
- Domain logic is complex enough to justify encapsulation (non-trivial validation, multi-step workflows, domain events)
- The underlying infrastructure (API structure, storage mechanism) may change
- You are building a design system library or shared core used across multiple products

**Do not use (or use a lighter variant) when:**

- The feature is a pure CRUD form with no business rules
- The app is a prototype or MVP with a 3-month lifespan
- The team is small (1-2 developers) and the cognitive overhead outweighs the benefit
- The domain is thin — the app is primarily a data display layer with no domain logic
- The team is not yet comfortable with interfaces and dependency injection

**Lighter variants for intermediate scenarios:**
- Apply hexagonal architecture only to specific high-value features, not the whole app
- Skip inbound ports if the UI coupling is acceptable; only use outbound ports to protect the core from infrastructure
- Use a service layer with interface-based repositories without the full hexagonal naming

---

## Interview Q&A

**Q: What is the fundamental difference between a port and an adapter?**

A: A port is a TypeScript interface — it is part of the domain core and expresses a capability the core needs (outbound) or exposes (inbound). It has no implementation. An adapter is a concrete class in the outer layer that implements a port interface by wiring to real infrastructure. The port is stable; adapters are swappable.

---

**Q: Why is a React component considered a "primary adapter" in hexagonal architecture?**

A: A primary (driving) adapter is something that initiates interaction with the application core. A React component captures user intent (form submission, button click) and translates it into a call on an inbound port interface (`IAuthUseCase.login(...)`). The component sits outside the domain core, depends on the port interface rather than any core class, and is replaceable — you could swap the React component for a Vue component or a CLI prompt without modifying the domain.

---

**Q: You have an `AuthService` that currently calls `axios` directly. How would you refactor it to hexagonal architecture?**

A: Extract the axios logic into a concrete class (`HttpUserRepository`) that implements an `IUserRepository` interface. Move the `IUserRepository` interface into the domain layer. Inject `IUserRepository` into `AuthService` via the constructor. The `AuthService` now depends only on the interface; axios lives exclusively in `HttpUserRepository`. For tests, inject `InMemoryUserRepository` instead.

---

**Q: How does hexagonal architecture differ from Clean Architecture?**

A: They share the same dependency rule (dependencies point inward toward the core) and the same goal (framework and infrastructure independence). The differences are naming and granularity. Clean Architecture prescribes four named layers (Entities, Use Cases, Interface Adapters, Frameworks & Drivers) and emphasizes the Entities/Use Cases distinction. Hexagonal Architecture focuses on the core-vs-outside boundary and makes explicit the primary/secondary adapter split. In practice, many teams implement Clean Architecture's inner layers inside the hexagonal core.

---

**Q: How does hexagonal architecture compare to layered (N-tier) architecture?**

A: Layered architecture organizes code into horizontal layers (presentation → service → data) with dependencies flowing downward. This works, but it does not prevent the service layer from directly importing infrastructure libraries. Hexagonal architecture makes the boundary explicit through port interfaces, so the core structurally cannot depend on infrastructure. Layered architecture is symmetric only by convention; hexagonal architecture encodes the symmetry in the type system.

---

**Q: What enforces the architectural boundary in practice? Can a developer just import a concrete class from the adapters folder into the domain?**

A: TypeScript alone does not prevent it. Enforcement requires tooling: `dependency-cruiser` rules that flag imports crossing forbidden boundaries, NX module boundary lint rules, or ESLint `import/no-restricted-paths` rules. Teams also use code review as a control. The architecture's value depends on this discipline being consistent — one violation (a core class importing axios) quietly destroys the testability and independence guarantees.

---

**Q: Is hexagonal architecture worth the overhead for a React CRUD app?**

A: Usually not. For a simple form that reads and writes one entity via a REST API, the interface/adapter indirection adds files and wiring with no payoff — there are no domain rules to protect, and swapping the adapter is unlikely. The pattern is justified when domain complexity (multi-step workflows, non-trivial validation, business rules that change independently of infrastructure) makes isolation valuable, or when test coverage requirements mean the cost of infrastructure in tests is painful enough to motivate the abstraction.

---

**Q: How do you wire adapters together in a React app without Angular's DI system?**

A: The "composition root" pattern. Create a factory function (or a context/provider at the app root) that instantiates all adapters and the use case, then provides the use case instance to the component tree via React context. Components receive `IAuthUseCase` from context and never import concrete classes. This is effectively manual DI. Libraries like `tsyringe` or `inversify` can automate it if the app grows large enough.

---

**Q: Could you use hexagonal architecture for server-side rendering (SSR) where localStorage is unavailable?**

A: Yes — this is one of its strongest use cases. Because `ITokenStorage` is an interface, you inject `LocalStorageTokenStorage` in the browser and a `CookieTokenStorage` or `InMemoryTokenStorage` on the server side. The `AuthUseCase` core code does not change. Without the port abstraction, the storage calls are scattered through the codebase and each one needs an `if (typeof window !== 'undefined')` guard.

---

## Key Takeaways

- **Hexagonal Architecture = Ports & Adapters.** The hexagon is a visual metaphor; the pattern is about isolating the domain core with interface boundaries.
- **Ports are TypeScript interfaces.** They live in the domain layer. They have no implementation. They define what the core can do (inbound) and what the core needs (outbound).
- **Adapters are concrete implementations.** Primary adapters drive the core (React components, test harnesses). Secondary adapters are driven by the core (HTTP clients, localStorage wrappers).
- **The domain core imports nothing outside the domain folder.** No framework imports, no infrastructure imports. This is the architectural invariant.
- **Testability is the most immediate benefit.** Inject in-memory adapters; test domain logic with zero infrastructure dependencies in pure Node.js.
- **Framework independence is the long-term benefit.** The entire primary adapter layer can be replaced (React → Angular) without touching domain logic.
- **Overhead is real.** Every external interaction needs an interface + adapter + mock. This is worth it at scale; it is ceremony at small scale.
- **Tooling enforcement is required.** TypeScript alone does not prevent boundary violations; use `dependency-cruiser`, NX boundaries, or ESLint `import/no-restricted-paths`.

---

*References:*
- Alistair Cockburn, "Hexagonal Architecture" (2005) — alistair.cockburn.us/hexagonal-architecture
- Robert C. Martin, "The Clean Architecture" (2012) — blog.cleancoder.com
- Tom Hombergs, "Get Your Hands Dirty on Clean Architecture" (2019)
