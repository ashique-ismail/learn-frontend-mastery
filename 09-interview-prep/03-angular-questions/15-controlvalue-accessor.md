# ControlValueAccessor — When and How

## What It Is

`ControlValueAccessor` (CVA) is an Angular interface that creates a **bridge between a custom component and Angular's form system**. Implementing it lets your component work seamlessly with both template-driven forms (`[(ngModel)]`) and reactive forms (`formControlName`).

Without CVA, custom form controls are opaque to Angular — the form can't read their value, mark them dirty/touched, or run validators.

---

## The Interface

```ts
interface ControlValueAccessor {
  writeValue(value: any): void;             // form → component (set value)
  registerOnChange(fn: Function): void;     // component tells form when value changes
  registerOnTouched(fn: Function): void;    // component tells form when touched
  setDisabledState?(disabled: boolean): void; // form disables the component
}
```

---

## Minimal Implementation: Star Rating

```ts
import { Component, forwardRef, signal } from '@angular/core';
import {
  ControlValueAccessor,
  NG_VALUE_ACCESSOR,
  ReactiveFormsModule,
} from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  standalone: true,
  template: `
    <div class="stars" [attr.aria-label]="'Rating: ' + value() + ' of 5'">
      @for (star of [1,2,3,4,5]; track star) {
        <button
          type="button"
          (click)="onStarClick(star)"
          (blur)="onTouched()"
          [disabled]="disabled()"
          [attr.aria-label]="star + ' star'"
        >
          {{ star <= value() ? '★' : '☆' }}
        </button>
      }
    </div>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => StarRatingComponent),
      multi: true, // allows multiple CVA providers on same element
    },
  ],
})
export class StarRatingComponent implements ControlValueAccessor {
  value = signal(0);
  disabled = signal(false);

  // Registered callbacks — Angular calls these to wire up the form
  private onChange: (value: number) => void = () => {};
  onTouched: () => void = () => {};

  // Angular calls this to push the form control's value into the component
  writeValue(value: number): void {
    this.value.set(value ?? 0);
  }

  // Angular registers its change notification callback
  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  // Angular registers its touched notification callback
  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  // Angular calls this when the form control is disabled/enabled
  setDisabledState(disabled: boolean): void {
    this.disabled.set(disabled);
  }

  onStarClick(star: number): void {
    this.value.set(star);
    this.onChange(star);    // notify Angular form of new value
    this.onTouched();       // mark as touched
  }
}
```

---

## Using the Custom Control

```ts
// Template-driven
<app-star-rating [(ngModel)]="rating" name="rating" required></app-star-rating>

// Reactive forms
form = new FormGroup({
  rating: new FormControl(0, Validators.required),
});
<app-star-rating formControlName="rating"></app-star-rating>
```

Angular handles validation, dirty/pristine state, enabled/disabled state automatically.

---

## Example with Async Options: Color Picker

```ts
@Component({
  selector: 'app-color-picker',
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => ColorPickerComponent),
    multi: true,
  }],
  template: `
    <input
      type="color"
      [value]="currentValue"
      (input)="onInput($event)"
      (blur)="onTouched()"
      [disabled]="isDisabled"
    />
  `
})
export class ColorPickerComponent implements ControlValueAccessor {
  currentValue = '#000000';
  isDisabled = false;
  private onChange: (v: string) => void = () => {};
  onTouched: () => void = () => {};

  writeValue(value: string): void {
    this.currentValue = value || '#000000';
  }
  registerOnChange(fn: (v: string) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  setDisabledState(disabled: boolean): void { this.isDisabled = disabled; }

  onInput(event: Event): void {
    const color = (event.target as HTMLInputElement).value;
    this.currentValue = color;
    this.onChange(color);
  }
}
```

---

## Custom Validators on CVA Components

```ts
// Add NG_VALIDATORS to expose validation
providers: [
  { provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => Comp), multi: true },
  { provide: NG_VALIDATORS, useExisting: forwardRef(() => Comp), multi: true },
]

// Implement Validator interface
export class Comp implements ControlValueAccessor, Validator {
  validate(control: AbstractControl): ValidationErrors | null {
    return control.value < 1 ? { required: true } : null;
  }
}
```

---

## Common Interview Questions

**Q: Why do you need `forwardRef` in the providers array?**
The component class isn't fully defined when the providers array is evaluated. `forwardRef(() => ComponentClass)` creates a forward reference that Angular resolves lazily, after the class is defined.

**Q: What does `multi: true` mean?**
It tells Angular that multiple values can be registered for this injection token. Angular collects all CVA providers on an element into an array and picks the correct one. Without `multi: true`, your CVA would override Angular's built-in `DefaultValueAccessor`.

**Q: When would you use CVA over just passing `(valueChange)` outputs?**
CVA when you want full form integration: validation (required, disabled, pristine/dirty/touched state), easy use with `formControlName` and `ngModel`, and Reactive Forms support. Use `(valueChange)` for simple parent-child communication that doesn't need to integrate with Angular Forms.
