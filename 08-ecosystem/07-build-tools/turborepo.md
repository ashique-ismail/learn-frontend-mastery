# Turborepo

## What It Does

Turborepo is a monorepo build system that speeds up builds and CI by caching task outputs. It understands your workspace dependency graph and runs tasks in the optimal order with maximum parallelism.

---

## The Core Value Proposition

```
Without Turborepo (each CI run):
  Build all 12 packages: 8 minutes

With Turborepo (cache hit on unchanged packages):
  Build 2 changed packages: 45 seconds
  Restore 10 unchanged from cache: instant
```

---

## Setup

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],  // ^ = run deps' build first
      "outputs": ["dist/**", ".next/**"]  // cache these outputs
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**/*.ts", "test/**/*.ts"]  // cache invalidation inputs
    },
    "lint": {
      "outputs": []  // no outputs to cache, but run result is cached
    },
    "dev": {
      "cache": false,  // never cache dev server
      "persistent": true  // long-running process
    }
  }
}
```

---

## How Caching Works

1. Turborepo hashes: inputs (source files), task config, environment variables
2. If hash exists in cache: restore outputs instantly
3. If no cache hit: run the task, store output in cache

```bash
# Local cache: ~/.turbo/
# Remote cache: Vercel or self-hosted

# Enable remote caching
npx turbo login
npx turbo link  # link to Vercel remote cache

# CI: share cache between runs and team members
npx turbo build --cache-dir=.turbo
```

---

## `--filter` for Selective Builds

```bash
# Build only the web app and its dependencies
npx turbo build --filter=web

# Build only packages that changed vs main branch
npx turbo build --filter=[main...HEAD]

# Build only packages affected by changes in ./packages/ui
npx turbo build --filter=...{./packages/ui}...

# Build only packages that depend on @acme/ui
npx turbo build --filter=...@acme/ui
```

---

## Dependency Graph Awareness

```json
// packages/web/package.json
{
  "dependencies": {
    "@acme/ui": "*",    // depends on ui package
    "@acme/utils": "*"
  }
}
```

Turborepo knows `web` depends on `ui` and `utils`. When you run `turbo build`:
1. Build `utils` first (no deps)
2. Build `ui` first (no deps) — parallel with utils
3. Build `web` after both complete

---

## Monorepo Structure

```
my-monorepo/
├── apps/
│   ├── web/          package.json, next.config.ts
│   └── mobile/       package.json, app.json
├── packages/
│   ├── ui/           package.json, src/
│   ├── utils/        package.json, src/
│   └── config/       tsconfig.json, eslint.config.js
├── turbo.json
└── package.json      (workspaces config)
```

```json
// Root package.json
{
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "test": "turbo test",
    "lint": "turbo lint"
  }
}
```

---

## Pruned Docker Builds

```dockerfile
# Turborepo generates a pruned workspace for Docker
FROM node:20 AS builder
COPY . .
RUN npx turbo prune --scope=web --docker  # prune to only web and its deps

FROM node:20 AS installer
COPY --from=builder /out/json/ .
RUN npm install

FROM node:20 AS runner
COPY --from=builder /out/full/ .
COPY --from=installer /node_modules ./node_modules
RUN npx turbo build --filter=web
```

---

## Comparison with Nx

| | Turborepo | Nx |
|---|---|---|
| Configuration | Minimal (`turbo.json`) | More setup (nx.json + project.json) |
| Plugins | None (pure task runner) | Many (generators, executors) |
| Code generation | Not built-in | Built-in (nx generate) |
| Remote cache | Vercel (or self-host) | Nx Cloud or custom |
| Learning curve | Low | Higher |
| Best for | Simple monorepos, speed | Large enterprise monorepos |

---

## Common Interview Questions

**Q: How does Turborepo's cache work with environment variables?**
Add env vars to `globalEnv` or per-task `env` in turbo.json. Changes to these env vars invalidate the cache for affected tasks.

**Q: Can Turborepo replace npm workspaces?**
No. Turborepo sits on top of a package manager's workspace support (npm, pnpm, yarn). Workspaces handle package hoisting and `node_modules`. Turborepo handles task orchestration and caching.

**Q: What's the `^` in `"dependsOn": ["^build"]`?**
`^build` means "run the `build` task for all dependencies of this package first." Without `^`, it means "run the `build` task for this package first" (same package, different task).
