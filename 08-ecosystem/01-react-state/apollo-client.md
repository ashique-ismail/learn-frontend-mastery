# Apollo Client - Complete State Management for GraphQL

## Overview

Apollo Client is a comprehensive state management library for JavaScript that enables you to manage both local and remote data with GraphQL. It's production-ready and feature-rich with intelligent caching, declarative data fetching, and powerful developer tools.

### Key Features

- **Declarative data fetching**: Query components
- **Normalized caching**: Intelligent cache management
- **Local state management**: Manage local and remote data
- **Optimistic UI**: Update UI before server response
- **Pagination**: Built-in support
- **TypeScript**: Full type generation
- **DevTools**: Debugging and inspection

## Installation

```bash
npm install @apollo/client graphql
```

## Setup

```typescript
import { ApolloClient, InMemoryCache, ApolloProvider } from '@apollo/client';

const client = new ApolloClient({
  uri: 'https://api.example.com/graphql',
  cache: new InMemoryCache(),
});

function App() {
  return (
    <ApolloProvider client={client}>
      <YourApp />
    </ApolloProvider>
  );
}
```

## Queries with useQuery

```typescript
import { gql, useQuery } from '@apollo/client';

const GET_USERS = gql`
  query GetUsers {
    users {
      id
      name
      email
    }
  }
`;

function UserList() {
  const { loading, error, data } = useQuery(GET_USERS);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data.users.map(user => (
        <li key={user.id}>
          {user.name} - {user.email}
        </li>
      ))}
    </ul>
  );
}
```

## Queries with Variables

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

function UserProfile({ userId }: { userId: string }) {
  const { loading, error, data } = useQuery(GET_USER, {
    variables: { id: userId },
  });

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <div>
      <h2>{data.user.name}</h2>
      <p>{data.user.email}</p>
      <h3>Posts</h3>
      <ul>
        {data.user.posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Mutations with useMutation

```typescript
import { gql, useMutation } from '@apollo/client';

const CREATE_POST = gql`
  mutation CreatePost($title: String!, $body: String!) {
    createPost(title: $title, body: $body) {
      id
      title
      body
    }
  }
`;

function CreatePost() {
  const [createPost, { data, loading, error }] = useMutation(CREATE_POST);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    createPost({
      variables: {
        title: 'New Post',
        body: 'Post content',
      },
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Post'}
      </button>
      {error && <p>Error: {error.message}</p>}
      {data && <p>Post created: {data.createPost.title}</p>}
    </form>
  );
}
```

## Cache Updates

### Refetch Queries

```typescript
const [createPost] = useMutation(CREATE_POST, {
  refetchQueries: [
    { query: GET_POSTS },
    { query: GET_USER, variables: { id: userId } },
  ],
});
```

### Update Cache Manually

```typescript
const [createPost] = useMutation(CREATE_POST, {
  update(cache, { data: { createPost } }) {
    const existingPosts = cache.readQuery<{ posts: Post[] }>({
      query: GET_POSTS,
    });

    if (existingPosts) {
      cache.writeQuery({
        query: GET_POSTS,
        data: {
          posts: [...existingPosts.posts, createPost],
        },
      });
    }
  },
});
```

## Optimistic Response

```typescript
const [updatePost] = useMutation(UPDATE_POST, {
  optimisticResponse: {
    updatePost: {
      __typename: 'Post',
      id: postId,
      title: newTitle,
      body: newBody,
    },
  },
});
```

## Pagination

### Offset-based Pagination

```typescript
const GET_POSTS = gql`
  query GetPosts($offset: Int!, $limit: Int!) {
    posts(offset: $offset, limit: $limit) {
      id
      title
    }
  }
`;

function Posts() {
  const [page, setPage] = React.useState(0);
  const limit = 10;

  const { loading, data } = useQuery(GET_POSTS, {
    variables: { offset: page * limit, limit },
  });

  return (
    <div>
      {data?.posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
      <button onClick={() => setPage(p => p - 1)} disabled={page === 0}>
        Previous
      </button>
      <button onClick={() => setPage(p => p + 1)}>
        Next
      </button>
    </div>
  );
}
```

### Cursor-based Pagination

```typescript
const GET_POSTS = gql`
  query GetPosts($cursor: String) {
    posts(after: $cursor, first: 10) {
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

function Posts() {
  const { data, fetchMore } = useQuery(GET_POSTS);

  const loadMore = () => {
    fetchMore({
      variables: {
        cursor: data.posts.pageInfo.endCursor,
      },
      updateQuery: (prev, { fetchMoreResult }) => {
        if (!fetchMoreResult) return prev;
        return {
          posts: {
            ...fetchMoreResult.posts,
            edges: [...prev.posts.edges, ...fetchMoreResult.posts.edges],
          },
        };
      },
    });
  };

  return (
    <div>
      {data?.posts.edges.map(({ node }) => (
        <div key={node.id}>{node.title}</div>
      ))}
      {data?.posts.pageInfo.hasNextPage && (
        <button onClick={loadMore}>Load More</button>
      )}
    </div>
  );
}
```

## Field Policies

```typescript
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        posts: {
          merge(existing = [], incoming) {
            return [...existing, ...incoming];
          },
        },
      },
    },
  },
});
```

## Subscriptions

```typescript
import { useSubscription, gql } from '@apollo/client';

const MESSAGE_SUBSCRIPTION = gql`
  subscription OnMessageAdded {
    messageAdded {
      id
      text
      user {
        name
      }
    }
  }
`;

function Messages() {
  const { data, loading } = useSubscription(MESSAGE_SUBSCRIPTION);

  if (loading) return <p>Loading...</p>;

  return (
    <div>
      <p>New message: {data.messageAdded.text}</p>
    </div>
  );
}
```

## Local State Management

```typescript
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        isLoggedIn: {
          read() {
            return isLoggedInVar();
          },
        },
      },
    },
  },
});

const isLoggedInVar = makeVar(false);

// Usage
function LoginButton() {
  const isLoggedIn = useReactiveVar(isLoggedInVar);

  return (
    <button onClick={() => isLoggedInVar(!isLoggedIn)}>
      {isLoggedIn ? 'Logout' : 'Login'}
    </button>
  );
}
```

## Error Handling

```typescript
import { ApolloError } from '@apollo/client';

function UserProfile() {
  const { data, error } = useQuery(GET_USER);

  if (error) {
    if (error.networkError) {
      return <p>Network error: {error.networkError.message}</p>;
    }
    if (error.graphQLErrors.length > 0) {
      return (
        <div>
          {error.graphQLErrors.map((err, i) => (
            <p key={i}>{err.message}</p>
          ))}
        </div>
      );
    }
  }

  return <div>{data?.user.name}</div>;
}
```

## Polling

```typescript
function LiveData() {
  const { data } = useQuery(GET_DATA, {
    pollInterval: 5000, // Poll every 5 seconds
  });

  return <div>Data: {data?.value}</div>;
}
```

## Lazy Queries

```typescript
import { useLazyQuery } from '@apollo/client';

function SearchUsers() {
  const [search, { loading, data }] = useLazyQuery(SEARCH_USERS);

  return (
    <div>
      <button onClick={() => search({ variables: { query: 'John' } })}>
        Search
      </button>
      {loading && <p>Loading...</p>}
      {data && data.users.map(user => <div key={user.id}>{user.name}</div>)}
    </div>
  );
}
```

## TypeScript with Code Generation

```bash
npm install -D @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-operations @graphql-codegen/typescript-react-apollo
```

```yaml
# codegen.yml
schema: https://api.example.com/graphql
documents: './src/**/*.tsx'
generates:
  ./src/generated/graphql.tsx:
    plugins:
      - typescript
      - typescript-operations
      - typescript-react-apollo
```

```typescript
// Generated hooks
import { useGetUsersQuery, useCreatePostMutation } from './generated/graphql';

function Users() {
  const { data, loading } = useGetUsersQuery();
  // Fully typed!
}
```

## Testing

```typescript
import { MockedProvider } from '@apollo/client/testing';
import { render, waitFor } from '@testing-library/react';

const mocks = [
  {
    request: {
      query: GET_USER,
      variables: { id: '1' },
    },
    result: {
      data: {
        user: { id: '1', name: 'John', email: 'john@example.com' },
      },
    },
  },
];

test('renders user', async () => {
  const { getByText } = render(
    <MockedProvider mocks={mocks}>
      <UserProfile userId="1" />
    </MockedProvider>
  );

  await waitFor(() => {
    expect(getByText('John')).toBeInTheDocument();
  });
});
```

## Common Mistakes

1. **Not normalizing cache**: Use unique IDs
2. **Over-fetching**: Request only needed fields
3. **Missing error handling**: Always handle errors
4. **Not using fragments**: Reuse field selections
5. **Incorrect cache updates**: Update cache properly after mutations

## Best Practices

1. **Use TypeScript with codegen**: Full type safety
2. **Fragments for reusability**: Share field selections
3. **Optimize with field policies**: Configure cache behavior
4. **Handle loading and error states**: Good UX
5. **Use optimistic responses**: Better perceived performance
6. **Leverage Apollo DevTools**: Debug and inspect
7. **Configure cache properly**: Understand cache behavior

## When to Use Apollo Client

### Good Use Cases
- GraphQL APIs
- Complex data requirements
- Real-time subscriptions
- Normalized caching needs
- Type-safe development

### Not Ideal For
- REST APIs (use TanStack Query/SWR)
- Simple data fetching
- When bundle size is critical

## Interview Questions

1. **What is normalized caching?**
   - Storing data by unique IDs to avoid duplication

2. **How does Apollo Client handle mutations?**
   - With useMutation hook and cache updates

3. **What are optimistic responses?**
   - Updating UI before server confirms changes

4. **How do subscriptions work?**
   - WebSocket connection for real-time updates

5. **What is the difference between refetchQueries and update?**
   - refetchQueries re-runs queries; update manually modifies cache

## Key Takeaways

- Apollo Client is comprehensive GraphQL state management
- Normalized caching for efficient data storage
- Declarative data fetching with hooks
- Built-in optimistic UI updates
- Real-time subscriptions support
- Excellent TypeScript integration with codegen
- Powerful DevTools for debugging
- Industry standard for GraphQL in React

## Resources

- [Official Documentation](https://www.apollographql.com/docs/react/)
- [GraphQL Code Generator](https://www.graphql-code-generator.com/)
- [Examples](https://github.com/apollographql/apollo-client/tree/main/examples)
- [DevTools](https://www.apollographql.com/docs/react/development-testing/developer-tooling/)
