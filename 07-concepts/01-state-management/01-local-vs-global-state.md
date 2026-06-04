# Local vs Global State

## Overview

State management is one of the most critical architectural decisions in modern web applications. Understanding when to use local state versus global state directly impacts your application's maintainability, performance, and developer experience. This guide explores the fundamental differences, trade-offs, and practical patterns for managing state at different scopes.

## What is Local State?

Local state (also called component state) is data that belongs to a single component and is not shared with other parts of the application. It's encapsulated within the component's scope and lifecycle.

### Characteristics of Local State

- **Scoped to a component**: Only accessible within the component that owns it
- **Lifecycle-bound**: Created when component mounts, destroyed when it unmounts
- **Self-contained**: Changes don't affect other components directly
- **Performance-efficient**: Updates only trigger re-renders in the component tree below
- **Easy to reason about**: Clear ownership and predictable behavior

### When to Use Local State

1. **UI-specific state**: Toggle states, form inputs, modal visibility
2. **Temporary data**: Search queries, filter selections, pagination state
3. **Component-specific logic**: Hover states, focus management, animation states
4. **Derived state**: Calculations based on props or other local state
5. **Ephemeral data**: Data that doesn't need to persist beyond component lifecycle

## What is Global State?

Global state (also called shared state or application state) is data that needs to be accessed or modified by multiple components across different parts of the application, often at different nesting levels.

### Characteristics of Global State

- **Accessible anywhere**: Available to any component that needs it
- **Persistent**: Survives component mounting/unmounting
- **Shared**: Multiple components can read and update it
- **Centralized**: Single source of truth for shared data
- **Cross-cutting**: Used by components at different nesting levels

### When to Use Global State

1. **User authentication**: Current user, permissions, session data
2. **Theme/preferences**: Dark mode, language, accessibility settings
3. **Shopping cart**: E-commerce cart items, totals
4. **Notifications**: Toast messages, alerts visible across the app
5. **Cached API data**: Shared data fetched from servers
6. **Real-time data**: WebSocket connections, live updates

## The Problem: Prop Drilling

Before diving into solutions, understand the problem global state solves: prop drilling.

```
┌─────────────┐
│   App       │
│  user={u}   │
└──────┬──────┘
       │
       ├─────────────────┐
       ↓                 ↓
┌─────────────┐   ┌─────────────┐
│  Layout     │   │  Sidebar    │
│  user={u}   │   │  user={u}   │
└──────┬──────┘   └──────┬──────┘
       │                 │
       ↓                 ↓
┌─────────────┐   ┌─────────────┐
│  Header     │   │  UserMenu   │
│  user={u}   │   │  user={u}   │ ← Actually uses it
└─────────────┘   └─────────────┘
       │
       ↓
┌─────────────┐
│  UserBadge  │ ← Actually uses it
│  user={u}   │
└─────────────┘

Problem: Middle components pass props they don't use
```

## Local State Examples

### React Local State

```typescript
// Simple counter - perfect for local state
import { useState } from 'react';

function Counter() {
  // Local state - only this component needs it
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

```typescript
// Form with local state
import { useState } from 'react';

function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });
  
  const [errors, setErrors] = useState<Record<string, string>>({});
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
    
    // Clear error when user types
    if (errors[name]) {
      setErrors(prev => {
        const next = { ...prev };
        delete next[name];
        return next;
      });
    }
  };
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // Validate and submit
    const newErrors: Record<string, string> = {};
    
    if (!formData.name) newErrors.name = 'Name is required';
    if (!formData.email) newErrors.email = 'Email is required';
    
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    
    // Submit form...
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={formData.name}
        onChange={handleChange}
        placeholder="Name"
      />
      {errors.name && <span className="error">{errors.name}</span>}
      
      <input
        name="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="Email"
      />
      {errors.email && <span className="error">{errors.email}</span>}
      
      <textarea
        name="message"
        value={formData.message}
        onChange={handleChange}
        placeholder="Message"
      />
      
      <button type="submit">Send</button>
    </form>
  );
}
```

### Angular Local State

```typescript
// Simple counter with Angular signals
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <p>Count: {{ count() }}</p>
      <button (click)="increment()">Increment</button>
    </div>
  `
})
export class CounterComponent {
  // Local state using signals
  count = signal(0);
  
  increment() {
    this.count.update(c => c + 1);
  }
}
```

```typescript
// Form with local state
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-contact-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="name" placeholder="Name">
      <span *ngIf="form.get('name')?.errors?.['required'] && form.get('name')?.touched">
        Name is required
      </span>
      
      <input formControlName="email" placeholder="Email">
      <span *ngIf="form.get('email')?.errors?.['required'] && form.get('email')?.touched">
        Email is required
      </span>
      
      <textarea formControlName="message" placeholder="Message"></textarea>
      
      <button type="submit" [disabled]="form.invalid">Send</button>
    </form>
  `
})
export class ContactFormComponent {
  form: FormGroup;
  
  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
      message: ['']
    });
  }
  
  onSubmit() {
    if (this.form.valid) {
      console.log(this.form.value);
    }
  }
}
```

## Global State Examples

### React Context for Global State

```typescript
// User context - global authentication state
import { createContext, useContext, useState, ReactNode } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

interface UserContextType {
  user: User | null;
  login: (user: User) => void;
  logout: () => void;
  isAuthenticated: boolean;
}

const UserContext = createContext<UserContextType | undefined>(undefined);

export function UserProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  
  const login = (user: User) => {
    setUser(user);
    localStorage.setItem('user', JSON.stringify(user));
  };
  
  const logout = () => {
    setUser(null);
    localStorage.removeItem('user');
  };
  
  const value = {
    user,
    login,
    logout,
    isAuthenticated: !!user
  };
  
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}

export function useUser() {
  const context = useContext(UserContext);
  if (!context) {
    throw new Error('useUser must be used within UserProvider');
  }
  return context;
}
```

```typescript
// Usage in deeply nested components
function UserBadge() {
  const { user, isAuthenticated } = useUser();
  
  if (!isAuthenticated) {
    return <div>Not logged in</div>;
  }
  
  return (
    <div className="user-badge">
      <img src={`/avatars/${user.id}.jpg`} alt={user.name} />
      <span>{user.name}</span>
    </div>
  );
}

function UserMenu() {
  const { user, logout } = useUser();
  
  return (
    <div className="user-menu">
      <p>Hello, {user?.name}</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

// No prop drilling needed!
function App() {
  return (
    <UserProvider>
      <Layout>
        <Header>
          <UserBadge />
        </Header>
        <Sidebar>
          <UserMenu />
        </Sidebar>
        <Content />
      </Layout>
    </UserProvider>
  );
}
```

### Angular Services for Global State

```typescript
// User service - global authentication state
import { Injectable, signal, computed } from '@angular/core';

export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Global state using signals
  private userSignal = signal<User | null>(null);
  
  // Computed derived state
  user = this.userSignal.asReadonly();
  isAuthenticated = computed(() => this.userSignal() !== null);
  isAdmin = computed(() => this.userSignal()?.role === 'admin');
  
  login(user: User) {
    this.userSignal.set(user);
    localStorage.setItem('user', JSON.stringify(user));
  }
  
  logout() {
    this.userSignal.set(null);
    localStorage.removeItem('user');
  }
  
  constructor() {
    // Restore from localStorage
    const stored = localStorage.getItem('user');
    if (stored) {
      try {
        this.userSignal.set(JSON.parse(stored));
      } catch (e) {
        localStorage.removeItem('user');
      }
    }
  }
}
```

```typescript
// Usage in components
import { Component, inject } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-badge',
  template: `
    <div *ngIf="userService.isAuthenticated()">
      <img [src]="'/avatars/' + userService.user()?.id + '.jpg'" 
           [alt]="userService.user()?.name">
      <span>{{ userService.user()?.name }}</span>
    </div>
    <div *ngIf="!userService.isAuthenticated()">
      Not logged in
    </div>
  `
})
export class UserBadgeComponent {
  userService = inject(UserService);
}

@Component({
  selector: 'app-user-menu',
  template: `
    <div class="user-menu">
      <p>Hello, {{ userService.user()?.name }}</p>
      <button (click)="userService.logout()">Logout</button>
    </div>
  `
})
export class UserMenuComponent {
  userService = inject(UserService);
}
```

## Lifting State: The Middle Ground

Sometimes state starts as local but needs to be shared with sibling components. The solution is "lifting state up" to a common parent.

```typescript
// Before: Two components with separate local state (out of sync)
function FilterBar() {
  const [searchTerm, setSearchTerm] = useState('');
  return <input value={searchTerm} onChange={e => setSearchTerm(e.target.value)} />;
}

function ProductList() {
  const [searchTerm, setSearchTerm] = useState(''); // Duplicate state!
  // How does this component know about FilterBar's search term?
}
```

```typescript
// After: Lifted state to common parent
function ProductPage() {
  // State lifted to parent component
  const [searchTerm, setSearchTerm] = useState('');
  
  return (
    <div>
      <FilterBar searchTerm={searchTerm} onSearchChange={setSearchTerm} />
      <ProductList searchTerm={searchTerm} />
    </div>
  );
}

function FilterBar({ searchTerm, onSearchChange }: {
  searchTerm: string;
  onSearchChange: (term: string) => void;
}) {
  return (
    <input 
      value={searchTerm} 
      onChange={e => onSearchChange(e.target.value)} 
    />
  );
}

function ProductList({ searchTerm }: { searchTerm: string }) {
  const filteredProducts = products.filter(p => 
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  );
  
  return (
    <ul>
      {filteredProducts.map(p => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

### Lifting State Architecture

```
Before (out of sync):
┌─────────────────┐
│   ProductPage   │
└────────┬────────┘
         │
    ┌────┴─────┐
    ↓          ↓
┌─────────┐ ┌───────────┐
│FilterBar│ │ProductList│
│state: S │ │state: S   │ ← Duplicate state!
└─────────┘ └───────────┘

After (lifted state):
┌─────────────────┐
│   ProductPage   │
│   state: S      │ ← Single source of truth
└────────┬────────┘
         │
    ┌────┴─────┐
    ↓          ↓
┌─────────┐ ┌───────────┐
│FilterBar│ │ProductList│
│props: S │ │props: S   │ ← Receive via props
└─────────┘ └───────────┘
```

## Decision Matrix: Local vs Global State

```
┌────────────────────────────────────────────────────────┐
│ Question                          │ Local │ Global     │
├────────────────────────────────────────────────────────┤
│ Used by single component?         │  ✓    │            │
│ Used by multiple components?      │       │    ✓       │
│ Needs to persist after unmount?   │       │    ✓       │
│ UI-specific (hover, focus)?       │  ✓    │            │
│ Business logic/data?              │       │    ✓       │
│ Temporary/ephemeral?              │  ✓    │            │
│ Needs to be cached/reused?        │       │    ✓       │
│ Part of URL/shareable?            │       │    ✓       │
│ Affects multiple routes?          │       │    ✓       │
│ Form input before submission?     │  ✓    │            │
│ Authentication/user data?         │       │    ✓       │
│ Theme/preferences?                │       │    ✓       │
└────────────────────────────────────────────────────────┘
```

## Hybrid Approach: Server State vs Client State

Modern applications often distinguish between server state and client state:

### Server State
- Data fetched from APIs
- Cached and synchronized with server
- Shared across components
- Tools: React Query, SWR, Apollo Client, RTK Query

### Client State
- UI state, preferences, temporary data
- Doesn't exist on server
- Tools: Context, Zustand, Redux, Signals

```typescript
// Hybrid example with React Query for server state
import { useQuery } from '@tanstack/react-query';
import { useState } from 'react';

function ProductList() {
  // Server state: managed by React Query
  const { data: products, isLoading } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts
  });
  
  // Client state: local to this component
  const [selectedCategory, setSelectedCategory] = useState('all');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('asc');
  
  // Derived state: computed from server + client state
  const filteredProducts = products
    ?.filter(p => selectedCategory === 'all' || p.category === selectedCategory)
    .sort((a, b) => sortOrder === 'asc' 
      ? a.price - b.price 
      : b.price - a.price
    );
  
  if (isLoading) return <div>Loading...</div>;
  
  return (
    <div>
      <select value={selectedCategory} onChange={e => setSelectedCategory(e.target.value)}>
        <option value="all">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>
      
      <button onClick={() => setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc')}>
        Sort: {sortOrder}
      </button>
      
      <ul>
        {filteredProducts?.map(p => (
          <li key={p.id}>{p.name} - ${p.price}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Performance Considerations

### Local State Performance

```typescript
// Good: Local state doesn't affect siblings
function Sidebar() {
  const [isExpanded, setIsExpanded] = useState(false);
  // Re-renders only affect Sidebar and its children
  return (
    <div>
      <button onClick={() => setIsExpanded(!isExpanded)}>Toggle</button>
      {isExpanded && <SidebarContent />}
    </div>
  );
}
```

### Global State Performance Issues

```typescript
// Bad: Single global state object causes unnecessary re-renders
const globalState = {
  user: { ... },
  theme: 'dark',
  cart: [...],
  notifications: [...],
  sidebar: { isExpanded: false } // UI state in global store!
};

// Problem: Toggling sidebar re-renders ENTIRE app
function Sidebar() {
  const { sidebar, updateSidebar } = useGlobalState();
  // Every component subscribed to globalState re-renders!
  return (
    <button onClick={() => updateSidebar({ isExpanded: !sidebar.isExpanded })}>
      Toggle
    </button>
  );
}
```

```typescript
// Good: Granular subscriptions with selectors
function Sidebar() {
  // Only subscribes to sidebar slice
  const isExpanded = useGlobalState(state => state.sidebar.isExpanded);
  const updateSidebar = useGlobalState(state => state.updateSidebar);
  
  // Only this component re-renders on sidebar state changes
  return (
    <button onClick={() => updateSidebar({ isExpanded: !isExpanded })}>
      Toggle
    </button>
  );
}
```

## State Colocation

Keep state as close as possible to where it's used. This improves performance and maintainability.

```typescript
// Bad: State too high in tree
function App() {
  const [modalOpen, setModalOpen] = useState(false);
  const [formData, setFormData] = useState({});
  const [filterTerm, setFilterTerm] = useState('');
  const [sortOrder, setSortOrder] = useState('asc');
  // ... all state in root component
  
  return (
    <Layout>
      <ProductPage 
        filterTerm={filterTerm}
        setFilterTerm={setFilterTerm}
        sortOrder={sortOrder}
        setSortOrder={setSortOrder}
      />
      <Modal open={modalOpen} setOpen={setModalOpen}>
        <Form data={formData} setData={setFormData} />
      </Modal>
    </Layout>
  );
}
```

```typescript
// Good: State colocated with components that use it
function App() {
  return (
    <Layout>
      <ProductPage /> {/* Manages its own filter/sort state */}
      <ModalWithForm /> {/* Manages its own modal/form state */}
    </Layout>
  );
}

function ProductPage() {
  const [filterTerm, setFilterTerm] = useState('');
  const [sortOrder, setSortOrder] = useState('asc');
  // State only affects this subtree
}

function ModalWithForm() {
  const [open, setOpen] = useState(false);
  const [formData, setFormData] = useState({});
  // State only affects this subtree
}
```

## Common Mistakes

### 1. Everything in Global State

```typescript
// Bad: UI state in global store
const store = {
  user: { ... },
  theme: 'dark',
  hoverState: false,        // Should be local!
  mousePosition: { x, y },  // Should be local!
  inputValue: '',           // Should be local!
  modalOpen: false          // Should be local!
};
```

### 2. Premature Global State

```typescript
// Bad: Making state global before it's needed
const ThemeContext = createContext();

function Button() {
  const theme = useContext(ThemeContext);
  // Only this component uses theme - doesn't need global state yet
  return <button className={theme}>Click</button>;
}
```

### 3. Not Lifting State When Needed

```typescript
// Bad: Duplicate state in siblings
function FilterBar() {
  const [category, setCategory] = useState('all');
  // ...
}

function ProductList() {
  const [category, setCategory] = useState('all'); // Out of sync!
  // ...
}

// Good: Lift to parent
function ProductPage() {
  const [category, setCategory] = useState('all');
  return (
    <>
      <FilterBar category={category} onChange={setCategory} />
      <ProductList category={category} />
    </>
  );
}
```

### 4. Overusing Context

```typescript
// Bad: Separate context for everything
<UserContext.Provider>
  <ThemeContext.Provider>
    <CartContext.Provider>
      <NotificationContext.Provider>
        <ModalContext.Provider>
          <App />
        </ModalContext.Provider>
      </NotificationContext.Provider>
    </CartContext.Provider>
  </ThemeContext.Provider>
</UserContext.Provider>

// Better: Use a state management library
<StateProvider>
  <App />
</StateProvider>
```

### 5. Props Drilling Instead of Composition

```typescript
// Bad: Passing props through many layers
function App() {
  const [user, setUser] = useState(null);
  return <Layout user={user}><Content user={user} /></Layout>;
}

// Good: Use composition to avoid drilling
function App() {
  const [user, setUser] = useState(null);
  return (
    <Layout>
      <Content>
        <UserProfile user={user} /> {/* Direct prop, no drilling */}
      </Content>
    </Layout>
  );
}
```

## Best Practices

### 1. Start with Local State

Always begin with local state and only make it global when you have a concrete need.

```typescript
// Start here
function Component() {
  const [state, setState] = useState(initialValue);
  // ...
}

// Move to global only when:
// - Multiple components need it
// - It needs to persist
// - You're experiencing prop drilling
```

### 2. Use Composition to Avoid Prop Drilling

```typescript
// Instead of drilling props
function Page({ user, theme, cart }) {
  return <Layout user={user} theme={theme} cart={cart} />;
}

// Use component composition
function Page() {
  return (
    <Layout
      header={<Header />}
      sidebar={<Sidebar />}
      content={<Content />}
    />
  );
}
```

### 3. Separate Server State from Client State

```typescript
// Server state: React Query, SWR
const { data: products } = useQuery(['products'], fetchProducts);

// Client state: useState, Context, Zustand
const [selectedIds, setSelectedIds] = useState<string[]>([]);
```

### 4. Colocate State

Keep state as close as possible to where it's used.

```typescript
// Bad: State far from usage
function App() {
  const [modalOpen, setModalOpen] = useState(false);
  return <DeepComponent modalOpen={modalOpen} />;
}

// Good: State close to usage
function ModalContainer() {
  const [open, setOpen] = useState(false);
  return <Modal open={open} onClose={() => setOpen(false)} />;
}
```

### 5. Use Derived State

Don't store state that can be computed from other state.

```typescript
// Bad: Storing derived state
const [products, setProducts] = useState([]);
const [filteredProducts, setFilteredProducts] = useState([]);

useEffect(() => {
  setFilteredProducts(products.filter(p => p.inStock));
}, [products]);

// Good: Compute derived state
const [products, setProducts] = useState([]);
const filteredProducts = products.filter(p => p.inStock);
```

### 6. Normalize Global State

```typescript
// Bad: Nested/denormalized state
const state = {
  posts: [
    { id: 1, author: { id: 1, name: 'John' }, comments: [...] },
    { id: 2, author: { id: 1, name: 'John' }, comments: [...] } // Duplicate author
  ]
};

// Good: Normalized state
const state = {
  posts: {
    byId: { 1: { id: 1, authorId: 1, commentIds: [1, 2] } },
    allIds: [1, 2]
  },
  authors: {
    byId: { 1: { id: 1, name: 'John' } },
    allIds: [1]
  },
  comments: {
    byId: { 1: { id: 1, text: '...' } },
    allIds: [1, 2]
  }
};
```

### 7. Consider URL State

For shareable/bookmarkable state, use URL parameters.

```typescript
// State in URL
function ProductPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get('category') || 'all';
  const sort = searchParams.get('sort') || 'asc';
  
  const updateFilter = (category: string) => {
    setSearchParams({ category, sort });
  };
  
  // URL: /products?category=electronics&sort=asc
}
```

## Interview Questions

### Q1: When should you use local state vs global state?

**Answer:** Use local state when data is only needed by a single component or its direct children, such as form inputs, toggle states, or UI-specific state. Use global state when data needs to be accessed by multiple components at different nesting levels, needs to persist across navigation, or represents shared application state like authentication, theme, or cached API data.

The decision often follows this hierarchy:
1. Start with local state
2. If siblings need it, lift to common parent
3. If prop drilling becomes excessive (3+ levels), consider global state
4. For server data, use specialized libraries like React Query
5. For cross-cutting concerns (auth, theme), use global state

### Q2: What is prop drilling and how do you solve it?

**Answer:** Prop drilling occurs when you pass props through multiple intermediate components that don't use the data, just to get it to deeply nested components that do need it.

Solutions:
1. **Component composition**: Use children props to avoid drilling
2. **Context API**: Create a context for shared data
3. **State management libraries**: Redux, Zustand, Jotai for complex cases
4. **Lifting state**: Move state to common ancestor (for nearby components)

Example of composition:
```typescript
// Instead of: <Layout user={user}><Content user={user} /></Layout>
// Do: <Layout><Content><UserProfile user={user} /></Content></Layout>
```

### Q3: How do you decide whether to lift state up?

**Answer:** Lift state up when:

1. **Sibling components need to share data**: Search filter and results list
2. **State needs to be synchronized**: Multiple components showing same data
3. **Parent needs control over child state**: Form validation, submission control
4. **Derived state depends on multiple sources**: Calculations from several inputs

Don't lift state when:
- Only one component tree needs it
- State is purely UI-specific to one component
- Lifting would cause excessive re-renders
- Global state would be more appropriate (many components, different levels)

### Q4: What's the difference between client state and server state?

**Answer:** 

**Client state**: Data that exists only in the browser
- UI state (modals, tabs, filters)
- Form inputs before submission
- User preferences
- Managed by: useState, Context, Redux, Zustand

**Server state**: Data fetched from APIs
- User data, posts, products
- Requires caching, revalidation, synchronization
- Has loading/error states
- Can be stale or outdated
- Managed by: React Query, SWR, Apollo Client, RTK Query

Modern applications often use specialized libraries for server state and simple solutions for client state, rather than mixing both in Redux/Context.

### Q5: How do you prevent unnecessary re-renders with global state?

**Answer:** Several strategies:

1. **Granular subscriptions**: Only subscribe to needed slice
```typescript
const user = useStore(state => state.user); // Not entire state
```

2. **Memoization**: Use React.memo, useMemo, useCallback
```typescript
const Component = React.memo(({ user }) => {...});
```

3. **State splitting**: Separate fast-changing from slow-changing state
```typescript
// Bad: One store with both
{ user: {...}, mousePosition: { x, y } }

// Good: Separate stores
userStore: { user: {...} }
uiStore: { mousePosition: { x, y } }
```

4. **Derived state**: Compute outside render
```typescript
const selector = (state) => state.items.filter(i => i.active);
const activeItems = useStore(selector);
```

5. **Atom-based state**: Use Recoil/Jotai for fine-grained reactivity

### Q6: Should form state be local or global?

**Answer:** Generally **local** until submission:

**Local form state** (most cases):
```typescript
function Form() {
  const [data, setData] = useState({});
  const handleSubmit = async () => {
    await api.post('/endpoint', data);
    // Now it becomes server state
  };
}
```

**Global form state** when:
- Multi-step forms spanning routes
- Draft saving (like email drafts)
- Form data affects other parts of UI
- Collaborative editing

Libraries like React Hook Form keep form state local for performance, only surfacing to global state on submission.

### Q7: How do you handle state that needs to be in URL?

**Answer:** Use URL state for shareable/bookmarkable data:

```typescript
// React Router
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const category = searchParams.get('category') || 'all';
  const page = parseInt(searchParams.get('page') || '1');
  
  const updateCategory = (cat: string) => {
    setSearchParams({ category: cat, page: '1' });
  };
  
  // URL: /products?category=electronics&page=1
}

// Next.js
function ProductList() {
  const router = useRouter();
  const { category = 'all', page = '1' } = router.query;
  
  const updateCategory = (cat: string) => {
    router.push({
      pathname: '/products',
      query: { category: cat, page: 1 }
    });
  };
}
```

Put in URL:
- Search queries
- Filters, sorting
- Pagination
- Active tabs
- Modal state (if shareable)

Keep in local/global state:
- Form inputs (before submission)
- UI hover/focus state
- Temporary selections

### Q8: What are the trade-offs of using Context vs a state management library?

**Answer:**

**Context Pros:**
- Built into React, no dependencies
- Simple for small-to-medium apps
- Type-safe with TypeScript
- Good for rarely-changing data (theme, auth)

**Context Cons:**
- All consumers re-render on any change
- No built-in devtools
- No middleware/plugins
- Requires manual optimization
- Can lead to provider hell

**State Management Library (Redux, Zustand) Pros:**
- Granular subscriptions (less re-renders)
- DevTools for debugging
- Middleware ecosystem
- Better for complex/frequent updates
- Predictable patterns

**State Management Library Cons:**
- Extra dependency
- Learning curve
- More boilerplate (Redux)
- Can be overkill for simple apps

**Recommendation:** Use Context for theme/auth, specialized libraries (React Query) for server state, and state management libraries for complex client state with frequent updates.

## Key Takeaways

- Start with local state and only move to global when there's a concrete need
- Local state is for component-specific data; global state is for shared data
- Prop drilling can be solved with composition, Context, or state management libraries
- Distinguish between server state (API data) and client state (UI state)
- Use specialized libraries like React Query for server state
- Keep state as close as possible to where it's used (colocation)
- Lift state only to the nearest common ancestor
- Don't store derived state - compute it
- Use URL state for shareable/bookmarkable data
- Optimize global state with granular subscriptions and memoization
- Form state should usually be local until submission
- Context is great for infrequent updates; state libraries for frequent updates
- Normalize global state to avoid duplication and simplify updates
- Consider performance implications when choosing state location
- Use composition to avoid unnecessary prop drilling

## Resources

- [React State Management](https://react.dev/learn/managing-state)
- [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)
- [Application State Management with React](https://kentcdodds.com/blog/application-state-management-with-react)
- [React Query vs Redux](https://tkdodo.eu/blog/react-query-vs-redux)
- [Angular State Management](https://angular.dev/guide/signals)
- [State Colocation](https://kentcdodds.com/blog/state-colocation-will-make-your-react-app-faster)
- [Lifting State Up](https://react.dev/learn/sharing-state-between-components)
- [Client vs Server State](https://tanstack.com/query/latest/docs/framework/react/guides/does-this-replace-client-state)
