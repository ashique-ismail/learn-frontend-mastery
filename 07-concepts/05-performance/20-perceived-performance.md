# Perceived Performance

## Overview

Perceived performance is how fast your application FEELS to users, which can differ significantly from actual performance metrics. Users perceive an app as faster when they see immediate feedback, progressive loading, and meaningful content quickly - even if total load time is the same. This guide covers skeleton screens, optimistic UI updates, loading states, progressive enhancement, and techniques to improve perceived speed.

## The Psychology of Perceived Performance

```
Actual vs Perceived Performance:

App A (Fast metrics, poor perception):
┌────────────────────────────────────────────────────┐
│  [White screen] ... [White screen] ... [Full Page] │
│       2s                2s                1s        │
│  Total: 5s - Feels like 10s (long blank period)   │
└────────────────────────────────────────────────────┘

App B (Same metrics, better perception):
┌────────────────────────────────────────────────────┐
│  [Skeleton] → [Partial Content] → [Full Page]      │
│      1s             2s                  2s          │
│  Total: 5s - Feels like 3s (progressive loading)   │
└────────────────────────────────────────────────────┘

Key Principles:
1. Show something immediately (< 100ms)
2. Make it meaningful (not just spinners)
3. Show progress (skeleton → partial → complete)
4. Provide instant feedback to interactions
5. Optimize for "time to interactive content"
```

## Skeleton Screens

Skeleton screens show content placeholders while loading:

### Basic Skeleton Component (React)

```typescript
// Skeleton.tsx
import React from 'react';
import './Skeleton.css';

interface SkeletonProps {
  width?: string | number;
  height?: string | number;
  borderRadius?: string | number;
  className?: string;
}

export const Skeleton: React.FC<SkeletonProps> = ({
  width = '100%',
  height = '1rem',
  borderRadius = '4px',
  className = ''
}) => {
  return (
    <div
      className={`skeleton ${className}`}
      style={{
        width,
        height,
        borderRadius
      }}
    />
  );
};

// Skeleton.css
.skeleton {
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 200% 100%;
  animation: loading 1.5s ease-in-out infinite;
}

@keyframes loading {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}

// Dark mode support
@media (prefers-color-scheme: dark) {
  .skeleton {
    background: linear-gradient(
      90deg,
      #2a2a2a 25%,
      #3a3a3a 50%,
      #2a2a2a 75%
    );
  }
}
```

### Skeleton Screen Templates

```typescript
// CardSkeleton.tsx
export const CardSkeleton: React.FC = () => {
  return (
    <div className="card-skeleton">
      <Skeleton height={200} borderRadius={8} />
      <div style={{ padding: '16px' }}>
        <Skeleton width="60%" height={24} />
        <Skeleton width="90%" height={16} style={{ marginTop: 8 }} />
        <Skeleton width="70%" height={16} style={{ marginTop: 4 }} />
        <div style={{ marginTop: 16, display: 'flex', gap: '8px' }}>
          <Skeleton width={80} height={32} borderRadius={16} />
          <Skeleton width={80} height={32} borderRadius={16} />
        </div>
      </div>
    </div>
  );
};

// ListSkeleton.tsx
export const ListSkeleton: React.FC<{ count?: number }> = ({ count = 5 }) => {
  return (
    <div className="list-skeleton">
      {Array.from({ length: count }).map((_, i) => (
        <div key={i} className="list-item-skeleton">
          <Skeleton width={48} height={48} borderRadius="50%" />
          <div style={{ flex: 1, marginLeft: 12 }}>
            <Skeleton width="40%" height={16} />
            <Skeleton width="80%" height={14} style={{ marginTop: 8 }} />
          </div>
        </div>
      ))}
    </div>
  );
};

// ProfileSkeleton.tsx
export const ProfileSkeleton: React.FC = () => {
  return (
    <div className="profile-skeleton">
      <Skeleton width={120} height={120} borderRadius="50%" />
      <Skeleton width={200} height={28} style={{ marginTop: 16 }} />
      <Skeleton width={150} height={16} style={{ marginTop: 8 }} />
      <div style={{ marginTop: 24, display: 'flex', gap: '16px' }}>
        <Skeleton width={100} height={40} />
        <Skeleton width={100} height={40} />
      </div>
    </div>
  );
};
```

### React Suspense with Skeleton

```typescript
// UserProfile.tsx
import React, { Suspense } from 'react';
import { ProfileSkeleton } from './ProfileSkeleton';

const ProfileData = React.lazy(() => import('./ProfileData'));

export const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <ProfileData userId={userId} />
    </Suspense>
  );
};

// With data fetching
function useUser(userId: string) {
  const [user, setUser] = React.useState(null);
  const [loading, setLoading] = React.useState(true);
  
  React.useEffect(() => {
    fetchUser(userId).then(data => {
      setUser(data);
      setLoading(false);
    });
  }, [userId]);
  
  return { user, loading };
}

function UserProfile({ userId }: { userId: string }) {
  const { user, loading } = useUser(userId);
  
  if (loading) return <ProfileSkeleton />;
  
  return <div>User data: {user.name}</div>;
}
```

### Angular Skeleton Directive

```typescript
// skeleton.directive.ts
import { Directive, ElementRef, Input, OnInit, Renderer2 } from '@angular/core';

@Directive({
  selector: '[appSkeleton]'
})
export class SkeletonDirective implements OnInit {
  @Input() appSkeleton: 'text' | 'circle' | 'rect' = 'text';
  @Input() width?: string;
  @Input() height?: string;
  
  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}
  
  ngOnInit(): void {
    this.renderer.addClass(this.el.nativeElement, 'skeleton');
    this.renderer.addClass(this.el.nativeElement, `skeleton-${this.appSkeleton}`);
    
    if (this.width) {
      this.renderer.setStyle(this.el.nativeElement, 'width', this.width);
    }
    
    if (this.height) {
      this.renderer.setStyle(this.el.nativeElement, 'height', this.height);
    }
  }
}

// skeleton.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-skeleton',
  template: `
    <div
      class="skeleton"
      [class.skeleton-circle]="shape === 'circle'"
      [class.skeleton-rect]="shape === 'rect'"
      [style.width]="width"
      [style.height]="height"
      [style.borderRadius]="borderRadius"
    ></div>
  `,
  styles: [`
    .skeleton {
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 200% 100%;
      animation: loading 1.5s ease-in-out infinite;
    }
    
    .skeleton-circle {
      border-radius: 50%;
    }
    
    @keyframes loading {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class SkeletonComponent {
  @Input() width = '100%';
  @Input() height = '1rem';
  @Input() borderRadius = '4px';
  @Input() shape: 'text' | 'circle' | 'rect' = 'text';
}

// Usage in template
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="loading">
      <div *ngFor="let item of [1,2,3,4,5]" class="list-item">
        <app-skeleton shape="circle" width="48px" height="48px"></app-skeleton>
        <div style="flex: 1; margin-left: 12px;">
          <app-skeleton width="40%" height="16px"></app-skeleton>
          <app-skeleton width="80%" height="14px" style="margin-top: 8px;"></app-skeleton>
        </div>
      </div>
    </div>
    
    <div *ngIf="!loading">
      <div *ngFor="let user of users" class="list-item">
        <!-- Real content -->
      </div>
    </div>
  `
})
export class UserListComponent {
  loading = true;
  users: User[] = [];
  
  ngOnInit(): void {
    this.fetchUsers().then(users => {
      this.users = users;
      this.loading = false;
    });
  }
}
```

## Optimistic UI

Update UI immediately, assume success:

```typescript
// Optimistic Update Hook
import { useState, useCallback } from 'react';

interface OptimisticState<T> {
  data: T[];
  add: (item: T) => Promise<void>;
  remove: (id: string) => Promise<void>;
  update: (id: string, updates: Partial<T>) => Promise<void>;
}

export function useOptimistic<T extends { id: string }>(
  initialData: T[],
  api: {
    add: (item: T) => Promise<T>;
    remove: (id: string) => Promise<void>;
    update: (id: string, updates: Partial<T>) => Promise<T>;
  }
): OptimisticState<T> {
  const [data, setData] = useState<T[]>(initialData);
  
  const add = useCallback(async (item: T) => {
    // Optimistic update
    setData(prev => [...prev, item]);
    
    try {
      await api.add(item);
    } catch (error) {
      // Rollback on error
      setData(prev => prev.filter(i => i.id !== item.id));
      throw error;
    }
  }, [api]);
  
  const remove = useCallback(async (id: string) => {
    // Store for rollback
    const item = data.find(i => i.id === id);
    
    // Optimistic remove
    setData(prev => prev.filter(i => i.id !== id));
    
    try {
      await api.remove(id);
    } catch (error) {
      // Rollback
      if (item) {
        setData(prev => [...prev, item]);
      }
      throw error;
    }
  }, [data, api]);
  
  const update = useCallback(async (id: string, updates: Partial<T>) => {
    // Store original for rollback
    const original = data.find(i => i.id === id);
    
    // Optimistic update
    setData(prev =>
      prev.map(item =>
        item.id === id ? { ...item, ...updates } : item
      )
    );
    
    try {
      await api.update(id, updates);
    } catch (error) {
      // Rollback
      if (original) {
        setData(prev =>
          prev.map(item => (item.id === id ? original : item))
        );
      }
      throw error;
    }
  }, [data, api]);
  
  return { data, add, remove, update };
}

// Usage
function TodoList() {
  const { data: todos, add, remove, update } = useOptimistic(
    initialTodos,
    {
      add: (todo) => fetch('/api/todos', { method: 'POST', body: JSON.stringify(todo) }),
      remove: (id) => fetch(`/api/todos/${id}`, { method: 'DELETE' }),
      update: (id, updates) => fetch(`/api/todos/${id}`, { method: 'PATCH', body: JSON.stringify(updates) })
    }
  );
  
  const handleAdd = async () => {
    try {
      await add({ id: Date.now().toString(), text: 'New todo', done: false });
      // UI already updated!
    } catch (error) {
      toast.error('Failed to add todo');
      // UI already rolled back!
    }
  };
  
  const handleToggle = async (id: string, done: boolean) => {
    try {
      await update(id, { done: !done });
      // Instant UI feedback!
    } catch (error) {
      toast.error('Failed to update');
    }
  };
  
  return (
    <div>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={() => handleToggle(todo.id, todo.done)}
        />
      ))}
    </div>
  );
}
```

### React Query Optimistic Updates

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useTodoMutations() {
  const queryClient = useQueryClient();
  
  const addTodo = useMutation({
    mutationFn: (todo: Todo) => fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify(todo)
    }),
    
    // Optimistic update
    onMutate: async (newTodo) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      
      // Snapshot previous value
      const previousTodos = queryClient.getQueryData(['todos']);
      
      // Optimistically update
      queryClient.setQueryData(['todos'], (old: Todo[]) => [...old, newTodo]);
      
      // Return context with snapshot
      return { previousTodos };
    },
    
    // Rollback on error
    onError: (err, newTodo, context) => {
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    
    // Refetch on success or error
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    }
  });
  
  return { addTodo };
}

// Usage
function TodoList() {
  const { data: todos } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos });
  const { addTodo } = useTodoMutations();
  
  const handleAdd = () => {
    addTodo.mutate({
      id: Date.now().toString(),
      text: 'New todo',
      done: false
    });
    // UI updates instantly!
  };
  
  return (
    <div>
      <button onClick={handleAdd}>Add Todo</button>
      {todos?.map(todo => <TodoItem key={todo.id} todo={todo} />)}
    </div>
  );
}
```

## Loading States

Provide meaningful feedback during operations:

### Progressive Loading States

```typescript
// LoadingButton.tsx
import React from 'react';

interface LoadingButtonProps {
  onClick: () => Promise<void>;
  children: React.ReactNode;
  successMessage?: string;
  errorMessage?: string;
}

type ButtonState = 'idle' | 'loading' | 'success' | 'error';

export const LoadingButton: React.FC<LoadingButtonProps> = ({
  onClick,
  children,
  successMessage = 'Success!',
  errorMessage = 'Error'
}) => {
  const [state, setState] = React.useState<ButtonState>('idle');
  
  const handleClick = async () => {
    setState('loading');
    
    try {
      await onClick();
      setState('success');
      setTimeout(() => setState('idle'), 2000);
    } catch (error) {
      setState('error');
      setTimeout(() => setState('idle'), 2000);
    }
  };
  
  const content = {
    idle: children,
    loading: (
      <>
        <Spinner size="small" />
        <span style={{ marginLeft: 8 }}>Loading...</span>
      </>
    ),
    success: (
      <>
        <CheckIcon />
        <span style={{ marginLeft: 8 }}>{successMessage}</span>
      </>
    ),
    error: (
      <>
        <ErrorIcon />
        <span style={{ marginLeft: 8 }}>{errorMessage}</span>
      </>
    )
  };
  
  return (
    <button
      onClick={handleClick}
      disabled={state !== 'idle'}
      className={`loading-button loading-button--${state}`}
    >
      {content[state]}
    </button>
  );
};

// Usage
function SubmitForm() {
  const handleSubmit = async () => {
    await submitFormData();
  };
  
  return (
    <LoadingButton
      onClick={handleSubmit}
      successMessage="Form submitted!"
      errorMessage="Submission failed"
    >
      Submit
    </LoadingButton>
  );
}
```

### Content Loading with Phases

```typescript
// ContentLoader.tsx
import React from 'react';

type LoadingPhase = 'initial' | 'loading' | 'partial' | 'complete' | 'error';

interface ContentLoaderProps {
  children: React.ReactNode;
  skeleton: React.ReactNode;
  partialContent?: React.ReactNode;
  error?: React.ReactNode;
  phase: LoadingPhase;
}

export const ContentLoader: React.FC<ContentLoaderProps> = ({
  children,
  skeleton,
  partialContent,
  error,
  phase
}) => {
  switch (phase) {
    case 'initial':
    case 'loading':
      return <>{skeleton}</>;
      
    case 'partial':
      return (
        <div className="content-loader--partial">
          {partialContent}
          <div className="content-loader__spinner">
            <Spinner />
          </div>
        </div>
      );
      
    case 'complete':
      return <>{children}</>;
      
    case 'error':
      return <>{error || <DefaultError />}</>;
      
    default:
      return <>{skeleton}</>;
  }
};

// Usage with progressive loading
function ArticlePage({ articleId }: { articleId: string }) {
  const [phase, setPhase] = React.useState<LoadingPhase>('initial');
  const [article, setArticle] = React.useState<Article | null>(null);
  const [comments, setComments] = React.useState<Comment[]>([]);
  
  React.useEffect(() => {
    // Phase 1: Loading
    setPhase('loading');
    
    // Phase 2: Load critical content first
    fetchArticle(articleId).then(data => {
      setArticle(data);
      setPhase('partial'); // Show article, loading comments
      
      // Phase 3: Load secondary content
      fetchComments(articleId).then(comments => {
        setComments(comments);
        setPhase('complete');
      });
    }).catch(() => {
      setPhase('error');
    });
  }, [articleId]);
  
  return (
    <ContentLoader
      phase={phase}
      skeleton={<ArticleSkeleton />}
      partialContent={
        article && (
          <>
            <ArticleContent article={article} />
            <CommentsSkeleton />
          </>
        )
      }
      error={<ErrorMessage />}
    >
      {article && (
        <>
          <ArticleContent article={article} />
          <Comments comments={comments} />
        </>
      )}
    </ContentLoader>
  );
}
```

## Progressive Enhancement

Build core functionality first, enhance progressively:

```typescript
// Progressive Image Loading
interface ProgressiveImageProps {
  src: string;
  placeholder: string;
  alt: string;
}

export const ProgressiveImage: React.FC<ProgressiveImageProps> = ({
  src,
  placeholder,
  alt
}) => {
  const [currentSrc, setCurrentSrc] = React.useState(placeholder);
  const [loading, setLoading] = React.useState(true);
  
  React.useEffect(() => {
    // Preload full image
    const img = new Image();
    img.src = src;
    
    img.onload = () => {
      setCurrentSrc(src);
      setLoading(false);
    };
  }, [src]);
  
  return (
    <div className="progressive-image">
      <img
        src={currentSrc}
        alt={alt}
        className={loading ? 'progressive-image--loading' : 'progressive-image--loaded'}
        style={{
          filter: loading ? 'blur(10px)' : 'none',
          transition: 'filter 0.3s'
        }}
      />
    </div>
  );
};

// BlurHash implementation
import { Blurhash } from 'react-blurhash';

interface BlurHashImageProps {
  src: string;
  blurHash: string;
  width: number;
  height: number;
  alt: string;
}

export const BlurHashImage: React.FC<BlurHashImageProps> = ({
  src,
  blurHash,
  width,
  height,
  alt
}) => {
  const [loaded, setLoaded] = React.useState(false);
  
  return (
    <div style={{ position: 'relative', width, height }}>
      {!loaded && (
        <Blurhash
          hash={blurHash}
          width={width}
          height={height}
          resolutionX={32}
          resolutionY={32}
          punch={1}
        />
      )}
      <img
        src={src}
        alt={alt}
        onLoad={() => setLoaded(true)}
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: '100%',
          opacity: loaded ? 1 : 0,
          transition: 'opacity 0.3s'
        }}
      />
    </div>
  );
};
```

### Progressive Form Loading

```typescript
// Load form fields progressively
function ProgressiveForm() {
  const [fieldsLoaded, setFieldsLoaded] = React.useState(0);
  const totalFields = 5;
  
  React.useEffect(() => {
    // Load fields one by one
    const interval = setInterval(() => {
      setFieldsLoaded(prev => {
        if (prev >= totalFields) {
          clearInterval(interval);
          return prev;
        }
        return prev + 1;
      });
    }, 100);
    
    return () => clearInterval(interval);
  }, []);
  
  return (
    <form>
      {fieldsLoaded >= 1 ? (
        <input type="text" placeholder="Name" />
      ) : (
        <Skeleton height={40} />
      )}
      
      {fieldsLoaded >= 2 ? (
        <input type="email" placeholder="Email" />
      ) : (
        <Skeleton height={40} />
      )}
      
      {fieldsLoaded >= 3 ? (
        <input type="tel" placeholder="Phone" />
      ) : (
        <Skeleton height={40} />
      )}
      
      {fieldsLoaded >= 4 ? (
        <textarea placeholder="Message" />
      ) : (
        <Skeleton height={120} />
      )}
      
      {fieldsLoaded >= 5 ? (
        <button type="submit">Submit</button>
      ) : (
        <Skeleton height={40} />
      )}
    </form>
  );
}
```

## Instant Feedback Techniques

### Micro-interactions

```typescript
// ButtonWithFeedback.tsx
export const ButtonWithFeedback: React.FC<{
  onClick: () => void;
  children: React.ReactNode;
}> = ({ onClick, children }) => {
  const [ripples, setRipples] = React.useState<Array<{ x: number; y: number; id: number }>>([]);
  
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    const rect = e.currentTarget.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    const id = Date.now();
    setRipples(prev => [...prev, { x, y, id }]);
    
    // Remove ripple after animation
    setTimeout(() => {
      setRipples(prev => prev.filter(r => r.id !== id));
    }, 600);
    
    onClick();
  };
  
  return (
    <button className="button-with-feedback" onClick={handleClick}>
      {ripples.map(ripple => (
        <span
          key={ripple.id}
          className="ripple"
          style={{
            left: ripple.x,
            top: ripple.y
          }}
        />
      ))}
      {children}
    </button>
  );
};

// CSS
.button-with-feedback {
  position: relative;
  overflow: hidden;
}

.ripple {
  position: absolute;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.6);
  width: 0;
  height: 0;
  transform: translate(-50%, -50%);
  animation: ripple-animation 0.6s ease-out;
}

@keyframes ripple-animation {
  to {
    width: 200px;
    height: 200px;
    opacity: 0;
  }
}
```

### Instant Search Feedback

```typescript
// InstantSearch.tsx
function InstantSearch() {
  const [query, setQuery] = React.useState('');
  const [results, setResults] = React.useState<Result[]>([]);
  const [searching, setSearching] = React.useState(false);
  
  const debouncedQuery = useDebounce(query, 300);
  
  React.useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }
    
    setSearching(true);
    
    searchAPI(debouncedQuery).then(data => {
      setResults(data);
      setSearching(false);
    });
  }, [debouncedQuery]);
  
  return (
    <div className="instant-search">
      <div className="search-input-wrapper">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search..."
        />
        {searching && <Spinner size="small" />}
      </div>
      
      {query && (
        <div className="search-results">
          {searching ? (
            // Show skeleton while searching
            <ResultsSkeleton count={5} />
          ) : results.length > 0 ? (
            results.map(result => (
              <ResultItem key={result.id} result={result} />
            ))
          ) : (
            <div>No results found</div>
          )}
        </div>
      )}
    </div>
  );
}
```

## Common Mistakes

### 1. Blank Loading States

```typescript
// ❌ Bad: Just a spinner
function BadLoading() {
  const { data, loading } = useData();
  
  if (loading) return <Spinner />;
  
  return <Content data={data} />;
}

// ✅ Good: Skeleton that matches content shape
function GoodLoading() {
  const { data, loading } = useData();
  
  if (loading) return <ContentSkeleton />;
  
  return <Content data={data} />;
}
```

### 2. No Instant Feedback

```typescript
// ❌ Bad: No feedback on click
<button onClick={handleSave}>Save</button>

// ✅ Good: Instant feedback, then async operation
<LoadingButton onClick={handleSave}>Save</LoadingButton>
```

### 3. Not Using Optimistic Updates

```typescript
// ❌ Bad: Wait for server response
const handleLike = async () => {
  const result = await likePost(postId);
  setLikes(result.likes); // Delay before UI updates
};

// ✅ Good: Update immediately
const handleLike = async () => {
  setLikes(prev => prev + 1); // Instant feedback
  try {
    await likePost(postId);
  } catch {
    setLikes(prev => prev - 1); // Rollback on error
  }
};
```

### 4. Jarring Skeleton Transitions

```typescript
// ❌ Bad: Abrupt skeleton → content transition
<div>{loading ? <Skeleton /> : <Content />}</div>

// ✅ Good: Smooth fade transition
<div className={loading ? 'loading' : 'loaded'}>
  {loading ? <Skeleton /> : <Content />}
</div>

// CSS
.loading, .loaded {
  transition: opacity 0.3s;
}
```

### 5. Too Many Loading Indicators

```typescript
// ❌ Bad: Spinner for every request
function BadPage() {
  const { data1, loading1 } = useData1();
  const { data2, loading2 } = useData2();
  const { data3, loading3 } = useData3();
  
  if (loading1 || loading2 || loading3) return <Spinner />;
  
  return <Content />;
}

// ✅ Good: Progressive loading
function GoodPage() {
  const { data1, loading1 } = useData1();
  const { data2, loading2 } = useData2();
  const { data3, loading3 } = useData3();
  
  return (
    <>
      {loading1 ? <Skeleton1 /> : <Content1 data={data1} />}
      {loading2 ? <Skeleton2 /> : <Content2 data={data2} />}
      {loading3 ? <Skeleton3 /> : <Content3 data={data3} />}
    </>
  );
}
```

## Best Practices

1. **Show content structure immediately** - skeleton screens over blank pages
2. **Use optimistic updates** - instant feedback for user actions
3. **Progressive loading** - show critical content first, secondary content later
4. **Meaningful loading states** - match content shape, not generic spinners
5. **Instant interaction feedback** - ripples, state changes, micro-interactions
6. **Smooth transitions** - fade between skeleton and real content
7. **Progressive image loading** - blur-up or low-quality placeholders
8. **Debounce search** - with instant visual feedback
9. **Prioritize above-the-fold** - load visible content first
10. **Measure perceived performance** - user surveys, session recordings

## When to Use

### Use Skeleton Screens
- Initial page load
- Route transitions
- Lazy-loaded components
- Data fetching for content-heavy pages

### Use Optimistic UI
- Like/favorite actions
- Form submissions
- CRUD operations
- Real-time features (chat, collaborative editing)

### Use Progressive Loading
- Long lists (infinite scroll)
- Multi-section pages
- Dashboard with multiple data sources
- Media-heavy content

### Don't Overuse
- Very fast operations (<100ms) - no need for loading state
- Background operations - unless user needs feedback
- Preloaded content - skeleton unnecessary

## Interview Questions

### 1. What is perceived performance and why does it matter?

**Answer**: Perceived performance is how fast users FEEL an application is, which can differ from actual metrics. It matters because user perception drives satisfaction - an app that feels fast keeps users engaged even if load times are similar. Techniques like skeleton screens, optimistic updates, and progressive loading improve perceived speed by providing immediate feedback and showing content progressively rather than all-at-once after a blank screen.

### 2. Explain the benefits of skeleton screens over traditional loading spinners.

**Answer**: Skeleton screens show the structure of content being loaded, providing context and reducing perceived wait time. Benefits: (1) Users understand what's loading, (2) Maintains layout stability (no layout shift), (3) Feels 20-30% faster than spinners, (4) Reduces bounce rate, (5) Provides visual continuity. Spinners are generic and give no context, making waits feel longer. Skeletons match final content shape, preparing users mentally for what's coming.

### 3. What are optimistic updates and when should you use them?

**Answer**: Optimistic updates assume success and update the UI immediately before the server responds, then rollback on error. Use for: actions with high success rates (likes, favorites, toggles), form submissions, CRUD operations, real-time features. Don't use for: critical operations (payments), destructive actions without confirmation, unreliable networks. Implementation requires storing previous state for rollback and proper error handling with user feedback.

### 4. How do you implement rollback for failed optimistic updates?

**Answer**: Store the previous state before the optimistic update, apply the update immediately, attempt the async operation, and restore previous state on error with user notification. Example: saving previous todo list, adding item optimistically, if API fails, restore previous list and show error toast. Can use libraries like React Query (onMutate/onError) or custom hooks with snapshot/restore pattern.

### 5. What is progressive enhancement in the context of performance?

**Answer**: Progressive enhancement builds core functionality first (works on all devices/networks), then adds enhancements for capable browsers/fast connections. For performance: load critical content first (text, structure), then images, then interactive features. Examples: load article text immediately then comments, show low-quality image placeholder then high-res, basic form fields then enhanced validation. Ensures usability even on slow connections.

### 6. How would you measure perceived performance?

**Answer**: 
1. User surveys (how fast did it feel?)
2. Session recordings to see user behavior during loading
3. Custom metrics: time to interactive content, skeleton to content time
4. A/B testing different loading strategies
5. Bounce rate during loading states
6. User engagement metrics (scroll depth, clicks during loading)
7. Core Web Vitals (FCP, LCP, CLS)

### 7. What is the optimal delay before showing a loading indicator?

**Answer**: Show loading indicators only if operation takes >100-200ms. Below 100ms, loading indicators feel like flicker and actually degrade experience. Implementation: use a delay timer - start operation, wait 100ms, if still loading then show indicator. This prevents flashing indicators for fast operations while providing feedback for slow ones. Some recommend 300ms for spinners, immediate for skeletons.

### 8. Explain the psychology behind why skeleton screens feel faster than spinners.

**Answer**: Skeleton screens leverage psychological principles: (1) Perceived progress - users see structure forming, (2) Reduced uncertainty - know what's coming, (3) Maintained context - no blank void, (4) Attention distribution - eyes can scan structure vs fixating on spinner, (5) Priming - brain prepares to process content. Spinners create anxiety (how long?), skeletons create anticipation (content is coming). Studies show 20-30% improvement in perceived speed.

## Key Takeaways

1. **Perceived performance often matters more than actual metrics** - users judge by feel
2. **Show something meaningful immediately** - skeleton screens over blank pages/spinners
3. **Optimistic updates provide instant feedback** - assume success, rollback on error
4. **Progressive loading reduces perceived wait** - critical content first, secondary later
5. **Smooth transitions prevent jarring experiences** - fade between skeleton and content
6. **Match skeleton to content shape** - provides context and reduces layout shift
7. **Instant feedback for all interactions** - ripples, state changes, micro-interactions
8. **100ms threshold for loading indicators** - don't show for faster operations
9. **Progressive image loading improves UX** - blur-up or low-quality placeholders first
10. **Measure perceived performance with user metrics** - surveys, engagement, bounce rate

## Resources

- [Skeleton Screens (Luke Wroblewski)](https://www.lukew.com/ff/entry.asp?1797)
- [Optimistic UI Updates (Apollo)](https://www.apollographql.com/docs/react/performance/optimistic-ui/)
- [Perceived Performance (MDN)](https://developer.mozilla.org/en-US/docs/Learn/Performance/Perceived_performance)
- [Psychology of Waiting (web.dev)](https://web.dev/rail/)
- [Progressive Loading (Smashing Magazine)](https://www.smashingmagazine.com/2016/02/preload-what-is-it-good-for/)
- [BlurHash](https://blurha.sh/)
- [React Suspense](https://react.dev/reference/react/Suspense)
