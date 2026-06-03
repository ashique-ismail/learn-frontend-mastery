# Custom Synchronous Validators in Angular

## Overview

While Angular provides many built-in validators, real-world applications often require custom validation logic. Custom synchronous validators allow you to implement domain-specific validation rules, business logic constraints, and complex validation patterns that go beyond simple format checking.

This guide covers how to create custom validators, implement them at the control and form level, and build reusable validation functions that integrate seamlessly with Angular's forms system.

## Table of Contents

1. [ValidatorFn Interface](#validatorfn-interface)
2. [Basic Custom Validators](#basic-custom-validators)
3. [Validators with Parameters](#validators-with-parameters)
4. [Form-Level Validators](#form-level-validators)
5. [Reusable Validator Library](#reusable-validator-library)
6. [Context-Aware Validators](#context-aware-validators)
7. [Class-Based Validators](#class-based-validators)
8. [Composition and HOC Validators](#composition-and-hoc-validators)
9. [Testing Custom Validators](#testing-custom-validators)
10. [Error Message Integration](#error-message-integration)

## ValidatorFn Interface

Understanding the ValidatorFn type is fundamental to creating custom validators.

### ValidatorFn Type Signature

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// ValidatorFn type definition
type ValidatorFn = (control: AbstractControl) => ValidationErrors | null;

// ValidationErrors type
interface ValidationErrors {
  [key: string]: any;
}

// Basic custom validator
function customValidator(control: AbstractControl): ValidationErrors | null {
  const isValid = /* validation logic */;
  
  // Return null if valid
  if (isValid) {
    return null;
  }
  
  // Return error object if invalid
  return {
    customError: {
      message: 'Validation failed',
      actualValue: control.value
    }
  };
}
```

### Return Value Structure

```typescript
import { FormControl } from '@angular/forms';

function demoValidator(control: AbstractControl): ValidationErrors | null {
  // Valid: return null
  if (control.value === 'valid') {
    return null;
  }
  
  // Invalid: return object with error key
  return {
    // Error key (used to check: control.hasError('demoError'))
    demoError: {
      // Error details (accessed via control.getError('demoError'))
      message: 'Value must be "valid"',
      actualValue: control.value,
      expectedValue: 'valid'
    }
  };
}

// Usage
const control = new FormControl('', demoValidator);
control.setValue('invalid');

console.log(control.valid); // false
console.log(control.hasError('demoError')); // true
console.log(control.getError('demoError'));
// { message: '...', actualValue: 'invalid', expectedValue: 'valid' }
```

## Basic Custom Validators

Simple validators for common use cases.

### No Whitespace Validator

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function noWhitespaceValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    // Null or empty is valid (use Validators.required separately)
    if (!control.value) {
      return null;
    }

    const isWhitespace = (control.value as string).trim().length === 0;
    return isWhitespace ? { whitespace: true } : null;
  };
}

// Usage
import { Component } from '@angular/core';
import { FormControl, Validators } from '@angular/forms';

@Component({
  selector: 'app-whitespace-demo',
  template: `
    <input [formControl]="nameControl" />
    <div *ngIf="nameControl.hasError('required')">Name is required</div>
    <div *ngIf="nameControl.hasError('whitespace')">
      Name cannot be only whitespace
    </div>
  `
})
export class WhitespaceDemoComponent {
  nameControl = new FormControl('', [
    Validators.required,
    noWhitespaceValidator()
  ]);
}
```

### URL Validator

```typescript
export function urlValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    try {
      new URL(control.value);
      return null; // Valid URL
    } catch {
      return { 
        invalidUrl: { 
          message: 'Invalid URL format',
          value: control.value 
        } 
      };
    }
  };
}

// Usage
@Component({
  selector: 'app-url-demo',
  template: `
    <input [formControl]="websiteControl" placeholder="https://example.com" />
    <div *ngIf="websiteControl.hasError('invalidUrl')">
      {{ websiteControl.getError('invalidUrl').message }}
    </div>
  `
})
export class UrlDemoComponent {
  websiteControl = new FormControl('', urlValidator());
}
```

### Future Date Validator

```typescript
export function futureDateValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const inputDate = new Date(control.value);
    const today = new Date();
    today.setHours(0, 0, 0, 0); // Reset time to compare dates only

    if (inputDate < today) {
      return {
        futureDate: {
          message: 'Date must be in the future',
          actualDate: inputDate.toISOString(),
          minDate: today.toISOString()
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-appointment-form',
  template: `
    <label>Appointment Date</label>
    <input [formControl]="dateControl" type="date" />
    <div *ngIf="dateControl.hasError('futureDate')">
      Appointment must be scheduled for a future date
    </div>
  `
})
export class AppointmentFormComponent {
  dateControl = new FormControl('', [
    Validators.required,
    futureDateValidator()
  ]);
}
```

### Numeric Range Validator

```typescript
export function rangeValidator(min: number, max: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value && control.value !== 0) {
      return null;
    }

    const value = Number(control.value);

    if (isNaN(value)) {
      return { notANumber: { value: control.value } };
    }

    if (value < min || value > max) {
      return {
        range: {
          min,
          max,
          actual: value,
          message: `Value must be between ${min} and ${max}`
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-temperature-input',
  template: `
    <label>Temperature (°F)</label>
    <input [formControl]="tempControl" type="number" />
    <div *ngIf="tempControl.hasError('range')">
      Temperature must be between 
      {{ tempControl.getError('range').min }} and 
      {{ tempControl.getError('range').max }}
    </div>
  `
})
export class TemperatureInputComponent {
  tempControl = new FormControl('', rangeValidator(-50, 150));
}
```

## Validators with Parameters

Creating configurable validators with parameters.

### Username Validator

```typescript
export interface UsernameValidatorConfig {
  minLength?: number;
  maxLength?: number;
  allowNumbers?: boolean;
  allowSpecialChars?: boolean;
  reservedNames?: string[];
}

export function usernameValidator(config: UsernameValidatorConfig = {}): ValidatorFn {
  const {
    minLength = 3,
    maxLength = 20,
    allowNumbers = true,
    allowSpecialChars = false,
    reservedNames = []
  } = config;

  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const username = control.value as string;
    const errors: any = {};

    // Length check
    if (username.length < minLength) {
      errors.minLength = { required: minLength, actual: username.length };
    }
    if (username.length > maxLength) {
      errors.maxLength = { required: maxLength, actual: username.length };
    }

    // Character checks
    if (!allowNumbers && /\d/.test(username)) {
      errors.noNumbers = { message: 'Username cannot contain numbers' };
    }

    if (!allowSpecialChars && /[^a-zA-Z0-9_-]/.test(username)) {
      errors.noSpecialChars = { 
        message: 'Username can only contain letters, numbers, underscore, and hyphen' 
      };
    }

    // Reserved names check
    if (reservedNames.includes(username.toLowerCase())) {
      errors.reserved = { 
        message: 'This username is reserved',
        value: username 
      };
    }

    // Must start with letter
    if (!/^[a-zA-Z]/.test(username)) {
      errors.startWithLetter = { message: 'Username must start with a letter' };
    }

    return Object.keys(errors).length > 0 ? { username: errors } : null;
  };
}

// Usage
@Component({
  selector: 'app-registration',
  template: `
    <input [formControl]="usernameControl" placeholder="Username" />
    <div *ngIf="usernameControl.hasError('username')">
      <div *ngFor="let error of getErrors()">{{ error }}</div>
    </div>
  `
})
export class RegistrationComponent {
  usernameControl = new FormControl('', [
    Validators.required,
    usernameValidator({
      minLength: 4,
      maxLength: 15,
      allowNumbers: true,
      allowSpecialChars: false,
      reservedNames: ['admin', 'root', 'system']
    })
  ]);

  getErrors(): string[] {
    const usernameErrors = this.usernameControl.getError('username');
    if (!usernameErrors) return [];

    return Object.values(usernameErrors).map((err: any) => err.message || err);
  }
}
```

### Password Strength Validator

```typescript
export interface PasswordStrength {
  minLength?: number;
  requireUppercase?: boolean;
  requireLowercase?: boolean;
  requireNumbers?: boolean;
  requireSpecialChars?: boolean;
  minScore?: number; // 0-4
}

export function passwordStrengthValidator(config: PasswordStrength = {}): ValidatorFn {
  const {
    minLength = 8,
    requireUppercase = true,
    requireLowercase = true,
    requireNumbers = true,
    requireSpecialChars = true,
    minScore = 3
  } = config;

  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const password = control.value as string;
    const errors: string[] = [];
    let score = 0;

    // Length check
    if (password.length < minLength) {
      errors.push(`Password must be at least ${minLength} characters`);
    } else {
      score++;
      if (password.length >= minLength + 4) score++;
    }

    // Character type checks
    if (requireUppercase && !/[A-Z]/.test(password)) {
      errors.push('Password must contain an uppercase letter');
    } else if (/[A-Z]/.test(password)) {
      score++;
    }

    if (requireLowercase && !/[a-z]/.test(password)) {
      errors.push('Password must contain a lowercase letter');
    } else if (/[a-z]/.test(password)) {
      score++;
    }

    if (requireNumbers && !/\d/.test(password)) {
      errors.push('Password must contain a number');
    } else if (/\d/.test(password)) {
      score++;
    }

    if (requireSpecialChars && !/[^A-Za-z0-9]/.test(password)) {
      errors.push('Password must contain a special character');
    } else if (/[^A-Za-z0-9]/.test(password)) {
      score++;
    }

    // Calculate strength
    const strength = score >= 5 ? 'strong' : score >= 3 ? 'medium' : 'weak';

    if (errors.length > 0 || score < minScore) {
      return {
        passwordStrength: {
          errors,
          score,
          strength,
          minScore
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-password-field',
  template: `
    <input [formControl]="passwordControl" type="password" />
    
    <div class="strength-indicator" *ngIf="passwordControl.value">
      Strength: 
      <span [class]="getStrengthClass()">
        {{ getStrength() }}
      </span>
    </div>

    <div *ngIf="passwordControl.hasError('passwordStrength')">
      <ul>
        <li *ngFor="let error of getPasswordErrors()">{{ error }}</li>
      </ul>
    </div>
  `,
  styles: [`
    .weak { color: red; }
    .medium { color: orange; }
    .strong { color: green; }
  `]
})
export class PasswordFieldComponent {
  passwordControl = new FormControl('', [
    Validators.required,
    passwordStrengthValidator({
      minLength: 10,
      requireUppercase: true,
      requireLowercase: true,
      requireNumbers: true,
      requireSpecialChars: true,
      minScore: 3
    })
  ]);

  getPasswordErrors(): string[] {
    return this.passwordControl.getError('passwordStrength')?.errors || [];
  }

  getStrength(): string {
    return this.passwordControl.getError('passwordStrength')?.strength || 'strong';
  }

  getStrengthClass(): string {
    const strength = this.getStrength();
    return this.passwordControl.valid ? 'strong' : strength;
  }
}
```

### Credit Card Validator

```typescript
export function creditCardValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const cardNumber = control.value.replace(/\s/g, '');

    // Check if only digits
    if (!/^\d+$/.test(cardNumber)) {
      return { creditCard: { message: 'Card number must contain only digits' } };
    }

    // Check length (13-19 digits)
    if (cardNumber.length < 13 || cardNumber.length > 19) {
      return { creditCard: { message: 'Invalid card number length' } };
    }

    // Luhn algorithm
    let sum = 0;
    let isEven = false;

    for (let i = cardNumber.length - 1; i >= 0; i--) {
      let digit = parseInt(cardNumber[i], 10);

      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }

      sum += digit;
      isEven = !isEven;
    }

    const isValid = sum % 10 === 0;

    if (!isValid) {
      return {
        creditCard: {
          message: 'Invalid card number',
          value: control.value
        }
      };
    }

    return null;
  };
}

// Detect card type
export function detectCardType(cardNumber: string): string {
  const cleaned = cardNumber.replace(/\s/g, '');
  
  if (/^4/.test(cleaned)) return 'Visa';
  if (/^5[1-5]/.test(cleaned)) return 'Mastercard';
  if (/^3[47]/.test(cleaned)) return 'American Express';
  if (/^6(?:011|5)/.test(cleaned)) return 'Discover';
  
  return 'Unknown';
}

// Usage
@Component({
  selector: 'app-credit-card-input',
  template: `
    <input [formControl]="cardControl" 
           placeholder="1234 5678 9012 3456"
           maxlength="19" />
    
    <div *ngIf="cardControl.valid && cardControl.value">
      Card Type: {{ getCardType() }}
    </div>

    <div *ngIf="cardControl.hasError('creditCard')">
      {{ cardControl.getError('creditCard').message }}
    </div>
  `
})
export class CreditCardInputComponent {
  cardControl = new FormControl('', [
    Validators.required,
    creditCardValidator()
  ]);

  getCardType(): string {
    return detectCardType(this.cardControl.value || '');
  }
}
```

## Form-Level Validators

Validators that validate entire forms or form groups.

### Password Match Validator

```typescript
export function passwordMatchValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const password = control.get('password');
    const confirmPassword = control.get('confirmPassword');

    if (!password || !confirmPassword) {
      return null;
    }

    if (confirmPassword.errors && !confirmPassword.errors['passwordMismatch']) {
      // Return if another validator has already found an error
      return null;
    }

    if (password.value !== confirmPassword.value) {
      confirmPassword.setErrors({ passwordMismatch: true });
      return { passwordMismatch: true };
    } else {
      confirmPassword.setErrors(null);
      return null;
    }
  };
}

// Usage
@Component({
  selector: 'app-password-form',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Password</label>
        <input formControlName="password" type="password" />
      </div>

      <div>
        <label>Confirm Password</label>
        <input formControlName="confirmPassword" type="password" />
        <div *ngIf="form.controls.confirmPassword.hasError('passwordMismatch')">
          Passwords do not match
        </div>
      </div>

      <button [disabled]="form.invalid">Submit</button>
    </form>
  `
})
export class PasswordFormComponent {
  form = new FormGroup({
    password: new FormControl('', [Validators.required, Validators.minLength(8)]),
    confirmPassword: new FormControl('', Validators.required)
  }, { validators: passwordMatchValidator() });
}
```

### Date Range Validator

```typescript
export function dateRangeValidator(startDateField: string, endDateField: string): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const startDate = control.get(startDateField);
    const endDate = control.get(endDateField);

    if (!startDate || !endDate || !startDate.value || !endDate.value) {
      return null;
    }

    const start = new Date(startDate.value);
    const end = new Date(endDate.value);

    if (start > end) {
      return {
        dateRange: {
          message: 'End date must be after start date',
          startDate: start.toISOString(),
          endDate: end.toISOString()
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-event-form',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Start Date</label>
        <input formControlName="startDate" type="date" />
      </div>

      <div>
        <label>End Date</label>
        <input formControlName="endDate" type="date" />
      </div>

      <div *ngIf="form.hasError('dateRange')" class="form-error">
        {{ form.getError('dateRange').message }}
      </div>

      <button [disabled]="form.invalid">Create Event</button>
    </form>
  `
})
export class EventFormComponent {
  form = new FormGroup({
    startDate: new FormControl('', Validators.required),
    endDate: new FormControl('', Validators.required)
  }, { validators: dateRangeValidator('startDate', 'endDate') });
}
```

### Conditional Required Validator

```typescript
export function conditionalRequiredValidator(
  conditionField: string,
  conditionValue: any,
  requiredFields: string[]
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const conditionControl = control.get(conditionField);
    
    if (!conditionControl || conditionControl.value !== conditionValue) {
      return null;
    }

    const errors: { [key: string]: any } = {};

    requiredFields.forEach(field => {
      const fieldControl = control.get(field);
      if (fieldControl && !fieldControl.value) {
        errors[field] = { required: true };
        fieldControl.setErrors({ required: true });
      }
    });

    return Object.keys(errors).length > 0 ? { conditionalRequired: errors } : null;
  };
}

// Usage
@Component({
  selector: 'app-shipping-form',
  template: `
    <form [formGroup]="form">
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
        <div *ngIf="form.controls.deliveryAddress.hasError('required')">
          Delivery address is required for delivery
        </div>

        <label>Delivery Phone</label>
        <input formControlName="deliveryPhone" />
        <div *ngIf="form.controls.deliveryPhone.hasError('required')">
          Phone number is required for delivery
        </div>
      </div>
    </form>
  `
})
export class ShippingFormComponent {
  form = new FormGroup({
    shippingMethod: new FormControl('pickup', { nonNullable: true }),
    deliveryAddress: new FormControl(''),
    deliveryPhone: new FormControl('')
  }, {
    validators: conditionalRequiredValidator(
      'shippingMethod',
      'delivery',
      ['deliveryAddress', 'deliveryPhone']
    )
  });
}
```

## Reusable Validator Library

Creating a comprehensive validator library.

### Validators Library

```typescript
// validators.ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export class CustomValidators {
  // String validators
  static noWhitespace(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const isWhitespace = (control.value as string).trim().length === 0;
      return isWhitespace ? { whitespace: true } : null;
    };
  }

  static alphaNumeric(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const valid = /^[a-zA-Z0-9]+$/.test(control.value);
      return valid ? null : { alphaNumeric: true };
    };
  }

  static noSpecialChars(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const valid = /^[a-zA-Z0-9\s]+$/.test(control.value);
      return valid ? null : { noSpecialChars: true };
    };
  }

  // Numeric validators
  static integer(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value && control.value !== 0) return null;
      const valid = Number.isInteger(Number(control.value));
      return valid ? null : { integer: true };
    };
  }

  static positive(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value && control.value !== 0) return null;
      const valid = Number(control.value) > 0;
      return valid ? null : { positive: true };
    };
  }

  static divisibleBy(divisor: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value && control.value !== 0) return null;
      const value = Number(control.value);
      const valid = value % divisor === 0;
      return valid ? null : { divisibleBy: { divisor, value } };
    };
  }

  // Date validators
  static minDate(minDate: Date): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const inputDate = new Date(control.value);
      const valid = inputDate >= minDate;
      return valid ? null : { minDate: { min: minDate, actual: inputDate } };
    };
  }

  static maxDate(maxDate: Date): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const inputDate = new Date(control.value);
      const valid = inputDate <= maxDate;
      return valid ? null : { maxDate: { max: maxDate, actual: inputDate } };
    };
  }

  static ageRange(minAge: number, maxAge: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      
      const birthDate = new Date(control.value);
      const today = new Date();
      let age = today.getFullYear() - birthDate.getFullYear();
      const monthDiff = today.getMonth() - birthDate.getMonth();
      
      if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
        age--;
      }

      if (age < minAge || age > maxAge) {
        return { ageRange: { minAge, maxAge, actualAge: age } };
      }

      return null;
    };
  }

  // File validators
  static fileSize(maxSizeInBytes: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const file = control.value as File;
      if (!file) return null;

      if (file.size > maxSizeInBytes) {
        return {
          fileSize: {
            maxSize: maxSizeInBytes,
            actualSize: file.size,
            maxSizeMB: (maxSizeInBytes / (1024 * 1024)).toFixed(2),
            actualSizeMB: (file.size / (1024 * 1024)).toFixed(2)
          }
        };
      }

      return null;
    };
  }

  static fileType(allowedTypes: string[]): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const file = control.value as File;
      if (!file) return null;

      const fileType = file.type;
      const valid = allowedTypes.includes(fileType);

      return valid ? null : { 
        fileType: { 
          allowed: allowedTypes, 
          actual: fileType 
        } 
      };
    };
  }

  // Array validators
  static minItems(min: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const array = control.value as any[];
      return array.length >= min ? null : { minItems: { min, actual: array.length } };
    };
  }

  static maxItems(max: number): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const array = control.value as any[];
      return array.length <= max ? null : { maxItems: { max, actual: array.length } };
    };
  }

  static uniqueItems(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) return null;
      const array = control.value as any[];
      const unique = new Set(array);
      return unique.size === array.length ? null : { uniqueItems: true };
    };
  }
}
```

### Usage Examples

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';
import { CustomValidators } from './validators';

@Component({
  selector: 'app-demo',
  template: `
    <form [formGroup]="form">
      <!-- String validators -->
      <input formControlName="username" />
      <input formControlName="displayName" />

      <!-- Numeric validators -->
      <input formControlName="quantity" type="number" />
      <input formControlName="amount" type="number" />

      <!-- Date validators -->
      <input formControlName="birthDate" type="date" />
      <input formControlName="appointmentDate" type="date" />

      <!-- Array validators -->
      <select formControlName="tags" multiple>
        <option value="angular">Angular</option>
        <option value="react">React</option>
        <option value="vue">Vue</option>
      </select>
    </form>
  `
})
export class DemoComponent {
  form = new FormGroup({
    username: new FormControl('', [
      Validators.required,
      CustomValidators.noWhitespace(),
      CustomValidators.alphaNumeric(),
      Validators.minLength(3)
    ]),
    displayName: new FormControl('', [
      CustomValidators.noSpecialChars()
    ]),
    quantity: new FormControl<number | null>(null, [
      Validators.required,
      CustomValidators.integer(),
      CustomValidators.positive(),
      CustomValidators.divisibleBy(5)
    ]),
    amount: new FormControl<number | null>(null, [
      CustomValidators.positive()
    ]),
    birthDate: new FormControl('', [
      Validators.required,
      CustomValidators.ageRange(18, 100)
    ]),
    appointmentDate: new FormControl('', [
      CustomValidators.minDate(new Date())
    ]),
    tags: new FormControl<string[]>([], [
      CustomValidators.minItems(1),
      CustomValidators.maxItems(5),
      CustomValidators.uniqueItems()
    ])
  });
}
```

## Context-Aware Validators

Validators that depend on external context or services.

### Existing Email Validator (with Service)

```typescript
import { Injectable } from '@angular/core';
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

@Injectable({ providedIn: 'root' })
export class UserService {
  private existingEmails = ['user@example.com', 'admin@example.com'];

  emailExists(email: string): boolean {
    return this.existingEmails.includes(email.toLowerCase());
  }
}

// Validator factory that injects service
export function uniqueEmailValidator(userService: UserService): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const exists = userService.emailExists(control.value);
    return exists ? { emailTaken: { value: control.value } } : null;
  };
}

// Usage
@Component({
  selector: 'app-registration',
  template: `
    <input [formControl]="emailControl" type="email" />
    <div *ngIf="emailControl.hasError('emailTaken')">
      This email is already registered
    </div>
  `
})
export class RegistrationComponent {
  emailControl: FormControl;

  constructor(private userService: UserService) {
    this.emailControl = new FormControl('', [
      Validators.required,
      Validators.email,
      uniqueEmailValidator(this.userService)
    ]);
  }
}
```

### Business Hours Validator

```typescript
export interface BusinessHoursConfig {
  startHour: number;  // 0-23
  endHour: number;    // 0-23
  workingDays: number[]; // 0-6 (0 = Sunday)
}

export function businessHoursValidator(config: BusinessHoursConfig): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;
    }

    const selectedDate = new Date(control.value);
    const dayOfWeek = selectedDate.getDay();
    const hour = selectedDate.getHours();

    // Check if day is a working day
    if (!config.workingDays.includes(dayOfWeek)) {
      return {
        businessHours: {
          message: 'Selected day is not a working day',
          selectedDay: dayOfWeek,
          workingDays: config.workingDays
        }
      };
    }

    // Check if hour is within business hours
    if (hour < config.startHour || hour >= config.endHour) {
      return {
        businessHours: {
          message: 'Selected time is outside business hours',
          selectedHour: hour,
          businessHours: `${config.startHour}:00 - ${config.endHour}:00`
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-appointment-booking',
  template: `
    <input [formControl]="appointmentControl" type="datetime-local" />
    <div *ngIf="appointmentControl.hasError('businessHours')">
      {{ appointmentControl.getError('businessHours').message }}
    </div>
  `
})
export class AppointmentBookingComponent {
  appointmentControl = new FormControl('', [
    Validators.required,
    businessHoursValidator({
      startHour: 9,
      endHour: 17,
      workingDays: [1, 2, 3, 4, 5] // Monday to Friday
    })
  ]);
}
```

## Class-Based Validators

Creating validators as injectable services.

### Class Validator

```typescript
import { Injectable } from '@angular/core';
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

@Injectable({ providedIn: 'root' })
export class EmailDomainValidator {
  private allowedDomains: string[] = [];

  setAllowedDomains(domains: string[]) {
    this.allowedDomains = domains;
  }

  validator(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      if (!control.value) {
        return null;
      }

      const email = control.value as string;
      const domain = email.split('@')[1];

      if (!domain || !this.allowedDomains.includes(domain)) {
        return {
          emailDomain: {
            allowedDomains: this.allowedDomains,
            actualDomain: domain
          }
        };
      }

      return null;
    };
  }
}

// Usage
@Component({
  selector: 'app-company-signup',
  template: `
    <input [formControl]="emailControl" type="email" />
    <div *ngIf="emailControl.hasError('emailDomain')">
      Email must be from one of these domains: 
      {{ emailControl.getError('emailDomain').allowedDomains.join(', ') }}
    </div>
  `
})
export class CompanySignupComponent {
  emailControl: FormControl;

  constructor(private emailDomainValidator: EmailDomainValidator) {
    // Configure allowed domains
    this.emailDomainValidator.setAllowedDomains([
      'company.com',
      'company.co.uk',
      'subsidiary.com'
    ]);

    this.emailControl = new FormControl('', [
      Validators.required,
      Validators.email,
      this.emailDomainValidator.validator()
    ]);
  }
}
```

## Composition and HOC Validators

Combining validators for reusability.

### Compose Validators

```typescript
export function composeValidators(...validators: ValidatorFn[]): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const errors: ValidationErrors = {};

    validators.forEach(validator => {
      const error = validator(control);
      if (error) {
        Object.assign(errors, error);
      }
    });

    return Object.keys(errors).length > 0 ? errors : null;
  };
}

// Usage
const strongPasswordValidator = composeValidators(
  Validators.required,
  Validators.minLength(12),
  CustomValidators.uppercase(),
  CustomValidators.lowercase(),
  CustomValidators.digit(),
  CustomValidators.specialChar()
);

const passwordControl = new FormControl('', strongPasswordValidator);
```

### Conditional Validator HOC

```typescript
export function conditionalValidator(
  condition: (control: AbstractControl) => boolean,
  validator: ValidatorFn
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (condition(control)) {
      return validator(control);
    }
    return null;
  };
}

// Usage
const emailControl = new FormControl('', [
  conditionalValidator(
    (control) => control.parent?.get('contactMethod')?.value === 'email',
    Validators.compose([Validators.required, Validators.email])!
  )
]);
```

## Testing Custom Validators

How to test custom validators.

### Unit Tests

```typescript
import { FormControl } from '@angular/forms';
import { noWhitespaceValidator } from './validators';

describe('noWhitespaceValidator', () => {
  it('should return null for valid value', () => {
    const control = new FormControl('valid', noWhitespaceValidator());
    expect(control.errors).toBeNull();
  });

  it('should return error for whitespace only', () => {
    const control = new FormControl('   ', noWhitespaceValidator());
    expect(control.errors).toEqual({ whitespace: true });
  });

  it('should return null for empty value', () => {
    const control = new FormControl('', noWhitespaceValidator());
    expect(control.errors).toBeNull();
  });

  it('should allow spaces within text', () => {
    const control = new FormControl('hello world', noWhitespaceValidator());
    expect(control.errors).toBeNull();
  });
});

describe('passwordMatchValidator', () => {
  it('should return null when passwords match', () => {
    const form = new FormGroup({
      password: new FormControl('password123'),
      confirmPassword: new FormControl('password123')
    }, { validators: passwordMatchValidator() });

    expect(form.errors).toBeNull();
  });

  it('should return error when passwords do not match', () => {
    const form = new FormGroup({
      password: new FormControl('password123'),
      confirmPassword: new FormControl('different')
    }, { validators: passwordMatchValidator() });

    expect(form.errors).toEqual({ passwordMismatch: true });
  });
});
```

## Error Message Integration

Integrating validators with error messages.

### Error Message Service

```typescript
@Injectable({ providedIn: 'root' })
export class ValidationErrorService {
  getErrorMessage(control: AbstractControl, fieldName: string): string {
    if (!control.errors) return '';

    const errors = control.errors;

    // Built-in validators
    if (errors['required']) return `${fieldName} is required`;
    if (errors['email']) return `Invalid ${fieldName} format`;
    if (errors['minlength']) {
      return `${fieldName} must be at least ${errors['minlength'].requiredLength} characters`;
    }
    if (errors['maxlength']) {
      return `${fieldName} cannot exceed ${errors['maxlength'].requiredLength} characters`;
    }

    // Custom validators
    if (errors['whitespace']) return `${fieldName} cannot be only whitespace`;
    if (errors['alphaNumeric']) return `${fieldName} can only contain letters and numbers`;
    if (errors['positive']) return `${fieldName} must be a positive number`;
    if (errors['creditCard']) return errors['creditCard'].message;
    if (errors['passwordStrength']) {
      return errors['passwordStrength'].errors[0] || 'Password is too weak';
    }

    return `Invalid ${fieldName}`;
  }
}
```

## Common Mistakes

### 1. Not Returning Null for Valid Values

```typescript
// WRONG
function badValidator(control: AbstractControl): ValidationErrors | null {
  return control.value === 'valid' ? {} : { invalid: true };
}

// CORRECT
function goodValidator(control: AbstractControl): ValidationErrors | null {
  return control.value === 'valid' ? null : { invalid: true };
}
```

### 2. Not Handling Empty Values

```typescript
// WRONG: Crashes if value is null/undefined
function badValidator(control: AbstractControl): ValidationErrors | null {
  return control.value.length > 5 ? null : { tooShort: true };
}

// CORRECT: Check for empty values first
function goodValidator(control: AbstractControl): ValidationErrors | null {
  if (!control.value) return null;
  return control.value.length > 5 ? null : { tooShort: true };
}
```

### 3. Calling updateValueAndValidity in Validator

```typescript
// WRONG: Causes infinite loop
function badValidator(control: AbstractControl): ValidationErrors | null {
  control.updateValueAndValidity(); // DON'T DO THIS
  return null;
}

// CORRECT: Validators should be pure functions
function goodValidator(control: AbstractControl): ValidationErrors | null {
  return someCheck(control.value) ? null : { invalid: true };
}
```

## Best Practices

1. **Return null for valid values** - Not empty object
2. **Check for null/undefined values first** - Before accessing value properties
3. **Make validators pure functions** - No side effects
4. **Provide detailed error objects** - Include context for error messages
5. **Use factory functions for configurable validators** - Enable parameter passing
6. **Test validators thoroughly** - Unit test all edge cases
7. **Document validator behavior** - Clear JSDoc comments
8. **Reuse validators in a library** - DRY principle
9. **Consider performance** - Avoid expensive operations in validators
10. **Combine with built-in validators** - Don't reinvent the wheel

## Interview Questions

### Q1: What should a validator function return for valid/invalid values?

**Answer:** Return `null` for valid values and a `ValidationErrors` object for invalid values:
```typescript
// Valid
return null;

// Invalid
return { errorKey: { details } };
```

### Q2: How do you create a validator with parameters?

**Answer:** Use a factory function that returns the validator:
```typescript
function minAge(age: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    // validation logic using age parameter
  };
}
```

### Q3: What's the difference between control-level and form-level validators?

**Answer:** Control-level validators validate individual controls, while form-level validators validate the entire form/group and can access multiple fields:
```typescript
// Control-level
new FormControl('', myValidator())

// Form-level
new FormGroup({...}, { validators: myFormValidator() })
```

### Q4: How do you test custom validators?

**Answer:** Create a FormControl with the validator and check the errors property:
```typescript
const control = new FormControl('value', myValidator());
expect(control.errors).toEqual({ errorKey: {...} });
```

### Q5: Should validators have side effects?

**Answer:** No, validators should be pure functions with no side effects. They should only read the control value and return validation results.

## Key Takeaways

1. **ValidatorFn returns null or ValidationErrors** - Null means valid
2. **Factory functions enable parameters** - Return ValidatorFn from function
3. **Always check for null/undefined** - Before accessing value properties
4. **Form-level validators access multiple fields** - Use for cross-field validation
5. **Validators should be pure** - No side effects or external mutations
6. **Error objects provide context** - Include useful info for messages
7. **Test validators in isolation** - Easy to unit test
8. **Reuse validators across projects** - Build a library
9. **Combine built-in and custom validators** - Use Validators.compose
10. **Document validator requirements** - Clear expectations

## Resources

- [Angular Validators API](https://angular.io/api/forms/Validators)
- [ValidatorFn Interface](https://angular.io/api/forms/ValidatorFn)
- [Custom Validators Guide](https://angular.io/guide/form-validation#custom-validators)
- [ValidationErrors Type](https://angular.io/api/forms/ValidationErrors)
- [Testing Forms](https://angular.io/guide/testing-components-scenarios#forms)
