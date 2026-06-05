# Apollo Angular

## The Idea

**In plain English:** Apollo Angular is a tool that lets your Angular app ask a server for exactly the data it needs — no more, no less — and automatically remembers the answers so it doesn't have to ask again. GraphQL is the special language used to describe those precise data requests.

**Real-world analogy:** Imagine a personal shopper at a grocery store. You hand them a list that says exactly what you want (three apples, one loaf of bread), they go get it, and they keep a copy of your list so next time they can just hand you the bag without going to the store again.

- The shopping list = the GraphQL query (specifying exactly what data you want)
- The personal shopper = Apollo Angular (handles fetching and communicating with the server)
- The copy they keep = the Apollo cache (stored results so the app doesn't re-fetch unnecessarily)

---

## Overview

Apollo Angular is the official Angular integration for Apollo Client — a full-featured GraphQL client that handles caching, state management, optimistic updates, and real-time subscriptions. It bridges Apollo's battle-tested caching layer with Angular's dependency injection, RxJS observables, and signals model.

## Installation and Setup

```bash
ng add apollo-angular
# or manually:
npm install @apollo/client apollo-angular graphql
```

### Basic Configuration

```typescript
// graphql.module.ts (module-based)
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { ApolloModule, APOLLO_OPTIONS } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { InMemoryCache } from '@apollo/client/core';

@NgModule({
  imports: [ApolloModule, HttpClientModule],
  providers: [
    {
      provide: APOLLO_OPTIONS,
      useFactory: (httpLink: HttpLink) => ({
        cache: new InMemoryCache(),
        link: httpLink.create({ uri: '/graphql' }),
      }),
      deps: [HttpLink],
    },
  ],
})
export class GraphQLModule {}
```

### Standalone / `app.config.ts` Setup

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideApollo } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { InMemoryCache } from '@apollo/client/core';
import { inject } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideApollo(() => {
      const httpLink = inject(HttpLink);
      return {
        cache: new InMemoryCache(),
        link: httpLink.create({ uri: '/graphql' }),
        defaultOptions: {
          watchQuery: { fetchPolicy: 'cache-and-network' },
          query: { fetchPolicy: 'network-only' },
        },
      };
    }),
  ],
};
```

## Core Operations

### 1. Queries — `Apollo.watchQuery` and `Apollo.query`

```typescript
import { Component, inject } from '@angular/core';
import { Apollo, gql } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { AsyncPipe, NgFor, NgIf } from '@angular/common';

interface User {
  id: string;
  name: string;
  email: string;
}

const GET_USERS = gql`
  query GetUsers {
    users {
      id
      name
      email
    }
  }
`;

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [AsyncPipe, NgFor, NgIf],
  template: `
    <ng-container *ngIf="users$ | async as result">
      <div *ngIf="result.loading">Loading...</div>
      <div *ngIf="result.error">Error: {{ result.error.message }}</div>
      <ul *ngIf="result.data">
        <li *ngFor="let user of result.data.users">
          {{ user.name }} — {{ user.email }}
        </li>
      </ul>
    </ng-container>
  `,
})
export class UserListComponent {
  private apollo = inject(Apollo);

  // watchQuery: re-emits whenever cache updates
  users$ = this.apollo.watchQuery<{ users: User[] }>({
    query: GET_USERS,
  }).valueChanges;

  // One-shot query (does not watch cache)
  fetchOnce(): void {
    this.apollo
      .query<{ users: User[] }>({ query: GET_USERS })
      .subscribe(({ data }) => console.log(data.users));
  }
}
```

### 2. Query with Variables

```typescript
const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
      posts {
        id
        title
      }
    }
  }
`;

@Component({
  selector: 'app-user-detail',
  standalone: true,
  imports: [AsyncPipe, NgIf, NgFor],
  template: `
    <ng-container *ngIf="user$ | async as { data, loading, error }">
      <div *ngIf="loading">Loading...</div>
      <div *ngIf="data?.user as user">
        <h2>{{ user.name }}</h2>
        <p>{{ user.email }}</p>
        <ul>
          <li *ngFor="let post of user.posts">{{ post.title }}</li>
        </ul>
      </div>
    </ng-container>
  `,
})
export class UserDetailComponent {
  private apollo = inject(Apollo);
  userId = input.required<string>();

  user$ = toObservable(this.userId).pipe(
    switchMap((id) =>
      this.apollo.watchQuery<{ user: User }>({
        query: GET_USER,
        variables: { id },
      }).valueChanges
    )
  );
}
```

### 3. Mutations

```typescript
const CREATE_USER = gql`
  mutation CreateUser($name: String!, $email: String!) {
    createUser(name: $name, email: $email) {
      id
      name
      email
    }
  }
`;

@Component({
  selector: 'app-create-user',
  standalone: true,
  template: `
    <form (submit)="submit($event)">
      <input [(ngModel)]="name" name="name" placeholder="Name" />
      <input [(ngModel)]="email" name="email" placeholder="Email" />
      <button type="submit" [disabled]="loading">
        {{ loading ? 'Creating…' : 'Create' }}
      </button>
      <div *ngIf="error">{{ error.message }}</div>
    </form>
  `,
})
export class CreateUserComponent {
  private apollo = inject(Apollo);

  name = '';
  email = '';
  loading = false;
  error: any = null;

  submit(event: Event): void {
    event.preventDefault();
    this.loading = true;

    this.apollo
      .mutate<{ createUser: User }>({
        mutation: CREATE_USER,
        variables: { name: this.name, email: this.email },
        // Update the cache after mutation
        update: (cache, { data }) => {
          const existing = cache.readQuery<{ users: User[] }>({
            query: GET_USERS,
          });
          if (existing && data) {
            cache.writeQuery({
              query: GET_USERS,
              data: { users: [...existing.users, data.createUser] },
            });
          }
        },
      })
      .subscribe({
        next: ({ data }) => {
          this.loading = false;
          console.log('Created:', data?.createUser);
        },
        error: (err) => {
          this.loading = false;
          this.error = err;
        },
      });
  }
}
```

### 4. Subscriptions

```typescript
// Setup: add WebSocket link alongside HTTP link
import { split } from '@apollo/client/core';
import { getMainDefinition } from '@apollo/client/utilities';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';

export const appConfig: ApplicationConfig = {
  providers: [
    provideApollo(() => {
      const httpLink = inject(HttpLink);
      const wsLink = new GraphQLWsLink(
        createClient({ url: 'ws://localhost:4000/graphql' })
      );

      const splitLink = split(
        ({ query }) => {
          const def = getMainDefinition(query);
          return (
            def.kind === 'OperationDefinition' &&
            def.operation === 'subscription'
          );
        },
        wsLink,
        httpLink.create({ uri: '/graphql' })
      );

      return { cache: new InMemoryCache(), link: splitLink };
    }),
  ],
};

// Usage
const MESSAGE_ADDED = gql`
  subscription OnMessageAdded($roomId: ID!) {
    messageAdded(roomId: $roomId) {
      id
      content
      author { name }
    }
  }
`;

@Component({ selector: 'app-chat', standalone: true, template: `...` })
export class ChatComponent implements OnInit, OnDestroy {
  private apollo = inject(Apollo);
  private sub?: Subscription;

  ngOnInit(): void {
    this.sub = this.apollo
      .subscribe<{ messageAdded: Message }>({
        query: MESSAGE_ADDED,
        variables: { roomId: '1' },
      })
      .subscribe(({ data }) => {
        console.log('New message:', data?.messageAdded);
      });
  }

  ngOnDestroy(): void {
    this.sub?.unsubscribe();
  }
}
```

## Cache Management

### Reading and Writing the Cache Directly

```typescript
@Injectable({ providedIn: 'root' })
export class UserCacheService {
  private apollo = inject(Apollo);

  getUser(id: string): User | null {
    return this.apollo.client.readFragment<User>({
      id: `User:${id}`,
      fragment: gql`
        fragment UserFields on User {
          id
          name
          email
        }
      `,
    });
  }

  updateUserName(id: string, name: string): void {
    this.apollo.client.writeFragment({
      id: `User:${id}`,
      fragment: gql`
        fragment UpdateUser on User {
          name
        }
      `,
      data: { name },
    });
  }

  evictUser(id: string): void {
    this.apollo.client.cache.evict({ id: `User:${id}` });
    this.apollo.client.cache.gc();
  }
}
```

### Fetch Policies

```typescript
// network-only — always hit network, update cache
this.apollo.query({ query: GET_USERS, fetchPolicy: 'network-only' });

// cache-first (default) — serve cache, skip network if fresh
this.apollo.query({ query: GET_USERS, fetchPolicy: 'cache-first' });

// cache-and-network — emit cache immediately, then re-emit network result
this.apollo.watchQuery({ query: GET_USERS, fetchPolicy: 'cache-and-network' });

// no-cache — skip cache entirely, don't store result
this.apollo.query({ query: GET_USERS, fetchPolicy: 'no-cache' });
```

### Optimistic Updates

```typescript
this.apollo.mutate({
  mutation: TOGGLE_LIKE,
  variables: { postId },
  // Instant UI update before network response
  optimisticResponse: {
    __typename: 'Mutation',
    toggleLike: {
      __typename: 'Post',
      id: postId,
      liked: !currentlyLiked,
      likeCount: currentlyLiked ? likeCount - 1 : likeCount + 1,
    },
  },
}).subscribe();
```

## Authentication with Apollo Link

```typescript
import { setContext } from '@apollo/client/link/context';
import { onError } from '@apollo/client/link/error';
import { from } from '@apollo/client/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideApollo(() => {
      const httpLink = inject(HttpLink);
      const authService = inject(AuthService);

      const authLink = setContext((_, { headers }) => ({
        headers: {
          ...headers,
          authorization: authService.token
            ? `Bearer ${authService.token}`
            : '',
        },
      }));

      const errorLink = onError(({ graphQLErrors, networkError }) => {
        graphQLErrors?.forEach(({ message, extensions }) => {
          if (extensions?.['code'] === 'UNAUTHENTICATED') {
            authService.logout();
          }
        });
        if (networkError) console.error('Network error', networkError);
      });

      return {
        cache: new InMemoryCache(),
        link: from([errorLink, authLink, httpLink.create({ uri: '/graphql' })]),
      };
    }),
  ],
};
```

## Typed GraphQL with Code Generation

```bash
npm install -D @graphql-codegen/cli @graphql-codegen/typescript \
  @graphql-codegen/typescript-operations @graphql-codegen/typescript-apollo-angular
```

```yaml
# codegen.yml
schema: http://localhost:4000/graphql
documents: 'src/**/*.graphql'
generates:
  src/generated/graphql.ts:
    plugins:
      - typescript
      - typescript-operations
      - typescript-apollo-angular
```

```typescript
// After running `graphql-codegen`:
import { GetUsersGQL, CreateUserGQL } from '../generated/graphql';

@Component({ selector: 'app-users', standalone: true, template: `...` })
export class UsersComponent {
  private getUsersGql = inject(GetUsersGQL);
  private createUserGql = inject(CreateUserGQL);

  // Fully typed — no manual type annotations needed
  users$ = this.getUsersGql.watch().valueChanges.pipe(
    map(({ data }) => data.users)
  );

  create(name: string, email: string): void {
    this.createUserGql
      .mutate({ name, email })
      .subscribe(({ data }) => console.log(data?.createUser));
  }
}
```

## Comparison: Apollo vs TanStack Query for Angular

| Concern | Apollo Angular | TanStack Query Angular |
|---|---|---|
| Protocol | GraphQL only | REST / any async |
| Caching | Normalized by type+id | Query-key based |
| Schema awareness | Yes (type system) | No |
| Subscriptions | Built-in via WS link | Not built-in |
| Bundle size | Larger (~35 kB) | Smaller (~15 kB) |
| Learning curve | Higher | Lower |
| Code generation | First-class support | N/A |

**Use Apollo when:** Your API is GraphQL and you want normalized caching, subscriptions, and codegen.  
**Use TanStack Query when:** Your API is REST or you want a lighter, simpler caching layer.

## Common Pitfalls

1. **Not normalizing the cache** — Ensure every object returns `__typename` and `id` so Apollo can merge updates correctly.
2. **Over-fetching with `no-cache`** — Defeats the purpose of Apollo; use sparingly.
3. **Forgetting to update the cache after mutations** — Use `update`, `refetchQueries`, or optimistic responses.
4. **Multiple Apollo clients** — Stick to one unless you genuinely have two GraphQL endpoints.
5. **Memory leaks with watchQuery** — Always unsubscribe or use the `async` pipe.

## Interview Questions

**Q: What makes Apollo's cache different from a standard HTTP cache?**  
A: Apollo's `InMemoryCache` is a normalized, object-level cache keyed by `__typename + id`. When a mutation returns an updated `User:42`, every query that referenced that object automatically re-renders — you don't need to invalidate by URL.

**Q: When would you choose `watchQuery` over `query`?**  
A: `watchQuery` returns a live observable that re-emits whenever the cached data changes. Use it for UI that must react to cache updates (e.g., after a mutation). Use `query` (one-shot) for background fetches or when you only need the current snapshot.

**Q: How do optimistic responses work?**  
A: Apollo writes the `optimisticResponse` to a temporary layer over the real cache, so the UI updates instantly. When the real response arrives, the temporary layer is discarded and the actual result is written. If the mutation errors, the temporary layer is rolled back automatically.

**Q: What is the `update` function on a mutation used for?**  
A: It gives you imperative access to the cache after the server responds. Use it to insert the new object into cached list queries that Apollo can't update automatically (because the list query wasn't watching that specific entity).
