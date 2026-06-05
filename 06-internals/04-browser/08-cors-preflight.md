# CORS and Preflight Requests

## The Idea

**In plain English:** CORS (Cross-Origin Resource Sharing) is the browser's security rule that prevents a webpage on one website from secretly reading data from a completely different website — unless that other site explicitly says it's okay.

**Real-world analogy:** Think of a nightclub with a guest list. If you show up and you're not on the list, the bouncer turns you away — even if the party host inside would actually be happy to see you. The host has to add your name to the list first (set the right headers on the server), otherwise the bouncer (the browser) blocks you regardless.

- The nightclub = the server being requested
- The guest list = the `Access-Control-Allow-Origin` response header
- The bouncer = the browser enforcing CORS
- The preflight "are you on the list?" check = the `OPTIONS` request sent before the real request

---

## Table of Contents
- [Introduction](#introduction)
- [Same-Origin Policy](#same-origin-policy)
- [CORS Headers](#cors-headers)
- [Preflight Requests](#preflight-requests)
- [Credentials Mode](#credentials-mode)
- [Security Implications](#security-implications)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Browser Differences](#browser-differences)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Cross-Origin Resource Sharing (CORS) is a security mechanism that allows web pages to make requests to different domains than the one serving the web page. Understanding CORS is essential for building modern web applications that interact with APIs, CDNs, and third-party services. The preflight mechanism ensures that cross-origin requests are safe before allowing them to proceed.

## Same-Origin Policy

### Understanding Origins

An origin is defined by the protocol, domain, and port:

```javascript
// Origin comparison examples
const originExamples = {
  baseUrl: 'https://example.com:443/page',
  
  sameOrigin: [
    'https://example.com:443/api',      // Same everything
    'https://example.com/other',        // Port 443 is default for HTTPS
  ],
  
  differentOrigin: [
    'http://example.com',               // Different protocol
    'https://api.example.com',          // Different subdomain
    'https://example.com:8080',         // Different port
    'https://example.org',              // Different domain
  ]
};

// Check if two URLs have the same origin
function isSameOrigin(url1, url2) {
  const origin1 = new URL(url1);
  const origin2 = new URL(url2);
  
  return origin1.protocol === origin2.protocol &&
         origin1.hostname === origin2.hostname &&
         origin1.port === origin2.port;
}

// Usage
console.log(isSameOrigin(
  'https://example.com/page1',
  'https://example.com/page2'
)); // true

console.log(isSameOrigin(
  'https://example.com',
  'https://api.example.com'
)); // false (different subdomain)
```

### What Same-Origin Policy Restricts

```javascript
// SOP restrictions
class SameOriginPolicyDemo {
  static demonstrateRestrictions() {
    const restrictions = {
      xhr: 'XMLHttpRequest to different origin blocked',
      fetch: 'Fetch API to different origin blocked',
      canvas: 'Reading canvas with cross-origin image blocked',
      iframe: 'Accessing cross-origin iframe content blocked',
      localStorage: 'Cannot access another origin\'s localStorage',
      cookies: 'Cannot read another origin\'s cookies',
      dom: 'Cannot access cross-origin document/window properties'
    };

    return restrictions;
  }

  // Example: Blocked cross-origin request
  static async blockedRequest() {
    try {
      // This will fail if CORS headers not present
      const response = await fetch('https://different-origin.com/api/data');
      const data = await response.json();
    } catch (error) {
      console.error('CORS Error:', error);
      // Error: CORS policy: No 'Access-Control-Allow-Origin' header
    }
  }

  // Example: Blocked canvas access
  static blockedCanvasAccess() {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    const img = new Image();
    
    img.crossOrigin = 'anonymous'; // Required for cross-origin
    img.src = 'https://different-origin.com/image.jpg';
    
    img.onload = () => {
      ctx.drawImage(img, 0, 0);
      
      try {
        // This will fail without proper CORS headers
        const imageData = canvas.toDataURL();
      } catch (error) {
        console.error('Canvas tainted by cross-origin data');
      }
    };
  }

  // Example: Blocked iframe access
  static blockedIframeAccess() {
    const iframe = document.createElement('iframe');
    iframe.src = 'https://different-origin.com/page';
    document.body.appendChild(iframe);
    
    iframe.onload = () => {
      try {
        // This will fail
        const doc = iframe.contentDocument;
        console.log(doc.body.innerHTML);
      } catch (error) {
        console.error('Cannot access cross-origin iframe:', error);
        // DOMException: Blocked a frame with origin
      }
    };
  }
}
```

### Allowed Cross-Origin Operations

```javascript
// Operations that are allowed without CORS
const allowedCrossOriginOps = {
  // Embedding resources
  images: {
    allowed: true,
    example: '<img src="https://other-domain.com/image.jpg">',
    caveat: 'Cannot read pixel data without CORS'
  },
  
  scripts: {
    allowed: true,
    example: '<script src="https://cdn.com/library.js"></script>',
    caveat: 'Script can execute but source not readable'
  },
  
  styles: {
    allowed: true,
    example: '<link rel="stylesheet" href="https://cdn.com/style.css">',
    caveat: 'Cannot read CSSOM without CORS'
  },
  
  media: {
    allowed: true,
    example: '<video src="https://cdn.com/video.mp4">',
    caveat: 'Cannot manipulate media data without CORS'
  },
  
  // Simple form submissions
  forms: {
    allowed: true,
    example: '<form action="https://other-domain.com/submit" method="POST">',
    caveat: 'Cannot read response without CORS'
  },
  
  // WebSockets
  websockets: {
    allowed: true,
    example: 'new WebSocket("wss://other-domain.com")',
    caveat: 'Server must validate Origin header'
  }
};

// Demonstration
class AllowedCrossOriginDemo {
  static loadCrossOriginImage() {
    const img = new Image();
    img.src = 'https://example.com/image.jpg';
    img.onload = () => {
      console.log('Image loaded successfully');
      // Image displayed but cannot access pixel data
    };
    document.body.appendChild(img);
  }

  static loadCrossOriginScript() {
    const script = document.createElement('script');
    script.src = 'https://cdn.example.com/library.js';
    script.onload = () => {
      console.log('Script loaded and executed');
      // Can use library but cannot read source code
    };
    document.head.appendChild(script);
  }

  static submitCrossOriginForm() {
    const form = document.createElement('form');
    form.method = 'POST';
    form.action = 'https://api.example.com/submit';
    
    const input = document.createElement('input');
    input.name = 'data';
    input.value = 'test';
    form.appendChild(input);
    
    document.body.appendChild(form);
    form.submit(); // Allowed, but cannot read response
  }
}
```

## CORS Headers

### Access-Control-Allow-Origin

The fundamental CORS header:

```javascript
// Server-side CORS configuration examples
class CORSHeaderExamples {
  // Example 1: Allow all origins (dangerous)
  static allowAllOrigins(req, res) {
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.json({ message: 'Public API' });
  }

  // Example 2: Allow specific origin
  static allowSpecificOrigin(req, res) {
    res.setHeader('Access-Control-Allow-Origin', 'https://trusted-site.com');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST');
    res.json({ message: 'Restricted API' });
  }

  // Example 3: Dynamic origin validation
  static dynamicOriginValidation(req, res) {
    const allowedOrigins = [
      'https://app.example.com',
      'https://admin.example.com',
      'http://localhost:3000' // Development
    ];
    
    const origin = req.headers.origin;
    
    if (allowedOrigins.includes(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Access-Control-Allow-Credentials', 'true');
    } else {
      // Don't set CORS headers for untrusted origins
      res.status(403).json({ error: 'Origin not allowed' });
      return;
    }
    
    res.json({ message: 'Authenticated API' });
  }

  // Example 4: Regex-based origin matching
  static regexOriginMatching(req, res) {
    const origin = req.headers.origin;
    const allowedPattern = /^https:\/\/[\w-]+\.example\.com$/;
    
    if (allowedPattern.test(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.json({ message: 'Subdomain access granted' });
    } else {
      res.status(403).json({ error: 'Origin not allowed' });
    }
  }
}

// Client-side: Making CORS requests
class CORSClientExamples {
  // Simple CORS request
  static async simpleCORSRequest() {
    try {
      const response = await fetch('https://api.example.com/data', {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json'
        }
      });
      
      const data = await response.json();
      return data;
    } catch (error) {
      console.error('CORS Error:', error);
      throw error;
    }
  }

  // CORS request with credentials
  static async corsWithCredentials() {
    const response = await fetch('https://api.example.com/user', {
      method: 'GET',
      credentials: 'include', // Send cookies
      headers: {
        'Content-Type': 'application/json'
      }
    });
    
    return response.json();
  }

  // Custom headers (triggers preflight)
  static async corsWithCustomHeaders() {
    const response = await fetch('https://api.example.com/data', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Custom-Header': 'custom-value', // Custom header
        'Authorization': 'Bearer token123'
      },
      body: JSON.stringify({ data: 'example' })
    });
    
    return response.json();
  }
}
```

### Complete CORS Header Reference

```javascript
// Comprehensive CORS headers
const CORSHeaders = {
  // Request headers
  request: {
    'Origin': 'https://example.com',
    'Access-Control-Request-Method': 'POST',
    'Access-Control-Request-Headers': 'Content-Type, Authorization'
  },
  
  // Response headers
  response: {
    // Origin control
    'Access-Control-Allow-Origin': {
      values: ['*', 'https://example.com', 'null'],
      description: 'Specifies allowed origin(s)',
      notes: 'Cannot use * with credentials'
    },
    
    // Methods
    'Access-Control-Allow-Methods': {
      values: ['GET, POST, PUT, DELETE, OPTIONS'],
      description: 'Allowed HTTP methods',
      notes: 'Applies to preflight requests'
    },
    
    // Headers
    'Access-Control-Allow-Headers': {
      values: ['Content-Type, Authorization, X-Custom-Header'],
      description: 'Allowed request headers',
      notes: 'Required if custom headers sent'
    },
    
    // Credentials
    'Access-Control-Allow-Credentials': {
      values: ['true'],
      description: 'Allow cookies and auth headers',
      notes: 'Requires specific origin (not *)'
    },
    
    // Exposed headers
    'Access-Control-Expose-Headers': {
      values: ['X-Custom-Response-Header, X-Total-Count'],
      description: 'Headers accessible to JavaScript',
      notes: 'Default: Cache-Control, Content-Language, etc.'
    },
    
    // Preflight cache
    'Access-Control-Max-Age': {
      values: ['86400'], // 24 hours
      description: 'Cache preflight response duration',
      notes: 'In seconds, browser may have max limit'
    }
  }
};

// Server implementation
class FullCORSImplementation {
  static handleRequest(req, res) {
    // Handle preflight
    if (req.method === 'OPTIONS') {
      return this.handlePreflight(req, res);
    }
    
    // Handle actual request
    return this.handleActualRequest(req, res);
  }

  static handlePreflight(req, res) {
    const origin = req.headers.origin;
    const requestMethod = req.headers['access-control-request-method'];
    const requestHeaders = req.headers['access-control-request-headers'];
    
    // Validate origin
    if (!this.isOriginAllowed(origin)) {
      res.status(403).send('Origin not allowed');
      return;
    }
    
    // Set CORS headers
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', requestHeaders || 'Content-Type, Authorization');
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Access-Control-Max-Age', '86400');
    
    // No content for preflight
    res.status(204).send();
  }

  static handleActualRequest(req, res) {
    const origin = req.headers.origin;
    
    if (this.isOriginAllowed(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Access-Control-Allow-Credentials', 'true');
      res.setHeader('Access-Control-Expose-Headers', 'X-Total-Count, X-Page-Number');
    }
    
    // Process request
    res.json({ data: 'response' });
  }

  static isOriginAllowed(origin) {
    const allowed = [
      'https://app.example.com',
      'https://admin.example.com'
    ];
    return allowed.includes(origin);
  }
}
```

## Preflight Requests

### When Preflight is Triggered

```javascript
// Understanding preflight triggers
class PreflightTriggers {
  // Simple requests (NO preflight)
  static simpleRequests = {
    methods: ['GET', 'HEAD', 'POST'],
    
    contentTypes: [
      'application/x-www-form-urlencoded',
      'multipart/form-data',
      'text/plain'
    ],
    
    headers: [
      'Accept',
      'Accept-Language',
      'Content-Language',
      'Content-Type', // Only above types
      'Range' // Simple range
    ],
    
    examples: [
      // Example 1: Simple GET
      {
        method: 'GET',
        headers: { 'Accept': 'application/json' },
        preflight: false
      },
      
      // Example 2: Simple POST with form data
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: 'name=value',
        preflight: false
      },
      
      // Example 3: Simple POST with text
      {
        method: 'POST',
        headers: { 'Content-Type': 'text/plain' },
        body: 'plain text',
        preflight: false
      }
    ]
  };

  // Preflighted requests (preflight required)
  static preflightedRequests = {
    reasons: [
      'Non-simple HTTP methods',
      'Custom headers',
      'Non-simple Content-Type',
      'ReadableStream body'
    ],
    
    examples: [
      // Example 1: Custom method
      {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        preflight: true,
        reason: 'PUT is not a simple method'
      },
      
      // Example 2: JSON Content-Type
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        preflight: true,
        reason: 'application/json is not a simple content type'
      },
      
      // Example 3: Custom header
      {
        method: 'GET',
        headers: { 
          'Authorization': 'Bearer token',
          'X-Custom-Header': 'value'
        },
        preflight: true,
        reason: 'Authorization and custom headers'
      },
      
      // Example 4: DELETE method
      {
        method: 'DELETE',
        preflight: true,
        reason: 'DELETE is not a simple method'
      }
    ]
  };

  // Detection utility
  static isSimpleRequest(method, headers, contentType) {
    // Check method
    if (!['GET', 'HEAD', 'POST'].includes(method.toUpperCase())) {
      return false;
    }
    
    // Check Content-Type
    const simpleTypes = [
      'application/x-www-form-urlencoded',
      'multipart/form-data',
      'text/plain'
    ];
    
    if (contentType && !simpleTypes.includes(contentType.split(';')[0].trim())) {
      return false;
    }
    
    // Check headers
    const simpleHeaders = [
      'accept', 'accept-language', 'content-language',
      'content-type', 'range'
    ];
    
    for (const header of Object.keys(headers)) {
      if (!simpleHeaders.includes(header.toLowerCase())) {
        return false;
      }
    }
    
    return true;
  }
}

// Examples
const examples = [
  {
    method: 'GET',
    headers: { 'Accept': 'application/json' },
    expected: 'No preflight'
  },
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    expected: 'Preflight required'
  },
  {
    method: 'PUT',
    headers: { 'Content-Type': 'text/plain' },
    expected: 'Preflight required (method)'
  }
];

examples.forEach(ex => {
  const isSimple = PreflightTriggers.isSimpleRequest(
    ex.method,
    ex.headers,
    ex.headers['Content-Type']
  );
  console.log(`${ex.method}: ${isSimple ? 'Simple' : 'Preflight'} - ${ex.expected}`);
});
```

### Preflight Request Flow

```javascript
// Complete preflight flow
class PreflightFlow {
  static async demonstrateFlow() {
    console.log('=== CORS Preflight Flow ===\n');
    
    // Step 1: Browser detects preflighted request
    console.log('Step 1: Browser prepares POST with JSON');
    const actualRequest = {
      method: 'POST',
      url: 'https://api.example.com/users',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer token123'
      },
      body: JSON.stringify({ name: 'John' })
    };
    console.log('Actual request:', actualRequest);
    
    // Step 2: Browser automatically sends preflight
    console.log('\nStep 2: Browser sends OPTIONS request');
    const preflightRequest = {
      method: 'OPTIONS',
      url: 'https://api.example.com/users',
      headers: {
        'Origin': 'https://myapp.com',
        'Access-Control-Request-Method': 'POST',
        'Access-Control-Request-Headers': 'Content-Type, Authorization'
      }
    };
    console.log('Preflight request:', preflightRequest);
    
    // Step 3: Server responds to preflight
    console.log('\nStep 3: Server responds to preflight');
    const preflightResponse = {
      status: 204,
      headers: {
        'Access-Control-Allow-Origin': 'https://myapp.com',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        'Access-Control-Max-Age': '86400'
      }
    };
    console.log('Preflight response:', preflightResponse);
    
    // Step 4: Browser validates preflight response
    console.log('\nStep 4: Browser validates preflight');
    const validation = this.validatePreflight(
      actualRequest,
      preflightResponse
    );
    console.log('Validation:', validation);
    
    if (validation.allowed) {
      // Step 5: Browser sends actual request
      console.log('\nStep 5: Browser sends actual request');
      console.log('Actual request sent with original headers and body');
      
      // Step 6: Server processes actual request
      console.log('\nStep 6: Server responds to actual request');
      const actualResponse = {
        status: 201,
        headers: {
          'Access-Control-Allow-Origin': 'https://myapp.com',
          'Content-Type': 'application/json'
        },
        body: { id: 123, name: 'John' }
      };
      console.log('Actual response:', actualResponse);
    } else {
      console.log('\nPreflight failed:', validation.reason);
      console.log('Actual request NOT sent');
    }
  }

  static validatePreflight(actualRequest, preflightResponse) {
    const allowedOrigin = preflightResponse.headers['Access-Control-Allow-Origin'];
    const allowedMethods = preflightResponse.headers['Access-Control-Allow-Methods'];
    const allowedHeaders = preflightResponse.headers['Access-Control-Allow-Headers'];
    
    // Check origin
    if (allowedOrigin !== '*' && allowedOrigin !== 'https://myapp.com') {
      return { allowed: false, reason: 'Origin not allowed' };
    }
    
    // Check method
    if (!allowedMethods.includes(actualRequest.method)) {
      return { allowed: false, reason: 'Method not allowed' };
    }
    
    // Check headers
    const requestHeaders = Object.keys(actualRequest.headers);
    const allowedHeadersArray = allowedHeaders.toLowerCase().split(',').map(h => h.trim());
    
    for (const header of requestHeaders) {
      if (!allowedHeadersArray.includes(header.toLowerCase())) {
        return { allowed: false, reason: `Header ${header} not allowed` };
      }
    }
    
    return { allowed: true };
  }
}

// Run demonstration
PreflightFlow.demonstrateFlow();
```

### Optimizing Preflight Performance

```javascript
// Preflight optimization strategies
class PreflightOptimization {
  // Strategy 1: Maximize Access-Control-Max-Age
  static setMaxAge(res) {
    // Cache preflight for as long as possible
    res.setHeader('Access-Control-Max-Age', '86400'); // 24 hours
    
    // Note: Browsers may have their own limits
    // Chrome: 2 hours max, Firefox: 24 hours max
  }

  // Strategy 2: Use simple requests when possible
  static simplifyRequest() {
    // Instead of this (triggers preflight):
    const preflightRequest = {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ name: 'John' })
    };
    
    // Consider this (no preflight):
    const simpleRequest = {
      method: 'POST',
      headers: {
        'Content-Type': 'text/plain'
      },
      body: 'name=John' // Parse on server
    };
    
    return { preflightRequest, simpleRequest };
  }

  // Strategy 3: Batch operations to reduce preflights
  static batchOperations() {
    // Instead of multiple DELETE requests (each triggers preflight):
    const multipleDeletes = [
      fetch('https://api.example.com/items/1', { method: 'DELETE' }),
      fetch('https://api.example.com/items/2', { method: 'DELETE' }),
      fetch('https://api.example.com/items/3', { method: 'DELETE' })
    ]; // 3 preflights + 3 requests = 6 total
    
    // Use single request with multiple IDs:
    const batchDelete = fetch('https://api.example.com/items', {
      method: 'POST', // Or DELETE with body
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ids: [1, 2, 3] })
    }); // 1 preflight + 1 request = 2 total
    
    return { multipleDeletes, batchDelete };
  }

  // Strategy 4: Monitor preflight cache hit rate
  static monitorPreflightCache() {
    const observer = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      
      entries.forEach(entry => {
        if (entry.name.includes('api.example.com')) {
          // Check if preflight was sent
          const hasOptions = performance.getEntriesByName(entry.name)
            .some(e => e.initiatorType === 'fetch' && 
                       e.requestStart > 0);
          
          if (hasOptions) {
            console.log('Preflight sent for:', entry.name);
          } else {
            console.log('Preflight cached for:', entry.name);
          }
        }
      });
    });
    
    observer.observe({ entryTypes: ['resource'] });
  }

  // Strategy 5: Use proxy for same-origin requests
  static setupProxy() {
    // Development: Proxy API requests through same origin
    const proxyConfig = {
      '/api': {
        target: 'https://api.example.com',
        changeOrigin: true,
        pathRewrite: { '^/api': '' }
      }
    };
    
    // Request becomes same-origin (no CORS):
    // fetch('/api/users') -> proxied to https://api.example.com/users
    
    return proxyConfig;
  }
}
```

## Credentials Mode

### Understanding Credentials

```javascript
// Credentials in CORS requests
class CORSCredentials {
  // Credentials modes
  static modes = {
    omit: {
      description: 'Never send cookies or auth headers',
      usage: 'Public APIs, anonymous requests',
      example: { credentials: 'omit' }
    },
    
    'same-origin': {
      description: 'Send credentials only for same-origin requests',
      usage: 'Default behavior',
      example: { credentials: 'same-origin' }
    },
    
    include: {
      description: 'Always send credentials, even cross-origin',
      usage: 'Authenticated APIs',
      example: { credentials: 'include' },
      requires: [
        'Access-Control-Allow-Origin must be specific (not *)',
        'Access-Control-Allow-Credentials: true'
      ]
    }
  };

  // Example 1: Request without credentials
  static async publicAPIRequest() {
    const response = await fetch('https://api.example.com/public', {
      credentials: 'omit' // No cookies sent
    });
    
    return response.json();
  }

  // Example 2: Request with credentials
  static async authenticatedRequest() {
    const response = await fetch('https://api.example.com/user', {
      credentials: 'include', // Send cookies and Authorization
      headers: {
        'Authorization': 'Bearer token123'
      }
    });
    
    // Server must respond with:
    // Access-Control-Allow-Origin: https://myapp.com (specific!)
    // Access-Control-Allow-Credentials: true
    
    return response.json();
  }

  // Server-side: Handling credentials
  static handleCredentials(req, res) {
    const origin = req.headers.origin;
    
    // INCORRECT: Cannot use * with credentials
    // res.setHeader('Access-Control-Allow-Origin', '*');
    // res.setHeader('Access-Control-Allow-Credentials', 'true');
    
    // CORRECT: Use specific origin
    if (this.isOriginAllowed(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Access-Control-Allow-Credentials', 'true');
      
      // Verify authentication
      const token = req.cookies.sessionToken;
      if (!this.isValidToken(token)) {
        res.status(401).json({ error: 'Unauthorized' });
        return;
      }
      
      res.json({ user: 'authenticated' });
    } else {
      res.status(403).json({ error: 'Origin not allowed' });
    }
  }

  static isOriginAllowed(origin) {
    const allowed = ['https://app.example.com', 'https://admin.example.com'];
    return allowed.includes(origin);
  }

  static isValidToken(token) {
    // Validate session token
    return token && token.length > 0;
  }
}

// Cookie handling in CORS
class CORSCookies {
  // Setting cookies in CORS response
  static setCookie(res) {
    res.setHeader('Access-Control-Allow-Origin', 'https://app.example.com');
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    
    // Cookie attributes for cross-origin
    res.setHeader('Set-Cookie', [
      'sessionId=abc123; SameSite=None; Secure; HttpOnly',
      'preferences=theme:dark; SameSite=None; Secure'
    ]);
    
    // SameSite=None: Required for cross-origin cookies
    // Secure: Required with SameSite=None (HTTPS only)
    // HttpOnly: Prevent JavaScript access (security)
  }

  // Reading cookies in CORS request
  static async readCookies() {
    const response = await fetch('https://api.example.com/data', {
      credentials: 'include', // Required to send cookies
      headers: {
        'Content-Type': 'application/json'
      }
    });
    
    // Cookies sent automatically
    // Cannot access response cookies from JavaScript
    // (Set-Cookie not exposed by default)
    
    return response.json();
  }

  // Common cookie issues
  static cookieIssues = {
    sameSiteNone: {
      problem: 'Cookie rejected without SameSite=None',
      solution: 'Add SameSite=None; Secure attributes'
    },
    
    secure: {
      problem: 'SameSite=None requires Secure flag',
      solution: 'Use HTTPS or localhost for development'
    },
    
    partitioned: {
      problem: 'Third-party cookies blocked by browser',
      solution: 'Use CHIPS (Partitioned cookies) or alternative auth'
    }
  };
}
```

### Authentication Patterns

```javascript
// CORS authentication patterns
class CORSAuthPatterns {
  // Pattern 1: Token in Authorization header
  static async tokenAuthPattern() {
    // Client stores token in localStorage/sessionStorage
    const token = localStorage.getItem('authToken');
    
    const response = await fetch('https://api.example.com/protected', {
      credentials: 'omit', // No cookies needed
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    });
    
    return response.json();
  }

  // Pattern 2: Session cookie
  static async cookieAuthPattern() {
    // Server sets cookie on login
    const loginResponse = await fetch('https://api.example.com/login', {
      method: 'POST',
      credentials: 'include', // Store cookie
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ username: 'user', password: 'pass' })
    });
    
    // Subsequent requests automatically include cookie
    const dataResponse = await fetch('https://api.example.com/data', {
      credentials: 'include' // Send cookie
    });
    
    return dataResponse.json();
  }

  // Pattern 3: Custom header with CSRF token
  static async csrfPattern() {
    // Get CSRF token from server
    const tokenResponse = await fetch('https://api.example.com/csrf-token', {
      credentials: 'include'
    });
    const { csrfToken } = await tokenResponse.json();
    
    // Include token in subsequent requests
    const response = await fetch('https://api.example.com/action', {
      method: 'POST',
      credentials: 'include',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': csrfToken
      },
      body: JSON.stringify({ action: 'update' })
    });
    
    return response.json();
  }

  // Pattern 4: OAuth flow
  static async oauthPattern() {
    // Redirect to OAuth provider
    const authUrl = 'https://oauth.example.com/authorize' +
                    '?client_id=abc123' +
                    '&redirect_uri=https://app.example.com/callback' +
                    '&response_type=code';
    
    // After redirect and code exchange, use access token
    const accessToken = 'received_from_oauth';
    
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    });
    
    return response.json();
  }
}
```

## Security Implications

### CORS Vulnerabilities

```javascript
// Common CORS security issues
class CORSSecurityIssues {
  // Issue 1: Reflecting Origin header without validation
  static insecureOriginReflection(req, res) {
    // VULNERABLE: Reflects any origin
    const origin = req.headers.origin;
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    
    // Attacker can read sensitive data from any origin
    
    res.json({ sensitive: 'data' });
  }

  static secureOriginValidation(req, res) {
    // SECURE: Validate origin against whitelist
    const origin = req.headers.origin;
    const allowed = ['https://app.example.com', 'https://admin.example.com'];
    
    if (allowed.includes(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Access-Control-Allow-Credentials', 'true');
    } else {
      // Don't set CORS headers for untrusted origins
      res.status(403).json({ error: 'Forbidden' });
      return;
    }
    
    res.json({ sensitive: 'data' });
  }

  // Issue 2: Using null origin
  static nullOriginVulnerability(req, res) {
    // VULNERABLE: Allowing null origin
    const origin = req.headers.origin;
    
    if (origin === 'null' || this.isAllowed(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      // Attacker can set origin to null via sandbox iframe
    }
  }

  // Issue 3: Wildcard with credentials
  static wildcardWithCredentials(req, res) {
    // INVALID: Browser rejects this
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    // Error: Cannot use wildcard with credentials
  }

  // Issue 4: Insufficient validation
  static insufficientValidation(req, res) {
    // VULNERABLE: Weak pattern matching
    const origin = req.headers.origin;
    
    if (origin.includes('example.com')) {
      // Matches: https://evil-example.com (attacker site!)
      res.setHeader('Access-Control-Allow-Origin', origin);
    }
    
    // SECURE: Exact matching or proper regex
    const allowed = /^https:\/\/[\w-]+\.example\.com$/;
    if (allowed.test(origin)) {
      res.setHeader('Access-Control-Allow-Origin', origin);
    }
  }
}

// CSRF protection with CORS
class CSRFProtection {
  // Custom header approach
  static requireCustomHeader(req, res) {
    // Custom headers trigger preflight
    // Preflight verifies origin before actual request
    
    const customHeader = req.headers['x-requested-with'];
    
    if (customHeader !== 'XMLHttpRequest') {
      res.status(403).json({ error: 'Missing custom header' });
      return;
    }
    
    // Process request
    res.json({ success: true });
  }

  // CSRF token approach
  static verifyCSRFToken(req, res) {
    const token = req.headers['x-csrf-token'];
    const sessionToken = req.cookies.sessionToken;
    
    if (!this.isValidCSRFToken(token, sessionToken)) {
      res.status(403).json({ error: 'Invalid CSRF token' });
      return;
    }
    
    // Process request
    res.json({ success: true });
  }

  static isValidCSRFToken(token, sessionToken) {
    // Verify CSRF token matches session
    return token && sessionToken && token === this.generateToken(sessionToken);
  }

  static generateToken(sessionToken) {
    // Generate CSRF token from session
    return `csrf_${sessionToken}`;
  }

  // SameSite cookie approach
  static setSameSiteCookie(res) {
    // SameSite=Strict prevents CSRF
    res.setHeader('Set-Cookie', 
      'sessionId=abc123; SameSite=Strict; Secure; HttpOnly'
    );
    
    // SameSite=Strict: Never sent cross-origin
    // SameSite=Lax: Sent on top-level GET navigation
    // SameSite=None: Always sent (requires Secure)
  }
}

// Content Security Policy and CORS
class CSPandCORS {
  static setupCSP(res) {
    // CSP can restrict which origins can be fetched
    const csp = [
      "default-src 'self'",
      "connect-src 'self' https://api.example.com",
      "img-src 'self' https://cdn.example.com",
      "script-src 'self' 'unsafe-inline'"
    ].join('; ');
    
    res.setHeader('Content-Security-Policy', csp);
    
    // Note: CSP and CORS are complementary
    // CSP controls what YOUR page can fetch
    // CORS controls what OTHER pages can fetch from YOU
  }
}
```

## Common Misconceptions

### Misconception 1: "CORS is a security feature"

**Reality:** CORS is a relaxation of the Same-Origin Policy, not additional security. It allows cross-origin requests that would otherwise be blocked.

```javascript
const clarification = {
  sop: 'Blocks all cross-origin requests by default (secure)',
  cors: 'Explicitly allows specific cross-origin requests (less secure)',
  
  security: {
    real: 'CORS prevents browser from reading response',
    not: 'CORS does NOT prevent request from being sent'
  },
  
  implications: {
    stateChanging: 'POST/PUT/DELETE requests ARE sent (before preflight response)',
    serverMustValidate: 'Server must validate Origin and authenticate',
    browserEnforced: 'CORS only enforced by browsers (curl/postman ignore it)'
  }
};
```

### Misconception 2: "Preflight prevents all malicious requests"

**Reality:** Preflight checks permissions before sending, but simple requests skip preflight.

```javascript
const preflightLimitations = {
  simpleRequests: 'No preflight for GET/POST with simple content types',
  stillSent: 'Request reaches server even if CORS denied',
  csrfRisk: 'Simple POST can still trigger state changes'
};
```

### Misconception 3: "Access-Control-Allow-Origin: * is safe for public APIs"

**Reality:** While common, it has implications for credentials and should be used carefully.

```javascript
const wildcardImplications = {
  noCredentials: 'Cannot be used with credentials: include',
  publicOnly: 'Suitable only for truly public, read-only data',
  alternatives: {
    recommend: 'Use specific origins even for public APIs',
    reason: 'Defense in depth, easier to add auth later'
  }
};
```

### Misconception 4: "CORS errors mean the server is down"

**Reality:** CORS errors mean the server responded but without proper headers.

```javascript
class CORSErrorDiagnosis {
  static diagnoseError(error) {
    if (error.message.includes('CORS')) {
      return {
        problem: 'Server responded but missing CORS headers',
        notProblem: 'Server is accessible',
        check: [
          'Server logs (request was received)',
          'Response headers (CORS headers present?)',
          'Origin validation (is your origin allowed?)',
          'Preflight response (204 with correct headers?)'
        ]
      };
    }
  }
}
```

## Performance Implications

### Preflight Overhead

```javascript
// Measuring preflight impact
class PreflightPerformanceAnalysis {
  static async measurePreflightOverhead() {
    const results = {
      simple: [],
      preflight: []
    };
    
    // Measure simple request (no preflight)
    for (let i = 0; i < 10; i++) {
      const start = performance.now();
      await fetch('https://api.example.com/simple', {
        method: 'GET',
        headers: { 'Accept': 'application/json' }
      });
      results.simple.push(performance.now() - start);
    }
    
    // Measure preflighted request
    for (let i = 0; i < 10; i++) {
      // Clear cache between requests
      const start = performance.now();
      await fetch('https://api.example.com/preflight', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer token'
        },
        body: JSON.stringify({ data: 'test' })
      });
      results.preflight.push(performance.now() - start);
    }
    
    const avgSimple = results.simple.reduce((a, b) => a + b) / results.simple.length;
    const avgPreflight = results.preflight.reduce((a, b) => a + b) / results.preflight.length;
    
    return {
      simple: `${avgSimple.toFixed(2)}ms`,
      preflight: `${avgPreflight.toFixed(2)}ms`,
      overhead: `${(avgPreflight - avgSimple).toFixed(2)}ms`,
      percentage: `${((avgPreflight / avgSimple - 1) * 100).toFixed(1)}%`
    };
  }

  static calculateTheoretical() {
    return {
      simpleRequest: {
        steps: '1 RTT + processing time',
        typical: '100-300ms'
      },
      preflightRequest: {
        steps: '2 RTT + processing time',
        typical: '200-600ms',
        cached: '1 RTT + processing time (preflight cached)'
      },
      improvement: {
        firstRequest: 'Double latency',
        cachedRequests: 'Same as simple request',
        maxAge: 'Cache for 24 hours'
      }
    };
  }
}
```

### Optimization Strategies

```javascript
// Performance optimization techniques
class CORSPerformanceOptimization {
  // 1. Proxy same-origin requests
  static useProxy() {
    // Development: webpack/vite proxy
    const devProxy = {
      '/api': 'https://api.example.com'
    };
    
    // Production: Nginx reverse proxy
    const nginxConfig = `
      location /api {
        proxy_pass https://api.example.com;
        proxy_set_header Host $host;
      }
    `;
    
    // Benefit: No CORS headers needed, no preflight
  }

  // 2. Minimize custom headers
  static minimizeHeaders() {
    // Instead of:
    const manyHeaders = {
      headers: {
        'X-Custom-1': 'value1',
        'X-Custom-2': 'value2',
        'X-Custom-3': 'value3'
      }
    }; // Each header must be in Access-Control-Allow-Headers
    
    // Consider:
    const consolidatedHeaders = {
      headers: {
        'X-Metadata': JSON.stringify({
          custom1: 'value1',
          custom2: 'value2',
          custom3: 'value3'
        })
      }
    }; // Single header
  }

  // 3. Batch requests
  static batchRequests() {
    // Instead of multiple requests:
    const individual = [
      fetch('/api/user/1'),
      fetch('/api/user/2'),
      fetch('/api/user/3')
    ]; // 3 preflights (if not cached)
    
    // Use single batch request:
    const batch = fetch('/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ids: [1, 2, 3] })
    }); // 1 preflight
  }

  // 4. Connection keep-alive
  static useKeepAlive() {
    // Reuse connections for multiple requests
    const fetchWithKeepAlive = (url) => {
      return fetch(url, {
        keepalive: true // Hint to keep connection alive
      });
    };
    
    // Benefit: Amortize connection setup cost
  }

  // 5. Resource hints
  static useResourceHints() {
    // Preconnect to API domain
    const link = document.createElement('link');
    link.rel = 'preconnect';
    link.href = 'https://api.example.com';
    document.head.appendChild(link);
    
    // Benefit: DNS + TCP + TLS done in parallel with page load
  }
}
```

## Browser Differences

### CORS Implementation Variations

```javascript
const browserDifferences = {
  chrome: {
    preflightCache: 'Max 2 hours (7200s), ignores higher values',
    credentials: 'Strict SameSite=None enforcement since v80',
    errors: 'Detailed CORS error messages in console',
    quirks: [
      'Removes server push (no impact on CORS)',
      'Partitioned cookies (CHIPS) support'
    ]
  },
  
  firefox: {
    preflightCache: 'Max 24 hours (86400s)',
    credentials: 'SameSite=Lax default since v69',
    errors: 'Detailed CORS error messages',
    quirks: [
      'Can disable CORS checks (privacy.file_unique_origin=false)',
      'Total Cookie Protection affects cross-site cookies'
    ]
  },
  
  safari: {
    preflightCache: 'Conservative caching',
    credentials: 'Intelligent Tracking Prevention affects cookies',
    errors: 'Less detailed error messages',
    quirks: [
      'ITP blocks third-party cookies by default',
      'Requires user interaction for some cross-origin storage',
      'Strict same-site cookie behavior'
    ]
  },
  
  edge: {
    preflightCache: 'Follows Chromium (max 2 hours)',
    credentials: 'Same as Chrome since v79 (Chromium-based)',
    errors: 'Same as Chrome',
    quirks: ['Legacy Edge (pre-79) had different behavior']
  }
};

// Feature detection
class CORSFeatureDetection {
  static async detectFeatures() {
    return {
      credentials: this.testCredentialsMode(),
      privateNetwork: this.testPrivateNetworkAccess(),
      timing: this.testTimingAPI()
    };
  }

  static async testCredentialsMode() {
    try {
      await fetch('https://example.com', {
        credentials: 'include',
        mode: 'cors'
      });
      return 'Supported';
    } catch (e) {
      return 'Not supported or blocked';
    }
  }

  static testPrivateNetworkAccess() {
    // Check for Private Network Access (Chrome)
    return 'PrivateNetworkAccess' in window;
  }

  static testTimingAPI() {
    const entry = performance.getEntriesByType('navigation')[0];
    return 'Timing-Allow-Origin respected' in entry;
  }
}
```

## Interview Questions

### Question 1: What is the Same-Origin Policy and why does it exist?

**Answer:** The Same-Origin Policy (SOP) is a critical security mechanism that restricts how documents or scripts loaded from one origin can interact with resources from another origin. An origin is defined by protocol, domain, and port. SOP exists to prevent malicious scripts on one page from accessing sensitive data on another page (like reading your banking information through JavaScript). Without SOP, any website could make requests to your bank's API and read the response. It's the default security posture that CORS selectively relaxes.

### Question 2: Explain when a preflight request is sent and what information it contains

**Answer:** Preflight requests (OPTIONS method) are sent when a request is not "simple." A simple request must: use GET/HEAD/POST method, only include simple headers (Accept, Accept-Language, Content-Language, Content-Type with specific values), and use Content-Type of application/x-www-form-urlencoded, multipart/form-data, or text/plain. Preflight is triggered by: custom HTTP methods (PUT/DELETE), custom headers (Authorization), non-simple Content-Type (application/json). The preflight contains: Origin, Access-Control-Request-Method, and Access-Control-Request-Headers. The server responds with allowed origins, methods, headers, and caching duration (Access-Control-Max-Age).

### Question 3: Why can't you use `Access-Control-Allow-Origin: *` with credentials?

**Answer:** Allowing all origins (*) with credentials would be a serious security vulnerability. If any origin could make credentialed requests and read responses, malicious sites could read sensitive user data from authenticated APIs. The restriction forces servers to explicitly trust specific origins when credentials are involved. This prevents scenarios where an attacker's site could make requests with your cookies/auth tokens and read the response. The combination `Access-Control-Allow-Origin: *` and `Access-Control-Allow-Credentials: true` is explicitly forbidden by the CORS specification and browsers will reject it.

### Question 4: How does CORS prevent CSRF attacks, and where does it fall short?

**Answer:** CORS provides some CSRF protection through preflights - custom headers trigger preflight, and the browser verifies permissions before sending the actual request with credentials. However, simple requests (GET/POST with simple content types) don't trigger preflight, so the request still reaches the server and can cause state changes before CORS blocks reading the response. CORS alone is insufficient for CSRF protection. Additional measures needed: CSRF tokens, SameSite cookies, custom headers (X-Requested-With), or checking the Origin/Referer headers server-side. CORS primarily prevents reading responses, not sending requests.

### Question 5: What happens to cookies in cross-origin requests with different credentials modes?

**Answer:** `credentials: 'omit'` never sends cookies, even for same-origin requests. `credentials: 'same-origin'` (default) only sends cookies for same-origin requests. `credentials: 'include'` sends cookies for all requests, including cross-origin. For cross-origin cookies to work: the cookie must have `SameSite=None; Secure` attributes, the server must respond with specific origin (not *) and `Access-Control-Allow-Credentials: true`, and the browser must not block third-party cookies (tracking protection). Increasingly, browsers block third-party cookies by default, breaking many CORS credential scenarios and requiring alternative authentication methods like tokens in headers.

### Question 6: Explain the security implications of reflecting the Origin header without validation

**Answer:** Reflecting the Origin header without validation is a critical vulnerability. If your server does: `Access-Control-Allow-Origin: ${req.headers.origin}` with `Access-Control-Allow-Credentials: true`, any attacker's website can make authenticated requests and read sensitive data. The attacker simply makes a fetch request from their site, and the server allows it by reflecting their origin. Proper implementation requires: maintaining a whitelist of allowed origins, using exact matching or careful regex patterns, never reflecting null origins, and considering that browsers don't send Origin for same-origin requests. Defense requires validating the Origin header against a trusted list before reflecting it.

### Question 7: How do you optimize performance when dealing with multiple CORS requests?

**Answer:** Key strategies: 1) Maximize `Access-Control-Max-Age` (24 hours) to cache preflight responses, 2) Minimize custom headers to avoid unnecessary preflights, 3) Batch operations into single requests instead of many small requests, 4) Use simple requests when possible (GET with standard headers), 5) Implement proxying for development and optionally production to make requests same-origin, 6) Use preconnect resource hints for API domains, 7) Monitor preflight cache hit rates, 8) Consider GraphQL or similar to batch queries into single endpoint. Remember that preflight caching is per-browser and doesn't persist across sessions in some browsers, and the first request of each type still incurs double RTT.

### Question 8: What is Private Network Access and how does it relate to CORS?

**Answer:** Private Network Access (PNA) is a proposed specification that extends CORS to protect requests from public networks to private networks (localhost, private IP ranges, local network). It prevents malicious public websites from attacking devices on your local network (routers, IoT devices, local servers). PNA introduces: preflight requests for public-to-private requests even for simple requests, a new `Access-Control-Request-Private-Network: true` header, and requires explicit opt-in via `Access-Control-Allow-Private-Network: true`. This addresses the security gap where CORS didn't protect local network resources. Currently implemented in Chrome, it will affect local development servers and internal APIs accessed from public sites.

## Key Takeaways

1. **Same-Origin Policy Foundation**: SOP is the default security posture blocking cross-origin access; CORS is a controlled relaxation, not additional security. Origin is defined by protocol, domain, and port.

2. **Preflight Mechanism**: OPTIONS requests verify permissions before sending non-simple requests. Triggered by custom methods (PUT/DELETE), custom headers, or JSON content type. Cached per Access-Control-Max-Age.

3. **Origin Validation Critical**: Never reflect Origin header without validation. Use whitelist of exact origins or carefully crafted regex. Reflecting arbitrary origins with credentials is a critical security vulnerability.

4. **Credentials Complexity**: `credentials: 'include'` requires specific origin (not *), Access-Control-Allow-Credentials: true, and SameSite=None; Secure cookies. Third-party cookie blocking affects many scenarios.

5. **Simple Requests Skip Preflight**: GET/HEAD/POST with simple headers and content types avoid preflight, but this means requests reach the server before CORS can block them. CSRF protection still needed.

6. **Performance Overhead**: Preflights add one RTT, doubling latency for initial requests. Optimization strategies: maximize Max-Age, minimize custom headers, batch requests, use proxies for same-origin.

7. **Not CSRF Protection**: CORS prevents reading responses but doesn't prevent sending requests. Simple POST requests still trigger state changes. Use CSRF tokens, SameSite cookies, or custom headers.

8. **Browser Implementation Varies**: Preflight cache duration differs (Chrome: 2h max, Firefox: 24h max), cookie handling varies (Safari ITP, Chrome partitioned cookies), error messages vary in detail.

9. **Headers Expose Control**: Access-Control-Expose-Headers required to read custom response headers from JavaScript. Default only exposes Cache-Control, Content-Language, Content-Type, Expires, Last-Modified, Pragma.

10. **Future Considerations**: Private Network Access extends CORS to local networks, third-party cookie deprecation affects credentials mode, partitioned cookies (CHIPS) affect cross-site scenarios.

## Resources

### Official Specifications
- **CORS Specification**: https://fetch.spec.whatwg.org/#http-cors-protocol
- **Fetch Standard**: https://fetch.spec.whatwg.org/
- **Same-Origin Policy**: https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy

### Documentation
- **MDN CORS Guide**: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- **MDN Fetch API**: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- **Chrome Private Network Access**: https://developer.chrome.com/blog/private-network-access-preflight/

### Tools
- **CORS Tester**: https://cors-test.codehappy.dev/
- **Browser DevTools**: Network tab shows CORS errors and preflight requests
- **Postman**: CORS not enforced, useful for testing server-side

### Articles
- **Understanding CORS**: https://web.dev/cross-origin-resource-sharing/
- **CORS Security Best Practices**: https://owasp.org/www-community/vulnerabilities/CORS_Misconfiguration
- **Preflight Optimization**: https://www.html5rocks.com/en/tutorials/cors/

### Security
- **OWASP CORS Misconfigurations**: https://owasp.org/www-community/vulnerabilities/CORS_Misconfiguration
- **PortSwigger CORS Vulnerabilities**: https://portswigger.net/web-security/cors
