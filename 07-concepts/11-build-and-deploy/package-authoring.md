# Package Authoring and Publishing

## Overview

Writing and publishing npm packages — whether an internal monorepo library, a design system, or an open-source utility — requires understanding the modern `package.json` surface, dual ESM/CJS output, TypeScript declaration files, tree shaking prerequisites, and versioning strategy. Getting it wrong means consumers get bloated bundles, broken TypeScript completions, or silent runtime errors.

---

## `package.json` — The Modern Field Surface

```json
{
  "name": "@acme/ui",
  "version": "1.2.3",
  "description": "ACME Design System components",
  "license": "MIT",
  "author": "ACME Corp",
  "repository": {
    "type": "git",
    "url": "https://github.com/acme/ui.git"
  },

  // Bundler entry points (modern — takes precedence over "main")
  "exports": {
    ".": {
      "import":  "./dist/index.mjs",   // ESM
      "require": "./dist/index.cjs",   // CJS
      "types":   "./dist/index.d.ts"   // TypeScript
    },
    "./button": {
      "import":  "./dist/button.mjs",
      "require": "./dist/button.cjs",
      "types":   "./dist/button.d.ts"
    },
    "./styles.css": "./dist/styles.css"
  },

  // Legacy fallbacks (Node < 12, old bundlers)
  "main":   "./dist/index.cjs",
  "module": "./dist/index.mjs",   // non-standard but respected by Rollup/Vite
  "types":  "./dist/index.d.ts",

  // Mark side-effect-free for aggressive tree shaking
  "sideEffects": false,
  // Or list files with side effects:
  // "sideEffects": ["./dist/styles.css", "./src/register.ts"]

  // Files to include in the published package (allowlist)
  "files": ["dist", "README.md", "CHANGELOG.md"],

  "scripts": {
    "build":    "tsup src/index.ts --format cjs,esm --dts",
    "prepublishOnly": "npm run build && npm test"
  },

  // Runtime dependencies — bundled into consumer app
  "dependencies": {},

  // Development-only — never shipped to consumers
  "devDependencies": {
    "typescript": "^5.0.0",
    "tsup": "^8.0.0",
    "vitest": "^1.0.0"
  },

  // Peer dependencies — consumer must install these themselves
  "peerDependencies": {
    "react": ">=18.0.0",
    "react-dom": ">=18.0.0"
  },
  "peerDependenciesMeta": {
    "react": { "optional": false },
    "react-dom": { "optional": false }
  }
}
```

---

## The `exports` Field

The `exports` field is the modern entry point map. It supersedes `main` and `module` for Node 12+ and modern bundlers.

```json
{
  "exports": {
    // Default export (".") — import('@acme/ui') or require('@acme/ui')
    ".": {
      "import":  "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types":   "./dist/index.d.ts"
    },

    // Subpath exports — import('@acme/ui/button')
    "./button": {
      "import":  "./dist/button/index.mjs",
      "require": "./dist/button/index.cjs",
      "types":   "./dist/button/index.d.ts"
    },

    // Condition: browser vs node (e.g. different fetch implementation)
    "./utils": {
      "browser": "./dist/utils.browser.mjs",
      "node":    "./dist/utils.node.mjs",
      "default": "./dist/utils.mjs"
    },

    // CSS side-effect import
    "./styles": "./dist/styles.css",

    // Prevent importing internal paths
    "./internal/*": null
  }
}
```

**Without `exports`, consumers can import any file in your package** — including internal implementation details. The `exports` field locks that down.

---

## Dual ESM/CJS Build with tsup

`tsup` (built on esbuild) is the fastest way to build a dual-format TypeScript library:

```typescript
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts', 'src/button/index.ts'],
  format: ['cjs', 'esm'],    // output both .cjs and .mjs
  dts: true,                  // generate .d.ts files
  splitting: true,            // code split for tree shaking
  sourcemap: true,
  clean: true,
  treeshake: true,
  external: ['react', 'react-dom'], // don't bundle peer deps

  // Rename output extensions
  outExtension: ({ format }) => ({
    js: format === 'esm' ? '.mjs' : '.cjs',
  }),
});
```

```bash
# Build
npx tsup

# Output:
# dist/index.mjs      ← ESM
# dist/index.cjs      ← CJS
# dist/index.d.ts     ← TypeScript types
# dist/button/index.mjs
# dist/button/index.cjs
# dist/button/index.d.ts
```

---

## TypeScript Configuration for Libraries

```json
// tsconfig.json for a library
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",  // or "node16" for Node-first
    "lib": ["ES2020", "DOM"],

    "declaration": true,            // generate .d.ts
    "declarationMap": true,         // source maps for types (Go to Definition works)
    "emitDeclarationOnly": false,   // tsup handles JS emit

    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,

    // Point to your source for development (monorepo consumers)
    "paths": {
      "@acme/ui": ["./src/index.ts"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

---

## Tree Shaking: Making It Work

For consumers to tree-shake your library:

1. **`"sideEffects": false`** in `package.json` — tells bundlers it's safe to drop unused exports
2. **ES Modules output** — CJS cannot be statically analyzed for tree shaking
3. **Named exports** — `export { Button }` not `export default { Button }`
4. **No barrel re-exports of everything** — avoid `export * from './everything'`

```typescript
// ✓ Named exports — tree-shakeable
export { Button } from './button';
export { Input }  from './input';
export type { ButtonProps } from './button';

// ❌ Re-exporting a namespace — prevents tree shaking of individual items
export * as Components from './components';
```

---

## Semantic Versioning

```
MAJOR.MINOR.PATCH
  │     │     └─ Backward-compatible bug fixes
  │     └─────── Backward-compatible new features
  └───────────── Breaking changes
```

**Pre-release tags:**

```
1.0.0-alpha.1    ← early, API unstable
1.0.0-beta.2     ← feature complete, testing
1.0.0-rc.1       ← release candidate
1.0.0            ← stable
```

**npm dist-tags:**

```bash
# Publish to 'next' tag — doesn't affect 'latest'
npm publish --tag next

# Consumers opt in explicitly
npm install @acme/ui@next

# Promote to latest when ready
npm dist-tag add @acme/ui@2.0.0 latest
```

---

## Changesets for Versioning

`changesets` is the standard tooling for monorepos and libraries:

```bash
npm install -D @changesets/cli
npx changeset init
```

Workflow:
1. When making a change, run `npx changeset` — select `major/minor/patch`, write a summary
2. This creates a `.changeset/unique-name.md` file
3. On release, `npx changeset version` bumps versions and writes `CHANGELOG.md`
4. `npx changeset publish` publishes to npm

```markdown
<!-- .changeset/teal-wolves-fly.md — generated by `npx changeset` -->
---
"@acme/ui": minor
---

Added `size` prop to Button component.
```

---

## Peer Dependencies — When and Why

Use `peerDependencies` for dependencies the consumer controls:

```json
{
  "peerDependencies": {
    "react": ">=17.0.0"
  }
}
```

**Why not `dependencies`?**
- If React is a `dependency`, npm installs it inside your `node_modules/@acme/ui/node_modules/react`
- Your React and the consumer's React are two different instances
- React's Context, hooks, and `useState` break silently across instance boundaries

**Check for duplicate React at build time:**

```javascript
// webpack.config.js — deduplicate React
resolve: {
  alias: {
    react: path.resolve('./node_modules/react'),
  },
}
```

---

## Testing Before Publish

```bash
# See exactly what will be published
npm pack --dry-run

# Locally link for integration testing
npm link
cd ../my-consumer-app && npm link @acme/ui

# Test in a clean environment
npx verdaccio      # local npm registry
npm publish --registry http://localhost:4873
npm install @acme/ui --registry http://localhost:4873
```

---

## Publishing Checklist

- [ ] `exports` field maps ESM, CJS, and types
- [ ] `files` allowlist — only `dist/`, docs, and license
- [ ] `sideEffects` declared
- [ ] Peer dependencies declared (not in `dependencies`)
- [ ] Internal paths blocked via `exports` (no `./src/*`)
- [ ] `prepublishOnly` script runs build + tests
- [ ] CHANGELOG.md updated (via changesets)
- [ ] `.npmignore` or `files` excludes test fixtures, source maps (if desired), and config files

---

## Interview Questions

**Q: What is the `exports` field in `package.json` and why does it matter?**  
A: It's the modern entry point map, taking precedence over `main` and `module`. It maps subpath imports to specific files, lets you provide different builds for ESM vs CJS vs browser vs Node, and locks down internal files — consumers can only import what you explicitly export.

**Q: Why should React be a `peerDependency` in a component library, not a `dependency`?**  
A: If React is a regular dependency, npm installs a separate React instance inside your package. React's hook and context system breaks when two instances exist — hooks called in one instance can't share state with the other. `peerDependency` ensures the consumer's single React instance is used.

**Q: What is `"sideEffects": false` and how does it affect tree shaking?**  
A: It tells bundlers (Webpack, Rollup, Vite) that no module in your package has side effects when imported — importing `button.js` won't register a global, patch a prototype, or do any work unless its exports are used. Without this flag, bundlers are conservative and keep all imported modules even if their exports are unused.
