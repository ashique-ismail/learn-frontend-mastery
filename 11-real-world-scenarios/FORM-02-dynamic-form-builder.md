# Dynamic Form Builder (Data-Driven Fields)

## The Idea

**In plain English:** Instead of hand-coding each `<input>` in JSX or a template, you define what a form looks like in a JSON schema, and the component reads that schema to render itself. Add a field to the JSON and it appears in the UI — no code change required. The schema also drives validation rules and conditional visibility.

**Real-world analogy:** Think of a paper form factory.

- A **form designer** (backend team or CMS editor) fills in a spreadsheet: "Row 1: text field called 'Full Name', required. Row 2: dropdown called 'Country'. Row 3: text field called 'Province', only shown when Country = Canada."
- A **printer** (your front-end component) takes that spreadsheet and produces the actual form. The printer does not care what the fields are — it only knows how to print "text field", "dropdown", "checkbox".
- When the designer adds a new row to the spreadsheet, the printer produces a form with one more field automatically.
- A **condition checker** (your show/hide logic) reads the "only shown when..." column and hides or shows rows based on the user's current answers.

The key insight: the schema is the source of truth. The renderer is generic. The consumer only touches the schema.

---

## Learning Objectives

- Design a field schema that captures type, label, validation rules, and conditions
- Build a recursive renderer that handles grouped/nested fields
- Implement conditional show/hide without coupling fields to each other
- Use `react-hook-form` + Zod for schema-driven validation
- Mirror the pattern in Angular with `ReactiveFormsModule` and `FormArray`

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Render fields from a JSON array | ❌ | CSS cannot iterate over data |
| Conditionally show field based on another field's value | ❌ | Requires reading runtime form state |
| Generate validation rules from schema | ❌ | JS-only concern |
| Add/remove fields dynamically at runtime | ❌ | DOM mutation requires JS |
| Deeply nested repeating groups | ❌ | Recursive rendering requires JS |

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Each rendered field follows this pattern -->
<div class="form-field" data-field-type="text">
  <label class="form-label" for="field-fullName">
    Full Name <span class="form-required" aria-hidden="true">*</span>
  </label>
  <input
    id="field-fullName"
    name="fullName"
    type="text"
    class="form-input"
    aria-required="true"
    aria-describedby="field-fullName-error"
  />
  <p id="field-fullName-error" class="form-error" role="alert" hidden>
    Full name is required
  </p>
</div>

<!-- Conditional field wrapper — JS toggles hidden attribute -->
<div class="form-field" data-field-type="text" hidden data-condition='{"field":"country","value":"CA"}'>
  <label class="form-label" for="field-province">Province</label>
  <input id="field-province" name="province" class="form-input" />
</div>

<!-- Repeating group -->
<fieldset class="form-group">
  <legend class="form-group-legend">Work Experience</legend>
  <!-- repeated entries here -->
  <button type="button" class="form-add-btn">+ Add Entry</button>
</fieldset>
```

### The CSS

```css
:root {
  --form-accent: #2563eb;
  --form-error:  #dc2626;
  --form-border: #d1d5db;
  --form-focus:  0 0 0 3px rgba(37, 99, 235, 0.3);
}

.form-field {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  margin-bottom: 1.25rem;
}

.form-label {
  font-size: 0.875rem;
  font-weight: 500;
  color: #374151;
}

.form-required { color: var(--form-error); margin-left: 0.125rem; }

.form-input,
.form-select,
.form-textarea {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid var(--form-border);
  border-radius: 6px;
  font-size: 1rem;
  transition: border-color 0.15s, box-shadow 0.15s;
}

.form-input:focus,
.form-select:focus,
.form-textarea:focus {
  outline: none;
  border-color: var(--form-accent);
  box-shadow: var(--form-focus);
}

.form-input[aria-invalid="true"] {
  border-color: var(--form-error);
}

.form-error {
  font-size: 0.75rem;
  color: var(--form-error);
  min-height: 1rem;
}

/* Conditional field — hidden by default, revealed by JS */
.form-field[data-hidden="true"] {
  display: none;
}

/* Repeating group */
.form-group {
  border: 1px solid var(--form-border);
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1.25rem;
}

.form-group-legend {
  font-weight: 600;
  padding: 0 0.5rem;
}

.form-group-entry {
  padding: 0.75rem;
  background: #f9fafb;
  border-radius: 6px;
  margin-bottom: 0.75rem;
  position: relative;
}

.form-add-btn {
  background: none;
  border: 2px dashed var(--form-border);
  border-radius: 6px;
  padding: 0.5rem 1rem;
  cursor: pointer;
  color: var(--form-accent);
  width: 100%;
  font-size: 0.875rem;
}

.form-remove-btn {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  background: none;
  border: none;
  cursor: pointer;
  color: var(--form-error);
  font-size: 1.25rem;
  line-height: 1;
}
```

---

## React Implementation

### Field Schema Types

```tsx
// formSchema.types.ts
export type FieldType = 'text' | 'email' | 'select' | 'checkbox' | 'textarea' | 'group';

export interface FieldCondition {
  field: string;     // name of controlling field
  value: unknown;    // when that field equals this value, show the field
}

export interface FieldOption {
  label: string;
  value: string;
}

export interface FieldSchema {
  name:        string;
  type:        FieldType;
  label:       string;
  required?:   boolean;
  placeholder?: string;
  options?:    FieldOption[];   // for select
  condition?:  FieldCondition;  // show this field only when condition is met
  fields?:     FieldSchema[];   // for type === 'group' (recursive)
  min?:        number;          // min entries for group
  max?:        number;          // max entries for group
  validation?: {
    minLength?: number;
    maxLength?: number;
    pattern?:  string;
    message?:  string;
  };
}
```

### Building a Zod Schema from the Field Schema

```tsx
// buildZodSchema.ts
import { z } from 'zod';
import type { FieldSchema } from './formSchema.types';

export function buildZodSchema(fields: FieldSchema[]): z.ZodObject<any> {
  const shape: Record<string, z.ZodTypeAny> = {};

  for (const field of fields) {
    if (field.type === 'group') {
      const groupItem = buildZodSchema(field.fields ?? []);
      let arr = z.array(groupItem);
      if (field.min !== undefined) arr = arr.min(field.min);
      if (field.max !== undefined) arr = arr.max(field.max);
      shape[field.name] = arr.optional();
      continue;
    }

    let schema: z.ZodTypeAny = z.string();

    if (field.validation?.minLength) {
      schema = (schema as z.ZodString).min(
        field.validation.minLength,
        field.validation.message ?? `Minimum ${field.validation.minLength} characters`
      );
    }
    if (field.validation?.maxLength) {
      schema = (schema as z.ZodString).max(field.validation.maxLength);
    }
    if (field.validation?.pattern) {
      schema = (schema as z.ZodString).regex(
        new RegExp(field.validation.pattern),
        field.validation.message
      );
    }
    if (field.type === 'email') {
      schema = (schema as z.ZodString).email('Enter a valid email');
    }
    if (!field.required) {
      schema = schema.optional().or(z.literal(''));
    }

    shape[field.name] = schema;
  }

  return z.object(shape);
}
```

### The Recursive Field Renderer

```tsx
// DynamicField.tsx
import { useFormContext, useWatch, useFieldArray } from 'react-hook-form';
import type { FieldSchema } from './formSchema.types';

interface Props {
  field: FieldSchema;
  prefix?: string;  // for nested group fields: "experience.0."
}

export function DynamicField({ field, prefix = '' }: Props) {
  const { register, formState: { errors } } = useFormContext();
  const name = `${prefix}${field.name}`;

  // Evaluate condition
  const watchValue = useWatch({ name: field.condition?.field ?? '__none__' });
  const conditionMet = !field.condition || watchValue === field.condition.value;

  if (!conditionMet) return null;

  const error = name.split('.').reduce<any>((obj, key) => obj?.[key], errors);

  if (field.type === 'group') {
    return <GroupField field={field} prefix={prefix} />;
  }

  return (
    <div className="form-field">
      <label htmlFor={name} className="form-label">
        {field.label}
        {field.required && <span className="form-required" aria-hidden="true">*</span>}
      </label>

      {field.type === 'textarea' ? (
        <textarea
          id={name}
          className="form-textarea"
          placeholder={field.placeholder}
          aria-invalid={error ? 'true' : undefined}
          aria-describedby={error ? `${name}-error` : undefined}
          {...register(name)}
        />
      ) : field.type === 'select' ? (
        <select
          id={name}
          className="form-select"
          aria-invalid={error ? 'true' : undefined}
          {...register(name)}
        >
          <option value="">— Select —</option>
          {field.options?.map(opt => (
            <option key={opt.value} value={opt.value}>{opt.label}</option>
          ))}
        </select>
      ) : field.type === 'checkbox' ? (
        <input id={name} type="checkbox" {...register(name)} />
      ) : (
        <input
          id={name}
          type={field.type}
          className="form-input"
          placeholder={field.placeholder}
          aria-required={field.required}
          aria-invalid={error ? 'true' : undefined}
          aria-describedby={error ? `${name}-error` : undefined}
          {...register(name)}
        />
      )}

      {error && (
        <p id={`${name}-error`} className="form-error" role="alert">
          {error.message as string}
        </p>
      )}
    </div>
  );
}

// Repeating group
function GroupField({ field, prefix }: { field: FieldSchema; prefix: string }) {
  const name = `${prefix}${field.name}`;
  const { fields: entries, append, remove } = useFieldArray({ name });

  return (
    <fieldset className="form-group">
      <legend className="form-group-legend">{field.label}</legend>
      {entries.map((entry, idx) => (
        <div key={entry.id} className="form-group-entry">
          <button
            type="button"
            className="form-remove-btn"
            aria-label={`Remove entry ${idx + 1}`}
            onClick={() => remove(idx)}
          >
            ×
          </button>
          {field.fields?.map(subField => (
            <DynamicField
              key={subField.name}
              field={subField}
              prefix={`${name}.${idx}.`}
            />
          ))}
        </div>
      ))}
      {(!field.max || entries.length < field.max) && (
        <button type="button" className="form-add-btn" onClick={() => append({})}>
          + Add {field.label}
        </button>
      )}
    </fieldset>
  );
}
```

### The Form Shell

```tsx
// DynamicForm.tsx
import { FormProvider, useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { DynamicField } from './DynamicField';
import { buildZodSchema } from './buildZodSchema';
import type { FieldSchema } from './formSchema.types';

const EXAMPLE_SCHEMA: FieldSchema[] = [
  { name: 'fullName', type: 'text', label: 'Full Name', required: true },
  { name: 'email',    type: 'email', label: 'Email',   required: true },
  {
    name: 'country', type: 'select', label: 'Country', required: true,
    options: [{ label: 'United States', value: 'US' }, { label: 'Canada', value: 'CA' }],
  },
  {
    name: 'province', type: 'text', label: 'Province',
    condition: { field: 'country', value: 'CA' },
  },
  {
    name: 'experience', type: 'group', label: 'Work Experience', min: 1, max: 5,
    fields: [
      { name: 'company', type: 'text',  label: 'Company', required: true },
      { name: 'years',   type: 'text',  label: 'Years',   required: true,
        validation: { pattern: '^\\d+$', message: 'Numbers only' } },
    ],
  },
];

export function DynamicForm() {
  const zodSchema = buildZodSchema(EXAMPLE_SCHEMA);

  const methods = useForm({
    resolver: zodResolver(zodSchema),
    defaultValues: { experience: [{}] },
  });

  function onSubmit(data: Record<string, unknown>) {
    console.log('Submitted:', data);
  }

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)} noValidate>
        {EXAMPLE_SCHEMA.map(field => (
          <DynamicField key={field.name} field={field} />
        ))}
        <button type="submit">Submit</button>
      </form>
    </FormProvider>
  );
}
```

---

## Angular Implementation

### Service: Schema-to-FormGroup

```typescript
// form-builder.service.ts
import { Injectable } from '@angular/core';
import {
  FormBuilder, FormGroup, FormArray,
  Validators, AbstractControl, ValidatorFn
} from '@angular/forms';
import type { FieldSchema } from './formSchema.types';

@Injectable({ providedIn: 'root' })
export class DynamicFormBuilderService {
  constructor(private fb: FormBuilder) {}

  buildGroup(fields: FieldSchema[]): FormGroup {
    const controls: Record<string, AbstractControl> = {};

    for (const field of fields) {
      if (field.type === 'group') {
        controls[field.name] = this.fb.array(
          [this.buildGroup(field.fields ?? [])],
          field.min ? Validators.minLength(field.min) : null
        );
      } else {
        const validators: ValidatorFn[] = [];
        if (field.required)                    validators.push(Validators.required);
        if (field.type === 'email')             validators.push(Validators.email);
        if (field.validation?.minLength)        validators.push(Validators.minLength(field.validation.minLength));
        if (field.validation?.maxLength)        validators.push(Validators.maxLength(field.validation.maxLength));
        if (field.validation?.pattern)          validators.push(Validators.pattern(field.validation.pattern));

        controls[field.name] = this.fb.control('', validators);
      }
    }

    return this.fb.group(controls);
  }

  addGroupEntry(formArray: FormArray, fields: FieldSchema[]) {
    formArray.push(this.buildGroup(fields));
  }

  removeGroupEntry(formArray: FormArray, index: number) {
    formArray.removeAt(index);
  }
}
```

### Dynamic Field Component

```typescript
// dynamic-field.component.ts
import { Component, Input, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule, FormGroup, FormArray } from '@angular/forms';
import { DynamicFormBuilderService } from './form-builder.service';
import type { FieldSchema } from './formSchema.types';

@Component({
  selector: 'app-dynamic-field',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <ng-container [formGroup]="group">

      <!-- Conditional visibility -->
      @let condOk = !field.condition || group.get(field.condition.field)?.value === field.condition.value;

      @if (condOk) {
        <!-- Group (repeating) -->
        @if (field.type === 'group') {
          <fieldset class="form-group" [formArrayName]="field.name">
            <legend class="form-group-legend">{{ field.label }}</legend>
            @for (entry of getArray(field.name).controls; track $index) {
              <div class="form-group-entry" [formGroupName]="$index">
                <button type="button" class="form-remove-btn"
                  (click)="removeEntry(field, $index)"
                  [attr.aria-label]="'Remove entry ' + ($index + 1)">×</button>
                @for (sub of field.fields; track sub.name) {
                  <app-dynamic-field [field]="sub" [group]="getGroupAt(field.name, $index)" />
                }
              </div>
            }
            <button type="button" class="form-add-btn" (click)="addEntry(field)">
              + Add {{ field.label }}
            </button>
          </fieldset>
        }

        <!-- Leaf field -->
        @if (field.type !== 'group') {
          <div class="form-field">
            <label [for]="field.name" class="form-label">
              {{ field.label }}
              @if (field.required) { <span class="form-required" aria-hidden="true">*</span> }
            </label>

            @if (field.type === 'select') {
              <select [id]="field.name" [formControlName]="field.name" class="form-select">
                <option value="">— Select —</option>
                @for (opt of field.options; track opt.value) {
                  <option [value]="opt.value">{{ opt.label }}</option>
                }
              </select>
            } @else if (field.type === 'textarea') {
              <textarea [id]="field.name" [formControlName]="field.name"
                class="form-textarea" [placeholder]="field.placeholder ?? ''"></textarea>
            } @else {
              <input [id]="field.name" [type]="field.type" [formControlName]="field.name"
                class="form-input" [placeholder]="field.placeholder ?? ''"
                [attr.aria-invalid]="isInvalid(field.name) ? 'true' : null" />
            }

            @if (isInvalid(field.name)) {
              <p class="form-error" role="alert">{{ errorMessage(field) }}</p>
            }
          </div>
        }
      }
    </ng-container>
  `,
})
export class DynamicFieldComponent {
  @Input() field!: FieldSchema;
  @Input() group!: FormGroup;

  constructor(private builder: DynamicFormBuilderService) {}

  getArray(name: string) { return this.group.get(name) as FormArray; }
  getGroupAt(name: string, i: number) { return this.getArray(name).at(i) as FormGroup; }
  isInvalid(name: string) {
    const ctrl = this.group.get(name);
    return ctrl?.invalid && ctrl?.touched;
  }

  errorMessage(field: FieldSchema): string {
    const ctrl = this.group.get(field.name);
    if (!ctrl?.errors) return '';
    if (ctrl.errors['required'])  return `${field.label} is required`;
    if (ctrl.errors['email'])     return 'Enter a valid email';
    if (ctrl.errors['minlength']) return field.validation?.message ?? 'Too short';
    if (ctrl.errors['pattern'])   return field.validation?.message ?? 'Invalid format';
    return 'Invalid value';
  }

  addEntry(field: FieldSchema) {
    this.builder.addGroupEntry(this.getArray(field.name), field.fields ?? []);
  }

  removeEntry(field: FieldSchema, index: number) {
    this.builder.removeGroupEntry(this.getArray(field.name), index);
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Every input has a `<label>` | `for`/`id` pairing — never `placeholder` as sole label |
| Required fields marked visually and semantically | `aria-required="true"` and asterisk with `aria-hidden` |
| Error messages linked to inputs | `aria-describedby="${name}-error"` |
| `aria-invalid="true"` on failed fields | Screen readers announce "invalid entry" |
| Errors use `role="alert"` | Announced immediately without waiting for focus |
| Conditional fields removed from DOM when hidden | `return null` / `@if` — not just CSS hidden |
| Repeating group entries are focusable in order | DOM order matches visual order; no `tabindex` hacks |
| Remove buttons have descriptive `aria-label` | "Remove entry 2" not just "×" |
| Fieldset + legend for groups | Groups announced as a whole to screen readers |
| Focus moves to new entry on "Add" | `setTimeout(() => newEntry.querySelector('input')?.focus(), 0)` |

---

## Production Pitfalls

**1. Conditional fields keep stale validation errors**
When a field is hidden, its validation rules still run if it stays in the form model. Fix: when a condition turns false, unregister the field from the form (React Hook Form's `shouldUnregister: true` or Angular's `removeControl`). Re-register it with blank state when the condition becomes true again.

**2. `useWatch` causes excessive re-renders**
Watching every controlling field in the parent causes all children to re-render on each keystroke. Fix: watch only at the level where the condition is evaluated — inside each `DynamicField` component, not at the form root.

**3. Recursive rendering has no depth guard**
A schema where a group contains itself would cause infinite recursion. Fix: accept a `maxDepth` prop and short-circuit rendering if exceeded. Log a warning with the offending schema node name.

**4. Field `name` collisions in nested groups**
Using bare `field.name` inside a repeating group without a prefix creates multiple inputs sharing the same `id`. Fix: always compose the full path: `experience.0.company`. Use the `prefix` prop pattern from the React implementation above.

**5. `FormArray` `minLength` validator never fires in Angular**
Angular's `Validators.minLength` only works on string-valued controls, not arrays. For arrays, write a custom validator: `(arr: AbstractControl) => arr.value.length < min ? { minEntries: true } : null`.

**6. Schema updates at runtime break existing form state**
If the schema changes (e.g., a field is removed) while the user has data entered, the form state has keys that no longer exist in the schema. Fix: when the schema changes, merge existing values only for keys that still exist in the new schema.

---

## Interview Angle

**Q: "How would you build a form that renders itself from a JSON config rather than hardcoded JSX?"**

Strong answer covers:
1. **Schema design first** — field type, name, label, validation rules, and optional condition. The schema is a typed interface, not an arbitrary blob.
2. **Recursive renderer** — a single `DynamicField` component that handles leaf fields and delegates to itself for group fields. The recursion terminates at leaf nodes.
3. **Conditional logic lives in the renderer, not the schema** — the schema declares a condition; the renderer evaluates it against current form state using `useWatch` (React) or `valueChanges` subscription (Angular).
4. **Validation is derived from the schema** — `buildZodSchema` or Angular's validator builder walks the same schema array, producing a validation schema that mirrors the field schema exactly. One source of truth.

**Follow-up: "How do you handle deeply nested groups without prop drilling?"**
Use React Context (`FormProvider` from react-hook-form) or Angular's `ReactiveFormsModule` injection to access the root form from any depth. The `prefix` or `formGroupName` path system keeps each field pointing to the correct location in the model tree.
