# Accessible Forms

## The Idea

**In plain English:** Accessible forms are web forms (like login screens or sign-up pages) designed so that everyone can use them — including people who are blind and use software that reads the screen aloud, or people who only use a keyboard instead of a mouse. This means every question on the form has a clear label, errors are explained in plain language, and the form works without needing to see it visually.

**Real-world analogy:** Imagine filling out a paper job application at a front desk, where a helpful assistant reads each question aloud and tells you exactly what you wrote wrong before you hand it in.
- The printed label next to each blank line = the `<label>` element that names each input field
- The assistant saying "This field is required" out loud = `aria-required` and `role="alert"` announcing errors to screen readers
- The grouping of questions under headings like "Work History" = `<fieldset>` and `<legend>` grouping related fields together

---

## Table of Contents
- [Introduction](#introduction)
- [Why Form Accessibility Matters](#why-form-accessibility-matters)
- [Form Labels](#form-labels)
- [Input Types and Attributes](#input-types-and-attributes)
- [Error Messages and Validation](#error-messages-and-validation)
- [Required Fields](#required-fields)
- [Fieldsets and Legends](#fieldsets-and-legends)
- [Help Text and Instructions](#help-text-and-instructions)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Forms are one of the most critical components of web applications, enabling users to log in, make purchases, submit data, and interact with services. However, forms are also one of the most common sources of accessibility barriers. Accessible forms ensure that all users, including those using assistive technologies, can successfully complete and submit form data.

```
Accessible Form Structure
=========================

Form
├─ Label (visible, associated)
├─ Input (semantic, typed)
├─ Help Text (aria-describedby)
├─ Error Message (aria-invalid + role="alert")
└─ Submit Button (descriptive)

Screen Reader Flow:
"Email, edit text, required
 Enter your email address
 [user types]
 Invalid: Must be a valid email address"
```

Statistics show that inaccessible forms can exclude up to 71% of users with disabilities from completing online tasks.

## Why Form Accessibility Matters

### Legal and Business Impact

- **Lawsuits**: Forms are the #1 target of ADA lawsuits
- **Lost Revenue**: Inaccessible checkout forms lose customers
- **WCAG Compliance**: Required for government, education, enterprise
- **Brand Reputation**: Accessibility affects user trust

### User Impact

```typescript
// Example: Impact on Different Users

/*
Blind Users (Screen Readers):
- Need proper labels to understand fields
- Rely on error announcements
- Can't see visual validation cues

Motor Impaired Users:
- Need large click targets
- Benefit from keyboard shortcuts
- Require clear focus indicators

Cognitive Disabilities:
- Need clear instructions
- Benefit from inline help
- Require clear error messages

Mobile Users:
- Need appropriate input types for keyboards
- Benefit from autocomplete
- Require touch-friendly targets
*/
```

## Form Labels

### Explicit Labels

```typescript
// React: Proper label association
const ExplicitLabels: React.FC = () => {
  return (
    <form>
      {/* ✅ CORRECT: Explicit label with htmlFor */}
      <div className="form-group">
        <label htmlFor="email">Email Address</label>
        <input id="email" type="email" name="email" required />
      </div>

      {/* ✅ CORRECT: Implicit label (wrapping) */}
      <div className="form-group">
        <label>
          Phone Number
          <input type="tel" name="phone" />
        </label>
      </div>

      {/* ❌ WRONG: No label */}
      <input type="text" placeholder="Username" />
      {/* Placeholder is not a substitute for a label! */}

      {/* ❌ WRONG: Adjacent text without association */}
      <div>
        Username
        <input type="text" name="username" />
      </div>
    </form>
  );
};
```

```typescript
// Angular: Form labels with FormControl
@Component({
  selector: 'app-labeled-form',
  template: `
    <form [formGroup]="form">
      <!-- Explicit label -->
      <div class="form-group">
        <label for="firstName">First Name *</label>
        <input
          id="firstName"
          type="text"
          formControlName="firstName"
          [attr.aria-required]="true"
          [attr.aria-invalid]="firstName.invalid && firstName.touched"
        />
      </div>

      <!-- Label with required indicator -->
      <div class="form-group">
        <label for="email">
          Email Address
          <span class="required" aria-label="required">*</span>
        </label>
        <input
          id="email"
          type="email"
          formControlName="email"
          [attr.aria-required]="true"
        />
      </div>

      <!-- Label with help text -->
      <div class="form-group">
        <label for="password">Password</label>
        <input
          id="password"
          type="password"
          formControlName="password"
          aria-describedby="password-help"
        />
        <div id="password-help" class="help-text">
          Must be at least 8 characters
        </div>
      </div>
    </form>
  `
})
export class LabeledFormComponent {
  form = this.fb.group({
    firstName: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]]
  });

  get firstName() { return this.form.get('firstName')!; }
  get email() { return this.form.get('email')!; }
  get password() { return this.form.get('password')!; }

  constructor(private fb: FormBuilder) {}
}
```

### Aria-Label and Aria-Labelledby

```typescript
// React: Alternative labeling techniques
const AlternativeLabels: React.FC = () => {
  return (
    <form>
      {/* aria-label: Invisible label */}
      <input
        type="search"
        aria-label="Search products"
        placeholder="Search..."
      />

      {/* aria-labelledby: Reference existing text */}
      <div>
        <h2 id="shipping-heading">Shipping Address</h2>
        <input
          type="text"
          aria-labelledby="shipping-heading"
          placeholder="Street address"
        />
      </div>

      {/* Multiple labelledby references */}
      <div>
        <span id="card-label">Credit Card</span>
        <span id="card-type">Visa ending in 1234</span>
        <input
          type="text"
          aria-labelledby="card-label card-type"
        />
      </div>

      {/* Icon button with aria-label */}
      <button type="button" aria-label="Close dialog">
        <span aria-hidden="true">×</span>
      </button>
    </form>
  );
};
```

## Input Types and Attributes

### Semantic Input Types

```typescript
// React: Using appropriate input types
const SemanticInputs: React.FC = () => {
  return (
    <form>
      {/* Email with validation */}
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"  // Triggers email keyboard on mobile
        autoComplete="email"
        required
      />

      {/* Tel with pattern */}
      <label htmlFor="phone">Phone</label>
      <input
        id="phone"
        type="tel"  // Triggers number keyboard on mobile
        autoComplete="tel"
        pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}"
        placeholder="123-456-7890"
      />

      {/* URL with validation */}
      <label htmlFor="website">Website</label>
      <input
        id="website"
        type="url"  // Triggers URL keyboard on mobile
        autoComplete="url"
        placeholder="https://example.com"
      />

      {/* Number with min/max */}
      <label htmlFor="quantity">Quantity</label>
      <input
        id="quantity"
        type="number"
        min="1"
        max="10"
        step="1"
        defaultValue="1"
      />

      {/* Date with constraints */}
      <label htmlFor="birthdate">Birth Date</label>
      <input
        id="birthdate"
        type="date"
        max={new Date().toISOString().split('T')[0]}  // No future dates
        autoComplete="bday"
      />

      {/* Time input */}
      <label htmlFor="appointment">Appointment Time</label>
      <input
        id="appointment"
        type="time"
        min="09:00"
        max="17:00"
      />

      {/* Color picker */}
      <label htmlFor="color">Favorite Color</label>
      <input
        id="color"
        type="color"
        defaultValue="#0066cc"
      />

      {/* File upload */}
      <label htmlFor="document">Upload Document</label>
      <input
        id="document"
        type="file"
        accept=".pdf,.doc,.docx"
        aria-describedby="file-help"
      />
      <div id="file-help">PDF or Word documents only, max 5MB</div>
    </form>
  );
};
```

### Autocomplete Attributes

```typescript
// React: Autocomplete for faster form completion
const AutocompleteForm: React.FC = () => {
  return (
    <form autoComplete="on">
      <h2>Personal Information</h2>

      <label htmlFor="name">Full Name</label>
      <input
        id="name"
        type="text"
        autoComplete="name"
      />

      <label htmlFor="given-name">First Name</label>
      <input
        id="given-name"
        type="text"
        autoComplete="given-name"
      />

      <label htmlFor="family-name">Last Name</label>
      <input
        id="family-name"
        type="text"
        autoComplete="family-name"
      />

      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        autoComplete="email"
      />

      <h2>Address</h2>

      <label htmlFor="street">Street Address</label>
      <input
        id="street"
        type="text"
        autoComplete="street-address"
      />

      <label htmlFor="city">City</label>
      <input
        id="city"
        type="text"
        autoComplete="address-level2"
      />

      <label htmlFor="state">State</label>
      <input
        id="state"
        type="text"
        autoComplete="address-level1"
      />

      <label htmlFor="zip">ZIP Code</label>
      <input
        id="zip"
        type="text"
        autoComplete="postal-code"
      />

      <label htmlFor="country">Country</label>
      <input
        id="country"
        type="text"
        autoComplete="country-name"
      />

      <h2>Credit Card</h2>

      <label htmlFor="cc-name">Name on Card</label>
      <input
        id="cc-name"
        type="text"
        autoComplete="cc-name"
      />

      <label htmlFor="cc-number">Card Number</label>
      <input
        id="cc-number"
        type="text"
        autoComplete="cc-number"
        inputMode="numeric"
      />

      <label htmlFor="cc-exp">Expiration</label>
      <input
        id="cc-exp"
        type="text"
        autoComplete="cc-exp"
        placeholder="MM/YY"
      />

      <label htmlFor="cc-csc">Security Code</label>
      <input
        id="cc-csc"
        type="text"
        autoComplete="cc-csc"
        inputMode="numeric"
      />
    </form>
  );
};

/*
Common autocomplete values:
- Personal: name, given-name, family-name, nickname
- Contact: email, tel, url
- Address: street-address, address-line1/2/3, city, state, postal-code, country
- Payment: cc-name, cc-number, cc-exp, cc-csc, cc-type
- Auth: username, current-password, new-password
- Other: bday, sex, organization, job-title
*/
```

## Error Messages and Validation

### Real-Time Validation

```typescript
// React: Accessible form validation
const AccessibleValidation: React.FC = () => {
  const [email, setEmail] = React.useState('');
  const [password, setPassword] = React.useState('');
  const [touched, setTouched] = React.useState({
    email: false,
    password: false
  });

  const errors = {
    email: touched.email && !email ? 'Email is required' :
           touched.email && !/\S+@\S+\.\S+/.test(email) ? 'Email is invalid' : '',
    password: touched.password && !password ? 'Password is required' :
              touched.password && password.length < 8 ? 'Password must be at least 8 characters' : ''
  };

  return (
    <form noValidate>
      {/* Email field with inline validation */}
      <div className="form-group">
        <label htmlFor="email">
          Email Address
          <span aria-label="required">*</span>
        </label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          onBlur={() => setTouched({ ...touched, email: true })}
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
        {errors.email && (
          <div
            id="email-error"
            role="alert"
            className="error-message"
          >
            <span aria-hidden="true">⚠</span> {errors.email}
          </div>
        )}
      </div>

      {/* Password field with inline validation */}
      <div className="form-group">
        <label htmlFor="password">
          Password
          <span aria-label="required">*</span>
        </label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          onBlur={() => setTouched({ ...touched, password: true })}
          aria-required="true"
          aria-invalid={!!errors.password}
          aria-describedby="password-help password-error"
        />
        <div id="password-help" className="help-text">
          Must be at least 8 characters
        </div>
        {errors.password && (
          <div
            id="password-error"
            role="alert"
            className="error-message"
          >
            <span aria-hidden="true">⚠</span> {errors.password}
          </div>
        )}
      </div>

      <button
        type="submit"
        disabled={!!errors.email || !!errors.password || !email || !password}
      >
        Sign In
      </button>
    </form>
  );
};
```

```typescript
// Angular: Comprehensive form validation
@Component({
  selector: 'app-validated-form',
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <!-- Error summary at top -->
      <div
        *ngIf="formSubmitted && registrationForm.invalid"
        role="alert"
        class="error-summary"
        tabindex="-1"
        #errorSummary
      >
        <h3>Please correct the following errors:</h3>
        <ul>
          <li *ngFor="let error of getErrorSummary()">
            <a [href]="'#' + error.field">{{ error.message }}</a>
          </li>
        </ul>
      </div>

      <!-- Username field -->
      <div class="form-group" [class.has-error]="showError('username')">
        <label for="username">
          Username
          <span class="required" aria-label="required">*</span>
        </label>
        <input
          id="username"
          type="text"
          formControlName="username"
          [attr.aria-required]="true"
          [attr.aria-invalid]="showError('username')"
          [attr.aria-describedby]="getDescribedBy('username')"
        />
        <div id="username-help" class="help-text">
          3-20 characters, letters and numbers only
        </div>
        <div
          *ngIf="showError('username')"
          id="username-error"
          role="alert"
          class="error-message"
        >
          {{ getErrorMessage('username') }}
        </div>
      </div>

      <!-- Email field -->
      <div class="form-group" [class.has-error]="showError('email')">
        <label for="email">
          Email Address
          <span class="required" aria-label="required">*</span>
        </label>
        <input
          id="email"
          type="email"
          formControlName="email"
          autoComplete="email"
          [attr.aria-required]="true"
          [attr.aria-invalid]="showError('email')"
          [attr.aria-describedby]="showError('email') ? 'email-error' : null"
        />
        <div
          *ngIf="showError('email')"
          id="email-error"
          role="alert"
          class="error-message"
        >
          {{ getErrorMessage('email') }}
        </div>
      </div>

      <!-- Password with strength indicator -->
      <div class="form-group" [class.has-error]="showError('password')">
        <label for="password">
          Password
          <span class="required" aria-label="required">*</span>
        </label>
        <input
          id="password"
          type="password"
          formControlName="password"
          autoComplete="new-password"
          [attr.aria-required]="true"
          [attr.aria-invalid]="showError('password')"
          [attr.aria-describedby]="getDescribedBy('password')"
        />
        <div id="password-help" class="help-text">
          Minimum 8 characters, include uppercase, lowercase, number, and symbol
        </div>
        <div
          id="password-strength"
          role="status"
          aria-live="polite"
          *ngIf="password.value"
          class="strength-indicator"
        >
          Password strength: {{ getPasswordStrength() }}
        </div>
        <div
          *ngIf="showError('password')"
          id="password-error"
          role="alert"
          class="error-message"
        >
          {{ getErrorMessage('password') }}
        </div>
      </div>

      <!-- Confirm password -->
      <div class="form-group" [class.has-error]="showError('confirmPassword')">
        <label for="confirmPassword">
          Confirm Password
          <span class="required" aria-label="required">*</span>
        </label>
        <input
          id="confirmPassword"
          type="password"
          formControlName="confirmPassword"
          autoComplete="new-password"
          [attr.aria-required]="true"
          [attr.aria-invalid]="showError('confirmPassword')"
          [attr.aria-describedby]="showError('confirmPassword') ? 'confirmPassword-error' : null"
        />
        <div
          *ngIf="showError('confirmPassword')"
          id="confirmPassword-error"
          role="alert"
          class="error-message"
        >
          {{ getErrorMessage('confirmPassword') }}
        </div>
      </div>

      <!-- Terms acceptance -->
      <div class="form-group" [class.has-error]="showError('terms')">
        <label class="checkbox-label">
          <input
            id="terms"
            type="checkbox"
            formControlName="terms"
            [attr.aria-required]="true"
            [attr.aria-invalid]="showError('terms')"
            [attr.aria-describedby]="showError('terms') ? 'terms-error' : null"
          />
          <span>
            I agree to the <a href="/terms">Terms of Service</a>
            <span class="required" aria-label="required">*</span>
          </span>
        </label>
        <div
          *ngIf="showError('terms')"
          id="terms-error"
          role="alert"
          class="error-message"
        >
          You must accept the terms to continue
        </div>
      </div>

      <button
        type="submit"
        [disabled]="registrationForm.invalid && formSubmitted"
      >
        Create Account
      </button>

      <!-- Success message -->
      <div
        *ngIf="submissionSuccess"
        role="alert"
        aria-live="polite"
        class="success-message"
      >
        Account created successfully! Redirecting to dashboard...
      </div>
    </form>
  `,
  styles: [`
    .error-summary {
      background-color: #FEF2F2;
      border: 2px solid #DC2626;
      border-radius: 4px;
      padding: 16px;
      margin-bottom: 24px;
    }

    .error-summary h3 {
      color: #DC2626;
      margin: 0 0 12px;
    }

    .error-summary ul {
      margin: 0;
      padding-left: 20px;
    }

    .error-summary a {
      color: #DC2626;
      text-decoration: underline;
    }

    .form-group {
      margin-bottom: 20px;
    }

    .has-error input {
      border-color: #DC2626;
    }

    .error-message {
      color: #DC2626;
      margin-top: 4px;
      display: flex;
      align-items: center;
      gap: 8px;
    }

    .success-message {
      background-color: #F0FDF4;
      border: 2px solid #16A34A;
      border-radius: 4px;
      padding: 12px;
      color: #16A34A;
      margin-top: 16px;
    }

    .required {
      color: #DC2626;
      margin-left: 4px;
    }

    .help-text {
      font-size: 0.875rem;
      color: #6B7280;
      margin-top: 4px;
    }

    .strength-indicator {
      margin-top: 8px;
      font-weight: 600;
    }
  `]
})
export class ValidatedFormComponent implements AfterViewInit {
  @ViewChild('errorSummary') errorSummary?: ElementRef;

  formSubmitted = false;
  submissionSuccess = false;

  registrationForm = this.fb.group({
    username: ['', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20),
      Validators.pattern(/^[a-zA-Z0-9]+$/)
    ]],
    email: ['', [Validators.required, Validators.email]],
    password: ['', [
      Validators.required,
      Validators.minLength(8),
      this.passwordStrengthValidator
    ]],
    confirmPassword: ['', Validators.required],
    terms: [false, Validators.requiredTrue]
  }, {
    validators: this.passwordMatchValidator
  });

  constructor(private fb: FormBuilder) {}

  ngAfterViewInit() {
    // Focus error summary when it appears
    if (this.formSubmitted && this.registrationForm.invalid && this.errorSummary) {
      this.errorSummary.nativeElement.focus();
    }
  }

  get username() { return this.registrationForm.get('username')!; }
  get email() { return this.registrationForm.get('email')!; }
  get password() { return this.registrationForm.get('password')!; }
  get confirmPassword() { return this.registrationForm.get('confirmPassword')!; }
  get terms() { return this.registrationForm.get('terms')!; }

  showError(field: string): boolean {
    const control = this.registrationForm.get(field);
    return !!(control?.invalid && (control?.touched || this.formSubmitted));
  }

  getErrorMessage(field: string): string {
    const control = this.registrationForm.get(field);
    if (!control?.errors) return '';

    const errors = control.errors;

    if (errors['required']) return `${this.getFieldName(field)} is required`;
    if (errors['email']) return 'Please enter a valid email address';
    if (errors['minlength']) {
      return `${this.getFieldName(field)} must be at least ${errors['minlength'].requiredLength} characters`;
    }
    if (errors['maxlength']) {
      return `${this.getFieldName(field)} must be no more than ${errors['maxlength'].requiredLength} characters`;
    }
    if (errors['pattern']) return 'Only letters and numbers are allowed';
    if (errors['passwordStrength']) return errors['passwordStrength'];
    if (errors['passwordMismatch']) return 'Passwords do not match';

    return 'Invalid value';
  }

  getFieldName(field: string): string {
    const names: Record<string, string> = {
      username: 'Username',
      email: 'Email',
      password: 'Password',
      confirmPassword: 'Confirm Password'
    };
    return names[field] || field;
  }

  getDescribedBy(field: string): string {
    const parts = [`${field}-help`];
    if (this.showError(field)) {
      parts.push(`${field}-error`);
    }
    if (field === 'password' && this.password.value) {
      parts.push('password-strength');
    }
    return parts.join(' ');
  }

  getErrorSummary(): Array<{ field: string; message: string }> {
    const errors: Array<{ field: string; message: string }> = [];

    Object.keys(this.registrationForm.controls).forEach(field => {
      if (this.showError(field)) {
        errors.push({
          field,
          message: this.getErrorMessage(field)
        });
      }
    });

    return errors;
  }

  getPasswordStrength(): string {
    const pwd = this.password.value || '';
    let strength = 0;

    if (pwd.length >= 8) strength++;
    if (/[a-z]/.test(pwd) && /[A-Z]/.test(pwd)) strength++;
    if (/\d/.test(pwd)) strength++;
    if (/[^a-zA-Z0-9]/.test(pwd)) strength++;

    const levels = ['Weak', 'Fair', 'Good', 'Strong'];
    return levels[strength] || 'Weak';
  }

  passwordStrengthValidator(control: AbstractControl): ValidationErrors | null {
    const value = control.value || '';

    if (value.length < 8) return null; // Let minLength handle this

    const hasUpper = /[A-Z]/.test(value);
    const hasLower = /[a-z]/.test(value);
    const hasNumber = /\d/.test(value);
    const hasSymbol = /[^a-zA-Z0-9]/.test(value);

    const valid = hasUpper && hasLower && hasNumber && hasSymbol;

    return valid ? null : {
      passwordStrength: 'Password must include uppercase, lowercase, number, and symbol'
    };
  }

  passwordMatchValidator(group: AbstractControl): ValidationErrors | null {
    const password = group.get('password')?.value;
    const confirmPassword = group.get('confirmPassword')?.value;

    if (!password || !confirmPassword) return null;

    return password === confirmPassword ? null : { passwordMismatch: true };
  }

  onSubmit() {
    this.formSubmitted = true;

    if (this.registrationForm.valid) {
      // Submit form
      this.submissionSuccess = true;
      console.log('Form submitted:', this.registrationForm.value);
    } else {
      // Focus error summary
      setTimeout(() => {
        this.errorSummary?.nativeElement.focus();
      }, 100);
    }
  }
}
```

## Required Fields

### Indicating Required Fields

```typescript
// React: Multiple ways to indicate required fields
const RequiredFieldPatterns: React.FC = () => {
  return (
    <form>
      {/* Pattern 1: Asterisk with aria-label */}
      <label htmlFor="name">
        Name <span aria-label="required" className="required">*</span>
      </label>
      <input id="name" type="text" required aria-required="true" />

      {/* Pattern 2: (required) text */}
      <label htmlFor="email">
        Email <span className="required">(required)</span>
      </label>
      <input id="email" type="email" required aria-required="true" />

      {/* Pattern 3: Visual + sr-only text */}
      <label htmlFor="phone">
        Phone
        <span className="required" aria-hidden="true">*</span>
        <span className="sr-only">required</span>
      </label>
      <input id="phone" type="tel" required aria-required="true" />

      {/* Legend explaining required fields */}
      <p className="form-legend">
        <span className="required" aria-hidden="true">*</span>
        <span> indicates required field</span>
      </p>
    </form>
  );
};
```

## Fieldsets and Legends

### Grouping Related Fields

```typescript
// React: Fieldsets for form sections
const FieldsetExample: React.FC = () => {
  return (
    <form>
      <fieldset>
        <legend>Personal Information</legend>

        <label htmlFor="firstName">First Name</label>
        <input id="firstName" type="text" />

        <label htmlFor="lastName">Last Name</label>
        <input id="lastName" type="text" />

        <label htmlFor="birthdate">Birth Date</label>
        <input id="birthdate" type="date" />
      </fieldset>

      <fieldset>
        <legend>Shipping Address</legend>

        <label htmlFor="street">Street Address</label>
        <input id="street" type="text" autoComplete="street-address" />

        <label htmlFor="city">City</label>
        <input id="city" type="text" autoComplete="address-level2" />

        <div className="form-row">
          <div>
            <label htmlFor="state">State</label>
            <input id="state" type="text" autoComplete="address-level1" />
          </div>
          <div>
            <label htmlFor="zip">ZIP</label>
            <input id="zip" type="text" autoComplete="postal-code" />
          </div>
        </div>
      </fieldset>

      {/* Radio button group */}
      <fieldset>
        <legend>Shipping Method</legend>

        <label>
          <input type="radio" name="shipping" value="standard" />
          Standard (5-7 days)
        </label>

        <label>
          <input type="radio" name="shipping" value="express" />
          Express (2-3 days)
        </label>

        <label>
          <input type="radio" name="shipping" value="overnight" />
          Overnight
        </label>
      </fieldset>

      {/* Checkbox group */}
      <fieldset>
        <legend>Newsletter Preferences</legend>

        <label>
          <input type="checkbox" name="news" value="weekly" />
          Weekly Newsletter
        </label>

        <label>
          <input type="checkbox" name="news" value="promotions" />
          Promotions & Offers
        </label>

        <label>
          <input type="checkbox" name="news" value="updates" />
          Product Updates
        </label>
      </fieldset>
    </form>
  );
};
```

## Help Text and Instructions

```typescript
// React: Various help text patterns
const HelpTextPatterns: React.FC = () => {
  return (
    <form>
      {/* Pattern 1: Static help text */}
      <label htmlFor="username">Username</label>
      <input
        id="username"
        type="text"
        aria-describedby="username-help"
      />
      <div id="username-help" className="help-text">
        3-20 characters, letters and numbers only
      </div>

      {/* Pattern 2: Help button/tooltip */}
      <label htmlFor="ssn">
        Social Security Number
        <button
          type="button"
          aria-label="Why do we need this?"
          aria-describedby="ssn-tooltip"
          className="help-button"
        >
          ?
        </button>
      </label>
      <input id="ssn" type="text" />
      <div id="ssn-tooltip" role="tooltip" className="tooltip">
        We need this to verify your identity
      </div>

      {/* Pattern 3: Expandable help */}
      <label htmlFor="routing">Routing Number</label>
      <input
        id="routing"
        type="text"
        aria-describedby="routing-help"
      />
      <details>
        <summary>Where do I find this?</summary>
        <div id="routing-help">
          Your routing number is located at the bottom left of your check
        </div>
      </details>

      {/* Pattern 4: Character counter */}
      <label htmlFor="bio">Bio</label>
      <textarea
        id="bio"
        maxLength={500}
        aria-describedby="bio-counter"
      />
      <div id="bio-counter" aria-live="polite" className="character-counter">
        0 / 500 characters
      </div>
    </form>
  );
};
```

## Common Mistakes

### 1. Placeholder as Label

```typescript
// ❌ WRONG: Placeholder instead of label
<input type="text" placeholder="Enter your email" />

// ✅ CORRECT: Proper label + optional placeholder
<label htmlFor="email">Email Address</label>
<input id="email" type="email" placeholder="name@example.com" />
```

### 2. Missing Error Associations

```typescript
// ❌ WRONG: Error message not associated
<input id="email" type="email" />
<div className="error">Email is required</div>

// ✅ CORRECT: Error associated with aria-describedby
<input
  id="email"
  type="email"
  aria-invalid="true"
  aria-describedby="email-error"
/>
<div id="email-error" role="alert">
  Email is required
</div>
```

### 3. Disabled Submit Without Explanation

```typescript
// ❌ WRONG: Disabled button with no context
<button type="submit" disabled>Submit</button>

// ✅ CORRECT: Explain why button is disabled
<button
  type="submit"
  disabled={!formValid}
  aria-disabled={!formValid}
  title={!formValid ? 'Please complete all required fields' : ''}
>
  Submit
</button>
{!formValid && (
  <div role="status" className="form-status">
    Please complete all required fields to continue
  </div>
)}
```

## Best Practices

### 1. Focus Management

```typescript
// React: Focus first error on submit
const FormWithFocusManagement: React.FC = () => {
  const firstErrorRef = useRef<HTMLInputElement>(null);
  const [errors, setErrors] = useState<Record<string, string>>({});

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    // Validate form
    const newErrors = validateForm();
    setErrors(newErrors);

    if (Object.keys(newErrors).length > 0) {
      // Focus first field with error
      firstErrorRef.current?.focus();
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
    </form>
  );
};
```

### 2. Clear Visual Indicators

```typescript
// CSS for accessible form styling
const formStyles = `
  .form-group {
    margin-bottom: 20px;
  }

  label {
    display: block;
    margin-bottom: 4px;
    font-weight: 600;
  }

  .required {
    color: #DC2626;
  }

  input:focus,
  textarea:focus,
  select:focus {
    outline: 2px solid #2563EB;
    outline-offset: 2px;
  }

  [aria-invalid="true"] {
    border-color: #DC2626;
    border-width: 2px;
  }

  .error-message {
    color: #DC2626;
    margin-top: 4px;
  }

  .help-text {
    color: #6B7280;
    font-size: 0.875rem;
    margin-top: 4px;
  }
`;
```

## Interview Questions

### Junior Level
1. **Why are labels important for form accessibility?**
   - Associate text with input fields
   - Screen readers announce labels
   - Increase click target size
   - Required by WCAG

2. **What is the difference between placeholder and label?**
   - Labels are permanent, visible
   - Placeholders disappear when typing
   - Placeholders not announced reliably
   - Never use placeholder as only label

3. **How do you indicate a required field?**
   - Use required attribute
   - Add aria-required="true"
   - Visual indicator (asterisk)
   - Explain in legend/instructions

### Mid Level
4. **Explain aria-describedby and when to use it**
   - Associates help text with input
   - Screen reader announces after label
   - Use for help text and errors
   - Can reference multiple IDs

5. **How should form errors be announced?**
   - Use aria-invalid on field
   - role="alert" on error message
   - Associate with aria-describedby
   - Error summary at top with links

6. **What is the purpose of autocomplete attributes?**
   - Help browsers autofill forms
   - Faster for users
   - Especially helpful for disabilities
   - Standardized values

### Senior Level
7. **Design an accessible multi-step form**
   - Progress indicator
   - Save progress ability
   - Clear navigation between steps
   - Announce step changes
   - Allow backwards navigation
   - Validate each step

8. **How would you handle async validation?**
   - Show loading state
   - Announce results
   - Don't block form submission
   - Clear timeout handling
   - Debounce requests

9. **Explain accessible error recovery strategies**
   - Error summary at top
   - Link to first error
   - Inline error messages
   - Preserve user input
   - Suggest corrections
   - Multiple attempts allowed

10. **How do you make custom form controls accessible?**
    - Use ARIA roles appropriately
    - Implement keyboard navigation
    - Manage focus
    - Announce state changes
    - Match native behavior
    - Test with screen readers

## Key Takeaways

1. **Every input needs a label** - no exceptions
2. **Use semantic HTML** - correct input types
3. **Indicate required fields** - visually and programmatically
4. **Associate errors** - aria-invalid + aria-describedby
5. **Provide help text** - aria-describedby for instructions
6. **Use fieldsets** - group related fields
7. **Enable autocomplete** - use correct autocomplete values
8. **Announce changes** - live regions for dynamic feedback
9. **Focus management** - move focus to errors
10. **Test thoroughly** - with keyboard and screen readers

## Resources

### Official Documentation
- [WCAG 2.1 - Forms](https://www.w3.org/WAI/tutorials/forms/)
- [MDN - HTML Forms](https://developer.mozilla.org/en-US/docs/Learn/Forms)
- [W3C - Accessible Forms](https://www.w3.org/WAI/tutorials/forms/)

### Tools
- [axe DevTools](https://www.deque.com/axe/devtools/)
- [WAVE](https://wave.webaim.org/)
- [Accessibility Insights](https://accessibilityinsights.io/)

### Articles
- [WebAIM: Creating Accessible Forms](https://webaim.org/techniques/forms/)
- [Smashing Magazine: Form Design Patterns](https://www.smashingmagazine.com/printed-books/form-design-patterns/)
- [GOV.UK Design System - Forms](https://design-system.service.gov.uk/components/)
