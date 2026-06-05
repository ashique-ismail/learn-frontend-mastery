# GraphQL Architecture

## The Idea

**In plain English:** GraphQL is a way for a website or app to ask a server for exactly the data it needs — nothing more, nothing less — using a single door (one endpoint) instead of many separate doors. Think of it as placing a very specific food order rather than getting a fixed meal.

**Real-world analogy:** Imagine ordering at a build-your-own-sandwich counter. You tell the worker exactly which ingredients you want, and they assemble just that sandwich for you.

- The sandwich counter = the GraphQL server (the single endpoint that handles all requests)
- The ingredients menu = the schema (the full list of data fields available to request)
- Your specific order = the query (the exact fields your app asks for)

---

## Overview

GraphQL is a query language for APIs and a runtime for executing those queries with your existing data. Unlike REST, which exposes multiple endpoints, GraphQL exposes a single endpoint that accepts queries describing exactly what data is needed. The architecture consists of schemas, types, queries, mutations, subscriptions, and resolvers.

## Core Architecture

```text
GraphQL Request Flow:
┌──────────┐
│  Client  │
└─────┬────┘
      │ Query/Mutation/Subscription
      ▼
┌────────────────┐
│  GraphQL       │
│  Server        │
│  ┌──────────┐  │
│  │  Schema  │  │ (Type definitions)
│  └──────────┘  │
│  ┌──────────┐  │
│  │Resolvers │  │ (Business logic)
│  └──────────┘  │
└────────┬───────┘
         │
    ┌────┴────┬────────┬────────┐
    ▼         ▼        ▼        ▼
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│ DB  │  │ REST│  │Cache│  │Other│
└─────┘  └─────┘  └─────┘  └─────┘
```

## Schema Definition Language (SDL)

The schema is the contract between client and server:

```typescript
// schema.graphql
"""
User type represents an authenticated user in the system
"""
type User {
  id: ID!
  username: String!
  email: String!
  avatar: String
  bio: String
  createdAt: DateTime!
  posts(limit: Int, offset: Int): [Post!]!
  followers: [User!]!
  following: [User!]!
  followerCount: Int!
}

"""
Post type represents a blog post or article
"""
type Post {
  id: ID!
  title: String!
  content: String!
  excerpt: String
  author: User!
  comments: [Comment!]!
  likes: [User!]!
  likeCount: Int!
  published: Boolean!
  createdAt: DateTime!
  updatedAt: DateTime!
  tags: [String!]!
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
  parent: Comment
  replies: [Comment!]!
  createdAt: DateTime!
}

"""
Custom scalar for date/time values
"""
scalar DateTime

"""
Input type for creating a new post
"""
input CreatePostInput {
  title: String!
  content: String!
  excerpt: String
  tags: [String!]
  published: Boolean = false
}

"""
Input type for updating an existing post
"""
input UpdatePostInput {
  title: String
  content: String
  excerpt: String
  tags: [String!]
  published: Boolean
}

"""
Pagination input for list queries
"""
input PaginationInput {
  limit: Int = 10
  offset: Int = 0
}

"""
The root query type
"""
type Query {
  "Get current authenticated user"
  me: User
  
  "Get user by ID"
  user(id: ID!): User
  
  "Search users by username"
  searchUsers(query: String!, pagination: PaginationInput): [User!]!
  
  "Get post by ID"
  post(id: ID!): Post
  
  "Get all posts with optional filters"
  posts(
    authorId: ID
    published: Boolean
    tags: [String!]
    pagination: PaginationInput
  ): PostConnection!
  
  "Get user's feed"
  feed(pagination: PaginationInput): [Post!]!
}

"""
Pagination connection for posts
"""
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

"""
The root mutation type
"""
type Mutation {
  "Create a new post"
  createPost(input: CreatePostInput!): Post!
  
  "Update an existing post"
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  
  "Delete a post"
  deletePost(id: ID!): Boolean!
  
  "Like a post"
  likePost(postId: ID!): Post!
  
  "Unlike a post"
  unlikePost(postId: ID!): Post!
  
  "Follow a user"
  followUser(userId: ID!): User!
  
  "Unfollow a user"
  unfollowUser(userId: ID!): User!
  
  "Add comment to post"
  addComment(postId: ID!, text: String!, parentId: ID): Comment!
}

"""
The root subscription type for real-time updates
"""
type Subscription {
  "Subscribe to new posts from followed users"
  newPost: Post!
  
  "Subscribe to new comments on a post"
  newComment(postId: ID!): Comment!
  
  "Subscribe to like notifications"
  postLiked(postId: ID!): User!
  
  "Subscribe to user status changes"
  userStatusChanged(userId: ID!): UserStatus!
}

type UserStatus {
  userId: ID!
  online: Boolean!
  lastSeen: DateTime
}

"""
Enums for type-safe constants
"""
enum PostSortField {
  CREATED_AT
  UPDATED_AT
  LIKE_COUNT
  COMMENT_COUNT
}

enum SortOrder {
  ASC
  DESC
}
```

## Queries

Queries are read operations that fetch data:

### Basic Queries (React)

```typescript
// queries/user.queries.ts
import { gql } from '@apollo/client';

// Simple query
export const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      username
      email
      avatar
      bio
    }
  }
`;

// Query with nested fields
export const GET_USER_WITH_POSTS = gql`
  query GetUserWithPosts($id: ID!, $postLimit: Int = 10) {
    user(id: $id) {
      id
      username
      avatar
      posts(limit: $postLimit) {
        id
        title
        excerpt
        createdAt
        likeCount
      }
      followerCount
    }
  }
`;

// Query with fragments
const POST_FRAGMENT = gql`
  fragment PostFields on Post {
    id
    title
    excerpt
    createdAt
    likeCount
    author {
      id
      username
      avatar
    }
    tags
  }
`;

export const GET_FEED = gql`
  ${POST_FRAGMENT}
  query GetFeed($limit: Int, $offset: Int) {
    feed(pagination: { limit: $limit, offset: $offset }) {
      ...PostFields
      comments {
        id
        text
        author {
          username
        }
      }
    }
  }
`;

// Query with pagination
export const GET_POSTS_PAGINATED = gql`
  query GetPostsPaginated($limit: Int!, $after: String) {
    posts(pagination: { limit: $limit }, after: $after) {
      edges {
        node {
          id
          title
          excerpt
          author {
            username
          }
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
      totalCount
    }
  }
`;
```

```typescript
// components/UserProfile.tsx
import React from 'react';
import { useQuery } from '@apollo/client';
import { GET_USER_WITH_POSTS } from '../queries/user.queries';

interface UserProfileProps {
  userId: string;
}

const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
  const { data, loading, error, refetch } = useQuery(GET_USER_WITH_POSTS, {
    variables: { id: userId, postLimit: 5 },
    // Query options
    fetchPolicy: 'cache-first', // or 'network-only', 'cache-and-network'
    pollInterval: 30000, // Poll every 30 seconds
    notifyOnNetworkStatusChange: true
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  const { user } = data;

  return (
    <div>
      <img src={user.avatar} alt={user.username} />
      <h1>{user.username}</h1>
      <p>{user.bio}</p>
      <div>{user.followerCount} followers</div>
      
      <h2>Recent Posts</h2>
      {user.posts.map(post => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <p>{post.excerpt}</p>
          <div>{post.likeCount} likes</div>
        </article>
      ))}
      
      <button onClick={() => refetch()}>Refresh</button>
    </div>
  );
};
```

### Query Hooks Patterns (React)

```typescript
// hooks/useInfiniteScroll.ts
import { useQuery } from '@apollo/client';
import { GET_POSTS_PAGINATED } from '../queries/post.queries';

export const useInfiniteScroll = (limit = 10) => {
  const { data, loading, error, fetchMore } = useQuery(GET_POSTS_PAGINATED, {
    variables: { limit }
  });

  const loadMore = () => {
    if (!data?.posts.pageInfo.hasNextPage) return;

    fetchMore({
      variables: {
        after: data.posts.pageInfo.endCursor
      },
      updateQuery: (prev, { fetchMoreResult }) => {
        if (!fetchMoreResult) return prev;

        return {
          posts: {
            ...fetchMoreResult.posts,
            edges: [
              ...prev.posts.edges,
              ...fetchMoreResult.posts.edges
            ]
          }
        };
      }
    });
  };

  return {
    posts: data?.posts.edges.map(edge => edge.node) || [],
    loading,
    error,
    loadMore,
    hasMore: data?.posts.pageInfo.hasNextPage
  };
};
```

```typescript
// components/InfinitePostList.tsx
import React, { useEffect, useRef } from 'react';
import { useInfiniteScroll } from '../hooks/useInfiniteScroll';

const InfinitePostList: React.FC = () => {
  const { posts, loading, loadMore, hasMore } = useInfiniteScroll(10);
  const observerTarget = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      entries => {
        if (entries[0].isIntersecting && hasMore && !loading) {
          loadMore();
        }
      },
      { threshold: 1.0 }
    );

    if (observerTarget.current) {
      observer.observe(observerTarget.current);
    }

    return () => observer.disconnect();
  }, [hasMore, loading, loadMore]);

  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
      {loading && <div>Loading more...</div>}
      <div ref={observerTarget} />
    </div>
  );
};
```

### Queries (Angular)

```typescript
// services/user-query.service.ts
import { Injectable } from '@angular/core';
import { Apollo, gql } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      username
      email
      avatar
      bio
    }
  }
`;

@Injectable({
  providedIn: 'root'
})
export class UserQueryService {
  constructor(private apollo: Apollo) {}

  getUser(id: string): Observable<any> {
    return this.apollo.query({
      query: GET_USER,
      variables: { id }
    }).pipe(
      map(result => result.data.user)
    );
  }

  // With cache control
  getUserCached(id: string): Observable<any> {
    return this.apollo.watchQuery({
      query: GET_USER,
      variables: { id },
      fetchPolicy: 'cache-first'
    }).valueChanges.pipe(
      map(result => result.data.user)
    );
  }
}
```

```typescript
// components/user-profile.component.ts
import { Component, OnInit, Input } from '@angular/core';
import { UserQueryService } from '../services/user-query.service';

@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">Error: {{ error }}</div>
    <div *ngIf="user">
      <img [src]="user.avatar" [alt]="user.username">
      <h1>{{ user.username }}</h1>
      <p>{{ user.bio }}</p>
    </div>
  `
})
export class UserProfileComponent implements OnInit {
  @Input() userId: string;
  user: any;
  loading = true;
  error: string | null = null;

  constructor(private userQuery: UserQueryService) {}

  ngOnInit() {
    this.userQuery.getUser(this.userId).subscribe({
      next: (user) => {
        this.user = user;
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

## Mutations

Mutations are write operations that modify data:

### Basic Mutations (React)

```typescript
// mutations/post.mutations.ts
import { gql } from '@apollo/client';

export const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
      content
      author {
        id
        username
      }
      createdAt
    }
  }
`;

export const UPDATE_POST = gql`
  mutation UpdatePost($id: ID!, $input: UpdatePostInput!) {
    updatePost(id: $id, input: $input) {
      id
      title
      content
      updatedAt
    }
  }
`;

export const LIKE_POST = gql`
  mutation LikePost($postId: ID!) {
    likePost(postId: $postId) {
      id
      likeCount
      likes {
        id
        username
      }
    }
  }
`;

export const ADD_COMMENT = gql`
  mutation AddComment($postId: ID!, $text: String!, $parentId: ID) {
    addComment(postId: $postId, text: $text, parentId: $parentId) {
      id
      text
      author {
        id
        username
        avatar
      }
      createdAt
      parent {
        id
      }
    }
  }
`;
```

```typescript
// components/CreatePostForm.tsx
import React, { useState } from 'react';
import { useMutation } from '@apollo/client';
import { CREATE_POST } from '../mutations/post.mutations';
import { GET_FEED } from '../queries/post.queries';

const CreatePostForm: React.FC = () => {
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  const [tags, setTags] = useState<string[]>([]);

  const [createPost, { loading, error }] = useMutation(CREATE_POST, {
    // Optimistic response
    optimisticResponse: {
      createPost: {
        __typename: 'Post',
        id: 'temp-id',
        title,
        content,
        author: {
          __typename: 'User',
          id: 'current-user-id',
          username: 'Current User'
        },
        createdAt: new Date().toISOString()
      }
    },
    
    // Update cache after mutation
    update: (cache, { data }) => {
      const existingFeed: any = cache.readQuery({ query: GET_FEED });
      
      if (existingFeed) {
        cache.writeQuery({
          query: GET_FEED,
          data: {
            feed: [data.createPost, ...existingFeed.feed]
          }
        });
      }
    },
    
    // Refetch queries after mutation
    refetchQueries: [{ query: GET_FEED }],
    
    // Or just invalidate specific cache entries
    // refetchQueries: ['GetFeed']
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    try {
      await createPost({
        variables: {
          input: { title, content, tags, published: true }
        }
      });
      
      // Reset form
      setTitle('');
      setContent('');
      setTags([]);
    } catch (err) {
      console.error('Error creating post:', err);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Post title"
        required
      />
      <textarea
        value={content}
        onChange={(e) => setContent(e.target.value)}
        placeholder="Post content"
        required
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Post'}
      </button>
      {error && <div>Error: {error.message}</div>}
    </form>
  );
};
```

### Optimistic Updates (React)

```typescript
// components/LikeButton.tsx
import React from 'react';
import { useMutation } from '@apollo/client';
import { LIKE_POST } from '../mutations/post.mutations';

interface LikeButtonProps {
  postId: string;
  likeCount: number;
  isLiked: boolean;
}

const LikeButton: React.FC<LikeButtonProps> = ({ 
  postId, 
  likeCount, 
  isLiked 
}) => {
  const [likePost] = useMutation(LIKE_POST, {
    // Optimistic update - UI updates immediately
    optimisticResponse: {
      likePost: {
        __typename: 'Post',
        id: postId,
        likeCount: isLiked ? likeCount - 1 : likeCount + 1,
        likes: [] // We could calculate this more accurately
      }
    },
    
    // Update cache manually for fine-grained control
    update: (cache, { data }) => {
      cache.modify({
        id: cache.identify({ __typename: 'Post', id: postId }),
        fields: {
          likeCount: () => data.likePost.likeCount,
          likes: () => data.likePost.likes
        }
      });
    }
  });

  const handleLike = async () => {
    try {
      await likePost({ variables: { postId } });
    } catch (err) {
      console.error('Failed to like post:', err);
      // Optimistic update will be rolled back automatically
    }
  };

  return (
    <button onClick={handleLike}>
      {isLiked ? '❤️' : '🤍'} {likeCount}
    </button>
  );
};
```

### Mutations (Angular)

```typescript
// services/post-mutation.service.ts
import { Injectable } from '@angular/core';
import { Apollo, gql } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
      content
      createdAt
    }
  }
`;

@Injectable({
  providedIn: 'root'
})
export class PostMutationService {
  constructor(private apollo: Apollo) {}

  createPost(input: any): Observable<any> {
    return this.apollo.mutate({
      mutation: CREATE_POST,
      variables: { input },
      refetchQueries: ['GetFeed']
    }).pipe(
      map(result => result.data.createPost)
    );
  }
}
```

## Subscriptions

Subscriptions enable real-time updates via WebSocket:

### Subscription Setup (React)

```typescript
// apollo-client.ts
import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';

// HTTP link for queries and mutations
const httpLink = new HttpLink({
  uri: 'http://localhost:4000/graphql'
});

// WebSocket link for subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: 'ws://localhost:4000/graphql',
    connectionParams: {
      authToken: localStorage.getItem('token')
    }
  })
);

// Split between HTTP and WebSocket based on operation type
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  httpLink
);

export const client = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache()
});
```

### Subscription Queries

```typescript
// subscriptions/post.subscriptions.ts
import { gql } from '@apollo/client';

export const NEW_POST_SUBSCRIPTION = gql`
  subscription OnNewPost {
    newPost {
      id
      title
      excerpt
      author {
        id
        username
        avatar
      }
      createdAt
    }
  }
`;

export const NEW_COMMENT_SUBSCRIPTION = gql`
  subscription OnNewComment($postId: ID!) {
    newComment(postId: $postId) {
      id
      text
      author {
        id
        username
        avatar
      }
      createdAt
    }
  }
`;

export const POST_LIKED_SUBSCRIPTION = gql`
  subscription OnPostLiked($postId: ID!) {
    postLiked(postId: $postId) {
      id
      username
      avatar
    }
  }
`;
```

### Using Subscriptions (React)

```typescript
// components/LiveFeed.tsx
import React, { useEffect } from 'react';
import { useQuery, useSubscription } from '@apollo/client';
import { GET_FEED } from '../queries/post.queries';
import { NEW_POST_SUBSCRIPTION } from '../subscriptions/post.subscriptions';

const LiveFeed: React.FC = () => {
  const { data, loading } = useQuery(GET_FEED);
  
  // Subscribe to new posts
  const { data: newPostData } = useSubscription(NEW_POST_SUBSCRIPTION);

  useEffect(() => {
    if (newPostData?.newPost) {
      // Show notification
      showNotification(`New post: ${newPostData.newPost.title}`);
    }
  }, [newPostData]);

  if (loading) return <div>Loading...</div>;

  return (
    <div>
      {data?.feed.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  );
};
```

```typescript
// components/LiveComments.tsx
import React from 'react';
import { useQuery, useSubscription } from '@apollo/client';
import { GET_POST_COMMENTS } from '../queries/comment.queries';
import { NEW_COMMENT_SUBSCRIPTION } from '../subscriptions/post.subscriptions';

interface LiveCommentsProps {
  postId: string;
}

const LiveComments: React.FC<LiveCommentsProps> = ({ postId }) => {
  const { data, loading } = useQuery(GET_POST_COMMENTS, {
    variables: { postId }
  });

  // Subscribe to new comments with cache update
  useSubscription(NEW_COMMENT_SUBSCRIPTION, {
    variables: { postId },
    onData: ({ client, data: subscriptionData }) => {
      const newComment = subscriptionData.data.newComment;
      
      // Update cache with new comment
      const existingComments: any = client.readQuery({
        query: GET_POST_COMMENTS,
        variables: { postId }
      });

      client.writeQuery({
        query: GET_POST_COMMENTS,
        variables: { postId },
        data: {
          comments: [...existingComments.comments, newComment]
        }
      });
    }
  });

  if (loading) return <div>Loading comments...</div>;

  return (
    <div>
      {data?.comments.map(comment => (
        <div key={comment.id}>
          <strong>{comment.author.username}</strong>
          <p>{comment.text}</p>
          <time>{comment.createdAt}</time>
        </div>
      ))}
    </div>
  );
};
```

### Subscriptions (Angular)

```typescript
// services/subscription.service.ts
import { Injectable } from '@angular/core';
import { Apollo, gql } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

const NEW_POST = gql`
  subscription OnNewPost {
    newPost {
      id
      title
      excerpt
      author { username }
    }
  }
`;

@Injectable({
  providedIn: 'root'
})
export class SubscriptionService {
  constructor(private apollo: Apollo) {}

  subscribeToNewPosts(): Observable<any> {
    return this.apollo.subscribe({
      query: NEW_POST
    }).pipe(
      map(result => result.data.newPost)
    );
  }
}
```

```typescript
// components/live-feed.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { SubscriptionService } from '../services/subscription.service';

@Component({
  selector: 'app-live-feed',
  template: `
    <div *ngFor="let post of posts">
      <h2>{{ post.title }}</h2>
      <p>{{ post.excerpt }}</p>
    </div>
  `
})
export class LiveFeedComponent implements OnInit, OnDestroy {
  posts: any[] = [];
  private subscription: Subscription;

  constructor(private subService: SubscriptionService) {}

  ngOnInit() {
    this.subscription = this.subService.subscribeToNewPosts()
      .subscribe(newPost => {
        this.posts = [newPost, ...this.posts];
      });
  }

  ngOnDestroy() {
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
  }
}
```

## Resolvers

Resolvers are functions that handle fetching data for each field:

### Basic Resolvers

```typescript
// server/resolvers/user.resolvers.ts
import { User } from '../models/User';
import { Post } from '../models/Post';

export const userResolvers = {
  Query: {
    // Resolver for user query
    user: async (parent, { id }, context) => {
      // Check authentication
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      const user = await User.findById(id);
      if (!user) {
        throw new Error('User not found');
      }

      return user;
    },

    // Resolver with pagination
    searchUsers: async (parent, { query, pagination }, context) => {
      const { limit = 10, offset = 0 } = pagination || {};
      
      const users = await User.find({
        username: { $regex: query, $options: 'i' }
      })
        .limit(limit)
        .skip(offset);

      return users;
    }
  },

  User: {
    // Field resolver for computed field
    followerCount: async (user, args, context) => {
      // user is the parent User object
      return await context.loaders.followerCount.load(user.id);
    },

    // Field resolver for relationship
    posts: async (user, { limit, offset }, context) => {
      return await Post.find({ authorId: user.id })
        .limit(limit || 10)
        .skip(offset || 0)
        .sort({ createdAt: -1 });
    },

    // Field resolver with DataLoader
    followers: async (user, args, context) => {
      return context.loaders.userFollowers.load(user.id);
    }
  },

  Mutation: {
    // Resolver for creating data
    createPost: async (parent, { input }, context) => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      const post = new Post({
        ...input,
        authorId: context.user.id
      });

      await post.save();

      // Publish to subscription
      context.pubsub.publish('NEW_POST', { newPost: post });

      return post;
    },

    // Resolver for updating data
    updatePost: async (parent, { id, input }, context) => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      const post = await Post.findById(id);
      if (!post) {
        throw new Error('Post not found');
      }

      if (post.authorId !== context.user.id) {
        throw new Error('Not authorized');
      }

      Object.assign(post, input);
      await post.save();

      return post;
    }
  }
};
```

### Advanced Resolver Patterns

```typescript
// server/resolvers/advanced.resolvers.ts
import DataLoader from 'dataloader';
import { PubSub } from 'graphql-subscriptions';

// DataLoader for batching
const createUserLoader = () => {
  return new DataLoader(async (userIds: string[]) => {
    const users = await User.find({ _id: { $in: userIds } });
    const userMap = new Map(users.map(u => [u.id, u]));
    return userIds.map(id => userMap.get(id));
  });
};

// Context factory
export const createContext = ({ req, connection }) => {
  if (connection) {
    // WebSocket connection for subscriptions
    return {
      user: connection.context.user,
      loaders: {
        user: createUserLoader()
      }
    };
  }

  // HTTP request for queries/mutations
  return {
    user: req.user, // From auth middleware
    loaders: {
      user: createUserLoader(),
      followerCount: createFollowerCountLoader()
    },
    pubsub: new PubSub()
  };
};

// Resolver with authorization
const requireAuth = (resolver) => {
  return (parent, args, context, info) => {
    if (!context.user) {
      throw new Error('Not authenticated');
    }
    return resolver(parent, args, context, info);
  };
};

// Resolver with caching
const cacheResolver = (resolver, ttl = 60) => {
  const cache = new Map();
  
  return async (parent, args, context, info) => {
    const key = JSON.stringify({ parent, args });
    
    if (cache.has(key)) {
      const { value, timestamp } = cache.get(key);
      if (Date.now() - timestamp < ttl * 1000) {
        return value;
      }
    }

    const result = await resolver(parent, args, context, info);
    cache.set(key, { value: result, timestamp: Date.now() });
    
    return result;
  };
};

// Using decorators
export const resolvers = {
  Query: {
    user: requireAuth(cacheResolver(async (parent, { id }, context) => {
      return await context.loaders.user.load(id);
    }, 30))
  }
};
```

### Subscription Resolvers

```typescript
// server/resolvers/subscription.resolvers.ts
import { PubSub, withFilter } from 'graphql-subscriptions';

const pubsub = new PubSub();

export const subscriptionResolvers = {
  Subscription: {
    // Simple subscription
    newPost: {
      subscribe: () => pubsub.asyncIterator(['NEW_POST'])
    },

    // Filtered subscription
    newComment: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(['NEW_COMMENT']),
        (payload, variables) => {
          // Only send comments for specific post
          return payload.newComment.postId === variables.postId;
        }
      )
    },

    // Subscription with authentication
    postLiked: {
      subscribe: withFilter(
        (parent, args, context) => {
          if (!context.user) {
            throw new Error('Not authenticated');
          }
          return pubsub.asyncIterator(['POST_LIKED']);
        },
        (payload, variables) => {
          return payload.postLiked.postId === variables.postId;
        }
      )
    }
  }
};

// Publishing events
export const publishNewPost = (post) => {
  pubsub.publish('NEW_POST', { newPost: post });
};

export const publishNewComment = (comment) => {
  pubsub.publish('NEW_COMMENT', { newComment: comment });
};
```

## Common Mistakes

### Mistake 1: N+1 Queries Without DataLoader

```typescript
// BAD: N+1 problem
const resolvers = {
  Post: {
    author: async (post) => {
      // Called once per post!
      return await User.findById(post.authorId);
    }
  }
};

// GOOD: Use DataLoader
const userLoader = new DataLoader(async (ids) => {
  const users = await User.find({ _id: { $in: ids } });
  return ids.map(id => users.find(u => u.id === id));
});

const resolvers = {
  Post: {
    author: (post, args, { loaders }) => {
      return loaders.user.load(post.authorId);
    }
  }
};
```

### Mistake 2: Not Using Fragments

```typescript
// BAD: Repeating field selections
const query1 = gql`
  query { user(id: "1") { id, name, email, avatar } }
`;
const query2 = gql`
  query { user(id: "2") { id, name, email, avatar } }
`;

// GOOD: Use fragments
const USER_FIELDS = gql`
  fragment UserFields on User {
    id
    name
    email
    avatar
  }
`;

const query1 = gql`
  ${USER_FIELDS}
  query { user(id: "1") { ...UserFields } }
`;
```

### Mistake 3: Ignoring Cache Updates

```typescript
// BAD: Not updating cache after mutation
const [createPost] = useMutation(CREATE_POST);

// GOOD: Update cache
const [createPost] = useMutation(CREATE_POST, {
  update: (cache, { data }) => {
    const existing: any = cache.readQuery({ query: GET_POSTS });
    cache.writeQuery({
      query: GET_POSTS,
      data: { posts: [data.createPost, ...existing.posts] }
    });
  }
});
```

## Best Practices

1. Use DataLoader to batch and cache database queries
2. Implement proper error handling with custom error codes
3. Use fragments to avoid repeating field selections
4. Implement pagination for list queries
5. Add field-level authorization in resolvers
6. Use optimistic updates for better UX
7. Implement proper cache invalidation strategies
8. Use subscriptions sparingly (they're expensive)
9. Add query complexity analysis to prevent abuse
10. Version your schema with deprecation annotations

## Key Takeaways

1. Schema defines the contract between client and server
2. Queries fetch data, mutations modify data, subscriptions provide real-time updates
3. Resolvers contain the business logic for fetching data
4. DataLoader solves the N+1 problem through batching
5. Apollo Client provides powerful caching and state management
6. Subscriptions use WebSocket for real-time communication
7. Fragments promote code reuse in queries
8. Proper cache management is crucial for performance

## Interview Questions

1. Explain the difference between Query, Mutation, and Subscription
2. What is the N+1 problem and how does DataLoader solve it?
3. How does GraphQL caching work in Apollo Client?
4. What are resolvers and when do they execute?
5. How do you implement pagination in GraphQL?
6. What is the purpose of fragments?
7. How do subscriptions work under the hood?
8. How do you handle authorization in GraphQL?
9. What are the trade-offs of using subscriptions?
10. How do you prevent query complexity attacks?

## Resources

- GraphQL Official Documentation
- Apollo Server Documentation
- Apollo Client Documentation
- DataLoader GitHub Repository
- GraphQL Subscriptions
- Schema Design Best Practices
