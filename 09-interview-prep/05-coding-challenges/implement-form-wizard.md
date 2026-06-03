# Implement Multi-Step Form Wizard

## State Machine Approach

A form wizard is a natural fit for a state machine — it has a finite number of steps with clear transitions.

```ts
type WizardStep = 'account' | 'profile' | 'preferences' | 'review';

interface WizardState {
  currentStep: WizardStep;
  data: {
    account: Partial<AccountData>;
    profile: Partial<ProfileData>;
    preferences: Partial<PreferencesData>;
  };
  status: 'filling' | 'submitting' | 'success' | 'error';
  error: string | null;
}

type WizardAction =
  | { type: 'NEXT'; stepData: Partial<AccountData | ProfileData | PreferencesData> }
  | { type: 'BACK' }
  | { type: 'GO_TO'; step: WizardStep }
  | { type: 'SUBMIT' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_ERROR'; error: string };

const STEPS: WizardStep[] = ['account', 'profile', 'preferences', 'review'];

function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case 'NEXT': {
      const currentIndex = STEPS.indexOf(state.currentStep);
      const nextStep = STEPS[currentIndex + 1];
      return {
        ...state,
        currentStep: nextStep,
        data: { ...state.data, [state.currentStep]: action.stepData },
      };
    }
    case 'BACK': {
      const currentIndex = STEPS.indexOf(state.currentStep);
      return { ...state, currentStep: STEPS[currentIndex - 1] };
    }
    case 'SUBMIT':
      return { ...state, status: 'submitting' };
    case 'SUBMIT_SUCCESS':
      return { ...state, status: 'success' };
    case 'SUBMIT_ERROR':
      return { ...state, status: 'error', error: action.error };
    default:
      return state;
  }
}
```

---

## Wizard Component

```tsx
function FormWizard() {
  const [state, dispatch] = useReducer(wizardReducer, {
    currentStep: 'account',
    data: { account: {}, profile: {}, preferences: {} },
    status: 'filling',
    error: null,
  });

  const currentIndex = STEPS.indexOf(state.currentStep);

  async function handleSubmit() {
    dispatch({ type: 'SUBMIT' });
    try {
      await api.createUser(state.data);
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (err) {
      dispatch({ type: 'SUBMIT_ERROR', error: err.message });
    }
  }

  if (state.status === 'success') return <SuccessPage />;

  return (
    <div>
      {/* Progress indicator */}
      <WizardProgress steps={STEPS} currentIndex={currentIndex} />

      {/* Step panels */}
      <div aria-live="polite">
        {state.currentStep === 'account' && (
          <AccountStep
            data={state.data.account}
            onNext={(data) => dispatch({ type: 'NEXT', stepData: data })}
          />
        )}
        {state.currentStep === 'profile' && (
          <ProfileStep
            data={state.data.profile}
            onNext={(data) => dispatch({ type: 'NEXT', stepData: data })}
            onBack={() => dispatch({ type: 'BACK' })}
          />
        )}
        {state.currentStep === 'preferences' && (
          <PreferencesStep
            data={state.data.preferences}
            onNext={(data) => dispatch({ type: 'NEXT', stepData: data })}
            onBack={() => dispatch({ type: 'BACK' })}
          />
        )}
        {state.currentStep === 'review' && (
          <ReviewStep
            data={state.data}
            onSubmit={handleSubmit}
            onBack={() => dispatch({ type: 'BACK' })}
            isSubmitting={state.status === 'submitting'}
            error={state.error}
          />
        )}
      </div>
    </div>
  );
}
```

---

## Per-Step Validation with React Hook Form + Zod

```tsx
const accountSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});

function AccountStep({
  data,
  onNext,
}: {
  data: Partial<AccountData>;
  onNext: (data: AccountData) => void;
}) {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<AccountData>({
    resolver: zodResolver(accountSchema),
    defaultValues: data,
  });

  return (
    <form onSubmit={handleSubmit(onNext)}>
      <h2 id="step-heading">Create your account</h2>

      <label>
        Email
        <input {...register('email')} type="email" aria-describedby="email-error" />
        {errors.email && <span id="email-error" role="alert">{errors.email.message}</span>}
      </label>

      <label>
        Password
        <input {...register('password')} type="password" />
        {errors.password && <span role="alert">{errors.password.message}</span>}
      </label>

      <button type="submit">Next →</button>
    </form>
  );
}
```

---

## URL-Based Steps (Deep Linking)

```tsx
function useWizardWithURL() {
  const [searchParams, setSearchParams] = useSearchParams();
  const step = (searchParams.get('step') as WizardStep) ?? 'account';

  function goToStep(nextStep: WizardStep) {
    setSearchParams({ step: nextStep });
  }

  return { currentStep: step, goToStep };
}

// URL: /signup?step=profile
// User can bookmark mid-wizard, back button works naturally
```

---

## Progress Indicator

```tsx
function WizardProgress({
  steps,
  currentIndex,
}: {
  steps: string[];
  currentIndex: number;
}) {
  return (
    <nav aria-label="Form progress">
      <ol className="wizard-steps">
        {steps.map((step, index) => (
          <li
            key={step}
            aria-current={index === currentIndex ? 'step' : undefined}
            className={
              index < currentIndex ? 'completed' :
              index === currentIndex ? 'current' : 'upcoming'
            }
          >
            <span className="step-number" aria-hidden="true">{index + 1}</span>
            <span className="step-label">{step}</span>
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

---

## Common Interview Questions

**Q: How do you preserve form data when going back?**
Store partial form data in the reducer state (`state.data.account`, etc.). When the step mounts, pass `defaultValues` from the stored data so the form re-hydrates.

**Q: How do you handle server-side validation errors?**
After submission fails, parse the error response and use `setError` from React Hook Form to display field-level errors on the relevant step, or show a global error banner on the review step.

**Q: What's the tradeoff of URL-based vs state-based wizard?**
URL-based: browser back button works, deep linking, refresh-safe (if you persist data). State-based: simpler, no URL pollution, better for in-page flows. URL-based is better for multi-page registrations; state-based for checkout overlays.
