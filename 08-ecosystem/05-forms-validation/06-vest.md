# Vest Validation Framework

## Overview

Vest is a declarative validation framework inspired by unit testing libraries like Mocha and Jest. It provides a unique approach to validation by treating validation rules as tests, making it intuitive for developers familiar with testing frameworks. Vest supports both synchronous and asynchronous validation, field-level validation, and integrates well with any form library or framework.

## Installation and Setup

```bash
# Install Vest
npm install vest

# With TypeScript (types included)
npm install vest

# Common pairings
npm install vest react react-hook-form
```

## Core Concepts

### Basic Suite and Tests

```javascript
import { create, test, enforce } from 'vest';

// Create validation suite
const validateUser = create('user_validation', (data = {}) => {
  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  test('username', 'Username must be at least 3 characters', () => {
    enforce(data.username).longerThanOrEquals(3);
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  test('email', 'Email must be valid', () => {
    enforce(data.email).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
  });

  test('age', 'Age must be at least 18', () => {
    enforce(data.age).greaterThanOrEquals(18);
  });
});

// Run validation
const result = validateUser({
  username: 'john',
  email: 'john@example.com',
  age: 25,
});

// Check validation results
if (result.isValid()) {
  console.log('All fields valid');
} else {
  console.log('Validation errors:', result.getErrors());
}

// Check specific field
if (result.hasErrors('username')) {
  console.log('Username errors:', result.getErrors('username'));
}
```

### Enforce Assertions

```javascript
import { enforce } from 'vest';

// String validations
enforce('hello').isNotEmpty();
enforce('test').equals('test');
enforce('hello').notEquals('world');
enforce('hello').longerThan(3);
enforce('hello').longerThanOrEquals(5);
enforce('hi').shorterThan(5);
enforce('hi').shorterThanOrEquals(2);
enforce('test@example.com').matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
enforce('abc').isString();

// Number validations
enforce(10).isNumber();
enforce(10).greaterThan(5);
enforce(10).greaterThanOrEquals(10);
enforce(5).lessThan(10);
enforce(5).lessThanOrEquals(5);
enforce(10).isBetween(5, 15);
enforce(5).isNegative();
enforce(5).isPositive();
enforce(4).isEven();
enforce(5).isOdd();

// Array validations
enforce([1, 2, 3]).isArray();
enforce([1, 2, 3]).isEmpty(); // empty array
enforce([1, 2, 3]).isNotEmpty();
enforce([1, 2, 3]).lengthEquals(3);
enforce([1, 2, 3]).lengthNotEquals(5);
enforce([1, 2, 3]).contains(2);
enforce([1, 2, 3]).doesNotContain(5);

// Boolean validations
enforce(true).isTruthy();
enforce(false).isFalsy();
enforce(true).isBoolean();

// Object validations
enforce({}).isObject();
enforce({ a: 1 }).isEmpty(); // empty object
enforce({ a: 1 }).isNotEmpty();

// Type validations
enforce(null).isNull();
enforce(undefined).isUndefined();
enforce(null).isNullish(); // null or undefined
enforce('test').isNotNull();
enforce('test').isNotUndefined();

// Chaining
enforce('hello').isString().longerThan(3).matches(/^h/);
```

### Suite Organization

```javascript
import { create, test, enforce, group } from 'vest';

const validateRegistration = create('registration', (data = {}, currentField) => {
  // Always run these tests
  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  // Group related tests
  group('credentials', () => {
    test('email', 'Email is required', () => {
      enforce(data.email).isNotEmpty();
    });

    test('email', 'Email must be valid', () => {
      enforce(data.email).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
    });

    test('password', 'Password is required', () => {
      enforce(data.password).isNotEmpty();
    });

    test('password', 'Password must be at least 8 characters', () => {
      enforce(data.password).longerThanOrEquals(8);
    });
  });

  // Group profile fields
  group('profile', () => {
    test('firstName', 'First name is required', () => {
      enforce(data.firstName).isNotEmpty();
    });

    test('lastName', 'Last name is required', () => {
      enforce(data.lastName).isNotEmpty();
    });

    test('age', 'Must be at least 18', () => {
      enforce(data.age).greaterThanOrEquals(18);
    });
  });

  // Optional: only validate current field for better performance
  if (currentField) {
    return; // Skip tests for other fields
  }
});

// Run full validation
const fullResult = validateRegistration(formData);

// Run validation for single field
const fieldResult = validateRegistration(formData, 'email');
```

## Advanced Validation Patterns

### Conditional Validation (skipWhen/omitWhen)

```javascript
import { create, test, enforce, skipWhen, omitWhen } from 'vest';

const validateCheckout = create('checkout', (data = {}) => {
  test('paymentMethod', 'Payment method is required', () => {
    enforce(data.paymentMethod).isNotEmpty();
  });

  // Skip remaining tests if payment method is invalid
  skipWhen(
    result => result.hasErrors('paymentMethod'),
    () => {
      // Credit card validation
      skipWhen(data.paymentMethod !== 'credit_card', () => {
        test('cardNumber', 'Card number is required', () => {
          enforce(data.cardNumber).isNotEmpty();
        });

        test('cardNumber', 'Invalid card number', () => {
          enforce(data.cardNumber).matches(/^\d{16}$/);
        });

        test('cvv', 'CVV is required', () => {
          enforce(data.cvv).isNotEmpty();
        });
      });

      // PayPal validation
      skipWhen(data.paymentMethod !== 'paypal', () => {
        test('paypalEmail', 'PayPal email is required', () => {
          enforce(data.paypalEmail).isNotEmpty();
        });

        test('paypalEmail', 'Invalid email', () => {
          enforce(data.paypalEmail).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
        });
      });
    }
  );

  // Shipping validation
  test('shippingAddress', 'Shipping address is required', () => {
    enforce(data.shippingAddress).isNotEmpty();
  });

  // Only validate billing if different from shipping
  omitWhen(data.sameAsShipping, () => {
    test('billingAddress', 'Billing address is required', () => {
      enforce(data.billingAddress).isNotEmpty();
    });
  });
});
```

### Async Validation

```javascript
import { create, test, enforce } from 'vest';

const validateUserAsync = create('user_async', (data = {}) => {
  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  // Async test
  test('username', 'Username already taken', async () => {
    // Simulate API call
    const response = await fetch(`/api/check-username?username=${data.username}`);
    const { available } = await response.json();
    
    enforce(available).isTruthy();
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  // Another async test
  test('email', 'Email already registered', async () => {
    const response = await fetch(`/api/check-email?email=${data.email}`);
    const { available } = await response.json();
    
    return enforce(available).equals(true);
  });
});

// Handle async validation
const result = validateUserAsync(formData);

// Check if validation is pending
if (result.isPending()) {
  console.log('Validation in progress...');
  console.log('Pending fields:', result.getPending());
}

// Wait for async tests to complete
result.done((finalResult) => {
  if (finalResult.isValid()) {
    console.log('All validations passed');
  } else {
    console.log('Errors:', finalResult.getErrors());
  }
});

// Or use Promise
result.done().then((finalResult) => {
  console.log('Validation complete:', finalResult.isValid());
});
```

### Custom Enforce Rules

```javascript
import { enforce, create, test } from 'vest';

// Add custom enforcement rule
enforce.extend({
  isStrongPassword: (value) => {
    const hasUpperCase = /[A-Z]/.test(value);
    const hasLowerCase = /[a-z]/.test(value);
    const hasNumber = /[0-9]/.test(value);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(value);
    
    return {
      pass: hasUpperCase && hasLowerCase && hasNumber && hasSpecialChar,
      message: () => 'Password must contain uppercase, lowercase, number, and special character',
    };
  },
});

// Add custom phone validation
enforce.extend({
  isPhoneNumber: (value) => {
    const phoneRegex = /^\+?1?\s*\(?([0-9]{3})\)?[-.\s]?([0-9]{3})[-.\s]?([0-9]{4})$/;
    
    return {
      pass: phoneRegex.test(value),
      message: () => 'Must be a valid phone number',
    };
  },
});

// Add URL validation
enforce.extend({
  isValidUrl: (value) => {
    try {
      new URL(value);
      return { pass: true };
    } catch {
      return {
        pass: false,
        message: () => 'Must be a valid URL',
      };
    }
  },
});

// Use custom rules
const validateForm = create('form', (data = {}) => {
  test('password', 'Password must be strong', () => {
    enforce(data.password).isStrongPassword();
  });

  test('phone', 'Invalid phone number', () => {
    enforce(data.phone).isPhoneNumber();
  });

  test('website', 'Invalid URL', () => {
    enforce(data.website).isValidUrl();
  });
});

// Composable custom rules
enforce.extend({
  isValidCreditCard: (value) => {
    const cleaned = value.replace(/[\s-]/g, '');
    
    if (!/^\d{13,19}$/.test(cleaned)) {
      return { pass: false, message: () => 'Invalid card number length' };
    }
    
    // Luhn algorithm
    let sum = 0;
    let isEven = false;
    
    for (let i = cleaned.length - 1; i >= 0; i--) {
      let digit = parseInt(cleaned[i], 10);
      if (isEven) {
        digit *= 2;
        if (digit > 9) digit -= 9;
      }
      sum += digit;
      isEven = !isEven;
    }
    
    return {
      pass: sum % 10 === 0,
      message: () => 'Invalid credit card number',
    };
  },
});
```

### Field-Level Validation

```javascript
import { create, test, enforce, only } from 'vest';

const validateForm = create('form', (data = {}, changedField) => {
  // Use 'only' to validate specific field(s)
  only(changedField);

  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  test('username', 'Username too short', () => {
    enforce(data.username).longerThanOrEquals(3);
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  test('email', 'Invalid email format', () => {
    enforce(data.email).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
  });

  test('password', 'Password is required', () => {
    enforce(data.password).isNotEmpty();
  });

  test('password', 'Password too short', () => {
    enforce(data.password).longerThanOrEquals(8);
  });
});

// Validate single field
const usernameResult = validateForm(formData, 'username');

// Validate multiple specific fields
const credentialsResult = validateForm(formData, ['email', 'password']);

// Validate all fields (omit second argument)
const fullResult = validateForm(formData);
```

## React Integration

### Basic React Hook Form Integration

```javascript
import { useForm } from 'react-hook-form';
import { create, test, enforce } from 'vest';
import { vestResolver } from '@hookform/resolvers/vest';

// Create validation suite
const validationSuite = create('registration', (data = {}) => {
  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  test('username', 'Username must be at least 3 characters', () => {
    enforce(data.username).longerThanOrEquals(3);
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  test('email', 'Invalid email format', () => {
    enforce(data.email).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
  });

  test('password', 'Password is required', () => {
    enforce(data.password).isNotEmpty();
  });

  test('password', 'Password must be at least 8 characters', () => {
    enforce(data.password).longerThanOrEquals(8);
  });
});

function RegistrationForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({
    resolver: vestResolver(validationSuite),
    mode: 'onBlur',
  });

  const onSubmit = async (data) => {
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
        <input {...register('password')} type="password" placeholder="Password" />
        {errors.password && <span>{errors.password.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Register'}
      </button>
    </form>
  );
}
```

### Custom React Hook

```javascript
import { useState, useCallback } from 'react';
import { create, test, enforce } from 'vest';

// Create validation suite
const validationSuite = create('form', (data = {}, field) => {
  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  test('email', 'Invalid email', () => {
    enforce(data.email).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
  });
});

function useVestForm(suite, initialValues = {}) {
  const [values, setValues] = useState(initialValues);
  const [result, setResult] = useState(suite.get());
  const [touched, setTouched] = useState({});

  const handleChange = useCallback((fieldName, value) => {
    setValues(prev => ({ ...prev, [fieldName]: value }));
    
    // Validate field
    const newResult = suite({ ...values, [fieldName]: value }, fieldName);
    setResult(newResult);
  }, [suite, values]);

  const handleBlur = useCallback((fieldName) => {
    setTouched(prev => ({ ...prev, [fieldName]: true }));
    
    // Re-validate field on blur
    const newResult = suite(values, fieldName);
    setResult(newResult);
  }, [suite, values]);

  const handleSubmit = useCallback((onSubmit) => {
    return (e) => {
      e.preventDefault();
      
      // Validate all fields
      const finalResult = suite(values);
      setResult(finalResult);

      if (finalResult.isValid()) {
        onSubmit(values);
      }
    };
  }, [suite, values]);

  const getFieldError = useCallback((fieldName) => {
    if (!touched[fieldName]) return null;
    return result.getErrors(fieldName)?.[0];
  }, [result, touched]);

  return {
    values,
    errors: result.getErrors(),
    touched,
    isValid: result.isValid(),
    handleChange,
    handleBlur,
    handleSubmit,
    getFieldError,
  };
}

// Usage
function MyForm() {
  const {
    values,
    handleChange,
    handleBlur,
    handleSubmit,
    getFieldError,
    isValid,
  } = useVestForm(validationSuite, {
    username: '',
    email: '',
  });

  const onSubmit = (data) => {
    console.log('Submit:', data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input
          value={values.username}
          onChange={(e) => handleChange('username', e.target.value)}
          onBlur={() => handleBlur('username')}
          placeholder="Username"
        />
        {getFieldError('username') && (
          <span>{getFieldError('username')}</span>
        )}
      </div>

      <div>
        <input
          value={values.email}
          onChange={(e) => handleChange('email', e.target.value)}
          onBlur={() => handleBlur('email')}
          placeholder="Email"
        />
        {getFieldError('email') && (
          <span>{getFieldError('email')}</span>
        )}
      </div>

      <button type="submit" disabled={!isValid}>
        Submit
      </button>
    </form>
  );
}
```

## Warning-Level Validations

```javascript
import { create, test, enforce, warn } from 'vest';

const validateForm = create('form', (data = {}) => {
  // Error-level validation
  test('password', 'Password is required', () => {
    enforce(data.password).isNotEmpty();
  });

  // Warning-level validation (doesn't prevent submission)
  warn('password', 'Password should be longer', () => {
    enforce(data.password).longerThanOrEquals(12);
  });

  warn('password', 'Password should contain special characters', () => {
    enforce(data.password).matches(/[!@#$%^&*(),.?":{}|<>]/);
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  warn('email', 'Consider using a business email', () => {
    enforce(data.email).matches(/@company\.com$/);
  });
});

const result = validateForm(formData);

// Check for errors (blocks submission)
if (result.hasErrors()) {
  console.log('Errors:', result.getErrors());
}

// Check for warnings (doesn't block submission)
if (result.hasWarnings()) {
  console.log('Warnings:', result.getWarnings());
}

// Can submit if no errors, even with warnings
if (!result.hasErrors()) {
  console.log('Form can be submitted');
}
```

## TypeScript Integration

```typescript
import { create, test, enforce, SuiteResult } from 'vest';

// Define form data type
interface UserFormData {
  username: string;
  email: string;
  age: number;
  agreeToTerms: boolean;
}

// Create typed validation suite
const validateUser = create<UserFormData>('user', (data) => {
  test('username', 'Username is required', () => {
    enforce(data.username).isNotEmpty();
  });

  test('username', 'Username must be at least 3 characters', () => {
    enforce(data.username).longerThanOrEquals(3);
  });

  test('email', 'Email is required', () => {
    enforce(data.email).isNotEmpty();
  });

  test('email', 'Invalid email format', () => {
    enforce(data.email).matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/);
  });

  test('age', 'Must be at least 18', () => {
    enforce(data.age).greaterThanOrEquals(18);
  });

  test('agreeToTerms', 'Must accept terms', () => {
    enforce(data.agreeToTerms).isTruthy();
  });
});

// Type-safe usage
const formData: UserFormData = {
  username: 'john_doe',
  email: 'john@example.com',
  age: 25,
  agreeToTerms: true,
};

const result: SuiteResult<string> = validateUser(formData);

if (result.isValid()) {
  console.log('Valid user data');
}
```

## Common Mistakes

1. **Not using `only()` for field-level validation**: Validating all fields on every change instead of single field.

2. **Forgetting async validation handling**: Not using `.done()` or checking `.isPending()` for async tests.

3. **Overusing `skipWhen`**: Creating overly complex conditional logic instead of simpler test organization.

4. **Not managing touched state**: Showing validation errors before user interaction.

5. **Creating suite inside component**: Re-creating validation suite on every render instead of outside component.

6. **Mixing errors and warnings incorrectly**: Using `test()` when `warn()` would be more appropriate.

7. **Not leveraging groups**: Organizing all tests flat instead of using groups for better structure.

8. **Ignoring validation result caching**: Running validation unnecessarily when values haven't changed.

9. **Poor custom enforce rules**: Creating complex validation logic instead of simple, composable rules.

10. **Not handling pending state**: Allowing form submission while async validation is pending.

## Best Practices

1. **Create suites outside components**: Define validation suites at module level for reusability and performance.

2. **Use field-level validation**: Leverage `only()` to validate single fields on change for better UX.

3. **Organize with groups**: Use groups to structure related validation tests logically.

4. **Handle async properly**: Always check `isPending()` and use `.done()` for async validation.

5. **Leverage warnings**: Use `warn()` for suggestions that shouldn't block submission.

6. **Extend enforce wisely**: Create reusable custom enforcement rules for domain-specific validation.

7. **Manage touched state**: Only show errors for fields the user has interacted with.

8. **Use conditional validation**: Apply `skipWhen` and `omitWhen` for complex conditional logic.

9. **Type your suites**: Use TypeScript generics for type-safe validation suites.

10. **Test your validation**: Write unit tests for complex validation logic.

## When to Use Vest

**Use Vest when:**
- Want testing-like syntax for validation
- Need flexible field-level validation
- Building custom form solutions
- Want framework-agnostic validation
- Need granular control over validation flow
- Want clear separation of validation logic
- Need both sync and async validation

**Consider alternatives when:**
- Need schema-based validation (use Zod or Yup)
- Building simple forms with React Hook Form (use Yup)
- Need JSON Schema compliance (use AJV)
- Want backend validation (use Joi)
- Need minimal bundle size (use Valibot)

## Interview Questions

### Q1: How does Vest differ from schema-based validators like Yup or Zod?

**Answer**: Vest uses a test-driven approach where validation rules are written as test functions, similar to unit testing frameworks. Unlike schema-based validators that define structure declaratively, Vest provides imperative validation flow with better control over test execution order, conditional validation, and field-level validation. This makes Vest more flexible for complex validation scenarios but requires more code for simple cases.

### Q2: How do you implement conditional validation in Vest?

**Answer**: Use `skipWhen` to skip tests conditionally or `omitWhen` to completely omit tests:

```javascript
const suite = create((data) => {
  skipWhen(data.type !== 'business', () => {
    test('companyName', 'Required for business', () => {
      enforce(data.companyName).isNotEmpty();
    });
  });
  
  omitWhen(data.sameAddress, () => {
    test('billingAddress', 'Required', () => {
      enforce(data.billingAddress).isNotEmpty();
    });
  });
});
```

### Q3: How do you handle asynchronous validation in Vest?

**Answer**: Write async test functions and handle results using `.done()`:

```javascript
const suite = create((data) => {
  test('username', 'Already taken', async () => {
    const response = await checkUsername(data.username);
    enforce(response.available).isTruthy();
  });
});

const result = suite(data);

if (result.isPending()) {
  console.log('Validating...');
}

result.done((finalResult) => {
  if (finalResult.isValid()) {
    console.log('Valid!');
  }
});
```

### Q4: What's the difference between `test()` and `warn()` in Vest?

**Answer**: `test()` creates error-level validations that prevent form submission when they fail. `warn()` creates warning-level validations that provide user feedback but don't block submission. Use `warn()` for suggestions and best practices, `test()` for requirements. Check with `hasErrors()` for submission blocking and `hasWarnings()` for informational feedback.

### Q5: How do you implement field-level validation for better performance?

**Answer**: Use the `only()` function to validate specific fields:

```javascript
const suite = create((data, changedField) => {
  only(changedField); // Only run tests for this field
  
  test('username', 'Required', () => {
    enforce(data.username).isNotEmpty();
  });
  
  test('email', 'Required', () => {
    enforce(data.email).isNotEmpty();
  });
});

// Validate single field on change
const result = suite(formData, 'email');
```

### Q6: How do you create custom enforcement rules in Vest?

**Answer**: Use `enforce.extend()` to add custom rules:

```javascript
enforce.extend({
  isStrongPassword: (value) => {
    const isStrong = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&]).{8,}$/.test(value);
    return {
      pass: isStrong,
      message: () => 'Password must contain uppercase, lowercase, number, and special char',
    };
  },
});

// Use custom rule
test('password', () => {
  enforce(data.password).isStrongPassword();
});
```

### Q7: How do you organize complex validation suites in Vest?

**Answer**: Use groups to organize related tests and improve maintainability:

```javascript
const suite = create((data) => {
  group('credentials', () => {
    test('email', 'Required', () => enforce(data.email).isNotEmpty());
    test('password', 'Required', () => enforce(data.password).isNotEmpty());
  });
  
  group('profile', () => {
    test('firstName', 'Required', () => enforce(data.firstName).isNotEmpty());
    test('lastName', 'Required', () => enforce(data.lastName).isNotEmpty());
  });
});
```

## Key Takeaways

1. Vest uses a testing-framework-inspired syntax where validation rules are written as test functions with descriptive names.

2. Field-level validation is achieved using `only()` to run tests for specific fields, improving performance and UX.

3. The `enforce` API provides chainable assertions for common validation scenarios with clear, readable syntax.

4. Conditional validation uses `skipWhen()` to skip test blocks or `omitWhen()` to completely omit tests based on conditions.

5. Async validation is fully supported with `.done()` callbacks and `.isPending()` status checks.

6. Warnings provide user feedback without blocking submission using the `warn()` function instead of `test()`.

7. Groups organize related tests logically and can be used to structure complex validation suites.

8. Custom enforcement rules extend Vest's validation capabilities through `enforce.extend()`.

9. Vest is framework-agnostic and integrates with React, Vue, Angular, or any form library.

10. The declarative test-based approach provides fine-grained control over validation flow and error messages.

## Resources

- [Official Documentation](https://vestjs.dev/)
- [GitHub Repository](https://github.com/ealush/vest)
- [Vest Examples](https://vestjs.dev/docs/examples)
- [Vest with React](https://vestjs.dev/docs/integrations/react)
- [Custom Enforcement Rules](https://vestjs.dev/docs/enforce/custom_rules)
- [Async Validation Guide](https://vestjs.dev/docs/writing_tests/async_tests)
