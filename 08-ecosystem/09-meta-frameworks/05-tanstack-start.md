# TanStack Start

## What It Is

TanStack Start is a full-stack React framework built on TanStack Router and the Vinxi bundler. It provides server functions, type-safe routing, and file-based route definitions — competing with Next.js and Remix while maintaining TanStack Router's end-to-end type safety.

---

## Key Differentiators

- **TanStack Router integration** — full type-safe routing (typed params, search params)
- **Server functions** — type-safe server-side operations called from client
- **Framework-agnostic core** — same patterns could extend to other frameworks
- **Vinxi bundler** — flexible, framework-agnostic build system

---

## Setup (Early 2025)

```bash
npx create-tsrouter-app@latest my-app --framework start
```

```ts
// app.config.ts
import { defineConfig } from '@tanstack/start/config';

export default defineConfig({
  tsr: {
    appDirectory: './app',
  },
});
```

---

## File-Based Routing

```
app/
├── routes/
│   ├── __root.tsx         → root layout
│   ├── index.tsx          → /
│   ├── users/
│   │   ├── index.tsx      → /users
│   │   └── $userId.tsx    → /users/:userId
│   └── api/
│       └── users.ts       → /api/users (server route)
```

```tsx
// app/routes/users/$userId.tsx
import { createFileRoute } from '@tanstack/react-router';
import { fetchUser } from '../server/users';

export const Route = createFileRoute('/users/$userId')({
  loader: async ({ params }) => {
    return await fetchUser(params.userId);
  },
  component: UserDetail,
});

function UserDetail() {
  const user = Route.useLoaderData();
  const { userId } = Route.useParams();
  return <div>{user.name}</div>;
}
```

---

## Server Functions

The key innovation — type-safe functions that always run on the server, callable from client:

```ts
// app/server/users.ts
import { createServerFn } from '@tanstack/start';
import { z } from 'zod';

export const getUser = createServerFn({ method: 'GET' })
  .validator(z.object({ userId: z.string() }))
  .handler(async ({ data }) => {
    // Always runs on server — can access DB directly
    const user = await db.users.findById(data.userId);
    if (!user) throw new Error('User not found');
    return user;
  });

export const createUser = createServerFn({ method: 'POST' })
  .validator(z.object({ name: z.string(), email: z.string().email() }))
  .handler(async ({ data }) => {
    return await db.users.create(data);
  });
```

```tsx
// Client-side usage — fully typed, compiler knows it's a server call
function UserList() {
  const createUser = useMutation({
    mutationFn: (data: { name: string; email: string }) => createUser({ data }),
  });

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      const form = new FormData(e.target as HTMLFormElement);
      createUser.mutate({
        name: form.get('name') as string,
        email: form.get('email') as string,
      });
    }}>
      ...
    </form>
  );
}
```

---

## Comparison with Next.js and Remix

| | TanStack Start | Next.js App Router | Remix/RR7 |
|---|---|---|---|
| Type safety | End-to-end (strongest) | Good | Good |
| Search params | Fully typed schemas | String-based | String-based |
| Server functions | `createServerFn` | Server Actions | Actions |
| File-based routing | Yes | Yes | Yes |
| Data loading | Typed loaders | RSC async | Loaders |
| Maturity | Beta/early release | Production | Production |
| Ecosystem | Small, growing | Huge | Large |

---

## Status (2025)

TanStack Start is in active development. It has reached stability for production use at smaller scale, but:
- Ecosystem is smaller than Next.js/Remix
- Documentation is still being expanded
- Some features are still being stabilized
- Best suited for: teams already using TanStack Router who want a full-stack option

---

## When to Choose TanStack Start

- You're already using TanStack Router and want server capabilities
- Type safety of search params and route params is a priority
- You're building a new project and want to try the cutting edge
- Your team prefers explicit, typed APIs over convention-based magic

---

## Common Interview Questions

**Q: How do server functions differ from Next.js Server Actions?**
Both run exclusively on the server. TanStack Start server functions are explicitly typed end-to-end using Zod validation — the client knows the exact input/output types. Server Actions use form-native serialization which is less type-safe for complex data.

**Q: What is Vinxi?**
Vinxi is a framework-agnostic build system that TanStack Start (and Remix/React Router 7) is built on. It provides the development server, build pipeline, and SSR infrastructure while being framework-neutral.

**Q: Is TanStack Start production ready?**
As of early 2025, it has exited the beta phase and is suitable for production. The TanStack team uses it in their own products. Evaluate ecosystem maturity for your specific dependencies.
