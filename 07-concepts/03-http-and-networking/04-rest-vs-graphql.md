# REST vs GraphQL

## The Idea

**In plain English:** REST and GraphQL are two different ways an app can ask a server for data — REST works like ordering from a fixed menu where you get whatever the kitchen decides to include, while GraphQL lets you write your own custom order specifying exactly what you want and nothing more.

**Real-world analogy:** Imagine ordering food at a restaurant. With a traditional fixed combo meal (REST), you get a burger, fries, and a drink whether you want all three or not. With a build-your-own order station (GraphQL), you walk up, describe exactly what you want — "just the burger and a water" — and that is all you receive.

- The fixed combo meal = a REST endpoint that always returns the same set of fields
- The build-your-own order station = the GraphQL endpoint that accepts a custom query
- Listing only "burger and water" on your custom form = writing a GraphQL query that names only the fields you need

---

## Overview

REST (Representational State Transfer) and GraphQL represent two fundamentally different approaches to API design. While REST has been the dominant API architecture for over a decade, GraphQL emerged in 2015 to address specific limitations in REST APIs, particularly around over-fetching, under-fetching, and API versioning.

## REST Architecture

REST is an architectural style that uses HTTP methods and URIs to interact with resources:

```
Client Request Flow (REST):
┌─────────┐
│ Client  │
└────┬────┘
     │ GET /api/users/123
     ▼
┌─────────────────┐
│   REST API      │
└────┬────────────┘
     │ Returns entire user object
     ▼
{ id, name, email, address, posts, ... }
```

### REST Example (React)

```typescript
// services/userService.ts
interface User {
  id: string;
  name: string;
  email: string;
  address: Address;
  posts: Post[];
  friends: User[];
}

class UserService {
  private baseUrl = '/api';

  // Multiple requests needed for related data
  async getUserProfile(userId: string) {
    // Request 1: Get user
    const userResponse = await fetch(`${this.baseUrl}/users/${userId}`);
    const user = await userResponse.json();

    // Request 2: Get user's posts
    const postsResponse = await fetch(`${this.baseUrl}/users/${userId}/posts`);
    const posts = await postsResponse.json();

    // Request 3: Get user's friends
    const friendsResponse = await fetch(`${this.baseUrl}/users/${userId}/friends`);
    const friends = await friendsResponse.json();

    return { ...user, posts, friends };
  }

  // Over-fetching: Getting more data than needed
  async getUserName(userId: string): Promise<string> {
    const response = await fetch(`${this.baseUrl}/users/${userId}`);
    const user = await response.json(); // Gets ALL user fields
    return user.name; // Only need name
  }
}
```

```typescript
// components/UserProfile.tsx
import React, { useEffect, useState } from 'react';

interface UserProfileProps {
  userId: string;
}

const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadUser = async () => {
      try {
        setLoading(true);
        // Multiple round trips to server
        const userService = new UserService();
        const userData = await userService.getUserProfile(userId);
        setUser(userData);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    loadUser();
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <h2>Posts ({user.posts.length})</h2>
      {user.posts.map(post => (
        <article key={post.id}>{post.title}</article>
      ))}
    </div>
  );
};
```

### REST Example (Angular)

```typescript
// services/user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, forkJoin } from 'rxjs';
import { map } from 'rxjs/operators';

interface User {
  id: string;
  name: string;
  email: string;
  address: Address;
}

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private baseUrl = '/api';

  constructor(private http: HttpClient) {}

  // Multiple requests with forkJoin
  getUserProfile(userId: string): Observable<any> {
    return forkJoin({
      user: this.http.get(`${this.baseUrl}/users/${userId}`),
      posts: this.http.get(`${this.baseUrl}/users/${userId}/posts`),
      friends: this.http.get(`${this.baseUrl}/users/${userId}/friends`)
    }).pipe(
      map(({ user, posts, friends }) => ({
        ...user,
        posts,
        friends
      }))
    );
  }

  // Over-fetching example
  getUserName(userId: string): Observable<string> {
    return this.http.get<User>(`${this.baseUrl}/users/${userId}`)
      .pipe(map(user => user.name));
  }
}
```

```typescript
// components/user-profile.component.ts
import { Component, OnInit, Input } from '@angular/core';
import { UserService } from '../services/user.service';

@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">Error: {{ error }}</div>
    <div *ngIf="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
      <h2>Posts ({{ user.posts.length }})</h2>
      <article *ngFor="let post of user.posts">
        {{ post.title }}
      </article>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId: string;
  user: any;
  loading = true;
  error: string | null = null;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUserProfile(this.userId).subscribe({
      next: (data) => {
        this.user = data;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }
}
```

## GraphQL Architecture

GraphQL is a query language that allows clients to request exactly the data they need:

```
Client Request Flow (GraphQL):
┌─────────┐
│ Client  │
└────┬────┘
     │ Single query specifying exact fields
     │ query { user(id: "123") { name, email } }
     ▼
┌─────────────────┐
│  GraphQL API    │
│   (Resolvers)   │
└────┬────────────┘
     │ Returns only requested fields
     ▼
{ name: "John", email: "john@example.com" }
```

### GraphQL Example (React)

```typescript
// graphql/queries.ts
import { gql } from '@apollo/client';

export const GET_USER_PROFILE = gql`
  query GetUserProfile($userId: ID!) {
    user(id: $userId) {
      id
      name
      email
      posts {
        id
        title
        createdAt
      }
      friends {
        id
        name
        avatar
      }
    }
  }
`;

// Query only what you need
export const GET_USER_NAME = gql`
  query GetUserName($userId: ID!) {
    user(id: $userId) {
      name
    }
  }
`;

// Nested queries with arguments
export const GET_USER_WITH_RECENT_POSTS = gql`
  query GetUserWithRecentPosts($userId: ID!, $postLimit: Int = 5) {
    user(id: $userId) {
      id
      name
      posts(limit: $postLimit, orderBy: CREATED_AT_DESC) {
        id
        title
        excerpt
        author {
          name
          avatar
        }
        comments(limit: 3) {
          id
          text
          author {
            name
          }
        }
      }
    }
  }
`;
```

```typescript
// components/UserProfile.tsx
import React from 'react';
import { useQuery } from '@apollo/client';
import { GET_USER_PROFILE } from '../graphql/queries';

interface UserProfileProps {
  userId: string;
}

const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
  // Single request, exactly what we need
  const { data, loading, error } = useQuery(GET_USER_PROFILE, {
    variables: { userId }
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  const { user } = data;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <h2>Posts ({user.posts.length})</h2>
      {user.posts.map(post => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <time>{post.createdAt}</time>
        </article>
      ))}
      <h2>Friends ({user.friends.length})</h2>
      {user.friends.map(friend => (
        <div key={friend.id}>
          <img src={friend.avatar} alt={friend.name} />
          <span>{friend.name}</span>
        </div>
      ))}
    </div>
  );
};

export default UserProfile;
```

```typescript
// hooks/useOptimisticMutation.ts
import { useMutation } from '@apollo/client';
import { gql } from '@apollo/client';

const UPDATE_USER = gql`
  mutation UpdateUser($id: ID!, $input: UserInput!) {
    updateUser(id: $id, input: $input) {
      id
      name
      email
    }
  }
`;

export const useUpdateUser = () => {
  const [updateUser, { loading, error }] = useMutation(UPDATE_USER, {
    // Optimistic response
    optimisticResponse: (variables) => ({
      updateUser: {
        __typename: 'User',
        id: variables.id,
        ...variables.input
      }
    }),
    // Update cache
    update: (cache, { data }) => {
      cache.modify({
        id: cache.identify(data.updateUser),
        fields: {
          name: () => data.updateUser.name,
          email: () => data.updateUser.email
        }
      });
    }
  });

  return { updateUser, loading, error };
};
```

### GraphQL Example (Angular)

```typescript
// graphql/queries.ts
import { gql } from 'apollo-angular';

export const GET_USER_PROFILE = gql`
  query GetUserProfile($userId: ID!) {
    user(id: $userId) {
      id
      name
      email
      posts {
        id
        title
        createdAt
      }
      friends {
        id
        name
        avatar
      }
    }
  }
`;
```

```typescript
// services/user-graphql.service.ts
import { Injectable } from '@angular/core';
import { Apollo } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { GET_USER_PROFILE } from '../graphql/queries';

@Injectable({
  providedIn: 'root'
})
export class UserGraphQLService {
  constructor(private apollo: Apollo) {}

  getUserProfile(userId: string): Observable<any> {
    return this.apollo.query({
      query: GET_USER_PROFILE,
      variables: { userId }
    }).pipe(
      map(result => result.data.user)
    );
  }

  // Query with caching
  getUserProfileCached(userId: string): Observable<any> {
    return this.apollo.watchQuery({
      query: GET_USER_PROFILE,
      variables: { userId },
      fetchPolicy: 'cache-first' // Use cache if available
    }).valueChanges.pipe(
      map(result => result.data.user)
    );
  }
}
```

```typescript
// components/user-profile.component.ts
import { Component, OnInit, Input } from '@angular/core';
import { UserGraphQLService } from '../services/user-graphql.service';

@Component({
  selector: 'app-user-profile-graphql',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">Error: {{ error }}</div>
    <div *ngIf="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
      <h2>Posts ({{ user.posts.length }})</h2>
      <article *ngFor="let post of user.posts">
        <h3>{{ post.title }}</h3>
        <time>{{ post.createdAt }}</time>
      </article>
    </div>
  `
})
export class UserProfileGraphQLComponent implements OnInit {
  @Input() userId: string;
  user: any;
  loading = true;
  error: string | null = null;

  constructor(private userService: UserGraphQLService) {}

  ngOnInit() {
    this.userService.getUserProfile(this.userId).subscribe({
      next: (data) => {
        this.user = data;
        this.loading = false;
      },
      error: (err) => {
        this.error = err.message;
        this.loading = false;
      }
    });
  }
}
```

## The N+1 Problem

The N+1 problem occurs when fetching related data requires multiple queries:

```
REST N+1 Problem:
┌─────────┐
│ Client  │
└────┬────┘
     │ 1. GET /api/posts (returns 10 posts)
     ▼
┌─────────────────┐
│   REST API      │
└─────────────────┘
     │ 2. GET /api/users/1 (for post 1 author)
     │ 3. GET /api/users/2 (for post 2 author)
     │ 4. GET /api/users/3 (for post 3 author)
     │ ... (10 more requests)
     ▼
Total: 11 requests (1 + N)
```

### REST N+1 Problem

```typescript
// React: N+1 problem with REST
import React, { useEffect, useState } from 'react';

interface Post {
  id: string;
  title: string;
  authorId: string;
  author?: User;
}

const PostList: React.FC = () => {
  const [posts, setPosts] = useState<Post[]>([]);

  useEffect(() => {
    const loadPosts = async () => {
      // Request 1: Fetch all posts
      const postsResponse = await fetch('/api/posts');
      const postsData = await postsResponse.json();

      // Requests 2-N: Fetch each author (N+1 problem!)
      const postsWithAuthors = await Promise.all(
        postsData.map(async (post) => {
          const authorResponse = await fetch(`/api/users/${post.authorId}`);
          const author = await authorResponse.json();
          return { ...post, author };
        })
      );

      setPosts(postsWithAuthors);
    };

    loadPosts();
  }, []);

  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>By {post.author?.name}</p>
        </article>
      ))}
    </div>
  );
};
```

### GraphQL Solution to N+1

```typescript
// GraphQL: Single query with DataLoader
import { gql, useQuery } from '@apollo/client';

const GET_POSTS_WITH_AUTHORS = gql`
  query GetPostsWithAuthors {
    posts {
      id
      title
      author {
        id
        name
        avatar
      }
    }
  }
`;

const PostList: React.FC = () => {
  // Single request, GraphQL handles data fetching efficiently
  const { data, loading, error } = useQuery(GET_POSTS_WITH_AUTHORS);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {data.posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>By {post.author.name}</p>
        </article>
      ))}
    </div>
  );
};
```

### DataLoader Pattern (Server-side)

```typescript
// server/loaders/userLoader.ts
import DataLoader from 'dataloader';
import { User } from '../models/User';

// Batch loading to solve N+1 problem
export const createUserLoader = () => {
  return new DataLoader<string, User>(async (userIds) => {
    console.log('Batched user IDs:', userIds); // Called once per request cycle
    
    // Single database query for all user IDs
    const users = await User.find({
      _id: { $in: userIds }
    });

    // Return users in same order as requested IDs
    const userMap = new Map(users.map(user => [user.id, user]));
    return userIds.map(id => userMap.get(id));
  });
};

// GraphQL resolver
export const resolvers = {
  Post: {
    author: (post, args, context) => {
      // DataLoader batches and caches requests
      return context.loaders.user.load(post.authorId);
    }
  },
  Query: {
    posts: () => Post.find()
  }
};
```

## Caching Strategies

### REST Caching

```typescript
// React: REST caching with SWR
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const { data, error, mutate } = useSWR(
    `/api/users/${userId}`,
    fetcher,
    {
      revalidateOnFocus: true,
      dedupingInterval: 2000,
      refreshInterval: 30000
    }
  );

  const updateUser = async (updates: Partial<User>) => {
    // Optimistic update
    mutate({ ...data, ...updates }, false);
    
    // Make API call
    await fetch(`/api/users/${userId}`, {
      method: 'PATCH',
      body: JSON.stringify(updates)
    });
    
    // Revalidate
    mutate();
  };

  if (error) return <div>Error</div>;
  if (!data) return <div>Loading...</div>;

  return <div>{data.name}</div>;
};
```

```typescript
// Angular: REST caching with interceptor
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpResponse } from '@angular/common/http';
import { tap } from 'rxjs/operators';

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  private cache = new Map<string, HttpResponse<any>>();

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    // Only cache GET requests
    if (req.method !== 'GET') {
      return next.handle(req);
    }

    const cachedResponse = this.cache.get(req.url);
    if (cachedResponse) {
      return of(cachedResponse);
    }

    return next.handle(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.cache.set(req.url, event);
        }
      })
    );
  }
}
```

### GraphQL Caching

```typescript
// React: Apollo Client normalized cache
import { ApolloClient, InMemoryCache } from '@apollo/client';

const client = new ApolloClient({
  uri: '/graphql',
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          posts: {
            // Cache posts by offset/limit
            keyArgs: false,
            merge(existing = [], incoming, { args }) {
              const merged = existing.slice(0);
              const offset = args?.offset ?? 0;
              for (let i = 0; i < incoming.length; i++) {
                merged[offset + i] = incoming[i];
              }
              return merged;
            }
          }
        }
      },
      User: {
        fields: {
          friends: {
            // Merge friend lists
            merge(existing = [], incoming) {
              return [...existing, ...incoming];
            }
          }
        }
      }
    }
  })
});
```

```typescript
// React: Cache updates after mutations
import { useMutation, useQuery, gql } from '@apollo/client';

const GET_TODOS = gql`
  query GetTodos {
    todos {
      id
      text
      completed
    }
  }
`;

const ADD_TODO = gql`
  mutation AddTodo($text: String!) {
    addTodo(text: $text) {
      id
      text
      completed
    }
  }
`;

const TodoList: React.FC = () => {
  const { data } = useQuery(GET_TODOS);
  const [addTodo] = useMutation(ADD_TODO, {
    update: (cache, { data: { addTodo } }) => {
      // Read existing todos from cache
      const existing: any = cache.readQuery({ query: GET_TODOS });
      
      // Write updated todos back to cache
      cache.writeQuery({
        query: GET_TODOS,
        data: {
          todos: [...existing.todos, addTodo]
        }
      });
    }
  });

  return (
    <div>
      {data?.todos.map(todo => (
        <div key={todo.id}>{todo.text}</div>
      ))}
    </div>
  );
};
```

## Trade-offs Analysis

```
REST Advantages:
├── Simple, well-understood
├── Great HTTP caching (CDN, browser)
├── Easy to debug (URLs are resources)
├── Stateless
└── Works well with HTTP verbs

REST Disadvantages:
├── Over-fetching (getting unused data)
├── Under-fetching (multiple requests)
├── API versioning complexity
├── No type safety
└── N+1 queries

GraphQL Advantages:
├── Fetch exactly what you need
├── Single request for nested data
├── Strong type system
├── No versioning needed
├── Excellent developer experience
└── Normalized caching

GraphQL Disadvantages:
├── Complexity (learning curve)
├── Harder to cache (no URL-based caching)
├── Query complexity attacks
├── File upload complications
└── Over-fetching on server side
```

### Performance Comparison

```typescript
// React: Performance comparison component
import React, { useState, useEffect } from 'react';

const PerformanceComparison: React.FC = () => {
  const [restTime, setRestTime] = useState(0);
  const [graphqlTime, setGraphqlTime] = useState(0);

  const testREST = async () => {
    const start = performance.now();
    
    // Multiple REST requests
    const userRes = await fetch('/api/users/1');
    const user = await userRes.json();
    
    const postsRes = await fetch(`/api/users/1/posts`);
    const posts = await postsRes.json();
    
    const friendsRes = await fetch(`/api/users/1/friends`);
    const friends = await friendsRes.json();
    
    const end = performance.now();
    setRestTime(end - start);
  };

  const testGraphQL = async () => {
    const start = performance.now();
    
    // Single GraphQL query
    const response = await fetch('/graphql', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        query: `
          query {
            user(id: "1") {
              name
              posts { title }
              friends { name }
            }
          }
        `
      })
    });
    await response.json();
    
    const end = performance.now();
    setGraphqlTime(end - start);
  };

  return (
    <div>
      <button onClick={testREST}>Test REST</button>
      <button onClick={testGraphQL}>Test GraphQL</button>
      <div>REST: {restTime.toFixed(2)}ms</div>
      <div>GraphQL: {graphqlTime.toFixed(2)}ms</div>
    </div>
  );
};
```

## When to Use Each

### Use REST When:

```typescript
// 1. Simple CRUD operations
class ArticleService {
  // REST is perfect for straightforward CRUD
  getArticle(id: string) {
    return fetch(`/api/articles/${id}`);
  }

  createArticle(data: Article) {
    return fetch('/api/articles', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  updateArticle(id: string, data: Partial<Article>) {
    return fetch(`/api/articles/${id}`, {
      method: 'PATCH',
      body: JSON.stringify(data)
    });
  }

  deleteArticle(id: string) {
    return fetch(`/api/articles/${id}`, {
      method: 'DELETE'
    });
  }
}

// 2. Heavy caching requirements (CDN, browser cache)
// REST URLs can be cached at every level
fetch('/api/articles/123', {
  headers: {
    'Cache-Control': 'public, max-age=3600'
  }
});

// 3. File uploads/downloads
const uploadFile = async (file: File) => {
  const formData = new FormData();
  formData.append('file', file);
  
  return fetch('/api/upload', {
    method: 'POST',
    body: formData
  });
};
```

### Use GraphQL When:

```typescript
// 1. Complex, nested data requirements
const DASHBOARD_QUERY = gql`
  query Dashboard {
    currentUser {
      name
      notifications(unreadOnly: true) {
        id
        message
        sender { name, avatar }
      }
      projects {
        id
        name
        tasks(status: IN_PROGRESS) {
          id
          title
          assignees { name }
        }
      }
      team {
        members {
          name
          currentTask { title }
        }
      }
    }
  }
`;

// 2. Mobile applications (bandwidth optimization)
const MOBILE_QUERY = gql`
  query MobileUserProfile {
    user(id: $id) {
      name
      avatar(size: SMALL) # Only small images
      # Omit heavy fields like full bio, all posts, etc.
    }
  }
`;

// 3. Rapidly evolving requirements
// No API versioning needed - just add fields
const USER_QUERY_V1 = gql`
  query User { user { name } }
`;

const USER_QUERY_V2 = gql`
  query User { 
    user { 
      name 
      email # New field added without breaking V1
    } 
  }
`;
```

## Common Mistakes

### Mistake 1: Using GraphQL for Simple APIs

```typescript
// BAD: GraphQL overkill for simple CRUD
const GET_ARTICLE = gql`
  query GetArticle($id: ID!) {
    article(id: $id) {
      id
      title
      content
    }
  }
`;

// GOOD: REST is simpler here
fetch('/api/articles/123')
  .then(r => r.json());
```

### Mistake 2: Not Handling REST Over-fetching

```typescript
// BAD: Fetching entire user object
const getUserEmail = async (id: string) => {
  const user = await fetch(`/api/users/${id}`).then(r => r.json());
  return user.email; // Wasted bandwidth on unused fields
};

// GOOD: Create specific endpoints
const getUserEmail = async (id: string) => {
  const data = await fetch(`/api/users/${id}/email`).then(r => r.json());
  return data.email;
};

// BETTER: Use field selection if supported
const getUserEmail = async (id: string) => {
  const user = await fetch(`/api/users/${id}?fields=email`)
    .then(r => r.json());
  return user.email;
};
```

### Mistake 3: GraphQL Query Complexity Attack

```typescript
// BAD: Allowing deeply nested queries
// Client can send expensive query:
query MaliciousQuery {
  users {
    friends {
      friends {
        friends {
          friends {
            # ... 100 levels deep
          }
        }
      }
    }
  }
}

// GOOD: Implement query complexity analysis
import { GraphQLSchema } from 'graphql';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const schema = new GraphQLSchema({
  // ... schema definition
});

const complexityLimit = createComplexityLimitRule(1000, {
  scalarCost: 1,
  objectCost: 2,
  listFactor: 10
});

// Use in server
app.use('/graphql', graphqlHTTP({
  schema,
  validationRules: [complexityLimit]
}));
```

### Mistake 4: Not Implementing DataLoader

```typescript
// BAD: N+1 queries in resolvers
const resolvers = {
  Post: {
    author: async (post) => {
      // Called once per post - N+1 problem!
      return await User.findById(post.authorId);
    }
  }
};

// GOOD: Use DataLoader
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (ids) => {
  const users = await User.find({ _id: { $in: ids } });
  return ids.map(id => users.find(user => user.id === id));
});

const resolvers = {
  Post: {
    author: (post, args, { loaders }) => {
      return loaders.user.load(post.authorId);
    }
  }
};
```

## Best Practices

### REST Best Practices

```typescript
// 1. Use proper HTTP status codes
class APIResponse {
  static success(data: any, status = 200) {
    return { status, data };
  }

  static created(data: any) {
    return { status: 201, data };
  }

  static noContent() {
    return { status: 204 };
  }

  static badRequest(message: string) {
    return { status: 400, error: message };
  }

  static notFound(resource: string) {
    return { status: 404, error: `${resource} not found` };
  }
}

// 2. Implement HATEOAS (Hypermedia)
interface Article {
  id: string;
  title: string;
  _links: {
    self: { href: string };
    author: { href: string };
    comments: { href: string };
  };
}

// 3. Use field selection
// GET /api/users/123?fields=name,email,avatar
```

### GraphQL Best Practices

```typescript
// 1. Fragment for reusable queries
const USER_FRAGMENT = gql`
  fragment UserFields on User {
    id
    name
    email
    avatar
  }
`;

const GET_USER = gql`
  ${USER_FRAGMENT}
  query GetUser($id: ID!) {
    user(id: $id) {
      ...UserFields
      posts {
        id
        title
      }
    }
  }
`;

// 2. Implement pagination
const GET_POSTS = gql`
  query GetPosts($first: Int!, $after: String) {
    posts(first: $first, after: $after) {
      edges {
        node {
          id
          title
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`;

// 3. Error handling
const resolvers = {
  Query: {
    user: async (parent, { id }) => {
      const user = await User.findById(id);
      if (!user) {
        throw new GraphQLError('User not found', {
          extensions: {
            code: 'USER_NOT_FOUND',
            userId: id
          }
        });
      }
      return user;
    }
  }
};

// 4. Batching and caching
import { BatchHttpLink } from '@apollo/client/link/batch-http';

const link = new BatchHttpLink({
  uri: '/graphql',
  batchMax: 10,
  batchInterval: 20
});
```

## Key Takeaways

1. REST is simpler for CRUD operations and better for caching
2. GraphQL excels at complex, nested data requirements and eliminates over-fetching
3. The N+1 problem affects both but GraphQL has better tooling (DataLoader)
4. REST uses URL-based caching; GraphQL uses normalized caching
5. Choose based on your specific use case, not hype
6. GraphQL has a steeper learning curve but offers better developer experience
7. REST is more mature with broader ecosystem support
8. Both can coexist in the same application

## Interview Questions

1. Explain the N+1 problem and how DataLoader solves it in GraphQL
2. When would you choose REST over GraphQL?
3. How does GraphQL caching differ from REST caching?
4. What are the security concerns with GraphQL (query complexity, depth limiting)?
5. How do you handle file uploads in GraphQL vs REST?
6. Explain over-fetching and under-fetching with examples
7. How does versioning work in GraphQL vs REST?
8. What are the performance trade-offs between REST and GraphQL?
9. How do you implement pagination in both REST and GraphQL?
10. What is the GraphQL schema stitching and federation?

## Resources

- GraphQL Official Documentation
- Apollo Client Documentation
- REST API Design Best Practices
- DataLoader GitHub Repository
- Roy Fielding's REST Dissertation
- GraphQL vs REST Performance Analysis
- HATEOAS Principles
- GraphQL Complexity Analysis
