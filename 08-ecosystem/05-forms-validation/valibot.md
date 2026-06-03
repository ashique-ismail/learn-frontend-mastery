# Valibot - Lightweight Schema Validation

## Overview

Valibot is a modern, lightweight schema validation library designed for optimal bundle size and tree-shaking. With a modular architecture where every validation function is a separate import, Valibot achieves significantly smaller bundle sizes compared to Zod, Yup, or Joi. It provides TypeScript-first design with excellent type inference and a functional, composable API.

## Installation and Setup

```bash
# Install Valibot
npm install valibot

# TypeScript types included
# No additional packages needed

# Common pairings
npm install valibot react react-hook-form
```

## Core Concepts

### Basic Schema Definition

```typescript
import * as v from 'valibot';

// Simple schema
const UserSchema = v.object({
  username: v.string([
    v.minLength(3, 'Username must be at least 3 characters'),
    v.maxLength(30, 'Username cannot exceed 30 characters'),
  ]),
  email: v.string([v.email('Invalid email address')]),
  age: v.number([
    v.minValue(18, 'Must be at least 18'),
    v.maxValue(120, 'Invalid age'),
  ]),
  isActive: v.boolean(),
});

// Infer TypeScript type
type User = v.Output<typeof UserSchema>;

// Validate data
const result = v.safeParse(UserSchema, {
  username: 'john_doe',
  email: 'john@example.com',
  age: 25,
  isActive: true,
});

if (result.success) {
  console.log('Valid data:', result.output);
} else {
  console.error('Validation errors:', result.issues);
}

// Parse with exception on error
try {
  const data = v.parse(UserSchema, rawData);
  console.log('Valid:', data);
} catch (error) {
  console.error('Invalid:', error);
}
```

### Schema Types

```typescript
import * as v from 'valibot';

// String schema
const stringSchema = v.string([
  v.minLength(3),
  v.maxLength(50),
  v.regex(/^[a-zA-Z]+$/),
  v.trim(),
  v.toLowerCase(),
]);

// Number schema
const numberSchema = v.number([
  v.minValue(0),
  v.maxValue(100),
  v.integer(),
  v.multipleOf(5),
]);

// Boolean schema
const booleanSchema = v.boolean();

// Date schema
const dateSchema = v.date([
  v.minValue(new Date('2020-01-01')),
  v.maxValue(new Date()),
]);

// Array schema
const arraySchema = v.array(v.string(), [
  v.minLength(1, 'At least one item required'),
  v.maxLength(10, 'Maximum 10 items'),
]);

// Object schema
const objectSchema = v.object({
  name: v.string(),
  age: v.number(),
});

// Tuple schema
const tupleSchema = v.tuple([v.string(), v.number(), v.boolean()]);

// Union schema
const unionSchema = v.union([v.string(), v.number()]);

// Enum schema
const enumSchema = v.enum(['admin', 'user', 'guest'] as const);

// Literal schema
const literalSchema = v.literal('specific-value');

// Null/Undefined
const nullSchema = v.null();
const undefinedSchema = v.undefined();

// Nullable (value or null)
const nullableSchema = v.nullable(v.string());

// Optional (value or undefined)
const optionalSchema = v.optional(v.string());

// Nullish (value, null, or undefined)
const nullishSchema = v.nullish(v.string());
```

### Validation Actions (Pipelines)

```typescript
import * as v from 'valibot';

// String validations
const emailSchema = v.string([
  v.trim(),
  v.toLowerCase(),
  v.email('Invalid email'),
  v.endsWith('@company.com', 'Must be a company email'),
]);

// Number validations
const ageSchema = v.number([
  v.integer('Age must be a whole number'),
  v.minValue(18, 'Must be at least 18'),
  v.maxValue(120, 'Invalid age'),
  v.notValue(100, 'Cannot be exactly 100'),
]);

// Array validations
const tagsSchema = v.array(v.string(), [
  v.minLength(1, 'At least one tag required'),
  v.maxLength(5, 'Maximum 5 tags'),
  v.includes('required-tag', 'Must include required-tag'),
]);

// String pattern validations
const usernameSchema = v.string([
  v.trim(),
  v.minLength(3),
  v.maxLength(30),
  v.regex(/^[a-zA-Z0-9_]+$/, 'Only alphanumeric and underscore'),
  v.notIncludes(' ', 'No spaces allowed'),
]);

// URL validation
const urlSchema = v.string([
  v.url('Must be a valid URL'),
  v.startsWith('https://', 'Must use HTTPS'),
]);

// Custom validation
const customSchema = v.string([
  v.custom((value) => !value.includes('forbidden'), 'Contains forbidden word'),
]);

// Async validation
const asyncSchema = v.string([
  v.customAsync(async (value) => {
    const response = await fetch(`/api/check?value=${value}`);
    const { valid } = await response.json();
    return valid;
  }, 'Value already exists'),
]);
```

## Advanced Patterns

### Object Schema Composition

```typescript
import * as v from 'valibot';

// Base schemas
const BaseUserSchema = v.object({
  id: v.number(),
  username: v.string([v.minLength(3)]),
  email: v.string([v.email()]),
});

// Extend schema
const ExtendedUserSchema = v.object({
  ...BaseUserSchema.entries,
  role: v.enum(['admin', 'user', 'guest'] as const),
  createdAt: v.date(),
});

// Merge schemas
const addressSchema = v.object({
  street: v.string(),
  city: v.string(),
  zipCode: v.string([v.regex(/^\d{5}$/)]),
});

const UserWithAddressSchema = v.object({
  ...BaseUserSchema.entries,
  address: addressSchema,
});

// Pick specific fields
const LoginSchema = v.object({
  email: BaseUserSchema.entries.email,
  password: v.string([v.minLength(8)]),
});

// Partial schema (all fields optional)
const UpdateUserSchema = v.partial(BaseUserSchema);

// Required schema (all fields required)
const RequiredUserSchema = v.required(UpdateUserSchema);
```

### Nested and Recursive Schemas

```typescript
import * as v from 'valibot';

// Nested object schema
const AddressSchema = v.object({
  street: v.string(),
  city: v.string(),
  country: v.string(),
  coordinates: v.object({
    lat: v.number(),
    lng: v.number(),
  }),
});

const UserWithNestedSchema = v.object({
  name: v.string(),
  addresses: v.array(AddressSchema),
});

// Recursive schema for tree structures
type Category = {
  id: number;
  name: string;
  children?: Category[];
};

const CategorySchema: v.BaseSchema<Category> = v.object({
  id: v.number(),
  name: v.string(),
  children: v.optional(v.array(v.lazy(() => CategorySchema))),
});

// Recursive schema for linked lists
type LinkedListNode = {
  value: number;
  next?: LinkedListNode;
};

const LinkedListSchema: v.BaseSchema<LinkedListNode> = v.object({
  value: v.number(),
  next: v.optional(v.lazy(() => LinkedListSchema)),
});
```

### Union and Discriminated Unions

```typescript
import * as v from 'valibot';

// Simple union
const StringOrNumberSchema = v.union([v.string(), v.number()]);

// Discriminated union (tagged union)
const PaymentMethodSchema = v.union([
  v.object({
    type: v.literal('credit_card'),
    cardNumber: v.string([v.regex(/^\d{16}$/)]),
    cvv: v.string([v.length(3)]),
    expiry: v.string(),
  }),
  v.object({
    type: v.literal('paypal'),
    email: v.string([v.email()]),
  }),
  v.object({
    type: v.literal('bank_transfer'),
    accountNumber: v.string(),
    routingNumber: v.string(),
  }),
]);

type PaymentMethod = v.Output<typeof PaymentMethodSchema>;

// Validate discriminated union
const payment: PaymentMethod = {
  type: 'credit_card',
  cardNumber: '1234567890123456',
  cvv: '123',
  expiry: '12/25',
};

const result = v.safeParse(PaymentMethodSchema, payment);

// Type narrowing works automatically
if (result.success && result.output.type === 'credit_card') {
  console.log('Card number:', result.output.cardNumber);
}
```

### Transformations

```typescript
import * as v from 'valibot';

// Transform string to number
const StringToNumberSchema = v.transform(
  v.string([v.regex(/^\d+$/, 'Must be numeric')]),
  (value) => parseInt(value, 10)
);

// Transform and validate
const PriceSchema = v.transform(
  v.string([v.regex(/^\$?\d+(\.\d{2})?$/)]),
  (value) => parseFloat(value.replace('$', '')),
  v.number([v.minValue(0)])
);

// Multiple transformations
const NormalizedEmailSchema = v.transform(
  v.string([v.email()]),
  (value) => value.toLowerCase().trim()
);

// Complex transformation
const DateStringSchema = v.transform(
  v.string([v.isoDate()]),
  (value) => new Date(value),
  v.date([v.minValue(new Date('2020-01-01'))])
);

// Transform array
const CsvToArraySchema = v.transform(
  v.string(),
  (value) => value.split(',').map((s) => s.trim()),
  v.array(v.string([v.minLength(1)]))
);

// Conditional transformation
const ConditionalSchema = v.object({
  type: v.enum(['celsius', 'fahrenheit'] as const),
  temperature: v.number(),
}).transform((value) => {
  if (value.type === 'fahrenheit') {
    return {
      ...value,
      temperature: ((value.temperature - 32) * 5) / 9,
      type: 'celsius' as const,
    };
  }
  return value;
});
```

### Default Values and Fallbacks

```typescript
import * as v from 'valibot';

// Default value
const UserSchema = v.object({
  username: v.string(),
  role: v.optional(v.string(), 'user'), // Default: 'user'
  isActive: v.optional(v.boolean(), true), // Default: true
  createdAt: v.optional(v.date(), () => new Date()), // Default: current date
});

// Fallback on error
const SafeNumberSchema = v.fallback(
  v.number([v.minValue(0)]),
  0 // Fallback value if validation fails
);

// Fallback with function
const SafeDateSchema = v.fallback(
  v.date(),
  () => new Date() // Fallback function
);

// Nullish with default
const ConfigSchema = v.object({
  apiUrl: v.nullish(v.string([v.url()]), 'https://api.example.com'),
  timeout: v.nullish(v.number([v.minValue(0)]), 5000),
  retries: v.nullish(v.number([v.integer(), v.minValue(0)]), 3),
});
```

## React Integration

### React Hook Form Integration

```typescript
import { useForm } from 'react-hook-form';
import { valibotResolver } from '@hookform/resolvers/valibot';
import * as v from 'valibot';

// Define schema
const FormSchema = v.object({
  username: v.string([
    v.minLength(3, 'Username must be at least 3 characters'),
    v.maxLength(30, 'Username cannot exceed 30 characters'),
  ]),
  email: v.string([v.email('Invalid email address')]),
  age: v.number([
    v.integer('Age must be a whole number'),
    v.minValue(18, 'Must be at least 18'),
  ]),
  agreeToTerms: v.boolean([
    v.custom((value) => value === true, 'Must accept terms'),
  ]),
});

type FormData = v.Output<typeof FormSchema>;

function RegistrationForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: valibotResolver(FormSchema),
    mode: 'onBlur',
  });

  const onSubmit = async (data: FormData) => {
    console.log('Valid data:', data);
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
        <input
          {...register('age', { valueAsNumber: true })}
          type="number"
          placeholder="Age"
        />
        {errors.age && <span>{errors.age.message}</span>}
      </div>

      <div>
        <label>
          <input {...register('agreeToTerms')} type="checkbox" />
          I agree to terms and conditions
        </label>
        {errors.agreeToTerms && <span>{errors.agreeToTerms.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Register'}
      </button>
    </form>
  );
}
```

### Custom React Hook

```typescript
import { useState, useCallback } from 'react';
import * as v from 'valibot';

function useValibotForm<T extends v.BaseSchema>(
  schema: T,
  initialValues: v.Output<T>
) {
  const [values, setValues] = useState<v.Output<T>>(initialValues);
  const [errors, setErrors] = useState<Record<string, string>>({});
  const [touched, setTouched] = useState<Record<string, boolean>>({});

  const validateField = useCallback(
    (fieldName: keyof v.Output<T>, value: any) => {
      const result = v.safeParse(schema, { ...values, [fieldName]: value });

      if (!result.success) {
        const fieldError = result.issues.find((issue) =>
          issue.path?.some((p) => p.key === fieldName)
        );

        if (fieldError) {
          setErrors((prev) => ({
            ...prev,
            [fieldName]: fieldError.message,
          }));
          return false;
        }
      }

      setErrors((prev) => {
        const newErrors = { ...prev };
        delete newErrors[fieldName as string];
        return newErrors;
      });

      return true;
    },
    [schema, values]
  );

  const handleChange = useCallback(
    (fieldName: keyof v.Output<T>, value: any) => {
      setValues((prev) => ({ ...prev, [fieldName]: value }));

      if (touched[fieldName as string]) {
        validateField(fieldName, value);
      }
    },
    [touched, validateField]
  );

  const handleBlur = useCallback(
    (fieldName: keyof v.Output<T>) => {
      setTouched((prev) => ({ ...prev, [fieldName]: true }));
      validateField(fieldName, values[fieldName]);
    },
    [values, validateField]
  );

  const handleSubmit = useCallback(
    (onSubmit: (data: v.Output<T>) => void) => {
      return (e: React.FormEvent) => {
        e.preventDefault();

        const result = v.safeParse(schema, values);

        if (result.success) {
          onSubmit(result.output);
        } else {
          const newErrors: Record<string, string> = {};
          result.issues.forEach((issue) => {
            const key = issue.path?.[0]?.key as string;
            if (key && !newErrors[key]) {
              newErrors[key] = issue.message;
            }
          });
          setErrors(newErrors);
        }
      };
    },
    [schema, values]
  );

  return {
    values,
    errors,
    touched,
    handleChange,
    handleBlur,
    handleSubmit,
  };
}

// Usage
const UserSchema = v.object({
  username: v.string([v.minLength(3)]),
  email: v.string([v.email()]),
});

function MyForm() {
  const { values, errors, touched, handleChange, handleBlur, handleSubmit } =
    useValibotForm(UserSchema, {
      username: '',
      email: '',
    });

  const onSubmit = (data: v.Output<typeof UserSchema>) => {
    console.log('Submit:', data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        value={values.username}
        onChange={(e) => handleChange('username', e.target.value)}
        onBlur={() => handleBlur('username')}
      />
      {touched.username && errors.username && <span>{errors.username}</span>}

      <input
        value={values.email}
        onChange={(e) => handleChange('email', e.target.value)}
        onBlur={() => handleBlur('email')}
      />
      {touched.email && errors.email && <span>{errors.email}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

## Bundle Size Optimization

```typescript
import * as v from 'valibot';

// Tree-shakeable imports - only import what you need
import { object, string, number, minLength, email, parse } from 'valibot';

// Optimized schema (smaller bundle)
const OptimizedSchema = object({
  email: string([email()]),
  age: number([minLength(18)]),
});

// Each validation is tree-shakeable
const data = parse(OptimizedSchema, rawData);

// Compare bundle sizes:
// Valibot: ~5-10 KB (only used validators)
// Zod: ~15-20 KB (entire library)
// Yup: ~20-30 KB (entire library)
// Joi: ~40-50 KB (entire library)

// Modular validation functions
import { isEmail } from 'valibot';

function validateEmailQuick(email: string): boolean {
  return isEmail(email);
}
```

## Common Mistakes

1. **Not using safeParse**: Using `parse()` without error handling instead of `safeParse()`.

2. **Missing validation actions**: Forgetting to pass validation array to schema types.

3. **Incorrect type inference**: Not using `v.Output<>` for type inference.

4. **Over-nesting validation logic**: Creating complex nested structures instead of composing smaller schemas.

5. **Not leveraging tree-shaking**: Importing entire library instead of specific functions.

6. **Ignoring async validation**: Not handling async custom validators properly.

7. **Misusing transforms**: Applying transformations that change type without proper typing.

8. **Poor error handling**: Not checking `result.success` before accessing `result.output`.

9. **Default value confusion**: Using optional when default values would be clearer.

10. **Not testing schemas**: Forgetting to unit test complex validation schemas.

## Best Practices

1. **Import only what you need**: Use named imports for optimal tree-shaking and bundle size.

2. **Use safeParse for validation**: Always use `safeParse()` and check `success` before accessing data.

3. **Leverage type inference**: Use `v.Output<>` to infer TypeScript types from schemas.

4. **Compose reusable schemas**: Build complex schemas from smaller, reusable pieces.

5. **Add clear error messages**: Provide user-friendly messages for all validations.

6. **Use transforms for normalization**: Clean and normalize data using transform functions.

7. **Leverage defaults wisely**: Use optional with defaults for cleaner schema definitions.

8. **Handle async validation**: Properly handle async custom validators with appropriate UI feedback.

9. **Test your schemas**: Write unit tests for validation logic.

10. **Monitor bundle size**: Use bundle analyzers to ensure Valibot's size benefits are realized.

## When to Use Valibot

**Use Valibot when:**
- Bundle size is critical concern
- Building client-side applications
- Need excellent tree-shaking
- Want TypeScript-first validation
- Need modular, composable validation
- Building performance-critical apps
- Want modern, functional API

**Consider alternatives when:**
- Need extensive ecosystem (use Zod or Yup)
- Working with legacy codebases (use Joi)
- Need maximum validation performance (use AJV)
- Want testing-like syntax (use Vest)
- Need mature, battle-tested library (use Yup/Joi)

## Interview Questions

### Q1: What makes Valibot's bundle size smaller than Zod or Yup?

**Answer**: Valibot achieves smaller bundle size through extreme modularity where every validation function is a separate export, enabling perfect tree-shaking. Unlike Zod or Yup which include validation logic in monolithic classes, Valibot's functional approach means you only bundle the exact validators you use. This can result in 50-70% smaller bundles compared to Zod.

### Q2: How do you create a schema with transformations in Valibot?

**Answer**: Use the `transform()` function:

```typescript
const PriceSchema = v.transform(
  v.string([v.regex(/^\$?\d+(\.\d{2})?$/)]),
  (value) => parseFloat(value.replace('$', '')),
  v.number([v.minValue(0)])
);
```

The first argument is input schema, second is transform function, third (optional) is output schema for additional validation.

### Q3: How does Valibot handle optional and nullable values?

**Answer**: Valibot provides separate functions: `optional()` for undefined values, `nullable()` for null values, and `nullish()` for both:

```typescript
const Schema = v.object({
  optional: v.optional(v.string()), // string | undefined
  nullable: v.nullable(v.string()), // string | null
  nullish: v.nullish(v.string()),   // string | null | undefined
  withDefault: v.optional(v.string(), 'default'), // string (never undefined)
});
```

### Q4: How do you implement discriminated unions in Valibot?

**Answer**: Use `union()` with `literal()` type for discriminator:

```typescript
const PaymentSchema = v.union([
  v.object({
    type: v.literal('card'),
    cardNumber: v.string(),
  }),
  v.object({
    type: v.literal('paypal'),
    email: v.string([v.email()]),
  }),
]);
```

TypeScript automatically narrows types based on the discriminator field.

### Q5: How do you create recursive schemas in Valibot?

**Answer**: Use `lazy()` function to defer schema resolution:

```typescript
type Category = {
  id: number;
  name: string;
  children?: Category[];
};

const CategorySchema: v.BaseSchema<Category> = v.object({
  id: v.number(),
  name: v.string(),
  children: v.optional(v.array(v.lazy(() => CategorySchema))),
});
```

### Q6: How does Valibot's type inference compare to Zod?

**Answer**: Both provide excellent type inference, but Valibot uses `v.Output<>` utility type while Zod uses `z.infer<>`. Both achieve similar results, but Valibot's functional architecture can provide slightly better performance in TypeScript compilation. The main difference is syntax preference rather than capability.

### Q7: How do you handle async validation in Valibot?

**Answer**: Use `customAsync()` validation action:

```typescript
const Schema = v.string([
  v.email(),
  v.customAsync(async (value) => {
    const response = await fetch(`/api/check-email?email=${value}`);
    const { available } = await response.json();
    return available;
  }, 'Email already registered'),
]);

// Parse asynchronously
const result = await v.parseAsync(Schema, data);
```

## Key Takeaways

1. Valibot prioritizes bundle size through extreme modularity and perfect tree-shaking, achieving 50-70% smaller bundles than alternatives.

2. The functional, composable API uses validation actions (pipelines) passed as arrays to schema types.

3. Type inference uses `v.Output<>` utility type to extract TypeScript types from schemas.

4. Validation has two main methods: `parse()` throws on error, `safeParse()` returns result object with success status.

5. Schemas are highly composable through object spreading, `partial()`, `required()`, and schema merging.

6. Transformations use `transform()` function to modify validated data before returning it.

7. Optional, nullable, and nullish values are handled through separate wrapper functions with clear semantics.

8. Recursive schemas use `lazy()` function to defer resolution and avoid circular reference issues.

9. Discriminated unions provide type-safe validation with automatic TypeScript type narrowing.

10. Every import is tree-shakeable, allowing you to import only the validators you actually use.

## Resources

- [Official Documentation](https://valibot.dev/)
- [GitHub Repository](https://github.com/fabian-hiller/valibot)
- [Bundle Size Comparison](https://bundlephobia.com/package/valibot)
- [Valibot vs Zod](https://valibot.dev/guides/introduction/)
- [API Reference](https://valibot.dev/api/)
- [Migration from Zod](https://valibot.dev/guides/migrate-from-zod/)
