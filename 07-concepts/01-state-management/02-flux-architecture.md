# Flux Architecture

## The Idea

**In plain English:** Flux is a set of rules for how data moves through an app — it forces all changes to travel in one direction, like a one-way street, so you always know exactly where something changed and why. A "state" is just the app's memory of what's happening right now (like which items are in a list).

**Real-world analogy:** Think of a restaurant where customers order food. A customer (the View) tells the waiter (the Action) what they want. The waiter gives the order ticket to the kitchen manager (the Dispatcher), who posts it on the board for all cooks (the Stores) to see. Each cook handles their part and updates the food tray (the state), which the waiter then delivers back to the customer.

- The customer = the View (what the user sees and interacts with)
- The order ticket = the Action (a description of what the user wants to happen)
- The kitchen manager = the Dispatcher (the central hub that broadcasts the order to everyone)
- The cook = the Store (holds the data and knows how to update it based on the order)

---

## Overview

Flux is an application architecture pattern developed by Facebook for building scalable client-side web applications. It complements React's composable view components by utilizing a unidirectional data flow, making the application state more predictable and easier to debug. While Redux has largely superseded Flux in popularity, understanding Flux is essential as it forms the foundation for modern state management patterns including Redux, Zustand, and other state management libraries.

## The Problem Flux Solves

Before Flux, many applications used the Model-View-Controller (MVC) pattern, which can become complex as applications grow:

```
Traditional MVC - Bidirectional Data Flow:

┌──────┐ ←──→ ┌──────┐
│Model │ ←──→ │View  │
└───┬──┘ ←──→ └───┬──┘
    │  ←──→      │
    ↓  ←──→      ↓
┌────────────────┐
│  Controller    │
└────────────────┘

Problems:
- Bidirectional data flow creates cascading updates
- Hard to predict how changes propagate
- Difficult to debug complex interactions
- Models and views are tightly coupled
- Adding features increases complexity exponentially
```

Facebook encountered this in their chat system - unread message count wouldn't sync properly because of the complex web of dependencies between models and views.

## Flux Architecture: Unidirectional Data Flow

Flux enforces a unidirectional data flow, making state mutations predictable and traceable:

```
Flux Unidirectional Data Flow:

┌────────┐      ┌────────────┐      ┌───────┐      ┌──────┐
│ Action │ ───→ │ Dispatcher │ ───→ │ Store │ ───→ │ View │
└────────┘      └────────────┘      └───────┘      └───┬──┘
    ↑                                                   │
    └───────────────────────────────────────────────────┘
                    User interactions

Flow:
1. View dispatches Action
2. Dispatcher sends Action to ALL Stores
3. Stores update state based on Action
4. Stores emit change event
5. Views re-render with new state
```

## Core Concepts

### 1. Actions

Actions are simple objects that describe events in the application. They have a `type` (what happened) and optional `payload` (data).

```typescript
// Action types - constants to prevent typos
const ActionTypes = {
  ADD_TODO: 'ADD_TODO',
  TOGGLE_TODO: 'TOGGLE_TODO',
  DELETE_TODO: 'DELETE_TODO',
  LOAD_TODOS: 'LOAD_TODOS',
  LOAD_TODOS_SUCCESS: 'LOAD_TODOS_SUCCESS',
  LOAD_TODOS_ERROR: 'LOAD_TODOS_ERROR'
} as const;

// Action interfaces
interface AddTodoAction {
  type: typeof ActionTypes.ADD_TODO;
  payload: {
    text: string;
  };
}

interface ToggleTodoAction {
  type: typeof ActionTypes.TOGGLE_TODO;
  payload: {
    id: string;
  };
}

interface DeleteTodoAction {
  type: typeof ActionTypes.DELETE_TODO;
  payload: {
    id: string;
  };
}

interface LoadTodosSuccessAction {
  type: typeof ActionTypes.LOAD_TODOS_SUCCESS;
  payload: {
    todos: Todo[];
  };
}

// Union type for all actions
type TodoAction = 
  | AddTodoAction 
  | ToggleTodoAction 
  | DeleteTodoAction
  | LoadTodosSuccessAction;
```

### 2. Action Creators

Action creators are functions that create and return action objects. They encapsulate action creation logic.

```typescript
// Simple action creators
const TodoActions = {
  addTodo(text: string): AddTodoAction {
    return {
      type: ActionTypes.ADD_TODO,
      payload: { text }
    };
  },
  
  toggleTodo(id: string): ToggleTodoAction {
    return {
      type: ActionTypes.TOGGLE_TODO,
      payload: { id }
    };
  },
  
  deleteTodo(id: string): DeleteTodoAction {
    return {
      type: ActionTypes.DELETE_TODO,
      payload: { id }
    };
  },
  
  loadTodosSuccess(todos: Todo[]): LoadTodosSuccessAction {
    return {
      type: ActionTypes.LOAD_TODOS_SUCCESS,
      payload: { todos }
    };
  }
};

// Async action creator (with dispatcher)
const loadTodos = () => {
  // Dispatch loading action
  AppDispatcher.dispatch({
    type: ActionTypes.LOAD_TODOS
  });
  
  // Fetch data
  fetch('/api/todos')
    .then(res => res.json())
    .then(todos => {
      AppDispatcher.dispatch(TodoActions.loadTodosSuccess(todos));
    })
    .catch(error => {
      AppDispatcher.dispatch({
        type: ActionTypes.LOAD_TODOS_ERROR,
        payload: { error }
      });
    });
};
```

### 3. Dispatcher

The dispatcher is the central hub managing all data flow in a Flux application. It broadcasts actions to all registered stores.

```typescript
// Simple Flux Dispatcher implementation
type Callback<T> = (action: T) => void;

class Dispatcher<T> {
  private callbacks: Map<string, Callback<T>> = new Map();
  private isPending: Map<string, boolean> = new Map();
  private isHandled: Map<string, boolean> = new Map();
  private pendingPayload: T | null = null;
  
  /**
   * Registers a callback to be invoked with every dispatched action
   */
  register(callback: Callback<T>): string {
    const id = `ID_${Date.now()}_${Math.random()}`;
    this.callbacks.set(id, callback);
    return id;
  }
  
  /**
   * Removes a callback based on its token
   */
  unregister(id: string): void {
    if (!this.callbacks.has(id)) {
      throw new Error(`Dispatcher.unregister: ${id} does not map to a registered callback.`);
    }
    this.callbacks.delete(id);
  }
  
  /**
   * Dispatches an action to all registered callbacks
   */
  dispatch(action: T): void {
    if (this.isDispatching()) {
      throw new Error('Cannot dispatch in the middle of a dispatch.');
    }
    
    this.startDispatching(action);
    
    try {
      for (const [id, callback] of this.callbacks) {
        if (this.isPending.get(id)) {
          continue;
        }
        this.invokeCallback(id);
      }
    } finally {
      this.stopDispatching();
    }
  }
  
  /**
   * Checks if the dispatcher is currently dispatching
   */
  isDispatching(): boolean {
    return this.pendingPayload !== null;
  }
  
  /**
   * Allows a store to wait for other stores to update first
   */
  waitFor(ids: string[]): void {
    if (!this.isDispatching()) {
      throw new Error('Dispatcher.waitFor: Must be invoked while dispatching.');
    }
    
    for (const id of ids) {
      if (this.isPending.get(id)) {
        if (!this.isHandled.get(id)) {
          throw new Error(`Dispatcher.waitFor: circular dependency detected while waiting for ${id}.`);
        }
        continue;
      }
      
      if (!this.callbacks.has(id)) {
        throw new Error(`Dispatcher.waitFor: ${id} does not map to a registered callback.`);
      }
      
      this.invokeCallback(id);
    }
  }
  
  private invokeCallback(id: string): void {
    this.isPending.set(id, true);
    this.callbacks.get(id)!(this.pendingPayload!);
    this.isHandled.set(id, true);
  }
  
  private startDispatching(payload: T): void {
    for (const id of this.callbacks.keys()) {
      this.isPending.set(id, false);
      this.isHandled.set(id, false);
    }
    this.pendingPayload = payload;
  }
  
  private stopDispatching(): void {
    this.pendingPayload = null;
  }
}

// Create singleton dispatcher instance
const AppDispatcher = new Dispatcher<TodoAction>();
export default AppDispatcher;
```

### 4. Stores

Stores contain application state and business logic. They register with the dispatcher and respond to actions.

```typescript
import { EventEmitter } from 'events';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: number;
}

class TodoStore extends EventEmitter {
  private todos: Map<string, Todo> = new Map();
  private loading = false;
  private error: string | null = null;
  
  constructor() {
    super();
    
    // Register with dispatcher
    AppDispatcher.register((action: TodoAction) => {
      switch (action.type) {
        case ActionTypes.ADD_TODO:
          this.addTodo(action.payload.text);
          this.emitChange();
          break;
          
        case ActionTypes.TOGGLE_TODO:
          this.toggleTodo(action.payload.id);
          this.emitChange();
          break;
          
        case ActionTypes.DELETE_TODO:
          this.deleteTodo(action.payload.id);
          this.emitChange();
          break;
          
        case ActionTypes.LOAD_TODOS:
          this.loading = true;
          this.emitChange();
          break;
          
        case ActionTypes.LOAD_TODOS_SUCCESS:
          this.loading = false;
          this.setTodos(action.payload.todos);
          this.emitChange();
          break;
          
        case ActionTypes.LOAD_TODOS_ERROR:
          this.loading = false;
          this.error = action.payload.error.message;
          this.emitChange();
          break;
      }
    });
  }
  
  // Private methods that modify state
  private addTodo(text: string): void {
    const todo: Todo = {
      id: `todo_${Date.now()}`,
      text,
      completed: false,
      createdAt: Date.now()
    };
    this.todos.set(todo.id, todo);
  }
  
  private toggleTodo(id: string): void {
    const todo = this.todos.get(id);
    if (todo) {
      todo.completed = !todo.completed;
    }
  }
  
  private deleteTodo(id: string): void {
    this.todos.delete(id);
  }
  
  private setTodos(todos: Todo[]): void {
    this.todos.clear();
    todos.forEach(todo => this.todos.set(todo.id, todo));
  }
  
  // Public API - getters only (read-only)
  public getTodos(): Todo[] {
    return Array.from(this.todos.values())
      .sort((a, b) => b.createdAt - a.createdAt);
  }
  
  public getTodo(id: string): Todo | undefined {
    return this.todos.get(id);
  }
  
  public getCompletedTodos(): Todo[] {
    return this.getTodos().filter(todo => todo.completed);
  }
  
  public getActiveTodos(): Todo[] {
    return this.getTodos().filter(todo => !todo.completed);
  }
  
  public isLoading(): boolean {
    return this.loading;
  }
  
  public getError(): string | null {
    return this.error;
  }
  
  // Event emitter methods
  public emitChange(): void {
    this.emit('change');
  }
  
  public addChangeListener(callback: () => void): void {
    this.on('change', callback);
  }
  
  public removeChangeListener(callback: () => void): void {
    this.removeListener('change', callback);
  }
}

// Create singleton store instance
const todoStore = new TodoStore();
export default todoStore;
```

### 5. Views (React Components)

Views listen to stores and re-render when store data changes.

```typescript
import React, { useState, useEffect } from 'react';
import todoStore from './TodoStore';
import AppDispatcher from './Dispatcher';
import { TodoActions } from './TodoActions';

function TodoList() {
  const [todos, setTodos] = useState(todoStore.getTodos());
  const [loading, setLoading] = useState(todoStore.isLoading());
  const [newTodoText, setNewTodoText] = useState('');
  
  useEffect(() => {
    // Subscribe to store changes
    const handleChange = () => {
      setTodos(todoStore.getTodos());
      setLoading(todoStore.isLoading());
    };
    
    todoStore.addChangeListener(handleChange);
    
    // Cleanup subscription
    return () => {
      todoStore.removeChangeListener(handleChange);
    };
  }, []);
  
  const handleAddTodo = (e: React.FormEvent) => {
    e.preventDefault();
    if (newTodoText.trim()) {
      // Dispatch action via dispatcher
      AppDispatcher.dispatch(TodoActions.addTodo(newTodoText));
      setNewTodoText('');
    }
  };
  
  const handleToggle = (id: string) => {
    AppDispatcher.dispatch(TodoActions.toggleTodo(id));
  };
  
  const handleDelete = (id: string) => {
    AppDispatcher.dispatch(TodoActions.deleteTodo(id));
  };
  
  if (loading) {
    return <div>Loading todos...</div>;
  }
  
  return (
    <div>
      <form onSubmit={handleAddTodo}>
        <input
          type="text"
          value={newTodoText}
          onChange={e => setNewTodoText(e.target.value)}
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
              onChange={() => handleToggle(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => handleDelete(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
      
      <div>
        Total: {todos.length} | 
        Active: {todoStore.getActiveTodos().length} | 
        Completed: {todoStore.getCompletedTodos().length}
      </div>
    </div>
  );
}

export default TodoList;
```

## Complete Flux Data Flow

```
Complete Flux Flow Example - Adding a Todo:

1. User types "Buy milk" and clicks Add
   │
   ↓
2. View calls: AppDispatcher.dispatch(TodoActions.addTodo('Buy milk'))
   │
   ↓
3. Dispatcher receives action:
   {
     type: 'ADD_TODO',
     payload: { text: 'Buy milk' }
   }
   │
   ↓
4. Dispatcher broadcasts to ALL registered stores
   │
   ├──→ TodoStore receives action
   │    - Calls addTodo('Buy milk')
   │    - Updates internal todos Map
   │    - Emits 'change' event
   │
   └──→ Other stores receive action (ignore it)
   │
   ↓
5. TodoStore emits 'change' event
   │
   ↓
6. TodoList component's change listener triggers
   │
   ↓
7. Component calls todoStore.getTodos()
   │
   ↓
8. Component calls setTodos() with new data
   │
   ↓
9. React re-renders TodoList with updated todos
   │
   ↓
10. User sees "Buy milk" in the list
```

## Store Dependencies with waitFor

Sometimes stores depend on other stores. The `waitFor` method handles this:

```typescript
class UserStore extends EventEmitter {
  private users: Map<string, User> = new Map();
  private dispatchToken: string;
  
  constructor() {
    super();
    
    this.dispatchToken = AppDispatcher.register((action) => {
      switch (action.type) {
        case ActionTypes.LOAD_USERS_SUCCESS:
          this.setUsers(action.payload.users);
          this.emitChange();
          break;
      }
    });
  }
  
  getDispatchToken(): string {
    return this.dispatchToken;
  }
  
  getUser(id: string): User | undefined {
    return this.users.get(id);
  }
  
  private setUsers(users: User[]): void {
    this.users.clear();
    users.forEach(user => this.users.set(user.id, user));
  }
}

class TodoStoreWithDependency extends EventEmitter {
  private todos: Map<string, Todo> = new Map();
  
  constructor() {
    super();
    
    AppDispatcher.register((action) => {
      switch (action.type) {
        case ActionTypes.LOAD_TODOS_SUCCESS:
          // Wait for UserStore to finish processing first
          AppDispatcher.waitFor([userStore.getDispatchToken()]);
          
          // Now we can safely use UserStore data
          const todos = action.payload.todos.map(todo => ({
            ...todo,
            author: userStore.getUser(todo.authorId)
          }));
          
          this.setTodos(todos);
          this.emitChange();
          break;
      }
    });
  }
}
```

```
Store Dependencies Flow:

Action dispatched: LOAD_TODOS_SUCCESS
   │
   ├──→ UserStore processes first
   │    - Updates users
   │    - Emits change
   │
   └──→ TodoStore waits for UserStore
        - waitFor([userStore.token])
        - Once UserStore done, processes todos
        - Can now safely access user data
        - Emits change

Order guaranteed: UserStore → TodoStore → Views
```

## Angular Implementation with Flux Pattern

```typescript
// dispatcher.service.ts
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

export interface Action {
  type: string;
  payload?: any;
}

@Injectable({
  providedIn: 'root'
})
export class DispatcherService {
  private action$ = new Subject<Action>();
  
  // Observable for stores to subscribe to
  getActions() {
    return this.action$.asObservable();
  }
  
  dispatch(action: Action): void {
    this.action$.next(action);
  }
}

// todo.store.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { DispatcherService, Action } from './dispatcher.service';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

@Injectable({
  providedIn: 'root'
})
export class TodoStore {
  private todos$ = new BehaviorSubject<Todo[]>([]);
  
  constructor(private dispatcher: DispatcherService) {
    // Subscribe to all actions
    this.dispatcher.getActions().subscribe((action: Action) => {
      switch (action.type) {
        case 'ADD_TODO':
          this.addTodo(action.payload.text);
          break;
        case 'TOGGLE_TODO':
          this.toggleTodo(action.payload.id);
          break;
        case 'DELETE_TODO':
          this.deleteTodo(action.payload.id);
          break;
      }
    });
  }
  
  // Public observable for components
  getTodos(): Observable<Todo[]> {
    return this.todos$.asObservable();
  }
  
  private addTodo(text: string): void {
    const currentTodos = this.todos$.value;
    const newTodo: Todo = {
      id: `todo_${Date.now()}`,
      text,
      completed: false
    };
    this.todos$.next([...currentTodos, newTodo]);
  }
  
  private toggleTodo(id: string): void {
    const todos = this.todos$.value.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    );
    this.todos$.next(todos);
  }
  
  private deleteTodo(id: string): void {
    const todos = this.todos$.value.filter(todo => todo.id !== id);
    this.todos$.next(todos);
  }
}

// todo-actions.service.ts
import { Injectable } from '@angular/core';
import { DispatcherService } from './dispatcher.service';

@Injectable({
  providedIn: 'root'
})
export class TodoActions {
  constructor(private dispatcher: DispatcherService) {}
  
  addTodo(text: string): void {
    this.dispatcher.dispatch({
      type: 'ADD_TODO',
      payload: { text }
    });
  }
  
  toggleTodo(id: string): void {
    this.dispatcher.dispatch({
      type: 'TOGGLE_TODO',
      payload: { id }
    });
  }
  
  deleteTodo(id: string): void {
    this.dispatcher.dispatch({
      type: 'DELETE_TODO',
      payload: { id }
    });
  }
}

// todo-list.component.ts
import { Component } from '@angular/core';
import { TodoStore } from './todo.store';
import { TodoActions } from './todo-actions.service';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-todo-list',
  template: `
    <div>
      <form (ngSubmit)="addTodo()">
        <input [(ngModel)]="newTodoText" placeholder="What needs to be done?">
        <button type="submit">Add</button>
      </form>
      
      <ul>
        <li *ngFor="let todo of todos$ | async">
          <input
            type="checkbox"
            [checked]="todo.completed"
            (change)="toggleTodo(todo.id)"
          >
          <span [style.text-decoration]="todo.completed ? 'line-through' : 'none'">
            {{ todo.text }}
          </span>
          <button (click)="deleteTodo(todo.id)">Delete</button>
        </li>
      </ul>
    </div>
  `
})
export class TodoListComponent {
  todos$: Observable<any[]>;
  newTodoText = '';
  
  constructor(
    private todoStore: TodoStore,
    private todoActions: TodoActions
  ) {
    this.todos$ = this.todoStore.getTodos();
  }
  
  addTodo(): void {
    if (this.newTodoText.trim()) {
      this.todoActions.addTodo(this.newTodoText);
      this.newTodoText = '';
    }
  }
  
  toggleTodo(id: string): void {
    this.todoActions.toggleTodo(id);
  }
  
  deleteTodo(id: string): void {
    this.todoActions.deleteTodo(id);
  }
}
```

## Flux vs MVC

```
MVC (Bidirectional):
┌───────┐
│ Model │ ←─────→ ┌──────┐
└───┬───┘         │ View │
    ↕             └───┬──┘
┌────────────┐       ↕
│Controller │ ←──────┘
└────────────┘

Issues:
- Cascading updates
- Hard to trace changes
- Unpredictable state

Flux (Unidirectional):
┌────────┐   ┌────────────┐   ┌───────┐   ┌──────┐
│ Action │→→→│ Dispatcher │→→→│ Store │→→→│ View │
└────────┘   └────────────┘   └───────┘   └───┬──┘
    ↑                                          │
    └──────────────────────────────────────────┘

Benefits:
- Predictable data flow
- Easy to trace changes
- Debugging is straightforward
- Explicit state updates
```

## Common Mistakes

### 1. Modifying State in Actions

```typescript
// Bad: Action modifies state directly
const badAction = {
  addTodo(text: string) {
    const todo = { id: '1', text, completed: false };
    todoStore.todos.push(todo); // WRONG! Actions shouldn't modify stores
    return { type: 'ADD_TODO', payload: { text } };
  }
};

// Good: Action only describes what happened
const goodAction = {
  addTodo(text: string) {
    return {
      type: 'ADD_TODO',
      payload: { text }
    };
  }
};
```

### 2. Views Directly Modifying Stores

```typescript
// Bad: View directly modifies store
function TodoItem({ todo }) {
  const handleToggle = () => {
    todo.completed = !todo.completed; // WRONG! Breaks unidirectional flow
    todoStore.emitChange();
  };
  return <input type="checkbox" onChange={handleToggle} />;
}

// Good: View dispatches action
function TodoItem({ todo }) {
  const handleToggle = () => {
    AppDispatcher.dispatch(TodoActions.toggleTodo(todo.id));
  };
  return <input type="checkbox" onChange={handleToggle} />;
}
```

### 3. Dispatching During Dispatch

```typescript
// Bad: Dispatching inside a store's action handler
class BadStore extends EventEmitter {
  constructor() {
    AppDispatcher.register((action) => {
      if (action.type === 'ADD_TODO') {
        // WRONG! Can't dispatch during dispatch
        AppDispatcher.dispatch({ type: 'TODO_ADDED' });
      }
    });
  }
}

// Good: Use setTimeout to defer dispatch
class GoodStore extends EventEmitter {
  constructor() {
    AppDispatcher.register((action) => {
      if (action.type === 'ADD_TODO') {
        this.addTodo(action.payload);
        // Defer secondary action
        setTimeout(() => {
          AppDispatcher.dispatch({ type: 'TODO_ADDED' });
        }, 0);
      }
    });
  }
}
```

### 4. Storing Derived State

```typescript
// Bad: Storing computed values
class BadTodoStore {
  private todos: Todo[] = [];
  private completedCount: number = 0; // Duplicate data!
  
  addTodo(todo: Todo) {
    this.todos.push(todo);
    this.completedCount = this.todos.filter(t => t.completed).length;
  }
}

// Good: Calculate derived state on demand
class GoodTodoStore {
  private todos: Todo[] = [];
  
  getCompletedCount(): number {
    return this.todos.filter(t => t.completed).length;
  }
}
```

### 5. Multiple Stores for Same Domain

```typescript
// Bad: Splitting related data across stores
class TodoStore {
  private todos: Todo[] = [];
}
class TodoMetaStore {
  private completedIds: string[] = []; // Related to todos!
}

// Good: Keep related data together
class TodoStore {
  private todos: Map<string, Todo> = new Map();
  
  getCompletedTodos() {
    return Array.from(this.todos.values()).filter(t => t.completed);
  }
}
```

## Best Practices

### 1. Use Constants for Action Types

```typescript
// Prevents typos and enables autocomplete
export const ActionTypes = {
  ADD_TODO: 'ADD_TODO',
  TOGGLE_TODO: 'TOGGLE_TODO'
} as const;

// Type safety
type ActionType = typeof ActionTypes[keyof typeof ActionTypes];
```

### 2. Make Stores Immutable

```typescript
class TodoStore {
  private todos: Todo[] = [];
  
  // Bad: Returns mutable reference
  getTodos() {
    return this.todos;
  }
  
  // Good: Returns new array
  getTodos() {
    return [...this.todos];
  }
  
  // Better: Return frozen copy for strict immutability
  getTodos() {
    return Object.freeze([...this.todos]);
  }
}
```

### 3. Keep Actions Simple and Serializable

```typescript
// Good: Simple, serializable actions
const action = {
  type: 'ADD_TODO',
  payload: { text: 'Buy milk', dueDate: '2024-01-01' }
};

// Bad: Actions with functions or complex objects
const badAction = {
  type: 'ADD_TODO',
  payload: {
    getText: () => 'Buy milk', // Functions aren't serializable
    date: new Date() // Better as ISO string
  }
};
```

### 4. One Store Per Domain

```typescript
// Good: Organized by domain
class UserStore { /* user-related state */ }
class TodoStore { /* todo-related state */ }
class NotificationStore { /* notification state */ }

// Bad: Single mega-store
class AppStore {
  users = [];
  todos = [];
  notifications = [];
  settings = {};
  cart = {};
  // Everything in one place!
}
```

### 5. Use TypeScript for Type Safety

```typescript
// Define action types
type TodoAction =
  | { type: 'ADD_TODO'; payload: { text: string } }
  | { type: 'TOGGLE_TODO'; payload: { id: string } }
  | { type: 'DELETE_TODO'; payload: { id: string } };

// Type-safe dispatcher
class TypedDispatcher<T> {
  dispatch(action: T): void {
    // TypeScript ensures only valid actions can be dispatched
  }
}

const dispatcher = new TypedDispatcher<TodoAction>();
dispatcher.dispatch({ type: 'ADD_TODO', payload: { text: 'test' } }); // ✓
dispatcher.dispatch({ type: 'INVALID', payload: {} }); // ✗ Type error
```

## When to Use Flux

### Good Use Cases

1. **Complex data flow**: Multiple components need same data
2. **Shared state**: State used across different parts of the app
3. **Debugging requirements**: Need to trace state changes
4. **Team collaboration**: Multiple developers working on same codebase
5. **Time-travel debugging**: Need to replay actions
6. **Predictable state**: Need explicit state transitions

### When NOT to Use Flux

1. **Simple applications**: Few components, minimal shared state
2. **Mostly static content**: Content sites with little interaction
3. **Form-heavy apps**: React Hook Form or Formik might be better
4. **Server-driven UI**: Server components, minimal client state
5. **Prototypes**: Adds unnecessary complexity for quick prototypes

## Modern Alternatives to Flux

While Flux laid the foundation, modern tools have improved on the pattern:

```typescript
// Redux (Simplified Flux)
// - Single store instead of multiple
// - Reducers instead of stores
// - Middleware for async logic

// Zustand (Minimal Flux)
// - No dispatcher
// - No action creators
// - Simpler API

// Recoil/Jotai (Atomic Flux)
// - Atoms instead of stores
// - Fine-grained subscriptions
// - Better performance
```

## Interview Questions

### Q1: What problem does Flux solve?

**Answer:** Flux solves the complexity of bidirectional data flow in MVC architectures. In traditional MVC, models and views can update each other, creating cascading updates that are hard to predict and debug. Flux enforces unidirectional data flow: Actions → Dispatcher → Stores → Views, making state changes predictable and traceable. Facebook developed it to solve issues in their chat system where message count indicators wouldn't sync properly due to complex model/view dependencies.

### Q2: What's the difference between Flux and Redux?

**Answer:** 

**Flux:**
- Multiple stores (one per domain)
- Stores contain logic and state
- Explicit dispatcher
- No middleware concept
- Stores directly registered with dispatcher

**Redux:**
- Single store with combined reducers
- Pure reducer functions (no classes)
- Implicit dispatch via `store.dispatch()`
- Rich middleware ecosystem
- Time-travel debugging built-in
- Smaller bundle size

Redux simplified Flux by removing the explicit dispatcher, using pure functions instead of store classes, and providing better developer tools.

### Q3: Why is unidirectional data flow important?

**Answer:** Unidirectional data flow makes applications predictable and debuggable:

1. **Predictability**: Data flows in one direction (Actions → Stores → Views), so you always know where changes come from
2. **Debugging**: You can trace any state change back to the action that caused it
3. **Testing**: Easy to test each part in isolation
4. **Time-travel**: Can replay actions to reproduce bugs
5. **Reasoning**: Eliminates cascading updates where changes trigger other changes unpredictably

Without it, bidirectional flow creates webs of dependencies where it's unclear what will happen when state changes.

### Q4: What is the role of the dispatcher in Flux?

**Answer:** The dispatcher is the central hub that broadcasts actions to all registered stores. Key responsibilities:

1. **Broadcast actions**: Sends every action to every store
2. **Manage dependencies**: `waitFor()` ensures stores update in correct order
3. **Prevent nested dispatches**: Throws error if you dispatch during dispatch
4. **Single entry point**: All state changes go through dispatcher

Example:
```typescript
AppDispatcher.dispatch({ type: 'ADD_TODO', payload: { text: 'Buy milk' } });
// Dispatcher broadcasts to ALL stores
// Stores decide whether to respond based on action type
```

### Q5: How do stores communicate with each other in Flux?

**Answer:** Stores don't communicate directly - they use the dispatcher's `waitFor()` method to manage dependencies:

```typescript
class TodoStore {
  constructor() {
    this.dispatchToken = AppDispatcher.register((action) => {
      if (action.type === 'LOAD_DATA') {
        // Wait for UserStore to finish first
        AppDispatcher.waitFor([UserStore.dispatchToken]);
        // Now we can use UserStore data
        const author = UserStore.getUser(action.payload.authorId);
      }
    });
  }
}
```

This ensures deterministic update order while maintaining store independence. Stores never directly call methods on other stores.

### Q6: What are action creators and why use them?

**Answer:** Action creators are functions that create and return action objects. Benefits:

1. **Encapsulation**: Hide action structure details
2. **Type safety**: TypeScript can validate parameters
3. **Consistency**: Ensure actions have correct shape
4. **Documentation**: Function signature documents what's needed
5. **Testability**: Easy to test action creation separately
6. **Async logic**: Place to handle API calls

```typescript
// Without action creator
dispatch({ type: 'ADD_TODO', payload: { text: 'Buy milk', id: Date.now() }});

// With action creator
const addTodo = (text: string) => ({
  type: 'ADD_TODO',
  payload: { text, id: Date.now() }
});
dispatch(addTodo('Buy milk')); // Cleaner, type-safe
```

### Q7: How do you handle async operations in Flux?

**Answer:** Async operations are typically handled in action creators:

```typescript
const loadTodos = () => {
  // Dispatch loading action
  AppDispatcher.dispatch({ type: 'LOAD_TODOS_REQUEST' });
  
  // Async operation
  fetch('/api/todos')
    .then(res => res.json())
    .then(todos => {
      // Dispatch success action
      AppDispatcher.dispatch({
        type: 'LOAD_TODOS_SUCCESS',
        payload: { todos }
      });
    })
    .catch(error => {
      // Dispatch error action
      AppDispatcher.dispatch({
        type: 'LOAD_TODOS_ERROR',
        payload: { error }
      });
    });
};
```

This creates a clear flow: REQUEST → async operation → SUCCESS/ERROR, allowing stores to update state at each stage (loading, success, error).

### Q8: What are the limitations of Flux?

**Answer:** 

1. **Boilerplate**: Requires actions, dispatcher, stores, action creators
2. **Multiple stores complexity**: Managing dependencies between stores
3. **No standardization**: Many Flux implementations, no official one
4. **Learning curve**: Concepts like dispatcher, waitFor are unfamiliar
5. **Over-engineering**: Too complex for simple apps
6. **No built-in async**: Need custom solutions for async logic
7. **No time-travel**: Unlike Redux, no built-in time-travel debugging

These limitations led to Redux and other modern state management solutions that simplify while keeping unidirectional flow.

## Key Takeaways

- Flux enforces unidirectional data flow: Actions → Dispatcher → Stores → Views
- Actions are plain objects describing what happened
- Dispatcher is the central hub broadcasting actions to all stores
- Stores contain application state and business logic
- Views subscribe to stores and re-render on changes
- Store dependencies are managed with `waitFor()`
- Actions should be simple, serializable, and descriptive
- Stores should provide read-only access to state
- Never dispatch during a dispatch
- Flux makes state changes predictable and traceable
- Better for complex applications with shared state
- Modern alternatives (Redux, Zustand) build on Flux principles
- TypeScript adds valuable type safety to Flux pattern
- Keep one store per domain for better organization
- Use action constants to prevent typos and enable autocomplete

## Resources

- [Flux Official Documentation](https://facebook.github.io/flux/)
- [Flux Architecture Explained](https://github.com/facebook/flux/tree/main/examples)
- [In-Depth Overview of Flux](https://facebook.github.io/flux/docs/in-depth-overview)
- [Flux vs Redux](https://redux.js.org/understanding/history-and-design/prior-art#flux)
- [Original Flux Announcement](https://www.youtube.com/watch?v=nYkdrAPrdcw)
- [Flux Dispatcher Source](https://github.com/facebook/flux/blob/main/src/Dispatcher.js)
- [Flux Design Patterns](https://github.com/krasimir/react-in-patterns/blob/master/book/chapter-08/README.md)
- [Comparing Flux Implementations](https://www.slant.co/topics/6477/~flux-implementations)
