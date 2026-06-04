# Monorepos

## Table of Contents
1. [Introduction](#introduction)
2. [Monorepo vs Polyrepo](#monorepo-vs-polyrepo)
3. [Monorepo Structure](#monorepo-structure)
4. [Workspace Management](#workspace-management)
5. [Implementation Examples](#implementation-examples)
6. [Common Mistakes](#common-mistakes)
7. [Best Practices](#best-practices)
8. [When to Use/Not to Use](#when-to-usenot-to-use)
9. [Interview Questions](#interview-questions)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## Introduction

A monorepo (monolithic repository) is a software development strategy where code for many projects is stored in a single repository. Rather than having separate repositories for each project, library, or microservice, everything lives together with shared tooling, dependencies, and workflows.

Major companies like Google, Facebook, Microsoft, and Uber use monorepos to manage thousands of projects and millions of lines of code.

## Monorepo vs Polyrepo

### Polyrepo Structure

```
company-repos/
├── web-app/               # Separate repo
│   └── .git
├── mobile-app/            # Separate repo
│   └── .git
├── shared-ui-lib/         # Separate repo
│   └── .git
├── api-gateway/           # Separate repo
│   └── .git
└── user-service/          # Separate repo
    └── .git
```

Problems:
- Dependency management across repos
- Version synchronization challenges
- Code sharing requires publishing packages
- Changes span multiple PRs
- Inconsistent tooling and standards

### Monorepo Structure

```
company-monorepo/
├── .git                   # Single repo
├── apps/
│   ├── web-app/
│   ├── mobile-app/
│   └── admin-dashboard/
├── packages/
│   ├── shared-ui/
│   ├── api-client/
│   └── utils/
└── services/
    ├── api-gateway/
    └── user-service/
```

Benefits:
- Atomic changes across projects
- Easier code sharing and refactoring
- Consistent tooling and standards
- Single source of truth
- Better code reuse

## Monorepo Structure

### Standard Monorepo Layout

```
my-monorepo/
├── apps/                          # Applications
│   ├── web/                       # Web app
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── mobile/                    # Mobile app
│   │   ├── src/
│   │   └── package.json
│   └── admin/                     # Admin dashboard
│       ├── src/
│       └── package.json
├── packages/                      # Shared libraries
│   ├── ui-components/             # UI component library
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── api-client/                # API client
│   │   ├── src/
│   │   └── package.json
│   ├── utils/                     # Utilities
│   │   ├── src/
│   │   └── package.json
│   └── types/                     # Shared types
│       ├── src/
│       └── package.json
├── tools/                         # Build tools and scripts
│   ├── scripts/
│   └── generators/
├── package.json                   # Root package.json
├── tsconfig.base.json            # Base TypeScript config
├── nx.json                       # Nx configuration
└── turbo.json                    # Turbo configuration
```

### Workspace Dependencies

```
┌─────────────────────────────────────────────────┐
│                  Apps Layer                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │   Web   │  │  Mobile │  │  Admin  │         │
│  └────┬────┘  └────┬────┘  └────┬────┘         │
│       │            │             │               │
└───────┼────────────┼─────────────┼──────────────┘
        │            │             │
        ▼            ▼             ▼
┌─────────────────────────────────────────────────┐
│              Packages Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │    UI    │  │   API    │  │  Utils   │      │
│  │Components│  │  Client  │  │          │      │
│  └──────────┘  └──────────┘  └──────────┘      │
└─────────────────────────────────────────────────┘
```

## Workspace Management

### npm Workspaces

```json
// Root package.json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces",
    "dev:web": "npm run dev --workspace=apps/web",
    "dev:mobile": "npm run dev --workspace=apps/mobile"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "eslint": "^8.0.0"
  }
}

// apps/web/package.json
{
  "name": "@mycompany/web",
  "version": "1.0.0",
  "dependencies": {
    "@mycompany/ui-components": "*",
    "@mycompany/api-client": "*",
    "react": "^18.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest"
  }
}

// packages/ui-components/package.json
{
  "name": "@mycompany/ui-components",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "peerDependencies": {
    "react": "^18.0.0"
  },
  "scripts": {
    "build": "tsup src/index.ts --dts",
    "test": "vitest"
  }
}
```

### Yarn Workspaces

```json
// package.json
{
  "private": true,
  "workspaces": {
    "packages": [
      "apps/*",
      "packages/*"
    ]
  },
  "scripts": {
    "build": "yarn workspaces run build",
    "test": "yarn workspaces run test"
  }
}
```

### pnpm Workspaces

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'services/*'
```

```json
// package.json
{
  "scripts": {
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "dev:web": "pnpm --filter web dev"
  }
}
```

## Implementation Examples

### Example 1: Basic Monorepo with npm Workspaces

```typescript
// ============================================
// packages/utils/src/index.ts
// ============================================

export function formatCurrency(amount: number, currency: string = 'USD'): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency
  }).format(amount);
}

export function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: NodeJS.Timeout;
  return (...args: Parameters<T>) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}

// ============================================
// packages/utils/package.json
// ============================================

{
  "name": "@mycompany/utils",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsup src/index.ts --dts --format cjs,esm",
    "test": "vitest",
    "dev": "tsup src/index.ts --dts --format cjs,esm --watch"
  },
  "devDependencies": {
    "tsup": "^7.0.0",
    "typescript": "^5.0.0",
    "vitest": "^0.34.0"
  }
}

// ============================================
// packages/ui-components/src/Button.tsx
// ============================================

import { ReactNode, ButtonHTMLAttributes } from 'react';
import './Button.css';

export interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  children: ReactNode;
}

export function Button({
  variant = 'primary',
  size = 'medium',
  children,
  className = '',
  ...props
}: ButtonProps) {
  const classes = `btn btn-${variant} btn-${size} ${className}`.trim();
  
  return (
    <button className={classes} {...props}>
      {children}
    </button>
  );
}

// ============================================
// packages/ui-components/src/Card.tsx
// ============================================

import { ReactNode } from 'react';
import './Card.css';

export interface CardProps {
  title?: string;
  children: ReactNode;
  footer?: ReactNode;
  className?: string;
}

export function Card({ title, children, footer, className = '' }: CardProps) {
  return (
    <div className={`card ${className}`.trim()}>
      {title && <div className="card-header">{title}</div>}
      <div className="card-body">{children}</div>
      {footer && <div className="card-footer">{footer}</div>}
    </div>
  );
}

// ============================================
// packages/ui-components/src/index.ts
// ============================================

export { Button } from './Button';
export type { ButtonProps } from './Button';
export { Card } from './Card';
export type { CardProps } from './Card';

// ============================================
// packages/ui-components/package.json
// ============================================

{
  "name": "@mycompany/ui-components",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsup src/index.ts --dts --format cjs,esm --external react",
    "dev": "tsup src/index.ts --dts --format cjs,esm --external react --watch",
    "test": "vitest"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.0.0",
    "react": "^18.0.0",
    "tsup": "^7.0.0",
    "typescript": "^5.0.0",
    "vitest": "^0.34.0"
  }
}

// ============================================
// packages/api-client/src/index.ts
// ============================================

export class ApiClient {
  constructor(private baseUrl: string) {}

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  }

  async put<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return response.json();
  }

  async delete(endpoint: string): Promise<void> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'DELETE'
    });
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
  }
}

export const createApiClient = (baseUrl: string) => new ApiClient(baseUrl);

// ============================================
// apps/web/src/App.tsx
// ============================================

import { Button, Card } from '@mycompany/ui-components';
import { formatCurrency } from '@mycompany/utils';
import { createApiClient } from '@mycompany/api-client';
import { useState, useEffect } from 'react';

const api = createApiClient('https://api.example.com');

interface Product {
  id: string;
  name: string;
  price: number;
}

function App() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    api.get<Product[]>('/products')
      .then(setProducts)
      .finally(() => setLoading(false));
  }, []);

  if (loading) {
    return <div>Loading...</div>;
  }

  return (
    <div className="app">
      <h1>Products</h1>
      <div className="product-grid">
        {products.map(product => (
          <Card key={product.id} title={product.name}>
            <p>{formatCurrency(product.price)}</p>
            <Button variant="primary">Add to Cart</Button>
          </Card>
        ))}
      </div>
    </div>
  );
}

export default App;

// ============================================
// apps/web/package.json
// ============================================

{
  "name": "@mycompany/web",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@mycompany/ui-components": "*",
    "@mycompany/utils": "*",
    "@mycompany/api-client": "*",
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@types/react": "^18.0.0",
    "@types/react-dom": "^18.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "typescript": "^5.0.0",
    "vite": "^4.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### Example 2: Nx Monorepo

```json
// nx.json
{
  "extends": "nx/presets/npm.json",
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"]
      }
    }
  },
  "namedInputs": {
    "default": ["{projectRoot}/**/*"],
    "production": ["!{projectRoot}/**/*.spec.ts"]
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    },
    "test": {
      "inputs": ["default", "^production"]
    }
  }
}

// packages/ui-components/project.json
{
  "name": "ui-components",
  "sourceRoot": "packages/ui-components/src",
  "projectType": "library",
  "targets": {
    "build": {
      "executor": "@nrwl/vite:build",
      "outputs": ["{projectRoot}/dist"],
      "options": {
        "outputPath": "dist/packages/ui-components"
      }
    },
    "test": {
      "executor": "@nrwl/vite:test",
      "options": {
        "passWithNoTests": true
      }
    },
    "lint": {
      "executor": "@nrwl/linter:eslint",
      "options": {
        "lintFilePatterns": ["packages/ui-components/**/*.{ts,tsx}"]
      }
    }
  }
}
```

### Example 3: Turborepo

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**/*.tsx", "src/**/*.ts", "test/**/*.ts"]
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}

// Root package.json scripts
{
  "scripts": {
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "dev": "turbo run dev --parallel",
    "dev:web": "turbo run dev --filter=web",
    "build:web": "turbo run build --filter=web..."
  }
}
```

### Example 4: Shared TypeScript Configuration

```json
// tsconfig.base.json (root)
{
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": {
      "@mycompany/ui-components": ["packages/ui-components/src"],
      "@mycompany/utils": ["packages/utils/src"],
      "@mycompany/api-client": ["packages/api-client/src"]
    }
  }
}

// apps/web/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "outDir": "./dist"
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/ui-components" },
    { "path": "../../packages/utils" },
    { "path": "../../packages/api-client" }
  ]
}

// packages/ui-components/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM"],
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "composite": true
  },
  "include": ["src"]
}
```

## Common Mistakes

### 1. Not Managing Dependencies Properly

```json
// BAD: Different versions across packages
// packages/ui-components/package.json
{
  "dependencies": {
    "react": "^17.0.0"
  }
}

// apps/web/package.json
{
  "dependencies": {
    "react": "^18.0.0"  // Version mismatch!
  }
}

// GOOD: Consistent versions
// Root package.json
{
  "devDependencies": {
    "react": "^18.0.0"  // Version defined once
  }
}

// packages/ui-components/package.json
{
  "peerDependencies": {
    "react": "^18.0.0"
  }
}

// apps/web/package.json
{
  "dependencies": {
    "react": "^18.0.0"  // Same version
  }
}
```

### 2. Circular Dependencies

```typescript
// BAD: Packages depending on each other
// packages/auth/index.ts
import { api } from '@mycompany/api-client';

// packages/api-client/index.ts
import { getToken } from '@mycompany/auth';  // Circular!

// GOOD: Extract shared dependency
// packages/shared/index.ts
export interface AuthToken {
  token: string;
  expiresAt: number;
}

// packages/auth/index.ts
import type { AuthToken } from '@mycompany/shared';

// packages/api-client/index.ts
import type { AuthToken } from '@mycompany/shared';
```

### 3. No Build Dependency Graph

```json
// BAD: No dependency ordering
{
  "scripts": {
    "build": "npm run build --workspaces"  // Random order
  }
}

// GOOD: Respect dependency graph
{
  "scripts": {
    "build": "npm run build --workspaces --if-present --workspace-concurrency=Infinity"
  }
}

// Or use Nx/Turbo which handle this automatically
{
  "scripts": {
    "build": "nx run-many --target=build --all"
  }
}
```

## Best Practices

### 1. Use Consistent Naming

```
my-monorepo/
├── apps/
│   ├── @mycompany-web/           # Use scoped names
│   └── @mycompany-mobile/
└── packages/
    ├── @mycompany-ui/
    ├── @mycompany-utils/
    └── @mycompany-api/
```

### 2. Share Configuration

```typescript
// shared-config/
├── eslint-config/
│   └── index.js
├── tsconfig/
│   ├── base.json
│   ├── react.json
│   └── node.json
└── vitest-config/
    └── index.ts

// packages/ui-components/package.json
{
  "eslintConfig": {
    "extends": ["@mycompany/eslint-config"]
  }
}
```

### 3. Optimize Build Caching

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"],
      "inputs": ["src/**/*.{ts,tsx}", "!**/*.test.{ts,tsx}"]
    }
  }
}

// Only rebuilds when source files change, not tests
```

### 4. Use Workspace Protocols

```json
// Use workspace protocol for internal dependencies
{
  "dependencies": {
    "@mycompany/ui-components": "workspace:*",
    "@mycompany/utils": "workspace:^1.0.0"
  }
}
```

## When to Use/Not to Use

### Use Monorepo When:

1. **Multiple Related Projects**: Web, mobile, shared libraries
2. **Frequent Cross-Project Changes**: Changes span multiple projects
3. **Shared Dependencies**: Common tooling and libraries
4. **Team Coordination**: Need atomic changes across projects
5. **Code Sharing**: Want to share code without publishing

### Don't Use When:

1. **Completely Independent Projects**: No code sharing
2. **Different Tech Stacks**: Can't share tooling
3. **Security Concerns**: Projects with different access levels
4. **Repository Too Large**: Git performance issues (rare)
5. **Different Release Cycles**: Projects need independent versioning

## Interview Questions

### Junior Level

**Q: What is a monorepo?**

A: A monorepo is a single repository that contains multiple projects, applications, or packages. Instead of having separate repositories for each project, everything is stored together with shared tooling and dependencies.

**Q: What are workspaces?**

A: Workspaces are a package manager feature that allows managing multiple packages within a single repository. They enable sharing dependencies and running commands across all packages.

### Mid Level

**Q: What are the main challenges of monorepos?**

A: Key challenges include:
- Build performance (need caching and incremental builds)
- Dependency management (keeping versions consistent)
- CI/CD complexity (affected projects detection)
- Code ownership and permissions
- Repository size over time

**Q: How do you handle versioning in a monorepo?**

A: Strategies include:
- Independent versioning: Each package has its own version
- Fixed versioning: All packages share the same version
- Semantic versioning with Changesets or Lerna
- Version ranges for internal dependencies

```json
// Independent versioning
{
  "name": "@mycompany/ui-components",
  "version": "2.1.0"
}

{
  "name": "@mycompany/utils",
  "version": "1.5.3"
}

// Fixed versioning
{
  "version": "1.0.0"  // All packages at 1.0.0
}
```

### Senior Level

**Q: How do you optimize CI/CD for a monorepo?**

A: Optimization strategies:
- Detect affected projects (only test/build what changed)
- Parallel execution of independent tasks
- Shared build cache across CI runs
- Distributed task execution
- Smart deployment (only deploy changed apps)

```yaml
# GitHub Actions example
name: CI
on: push

jobs:
  affected:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Determine affected projects
        run: |
          npx nx affected:apps --base=origin/main --head=HEAD
      
      - name: Build affected
        run: |
          npx nx affected:build --base=origin/main --head=HEAD --parallel=3
      
      - name: Test affected
        run: |
          npx nx affected:test --base=origin/main --head=HEAD --parallel=3
```

**Q: How do you prevent unwanted dependencies between packages?**

A: Use:
- Lint rules (eslint-plugin-import with restrictions)
- Architecture decision records
- Dependency graph visualization
- Code ownership files
- Automated checks in CI

```json
// .eslintrc.js
{
  "rules": {
    "import/no-restricted-paths": ["error", {
      "zones": [
        {
          "target": "./packages/ui-components",
          "from": "./apps/**",
          "message": "UI components can't depend on apps"
        },
        {
          "target": "./packages/utils",
          "from": "./packages/!(utils)",
          "message": "Utils must be dependency-free"
        }
      ]
    }]
  }
}
```

## Key Takeaways

1. Monorepos store multiple projects in a single repository
2. Enable code sharing without publishing packages
3. Provide consistent tooling and dependencies across projects
4. Allow atomic changes spanning multiple projects
5. Require good tooling (Nx, Turbo) for performance
6. npm/yarn/pnpm workspaces handle package management
7. Build caching is essential for good performance
8. Dependency graph management prevents circular dependencies
9. Best for related projects with shared code
10. Major companies use monorepos at scale successfully

## Resources

### Tools
- Nx: https://nx.dev/ - Powerful monorepo tool
- Turborepo: https://turbo.build/ - High-performance build system
- Lerna: https://lerna.js.org/ - Original monorepo tool
- pnpm: https://pnpm.io/ - Fast, disk-efficient package manager
- Changesets: https://github.com/changesets/changesets - Version management

### Articles
- Monorepos Explained: https://monorepo.tools/
- Google's Monorepo: https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/
- Nx vs Turborepo: https://nx.dev/concepts/turbo-and-nx

### Examples
- Nx Examples: https://github.com/nrwl/nx-examples
- Turborepo Examples: https://github.com/vercel/turbo/tree/main/examples
- Lerna Examples: https://github.com/lerna/lerna/tree/main/commands

### Books
- "Monorepo Tools" by Nrwl (Nx creators)
