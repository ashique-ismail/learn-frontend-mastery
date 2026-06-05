# Input Validation and Sanitization

## The Idea

**In plain English:** Input validation means checking that data submitted by a user is what you actually expect (a number, not a paragraph of code). Sanitization means cleaning up any dangerous content before you use it — so a malicious user can't smuggle in code that harms other users.

**Real-world analogy:** Think of a nightclub with a bag check. The bouncer first checks if the bag is the right size (validation — "no oversized bags allowed"). Then security opens it and removes anything dangerous like weapons (sanitization — stripping out harmful content) before letting the person in.

- The bag size check = validation (is this the expected format/type/length?)
- Removing dangerous items from the bag = sanitization (stripping `<script>` tags, escaping special characters)
- The nightclub itself = your application's database or UI
- A person smuggling in weapons = a user submitting malicious input (XSS, SQL injection)

---

## Table of Contents
- [Introduction](#introduction)
- [Client-Side vs Server-Side Validation](#client-side-vs-server-side-validation)
- [Input Validation Strategies](#input-validation-strategies)
- [Sanitization Techniques](#sanitization-techniques)
- [SQL Injection Prevention](#sql-injection-prevention)
- [XSS Prevention](#xss-prevention)
- [Command Injection Prevention](#command-injection-prevention)
- [Implementation Examples](#implementation-examples)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

**Input Validation and Sanitization** are critical security practices that ensure user input is safe, expected, and properly formatted before processing. Validation checks if input meets requirements, while sanitization removes or encodes dangerous content.

### Attack Surface

```
┌─────────────────────────────────────────────────────────────┐
│            Input-Based Attack Vectors                        │
└─────────────────────────────────────────────────────────────┘

User Input Points:
  ├─ Form fields (text, email, password)
  ├─ URL parameters (?id=123&sort=name)
  ├─ Request headers (User-Agent, Referer)
  ├─ Cookies
  ├─ File uploads
  ├─ API request bodies (JSON, XML)
  └─ WebSocket messages

Attack Types:
  ├─ SQL Injection: ' OR '1'='1
  ├─ XSS: <script>alert('XSS')</script>
  ├─ Command Injection: ; rm -rf /
  ├─ Path Traversal: ../../etc/passwd
  ├─ XXE: <?xml version="1.0"?><!DOCTYPE root [<!ENTITY test SYSTEM 'file:///etc/passwd'>]>
  └─ LDAP Injection: *)(uid=*))(|(uid=*
```

## Client-Side vs Server-Side Validation

### Validation Layers

```
┌─────────────────────────────────────────────────────────────┐
│          Defense-in-Depth Validation                         │
└─────────────────────────────────────────────────────────────┘

Layer 1: Client-Side (UX Enhancement)
   ┌────────────────────────────────────┐
   │ • Immediate feedback               │
   │ • Reduces server load              │
   │ • Better user experience           │
   │ • EASILY BYPASSED - NOT SECURE     │
   └────────────────────────────────────┘
                    ↓
Layer 2: Server-Side (Security Critical)
   ┌────────────────────────────────────┐
   │ • Authoritative validation         │
   │ • Cannot be bypassed               │
   │ • ALWAYS REQUIRED                  │
   │ • Protects business logic          │
   └────────────────────────────────────┘
                    ↓
Layer 3: Database (Final Defense)
   ┌────────────────────────────────────┐
   │ • Constraints (NOT NULL, UNIQUE)   │
   │ • Data type enforcement            │
   │ • Foreign key validation           │
   │ • Additional safety net            │
   └────────────────────────────────────┘
```

### React: Client-Side Validation

```typescript
// React: Form validation (UX only, not security)
import { useState } from 'react';

interface FormData {
  email: string;
  password: string;
  age: number;
}

interface FormErrors {
  email?: string;
  password?: string;
  age?: string;
}

const RegistrationForm: React.FC = () => {
  const [formData, setFormData] = useState<FormData>({
    email: '',
    password: '',
    age: 0
  });
  
  const [errors, setErrors] = useState<FormErrors>({});
  const [submitting, setSubmitting] = useState(false);

  // Client-side validation (UX improvement)
  const validateForm = (): boolean => {
    const newErrors: FormErrors = {};

    // Email validation
    if (!formData.email) {
      newErrors.email = 'Email is required';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email)) {
      newErrors.email = 'Invalid email format';
    }

    // Password validation
    if (!formData.password) {
      newErrors.password = 'Password is required';
    } else if (formData.password.length < 8) {
      newErrors.password = 'Password must be at least 8 characters';
    } else if (!/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(formData.password)) {
      newErrors.password = 'Password must contain uppercase, lowercase, and number';
    }

    // Age validation
    if (!formData.age || formData.age < 18 || formData.age > 120) {
      newErrors.age = 'Age must be between 18 and 120';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    // Client-side validation
    if (!validateForm()) {
      return;
    }

    setSubmitting(true);

    try {
      // Server MUST validate again (client-side can be bypassed)
      const response = await fetch('/api/register', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(formData)
      });

      if (!response.ok) {
        // Server validation failed
        const serverErrors = await response.json();
        setErrors(serverErrors.errors);
      } else {
        alert('Registration successful!');
      }
    } catch (error) {
      console.error('Registration failed:', error);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          type="email"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
          placeholder="Email"
        />
        {errors.email && <span className="error">{errors.email}</span>}
      </div>

      <div>
        <input
          type="password"
          value={formData.password}
          onChange={(e) => setFormData({ ...formData, password: e.target.value })}
          placeholder="Password"
        />
        {errors.password && <span className="error">{errors.password}</span>}
      </div>

      <div>
        <input
          type="number"
          value={formData.age}
          onChange={(e) => setFormData({ ...formData, age: parseInt(e.target.value) })}
          placeholder="Age"
        />
        {errors.age && <span className="error">{errors.age}</span>}
      </div>

      <button type="submit" disabled={submitting}>
        Register
      </button>
    </form>
  );
};
```

### Backend: Server-Side Validation (Critical)

```typescript
// Express: Server-side validation (REQUIRED for security)
import express from 'express';
import { body, validationResult, ValidationChain } from 'express-validator';

// Validation rules
const registrationValidation: ValidationChain[] = [
  body('email')
    .trim()
    .notEmpty().withMessage('Email is required')
    .isEmail().withMessage('Invalid email format')
    .normalizeEmail()
    .isLength({ max: 255 }).withMessage('Email too long'),

  body('password')
    .notEmpty().withMessage('Password is required')
    .isLength({ min: 8 }).withMessage('Password must be at least 8 characters')
    .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .withMessage('Password must contain uppercase, lowercase, and number'),

  body('age')
    .isInt({ min: 18, max: 120 }).withMessage('Age must be between 18 and 120')
    .toInt(),

  body('name')
    .trim()
    .notEmpty().withMessage('Name is required')
    .isLength({ max: 100 }).withMessage('Name too long')
    .matches(/^[a-zA-Z\s'-]+$/).withMessage('Name contains invalid characters')
];

// Route with validation
app.post('/api/register', registrationValidation, async (req, res) => {
  // Check validation results
  const errors = validationResult(req);
  
  if (!errors.isEmpty()) {
    return res.status(400).json({ 
      errors: errors.array() 
    });
  }

  // Validation passed - safe to process
  const { email, password, age, name } = req.body;

  try {
    // Additional business logic validation
    const existingUser = await db.users.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ 
        errors: [{ msg: 'Email already registered' }] 
      });
    }

    // Hash password (never store plaintext)
    const hashedPassword = await bcrypt.hash(password, 12);

    // Create user
    await db.users.create({
      email,
      password: hashedPassword,
      age,
      name
    });

    res.status(201).json({ message: 'Registration successful' });
  } catch (error) {
    console.error('Registration error:', error);
    res.status(500).json({ error: 'Registration failed' });
  }
});
```

## Input Validation Strategies

### Whitelist vs Blacklist Validation

```typescript
// Validation approaches

// ❌ BAD: Blacklist approach (incomplete, easy to bypass)
const blacklistValidation = (input: string): boolean => {
  const dangerousPatterns = [
    '<script',
    'javascript:',
    'onerror',
    'onclick'
  ];
  
  return !dangerousPatterns.some(pattern => 
    input.toLowerCase().includes(pattern)
  );
  // Easy to bypass: <scr<script>ipt>, javasc&#114;ipt:, etc.
};

// ✓ GOOD: Whitelist approach (explicitly allow safe patterns)
const whitelistValidation = (input: string, type: string): boolean => {
  const patterns: Record<string, RegExp> = {
    alphanumeric: /^[a-zA-Z0-9]+$/,
    email: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/,
    phone: /^\+?[1-9]\d{1,14}$/,
    url: /^https:\/\/[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(\/.*)?$/,
    username: /^[a-zA-Z0-9_-]{3,20}$/,
    hex: /^[0-9A-Fa-f]+$/,
    uuid: /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i
  };

  const pattern = patterns[type];
  return pattern ? pattern.test(input) : false;
};

// Usage
if (!whitelistValidation(username, 'username')) {
  throw new Error('Invalid username format');
}
```

### Type-Based Validation

```typescript
// Comprehensive validation class
class InputValidator {
  // String validation
  validateString(
    value: any,
    options: {
      minLength?: number;
      maxLength?: number;
      pattern?: RegExp;
      required?: boolean;
    } = {}
  ): string {
    // Type check
    if (typeof value !== 'string') {
      throw new Error('Value must be a string');
    }

    const trimmed = value.trim();

    // Required check
    if (options.required && !trimmed) {
      throw new Error('Value is required');
    }

    // Length validation
    if (options.minLength && trimmed.length < options.minLength) {
      throw new Error(`Minimum length is ${options.minLength}`);
    }

    if (options.maxLength && trimmed.length > options.maxLength) {
      throw new Error(`Maximum length is ${options.maxLength}`);
    }

    // Pattern validation
    if (options.pattern && !options.pattern.test(trimmed)) {
      throw new Error('Value does not match required pattern');
    }

    return trimmed;
  }

  // Number validation
  validateNumber(
    value: any,
    options: {
      min?: number;
      max?: number;
      integer?: boolean;
      required?: boolean;
    } = {}
  ): number {
    // Type coercion and check
    const num = Number(value);

    if (isNaN(num)) {
      throw new Error('Value must be a number');
    }

    // Required check
    if (options.required && num === 0) {
      throw new Error('Value is required');
    }

    // Integer check
    if (options.integer && !Number.isInteger(num)) {
      throw new Error('Value must be an integer');
    }

    // Range validation
    if (options.min !== undefined && num < options.min) {
      throw new Error(`Minimum value is ${options.min}`);
    }

    if (options.max !== undefined && num > options.max) {
      throw new Error(`Maximum value is ${options.max}`);
    }

    return num;
  }

  // Email validation
  validateEmail(value: string): string {
    const trimmed = value.trim().toLowerCase();
    
    // Basic format check
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(trimmed)) {
      throw new Error('Invalid email format');
    }

    // Additional checks
    if (trimmed.length > 255) {
      throw new Error('Email too long');
    }

    const [localPart, domain] = trimmed.split('@');

    if (localPart.length > 64) {
      throw new Error('Email local part too long');
    }

    // Check for dangerous characters
    const dangerousChars = /[<>()[\]\\,;:\s@"]/;
    if (dangerousChars.test(localPart.replace(/\./g, ''))) {
      throw new Error('Email contains invalid characters');
    }

    return trimmed;
  }

  // URL validation
  validateURL(value: string, options: { httpsOnly?: boolean } = {}): string {
    try {
      const url = new URL(value);

      // Protocol check
      const allowedProtocols = options.httpsOnly ? ['https:'] : ['http:', 'https:'];
      
      if (!allowedProtocols.includes(url.protocol)) {
        throw new Error('Invalid URL protocol');
      }

      // No localhost in production
      if (process.env.NODE_ENV === 'production') {
        if (url.hostname === 'localhost' || url.hostname === '127.0.0.1') {
          throw new Error('Localhost URLs not allowed');
        }
      }

      return url.toString();
    } catch (error) {
      throw new Error('Invalid URL format');
    }
  }

  // Date validation
  validateDate(
    value: any,
    options: {
      minDate?: Date;
      maxDate?: Date;
      required?: boolean;
    } = {}
  ): Date {
    const date = new Date(value);

    if (isNaN(date.getTime())) {
      throw new Error('Invalid date');
    }

    if (options.minDate && date < options.minDate) {
      throw new Error('Date is too early');
    }

    if (options.maxDate && date > options.maxDate) {
      throw new Error('Date is too late');
    }

    return date;
  }

  // Array validation
  validateArray<T>(
    value: any,
    itemValidator: (item: any) => T,
    options: {
      minLength?: number;
      maxLength?: number;
      required?: boolean;
    } = {}
  ): T[] {
    if (!Array.isArray(value)) {
      throw new Error('Value must be an array');
    }

    if (options.required && value.length === 0) {
      throw new Error('Array cannot be empty');
    }

    if (options.minLength && value.length < options.minLength) {
      throw new Error(`Array must have at least ${options.minLength} items`);
    }

    if (options.maxLength && value.length > options.maxLength) {
      throw new Error(`Array must have at most ${options.maxLength} items`);
    }

    // Validate each item
    return value.map((item, index) => {
      try {
        return itemValidator(item);
      } catch (error) {
        throw new Error(`Item ${index}: ${error.message}`);
      }
    });
  }
}

// Usage example
const validator = new InputValidator();

try {
  const email = validator.validateEmail(req.body.email);
  const age = validator.validateNumber(req.body.age, { min: 18, max: 120, integer: true });
  const website = validator.validateURL(req.body.website, { httpsOnly: true });
  
  // Process validated data
} catch (error) {
  res.status(400).json({ error: error.message });
}
```

## Sanitization Techniques

### HTML Sanitization with DOMPurify

```typescript
// DOMPurify usage for XSS prevention
import DOMPurify from 'dompurify';

class HTMLSanitizer {
  // Basic sanitization
  sanitize(html: string): string {
    return DOMPurify.sanitize(html);
  }

  // Strict sanitization (remove all HTML tags)
  sanitizeStrict(html: string): string {
    return DOMPurify.sanitize(html, { ALLOWED_TAGS: [] });
  }

  // Allow specific tags only
  sanitizeWithTags(html: string, allowedTags: string[]): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: allowedTags,
      ALLOWED_ATTR: ['href', 'src', 'alt', 'title']
    });
  }

  // Sanitize for rich text editor
  sanitizeRichText(html: string): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: [
        'p', 'br', 'strong', 'em', 'u', 's',
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'ul', 'ol', 'li',
        'a', 'img',
        'blockquote', 'code', 'pre'
      ],
      ALLOWED_ATTR: [
        'href', 'src', 'alt', 'title',
        'target', 'rel'
      ],
      ALLOW_DATA_ATTR: false,
      KEEP_CONTENT: true
    });
  }

  // Remove all scripts and event handlers
  sanitizeSecure(html: string): string {
    return DOMPurify.sanitize(html, {
      FORBID_TAGS: ['script', 'style', 'iframe', 'object', 'embed'],
      FORBID_ATTR: ['onerror', 'onload', 'onclick', 'onmouseover'],
      KEEP_CONTENT: false
    });
  }
}

// React component with sanitization
const UserComment: React.FC<{ comment: string }> = ({ comment }) => {
  const sanitizer = new HTMLSanitizer();
  const sanitizedComment = sanitizer.sanitizeRichText(comment);

  return (
    <div 
      className="comment"
      dangerouslySetInnerHTML={{ __html: sanitizedComment }}
    />
  );
};
```

### Output Encoding

```typescript
// Output encoding for different contexts

class OutputEncoder {
  // HTML context encoding
  encodeHTML(str: string): string {
    return str
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#x27;')
      .replace(/\//g, '&#x2F;');
  }

  // JavaScript context encoding
  encodeJavaScript(str: string): string {
    return str
      .replace(/\\/g, '\\\\')
      .replace(/'/g, "\\'")
      .replace(/"/g, '\\"')
      .replace(/\n/g, '\\n')
      .replace(/\r/g, '\\r')
      .replace(/\t/g, '\\t')
      .replace(/</g, '\\x3C')
      .replace(/>/g, '\\x3E');
  }

  // URL context encoding
  encodeURL(str: string): string {
    return encodeURIComponent(str);
  }

  // CSS context encoding
  encodeCSS(str: string): string {
    return str.replace(/[^a-zA-Z0-9]/g, (char) => {
      const hex = char.charCodeAt(0).toString(16);
      return '\\' + hex.padStart(6, '0');
    });
  }

  // SQL context encoding (use parameterized queries instead!)
  escapeSQL(str: string): string {
    return str
      .replace(/\\/g, '\\\\')
      .replace(/'/g, "''")
      .replace(/"/g, '""')
      .replace(/\n/g, '\\n')
      .replace(/\r/g, '\\r')
      .replace(/\x00/g, '\\0')
      .replace(/\x1a/g, '\\Z');
  }
}

// Usage in different contexts
const encoder = new OutputEncoder();

// HTML context
const htmlSafe = encoder.encodeHTML(userInput);
html = `<div>${htmlSafe}</div>`;

// JavaScript context
const jsSafe = encoder.encodeJavaScript(userInput);
js = `var message = '${jsSafe}';`;

// URL context
const urlSafe = encoder.encodeURL(userInput);
url = `/search?q=${urlSafe}`;

// CSS context
const cssSafe = encoder.encodeCSS(userInput);
css = `.${cssSafe} { color: red; }`;
```

## SQL Injection Prevention

### Parameterized Queries (Best Practice)

```typescript
// SQL injection prevention with parameterized queries

// ❌ DANGEROUS: String concatenation
const unsafeQuery = async (username: string) => {
  // SQL Injection vulnerable!
  const query = `SELECT * FROM users WHERE username = '${username}'`;
  // Attack: username = "admin' OR '1'='1"
  // Result: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
  // Returns all users!
  
  return db.query(query);
};

// ✓ SAFE: Parameterized query (PostgreSQL)
const safeQueryPg = async (username: string) => {
  const query = 'SELECT * FROM users WHERE username = $1';
  const params = [username];
  
  return db.query(query, params);
  // Username treated as data, not code
  // Attack string is harmless: username = "admin' OR '1'='1"
  // Searches for user with exact username "admin' OR '1'='1"
};

// ✓ SAFE: Parameterized query (MySQL)
const safeQueryMySQL = async (username: string) => {
  const query = 'SELECT * FROM users WHERE username = ?';
  const params = [username];
  
  return db.query(query, params);
};

// ✓ SAFE: Named parameters
const safeQueryNamed = async (username: string, email: string) => {
  const query = 'SELECT * FROM users WHERE username = :username OR email = :email';
  const params = { username, email };
  
  return db.query(query, params);
};
```

### ORM Usage (TypeORM, Sequelize)

```typescript
// TypeORM: Safe by default
import { getRepository } from 'typeorm';

class UserService {
  async findByUsername(username: string) {
    const userRepo = getRepository(User);
    
    // ✓ SAFE: Parameterized automatically
    return userRepo.findOne({ where: { username } });
  }

  async searchUsers(searchTerm: string) {
    const userRepo = getRepository(User);
    
    // ✓ SAFE: Query builder with parameters
    return userRepo
      .createQueryBuilder('user')
      .where('user.username LIKE :search OR user.email LIKE :search', {
        search: `%${searchTerm}%`
      })
      .getMany();
  }

  // ❌ DANGEROUS: Raw query without parameters
  async unsafeSearch(searchTerm: string) {
    return getRepository(User)
      .query(`SELECT * FROM users WHERE username LIKE '%${searchTerm}%'`);
    // SQL Injection vulnerable!
  }

  // ✓ SAFE: Raw query with parameters
  async safeRawQuery(searchTerm: string) {
    return getRepository(User)
      .query('SELECT * FROM users WHERE username LIKE $1', [`%${searchTerm}%`]);
  }
}

// Sequelize: Safe by default
import { Sequelize, Model, DataTypes } from 'sequelize';

class UserServiceSequelize {
  async findByEmail(email: string) {
    // ✓ SAFE: Parameterized automatically
    return User.findOne({ where: { email } });
  }

  async searchUsers(searchTerm: string) {
    // ✓ SAFE: Sequelize handles escaping
    return User.findAll({
      where: {
        [Sequelize.Op.or]: [
          { username: { [Sequelize.Op.like]: `%${searchTerm}%` } },
          { email: { [Sequelize.Op.like]: `%${searchTerm}%` } }
        ]
      }
    });
  }
}
```

### Input Validation for SQL

```typescript
// Additional SQL input validation
class SQLInputValidator {
  // Validate table/column names (identifiers)
  validateIdentifier(identifier: string): string {
    // Allow only alphanumeric and underscore
    if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(identifier)) {
      throw new Error('Invalid identifier');
    }

    // Check against whitelist
    const allowedIdentifiers = ['users', 'posts', 'comments', 'id', 'username', 'email'];
    if (!allowedIdentifiers.includes(identifier)) {
      throw new Error('Identifier not allowed');
    }

    return identifier;
  }

  // Validate sort direction
  validateSortDirection(direction: string): 'ASC' | 'DESC' {
    const upper = direction.toUpperCase();
    if (upper !== 'ASC' && upper !== 'DESC') {
      throw new Error('Invalid sort direction');
    }
    return upper as 'ASC' | 'DESC';
  }

  // Validate and sanitize LIKE patterns
  sanitizeLikePattern(pattern: string): string {
    // Escape special LIKE characters
    return pattern
      .replace(/\\/g, '\\\\')
      .replace(/%/g, '\\%')
      .replace(/_/g, '\\_');
  }

  // Safe dynamic query builder
  buildSafeQuery(options: {
    table: string;
    columns: string[];
    sortBy?: string;
    sortDirection?: string;
    limit?: number;
  }): { query: string; params: any[] } {
    // Validate identifiers
    const table = this.validateIdentifier(options.table);
    const columns = options.columns.map(col => this.validateIdentifier(col));

    let query = `SELECT ${columns.join(', ')} FROM ${table}`;
    const params: any[] = [];

    // Optional sorting
    if (options.sortBy) {
      const sortColumn = this.validateIdentifier(options.sortBy);
      const direction = this.validateSortDirection(options.sortDirection || 'ASC');
      query += ` ORDER BY ${sortColumn} ${direction}`;
    }

    // Optional limit
    if (options.limit) {
      query += ' LIMIT $' + (params.length + 1);
      params.push(options.limit);
    }

    return { query, params };
  }
}

// Usage
const validator = new SQLInputValidator();

app.get('/api/users', (req, res) => {
  try {
    const { query, params } = validator.buildSafeQuery({
      table: req.query.table as string || 'users',
      columns: ['id', 'username', 'email'],
      sortBy: req.query.sortBy as string,
      sortDirection: req.query.sortDirection as string,
      limit: parseInt(req.query.limit as string) || 100
    });

    const results = await db.query(query, params);
    res.json(results);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

## XSS Prevention

### React: Automatic XSS Protection

```typescript
// React automatically escapes content in JSX

const UserProfile: React.FC<{ user: User }> = ({ user }) => {
  // ✓ SAFE: React escapes user.name automatically
  return <div>Welcome, {user.name}!</div>;
  // Input: <script>alert('XSS')</script>
  // Output: &lt;script&gt;alert('XSS')&lt;/script&gt;

  // ❌ DANGEROUS: Bypassing React's protection
  return (
    <div dangerouslySetInnerHTML={{ __html: user.bio }} />
  );
  // Only use with sanitized content!

  // ✓ SAFE: Sanitize before using dangerouslySetInnerHTML
  const sanitizer = new HTMLSanitizer();
  return (
    <div dangerouslySetInnerHTML={{ __html: sanitizer.sanitizeRichText(user.bio) }} />
  );
};

// Handling URLs
const UserLink: React.FC<{ url: string }> = ({ url }) => {
  // ❌ DANGEROUS: javascript: URLs
  return <a href={url}>Click here</a>;
  // Attack: url = "javascript:alert('XSS')"

  // ✓ SAFE: Validate URL protocol
  const sanitizeURL = (url: string): string => {
    try {
      const parsed = new URL(url);
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        return '#';
      }
      return url;
    } catch {
      return '#';
    }
  };

  return <a href={sanitizeURL(url)}>Click here</a>;
};
```

### Angular: Built-in XSS Protection

```typescript
// Angular: Automatic sanitization
import { Component } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({
  selector: 'app-user-profile',
  template: `
    <!-- ✓ SAFE: Angular escapes automatically -->
    <div>{{ userName }}</div>

    <!-- ❌ DANGEROUS: Bypassing sanitization -->
    <div [innerHTML]="unsafeBio"></div>

    <!-- ✓ SAFE: Use DomSanitizer -->
    <div [innerHTML]="safeBio"></div>

    <!-- ✓ SAFE: URL binding with sanitization -->
    <a [href]="safeURL">Link</a>
  `
})
export class UserProfileComponent {
  userName = '<script>alert("XSS")</script>';  // Automatically escaped
  unsafeBio: string;
  safeBio: SafeHtml;
  safeURL: string;

  constructor(private sanitizer: DomSanitizer) {
    // Sanitize HTML content
    const rawBio = '<p>Hello <script>alert("XSS")</script></p>';
    this.safeBio = this.sanitizer.sanitize(1, rawBio) || '';

    // Sanitize URL
    const rawURL = 'javascript:alert("XSS")';
    this.safeURL = this.sanitizer.sanitize(4, rawURL) || '#';
  }

  // For rich text, use DOMPurify
  sanitizeRichText(html: string): SafeHtml {
    const clean = DOMPurify.sanitize(html);
    return this.sanitizer.bypassSecurityTrustHtml(clean);
  }
}
```

### Server-Side XSS Prevention

```typescript
// Express: Set security headers
import helmet from 'helmet';

app.use(helmet());

// Content Security Policy
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'strict-dynamic'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", 'data:', 'https:'],
    objectSrc: ["'none'"],
    baseUri: ["'self'"]
  }
}));

// X-XSS-Protection header
app.use(helmet.xssFilter());

// Server-side sanitization before storing
class XSSPreventionService {
  sanitizeBeforeStorage(userInput: string): string {
    // Remove all HTML tags for plain text fields
    return userInput.replace(/<[^>]*>/g, '');
  }

  sanitizeRichText(html: string): string {
    // Use DOMPurify on server-side (with jsdom)
    const { JSDOM } = require('jsdom');
    const { window } = new JSDOM('');
    const DOMPurify = require('dompurify')(window);

    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'u', 'a'],
      ALLOWED_ATTR: ['href']
    });
  }
}
```

## Command Injection Prevention

```typescript
// Command injection prevention

// ❌ DANGEROUS: Executing user input as shell commands
const unsafeImageProcess = (filename: string) => {
  const { exec } = require('child_process');
  
  // Command injection vulnerable!
  exec(`convert ${filename} output.png`, (error, stdout) => {
    // Attack: filename = "image.jpg; rm -rf /"
    // Executes: convert image.jpg; rm -rf / output.png
  });
};

// ✓ SAFE: Use parameterized execution
const safeImageProcess = (filename: string) => {
  const { spawn } = require('child_process');
  
  // Validate filename first
  if (!/^[a-zA-Z0-9._-]+$/.test(filename)) {
    throw new Error('Invalid filename');
  }

  // Use spawn with array of arguments (not string)
  const process = spawn('convert', [filename, 'output.png']);
  
  process.on('close', (code) => {
    console.log(`Process exited with code ${code}`);
  });
};

// ✓ BETTER: Use libraries instead of shell commands
const bestImageProcess = async (filename: string) => {
  // Use sharp library instead of ImageMagick
  const sharp = require('sharp');
  
  await sharp(filename)
    .toFormat('png')
    .toFile('output.png');
  // No shell commands = no command injection
};

// Safe path handling
class PathValidator {
  validatePath(userPath: string): string {
    const path = require('path');
    
    // Prevent path traversal
    const normalized = path.normalize(userPath);
    const basePath = path.resolve('./uploads');
    const fullPath = path.resolve(basePath, normalized);

    // Ensure path is within allowed directory
    if (!fullPath.startsWith(basePath)) {
      throw new Error('Path traversal detected');
    }

    return fullPath;
  }
}
```

## Common Mistakes

### 1. Client-Side Only Validation

```typescript
// ❌ BAD: Only client-side validation
// React component
const LoginForm: React.FC = () => {
  const handleSubmit = (email: string, password: string) => {
    if (!email || !password) {
      alert('Please fill all fields');
      return;
    }
    
    // Send to server without server-side validation
    fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify({ email, password })
    });
  };
};

// Backend accepts anything
app.post('/api/login', (req, res) => {
  const { email, password } = req.body;
  // No validation! Attacker bypasses client-side checks
});

// ✓ GOOD: Server-side validation required
app.post('/api/login', [
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 })
], (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  // Process validated input
});
```

### 2. Blacklist Validation

```typescript
// ❌ BAD: Blacklist approach
const isInvalidUsername = (username: string): boolean => {
  const blocked = ['admin', 'root', 'system'];
  return blocked.includes(username.toLowerCase());
};
// Easy to bypass: ADMIN, Admin, admin1, etc.

// ✓ GOOD: Whitelist approach
const isValidUsername = (username: string): boolean => {
  return /^[a-z0-9_]{3,20}$/.test(username);
};
// Only allows explicitly safe characters
```

### 3. Insufficient Sanitization

```typescript
// ❌ BAD: Basic string replacement
const sanitize = (html: string): string => {
  return html
    .replace(/<script>/g, '')
    .replace(/<\/script>/g, '');
};
// Bypass: <scr<script>ipt>alert('XSS')</script>

// ✓ GOOD: Use proper sanitization library
const sanitize = (html: string): string => {
  return DOMPurify.sanitize(html);
};
```

### 4. Trusting Type Coercion

```typescript
// ❌ BAD: Assuming type safety
app.get('/api/user/:id', (req, res) => {
  const userId = req.params.id;  // String, but might be used as number
  
  // SQL injection if used in string concatenation
  const query = `SELECT * FROM users WHERE id = ${userId}`;
});

// ✓ GOOD: Explicit validation and type conversion
app.get('/api/user/:id', (req, res) => {
  const userId = parseInt(req.params.id, 10);
  
  if (isNaN(userId) || userId <= 0) {
    return res.status(400).json({ error: 'Invalid user ID' });
  }
  
  // Use parameterized query
  const query = 'SELECT * FROM users WHERE id = $1';
  db.query(query, [userId]);
});
```

### 5. Not Encoding Output

```typescript
// ❌ BAD: Inserting user input without encoding
const displayComment = (comment: string) => {
  document.getElementById('comment').innerHTML = comment;
  // XSS if comment contains <script> tags
};

// ✓ GOOD: Encode before inserting
const displayComment = (comment: string) => {
  const encoder = new OutputEncoder();
  const safe = encoder.encodeHTML(comment);
  document.getElementById('comment').innerHTML = safe;
};

// ✓ BETTER: Use textContent (auto-escapes)
const displayComment = (comment: string) => {
  document.getElementById('comment').textContent = comment;
};
```

## Best Practices

### 1. Validate at Every Layer

```typescript
// Multi-layer validation
class UserRegistrationService {
  async register(data: any) {
    // Layer 1: Type validation
    this.validateTypes(data);
    
    // Layer 2: Format validation
    this.validateFormats(data);
    
    // Layer 3: Business logic validation
    await this.validateBusinessRules(data);
    
    // Layer 4: Database constraints (automatic)
    return await this.createUser(data);
  }

  private validateTypes(data: any) {
    if (typeof data.email !== 'string') throw new Error('Email must be string');
    if (typeof data.age !== 'number') throw new Error('Age must be number');
  }

  private validateFormats(data: any) {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
      throw new Error('Invalid email format');
    }
    if (data.age < 18 || data.age > 120) {
      throw new Error('Invalid age');
    }
  }

  private async validateBusinessRules(data: any) {
    // Check if email already exists
    const existing = await db.users.findOne({ email: data.email });
    if (existing) {
      throw new Error('Email already registered');
    }
  }
}
```

### 2. Use Type-Safe ORMs

```typescript
// Type-safe database operations
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity()
class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 255 })
  email: string;

  @Column({ length: 100 })
  username: string;
}

// Type-safe queries
const userRepo = getRepository(User);

// ✓ TypeScript ensures type safety
const user = await userRepo.findOne({ where: { email: validatedEmail } });

// Compile-time error if wrong type
// const user = await userRepo.findOne({ where: { email: 123 } });  // Error!
```

### 3. Sanitize on Input, Encode on Output

```typescript
// Input: Sanitize when receiving
app.post('/api/comment', (req, res) => {
  const rawComment = req.body.comment;
  
  // Sanitize HTML
  const sanitizedComment = DOMPurify.sanitize(rawComment);
  
  // Store sanitized version
  await db.comments.create({ content: sanitizedComment });
});

// Output: Encode based on context
app.get('/api/comments', async (req, res) => {
  const comments = await db.comments.findAll();
  
  // Comments already sanitized on input
  // Client will handle output encoding based on display context
  res.json(comments);
});

// React: Auto-encodes in JSX
const Comment = ({ content }) => <div>{content}</div>;

// If using dangerouslySetInnerHTML, content already sanitized
const RichComment = ({ content }) => (
  <div dangerouslySetInnerHTML={{ __html: content }} />
);
```

### 4. Use Allowlists, Not Blacklists

```typescript
// Allowlist-based validation
class AllowlistValidator {
  private allowedFileTypes = ['.jpg', '.jpeg', '.png', '.gif', '.pdf'];
  private allowedMimeTypes = [
    'image/jpeg',
    'image/png',
    'image/gif',
    'application/pdf'
  ];

  validateFile(file: Express.Multer.File): boolean {
    // Check extension (allowlist)
    const ext = path.extname(file.originalname).toLowerCase();
    if (!this.allowedFileTypes.includes(ext)) {
      throw new Error('File type not allowed');
    }

    // Check MIME type (allowlist)
    if (!this.allowedMimeTypes.includes(file.mimetype)) {
      throw new Error('MIME type not allowed');
    }

    // Verify actual file content matches MIME type
    const actualType = this.detectFileType(file.buffer);
    if (actualType !== file.mimetype) {
      throw new Error('File content does not match declared type');
    }

    return true;
  }

  private detectFileType(buffer: Buffer): string {
    // Check file signature (magic numbers)
    const signatures: Record<string, number[]> = {
      'image/jpeg': [0xFF, 0xD8, 0xFF],
      'image/png': [0x89, 0x50, 0x4E, 0x47],
      'image/gif': [0x47, 0x49, 0x46],
      'application/pdf': [0x25, 0x50, 0x44, 0x46]
    };

    for (const [mimeType, signature] of Object.entries(signatures)) {
      if (this.matchesSignature(buffer, signature)) {
        return mimeType;
      }
    }

    throw new Error('Unknown file type');
  }

  private matchesSignature(buffer: Buffer, signature: number[]): boolean {
    for (let i = 0; i < signature.length; i++) {
      if (buffer[i] !== signature[i]) {
        return false;
      }
    }
    return true;
  }
}
```

## When to Use/Not to Use

### When to Validate

**Always validate:**
- All user input
- URL parameters
- Request headers
- File uploads
- API request bodies
- Database query results (from untrusted sources)

**Server-side validation required:**
- Security-critical operations
- Data persistence
- Business logic enforcement
- Financial transactions

**Client-side validation optional:**
- UX improvement
- Immediate feedback
- Reduced server load
- Should mirror server-side rules

### When to Sanitize

**Sanitize on input:**
- HTML content from rich text editors
- User-generated content for storage
- Data from external APIs

**Encode on output:**
- Displaying user content in HTML
- Inserting into JavaScript
- Building URLs
- Generating CSS

### Validation vs Sanitization

```
┌─────────────────────────────────────────────────────────────┐
│         Validation vs Sanitization                           │
└─────────────────────────────────────────────────────────────┘

Validation:
  Purpose: Check if input meets requirements
  Action:  Accept or reject
  Example: Is email format valid? Yes/No
  Use:     User registration, form submission

Sanitization:
  Purpose: Make input safe for use
  Action:  Transform/clean
  Example: Remove <script> tags from HTML
  Use:     Rich text content, user comments

Both:
  Email input:
    1. Validate format (is it email-like?)
    2. Sanitize (normalize to lowercase, trim)
    3. Validate again (check length, allowed chars)
```

## Interview Questions

### Q1: What's the difference between validation and sanitization?

**Answer:**

**Validation:** Checks if input meets requirements and rejects invalid input.
- Returns true/false or throws error
- Example: "Is this a valid email?" → Yes or No
- Input unchanged, just verified

**Sanitization:** Transforms input to make it safe.
- Removes/encodes dangerous content
- Example: Remove `<script>` tags from HTML
- Input modified to be safe

**When to use:**
- Validation: User registration, form submission (reject bad input)
- Sanitization: Rich text content, comments (accept but clean)

**Best practice:** Often use both together:
```typescript
// 1. Validate format
if (!emailRegex.test(email)) reject();

// 2. Sanitize
email = email.trim().toLowerCase();

// 3. Validate again
if (email.length > 255) reject();
```

### Q2: Why is client-side validation alone insufficient for security?

**Answer:** Client-side validation can be completely bypassed:

**Bypass methods:**
1. **Browser DevTools:** Modify JavaScript, disable validation
2. **Direct API calls:** Use curl, Postman to skip frontend
3. **Proxy tools:** Intercept and modify requests (Burp Suite)
4. **Browser extensions:** Auto-fill invalid data

**Example attack:**
```typescript
// Frontend validation
if (age < 18) return alert('Must be 18+');

// Attacker bypasses by calling API directly:
fetch('/api/register', {
  body: JSON.stringify({ age: 10 })  // Validation skipped!
});
```

**Correct approach:**
- Client-side: UX enhancement only
- Server-side: Security enforcement (always required)
- Database: Final constraint layer

### Q3: Explain parameterized queries and why they prevent SQL injection.

**Answer:** Parameterized queries separate SQL code from data, preventing injection.

**How it works:**
```typescript
// Vulnerable (string concatenation)
const query = `SELECT * FROM users WHERE username = '${username}'`;
// Attack: username = "admin' OR '1'='1"
// Result: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
// Returns all users!

// Safe (parameterized)
const query = 'SELECT * FROM users WHERE username = $1';
const params = [username];
// Attack: username = "admin' OR '1'='1"
// Result: SELECT * FROM users WHERE username = 'admin'' OR ''1''=''1'
// Searches for literal string "admin' OR '1'='1"
```

**Why secure:**
1. **Separation:** SQL structure sent separately from data
2. **Escaping:** Database driver properly escapes data
3. **Type enforcement:** Data treated as data, not code
4. **Parser protection:** SQL parser doesn't interpret data as commands

**Always use parameterized queries**, never string concatenation for SQL.

### Q4: How do you safely handle user-uploaded files?

**Answer:** Multiple validation layers for file uploads:

**1. Validate file type:**
```typescript
// Check extension (allowlist)
const allowedExtensions = ['.jpg', '.png', '.pdf'];
const ext = path.extname(file.originalname);
if (!allowedExtensions.includes(ext)) reject();

// Check MIME type
const allowedMimeTypes = ['image/jpeg', 'image/png'];
if (!allowedMimeTypes.includes(file.mimetype)) reject();

// Verify file signature (magic numbers)
const signature = file.buffer.slice(0, 4);
if (!matchesJPEGSignature(signature)) reject();
```

**2. Validate file size:**
```typescript
const maxSize = 5 * 1024 * 1024; // 5MB
if (file.size > maxSize) reject();
```

**3. Scan for malware:**
```typescript
const virusScan = await clamav.scanBuffer(file.buffer);
if (virusScan.isInfected) reject();
```

**4. Generate safe filename:**
```typescript
// Never use original filename directly
const safeFilename = `${uuid()}.${ext}`;
```

**5. Store outside web root:**
```typescript
// Store in non-public directory
const uploadPath = '/var/uploads/';  // Not in /public/
```

**6. Serve with correct headers:**
```typescript
res.setHeader('Content-Type', 'application/octet-stream');
res.setHeader('Content-Disposition', 'attachment');
res.setHeader('X-Content-Type-Options', 'nosniff');
```

### Q5: What is output encoding and why is it important?

**Answer:** Output encoding transforms data to be safe for a specific context.

**Why needed:** Same data requires different encoding based on where it's displayed:

```typescript
const userInput = '<script>alert("XSS")</script>';

// HTML context
htmlSafe = '&lt;script&gt;alert("XSS")&lt;/script&gt;'
// Displays as text, not executed

// JavaScript context
jsSafe = '<scr\\x3Eipt>alert(\\"XSS\\")<\\/script>'
// Safe inside JS string

// URL context
urlSafe = '%3Cscript%3Ealert(%22XSS%22)%3C%2Fscript%3E'
// Safe in URL parameter

// CSS context
cssSafe = '\\00003c\\000073\\000063...'
// Safe in CSS value
```

**Context matters:** Encoding for HTML won't protect against XSS in JavaScript context.

**Best practice:**
- Use framework auto-escaping (React JSX, Angular templates)
- Encode manually when necessary
- Choose encoding based on output context
- Sanitize on input, encode on output

### Q6: How do you prevent command injection attacks?

**Answer:** Never execute user input as shell commands:

**Vulnerable approach:**
```typescript
// DANGEROUS
exec(`convert ${userFilename} output.png`);
// Attack: userFilename = "image.jpg; rm -rf /"
```

**Safe approaches:**

**1. Use spawn with argument array:**
```typescript
spawn('convert', [userFilename, 'output.png']);
// Arguments treated as data, not code
```

**2. Validate input strictly:**
```typescript
if (!/^[a-zA-Z0-9._-]+$/.test(filename)) {
  throw new Error('Invalid filename');
}
```

**3. Use libraries instead of shell:**
```typescript
// Instead of ImageMagick CLI
const sharp = require('sharp');
await sharp(filename).toFormat('png').toFile('output.png');
// No shell commands = no injection
```

**4. Whitelist commands:**
```typescript
const allowedCommands = ['convert', 'identify'];
if (!allowedCommands.includes(command)) reject();
```

**Best practice:** Avoid shell commands entirely when possible.

### Q7: What's the principle of "least privilege" in input validation?

**Answer:** Only accept the minimum necessary input; reject everything else.

**Examples:**

**1. Field length:**
```typescript
// Don't accept arbitrarily long input
username: { minLength: 3, maxLength: 20 }  // Not unlimited
```

**2. Character whitelist:**
```typescript
// Only allow necessary characters
username: /^[a-zA-Z0-9_-]+$/  // Not all Unicode
```

**3. Numeric ranges:**
```typescript
age: { min: 0, max: 150 }  // Not any number
```

**4. File types:**
```typescript
// Only allow needed formats
allowedTypes: ['.jpg', '.png']  // Not all extensions
```

**5. API fields:**
```typescript
// Only accept expected fields
const { email, password } = req.body;  // Ignore extra fields
```

**Benefits:**
- Reduces attack surface
- Easier to validate comprehensively
- Clearer error messages
- Better documentation

**Anti-pattern:** "Accept everything, filter bad stuff" (blacklist approach)

### Q8: How do you validate JSON input securely?

**Answer:** Multi-layer JSON validation:

**1. Parse safely:**
```typescript
try {
  const data = JSON.parse(req.body);
} catch (error) {
  return res.status(400).json({ error: 'Invalid JSON' });
}
```

**2. Validate structure:**
```typescript
// Use JSON schema validation
const schema = {
  type: 'object',
  required: ['email', 'age'],
  properties: {
    email: { type: 'string', format: 'email' },
    age: { type: 'integer', minimum: 18, maximum: 120 },
    bio: { type: 'string', maxLength: 500 }
  },
  additionalProperties: false  // Reject extra fields
};

const valid = ajv.validate(schema, data);
if (!valid) return res.status(400).json({ errors: ajv.errors });
```

**3. Validate content:**
```typescript
// Sanitize string fields
data.bio = DOMPurify.sanitize(data.bio);

// Validate email format
if (!emailRegex.test(data.email)) reject();
```

**4. Limit size:**
```typescript
app.use(express.json({ limit: '100kb' }));  // Prevent DoS
```

**5. Prevent prototype pollution:**
```typescript
// Avoid Object.assign, use explicit assignment
const user = {
  email: data.email,
  age: data.age,
  bio: data.bio
};
// Don't: Object.assign({}, data)  // Can pollute prototype
```

**Libraries:** joi, ajv, class-validator for comprehensive validation.

## Key Takeaways

1. **Never trust client-side validation** - Always validate on server; client-side is UX only, trivially bypassed
2. **Use parameterized queries exclusively** - Never concatenate user input into SQL strings; prevents injection attacks
3. **Whitelist validation over blacklist** - Explicitly allow safe patterns rather than trying to block dangerous ones
4. **Sanitize input, encode output** - Clean data when received, encode appropriately when displayed based on context
5. **Validate at every layer** - Client UX, server security, database constraints create defense in depth
6. **Use proven libraries** - DOMPurify for HTML, express-validator for inputs, ORMs for SQL; don't roll your own
7. **Apply principle of least privilege** - Only accept minimum necessary input; reject everything else by default
8. **Context-aware encoding** - HTML encoding differs from JavaScript/URL/CSS; use appropriate method for display context
9. **Avoid shell commands** - Use libraries instead of exec/spawn when possible; if necessary, validate rigorously
10. **Comprehensive file upload validation** - Check type (extension, MIME, signature), size, scan malware, generate safe names

## Resources

### Official Documentation
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

### Libraries and Tools
- [express-validator](https://express-validator.github.io/) - Express validation middleware
- [DOMPurify](https://github.com/cure53/DOMPurify) - HTML sanitization
- [joi](https://joi.dev/) - Schema validation for Node.js
- [ajv](https://ajv.js.org/) - JSON schema validator
- [class-validator](https://github.com/typestack/class-validator) - TypeScript/JavaScript validation

### Articles and Guides
- [SQL Injection Attack Tutorial](https://www.acunetix.com/websitesecurity/sql-injection/)
- [XSS Attack Examples](https://owasp.org/www-community/attacks/xss/)
- [Command Injection](https://owasp.org/www-community/attacks/Command_Injection)

### Testing Tools
- [SQLMap](http://sqlmap.org/) - SQL injection testing tool
- [Burp Suite](https://portswigger.net/burp) - Web security testing
- [OWASP ZAP](https://www.zaproxy.org/) - Security scanner
