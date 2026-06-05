# React Reconciliation and Keys

## The Idea

**In plain English:** React reconciliation is how React figures out the minimum number of changes needed to update what you see on screen when something changes — instead of wiping everything and redrawing from scratch, it compares the old screen with the new one and only fixes the differences. A "key" is a special label you put on each item in a list so React can tell which item is which, even after the list gets reordered or items get removed.

**Real-world analogy:** Imagine a restaurant that posts its daily specials on a corkboard using index cards, one card per dish. At the end of the day, the manager compares today's menu to yesterday's and only swaps out the cards that actually changed — instead of clearing the whole board and repinning every card from scratch. If each card has the dish's unique recipe number written on it, the manager can instantly find "Pasta Carbonara" even if someone moved it to a different spot on the board.

- The corkboard = the browser DOM (what users actually see)
- Each index card = a React element in the list
- The act of comparing old cards to new cards = reconciliation
- The recipe number on each card = the `key` prop
- Swapping only the changed cards = the minimal DOM update React performs

---

## Overview

React's reconciliation algorithm is the process by which React updates the DOM efficiently. Understanding how reconciliation works, especially the role of the `key` prop, is fundamental to writing performant React applications. This guide explores the reconciliation process, why keys matter, and how to use them correctly.

## Table of Contents

1. [The Reconciliation Algorithm](#the-reconciliation-algorithm)
2. [Virtual DOM Diffing](#virtual-dom-diffing)
3. [The Key Prop](#the-key-prop)
4. [List Rendering Optimization](#list-rendering-optimization)
5. [Common Key Mistakes](#common-key-mistakes)
6. [Best Practices](#best-practices)
7. [Real-World Use Cases](#real-world-use-cases)
8. [Key Takeaways](#key-takeaways)
9. [Resources](#resources)

## The Reconciliation Algorithm

React uses a heuristic O(n) algorithm to determine how to update the DOM efficiently. The algorithm makes assumptions to optimize performance:

### How Reconciliation Works

```typescript
// When state changes, React creates a new virtual DOM tree
import React, { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  // On every render, React creates a new tree
  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// React compares:
// Old tree: <h1>Count: 0</h1>
// New tree: <h1>Count: 1</h1>
// Result: Updates only the text node
```

### Element Type Comparison

```typescript
// Different element types cause full remount
function App({ isLoggedIn }: { isLoggedIn: boolean }) {
  if (isLoggedIn) {
    // This creates a <div> element
    return (
      <div>
        <UserProfile />
      </div>
    );
  }
  
  // This creates a <section> element - DIFFERENT TYPE
  // React will unmount <div> and mount <section>
  // UserProfile will be destroyed and recreated
  return (
    <section>
      <UserProfile />
    </section>
  );
}

// Better approach - keep same element type
function AppOptimized({ isLoggedIn }: { isLoggedIn: boolean }) {
  return (
    <div className={isLoggedIn ? 'logged-in' : 'logged-out'}>
      <UserProfile />
    </div>
  );
}
```

### Component Type Comparison

```typescript
import React, { useState, useEffect } from 'react';

function Input({ value }: { value: string }) {
  useEffect(() => {
    console.log('Input mounted');
    return () => console.log('Input unmounted');
  }, []);
  
  return <input defaultValue={value} />;
}

function TextArea({ value }: { value: string }) {
  useEffect(() => {
    console.log('TextArea mounted');
    return () => console.log('TextArea unmounted');
  }, []);
  
  return <textarea defaultValue={value} />;
}

function Form() {
  const [isMultiline, setIsMultiline] = useState(false);
  
  // Different component types cause unmount/mount
  return (
    <div>
      <label>
        <input
          type="checkbox"
          checked={isMultiline}
          onChange={(e) => setIsMultiline(e.target.checked)}
        />
        Multiline
      </label>
      
      {isMultiline ? (
        <TextArea value="Hello" /> // TextArea unmounts/mounts on toggle
      ) : (
        <Input value="Hello" /> // Input unmounts/mounts on toggle
      )}
    </div>
  );
}
```

## Virtual DOM Diffing

React's diffing algorithm compares trees element by element:

### Shallow Comparison

```typescript
import React, { useState } from 'react';

function User({ name, age }: { name: string; age: number }) {
  console.log('User rendered');
  
  return (
    <div>
      <h2>{name}</h2>
      <p>Age: {age}</p>
    </div>
  );
}

function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      {/* User re-renders even though props don't change */}
      <User name="John" age={30} />
    </div>
  );
}

// Optimized with React.memo
const UserOptimized = React.memo(function User({ 
  name, 
  age 
}: { 
  name: string; 
  age: number 
}) {
  console.log('User rendered');
  
  return (
    <div>
      <h2>{name}</h2>
      <p>Age: {age}</p>
    </div>
  );
});
```

### Attribute Diffing

```typescript
import React, { useState } from 'react';

function StyledDiv() {
  const [isActive, setIsActive] = useState(false);
  
  // React only updates the className attribute
  return (
    <div 
      className={isActive ? 'active' : 'inactive'}
      style={{ padding: '20px' }}
      data-testid="styled-div"
    >
      <button onClick={() => setIsActive(!isActive)}>
        Toggle
      </button>
    </div>
  );
}

// On toggle, React compares:
// Old: className="inactive"
// New: className="active"
// Result: Only className attribute is updated
```

## The Key Prop

The `key` prop helps React identify which items have changed, been added, or removed:

### Why Keys Matter

```typescript
import React, { useState } from 'react';

interface Todo {
  id: number;
  text: string;
}

// WITHOUT keys - BAD
function TodoListBad() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Learn React' },
    { id: 2, text: 'Build app' },
  ]);
  
  const removeTodo = (id: number) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };
  
  return (
    <ul>
      {todos.map(todo => (
        // No key - React uses index implicitly
        <li>
          <input type="checkbox" />
          {todo.text}
          <button onClick={() => removeTodo(todo.id)}>Remove</button>
        </li>
      ))}
    </ul>
  );
  // Problem: Removing first item causes checkbox state to be wrong
}

// WITH keys - GOOD
function TodoListGood() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Learn React' },
    { id: 2, text: 'Build app' },
  ]);
  
  const removeTodo = (id: number) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input type="checkbox" />
          {todo.text}
          <button onClick={() => removeTodo(todo.id)}>Remove</button>
        </li>
      ))}
    </ul>
  );
  // With key, React tracks each item correctly
}
```

### How Keys Work

```typescript
import React, { useState } from 'react';

interface Item {
  id: string;
  name: string;
}

function ItemList() {
  const [items, setItems] = useState<Item[]>([
    { id: 'a', name: 'Apple' },
    { id: 'b', name: 'Banana' },
    { id: 'c', name: 'Cherry' },
  ]);
  
  // When we reverse the array
  const reverse = () => {
    setItems([...items].reverse());
  };
  
  return (
    <div>
      <button onClick={reverse}>Reverse</button>
      <ul>
        {items.map(item => (
          <li key={item.id}>
            {item.name}
            <ExpensiveComponent id={item.id} />
          </li>
        ))}
      </ul>
    </div>
  );
}

// With keys:
// Old: [a, b, c]
// New: [c, b, a]
// React knows 'a' and 'c' swapped positions - reorders DOM
// ExpensiveComponent instances are preserved

// Without keys (using index):
// React thinks all items changed
// All ExpensiveComponent instances are recreated
```

### Keys and Component State

```typescript
import React, { useState } from 'react';

function Counter({ id }: { id: string }) {
  const [count, setCount] = useState(0);
  
  console.log(`Counter ${id} rendered, count: ${count}`);
  
  return (
    <div>
      <p>Counter {id}: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

function CounterList() {
  const [counters, setCounters] = useState(['a', 'b', 'c']);
  
  const shuffle = () => {
    setCounters([...counters].sort(() => Math.random() - 0.5));
  };
  
  return (
    <div>
      <button onClick={shuffle}>Shuffle</button>
      {counters.map(id => (
        <Counter key={id} id={id} />
      ))}
    </div>
  );
}

// With proper keys, each Counter maintains its own state
// even when shuffled
```

## List Rendering Optimization

### Stable Keys

```typescript
import React, { useState } from 'react';

interface User {
  id: number;
  name: string;
  email: string;
}

function UserList({ users }: { users: User[] }) {
  // GOOD: Use stable, unique identifiers
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          {user.name} - {user.email}
        </li>
      ))}
    </ul>
  );
}

// BAD: Using index as key
function UserListBadIndex({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user, index) => (
        <li key={index}> {/* Don't do this! */}
          {user.name} - {user.email}
        </li>
      ))}
    </ul>
  );
}

// BAD: Using generated keys
function UserListBadGenerated({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map(user => (
        <li key={Math.random()}> {/* Don't do this! */}
          {user.name} - {user.email}
        </li>
      ))}
    </ul>
  );
}
```

### Composite Keys

```typescript
import React from 'react';

interface Post {
  userId: number;
  postId: number;
  title: string;
}

function PostList({ posts }: { posts: Post[] }) {
  // When you need composite keys
  return (
    <ul>
      {posts.map(post => (
        <li key={`${post.userId}-${post.postId}`}>
          {post.title}
        </li>
      ))}
    </ul>
  );
}

// Better: Create unique ID in data
interface PostWithId extends Post {
  id: string;
}

function transformPosts(posts: Post[]): PostWithId[] {
  return posts.map(post => ({
    ...post,
    id: `${post.userId}-${post.postId}`,
  }));
}

function PostListBetter({ posts }: { posts: Post[] }) {
  const postsWithIds = transformPosts(posts);
  
  return (
    <ul>
      {postsWithIds.map(post => (
        <li key={post.id}>
          {post.title}
        </li>
      ))}
    </ul>
  );
}
```

### Dynamic Lists

```typescript
import React, { useState } from 'react';

interface Task {
  id: string;
  text: string;
  completed: boolean;
}

function TaskList() {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [nextId, setNextId] = useState(1);
  
  const addTask = (text: string) => {
    const newTask: Task = {
      id: `task-${nextId}`,
      text,
      completed: false,
    };
    setTasks([...tasks, newTask]);
    setNextId(nextId + 1);
  };
  
  const removeTask = (id: string) => {
    setTasks(tasks.filter(task => task.id !== id));
  };
  
  const toggleTask = (id: string) => {
    setTasks(tasks.map(task =>
      task.id === id ? { ...task, completed: !task.completed } : task
    ));
  };
  
  return (
    <div>
      <button onClick={() => addTask(`Task ${nextId}`)}>
        Add Task
      </button>
      <ul>
        {tasks.map(task => (
          <li key={task.id}>
            <input
              type="checkbox"
              checked={task.completed}
              onChange={() => toggleTask(task.id)}
            />
            <span style={{ 
              textDecoration: task.completed ? 'line-through' : 'none' 
            }}>
              {task.text}
            </span>
            <button onClick={() => removeTask(task.id)}>Remove</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Nested Lists

```typescript
import React from 'react';

interface Comment {
  id: string;
  text: string;
  replies?: Comment[];
}

function CommentThread({ comments }: { comments: Comment[] }) {
  return (
    <ul>
      {comments.map(comment => (
        <li key={comment.id}>
          <Comment comment={comment} />
        </li>
      ))}
    </ul>
  );
}

function Comment({ comment }: { comment: Comment }) {
  return (
    <div>
      <p>{comment.text}</p>
      {comment.replies && comment.replies.length > 0 && (
        <div style={{ marginLeft: '20px' }}>
          {/* Nested list also needs keys */}
          {comment.replies.map(reply => (
            <div key={reply.id}>
              <Comment comment={reply} />
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

## Common Key Mistakes

### Using Index as Key

```typescript
import React, { useState } from 'react';

interface Item {
  text: string;
}

// BAD: Index as key with reordering
function BadExample() {
  const [items, setItems] = useState<Item[]>([
    { text: 'Item 1' },
    { text: 'Item 2' },
    { text: 'Item 3' },
  ]);
  
  const removeFirst = () => {
    setItems(items.slice(1));
  };
  
  return (
    <div>
      <button onClick={removeFirst}>Remove First</button>
      {items.map((item, index) => (
        <div key={index}> {/* BAD */}
          <input defaultValue={item.text} />
        </div>
      ))}
    </div>
  );
}
// Problem: After removal, indexes shift, causing wrong inputs to match wrong data

// GOOD: Stable unique keys
function GoodExample() {
  const [items, setItems] = useState([
    { id: '1', text: 'Item 1' },
    { id: '2', text: 'Item 2' },
    { id: '3', text: 'Item 3' },
  ]);
  
  const removeFirst = () => {
    setItems(items.slice(1));
  };
  
  return (
    <div>
      <button onClick={removeFirst}>Remove First</button>
      {items.map(item => (
        <div key={item.id}> {/* GOOD */}
          <input defaultValue={item.text} />
        </div>
      ))}
    </div>
  );
}
```

### When Index Is Acceptable

```typescript
import React from 'react';

// OK: Static list that never changes
function StaticList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
}

// OK: List with no state and no reordering
function ReadOnlyList({ data }: { data: readonly string[] }) {
  return (
    <div>
      {data.map((item, index) => (
        <p key={index}>{item}</p>
      ))}
    </div>
  );
}

// NOT OK: List with inputs or state
function InteractiveList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, index) => (
        // BAD: Input state can get mixed up
        <li key={index}>
          <input defaultValue={item} />
        </li>
      ))}
    </ul>
  );
}
```

### Non-Unique Keys

```typescript
import React from 'react';

interface Product {
  category: string;
  name: string;
  id: string;
}

// BAD: Non-unique keys
function ProductListBad({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map(product => (
        // BAD: category might not be unique
        <li key={product.category}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}

// GOOD: Unique keys
function ProductListGood({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.category}: {product.name}
        </li>
      ))}
    </ul>
  );
}
```

### Changing Keys

```typescript
import React, { useState } from 'react';

// BAD: Keys that change between renders
function BadCounter() {
  const [items] = useState(['a', 'b', 'c']);
  
  return (
    <div>
      {items.map(item => (
        // BAD: Math.random() creates new key every render
        <div key={Math.random()}>
          <ExpensiveComponent item={item} />
        </div>
      ))}
    </div>
  );
}

// GOOD: Stable keys
function GoodCounter() {
  const [items] = useState(['a', 'b', 'c']);
  
  return (
    <div>
      {items.map(item => (
        <div key={item}>
          <ExpensiveComponent item={item} />
        </div>
      ))}
    </div>
  );
}

function ExpensiveComponent({ item }: { item: string }) {
  // Expensive computation
  return <div>{item}</div>;
}
```

## Best Practices

### Use Stable IDs from Backend

```typescript
import React, { useEffect, useState } from 'react';

interface User {
  id: number; // From database
  name: string;
  email: string;
}

function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  
  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then((data: User[]) => setUsers(data));
  }, []);
  
  return (
    <ul>
      {users.map(user => (
        // Use database ID as key
        <li key={user.id}>
          {user.name} ({user.email})
        </li>
      ))}
    </ul>
  );
}
```

### Generate IDs for Client-Side Data

```typescript
import React, { useState } from 'react';
import { nanoid } from 'nanoid';

interface Note {
  id: string;
  text: string;
  createdAt: Date;
}

function NoteApp() {
  const [notes, setNotes] = useState<Note[]>([]);
  
  const addNote = (text: string) => {
    const newNote: Note = {
      id: nanoid(), // Generate unique ID
      text,
      createdAt: new Date(),
    };
    setNotes([...notes, newNote]);
  };
  
  return (
    <div>
      <button onClick={() => addNote('New note')}>Add Note</button>
      <ul>
        {notes.map(note => (
          <li key={note.id}>
            {note.text} - {note.createdAt.toLocaleString()}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Keys in Fragments

```typescript
import React from 'react';

interface Section {
  id: string;
  title: string;
  content: string;
}

function Sections({ sections }: { sections: Section[] }) {
  return (
    <div>
      {sections.map(section => (
        // Fragment can have key prop
        <React.Fragment key={section.id}>
          <h2>{section.title}</h2>
          <p>{section.content}</p>
          <hr />
        </React.Fragment>
      ))}
    </div>
  );
}

// Short syntax doesn't support keys
function SectionsBad({ sections }: { sections: Section[] }) {
  return (
    <div>
      {sections.map(section => (
        // Can't use key with <>
        <> 
          <h2>{section.title}</h2>
          <p>{section.content}</p>
        </>
      ))}
    </div>
  );
}
```

### Resetting Component State with Keys

```typescript
import React, { useState } from 'react';

function Form({ userId }: { userId: string }) {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  
  return (
    <form>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Name"
      />
      <input
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
    </form>
  );
}

function UserEditor({ userId }: { userId: string }) {
  // Use key to reset form state when userId changes
  return <Form key={userId} userId={userId} />;
}

// When userId changes, React unmounts old Form and mounts new Form
// This resets all internal state automatically
```

## Real-World Use Cases

### Infinite Scroll List

```typescript
import React, { useState, useEffect, useCallback } from 'react';

interface Post {
  id: number;
  title: string;
  body: string;
}

function InfinitePostList() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  
  const loadMore = useCallback(async () => {
    setLoading(true);
    const response = await fetch(
      `https://jsonplaceholder.typicode.com/posts?_page=${page}&_limit=10`
    );
    const newPosts: Post[] = await response.json();
    setPosts(prev => [...prev, ...newPosts]);
    setPage(prev => prev + 1);
    setLoading(false);
  }, [page]);
  
  useEffect(() => {
    loadMore();
  }, []); // Load initial posts
  
  return (
    <div>
      <ul>
        {posts.map(post => (
          <li key={post.id}> {/* Stable ID from API */}
            <h3>{post.title}</h3>
            <p>{post.body}</p>
          </li>
        ))}
      </ul>
      <button onClick={loadMore} disabled={loading}>
        {loading ? 'Loading...' : 'Load More'}
      </button>
    </div>
  );
}
```

### Sortable Table

```typescript
import React, { useState } from 'react';

interface User {
  id: number;
  name: string;
  age: number;
  email: string;
}

type SortKey = 'name' | 'age' | 'email';

function SortableTable({ users }: { users: User[] }) {
  const [sortKey, setSortKey] = useState<SortKey>('name');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('asc');
  
  const sortedUsers = [...users].sort((a, b) => {
    const aVal = a[sortKey];
    const bVal = b[sortKey];
    
    if (aVal < bVal) return sortOrder === 'asc' ? -1 : 1;
    if (aVal > bVal) return sortOrder === 'asc' ? 1 : -1;
    return 0;
  });
  
  const handleSort = (key: SortKey) => {
    if (sortKey === key) {
      setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc');
    } else {
      setSortKey(key);
      setSortOrder('asc');
    }
  };
  
  return (
    <table>
      <thead>
        <tr>
          <th onClick={() => handleSort('name')}>Name</th>
          <th onClick={() => handleSort('age')}>Age</th>
          <th onClick={() => handleSort('email')}>Email</th>
        </tr>
      </thead>
      <tbody>
        {sortedUsers.map(user => (
          <tr key={user.id}> {/* Key ensures row identity preserved */}
            <td>{user.name}</td>
            <td>{user.age}</td>
            <td>{user.email}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

### Drag and Drop List

```typescript
import React, { useState } from 'react';

interface Item {
  id: string;
  text: string;
}

function DraggableList() {
  const [items, setItems] = useState<Item[]>([
    { id: '1', text: 'Item 1' },
    { id: '2', text: 'Item 2' },
    { id: '3', text: 'Item 3' },
  ]);
  const [draggedId, setDraggedId] = useState<string | null>(null);
  
  const handleDragStart = (id: string) => {
    setDraggedId(id);
  };
  
  const handleDragOver = (e: React.DragEvent, id: string) => {
    e.preventDefault();
    
    if (draggedId === null || draggedId === id) return;
    
    const draggedIndex = items.findIndex(item => item.id === draggedId);
    const targetIndex = items.findIndex(item => item.id === id);
    
    if (draggedIndex === -1 || targetIndex === -1) return;
    
    const newItems = [...items];
    const [removed] = newItems.splice(draggedIndex, 1);
    newItems.splice(targetIndex, 0, removed);
    
    setItems(newItems);
  };
  
  const handleDragEnd = () => {
    setDraggedId(null);
  };
  
  return (
    <ul>
      {items.map(item => (
        <li
          key={item.id} // Critical for maintaining item identity during drag
          draggable
          onDragStart={() => handleDragStart(item.id)}
          onDragOver={(e) => handleDragOver(e, item.id)}
          onDragEnd={handleDragEnd}
          style={{
            opacity: draggedId === item.id ? 0.5 : 1,
            cursor: 'move',
          }}
        >
          {item.text}
        </li>
      ))}
    </ul>
  );
}
```

## Key Takeaways

1. **Reconciliation is O(n)**: React's algorithm makes assumptions to achieve linear time complexity
2. **Different element types remount**: Changing from `<div>` to `<section>` destroys and recreates the component tree
3. **Keys identify elements**: Keys help React match old and new elements efficiently
4. **Use stable, unique keys**: IDs from databases or generated UUIDs work best
5. **Index as key is dangerous**: Only use index when list is static and never reordered
6. **Keys preserve component state**: Proper keys maintain component state across reorders
7. **Keys can force remounts**: Changing key forces component to unmount and remount
8. **Non-unique keys cause bugs**: Duplicate keys lead to unpredictable behavior
9. **Fragment keys**: Use `<React.Fragment key={}>` when mapping multiple elements
10. **Performance impact**: Wrong keys can cause unnecessary unmounts and remounts

## Resources

- [React Reconciliation Docs](https://react.dev/learn/preserving-and-resetting-state)
- [Keys in React Lists](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key)
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
- [Index as Key is an Anti-Pattern](https://robinpokorny.medium.com/index-as-a-key-is-an-anti-pattern-e0349aece318)
- [Understanding React's Key Prop](https://kentcdodds.com/blog/understanding-reacts-key-prop)
- [React Performance Optimization](https://react.dev/learn/render-and-commit)
