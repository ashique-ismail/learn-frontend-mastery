# AJV (Another JSON Schema Validator)

## The Idea

**In plain English:** AJV is a tool that checks whether a piece of data (like a form submission or an API response) matches a set of rules you define — for example, "the email field must be a valid email address and the age must be a number between 18 and 120." A schema is just a written description of what valid data is supposed to look like.

**Real-world analogy:** Imagine a nightclub bouncer holding a checklist before letting anyone in: the list says guests must be over 18, must have a valid ID, and must be on the invite list. AJV works the same way for your data.

- The checklist = the JSON Schema (the rules you write)
- The bouncer = the compiled validator function (the code that runs the check)
- Each guest trying to enter = a piece of data being validated
- Being turned away = a validation error

---

## Overview

AJV is the fastest JSON Schema validator for Node.js and browsers. It implements JSON Schema specifications (draft-07, draft-2019-09, draft-2020-12) and provides exceptional performance through schema compilation and code generation. AJV is widely used for API validation, configuration validation, and data validation in high-performance applications.

## Installation and Setup

```bash
# Install AJV
npm install ajv

# Install with JSON Schema formats
npm install ajv ajv-formats

# Install with additional keywords
npm install ajv ajv-keywords

# TypeScript types
npm install ajv @types/ajv
```

## Core Concepts

### Basic Schema and Validation

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// Define JSON Schema
const schema = {
  type: 'object',
  properties: {
    username: { type: 'string', minLength: 3, maxLength: 30 },
    email: { type: 'string', format: 'email' },
    age: { type: 'integer', minimum: 18, maximum: 120 },
    isActive: { type: 'boolean' },
  },
  required: ['username', 'email', 'age'],
  additionalProperties: false,
};

// Compile schema to validation function
const validate = ajv.compile(schema);

// Validate data
const data = {
  username: 'john_doe',
  email: 'john@example.com',
  age: 25,
  isActive: true,
};

const valid = validate(data);

if (!valid) {
  console.error('Validation errors:', validate.errors);
} else {
  console.log('Valid data:', data);
}
```

### JSON Schema Types

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// String schema
const stringSchema = {
  type: 'string',
  minLength: 3,
  maxLength: 50,
  pattern: '^[a-zA-Z]+$',
};

// Number schema
const numberSchema = {
  type: 'number',
  minimum: 0,
  maximum: 100,
  multipleOf: 5,
  exclusiveMinimum: 0,
  exclusiveMaximum: 100,
};

// Integer schema
const integerSchema = {
  type: 'integer',
  minimum: 1,
  maximum: 1000,
};

// Boolean schema
const booleanSchema = {
  type: 'boolean',
};

// Array schema
const arraySchema = {
  type: 'array',
  items: { type: 'string' },
  minItems: 1,
  maxItems: 10,
  uniqueItems: true,
};

// Object schema
const objectSchema = {
  type: 'object',
  properties: {
    name: { type: 'string' },
    age: { type: 'integer' },
  },
  required: ['name'],
  additionalProperties: false,
};

// Null schema
const nullSchema = {
  type: 'null',
};

// Multiple types
const multiTypeSchema = {
  type: ['string', 'number'],
};

// Any type (no type constraint)
const anySchema = {
  // No type specified means any type is valid
};

// Enum values
const enumSchema = {
  type: 'string',
  enum: ['admin', 'user', 'guest'],
};

// Const value
const constSchema = {
  const: 'specific-value',
};
```

### Schema Composition

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// AllOf (must match all schemas)
const allOfSchema = {
  allOf: [
    { type: 'string' },
    { minLength: 5 },
    { maxLength: 50 },
    { pattern: '^[a-zA-Z]+$' },
  ],
};

// AnyOf (must match at least one schema)
const anyOfSchema = {
  anyOf: [
    { type: 'string', minLength: 5 },
    { type: 'number', minimum: 0 },
  ],
};

// OneOf (must match exactly one schema)
const oneOfSchema = {
  oneOf: [
    {
      type: 'object',
      properties: {
        type: { const: 'credit_card' },
        cardNumber: { type: 'string' },
        cvv: { type: 'string' },
      },
      required: ['type', 'cardNumber', 'cvv'],
    },
    {
      type: 'object',
      properties: {
        type: { const: 'paypal' },
        email: { type: 'string', format: 'email' },
      },
      required: ['type', 'email'],
    },
    {
      type: 'object',
      properties: {
        type: { const: 'bank_transfer' },
        accountNumber: { type: 'string' },
        routingNumber: { type: 'string' },
      },
      required: ['type', 'accountNumber', 'routingNumber'],
    },
  ],
};

// Not (must not match schema)
const notSchema = {
  type: 'string',
  not: {
    pattern: 'forbidden',
  },
};

// Combined composition
const complexSchema = {
  type: 'object',
  allOf: [
    {
      properties: {
        name: { type: 'string' },
      },
    },
    {
      properties: {
        age: { type: 'integer' },
      },
    },
  ],
  anyOf: [
    { required: ['email'] },
    { required: ['phone'] },
  ],
};
```

## Advanced Features

### Schema References ($ref)

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// Define reusable schemas
const addressSchema = {
  $id: 'https://example.com/schemas/address.json',
  type: 'object',
  properties: {
    street: { type: 'string' },
    city: { type: 'string' },
    state: { type: 'string', minLength: 2, maxLength: 2 },
    zipCode: { type: 'string', pattern: '^\\d{5}(-\\d{4})?$' },
  },
  required: ['street', 'city', 'state', 'zipCode'],
};

// Add schema to AJV instance
ajv.addSchema(addressSchema);

// Use schema reference
const userSchema = {
  type: 'object',
  properties: {
    name: { type: 'string' },
    email: { type: 'string', format: 'email' },
    shippingAddress: { $ref: 'https://example.com/schemas/address.json' },
    billingAddress: { $ref: 'https://example.com/schemas/address.json' },
  },
  required: ['name', 'email', 'shippingAddress'],
};

// Internal references
const schemaWithInternalRefs = {
  $id: 'https://example.com/schemas/user.json',
  type: 'object',
  definitions: {
    positiveInteger: {
      type: 'integer',
      minimum: 1,
    },
    email: {
      type: 'string',
      format: 'email',
    },
  },
  properties: {
    userId: { $ref: '#/definitions/positiveInteger' },
    email: { $ref: '#/definitions/email' },
    age: { $ref: '#/definitions/positiveInteger' },
  },
};

// Recursive schema with $ref
const treeSchema = {
  $id: 'https://example.com/schemas/tree.json',
  type: 'object',
  properties: {
    value: { type: 'number' },
    children: {
      type: 'array',
      items: { $ref: '#' }, // References root schema
    },
  },
};
```

### Conditional Schemas (if/then/else)

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// Simple if/then/else
const conditionalSchema = {
  type: 'object',
  properties: {
    country: { type: 'string' },
    postalCode: { type: 'string' },
  },
  if: {
    properties: { country: { const: 'US' } },
  },
  then: {
    properties: {
      postalCode: { pattern: '^\\d{5}(-\\d{4})?$' },
    },
  },
  else: {
    properties: {
      postalCode: { pattern: '^[A-Z0-9]{3,10}$' },
    },
  },
};

// Multiple conditions
const accountSchema = {
  type: 'object',
  properties: {
    accountType: { type: 'string', enum: ['personal', 'business'] },
    companyName: { type: 'string' },
    taxId: { type: 'string' },
  },
  required: ['accountType'],
  if: {
    properties: { accountType: { const: 'business' } },
  },
  then: {
    required: ['companyName', 'taxId'],
    properties: {
      taxId: { pattern: '^\\d{2}-\\d{7}$' },
    },
  },
};

// Nested conditions
const discountSchema = {
  type: 'object',
  properties: {
    customerType: { type: 'string', enum: ['regular', 'premium', 'vip'] },
    purchaseAmount: { type: 'number', minimum: 0 },
    discount: { type: 'number', minimum: 0, maximum: 100 },
  },
  if: {
    properties: { customerType: { const: 'regular' } },
  },
  then: {
    properties: { discount: { maximum: 5 } },
  },
  else: {
    if: {
      properties: { customerType: { const: 'premium' } },
    },
    then: {
      properties: { discount: { maximum: 15 } },
    },
    else: {
      properties: { discount: { maximum: 30 } },
    },
  },
};
```

### Custom Formats

```javascript
const Ajv = require('ajv');
const addFormats = require('ajv-formats');

const ajv = new Ajv();
addFormats(ajv);

// Built-in formats from ajv-formats
const schemaWithFormats = {
  type: 'object',
  properties: {
    email: { type: 'string', format: 'email' },
    url: { type: 'string', format: 'uri' },
    date: { type: 'string', format: 'date' },
    time: { type: 'string', format: 'time' },
    datetime: { type: 'string', format: 'date-time' },
    ipv4: { type: 'string', format: 'ipv4' },
    ipv6: { type: 'string', format: 'ipv6' },
    uuid: { type: 'string', format: 'uuid' },
  },
};

// Add custom format
ajv.addFormat('phone', {
  type: 'string',
  validate: (data) => {
    return /^\+?1?\s*\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})$/.test(data);
  },
});

// Custom format with detailed validation
ajv.addFormat('credit-card', {
  type: 'string',
  validate: (data) => {
    // Remove spaces and dashes
    const cleaned = data.replace(/[\s-]/g, '');
    
    // Check if only digits
    if (!/^\d+$/.test(cleaned)) return false;
    
    // Check length
    if (cleaned.length < 13 || cleaned.length > 19) return false;
    
    // Luhn algorithm
    let sum = 0;
    let isEven = false;
    
    for (let i = cleaned.length - 1; i >= 0; i--) {
      let digit = parseInt(cleaned[i], 10);
      
      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }
      
      sum += digit;
      isEven = !isEven;
    }
    
    return sum % 10 === 0;
  },
});

// Async format validation
ajv.addFormat('unique-email', {
  type: 'string',
  async: true,
  validate: async (data) => {
    // Simulate database check
    const response = await fetch(`/api/check-email?email=${data}`);
    const { available } = await response.json();
    return available;
  },
});

// Use custom formats
const customFormatSchema = {
  type: 'object',
  properties: {
    phone: { type: 'string', format: 'phone' },
    creditCard: { type: 'string', format: 'credit-card' },
    email: { type: 'string', format: 'unique-email' },
  },
};
```

### Custom Keywords

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// Add custom keyword
ajv.addKeyword({
  keyword: 'range',
  type: 'number',
  schemaType: 'array',
  compile: ([min, max]) => {
    return (data) => data >= min && data <= max;
  },
  error: {
    message: ({ schemaCode }) => {
      return `should be in range ${schemaCode}`;
    },
  },
});

// Use custom keyword
const rangeSchema = {
  type: 'object',
  properties: {
    age: { type: 'number', range: [18, 65] },
  },
};

// Custom keyword with metaschema
ajv.addKeyword({
  keyword: 'isEven',
  type: 'number',
  schemaType: 'boolean',
  compile: (schemaValue) => {
    return (data) => {
      if (schemaValue) {
        return data % 2 === 0;
      }
      return true;
    };
  },
  metaSchema: {
    type: 'boolean',
  },
  error: {
    message: 'must be an even number',
  },
});

// Macro keyword (expands to JSON Schema)
ajv.addKeyword({
  keyword: 'range-macro',
  type: 'number',
  macro: ([min, max]) => {
    return {
      minimum: min,
      maximum: max,
    };
  },
});

// Custom keyword with validation function
ajv.addKeyword({
  keyword: 'divisibleBy',
  type: 'number',
  schemaType: 'number',
  validate: function validate(schema, data) {
    if (data % schema !== 0) {
      validate.errors = [
        {
          keyword: 'divisibleBy',
          message: `should be divisible by ${schema}`,
          params: { divisor: schema },
        },
      ];
      return false;
    }
    return true;
  },
});
```

### Asynchronous Validation

```javascript
const Ajv = require('ajv');
const ajv = new Ajv();

// Async custom keyword
ajv.addKeyword({
  keyword: 'uniqueUsername',
  async: true,
  type: 'string',
  validate: async (schema, data) => {
    // Simulate database check
    const response = await fetch(`/api/check-username?username=${data}`);
    const { available } = await response.json();
    return available;
  },
  errors: false,
});

// Schema with async validation
const asyncSchema = {
  type: 'object',
  properties: {
    username: {
      type: 'string',
      uniqueUsername: true,
    },
    email: {
      type: 'string',
      format: 'email',
    },
  },
};

// Validate asynchronously
const validate = ajv.compile(asyncSchema);

async function validateUser(data) {
  try {
    const valid = await validate(data);
    if (!valid) {
      console.error('Validation errors:', validate.errors);
      return false;
    }
    return true;
  } catch (error) {
    console.error('Async validation error:', error);
    return false;
  }
}

// Usage
validateUser({
  username: 'newuser',
  email: 'newuser@example.com',
}).then(valid => {
  console.log('Validation result:', valid);
});
```

## Performance Optimization

### Schema Compilation and Caching

```javascript
const Ajv = require('ajv');

// Enable compilation caching
const ajv = new Ajv({
  code: { source: true, optimize: true },
  cache: new Map(), // Enable caching
});

// Compile once, use many times
const userSchema = { /* schema definition */ };
const validateUser = ajv.compile(userSchema);

// Use compiled validator repeatedly
for (const user of users) {
  const valid = validateUser(user);
  if (!valid) {
    console.error(validateUser.errors);
  }
}

// Serialize compiled schema
const standaloneCode = require('ajv/dist/standalone').default;
const moduleCode = standaloneCode(ajv, validateUser);

// Save to file for faster startup
const fs = require('fs');
fs.writeFileSync('validate.js', moduleCode);

// Use pre-compiled validator
const validatePrecompiled = require('./validate');
```

### Strict Mode and Optimization

```javascript
const Ajv = require('ajv');

// Strict mode for better performance
const ajv = new Ajv({
  strict: true, // Strict schema validation
  allErrors: false, // Stop on first error (faster)
  verbose: false, // Less error details (faster)
  validateFormats: true, // Validate formats
  removeAdditional: true, // Remove additional properties
  useDefaults: true, // Apply default values
  coerceTypes: false, // Disable type coercion (faster)
});

// JIT compilation (fastest)
const ajvJIT = new Ajv({
  code: { optimize: true, source: true },
  allErrors: false,
});

// Optimize for production
const ajvProduction = new Ajv({
  code: { optimize: true },
  allErrors: false,
  verbose: false,
  strict: true,
  validateFormats: false, // Skip format validation if not needed
  removeAdditional: 'all', // Remove all additional properties
});
```

## Express.js Integration

```javascript
const express = require('express');
const Ajv = require('ajv');
const addFormats = require('ajv-formats');

const app = express();
app.use(express.json());

const ajv = new Ajv({ allErrors: true, removeAdditional: true });
addFormats(ajv);

// Generic validation middleware
function validateSchema(schema) {
  const validate = ajv.compile(schema);
  
  return (req, res, next) => {
    const valid = validate(req.body);
    
    if (!valid) {
      return res.status(400).json({
        success: false,
        errors: validate.errors.map(err => ({
          field: err.instancePath.slice(1).replace(/\//g, '.'),
          message: err.message,
          params: err.params,
        })),
      });
    }
    
    // Replace body with validated/cleaned data
    req.body = validate.data || req.body;
    next();
  };
}

// Define schemas
const createUserSchema = {
  type: 'object',
  properties: {
    username: { type: 'string', minLength: 3, maxLength: 30 },
    email: { type: 'string', format: 'email' },
    password: { type: 'string', minLength: 8 },
    role: { type: 'string', enum: ['user', 'admin'], default: 'user' },
  },
  required: ['username', 'email', 'password'],
  additionalProperties: false,
};

const updateUserSchema = {
  type: 'object',
  properties: {
    username: { type: 'string', minLength: 3, maxLength: 30 },
    email: { type: 'string', format: 'email' },
    role: { type: 'string', enum: ['user', 'admin'] },
  },
  minProperties: 1,
  additionalProperties: false,
};

// Apply middleware
app.post('/api/users', validateSchema(createUserSchema), (req, res) => {
  res.json({ success: true, user: req.body });
});

app.put('/api/users/:id', validateSchema(updateUserSchema), (req, res) => {
  res.json({ success: true, id: req.params.id, updates: req.body });
});

// Validate query parameters
const searchSchema = {
  type: 'object',
  properties: {
    q: { type: 'string', minLength: 1 },
    page: { type: 'integer', minimum: 1, default: 1 },
    limit: { type: 'integer', minimum: 1, maximum: 100, default: 20 },
  },
  required: ['q'],
};

function validateQuery(schema) {
  const validate = ajv.compile(schema);
  
  return (req, res, next) => {
    // Convert query strings to appropriate types
    const query = { ...req.query };
    if (query.page) query.page = parseInt(query.page, 10);
    if (query.limit) query.limit = parseInt(query.limit, 10);
    
    const valid = validate(query);
    
    if (!valid) {
      return res.status(400).json({
        success: false,
        errors: validate.errors,
      });
    }
    
    req.query = query;
    next();
  };
}

app.get('/api/search', validateQuery(searchSchema), (req, res) => {
  res.json({ success: true, query: req.query });
});
```

## TypeScript Integration

```typescript
import Ajv, { JSONSchemaType } from 'ajv';

const ajv = new Ajv();

// Define TypeScript interface
interface User {
  id: number;
  username: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
  profile?: {
    firstName: string;
    lastName: string;
  };
}

// Create type-safe schema
const userSchema: JSONSchemaType<User> = {
  type: 'object',
  properties: {
    id: { type: 'number' },
    username: { type: 'string' },
    email: { type: 'string', format: 'email' },
    role: { type: 'string', enum: ['admin', 'user', 'guest'] },
    profile: {
      type: 'object',
      properties: {
        firstName: { type: 'string' },
        lastName: { type: 'string' },
      },
      required: ['firstName', 'lastName'],
      nullable: true,
    },
  },
  required: ['id', 'username', 'email', 'role'],
  additionalProperties: false,
};

// Compile with type safety
const validate = ajv.compile(userSchema);

// Type guard function
function isUser(data: unknown): data is User {
  return validate(data);
}

// Usage
const userData: unknown = { /* data */ };

if (isUser(userData)) {
  // TypeScript knows userData is User type
  console.log(userData.username);
}
```

## Common Mistakes

1. **Not compiling schemas**: Calling `ajv.validate(schema, data)` repeatedly instead of compiling once.

2. **Ignoring performance options**: Not using `allErrors: false` or code optimization in production.

3. **Missing format validation**: Not installing `ajv-formats` package when using format keywords.

4. **Incorrect $ref usage**: Using relative paths without proper $id or base URI configuration.

5. **Not handling async validation**: Using synchronous `validate()` with async keywords.

6. **Excessive strictness**: Enabling strict mode without understanding implications.

7. **Type coercion confusion**: Expecting automatic type conversion without `coerceTypes: true`.

8. **Poor error handling**: Not properly formatting AJV errors for user consumption.

9. **Schema bloat**: Creating overly complex schemas instead of using $ref and composition.

10. **Not caching compiled validators**: Recompiling schemas on every request in production.

## Best Practices

1. **Compile schemas once**: Store compiled validators and reuse them for better performance.

2. **Use $ref for reusability**: Define common schemas once and reference them.

3. **Enable appropriate optimizations**: Use JIT compilation and code generation for production.

4. **Handle errors gracefully**: Transform AJV errors into user-friendly messages.

5. **Leverage formats**: Use built-in and custom formats for common validation patterns.

6. **Use strict mode carefully**: Enable for schema development, consider disabling for runtime.

7. **Implement proper TypeScript typing**: Use JSONSchemaType for type-safe schemas.

8. **Cache validation results**: For expensive validations, cache results when appropriate.

9. **Document schemas**: Add descriptions and examples to schema definitions.

10. **Test schemas thoroughly**: Write unit tests for complex validation logic.

## When to Use AJV

**Use AJV when:**

- Need maximum validation performance
- Working with JSON Schema standard
- Validating large volumes of data
- Building high-performance APIs
- Need OpenAPI/Swagger integration
- Want code generation for validators
- Validating configuration files

**Consider alternatives when:**

- Need simpler API for forms (use Yup or Zod)
- Want TypeScript-first approach (use Zod)
- Building React forms (use Yup with Formik)
- Need minimal learning curve (use Joi)
- Want functional validation (use Vest)

## Interview Questions

### Q1: What makes AJV faster than other validation libraries?

**Answer**: AJV achieves superior performance through: 1) Schema compilation to optimized JavaScript functions, 2) JIT compilation with code generation, 3) Early termination with `allErrors: false`, 4) Optional features that can be disabled, 5) Efficient schema caching, 6) Optimized validation algorithms. AJV can be 10-100x faster than alternatives like Joi or Yup.

### Q2: How do you implement conditional validation in AJV?

**Answer**: Use `if/then/else` keywords from JSON Schema:

```javascript
const schema = {
  type: 'object',
  properties: {
    accountType: { type: 'string' },
    companyName: { type: 'string' },
  },
  if: {
    properties: { accountType: { const: 'business' } },
  },
  then: {
    required: ['companyName'],
  },
};
```

Alternatively, use `oneOf` for multiple mutually exclusive conditions.

### Q3: How do you create and use custom formats in AJV?

**Answer**: Use `addFormat()` method:

```javascript
ajv.addFormat('phone', {
  type: 'string',
  validate: (data) => /^\d{10}$/.test(data),
});

const schema = {
  type: 'object',
  properties: {
    phone: { type: 'string', format: 'phone' },
  },
};
```

For async validation, set `async: true` in the format definition.

### Q4: What's the difference between `$ref` and schema composition (allOf, anyOf, oneOf)?

**Answer**: `$ref` references an entire schema defined elsewhere, promoting reusability. Composition keywords (`allOf`, `anyOf`, `oneOf`) combine multiple schema constraints: `allOf` requires all schemas to match, `anyOf` requires at least one, and `oneOf` requires exactly one. Use `$ref` for reusability, composition for logical combinations.

### Q5: How do you optimize AJV validation for production?

**Answer**: Strategies include: 1) Compile schemas once at startup, 2) Enable JIT compilation with `code: { optimize: true }`, 3) Use `allErrors: false` to stop at first error, 4) Disable unnecessary features like `verbose` and `validateFormats`, 5) Use standalone code generation for deployment, 6) Enable `removeAdditional` to strip unknown properties, 7) Cache validation results for identical inputs.

### Q6: How do you handle asynchronous validation in AJV?

**Answer**: Create async custom keywords or formats:

```javascript
ajv.addKeyword({
  keyword: 'uniqueEmail',
  async: true,
  validate: async (schema, data) => {
    const response = await checkDatabase(data);
    return response.available;
  },
});

// Use with async validation
const validate = ajv.compile(schema);
const valid = await validate(data);
```

Note that async validation is slower and should be used sparingly.

### Q7: How do you integrate AJV with Express.js for API validation?

**Answer**: Create validation middleware that compiles schemas once and validates request data:

```javascript
function validateSchema(schema) {
  const validate = ajv.compile(schema);
  return (req, res, next) => {
    const valid = validate(req.body);
    if (!valid) {
      return res.status(400).json({ errors: validate.errors });
    }
    req.body = validate.data || req.body;
    next();
  };
}

app.post('/api/users', validateSchema(userSchema), handler);
```

## Key Takeaways

1. AJV is the fastest JSON Schema validator, achieving performance through schema compilation and code generation.

2. Compile schemas once using `ajv.compile()` and reuse the validator function for optimal performance.

3. JSON Schema standard provides powerful composition with `allOf`, `anyOf`, `oneOf`, and `not` keywords.

4. Conditional validation is implemented using `if/then/else` keywords from JSON Schema draft-07 and later.

5. Schema references (`$ref`) enable reusability and modularity in complex validation schemas.

6. Custom formats and keywords extend AJV's validation capabilities for domain-specific requirements.

7. Production optimization includes JIT compilation, `allErrors: false`, and disabling verbose error reporting.

8. TypeScript integration through `JSONSchemaType` provides compile-time type safety for schemas.

9. Async validation is supported but should be used sparingly due to performance impact.

10. AJV integrates seamlessly with Express.js through middleware that compiles and caches validation functions.

## Resources

- [Official Documentation](https://ajv.js.org/)
- [JSON Schema Specification](https://json-schema.org/)
- [AJV GitHub Repository](https://github.com/ajv-validator/ajv)
- [ajv-formats Package](https://github.com/ajv-validator/ajv-formats)
- [ajv-keywords Package](https://github.com/ajv-validator/ajv-keywords)
- [Performance Benchmarks](https://github.com/ebdrup/json-schema-benchmark)
- [JSON Schema Tools](https://json-schema.org/implementations.html)
