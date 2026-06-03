# useId

## Overview

`useId` is a React 18+ hook that generates **unique, stable IDs** suitable for accessibility attributes. It solves the problem of generating IDs that are consistent between server and client rendering.

```jsx
const id = useId();
```

## Why useId Exists

### The Problem Before useId

```jsx
// Bad: Counter-based IDs
let counter = 0;
function Form() {
  const [id] = useState(() => `field-${counter++}`);
  
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}

// Problem 1: Not SSR-safe (server/client mismatch)
// Problem 2: Not safe with concurrent rendering
// Problem 3: Not safe when components unmount/remount
```

```jsx
// Bad: Random IDs
function Form() {
  const [id] = useState(() => `field-${Math.random()}`);
  
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}

// Problem: Server generates one ID, client generates different ID
// → Hydration mismatch!
```

### The Solution: useId

```jsx
function Form() {
  const id = useId();
  
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}

// ✓ Same ID on server and client
// ✓ Unique across the tree
// ✓ Safe with concurrent features
```

## Basic Usage

### Form Fields

```jsx
function TextField({ label, ...props }) {
  const id = useId();
  
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} {...props} />
    </div>
  );
}

// Each instance gets unique ID
function Form() {
  return (
    <>
      <TextField label="First Name" />  {/* Unique ID */}
      <TextField label="Last Name" />   {/* Different unique ID */}
    </>
  );
}
```

### ARIA Attributes

```jsx
function Accordion({ title, children }) {
  const id = useId();
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button
        aria-expanded={isOpen}
        aria-controls={id}
        onClick={() => setIsOpen(!isOpen)}
      >
        {title}
      </button>
      
      <div id={id} hidden={!isOpen}>
        {children}
      </div>
    </div>
  );
}
```

### Multiple Related IDs

```jsx
function FormField({ label, error, hint }) {
  const baseId = useId();
  
  return (
    <div>
      <label htmlFor={baseId}>
        {label}
      </label>
      
      <input
        id={baseId}
        aria-describedby={
          [
            hint && `${baseId}-hint`,
            error && `${baseId}-error`
          ].filter(Boolean).join(' ')
        }
      />
      
      {hint && (
        <div id={`${baseId}-hint`} className="hint">
          {hint}
        </div>
      )}
      
      {error && (
        <div id={`${baseId}-error`} className="error">
          {error}
        </div>
      )}
    </div>
  );
}

// Usage
<FormField
  label="Email"
  hint="We'll never share your email"
  error={errors.email}
/>
```

## Generated ID Format

```jsx
const id = useId();
console.log(id);  // ":r0:"

// Format: ":r{number}:"
// - Starts and ends with ":"
// - Contains "r" prefix
// - Number increments per component instance
// - Stable across renders
// - Same on server and client
```

## Common Patterns

### Pattern 1: Radio Button Group

```jsx
function RadioGroup({ label, options, value, onChange }) {
  const groupId = useId();
  
  return (
    <fieldset>
      <legend>{label}</legend>
      {options.map((option, index) => {
        const id = `${groupId}-${index}`;
        return (
          <div key={option.value}>
            <input
              type="radio"
              id={id}
              name={groupId}
              value={option.value}
              checked={value === option.value}
              onChange={() => onChange(option.value)}
            />
            <label htmlFor={id}>{option.label}</label>
          </div>
        );
      })}
    </fieldset>
  );
}

// Usage
<RadioGroup
  label="Size"
  options={[
    { value: 's', label: 'Small' },
    { value: 'm', label: 'Medium' },
    { value: 'l', label: 'Large' }
  ]}
  value={size}
  onChange={setSize}
/>
```

### Pattern 2: Combobox with Listbox

```jsx
function Combobox({ label, options }) {
  const id = useId();
  const [isOpen, setIsOpen] = useState(false);
  const [selectedIndex, setSelectedIndex] = useState(-1);
  
  return (
    <div>
      <label id={`${id}-label`} htmlFor={`${id}-input`}>
        {label}
      </label>
      
      <input
        id={`${id}-input`}
        role="combobox"
        aria-controls={`${id}-listbox`}
        aria-expanded={isOpen}
        aria-activedescendant={
          selectedIndex >= 0 ? `${id}-option-${selectedIndex}` : undefined
        }
        aria-labelledby={`${id}-label`}
      />
      
      {isOpen && (
        <ul
          id={`${id}-listbox`}
          role="listbox"
          aria-labelledby={`${id}-label`}
        >
          {options.map((option, index) => (
            <li
              key={index}
              id={`${id}-option-${index}`}
              role="option"
              aria-selected={index === selectedIndex}
            >
              {option}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Pattern 3: Tabs with Panels

```jsx
function Tabs({ tabs }) {
  const id = useId();
  const [activeIndex, setActiveIndex] = useState(0);
  
  return (
    <div>
      <div role="tablist" aria-labelledby={`${id}-label`}>
        {tabs.map((tab, index) => (
          <button
            key={index}
            id={`${id}-tab-${index}`}
            role="tab"
            aria-selected={index === activeIndex}
            aria-controls={`${id}-panel-${index}`}
            onClick={() => setActiveIndex(index)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      
      {tabs.map((tab, index) => (
        <div
          key={index}
          id={`${id}-panel-${index}`}
          role="tabpanel"
          aria-labelledby={`${id}-tab-${index}`}
          hidden={index !== activeIndex}
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}
```

### Pattern 4: Tooltip

```jsx
function Tooltip({ children, content }) {
  const id = useId();
  const [isVisible, setIsVisible] = useState(false);
  
  return (
    <>
      <button
        aria-describedby={isVisible ? id : undefined}
        onMouseEnter={() => setIsVisible(true)}
        onMouseLeave={() => setIsVisible(false)}
        onFocus={() => setIsVisible(true)}
        onBlur={() => setIsVisible(false)}
      >
        {children}
      </button>
      
      {isVisible && (
        <div id={id} role="tooltip">
          {content}
        </div>
      )}
    </>
  );
}
```

### Pattern 5: Form with Validation

```jsx
function ValidatedInput({ label, validate, ...props }) {
  const id = useId();
  const [value, setValue] = useState('');
  const [error, setError] = useState('');
  
  const handleChange = (e) => {
    const newValue = e.target.value;
    setValue(newValue);
    
    const validationError = validate(newValue);
    setError(validationError);
  };
  
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      
      <input
        id={id}
        value={value}
        onChange={handleChange}
        aria-invalid={!!error}
        aria-describedby={error ? `${id}-error` : undefined}
        {...props}
      />
      
      {error && (
        <div id={`${id}-error`} role="alert">
          {error}
        </div>
      )}
    </div>
  );
}

// Usage
<ValidatedInput
  label="Email"
  validate={(v) => !v.includes('@') ? 'Invalid email' : ''}
/>
```

## Advanced Examples

### Example 1: Dynamic Form Builder

```jsx
function DynamicForm({ fields }) {
  const formId = useId();
  
  return (
    <form id={formId}>
      {fields.map((field, index) => (
        <DynamicField
          key={index}
          field={field}
          fieldId={`${formId}-field-${index}`}
        />
      ))}
    </form>
  );
}

function DynamicField({ field, fieldId }) {
  const id = useId();
  // Combine passed ID with generated ID for uniqueness
  const uniqueId = `${fieldId}-${id}`;
  
  switch (field.type) {
    case 'text':
      return (
        <>
          <label htmlFor={uniqueId}>{field.label}</label>
          <input id={uniqueId} type="text" />
        </>
      );
    case 'select':
      return (
        <>
          <label htmlFor={uniqueId}>{field.label}</label>
          <select id={uniqueId}>
            {field.options.map((opt) => (
              <option key={opt.value} value={opt.value}>
                {opt.label}
              </option>
            ))}
          </select>
        </>
      );
    default:
      return null;
  }
}
```

### Example 2: Reusable Dialog

```jsx
function Dialog({ title, children, isOpen, onClose }) {
  const id = useId();
  
  if (!isOpen) return null;
  
  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby={`${id}-title`}
      aria-describedby={`${id}-description`}
    >
      <h2 id={`${id}-title`}>{title}</h2>
      
      <div id={`${id}-description`}>
        {children}
      </div>
      
      <button onClick={onClose} aria-label="Close dialog">
        ×
      </button>
    </div>
  );
}
```

### Example 3: Multi-Step Form

```jsx
function MultiStepForm({ steps }) {
  const formId = useId();
  const [currentStep, setCurrentStep] = useState(0);
  
  return (
    <div>
      <div role="group" aria-labelledby={`${formId}-progress`}>
        <div id={`${formId}-progress`}>
          Step {currentStep + 1} of {steps.length}
        </div>
        
        <div
          role="progressbar"
          aria-valuenow={currentStep + 1}
          aria-valuemin={1}
          aria-valuemax={steps.length}
          aria-labelledby={`${formId}-progress`}
        />
      </div>
      
      <div
        id={`${formId}-step-${currentStep}`}
        role="region"
        aria-labelledby={`${formId}-step-title-${currentStep}`}
      >
        <h2 id={`${formId}-step-title-${currentStep}`}>
          {steps[currentStep].title}
        </h2>
        {steps[currentStep].content}
      </div>
      
      <button
        onClick={() => setCurrentStep(s => s - 1)}
        disabled={currentStep === 0}
      >
        Previous
      </button>
      
      <button
        onClick={() => setCurrentStep(s => s + 1)}
        disabled={currentStep === steps.length - 1}
      >
        Next
      </button>
    </div>
  );
}
```

## TypeScript

```tsx
import { useId } from 'react';

interface TextFieldProps {
  label: string;
  error?: string;
  hint?: string;
}

function TextField({ label, error, hint, ...props }: TextFieldProps) {
  const id = useId();
  
  const describedBy = [
    hint && `${id}-hint`,
    error && `${id}-error`
  ].filter((x): x is string => Boolean(x)).join(' ');
  
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      
      <input
        id={id}
        aria-describedby={describedBy || undefined}
        aria-invalid={!!error}
        {...props}
      />
      
      {hint && <div id={`${id}-hint`}>{hint}</div>}
      {error && <div id={`${id}-error`}>{error}</div>}
    </div>
  );
}
```

## Common Mistakes

### Mistake 1: Using useId for Keys

```jsx
// Bad: Don't use for list keys
function List({ items }) {
  const id = useId();
  
  return (
    <ul>
      {items.map((item, index) => (
        <li key={`${id}-${index}`}>  {/* Don't do this! */}
          {item}
        </li>
      ))}
    </ul>
  );
}

// Good: Use stable data IDs
function List({ items }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>  {/* Use data ID */}
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

**Why**: List keys should be based on data identity, not component instances.

### Mistake 2: Generating Multiple IDs

```jsx
// Bad: Multiple useId calls
function Form() {
  const nameId = useId();
  const emailId = useId();
  const phoneId = useId();
  
  return (/* ... */);
}

// Good: One base ID
function Form() {
  const id = useId();
  
  return (
    <>
      <label htmlFor={`${id}-name`}>Name</label>
      <input id={`${id}-name`} />
      
      <label htmlFor={`${id}-email`}>Email</label>
      <input id={`${id}-email`} />
      
      <label htmlFor={`${id}-phone`}>Phone</label>
      <input id={`${id}-phone`} />
    </>
  );
}
```

**Why**: One `useId` call is cheaper and generates related IDs.

### Mistake 3: Using for CSS

```jsx
// Bad: Using ID for styling
function Component() {
  const id = useId();
  
  return (
    <>
      <style>{`
        #${id} {
          color: red;
        }
      `}</style>
      <div id={id}>Content</div>
    </>
  );
}

// Good: Use className
function Component() {
  return (
    <div className="component">
      Content
    </div>
  );
}
```

**Why**: IDs should be for accessibility, not styling.

### Mistake 4: Conditional useId

```jsx
// Bad: Conditional hook
function Component({ needsId }) {
  if (needsId) {
    const id = useId();  // Breaks rules of hooks!
  }
}

// Good: Always call hook
function Component({ needsId }) {
  const id = useId();
  
  if (!needsId) {
    return <div>No ID needed</div>;
  }
  
  return <div id={id}>Has ID</div>;
}
```

## SSR Compatibility

### Why It's SSR-Safe

```jsx
// Server renders:
<label htmlFor=":r0:">Name</label>
<input id=":r0:" />

// Client hydrates:
<label htmlFor=":r0:">Name</label>
<input id=":r0:" />

// Same ID! No hydration mismatch.
```

### With Multiple Root Trees

```jsx
// If you have multiple React roots:
createRoot(document.getElementById('root1')).render(<App1 />);
createRoot(document.getElementById('root2')).render(<App2 />);

// IDs are still unique across roots
```

## Migration from Old Patterns

### From Counter-Based IDs

```jsx
// Before
let counter = 0;
function Field() {
  const [id] = useState(() => `field-${counter++}`);
  return <input id={id} />;
}

// After
function Field() {
  const id = useId();
  return <input id={id} />;
}
```

### From uuid/nanoid

```jsx
// Before
import { nanoid } from 'nanoid';

function Field() {
  const [id] = useState(() => nanoid());
  return <input id={id} />;
}

// After (if SSR-safe IDs needed)
function Field() {
  const id = useId();
  return <input id={id} />;
}

// Keep uuid/nanoid for data IDs
function DataItem() {
  const [dataId] = useState(() => nanoid());  // This is fine
  // dataId is for your data model, not DOM
}
```

## When NOT to Use useId

1. **List Keys**: Use data IDs instead
2. **Data IDs**: Use uuid/nanoid for database IDs
3. **Session IDs**: Use secure random generators
4. **CSS Selectors**: Use className instead
5. **Tracking**: Use analytics-specific IDs

## When TO Use useId

1. **Form field IDs**: `<label htmlFor={id}>`
2. **ARIA attributes**: `aria-labelledby`, `aria-describedby`, etc.
3. **Multiple related elements**: Tabs, accordion panels, etc.
4. **SSR applications**: Any ID that must match server/client
5. **Accessibility patterns**: Any WCAG-required relationships

## Browser Support

- React 18+
- Works in all browsers (it's just a string)
- SSR requires React 18+ on server

## Resources

- [React Docs: useId](https://react.dev/reference/react/useId)
- [Accessible Rich Internet Applications (ARIA)](https://www.w3.org/WAI/ARIA/apg/)
- [Using ARIA: Roles, States, and Properties](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Techniques)
