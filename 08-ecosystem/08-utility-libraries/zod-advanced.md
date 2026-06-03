# Zod Advanced Patterns

## Beyond Basic Types

```ts
import { z } from 'zod';

// Basic recap
const User = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
});
type User = z.infer<typeof User>;
```

---

## Discriminated Unions

For type-safe sum types where a `type`/`kind` field determines the shape:

```ts
const Shape = z.discriminatedUnion('type', [
  z.object({ type: z.literal('circle'), radius: z.number() }),
  z.object({ type: z.literal('rectangle'), width: z.number(), height: z.number() }),
  z.object({ type: z.literal('triangle'), base: z.number(), height: z.number() }),
]);

type Shape = z.infer<typeof Shape>;
// { type: 'circle'; radius: number } | { type: 'rectangle'; width: number; height: number } | ...

// Runtime validation + TypeScript narrowing
const shape = Shape.parse({ type: 'circle', radius: 5 });
if (shape.type === 'circle') {
  console.log(shape.radius); // TypeScript knows this is safe
}
```

**Why not `z.union()`?**
`discriminatedUnion` is faster (O(1) lookup by discriminant key) and gives better error messages.

---

## Transforms

```ts
// Transform during parsing
const StringToNumber = z.string().transform(val => parseInt(val, 10));
StringToNumber.parse('42'); // → 42 (number)

// Transform with validation
const Timestamp = z.string().transform((str, ctx) => {
  const date = new Date(str);
  if (isNaN(date.getTime())) {
    ctx.addIssue({ code: z.ZodIssueCode.custom, message: 'Invalid date' });
    return z.NEVER; // signal failure
  }
  return date;
});

// Pipeline: validate input → transform → validate output
const TrimmedEmail = z
  .string()
  .transform(s => s.toLowerCase().trim())
  .pipe(z.string().email()); // validate the transformed value
```

---

## Refinements

Custom validation logic:

```ts
// Single-field validation
const Password = z.string().min(8).refine(
  (val) => /[A-Z]/.test(val),
  { message: 'Must contain uppercase letter' }
);

// Multiple refinements
const StrongPassword = z.string()
  .min(8, 'At least 8 characters')
  .refine(val => /[A-Z]/.test(val), 'Need uppercase')
  .refine(val => /[0-9]/.test(val), 'Need a number')
  .refine(val => /[^A-Za-z0-9]/.test(val), 'Need special character');

// `superRefine` for multiple issues in one pass
const PasswordConfirm = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).superRefine(({ password, confirmPassword }, ctx) => {
  if (password !== confirmPassword) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'Passwords do not match',
      path: ['confirmPassword'], // attach error to specific field
    });
  }
});
```

---

## Branded Types

Create nominal types — structurally identical types that TypeScript treats as distinct:

```ts
// Without branding: UserId and ProductId are both just 'string'
type UserId = string;
type ProductId = string;
function deleteUser(id: UserId) {}
const productId = 'prod-123' as ProductId;
deleteUser(productId); // ✅ TypeScript allows this — WRONG!

// With Zod branding: runtime-validated nominal types
const UserId = z.string().uuid().brand<'UserId'>();
const ProductId = z.string().uuid().brand<'ProductId'>();
type UserId = z.infer<typeof UserId>;   // string & { __brand: 'UserId' }
type ProductId = z.infer<typeof ProductId>;

function deleteUser(id: UserId) {}
const productId = ProductId.parse('550e8400-e29b-41d4-a716-446655440000');
deleteUser(productId); // ❌ TypeScript error: ProductId ≠ UserId
```

---

## Coerce

Parse values that may come in different types (e.g., URL query params are always strings):

```ts
const SearchParams = z.object({
  page: z.coerce.number().int().min(1).default(1),  // '2' → 2
  perPage: z.coerce.number().int().min(1).max(100).default(20),
  active: z.coerce.boolean(),  // 'true' → true, '1' → true
  date: z.coerce.date(),       // '2024-01-01' → Date object
});

SearchParams.parse({ page: '3', perPage: '50', active: 'true', date: '2024-01-01' });
// → { page: 3, perPage: 50, active: true, date: Date }
```

---

## ZodError Formatting

```ts
const result = UserSchema.safeParse(invalidData);

if (!result.success) {
  // Flat format: { fieldErrors: { email: ['Invalid email'] }, formErrors: [] }
  const flat = result.error.flatten();
  console.log(flat.fieldErrors.email); // ['Invalid email']

  // Formatted: nested object following the input structure
  const formatted = result.error.format();
  console.log(formatted.email?._errors); // ['Invalid email']

  // All issues as array
  result.error.issues.forEach(issue => {
    console.log(issue.path.join('.'), issue.message);
  });
}
```

---

## Schema Composition

```ts
const BaseEntity = z.object({
  id: z.string().uuid(),
  createdAt: z.date(),
  updatedAt: z.date(),
});

const User = BaseEntity.extend({
  name: z.string(),
  email: z.string().email(),
});

// Partial: all fields optional
const UserUpdate = User.partial();

// Pick/Omit
const UserPreview = User.pick({ id: true, name: true });
const UserWithoutDates = User.omit({ createdAt: true, updatedAt: true });

// Merge two schemas
const AdminUser = User.merge(z.object({ permissions: z.array(z.string()) }));
```

---

## Common Interview Questions

**Q: What's the difference between `parse` and `safeParse`?**
`parse()` throws a `ZodError` on validation failure. `safeParse()` returns `{ success: true, data }` or `{ success: false, error }`. Use `safeParse` when you want to handle errors gracefully (form validation), use `parse` when invalid data is a programming error (config loading).

**Q: How does Zod integrate with React Hook Form?**
`@hookform/resolvers/zod` provides a resolver: `useForm({ resolver: zodResolver(schema) })`. The schema validates the entire form on submit and provides per-field errors via `formState.errors`.

**Q: What's the performance cost of runtime validation?**
Minimal for most use cases. `parse()` traverses the input once. For hot paths (called thousands of times per second), consider validating at boundaries only (API input, form submission) rather than on every function call.
