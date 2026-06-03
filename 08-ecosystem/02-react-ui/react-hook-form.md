# React Hook Form - Performant Form Validation Library

## Table of Contents
- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
- [Form Registration](#form-registration)
- [Validation](#validation)
- [Advanced Features](#advanced-features)
- [Integration](#integration)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

React Hook Form is a performant, flexible form library for React that minimizes re-renders and provides easy validation. It uses uncontrolled components and refs to reduce unnecessary renders while maintaining full form control.

**Key Features:**
- Minimal re-renders
- Easy to integrate
- TypeScript support
- Schema validation (Zod, Yup, Joi)
- Tiny size (9kb)
- No dependencies

## Installation & Setup

```bash
# Install React Hook Form
npm install react-hook-form

# With Zod for validation
npm install react-hook-form zod @hookform/resolvers

# With Yup
npm install react-hook-form yup @hookform/resolvers
```

### Basic Setup

```jsx
import { useForm } from 'react-hook-form';

function App() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  const onSubmit = (data) => {
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('firstName')} />
      <input {...register('email')} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Core Concepts

### 1. useForm Hook

The main hook for form management:

```jsx
import { useForm } from 'react-hook-form';

function FormExample() {
  const {
    register,           // Register inputs
    handleSubmit,       // Handle form submission
    formState,          // Form state (errors, isDirty, etc.)
    watch,              // Watch input values
    reset,              // Reset form
    setValue,           // Set input value
    getValues,          // Get current values
    trigger,            // Trigger validation
    control,            // For controlled components
  } = useForm({
    defaultValues: {
      firstName: '',
      lastName: '',
      email: '',
    },
    mode: 'onSubmit',   // Validation mode
  });

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('firstName')} placeholder="First Name" />
      <input {...register('lastName')} placeholder="Last Name" />
      <input {...register('email')} placeholder="Email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Register Function

Register inputs with validation rules:

```jsx
function RegisterExample() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input
        {...register('username', {
          required: 'Username is required',
          minLength: {
            value: 3,
            message: 'Username must be at least 3 characters',
          },
          pattern: {
            value: /^[a-zA-Z0-9_]+$/,
            message: 'Username can only contain letters, numbers and underscores',
          },
        })}
      />
      {errors.username && <span>{errors.username.message}</span>}

      <input
        type="email"
        {...register('email', {
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email address',
          },
        })}
      />
      {errors.email && <span>{errors.email.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 3. Form State

Access form state information:

```jsx
function FormStateExample() {
  const { register, handleSubmit, formState } = useForm();

  const {
    errors,          // Validation errors
    isDirty,         // Form has been modified
    isValid,         // Form is valid
    isSubmitting,    // Form is submitting
    isSubmitted,     // Form has been submitted
    submitCount,     // Number of submissions
    touchedFields,   // Fields that have been touched
    dirtyFields,     // Fields that have been modified
  } = formState;

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('name', { required: true })} />
      
      <div>
        <p>Is Dirty: {isDirty ? 'Yes' : 'No'}</p>
        <p>Is Valid: {isValid ? 'Yes' : 'No'}</p>
        <p>Is Submitting: {isSubmitting ? 'Yes' : 'No'}</p>
        <p>Submit Count: {submitCount}</p>
      </div>

      <button type="submit" disabled={isSubmitting || !isValid}>
        Submit
      </button>
    </form>
  );
}
```

## Form Registration

### 1. Input Types

Register different input types:

```jsx
function InputTypesExample() {
  const { register, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {/* Text input */}
      <input {...register('text')} type="text" />

      {/* Number input */}
      <input
        {...register('age', { valueAsNumber: true })}
        type="number"
      />

      {/* Date input */}
      <input
        {...register('birthdate', { valueAsDate: true })}
        type="date"
      />

      {/* Checkbox */}
      <input
        {...register('agree')}
        type="checkbox"
      />

      {/* Radio buttons */}
      <input {...register('gender')} type="radio" value="male" />
      <input {...register('gender')} type="radio" value="female" />

      {/* Select */}
      <select {...register('country')}>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
      </select>

      {/* Textarea */}
      <textarea {...register('comments')} />

      {/* File input */}
      <input
        {...register('file')}
        type="file"
      />

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Watch Values

Monitor input values:

```jsx
function WatchExample() {
  const { register, watch } = useForm({
    defaultValues: {
      firstName: '',
      lastName: '',
    },
  });

  // Watch all fields
  const watchAllFields = watch();

  // Watch specific field
  const firstName = watch('firstName');

  // Watch multiple fields
  const [firstNameWatch, lastNameWatch] = watch(['firstName', 'lastName']);

  // Watch with callback
  useEffect(() => {
    const subscription = watch((value, { name, type }) => {
      console.log('Field changed:', name, value);
    });
    return () => subscription.unsubscribe();
  }, [watch]);

  return (
    <form>
      <input {...register('firstName')} />
      <input {...register('lastName')} />
      
      <div>
        <p>First Name: {firstName}</p>
        <p>Full Form: {JSON.stringify(watchAllFields)}</p>
      </div>
    </form>
  );
}
```

### 3. Set and Get Values

Programmatically control form values:

```jsx
function SetGetValuesExample() {
  const { register, setValue, getValues, handleSubmit } = useForm();

  const fillForm = () => {
    setValue('firstName', 'John');
    setValue('lastName', 'Doe');
    setValue('email', 'john@example.com', {
      shouldValidate: true,  // Trigger validation
      shouldDirty: true,     // Mark as dirty
      shouldTouch: true,     // Mark as touched
    });
  };

  const logValues = () => {
    // Get all values
    console.log(getValues());
    
    // Get specific value
    console.log(getValues('firstName'));
    
    // Get multiple values
    console.log(getValues(['firstName', 'lastName']));
  };

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('firstName')} />
      <input {...register('lastName')} />
      <input {...register('email')} />
      
      <button type="button" onClick={fillForm}>Fill Form</button>
      <button type="button" onClick={logValues}>Log Values</button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Validation

### 1. Built-in Validation

Use built-in validation rules:

```jsx
function BuiltInValidationExample() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input
        {...register('username', {
          required: 'Username is required',
          minLength: {
            value: 3,
            message: 'Minimum 3 characters',
          },
          maxLength: {
            value: 20,
            message: 'Maximum 20 characters',
          },
        })}
      />
      {errors.username && <span>{errors.username.message}</span>}

      <input
        type="number"
        {...register('age', {
          min: {
            value: 18,
            message: 'Must be 18 or older',
          },
          max: {
            value: 99,
            message: 'Must be under 100',
          },
        })}
      />
      {errors.age && <span>{errors.age.message}</span>}

      <input
        {...register('email', {
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email',
          },
        })}
      />
      {errors.email && <span>{errors.email.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Custom Validation

Create custom validation functions:

```jsx
function CustomValidationExample() {
  const { register, handleSubmit, watch, formState: { errors } } = useForm();
  
  const password = watch('password');

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input
        {...register('password', {
          required: 'Password is required',
          validate: {
            hasUpperCase: (value) =>
              /[A-Z]/.test(value) || 'Must contain uppercase letter',
            hasLowerCase: (value) =>
              /[a-z]/.test(value) || 'Must contain lowercase letter',
            hasNumber: (value) =>
              /[0-9]/.test(value) || 'Must contain number',
            hasSpecialChar: (value) =>
              /[!@#$%^&*]/.test(value) || 'Must contain special character',
          },
        })}
        type="password"
      />
      {errors.password && <span>{errors.password.message}</span>}

      <input
        {...register('confirmPassword', {
          required: 'Please confirm password',
          validate: (value) =>
            value === password || 'Passwords do not match',
        })}
        type="password"
      />
      {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 3. Async Validation

Validate with async functions:

```jsx
function AsyncValidationExample() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  const checkUsernameAvailability = async (username) => {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    const taken = ['admin', 'user', 'test'];
    return !taken.includes(username.toLowerCase()) || 'Username already taken';
  };

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input
        {...register('username', {
          required: 'Username is required',
          validate: checkUsernameAvailability,
        })}
      />
      {errors.username && <span>{errors.username.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 4. Schema Validation with Zod

Use Zod for schema-based validation:

```jsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const schema = z.object({
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username must be less than 20 characters'),
  email: z.string()
    .email('Invalid email address'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain uppercase letter')
    .regex(/[a-z]/, 'Password must contain lowercase letter')
    .regex(/[0-9]/, 'Password must contain number'),
  age: z.number()
    .min(18, 'Must be 18 or older')
    .max(99, 'Must be under 100'),
  terms: z.boolean()
    .refine(val => val === true, 'You must accept terms'),
});

function ZodValidationExample() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('username')} />
      {errors.username && <span>{errors.username.message}</span>}

      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <input {...register('password')} type="password" />
      {errors.password && <span>{errors.password.message}</span>}

      <input {...register('age', { valueAsNumber: true })} type="number" />
      {errors.age && <span>{errors.age.message}</span>}

      <input {...register('terms')} type="checkbox" />
      {errors.terms && <span>{errors.terms.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 5. Yup Schema Validation

Use Yup for validation:

```jsx
import { useForm } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';

const schema = yup.object({
  firstName: yup.string()
    .required('First name is required')
    .min(2, 'Too short'),
  lastName: yup.string()
    .required('Last name is required'),
  email: yup.string()
    .email('Invalid email')
    .required('Email is required'),
  age: yup.number()
    .positive('Must be positive')
    .integer('Must be an integer')
    .min(18, 'Must be 18+')
    .required('Age is required'),
  website: yup.string()
    .url('Must be a valid URL')
    .nullable(),
}).required();

function YupValidationExample() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm({
    resolver: yupResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('firstName')} />
      {errors.firstName && <span>{errors.firstName.message}</span>}

      <input {...register('lastName')} />
      {errors.lastName && <span>{errors.lastName.message}</span>}

      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}

      <input {...register('age')} type="number" />
      {errors.age && <span>{errors.age.message}</span>}

      <input {...register('website')} />
      {errors.website && <span>{errors.website.message}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

## Advanced Features

### 1. Field Arrays

Handle dynamic form fields:

```jsx
import { useForm, useFieldArray } from 'react-hook-form';

function FieldArrayExample() {
  const { register, control, handleSubmit } = useForm({
    defaultValues: {
      users: [{ name: '', email: '' }],
    },
  });

  const { fields, append, remove, move } = useFieldArray({
    control,
    name: 'users',
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input
            {...register(`users.${index}.name`)}
            placeholder="Name"
          />
          <input
            {...register(`users.${index}.email`)}
            placeholder="Email"
          />
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}

      <button
        type="button"
        onClick={() => append({ name: '', email: '' })}
      >
        Add User
      </button>

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Controlled Components with Controller

Use Controller for controlled components:

```jsx
import { useForm, Controller } from 'react-hook-form';
import Select from 'react-select';
import DatePicker from 'react-datepicker';

function ControllerExample() {
  const { control, handleSubmit } = useForm({
    defaultValues: {
      country: null,
      birthday: new Date(),
    },
  });

  const countries = [
    { value: 'us', label: 'United States' },
    { value: 'uk', label: 'United Kingdom' },
    { value: 'ca', label: 'Canada' },
  ];

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="country"
        control={control}
        rules={{ required: 'Country is required' }}
        render={({ field, fieldState: { error } }) => (
          <>
            <Select
              {...field}
              options={countries}
              placeholder="Select country"
            />
            {error && <span>{error.message}</span>}
          </>
        )}
      />

      <Controller
        name="birthday"
        control={control}
        render={({ field }) => (
          <DatePicker
            selected={field.value}
            onChange={field.onChange}
            dateFormat="MM/dd/yyyy"
          />
        )}
      />

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 3. Form Reset

Reset form to default or specific values:

```jsx
function ResetExample() {
  const { register, handleSubmit, reset } = useForm({
    defaultValues: {
      firstName: '',
      lastName: '',
    },
  });

  const onSubmit = (data) => {
    console.log(data);
    // Reset after submission
    reset();
  };

  const resetToDefault = () => {
    reset(); // Reset to defaultValues
  };

  const resetToSpecific = () => {
    reset({
      firstName: 'John',
      lastName: 'Doe',
    });
  };

  const resetPartial = () => {
    reset(
      { firstName: 'Jane' },
      { keepDefaultValues: true } // Keep other defaults
    );
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('firstName')} />
      <input {...register('lastName')} />
      
      <button type="submit">Submit</button>
      <button type="button" onClick={resetToDefault}>Reset</button>
      <button type="button" onClick={resetToSpecific}>Fill Form</button>
      <button type="button" onClick={resetPartial}>Reset Partial</button>
    </form>
  );
}
```

### 4. Trigger Validation

Manually trigger validation:

```jsx
function TriggerValidationExample() {
  const { register, trigger, formState: { errors } } = useForm();

  const validateField = async (fieldName) => {
    const isValid = await trigger(fieldName);
    console.log(`${fieldName} is valid:`, isValid);
  };

  const validateAll = async () => {
    const isValid = await trigger();
    console.log('Form is valid:', isValid);
  };

  const validateMultiple = async () => {
    const isValid = await trigger(['firstName', 'lastName']);
    console.log('Selected fields are valid:', isValid);
  };

  return (
    <form>
      <input
        {...register('firstName', { required: 'Required' })}
        onBlur={() => validateField('firstName')}
      />
      {errors.firstName && <span>{errors.firstName.message}</span>}

      <input
        {...register('lastName', { required: 'Required' })}
      />
      {errors.lastName && <span>{errors.lastName.message}</span>}

      <button type="button" onClick={validateAll}>
        Validate All
      </button>
      <button type="button" onClick={validateMultiple}>
        Validate Names
      </button>
    </form>
  );
}
```

### 5. Conditional Fields

Show/hide fields based on conditions:

```jsx
function ConditionalFieldsExample() {
  const { register, watch, handleSubmit } = useForm();
  
  const hasAccount = watch('hasAccount');
  const accountType = watch('accountType');

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <label>
        <input
          {...register('hasAccount')}
          type="checkbox"
        />
        I have an account
      </label>

      {hasAccount && (
        <>
          <select {...register('accountType')}>
            <option value="personal">Personal</option>
            <option value="business">Business</option>
          </select>

          {accountType === 'business' && (
            <input
              {...register('companyName', { required: 'Required for business' })}
              placeholder="Company Name"
            />
          )}
        </>
      )}

      <button type="submit">Submit</button>
    </form>
  );
}
```

## Integration

### 1. React Select Integration

```jsx
import { useForm, Controller } from 'react-hook-form';
import Select from 'react-select';

function ReactSelectIntegration() {
  const { control, handleSubmit } = useForm();

  const options = [
    { value: 'chocolate', label: 'Chocolate' },
    { value: 'vanilla', label: 'Vanilla' },
    { value: 'strawberry', label: 'Strawberry' },
  ];

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="flavor"
        control={control}
        rules={{ required: 'Please select a flavor' }}
        render={({ field, fieldState: { error } }) => (
          <>
            <Select
              {...field}
              options={options}
              isMulti
              placeholder="Select flavors"
            />
            {error && <span>{error.message}</span>}
          </>
        )}
      />

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Material-UI Integration

```jsx
import { useForm, Controller } from 'react-hook-form';
import { TextField, Checkbox, FormControlLabel } from '@mui/material';

function MaterialUIIntegration() {
  const { control, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="email"
        control={control}
        defaultValue=""
        rules={{
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email',
          },
        }}
        render={({ field, fieldState: { error } }) => (
          <TextField
            {...field}
            label="Email"
            error={!!error}
            helperText={error?.message}
            fullWidth
          />
        )}
      />

      <Controller
        name="agreeToTerms"
        control={control}
        defaultValue={false}
        rules={{ required: 'You must agree to terms' }}
        render={({ field, fieldState: { error } }) => (
          <>
            <FormControlLabel
              control={<Checkbox {...field} checked={field.value} />}
              label="I agree to terms"
            />
            {error && <span>{error.message}</span>}
          </>
        )}
      />

      <button type="submit">Submit</button>
    </form>
  );
}
```

### 3. Custom Input Component

```jsx
import { forwardRef } from 'react';
import { useForm } from 'react-hook-form';

const CustomInput = forwardRef(({ onChange, onBlur, name, label }, ref) => (
  <div>
    <label>{label}</label>
    <input
      name={name}
      ref={ref}
      onChange={onChange}
      onBlur={onBlur}
    />
  </div>
));

function CustomInputIntegration() {
  const { register, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <CustomInput
        label="First Name"
        {...register('firstName', { required: true })}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Performance Optimization

### 1. Minimize Re-renders

Use mode option to control validation timing:

```jsx
function PerformanceExample() {
  const { register, handleSubmit } = useForm({
    mode: 'onBlur',        // Validate on blur
    // mode: 'onChange',   // Validate on change (more re-renders)
    // mode: 'onSubmit',   // Validate on submit (default)
    // mode: 'onTouched',  // Validate after first blur, then onChange
    // mode: 'all',        // Validate on blur and change
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('email', { required: true })} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Isolate Re-renders

Use useWatch for specific field watching:

```jsx
import { useForm, useWatch } from 'react-hook-form';

function IsolatedComponent({ control }) {
  const firstName = useWatch({
    control,
    name: 'firstName',
    defaultValue: '',
  });

  // Only this component re-renders when firstName changes
  return <div>First Name: {firstName}</div>;
}

function IsolateRerenders() {
  const { register, control, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('firstName')} />
      <input {...register('lastName')} />
      
      <IsolatedComponent control={control} />
      
      <button type="submit">Submit</button>
    </form>
  );
}
```

### 3. Optimize Field Arrays

Use shouldUnregister for dynamic fields:

```jsx
function OptimizedFieldArray() {
  const { register, control, handleSubmit } = useForm({
    shouldUnregister: true, // Unregister removed fields
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input {...register(`items.${index}.value`)} />
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => append({ value: '' })}>
        Add
      </button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Common Mistakes

### 1. Not Using Key in Field Arrays

```jsx
// ❌ Wrong - using index as key
{fields.map((field, index) => (
  <div key={index}>
    <input {...register(`items.${index}.value`)} />
  </div>
))}

// ✅ Correct - using field.id
{fields.map((field, index) => (
  <div key={field.id}>
    <input {...register(`items.${index}.value`)} />
  </div>
))}
```

### 2. Incorrect Default Values

```jsx
// ❌ Wrong - changing defaultValues after mount
const [defaults, setDefaults] = useState({});
const { register } = useForm({ defaultValues: defaults });

// ✅ Correct - use reset to update values
const { register, reset } = useForm({ defaultValues: {} });
useEffect(() => {
  reset(newDefaults);
}, [newDefaults]);
```

### 3. Not Handling Async Submit

```jsx
// ❌ Wrong - not handling async errors
const onSubmit = async (data) => {
  await api.submit(data); // Error not caught
};

// ✅ Correct - wrap in try-catch
const onSubmit = async (data) => {
  try {
    await api.submit(data);
  } catch (error) {
    console.error(error);
  }
};
```

### 4. Accessing Form State Incorrectly

```jsx
// ❌ Wrong - accessing formState properties individually
const { formState } = useForm();
const errors = formState.errors; // Doesn't subscribe

// ✅ Correct - destructure directly
const { formState: { errors } } = useForm();
```

### 5. Using watch Without Dependencies

```jsx
// ❌ Wrong - watch in render without useEffect
function Component() {
  const { watch } = useForm();
  const value = watch('field'); // New subscription every render
}

// ✅ Correct - use in useEffect or as callback
function Component() {
  const { watch } = useForm();
  useEffect(() => {
    const subscription = watch((value) => console.log(value));
    return () => subscription.unsubscribe();
  }, [watch]);
}
```

## Best Practices

### 1. Use TypeScript

Define form types:

```typescript
import { useForm, SubmitHandler } from 'react-hook-form';

interface FormInputs {
  firstName: string;
  lastName: string;
  email: string;
  age: number;
}

function TypedForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<FormInputs>();

  const onSubmit: SubmitHandler<FormInputs> = (data) => {
    console.log(data); // data is properly typed
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('firstName')} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### 2. Create Reusable Form Components

```jsx
// FormInput.jsx
function FormInput({ register, name, label, rules, errors }) {
  return (
    <div>
      <label>{label}</label>
      <input {...register(name, rules)} />
      {errors[name] && <span>{errors[name].message}</span>}
    </div>
  );
}

// Usage
function Form() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <FormInput
        register={register}
        name="email"
        label="Email"
        rules={{ required: 'Email is required' }}
        errors={errors}
      />
    </form>
  );
}
```

### 3. Use Schema Validation

Prefer schema validation over inline rules:

```jsx
// Better maintainability and type safety
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

const { register } = useForm({
  resolver: zodResolver(schema),
});
```

### 4. Handle Loading States

```jsx
function LoadingStateForm() {
  const { register, handleSubmit, formState: { isSubmitting } } = useForm();

  const onSubmit = async (data) => {
    await new Promise(resolve => setTimeout(resolve, 2000));
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} disabled={isSubmitting} />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

### 5. Separate Validation Logic

```jsx
// validation.js
export const emailValidation = {
  required: 'Email is required',
  pattern: {
    value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
    message: 'Invalid email',
  },
};

export const passwordValidation = {
  required: 'Password is required',
  minLength: {
    value: 8,
    message: 'Minimum 8 characters',
  },
};

// form.jsx
import { emailValidation, passwordValidation } from './validation';

function Form() {
  const { register } = useForm();
  
  return (
    <form>
      <input {...register('email', emailValidation)} />
      <input {...register('password', passwordValidation)} />
    </form>
  );
}
```

## When to Use

### Use React Hook Form When

1. **Performance critical** - Need minimal re-renders
2. **Complex forms** - Multiple fields, validation, dynamic fields
3. **TypeScript projects** - Excellent type inference
4. **Schema validation** - Using Zod, Yup, or Joi
5. **Controlled components** - Need to integrate third-party UI libraries

### Alternatives

- **Formik** - More opinionated, larger bundle
- **Final Form** - Subscription-based, plugin system
- **Redux Form** - Redux integration (outdated)
- **React State** - Simple forms, minimal validation

## Interview Questions

### 1. How does React Hook Form achieve better performance than Formik?

**Answer:** React Hook Form uses uncontrolled components with refs instead of controlled components. This means:
- Form inputs don't cause re-renders on every keystroke
- Only validates and re-renders when necessary
- Uses subscriptions to watch specific fields
- Isolates re-renders to components that need updates
- Smaller bundle size (9kb vs 27kb for Formik)

### 2. Explain the difference between register and Controller.

**Answer:**
- **register**: For native HTML inputs, returns ref and event handlers. Uncontrolled approach.
- **Controller**: For controlled components (third-party UI libraries). Provides render prop with value and onChange.

```jsx
// register - uncontrolled
<input {...register('name')} />

// Controller - controlled
<Controller
  name="name"
  control={control}
  render={({ field }) => <CustomInput {...field} />}
/>
```

### 3. What are the different validation modes and when would you use each?

**Answer:**
- **onSubmit** (default): Validate on form submission. Best for performance.
- **onBlur**: Validate when field loses focus. Good UX balance.
- **onChange**: Validate on every change. Real-time feedback but more re-renders.
- **onTouched**: Validate on blur first, then on change. Progressive enhancement.
- **all**: Validate on both blur and change. Maximum feedback.

Choose based on UX requirements vs performance needs.

### 4. How do you handle dynamic form fields?

**Answer:** Use `useFieldArray` hook:
```jsx
const { fields, append, remove, move, insert } = useFieldArray({
  control,
  name: 'items',
});

// Add field
append({ value: '' });

// Remove field
remove(index);

// Move field
move(fromIndex, toIndex);

// Always use field.id as key
{fields.map((field, index) => (
  <div key={field.id}>
    <input {...register(`items.${index}.value`)} />
  </div>
))}
```

### 5. How do you integrate React Hook Form with UI libraries like Material-UI?

**Answer:** Use the Controller component:
```jsx
<Controller
  name="email"
  control={control}
  rules={{ required: true }}
  render={({ field, fieldState: { error } }) => (
    <TextField
      {...field}
      error={!!error}
      helperText={error?.message}
    />
  )}
/>
```

Controller provides proper value/onChange props and handles validation integration.

## Key Takeaways

1. **React Hook Form uses uncontrolled components** for better performance and minimal re-renders
2. **Register function connects inputs** with validation rules and form state
3. **Schema validation with Zod/Yup** provides type safety and reusable validation
4. **useFieldArray handles dynamic fields** with proper key management using field.id
5. **Controller component integrates controlled components** from UI libraries
6. **FormState provides comprehensive state** including errors, isDirty, isValid, isSubmitting
7. **Watch function monitors field values** without causing unnecessary re-renders
8. **Validation modes control timing** - onSubmit, onBlur, onChange, onTouched, all
9. **TypeScript support is excellent** with full type inference for form data
10. **Performance optimized by default** through uncontrolled inputs and targeted subscriptions

## Resources

### Official Documentation
- [React Hook Form Docs](https://react-hook-form.com/)
- [API Reference](https://react-hook-form.com/api)
- [Examples](https://github.com/react-hook-form/react-hook-form/tree/master/examples)

### Tutorials
- [React Hook Form Tutorial](https://www.freecodecamp.org/news/how-to-create-forms-in-react-using-react-hook-form/)
- [Advanced Patterns](https://www.youtube.com/watch?v=KzcPKB9SOEg)

### Integration
- [Zod Resolver](https://github.com/react-hook-form/resolvers#zod)
- [Yup Resolver](https://github.com/react-hook-form/resolvers#yup)
- [Material-UI Integration](https://react-hook-form.com/get-started#IntegratingwithUIlibraries)

### Tools
- [Form Builder](https://react-hook-form.com/form-builder)
- [DevTools](https://react-hook-form.com/dev-tools)

### Community
- [GitHub Repository](https://github.com/react-hook-form/react-hook-form)
- [Discord Community](https://discord.gg/yYv7GZ8)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/react-hook-form)
