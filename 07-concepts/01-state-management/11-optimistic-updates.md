# Optimistic Updates

## Overview

Optimistic updates are a UX pattern where the UI immediately reflects the expected result of an action before receiving server confirmation. Instead of showing loading states and waiting for network responses, the interface updates instantly, providing a snappy, responsive feel. If the server request fails, the UI rolls back to its previous state and shows an error. This technique is essential for building apps that feel fast and responsive, especially on slow networks.

## Core Concepts

### The Optimistic Update Flow

```
┌──────────────────────────────────────────────────────────┐
│           Optimistic Update Lifecycle                     │
└──────────────────────────────────────────────────────────┘

User Action
    │
    ▼
┌─────────────────┐
│  Store Current  │  ← Snapshot state for rollback
│      State      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Apply Update   │  ← Update UI immediately
│   Optimistically│
└────────┬────────┘
         │
         ├───────┐
         │       │
         ▼       ▼
    ┌────────┐ ┌────────┐
    │Success │ │ Failure│
    │        │ │        │
    │Confirm │ │Rollback│
    │Update  │ │to Saved│
    └────────┘ └────────┘
                    │
                    ▼
              ┌──────────┐
              │Show Error│
              └──────────┘
```

## Basic Optimistic Update Pattern

### React with useState

```typescript
import React, { useState } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [error, setError] = useState<string | null>(null);

  const toggleTodo = async (id: string) => {
    // Store original state for rollback
    const originalTodos = [...todos];
    
    // Optimistic update - apply immediately
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
    setError(null);

    try {
      const todo = todos.find(t => t.id === id);
      const response = await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed: !todo?.completed })
      });

      if (!response.ok) throw new Error('Update failed');

      // Optionally update with server response
      const updatedTodo = await response.json();
      setTodos(todos.map(todo =>
        todo.id === id ? updatedTodo : todo
      ));
    } catch (err) {
      // Rollback on error
      setTodos(originalTodos);
      setError('Failed to update todo. Please try again.');
    }
  };

  const deleteTodo = async (id: string) => {
    const originalTodos = [...todos];
    
    // Remove immediately
    setTodos(todos.filter(todo => todo.id !== id));
    setError(null);

    try {
      const response = await fetch(`/api/todos/${id}`, {
        method: 'DELETE'
      });

      if (!response.ok) throw new Error('Delete failed');
    } catch (err) {
      // Restore deleted item
      setTodos(originalTodos);
      setError('Failed to delete todo. Please try again.');
    }
  };

  const addTodo = async (text: string) => {
    // Create temporary ID
    const tempId = `temp-${Date.now()}`;
    const newTodo: Todo = {
      id: tempId,
      text,
      completed: false
    };

    // Add optimistically
    setTodos([...todos, newTodo]);
    setError(null);

    try {
      const response = await fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text, completed: false })
      });

      if (!response.ok) throw new Error('Create failed');

      // Replace temp ID with real ID from server
      const createdTodo = await response.json();
      setTodos(todos.map(todo =>
        todo.id === tempId ? createdTodo : todo
      ));
    } catch (err) {
      // Remove optimistic item
      setTodos(todos.filter(todo => todo.id !== tempId));
      setError('Failed to create todo. Please try again.');
    }
  };

  return (
    <div>
      {error && (
        <div style={{ background: 'red', color: 'white', padding: 10 }}>
          {error}
        </div>
      )}
      
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ 
              textDecoration: todo.completed ? 'line-through' : 'none',
              opacity: todo.id.startsWith('temp-') ? 0.5 : 1
            }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### React with useReducer for Complex State

```typescript
import React, { useReducer } from 'react';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  version: number;
}

interface State {
  todos: Todo[];
  pendingActions: Map<string, () => void>; // Rollback functions
  errors: Map<string, string>;
}

type Action =
  | { type: 'OPTIMISTIC_TOGGLE'; id: string; rollback: () => void }
  | { type: 'CONFIRM_TOGGLE'; id: string }
  | { type: 'ROLLBACK_TOGGLE'; id: string; error: string }
  | { type: 'OPTIMISTIC_DELETE'; id: string; rollback: () => void }
  | { type: 'CONFIRM_DELETE'; id: string }
  | { type: 'ROLLBACK_DELETE'; id: string; error: string }
  | { type: 'CLEAR_ERROR'; id: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'OPTIMISTIC_TOGGLE':
      const newPendingActions = new Map(state.pendingActions);
      newPendingActions.set(action.id, action.rollback);
      
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.id
            ? { ...todo, completed: !todo.completed }
            : todo
        ),
        pendingActions: newPendingActions
      };

    case 'CONFIRM_TOGGLE':
      const confirmedPending = new Map(state.pendingActions);
      confirmedPending.delete(action.id);
      
      return {
        ...state,
        pendingActions: confirmedPending
      };

    case 'ROLLBACK_TOGGLE':
      const rollback = state.pendingActions.get(action.id);
      if (rollback) rollback();
      
      const updatedPending = new Map(state.pendingActions);
      updatedPending.delete(action.id);
      
      const newErrors = new Map(state.errors);
      newErrors.set(action.id, action.error);
      
      return {
        ...state,
        pendingActions: updatedPending,
        errors: newErrors
      };

    case 'OPTIMISTIC_DELETE':
      const deletePending = new Map(state.pendingActions);
      deletePending.set(action.id, action.rollback);
      
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.id),
        pendingActions: deletePending
      };

    case 'CONFIRM_DELETE':
      const deleteConfirmed = new Map(state.pendingActions);
      deleteConfirmed.delete(action.id);
      
      return {
        ...state,
        pendingActions: deleteConfirmed
      };

    case 'ROLLBACK_DELETE':
      const deleteRollback = state.pendingActions.get(action.id);
      if (deleteRollback) deleteRollback();
      
      const deletePendingUpdated = new Map(state.pendingActions);
      deletePendingUpdated.delete(action.id);
      
      const deleteErrors = new Map(state.errors);
      deleteErrors.set(action.id, action.error);
      
      return {
        ...state,
        pendingActions: deletePendingUpdated,
        errors: deleteErrors
      };

    case 'CLEAR_ERROR':
      const clearedErrors = new Map(state.errors);
      clearedErrors.delete(action.id);
      
      return {
        ...state,
        errors: clearedErrors
      };

    default:
      return state;
  }
}

function OptimisticTodoList() {
  const [state, dispatch] = useReducer(reducer, {
    todos: [],
    pendingActions: new Map(),
    errors: new Map()
  });

  const toggleTodo = async (id: string) => {
    const originalTodos = [...state.todos];
    
    dispatch({
      type: 'OPTIMISTIC_TOGGLE',
      id,
      rollback: () => {
        // Restore original state
        state.todos = originalTodos;
      }
    });

    try {
      const todo = state.todos.find(t => t.id === id);
      const response = await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ 
          completed: !todo?.completed,
          version: todo?.version
        })
      });

      if (!response.ok) throw new Error('Update failed');

      dispatch({ type: 'CONFIRM_TOGGLE', id });
    } catch (err) {
      dispatch({
        type: 'ROLLBACK_TOGGLE',
        id,
        error: err.message
      });
    }
  };

  return (
    <div>
      {Array.from(state.errors.entries()).map(([id, error]) => (
        <div key={id} className="error">
          {error}
          <button onClick={() => dispatch({ type: 'CLEAR_ERROR', id })}>
            Dismiss
          </button>
        </div>
      ))}
      
      <ul>
        {state.todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
              disabled={state.pendingActions.has(todo.id)}
            />
            <span>{todo.text}</span>
            {state.pendingActions.has(todo.id) && (
              <span className="pending">Syncing...</span>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## React Query Optimistic Updates

### Mutation with Optimistic Update

```typescript
import { useMutation, useQueryClient, useQuery } from '@tanstack/react-query';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoListWithReactQuery() {
  const queryClient = useQueryClient();

  // Fetch todos
  const { data: todos = [] } = useQuery<Todo[]>({
    queryKey: ['todos'],
    queryFn: async () => {
      const response = await fetch('/api/todos');
      return response.json();
    }
  });

  // Toggle todo mutation
  const toggleMutation = useMutation({
    mutationFn: async ({ id, completed }: { id: string; completed: boolean }) => {
      const response = await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ completed })
      });
      
      if (!response.ok) throw new Error('Update failed');
      return response.json();
    },
    
    // Optimistic update
    onMutate: async ({ id, completed }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot previous value
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      // Optimistically update
      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old?.map(todo =>
          todo.id === id ? { ...todo, completed } : todo
        )
      );

      // Return context with snapshot
      return { previousTodos };
    },
    
    // Rollback on error
    onError: (err, variables, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos);
      }
    },
    
    // Always refetch after error or success
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    }
  });

  // Delete mutation
  const deleteMutation = useMutation({
    mutationFn: async (id: string) => {
      const response = await fetch(`/api/todos/${id}`, {
        method: 'DELETE'
      });
      
      if (!response.ok) throw new Error('Delete failed');
    },
    
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old?.filter(todo => todo.id !== id)
      );

      return { previousTodos };
    },
    
    onError: (err, id, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos);
      }
    },
    
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    }
  });

  // Add mutation with temp ID
  const addMutation = useMutation({
    mutationFn: async (text: string) => {
      const response = await fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text, completed: false })
      });
      
      if (!response.ok) throw new Error('Create failed');
      return response.json();
    },
    
    onMutate: async (text) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      const tempTodo: Todo = {
        id: `temp-${Date.now()}`,
        text,
        completed: false
      };

      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old ? [...old, tempTodo] : [tempTodo]
      );

      return { previousTodos, tempId: tempTodo.id };
    },
    
    // Replace temp ID with real ID on success
    onSuccess: (newTodo, text, context) => {
      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old?.map(todo =>
          todo.id === context?.tempId ? newTodo : todo
        )
      );
    },
    
    onError: (err, text, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos);
      }
    }
  });

  return (
    <div>
      {toggleMutation.isError && (
        <div className="error">
          Error: {toggleMutation.error.message}
        </div>
      )}
      
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() =>
                toggleMutation.mutate({
                  id: todo.id,
                  completed: !todo.completed
                })
              }
            />
            <span style={{
              opacity: todo.id.startsWith('temp-') ? 0.5 : 1
            }}>
              {todo.text}
            </span>
            <button
              onClick={() => deleteMutation.mutate(todo.id)}
              disabled={deleteMutation.isPending}
            >
              Delete
            </button>
          </li>
        ))}
      </ul>
      
      <button onClick={() => addMutation.mutate('New Todo')}>
        Add Todo
      </button>
    </div>
  );
}
```

## Angular Optimistic Updates

### With RxJS

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, throwError } from 'rxjs';
import { catchError, tap, finalize } from 'rxjs/operators';
import { HttpClient } from '@angular/common/http';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

@Injectable({ providedIn: 'root' })
export class OptimisticTodoService {
  private todosSubject = new BehaviorSubject<Todo[]>([]);
  todos$ = this.todosSubject.asObservable();

  private errorSubject = new BehaviorSubject<string | null>(null);
  error$ = this.errorSubject.asObservable();

  constructor(private http: HttpClient) {}

  loadTodos() {
    this.http.get<Todo[]>('/api/todos').subscribe(todos => {
      this.todosSubject.next(todos);
    });
  }

  toggleTodo(id: string) {
    // Store original state
    const originalTodos = this.todosSubject.value;
    const todo = originalTodos.find(t => t.id === id);
    
    if (!todo) return;

    // Optimistic update
    const optimisticTodos = originalTodos.map(t =>
      t.id === id ? { ...t, completed: !t.completed } : t
    );
    this.todosSubject.next(optimisticTodos);
    this.errorSubject.next(null);

    // Send to server
    this.http
      .patch<Todo>(`/api/todos/${id}`, { completed: !todo.completed })
      .pipe(
        tap(updatedTodo => {
          // Update with server response
          const todos = this.todosSubject.value.map(t =>
            t.id === id ? updatedTodo : t
          );
          this.todosSubject.next(todos);
        }),
        catchError(error => {
          // Rollback on error
          this.todosSubject.next(originalTodos);
          this.errorSubject.next('Failed to update todo');
          return throwError(() => error);
        })
      )
      .subscribe();
  }

  deleteTodo(id: string) {
    const originalTodos = this.todosSubject.value;

    // Optimistic deletion
    this.todosSubject.next(originalTodos.filter(t => t.id !== id));
    this.errorSubject.next(null);

    this.http
      .delete(`/api/todos/${id}`)
      .pipe(
        catchError(error => {
          // Rollback
          this.todosSubject.next(originalTodos);
          this.errorSubject.next('Failed to delete todo');
          return throwError(() => error);
        })
      )
      .subscribe();
  }

  addTodo(text: string) {
    const originalTodos = this.todosSubject.value;
    const tempId = `temp-${Date.now()}`;
    const tempTodo: Todo = { id: tempId, text, completed: false };

    // Add optimistically
    this.todosSubject.next([...originalTodos, tempTodo]);
    this.errorSubject.next(null);

    this.http
      .post<Todo>('/api/todos', { text, completed: false })
      .pipe(
        tap(createdTodo => {
          // Replace temp with real
          const todos = this.todosSubject.value.map(t =>
            t.id === tempId ? createdTodo : t
          );
          this.todosSubject.next(todos);
        }),
        catchError(error => {
          // Remove temp todo
          this.todosSubject.next(
            this.todosSubject.value.filter(t => t.id !== tempId)
          );
          this.errorSubject.next('Failed to create todo');
          return throwError(() => error);
        })
      )
      .subscribe();
  }
}

// Component
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-todo-list',
  template: `
    <div>
      <div *ngIf="error$ | async as error" class="error">
        {{ error }}
      </div>
      
      <ul>
        <li *ngFor="let todo of todos$ | async">
          <input
            type="checkbox"
            [checked]="todo.completed"
            (change)="todoService.toggleTodo(todo.id)"
          />
          <span [class.temp]="todo.id.startsWith('temp-')">
            {{ todo.text }}
          </span>
          <button (click)="todoService.deleteTodo(todo.id)">
            Delete
          </button>
        </li>
      </ul>
      
      <button (click)="todoService.addTodo('New Todo')">
        Add Todo
      </button>
    </div>
  `,
  styles: [`
    .temp { opacity: 0.5; }
    .error { background: red; color: white; padding: 10px; }
  `]
})
export class TodoListComponent implements OnInit {
  todos$ = this.todoService.todos$;
  error$ = this.todoService.error$;

  constructor(public todoService: OptimisticTodoService) {}

  ngOnInit() {
    this.todoService.loadTodos();
  }
}
```

## Conflict Resolution

### Version-Based Conflict Detection

```typescript
interface VersionedTodo {
  id: string;
  text: string;
  completed: boolean;
  version: number;
}

async function updateWithVersionCheck(
  id: string,
  updates: Partial<VersionedTodo>,
  currentVersion: number
) {
  const response = await fetch(`/api/todos/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      ...updates,
      version: currentVersion
    })
  });

  if (response.status === 409) {
    // Conflict - server has newer version
    const serverData = await response.json();
    
    // Show conflict resolution UI
    return {
      conflict: true,
      serverVersion: serverData,
      clientVersion: updates
    };
  }

  if (!response.ok) throw new Error('Update failed');
  return { conflict: false, data: await response.json() };
}

function TodoWithConflictResolution() {
  const [showConflict, setShowConflict] = useState(false);
  const [conflictData, setConflictData] = useState<any>(null);

  const handleUpdate = async (id: string, updates: any, version: number) => {
    const result = await updateWithVersionCheck(id, updates, version);
    
    if (result.conflict) {
      setConflictData(result);
      setShowConflict(true);
    } else {
      // Success - update local state
      updateLocalTodo(result.data);
    }
  };

  const resolveConflict = (useServer: boolean) => {
    if (useServer) {
      updateLocalTodo(conflictData.serverVersion);
    } else {
      // Force update with client version
      forceUpdate(conflictData.clientVersion);
    }
    setShowConflict(false);
  };

  if (showConflict) {
    return (
      <div className="conflict-modal">
        <h3>Conflict Detected</h3>
        <div>
          <h4>Your Changes:</h4>
          <pre>{JSON.stringify(conflictData.clientVersion, null, 2)}</pre>
        </div>
        <div>
          <h4>Server Version:</h4>
          <pre>{JSON.stringify(conflictData.serverVersion, null, 2)}</pre>
        </div>
        <button onClick={() => resolveConflict(true)}>
          Use Server Version
        </button>
        <button onClick={() => resolveConflict(false)}>
          Use My Changes
        </button>
      </div>
    );
  }

  // Normal UI...
}
```

### Last-Write-Wins with Timestamps

```typescript
interface TimestampedTodo {
  id: string;
  text: string;
  completed: boolean;
  updatedAt: number;
}

async function updateWithTimestamp(
  id: string,
  updates: Partial<TimestampedTodo>,
  localTimestamp: number
) {
  const now = Date.now();
  
  const response = await fetch(`/api/todos/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      ...updates,
      updatedAt: now,
      clientTimestamp: localTimestamp
    })
  });

  if (!response.ok) throw new Error('Update failed');
  
  const serverData = await response.json();
  
  // Server may reject if its timestamp is newer
  if (serverData.updatedAt > now) {
    return {
      conflict: true,
      serverData
    };
  }

  return { conflict: false, data: serverData };
}
```

## Advanced Patterns

### Queue-Based Optimistic Updates

```typescript
interface OptimisticAction {
  id: string;
  type: 'add' | 'update' | 'delete';
  data: any;
  rollback: () => void;
  timestamp: number;
}

class OptimisticQueue {
  private queue: OptimisticAction[] = [];
  private processing = false;

  add(action: OptimisticAction) {
    this.queue.push(action);
    this.process();
  }

  private async process() {
    if (this.processing || this.queue.length === 0) return;

    this.processing = true;
    const action = this.queue[0];

    try {
      await this.executeAction(action);
      this.queue.shift(); // Remove on success
    } catch (error) {
      action.rollback();
      this.queue.shift();
      console.error('Action failed:', error);
    } finally {
      this.processing = false;
      if (this.queue.length > 0) {
        this.process(); // Process next
      }
    }
  }

  private async executeAction(action: OptimisticAction) {
    switch (action.type) {
      case 'add':
        return fetch('/api/todos', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(action.data)
        });
      case 'update':
        return fetch(`/api/todos/${action.data.id}`, {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(action.data)
        });
      case 'delete':
        return fetch(`/api/todos/${action.data.id}`, {
          method: 'DELETE'
        });
    }
  }

  getQueueLength() {
    return this.queue.length;
  }
}

const queue = new OptimisticQueue();

function TodoWithQueue() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [queueLength, setQueueLength] = useState(0);

  const toggleTodo = (id: string) => {
    const original = [...todos];
    
    setTodos(todos.map(t =>
      t.id === id ? { ...t, completed: !t.completed } : t
    ));

    queue.add({
      id: `toggle-${id}`,
      type: 'update',
      data: { id, completed: !todos.find(t => t.id === id)?.completed },
      rollback: () => setTodos(original),
      timestamp: Date.now()
    });

    setQueueLength(queue.getQueueLength());
  };

  return (
    <div>
      {queueLength > 0 && (
        <div className="sync-indicator">
          Syncing {queueLength} change(s)...
        </div>
      )}
      {/* Todo list UI */}
    </div>
  );
}
```

### Retry Logic with Exponential Backoff

```typescript
async function updateWithRetry(
  fn: () => Promise<any>,
  maxRetries = 3,
  baseDelay = 1000
) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }
      
      const delay = baseDelay * Math.pow(2, attempt);
      console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

const toggleMutationWithRetry = useMutation({
  mutationFn: async (variables) => {
    return updateWithRetry(() =>
      fetch(`/api/todos/${variables.id}`, {
        method: 'PATCH',
        body: JSON.stringify(variables)
      }).then(r => {
        if (!r.ok) throw new Error('Update failed');
        return r.json();
      })
    );
  },
  // ... optimistic update config
});
```

## Common Mistakes

### 1. Not Storing Original State

```typescript
// ❌ Wrong - can't rollback
const toggleTodo = async (id: string) => {
  setTodos(todos.map(t =>
    t.id === id ? { ...t, completed: !t.completed } : t
  ));
  
  await api.update(id); // What if this fails?
};

// ✅ Correct - store for rollback
const toggleTodo = async (id: string) => {
  const original = [...todos];
  setTodos(todos.map(t =>
    t.id === id ? { ...t, completed: !t.completed } : t
  ));
  
  try {
    await api.update(id);
  } catch {
    setTodos(original);
  }
};
```

### 2. Not Canceling Inflight Requests

```typescript
// ❌ Wrong - race conditions
const search = async (query: string) => {
  setResults(mockResults); // Optimistic
  const data = await api.search(query);
  setResults(data); // Might be stale!
};

// ✅ Correct - cancel previous
const search = async (query: string) => {
  if (abortController) abortController.abort();
  const controller = new AbortController();
  setAbortController(controller);
  
  setResults(mockResults);
  try {
    const data = await api.search(query, { signal: controller.signal });
    setResults(data);
  } catch (err) {
    if (err.name !== 'AbortError') {
      // Handle error
    }
  }
};
```

### 3. Not Handling Network Errors

```typescript
// ❌ Wrong - no error handling
const addTodo = (text: string) => {
  setTodos([...todos, newTodo]);
  api.create(newTodo); // Fire and forget!
};

// ✅ Correct - handle errors
const addTodo = async (text: string) => {
  const original = [...todos];
  setTodos([...todos, newTodo]);
  
  try {
    await api.create(newTodo);
  } catch (error) {
    setTodos(original);
    setError('Failed to add todo');
  }
};
```

### 4. Using Optimistic Updates for Critical Actions

```typescript
// ❌ Wrong - don't be optimistic with payments!
const processPayment = async () => {
  setPaymentStatus('success'); // Optimistic - BAD!
  await api.pay();
};

// ✅ Correct - wait for confirmation
const processPayment = async () => {
  setPaymentStatus('processing');
  try {
    await api.pay();
    setPaymentStatus('success');
  } catch {
    setPaymentStatus('failed');
  }
};
```

### 5. Not Visual Indicating Pending State

```typescript
// ❌ Wrong - no indication it's pending
<li>{todo.text}</li>

// ✅ Correct - show pending state
<li style={{ opacity: isPending ? 0.5 : 1 }}>
  {todo.text}
  {isPending && <span>Syncing...</span>}
</li>
```

## Best Practices

### 1. Use for Non-Critical Operations

Optimistic updates work best for actions where:
- Failure is rare
- Rollback is acceptable
- User expects instant feedback

Avoid for:
- Financial transactions
- Account deletions
- Data exports
- Critical security operations

### 2. Provide Clear Feedback

```typescript
// Show different states clearly
{isPending && <span className="syncing">Syncing...</span>}
{isError && <span className="error">Failed - retrying...</span>}
{isSynced && <span className="success">✓</span>}
```

### 3. Use Temporary IDs

```typescript
// Generate temp IDs for new items
const tempId = `temp-${Date.now()}-${Math.random()}`;
```

### 4. Implement Proper Error Recovery

```typescript
// Allow manual retry
{error && (
  <button onClick={retry}>
    Retry
  </button>
)}
```

## When to Use Optimistic Updates

### Use When:
- Building responsive, snappy UIs
- Network latency is high
- Actions are frequent (likes, toggles)
- Failure rate is low
- Rollback is acceptable

### Avoid When:
- Actions are irreversible
- Data consistency is critical
- Financial transactions
- Legal/compliance requirements
- Users need explicit confirmation

## Interview Questions

### Q1: What are optimistic updates?
**Answer**: Optimistic updates immediately reflect expected results in the UI before server confirmation. The interface updates instantly, then either confirms the change or rolls back on error. This provides responsive UX, especially on slow networks, at the cost of occasional rollbacks.

### Q2: How do you handle rollbacks?
**Answer**: Store the original state before applying the optimistic update. If the server request fails, restore the saved state and show an error message. In React Query, this is done via the onMutate (save), onError (restore), and onSettled (cleanup) callbacks.

### Q3: What are the risks of optimistic updates?
**Answer**: Users see changes that might not persist. Multiple concurrent updates can conflict. Race conditions can occur with network timing. Users might act on incorrect state. Financial or critical operations should never use optimistic updates without explicit confirmation.

### Q4: How do you handle conflicts?
**Answer**: Use version numbers or timestamps to detect conflicts. When detected, present resolution UI showing both versions. Implement strategies like last-write-wins, merge, or manual resolution. Cancel inflight requests to prevent race conditions.

### Q5: When should you avoid optimistic updates?
**Answer**: Avoid for critical operations (payments, deletions), when rollback confuses users, for actions requiring validation, when failure rates are high, or when consistency is paramount. Use traditional loading states when users need explicit confirmation.

## Key Takeaways

1. Optimistic updates provide instant feedback before server confirmation
2. Always store original state for rollback capability
3. Handle errors gracefully with clear rollback and retry
4. Use temporary IDs for newly created items
5. Cancel inflight requests to prevent race conditions
6. Indicate pending state visually (opacity, spinners)
7. Avoid optimistic updates for critical operations
8. React Query provides excellent built-in optimistic update support
9. Queue operations for better reliability
10. Implement conflict resolution for concurrent updates

## Resources

- [React Query Optimistic Updates](https://tanstack.com/query/latest/docs/react/guides/optimistic-updates)
- [Optimistic UI Patterns](https://www.apollographql.com/docs/react/performance/optimistic-ui/)
- [Conflict-Free Replicated Data Types (CRDTs)](https://crdt.tech/)
- [Offline First Design](https://offlinefirst.org/)
- [Network Resilience Patterns](https://www.patterns.dev/posts/optimistic-ui-pattern/)
