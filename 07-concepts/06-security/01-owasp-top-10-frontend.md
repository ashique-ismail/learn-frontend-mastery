# OWASP Top 10 for Frontend

## The Idea

**In plain English:** The OWASP Top 10 is a list of the ten most common security mistakes that attackers exploit when breaking into websites and apps. Think of it as a "most-wanted" list of dangers that every developer should know and guard against.

**Real-world analogy:** Imagine a bank publishes a report every few years listing the ten most common ways robbers have broken in — through unlocked back doors, fake uniforms, stolen master keys, and so on. Security teams study that list and patch every weakness on it before a robber shows up.

- The bank = a web application
- The published list of break-in methods = the OWASP Top 10
- The unlocked back door / fake uniform / stolen key = individual vulnerabilities (e.g., broken access control, weak passwords, injected scripts)

---

## Table of Contents
- [Introduction](#introduction)
- [A01 Broken Access Control](#a01-broken-access-control)
- [A02 Cryptographic Failures](#a02-cryptographic-failures)
- [A03 Injection](#a03-injection)
- [A04 Insecure Design](#a04-insecure-design)
- [A05 Security Misconfiguration](#a05-security-misconfiguration)
- [A06 Vulnerable Components](#a06-vulnerable-components)
- [A07 Authentication Failures](#a07-authentication-failures)
- [A08 Data Integrity Failures](#a08-data-integrity-failures)
- [A09 Security Logging Failures](#a09-security-logging-failures)
- [A10 Server-Side Request Forgery](#a10-server-side-request-forgery)
- [Frontend-Specific Vulnerabilities](#frontend-specific-vulnerabilities)
- [Security Checklist](#security-checklist)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

The **OWASP Top 10** is a standard awareness document representing a broad consensus about the most critical security risks to web applications. This guide focuses on how these vulnerabilities manifest in frontend development and how to prevent them.

### OWASP Top 10 2021 Overview

```
┌─────────────────────────────────────────────────────────────┐
│              OWASP Top 10 (2021)                             │
└─────────────────────────────────────────────────────────────┘

A01: Broken Access Control
     ↑ #5 → #1 (most common)
     
A02: Cryptographic Failures
     (formerly Sensitive Data Exposure)
     
A03: Injection
     ↓ #1 → #3 (still critical)
     
A04: Insecure Design
     NEW CATEGORY
     
A05: Security Misconfiguration
     
A06: Vulnerable and Outdated Components
     
A07: Identification and Authentication Failures
     
A08: Software and Data Integrity Failures
     NEW CATEGORY
     
A09: Security Logging and Monitoring Failures
     
A10: Server-Side Request Forgery (SSRF)
     NEW CATEGORY
```

## A01 Broken Access Control

### Frontend Access Control Issues

```typescript
// ❌ BAD: Client-side only access control
const AdminPanel: React.FC = () => {
  const { user } = useAuth();
  
  // Client-side check - easily bypassed
  if (user.role !== 'admin') {
    return <div>Access Denied</div>;
  }
  
  // Sensitive data still accessible via DevTools/API
  return (
    <div>
      <h1>Admin Panel</h1>
      <button onClick={() => deleteUser(userId)}>Delete User</button>
    </div>
  );
};

// Attacker can:
// 1. Modify user.role in browser
// 2. Call API directly: fetch('/api/admin/delete-user', {...})
// 3. Bypass UI restrictions

// ✓ GOOD: Server-side validation + frontend UI control
const AdminPanel: React.FC = () => {
  const { user } = useAuth();
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    // Server validates access
    fetch('/api/admin/dashboard', {
      headers: {
        'Authorization': `Bearer ${user.token}`
      }
    })
      .then(res => {
        if (res.status === 403) {
          throw new Error('Access denied');
        }
        return res.json();
      })
      .then(setData)
      .catch(setError);
  }, []);

  if (error) {
    return <div>Access Denied</div>;
  }

  // UI reflects server-side permissions
  return data ? <AdminDashboard data={data} /> : <Loading />;
};

// Backend must enforce access control
app.delete('/api/admin/delete-user/:id', 
  authenticateToken,
  requireAdmin,  // ← Server-side check
  async (req, res) => {
    // Double-check permissions
    if (req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Forbidden' });
    }
    
    await deleteUser(req.params.id);
    res.json({ success: true });
  }
);
```

### Path Traversal Prevention

```typescript
// Path traversal in file downloads
class FileDownloadHandler {
  // ❌ DANGEROUS: Unsanitized file path
  async downloadUnsafe(req: Request, res: Response) {
    const filename = req.query.file as string;
    // Attack: ?file=../../../etc/passwd
    const filePath = `./uploads/${filename}`;
    res.sendFile(filePath);
  }

  // ✓ SAFE: Sanitized and validated path
  async downloadSafe(req: Request, res: Response) {
    const filename = req.query.file as string;
    
    // 1. Validate filename format
    if (!/^[a-zA-Z0-9_-]+\.[a-zA-Z0-9]+$/.test(filename)) {
      return res.status(400).json({ error: 'Invalid filename' });
    }
    
    // 2. Resolve and validate path
    const uploadDir = path.resolve('./uploads');
    const requestedPath = path.resolve(uploadDir, filename);
    
    // 3. Ensure path is within uploads directory
    if (!requestedPath.startsWith(uploadDir)) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    // 4. Check file exists and user has access
    if (!await this.userCanAccessFile(req.user, filename)) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    res.sendFile(requestedPath);
  }
}
```

## A02 Cryptographic Failures

### Sensitive Data Exposure

```typescript
// Frontend cryptographic failures

// ❌ BAD: Storing sensitive data in localStorage
localStorage.setItem('creditCard', '4111111111111111');
localStorage.setItem('ssn', '123-45-6789');
localStorage.setItem('password', 'mypassword123');

// Attackers can access via:
// - XSS attacks
// - Browser extensions
// - Physical access to device

// ✓ GOOD: Never store sensitive data on client
// - Credit cards: Use tokenization (Stripe, etc.)
// - SSN: Never store client-side
// - Passwords: Never store, only transmit over HTTPS

// ✓ GOOD: Secure session management
class SecureSessionManager {
  // Access token in memory (lost on refresh)
  private accessToken: string | null = null;
  
  // Refresh token in httpOnly cookie (server-side)
  async login(email: string, password: string) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
      credentials: 'include'  // httpOnly cookie
    });
    
    const { accessToken } = await response.json();
    this.accessToken = accessToken;  // Memory only
  }
  
  // Auto-refresh before expiry
  async refreshToken() {
    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      credentials: 'include'  // Sends httpOnly cookie
    });
    
    const { accessToken } = await response.json();
    this.accessToken = accessToken;
  }
}
```

### Weak Cryptography

```typescript
// ❌ BAD: Weak hashing on client-side
const hash = btoa(password);  // Base64 is encoding, not encryption!
const md5Hash = MD5(password);  // MD5 is broken

// ❌ BAD: Client-side encryption (false security)
const encrypted = simpleCipher(data, 'hardcoded-key');
// Key is in source code, easily extracted

// ✓ GOOD: Send passwords over HTTPS only
async function login(email: string, password: string) {
  // Password sent as plain text over HTTPS (secure channel)
  await fetch('https://api.example.com/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })  // HTTPS encrypts
  });
}

// Server handles hashing with bcrypt/argon2
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  
  // Server-side hashing with strong algorithm
  const match = await bcrypt.compare(password, user.passwordHash);
  
  if (match) {
    // Issue token
  }
});
```

## A03 Injection

### Cross-Site Scripting (XSS)

```typescript
// XSS prevention in React and Angular

// React: Automatic XSS protection
const UserProfile: React.FC<{ user: User }> = ({ user }) => {
  // ✓ SAFE: React automatically escapes
  return <div>{user.name}</div>;
  
  // ❌ DANGEROUS: Bypassing React's protection
  return <div dangerouslySetInnerHTML={{ __html: user.bio }} />;
  
  // ✓ SAFE: Sanitize before using dangerouslySetInnerHTML
  const sanitizedBio = DOMPurify.sanitize(user.bio);
  return <div dangerouslySetInnerHTML={{ __html: sanitizedBio }} />;
};

// Angular: Automatic sanitization
@Component({
  template: `
    <!-- ✓ SAFE: Angular auto-sanitizes -->
    <div>{{ userName }}</div>
    
    <!-- ❌ DANGEROUS: Bypassing sanitization -->
    <div [innerHTML]="userBio"></div>
    
    <!-- ✓ SAFE: Use DomSanitizer -->
    <div [innerHTML]="sanitizedBio"></div>
  `
})
export class UserProfileComponent {
  userName = '<script>alert("XSS")</script>';  // Escaped
  userBio: string;
  sanitizedBio: SafeHtml;
  
  constructor(private sanitizer: DomSanitizer) {
    this.sanitizedBio = this.sanitizer.sanitize(1, this.userBio) || '';
  }
}
```

### DOM-Based XSS

```typescript
// DOM-based XSS vulnerabilities

// ❌ DANGEROUS: innerHTML with user input
function displayMessage(message: string) {
  document.getElementById('output')!.innerHTML = message;
  // Attack: message = '<img src=x onerror=alert("XSS")>'
}

// ✓ SAFE: textContent (auto-escapes)
function displayMessageSafe(message: string) {
  document.getElementById('output')!.textContent = message;
  // HTML tags displayed as text, not executed
}

// ❌ DANGEROUS: eval with user input
function calculate(expression: string) {
  return eval(expression);
  // Attack: expression = 'alert("XSS"); //...'
}

// ✓ SAFE: Use safe alternatives
function calculateSafe(expression: string) {
  // Use math library like math.js with safe evaluation
  return math.evaluate(expression, { allowedFunctions: ['sin', 'cos'] });
}

// ❌ DANGEROUS: Location manipulation
function redirect() {
  const url = new URLSearchParams(window.location.search).get('redirect');
  window.location.href = url!;
  // Attack: ?redirect=javascript:alert('XSS')
}

// ✓ SAFE: Validate and sanitize URLs
function redirectSafe() {
  const url = new URLSearchParams(window.location.search).get('redirect');
  
  if (!url) return;
  
  // Validate URL
  try {
    const parsed = new URL(url, window.location.origin);
    
    // Only allow same origin or whitelisted domains
    const allowedOrigins = ['https://example.com', 'https://app.example.com'];
    
    if (parsed.origin === window.location.origin || allowedOrigins.includes(parsed.origin)) {
      window.location.href = parsed.href;
    }
  } catch {
    // Invalid URL
    console.error('Invalid redirect URL');
  }
}
```

## A04 Insecure Design

### Security in Design Phase

```typescript
// Design-level security considerations

// ❌ INSECURE DESIGN: Password reset via security questions
class InsecurePasswordReset {
  async resetPassword(email: string, securityAnswer: string, newPassword: string) {
    const user = await User.findOne({ email });
    
    // Security questions are guessable
    // Mother's maiden name, pet name, etc. are public on social media
    if (user.securityAnswer === securityAnswer) {
      user.password = await hash(newPassword);
      await user.save();
    }
  }
}

// ✓ SECURE DESIGN: Token-based password reset
class SecurePasswordReset {
  async initiateReset(email: string) {
    const user = await User.findOne({ email });
    if (!user) {
      // Don't reveal if email exists
      return { message: 'If email exists, reset link sent' };
    }
    
    // Generate cryptographically secure token
    const token = crypto.randomBytes(32).toString('hex');
    const hashedToken = await hash(token);
    
    await PasswordResetToken.create({
      userId: user.id,
      token: hashedToken,
      expiresAt: new Date(Date.now() + 3600000)  // 1 hour
    });
    
    // Send via email (secure channel)
    await sendEmail(user.email, `Reset link: https://app.com/reset?token=${token}`);
    
    return { message: 'If email exists, reset link sent' };
  }
  
  async resetPassword(token: string, newPassword: string) {
    const hashedToken = await hash(token);
    const resetToken = await PasswordResetToken.findOne({
      token: hashedToken,
      expiresAt: { $gt: new Date() },
      used: false
    });
    
    if (!resetToken) {
      throw new Error('Invalid or expired token');
    }
    
    const user = await User.findById(resetToken.userId);
    user.password = await hash(newPassword);
    await user.save();
    
    // Mark token as used
    resetToken.used = true;
    await resetToken.save();
    
    // Invalidate all sessions
    await Session.deleteMany({ userId: user.id });
  }
}

// ❌ INSECURE DESIGN: Predictable account IDs in URLs
// https://example.com/profile/123
// Attacker can enumerate: /profile/1, /profile/2, /profile/3...

// ✓ SECURE DESIGN: Non-enumerable UUIDs
// https://example.com/profile/550e8400-e29b-41d4-a716-446655440000
// Cannot guess other user IDs
```

## A05 Security Misconfiguration

### Common Misconfigurations

```typescript
// Frontend security misconfigurations

// ❌ BAD: Debug mode in production
if (process.env.NODE_ENV === 'development') {
  console.log('API Key:', apiKey);
  console.log('User data:', userData);
}
// If env var not set, defaults to development!

// ✓ GOOD: Explicitly check production
const isProduction = process.env.NODE_ENV === 'production';

if (!isProduction) {
  console.log('Debug info');
}

// Never log sensitive data even in development

// ❌ BAD: Exposing stack traces
app.use((err, req, res, next) => {
  res.status(500).json({
    error: err.message,
    stack: err.stack  // Reveals server structure!
  });
});

// ✓ GOOD: Generic error messages in production
app.use((err, req, res, next) => {
  console.error(err);  // Log server-side
  
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    res.status(500).json({
      error: err.message,
      stack: err.stack
    });
  }
});

// ❌ BAD: Default credentials
const config = {
  adminUser: 'admin',
  adminPassword: 'admin123'
};

// ✓ GOOD: Strong, unique credentials
const config = {
  adminUser: process.env.ADMIN_USER,
  adminPassword: process.env.ADMIN_PASSWORD
};

if (!config.adminPassword || config.adminPassword.length < 16) {
  throw new Error('Strong admin password required');
}

// ❌ BAD: Missing security headers
app.get('/', (req, res) => {
  res.send('<h1>Hello</h1>');
});

// ✓ GOOD: Comprehensive security headers
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'strict-dynamic'"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));
```

## A06 Vulnerable Components

### Dependency Management

```typescript
// Managing dependencies securely

// 1. Regular audits
// package.json scripts
{
  "scripts": {
    "audit": "npm audit",
    "audit:fix": "npm audit fix",
    "check:updates": "npm outdated"
  }
}

// 2. Automated dependency updates
// Use Dependabot, Renovate, or Snyk

// 3. Verify integrity
{
  "dependencies": {
    "react": "18.2.0"  // Pin versions
  }
}

// 4. Subresource Integrity for CDN resources
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous"
></script>

// Implementation
class DependencySecurityManager {
  async checkVulnerabilities(): Promise<VulnReport> {
    // Run npm audit programmatically
    const { execSync } = require('child_process');
    const auditOutput = execSync('npm audit --json').toString();
    const auditData = JSON.parse(auditOutput);
    
    // Parse vulnerabilities
    const vulnerabilities = {
      critical: auditData.metadata.vulnerabilities.critical || 0,
      high: auditData.metadata.vulnerabilities.high || 0,
      moderate: auditData.metadata.vulnerabilities.moderate || 0,
      low: auditData.metadata.vulnerabilities.low || 0
    };
    
    // Alert if critical vulnerabilities
    if (vulnerabilities.critical > 0 || vulnerabilities.high > 0) {
      await this.alertSecurityTeam(vulnerabilities);
    }
    
    return vulnerabilities;
  }
  
  async updateDependencies(): Promise<void> {
    // Check for outdated packages
    const outdated = execSync('npm outdated --json').toString();
    const packages = JSON.parse(outdated);
    
    // Update non-breaking (patch/minor)
    for (const [name, info] of Object.entries(packages)) {
      if (this.isSafeUpdate(info)) {
        execSync(`npm install ${name}@${info.latest}`);
      }
    }
  }
  
  private isSafeUpdate(info: any): boolean {
    const current = info.current.split('.');
    const latest = info.latest.split('.');
    
    // Only auto-update patch versions
    return current[0] === latest[0] && current[1] === latest[1];
  }
}

// CI/CD integration
// .github/workflows/security.yml
/*
name: Security Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run npm audit
        run: npm audit --audit-level=high
      - name: Check for vulnerabilities
        run: npm run audit
*/
```

## A07 Authentication Failures

### Weak Authentication

```typescript
// Authentication security issues

// ❌ BAD: Weak password requirements
function validatePassword(password: string): boolean {
  return password.length >= 6;  // Too weak!
}

// ✓ GOOD: Strong password requirements
function validatePasswordStrong(password: string): boolean {
  const requirements = {
    minLength: 12,
    hasUppercase: /[A-Z]/.test(password),
    hasLowercase: /[a-z]/.test(password),
    hasNumber: /\d/.test(password),
    hasSpecial: /[!@#$%^&*(),.?":{}|<>]/.test(password)
  };
  
  return (
    password.length >= requirements.minLength &&
    requirements.hasUppercase &&
    requirements.hasLowercase &&
    requirements.hasNumber &&
    requirements.hasSpecial
  );
}

// ❌ BAD: No rate limiting on login
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  
  if (user && await user.comparePassword(password)) {
    // Login successful
  } else {
    res.status(401).json({ error: 'Invalid credentials' });
  }
  // Attacker can brute force unlimited attempts
});

// ✓ GOOD: Rate limiting and account lockout
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,  // 5 attempts
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false
});

app.post('/login', loginLimiter, async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  
  if (!user) {
    // Don't reveal if email exists
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // Check if account is locked
  if (user.lockedUntil && user.lockedUntil > new Date()) {
    return res.status(423).json({
      error: 'Account locked due to multiple failed attempts'
    });
  }
  
  if (await user.comparePassword(password)) {
    // Reset failed attempts on successful login
    user.failedLoginAttempts = 0;
    user.lockedUntil = null;
    await user.save();
    
    // Issue token
    const token = generateToken(user);
    res.json({ token });
  } else {
    // Increment failed attempts
    user.failedLoginAttempts = (user.failedLoginAttempts || 0) + 1;
    
    // Lock account after 5 failures
    if (user.failedLoginAttempts >= 5) {
      user.lockedUntil = new Date(Date.now() + 30 * 60 * 1000);  // 30 min
    }
    
    await user.save();
    res.status(401).json({ error: 'Invalid credentials' });
  }
});

// Multi-Factor Authentication
class MFAImplementation {
  async setupMFA(userId: string): Promise<{ secret: string; qrCode: string }> {
    const speakeasy = require('speakeasy');
    const qrcode = require('qrcode');
    
    const secret = speakeasy.generateSecret({
      name: `MyApp (${userId})`,
      issuer: 'MyApp'
    });
    
    const qrCode = await qrcode.toDataURL(secret.otpauth_url);
    
    // Store secret (encrypted) in database
    await User.updateOne(
      { _id: userId },
      { mfaSecret: encrypt(secret.base32) }
    );
    
    return {
      secret: secret.base32,
      qrCode
    };
  }
  
  async verifyMFA(userId: string, token: string): Promise<boolean> {
    const speakeasy = require('speakeasy');
    const user = await User.findById(userId);
    
    if (!user.mfaSecret) {
      throw new Error('MFA not enabled');
    }
    
    const secret = decrypt(user.mfaSecret);
    
    return speakeasy.totp.verify({
      secret,
      encoding: 'base32',
      token,
      window: 1  // Allow 1 time-step tolerance
    });
  }
}
```

## A08 Data Integrity Failures

### Subresource Integrity

```typescript
// Ensuring integrity of external resources

// ❌ BAD: CDN script without integrity check
<script src="https://cdn.example.com/library.js"></script>
// If CDN compromised, malicious code executed

// ✓ GOOD: SRI (Subresource Integrity)
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous"
></script>
// Browser verifies hash before executing

// Generate SRI hash
import crypto from 'crypto';
import fs from 'fs';

function generateSRIHash(filePath: string): string {
  const fileContent = fs.readFileSync(filePath);
  const hash = crypto.createHash('sha384').update(fileContent).digest('base64');
  return `sha384-${hash}`;
}

// Webpack plugin for automatic SRI
class SRIPlugin {
  apply(compiler: any) {
    compiler.hooks.compilation.tap('SRIPlugin', (compilation: any) => {
      compilation.hooks.processAssets.tap('SRIPlugin', (assets: any) => {
        Object.keys(assets).forEach((filename) => {
          if (filename.endsWith('.js') || filename.endsWith('.css')) {
            const source = assets[filename].source();
            const hash = crypto.createHash('sha384').update(source).digest('base64');
            
            // Store hash for HTML injection
            this.hashes[filename] = `sha384-${hash}`;
          }
        });
      });
    });
  }
}
```

### Unsigned Code Execution

```typescript
// Preventing tampering with frontend code

// React: Code splitting with integrity checks
import React, { lazy, Suspense } from 'react';

// Lazy-loaded components
const AdminPanel = lazy(() => import('./AdminPanel'));

// Verify code integrity on load
async function verifyComponentIntegrity(componentPath: string): Promise<boolean> {
  // In production, verify component hash
  if (process.env.NODE_ENV === 'production') {
    const componentCode = await fetch(componentPath).then(r => r.text());
    const hash = await crypto.subtle.digest(
      'SHA-256',
      new TextEncoder().encode(componentCode)
    );
    
    const expectedHash = await fetch('/api/component-hashes').then(r => r.json());
    
    return compareHashes(hash, expectedHash[componentPath]);
  }
  
  return true;
}

// Service Worker integrity checks
self.addEventListener('fetch', (event) => {
  event.respondWith(
    (async () => {
      const response = await fetch(event.request);
      
      // Verify integrity of JavaScript files
      if (event.request.url.endsWith('.js')) {
        const clone = response.clone();
        const content = await clone.text();
        const hash = await hashContent(content);
        
        // Compare with known good hashes
        if (!await verifyHash(event.request.url, hash)) {
          console.error('Integrity check failed for', event.request.url);
          throw new Error('Integrity check failed');
        }
      }
      
      return response;
    })()
  );
});
```

## A09 Security Logging Failures

### Frontend Logging

```typescript
// Security event logging

class SecurityLogger {
  private static endpoint = '/api/security-events';
  
  // Log security events
  static async logEvent(event: SecurityEvent): Promise<void> {
    const logEntry = {
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
      url: window.location.href,
      referrer: document.referrer,
      ...event
    };
    
    // Send to server (non-blocking)
    fetch(this.endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(logEntry),
      keepalive: true  // Ensure delivery even if page unloads
    }).catch(err => {
      console.error('Failed to log security event:', err);
    });
  }
  
  // Failed login attempt
  static logFailedLogin(email: string): void {
    this.logEvent({
      type: 'FAILED_LOGIN',
      severity: 'WARNING',
      details: { email }
    });
  }
  
  // XSS attempt detected
  static logXSSAttempt(input: string): void {
    this.logEvent({
      type: 'XSS_ATTEMPT',
      severity: 'HIGH',
      details: { input }
    });
  }
  
  // CSRF token validation failed
  static logCSRFFailure(): void {
    this.logEvent({
      type: 'CSRF_FAILURE',
      severity: 'HIGH',
      details: {}
    });
  }
  
  // Unexpected API error
  static logAPIError(endpoint: string, status: number): void {
    this.logEvent({
      type: 'API_ERROR',
      severity: status >= 500 ? 'HIGH' : 'MEDIUM',
      details: { endpoint, status }
    });
  }
  
  // Permission denied
  static logAccessDenied(resource: string): void {
    this.logEvent({
      type: 'ACCESS_DENIED',
      severity: 'MEDIUM',
      details: { resource }
    });
  }
}

// Usage in React
const LoginForm: React.FC = () => {
  const handleSubmit = async (email: string, password: string) => {
    try {
      const response = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify({ email, password })
      });
      
      if (response.status === 401) {
        SecurityLogger.logFailedLogin(email);
        alert('Invalid credentials');
      }
    } catch (error) {
      SecurityLogger.logAPIError('/api/login', 500);
    }
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
};

// Backend: Security event processing
app.post('/api/security-events', async (req, res) => {
  const event = req.body;
  
  // Store in database
  await SecurityEvent.create({
    ...event,
    ip: req.ip,
    userId: req.user?.id
  });
  
  // Alert on high-severity events
  if (event.severity === 'HIGH' || event.severity === 'CRITICAL') {
    await alertSecurityTeam(event);
  }
  
  // Check for patterns (brute force, etc.)
  await checkForAttackPatterns(event);
  
  res.status(204).send();
});
```

## A10 Server-Side Request Forgery

### SSRF Prevention

```typescript
// SSRF vulnerabilities in frontend applications

// ❌ DANGEROUS: User-controlled URL fetching
app.get('/api/fetch-url', async (req, res) => {
  const url = req.query.url as string;
  
  // Attacker can access internal resources
  // ?url=http://localhost:6379/  (Redis)
  // ?url=http://169.254.169.254/latest/meta-data/  (AWS metadata)
  
  const response = await fetch(url);
  const data = await response.text();
  res.send(data);
});

// ✓ SAFE: Whitelist allowed domains
class SSRFProtection {
  private allowedDomains = [
    'api.example.com',
    'cdn.example.com'
  ];
  
  async fetchURL(url: string): Promise<Response> {
    // 1. Validate URL format
    let parsed: URL;
    try {
      parsed = new URL(url);
    } catch {
      throw new Error('Invalid URL');
    }
    
    // 2. Check protocol (HTTPS only)
    if (parsed.protocol !== 'https:') {
      throw new Error('Only HTTPS allowed');
    }
    
    // 3. Check domain whitelist
    if (!this.allowedDomains.includes(parsed.hostname)) {
      throw new Error('Domain not allowed');
    }
    
    // 4. Prevent redirects to internal resources
    const response = await fetch(url, {
      redirect: 'manual'  // Don't follow redirects
    });
    
    if (response.status >= 300 && response.status < 400) {
      throw new Error('Redirects not allowed');
    }
    
    return response;
  }
  
  // Prevent DNS rebinding attacks
  async validateIPAddress(hostname: string): Promise<boolean> {
    const dns = require('dns').promises;
    const addresses = await dns.resolve4(hostname);
    
    // Block private IP ranges
    const privateRanges = [
      /^10\./,
      /^172\.(1[6-9]|2[0-9]|3[0-1])\./,
      /^192\.168\./,
      /^127\./,
      /^169\.254\./,  // Link-local
      /^::1$/,  // IPv6 localhost
      /^fc00:/,  // IPv6 private
      /^fe80:/   // IPv6 link-local
    ];
    
    for (const addr of addresses) {
      for (const range of privateRanges) {
        if (range.test(addr)) {
          throw new Error('Private IP addresses not allowed');
        }
      }
    }
    
    return true;
  }
}
```

## Frontend-Specific Vulnerabilities

### Prototype Pollution

```typescript
// Prototype pollution attacks

// ❌ VULNERABLE: Unsafe object merge
function merge(target: any, source: any) {
  for (const key in source) {
    target[key] = source[key];
  }
  return target;
}

// Attack
const maliciousInput = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, maliciousInput);

// Now ALL objects have isAdmin: true
console.log({}.isAdmin);  // true!

// ✓ SAFE: Prevent prototype pollution
function safeMerge(target: any, source: any) {
  for (const key in source) {
    if (Object.prototype.hasOwnProperty.call(source, key)) {
      // Prevent prototype pollution
      if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
        continue;
      }
      
      target[key] = source[key];
    }
  }
  return target;
}

// ✓ BETTER: Use Object.assign or spread operator
const merged = { ...obj1, ...obj2 };
// Or: const merged = Object.assign({}, obj1, obj2);

// ✓ BEST: Use Map for user-controlled keys
const safeMap = new Map();
safeMap.set(userKey, userValue);  // No prototype pollution
```

### Click jacking

```typescript
// Clickjacking prevention

// Server-side: X-Frame-Options header
app.use((req, res, next) => {
  res.setHeader('X-Frame-Options', 'DENY');
  // Or: 'SAMEORIGIN' to allow same-origin framing
  next();
});

// Or use CSP frame-ancestors
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "frame-ancestors 'none'"  // More flexible than X-Frame-Options
  );
  next();
});

// Client-side: Frame-busting (defense in depth)
if (window.top !== window.self) {
  // Page is in an iframe
  window.top.location = window.self.location;
}

// Modern approach: Check if framed
if (window !== window.parent) {
  document.body.innerHTML = 'This page cannot be displayed in a frame';
}
```

## Security Checklist

### Pre-Deployment Security Checklist

```typescript
// Automated security checklist
class SecurityChecklist {
  async runChecks(): Promise<ChecklistResults> {
    const results = {
      https: await this.checkHTTPS(),
      headers: await this.checkSecurityHeaders(),
      dependencies: await this.checkDependencies(),
      authentication: await this.checkAuthentication(),
      authorization: await this.checkAuthorization(),
      inputValidation: await this.checkInputValidation(),
      encryption: await this.checkEncryption(),
      logging: await this.checkLogging(),
      errorHandling: await this.checkErrorHandling(),
      xss: await this.checkXSSProtection()
    };
    
    return results;
  }
  
  private async checkHTTPS(): Promise<CheckResult> {
    // Verify HTTPS is enforced
    return {
      passed: process.env.FORCE_HTTPS === 'true',
      message: 'HTTPS enforcement',
      severity: 'CRITICAL'
    };
  }
  
  private async checkSecurityHeaders(): Promise<CheckResult> {
    const requiredHeaders = [
      'Strict-Transport-Security',
      'Content-Security-Policy',
      'X-Content-Type-Options',
      'X-Frame-Options'
    ];
    
    // Test if headers are set
    const response = await fetch('https://example.com');
    const missingHeaders = requiredHeaders.filter(
      header => !response.headers.has(header)
    );
    
    return {
      passed: missingHeaders.length === 0,
      message: `Missing headers: ${missingHeaders.join(', ')}`,
      severity: 'HIGH'
    };
  }
  
  private async checkDependencies(): Promise<CheckResult> {
    // Run npm audit
    const { execSync } = require('child_process');
    try {
      execSync('npm audit --audit-level=high');
      return {
        passed: true,
        message: 'No high-severity vulnerabilities',
        severity: 'HIGH'
      };
    } catch {
      return {
        passed: false,
        message: 'Vulnerabilities found in dependencies',
        severity: 'HIGH'
      };
    }
  }
}

// CI/CD integration
/*
// .github/workflows/security-checklist.yml
name: Security Checklist
on: [push, pull_request]
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run security checklist
        run: npm run security:check
      - name: Fail if critical issues
        run: npm run security:verify
*/
```

## Common Mistakes

### 1. Trusting Client-Side Validation

```typescript
// ❌ BAD: Only client-side validation
if (age < 18) {
  alert('Must be 18+');
  return;
}
// Easily bypassed

// ✓ GOOD: Server-side validation
app.post('/register', [
  body('age').isInt({ min: 18, max: 120 })
], (req, res) => {
  // Server validates
});
```

### 2. Storing Sensitive Data in localStorage

```typescript
// ❌ BAD
localStorage.setItem('creditCard', card);

// ✓ GOOD: Never store sensitive data client-side
// Use tokenization or server-side storage
```

### 3. Not Sanitizing User Input

```typescript
// ❌ BAD
document.innerHTML = userInput;

// ✓ GOOD
document.textContent = userInput;
// Or use DOMPurify.sanitize()
```

### 4. Ignoring Dependency Vulnerabilities

```typescript
// ❌ BAD: Never updating dependencies

// ✓ GOOD: Regular audits and updates
// npm audit
// npm audit fix
// Set up Dependabot or Renovate
```

### 5. Weak Error Handling

```typescript
// ❌ BAD: Exposing stack traces
catch (error) {
  res.json({ error: error.stack });
}

// ✓ GOOD: Generic messages
catch (error) {
  console.error(error);
  res.status(500).json({ error: 'An error occurred' });
}
```

## Best Practices

### 1. Defense in Depth

```typescript
// Multiple security layers
class DefenseInDepth {
  // Layer 1: Input validation
  validateInput(data: any): boolean {
    // Whitelist validation
  }
  
  // Layer 2: Sanitization
  sanitizeInput(data: string): string {
    return DOMPurify.sanitize(data);
  }
  
  // Layer 3: Output encoding
  encodeOutput(data: string): string {
    return encodeHTML(data);
  }
  
  // Layer 4: CSP headers
  // Layer 5: Rate limiting
  // Layer 6: Monitoring and logging
}
```

### 2. Principle of Least Privilege

```typescript
// Grant minimum necessary permissions
const permissions = {
  user: ['read:own-profile', 'update:own-profile'],
  admin: ['read:all-profiles', 'update:all-profiles', 'delete:profiles']
};

// Check permissions for every action
if (!hasPermission(user, 'delete:profiles')) {
  throw new Error('Insufficient permissions');
}
```

### 3. Secure by Default

```typescript
// Default to secure settings
const secureDefaults = {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 3600000
};

// Explicitly opt-out if needed (with justification)
const cookieOptions = {
  ...secureDefaults,
  httpOnly: false  // Required for client-side access (documented reason)
};
```

## Interview Questions

### Q1: Explain the OWASP Top 10 and why it's important.

**Answer:** OWASP Top 10 is a standard awareness document listing the most critical web application security risks, updated every 3-4 years based on data from security organizations worldwide.

**Why important:**
- **Industry standard:** Recognized globally as security baseline
- **Risk prioritization:** Focuses on most common/critical threats
- **Compliance:** Many regulations reference OWASP Top 10
- **Education:** Teaches developers about security threats
- **Measurable:** Provides framework for security assessment

**2021 Top 3:**
1. **Broken Access Control** (most common): Unauthorized access to resources
2. **Cryptographic Failures**: Sensitive data exposure
3. **Injection**: SQL injection, XSS, command injection

**Best practice:** Address all Top 10 categories in security strategy, prioritizing based on application-specific risks.

### Q2: How does Broken Access Control manifest in frontend applications?

**Answer:** Broken Access Control occurs when users can access resources they shouldn't.

**Frontend manifestations:**

**1. Client-side only checks:**
```typescript
// ❌ Attacker can bypass
if (user.role !== 'admin') return <AccessDenied />;

// ✓ Server must validate
app.get('/admin/data', requireAdmin, (req, res) => {...});
```

**2. Predictable resource IDs:**
```
/api/user/123  // Can enumerate: 124, 125, 126...
```
Use UUIDs instead.

**3. Path traversal:**
```
/api/file?path=../../etc/passwd
```
Validate and sanitize file paths.

**4. Insecure Direct Object References:**
```typescript
// Attacker changes userID in request
fetch('/api/profile/456')  // Access someone else's profile
```
Server must verify ownership.

**Prevention:**
- Server-side access control (never client-side only)
- Non-enumerable identifiers (UUIDs)
- Validate user owns requested resource
- Default deny (whitelist permissions)
- Audit access control logic

### Q3: What are cryptographic failures and how do they affect frontend security?

**Answer:** Cryptographic Failures involve improper use of encryption, hashing, or insecure data transmission.

**Common frontend failures:**

**1. Storing sensitive data in localStorage:**
```typescript
// ❌ Vulnerable to XSS
localStorage.setItem('creditCard', '4111...');

// ✓ Never store sensitive data client-side
// Use tokenization (Stripe) or server-side storage
```

**2. Transmitting data over HTTP:**
```typescript
// ❌ Plaintext transmission
fetch('http://api.example.com/login', {...})

// ✓ Always HTTPS
fetch('https://api.example.com/login', {...})
```

**3. Weak client-side encryption:**
```typescript
// ❌ False sense of security
const encrypted = btoa(password);  // Base64 is encoding, not encryption!

// ✓ Send over HTTPS, let server handle encryption
```

**4. Hardcoded secrets:**
```typescript
// ❌ Secrets in source code
const apiKey = 'sk_live_abc123...';

// ✓ Environment variables, server-side secrets
```

**Prevention:**
- HTTPS everywhere
- Never store sensitive data client-side
- Use httpOnly cookies for tokens
- Let server handle cryptography
- Never hardcode secrets

### Q4: How do you prevent injection attacks in modern frontend frameworks?

**Answer:** Modern frameworks provide built-in protections, but developers must use them correctly.

**React XSS prevention:**
```typescript
// ✓ Automatic escaping
<div>{userInput}</div>

// ❌ Bypassing protection
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✓ Safe with sanitization
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

**Angular sanitization:**
```typescript
// ✓ Automatic sanitization
<div>{{ userInput }}</div>

// ❌ Bypassing sanitization
<div [innerHTML]="userInput"></div>

// ✓ Use DomSanitizer
sanitizedInput = this.sanitizer.sanitize(SecurityContext.HTML, userInput);
```

**SQL injection (backend):**
```typescript
// ❌ String concatenation
db.query(`SELECT * FROM users WHERE id = ${userId}`)

// ✓ Parameterized queries
db.query('SELECT * FROM users WHERE id = $1', [userId])
```

**Key principles:**
- Use framework built-in protections
- Sanitize when bypassing protections
- Parameterized queries for databases
- Validate and whitelist input
- Context-aware output encoding

### Q5: What is Insecure Design and how does it differ from implementation bugs?

**Answer:** Insecure Design is a flaw in the application's architecture/design, not a coding bug.

**Difference:**

**Implementation bug:** Code doesn't correctly implement secure design
```typescript
// Bug: Missing validation
if (user.password == password) { ... }  // Should use crypto.timingSafeEqual
```

**Insecure design:** Fundamentally flawed approach
```typescript
// Design flaw: Security questions for password reset
// Problem: Questions are guessable (social media)
// Solution: Use cryptographic tokens instead
```

**Examples of insecure design:**

**1. Account enumeration:**
```typescript
// ❌ Design flaw
if (!userExists) return 'User not found';
if (!passwordMatch) return 'Invalid password';
// Reveals which emails are registered

// ✓ Secure design
return 'Invalid email or password';  // Generic message
```

**2. Rate limiting on result, not attempt:**
```typescript
// ❌ Design flaw
if (failedAttempts > 5 && lastAttempt > 1hour ago) { lock(); }
// Attacker can try 5 passwords every hour forever

// ✓ Secure design
if (totalFailedAttempts > 10) { permanentLock(); }
```

**Prevention:**
- Threat modeling during design
- Security requirements in specifications
- Peer review of architecture
- Learn from past incidents
- Security patterns and anti-patterns

### Q6: How do you secure third-party dependencies?

**Answer:** Multi-layered approach to dependency security:

**1. Audit regularly:**
```bash
npm audit
npm audit fix
```

**2. Automated scanning:**
- Dependabot (GitHub)
- Snyk
- npm audit in CI/CD

**3. Subresource Integrity for CDN:**
```html
<script 
  src="https://cdn.com/lib.js"
  integrity="sha384-hash"
  crossorigin="anonymous">
</script>
```

**4. Pin versions:**
```json
{
  "dependencies": {
    "react": "18.2.0"  // Exact version, not ^18.2.0
  }
}
```

**5. Review dependencies:**
- Check maintainership
- Review code for suspicious patterns
- Prefer popular, well-maintained packages
- Minimize dependency count

**6. Keep updated:**
```bash
npm outdated
npm update
```

**7. Monitor security advisories:**
- Subscribe to security mailing lists
- Enable GitHub security alerts

**Best practice:** Automate audits in CI/CD, fail builds on high-severity vulnerabilities.

### Q7: What is prototype pollution and how do you prevent it?

**Answer:** Prototype pollution modifies JavaScript object prototypes, affecting all objects.

**Attack example:**
```typescript
const malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
Object.assign({}, malicious);

// Now ALL objects have isAdmin
console.log({}.isAdmin);  // true!
```

**How it works:**
- JavaScript objects inherit from prototypes
- Modifying Object.prototype affects all objects
- Attacker injects `__proto__`, `constructor`, or `prototype` keys

**Prevention:**

**1. Avoid unsafe merges:**
```typescript
// ❌ Vulnerable
for (let key in source) {
  target[key] = source[key];
}

// ✓ Check for prototype keys
if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
  continue;
}
```

**2. Use safe methods:**
```typescript
// ✓ Safe
const merged = { ...obj1, ...obj2 };
const merged = Object.assign({}, obj1, obj2);
```

**3. Use Map for user keys:**
```typescript
// ✓ No prototype pollution
const map = new Map();
map.set(userKey, value);
```

**4. Create null-prototype objects:**
```typescript
const obj = Object.create(null);  // No prototype
obj.__proto__ = 'malicious';  // Just a regular property
```

**5. Freeze prototypes:**
```typescript
Object.freeze(Object.prototype);
```

### Q8: How do you implement secure logging for security events?

**Answer:** Comprehensive security logging strategy:

**What to log:**
- Authentication events (login, logout, failures)
- Authorization failures
- Input validation failures
- Security exceptions (XSS attempts, CSRF failures)
- Configuration changes
- Data access (especially sensitive data)

**What NOT to log:**
- Passwords (ever!)
- Credit cards
- SSNs
- Session tokens
- Private keys

**Implementation:**
```typescript
class SecurityLogger {
  static logSecurityEvent(event: SecurityEvent) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      type: event.type,
      severity: event.severity,
      userId: event.userId,
      ip: event.ip,
      userAgent: event.userAgent,
      details: this.sanitizeDetails(event.details)  // Remove sensitive data
    };
    
    // Send to logging service
    this.sendToLogService(logEntry);
    
    // Alert on high-severity
    if (event.severity === 'CRITICAL') {
      this.alertSecurityTeam(logEntry);
    }
  }
  
  private static sanitizeDetails(details: any): any {
    const sanitized = { ...details };
    
    // Remove sensitive fields
    delete sanitized.password;
    delete sanitized.creditCard;
    delete sanitized.token;
    
    return sanitized;
  }
}
```

**Log analysis:**
- Monitor for patterns (brute force, enumeration)
- Alert on anomalies
- Regular audit of logs
- Retain logs per compliance requirements

**Best practice:** Centralized logging, tamper-proof storage, real-time alerting.

## Key Takeaways

1. **Server-side validation is mandatory** - Client-side checks are UX only; never trust client for security decisions
2. **Never store sensitive data client-side** - No passwords, credit cards, or PII in localStorage/sessionStorage
3. **Use framework built-in protections** - React/Angular auto-escape by default; use DOMPurify when bypassing
4. **Audit dependencies regularly** - npm audit, Dependabot, Snyk for vulnerable package detection
5. **Implement defense in depth** - Multiple security layers: validation, sanitization, encoding, CSP, monitoring
6. **Rate limiting is essential** - Prevent brute force on login, API endpoints; implement account lockout
7. **Security logging for forensics** - Log authentication events, authorization failures, security exceptions
8. **HTTPS everywhere in production** - No exceptions; use HSTS to enforce; upgrade-insecure-requests CSP
9. **Secure by default configuration** - httpOnly cookies, strict CSP, frame protection, security headers
10. **Regular security testing** - Automated scans, penetration testing, code review, security checklists

## Resources

### Official Documentation
- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)

### Tools
- [OWASP ZAP](https://www.zaproxy.org/) - Security testing tool
- [Burp Suite](https://portswigger.net/burp) - Web security testing
- [npm audit](https://docs.npmjs.com/cli/v8/commands/npm-audit) - Dependency scanning
- [Snyk](https://snyk.io/) - Dependency vulnerability scanning

### Learning Resources
- [OWASP WebGoat](https://owasp.org/www-project-webgoat/) - Vulnerable app for learning
- [PortSwigger Academy](https://portswigger.net/web-security) - Free web security training
- [HackTheBox](https://www.hackthebox.com/) - Penetration testing practice

### Standards and Compliance
- [PCI DSS](https://www.pcisecuritystandards.org/) - Payment card security
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [ISO 27001](https://www.iso.org/isoiec-27001-information-security.html) - Information security management
