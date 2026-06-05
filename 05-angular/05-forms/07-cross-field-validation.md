# Cross-Field Validation in Angular

## The Idea

**In plain English:** Cross-field validation is when a form checks that two or more answers work together correctly — not just that each answer looks right on its own. For example, making sure the password you typed twice is the same both times, or that a trip end date is not before the start date.

**Real-world analogy:** Imagine filling out a paper form to book a hotel room. The desk clerk checks not just each line by itself, but also whether your check-out date is after your check-in date. If you wrote "check in January 10, check out January 5," the clerk would reject the whole form even though both dates look like valid dates individually.

- The hotel booking form = the Angular FormGroup
- The check-in and check-out date fields = the individual form controls being compared
- The desk clerk's combined check = the cross-field validator function that reads both fields and decides if the combination is valid

---

## Overview

Cross-field validation validates relationships between multiple form fields, such as ensuring passwords match, end dates come after start dates, or conditional requirements based on other field values. These validators operate at the FormGroup level and can access multiple controls simultaneously.

This guide covers how to implement form-level validators, compare fields, create conditional validation rules, and build complex multi-field validation logic.

## Table of Contents

1. [Form-Level Validators](#form-level-validators)
2. [Password Match Validation](#password-match-validation)
3. [Date Range Validation](#date-range-validation)
4. [Conditional Required Fields](#conditional-required-fields)
5. [Interdependent Fields](#interdependent-fields)
6. [Complex Business Rules](#complex-business-rules)
7. [Nested FormGroup Validation](#nested-formgroup-validation)
8. [Displaying Cross-Field Errors](#displaying-cross-field-errors)
9. [Dynamic Cross-Field Validation](#dynamic-cross-field-validation)
10. [Testing Cross-Field Validators](#testing-cross-field-validators)

## Form-Level Validators

Understanding how to create validators at the form group level.

### Basic Form Validator Structure

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

// Form-level validator function
export function formLevelValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    // control is the FormGroup
    const field1 = control.get('field1');
    const field2 = control.get('field2');

    if (!field1 || !field2) {
      return null;
    }

    // Compare fields
    if (field1.value !== field2.value) {
      return { mismatch: true };
    }

    return null;
  };
}

// Usage
import { FormGroup, FormControl } from '@angular/forms';

const form = new FormGroup({
  field1: new FormControl(''),
  field2: new FormControl('')
}, { validators: formLevelValidator() });

// Check form-level errors
console.log(form.errors); // { mismatch: true } or null

// Check if specific error exists
console.log(form.hasError('mismatch')); // true or false
```

### Accessing Child Controls

```typescript
export function compareFieldsValidator(field1Name: string, field2Name: string): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const field1 = control.get(field1Name);
    const field2 = control.get(field2Name);

    // Always check if controls exist
    if (!field1 || !field2) {
      return null;
    }

    // Wait until both have values
    if (!field1.value || !field2.value) {
      return null;
    }

    // Compare values
    if (field1.value !== field2.value) {
      return {
        fieldsMismatch: {
          field1: field1Name,
          field2: field2Name,
          value1: field1.value,
          value2: field2.value
        }
      };
    }

    return null;
  };
}

// Usage
const form = new FormGroup({
  email: new FormControl(''),
  confirmEmail: new FormControl('')
}, { validators: compareFieldsValidator('email', 'confirmEmail') });
```

## Password Match Validation

The most common cross-field validation pattern.

### Basic Password Match

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function passwordMatchValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const password = control.get('password');
    const confirmPassword = control.get('confirmPassword');

    if (!password || !confirmPassword) {
      return null;
    }

    // Don't validate if confirmPassword is empty (let required validator handle it)
    if (!confirmPassword.value) {
      return null;
    }

    if (password.value !== confirmPassword.value) {
      // Set error on confirmPassword control
      confirmPassword.setErrors({ passwordMismatch: true });
      return { passwordMismatch: true };
    } else {
      // Clear the error if passwords match
      // But preserve other errors
      const errors = confirmPassword.errors;
      if (errors) {
        delete errors['passwordMismatch'];
        confirmPassword.setErrors(Object.keys(errors).length > 0 ? errors : null);
      }
    }

    return null;
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
        <div *ngIf="form.controls.password.invalid && form.controls.password.touched">
          <span *ngIf="form.controls.password.hasError('required')">
            Password is required
          </span>
          <span *ngIf="form.controls.password.hasError('minlength')">
            Password must be at least 8 characters
          </span>
        </div>
      </div>

      <div>
        <label>Confirm Password</label>
        <input formControlName="confirmPassword" type="password" />
        <div *ngIf="form.controls.confirmPassword.invalid && form.controls.confirmPassword.touched">
          <span *ngIf="form.controls.confirmPassword.hasError('required')">
            Please confirm password
          </span>
          <span *ngIf="form.controls.confirmPassword.hasError('passwordMismatch')">
            Passwords do not match
          </span>
        </div>
      </div>

      <button [disabled]="form.invalid">Submit</button>
    </form>
  `
})
export class PasswordFormComponent {
  form = new FormGroup({
    password: new FormControl('', [
      Validators.required,
      Validators.minLength(8)
    ]),
    confirmPassword: new FormControl('', Validators.required)
  }, { validators: passwordMatchValidator() });
}
```

### Password Match with Real-Time Updates

```typescript
export function passwordMatchValidatorV2(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const password = control.get('password');
    const confirmPassword = control.get('confirmPassword');

    if (!password || !confirmPassword) {
      return null;
    }

    // Skip if either field is empty
    if (!password.value || !confirmPassword.value) {
      return null;
    }

    const passwordsMatch = password.value === confirmPassword.value;

    if (!passwordsMatch) {
      confirmPassword.setErrors({ 
        ...confirmPassword.errors,
        passwordMismatch: true 
      });
      return { passwordMismatch: true };
    } else {
      // Remove only passwordMismatch error, keep others
      if (confirmPassword.hasError('passwordMismatch')) {
        const { passwordMismatch, ...otherErrors } = confirmPassword.errors!;
        confirmPassword.setErrors(
          Object.keys(otherErrors).length > 0 ? otherErrors : null
        );
      }
    }

    return null;
  };
}

// Usage with value changes subscription
@Component({
  selector: 'app-password-form-v2',
  template: `
    <form [formGroup]="form">
      <input formControlName="password" type="password" />
      <input formControlName="confirmPassword" type="password" />
      
      <div class="match-indicator" *ngIf="form.value.password && form.value.confirmPassword">
        <span *ngIf="!form.hasError('passwordMismatch')" class="success">
          ✓ Passwords match
        </span>
        <span *ngIf="form.hasError('passwordMismatch')" class="error">
          ✗ Passwords do not match
        </span>
      </div>
    </form>
  `
})
export class PasswordFormV2Component {
  form = new FormGroup({
    password: new FormControl(''),
    confirmPassword: new FormControl('')
  }, { validators: passwordMatchValidatorV2() });

  ngOnInit() {
    // Re-validate confirmPassword when password changes
    this.form.controls.password.valueChanges.subscribe(() => {
      this.form.controls.confirmPassword.updateValueAndValidity();
    });
  }
}
```

## Date Range Validation

Validating date relationships.

### Start and End Date Validator

```typescript
export function dateRangeValidator(
  startDateField: string,
  endDateField: string,
  allowSameDay: boolean = true
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const startDate = control.get(startDateField);
    const endDate = control.get(endDateField);

    if (!startDate || !endDate) {
      return null;
    }

    if (!startDate.value || !endDate.value) {
      return null;
    }

    const start = new Date(startDate.value);
    const end = new Date(endDate.value);

    // Normalize to compare dates only (ignore time)
    start.setHours(0, 0, 0, 0);
    end.setHours(0, 0, 0, 0);

    const isValid = allowSameDay ? end >= start : end > start;

    if (!isValid) {
      return {
        dateRange: {
          startDate: start.toISOString(),
          endDate: end.toISOString(),
          allowSameDay,
          message: allowSameDay 
            ? 'End date must be on or after start date'
            : 'End date must be after start date'
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
  }, { validators: dateRangeValidator('startDate', 'endDate', true) });
}
```

### Date and Time Range Validator

```typescript
export function dateTimeRangeValidator(
  startField: string,
  endField: string,
  minDurationMinutes?: number
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const start = control.get(startField);
    const end = control.get(endField);

    if (!start || !end || !start.value || !end.value) {
      return null;
    }

    const startDateTime = new Date(start.value);
    const endDateTime = new Date(end.value);

    if (endDateTime <= startDateTime) {
      return {
        dateTimeRange: {
          message: 'End time must be after start time'
        }
      };
    }

    if (minDurationMinutes) {
      const durationMinutes = (endDateTime.getTime() - startDateTime.getTime()) / (1000 * 60);
      if (durationMinutes < minDurationMinutes) {
        return {
          minDuration: {
            required: minDurationMinutes,
            actual: Math.round(durationMinutes),
            message: `Duration must be at least ${minDurationMinutes} minutes`
          }
        };
      }
    }

    return null;
  };
}

// Usage for appointment booking
@Component({
  selector: 'app-appointment-booking',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Start Time</label>
        <input formControlName="startTime" type="datetime-local" />
      </div>

      <div>
        <label>End Time</label>
        <input formControlName="endTime" type="datetime-local" />
      </div>

      <div *ngIf="form.hasError('dateTimeRange')" class="error">
        {{ form.getError('dateTimeRange').message }}
      </div>

      <div *ngIf="form.hasError('minDuration')" class="error">
        {{ form.getError('minDuration').message }}
      </div>

      <div *ngIf="form.valid && form.value.startTime && form.value.endTime">
        Duration: {{ calculateDuration() }} minutes
      </div>
    </form>
  `
})
export class AppointmentBookingComponent {
  form = new FormGroup({
    startTime: new FormControl('', Validators.required),
    endTime: new FormControl('', Validators.required)
  }, { validators: dateTimeRangeValidator('startTime', 'endTime', 30) });

  calculateDuration(): number {
    const start = new Date(this.form.value.startTime!);
    const end = new Date(this.form.value.endTime!);
    return Math.round((end.getTime() - start.getTime()) / (1000 * 60));
  }
}
```

## Conditional Required Fields

Fields that are required based on other field values.

### Conditional Required Validator

```typescript
export function conditionalRequiredValidator(
  triggerField: string,
  triggerValue: any,
  requiredFields: string[]
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const trigger = control.get(triggerField);
    
    if (!trigger || trigger.value !== triggerValue) {
      // Condition not met, clear errors
      requiredFields.forEach(fieldName => {
        const field = control.get(fieldName);
        if (field?.hasError('conditionalRequired')) {
          const { conditionalRequired, ...otherErrors } = field.errors!;
          field.setErrors(Object.keys(otherErrors).length > 0 ? otherErrors : null);
        }
      });
      return null;
    }

    // Condition met, check required fields
    const errors: { [key: string]: any } = {};

    requiredFields.forEach(fieldName => {
      const field = control.get(fieldName);
      if (field && !field.value) {
        field.setErrors({ 
          ...field.errors,
          conditionalRequired: true 
        });
        errors[fieldName] = { required: true };
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
          <option value="pickup">Store Pickup</option>
          <option value="delivery">Home Delivery</option>
        </select>
      </div>

      <div *ngIf="form.value.shippingMethod === 'delivery'">
        <h3>Delivery Information</h3>
        
        <div>
          <label>Street Address</label>
          <input formControlName="deliveryAddress" />
          <div *ngIf="form.controls.deliveryAddress.hasError('conditionalRequired')">
            Address is required for delivery
          </div>
        </div>

        <div>
          <label>City</label>
          <input formControlName="deliveryCity" />
          <div *ngIf="form.controls.deliveryCity.hasError('conditionalRequired')">
            City is required for delivery
          </div>
        </div>

        <div>
          <label>Phone Number</label>
          <input formControlName="deliveryPhone" />
          <div *ngIf="form.controls.deliveryPhone.hasError('conditionalRequired')">
            Phone is required for delivery
          </div>
        </div>
      </div>

      <button [disabled]="form.invalid">Continue</button>
    </form>
  `
})
export class ShippingFormComponent {
  form = new FormGroup({
    shippingMethod: new FormControl('pickup', { nonNullable: true }),
    deliveryAddress: new FormControl(''),
    deliveryCity: new FormControl(''),
    deliveryPhone: new FormControl('')
  }, {
    validators: conditionalRequiredValidator(
      'shippingMethod',
      'delivery',
      ['deliveryAddress', 'deliveryCity', 'deliveryPhone']
    )
  });
}
```

### Multiple Condition Validator

```typescript
export function multiConditionRequiredValidator(
  conditions: Array<{
    field: string;
    value: any;
    requiredFields: string[];
  }>
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    let hasErrors = false;

    conditions.forEach(({ field, value, requiredFields }) => {
      const triggerControl = control.get(field);
      
      if (triggerControl && triggerControl.value === value) {
        requiredFields.forEach(requiredField => {
          const fieldControl = control.get(requiredField);
          if (fieldControl && !fieldControl.value) {
            fieldControl.setErrors({
              ...fieldControl.errors,
              conditionalRequired: { trigger: field, triggerValue: value }
            });
            hasErrors = true;
          }
        });
      }
    });

    return hasErrors ? { multiConditionRequired: true } : null;
  };
}

// Usage
const form = new FormGroup({
  paymentMethod: new FormControl(''),
  creditCardNumber: new FormControl(''),
  paypalEmail: new FormControl(''),
  bankAccount: new FormControl('')
}, {
  validators: multiConditionRequiredValidator([
    {
      field: 'paymentMethod',
      value: 'credit-card',
      requiredFields: ['creditCardNumber']
    },
    {
      field: 'paymentMethod',
      value: 'paypal',
      requiredFields: ['paypalEmail']
    },
    {
      field: 'paymentMethod',
      value: 'bank-transfer',
      requiredFields: ['bankAccount']
    }
  ])
});
```

## Interdependent Fields

Fields that depend on each other's values.

### Min/Max Range Validator

```typescript
export function numericRangeValidator(
  minField: string,
  maxField: string
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const min = control.get(minField);
    const max = control.get(maxField);

    if (!min || !max || !min.value || !max.value) {
      return null;
    }

    const minValue = Number(min.value);
    const maxValue = Number(max.value);

    if (isNaN(minValue) || isNaN(maxValue)) {
      return null;
    }

    if (minValue > maxValue) {
      return {
        rangeInvalid: {
          min: minValue,
          max: maxValue,
          message: 'Maximum value must be greater than minimum value'
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-price-filter',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Min Price</label>
        <input formControlName="minPrice" type="number" />
      </div>

      <div>
        <label>Max Price</label>
        <input formControlName="maxPrice" type="number" />
      </div>

      <div *ngIf="form.hasError('rangeInvalid')" class="error">
        {{ form.getError('rangeInvalid').message }}
      </div>

      <button [disabled]="form.invalid">Apply Filter</button>
    </form>
  `
})
export class PriceFilterComponent {
  form = new FormGroup({
    minPrice: new FormControl<number | null>(null),
    maxPrice: new FormControl<number | null>(null)
  }, { validators: numericRangeValidator('minPrice', 'maxPrice') });
}
```

### At Least One Required Validator

```typescript
export function atLeastOneRequiredValidator(fields: string[]): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const hasValue = fields.some(fieldName => {
      const field = control.get(fieldName);
      return field && field.value;
    });

    if (!hasValue) {
      return {
        atLeastOneRequired: {
          fields,
          message: `At least one of these fields is required: ${fields.join(', ')}`
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-contact-form',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Email</label>
        <input formControlName="email" type="email" />
      </div>

      <div>
        <label>Phone</label>
        <input formControlName="phone" type="tel" />
      </div>

      <div *ngIf="form.hasError('atLeastOneRequired') && form.touched" class="error">
        Please provide at least one contact method (email or phone)
      </div>

      <button [disabled]="form.invalid">Submit</button>
    </form>
  `
})
export class ContactFormComponent {
  form = new FormGroup({
    email: new FormControl(''),
    phone: new FormControl('')
  }, { validators: atLeastOneRequiredValidator(['email', 'phone']) });
}
```

## Complex Business Rules

Advanced validation scenarios.

### Budget Allocation Validator

```typescript
export function budgetAllocationValidator(
  totalBudgetField: string,
  allocationFields: string[]
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const totalBudget = control.get(totalBudgetField);
    
    if (!totalBudget || !totalBudget.value) {
      return null;
    }

    const total = Number(totalBudget.value);
    let allocated = 0;

    allocationFields.forEach(fieldName => {
      const field = control.get(fieldName);
      if (field && field.value) {
        allocated += Number(field.value);
      }
    });

    if (allocated > total) {
      return {
        budgetExceeded: {
          total,
          allocated,
          difference: allocated - total,
          message: `Budget exceeded by $${(allocated - total).toFixed(2)}`
        }
      };
    }

    if (allocated < total) {
      return {
        budgetUnderAllocated: {
          total,
          allocated,
          remaining: total - allocated,
          message: `$${(total - allocated).toFixed(2)} remaining to allocate`
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-budget-form',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Total Budget</label>
        <input formControlName="totalBudget" type="number" />
      </div>

      <h3>Allocations</h3>
      <div>
        <label>Marketing</label>
        <input formControlName="marketing" type="number" />
      </div>
      <div>
        <label>Development</label>
        <input formControlName="development" type="number" />
      </div>
      <div>
        <label>Operations</label>
        <input formControlName="operations" type="number" />
      </div>

      <div class="budget-summary">
        <div *ngIf="form.hasError('budgetExceeded')" class="error">
          {{ form.getError('budgetExceeded').message }}
        </div>
        <div *ngIf="form.hasError('budgetUnderAllocated')" class="warning">
          {{ form.getError('budgetUnderAllocated').message }}
        </div>
        <div *ngIf="form.valid" class="success">
          ✓ Budget fully allocated
        </div>
      </div>

      <button [disabled]="form.invalid">Submit Budget</button>
    </form>
  `
})
export class BudgetFormComponent {
  form = new FormGroup({
    totalBudget: new FormControl<number | null>(null, Validators.required),
    marketing: new FormControl<number>(0, { nonNullable: true }),
    development: new FormControl<number>(0, { nonNullable: true }),
    operations: new FormControl<number>(0, { nonNullable: true })
  }, {
    validators: budgetAllocationValidator(
      'totalBudget',
      ['marketing', 'development', 'operations']
    )
  });
}
```

### Age and Date of Birth Validator

```typescript
export function ageDateValidator(
  dateOfBirthField: string,
  ageField: string
): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const dob = control.get(dateOfBirthField);
    const age = control.get(ageField);

    if (!dob || !age || !dob.value || !age.value) {
      return null;
    }

    const birthDate = new Date(dob.value);
    const today = new Date();
    
    let calculatedAge = today.getFullYear() - birthDate.getFullYear();
    const monthDiff = today.getMonth() - birthDate.getMonth();
    
    if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
      calculatedAge--;
    }

    const enteredAge = Number(age.value);

    if (calculatedAge !== enteredAge) {
      return {
        ageMismatch: {
          calculated: calculatedAge,
          entered: enteredAge,
          message: `Age doesn't match date of birth. Calculated age is ${calculatedAge}`
        }
      };
    }

    return null;
  };
}
```

## Nested FormGroup Validation

Validating nested form structures.

### Nested Group Validator

```typescript
export function addressValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const street = control.get('street');
    const city = control.get('city');
    const state = control.get('state');
    const zipCode = control.get('zipCode');

    // All or none pattern - either complete address or empty
    const values = [street?.value, city?.value, state?.value, zipCode?.value];
    const filledCount = values.filter(v => v).length;

    if (filledCount > 0 && filledCount < 4) {
      return {
        incompleteAddress: {
          message: 'Please complete all address fields or leave them empty',
          filled: filledCount,
          required: 4
        }
      };
    }

    return null;
  };
}

// Usage
@Component({
  selector: 'app-user-profile',
  template: `
    <form [formGroup]="form">
      <div formGroupName="billingAddress">
        <h3>Billing Address</h3>
        <input formControlName="street" placeholder="Street" />
        <input formControlName="city" placeholder="City" />
        <input formControlName="state" placeholder="State" />
        <input formControlName="zipCode" placeholder="ZIP" />
        
        <div *ngIf="billingAddress.hasError('incompleteAddress')" class="error">
          {{ billingAddress.getError('incompleteAddress').message }}
        </div>
      </div>

      <div formGroupName="shippingAddress">
        <h3>Shipping Address</h3>
        <input formControlName="street" placeholder="Street" />
        <input formControlName="city" placeholder="City" />
        <input formControlName="state" placeholder="State" />
        <input formControlName="zipCode" placeholder="ZIP" />
        
        <div *ngIf="shippingAddress.hasError('incompleteAddress')" class="error">
          {{ shippingAddress.getError('incompleteAddress').message }}
        </div>
      </div>
    </form>
  `
})
export class UserProfileComponent {
  form = new FormGroup({
    billingAddress: new FormGroup({
      street: new FormControl(''),
      city: new FormControl(''),
      state: new FormControl(''),
      zipCode: new FormControl('')
    }, { validators: addressValidator() }),
    
    shippingAddress: new FormGroup({
      street: new FormControl(''),
      city: new FormControl(''),
      state: new FormControl(''),
      zipCode: new FormControl('')
    }, { validators: addressValidator() })
  });

  get billingAddress() {
    return this.form.controls.billingAddress;
  }

  get shippingAddress() {
    return this.form.controls.shippingAddress;
  }
}
```

## Displaying Cross-Field Errors

Best practices for showing form-level errors.

### Error Display Component

```typescript
@Component({
  selector: 'app-form-errors',
  template: `
    <div *ngIf="hasErrors()" class="form-errors">
      <h4>Please correct the following errors:</h4>
      <ul>
        <li *ngFor="let error of getErrors()">{{ error }}</li>
      </ul>
    </div>
  `,
  styles: [`
    .form-errors {
      background: #ffebee;
      border: 1px solid #ef5350;
      padding: 1rem;
      margin: 1rem 0;
      border-radius: 4px;
    }
    .form-errors h4 {
      color: #c62828;
      margin-top: 0;
    }
  `]
})
export class FormErrorsComponent {
  @Input() form!: FormGroup;

  hasErrors(): boolean {
    return this.form.invalid && (this.form.dirty || this.form.touched);
  }

  getErrors(): string[] {
    const errors: string[] = [];

    if (!this.form.errors) {
      return errors;
    }

    // Form-level errors
    if (this.form.hasError('passwordMismatch')) {
      errors.push('Passwords do not match');
    }
    if (this.form.hasError('dateRange')) {
      errors.push(this.form.getError('dateRange').message);
    }
    if (this.form.hasError('budgetExceeded')) {
      errors.push(this.form.getError('budgetExceeded').message);
    }
    if (this.form.hasError('atLeastOneRequired')) {
      errors.push(this.form.getError('atLeastOneRequired').message);
    }

    return errors;
  }
}

// Usage
@Component({
  selector: 'app-complex-form',
  template: `
    <app-form-errors [form]="form"></app-form-errors>
    
    <form [formGroup]="form">
      <!-- form fields -->
    </form>
  `
})
export class ComplexFormComponent {
  form: FormGroup = /* ... */;
}
```

## Dynamic Cross-Field Validation

Adding/removing validators based on form state.

### Dynamic Validator Management

```typescript
@Component({
  selector: 'app-dynamic-validation',
  template: `
    <form [formGroup]="form">
      <div>
        <label>
          <input formControlName="requireEndDate" type="checkbox" />
          Event has end date
        </label>
      </div>

      <div>
        <label>Start Date</label>
        <input formControlName="startDate" type="date" />
      </div>

      <div *ngIf="form.value.requireEndDate">
        <label>End Date</label>
        <input formControlName="endDate" type="date" />
      </div>

      <div *ngIf="form.hasError('dateRange')" class="error">
        End date must be after start date
      </div>
    </form>
  `
})
export class DynamicValidationComponent {
  form = new FormGroup({
    requireEndDate: new FormControl(false, { nonNullable: true }),
    startDate: new FormControl('', Validators.required),
    endDate: new FormControl('')
  });

  ngOnInit() {
    this.form.controls.requireEndDate.valueChanges.subscribe(required => {
      if (required) {
        // Add end date validation
        this.form.controls.endDate.setValidators(Validators.required);
        this.form.setValidators(dateRangeValidator('startDate', 'endDate'));
      } else {
        // Remove end date validation
        this.form.controls.endDate.clearValidators();
        this.form.clearValidators();
        this.form.controls.endDate.setValue('');
      }
      
      this.form.controls.endDate.updateValueAndValidity();
      this.form.updateValueAndValidity();
    });
  }
}
```

## Testing Cross-Field Validators

Unit testing form-level validators.

### Test Suite

```typescript
import { FormGroup, FormControl } from '@angular/forms';
import { passwordMatchValidator, dateRangeValidator } from './validators';

describe('Cross-Field Validators', () => {
  describe('passwordMatchValidator', () => {
    it('should return null when passwords match', () => {
      const form = new FormGroup({
        password: new FormControl('password123'),
        confirmPassword: new FormControl('password123')
      }, { validators: passwordMatchValidator() });

      expect(form.errors).toBeNull();
      expect(form.valid).toBe(true);
    });

    it('should return error when passwords do not match', () => {
      const form = new FormGroup({
        password: new FormControl('password123'),
        confirmPassword: new FormControl('different')
      }, { validators: passwordMatchValidator() });

      expect(form.hasError('passwordMismatch')).toBe(true);
      expect(form.valid).toBe(false);
    });

    it('should set error on confirmPassword control', () => {
      const form = new FormGroup({
        password: new FormControl('password123'),
        confirmPassword: new FormControl('different')
      }, { validators: passwordMatchValidator() });

      expect(form.controls['confirmPassword'].hasError('passwordMismatch')).toBe(true);
    });
  });

  describe('dateRangeValidator', () => {
    it('should return null when end date is after start date', () => {
      const form = new FormGroup({
        startDate: new FormControl('2024-01-01'),
        endDate: new FormControl('2024-01-31')
      }, { validators: dateRangeValidator('startDate', 'endDate') });

      expect(form.errors).toBeNull();
    });

    it('should return error when end date is before start date', () => {
      const form = new FormGroup({
        startDate: new FormControl('2024-01-31'),
        endDate: new FormControl('2024-01-01')
      }, { validators: dateRangeValidator('startDate', 'endDate') });

      expect(form.hasError('dateRange')).toBe(true);
    });

    it('should allow same day when configured', () => {
      const form = new FormGroup({
        startDate: new FormControl('2024-01-01'),
        endDate: new FormControl('2024-01-01')
      }, { validators: dateRangeValidator('startDate', 'endDate', true) });

      expect(form.errors).toBeNull();
    });
  });
});
```

## Common Mistakes

### 1. Not Checking if Controls Exist

```typescript
// WRONG
function badValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const field1 = control.get('field1')!; // Might be null!
    return field1.value === 'test' ? null : { invalid: true };
  };
}

// CORRECT
function goodValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const field1 = control.get('field1');
    if (!field1) return null; // Check existence
    return field1.value === 'test' ? null : { invalid: true };
  };
}
```

### 2. Not Preserving Other Errors

```typescript
// WRONG: Overwrites all errors
confirmPassword.setErrors({ passwordMismatch: true });

// CORRECT: Preserves other errors
confirmPassword.setErrors({
  ...confirmPassword.errors,
  passwordMismatch: true
});
```

### 3. Not Triggering Re-validation

```typescript
// WRONG: Password changes but confirmPassword not re-validated
this.form.controls.password.valueChanges.subscribe(() => {
  // confirmPassword still shows old validation state
});

// CORRECT: Trigger re-validation
this.form.controls.password.valueChanges.subscribe(() => {
  this.form.controls.confirmPassword.updateValueAndValidity();
});
```

## Best Practices

1. **Check control existence** - Always verify controls exist before accessing
2. **Preserve other errors** - Don't overwrite unrelated validation errors
3. **Trigger re-validation** - Update dependent fields when trigger changes
4. **Clear errors when condition not met** - Remove conditional errors appropriately
5. **Provide clear error messages** - Include context in error objects
6. **Test thoroughly** - Unit test all validation scenarios
7. **Use descriptive error keys** - Make errors easy to identify
8. **Document validator behavior** - Clear comments on complex logic
9. **Handle null/undefined values** - Don't assume values exist
10. **Display errors contextually** - Show errors near relevant fields

## Interview Questions

### Q1: What's the difference between control-level and form-level validators?

**Answer:** Control-level validators validate individual fields, while form-level validators validate relationships between multiple fields. Form-level validators are applied to FormGroup and can access all child controls.

### Q2: How do you trigger re-validation when a dependent field changes?

**Answer:** Subscribe to valueChanges and call updateValueAndValidity():
```typescript
control1.valueChanges.subscribe(() => {
  control2.updateValueAndValidity();
});
```

### Q3: Should you set errors on child controls in a form validator?

**Answer:** Yes, it's common to set errors on specific controls (like confirmPassword) while also returning form-level errors. This allows showing errors next to the relevant field.

### Q4: How do you clear conditional errors when the condition is no longer met?

**Answer:** Remove the specific error key while preserving other errors:
```typescript
const { conditionalError, ...otherErrors } = control.errors!;
control.setErrors(Object.keys(otherErrors).length > 0 ? otherErrors : null);
```

### Q5: What's the at-least-one pattern?

**Answer:** A validation pattern requiring at least one field from a group to have a value, commonly used for alternative contact methods (email OR phone).

## Key Takeaways

1. **Form-level validators access multiple controls** - Use for cross-field validation
2. **Always check control existence** - Prevent null reference errors
3. **Preserve unrelated errors** - Don't overwrite other validation states
4. **Trigger dependent re-validation** - Update related fields when values change
5. **Set errors on specific controls** - Better UX for field-specific messages
6. **Clear conditional errors** - Remove when conditions no longer apply
7. **Test all scenarios** - Include edge cases and condition changes
8. **Provide detailed error objects** - Include context for error messages
9. **Use form.errors for form-level messages** - Display summary errors
10. **Document complex validators** - Make maintenance easier

## Resources

- [Angular Form Validation](https://angular.io/guide/form-validation)
- [AbstractControl API](https://angular.io/api/forms/AbstractControl)
- [FormGroup Validators](https://angular.io/api/forms/FormGroup#validators)
- [Cross-field Validation Examples](https://angular.io/guide/form-validation#cross-field-validation)
- [Testing Forms](https://angular.io/guide/testing-components-scenarios#forms)
