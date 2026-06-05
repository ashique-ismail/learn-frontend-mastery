# Normalization and Relational State

## The Idea

**In plain English:** Normalization means storing each piece of information in exactly one place and pointing to it by ID, so that when something changes you only update it once. Think of it like a well-organized filing system where every fact lives in one folder, and everything else just holds a reference (a label) to that folder instead of keeping its own copy.

**Real-world analogy:** Imagine a school where every student's contact details are written on a single card in the main office. Each classroom has a list of student ID numbers, not a copy of every student's details. When a student's phone number changes, the office updates one card and every classroom's list instantly reflects the correct information.

- The student card in the main office = the entity stored once in `entities` by ID
- The classroom's list of ID numbers = the `ids` array that points to entities
- Copying the details onto every classroom notice board = nested (un-normalized) state that goes stale

---

## Overview

Normalizing state means storing data in a flat, deduplicated structure keyed by ID — the same principle relational databases use. In complex frontend applications with nested or repeated entities (users with posts with comments), storing raw server responses in component state leads to duplication, inconsistency, and O(n) lookups. Normalized state solves all three problems at the cost of slightly more complex reads that must "join" data back together. This guide covers the normalization pattern, selector-based reads, and when strategic denormalization makes sense.

## Why Normalization Matters

```text
Without normalization — duplicated data everywhere:

{
  posts: [
    {
      id: "p1",
      title: "Hello",
      author: { id: "u1", name: "Alice", avatar: "/alice.png" }
    },
    {
      id: "p2",
      title: "World",
      author: { id: "u1", name: "Alice", avatar: "/alice.png" }  // duplicated!
    }
  ]
}

Problem: Alice's avatar changes → need to update every post. Miss one → stale data.

With normalization — single source of truth:

{
  entities: {
    users: {
      u1: { id: "u1", name: "Alice", avatar: "/alice.png" }
    },
    posts: {
      p1: { id: "p1", title: "Hello", authorId: "u1" },
      p2: { id: "p2", title: "World", authorId: "u1" }
    }
  },
  ids: {
    posts: ["p1", "p2"]
  }
}

Update Alice once → all posts see the change automatically.
```

## The Normalized Shape

### Entity Dictionary + ID Array

```typescript
// The canonical normalized state shape
interface EntityState<T> {
  ids: string[];           // Ordered list for iteration and display order
  entities: Record<string, T>;  // O(1) lookup by ID
}

// Concrete example
interface User {
  id: string;
  name: string;
  email: string;
  avatarUrl: string;
}

interface Post {
  id: string;
  title: string;
  body: string;
  authorId: string;      // reference, not nested object
  tagIds: string[];      // array of references
  createdAt: string;
}

interface Comment {
  id: string;
  body: string;
  authorId: string;
  postId: string;
}

interface Tag {
  id: string;
  label: string;
  color: string;
}

// Normalized store
interface NormalizedState {
  users: EntityState<User>;
  posts: EntityState<Post>;
  comments: EntityState<Comment>;
  tags: EntityState<Tag>;
}
```

### Manual Normalization

```typescript
// Normalizing a nested API response by hand
interface ApiPost {
  id: string;
  title: string;
  author: {
    id: string;
    name: string;
    email: string;
  };
  comments: Array<{
    id: string;
    body: string;
    author: { id: string; name: string; email: string };
  }>;
  tags: Array<{ id: string; label: string; color: string }>;
}

function normalizePosts(apiPosts: ApiPost[]): Partial<NormalizedState> {
  const users: Record<string, User> = {};
  const posts: Record<string, Post> = {};
  const comments: Record<string, Comment> = {};
  const tags: Record<string, Tag> = {};

  const postIds: string[] = [];

  for (const post of apiPosts) {
    // Normalize author
    users[post.author.id] = {
      id: post.author.id,
      name: post.author.name,
      email: post.author.email,
      avatarUrl: '',
    };

    // Normalize tags
    const tagIds: string[] = [];
    for (const tag of post.tags) {
      tags[tag.id] = tag;
      tagIds.push(tag.id);
    }

    // Normalize comments
    for (const comment of post.comments) {
      users[comment.author.id] = {
        id: comment.author.id,
        name: comment.author.name,
        email: comment.author.email,
        avatarUrl: '',
      };
      comments[comment.id] = {
        id: comment.id,
        body: comment.body,
        authorId: comment.author.id,
        postId: post.id,
      };
    }

    // Normalize post
    posts[post.id] = {
      id: post.id,
      title: post.title,
      body: '',
      authorId: post.author.id,
      tagIds,
      createdAt: '',
    };

    postIds.push(post.id);
  }

  return {
    users: { ids: Object.keys(users), entities: users },
    posts: { ids: postIds, entities: posts },
    comments: { ids: Object.keys(comments), entities: comments },
    tags: { ids: Object.keys(tags), entities: tags },
  };
}
```

## Normalizr Library

```typescript
import { normalize, schema } from 'normalizr';

// Define schemas
const userSchema = new schema.Entity('users');

const tagSchema = new schema.Entity('tags');

const commentSchema = new schema.Entity('comments', {
  author: userSchema,
});

const postSchema = new schema.Entity('posts', {
  author: userSchema,
  comments: [commentSchema],
  tags: [tagSchema],
});

// Normalize API response
const apiResponse = await fetchPosts();

const normalized = normalize(apiResponse, [postSchema]);
// normalized.result   → ["p1", "p2", ...]   (ordered IDs)
// normalized.entities → { users: {...}, posts: {...}, comments: {...}, tags: {...} }

// Merging into Redux store
dispatch(entitiesReceived(normalized.entities));
dispatch(postIdsReceived(normalized.result));
```

## Redux Toolkit: createEntityAdapter

RTK ships a first-class normalization primitive:

```typescript
import {
  createEntityAdapter,
  createSlice,
  createAsyncThunk,
  PayloadAction,
} from '@reduxjs/toolkit';

// createEntityAdapter generates the normalized shape + CRUD reducers
const usersAdapter = createEntityAdapter<User>({
  selectId: (user) => user.id,          // default: entity.id
  sortComparer: (a, b) => a.name.localeCompare(b.name),  // optional
});

const postsAdapter = createEntityAdapter<Post>({
  sortComparer: (a, b) => b.createdAt.localeCompare(a.createdAt), // newest first
});

// Initial state from adapter: { ids: [], entities: {} }
const postsSlice = createSlice({
  name: 'posts',
  initialState: postsAdapter.getInitialState({
    status: 'idle' as 'idle' | 'loading' | 'succeeded' | 'failed',
    error: null as string | null,
  }),
  reducers: {
    postUpdated: postsAdapter.updateOne,
    postRemoved: postsAdapter.removeOne,
  },
  extraReducers: (builder) => {
    builder.addCase(fetchPosts.fulfilled, (state, action) => {
      // action.payload is normalized entities
      postsAdapter.setAll(state, action.payload.posts);
      state.status = 'succeeded';
    });
  },
});

// Available adapter CRUD methods:
// addOne, addMany, setOne, setMany, setAll
// updateOne, updateMany, upsertOne, upsertMany
// removeOne, removeMany, removeAll

// Adapter-generated selectors
const postsSelectors = postsAdapter.getSelectors(
  (state: RootState) => state.posts
);

// selectAll      → Post[] (in sorted order)
// selectById     → Post | undefined
// selectIds      → string[]
// selectEntities → Record<string, Post>
// selectTotal    → number
```

## Selector Patterns for Normalized State

```typescript
import { createSelector } from '@reduxjs/toolkit';

// Base selectors (inexpensive, direct reads)
const selectPostEntities = (state: RootState) => state.posts.entities;
const selectUserEntities = (state: RootState) => state.users.entities;
const selectTagEntities  = (state: RootState) => state.tags.entities;
const selectPostIds      = (state: RootState) => state.posts.ids;

// "Denormalizing" for the view layer — reassemble the shape UI needs
const selectAllPostsWithAuthor = createSelector(
  [postsSelectors.selectAll, selectUserEntities],
  (posts, userEntities) =>
    posts.map((post) => ({
      ...post,
      author: userEntities[post.authorId],
    }))
);

// Per-entity selector with parameter
const makeSelectPostById = (postId: string) =>
  createSelector(
    [selectPostEntities, selectUserEntities, selectTagEntities],
    (postEntities, userEntities, tagEntities) => {
      const post = postEntities[postId];
      if (!post) return null;
      return {
        ...post,
        author: userEntities[post.authorId],
        tags: post.tagIds.map((id) => tagEntities[id]).filter(Boolean),
      };
    }
  );

// Usage in component
function PostCard({ postId }: { postId: string }) {
  // Selector factory called outside render or with useMemo to keep reference stable
  const selectPost = useMemo(() => makeSelectPostById(postId), [postId]);
  const post = useSelector(selectPost);

  if (!post) return null;
  return (
    <article>
      <h2>{post.title}</h2>
      <span>by {post.author?.name}</span>
      {post.tags.map((tag) => (
        <Tag key={tag.id} label={tag.label} color={tag.color} />
      ))}
    </article>
  );
}
```

## Updating Normalized State

```typescript
// Optimistic updates are clean with normalization
const updatePostTitle = createAsyncThunk(
  'posts/updateTitle',
  async ({ postId, title }: { postId: string; title: string }, thunkAPI) => {
    await api.updatePost(postId, { title });
    return { postId, title };
  }
);

const postsSlice = createSlice({
  name: 'posts',
  initialState: postsAdapter.getInitialState(),
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(updatePostTitle.pending, (state, action) => {
        // Optimistic: update immediately
        postsAdapter.updateOne(state, {
          id: action.meta.arg.postId,
          changes: { title: action.meta.arg.title },
        });
      })
      .addCase(updatePostTitle.rejected, (state, action) => {
        // Rollback: revert to previous value stored in meta
        postsAdapter.updateOne(state, {
          id: action.meta.arg.postId,
          changes: { title: action.meta.arg.previousTitle },
        });
      });
  },
});
```

## Denormalization for Reads

Normalization is about storage; denormalization is about consumption. They are not opposites — you store normalized and denormalize at read time via selectors.

```typescript
// ❌ Over-normalizing: too fine-grained
interface Address {
  streetId: string;
  cityId: string;
  countryId: string;
}
// Normalizing an address into 3 separate entity tables is overkill
// if the address is never updated independently

// ✅ Practical rule: normalize entities that:
//   1. Appear in multiple places
//   2. Are updated independently
//   3. Have many-to-many relationships

// ❌ Under-normalizing: user profile deeply nested in each post
interface Post {
  id: string;
  title: string;
  author: {
    id: string;
    name: string;
    role: string;
    followerCount: number;
    // ...10 more fields
  };
}

// If author data changes, every post copy is stale
// O(n) scan to update all posts when user changes their name

// ✅ Right normalization level:
interface Post {
  id: string;
  title: string;
  authorId: string;  // reference only
}
// Read time: join via selector — zero cost if memoized
```

## Pagination with Normalized State

```typescript
interface PaginatedState<T> extends EntityState<T> {
  pagination: {
    currentPage: number;
    totalPages: number;
    pageSize: number;
    // Page → ID arrays mapping
    pageIds: Record<number, string[]>;
  };
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
}

const postsSlice = createSlice({
  name: 'posts',
  initialState: postsAdapter.getInitialState<{
    pagination: { pageIds: Record<number, string[]>; totalPages: number };
  }>({
    pagination: { pageIds: {}, totalPages: 0 },
  }),
  extraReducers: (builder) => {
    builder.addCase(fetchPostsPage.fulfilled, (state, action) => {
      const { posts, page, totalPages } = action.payload;

      // Upsert entities (don't overwrite other pages)
      postsAdapter.upsertMany(state, posts);

      // Store page order
      state.pagination.pageIds[page] = posts.map((p) => p.id);
      state.pagination.totalPages = totalPages;
    });
  },
});

// Selector for current page
const selectCurrentPagePosts = createSelector(
  [postsSelectors.selectEntities, selectCurrentPage, selectPageIds],
  (entities, currentPage, pageIds) => {
    const ids = pageIds[currentPage] ?? [];
    return ids.map((id) => entities[id]).filter((p): p is Post => Boolean(p));
  }
);
```

## Common Mistakes

### 1. Forgetting to Update Both ids and entities

```typescript
// ❌ Bad: Only updating entities, not ids array
state.posts.entities[newPost.id] = newPost;
// ids doesn't include newPost.id → post invisible to selectAll

// ✅ Good: Use adapter methods that keep both in sync
postsAdapter.addOne(state, newPost);
```

### 2. Deep Mutation on Nested Objects

```typescript
// ❌ Bad: Mutating nested entity reference
state.posts.entities[id].author.name = 'New Name';
// author is a nested object — this is incorrect if author is also in users slice

// ✅ Good: Update in the canonical slice
state.users.entities[userId] = { ...state.users.entities[userId], name: 'New Name' };
// All references via authorId now see new name through selectors
```

### 3. Recreating Selector Factories on Every Render

```typescript
// ❌ Bad: New selector instance each render → no memoization benefit
function PostCard({ postId }: { postId: string }) {
  const post = useSelector(makeSelectPostById(postId)); // new selector every render
}

// ✅ Good: Stable selector reference
function PostCard({ postId }: { postId: string }) {
  const selectPost = useMemo(() => makeSelectPostById(postId), [postId]);
  const post = useSelector(selectPost);
}
```

### 4. Normalizing Non-Entity Data

```typescript
// ❌ Bad: Normalizing UI state
entities: {
  modals: { "confirmDelete": { id: "confirmDelete", isOpen: true } }
}

// ✅ Good: UI state stays in component or simple slice
const uiSlice = createSlice({
  name: 'ui',
  initialState: { confirmDeleteOpen: false, selectedPostId: null as string | null },
  reducers: { /* ... */ }
});
```

### 5. Selecting Entire Entities Slice in One Component

```typescript
// ❌ Bad: Re-renders on any entity change
const allUsers = useSelector((s) => s.users.entities);

// ✅ Good: Select only what the component needs
const user = useSelector((s) => s.users.entities[userId]);
```

## Interview Questions

### 1. What problem does state normalization solve that you can't solve with nested state?

**Answer**: Nested state creates three compounding problems: (1) **Duplication** — the same user object appears inside every post they authored; when their avatar changes, you must find and update every copy. (2) **Reference inequality** — even a no-op update to one post creates a new array that causes all post-list components to re-render. (3) **O(n) lookups** — finding a specific post by ID requires scanning the array. Normalization gives a single source of truth per entity, O(1) lookups, and surgical updates that touch exactly one entity.

### 2. When would you choose NOT to normalize state?

**Answer**: When data is (a) read-only and never updated after load, (b) never shared between components, (c) too simple to have meaningful relationships, or (d) already flat (e.g., a list of strings). Normalization adds cognitive overhead — you need selectors to reconstruct read shapes. For a settings page with a handful of toggles, normalizing is over-engineering. The test: does this entity appear in more than one place in the UI, or is it mutated independently of its containers?

### 3. Explain the ids + entities shape and why it uses both

**Answer**: `entities` is a dictionary (`Record<string, T>`) for O(1) lookup by ID. `ids` is an ordered string array that serves two purposes: it defines the display order (sorted list, pagination position), and it lets you iterate entities in a predictable sequence without depending on object key order (which is insertion-order in V8 but not guaranteed by spec across environments). When you `setAll`, both are reset. When you `addOne`, both are updated atomically by the adapter. If you only had `entities`, you'd lose ordering; if you only had `ids`, you'd need O(n) scan.

### 4. How does RTK's createEntityAdapter differ from Normalizr?

**Answer**: Normalizr is a normalization-only library — it transforms a nested API response into a flat entity map. It doesn't know about Redux or state management; you wire the output yourself. `createEntityAdapter` is RTK's built-in tool that: generates the initial `{ ids, entities }` state shape, provides typed CRUD reducers (`addOne`, `updateOne`, `removeMany`, etc.), and generates pre-built selectors (`selectAll`, `selectById`, `selectTotal`). In modern Redux projects you typically use both: Normalizr (or manual normalization) to parse API responses, and `createEntityAdapter` to manage the slice state.

### 5. How do you handle many-to-many relationships in normalized state?

**Answer**: Store an array of foreign keys on one side (or both for symmetric relationships). Example: a `Post` has `tagIds: string[]` and a `Tag` has `postIds: string[]` (or you derive one from the other). When reading, use a selector that maps over the ID array and joins from the entities dict. For very large fan-out relationships (a user following thousands of others), consider a separate join-table-style slice: `follows: { followerIds: Record<userId, userId[]> }` to avoid bloating the user entity. The selector is the join layer; it should be memoized with `createSelector`.
