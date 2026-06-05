# Controlled vs Uncontrolled Components

## The Idea

**In plain English:** A controlled component is a form input (like a text box) where React is always in charge of what value it shows — every keystroke goes through React first. An uncontrolled component lets the browser's own memory hold the value, and React only checks it when needed (like when you hit Submit).

**Real-world analogy:** Imagine a whiteboard in a classroom where a strict teacher (React) erases and rewrites whatever a student tries to write, keeping the official version in their own notebook. Compare that to a sticky note on a student's desk — the student writes freely, and the teacher only reads it when they walk over and check.

- The teacher's notebook = React state (the single source of truth for a controlled input)
- The teacher rewriting the whiteboard = React re-rendering the input value from state on every change
- The sticky note on the desk = the DOM storing the value internally in an uncontrolled input
- The teacher walking over to read the sticky note = using a `ref` to grab the value when you need it

---

## Overview

In React, form inputs can be controlled or uncontrolled. This distinction is fundamental to understanding how React manages form state and DOM interactions. Controlled components give React control over form data through state, while uncontrolled components let the DOM handle state internally.

## Controlled Components

A controlled component is a form element whose value is controlled by React state.

### Basic Controlled Input

```jsx
function ControlledInput() {
  const [value, setValue] = useState('');
  
  return (
    <input
      type="text"
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

**Key Characteristics:**
- React state is the "single source of truth"
- Value flows: State → Input
- Changes flow: Input → Handler → State
- React controls what's displayed

### Controlled Form Example

```jsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  const handleSubmit = (e) => {
    e.preventDefault();
    console.log({ email, password });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Multiple Controlled Inputs

```jsx
function UserForm() {
  const [formData, setFormData] = useState({
    firstName: '',
    lastName: '',
    email: '',
    age: ''
  });
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };
  
  return (
    <form>
      <input
        name="firstName"
        value={formData.firstName}
        onChange={handleChange}
        placeholder="First Name"
      />
      <input
        name="lastName"
        value={formData.lastName}
        onChange={handleChange}
        placeholder="Last Name"
      />
      <input
        name="email"
        type="email"
        value={formData.email}
        onChange={handleChange}
        placeholder="Email"
      />
      <input
        name="age"
        type="number"
        value={formData.age}
        onChange={handleChange}
        placeholder="Age"
      />
    </form>
  );
}
```

### Controlled Textarea

```jsx
function CommentForm() {
  const [comment, setComment] = useState('');
  
  return (
    <textarea
      value={comment}
      onChange={(e) => setComment(e.target.value)}
      placeholder="Leave a comment..."
      rows={4}
    />
  );
}
```

### Controlled Select

```jsx
function CountrySelector() {
  const [country, setCountry] = useState('us');
  
  return (
    <select value={country} onChange={(e) => setCountry(e.target.value)}>
      <option value="">Select a country</option>
      <option value="us">United States</option>
      <option value="uk">United Kingdom</option>
      <option value="ca">Canada</option>
      <option value="au">Australia</option>
    </select>
  );
}
```

### Controlled Checkbox

```jsx
function TermsCheckbox() {
  const [accepted, setAccepted] = useState(false);
  
  return (
    <label>
      <input
        type="checkbox"
        checked={accepted}
        onChange={(e) => setAccepted(e.target.checked)}
      />
      I accept the terms and conditions
    </label>
  );
}
```

### Controlled Radio Buttons

```jsx
function PaymentMethod() {
  const [method, setMethod] = useState('credit');
  
  return (
    <div>
      <label>
        <input
          type="radio"
          value="credit"
          checked={method === 'credit'}
          onChange={(e) => setMethod(e.target.value)}
        />
        Credit Card
      </label>
      
      <label>
        <input
          type="radio"
          value="debit"
          checked={method === 'debit'}
          onChange={(e) => setMethod(e.target.value)}
        />
        Debit Card
      </label>
      
      <label>
        <input
          type="radio"
          value="paypal"
          checked={method === 'paypal'}
          onChange={(e) => setMethod(e.target.value)}
        />
        PayPal
      </label>
    </div>
  );
}
```

## Uncontrolled Components

Uncontrolled components store their own state in the DOM. React uses refs to access values when needed.

### Basic Uncontrolled Input

```jsx
function UncontrolledInput() {
  const inputRef = useRef(null);
  
  const handleSubmit = (e) => {
    e.preventDefault();
    // Access value via ref
    console.log(inputRef.current.value);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        ref={inputRef}
        defaultValue="Initial value"
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Key Characteristics:**
- DOM is the source of truth
- Use `ref` to access current value
- Use `defaultValue` instead of `value`
- Less React code, more traditional HTML behavior

### Uncontrolled Form Example

```jsx
function UncontrolledForm() {
  const emailRef = useRef();
  const passwordRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    const formData = {
      email: emailRef.current.value,
      password: passwordRef.current.value
    };
    
    console.log(formData);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        ref={emailRef}
        defaultValue=""
        placeholder="Email"
      />
      <input
        type="password"
        ref={passwordRef}
        defaultValue=""
        placeholder="Password"
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Uncontrolled with FormData

```jsx
function ModernUncontrolledForm() {
  const handleSubmit = (e) => {
    e.preventDefault();
    
    // Use FormData API
    const formData = new FormData(e.target);
    const data = Object.fromEntries(formData);
    
    console.log(data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        name="firstName"
        defaultValue=""
        placeholder="First Name"
      />
      <input
        name="lastName"
        defaultValue=""
        placeholder="Last Name"
      />
      <input
        name="email"
        type="email"
        defaultValue=""
        placeholder="Email"
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Uncontrolled File Input

File inputs are always uncontrolled because their value is read-only:

```jsx
function FileUpload() {
  const fileInputRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    const file = fileInputRef.current.files[0];
    
    if (file) {
      console.log('File name:', file.name);
      console.log('File size:', file.size);
      console.log('File type:', file.type);
      
      // Process file
      const formData = new FormData();
      formData.append('file', file);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="file"
        ref={fileInputRef}
        accept="image/*"
      />
      <button type="submit">Upload</button>
    </form>
  );
}
```

## Comparison

### Controlled vs Uncontrolled

| Aspect | Controlled | Uncontrolled |
|--------|-----------|--------------|
| State location | React state | DOM |
| Value access | Direct from state | Via ref |
| Updates | Instant | On demand |
| Validation | Real-time | On submit |
| Initial value | `value` prop | `defaultValue` prop |
| Code complexity | More React code | Less React code |
| Use case | Most forms | Simple forms, integration |

### When to Use Controlled

```jsx
// 1. Real-time validation
function PasswordInput() {
  const [password, setPassword] = useState('');
  const [strength, setStrength] = useState('weak');
  
  const handleChange = (e) => {
    const value = e.target.value;
    setPassword(value);
    
    // Real-time strength calculation
    if (value.length > 12 && /[A-Z]/.test(value) && /[0-9]/.test(value)) {
      setStrength('strong');
    } else if (value.length > 8) {
      setStrength('medium');
    } else {
      setStrength('weak');
    }
  };
  
  return (
    <div>
      <input
        type="password"
        value={password}
        onChange={handleChange}
      />
      <span className={`strength-${strength}`}>
        Strength: {strength}
      </span>
    </div>
  );
}

// 2. Format input as user types
function PhoneInput() {
  const [phone, setPhone] = useState('');
  
  const handleChange = (e) => {
    const value = e.target.value.replace(/\D/g, '');
    
    // Format: (123) 456-7890
    let formatted = value;
    if (value.length > 3) {
      formatted = `(${value.slice(0, 3)}) ${value.slice(3)}`;
    }
    if (value.length > 6) {
      formatted = `(${value.slice(0, 3)}) ${value.slice(3, 6)}-${value.slice(6, 10)}`;
    }
    
    setPhone(formatted);
  };
  
  return (
    <input
      type="tel"
      value={phone}
      onChange={handleChange}
      placeholder="(123) 456-7890"
    />
  );
}

// 3. Conditional fields
function AddressForm() {
  const [country, setCountry] = useState('us');
  const [state, setState] = useState('');
  const [province, setProvince] = useState('');
  
  return (
    <div>
      <select value={country} onChange={(e) => setCountry(e.target.value)}>
        <option value="us">United States</option>
        <option value="ca">Canada</option>
      </select>
      
      {country === 'us' && (
        <input
          value={state}
          onChange={(e) => setState(e.target.value)}
          placeholder="State"
        />
      )}
      
      {country === 'ca' && (
        <input
          value={province}
          onChange={(e) => setProvince(e.target.value)}
          placeholder="Province"
        />
      )}
    </div>
  );
}

// 4. Disabling submit until valid
function Form() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  const isValid = email.includes('@') && password.length >= 8;
  
  return (
    <form>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button disabled={!isValid}>Submit</button>
    </form>
  );
}
```

### When to Use Uncontrolled

```jsx
// 1. Simple forms
function NewsletterSignup() {
  const emailRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    subscribeToNewsletter(emailRef.current.value);
    e.target.reset(); // Easy reset
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input ref={emailRef} type="email" placeholder="Email" />
      <button>Subscribe</button>
    </form>
  );
}

// 2. Integrating with non-React code
function LegacyFormWrapper() {
  const formRef = useRef();
  
  useEffect(() => {
    // Initialize third-party form library
    const legacyForm = initLegacyForm(formRef.current);
    
    return () => legacyForm.destroy();
  }, []);
  
  return <form ref={formRef}>...</form>;
}

// 3. File uploads
function ImageUpload() {
  const fileRef = useRef();
  
  const handleUpload = async () => {
    const file = fileRef.current.files[0];
    if (file) {
      await uploadFile(file);
    }
  };
  
  return (
    <div>
      <input type="file" ref={fileRef} accept="image/*" />
      <button onClick={handleUpload}>Upload</button>
    </div>
  );
}

// 4. Performance-critical forms
function LargeFormWithManyFields() {
  // Uncontrolled to avoid re-renders on every keystroke
  const formRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    const data = Object.fromEntries(formData);
    submitForm(data);
  };
  
  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      {/* 50+ fields that don't need real-time validation */}
      <input name="field1" defaultValue="" />
      <input name="field2" defaultValue="" />
      {/* ... many more fields ... */}
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Hybrid Approach

Sometimes you need both:

```jsx
function SearchableSelect() {
  // Controlled for the selected value
  const [selected, setSelected] = useState('');
  // Uncontrolled for search input
  const searchRef = useRef();
  
  const handleSelect = (value) => {
    setSelected(value);
    if (searchRef.current) {
      searchRef.current.value = '';
    }
  };
  
  return (
    <div>
      <input
        ref={searchRef}
        placeholder="Search..."
        type="text"
      />
      <div>Selected: {selected}</div>
      <select value={selected} onChange={(e) => handleSelect(e.target.value)}>
        <option value="">Select...</option>
        <option value="a">Option A</option>
        <option value="b">Option B</option>
      </select>
    </div>
  );
}
```

## Default Values

### Controlled Components

```jsx
function ControlledWithInitial({ initialValue = '' }) {
  const [value, setValue] = useState(initialValue);
  
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

### Uncontrolled Components

```jsx
function UncontrolledWithDefault({ defaultValue = '' }) {
  const inputRef = useRef();
  
  return (
    <input
      ref={inputRef}
      defaultValue={defaultValue}
    />
  );
}
```

### Switching from Uncontrolled to Controlled

```jsx
// Warning: A component is changing an uncontrolled input to be controlled
function ProblematicComponent() {
  const [value, setValue] = useState(); // undefined!
  
  return (
    <input
      value={value} // undefined becomes controlled when set
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// Fix: Initialize with empty string
function FixedComponent() {
  const [value, setValue] = useState(''); // Always controlled
  
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

## Advanced Patterns

### Controlled with Debouncing

```jsx
function DebouncedInput({ onChange, delay = 500 }) {
  const [value, setValue] = useState('');
  
  useEffect(() => {
    const timer = setTimeout(() => {
      onChange(value);
    }, delay);
    
    return () => clearTimeout(timer);
  }, [value, delay, onChange]);
  
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

### Controlled with Validation

```jsx
function ValidatedInput({ validator, onValidChange }) {
  const [value, setValue] = useState('');
  const [error, setError] = useState('');
  
  const handleChange = (e) => {
    const newValue = e.target.value;
    setValue(newValue);
    
    const validationError = validator(newValue);
    setError(validationError);
    
    if (!validationError) {
      onValidChange(newValue);
    }
  };
  
  return (
    <div>
      <input
        value={value}
        onChange={handleChange}
        className={error ? 'error' : ''}
      />
      {error && <span className="error-message">{error}</span>}
    </div>
  );
}

// Usage
<ValidatedInput
  validator={(value) => {
    if (!value) return 'Required';
    if (value.length < 3) return 'Too short';
    return '';
  }}
  onValidChange={(value) => console.log('Valid value:', value)}
/>
```

### Focus Management

```jsx
function FormWithFocus() {
  const firstInputRef = useRef();
  const lastInputRef = useRef();
  
  useEffect(() => {
    // Focus first input on mount
    firstInputRef.current?.focus();
  }, []);
  
  const handleFirstKeyDown = (e) => {
    if (e.key === 'Enter') {
      lastInputRef.current?.focus();
    }
  };
  
  return (
    <form>
      <input
        ref={firstInputRef}
        onKeyDown={handleFirstKeyDown}
        placeholder="First field"
      />
      <input
        ref={lastInputRef}
        placeholder="Last field"
      />
    </form>
  );
}
```

## Best Practices

### 1. Prefer Controlled for Most Cases

Controlled components provide better control and are more React-like:

```jsx
// Preferred
function Form() {
  const [email, setEmail] = useState('');
  
  return (
    <input
      type="email"
      value={email}
      onChange={(e) => setEmail(e.target.value)}
    />
  );
}
```

### 2. Use Uncontrolled for Simple Cases

```jsx
// Good for simple submissions
function QuickForm() {
  const handleSubmit = (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    api.submit(Object.fromEntries(formData));
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" />
      <button>Submit</button>
    </form>
  );
}
```

### 3. Never Mix Controlled and Uncontrolled

```jsx
// Wrong: Mixing approaches
function Bad() {
  const [value, setValue] = useState('');
  
  return (
    <input
      value={value}
      defaultValue="something" // This will be ignored!
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

### 4. Always Initialize Controlled Values

```jsx
// Wrong
const [value, setValue] = useState(); // undefined

// Correct
const [value, setValue] = useState(''); // empty string
```

## Summary

- **Controlled components**: React state manages value, provides full control
- **Uncontrolled components**: DOM manages state, accessed via refs
- Use controlled for: validation, formatting, conditional fields, most forms
- Use uncontrolled for: simple forms, file inputs, third-party integration, performance
- Never mix `value` and `defaultValue` on the same input
- Always initialize controlled components with a defined value
- Controlled provides better user experience and developer experience
- Uncontrolled can be simpler for basic forms
- File inputs are always uncontrolled
- Modern forms typically use controlled components with form libraries like React Hook Form

Understanding the difference between controlled and uncontrolled components is crucial for building robust, maintainable forms in React. Choose the right approach based on your specific needs.
