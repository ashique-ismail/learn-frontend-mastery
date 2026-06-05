# Nested Routes and Layouts in React Router

## The Idea

**In plain English:** Nested routes let you build web pages that have a permanent outer frame (like a navigation bar and sidebar) while only the middle section changes as you move between pages. A "layout" is just the shared wrapper that stays the same, and a "route" is a rule that says which page content to show based on the web address.

**Real-world analogy:** Think of a school binder with a fixed cover and dividers, but different pages inside each section. When you flip to the "Math" section, the binder cover and dividers stay put, only the paper inside changes.

- The binder cover and dividers = the layout component (header, sidebar, footer that never change)
- Each section of paper = a child route component (the page content that swaps out)
- The slot where pages slide in = the `<Outlet />` component (the placeholder where child content renders)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Route Nesting Basics](#route-nesting-basics)
3. [Layout Routes](#layout-routes)
4. [Index Routes](#index-routes)
5. [Outlet Component](#outlet-component)
6. [Pathless Routes](#pathless-routes)
7. [Advanced Patterns](#advanced-patterns)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)
11. [Resources](#resources)

## Introduction

Nested routing is one of React Router's most powerful features. It allows you to create hierarchical UI structures where parent components define layouts for their children, enabling code reuse, shared layouts, and intuitive URL structures that match your UI hierarchy.

## Route Nesting Basics

### Simple Nested Routes

```typescript
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        path: 'about',
        element: <About />
      },
      {
        path: 'contact',
        element: <Contact />
      }
    ]
  }
]);

function Root() {
  return (
    <div>
      <header>
        <nav>
          <Link to="/">Home</Link>
          <Link to="/about">About</Link>
          <Link to="/contact">Contact</Link>
        </nav>
      </header>
      <main>
        <Outlet /> {/* Child routes render here */}
      </main>
      <footer>© 2024 My App</footer>
    </div>
  );
}
```

### Deep Nesting

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        path: 'dashboard',
        element: <DashboardLayout />,
        children: [
          {
            path: 'overview',
            element: <DashboardOverview />
          },
          {
            path: 'analytics',
            element: <AnalyticsLayout />,
            children: [
              {
                path: 'traffic',
                element: <TrafficAnalytics />
              },
              {
                path: 'conversions',
                element: <ConversionAnalytics />
              }
            ]
          }
        ]
      }
    ]
  }
]);

// URL: /dashboard/analytics/traffic
// Renders: Root → DashboardLayout → AnalyticsLayout → TrafficAnalytics
```

### Route Hierarchy with Loaders

```typescript
interface RootData {
  user: User;
}

interface DashboardData {
  projects: Project[];
}

interface ProjectData {
  project: Project;
  tasks: Task[];
}

const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    loader: rootLoader, // Loads user
    children: [
      {
        path: 'dashboard',
        element: <Dashboard />,
        loader: dashboardLoader, // Loads projects
        children: [
          {
            path: 'projects/:projectId',
            element: <ProjectDetail />,
            loader: projectLoader // Loads project + tasks
          }
        ]
      }
    ]
  }
]);

// All loaders run in parallel when navigating to /dashboard/projects/123
async function rootLoader() {
  return { user: await fetchUser() };
}

async function dashboardLoader() {
  return { projects: await fetchProjects() };
}

async function projectLoader({ params }: LoaderFunctionArgs) {
  const [project, tasks] = await Promise.all([
    fetchProject(params.projectId),
    fetchProjectTasks(params.projectId)
  ]);
  return { project, tasks };
}
```

## Layout Routes

### Basic Layout Pattern

```typescript
// Shared layout for authenticated pages
function AuthLayout() {
  const { user } = useLoaderData() as { user: User };
  
  return (
    <div className="auth-layout">
      <Sidebar user={user} />
      <main>
        <Outlet /> {/* Different pages share this layout */}
      </main>
    </div>
  );
}

const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        path: 'app',
        element: <AuthLayout />,
        loader: authLayoutLoader,
        children: [
          {
            path: 'dashboard',
            element: <Dashboard />
          },
          {
            path: 'profile',
            element: <Profile />
          },
          {
            path: 'settings',
            element: <Settings />
          }
        ]
      }
    ]
  }
]);
```

### Multiple Layout Levels

```typescript
const router = createBrowserRouter([
  {
    // Level 1: Root layout (header, footer)
    path: '/',
    element: <RootLayout />,
    children: [
      {
        // Level 2: Dashboard layout (sidebar, topbar)
        path: 'dashboard',
        element: <DashboardLayout />,
        children: [
          {
            // Level 3: Split view layout
            path: 'projects',
            element: <SplitViewLayout />,
            children: [
              {
                path: ':projectId',
                element: <ProjectDetail />
              }
            ]
          }
        ]
      }
    ]
  }
]);

function RootLayout() {
  return (
    <>
      <Header />
      <Outlet /> {/* DashboardLayout renders here */}
      <Footer />
    </>
  );
}

function DashboardLayout() {
  return (
    <div className="dashboard">
      <Sidebar />
      <div className="content">
        <TopBar />
        <Outlet /> {/* SplitViewLayout renders here */}
      </div>
    </div>
  );
}

function SplitViewLayout() {
  return (
    <div className="split-view">
      <ProjectList />
      <Outlet /> {/* ProjectDetail renders here */}
    </div>
  );
}
```

### Context-Providing Layouts

```typescript
import { createContext, useContext } from 'react';

interface DashboardContextType {
  user: User;
  settings: Settings;
  refetch: () => void;
}

const DashboardContext = createContext<DashboardContextType | null>(null);

export function useDashboard() {
  const context = useContext(DashboardContext);
  if (!context) throw new Error('useDashboard must be used within DashboardLayout');
  return context;
}

function DashboardLayout() {
  const data = useLoaderData() as DashboardData;
  const revalidator = useRevalidator();
  
  const value: DashboardContextType = {
    user: data.user,
    settings: data.settings,
    refetch: revalidator.revalidate
  };
  
  return (
    <DashboardContext.Provider value={value}>
      <div className="dashboard-layout">
        <Sidebar />
        <main>
          <Outlet />
        </main>
      </div>
    </DashboardContext.Provider>
  );
}

// Child components
function ProjectList() {
  const { user, refetch } = useDashboard();
  
  return (
    <div>
      <h1>{user.name}'s Projects</h1>
      <button onClick={refetch}>Refresh</button>
      {/* ... */}
    </div>
  );
}
```

## Index Routes

### Basic Index Route

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        index: true, // Renders at parent's path
        element: <Home />
      },
      {
        path: 'about',
        element: <About />
      }
    ]
  }
]);

// URL: / → renders Home inside Root
// URL: /about → renders About inside Root
```

### Index Routes at Different Levels

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        index: true,
        element: <LandingPage /> // Renders at /
      },
      {
        path: 'dashboard',
        element: <DashboardLayout />,
        children: [
          {
            index: true,
            element: <DashboardHome /> // Renders at /dashboard
          },
          {
            path: 'projects',
            element: <ProjectsLayout />,
            children: [
              {
                index: true,
                element: <ProjectList /> // Renders at /dashboard/projects
              },
              {
                path: ':id',
                element: <ProjectDetail /> // Renders at /dashboard/projects/123
              }
            ]
          }
        ]
      }
    ]
  }
]);
```

### Index Route with Data

```typescript
const router = createBrowserRouter([
  {
    path: 'inbox',
    element: <InboxLayout />,
    children: [
      {
        index: true,
        element: <InboxEmpty />,
        loader: async () => {
          const unreadCount = await fetchUnreadCount();
          return { unreadCount };
        }
      },
      {
        path: ':messageId',
        element: <Message />,
        loader: messageLoader
      }
    ]
  }
]);

function InboxEmpty() {
  const { unreadCount } = useLoaderData() as { unreadCount: number };
  
  return (
    <div className="empty-state">
      <p>You have {unreadCount} unread messages</p>
      <p>Select a message to read</p>
    </div>
  );
}
```

## Outlet Component

### Basic Outlet Usage

```typescript
import { Outlet, useOutletContext } from 'react-router-dom';

function ParentLayout() {
  return (
    <div>
      <header>Parent Header</header>
      <Outlet /> {/* Child routes render here */}
      <footer>Parent Footer</footer>
    </div>
  );
}
```

### Outlet with Context

```typescript
interface OutletContextType {
  user: User;
  theme: 'light' | 'dark';
  updateTheme: (theme: 'light' | 'dark') => void;
}

function DashboardLayout() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const { user } = useLoaderData() as { user: User };
  
  const context: OutletContextType = {
    user,
    theme,
    updateTheme: setTheme
  };
  
  return (
    <div className={`dashboard ${theme}`}>
      <Sidebar />
      <main>
        <Outlet context={context} />
      </main>
    </div>
  );
}

// Child component
function Settings() {
  const { user, theme, updateTheme } = useOutletContext<OutletContextType>();
  
  return (
    <div>
      <h1>{user.name}'s Settings</h1>
      <button onClick={() => updateTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </div>
  );
}
```

### Conditional Outlet Rendering

```typescript
function ConditionalLayout() {
  const { isAuthorized } = useLoaderData() as { isAuthorized: boolean };
  
  if (!isAuthorized) {
    return <Navigate to="/login" replace />;
  }
  
  return (
    <div>
      <Header />
      <Outlet />
      <Footer />
    </div>
  );
}
```

### Outlet with Animations

```typescript
import { AnimatePresence, motion } from 'framer-motion';
import { useLocation, useOutlet } from 'react-router-dom';

function AnimatedLayout() {
  const location = useLocation();
  const outlet = useOutlet();
  
  return (
    <div>
      <Header />
      <AnimatePresence mode="wait">
        <motion.div
          key={location.pathname}
          initial={{ opacity: 0, x: -20 }}
          animate={{ opacity: 1, x: 0 }}
          exit={{ opacity: 0, x: 20 }}
          transition={{ duration: 0.3 }}
        >
          {outlet}
        </motion.div>
      </AnimatePresence>
      <Footer />
    </div>
  );
}
```

## Pathless Routes

### Grouping Routes with Shared Layout

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        // Pathless route - no path segment added
        element: <AuthLayout />,
        children: [
          {
            path: 'dashboard',
            element: <Dashboard />
          },
          {
            path: 'profile',
            element: <Profile />
          },
          {
            path: 'settings',
            element: <Settings />
          }
        ]
      }
    ]
  }
]);

// URLs remain: /dashboard, /profile, /settings
// But all share AuthLayout wrapper
```

### Pathless Routes for Error Boundaries

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        // Pathless route with error boundary
        errorElement: <DashboardError />,
        children: [
          {
            path: 'dashboard',
            element: <Dashboard />,
            loader: dashboardLoader
          },
          {
            path: 'analytics',
            element: <Analytics />,
            loader: analyticsLoader
          }
        ]
      }
    ]
  }
]);
```

### Pathless Routes for Context

```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      {
        element: <WorkspaceProvider />, // Provides workspace context
        loader: workspaceLoader,
        children: [
          {
            path: 'projects',
            element: <Projects />
          },
          {
            path: 'team',
            element: <Team />
          }
        ]
      }
    ]
  }
]);

function WorkspaceProvider() {
  const workspace = useLoaderData() as Workspace;
  
  return (
    <WorkspaceContext.Provider value={workspace}>
      <Outlet />
    </WorkspaceContext.Provider>
  );
}
```

## Advanced Patterns

### Master-Detail Layout

```typescript
const router = createBrowserRouter([
  {
    path: '/messages',
    element: <MessagesLayout />,
    loader: messagesLoader,
    children: [
      {
        index: true,
        element: <NoMessageSelected />
      },
      {
        path: ':messageId',
        element: <MessageDetail />,
        loader: messageDetailLoader
      }
    ]
  }
]);

function MessagesLayout() {
  const { messages } = useLoaderData() as { messages: Message[] };
  const { messageId } = useParams();
  
  return (
    <div className="split-layout">
      <aside className="message-list">
        {messages.map(msg => (
          <Link
            key={msg.id}
            to={msg.id}
            className={msg.id === messageId ? 'active' : ''}
          >
            <div>{msg.subject}</div>
            <div>{msg.preview}</div>
          </Link>
        ))}
      </aside>
      <main className="message-detail">
        <Outlet />
      </main>
    </div>
  );
}
```

### Tab-Based Navigation

```typescript
function ProfileLayout() {
  const location = useLocation();
  
  const tabs = [
    { path: '', label: 'Overview' },
    { path: 'posts', label: 'Posts' },
    { path: 'followers', label: 'Followers' },
    { path: 'following', label: 'Following' }
  ];
  
  return (
    <div>
      <div className="tabs">
        {tabs.map(tab => (
          <Link
            key={tab.path}
            to={tab.path}
            className={location.pathname.endsWith(tab.path) ? 'active' : ''}
          >
            {tab.label}
          </Link>
        ))}
      </div>
      <div className="tab-content">
        <Outlet />
      </div>
    </div>
  );
}

const router = createBrowserRouter([
  {
    path: '/profile/:userId',
    element: <ProfileLayout />,
    children: [
      {
        index: true,
        element: <ProfileOverview />
      },
      {
        path: 'posts',
        element: <ProfilePosts />
      },
      {
        path: 'followers',
        element: <ProfileFollowers />
      },
      {
        path: 'following',
        element: <ProfileFollowing />
      }
    ]
  }
]);
```

### Breadcrumb Navigation

```typescript
import { useMatches } from 'react-router-dom';

interface RouteHandle {
  crumb?: (data?: any) => React.ReactNode;
}

const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    handle: {
      crumb: () => <Link to="/">Home</Link>
    } as RouteHandle,
    children: [
      {
        path: 'projects',
        element: <Projects />,
        handle: {
          crumb: () => <Link to="/projects">Projects</Link>
        } as RouteHandle,
        loader: projectsLoader,
        children: [
          {
            path: ':projectId',
            element: <ProjectDetail />,
            loader: projectLoader,
            handle: {
              crumb: (data: { project: Project }) => (
                <Link to={`/projects/${data.project.id}`}>
                  {data.project.name}
                </Link>
              )
            } as RouteHandle
          }
        ]
      }
    ]
  }
]);

function Breadcrumbs() {
  const matches = useMatches();
  
  const crumbs = matches
    .filter(match => (match.handle as RouteHandle)?.crumb)
    .map((match, index) => {
      const handle = match.handle as RouteHandle;
      return (
        <span key={index}>
          {handle.crumb!(match.data)}
          {index < matches.length - 1 && ' / '}
        </span>
      );
    });
  
  return <nav className="breadcrumbs">{crumbs}</nav>;
}
```

### Modal Routes

```typescript
function App() {
  const location = useLocation();
  const state = location.state as { backgroundLocation?: Location };
  
  return (
    <>
      <Routes location={state?.backgroundLocation || location}>
        <Route path="/" element={<Home />} />
        <Route path="/gallery" element={<Gallery />} />
        <Route path="/gallery/:id" element={<ImageDetail />} />
      </Routes>
      
      {state?.backgroundLocation && (
        <Routes>
          <Route path="/gallery/:id" element={<ImageModal />} />
        </Routes>
      )}
    </>
  );
}

function Gallery() {
  const location = useLocation();
  
  return (
    <div className="gallery">
      {images.map(img => (
        <Link
          key={img.id}
          to={`/gallery/${img.id}`}
          state={{ backgroundLocation: location }}
        >
          <img src={img.thumbnail} alt={img.title} />
        </Link>
      ))}
    </div>
  );
}

function ImageModal() {
  const navigate = useNavigate();
  const { id } = useParams();
  
  return (
    <div className="modal-overlay" onClick={() => navigate(-1)}>
      <div className="modal-content">
        <img src={`/images/${id}`} alt="Full size" />
      </div>
    </div>
  );
}
```

## Common Mistakes

### 1. Forgetting Outlet

```typescript
// ❌ Bad - child routes won't render
function Layout() {
  return (
    <div>
      <Header />
      <main>
        {/* Missing <Outlet /> */}
      </main>
    </div>
  );
}

// ✅ Good
function Layout() {
  return (
    <div>
      <Header />
      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

### 2. Incorrect Path Nesting

```typescript
// ❌ Bad - paths don't compose correctly
const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Dashboard />,
    children: [
      {
        path: '/projects', // Leading slash breaks nesting
        element: <Projects />
      }
    ]
  }
]);

// ✅ Good - relative paths
const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Dashboard />,
    children: [
      {
        path: 'projects', // No leading slash
        element: <Projects />
      }
    ]
  }
]);
```

### 3. Index Route with Path

```typescript
// ❌ Bad - can't have both index and path
{
  index: true,
  path: 'home', // Error!
  element: <Home />
}

// ✅ Good - use one or the other
{
  index: true,
  element: <Home />
}
```

### 4. Not Handling Loading States

```typescript
// ❌ Bad - no loading feedback
function Layout() {
  return (
    <div>
      <Sidebar />
      <Outlet />
    </div>
  );
}

// ✅ Good - show loading state
function Layout() {
  const navigation = useNavigation();
  
  return (
    <div>
      <Sidebar />
      <main className={navigation.state === 'loading' ? 'loading' : ''}>
        <Outlet />
      </main>
    </div>
  );
}
```

## Best Practices

### 1. Organize Routes by Feature

```typescript
// routes/dashboard.tsx
export const dashboardRoutes: RouteObject[] = [
  {
    path: 'dashboard',
    element: <DashboardLayout />,
    children: [
      {
        index: true,
        element: <DashboardHome />
      },
      {
        path: 'analytics',
        element: <Analytics />
      }
    ]
  }
];

// routes/index.tsx
export const routes: RouteObject[] = [
  {
    path: '/',
    element: <Root />,
    children: [
      ...dashboardRoutes,
      ...profileRoutes,
      ...settingsRoutes
    ]
  }
];
```

### 2. Use Layouts for Shared UI

```typescript
// Reusable layout components
function TwoColumnLayout() {
  return (
    <div className="two-column">
      <aside><Outlet context={{ position: 'sidebar' }} /></aside>
      <main><Outlet context={{ position: 'main' }} /></main>
    </div>
  );
}

// Apply to multiple route groups
const routes = [
  {
    path: '/admin',
    element: <TwoColumnLayout />,
    children: [/* admin routes */]
  },
  {
    path: '/dashboard',
    element: <TwoColumnLayout />,
    children: [/* dashboard routes */]
  }
];
```

### 3. Leverage Index Routes

```typescript
// Clear default views at each level
const router = createBrowserRouter([
  {
    path: '/',
    element: <Root />,
    children: [
      { index: true, element: <Home /> }, // Default at /
      {
        path: 'products',
        element: <ProductsLayout />,
        children: [
          { index: true, element: <ProductList /> }, // Default at /products
          {
            path: ':id',
            element: <ProductDetail />
          }
        ]
      }
    ]
  }
]);
```

### 4. Type-Safe Outlet Context

```typescript
// Create typed hook
function createOutletContext<T>() {
  return () => useOutletContext<T>();
}

interface DashboardContextType {
  user: User;
  workspace: Workspace;
}

const useDashboardContext = createOutletContext<DashboardContextType>();

// In layout
function DashboardLayout() {
  const context: DashboardContextType = {
    user: data.user,
    workspace: data.workspace
  };
  
  return (
    <div>
      <Outlet context={context} />
    </div>
  );
}

// In child
function DashboardPage() {
  const { user, workspace } = useDashboardContext();
  return <div>{user.name} in {workspace.name}</div>;
}
```

## Interview Questions

### Q1: What's the difference between nesting routes in the config vs using multiple Routes components?

**Answer:** Nested routes in config:
- Enable data loading in parallel for all matched routes
- Support layout composition with Outlet
- Provide hierarchical error boundaries
- Allow parent loaders to run before child navigation

Multiple Routes components lack these benefits and cause waterfall loading.

### Q2: When should you use an index route vs a regular child route?

**Answer:** Use index routes when:
- You need a default view at the parent's exact path
- The parent is a layout/container
- You want `/parent` to show something without redirecting

Use regular routes when:
- The path should extend the parent's path
- It's a distinct destination (e.g., `/parent/child`)

### Q3: How does React Router determine which route to render?

**Answer:**
1. Ranks all routes by specificity (static > dynamic > splat)
2. Matches URL against route paths from most to least specific
3. Renders all matched routes in hierarchy (parent → children)
4. Each parent's Outlet renders its matched child

### Q4: How do pathless routes differ from regular routes?

**Answer:** Pathless routes:
- Don't add segments to the URL
- Used for grouping, layouts, error boundaries, context
- Still participate in the matching hierarchy
- Children inherit parent's path as base

Regular routes add their path segment to the URL.

## Resources

### Official Documentation
- React Router Nested Routes: https://reactrouter.com/en/main/start/concepts#nested-routes
- Outlet API: https://reactrouter.com/en/main/components/outlet

### Tutorials
- Building Nested Layouts: https://reactrouter.com/en/main/start/tutorial#nested-routes

### Articles
- Route Module Organization: https://remix.run/docs/en/main/discussion/routes

### Examples
- Nested Routes Demo: https://github.com/remix-run/react-router/tree/main/examples/basic

---

**Next Steps:**
- Practice building multi-level nested layouts
- Implement master-detail patterns
- Create reusable layout components
- Build breadcrumb navigation systems
