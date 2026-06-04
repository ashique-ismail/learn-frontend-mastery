# Module Federation

## Overview

Module Federation is a revolutionary Webpack 5 feature that enables JavaScript applications to dynamically load code from another application at runtime. It's a game-changer for micro-frontend architectures, allowing independent teams to develop, deploy, and scale their applications autonomously while sharing code seamlessly. Think of it as "npm install" but at runtime instead of build time.

## Core Concepts

### Module Federation Architecture

```
┌─────────────────────────────────────┐
│         Host Application            │
│  ┌──────────────────────────────┐   │
│  │    Local Modules             │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │    Federated Modules         │   │
│  │  ┌────────┐      ┌────────┐  │   │
│  │  │Remote 1│      │Remote 2│  │   │
│  │  └────┬───┘      └───┬────┘  │   │
│  └───────┼──────────────┼───────┘   │
└──────────┼──────────────┼───────────┘
           │              │
     ┌─────▼──────┐ ┌────▼──────┐
     │ Remote App │ │ Remote App│
     │     #1     │ │     #2    │
     └────────────┘ └───────────┘
```

### Key Terminology

- **Host**: Application that consumes federated modules
- **Remote**: Application that exposes modules for consumption
- **Bidirectional**: Application acting as both host and remote
- **Shared Dependencies**: Libraries shared between host and remotes
- **Container**: Entry point for federated module

## Basic Setup

### Simple Remote Configuration

```javascript
// remote-app/webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');
const packageJson = require('./package.json');

module.exports = {
  output: {
    publicPath: 'http://localhost:3001/',
    uniqueName: 'remoteApp'
  },
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'remoteApp',
      filename: 'remoteEntry.js',
      
      exposes: {
        './Button': './src/components/Button',
        './Header': './src/components/Header',
        './utils': './src/utils'
      },
      
      shared: {
        react: {
          singleton: true,
          requiredVersion: packageJson.dependencies.react
        },
        'react-dom': {
          singleton: true,
          requiredVersion: packageJson.dependencies['react-dom']
        }
      }
    })
  ]
};
```

### Simple Host Configuration

```javascript
// host-app/webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  output: {
    publicPath: 'http://localhost:3000/'
  },
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'hostApp',
      
      remotes: {
        remoteApp: 'remoteApp@http://localhost:3001/remoteEntry.js'
      },
      
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};
```

### Consuming Remote Components

```typescript
// host-app/src/App.tsx
import React, { lazy, Suspense } from 'react';

// Dynamic import from remote
const RemoteButton = lazy(() => import('remoteApp/Button'));
const RemoteHeader = lazy(() => import('remoteApp/Header'));

function App() {
  return (
    <div>
      <h1>Host Application</h1>
      
      <Suspense fallback={<div>Loading Remote Header...</div>}>
        <RemoteHeader />
      </Suspense>
      
      <Suspense fallback={<div>Loading Remote Button...</div>}>
        <RemoteButton onClick={() => alert('Clicked!')}>
          Click Me
        </RemoteButton>
      </Suspense>
    </div>
  );
}

export default App;
```

## Advanced Configuration

### Bidirectional Setup

```javascript
// app1/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      
      // Expose modules (acting as remote)
      exposes: {
        './ProductList': './src/components/ProductList',
        './Cart': './src/components/Cart'
      },
      
      // Consume modules (acting as host)
      remotes: {
        app2: 'app2@http://localhost:3002/remoteEntry.js'
      },
      
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        'react-router-dom': { singleton: true }
      }
    })
  ]
};

// app2/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app2',
      filename: 'remoteEntry.js',
      
      exposes: {
        './UserProfile': './src/components/UserProfile',
        './Analytics': './src/components/Analytics'
      },
      
      remotes: {
        app1: 'app1@http://localhost:3001/remoteEntry.js'
      },
      
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        'react-router-dom': { singleton: true }
      }
    })
  ]
};
```

### Dynamic Remote Loading

```typescript
// Dynamic remote configuration
function loadRemote(url: string, scope: string, module: string) {
  return async () => {
    // Load the remote container
    await __webpack_init_sharing__('default');
    const container = (window as any)[scope];
    
    if (!container) {
      // Inject script if not loaded
      await new Promise<void>((resolve, reject) => {
        const script = document.createElement('script');
        script.src = url;
        script.onload = () => {
          const container = (window as any)[scope];
          container.init(__webpack_share_scopes__.default);
          resolve();
        };
        script.onerror = reject;
        document.head.appendChild(script);
      });
    }
    
    // Get the module
    const factory = await container.get(module);
    return factory();
  };
}

// Usage
const RemoteComponent = lazy(() =>
  loadRemote(
    'http://localhost:3001/remoteEntry.js',
    'remoteApp',
    './Button'
  )
);
```

### Runtime Remote Configuration

```typescript
// remoteConfig.ts
interface RemoteConfig {
  url: string;
  scope: string;
  module: string;
}

const remotes: Record<string, RemoteConfig> = {
  development: {
    url: 'http://localhost:3001/remoteEntry.js',
    scope: 'remoteApp',
    module: './Button'
  },
  staging: {
    url: 'https://staging.example.com/remoteEntry.js',
    scope: 'remoteApp',
    module: './Button'
  },
  production: {
    url: 'https://production.example.com/remoteEntry.js',
    scope: 'remoteApp',
    module: './Button'
  }
};

export function getRemoteConfig(env: string = process.env.NODE_ENV): RemoteConfig {
  return remotes[env] || remotes.production;
}

// Dynamic loader with error handling
export function createRemoteLoader(config: RemoteConfig) {
  return async () => {
    try {
      return await loadRemote(config.url, config.scope, config.module);
    } catch (error) {
      console.error('Failed to load remote:', error);
      // Fallback component
      return () => <div>Failed to load remote component</div>;
    }
  };
}

// Usage in component
import { lazy } from 'react';
import { getRemoteConfig, createRemoteLoader } from './remoteConfig';

const config = getRemoteConfig();
const RemoteButton = lazy(createRemoteLoader(config));
```

## Shared Dependencies

### Advanced Shared Configuration

```javascript
// webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app',
      
      shared: {
        // Simple sharing
        react: { singleton: true },
        
        // With version requirements
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
          strictVersion: false // Allow version mismatch warnings
        },
        
        // Custom sharing
        lodash: {
          singleton: false, // Allow multiple versions
          requiredVersion: false,
          eager: false, // Load asynchronously
          shareScope: 'default'
        },
        
        // Share specific package files
        '@mui/material': {
          singleton: true,
          requiredVersion: '^5.0.0',
          shareKey: '@mui/material'
        },
        
        // Share all dependencies automatically
        ...Object.keys(packageJson.dependencies).reduce((acc, dep) => {
          acc[dep] = {
            singleton: true,
            requiredVersion: packageJson.dependencies[dep]
          };
          return acc;
        }, {})
      }
    })
  ]
};
```

### Version Conflict Resolution

```javascript
// Handling version conflicts
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
          strictVersion: false,
          version: '18.2.0',
          
          // Custom sharing logic
          shareScope: 'default',
          
          // Eagerly load shared module
          eager: false,
          
          // Provide fallback
          import: 'react',
          packageName: 'react'
        }
      }
    })
  ]
};

// Version compatibility check
function checkCompatibility(required: string, actual: string): boolean {
  // Implement semantic version checking
  const [reqMajor] = required.split('.');
  const [actMajor] = actual.split('.');
  return reqMajor === actMajor;
}
```

## React Integration

### Federated React App

```typescript
// remote/src/bootstrap.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(
  document.getElementById('root') as HTMLElement
);

root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

// remote/src/index.tsx
// Import bootstrap dynamically to ensure shared dependencies load first
import('./bootstrap');

// remote/src/components/Button.tsx
import React from 'react';

interface ButtonProps {
  children: React.ReactNode;
  onClick?: () => void;
  variant?: 'primary' | 'secondary';
}

export const Button: React.FC<ButtonProps> = ({
  children,
  onClick,
  variant = 'primary'
}) => {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
    >
      {children}
    </button>
  );
};

// host/src/App.tsx
import React, { lazy, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

const RemoteButton = lazy(() => import('remoteApp/Button'));

function App() {
  return (
    <div>
      <h1>Host Application</h1>
      
      <ErrorBoundary
        fallback={<div>Failed to load remote component</div>}
        onError={(error) => console.error('Remote load error:', error)}
      >
        <Suspense fallback={<div>Loading...</div>}>
          <RemoteButton variant="primary" onClick={() => alert('Clicked!')}>
            Remote Button
          </RemoteButton>
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}

export default App;
```

### Shared State Management

```typescript
// shared-state/src/store.ts
import { create } from 'zustand';

interface AppState {
  user: User | null;
  setUser: (user: User | null) => void;
  cart: CartItem[];
  addToCart: (item: CartItem) => void;
}

export const useAppStore = create<AppState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
  cart: [],
  addToCart: (item) => set((state) => ({
    cart: [...state.cart, item]
  }))
}));

// webpack.config.js for shared-state module
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'sharedState',
      filename: 'remoteEntry.js',
      
      exposes: {
        './store': './src/store'
      },
      
      shared: {
        zustand: { singleton: true },
        react: { singleton: true }
      }
    })
  ]
};

// Using shared state in host
import { useAppStore } from 'sharedState/store';

function HostComponent() {
  const { user, setUser } = useAppStore();
  
  return (
    <div>
      <p>User: {user?.name}</p>
      <button onClick={() => setUser({ name: 'John' })}>
        Set User
      </button>
    </div>
  );
}

// Using shared state in remote
import { useAppStore } from 'sharedState/store';

function RemoteComponent() {
  const { cart, addToCart } = useAppStore();
  
  return (
    <div>
      <p>Cart items: {cart.length}</p>
      <button onClick={() => addToCart({ id: '1', name: 'Product' })}>
        Add to Cart
      </button>
    </div>
  );
}
```

## TypeScript Support

```typescript
// Type declarations for remote modules
// host/src/types/remotes.d.ts
declare module 'remoteApp/Button' {
  import { FC } from 'react';
  
  interface ButtonProps {
    children: React.ReactNode;
    onClick?: () => void;
    variant?: 'primary' | 'secondary';
  }
  
  export const Button: FC<ButtonProps>;
}

declare module 'remoteApp/Header' {
  import { FC } from 'react';
  
  interface HeaderProps {
    title: string;
  }
  
  export const Header: FC<HeaderProps>;
}

// Generate types automatically
// remote/src/types/index.ts
export type { ButtonProps } from '../components/Button';
export type { HeaderProps } from '../components/Header';

// remote/webpack.config.js
const { TsconfigPathsPlugin } = require('tsconfig-paths-webpack-plugin');

module.exports = {
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
    plugins: [new TsconfigPathsPlugin()]
  },
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'remoteApp',
      exposes: {
        './Button': './src/components/Button',
        './types': './src/types'
      }
    })
  ]
};
```

## Error Handling and Fallbacks

```typescript
// Remote loader with retry
async function loadRemoteWithRetry(
  url: string,
  scope: string,
  module: string,
  maxRetries: number = 3
): Promise<any> {
  let lastError: Error;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await loadRemote(url, scope, module);
    } catch (error) {
      lastError = error as Error;
      console.warn(`Retry ${i + 1}/${maxRetries} failed:`, error);
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
  
  throw lastError!;
}

// Fallback component
function RemoteFallback({ error }: { error?: Error }) {
  return (
    <div className="remote-fallback">
      <p>Unable to load remote component</p>
      {error && <p>Error: {error.message}</p>}
      <button onClick={() => window.location.reload()}>
        Reload Page
      </button>
    </div>
  );
}

// Component with fallback
function App() {
  return (
    <ErrorBoundary FallbackComponent={RemoteFallback}>
      <Suspense fallback={<LoadingSpinner />}>
        <RemoteButton />
      </Suspense>
    </ErrorBoundary>
  );
}

// Health check for remotes
async function checkRemoteHealth(url: string): Promise<boolean> {
  try {
    const response = await fetch(url, { method: 'HEAD' });
    return response.ok;
  } catch {
    return false;
  }
}

// Conditional loading
const RemoteButton = lazy(async () => {
  const isHealthy = await checkRemoteHealth('http://localhost:3001/remoteEntry.js');
  
  if (isHealthy) {
    return import('remoteApp/Button');
  } else {
    // Load local fallback
    return import('./components/FallbackButton');
  }
});
```

## Deployment Strategies

### CDN Deployment

```javascript
// Production webpack config with CDN
module.exports = {
  output: {
    publicPath: 'https://cdn.example.com/app1/',
  },
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      
      remotes: {
        app2: 'app2@https://cdn.example.com/app2/remoteEntry.js',
        app3: 'app3@https://cdn.example.com/app3/remoteEntry.js'
      }
    })
  ]
};

// Dynamic public path
// webpack.config.js
module.exports = {
  output: {
    publicPath: 'auto' // Automatically determine public path
  }
};

// Or set at runtime
// public-path.ts
__webpack_public_path__ = window.RUNTIME_PUBLIC_PATH || '/';
```

### Version Management

```typescript
// version-manager.ts
interface RemoteVersion {
  name: string;
  version: string;
  url: string;
}

class RemoteVersionManager {
  private versions: Map<string, RemoteVersion> = new Map();
  
  async fetchVersions(): Promise<void> {
    const response = await fetch('/api/remote-versions');
    const versions: RemoteVersion[] = await response.json();
    
    versions.forEach(v => {
      this.versions.set(v.name, v);
    });
  }
  
  getRemoteUrl(name: string): string {
    const version = this.versions.get(name);
    return version ? version.url : '';
  }
  
  async checkForUpdates(): Promise<void> {
    await this.fetchVersions();
    // Trigger re-render if versions changed
  }
}

export const versionManager = new RemoteVersionManager();

// Usage
await versionManager.fetchVersions();
const remoteUrl = versionManager.getRemoteUrl('remoteApp');
```

## Performance Optimization

```javascript
// Preload remotes
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      remotes: {
        app1: 'promise new Promise(resolve => {
          const script = document.createElement("script");
          script.src = "http://localhost:3001/remoteEntry.js";
          script.onload = () => resolve(window.app1);
          document.head.appendChild(script);
        })'
      }
    }),
    
    // Prefetch remote
    new HtmlWebpackPlugin({
      templateContent: `
        <!DOCTYPE html>
        <html>
          <head>
            <link rel="prefetch" href="http://localhost:3001/remoteEntry.js" />
          </head>
          <body>
            <div id="root"></div>
          </body>
        </html>
      `
    })
  ]
};

// Lazy loading with preload
const RemoteComponent = lazy(() => {
  // Preload the chunk
  const preloadLink = document.createElement('link');
  preloadLink.rel = 'preload';
  preloadLink.as = 'script';
  preloadLink.href = 'http://localhost:3001/remoteEntry.js';
  document.head.appendChild(preloadLink);
  
  return import('remoteApp/Component');
});
```

## Common Mistakes

1. **Not using singleton for React/ReactDOM** - causes multiple React instances
2. **Forgetting bootstrap file** - shared dependencies not initialized
3. **Incorrect publicPath** - broken remote loading
4. **Version conflicts** - incompatible shared dependencies
5. **Missing error boundaries** - crashes when remote fails
6. **Not handling loading states** - poor UX
7. **Hardcoded remote URLs** - difficult environment management
8. **Not versioning remotes** - cache issues

## Best Practices

1. Always use singleton for React and major libraries
2. Implement error boundaries around remote components
3. Use environment-specific remote URLs
4. Version your remotes properly
5. Add health checks before loading remotes
6. Implement fallback components
7. Monitor remote load performance
8. Document exposed components and APIs
9. Use TypeScript for type safety
10. Test remote integration thoroughly

## Key Takeaways

1. Module Federation enables runtime code sharing
2. Perfect for micro-frontend architectures
3. Shared dependencies reduce bundle duplication
4. Requires careful version management
5. Error handling is critical for production
6. Dynamic remotes provide deployment flexibility
7. TypeScript support requires manual declarations
8. Performance benefits from code splitting
9. Independent deployment of remotes
10. Revolutionizes frontend architecture patterns

## Resources

- [Webpack Module Federation](https://webpack.js.org/concepts/module-federation/)
- [Module Federation Examples](https://github.com/module-federation/module-federation-examples)
- [Micro Frontends](https://micro-frontends.org/)
- [Zack Jackson Blog](https://medium.com/@ScriptedAlchemy)
- [Module Federation Plugin Docs](https://webpack.js.org/plugins/module-federation-plugin/)
