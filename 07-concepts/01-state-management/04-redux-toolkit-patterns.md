# Redux Toolkit Patterns

## Overview

Redux Toolkit (RTK) is the official, opinionated, batteries-included toolset for efficient Redux development. It was created to address three common concerns about Redux: too much boilerplate, too many packages to configure, and too much complexity. RTK simplifies Redux development while following best practices, and includes powerful utilities like `createSlice`, `createAsyncThunk`, and RTK Query for data fetching.

## The Problem Redux Toolkit Solves

### Traditional Redux Boilerplate

```typescript
// Traditional Redux requires lots of boilerplate

// 1. Action types
const ADD_TODO = 'todos/ADD_TODO';
const TOGGLE_TODO = 'todos/TOGGLE_TODO';
const DELETE_TODO = 'todos/DELETE_TODO';

// 2. Action creators
const addTodo = (text) => ({
  type: ADD_TODO,
  payload: { id: Date.now(), text }
});

const toggleTodo = (id) => ({
  type: TOGGLE_TODO,
  payload: { id }
});

// 3. Reducer with manual immutable updates
function todosReducer(state = [], action) {
  switch (action.type) {
    case ADD_TODO:
      return [...state, action.payload];
    case TOGGLE_TODO:
      return state.map(todo =>
        todo.id === action.payload.id
          ? { ...todo, completed: !todo.completed }
          : todo
      );
    case DELETE_TODO:
      return state.filter(todo => todo.id !== action.payload.id);
    default:
      return state;
  }
}

// 4. Store configuration
import { createStore, combineReducers, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import { composeWithDevTools } from 'redux-devtools-extension';

const rootReducer = combineReducers({
  todos: todosReducer
});

const store = createStore(
  rootReducer,
  composeWithDevTools(applyMiddleware(thunk))
);
```

### Redux Toolkit Simplification

```typescript
// Same functionality with Redux Toolkit
import { createSlice, configureStore } from '@reduxjs/toolkit';

// 1. Create slice (combines types, actions, reducer)
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: {
      reducer: (state, action) => {
        state.push(action.payload); // Direct mutation with Immer!
      },
      prepare: (text) => ({
        payload: { id: Date.now(), text, completed: false }
      })
    },
    toggleTodo: (state, action) => {
      const todo = state.find(t => t.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed; // Direct mutation!
      }
    },
    deleteTodo: (state, action) => {
      return state.filter(todo => todo.id !== action.payload);
    }
  }
});

// 2. Export auto-generated actions
export const { addTodo, toggleTodo, deleteTodo } = todosSlice.actions;

// 3. Configure store with defaults
const store = configureStore({
  reducer: {
    todos: todosSlice.reducer
  }
  // Redux DevTools, thunk, and serializability checks included by default!
});
```

**Result**: ~50 lines of code → ~25 lines, with better defaults!

## Core RTK Patterns

### 1. createSlice: The Foundation

`createSlice` is the core API that generates action creators and reducers:

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: number;
}

interface TodosState {
  items: Todo[];
  filter: 'all' | 'active' | 'completed';
  loading: boolean;
}

const initialState: TodosState = {
  items: [],
  filter: 'all',
  loading: false
};

const todosSlice = createSlice({
  name: 'todos', // Used for action types: 'todos/addTodo'
  initialState,
  reducers: {
    // Simple reducer: action payload is the todo text
    addTodo: {
      reducer: (state, action: PayloadAction<Todo>) => {
        state.items.push(action.payload);
      },
      // prepare callback creates the payload
      prepare: (text: string) => ({
        payload: {
          id: crypto.randomUUID(),
          text,
          completed: false,
          createdAt: Date.now()
        }
      })
    },
    
    // Reducer without prepare
    toggleTodo: (state, action: PayloadAction<string>) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed;
      }
    },
    
    // Multiple changes in one reducer
    deleteTodo: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(t => t.id !== action.payload);
    },
    
    // Update multiple fields
    updateTodo: (
      state,
      action: PayloadAction<{ id: string; text: string }>
    ) => {
      const todo = state.items.find(t => t.id === action.payload.id);
      if (todo) {
        todo.text = action.payload.text;
      }
    },
    
    // Change non-array state
    setFilter: (
      state,
      action: PayloadAction<'all' | 'active' | 'completed'>
    ) => {
      state.filter = action.payload;
    },
    
    // Clear all
    clearCompleted: (state) => {
      state.items = state.items.filter(t => !t.completed);
    }
  }
});

// Auto-generated action creators
export const {
  addTodo,
  toggleTodo,
  deleteTodo,
  updateTodo,
  setFilter,
  clearCompleted
} = todosSlice.actions;

// Export reducer
export default todosSlice.reducer;

// Usage in components
dispatch(addTodo('Buy milk')); // Calls prepare(), then reducer
dispatch(toggleTodo('todo-id-123'));
dispatch(setFilter('active'));
```

### 2. Immer Integration: Write Mutable Code

RTK uses Immer under the hood, allowing you to write "mutating" code:

```typescript
const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [], count: 0 },
  reducers: {
    // Option 1: "Mutate" the draft state (Immer)
    addTodo: (state, action) => {
      state.items.push(action.payload); // Looks like mutation
      state.count += 1;
      // Immer converts this to immutable update
    },
    
    // Option 2: Return new state (traditional)
    addTodoImmutable: (state, action) => {
      return {
        items: [...state.items, action.payload],
        count: state.count + 1
      };
    },
    
    // ⚠️ DON'T mix both patterns!
    badMixedPattern: (state, action) => {
      state.count += 1; // Mutation
      return { ...state, items: [...state.items] }; // ❌ Don't do this!
    },
    
    // Complex nested updates made easy
    updateNestedData: (state, action) => {
      // Without Immer, this would be:
      // return {
      //   ...state,
      //   users: {
      //     ...state.users,
      //     byId: {
      //       ...state.users.byId,
      //       [action.id]: {
      //         ...state.users.byId[action.id],
      //         profile: {
      //           ...state.users.byId[action.id].profile,
      //           name: action.name
      //         }
      //       }
      //     }
      //   }
      // };
      
      // With Immer:
      state.users.byId[action.id].profile.name = action.name;
    }
  }
});
```

### 3. createAsyncThunk: Handling Async Logic

`createAsyncThunk` generates action creators for async operations:

```typescript
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Define async thunk
export const fetchTodos = createAsyncThunk(
  'todos/fetchTodos', // Action type prefix
  async (userId: string, thunkAPI) => {
    // thunkAPI provides: dispatch, getState, extra, etc.
    const response = await fetch(`/api/users/${userId}/todos`);
    if (!response.ok) {
      // Rejection with error message
      return thunkAPI.rejectWithValue('Failed to fetch todos');
    }
    const data = await response.json();
    return data; // This becomes action.payload on success
  }
);

// Create async thunk with arguments
export const createTodo = createAsyncThunk(
  'todos/createTodo',
  async ({ text, userId }: { text: string; userId: string }, thunkAPI) => {
    try {
      const response = await fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text, userId })
      });
      return await response.json();
    } catch (error) {
      return thunkAPI.rejectWithValue(error.message);
    }
  }
);

// Async thunk with conditional execution
export const updateTodo = createAsyncThunk(
  'todos/updateTodo',
  async (todo: Todo, thunkAPI) => {
    const state = thunkAPI.getState() as RootState;
    
    // Conditional logic using state
    if (!state.user.isAuthenticated) {
      return thunkAPI.rejectWithValue('Not authenticated');
    }
    
    const response = await fetch(`/api/todos/${todo.id}`, {
      method: 'PUT',
      body: JSON.stringify(todo)
    });
    return await response.json();
  },
  {
    // Condition to skip execution
    condition: (todo, { getState }) => {
      const state = getState() as RootState;
      const existingTodo = state.todos.items.find(t => t.id === todo.id);
      // Skip if todo hasn't changed
      return existingTodo?.text !== todo.text;
    }
  }
);

// Handle in slice with extraReducers
const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [] as Todo[],
    loading: false,
    error: null as string | null,
    lastFetch: null as number | null
  },
  reducers: {
    // Regular synchronous actions
    addTodo: (state, action) => {
      state.items.push(action.payload);
    }
  },
  extraReducers: (builder) => {
    // Handle fetchTodos lifecycle
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
        state.lastFetch = Date.now();
      })
      .addCase(fetchTodos.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string || action.error.message;
      })
      
      // Handle createTodo lifecycle
      .addCase(createTodo.fulfilled, (state, action) => {
        state.items.push(action.payload);
      })
      
      // Handle updateTodo
      .addCase(updateTodo.fulfilled, (state, action) => {
        const index = state.items.findIndex(t => t.id === action.payload.id);
        if (index !== -1) {
          state.items[index] = action.payload;
        }
      });
  }
});

// Usage in components
function TodoList() {
  const dispatch = useDispatch();
  const { items, loading, error } = useSelector((state: RootState) => state.todos);
  
  useEffect(() => {
    dispatch(fetchTodos('user-123'));
  }, [dispatch]);
  
  const handleCreate = async (text: string) => {
    const result = await dispatch(createTodo({ text, userId: 'user-123' }));
    
    if (createTodo.fulfilled.match(result)) {
      console.log('Todo created:', result.payload);
    } else {
      console.error('Failed:', result.payload);
    }
  };
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <ul>
      {items.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

### 4. configureStore: Better Defaults

`configureStore` sets up the Redux store with good defaults:

```typescript
import { configureStore } from '@reduxjs/toolkit';
import todosReducer from './features/todos/todosSlice';
import userReducer from './features/user/userSlice';
import logger from 'redux-logger';

const store = configureStore({
  reducer: {
    todos: todosReducer,
    user: userReducer
  },
  
  // Automatically includes:
  // - redux-thunk middleware
  // - Redux DevTools Extension
  // - Serializability check middleware (dev only)
  // - Immutability check middleware (dev only)
  
  // Add custom middleware
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      // Configure defaults
      serializableCheck: {
        // Ignore these action types
        ignoredActions: ['todos/uploadFile'],
        // Ignore these paths in state
        ignoredPaths: ['todos.fileUpload']
      },
      thunk: {
        // Pass extra argument to thunks
        extraArgument: { api: myApiService }
      }
    }).concat(logger), // Add custom middleware
  
  // Development options
  devTools: process.env.NODE_ENV !== 'production',
  
  // Preloaded state (for SSR/hydration)
  preloadedState: {
    todos: {
      items: [],
      loading: false,
      error: null
    }
  },
  
  // Enhancers (rarely needed)
  enhancers: (getDefaultEnhancers) =>
    getDefaultEnhancers().concat(customEnhancer)
});

// Export types
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Export store
export default store;

// Type-safe hooks
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### 5. createEntityAdapter: Normalized State

`createEntityAdapter` provides prebuilt reducers for normalized state:

```typescript
import { createEntityAdapter, createSlice, EntityState } from '@reduxjs/toolkit';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Create adapter
const todosAdapter = createEntityAdapter<Todo>({
  // Optional: Custom ID selector
  selectId: (todo) => todo.id,
  
  // Optional: Sort comparer
  sortComparer: (a, b) => b.createdAt - a.createdAt
});

// Get initial state
// EntityState<Todo> = { ids: string[], entities: { [id: string]: Todo } }
const initialState = todosAdapter.getInitialState({
  // Additional state
  loading: false,
  error: null as string | null
});

const todosSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    // Use adapter methods for CRUD operations
    todoAdded: todosAdapter.addOne,
    todosReceived: todosAdapter.setAll,
    todoUpdated: todosAdapter.updateOne,
    todoRemoved: todosAdapter.removeOne,
    
    // Custom reducers with adapter methods
    toggleTodo: (state, action) => {
      const todo = state.entities[action.payload];
      if (todo) {
        todosAdapter.updateOne(state, {
          id: action.payload,
          changes: { completed: !todo.completed }
        });
      }
    }
  }
});

// Adapter provides selectors
export const {
  selectAll: selectAllTodos,
  selectById: selectTodoById,
  selectIds: selectTodoIds,
  selectEntities: selectTodoEntities,
  selectTotal: selectTodosTotal
} = todosAdapter.getSelectors((state: RootState) => state.todos);

// Usage
function TodoList() {
  // Select all todos (sorted by createdAt)
  const todos = useAppSelector(selectAllTodos);
  
  // Select specific todo
  const todo = useAppSelector(state => selectTodoById(state, 'todo-123'));
  
  // Select total count
  const total = useAppSelector(selectTodosTotal);
  
  const dispatch = useAppDispatch();
  
  // Add one todo
  dispatch(todoAdded({ id: '1', text: 'Buy milk', completed: false }));
  
  // Set all todos
  dispatch(todosReceived([
    { id: '1', text: 'Todo 1', completed: false },
    { id: '2', text: 'Todo 2', completed: true }
  ]));
  
  // Update one todo
  dispatch(todoUpdated({ id: '1', changes: { text: 'Updated text' } }));
  
  // Remove one todo
  dispatch(todoRemoved('1'));
  
  return <ul>{todos.map(todo => <li key={todo.id}>{todo.text}</li>)}</ul>;
}
```

Adapter CRUD methods:
- `addOne`, `addMany`: Add entities
- `setOne`, `setMany`, `setAll`: Replace entities
- `removeOne`, `removeMany`, `removeAll`: Delete entities
- `updateOne`, `updateMany`: Update entities
- `upsertOne`, `upsertMany`: Add or update

### 6. RTK Query: Data Fetching Solution

RTK Query is a powerful data fetching and caching tool built into RTK:

```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  userId: string;
}

// Define API slice
export const todosApi = createApi({
  reducerPath: 'todosApi',
  baseQuery: fetchBaseQuery({
    baseUrl: '/api',
    // Add auth headers
    prepareHeaders: (headers, { getState }) => {
      const token = (getState() as RootState).auth.token;
      if (token) {
        headers.set('authorization', `Bearer ${token}`);
      }
      return headers;
    }
  }),
  
  // Tag types for cache invalidation
  tagTypes: ['Todo'],
  
  endpoints: (builder) => ({
    // Query: GET request
    getTodos: builder.query<Todo[], string>({
      query: (userId) => `/users/${userId}/todos`,
      // Tags for cache invalidation
      providesTags: (result) =>
        result
          ? [
              ...result.map(({ id }) => ({ type: 'Todo' as const, id })),
              { type: 'Todo', id: 'LIST' }
            ]
          : [{ type: 'Todo', id: 'LIST' }]
    }),
    
    // Query with params
    getTodo: builder.query<Todo, string>({
      query: (id) => `/todos/${id}`,
      providesTags: (result, error, id) => [{ type: 'Todo', id }]
    }),
    
    // Mutation: POST request
    createTodo: builder.mutation<Todo, Partial<Todo>>({
      query: (body) => ({
        url: '/todos',
        method: 'POST',
        body
      }),
      // Invalidate cache to refetch
      invalidatesTags: [{ type: 'Todo', id: 'LIST' }]
    }),
    
    // Mutation: PUT request
    updateTodo: builder.mutation<Todo, Partial<Todo>>({
      query: ({ id, ...patch }) => ({
        url: `/todos/${id}`,
        method: 'PUT',
        body: patch
      }),
      // Optimistic update
      async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
        // Optimistically update cache
        const patchResult = dispatch(
          todosApi.util.updateQueryData('getTodo', id!, (draft) => {
            Object.assign(draft, patch);
          })
        );
        
        try {
          await queryFulfilled;
        } catch {
          // Rollback on error
          patchResult.undo();
        }
      },
      invalidatesTags: (result, error, { id }) => [{ type: 'Todo', id }]
    }),
    
    // Mutation: DELETE request
    deleteTodo: builder.mutation<void, string>({
      query: (id) => ({
        url: `/todos/${id}`,
        method: 'DELETE'
      }),
      invalidatesTags: (result, error, id) => [{ type: 'Todo', id }]
    })
  })
});

// Auto-generated hooks
export const {
  useGetTodosQuery,
  useGetTodoQuery,
  useCreateTodoMutation,
  useUpdateTodoMutation,
  useDeleteTodoMutation
} = todosApi;

// Add to store
const store = configureStore({
  reducer: {
    [todosApi.reducerPath]: todosApi.reducer,
    // other reducers
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(todosApi.middleware)
});

// Usage in components
function TodoList({ userId }: { userId: string }) {
  // Query hook - automatically fetches and caches
  const {
    data: todos,
    isLoading,
    isError,
    error,
    refetch
  } = useGetTodosQuery(userId, {
    // Polling interval
    pollingInterval: 10000,
    // Skip query conditionally
    skip: !userId,
    // Refetch on mount/focus
    refetchOnMountOrArgChange: true
  });
  
  // Mutation hooks
  const [createTodo, { isLoading: isCreating }] = useCreateTodoMutation();
  const [updateTodo] = useUpdateTodoMutation();
  const [deleteTodo] = useDeleteTodoMutation();
  
  const handleCreate = async (text: string) => {
    try {
      await createTodo({ text, userId }).unwrap();
      // Success - cache automatically updated
    } catch (error) {
      console.error('Failed to create todo:', error);
    }
  };
  
  const handleToggle = (id: string, completed: boolean) => {
    updateTodo({ id, completed: !completed });
  };
  
  const handleDelete = (id: string) => {
    if (confirm('Delete todo?')) {
      deleteTodo(id);
    }
  };
  
  if (isLoading) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.toString()}</div>;
  
  return (
    <div>
      <button onClick={() => handleCreate('New todo')}>
        Add Todo {isCreating && '...'}
      </button>
      
      <ul>
        {todos?.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => handleToggle(todo.id, todo.completed)}
            />
            <span>{todo.text}</span>
            <button onClick={() => handleDelete(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
      
      <button onClick={refetch}>Refresh</button>
    </div>
  );
}
```

RTK Query features:
- Automatic caching and deduplication
- Optimistic updates
- Polling and refetching
- Cache invalidation with tags
- TypeScript support
- DevTools integration
- Streaming updates (SSE, WebSockets)

### 7. createSelector: Memoized Selectors

```typescript
import { createSelector } from '@reduxjs/toolkit';

// Simple selectors
const selectTodos = (state: RootState) => state.todos.items;
const selectFilter = (state: RootState) => state.todos.filter;

// Memoized selector - only recalculates when inputs change
export const selectFilteredTodos = createSelector(
  [selectTodos, selectFilter],
  (todos, filter) => {
    console.log('Filtering todos...'); // Only logs when todos or filter change
    
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

// Selector with parameters
export const selectTodoById = createSelector(
  [selectTodos, (state: RootState, todoId: string) => todoId],
  (todos, todoId) => todos.find(t => t.id === todoId)
);

// Composed selectors
export const selectCompletedCount = createSelector(
  [selectTodos],
  (todos) => todos.filter(t => t.completed).length
);

export const selectActiveCount = createSelector(
  [selectTodos],
  (todos) => todos.filter(t => !t.completed).length
);

export const selectTodoStats = createSelector(
  [selectTodos, selectCompletedCount, selectActiveCount],
  (todos, completed, active) => ({
    total: todos.length,
    completed,
    active,
    percentComplete: todos.length ? (completed / todos.length) * 100 : 0
  })
);

// Usage
function TodoStats() {
  const stats = useAppSelector(selectTodoStats);
  
  return (
    <div>
      <p>Total: {stats.total}</p>
      <p>Active: {stats.active}</p>
      <p>Completed: {stats.completed}</p>
      <p>Progress: {stats.percentComplete.toFixed(0)}%</p>
    </div>
  );
}
```

## Common Patterns

### 1. Loading States

```typescript
interface LoadingState {
  idle: 'idle';
  pending: 'pending';
  succeeded: 'succeeded';
  failed: 'failed';
}

const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [],
    status: 'idle' as LoadingState,
    error: null as string | null
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.status = 'pending';
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message || 'Failed to fetch';
      });
  }
});
```

### 2. Optimistic Updates

```typescript
const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [] },
  reducers: {
    // Optimistically add todo
    todoAddedOptimistic: (state, action) => {
      state.items.push({ ...action.payload, _optimistic: true });
    },
    // Confirm when server responds
    todoAddedConfirmed: (state, action) => {
      const index = state.items.findIndex(t => t._tempId === action.payload.tempId);
      if (index !== -1) {
        state.items[index] = action.payload.todo;
      }
    },
    // Rollback on error
    todoAddedRollback: (state, action) => {
      state.items = state.items.filter(t => t._tempId !== action.payload);
    }
  }
});

// Thunk with optimistic update
export const createTodoOptimistic = createAsyncThunk(
  'todos/createOptimistic',
  async (text: string, { dispatch }) => {
    const tempId = `temp_${Date.now()}`;
    
    // Optimistic update
    dispatch(todoAddedOptimistic({ _tempId: tempId, text, completed: false }));
    
    try {
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ text })
      });
      const todo = await response.json();
      
      // Confirm with real data
      dispatch(todoAddedConfirmed({ tempId, todo }));
      return todo;
    } catch (error) {
      // Rollback on error
      dispatch(todoAddedRollback(tempId));
      throw error;
    }
  }
);
```

### 3. Pagination State

```typescript
const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [],
    page: 1,
    pageSize: 20,
    total: 0,
    hasMore: true
  },
  reducers: {
    pageChanged: (state, action) => {
      state.page = action.payload;
    }
  },
  extraReducers: (builder) => {
    builder.addCase(fetchTodosPage.fulfilled, (state, action) => {
      state.items = action.payload.items;
      state.total = action.payload.total;
      state.hasMore = state.page * state.pageSize < state.total;
    });
  }
});
```

### 4. Undo/Redo

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface HistoryState<T> {
  past: T[];
  present: T;
  future: T[];
}

function withHistory<T>(initialState: T) {
  const history: HistoryState<T> = {
    past: [],
    present: initialState,
    future: []
  };
  
  return {
    initialState: history,
    reducers: {
      undo: (state: HistoryState<T>) => {
        if (state.past.length === 0) return;
        
        const previous = state.past[state.past.length - 1];
        const newPast = state.past.slice(0, -1);
        
        state.past = newPast;
        state.future = [state.present, ...state.future];
        state.present = previous;
      },
      redo: (state: HistoryState<T>) => {
        if (state.future.length === 0) return;
        
        const next = state.future[0];
        const newFuture = state.future.slice(1);
        
        state.past = [...state.past, state.present];
        state.future = newFuture;
        state.present = next;
      },
      updatePresent: (state: HistoryState<T>, action: PayloadAction<T>) => {
        state.past = [...state.past, state.present];
        state.present = action.payload;
        state.future = [];
      }
    }
  };
}

// Usage
const todosSlice = createSlice({
  name: 'todos',
  ...withHistory([]),
  reducers: {
    addTodo: (state, action) => {
      state.past = [...state.past, state.present];
      state.present = [...state.present, action.payload];
      state.future = [];
    }
  }
});
```

## Common Mistakes

### 1. Mixing Immer Patterns

```typescript
// Bad: Mixing mutation and return
const badSlice = createSlice({
  name: 'bad',
  initialState: { items: [] },
  reducers: {
    addItem: (state, action) => {
      state.items.push(action.payload); // Mutation
      return { ...state, count: state.items.length }; // ❌ Don't return after mutating!
    }
  }
});

// Good: Choose one pattern
const goodSlice = createSlice({
  name: 'good',
  initialState: { items: [], count: 0 },
  reducers: {
    addItem: (state, action) => {
      state.items.push(action.payload);
      state.count = state.items.length;
      // No return - Immer handles it
    }
  }
});
```

### 2. Not Using TypeScript

```typescript
// Bad: No types
const badSlice = createSlice({
  name: 'bad',
  initialState: { value: 0 },
  reducers: {
    update: (state, action) => {
      state.value = action.payload; // What type is payload?
    }
  }
});

// Good: Full TypeScript
interface CounterState {
  value: number;
}

const goodSlice = createSlice({
  name: 'good',
  initialState: { value: 0 } as CounterState,
  reducers: {
    update: (state, action: PayloadAction<number>) => {
      state.value = action.payload; // Type-safe!
    }
  }
});
```

### 3. Not Handling Async Errors

```typescript
// Bad: No error handling
const fetchTodos = createAsyncThunk('todos/fetch', async () => {
  const response = await fetch('/api/todos');
  return response.json(); // What if this fails?
});

// Good: Proper error handling
const fetchTodos = createAsyncThunk(
  'todos/fetch',
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch('/api/todos');
      if (!response.ok) {
        return rejectWithValue('Server error');
      }
      return await response.json();
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);
```

### 4. Over-normalizing State

```typescript
// Bad: Over-normalized for simple data
const badState = {
  todos: {
    byId: { '1': { id: '1', text: 'Todo' } },
    allIds: ['1']
  }
  // Overkill for a simple todo list
};

// Good: Keep it simple when appropriate
const goodState = {
  todos: [{ id: '1', text: 'Todo' }]
  // Simple array is fine for small lists
};
```

### 5. Not Using RTK Query for Server State

```typescript
// Bad: Manual API state management
const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [], loading: false },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => { state.loading = true; })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.items = action.payload;
        state.loading = false;
      });
  }
});
// Need to manage caching, refetching, invalidation manually

// Good: Use RTK Query
const todosApi = createApi({
  reducerPath: 'todosApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    getTodos: builder.query<Todo[], void>({
      query: () => '/todos'
    })
  })
});
// Automatic caching, refetching, invalidation!
```

## Best Practices

### 1. Use TypeScript

Always use TypeScript for type safety:

```typescript
// Define types
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Type state
interface TodosState {
  items: Todo[];
  loading: boolean;
}

// Type actions
const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [], loading: false } as TodosState,
  reducers: {
    addTodo: (state, action: PayloadAction<Todo>) => {
      state.items.push(action.payload);
    }
  }
});

// Export typed hooks
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### 2. Organize by Feature

```
src/
  features/
    todos/
      todosSlice.ts
      todosAPI.ts
      TodoList.tsx
      TodoItem.tsx
    users/
      usersSlice.ts
      usersAPI.ts
      UserProfile.tsx
  app/
    store.ts
    hooks.ts
```

### 3. Use RTK Query for Server State

RTK Query handles caching, refetching, and invalidation automatically.

### 4. Keep Slices Focused

One slice per domain/feature:

```typescript
// Good: Focused slices
const todosSlice = createSlice({ /* todos only */ });
const userSlice = createSlice({ /* user only */ });
const uiSlice = createSlice({ /* UI state only */ });

// Bad: Everything in one slice
const appSlice = createSlice({ /* todos, user, UI, cart, ... */ });
```

### 5. Use createEntityAdapter for Collections

For lists of entities, use `createEntityAdapter` for normalized state.

## Interview Questions

### Q1: What problem does Redux Toolkit solve?

**Answer:** Redux Toolkit solves three main problems with traditional Redux:

1. **Too much boilerplate**: Traditional Redux requires separate action types, action creators, and reducers with lots of switch statements. RTK combines these with `createSlice`.

2. **Complex configuration**: Setting up store with middleware, DevTools, and proper defaults is tedious. `configureStore` provides good defaults out of the box.

3. **Immutable updates are verbose**: Manual immutable updates with spread operators are error-prone. RTK uses Immer to allow "mutating" code that's converted to immutable updates.

Example: Traditional Redux needs ~50 lines for what RTK does in ~20 lines, with better defaults.

### Q2: How does Immer work in Redux Toolkit?

**Answer:** Immer allows you to write code that looks like it mutates state, but actually produces immutable updates:

```typescript
// Without Immer (traditional)
return {
  ...state,
  items: state.items.map(item =>
    item.id === id ? { ...item, completed: !item.completed } : item
  )
};

// With Immer (RTK)
const item = state.items.find(i => i.id === id);
if (item) item.completed = !item.completed;
```

Immer creates a "draft" of your state. Changes to the draft are tracked and converted to immutable updates. You can either mutate the draft OR return new state, but not both.

### Q3: What is createAsyncThunk and when would you use it?

**Answer:** `createAsyncThunk` is a function that generates action creators for async operations. It automatically dispatches pending/fulfilled/rejected actions:

```typescript
const fetchTodos = createAsyncThunk(
  'todos/fetch',
  async (userId: string) => {
    const response = await fetch(`/api/users/${userId}/todos`);
    return response.json();
  }
);

// Generates:
// - fetchTodos.pending
// - fetchTodos.fulfilled
// - fetchTodos.rejected
```

Use it for:
- API calls
- Async operations (file uploads, etc.)
- Operations that need loading/error states

For simpler server state management, consider RTK Query instead.

### Q4: What's the difference between reducers and extraReducers in createSlice?

**Answer:**

**reducers**: Define actions that belong to this slice. RTK auto-generates action creators.
```typescript
reducers: {
  addTodo: (state, action) => { /* ... */ }
}
// Auto-generates addTodo action creator
```

**extraReducers**: Handle actions from outside the slice (other slices, async thunks, etc.). No action creators generated.
```typescript
extraReducers: (builder) => {
  builder.addCase(fetchTodos.fulfilled, (state, action) => {
    // Handle async thunk action
  });
}
```

Use `reducers` for synchronous slice-specific actions, `extraReducers` for external actions.

### Q5: How does RTK Query differ from Redux Thunk?

**Answer:**

**Redux Thunk:**
- Manual async logic
- Manual loading/error state management
- No automatic caching
- No automatic refetching
- Need to write reducers for server state

**RTK Query:**
- Automatic data fetching and caching
- Built-in loading/error states
- Automatic cache invalidation with tags
- Polling and refetching built-in
- Optimistic updates support
- Generated hooks for components

RTK Query is purpose-built for server state, while thunks are general-purpose async middleware. Use RTK Query for API data, thunks for other async logic.

### Q6: What is createEntityAdapter and when should you use it?

**Answer:** `createEntityAdapter` manages normalized state for collections:

```typescript
const adapter = createEntityAdapter<Todo>();
// Creates state: { ids: string[], entities: { [id: string]: Todo } }

// Provides CRUD methods:
adapter.addOne(state, todo);
adapter.updateOne(state, { id: '1', changes: { text: 'Updated' } });
adapter.removeOne(state, '1');

// Provides selectors:
selectAll, selectById, selectIds, etc.
```

Use when:
- Managing collections of items
- Need normalized state
- Want prebuilt CRUD operations
- Need efficient lookups by ID

Don't use for simple arrays or non-collection state.

### Q7: How do you handle optimistic updates in RTK?

**Answer:** Two approaches:

**1. With RTK Query:**
```typescript
updateTodo: builder.mutation({
  async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
    // Optimistically update cache
    const patchResult = dispatch(
      api.util.updateQueryData('getTodo', id, (draft) => {
        Object.assign(draft, patch);
      })
    );
    
    try {
      await queryFulfilled;
    } catch {
      patchResult.undo(); // Rollback on error
    }
  }
});
```

**2. With createAsyncThunk:**
```typescript
// Dispatch optimistic action immediately
dispatch(todoUpdatedOptimistic({ id, changes }));

// Then dispatch async thunk
dispatch(updateTodo({ id, changes }))
  .unwrap()
  .then(() => dispatch(todoUpdatedConfirmed(id)))
  .catch(() => dispatch(todoUpdatedRollback(id)));
```

### Q8: What are the benefits of using configureStore over createStore?

**Answer:**

**configureStore** includes by default:
1. **redux-thunk middleware** for async logic
2. **Redux DevTools Extension** automatically enabled
3. **Serializability check** in development (warns about non-serializable values)
4. **Immutability check** in development (warns about mutations)
5. **Automatic reducer combination** (no need for combineReducers)
6. **Better TypeScript support** with inferred types

**createStore** requires manual setup for all of these. `configureStore` provides better defaults and catches common mistakes in development.

## Key Takeaways

- Redux Toolkit dramatically reduces Redux boilerplate
- `createSlice` combines action types, creators, and reducers
- Immer allows "mutating" code that produces immutable updates
- `createAsyncThunk` handles async operations with automatic pending/fulfilled/rejected actions
- `configureStore` provides better defaults than `createStore`
- `createEntityAdapter` manages normalized collections with prebuilt CRUD operations
- RTK Query is purpose-built for server state with automatic caching
- Always use TypeScript for type safety
- Organize code by features, not file types
- Use RTK Query for server state, slices for client state
- `extraReducers` handle external actions, `reducers` define slice actions
- Keep slices focused on single domains
- RTK includes DevTools, thunk, and development checks by default
- Memoize derived state with `createSelector`
- Use `prepare` callbacks in reducers for complex action payloads

## Resources

- [Redux Toolkit Official Docs](https://redux-toolkit.js.org/)
- [RTK Query Overview](https://redux-toolkit.js.org/rtk-query/overview)
- [Redux Essentials Tutorial](https://redux.js.org/tutorials/essentials/part-1-overview-concepts)
- [createAsyncThunk API](https://redux-toolkit.js.org/api/createAsyncThunk)
- [createEntityAdapter API](https://redux-toolkit.js.org/api/createEntityAdapter)
- [RTK Query Examples](https://redux-toolkit.js.org/rtk-query/usage/examples)
- [Why Redux Toolkit](https://redux-toolkit.js.org/introduction/getting-started#what-is-redux-toolkit)
- [Immer Documentation](https://immerjs.github.io/immer/)
