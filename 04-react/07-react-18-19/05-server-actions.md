# Server Actions in React 19

## The Idea

**In plain English:** Server Actions are special functions that run on the computer hosting your website (the server), not in your visitor's browser — so when someone fills out a form and clicks submit, the function saving their data runs safely on the server, not on their device. A "function" here is just a named set of instructions the computer follows to do a job.

**Real-world analogy:** Think of ordering food at a restaurant by writing your order on a paper slip and handing it to the waiter. You never enter the kitchen yourself; the kitchen receives the slip and prepares the meal, then the waiter brings the result back to your table.

- The paper slip = the HTML form the user fills out in the browser
- The waiter carrying the slip to the kitchen = the network request that sends the form data to the server
- The kitchen preparing the meal = the Server Action function running securely on the server
- The finished meal returned to your table = the updated page or response sent back to the browser

---

## Table of Contents

- [Introduction](#introduction)
- [What are Server Actions?](#what-are-server-actions)
- ["use server" Directive](#use-server-directive)
- [Form Actions](#form-actions)
- [Progressive Enhancement](#progressive-enhancement)
- [useActionState Hook](#useactionstate-hook)
- [useFormStatus Hook](#useformstatus-hook)
- [Revalidation](#revalidation)
- [Error Handling](#error-handling)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Server Actions are asynchronous functions that run on the server and can be called from Client or Server Components. They enable direct server-side mutations without requiring API routes, with built-in support for forms, optimistic updates, and progressive enhancement.

## What are Server Actions?

Server Actions are functions marked with `"use server"` that execute on the server. They can be defined inline in Server Components or in separate files and imported into Client Components.

**Traditional API Route Pattern:**
```typescript
// app/api/create-post/route.ts
export async function POST(request: Request) {
  const { title, content } = await request.json();
  
  const post = await db.post.create({
    data: { title, content }
  });
  
  return Response.json(post);
}

// Client Component
'use client';

export function CreatePostForm() {
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    
    const response = await fetch('/api/create-post', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: formData.get('title'),
        content: formData.get('content')
      })
    });
    
    const post = await response.json();
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" />
      <textarea name="content" />
      <button>Create Post</button>
    </form>
  );
}
```

**Server Actions Pattern:**
```typescript
// app/actions.ts
'use server';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  
  const post = await db.post.create({
    data: { title, content }
  });
  
  return post;
}

// Client Component
'use client';

import { createPost } from './actions';

export function CreatePostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <textarea name="content" />
      <button>Create Post</button>
    </form>
  );
}
```

## "use server" Directive

The `"use server"` directive marks functions or files as Server Actions.

### File-Level Directive

```typescript
// app/actions.ts
'use server';

// All exported functions are Server Actions
export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  
  const user = await db.user.create({
    data: { name, email }
  });
  
  revalidatePath('/users');
  return user;
}

export async function deleteUser(userId: string) {
  await db.user.delete({ where: { id: userId } });
  revalidatePath('/users');
}

export async function updateUserRole(userId: string, role: string) {
  await db.user.update({
    where: { id: userId },
    data: { role }
  });
  revalidatePath('/users');
}
```

### Function-Level Directive

```typescript
// Server Component
async function Page() {
  // Inline Server Action
  async function createPost(formData: FormData) {
    'use server';
    
    const title = formData.get('title') as string;
    const content = formData.get('content') as string;
    
    await db.post.create({
      data: { title, content }
    });
    
    revalidatePath('/posts');
    redirect('/posts');
  }

  return (
    <form action={createPost}>
      <input name="title" />
      <textarea name="content" />
      <button>Create</button>
    </form>
  );
}
```

### Importing Server Actions

```typescript
// app/actions.ts
'use server';

export async function subscribe(email: string) {
  await db.subscription.create({
    data: { email }
  });
  return { success: true };
}

// Client Component
'use client';

import { subscribe } from './actions';

export function Newsletter() {
  const [email, setEmail] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const result = await subscribe(email);
    if (result.success) {
      setEmail('');
      alert('Subscribed!');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={email}
        onChange={e => setEmail(e.target.value)}
      />
      <button>Subscribe</button>
    </form>
  );
}
```

## Form Actions

Server Actions integrate seamlessly with HTML forms using the `action` attribute.

### Basic Form Action

```typescript
// app/actions.ts
'use server';

export async function createTodo(formData: FormData) {
  const title = formData.get('title') as string;
  
  await db.todo.create({
    data: { title, completed: false }
  });
  
  revalidatePath('/todos');
}

// app/todos/page.tsx
import { createTodo } from '../actions';

export default function TodosPage() {
  return (
    <form action={createTodo}>
      <input name="title" placeholder="New todo..." required />
      <button type="submit">Add Todo</button>
    </form>
  );
}
```

### Form with Validation

```typescript
'use server';

import { z } from 'zod';

const PostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(10),
  published: z.boolean().default(false)
});

export async function createPost(formData: FormData) {
  const validatedFields = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
    published: formData.get('published') === 'on'
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
      message: 'Invalid fields'
    };
  }

  const { title, content, published } = validatedFields.data;

  try {
    await db.post.create({
      data: { title, content, published }
    });
  } catch (error) {
    return {
      message: 'Database error: Failed to create post'
    };
  }

  revalidatePath('/posts');
  redirect('/posts');
}
```

### Multiple Actions in One Form

```typescript
'use server';

export async function saveDraft(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  
  await db.post.create({
    data: { title, content, status: 'draft' }
  });
  
  revalidatePath('/posts/drafts');
}

export async function publishPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  
  await db.post.create({
    data: { title, content, status: 'published' }
  });
  
  revalidatePath('/posts');
}

// Component
export function PostForm() {
  return (
    <form>
      <input name="title" />
      <textarea name="content" />
      
      <button formAction={saveDraft}>Save Draft</button>
      <button formAction={publishPost}>Publish</button>
    </form>
  );
}
```

### Passing Additional Arguments

```typescript
'use server';

export async function updatePost(postId: string, formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  
  await db.post.update({
    where: { id: postId },
    data: { title, content }
  });
  
  revalidatePath(`/posts/${postId}`);
}

// Using .bind() to pass arguments
'use client';

import { updatePost } from './actions';

export function EditPostForm({ postId }: { postId: string }) {
  const updatePostWithId = updatePost.bind(null, postId);

  return (
    <form action={updatePostWithId}>
      <input name="title" />
      <textarea name="content" />
      <button>Update Post</button>
    </form>
  );
}
```

## Progressive Enhancement

Forms with Server Actions work without JavaScript, providing progressive enhancement.

### Basic Progressive Enhancement

```typescript
// app/actions.ts
'use server';

export async function addComment(formData: FormData) {
  const postId = formData.get('postId') as string;
  const content = formData.get('content') as string;
  
  await db.comment.create({
    data: { postId, content }
  });
  
  revalidatePath(`/posts/${postId}`);
}

// Works without JavaScript!
export function CommentForm({ postId }: { postId: string }) {
  return (
    <form action={addComment}>
      <input type="hidden" name="postId" value={postId} />
      <textarea name="content" required />
      <button>Add Comment</button>
    </form>
  );
}
```

### Enhanced with JavaScript

```typescript
'use client';

import { addComment } from './actions';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Adding...' : 'Add Comment'}
    </button>
  );
}

export function CommentForm({ postId }: { postId: string }) {
  const [optimisticComments, addOptimisticComment] = useOptimistic(
    comments,
    (state, newComment: string) => [...state, { content: newComment, pending: true }]
  );

  async function handleSubmit(formData: FormData) {
    const content = formData.get('content') as string;
    addOptimisticComment(content);
    await addComment(formData);
  }

  return (
    <>
      <form action={handleSubmit}>
        <input type="hidden" name="postId" value={postId} />
        <textarea name="content" required />
        <SubmitButton />
      </form>
      
      <CommentList comments={optimisticComments} />
    </>
  );
}
```

### Fallback for No-JS

```typescript
export function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.title}
          
          {/* Works without JS - full page reload */}
          <form action={deleteTodo} method="post">
            <input type="hidden" name="id" value={todo.id} />
            <button>Delete</button>
          </form>
        </li>
      ))}
    </ul>
  );
}
```

## useActionState Hook

`useActionState` (formerly `useFormState`) manages form state with Server Actions.

### Basic Usage

```typescript
'use server';

export async function createUser(
  prevState: any,
  formData: FormData
) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // Validation
  if (!name || name.length < 2) {
    return { error: 'Name must be at least 2 characters' };
  }

  if (!email || !email.includes('@')) {
    return { error: 'Invalid email address' };
  }

  try {
    const user = await db.user.create({
      data: { name, email }
    });
    
    return { success: true, user };
  } catch (error) {
    return { error: 'Failed to create user' };
  }
}

// Client Component
'use client';

import { useActionState } from 'react';
import { createUser } from './actions';

export function CreateUserForm() {
  const [state, formAction] = useActionState(createUser, null);

  return (
    <form action={formAction}>
      <input name="name" required />
      <input name="email" type="email" required />
      
      {state?.error && (
        <p className="error">{state.error}</p>
      )}
      
      {state?.success && (
        <p className="success">User created: {state.user.name}</p>
      )}
      
      <button type="submit">Create User</button>
    </form>
  );
}
```

### With Field-Specific Errors

```typescript
'use server';

type FormState = {
  errors?: {
    name?: string[];
    email?: string[];
    password?: string[];
  };
  message?: string;
  success?: boolean;
};

export async function registerUser(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;

  const errors: FormState['errors'] = {};

  if (!name || name.length < 2) {
    errors.name = ['Name must be at least 2 characters'];
  }

  if (!email || !email.includes('@')) {
    errors.email = ['Invalid email address'];
  }

  if (!password || password.length < 8) {
    errors.password = ['Password must be at least 8 characters'];
  }

  if (Object.keys(errors).length > 0) {
    return { errors, message: 'Validation failed' };
  }

  try {
    await db.user.create({
      data: { name, email, password: await hash(password) }
    });
    
    return { success: true, message: 'User registered successfully' };
  } catch (error) {
    return { message: 'Failed to register user' };
  }
}

// Client Component
'use client';

import { useActionState } from 'react';
import { registerUser } from './actions';

export function RegisterForm() {
  const [state, formAction] = useActionState(registerUser, {});

  return (
    <form action={formAction}>
      <div>
        <label>Name</label>
        <input name="name" />
        {state.errors?.name && (
          <p className="error">{state.errors.name.join(', ')}</p>
        )}
      </div>

      <div>
        <label>Email</label>
        <input name="email" type="email" />
        {state.errors?.email && (
          <p className="error">{state.errors.email.join(', ')}</p>
        )}
      </div>

      <div>
        <label>Password</label>
        <input name="password" type="password" />
        {state.errors?.password && (
          <p className="error">{state.errors.password.join(', ')}</p>
        )}
      </div>

      {state.message && (
        <p className={state.success ? 'success' : 'error'}>
          {state.message}
        </p>
      )}

      <button type="submit">Register</button>
    </form>
  );
}
```

## useFormStatus Hook

`useFormStatus` provides the status of the parent form.

### Basic Loading State

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

export function MyForm({ action }: { action: (formData: FormData) => void }) {
  return (
    <form action={action}>
      <input name="title" />
      <SubmitButton />
    </form>
  );
}
```

### All Status Properties

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function FormStatus() {
  const { pending, data, method, action } = useFormStatus();

  return (
    <div>
      <p>Pending: {pending.toString()}</p>
      <p>Method: {method}</p>
      <p>Action: {action}</p>
      {data && (
        <p>Submitting data: {JSON.stringify(Object.fromEntries(data))}</p>
      )}
    </div>
  );
}
```

### Conditional Rendering Based on Status

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function FormControls() {
  const { pending } = useFormStatus();

  return (
    <>
      <input name="title" disabled={pending} />
      <textarea name="content" disabled={pending} />
      
      {pending ? (
        <div className="loading-overlay">
          <Spinner />
          <p>Saving...</p>
        </div>
      ) : (
        <button type="submit">Save</button>
      )}
    </>
  );
}
```

### Multiple Submit Buttons

```typescript
'use client';

import { useFormStatus } from 'react-dom';

function SaveButtons() {
  const { pending } = useFormStatus();

  return (
    <div className="button-group">
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
    </div>
  );
}
```

## Revalidation

Server Actions can revalidate cached data using `revalidatePath` and `revalidateTag`.

### revalidatePath

```typescript
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  
  await db.post.create({
    data: { title, content }
  });
  
  // Revalidate specific path
  revalidatePath('/posts');
}

export async function updatePost(postId: string, formData: FormData) {
  await db.post.update({
    where: { id: postId },
    data: {
      title: formData.get('title') as string,
      content: formData.get('content') as string
    }
  });
  
  // Revalidate specific post
  revalidatePath(`/posts/${postId}`);
  
  // Also revalidate list
  revalidatePath('/posts');
}
```

### revalidateTag

```typescript
'use server';

import { revalidateTag } from 'next/cache';

export async function createProduct(formData: FormData) {
  const name = formData.get('name') as string;
  const category = formData.get('category') as string;
  
  await db.product.create({
    data: { name, category }
  });
  
  // Revalidate all data tagged with 'products'
  revalidateTag('products');
  
  // Revalidate category-specific data
  revalidateTag(`category-${category}`);
}

// Data fetching with tags
async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { tags: ['products'] }
  });
  return res.json();
}

async function getProductsByCategory(category: string) {
  const res = await fetch(`https://api.example.com/products?category=${category}`, {
    next: { tags: ['products', `category-${category}`] }
  });
  return res.json();
}
```

### Granular Revalidation

```typescript
'use server';

export async function updateUserProfile(
  userId: string,
  formData: FormData
) {
  await db.user.update({
    where: { id: userId },
    data: {
      name: formData.get('name') as string,
      bio: formData.get('bio') as string
    }
  });
  
  // Revalidate user's profile page
  revalidatePath(`/users/${userId}`);
  
  // Revalidate user's posts (shows updated author name)
  revalidatePath(`/users/${userId}/posts`);
  
  // Don't revalidate unrelated pages
}
```

## Error Handling

### Try-Catch in Server Actions

```typescript
'use server';

export async function deletePost(postId: string) {
  try {
    await db.post.delete({
      where: { id: postId }
    });
    
    revalidatePath('/posts');
    
    return { success: true };
  } catch (error) {
    console.error('Failed to delete post:', error);
    
    return {
      success: false,
      error: 'Failed to delete post'
    };
  }
}
```

### Validation Errors

```typescript
'use server';

import { z } from 'zod';

const ContactSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email'),
  message: z.string().min(10, 'Message must be at least 10 characters')
});

export async function submitContact(formData: FormData) {
  const validatedFields = ContactSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    message: formData.get('message')
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors
    };
  }

  try {
    await db.contact.create({
      data: validatedFields.data
    });
    
    return { success: true };
  } catch (error) {
    return {
      error: 'Failed to submit contact form'
    };
  }
}
```

### Error Boundaries

```typescript
// error.tsx (Error Boundary)
'use client';

export default function Error({
  error,
  reset
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// Server Action that throws
'use server';

export async function riskyAction(formData: FormData) {
  const data = formData.get('data');
  
  if (!data) {
    throw new Error('Data is required');
  }
  
  // Caught by error boundary
  await db.risky.create({ data });
  
  revalidatePath('/risky');
}
```

## Common Mistakes

### 1. Exposing Sensitive Data

```typescript
// ❌ BAD: Returning sensitive data
'use server';

export async function getUser(userId: string) {
  const user = await db.user.findUnique({
    where: { id: userId },
    include: { password: true } // Exposed to client!
  });
  
  return user;
}

// ✅ GOOD: Only return necessary data
'use server';

export async function getUser(userId: string) {
  const user = await db.user.findUnique({
    where: { id: userId },
    select: {
      id: true,
      name: true,
      email: true
      // password NOT included
    }
  });
  
  return user;
}
```

### 2. Not Validating Input

```typescript
// ❌ BAD: No validation
'use server';

export async function createPost(formData: FormData) {
  await db.post.create({
    data: {
      title: formData.get('title'),
      content: formData.get('content')
    }
  });
}

// ✅ GOOD: Validate input
'use server';

import { z } from 'zod';

const PostSchema = z.object({
  title: z.string().min(1).max(100),
  content: z.string().min(10)
});

export async function createPost(formData: FormData) {
  const validatedFields = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content')
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors
    };
  }

  await db.post.create({
    data: validatedFields.data
  });
  
  return { success: true };
}
```

### 3. Forgetting Revalidation

```typescript
// ❌ BAD: No revalidation
'use server';

export async function updateSettings(formData: FormData) {
  await db.settings.update({
    data: { theme: formData.get('theme') }
  });
  // Page shows stale data!
}

// ✅ GOOD: Revalidate
'use server';

export async function updateSettings(formData: FormData) {
  await db.settings.update({
    data: { theme: formData.get('theme') }
  });
  
  revalidatePath('/settings');
}
```

## Best Practices

### 1. Use Zod for Validation

```typescript
'use server';

import { z } from 'zod';

const UserSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
  age: z.number().int().positive().max(120)
});

export async function createUser(formData: FormData) {
  const validatedFields = UserSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    age: Number(formData.get('age'))
  });

  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors
    };
  }

  const user = await db.user.create({
    data: validatedFields.data
  });

  revalidatePath('/users');
  return { success: true, user };
}
```

### 2. Provide Good Error Messages

```typescript
'use server';

export async function deleteAccount(userId: string) {
  try {
    // Check if user exists
    const user = await db.user.findUnique({ where: { id: userId } });
    
    if (!user) {
      return { error: 'User not found' };
    }

    // Check if user has permissions
    const session = await getSession();
    if (session.userId !== userId) {
      return { error: 'Unauthorized' };
    }

    await db.user.delete({ where: { id: userId } });
    
    return { success: true };
  } catch (error) {
    console.error('Delete account error:', error);
    return { error: 'Failed to delete account. Please try again.' };
  }
}
```

### 3. Use Optimistic Updates

```typescript
'use client';

import { useOptimistic } from 'react';
import { addTodo } from './actions';

export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo: string) => [
      ...state,
      { id: 'temp', title: newTodo, completed: false, pending: true }
    ]
  );

  async function handleSubmit(formData: FormData) {
    const title = formData.get('title') as string;
    addOptimisticTodo(title);
    await addTodo(formData);
  }

  return (
    <>
      <form action={handleSubmit}>
        <input name="title" />
        <button>Add</button>
      </form>
      
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            {todo.title}
          </li>
        ))}
      </ul>
    </>
  );
}
```

## Interview Questions

### Q1: What are Server Actions and how do they differ from API routes?

**Answer:** Server Actions are functions marked with "use server" that run on the server and can be called from Client or Server Components. Unlike API routes which require separate endpoint definitions and fetch calls, Server Actions can be called directly like regular functions. They integrate natively with forms, provide automatic serialization, and support progressive enhancement.

### Q2: How does the "use server" directive work?

**Answer:** The "use server" directive can be used at file-level (making all exports Server Actions) or function-level (for inline actions). It tells React to serialize the function reference and make it callable from the client. When called, the function executes on the server with secure access to databases and secrets.

### Q3: What is progressive enhancement in the context of Server Actions?

**Answer:** Progressive enhancement means forms with Server Actions work without JavaScript. The form submits to the server, executes the action, and returns updated HTML. When JavaScript loads, the form is enhanced with features like optimistic updates, loading states, and client-side validation, but the core functionality works without JS.

### Q4: How do you handle errors in Server Actions?

**Answer:** Use try-catch blocks to handle errors, validate input with libraries like Zod, return error objects from actions, and use Error Boundaries to catch unexpected errors. Always provide meaningful error messages to users and log errors server-side for debugging.

### Q5: When should you revalidate data after a Server Action?

**Answer:** Revalidate whenever the action modifies data that's displayed elsewhere. Use `revalidatePath()` for specific routes or `revalidateTag()` for tagged fetch requests. Be granular - only revalidate affected paths to avoid unnecessary work.

## Key Takeaways

1. **Server Actions eliminate API routes** - Call server functions directly from components
2. **"use server" marks server-side functions** - File-level or function-level directive
3. **Native form integration** - Use with action attribute for zero-JS forms
4. **Progressive enhancement** - Works without JavaScript, enhanced when available
5. **useActionState for form state** - Manage errors and success states
6. **useFormStatus for loading states** - Access pending state in form children
7. **Always validate input** - Use Zod or similar for type-safe validation
8. **Revalidate after mutations** - Use revalidatePath or revalidateTag to update cache

## Resources

### Official Documentation
- [Server Actions in Next.js](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [useActionState Hook](https://react.dev/reference/react/useActionState)
- [useFormStatus Hook](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [useOptimistic Hook](https://react.dev/reference/react/useOptimistic)

### Articles
- "Server Actions in React 19" - React Team
- "Building Forms with Server Actions" - Next.js Team
- "Progressive Enhancement with Server Actions" - Web.dev

### Libraries
- [Zod](https://zod.dev) - TypeScript-first schema validation
- [React Hook Form + Server Actions](https://react-hook-form.com/docs/useformstate)

---

*Last Updated: 2026-05 - React 19 current*
