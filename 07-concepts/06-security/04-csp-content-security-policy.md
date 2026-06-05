# CSP (Content Security Policy)

## The Idea

**In plain English:** CSP is a set of rules you send to a web browser telling it exactly which places on the internet it is allowed to load scripts, images, and other content from for your website. If something tries to sneak in from a location that is not on the approved list, the browser ignores it completely.

**Real-world analogy:** Imagine a school that only lets students eat food brought from an approved list of caterers. A lunch monitor checks every tray at the cafeteria door and turns away anything that did not come from an approved supplier — even if it looks perfectly fine.
- The approved caterer list = the CSP header (the list of trusted sources)
- The lunch monitor = the browser enforcing the policy
- An unknown supplier trying to sneak food in = an attacker injecting a malicious script
- A student's tray being turned away = the browser blocking the unauthorized resource

---

## Table of Contents
- [Introduction](#introduction)
- [How CSP Works](#how-csp-works)
- [CSP Directives](#csp-directives)
- [Nonce-Based CSP](#nonce-based-csp)
- [Hash-Based CSP](#hash-based-csp)
- [CSP Reporting](#csp-reporting)
- [Strict CSP](#strict-csp)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Content Security Policy (CSP)** is an HTTP security header that helps prevent Cross-Site Scripting (XSS), clickjacking, and other code injection attacks by specifying which sources of content browsers should allow to load on a webpage.

### Why CSP is Critical

```
┌─────────────────────────────────────────────────────────────┐
│              CSP Protection Layers                           │
└─────────────────────────────────────────────────────────────┘

Without CSP:
   Attacker injects: <script src="http://evil.com/steal.js"></script>
   Browser: ✓ Executes malicious script
   Result: XSS attack succeeds

With CSP:
   CSP Header: script-src 'self' https://trusted-cdn.com
   Attacker injects: <script src="http://evil.com/steal.js"></script>
   Browser: ✗ Blocks malicious script
   Result: XSS attack prevented

┌──────────────────────────────────────────────────────────────┐
│  Attack Type            │  CSP Protection                     │
├──────────────────────────────────────────────────────────────┤
│  Injected <script> tags │  script-src directive               │
│  Inline event handlers  │  Requires 'unsafe-inline' (bad!)    │
│  eval() and setTimeout  │  Blocked by default                 │
│  External stylesheets   │  style-src directive                │
│  Clickjacking           │  frame-ancestors directive          │
│  Form hijacking         │  form-action directive              │
│  Mixed content          │  upgrade-insecure-requests          │
└──────────────────────────────────────────────────────────────┘
```

## How CSP Works

CSP is delivered via HTTP header or meta tag, instructing browsers which resources are safe to load.

### Basic CSP Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    CSP Request Flow                          │
└─────────────────────────────────────────────────────────────┘

1. Browser requests page from server
   ┌─────────┐                     ┌─────────┐
   │ Browser │ ──── GET /page ───> │ Server  │
   └─────────┘                     └─────────┘

2. Server responds with CSP header
   ┌─────────┐                     ┌─────────┐
   │ Browser │ <──── CSP Header ── │ Server  │
   └─────────┘     HTML Content    └─────────┘
            
            Content-Security-Policy: 
              script-src 'self' https://cdn.example.com

3. Browser parses HTML and checks resources against CSP
   ┌─────────────────────────────────────────┐
   │ <script src="/app.js">        ✓ Allowed │
   │   (matches 'self')                       │
   │                                          │
   │ <script src="https://cdn.example.com    │
   │         /library.js">          ✓ Allowed │
   │   (matches allowlist)                    │
   │                                          │
   │ <script src="http://evil.com/           │
   │         malware.js">           ✗ Blocked │
   │   (not in allowlist)                     │
   └─────────────────────────────────────────┘

4. Browser reports violations (if report-uri configured)
   ┌─────────┐                     ┌──────────────┐
   │ Browser │ ── Violation Report │ Report       │
   │         │ ──────────────────> │ Endpoint     │
   └─────────┘                     └──────────────┘
```

### CSP Delivery Methods

```typescript
// Method 1: HTTP Header (Recommended)
// Backend (Express.js)
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' https://trusted-cdn.com; style-src 'self' 'unsafe-inline'"
  );
  next();
});

// Method 2: Meta Tag (Limited functionality - no report-uri, frame-ancestors)
// HTML
const HTMLWithCSP: React.FC = () => {
  return (
    <html>
      <head>
        <meta
          httpEquiv="Content-Security-Policy"
          content="default-src 'self'; script-src 'self' https://trusted-cdn.com"
        />
      </head>
      <body>
        <App />
      </body>
    </html>
  );
};

// Method 3: Report-Only Mode (Testing)
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy-Report-Only',
    "default-src 'self'; report-uri /csp-violation-report"
  );
  next();
});
```

## CSP Directives

### Core Directives

```typescript
// Comprehensive CSP configuration
interface CSPConfig {
  // Fallback for all fetch directives
  'default-src'?: string[];
  
  // Script sources
  'script-src'?: string[];
  'script-src-elem'?: string[];  // <script> elements
  'script-src-attr'?: string[];  // Inline event handlers
  
  // Style sources
  'style-src'?: string[];
  'style-src-elem'?: string[];   // <style> and <link rel="stylesheet">
  'style-src-attr'?: string[];   // Inline style attributes
  
  // Image sources
  'img-src'?: string[];
  
  // Font sources
  'font-src'?: string[];
  
  // Media sources
  'media-src'?: string[];
  
  // Object sources (plugins)
  'object-src'?: string[];
  
  // Frame sources
  'frame-src'?: string[];
  'frame-ancestors'?: string[];  // Who can embed this page
  
  // Connection sources
  'connect-src'?: string[];      // fetch, XHR, WebSocket, EventSource
  
  // Worker sources
  'worker-src'?: string[];
  
  // Manifest sources
  'manifest-src'?: string[];
  
  // Form action targets
  'form-action'?: string[];
  
  // Base URI restriction
  'base-uri'?: string[];
  
  // Sandbox restrictions
  'sandbox'?: string[];
  
  // Upgrade insecure requests
  'upgrade-insecure-requests'?: boolean;
  
  // Reporting
  'report-uri'?: string;
  'report-to'?: string;
}

// Example: Strict CSP configuration
const strictCSP: CSPConfig = {
  'default-src': ["'self'"],
  'script-src': ["'self'", "'strict-dynamic'"],
  'style-src': ["'self'", "'unsafe-inline'"],
  'img-src': ["'self'", 'data:', 'https:'],
  'font-src': ["'self'", 'data:'],
  'connect-src': ["'self'", 'https://api.example.com'],
  'frame-ancestors': ["'none'"],
  'base-uri': ["'self'"],
  'form-action': ["'self'"],
  'object-src': ["'none'"],
  'upgrade-insecure-requests': true,
  'report-uri': '/csp-violation-report'
};
```

### Directive Values

```typescript
// CSP source values
const cspSourceValues = {
  // Keywords (must be quoted)
  "'none'": "Blocks all sources",
  "'self'": "Same origin (scheme, host, port)",
  "'unsafe-inline'": "Allows inline scripts/styles (AVOID!)",
  "'unsafe-eval'": "Allows eval() and similar (AVOID!)",
  "'strict-dynamic'": "Trust scripts loaded by trusted scripts",
  "'unsafe-hashes'": "Allow inline event handlers with hash",
  
  // Schemes
  "data:": "Data URIs (data:image/png;base64...)",
  "https:": "Any HTTPS source",
  "http:": "Any HTTP source (avoid)",
  "blob:": "Blob URLs",
  "filesystem:": "Filesystem URLs",
  
  // Hosts
  "example.com": "Exact host",
  "*.example.com": "Any subdomain of example.com",
  "https://example.com": "HTTPS only from example.com",
  "example.com:443": "Specific port",
  
  // Nonces
  "'nonce-{base64-value}'": "Specific inline script/style with matching nonce",
  
  // Hashes
  "'sha256-{base64-value}'": "Inline script/style with matching hash",
  "'sha384-{base64-value}'": "SHA-384 hash",
  "'sha512-{base64-value}'": "SHA-512 hash"
};
```

### Common CSP Configurations

```typescript
// Configuration 1: Basic Security
const basicCSP = 
  "default-src 'self'; " +
  "script-src 'self'; " +
  "style-src 'self'; " +
  "img-src 'self' data:; " +
  "font-src 'self'; " +
  "connect-src 'self'";

// Configuration 2: With CDN Support
const cdnCSP = 
  "default-src 'self'; " +
  "script-src 'self' https://cdn.jsdelivr.net https://unpkg.com; " +
  "style-src 'self' https://cdn.jsdelivr.net 'unsafe-inline'; " +
  "img-src 'self' data: https:; " +
  "font-src 'self' data: https://fonts.gstatic.com; " +
  "connect-src 'self' https://api.example.com";

// Configuration 3: SPA with API
const spaCSP = 
  "default-src 'self'; " +
  "script-src 'self' 'strict-dynamic'; " +
  "style-src 'self' 'unsafe-inline'; " +
  "img-src 'self' data: blob: https:; " +
  "font-src 'self' data:; " +
  "connect-src 'self' https://api.example.com wss://socket.example.com; " +
  "frame-ancestors 'none'; " +
  "base-uri 'self'; " +
  "form-action 'self'";

// Configuration 4: Strict CSP with Nonces
const nonceCSP = (nonce: string) => 
  "default-src 'self'; " +
  `script-src 'nonce-${nonce}' 'strict-dynamic'; ` +
  "style-src 'self' 'unsafe-inline'; " +
  "object-src 'none'; " +
  "base-uri 'none'; " +
  "report-uri /csp-violation";
```

## Nonce-Based CSP

Nonces (Number used ONCE) are random values that allow specific inline scripts/styles.

### Backend: Generating Nonces

```typescript
// Express middleware for nonce generation
import crypto from 'crypto';
import express from 'express';

interface RequestWithNonce extends express.Request {
  nonce?: string;
}

const generateNonce = (): string => {
  return crypto.randomBytes(16).toString('base64');
};

const nonceMiddleware = (
  req: RequestWithNonce, 
  res: express.Response, 
  next: express.NextFunction
) => {
  const nonce = generateNonce();
  req.nonce = nonce;
  
  // Set CSP header with nonce
  res.setHeader(
    'Content-Security-Policy',
    `default-src 'self'; script-src 'nonce-${nonce}' 'strict-dynamic'; object-src 'none'; base-uri 'none'`
  );
  
  next();
};

app.use(nonceMiddleware);

// Route handler
app.get('/', (req: RequestWithNonce, res) => {
  const nonce = req.nonce;
  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <title>CSP with Nonce</title>
      </head>
      <body>
        <h1>Secure Page</h1>
        
        <!-- Allowed: Has matching nonce -->
        <script nonce="${nonce}">
          console.log('This script is allowed');
        </script>
        
        <!-- Blocked: No nonce -->
        <script>
          console.log('This script is blocked');
        </script>
        
        <!-- Allowed: External script loaded by trusted script -->
        <script nonce="${nonce}">
          const script = document.createElement('script');
          script.src = '/external.js';
          document.head.appendChild(script);
        </script>
      </body>
    </html>
  `);
});
```

### React: Server-Side Rendering with Nonces

```typescript
// React SSR with nonce
import express from 'express';
import React from 'react';
import { renderToString } from 'react-dom/server';
import crypto from 'crypto';

// Context for nonce
const NonceContext = React.createContext<string>('');

export const useNonce = () => React.useContext(NonceContext);

// Component using nonce
const InlineScript: React.FC<{ children: string }> = ({ children }) => {
  const nonce = useNonce();
  
  return (
    <script
      nonce={nonce}
      dangerouslySetInnerHTML={{ __html: children }}
    />
  );
};

// App component
const App: React.FC = () => {
  return (
    <div>
      <h1>My App</h1>
      <InlineScript>
        {`
          window.__INITIAL_STATE__ = ${JSON.stringify({ user: 'john' })};
        `}
      </InlineScript>
    </div>
  );
};

// SSR handler
app.get('/', (req, res) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  
  // Set CSP with nonce
  res.setHeader(
    'Content-Security-Policy',
    `default-src 'self'; script-src 'nonce-${nonce}' 'strict-dynamic'; style-src 'self' 'unsafe-inline'`
  );
  
  const html = renderToString(
    <NonceContext.Provider value={nonce}>
      <App />
    </NonceContext.Provider>
  );
  
  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <title>React SSR with CSP</title>
        <script nonce="${nonce}" src="/client.js"></script>
      </head>
      <body>
        <div id="root">${html}</div>
      </body>
    </html>
  `);
});
```

### Angular: Nonce Implementation

```typescript
// Angular: CSP Nonce service
import { Injectable, Inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({
  providedIn: 'root'
})
export class NonceService {
  private nonce: string | null = null;

  constructor(@Inject(PLATFORM_ID) private platformId: object) {
    if (isPlatformBrowser(this.platformId)) {
      this.extractNonce();
    }
  }

  private extractNonce(): void {
    // Extract nonce from script tag
    const scripts = document.getElementsByTagName('script');
    for (let i = 0; i < scripts.length; i++) {
      const nonceAttr = scripts[i].getAttribute('nonce');
      if (nonceAttr) {
        this.nonce = nonceAttr;
        break;
      }
    }
  }

  getNonce(): string | null {
    return this.nonce;
  }

  // Create script element with nonce
  createScript(src: string): HTMLScriptElement {
    const script = document.createElement('script');
    script.src = src;
    if (this.nonce) {
      script.setAttribute('nonce', this.nonce);
    }
    return script;
  }

  // Execute inline script with nonce
  executeInlineScript(code: string): void {
    const script = document.createElement('script');
    script.textContent = code;
    if (this.nonce) {
      script.setAttribute('nonce', this.nonce);
    }
    document.head.appendChild(script);
  }
}

// Usage in component
@Component({
  selector: 'app-dynamic-script',
  template: '<div>Loading external script...</div>'
})
export class DynamicScriptComponent implements OnInit {
  constructor(private nonceService: NonceService) {}

  ngOnInit(): void {
    // Load external script with nonce
    const script = this.nonceService.createScript(
      'https://cdn.example.com/library.js'
    );
    document.head.appendChild(script);

    // Execute inline script with nonce
    this.nonceService.executeInlineScript(`
      console.log('Script executed with nonce');
    `);
  }
}
```

## Hash-Based CSP

Hashes allow specific inline scripts/styles based on their content.

```typescript
// Generate hash for inline script
import crypto from 'crypto';

const generateScriptHash = (scriptContent: string): string => {
  const hash = crypto
    .createHash('sha256')
    .update(scriptContent)
    .digest('base64');
  return `sha256-${hash}`;
};

// Example usage
const inlineScript = `console.log('Hello World');`;
const hash = generateScriptHash(inlineScript);

// CSP header
const cspWithHash = `script-src 'self' '${hash}'`;

// HTML
const html = `
  <script>${inlineScript}</script>
`;

// Build-time hash generation for React
const PrecomputedHashes: React.FC = () => {
  const script1 = `window.__CONFIG__ = ${JSON.stringify({ apiUrl: '/api' })};`;
  const script2 = `console.log('App initialized');`;
  
  const hash1 = generateScriptHash(script1);
  const hash2 = generateScriptHash(script2);
  
  // These hashes would be added to CSP header
  console.log('CSP hashes:', hash1, hash2);
  
  return (
    <>
      <script dangerouslySetInnerHTML={{ __html: script1 }} />
      <script dangerouslySetInnerHTML={{ __html: script2 }} />
    </>
  );
};
```

### Webpack Plugin for Hash Generation

```typescript
// Webpack plugin to generate CSP hashes
class CSPHashPlugin {
  private hashes: Set<string> = new Set();

  apply(compiler: any) {
    compiler.hooks.compilation.tap('CSPHashPlugin', (compilation: any) => {
      compilation.hooks.processAssets.tap(
        {
          name: 'CSPHashPlugin',
          stage: compiler.webpack.Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE
        },
        (assets: any) => {
          Object.keys(assets).forEach((filename) => {
            if (filename.endsWith('.html')) {
              const source = assets[filename].source();
              
              // Extract inline scripts
              const scriptRegex = /<script>([\s\S]*?)<\/script>/g;
              let match;
              
              while ((match = scriptRegex.exec(source)) !== null) {
                const scriptContent = match[1];
                const hash = crypto
                  .createHash('sha256')
                  .update(scriptContent)
                  .digest('base64');
                this.hashes.add(`'sha256-${hash}'`);
              }
            }
          });

          // Generate CSP file with hashes
          const csp = `script-src 'self' ${Array.from(this.hashes).join(' ')}`;
          compilation.emitAsset(
            'csp-hashes.txt',
            new compiler.webpack.sources.RawSource(csp)
          );
        }
      );
    });
  }
}

// Webpack config
module.exports = {
  plugins: [
    new CSPHashPlugin()
  ]
};
```

## CSP Reporting

CSP violations can be reported to a specified endpoint for monitoring.

### Backend: Violation Report Handler

```typescript
// Express endpoint for CSP violation reports
import express from 'express';

interface CSPViolation {
  'document-uri': string;
  'referrer': string;
  'violated-directive': string;
  'effective-directive': string;
  'original-policy': string;
  'blocked-uri': string;
  'status-code': number;
  'source-file'?: string;
  'line-number'?: number;
  'column-number'?: number;
}

interface CSPReport {
  'csp-report': CSPViolation;
}

app.post('/csp-violation-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  const report: CSPReport = req.body;
  const violation = report['csp-report'];

  // Log violation
  console.error('CSP Violation:', {
    documentUri: violation['document-uri'],
    violatedDirective: violation['violated-directive'],
    blockedUri: violation['blocked-uri'],
    sourceFile: violation['source-file'],
    lineNumber: violation['line-number']
  });

  // Store in database
  storeViolation(violation);

  // Alert if critical
  if (isCriticalViolation(violation)) {
    alertSecurityTeam(violation);
  }

  res.status(204).send();
});

const storeViolation = (violation: CSPViolation) => {
  // Store in database for analysis
  db.cspViolations.insert({
    timestamp: new Date(),
    documentUri: violation['document-uri'],
    violatedDirective: violation['violated-directive'],
    blockedUri: violation['blocked-uri'],
    userAgent: violation['user-agent']
  });
};

const isCriticalViolation = (violation: CSPViolation): boolean => {
  // Flag injected scripts
  const blockedUri = violation['blocked-uri'];
  const suspiciousPatterns = [
    'eval',
    'javascript:',
    'data:text/html',
    'about:blank'
  ];
  
  return suspiciousPatterns.some(pattern => 
    blockedUri.includes(pattern)
  );
};
```

### Modern Reporting API

```typescript
// Using Reporting API (newer standard)
app.use((req, res, next) => {
  // Configure Report-To header
  res.setHeader('Report-To', JSON.stringify({
    group: 'csp-endpoint',
    max_age: 10886400,
    endpoints: [
      { url: 'https://example.com/csp-reports' }
    ]
  }));

  // CSP with report-to
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; report-to csp-endpoint"
  );

  next();
});

// React: Client-side reporting monitoring
const CSPMonitor: React.FC = () => {
  useEffect(() => {
    // Listen for security policy violations
    const handleViolation = (event: SecurityPolicyViolationEvent) => {
      console.warn('CSP Violation:', {
        violatedDirective: event.violatedDirective,
        blockedURI: event.blockedURI,
        sourceFile: event.sourceFile,
        lineNumber: event.lineNumber
      });

      // Send to analytics
      analytics.track('csp_violation', {
        directive: event.violatedDirective,
        blocked: event.blockedURI
      });
    };

    document.addEventListener('securitypolicyviolation', handleViolation);

    return () => {
      document.removeEventListener('securitypolicyviolation', handleViolation);
    };
  }, []);

  return <div>CSP Monitor Active</div>;
};
```

### Angular: CSP Violation Dashboard

```typescript
// Angular service for CSP monitoring
@Injectable({
  providedIn: 'root'
})
export class CspMonitoringService {
  private violations$ = new Subject<SecurityPolicyViolationEvent>();

  constructor(private http: HttpClient) {
    this.initListener();
  }

  private initListener(): void {
    if (typeof document !== 'undefined') {
      document.addEventListener('securitypolicyviolation', (event) => {
        this.violations$.next(event);
        this.reportViolation(event);
      });
    }
  }

  getViolations(): Observable<SecurityPolicyViolationEvent> {
    return this.violations$.asObservable();
  }

  private reportViolation(event: SecurityPolicyViolationEvent): void {
    const report = {
      documentUri: event.documentURI,
      violatedDirective: event.violatedDirective,
      blockedUri: event.blockedURI,
      sourceFile: event.sourceFile,
      lineNumber: event.lineNumber,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent
    };

    this.http.post('/api/csp-violations', report).subscribe({
      error: (err) => console.error('Failed to report CSP violation:', err)
    });
  }
}

// Component to display violations
@Component({
  selector: 'app-csp-dashboard',
  template: `
    <div class="csp-dashboard">
      <h2>CSP Violations</h2>
      <div *ngFor="let violation of violations$ | async" class="violation">
        <p><strong>Directive:</strong> {{ violation.violatedDirective }}</p>
        <p><strong>Blocked:</strong> {{ violation.blockedURI }}</p>
        <p><strong>Source:</strong> {{ violation.sourceFile }}:{{ violation.lineNumber }}</p>
      </div>
    </div>
  `
})
export class CspDashboardComponent {
  violations$ = this.cspMonitoring.getViolations();

  constructor(private cspMonitoring: CspMonitoringService) {}
}
```

## Strict CSP

Strict CSP eliminates `'unsafe-inline'` using nonces and `'strict-dynamic'`.

```typescript
// Strict CSP implementation
const strictCSPConfig = {
  generateHeaders: (nonce: string) => ({
    'Content-Security-Policy': [
      // Strict script policy with nonce
      `script-src 'nonce-${nonce}' 'strict-dynamic' https:`,
      // Object and base restrictions
      "object-src 'none'",
      "base-uri 'none'",
      // Allow inline styles (consider moving to external)
      "style-src 'self' 'unsafe-inline'",
      // Frame protections
      "frame-ancestors 'none'",
      // Default fallback
      "default-src 'self'",
      // Upgrade insecure requests
      "upgrade-insecure-requests"
    ].join('; ')
  })
};

// Express middleware
app.use((req, res, next) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  req.nonce = nonce;
  
  const headers = strictCSPConfig.generateHeaders(nonce);
  Object.entries(headers).forEach(([key, value]) => {
    res.setHeader(key, value);
  });
  
  next();
});
```

### Benefits of Strict CSP

```
┌─────────────────────────────────────────────────────────────┐
│          Strict CSP vs Traditional CSP                       │
└─────────────────────────────────────────────────────────────┘

Traditional CSP:
  script-src 'self' https://cdn1.com https://cdn2.com 'unsafe-inline'
  
  Problems:
  - 'unsafe-inline' defeats XSS protection
  - Maintaining allowlist is difficult
  - New CDN = CSP update required

Strict CSP:
  script-src 'nonce-{random}' 'strict-dynamic'
  
  Benefits:
  - No 'unsafe-inline' needed
  - No allowlist maintenance
  - 'strict-dynamic' propagates trust
  - Much stronger XSS protection

How 'strict-dynamic' works:
  1. <script nonce="abc123" src="/app.js">  ✓ Trusted
  2. app.js loads: <script src="/module.js">  ✓ Also trusted
  3. Attacker injects: <script src="evil.js">  ✗ Blocked
```

## Common Mistakes

### 1. Using 'unsafe-inline'

```typescript
// ❌ BAD: Defeats CSP protection
const badCSP = "script-src 'self' 'unsafe-inline'";
// Any injected <script> will execute

// ✓ GOOD: Use nonces or hashes
const goodCSP = (nonce: string) => 
  `script-src 'nonce-${nonce}' 'strict-dynamic'`;
```

### 2. Missing object-src 'none'

```typescript
// ❌ BAD: Allows plugins (Flash, Java)
const incompleteCSP = "default-src 'self'";

// ✓ GOOD: Explicitly block objects
const completeCSP = 
  "default-src 'self'; object-src 'none'; base-uri 'none'";
```

### 3. Overly Permissive Policies

```typescript
// ❌ BAD: Too permissive
const tooPermissive = "script-src 'self' https: data: 'unsafe-eval'";
// Allows any HTTPS script, data URIs, and eval()

// ✓ GOOD: Specific allowlist
const restrictive = 
  "script-src 'self' https://cdn.example.com https://apis.google.com";
```

### 4. Not Testing Report-Only First

```typescript
// ❌ BAD: Deploy enforcing CSP immediately
res.setHeader('Content-Security-Policy', strictPolicy);
// May break production

// ✓ GOOD: Test with Report-Only first
res.setHeader('Content-Security-Policy-Report-Only', strictPolicy);
// Monitor violations without breaking functionality
// After validation, switch to enforcing mode
```

### 5. Ignoring frame-ancestors

```typescript
// ❌ BAD: No frame protection
const noFrameProtection = "default-src 'self'";
// Page can be embedded in iframes (clickjacking risk)

// ✓ GOOD: Restrict framing
const withFrameProtection = 
  "default-src 'self'; frame-ancestors 'none'";
// Prevents clickjacking attacks
```

## Best Practices

### 1. Start with Report-Only

```typescript
// Progressive CSP deployment
class CSPDeployment {
  private phase: 'report-only' | 'enforcing' = 'report-only';
  private violationCount = 0;

  getCSPHeader(): { name: string; value: string } {
    const policy = this.buildPolicy();
    
    if (this.phase === 'report-only') {
      return {
        name: 'Content-Security-Policy-Report-Only',
        value: policy
      };
    }
    
    return {
      name: 'Content-Security-Policy',
      value: policy
    };
  }

  private buildPolicy(): string {
    return [
      "default-src 'self'",
      "script-src 'self' 'strict-dynamic'",
      "object-src 'none'",
      "base-uri 'none'",
      "report-uri /csp-violations"
    ].join('; ');
  }

  async checkViolations(): Promise<void> {
    const violations = await db.cspViolations.count({
      timestamp: { $gte: Date.now() - 86400000 } // Last 24h
    });

    if (violations === 0 && this.phase === 'report-only') {
      this.phase = 'enforcing';
      console.log('No violations detected. Switching to enforcing mode.');
    }
  }
}
```

### 2. Use Strict CSP

```typescript
// Implement strict CSP from the start
const strictCSP = {
  middleware: (req: any, res: any, next: any) => {
    const nonce = crypto.randomBytes(16).toString('base64');
    req.cspNonce = nonce;
    
    res.setHeader(
      'Content-Security-Policy',
      [
        `script-src 'nonce-${nonce}' 'strict-dynamic'`,
        "object-src 'none'",
        "base-uri 'none'",
        "require-trusted-types-for 'script'"
      ].join('; ')
    );
    
    next();
  }
};

app.use(strictCSP.middleware);
```

### 3. Automate CSP Management

```typescript
// Automated CSP header generation
class CSPManager {
  private config: CSPConfig;

  constructor(config: CSPConfig) {
    this.config = config;
  }

  generateHeader(nonce?: string): string {
    const directives: string[] = [];

    Object.entries(this.config).forEach(([directive, values]) => {
      if (Array.isArray(values)) {
        let sources = values.join(' ');
        
        // Inject nonce if needed
        if (nonce && directive.includes('script-src')) {
          sources = `'nonce-${nonce}' ${sources}`;
        }
        
        directives.push(`${directive} ${sources}`);
      }
    });

    return directives.join('; ');
  }

  addSource(directive: string, source: string): void {
    if (!this.config[directive]) {
      this.config[directive] = [];
    }
    
    if (!this.config[directive].includes(source)) {
      this.config[directive].push(source);
    }
  }

  removeSource(directive: string, source: string): void {
    if (this.config[directive]) {
      this.config[directive] = this.config[directive].filter(
        s => s !== source
      );
    }
  }
}

// Usage
const cspManager = new CSPManager({
  'default-src': ["'self'"],
  'script-src': ["'self'", "'strict-dynamic'"],
  'style-src': ["'self'", "'unsafe-inline'"]
});

app.use((req, res, next) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  req.cspNonce = nonce;
  
  res.setHeader(
    'Content-Security-Policy',
    cspManager.generateHeader(nonce)
  );
  
  next();
});
```

### 4. Monitor and Alert

```typescript
// CSP violation monitoring system
class CSPViolationMonitor {
  private readonly ALERT_THRESHOLD = 10;
  private violationCounts = new Map<string, number>();

  async handleViolation(violation: CSPViolation): Promise<void> {
    // Log violation
    await this.logViolation(violation);

    // Check for attack patterns
    if (this.isAttackPattern(violation)) {
      await this.alertSecurityTeam(violation);
    }

    // Track violation frequency
    this.trackViolation(violation);

    // Check threshold
    if (this.shouldAlert(violation)) {
      await this.alertHighViolationRate(violation);
    }
  }

  private isAttackPattern(violation: CSPViolation): boolean {
    const blockedUri = violation['blocked-uri'];
    
    // Check for known attack patterns
    const attackPatterns = [
      /eval\(/,
      /javascript:/,
      /data:text\/html/,
      /vbscript:/,
      /<script/
    ];

    return attackPatterns.some(pattern => 
      pattern.test(blockedUri)
    );
  }

  private trackViolation(violation: CSPViolation): void {
    const key = `${violation['violated-directive']}:${violation['blocked-uri']}`;
    const count = this.violationCounts.get(key) || 0;
    this.violationCounts.set(key, count + 1);
  }

  private shouldAlert(violation: CSPViolation): boolean {
    const key = `${violation['violated-directive']}:${violation['blocked-uri']}`;
    const count = this.violationCounts.get(key) || 0;
    return count >= this.ALERT_THRESHOLD;
  }
}
```

## When to Use/Not to Use

### When to Use CSP

1. **All web applications** - CSP is a fundamental security layer
2. **User-generated content** - Prevents XSS from untrusted input
3. **Third-party integrations** - Controls external script execution
4. **Financial applications** - Critical for PCI DSS compliance
5. **High-security environments** - Defense-in-depth strategy

### CSP Considerations

```
┌─────────────────────────────────────────────────────────────┐
│              CSP Implementation Checklist                    │
└─────────────────────────────────────────────────────────────┘

Phase 1: Planning
  □ Audit existing inline scripts
  □ Identify external resource dependencies
  □ Choose nonce vs hash approach
  □ Plan reporting infrastructure

Phase 2: Implementation
  □ Deploy CSP in Report-Only mode
  □ Monitor violations for 1-2 weeks
  □ Fix legitimate violations
  □ Refine policy iteratively

Phase 3: Enforcement
  □ Switch to enforcing mode
  □ Continue monitoring
  □ Update policy as app evolves
  □ Regular security audits

Complexity Levels:
  Basic:    "default-src 'self'"
  Medium:   Nonce-based CSP with CDN allowlist
  Advanced: Strict CSP with 'strict-dynamic' and Trusted Types
```

### CSP Limitations

1. **Browser compatibility** - Older browsers may not support all directives
2. **Legacy code** - Inline scripts/styles require refactoring
3. **Third-party widgets** - Some widgets incompatible with strict CSP
4. **Development overhead** - Requires careful planning and testing

---

## Supplementary: Trusted Types

Trusted Types is a browser API that forces all DOM sinks that accept HTML, scripts, or URLs to receive only values created through a defined policy — preventing DOM-based XSS by making injection a type error rather than a runtime vulnerability.

### The Problem

```javascript
// Any of these can execute attacker-controlled HTML/JS
element.innerHTML = userInput;         // DOM XSS sink
document.write(userInput);
location.href = userInput;
new Worker(userInput);
eval(userInput);
script.src = userInput;
```

### Enabling Trusted Types via CSP

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types myPolicy
```

Once enabled, assigning a plain string to a sink like `innerHTML` throws a `TypeError`.

### Defining and Using Policies

```javascript
// Create a policy — the ONLY way to produce Trusted values
const policy = trustedTypes.createPolicy('myPolicy', {
  createHTML: (input) => {
    // Sanitize — only return after cleaning
    return DOMPurify.sanitize(input, { RETURN_TRUSTED_TYPE: true });
  },
  createScriptURL: (url) => {
    // Only allow same-origin script URLs
    const parsed = new URL(url, location.origin);
    if (parsed.origin !== location.origin) throw new Error('Disallowed URL');
    return url;
  },
  createScript: (script) => {
    // Highly restrictive — usually disallow
    throw new Error('Dynamic script creation not allowed');
  },
});

// Using the policy
const safeHTML = policy.createHTML('<b>Hello</b>');
element.innerHTML = safeHTML; // ✓ — TrustedHTML, not string

const safeURL = policy.createScriptURL('/scripts/app.js');
worker = new Worker(safeURL); // ✓ — TrustedScriptURL
```

### Default Policy (Compatibility Shim)

For code you can't change (third-party libraries):

```javascript
// Catches all untyped assignments — use to audit or temporarily allow
trustedTypes.createPolicy('default', {
  createHTML: (input) => {
    console.warn('Untyped innerHTML assignment:', input); // audit
    return DOMPurify.sanitize(input); // sanitize rather than block
  },
});
```

### Framework Support
- **Angular**: DOMSanitizer integrates with Trusted Types natively in Angular 11+
- **React**: Sets `innerHTML` safely via its own escaping — Trusted Types compatible
- **DOMPurify**: Supports `RETURN_TRUSTED_TYPE: true` option

---

## Interview Questions

### Q1: What is Content Security Policy and how does it prevent XSS?

**Answer:** CSP is an HTTP header that tells browsers which sources of content are safe to load and execute. It prevents XSS by:

1. **Blocking inline scripts** - By default, blocks `<script>` tags and inline event handlers unless explicitly allowed via nonce/hash
2. **Restricting external sources** - Only scripts from allowlisted domains can execute
3. **Preventing eval()** - Blocks `eval()`, `new Function()`, and `setTimeout(string)` by default
4. **Controlling data injection** - Prevents loading of malicious external resources

Example: With `script-src 'self'`, if an attacker injects `<script src="http://evil.com/malware.js"></script>`, the browser blocks it because evil.com is not in the allowlist.

### Q2: Explain the difference between nonce-based and hash-based CSP.

**Answer:**

**Nonce-based CSP:**
- Server generates random value per request
- Adds nonce to CSP header and script tags
- Pros: Dynamic, works for varying inline scripts
- Cons: Requires server-side rendering or nonce injection

**Hash-based CSP:**
- Compute SHA hash of script content
- Add hash to CSP header
- Pros: Works with static sites, no per-request randomness needed
- Cons: Content changes require new hash, can't be used for dynamic scripts

**When to use each:**
- Nonce: Server-rendered pages, SPAs with SSR
- Hash: Static sites, build-time scripts, unchanging inline scripts

### Q3: What is 'strict-dynamic' and why is it useful?

**Answer:** `'strict-dynamic'` is a CSP source expression that propagates trust to scripts loaded by already-trusted scripts.

**How it works:**
```javascript
// CSP: script-src 'nonce-abc123' 'strict-dynamic'

// Initial script (trusted via nonce)
<script nonce="abc123" src="/app.js"></script>

// app.js dynamically loads (automatically trusted)
const script = document.createElement('script');
script.src = '/module.js';  // ✓ Allowed via strict-dynamic
document.head.appendChild(script);

// Attacker injection (not trusted)
<script src="http://evil.com/malware.js"></script>  // ✗ Blocked
```

**Benefits:**
- Eliminates need for domain allowlists
- Backwards compatible (falls back to allowlist in older browsers)
- Simplifies CSP management
- Enables modern app architectures (code splitting, lazy loading)

**Best practice:** Combine with nonces and disable `'unsafe-inline'` for maximum security.

### Q4: How do you handle third-party scripts with strict CSP?

**Answer:** Strategies for third-party scripts under strict CSP:

**1. Load from trusted script:**
```typescript
// Initial script with nonce
<script nonce="abc123">
  const script = document.createElement('script');
  script.src = 'https://third-party.com/widget.js';
  document.head.appendChild(script);  // Trusted via strict-dynamic
</script>
```

**2. Use hash for static third-party scripts:**
```typescript
// Compute hash of third-party script
const hash = generateHash(thirdPartyScriptContent);
// CSP: script-src 'sha256-{hash}' 'strict-dynamic'
```

**3. Allowlist specific domains:**
```typescript
// CSP: script-src 'nonce-abc' 'strict-dynamic' https://trusted-cdn.com
```

**4. Host locally:**
- Download and serve from your domain
- Include in build process
- Ensures control and stability

**5. Use sandbox iframes:**
```html
<iframe 
  sandbox="allow-scripts" 
  src="https://third-party.com/widget"
  csp="script-src 'unsafe-inline' https://third-party.com">
</iframe>
```

### Q5: What is CSP Report-Only mode and when should you use it?

**Answer:** Report-Only mode sends violation reports without blocking content.

**Header:**
```
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp
```

**When to use:**
1. **Initial CSP deployment** - Test policy without breaking production
2. **Policy changes** - Validate new rules before enforcement
3. **Continuous monitoring** - Run alongside enforcing policy for additional visibility
4. **Legacy code** - Identify all inline scripts/styles before refactoring

**Workflow:**
```
1. Deploy Report-Only → Monitor 1-2 weeks
2. Analyze violations → Fix legitimate issues
3. Refine policy → Re-deploy Report-Only
4. Repeat until zero legitimate violations
5. Switch to enforcing mode → Continue monitoring
```

**Benefits:**
- No user impact during testing
- Comprehensive violation discovery
- Safe policy iteration

### Q6: How do you implement CSP in a single-page application?

**Answer:** SPA CSP implementation strategies:

**Option 1: Nonce-based with SSR:**
```typescript
// Server generates nonce
const nonce = crypto.randomBytes(16).toString('base64');
res.setHeader('CSP', `script-src 'nonce-${nonce}' 'strict-dynamic'`);

// Inject nonce into initial HTML
<script nonce="${nonce}" src="/app.js"></script>
```

**Option 2: Hash-based for static builds:**
```typescript
// Build tool generates hashes
const hashes = generateHashesForInlineScripts();
// Add to CSP at build time
```

**Option 3: Externalize all scripts:**
```typescript
// Move all inline scripts to external files
// CSP: script-src 'self'
<script src="/config.js"></script>  // Instead of inline
```

**Challenges:**
- Dynamic imports (use strict-dynamic)
- Analytics/tracking (load from trusted script)
- Third-party widgets (iframe sandboxing)

**Best practice:**
- Use nonce-based CSP with SSR
- Apply strict-dynamic for dynamic imports
- Externalize configuration
- Monitor violations continuously

### Q7: What are Trusted Types and how do they complement CSP?

**Answer:** Trusted Types is a browser API that prevents DOM XSS by requiring explicit validation before dangerous sinks (innerHTML, eval, etc.).

**How it works:**
```typescript
// CSP: require-trusted-types-for 'script'

// Without Trusted Types (blocked)
element.innerHTML = userInput;  // ✗ Error

// With Trusted Types (safe)
const policy = trustedTypes.createPolicy('myPolicy', {
  createHTML: (input) => {
    return DOMPurify.sanitize(input);
  }
});

element.innerHTML = policy.createHTML(userInput);  // ✓ Allowed
```

**Benefits:**
- Prevents DOM XSS at runtime
- Forces sanitization at dangerous sinks
- Complements CSP (CSP blocks script loading, TT blocks DOM injection)
- Audit-friendly (all dangerous operations go through policies)

**CSP directive:**
```
Content-Security-Policy: require-trusted-types-for 'script'; 
                         trusted-types myPolicy defaultPolicy
```

**Together with CSP:** Defense in depth - CSP prevents unauthorized script loading, Trusted Types prevent DOM XSS from existing scripts.

### Q8: How do you debug CSP violations in production?

**Answer:** Systematic approach to CSP debugging:

**1. Monitor violation reports:**
```typescript
// Backend: Log and analyze violations
app.post('/csp-report', (req, res) => {
  const violation = req.body['csp-report'];
  
  console.error('CSP Violation:', {
    blockedURI: violation['blocked-uri'],
    violatedDirective: violation['violated-directive'],
    documentURI: violation['document-uri'],
    sourceFile: violation['source-file'],
    lineNumber: violation['line-number']
  });
  
  // Store for analysis
  db.violations.insert(violation);
  res.status(204).send();
});
```

**2. Browser DevTools:**
- Console shows CSP violations with details
- Network tab shows blocked requests
- Security tab shows CSP policy

**3. Test in Report-Only mode:**
```typescript
// Deploy fix in Report-Only first
res.setHeader('Content-Security-Policy-Report-Only', updatedPolicy);
```

**4. Categorize violations:**
- Legitimate (need policy update)
- Attack attempts (investigate further)
- Development artifacts (fix in code)
- Third-party issues (negotiate or replace)

**5. Common solutions:**
- Add missing source to allowlist
- Add nonce to inline script
- Externalize inline code
- Use strict-dynamic for dynamic scripts

## Key Takeaways

1. **CSP is essential XSS defense** - Provides strong protection layer against code injection attacks by controlling resource loading
2. **Start with Report-Only mode** - Test policies thoroughly without breaking production before enforcing restrictions
3. **Avoid 'unsafe-inline'** - Use nonces or hashes for inline scripts; 'unsafe-inline' defeats CSP's XSS protection
4. **Use 'strict-dynamic' for modern apps** - Eliminates allowlist maintenance and enables dynamic script loading patterns
5. **Implement comprehensive policies** - Include object-src, base-uri, frame-ancestors, not just script-src
6. **Monitor violations continuously** - Set up reporting to detect both attacks and legitimate policy violations
7. **Nonces require server-side rendering** - Generate unique nonce per request and inject into HTML and CSP header
8. **Hash-based CSP for static content** - Use hashes for unchanging inline scripts, especially in static site builds
9. **Defense in depth** - Combine CSP with other security measures (HTTPS, SameSite cookies, input validation)
10. **Plan for third-party scripts** - Third-party integrations often conflict with strict CSP; plan integration strategy carefully

## Resources

### Official Documentation
- [MDN: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [CSP Level 3 Specification](https://www.w3.org/TR/CSP3/)
- [Google: CSP Best Practices](https://web.dev/strict-csp/)

### Tools
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/) - Analyze and improve your CSP
- [Report URI](https://report-uri.com/) - CSP violation reporting service
- [CSP Generator](https://www.cspisawesome.com/) - Visual CSP policy builder

### Libraries
- [Helmet.js](https://helmetjs.github.io/) - Express security headers including CSP
- [csp-header](https://www.npmjs.com/package/csp-header) - CSP header builder
- [Trusted Types](https://github.com/w3c/webappsec-trusted-types) - DOM XSS prevention

### Articles and Guides
- [CSP: A Successful Mess Between Hardening And Mitigation](https://research.google/pubs/pub45542/)
- [Strict CSP](https://web.dev/strict-csp/)
- [CSP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html)
