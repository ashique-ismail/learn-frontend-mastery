# Login Form Accessibility Deep Dive

## The Idea

**In plain English:** A login form looks simple — two inputs and a button — but it is one of the most accessibility-dense components on any site. Every user who cannot log in because of an inaccessible error message, a focus jump that lands in the wrong place, or a password field that cannot be revealed is a user you have locked out. Getting it right means wiring ARIA attributes so screen readers narrate errors automatically, moving focus deliberately after a failed submission, surfacing async server errors without page reloads, and building a password-reveal toggle that communicates its state correctly.

**Real-world analogy:** Imagine a bank teller window with a thick glass partition and an intercom.

- The **form fields** are the forms you slide under the glass — the teller cannot help you until the right information is on the paper.
- An **inline validation error** is the teller pushing the form back and circling the missing field in red marker. Without the intercom (ARIA), a blind customer cannot hear what was circled.
- **`aria-describedby`** is the intercom that lets the teller read the annotation aloud the moment the customer picks up the form.
- **Focus management after a failed submit** is the teller handing the pen back and pointing to the first wrong field — not leaving the customer to search for it.
- **`aria-live` for server errors** is an announcement over the branch PA system: "There is an issue with your account — please see the teller." It fires without the customer having to ask.
- **The password reveal toggle** is a frosted-glass window: you can choose to make it clear or opaque, and the teller announces "you are now showing your password" so the customer knows the state has changed.

---

## Learning Objectives

- Understand why `aria-describedby` is the correct ARIA primitive for linking inputs to error messages (not `aria-errormessage` alone)
- Apply `aria-invalid` correctly on fields that fail validation and remove it when the field recovers
- Implement deliberate focus management after a failed form submission — move to the first invalid field
- Build an `aria-live` region that announces async server errors without a page reload or intrusive alert
- Create an accessible password-reveal toggle that announces its state and does not confuse screen readers
- Avoid the most common production mistakes: premature validation, live-region timing, and toggle labeling

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Style an error state on an input | ✅ via `:invalid` pseudo-class | — |
| Show/hide an error message | ✅ via sibling selectors + `:invalid` | — |
| Announce the error message to a screen reader | ❌ | Requires `aria-describedby` pointing to the message element |
| Mark a field as currently invalid | ❌ | `aria-invalid` must be toggled by JS; `:invalid` is CSS-only and fires too eagerly |
| Move focus to the first invalid field after submit | ❌ | Focus management is imperative — requires `element.focus()` |
| Surface an async server error without a reload | ❌ | Requires an `aria-live` region updated by JS |
| Toggle password visibility and announce the state | ❌ | Input `type` change requires JS; announcing new state requires ARIA |

**Conclusion:** CSS owns visual feedback (red border, error icon, visible error text). JS owns every semantic layer: ARIA state, focus placement, and live announcements.

---

## HTML & CSS Foundation

### The Structure

```html
<form id="login-form" novalidate aria-label="Sign in to your account">

  <!-- Email field with linked error -->
  <div class="field-group">
    <label for="email">Email address</label>
    <input
      id="email"
      type="email"
      name="email"
      autocomplete="email"
      aria-required="true"
      aria-invalid="false"
      aria-describedby="email-error"
    />
    <!-- Error container: empty when valid, populated on error -->
    <span id="email-error" class="field-error" role="alert" aria-live="assertive"></span>
  </div>

  <!-- Password field with linked error and reveal toggle -->
  <div class="field-group">
    <label for="password">Password</label>
    <div class="password-wrapper">
      <input
        id="password"
        type="password"
        name="password"
        autocomplete="current-password"
        aria-required="true"
        aria-invalid="false"
        aria-describedby="password-error"
      />
      <button
        type="button"
        class="password-reveal"
        aria-label="Show password"
        aria-pressed="false"
        aria-controls="password"
      >
        <!-- Icon swapped by JS; hidden from AT since label carries the meaning -->
        <span class="reveal-icon" aria-hidden="true">👁</span>
      </button>
    </div>
    <span id="password-error" class="field-error" role="alert" aria-live="assertive"></span>
  </div>

  <!-- Async server error: injected after a failed API call -->
  <div
    id="server-error"
    class="server-error"
    role="alert"
    aria-live="assertive"
    aria-atomic="true"
  ></div>

  <button type="submit" class="submit-btn">Sign in</button>

</form>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --color-error:    #c0392b;
  --color-error-bg: #fdf3f2;
  --color-border:   #767676;
  --color-focus:    #0066cc;
  --input-radius:   4px;
  --transition:     0.15s ease;
}

/* ─── Field group layout ─── */
.field-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin-bottom: 1.25rem;
}

label {
  font-size: 0.9rem;
  font-weight: 600;
  color: #222;
}

/* ─── Input base ─── */
input[type="email"],
input[type="password"],
input[type="text"] {   /* text: when password is revealed */
  width: 100%;
  padding: 0.6rem 0.75rem;
  border: 2px solid var(--color-border);
  border-radius: var(--input-radius);
  font-size: 1rem;
  transition: border-color var(--transition), box-shadow var(--transition);
  box-sizing: border-box;
}

input:focus {
  outline: none;
  border-color: var(--color-focus);
  box-shadow: 0 0 0 3px rgba(0, 102, 204, 0.25);
}

/* Error state — driven by aria-invalid, not just :invalid */
input[aria-invalid="true"] {
  border-color: var(--color-error);
  background: var(--color-error-bg);
}

input[aria-invalid="true"]:focus {
  box-shadow: 0 0 0 3px rgba(192, 57, 43, 0.25);
}

/* ─── Inline error message ─── */
.field-error {
  font-size: 0.85rem;
  color: var(--color-error);
  min-height: 1.2em;   /* reserve space so layout doesn't shift on error */
}

.field-error:empty {
  display: none;       /* collapse when no error — avoids gap */
}

/* ─── Password wrapper ─── */
.password-wrapper {
  position: relative;
  display: flex;
  align-items: stretch;
}

.password-wrapper input {
  padding-right: 3rem;  /* make room for the reveal button */
  flex: 1;
}

/* ─── Password reveal button ─── */
.password-reveal {
  position: absolute;
  right: 0;
  top: 0;
  bottom: 0;
  width: 2.75rem;
  background: none;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 0 var(--input-radius) var(--input-radius) 0;
  color: #555;
  transition: color var(--transition);
}

.password-reveal:hover { color: #111; }

.password-reveal:focus-visible {
  outline: 2px solid var(--color-focus);
  outline-offset: -2px;
}

/* Visual indicator when password is shown */
.password-reveal[aria-pressed="true"] {
  color: var(--color-focus);
}

/* ─── Server error banner ─── */
.server-error {
  padding: 0.75rem 1rem;
  border-radius: var(--input-radius);
  background: var(--color-error-bg);
  border-left: 4px solid var(--color-error);
  color: var(--color-error);
  font-size: 0.9rem;
  margin-bottom: 1rem;
}

.server-error:empty {
  display: none;
}

/* ─── Submit button ─── */
.submit-btn {
  width: 100%;
  padding: 0.75rem;
  background: #0066cc;
  color: #fff;
  border: none;
  border-radius: var(--input-radius);
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background var(--transition);
}

.submit-btn:hover   { background: #0052a3; }
.submit-btn:focus-visible {
  outline: 3px solid var(--color-focus);
  outline-offset: 2px;
}
.submit-btn:disabled {
  background: #aaa;
  cursor: not-allowed;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  * { transition: none !important; }
}
```

**What CSS owns:** error border color, focus ring, password-reveal hover state, server error banner shape, layout stability via `min-height` on the error span.

**What CSS cannot own:** ARIA attributes, focus placement, live announcements, or the `type` switch on the password input.

---

## React Implementation

### Types and Validation Logic

```tsx
// loginForm.types.ts
export interface LoginFormState {
  email: string;
  password: string;
}

export interface FieldErrors {
  email?: string;
  password?: string;
}

export interface ServerError {
  message: string;
}

// Pure validation — no side effects, fully testable
export function validateLoginForm(values: LoginFormState): FieldErrors {
  const errors: FieldErrors = {};

  if (!values.email.trim()) {
    errors.email = 'Email address is required.';
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(values.email)) {
    errors.email = 'Enter a valid email address (e.g. name@example.com).';
  }

  if (!values.password) {
    errors.password = 'Password is required.';
  } else if (values.password.length < 8) {
    errors.password = 'Password must be at least 8 characters.';
  }

  return errors;
}
```

### The Form Hook

```tsx
// useLoginForm.ts
import { useState, useRef, useCallback } from 'react';
import type { LoginFormState, FieldErrors, ServerError } from './loginForm.types';
import { validateLoginForm } from './loginForm.types';

interface UseLoginFormReturn {
  values: LoginFormState;
  errors: FieldErrors;
  serverError: string;
  isSubmitting: boolean;
  showPassword: boolean;
  emailRef: React.RefObject<HTMLInputElement>;
  passwordRef: React.RefObject<HTMLInputElement>;
  handleChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
  handleSubmit: (e: React.FormEvent<HTMLFormElement>) => Promise<void>;
  togglePasswordReveal: () => void;
}

// Simulated API call — replace with your actual fetch/axios call
async function callLoginAPI(values: LoginFormState): Promise<void> {
  await new Promise(r => setTimeout(r, 800));
  if (values.email === 'locked@example.com') {
    throw new Error('Your account is locked. Contact support to unlock it.');
  }
  // success — redirect or update auth context here
}

export function useLoginForm(): UseLoginFormReturn {
  const [values, setValues]           = useState<LoginFormState>({ email: '', password: '' });
  const [errors, setErrors]           = useState<FieldErrors>({});
  const [serverError, setServerError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [showPassword, setShowPassword] = useState(false);

  // Refs for programmatic focus after failed submit
  const emailRef    = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setValues(prev => ({ ...prev, [name]: value }));

    // Clear the error for this field as the user corrects it
    setErrors(prev => ({ ...prev, [name]: undefined }));

    // Clear server error on any change — it is no longer relevant
    setServerError('');
  }, []);

  const handleSubmit = useCallback(async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setServerError('');

    const newErrors = validateLoginForm(values);
    setErrors(newErrors);

    const errorKeys = Object.keys(newErrors) as Array<keyof FieldErrors>;

    if (errorKeys.length > 0) {
      // Move focus to the first invalid field — do not leave it on the submit button
      const firstErrorField = errorKeys[0] === 'email' ? emailRef : passwordRef;
      firstErrorField.current?.focus();
      return;
    }

    setIsSubmitting(true);
    try {
      await callLoginAPI(values);
      // On success: redirect, update context, etc.
    } catch (err) {
      const message = err instanceof Error
        ? err.message
        : 'Something went wrong. Please try again.';
      setServerError(message);
      // After a server error, focus the email field so the user can correct credentials
      emailRef.current?.focus();
    } finally {
      setIsSubmitting(false);
    }
  }, [values]);

  const togglePasswordReveal = useCallback(() => {
    setShowPassword(prev => !prev);
    // Keep focus on the password input after the toggle
    passwordRef.current?.focus();
  }, []);

  return {
    values,
    errors,
    serverError,
    isSubmitting,
    showPassword,
    emailRef,
    passwordRef,
    handleChange,
    handleSubmit,
    togglePasswordReveal,
  };
}
```

### The Component

```tsx
// LoginForm.tsx
import React from 'react';
import { useLoginForm } from './useLoginForm';

export function LoginForm() {
  const {
    values,
    errors,
    serverError,
    isSubmitting,
    showPassword,
    emailRef,
    passwordRef,
    handleChange,
    handleSubmit,
    togglePasswordReveal,
  } = useLoginForm();

  return (
    <form
      id="login-form"
      onSubmit={handleSubmit}
      noValidate
      aria-label="Sign in to your account"
    >
      {/* ── Email ── */}
      <div className="field-group">
        <label htmlFor="email">Email address</label>
        <input
          ref={emailRef}
          id="email"
          type="email"
          name="email"
          value={values.email}
          onChange={handleChange}
          autoComplete="email"
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby="email-error"
          disabled={isSubmitting}
        />
        {/* role="alert" + aria-live fires the announcement when text is injected */}
        <span
          id="email-error"
          className="field-error"
          role="alert"
          aria-live="assertive"
        >
          {errors.email}
        </span>
      </div>

      {/* ── Password ── */}
      <div className="field-group">
        <label htmlFor="password">Password</label>
        <div className="password-wrapper">
          <input
            ref={passwordRef}
            id="password"
            type={showPassword ? 'text' : 'password'}
            name="password"
            value={values.password}
            onChange={handleChange}
            autoComplete="current-password"
            aria-required="true"
            aria-invalid={!!errors.password}
            aria-describedby="password-error"
            disabled={isSubmitting}
          />
          <button
            type="button"
            className="password-reveal"
            aria-label={showPassword ? 'Hide password' : 'Show password'}
            aria-pressed={showPassword}
            aria-controls="password"
            onClick={togglePasswordReveal}
            tabIndex={0}
          >
            <span className="reveal-icon" aria-hidden="true">
              {showPassword ? '🙈' : '👁'}
            </span>
          </button>
        </div>
        <span
          id="password-error"
          className="field-error"
          role="alert"
          aria-live="assertive"
        >
          {errors.password}
        </span>
      </div>

      {/* ── Server error — async, injected after API response ── */}
      <div
        id="server-error"
        className="server-error"
        role="alert"
        aria-live="assertive"
        aria-atomic="true"
      >
        {serverError}
      </div>

      <button
        type="submit"
        className="submit-btn"
        disabled={isSubmitting}
        aria-busy={isSubmitting}
      >
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </button>
    </form>
  );
}
```

---

## Angular Implementation

### Validation Helpers

```typescript
// login-form.validators.ts
import { AbstractControl, ValidationErrors } from '@angular/forms';

export function emailValidator(control: AbstractControl): ValidationErrors | null {
  const v = control.value as string;
  if (!v?.trim()) return { required: 'Email address is required.' };
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v)) {
    return { format: 'Enter a valid email address (e.g. name@example.com).' };
  }
  return null;
}

export function passwordValidator(control: AbstractControl): ValidationErrors | null {
  const v = control.value as string;
  if (!v) return { required: 'Password is required.' };
  if (v.length < 8) return { minlength: 'Password must be at least 8 characters.' };
  return null;
}

// Helper: extract the first error message string from a control
export function firstError(errors: ValidationErrors | null): string {
  if (!errors) return '';
  return Object.values(errors)[0] as string;
}
```

### Login Form Component

```typescript
// login-form.component.ts
import {
  Component, signal, computed, inject, OnInit, ElementRef, viewChild
} from '@angular/core';
import { ReactiveFormsModule, FormBuilder, Validators, FormGroup } from '@angular/forms';
import { NgIf, AsyncPipe } from '@angular/common';
import { emailValidator, passwordValidator, firstError } from './login-form.validators';

// Replace with your actual auth service
import { AuthService } from './auth.service';

@Component({
  selector: 'app-login-form',
  standalone: true,
  imports: [ReactiveFormsModule, NgIf],
  template: `
    <form
      [formGroup]="form"
      (ngSubmit)="onSubmit()"
      novalidate
      aria-label="Sign in to your account"
    >
      <!-- Email -->
      <div class="field-group">
        <label for="email">Email address</label>
        <input
          #emailInput
          id="email"
          type="email"
          formControlName="email"
          autocomplete="email"
          aria-required="true"
          [attr.aria-invalid]="emailInvalid()"
          aria-describedby="email-error"
        />
        <span
          id="email-error"
          class="field-error"
          role="alert"
          aria-live="assertive"
        >{{ emailError() }}</span>
      </div>

      <!-- Password -->
      <div class="field-group">
        <label for="password">Password</label>
        <div class="password-wrapper">
          <input
            #passwordInput
            id="password"
            [type]="showPassword() ? 'text' : 'password'"
            formControlName="password"
            autocomplete="current-password"
            aria-required="true"
            [attr.aria-invalid]="passwordInvalid()"
            aria-describedby="password-error"
          />
          <button
            type="button"
            class="password-reveal"
            [attr.aria-label]="showPassword() ? 'Hide password' : 'Show password'"
            [attr.aria-pressed]="showPassword()"
            aria-controls="password"
            (click)="toggleReveal()"
          >
            <span class="reveal-icon" aria-hidden="true">
              {{ showPassword() ? '🙈' : '👁' }}
            </span>
          </button>
        </div>
        <span
          id="password-error"
          class="field-error"
          role="alert"
          aria-live="assertive"
        >{{ passwordError() }}</span>
      </div>

      <!-- Server error -->
      <div
        id="server-error"
        class="server-error"
        role="alert"
        aria-live="assertive"
        aria-atomic="true"
      >{{ serverError() }}</div>

      <button
        type="submit"
        class="submit-btn"
        [disabled]="isSubmitting()"
        [attr.aria-busy]="isSubmitting()"
      >
        {{ isSubmitting() ? 'Signing in…' : 'Sign in' }}
      </button>
    </form>
  `,
})
export class LoginFormComponent implements OnInit {
  private fb      = inject(FormBuilder);
  private auth    = inject(AuthService);
  private el      = inject(ElementRef);

  // Signals
  showPassword = signal(false);
  isSubmitting = signal(false);
  serverError  = signal('');

  // Template refs for focus management
  emailInput    = viewChild<ElementRef<HTMLInputElement>>('emailInput');
  passwordInput = viewChild<ElementRef<HTMLInputElement>>('passwordInput');

  form!: FormGroup;

  ngOnInit() {
    this.form = this.fb.group({
      email:    ['', [emailValidator]],
      password: ['', [passwordValidator]],
    });

    // Clear server error whenever the user edits any field
    this.form.valueChanges.subscribe(() => this.serverError.set(''));
  }

  // Computed signals for template bindings
  emailInvalid = computed(() => {
    const ctrl = this.form?.get('email');
    return ctrl?.invalid && ctrl?.touched ? true : null;
  });

  passwordInvalid = computed(() => {
    const ctrl = this.form?.get('password');
    return ctrl?.invalid && ctrl?.touched ? true : null;
  });

  emailError = computed(() => {
    const ctrl = this.form?.get('email');
    if (!ctrl?.touched) return '';
    return firstError(ctrl.errors);
  });

  passwordError = computed(() => {
    const ctrl = this.form?.get('password');
    if (!ctrl?.touched) return '';
    return firstError(ctrl.errors);
  });

  toggleReveal() {
    this.showPassword.update(v => !v);
    // Return focus to the password input after state change
    this.passwordInput()?.nativeElement.focus();
  }

  async onSubmit() {
    // Mark all fields touched so errors render immediately
    this.form.markAllAsTouched();

    if (this.form.invalid) {
      // Move focus to the first invalid field
      if (this.form.get('email')?.invalid) {
        this.emailInput()?.nativeElement.focus();
      } else {
        this.passwordInput()?.nativeElement.focus();
      }
      return;
    }

    this.isSubmitting.set(true);
    this.serverError.set('');

    try {
      await this.auth.login(this.form.value);
      // On success: router.navigate(['/dashboard'])
    } catch (err: unknown) {
      const message = err instanceof Error
        ? err.message
        : 'Something went wrong. Please try again.';
      this.serverError.set(message);
      // Focus email so the user can correct credentials
      this.emailInput()?.nativeElement.focus();
    } finally {
      this.isSubmitting.set(false);
    }
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Each input is linked to its error message | `aria-describedby="<field>-error"` on input; matching `id` on the error span |
| Invalid fields are marked as such | `aria-invalid="true"` set when field has an error after validation |
| `aria-invalid` removed when field is corrected | Reset to `false` / `null` as the user types or re-validates |
| Error messages are announced automatically | Error spans carry `role="alert"` + `aria-live="assertive"` — injection fires announcement |
| Focus moves to first invalid field after submit | `element.focus()` called on `emailRef` / `emailInput()` before returning from submit handler |
| Server error announced without page reload | Dedicated `aria-live="assertive"` + `aria-atomic="true"` container updated by JS |
| Password reveal communicates its state | `aria-pressed` (toggle button pattern) + `aria-label` switches between "Show" and "Hide" |
| Password reveal keeps input focused | `passwordRef.current?.focus()` / `passwordInput().focus()` called after toggle |
| Submit button communicates loading state | `aria-busy="true"` + visible label change to "Signing in…" |
| Form is not validated on load | `novalidate` on `<form>`; validation only triggered by submit or blur |
| Fields have visible labels | `<label for="...">` — not placeholder-only labels |
| Minimum tap target on reveal button | `width: 2.75rem; height: 100%` via CSS |

---

## Production Pitfalls

**1. Injecting the error element itself is not enough — the live region must already exist in the DOM**
If you conditionally render the error span with `{errors.email && <span ...>}`, the live region does not exist when the user first loads the page. Screen readers only pick up text injected into a pre-existing live region. Fix: always render the error container (with empty text), and update its content. The CSS `display: none` on `:empty` keeps it invisible to sighted users without removing it from the accessibility tree.

**2. `aria-describedby` announces on focus, not on injection**
`aria-describedby` reads the linked element's content when the user focuses the input, not when the error appears. If you want the error spoken immediately on inject, the error span needs its own `role="alert"` or `aria-live` attribute in addition to being referenced by `aria-describedby`. Both mechanisms serve different purposes: `aria-describedby` gives the user context when they tab back to the field; `aria-live` gives them immediate feedback when the error appears.

**3. Using `role="alert"` without `aria-live` is unreliable across browsers**
`role="alert"` implies `aria-live="assertive"` by spec, but browser + screen reader combinations do not always honour this. Set both explicitly: `role="alert" aria-live="assertive"`. This costs nothing and avoids silent failures on VoiceOver + Safari.

**4. Firing `element.focus()` before the DOM updates causes a screen reader to read stale content**
In React, calling `focus()` inside a synchronous event handler runs before the new error message has been painted. Wrap the focus call in `requestAnimationFrame(() => ref.current?.focus())` if you observe this in testing, especially on older browser/AT combinations.

**5. The password reveal toggle must use `type="button"` to avoid accidental form submission**
Without `type="button"`, a `<button>` inside a `<form>` defaults to `type="submit"`. Clicking the reveal icon will submit the form before the user has finished typing. This is a silent failure — no error, no feedback, just an early submit.

**6. Changing `input[type]` from `password` to `text` resets autocomplete on some browsers**
Some password managers lose their fill position when the `type` attribute changes. Test your reveal toggle with at least one password manager (1Password, Bitwarden) to confirm that the filled value is preserved.

**7. Overly eager inline validation destroys usability**
Marking a field `aria-invalid` and announcing an error while the user is still typing is aggressive and disorienting. Validate on `blur` for empty/format errors and on `submit` for required errors. Only clear errors immediately (on `change`) — never add them on `change`.

---

## Interview Angle

**Q: "Walk me through how you'd make a login form accessible end to end."**

A complete answer covers four layers:
1. **Semantic HTML first** — `<label for>`, `novalidate` on the form, `autocomplete` attributes on inputs, and a `<button type="submit">`. None of this requires JS and all of it is load-bearing.
2. **ARIA linking** — every input carries `aria-describedby` pointing to its error container and `aria-invalid` that flips to `true` only after a failed validation attempt (not on load, not while typing). The error containers are pre-rendered in the DOM with `role="alert"` and `aria-live="assertive"` so injection is heard immediately.
3. **Deliberate focus management** — after a failed submit, programmatically focus the first invalid field with `element.focus()`. After a server error, focus the email field. After a password reveal toggle, return focus to the password input. Focus should never be lost or left stranded on a no-longer-relevant element.
4. **Async server errors** — a dedicated live region separate from the field errors, marked `aria-atomic="true"` so the full sentence is read, not just the changed portion. Clear it on any subsequent input change so it does not persist as stale information.

**Follow-up: "What's the difference between `aria-describedby`, `aria-errormessage`, and `role="alert"` — when do you use each?"**

`aria-describedby` creates a persistent association between an input and supplemental text (error messages, format hints, character counts). The linked text is read when the user focuses the input. `aria-errormessage` is a more specific ARIA 1.1 attribute intended only for error text, and it requires `aria-invalid="true"` to be active before the message is surfaced — support was historically inconsistent so `aria-describedby` remains the safer choice for broad compatibility. `role="alert"` (with `aria-live="assertive"`) fires an interruption announcement the moment text is injected into the container — it is the right tool for errors that appear asynchronously or immediately after an action. In a login form you typically use both: `aria-describedby` so the user hears the error again when they tab back to the field, and `role="alert"` on the error container so the error is announced the instant it appears.
