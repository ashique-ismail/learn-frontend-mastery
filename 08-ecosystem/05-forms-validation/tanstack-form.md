# TanStack Form

## What It Is

TanStack Form is a headless, framework-agnostic form management library with first-class TypeScript support. It provides type-safe field paths, validation at multiple levels, and adapters for React, Angular, Vue, and Solid.

---

## Core Philosophy

- **Type-safe throughout**: form values, field names, and errors are all typed
- **No magic strings**: field access is typed via generics, not string paths
- **Validation anywhere**: field-level, form-level, sync and async
- **Framework-agnostic core**: same API across React, Angular, Vue

---

## React Setup

```bash
npm install @tanstack/react-form
```

```tsx
import { useForm } from '@tanstack/react-form';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'At least 8 characters'),
  age: z.number().min(18, 'Must be 18+'),
});

function LoginForm() {
  const form = useForm({
    defaultValues: { email: '', password: '', age: 0 },
    onSubmit: async ({ value }) => {
      await api.login(value);
    },
    validators: {
      onChange: schema, // Zod schema validates on every change
    },
  });

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        form.handleSubmit();
      }}
    >
      <form.Field name="email">
        {(field) => (
          <label>
            Email
            <input
              value={field.state.value}
              onChange={(e) => field.handleChange(e.target.value)}
              onBlur={field.handleBlur}
            />
            {field.state.meta.errors.map((error) => (
              <span key={error} className="error">{error}</span>
            ))}
          </label>
        )}
      </form.Field>

      <form.Field name="password">
        {(field) => (
          <label>
            Password
            <input
              type="password"
              value={field.state.value}
              onChange={(e) => field.handleChange(e.target.value)}
              onBlur={field.handleBlur}
            />
            {field.state.meta.errors.join(', ')}
          </label>
        )}
      </form.Field>

      <form.Subscribe
        selector={(state) => [state.canSubmit, state.isSubmitting]}
      >
        {([canSubmit, isSubmitting]) => (
          <button type="submit" disabled={!canSubmit}>
            {isSubmitting ? 'Logging in...' : 'Log In'}
          </button>
        )}
      </form.Subscribe>
    </form>
  );
}
```

---

## Async Validation

```tsx
<form.Field
  name="username"
  asyncDebounceMs={500}  // debounce async validation
  validators={{
    onChange: z.string().min(3),       // sync: immediate
    onChangeAsync: async ({ value }) => { // async: after debounce
      const exists = await api.checkUsername(value);
      if (exists) return 'Username already taken';
      return undefined; // no error
    },
  }}
>
  {(field) => (
    <>
      <input value={field.state.value} onChange={(e) => field.handleChange(e.target.value)} />
      {field.state.meta.isValidating && <span>Checking...</span>}
      {field.state.meta.errors}
    </>
  )}
</form.Field>
```

---

## Arrays and Nested Objects

```tsx
const form = useForm({
  defaultValues: {
    name: '',
    addresses: [{ street: '', city: '' }],
  },
});

// Nested field: 'addresses[0].street'
<form.Field name="addresses[0].street">
  {(field) => <input value={field.state.value} onChange={...} />}
</form.Field>

// Dynamic arrays
<form.Field name="addresses" mode="array">
  {(field) => (
    <>
      {field.state.value.map((_, i) => (
        <form.Field key={i} name={`addresses[${i}].street`}>
          {(subField) => (
            <input value={subField.state.value} onChange={...} />
          )}
        </form.Field>
      ))}
      <button onClick={() => field.pushValue({ street: '', city: '' })}>
        Add Address
      </button>
    </>
  )}
</form.Field>
```

---

## Form State

```tsx
<form.Subscribe selector={(state) => state}>
  {(state) => (
    <pre>
      {JSON.stringify({
        values: state.values,
        isDirty: state.isDirty,         // any field changed from default
        isValid: state.isValid,         // no errors
        isSubmitting: state.isSubmitting,
        isSubmitted: state.isSubmitted,
        canSubmit: state.canSubmit,     // valid + not submitting
        submissionAttempts: state.submissionAttempts,
      }, null, 2)}
    </pre>
  )}
</form.Subscribe>
```

---

## Comparison with React Hook Form

| | TanStack Form | React Hook Form |
|---|---|---|
| Type safety | End-to-end (field paths typed) | Good (basic inference) |
| Validation | Built-in, multi-level | External resolvers |
| Framework support | React, Angular, Vue, Solid | React only |
| Bundle size | ~12 KB | ~9 KB |
| Ecosystem/maturity | Newer, growing | Massive, very mature |
| Performance | Comparable | Excellent (uncontrolled) |
| Mental model | Controlled, explicit | Uncontrolled by default |

---

## Common Interview Questions

**Q: How does TanStack Form's type safety work?**
The form's generic type parameter constrains field names to only valid paths in the `defaultValues` object. `form.Field name="emayl"` would be a TypeScript error if the field is named `email`. Field values are also typed — `field.handleChange` expects the correct type for that field.

**Q: When would you choose TanStack Form over React Hook Form?**
When type-safe field paths are critical (reduces typos in large forms), when using frameworks other than React, or when building new projects and wanting the latest patterns. React Hook Form is the safer choice for existing projects or when ecosystem plugins matter.
