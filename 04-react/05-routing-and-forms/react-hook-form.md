# React Hook Form

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Basic Usage](#basic-usage)
4. [Form Validation](#form-validation)
5. [Controller Component](#controller-component)
6. [Advanced Patterns](#advanced-patterns)
7. [Performance Optimization](#performance-optimization)
8. [Integration with UI Libraries](#integration-with-ui-libraries)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)
12. [Resources](#resources)

## Introduction

React Hook Form (RHF) is a performant, flexible form library built on uncontrolled inputs. It minimizes re-renders, reduces boilerplate, and provides excellent developer experience with TypeScript support and easy validation integration.

**Key Benefits:**
- Minimal re-renders (uncontrolled by default)
- Smaller bundle size (~9kb gzipped)
- No dependencies
- Built-in validation
- Easy integration with UI libraries
- Excellent TypeScript support

## Core Concepts

### Uncontrolled vs Controlled

```typescript
// Traditional controlled input - re-renders on every keystroke
function ControlledForm() {
  const [name, setName] = useState('');
  
  return (
    <input 
      value={name} 
      onChange={(e) => setName(e.target.value)} 
    />
  );
}

// React Hook Form - uncontrolled, minimal re-renders
import { useForm } from 'react-hook-form';

function RHFForm() {
  const { register } = useForm();
  
  return <input {...register('name')} />;
}
```

### Core API

```typescript
import { useForm } from 'react-hook-form';

interface FormData {
  email: string;
  password: string;
  rememberMe: boolean;
}

function LoginForm() {
  const {
    register,        // Register inputs
    handleSubmit,    // Handle form submission
    watch,           // Watch input values
    formState,       // Form state (errors, isDirty, etc.)
    reset,           // Reset form
    setValue,        // Set input value programmatically
    getValues,       // Get current form values
    trigger,         // Trigger validation manually
    control          // For Controller component
  } = useForm<FormData>({
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false
    }
  });
  
  const { errors, isDirty, isValid, isSubmitting } = formState;
  
  const onSubmit = (data: FormData) => {
    console.log(data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
}
```

## Basic Usage

### Simple Form

```typescript
import { useForm, SubmitHandler } from 'react-hook-form';

interface ContactFormData {
  name: string;
  email: string;
  message: string;
}

function ContactForm() {
  const { 
    register, 
    handleSubmit, 
    formState: { errors } 
  } = useForm<ContactFormData>();
  
  const onSubmit: SubmitHandler<ContactFormData> = async (data) => {
    await fetch('/api/contact', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input
          {...register('name', { 
            required: 'Name is required',
            minLength: { value: 2, message: 'Name too short' }
          })}
          placeholder="Your name"
        />
        {errors.name && <span>{errors.name.message}</span>}
      </div>
      
      <div>
        <input
          {...register('email', {
            required: 'Email is required',
            pattern: {
              value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
              message: 'Invalid email address'
            }
          })}
          placeholder="your@email.com"
        />
        {errors.email && <span>{errors.email.message}</span>}
      </div>
      
      <div>
        <textarea
          {...register('message', {
            required: 'Message is required',
            maxLength: { value: 500, message: 'Message too long' }
          })}
          placeholder="Your message"
        />
        {errors.message && <span>{errors.message.message}</span>}
      </div>
      
      <button type="submit">Send</button>
    </form>
  );
}
```

### Form with Default Values

```typescript
interface UserFormData {
  name: string;
  email: string;
  age: number;
}

function EditUserForm({ user }: { user: UserFormData }) {
  const { register, handleSubmit, reset } = useForm<UserFormData>({
    defaultValues: user
  });
  
  const onSubmit = async (data: UserFormData) => {
    await updateUser(data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      <input {...register('email')} />
      <input {...register('age', { valueAsNumber: true })} />
      
      <button type="submit">Save</button>
      <button type="button" onClick={() => reset()}>
        Reset
      </button>
    </form>
  );
}
```

### Watching Values

```typescript
function DynamicForm() {
  const { register, watch } = useForm<{
    showEmail: boolean;
    email?: string;
  }>();
  
  // Watch single field
  const showEmail = watch('showEmail');
  
  // Watch multiple fields
  const [showEmail, email] = watch(['showEmail', 'email']);
  
  // Watch all fields
  const allValues = watch();
  
  return (
    <form>
      <input type="checkbox" {...register('showEmail')} />
      <label>Show email field</label>
      
      {showEmail && (
        <input {...register('email')} placeholder="Email" />
      )}
    </form>
  );
}
```

### Form State

```typescript
function FormWithState() {
  const { 
    register, 
    handleSubmit, 
    formState: {
      errors,           // Validation errors
      isDirty,          // Any field changed
      dirtyFields,      // Which fields changed
      isValid,          // No validation errors
      isSubmitting,     // Currently submitting
      isSubmitted,      // Has been submitted
      submitCount,      // Number of submissions
      touchedFields,    // Fields that were focused
      isValidating      // Currently validating
    }
  } = useForm({ mode: 'onChange' });
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name', { required: true })} />
      
      <button 
        type="submit" 
        disabled={!isDirty || !isValid || isSubmitting}
      >
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>
      
      <div>
        {isDirty && <p>You have unsaved changes</p>}
        {submitCount > 0 && <p>Submitted {submitCount} times</p>}
      </div>
    </form>
  );
}
```

## Form Validation

### Built-in Validation Rules

```typescript
function ValidationExample() {
  const { register, formState: { errors } } = useForm();
  
  return (
    <form>
      {/* Required */}
      <input {...register('username', { 
        required: 'Username is required' 
      })} />
      
      {/* Min/Max length */}
      <input {...register('password', {
        required: 'Password is required',
        minLength: { value: 8, message: 'Min 8 characters' },
        maxLength: { value: 20, message: 'Max 20 characters' }
      })} />
      
      {/* Min/Max value */}
      <input type="number" {...register('age', {
        min: { value: 18, message: 'Must be 18+' },
        max: { value: 120, message: 'Invalid age' }
      })} />
      
      {/* Pattern (regex) */}
      <input {...register('email', {
        pattern: {
          value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
          message: 'Invalid email'
        }
      })} />
      
      {/* Custom validation */}
      <input {...register('confirmPassword', {
        validate: (value, formValues) => 
          value === formValues.password || 'Passwords must match'
      })} />
      
      {/* Multiple validators */}
      <input {...register('coupon', {
        validate: {
          notEmpty: (value) => value.trim() !== '' || 'Required',
          validFormat: (value) => 
            /^[A-Z0-9]{6}$/.test(value) || 'Invalid format',
          notUsed: async (value) => {
            const isUsed = await checkCouponUsed(value);
            return !isUsed || 'Coupon already used';
          }
        }
      })} />
    </form>
  );
}
```

### Custom Validation Functions

```typescript
// Reusable validators
const validators = {
  email: (value: string) => 
    /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i.test(value) || 
    'Invalid email',
  
  strongPassword: (value: string) => {
    if (value.length < 8) return 'Min 8 characters';
    if (!/[A-Z]/.test(value)) return 'Need uppercase letter';
    if (!/[a-z]/.test(value)) return 'Need lowercase letter';
    if (!/[0-9]/.test(value)) return 'Need number';
    if (!/[^A-Za-z0-9]/.test(value)) return 'Need special character';
    return true;
  },
  
  matchesField: (fieldName: string) => (value: string, formValues: any) =>
    value === formValues[fieldName] || `Must match ${fieldName}`,
  
  uniqueEmail: async (email: string) => {
    const exists = await checkEmailExists(email);
    return !exists || 'Email already registered';
  }
};

function SignupForm() {
  const { register } = useForm();
  
  return (
    <form>
      <input 
        {...register('email', { validate: validators.email })} 
      />
      
      <input 
        type="password"
        {...register('password', { 
          validate: validators.strongPassword 
        })} 
      />
      
      <input 
        type="password"
        {...register('confirmPassword', {
          validate: validators.matchesField('password')
        })} 
      />
    </form>
  );
}
```

### Async Validation

```typescript
function AsyncValidation() {
  const { register, formState: { errors, isValidating } } = useForm({
    mode: 'onBlur' // Validate on blur to avoid too many API calls
  });
  
  return (
    <form>
      <input
        {...register('username', {
          required: 'Username required',
          validate: async (value) => {
            // Show loading during validation
            const available = await checkUsernameAvailable(value);
            return available || 'Username taken';
          }
        })}
      />
      {isValidating && <span>Checking...</span>}
      {errors.username && <span>{errors.username.message}</span>}
    </form>
  );
}
```

### Validation Modes

```typescript
function ValidationModes() {
  // onChange - validate on every change (most reactive)
  const form1 = useForm({ mode: 'onChange' });
  
  // onBlur - validate when field loses focus (less API calls)
  const form2 = useForm({ mode: 'onBlur' });
  
  // onSubmit - validate only on submit (default, best performance)
  const form3 = useForm({ mode: 'onSubmit' });
  
  // onTouched - validate after field is touched, then on change
  const form4 = useForm({ mode: 'onTouched' });
  
  // all - validate on both blur and change
  const form5 = useForm({ mode: 'all' });
  
  // Custom revalidation
  const form6 = useForm({
    mode: 'onSubmit',
    reValidateMode: 'onChange' // After first submit, validate on change
  });
}
```

## Controller Component

Use `Controller` for controlled components (UI libraries, custom inputs).

### Basic Controller

```typescript
import { useForm, Controller } from 'react-hook-form';
import Select from 'react-select';

interface FormData {
  country: { value: string; label: string } | null;
}

function ControllerExample() {
  const { control, handleSubmit } = useForm<FormData>();
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="country"
        control={control}
        rules={{ required: 'Please select a country' }}
        render={({ field, fieldState: { error } }) => (
          <div>
            <Select
              {...field}
              options={countryOptions}
              placeholder="Select country"
            />
            {error && <span>{error.message}</span>}
          </div>
        )}
      />
    </form>
  );
}
```

### Controller with Custom Input

```typescript
interface RatingInputProps {
  value: number;
  onChange: (value: number) => void;
}

function RatingInput({ value, onChange }: RatingInputProps) {
  return (
    <div>
      {[1, 2, 3, 4, 5].map(star => (
        <button
          key={star}
          type="button"
          onClick={() => onChange(star)}
          className={star <= value ? 'active' : ''}
        >
          ★
        </button>
      ))}
    </div>
  );
}

function ReviewForm() {
  const { control, handleSubmit } = useForm<{ rating: number }>();
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="rating"
        control={control}
        defaultValue={0}
        rules={{ 
          required: 'Please rate',
          min: { value: 1, message: 'Min rating is 1' }
        }}
        render={({ field, fieldState: { error } }) => (
          <div>
            <RatingInput 
              value={field.value}
              onChange={field.onChange}
            />
            {error && <span>{error.message}</span>}
          </div>
        )}
      />
    </form>
  );
}
```

### Multiple Controllers

```typescript
import { DatePicker } from '@mui/x-date-pickers';
import { Checkbox } from '@mui/material';

interface EventFormData {
  startDate: Date | null;
  endDate: Date | null;
  allDay: boolean;
}

function EventForm() {
  const { control, watch } = useForm<EventFormData>({
    defaultValues: {
      startDate: null,
      endDate: null,
      allDay: false
    }
  });
  
  const startDate = watch('startDate');
  
  return (
    <form>
      <Controller
        name="startDate"
        control={control}
        rules={{ required: 'Start date required' }}
        render={({ field, fieldState: { error } }) => (
          <div>
            <DatePicker
              label="Start Date"
              value={field.value}
              onChange={field.onChange}
            />
            {error && <span>{error.message}</span>}
          </div>
        )}
      />
      
      <Controller
        name="endDate"
        control={control}
        rules={{
          required: 'End date required',
          validate: (value) => 
            !startDate || !value || value >= startDate ||
            'End date must be after start date'
        }}
        render={({ field, fieldState: { error } }) => (
          <div>
            <DatePicker
              label="End Date"
              value={field.value}
              onChange={field.onChange}
              minDate={startDate}
            />
            {error && <span>{error.message}</span>}
          </div>
        )}
      />
      
      <Controller
        name="allDay"
        control={control}
        render={({ field }) => (
          <Checkbox
            checked={field.value}
            onChange={field.onChange}
            label="All Day Event"
          />
        )}
      />
    </form>
  );
}
```

## Advanced Patterns

### Dynamic Form Fields

```typescript
import { useFieldArray } from 'react-hook-form';

interface TodoFormData {
  todos: Array<{
    text: string;
    completed: boolean;
  }>;
}

function TodoListForm() {
  const { register, control, handleSubmit } = useForm<TodoFormData>({
    defaultValues: {
      todos: [{ text: '', completed: false }]
    }
  });
  
  const { fields, append, remove } = useFieldArray({
    control,
    name: 'todos'
  });
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        <div key={field.id}>
          <input
            {...register(`todos.${index}.text`, {
              required: 'Todo text required'
            })}
            placeholder="Todo"
          />
          
          <input
            type="checkbox"
            {...register(`todos.${index}.completed`)}
          />
          
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}
      
      <button
        type="button"
        onClick={() => append({ text: '', completed: false })}
      >
        Add Todo
      </button>
      
      <button type="submit">Save All</button>
    </form>
  );
}
```

### Nested Objects

```typescript
interface AddressFormData {
  user: {
    name: string;
    email: string;
  };
  address: {
    street: string;
    city: string;
    zipCode: string;
  };
}

function AddressForm() {
  const { register, handleSubmit } = useForm<AddressFormData>();
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('user.name', { required: true })} />
      <input {...register('user.email', { required: true })} />
      
      <input {...register('address.street', { required: true })} />
      <input {...register('address.city', { required: true })} />
      <input {...register('address.zipCode', { required: true })} />
    </form>
  );
}
```

### Conditional Fields

```typescript
function ConditionalForm() {
  const { register, watch } = useForm<{
    accountType: 'personal' | 'business';
    companyName?: string;
    taxId?: string;
  }>();
  
  const accountType = watch('accountType');
  
  return (
    <form>
      <select {...register('accountType')}>
        <option value="personal">Personal</option>
        <option value="business">Business</option>
      </select>
      
      {accountType === 'business' && (
        <>
          <input
            {...register('companyName', {
              required: 'Company name required for business accounts'
            })}
            placeholder="Company Name"
          />
          
          <input
            {...register('taxId', {
              required: 'Tax ID required for business accounts'
            })}
            placeholder="Tax ID"
          />
        </>
      )}
    </form>
  );
}
```

### Form Context

```typescript
import { FormProvider, useFormContext } from 'react-hook-form';

// Child component accessing form context
function EmailField() {
  const { register, formState: { errors } } = useFormContext();
  
  return (
    <div>
      <input {...register('email', { required: 'Email required' })} />
      {errors.email && <span>{errors.email.message}</span>}
    </div>
  );
}

function PasswordField() {
  const { register, formState: { errors } } = useFormContext();
  
  return (
    <div>
      <input
        type="password"
        {...register('password', { required: 'Password required' })}
      />
      {errors.password && <span>{errors.password.message}</span>}
    </div>
  );
}

// Parent form providing context
function SignupForm() {
  const methods = useForm();
  
  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        <EmailField />
        <PasswordField />
        <button type="submit">Sign Up</button>
      </form>
    </FormProvider>
  );
}
```

### Wizard/Multi-Step Forms

```typescript
function MultiStepForm() {
  const [step, setStep] = useState(1);
  const methods = useForm<{
    step1: { name: string; email: string };
    step2: { address: string; city: string };
    step3: { cardNumber: string; cvv: string };
  }>();
  
  const onSubmit = async (data: any) => {
    if (step < 3) {
      setStep(step + 1);
    } else {
      await submitForm(data);
    }
  };
  
  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {step === 1 && <Step1 />}
        {step === 2 && <Step2 />}
        {step === 3 && <Step3 />}
        
        <div>
          {step > 1 && (
            <button type="button" onClick={() => setStep(step - 1)}>
              Previous
            </button>
          )}
          <button type="submit">
            {step === 3 ? 'Submit' : 'Next'}
          </button>
        </div>
      </form>
    </FormProvider>
  );
}

function Step1() {
  const { register } = useFormContext();
  return (
    <div>
      <input {...register('step1.name', { required: true })} />
      <input {...register('step1.email', { required: true })} />
    </div>
  );
}
```

## Performance Optimization

### Isolate Re-renders

```typescript
// ❌ Bad - entire form re-renders on every change
function SlowForm() {
  const { register, watch } = useForm();
  const values = watch(); // Watches everything
  
  return (
    <form>
      <input {...register('field1')} />
      <input {...register('field2')} />
      <input {...register('field3')} />
      <pre>{JSON.stringify(values)}</pre>
    </form>
  );
}

// ✅ Good - only debug component re-renders
function FastForm() {
  const { register, control } = useForm();
  
  return (
    <form>
      <input {...register('field1')} />
      <input {...register('field2')} />
      <input {...register('field3')} />
      <FormDebug control={control} />
    </form>
  );
}

function FormDebug({ control }: { control: Control }) {
  const values = useWatch({ control });
  return <pre>{JSON.stringify(values)}</pre>;
}
```

### shouldUnregister

```typescript
function ConditionalFieldForm() {
  const { register, watch } = useForm({
    shouldUnregister: true // Unregister fields when unmounted
  });
  
  const showOptional = watch('showOptional');
  
  return (
    <form>
      <input type="checkbox" {...register('showOptional')} />
      
      {showOptional && (
        // This field is removed from form values when hidden
        <input {...register('optionalField')} />
      )}
    </form>
  );
}
```

### Lazy Validation

```typescript
function LazyValidationForm() {
  const { register, trigger } = useForm({
    mode: 'onSubmit' // Don't validate until submit
  });
  
  return (
    <form>
      <input {...register('email')} />
      
      {/* Manually trigger validation */}
      <button
        type="button"
        onClick={() => trigger('email')}
      >
        Validate Email
      </button>
    </form>
  );
}
```

## Integration with UI Libraries

### Material-UI

```typescript
import { TextField, Checkbox, FormControlLabel } from '@mui/material';
import { Controller } from 'react-hook-form';

function MUIForm() {
  const { control, handleSubmit } = useForm();
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Controller
        name="username"
        control={control}
        defaultValue=""
        rules={{ required: 'Username required' }}
        render={({ field, fieldState: { error } }) => (
          <TextField
            {...field}
            label="Username"
            error={!!error}
            helperText={error?.message}
            fullWidth
          />
        )}
      />
      
      <Controller
        name="agree"
        control={control}
        defaultValue={false}
        rules={{ required: 'You must agree' }}
        render={({ field, fieldState: { error } }) => (
          <div>
            <FormControlLabel
              control={
                <Checkbox
                  checked={field.value}
                  onChange={field.onChange}
                />
              }
              label="I agree to terms"
            />
            {error && <span>{error.message}</span>}
          </div>
        )}
      />
    </form>
  );
}
```

### Chakra UI

```typescript
import { Input, Checkbox, FormControl, FormLabel, FormErrorMessage } from '@chakra-ui/react';

function ChakraForm() {
  const { register, handleSubmit, formState: { errors } } = useForm();
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <FormControl isInvalid={!!errors.email}>
        <FormLabel>Email</FormLabel>
        <Input
          {...register('email', {
            required: 'Email required',
            pattern: {
              value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
              message: 'Invalid email'
            }
          })}
        />
        <FormErrorMessage>
          {errors.email?.message}
        </FormErrorMessage>
      </FormControl>
    </form>
  );
}
```

## Common Mistakes

### 1. Not Using TypeScript

```typescript
// ❌ Bad - no type safety
const { register } = useForm();
register('usrname'); // Typo not caught

// ✅ Good - type-safe
interface FormData {
  username: string;
}
const { register } = useForm<FormData>();
register('usrname'); // TypeScript error!
```

### 2. Forgetting defaultValues

```typescript
// ❌ Bad - controlled to uncontrolled warning
const { register } = useForm();
<input {...register('name')} />; // undefined initially

// ✅ Good - always provide defaults
const { register } = useForm({
  defaultValues: { name: '' }
});
```

### 3. Using useState with RHF

```typescript
// ❌ Bad - defeats the purpose of RHF
const [email, setEmail] = useState('');
const { register } = useForm();

<input
  {...register('email')}
  value={email}
  onChange={(e) => setEmail(e.target.value)}
/>

// ✅ Good - let RHF manage state
const { register, watch } = useForm();
const email = watch('email'); // If you need the value

<input {...register('email')} />
```

### 4. Not Handling Async Validation

```typescript
// ❌ Bad - blocks UI
<input {...register('username', {
  validate: async (value) => {
    // Runs on every change if mode is 'onChange'
    return await checkUsername(value);
  }
})} />

// ✅ Good - use onBlur mode
const { register } = useForm({ mode: 'onBlur' });
<input {...register('username', {
  validate: async (value) => await checkUsername(value)
})} />
```

## Best Practices

### 1. Type Your Forms

```typescript
// Define form schema
interface SignupFormData {
  email: string;
  password: string;
  confirmPassword: string;
  age: number;
  terms: boolean;
}

// Use throughout
const { register, handleSubmit } = useForm<SignupFormData>();
const onSubmit: SubmitHandler<SignupFormData> = (data) => {
  // data is fully typed
};
```

### 2. Create Reusable Form Components

```typescript
// components/FormInput.tsx
interface FormInputProps {
  name: string;
  label: string;
  rules?: RegisterOptions;
  type?: string;
}

function FormInput({ name, label, rules, type = 'text' }: FormInputProps) {
  const { register, formState: { errors } } = useFormContext();
  const error = errors[name];
  
  return (
    <div className="form-field">
      <label htmlFor={name}>{label}</label>
      <input
        id={name}
        type={type}
        {...register(name, rules)}
        className={error ? 'error' : ''}
      />
      {error && <span className="error-message">{error.message}</span>}
    </div>
  );
}

// Usage
<FormProvider {...methods}>
  <form>
    <FormInput
      name="email"
      label="Email"
      rules={{ required: 'Email required' }}
    />
    <FormInput
      name="password"
      label="Password"
      type="password"
      rules={{ required: 'Password required', minLength: 8 }}
    />
  </form>
</FormProvider>
```

### 3. Separate Validation Logic

```typescript
// validations/signup.ts
export const signupValidation = {
  email: {
    required: 'Email is required',
    pattern: {
      value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
      message: 'Invalid email address'
    }
  },
  password: {
    required: 'Password is required',
    minLength: {
      value: 8,
      message: 'Password must be at least 8 characters'
    },
    validate: {
      hasUpperCase: (value: string) =>
        /[A-Z]/.test(value) || 'Need at least one uppercase letter',
      hasLowerCase: (value: string) =>
        /[a-z]/.test(value) || 'Need at least one lowercase letter',
      hasNumber: (value: string) =>
        /[0-9]/.test(value) || 'Need at least one number'
    }
  }
};

// Use in form
<input {...register('email', signupValidation.email)} />
<input {...register('password', signupValidation.password)} />
```

### 4. Handle Loading and Error States

```typescript
function FormWithStates() {
  const { 
    register, 
    handleSubmit, 
    formState: { isSubmitting, errors },
    setError
  } = useForm();
  
  const onSubmit = async (data) => {
    try {
      await saveData(data);
    } catch (error) {
      setError('root', {
        message: 'Failed to save. Please try again.'
      });
    }
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {errors.root && (
        <div className="error-banner">{errors.root.message}</div>
      )}
      
      {/* form fields */}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

## Interview Questions

### Q1: How does React Hook Form achieve better performance than traditional controlled inputs?

**Answer:** RHF uses uncontrolled inputs and refs, avoiding React state updates on every keystroke. Traditional controlled inputs cause re-renders on every change. RHF only re-renders when necessary (validation errors, submission). This reduces bundle size and improves performance, especially in large forms.

### Q2: When should you use Controller vs register?

**Answer:**
- Use `register` for native HTML inputs (input, select, textarea)
- Use `Controller` for:
  - UI library components (Material-UI, Chakra)
  - Custom components that don't expose ref
  - Components needing controlled behavior
  - Rich editors, date pickers, etc.

### Q3: How do you handle dynamic form fields?

**Answer:** Use `useFieldArray` hook:
- Provides `fields`, `append`, `remove`, `update` methods
- Manages array state efficiently
- Handles validation for array items
- Generates unique keys for React lists
- Tracks which items changed

### Q4: What are the different validation modes?

**Answer:**
- `onSubmit` (default): Validate on submit
- `onChange`: Validate on every change (most reactive)
- `onBlur`: Validate when field loses focus
- `onTouched`: Validate after touched, then onChange
- `all`: Validate on both blur and change
- Choose based on UX needs vs performance

## Resources

### Official Documentation
- React Hook Form Docs: https://react-hook-form.com/
- API Reference: https://react-hook-form.com/api

### Tutorials
- Getting Started: https://react-hook-form.com/get-started
- Advanced Usage: https://react-hook-form.com/advanced-usage

### Tools
- Form Builder: https://react-hook-form.com/form-builder
- DevTools: https://react-hook-form.com/dev-tools

### Integrations
- Zod Integration: https://github.com/react-hook-form/resolvers#zod
- Yup Integration: https://github.com/react-hook-form/resolvers#yup

---

**Next Steps:**
- Build forms with React Hook Form
- Integrate with validation libraries (Zod)
- Create reusable form components
- Implement complex multi-step forms
