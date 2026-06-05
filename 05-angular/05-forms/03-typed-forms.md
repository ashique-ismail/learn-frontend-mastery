# Typed Forms in Angular

## The Idea

**In plain English:** Typed forms are a way to build web forms where the computer checks — before the page even loads — that you are using the right kind of data (like a number where a number is expected, or text where text is expected). "Type" just means the category of data: text, number, yes/no, etc.

**Real-world analogy:** Think of a vending machine that only accepts specific coins in each slot. The "25-cent" slot physically cannot fit a dime, so you cannot make a mistake at all — the machine's shape enforces the rule before any money is lost. In the same way, typed forms are shaped so the wrong kind of data cannot fit in the wrong field, and the error is caught immediately.

- The vending machine = the Angular form
- Each coin slot = a form control (an individual input field)
- The shape of the slot = the TypeScript type assigned to that control
- Trying to insert a dime in the quarter slot = passing a string where a number is expected

---

## Overview

Typed forms, introduced in Angular 14, bring full type safety to reactive forms. Before Angular 14, forms were loosely typed with `any`, making it easy to introduce runtime errors. Typed forms use generics to provide compile-time type checking, autocompletion, and refactoring support for form values and controls.

This guide covers how to use typed forms, migrate from untyped forms, and leverage TypeScript's type system for safer form handling.

## Table of Contents

1. [Type Safety Benefits](#type-safety-benefits)
2. [FormControl with Types](#formcontrol-with-types)
3. [FormGroup and Type Inference](#formgroup-and-type-inference)
4. [FormArray with Types](#formarray-with-types)
5. [Nullable vs Non-Nullable Controls](#nullable-vs-non-nullable-controls)
6. [NonNullableFormBuilder](#nonnullableformbuilder)
7. [Partial Values and getRawValue](#partial-values-and-getrawvalue)
8. [Migration from Untyped Forms](#migration-from-untyped-forms)
9. [Type Narrowing and Validation](#type-narrowing-and-validation)
10. [Advanced Type Patterns](#advanced-type-patterns)

## Type Safety Benefits

Typed forms provide compile-time safety that catches errors before runtime.

### Before Angular 14 (Untyped)

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup } from '@angular/forms';

@Component({
  selector: 'app-user-form-old',
  template: `
    <form [formGroup]="userForm">
      <input formControlName="name" />
      <input formControlName="age" type="number" />
    </form>
  `
})
export class UserFormOldComponent {
  userForm = new FormGroup({
    name: new FormControl(''),
    age: new FormControl(0)
  });

  ngOnInit() {
    // No type checking - any typos compile but fail at runtime
    const name = this.userForm.value.namme; // Typo! No error
    
    // Value is typed as any
    const age = this.userForm.value.age; // type: any
    
    // Can set invalid values
    this.userForm.patchValue({ age: 'not a number' }); // Compiles!
  }
}
```

### Angular 14+ (Typed)

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup } from '@angular/forms';

interface UserForm {
  name: string;
  age: number;
}

@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm">
      <input formControlName="name" />
      <input formControlName="age" type="number" />
    </form>
  `
})
export class UserFormComponent {
  userForm = new FormGroup({
    name: new FormControl<string>('', { nonNullable: true }),
    age: new FormControl<number>(0, { nonNullable: true })
  });

  ngOnInit() {
    // Type error: Property 'namme' does not exist
    // const name = this.userForm.value.namme;
    
    // Correctly typed as string | null
    const name = this.userForm.value.name; // type: string
    
    // Type error: Type 'string' is not assignable to type 'number'
    // this.userForm.patchValue({ age: 'not a number' });
    
    // Autocompletion works
    this.userForm.patchValue({
      name: 'John',
      age: 30
    });
  }
}
```

## FormControl with Types

FormControl now accepts a generic type parameter.

### Basic Typed Controls

```typescript
import { FormControl } from '@angular/forms';

// String control
const nameControl = new FormControl<string>('');
nameControl.value; // type: string | null
nameControl.setValue('John'); // OK
// nameControl.setValue(123); // Error: Type 'number' is not assignable

// Number control
const ageControl = new FormControl<number>(0);
ageControl.value; // type: number | null

// Boolean control
const subscribedControl = new FormControl<boolean>(false);
subscribedControl.value; // type: boolean | null

// Date control
const birthdateControl = new FormControl<Date | null>(null);
birthdateControl.value; // type: Date | null

// Union type control
type Status = 'active' | 'inactive' | 'pending';
const statusControl = new FormControl<Status>('active');
statusControl.value; // type: Status | null
// statusControl.setValue('invalid'); // Error
```

### Controls with Initial Values

```typescript
// Type is inferred from initial value
const emailControl = new FormControl('user@example.com');
emailControl.value; // type: string | null

// Explicit type with initial value
const scoreControl = new FormControl<number>(100);
scoreControl.value; // type: number | null

// Array type
const tagsControl = new FormControl<string[]>([]);
tagsControl.value; // type: string[] | null

// Object type
interface Address {
  street: string;
  city: string;
}
const addressControl = new FormControl<Address | null>(null);
addressControl.value; // type: Address | null
```

### Non-Nullable Controls

```typescript
// Nullable by default
const nullableControl = new FormControl<string>('');
nullableControl.value; // type: string | null
nullableControl.setValue(null); // OK

// Non-nullable control
const nonNullableControl = new FormControl<string>('', { 
  nonNullable: true 
});
nonNullableControl.value; // type: string (no null!)
// nonNullableControl.setValue(null); // Error

// When reset, returns initial value instead of null
nonNullableControl.reset();
console.log(nonNullableControl.value); // '' (not null)
```

## FormGroup and Type Inference

FormGroup can infer types from its controls.

### Type-Safe FormGroup

```typescript
import { FormControl, FormGroup } from '@angular/forms';

// Define interface for form structure
interface LoginForm {
  email: string;
  password: string;
  rememberMe: boolean;
}

// Create typed form group
const loginForm = new FormGroup({
  email: new FormControl<string>('', { nonNullable: true }),
  password: new FormControl<string>('', { nonNullable: true }),
  rememberMe: new FormControl<boolean>(false, { nonNullable: true })
});

// Type-safe access
loginForm.value; // type: { email: string; password: string; rememberMe: boolean; }
loginForm.controls.email; // type: FormControl<string>
loginForm.controls.password; // type: FormControl<string>

// Type-safe patching
loginForm.patchValue({
  email: 'user@example.com',
  rememberMe: true
  // password is optional in patchValue
});

// Type error on invalid properties
// loginForm.patchValue({ invalidProp: 'value' }); // Error
```

### Nested FormGroups

```typescript
interface AddressForm {
  street: string;
  city: string;
  zipCode: string;
}

interface UserProfileForm {
  name: string;
  email: string;
  address: AddressForm;
}

const profileForm = new FormGroup({
  name: new FormControl<string>('', { nonNullable: true }),
  email: new FormControl<string>('', { nonNullable: true }),
  address: new FormGroup({
    street: new FormControl<string>('', { nonNullable: true }),
    city: new FormControl<string>('', { nonNullable: true }),
    zipCode: new FormControl<string>('', { nonNullable: true })
  })
});

// Type-safe nested access
profileForm.value.address?.city; // type: string
profileForm.controls.address.controls.city; // type: FormControl<string>

// Type-safe nested patching
profileForm.patchValue({
  name: 'John Doe',
  address: {
    city: 'New York'
  }
});
```

### Optional Form Fields

```typescript
interface OptionalFieldsForm {
  requiredField: string;
  optionalField?: string;
}

const form = new FormGroup({
  requiredField: new FormControl<string>('', { nonNullable: true }),
  optionalField: new FormControl<string | undefined>(undefined)
});

// patchValue recognizes optional fields
form.patchValue({
  requiredField: 'value'
  // optionalField can be omitted
});

form.patchValue({
  requiredField: 'value',
  optionalField: 'optional value'
});
```

## FormArray with Types

FormArray is typed by the type of controls it contains.

### Basic Typed FormArray

```typescript
import { FormArray, FormControl, FormGroup } from '@angular/forms';

// Array of strings
const tagsArray = new FormArray<FormControl<string>>([
  new FormControl<string>('angular', { nonNullable: true }),
  new FormControl<string>('typescript', { nonNullable: true })
]);

tagsArray.value; // type: string[]
tagsArray.at(0).value; // type: string

// Add new control
tagsArray.push(new FormControl<string>('rxjs', { nonNullable: true }));

// Type-safe iteration
tagsArray.controls.forEach(control => {
  control.value; // type: string
});
```

### FormArray of FormGroups

```typescript
interface PhoneNumber {
  type: 'mobile' | 'home' | 'work';
  number: string;
}

const phonesArray = new FormArray<FormGroup<{
  type: FormControl<'mobile' | 'home' | 'work'>;
  number: FormControl<string>;
}>>([]);

// Add phone number
function addPhone() {
  phonesArray.push(new FormGroup({
    type: new FormControl<'mobile' | 'home' | 'work'>('mobile', { 
      nonNullable: true 
    }),
    number: new FormControl<string>('', { nonNullable: true })
  }));
}

addPhone();

// Type-safe access
phonesArray.at(0).value; // type: { type: 'mobile' | 'home' | 'work'; number: string; }
phonesArray.at(0).controls.type.value; // type: 'mobile' | 'home' | 'work'
```

### Complete Example with FormArray

```typescript
import { Component } from '@angular/core';
import { FormArray, FormControl, FormGroup, FormBuilder } from '@angular/forms';

interface Education {
  institution: string;
  degree: string;
  year: number;
}

@Component({
  selector: 'app-education-form',
  template: `
    <form [formGroup]="form">
      <div formArrayName="education">
        <div *ngFor="let edu of educationArray.controls; let i = index" 
             [formGroupName]="i">
          <input formControlName="institution" placeholder="Institution" />
          <input formControlName="degree" placeholder="Degree" />
          <input formControlName="year" type="number" placeholder="Year" />
          <button type="button" (click)="removeEducation(i)">Remove</button>
        </div>
      </div>
      <button type="button" (click)="addEducation()">Add Education</button>
    </form>
  `
})
export class EducationFormComponent {
  form = new FormGroup({
    education: new FormArray<FormGroup<{
      institution: FormControl<string>;
      degree: FormControl<string>;
      year: FormControl<number>;
    }>>([])
  });

  get educationArray() {
    return this.form.controls.education;
  }

  addEducation() {
    this.educationArray.push(new FormGroup({
      institution: new FormControl<string>('', { nonNullable: true }),
      degree: new FormControl<string>('', { nonNullable: true }),
      year: new FormControl<number>(new Date().getFullYear(), { 
        nonNullable: true 
      })
    }));
  }

  removeEducation(index: number) {
    this.educationArray.removeAt(index);
  }

  getFormValue(): Education[] {
    return this.educationArray.value as Education[];
  }
}
```

## Nullable vs Non-Nullable Controls

Understanding nullability is crucial for typed forms.

### Default Nullable Behavior

```typescript
// By default, controls are nullable
const control = new FormControl<string>('hello');
control.value; // type: string | null

control.reset(); // Sets value to null
console.log(control.value); // null

control.setValue(null); // OK
```

### Non-Nullable Controls

```typescript
// Non-nullable controls never return null
const control = new FormControl<string>('hello', { nonNullable: true });
control.value; // type: string (no null!)

control.reset(); // Sets value to initial value ('hello')
console.log(control.value); // 'hello'

// control.setValue(null); // Type error
```

### Practical Example

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup } from '@angular/forms';

@Component({
  selector: 'app-search-form',
  template: `
    <form [formGroup]="searchForm">
      <!-- Query should never be null -->
      <input formControlName="query" placeholder="Search..." />
      
      <!-- Category can be null (optional filter) -->
      <select formControlName="category">
        <option [value]="null">All Categories</option>
        <option value="books">Books</option>
        <option value="electronics">Electronics</option>
      </select>
      
      <button (click)="search()">Search</button>
    </form>
  `
})
export class SearchFormComponent {
  searchForm = new FormGroup({
    // Non-nullable: always has a value
    query: new FormControl<string>('', { nonNullable: true }),
    
    // Nullable: can be null to represent "all categories"
    category: new FormControl<string | null>(null)
  });

  search() {
    const query = this.searchForm.value.query; // type: string
    const category = this.searchForm.value.category; // type: string | null
    
    // No null check needed for query
    console.log(`Searching for: ${query}`);
    
    // Must check category for null
    if (category) {
      console.log(`In category: ${category}`);
    }
  }
}
```

## NonNullableFormBuilder

FormBuilder provides a convenient way to create non-nullable forms.

### Using NonNullableFormBuilder

```typescript
import { Component, inject } from '@angular/core';
import { FormBuilder, NonNullableFormBuilder } from '@angular/forms';

@Component({
  selector: 'app-user-form',
  template: `<form [formGroup]="userForm">...</form>`
})
export class UserFormComponent {
  private fb = inject(NonNullableFormBuilder);

  // All controls are non-nullable by default
  userForm = this.fb.group({
    firstName: [''], // type: FormControl<string>
    lastName: [''],
    age: [0], // type: FormControl<number>
    email: ['user@example.com']
  });

  ngOnInit() {
    // All values are non-nullable
    this.userForm.value.firstName; // type: string
    this.userForm.value.age; // type: number
    
    // Reset returns to initial values, not null
    this.userForm.reset();
    console.log(this.userForm.value.firstName); // ''
    console.log(this.userForm.value.age); // 0
  }
}
```

### Mixing Nullable and Non-Nullable

```typescript
import { Component, inject } from '@angular/core';
import { FormControl, NonNullableFormBuilder } from '@angular/forms';

@Component({
  selector: 'app-profile-form',
  template: `<form [formGroup]="profileForm">...</form>`
})
export class ProfileFormComponent {
  private fb = inject(NonNullableFormBuilder);

  profileForm = this.fb.group({
    // Non-nullable fields (from NonNullableFormBuilder)
    name: ['John Doe'],
    email: ['john@example.com'],
    
    // Explicitly nullable field
    middleName: new FormControl<string | null>(null),
    
    // Optional field (undefined)
    nickname: new FormControl<string | undefined>(undefined)
  });

  submitForm() {
    const value = this.profileForm.value;
    
    value.name; // type: string
    value.email; // type: string
    value.middleName; // type: string | null
    value.nickname; // type: string | undefined
  }
}
```

### Regular FormBuilder with Types

```typescript
import { Component, inject } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';

@Component({
  selector: 'app-registration-form',
  template: `<form [formGroup]="form">...</form>`
})
export class RegistrationFormComponent {
  private fb = inject(FormBuilder);

  // Regular FormBuilder - controls are nullable
  form = this.fb.group({
    username: ['', Validators.required], // FormControl<string | null>
    password: ['', Validators.required],
    age: [null as number | null] // Explicit nullable number
  });

  // Can use nonNullable option
  formNonNull = this.fb.group({
    username: ['', { 
      validators: Validators.required, 
      nonNullable: true 
    }]
  });
}
```

## Partial Values and getRawValue

Understanding value types and disabled controls.

### value vs getRawValue

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup } from '@angular/forms';

@Component({
  selector: 'app-form-values',
  template: `<form [formGroup]="form">...</form>`
})
export class FormValuesComponent {
  form = new FormGroup({
    username: new FormControl<string>('john', { nonNullable: true }),
    email: new FormControl<string>('john@example.com', { nonNullable: true }),
    role: new FormControl<string>('admin', { nonNullable: true })
  });

  ngOnInit() {
    // Disable role field
    this.form.controls.role.disable();

    // value excludes disabled controls
    const value = this.form.value;
    console.log(value);
    // { username: 'john', email: 'john@example.com' }
    // Note: role is missing!

    // getRawValue includes disabled controls
    const rawValue = this.form.getRawValue();
    console.log(rawValue);
    // { username: 'john', email: 'john@example.com', role: 'admin' }

    // Type differences
    value.role; // type: string | undefined (might be disabled)
    rawValue.role; // type: string (always present)
  }
}
```

### Partial Types with Disabled Controls

```typescript
interface UserForm {
  id: number;
  name: string;
  email: string;
  role: string;
}

const form = new FormGroup({
  id: new FormControl<number>(1, { nonNullable: true }),
  name: new FormControl<string>('', { nonNullable: true }),
  email: new FormControl<string>('', { nonNullable: true }),
  role: new FormControl<string>('user', { nonNullable: true })
});

// Disable ID (should never be edited)
form.controls.id.disable();

// form.value type is Partial-like
// type: { id?: number; name: string; email: string; role: string; }
const partialValue = form.value;

// form.getRawValue() includes all fields
// type: { id: number; name: string; email: string; role: string; }
const completeValue = form.getRawValue();

function saveUser() {
  // Use getRawValue for complete data including disabled fields
  const userData = form.getRawValue();
  // userData.id is guaranteed to exist
  return saveToApi(userData);
}
```

## Migration from Untyped Forms

Strategies for migrating existing forms to typed forms.

### Untyped Forms (Backward Compatibility)

```typescript
import { UntypedFormControl, UntypedFormGroup } from '@angular/forms';

// Untyped forms for gradual migration
const legacyForm = new UntypedFormGroup({
  field1: new UntypedFormControl(''),
  field2: new UntypedFormControl(0)
});

// Value is typed as any
legacyForm.value; // type: any
```

### Step-by-Step Migration

```typescript
// BEFORE: Untyped form
import { Component } from '@angular/core';
import { FormBuilder } from '@angular/forms';

@Component({
  selector: 'app-old-form',
  template: `<form [formGroup]="form">...</form>`
})
export class OldFormComponent {
  form = this.fb.group({
    name: [''],
    age: [0]
  });

  constructor(private fb: FormBuilder) {}

  // No type safety
  submit() {
    const data = this.form.value; // type: any
    console.log(data.namme); // Typo, no error!
  }
}

// STEP 1: Define interface
interface UserFormValue {
  name: string;
  age: number;
}

// STEP 2: Add explicit types to FormBuilder
export class TypedFormComponent {
  form = this.fb.group({
    name: [''] as FormControl<string>,
    age: [0] as FormControl<number>
  });

  constructor(private fb: FormBuilder) {}
}

// STEP 3: Use NonNullableFormBuilder for better types
export class FullyTypedFormComponent {
  form = this.fb.group({
    name: [''],
    age: [0]
  });

  constructor(private fb: NonNullableFormBuilder) {}

  submit() {
    const data = this.form.value;
    data.name; // type: string
    data.age; // type: number
    // data.namme; // Error: Property does not exist
  }
}
```

### Complex Migration Example

```typescript
// BEFORE: Complex untyped form
import { Component } from '@angular/core';
import { FormArray, FormBuilder, FormGroup } from '@angular/forms';

@Component({
  selector: 'app-complex-old',
  template: `...`
})
export class ComplexOldComponent {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      profile: this.fb.group({
        firstName: [''],
        lastName: ['']
      }),
      addresses: this.fb.array([])
    });
  }

  get addresses(): FormArray {
    return this.form.get('addresses') as FormArray;
  }
}

// AFTER: Typed form with interfaces
interface ProfileForm {
  firstName: string;
  lastName: string;
}

interface AddressForm {
  street: string;
  city: string;
  zipCode: string;
}

interface CompleteForm {
  profile: ProfileForm;
  addresses: AddressForm[];
}

@Component({
  selector: 'app-complex-new',
  template: `...`
})
export class ComplexNewComponent {
  form = new FormGroup({
    profile: new FormGroup({
      firstName: new FormControl<string>('', { nonNullable: true }),
      lastName: new FormControl<string>('', { nonNullable: true })
    }),
    addresses: new FormArray<FormGroup<{
      street: FormControl<string>;
      city: FormControl<string>;
      zipCode: FormControl<string>;
    }>>([])
  });

  private fb = inject(NonNullableFormBuilder);

  get addresses() {
    return this.form.controls.addresses;
  }

  addAddress() {
    this.addresses.push(this.fb.group({
      street: [''],
      city: [''],
      zipCode: ['']
    }));
  }

  submit() {
    const value = this.form.getRawValue();
    // Fully typed!
    value.profile.firstName; // type: string
    value.addresses; // type: AddressForm[]
  }
}
```

## Type Narrowing and Validation

Combining TypeScript types with form validation.

### Type Guards with Forms

```typescript
import { FormControl, Validators } from '@angular/forms';

type EmailOrPhone = { 
  type: 'email'; 
  value: string; 
} | { 
  type: 'phone'; 
  number: string; 
};

const contactControl = new FormControl<EmailOrPhone>({
  type: 'email',
  value: 'user@example.com'
}, { nonNullable: true });

function processContact() {
  const contact = contactControl.value;
  
  // Type narrowing
  if (contact.type === 'email') {
    console.log(contact.value); // OK: type is narrowed
  } else {
    console.log(contact.number); // OK: type is narrowed
  }
}
```

### Discriminated Unions

```typescript
import { Component } from '@angular/core';
import { FormControl, FormGroup } from '@angular/forms';

type PaymentMethod = 
  | { type: 'credit-card'; cardNumber: string; cvv: string; }
  | { type: 'paypal'; email: string; }
  | { type: 'bank-transfer'; accountNumber: string; routingNumber: string; };

@Component({
  selector: 'app-payment',
  template: `
    <form [formGroup]="form">
      <select formControlName="paymentType">
        <option value="credit-card">Credit Card</option>
        <option value="paypal">PayPal</option>
        <option value="bank-transfer">Bank Transfer</option>
      </select>

      <ng-container [ngSwitch]="form.value.paymentType">
        <div *ngSwitchCase="'credit-card'" formGroupName="details">
          <input formControlName="cardNumber" placeholder="Card Number" />
          <input formControlName="cvv" placeholder="CVV" />
        </div>
        <div *ngSwitchCase="'paypal'" formGroupName="details">
          <input formControlName="email" placeholder="PayPal Email" />
        </div>
        <div *ngSwitchCase="'bank-transfer'" formGroupName="details">
          <input formControlName="accountNumber" placeholder="Account" />
          <input formControlName="routingNumber" placeholder="Routing" />
        </div>
      </ng-container>
    </form>
  `
})
export class PaymentComponent {
  form = new FormGroup({
    paymentType: new FormControl<'credit-card' | 'paypal' | 'bank-transfer'>(
      'credit-card', 
      { nonNullable: true }
    ),
    details: new FormGroup({
      // All possible fields
      cardNumber: new FormControl<string>(''),
      cvv: new FormControl<string>(''),
      email: new FormControl<string>(''),
      accountNumber: new FormControl<string>(''),
      routingNumber: new FormControl<string>('')
    })
  });

  processPayment() {
    const type = this.form.value.paymentType;
    const details = this.form.value.details;

    // Validate based on type
    if (type === 'credit-card' && details?.cardNumber && details?.cvv) {
      this.processCreditCard(details.cardNumber, details.cvv);
    } else if (type === 'paypal' && details?.email) {
      this.processPayPal(details.email);
    }
  }

  private processCreditCard(cardNumber: string, cvv: string) {
    // Process credit card
  }

  private processPayPal(email: string) {
    // Process PayPal
  }
}
```

## Advanced Type Patterns

Sophisticated patterns for complex forms.

### Generic Form Type Helper

```typescript
import { FormControl, FormGroup, FormArray } from '@angular/forms';

// Helper type to convert model to form controls
type FormControlsOf<T> = {
  [K in keyof T]: T[K] extends Array<infer U>
    ? FormArray<FormGroup<FormControlsOf<U>>>
    : T[K] extends object
    ? FormGroup<FormControlsOf<T[K]>>
    : FormControl<T[K]>;
};

// Usage
interface User {
  name: string;
  email: string;
  settings: {
    notifications: boolean;
    theme: string;
  };
  addresses: Array<{
    street: string;
    city: string;
  }>;
}

// Automatically generates correct form structure type
type UserFormControls = FormControlsOf<User>;
// Results in:
// {
//   name: FormControl<string>;
//   email: FormControl<string>;
//   settings: FormGroup<{
//     notifications: FormControl<boolean>;
//     theme: FormControl<string>;
//   }>;
//   addresses: FormArray<FormGroup<{
//     street: FormControl<string>;
//     city: FormControl<string>;
//   }>>;
// }

const userForm: FormGroup<UserFormControls> = new FormGroup({
  name: new FormControl<string>('', { nonNullable: true }),
  email: new FormControl<string>('', { nonNullable: true }),
  settings: new FormGroup({
    notifications: new FormControl<boolean>(true, { nonNullable: true }),
    theme: new FormControl<string>('light', { nonNullable: true })
  }),
  addresses: new FormArray<FormGroup<{
    street: FormControl<string>;
    city: FormControl<string>;
  }>>([])
});
```

### Type-Safe Form Factory

```typescript
import { FormBuilder, FormGroup, NonNullableFormBuilder } from '@angular/forms';
import { inject } from '@angular/core';

class TypedFormFactory {
  private fb = inject(NonNullableFormBuilder);

  createForm<T extends Record<string, any>>(
    model: T
  ): FormGroup<FormControlsOf<T>> {
    const controls: any = {};

    for (const [key, value] of Object.entries(model)) {
      if (Array.isArray(value)) {
        controls[key] = this.fb.array([]);
      } else if (typeof value === 'object' && value !== null) {
        controls[key] = this.createForm(value);
      } else {
        controls[key] = this.fb.control(value);
      }
    }

    return this.fb.group(controls) as FormGroup<FormControlsOf<T>>;
  }
}

// Usage
const factory = new TypedFormFactory();

const userModel = {
  name: '',
  age: 0,
  settings: {
    notifications: true,
    theme: 'light'
  }
};

const form = factory.createForm(userModel);
// Fully typed based on model!
form.value.name; // type: string
form.value.settings?.theme; // type: string
```

### Conditional Form Types

```typescript
import { FormControl, FormGroup } from '@angular/forms';

// Different form shapes based on user role
type AdminForm = {
  username: string;
  permissions: string[];
  department: string;
};

type UserForm = {
  username: string;
  email: string;
};

type RoleBasedForm<T extends 'admin' | 'user'> = 
  T extends 'admin' ? AdminForm : UserForm;

function createRoleForm<T extends 'admin' | 'user'>(
  role: T
): FormGroup<FormControlsOf<RoleBasedForm<T>>> {
  if (role === 'admin') {
    return new FormGroup({
      username: new FormControl<string>('', { nonNullable: true }),
      permissions: new FormControl<string[]>([], { nonNullable: true }),
      department: new FormControl<string>('', { nonNullable: true })
    }) as any;
  } else {
    return new FormGroup({
      username: new FormControl<string>('', { nonNullable: true }),
      email: new FormControl<string>('', { nonNullable: true })
    }) as any;
  }
}

const adminForm = createRoleForm('admin');
adminForm.value.permissions; // Available for admin

const userForm = createRoleForm('user');
userForm.value.email; // Available for user
// userForm.value.permissions; // Error: doesn't exist
```

## Common Mistakes

### 1. Forgetting NonNullable Option

```typescript
// WRONG: Reset sets value to null
const control = new FormControl<string>('initial');
control.reset();
console.log(control.value); // null (unexpected!)

// CORRECT: Reset returns to initial value
const control2 = new FormControl<string>('initial', { nonNullable: true });
control2.reset();
console.log(control2.value); // 'initial'
```

### 2. Mismatching Types

```typescript
// WRONG: Type mismatch
const form = new FormGroup({
  age: new FormControl<number>(0, { nonNullable: true })
});

// Runtime: input returns string, but type says number
form.patchValue({ age: '25' as any }); // Type assertion hides the problem

// CORRECT: Parse input values
const ageValue = parseInt(inputElement.value, 10);
form.patchValue({ age: ageValue });
```

### 3. Incorrect FormArray Types

```typescript
// WRONG: Loose typing
const array = new FormArray([
  new FormControl('item1'),
  new FormControl('item2')
]);

// CORRECT: Explicit type
const array2 = new FormArray<FormControl<string>>([
  new FormControl<string>('item1', { nonNullable: true }),
  new FormControl<string>('item2', { nonNullable: true })
]);
```

### 4. Not Handling Disabled Controls

```typescript
// WRONG: Disabled field value is missing
form.controls.id.disable();
const data = form.value; // id is missing!
saveToApi(data); // Incomplete data

// CORRECT: Use getRawValue()
const completeData = form.getRawValue();
saveToApi(completeData); // Includes disabled fields
```

## Best Practices

### 1. Define Interfaces First

```typescript
// Define your data model
interface User {
  id: number;
  name: string;
  email: string;
}

// Then create forms based on the model
const userForm = new FormGroup({
  id: new FormControl<number>(0, { nonNullable: true }),
  name: new FormControl<string>('', { nonNullable: true }),
  email: new FormControl<string>('', { nonNullable: true })
});

// Type-safe conversion
function toUser(): User {
  return this.userForm.getRawValue() as User;
}
```

### 2. Use NonNullableFormBuilder by Default

```typescript
import { Component, inject } from '@angular/core';
import { NonNullableFormBuilder } from '@angular/forms';

@Component({
  selector: 'app-form',
  template: `...`
})
export class FormComponent {
  private fb = inject(NonNullableFormBuilder);

  // Cleaner syntax, non-nullable by default
  form = this.fb.group({
    name: [''],
    email: [''],
    age: [0]
  });
}
```

### 3. Create Reusable Form Factories

```typescript
import { FormGroup, NonNullableFormBuilder } from '@angular/forms';
import { inject, Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AddressFormFactory {
  private fb = inject(NonNullableFormBuilder);

  createAddressForm() {
    return this.fb.group({
      street: [''],
      city: [''],
      state: [''],
      zipCode: ['']
    });
  }
}

// Usage in components
@Component({...})
export class CheckoutComponent {
  private addressFactory = inject(AddressFormFactory);

  form = this.fb.group({
    billing: this.addressFactory.createAddressForm(),
    shipping: this.addressFactory.createAddressForm()
  });
}
```

### 4. Type Guards for Complex Forms

```typescript
function isValidFormValue<T>(value: Partial<T>): value is T {
  // Add runtime validation
  return Object.values(value).every(v => v !== null && v !== undefined);
}

// Usage
if (isValidFormValue<User>(form.value)) {
  // TypeScript knows form.value is complete User
  saveUser(form.value);
}
```

## Interview Questions

### Q1: What are the benefits of typed forms in Angular 14+?

**Answer:** Typed forms provide:
- Compile-time type checking for form values
- Autocompletion and IntelliSense for form controls
- Refactoring safety (renaming fields updates all references)
- Elimination of runtime errors from typos
- Better documentation through types
- Type inference that propagates through form hierarchies

### Q2: What's the difference between value and getRawValue()?

**Answer:** `value` excludes disabled controls, while `getRawValue()` includes them. Example:
```typescript
form.controls.id.disable();
form.value.id; // undefined (disabled)
form.getRawValue().id; // has value (includes disabled)
```

### Q3: How do non-nullable controls work?

**Answer:** Controls with `{ nonNullable: true }` never return null. On reset, they return to their initial value instead of null. Example:
```typescript
const control = new FormControl('initial', { nonNullable: true });
control.reset(); // value is 'initial', not null
control.value; // type: string (not string | null)
```

### Q4: How do you create a FormArray with proper types?

**Answer:**
```typescript
const array = new FormArray<FormControl<string>>([
  new FormControl<string>('', { nonNullable: true })
]);
array.value; // type: string[]
```

### Q5: What's NonNullableFormBuilder?

**Answer:** A FormBuilder variant that creates non-nullable controls by default, eliminating the need to specify `{ nonNullable: true }` on each control.

## Key Takeaways

1. **Typed forms provide compile-time safety** - Catch errors before runtime
2. **Use NonNullableFormBuilder** - Simplifies form creation with better defaults
3. **Understand nullable vs non-nullable** - Reset behavior differs significantly
4. **getRawValue() includes disabled controls** - Use it for complete form data
5. **Define interfaces first** - Model-driven approach improves type safety
6. **FormArray needs explicit types** - Specify control types for proper inference
7. **Disabled controls affect form.value** - They're excluded but present in getRawValue
8. **Migration is gradual** - UntypedForm* classes support incremental migration
9. **Type narrowing works with forms** - Discriminated unions enable type-safe conditional forms
10. **Generic helpers improve reusability** - Create factories and type utilities

## Resources

- [Angular Typed Forms Guide](https://angular.io/guide/typed-forms)
- [Angular 14 Release - Typed Forms](https://blog.angular.io/angular-v14-is-now-available-391a6db736af)
- [FormControl API](https://angular.io/api/forms/FormControl)
- [NonNullableFormBuilder API](https://angular.io/api/forms/NonNullableFormBuilder)
- [TypeScript Advanced Types](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)
- [RxJS with Typed Forms](https://www.learnrxjs.io/)
