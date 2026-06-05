# Optimistic Updates

## The Idea

**In plain English:** An optimistic update is when an app immediately shows you the result of your action (like a like count going up) before it has actually confirmed with the server that it worked — it assumes (optimistically) that it will succeed, and quietly corrects itself if it doesn't.

**Real-world analogy:** Imagine you write a message on a sticky note and slap it on a shared whiteboard before waiting to hear back that there's space for it. Everyone can read it right away. If the board manager later tells you there's no room, you peel it off and the board goes back to how it was.

- The sticky note appearing instantly = the UI updating before the server responds
- The board manager confirming space = the server successfully saving the change
- Peeling the note off = rolling back the UI when the server returns an error

---

## Overview

Optimistic Updates are a UX pattern where the UI is updated immediately to reflect the expected result of an action, before receiving confirmation from the server. This creates a perception of instant responsiveness, significantly improving the user experience by eliminating waiting time for server round-trips.

If the server request fails, the UI rolls back to the previous state and shows an error. This pattern is commonly used in social media apps, collaborative tools, and any application where immediate feedback is crucial.

## Core Concepts

### Traditional vs Optimistic Approach

```javascript
// Traditional: Wait for server response
function LikeButton({ postId, initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);
  const [loading, setLoading] = useState(false);

  const handleLike = async () => {
    setLoading(true);
    try {
      const response = await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
      });
      const data = await response.json();
      setLikes(data.likes); // Update after server responds
    } catch (error) {
      console.error('Failed to like post');
    } finally {
      setLoading(false);
    }
  };

  return (
    <button onClick={handleLike} disabled={loading}>
      {loading ? 'Liking...' : `♥ ${likes}`}
    </button>
  );
}

// Optimistic: Update immediately, rollback on error
function LikeButton({ postId, initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);
  const [isLiked, setIsLiked] = useState(false);

  const handleLike = async () => {
    // Optimistically update UI immediately
    const previousLikes = likes;
    const previousIsLiked = isLiked;
    
    setLikes(likes + 1);
    setIsLiked(true);

    try {
      await fetch(`/api/posts/${postId}/like`, {
        method: 'POST',
      });
      // Success - optimistic update was correct!
    } catch (error) {
      // Rollback on error
      setLikes(previousLikes);
      setIsLiked(previousIsLiked);
      toast.error('Failed to like post');
    }
  };

  return (
    <button onClick={handleLike} className={isLiked ? 'liked' : ''}>
      ♥ {likes}
    </button>
  );
}
```

## Basic Patterns

### Simple Optimistic Update Hook

```javascript
function useOptimistic(serverAction) {
  const [isUpdating, setIsUpdating] = useState(false);
  const [error, setError] = useState(null);

  const execute = async (optimisticUpdate, rollback) => {
    setIsUpdating(true);
    setError(null);

    // Apply optimistic update immediately
    optimisticUpdate();

    try {
      // Perform server action
      const result = await serverAction();
      setIsUpdating(false);
      return result;
    } catch (err) {
      // Rollback on error
      rollback();
      setError(err.message);
      setIsUpdating(false);
      throw err;
    }
  };

  return { execute, isUpdating, error };
}

// Usage
function TodoItem({ todo, onUpdate }) {
  const [completed, setCompleted] = useState(todo.completed);
  const { execute, error } = useOptimistic(() =>
    fetch(`/api/todos/${todo.id}`, {
      method: 'PATCH',
      body: JSON.stringify({ completed: !completed }),
    })
  );

  const handleToggle = () => {
    const previousCompleted = completed;

    execute(
      () => setCompleted(!completed), // Optimistic
      () => setCompleted(previousCompleted) // Rollback
    );
  };

  return (
    <div>
      <input
        type="checkbox"
        checked={completed}
        onChange={handleToggle}
      />
      {todo.title}
      {error && <span className="error">{error}</span>}
    </div>
  );
}
```

### React 19 useOptimistic Hook

```javascript
import { useOptimistic } from 'react';

function TodoList({ todos }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo) => [...state, { ...newTodo, pending: true }]
  );

  const handleAddTodo = async (title) => {
    const tempId = Date.now();
    
    // Add optimistic todo
    addOptimisticTodo({ id: tempId, title, completed: false });

    try {
      // Send to server
      await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ title }),
      });
      // Server will send updated list via props
    } catch (error) {
      toast.error('Failed to add todo');
      // State automatically rolls back
    }
  };

  return (
    <ul>
      {optimisticTodos.map(todo => (
        <li
          key={todo.id}
          className={todo.pending ? 'pending' : ''}
        >
          {todo.title}
        </li>
      ))}
    </ul>
  );
}
```

## Advanced Patterns

### Optimistic Updates with Undo

```javascript
function useOptimisticWithUndo(serverAction) {
  const [history, setHistory] = useState([]);
  const [currentState, setCurrentState] = useState(null);

  const execute = async (newState) => {
    // Save current state for undo
    setHistory(prev => [...prev, currentState]);
    
    // Apply optimistic update
    setCurrentState(newState);

    try {
      const result = await serverAction(newState);
      // Clear undo history on success
      setHistory([]);
      return result;
    } catch (error) {
      // Keep history for manual undo
      toast.error('Update failed. Click undo to revert.', {
        action: {
          label: 'Undo',
          onClick: undo,
        },
      });
      throw error;
    }
  };

  const undo = () => {
    if (history.length === 0) return;
    
    const previousState = history[history.length - 1];
    setCurrentState(previousState);
    setHistory(prev => prev.slice(0, -1));
  };

  return { state: currentState, execute, undo, canUndo: history.length > 0 };
}

// Usage
function Editor() {
  const { state, execute, undo, canUndo } = useOptimisticWithUndo(
    (content) => fetch('/api/save', {
      method: 'POST',
      body: JSON.stringify({ content }),
    })
  );

  return (
    <div>
      <textarea
        value={state?.content || ''}
        onChange={(e) => execute({ content: e.target.value })}
      />
      {canUndo && <button onClick={undo}>Undo</button>}
    </div>
  );
}
```

### Optimistic List Operations

```javascript
function useOptimisticList(initialItems) {
  const [items, setItems] = useState(initialItems);
  const [pendingOps, setPendingOps] = useState(new Map());

  const addItem = async (item) => {
    const tempId = `temp-${Date.now()}`;
    const optimisticItem = { ...item, id: tempId, pending: true };

    // Add optimistically
    setItems(prev => [...prev, optimisticItem]);
    setPendingOps(prev => new Map(prev).set(tempId, 'add'));

    try {
      const response = await fetch('/api/items', {
        method: 'POST',
        body: JSON.stringify(item),
      });
      const savedItem = await response.json();

      // Replace temp item with real one
      setItems(prev =>
        prev.map(i => (i.id === tempId ? savedItem : i))
      );
      setPendingOps(prev => {
        const next = new Map(prev);
        next.delete(tempId);
        return next;
      });
    } catch (error) {
      // Remove on error
      setItems(prev => prev.filter(i => i.id !== tempId));
      setPendingOps(prev => {
        const next = new Map(prev);
        next.delete(tempId);
        return next;
      });
      throw error;
    }
  };

  const updateItem = async (id, updates) => {
    // Save original
    const original = items.find(i => i.id === id);
    
    // Update optimistically
    setItems(prev =>
      prev.map(i => (i.id === id ? { ...i, ...updates } : i))
    );
    setPendingOps(prev => new Map(prev).set(id, 'update'));

    try {
      await fetch(`/api/items/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(updates),
      });
      setPendingOps(prev => {
        const next = new Map(prev);
        next.delete(id);
        return next;
      });
    } catch (error) {
      // Restore original
      setItems(prev =>
        prev.map(i => (i.id === id ? original : i))
      );
      setPendingOps(prev => {
        const next = new Map(prev);
        next.delete(id);
        return next;
      });
      throw error;
    }
  };

  const deleteItem = async (id) => {
    // Save for rollback
    const deletedItem = items.find(i => i.id === id);
    const index = items.findIndex(i => i.id === id);

    // Remove optimistically
    setItems(prev => prev.filter(i => i.id !== id));
    setPendingOps(prev => new Map(prev).set(id, 'delete'));

    try {
      await fetch(`/api/items/${id}`, { method: 'DELETE' });
      setPendingOps(prev => {
        const next = new Map(prev);
        next.delete(id);
        return next;
      });
    } catch (error) {
      // Restore at original position
      setItems(prev => {
        const next = [...prev];
        next.splice(index, 0, deletedItem);
        return next;
      });
      setPendingOps(prev => {
        const next = new Map(prev);
        next.delete(id);
        return next;
      });
      throw error;
    }
  };

  return {
    items,
    pendingOps,
    addItem,
    updateItem,
    deleteItem,
  };
}

// Usage
function TodoList() {
  const { items, pendingOps, addItem, updateItem, deleteItem } =
    useOptimisticList([]);

  return (
    <ul>
      {items.map(item => (
        <li
          key={item.id}
          className={pendingOps.has(item.id) ? 'pending' : ''}
        >
          <input
            type="checkbox"
            checked={item.completed}
            onChange={() =>
              updateItem(item.id, { completed: !item.completed })
            }
          />
          {item.title}
          <button onClick={() => deleteItem(item.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

## Integration with React Query

### Optimistic Updates with React Query

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useTodoMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (todo) =>
      fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(todo),
      }).then(r => r.json()),
    
    onMutate: async (newTodo) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot previous value
      const previousTodos = queryClient.getQueryData(['todos']);

      // Optimistically update
      queryClient.setQueryData(['todos'], (old) => [
        ...old,
        { ...newTodo, id: Date.now(), pending: true },
      ]);

      // Return context with snapshot
      return { previousTodos };
    },

    onError: (err, newTodo, context) => {
      // Rollback on error
      queryClient.setQueryData(['todos'], context.previousTodos);
      toast.error('Failed to add todo');
    },

    onSettled: () => {
      // Refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}

// Usage
function AddTodo() {
  const [title, setTitle] = useState('');
  const mutation = useTodoMutation();

  const handleSubmit = (e) => {
    e.preventDefault();
    mutation.mutate({ title, completed: false });
    setTitle('');
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Add todo..."
      />
      <button type="submit" disabled={mutation.isPending}>
        Add
      </button>
    </form>
  );
}
```

### Multiple Optimistic Updates

```javascript
function useMultipleOptimisticUpdates() {
  const queryClient = useQueryClient();

  const likeMutation = useMutation({
    mutationFn: (postId) =>
      fetch(`/api/posts/${postId}/like`, { method: 'POST' }),
    
    onMutate: async (postId) => {
      await queryClient.cancelQueries({ queryKey: ['posts'] });
      const previous = queryClient.getQueryData(['posts']);

      queryClient.setQueryData(['posts'], (old) =>
        old.map(post =>
          post.id === postId
            ? { ...post, likes: post.likes + 1, liked: true }
            : post
        )
      );

      return { previous };
    },

    onError: (err, postId, context) => {
      queryClient.setQueryData(['posts'], context.previous);
    },
  });

  const commentMutation = useMutation({
    mutationFn: ({ postId, comment }) =>
      fetch(`/api/posts/${postId}/comments`, {
        method: 'POST',
        body: JSON.stringify({ comment }),
      }),
    
    onMutate: async ({ postId, comment }) => {
      await queryClient.cancelQueries({ queryKey: ['posts'] });
      const previous = queryClient.getQueryData(['posts']);

      const optimisticComment = {
        id: Date.now(),
        text: comment,
        pending: true,
      };

      queryClient.setQueryData(['posts'], (old) =>
        old.map(post =>
          post.id === postId
            ? {
                ...post,
                comments: [...post.comments, optimisticComment],
              }
            : post
        )
      );

      return { previous };
    },

    onError: (err, variables, context) => {
      queryClient.setQueryData(['posts'], context.previous);
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  return { likeMutation, commentMutation };
}
```

## Real-World Examples

### Social Media Like/Unlike

```javascript
function LikeButton({ post }) {
  const [likes, setLikes] = useState(post.likes);
  const [isLiked, setIsLiked] = useState(post.isLiked);
  const [isProcessing, setIsProcessing] = useState(false);

  const toggleLike = async () => {
    if (isProcessing) return;

    setIsProcessing(true);
    const previousLikes = likes;
    const previousIsLiked = isLiked;

    // Optimistic update
    setLikes(isLiked ? likes - 1 : likes + 1);
    setIsLiked(!isLiked);

    try {
      await fetch(`/api/posts/${post.id}/like`, {
        method: isLiked ? 'DELETE' : 'POST',
      });
      setIsProcessing(false);
    } catch (error) {
      // Rollback
      setLikes(previousLikes);
      setIsLiked(previousIsLiked);
      setIsProcessing(false);
      toast.error('Failed to update like');
    }
  };

  return (
    <button
      onClick={toggleLike}
      className={isLiked ? 'liked' : ''}
      disabled={isProcessing}
    >
      <Heart fill={isLiked ? 'red' : 'none'} />
      {likes}
    </button>
  );
}
```

### Collaborative Todo List

```javascript
function CollaborativeTodoList() {
  const [todos, setTodos] = useState([]);
  const [pendingUpdates, setPendingUpdates] = useState(new Set());

  const addTodo = async (title) => {
    const tempId = `temp-${Date.now()}`;
    const optimisticTodo = {
      id: tempId,
      title,
      completed: false,
      pending: true,
    };

    setTodos(prev => [...prev, optimisticTodo]);
    setPendingUpdates(prev => new Set(prev).add(tempId));

    try {
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ title }),
      });
      const savedTodo = await response.json();

      setTodos(prev =>
        prev.map(t => (t.id === tempId ? { ...savedTodo, pending: false } : t))
      );
      setPendingUpdates(prev => {
        const next = new Set(prev);
        next.delete(tempId);
        return next;
      });
    } catch (error) {
      setTodos(prev => prev.filter(t => t.id !== tempId));
      setPendingUpdates(prev => {
        const next = new Set(prev);
        next.delete(tempId);
        return next;
      });
      toast.error('Failed to add todo');
    }
  };

  const toggleTodo = async (id) => {
    const todo = todos.find(t => t.id === id);
    if (!todo) return;

    setTodos(prev =>
      prev.map(t =>
        t.id === id ? { ...t, completed: !t.completed } : t
      )
    );
    setPendingUpdates(prev => new Set(prev).add(id));

    try {
      await fetch(`/api/todos/${id}`, {
        method: 'PATCH',
        body: JSON.stringify({ completed: !todo.completed }),
      });
      setPendingUpdates(prev => {
        const next = new Set(prev);
        next.delete(id);
        return next;
      });
    } catch (error) {
      setTodos(prev =>
        prev.map(t =>
          t.id === id ? { ...t, completed: todo.completed } : t
        )
      );
      setPendingUpdates(prev => {
        const next = new Set(prev);
        next.delete(id);
        return next;
      });
      toast.error('Failed to update todo');
    }
  };

  return (
    <ul>
      {todos.map(todo => (
        <li
          key={todo.id}
          className={pendingUpdates.has(todo.id) ? 'pending' : ''}
        >
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.title}
          {todo.pending && <Spinner size="small" />}
        </li>
      ))}
    </ul>
  );
}
```

## Best Practices

### 1. Always Have Rollback Logic

```javascript
// Good: Proper rollback
const handleUpdate = async () => {
  const previous = state;
  setState(newState);
  
  try {
    await updateServer(newState);
  } catch (error) {
    setState(previous); // Rollback
    showError(error);
  }
};

// Bad: No rollback
const handleUpdate = async () => {
  setState(newState);
  await updateServer(newState); // What if this fails?
};
```

### 2. Show Pending State

```javascript
// Good: Visual feedback for pending
<TodoItem
  todo={todo}
  className={todo.pending ? 'pending opacity-50' : ''}
/>

// Include pending indicator
{todo.pending && <Spinner />}
```

### 3. Handle Network Errors Gracefully

```javascript
try {
  await mutation();
} catch (error) {
  if (error.message === 'Network request failed') {
    toast.error('No internet connection. Changes will sync when online.');
    // Queue for later retry
  } else {
    toast.error('Update failed. Please try again.');
    // Rollback immediately
  }
}
```

### 4. Prevent Double Submissions

```javascript
const [isProcessing, setIsProcessing] = useState(false);

const handleSubmit = async () => {
  if (isProcessing) return; // Prevent double submit
  
  setIsProcessing(true);
  try {
    // ... optimistic update
  } finally {
    setIsProcessing(false);
  }
};
```

## Common Mistakes

### 1. Not Saving Previous State

```javascript
// Bad: Can't rollback
const handleUpdate = () => {
  setState(newState);
  updateServer(newState).catch(() => {
    // Can't rollback - previous state lost!
  });
};

// Good: Save for rollback
const handleUpdate = () => {
  const previous = state;
  setState(newState);
  updateServer(newState).catch(() => {
    setState(previous);
  });
};
```

### 2. Forgetting to Clear Pending State

```javascript
// Bad: Pending state never clears
const addItem = () => {
  setItems([...items, { ...newItem, pending: true }]);
  saveItem(newItem); // pending never cleared
};

// Good: Clear pending on success
const addItem = async () => {
  const tempId = Date.now();
  setItems([...items, { ...newItem, id: tempId, pending: true }]);
  
  try {
    const saved = await saveItem(newItem);
    setItems(items => 
      items.map(i => i.id === tempId ? { ...saved, pending: false } : i)
    );
  } catch (error) {
    setItems(items => items.filter(i => i.id !== tempId));
  }
};
```

### 3. Not Handling Race Conditions

```javascript
// Bad: Race condition possible
const handleClick = async () => {
  setState('loading');
  await update();
  setState('success'); // May be stale
};

// Good: Use version/request ID
const handleClick = async () => {
  const requestId = ++requestIdRef.current;
  setState('loading');
  await update();
  if (requestId === requestIdRef.current) {
    setState('success');
  }
};
```

## Key Takeaways

1. **Immediate Feedback**: Update UI instantly for better UX
2. **Always Rollback**: Handle errors by reverting to previous state
3. **Show Pending State**: Indicate when operations are in flight
4. **Prevent Race Conditions**: Track request IDs or use cancellation
5. **Queue Offline Updates**: Store failed updates for retry
6. **Use Libraries**: React Query handles most complexity
7. **Test Error Cases**: Verify rollback logic works correctly
8. **Debounce Rapid Changes**: Avoid excessive server requests

## Additional Resources

- [React useOptimistic Hook](https://react.dev/reference/react/useOptimistic)
- [TanStack Query Optimistic Updates](https://tanstack.com/query/latest/docs/react/guides/optimistic-updates)
- [Optimistic UI Patterns](https://www.smashingmagazine.com/2016/11/true-lies-of-optimistic-user-interfaces/)
- [SWR Optimistic UI](https://swr.vercel.app/docs/mutation#optimistic-updates)
