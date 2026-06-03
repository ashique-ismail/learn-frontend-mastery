# Secure Coding Practices

## Table of Contents
- [Introduction](#introduction)
- [Principle of Least Privilege](#principle-of-least-privilege)
- [Defense in Depth](#defense-in-depth)
- [Secure Defaults](#secure-defaults)
- [Input Validation Patterns](#input-validation-patterns)
- [Output Encoding](#output-encoding)
- [Security Code Reviews](#security-code-reviews)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Secure coding practices** are guidelines and patterns that help developers write code resistant to security vulnerabilities. These practices form the foundation of application security, preventing common vulnerabilities before they're introduced.

### Security Mindset

```
┌─────────────────────────────────────────────────────────────┐
│           Shift-Left Security Approach                       │
└─────────────────────────────────────────────────────────────┘

Traditional Approach (Security as Afterthought):
  Design → Code → Test → [Security Review] → Deploy
                           ↑ Too Late!

Secure Development Lifecycle:
  [Security] Requirements → [Security] Design → 
  [Security] Code → [Security] Test → [Security] Deploy
  ↑ Security integrated at every phase

Cost of Fixing Vulnerabilities:
  Design Phase:    $1
  Development:     $10
  Testing:         $100
  Production:      $1000+
  
  ⚡ Key Takeaway: Build security in, don't bolt it on!
```

## Principle of Least Privilege

### Access Control Implementation

```typescript
// Least Privilege: Grant minimum necessary permissions

// ❌ BAD: Over-privileged user roles
interface User {
  id: string;
  role: 'admin' | 'user';  // Only two roles
}

function canDeleteUser(user: User): boolean {
  return user.role === 'admin';  // Admin can do everything
}

// ✓ GOOD: Fine-grained permissions
interface User {
  id: string;
  permissions: Set<Permission>;
}

enum Permission {
  READ_OWN_PROFILE = 'read:own-profile',
  UPDATE_OWN_PROFILE = 'update:own-profile',
  READ_ALL_PROFILES = 'read:all-profiles',
  UPDATE_ALL_PROFILES = 'update:all-profiles',
  DELETE_PROFILES = 'delete:profiles',
  MANAGE_ROLES = 'manage:roles'
}

class PermissionChecker {
  hasPermission(user: User, permission: Permission): boolean {
    return user.permissions.has(permission);
  }

  requirePermission(user: User, permission: Permission): void {
    if (!this.hasPermission(user, permission)) {
      throw new ForbiddenError(
        `User lacks permission: ${permission}`
      );
    }
  }

  requireAnyPermission(
    user: User, 
    permissions: Permission[]
  ): void {
    const hasAny = permissions.some(p => 
      this.hasPermission(user, p)
    );
    
    if (!hasAny) {
      throw new ForbiddenError(
        `User lacks any of: ${permissions.join(', ')}`
      );
    }
  }

  requireAllPermissions(
    user: User,
    permissions: Permission[]
  ): void {
    const hasAll = permissions.every(p => 
      this.hasPermission(user, p)
    );
    
    if (!hasAll) {
      throw new ForbiddenError(
        `User lacks all of: ${permissions.join(', ')}`
      );
    }
  }
}

// Usage
class UserService {
  constructor(private permissionChecker: PermissionChecker) {}

  async getProfile(requestingUser: User, targetUserId: string): Promise<Profile> {
    // Check if user can read this profile
    if (targetUserId === requestingUser.id) {
      this.permissionChecker.requirePermission(
        requestingUser,
        Permission.READ_OWN_PROFILE
      );
    } else {
      this.permissionChecker.requirePermission(
        requestingUser,
        Permission.READ_ALL_PROFILES
      );
    }
    
    return await this.profileRepository.findById(targetUserId);
  }

  async deleteProfile(requestingUser: User, targetUserId: string): Promise<void> {
    // Only users with delete permission can delete
    this.permissionChecker.requirePermission(
      requestingUser,
      Permission.DELETE_PROFILES
    );
    
    // Additional business logic checks
    if (targetUserId === requestingUser.id) {
      throw new Error('Cannot delete own profile');
    }
    
    await this.profileRepository.delete(targetUserId);
  }
}
```

### Resource Access Control

```typescript
// Principle of Least Privilege for file system access

// ❌ BAD: Broad file system access
import fs from 'fs';

function readUserFile(filename: string): string {
  // Can read ANY file on system!
  return fs.readFileSync(filename, 'utf8');
}

// ✓ GOOD: Restricted file system access
import path from 'path';

class SecureFileReader {
  private readonly baseDirectory: string;
  private readonly allowedExtensions: Set<string>;

  constructor(baseDirectory: string, allowedExtensions: string[]) {
    this.baseDirectory = path.resolve(baseDirectory);
    this.allowedExtensions = new Set(allowedExtensions);
  }

  readFile(filename: string): string {
    // 1. Validate filename format
    if (!/^[a-zA-Z0-9_-]+\.[a-zA-Z0-9]+$/.test(filename)) {
      throw new Error('Invalid filename format');
    }

    // 2. Check file extension
    const ext = path.extname(filename);
    if (!this.allowedExtensions.has(ext)) {
      throw new Error(`File extension ${ext} not allowed`);
    }

    // 3. Resolve full path
    const fullPath = path.resolve(this.baseDirectory, filename);

    // 4. Ensure path is within base directory (prevent path traversal)
    if (!fullPath.startsWith(this.baseDirectory)) {
      throw new Error('Path traversal attempt detected');
    }

    // 5. Check file exists and is readable
    if (!fs.existsSync(fullPath)) {
      throw new Error('File not found');
    }

    // 6. Read file
    return fs.readFileSync(fullPath, 'utf8');
  }
}

// Usage
const fileReader = new SecureFileReader(
  './user-uploads',
  ['.txt', '.json', '.csv']
);

try {
  const content = fileReader.readFile('document.txt');  // ✓ Allowed
  // fileReader.readFile('../../../etc/passwd');  // ✗ Blocked
  // fileReader.readFile('script.exe');  // ✗ Blocked
} catch (error) {
  console.error('File access denied:', error.message);
}
```

### API Access Control

```typescript
// Least Privilege in API design

// ❌ BAD: One endpoint with too much power
@Post('/users/manage')
async manageUser(
  @Body() body: { action: string; userId: string; data: any }
) {
  // One endpoint does everything - difficult to secure
  switch (body.action) {
    case 'create': return this.createUser(body.data);
    case 'update': return this.updateUser(body.userId, body.data);
    case 'delete': return this.deleteUser(body.userId);
    case 'ban': return this.banUser(body.userId);
  }
}

// ✓ GOOD: Separate endpoints with specific permissions
@Controller('users')
export class UserController {
  @Post()
  @RequirePermission(Permission.CREATE_USER)
  async createUser(@Body() data: CreateUserDto) {
    return this.userService.create(data);
  }

  @Put(':id')
  @RequirePermission(Permission.UPDATE_USER)
  async updateUser(
    @Param('id') id: string,
    @Body() data: UpdateUserDto,
    @CurrentUser() user: User
  ) {
    // Users can only update their own profile
    if (id !== user.id && !user.permissions.has(Permission.UPDATE_ALL_USERS)) {
      throw new ForbiddenException();
    }
    
    return this.userService.update(id, data);
  }

  @Delete(':id')
  @RequirePermission(Permission.DELETE_USER)
  async deleteUser(@Param('id') id: string) {
    return this.userService.delete(id);
  }
}
```

## Defense in Depth

### Multi-Layer Security

```typescript
// Defense in Depth: Multiple security layers

class DefenseInDepthExample {
  async processUserInput(input: string, user: User): Promise<Result> {
    // Layer 1: Authentication
    if (!user.isAuthenticated) {
      throw new UnauthorizedError('User not authenticated');
    }

    // Layer 2: Rate Limiting
    if (!await this.rateLimiter.checkLimit(user.id)) {
      throw new TooManyRequestsError('Rate limit exceeded');
    }

    // Layer 3: Input Validation
    const validationResult = this.validateInput(input);
    if (!validationResult.valid) {
      throw new ValidationError(validationResult.errors);
    }

    // Layer 4: Sanitization
    const sanitizedInput = this.sanitizeInput(input);

    // Layer 5: Authorization
    if (!await this.checkPermissions(user, 'process_input')) {
      throw new ForbiddenError('Insufficient permissions');
    }

    // Layer 6: Business Logic
    const result = await this.processWithBusinessRules(sanitizedInput);

    // Layer 7: Output Encoding
    const encodedResult = this.encodeOutput(result);

    // Layer 8: Audit Logging
    await this.auditLog.log({
      user: user.id,
      action: 'process_input',
      input: sanitizedInput,
      result: 'success'
    });

    return encodedResult;
  }

  private validateInput(input: string): ValidationResult {
    const errors: string[] = [];

    if (input.length === 0) {
      errors.push('Input cannot be empty');
    }

    if (input.length > 1000) {
      errors.push('Input too long (max 1000 chars)');
    }

    if (!/^[a-zA-Z0-9\s.,!?-]+$/.test(input)) {
      errors.push('Input contains invalid characters');
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  private sanitizeInput(input: string): string {
    // Remove potentially dangerous patterns
    return input
      .replace(/<script[^>]*>.*?<\/script>/gi, '')
      .replace(/javascript:/gi, '')
      .replace(/on\w+\s*=/gi, '')
      .trim();
  }

  private encodeOutput(result: any): any {
    // HTML-encode strings in result
    if (typeof result === 'string') {
      return this.htmlEncode(result);
    }
    
    if (typeof result === 'object') {
      const encoded: any = {};
      for (const [key, value] of Object.entries(result)) {
        encoded[key] = this.encodeOutput(value);
      }
      return encoded;
    }
    
    return result;
  }

  private htmlEncode(str: string): string {
    return str
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;');
  }
}
```

### Security Middleware Stack

```typescript
// Express: Defense in Depth with middleware

import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import { body, validationResult } from 'express-validator';

const app = express();

// Layer 1: Security Headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'strict-dynamic'"],
      objectSrc: ["'none'"],
      baseUri: ["'self'"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));

// Layer 2: Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false
});
app.use('/api/', limiter);

// Layer 3: CORS
app.use((req, res, next) => {
  const allowedOrigins = ['https://example.com', 'https://app.example.com'];
  const origin = req.headers.origin;
  
  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
  }
  
  next();
});

// Layer 4: Authentication
const authenticateToken = (req: any, res: any, next: any) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  try {
    const user = verifyToken(token);
    req.user = user;
    next();
  } catch {
    return res.status(403).json({ error: 'Invalid token' });
  }
};

// Layer 5: Input Validation
const validateUser = [
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 12 }).matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  body('name').trim().isLength({ min: 2, max: 100 }).escape()
];

// Layer 6: Authorization
const requirePermission = (permission: string) => {
  return (req: any, res: any, next: any) => {
    if (!req.user.permissions.includes(permission)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};

// Layer 7: Audit Logging
const auditLog = (action: string) => {
  return (req: any, res: any, next: any) => {
    logAuditEvent({
      user: req.user.id,
      action,
      timestamp: new Date(),
      ip: req.ip,
      userAgent: req.headers['user-agent']
    });
    next();
  };
};

// Protected endpoint with all layers
app.post('/api/users',
  limiter,  // Rate limiting
  authenticateToken,  // Authentication
  requirePermission('create:user'),  // Authorization
  validateUser,  // Input validation
  auditLog('create_user'),  // Audit logging
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    
    // Create user
    const user = await createUser(req.body);
    res.json(user);
  }
);
```

## Secure Defaults

### Configuration with Secure Defaults

```typescript
// Secure by Default: Safe configuration out of the box

// ❌ BAD: Insecure defaults
class InsecureConfig {
  sessionTimeout = Infinity;  // Never expires
  allowHttp = true;  // HTTP allowed
  csrfProtection = false;  // CSRF disabled
  logLevel = 'debug';  // Verbose logging in production
}

// ✓ GOOD: Secure defaults
class SecureConfig {
  // Session timeout: 1 hour (secure default)
  sessionTimeout = 3600000;
  
  // HTTPS only (secure default)
  allowHttp = process.env.NODE_ENV === 'development';
  
  // CSRF protection enabled (secure default)
  csrfProtection = true;
  
  // Minimal logging in production (secure default)
  logLevel = process.env.NODE_ENV === 'production' ? 'error' : 'debug';
  
  // Secure cookie settings (secure defaults)
  cookieSettings = {
    httpOnly: true,  // Prevent JavaScript access
    secure: process.env.NODE_ENV === 'production',  // HTTPS only
    sameSite: 'strict' as const,  // CSRF protection
    maxAge: this.sessionTimeout
  };
  
  // Security headers (secure defaults)
  securityHeaders = {
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Content-Security-Policy': "default-src 'self'"
  };
  
  // Rate limiting (secure defaults)
  rateLimit = {
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,  // 100 requests per window
    message: 'Too many requests'
  };
}

// Make insecure options explicit
class ConfigBuilder {
  private config = new SecureConfig();
  
  // Force explicit opt-in for insecure options
  allowHttpInProduction(): this {
    console.warn('WARNING: Allowing HTTP in production is insecure');
    this.config.allowHttp = true;
    return this;
  }
  
  disableCSRFProtection(): this {
    console.warn('WARNING: Disabling CSRF protection is insecure');
    this.config.csrfProtection = false;
    return this;
  }
  
  extendSessionTimeout(ms: number): this {
    if (ms > 24 * 60 * 60 * 1000) {  // > 24 hours
      console.warn('WARNING: Session timeout longer than 24 hours is insecure');
    }
    this.config.sessionTimeout = ms;
    return this;
  }
  
  build(): SecureConfig {
    return this.config;
  }
}

// Usage: Secure by default, explicit for insecure
const config = new ConfigBuilder()
  // .allowHttpInProduction()  // Must explicitly enable
  // .disableCSRFProtection()   // Must explicitly disable
  .build();
```

### React Component Secure Defaults

```typescript
// React: Secure defaults in components

// ❌ BAD: Unsafe default props
interface UnsafeButtonProps {
  onClick?: () => void;
  allowDoubleClick?: boolean;  // Default undefined (falsy)
  sanitizeInput?: boolean;  // Default undefined (falsy)
}

// ✓ GOOD: Secure default props
interface SecureButtonProps {
  onClick?: () => void;
  allowDoubleClick?: boolean;
  sanitizeInput?: boolean;
}

const SecureButton: React.FC<SecureButtonProps> = ({ 
  onClick,
  allowDoubleClick = false,  // Secure default: false
  sanitizeInput = true,  // Secure default: true
  children 
}) => {
  const [lastClick, setLastClick] = useState<number>(0);
  
  const handleClick = () => {
    // Prevent double-click by default
    if (!allowDoubleClick) {
      const now = Date.now();
      if (now - lastClick < 1000) {
        return;  // Ignore click within 1 second
      }
      setLastClick(now);
    }
    
    onClick?.();
  };
  
  // Sanitize children by default
  const sanitizedChildren = sanitizeInput 
    ? DOMPurify.sanitize(String(children))
    : children;
  
  return (
    <button onClick={handleClick}>
      {sanitizedChildren}
    </button>
  );
};

// Usage
<SecureButton onClick={handleAction}>
  Click Me
</SecureButton>

// To disable security (explicit)
<SecureButton 
  onClick={handleAction}
  allowDoubleClick={true}  // Explicit
  sanitizeInput={false}  // Explicit
>
  Raw HTML Content
</SecureButton>
```

## Input Validation Patterns

### Comprehensive Input Validation

```typescript
// Input validation with builder pattern

class InputValidator {
  private rules: ValidationRule[] = [];

  string(fieldName: string): StringValidator {
    return new StringValidator(this, fieldName);
  }

  number(fieldName: string): NumberValidator {
    return new NumberValidator(this, fieldName);
  }

  email(fieldName: string): EmailValidator {
    return new EmailValidator(this, fieldName);
  }

  validate(data: any): ValidationResult {
    const errors: ValidationError[] = [];
    
    for (const rule of this.rules) {
      const value = data[rule.fieldName];
      const result = rule.validate(value);
      
      if (!result.valid) {
        errors.push({
          field: rule.fieldName,
          message: result.message
        });
      }
    }
    
    return {
      valid: errors.length === 0,
      errors
    };
  }

  addRule(rule: ValidationRule): void {
    this.rules.push(rule);
  }
}

class StringValidator {
  private fieldName: string;
  private parent: InputValidator;

  constructor(parent: InputValidator, fieldName: string) {
    this.parent = parent;
    this.fieldName = fieldName;
  }

  required(): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: any) => ({
        valid: value !== undefined && value !== null && value !== '',
        message: `${this.fieldName} is required`
      })
    });
    return this;
  }

  minLength(min: number): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: string) => ({
        valid: !value || value.length >= min,
        message: `${this.fieldName} must be at least ${min} characters`
      })
    });
    return this;
  }

  maxLength(max: number): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: string) => ({
        valid: !value || value.length <= max,
        message: `${this.fieldName} must be at most ${max} characters`
      })
    });
    return this;
  }

  pattern(regex: RegExp, message?: string): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: string) => ({
        valid: !value || regex.test(value),
        message: message || `${this.fieldName} format is invalid`
      })
    });
    return this;
  }

  alphanumeric(): this {
    return this.pattern(
      /^[a-zA-Z0-9]+$/,
      `${this.fieldName} must be alphanumeric`
    );
  }

  noSpecialChars(): this {
    return this.pattern(
      /^[a-zA-Z0-9\s-_]+$/,
      `${this.fieldName} contains invalid characters`
    );
  }

  build(): InputValidator {
    return this.parent;
  }
}

class NumberValidator {
  private fieldName: string;
  private parent: InputValidator;

  constructor(parent: InputValidator, fieldName: string) {
    this.parent = parent;
    this.fieldName = fieldName;
  }

  required(): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: any) => ({
        valid: value !== undefined && value !== null,
        message: `${this.fieldName} is required`
      })
    });
    return this;
  }

  min(min: number): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: number) => ({
        valid: value === undefined || value >= min,
        message: `${this.fieldName} must be at least ${min}`
      })
    });
    return this;
  }

  max(max: number): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: number) => ({
        valid: value === undefined || value <= max,
        message: `${this.fieldName} must be at most ${max}`
      })
    });
    return this;
  }

  integer(): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: number) => ({
        valid: value === undefined || Number.isInteger(value),
        message: `${this.fieldName} must be an integer`
      })
    });
    return this;
  }

  build(): InputValidator {
    return this.parent;
  }
}

class EmailValidator {
  private fieldName: string;
  private parent: InputValidator;

  constructor(parent: InputValidator, fieldName: string) {
    this.parent = parent;
    this.fieldName = fieldName;
  }

  required(): this {
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: any) => ({
        valid: value !== undefined && value !== null && value !== '',
        message: `${this.fieldName} is required`
      })
    });
    return this;
  }

  validFormat(): this {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    this.parent.addRule({
      fieldName: this.fieldName,
      validate: (value: string) => ({
        valid: !value || emailRegex.test(value),
        message: `${this.fieldName} must be a valid email address`
      })
    });
    return this;
  }

  build(): InputValidator {
    return this.parent;
  }
}

// Usage
const validator = new InputValidator();

validator
  .string('username')
    .required()
    .minLength(3)
    .maxLength(20)
    .alphanumeric()
    .build()
  .string('password')
    .required()
    .minLength(12)
    .pattern(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/,
      'Password must contain uppercase, lowercase, number, and special char'
    )
    .build()
  .email('email')
    .required()
    .validFormat()
    .build()
  .number('age')
    .required()
    .min(18)
    .max(120)
    .integer()
    .build();

const result = validator.validate({
  username: 'user123',
  password: 'SecurePass123!',
  email: 'user@example.com',
  age: 25
});

if (!result.valid) {
  console.error('Validation errors:', result.errors);
}
```

## Output Encoding

### Context-Aware Encoding

```typescript
// Output encoding for different contexts

class OutputEncoder {
  // HTML context
  encodeHTML(str: string): string {
    return str
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;')
      .replace(/\//g, '&#x2F;');
  }

  // HTML attribute context
  encodeHTMLAttribute(str: string): string {
    return str
      .replace(/&/g, '&amp;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  }

  // JavaScript context
  encodeJavaScript(str: string): string {
    return str
      .replace(/\\/g, '\\\\')
      .replace(/'/g, "\\'")
      .replace(/"/g, '\\"')
      .replace(/\n/g, '\\n')
      .replace(/\r/g, '\\r')
      .replace(/\t/g, '\\t')
      .replace(/\//g, '\\/')
      .replace(/</g, '\\x3C')
      .replace(/>/g, '\\x3E');
  }

  // URL context
  encodeURL(str: string): string {
    return encodeURIComponent(str);
  }

  // CSS context
  encodeCSS(str: string): string {
    return str.replace(/[^a-zA-Z0-9]/g, (char) => {
      const hex = char.charCodeAt(0).toString(16).padStart(6, '0');
      return `\\${hex}`;
    });
  }

  // SQL context (use parameterized queries instead!)
  encodeSQL(str: string): string {
    return str
      .replace(/'/g, "''")
      .replace(/\\/g, '\\\\')
      .replace(/\n/g, '\\n')
      .replace(/\r/g, '\\r')
      .replace(/\x00/g, '\\0')
      .replace(/\x1a/g, '\\Z');
  }

  // JSON context
  encodeJSON(obj: any): string {
    return JSON.stringify(obj)
      .replace(/</g, '\\u003c')
      .replace(/>/g, '\\u003e');
  }
}

// Usage examples
const encoder = new OutputEncoder();
const userInput = '<script>alert("XSS")</script>';

// HTML context
const htmlSafe = encoder.encodeHTML(userInput);
// Output: &lt;script&gt;alert(&quot;XSS&quot;)&lt;&#x2F;script&gt;

// JavaScript context
const jsSafe = encoder.encodeJavaScript(userInput);
// var message = '${jsSafe}';

// URL context
const urlSafe = encoder.encodeURL(userInput);
// https://example.com/search?q=${urlSafe}

// CSS context
const cssSafe = encoder.encodeCSS(userInput);
// .${cssSafe} { color: red; }
```

## Security Code Reviews

### Security Review Checklist

```typescript
// Automated security checks for code reviews

class SecurityCodeReviewer {
  async reviewCode(filePath: string): Promise<SecurityIssue[]> {
    const code = await fs.readFile(filePath, 'utf8');
    const issues: SecurityIssue[] = [];

    // Check for dangerous patterns
    issues.push(...this.checkDangerousPatterns(code));
    
    // Check for hardcoded secrets
    issues.push(...this.checkHardcodedSecrets(code));
    
    // Check for SQL injection vulnerabilities
    issues.push(...this.checkSQLInjection(code));
    
    // Check for XSS vulnerabilities
    issues.push(...this.checkXSS(code));
    
    // Check for insecure randomness
    issues.push(...this.checkInsecureRandom(code));
    
    // Check for weak cryptography
    issues.push(...this.checkWeakCrypto(code));

    return issues;
  }

  private checkDangerousPatterns(code: string): SecurityIssue[] {
    const issues: SecurityIssue[] = [];
    const dangerousPatterns = [
      {
        pattern: /eval\s*\(/g,
        message: 'Use of eval() is dangerous',
        severity: 'HIGH'
      },
      {
        pattern: /new Function\s*\(/g,
        message: 'Use of Function constructor is dangerous',
        severity: 'HIGH'
      },
      {
        pattern: /innerHTML\s*=/g,
        message: 'Direct innerHTML assignment can lead to XSS',
        severity: 'MEDIUM'
      },
      {
        pattern: /document\.write\s*\(/g,
        message: 'document.write() can be dangerous',
        severity: 'MEDIUM'
      },
      {
        pattern: /dangerouslySetInnerHTML/g,
        message: 'dangerouslySetInnerHTML without sanitization',
        severity: 'HIGH'
      }
    ];

    for (const { pattern, message, severity } of dangerousPatterns) {
      const matches = code.match(pattern);
      if (matches) {
        issues.push({
          type: 'DANGEROUS_PATTERN',
          message,
          severity,
          occurrences: matches.length
        });
      }
    }

    return issues;
  }

  private checkHardcodedSecrets(code: string): SecurityIssue[] {
    const issues: SecurityIssue[] = [];
    const secretPatterns = [
      {
        pattern: /password\s*=\s*['"][^'"]{8,}['"]/gi,
        message: 'Possible hardcoded password'
      },
      {
        pattern: /api[_-]?key\s*=\s*['"][^'"]+['"]/gi,
        message: 'Possible hardcoded API key'
      },
      {
        pattern: /secret\s*=\s*['"][^'"]+['"]/gi,
        message: 'Possible hardcoded secret'
      },
      {
        pattern: /token\s*=\s*['"][A-Za-z0-9+/=]{20,}['"]/gi,
        message: 'Possible hardcoded token'
      },
      {
        pattern: /(?:sk|pk)_live_[A-Za-z0-9]{24}/g,
        message: 'Possible Stripe live key'
      }
    ];

    for (const { pattern, message } of secretPatterns) {
      if (pattern.test(code)) {
        issues.push({
          type: 'HARDCODED_SECRET',
          message,
          severity: 'CRITICAL'
        });
      }
    }

    return issues;
  }

  private checkSQLInjection(code: string): SecurityIssue[] {
    const issues: SecurityIssue[] = [];
    
    // Check for string concatenation in SQL
    const sqlConcatPattern = /(?:query|execute)\s*\([^)]*\+[^)]*\)/g;
    const sqlTemplatePattern = /(?:query|execute)\s*\(`[^`]*\$\{[^}]+\}[^`]*`\)/g;

    if (sqlConcatPattern.test(code)) {
      issues.push({
        type: 'SQL_INJECTION',
        message: 'SQL query uses string concatenation instead of parameterized queries',
        severity: 'CRITICAL'
      });
    }

    if (sqlTemplatePattern.test(code)) {
      issues.push({
        type: 'SQL_INJECTION',
        message: 'SQL query uses template literals instead of parameterized queries',
        severity: 'CRITICAL'
      });
    }

    return issues;
  }

  private checkXSS(code: string): SecurityIssue[] {
    const issues: SecurityIssue[] = [];
    
    // Check for unsafe DOM manipulation
    const unsafeDOMPatterns = [
      /\.innerHTML\s*=\s*[^;]+(?!DOMPurify)/g,
      /\.outerHTML\s*=\s*[^;]+(?!DOMPurify)/g,
      /document\.write\s*\([^)]*\$/g,
      /dangerouslySetInnerHTML[^}]*(?!DOMPurify)/g
    ];

    for (const pattern of unsafeDOMPatterns) {
      if (pattern.test(code)) {
        issues.push({
          type: 'XSS',
          message: 'Unsafe DOM manipulation without sanitization',
          severity: 'HIGH'
        });
      }
    }

    return issues;
  }

  private checkInsecureRandom(code: string): SecurityIssue[] {
    const issues: SecurityIssue[] = [];
    
    // Check for Math.random() in security contexts
    if (/Math\.random\(\)/g.test(code)) {
      // Check if it's in a security-sensitive context
      const securityContexts = [
        /token/i,
        /session/i,
        /nonce/i,
        /csrf/i,
        /password/i
      ];
      
      for (const context of securityContexts) {
        if (context.test(code)) {
          issues.push({
            type: 'WEAK_RANDOMNESS',
            message: 'Math.random() used in security context - use crypto.randomBytes() instead',
            severity: 'HIGH'
          });
          break;
        }
      }
    }

    return issues;
  }

  private checkWeakCrypto(code: string): SecurityIssue[] {
    const issues: SecurityIssue[] = [];
    
    // Check for weak hash algorithms
    const weakAlgorithms = ['md5', 'sha1', 'rc4', 'des'];
    
    for (const algo of weakAlgorithms) {
      const pattern = new RegExp(`['"]${algo}['"]`, 'gi');
      if (pattern.test(code)) {
        issues.push({
          type: 'WEAK_CRYPTO',
          message: `Weak cryptographic algorithm: ${algo.toUpperCase()}`,
          severity: 'HIGH'
        });
      }
    }

    return issues;
  }
}

// Usage in CI/CD
async function runSecurityReview(files: string[]): Promise<void> {
  const reviewer = new SecurityCodeReviewer();
  let hasIssues = false;

  for (const file of files) {
    const issues = await reviewer.reviewCode(file);
    
    if (issues.length > 0) {
      console.error(`\n❌ Security issues in ${file}:`);
      
      for (const issue of issues) {
        console.error(`  [${issue.severity}] ${issue.message}`);
        
        if (issue.severity === 'CRITICAL') {
          hasIssues = true;
        }
      }
    }
  }

  if (hasIssues) {
    console.error('\n❌ Critical security issues found - build failed');
    process.exit(1);
  }
}
```

## Common Mistakes

### 1. Trusting Client-Side Data

```typescript
// ❌ BAD
const price = req.body.price;  // Trusting client
await chargeUser(price);

// ✓ GOOD
const productId = req.body.productId;
const product = await getProduct(productId);
await chargeUser(product.price);  // Using server price
```

### 2. Insufficient Input Validation

```typescript
// ❌ BAD
if (age > 0) { /* proceed */ }

// ✓ GOOD
if (Number.isInteger(age) && age >= 18 && age <= 120) {
  /* proceed */
}
```

### 3. Using Weak Randomness for Security

```typescript
// ❌ BAD
const token = Math.random().toString(36);

// ✓ GOOD
const token = crypto.randomBytes(32).toString('hex');
```

### 4. Not Encoding Output

```typescript
// ❌ BAD
element.innerHTML = userInput;

// ✓ GOOD
element.textContent = userInput;
// Or: element.innerHTML = DOMPurify.sanitize(userInput);
```

### 5. Ignoring Error Messages

```typescript
// ❌ BAD
try {
  processData();
} catch (error) {
  // Silently ignore
}

// ✓ GOOD
try {
  processData();
} catch (error) {
  logger.error('Data processing failed:', error);
  // Handle appropriately
}
```

## Best Practices

### 1. Fail Securely

```typescript
// Fail securely: Deny by default

class SecureFailureHandler {
  async performAction(user: User, action: string): Promise<Result> {
    try {
      // Check permissions
      if (!await this.hasPermission(user, action)) {
        // Fail closed: Deny access
        throw new ForbiddenError('Access denied');
      }

      // Perform action
      return await this.execute(action);
    } catch (error) {
      // Log error
      logger.error('Action failed:', error);
      
      // Fail securely: Don't leak information
      if (error instanceof ForbiddenError) {
        throw error;
      }
      
      // Generic error message
      throw new Error('Operation failed');
    }
  }

  private async hasPermission(user: User, action: string): Promise<boolean> {
    try {
      return await this.permissionService.check(user, action);
    } catch (error) {
      // If permission check fails, deny access
      return false;
    }
  }
}
```

### 2. Minimize Attack Surface

```typescript
// Minimize exposed functionality

// ❌ BAD: Exposing unnecessary endpoints
@Controller('admin')
export class AdminController {
  @Get('debug-info')
  getDebugInfo() {
    return {
      environment: process.env,
      database: dbConfig,
      secrets: secrets
    };
  }
}

// ✓ GOOD: Minimal necessary endpoints
@Controller('admin')
@UseGuards(AuthGuard, AdminGuard)
export class AdminController {
  @Get('users')
  async getUsers(@Query() query: PaginationDto) {
    return this.userService.findAll(query);
  }

  // Debug endpoints only in development
  @Get('health')
  @Env('development')
  getHealth() {
    return { status: 'ok' };
  }
}
```

### 3. Separation of Duties

```typescript
// Separate privileges for different operations

class BankingService {
  async initiateTransfer(
    fromAccount: string,
    toAccount: string,
    amount: number,
    initiatedBy: User
  ): Promise<Transfer> {
    // User can initiate transfer
    this.requirePermission(initiatedBy, 'initiate:transfer');
    
    const transfer = await this.createPendingTransfer(
      fromAccount,
      toAccount,
      amount,
      initiatedBy.id
    );
    
    // Transfer requires approval (separation of duties)
    await this.notifyApprovers(transfer);
    
    return transfer;
  }

  async approveTransfer(
    transferId: string,
    approvedBy: User
  ): Promise<void> {
    // Different user must approve
    this.requirePermission(approvedBy, 'approve:transfer');
    
    const transfer = await this.getTransfer(transferId);
    
    // Ensure approver is not initiator
    if (transfer.initiatedBy === approvedBy.id) {
      throw new Error('Cannot approve own transfer');
    }
    
    await this.executeTransfer(transfer);
  }
}
```

### 4. Security Logging

```typescript
// Comprehensive security logging

class SecurityLogger {
  logAuthenticationAttempt(
    email: string,
    success: boolean,
    ip: string
  ): void {
    this.log({
      event: 'AUTHENTICATION_ATTEMPT',
      email,
      success,
      ip,
      timestamp: new Date()
    });

    if (!success) {
      this.checkForBruteForce(email, ip);
    }
  }

  logAuthorizationFailure(
    user: User,
    resource: string,
    action: string
  ): void {
    this.log({
      event: 'AUTHORIZATION_FAILURE',
      userId: user.id,
      resource,
      action,
      timestamp: new Date()
    });

    // Alert on repeated failures
    this.checkForPrivilegeEscalation(user.id);
  }

  logDataAccess(
    user: User,
    resource: string,
    operation: 'read' | 'write' | 'delete'
  ): void {
    this.log({
      event: 'DATA_ACCESS',
      userId: user.id,
      resource,
      operation,
      timestamp: new Date()
    });
  }

  logSecurityException(
    type: string,
    details: any,
    severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL'
  ): void {
    this.log({
      event: 'SECURITY_EXCEPTION',
      type,
      details,
      severity,
      timestamp: new Date()
    });

    if (severity === 'CRITICAL') {
      this.alertSecurityTeam(type, details);
    }
  }
}
```

## When to Use/Not to Use

### When to Apply Secure Coding Practices

**Always:**
- Input validation
- Output encoding
- Authentication and authorization
- Security logging
- Error handling

**Based on context:**
- Encryption (for sensitive data)
- Rate limiting (for public endpoints)
- CSRF protection (for state-changing operations)

### Security vs. Usability Trade-offs

```typescript
// Balance security and usability

class SecurityUsabilityBalance {
  // Example: Password requirements
  getPasswordRequirements(userType: 'consumer' | 'admin'): PasswordPolicy {
    if (userType === 'admin') {
      // Stricter for admins
      return {
        minLength: 16,
        requireUppercase: true,
        requireLowercase: true,
        requireNumber: true,
        requireSpecial: true,
        requireMFA: true
      };
    } else {
      // More lenient for consumers
      return {
        minLength: 12,
        requireUppercase: true,
        requireLowercase: true,
        requireNumber: true,
        requireSpecial: false,
        requireMFA: false
      };
    }
  }

  // Example: Session timeout
  getSessionTimeout(riskLevel: 'low' | 'medium' | 'high'): number {
    switch (riskLevel) {
      case 'low':  // Public data
        return 24 * 60 * 60 * 1000;  // 24 hours
      case 'medium':  // Personal data
        return 4 * 60 * 60 * 1000;  // 4 hours
      case 'high':  // Financial transactions
        return 15 * 60 * 1000;  // 15 minutes
    }
  }
}
```

## Interview Questions

### Q1: Explain the principle of least privilege with an example.

**Answer:** Least Privilege means granting minimum necessary permissions to complete a task.

**Bad example:**
```typescript
// Admin role can do everything
if (user.role === 'admin') {
  // Allow all actions
}
```

**Good example:**
```typescript
// Fine-grained permissions
const permissions = {
  user: ['read:own-profile', 'update:own-profile'],
  moderator: ['read:all-profiles', 'update:posts'],
  admin: ['delete:users', 'manage:roles']
};

// Check specific permission
if (hasPermission(user, 'delete:users')) {
  deleteUser();
}
```

**Why important:**
- Limits damage from compromised accounts
- Reduces insider threat risk
- Easier to audit
- Principle of fail-secure

**Real-world analogy:** Bank teller can process transactions but can't access vault.

### Q2: What is defense in depth and why is it important?

**Answer:** Defense in Depth means implementing multiple security layers so if one fails, others still protect.

**Layers example:**
1. **Network:** Firewall
2. **Application:** Authentication
3. **Application:** Authorization
4. **Application:** Input validation
5. **Application:** Output encoding
6. **Application:** Rate limiting
7. **Data:** Encryption at rest
8. **Monitoring:** Intrusion detection

**Why important:**
- No single point of failure
- Redundant security
- Slows attackers (more barriers)
- Catches different attack types

**Anti-pattern:** Relying on single security measure (e.g., only firewall).

**Example:** Even with strong auth, still validate input (defense in depth).

### Q3: What does it mean to "fail securely"?

**Answer:** Fail securely means when errors occur, default to denying access rather than granting it.

**Examples:**

**1. Permission checks:**
```typescript
// ❌ Fails insecurely
try {
  if (checkPermission(user, action)) {
    allow();
  }
} catch (error) {
  allow();  // Grants access on error!
}

// ✓ Fails securely
try {
  if (checkPermission(user, action)) {
    allow();
  } else {
    deny();
  }
} catch (error) {
  deny();  // Denies access on error
}
```

**2. Error messages:**
```typescript
// ❌ Leaks information
if (!userExists) {
  return 'User not found';
}
if (!passwordMatch) {
  return 'Invalid password';
}

// ✓ Generic message
return 'Invalid email or password';
```

**3. Default deny:**
```typescript
// ❌ Default allow
if (user.role === 'admin') {
  allow();
} else {
  // Implicit allow for everyone else
}

// ✓ Default deny
if (user.role === 'admin') {
  allow();
} else {
  deny();  // Explicit denial
}
```

**Why important:** Prevents accidental security bypasses due to errors.

### Q4: Why is secure by default important in configuration?

**Answer:** Secure by default means safe configuration out of the box, requiring explicit action to weaken security.

**Benefits:**

**1. Prevents mistakes:**
```typescript
// ❌ Insecure by default
class Config {
  csrfProtection = false;  // Must remember to enable
}

// ✓ Secure by default
class Config {
  csrfProtection = true;  // Secure unless explicitly disabled
}
```

**2. Protects less-experienced developers:**
- Don't need to know all security settings
- Safe defaults guide toward security

**3. Defense in depth:**
- Even if developer forgets, still protected
- Multiple layers of security

**Examples:**
- HTTPS-only cookies by default
- httpOnly flag true by default
- SameSite=Lax by default (modern browsers)
- Short session timeouts by default

**Anti-pattern:** Require developers to remember every security setting.

### Q5: What is the difference between validation and sanitization?

**Answer:**

**Validation:** Check if input meets requirements (accept/reject)
```typescript
// Validation
if (!/^[a-zA-Z0-9]+$/.test(username)) {
  throw new Error('Invalid username');
}
```

**Sanitization:** Clean input to make it safe (transform)
```typescript
// Sanitization
const cleaned = username.replace(/[^a-zA-Z0-9]/g, '');
```

**When to use each:**

**Validation:**
- User registration (strict format required)
- API endpoints (reject malformed requests)
- Configuration (must be valid)

**Sanitization:**
- User-generated content (comments, posts)
- Search queries (allow but clean)
- Display data (prevent XSS)

**Best practice:** Often use both
```typescript
// 1. Validate format
if (!isValidEmail(email)) {
  reject();
}

// 2. Sanitize for storage/display
email = email.trim().toLowerCase();
```

**Key difference:** Validation rejects, sanitization transforms.

### Q6: How do you implement separation of duties in code?

**Answer:** Separation of duties requires different users/roles for critical operations to prevent fraud.

**Implementation:**

**1. Multiple approvals:**
```typescript
class FinancialTransaction {
  status: 'pending' | 'approved' | 'completed';
  initiatedBy: string;
  approvedBy?: string;

  async initiate(user: User) {
    this.initiatedBy = user.id;
    this.status = 'pending';
  }

  async approve(user: User) {
    // Cannot approve own transaction
    if (this.initiatedBy === user.id) {
      throw new Error('Cannot approve own transaction');
    }
    
    this.approvedBy = user.id;
    this.status = 'approved';
  }
}
```

**2. Role-based workflows:**
```typescript
// Developer can create code
// Reviewer can approve code
// Deployer can deploy code
// No single person can do all three
```

**3. Audit trail:**
```typescript
class AuditLog {
  logAction(action: string, user: string, details: any) {
    // Record who did what, when
    // Detect if same user doing everything
  }
}
```

**Real-world examples:**
- Bank transfers (initiate vs. approve)
- Code deployment (write vs. review vs. deploy)
- Purchase orders (request vs. approve vs. pay)

**Why important:**
- Prevents insider fraud
- Requires collusion (harder)
- Accountability and audit trail

### Q7: What security considerations are important for logging?

**Answer:** Security logging considerations:

**What to log:**
- Authentication events (login, logout, failures)
- Authorization failures
- Data access (especially sensitive data)
- Configuration changes
- Security exceptions

**What NOT to log:**
- Passwords (ever!)
- Credit card numbers
- Personal identification numbers
- Session tokens
- Private keys

**Example:**
```typescript
// ❌ BAD
logger.info('Login attempt', { 
  email, 
  password  // Never log passwords!
});

// ✓ GOOD
logger.info('Login attempt', { 
  email, 
  success: false,
  ip: req.ip
});
```

**Log sanitization:**
```typescript
function sanitizeForLogging(data: any): any {
  const sanitized = { ...data };
  
  // Remove sensitive fields
  delete sanitized.password;
  delete sanitized.creditCard;
  delete sanitized.ssn;
  
  return sanitized;
}
```

**Security considerations:**
- **Storage:** Secure log storage (encrypted)
- **Access:** Restrict who can view logs
- **Retention:** Define retention policy
- **Integrity:** Prevent tampering (append-only)
- **Monitoring:** Alert on suspicious patterns

**Best practice:** Log enough for forensics without logging secrets.

### Q8: How do you balance security and usability?

**Answer:** Security vs. usability is about finding appropriate balance for risk level.

**Framework for decision:**

**1. Risk assessment:**
```typescript
const securityLevel = assessRisk({
  dataType: 'financial',  // High risk
  userType: 'consumer',   // Lower trust
  operation: 'transfer',  // Critical operation
});

// Result: Apply strict security
```

**2. Proportionate measures:**
```
Low risk (public blog):
  - Simple password
  - Long session
  - Basic rate limiting

High risk (banking):
  - Strong password + MFA
  - Short session
  - Aggressive rate limiting
  - Transaction verification
```

**3. User experience:**
- **Progressive enhancement:** Start lenient, tighten based on behavior
- **Contextual security:** More security for risky operations
- **Clear communication:** Explain why security measures exist

**Examples:**

**Good balance:**
```typescript
// Remember device (security + usability)
if (isKnownDevice(user)) {
  skipMFA();  // Usability win
} else {
  requireMFA();  // Security for unknown device
}
```

**Bad balance:**
```typescript
// Too much security
requireMFA();  // Every login, known device or not
changePasswordEvery30Days();
sessionTimeout15Minutes();
// Result: Users frustrated, find workarounds

// Too little security
noPasswordRequirements();
neverExpireSessions();
noMFA();
// Result: Security breaches
```

**Best practice:**
- Apply security appropriate to risk
- Make security transparent when possible
- Educate users on why security matters
- Monitor and adjust based on feedback

## Key Takeaways

1. **Least privilege always** - Grant minimum necessary permissions; fine-grained permissions better than broad roles
2. **Defense in depth** - Multiple security layers so single failure doesn't compromise system
3. **Fail securely** - Default to deny on errors; never grant access when uncertain
4. **Secure by default** - Safe configuration out of box; require explicit action to weaken security
5. **Validate input, encode output** - Never trust input; always encode output for context
6. **Separate duties** - Critical operations require multiple parties to prevent fraud
7. **Security logging** - Log security events but never secrets; enable forensics and monitoring
8. **Code reviews** - Automated tools + human review catch vulnerabilities before production
9. **Balance security and usability** - Apply security appropriate to risk level
10. **Security mindset** - Build security in from start (shift-left); cheaper and more effective

## Resources

### Standards and Guidelines
- [OWASP Secure Coding Practices](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/)
- [CERT Secure Coding Standards](https://wiki.sei.cmu.edu/confluence/display/seccode)
- [CWE/SANS Top 25](https://www.sans.org/top25-software-errors/)

### Tools
- [SonarQube](https://www.sonarqube.org/) - Static code analysis
- [ESLint Security Plugin](https://github.com/nodesecurity/eslint-plugin-security)
- [Semgrep](https://semgrep.dev/) - Pattern-based code scanning

### Books
- "Secure Coding in C and C++" by Robert C. Seacord
- "The Web Application Hacker's Handbook" by Dafydd Stuttard
- "Security Engineering" by Ross Anderson

### Training
- [Secure Code Warrior](https://www.securecodewarrior.com/)
- [HackerOne CTF](https://www.hackerone.com/ethical-hacker/capture-the-flag)
- [OWASP WebGoat](https://owasp.org/www-project-webgoat/)
