# ngx-formly

## What It Is

ngx-formly is a dynamic form generation library for Angular. Instead of writing form templates manually, you define forms as JSON configuration and Formly renders them.

Ideal for: admin panels, configurable forms, form builders, forms generated from a backend API.

---

## Setup

```bash
npm install @ngx-formly/core @ngx-formly/bootstrap  # or material, primeng, etc.
```

```ts
// app.config.ts
import { FormlyModule } from '@ngx-formly/core';
import { FormlyBootstrapModule } from '@ngx-formly/bootstrap';

export const appConfig: ApplicationConfig = {
  providers: [
    importProvidersFrom(
      ReactiveFormsModule,
      FormlyModule.forRoot({
        validationMessages: [
          { name: 'required', message: 'This field is required' },
          { name: 'email', message: 'Invalid email address' },
        ],
      }),
      FormlyBootstrapModule,
    ),
  ],
};
```

---

## Basic Usage

```ts
@Component({
  imports: [ReactiveFormsModule, FormlyModule, FormlyBootstrapModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <formly-form
        [model]="model"
        [fields]="fields"
        [options]="options"
        [form]="form"
      />
      <button type="submit" [disabled]="form.invalid">Submit</button>
    </form>
  `,
})
export class UserFormComponent {
  form = new FormGroup({});
  model: Partial<User> = { role: 'user' };
  options: FormlyFormOptions = {};

  fields: FormlyFieldConfig[] = [
    {
      key: 'name',
      type: 'input',
      props: {
        label: 'Name',
        placeholder: 'Full name',
        required: true,
      },
    },
    {
      key: 'email',
      type: 'input',
      props: {
        label: 'Email',
        type: 'email',
        required: true,
      },
    },
    {
      key: 'role',
      type: 'select',
      props: {
        label: 'Role',
        options: [
          { value: 'admin', label: 'Administrator' },
          { value: 'user', label: 'User' },
          { value: 'viewer', label: 'Viewer' },
        ],
      },
    },
  ];

  onSubmit() {
    if (this.form.valid) {
      console.log(this.model);  // model is automatically updated
    }
  }
}
```

---

## Built-in Field Types

```ts
// Standard types (vary by UI library adapter)
type: 'input'     // text, email, number, password
type: 'textarea'
type: 'select'    // dropdown
type: 'checkbox'
type: 'radio'
type: 'multicheckbox'
```

---

## Validation

```ts
{
  key: 'age',
  type: 'input',
  props: {
    label: 'Age',
    type: 'number',
    required: true,
  },
  validators: {
    validation: [Validators.min(18), Validators.max(120)],
    ageRange: {
      expression: (c) => c.value >= 18 && c.value <= 120,
      message: (error, field) => `Age must be between 18 and 120`,
    },
  },
  asyncValidators: {
    uniqueAge: {
      expression: (c, field) =>
        this.userService.checkAge(c.value).pipe(
          map(result => !result.taken),
        ),
      message: 'This age is already taken (ridiculous example)',
    },
  },
},
```

---

## Field Expressions — Dynamic Show/Hide

```ts
fields: FormlyFieldConfig[] = [
  {
    key: 'hasAddress',
    type: 'checkbox',
    props: { label: 'Add address?' },
  },
  {
    key: 'address',
    type: 'input',
    props: { label: 'Address' },
    expressions: {
      // Show only when hasAddress is true
      hide: '!model.hasAddress',
      // Disable based on another field
      'props.disabled': 'model.role !== "admin"',
      // Dynamic label
      'props.label': `'Street Address' + (model.country === 'US' ? ' (US format)' : '')`,
    },
  },
];
```

---

## Custom Field Type

```ts
@Component({
  selector: 'formly-field-rating',
  template: `
    <div class="star-rating">
      @for (star of stars; track star) {
        <button
          type="button"
          (click)="formControl.setValue(star)"
          [class.active]="formControl.value >= star"
        >★</button>
      }
    </div>
  `,
})
export class RatingFieldComponent extends FieldType {
  stars = [1, 2, 3, 4, 5];
}

// Register in FormlyModule
FormlyModule.forRoot({
  types: [
    { name: 'rating', component: RatingFieldComponent },
  ],
})

// Use it
{ key: 'satisfaction', type: 'rating', props: { label: 'Satisfaction' } }
```

---

## Wrappers (Reusable Field Templates)

```ts
@Component({
  selector: 'formly-wrapper-panel',
  template: `
    <div class="panel" [ngClass]="props['panelClass']">
      <div class="panel-header">{{ props['panelTitle'] }}</div>
      <ng-container #fieldComponent></ng-container>
    </div>
  `,
})
export class PanelWrapperComponent extends FieldWrapper {}

// Usage: wrap any field in a panel
{
  key: 'settings',
  wrappers: ['panel'],
  props: { panelTitle: 'Advanced Settings', panelClass: 'info-panel' },
  fieldGroup: [
    { key: 'timeout', type: 'input', props: { label: 'Timeout (ms)' } },
    { key: 'retries', type: 'input', props: { label: 'Max retries' } },
  ],
}
```

---

## Field Groups and Repeating Sections

```ts
// Nested object
{
  key: 'address',
  fieldGroup: [
    { key: 'street', type: 'input', props: { label: 'Street' } },
    { key: 'city', type: 'input', props: { label: 'City' } },
    { key: 'zip', type: 'input', props: { label: 'ZIP Code' } },
  ],
}

// Repeating section
{
  key: 'contacts',
  type: 'repeat',  // custom repeat type (from @ngx-formly/core/json-schema or custom)
  props: { addText: 'Add Contact' },
  fieldArray: {
    fieldGroup: [
      { key: 'name', type: 'input', props: { label: 'Contact Name' } },
      { key: 'phone', type: 'input', props: { label: 'Phone' } },
    ],
  },
}
```

---

## Common Interview Questions

**Q: When would you choose ngx-formly over Angular Reactive Forms?**
When forms are data-driven (defined in a database or API), need to be configurable without code changes, or are highly repetitive with many similar fields. For complex custom business logic, hand-rolled reactive forms give more control.

**Q: How do you handle FormArray with formly?**
Use `type: 'repeat'` (a custom type you build or use from a formly extension) with `fieldArray` to define the template for each item. Formly manages the FormArray binding internally.

**Q: Can ngx-formly work with any Angular form UI library?**
Yes — it has official adapters for Bootstrap (`@ngx-formly/bootstrap`), Material (`@ngx-formly/material`), PrimeNG, Ionic, and others. Custom types let you use any component.
