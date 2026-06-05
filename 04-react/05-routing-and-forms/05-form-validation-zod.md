# Form Validation with Zod

## The Idea

**In plain English:** Form validation with Zod means writing a set of rules that automatically checks whether the information a user typed into a form (like an email address or password) is correct before sending it anywhere. Zod is a library — a ready-made toolkit — that lets you describe exactly what valid data looks like, then checks incoming data against that description instantly.

**Real-world analogy:** Imagine a bouncer at a club entrance who has a checklist on a clipboard. Before letting anyone in, they verify: Is this person carrying a valid ID? Is the ID format correct? Are they old enough? If anything fails the checklist, the bouncer immediately explains why entry is denied.

- The checklist on the clipboard = the Zod schema (the set of rules you define)
- A person trying to enter = the form data the user submitted
- The bouncer checking each item = Zod running validation against the schema
- The bouncer's explanation of what went wrong = the error messages Zod returns

---

## Table of Contents

1. [Introduction](#introduction)
2. [Zod Basics](#zod-basics)
3. [Integration with React Hook Form](#integration-with-react-hook-form)
4. [Schema Composition](#schema-composition)
5. [Custom Validation](#custom-validation)
6. [Error Handling](#error-handling)
7. [Advanced Patterns](#advanced-patterns)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)
11. [Resources](#resources)

## Introduction

Zod is a TypeScript-first schema validation library that provides type safety, runtime validation, and excellent developer experience. When combined with React Hook Form, it creates a powerful, type-safe form validation solution.

**Why Zod?**
- TypeScript-first with automatic type inference
- Zero dependencies
- Works in Node.js and browsers
- Composable schemas
- Excellent error messages
- Small bundle size

## Zod Basics

### Installation

```bash
npm install zod @hookform/resolvers
```

### Basic Schema Types

```typescript
import { z } from 'zod';

// Primitives
const stringSchema = z.string();
const numberSchema = z.number();
const booleanSchema = z.boolean();
const dateSchema = z.date();

// Literals
const literalSchema = z.literal('hello');
const literalNumberSchema = z.literal(42);

// Strings with constraints
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();
const minMaxSchema = z.string().min(3).max(20);
const regexSchema = z.string().regex(/^[A-Z0-9]+$/);

// Numbers with constraints
const positiveSchema = z.number().positive();
const integerSchema = z.number().int();
const rangeSchema = z.number().min(0).max(100);
const multipleSchema = z.number().multipleOf(5);

// Dates
const futureDate = z.date().min(new Date());
const pastDate = z.date().max(new Date());

// Arrays
const stringArray = z.array(z.string());
const nonEmptyArray = z.array(z.string()).nonempty();
const minMaxArray = z.array(z.string()).min(2).max(10);

// Objects
const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int().positive()
});

// Optionals and nullables
const optionalString = z.string().optional();
const nullableString = z.string().nullable();
const nullishString = z.string().nullish(); // optional + nullable

// Unions
const stringOrNumber = z.union([z.string(), z.number()]);
const statusUnion = z.enum(['active', 'inactive', 'pending']);

// Type inference
type User = z.infer<typeof userSchema>;
// {
//   name: string;
//   email: string;
//   age: number;
// }
```

### Object Schemas

```typescript
// Basic object
const addressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zipCode: z.string().regex(/^\d{5}$/),
  country: z.string().default('USA')
});

// Nested objects
const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  address: addressSchema,
  preferences: z.object({
    newsletter: z.boolean(),
    notifications: z.boolean()
  })
});

// Partial, Required, Pick, Omit
const partialUser = userSchema.partial(); // All fields optional
const requiredUser = userSchema.required(); // All fields required
const userWithoutEmail = userSchema.omit({ email: true });
const onlyNameAndEmail = userSchema.pick({ name: true, email: true });

// Extend
const adminSchema = userSchema.extend({
  role: z.literal('admin'),
  permissions: z.array(z.string())
});

// Merge
const timestampSchema = z.object({
  createdAt: z.date(),
  updatedAt: z.date()
});

const userWithTimestamps = userSchema.merge(timestampSchema);
```

## Integration with React Hook Form

### Basic Integration

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Define schema
const loginSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Password must contain uppercase, lowercase, and number'
    )
});

type LoginFormData = z.infer<typeof loginSchema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors }
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema)
  });

  const onSubmit = (data: LoginFormData) => {
    console.log(data); // Fully typed and validated
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('email')} placeholder="Email" />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <div>
        <input
          {...register('password')}
          type="password"
          placeholder="Password"
        />
        {errors.password && <span>{errors.password.message}</span>}
      </div>

      <button type="submit">Login</button>
    </form>
  );
}
```

### Complex Form Example

```typescript
const signupSchema = z
  .object({
    username: z
      .string()
      .min(3, 'Username must be at least 3 characters')
      .max(20, 'Username must be at most 20 characters')
      .regex(
        /^[a-zA-Z0-9_]+$/,
        'Username can only contain letters, numbers, and underscores'
      ),
    email: z.string().email('Invalid email address'),
    password: z
      .string()
      .min(8, 'Password must be at least 8 characters')
      .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
      .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
      .regex(/[0-9]/, 'Password must contain at least one number')
      .regex(
        /[^A-Za-z0-9]/,
        'Password must contain at least one special character'
      ),
    confirmPassword: z.string(),
    age: z
      .number()
      .int('Age must be a whole number')
      .min(18, 'You must be at least 18 years old')
      .max(120, 'Invalid age'),
    terms: z.literal(true, {
      errorMap: () => ({ message: 'You must accept the terms' })
    })
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword']
  });

type SignupFormData = z.infer<typeof signupSchema>;

function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors }
  } = useForm<SignupFormData>({
    resolver: zodResolver(signupSchema)
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('username')} />
        {errors.username && <span>{errors.username.message}</span>}
      </div>

      <div>
        <input {...register('email')} />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <div>
        <input {...register('password')} type="password" />
        {errors.password && <span>{errors.password.message}</span>}
      </div>

      <div>
        <input {...register('confirmPassword')} type="password" />
        {errors.confirmPassword && (
          <span>{errors.confirmPassword.message}</span>
        )}
      </div>

      <div>
        <input
          {...register('age', { valueAsNumber: true })}
          type="number"
        />
        {errors.age && <span>{errors.age.message}</span>}
      </div>

      <div>
        <input {...register('terms')} type="checkbox" />
        <label>I accept the terms and conditions</label>
        {errors.terms && <span>{errors.terms.message}</span>}
      </div>

      <button type="submit">Sign Up</button>
    </form>
  );
}
```

### Default Values with Zod

```typescript
const profileSchema = z.object({
  name: z.string().default(''),
  bio: z.string().default(''),
  website: z.string().url().optional().or(z.literal('')),
  newsletter: z.boolean().default(false)
});

function ProfileForm() {
  const {
    register,
    handleSubmit
  } = useForm({
    resolver: zodResolver(profileSchema),
    defaultValues: profileSchema.parse({}) // Uses schema defaults
  });

  return <form>{/* form fields */}</form>;
}
```

## Schema Composition

### Reusable Schema Building Blocks

```typescript
// Common validation schemas
const commonSchemas = {
  email: z.string().email('Invalid email'),
  
  password: z
    .string()
    .min(8, 'Min 8 characters')
    .regex(/[A-Z]/, 'Need uppercase')
    .regex(/[a-z]/, 'Need lowercase')
    .regex(/[0-9]/, 'Need number'),
  
  phone: z
    .string()
    .regex(/^\+?[1-9]\d{1,14}$/, 'Invalid phone number'),
  
  url: z.string().url('Invalid URL'),
  
  positiveInteger: z.number().int().positive(),
  
  dateInFuture: z.date().min(new Date(), 'Date must be in the future'),
  
  nonEmptyString: z.string().trim().min(1, 'Required'),
  
  slug: z
    .string()
    .regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, 'Invalid slug format')
};

// Compose into forms
const userSettingsSchema = z.object({
  email: commonSchemas.email,
  phone: commonSchemas.phone.optional(),
  website: commonSchemas.url.optional()
});

const createPostSchema = z.object({
  title: commonSchemas.nonEmptyString,
  slug: commonSchemas.slug,
  publishDate: commonSchemas.dateInFuture
});
```

### Schema Factories

```typescript
// Factory for creating entity schemas
function createEntitySchema<T extends z.ZodRawShape>(fields: T) {
  return z.object({
    id: z.string().uuid(),
    createdAt: z.date(),
    updatedAt: z.date(),
    ...fields
  });
}

// Usage
const userSchema = createEntitySchema({
  name: z.string(),
  email: z.string().email()
});

const postSchema = createEntitySchema({
  title: z.string(),
  content: z.string(),
  authorId: z.string().uuid()
});

// Pagination schema factory
function createPaginatedSchema<T extends z.ZodTypeAny>(itemSchema: T) {
  return z.object({
    items: z.array(itemSchema),
    total: z.number(),
    page: z.number(),
    pageSize: z.number()
  });
}

const paginatedUsers = createPaginatedSchema(userSchema);
```

### Discriminated Unions

```typescript
// Form that changes based on account type
const personalAccountSchema = z.object({
  type: z.literal('personal'),
  firstName: z.string(),
  lastName: z.string()
});

const businessAccountSchema = z.object({
  type: z.literal('business'),
  companyName: z.string(),
  taxId: z.string()
});

const accountSchema = z.discriminatedUnion('type', [
  personalAccountSchema,
  businessAccountSchema
]);

type Account = z.infer<typeof accountSchema>;

function AccountForm() {
  const { register, watch } = useForm<Account>({
    resolver: zodResolver(accountSchema)
  });

  const accountType = watch('type');

  return (
    <form>
      <select {...register('type')}>
        <option value="personal">Personal</option>
        <option value="business">Business</option>
      </select>

      {accountType === 'personal' && (
        <>
          <input {...register('firstName')} placeholder="First Name" />
          <input {...register('lastName')} placeholder="Last Name" />
        </>
      )}

      {accountType === 'business' && (
        <>
          <input {...register('companyName')} placeholder="Company Name" />
          <input {...register('taxId')} placeholder="Tax ID" />
        </>
      )}
    </form>
  );
}
```

## Custom Validation

### Refine Method

```typescript
// Simple refine
const passwordMatchSchema = z
  .object({
    password: z.string(),
    confirmPassword: z.string()
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords must match',
    path: ['confirmPassword']
  });

// Multiple refine checks
const eventSchema = z
  .object({
    startDate: z.date(),
    endDate: z.date(),
    capacity: z.number().positive(),
    registered: z.number()
  })
  .refine((data) => data.endDate > data.startDate, {
    message: 'End date must be after start date',
    path: ['endDate']
  })
  .refine((data) => data.registered <= data.capacity, {
    message: 'Registered attendees exceed capacity',
    path: ['registered']
  });
```

### Async Validation

```typescript
// Async schema for checking username availability
const asyncSignupSchema = z.object({
  username: z
    .string()
    .min(3)
    .refine(
      async (username) => {
        const response = await fetch(`/api/check-username/${username}`);
        const { available } = await response.json();
        return available;
      },
      {
        message: 'Username is already taken'
      }
    ),
  email: z
    .string()
    .email()
    .refine(
      async (email) => {
        const response = await fetch(`/api/check-email/${email}`);
        const { available } = await response.json();
        return available;
      },
      {
        message: 'Email is already registered'
      }
    )
});

// Use with React Hook Form
function AsyncValidationForm() {
  const { register, handleSubmit } = useForm({
    resolver: zodResolver(asyncSignupSchema),
    mode: 'onBlur' // Validate on blur to avoid too many API calls
  });

  return <form>{/* fields */}</form>;
}
```

### Custom Transform

```typescript
// Transform input before validation
const trimmedStringSchema = z.string().transform((val) => val.trim());

const normalizedEmailSchema = z
  .string()
  .email()
  .transform((val) => val.toLowerCase());

const phoneSchema = z
  .string()
  .transform((val) => val.replace(/\D/g, '')) // Remove non-digits
  .pipe(z.string().length(10, 'Phone must be 10 digits'));

// Preprocess before validation
const dateFromStringSchema = z.preprocess(
  (val) => (typeof val === 'string' ? new Date(val) : val),
  z.date()
);

const numberFromStringSchema = z.preprocess(
  (val) => (typeof val === 'string' ? Number(val) : val),
  z.number()
);
```

### SuperRefine for Complex Validation

```typescript
const complexFormSchema = z
  .object({
    role: z.enum(['user', 'admin', 'moderator']),
    permissions: z.array(z.string()),
    email: z.string().email()
  })
  .superRefine((data, ctx) => {
    // Custom validation logic with full control
    
    if (data.role === 'admin' && data.permissions.length === 0) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Admin must have at least one permission',
        path: ['permissions']
      });
    }

    if (data.role === 'user' && data.permissions.includes('delete_users')) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Regular users cannot have delete_users permission',
        path: ['permissions']
      });
    }

    if (data.email.endsWith('@competitor.com')) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Competitor emails not allowed',
        path: ['email']
      });
    }
  });
```

## Error Handling

### Custom Error Messages

```typescript
// Global error map
import { z } from 'zod';

const customErrorMap: z.ZodErrorMap = (issue, ctx) => {
  if (issue.code === z.ZodIssueCode.invalid_type) {
    if (issue.expected === 'string') {
      return { message: 'This field is required' };
    }
  }
  
  if (issue.code === z.ZodIssueCode.too_small) {
    if (issue.type === 'string') {
      return { 
        message: `Must be at least ${issue.minimum} characters` 
      };
    }
  }
  
  return { message: ctx.defaultError };
};

z.setErrorMap(customErrorMap);

// Per-schema error messages
const userSchema = z.object({
  name: z.string({
    required_error: 'Name is required',
    invalid_type_error: 'Name must be a string'
  }),
  age: z.number({
    required_error: 'Age is required',
    invalid_type_error: 'Age must be a number'
  })
});
```

### Error Formatting

```typescript
function DisplayErrors() {
  const {
    formState: { errors }
  } = useForm({
    resolver: zodResolver(schema)
  });

  // Flatten nested errors
  const flatErrors = Object.entries(errors).map(([field, error]) => ({
    field,
    message: error?.message
  }));

  return (
    <div className="errors">
      {flatErrors.map(({ field, message }) => (
        <div key={field} className="error">
          <strong>{field}:</strong> {message}
        </div>
      ))}
    </div>
  );
}

// Error boundary component
function FormErrorBoundary({ errors }: { errors: FieldErrors }) {
  const errorMessages = Object.values(errors)
    .map((error) => error?.message)
    .filter(Boolean);

  if (errorMessages.length === 0) return null;

  return (
    <div className="error-banner">
      <h4>Please fix the following errors:</h4>
      <ul>
        {errorMessages.map((message, index) => (
          <li key={index}>{message}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Advanced Patterns

### Dynamic Schema Based on Conditions

```typescript
function createFormSchema(userRole: 'user' | 'admin') {
  const baseSchema = z.object({
    name: z.string(),
    email: z.string().email()
  });

  if (userRole === 'admin') {
    return baseSchema.extend({
      permissions: z.array(z.string()).min(1),
      department: z.string()
    });
  }

  return baseSchema;
}

function DynamicForm({ userRole }: { userRole: 'user' | 'admin' }) {
  const schema = useMemo(() => createFormSchema(userRole), [userRole]);

  const { register } = useForm({
    resolver: zodResolver(schema)
  });

  return <form>{/* fields */}</form>;
}
```

### Array Validation

```typescript
const todoListSchema = z.object({
  todos: z
    .array(
      z.object({
        text: z.string().min(1, 'Todo cannot be empty'),
        completed: z.boolean(),
        priority: z.enum(['low', 'medium', 'high'])
      })
    )
    .min(1, 'Add at least one todo')
    .max(50, 'Maximum 50 todos allowed')
});

function TodoListForm() {
  const { register, control } = useForm({
    resolver: zodResolver(todoListSchema)
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'todos'
  });

  return (
    <form>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`todos.${index}.text`)} />
          <select {...register(`todos.${index}.priority`)}>
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </select>
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}
      
      <button
        type="button"
        onClick={() =>
          append({ text: '', completed: false, priority: 'medium' })
        }
      >
        Add Todo
      </button>
    </form>
  );
}
```

### File Upload Validation

```typescript
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ACCEPTED_IMAGE_TYPES = [
  'image/jpeg',
  'image/jpg',
  'image/png',
  'image/webp'
];

const fileSchema = z.object({
  avatar: z
    .custom<FileList>()
    .refine((files) => files?.length === 1, 'Image is required')
    .refine(
      (files) => files?.[0]?.size <= MAX_FILE_SIZE,
      'Max file size is 5MB'
    )
    .refine(
      (files) => ACCEPTED_IMAGE_TYPES.includes(files?.[0]?.type),
      'Only .jpg, .jpeg, .png and .webp formats are supported'
    )
    .transform((files) => files[0])
});

function FileUploadForm() {
  const {
    register,
    handleSubmit,
    formState: { errors }
  } = useForm({
    resolver: zodResolver(fileSchema)
  });

  const onSubmit = async (data: { avatar: File }) => {
    const formData = new FormData();
    formData.append('avatar', data.avatar);
    await uploadFile(formData);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('avatar')} type="file" accept="image/*" />
      {errors.avatar && <span>{errors.avatar.message}</span>}
      <button type="submit">Upload</button>
    </form>
  );
}
```

### Multi-Step Form Validation

```typescript
// Step schemas
const step1Schema = z.object({
  name: z.string().min(1),
  email: z.string().email()
});

const step2Schema = z.object({
  address: z.string().min(1),
  city: z.string().min(1),
  zipCode: z.string().regex(/^\d{5}$/)
});

const step3Schema = z.object({
  cardNumber: z.string().length(16),
  cvv: z.string().length(3)
});

// Combined schema for final validation
const completeFormSchema = step1Schema
  .merge(step2Schema)
  .merge(step3Schema);

type CompleteFormData = z.infer<typeof completeFormSchema>;

function MultiStepForm() {
  const [step, setStep] = useState(1);
  
  const methods = useForm<CompleteFormData>({
    resolver: zodResolver(
      step === 1
        ? step1Schema
        : step === 2
        ? step2Schema
        : step3Schema
    )
  });

  const onSubmit = async (data: CompleteFormData) => {
    if (step < 3) {
      setStep(step + 1);
    } else {
      // Final submission - validate complete form
      const result = completeFormSchema.safeParse(data);
      if (result.success) {
        await submitForm(result.data);
      }
    }
  };

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {step === 1 && <Step1 />}
        {step === 2 && <Step2 />}
        {step === 3 && <Step3 />}
        
        <button type="submit">
          {step === 3 ? 'Submit' : 'Next'}
        </button>
      </form>
    </FormProvider>
  );
}
```

## Common Mistakes

### 1. Not Using Type Inference

```typescript
// ❌ Bad - manually typing
interface FormData {
  email: string;
  password: string;
}

const schema = z.object({
  email: z.string().email(),
  password: z.string()
});

// ✅ Good - infer from schema
const schema = z.object({
  email: z.string().email(),
  password: z.string()
});

type FormData = z.infer<typeof schema>;
```

### 2. Forgetting valueAsNumber/valueAsDate

```typescript
// ❌ Bad - age will be string
<input {...register('age')} type="number" />

// ✅ Good - age will be number
<input {...register('age', { valueAsNumber: true })} type="number" />

// Or transform in schema
const schema = z.object({
  age: z.preprocess((val) => Number(val), z.number())
});
```

### 3. Not Handling Optional Fields

```typescript
// ❌ Bad - optional field not handled
const schema = z.object({
  website: z.string().url() // Empty string fails
});

// ✅ Good - handle empty optional fields
const schema = z.object({
  website: z.string().url().optional().or(z.literal(''))
});
```

### 4. Overusing Async Validation

```typescript
// ❌ Bad - validates on every keystroke
const { register } = useForm({
  resolver: zodResolver(asyncSchema),
  mode: 'onChange' // Too many API calls!
});

// ✅ Good - validate on blur
const { register } = useForm({
  resolver: zodResolver(asyncSchema),
  mode: 'onBlur'
});
```

## Best Practices

### 1. Organize Schemas

```typescript
// schemas/user.ts
export const userSchemas = {
  login: z.object({
    email: z.string().email(),
    password: z.string()
  }),

  signup: z.object({
    username: z.string().min(3),
    email: z.string().email(),
    password: z.string().min(8)
  }),

  profile: z.object({
    name: z.string(),
    bio: z.string().optional(),
    website: z.string().url().optional()
  })
};

// Use in forms
import { userSchemas } from './schemas/user';

function LoginForm() {
  const { register } = useForm({
    resolver: zodResolver(userSchemas.login)
  });
}
```

### 2. Create Reusable Validators

```typescript
// utils/validators.ts
export const validators = {
  email: z.string().email('Invalid email'),
  
  password: z
    .string()
    .min(8)
    .regex(/[A-Z]/, 'Need uppercase')
    .regex(/[a-z]/, 'Need lowercase')
    .regex(/[0-9]/, 'Need number'),
  
  phone: z.string().regex(/^\+?[1-9]\d{1,14}$/),
  
  slug: z.string().regex(/^[a-z0-9-]+$/),
  
  url: z.string().url(),
  
  uuid: z.string().uuid()
};
```

### 3. Test Your Schemas

```typescript
import { describe, it, expect } from 'vitest';

describe('signupSchema', () => {
  it('validates correct data', () => {
    const data = {
      email: 'test@example.com',
      password: 'Password123!',
      confirmPassword: 'Password123!'
    };

    const result = signupSchema.safeParse(data);
    expect(result.success).toBe(true);
  });

  it('rejects invalid email', () => {
    const data = {
      email: 'invalid-email',
      password: 'Password123!',
      confirmPassword: 'Password123!'
    };

    const result = signupSchema.safeParse(data);
    expect(result.success).toBe(false);
    if (!result.success) {
      expect(result.error.issues[0].path).toEqual(['email']);
    }
  });

  it('rejects mismatched passwords', () => {
    const data = {
      email: 'test@example.com',
      password: 'Password123!',
      confirmPassword: 'Different123!'
    };

    const result = signupSchema.safeParse(data);
    expect(result.success).toBe(false);
  });
});
```

### 4. Provide Clear Error Messages

```typescript
const schema = z.object({
  username: z
    .string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username must be at most 20 characters')
    .regex(
      /^[a-zA-Z0-9_]+$/,
      'Username can only contain letters, numbers, and underscores'
    ),
  
  age: z
    .number({
      required_error: 'Age is required',
      invalid_type_error: 'Age must be a number'
    })
    .int('Age must be a whole number')
    .min(18, 'You must be at least 18 years old')
});
```

## Interview Questions

### Q1: What advantages does Zod provide over manual validation?

**Answer:**
- TypeScript type inference (types from schema)
- Runtime validation
- Composable schemas (reuse, extend, merge)
- Better error messages
- Automatic form integration
- Async validation support
- Transform/preprocess data
- Single source of truth for types and validation

### Q2: How do you handle conditional validation with Zod?

**Answer:** Use discriminated unions or refine:
- Discriminated unions for different form structures
- Refine for cross-field validation
- SuperRefine for complex conditional logic
- Dynamic schema generation based on conditions

### Q3: What's the difference between refine, superRefine, and transform?

**Answer:**
- `refine`: Simple validation returning boolean
- `superRefine`: Complex validation with full control over errors
- `transform`: Modify data before/after validation
- `preprocess`: Transform before validation

### Q4: How do you optimize performance with async validation?

**Answer:**
- Use `mode: 'onBlur'` instead of `onChange`
- Debounce validation calls
- Cache validation results
- Validate only changed fields
- Use field-level validation instead of form-level

## Resources

### Official Documentation
- Zod Documentation: https://zod.dev/
- React Hook Form Resolvers: https://github.com/react-hook-form/resolvers

### Tutorials
- Zod Tutorial: https://zod.dev/
- Integration Guide: https://react-hook-form.com/get-started#SchemaValidation

### Tools
- Zod to TypeScript: https://github.com/sachinraja/zod-to-ts
- Zod Mock: https://github.com/anatine/zod-plugins

---

**Next Steps:**
- Practice building type-safe forms
- Create reusable validation schemas
- Implement async validation
- Build complex multi-step forms with Zod
