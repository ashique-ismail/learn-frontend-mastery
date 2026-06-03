# Yup Schema Validation

## Overview

Yup is a JavaScript schema builder for runtime value parsing and validation. It provides a declarative API for defining validation schemas that can be reused across your application. Yup is particularly popular in the React ecosystem and integrates seamlessly with form libraries like Formik and React Hook Form.

## Installation and Setup

```bash
# Install Yup
npm install yup

# With TypeScript
npm install yup @types/yup

# Common pairings
npm install yup formik react-hook-form
```

## Core Concepts

### Basic Schema Definition

```typescript
import * as yup from 'yup';

// Simple schema
const userSchema = yup.object({
  username: yup.string().required('Username is required'),
  email: yup.string().email('Invalid email').required('Email is required'),
  age: yup.number().positive().integer().min(18, 'Must be at least 18'),
  website: yup.string().url('Must be a valid URL').nullable(),
});

// Validate data
const data = {
  username: 'john_doe',
  email: 'john@example.com',
  age: 25,
  website: 'https://example.com',
};

try {
  const validated = await userSchema.validate(data);
  console.log('Valid data:', validated);
} catch (error) {
  console.error('Validation error:', error.message);
}
```

### Schema Types

```typescript
import * as yup from 'yup';

// String schema
const stringSchema = yup.string()
  .min(3, 'Too short')
  .max(50, 'Too long')
  .matches(/^[a-zA-Z]+$/, 'Only letters allowed')
  .trim()
  .lowercase();

// Number schema
const numberSchema = yup.number()
  .min(0)
  .max(100)
  .positive()
  .integer()
  .truncate(); // Round to integer

// Boolean schema
const booleanSchema = yup.boolean()
  .required()
  .oneOf([true], 'Must accept terms');

// Date schema
const dateSchema = yup.date()
  .min(new Date(), 'Must be in the future')
  .max(new Date('2030-12-31'), 'Too far in future');

// Array schema
const arraySchema = yup.array()
  .of(yup.string())
  .min(2, 'At least 2 items required')
  .max(10, 'Maximum 10 items')
  .compact() // Remove falsy values
  .ensure(); // Ensure always returns array

// Object schema
const objectSchema = yup.object({
  name: yup.string().required(),
  nested: yup.object({
    value: yup.number(),
  }),
});

// Mixed schema (any type)
const mixedSchema = yup.mixed()
  .oneOf(['admin', 'user', 'guest'])
  .required();
```

### Nullable and Optional Fields

```typescript
import * as yup from 'yup';

const schema = yup.object({
  // Optional: can be undefined
  optionalField: yup.string(),
  
  // Nullable: can be null
  nullableField: yup.string().nullable(),
  
  // Both nullable and optional
  flexibleField: yup.string().nullable().optional(),
  
  // Not required but with default
  withDefault: yup.string().default('default value'),
  
  // Defined (not undefined, but can be null)
  definedField: yup.string().defined(),
  
  // Strip unknown fields
  strictField: yup.string().strict(),
});

// Transform undefined to null
const transformSchema = yup.object({
  field: yup.string().transform((value, originalValue) => 
    originalValue === undefined ? null : value
  ),
});
```

## Advanced Validation Patterns

### Conditional Validation (when)

```typescript
import * as yup from 'yup';

const registrationSchema = yup.object({
  accountType: yup.string().oneOf(['personal', 'business']).required(),
  
  // Company name required only for business accounts
  companyName: yup.string().when('accountType', {
    is: 'business',
    then: (schema) => schema.required('Company name required for business'),
    otherwise: (schema) => schema.notRequired(),
  }),
  
  // Multiple conditions
  taxId: yup.string().when(['accountType', 'country'], {
    is: (accountType: string, country: string) => 
      accountType === 'business' && country === 'US',
    then: (schema) => schema.required('Tax ID required for US businesses'),
    otherwise: (schema) => schema.notRequired(),
  }),
  
  country: yup.string().required(),
});

// Using when with multiple fields
const passwordSchema = yup.object({
  hasPassword: yup.boolean(),
  password: yup.string().when('hasPassword', {
    is: true,
    then: (schema) => schema
      .required('Password required')
      .min(8, 'At least 8 characters'),
  }),
  confirmPassword: yup.string().when('hasPassword', {
    is: true,
    then: (schema) => schema
      .required('Confirm password')
      .oneOf([yup.ref('password')], 'Passwords must match'),
  }),
});
```

### Cross-Field Validation

```typescript
import * as yup from 'yup';

const dateRangeSchema = yup.object({
  startDate: yup.date().required('Start date required'),
  endDate: yup.date()
    .required('End date required')
    .min(yup.ref('startDate'), 'End date must be after start date'),
});

const passwordChangeSchema = yup.object({
  currentPassword: yup.string().required('Current password required'),
  newPassword: yup.string()
    .required('New password required')
    .min(8, 'At least 8 characters')
    .notOneOf(
      [yup.ref('currentPassword')],
      'New password must be different'
    ),
  confirmPassword: yup.string()
    .required('Confirm new password')
    .oneOf([yup.ref('newPassword')], 'Passwords must match'),
});

// Complex cross-field validation
const billingSchema = yup.object({
  sameAsShipping: yup.boolean(),
  shippingAddress: yup.string().required(),
  billingAddress: yup.string().when('sameAsShipping', {
    is: false,
    then: (schema) => schema.required('Billing address required'),
  }),
}).test('address-validation', 'Invalid address combination', function(value) {
  const { sameAsShipping, shippingAddress, billingAddress } = value;
  
  if (sameAsShipping && billingAddress && billingAddress !== shippingAddress) {
    return this.createError({
      path: 'billingAddress',
      message: 'Billing address should match shipping when same address is selected',
    });
  }
  
  return true;
});
```

### Custom Validation Methods

```typescript
import * as yup from 'yup';

// Add custom method to string schema
yup.addMethod(yup.string, 'strongPassword', function(message) {
  return this.test('strong-password', message, function(value) {
    const { path, createError } = this;
    
    if (!value) return true; // Let required() handle empty values
    
    const hasUpperCase = /[A-Z]/.test(value);
    const hasLowerCase = /[a-z]/.test(value);
    const hasNumber = /[0-9]/.test(value);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(value);
    
    if (!hasUpperCase) {
      return createError({ 
        path, 
        message: message || 'Must contain uppercase letter' 
      });
    }
    
    if (!hasLowerCase) {
      return createError({ 
        path, 
        message: message || 'Must contain lowercase letter' 
      });
    }
    
    if (!hasNumber) {
      return createError({ 
        path, 
        message: message || 'Must contain number' 
      });
    }
    
    if (!hasSpecialChar) {
      return createError({ 
        path, 
        message: message || 'Must contain special character' 
      });
    }
    
    return true;
  });
});

// Extend TypeScript types
declare module 'yup' {
  interface StringSchema {
    strongPassword(message?: string): this;
  }
}

// Use custom method
const schema = yup.object({
  password: yup.string().required().strongPassword(),
});

// Custom validation for phone numbers
yup.addMethod(yup.string, 'phone', function(message = 'Invalid phone number') {
  return this.test('phone', message, function(value) {
    if (!value) return true;
    
    // Simple phone validation (US format)
    const phoneRegex = /^\+?1?\s*\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})$/;
    return phoneRegex.test(value);
  });
});

declare module 'yup' {
  interface StringSchema {
    phone(message?: string): this;
  }
}
```

### Custom Test Validations

```typescript
import * as yup from 'yup';

const fileSchema = yup.mixed()
  .test('fileSize', 'File too large', (value: any) => {
    if (!value) return true;
    return value.size <= 5 * 1024 * 1024; // 5MB
  })
  .test('fileType', 'Invalid file type', (value: any) => {
    if (!value) return true;
    return ['image/jpeg', 'image/png', 'image/gif'].includes(value.type);
  });

// Test with async validation
const usernameSchema = yup.string()
  .required('Username required')
  .min(3, 'At least 3 characters')
  .test('unique-username', 'Username already taken', async (value) => {
    if (!value) return true;
    
    // Simulate API call
    const response = await fetch(`/api/check-username?username=${value}`);
    const { available } = await response.json();
    return available;
  });

// Test with context and parent values
const discountSchema = yup.object({
  price: yup.number().required().positive(),
  discount: yup.number()
    .required()
    .test('valid-discount', 'Discount invalid', function(value) {
      const { price } = this.parent;
      
      if (value < 0) {
        return this.createError({ message: 'Discount cannot be negative' });
      }
      
      if (value > price) {
        return this.createError({ 
          message: `Discount cannot exceed price of $${price}` 
        });
      }
      
      return true;
    }),
});
```

## Async Validation

```typescript
import * as yup from 'yup';

// Async schema validation
const asyncUserSchema = yup.object({
  email: yup.string()
    .email('Invalid email')
    .required('Email required')
    .test('email-exists', 'Email already registered', async (value) => {
      if (!value) return true;
      
      const response = await fetch(`/api/check-email?email=${value}`);
      const { exists } = await response.json();
      return !exists;
    }),
  
  username: yup.string()
    .required('Username required')
    .test('username-available', 'Username taken', async (value) => {
      if (!value) return true;
      
      // Debounce in real implementation
      const response = await fetch(`/api/check-username?username=${value}`);
      const { available } = await response.json();
      return available;
    }),
});

// Validate with abort signal (for cancellation)
const validateWithCancel = async (data: any, signal: AbortSignal) => {
  try {
    const validated = await asyncUserSchema.validate(data, {
      abortEarly: false, // Collect all errors
      context: { signal }, // Pass signal to tests
    });
    return { data: validated, errors: null };
  } catch (error) {
    if (error instanceof yup.ValidationError) {
      return { data: null, errors: error.errors };
    }
    throw error;
  }
};

// Usage with AbortController
const controller = new AbortController();
const validation = validateWithCancel(data, controller.signal);

// Cancel validation
setTimeout(() => controller.abort(), 5000);
```

## Transforms and Preprocessing

```typescript
import * as yup from 'yup';

const transformSchema = yup.object({
  // Trim whitespace
  username: yup.string()
    .trim()
    .lowercase()
    .required(),
  
  // Convert to uppercase
  countryCode: yup.string()
    .uppercase()
    .length(2, 'Must be 2 characters'),
  
  // Custom transform
  phone: yup.string()
    .transform((value, originalValue) => {
      // Remove all non-digits
      return originalValue.replace(/\D/g, '');
    })
    .matches(/^\d{10}$/, 'Must be 10 digits'),
  
  // Transform with conditional logic
  price: yup.number()
    .transform((value, originalValue) => {
      // Handle currency strings
      if (typeof originalValue === 'string') {
        return parseFloat(originalValue.replace(/[$,]/g, ''));
      }
      return value;
    })
    .positive('Price must be positive'),
  
  // Array transform
  tags: yup.array()
    .of(yup.string())
    .transform((value, originalValue) => {
      // Split comma-separated string into array
      if (typeof originalValue === 'string') {
        return originalValue.split(',').map(tag => tag.trim());
      }
      return value;
    }),
  
  // Date transform
  birthDate: yup.date()
    .transform((value, originalValue) => {
      // Parse various date formats
      if (originalValue instanceof Date) return originalValue;
      if (typeof originalValue === 'string') {
        const parsed = new Date(originalValue);
        return isNaN(parsed.getTime()) ? undefined : parsed;
      }
      return value;
    })
    .max(new Date(), 'Cannot be in future'),
});
```

## Error Handling and Messages

```typescript
import * as yup from 'yup';

// Custom error messages
const schema = yup.object({
  email: yup.string()
    .email('Please provide a valid email address')
    .required('Email is required to continue'),
  
  age: yup.number()
    .typeError('Age must be a number')
    .required('Please enter your age')
    .min(18, 'You must be at least ${min} years old')
    .max(120, 'Age cannot exceed ${max} years'),
});

// Validation options
try {
  const validated = await schema.validate(data, {
    abortEarly: false, // Collect all errors
    stripUnknown: true, // Remove unknown fields
    strict: false, // Allow type casting
    context: { userId: 123 }, // Additional context
  });
} catch (error) {
  if (error instanceof yup.ValidationError) {
    // Access all errors
    console.log(error.errors); // Array of error messages
    console.log(error.inner); // Array of inner errors
    
    // Get errors by path
    const errorsByPath = error.inner.reduce((acc, err) => {
      if (err.path) {
        acc[err.path] = err.message;
      }
      return acc;
    }, {} as Record<string, string>);
    
    console.log(errorsByPath);
    // { email: 'Email is required', age: 'Age must be a number' }
  }
}

// Validate sync (throws immediately)
try {
  const validated = schema.validateSync(data);
} catch (error) {
  console.error('Sync validation error:', error);
}

// Validate at path
try {
  await schema.validateAt('email', data);
} catch (error) {
  console.error('Email validation error:', error);
}
```

## Integration with React Hook Form

```typescript
import { useForm } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';

const schema = yup.object({
  firstName: yup.string().required('First name is required'),
  lastName: yup.string().required('Last name is required'),
  email: yup.string().email('Invalid email').required('Email is required'),
  age: yup.number()
    .typeError('Age must be a number')
    .required('Age is required')
    .min(18, 'Must be at least 18'),
  agreeToTerms: yup.boolean()
    .oneOf([true], 'You must accept the terms and conditions'),
});

type FormData = yup.InferType<typeof schema>;

function RegistrationForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({
    resolver: yupResolver(schema),
    mode: 'onBlur', // Validate on blur
  });

  const onSubmit = async (data: FormData) => {
    console.log('Valid data:', data);
    // Submit to API
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('firstName')} placeholder="First Name" />
        {errors.firstName && <span>{errors.firstName.message}</span>}
      </div>

      <div>
        <input {...register('lastName')} placeholder="Last Name" />
        {errors.lastName && <span>{errors.lastName.message}</span>}
      </div>

      <div>
        <input {...register('email')} type="email" placeholder="Email" />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <div>
        <input {...register('age')} type="number" placeholder="Age" />
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
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

## Integration with Formik

```typescript
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as yup from 'yup';

const validationSchema = yup.object({
  username: yup.string()
    .min(3, 'At least 3 characters')
    .max(20, 'Maximum 20 characters')
    .required('Username is required'),
  email: yup.string()
    .email('Invalid email address')
    .required('Email is required'),
  password: yup.string()
    .min(8, 'At least 8 characters')
    .matches(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Must contain uppercase, lowercase, and number'
    )
    .required('Password is required'),
  confirmPassword: yup.string()
    .oneOf([yup.ref('password')], 'Passwords must match')
    .required('Confirm your password'),
});

function SignupForm() {
  return (
    <Formik
      initialValues={{
        username: '',
        email: '',
        password: '',
        confirmPassword: '',
      }}
      validationSchema={validationSchema}
      onSubmit={async (values, { setSubmitting }) => {
        try {
          await submitForm(values);
          console.log('Form submitted:', values);
        } catch (error) {
          console.error('Submission error:', error);
        } finally {
          setSubmitting(false);
        }
      }}
      validateOnChange={true}
      validateOnBlur={true}
    >
      {({ isSubmitting, isValid, dirty }) => (
        <Form>
          <div>
            <Field name="username" placeholder="Username" />
            <ErrorMessage name="username" component="div" />
          </div>

          <div>
            <Field name="email" type="email" placeholder="Email" />
            <ErrorMessage name="email" component="div" />
          </div>

          <div>
            <Field name="password" type="password" placeholder="Password" />
            <ErrorMessage name="password" component="div" />
          </div>

          <div>
            <Field 
              name="confirmPassword" 
              type="password" 
              placeholder="Confirm Password" 
            />
            <ErrorMessage name="confirmPassword" component="div" />
          </div>

          <button 
            type="submit" 
            disabled={isSubmitting || !isValid || !dirty}
          >
            {isSubmitting ? 'Creating Account...' : 'Sign Up'}
          </button>
        </Form>
      )}
    </Formik>
  );
}
```

## TypeScript Integration

```typescript
import * as yup from 'yup';
import { InferType } from 'yup';

// Define schema
const userSchema = yup.object({
  id: yup.number().required(),
  name: yup.string().required(),
  email: yup.string().email().required(),
  role: yup.string().oneOf(['admin', 'user', 'guest']).required(),
  profile: yup.object({
    bio: yup.string().nullable(),
    avatar: yup.string().url().nullable(),
  }).required(),
  settings: yup.object({
    notifications: yup.boolean().default(true),
    theme: yup.string().oneOf(['light', 'dark']).default('light'),
  }).required(),
});

// Infer TypeScript type from schema
type User = InferType<typeof userSchema>;

// Use inferred type
const user: User = {
  id: 1,
  name: 'John Doe',
  email: 'john@example.com',
  role: 'admin',
  profile: {
    bio: 'Software developer',
    avatar: 'https://example.com/avatar.jpg',
  },
  settings: {
    notifications: true,
    theme: 'dark',
  },
};

// Generic validation function
async function validateData<T>(
  schema: yup.Schema<T>,
  data: unknown
): Promise<T> {
  return await schema.validate(data, { stripUnknown: true });
}

// Usage
const validatedUser = await validateData(userSchema, rawData);
```

## Schema Composition and Reusability

```typescript
import * as yup from 'yup';

// Base schemas
const emailSchema = yup.string().email().required();
const passwordSchema = yup.string().min(8).required();
const phoneSchema = yup.string().matches(/^\d{10}$/).nullable();

// Address schema (reusable)
const addressSchema = yup.object({
  street: yup.string().required(),
  city: yup.string().required(),
  state: yup.string().length(2).required(),
  zip: yup.string().matches(/^\d{5}(-\d{4})?$/).required(),
  country: yup.string().default('US'),
});

// Compose schemas
const userProfileSchema = yup.object({
  email: emailSchema,
  password: passwordSchema,
  phone: phoneSchema,
  shippingAddress: addressSchema,
  billingAddress: addressSchema,
});

// Extend schemas
const adminProfileSchema = userProfileSchema.shape({
  role: yup.string().oneOf(['admin', 'superadmin']).required(),
  permissions: yup.array().of(yup.string()).min(1).required(),
});

// Partial schemas
const updateProfileSchema = userProfileSchema.partial();

// Pick specific fields
const loginSchema = userProfileSchema.pick(['email', 'password']);

// Omit fields
const publicProfileSchema = userProfileSchema.omit(['password']);

// Lazy schemas (for recursive structures)
const categorySchema: yup.Schema = yup.lazy(() =>
  yup.object({
    id: yup.number().required(),
    name: yup.string().required(),
    parent: categorySchema.nullable(),
    children: yup.array().of(categorySchema),
  })
);
```

## Common Mistakes

1. **Not handling async validation properly**: Forgetting to await validation results or not handling async test methods correctly.

2. **Confusing nullable vs optional**: Using `.nullable()` when you mean `.optional()` or vice versa.

3. **Incorrect oneOf usage**: Not understanding that `oneOf([true])` is different from `.required()` for booleans.

4. **Validation order matters**: Chain methods in logical order (type coercion before validation).

5. **Not using abortEarly: false**: Missing multiple validation errors by stopping at the first one.

6. **Memory leaks with async validation**: Not cancelling pending validations when component unmounts.

7. **Overusing custom tests**: Creating custom tests when built-in methods would suffice.

8. **Poor error messages**: Using generic error messages instead of user-friendly, specific ones.

9. **Not leveraging transforms**: Manually cleaning data instead of using transform methods.

10. **Schema in render**: Creating schemas inside component render causing unnecessary recreations.

## Best Practices

1. **Define schemas outside components**: Create schemas as constants to avoid recreation on each render.

2. **Use TypeScript inference**: Leverage `InferType` for type safety without duplicate type definitions.

3. **Compose reusable schemas**: Build complex schemas from smaller, reusable pieces.

4. **Provide clear error messages**: Always include user-friendly error messages with context.

5. **Use transforms for data normalization**: Clean and normalize data using transforms before validation.

6. **Leverage conditional validation**: Use `when()` for complex conditional validation logic.

7. **Implement proper async validation**: Debounce async validations and handle cancellation.

8. **Strip unknown fields in production**: Use `stripUnknown: true` for API data validation.

9. **Test your schemas**: Write unit tests for complex validation schemas.

10. **Document complex validations**: Add comments explaining business logic in validation rules.

## When to Use Yup

**Use Yup when:**
- Building forms with complex validation requirements
- Need schema-based validation with TypeScript support
- Using Formik or React Hook Form
- Want declarative validation API
- Need cross-field validation
- Building reusable validation logic
- Want good error messages out of the box

**Consider alternatives when:**
- Need runtime type validation only (use Zod)
- Building backend-only validation (use Joi)
- Need minimal bundle size (use Valibot)
- Want functional validation framework (use Vest)
- Need JSON Schema validation (use AJV)

## Interview Questions

### Q1: What's the difference between `.nullable()` and `.optional()` in Yup?

**Answer**: `.optional()` means the field can be undefined (not present in the object), while `.nullable()` means the field can explicitly be null. A field can be both nullable and optional, allowing it to be undefined, null, or have a value.

```typescript
// Optional: can be undefined
yup.object({ field: yup.string() }) // field can be undefined

// Nullable: can be null
yup.object({ field: yup.string().nullable() }) // field can be null

// Both: can be undefined or null
yup.object({ field: yup.string().nullable().optional() })
```

### Q2: How do you implement conditional validation in Yup?

**Answer**: Use the `.when()` method to create conditional validation based on other field values:

```typescript
const schema = yup.object({
  type: yup.string().oneOf(['personal', 'business']),
  companyName: yup.string().when('type', {
    is: 'business',
    then: (schema) => schema.required('Company name required'),
    otherwise: (schema) => schema.notRequired(),
  }),
});
```

### Q3: How do you validate cross-field dependencies in Yup?

**Answer**: Use `.ref()` to reference other fields, `.when()` for conditional validation, or custom `.test()` methods for complex logic:

```typescript
const schema = yup.object({
  password: yup.string().required(),
  confirmPassword: yup.string()
    .required()
    .oneOf([yup.ref('password')], 'Passwords must match'),
  endDate: yup.date()
    .min(yup.ref('startDate'), 'End must be after start'),
});
```

### Q4: How do you handle async validation in Yup?

**Answer**: Use `.test()` with an async function:

```typescript
const schema = yup.string().test(
  'unique-email',
  'Email already exists',
  async (value) => {
    if (!value) return true;
    const response = await fetch(`/api/check-email?email=${value}`);
    const { available } = await response.json();
    return available;
  }
);
```

### Q5: What's the difference between `validate()` and `validateSync()` in Yup?

**Answer**: `validate()` is asynchronous and returns a Promise, supporting async validations. `validateSync()` is synchronous and throws immediately, but cannot handle async test methods. Use `validate()` for forms with async validation, `validateSync()` for simple, synchronous validation needs.

### Q6: How do you create custom validation methods in Yup?

**Answer**: Use `yup.addMethod()` to extend schema types:

```typescript
yup.addMethod(yup.string, 'phone', function(message = 'Invalid phone') {
  return this.test('phone', message, function(value) {
    if (!value) return true;
    return /^\d{10}$/.test(value);
  });
});

// TypeScript declaration
declare module 'yup' {
  interface StringSchema {
    phone(message?: string): this;
  }
}
```

### Q7: How do you optimize Yup validation performance?

**Answer**: Strategies include: 1) Define schemas outside components to avoid recreation, 2) Use `validateSync()` for synchronous validation, 3) Implement debouncing for async validations, 4) Use `abortEarly: true` when you only need the first error, 5) Consider lazy schemas for recursive structures, 6) Cache validation results when appropriate.

## Key Takeaways

1. Yup provides a declarative API for defining validation schemas with excellent TypeScript support.

2. Use `.nullable()` and `.optional()` appropriately to handle null and undefined values correctly.

3. The `.when()` method enables conditional validation based on other field values.

4. Cross-field validation is achieved using `.ref()` for simple cases and `.test()` for complex logic.

5. Transforms preprocess data before validation using the `.transform()` method.

6. Async validation is supported through `.test()` with async functions, but requires proper error handling.

7. Yup integrates seamlessly with React Hook Form via `@hookform/resolvers/yup` and with Formik natively.

8. Custom validation methods can be added using `yup.addMethod()` for reusable validation logic.

9. TypeScript types can be inferred from schemas using `InferType<typeof schema>` for type safety.

10. Error handling provides detailed validation errors through `ValidationError` with `errors` and `inner` arrays, and supports `abortEarly: false` to collect all errors at once.

## Resources

- [Official Documentation](https://github.com/jquense/yup)
- [Yup API Reference](https://github.com/jquense/yup#api)
- [React Hook Form with Yup](https://react-hook-form.com/get-started#SchemaValidation)
- [Formik Documentation](https://formik.org/docs/guides/validation)
- [TypeScript with Yup](https://github.com/jquense/yup#typescript-integration)
- [Yup vs Zod Comparison](https://github.com/colinhacks/zod#comparison)
