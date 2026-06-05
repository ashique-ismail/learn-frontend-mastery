# Implement Undo/Redo

## The Idea

**In plain English:** Undo/Redo is a feature that lets you reverse your last action (undo) or bring it back (redo). The program keeps two ordered lists — a "past" list of everything you've done and a "future" list of things you've undone — so it can travel backward and forward through your history of changes.

**Real-world analogy:** Imagine a stack of paper trays on a desk. As you write pages and finish them, you place each page face-down in the LEFT tray (your past). When you realise you made a mistake, you lift the top page off the LEFT tray and put it face-down in the RIGHT tray (your future) — that is undo. When you want it back, you take the top page from the RIGHT tray and put it back in the LEFT tray — that is redo. A new page you write immediately clears out everything in the RIGHT tray, because you have moved on to a new timeline.

- The left tray = the "past" stack (every previous state you saved)
- The right tray = the "future" stack (states you have undone but could restore)
- The page you are currently writing = the "present" state (what is on screen right now)

---

## Core Data Structure

Keep a stack of past states and a stack of future states:

```text
past:    [state0, state1, state2]
present: state3
future:  []

After undo:
past:    [state0, state1]
present: state2
future:  [state3]

After redo:
past:    [state0, state1, state2]
present: state3
future:  []
```

---

## `useUndoRedo` Hook

```ts
interface UndoState<T> {
  past: T[];
  present: T;
  future: T[];
}

function useUndoRedo<T>(initialState: T, maxHistory = 50) {
  const [state, dispatch] = useReducer(
    undoRedoReducer<T>,
    { past: [], present: initialState, future: [] }
  );

  const set = useCallback((newPresent: T) => {
    dispatch({ type: 'SET', payload: newPresent });
  }, []);

  const undo = useCallback(() => dispatch({ type: 'UNDO' }), []);
  const redo = useCallback(() => dispatch({ type: 'REDO' }), []);
  const reset = useCallback((newState: T) => dispatch({ type: 'RESET', payload: newState }), []);

  return {
    state: state.present,
    set,
    undo,
    redo,
    reset,
    canUndo: state.past.length > 0,
    canRedo: state.future.length > 0,
    historyLength: state.past.length,
  };
}

function undoRedoReducer<T>(
  state: UndoState<T>,
  action: { type: string; payload?: T },
  maxHistory = 50
): UndoState<T> {
  switch (action.type) {
    case 'SET': {
      if (action.payload === state.present) return state; // no change
      return {
        past: [...state.past, state.present].slice(-maxHistory),
        present: action.payload!,
        future: [], // clear redo stack on new action
      };
    }
    case 'UNDO': {
      if (state.past.length === 0) return state;
      const previous = state.past[state.past.length - 1];
      return {
        past: state.past.slice(0, -1),
        present: previous,
        future: [state.present, ...state.future],
      };
    }
    case 'REDO': {
      if (state.future.length === 0) return state;
      const next = state.future[0];
      return {
        past: [...state.past, state.present],
        present: next,
        future: state.future.slice(1),
      };
    }
    case 'RESET':
      return { past: [], present: action.payload!, future: [] };
    default:
      return state;
  }
}
```

---

## Usage: Text Editor

```tsx
function TextEditor() {
  const { state: text, set, undo, redo, canUndo, canRedo } = useUndoRedo('');

  // Keyboard shortcuts
  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      const isMac = navigator.platform.toUpperCase().includes('MAC');
      const modifier = isMac ? e.metaKey : e.ctrlKey;

      if (modifier && e.key === 'z') {
        e.preventDefault();
        if (e.shiftKey) redo();
        else undo();
      }
      if (modifier && e.key === 'y') {
        e.preventDefault();
        redo();
      }
    }
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [undo, redo]);

  return (
    <div>
      <toolbar>
        <button onClick={undo} disabled={!canUndo} aria-label="Undo">↩</button>
        <button onClick={redo} disabled={!canRedo} aria-label="Redo">↪</button>
      </toolbar>
      <textarea
        value={text}
        onChange={e => set(e.target.value)}
      />
    </div>
  );
}
```

---

## Command Pattern Alternative

Better for complex operations (move, resize, delete) where you can implement inverse operations:

```ts
interface Command {
  execute(): void;
  undo(): void;
  description: string;
}

class UndoManager {
  private undoStack: Command[] = [];
  private redoStack: Command[] = [];

  execute(command: Command) {
    command.execute();
    this.undoStack.push(command);
    this.redoStack = []; // clear redo
  }

  undo() {
    const command = this.undoStack.pop();
    if (!command) return;
    command.undo();
    this.redoStack.push(command);
  }

  redo() {
    const command = this.redoStack.pop();
    if (!command) return;
    command.execute();
    this.undoStack.push(command);
  }
}

// Example command
class MoveItemCommand implements Command {
  constructor(
    private item: Item,
    private from: Position,
    private to: Position,
    private setPosition: (pos: Position) => void
  ) {}

  execute() { this.setPosition(this.to); }
  undo() { this.setPosition(this.from); }
  description = `Move item from ${this.from} to ${this.to}`;
}
```

---

## Efficient State with Immer

For complex nested state, use Immer to avoid copying the entire tree:

```ts
import { produce } from 'immer';

function useUndoRedoImmer<T>(initialState: T) {
  const { state, set, undo, redo, canUndo, canRedo } = useUndoRedo(initialState);

  const update = useCallback((recipe: (draft: T) => void) => {
    set(produce(state, recipe));
  }, [state, set]);

  return { state, update, undo, redo, canUndo, canRedo };
}

// Usage
const { state: todos, update } = useUndoRedoImmer(initialTodos);

function addTodo(text: string) {
  update(draft => {
    draft.push({ id: Date.now(), text, done: false });
  });
}
```

---

## Common Interview Questions

**Q: Why does setting new state clear the redo stack?**
After you undo a few steps and then make a new change, you've created a new timeline. The "future" states from the old timeline are no longer reachable. Most applications (Word, VS Code) follow this model. Some (Photoshop) implement branching history instead.

**Q: How do you limit memory usage?**
Apply a `maxHistory` limit and slice the `past` array: `past.slice(-maxHistory)`. The oldest states are dropped when the limit is reached.

**Q: State snapshots vs commands — which is better?**
State snapshots are simpler to implement. Commands are more memory-efficient for large state (store the operation, not the full state copy) and enable describing history to users ("Undo: Move 5 items").
