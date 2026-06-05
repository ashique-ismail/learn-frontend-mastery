# Code Splitting with React.lazy and Suspense

## The Idea

**In plain English:** Code splitting is a way to break your website's code into smaller pieces so the browser only downloads what it needs right now, instead of downloading everything at once. Think of it like only grabbing the chapters of a book you plan to read today, rather than the entire library.

**Real-world analogy:** Imagine a restaurant that prints a separate mini-menu for each section (appetizers, mains, desserts) instead of one giant laminated booklet. A customer who only wants dessert gets handed just the dessert card — it arrives fast, and the kitchen doesn't have to prep every dish upfront.

- The separate mini-menu cards = the individual code chunks loaded on demand
- The customer choosing a section = the user navigating to a specific page or opening a feature
- React.lazy + Suspense = the waiter who fetches the right card and says "one moment please" while it's being retrieved

---

## Overview

Code splitting is a technique to split your JavaScript bundle into smaller chunks that can be loaded on demand. React provides `React.lazy` for component-level code splitting and `Suspense` for handling loading states. This guide covers how to implement code splitting effectively to reduce initial bundle size and improve application performance.

## Table of Contents

1. [Why Code Splitting](#why-code-splitting)
2. [React.lazy](#reactlazy)
3. [Suspense Boundaries](#suspense-boundaries)
4. [Route-Based Code Splitting](#route-based-code-splitting)
5. [Component-Based Code Splitting](#component-based-code-splitting)
6. [Error Boundaries with Suspense](#error-boundaries-with-suspense)
7. [Bundle Optimization](#bundle-optimization)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Real-World Use Cases](#real-world-use-cases)
11. [Key Takeaways](#key-takeaways)
12. [Resources](#resources)

## Why Code Splitting

Code splitting reduces initial load time by loading only what's needed:

### Bundle Size Impact

```typescript
// Before code splitting - everything loaded upfront
import React from 'react';
import HeavyChart from './HeavyChart'; // 500kb
import HeavyTable from './HeavyTable'; // 300kb
import HeavyEditor from './HeavyEditor'; // 400kb

function App() {
  const [view, setView] = React.useState<'chart' | 'table' | 'editor'>('chart');
  
  return (
    <div>
      <nav>
        <button onClick={() => setView('chart')}>Chart</button>
        <button onClick={() => setView('table')}>Table</button>
        <button onClick={() => setView('editor')}>Editor</button>
      </nav>
      
      {view === 'chart' && <HeavyChart />}
      {view === 'table' && <HeavyTable />}
      {view === 'editor' && <HeavyEditor />}
    </div>
  );
}

// Initial bundle: 1.2MB
// User only needs 500kb but loads everything
```

### Dynamic Import Syntax

```typescript
// Dynamic import returns a Promise
const module = await import('./module');

// Example with named export
const { Component } = await import('./Component');

// Example with default export
const Component = (await import('./Component')).default;

// Dynamic import in function
async function loadComponent() {
  const { default: Component } = await import('./Component');
  return Component;
}
```

## React.lazy

`React.lazy` enables component-level code splitting:

### Basic Usage

```typescript
import React, { Suspense, lazy } from 'react';

// Lazy load component
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <div>
      <h1>My App</h1>
      
      {/* Must wrap in Suspense */}
      <Suspense fallback={<div>Loading...</div>}>
        <HeavyComponent />
      </Suspense>
    </div>
  );
}

// HeavyComponent.tsx
export default function HeavyComponent() {
  return <div>Heavy Component Loaded</div>;
}
```

### TypeScript with React.lazy

```typescript
import React, { Suspense, lazy } from 'react';

// Type inference works automatically
const Dashboard = lazy(() => import('./Dashboard'));

// Explicit typing
const Settings = lazy<React.ComponentType<{ userId: string }>>(
  () => import('./Settings')
);

// With props interface
interface UserProfileProps {
  userId: string;
  onUpdate: () => void;
}

const UserProfile = lazy<React.ComponentType<UserProfileProps>>(
  () => import('./UserProfile')
);

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Dashboard />
      <Settings userId="123" />
      <UserProfile userId="456" onUpdate={() => console.log('updated')} />
    </Suspense>
  );
}
```

### Named Exports

```typescript
import React, { Suspense, lazy } from 'react';

// Component with named export needs re-export as default
const Chart = lazy(() =>
  import('./Charts').then(module => ({ default: module.LineChart }))
);

const Table = lazy(() =>
  import('./Tables').then(module => ({ default: module.DataTable }))
);

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Chart data={[1, 2, 3]} />
      <Table rows={[]} />
    </Suspense>
  );
}

// Charts.tsx
export function LineChart({ data }: { data: number[] }) {
  return <div>Line Chart</div>;
}

export function BarChart({ data }: { data: number[] }) {
  return <div>Bar Chart</div>;
}
```

### Conditional Lazy Loading

```typescript
import React, { Suspense, lazy, useState } from 'react';

const AdminPanel = lazy(() => import('./AdminPanel'));
const UserDashboard = lazy(() => import('./UserDashboard'));

function App() {
  const [isAdmin, setIsAdmin] = useState(false);
  const [showPanel, setShowPanel] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsAdmin(!isAdmin)}>
        Toggle Admin
      </button>
      <button onClick={() => setShowPanel(!showPanel)}>
        Toggle Panel
      </button>
      
      {showPanel && (
        <Suspense fallback={<div>Loading panel...</div>}>
          {isAdmin ? <AdminPanel /> : <UserDashboard />}
        </Suspense>
      )}
    </div>
  );
}
```

## Suspense Boundaries

Suspense components define loading boundaries:

### Basic Suspense

```typescript
import React, { Suspense, lazy } from 'react';

const Content = lazy(() => import('./Content'));

function App() {
  return (
    <div>
      <header>Always Visible Header</header>
      
      <Suspense fallback={<div>Loading content...</div>}>
        <Content />
      </Suspense>
      
      <footer>Always Visible Footer</footer>
    </div>
  );
}
```

### Nested Suspense

```typescript
import React, { Suspense, lazy } from 'react';

const Sidebar = lazy(() => import('./Sidebar'));
const MainContent = lazy(() => import('./MainContent'));
const Widget = lazy(() => import('./Widget'));

function App() {
  return (
    <div>
      {/* Separate loading states for different sections */}
      <Suspense fallback={<div>Loading sidebar...</div>}>
        <Sidebar />
      </Suspense>
      
      <Suspense fallback={<div>Loading main content...</div>}>
        <MainContent />
        
        {/* Nested Suspense for widget inside main content */}
        <Suspense fallback={<div>Loading widget...</div>}>
          <Widget />
        </Suspense>
      </Suspense>
    </div>
  );
}
```

### Multiple Components in One Boundary

```typescript
import React, { Suspense, lazy } from 'react';

const ProfileHeader = lazy(() => import('./ProfileHeader'));
const ProfileStats = lazy(() => import('./ProfileStats'));
const ProfilePosts = lazy(() => import('./ProfilePosts'));

function Profile() {
  return (
    <Suspense fallback={<div>Loading profile...</div>}>
      {/* All three components share same loading state */}
      <ProfileHeader />
      <ProfileStats />
      <ProfilePosts />
    </Suspense>
  );
}

// Better: Separate boundaries for independent loading
function ProfileBetter() {
  return (
    <div>
      <Suspense fallback={<div>Loading header...</div>}>
        <ProfileHeader />
      </Suspense>
      
      <Suspense fallback={<div>Loading stats...</div>}>
        <ProfileStats />
      </Suspense>
      
      <Suspense fallback={<div>Loading posts...</div>}>
        <ProfilePosts />
      </Suspense>
    </div>
  );
}
```

### Custom Loading Components

```typescript
import React, { Suspense, lazy } from 'react';

function LoadingSpinner() {
  return (
    <div style={{ 
      display: 'flex', 
      justifyContent: 'center', 
      padding: '20px' 
    }}>
      <div className="spinner">Loading...</div>
    </div>
  );
}

function LoadingSkeleton() {
  return (
    <div className="skeleton">
      <div className="skeleton-line" />
      <div className="skeleton-line" />
      <div className="skeleton-line" />
    </div>
  );
}

const Content = lazy(() => import('./Content'));

function App() {
  const [loadingType, setLoadingType] = React.useState<'spinner' | 'skeleton'>('spinner');
  
  return (
    <Suspense 
      fallback={
        loadingType === 'spinner' 
          ? <LoadingSpinner /> 
          : <LoadingSkeleton />
      }
    >
      <Content />
    </Suspense>
  );
}
```

## Route-Based Code Splitting

Split code by routes for optimal initial load:

### React Router with Lazy Loading

```typescript
import React, { Suspense, lazy } from 'react';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

// Lazy load route components
const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
        <Link to="/dashboard">Dashboard</Link>
        <Link to="/profile">Profile</Link>
        <Link to="/settings">Settings</Link>
      </nav>
      
      <Suspense fallback={<div>Loading page...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### Nested Routes

```typescript
import React, { Suspense, lazy } from 'react';
import { Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Analytics = lazy(() => import('./pages/Analytics'));
const Reports = lazy(() => import('./pages/Reports'));
const Users = lazy(() => import('./pages/Users'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />}>
          <Route 
            path="analytics" 
            element={
              <Suspense fallback={<div>Loading analytics...</div>}>
                <Analytics />
              </Suspense>
            } 
          />
          <Route 
            path="reports" 
            element={
              <Suspense fallback={<div>Loading reports...</div>}>
                <Reports />
              </Suspense>
            } 
          />
          <Route 
            path="users" 
            element={
              <Suspense fallback={<div>Loading users...</div>}>
                <Users />
              </Suspense>
            } 
          />
        </Route>
      </Routes>
    </Suspense>
  );
}
```

### Route Preloading

```typescript
import React, { Suspense, lazy } from 'react';
import { Link } from 'react-router-dom';

// Store lazy imports for preloading
const dashboardImport = () => import('./pages/Dashboard');
const settingsImport = () => import('./pages/Settings');

const Dashboard = lazy(dashboardImport);
const Settings = lazy(settingsImport);

function Navigation() {
  // Preload on hover
  const preloadDashboard = () => {
    dashboardImport();
  };
  
  const preloadSettings = () => {
    settingsImport();
  };
  
  return (
    <nav>
      <Link 
        to="/dashboard" 
        onMouseEnter={preloadDashboard}
      >
        Dashboard
      </Link>
      <Link 
        to="/settings" 
        onMouseEnter={preloadSettings}
      >
        Settings
      </Link>
    </nav>
  );
}
```

## Component-Based Code Splitting

Split individual components for better granularity:

### Modal Components

```typescript
import React, { Suspense, lazy, useState } from 'react';

const SettingsModal = lazy(() => import('./SettingsModal'));
const HelpModal = lazy(() => import('./HelpModal'));
const FeedbackModal = lazy(() => import('./FeedbackModal'));

function App() {
  const [openModal, setOpenModal] = useState<
    'settings' | 'help' | 'feedback' | null
  >(null);
  
  return (
    <div>
      <button onClick={() => setOpenModal('settings')}>Settings</button>
      <button onClick={() => setOpenModal('help')}>Help</button>
      <button onClick={() => setOpenModal('feedback')}>Feedback</button>
      
      {openModal && (
        <Suspense fallback={<div>Loading modal...</div>}>
          {openModal === 'settings' && (
            <SettingsModal onClose={() => setOpenModal(null)} />
          )}
          {openModal === 'help' && (
            <HelpModal onClose={() => setOpenModal(null)} />
          )}
          {openModal === 'feedback' && (
            <FeedbackModal onClose={() => setOpenModal(null)} />
          )}
        </Suspense>
      )}
    </div>
  );
}
```

### Tab Components

```typescript
import React, { Suspense, lazy, useState } from 'react';

const ProfileTab = lazy(() => import('./tabs/ProfileTab'));
const PostsTab = lazy(() => import('./tabs/PostsTab'));
const PhotosTab = lazy(() => import('./tabs/PhotosTab'));
const VideosTab = lazy(() => import('./tabs/VideosTab'));

function UserProfile() {
  const [activeTab, setActiveTab] = useState<
    'profile' | 'posts' | 'photos' | 'videos'
  >('profile');
  
  return (
    <div>
      <div className="tabs">
        <button onClick={() => setActiveTab('profile')}>Profile</button>
        <button onClick={() => setActiveTab('posts')}>Posts</button>
        <button onClick={() => setActiveTab('photos')}>Photos</button>
        <button onClick={() => setActiveTab('videos')}>Videos</button>
      </div>
      
      <Suspense fallback={<div>Loading tab...</div>}>
        {activeTab === 'profile' && <ProfileTab />}
        {activeTab === 'posts' && <PostsTab />}
        {activeTab === 'photos' && <PhotosTab />}
        {activeTab === 'videos' && <VideosTab />}
      </Suspense>
    </div>
  );
}
```

### Third-Party Library Splitting

```typescript
import React, { Suspense, lazy, useState } from 'react';

// Split heavy third-party libraries
const MarkdownEditor = lazy(() => 
  import('./MarkdownEditor') // Contains react-markdown
);

const CodeEditor = lazy(() => 
  import('./CodeEditor') // Contains monaco-editor
);

const Chart = lazy(() => 
  import('./Chart') // Contains recharts
);

function App() {
  const [showEditor, setShowEditor] = useState(false);
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowEditor(!showEditor)}>
        Toggle Editor
      </button>
      <button onClick={() => setShowChart(!showChart)}>
        Toggle Chart
      </button>
      
      {showEditor && (
        <Suspense fallback={<div>Loading editor...</div>}>
          <MarkdownEditor />
        </Suspense>
      )}
      
      {showChart && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <Chart data={[1, 2, 3]} />
        </Suspense>
      )}
    </div>
  );
}
```

## Error Boundaries with Suspense

Handle loading errors gracefully:

### Error Boundary Component

```typescript
import React, { Component, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }
  
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div>
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

export default ErrorBoundary;
```

### Combining Error Boundary and Suspense

```typescript
import React, { Suspense, lazy } from 'react';
import ErrorBoundary from './ErrorBoundary';

const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <ErrorBoundary 
      fallback={<div>Failed to load dashboard</div>}
    >
      <Suspense fallback={<div>Loading dashboard...</div>}>
        <Dashboard />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### Retry Logic

```typescript
import React, { Suspense, lazy, useState } from 'react';

function lazyWithRetry<T extends React.ComponentType<any>>(
  componentImport: () => Promise<{ default: T }>,
  retries = 3
): React.LazyExoticComponent<T> {
  return lazy(async () => {
    for (let i = 0; i < retries; i++) {
      try {
        return await componentImport();
      } catch (error) {
        if (i === retries - 1) throw error;
        
        // Wait before retry
        await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
      }
    }
    throw new Error('Failed after retries');
  });
}

const Dashboard = lazyWithRetry(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Dashboard />
    </Suspense>
  );
}
```

## Bundle Optimization

Optimize bundle splitting for production:

### Webpack Magic Comments

```typescript
import React, { Suspense, lazy } from 'react';

// Prefetch - load during idle time
const Dashboard = lazy(() =>
  import(
    /* webpackPrefetch: true */
    /* webpackChunkName: "dashboard" */
    './Dashboard'
  )
);

// Preload - load in parallel with parent
const Settings = lazy(() =>
  import(
    /* webpackPreload: true */
    /* webpackChunkName: "settings" */
    './Settings'
  )
);

// Custom chunk name
const Admin = lazy(() =>
  import(
    /* webpackChunkName: "admin-panel" */
    './AdminPanel'
  )
);
```

### Analyzing Bundle Size

```typescript
// package.json scripts
{
  "scripts": {
    "analyze": "vite-bundle-visualizer"
  }
}

// Use webpack-bundle-analyzer for webpack
// Use rollup-plugin-visualizer for Vite
```

## Common Mistakes

### Missing Suspense Boundary

```typescript
import React, { lazy } from 'react';

const Component = lazy(() => import('./Component'));

// BAD: No Suspense boundary
function AppBad() {
  return <Component />; // Error!
}

// GOOD: Wrapped in Suspense
function AppGood() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Component />
    </Suspense>
  );
}
```

### Lazy Loading Too Much

```typescript
import React, { Suspense, lazy } from 'react';

// BAD: Lazy loading tiny components
const Button = lazy(() => import('./Button')); // 1kb
const Icon = lazy(() => import('./Icon')); // 500 bytes

// Each lazy import adds overhead!
// Multiple network requests for tiny components

// GOOD: Only lazy load substantial components
const Dashboard = lazy(() => import('./Dashboard')); // 100kb
const Editor = lazy(() => import('./Editor')); // 500kb
```

### Not Handling Errors

```typescript
import React, { Suspense, lazy } from 'react';

const Component = lazy(() => import('./Component'));

// BAD: No error handling
function AppBad() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Component />
    </Suspense>
  );
}

// GOOD: With error boundary
function AppGood() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <Component />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Best Practices

### Route-Based First

```typescript
// Start with route-based splitting
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Admin = lazy(() => import('./pages/Admin'));

// Then add component-level splitting for heavy features
const Analytics = lazy(() => import('./features/Analytics'));
const Reports = lazy(() => import('./features/Reports'));
```

### Preload Critical Routes

```typescript
import React, { useEffect } from 'react';

const dashboardImport = () => import('./Dashboard');
const Dashboard = lazy(dashboardImport);

function App() {
  useEffect(() => {
    // Preload dashboard after initial render
    const timer = setTimeout(() => {
      dashboardImport();
    }, 1000);
    
    return () => clearTimeout(timer);
  }, []);
  
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Dashboard />
    </Suspense>
  );
}
```

### Measure Impact

```typescript
// Before optimization
console.time('initial-load');
// ... app code
console.timeEnd('initial-load');

// After code splitting
// Check bundle sizes in build output
// Use Lighthouse to measure performance
```

## Real-World Use Cases

### Admin Dashboard

```typescript
import React, { Suspense, lazy } from 'react';
import { Routes, Route } from 'react-router-dom';

// Public routes - loaded immediately
import Home from './pages/Home';
import Login from './pages/Login';

// Admin routes - lazy loaded
const AdminDashboard = lazy(() => import('./admin/Dashboard'));
const AdminUsers = lazy(() => import('./admin/Users'));
const AdminSettings = lazy(() => import('./admin/Settings'));
const AdminReports = lazy(() => import('./admin/Reports'));

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/login" element={<Login />} />
      
      <Route path="/admin/*" element={
        <Suspense fallback={<div>Loading admin...</div>}>
          <Routes>
            <Route path="dashboard" element={<AdminDashboard />} />
            <Route path="users" element={<AdminUsers />} />
            <Route path="settings" element={<AdminSettings />} />
            <Route path="reports" element={<AdminReports />} />
          </Routes>
        </Suspense>
      } />
    </Routes>
  );
}
```

## Key Takeaways

1. **React.lazy for components**: Dynamically import components
2. **Suspense for loading**: Handle loading states declaratively
3. **Route-based splitting**: Split by routes first for maximum impact
4. **Error boundaries**: Always handle loading errors
5. **Preload strategically**: Prefetch likely-needed routes
6. **Don't over-split**: Only split substantial components
7. **Nested Suspense**: Create granular loading boundaries
8. **TypeScript support**: Full type safety with lazy components
9. **Webpack comments**: Use magic comments for optimization
10. **Measure impact**: Use bundle analyzer and profiling tools

## Resources

- [React.lazy Documentation](https://react.dev/reference/react/lazy)
- [Suspense Documentation](https://react.dev/reference/react/Suspense)
- [Code Splitting Guide](https://react.dev/learn/code-splitting)
- [Webpack Code Splitting](https://webpack.js.org/guides/code-splitting/)
- [Vite Code Splitting](https://vitejs.dev/guide/features.html#code-splitting)
- [Web.dev Code Splitting](https://web.dev/code-splitting-suspense/)
