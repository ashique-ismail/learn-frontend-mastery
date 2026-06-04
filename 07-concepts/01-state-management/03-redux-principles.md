# Redux Principles

## Overview

Redux is a predictable state container for JavaScript applications, built on three fundamental principles that make state management simple, testable, and maintainable. Created by Dan Abramov in 2015, Redux evolved from Flux but simplified the architecture with a single store and pure functions. Understanding these core principles is essential for mastering not just Redux, but modern state management patterns in general.

## The Three Core Principles

### 1. Single Source of Truth

**The entire application state is stored in a single object tree within a single store.**

```
Traditional Multiple Stores (Flux):
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  UserStore  │  │  TodoStore  │  │  CartStore  │
└─────────────┘  └─────────────┘  └─────────────┘

Redux Single Store:
┌────────────────────────────────────────────┐
│            Redux Store                      │
│  ┌────────┬─────────┬─────────┬──────────┐│
│  │  user  │  todos  │  cart   │  ui      ││
│  └────────┴─────────┴─────────┴──────────┘│
└────────────────────────────────────────────┘
```

This single source of truth makes debugging, serialization, and hydration straightforward.

### 2. State is Read-Only

**The only way to change state is to emit an action, an object describing what happened.**

Views and network callbacks never write directly to state. Instead, they express intent through actions.

### 3. Changes are Made with Pure Functions

**To specify how state transforms in response to actions, you write pure reducers.**

Reducers are pure functions that take previous state and an action, and return new state.

```typescript
(previousState, action) => newState
```

## Single Source of Truth in Detail

### Why a Single Store?

```typescript
// The entire application state in one place
interface RootState {
  user: {
    currentUser: User | null;
    isAuthenticated: boolean;
    loading: boolean;
  };
  todos: {
    items: Todo[];
    filter: 'all' | 'active' | 'completed';
    loading: boolean;
  };
  cart: {
    items: CartItem[];
    total: number;
  };
  ui: {
    theme: 'light' | 'dark';
    sidebarOpen: boolean;
    modalStack: string[];
  };
}

// Benefits:
// 1. Easy to serialize entire app state
const serialized = JSON.stringify(store.getState());

// 2. Easy to hydrate app state
const state = JSON.parse(localStorage.getItem('state'));
const store = createStore(rootReducer, state);

// 3. Easy to implement undo/redo
const history: RootState[] = [];
history.push(store.getState()); // Save state snapshot

// 4. Time-travel debugging
// Redux DevTools can replay any state
```

### Single Store Benefits

```typescript
// Server-Side Rendering
import { renderToString } from 'react-dom/server';

function handleRequest(req, res) {
  const store = createStore(rootReducer);
  
  // Fetch all data needed
  await store.dispatch(loadUser(req.userId));
  await store.dispatch(loadTodos());
  
  // Render with complete state
  const html = renderToString(
    <Provider store={store}>
      <App />
    </Provider>
  );
  
  // Send state to client for hydration
  const state = store.getState();
  res.send(`
    <div id="root">${html}</div>
    <script>
      window.__INITIAL_STATE__ = ${JSON.stringify(state)};
    </script>
  `);
}

// Client-side hydration
const initialState = window.__INITIAL_STATE__;
const store = createStore(rootReducer, initialState);
```

### State Shape Best Practices

```typescript
// Bad: Deeply nested state
const badState = {
  users: {
    john: {
      profile: { name: 'John', email: 'john@example.com' },
      todos: [
        { id: 1, text: 'Todo 1', assignedTo: { name: 'John' } },
        { id: 2, text: 'Todo 2', assignedTo: { name: 'John' } }
      ]
    }
  }
};

// Good: Normalized, flat state
const goodState = {
  entities: {
    users: {
      byId: {
        'john': { id: 'john', name: 'John', email: 'john@example.com' }
      },
      allIds: ['john']
    },
    todos: {
      byId: {
        '1': { id: '1', text: 'Todo 1', userId: 'john' },
        '2': { id: '2', text: 'Todo 2', userId: 'john' }
      },
      allIds: ['1', '2']
    }
  },
  ui: {
    currentUser: 'john',
    selectedTodoId: '1'
  }
};
```

## State is Read-Only in Detail

### The Problem with Mutable State

```typescript
// Bad: Direct mutation (NEVER DO THIS)
function TodoList() {
  const todos = useSelector(state => state.todos);
  
  const handleToggle = (id: string) => {
    const todo = todos.find(t => t.id === id);
    todo.completed = !todo.completed; // ❌ MUTATION!
    // React won't detect this change
  };
}

// Good: Dispatch actions
function TodoList() {
  const todos = useSelector(state => state.todos);
  const dispatch = useDispatch();
  
  const handleToggle = (id: string) => {
    dispatch({ type: 'TOGGLE_TODO', payload: { id } }); // ✅ Immutable
  };
}
```

### Actions: Expressing Intent

```typescript
// Actions are plain objects with a type
interface Action {
  type: string;
  payload?: any;
  meta?: any;
  error?: boolean;
}

// Example actions
const addTodo = (text: string) => ({
  type: 'ADD_TODO',
  payload: { text, id: Date.now() }
});

const toggleTodo = (id: string) => ({
  type: 'TOGGLE_TODO',
  payload: { id }
});

const loadTodosSuccess = (todos: Todo[]) => ({
  type: 'LOAD_TODOS_SUCCESS',
  payload: { todos }
});

const loadTodosError = (error: Error) => ({
  type: 'LOAD_TODOS_ERROR',
  payload: error,
  error: true
});
```

### Flux Standard Action (FSA)

```typescript
// FSA-compliant action structure
interface FluxStandardAction<P = any, M = any> {
  type: string;           // Required: describes action
  payload?: P;            // Optional: action data
  error?: boolean;        // Optional: true if action represents error
  meta?: M;              // Optional: extra info not part of payload
}

// Examples
const successAction: FluxStandardAction = {
  type: 'FETCH_USER_SUCCESS',
  payload: { user: { id: 1, name: 'John' } }
};

const errorAction: FluxStandardAction = {
  type: 'FETCH_USER_ERROR',
  payload: new Error('Network error'),
  error: true
};

const metaAction: FluxStandardAction = {
  type: 'ADD_TODO',
  payload: { text: 'Buy milk' },
  meta: { timestamp: Date.now(), source: 'mobile' }
};
```

### Action Flow

```
User Interaction Flow:

1. User clicks "Add Todo"
   │
   ↓
2. Component dispatches action:
   dispatch(addTodo('Buy milk'))
   │
   ↓
3. Action object created:
   { type: 'ADD_TODO', payload: { text: 'Buy milk' } }
   │
   ↓
4. Redux passes action to reducer
   │
   ↓
5. Reducer returns new state
   │
   ↓
6. Redux updates store
   │
   ↓
7. Subscribed components re-render
```

## Pure Reducers in Detail

### What is a Pure Function?

A pure function:
1. Same input always produces same output
2. No side effects (no API calls, mutations, etc.)
3. Doesn't depend on external state

```typescript
// Pure function
function add(a: number, b: number): number {
  return a + b; // ✅ Predictable, no side effects
}

// Impure functions
function impure1(a: number): number {
  return a + Math.random(); // ❌ Non-deterministic
}

function impure2(a: number): number {
  fetch('/api/data'); // ❌ Side effect
  return a;
}

let external = 0;
function impure3(a: number): number {
  return a + external; // ❌ Depends on external state
}
```

### Reducer Signature

```typescript
type Reducer<S, A> = (state: S | undefined, action: A) => S;

// Example reducer
function counterReducer(state = 0, action: Action): number {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1; // New value
    case 'DECREMENT':
      return state - 1; // New value
    case 'ADD':
      return state + action.payload.amount; // New value
    default:
      return state; // Current state unchanged
  }
}
```

### Immutable Updates

```typescript
// Todo reducer with immutable updates
interface TodoState {
  items: Todo[];
  filter: 'all' | 'active' | 'completed';
}

function todoReducer(
  state: TodoState = { items: [], filter: 'all' },
  action: Action
): TodoState {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        items: [
          ...state.items, // Spread existing items
          {
            id: action.payload.id,
            text: action.payload.text,
            completed: false
          }
        ]
      };
    
    case 'TOGGLE_TODO':
      return {
        ...state,
        items: state.items.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed } // New object
            : todo // Keep existing reference
        )
      };
    
    case 'DELETE_TODO':
      return {
        ...state,
        items: state.items.filter(todo => todo.id !== action.payload.id)
      };
    
    case 'SET_FILTER':
      return {
        ...state,
        filter: action.payload.filter
      };
    
    default:
      return state;
  }
}
```

### Combining Reducers

```typescript
// Individual reducers for different state slices
function userReducer(state = null, action: Action) {
  switch (action.type) {
    case 'LOGIN_SUCCESS':
      return action.payload.user;
    case 'LOGOUT':
      return null;
    default:
      return state;
  }
}

function todosReducer(state: Todo[] = [], action: Action) {
  switch (action.type) {
    case 'ADD_TODO':
      return [...state, action.payload.todo];
    case 'TOGGLE_TODO':
      return state.map(todo =>
        todo.id === action.payload.id
          ? { ...todo, completed: !todo.completed }
          : todo
      );
    default:
      return state;
  }
}

function cartReducer(state: CartState = { items: [] }, action: Action) {
  switch (action.type) {
    case 'ADD_TO_CART':
      return {
        ...state,
        items: [...state.items, action.payload.item]
      };
    default:
      return state;
  }
}

// Combine into root reducer
import { combineReducers } from 'redux';

const rootReducer = combineReducers({
  user: userReducer,
  todos: todosReducer,
  cart: cartReducer
});

// Resulting state shape:
// {
//   user: ...,
//   todos: [...],
//   cart: { items: [...] }
// }
```

### Reducer Composition

```typescript
// Compose reducers for nested state
function todoReducer(state: Todo, action: Action): Todo {
  switch (action.type) {
    case 'TOGGLE_TODO':
      return { ...state, completed: !state.completed };
    case 'UPDATE_TODO_TEXT':
      return { ...state, text: action.payload.text };
    default:
      return state;
  }
}

function todosReducer(state: Todo[] = [], action: Action): Todo[] {
  switch (action.type) {
    case 'ADD_TODO':
      return [...state, action.payload.todo];
    
    case 'TOGGLE_TODO':
    case 'UPDATE_TODO_TEXT':
      // Delegate to single todo reducer
      return state.map(todo =>
        todo.id === action.payload.id
          ? todoReducer(todo, action) // Compose!
          : todo
      );
    
    case 'DELETE_TODO':
      return state.filter(todo => todo.id !== action.payload.id);
    
    default:
      return state;
  }
}
```

## Complete Redux Flow Example

```typescript
// 1. Define action types
const ADD_TODO = 'ADD_TODO';
const TOGGLE_TODO = 'TOGGLE_TODO';

// 2. Define action creators
const addTodo = (text: string) => ({
  type: ADD_TODO,
  payload: { id: Date.now(), text }
});

const toggleTodo = (id: number) => ({
  type: TOGGLE_TODO,
  payload: { id }
});

// 3. Define initial state
interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

const initialState: Todo[] = [];

// 4. Define reducer (pure function)
function todosReducer(
  state = initialState,
  action: { type: string; payload?: any }
): Todo[] {
  switch (action.type) {
    case ADD_TODO:
      return [
        ...state,
        {
          id: action.payload.id,
          text: action.payload.text,
          completed: false
        }
      ];
    
    case TOGGLE_TODO:
      return state.map(todo =>
        todo.id === action.payload.id
          ? { ...todo, completed: !todo.completed }
          : todo
      );
    
    default:
      return state;
  }
}

// 5. Create store
import { createStore } from 'redux';
const store = createStore(todosReducer);

// 6. Subscribe to changes
store.subscribe(() => {
  console.log('State:', store.getState());
});

// 7. Dispatch actions
store.dispatch(addTodo('Learn Redux'));
// State: [{ id: 1234, text: 'Learn Redux', completed: false }]

store.dispatch(toggleTodo(1234));
// State: [{ id: 1234, text: 'Learn Redux', completed: true }]
```

## React Integration

```typescript
// React component using Redux
import { useSelector, useDispatch } from 'react-redux';
import { addTodo, toggleTodo } from './actions';
import { RootState } from './store';

function TodoList() {
  // Select state from store
  const todos = useSelector((state: RootState) => state.todos);
  const dispatch = useDispatch();
  const [text, setText] = useState('');
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (text.trim()) {
      dispatch(addTodo(text)); // Dispatch action
      setText('');
    }
  };
  
  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          value={text}
          onChange={e => setText(e.target.value)}
          placeholder="What needs to be done?"
        />
        <button type="submit">Add</button>
      </form>
      
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => dispatch(toggleTodo(todo.id))}
            />
            <span style={{
              textDecoration: todo.completed ? 'line-through' : 'none'
            }}>
              {todo.text}
            </span>
          </li>
        ))}
      </ul>
    </div>
  );
}

// App setup with Provider
import { Provider } from 'react-redux';
import { store } from './store';

function App() {
  return (
    <Provider store={store}>
      <TodoList />
    </Provider>
  );
}
```

## Angular Integration with Redux

```typescript
// Using NgRx (Angular's Redux implementation)
import { createAction, props } from '@ngrx/store';
import { createReducer, on } from '@ngrx/store';

// Actions
export const addTodo = createAction(
  '[Todo] Add Todo',
  props<{ text: string }>()
);

export const toggleTodo = createAction(
  '[Todo] Toggle Todo',
  props<{ id: string }>()
);

// Reducer
interface TodoState {
  items: Todo[];
}

const initialState: TodoState = {
  items: []
};

export const todoReducer = createReducer(
  initialState,
  on(addTodo, (state, { text }) => ({
    ...state,
    items: [
      ...state.items,
      { id: Date.now().toString(), text, completed: false }
    ]
  })),
  on(toggleTodo, (state, { id }) => ({
    ...state,
    items: state.items.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    )
  }))
);

// Component
import { Component } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-todo-list',
  template: `
    <div>
      <form (ngSubmit)="onSubmit()">
        <input [(ngModel)]="text" placeholder="What needs to be done?">
        <button type="submit">Add</button>
      </form>
      
      <ul>
        <li *ngFor="let todo of todos$ | async">
          <input
            type="checkbox"
            [checked]="todo.completed"
            (change)="onToggle(todo.id)"
          >
          <span [style.text-decoration]="todo.completed ? 'line-through' : 'none'">
            {{ todo.text }}
          </span>
        </li>
      </ul>
    </div>
  `
})
export class TodoListComponent {
  todos$: Observable<Todo[]>;
  text = '';
  
  constructor(private store: Store<{ todos: TodoState }>) {
    this.todos$ = store.select(state => state.todos.items);
  }
  
  onSubmit() {
    if (this.text.trim()) {
      this.store.dispatch(addTodo({ text: this.text }));
      this.text = '';
    }
  }
  
  onToggle(id: string) {
    this.store.dispatch(toggleTodo({ id }));
  }
}
```

## Time-Travel Debugging

One of Redux's most powerful features:

```typescript
// Redux DevTools enables time-travel
const store = createStore(
  rootReducer,
  window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__()
);

// In DevTools, you can:
// 1. See every action dispatched
// 2. Inspect state before/after each action
// 3. Jump to any point in time
// 4. Replay actions
// 5. Skip actions
// 6. Export/import state

// Example action history:
// [
//   { type: 'ADD_TODO', payload: { text: 'Learn Redux' } },
//   { type: 'ADD_TODO', payload: { text: 'Build app' } },
//   { type: 'TOGGLE_TODO', payload: { id: 1 } },
//   { type: 'DELETE_TODO', payload: { id: 2 } }
// ]

// Can jump to any action and see exact state at that point
```

## Middleware: Extending Redux

While reducers must be pure, middleware handles side effects:

```typescript
// Simple logger middleware
const loggerMiddleware = (store) => (next) => (action) => {
  console.log('Dispatching:', action);
  const result = next(action);
  console.log('Next state:', store.getState());
  return result;
};

// Async middleware (redux-thunk)
const thunk = (store) => (next) => (action) => {
  if (typeof action === 'function') {
    return action(store.dispatch, store.getState);
  }
  return next(action);
};

// Usage with thunk
const loadTodos = () => {
  return async (dispatch, getState) => {
    dispatch({ type: 'LOAD_TODOS_REQUEST' });
    
    try {
      const response = await fetch('/api/todos');
      const todos = await response.json();
      dispatch({ type: 'LOAD_TODOS_SUCCESS', payload: { todos } });
    } catch (error) {
      dispatch({ type: 'LOAD_TODOS_ERROR', payload: { error } });
    }
  };
};

// Apply middleware
import { createStore, applyMiddleware } from 'redux';
const store = createStore(
  rootReducer,
  applyMiddleware(thunk, loggerMiddleware)
);
```

## Common Mistakes

### 1. Mutating State in Reducers

```typescript
// Bad: Direct mutation
function badReducer(state = { items: [] }, action) {
  switch (action.type) {
    case 'ADD_ITEM':
      state.items.push(action.payload.item); // ❌ MUTATION!
      return state;
    case 'UPDATE_ITEM':
      const item = state.items.find(i => i.id === action.payload.id);
      item.name = action.payload.name; // ❌ MUTATION!
      return state;
  }
}

// Good: Immutable updates
function goodReducer(state = { items: [] }, action) {
  switch (action.type) {
    case 'ADD_ITEM':
      return {
        ...state,
        items: [...state.items, action.payload.item] // ✅ New array
      };
    case 'UPDATE_ITEM':
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.payload.id
            ? { ...item, name: action.payload.name } // ✅ New object
            : item
        )
      };
  }
}
```

### 2. Side Effects in Reducers

```typescript
// Bad: Side effects in reducer
function badReducer(state = {}, action) {
  switch (action.type) {
    case 'SAVE_TODO':
      fetch('/api/todos', { // ❌ Side effect!
        method: 'POST',
        body: JSON.stringify(action.payload)
      });
      return { ...state, todo: action.payload };
  }
}

// Good: Side effects in middleware/thunks
const saveTodo = (todo) => {
  return async (dispatch) => {
    try {
      await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(todo)
      });
      dispatch({ type: 'SAVE_TODO_SUCCESS', payload: todo });
    } catch (error) {
      dispatch({ type: 'SAVE_TODO_ERROR', payload: error });
    }
  };
};
```

### 3. Non-Serializable Values in State

```typescript
// Bad: Functions, promises, dates in state
const badState = {
  user: { id: 1, name: 'John' },
  callback: () => {}, // ❌ Function
  promise: fetch('/api'), // ❌ Promise
  date: new Date() // ❌ Date object
};

// Good: Only serializable data
const goodState = {
  user: { id: 1, name: 'John' },
  timestamp: Date.now(), // ✅ Number
  isoDate: new Date().toISOString() // ✅ String
};
```

### 4. Deeply Nested State Updates

```typescript
// Bad: Complex nested updates
function badReducer(state, action) {
  return {
    ...state,
    users: {
      ...state.users,
      [action.userId]: {
        ...state.users[action.userId],
        profile: {
          ...state.users[action.userId].profile,
          settings: {
            ...state.users[action.userId].profile.settings,
            notifications: action.payload
          }
        }
      }
    }
  };
}

// Good: Normalize state structure
// State: { users: { byId: {...}, allIds: [...] } }
function goodReducer(state, action) {
  return {
    ...state,
    users: {
      ...state.users,
      byId: {
        ...state.users.byId,
        [action.userId]: {
          ...state.users.byId[action.userId],
          notificationSettings: action.payload
        }
      }
    }
  };
}
```

### 5. Accessing Store Directly

```typescript
// Bad: Importing store directly in components
import { store } from './store';

function BadComponent() {
  const todos = store.getState().todos; // ❌ Won't re-render on changes
  return <div>{todos.length}</div>;
}

// Good: Use hooks/connect
function GoodComponent() {
  const todos = useSelector(state => state.todos); // ✅ Subscribes properly
  return <div>{todos.length}</div>;
}
```

## Best Practices

### 1. Use TypeScript for Type Safety

```typescript
// Define state types
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

interface TodoState {
  items: Todo[];
  loading: boolean;
  error: string | null;
}

// Define action types
type TodoAction =
  | { type: 'ADD_TODO'; payload: { text: string } }
  | { type: 'TOGGLE_TODO'; payload: { id: string } }
  | { type: 'DELETE_TODO'; payload: { id: string } };

// Type-safe reducer
function todoReducer(
  state: TodoState = { items: [], loading: false, error: null },
  action: TodoAction
): TodoState {
  // TypeScript ensures correct action types and payloads
}
```

### 2. Use Action Creators

```typescript
// Consistent action creation
const todoActions = {
  addTodo: (text: string) => ({
    type: 'ADD_TODO' as const,
    payload: { text, id: Date.now().toString() }
  }),
  toggleTodo: (id: string) => ({
    type: 'TOGGLE_TODO' as const,
    payload: { id }
  })
};

// Usage
dispatch(todoActions.addTodo('Buy milk'));
```

### 3. Normalize State Shape

```typescript
// Use normalizr or manual normalization
const normalizedState = {
  entities: {
    users: {
      byId: { '1': { id: '1', name: 'John' } },
      allIds: ['1']
    },
    todos: {
      byId: {
        '1': { id: '1', text: 'Todo 1', userId: '1' }
      },
      allIds: ['1']
    }
  },
  result: ['1'] // Top-level IDs
};
```

### 4. Use Selectors

```typescript
// Reusable selectors
const selectTodos = (state: RootState) => state.todos.items;
const selectCompletedTodos = (state: RootState) =>
  state.todos.items.filter(t => t.completed);
const selectActiveTodos = (state: RootState) =>
  state.todos.items.filter(t => !t.completed);

// Memoized selectors with reselect
import { createSelector } from 'reselect';

const selectCompletedTodosMemoized = createSelector(
  [selectTodos],
  (todos) => todos.filter(t => t.completed)
);
```

### 5. Keep Reducers Small and Focused

```typescript
// Split by domain
const userReducer = (state, action) => { /* handle user actions */ };
const todoReducer = (state, action) => { /* handle todo actions */ };
const cartReducer = (state, action) => { /* handle cart actions */ };

// Combine
const rootReducer = combineReducers({
  user: userReducer,
  todos: todoReducer,
  cart: cartReducer
});
```

## When to Use Redux

### Good Use Cases

- Large applications with lots of shared state
- Complex state update logic
- Need for time-travel debugging
- State needs to persist/hydrate (SSR, localStorage)
- Many components access same state
- Need middleware for cross-cutting concerns

### When NOT to Use Redux

- Simple applications with minimal state
- Most state is local/component-specific
- Form-heavy applications (use React Hook Form)
- Server state (use React Query/SWR)
- Quick prototypes

## Interview Questions

### Q1: What are the three principles of Redux?

**Answer:** 

1. **Single Source of Truth**: The entire application state is stored in a single object tree within one store. This makes debugging, serialization, and SSR easier.

2. **State is Read-Only**: State can only be changed by dispatching actions. Views never write directly to state, ensuring predictable state updates.

3. **Changes are Made with Pure Functions**: Reducers are pure functions that take (state, action) and return new state. No side effects, no mutations, same input always produces same output.

### Q2: What makes a reducer a pure function?

**Answer:** A reducer is pure when it:

1. **Returns same output for same input**: `reducer(state1, action1)` always returns the same result
2. **No side effects**: No API calls, DOM manipulation, or other side effects
3. **No mutations**: Returns new state object, never modifies input state
4. **No external dependencies**: Only uses its parameters, no global variables

Example:
```typescript
// Pure
function pureReducer(state = 0, action) {
  return action.type === 'INCREMENT' ? state + 1 : state;
}

// Impure
function impureReducer(state = 0, action) {
  fetch('/api'); // Side effect!
  state.count++; // Mutation!
  return state;
}
```

### Q3: Why is immutability important in Redux?

**Answer:** Immutability is crucial for:

1. **Change detection**: Redux uses shallow equality (`===`) to detect changes. If you mutate state, references stay the same, so React won't re-render.

2. **Time-travel debugging**: DevTools need state snapshots. Mutations destroy previous states.

3. **Performance optimization**: React.memo, useMemo, and PureComponent rely on reference equality.

4. **Predictability**: Knowing state never mutates makes debugging easier.

Example:
```typescript
// Mutation - React won't re-render
state.todos.push(newTodo);
return state; // Same reference!

// Immutable - React detects change
return {
  ...state,
  todos: [...state.todos, newTodo] // New reference!
};
```

### Q4: How does Redux differ from Flux?

**Answer:**

**Redux:**
- Single store
- No explicit dispatcher
- Reducers are pure functions
- No store classes
- Time-travel debugging

**Flux:**
- Multiple stores
- Explicit dispatcher
- Stores contain logic and state
- Store classes with methods
- No built-in time-travel

Redux simplified Flux by removing the dispatcher and using pure functions instead of store classes.

### Q5: What is the purpose of action creators?

**Answer:** Action creators:

1. **Encapsulate action structure**: Hide implementation details
2. **Provide type safety**: TypeScript can validate parameters
3. **Enable reusability**: Same action used in multiple places
4. **Simplify testing**: Easy to test action creation separately
5. **Handle async logic**: Can contain thunks/sagas

Example:
```typescript
// Without action creator
dispatch({ type: 'ADD_TODO', payload: { id: Date.now(), text: 'Buy milk' }});

// With action creator
const addTodo = (text) => ({
  type: 'ADD_TODO',
  payload: { id: Date.now(), text }
});
dispatch(addTodo('Buy milk')); // Cleaner, reusable
```

### Q6: How do you handle async operations in Redux?

**Answer:** Use middleware:

1. **Redux Thunk**: Dispatch functions instead of objects
```typescript
const loadTodos = () => async (dispatch) => {
  dispatch({ type: 'LOAD_REQUEST' });
  const data = await fetch('/api/todos');
  dispatch({ type: 'LOAD_SUCCESS', payload: data });
};
```

2. **Redux Saga**: Use generator functions
```typescript
function* loadTodosSaga() {
  yield put({ type: 'LOAD_REQUEST' });
  const data = yield call(fetch, '/api/todos');
  yield put({ type: 'LOAD_SUCCESS', payload: data });
}
```

3. **RTK Query**: Built-in data fetching
```typescript
const api = createApi({
  endpoints: (builder) => ({
    getTodos: builder.query({ query: () => '/todos' })
  })
});
```

### Q7: What is the purpose of combineReducers?

**Answer:** `combineReducers` combines multiple reducer functions into a single root reducer, managing different slices of state independently:

```typescript
const rootReducer = combineReducers({
  user: userReducer,     // Manages state.user
  todos: todosReducer,   // Manages state.todos
  cart: cartReducer      // Manages state.cart
});

// Resulting state shape:
// { user: {...}, todos: [...], cart: {...} }
```

Benefits:
- Separation of concerns
- Each reducer handles its domain
- Easy to test reducers independently
- Scales well with app growth

### Q8: When should you use Redux vs Context API?

**Answer:**

**Use Redux when:**
- Complex state with many slices
- Frequent state updates
- Need middleware (logging, analytics)
- Time-travel debugging required
- Large team, need standardization

**Use Context when:**
- Simple, infrequent updates (theme, auth)
- Small-to-medium apps
- Don't need middleware
- Want to avoid dependencies
- State updates are rare

**Use both:**
- Context for infrequent data (theme)
- Redux for frequent updates (UI state)
- React Query for server state

## Key Takeaways

- Redux is built on three principles: single source of truth, read-only state, pure reducers
- Single store contains entire application state in one object tree
- State can only change by dispatching actions
- Actions are plain objects describing what happened
- Reducers are pure functions: (state, action) => newState
- Never mutate state - always return new objects/arrays
- Use combineReducers to manage different state slices
- Middleware handles side effects (async, logging, etc.)
- TypeScript provides valuable type safety
- Normalize state structure for complex data
- Use selectors to derive data from state
- Redux DevTools enable time-travel debugging
- Keep reducers focused and testable
- Action creators encapsulate action creation logic
- Redux works with any UI library, not just React

## Resources

- [Redux Official Documentation](https://redux.js.org/)
- [Redux Fundamentals Tutorial](https://redux.js.org/tutorials/fundamentals/part-1-overview)
- [Getting Started with Redux (Dan Abramov)](https://egghead.io/courses/getting-started-with-redux)
- [Redux Style Guide](https://redux.js.org/style-guide/style-guide)
- [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)
- [NgRx (Angular + Redux)](https://ngrx.io/)
- [Redux Toolkit](https://redux-toolkit.js.org/)
- [Normalizing State Shape](https://redux.js.org/usage/structuring-reducers/normalizing-state-shape)
