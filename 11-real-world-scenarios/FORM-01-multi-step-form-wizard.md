# Multi-Step Form Wizard with Validation

## The Idea

**In plain English:** You have a long form — say, a checkout flow or an onboarding survey — that is too complex to show all at once. You break it into named steps. Users move forward and backward. Each step validates before allowing progression. Partial data is preserved if the user navigates back. The current step is optionally reflected in the URL so it survives a page refresh.

**Real-world analogy:** Think of a passport application at a government office.

- The clerk hands you **Form A** first (personal details). You fill it in and hand it back. The clerk checks it before handing you **Form B** (travel history). You cannot get Form B until Form A is accepted.
- If you need to step away and come back, the clerk keeps what you already submitted in a folder — you pick up where you left off, not from the beginning.
- The **step indicator at the top of the counter** shows "Step 2 of 4" — you always know how far along you are.
- The **"Previous" button** lets you go back to Form A to fix a typo, but it does not erase what you already wrote on Form B.
- The **URL in your browser** is like the ticket number the clerk gave you: it identifies where in the process you are, so you can bookmark or share the link.

The key insight: the wizard is a **state machine**. Each step is a node. Transitions (next/back) are edges. Validation is a guard on the forward edge.

---

## Learning Objectives

- Model multi-step state with `useReducer` and understand why it beats multiple `useState` calls
- Implement per-step validation with schema libraries (Zod)
- Persist partial form state to `sessionStorage` so progress survives a refresh
- Sync current step to the URL (`?step=2`) without a routing library dependency
- Build the same pattern in Angular using signals and a wizard service

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide step panels | ✅ with `:target` selector | — |
| Validate fields before advancing | ❌ | Requires JS constraint checking |
| Track which steps are complete | ❌ | CSS has no cross-element boolean state |
| Persist data across steps | ❌ | sessionStorage requires JS |
| Sync step to URL | ❌ | History API requires JS |
| Animate step transitions | ⚠️ | CSS can animate, JS must trigger class |
| Show inline field errors | ❌ | Dynamic error messages require JS |

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Step indicator bar -->
<nav aria-label="Form progress" class="wizard-steps">
  <ol class="wizard-step-list">
    <li class="wizard-step is-complete" aria-current="false">
      <span class="wizard-step-number" aria-hidden="true">1</span>
      <span class="wizard-step-label">Personal Info</span>
    </li>
    <li class="wizard-step is-active" aria-current="step">
      <span class="wizard-step-number" aria-hidden="true">2</span>
      <span class="wizard-step-label">Address</span>
    </li>
    <li class="wizard-step" aria-current="false">
      <span class="wizard-step-number" aria-hidden="true">3</span>
      <span class="wizard-step-label">Review</span>
    </li>
  </ol>
</nav>

<!-- Step panels — only active one is visible -->
<div class="wizard-panel" data-step="1" hidden>
  <fieldset>
    <legend>Personal Information</legend>
    <!-- fields -->
  </fieldset>
</div>

<div class="wizard-panel" data-step="2">
  <fieldset>
    <legend>Address</legend>
    <!-- fields -->
  </fieldset>
</div>

<!-- Navigation controls -->
<div class="wizard-nav">
  <button class="wizard-btn wizard-btn--back" type="button">Previous</button>
  <button class="wizard-btn wizard-btn--next" type="button">Next</button>
  <button class="wizard-btn wizard-btn--submit" type="submit" hidden>Submit</button>
</div>
```

### The CSS

```css
:root {
  --wizard-accent: #0066cc;
  --wizard-muted: #6b7280;
  --wizard-complete: #16a34a;
  --wizard-error: #dc2626;
  --wizard-transition: 0.25s ease;
}

/* ─── Step indicator ─── */
.wizard-step-list {
  display: flex;
  align-items: center;
  list-style: none;
  margin: 0;
  padding: 0;
  gap: 0;
  counter-reset: steps;
}

.wizard-step {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  position: relative;
  color: var(--wizard-muted);
  font-size: 0.75rem;
}

/* Connector line between steps */
.wizard-step:not(:last-child)::after {
  content: '';
  position: absolute;
  top: 16px;
  left: calc(50% + 16px);
  right: calc(-50% + 16px);
  height: 2px;
  background: #e5e7eb;
  transition: background var(--wizard-transition);
}

.wizard-step.is-complete::after {
  background: var(--wizard-complete);
}

.wizard-step-number {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 2px solid currentColor;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  margin-bottom: 0.25rem;
  transition: background var(--wizard-transition), color var(--wizard-transition);
}

.wizard-step.is-active {
  color: var(--wizard-accent);
}

.wizard-step.is-complete {
  color: var(--wizard-complete);
}

.wizard-step.is-complete .wizard-step-number {
  background: var(--wizard-complete);
  color: #fff;
  border-color: var(--wizard-complete);
}

/* ─── Panel transitions ─── */
.wizard-panel {
  animation: slideIn var(--wizard-transition) forwards;
}

@keyframes slideIn {
  from { opacity: 0; transform: translateX(20px); }
  to   { opacity: 1; transform: translateX(0); }
}

/* ─── Field errors ─── */
.field-error {
  color: var(--wizard-error);
  font-size: 0.75rem;
  margin-top: 0.25rem;
}

.wizard-input[aria-invalid="true"] {
  border-color: var(--wizard-error);
  outline-color: var(--wizard-error);
}

/* ─── Navigation buttons ─── */
.wizard-nav {
  display: flex;
  gap: 0.75rem;
  justify-content: space-between;
  margin-top: 1.5rem;
}

.wizard-btn {
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
  border: 2px solid transparent;
  transition: background var(--wizard-transition);
}

.wizard-btn--next,
.wizard-btn--submit {
  background: var(--wizard-accent);
  color: #fff;
  margin-left: auto;
}

.wizard-btn--back {
  background: transparent;
  border-color: var(--wizard-muted);
  color: var(--wizard-muted);
}
```

---

## React Implementation

### State Shape with `useReducer`

```tsx
// wizard.types.ts
import { z } from 'zod';

export const step1Schema = z.object({
  firstName: z.string().min(1, 'First name is required'),
  lastName:  z.string().min(1, 'Last name is required'),
  email:     z.string().email('Enter a valid email'),
});

export const step2Schema = z.object({
  street: z.string().min(1, 'Street is required'),
  city:   z.string().min(1, 'City is required'),
  zip:    z.string().regex(/^\d{5}$/, 'Enter a 5-digit ZIP'),
});

export const step3Schema = z.object({
  agreed: z.literal(true, { errorMap: () => ({ message: 'You must agree' }) }),
});

export type Step1Data = z.infer<typeof step1Schema>;
export type Step2Data = z.infer<typeof step2Schema>;
export type Step3Data = z.infer<typeof step3Schema>;

export interface WizardData {
  step1: Partial<Step1Data>;
  step2: Partial<Step2Data>;
  step3: Partial<Step3Data>;
}

export interface WizardState {
  currentStep: 1 | 2 | 3;
  data: WizardData;
  errors: Record<string, string>;
  completedSteps: Set<1 | 2 | 3>;
}

export type WizardAction =
  | { type: 'NEXT'; payload: Partial<WizardData['step1'] & WizardData['step2'] & WizardData['step3']> }
  | { type: 'BACK' }
  | { type: 'SET_ERRORS'; errors: Record<string, string> }
  | { type: 'RESTORE'; state: WizardState };
```

### The Reducer

```tsx
// wizardReducer.ts
import type { WizardState, WizardAction } from './wizard.types';

export function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case 'NEXT': {
      const step = state.currentStep;
      const key  = `step${step}` as keyof WizardState['data'];
      const nextStep = Math.min(3, step + 1) as 1 | 2 | 3;
      return {
        ...state,
        currentStep: nextStep,
        errors: {},
        data: { ...state.data, [key]: { ...state.data[key], ...action.payload } },
        completedSteps: new Set([...state.completedSteps, step]),
      };
    }
    case 'BACK':
      return {
        ...state,
        currentStep: Math.max(1, state.currentStep - 1) as 1 | 2 | 3,
        errors: {},
      };
    case 'SET_ERRORS':
      return { ...state, errors: action.errors };
    case 'RESTORE':
      return action.state;
    default:
      return state;
  }
}

export const initialState: WizardState = {
  currentStep: 1,
  data: { step1: {}, step2: {}, step3: {} },
  errors: {},
  completedSteps: new Set(),
};
```

### The Wizard Hook

```tsx
// useWizard.ts
import { useReducer, useEffect, useCallback } from 'react';
import { wizardReducer, initialState } from './wizardReducer';
import { step1Schema, step2Schema, step3Schema } from './wizard.types';
import type { WizardState } from './wizard.types';

const STORAGE_KEY = 'wizard-state';
const SCHEMAS     = [step1Schema, step2Schema, step3Schema] as const;

function serializeState(state: WizardState) {
  return JSON.stringify({ ...state, completedSteps: [...state.completedSteps] });
}

function deserializeState(raw: string): WizardState {
  const parsed = JSON.parse(raw);
  return { ...parsed, completedSteps: new Set(parsed.completedSteps) };
}

export function useWizard() {
  const [state, dispatch] = useReducer(wizardReducer, initialState, (init) => {
    try {
      const saved = sessionStorage.getItem(STORAGE_KEY);
      return saved ? deserializeState(saved) : init;
    } catch {
      return init;
    }
  });

  // Sync to URL
  useEffect(() => {
    const url = new URL(window.location.href);
    url.searchParams.set('step', String(state.currentStep));
    window.history.replaceState(null, '', url.toString());
  }, [state.currentStep]);

  // Persist to sessionStorage
  useEffect(() => {
    try {
      sessionStorage.setItem(STORAGE_KEY, serializeState(state));
    } catch { /* quota exceeded — ignore */ }
  }, [state]);

  const advance = useCallback(async (stepData: Record<string, unknown>) => {
    const schema = SCHEMAS[state.currentStep - 1];
    const result = schema.safeParse(stepData);

    if (!result.success) {
      const errors: Record<string, string> = {};
      for (const issue of result.error.issues) {
        errors[issue.path[0] as string] = issue.message;
      }
      dispatch({ type: 'SET_ERRORS', errors });
      return false;
    }

    dispatch({ type: 'NEXT', payload: stepData });
    return true;
  }, [state.currentStep]);

  const goBack = useCallback(() => dispatch({ type: 'BACK' }), []);

  const reset = useCallback(() => {
    sessionStorage.removeItem(STORAGE_KEY);
    dispatch({ type: 'RESTORE', state: initialState });
  }, []);

  return { state, advance, goBack, reset };
}
```

### Step Components and Wizard Shell

```tsx
// WizardForm.tsx
import React, { useRef } from 'react';
import { useWizard } from './useWizard';

export function WizardForm() {
  const { state, advance, goBack, reset } = useWizard();
  const { currentStep, errors, completedSteps } = state;

  const formRef = useRef<HTMLFormElement>(null);

  async function handleNext(e: React.FormEvent) {
    e.preventDefault();
    if (!formRef.current) return;

    const fd   = new FormData(formRef.current);
    const data = Object.fromEntries(fd.entries());

    const success = await advance(data);
    if (success && currentStep === 3) {
      await submitToApi({ ...state.data.step1, ...state.data.step2, ...data });
      reset();
    }
  }

  const steps = ['Personal Info', 'Address', 'Review'] as const;

  return (
    <div className="wizard">
      {/* Step indicator */}
      <nav aria-label="Form progress" className="wizard-steps">
        <ol className="wizard-step-list">
          {steps.map((label, i) => {
            const num = (i + 1) as 1 | 2 | 3;
            return (
              <li
                key={num}
                className={[
                  'wizard-step',
                  currentStep === num ? 'is-active' : '',
                  completedSteps.has(num) ? 'is-complete' : '',
                ].join(' ')}
                aria-current={currentStep === num ? 'step' : undefined}
              >
                <span className="wizard-step-number" aria-hidden="true">
                  {completedSteps.has(num) ? '✓' : num}
                </span>
                <span className="wizard-step-label">{label}</span>
              </li>
            );
          })}
        </ol>
      </nav>

      {/* The active step panel */}
      <form ref={formRef} onSubmit={handleNext} noValidate>
        {currentStep === 1 && <Step1 data={state.data.step1} errors={errors} />}
        {currentStep === 2 && <Step2 data={state.data.step2} errors={errors} />}
        {currentStep === 3 && <Step3 data={state.data.step3} errors={errors} allData={state.data} />}

        <div className="wizard-nav">
          {currentStep > 1 && (
            <button type="button" className="wizard-btn wizard-btn--back" onClick={goBack}>
              Previous
            </button>
          )}
          {currentStep < 3 && (
            <button type="submit" className="wizard-btn wizard-btn--next">Next</button>
          )}
          {currentStep === 3 && (
            <button type="submit" className="wizard-btn wizard-btn--submit">Submit</button>
          )}
        </div>
      </form>
    </div>
  );
}

/* ── Field helper ── */
function Field({
  name, label, type = 'text', defaultValue, error,
}: {
  name: string; label: string; type?: string;
  defaultValue?: string; error?: string;
}) {
  const id = `field-${name}`;
  return (
    <div className="field">
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        name={name}
        type={type}
        className="wizard-input"
        defaultValue={defaultValue}
        aria-invalid={error ? 'true' : undefined}
        aria-describedby={error ? `${id}-error` : undefined}
      />
      {error && <p id={`${id}-error`} className="field-error" role="alert">{error}</p>}
    </div>
  );
}

function Step1({ data, errors }: { data: Partial<any>; errors: Record<string, string> }) {
  return (
    <fieldset>
      <legend>Personal Information</legend>
      <Field name="firstName" label="First Name" defaultValue={data.firstName} error={errors.firstName} />
      <Field name="lastName"  label="Last Name"  defaultValue={data.lastName}  error={errors.lastName}  />
      <Field name="email"     label="Email"      type="email" defaultValue={data.email} error={errors.email} />
    </fieldset>
  );
}

function Step2({ data, errors }: { data: Partial<any>; errors: Record<string, string> }) {
  return (
    <fieldset>
      <legend>Address</legend>
      <Field name="street" label="Street" defaultValue={data.street} error={errors.street} />
      <Field name="city"   label="City"   defaultValue={data.city}   error={errors.city}   />
      <Field name="zip"    label="ZIP"    defaultValue={data.zip}    error={errors.zip}    />
    </fieldset>
  );
}

function Step3({ allData, errors }: { data: Partial<any>; errors: Record<string, string>; allData: any }) {
  return (
    <fieldset>
      <legend>Review your details</legend>
      <dl>
        <dt>Name</dt><dd>{allData.step1.firstName} {allData.step1.lastName}</dd>
        <dt>Email</dt><dd>{allData.step1.email}</dd>
        <dt>Address</dt><dd>{allData.step2.street}, {allData.step2.city} {allData.step2.zip}</dd>
      </dl>
      <label>
        <input type="checkbox" name="agreed" value="true" aria-invalid={errors.agreed ? 'true' : undefined} />
        I confirm the above details are correct
      </label>
      {errors.agreed && <p className="field-error" role="alert">{errors.agreed}</p>}
    </fieldset>
  );
}

async function submitToApi(data: Record<string, unknown>) {
  await fetch('/api/submit', { method: 'POST', body: JSON.stringify(data),
    headers: { 'Content-Type': 'application/json' } });
}
```

---

## Angular Implementation

### Wizard Service with Signals

```typescript
// wizard.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { z } from 'zod';

const step1Schema = z.object({
  firstName: z.string().min(1),
  lastName:  z.string().min(1),
  email:     z.string().email(),
});
const step2Schema = z.object({
  street: z.string().min(1),
  city:   z.string().min(1),
  zip:    z.string().regex(/^\d{5}$/),
});
const SCHEMAS = [step1Schema, step2Schema];

export interface WizardData {
  step1: Record<string, string>;
  step2: Record<string, string>;
}

@Injectable({ providedIn: 'root' })
export class WizardService {
  readonly currentStep   = signal<1 | 2 | 3>(1);
  readonly data          = signal<WizardData>({ step1: {}, step2: {} });
  readonly errors        = signal<Record<string, string>>({});
  readonly completedSteps = signal<Set<number>>(new Set());
  readonly isLastStep    = computed(() => this.currentStep() === 3);

  advance(stepData: Record<string, string>): boolean {
    const step   = this.currentStep();
    const schema = SCHEMAS[step - 1];

    if (schema) {
      const result = schema.safeParse(stepData);
      if (!result.success) {
        const errs: Record<string, string> = {};
        for (const iss of result.error.issues) errs[iss.path[0] as string] = iss.message;
        this.errors.set(errs);
        return false;
      }
    }

    this.errors.set({});
    const key = `step${step}` as keyof WizardData;
    this.data.update(d => ({ ...d, [key]: stepData }));
    this.completedSteps.update(s => new Set([...s, step]));
    if (step < 3) this.currentStep.update(s => (s + 1) as 1 | 2 | 3);
    this.syncUrl();
    return true;
  }

  goBack() {
    this.errors.set({});
    this.currentStep.update(s => Math.max(1, s - 1) as 1 | 2 | 3);
    this.syncUrl();
  }

  private syncUrl() {
    const url = new URL(location.href);
    url.searchParams.set('step', String(this.currentStep()));
    history.replaceState(null, '', url.toString());
  }
}
```

### Wizard Component

```typescript
// wizard.component.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { WizardService } from './wizard.service';

@Component({
  selector: 'app-wizard',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <!-- Step indicator -->
    <nav aria-label="Form progress" class="wizard-steps">
      <ol class="wizard-step-list">
        @for (step of steps; track step.num) {
          <li
            class="wizard-step"
            [class.is-active]="wizard.currentStep() === step.num"
            [class.is-complete]="wizard.completedSteps().has(step.num)"
            [attr.aria-current]="wizard.currentStep() === step.num ? 'step' : null"
          >
            <span class="wizard-step-number" aria-hidden="true">
              {{ wizard.completedSteps().has(step.num) ? '✓' : step.num }}
            </span>
            <span class="wizard-step-label">{{ step.label }}</span>
          </li>
        }
      </ol>
    </nav>

    <!-- Step 1 -->
    @if (wizard.currentStep() === 1) {
      <fieldset>
        <legend>Personal Information</legend>
        <div class="field">
          <label for="firstName">First Name</label>
          <input id="firstName" name="firstName" [(ngModel)]="formData['firstName']"
            [attr.aria-invalid]="wizard.errors()['firstName'] ? 'true' : null" />
          @if (wizard.errors()['firstName']) {
            <p class="field-error" role="alert">{{ wizard.errors()['firstName'] }}</p>
          }
        </div>
        <div class="field">
          <label for="email">Email</label>
          <input id="email" name="email" type="email" [(ngModel)]="formData['email']"
            [attr.aria-invalid]="wizard.errors()['email'] ? 'true' : null" />
          @if (wizard.errors()['email']) {
            <p class="field-error" role="alert">{{ wizard.errors()['email'] }}</p>
          }
        </div>
      </fieldset>
    }

    <!-- Step 2 -->
    @if (wizard.currentStep() === 2) {
      <fieldset>
        <legend>Address</legend>
        <div class="field">
          <label for="street">Street</label>
          <input id="street" name="street" [(ngModel)]="formData['street']"
            [attr.aria-invalid]="wizard.errors()['street'] ? 'true' : null" />
        </div>
        <div class="field">
          <label for="zip">ZIP</label>
          <input id="zip" name="zip" [(ngModel)]="formData['zip']"
            [attr.aria-invalid]="wizard.errors()['zip'] ? 'true' : null" />
        </div>
      </fieldset>
    }

    <!-- Step 3: review -->
    @if (wizard.currentStep() === 3) {
      <fieldset>
        <legend>Review</legend>
        <dl>
          <dt>Name</dt><dd>{{ wizard.data().step1['firstName'] }} {{ wizard.data().step1['lastName'] }}</dd>
          <dt>Email</dt><dd>{{ wizard.data().step1['email'] }}</dd>
          <dt>Street</dt><dd>{{ wizard.data().step2['street'] }}, {{ wizard.data().step2['zip'] }}</dd>
        </dl>
      </fieldset>
    }

    <!-- Navigation -->
    <div class="wizard-nav">
      @if (wizard.currentStep() > 1) {
        <button type="button" class="wizard-btn wizard-btn--back" (click)="wizard.goBack()">Previous</button>
      }
      @if (!wizard.isLastStep()) {
        <button type="button" class="wizard-btn wizard-btn--next" (click)="next()">Next</button>
      } @else {
        <button type="button" class="wizard-btn wizard-btn--submit" (click)="submit()">Submit</button>
      }
    </div>
  `,
})
export class WizardComponent {
  wizard   = inject(WizardService);
  formData: Record<string, string> = {};

  steps = [
    { num: 1, label: 'Personal Info' },
    { num: 2, label: 'Address' },
    { num: 3, label: 'Review' },
  ] as const;

  next()   { this.wizard.advance(this.formData); }
  submit() { if (this.wizard.advance(this.formData)) console.log('Submit', this.wizard.data()); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Step indicator uses `aria-current="step"` | Set on the active `<li>` |
| Each fieldset has a `<legend>` | Screen readers announce step context |
| Error messages use `role="alert"` | Announced immediately on validation failure |
| Inputs link to errors via `aria-describedby` | `id="${name}-error"` pattern |
| `aria-invalid="true"` on invalid inputs | Triggers error styling and screen reader cue |
| "Previous" button is `type="button"` | Prevents accidental form submission |
| Progress communicated via `aria-label` on nav | Screen readers understand step indicator |
| Focus moves to first error on failed validation | `querySelectorAll('[aria-invalid]')[0].focus()` |
| Keyboard-only users can complete all steps | No mouse-only interactions |
| `noValidate` on form suppresses browser UI | Custom errors are consistent across browsers |

---

## Production Pitfalls

**1. Back button clears field data**
If you use controlled inputs (`value=`) with React state, going back destroys the second step's data because the component unmounts. Fix: store all step data centrally (in the reducer or service) and populate `defaultValue` (uncontrolled) or restore controlled state from the store on mount.

**2. Validation on every keystroke creates noise**
Running Zod on `onChange` shows errors before the user finishes typing. Fix: validate `onBlur` per field, but always re-validate the full step on "Next" click. Track a `touched` set so errors only show for fields the user has interacted with.

**3. URL step can be manually spoofed**
A user who edits `?step=3` in the URL jumps to Review with empty data. Fix: on load, read the URL param but clamp it to the highest completed step. If `sessionStorage` has no saved data, always start at step 1.

**4. `sessionStorage` quota exceeded on mobile**
Large file attachments (encoded as base64) can push sessionStorage past its 5MB limit. Fix: never store blobs in session state. Store only text field values; upload files immediately on selection and store the returned URL.

**5. Concurrent tabs share sessionStorage**
Two tabs of the same wizard will interfere. Fix: either use `localStorage` with a unique wizard instance ID in the key, or generate a UUID per wizard session and namespace the storage key.

**6. Angular `FormsModule` two-way binding loses type safety**
`[(ngModel)]` binds to `Record<string, string>`. At submission time, pass the object directly to the service. Use Zod to parse at that boundary — it acts as a type guard so you get strong types after validation passes.

---

## Interview Angle

**Q: "How would you build a multi-step wizard that preserves data when the user goes back?"**

Strong answer covers:
1. **State machine mental model** — each step is a node; transitions are guarded by validation. `useReducer` (React) or a signal-based service (Angular) is preferable to a bag of `useState` calls because all transitions are explicit and auditable.
2. **Uncontrolled vs. controlled inputs** — for back-navigation to preserve values, either store values centrally and restore `defaultValue`, or use fully controlled inputs tied to the reducer store.
3. **Validation strategy** — validate per-field `onBlur`, validate full step on "Next". Use a schema library (Zod) for a single source of truth for types and error messages.
4. **Persistence** — `sessionStorage` survives refresh within the tab, not across tabs. Namespace storage keys to avoid collisions. Never store blobs.

**Follow-up: "How do you prevent users from deep-linking to step 3?"**
Check on mount: if `?step=N` exceeds the highest completed step (tracked in state), silently redirect to the correct step. The completed steps set is the authority, not the URL.
