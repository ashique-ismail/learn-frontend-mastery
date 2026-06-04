# Zod - TypeScript-First Schema Validation

## Overview

Zod is a TypeScript-first schema declaration and validation library that provides runtime type checking, type inference, and data parsing. Unlike other validation libraries, Zod is designed to work seamlessly with TypeScript's type system, offering zero-cost type safety with excellent developer experience, composable schemas, and powerful transformation capabilities.

## Installation

```bash
# Install Zod
npm install zod

# TypeScript (should already be installed)
npm install -D typescript
```

## Core Concepts

### 1. Schema Declaration

Schemas define the shape and validation rules for your data.

### 2. Type Inference

TypeScript types are automatically inferred from schemas.

### 3. Parsing vs Validation

- **parse()**: Throws error on validation failure
- **safeParse()**: Returns success/failure object

### 4. Refinements

Custom validation logic beyond basic types.

### 5. Transformations

Transform data during parsing.

## Basic Examples

### 1. Primitive Types

```typescript
import { z } from 'zod';

// String schema
const nameSchema = z.string();

// Parse (throws on error)
const name = nameSchema.parse('John'); // OK
// nameSchema.parse(123); // Throws ZodError

// Safe parse (returns result object)
const result = nameSchema.safeParse('John');
if (result.success) {
  console.log(result.data); // 'John'
} else {
  console.log(result.error); // ZodError
}

// Number schema
const ageSchema = z.number();
const age = ageSchema.parse(25); // OK

// Boolean schema
const isActiveSchema = z.boolean();
const isActive = isActiveSchema.parse(true); // OK

// Date schema
const dateSchema = z.date();
const today = dateSchema.parse(new Date()); // OK

// BigInt schema
const bigIntSchema = z.bigint();
const big = bigIntSchema.parse(BigInt(9007199254740991)); // OK

// Symbol schema
const symbolSchema = z.symbol();
const sym = symbolSchema.parse(Symbol('id')); // OK

// Null and Undefined
const nullSchema = z.null();
const undefinedSchema = z.undefined();
const voidSchema = z.void(); // Accepts undefined

// Any and Unknown
const anySchema = z.any(); // Allows anything
const unknownSchema = z.unknown(); // Allows anything but safer

// Never
const neverSchema = z.never(); // Never valid (useful in discriminated unions)
```

### 2. String Validations

```typescript
// String with constraints
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();

// Length constraints
const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .max(100, 'Password must be less than 100 characters');

// Pattern matching (regex)
const phoneSchema = z.string().regex(
  /^\+?[1-9]\d{1,14}$/,
  'Invalid phone number format'
);

// String transformations
const trimmedSchema = z.string().trim();
const lowerCaseSchema = z.string().toLowerCase();
const upperCaseSchema = z.string().toUpperCase();

// Combined validations
const usernameSchema = z.string()
  .min(3, 'Username must be at least 3 characters')
  .max(20, 'Username must be less than 20 characters')
  .regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores')
  .toLowerCase();

// Starts with / Ends with
const httpsUrlSchema = z.string()
  .url()
  .startsWith('https://', 'Must use HTTPS');

const emailDomainSchema = z.string()
  .email()
  .endsWith('@company.com', 'Must use company email');

// Datetime
const isoDateSchema = z.string().datetime();
const dateWithOffsetSchema = z.string().datetime({ offset: true });

// IP addresses
const ipv4Schema = z.string().ip({ version: 'v4' });
const ipv6Schema = z.string().ip({ version: 'v6' });
const anyIpSchema = z.string().ip();
```

### 3. Number Validations

```typescript
// Number with constraints
const positiveSchema = z.number().positive();
const negativeSchema = z.number().negative();
const nonNegativeSchema = z.number().nonnegative();
const nonPositiveSchema = z.number().nonpositive();

// Range constraints
const ageSchema = z.number()
  .int('Age must be an integer')
  .min(18, 'Must be at least 18 years old')
  .max(120, 'Age seems unrealistic');

// Multiple constraints
const priceSchema = z.number()
  .positive('Price must be positive')
  .multipleOf(0.01, 'Price must have at most 2 decimal places');

// Integer
const integerSchema = z.number().int();

// Finite (not Infinity or NaN)
const finiteSchema = z.number().finite();

// Safe integer
const safeIntSchema = z.number().int().safe();
```

### 4. Object Schemas

```typescript
// Basic object schema
const userSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  age: z.number().optional()
});

// Type inference
type User = z.infer<typeof userSchema>;
// type User = {
//   id: number;
//   name: string;
//   email: string;
//   age?: number | undefined;
// }

// Parse object
const user = userSchema.parse({
  id: 1,
  name: 'John Doe',
  email: 'john@example.com'
});

// Nested objects
const addressSchema = z.object({
  street: z.string(),
  city: z.string(),
  state: z.string(),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/)
});

const personSchema = z.object({
  name: z.string(),
  address: addressSchema,
  phoneNumbers: z.array(z.string())
});

// Object methods
const extendedSchema = userSchema.extend({
  role: z.string()
});

const mergedSchema = userSchema.merge(z.object({
  createdAt: z.date()
}));

const partialSchema = userSchema.partial(); // All fields optional

const requiredSchema = partialSchema.required(); // All fields required

const pickedSchema = userSchema.pick({ 
  name: true, 
  email: true 
});

const omittedSchema = userSchema.omit({ 
  age: true 
});

// Deep partial
const deepPartialSchema = userSchema.deepPartial();

// Pass through unknown keys
const passthroughSchema = userSchema.passthrough();

// Strip unknown keys (default)
const stripSchema = userSchema.strip();

// Strict (error on unknown keys)
const strictSchema = userSchema.strict();

// Catchall for unknown keys
const catchallSchema = userSchema.catchall(z.string());
```

### 5. Array Schemas

```typescript
// Basic array
const numbersSchema = z.array(z.number());
const numbers = numbersSchema.parse([1, 2, 3]); // OK

// Array with constraints
const nonEmptyArraySchema = z.array(z.string())
  .nonempty('Array must have at least one element');

const limitedArraySchema = z.array(z.number())
  .min(1, 'At least one item required')
  .max(10, 'Maximum 10 items allowed')
  .length(5, 'Exactly 5 items required'); // or specific length

// Array of objects
const usersSchema = z.array(
  z.object({
    id: z.number(),
    name: z.string()
  })
);

// Unique items
const uniqueNumbersSchema = z.array(z.number())
  .refine(
    (items) => new Set(items).size === items.length,
    'All items must be unique'
  );
```

### 6. Tuple Schemas

```typescript
// Fixed-length array with specific types
const coordinateSchema = z.tuple([
  z.number(), // x
  z.number()  // y
]);

const coordinate = coordinateSchema.parse([10, 20]); // OK
// coordinate type: [number, number]

// Tuple with optional and rest elements
const httpResponseSchema = z.tuple([
  z.number(),          // status code
  z.string(),          // message
  z.object({}).optional() // optional body
]);

// Rest elements
const varargsSchema = z.tuple([
  z.string(), // first arg required
]).rest(z.number()); // ...rest are numbers
```

### 7. Union and Discriminated Union

```typescript
// Union (OR)
const stringOrNumberSchema = z.union([
  z.string(),
  z.number()
]);

const value = stringOrNumberSchema.parse('hello'); // OK
const value2 = stringOrNumberSchema.parse(42); // OK

// Shorthand
const shorthandUnionSchema = z.string().or(z.number());

// Discriminated union
const successSchema = z.object({
  status: z.literal('success'),
  data: z.object({
    id: z.number(),
    name: z.string()
  })
});

const errorSchema = z.object({
  status: z.literal('error'),
  error: z.object({
    code: z.string(),
    message: z.string()
  })
});

const responseSchema = z.discriminatedUnion('status', [
  successSchema,
  errorSchema
]);

// Type inference
type Response = z.infer<typeof responseSchema>;
// Response = 
//   | { status: 'success'; data: { id: number; name: string } }
//   | { status: 'error'; error: { code: string; message: string } }

// Usage
function handleResponse(response: Response) {
  if (response.status === 'success') {
    console.log(response.data.name); // TypeScript knows data exists
  } else {
    console.log(response.error.message); // TypeScript knows error exists
  }
}
```

### 8. Enums and Literals

```typescript
// Native enum
enum Role {
  Admin = 'admin',
  User = 'user',
  Guest = 'guest'
}

const roleSchema = z.nativeEnum(Role);
const role = roleSchema.parse(Role.Admin); // OK

// Zod enum
const colorSchema = z.enum(['red', 'green', 'blue']);
type Color = z.infer<typeof colorSchema>; // 'red' | 'green' | 'blue'

// Get enum values
const colors = colorSchema.options; // ['red', 'green', 'blue']

// Literal values
const literalSchema = z.literal('hello');
const value = literalSchema.parse('hello'); // OK
// literalSchema.parse('world'); // Error

// Multiple literals (union)
const statusSchema = z.union([
  z.literal('pending'),
  z.literal('approved'),
  z.literal('rejected')
]);
// Or use enum instead
```

### 9. Optional and Nullable

```typescript
// Optional (undefined allowed)
const optionalStringSchema = z.string().optional();
type OptionalString = z.infer<typeof optionalStringSchema>; // string | undefined

// Nullable (null allowed)
const nullableStringSchema = z.string().nullable();
type NullableString = z.infer<typeof nullableStringSchema>; // string | null

// Both optional and nullable
const optionalNullableSchema = z.string().optional().nullable();
type OptionalNullable = z.infer<typeof optionalNullableSchema>; 
// string | null | undefined

// Nullish (null or undefined)
const nullishSchema = z.string().nullish();
type Nullish = z.infer<typeof nullishSchema>; // string | null | undefined
```

### 10. Default Values

```typescript
// Default value if undefined
const countSchema = z.number().default(0);
const count = countSchema.parse(undefined); // 0

// Default with function
const timestampSchema = z.date().default(() => new Date());

// In objects
const configSchema = z.object({
  name: z.string(),
  port: z.number().default(3000),
  debug: z.boolean().default(false),
  features: z.array(z.string()).default([])
});

const config = configSchema.parse({ name: 'MyApp' });
// config = { name: 'MyApp', port: 3000, debug: false, features: [] }
```

## Advanced Features

### 1. Refinements (Custom Validation)

```typescript
// Simple refinement
const evenNumberSchema = z.number().refine(
  (n) => n % 2 === 0,
  { message: 'Number must be even' }
);

// Multiple refinements
const passwordSchema = z.string()
  .min(8)
  .refine(
    (password) => /[A-Z]/.test(password),
    { message: 'Password must contain at least one uppercase letter' }
  )
  .refine(
    (password) => /[a-z]/.test(password),
    { message: 'Password must contain at least one lowercase letter' }
  )
  .refine(
    (password) => /[0-9]/.test(password),
    { message: 'Password must contain at least one number' }
  );

// Refinement with path (for error reporting)
const userSchema = z.object({
  password: z.string(),
  confirmPassword: z.string()
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: 'Passwords must match',
    path: ['confirmPassword'] // Error appears on confirmPassword field
  }
);

// Superrefine (more control)
const ageSchema = z.number().superRefine((val, ctx) => {
  if (val < 18) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Must be 18 or older',
    });
  }
  if (val > 120) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Age seems unrealistic',
    });
  }
});
```

### 2. Transformations

```typescript
// Transform string to number
const stringToNumberSchema = z.string().transform((val) => parseInt(val, 10));

const num = stringToNumberSchema.parse('42'); // 42 (number)

// Transform with validation
const dateStringSchema = z.string()
  .regex(/^\d{4}-\d{2}-\d{2}$/)
  .transform((str) => new Date(str));

// Transform in objects
const userInputSchema = z.object({
  name: z.string().trim().toLowerCase(),
  age: z.string().transform((val) => parseInt(val, 10)),
  tags: z.string().transform((str) => str.split(',').map(s => s.trim()))
});

const input = userInputSchema.parse({
  name: '  John Doe  ',
  age: '25',
  tags: 'javascript, typescript, react'
});
// {
//   name: 'john doe',
//   age: 25,
//   tags: ['javascript', 'typescript', 'react']
// }

// Chaining transform and refine
const processedSchema = z.string()
  .transform((val) => val.trim())
  .refine((val) => val.length > 0, 'Cannot be empty after trimming');

// Preprocess (transform before validation)
const preprocessedSchema = z.preprocess(
  (val) => {
    if (typeof val === 'string') {
      return parseInt(val, 10);
    }
    return val;
  },
  z.number()
);
```

### 3. Async Validation

```typescript
// Async refinement
const usernameSchema = z.string().refine(
  async (username) => {
    const response = await fetch(`/api/check-username/${username}`);
    const data = await response.json();
    return data.available;
  },
  { message: 'Username is already taken' }
);

// Use parseAsync or safeParseAsync
const result = await usernameSchema.safeParseAsync('john_doe');

// Async transform
const fetchUserSchema = z.number().transform(async (id) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
});

const user = await fetchUserSchema.parseAsync(1);

// Complex async validation
const emailSchema = z.string()
  .email()
  .refine(
    async (email) => {
      const exists = await checkEmailExists(email);
      return !exists;
    },
    { message: 'Email is already registered' }
  );

async function checkEmailExists(email: string): Promise<boolean> {
  // API call to check email
  return false;
}
```

### 4. Error Handling

```typescript
import { z, ZodError } from 'zod';

// Basic error handling
const schema = z.object({
  name: z.string(),
  age: z.number()
});

try {
  schema.parse({ name: 'John', age: 'invalid' });
} catch (error) {
  if (error instanceof ZodError) {
    console.log(error.issues);
    // [
    //   {
    //     code: 'invalid_type',
    //     expected: 'number',
    //     received: 'string',
    //     path: ['age'],
    //     message: 'Expected number, received string'
    //   }
    // ]
  }
}

// Safe parse (no exception)
const result = schema.safeParse({ name: 'John', age: 'invalid' });

if (!result.success) {
  console.log(result.error.issues);
  console.log(result.error.format());
  // {
  //   _errors: [],
  //   age: {
  //     _errors: ['Expected number, received string']
  //   }
  // }
}

// Custom error messages
const customSchema = z.object({
  email: z.string({ 
    required_error: 'Email is required',
    invalid_type_error: 'Email must be a string'
  }).email('Invalid email format'),
  age: z.number({
    required_error: 'Age is required',
    invalid_type_error: 'Age must be a number'
  }).min(18, 'Must be at least 18 years old')
});

// Error map (customize all errors)
const customErrorMap: z.ZodErrorMap = (issue, ctx) => {
  if (issue.code === z.ZodIssueCode.invalid_type) {
    if (issue.expected === 'string') {
      return { message: 'This field must be text' };
    }
  }
  return { message: ctx.defaultError };
};

z.setErrorMap(customErrorMap);

// Flatten errors
const flatErrors = result.error?.flatten();
// {
//   formErrors: [],
//   fieldErrors: {
//     age: ['Expected number, received string']
//   }
// }
```

### 5. Coercion

```typescript
// Coerce strings to numbers
const numberCoercionSchema = z.coerce.number();
const num = numberCoercionSchema.parse('42'); // 42 (number)

// Coerce to boolean
const booleanCoercionSchema = z.coerce.boolean();
const bool = booleanCoercionSchema.parse('true'); // true (boolean)

// Coerce to date
const dateCoercionSchema = z.coerce.date();
const date = dateCoercionSchema.parse('2024-01-01'); // Date object

// Form data parsing with coercion
const formSchema = z.object({
  name: z.string(),
  age: z.coerce.number(),
  subscribe: z.coerce.boolean(),
  startDate: z.coerce.date()
});

// Parse FormData directly
const formData = new FormData();
formData.append('name', 'John');
formData.append('age', '25');
formData.append('subscribe', 'true');
formData.append('startDate', '2024-01-01');

const parsed = formSchema.parse(Object.fromEntries(formData));
// {
//   name: 'John',
//   age: 25,
//   subscribe: true,
//   startDate: Date(2024-01-01)
// }
```

## Integration Examples

### 1. React Hook Form Integration

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Define schema
const registrationSchema = z.object({
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username must be less than 20 characters'),
  email: z.string().email('Invalid email address'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number'),
  confirmPassword: z.string(),
  age: z.number().min(18, 'Must be at least 18 years old'),
  acceptTerms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms' })
  })
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword']
});

type RegistrationForm = z.infer<typeof registrationSchema>;

function RegistrationComponent() {
  const {
    register,
    handleSubmit,
    formState: { errors }
  } = useForm<RegistrationForm>({
    resolver: zodResolver(registrationSchema)
  });

  const onSubmit = (data: RegistrationForm) => {
    console.log('Form data:', data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('username')} placeholder="Username" />
        {errors.username && <span>{errors.username.message}</span>}
      </div>

      <div>
        <input {...register('email')} type="email" placeholder="Email" />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <div>
        <input {...register('password')} type="password" placeholder="Password" />
        {errors.password && <span>{errors.password.message}</span>}
      </div>

      <div>
        <input {...register('confirmPassword')} type="password" placeholder="Confirm Password" />
        {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}
      </div>

      <div>
        <input {...register('age', { valueAsNumber: true })} type="number" placeholder="Age" />
        {errors.age && <span>{errors.age.message}</span>}
      </div>

      <div>
        <label>
          <input {...register('acceptTerms')} type="checkbox" />
          I accept the terms and conditions
        </label>
        {errors.acceptTerms && <span>{errors.acceptTerms.message}</span>}
      </div>

      <button type="submit">Register</button>
    </form>
  );
}
```

### 2. API Request/Response Validation

```typescript
// Define API schemas
const createUserRequestSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'guest'])
});

const userResponseSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.string().datetime()
});

const apiErrorSchema = z.object({
  error: z.string(),
  message: z.string(),
  statusCode: z.number()
});

// Type inference
type CreateUserRequest = z.infer<typeof createUserRequestSchema>;
type UserResponse = z.infer<typeof userResponseSchema>;
type ApiError = z.infer<typeof apiErrorSchema>;

// API function with validation
async function createUser(data: unknown): Promise<UserResponse> {
  // Validate request data
  const validatedData = createUserRequestSchema.parse(data);

  // Make API call
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(validatedData)
  });

  const responseData = await response.json();

  // Validate response
  if (!response.ok) {
    const error = apiErrorSchema.parse(responseData);
    throw new Error(error.message);
  }

  return userResponseSchema.parse(responseData);
}

// Usage
try {
  const user = await createUser({
    name: 'John Doe',
    email: 'john@example.com',
    role: 'user'
  });
  console.log('User created:', user);
} catch (error) {
  if (error instanceof z.ZodError) {
    console.error('Validation error:', error.issues);
  } else {
    console.error('API error:', error);
  }
}
```

### 3. Environment Variables Validation

```typescript
// env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  API_KEY: z.string().min(1),
  REDIS_HOST: z.string().default('localhost'),
  REDIS_PORT: z.coerce.number().default(6379),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(['error', 'warn', 'info', 'debug']).default('info'),
  ENABLE_FEATURE_X: z.coerce.boolean().default(false)
});

// Validate environment variables
export const env = envSchema.parse(process.env);

// TypeScript knows the exact types
export type Env = z.infer<typeof envSchema>;

// Usage in application
import { env } from './env';

console.log(env.PORT); // number
console.log(env.NODE_ENV); // 'development' | 'production' | 'test'
console.log(env.ENABLE_FEATURE_X); // boolean
```

### 4. Configuration Files

```typescript
// config.schema.ts
import { z } from 'zod';

const databaseConfigSchema = z.object({
  host: z.string(),
  port: z.number().min(1).max(65535),
  database: z.string(),
  username: z.string(),
  password: z.string(),
  ssl: z.boolean().default(false),
  poolSize: z.number().min(1).max(100).default(10)
});

const serverConfigSchema = z.object({
  port: z.number().min(1).max(65535).default(3000),
  host: z.string().default('localhost'),
  cors: z.object({
    enabled: z.boolean().default(true),
    origins: z.array(z.string()).default(['*'])
  })
});

const loggingConfigSchema = z.object({
  level: z.enum(['error', 'warn', 'info', 'debug']).default('info'),
  format: z.enum(['json', 'text']).default('json'),
  destination: z.string().default('stdout')
});

export const appConfigSchema = z.object({
  database: databaseConfigSchema,
  server: serverConfigSchema,
  logging: loggingConfigSchema,
  features: z.record(z.boolean()).default({})
});

export type AppConfig = z.infer<typeof appConfigSchema>;

// Load and validate config
import fs from 'fs';
import { appConfigSchema, AppConfig } from './config.schema';

export function loadConfig(path: string): AppConfig {
  const rawConfig = JSON.parse(fs.readFileSync(path, 'utf-8'));
  return appConfigSchema.parse(rawConfig);
}

// Usage
const config = loadConfig('./config.json');
console.log(config.database.host); // TypeScript knows this is a string
console.log(config.server.port); // TypeScript knows this is a number
```

### 5. GraphQL Resolver Validation

```typescript
import { z } from 'zod';

// Input validation schemas
const createPostInputSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  tags: z.array(z.string()).max(5),
  published: z.boolean().default(false)
});

const updatePostInputSchema = createPostInputSchema.partial().extend({
  id: z.string().uuid()
});

type CreatePostInput = z.infer<typeof createPostInputSchema>;
type UpdatePostInput = z.infer<typeof updatePostInputSchema>;

// Resolvers
const resolvers = {
  Mutation: {
    createPost: async (_: any, args: { input: unknown }) => {
      // Validate input
      const input = createPostInputSchema.parse(args.input);
      
      // Create post
      const post = await db.post.create({
        data: input
      });
      
      return post;
    },
    
    updatePost: async (_: any, args: { input: unknown }) => {
      // Validate input
      const input = updatePostInputSchema.parse(args.input);
      
      // Update post
      const post = await db.post.update({
        where: { id: input.id },
        data: input
      });
      
      return post;
    }
  }
};
```

## Common Mistakes

### 1. Not Using safeParse in User Input

**Wrong:**
```typescript
function handleUserInput(data: unknown) {
  const user = userSchema.parse(data); // Can throw
  // Process user
}
```

**Correct:**
```typescript
function handleUserInput(data: unknown) {
  const result = userSchema.safeParse(data);
  if (!result.success) {
    console.error('Validation failed:', result.error);
    return;
  }
  const user = result.data;
  // Process user
}
```

### 2. Not Inferring Types

**Wrong:**
```typescript
const userSchema = z.object({
  name: z.string(),
  age: z.number()
});

// Manually defining type (error-prone)
interface User {
  name: string;
  age: number;
}
```

**Correct:**
```typescript
const userSchema = z.object({
  name: z.string(),
  age: z.number()
});

// Infer type from schema (single source of truth)
type User = z.infer<typeof userSchema>;
```

### 3. Overusing any or unknown

**Wrong:**
```typescript
const schema = z.any(); // Loses type safety
```

**Correct:**
```typescript
const schema = z.object({
  // Define specific shape
  name: z.string(),
  data: z.record(z.unknown()) // Use unknown for truly dynamic data
});
```

### 4. Not Handling Async Validation Properly

**Wrong:**
```typescript
const result = schema.safeParse(data); // Won't work with async refinements
```

**Correct:**
```typescript
const result = await schema.safeParseAsync(data); // Use async version
```

## Best Practices

### 1. Create Reusable Schemas

```typescript
// common.schemas.ts
export const emailSchema = z.string().email().toLowerCase();
export const passwordSchema = z.string().min(8).max(100);
export const phoneSchema = z.string().regex(/^\+?[1-9]\d{1,14}$/);
export const urlSchema = z.string().url();

// Use in other schemas
const userSchema = z.object({
  email: emailSchema,
  password: passwordSchema,
  phone: phoneSchema.optional()
});
```

### 2. Use Discriminated Unions for Variants

```typescript
const shapeSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('circle'),
    radius: z.number()
  }),
  z.object({
    type: z.literal('rectangle'),
    width: z.number(),
    height: z.number()
  }),
  z.object({
    type: z.literal('triangle'),
    base: z.number(),
    height: z.number()
  })
]);
```

### 3. Separate Concerns

```typescript
// schemas/user.schema.ts - Domain schemas
export const userSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: emailSchema
});

// schemas/api/user.schema.ts - API-specific schemas
export const createUserRequestSchema = userSchema.omit({ id: true });
export const updateUserRequestSchema = userSchema.partial();
export const userResponseSchema = userSchema;
```

### 4. Document Schemas

```typescript
const userSchema = z.object({
  name: z.string()
    .min(1, 'Name is required')
    .max(100, 'Name is too long')
    .describe('The user\'s full name'),
  
  email: z.string()
    .email('Invalid email format')
    .describe('User\'s email address for login and notifications'),
  
  role: z.enum(['admin', 'user', 'guest'])
    .describe('User\'s role determining access permissions')
});
```

### 5. Use Brand Types for Validation

```typescript
// Create branded types for validated data
const UserIdSchema = z.string().uuid().brand('UserId');
type UserId = z.infer<typeof UserIdSchema>;

const EmailSchema = z.string().email().brand('Email');
type Email = z.infer<typeof EmailSchema>;

// Now these are distinct types
function sendEmail(to: Email, subject: string) {
  // Implementation
}

const email = EmailSchema.parse('user@example.com');
sendEmail(email, 'Hello'); // OK

const rawString = 'user@example.com';
// sendEmail(rawString, 'Hello'); // Type error - need Email brand
```

## When to Use Zod

### Use When:

- Building TypeScript applications requiring runtime validation
- Need type inference from schemas (DRY principle)
- Validating user input, API requests/responses
- Parsing environment variables or configuration files
- Building forms with runtime validation
- Need composable, reusable validation schemas
- Want excellent TypeScript integration

### Avoid When:

- Working with JavaScript without TypeScript (consider Joi or Yup)
- Need extremely complex validation logic (might need custom solution)
- Bundle size is critical and validation is simple (use simple checks)
- Already heavily invested in another validation library

## Interview Questions

### Q1: What makes Zod different from other validation libraries like Yup or Joi?

**Answer:**

**Zod Advantages:**
1. **TypeScript-First**: Designed from ground up for TypeScript with perfect type inference
2. **Zero Dependencies**: No runtime dependencies
3. **Type Inference**: Automatically infers TypeScript types from schemas
4. **Immutable**: Schema methods return new schemas (functional)
5. **Smaller Bundle**: More tree-shakeable

**Example:**
```typescript
// Zod - Type automatically inferred
const userSchema = z.object({
  name: z.string(),
  age: z.number()
});
type User = z.infer<typeof userSchema>; // Automatic

// Yup - Need separate type definition
const userSchema = yup.object({
  name: yup.string().required(),
  age: yup.number().required()
});
interface User { // Manual
  name: string;
  age: number;
}
```

### Q2: Explain the difference between parse() and safeParse() in Zod.

**Answer:**

**parse():**
- Throws `ZodError` if validation fails
- Returns parsed data if successful
- Use when you want exceptions for invalid data

```typescript
try {
  const user = userSchema.parse(data);
  // user is validated
} catch (error) {
  // Handle ZodError
}
```

**safeParse():**
- Never throws errors
- Returns result object with success boolean
- Use for user input or untrusted data

```typescript
const result = userSchema.safeParse(data);
if (result.success) {
  const user = result.data; // Validated data
} else {
  console.log(result.error); // ZodError
}
```

**Best Practice:** Use `safeParse()` for user input, `parse()` for internal data you control.

### Q3: How do you handle cross-field validation in Zod?

**Answer:** Use `.refine()` method on the entire object schema:

```typescript
const schema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string()
}).refine(
  (data) => data.password === data.confirmPassword,
  {
    message: 'Passwords must match',
    path: ['confirmPassword'] // Show error on this field
  }
);

// More complex example
const bookingSchema = z.object({
  checkIn: z.date(),
  checkOut: z.date(),
  guests: z.number().min(1)
}).refine(
  (data) => data.checkOut > data.checkIn,
  {
    message: 'Check-out must be after check-in',
    path: ['checkOut']
  }
).refine(
  (data) => {
    const nights = (data.checkOut.getTime() - data.checkIn.getTime()) / (1000 * 60 * 60 * 24);
    return nights <= 30;
  },
  {
    message: 'Maximum stay is 30 nights',
    path: ['checkOut']
  }
);
```

### Q4: How do you implement async validation in Zod?

**Answer:** Use `.refine()` with async function and `parseAsync()` or `safeParseAsync()`:

```typescript
const emailSchema = z.string()
  .email()
  .refine(
    async (email) => {
      const response = await fetch(`/api/check-email/${email}`);
      const data = await response.json();
      return data.available;
    },
    { message: 'Email is already registered' }
  );

// Use async parse methods
const result = await emailSchema.safeParseAsync('user@example.com');

if (result.success) {
  console.log('Email is available:', result.data);
} else {
  console.log('Validation failed:', result.error);
}

// In practice
async function validateRegistration(data: unknown) {
  const registrationSchema = z.object({
    username: z.string().refine(
      async (username) => {
        return await isUsernameAvailable(username);
      },
      { message: 'Username taken' }
    ),
    email: z.string().email().refine(
      async (email) => {
        return await isEmailAvailable(email);
      },
      { message: 'Email already registered' }
    )
  });

  return await registrationSchema.safeParseAsync(data);
}
```

### Q5: How do you handle form data with Zod including transformations?

**Answer:** Use coercion and transformations:

```typescript
import { z } from 'zod';

// Form schema with coercion
const formSchema = z.object({
  // Text inputs
  name: z.string()
    .trim()
    .min(1, 'Name is required'),
  
  // Number inputs (FormData values are strings)
  age: z.coerce.number()
    .int()
    .min(18, 'Must be 18 or older'),
  
  // Checkbox (coerce to boolean)
  subscribe: z.coerce.boolean(),
  
  // Date inputs
  startDate: z.coerce.date(),
  
  // Multi-select or comma-separated tags
  tags: z.string()
    .transform((str) => str.split(',').map(s => s.trim()))
    .pipe(z.array(z.string().min(1))),
  
  // Email with normalization
  email: z.string()
    .email()
    .toLowerCase()
    .trim(),
  
  // Phone number formatting
  phone: z.string()
    .regex(/^\d{10}$/, 'Invalid phone number')
    .transform((val) => val.replace(/(\d{3})(\d{3})(\d{4})/, '($1) $2-$3'))
});

// Parse FormData
function handleFormSubmit(formData: FormData) {
  const rawData = Object.fromEntries(formData);
  const result = formSchema.safeParse(rawData);
  
  if (!result.success) {
    // Handle validation errors
    const errors = result.error.flatten();
    console.log(errors.fieldErrors);
    return;
  }
  
  // result.data is fully typed and transformed
  console.log(result.data);
  // {
  //   name: 'john doe',
  //   age: 25,
  //   subscribe: true,
  //   startDate: Date,
  //   tags: ['typescript', 'react'],
  //   email: 'john@example.com',
  //   phone: '(555) 123-4567'
  // }
}
```

## Key Takeaways

1. **TypeScript-First**: Designed specifically for TypeScript with excellent type inference
2. **Type Inference**: Use `z.infer<typeof schema>` to automatically derive types
3. **Composable**: Build complex schemas from simpler ones
4. **Immutable**: Schema methods return new schemas
5. **Parse vs SafeParse**: Use `safeParse` for untrusted input
6. **Refinements**: Custom validation logic with `.refine()` and `.superRefine()`
7. **Transformations**: Transform data during parsing with `.transform()`
8. **Async Support**: Async validation with `parseAsync` and `safeParseAsync`
9. **Error Handling**: Rich error information with paths and messages
10. **Coercion**: Convert string inputs to proper types with `z.coerce`
11. **Discriminated Unions**: Type-safe variant handling
12. **Default Values**: Use `.default()` for optional fields
13. **Integration**: Works seamlessly with React Hook Form, Next.js, etc.
14. **Zero Dependencies**: No runtime dependencies
15. **Bundle Size**: Tree-shakeable for smaller bundles

## Resources

- **Official Documentation**: https://zod.dev/
- **GitHub Repository**: https://github.com/colinhacks/zod
- **React Hook Form Integration**: https://github.com/react-hook-form/resolvers
- **TypeScript Playground**: https://www.typescriptlang.org/play
- **Zod to JSON Schema**: https://github.com/StefanTerdell/zod-to-json-schema
- **Zod to TypeScript**: https://github.com/sachinraja/zod-to-ts
- **Community Examples**: https://github.com/colinhacks/zod/discussions
- **Video Tutorials**: Search "Zod TypeScript" on YouTube
- **Comparison Articles**: "Zod vs Yup vs Joi"
- **Next.js Integration**: https://nextjs.org/docs/basic-features/data-fetching