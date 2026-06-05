# Subresource Integrity (SRI)

## The Idea

**In plain English:** Subresource Integrity (SRI) is a way for your browser to verify that a file loaded from an external server (like a shared library host) hasn't been secretly swapped out for a malicious version. Before running the file, the browser checks its "fingerprint" (called a hash) against one you pre-approved — if they don't match, the file is blocked.

**Real-world analogy:** Imagine you order a sealed jar of peanut butter from a warehouse. Before shipping, the factory stamps a unique seal number on the lid. When the jar arrives, you check that the seal number matches the one the factory told you to expect. If someone at the warehouse replaced the peanut butter with something harmful and re-sealed it, the seal number won't match and you throw it out.

- The seal number stamped by the factory = the hash in the `integrity` attribute you write in your HTML
- The warehouse = the CDN (Content Delivery Network) hosting the file
- Checking the seal number before eating = the browser computing and comparing the hash before running the script

---

## Overview

Subresource Integrity (SRI) is a security feature that lets browsers verify that resources they fetch from third-party servers haven't been tampered with. When you load a JavaScript file from a CDN, SRI gives you a cryptographic guarantee that what the CDN serves matches exactly what you expected — a compromised CDN cannot inject malicious code without breaking the hash check.

## The CDN Supply Chain Risk

```text
Without SRI — CDN compromise impacts your users:

  Your HTML:
  <script src="https://cdn.example.com/jquery-3.7.js"></script>
  └─ No verification — browser trusts whatever CDN serves

  CDN compromise scenario:
  Attacker gains CDN access
    → Modifies jquery-3.7.js to include malware
    → All sites using that CDN now serve malware to users
    → Users trust it because it loads from a "legitimate" source

With SRI — tampered file is REJECTED:
  <script
    src="https://cdn.example.com/jquery-3.7.js"
    integrity="sha384-abc123...">
  </script>
  └─ Browser downloads file
  └─ Computes hash
  └─ Compares to integrity attribute
  └─ MISMATCH → file not executed → users protected
```

## SRI Hash Format

```
integrity="sha384-abc123..."

Format: <algorithm>-<base64-encoded-hash>
  sha256: minimum acceptable
  sha384: recommended (good balance)
  sha512: maximum security, larger value

Multiple hashes (any one match is sufficient):
  integrity="sha384-hash1 sha512-hash2"
```

## Generating SRI Hashes

### Using openssl (command line)

```bash
# Generate SHA-384 hash for a file
cat jquery.min.js | openssl dgst -sha384 -binary | openssl base64 -A
# Output: abc123def456...

# Full integrity attribute value:
echo -n "sha384-$(cat jquery.min.js | openssl dgst -sha384 -binary | openssl base64 -A)"
# sha384-abc123def456...
```

### Using shasum

```bash
# macOS / Linux
shasum -b -a 384 jquery.min.js | awk '{ print $1 }' | xxd -r -p | base64
```

### Using Node.js

```javascript
const crypto = require('crypto');
const fs = require('fs');

function generateSRIHash(filePath, algorithm = 'sha384') {
  const content = fs.readFileSync(filePath);
  const hash = crypto
    .createHash(algorithm)
    .update(content)
    .digest('base64');
  return `${algorithm}-${hash}`;
}

console.log(generateSRIHash('./jquery.min.js'));
// sha384-abc123def456...
```

### Online Tool

```
https://www.srihash.org/ — enter a URL, get the integrity attribute
```

## Using SRI in HTML

### External Script

```html
<!-- jQuery from cdnjs.cloudflare.com -->
<script
  src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.1/jquery.min.js"
  integrity="sha512-v2CJ7UaYy4JwqLDIrZUI/4hqeoQieOmAZNXBeQyjo21dadnwR+8ZaIJVT8EE2iyI61OV8e6M8PP2/4hpQINQ/g=="
  crossorigin="anonymous"
  referrerpolicy="no-referrer">
</script>
```

### External Stylesheet

```html
<!-- Bootstrap CSS from jsDelivr -->
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css"
  integrity="sha384-T3c6CoIi6uLrA9TneNEoa7RxnatzjcDSCmG1MXxSR1GAsXEV/Dwwykc2MPK8M2HN"
  crossorigin="anonymous">
```

### The crossorigin Attribute

`crossorigin="anonymous"` is **required** for SRI to work on cross-origin resources:

```html
<!-- ❌ SRI check will be SKIPPED without crossorigin -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-...">
  <!-- browser ignores integrity without crossorigin on cross-origin resources -->
</script>

<!-- ✅ Required for SRI verification -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-..."
  crossorigin="anonymous">
</script>
```

```
crossorigin values:
  anonymous: request sent without credentials (cookies, HTTP auth)
  use-credentials: request sent WITH credentials (for CDNs that need auth)
```

## What Happens When SRI Fails

```
Browser behavior on SRI mismatch:

1. Browser downloads the resource
2. Computes the hash of the response body
3. Compares to each hash in the integrity attribute
4. At least one match required

If NO match:
  → Script: not executed, network error thrown, console error
  → Stylesheet: not applied
  → Browser logs: "Failed to find a valid digest in the 'integrity' attribute..."

Optional: fallback handling
```

```javascript
// Detect SRI failure for a script
const script = document.querySelector('script[integrity]');
script.addEventListener('error', (e) => {
  // SRI failure causes a load error
  console.error('Script failed to load (possible SRI mismatch):', e);
  // Load a fallback or alert operations
  loadFallbackScript();
});
```

## Webpack SRI Plugin

For bundled apps that need SRI on generated chunks:

```bash
npm install --save-dev webpack-subresource-integrity
```

```javascript
// webpack.config.js
const { SubresourceIntegrityPlugin } = require('webpack-subresource-integrity');

module.exports = {
  output: {
    crossOriginLoading: 'anonymous', // Required for SRI on dynamic imports
  },
  plugins: [
    new SubresourceIntegrityPlugin({
      hashFuncNames: ['sha384'],
      enabled: process.env.NODE_ENV === 'production',
    }),
  ],
};
```

```html
<!-- Generated output with SRI hashes -->
<script
  src="/static/main.a1b2c3.js"
  integrity="sha384-abc123..."
  crossorigin="anonymous">
</script>
```

## Vite SRI Plugin

```bash
npm install --save-dev vite-plugin-subresource-integrity
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { createHtmlPlugin } from 'vite-plugin-html';
import sri from 'vite-plugin-subresource-integrity';

export default defineConfig({
  plugins: [
    sri({
      algorithms: ['sha384'],
    }),
  ],
});
```

## SRI with Content Security Policy

SRI and CSP work together for defense-in-depth:

```
Content-Security-Policy: require-sri-for script style;
  → Browser REQUIRES SRI hashes on all script/style tags
  → Resources without integrity attribute are BLOCKED
  → This prevents accidental inline scripts and non-SRI CDN resources
```

```typescript
// Express: combine CSP + SRI requirement
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    [
      "default-src 'self'",
      "script-src 'self' https://cdnjs.cloudflare.com",
      "style-src 'self' https://cdn.jsdelivr.net",
      "require-sri-for script style", // mandate SRI
    ].join('; ')
  );
  next();
});
```

## Build-Time SRI Generation

For self-hosted assets where you control the CDN:

```typescript
// scripts/generate-sri.ts
import { createHash } from 'crypto';
import { readFileSync, writeFileSync } from 'fs';
import { glob } from 'glob';
import path from 'path';

interface SRIManifest {
  [filename: string]: string; // filename → integrity value
}

async function generateSRIManifest(distDir: string): Promise<SRIManifest> {
  const files = await glob(`${distDir}/**/*.{js,css}`);
  const manifest: SRIManifest = {};

  for (const file of files) {
    const content = readFileSync(file);
    const hash = createHash('sha384').update(content).digest('base64');
    const key = path.relative(distDir, file);
    manifest[key] = `sha384-${hash}`;
  }

  return manifest;
}

async function main() {
  const manifest = await generateSRIManifest('./dist');
  writeFileSync('./dist/sri-manifest.json', JSON.stringify(manifest, null, 2));
  console.log('SRI manifest generated:', Object.keys(manifest).length, 'files');
}

main();
```

```typescript
// Server: read manifest and inject into HTML
import sriManifest from './dist/sri-manifest.json';

app.get('/', (req, res) => {
  const mainScript = 'assets/main.a1b2c3.js';
  const integrity = sriManifest[mainScript];

  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <script
          src="/${mainScript}"
          integrity="${integrity}"
          crossorigin="anonymous">
        </script>
      </head>
      <body><div id="root"></div></body>
    </html>
  `);
});
```

## Automating SRI in CI/CD

```yaml
# .github/workflows/build.yml
- name: Build
  run: npm run build

- name: Generate SRI hashes
  run: node scripts/generate-sri.ts

- name: Verify SRI in HTML output
  run: |
    # Check that all script/link tags in index.html have integrity attributes
    python3 -c "
    import re, sys
    html = open('dist/index.html').read()
    scripts = re.findall(r'<script[^>]+src=[^>]+>', html)
    missing = [s for s in scripts if 'integrity=' not in s and 'cdn' in s]
    if missing:
        print('Missing SRI on external scripts:', missing)
        sys.exit(1)
    print('All external scripts have SRI hashes')
    "
```

## Common Mistakes

### 1. Missing crossorigin Attribute

```html
<!-- ❌ SRI silently skipped — no crossorigin on cross-origin resource -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-...">
</script>

<!-- ✅ Required -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-..."
  crossorigin="anonymous">
</script>
```

### 2. Hash Computed on Wrong File

```bash
# ❌ Computing hash on local dev version, not the CDN version
openssl dgst -sha384 -binary ./node_modules/react/umd/react.production.min.js

# The CDN may serve a slightly different file (whitespace, comments, compression)
# ✅ Download the exact CDN file and hash that
curl -s https://cdn.example.com/react.min.js | openssl dgst -sha384 -binary | openssl base64 -A
```

### 3. Using SRI for Same-Origin Resources Unnecessarily

```html
<!-- SRI on same-origin files: technically works but unnecessary -->
<!-- If attacker controls your origin, they can change the HTML too -->
<!-- SRI is valuable for CROSS-ORIGIN (CDN) resources -->
<script
  src="/static/main.js"
  integrity="sha384-...">   ← overhead with minimal security benefit for same-origin
</script>
```

### 4. Not Updating Hashes on Library Updates

```html
<!-- ❌ Updated the version in the URL but not the hash -->
<script
  src="https://cdn.example.com/lib-v2.js"
  integrity="sha384-hash-for-v1">  ← hash is for v1!
  <!-- SRI check will fail — lib won't load -->
</script>

<!-- ✅ Always regenerate hashes when updating CDN URLs -->
```

## Interview Questions

### 1. What is Subresource Integrity and what attack does it prevent?

**Answer:** SRI lets you specify a cryptographic hash in the `integrity` attribute of `<script>` and `<link>` tags. When the browser fetches the resource, it computes the hash and compares it to the attribute. If they don't match, the resource is rejected. It prevents supply chain attacks via CDN compromise — if an attacker modifies a JavaScript file on a CDN (by gaining access or via a BGP hijack), the modified file won't match the expected hash and won't execute in any browser that fetches it with an SRI attribute. It provides a cryptographic guarantee that the specific file version you tested is the one that runs in production.

### 2. Why is the crossorigin attribute required for SRI to work?

**Answer:** SRI verification requires the browser to read the full response body to compute its hash. Cross-Origin Resource Sharing (CORS) controls whether a cross-origin response body is readable. Without `crossorigin="anonymous"`, the browser fetches the resource in "no-cors" mode where the response body is opaque — the browser cannot read it to compute the hash, so SRI verification is silently skipped. Adding `crossorigin="anonymous"` tells the browser to make a CORS request; if the CDN responds with `Access-Control-Allow-Origin: *` (as all public CDNs do), the body is readable and SRI verification proceeds.

### 3. What are the differences between sha256, sha384, and sha512 for SRI?

**Answer:** All three are SHA-2 family hash functions. SHA-256 produces a 256-bit hash, SHA-384 a 384-bit hash, and SHA-512 a 512-bit hash. Longer hashes have lower collision probability but are larger strings. SHA-384 is the commonly recommended choice — it's been shown to be strong against all known attacks and is the default in most SRI hash generators and CDN documentation. SHA-256 is the minimum acceptable by the W3C spec. You can specify multiple algorithms in one `integrity` attribute (`integrity="sha384-... sha512-..."`); the browser requires at least one match. Including both provides forward compatibility if SHA-384 is ever weakened.

### 4. How would you implement SRI for a build pipeline that uses code splitting?

**Answer:** Use `webpack-subresource-integrity` or `vite-plugin-subresource-integrity`. These plugins compute SRI hashes for all generated chunks at build time and inject `integrity` attributes into the HTML output and into dynamically generated `<script>` tags for lazy-loaded chunks. The plugin also sets `output.crossOriginLoading: 'anonymous'` in Webpack so dynamic imports use CORS mode. For a fully custom setup, generate a manifest of `filename → sha384 hash` after the build, then have your HTML template inject the hashes from the manifest. Verify in CI that every external script and stylesheet tag in the output HTML has an `integrity` attribute.
