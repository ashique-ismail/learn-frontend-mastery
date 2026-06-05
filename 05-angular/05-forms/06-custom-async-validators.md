# Custom Async Validators in Angular

## The Idea

**In plain English:** A custom async validator is a rule you write yourself that checks whether a form field's value is acceptable by asking an outside source (like a server) and waiting for an answer before deciding if the input is valid or not. "Async" just means it doesn't get the answer instantly — it has to wait, like sending a question and getting a reply.

**Real-world analogy:** Imagine you're signing up for a library card and you want to pick a unique library ID. You write your chosen ID on a slip of paper and hand it to the librarian, who disappears into a back room to check the records. You stand at the counter and wait. A minute later they return and tell you whether that ID is already taken or is free to use.

- The slip of paper = the form field value you typed
- The librarian checking the back room = the HTTP request sent to the server
- The waiting time at the counter = the "pending" state while Angular waits for the response
- The librarian's answer ("taken" or "available") = the validation result (error object or null)

---

## Overview

Async validators enable validation that requires asynchronous operations like HTTP requests, database queries, or time-delayed checks. Common use cases include checking username availability, validating email uniqueness, verifying discount codes, and validating against external APIs.

Unlike synchronous validators, async validators return Promises or Observables that resolve to validation errors or null. Angular handles the asynchronous nature automatically, providing loading states and debouncing capabilities.

This guide covers how to create async validators, optimize performance, handle errors, and integrate with real-world backend APIs.

## Table of Contents

1. [AsyncValidatorFn Interface](#asyncvalidatorfn-interface)
2. [Basic Async Validators](#basic-async-validators)
3. [HTTP-Based Validators](#http-based-validators)
4. [Debouncing and Performance](#debouncing-and-performance)
5. [Observable vs Promise](#observable-vs-promise)
6. [Error Handling](#error-handling)
7. [Combining Sync and Async Validators](#combining-sync-and-async-validators)
8. [Status Changes and Loading States](#status-changes-and-loading-states)
9. [Caching Validation Results](#caching-validation-results)
10. [Testing Async Validators](#testing-async-validators)

## AsyncValidatorFn Interface

Understanding the async validator type signature.

### Type Signature

```typescript
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable } from 'rxjs';

// AsyncValidatorFn type definition
type AsyncValidatorFn = (
  control: AbstractControl
) => Promise<ValidationErrors | null> | Observable<ValidationErrors | null>;

// Returns Promise
function asyncValidatorPromise(control: AbstractControl): Promise<ValidationErrors | null> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(control.value === 'taken' ? { taken: true } : null);
    }, 1000);
  });
}

// Returns Observable
function asyncValidatorObservable(control: AbstractControl): Observable<ValidationErrors | null> {
  return of(control.value === 'taken' ? { taken: true } : null).pipe(
    delay(1000)
  );
}
```

### Control Status During Validation

```typescript
import { Component } from '@angular/core';
import { FormControl } from '@angular/forms';
import { of } from 'rxjs';
import { delay, map } from 'rxjs/operators';

@Component({
  selector: 'app-status-demo',
  template: `
    <input [formControl]="control" />
    <div>Status: {{ control.status }}</div>
    <div>Valid: {{ control.valid }}</div>
    <div>Invalid: {{ control.invalid }}</div>
    <div>Pending: {{ control.pending }}</div>
  `
})
export class StatusDemoComponent {
  control = new FormControl('', {
    asyncValidators: [(control) => {
      // Status changes: PENDING -> VALID/INVALID
      return of(control.value === 'taken').pipe(
        delay(1000),
        map(isTaken => isTaken ? { taken: true } : null)
      );
    }]
  });

  ngOnInit() {
    this.control.statusChanges.subscribe(status => {
      console.log('Status:', status); // PENDING, then VALID or INVALID
    });
  }
}
```

## Basic Async Validators

Simple async validators with delays.

### Simulated Async Validation

```typescript
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { delay, map } from 'rxjs/operators';

export function simulatedAsyncValidator(
  forbiddenValue: string,
  delayMs: number = 1000
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) {
      return of(null);
    }

    return of(control.value).pipe(
      delay(delayMs),
      map(value => {
        return value === forbiddenValue ? { forbidden: { value } } : null;
      })
    );
  };
}

// Usage
@Component({
  selector: 'app-async-demo',
  template: `
    <input [formControl]="control" placeholder="Enter username" />
    
    <div *ngIf="control.pending">Checking availability...</div>
    <div *ngIf="control.hasError('forbidden') && !control.pending">
      This username is taken
    </div>
    <div *ngIf="control.valid && !control.pending">
      Username available!
    </div>
  `
})
export class AsyncDemoComponent {
  control = new FormControl('', {
    asyncValidators: simulatedAsyncValidator('admin', 1000)
  });
}
```

### Promise-Based Validator

```typescript
export function promiseValidator(forbiddenValue: string): AsyncValidatorFn {
  return (control: AbstractControl): Promise<ValidationErrors | null> => {
    if (!control.value) {
      return Promise.resolve(null);
    }

    return new Promise((resolve) => {
      setTimeout(() => {
        if (control.value === forbiddenValue) {
          resolve({ forbidden: { value: control.value } });
        } else {
          resolve(null);
        }
      }, 1000);
    });
  };
}

// Usage
const control = new FormControl('', {
  asyncValidators: promiseValidator('reserved')
});
```

## HTTP-Based Validators

Real-world async validators using HTTP services.

### Username Availability Validator

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { map, catchError, debounceTime, distinctUntilChanged, switchMap, first } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UsernameValidatorService {
  constructor(private http: HttpClient) {}

  checkUsername(username: string): Observable<boolean> {
    return this.http.get<{ available: boolean }>(
      `/api/users/check-username/${username}`
    ).pipe(
      map(response => response.available),
      catchError(() => of(true)) // Assume available on error
    );
  }

  usernameAvailableValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }

      return of(control.value).pipe(
        debounceTime(500), // Wait 500ms after user stops typing
        distinctUntilChanged(), // Only check if value changed
        switchMap(username => this.checkUsername(username)),
        map(available => available ? null : { usernameTaken: true }),
        first() // Complete after first emission
      );
    };
  }
}

// Usage
@Component({
  selector: 'app-registration',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Username</label>
        <input formControlName="username" />
        
        <div *ngIf="usernameControl.pending" class="info">
          <span class="spinner"></span> Checking availability...
        </div>
        
        <div *ngIf="usernameControl.hasError('usernameTaken') && !usernameControl.pending"
             class="error">
          Username is already taken
        </div>
        
        <div *ngIf="usernameControl.valid && !usernameControl.pending && usernameControl.dirty"
             class="success">
          Username available!
        </div>
      </div>

      <button [disabled]="form.invalid || form.pending">Register</button>
    </form>
  `,
  styles: [`
    .error { color: red; }
    .success { color: green; }
    .info { color: blue; }
    .spinner { /* spinner styles */ }
  `]
})
export class RegistrationComponent {
  form: FormGroup;

  constructor(
    private fb: FormBuilder,
    private usernameValidator: UsernameValidatorService
  ) {
    this.form = this.fb.group({
      username: ['', {
        validators: [Validators.required, Validators.minLength(3)],
        asyncValidators: [this.usernameValidator.usernameAvailableValidator()],
        updateOn: 'blur' // Only validate on blur, not on every keystroke
      }]
    });
  }

  get usernameControl() {
    return this.form.controls['username'];
  }
}
```

### Email Uniqueness Validator

```typescript
@Injectable({ providedIn: 'root' })
export class EmailValidatorService {
  constructor(private http: HttpClient) {}

  emailExistsValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }

      // Don't validate if email format is invalid
      if (control.hasError('email')) {
        return of(null);
      }

      return of(control.value).pipe(
        debounceTime(800),
        distinctUntilChanged(),
        switchMap(email => 
          this.http.post<{ exists: boolean }>('/api/auth/check-email', { email })
        ),
        map(response => response.exists ? { emailExists: true } : null),
        catchError(() => of(null)) // Continue on error
      );
    };
  }
}

// Usage
@Component({
  selector: 'app-signup',
  template: `
    <input [formControl]="emailControl" type="email" />
    
    <div *ngIf="emailControl.pending">
      Verifying email...
    </div>
    
    <div *ngIf="emailControl.hasError('email')">
      Invalid email format
    </div>
    
    <div *ngIf="emailControl.hasError('emailExists') && !emailControl.pending">
      This email is already registered
      <a routerLink="/login">Login instead?</a>
    </div>
  `
})
export class SignupComponent {
  emailControl: FormControl;

  constructor(private emailValidator: EmailValidatorService) {
    this.emailControl = new FormControl('', {
      validators: [Validators.required, Validators.email],
      asyncValidators: [this.emailValidator.emailExistsValidator()]
    });
  }
}
```

### Discount Code Validator

```typescript
@Injectable({ providedIn: 'root' })
export class DiscountCodeValidatorService {
  constructor(private http: HttpClient) {}

  validateDiscountCode(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }

      const code = control.value.toUpperCase();

      return this.http.post<{
        valid: boolean;
        message?: string;
        discount?: number;
      }>('/api/discount/validate', { code }).pipe(
        debounceTime(600),
        map(response => {
          if (response.valid) {
            // Store discount info in control value
            return null;
          }
          return {
            invalidCode: {
              message: response.message || 'Invalid discount code',
              code
            }
          };
        }),
        catchError(error => {
          if (error.status === 404) {
            return of({ invalidCode: { message: 'Code not found', code } });
          }
          return of({ invalidCode: { message: 'Could not validate code', code } });
        })
      );
    };
  }
}

// Usage
@Component({
  selector: 'app-checkout',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Discount Code (Optional)</label>
        <input formControlName="discountCode" 
               placeholder="Enter code"
               (input)="onDiscountCodeChange()" />
        
        <div *ngIf="form.controls.discountCode.pending">
          Validating code...
        </div>
        
        <div *ngIf="form.controls.discountCode.hasError('invalidCode') && !form.controls.discountCode.pending">
          {{ form.controls.discountCode.getError('invalidCode').message }}
        </div>
        
        <div *ngIf="form.controls.discountCode.valid && form.controls.discountCode.value">
          <span class="success">✓ Code applied!</span>
        </div>
      </div>

      <div>
        <strong>Total: ${{ calculateTotal() }}</strong>
      </div>

      <button [disabled]="form.invalid || form.pending">
        Complete Purchase
      </button>
    </form>
  `
})
export class CheckoutComponent {
  form: FormGroup;
  baseTotal = 100;

  constructor(
    private fb: FormBuilder,
    private discountValidator: DiscountCodeValidatorService
  ) {
    this.form = this.fb.group({
      discountCode: ['', {
        asyncValidators: [this.discountValidator.validateDiscountCode()],
        updateOn: 'change'
      }]
    });
  }

  onDiscountCodeChange() {
    // Transform to uppercase
    const control = this.form.controls['discountCode'];
    if (control.value) {
      control.setValue(control.value.toUpperCase(), { emitEvent: false });
    }
  }

  calculateTotal(): number {
    // Calculate with discount
    return this.baseTotal;
  }
}
```

## Debouncing and Performance

Optimizing async validators for better UX.

### Debounce Strategies

```typescript
import { AbstractControl, AsyncValidatorFn } from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap, map, first } from 'rxjs/operators';

// Strategy 1: Simple debounce
export function debouncedValidator(
  validationFn: (value: any) => Observable<boolean>,
  debounceMs: number = 500
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return of(control.value).pipe(
      debounceTime(debounceMs),
      switchMap(value => validationFn(value)),
      map(isValid => isValid ? null : { invalid: true })
    );
  };
}

// Strategy 2: Debounce with minimum length
export function debouncedMinLengthValidator(
  validationFn: (value: any) => Observable<boolean>,
  minLength: number = 3,
  debounceMs: number = 500
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value || control.value.length < minLength) {
      return of(null);
    }

    return of(control.value).pipe(
      debounceTime(debounceMs),
      distinctUntilChanged(),
      switchMap(value => validationFn(value)),
      map(isValid => isValid ? null : { invalid: true }),
      first()
    );
  };
}

// Strategy 3: Progressive debounce (faster for longer inputs)
export function progressiveDebouncedValidator(
  validationFn: (value: any) => Observable<boolean>
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    const length = control.value?.length || 0;
    // Shorter debounce for longer inputs
    const debounceMs = Math.max(200, 1000 - (length * 50));

    return of(control.value).pipe(
      debounceTime(debounceMs),
      distinctUntilChanged(),
      switchMap(value => validationFn(value)),
      map(isValid => isValid ? null : { invalid: true })
    );
  };
}
```

### UpdateOn Strategy

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-updateon-demo',
  template: `
    <form [formGroup]="form">
      <!-- Validate on every change (default) -->
      <div>
        <input formControlName="realtimeUsername" />
        <span *ngIf="form.controls.realtimeUsername.pending">Checking...</span>
      </div>

      <!-- Validate only on blur -->
      <div>
        <input formControlName="blurUsername" />
        <span *ngIf="form.controls.blurUsername.pending">Checking...</span>
      </div>

      <!-- Validate only on submit -->
      <div>
        <input formControlName="submitUsername" />
        <span *ngIf="form.controls.submitUsername.pending">Checking...</span>
      </div>

      <button type="submit">Submit</button>
    </form>
  `
})
export class UpdateOnDemoComponent {
  form: FormGroup;

  constructor(
    private fb: FormBuilder,
    private validator: UsernameValidatorService
  ) {
    this.form = this.fb.group({
      // Validates on every keystroke (can be expensive)
      realtimeUsername: ['', {
        asyncValidators: [this.validator.usernameAvailableValidator()],
        updateOn: 'change' // default
      }],

      // Validates only when user leaves field (better UX)
      blurUsername: ['', {
        asyncValidators: [this.validator.usernameAvailableValidator()],
        updateOn: 'blur'
      }],

      // Validates only on form submission (least HTTP requests)
      submitUsername: ['', {
        asyncValidators: [this.validator.usernameAvailableValidator()],
        updateOn: 'submit'
      }]
    });
  }
}
```

## Observable vs Promise

Comparing Observable and Promise-based validators.

### Observable-Based (Recommended)

```typescript
// Observable allows cancellation via switchMap
export function observableValidator(http: HttpClient): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return of(control.value).pipe(
      debounceTime(500),
      distinctUntilChanged(),
      switchMap(value => 
        http.get<{ valid: boolean }>(`/api/validate/${value}`)
      ),
      map(response => response.valid ? null : { invalid: true }),
      catchError(() => of(null))
    );
  };
}

// Benefits:
// - Automatic cancellation with switchMap
// - Easy to add operators (debounce, retry, etc.)
// - Better error handling
```

### Promise-Based

```typescript
// Promise cannot be cancelled once started
export function promiseValidator(http: HttpClient): AsyncValidatorFn {
  return (control: AbstractControl): Promise<ValidationErrors | null> => {
    return new Promise((resolve) => {
      setTimeout(() => {
        http.get<{ valid: boolean }>(`/api/validate/${control.value}`)
          .toPromise()
          .then(response => {
            resolve(response?.valid ? null : { invalid: true });
          })
          .catch(() => resolve(null));
      }, 500);
    });
  };
}

// Drawbacks:
// - Cannot cancel previous requests
// - Hard to implement debouncing
// - Less composable
```

### Conversion Between Observable and Promise

```typescript
// Observable to Promise
function toPromise(obs: Observable<any>): Promise<any> {
  return obs.pipe(first()).toPromise();
}

// Promise to Observable
function toObservable(promise: Promise<any>): Observable<any> {
  return from(promise);
}
```

## Error Handling

Gracefully handling errors in async validators.

### Error Handling Strategies

```typescript
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { catchError, map, retry, timeout } from 'rxjs/operators';
import { of, throwError } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class RobustValidatorService {
  constructor(private http: HttpClient) {}

  // Strategy 1: Return null on error (assume valid)
  lenientValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      return this.http.get<{ valid: boolean }>(`/api/validate/${control.value}`).pipe(
        map(response => response.valid ? null : { invalid: true }),
        catchError(error => {
          console.error('Validation error:', error);
          return of(null); // Assume valid on error
        })
      );
    };
  }

  // Strategy 2: Return error state on failure (assume invalid)
  strictValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      return this.http.get<{ valid: boolean }>(`/api/validate/${control.value}`).pipe(
        map(response => response.valid ? null : { invalid: true }),
        catchError(error => {
          console.error('Validation error:', error);
          return of({ validationError: { message: 'Could not validate' } });
        })
      );
    };
  }

  // Strategy 3: Retry with timeout
  retryValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      return this.http.get<{ valid: boolean }>(`/api/validate/${control.value}`).pipe(
        timeout(5000), // 5 second timeout
        retry(2), // Retry twice on failure
        map(response => response.valid ? null : { invalid: true }),
        catchError((error: HttpErrorResponse) => {
          if (error.name === 'TimeoutError') {
            return of({ timeout: { message: 'Validation timed out' } });
          }
          return of({ networkError: { message: 'Network error occurred' } });
        })
      );
    };
  }

  // Strategy 4: Different handling based on error type
  smartValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      return this.http.get<{ valid: boolean }>(`/api/validate/${control.value}`).pipe(
        map(response => response.valid ? null : { invalid: true }),
        catchError((error: HttpErrorResponse) => {
          // Handle different error codes
          if (error.status === 404) {
            return of(null); // Not found = valid (doesn't exist)
          }
          if (error.status === 409) {
            return of({ conflict: true }); // Conflict = invalid (exists)
          }
          if (error.status >= 500) {
            return of(null); // Server error = assume valid
          }
          if (error.status === 0) {
            return of({ offline: { message: 'You appear to be offline' } });
          }
          return of({ unknownError: { message: 'Validation failed' } });
        })
      );
    };
  }
}
```

### User Feedback During Errors

```typescript
@Component({
  selector: 'app-validated-input',
  template: `
    <input [formControl]="control" />
    
    <div *ngIf="control.pending" class="info">
      Validating...
    </div>
    
    <div *ngIf="control.hasError('validationError')" class="warning">
      Could not validate. Please try again.
      <button (click)="control.updateValueAndValidity()">Retry</button>
    </div>
    
    <div *ngIf="control.hasError('timeout')" class="warning">
      Validation timed out. Check your connection.
    </div>
    
    <div *ngIf="control.hasError('offline')" class="warning">
      You appear to be offline. Please check your connection.
    </div>
  `
})
export class ValidatedInputComponent {
  control: FormControl;

  constructor(private validator: RobustValidatorService) {
    this.control = new FormControl('', {
      asyncValidators: [this.validator.smartValidator()]
    });
  }
}
```

## Combining Sync and Async Validators

Using both types of validators together.

### Sync First, Then Async

```typescript
@Component({
  selector: 'app-combined-validation',
  template: `
    <form [formGroup]="form">
      <div>
        <label>Email</label>
        <input formControlName="email" type="email" />
        
        <!-- Show sync errors first -->
        <div *ngIf="emailControl.hasError('required') && emailControl.touched">
          Email is required
        </div>
        <div *ngIf="emailControl.hasError('email') && !emailControl.hasError('required')">
          Invalid email format
        </div>
        
        <!-- Only show async validation when sync validation passes -->
        <div *ngIf="emailControl.pending && !emailControl.hasError('email')">
          Checking if email is available...
        </div>
        <div *ngIf="emailControl.hasError('emailExists') && emailControl.valid">
          This email is already registered
        </div>
      </div>

      <div>
        <label>Username</label>
        <input formControlName="username" />
        
        <div *ngIf="usernameControl.hasError('required') && usernameControl.touched">
          Username is required
        </div>
        <div *ngIf="usernameControl.hasError('minlength')">
          Username must be at least 3 characters
        </div>
        <div *ngIf="usernameControl.hasError('pattern')">
          Username can only contain letters and numbers
        </div>
        
        <div *ngIf="usernameControl.pending">
          Checking availability...
        </div>
        <div *ngIf="usernameControl.hasError('usernameTaken')">
          Username is taken
        </div>
      </div>

      <button [disabled]="form.invalid || form.pending">
        Register
      </button>
    </form>
  `
})
export class CombinedValidationComponent {
  form: FormGroup;

  constructor(
    private fb: FormBuilder,
    private usernameValidator: UsernameValidatorService,
    private emailValidator: EmailValidatorService
  ) {
    this.form = this.fb.group({
      email: ['', {
        validators: [Validators.required, Validators.email],
        asyncValidators: [this.emailValidator.emailExistsValidator()],
        updateOn: 'blur'
      }],
      username: ['', {
        validators: [
          Validators.required,
          Validators.minLength(3),
          Validators.maxLength(20),
          Validators.pattern(/^[a-zA-Z0-9]+$/)
        ],
        asyncValidators: [this.usernameValidator.usernameAvailableValidator()],
        updateOn: 'blur'
      }]
    });
  }

  get emailControl() {
    return this.form.controls['email'];
  }

  get usernameControl() {
    return this.form.controls['username'];
  }
}
```

### Short-Circuit Async Validation

```typescript
// Only run async validator if sync validators pass
export function conditionalAsyncValidator(
  asyncValidator: AsyncValidatorFn
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    // If control has sync errors, don't run async validation
    if (control.errors) {
      return of(null);
    }

    return asyncValidator(control);
  };
}

// Usage
const control = new FormControl('', {
  validators: [Validators.required, Validators.email],
  asyncValidators: [
    conditionalAsyncValidator(emailExistsValidator())
  ]
});
```

## Status Changes and Loading States

Tracking validation state in the UI.

### Reactive Loading State

```typescript
@Component({
  selector: 'app-loading-demo',
  template: `
    <input [formControl]="control" />
    
    <div class="status-indicator">
      <span *ngIf="isValidating$ | async" class="spinner">
        Validating...
      </span>
      <span *ngIf="isValid$ | async" class="success">
        ✓ Valid
      </span>
      <span *ngIf="isInvalid$ | async" class="error">
        ✗ Invalid
      </span>
    </div>
  `
})
export class LoadingDemoComponent {
  control = new FormControl('', {
    asyncValidators: [someAsyncValidator()]
  });

  isValidating$ = this.control.statusChanges.pipe(
    map(status => status === 'PENDING')
  );

  isValid$ = this.control.statusChanges.pipe(
    map(status => status === 'VALID')
  );

  isInvalid$ = this.control.statusChanges.pipe(
    map(status => status === 'INVALID')
  );
}
```

### Progress Indicator

```typescript
@Component({
  selector: 'app-progress-indicator',
  template: `
    <div class="field-wrapper">
      <input [formControl]="control" />
      
      <div class="progress-bar" *ngIf="control.pending">
        <div class="progress-bar-fill" 
             [style.animation-duration]="estimatedDuration + 'ms'">
        </div>
      </div>
      
      <div class="validation-message">
        <ng-container [ngSwitch]="control.status">
          <span *ngSwitchCase="'PENDING'" class="info">
            <mat-spinner diameter="16"></mat-spinner>
            Checking...
          </span>
          <span *ngSwitchCase="'VALID'" class="success">
            ✓ Available
          </span>
          <span *ngSwitchCase="'INVALID'" class="error">
            ✗ {{ getErrorMessage() }}
          </span>
        </ng-container>
      </div>
    </div>
  `,
  styles: [`
    .progress-bar {
      height: 2px;
      background: #e0e0e0;
      overflow: hidden;
    }
    .progress-bar-fill {
      height: 100%;
      background: #2196f3;
      animation: progress linear;
    }
    @keyframes progress {
      from { width: 0%; }
      to { width: 100%; }
    }
  `]
})
export class ProgressIndicatorComponent {
  control: FormControl;
  estimatedDuration = 1000; // ms

  constructor(private validator: UsernameValidatorService) {
    this.control = new FormControl('', {
      asyncValidators: [this.validator.usernameAvailableValidator()]
    });
  }

  getErrorMessage(): string {
    if (this.control.hasError('usernameTaken')) {
      return 'Username is taken';
    }
    return 'Invalid';
  }
}
```

## Caching Validation Results

Optimizing by caching previous validation results.

### Cached Validator

```typescript
@Injectable({ providedIn: 'root' })
export class CachedValidatorService {
  private cache = new Map<string, boolean>();
  private pendingRequests = new Map<string, Observable<boolean>>();

  constructor(private http: HttpClient) {}

  cachedUsernameValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }

      const username = control.value.toLowerCase();

      // Check cache first
      if (this.cache.has(username)) {
        const isAvailable = this.cache.get(username)!;
        return of(isAvailable ? null : { usernameTaken: true });
      }

      // Check if request is already pending
      if (this.pendingRequests.has(username)) {
        return this.pendingRequests.get(username)!.pipe(
          map(isAvailable => isAvailable ? null : { usernameTaken: true })
        );
      }

      // Make new request
      const request$ = this.http.get<{ available: boolean }>(
        `/api/users/check-username/${username}`
      ).pipe(
        map(response => response.available),
        tap(isAvailable => {
          // Cache the result
          this.cache.set(username, isAvailable);
          // Remove from pending
          this.pendingRequests.delete(username);
        }),
        catchError(() => {
          this.pendingRequests.delete(username);
          return of(true); // Assume available on error
        }),
        shareReplay(1) // Share among multiple subscribers
      );

      this.pendingRequests.set(username, request$);

      return request$.pipe(
        map(isAvailable => isAvailable ? null : { usernameTaken: true })
      );
    };
  }

  clearCache() {
    this.cache.clear();
  }

  removefromCache(username: string) {
    this.cache.delete(username.toLowerCase());
  }
}
```

### Time-Based Cache Expiration

```typescript
interface CacheEntry<T> {
  value: T;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class TimedCacheValidatorService {
  private cache = new Map<string, CacheEntry<boolean>>();
  private cacheLifetime = 5 * 60 * 1000; // 5 minutes

  constructor(private http: HttpClient) {}

  cachedValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }

      const key = control.value.toLowerCase();
      const cached = this.cache.get(key);

      // Check if cache is still valid
      if (cached && (Date.now() - cached.timestamp) < this.cacheLifetime) {
        return of(cached.value ? null : { invalid: true });
      }

      // Make request and cache result
      return this.http.get<{ valid: boolean }>(`/api/validate/${key}`).pipe(
        map(response => {
          // Cache with timestamp
          this.cache.set(key, {
            value: response.valid,
            timestamp: Date.now()
          });
          return response.valid ? null : { invalid: true };
        }),
        catchError(() => of(null))
      );
    };
  }
}
```

## Testing Async Validators

Unit testing async validators.

### Testing with TestBed

```typescript
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { FormControl } from '@angular/forms';
import { UsernameValidatorService } from './username-validator.service';

describe('UsernameValidatorService', () => {
  let service: UsernameValidatorService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UsernameValidatorService]
    });

    service = TestBed.inject(UsernameValidatorService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should return null for available username', (done) => {
    const control = new FormControl('newuser');
    const validator = service.usernameAvailableValidator();

    validator(control).subscribe(result => {
      expect(result).toBeNull();
      done();
    });

    const req = httpMock.expectOne('/api/users/check-username/newuser');
    expect(req.request.method).toBe('GET');
    req.flush({ available: true });
  });

  it('should return error for taken username', (done) => {
    const control = new FormControl('admin');
    const validator = service.usernameAvailableValidator();

    validator(control).subscribe(result => {
      expect(result).toEqual({ usernameTaken: true });
      done();
    });

    const req = httpMock.expectOne('/api/users/check-username/admin');
    req.flush({ available: false });
  });

  it('should handle HTTP errors gracefully', (done) => {
    const control = new FormControl('testuser');
    const validator = service.usernameAvailableValidator();

    validator(control).subscribe(result => {
      expect(result).toBeNull(); // Should assume available on error
      done();
    });

    const req = httpMock.expectOne('/api/users/check-username/testuser');
    req.error(new ErrorEvent('Network error'));
  });
});
```

### Testing Debouncing

```typescript
import { fakeAsync, tick } from '@angular/core/testing';

describe('Debounced Validator', () => {
  it('should debounce validation requests', fakeAsync(() => {
    const control = new FormControl('');
    const validator = service.usernameAvailableValidator();

    // Set value multiple times quickly
    control.setValue('a');
    control.setValue('ab');
    control.setValue('abc');

    // No request yet (debouncing)
    httpMock.expectNone('/api/users/check-username/a');
    httpMock.expectNone('/api/users/check-username/ab');

    // Fast forward past debounce time
    tick(500);

    // Only one request for final value
    const req = httpMock.expectOne('/api/users/check-username/abc');
    req.flush({ available: true });
  }));
});
```

## Common Mistakes

### 1. Not Handling Empty Values

```typescript
// WRONG
function badValidator(control: AbstractControl): Observable<ValidationErrors | null> {
  return http.get(`/api/validate/${control.value}`); // Crashes if value is null
}

// CORRECT
function goodValidator(control: AbstractControl): Observable<ValidationErrors | null> {
  if (!control.value) {
    return of(null);
  }
  return http.get(`/api/validate/${control.value}`);
}
```

### 2. Not Using switchMap

```typescript
// WRONG: Multiple overlapping requests
function badValidator(control: AbstractControl): Observable<ValidationErrors | null> {
  return http.get(`/api/validate/${control.value}`).pipe(
    map(response => response.valid ? null : { invalid: true })
  );
}

// CORRECT: Cancels previous requests
function goodValidator(control: AbstractControl): Observable<ValidationErrors | null> {
  return of(control.value).pipe(
    switchMap(value => http.get(`/api/validate/${value}`)),
    map(response => response.valid ? null : { invalid: true })
  );
}
```

### 3. Forgetting Error Handling

```typescript
// WRONG: Validator throws error on HTTP failure
function badValidator(control: AbstractControl): Observable<ValidationErrors | null> {
  return http.get(`/api/validate/${control.value}`).pipe(
    map(response => response.valid ? null : { invalid: true })
  );
}

// CORRECT: Handles errors gracefully
function goodValidator(control: AbstractControl): Observable<ValidationErrors | null> {
  return http.get(`/api/validate/${control.value}`).pipe(
    map(response => response.valid ? null : { invalid: true }),
    catchError(() => of(null))
  );
}
```

## Best Practices

1. **Always use switchMap** - Cancel previous requests
2. **Implement debouncing** - Reduce server load
3. **Handle errors gracefully** - Don't block form submission
4. **Use updateOn: 'blur'** - Better UX than every keystroke
5. **Cache results** - Avoid redundant requests
6. **Show loading states** - Visual feedback during validation
7. **Short-circuit on sync errors** - Don't validate if basic checks fail
8. **Test with mock HTTP** - Unit test async validators
9. **Set reasonable timeouts** - Don't wait forever
10. **Prefer Observable over Promise** - More control and composability

## Interview Questions

### Q1: What's the difference between sync and async validators?

**Answer:** Sync validators return ValidationErrors | null immediately, while async validators return Observable<ValidationErrors | null> or Promise<ValidationErrors | null> for operations that take time (HTTP requests, etc.).

### Q2: Why use switchMap in async validators?

**Answer:** switchMap cancels previous HTTP requests when a new value is emitted, preventing race conditions and reducing unnecessary server load when users type quickly.

### Q3: What is the 'pending' status?

**Answer:** The 'pending' status indicates that async validation is in progress. The control is neither valid nor invalid until async validators complete.

### Q4: How do you optimize async validator performance?

**Answer:** Use debouncing, updateOn: 'blur', caching results, switchMap for cancellation, and short-circuit validation if sync validators fail.

### Q5: Should async validators throw errors?

**Answer:** No, use catchError to handle failures gracefully. Either assume valid/invalid on error, or return a specific error object indicating the validation could not be completed.

## Key Takeaways

1. **Async validators return Observable or Promise** - For time-consuming validation
2. **Use switchMap for cancellation** - Prevents race conditions
3. **Debounce user input** - Reduce server requests
4. **Handle errors with catchError** - Never let validators throw
5. **Show pending status** - Give users feedback
6. **Cache validation results** - Improve performance
7. **UpdateOn 'blur' improves UX** - Validate after user leaves field
8. **Combine sync and async validators** - Run sync checks first
9. **Observable preferred over Promise** - More control and operators
10. **Test with HttpTestingController** - Mock HTTP in tests

## Resources

- [Angular Async Validators](https://angular.io/api/forms/AsyncValidatorFn)
- [Form Validation Guide](https://angular.io/guide/form-validation#async-validation)
- [HttpClient Testing](https://angular.io/guide/http#testing-http-requests)
- [RxJS switchMap](https://www.learnrxjs.io/learn-rxjs/operators/transformation/switchmap)
- [RxJS debounceTime](https://www.learnrxjs.io/learn-rxjs/operators/filtering/debouncetime)
