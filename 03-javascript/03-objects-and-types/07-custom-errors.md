# Custom Error Classes

## The Idea

**In plain English:** A custom error is a special kind of mistake message you design yourself so that when something goes wrong in your program, the error tells you exactly what type of problem occurred — not just "something broke." Think of it like creating your own categories of problems instead of using a single generic label for every issue.

**Real-world analogy:** Imagine a hospital triage system where every patient gets tagged with a specific wristband color based on their condition — red for emergency, yellow for urgent, green for minor. A generic hospital just says "patient is sick," but triage lets doctors instantly know how to respond.

- The wristband color = the custom error class name (e.g. `UnauthorizedError`, `NotFoundError`)
- The patient's chart with details = the extra fields on the error (e.g. `status`, `url`, `field`)
- The doctor checking the wristband color = `instanceof` in a catch block to handle each error type differently

---

## Overview

JavaScript's built-in `Error` class is a starting point, not an endpoint. Custom error classes let you carry structured context, enable `instanceof` checks in catch blocks, and communicate failure semantics precisely — the difference between "something went wrong" and "the user's session expired with a 401 from /api/profile".

---

## Subclassing Error

```javascript
class AppError extends Error {
  constructor(message, options) {
    super(message, options); // options.cause supported in ES2022+
    this.name = this.constructor.name; // 'AppError', not 'Error'

    // Fix prototype chain in TypeScript/transpiled environments
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

// Usage
const err = new AppError('Something went wrong');
err instanceof AppError; // true
err instanceof Error;    // true
err.name;                // 'AppError'
err.message;             // 'Something went wrong'
```

The `Object.setPrototypeOf` call is necessary when targeting ES5 via TypeScript/Babel — without it, `instanceof` checks for subclasses fail because the prototype chain isn't set correctly after `super()`.

---

## Error Hierarchy

```typescript
// Base with shared fields
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    options?: ErrorOptions
  ) {
    super(message, options);
    this.name = this.constructor.name;
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

// Network layer
class NetworkError extends AppError {
  constructor(
    message: string,
    public readonly status: number,
    public readonly url: string,
    options?: ErrorOptions
  ) {
    super(message, `HTTP_${status}`, options);
  }
}

class NotFoundError extends NetworkError {
  constructor(url: string) {
    super(`Resource not found: ${url}`, 404, url);
  }
}

class UnauthorizedError extends NetworkError {
  constructor(url: string) {
    super(`Unauthorized: ${url}`, 401, url);
  }
}

class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly field: string,
    public readonly value: unknown
  ) {
    super(message, 'VALIDATION_ERROR');
  }
}

class TimeoutError extends AppError {
  constructor(
    public readonly timeoutMs: number,
    public readonly operation: string
  ) {
    super(`Operation '${operation}' timed out after ${timeoutMs}ms`, 'TIMEOUT');
  }
}
```

---

## Error Cause (ES2022)

The `cause` option threads the original error through the chain without losing it:

```javascript
async function fetchUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new NetworkError('Fetch failed', res.status, res.url);
    return res.json();
  } catch (err) {
    // Wrap with context, preserve original as cause
    throw new AppError(`Failed to load user ${id}`, 'USER_LOAD_FAILED', {
      cause: err,
    });
  }
}

// When caught:
try {
  await fetchUser(42);
} catch (err) {
  console.error(err.message);        // "Failed to load user 42"
  console.error(err.cause.message);  // "Fetch failed" (original NetworkError)
  console.error(err.cause.status);   // 404
}
```

---

## AggregateError

`AggregateError` is a built-in that holds multiple errors. Used by `Promise.any()` when all promises reject:

```javascript
// Built-in: thrown by Promise.any() when all reject
try {
  await Promise.any([
    fetch('/api/a').then(r => { if (!r.ok) throw new Error('a failed'); }),
    fetch('/api/b').then(r => { if (!r.ok) throw new Error('b failed'); }),
  ]);
} catch (err) {
  if (err instanceof AggregateError) {
    err.errors.forEach(e => console.error(e.message));
    // 'a failed'
    // 'b failed'
  }
}

// Custom aggregate error
class MultiValidationError extends AggregateError {
  constructor(errors: ValidationError[]) {
    super(errors, `${errors.length} validation error(s)`);
    this.name = 'MultiValidationError';
    Object.setPrototypeOf(this, new.target.prototype);
  }

  get fields(): string[] {
    return (this.errors as ValidationError[]).map(e => e.field);
  }
}
```

---

## Discriminated Union Pattern (TypeScript)

For functions that can fail in well-known ways, a Result type avoids try/catch entirely:

```typescript
type Result<T, E extends AppError = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function safeGetUser(id: string): Promise<Result<User, NetworkError>> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (res.status === 404) return { ok: false, error: new NotFoundError(`/api/users/${id}`) };
    if (res.status === 401) return { ok: false, error: new UnauthorizedError(`/api/users/${id}`) };
    if (!res.ok) return { ok: false, error: new NetworkError('Fetch failed', res.status, res.url) };
    return { ok: true, value: await res.json() };
  } catch (err) {
    return { ok: false, error: new NetworkError('Network unavailable', 0, `/api/users/${id}`, { cause: err }) };
  }
}

// Caller — no try/catch, pattern matching on discriminant
const result = await safeGetUser('42');
if (!result.ok) {
  if (result.error instanceof UnauthorizedError) {
    router.navigate('/login');
  } else if (result.error instanceof NotFoundError) {
    showNotFound();
  } else {
    showGenericError(result.error.message);
  }
  return;
}
// TypeScript knows result.value is User here
renderUser(result.value);
```

---

## Catching Specific Errors

```typescript
try {
  await fetchUser(id);
} catch (err) {
  // Narrowing by class
  if (err instanceof UnauthorizedError) {
    redirectToLogin();
  } else if (err instanceof NotFoundError) {
    showNotFound();
  } else if (err instanceof NetworkError) {
    showNetworkError(err.status);
  } else if (err instanceof AppError) {
    reportToSentry(err);
    showGenericError(err.message);
  } else {
    // Unexpected / programmer error — rethrow
    throw err;
  }
}
```

---

## Serialization for Logging

```typescript
class SerializableError extends AppError {
  toJSON() {
    return {
      name:    this.name,
      message: this.message,
      code:    this.code,
      stack:   this.stack,
      cause:   this.cause instanceof Error ? (this.cause as any).toJSON?.() : String(this.cause),
    };
  }
}

// Sentry / logging service
function reportError(err: unknown): void {
  const payload = err instanceof SerializableError
    ? err.toJSON()
    : { name: 'UnknownError', message: String(err) };

  logger.error(payload);
}
```

---

## Common Pitfalls

1. **Forgetting `this.name = this.constructor.name`** — `err.name` shows `'Error'` instead of your class name, breaking log filtering.
2. **Forgetting `Object.setPrototypeOf`** — `instanceof` fails for subclasses in transpiled code.
3. **Swallowing the `cause`** — When re-throwing with context, always pass `{ cause: originalError }` so the stack trace chain is preserved.
4. **Catching too broadly** — `catch (err) { ... }` without re-throwing unexpected errors hides programmer mistakes.
5. **Custom errors not extending `Error`** — Some tools (Sentry, `instanceof Error`) won't recognize them correctly.

---

## Interview Questions

**Q: Why extend `Error` instead of creating plain objects for errors?**  
A: `Error` subclasses automatically capture a stack trace, work with `instanceof`, integrate with browser DevTools error highlighting, and are expected by error tracking tools like Sentry. Plain objects lose stack traces and can't be caught with `instanceof`.

**Q: What is `error.cause` and when should you use it?**  
A: ES2022 introduced an `options.cause` field for `Error` that lets you chain errors — wrap a low-level error with context without losing the original. Use it when re-throwing: `throw new AppError('User load failed', { cause: originalNetworkError })`.

**Q: What's the difference between `AggregateError` and a custom error holding an array?**  
A: `AggregateError` is built-in and thrown by `Promise.any()`, so catching it is idiomatic. It has `.errors` as a standard property. A custom error holding an array is fine but adds a non-standard API. For validation errors where `Promise.any` semantics don't apply, a custom class with `.errors: ValidationError[]` is clearer.
