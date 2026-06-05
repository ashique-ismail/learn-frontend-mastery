# Redux Toolkit - The Official Redux Toolset

## The Idea

**In plain English:** Redux Toolkit is a set of helper tools that makes it easier to manage all the shared data (called "state") in your app — things like who is logged in, what's in a shopping cart, or which posts are loaded — so every part of the app can read and update that data in a safe, organized way.

**Real-world analogy:** Think of a busy school office with a single whiteboard that every teacher can see. A secretary controls what gets written on it — teachers submit a request slip saying what should change, the secretary updates the board, and everyone sees the new info. Redux Toolkit is the rulebook that makes this whole system run smoothly.

- The whiteboard = the Redux store (the single place where all shared app data lives)
- A teacher's request slip = an action (a message describing what change is needed)
- The secretary who updates the board = the reducer (the function that applies the change)
- The rulebook for how requests are handled = Redux Toolkit (the toolset that defines the rules and simplifies the process)

---

## Overview

Redux Toolkit (RTK) is the official, opinionated, batteries-included toolset for efficient Redux development. It provides utilities to simplify Redux code, including store setup, creating reducers and actions, and immutable update logic using Immer. RTK Query is included for data fetching and caching.

### Key Features

- **Simplified store setup**: `configureStore()` with good defaults
- **createSlice**: Combines reducers and actions
- **Immer integration**: Write "mutating" immutable updates
- **RTK Query**: Powerful data fetching and caching
- **DevTools**: Built-in Redux DevTools integration
- **TypeScript**: Excellent TypeScript support
- **EntityAdapter**: Normalize state shape

## Installation and Setup

```bash
# npm
npm install @reduxjs/toolkit react-redux

# yarn
yarn add @reduxjs/toolkit react-redux

# pnpm
pnpm add @reduxjs/toolkit react-redux
```

## Store Configuration

### Basic Store Setup

```typescript
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './features/counter/counterSlice';
import userReducer from './features/user/userSlice';

export const store = configureStore({
  reducer: {
    counter: counterReducer,
    user: userReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Provider Setup

```typescript
import React from 'react';
import { Provider } from 'react-redux';
import { store } from './app/store';
import App from './App';

function Root() {
  return (
    <Provider store={store}>
      <App />
    </Provider>
  );
}

export default Root;
```

### Typed Hooks

```typescript
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

## createSlice

### Basic Slice

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CounterState {
  value: number;
  status: 'idle' | 'loading';
}

const initialState: CounterState = {
  value: 0,
  status: 'idle',
};

const counterSlice = createSlice({
  name: 'counter',
  initialState,
  reducers: {
    increment: (state) => {
      // Immer allows "mutating" code
      state.value += 1;
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action: PayloadAction<number>) => {
      state.value += action.payload;
    },
    reset: (state) => {
      state.value = 0;
    },
  },
});

export const { increment, decrement, incrementByAmount, reset } = counterSlice.actions;
export default counterSlice.reducer;

// Usage in component
import { useAppDispatch, useAppSelector } from '../../app/hooks';
import { increment, decrement, reset } from './counterSlice';

function Counter() {
  const count = useAppSelector((state) => state.counter.value);
  const dispatch = useAppDispatch();

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
      <button onClick={() => dispatch(reset())}>Reset</button>
    </div>
  );
}
```

### Complex Slice with Nested State

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
}

interface TodoState {
  items: Todo[];
  filter: 'all' | 'active' | 'completed';
  loading: boolean;
  error: string | null;
}

const initialState: TodoState = {
  items: [],
  filter: 'all',
  loading: false,
  error: null,
};

const todoSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    addTodo: (state, action: PayloadAction<{ text: string; priority: Todo['priority'] }>) => {
      state.items.push({
        id: crypto.randomUUID(),
        text: action.payload.text,
        completed: false,
        priority: action.payload.priority,
      });
    },
    toggleTodo: (state, action: PayloadAction<string>) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed;
      }
    },
    deleteTodo: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(t => t.id !== action.payload);
    },
    updateTodo: (state, action: PayloadAction<{ id: string; text: string }>) => {
      const todo = state.items.find(t => t.id === action.payload.id);
      if (todo) {
        todo.text = action.payload.text;
      }
    },
    setFilter: (state, action: PayloadAction<TodoState['filter']>) => {
      state.filter = action.payload;
    },
    clearCompleted: (state) => {
      state.items = state.items.filter(t => !t.completed);
    },
  },
});

export const { addTodo, toggleTodo, deleteTodo, updateTodo, setFilter, clearCompleted } = todoSlice.actions;
export default todoSlice.reducer;
```

## createAsyncThunk

### Basic Async Operations

```typescript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserState {
  user: User | null;
  loading: boolean;
  error: string | null;
}

const initialState: UserState = {
  user: null,
  loading: false,
  error: null,
};

// Async thunk
export const fetchUser = createAsyncThunk(
  'user/fetchUser',
  async (userId: string, { rejectWithValue }) => {
    try {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error('Failed to fetch user');
      return (await response.json()) as User;
    } catch (error) {
      return rejectWithValue((error as Error).message);
    }
  }
);

export const updateUser = createAsyncThunk(
  'user/updateUser',
  async (userData: Partial<User> & { id: string }, { rejectWithValue }) => {
    try {
      const response = await fetch(`/api/users/${userData.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData),
      });
      if (!response.ok) throw new Error('Failed to update user');
      return (await response.json()) as User;
    } catch (error) {
      return rejectWithValue((error as Error).message);
    }
  }
);

const userSlice = createSlice({
  name: 'user',
  initialState,
  reducers: {
    logout: (state) => {
      state.user = null;
      state.error = null;
    },
  },
  extraReducers: (builder) => {
    builder
      // Fetch user cases
      .addCase(fetchUser.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false;
        state.user = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string;
      })
      // Update user cases
      .addCase(updateUser.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(updateUser.fulfilled, (state, action) => {
        state.loading = false;
        state.user = action.payload;
      })
      .addCase(updateUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string;
      });
  },
});

export const { logout } = userSlice.actions;
export default userSlice.reducer;

// Usage in component
function UserProfile() {
  const { user, loading, error } = useAppSelector(state => state.user);
  const dispatch = useAppDispatch();

  React.useEffect(() => {
    dispatch(fetchUser('123'));
  }, [dispatch]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>No user</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => dispatch(updateUser({ id: user.id, name: 'New Name' }))}>
        Update Name
      </button>
    </div>
  );
}
```

### Async Thunk with Conditional Dispatch

```typescript
import { createAsyncThunk } from '@reduxjs/toolkit';
import type { RootState } from '../store';

export const fetchUserIfNeeded = createAsyncThunk(
  'user/fetchUserIfNeeded',
  async (userId: string, { getState, requestId }) => {
    const state = getState() as RootState;
    const { currentRequestId, loading } = state.user;
    
    // Don't fetch if already loading
    if (loading && currentRequestId !== requestId) {
      return;
    }
    
    const response = await fetch(`/api/users/${userId}`);
    return response.json();
  },
  {
    condition: (userId, { getState }) => {
      const state = getState() as RootState;
      const { user, loading } = state.user;
      
      // Don't fetch if user already loaded
      if (user?.id === userId || loading) {
        return false;
      }
      return true;
    },
  }
);
```

## Entity Adapter

### Normalized State Management

```typescript
import { createSlice, createEntityAdapter, createAsyncThunk } from '@reduxjs/toolkit';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
}

// Create entity adapter
const productsAdapter = createEntityAdapter<Product>({
  selectId: (product) => product.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name),
});

interface ProductState {
  loading: boolean;
  error: string | null;
}

const initialState = productsAdapter.getInitialState<ProductState>({
  loading: false,
  error: null,
});

export const fetchProducts = createAsyncThunk(
  'products/fetchProducts',
  async () => {
    const response = await fetch('/api/products');
    return (await response.json()) as Product[];
  }
);

const productSlice = createSlice({
  name: 'products',
  initialState,
  reducers: {
    productAdded: productsAdapter.addOne,
    productUpdated: productsAdapter.updateOne,
    productRemoved: productsAdapter.removeOne,
    productsReceived: productsAdapter.setAll,
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.loading = false;
        productsAdapter.setAll(state, action.payload);
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch';
      });
  },
});

export const { productAdded, productUpdated, productRemoved, productsReceived } = productSlice.actions;
export default productSlice.reducer;

// Selectors
export const {
  selectAll: selectAllProducts,
  selectById: selectProductById,
  selectIds: selectProductIds,
} = productsAdapter.getSelectors((state: RootState) => state.products);

// Custom selectors
export const selectProductsByCategory = (state: RootState, category: string) =>
  selectAllProducts(state).filter(product => product.category === category);

// Usage
function ProductList() {
  const products = useAppSelector(selectAllProducts);
  const dispatch = useAppDispatch();

  React.useEffect(() => {
    dispatch(fetchProducts());
  }, [dispatch]);

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.name} - ${product.price}
        </li>
      ))}
    </ul>
  );
}
```

## RTK Query

### API Setup

```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

interface Post {
  id: string;
  title: string;
  body: string;
  userId: string;
}

interface User {
  id: string;
  name: string;
  email: string;
}

export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Post', 'User'],
  endpoints: (builder) => ({
    // Queries
    getPosts: builder.query<Post[], void>({
      query: () => '/posts',
      providesTags: ['Post'],
    }),
    getPost: builder.query<Post, string>({
      query: (id) => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Post', id }],
    }),
    getUser: builder.query<User, string>({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }],
    }),
    // Mutations
    addPost: builder.mutation<Post, Omit<Post, 'id'>>({
      query: (body) => ({
        url: '/posts',
        method: 'POST',
        body,
      }),
      invalidatesTags: ['Post'],
    }),
    updatePost: builder.mutation<Post, Partial<Post> & { id: string }>({
      query: ({ id, ...patch }) => ({
        url: `/posts/${id}`,
        method: 'PATCH',
        body: patch,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'Post', id }],
    }),
    deletePost: builder.mutation<{ success: boolean }, string>({
      query: (id) => ({
        url: `/posts/${id}`,
        method: 'DELETE',
      }),
      invalidatesTags: (result, error, id) => [{ type: 'Post', id }],
    }),
  }),
});

export const {
  useGetPostsQuery,
  useGetPostQuery,
  useGetUserQuery,
  useAddPostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = api;
```

### Store with RTK Query

```typescript
import { configureStore } from '@reduxjs/toolkit';
import { api } from './services/api';

export const store = configureStore({
  reducer: {
    [api.reducerPath]: api.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(api.middleware),
});
```

### Using RTK Query

```typescript
import { useGetPostsQuery, useAddPostMutation, useDeletePostMutation } from './services/api';

function PostList() {
  const { data: posts, isLoading, isError, error, refetch } = useGetPostsQuery();
  const [addPost, { isLoading: isAdding }] = useAddPostMutation();
  const [deletePost] = useDeletePostMutation();

  const handleAddPost = async () => {
    try {
      await addPost({
        title: 'New Post',
        body: 'Content',
        userId: '1',
      }).unwrap();
    } catch (error) {
      console.error('Failed to add post:', error);
    }
  };

  if (isLoading) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.toString()}</div>;

  return (
    <div>
      <button onClick={handleAddPost} disabled={isAdding}>
        Add Post
      </button>
      <button onClick={refetch}>Refresh</button>
      <ul>
        {posts?.map(post => (
          <li key={post.id}>
            {post.title}
            <button onClick={() => deletePost(post.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Optimistic Updates with RTK Query

```typescript
const api = createApi({
  // ... base config
  endpoints: (builder) => ({
    updatePost: builder.mutation<Post, Partial<Post> & { id: string }>({
      query: ({ id, ...patch }) => ({
        url: `/posts/${id}`,
        method: 'PATCH',
        body: patch,
      }),
      async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
        // Optimistic update
        const patchResult = dispatch(
          api.util.updateQueryData('getPost', id, (draft) => {
            Object.assign(draft, patch);
          })
        );
        
        try {
          await queryFulfilled;
        } catch {
          // Undo on error
          patchResult.undo();
        }
      },
    }),
  }),
});
```

### Conditional Fetching

```typescript
function PostDetail({ postId }: { postId: string | null }) {
  const { data, isLoading } = useGetPostQuery(postId!, {
    skip: !postId, // Skip if no postId
  });

  if (!postId) return <div>Select a post</div>;
  if (isLoading) return <div>Loading...</div>;

  return <div>{data?.title}</div>;
}
```

### Polling and Refetching

```typescript
function LivePosts() {
  const { data, refetch } = useGetPostsQuery(undefined, {
    pollingInterval: 5000, // Poll every 5 seconds
    refetchOnMountOrArgChange: true,
    refetchOnFocus: true,
    refetchOnReconnect: true,
  });

  return (
    <div>
      <button onClick={refetch}>Refresh Now</button>
      <ul>
        {data?.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Middleware

### Custom Middleware

```typescript
import { Middleware } from '@reduxjs/toolkit';

const loggerMiddleware: Middleware = (store) => (next) => (action) => {
  console.log('Dispatching:', action);
  const result = next(action);
  console.log('Next state:', store.getState());
  return result;
};

const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(loggerMiddleware),
});
```

## Selectors with createSelector

```typescript
import { createSelector } from '@reduxjs/toolkit';

// Memoized selector
const selectTodos = (state: RootState) => state.todos.items;
const selectFilter = (state: RootState) => state.todos.filter;

export const selectFilteredTodos = createSelector(
  [selectTodos, selectFilter],
  (todos, filter) => {
    switch (filter) {
      case 'active':
        return todos.filter(t => !t.completed);
      case 'completed':
        return todos.filter(t => t.completed);
      default:
        return todos;
    }
  }
);

export const selectTodoStats = createSelector(
  [selectTodos],
  (todos) => ({
    total: todos.length,
    completed: todos.filter(t => t.completed).length,
    active: todos.filter(t => !t.completed).length,
  })
);
```

## Common Mistakes

1. **Not using TypeScript types**: Leverage full type safety
2. **Mutating outside Immer**: Only in reducers, not in thunks
3. **Missing error handling**: Always handle async errors
4. **Overusing RTK Query**: Not everything needs caching
5. **Not normalizing data**: Use EntityAdapter for collections

## Best Practices

1. **Use TypeScript**: Full type inference and safety
2. **Normalize state**: EntityAdapter for relational data
3. **Use RTK Query**: For server state and caching
4. **Keep slices focused**: Single responsibility
5. **Memoize selectors**: Use createSelector for derived state
6. **Handle loading states**: Provide feedback to users
7. **Use DevTools**: Debug with Redux DevTools

## When to Use Redux Toolkit

### Good Use Cases

- Large applications with complex state
- Need for time-travel debugging
- Team familiar with Redux patterns
- Server state caching with RTK Query
- Normalized data structures

### Not Ideal For

- Simple applications
- Local component state
- When bundle size is critical

## Interview Questions

1. **What problem does Redux Toolkit solve?**
   - Simplifies Redux boilerplate and provides good defaults

2. **How does createSlice work?**
   - Combines reducers and actions with Immer for immutable updates

3. **What is RTK Query?**
   - Data fetching and caching solution built on Redux Toolkit

4. **How does EntityAdapter help?**
   - Provides utilities for normalizing and managing collections

5. **What is the purpose of createAsyncThunk?**
   - Handles async logic with pending/fulfilled/rejected states

## Key Takeaways

- Redux Toolkit simplifies Redux development
- createSlice combines reducers and actions
- Immer enables "mutating" immutable updates
- RTK Query provides powerful data fetching
- EntityAdapter normalizes data structures
- Excellent TypeScript support
- Built-in DevTools integration
- Official recommended approach for Redux

## Resources

- [Official Documentation](https://redux-toolkit.js.org/)
- [RTK Query](https://redux-toolkit.js.org/rtk-query/overview)
- [Usage Guide](https://redux-toolkit.js.org/usage/usage-guide)
- [API Reference](https://redux-toolkit.js.org/api/configureStore)
- [GitHub Repository](https://github.com/reduxjs/redux-toolkit)
