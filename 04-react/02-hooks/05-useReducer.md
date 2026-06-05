# useReducer Hook - Advanced State Management

## The Idea

**In plain English:** `useReducer` is a way to manage all the changing information in your app by sending labeled instructions ("actions") to a central decision-maker (the "reducer") that figures out what the new information should look like. Instead of updating many separate pieces of data one by one, you describe *what happened* and let the reducer handle the rest.

**Real-world analogy:** Think of a bank teller at a branch. You walk up and hand them a slip that says "Withdraw $50 from checking." You don't reach into the drawer yourself — you submit a request, and the teller follows the bank's rules to update your account.

- The bank teller = the reducer (the function that decides how state changes)
- The slip of paper you hand in = the action (the instruction describing what should happen)
- Your account balance = the state (the current data being tracked)

---

## Table of Contents

- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Core API](#core-api)
- [When to Use useReducer](#when-to-use-usereducer)
- [Advanced Patterns](#advanced-patterns)
- [Performance Considerations](#performance-considerations)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

`useReducer` is a React Hook for managing complex state logic in components. It's an alternative to `useState` that's particularly useful when state updates depend on previous state or when state logic becomes complex.

### What Makes useReducer Special?

```
useState Pattern:                  useReducer Pattern:
┌─────────────┐                   ┌─────────────┐
│  Component  │                   │  Component  │
└──────┬──────┘                   └──────┬──────┘
       │                                  │
       │ setState()                       │ dispatch(action)
       ↓                                  ↓
┌─────────────┐                   ┌─────────────┐
│    State    │                   │   Reducer   │
└─────────────┘                   └──────┬──────┘
                                         │
                                         ↓
                                  ┌─────────────┐
                                  │    State    │
                                  └─────────────┘
```

**Key Advantages:**
- Centralized state logic
- Predictable state updates
- Better for complex state objects
- Easier to test
- Improved debugging with action logging

## Basic Concepts

### The Reducer Function

A reducer is a pure function that takes current state and an action, then returns new state:

```javascript
function reducer(state, action) {
  // Return new state based on action
  return newState;
}
```

### Anatomy of useReducer

```javascript
const [state, dispatch] = useReducer(reducer, initialState, init);
```

**Parameters:**
- `reducer`: Function that determines how state updates
- `initialState`: Initial state value
- `init`: Optional function for lazy initialization

**Returns:**
- `state`: Current state value
- `dispatch`: Function to trigger state updates

## Core API

### Basic Usage

```javascript
import { useReducer } from 'react';

// 1. Define action types (optional but recommended)
const ActionTypes = {
  INCREMENT: 'INCREMENT',
  DECREMENT: 'DECREMENT',
  RESET: 'RESET'
};

// 2. Define reducer function
function counterReducer(state, action) {
  switch (action.type) {
    case ActionTypes.INCREMENT:
      return { count: state.count + 1 };
    case ActionTypes.DECREMENT:
      return { count: state.count - 1 };
    case ActionTypes.RESET:
      return { count: 0 };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

// 3. Use in component
function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: ActionTypes.INCREMENT })}>
        +1
      </button>
      <button onClick={() => dispatch({ type: ActionTypes.DECREMENT })}>
        -1
      </button>
      <button onClick={() => dispatch({ type: ActionTypes.RESET })}>
        Reset
      </button>
    </div>
  );
}
```

### With Action Payloads

```javascript
const ActionTypes = {
  ADD_TODO: 'ADD_TODO',
  TOGGLE_TODO: 'TOGGLE_TODO',
  DELETE_TODO: 'DELETE_TODO'
};

function todoReducer(state, action) {
  switch (action.type) {
    case ActionTypes.ADD_TODO:
      return {
        ...state,
        todos: [
          ...state.todos,
          {
            id: Date.now(),
            text: action.payload.text,
            completed: false
          }
        ]
      };
    
    case ActionTypes.TOGGLE_TODO:
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo
        )
      };
    
    case ActionTypes.DELETE_TODO:
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload.id)
      };
    
    default:
      return state;
  }
}

function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, { todos: [] });
  const [inputText, setInputText] = useState('');

  const handleAddTodo = () => {
    if (inputText.trim()) {
      dispatch({
        type: ActionTypes.ADD_TODO,
        payload: { text: inputText }
      });
      setInputText('');
    }
  };

  return (
    <div>
      <input
        value={inputText}
        onChange={(e) => setInputText(e.target.value)}
        onKeyPress={(e) => e.key === 'Enter' && handleAddTodo()}
      />
      <button onClick={handleAddTodo}>Add</button>
      
      <ul>
        {state.todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => dispatch({
                type: ActionTypes.TOGGLE_TODO,
                payload: { id: todo.id }
              })}
            />
            <span style={{ 
              textDecoration: todo.completed ? 'line-through' : 'none' 
            }}>
              {todo.text}
            </span>
            <button onClick={() => dispatch({
              type: ActionTypes.DELETE_TODO,
              payload: { id: todo.id }
            })}>
              Delete
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Lazy Initialization

```javascript
// Expensive initial state computation
function init(initialCount) {
  // This runs only once
  return {
    count: initialCount,
    history: [initialCount],
    metadata: {
      createdAt: Date.now(),
      version: '1.0'
    }
  };
}

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      const newCount = state.count + 1;
      return {
        ...state,
        count: newCount,
        history: [...state.history, newCount]
      };
    case 'reset':
      return init(action.payload);
    default:
      return state;
  }
}

function Counter({ initialCount = 0 }) {
  // Third parameter is init function
  const [state, dispatch] = useReducer(reducer, initialCount, init);

  return (
    <div>
      <p>Count: {state.count}</p>
      <p>History: {state.history.join(', ')}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>
        Increment
      </button>
      <button onClick={() => dispatch({ 
        type: 'reset', 
        payload: 0 
      })}>
        Reset
      </button>
    </div>
  );
}
```

## When to Use useReducer

### useState vs useReducer Decision Matrix

| Scenario | Use useState | Use useReducer |
|----------|--------------|----------------|
| Simple values | ✅ | ❌ |
| Independent state pieces | ✅ | ❌ |
| Complex objects | ❌ | ✅ |
| Related state updates | ❌ | ✅ |
| State depends on previous | Maybe | ✅ |
| Multiple sub-values | ❌ | ✅ |
| Need to test logic | ❌ | ✅ |

### Good Use Cases for useReducer

```javascript
// ❌ BAD: Simple independent state - use useState
const [name, setName] = useState('');
const [email, setEmail] = useState('');

// ✅ GOOD: Related state with complex updates
const [formState, dispatch] = useReducer(formReducer, {
  fields: { name: '', email: '' },
  errors: {},
  touched: {},
  isSubmitting: false
});

// ❌ BAD: Single boolean - use useState
const [isOpen, setIsOpen] = useState(false);

// ✅ GOOD: Complex UI state
const [modalState, dispatch] = useReducer(modalReducer, {
  isOpen: false,
  content: null,
  size: 'medium',
  dismissible: true,
  onClose: null
});
```

### Complex State Example

```javascript
// Managing a shopping cart
const cartReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_ITEM':
      const existingItem = state.items.find(
        item => item.id === action.payload.id
      );
      
      if (existingItem) {
        return {
          ...state,
          items: state.items.map(item =>
            item.id === action.payload.id
              ? { ...item, quantity: item.quantity + 1 }
              : item
          ),
          total: state.total + action.payload.price
        };
      }
      
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
        total: state.total + action.payload.price
      };
    
    case 'REMOVE_ITEM':
      const itemToRemove = state.items.find(
        item => item.id === action.payload.id
      );
      
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.payload.id),
        total: state.total - (itemToRemove.price * itemToRemove.quantity)
      };
    
    case 'UPDATE_QUANTITY':
      const oldItem = state.items.find(
        item => item.id === action.payload.id
      );
      const priceDiff = 
        (action.payload.quantity - oldItem.quantity) * oldItem.price;
      
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.payload.id
            ? { ...item, quantity: action.payload.quantity }
            : item
        ),
        total: state.total + priceDiff
      };
    
    case 'APPLY_DISCOUNT':
      return {
        ...state,
        discount: action.payload.discount,
        discountCode: action.payload.code
      };
    
    case 'CLEAR_CART':
      return {
        items: [],
        total: 0,
        discount: 0,
        discountCode: null
      };
    
    default:
      return state;
  }
};

function ShoppingCart() {
  const [state, dispatch] = useReducer(cartReducer, {
    items: [],
    total: 0,
    discount: 0,
    discountCode: null
  });

  const finalTotal = state.total - state.discount;

  return (
    <div>
      <h2>Shopping Cart</h2>
      {state.items.map(item => (
        <div key={item.id}>
          <span>{item.name} x {item.quantity}</span>
          <span>${item.price * item.quantity}</span>
          <button onClick={() => dispatch({
            type: 'UPDATE_QUANTITY',
            payload: { id: item.id, quantity: item.quantity + 1 }
          })}>+</button>
          <button onClick={() => dispatch({
            type: 'REMOVE_ITEM',
            payload: { id: item.id }
          })}>Remove</button>
        </div>
      ))}
      <div>
        <p>Subtotal: ${state.total}</p>
        {state.discount > 0 && <p>Discount: -${state.discount}</p>}
        <p><strong>Total: ${finalTotal}</strong></p>
      </div>
    </div>
  );
}
```

## Advanced Patterns

### TypeScript Integration

```typescript
// Define state type
interface State {
  count: number;
  error: string | null;
  isLoading: boolean;
}

// Define action types
type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'SET_ERROR'; payload: string }
  | { type: 'CLEAR_ERROR' }
  | { type: 'SET_LOADING'; payload: boolean };

// Typed reducer
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1, error: null };
    case 'DECREMENT':
      return { ...state, count: state.count - 1, error: null };
    case 'SET_ERROR':
      return { ...state, error: action.payload };
    case 'CLEAR_ERROR':
      return { ...state, error: null };
    case 'SET_LOADING':
      return { ...state, isLoading: action.payload };
    default:
      // TypeScript ensures exhaustive checking
      const _exhaustive: never = action;
      return state;
  }
}

// Usage with full type safety
function TypedCounter() {
  const [state, dispatch] = useReducer(reducer, {
    count: 0,
    error: null,
    isLoading: false
  });

  // TypeScript will catch invalid actions
  // dispatch({ type: 'INVALID' }); // Error!
  
  return (
    <div>
      {state.error && <div className="error">{state.error}</div>}
      <p>Count: {state.count}</p>
    </div>
  );
}
```

### Action Creators Pattern

```javascript
// Action creators for better maintainability
const actions = {
  addTodo: (text) => ({
    type: 'ADD_TODO',
    payload: { text, id: Date.now() }
  }),
  
  toggleTodo: (id) => ({
    type: 'TOGGLE_TODO',
    payload: { id }
  }),
  
  deleteTodo: (id) => ({
    type: 'DELETE_TODO',
    payload: { id }
  }),
  
  setFilter: (filter) => ({
    type: 'SET_FILTER',
    payload: { filter }
  })
};

function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, initialState);

  return (
    <div>
      <button onClick={() => dispatch(actions.addTodo('New task'))}>
        Add Todo
      </button>
      <button onClick={() => dispatch(actions.setFilter('completed'))}>
        Show Completed
      </button>
    </div>
  );
}
```

### Middleware Pattern

```javascript
// Logger middleware
function withLogger(reducer) {
  return (state, action) => {
    console.group(action.type);
    console.log('Previous State:', state);
    console.log('Action:', action);
    const newState = reducer(state, action);
    console.log('Next State:', newState);
    console.groupEnd();
    return newState;
  };
}

// Async middleware
function withAsync(reducer) {
  return (state, action) => {
    if (action.type.endsWith('_REQUEST')) {
      return { ...state, isLoading: true, error: null };
    }
    if (action.type.endsWith('_SUCCESS')) {
      return { ...state, isLoading: false };
    }
    if (action.type.endsWith('_FAILURE')) {
      return { ...state, isLoading: false, error: action.payload };
    }
    return reducer(state, action);
  };
}

// Compose middleware
function compose(...middlewares) {
  return (reducer) => 
    middlewares.reduceRight((acc, middleware) => middleware(acc), reducer);
}

// Usage
const enhancedReducer = compose(
  withLogger,
  withAsync
)(baseReducer);

function Component() {
  const [state, dispatch] = useReducer(enhancedReducer, initialState);
  // ...
}
```

### Immer Integration for Immutability

```javascript
import { useReducer } from 'react';
import { produce } from 'immer';

const todoReducer = produce((draft, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      draft.todos.push({
        id: Date.now(),
        text: action.payload.text,
        completed: false
      });
      break;
    
    case 'TOGGLE_TODO':
      const todo = draft.todos.find(t => t.id === action.payload.id);
      if (todo) {
        todo.completed = !todo.completed;
      }
      break;
    
    case 'DELETE_TODO':
      const index = draft.todos.findIndex(t => t.id === action.payload.id);
      if (index !== -1) {
        draft.todos.splice(index, 1);
      }
      break;
  }
});

// No need to manually spread or copy - Immer handles it!
```

### Context + useReducer Pattern

```javascript
// Create context for global state management
const TodoContext = createContext(null);
const TodoDispatchContext = createContext(null);

export function TodoProvider({ children }) {
  const [state, dispatch] = useReducer(todoReducer, initialState);

  return (
    <TodoContext.Provider value={state}>
      <TodoDispatchContext.Provider value={dispatch}>
        {children}
      </TodoDispatchContext.Provider>
    </TodoContext.Provider>
  );
}

// Custom hooks for consuming context
export function useTodos() {
  const context = useContext(TodoContext);
  if (context === null) {
    throw new Error('useTodos must be used within TodoProvider');
  }
  return context;
}

export function useTodoDispatch() {
  const context = useContext(TodoDispatchContext);
  if (context === null) {
    throw new Error('useTodoDispatch must be used within TodoProvider');
  }
  return context;
}

// Usage in components
function TodoList() {
  const { todos } = useTodos();
  const dispatch = useTodoDispatch();

  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} dispatch={dispatch} />
      ))}
    </ul>
  );
}
```

## Performance Considerations

### Avoiding Unnecessary Renders

```javascript
// ❌ BAD: Creating new objects on every render
function TodoApp() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  // New object created every render!
  const handlers = {
    addTodo: (text) => dispatch({ type: 'ADD_TODO', payload: { text } }),
    toggleTodo: (id) => dispatch({ type: 'TOGGLE_TODO', payload: { id } })
  };
  
  return <TodoList handlers={handlers} />;
}

// ✅ GOOD: Stable callback references
function TodoApp() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  const addTodo = useCallback((text) => {
    dispatch({ type: 'ADD_TODO', payload: { text } });
  }, []);
  
  const toggleTodo = useCallback((id) => {
    dispatch({ type: 'TOGGLE_TODO', payload: { id } });
  }, []);
  
  return (
    <TodoList 
      addTodo={addTodo} 
      toggleTodo={toggleTodo} 
    />
  );
}

// ✅ BETTER: Pass dispatch directly
function TodoApp() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  // dispatch is stable across renders
  return <TodoList dispatch={dispatch} />;
}
```

### Memoizing Expensive Computations

```javascript
function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, initialState);
  
  // Memoize filtered todos
  const filteredTodos = useMemo(() => {
    switch (state.filter) {
      case 'completed':
        return state.todos.filter(todo => todo.completed);
      case 'active':
        return state.todos.filter(todo => !todo.completed);
      default:
        return state.todos;
    }
  }, [state.todos, state.filter]);
  
  // Memoize statistics
  const stats = useMemo(() => ({
    total: state.todos.length,
    completed: state.todos.filter(t => t.completed).length,
    active: state.todos.filter(t => !t.completed).length
  }), [state.todos]);
  
  return (
    <div>
      <Stats stats={stats} />
      <TodoList todos={filteredTodos} dispatch={dispatch} />
    </div>
  );
}
```

## Common Mistakes

### 1. Mutating State Directly

```javascript
// ❌ WRONG: Mutating state
function reducer(state, action) {
  switch (action.type) {
    case 'ADD_TODO':
      state.todos.push(action.payload); // Mutation!
      return state;
    case 'TOGGLE_TODO':
      const todo = state.todos.find(t => t.id === action.payload.id);
      todo.completed = !todo.completed; // Mutation!
      return state;
  }
}

// ✅ CORRECT: Return new state
function reducer(state, action) {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, action.payload]
      };
    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, completed: !todo.completed }
            : todo
        )
      };
  }
}
```

### 2. Not Handling Default Case

```javascript
// ❌ WRONG: No default case
function reducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    case 'DECREMENT':
      return { count: state.count - 1 };
    // Typo in action type will silently fail!
  }
}

// ✅ CORRECT: Handle unknown actions
function reducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    case 'DECREMENT':
      return { count: state.count - 1 };
    default:
      throw new Error(`Unknown action type: ${action.type}`);
  }
}
```

### 3. Async Logic in Reducer

```javascript
// ❌ WRONG: Async operations in reducer
function reducer(state, action) {
  switch (action.type) {
    case 'FETCH_DATA':
      fetch('/api/data')
        .then(res => res.json())
        .then(data => {
          // This won't work!
          return { ...state, data };
        });
      break;
  }
}

// ✅ CORRECT: Handle async in component
function Component() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  useEffect(() => {
    dispatch({ type: 'FETCH_START' });
    
    fetch('/api/data')
      .then(res => res.json())
      .then(data => {
        dispatch({ type: 'FETCH_SUCCESS', payload: data });
      })
      .catch(error => {
        dispatch({ type: 'FETCH_ERROR', payload: error.message });
      });
  }, []);
  
  return <div>{/* ... */}</div>;
}
```

### 4. Over-Engineering Simple State

```javascript
// ❌ WRONG: useReducer for simple boolean
const [state, dispatch] = useReducer(
  (state, action) => {
    switch (action.type) {
      case 'OPEN':
        return { isOpen: true };
      case 'CLOSE':
        return { isOpen: false };
      default:
        return state;
    }
  },
  { isOpen: false }
);

// ✅ CORRECT: useState for simple values
const [isOpen, setIsOpen] = useState(false);
```

## Best Practices

### 1. Use Consistent Action Naming

```javascript
// ✅ GOOD: Consistent naming convention
const ActionTypes = {
  // Entity_Action pattern
  TODO_ADD: 'TODO_ADD',
  TODO_UPDATE: 'TODO_UPDATE',
  TODO_DELETE: 'TODO_DELETE',
  TODO_TOGGLE: 'TODO_TOGGLE',
  
  // Or past tense
  TODO_ADDED: 'TODO_ADDED',
  TODO_UPDATED: 'TODO_UPDATED',
  
  // Or verb-noun
  ADD_TODO: 'ADD_TODO',
  UPDATE_TODO: 'UPDATE_TODO'
};
```

### 2. Normalize Complex State

```javascript
// ❌ BAD: Nested arrays
const initialState = {
  users: [
    { id: 1, name: 'John', posts: [{ id: 1, title: 'Hello' }] }
  ]
};

// ✅ GOOD: Normalized structure
const initialState = {
  users: {
    byId: {
      1: { id: 1, name: 'John', postIds: [1] }
    },
    allIds: [1]
  },
  posts: {
    byId: {
      1: { id: 1, title: 'Hello', userId: 1 }
    },
    allIds: [1]
  }
};
```

### 3. Split Large Reducers

```javascript
// ✅ GOOD: Combine smaller reducers
function todosReducer(state, action) {
  // Handle todo-specific actions
}

function filtersReducer(state, action) {
  // Handle filter-specific actions
}

function uiReducer(state, action) {
  // Handle UI-specific actions
}

function mainReducer(state, action) {
  return {
    todos: todosReducer(state.todos, action),
    filters: filtersReducer(state.filters, action),
    ui: uiReducer(state.ui, action)
  };
}
```

### 4. Document State Shape

```javascript
/**
 * Application state shape
 * 
 * @typedef {Object} AppState
 * @property {Todo[]} todos - List of todos
 * @property {string} filter - Current filter ('all' | 'active' | 'completed')
 * @property {boolean} isLoading - Loading state
 * @property {string|null} error - Error message if any
 */

/**
 * @typedef {Object} Todo
 * @property {number} id - Unique identifier
 * @property {string} text - Todo text
 * @property {boolean} completed - Completion status
 * @property {number} createdAt - Creation timestamp
 */
```

## Real-World Examples

### Form Management with Validation

```javascript
const formReducer = (state, action) => {
  switch (action.type) {
    case 'FIELD_CHANGE':
      return {
        ...state,
        values: {
          ...state.values,
          [action.payload.field]: action.payload.value
        },
        touched: {
          ...state.touched,
          [action.payload.field]: true
        }
      };
    
    case 'FIELD_BLUR':
      const errors = validateField(
        action.payload.field,
        state.values[action.payload.field]
      );
      return {
        ...state,
        errors: {
          ...state.errors,
          [action.payload.field]: errors
        }
      };
    
    case 'SUBMIT_START':
      return {
        ...state,
        isSubmitting: true,
        submitError: null
      };
    
    case 'SUBMIT_SUCCESS':
      return {
        ...initialFormState,
        submitSuccess: true
      };
    
    case 'SUBMIT_ERROR':
      return {
        ...state,
        isSubmitting: false,
        submitError: action.payload.error
      };
    
    case 'RESET':
      return initialFormState;
    
    default:
      return state;
  }
};

function RegistrationForm() {
  const [state, dispatch] = useReducer(formReducer, {
    values: { email: '', password: '', confirmPassword: '' },
    errors: {},
    touched: {},
    isSubmitting: false,
    submitError: null,
    submitSuccess: false
  });

  const handleChange = (field) => (e) => {
    dispatch({
      type: 'FIELD_CHANGE',
      payload: { field, value: e.target.value }
    });
  };

  const handleBlur = (field) => () => {
    dispatch({
      type: 'FIELD_BLUR',
      payload: { field }
    });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    dispatch({ type: 'SUBMIT_START' });
    
    try {
      await registerUser(state.values);
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (error) {
      dispatch({ 
        type: 'SUBMIT_ERROR', 
        payload: { error: error.message } 
      });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={state.values.email}
        onChange={handleChange('email')}
        onBlur={handleBlur('email')}
      />
      {state.touched.email && state.errors.email && (
        <span className="error">{state.errors.email}</span>
      )}
      
      {/* More fields... */}
      
      <button 
        type="submit" 
        disabled={state.isSubmitting || Object.keys(state.errors).length > 0}
      >
        {state.isSubmitting ? 'Submitting...' : 'Register'}
      </button>
      
      {state.submitError && (
        <div className="error">{state.submitError}</div>
      )}
    </form>
  );
}
```

### Infinite Scroll with Data Loading

```javascript
const dataReducer = (state, action) => {
  switch (action.type) {
    case 'FETCH_INIT':
      return {
        ...state,
        isLoading: true,
        error: null
      };
    
    case 'FETCH_SUCCESS':
      return {
        ...state,
        isLoading: false,
        data: [...state.data, ...action.payload.data],
        hasMore: action.payload.hasMore,
        page: state.page + 1
      };
    
    case 'FETCH_ERROR':
      return {
        ...state,
        isLoading: false,
        error: action.payload.error
      };
    
    case 'RESET':
      return initialState;
    
    default:
      return state;
  }
};

function InfiniteList() {
  const [state, dispatch] = useReducer(dataReducer, {
    data: [],
    isLoading: false,
    error: null,
    page: 1,
    hasMore: true
  });

  const loadMore = useCallback(async () => {
    if (state.isLoading || !state.hasMore) return;
    
    dispatch({ type: 'FETCH_INIT' });
    
    try {
      const response = await fetchData(state.page);
      dispatch({
        type: 'FETCH_SUCCESS',
        payload: {
          data: response.data,
          hasMore: response.hasMore
        }
      });
    } catch (error) {
      dispatch({
        type: 'FETCH_ERROR',
        payload: { error: error.message }
      });
    }
  }, [state.isLoading, state.hasMore, state.page]);

  useEffect(() => {
    loadMore();
  }, []);

  return (
    <div>
      {state.data.map(item => (
        <ItemCard key={item.id} item={item} />
      ))}
      
      {state.isLoading && <Spinner />}
      {state.error && <Error message={state.error} />}
      
      {state.hasMore && !state.isLoading && (
        <button onClick={loadMore}>Load More</button>
      )}
    </div>
  );
}
```

## Interview Questions

### Q1: When should you use useReducer instead of useState?

**Answer:** Use useReducer when:
- State logic is complex (multiple sub-values, complex transitions)
- Next state depends on previous state
- You need predictable state updates
- Multiple actions update related state
- State logic needs to be tested independently
- You're managing a state machine

Example: Form with validation, shopping cart, undo/redo functionality, multi-step wizards.

### Q2: How does useReducer compare to Redux?

**Answer:**
- **Similarities:** Both use reducer pattern, actions, and predictable updates
- **Differences:**
  - useReducer is local to component (or tree via Context)
  - Redux is global across entire app
  - Redux has middleware, DevTools, time-travel debugging
  - useReducer is simpler, less boilerplate
  - Redux has better ecosystem (persistence, async handling)

Use useReducer for component-level complexity; Redux for app-level state management.

### Q3: Can you explain the dispatch function stability?

**Answer:** 
The dispatch function returned by useReducer is guaranteed to be stable across re-renders. This means:
- It has the same reference on every render
- Safe to omit from dependency arrays
- Can be passed to child components without causing re-renders
- Different from setState which is also stable

```javascript
const [state, dispatch] = useReducer(reducer, initialState);

// dispatch never changes, so this effect runs once
useEffect(() => {
  dispatch({ type: 'INIT' });
}, []); // Safe to omit dispatch
```

## Key Takeaways

1. **useReducer centralizes state logic** - All state transitions in one place
2. **Use for complex state** - Multiple related values or complex updates
3. **Dispatch is stable** - Never changes across re-renders
4. **Reducers must be pure** - No side effects, same input = same output
5. **Action types are documentation** - Make state changes explicit
6. **Combine with Context** - For sharing state across component trees
7. **TypeScript enhances safety** - Type actions and state for better DX
8. **Test reducers independently** - Pure functions are easy to test

## Resources

### Official Documentation
- [React useReducer Docs](https://react.dev/reference/react/useReducer)
- [Extracting State Logic into a Reducer](https://react.dev/learn/extracting-state-logic-into-a-reducer)

### Articles
- [When to useReducer](https://kentcdodds.com/blog/should-i-usestate-or-usereducer)
- [How to useReducer in React](https://www.robinwieruch.de/react-usereducer-hook)

### Tools
- [Immer](https://immerjs.github.io/immer/) - Immutable state updates
- [Redux DevTools](https://github.com/reduxjs/redux-devtools) - Can be used with useReducer

### Videos
- [React useReducer Hook Ultimate Guide](https://www.youtube.com/watch?v=kK_Zo25zXjQ)
- [useState vs useReducer](https://www.youtube.com/watch?v=o-nCM1857AQ)

---

**Next Steps:** Learn about useCallback and useMemo for performance optimization, then explore Context API for state sharing across components.
