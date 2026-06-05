# Controlled vs Uncontrolled Components

## The Idea

**In plain English:** In React, a "controlled" input is one where your app keeps track of what the user types (using state), while an "uncontrolled" input lets the browser handle it and your app only checks the value when it needs to.

**Real-world analogy:** Imagine a whiteboard at the front of a classroom. In a controlled setup, the teacher (your React app) erases and rewrites what's on the board after every student suggestion — the teacher always decides what's shown. In an uncontrolled setup, the student writes on the board freely, and the teacher only walks over to read it when they need to know the answer.

- The teacher = React (your app's state and logic)
- The whiteboard = the input field displayed on screen
- The student writing freely = the browser managing the input's value on its own

---

## The Core Difference

**Controlled**: React state is the **single source of truth**. The DOM value is always a reflection of state.

**Uncontrolled**: The DOM itself holds the value. React reads it when needed via a `ref`.

---

## Controlled Components

```tsx
function ControlledForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <form onSubmit={(e) => { e.preventDefault(); login(email, password); }}>
      <input
        value={email}           // ← controlled by state
        onChange={e => setEmail(e.target.value)}
        type="email"
      />
      <input
        value={password}        // ← controlled by state
        onChange={e => setPassword(e.target.value)}
        type="password"
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

**Characteristics:**
- Value always matches `state` — React controls what's displayed
- `onChange` required to update state; without it the input is frozen
- Instantly access current value anywhere (it's just state)
- Can transform/validate on every keystroke
- Enables conditional rendering based on input values

---

## Uncontrolled Components

```tsx
function UncontrolledForm() {
  const emailRef = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    login(emailRef.current!.value, passwordRef.current!.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={emailRef} type="email" defaultValue="" />
      <input ref={passwordRef} type="password" />
      <button type="submit">Login</button>
    </form>
  );
}
```

**Characteristics:**
- DOM manages the value — React only reads it on demand via `ref`
- `defaultValue` (not `value`) sets the initial value
- Value is only available when you explicitly read `ref.current.value`
- Cannot react to changes in real-time (no onChange-driven logic)

---

## When to Use Which

**Use controlled when you need:**
- Real-time validation (show error as user types)
- Conditional rendering based on input value
- Synchronizing multiple related inputs
- Transforming input (e.g., force uppercase, format phone number)
- Programmatically changing the value from outside

```tsx
// Controlled — real-time validation
const [username, setUsername] = useState('');
const isValid = username.length >= 3;

<input value={username} onChange={e => setUsername(e.target.value)} />
{!isValid && <span>Must be at least 3 characters</span>}
```

**Use uncontrolled when:**
- Simple forms where you only need the value on submit
- Integrating with non-React DOM libraries
- File inputs (always uncontrolled — `<input type="file">` value cannot be set)
- Performance-critical forms with many fields (less re-rendering)

```tsx
// File input is always uncontrolled
const fileRef = useRef<HTMLInputElement>(null);
<input type="file" ref={fileRef} />
// Read: fileRef.current.files[0]
```

---

## React Hook Form — Best of Both

React Hook Form uses uncontrolled inputs by default (better performance) while providing controlled-like DX:

```tsx
import { useForm } from 'react-hook-form';

function Form() {
  const { register, handleSubmit, watch, formState: { errors } } = useForm();

  // watch() re-renders on value change (opt-in controlled behavior)
  const username = watch('username');

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* register attaches ref + onChange/onBlur — uncontrolled under the hood */}
      <input {...register('username', { required: true, minLength: 3 })} />
      {errors.username && <span>Required, min 3 chars</span>}
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Hybrid Pattern

Controlled for the fields you need to react to, uncontrolled for the rest:

```tsx
function HybridForm() {
  const [plan, setPlan] = useState('free');         // controlled
  const companyRef = useRef<HTMLInputElement>(null); // uncontrolled

  return (
    <form>
      <select value={plan} onChange={e => setPlan(e.target.value)}>
        <option value="free">Free</option>
        <option value="pro">Pro</option>
      </select>

      {/* Only show company field for pro plan — needs controlled 'plan' */}
      {plan === 'pro' && <input ref={companyRef} placeholder="Company name" />}
    </form>
  );
}
```

---

## Common Interview Questions

**Q: What happens if you set `value` without `onChange` on an input?**
The input becomes read-only (frozen). React warns: "You provided a `value` prop without an `onChange` handler." Use `readOnly` if intentional, or add `onChange`.

**Q: What's `defaultValue` vs `value`?**
`value` = controlled (React owns it). `defaultValue` = uncontrolled (sets initial value, then DOM owns it). Never use both together on the same input.

**Q: Can you switch between controlled and uncontrolled?**
No — React warns against it. Decide at mount time. Changing from `undefined` value to a defined value (or vice versa) triggers a warning and unexpected behavior.
