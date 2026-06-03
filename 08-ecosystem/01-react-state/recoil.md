# Recoil - A State Management Library for React

## Overview

Recoil is a state management library for React developed by Facebook. It provides a way to build data-flow graphs that flow from atoms (shared state) through selectors (pure functions) and into React components. It offers orthogonal but optimized updates and subscription mechanisms.

### Key Features

- **Atom-based**: Minimal and composable units of state
- **Derived state**: Selectors compute derived state
- **Async built-in**: First-class async support
- **Concurrent Mode ready**: Built for React Concurrent features
- **DevTools**: Time travel and debugging
- **Atom families**: Dynamic atom creation
- **Fine-grained subscriptions**: Efficient re-renders

## Installation and Setup

```bash
# npm
npm install recoil

# yarn
yarn add recoil

# pnpm
pnpm add recoil
```

### Root Setup

```typescript
import React from 'react';
import { RecoilRoot } from 'recoil';
import App from './App';

function Root() {
  return (
    <RecoilRoot>
      <App />
    </RecoilRoot>
  );
}

export default Root;
```

## Atoms

### Basic Atoms

```typescript
import { atom, useRecoilState, useRecoilValue, useSetRecoilState } from 'recoil';

// Define atoms
const countState = atom({
  key: 'countState', // unique ID
  default: 0, // default value
});

const textState = atom({
  key: 'textState',
  default: 'Hello',
});

// Using atoms in components
function Counter() {
  const [count, setCount] = useRecoilState(countState);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

// Read-only
function CountDisplay() {
  const count = useRecoilValue(countState);
  return <div>Count: {count}</div>;
}

// Write-only
function CountControls() {
  const setCount = useSetRecoilState(countState);
  
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <button onClick={() => setCount(c => c - 1)}>-</button>
    </div>
  );
}
```

### Complex Atoms

```typescript
import { atom } from 'recoil';

interface User {
  id: string;
  name: string;
  email: string;
  preferences: {
    theme: 'light' | 'dark';
    notifications: boolean;
  };
}

const userState = atom<User | null>({
  key: 'userState',
  default: null,
});

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

const todosState = atom<Todo[]>({
  key: 'todosState',
  default: [],
});

const filterState = atom<'all' | 'active' | 'completed'>({
  key: 'filterState',
  default: 'all',
});

function UserProfile() {
  const [user, setUser] = useRecoilState(userState);

  if (!user) return <div>Not logged in</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <label>
        <input
          type="checkbox"
          checked={user.preferences.notifications}
          onChange={(e) => setUser({
            ...user,
            preferences: {
              ...user.preferences,
              notifications: e.target.checked,
            },
          })}
        />
        Notifications
      </label>
    </div>
  );
}
```

## Selectors

### Read-Only Selectors

```typescript
import { atom, selector, useRecoilValue } from 'recoil';

const fahrenheitState = atom({
  key: 'fahrenheitState',
  default: 32,
});

const celsiusState = selector({
  key: 'celsiusState',
  get: ({ get }) => {
    const fahrenheit = get(fahrenheitState);
    return ((fahrenheit - 32) * 5) / 9;
  },
});

function TemperatureDisplay() {
  const fahrenheit = useRecoilValue(fahrenheitState);
  const celsius = useRecoilValue(celsiusState);

  return (
    <div>
      <p>Fahrenheit: {fahrenheit}°F</p>
      <p>Celsius: {celsius.toFixed(2)}°C</p>
    </div>
  );
}

// Multiple dependencies
const priceState = atom({
  key: 'priceState',
  default: 100,
});

const quantityState = atom({
  key: 'quantityState',
  default: 1,
});

const taxRateState = atom({
  key: 'taxRateState',
  default: 0.1,
});

const subtotalState = selector({
  key: 'subtotalState',
  get: ({ get }) => {
    return get(priceState) * get(quantityState);
  },
});

const taxState = selector({
  key: 'taxState',
  get: ({ get }) => {
    return get(subtotalState) * get(taxRateState);
  },
});

const totalState = selector({
  key: 'totalState',
  get: ({ get }) => {
    return get(subtotalState) + get(taxState);
  },
});

function PriceBreakdown() {
  const subtotal = useRecoilValue(subtotalState);
  const tax = useRecoilValue(taxState);
  const total = useRecoilValue(totalState);

  return (
    <div>
      <p>Subtotal: ${subtotal.toFixed(2)}</p>
      <p>Tax: ${tax.toFixed(2)}</p>
      <p><strong>Total: ${total.toFixed(2)}</strong></p>
    </div>
  );
}
```

### Writable Selectors

```typescript
import { atom, selector, useRecoilState } from 'recoil';

const tempCelsiusState = atom({
  key: 'tempCelsiusState',
  default: 0,
});

const tempFahrenheitState = selector({
  key: 'tempFahrenheitState',
  get: ({ get }) => {
    const celsius = get(tempCelsiusState);
    return (celsius * 9) / 5 + 32;
  },
  set: ({ set }, newValue) => {
    const fahrenheit = newValue as number;
    const celsius = ((fahrenheit - 32) * 5) / 9;
    set(tempCelsiusState, celsius);
  },
});

function TempConverter() {
  const [celsius, setCelsius] = useRecoilState(tempCelsiusState);
  const [fahrenheit, setFahrenheit] = useRecoilState(tempFahrenheitState);

  return (
    <div>
      <div>
        <label>Celsius: </label>
        <input
          type="number"
          value={celsius}
          onChange={(e) => setCelsius(Number(e.target.value))}
        />
      </div>
      <div>
        <label>Fahrenheit: </label>
        <input
          type="number"
          value={fahrenheit}
          onChange={(e) => setFahrenheit(Number(e.target.value))}
        />
      </div>
    </div>
  );
}
```

### Selector with Filtering

```typescript
import { atom, selector, useRecoilValue } from 'recoil';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

const todoListState = atom<Todo[]>({
  key: 'todoListState',
  default: [],
});

const todoListFilterState = atom<'all' | 'active' | 'completed'>({
  key: 'todoListFilterState',
  default: 'all',
});

const filteredTodoListState = selector({
  key: 'filteredTodoListState',
  get: ({ get }) => {
    const filter = get(todoListFilterState);
    const list = get(todoListState);

    switch (filter) {
      case 'completed':
        return list.filter((item) => item.completed);
      case 'active':
        return list.filter((item) => !item.completed);
      default:
        return list;
    }
  },
});

const todoListStatsState = selector({
  key: 'todoListStatsState',
  get: ({ get }) => {
    const todoList = get(todoListState);
    const totalNum = todoList.length;
    const totalCompletedNum = todoList.filter((item) => item.completed).length;
    const totalUncompletedNum = totalNum - totalCompletedNum;
    const percentCompleted = totalNum === 0 ? 0 : (totalCompletedNum / totalNum) * 100;

    return {
      totalNum,
      totalCompletedNum,
      totalUncompletedNum,
      percentCompleted,
    };
  },
});

function TodoStats() {
  const stats = useRecoilValue(todoListStatsState);

  return (
    <div>
      <p>Total items: {stats.totalNum}</p>
      <p>Items completed: {stats.totalCompletedNum}</p>
      <p>Items not completed: {stats.totalUncompletedNum}</p>
      <p>Percent completed: {stats.percentCompleted.toFixed(0)}%</p>
    </div>
  );
}
```

## Async Selectors

### Fetching Data

```typescript
import { selector, useRecoilValue } from 'recoil';
import { Suspense } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

const currentUserIDState = atom({
  key: 'currentUserIDState',
  default: '1',
});

const currentUserState = selector({
  key: 'currentUserState',
  get: async ({ get }) => {
    const userId = get(currentUserIDState);
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) throw new Error('Failed to fetch user');
    return response.json() as Promise<User>;
  },
});

function CurrentUserInfo() {
  const user = useRecoilValue(currentUserState);
  
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <CurrentUserInfo />
    </Suspense>
  );
}
```

### Error Handling with ErrorBoundary

```typescript
import { selector, useRecoilValue } from 'recoil';
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

const productsQuery = selector({
  key: 'productsQuery',
  get: async () => {
    const response = await fetch('/api/products');
    if (!response.ok) throw new Error('Failed to fetch products');
    return response.json();
  },
});

function ProductList() {
  const products = useRecoilValue(productsQuery);
  
  return (
    <ul>
      {products.map((product: any) => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  );
}

function App() {
  return (
    <ErrorBoundary
      fallback={<div>Error loading products</div>}
      onReset={() => window.location.reload()}
    >
      <Suspense fallback={<div>Loading...</div>}>
        <ProductList />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Async Selector with Dependencies

```typescript
import { atom, selector, useRecoilValue, useSetRecoilState } from 'recoil';

const searchQueryState = atom({
  key: 'searchQueryState',
  default: '',
});

const searchResultsState = selector({
  key: 'searchResultsState',
  get: async ({ get }) => {
    const query = get(searchQueryState);
    if (!query) return [];
    
    const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
    if (!response.ok) throw new Error('Search failed');
    return response.json();
  },
});

function SearchBox() {
  const setSearchQuery = useSetRecoilState(searchQueryState);

  return (
    <input
      type="text"
      placeholder="Search..."
      onChange={(e) => setSearchQuery(e.target.value)}
    />
  );
}

function SearchResults() {
  const results = useRecoilValue(searchResultsState);

  return (
    <ul>
      {results.map((result: any) => (
        <li key={result.id}>{result.title}</li>
      ))}
    </ul>
  );
}

function SearchApp() {
  return (
    <div>
      <SearchBox />
      <Suspense fallback={<div>Searching...</div>}>
        <SearchResults />
      </Suspense>
    </div>
  );
}
```

## Atom Families

### Dynamic Atom Creation

```typescript
import { atomFamily, useRecoilState } from 'recoil';

interface ItemState {
  label: string;
  checked: boolean;
}

const itemState = atomFamily<ItemState, string>({
  key: 'itemState',
  default: (id) => ({
    label: `Item ${id}`,
    checked: false,
  }),
});

function ChecklistItem({ id }: { id: string }) {
  const [item, setItem] = useRecoilState(itemState(id));

  return (
    <div>
      <input
        type="checkbox"
        checked={item.checked}
        onChange={(e) => setItem({ ...item, checked: e.target.checked })}
      />
      <input
        value={item.label}
        onChange={(e) => setItem({ ...item, label: e.target.value })}
      />
    </div>
  );
}

function Checklist() {
  const itemIds = ['a', 'b', 'c'];
  
  return (
    <div>
      {itemIds.map(id => (
        <ChecklistItem key={id} id={id} />
      ))}
    </div>
  );
}
```

### Async Atom Family

```typescript
import { atomFamily, selectorFamily, useRecoilValue } from 'recoil';

interface User {
  id: string;
  name: string;
  email: string;
}

const userQuery = selectorFamily<User, string>({
  key: 'userQuery',
  get: (userId) => async () => {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) throw new Error('Failed to fetch user');
    return response.json();
  },
});

function UserInfo({ userId }: { userId: string }) {
  const user = useRecoilValue(userQuery(userId));
  
  return (
    <div>
      <h4>{user.name}</h4>
      <p>{user.email}</p>
    </div>
  );
}

function UserList() {
  const userIds = ['1', '2', '3'];
  
  return (
    <div>
      {userIds.map(userId => (
        <Suspense key={userId} fallback={<div>Loading user...</div>}>
          <UserInfo userId={userId} />
        </Suspense>
      ))}
    </div>
  );
}
```

## Selector Families

```typescript
import { selectorFamily, atom, useRecoilValue } from 'recoil';

const itemsState = atom<Array<{ id: string; name: string; categoryId: string }>>({
  key: 'itemsState',
  default: [],
});

const itemsByCategory = selectorFamily<
  Array<{ id: string; name: string; categoryId: string }>,
  string
>({
  key: 'itemsByCategory',
  get: (categoryId) => ({ get }) => {
    const items = get(itemsState);
    return items.filter(item => item.categoryId === categoryId);
  },
});

function CategoryItems({ categoryId }: { categoryId: string }) {
  const items = useRecoilValue(itemsByCategory(categoryId));
  
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

## Persistence

### Local Storage Persistence

```typescript
import { atom, DefaultValue } from 'recoil';

const localStorageEffect = (key: string) => ({ setSelf, onSet }: any) => {
  const savedValue = localStorage.getItem(key);
  if (savedValue != null) {
    setSelf(JSON.parse(savedValue));
  }

  onSet((newValue: any, _: any, isReset: boolean) => {
    if (isReset) {
      localStorage.removeItem(key);
    } else {
      localStorage.setItem(key, JSON.stringify(newValue));
    }
  });
};

const persistedState = atom({
  key: 'persistedState',
  default: {
    theme: 'light',
    language: 'en',
  },
  effects: [
    localStorageEffect('app_settings'),
  ],
});

function Settings() {
  const [settings, setSettings] = useRecoilState(persistedState);

  return (
    <div>
      <button onClick={() => setSettings({ ...settings, theme: 'dark' })}>
        Set Dark Theme
      </button>
      <p>Current theme: {settings.theme}</p>
    </div>
  );
}
```

### Async Storage Effect

```typescript
import { atom } from 'recoil';

const asyncStorageEffect = (key: string) => ({ setSelf, onSet }: any) => {
  setSelf(
    fetch(`/api/settings/${key}`)
      .then(res => res.json())
      .catch(() => {})
  );

  onSet((newValue: any, _: any, isReset: boolean) => {
    if (isReset) {
      fetch(`/api/settings/${key}`, { method: 'DELETE' });
    } else {
      fetch(`/api/settings/${key}`, {
        method: 'PUT',
        body: JSON.stringify(newValue),
        headers: { 'Content-Type': 'application/json' },
      });
    }
  });
};

const userPreferencesState = atom({
  key: 'userPreferencesState',
  default: { notifications: true },
  effects: [
    asyncStorageEffect('user_preferences'),
  ],
});
```

## useRecoilCallback

### Imperative State Updates

```typescript
import { useRecoilCallback } from 'recoil';

function DataManager() {
  const fetchAndUpdateData = useRecoilCallback(({ snapshot, set }) => async () => {
    // Read current state
    const currentUser = await snapshot.getPromise(userState);
    
    // Perform async operation
    const response = await fetch(`/api/user/${currentUser?.id}/data`);
    const data = await response.json();
    
    // Update multiple atoms
    set(dataState, data);
    set(lastFetchedState, new Date());
  }, []);

  return (
    <button onClick={fetchAndUpdateData}>
      Fetch Data
    </button>
  );
}
```

### Batch Updates

```typescript
import { useRecoilCallback } from 'recoil';

function BatchUpdater() {
  const updateMultiple = useRecoilCallback(({ set }) => (updates: any[]) => {
    updates.forEach(update => {
      set(update.atom, update.value);
    });
  }, []);

  const handleBatchUpdate = () => {
    updateMultiple([
      { atom: countState, value: 10 },
      { atom: textState, value: 'Updated' },
      { atom: userState, value: { id: '1', name: 'John' } },
    ]);
  };

  return (
    <button onClick={handleBatchUpdate}>
      Batch Update
    </button>
  );
}
```

## Testing

```typescript
import { renderRecoilHook } from 'recoil';
import { act } from '@testing-library/react';

describe('Recoil State', () => {
  it('should increment count', () => {
    const { result } = renderRecoilHook(() => useRecoilState(countState));
    
    expect(result.current[0]).toBe(0);
    
    act(() => {
      result.current[1](1);
    });
    
    expect(result.current[0]).toBe(1);
  });
});

// Snapshot testing
import { snapshot_UNSTABLE } from 'recoil';

test('selector returns correct value', () => {
  const snapshot = snapshot_UNSTABLE();
  snapshot.set(fahrenheitState, 32);
  
  const celsius = snapshot.getLoadable(celsiusState).getValue();
  expect(celsius).toBe(0);
});
```

## Common Mistakes

1. **Missing RecoilRoot**: Must wrap app in RecoilRoot
2. **Non-unique keys**: Each atom/selector needs a unique key
3. **Not handling async errors**: Use ErrorBoundary with async selectors
4. **Missing Suspense**: Async selectors require Suspense
5. **Complex default values**: Keep defaults simple and serializable

## Best Practices

1. **Use descriptive keys**: Make atom/selector keys meaningful
2. **Organize by feature**: Group related atoms together
3. **Use TypeScript**: Full type safety for state
4. **Leverage selectors**: Compute derived state
5. **Handle errors**: Use ErrorBoundary for async
6. **Use families wisely**: For dynamic, parameterized state
7. **Persist carefully**: Only persist what's necessary

## When to Use Recoil

### Good Use Cases
- Complex derived state
- Async data dependencies
- Fine-grained reactivity
- React Concurrent Mode apps
- Large-scale applications

### Not Ideal For
- Simple local state
- Small apps
- When bundle size is critical

## Interview Questions

1. **What are atoms in Recoil?**
   - Minimal units of state that components can subscribe to

2. **How do selectors differ from atoms?**
   - Selectors compute derived state from atoms or other selectors

3. **How does Recoil handle async operations?**
   - Built-in async support with Suspense and ErrorBoundary

4. **What are atom families used for?**
   - Creating atoms dynamically based on parameters

5. **How do you persist Recoil state?**
   - Using atom effects with localStorage or other storage

## Key Takeaways

- Recoil provides atom-based state management
- Selectors enable derived and computed state
- First-class async support with Suspense
- Atom and selector families for dynamic state
- Effects system for persistence and side effects
- Fine-grained subscriptions for performance
- Concurrent Mode ready
- Developed and used by Facebook

## Resources

- [Official Documentation](https://recoiljs.org/)
- [API Reference](https://recoiljs.org/docs/api-reference/core/atom)
- [Best Practices](https://recoiljs.org/docs/guides/best-practices)
- [GitHub Repository](https://github.com/facebookexperimental/Recoil)
