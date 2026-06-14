# Login / Register Form with Server Errors

## The Idea

**In plain English:** You have a login and registration form. The user fills in fields, submits, and one of three things happens: the request succeeds and you redirect them; the request fails with a generic server error ("Invalid credentials"); or it fails with field-level errors where the server tells you exactly which field is wrong ("That email is already registered", "Password must be at least 8 characters"). Your form must map those server responses back onto the correct field, show a loading/disabled state while the request is in-flight, let the user reveal their password, and after a successful login redirect to the page they originally tried to visit — not just `/dashboard`.

**Real-world analogy:** Think of a hotel check-in desk.

- **Client-side validation** = the receptionist glancing at your booking printout before touching the computer. They spot instantly that you forgot to write your name. They tell you right there, no server round-trip.
- **Submission loading state** = the receptionist typing into the system while you stand there. The "Submit" button is the receptionist — they're busy, they can't take another request yet.
- **Field-level server errors** = the computer system responding "That room is already taken by another guest." The receptionist writes the error directly on the field for "room number" — not a generic notice stuck to the whole desk.
- **Password reveal toggle** = turning the printout face-up so you can confirm your booking reference is correct before handing it over.
- **Return URL redirect** = you originally wanted Room 212. After fixing the booking conflict, the receptionist takes you to Room 212 — not back to the lobby.

The key insight: client-side validation (schema) and server-side validation (API errors) are two separate error lifecycles. Your form must manage both without confusing them.

---

## Learning Objectives

- Understand why client-side schema validation (Zod) does not replace server error handling
- Map field-level API errors back to specific form fields without coupling the form to the API shape
- Implement submission loading and disabled states that prevent double-submit
- Build a controlled password reveal toggle that does not disrupt accessibility
- Handle redirect-after-login with a `returnUrl` query parameter
- Know the difference between React Hook Form's `setError` and Zod's `refine` — when to use each
- Implement the same pattern in Angular reactive forms with `FormGroup`, `setErrors`, and Angular signals

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide inline validation message | ✅ via `:invalid` / `:user-invalid` | Only works for built-in constraints, not Zod or server errors |
| Disable the submit button while loading | ❌ | Requires JS to set `disabled` attribute dynamically |
| Map API error to a specific field | ❌ | Requires JS to inspect the response and call `setError` / `setErrors` |
| Toggle password input type between `text` and `password` | ❌ | `input[type]` cannot be changed by CSS |
| Redirect to `returnUrl` after success | ❌ | Navigation requires JS |
| Show spinner inside button during fetch | ⚠️ | CSS can style an existing spinner but JS must insert/remove it |
| Prevent double-submit on slow networks | ❌ | Requires JS to track in-flight request state |

**Conclusion:** CSS owns the visual layer (error colours, spinner animation, button appearance). JS/framework owns everything else: request lifecycle, error routing, field state, and navigation.

---

## HTML & CSS Foundation

### The Structure

```html
<form class="auth-form" novalidate>
  <h1 class="auth-form__title">Sign in</h1>

  <!-- Global / non-field error (e.g. "Invalid credentials") -->
  <div class="auth-form__error" role="alert" aria-live="assertive" hidden>
    <span class="auth-form__error-icon" aria-hidden="true">!</span>
    <span class="auth-form__error-text"></span>
  </div>

  <!-- Email field -->
  <div class="field">
    <label class="field__label" for="email">Email</label>
    <input
      class="field__input"
      id="email"
      type="email"
      name="email"
      autocomplete="email"
      aria-describedby="email-error"
      aria-invalid="false"
    />
    <span class="field__error" id="email-error" role="alert" aria-live="polite"></span>
  </div>

  <!-- Password field with reveal toggle -->
  <div class="field">
    <label class="field__label" for="password">Password</label>
    <div class="field__input-wrapper">
      <input
        class="field__input"
        id="password"
        type="password"
        name="password"
        autocomplete="current-password"
        aria-describedby="password-error"
        aria-invalid="false"
      />
      <button
        type="button"
        class="field__reveal-btn"
        aria-label="Show password"
        aria-pressed="false"
      >
        <!-- SVG eye icon toggled by JS -->
        <svg class="icon icon--eye" aria-hidden="true" focusable="false">...</svg>
      </button>
    </div>
    <span class="field__error" id="password-error" role="alert" aria-live="polite"></span>
  </div>

  <!-- Submit -->
  <button class="btn btn--primary btn--full" type="submit" aria-describedby="submit-status">
    <span class="btn__label">Sign in</span>
    <span class="btn__spinner" aria-hidden="true" hidden></span>
  </button>
  <span class="sr-only" id="submit-status" aria-live="polite"></span>
</form>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --color-error:   #d32f2f;
  --color-error-bg: #fff5f5;
  --color-border:  #ccc;
  --color-border-focus: #1a73e8;
  --color-border-error: #d32f2f;
  --radius: 6px;
  --transition: 0.15s ease;
}

/* ─── Form shell ─── */
.auth-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  max-width: 400px;
  margin: 0 auto;
  padding: 2rem;
}

/* ─── Global error banner ─── */
.auth-form__error {
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  background: var(--color-error-bg);
  border: 1px solid var(--color-error);
  border-radius: var(--radius);
  color: var(--color-error);
  font-size: 0.875rem;
}

/* ─── Field group ─── */
.field {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.field__label {
  font-size: 0.875rem;
  font-weight: 600;
}

/* Password wrapper: relative so the reveal button can be positioned inside */
.field__input-wrapper {
  position: relative;
  display: flex;
}

.field__input {
  width: 100%;
  padding: 0.625rem 0.75rem;
  border: 1.5px solid var(--color-border);
  border-radius: var(--radius);
  font-size: 1rem;
  transition: border-color var(--transition), box-shadow var(--transition);
}

/* Extra right padding to avoid text running under the reveal button */
.field__input-wrapper .field__input {
  padding-right: 2.75rem;
}

.field__input:focus {
  outline: none;
  border-color: var(--color-border-focus);
  box-shadow: 0 0 0 3px rgba(26, 115, 232, 0.2);
}

/* Error state — driven by aria-invalid="true" (not a class) */
.field__input[aria-invalid="true"] {
  border-color: var(--color-border-error);
}

.field__input[aria-invalid="true"]:focus {
  box-shadow: 0 0 0 3px rgba(211, 47, 47, 0.2);
}

/* ─── Inline field error ─── */
.field__error {
  font-size: 0.8125rem;
  color: var(--color-error);
  min-height: 1.2em; /* prevents layout shift when error appears */
}

/* ─── Password reveal button ─── */
.field__reveal-btn {
  position: absolute;
  right: 0;
  top: 0;
  height: 100%;
  width: 2.75rem;
  display: flex;
  align-items: center;
  justify-content: center;
  background: none;
  border: none;
  cursor: pointer;
  color: #666;
  border-radius: 0 var(--radius) var(--radius) 0;
  transition: color var(--transition);
}

.field__reveal-btn:hover  { color: #333; }
.field__reveal-btn:focus-visible {
  outline: 2px solid var(--color-border-focus);
  outline-offset: -2px;
}

/* ─── Submit button ─── */
.btn--primary {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 0.75rem 1.5rem;
  background: #1a73e8;
  color: #fff;
  border: none;
  border-radius: var(--radius);
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background var(--transition), opacity var(--transition);
}

.btn--full { width: 100%; }

.btn--primary:hover:not(:disabled) { background: #1558b0; }

.btn--primary:disabled {
  opacity: 0.65;
  cursor: not-allowed;
}

/* ─── Spinner inside button ─── */
.btn__spinner {
  width: 1em;
  height: 1em;
  border: 2px solid rgba(255,255,255,0.4);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }

@media (prefers-reduced-motion: reduce) {
  .btn__spinner { animation: none; opacity: 0.7; }
}
```

**What CSS owns:** error colour scheme, `aria-invalid` border style (no extra class needed), minimum-height on error spans to prevent layout shift, reveal button positioning, spinner animation.

**What CSS cannot own:** which fields are invalid, mapping API errors to fields, toggling `input[type]`, disabling the button, redirecting after login.

---

## React Implementation

### Schema and Types

```tsx
// auth.schema.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email:    z.string().email('Enter a valid email address'),
  password: z.string().min(1, 'Password is required'),
});

export const registerSchema = z.object({
  email:    z.string().email('Enter a valid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter'),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],  // attach error to confirmPassword field
});

export type LoginFormValues    = z.infer<typeof loginSchema>;
export type RegisterFormValues = z.infer<typeof registerSchema>;

// API error shape returned by the server
export interface ApiFieldError {
  field: string;   // matches form field name
  message: string;
}
export interface ApiError {
  message: string;                // global error
  fieldErrors?: ApiFieldError[];  // field-level errors
}
```

### The Login Form Component

```tsx
// LoginForm.tsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useNavigate, useSearchParams } from 'react-router-dom';
import { loginSchema, LoginFormValues, ApiError } from './auth.schema';
import { EyeIcon, EyeOffIcon } from './icons';

async function loginRequest(values: LoginFormValues): Promise<void> {
  const res = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(values),
  });
  if (!res.ok) {
    const err: ApiError = await res.json();
    throw err; // throw the structured API error
  }
}

export function LoginForm() {
  const navigate            = useNavigate();
  const [searchParams]      = useSearchParams();
  const returnUrl           = searchParams.get('returnUrl') ?? '/dashboard';

  const [showPassword, setShowPassword] = useState(false);
  const [globalError, setGlobalError]   = useState<string | null>(null);

  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (values: LoginFormValues) => {
    setGlobalError(null); // clear previous global error on each attempt

    try {
      await loginRequest(values);
      // Success: navigate to the originally requested URL
      navigate(returnUrl, { replace: true });

    } catch (err) {
      const apiErr = err as ApiError;

      // Map field-level errors returned by the server onto their fields
      if (apiErr.fieldErrors?.length) {
        apiErr.fieldErrors.forEach(({ field, message }) => {
          setError(
            field as keyof LoginFormValues,
            { type: 'server', message },
            { shouldFocus: true }, // focuses the first invalid field
          );
        });
      } else {
        // Fall back to global error banner
        setGlobalError(apiErr.message ?? 'Something went wrong. Try again.');
      }
    }
  };

  return (
    <form className="auth-form" onSubmit={handleSubmit(onSubmit)} noValidate>
      <h1 className="auth-form__title">Sign in</h1>

      {/* Global error banner */}
      {globalError && (
        <div className="auth-form__error" role="alert">
          <span className="auth-form__error-icon" aria-hidden="true">!</span>
          <span className="auth-form__error-text">{globalError}</span>
        </div>
      )}

      {/* Email */}
      <div className="field">
        <label className="field__label" htmlFor="email">Email</label>
        <input
          className="field__input"
          id="email"
          type="email"
          autoComplete="email"
          aria-describedby="email-error"
          aria-invalid={!!errors.email}
          {...register('email')}
        />
        <span className="field__error" id="email-error" role="alert">
          {errors.email?.message}
        </span>
      </div>

      {/* Password */}
      <div className="field">
        <label className="field__label" htmlFor="password">Password</label>
        <div className="field__input-wrapper">
          <input
            className="field__input"
            id="password"
            type={showPassword ? 'text' : 'password'}
            autoComplete="current-password"
            aria-describedby="password-error"
            aria-invalid={!!errors.password}
            {...register('password')}
          />
          <button
            type="button"
            className="field__reveal-btn"
            aria-label={showPassword ? 'Hide password' : 'Show password'}
            aria-pressed={showPassword}
            onClick={() => setShowPassword(v => !v)}
          >
            {showPassword ? <EyeOffIcon aria-hidden /> : <EyeIcon aria-hidden />}
          </button>
        </div>
        <span className="field__error" id="password-error" role="alert">
          {errors.password?.message}
        </span>
      </div>

      {/* Submit */}
      <button
        className="btn btn--primary btn--full"
        type="submit"
        disabled={isSubmitting}
        aria-describedby="submit-status"
      >
        {isSubmitting ? (
          <>
            <span className="btn__spinner" aria-hidden="true" />
            <span>Signing in…</span>
          </>
        ) : (
          'Sign in'
        )}
      </button>
      <span className="sr-only" id="submit-status" aria-live="polite">
        {isSubmitting ? 'Signing in, please wait.' : ''}
      </span>
    </form>
  );
}
```

### Key Design Decisions

```tsx
// WHY setError, not re-running the schema
//
// Zod runs synchronously against local state. Server errors are async
// and carry information the schema cannot know (e.g. "that email is taken").
// React Hook Form's setError() injects errors into the same `errors` object
// that the schema populates — so the UI treats them identically.

// WHY { shouldFocus: true } on the first setError call
//
// After a server round-trip the user has lost their context.
// shouldFocus moves the cursor to the first broken field automatically.
// RHF applies this only to the first call in a loop — subsequent setError
// calls in the forEach do NOT steal focus back.

// WHY navigate(returnUrl, { replace: true })
//
// replace:true removes the login page from history. After login the user
// cannot hit the browser back button and land on the login page again —
// which would be confusing and potentially re-trigger form state.
```

---

## Angular Implementation

### Schema and Service

```typescript
// auth.model.ts
export interface LoginPayload    { email: string; password: string; }
export interface RegisterPayload { email: string; password: string; confirmPassword: string; }

export interface ApiFieldError  { field: string; message: string; }
export interface ApiErrorResponse {
  message: string;
  fieldErrors?: ApiFieldError[];
}
```

```typescript
// auth.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, catchError, throwError } from 'rxjs';
import { LoginPayload, ApiErrorResponse } from './auth.model';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);

  login(payload: LoginPayload): Observable<void> {
    return this.http.post<void>('/api/auth/login', payload).pipe(
      catchError((res: HttpErrorResponse) =>
        throwError(() => res.error as ApiErrorResponse)
      )
    );
  }
}
```

### Login Form Component

```typescript
// login-form.component.ts
import {
  Component, inject, signal, ChangeDetectionStrategy
} from '@angular/core';
import {
  FormBuilder, Validators, ReactiveFormsModule, AbstractControl
} from '@angular/forms';
import { Router, ActivatedRoute } from '@angular/router';
import { NgIf }   from '@angular/common';
import { AuthService }       from './auth.service';
import { ApiErrorResponse }  from './auth.model';

// Custom validator: confirm password matches password
function passwordMatch(group: AbstractControl) {
  const pw  = group.get('password')?.value;
  const cpw = group.get('confirmPassword')?.value;
  return pw === cpw ? null : { passwordMismatch: true };
}

@Component({
  selector: 'app-login-form',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule, NgIf],
  template: `
    <form class="auth-form" [formGroup]="form" (ngSubmit)="submit()" novalidate>
      <h1 class="auth-form__title">Sign in</h1>

      <!-- Global error banner -->
      <div
        *ngIf="globalError()"
        class="auth-form__error"
        role="alert"
      >
        <span class="auth-form__error-icon" aria-hidden="true">!</span>
        <span class="auth-form__error-text">{{ globalError() }}</span>
      </div>

      <!-- Email -->
      <div class="field">
        <label class="field__label" for="email">Email</label>
        <input
          class="field__input"
          id="email"
          type="email"
          formControlName="email"
          autocomplete="email"
          aria-describedby="email-error"
          [attr.aria-invalid]="isInvalid('email')"
        />
        <span class="field__error" id="email-error" role="alert" aria-live="polite">
          {{ getError('email') }}
        </span>
      </div>

      <!-- Password -->
      <div class="field">
        <label class="field__label" for="password">Password</label>
        <div class="field__input-wrapper">
          <input
            class="field__input"
            id="password"
            [type]="showPassword() ? 'text' : 'password'"
            formControlName="password"
            autocomplete="current-password"
            aria-describedby="password-error"
            [attr.aria-invalid]="isInvalid('password')"
          />
          <button
            type="button"
            class="field__reveal-btn"
            [attr.aria-label]="showPassword() ? 'Hide password' : 'Show password'"
            [attr.aria-pressed]="showPassword()"
            (click)="showPassword.update(v => !v)"
          >
            <!-- swap SVG based on showPassword() -->
          </button>
        </div>
        <span class="field__error" id="password-error" role="alert" aria-live="polite">
          {{ getError('password') }}
        </span>
      </div>

      <!-- Submit -->
      <button
        class="btn btn--primary btn--full"
        type="submit"
        [disabled]="isSubmitting()"
        aria-describedby="submit-status"
      >
        <span *ngIf="isSubmitting()" class="btn__spinner" aria-hidden="true"></span>
        <span>{{ isSubmitting() ? 'Signing in…' : 'Sign in' }}</span>
      </button>
      <span class="sr-only" id="submit-status" aria-live="polite">
        {{ isSubmitting() ? 'Signing in, please wait.' : '' }}
      </span>
    </form>
  `,
})
export class LoginFormComponent {
  private fb      = inject(FormBuilder);
  private auth    = inject(AuthService);
  private router  = inject(Router);
  private route   = inject(ActivatedRoute);

  showPassword = signal(false);
  isSubmitting = signal(false);
  globalError  = signal<string | null>(null);

  form = this.fb.group({
    email:    ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required]],
  });

  isInvalid(field: string): boolean {
    const ctrl = this.form.get(field);
    return !!(ctrl?.invalid && (ctrl.dirty || ctrl.touched));
  }

  getError(field: string): string {
    const ctrl = this.form.get(field);
    if (!ctrl?.errors) return '';
    if (ctrl.errors['required'])      return 'This field is required.';
    if (ctrl.errors['email'])         return 'Enter a valid email address.';
    if (ctrl.errors['minlength'])     return `Minimum ${ctrl.errors['minlength'].requiredLength} characters.`;
    if (ctrl.errors['server'])        return ctrl.errors['server'];  // injected from API
    return '';
  }

  submit() {
    this.form.markAllAsTouched();
    if (this.form.invalid) return;

    this.globalError.set(null);
    this.isSubmitting.set(true);

    const returnUrl = this.route.snapshot.queryParamMap.get('returnUrl') ?? '/dashboard';

    this.auth.login(this.form.getRawValue()).subscribe({
      next: () => {
        this.router.navigateByUrl(returnUrl, { replaceUrl: true });
      },
      error: (err: ApiErrorResponse) => {
        this.isSubmitting.set(false);

        if (err.fieldErrors?.length) {
          // Map each server field error into the matching form control
          err.fieldErrors.forEach(({ field, message }) => {
            this.form.get(field)?.setErrors({ server: message });
          });
          // Focus the first errored field
          const firstField = err.fieldErrors[0].field;
          document.getElementById(firstField)?.focus();
        } else {
          this.globalError.set(err.message ?? 'Something went wrong. Try again.');
        }
      },
    });
  }
}
```

### Why `setErrors({ server: message })` and not a new `FormControl`

```typescript
// Angular's setErrors() merges into the existing errors object.
// The control becomes invalid, which:
//   1. triggers [attr.aria-invalid]="isInvalid('email')" → "true"
//   2. makes getError('email') return the server message
//   3. prevents form submission until the user edits the field
//
// IMPORTANT: setErrors() is wiped on the next valueChanges emission,
// so the server error clears automatically as soon as the user edits
// the field — no extra cleanup needed.
//
// Contrast with adding a validator: validators persist across resubmissions.
// A server error from attempt #1 should not block attempt #2 if the field changed.
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Field errors linked to inputs | `aria-describedby="field-id-error"` on every input |
| Error state conveyed without colour alone | `aria-invalid="true"` on the input (border change + ARIA state) |
| Field errors announced on submission | `role="alert"` on each error span triggers announcement when text content changes |
| Global error banner announced immediately | `role="alert"` + `aria-live="assertive"` announces without user action |
| Submit button announces loading state | `aria-live="polite"` span with id `submit-status` updates text; `aria-describedby` links it |
| Submit disabled during in-flight request | `disabled` attribute; screen reader reads "button, dimmed" |
| Password reveal toggle is keyboard accessible | `type="button"` with `aria-pressed` (toggle button pattern) and `aria-label` |
| Reveal toggle label updates on state change | `aria-label` switches between "Show password" / "Hide password" |
| Focus moves to first server-errored field | `{ shouldFocus: true }` in RHF / explicit `focus()` call in Angular |
| Form uses `novalidate` attribute | Disables native browser validation UI (which we replace with our own) |
| Error messages are plain language | Messages explain what to do, not just what is wrong |

---

## Production Pitfalls

**1. Server errors survive resubmission in Angular**
`setErrors({ server: ... })` clears on `valueChanges`, but if the user submits again without changing the field, the control is valid and the server error is gone before the new request fires. Guard with `form.markAllAsTouched()` after setting server errors, and re-validate on submit regardless of prior state.

**2. `returnUrl` open redirect vulnerability**
Never redirect to an arbitrary string from the query param. Validate that the return URL is an internal relative path before navigating:
```typescript
const isSafeReturnUrl = (url: string) =>
  url.startsWith('/') && !url.startsWith('//');
const safe = isSafeReturnUrl(returnUrl) ? returnUrl : '/dashboard';
navigate(safe, { replace: true });
```
An attacker can craft `?returnUrl=https://evil.com` and phish credentials.

**3. Double-submit on slow connections**
`isSubmitting` state prevents a second click, but HTTP requests can also be triggered by keyboard Enter. Always disable the button (not just set a flag) to cover all interaction paths. RHF's `isSubmitting` already gates `handleSubmit`, but the button's `disabled` prop ensures the OS-level submit event is also blocked.

**4. Password reveal exposes the value in browser autofill**
When you switch `type="password"` to `type="text"`, some password managers re-offer to save the plain-text input separately. Use `autocomplete="current-password"` (never `off`) so the browser handles the credential correctly regardless of the visible input type.

**5. Zod `refine` errors appear on the wrong field**
A cross-field validation like "passwords must match" attaches to the root of the schema by default. You must specify `path: ['confirmPassword']` inside `refine()` to attach the error to the correct field. Without `path`, `errors.confirmPassword` is undefined and the error is silently swallowed.

**6. `aria-live` regions must exist in the DOM before the error fires**
If you conditionally render the error span (`{errors.email && <span>...`}`) the region does not exist when the browser registers it for live announcements. Render the span unconditionally with empty text content; the browser watches the region from page load and announces when text is injected. This is the `min-height` trick in the CSS — it also prevents layout shift.

**7. RHF `setError` with `shouldFocus: true` only focuses the first call in a loop**
When iterating `fieldErrors`, pass `{ shouldFocus: true }` only on the first iteration, or sort the errors by field DOM order and always focus the one that appears first in the document. Focusing the last-set field leaves users confused about which field is actually first.

---

## Interview Angle

**Q: "How do you handle server-returned field-level errors in a form managed by React Hook Form and Zod?"**

Zod and the server are two separate validation layers. Zod runs synchronously at submit time against local state — it catches format errors ("not an email") before the network. The server catches semantic errors that require database knowledge ("that email is already taken"). After a failed request, parse the API response and call `setError(fieldName, { type: 'server', message })` for each `fieldErrors` entry. RHF merges these into the same `errors` object that Zod populates, so the template code is identical for both error sources. Use `{ shouldFocus: true }` on the first `setError` call to move focus to the first invalid field. For errors that are not field-specific ("Invalid credentials"), set a separate piece of local state and render a `role="alert"` banner — do not force a global error into an arbitrary field.

**Follow-up: "How do you prevent the server error from persisting after the user corrects their input and resubmits?"**

`setError` in RHF does not automatically clear — but the error is attached to the field's `errors` object, and as soon as the user next passes client-side Zod validation for that field, the schema-based errors overwrite it. For fields that are not validated by Zod (unlikely but possible), call `clearErrors(field)` in an `onChange` handler. In Angular, `setErrors({ server: message })` is automatically cleared by `valueChanges` — the moment the user types, the control's errors are reset and re-evaluated from validators only. This means the Angular approach requires no cleanup code, but also means you must re-apply the error on every failed submission attempt, not cache it.