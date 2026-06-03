# RTK Query

## Overview

RTK Query is the data fetching and caching solution built into Redux Toolkit. It generates React hooks automatically and integrates deeply with the Redux DevTools. Part of `@reduxjs/toolkit`.

---

## Setup

```ts
// store/api.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const api = createApi({
  reducerPath: 'api',        // Redux store key
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['User', 'Post'], // cache invalidation tags
  endpoints: (builder) => ({
    getUsers: builder.query<User[], void>({
      query: () => '/users',
      providesTags: ['User'],
    }),
    getUser: builder.query<User, string>({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }],
    }),
    createUser: builder.mutation<User, Partial<User>>({
      query: (body) => ({ url: '/users', method: 'POST', body }),
      invalidatesTags: ['User'], // invalidates all User cache entries
    }),
    updateUser: builder.mutation<User, { id: string } & Partial<User>>({
      query: ({ id, ...body }) => ({ url: `/users/${id}`, method: 'PATCH', body }),
      invalidatesTags: (result, error, { id }) => [{ type: 'User', id }],
    }),
  }),
});

// Auto-generated hooks (naming: use{EndpointName}Query/Mutation)
export const {
  useGetUsersQuery,
  useGetUserQuery,
  useCreateUserMutation,
  useUpdateUserMutation,
} = api;

// store.ts
import { configureStore } from '@reduxjs/toolkit';

export const store = configureStore({
  reducer: { [api.reducerPath]: api.reducer },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(api.middleware),
});
```

---

## Usage

```tsx
function UserList() {
  const { data: users, isLoading, isError, refetch } = useGetUsersQuery();

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage />;

  return (
    <ul>
      {users?.map(user => <UserItem key={user.id} user={user} />)}
    </ul>
  );
}

function CreateUserForm() {
  const [createUser, { isLoading, error }] = useCreateUserMutation();

  async function handleSubmit(data: Partial<User>) {
    try {
      await createUser(data).unwrap(); // unwrap() throws on error
      toast.success('User created');
    } catch (err) {
      toast.error('Failed to create user');
    }
  }
}
```

---

## Cache Invalidation with Tags

```ts
// Tag pattern: { type: 'User', id: '123' } for per-entity invalidation

getUser: builder.query({
  providesTags: (result, error, id) => [
    { type: 'User', id },  // specific user
    { type: 'User', id: 'LIST' }, // signal that this user appears in lists
  ],
}),

deleteUser: builder.mutation({
  query: (id) => ({ url: `/users/${id}`, method: 'DELETE' }),
  invalidatesTags: (result, error, id) => [
    { type: 'User', id },           // invalidate this user
    { type: 'User', id: 'LIST' },   // invalidate lists containing users
  ],
}),
```

---

## Optimistic Updates

```ts
updateUser: builder.mutation<User, { id: string } & Partial<User>>({
  query: ({ id, ...patch }) => ({ url: `/users/${id}`, method: 'PATCH', body: patch }),

  async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
    // Optimistically update the cache
    const patchResult = dispatch(
      api.util.updateQueryData('getUser', id, (draft) => {
        Object.assign(draft, patch);
      })
    );

    try {
      await queryFulfilled;
    } catch {
      patchResult.undo(); // revert on failure
    }
  },
}),
```

---

## Polling

```tsx
function LiveStats() {
  // Automatically refetch every 30 seconds
  const { data } = useGetStatsQuery(undefined, { pollingInterval: 30_000 });
  return <Stats data={data} />;
}
```

---

## Comparison with TanStack Query

| | RTK Query | TanStack Query |
|---|---|---|
| Redux integration | Built-in | External (zustand/jotai/etc.) |
| Mutations | Defined in API slice | `useMutation` hook |
| Cache invalidation | Tag-based | Manual queryClient.invalidate |
| DevTools | Redux DevTools | TanStack Query DevTools |
| Setup boilerplate | More (store, reducerPath) | Less |
| Framework support | React only | React, Vue, Angular, Svelte |
| Bundle size | ~11 KB (on top of RTK) | ~12 KB standalone |

RTK Query is the right choice when you're already using Redux. TanStack Query is more popular for new projects without Redux.

---

## Common Interview Questions

**Q: How do RTK Query tags work for invalidation?**
Endpoints that `providesTags` declare what data they return. Mutations that `invalidatesTags` declare what data they change. When a mutation completes, RTK Query refetches all active queries whose provided tags overlap with the invalidated tags.

**Q: What does `.unwrap()` do?**
By default, RTK Query mutations return `{ data, error }`. `.unwrap()` converts this to a regular Promise — it resolves with the data on success or throws the error on failure, enabling try/catch usage.
