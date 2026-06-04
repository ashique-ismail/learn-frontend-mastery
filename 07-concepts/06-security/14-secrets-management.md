# Secrets Management

## Overview

Secrets — API keys, database passwords, OAuth client secrets, signing keys — are among the most sensitive assets in any application. The frontend's relationship with secrets is unique: anything shipped to the browser is inherently public. This guide covers the hard line between what can live client-side versus what must stay server-side, how build-time vs runtime environment variables work, CI/CD secrets management, and how to prevent secrets from leaking through `.env` files, version control, or bundle output.

## The Fundamental Rule: Nothing Secret in Client-Side JS

```
Client-side JavaScript is public code.

Anyone can:
  View source
  Open DevTools → Sources tab
  curl https://yourapp.com/static/main.js | grep -E "key|secret|token"
  Unpack the source map
  Run the bundle through a deobfuscator

This means:
  ❌ API keys for third-party services (Stripe, SendGrid, Twilio)
  ❌ Database connection strings
  ❌ OAuth client secrets
  ❌ Signing keys / JWT secrets
  ❌ Admin service account credentials
  ❌ Internal API URLs that should be private

These MUST live on the server only.
```

## What CAN Go Client-Side

```
Truly public configuration (not secrets):

✅ PUBLIC API keys (designed for client-side use):
   - Google Maps API key (domain-restricted)
   - Firebase config (security enforced by Firebase Security Rules)
   - Stripe publishable key (starts with pk_live_ — public by design)
   - Sentry DSN (public — Sentry designed it this way)
   - Analytics IDs (GA measurement ID, Mixpanel token)
   - Algolia search-only API key

✅ Feature flags (non-sensitive values)
✅ App configuration (feature toggles, region settings)
✅ Build metadata (app version, commit SHA)
```

## Environment Variables at Build Time

Build-time env vars are baked into the JavaScript bundle at compile time — they are NOT secrets.

### Vite

```typescript
// vite.config.ts
// Variables prefixed with VITE_ are exposed to client code
// Variables WITHOUT the prefix are server-only (build scripts only)

// .env
VITE_STRIPE_PUBLISHABLE_KEY=pk_live_...   // ✅ exposed to browser
VITE_ANALYTICS_ID=G-ABC123               // ✅ exposed to browser
VITE_API_BASE_URL=https://api.example.com // ✅ exposed to browser

STRIPE_SECRET_KEY=sk_live_...            // ✅ NOT exposed (no VITE_ prefix)
DATABASE_URL=postgres://...              // ✅ NOT exposed
```

```typescript
// Usage in app code
const stripeKey = import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY;
// This value is a string literal in the production bundle
// It IS visible to anyone who downloads your bundle
// That's fine — Stripe publishable keys are designed to be public

// ❌ Wrong — this secret will be in the bundle
const stripeSecret = import.meta.env.VITE_STRIPE_SECRET_KEY;
// Don't add VITE_ prefix to real secrets
```

### Create React App

```
# .env
REACT_APP_MAPS_API_KEY=AIzaSy...    # ✅ bundled — public
REACT_APP_ANALYTICS_ID=G-XYZ       # ✅ bundled — public

DATABASE_PASSWORD=secret123         # NOT bundled (no REACT_APP_ prefix)
```

### Next.js

```typescript
// next.config.js
module.exports = {
  env: {
    // These are baked in at build time — PUBLIC, visible in browser
    NEXT_PUBLIC_STRIPE_KEY: process.env.NEXT_PUBLIC_STRIPE_KEY,
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  },
};

// Or use NEXT_PUBLIC_ prefix in .env:
// NEXT_PUBLIC_STRIPE_KEY=pk_live_...   → available in browser
// STRIPE_SECRET_KEY=sk_live_...        → server only (API routes, getServerSideProps)
```

```typescript
// Next.js API route — server-only secret used correctly
// pages/api/create-payment.ts
import Stripe from 'stripe';

// process.env.STRIPE_SECRET_KEY — only available server-side, never in browser
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2023-10-16' });

export default async function handler(req, res) {
  const paymentIntent = await stripe.paymentIntents.create({
    amount: req.body.amount,
    currency: 'usd',
  });
  // Return only the client_secret — designed to be shared with the browser
  res.json({ clientSecret: paymentIntent.client_secret });
}
```

## Environment Variables at Runtime

Runtime env vars are loaded when the server starts — they never touch the bundle:

```typescript
// server.ts — Express API server
// Variables read at runtime — not baked into any file
const dbUrl = process.env.DATABASE_URL;          // only in Node.js memory
const jwtSecret = process.env.JWT_SECRET;         // only in Node.js memory
const stripeSecretKey = process.env.STRIPE_SECRET_KEY;

if (!dbUrl || !jwtSecret || !stripeSecretKey) {
  throw new Error('Missing required environment variables');
}

// Validate at startup — fail fast rather than failing at request time
```

```typescript
// Type-safe environment variable validation
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
});

// Throws at startup if any required var is missing or invalid
export const env = envSchema.parse(process.env);
```

## .env Files

`.env` files are for LOCAL development convenience — they are NOT a secrets management system:

```bash
# .env.local — local development only (in .gitignore)
DATABASE_URL=postgres://localhost/myapp_dev
JWT_SECRET=dev-only-not-a-real-secret-abc123
STRIPE_SECRET_KEY=sk_test_...

# .env.example — committed to git (template with no real values)
DATABASE_URL=postgres://localhost/myapp
JWT_SECRET=your-jwt-secret-here
STRIPE_SECRET_KEY=sk_test_your_key_here

# .env.production — NEVER commit this (or ideally, don't have this file)
# Production secrets should come from environment injection, not files
```

### .gitignore for .env Files

```gitignore
# .gitignore — ALWAYS include these
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
.env.production    # if it exists with real values

# OK to commit (no real secrets)
.env.example
.env.test          # if only contains test/mock values
```

## CI/CD Secrets

Never put secrets in YAML files, shell scripts, or source code. Use the CI/CD platform's secrets store:

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build
        env:
          # GitHub Actions secrets — encrypted, not visible in logs
          NEXT_PUBLIC_STRIPE_KEY: ${{ secrets.NEXT_PUBLIC_STRIPE_KEY }}
          NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}
        run: npm run build

      - name: Deploy
        env:
          # Server secrets — only used by deploy script, not bundled
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
        run: npx vercel --token $VERCEL_TOKEN --prod
```

```
GitHub Secrets setup:
  Repository → Settings → Secrets and variables → Actions → New repository secret
  Secrets: encrypted at rest, never shown in logs (masked)
  Accessible as: ${{ secrets.SECRET_NAME }}
```

### Environment-Specific Secrets in CI

```yaml
# Separate secrets per environment
# GitHub: use Environments for deployment protection + scoped secrets
jobs:
  deploy-production:
    environment: production   # Requires approval from codeowners
    steps:
      - name: Deploy
        env:
          # Scoped to 'production' environment — separate from staging secrets
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
          API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
```

## Cloud Secrets Managers

For production, use a dedicated secrets management service:

```typescript
// AWS Secrets Manager
import {
  SecretsManagerClient,
  GetSecretValueCommand,
} from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'us-east-1' });

async function getSecret(secretName: string): Promise<string> {
  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await client.send(command);

  if (response.SecretString) {
    return response.SecretString;
  }
  throw new Error(`Secret ${secretName} not found`);
}

// Load at server startup — cache in memory, refresh before expiry
let cachedDbPassword: string;

async function initDB() {
  cachedDbPassword = await getSecret('prod/myapp/db-password');
  await connectDatabase(process.env.DB_HOST!, cachedDbPassword);
}
```

```typescript
// HashiCorp Vault
import vault from 'node-vault';

const client = vault({
  apiVersion: 'v1',
  endpoint: process.env.VAULT_ADDR!,
  token: process.env.VAULT_TOKEN!,
});

async function getDatabaseCredentials() {
  const result = await client.read('database/creds/my-role');
  return {
    username: result.data.username,
    password: result.data.password,
    // Vault generates dynamic credentials — rotated automatically
  };
}
```

## Secret Scanning

Prevent secrets from ever entering version control:

### Pre-Commit Hooks with git-secrets

```bash
# Install git-secrets
brew install git-secrets  # macOS
# or: pip install detect-secrets

# Add patterns to detect
git secrets --add 'AKIA[0-9A-Z]{16}'       # AWS access key
git secrets --add 'sk_live_[0-9a-zA-Z]{24}' # Stripe secret key
git secrets --add '-----BEGIN RSA PRIVATE KEY-----'

# Scan entire repo history
git secrets --scan-history
```

### GitHub Secret Scanning

GitHub automatically scans for known secret patterns (AWS keys, Stripe keys, GitHub tokens, etc.) and alerts repository owners. Enable in:
`Repository → Settings → Security → Secret scanning`

GitHub will also push-protect — blocking a push that contains known secret patterns.

### Using detect-secrets

```bash
# Install
pip install detect-secrets

# Scan repository
detect-secrets scan > .secrets.baseline

# Audit (review detected secrets — some may be false positives)
detect-secrets audit .secrets.baseline

# Add as pre-commit hook
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

## Preventing Bundle Exposure

Even public keys should be confirmed intentionally public:

```typescript
// webpack.config.js — audit what ends up in the bundle
const { DefinePlugin } = require('webpack');

module.exports = {
  plugins: [
    new DefinePlugin({
      // Only explicitly listed values enter the bundle
      'process.env.REACT_APP_STRIPE_KEY': JSON.stringify(process.env.REACT_APP_STRIPE_KEY),
      'process.env.REACT_APP_API_URL': JSON.stringify(process.env.REACT_APP_API_URL),
      // DO NOT add:
      // 'process.env.STRIPE_SECRET_KEY' ← would expose to browser
    }),
  ],
};
```

```bash
# After build: verify no secrets in bundle
grep -r "sk_live_" dist/        # Check for Stripe secret keys
grep -r "AKIA" dist/            # Check for AWS access keys
grep -r "ghp_" dist/            # Check for GitHub personal access tokens
grep -r "password" dist/ | grep -v "forgot_password\|password_confirmation"
```

## Common Mistakes

### 1. Storing Secrets in Committed .env Files

```bash
# ❌ .env committed to git — visible in git history
git add .env
git commit -m "add config"
# Even if you delete it later, it's in git history forever

# ✅ .env in .gitignore, rotate any exposed keys immediately
# Use: trufflesecurity/trufflehog to scan git history for leaks
```

### 2. Exposing Backend Keys Through a Client-Facing Proxy

```typescript
// ❌ Sending the secret key to the browser so it can call a service directly
// pages/api/get-key.ts
export default function handler(req, res) {
  if (req.session?.user) {
    res.json({ key: process.env.OPENAI_API_KEY }); // ← sends secret to browser!
  }
}

// ✅ Make the API call server-side; return only the result
export default async function handler(req, res) {
  const { OpenAI } = await import('openai');
  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  const completion = await openai.chat.completions.create({
    messages: [{ role: 'user', content: req.body.prompt }],
    model: 'gpt-4o',
  });
  res.json({ content: completion.choices[0].message.content });
}
```

### 3. Logging Secrets

```typescript
// ❌ Logging secrets to console, file, or monitoring
console.log('Connecting with key:', process.env.DATABASE_URL);
// DATABASE_URL often contains the password: postgres://user:password@host/db

// ✅ Log only safe metadata
console.log('Connecting to database:', process.env.DB_HOST);

// ✅ Redact secrets in error handlers
app.use((err: Error, req: Request, res: Response) => {
  const sanitizedError = {
    message: err.message.replace(/sk_[a-z0-9_]+/g, '[REDACTED]'),
    stack: process.env.NODE_ENV === 'development' ? err.stack : undefined,
  };
  res.status(500).json(sanitizedError);
});
```

## Interview Questions

### 1. Why can't you put real secrets in client-side JavaScript, even if obfuscated?

**Answer:** Client-side JavaScript is always downloadable and executable by anyone who visits your site. Obfuscation (minification, variable renaming) is not encryption — any developer can un-minify or run the bundle through a deobfuscator in minutes. Source maps may expose the original code. More importantly, the secret must be used at runtime — it's sent in API requests that can be intercepted, or it's called from `fetch()` calls visible in DevTools Network tab. Even without source access, intercepting a single request reveals the key. The only trustworthy security boundary is the server — code that runs server-side is never sent to the browser.

### 2. What is the difference between build-time and runtime environment variables?

**Answer:** Build-time variables are read by the bundler (Webpack/Vite/etc.) at compile time and replaced with their string values inline in the JavaScript bundle. In the output bundle, `import.meta.env.VITE_API_URL` becomes the literal string `"https://api.example.com"`. This means the value is baked into the file — visible to anyone who downloads it, and changing it requires a rebuild and redeploy. Runtime variables are read by the server process when it starts, via `process.env`. They're never included in any client-facing file. Changing them requires only restarting the server, not rebuilding. Secrets must be runtime variables, loaded by the server only.

### 3. How do you prevent secrets from ending up in git history?

**Answer:** Three layers: (1) `.gitignore` — list `.env`, `.env.local`, and all environment-specific files with real values before writing any secrets; (2) pre-commit hooks — use `git-secrets` or `detect-secrets` to scan staged files for known secret patterns before allowing the commit; (3) GitHub secret scanning — GitHub scans every push for known provider token patterns and alerts you (or blocks the push). If a secret is already in history: rotate it immediately (assume it's compromised), use `git filter-repo` to scrub the history, and force-push — but the history is available to anyone who cloned it before, so rotation is non-negotiable.

### 4. When is it acceptable to put an API key in client-side code?

**Answer:** When the key is explicitly designed for public client-side use and carries no secret authority: Google Maps API keys (restricted to your domain in Google Cloud Console), Stripe publishable keys (`pk_live_` — can only tokenize card data, cannot charge), Firebase config (security enforced by Security Rules, not the key itself), Algolia search-only API keys (read-only, no write access), Sentry DSN (publicly visible by design — only receives error reports), and analytics IDs. The test is: if this key were stolen, could an attacker do anything harmful? For publishable/public keys, the answer is designed to be no. For secret keys (database passwords, Stripe secret key, JWT signing secrets), the answer is yes, and they must stay server-side.
