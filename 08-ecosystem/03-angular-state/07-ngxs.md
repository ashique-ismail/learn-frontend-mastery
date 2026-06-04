# NGXS - State Management for Angular

## Table of Contents
- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core Concepts)
- [State](#state)
- [Actions](#actions)
- [Selectors](#selectors)
- [Plugins](#plugins)
- [Advanced Features](#advanced-features)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

NGXS is a state management pattern and library for Angular. It provides a simple and more opinionated approach compared to NgRx, using decorators and classes to define state, actions, and operations.

**Key Features:**
- Class-based approach with decorators
- Less boilerplate than NgRx
- Built-in async actions
- Plugin system
- TypeScript-first
- DevTools integration

## Installation & Setup

```bash
# Install NGXS
npm install @ngxs/store

# DevTools
npm install @ngxs/devtools-plugin --save-dev

# Other plugins
npm install @ngxs/logger-plugin @ngxs/storage-plugin
```

### Basic Setup

```typescript
// app.config.ts (Standalone)
import { provideStore } from '@ngxs/store';
import { TodosState } from './state/todos.state';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore([TodosState]),
  ],
};

// Or Module-based
import { NgxsModule } from '@ngxs/store';

@NgModule({
  imports: [
    NgxsModule.forRoot([TodosState]),
  ],
})
export class AppModule {}
```

## Core Concepts

### 1. State Class

```typescript
// todos.state.ts
import { State, Action, StateContext, Selector } from '@ngxs/store';
import { Injectable } from '@angular/core';

export interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

export interface TodosStateModel {
  todos: Todo[];
  loading: boolean;
}

@State<TodosStateModel>({
  name: 'todos',
  defaults: {
    todos: [],
    loading: false,
  },
})
@Injectable()
export class TodosState {
  @Selector()
  static getTodos(state: TodosStateModel) {
    return state.todos;
  }

  @Selector()
  static isLoading(state: TodosStateModel) {
    return state.loading;
  }

  @Action(AddTodo)
  addTodo(ctx: StateContext<TodosStateModel>, action: AddTodo) {
    const state = ctx.getState();
    ctx.patchState({
      todos: [...state.todos, action.todo],
    });
  }
}
```

### 2. Using in Components

```typescript
import { Component } from '@angular/core';
import { Store, Select } from '@ngxs/store';
import { Observable } from 'rxjs';
import { TodosState } from './state/todos.state';
import { AddTodo, RemoveTodo } from './state/todos.actions';

@Component({
  selector: 'app-todos',
  template: `
    <div *ngFor="let todo of todos$ | async">
      {{ todo.title }}
      <button (click)="remove(todo.id)">Remove</button>
    </div>
    <button (click)="add()">Add Todo</button>
  `,
})
export class TodosComponent {
  @Select(TodosState.getTodos) todos$!: Observable<Todo[]>;
  @Select(TodosState.isLoading) loading$!: Observable<boolean>;

  constructor(private store: Store) {}

  add() {
    this.store.dispatch(new AddTodo({
      id: Date.now().toString(),
      title: 'New Todo',
      completed: false,
    }));
  }

  remove(id: string) {
    this.store.dispatch(new RemoveTodo(id));
  }
}
```

## State

### 1. State Definition

```typescript
export interface UserStateModel {
  user: User | null;
  token: string | null;
  loading: boolean;
  error: string | null;
}

@State<UserStateModel>({
  name: 'user',
  defaults: {
    user: null,
    token: null,
    loading: false,
    error: null,
  },
})
@Injectable()
export class UserState {
  // State implementation
}
```

### 2. State Context Methods

```typescript
@State<TodosStateModel>({ name: 'todos', defaults: { todos: [] } })
@Injectable()
export class TodosState {
  @Action(UpdateTodos)
  updateTodos(ctx: StateContext<TodosStateModel>, action: UpdateTodos) {
    // Get current state
    const state = ctx.getState();

    // Set entire state
    ctx.setState({ todos: action.todos });

    // Patch state (merge)
    ctx.patchState({ loading: false });

    // Dispatch another action
    ctx.dispatch(new LoadTodos());

    // Dispatch multiple actions
    ctx.dispatch([new Action1(), new Action2()]);
  }
}
```

### 3. Sub-States

```typescript
export interface AppStateModel {
  user: UserStateModel;
  settings: SettingsStateModel;
}

@State<AppStateModel>({
  name: 'app',
  defaults: {
    user: { /* ... */ },
    settings: { /* ... */ },
  },
  children: [UserState, SettingsState],
})
@Injectable()
export class AppState {}
```

## Actions

### 1. Defining Actions

```typescript
// todos.actions.ts
export class AddTodo {
  static readonly type = '[Todos] Add Todo';
  constructor(public todo: Todo) {}
}

export class RemoveTodo {
  static readonly type = '[Todos] Remove Todo';
  constructor(public id: string) {}
}

export class LoadTodos {
  static readonly type = '[Todos] Load Todos';
}

export class LoadTodosSuccess {
  static readonly type = '[Todos] Load Todos Success';
  constructor(public todos: Todo[]) {}
}

export class LoadTodosFailure {
  static readonly type = '[Todos] Load Todos Failure';
  constructor(public error: string) {}
}
```

### 2. Action Handlers

```typescript
@State<TodosStateModel>({ name: 'todos', defaults: { todos: [], loading: false } })
@Injectable()
export class TodosState {
  constructor(private todoService: TodoService) {}

  @Action(AddTodo)
  addTodo(ctx: StateContext<TodosStateModel>, action: AddTodo) {
    const state = ctx.getState();
    ctx.setState({
      ...state,
      todos: [...state.todos, action.todo],
    });
  }

  @Action(RemoveTodo)
  removeTodo(ctx: StateContext<TodosStateModel>, action: RemoveTodo) {
    const state = ctx.getState();
    ctx.setState({
      ...state,
      todos: state.todos.filter(t => t.id !== action.id),
    });
  }

  @Action(LoadTodos)
  loadTodos(ctx: StateContext<TodosStateModel>) {
    ctx.patchState({ loading: true });

    return this.todoService.getTodos().pipe(
      tap(todos => {
        ctx.setState({
          todos,
          loading: false,
        });
      }),
      catchError(error => {
        ctx.patchState({ loading: false });
        return throwError(error);
      })
    );
  }
}
```

### 3. Async Actions

```typescript
@State<UserStateModel>({ name: 'user', defaults: { user: null, loading: false } })
@Injectable()
export class UserState {
  constructor(private userService: UserService) {}

  @Action(LoadUser)
  loadUser(ctx: StateContext<UserStateModel>, action: LoadUser) {
    ctx.patchState({ loading: true });

    // Return observable - NGXS handles subscription
    return this.userService.getUser(action.id).pipe(
      tap(user => {
        ctx.patchState({
          user,
          loading: false,
        });
      }),
      catchError(error => {
        ctx.patchState({ loading: false, error: error.message });
        return throwError(error);
      })
    );
  }
}
```

## Selectors

### 1. Static Selectors

```typescript
@State<TodosStateModel>({ name: 'todos', defaults: { todos: [] } })
@Injectable()
export class TodosState {
  @Selector()
  static getTodos(state: TodosStateModel) {
    return state.todos;
  }

  @Selector()
  static getActiveTodos(state: TodosStateModel) {
    return state.todos.filter(t => !t.completed);
  }

  @Selector()
  static getTodoCount(state: TodosStateModel) {
    return state.todos.length;
  }
}

// Usage in component
@Select(TodosState.getTodos) todos$!: Observable<Todo[]>;
@Select(TodosState.getTodoCount) count$!: Observable<number>;
```

### 2. Parameterized Selectors

```typescript
@State<TodosStateModel>({ name: 'todos', defaults: { todos: [] } })
@Injectable()
export class TodosState {
  @Selector()
  static getTodoById(state: TodosStateModel) {
    return (id: string) => {
      return state.todos.find(t => t.id === id);
    };
  }

  @Selector()
  static getTodosByStatus(state: TodosStateModel) {
    return (completed: boolean) => {
      return state.todos.filter(t => t.completed === completed);
    };
  }
}

// Usage
this.store.select(TodosState.getTodoById).pipe(
  map(filterFn => filterFn('123'))
);

this.store.select(TodosState.getTodosByStatus).pipe(
  map(filterFn => filterFn(true))
);
```

### 3. Composed Selectors

```typescript
@State<UserStateModel>({ name: 'user', defaults: { user: null } })
@Injectable()
export class UserState {
  @Selector()
  static getUser(state: UserStateModel) {
    return state.user;
  }

  @Selector()
  static getUserName(state: UserStateModel) {
    return state.user?.name;
  }
}

@State<TodosStateModel>({ name: 'todos', defaults: { todos: [] } })
@Injectable()
export class TodosState {
  @Selector()
  static getTodos(state: TodosStateModel) {
    return state.todos;
  }

  // Compose with other selectors
  @Selector([TodosState.getTodos, UserState.getUserName])
  static getUserTodos(todos: Todo[], userName: string) {
    return todos.filter(t => t.owner === userName);
  }
}
```

### 4. Memoized Selectors

```typescript
import { createSelector } from '@ngxs/store';

@State<ProductsStateModel>({ name: 'products', defaults: { products: [] } })
@Injectable()
export class ProductsState {
  @Selector()
  static getProducts(state: ProductsStateModel) {
    return state.products;
  }

  // Custom memoized selector
  static getExpensiveProducts = createSelector(
    [ProductsState.getProducts],
    (products: Product[]) => {
      console.log('Computing expensive operation');
      return products.map(p => ({
        ...p,
        computed: heavyComputation(p),
      }));
    }
  );
}
```

## Plugins

### 1. Logger Plugin

```typescript
import { NgxsLoggerPluginModule } from '@ngxs/logger-plugin';

@NgModule({
  imports: [
    NgxsModule.forRoot([]),
    NgxsLoggerPluginModule.forRoot({
      disabled: environment.production,
      collapsed: false,
      logger: console,
    }),
  ],
})
export class AppModule {}
```

### 2. Storage Plugin

```typescript
import { NgxsStoragePluginModule, StorageOption } from '@ngxs/storage-plugin';

@NgModule({
  imports: [
    NgxsModule.forRoot([TodosState, UserState]),
    NgxsStoragePluginModule.forRoot({
      key: ['todos', 'user.token'], // States to persist
      storage: StorageOption.LocalStorage,
    }),
  ],
})
export class AppModule {}
```

### 3. DevTools Plugin

```typescript
import { NgxsReduxDevtoolsPluginModule } from '@ngxs/devtools-plugin';

@NgModule({
  imports: [
    NgxsModule.forRoot([]),
    NgxsReduxDevtoolsPluginModule.forRoot({
      disabled: environment.production,
      maxAge: 25,
      name: 'My App',
    }),
  ],
})
export class AppModule {}
```

### 4. Form Plugin

```typescript
import { NgxsFormPluginModule } from '@ngxs/form-plugin';

@NgModule({
  imports: [
    NgxsModule.forRoot([FormState]),
    NgxsFormPluginModule.forRoot(),
  ],
})
export class AppModule {}

// In state
export interface FormStateModel {
  userForm: {
    model: UserForm;
    dirty: boolean;
    status: string;
    errors: any;
  };
}

// In component
import { UpdateFormValue } from '@ngxs/form-plugin';

this.store.dispatch(new UpdateFormValue({
  path: 'userForm',
  value: { name: 'John' },
}));
```

## Advanced Features

### 1. Action Lifecycle

```typescript
@State<TodosStateModel>({ name: 'todos', defaults: { todos: [] } })
@Injectable()
export class TodosState {
  @Action(AddTodo, { cancelUncompleted: true }) // Cancel previous
  addTodo(ctx: StateContext<TodosStateModel>, action: AddTodo) {
    // Implementation
  }

  @Action([LoadTodos, RefreshTodos]) // Multiple actions
  loadOrRefresh(ctx: StateContext<TodosStateModel>) {
    // Handle both actions
  }
}
```

### 2. Reset State

```typescript
import { NgxsModuleOptions } from '@ngxs/store';

export const ngxsConfig: NgxsModuleOptions = {
  developmentMode: !environment.production,
};

// In state
@Action(ResetState)
resetState(ctx: StateContext<TodosStateModel>) {
  ctx.setState({
    todos: [],
    loading: false,
  });
}
```

### 3. State Operators

```typescript
import { patch, append, removeItem, updateItem } from '@ngxs/store/operators';

@Action(AddTodo)
addTodo(ctx: StateContext<TodosStateModel>, action: AddTodo) {
  ctx.setState(
    patch({
      todos: append([action.todo]),
    })
  );
}

@Action(RemoveTodo)
removeTodo(ctx: StateContext<TodosStateModel>, action: RemoveTodo) {
  ctx.setState(
    patch({
      todos: removeItem<Todo>(t => t.id === action.id),
    })
  );
}

@Action(UpdateTodo)
updateTodo(ctx: StateContext<TodosStateModel>, action: UpdateTodo) {
  ctx.setState(
    patch({
      todos: updateItem<Todo>(
        t => t.id === action.id,
        patch({ completed: true })
      ),
    })
  );
}
```

### 4. Action Chaining

```typescript
@Action(CreateOrder)
createOrder(ctx: StateContext<OrderStateModel>, action: CreateOrder) {
  return this.orderService.create(action.order).pipe(
    tap(order => {
      ctx.patchState({ order });
      // Dispatch another action
      ctx.dispatch(new ClearCart());
    })
  );
}

@Action(ClearCart)
clearCart(ctx: StateContext<CartStateModel>) {
  ctx.setState({ items: [] });
}
```

## Performance Optimization

### 1. Memoization

```typescript
// Selectors are automatically memoized
@Selector()
static getExpensiveData(state: StateModel) {
  console.log('Computing...'); // Only logs when dependencies change
  return heavyComputation(state.data);
}
```

### 2. OnPush Detection

```typescript
@Component({
  selector: 'app-todos',
  template: `<div *ngFor="let todo of todos$ | async">{{ todo.title }}</div>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class TodosComponent {
  @Select(TodosState.getTodos) todos$!: Observable<Todo[]>;
}
```

### 3. State Snapshots

```typescript
// Use snapshot for one-time reads
const currentState = this.store.snapshot();
const todos = currentState.todos.todos;

// Instead of subscribing
this.store.select(TodosState.getTodos).pipe(take(1)).subscribe(...);
```

## Common Mistakes

### 1. Mutating State

```typescript
// ❌ Wrong
@Action(AddTodo)
addTodo(ctx: StateContext<TodosStateModel>, action: AddTodo) {
  const state = ctx.getState();
  state.todos.push(action.todo); // Mutation!
  ctx.setState(state);
}

// ✅ Correct
@Action(AddTodo)
addTodo(ctx: StateContext<TodosStateModel>, action: AddTodo) {
  ctx.setState(
    patch({ todos: append([action.todo]) })
  );
}
```

### 2. Not Returning Observables

```typescript
// ❌ Wrong
@Action(LoadTodos)
loadTodos(ctx: StateContext<TodosStateModel>) {
  this.service.getTodos().subscribe(todos => {
    ctx.patchState({ todos });
  });
}

// ✅ Correct
@Action(LoadTodos)
loadTodos(ctx: StateContext<TodosStateModel>) {
  return this.service.getTodos().pipe(
    tap(todos => ctx.patchState({ todos }))
  );
}
```

### 3. Subscribing in Actions

```typescript
// ❌ Wrong
@Action(LoadTodos)
loadTodos(ctx: StateContext<TodosStateModel>) {
  this.service.getTodos().subscribe(...); // Don't subscribe
}

// ✅ Correct
@Action(LoadTodos)
loadTodos(ctx: StateContext<TodosStateModel>) {
  return this.service.getTodos(); // Return observable
}
```

### 4. Missing Injectable

```typescript
// ❌ Wrong
@State<TodosStateModel>({ name: 'todos', defaults: {} })
export class TodosState {} // Missing @Injectable

// ✅ Correct
@State<TodosStateModel>({ name: 'todos', defaults: {} })
@Injectable()
export class TodosState {}
```

### 5. Incorrect Selector Usage

```typescript
// ❌ Wrong
@Select('todos') todos$!: Observable<any>; // Stringly typed

// ✅ Correct
@Select(TodosState.getTodos) todos$!: Observable<Todo[]>;
```

## Best Practices

### 1. Organize by Feature

```
src/
  app/
    todos/
      state/
        todos.actions.ts
        todos.state.ts
      todos.component.ts
```

### 2. Use State Operators

```typescript
import { patch, append, removeItem } from '@ngxs/store/operators';

// Cleaner than manual immutable updates
ctx.setState(
  patch({
    todos: append([newTodo]),
  })
);
```

### 3. Type Actions

```typescript
export class LoadTodos {
  static readonly type = '[Todos] Load Todos';
  constructor(public userId: string) {}
}

// Fully typed action
ctx.dispatch(new LoadTodos('123'));
```

### 4. Use Async Patterns

```typescript
// Let NGXS handle async
@Action(LoadData)
loadData(ctx: StateContext<StateModel>) {
  return this.service.load().pipe(
    tap(data => ctx.patchState({ data }))
  );
}
```

### 5. Leverage Plugins

```typescript
// Use built-in plugins
NgxsStoragePluginModule.forRoot({ key: 'session' });
NgxsLoggerPluginModule.forRoot();
NgxsReduxDevtoolsPluginModule.forRoot();
```

## When to Use

### Use NGXS When

1. **Less boilerplate** - Want simpler than NgRx
2. **Class-based** - Prefer OOP over functional
3. **Quick setup** - Need to get started fast
4. **Plugin ecosystem** - Built-in storage, forms, etc.
5. **TypeScript focus** - Strong typing support

### Alternatives

- **NgRx Store** - More functional, larger ecosystem
- **Akita** - Less opinionated than NGXS
- **Elf** - More modern, tree-shakeable
- **NgRx Component Store** - Local state only

## Interview Questions

### 1. What is NGXS and how does it differ from NgRx?

**Answer:** NGXS is class-based state management using decorators:
- **Less boilerplate**: No separate reducers/effects
- **Decorators**: @State, @Action, @Selector
- **Async built-in**: Actions can return observables
- **Plugins**: Storage, forms, logger included
- **Simpler API**: Easier to learn

NgRx is more functional and has larger ecosystem.

### 2. Explain the @Action decorator.

**Answer:**
```typescript
@Action(AddTodo)
addTodo(ctx: StateContext<StateModel>, action: AddTodo) {
  ctx.patchState({ todos: [...ctx.getState().todos, action.todo] });
}
```
@Action marks method to handle specific action type. Context provides getState, setState, patchState, and dispatch.

### 3. What are state operators in NGXS?

**Answer:** Operators help with immutable updates:
```typescript
import { patch, append, removeItem, updateItem } from '@ngxs/store/operators';

ctx.setState(
  patch({
    todos: append([newTodo]),
  })
);
```
Cleaner than manual spreading, prevent mutations, composable.

### 4. How do you handle async actions?

**Answer:** Return observable from action handler:
```typescript
@Action(LoadData)
loadData(ctx: StateContext<StateModel>) {
  return this.service.load().pipe(
    tap(data => ctx.patchState({ data }))
  );
}
```
NGXS subscribes automatically and handles errors.

### 5. What plugins does NGXS provide?

**Answer:**
- **Storage**: Persist state to localStorage/sessionStorage
- **Logger**: Console logging of actions
- **DevTools**: Redux DevTools integration
- **Forms**: Two-way binding with Angular forms
- **Router**: Router state integration
- **WebSocket**: WebSocket handling

## Key Takeaways

1. **NGXS uses classes and decorators** for state, actions, and selectors
2. **Less boilerplate than NgRx** - no separate reducers or effects
3. **Async actions built-in** - return observables from action handlers
4. **State operators** simplify immutable updates
5. **Plugin ecosystem** - storage, forms, logger, DevTools
6. **Selectors are memoized** automatically for performance
7. **StateContext provides** getState, setState, patchState, dispatch
8. **Type-safe** with TypeScript throughout
9. **Sub-states** for organizing large state trees
10. **Action lifecycle hooks** for cancellation and multiple handlers

## Resources

### Official Documentation
- [NGXS Docs](https://www.ngxs.io/)
- [API Reference](https://www.ngxs.io/api)
- [Recipes](https://www.ngxs.io/recipes)

### Tutorials
- [NGXS Tutorial](https://www.ngxs.io/getting-started)
- [Austin's NGXS Course](https://ultimatecourses.com/courses/angular/ngxs)

### Plugins
- [Storage Plugin](https://www.ngxs.io/plugins/storage)
- [Form Plugin](https://www.ngxs.io/plugins/form)

### Community
- [GitHub Repository](https://github.com/ngxs/store)
- [Slack Community](https://ngxs.slack.com)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/ngxs)
