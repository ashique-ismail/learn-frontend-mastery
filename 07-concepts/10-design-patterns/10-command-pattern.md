# Command Pattern

## Overview

The Command pattern encapsulates a request as an object, separating the what (the command) from the who (the invoker) and the how (the receiver). Commands are first-class objects — they can be stored, queued, logged, undone, retried, and composed. In frontend applications, the Command pattern is the foundation for undo/redo systems, operation history, macro recording, and offline-capable action queues.

## Core Structure

```
Command Pattern roles:

  Command Interface          defines execute() and optionally undo()
       ↑
  ConcreteCommand            implements the action; holds receiver reference
       |
  Invoker                    stores and executes commands; manages history
       |
  Receiver                   the actual object that does the work
  (Editor, State, Store)
```

```typescript
// Command interface
interface Command {
  execute(): void;
  undo(): void;
  description: string;
}

// Receiver — the object being operated on
class TextEditor {
  private content: string = '';

  insert(text: string, position: number): void {
    this.content =
      this.content.slice(0, position) + text + this.content.slice(position);
  }

  delete(position: number, length: number): void {
    this.content =
      this.content.slice(0, position) + this.content.slice(position + length);
  }

  getContent(): string {
    return this.content;
  }
}

// Concrete Command — Insert text
class InsertTextCommand implements Command {
  description = 'Insert text';

  constructor(
    private editor: TextEditor,
    private text: string,
    private position: number
  ) {}

  execute(): void {
    this.editor.insert(this.text, this.position);
  }

  undo(): void {
    this.editor.delete(this.position, this.text.length);
  }
}

// Concrete Command — Delete text
class DeleteTextCommand implements Command {
  description = 'Delete text';
  private deletedText: string = '';

  constructor(
    private editor: TextEditor,
    private position: number,
    private length: number,
    private getContent: () => string
  ) {}

  execute(): void {
    // Capture deleted text before deleting (needed for undo)
    this.deletedText = this.getContent().slice(
      this.position,
      this.position + this.length
    );
    this.editor.delete(this.position, this.length);
  }

  undo(): void {
    this.editor.insert(this.deletedText, this.position);
  }
}
```

## Invoker with Undo/Redo History

```typescript
// Invoker — manages command history, enables undo/redo
class CommandHistory {
  private history: Command[] = [];
  private undoneStack: Command[] = [];
  private maxHistorySize: number;

  constructor(maxHistorySize = 100) {
    this.maxHistorySize = maxHistorySize;
  }

  execute(command: Command): void {
    command.execute();
    this.history.push(command);

    // Executing a new command clears the redo stack
    this.undoneStack = [];

    // Enforce max history size
    if (this.history.length > this.maxHistorySize) {
      this.history.shift(); // remove oldest
    }
  }

  undo(): Command | undefined {
    const command = this.history.pop();
    if (!command) return undefined;

    command.undo();
    this.undoneStack.push(command);
    return command;
  }

  redo(): Command | undefined {
    const command = this.undoneStack.pop();
    if (!command) return undefined;

    command.execute();
    this.history.push(command);
    return command;
  }

  canUndo(): boolean {
    return this.history.length > 0;
  }

  canRedo(): boolean {
    return this.undoneStack.length > 0;
  }

  getHistory(): string[] {
    return this.history.map((c) => c.description);
  }
}

// Usage
const editor = new TextEditor();
const history = new CommandHistory();

// Execute operations
history.execute(new InsertTextCommand(editor, 'Hello, ', 0));
history.execute(new InsertTextCommand(editor, 'World!', 7));
console.log(editor.getContent()); // "Hello, World!"

// Undo last operation
history.undo();
console.log(editor.getContent()); // "Hello, "

// Redo
history.redo();
console.log(editor.getContent()); // "Hello, World!"
```

## React Integration — Undo/Redo Hook

```typescript
import { useState, useCallback, useRef } from 'react';

interface UndoableCommand<T> {
  execute(state: T): T;
  undo(state: T): T;
  description?: string;
}

function useUndoable<T>(initialState: T) {
  const [state, setState] = useState<T>(initialState);
  const historyRef = useRef<UndoableCommand<T>[]>([]);
  const undoneRef = useRef<UndoableCommand<T>[]>([]);

  const execute = useCallback((command: UndoableCommand<T>) => {
    setState((current) => {
      const newState = command.execute(current);
      historyRef.current = [...historyRef.current, command];
      undoneRef.current = []; // clear redo stack on new action
      return newState;
    });
  }, []);

  const undo = useCallback(() => {
    const command = historyRef.current[historyRef.current.length - 1];
    if (!command) return;

    setState((current) => {
      const newState = command.undo(current);
      historyRef.current = historyRef.current.slice(0, -1);
      undoneRef.current = [...undoneRef.current, command];
      return newState;
    });
  }, []);

  const redo = useCallback(() => {
    const command = undoneRef.current[undoneRef.current.length - 1];
    if (!command) return;

    setState((current) => {
      const newState = command.execute(current);
      undoneRef.current = undoneRef.current.slice(0, -1);
      historyRef.current = [...historyRef.current, command];
      return newState;
    });
  }, []);

  return {
    state,
    execute,
    undo,
    redo,
    canUndo: historyRef.current.length > 0,
    canRedo: undoneRef.current.length > 0,
  };
}

// Usage — Canvas drawing app
interface DrawingState {
  shapes: Shape[];
}

const addShape: UndoableCommand<DrawingState> = {
  description: 'Add circle',
  execute(state) {
    return { shapes: [...state.shapes, newCircle] };
  },
  undo(state) {
    return { shapes: state.shapes.slice(0, -1) };
  },
};

function DrawingCanvas() {
  const { state, execute, undo, redo, canUndo, canRedo } = useUndoable<DrawingState>({
    shapes: [],
  });

  return (
    <div>
      <Canvas shapes={state.shapes} />
      <button onClick={() => execute(addShape)}>Add Shape</button>
      <button onClick={undo} disabled={!canUndo}>Undo</button>
      <button onClick={redo} disabled={!canRedo}>Redo</button>
    </div>
  );
}
```

## Async Commands

```typescript
// Async command interface
interface AsyncCommand<T = void> {
  execute(): Promise<T>;
  undo(): Promise<void>;
  description: string;
  canUndo: boolean;
}

// Async command — API mutation + cache invalidation
class CreatePostCommand implements AsyncCommand<Post> {
  description = 'Create post';
  canUndo = true;
  private createdPostId: string | null = null;

  constructor(
    private postData: CreatePostInput,
    private apiClient: PostsApi,
    private queryClient: QueryClient
  ) {}

  async execute(): Promise<Post> {
    const post = await this.apiClient.create(this.postData);
    this.createdPostId = post.id;
    // Invalidate cache so list refreshes
    await this.queryClient.invalidateQueries({ queryKey: ['posts'] });
    return post;
  }

  async undo(): Promise<void> {
    if (!this.createdPostId) return;
    await this.apiClient.delete(this.createdPostId);
    await this.queryClient.invalidateQueries({ queryKey: ['posts'] });
    this.createdPostId = null;
  }
}

// Async invoker
class AsyncCommandHistory {
  private history: AsyncCommand[] = [];
  private undoneStack: AsyncCommand[] = [];
  private isProcessing = false;

  async execute(command: AsyncCommand): Promise<void> {
    if (this.isProcessing) throw new Error('Command already in progress');

    this.isProcessing = true;
    try {
      await command.execute();
      if (command.canUndo) {
        this.history.push(command);
        this.undoneStack = [];
      }
    } finally {
      this.isProcessing = false;
    }
  }

  async undo(): Promise<void> {
    const command = this.history.pop();
    if (!command) return;

    this.isProcessing = true;
    try {
      await command.undo();
      this.undoneStack.push(command);
    } finally {
      this.isProcessing = false;
    }
  }
}
```

## Macro Commands (Composing Commands)

```typescript
// MacroCommand — composite; executes multiple commands as one
class MacroCommand implements Command {
  description: string;
  private commands: Command[];

  constructor(description: string, ...commands: Command[]) {
    this.description = description;
    this.commands = commands;
  }

  execute(): void {
    this.commands.forEach((c) => c.execute());
  }

  undo(): void {
    // Undo in reverse order
    [...this.commands].reverse().forEach((c) => c.undo());
  }
}

// Example: "format paragraph" = indent + bold + add line break
const formatParagraph = new MacroCommand(
  'Format paragraph',
  new IndentCommand(editor, 4),
  new BoldCommand(editor, 0, 20),
  new InsertTextCommand(editor, '\n', content.length)
);

history.execute(formatParagraph);
// All three execute together — undo reverses all three
```

## Offline Action Queue

```typescript
// Queue commands for offline execution
class OfflineCommandQueue {
  private queue: SerializableCommand[] = [];
  private isOnline = navigator.onLine;

  constructor() {
    window.addEventListener('online', () => this.flush());
    window.addEventListener('offline', () => { this.isOnline = false; });
    // Load pending commands from localStorage on startup
    this.queue = JSON.parse(localStorage.getItem('commandQueue') || '[]');
  }

  enqueue(command: SerializableCommand): void {
    if (this.isOnline) {
      command.execute();
    } else {
      this.queue.push(command);
      this.persist();
    }
  }

  private async flush(): Promise<void> {
    this.isOnline = true;
    while (this.queue.length > 0) {
      const command = this.queue.shift()!;
      try {
        await command.execute();
      } catch (err) {
        // On failure, put it back and stop
        this.queue.unshift(command);
        break;
      }
    }
    this.persist();
  }

  private persist(): void {
    localStorage.setItem('commandQueue', JSON.stringify(this.queue));
  }
}
```

## Common Mistakes

### 1. Not Capturing State for Undo at Execute Time

```typescript
// ❌ Undo depends on current state — fragile if state has changed
class DeleteRowCommand {
  undo(): void {
    // Problem: what was the deleted row's content?
    // If we didn't save it, we can't restore it
    apiClient.restoreRow(this.rowId); // but rowId alone isn't enough!
  }
}

// ✅ Capture all needed state at execute() time
class DeleteRowCommand {
  private deletedRow: Row | null = null;

  execute(): void {
    this.deletedRow = currentState.rows.find(r => r.id === this.rowId)!;
    // Now we have everything needed for undo
    deleteRow(this.rowId);
  }

  undo(): void {
    if (this.deletedRow) restoreRow(this.deletedRow);
  }
}
```

### 2. Allowing Undo After Non-Undoable Operations

```typescript
// ❌ History contains operations that can't be undone
// After a "send email" command, "undo" would need to unsend an email — impossible

// ✅ Mark commands as undoable or not
interface Command {
  canUndo: boolean;
}

class SendEmailCommand implements Command {
  canUndo = false; // cannot undo a sent email
}

// History management: when a non-undoable command is executed,
// clear the history (you can't undo past it)
execute(command: Command): void {
  command.execute();
  if (!command.canUndo) {
    this.history = []; // clear history after non-undoable action
  } else {
    this.history.push(command);
  }
}
```

## Interview Questions

### 1. What is the Command pattern and what problem does it solve?

**Answer:** The Command pattern wraps a request (an action and its parameters) as an object. This solves three problems: (1) decoupling the invoker (who triggers the action) from the receiver (who performs it), enabling the invoker to be written without knowing anything about the receiver; (2) enabling undo/redo by storing command objects with an `undo()` method; (3) enabling queueing, logging, retry, and serialization of operations — a command object can be stored in an array, serialized to JSON, or sent over the network in ways that a function call cannot. In frontend, it's the natural pattern for any system with operation history.

### 2. How do you implement undo/redo using the Command pattern?

**Answer:** Maintain two stacks: a history stack (executed commands) and an undo stack (undone commands). `execute()` runs the command, pushes to history, clears the undo stack. `undo()` pops from history, calls `command.undo()`, pushes to undo stack. `redo()` pops from undo stack, calls `command.execute()`, pushes to history. The key requirement: each command must capture enough state at `execute()` time to perform `undo()` later — for example, a delete command must save the deleted content before deleting it. Commands that cannot be undone (email sent, payment processed) should either be excluded from history or should clear the history stack when executed.

### 3. How does the Command pattern enable offline-capable applications?

**Answer:** Commands are serializable objects — they can be stored in `localStorage` or IndexedDB when the network is unavailable. When the user goes offline, actions are enqueued rather than executed immediately. When connectivity returns, the queue flushes in order. Each command carries all the information needed to execute asynchronously — no need to reconstruct context. This enables optimistic UI (reflect the action immediately) + deferred server sync (execute when online). Conflict resolution (what if the server state changed while offline) must still be handled, but the command queue provides the ordered log of operations needed for any reconciliation strategy.
