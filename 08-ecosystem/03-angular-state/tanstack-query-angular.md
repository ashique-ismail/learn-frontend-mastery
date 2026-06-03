# TanStack Query Angular

## Overview

TanStack Query (formerly React Query) for Angular is a powerful data synchronization library that makes fetching, caching, synchronizing and updating server state in Angular applications simple and efficient. It provides automatic background refetching, caching, request deduplication, optimistic updates, and much more out of the box, eliminating the need to manually manage loading/error states and cache invalidation.

The Angular adapter brings the battle-tested TanStack Query patterns to Angular applications with full TypeScript support, Angular Signals integration, and seamless RxJS interoperability.

## Installation and Setup

```bash
npm install @tanstack/angular-query-experimental
# or
yarn add @tanstack/angular-query-experimental
```

### Basic Setup

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideQueryClient } from '@tanstack/angular-query-experimental';

export const appConfig: ApplicationConfig = {
  providers: [
    provideQueryClient({
      defaultOptions: {
        queries: {
          staleTime: 1000 * 60 * 5, // 5 minutes
          cacheTime: 1000 * 60 * 30, // 30 minutes
          refetchOnWindowFocus: true,
          refetchOnReconnect: true,
          retry: 3
        }
      }
    })
  ]
};

// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig);
```

### With Module-based Apps

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { QueryClientModule } from '@tanstack/angular-query-experimental';

@NgModule({
  imports: [
    QueryClientModule.forRoot({
      defaultOptions: {
        queries: {
          staleTime: 1000 * 60 * 5,
          retry: 2
        }
      }
    })
  ]
})
export class AppModule {}
```

## Core Concepts

### 1. Queries - Fetching Data

```typescript
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { injectQuery } from '@tanstack/angular-query-experimental';
import { HttpClient } from '@angular/common/http';
import { lastValueFrom } from 'rxjs';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div *ngIf="query.isLoading()">Loading users...</div>
    
    <div *ngIf="query.isError()">
      Error: {{ query.error()?.message }}
      <button (click)="query.refetch()">Retry</button>
    </div>

    <div *ngIf="query.isSuccess()">
      <div *ngFor="let user of query.data()">
        <h3>{{ user.name }}</h3>
        <p>{{ user.email }}</p>
      </div>
    </div>

    <div *ngIf="query.isFetching() && !query.isLoading()">
      Updating...
    </div>
  `
})
export class UserListComponent {
  private http = inject(HttpClient);

  // Basic query
  query = injectQuery(() => ({
    queryKey: ['users'],
    queryFn: () => lastValueFrom(
      this.http.get<User[]>('/api/users')
    )
  }));
}
```

### 2. Query Keys and Caching

```typescript
@Component({
  selector: 'app-user-detail',
  template: `
    <div *ngIf="userQuery.data() as user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
      
      <h3>Posts</h3>
      <div *ngIf="postsQuery.data() as posts">
        <div *ngFor="let post of posts">
          <h4>{{ post.title }}</h4>
          <p>{{ post.content }}</p>
        </div>
      </div>
    </div>
  `
})
export class UserDetailComponent {
  private http = inject(HttpClient);
  
  // Get user ID from route or input
  userId = signal(1);

  // Query with dynamic key
  userQuery = injectQuery(() => ({
    queryKey: ['user', this.userId()],
    queryFn: () => lastValueFrom(
      this.http.get<User>(`/api/users/${this.userId()}`)
    )
  }));

  // Dependent query - only runs when userQuery succeeds
  postsQuery = injectQuery(() => ({
    queryKey: ['posts', this.userId()],
    queryFn: () => lastValueFrom(
      this.http.get<Post[]>(`/api/users/${this.userId()}/posts`)
    ),
    enabled: this.userQuery.isSuccess()
  }));
}
```

### 3. Mutations - Modifying Data

```typescript
import { injectMutation, injectQueryClient } from '@tanstack/angular-query-experimental';

interface CreateUserDto {
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-form',
  template: `
    <form (submit)="handleSubmit($event)">
      <input
        type="text"
        [(ngModel)]="name"
        placeholder="Name"
      />
      <input
        type="email"
        [(ngModel)]="email"
        placeholder="Email"
      />
      
      <button
        type="submit"
        [disabled]="createMutation.isPending()"
      >
        {{ createMutation.isPending() ? 'Creating...' : 'Create User' }}
      </button>
    </form>

    <div *ngIf="createMutation.isError()">
      Error: {{ createMutation.error()?.message }}
    </div>

    <div *ngIf="createMutation.isSuccess()">
      User created successfully!
    </div>
  `
})
export class UserFormComponent {
  private http = inject(HttpClient);
  private queryClient = injectQueryClient();

  name = '';
  email = '';

  createMutation = injectMutation(() => ({
    mutationFn: (data: CreateUserDto) => lastValueFrom(
      this.http.post<User>('/api/users', data)
    ),
    onSuccess: (newUser) => {
      // Invalidate and refetch users query
      this.queryClient.invalidateQueries({ queryKey: ['users'] });
      
      // Or optimistically update cache
      this.queryClient.setQueryData<User[]>(['users'], (old) => {
        return old ? [...old, newUser] : [newUser];
      });
      
      // Reset form
      this.name = '';
      this.email = '';
    },
    onError: (error) => {
      console.error('Failed to create user:', error);
    }
  }));

  handleSubmit(event: Event): void {
    event.preventDefault();
    this.createMutation.mutate({
      name: this.name,
      email: this.email
    });
  }
}
```

### 4. Query Invalidation

```typescript
@Component({
  selector: 'app-user-actions',
  template: `
    <button (click)="updateUser()">Update User</button>
    <button (click)="deleteUser()">Delete User</button>
    <button (click)="refreshAll()">Refresh All</button>
  `
})
export class UserActionsComponent {
  private http = inject(HttpClient);
  private queryClient = injectQueryClient();
  
  userId = signal(1);

  updateMutation = injectMutation(() => ({
    mutationFn: (updates: Partial<User>) => lastValueFrom(
      this.http.patch<User>(`/api/users/${this.userId()}`, updates)
    ),
    onSuccess: () => {
      // Invalidate specific user
      this.queryClient.invalidateQueries({
        queryKey: ['user', this.userId()]
      });
      
      // Invalidate all users queries
      this.queryClient.invalidateQueries({
        queryKey: ['users']
      });
    }
  }));

  deleteMutation = injectMutation(() => ({
    mutationFn: (id: number) => lastValueFrom(
      this.http.delete(`/api/users/${id}`)
    ),
    onSuccess: () => {
      // Remove from cache
      this.queryClient.removeQueries({
        queryKey: ['user', this.userId()]
      });
      
      // Invalidate list
      this.queryClient.invalidateQueries({
        queryKey: ['users']
      });
    }
  }));

  updateUser(): void {
    this.updateMutation.mutate({ name: 'Updated Name' });
  }

  deleteUser(): void {
    this.deleteMutation.mutate(this.userId());
  }

  refreshAll(): void {
    // Invalidate all queries
    this.queryClient.invalidateQueries();
  }
}
```

### 5. Optimistic Updates

```typescript
interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

@Component({
  selector: 'app-todo-list',
  template: `
    <div *ngFor="let todo of todosQuery.data()">
      <input
        type="checkbox"
        [checked]="todo.completed"
        (change)="toggleTodo(todo)"
      />
      <span [class.completed]="todo.completed">
        {{ todo.title }}
      </span>
      <button (click)="deleteTodo(todo.id)">Delete</button>
    </div>
  `
})
export class TodoListComponent {
  private http = inject(HttpClient);
  private queryClient = injectQueryClient();

  todosQuery = injectQuery(() => ({
    queryKey: ['todos'],
    queryFn: () => lastValueFrom(
      this.http.get<Todo[]>('/api/todos')
    )
  }));

  toggleMutation = injectMutation(() => ({
    mutationFn: (todo: Todo) => lastValueFrom(
      this.http.patch<Todo>(`/api/todos/${todo.id}`, {
        completed: !todo.completed
      })
    ),
    // Optimistic update
    onMutate: async (todo) => {
      // Cancel outgoing refetches
      await this.queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot previous value
      const previousTodos = this.queryClient.getQueryData<Todo[]>(['todos']);

      // Optimistically update cache
      this.queryClient.setQueryData<Todo[]>(['todos'], (old) => {
        if (!old) return [];
        return old.map(t =>
          t.id === todo.id ? { ...t, completed: !t.completed } : t
        );
      });

      // Return context with snapshot
      return { previousTodos };
    },
    // Rollback on error
    onError: (err, todo, context) => {
      this.queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    // Refetch after success or error
    onSettled: () => {
      this.queryClient.invalidateQueries({ queryKey: ['todos'] });
    }
  }));

  deleteMutation = injectMutation(() => ({
    mutationFn: (id: number) => lastValueFrom(
      this.http.delete(`/api/todos/${id}`)
    ),
    onMutate: async (id) => {
      await this.queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = this.queryClient.getQueryData<Todo[]>(['todos']);

      this.queryClient.setQueryData<Todo[]>(['todos'], (old) => {
        return old ? old.filter(t => t.id !== id) : [];
      });

      return { previousTodos };
    },
    onError: (err, id, context) => {
      this.queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    onSettled: () => {
      this.queryClient.invalidateQueries({ queryKey: ['todos'] });
    }
  }));

  toggleTodo(todo: Todo): void {
    this.toggleMutation.mutate(todo);
  }

  deleteTodo(id: number): void {
    this.deleteMutation.mutate(id);
  }
}
```

### 6. Infinite Queries (Pagination)

```typescript
import { injectInfiniteQuery } from '@tanstack/angular-query-experimental';

interface PostsResponse {
  posts: Post[];
  nextCursor?: number;
  hasMore: boolean;
}

@Component({
  selector: 'app-infinite-posts',
  template: `
    <div *ngFor="let page of query.data()?.pages">
      <div *ngFor="let post of page.posts">
        <h3>{{ post.title }}</h3>
        <p>{{ post.content }}</p>
      </div>
    </div>

    <button
      (click)="query.fetchNextPage()"
      [disabled]="!query.hasNextPage() || query.isFetchingNextPage()"
    >
      {{
        query.isFetchingNextPage()
          ? 'Loading more...'
          : query.hasNextPage()
          ? 'Load More'
          : 'Nothing more to load'
      }}
    </button>

    <div *ngIf="query.isFetching() && !query.isFetchingNextPage()">
      Background refreshing...
    </div>
  `
})
export class InfinitePostsComponent {
  private http = inject(HttpClient);

  query = injectInfiniteQuery(() => ({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam = 0 }) => lastValueFrom(
      this.http.get<PostsResponse>(`/api/posts?cursor=${pageParam}`)
    ),
    getNextPageParam: (lastPage) => {
      return lastPage.hasMore ? lastPage.nextCursor : undefined;
    },
    initialPageParam: 0
  }));
}
```

### 7. Query Options and Configuration

```typescript
@Component({
  selector: 'app-advanced-query',
  template: `...`
})
export class AdvancedQueryComponent {
  private http = inject(HttpClient);

  // Query with full configuration
  query = injectQuery(() => ({
    queryKey: ['data'],
    queryFn: () => lastValueFrom(this.http.get('/api/data')),
    
    // Caching
    staleTime: 1000 * 60 * 5, // Consider data fresh for 5 minutes
    cacheTime: 1000 * 60 * 30, // Keep in cache for 30 minutes
    
    // Refetching
    refetchOnWindowFocus: true,
    refetchOnReconnect: true,
    refetchOnMount: true,
    refetchInterval: false, // Or number for polling
    refetchIntervalInBackground: false,
    
    // Retries
    retry: 3,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
    
    // Conditional fetching
    enabled: true, // Or computed(() => someCondition)
    
    // Callbacks
    onSuccess: (data) => {
      console.log('Data loaded:', data);
    },
    onError: (error) => {
      console.error('Error loading data:', error);
    },
    onSettled: (data, error) => {
      console.log('Query settled');
    },
    
    // Select/transform data
    select: (data) => {
      return data.map(item => ({
        ...item,
        formatted: formatItem(item)
      }));
    },
    
    // Placeholders
    placeholderData: [], // Show while loading
    
    // Keep previous data while refetching
    keepPreviousData: true
  }));
}
```

## Advanced Patterns

### Prefetching Data

```typescript
@Component({
  selector: 'app-prefetch-example',
  template: `
    <div (mouseenter)="prefetchUser(123)">
      Hover to prefetch user 123
    </div>
  `
})
export class PrefetchExampleComponent {
  private http = inject(HttpClient);
  private queryClient = injectQueryClient();

  prefetchUser(id: number): void {
    this.queryClient.prefetchQuery({
      queryKey: ['user', id],
      queryFn: () => lastValueFrom(
        this.http.get<User>(`/api/users/${id}`)
      ),
      staleTime: 1000 * 60 * 5
    });
  }

  // Prefetch on navigation
  onNavigate(id: number): void {
    this.queryClient.prefetchQuery({
      queryKey: ['user', id],
      queryFn: () => lastValueFrom(
        this.http.get<User>(`/api/users/${id}`)
      )
    });
    
    // Then navigate
    // this.router.navigate(['/user', id]);
  }
}
```

### Query Cancellation

```typescript
@Component({
  selector: 'app-search',
  template: `
    <input
      type="text"
      [(ngModel)]="searchTerm"
      (input)="onSearchChange()"
    />
    
    <div *ngIf="searchQuery.data() as results">
      <div *ngFor="let result of results">
        {{ result.name }}
      </div>
    </div>
  `
})
export class SearchComponent {
  private http = inject(HttpClient);
  
  searchTerm = signal('');

  searchQuery = injectQuery(() => ({
    queryKey: ['search', this.searchTerm()],
    queryFn: ({ signal }) => {
      // Pass AbortSignal to HTTP request
      return lastValueFrom(
        this.http.get<SearchResult[]>('/api/search', {
          params: { q: this.searchTerm() },
          context: new HttpContext().set(ABORT_SIGNAL, signal)
        })
      );
    },
    enabled: this.searchTerm().length > 2,
    staleTime: 0 // Always refetch
  }));

  onSearchChange(): void {
    // Query will automatically cancel previous request
    // when searchTerm changes
  }
}
```

### Parallel Queries

```typescript
@Component({
  selector: 'app-dashboard',
  template: `
    <div *ngIf="usersQuery.data() as users">
      Users: {{ users.length }}
    </div>
    
    <div *ngIf="postsQuery.data() as posts">
      Posts: {{ posts.length }}
    </div>
    
    <div *ngIf="commentsQuery.data() as comments">
      Comments: {{ comments.length }}
    </div>

    <div *ngIf="isLoading">Loading dashboard...</div>
  `
})
export class DashboardComponent {
  private http = inject(HttpClient);

  // Multiple parallel queries
  usersQuery = injectQuery(() => ({
    queryKey: ['users'],
    queryFn: () => lastValueFrom(this.http.get<User[]>('/api/users'))
  }));

  postsQuery = injectQuery(() => ({
    queryKey: ['posts'],
    queryFn: () => lastValueFrom(this.http.get<Post[]>('/api/posts'))
  }));

  commentsQuery = injectQuery(() => ({
    queryKey: ['comments'],
    queryFn: () => lastValueFrom(this.http.get<Comment[]>('/api/comments'))
  }));

  // Computed loading state
  isLoading = computed(() =>
    this.usersQuery.isLoading() ||
    this.postsQuery.isLoading() ||
    this.commentsQuery.isLoading()
  );

  // Or use useQueries for dynamic list
  // queries = injectQueries(() => ({
  //   queries: [
  //     { queryKey: ['users'], queryFn: () => ... },
  //     { queryKey: ['posts'], queryFn: () => ... },
  //     { queryKey: ['comments'], queryFn: () => ... }
  //   ]
  // }));
}
```

### Custom Hooks Pattern

```typescript
// users.queries.ts
export function createUserQueries(http: HttpClient) {
  return {
    useUsers: () => injectQuery(() => ({
      queryKey: ['users'],
      queryFn: () => lastValueFrom(http.get<User[]>('/api/users'))
    })),

    useUser: (id: Signal<number>) => injectQuery(() => ({
      queryKey: ['user', id()],
      queryFn: () => lastValueFrom(http.get<User>(`/api/users/${id()}`))
    })),

    useCreateUser: () => {
      const queryClient = injectQueryClient();
      return injectMutation(() => ({
        mutationFn: (data: CreateUserDto) => lastValueFrom(
          http.post<User>('/api/users', data)
        ),
        onSuccess: () => {
          queryClient.invalidateQueries({ queryKey: ['users'] });
        }
      }));
    },

    useUpdateUser: () => {
      const queryClient = injectQueryClient();
      return injectMutation(() => ({
        mutationFn: ({ id, data }: { id: number; data: Partial<User> }) =>
          lastValueFrom(http.patch<User>(`/api/users/${id}`, data)),
        onSuccess: (_, { id }) => {
          queryClient.invalidateQueries({ queryKey: ['user', id] });
          queryClient.invalidateQueries({ queryKey: ['users'] });
        }
      }));
    }
  };
}

// Component usage
@Component({
  selector: 'app-users',
  template: `...`
})
export class UsersComponent {
  private http = inject(HttpClient);
  private userQueries = createUserQueries(this.http);

  usersQuery = this.userQueries.useUsers();
  createMutation = this.userQueries.useCreateUser();
}
```

### Error Boundaries

```typescript
import { ErrorHandler, Injectable } from '@angular/core';

@Injectable()
export class QueryErrorHandler implements ErrorHandler {
  handleError(error: any): void {
    // Log to error reporting service
    console.error('Query error:', error);
    
    // Show user-friendly message
    // this.notificationService.showError(error.message);
  }
}

// Provide in app config
export const appConfig: ApplicationConfig = {
  providers: [
    provideQueryClient({
      defaultOptions: {
        queries: {
          onError: (error) => {
            console.error('Query error:', error);
          },
          retry: (failureCount, error: any) => {
            // Don't retry on 404
            if (error?.status === 404) return false;
            return failureCount < 3;
          }
        },
        mutations: {
          onError: (error) => {
            console.error('Mutation error:', error);
          }
        }
      }
    }),
    { provide: ErrorHandler, useClass: QueryErrorHandler }
  ]
};
```

## DevTools Integration

```typescript
import { provideQueryDevTools } from '@tanstack/angular-query-devtools-experimental';

export const appConfig: ApplicationConfig = {
  providers: [
    provideQueryClient(),
    provideQueryDevTools({
      initialIsOpen: false,
      position: 'bottom-right'
    })
  ]
};

// Add to app component
@Component({
  selector: 'app-root',
  template: `
    <router-outlet />
    <angular-query-devtools />
  `
})
export class AppComponent {}
```

## Common Mistakes

1. **Not using query keys correctly**: Query keys should be arrays and include all variables that affect the query.

2. **Forgetting to invalidate**: After mutations, invalidate related queries to keep data fresh.

3. **Overusing refetch**: Let TanStack Query handle refetching automatically instead of manual refetch calls.

4. **Not handling loading states**: Always show loading indicators for better UX.

5. **Mutating query data directly**: Use `setQueryData` instead of mutating cached data.

## Best Practices

1. **Consistent query keys**: Use a standardized format like `['resource', id, filters]`
2. **Invalidation strategy**: Invalidate related queries after mutations
3. **Optimistic updates**: For better UX on mutations
4. **Error handling**: Provide user-friendly error messages
5. **Prefetching**: Prefetch data on hover/navigation for instant UX
6. **Stale time configuration**: Set appropriate stale times based on data volatility
7. **Type safety**: Use TypeScript generics for all queries and mutations
8. **Query organization**: Group related queries in separate files

## When to Use TanStack Query

**Use TanStack Query when:**
- Fetching/syncing server data
- Need automatic caching and refetching
- Want optimistic updates
- Building data-heavy applications
- Need request deduplication
- Want pagination/infinite scroll

**Consider alternatives when:**
- Simple HTTP calls without caching needs
- Building fully offline-first apps
- Need GraphQL (use Apollo instead)
- WebSocket-heavy real-time apps

## Interview Questions

1. **Q: What problem does TanStack Query solve?**
   A: TanStack Query eliminates boilerplate for server state management by providing automatic caching, background refetching, deduplication, optimistic updates, and intelligent cache invalidation out of the box.

2. **Q: What's the difference between staleTime and cacheTime?**
   A: staleTime determines how long data is considered fresh (won't refetch). cacheTime determines how long unused data stays in cache before garbage collection. Default: staleTime=0, cacheTime=5min.

3. **Q: How do optimistic updates work?**
   A: In onMutate, snapshot previous data, update cache optimistically, return context. If mutation fails, onError rolls back using the snapshot. onSettled refetches to ensure consistency.

4. **Q: When should you invalidate vs setQueryData?**
   A: Use invalidateQueries to mark stale and trigger refetch (async). Use setQueryData to directly update cache synchronously. Invalidation is safer for ensuring fresh data; setQueryData is faster for optimistic updates.

5. **Q: How does query cancellation work?**
   A: TanStack Query passes an AbortSignal to queryFn. When a query is cancelled (component unmount, key change), the signal aborts. Your HTTP call must support AbortSignal to actually cancel the request.

6. **Q: What are query keys used for?**
   A: Query keys uniquely identify queries for caching and invalidation. They should be arrays including all variables affecting the query. Queries with same key share cache. Invalidation can target specific keys or patterns.

## Key Takeaways

1. TanStack Query manages server state with automatic caching and refetching
2. Query keys uniquely identify queries and enable intelligent caching
3. Mutations modify data with automatic invalidation and optimistic updates
4. staleTime controls freshness, cacheTime controls garbage collection
5. Invalidation strategies keep related queries synchronized
6. Optimistic updates improve perceived performance
7. Infinite queries enable pagination and infinite scroll
8. Prefetching eliminates loading states for better UX
9. Angular Signals integration provides reactive state
10. Query cancellation prevents wasted network requests

## Resources

- [TanStack Query Docs](https://tanstack.com/query/latest)
- [Angular Query Adapter](https://tanstack.com/query/latest/docs/framework/angular/overview)
- [Query Key Best Practices](https://tkdodo.eu/blog/effective-react-query-keys)
- [Optimistic Updates Guide](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates)
- [TanStack Query DevTools](https://tanstack.com/query/latest/docs/framework/react/devtools)
