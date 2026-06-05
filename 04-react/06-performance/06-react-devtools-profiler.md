# React DevTools Profiler

## The Idea

**In plain English:** React DevTools Profiler is a built-in inspector that records how long each piece of your app's screen (called a component) takes to update, so you can find the slow ones and speed them up.

**Real-world analogy:** Imagine a pit crew timing each mechanic during a Formula 1 pitstop to figure out who is slowing down the whole car change. A stopwatch is held on every person, and at the end you get a chart showing who took the longest.

- The pit crew members = React components (each one does a specific job on screen)
- The stopwatch timing each mechanic = the Profiler measuring each component's render time
- The final chart of slowest-to-fastest mechanics = the Ranked view showing which components need optimizing

---

## Overview

The React DevTools Profiler is an essential tool for identifying and fixing performance bottlenecks in React applications. It provides visual insights into component render behavior, helping you optimize render times and understand why components re-render. This guide covers how to use the Profiler effectively for performance analysis.

## Table of Contents

1. [Installing React DevTools](#installing-react-devtools)
2. [Profiler Tab Overview](#profiler-tab-overview)
3. [Recording and Analyzing](#recording-and-analyzing)
4. [Flamegraph View](#flamegraph-view)
5. [Ranked Chart](#ranked-chart)
6. [Component Tree](#component-tree)
7. [Identifying Performance Issues](#identifying-performance-issues)
8. [Commit Phase Analysis](#commit-phase-analysis)
9. [Profiler API](#profiler-api)
10. [Common Patterns](#common-patterns)
11. [Best Practices](#best-practices)
12. [Key Takeaways](#key-takeaways)
13. [Resources](#resources)

## Installing React DevTools

### Browser Extension

```bash
# Chrome
https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi

# Firefox
https://addons.mozilla.org/en-US/firefox/addon/react-devtools/

# Edge
https://microsoftedge.microsoft.com/addons/detail/react-developer-tools/gpphkfbcpidddadnkolkpfckpihlkkil
```

### Standalone App

```bash
npm install -g react-devtools

# Run
react-devtools
```

### Enable Profiling in Production

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [
    react({
      // Enable profiling in production builds
      babel: {
        plugins: [
          [
            'babel-plugin-react-compiler',
            {
              target: '18'
            }
          ]
        ]
      }
    })
  ],
  build: {
    sourcemap: true, // Include source maps for better debugging
  }
});
```

## Profiler Tab Overview

### Interface Components

The Profiler tab has several key areas:

1. **Record Button**: Start/stop profiling session
2. **Clear Button**: Clear recorded profiling data
3. **Load/Save**: Import/export profiling data
4. **View Toggle**: Switch between Flamegraph and Ranked views
5. **Commit Selector**: Navigate through recorded commits
6. **Component Details**: Show info about selected component

### Starting a Profiling Session

```typescript
import React, { useState } from 'react';

function App() {
  const [count, setCount] = useState(0);
  const [items, setItems] = useState<string[]>([]);
  
  return (
    <div>
      {/* 
        To profile this app:
        1. Open React DevTools
        2. Click Profiler tab
        3. Click Record button
        4. Interact with app (click buttons)
        5. Click Stop button
        6. Analyze results
      */}
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      <button onClick={() => setItems([...items, `Item ${items.length}`])}>
        Add Item
      </button>
      
      <ul>
        {items.map((item, i) => (
          <li key={i}>{item}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Recording and Analyzing

### Basic Workflow

```typescript
import React, { useState } from 'react';

// Example component to profile
function SlowComponent({ value }: { value: number }) {
  // Intentionally slow computation
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += Math.random();
  }
  
  return <div>Value: {value}, Result: {result}</div>;
}

function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
      <SlowComponent value={count} />
    </div>
  );
}

// Profiling steps:
// 1. Click Record
// 2. Click Increment button multiple times
// 3. Click Stop
// 4. You'll see SlowComponent taking significant time in flamegraph
```

### Interpreting Colors

The Profiler uses colors to indicate render duration:

- **Gray**: Component did not render
- **Blue/Green**: Fast render (< 10ms)
- **Yellow**: Moderate render (10-25ms)
- **Orange**: Slow render (25-50ms)
- **Red**: Very slow render (> 50ms)

```typescript
import React, { useState, useMemo } from 'react';

// This will show as RED in profiler
function VerySlowComponent() {
  let result = 0;
  for (let i = 0; i < 500000000; i++) {
    result += Math.random();
  }
  return <div>Slow: {result}</div>;
}

// This will show as YELLOW in profiler
function ModerateComponent() {
  let result = 0;
  for (let i = 0; i < 50000000; i++) {
    result += Math.random();
  }
  return <div>Moderate: {result}</div>;
}

// This will show as GREEN in profiler
function FastComponent() {
  return <div>Fast!</div>;
}

// This will show as GRAY (doesn't re-render when count changes)
const MemoizedComponent = React.memo(function MemoizedComponent() {
  return <div>Memoized - won't re-render</div>;
});

function ProfilerExample() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <VerySlowComponent />
      <ModerateComponent />
      <FastComponent />
      <MemoizedComponent />
    </div>
  );
}
```

## Flamegraph View

The Flamegraph shows component hierarchy and render times:

### Reading the Flamegraph

```typescript
import React, { useState } from 'react';

// Parent component
function Dashboard() {
  const [data, setData] = useState(0);
  
  return (
    <div>
      <Header />
      <Sidebar />
      <MainContent data={data} />
      <button onClick={() => setData(data + 1)}>Update</button>
    </div>
  );
}

// These components create the hierarchy shown in flamegraph
function Header() {
  return <div>Header</div>;
}

function Sidebar() {
  return <div>Sidebar</div>;
}

function MainContent({ data }: { data: number }) {
  return (
    <div>
      <Chart data={data} />
      <Table data={data} />
    </div>
  );
}

function Chart({ data }: { data: number }) {
  // Expensive computation
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += data * Math.random();
  }
  return <div>Chart: {result}</div>;
}

function Table({ data }: { data: number }) {
  return <div>Table: {data}</div>;
}

/*
Flamegraph will show:
┌─────────────────────────────────────────┐
│         Dashboard (total time)          │
├─────┬──────┬──────────────────────┬─────┤
│Header│Sidebar│   MainContent       │     │
│(gray)│(gray)│                      │     │
│      │      ├───────────┬──────────┤     │
│      │      │  Chart    │  Table   │     │
│      │      │   (RED)   │ (green)  │     │
└─────┴──────┴───────────┴──────────┴─────┘

Width = render duration
Color = performance (red = slow, green = fast, gray = didn't render)
*/
```

### Identifying Bottlenecks

```typescript
import React, { useState } from 'react';

function App() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      
      {/* Fast component - green in flamegraph */}
      <FastChild />
      
      {/* Slow component - red in flamegraph */}
      <SlowChild count={count} />
      
      {/* Multiple children */}
      <List count={count} />
    </div>
  );
}

const FastChild = React.memo(function FastChild() {
  return <div>I'm fast!</div>;
});

function SlowChild({ count }: { count: number }) {
  // This will appear as wide red bar in flamegraph
  let result = 0;
  for (let i = 0; i < 500000000; i++) {
    result += count * Math.random();
  }
  return <div>Slow: {result}</div>;
}

function List({ count }: { count: number }) {
  // If items re-render unnecessarily, you'll see multiple bars
  const items = Array.from({ length: 100 }, (_, i) => i);
  
  return (
    <ul>
      {items.map(item => (
        <ListItem key={item} item={item} count={count} />
      ))}
    </ul>
  );
}

// Without memo: 100 orange/yellow bars in flamegraph
// With memo: 100 gray bars (didn't render)
const ListItem = React.memo(function ListItem({ 
  item, 
  count 
}: { 
  item: number; 
  count: number 
}) {
  return <li>Item {item}: {count}</li>;
});
```

## Ranked Chart

Shows components sorted by render time:

### Using Ranked View

```typescript
import React, { useState } from 'react';

function App() {
  const [trigger, setTrigger] = useState(0);
  
  return (
    <div>
      <button onClick={() => setTrigger(trigger + 1)}>Trigger</button>
      
      {/* These components will appear in ranked chart,
          sorted by render duration */}
      <Component1 />
      <Component2 />
      <Component3 />
      <Component4 />
      <Component5 />
    </div>
  );
}

function Component1() {
  let r = 0;
  for (let i = 0; i < 500000000; i++) r += Math.random();
  return <div>C1: {r}</div>;
}

function Component2() {
  let r = 0;
  for (let i = 0; i < 100000000; i++) r += Math.random();
  return <div>C2: {r}</div>;
}

function Component3() {
  let r = 0;
  for (let i = 0; i < 300000000; i++) r += Math.random();
  return <div>C3: {r}</div>;
}

function Component4() {
  let r = 0;
  for (let i = 0; i < 50000000; i++) r += Math.random();
  return <div>C4: {r}</div>;
}

function Component5() {
  return <div>C5: fast!</div>;
}

/*
Ranked chart will show (sorted by duration):
1. Component1 - 450ms █████████████████████
2. Component3 - 270ms ██████████████
3. Component2 - 90ms  █████
4. Component4 - 45ms  ███
5. Component5 - 1ms   ▏

This helps quickly identify slowest components
*/
```

## Component Tree

View component hierarchy and render causes:

### Understanding Render Reasons

```typescript
import React, { useState, useContext, createContext } from 'react';

const ThemeContext = createContext('light');

function App() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <div>
        <button onClick={() => setCount(count + 1)}>
          Count: {count}
        </button>
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Toggle Theme
        </button>
        
        {/* Click on component in DevTools to see why it rendered */}
        <ChildA count={count} />
        <ChildB />
        <ChildC />
      </div>
    </ThemeContext.Provider>
  );
}

// Renders because: Props changed (count)
function ChildA({ count }: { count: number }) {
  console.log('ChildA rendered');
  return <div>Count: {count}</div>;
}

// Renders because: Parent rendered (App)
function ChildB() {
  console.log('ChildB rendered');
  return <div>ChildB</div>;
}

// Renders because: Context changed (theme)
function ChildC() {
  const theme = useContext(ThemeContext);
  console.log('ChildC rendered');
  return <div>Theme: {theme}</div>;
}

/*
DevTools will show render reason:
- ChildA: "Props changed: count"
- ChildB: "Parent component rendered"
- ChildC: "Context changed: ThemeContext"
*/
```

## Identifying Performance Issues

### Common Issues to Look For

```typescript
import React, { useState, useCallback } from 'react';

// Issue 1: Unnecessary Re-renders
function IssueUnnecessaryRenders() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      
      {/* Problem: Re-renders even though it doesn't use count */}
      <ExpensiveComponent />
    </div>
  );
}

// Solution: Memoize
const ExpensiveComponentMemo = React.memo(function ExpensiveComponent() {
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += Math.random();
  }
  return <div>Expensive: {result}</div>;
});

// Issue 2: Creating New Objects in Render
function IssueNewObjects() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      
      {/* Problem: New object every render, memo useless */}
      <MemoizedChild config={{ theme: 'light' }} />
    </div>
  );
}

// Solution: useMemo
function FixedNewObjects() {
  const [count, setCount] = useState(0);
  
  const config = React.useMemo(() => ({ theme: 'light' }), []);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <MemoizedChild config={config} />
    </div>
  );
}

const MemoizedChild = React.memo(function Child({ 
  config 
}: { 
  config: { theme: string } 
}) {
  return <div>Theme: {config.theme}</div>;
});

// Issue 3: Inline Function Props
function IssueInlineFunctions() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      
      {/* Problem: New function every render */}
      <Button onClick={() => console.log('clicked')} />
    </div>
  );
}

// Solution: useCallback
function FixedInlineFunctions() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <Button onClick={handleClick} />
    </div>
  );
}

const Button = React.memo(function Button({ 
  onClick 
}: { 
  onClick: () => void 
}) {
  return <button onClick={onClick}>Click me</button>;
});
```

## Commit Phase Analysis

### Understanding Commits

```typescript
import React, { useState } from 'react';

function App() {
  const [stateA, setStateA] = useState(0);
  const [stateB, setStateB] = useState(0);
  
  const updateBoth = () => {
    // React batches these into ONE commit
    setStateA(prev => prev + 1);
    setStateB(prev => prev + 1);
  };
  
  return (
    <div>
      <button onClick={() => setStateA(stateA + 1)}>
        Update A: {stateA}
      </button>
      
      <button onClick={() => setStateB(stateB + 1)}>
        Update B: {stateB}
      </button>
      
      <button onClick={updateBoth}>
        Update Both (batched)
      </button>
      
      <ComponentA value={stateA} />
      <ComponentB value={stateB} />
    </div>
  );
}

function ComponentA({ value }: { value: number }) {
  return <div>A: {value}</div>;
}

function ComponentB({ value }: { value: number }) {
  return <div>B: {value}</div>;
}

/*
Profiler shows commits:
- Click "Update A": 1 commit, only ComponentA renders
- Click "Update B": 1 commit, only ComponentB renders
- Click "Update Both": 1 commit, both render (batched)

Each commit represents one update to the DOM
*/
```

### Commit Duration

```typescript
import React, { useState } from 'react';

function App() {
  const [data, setData] = useState(0);
  
  return (
    <div>
      <button onClick={() => setData(data + 1)}>Update</button>
      
      {/* Fast commit: < 16ms (60fps) - smooth */}
      <FastComponent value={data} />
      
      {/* Slow commit: > 16ms - may drop frames */}
      <SlowComponent value={data} />
    </div>
  );
}

function FastComponent({ value }: { value: number }) {
  return <div>Fast: {value}</div>;
}

function SlowComponent({ value }: { value: number }) {
  // Takes > 100ms
  let result = 0;
  for (let i = 0; i < 500000000; i++) {
    result += value * Math.random();
  }
  return <div>Slow: {result}</div>;
}

/*
Profiler commit info shows:
- Commit duration: Total time for this update
- Render duration: Time spent in React
- Commit duration breakdown:
  - Render phase: Computing changes
  - Commit phase: Applying to DOM
  
Goal: Keep commits under 16ms for 60fps
*/
```

## Profiler API

Programmatically collect performance data:

### Using Profiler Component

```typescript
import React, { Profiler, useState } from 'react';

function App() {
  const [count, setCount] = useState(0);
  
  const onRender = (
    id: string, // "App" - component id
    phase: 'mount' | 'update', // Mount or update
    actualDuration: number, // Time spent rendering
    baseDuration: number, // Estimated time without memoization
    startTime: number, // When render started
    commitTime: number, // When commit finished
    interactions: Set<any> // Interactions that caused this render
  ) => {
    console.log(`${id} ${phase}:`, {
      actualDuration,
      baseDuration,
      startTime,
      commitTime,
    });
    
    // Send to analytics
    if (actualDuration > 16) {
      console.warn(`Slow render detected: ${actualDuration}ms`);
    }
  };
  
  return (
    <Profiler id="App" onRender={onRender}>
      <div>
        <button onClick={() => setCount(count + 1)}>
          Count: {count}
        </button>
        <ExpensiveComponent value={count} />
      </div>
    </Profiler>
  );
}

function ExpensiveComponent({ value }: { value: number }) {
  let result = 0;
  for (let i = 0; i < 100000000; i++) {
    result += value * Math.random();
  }
  return <div>Result: {result}</div>;
}
```

### Multiple Profilers

```typescript
import React, { Profiler, useState } from 'react';

function App() {
  const [data, setData] = useState(0);
  
  const logProfile = (id: string) => (
    _id: string,
    phase: 'mount' | 'update',
    actualDuration: number
  ) => {
    console.log(`[${id}] ${phase}: ${actualDuration.toFixed(2)}ms`);
  };
  
  return (
    <div>
      <button onClick={() => setData(data + 1)}>Update</button>
      
      <Profiler id="Header" onRender={logProfile('Header')}>
        <Header />
      </Profiler>
      
      <Profiler id="Sidebar" onRender={logProfile('Sidebar')}>
        <Sidebar />
      </Profiler>
      
      <Profiler id="Content" onRender={logProfile('Content')}>
        <Content data={data} />
      </Profiler>
    </div>
  );
}

function Header() {
  return <div>Header</div>;
}

function Sidebar() {
  return <div>Sidebar</div>;
}

function Content({ data }: { data: number }) {
  return <div>Content: {data}</div>;
}

// Output:
// [Content] update: 1.23ms
// (Header and Sidebar don't log - didn't re-render)
```

### Production Monitoring

```typescript
import React, { Profiler } from 'react';

// Send performance data to analytics
function sendToAnalytics(data: {
  componentId: string;
  phase: 'mount' | 'update';
  duration: number;
}) {
  // Send to your analytics service
  fetch('/api/performance', {
    method: 'POST',
    body: JSON.stringify(data),
  });
}

function App() {
  const onRender = (
    id: string,
    phase: 'mount' | 'update',
    actualDuration: number
  ) => {
    // Only log slow renders in production
    if (actualDuration > 50) {
      sendToAnalytics({
        componentId: id,
        phase,
        duration: actualDuration,
      });
    }
  };
  
  return (
    <Profiler id="App" onRender={onRender}>
      <YourApp />
    </Profiler>
  );
}

function YourApp() {
  return <div>Your app</div>;
}
```

## Common Patterns

### Before and After Optimization

```typescript
import React, { useState, useMemo, useCallback } from 'react';

// BEFORE: Everything re-renders
function AppBefore() {
  const [count, setCount] = useState(0);
  const [filter, setFilter] = useState('');
  
  const items = ['apple', 'banana', 'cherry', 'date'];
  
  const filteredItems = items.filter(item =>
    item.toLowerCase().includes(filter.toLowerCase())
  );
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      <input
        value={filter}
        onChange={e => setFilter(e.target.value)}
      />
      
      <List items={filteredItems} />
    </div>
  );
}

// AFTER: Optimized with memoization
function AppAfter() {
  const [count, setCount] = useState(0);
  const [filter, setFilter] = useState('');
  
  const items = useMemo(
    () => ['apple', 'banana', 'cherry', 'date'],
    []
  );
  
  const filteredItems = useMemo(
    () => items.filter(item =>
      item.toLowerCase().includes(filter.toLowerCase())
    ),
    [items, filter]
  );
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      
      <input
        value={filter}
        onChange={e => setFilter(e.target.value)}
      />
      
      <List items={filteredItems} />
    </div>
  );
}

const List = React.memo(function List({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item}>{item}</li>
      ))}
    </ul>
  );
});

/*
Profile both:
Before: List re-renders when count changes (unnecessary)
After: List only re-renders when filter changes (correct)
*/
```

## Best Practices

1. **Profile in production mode**: Development mode has extra checks
2. **Focus on the slowest components**: Use Ranked view to prioritize
3. **Look for unnecessary re-renders**: Gray bars mean memoization works
4. **Check commit frequency**: Too many commits = too many updates
5. **Measure before and after**: Verify optimizations actually help
6. **Don't over-optimize**: Only optimize slow components
7. **Use Profiler API in production**: Monitor real user performance
8. **Compare commits**: See how changes affect different updates
9. **Check context propagation**: Context changes cause wide re-renders
10. **Profile user interactions**: Record during typical use cases

## Key Takeaways

1. **Visual performance insights**: Flamegraph and ranked views show bottlenecks
2. **Color-coded performance**: Red = slow, green = fast, gray = didn't render
3. **Identify render causes**: See why each component rendered
4. **Commit analysis**: Understand update batching and duration
5. **Flamegraph shows hierarchy**: Width = duration, nesting = component tree
6. **Ranked view for prioritization**: Quickly find slowest components
7. **Profiler API for monitoring**: Programmatic performance tracking
8. **Focus on actual issues**: Only optimize measurably slow components
9. **Production profiling**: Enable in builds to measure real performance
10. **Iterative optimization**: Profile, optimize, measure, repeat

## Resources

- [React DevTools Profiler](https://react.dev/learn/react-developer-tools)
- [Profiler API](https://react.dev/reference/react/Profiler)
- [Optimizing Performance](https://react.dev/learn/render-and-commit)
- [React DevTools Tutorial](https://react.dev/learn/react-developer-tools)
- [Performance Optimization Guide](https://web.dev/react/)
