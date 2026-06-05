# Build System Architecture at Scale

## The Idea

**In plain English:** A build system is the automated process that takes your raw source code and converts it into something browsers can actually run — and at large companies with thousands of files, making that process fast enough so developers are not sitting idle waiting for it is a serious engineering challenge.

**Real-world analogy:** Think of a huge restaurant kitchen preparing hundreds of dishes for a banquet. When one ingredient changes (say, the sauce recipe), a poorly run kitchen re-cooks every single dish from scratch. A smart kitchen keeps pre-cooked components cached, only re-heats the dishes that actually used that sauce, and has multiple chefs working in parallel.

- The restaurant kitchen = the build system
- Each dish = a compiled output file or package
- Re-cooking only affected dishes = incremental/cached rebuilding
- Multiple chefs working in parallel = concurrent package builds

---

## Overview

"Our builds take 8 minutes. Developers are blocked waiting for CI. Hot module replacement takes 15 seconds. What do you do?" This is a real staff-level problem at companies with large codebases. Solving it requires understanding what builds actually do, what makes them slow, and how modern tools (Vite, Turbopack, Nx, Turborepo) address each bottleneck.

---

## What Happens in a Build

```
Source files
  ↓ Transform      (TypeScript → JS, JSX → JS, Sass → CSS)
  ↓ Bundle         (resolve imports, concatenate modules)
  ↓ Optimize       (tree shake, minify, split chunks)
  ↓ Hash           (fingerprint output filenames)
  ↓ Write          (output to disk)

Dev server adds:
  ↓ Watch          (detect file changes)
  ↓ Invalidate     (determine what needs to rebuild)
  ↓ HMR            (push changes to browser without full reload)
```

Each step has a different cost profile. Optimization is the most expensive. Transformation scales with file count. Bundling scales with dependency graph depth.

---

## Why Webpack Builds Get Slow

Webpack bundles everything into a single module graph. Every file change:

1. Finds all files that depend on the changed file (graph traversal)
2. Re-transforms every affected file (TypeScript/JSX → JS)
3. Re-bundles the affected module subtree
4. Re-runs optimization on the output
5. Writes the new bundle to disk

At 10,000 files, a single file change triggers re-processing of potentially thousands of modules. Even with caching, this takes 5–30 seconds.

**The core problem:** Webpack was designed for production bundles. It treats development as "just a build that outputs to memory." This is the wrong mental model for dev servers.

---

## Vite's Approach: ESM Dev Server

Vite doesn't bundle in development at all. It serves native ES Modules directly:

```
Browser requests /app.tsx
  → Vite transforms app.tsx (TS → JS) on demand
  → Browser parses and requests dependencies
  → Vite transforms each on demand
  → No bundle, no graph, no optimization
```

**What this means:**
- Server starts in <500ms regardless of project size (nothing to bundle)
- File changes: only the changed file is re-transformed (~20ms)
- HMR: only changed module is replaced (not the whole chunk)

**The trade-off:** In development, you make hundreds of HTTP requests (one per module). Modern browsers handle this fine locally, but it means dev and prod environments differ significantly.

```
Dev:  Browser requests 2000 individual JS files (fast on localhost)
Prod: Browser gets 3 optimized bundles (fast over the network)

This difference can hide bugs: code that works with ESM native modules
may behave differently in a bundled Webpack output.
```

---

## Build Caching: What Gets Cached and Where

### Filesystem Cache (Webpack 5, Rollup, esbuild)

```javascript
// webpack.config.js
module.exports = {
  cache: {
    type: 'filesystem',             // persist between builds
    cacheDirectory: '.webpack-cache',
    buildDependencies: {
      config: [__filename],         // invalidate when webpack config changes
    },
  },
};
```

With filesystem caching, a cold build takes 45s. A warm build (cache hit) takes 8s. The cache stores the compiled output of each module, keyed by content hash. If a file hasn't changed, its cached compilation is reused.

**Cache invalidation rules:**
- File content changes → cache miss for that file and its dependents
- Dependency version changes (package.json) → full cache invalidation
- Webpack config changes → full cache invalidation

### Remote Cache (Nx, Turborepo)

Team-level caching — if a coworker already ran a build for a given commit, you get their output without running the build yourself:

```yaml
# turbo.json
{
  "pipeline": {
    "build": {
      "outputs": ["dist/**"],
      "cache": true              // ← cache this task's output
    },
    "test": {
      "cache": true,
      "inputs": ["src/**", "tests/**"] // ← only invalidate on these files
    }
  },
  "remoteCache": {
    "signature": true            // ← verify cache integrity
  }
}
```

```bash
# CI output with remote cache hit:
# ✓ build (cached) → 0ms
# ✓ test  (cached) → 0ms
# ✓ lint  (run)    → 45s   ← only runs because lint config changed
```

**How it works:** Every task's output is stored in a content-addressable store (local disk or remote S3/R2), keyed by the hash of: source files + task inputs + dependency versions + tool version. Same inputs = same hash = cache hit = skip execution.

---

## Incremental Compilation: TypeScript Project References

For large TypeScript monorepos, `tsc` type-checks the entire codebase by default. With 1000 files this takes 60+ seconds.

**Project References** split the codebase into independently compilable sub-projects:

```json
// packages/ui/tsconfig.json
{
  "compilerOptions": {
    "composite": true,          // enables incremental build artifacts
    "declarationMap": true,
    "outDir": "./dist"
  },
  "include": ["src/**/*"]
}

// packages/app/tsconfig.json
{
  "references": [
    { "path": "../ui" }         // depends on ui package
  ]
}
```

```bash
# First build: compiles everything
tsc --build packages/app

# Second build: only recompiles changed projects
tsc --build packages/app --incremental
# ✓ packages/ui: up to date (skipped)
# ✓ packages/app: rebuilt (1 file changed)
```

**Build time reduction:** 60s → 8s for a single file change in a large project.

---

## Turbopack vs Vite: When to Choose Which

| | Vite | Turbopack |
|---|---|---|
| **Approach** | ESM native (no bundling in dev) | Incremental bundler (Rust) |
| **Cold start** | <500ms | ~1s |
| **HMR** | ~50ms | ~20ms |
| **Framework** | Framework-agnostic | Optimized for Next.js |
| **Plugin ecosystem** | Mature (Rollup-compatible) | Growing |
| **Large projects** | Slows as module count grows (many HTTP requests) | Better at scale (bundle-aware) |
| **Use for** | New projects, Vue, non-Next React | Next.js 15+ |

---

## Diagnosing a Slow Build

### Step 1: Profile the Build

```bash
# Webpack: built-in profiling
WEBPACK_BUILD_PROFILE=true npx webpack --profile --json > build-stats.json
npx webpack-bundle-analyzer build-stats.json

# Vite: measure plugin time
# vite.config.ts
export default defineConfig({
  plugins: [
    {
      name: 'build-timer',
      buildStart() { console.time('build'); },
      buildEnd() { console.timeEnd('build'); },
    },
  ],
});

# TypeScript: trace compilation
tsc --generateTrace tracing-output --incremental
```

### Step 2: Identify the Bottleneck

```
Slow cold start:
  → Too many files in initial bundle (check entry points)
  → TypeScript type checking blocking the bundle step
  → Large dependencies being transformed (check swc/esbuild usage)

Slow HMR:
  → Circular dependencies (A imports B imports A — large invalidation chain)
  → Top-level side effects in modules (import causes execution, re-execution on change)
  → Source maps for large files (disable in development if not debugging)

Slow CI builds:
  → No remote caching
  → All packages building even if unchanged
  → TypeScript full rebuild (enable project references)
```

### Step 3: Fix Priority Order

```
1. Enable filesystem caching (free, usually 50–80% improvement)
2. Enable remote caching (free for small teams with Turborepo)
3. Split TypeScript into project references (for TS-heavy projects)
4. Move from tsc to swc/esbuild for transforms (5–10x faster, no type checking)
5. Audit and eliminate large dependencies (moment.js, full lodash)
6. Enable parallelism (build packages concurrently, not serially)
```

---

## Build Pipeline Design for Monorepos

```
             ┌─────────────────────────────────────────┐
             │ CI Pipeline                              │
             │                                         │
Commit ──→  │ Affected detection                       │
             │  (git diff vs base branch)              │
             │         ↓                               │
             │ Only build/test changed packages        │
             │  packages/ui    → affected → build+test │
             │  packages/auth  → not affected → skip   │
             │  packages/app   → affected → build+test │
             │         ↓                               │
             │ Remote cache lookup                     │
             │  ui build: cache hit → skip             │
             │  app build: cache miss → run            │
             │         ↓                               │
             │ Deploy affected packages only           │
             └─────────────────────────────────────────┘

Total CI time: 4 minutes (vs 18 minutes without affected detection + caching)
```

---

## Interview Questions

**Q: "Our Webpack builds take 8 minutes. Walk me through your approach."**
A: I'd start by profiling — which phase takes the longest and which packages are responsible. Then address in order: (1) Enable filesystem cache if not already (free, 50–80% reduction). (2) Enable remote cache with Turborepo/Nx — if CI already ran this commit, skip entirely. (3) Add affected-build detection so unchanged packages don't rebuild. (4) If TypeScript is the bottleneck, enable project references for incremental type checking, or move transpilation to swc/esbuild (5–10x faster, moves type checking to a separate step). (5) If it's still slow: profile individual packages for expensive transforms or circular dependencies.

**Q: "Why is Vite faster than Webpack in development?"**
A: Vite doesn't bundle in development — it serves source files as native ES modules directly from the file system, transforming them on demand as the browser requests them. Webpack has to build the entire module graph upfront. Vite's server starts instantly regardless of project size; Webpack's startup time scales with file count. For HMR, Vite only re-transforms the changed file; Webpack must re-process the module and potentially its dependents.

**Q: "What is remote caching in a build system and why does it matter at scale?"**
A: Remote caching stores build and test outputs (indexed by a hash of inputs: source files + config + dependency versions) in a shared store (S3, Cloudflare R2, or a managed service). When any team member — or CI — runs a task whose inputs haven't changed, they download the cached output instead of rerunning. In a team of 50 engineers, most PRs touch only a subset of packages. With remote cache, CI completes in 2–3 minutes for a typical PR instead of 18 minutes, because most packages are already built and cached from previous runs.
