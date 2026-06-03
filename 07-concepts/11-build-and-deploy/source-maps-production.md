# Source Maps in Production

## Overview

Source maps translate minified, bundled JavaScript back to original source code, making error stack traces and debugging readable. The production decision is nuanced: source maps are essential for understanding production errors, but if public, they expose your source code to anyone on the internet. This guide covers the security/debuggability tradeoff, the three main strategies (no source maps, hidden/private, public), integration with error monitoring (Sentry), and how to strip source maps from the public bundle while keeping them for internal use.

## What Source Maps Do

```
Without source maps (minified stack trace):
  Error: Cannot read property 'id' of undefined
    at t (https://example.com/assets/main.a1b2c3.js:1:84726)
    at S (https://example.com/assets/main.a1b2c3.js:1:43812)
    at main.a1b2c3.js:1:91234

With source maps (readable stack trace in Sentry):
  Error: Cannot read property 'id' of undefined
    at CartItem (src/components/CartItem.tsx:47:12)
    at CheckoutPage (src/pages/CheckoutPage.tsx:23:5)
    at App (src/App.tsx:15:3)
```

Source maps contain the original source code (or mappings to it), making them a significant IP exposure risk if made public.

## Source Map Types

```
Source map types (webpack/Vite devtool setting):

Fastest (build time):
  eval                — no source maps; fastest rebuild; dev only
  eval-cheap-source-map — line-only; dev only

Medium (balance of speed and quality):
  cheap-source-map    — line-only, no column info
  source-map          — full; separate .map file; SLOW on large apps

Production recommended:
  hidden-source-map   — full quality; no browser reference; safest
  nosources-source-map — maps exist, stack traces work, source code NOT in map

Development recommended:
  eval-source-map     — fast rebuild; full source; dev only
```

## The Three Production Strategies

### Strategy 1: No Source Maps (Simplest, Least Debuggable)

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  devtool: false,   // No source maps at all
};
```

```
Pros: Zero security risk, smaller artifact size
Cons: Stack traces are unreadable in error monitoring
Use when: IP protection outweighs debuggability (heavily proprietary algorithms)
```

### Strategy 2: Hidden Source Maps (Recommended for Most Apps)

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  devtool: 'hidden-source-map',
  // Generates main.js.map but does NOT add //# sourceMappingURL comment to main.js
  // Browsers won't load the source map (no reference in the bundle)
  // But map files exist on your server for upload to Sentry
};
```

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    sourcemap: 'hidden', // same behavior
  },
});
```

```
Generated files:
  dist/assets/main.a1b2c3.js        ← deployed to CDN (no map reference)
  dist/assets/main.a1b2c3.js.map    ← NOT deployed to CDN; uploaded to Sentry

Result:
  - Browsers cannot access source maps (no hint they exist)
  - Sentry has them → readable stack traces in error monitoring
  - Your IP is protected
```

### Strategy 3: Public Source Maps (Most Debuggable)

```javascript
// webpack.config.js
module.exports = {
  devtool: 'source-map',  // Adds //# sourceMappingURL=main.js.map to bundle
};
```

```
Generated files:
  dist/assets/main.js     ← deployed; contains //# sourceMappingURL=main.js.map
  dist/assets/main.js.map ← deployed to CDN; publicly accessible

Result:
  - Anyone can download your source code via DevTools → Sources
  - Great for open-source projects, non-sensitive apps
  - Complete debuggability in any browser
```

### Strategy 4: nosources-source-map (Stack Traces Without Source Code)

```javascript
// webpack.config.js
module.exports = {
  devtool: 'nosources-source-map',
  // Source maps without source code content
  // Stack traces show file names and line numbers
  // But the actual source code is not included in the .map file
};
```

```
Pros: Readable stack traces (file + line), no source code exposure
Cons: Can't click into source in DevTools; Sentry shows file:line but no code context
Use when: You want some debuggability with minimal source exposure
```

## Sentry Integration

Sentry is the most common error monitoring tool and has first-class source map support:

### Upload Source Maps to Sentry

```bash
npm install --save-dev @sentry/cli @sentry/webpack-plugin
```

```javascript
// webpack.config.js — Sentry webpack plugin
const { sentryWebpackPlugin } = require('@sentry/webpack-plugin');

module.exports = {
  devtool: 'hidden-source-map',  // hidden — NOT public

  plugins: [
    sentryWebpackPlugin({
      org: 'my-org',
      project: 'my-project',
      authToken: process.env.SENTRY_AUTH_TOKEN,

      // Upload source maps after build
      sourcemaps: {
        assets: './dist/**/*.map',
        ignore: ['node_modules'],
        deleteFilesAfterUpload: './dist/**/*.map', // Remove from server!
      },

      // Associate with release version
      release: {
        name: process.env.RELEASE_VERSION || 'dev',
        deploy: {
          env: process.env.DEPLOY_ENV || 'production',
        },
      },
    }),
  ],
};
```

```typescript
// vite.config.ts — Sentry vite plugin
import { defineConfig } from 'vite';
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default defineConfig({
  build: {
    sourcemap: 'hidden',
  },
  plugins: [
    sentryVitePlugin({
      org: 'my-org',
      project: 'my-project',
      authToken: process.env.SENTRY_AUTH_TOKEN,
      sourcemaps: {
        assets: ['./dist/**/*.map'],
        ignore: ['node_modules'],
        deleteFilesAfterUpload: ['./dist/**/*.map'],
      },
    }),
  ],
});
```

### Sentry Initialization with Release

```typescript
// src/main.ts — must match the release name used in upload
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  release: import.meta.env.VITE_RELEASE_VERSION,
  // Sentry uses the release version to find the correct source maps
  environment: import.meta.env.MODE,
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration(),
  ],
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
});
```

## CI/CD Pipeline Integration

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    env:
      # Release version includes commit SHA for traceability
      RELEASE_VERSION: ${{ github.ref_name }}-${{ github.sha }}

    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Build with source maps
        env:
          VITE_RELEASE_VERSION: ${{ env.RELEASE_VERSION }}
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
        run: npm run build
        # Build generates:
        # dist/assets/main.a1b2c3.js (hidden — no map reference)
        # dist/assets/main.a1b2c3.js.map (will be deleted after upload)

      - name: Upload source maps to Sentry
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: my-org
          SENTRY_PROJECT: my-project
        run: |
          npx @sentry/cli releases new "${{ env.RELEASE_VERSION }}"
          npx @sentry/cli releases files "${{ env.RELEASE_VERSION }}" \
            upload-sourcemaps ./dist/assets --rewrite
          npx @sentry/cli releases finalize "${{ env.RELEASE_VERSION }}"
          # Delete source maps from dist — they should NOT be deployed
          rm -f dist/assets/*.map

      - name: Verify no source maps in dist
        run: |
          if find dist -name "*.map" | grep -q .; then
            echo "ERROR: Source maps found in dist directory!"
            find dist -name "*.map"
            exit 1
          fi
          echo "No source maps in dist — safe to deploy"

      - name: Deploy to CDN
        run: npm run deploy
        # Only .js/.css/.html files — no .map files
```

## Verifying Source Maps Aren't Exposed

```bash
# After deploy, verify source maps are not publicly accessible
DEPLOY_URL="https://example.com"

# Get the JS bundle URL from your HTML
JS_URL=$(curl -s "$DEPLOY_URL" | grep -oP 'src="[^"]+\.js"' | head -1 | sed 's/src="//; s/"//')

# Check if a source map reference exists in the bundle
if curl -s "$DEPLOY_URL$JS_URL" | grep -q "sourceMappingURL"; then
  echo "WARNING: Source map URL found in bundle!"
else
  echo "OK: No source map reference in bundle"
fi

# Check if .map file is accessible directly
MAP_URL="${DEPLOY_URL}${JS_URL}.map"
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$MAP_URL")
if [ "$STATUS" = "200" ]; then
  echo "SECURITY RISK: Source map publicly accessible at $MAP_URL"
else
  echo "OK: Source map not accessible (HTTP $STATUS)"
fi
```

## Stripping Source Map References

If you use a tool that generates `//# sourceMappingURL=` comments you want to remove:

```javascript
// webpack.config.js — remove sourceMappingURL from output
// When using hidden-source-map, webpack already does this
// But if using a tool that adds it, use string-replace-loader or similar

// Alternative: post-process with sed in CI
// sed -i 's|//# sourceMappingURL=.*||g' dist/assets/*.js
```

## Private Source Map Hosting

An alternative to third-party error monitoring: host source maps on a private, authenticated server:

```typescript
// Express private source map server (internal only)
import express from 'express';
import { verifyInternalToken } from './auth';

const sourceMapServer = express();

// Only accessible with internal service token
sourceMapServer.use(verifyInternalToken);

// Serve maps from secure storage
sourceMapServer.get('/maps/:filename', (req, res) => {
  const { filename } = req.params;
  // Serve from S3 private bucket or local secure storage
  serveFromPrivateStorage(`maps/${filename}`, res);
});

// Browser extension or DevTools can be configured to use this URL
// Or: serve to developers via VPN-restricted URL
```

## Common Mistakes

### 1. Deploying Source Maps to CDN Alongside the Bundle

```bash
# ❌ Deploying everything in dist/
aws s3 sync dist/ s3://my-bucket/
# main.a1b2c3.js AND main.a1b2c3.js.map both uploaded — source code public!

# ✅ Exclude map files from CDN deployment
aws s3 sync dist/ s3://my-bucket/ --exclude "*.map"
```

### 2. Using sourceMappingURL in Production Bundle

```
# ❌ hidden-source-map → deploys .map file anyway
# devtool: 'source-map' creates a public reference + deploys the file

# ✅ Workflow:
# devtool: 'hidden-source-map'
# Upload .map to Sentry
# Delete .map from dist BEFORE deploying
```

### 3. Mismatched Release Versions

```typescript
// ❌ Release version in Sentry SDK doesn't match source map upload release
// Sentry SDK: release: 'v1.0.0'
// Source map upload: release: 'main-abc123'
// Sentry can't find the source maps for this release → unreadable stack traces

// ✅ Use the same value in both
const RELEASE = process.env.VITE_RELEASE_VERSION;
// SDK init: release: RELEASE
// Upload: npx sentry-cli releases new "$RELEASE"
```

### 4. Not Uploading Source Maps Before Errors Occur

```
❌ Upload source maps AFTER deploying
  → Users hit errors BEFORE maps are uploaded
  → Those error reports can't be de-minified
  → Stack traces for early errors are unreadable

✅ Upload source maps BEFORE making deployment public
  CI order: build → upload maps to Sentry → deploy to CDN
  Atomic: Sentry has maps before any user can trigger an error with the new code
```

## Interview Questions

### 1. What is the security risk of public source maps in production?

**Answer:** Source maps contain your original source code — the `.map` file includes the full source as the `sourcesContent` field. Anyone who downloads it can read your intellectual property, discover business logic, find security vulnerabilities (hardcoded values, auth logic flaws), understand your data models, and potentially find exploitable code paths. For most web applications, this is a meaningful risk. Open-source projects have no issue with public maps — their code is already public. For proprietary SaaS or anything with sensitive business logic, use hidden source maps uploaded only to your error monitoring service.

### 2. What is hidden-source-map and how does it differ from no source maps?

**Answer:** `hidden-source-map` generates full `.map` files but omits the `//# sourceMappingURL=` comment from the JavaScript bundle. Browsers only load source maps when they see this comment — without it, they don't know the maps exist and don't fetch them. The maps are generated but not referenced. You upload the map files to Sentry (which receives them via the CLI before deployment), then delete them from the deployment artifact. Result: browsers see minified code, Sentry can de-minify stack traces using the privately-held maps. `false` (no source maps) generates nothing at all — no de-minification possible anywhere.

### 3. How do you ensure Sentry can de-minify stack traces in production?

**Answer:** Three things must align: (1) Build with `hidden-source-map` to generate .map files; (2) Upload .map files to Sentry via CLI or webpack/vite plugin with a specific `release` version identifier before deploying; (3) Initialize the Sentry SDK in your app with the same `release` version string. Sentry matches stack frames from error reports to source maps by release version. If any of the three steps use different version strings, Sentry can't find the right maps. The `deleteFilesAfterUpload` option removes .map files from the dist folder after upload, ensuring they're not accidentally deployed to the CDN.

### 4. What should you verify in CI to ensure source maps aren't accidentally deployed?

**Answer:** After building and before deploying: (1) Assert no `.map` files exist in the deployment artifact directory (`find dist -name "*.map"` should return nothing); (2) After deployment, fetch the deployed JS bundle and verify it contains no `//# sourceMappingURL=` comment; (3) Attempt to fetch `<bundle-url>.map` and assert it returns 403 or 404 — not 200. These three checks together confirm the build pipeline correctly kept source maps private. Add them as CI steps that fail the deployment if violated — this prevents the common mistake of accidentally deploying maps after a bundler configuration change.
