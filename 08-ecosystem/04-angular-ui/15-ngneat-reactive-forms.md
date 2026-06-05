# @ngneat/reactive-forms

## The Idea

**In plain English:** `@ngneat/reactive-forms` is a library that wraps Angular's built-in form tools so that TypeScript (the programming language layer that checks your code for mistakes) always knows exactly what kind of data each form field holds — like knowing a field must be a number, not just "anything." This matters because without it, TypeScript would treat every form value as an unknown blob, letting bugs slip through silently.

**Real-world analogy:** Imagine a restaurant order form where each box is labelled with exactly what goes in it — "Table number (digits only)", "Dish name (text only)", "Quantity (number only)". A manager reviews every completed form and flags it immediately if someone wrote "banana" in the "Table number" box, before the order even reaches the kitchen.

- The labelled boxes = typed FormControls (each field declares what data type it accepts)
- The completed order form = the FormGroup (holds all the fields together as one typed object)
- The manager who checks the form = TypeScript's compiler (catches type mismatches at coding time, not at runtime)

---

## Overview

`@ngneat/reactive-forms` is a strongly-typed wrapper around Angular's built-in Reactive Forms. Its primary value was delivering full TypeScript generics support — knowing the exact shape of each control's value — before Angular shipped native typed forms in v14. The library also adds convenience APIs that reduce boilerplate for common patterns.

> **Note:** Angular 14+ ships typed reactive forms natively. For new projects, prefer Angular's built-in typed forms. Use `@ngneat/reactive-forms` when: (a) you're on Angular 13 or earlier, (b) you need the library's specific extras (control streams, `markAllAsDirty`, etc.), or (c) you're maintaining an existing codebase already using it.

## Installation

```bash
npm install @ngneat/reactive-forms
```

## Core Concepts

### Typed FormControl, FormGroup, FormArray

```typescript
import {
  FormControl,
  FormGroup,
  FormArray,
} from '@ngneat/reactive-forms';

// Typed FormControl — value is string, not any
const nameControl = new FormControl<string>('');
const ageControl  = new FormControl<number | null>(null);

// TypeScript knows nameControl.value is string
const name: string = nameControl.value; // ✓ — no cast needed

// Typed FormGroup
interface LoginForm {
  email: string;
  password: string;
  rememberMe: boolean;
}

const loginForm = new FormGroup<LoginForm>({
  email:      new FormControl<string>(''),
  password:   new FormControl<string>(''),
  rememberMe: new FormControl<boolean>(false),
});

// loginForm.value is LoginForm, not any
const { email, password } = loginForm.value; // both typed as string
```

### Typed FormArray

```typescript
interface Address {
  street: string;
  city: string;
  zip: string;
}

const addressesArray = new FormArray<Address>([
  new FormGroup<Address>({
    street: new FormControl<string>(''),
    city:   new FormControl<string>(''),
    zip:    new FormControl<string>(''),
  }),
]);

// .value is Address[], not any[]
const addresses: Address[] = addressesArray.value;
```

## Reactive Helpers

### Observable Streams on Controls

`@ngneat/reactive-forms` exposes observable properties directly on controls, eliminating `.valueChanges.pipe(...)` boilerplate.

```typescript
import { FormControl } from '@ngneat/reactive-forms';

const searchControl = new FormControl<string>('');

// value$ emits the current value immediately, then on every change
searchControl.value$
  .pipe(debounceTime(300), distinctUntilChanged())
  .subscribe((term) => this.search(term));

// status$ — 'VALID' | 'INVALID' | 'DISABLED' | 'PENDING'
searchControl.status$.subscribe((status) => console.log(status));

// disabled$ / enabled$ — boolean observables
searchControl.disabled$.subscribe((disabled) => {
  // update UI affordances
});

// touch$ — emits when control is touched
searchControl.touch$.subscribe(() => this.showValidationError());
```

### Reactive Group Methods

```typescript
const form = new FormGroup<UserForm>({
  username: new FormControl<string>(''),
  email:    new FormControl<string>(''),
  age:      new FormControl<number | null>(null),
});

// Typed setValue / patchValue — argument must match UserForm shape
form.setValue({ username: 'alice', email: 'a@b.com', age: 30 });
form.patchValue({ email: 'new@b.com' });  // partial OK

// markAllAsDirty — built-in helper (not in Angular core until much later)
form.markAllAsDirty();

// reset with typed value
form.reset({ username: '', email: '', age: null });
```

## Validation Utilities

```typescript
import {
  FormControl,
  FormGroup,
  Validators,
} from '@ngneat/reactive-forms';

const profileForm = new FormGroup<ProfileForm>({
  username: new FormControl<string>('', [
    Validators.required,
    Validators.minLength(3),
    Validators.maxLength(20),
  ]),
  bio: new FormControl<string | null>(null),
});

// Async validator — same API as Angular core
const emailControl = new FormControl<string>(
  '',
  Validators.required,
  async (ctrl) => {
    const taken = await this.authService.isEmailTaken(ctrl.value);
    return taken ? { emailTaken: true } : null;
  }
);
```

## Usage in Components

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { ReactiveFormsModule } from '@angular/forms';
import { FormControl, FormGroup } from '@ngneat/reactive-forms';

interface RegistrationForm {
  name:     string;
  email:    string;
  password: string;
}

@Component({
  selector: 'app-registration',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="name"     placeholder="Name" />
      <input formControlName="email"    placeholder="Email" type="email" />
      <input formControlName="password" placeholder="Password" type="password" />

      <div *ngIf="form.controls.email.invalid && form.controls.email.touched">
        Invalid email
      </div>

      <button type="submit" [disabled]="form.invalid">Register</button>
    </form>
  `,
})
export class RegistrationComponent {
  form = new FormGroup<RegistrationForm>({
    name:     new FormControl<string>('', Validators.required),
    email:    new FormControl<string>('', [Validators.required, Validators.email]),
    password: new FormControl<string>('', [Validators.required, Validators.minLength(8)]),
  });

  submit(): void {
    if (this.form.valid) {
      // form.value is RegistrationForm — fully typed
      const value: RegistrationForm = this.form.value;
      console.log(value);
    }
  }
}
```

## Migration Guide: @ngneat/reactive-forms → Angular Native Typed Forms

Angular 14 introduced native typed forms. Here's a side-by-side comparison:

```typescript
// @ngneat/reactive-forms
import { FormControl, FormGroup } from '@ngneat/reactive-forms';

const form = new FormGroup<LoginForm>({
  email:    new FormControl<string>(''),
  password: new FormControl<string>(''),
});

// Angular 14+ native typed forms
import { FormControl, FormGroup } from '@angular/forms';

const form = new FormGroup({
  email:    new FormControl(''),       // inferred as FormControl<string>
  password: new FormControl(''),
});

// Explicit typing when needed
const ageControl = new FormControl<number | null>(null);
```

Key differences when migrating:
- Angular 14 uses inference rather than an explicit generic on `FormGroup`
- `value$` / `status$` streams are not available natively — keep using `.valueChanges` / `.statusChanges`
- `markAllAsDirty()` is available natively as of Angular 14
- The `@ngneat/reactive-forms` package can be dropped once all usages are migrated

## When to Still Use @ngneat/reactive-forms

| Scenario | Recommendation |
|---|---|
| Angular < 14 | Use `@ngneat/reactive-forms` |
| Angular ≥ 14, greenfield | Use native typed forms |
| Angular ≥ 14, existing codebase using library | Migrate gradually; library still works |
| Need `value$` / `status$` observables | Keep the library OR add them manually |
| Need `markAllAsDirty` on Angular < 14 | Keep the library |

## Interview Questions

**Q: What problem did @ngneat/reactive-forms solve before Angular 14?**  
A: Angular's built-in reactive forms had no TypeScript generics — `form.value` was typed as `any`. `@ngneat/reactive-forms` wrapped those classes with generics so TypeScript could validate that you were setting/getting values that matched the declared form shape.

**Q: What does the `value$` helper give you over `valueChanges`?**  
A: `valueChanges` only emits on future changes. `value$` (from `@ngneat/reactive-forms`) emits the current value immediately on subscription, then continues like `valueChanges`. This is equivalent to `startWith(control.value)` piped onto `valueChanges`.

**Q: Why might you choose native typed forms over this library today?**  
A: For Angular 14+, native typed forms have no extra dependency and cover the main use case (type-safe values). The library adds maintenance overhead and a larger API surface. Prefer the built-in unless you specifically need the observable helpers or are on an older Angular version.
