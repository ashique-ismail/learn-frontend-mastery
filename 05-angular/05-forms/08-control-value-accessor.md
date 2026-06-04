# ControlValueAccessor in Angular

## Overview

ControlValueAccessor is the bridge between Angular forms and custom form controls. It enables any component to integrate seamlessly with both template-driven and reactive forms by implementing a standard interface for reading, writing, and validating values.

Understanding ControlValueAccessor allows you to create reusable, form-integrated components like custom date pickers, rating widgets, rich text editors, and composite form controls that behave exactly like native form elements.

## Table of Contents

1. [The ControlValueAccessor Interface](#the-controlvalueaccessor-interface)
2. [Basic Implementation](#basic-implementation)
3. [Registering with NG_VALUE_ACCESSOR](#registering-with-ng_value_accessor)
4. [Handling Touched State](#handling-touched-state)
5. [Disabled State](#disabled-state)
6. [Integration with Validators](#integration-with-validators)
7. [Composite Form Controls](#composite-form-controls)
8. [Common Patterns](#common-patterns)
9. [Testing Custom Controls](#testing-custom-controls)
10. [Real-World Examples](#real-world-examples)

## The ControlValueAccessor Interface

Understanding the four methods that make up the interface.

### Interface Definition

```typescript
import { ControlValueAccessor } from '@angular/forms';

interface ControlValueAccessor {
  // Called by forms API to write value to the view
  writeValue(value: any): void;

  // Called by forms API to register callback for when value changes
  registerOnChange(fn: (value: any) => void): void;

  // Called by forms API to register callback for when control is touched
  registerOnTouched(fn: () => void): void;

  // Called by forms API to enable/disable the control (optional)
  setDisabledState?(isDisabled: boolean): void;
}
```

### Method Flow

```typescript
/**
 * Flow of ControlValueAccessor:
 * 
 * 1. Form writes value -> writeValue(value)
 *    - Updates internal component state
 *    - Renders value in view
 * 
 * 2. User interacts with component
 *    - Component calls onChange(newValue)
 *    - Forms API receives new value
 *    - Form model updated
 * 
 * 3. User leaves component
 *    - Component calls onTouched()
 *    - Control marked as touched
 *    - Validation errors may display
 * 
 * 4. Form disables control -> setDisabledState(true)
 *    - Component updates disabled state
 *    - User cannot interact
 */

@Component({
  selector: 'app-custom-input',
  template: `<input [value]="value" (input)="onInput($event)" (blur)="onBlur()" />`
})
export class CustomInputComponent implements ControlValueAccessor {
  value: string = '';
  
  // Store callbacks from forms API
  private onChange: (value: any) => void = () => {};
  private onTouched: () => void = () => {};

  // 1. Form writes to component
  writeValue(value: string): void {
    this.value = value || '';
  }

  // 2. Register change callback
  registerOnChange(fn: (value: any) => void): void {
    this.onChange = fn;
  }

  // 3. Register touched callback
  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  // User types in input
  onInput(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    this.value = value;
    this.onChange(value); // Notify forms API
  }

  // User leaves input
  onBlur(): void {
    this.onTouched(); // Notify forms API
  }
}
```

## Basic Implementation

Creating a simple custom form control.

### Counter Component

```typescript
import { Component, forwardRef } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <button (click)="decrement()" [disabled]="disabled">-</button>
      <span>{{ value }}</span>
      <button (click)="increment()" [disabled]="disabled">+</button>
    </div>
  `,
  styles: [`
    .counter {
      display: flex;
      align-items: center;
      gap: 1rem;
    }
    button {
      padding: 0.5rem 1rem;
      font-size: 1.5rem;
    }
    span {
      min-width: 3rem;
      text-align: center;
      font-size: 1.5rem;
    }
  `],
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => CounterComponent),
    multi: true
  }]
})
export class CounterComponent implements ControlValueAccessor {
  value: number = 0;
  disabled: boolean = false;

  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  increment(): void {
    if (!this.disabled) {
      this.value++;
      this.onChange(this.value);
      this.onTouched();
    }
  }

  decrement(): void {
    if (!this.disabled) {
      this.value--;
      this.onChange(this.value);
      this.onTouched();
    }
  }

  writeValue(value: number): void {
    this.value = value || 0;
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}

// Usage in forms
@Component({
  selector: 'app-form',
  template: `
    <form [formGroup]="form">
      <app-counter formControlName="quantity"></app-counter>
      <div>Quantity: {{ form.value.quantity }}</div>
    </form>
  `
})
export class FormComponent {
  form = new FormGroup({
    quantity: new FormControl(5, [Validators.min(0), Validators.max(10)])
  });
}
```

### Rating Component

```typescript
@Component({
  selector: 'app-rating',
  template: `
    <div class="rating">
      <span *ngFor="let star of stars; let i = index"
            class="star"
            [class.filled]="i < value"
            [class.disabled]="disabled"
            (click)="rate(i + 1)"
            (mouseenter)="hover = i + 1"
            (mouseleave)="hover = 0">
        {{ i < (hover || value) ? '★' : '☆' }}
      </span>
    </div>
  `,
  styles: [`
    .rating {
      display: flex;
      gap: 0.25rem;
    }
    .star {
      font-size: 2rem;
      cursor: pointer;
      color: #ddd;
      transition: color 0.2s;
    }
    .star.filled,
    .star:hover {
      color: #ffc107;
    }
    .star.disabled {
      cursor: not-allowed;
      opacity: 0.5;
    }
  `],
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => RatingComponent),
    multi: true
  }]
})
export class RatingComponent implements ControlValueAccessor {
  @Input() max: number = 5;
  
  value: number = 0;
  hover: number = 0;
  disabled: boolean = false;
  stars: number[] = [];

  private onChange: (value: number) => void = () => {};
  private onTouched: () => void = () => {};

  ngOnInit() {
    this.stars = Array(this.max).fill(0).map((_, i) => i);
  }

  rate(rating: number): void {
    if (!this.disabled) {
      this.value = rating;
      this.onChange(rating);
      this.onTouched();
    }
  }

  writeValue(value: number): void {
    this.value = value || 0;
  }

  registerOnChange(fn: (value: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}

// Usage
@Component({
  selector: 'app-review-form',
  template: `
    <form [formGroup]="form">
      <label>Rating</label>
      <app-rating formControlName="rating" [max]="5"></app-rating>
      
      <div *ngIf="form.controls.rating.invalid && form.controls.rating.touched">
        Rating is required
      </div>
    </form>
  `
})
export class ReviewFormComponent {
  form = new FormGroup({
    rating: new FormControl(0, [Validators.required, Validators.min(1)])
  });
}
```

## Registering with NG_VALUE_ACCESSOR

Proper registration of custom controls.

### Provider Configuration

```typescript
import { Component, forwardRef, Provider } from '@angular/core';
import { NG_VALUE_ACCESSOR } from '@angular/forms';

// Method 1: Inline provider
@Component({
  selector: 'app-custom-control',
  template: `...`,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => CustomControlComponent),
    multi: true  // Important: multi must be true
  }]
})
export class CustomControlComponent implements ControlValueAccessor {
  // Implementation
}

// Method 2: Extracted provider constant
const CUSTOM_CONTROL_VALUE_ACCESSOR: Provider = {
  provide: NG_VALUE_ACCESSOR,
  useExisting: forwardRef(() => CustomControlComponent),
  multi: true
};

@Component({
  selector: 'app-custom-control',
  template: `...`,
  providers: [CUSTOM_CONTROL_VALUE_ACCESSOR]
})
export class CustomControlComponent implements ControlValueAccessor {
  // Implementation
}

// Method 3: Factory provider (advanced)
function createControlValueAccessor(component: any): Provider {
  return {
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => component),
    multi: true
  };
}

@Component({
  selector: 'app-custom-control',
  template: `...`,
  providers: [createControlValueAccessor(CustomControlComponent)]
})
export class CustomControlComponent implements ControlValueAccessor {
  // Implementation
}
```

### Why forwardRef?

```typescript
/**
 * forwardRef is needed because the class is referenced
 * before it's defined (in the decorator).
 * 
 * Without forwardRef, you'd get: "Cannot access 'CustomControlComponent'
 * before initialization"
 */

// This breaks (circular reference):
@Component({
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: CustomControlComponent, // Error!
    multi: true
  }]
})
export class CustomControlComponent {}

// This works:
@Component({
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => CustomControlComponent), // OK
    multi: true
  }]
})
export class CustomControlComponent {}
```

## Handling Touched State

Managing the touched state properly.

### Touch Strategies

```typescript
@Component({
  selector: 'app-input-with-touch',
  template: `
    <input 
      [value]="value"
      (input)="onInput($event)"
      (blur)="markAsTouched()"
      (focus)="onFocus()"
      [disabled]="disabled" />
  `,
  providers: [/* NG_VALUE_ACCESSOR */]
})
export class InputWithTouchComponent implements ControlValueAccessor {
  value: string = '';
  disabled: boolean = false;
  private touched: boolean = false;

  private onChange: (value: string) => void = () => {};
  private onTouched: () => void = () => {};

  // Strategy 1: Mark as touched on blur
  markAsTouched(): void {
    if (!this.touched) {
      this.touched = true;
      this.onTouched();
    }
  }

  // Strategy 2: Mark as touched on first change
  onInput(event: Event): void {
    const value = (event.target as HTMLInputElement).value;
    this.value = value;
    this.onChange(value);
    
    // Mark touched on first change
    if (!this.touched) {
      this.touched = true;
      this.onTouched();
    }
  }

  // Strategy 3: Mark as touched when user leaves after editing
  private originalValue: string = '';

  onFocus(): void {
    this.originalValue = this.value;
  }

  onBlurWithChangeCheck(): void {
    if (this.value !== this.originalValue && !this.touched) {
      this.touched = true;
      this.onTouched();
    }
  }

  writeValue(value: string): void {
    this.value = value || '';
    this.originalValue = this.value;
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}
```

### Multiple Touch Points

```typescript
@Component({
  selector: 'app-date-range-picker',
  template: `
    <div class="date-range">
      <input type="date"
             [value]="startDate"
             (change)="onStartDateChange($event)"
             (blur)="markAsTouched()"
             [disabled]="disabled" />
      <span>to</span>
      <input type="date"
             [value]="endDate"
             (change)="onEndDateChange($event)"
             (blur)="markAsTouched()"
             [disabled]="disabled" />
    </div>
  `,
  providers: [/* NG_VALUE_ACCESSOR */]
})
export class DateRangePickerComponent implements ControlValueAccessor {
  startDate: string = '';
  endDate: string = '';
  disabled: boolean = false;
  private touched: boolean = false;

  private onChange: (value: DateRange) => void = () => {};
  private onTouched: () => void = () => {};

  onStartDateChange(event: Event): void {
    this.startDate = (event.target as HTMLInputElement).value;
    this.emitValue();
  }

  onEndDateChange(event: Event): void {
    this.endDate = (event.target as HTMLInputElement).value;
    this.emitValue();
  }

  markAsTouched(): void {
    if (!this.touched) {
      this.touched = true;
      this.onTouched();
    }
  }

  private emitValue(): void {
    this.onChange({
      startDate: this.startDate,
      endDate: this.endDate
    });
  }

  writeValue(value: DateRange): void {
    if (value) {
      this.startDate = value.startDate || '';
      this.endDate = value.endDate || '';
    } else {
      this.startDate = '';
      this.endDate = '';
    }
  }

  registerOnChange(fn: (value: DateRange) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}

interface DateRange {
  startDate: string;
  endDate: string;
}
```

## Disabled State

Properly handling disabled state.

### Full Disabled Support

```typescript
@Component({
  selector: 'app-toggle-switch',
  template: `
    <div class="toggle-switch"
         [class.disabled]="disabled"
         [class.checked]="value"
         (click)="toggle()"
         [attr.tabindex]="disabled ? -1 : 0"
         (keydown.space)="toggle()">
      <div class="switch-handle"></div>
      <span class="label">{{ value ? 'On' : 'Off' }}</span>
    </div>
  `,
  styles: [`
    .toggle-switch {
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
      cursor: pointer;
      user-select: none;
    }
    .toggle-switch.disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
    .switch-handle {
      width: 50px;
      height: 24px;
      background: #ccc;
      border-radius: 12px;
      position: relative;
      transition: background 0.3s;
    }
    .switch-handle::after {
      content: '';
      position: absolute;
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: white;
      top: 2px;
      left: 2px;
      transition: transform 0.3s;
    }
    .toggle-switch.checked .switch-handle {
      background: #4caf50;
    }
    .toggle-switch.checked .switch-handle::after {
      transform: translateX(26px);
    }
  `],
  providers: [/* NG_VALUE_ACCESSOR */]
})
export class ToggleSwitchComponent implements ControlValueAccessor {
  value: boolean = false;
  disabled: boolean = false;

  private onChange: (value: boolean) => void = () => {};
  private onTouched: () => void = () => {};

  toggle(): void {
    if (!this.disabled) {
      this.value = !this.value;
      this.onChange(this.value);
      this.onTouched();
    }
  }

  writeValue(value: boolean): void {
    this.value = !!value;
  }

  registerOnChange(fn: (value: boolean) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}

// Usage with disabled control
@Component({
  selector: 'app-form',
  template: `
    <form [formGroup]="form">
      <app-toggle-switch formControlName="notifications"></app-toggle-switch>
      <button (click)="toggleDisabled()">
        {{ form.controls.notifications.disabled ? 'Enable' : 'Disable' }}
      </button>
    </form>
  `
})
export class FormComponent {
  form = new FormGroup({
    notifications: new FormControl(true)
  });

  toggleDisabled() {
    const control = this.form.controls.notifications;
    control.disabled ? control.enable() : control.disable();
  }
}
```

## Integration with Validators

Adding validation support to custom controls.

### Custom Control with Validation

```typescript
import { Component, forwardRef } from '@angular/core';
import { 
  ControlValueAccessor, 
  NG_VALUE_ACCESSOR,
  NG_VALIDATORS,
  AbstractControl,
  ValidationErrors,
  Validator
} from '@angular/forms';

@Component({
  selector: 'app-email-input',
  template: `
    <div class="email-input">
      <input type="email"
             [value]="value"
             (input)="onInput($event)"
             (blur)="onBlur()"
             [disabled]="disabled"
             placeholder="Enter email" />
      <span class="validation-icon" *ngIf="value">
        {{ isValidEmail() ? '✓' : '✗' }}
      </span>
    </div>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => EmailInputComponent),
      multi: true
    },
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => EmailInputComponent),
      multi: true
    }
  ]
})
export class EmailInputComponent implements ControlValueAccessor, Validator {
  value: string = '';
  disabled: boolean = false;

  private onChange: (value: string) => void = () => {};
  private onTouched: () => void = () => {};

  // Validator implementation
  validate(control: AbstractControl): ValidationErrors | null {
    if (!this.value) {
      return null;
    }

    return this.isValidEmail() ? null : { invalidEmail: { value: this.value } };
  }

  isValidEmail(): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(this.value);
  }

  onInput(event: Event): void {
    this.value = (event.target as HTMLInputElement).value;
    this.onChange(this.value);
  }

  onBlur(): void {
    this.onTouched();
  }

  writeValue(value: string): void {
    this.value = value || '';
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}

// Usage - validation happens automatically
@Component({
  selector: 'app-form',
  template: `
    <form [formGroup]="form">
      <app-email-input formControlName="email"></app-email-input>
      <div *ngIf="form.controls.email.invalid && form.controls.email.touched">
        Invalid email format
      </div>
    </form>
  `
})
export class FormComponent {
  form = new FormGroup({
    email: new FormControl('', Validators.required)
    // Custom validation from component automatically applied
  });
}
```

## Composite Form Controls

Creating controls that manage multiple sub-controls.

### Address Input Component

```typescript
interface Address {
  street: string;
  city: string;
  state: string;
  zipCode: string;
}

@Component({
  selector: 'app-address-input',
  template: `
    <div class="address-input" [class.disabled]="disabled">
      <div class="field">
        <label>Street</label>
        <input [(ngModel)]="address.street"
               (ngModelChange)="onAddressChange()"
               (blur)="markAsTouched()"
               [disabled]="disabled" />
      </div>
      
      <div class="field">
        <label>City</label>
        <input [(ngModel)]="address.city"
               (ngModelChange)="onAddressChange()"
               (blur)="markAsTouched()"
               [disabled]="disabled" />
      </div>
      
      <div class="field-row">
        <div class="field">
          <label>State</label>
          <select [(ngModel)]="address.state"
                  (ngModelChange)="onAddressChange()"
                  (blur)="markAsTouched()"
                  [disabled]="disabled">
            <option value="">Select State</option>
            <option *ngFor="let state of states" [value]="state">
              {{ state }}
            </option>
          </select>
        </div>
        
        <div class="field">
          <label>ZIP Code</label>
          <input [(ngModel)]="address.zipCode"
                 (ngModelChange)="onAddressChange()"
                 (blur)="markAsTouched()"
                 [disabled]="disabled"
                 maxlength="5" />
        </div>
      </div>
    </div>
  `,
  styles: [`
    .address-input {
      display: flex;
      flex-direction: column;
      gap: 1rem;
      padding: 1rem;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
    .address-input.disabled {
      opacity: 0.5;
      pointer-events: none;
    }
    .field {
      display: flex;
      flex-direction: column;
      gap: 0.25rem;
    }
    .field-row {
      display: grid;
      grid-template-columns: 2fr 1fr;
      gap: 1rem;
    }
    label {
      font-weight: 500;
      font-size: 0.875rem;
    }
    input, select {
      padding: 0.5rem;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
  `],
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => AddressInputComponent),
    multi: true
  }]
})
export class AddressInputComponent implements ControlValueAccessor {
  address: Address = {
    street: '',
    city: '',
    state: '',
    zipCode: ''
  };

  states = ['CA', 'NY', 'TX', 'FL', 'IL', 'PA'];
  disabled: boolean = false;
  private touched: boolean = false;

  private onChange: (value: Address) => void = () => {};
  private onTouched: () => void = () => {};

  onAddressChange(): void {
    this.onChange(this.address);
  }

  markAsTouched(): void {
    if (!this.touched) {
      this.touched = true;
      this.onTouched();
    }
  }

  writeValue(value: Address): void {
    if (value) {
      this.address = { ...value };
    } else {
      this.address = {
        street: '',
        city: '',
        state: '',
        zipCode: ''
      };
    }
  }

  registerOnChange(fn: (value: Address) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}

// Usage
@Component({
  selector: 'app-checkout',
  template: `
    <form [formGroup]="form">
      <h3>Billing Address</h3>
      <app-address-input formControlName="billingAddress"></app-address-input>
      
      <h3>Shipping Address</h3>
      <label>
        <input type="checkbox" (change)="copybillingAddress($event)" />
        Same as billing address
      </label>
      <app-address-input formControlName="shippingAddress"></app-address-input>
    </form>
  `
})
export class CheckoutComponent {
  form = new FormGroup({
    billingAddress: new FormControl<Address | null>(null, Validators.required),
    shippingAddress: new FormControl<Address | null>(null, Validators.required)
  });

  copyBillingAddress(event: Event): void {
    if ((event.target as HTMLInputElement).checked) {
      const billing = this.form.value.billingAddress;
      this.form.controls.shippingAddress.setValue(billing);
    }
  }
}
```

## Common Patterns

Frequently used CVA implementations.

### File Upload Control

```typescript
@Component({
  selector: 'app-file-upload',
  template: `
    <div class="file-upload">
      <input type="file"
             #fileInput
             [accept]="accept"
             (change)="onFileSelected($event)"
             [disabled]="disabled"
             style="display: none;" />
      
      <button type="button" (click)="fileInput.click()" [disabled]="disabled">
        Choose File
      </button>
      
      <span *ngIf="fileName" class="file-name">
        {{ fileName }}
        <button type="button" (click)="clearFile()" [disabled]="disabled">×</button>
      </span>
    </div>
  `,
  providers: [/* NG_VALUE_ACCESSOR */]
})
export class FileUploadComponent implements ControlValueAccessor {
  @Input() accept: string = '*/*';
  
  fileName: string = '';
  disabled: boolean = false;

  private onChange: (value: File | null) => void = () => {};
  private onTouched: () => void = () => {};

  onFileSelected(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (file) {
      this.fileName = file.name;
      this.onChange(file);
    }
    this.onTouched();
  }

  clearFile(): void {
    this.fileName = '';
    this.onChange(null);
    this.onTouched();
  }

  writeValue(value: File | null): void {
    this.fileName = value?.name || '';
  }

  registerOnChange(fn: (value: File | null) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}
```

### Tag Input Control

```typescript
@Component({
  selector: 'app-tag-input',
  template: `
    <div class="tag-input" [class.disabled]="disabled">
      <div class="tags">
        <span *ngFor="let tag of tags; let i = index" class="tag">
          {{ tag }}
          <button type="button" (click)="removeTag(i)" [disabled]="disabled">×</button>
        </span>
      </div>
      <input type="text"
             [(ngModel)]="inputValue"
             (keydown.enter)="addTag()"
             (keydown.comma)="addTag()"
             (blur)="onBlur()"
             [disabled]="disabled"
             placeholder="Add tag..." />
    </div>
  `,
  providers: [/* NG_VALUE_ACCESSOR */]
})
export class TagInputComponent implements ControlValueAccessor {
  tags: string[] = [];
  inputValue: string = '';
  disabled: boolean = false;

  private onChange: (value: string[]) => void = () => {};
  private onTouched: () => void = () => {};

  addTag(): void {
    const tag = this.inputValue.trim().replace(/,$/g, '');
    if (tag && !this.tags.includes(tag) && !this.disabled) {
      this.tags = [...this.tags, tag];
      this.inputValue = '';
      this.onChange(this.tags);
    }
  }

  removeTag(index: number): void {
    if (!this.disabled) {
      this.tags = this.tags.filter((_, i) => i !== index);
      this.onChange(this.tags);
    }
  }

  onBlur(): void {
    this.onTouched();
  }

  writeValue(value: string[]): void {
    this.tags = value || [];
  }

  registerOnChange(fn: (value: string[]) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}
```

## Testing Custom Controls

Unit testing ControlValueAccessor implementations.

### Test Suite

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FormsModule, ReactiveFormsModule, FormControl } from '@angular/forms';
import { CounterComponent } from './counter.component';

describe('CounterComponent (ControlValueAccessor)', () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [CounterComponent],
      imports: [FormsModule, ReactiveFormsModule]
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should implement writeValue', () => {
    component.writeValue(5);
    expect(component.value).toBe(5);
  });

  it('should call onChange when value changes', () => {
    const onChangeSpy = jasmine.createSpy('onChange');
    component.registerOnChange(onChangeSpy);
    
    component.increment();
    
    expect(onChangeSpy).toHaveBeenCalledWith(1);
  });

  it('should call onTouched when interacted', () => {
    const onTouchedSpy = jasmine.createSpy('onTouched');
    component.registerOnTouched(onTouchedSpy);
    
    component.increment();
    
    expect(onTouchedSpy).toHaveBeenCalled();
  });

  it('should handle disabled state', () => {
    component.setDisabledState(true);
    expect(component.disabled).toBe(true);
    
    component.increment();
    expect(component.value).toBe(0); // Should not increment when disabled
  });

  it('should work with FormControl', () => {
    const control = new FormControl(10);
    const onChangeSpy = jasmine.createSpy('onChange');
    
    component.writeValue(control.value);
    component.registerOnChange(onChangeSpy);
    
    component.increment();
    
    expect(onChangeSpy).toHaveBeenCalledWith(11);
  });
});
```

## Real-World Examples

Complete examples of production-ready custom controls.

### Color Picker

```typescript
@Component({
  selector: 'app-color-picker',
  template: `
    <div class="color-picker">
      <div class="color-preview" 
           [style.background-color]="value"
           (click)="togglePicker()"
           [class.disabled]="disabled">
      </div>
      
      <div *ngIf="showPicker" class="color-palette">
        <div *ngFor="let color of colors" 
             class="color-swatch"
             [style.background-color]="color"
             [class.selected]="color === value"
             (click)="selectColor(color)">
        </div>
      </div>
      
      <input type="text"
             [value]="value"
             (input)="onInput($event)"
             (blur)="onBlur()"
             [disabled]="disabled"
             placeholder="#000000" />
    </div>
  `,
  providers: [/* NG_VALUE_ACCESSOR */]
})
export class ColorPickerComponent implements ControlValueAccessor {
  value: string = '#000000';
  showPicker: boolean = false;
  disabled: boolean = false;

  colors = [
    '#000000', '#FFFFFF', '#FF0000', '#00FF00', '#0000FF',
    '#FFFF00', '#FF00FF', '#00FFFF', '#FFA500', '#800080'
  ];

  private onChange: (value: string) => void = () => {};
  private onTouched: () => void = () => {};

  togglePicker(): void {
    if (!this.disabled) {
      this.showPicker = !this.showPicker;
    }
  }

  selectColor(color: string): void {
    if (!this.disabled) {
      this.value = color;
      this.onChange(color);
      this.showPicker = false;
      this.onTouched();
    }
  }

  onInput(event: Event): void {
    const color = (event.target as HTMLInputElement).value;
    this.value = color;
    this.onChange(color);
  }

  onBlur(): void {
    this.showPicker = false;
    this.onTouched();
  }

  writeValue(value: string): void {
    this.value = value || '#000000';
  }

  registerOnChange(fn: (value: string) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }
}
```

## Common Mistakes

### 1. Not Storing Callbacks

```typescript
// WRONG
registerOnChange(fn: (value: any) => void): void {
  // Callback not stored, changes won't be reported
}

// CORRECT
private onChange: (value: any) => void = () => {};

registerOnChange(fn: (value: any) => void): void {
  this.onChange = fn;
}
```

### 2. Calling onChange in writeValue

```typescript
// WRONG: Creates infinite loop
writeValue(value: any): void {
  this.value = value;
  this.onChange(value); // DON'T DO THIS
}

// CORRECT
writeValue(value: any): void {
  this.value = value;
  // Don't call onChange - this is the form writing to you
}
```

### 3. Forgetting multi: true

```typescript
// WRONG: Will replace other value accessors
providers: [{
  provide: NG_VALUE_ACCESSOR,
  useExisting: forwardRef(() => MyComponent)
  // Missing multi: true!
}]

// CORRECT
providers: [{
  provide: NG_VALUE_ACCESSOR,
  useExisting: forwardRef(() => MyComponent),
  multi: true // Essential!
}]
```

## Best Practices

1. **Store callbacks in private fields** - registerOnChange/registerOnTouched
2. **Don't call onChange in writeValue** - Causes infinite loops
3. **Always use forwardRef** - Needed for class reference in decorator
4. **Set multi: true in provider** - Required for NG_VALUE_ACCESSOR
5. **Handle null/undefined in writeValue** - Set sensible defaults
6. **Implement setDisabledState** - Proper disabled state handling
7. **Call onTouched appropriately** - Usually on blur or after interaction
8. **Test with FormControl** - Ensure proper integration
9. **Make controls accessible** - Support keyboard navigation
10. **Document expected value types** - Clear interface definitions

## Interview Questions

### Q1: What is ControlValueAccessor?

**Answer:** ControlValueAccessor is an interface that connects custom components with Angular's forms API, enabling them to work with formControlName, ngModel, and reactive forms. It defines four methods: writeValue, registerOnChange, registerOnTouched, and optionally setDisabledState.

### Q2: Why do you need forwardRef in the provider?

**Answer:** forwardRef is needed because the class is referenced in its own decorator before it's fully defined. Without it, you'd get a "cannot access before initialization" error due to JavaScript's execution order.

### Q3: What's the difference between writeValue and registerOnChange?

**Answer:** writeValue is called by the forms API to write a value TO your component. registerOnChange registers a callback that your component calls to notify the forms API of changes FROM your component.

### Q4: When should you call onTouched?

**Answer:** Call onTouched when the control should be marked as touched, typically on blur or after the user completes an interaction. This triggers validation error display in most validation strategies.

### Q5: Can you combine ControlValueAccessor with Validator?

**Answer:** Yes, implement both interfaces and provide both NG_VALUE_ACCESSOR and NG_VALIDATORS. This allows your custom control to have its own validation logic that integrates with the forms system.

## Key Takeaways

1. **CVA bridges custom components and forms** - Seamless integration with Angular forms
2. **Four methods define the contract** - writeValue, registerOnChange, registerOnTouched, setDisabledState
3. **Store callbacks for later use** - Don't call them in writeValue
4. **forwardRef is required** - For class self-reference in decorator
5. **multi: true is essential** - Allows multiple value accessors
6. **Call onChange when value changes** - Notify forms API of updates
7. **Call onTouched appropriately** - Mark control as touched
8. **Handle disabled state** - Implement setDisabledState
9. **Works with any form type** - Template-driven and reactive
10. **Compose complex controls** - Build reusable form components

## Resources

- [ControlValueAccessor API](https://angular.io/api/forms/ControlValueAccessor)
- [Custom Form Controls Guide](https://angular.io/guide/form-validation#creating-custom-validators)
- [NG_VALUE_ACCESSOR Token](https://angular.io/api/forms/NG_VALUE_ACCESSOR)
- [NG_VALIDATORS Token](https://angular.io/api/forms/NG_VALIDATORS)
- [forwardRef](https://angular.io/api/core/forwardRef)
