# Optimistic UI Updates with Rollback

## The Idea

**In plain English:** Instead of waiting for the server to confirm a mutation before updating the screen, you apply the change to the UI immediately — using a temporary client-generated ID — and let the network call happen in the background. If the server responds with success, you swap the temporary ID for the real one. If the server fails, you silently undo the change and put the UI back exactly as it was.

**Real-world analogy:** Think of a bank teller in the pre-digital era.

- When you hand over a deposit slip, the teller **immediately writes the amount in your passbook** and hands it back. You leave happy.
- In the background, the slip goes to a back-office clerk who processes it against the central ledger.
- If the clerk finds a problem (wrong account number, rubber cheque), the teller calls you back and **crosses out the entry** in your passbook — a rollback.
- The teller did not make you stand at the counter while the clerk finished. That would be pessimistic UI.

The mutation follows a strict state machine: `idle → pending → success | error`. On `error`, the previous snapshot is restored. The temporary ID (created with `crypto.randomUUID()`) acts as a placeholder in the list until the server returns the canonical ID.

---

## Learning Objectives

- Understand the full mutation state machine: `idle / pending / success / error`
- Generate collision-free client-side IDs using `crypto.randomUUID()`
- Capture a pre-mutation snapshot to enable deterministic rollback
- Implement optimistic updates with TanStack Query `useMutation` (`onMutate / onError / onSettled`)
- Implement the same pattern in Angular using signals and a service-level state machine
- Expose mutation status to the UI for loading spinners and error toasts without blocking interaction
- Avoid the common pitfalls: double-submit, ID mismatch after re-fetch, and rollback on partial server failure

---

## Why CSS Alone Isn't Enough

This section is not applicable — optimistic UI is a pure data and state management concern. CSS is used only to style the pending and error states once the state machine surfaces them.

---

## HTML & CSS Foundation

### Pending and Error Visual States

The state machine exposes a `status` string. CSS class bindings translate that string into visual feedback without blocking the interaction.

```html
<!-- A single todo item in all three visual states -->

<!-- Idle — normal -->
<li class="todo-item">
  <span class="todo-label">Buy milk</span>
  <button class="todo-delete">Delete</button>
</li>

<!-- Pending — dimmed, spinner, delete button disabled -->
<li class="todo-item todo-item--pending" aria-busy="true">
  <span class="todo-label">Buy milk</span>
  <span class="todo-spinner" aria-hidden="true"></span>
  <button class="todo-delete" disabled>Delete</button>
</li>

<!-- Error — red border, retry affordance -->
<li class="todo-item todo-item--error">
  <span class="todo-label">Buy milk</span>
  <span class="todo-error-badge" role="alert">Failed to save</span>
  <button class="todo-delete">Delete</button>
</li>
```

```css
/* ─── Tokens ─── */
:root {
  --color-pending-bg:  #f0f4ff;
  --color-pending-txt: #6b7280;
  --color-error-bg:    #fff1f2;
  --color-error-border:#f87171;
  --spinner-size:      18px;
  --item-transition:   opacity 0.2s ease, background-color 0.2s ease;
}

/* ─── Base item ─── */
.todo-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  background: #fff;
  transition: var(--item-transition);
}

/* ─── Pending state ─── */
.todo-item--pending {
  background: var(--color-pending-bg);
  opacity: 0.75;
  pointer-events: none;   /* prevents double-click while in-flight */
}

.todo-item--pending .todo-label {
  color: var(--color-pending-txt);
}

/* CSS spinner — no JS needed for the animation itself */
.todo-spinner {
  display: inline-block;
  width: var(--spinner-size);
  height: var(--spinner-size);
  border: 2px solid #c7d2fe;
  border-top-color: #4f46e5;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
  flex-shrink: 0;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* ─── Error state ─── */
.todo-item--error {
  background: var(--color-error-bg);
  border-color: var(--color-error-border);
}

.todo-error-badge {
  margin-left: auto;
  font-size: 0.75rem;
  color: #dc2626;
  font-weight: 500;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .todo-spinner {
    animation: none;
    border-top-color: #4f46e5;
    opacity: 0.6;
  }
  .todo-item {
    transition: none;
  }
}
```

**What CSS owns:** visual state transitions, spinner animation, reduced-motion fallback, pointer-events blocking during in-flight mutations.

**What CSS cannot own:** which item is pending, rollback logic, ID reconciliation, error recovery.

---

## React Implementation

### Types and the Mutation State Machine

```tsx
// types.ts
export type MutationStatus = 'idle' | 'pending' | 'success' | 'error';

export interface Todo {
  id: string;          // server-assigned UUID after success; temp UUID before
  text: string;
  completed: boolean;
}

export interface OptimisticTodo extends Todo {
  _status: MutationStatus;   // per-item mutation status
  _isOptimistic?: boolean;   // true while the item has a temp ID
}
```

### TanStack Query — `useMutation` with Optimistic Context

```tsx
// useTodos.ts
import { useQueryClient, useMutation, useQuery } from '@tanstack/react-query';
import type { Todo, OptimisticTodo } from './types';

const TODOS_KEY = ['todos'] as const;

async function fetchTodos(): Promise<Todo[]> {
  const res = await fetch('/api/todos');
  if (!res.ok) throw new Error('Failed to fetch todos');
  return res.json();
}

async function createTodo(text: string): Promise<Todo> {
  const res = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text }),
  });
  if (!res.ok) throw new Error('Failed to create todo');
  return res.json();   // server returns { id: <canonical UUID>, text, completed }
}

export function useTodos() {
  const queryClient = useQueryClient();

  const { data: todos = [], isLoading } = useQuery({
    queryKey: TODOS_KEY,
    queryFn: fetchTodos,
  });

  const addTodo = useMutation({
    mutationFn: createTodo,

    // 1. onMutate runs BEFORE the network call.
    //    It returns a context object that onError receives for rollback.
    onMutate: async (text: string) => {
      // Cancel any outgoing refetches for this query so they don't
      // overwrite the optimistic item we are about to inject.
      await queryClient.cancelQueries({ queryKey: TODOS_KEY });

      // Snapshot the current list — this is our rollback target.
      const previousTodos = queryClient.getQueryData<Todo[]>(TODOS_KEY);

      // Build the optimistic item with a collision-free temporary ID.
      const optimisticTodo: OptimisticTodo = {
        id: crypto.randomUUID(),   // temp ID — will be replaced after success
        text,
        completed: false,
        _status: 'pending',
        _isOptimistic: true,
      };

      // Immediately push the optimistic item into the cached list.
      queryClient.setQueryData<Todo[]>(TODOS_KEY, old => [
        ...(old ?? []),
        optimisticTodo,
      ]);

      // Return snapshot so onError can restore it.
      return { previousTodos, optimisticId: optimisticTodo.id };
    },

    // 2. onError fires if the mutation throws.
    //    context.previousTodos is the snapshot we returned from onMutate.
    onError: (_err, _text, context) => {
      if (context?.previousTodos !== undefined) {
        queryClient.setQueryData(TODOS_KEY, context.previousTodos);
      }
    },

    // 3. onSettled fires on BOTH success and error.
    //    Re-fetch from the server to reconcile the canonical ID.
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: TODOS_KEY });
    },
  });

  return { todos, isLoading, addTodo };
}
```

### TodoList Component

```tsx
// TodoList.tsx
import { useState, useId } from 'react';
import { useTodos } from './useTodos';
import type { OptimisticTodo } from './types';

export function TodoList() {
  const { todos, isLoading, addTodo } = useTodos();
  const [text, setText]               = useState('');
  const inputId                       = useId();

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const trimmed = text.trim();
    if (!trimmed) return;
    setText('');
    addTodo.mutate(trimmed);
  }

  if (isLoading) {
    return <p aria-busy="true">Loading todos…</p>;
  }

  return (
    <section aria-label="Todo list">
      <form onSubmit={handleSubmit} aria-label="Add a new todo">
        <label htmlFor={inputId}>New todo</label>
        <input
          id={inputId}
          value={text}
          onChange={e => setText(e.target.value)}
          placeholder="What needs doing?"
          autoComplete="off"
        />
        <button type="submit" disabled={addTodo.isPending && text.trim() === ''}>
          Add
        </button>
      </form>

      {/* Global mutation error — for when we want a toast instead of per-item error */}
      {addTodo.isError && (
        <p role="alert" style={{ color: 'red' }}>
          Could not save. Please try again.
        </p>
      )}

      <ul aria-label="Todos" style={{ listStyle: 'none', padding: 0 }}>
        {(todos as OptimisticTodo[]).map(todo => (
          <TodoItem key={todo.id} todo={todo} />
        ))}
      </ul>
    </section>
  );
}

/* ── Single todo item ── */
interface TodoItemProps {
  todo: OptimisticTodo;
}

function TodoItem({ todo }: TodoItemProps) {
  const isPending = todo._status === 'pending' || todo._isOptimistic;
  const isError   = todo._status === 'error';

  let className = 'todo-item';
  if (isPending) className += ' todo-item--pending';
  if (isError)   className += ' todo-item--error';

  return (
    <li className={className} aria-busy={isPending || undefined}>
      <span className="todo-label">{todo.text}</span>

      {isPending && (
        <span className="todo-spinner" aria-hidden="true" />
      )}

      {isError && (
        <span className="todo-error-badge" role="alert">
          Failed to save
        </span>
      )}

      <button
        className="todo-delete"
        disabled={isPending}
        aria-label={`Delete ${todo.text}`}
      >
        Delete
      </button>
    </li>
  );
}
```

### Per-Item Delete with Rollback

```tsx
// useDeleteTodo.ts — same onMutate/onError pattern, but for removal
import { useQueryClient, useMutation } from '@tanstack/react-query';
import type { Todo } from './types';

const TODOS_KEY = ['todos'] as const;

async function deleteTodo(id: string): Promise<void> {
  const res = await fetch(`/api/todos/${id}`, { method: 'DELETE' });
  if (!res.ok) throw new Error('Delete failed');
}

export function useDeleteTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: deleteTodo,

    onMutate: async (id: string) => {
      await queryClient.cancelQueries({ queryKey: TODOS_KEY });
      const previousTodos = queryClient.getQueryData<Todo[]>(TODOS_KEY);

      // Optimistically remove the item immediately
      queryClient.setQueryData<Todo[]>(TODOS_KEY, old =>
        (old ?? []).filter(t => t.id !== id)
      );

      return { previousTodos };
    },

    onError: (_err, _id, context) => {
      if (context?.previousTodos !== undefined) {
        queryClient.setQueryData(TODOS_KEY, context.previousTodos);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: TODOS_KEY });
    },
  });
}
```

---

## Angular Implementation

### Model and Service State Machine

```typescript
// todo.model.ts
export type MutationStatus = 'idle' | 'pending' | 'success' | 'error';

export interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

export interface OptimisticTodo extends Todo {
  _status: MutationStatus;
  _isOptimistic?: boolean;
}
```

```typescript
// todo.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import type { Todo, OptimisticTodo, MutationStatus } from './todo.model';

@Injectable({ providedIn: 'root' })
export class TodoService {
  private http = inject(HttpClient);

  // Source of truth — array of todos including in-flight optimistic items
  private _todos    = signal<OptimisticTodo[]>([]);
  private _status   = signal<MutationStatus>('idle');
  private _error    = signal<string | null>(null);

  readonly todos    = this._todos.asReadonly();
  readonly status   = this._status.asReadonly();
  readonly error    = this._error.asReadonly();
  readonly isPending = computed(() => this._status() === 'pending');

  async loadTodos(): Promise<void> {
    const data = await firstValueFrom(
      this.http.get<Todo[]>('/api/todos')
    );
    this._todos.set(data.map(t => ({ ...t, _status: 'idle' })));
  }

  async addTodo(text: string): Promise<void> {
    // 1. Snapshot for rollback
    const snapshot = this._todos();

    // 2. Create optimistic item with a temporary collision-free ID
    const tempId = crypto.randomUUID();
    const optimistic: OptimisticTodo = {
      id: tempId,
      text,
      completed: false,
      _status: 'pending',
      _isOptimistic: true,
    };

    // 3. Immediately update the UI
    this._todos.update(list => [...list, optimistic]);
    this._status.set('pending');
    this._error.set(null);

    try {
      // 4. Call the server
      const created = await firstValueFrom(
        this.http.post<Todo>('/api/todos', { text })
      );

      // 5. Replace the temp item with the real server item
      this._todos.update(list =>
        list.map(t =>
          t.id === tempId
            ? { ...created, _status: 'success', _isOptimistic: false }
            : t
        )
      );
      this._status.set('success');
    } catch (err) {
      // 6. Rollback to snapshot on any error
      this._todos.set(snapshot);
      this._status.set('error');
      this._error.set('Failed to save. Please try again.');
    }
  }

  async deleteTodo(id: string): Promise<void> {
    const snapshot = this._todos();

    // Mark item as pending in-place
    this._todos.update(list =>
      list.map(t => t.id === id ? { ...t, _status: 'pending' } : t)
    );

    try {
      await firstValueFrom(this.http.delete(`/api/todos/${id}`));
      // Remove on success
      this._todos.update(list => list.filter(t => t.id !== id));
    } catch {
      // Restore snapshot on failure
      this._todos.set(snapshot);
      this._status.set('error');
      this._error.set('Delete failed. The item has been restored.');
    }
  }
}
```

### Standalone TodoList Component

```typescript
// todo-list.component.ts
import {
  Component, OnInit, inject, signal
} from '@angular/core';
import { NgFor, NgIf, NgClass } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { TodoService } from './todo.service';

@Component({
  selector: 'app-todo-list',
  standalone: true,
  imports: [NgFor, NgIf, NgClass, FormsModule],
  template: `
    <section aria-label="Todo list">

      <!-- Add form -->
      <form (ngSubmit)="handleSubmit()" aria-label="Add a new todo">
        <label for="newTodo">New todo</label>
        <input
          id="newTodo"
          [(ngModel)]="newText"
          name="newTodo"
          placeholder="What needs doing?"
          autocomplete="off"
        />
        <button type="submit" [disabled]="todoService.isPending()">
          Add
        </button>
      </form>

      <!-- Global error banner -->
      <p
        *ngIf="todoService.error()"
        role="alert"
        style="color: red;"
      >
        {{ todoService.error() }}
      </p>

      <!-- Todo list -->
      <ul aria-label="Todos" style="list-style:none; padding:0;">
        <li
          *ngFor="let todo of todoService.todos(); trackBy: trackById"
          class="todo-item"
          [ngClass]="{
            'todo-item--pending': todo._status === 'pending' || todo._isOptimistic,
            'todo-item--error':   todo._status === 'error'
          }"
          [attr.aria-busy]="todo._status === 'pending' || null"
        >
          <span class="todo-label">{{ todo.text }}</span>

          <span
            *ngIf="todo._status === 'pending' || todo._isOptimistic"
            class="todo-spinner"
            aria-hidden="true"
          ></span>

          <span
            *ngIf="todo._status === 'error'"
            class="todo-error-badge"
            role="alert"
          >
            Failed to save
          </span>

          <button
            class="todo-delete"
            [disabled]="todo._status === 'pending' || !!todo._isOptimistic"
            [attr.aria-label]="'Delete ' + todo.text"
            (click)="todoService.deleteTodo(todo.id)"
          >
            Delete
          </button>
        </li>
      </ul>
    </section>
  `,
})
export class TodoListComponent implements OnInit {
  todoService = inject(TodoService);
  newText     = '';

  ngOnInit() {
    this.todoService.loadTodos();
  }

  async handleSubmit() {
    const trimmed = this.newText.trim();
    if (!trimmed) return;
    this.newText = '';
    await this.todoService.addTodo(trimmed);
  }

  trackById(_: number, todo: { id: string }) {
    return todo.id;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| In-flight item is announced as busy | `aria-busy="true"` on the `<li>` while `_status === 'pending'` |
| Error message is announced immediately | `role="alert"` on the error badge forces an assertive announcement |
| Global mutation error is announced | `role="alert"` on the error banner paragraph |
| Spinner is hidden from screen readers | `aria-hidden="true"` on `.todo-spinner` — the `aria-busy` on the parent communicates loading |
| Delete button disabled while item is in-flight | Prevents double-submit; communicated to AT via `disabled` attribute |
| Rollback is silent for screen readers | State is restored in memory; no DOM announcement needed unless re-announcing causes confusion |
| New item appears in list order | Appended at the end — screen reader virtual cursor discovers it naturally on next read |
| Form input has a visible label | `<label for="newTodo">` explicitly associated; not just a placeholder |
| Reduced motion respected | `@media (prefers-reduced-motion: reduce)` disables spinner animation |
| Focus is not stolen on mutation | Mutations are fire-and-forget; no forced focus changes occur during optimistic update |

---

## Production Pitfalls

**1. ID mismatch after re-fetch invalidates the list reconciliation**
After `onSettled` triggers a re-fetch, the server returns the real ID. If your list component uses index-based keys instead of `key={todo.id}` (React) or `trackBy: trackById` (Angular), the entire list re-renders from scratch and the just-confirmed item flickers. Always key by a stable ID. The temp UUID satisfies this requirement because it remains stable until the item is replaced by the canonical ID via `setQueryData`.

**2. Double-submit during the pending window**
If `pointer-events: none` is the only guard, a keyboard user can still activate the form a second time. Also disable the submit button while the mutation is in-flight (`addTodo.isPending` in TanStack Query; `todoService.isPending()` in Angular). Both approaches are necessary — CSS and attribute-level disabling.

**3. Forgetting to cancel in-flight queries in `onMutate`**
Without `cancelQueries`, a background refetch that completes after `setQueryData` will overwrite the optimistic item with stale server data. The `await queryClient.cancelQueries()` call in `onMutate` is not optional.

**4. Rollback on partial server success (batch mutations)**
If you optimistically apply three items at once and the server successfully creates two but fails on the third, rolling back the entire snapshot removes the two successful items from the UI until the next re-fetch. For batch operations, track per-item status individually rather than a single snapshot, and only roll back the failed item's segment.

**5. `crypto.randomUUID()` is not available in older environments**
`crypto.randomUUID()` requires a secure context (HTTPS or localhost) and is not available in IE11 or some older WebViews. Add a polyfill or fallback: `() => \`temp-\${Date.now()}-\${Math.random().toString(36).slice(2)}\`` for environments that need it.

**6. Server errors that return 2xx with an error payload**
Some APIs return `200 OK` with `{ success: false, message: "..." }` in the body. A naive `if (!res.ok) throw` check will treat this as success, skip `onError`, and leave the optimistic item with a temp ID in the UI forever. Always validate the response body shape, not only the HTTP status.

**7. Optimistic item left in the list when the component unmounts mid-flight**
If the user navigates away while a mutation is in-flight, the `onError` rollback fires but there is no component to receive the signal. In TanStack Query this is safe because the cache is global. In the Angular signal service, ensure the signal outlives the component (use `providedIn: 'root'`) or cancel pending requests with `takeUntilDestroyed()`.

---

## Interview Angle

**Q: "How do you implement optimistic UI without ending up in an inconsistent state when the server returns an error?"**

The key is a three-phase contract: before the network call fires (`onMutate`), you take a deep snapshot of the current query cache and return it as the mutation context. The optimistic item — stamped with `crypto.randomUUID()` as its temporary ID — is injected into the cache immediately. If the server returns an error, `onError` receives that context object and restores the snapshot atomically via `setQueryData`, reversing the UI change in a single synchronous operation. Regardless of success or failure, `onSettled` triggers a cache invalidation and re-fetch to reconcile the canonical server state. This means the UI is never authoritative for longer than one round-trip: it makes a bet, and the server either confirms it or cancels it.

**Follow-up: "How does TanStack Query's approach differ from writing the same pattern by hand in a signal service?"**

In TanStack Query the snapshot-context-rollback contract is baked into the `useMutation` API: the return value of `onMutate` is automatically forwarded to `onError` and `onSettled` as `context`. You never wire that plumbing manually. In an Angular signal service you own the entire state machine: you must explicitly store the snapshot in a local variable before the `try` block, and in the `catch` block you call `this._todos.set(snapshot)` yourself. The signal approach is more portable (no library dependency) but more error-prone — a developer can accidentally reassign `snapshot` before the `catch` runs, or forget to cover the `finally` path. The TanStack approach is safer by convention; the Angular approach requires discipline.
