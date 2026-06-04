# Immer: Immutable State Made Easy

## Overview

Immer is a tiny library that simplifies working with immutable state by letting you write code that appears to mutate data while actually creating immutable copies. Created by Michel Weststrate (creator of MobX), Immer uses JavaScript Proxies to track changes and produce immutable updates automatically.

**Key Features:**
- Write mutable-style code, get immutable updates
- Structural sharing for performance
- Automatic freezing in development
- Patches system for undo/redo functionality
- TypeScript support with excellent type inference
- Small bundle size (3KB gzipped)
- Integrates with Redux, React, and any state management
- Curried producers for reusable updates
- JSON Patch support
- No learning curve for basic usage

Immer has become the standard for immutable updates in React and Redux applications, replacing complex spread operators and deep cloning.

## Installation and Setup

### Installation

```bash
# npm
npm install immer

# yarn
yarn add immer

# pnpm
pnpm add immer
```

### Basic Usage

```javascript
import { produce } from 'immer';

// Current state
const baseState = {
  name: 'Alice',
  age: 30,
  todos: [
    { id: 1, text: 'Learn Immer', done: true },
    { id: 2, text: 'Build app', done: false }
  ]
};

// Update with Immer
const nextState = produce(baseState, draft => {
  draft.age = 31; // Looks like mutation!
  draft.todos[1].done = true; // But it's immutable
  draft.todos.push({ id: 3, text: 'Deploy', done: false });
});

console.log(baseState.age); // 30 (unchanged)
console.log(nextState.age); // 31 (new object)
console.log(baseState === nextState); // false
console.log(baseState.todos === nextState.todos); // false
console.log(baseState.name === nextState.name); // true (structural sharing)
```

## Core Concepts

### 1. Producers and Drafts

The fundamental pattern in Immer:

```javascript
import { produce } from 'immer';

// Producer function
const producer = produce((draft, increment) => {
  draft.count += increment;
});

// Use producer
const state = { count: 0 };
const nextState = producer(state, 5);

console.log(state.count); // 0
console.log(nextState.count); // 5

// Or inline
const nextState = produce(state, draft => {
  draft.count += 5;
});
```

### 2. Structural Sharing

Immer only creates new objects for changed parts:

```javascript
import { produce } from 'immer';

const state = {
  user: { name: 'Alice', age: 30 },
  settings: { theme: 'dark', notifications: true },
  todos: [1, 2, 3]
};

const nextState = produce(state, draft => {
  draft.user.age = 31; // Only user changes
});

// Only changed parts are new objects
console.log(state === nextState); // false
console.log(state.user === nextState.user); // false
console.log(state.settings === nextState.settings); // true (unchanged)
console.log(state.todos === nextState.todos); // true (unchanged)

// Memory efficient!
```

### 3. Returning Values

You can return a value to replace the entire draft:

```javascript
import { produce } from 'immer';

const state = { count: 0 };

// Modify draft (implicit return)
const next1 = produce(state, draft => {
  draft.count = 5;
});

// Return new value (explicit return)
const next2 = produce(state, draft => {
  return { count: 10 }; // Replaces entire state
});

// Return undefined still uses draft
const next3 = produce(state, draft => {
  draft.count = 15;
  return undefined; // Uses draft
});

// Return nothing from draft
const next4 = produce(state, draft => {
  return draft.count > 0 ? undefined : { count: -1 };
});
```

## Working with Different Data Types

### Arrays

```javascript
import { produce } from 'immer';

const todos = [
  { id: 1, text: 'Learn Immer', done: true },
  { id: 2, text: 'Build app', done: false },
  { id: 3, text: 'Deploy', done: false }
];

// Add item
const withNew = produce(todos, draft => {
  draft.push({ id: 4, text: 'Test', done: false });
});

// Remove item
const withoutSecond = produce(todos, draft => {
  draft.splice(1, 1); // Remove index 1
});

// Update item
const withUpdated = produce(todos, draft => {
  const todo = draft.find(t => t.id === 2);
  if (todo) {
    todo.done = true;
  }
});

// Filter (return new array)
const onlyActive = produce(todos, draft => {
  return draft.filter(t => !t.done);
});

// Sort (mutate draft)
const sorted = produce(todos, draft => {
  draft.sort((a, b) => a.text.localeCompare(b.text));
});

// Map (return new array)
const toggled = produce(todos, draft => {
  return draft.map(t => ({ ...t, done: !t.done }));
});
```

### Maps and Sets

```javascript
import { produce } from 'immer';

// Maps
const userMap = new Map([
  [1, { name: 'Alice', age: 30 }],
  [2, { name: 'Bob', age: 25 }]
]);

const updatedMap = produce(userMap, draft => {
  const user = draft.get(1);
  if (user) {
    user.age = 31;
  }
  draft.set(3, { name: 'Charlie', age: 35 });
  draft.delete(2);
});

console.log(userMap.size); // 2
console.log(updatedMap.size); // 2 (Alice updated, Bob deleted, Charlie added)

// Sets
const tags = new Set(['react', 'immer', 'redux']);

const updatedTags = produce(tags, draft => {
  draft.add('typescript');
  draft.delete('redux');
});

console.log(updatedTags.has('typescript')); // true
console.log(updatedTags.has('redux')); // false
```

### Nested Objects

```javascript
import { produce } from 'immer';

const state = {
  user: {
    profile: {
      personal: {
        name: 'Alice',
        age: 30
      },
      settings: {
        theme: 'dark',
        notifications: {
          email: true,
          push: false
        }
      }
    }
  }
};

// Deep update without spread operators!
const updated = produce(state, draft => {
  draft.user.profile.personal.age = 31;
  draft.user.profile.settings.notifications.push = true;
});

// Compare to traditional approach
const traditionalUpdate = {
  ...state,
  user: {
    ...state.user,
    profile: {
      ...state.user.profile,
      personal: {
        ...state.user.profile.personal,
        age: 31
      },
      settings: {
        ...state.user.profile.settings,
        notifications: {
          ...state.user.profile.settings.notifications,
          push: true
        }
      }
    }
  }
};
```

## React Integration

### useState with Immer

```javascript
import { useState } from 'react';
import { produce } from 'immer';

function TodoApp() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn Immer', done: false }
  ]);

  const addTodo = (text) => {
    setTodos(produce(draft => {
      draft.push({ id: Date.now(), text, done: false });
    }));
  };

  const toggleTodo = (id) => {
    setTodos(produce(draft => {
      const todo = draft.find(t => t.id === id);
      if (todo) {
        todo.done = !todo.done;
      }
    }));
  };

  const deleteTodo = (id) => {
    setTodos(produce(draft => {
      const index = draft.findIndex(t => t.id === id);
      if (index !== -1) {
        draft.splice(index, 1);
      }
    }));
  };

  return (
    <div>
      {todos.map(todo => (
        <div key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => toggleTodo(todo.id)}
          />
          <span>{todo.text}</span>
          <button onClick={() => deleteTodo(todo.id)}>Delete</button>
        </div>
      ))}
      <button onClick={() => addTodo('New Todo')}>Add</button>
    </div>
  );
}
```

### useImmer Hook

```javascript
import { useImmer } from 'use-immer';

function TodoApp() {
  const [todos, updateTodos] = useImmer([
    { id: 1, text: 'Learn Immer', done: false }
  ]);

  const addTodo = (text) => {
    updateTodos(draft => {
      draft.push({ id: Date.now(), text, done: false });
    });
  };

  const toggleTodo = (id) => {
    updateTodos(draft => {
      const todo = draft.find(t => t.id === id);
      if (todo) {
        todo.done = !todo.done;
      }
    });
  };

  return (
    <div>
      {/* Same JSX as above */}
    </div>
  );
}
```

### useReducer with Immer

```javascript
import { useReducer } from 'react';
import { produce } from 'immer';

const initialState = {
  todos: [],
  filter: 'all'
};

function reducer(state, action) {
  return produce(state, draft => {
    switch (action.type) {
      case 'ADD_TODO':
        draft.todos.push({
          id: Date.now(),
          text: action.payload,
          done: false
        });
        break;
      
      case 'TOGGLE_TODO':
        const todo = draft.todos.find(t => t.id === action.payload);
        if (todo) {
          todo.done = !todo.done;
        }
        break;
      
      case 'DELETE_TODO':
        const index = draft.todos.findIndex(t => t.id === action.payload);
        if (index !== -1) {
          draft.todos.splice(index, 1);
        }
        break;
      
      case 'SET_FILTER':
        draft.filter = action.payload;
        break;
    }
  });
}

function TodoApp() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <div>
      {/* Use state and dispatch */}
    </div>
  );
}
```

## Redux Integration

### Redux Toolkit (Built-in)

```javascript
import { createSlice } from '@reduxjs/toolkit';

// Redux Toolkit uses Immer internally
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      // Write "mutating" code!
      state.push({
        id: Date.now(),
        text: action.payload,
        done: false
      });
    },
    toggleTodo: (state, action) => {
      const todo = state.find(t => t.id === action.payload);
      if (todo) {
        todo.done = !todo.done;
      }
    },
    deleteTodo: (state, action) => {
      return state.filter(t => t.id !== action.payload);
    }
  }
});

export const { addTodo, toggleTodo, deleteTodo } = todosSlice.actions;
export default todosSlice.reducer;
```

### Plain Redux

```javascript
import { produce } from 'immer';

const initialState = {
  todos: [],
  loading: false,
  error: null
};

function todosReducer(state = initialState, action) {
  return produce(state, draft => {
    switch (action.type) {
      case 'FETCH_TODOS_START':
        draft.loading = true;
        draft.error = null;
        break;
      
      case 'FETCH_TODOS_SUCCESS':
        draft.loading = false;
        draft.todos = action.payload;
        break;
      
      case 'FETCH_TODOS_FAILURE':
        draft.loading = false;
        draft.error = action.payload;
        break;
      
      case 'ADD_TODO':
        draft.todos.push(action.payload);
        break;
      
      case 'UPDATE_TODO':
        const index = draft.todos.findIndex(t => t.id === action.payload.id);
        if (index !== -1) {
          draft.todos[index] = action.payload;
        }
        break;
      
      case 'DELETE_TODO':
        const deleteIndex = draft.todos.findIndex(t => t.id === action.payload);
        if (deleteIndex !== -1) {
          draft.todos.splice(deleteIndex, 1);
        }
        break;
    }
  });
}
```

## Patches for Undo/Redo

### Generating Patches

```javascript
import { produceWithPatches, applyPatches } from 'immer';

const baseState = {
  name: 'Alice',
  age: 30,
  todos: [
    { id: 1, text: 'Learn Immer', done: false }
  ]
};

// Generate patches
const [nextState, patches, inversePatches] = produceWithPatches(
  baseState,
  draft => {
    draft.age = 31;
    draft.todos[0].done = true;
    draft.todos.push({ id: 2, text: 'Build app', done: false });
  }
);

console.log(patches);
// [
//   { op: 'replace', path: ['age'], value: 31 },
//   { op: 'replace', path: ['todos', 0, 'done'], value: true },
//   { op: 'add', path: ['todos', 1], value: { id: 2, text: 'Build app', done: false } }
// ]

console.log(inversePatches);
// [
//   { op: 'replace', path: ['age'], value: 30 },
//   { op: 'replace', path: ['todos', 0, 'done'], value: false },
//   { op: 'remove', path: ['todos', 1] }
// ]

// Apply patches
const redone = applyPatches(baseState, patches);
console.log(redone === nextState); // true (equivalent)

// Undo with inverse patches
const undone = applyPatches(nextState, inversePatches);
console.log(undone); // Back to baseState
```

### Undo/Redo Implementation

```javascript
import { produceWithPatches, applyPatches } from 'immer';
import { useState } from 'react';

function useUndoable(initialState) {
  const [state, setState] = useState(initialState);
  const [history, setHistory] = useState({
    past: [],
    future: []
  });

  const updateState = (updater) => {
    const [nextState, patches, inversePatches] = produceWithPatches(
      state,
      updater
    );

    setState(nextState);
    setHistory({
      past: [...history.past, { state, patches: inversePatches }],
      future: [] // Clear future on new action
    });
  };

  const undo = () => {
    if (history.past.length === 0) return;

    const { past, future } = history;
    const { state: previousState, patches } = past[past.length - 1];

    setState(previousState);
    setHistory({
      past: past.slice(0, -1),
      future: [{ state, patches }, ...future]
    });
  };

  const redo = () => {
    if (history.future.length === 0) return;

    const { past, future } = history;
    const { state: nextState, patches } = future[0];

    setState(nextState);
    setHistory({
      past: [...past, { state, patches }],
      future: future.slice(1)
    });
  };

  return {
    state,
    updateState,
    undo,
    redo,
    canUndo: history.past.length > 0,
    canRedo: history.future.length > 0
  };
}

// Usage
function DrawingApp() {
  const {
    state,
    updateState,
    undo,
    redo,
    canUndo,
    canRedo
  } = useUndoable({ shapes: [] });

  const addShape = (shape) => {
    updateState(draft => {
      draft.shapes.push(shape);
    });
  };

  return (
    <div>
      <button onClick={undo} disabled={!canUndo}>Undo</button>
      <button onClick={redo} disabled={!canRedo}>Redo</button>
      <button onClick={() => addShape({ type: 'circle', x: 100, y: 100 })}>
        Add Circle
      </button>
      {/* Render shapes */}
    </div>
  );
}
```

## Advanced Patterns

### Curried Producers

```javascript
import { produce } from 'immer';

// Create reusable updaters
const updateAge = produce((draft, newAge) => {
  draft.age = newAge;
});

const updateName = produce((draft, newName) => {
  draft.name = newName;
});

const addTodo = produce((draft, text) => {
  draft.todos.push({ id: Date.now(), text, done: false });
});

// Use them
const user = { name: 'Alice', age: 30, todos: [] };

const user1 = updateAge(user, 31);
const user2 = updateName(user1, 'Bob');
const user3 = addTodo(user2, 'Learn Immer');
```

### Recipe Pattern

```javascript
import { produce } from 'immer';

// Create recipe functions
const recipes = {
  addTodo: (text) => (draft) => {
    draft.todos.push({ id: Date.now(), text, done: false });
  },

  toggleTodo: (id) => (draft) => {
    const todo = draft.todos.find(t => t.id === id);
    if (todo) {
      todo.done = !todo.done;
    }
  },

  updateSettings: (settings) => (draft) => {
    Object.assign(draft.settings, settings);
  }
};

// Apply recipe
const state = { todos: [], settings: {} };
const nextState = produce(state, recipes.addTodo('Learn Immer'));

// Compose recipes
const composed = produce(state, draft => {
  recipes.addTodo('First todo')(draft);
  recipes.addTodo('Second todo')(draft);
  recipes.updateSettings({ theme: 'dark' })(draft);
});
```

### Type-Safe Updates

```typescript
import { produce } from 'immer';

interface Todo {
  id: number;
  text: string;
  done: boolean;
}

interface State {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
}

// Type-safe producer
const addTodo = (text: string) =>
  produce((draft: State) => {
    draft.todos.push({
      id: Date.now(),
      text,
      done: false
    });
  });

// Type-safe recipe
type Recipe<T> = (draft: T) => void;

function applyRecipe<T>(state: T, recipe: Recipe<T>): T {
  return produce(state, recipe);
}

const state: State = { todos: [], filter: 'all' };
const next = applyRecipe(state, draft => {
  draft.filter = 'active'; // Type-safe
  // draft.filter = 'invalid'; // ❌ Type error
});
```

## Performance Optimization

### Freezing in Development

```javascript
import { setAutoFreeze } from 'immer';

// Default: true in development, false in production
setAutoFreeze(true);

const state = produce({ count: 0 }, draft => {
  draft.count = 1;
});

// In development, state is frozen
state.count = 2; // ❌ Error: Cannot assign to read only property

// Disable for performance in large apps
setAutoFreeze(false);
```

### Using ES5 Fallback

```javascript
import { setUseProxies } from 'immer';

// Default: true (use Proxies)
// Disable for older environments
setUseProxies(false); // Falls back to ES5 implementation

// Note: ES5 fallback is slower and has limitations
```

### Structural Sharing Benefits

```javascript
import { produce } from 'immer';

const largeState = {
  users: new Array(10000).fill(null).map((_, i) => ({
    id: i,
    name: `User ${i}`
  })),
  settings: { theme: 'dark' },
  config: { /* large object */ }
};

// Only update one user
const updated = produce(largeState, draft => {
  draft.users[0].name = 'Alice';
});

// Structural sharing means:
// - Only users array is new
// - Only users[0] is new
// - All other users are same reference
// - settings and config are same reference
// Memory efficient!
```

## Common Mistakes

### 1. Returning Modified Draft

```javascript
// Wrong: Returning modified draft
const wrong = produce(state, draft => {
  draft.count++;
  return draft; // ❌ Don't return draft!
});

// Correct: Either modify draft OR return value
const correct1 = produce(state, draft => {
  draft.count++; // Implicit return of draft
});

const correct2 = produce(state, draft => {
  return { count: state.count + 1 }; // Explicit return
});
```

### 2. Mixing Mutating and Non-Mutating

```javascript
// Wrong: Mixing approaches
const wrong = produce(todos, draft => {
  draft[0].done = true; // Mutating
  return draft.filter(t => !t.done); // Non-mutating
  // Returning value discards mutations!
});

// Correct: Use one approach
const correct1 = produce(todos, draft => {
  draft[0].done = true;
  // Mutations applied
});

const correct2 = produce(todos, draft => {
  return draft
    .map((t, i) => i === 0 ? { ...t, done: true } : t)
    .filter(t => !t.done);
  // Full replacement
});
```

### 3. Forgetting to Use Returned Value

```javascript
// Wrong: Not using returned value
const state = { count: 0 };
produce(state, draft => {
  draft.count++;
});
console.log(state.count); // Still 0! ❌

// Correct: Use returned value
const state = { count: 0 };
const nextState = produce(state, draft => {
  draft.count++;
});
console.log(nextState.count); // 1 ✓
```

### 4. Async Producers

```javascript
// Wrong: Async producer
const wrong = produce(state, async draft => {
  const data = await fetchData();
  draft.data = data;
});
// Returns Promise, not state!

// Correct: Update separately
async function updateWithData(state) {
  const data = await fetchData();
  return produce(state, draft => {
    draft.data = data;
  });
}
```

## Best Practices

### 1. Use for Complex Updates

```javascript
// Good: Complex nested update
const updated = produce(state, draft => {
  draft.user.profile.settings.notifications.email = true;
  draft.user.todos.push({ text: 'New todo' });
  draft.lastUpdated = Date.now();
});

// Not needed: Simple update
// const updated = produce(state, draft => {
//   draft.count++;
// });
// Better: const updated = { ...state, count: state.count + 1 };
```

### 2. Create Reusable Producers

```javascript
// Create library of updaters
const updaters = {
  setLoading: produce((draft, loading) => {
    draft.loading = loading;
  }),

  setError: produce((draft, error) => {
    draft.error = error;
    draft.loading = false;
  }),

  setData: produce((draft, data) => {
    draft.data = data;
    draft.loading = false;
    draft.error = null;
  })
};

// Use throughout app
const state1 = updaters.setLoading(state, true);
const state2 = updaters.setData(state1, data);
```

### 3. Use with TypeScript

```typescript
import { produce, Draft } from 'immer';

interface State {
  count: number;
  items: string[];
}

function increment(state: State): State {
  return produce(state, (draft: Draft<State>) => {
    draft.count++; // Type-safe
  });
}
```

### 4. Enable Patches for Debugging

```javascript
import { produceWithPatches } from 'immer';

const [nextState, patches, inversePatches] = produceWithPatches(
  state,
  draft => {
    // Complex updates
  }
);

console.log('Changes:', patches);
// See exactly what changed for debugging
```

## Interview Questions

### 1. How does Immer achieve immutability while allowing mutable-style code?

**Answer:** Immer uses JavaScript Proxies to wrap the original state in a draft object. When you modify the draft, Immer records all changes. After the producer function completes, Immer creates a new state with only changed parts copied (structural sharing). Unchanged parts reference the original objects. The Proxy intercepts all mutations, tracks them, and applies them to create an immutable result. This allows natural mutation syntax while maintaining immutability.

### 2. What is structural sharing and why is it important?

**Answer:** Structural sharing means unchanged parts of the state tree reuse references from the original state rather than being copied. Only modified paths create new objects. This is crucial for performance in large state trees and enables efficient React reconciliation (unchanged objects won't trigger re-renders). For example, updating one todo in an array of thousands only creates new objects for that todo and the array itself, not all other todos.

### 3. When should you return a value vs modifying the draft?

**Answer:** Modify the draft for incremental updates and leveraging structural sharing. Return a value to completely replace the state or when using functions like filter/map that return new arrays. Don't mix both in the same producer (return discards draft modifications). Generally, prefer draft modifications for most updates unless you're computing an entirely new state. Explicit returns are clearer for transformations that don't build on existing state.

### 4. How do patches enable undo/redo functionality?

**Answer:** `produceWithPatches` returns the next state plus patches (changes made) and inverse patches (changes to reverse). Patches follow JSON Patch format describing operations (add, remove, replace). To undo, apply inverse patches to current state. To redo, reapply original patches. This creates an efficient undo/redo system without storing full state copies. Each action stores only the differences, making history memory-efficient even for large states.

### 5. What are the performance implications of using Immer?

**Answer:** Immer adds minimal overhead (microseconds per produce call) through Proxy wrapping and change tracking. Benefits include structural sharing reducing memory usage and preventing unnecessary React re-renders. Immer is slower than direct object creation but much faster than deep cloning or complex spread operations. For most applications, the performance cost is negligible compared to developer productivity gains and reduced bugs. Use setAutoFreeze(false) in production for maximum performance.

### 6. Can Immer be used with any state management library?

**Answer:** Yes, Immer is framework-agnostic and works with any state container. It's built into Redux Toolkit by default. Use with plain Redux via produce in reducers. Works with React useState, useReducer, or zustand. Compatible with any JavaScript state management. The only requirement is that state must be serializable (no functions, classes with methods). Immer handles plain objects, arrays, Maps, and Sets.

### 7. What are the limitations of Immer?

**Answer:** Immer requires Proxy support (or ES5 fallback with limitations). It doesn't work with non-serializable data (class instances with methods, functions). Async producers aren't supported. Performance overhead exists for very high-frequency updates. Draft modifications and explicit returns shouldn't be mixed. Classes need special handling or should be avoided. For most applications these limitations are acceptable, but high-performance apps or old browsers might need alternatives.

## Key Takeaways

1. **Write mutable code** that produces immutable results through Proxy tracking
2. **Structural sharing** ensures only changed parts are copied for efficiency
3. **Patches system** enables undo/redo and change tracking
4. **Curried producers** create reusable state updaters
5. **TypeScript support** provides excellent type inference for draft objects
6. **Small bundle size** (3KB) makes it practical for any project
7. **Redux Toolkit** includes Immer by default for all reducers
8. **React integration** via useImmer or direct useState/useReducer usage
9. **Performance** is excellent due to structural sharing and minimal overhead
10. **Developer experience** dramatically improved compared to spread operators

## Resources

- **Official Documentation**: https://immerjs.github.io/immer/
- **GitHub Repository**: https://github.com/immerjs/immer
- **use-immer**: React hooks package
- **Redux Toolkit**: Built-in Immer integration
- **TypeScript Guide**: Using Immer with TypeScript
- **Performance Benchmarks**: Comparing to other immutability solutions
- **Egghead Course**: Introduction to Immer
- **Migration Guide**: From immutability helpers to Immer
- **Patches Documentation**: Advanced undo/redo patterns
- **Best Practices**: Common patterns and anti-patterns
