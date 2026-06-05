# State DevTools

## The Idea

**In plain English:** State DevTools are special browser tools that let you watch and rewind all the changes happening inside your app's memory (called "state") as you use it — like having a DVR for your app's data so you can pause, rewind, and inspect exactly what happened and when.

**Real-world analogy:** Imagine a security camera system at a store that records every shelf rearrangement made by staff throughout the day. At the end of the day, the manager can scrub back through the footage, pause on any moment, and see exactly which shelf looked like what at any point in time.

- The security camera footage = the recorded history of state snapshots in DevTools
- Each shelf rearrangement by staff = a dispatched action that changes app state
- The manager scrubbing through footage = a developer using time-travel debugging to jump between past states

---

## Overview

State DevTools are essential debugging tools that provide visibility into application state changes, enable time-travel debugging, action replay, state inspection, and performance monitoring. The Redux DevTools extension is the most popular, but similar tools exist for other state management libraries. Proper integration of devtools dramatically improves development experience, reduces debugging time, and helps understand complex state flows in production issues.

## Core Concepts

### DevTools Capabilities

```
┌──────────────────────────────────────────────────────────┐
│            State DevTools Features                        │
└──────────────────────────────────────────────────────────┘

Time-Travel Debugging
├─ Scrub through state history
├─ Jump to any previous state
├─ See state at any point in time
└─ Replay actions from any point

Action Inspection
├─ View all dispatched actions
├─ Inspect action payloads
├─ Filter actions by type
└─ Export/import action logs

State Diff
├─ See what changed between states
├─ Highlight modified values
├─ Track nested object changes
└─ Monitor array mutations

Performance
├─ Measure action duration
├─ Track render performance
├─ Identify slow updates
└─ Profile state changes
```

## Redux DevTools

### Basic Redux DevTools Setup

```typescript
import { createStore, applyMiddleware, compose } from 'redux';
import thunk from 'redux-thunk';

// Type definition for window with DevTools
declare global {
  interface Window {
    __REDUX_DEVTOOLS_EXTENSION_COMPOSE__?: typeof compose;
  }
}

const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;

const store = createStore(
  rootReducer,
  composeEnhancers(applyMiddleware(thunk))
);

export default store;
```

### Redux Toolkit with DevTools

```typescript
import { configureStore } from '@reduxjs/toolkit';
import userReducer from './features/userSlice';
import todosReducer from './features/todosSlice';

const store = configureStore({
  reducer: {
    user: userReducer,
    todos: todosReducer
  },
  // DevTools automatically enabled in development
  devTools: process.env.NODE_ENV !== 'production'
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export default store;
```

### Advanced DevTools Configuration

```typescript
import { configureStore } from '@reduxjs/toolkit';

const store = configureStore({
  reducer: rootReducer,
  devTools: {
    // Limit number of actions stored
    maxAge: 50,
    
    // Trace where actions are dispatched from
    trace: true,
    traceLimit: 25,
    
    // Custom action sanitizer - hide sensitive data
    actionSanitizer: (action) => {
      if (action.type === 'user/setCredentials') {
        return {
          ...action,
          payload: {
            ...action.payload,
            password: '***HIDDEN***'
          }
        };
      }
      return action;
    },
    
    // Custom state sanitizer
    stateSanitizer: (state) => ({
      ...state,
      user: state.user ? {
        ...state.user,
        token: '***HIDDEN***'
      } : null
    }),
    
    // Customize action names for better readability
    actionsDenylist: ['@@INIT', '@@REPLACE'],
    
    // Features to enable/disable
    features: {
      pause: true,
      lock: true,
      persist: true,
      export: true,
      import: 'custom',
      jump: true,
      skip: true,
      reorder: true,
      dispatch: true,
      test: true
    }
  }
});
```

### Action Creators with DevTools Metadata

```typescript
import { createAction } from '@reduxjs/toolkit';

// Add metadata for better DevTools display
const addTodo = createAction('todos/add', function prepare(text: string) {
  return {
    payload: {
      id: Date.now().toString(),
      text,
      completed: false,
      createdAt: new Date().toISOString()
    },
    meta: {
      timestamp: Date.now(),
      source: 'user-input'
    }
  };
});

// Custom action with trace information
function dispatchWithTrace(action: any) {
  const error = new Error();
  const stack = error.stack?.split('\n').slice(2, 4).join('\n');
  
  return {
    ...action,
    meta: {
      ...action.meta,
      trace: stack
    }
  };
}

// Usage
dispatch(dispatchWithTrace(addTodo('Buy milk')));
```

## Zustand DevTools

### Zustand with DevTools Integration

```typescript
import create from 'zustand';
import { devtools } from 'zustand/middleware';

interface TodoStore {
  todos: Array<{ id: string; text: string; completed: boolean }>;
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
  deleteTodo: (id: string) => void;
}

const useTodoStore = create<TodoStore>()(
  devtools(
    (set) => ({
      todos: [],
      
      addTodo: (text) =>
        set(
          (state) => ({
            todos: [
              ...state.todos,
              { id: Date.now().toString(), text, completed: false }
            ]
          }),
          false, // Don't replace state
          'todos/add' // Action name in DevTools
        ),
      
      toggleTodo: (id) =>
        set(
          (state) => ({
            todos: state.todos.map((todo) =>
              todo.id === id ? { ...todo, completed: !todo.completed } : todo
            )
          }),
          false,
          { type: 'todos/toggle', id } // Structured action
        ),
      
      deleteTodo: (id) =>
        set(
          (state) => ({
            todos: state.todos.filter((todo) => todo.id !== id)
          }),
          false,
          'todos/delete'
        )
    }),
    {
      name: 'TodoStore', // Store name in DevTools
      enabled: process.env.NODE_ENV === 'development'
    }
  )
);
```

### Multiple Zustand Stores with DevTools

```typescript
import create from 'zustand';
import { devtools } from 'zustand/middleware';

// User store
const useUserStore = create(
  devtools(
    (set) => ({
      user: null,
      login: (user) => set({ user }, false, 'user/login'),
      logout: () => set({ user: null }, false, 'user/logout')
    }),
    { name: 'UserStore' }
  )
);

// Settings store
const useSettingsStore = create(
  devtools(
    (set) => ({
      theme: 'light',
      setTheme: (theme) => set({ theme }, false, 'settings/setTheme')
    }),
    { name: 'SettingsStore' }
  )
);

// Combined DevTools view
function App() {
  const user = useUserStore(state => state.user);
  const theme = useSettingsStore(state => state.theme);
  
  // Both stores visible separately in DevTools
  return <div>{/* App content */}</div>;
}
```

## MobX DevTools

### MobX DevTools Setup

```typescript
import { configure } from 'mobx';
import { enableLogging } from 'mobx-logger';

// Enable strict mode for better debugging
configure({
  enforceActions: 'always',
  computedRequiresReaction: true,
  reactionRequiresObservable: true,
  observableRequiresReaction: true,
  disableErrorBoundaries: false
});

// Enable logging in development
if (process.env.NODE_ENV === 'development') {
  enableLogging({
    action: true,
    reaction: true,
    transaction: true,
    compute: true
  });
}
```

### MobX Store with Tracing

```typescript
import { makeObservable, observable, action, computed, trace } from 'mobx';

class TodoStore {
  todos = [];
  
  constructor() {
    makeObservable(this, {
      todos: observable,
      addTodo: action,
      completedCount: computed
    });
  }
  
  addTodo(text: string) {
    // Add tracing for debugging
    trace();
    
    this.todos.push({
      id: Date.now().toString(),
      text,
      completed: false
    });
  }
  
  get completedCount() {
    // Trace when computed value is accessed
    trace();
    return this.todos.filter(t => t.completed).length;
  }
}

// Spy on all MobX events
import { spy } from 'mobx';

if (process.env.NODE_ENV === 'development') {
  spy((event) => {
    if (event.type === 'action') {
      console.log(`[Action] ${event.name}`, event.arguments);
    } else if (event.type === 'reaction') {
      console.log(`[Reaction] ${event.name}`);
    } else if (event.type === 'compute') {
      console.log(`[Compute] ${event.name}`);
    }
  });
}
```

## React DevTools State Integration

### Custom Hook with DevTools Debug Value

```typescript
import { useDebugValue, useState, useEffect } from 'react';

function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  // Show custom label in React DevTools
  useDebugValue(isOnline ? 'Online' : 'Offline');
  
  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);
  
  return isOnline;
}

// Complex debug value with formatting
function useUserData(userId: string) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useDebugValue(
    user,
    (user) => user ? `User: ${user.name} (${user.email})` : 'Loading...'
  );
  
  // ... fetch logic
  
  return { user, loading };
}
```

### Profiling Components

```typescript
import React, { Profiler } from 'react';

function onRenderCallback(
  id: string,
  phase: 'mount' | 'update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number
) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
  
  // Log to analytics in production
  if (actualDuration > 100) {
    console.warn(`Slow render detected in ${id}`);
  }
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <div>
        <Profiler id="TodoList" onRender={onRenderCallback}>
          <TodoList />
        </Profiler>
        <Profiler id="Sidebar" onRender={onRenderCallback}>
          <Sidebar />
        </Profiler>
      </div>
    </Profiler>
  );
}
```

## Angular State DevTools

### NgRx DevTools

```typescript
import { StoreDevtoolsModule } from '@ngrx/store-devtools';
import { environment } from '../environments/environment';

@NgModule({
  imports: [
    StoreModule.forRoot(reducers),
    StoreDevtoolsModule.instrument({
      maxAge: 25, // Retain last 25 states
      logOnly: environment.production, // Only log in production
      autoPause: true, // Pause recording when window loses focus
      trace: true, // Include stack trace for actions
      traceLimit: 75, // Maximum stack trace frames
      
      // Custom action sanitizer
      actionSanitizer: (action, id) => {
        if (action.type === '[Auth] Login Success') {
          return {
            ...action,
            token: '***REDACTED***'
          };
        }
        return action;
      },
      
      // Custom state sanitizer
      stateSanitizer: (state, index) => ({
        ...state,
        auth: state.auth ? {
          ...state.auth,
          token: '***REDACTED***'
        } : null
      })
    })
  ]
})
export class AppModule {}
```

### Custom DevTools for Angular Services

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

interface DevToolsAction {
  type: string;
  payload?: any;
  timestamp: number;
  trace?: string;
}

@Injectable({ providedIn: 'root' })
export class DevToolsService {
  private actions$ = new BehaviorSubject<DevToolsAction[]>([]);
  private enabled = !environment.production;
  
  logAction(type: string, payload?: any) {
    if (!this.enabled) return;
    
    const trace = new Error().stack?.split('\n').slice(2, 5).join('\n');
    
    const action: DevToolsAction = {
      type,
      payload,
      timestamp: Date.now(),
      trace
    };
    
    const currentActions = this.actions$.value;
    this.actions$.next([...currentActions, action]);
    
    // Log to console with grouping
    console.groupCollapsed(`[Action] ${type}`);
    console.log('Payload:', payload);
    console.log('Trace:', trace);
    console.groupEnd();
  }
  
  getActions() {
    return this.actions$.asObservable();
  }
  
  clearActions() {
    this.actions$.next([]);
  }
  
  exportActions() {
    const actions = this.actions$.value;
    const json = JSON.stringify(actions, null, 2);
    const blob = new Blob([json], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    
    const link = document.createElement('a');
    link.href = url;
    link.download = `actions-${Date.now()}.json`;
    link.click();
  }
}

// Usage in services
@Injectable({ providedIn: 'root' })
export class TodoService {
  constructor(private devTools: DevToolsService) {}
  
  addTodo(text: string) {
    this.devTools.logAction('ADD_TODO', { text });
    // ... actual logic
  }
}
```

## Custom DevTools Implementation

### Building a Simple State Monitor

```typescript
class StateMonitor {
  private states: any[] = [];
  private maxStates = 50;
  private listeners = new Set<Function>();
  
  record(state: any, action?: string) {
    this.states.push({
      state: JSON.parse(JSON.stringify(state)),
      action,
      timestamp: Date.now(),
      trace: new Error().stack
    });
    
    // Limit stored states
    if (this.states.length > this.maxStates) {
      this.states.shift();
    }
    
    this.notifyListeners();
  }
  
  subscribe(listener: Function) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
  
  private notifyListeners() {
    this.listeners.forEach(listener => listener(this.states));
  }
  
  getStates() {
    return this.states;
  }
  
  jumpToState(index: number) {
    if (index >= 0 && index < this.states.length) {
      return this.states[index].state;
    }
    return null;
  }
  
  exportStates() {
    return JSON.stringify(this.states, null, 2);
  }
  
  importStates(json: string) {
    try {
      this.states = JSON.parse(json);
      this.notifyListeners();
    } catch (error) {
      console.error('Failed to import states:', error);
    }
  }
  
  clear() {
    this.states = [];
    this.notifyListeners();
  }
}

// React hook
function useStateMonitor() {
  const monitor = useRef(new StateMonitor());
  const [states, setStates] = useState([]);
  
  useEffect(() => {
    return monitor.current.subscribe(setStates);
  }, []);
  
  return monitor.current;
}

// Usage
function DebugPanel() {
  const monitor = useStateMonitor();
  const [selectedIndex, setSelectedIndex] = useState(-1);
  
  return (
    <div className="debug-panel">
      <h3>State History</h3>
      <button onClick={() => monitor.clear()}>Clear</button>
      <button onClick={() => {
        const json = monitor.exportStates();
        navigator.clipboard.writeText(json);
      }}>
        Copy to Clipboard
      </button>
      
      <div className="state-list">
        {monitor.getStates().map((entry, index) => (
          <div
            key={index}
            className={selectedIndex === index ? 'selected' : ''}
            onClick={() => setSelectedIndex(index)}
          >
            <span>{entry.action || 'Unknown'}</span>
            <span>{new Date(entry.timestamp).toLocaleTimeString()}</span>
          </div>
        ))}
      </div>
      
      {selectedIndex >= 0 && (
        <div className="state-detail">
          <pre>
            {JSON.stringify(
              monitor.getStates()[selectedIndex].state,
              null,
              2
            )}
          </pre>
        </div>
      )}
    </div>
  );
}
```

### Action Replay System

```typescript
interface RecordedAction {
  type: string;
  payload: any;
  timestamp: number;
  delay: number; // Time since previous action
}

class ActionRecorder {
  private recording = false;
  private actions: RecordedAction[] = [];
  private lastActionTime = 0;
  
  startRecording() {
    this.recording = true;
    this.actions = [];
    this.lastActionTime = Date.now();
  }
  
  stopRecording() {
    this.recording = false;
  }
  
  recordAction(type: string, payload: any) {
    if (!this.recording) return;
    
    const now = Date.now();
    const delay = this.lastActionTime ? now - this.lastActionTime : 0;
    
    this.actions.push({
      type,
      payload,
      timestamp: now,
      delay
    });
    
    this.lastActionTime = now;
  }
  
  async replay(dispatch: Function) {
    for (const action of this.actions) {
      if (action.delay > 0) {
        await new Promise(resolve => setTimeout(resolve, action.delay));
      }
      dispatch({ type: action.type, payload: action.payload });
    }
  }
  
  exportRecording() {
    return JSON.stringify(this.actions, null, 2);
  }
  
  importRecording(json: string) {
    try {
      this.actions = JSON.parse(json);
    } catch (error) {
      console.error('Failed to import recording:', error);
    }
  }
}

// Usage
const recorder = new ActionRecorder();

// Middleware to record actions
const recordingMiddleware = store => next => action => {
  recorder.recordAction(action.type, action.payload);
  return next(action);
};

function RecorderControls() {
  const [isRecording, setIsRecording] = useState(false);
  const dispatch = useDispatch();
  
  return (
    <div>
      <button onClick={() => {
        if (isRecording) {
          recorder.stopRecording();
        } else {
          recorder.startRecording();
        }
        setIsRecording(!isRecording);
      }}>
        {isRecording ? 'Stop Recording' : 'Start Recording'}
      </button>
      
      <button onClick={() => recorder.replay(dispatch)}>
        Replay Actions
      </button>
      
      <button onClick={() => {
        const json = recorder.exportRecording();
        downloadFile('recording.json', json);
      }}>
        Export
      </button>
    </div>
  );
}
```

## Production Monitoring

### Error Tracking with State Context

```typescript
import * as Sentry from '@sentry/react';

// Attach Redux state to error reports
const sentryReduxEnhancer = Sentry.createReduxEnhancer({
  // Optional: Customize state sanitization
  stateTransformer: (state) => ({
    ...state,
    user: state.user ? {
      ...state.user,
      token: '[REDACTED]'
    } : null
  }),
  
  // Optional: Only attach certain slices
  actionTransformer: (action) => {
    if (action.type === 'user/setPassword') {
      return {
        ...action,
        payload: '[REDACTED]'
      };
    }
    return action;
  }
});

const store = createStore(
  rootReducer,
  composeEnhancers(
    applyMiddleware(thunk),
    sentryReduxEnhancer
  )
);
```

### Performance Monitoring

```typescript
const performanceMiddleware = store => next => action => {
  const start = performance.now();
  const result = next(action);
  const duration = performance.now() - start;
  
  if (duration > 16) { // Longer than one frame
    console.warn(`Slow action: ${action.type} took ${duration.toFixed(2)}ms`);
    
    // Send to analytics
    if (window.gtag) {
      window.gtag('event', 'slow_action', {
        action_type: action.type,
        duration: Math.round(duration)
      });
    }
  }
  
  return result;
};
```

## Common Mistakes

### 1. Leaving DevTools Enabled in Production

```typescript
// ❌ Wrong - DevTools exposed in production
const store = createStore(
  rootReducer,
  window.__REDUX_DEVTOOLS_EXTENSION__?.()
);

// ✅ Correct - Only enable in development
const store = createStore(
  rootReducer,
  process.env.NODE_ENV === 'development' && window.__REDUX_DEVTOOLS_EXTENSION__?.()
);
```

### 2. Not Sanitizing Sensitive Data

```typescript
// ❌ Wrong - passwords visible in DevTools
dispatch({ type: 'LOGIN', payload: { password: 'secret123' } });

// ✅ Correct - sanitize sensitive data
devTools: {
  actionSanitizer: (action) => {
    if (action.type === 'LOGIN') {
      return { ...action, payload: { ...action.payload, password: '***' } };
    }
    return action;
  }
}
```

### 3. Storing Too Much History

```typescript
// ❌ Wrong - unlimited history causes memory issues
devTools: { maxAge: Infinity }

// ✅ Correct - reasonable limit
devTools: { maxAge: 50 }
```

### 4. Not Using Action Names

```typescript
// ❌ Wrong - unclear what action does
set((state) => ({ count: state.count + 1 }));

// ✅ Correct - descriptive action name
set(
  (state) => ({ count: state.count + 1 }),
  false,
  'counter/increment'
);
```

### 5. Ignoring DevTools Performance

```typescript
// ❌ Wrong - DevTools slowing down app
// (too many actions, large state objects)

// ✅ Correct - batch actions, use action filtering
dispatch(batchActions([
  action1(),
  action2(),
  action3()
]));
```

## Best Practices

### 1. Use Descriptive Action Names

```typescript
// Good - clear, hierarchical naming
'user/login/pending'
'user/login/fulfilled'
'user/login/rejected'
'todos/add'
'todos/update/status'
```

### 2. Implement State Sanitization

```typescript
devTools: {
  stateSanitizer: (state) => {
    if (state.auth?.token) {
      return {
        ...state,
        auth: { ...state.auth, token: '***HIDDEN***' }
      };
    }
    return state;
  }
}
```

### 3. Use Traces for Debugging

```typescript
devTools: {
  trace: true,
  traceLimit: 25
}
```

### 4. Export/Import State for Bug Reports

```typescript
// Users can export their state when reporting bugs
function ExportButton() {
  return (
    <button onClick={() => {
      const state = store.getState();
      const json = JSON.stringify(state, null, 2);
      downloadFile('state-export.json', json);
    }}>
      Export State for Bug Report
    </button>
  );
}
```

## Interview Questions

### Q1: How does time-travel debugging work?
**Answer**: DevTools store a history of state snapshots and actions. When you "jump" to a previous state, the tool replays all actions up to that point, reconstructing the state. This enables debugging by examining state at specific moments without re-running the app. It requires pure reducers and deterministic state updates.

### Q2: Why sanitize state/actions in DevTools?
**Answer**: DevTools can expose sensitive data (passwords, tokens, personal info) visible in browser extensions. Sanitization redacts this data in DevTools while keeping actual app functionality intact. This is critical for security, especially if users share DevTools logs for debugging.

### Q3: What's the performance impact of DevTools?
**Answer**: DevTools serialize state and actions on every change, adding overhead. In development this is acceptable, but in production it can cause memory leaks and slowdowns. Always disable or limit DevTools in production. Use maxAge limits and avoid deeply nested state for better performance.

### Q4: How do DevTools help with production debugging?
**Answer**: Integrate DevTools data with error tracking (Sentry, LogRocket). When errors occur, attach recent actions and state snapshots to error reports. This provides context for reproducing bugs. Implement export functionality so users can share state when reporting issues.

### Q5: What are action replay systems used for?
**Answer**: Action replay records user interactions as a sequence of actions with timing. Replaying them reproduces exact user flows, useful for bug reproduction, end-to-end testing, and creating test fixtures. It requires deterministic actions and can automate complex user scenarios.

## Key Takeaways

1. DevTools are essential for debugging complex state management
2. Always disable or limit DevTools in production
3. Sanitize sensitive data in action/state sanitizers
4. Use descriptive action names for clarity
5. Time-travel debugging requires pure, deterministic updates
6. Limit history size (maxAge) to prevent memory issues
7. Enable tracing to see where actions originate
8. Export/import state for bug reproduction
9. Integrate with error tracking for production debugging
10. Use profiling to identify performance bottlenecks

## Resources

- [Redux DevTools Extension](https://github.com/reduxjs/redux-devtools)
- [Redux DevTools Documentation](https://redux.js.org/usage/configuring-your-store#integrating-the-devtools-extension)
- [React DevTools](https://react.dev/learn/react-developer-tools)
- [Zustand DevTools](https://github.com/pmndrs/zustand#devtools)
- [NgRx DevTools](https://ngrx.io/guide/store-devtools)
- [MobX DevTools](https://github.com/mobxjs/mobx-devtools)
