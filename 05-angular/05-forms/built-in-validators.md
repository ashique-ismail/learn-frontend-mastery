# Built-in Validators in Angular

## Overview

Angular provides a comprehensive set of built-in validators for reactive and template-driven forms. These validators implement common validation patterns like required fields, min/max values, email format, and regular expressions. Understanding how to use and combine these validators is essential for building robust form validation.

This guide covers all built-in validators, how to apply them, combine multiple validators, and work with validation errors and error messages.

## Table of Contents

1. [Required Validator](#required-validator)
2. [Email Validator](#email-validator)
3. [Min and Max Validators](#min-and-max-validators)
4. [MinLength and MaxLength](#minlength-and-maxlength)
5. [Pattern Validator](#pattern-validator)
6. [RequiredTrue Validator](#requiredtrue-validator)
7. [Composing Validators](#composing-validators)
8. [Validation Errors](#validation-errors)
9. [Displaying Error Messages](#displaying-error-messages)
10. [Conditional Validation](#conditional-validation)

## Required Validator

The most common validator ensures a field has a value.

### Basic Required Validation

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-required-demo',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Name *</label>
        <input formControlName="name" />
        <div *ngIf="form.controls.name.invalid && form.controls.name.touched">
          <span class="error">Name is required</span>
        </div>
      </div>

      <div>
        <label>Email *</label>
        <input formControlName="email" type="email" />
        <div *ngIf="form.controls.email.invalid && form.controls.email.touched">
          <span class="error">Email is required</span>
        </div>
      </div>

      <button [disabled]="form.invalid">Submit</button>
    </form>
  `,
  styles: [`
    .error { color: red; font-size: 0.875rem; }
  `]
})
export class RequiredDemoComponent {
  form = new FormGroup({
    name: new FormControl('', { 
      validators: Validators.required,
      nonNullable: true 
    }),
    email: new FormControl('', { 
      validators: Validators.required,
      nonNullable: true 
    })
  });

  onSubmit() {
    if (this.form.valid) {
      console.log('Form submitted:', this.form.value);
    } else {
      // Mark all fields as touched to show errors
      this.form.markAllAsTouched();
    }
  }
}
```

### Required with Whitespace

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-whitespace-demo',
  template: `
    <input [formControl]="nameControl" placeholder="Name" />
    <div *ngIf="nameControl.invalid && nameControl.touched">
      <span class="error">Name cannot be empty or whitespace only</span>
    </div>
  `
})
export class WhitespaceDemoComponent {
  // Validators.required treats whitespace as valid
  nameControl = new FormControl('', Validators.required);

  ngOnInit() {
    this.nameControl.setValue('   '); // Valid! (just spaces)
    console.log(this.nameControl.valid); // true
  }
}

// Custom validator to reject whitespace
function noWhitespaceValidator(control: FormControl): { [key: string]: any } | null {
  const isWhitespace = (control.value || '').trim().length === 0;
  return isWhitespace ? { whitespace: true } : null;
}

@Component({
  selector: 'app-no-whitespace-demo',
  template: `
    <input [formControl]="nameControl" placeholder="Name" />
  `
})
export class NoWhitespaceDemoComponent {
  nameControl = new FormControl('', [
    Validators.required,
    noWhitespaceValidator
  ]);

  ngOnInit() {
    this.nameControl.setValue('   '); // Invalid!
    console.log(this.nameControl.errors);
    // { whitespace: true }
  }
}
```

## Email Validator

Validates email format using a basic regex pattern.

### Email Validation

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-email-demo',
  template: `
    <div>
      <label>Email</label>
      <input [formControl]="emailControl" type="email" placeholder="user@example.com" />
      
      <div *ngIf="emailControl.invalid && emailControl.touched">
        <span *ngIf="emailControl.errors?.['required']" class="error">
          Email is required
        </span>
        <span *ngIf="emailControl.errors?.['email']" class="error">
          Invalid email format
        </span>
      </div>

      <div *ngIf="emailControl.valid">
        <span class="success">Valid email!</span>
      </div>
    </div>
  `
})
export class EmailDemoComponent {
  emailControl = new FormControl('', [
    Validators.required,
    Validators.email
  ]);

  testEmails() {
    // Valid emails
    this.emailControl.setValue('user@example.com'); // valid
    this.emailControl.setValue('first.last@example.co.uk'); // valid
    this.emailControl.setValue('user+tag@example.com'); // valid

    // Invalid emails
    this.emailControl.setValue('notanemail'); // invalid
    this.emailControl.setValue('@example.com'); // invalid
    this.emailControl.setValue('user@'); // invalid
  }
}
```

### Email with Custom Pattern

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-email-pattern-demo',
  template: `
    <input [formControl]="emailControl" type="email" />
    <div *ngIf="emailControl.errors?.['pattern']">
      Email must be from company.com domain
    </div>
  `
})
export class EmailPatternDemoComponent {
  // Only accept emails from company.com
  emailControl = new FormControl('', [
    Validators.required,
    Validators.email,
    Validators.pattern(/^[a-z0-9._%+-]+@company\.com$/i)
  ]);
}
```

## Min and Max Validators

Validates numeric minimum and maximum values.

### Numeric Min/Max

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-minmax-demo',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Age (18-120)</label>
        <input formControlName="age" type="number" />
        <div *ngIf="ageControl.invalid && ageControl.touched">
          <span *ngIf="ageControl.errors?.['required']">Age is required</span>
          <span *ngIf="ageControl.errors?.['min']">
            Age must be at least {{ ageControl.errors?.['min'].min }}
          </span>
          <span *ngIf="ageControl.errors?.['max']">
            Age cannot exceed {{ ageControl.errors?.['max'].max }}
          </span>
        </div>
      </div>

      <div>
        <label>Rating (1-5)</label>
        <input formControlName="rating" type="number" step="0.1" />
        <div *ngIf="ratingControl.invalid && ratingControl.touched">
          <span *ngIf="ratingControl.errors?.['min']">
            Rating must be at least 1
          </span>
          <span *ngIf="ratingControl.errors?.['max']">
            Rating cannot exceed 5
          </span>
        </div>
      </div>

      <button [disabled]="form.invalid">Submit</button>
    </form>
  `
})
export class MinMaxDemoComponent {
  form = new FormGroup({
    age: new FormControl<number | null>(null, [
      Validators.required,
      Validators.min(18),
      Validators.max(120)
    ]),
    rating: new FormControl<number | null>(null, [
      Validators.min(1),
      Validators.max(5)
    ])
  });

  get ageControl() {
    return this.form.controls.age;
  }

  get ratingControl() {
    return this.form.controls.rating;
  }
}
```

### Range Validation with Error Messages

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-range-demo',
  template: `
    <div>
      <label>Price (${{ minPrice }} - ${{ maxPrice }})</label>
      <input [formControl]="priceControl" type="number" step="0.01" />
      <div *ngIf="priceControl.invalid && priceControl.touched">
        {{ getPriceError() }}
      </div>
    </div>
  `
})
export class RangeDemoComponent {
  minPrice = 10;
  maxPrice = 1000;

  priceControl = new FormControl<number | null>(null, [
    Validators.required,
    Validators.min(this.minPrice),
    Validators.max(this.maxPrice)
  ]);

  getPriceError(): string {
    if (this.priceControl.errors?.['required']) {
      return 'Price is required';
    }
    if (this.priceControl.errors?.['min']) {
      return `Price must be at least $${this.minPrice}`;
    }
    if (this.priceControl.errors?.['max']) {
      return `Price cannot exceed $${this.maxPrice}`;
    }
    return '';
  }
}
```

## MinLength and MaxLength

Validates string length constraints.

### String Length Validation

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-length-demo',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Username (3-20 characters)</label>
        <input formControlName="username" />
        <small>{{ usernameControl.value?.length || 0 }} / 20</small>
        <div *ngIf="usernameControl.invalid && usernameControl.touched">
          <span *ngIf="usernameControl.errors?.['required']">
            Username is required
          </span>
          <span *ngIf="usernameControl.errors?.['minlength']">
            Username must be at least 
            {{ usernameControl.errors?.['minlength'].requiredLength }} characters
            (current: {{ usernameControl.errors?.['minlength'].actualLength }})
          </span>
          <span *ngIf="usernameControl.errors?.['maxlength']">
            Username cannot exceed 
            {{ usernameControl.errors?.['maxlength'].requiredLength }} characters
            (current: {{ usernameControl.errors?.['maxlength'].actualLength }})
          </span>
        </div>
      </div>

      <div>
        <label>Bio (optional, max 500 characters)</label>
        <textarea formControlName="bio" rows="4"></textarea>
        <small>{{ bioControl.value?.length || 0 }} / 500</small>
        <div *ngIf="bioControl.errors?.['maxlength']">
          <span>Bio is too long</span>
        </div>
      </div>
    </form>
  `
})
export class LengthDemoComponent {
  form = new FormGroup({
    username: new FormControl('', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20)
    ]),
    bio: new FormControl('', Validators.maxLength(500))
  });

  get usernameControl() {
    return this.form.controls.username;
  }

  get bioControl() {
    return this.form.controls.bio;
  }
}
```

### Password Strength with Length

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-password-demo',
  template: `
    <div>
      <label>Password</label>
      <input [formControl]="passwordControl" type="password" />
      
      <div class="strength-indicator">
        <div [class.weak]="getPasswordStrength() === 'weak'"
             [class.medium]="getPasswordStrength() === 'medium'"
             [class.strong]="getPasswordStrength() === 'strong'">
          {{ getPasswordStrength() }}
        </div>
      </div>

      <div *ngIf="passwordControl.invalid && passwordControl.touched">
        <span *ngIf="passwordControl.errors?.['required']">
          Password is required
        </span>
        <span *ngIf="passwordControl.errors?.['minlength']">
          Password must be at least 8 characters
        </span>
      </div>

      <ul class="requirements">
        <li [class.met]="passwordControl.value!.length >= 8">
          At least 8 characters
        </li>
        <li [class.met]="/[A-Z]/.test(passwordControl.value!)">
          Contains uppercase letter
        </li>
        <li [class.met]="/[a-z]/.test(passwordControl.value!)">
          Contains lowercase letter
        </li>
        <li [class.met]="/[0-9]/.test(passwordControl.value!)">
          Contains number
        </li>
        <li [class.met]="/[^A-Za-z0-9]/.test(passwordControl.value!)">
          Contains special character
        </li>
      </ul>
    </div>
  `,
  styles: [`
    .requirements li { color: #999; }
    .requirements li.met { color: green; }
    .weak { color: red; }
    .medium { color: orange; }
    .strong { color: green; }
  `]
})
export class PasswordDemoComponent {
  passwordControl = new FormControl('', [
    Validators.required,
    Validators.minLength(8),
    Validators.maxLength(100)
  ]);

  getPasswordStrength(): string {
    const value = this.passwordControl.value || '';
    if (value.length < 8) return 'weak';

    let strength = 0;
    if (/[A-Z]/.test(value)) strength++;
    if (/[a-z]/.test(value)) strength++;
    if (/[0-9]/.test(value)) strength++;
    if (/[^A-Za-z0-9]/.test(value)) strength++;

    if (strength <= 2) return 'weak';
    if (strength === 3) return 'medium';
    return 'strong';
  }
}
```

## Pattern Validator

Validates against regular expression patterns.

### Common Pattern Examples

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-pattern-demo',
  template: `
    <form [formGroup]="form">
      <!-- Phone number -->
      <div>
        <label>Phone (US format)</label>
        <input formControlName="phone" placeholder="(123) 456-7890" />
        <div *ngIf="form.controls.phone.errors?.['pattern']">
          Invalid phone format. Use: (123) 456-7890
        </div>
      </div>

      <!-- Zip code -->
      <div>
        <label>Zip Code</label>
        <input formControlName="zipCode" placeholder="12345 or 12345-6789" />
        <div *ngIf="form.controls.zipCode.errors?.['pattern']">
          Invalid zip code format
        </div>
      </div>

      <!-- Username (alphanumeric, underscore, hyphen) -->
      <div>
        <label>Username</label>
        <input formControlName="username" placeholder="user_name-123" />
        <div *ngIf="form.controls.username.errors?.['pattern']">
          Username can only contain letters, numbers, underscores, and hyphens
        </div>
      </div>

      <!-- URL -->
      <div>
        <label>Website</label>
        <input formControlName="website" placeholder="https://example.com" />
        <div *ngIf="form.controls.website.errors?.['pattern']">
          Invalid URL format
        </div>
      </div>

      <!-- Hex color -->
      <div>
        <label>Color (hex)</label>
        <input formControlName="color" placeholder="#RRGGBB" />
        <div *ngIf="form.controls.color.errors?.['pattern']">
          Invalid hex color. Use: #RRGGBB
        </div>
      </div>
    </form>
  `
})
export class PatternDemoComponent {
  form = new FormGroup({
    // US phone number: (123) 456-7890
    phone: new FormControl('', 
      Validators.pattern(/^\(\d{3}\) \d{3}-\d{4}$/)
    ),

    // US zip code: 12345 or 12345-6789
    zipCode: new FormControl('', 
      Validators.pattern(/^\d{5}(-\d{4})?$/)
    ),

    // Username: letters, numbers, underscore, hyphen
    username: new FormControl('', 
      Validators.pattern(/^[a-zA-Z0-9_-]+$/)
    ),

    // URL with http(s)
    website: new FormControl('', 
      Validators.pattern(/^https?:\/\/.+\..+/)
    ),

    // Hex color: #RRGGBB
    color: new FormControl('', 
      Validators.pattern(/^#[0-9A-Fa-f]{6}$/)
    )
  });
}
```

### Credit Card Pattern

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-credit-card-demo',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Credit Card Number</label>
        <input formControlName="cardNumber" 
               placeholder="1234 5678 9012 3456"
               maxlength="19" />
        <div *ngIf="form.controls.cardNumber.invalid && form.controls.cardNumber.touched">
          <span *ngIf="form.controls.cardNumber.errors?.['required']">
            Card number is required
          </span>
          <span *ngIf="form.controls.cardNumber.errors?.['pattern']">
            Invalid card number format
          </span>
        </div>
        <div *ngIf="form.controls.cardNumber.valid">
          Card type: {{ getCardType() }}
        </div>
      </div>

      <div>
        <label>CVV</label>
        <input formControlName="cvv" 
               placeholder="123"
               maxlength="4" />
        <div *ngIf="form.controls.cvv.errors?.['pattern']">
          CVV must be 3 or 4 digits
        </div>
      </div>

      <div>
        <label>Expiry (MM/YY)</label>
        <input formControlName="expiry" 
               placeholder="12/25"
               maxlength="5" />
        <div *ngIf="form.controls.expiry.errors?.['pattern']">
          Invalid expiry format. Use: MM/YY
        </div>
      </div>
    </form>
  `
})
export class CreditCardDemoComponent {
  form = new FormGroup({
    // Card number: 16 digits with optional spaces
    cardNumber: new FormControl('', [
      Validators.required,
      Validators.pattern(/^(\d{4}\s?){3}\d{4}$/)
    ]),

    // CVV: 3 or 4 digits
    cvv: new FormControl('', [
      Validators.required,
      Validators.pattern(/^\d{3,4}$/)
    ]),

    // Expiry: MM/YY format
    expiry: new FormControl('', [
      Validators.required,
      Validators.pattern(/^(0[1-9]|1[0-2])\/\d{2}$/)
    ])
  });

  getCardType(): string {
    const number = this.form.value.cardNumber?.replace(/\s/g, '') || '';
    
    if (/^4/.test(number)) return 'Visa';
    if (/^5[1-5]/.test(number)) return 'Mastercard';
    if (/^3[47]/.test(number)) return 'American Express';
    if (/^6(?:011|5)/.test(number)) return 'Discover';
    
    return 'Unknown';
  }
}
```

## RequiredTrue Validator

Validates that a boolean is true (useful for checkboxes).

### Terms and Conditions

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-terms-demo',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div>
        <label>Email</label>
        <input formControlName="email" type="email" />
      </div>

      <div>
        <label>
          <input formControlName="agreeToTerms" type="checkbox" />
          I agree to the terms and conditions *
        </label>
        <div *ngIf="form.controls.agreeToTerms.invalid && form.controls.agreeToTerms.touched">
          <span class="error">You must agree to the terms</span>
        </div>
      </div>

      <div>
        <label>
          <input formControlName="subscribe" type="checkbox" />
          Subscribe to newsletter (optional)
        </label>
      </div>

      <button [disabled]="form.invalid">Register</button>
    </form>
  `
})
export class TermsDemoComponent {
  form = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    
    // Must be true
    agreeToTerms: new FormControl(false, Validators.requiredTrue),
    
    // Optional - no validation
    subscribe: new FormControl(false)
  });

  onSubmit() {
    if (this.form.valid) {
      console.log('Form submitted:', this.form.value);
    } else {
      this.form.markAllAsTouched();
    }
  }
}
```

### Age Verification

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-age-verification',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Date of Birth</label>
        <input formControlName="dateOfBirth" type="date" />
      </div>

      <div>
        <label>
          <input formControlName="confirmAge" type="checkbox" />
          I confirm that I am 18 years or older *
        </label>
        <div *ngIf="form.controls.confirmAge.invalid && form.controls.confirmAge.touched">
          <span class="error">You must be 18 or older</span>
        </div>
      </div>

      <button [disabled]="form.invalid">Continue</button>
    </form>
  `
})
export class AgeVerificationComponent {
  form = new FormGroup({
    dateOfBirth: new FormControl('', Validators.required),
    confirmAge: new FormControl(false, Validators.requiredTrue)
  });
}
```

## Composing Validators

Combining multiple validators on a single control.

### Multiple Validators Example

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-composed-validators',
  template: `
    <div>
      <label>Password</label>
      <input [formControl]="passwordControl" type="password" />
      
      <ul class="validation-messages">
        <li [class.invalid]="passwordControl.hasError('required') && passwordControl.touched">
          Password is required
        </li>
        <li [class.invalid]="passwordControl.hasError('minlength')">
          Must be at least 8 characters
        </li>
        <li [class.invalid]="passwordControl.hasError('maxlength')">
          Cannot exceed 50 characters
        </li>
        <li [class.invalid]="passwordControl.hasError('pattern')">
          Must contain uppercase, lowercase, number, and special character
        </li>
      </ul>
    </div>
  `,
  styles: [`
    .validation-messages li { color: green; }
    .validation-messages li.invalid { color: red; }
  `]
})
export class ComposedValidatorsComponent {
  passwordControl = new FormControl('', [
    Validators.required,
    Validators.minLength(8),
    Validators.maxLength(50),
    Validators.pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]+$/)
  ]);
}
```

### Compose Helper

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators, ValidatorFn } from '@angular/forms';

@Component({
  selector: 'app-compose-helper',
  template: `
    <input [formControl]="control" />
    <div *ngFor="let error of getErrors()">
      {{ error }}
    </div>
  `
})
export class ComposeHelperComponent {
  // Combine validators using Validators.compose
  control = new FormControl('', 
    Validators.compose([
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20),
      Validators.pattern(/^[a-zA-Z0-9]+$/)
    ])
  );

  getErrors(): string[] {
    const errors = this.control.errors;
    if (!errors) return [];

    const messages: string[] = [];
    
    if (errors['required']) messages.push('Field is required');
    if (errors['minlength']) messages.push(`Minimum ${errors['minlength'].requiredLength} characters`);
    if (errors['maxlength']) messages.push(`Maximum ${errors['maxlength'].requiredLength} characters`);
    if (errors['pattern']) messages.push('Only letters and numbers allowed');

    return messages;
  }
}
```

## Validation Errors

Understanding the error object structure.

### Error Structure

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-error-structure',
  template: `
    <input [formControl]="control" />
    <pre>{{ control.errors | json }}</pre>
  `
})
export class ErrorStructureComponent {
  control = new FormControl('', [
    Validators.required,
    Validators.minLength(5),
    Validators.maxLength(10),
    Validators.pattern(/^\d+$/)
  ]);

  ngOnInit() {
    // Empty value
    console.log(this.control.errors);
    // { required: true }

    // Too short
    this.control.setValue('123');
    console.log(this.control.errors);
    // { minlength: { requiredLength: 5, actualLength: 3 } }

    // Too long
    this.control.setValue('12345678901');
    console.log(this.control.errors);
    // { maxlength: { requiredLength: 10, actualLength: 11 } }

    // Wrong pattern
    this.control.setValue('abcde');
    console.log(this.control.errors);
    // { pattern: { requiredPattern: '^\\d+$', actualValue: 'abcde' } }

    // Valid
    this.control.setValue('12345');
    console.log(this.control.errors);
    // null
  }
}
```

### Checking Specific Errors

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-check-errors',
  template: `
    <form [formGroup]="form">
      <input formControlName="username" />
      
      <!-- Method 1: errors object -->
      <div *ngIf="form.controls.username.errors?.['required']">
        Required
      </div>

      <!-- Method 2: hasError() -->
      <div *ngIf="form.controls.username.hasError('minlength')">
        Too short
      </div>

      <!-- Method 3: getError() for error details -->
      <div *ngIf="form.controls.username.hasError('maxlength')">
        Maximum {{ form.controls.username.getError('maxlength')?.requiredLength }} characters
      </div>
    </form>
  `
})
export class CheckErrorsComponent {
  form = new FormGroup({
    username: new FormControl('', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20)
    ])
  });
}
```

## Displaying Error Messages

Best practices for showing validation errors.

### Error Message Component

```typescript
import { Component, Input } from '@angular/core';
import { AbstractControl } from '@angular/forms';

@Component({
  selector: 'app-error-message',
  template: `
    <div *ngIf="shouldShowErrors()" class="error-message">
      <div *ngIf="control.hasError('required')">
        {{ fieldName }} is required
      </div>
      <div *ngIf="control.hasError('email')">
        Invalid email format
      </div>
      <div *ngIf="control.hasError('minlength')">
        {{ fieldName }} must be at least 
        {{ control.getError('minlength')?.requiredLength }} characters
      </div>
      <div *ngIf="control.hasError('maxlength')">
        {{ fieldName }} cannot exceed 
        {{ control.getError('maxlength')?.requiredLength }} characters
      </div>
      <div *ngIf="control.hasError('min')">
        {{ fieldName }} must be at least {{ control.getError('min')?.min }}
      </div>
      <div *ngIf="control.hasError('max')">
        {{ fieldName }} cannot exceed {{ control.getError('max')?.max }}
      </div>
      <div *ngIf="control.hasError('pattern')">
        {{ fieldName }} format is invalid
      </div>
    </div>
  `,
  styles: [`
    .error-message {
      color: #d32f2f;
      font-size: 0.875rem;
      margin-top: 0.25rem;
    }
  `]
})
export class ErrorMessageComponent {
  @Input() control!: AbstractControl;
  @Input() fieldName: string = 'Field';

  shouldShowErrors(): boolean {
    return this.control.invalid && (this.control.dirty || this.control.touched);
  }
}

// Usage
@Component({
  selector: 'app-form-with-errors',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Email</label>
        <input formControlName="email" />
        <app-error-message 
          [control]="form.controls.email" 
          fieldName="Email">
        </app-error-message>
      </div>

      <div>
        <label>Password</label>
        <input formControlName="password" type="password" />
        <app-error-message 
          [control]="form.controls.password" 
          fieldName="Password">
        </app-error-message>
      </div>
    </form>
  `
})
export class FormWithErrorsComponent {
  form = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [Validators.required, Validators.minLength(8)])
  });
}
```

### Error Messages Map

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-error-map',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Username</label>
        <input formControlName="username" />
        <div class="error" *ngIf="form.controls.username.invalid && form.controls.username.touched">
          {{ getErrorMessage(form.controls.username) }}
        </div>
      </div>

      <div>
        <label>Age</label>
        <input formControlName="age" type="number" />
        <div class="error" *ngIf="form.controls.age.invalid && form.controls.age.touched">
          {{ getErrorMessage(form.controls.age) }}
        </div>
      </div>
    </form>
  `
})
export class ErrorMapComponent {
  form = new FormGroup({
    username: new FormControl('', [
      Validators.required,
      Validators.minLength(3),
      Validators.maxLength(20)
    ]),
    age: new FormControl<number | null>(null, [
      Validators.required,
      Validators.min(18),
      Validators.max(120)
    ])
  });

  private errorMessages: { [key: string]: (error: any) => string } = {
    required: () => 'This field is required',
    email: () => 'Invalid email format',
    minlength: (error) => `Minimum ${error.requiredLength} characters required`,
    maxlength: (error) => `Maximum ${error.requiredLength} characters allowed`,
    min: (error) => `Value must be at least ${error.min}`,
    max: (error) => `Value cannot exceed ${error.max}`,
    pattern: () => 'Invalid format'
  };

  getErrorMessage(control: AbstractControl): string {
    if (!control.errors) return '';

    const errorKey = Object.keys(control.errors)[0];
    const errorFactory = this.errorMessages[errorKey];
    
    return errorFactory ? errorFactory(control.errors[errorKey]) : 'Invalid value';
  }
}
```

## Conditional Validation

Adding or removing validators dynamically.

### Dynamic Validators

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-conditional-validation',
  template: `
    <form [formGroup]="form">
      <div>
        <label>
          <input formControlName="hasMiddleName" type="checkbox" />
          I have a middle name
        </label>
      </div>

      <div *ngIf="form.value.hasMiddleName">
        <label>Middle Name</label>
        <input formControlName="middleName" />
        <div *ngIf="form.controls.middleName.invalid && form.controls.middleName.touched">
          Middle name is required
        </div>
      </div>

      <div>
        <label>Shipping Method</label>
        <select formControlName="shippingMethod">
          <option value="pickup">Pickup</option>
          <option value="delivery">Delivery</option>
        </select>
      </div>

      <div *ngIf="form.value.shippingMethod === 'delivery'">
        <label>Delivery Address</label>
        <input formControlName="deliveryAddress" />
        <div *ngIf="form.controls.deliveryAddress.invalid && form.controls.deliveryAddress.touched">
          Delivery address is required
        </div>
      </div>
    </form>
  `
})
export class ConditionalValidationComponent {
  form = new FormGroup({
    hasMiddleName: new FormControl(false, { nonNullable: true }),
    middleName: new FormControl(''),
    shippingMethod: new FormControl<'pickup' | 'delivery'>('pickup', { 
      nonNullable: true 
    }),
    deliveryAddress: new FormControl('')
  });

  ngOnInit() {
    // Watch hasMiddleName checkbox
    this.form.controls.hasMiddleName.valueChanges.subscribe(hasMiddleName => {
      if (hasMiddleName) {
        this.form.controls.middleName.setValidators(Validators.required);
      } else {
        this.form.controls.middleName.clearValidators();
        this.form.controls.middleName.setValue('');
      }
      this.form.controls.middleName.updateValueAndValidity();
    });

    // Watch shipping method
    this.form.controls.shippingMethod.valueChanges.subscribe(method => {
      if (method === 'delivery') {
        this.form.controls.deliveryAddress.setValidators(Validators.required);
      } else {
        this.form.controls.deliveryAddress.clearValidators();
        this.form.controls.deliveryAddress.setValue('');
      }
      this.form.controls.deliveryAddress.updateValueAndValidity();
    });
  }
}
```

### setValidators vs addValidators

```typescript
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-validators-demo',
  template: `
    <input [formControl]="control" />
    <div>{{ control.errors | json }}</div>
  `
})
export class ValidatorsDemoComponent {
  control = new FormControl('', Validators.required);

  modifyValidators() {
    // setValidators REPLACES all validators
    this.control.setValidators([
      Validators.minLength(5),
      Validators.maxLength(10)
    ]);
    // Now only has minLength and maxLength (required is gone!)
    this.control.updateValueAndValidity();

    // addValidators ADDS to existing validators (Angular 15+)
    this.control.addValidators(Validators.pattern(/^\d+$/));
    // Now has minLength, maxLength, AND pattern
    this.control.updateValueAndValidity();

    // removeValidators removes specific validators (Angular 15+)
    this.control.removeValidators(Validators.minLength(5));
    // Now has maxLength and pattern (minLength removed)
    this.control.updateValueAndValidity();

    // hasValidator checks if a validator exists (Angular 15+)
    console.log(this.control.hasValidator(Validators.required)); // false
    console.log(this.control.hasValidator(Validators.maxLength(10))); // true
  }
}
```

## Common Mistakes

### 1. Not Calling updateValueAndValidity

```typescript
// WRONG: Validators changed but not applied
control.setValidators(Validators.required);
// Control still thinks it's valid!

// CORRECT: Update after changing validators
control.setValidators(Validators.required);
control.updateValueAndValidity();
```

### 2. Checking Errors Without touched/dirty

```typescript
// WRONG: Shows errors immediately
<div *ngIf="control.invalid">Error!</div>

// CORRECT: Wait until user interacts
<div *ngIf="control.invalid && control.touched">Error!</div>
```

### 3. String Pattern Instead of RegExp

```typescript
// WRONG: Pattern as string
Validators.pattern('^[0-9]+$') // Doesn't work!

// CORRECT: Pattern as RegExp
Validators.pattern(/^[0-9]+$/)
```

### 4. Forgetting to Mark Fields as Touched on Submit

```typescript
// WRONG: Errors don't show on submit
onSubmit() {
  if (this.form.valid) {
    // submit
  }
}

// CORRECT: Show all errors on submit attempt
onSubmit() {
  if (this.form.valid) {
    // submit
  } else {
    this.form.markAllAsTouched();
  }
}
```

## Best Practices

### 1. Create Reusable Error Component

```typescript
@Component({
  selector: 'app-field-error',
  template: `
    <small class="error" *ngIf="shouldShow()">
      <ng-content></ng-content>
    </small>
  `
})
export class FieldErrorComponent {
  @Input() control!: AbstractControl;
  
  shouldShow(): boolean {
    return this.control.invalid && (this.control.dirty || this.control.touched);
  }
}
```

### 2. Extract Validation Constants

```typescript
export const ValidationPatterns = {
  EMAIL: /^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,4}$/i,
  PHONE_US: /^\(\d{3}\) \d{3}-\d{4}$/,
  ZIP_US: /^\d{5}(-\d{4})?$/,
  PASSWORD: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/
};

export const ValidationLimits = {
  USERNAME_MIN: 3,
  USERNAME_MAX: 20,
  PASSWORD_MIN: 8,
  BIO_MAX: 500
};
```

### 3. Combine Validators in Functions

```typescript
function getPasswordValidators(): ValidatorFn[] {
  return [
    Validators.required,
    Validators.minLength(8),
    Validators.maxLength(100),
    Validators.pattern(ValidationPatterns.PASSWORD)
  ];
}

form = new FormGroup({
  password: new FormControl('', getPasswordValidators())
});
```

### 4. Error Message Service

```typescript
@Injectable({ providedIn: 'root' })
export class ErrorMessageService {
  getErrorMessage(control: AbstractControl, fieldName: string): string {
    if (!control.errors) return '';

    if (control.hasError('required')) {
      return `${fieldName} is required`;
    }
    if (control.hasError('email')) {
      return `Invalid ${fieldName} format`;
    }
    // ... more error types
    
    return `Invalid ${fieldName}`;
  }
}
```

## Interview Questions

### Q1: What's the difference between Validators.min and Validators.minLength?

**Answer:** `Validators.min` validates numeric values (minimum number), while `Validators.minLength` validates string length (minimum characters). Example:
```typescript
Validators.min(18) // Age must be at least 18
Validators.minLength(3) // String must have at least 3 characters
```

### Q2: How do you add validators dynamically?

**Answer:** Use `setValidators()` (replaces) or `addValidators()` (adds), then call `updateValueAndValidity()`:
```typescript
control.setValidators([Validators.required, Validators.email]);
control.updateValueAndValidity();
```

### Q3: When should you show validation errors?

**Answer:** Show errors only after user interaction (touched or dirty):
```typescript
*ngIf="control.invalid && (control.touched || control.dirty)"
```

### Q4: What does Validators.requiredTrue do?

**Answer:** It validates that a boolean value is `true`, commonly used for checkbox agreements:
```typescript
agreeToTerms: new FormControl(false, Validators.requiredTrue)
```

### Q5: How do you clear all validators from a control?

**Answer:** Use `clearValidators()` and update:
```typescript
control.clearValidators();
control.updateValueAndValidity();
```

## Key Takeaways

1. **Built-in validators cover common cases** - Use them before creating custom validators
2. **Compose multiple validators** - Combine validators for complex validation rules
3. **Update after modifying validators** - Always call `updateValueAndValidity()`
4. **Show errors after interaction** - Check `touched` or `dirty` before displaying
5. **Error objects contain details** - Access error info for better messages
6. **Pattern validator uses RegExp** - Not strings
7. **RequiredTrue for checkboxes** - Useful for terms acceptance
8. **Create reusable error components** - DRY principle for error display
9. **Dynamic validation is powerful** - Add/remove validators based on form state
10. **Extract validation constants** - Reusable patterns and limits

## Resources

- [Angular Validators API](https://angular.io/api/forms/Validators)
- [Form Validation Guide](https://angular.io/guide/form-validation)
- [AbstractControl API](https://angular.io/api/forms/AbstractControl)
- [RegExp Reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions)
- [Form Validation Patterns](https://www.patterns.dev/posts/reactivity-patterns)
