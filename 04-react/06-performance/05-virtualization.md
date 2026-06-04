# Virtualization in React

## Overview

Virtualization (or windowing) is a technique to render only visible items in large lists, dramatically improving performance when dealing with thousands of items. This guide covers TanStack Virtual (formerly react-virtual), react-window, and best practices for implementing virtualization in React applications.

## Table of Contents

1. [Why Virtualization](#why-virtualization)
2. [TanStack Virtual](#tanstack-virtual)
3. [React Window](#react-window)
4. [Variable Size Lists](#variable-size-lists)
5. [Grid Virtualization](#grid-virtualization)
6. [Infinite Scrolling](#infinite-scrolling)
7. [Performance Considerations](#performance-considerations)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Real-World Use Cases](#real-world-use-cases)
11. [Key Takeaways](#key-takeaways)
12. [Resources](#resources)

## Why Virtualization

Rendering thousands of DOM elements causes performance issues:

### The Problem

```typescript
import React from 'react';

interface Item {
  id: number;
  name: string;
}

// BAD: Rendering 10,000 items at once
function LargeList({ items }: { items: Item[] }) {
  // All 10,000 DOM nodes created immediately
  return (
    <div style={{ height: '600px', overflow: 'auto' }}>
      {items.map(item => (
        <div key={item.id} style={{ height: '50px', padding: '10px' }}>
          {item.name}
        </div>
      ))}
    </div>
  );
}

// Problems:
// - Initial render: ~5 seconds for 10,000 items
// - Memory: ~500MB for DOM nodes
// - Scrolling: Janky, dropped frames
// - React DevTools: Freezes trying to display tree
```

### The Solution

```typescript
// GOOD: Only render visible items
function VirtualizedList({ items }: { items: Item[] }) {
  // Only ~15 DOM nodes for visible items
  // Even with 100,000 items
  
  return (
    <VirtualList
      height={600}
      itemCount={items.length}
      itemSize={50}
    >
      {({ index, style }) => (
        <div style={style}>
          {items[index].name}
        </div>
      )}
    </VirtualList>
  );
}

// Benefits:
// - Initial render: <100ms
// - Memory: ~5MB
// - Scrolling: Smooth 60fps
// - React DevTools: Fast and responsive
```

## TanStack Virtual

Modern, flexible virtualization library:

### Basic Setup

```bash
npm install @tanstack/react-virtual
```

```typescript
import React from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Item {
  id: number;
  name: string;
}

function VirtualList({ items }: { items: Item[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50, // Estimated item height
    overscan: 5, // Render 5 extra items above/below viewport
  });
  
  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <div style={{ padding: '10px' }}>
              {items[virtualItem.index].name}
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Dynamic Size Items

```typescript
import React from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Post {
  id: number;
  title: string;
  content: string;
}

function DynamicList({ posts }: { posts: Post[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: posts.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100, // Initial estimate
    // Measure actual size after render
    measureElement: (element) => element.getBoundingClientRect().height,
  });
  
  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            data-index={virtualItem.index}
            ref={virtualizer.measureElement}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <div style={{ padding: '20px', borderBottom: '1px solid #ddd' }}>
              <h3>{posts[virtualItem.index].title}</h3>
              <p>{posts[virtualItem.index].content}</p>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Horizontal Virtualization

```typescript
import React from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Image {
  id: number;
  url: string;
  title: string;
}

function HorizontalGallery({ images }: { images: Image[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    horizontal: true, // Enable horizontal mode
    count: images.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 200,
  });
  
  return (
    <div
      ref={parentRef}
      style={{
        width: '100%',
        height: '300px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          width: `${virtualizer.getTotalSize()}px`,
          height: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              height: '100%',
              width: `${virtualItem.size}px`,
              transform: `translateX(${virtualItem.start}px)`,
            }}
          >
            <img
              src={images[virtualItem.index].url}
              alt={images[virtualItem.index].title}
              style={{ width: '100%', height: '100%', objectFit: 'cover' }}
            />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Scroll to Item

```typescript
import React, { useState } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function ScrollableList({ items }: { items: string[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  const [scrollToIndex, setScrollToIndex] = useState('');
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });
  
  const scrollToItem = () => {
    const index = parseInt(scrollToIndex);
    if (!isNaN(index) && index >= 0 && index < items.length) {
      virtualizer.scrollToIndex(index, {
        align: 'start', // 'start' | 'center' | 'end' | 'auto'
        behavior: 'smooth', // 'auto' | 'smooth'
      });
    }
  };
  
  return (
    <div>
      <div>
        <input
          type="number"
          value={scrollToIndex}
          onChange={e => setScrollToIndex(e.target.value)}
          placeholder="Index to scroll to"
        />
        <button onClick={scrollToItem}>Scroll</button>
      </div>
      
      <div
        ref={parentRef}
        style={{
          height: '600px',
          overflow: 'auto',
        }}
      >
        <div
          style={{
            height: `${virtualizer.getTotalSize()}px`,
            width: '100%',
            position: 'relative',
          }}
        >
          {virtualizer.getVirtualItems().map(virtualItem => (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <div style={{ padding: '10px' }}>
                Item {virtualItem.index}: {items[virtualItem.index]}
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

## React Window

Lightweight alternative to TanStack Virtual:

### Fixed Size List

```bash
npm install react-window
```

```typescript
import React from 'react';
import { FixedSizeList } from 'react-window';

interface Item {
  id: number;
  name: string;
}

function SimpleList({ items }: { items: Item[] }) {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <div style={{ padding: '10px' }}>
        {items[index].name}
      </div>
    </div>
  );
  
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

### Variable Size List

```typescript
import React from 'react';
import { VariableSizeList } from 'react-window';

interface Post {
  id: number;
  title: string;
  content: string;
}

function VariableList({ posts }: { posts: Post[] }) {
  const listRef = React.useRef<VariableSizeList>(null);
  
  // Calculate item size based on content
  const getItemSize = (index: number) => {
    const post = posts[index];
    const titleLines = Math.ceil(post.title.length / 50);
    const contentLines = Math.ceil(post.content.length / 80);
    return (titleLines * 20) + (contentLines * 16) + 40; // padding
  };
  
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <div style={{ padding: '20px', borderBottom: '1px solid #ddd' }}>
        <h3>{posts[index].title}</h3>
        <p>{posts[index].content}</p>
      </div>
    </div>
  );
  
  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      itemCount={posts.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}
```

### Fixed Size Grid

```typescript
import React from 'react';
import { FixedSizeGrid } from 'react-window';

interface GridItem {
  id: number;
  color: string;
}

function Grid({ items }: { items: GridItem[][] }) {
  const Cell = ({ 
    columnIndex, 
    rowIndex, 
    style 
  }: { 
    columnIndex: number;
    rowIndex: number;
    style: React.CSSProperties;
  }) => (
    <div
      style={{
        ...style,
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        border: '1px solid #ddd',
        backgroundColor: items[rowIndex][columnIndex].color,
      }}
    >
      {items[rowIndex][columnIndex].id}
    </div>
  );
  
  return (
    <FixedSizeGrid
      columnCount={items[0].length}
      columnWidth={100}
      height={600}
      rowCount={items.length}
      rowHeight={100}
      width={800}
    >
      {Cell}
    </FixedSizeGrid>
  );
}
```

## Variable Size Lists

Handle items with different heights:

### Auto-Sizing Content

```typescript
import React from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Message {
  id: number;
  author: string;
  text: string;
  timestamp: Date;
}

function ChatMessages({ messages }: { messages: Message[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    measureElement: (element) => element.getBoundingClientRect().height,
  });
  
  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => {
          const message = messages[virtualItem.index];
          
          return (
            <div
              key={virtualItem.key}
              data-index={virtualItem.index}
              ref={virtualizer.measureElement}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <div style={{ 
                padding: '15px', 
                borderBottom: '1px solid #eee' 
              }}>
                <div style={{ fontWeight: 'bold' }}>
                  {message.author}
                </div>
                <div style={{ marginTop: '5px' }}>
                  {message.text}
                </div>
                <div style={{ 
                  fontSize: '12px', 
                  color: '#666', 
                  marginTop: '5px' 
                }}>
                  {message.timestamp.toLocaleString()}
                </div>
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### Images with Loading

```typescript
import React, { useState } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Photo {
  id: number;
  url: string;
  caption: string;
}

function PhotoGallery({ photos }: { photos: Photo[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  const [loadedImages, setLoadedImages] = useState<Set<number>>(new Set());
  
  const virtualizer = useVirtualizer({
    count: photos.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 300,
    measureElement: (element) => element.getBoundingClientRect().height,
  });
  
  const handleImageLoad = (index: number) => {
    setLoadedImages(prev => new Set(prev).add(index));
    virtualizer.measure(); // Remeasure after image loads
  };
  
  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => {
          const photo = photos[virtualItem.index];
          
          return (
            <div
              key={virtualItem.key}
              data-index={virtualItem.index}
              ref={virtualizer.measureElement}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <div style={{ padding: '10px' }}>
                <img
                  src={photo.url}
                  alt={photo.caption}
                  onLoad={() => handleImageLoad(virtualItem.index)}
                  style={{ 
                    width: '100%', 
                    display: 'block',
                    backgroundColor: '#f0f0f0',
                  }}
                />
                <p>{photo.caption}</p>
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## Grid Virtualization

Virtualize in two dimensions:

### TanStack Virtual Grid

```typescript
import React from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Cell {
  row: number;
  col: number;
  value: string;
}

function VirtualizedGrid({ 
  rows, 
  cols 
}: { 
  rows: number; 
  cols: number;
}) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const rowVirtualizer = useVirtualizer({
    count: rows,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });
  
  const columnVirtualizer = useVirtualizer({
    horizontal: true,
    count: cols,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 150,
  });
  
  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        width: '100%',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: `${columnVirtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map(virtualRow => (
          <React.Fragment key={virtualRow.key}>
            {columnVirtualizer.getVirtualItems().map(virtualColumn => (
              <div
                key={virtualColumn.key}
                style={{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: `${virtualColumn.size}px`,
                  height: `${virtualRow.size}px`,
                  transform: `translateX(${virtualColumn.start}px) translateY(${virtualRow.start}px)`,
                  border: '1px solid #ddd',
                  display: 'flex',
                  alignItems: 'center',
                  justifyContent: 'center',
                }}
              >
                {virtualRow.index}, {virtualColumn.index}
              </div>
            ))}
          </React.Fragment>
        ))}
      </div>
    </div>
  );
}
```

## Infinite Scrolling

Combine virtualization with data fetching:

### Infinite List

```typescript
import React, { useState, useEffect } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface Item {
  id: number;
  name: string;
}

function InfiniteList() {
  const parentRef = React.useRef<HTMLDivElement>(null);
  const [items, setItems] = useState<Item[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  
  const virtualizer = useVirtualizer({
    count: hasMore ? items.length + 1 : items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });
  
  const loadMore = async () => {
    if (isLoading) return;
    
    setIsLoading(true);
    
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    const newItems = Array.from({ length: 20 }, (_, i) => ({
      id: items.length + i,
      name: `Item ${items.length + i}`,
    }));
    
    setItems(prev => [...prev, ...newItems]);
    setIsLoading(false);
    
    // Stop loading after 200 items
    if (items.length + newItems.length >= 200) {
      setHasMore(false);
    }
  };
  
  useEffect(() => {
    loadMore(); // Load initial items
  }, []);
  
  useEffect(() => {
    const [lastItem] = [...virtualizer.getVirtualItems()].reverse();
    
    if (!lastItem) return;
    
    if (
      lastItem.index >= items.length - 1 &&
      hasMore &&
      !isLoading
    ) {
      loadMore();
    }
  }, [
    hasMore,
    isLoading,
    items.length,
    virtualizer.getVirtualItems(),
  ]);
  
  return (
    <div
      ref={parentRef}
      style={{
        height: '600px',
        overflow: 'auto',
      }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map(virtualItem => {
          const isLoaderRow = virtualItem.index > items.length - 1;
          const item = items[virtualItem.index];
          
          return (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              {isLoaderRow ? (
                hasMore ? (
                  <div style={{ padding: '10px', textAlign: 'center' }}>
                    Loading more...
                  </div>
                ) : (
                  <div style={{ padding: '10px', textAlign: 'center' }}>
                    No more items
                  </div>
                )
              ) : (
                <div style={{ padding: '10px' }}>
                  {item.name}
                </div>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## Performance Considerations

### Overscan Optimization

```typescript
import { useVirtualizer } from '@tanstack/react-virtual';

function OptimizedList({ items }: { items: any[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    // Overscan: number of items to render outside viewport
    // Higher = smoother scrolling, more memory
    // Lower = less memory, potential white flash
    overscan: 5, // Good default
  });
  
  // For very fast scrolling, increase overscan
  // For memory-constrained devices, decrease overscan
  
  return <div ref={parentRef}>{/* ... */}</div>;
}
```

### Memoization

```typescript
import React, { useMemo } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

const Row = React.memo(function Row({ 
  item 
}: { 
  item: { id: number; name: string } 
}) {
  console.log('Row rendered:', item.id);
  return <div>{item.name}</div>;
});

function MemoizedList({ items }: { items: any[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });
  
  const virtualItems = virtualizer.getVirtualItems();
  
  // Memoize rendered rows
  const rows = useMemo(
    () => virtualItems.map(virtualItem => ({
      key: virtualItem.key,
      index: virtualItem.index,
      style: {
        position: 'absolute' as const,
        top: 0,
        left: 0,
        width: '100%',
        height: `${virtualItem.size}px`,
        transform: `translateY(${virtualItem.start}px)`,
      },
    })),
    [virtualItems]
  );
  
  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rows.map(row => (
          <div key={row.key} style={row.style}>
            <Row item={items[row.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Common Mistakes

### Not Setting Container Height

```typescript
// BAD: No height, virtualizer can't determine viewport
function BadList() {
  return (
    <div style={{ overflow: 'auto' }}>
      {/* Virtualizer needs fixed height to work */}
    </div>
  );
}

// GOOD: Fixed height container
function GoodList() {
  return (
    <div style={{ height: '600px', overflow: 'auto' }}>
      {/* Virtualizer can determine visible area */}
    </div>
  );
}
```

### Incorrect Item Size

```typescript
// BAD: estimateSize doesn't match actual size
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 50, // Estimate: 50px
});

// But actual item is 100px tall
<div style={{ height: '100px' }}>{item.name}</div>

// Result: Scroll positions are wrong, items overlap

// GOOD: Accurate estimate or measurement
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 100, // Matches actual size
  // OR use measureElement for dynamic sizing
  measureElement: (el) => el.getBoundingClientRect().height,
});
```

## Best Practices

### When to Use Virtualization

```typescript
// Use virtualization when:
// - List has > 100 items
// - Items are complex/expensive to render
// - Performance issues observed
// - Mobile devices targeted

// Don't use virtualization when:
// - List has < 50 simple items
// - Items need to be searchable (Ctrl+F)
// - SEO is critical (items not in DOM)
// - Complexity outweighs benefits
```

## Real-World Use Cases

### Data Table

```typescript
import React from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

function UserTable({ users }: { users: User[] }) {
  const parentRef = React.useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: users.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });
  
  return (
    <div>
      <table style={{ width: '100%', borderCollapse: 'collapse' }}>
        <thead>
          <tr>
            <th style={{ width: '30%' }}>Name</th>
            <th style={{ width: '40%' }}>Email</th>
            <th style={{ width: '30%' }}>Role</th>
          </tr>
        </thead>
      </table>
      
      <div
        ref={parentRef}
        style={{
          height: '600px',
          overflow: 'auto',
        }}
      >
        <div
          style={{
            height: `${virtualizer.getTotalSize()}px`,
            width: '100%',
            position: 'relative',
          }}
        >
          {virtualizer.getVirtualItems().map(virtualItem => {
            const user = users[virtualItem.index];
            
            return (
              <div
                key={virtualItem.key}
                style={{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: '100%',
                  height: `${virtualItem.size}px`,
                  transform: `translateY(${virtualItem.start}px)`,
                  display: 'flex',
                  borderBottom: '1px solid #ddd',
                }}
              >
                <div style={{ width: '30%', padding: '10px' }}>
                  {user.name}
                </div>
                <div style={{ width: '40%', padding: '10px' }}>
                  {user.email}
                </div>
                <div style={{ width: '30%', padding: '10px' }}>
                  {user.role}
                </div>
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}
```

## Key Takeaways

1. **Virtualization renders only visible items**: Dramatically improves performance for large lists
2. **TanStack Virtual is modern and flexible**: Best choice for new projects
3. **react-window is lightweight**: Good for simple use cases
4. **Requires fixed container height**: Virtualizer needs to know viewport size
5. **Accurate size estimates critical**: Wrong estimates cause scrolling issues
6. **Overscan prevents white flash**: Render extra items outside viewport
7. **Works with dynamic sizing**: measureElement for variable heights
8. **Combine with infinite scrolling**: Load data as user scrolls
9. **Memoize row components**: Prevent unnecessary re-renders
10. **Use for > 100 items**: Below that, regular rendering often sufficient

## Resources

- [TanStack Virtual Documentation](https://tanstack.com/virtual/latest)
- [React Window Documentation](https://react-window.vercel.app/)
- [Virtualizing Large Lists](https://web.dev/virtualize-long-lists-react-window/)
- [Performance Optimization](https://react.dev/learn/render-and-commit)
- [TanStack Virtual Examples](https://tanstack.com/virtual/latest/docs/examples/react/table)
