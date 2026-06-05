# React 19 Hooks: useOptimistic, useFormStatus, useActionState, use()

## The Idea

**In plain English:** React 19 hooks are special built-in tools that let your app update the screen instantly when you do something (like clicking a button), even before the internet has finished saving the change. A "hook" in React is a function whose name starts with "use" that gives your component superpowers like remembering data or running side-effects.

**Real-world analogy:** Imagine you order food at a restaurant using a touch screen. The screen instantly shows your order in the queue even though the kitchen hasn't confirmed it yet. If the kitchen is out of that dish, the item disappears from your queue. A separate status light on the screen shows "sending..." while the order travels to the kitchen, and a results panel shows any error message like "sorry, sold out."

- The touch screen showing your order immediately = `useOptimistic` (instant UI update before the server confirms)
- The "sending..." status light = `useFormStatus` (tracks whether the form is still being submitted)
- The results panel showing success or error from the kitchen = `useActionState` (holds the response from the server action)

---

## Table of Contents

- [Introduction](#introduction)
- [useOptimistic Hook](#useoptimistic-hook)
- [useFormStatus Hook](#useformstatus-hook)
- [useActionState Hook](#useactionstate-hook)
- [use() Hook](#use-hook)
- [Combining Hooks](#combining-hooks)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

React 19 introduces several new hooks designed to improve user experience with optimistic updates, form handling, and data fetching. These hooks integrate seamlessly with Server Actions and Suspense to create more responsive applications.

## useOptimistic Hook

`useOptimistic` allows you to show optimistic state while an async action is in progress, automatically reverting if the action fails.

### Basic Usage

```typescript
'use client';

import { useOptimistic } from 'react';

interface Message {
  id: string;
  text: string;
  sending?: boolean;
}

export function MessageThread({ messages }: { messages: Message[] }) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newMessage: string) => [
      ...state,
      { id: 'temp-' + Date.now(), text: newMessage, sending: true }
    ]
  );

  async function sendMessage(formData: FormData) {
    const text = formData.get('message') as string;
    
    // Immediately show optimistic message
    addOptimisticMessage(text);
    
    // Send to server
    await submitMessage(text);
    // Optimistic message automatically replaced with real one
  }

  return (
    <>
      <ul>
        {optimisticMessages.map(message => (
          <li key={message.id} style={{ opacity: message.sending ? 0.5 : 1 }}>
            {message.text}
            {message.sending && ' (Sending...)'}
          </li>
        ))}
      </ul>
      
      <form action={sendMessage}>
        <input name="message" placeholder="Type message..." />
        <button>Send</button>
      </form>
    </>
  );
}
```

### Like Button with Optimistic Update

```typescript
'use client';

import { useOptimistic } from 'react';
import { likePost } from './actions';

interface Post {
  id: string;
  title: string;
  likes: number;
  liked: boolean;
}

export function Post({ post }: { post: Post }) {
  const [optimisticPost, updateOptimisticPost] = useOptimistic(
    post,
    (state, liked: boolean) => ({
      ...state,
      liked,
      likes: state.likes + (liked ? 1 : -1)
    })
  );

  async function handleLike() {
    // Update UI immediately
    updateOptimisticPost(!optimisticPost.liked);
    
    // Send to server
    await likePost(post.id, !optimisticPost.liked);
    // If it fails, optimistic update reverts automatically
  }

  return (
    <article>
      <h2>{optimisticPost.title}</h2>
      <button onClick={handleLike}>
        {optimisticPost.liked ? '❤️' : '🤍'} {optimisticPost.likes}
      </button>
    </article>
  );
}
```

### Todo List with Optimistic Updates

```typescript
'use client';

import { useOptimistic } from 'react';
import { addTodo, toggleTodo, deleteTodo } from './actions';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  pending?: boolean;
}

export function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [optimisticTodos, updateOptimisticTodos] = useOptimistic(
    initialTodos,
    (state, action: TodoAction) => {
      switch (action.type) {
        case 'add':
          return [
            ...state,
            { id: 'temp', text: action.text, completed: false, pending: true }
          ];
        case 'toggle':
          return state.map(todo =>
            todo.id === action.id
              ? { ...todo, completed: !todo.completed, pending: true }
              : todo
          );
        case 'delete':
          return state.filter(todo => todo.id !== action.id);
        default:
          return state;
      }
    }
  );

  async function handleAdd(formData: FormData) {
    const text = formData.get('text') as string;
    updateOptimisticTodos({ type: 'add', text });
    await addTodo(text);
  }

  async function handleToggle(id: string) {
    updateOptimisticTodos({ type: 'toggle', id });
    await toggleTodo(id);
  }

  async function handleDelete(id: string) {
    updateOptimisticTodos({ type: 'delete', id });
    await deleteTodo(id);
  }

  return (
    <>
      <form action={handleAdd}>
        <input name="text" placeholder="New todo..." />
        <button>Add</button>
      </form>

      <ul>
        {optimisticTodos.map(todo => (
          <li
            key={todo.id}
            style={{
              opacity: todo.pending ? 0.5 : 1,
              textDecoration: todo.completed ? 'line-through' : 'none'
            }}
          >
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => handleToggle(todo.id)}
            />
            {todo.text}
            <button onClick={() => handleDelete(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </>
  );
}

type TodoAction =
  | { type: 'add'; text: string }
  | { type: 'toggle'; id: string }
  | { type: 'delete'; id: string };
```

### Complex State Updates

```typescript
'use client';

import { useOptimistic, useState } from 'react';

interface CartItem {
  productId: string;
  name: string;
  quantity: number;
  price: number;
}

export function ShoppingCart({ items }: { items: CartItem[] }) {
  const [optimisticCart, updateOptimisticCart] = useOptimistic(
    items,
    (state, action: CartAction) => {
      switch (action.type) {
        case 'add':
          const existing = state.find(item => item.productId === action.item.productId);
          if (existing) {
            return state.map(item =>
              item.productId === action.item.productId
                ? { ...item, quantity: item.quantity + action.item.quantity }
                : item
            );
          }
          return [...state, action.item];
        
        case 'remove':
          return state.filter(item => item.productId !== action.productId);
        
        case 'update-quantity':
          return state.map(item =>
            item.productId === action.productId
              ? { ...item, quantity: action.quantity }
              : item
          );
        
        default:
          return state;
      }
    }
  );

  const total = optimisticCart.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  async function addToCart(item: CartItem) {
    updateOptimisticCart({ type: 'add', item });
    await submitCartUpdate({ type: 'add', item });
  }

  async function removeFromCart(productId: string) {
    updateOptimisticCart({ type: 'remove', productId });
    await submitCartUpdate({ type: 'remove', productId });
  }

  async function updateQuantity(productId: string, quantity: number) {
    updateOptimisticCart({ type: 'update-quantity', productId, quantity });
    await submitCartUpdate({ type: 'update-quantity', productId, quantity });
  }

  return (
    <div className="cart">
      <h2>Shopping Cart</h2>
      
      {optimisticCart.map(item => (
        <div key={item.productId} className="cart-item">
          <span>{item.name}</span>
          <input
            type="number"
            value={item.quantity}
            onChange={(e) => updateQuantity(item.productId, parseInt(e.target.value))}
            min="1"
          />
          <span>${(item.price * item.quantity).toFixed(2)}</span>
          <button onClick={() => removeFromCart(item.productId)}>Remove</button>
        </div>
      ))}
      
      <div className="cart-total">
        Total: ${total.toFixed(2)}
      </div>
    </div>
  );
}

type CartAction =
  | { type: 'add'; item: CartItem }
  | { type: 'remove'; productId: string }
  | { type: 'update-quantity'; productId: string; quantity: number };
```

## useFormStatus Hook

`useFormStatus` provides information about the last form submission status. Must be used within a form component.

### Basic Form Status

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

export function ContactForm({ action }: { action: (formData: FormData) => void }) {
  return (
    <form action={action}>
      <input name="name" required />
      <input name="email" type="email" required />
      <textarea name="message" required />
      <SubmitButton />
    </form>
  );
}
```

### All Status Properties

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function FormDebugger() {
  const { pending, data, method, action } = useFormStatus();

  if (!pending) return null;

  return (
    <div className="form-status">
      <p>Form Status:</p>
      <ul>
        <li>Pending: {pending.toString()}</li>
        <li>Method: {method}</li>
        <li>Action: {action}</li>
        <li>
          Data:
          <pre>{data && JSON.stringify(Object.fromEntries(data), null, 2)}</pre>
        </li>
      </ul>
    </div>
  );
}

export function DiagnosticForm({ action }: { action: (formData: FormData) => void }) {
  return (
    <form action={action}>
      <input name="username" />
      <input name="password" type="password" />
      <FormDebugger />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Conditional UI Based on Status

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function FormControls() {
  const { pending } = useFormStatus();

  return (
    <div className="form-controls">
      {/* Disable all inputs while pending */}
      <input name="title" disabled={pending} placeholder="Title" />
      <textarea name="content" disabled={pending} placeholder="Content" />
      <select name="category" disabled={pending}>
        <option>Tech</option>
        <option>Business</option>
        <option>Lifestyle</option>
      </select>

      {/* Show loading overlay */}
      {pending && (
        <div className="loading-overlay">
          <Spinner />
          <p>Saving your post...</p>
        </div>
      )}

      {/* Conditional buttons */}
      {pending ? (
        <button type="button" disabled>
          <Spinner size="small" /> Saving...
        </button>
      ) : (
        <>
          <button type="submit" name="action" value="draft">
            Save Draft
          </button>
          <button type="submit" name="action" value="publish">
            Publish
          </button>
        </>
      )}
    </div>
  );
}

export function BlogPostForm({ action }: { action: (formData: FormData) => void }) {
  return (
    <form action={action}>
      <FormControls />
    </form>
  );
}
```

### Multiple Submit Buttons

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function ActionButtons() {
  const { pending, data } = useFormStatus();
  const action = data?.get('action');

  return (
    <div className="action-buttons">
      <button
        type="submit"
        name="action"
        value="save"
        disabled={pending}
      >
        {pending && action === 'save' ? 'Saving...' : 'Save'}
      </button>

      <button
        type="submit"
        name="action"
        value="save-and-continue"
        disabled={pending}
      >
        {pending && action === 'save-and-continue'
          ? 'Saving...'
          : 'Save & Continue'}
      </button>

      <button
        type="submit"
        name="action"
        value="submit"
        disabled={pending}
        className="primary"
      >
        {pending && action === 'submit' ? 'Submitting...' : 'Submit'}
      </button>
    </div>
  );
}
```

### Progress Indicator

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function UploadProgress() {
  const { pending, data } = useFormStatus();

  if (!pending) return null;

  const fileCount = data?.getAll('files').length || 0;

  return (
    <div className="upload-progress">
      <div className="progress-bar">
        <div className="progress-bar-fill" />
      </div>
      <p>Uploading {fileCount} file(s)...</p>
    </div>
  );
}

export function FileUploadForm({ action }: { action: (formData: FormData) => void }) {
  return (
    <form action={action}>
      <input type="file" name="files" multiple />
      <UploadProgress />
      <button type="submit">Upload Files</button>
    </form>
  );
}
```

## useActionState Hook

`useActionState` (formerly `useFormState`) manages state returned from Server Actions, providing error handling and success states.

### Basic Usage

```typescript
'use client';

import { useActionState } from 'react';
import { createUser } from './actions';

type FormState = {
  error?: string;
  success?: boolean;
  user?: { id: string; name: string };
};

export function CreateUserForm() {
  const [state, formAction] = useActionState<FormState>(
    createUser,
    {} as FormState
  );

  return (
    <form action={formAction}>
      <input name="name" placeholder="Name" required />
      <input name="email" type="email" placeholder="Email" required />

      {state.error && (
        <div className="error">{state.error}</div>
      )}

      {state.success && state.user && (
        <div className="success">
          User {state.user.name} created successfully!
        </div>
      )}

      <button type="submit">Create User</button>
    </form>
  );
}

// actions.ts
'use server';

export async function createUser(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  if (!name || name.length < 2) {
    return { error: 'Name must be at least 2 characters' };
  }

  try {
    const user = await db.user.create({ data: { name, email } });
    return { success: true, user };
  } catch (error) {
    return { error: 'Failed to create user' };
  }
}
```

### Field-Specific Errors

```typescript
'use client';

import { useActionState } from 'react';
import { registerUser } from './actions';

type FormState = {
  errors?: {
    name?: string[];
    email?: string[];
    password?: string[];
  };
  message?: string;
  success?: boolean;
};

export function RegisterForm() {
  const [state, formAction] = useActionState<FormState>(
    registerUser,
    {} as FormState
  );

  return (
    <form action={formAction}>
      <div className="form-field">
        <label htmlFor="name">Name</label>
        <input id="name" name="name" />
        {state.errors?.name && (
          <p className="field-error">{state.errors.name.join(', ')}</p>
        )}
      </div>

      <div className="form-field">
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" />
        {state.errors?.email && (
          <p className="field-error">{state.errors.email.join(', ')}</p>
        )}
      </div>

      <div className="form-field">
        <label htmlFor="password">Password</label>
        <input id="password" name="password" type="password" />
        {state.errors?.password && (
          <p className="field-error">{state.errors.password.join(', ')}</p>
        )}
      </div>

      {state.message && (
        <div className={state.success ? 'success' : 'error'}>
          {state.message}
        </div>
      )}

      <button type="submit">Register</button>
    </form>
  );
}
```

### With Optimistic Updates

```typescript
'use client';

import { useActionState, useOptimistic } from 'react';
import { addComment } from './actions';

type Comment = {
  id: string;
  text: string;
  author: string;
};

type FormState = {
  error?: string;
  comment?: Comment;
};

export function CommentForm({
  postId,
  comments
}: {
  postId: string;
  comments: Comment[];
}) {
  const [state, formAction] = useActionState<FormState>(
    addComment,
    {} as FormState
  );

  const [optimisticComments, addOptimisticComment] = useOptimistic(
    comments,
    (state, newComment: string) => [
      ...state,
      { id: 'temp', text: newComment, author: 'You', pending: true }
    ]
  );

  async function handleSubmit(formData: FormData) {
    const text = formData.get('text') as string;
    addOptimisticComment(text);
    await formAction(formData);
  }

  return (
    <>
      <ul className="comments">
        {optimisticComments.map(comment => (
          <li key={comment.id} className={comment.pending ? 'pending' : ''}>
            <strong>{comment.author}:</strong> {comment.text}
          </li>
        ))}
      </ul>

      <form action={handleSubmit}>
        <input type="hidden" name="postId" value={postId} />
        <textarea name="text" placeholder="Add a comment..." required />
        
        {state.error && <p className="error">{state.error}</p>}
        
        <button type="submit">Post Comment</button>
      </form>
    </>
  );
}
```

## use() Hook

`use()` is a new hook that can unwrap promises and read context. Unlike other hooks, it can be called conditionally.

### Reading Promises

```typescript
import { use, Suspense } from 'react';

async function fetchUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

function User({ userPromise }: { userPromise: Promise<User> }) {
  // use() unwraps the promise
  const user = use(userPromise);

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

export function UserPage({ userId }: { userId: string }) {
  const userPromise = fetchUser(userId);

  return (
    <Suspense fallback={<UserSkeleton />}>
      <User userPromise={userPromise} />
    </Suspense>
  );
}
```

### Conditional use()

```typescript
import { use } from 'react';

function Comment({ commentId, expand }: { commentId: string; expand: boolean }) {
  // Can call use() conditionally! (unlike other hooks)
  const comment = expand
    ? use(fetchComment(commentId))
    : null;

  return (
    <div>
      <p>Comment #{commentId}</p>
      {comment && (
        <div className="comment-details">
          <p>{comment.text}</p>
          <p>By {comment.author}</p>
        </div>
      )}
    </div>
  );
}
```

### Reading Context

```typescript
import { use, createContext } from 'react';

const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemedButton() {
  // use() can read context
  const theme = use(ThemeContext);

  return (
    <button className={`button-${theme}`}>
      Click me
    </button>
  );
}

// Can also be conditional
function OptionalThemedComponent({ useTheme }: { useTheme: boolean }) {
  const theme = useTheme ? use(ThemeContext) : 'light';
  
  return <div className={theme}>Content</div>;
}
```

### Parallel Data Fetching

```typescript
import { use, Suspense } from 'react';

function ProductPage({ productId }: { productId: string }) {
  // Start fetching in parallel
  const productPromise = fetchProduct(productId);
  const reviewsPromise = fetchReviews(productId);
  const recommendationsPromise = fetchRecommendations(productId);

  return (
    <>
      <Suspense fallback={<ProductSkeleton />}>
        <ProductInfo productPromise={productPromise} />
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <ProductReviews reviewsPromise={reviewsPromise} />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations recommendationsPromise={recommendationsPromise} />
      </Suspense>
    </>
  );
}

function ProductInfo({ productPromise }: { productPromise: Promise<Product> }) {
  const product = use(productPromise);
  return <div>{product.name}</div>;
}

function ProductReviews({ reviewsPromise }: { reviewsPromise: Promise<Review[]> }) {
  const reviews = use(reviewsPromise);
  return <ul>{reviews.map(r => <li key={r.id}>{r.text}</li>)}</ul>;
}

function Recommendations({
  recommendationsPromise
}: {
  recommendationsPromise: Promise<Product[]>;
}) {
  const recommendations = use(recommendationsPromise);
  return <ProductGrid products={recommendations} />;
}
```

## Combining Hooks

### Complete Form with All Hooks

```typescript
'use client';

import { useActionState, useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';
import { submitPost } from './actions';

type Post = {
  id: string;
  title: string;
  content: string;
  published: boolean;
};

type FormState = {
  error?: string;
  post?: Post;
};

function SubmitButtons() {
  const { pending } = useFormStatus();

  return (
    <>
      <button
        type="submit"
        name="action"
        value="draft"
        disabled={pending}
      >
        {pending ? 'Saving...' : 'Save Draft'}
      </button>
      <button
        type="submit"
        name="action"
        value="publish"
        disabled={pending}
      >
        {pending ? 'Publishing...' : 'Publish'}
      </button>
    </>
  );
}

export function PostEditor({ posts }: { posts: Post[] }) {
  const [state, formAction] = useActionState<FormState>(
    submitPost,
    {} as FormState
  );

  const [optimisticPosts, addOptimisticPost] = useOptimistic(
    posts,
    (state, newPost: Partial<Post>) => [
      ...state,
      { ...newPost, id: 'temp', pending: true } as Post
    ]
  );

  async function handleSubmit(formData: FormData) {
    const title = formData.get('title') as string;
    const content = formData.get('content') as string;
    
    addOptimisticPost({ title, content, published: false });
    await formAction(formData);
  }

  return (
    <>
      <form action={handleSubmit}>
        <input name="title" placeholder="Post title..." required />
        <textarea name="content" placeholder="Write your post..." required />

        {state.error && (
          <div className="error">{state.error}</div>
        )}

        <SubmitButtons />
      </form>

      <div className="posts-list">
        <h2>Your Posts</h2>
        {optimisticPosts.map(post => (
          <article
            key={post.id}
            style={{ opacity: post.pending ? 0.5 : 1 }}
          >
            <h3>{post.title}</h3>
            <p>{post.content}</p>
            <span className="status">
              {post.published ? 'Published' : 'Draft'}
              {post.pending && ' (Saving...)'}
            </span>
          </article>
        ))}
      </div>
    </>
  );
}
```

## Common Mistakes

### 1. Using useFormStatus Outside Form

```typescript
// ❌ WRONG: useFormStatus must be in a child of form
'use client';

function BadExample() {
  const { pending } = useFormStatus(); // Error! Not inside form

  return (
    <form>
      <input name="text" />
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ CORRECT: Extract to child component
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}

function GoodExample() {
  return (
    <form>
      <input name="text" />
      <SubmitButton />
    </form>
  );
}
```

### 2. Not Handling Optimistic Update Failures

```typescript
// ❌ BAD: No error handling
function BadOptimistic() {
  const [messages, addMessage] = useOptimistic(
    initialMessages,
    (state, msg: string) => [...state, { text: msg }]
  );

  async function send(text: string) {
    addMessage(text);
    await sendMessage(text); // What if this fails?
  }

  return <MessageList messages={messages} />;
}

// ✅ GOOD: Handle errors
function GoodOptimistic() {
  const [messages, addMessage] = useOptimistic(
    initialMessages,
    (state, msg: string) => [...state, { text: msg }]
  );
  const [error, setError] = useState<string | null>(null);

  async function send(text: string) {
    setError(null);
    addMessage(text);
    
    try {
      await sendMessage(text);
    } catch (err) {
      setError('Failed to send message');
      // Optimistic update automatically reverts
    }
  }

  return (
    <>
      {error && <div className="error">{error}</div>}
      <MessageList messages={messages} />
    </>
  );
}
```

### 3. Calling use() at Top Level Like Other Hooks

```typescript
// ❌ WRONG: Treating use() like useState
'use client';

import { use, useState } from 'react';

function Component({ showData }: { showData: boolean }) {
  const [count, setCount] = useState(0); // Must be at top level
  
  if (showData) {
    const data = use(dataPromise); // This is actually OK!
    return <div>{data}</div>;
  }
  
  // But this is inconsistent and confusing
  return <div>No data</div>;
}

// ✅ BETTER: Be consistent
function Component({ showData }: { showData: boolean }) {
  const data = showData ? use(dataPromise) : null;
  
  return <div>{data ? data : 'No data'}</div>;
}
```

## Best Practices

### 1. Combine useOptimistic with Error Boundaries

```typescript
'use client';

function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, text: string) => [
      ...state,
      { id: 'temp', text, completed: false, pending: true }
    ]
  );

  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <form action={async (formData) => {
        const text = formData.get('text') as string;
        addOptimisticTodo(text);
        await addTodo(formData);
      }}>
        <input name="text" />
        <button>Add</button>
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
    </ErrorBoundary>
  );
}
```

### 2. Use useActionState for Complex Validation

```typescript
'use client';

import { useActionState } from 'react';
import { z } from 'zod';

const PostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(10),
  tags: z.array(z.string()).min(1).max(5)
});

type FormState = {
  errors?: z.ZodFormattedError<z.infer<typeof PostSchema>>;
  success?: boolean;
};

export function PostForm() {
  const [state, formAction] = useActionState<FormState>(
    async (prevState, formData) => {
      const result = PostSchema.safeParse({
        title: formData.get('title'),
        content: formData.get('content'),
        tags: formData.getAll('tags')
      });

      if (!result.success) {
        return { errors: result.error.format() };
      }

      await createPost(result.data);
      return { success: true };
    },
    {} as FormState
  );

  return (
    <form action={formAction}>
      <input name="title" />
      {state.errors?.title && <span>{state.errors.title._errors[0]}</span>}
      
      <textarea name="content" />
      {state.errors?.content && <span>{state.errors.content._errors[0]}</span>}
      
      <button>Submit</button>
    </form>
  );
}
```

### 3. Use use() for Progressive Enhancement

```typescript
import { use, Suspense } from 'react';

function ConditionalData({
  dataPromise,
  showDetails
}: {
  dataPromise: Promise<Data>;
  showDetails: boolean;
}) {
  // Only fetch/unwrap if needed
  const data = showDetails ? use(dataPromise) : null;

  return (
    <div>
      <h2>Summary</h2>
      {data && (
        <div className="details">
          <p>{data.description}</p>
          <p>{data.stats}</p>
        </div>
      )}
    </div>
  );
}

export function ProgressiveDetails({ dataPromise }: { dataPromise: Promise<Data> }) {
  const [show, setShow] = useState(false);

  return (
    <>
      <button onClick={() => setShow(!show)}>
        {show ? 'Hide' : 'Show'} Details
      </button>
      
      <Suspense fallback={<DetailsSkeleton />}>
        <ConditionalData dataPromise={dataPromise} showDetails={show} />
      </Suspense>
    </>
  );
}
```

## Interview Questions

### Q1: What is useOptimistic and when should you use it?

**Answer:** `useOptimistic` allows showing optimistic UI updates immediately while an async action is in progress. Use it for actions like sending messages, liking posts, or adding todos where you want instant feedback. The optimistic state automatically reverts if the action fails. It improves perceived performance by making the UI feel instant.

### Q2: How does useFormStatus differ from useActionState?

**Answer:** `useFormStatus` provides information about the form submission itself (pending, data, method, action) and must be used in a child component of the form. `useActionState` manages the state returned from a Server Action, providing error handling and success states. Use `useFormStatus` for loading indicators, `useActionState` for validation errors and results.

### Q3: What makes the use() hook special compared to other hooks?

**Answer:** Unlike other hooks, `use()` can be called conditionally (inside if statements, loops, etc.) because it doesn't rely on render order. It can unwrap promises (for data fetching with Suspense) and read context. This makes it more flexible than hooks like useState or useEffect which must be called at the top level.

### Q4: Can you combine useOptimistic and useActionState?

**Answer:** Yes, they complement each other well. Use `useOptimistic` for immediate UI updates and `useActionState` to handle the result from the server (errors, validation, success). The optimistic state shows instant feedback, while the action state provides error handling and server validation results.

### Q5: Why must useFormStatus be in a child component?

**Answer:** `useFormStatus` reads the status of its parent form. React needs to establish this parent-child relationship during rendering. If you use it in the same component as the form, there's no parent form to read from. It's similar to how context consumers need to be children of providers.

## Key Takeaways

1. **useOptimistic for instant UI** - Show updates immediately, revert on failure
2. **useFormStatus for form state** - Must be in child component of form
3. **useActionState for server results** - Handle errors and validation from Server Actions
4. **use() is conditionally callable** - Unlike other hooks, can use in if statements
5. **Combine hooks for best UX** - useOptimistic + useFormStatus + useActionState together
6. **Always handle errors** - Optimistic updates can fail, show appropriate feedback
7. **Progressive enhancement** - Forms work without JS, enhanced when available
8. **Type-safe with TypeScript** - All hooks have strong TypeScript support

## Resources

### Official Documentation
- [useOptimistic Hook](https://react.dev/reference/react/useOptimistic)
- [useFormStatus Hook](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [useActionState Hook](https://react.dev/reference/react/useActionState)
- [use() Hook](https://react.dev/reference/react/use)

### Articles
- "React 19 Hooks Overview" - React Team
- "Optimistic UI Updates in React 19" - Kent C. Dodds
- "Form Handling in React 19" - Web.dev

### Examples
- [React 19 Examples](https://github.com/reactjs/react.dev/tree/main/examples)
- [Next.js Forms with Server Actions](https://github.com/vercel/next.js/tree/canary/examples/forms)

---

*Last Updated: 2026-05 - React 19 current*
