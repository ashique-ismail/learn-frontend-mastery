# Controlled Props Pattern

## Overview

The Controlled Props Pattern is a React design pattern that gives parent components complete control over a child component's state. Similar to how controlled form inputs work, this pattern allows components to be either controlled (parent manages state) or uncontrolled (component manages its own state), providing flexibility for different use cases.

This pattern is fundamental in building reusable components that work in various contexts, from simple self-contained widgets to complex, tightly integrated UI systems.

## Core Concepts

### What Are Controlled Props?

A component uses controlled props when:

1. **Parent passes state values** as props
2. **Parent passes change handlers** to update that state
3. **Component uses prop values** instead of internal state
4. **Component notifies parent** of state changes via callbacks

### Controlled vs Uncontrolled

```javascript
// Uncontrolled: Component manages its own state
function UncontrolledInput() {
  const [value, setValue] = useState('');
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// Controlled: Parent manages state
function ControlledInput({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}

// Usage
function Parent() {
  const [value, setValue] = useState('');
  return (
    <ControlledInput
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

### When to Use This Pattern

Use the Controlled Props Pattern when:

- Components need to sync state with parent components
- Multiple components need to share the same state
- Parent needs to validate or transform changes before applying them
- External factors should control component behavior
- You're building a component library with maximum flexibility

## Basic Implementation

### Simple Controlled Component

```javascript
function Counter({ value, onChange }) {
  const handleIncrement = () => {
    onChange(value + 1);
  };

  const handleDecrement = () => {
    onChange(value - 1);
  };

  return (
    <div>
      <button onClick={handleDecrement}>-</button>
      <span>{value}</span>
      <button onClick={handleIncrement}>+</button>
    </div>
  );
}

// Usage
function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <Counter value={count} onChange={setCount} />
      <p>The count is: {count}</p>
      <button onClick={() => setCount(0)}>Reset from parent</button>
    </div>
  );
}
```

### Hybrid: Controlled or Uncontrolled

```javascript
function FlexibleCounter({ value: controlledValue, onChange, initialValue = 0 }) {
  // Internal state for uncontrolled mode
  const [internalValue, setInternalValue] = useState(initialValue);

  // Use controlled value if provided, otherwise internal
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;

  const handleChange = (newValue) => {
    // Update internal state if uncontrolled
    if (!isControlled) {
      setInternalValue(newValue);
    }
    // Always notify parent (if callback provided)
    onChange?.(newValue);
  };

  return (
    <div>
      <button onClick={() => handleChange(value - 1)}>-</button>
      <span>{value}</span>
      <button onClick={() => handleChange(value + 1)}>+</button>
    </div>
  );
}

// Uncontrolled usage
function UncontrolledExample() {
  return (
    <FlexibleCounter
      initialValue={5}
      onChange={(v) => console.log('Changed to:', v)}
    />
  );
}

// Controlled usage
function ControlledExample() {
  const [count, setCount] = useState(0);
  return <FlexibleCounter value={count} onChange={setCount} />;
}
```

## Advanced Examples

### Controlled Toggle with Multiple Props

```javascript
function Toggle({
  // Controlled props
  on,
  onToggle,
  // Uncontrolled fallback
  defaultOn = false,
  // Styling
  className = '',
  disabled = false,
}) {
  const [internalOn, setInternalOn] = useState(defaultOn);

  const isControlled = on !== undefined;
  const isOn = isControlled ? on : internalOn;

  const handleToggle = () => {
    if (disabled) return;

    const newValue = !isOn;

    if (!isControlled) {
      setInternalOn(newValue);
    }

    onToggle?.(newValue);
  };

  return (
    <button
      onClick={handleToggle}
      disabled={disabled}
      className={`toggle ${isOn ? 'on' : 'off'} ${className}`}
      aria-pressed={isOn}
    >
      {isOn ? 'ON' : 'OFF'}
    </button>
  );
}

// Usage: Parent controls when toggle is on
function ControlledToggleExample() {
  const [isEnabled, setIsEnabled] = useState(false);
  const [canToggle, setCanToggle] = useState(true);

  return (
    <div>
      <Toggle
        on={isEnabled}
        onToggle={setIsEnabled}
        disabled={!canToggle}
      />
      <p>Status: {isEnabled ? 'Enabled' : 'Disabled'}</p>
      <button onClick={() => setCanToggle(!canToggle)}>
        {canToggle ? 'Lock' : 'Unlock'} Toggle
      </button>
    </div>
  );
}
```

### Controlled Tabs Component

```javascript
function Tabs({
  // Controlled props
  activeTab,
  onTabChange,
  // Uncontrolled fallback
  defaultActiveTab = 0,
  // Content
  children,
}) {
  const [internalActiveTab, setInternalActiveTab] = useState(
    defaultActiveTab
  );

  const isControlled = activeTab !== undefined;
  const currentTab = isControlled ? activeTab : internalActiveTab;

  const handleTabChange = (index) => {
    if (!isControlled) {
      setInternalActiveTab(index);
    }
    onTabChange?.(index);
  };

  const tabs = React.Children.toArray(children);

  return (
    <div>
      <div className="tab-list">
        {tabs.map((tab, index) => (
          <button
            key={index}
            onClick={() => handleTabChange(index)}
            className={currentTab === index ? 'active' : ''}
          >
            {tab.props.label}
          </button>
        ))}
      </div>
      <div className="tab-content">{tabs[currentTab]}</div>
    </div>
  );
}

function Tab({ children }) {
  return <div>{children}</div>;
}

// Controlled usage: sync tabs with URL
function ControlledTabsExample() {
  const [searchParams, setSearchParams] = useSearchParams();
  const activeTab = parseInt(searchParams.get('tab') || '0', 10);

  const handleTabChange = (index) => {
    setSearchParams({ tab: index.toString() });
  };

  return (
    <Tabs activeTab={activeTab} onTabChange={handleTabChange}>
      <Tab label="Home">Home Content</Tab>
      <Tab label="Profile">Profile Content</Tab>
      <Tab label="Settings">Settings Content</Tab>
    </Tabs>
  );
}

// Uncontrolled usage: internal state only
function UncontrolledTabsExample() {
  return (
    <Tabs defaultActiveTab={1} onTabChange={(i) => console.log('Tab:', i)}>
      <Tab label="Home">Home Content</Tab>
      <Tab label="Profile">Profile Content</Tab>
      <Tab label="Settings">Settings Content</Tab>
    </Tabs>
  );
}
```

### Controlled Accordion

```javascript
function Accordion({
  // Controlled props
  expandedItems,
  onExpandedChange,
  // Uncontrolled fallback
  defaultExpandedItems = [],
  // Config
  allowMultiple = false,
  children,
}) {
  const [internalExpanded, setInternalExpanded] = useState(
    defaultExpandedItems
  );

  const isControlled = expandedItems !== undefined;
  const expanded = isControlled ? expandedItems : internalExpanded;

  const handleToggle = (itemId) => {
    let newExpanded;

    if (allowMultiple) {
      // Toggle item in array
      newExpanded = expanded.includes(itemId)
        ? expanded.filter((id) => id !== itemId)
        : [...expanded, itemId];
    } else {
      // Only one item at a time
      newExpanded = expanded.includes(itemId) ? [] : [itemId];
    }

    if (!isControlled) {
      setInternalExpanded(newExpanded);
    }

    onExpandedChange?.(newExpanded);
  };

  return (
    <div className="accordion">
      {React.Children.map(children, (child) =>
        React.cloneElement(child, {
          isExpanded: expanded.includes(child.props.id),
          onToggle: () => handleToggle(child.props.id),
        })
      )}
    </div>
  );
}

function AccordionItem({ id, title, children, isExpanded, onToggle }) {
  return (
    <div className="accordion-item">
      <button onClick={onToggle} className="accordion-header">
        {title}
        <span>{isExpanded ? '▼' : '▶'}</span>
      </button>
      {isExpanded && (
        <div className="accordion-content">{children}</div>
      )}
    </div>
  );
}

// Controlled: Expand items based on search
function SearchableAccordion() {
  const [search, setSearch] = useState('');
  const [expandedItems, setExpandedItems] = useState([]);

  const items = [
    { id: 'item1', title: 'React Hooks', content: 'useState, useEffect...' },
    { id: 'item2', title: 'React Patterns', content: 'HOCs, Render Props...' },
    { id: 'item3', title: 'React Performance', content: 'Memoization...' },
  ];

  // Auto-expand items matching search
  useEffect(() => {
    if (search) {
      const matching = items
        .filter(
          (item) =>
            item.title.toLowerCase().includes(search.toLowerCase()) ||
            item.content.toLowerCase().includes(search.toLowerCase())
        )
        .map((item) => item.id);
      setExpandedItems(matching);
    } else {
      setExpandedItems([]);
    }
  }, [search]);

  return (
    <div>
      <input
        type="search"
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search..."
      />
      <Accordion
        expandedItems={expandedItems}
        onExpandedChange={setExpandedItems}
        allowMultiple={true}
      >
        {items.map((item) => (
          <AccordionItem
            key={item.id}
            id={item.id}
            title={item.title}
          >
            {item.content}
          </AccordionItem>
        ))}
      </Accordion>
    </div>
  );
}
```

## Real-World Pattern: Controlled Form Field

```javascript
function FormField({
  // Controlled props
  value,
  onChange,
  error,
  // Uncontrolled fallback
  defaultValue = '',
  // Field config
  name,
  label,
  type = 'text',
  required = false,
  disabled = false,
  placeholder = '',
  validate,
}) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const [internalError, setInternalError] = useState(null);

  const isControlled = value !== undefined;
  const currentValue = isControlled ? value : internalValue;
  const currentError = error !== undefined ? error : internalError;

  const handleChange = (e) => {
    const newValue = e.target.value;

    // Validate if function provided
    let validationError = null;
    if (validate) {
      validationError = validate(newValue);
      if (!isControlled) {
        setInternalError(validationError);
      }
    }

    if (!isControlled) {
      setInternalValue(newValue);
    }

    onChange?.(newValue, { error: validationError, name });
  };

  const handleBlur = () => {
    // Validate on blur
    if (validate && !isControlled) {
      const validationError = validate(currentValue);
      setInternalError(validationError);
    }
  };

  return (
    <div className="form-field">
      <label htmlFor={name}>
        {label}
        {required && <span className="required">*</span>}
      </label>
      <input
        id={name}
        name={name}
        type={type}
        value={currentValue}
        onChange={handleChange}
        onBlur={handleBlur}
        disabled={disabled}
        placeholder={placeholder}
        required={required}
        aria-invalid={!!currentError}
        aria-describedby={currentError ? `${name}-error` : undefined}
      />
      {currentError && (
        <span id={`${name}-error`} className="error">
          {currentError}
        </span>
      )}
    </div>
  );
}

// Controlled form with validation
function ControlledForm() {
  const [formData, setFormData] = useState({
    email: '',
    password: '',
  });
  const [errors, setErrors] = useState({});

  const validateEmail = (value) => {
    if (!value) return 'Email is required';
    if (!/\S+@\S+\.\S+/.test(value)) return 'Email is invalid';
    return null;
  };

  const validatePassword = (value) => {
    if (!value) return 'Password is required';
    if (value.length < 8) return 'Password must be at least 8 characters';
    return null;
  };

  const handleFieldChange = (value, { name, error }) => {
    setFormData((prev) => ({ ...prev, [name]: value }));
    setErrors((prev) => ({ ...prev, [name]: error }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const emailError = validateEmail(formData.email);
    const passwordError = validatePassword(formData.password);

    if (!emailError && !passwordError) {
      console.log('Submitting:', formData);
    } else {
      setErrors({ email: emailError, password: passwordError });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <FormField
        name="email"
        label="Email"
        type="email"
        value={formData.email}
        error={errors.email}
        onChange={handleFieldChange}
        validate={validateEmail}
        required
      />
      <FormField
        name="password"
        label="Password"
        type="password"
        value={formData.password}
        error={errors.password}
        onChange={handleFieldChange}
        validate={validatePassword}
        required
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Controlled Props with useControlledState Hook

```javascript
function useControlledState(controlledValue, defaultValue, onChange) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;

  const setValue = useCallback(
    (nextValue) => {
      const newValue =
        typeof nextValue === 'function'
          ? nextValue(value)
          : nextValue;

      if (!isControlled) {
        setInternalValue(newValue);
      }

      onChange?.(newValue);
    },
    [isControlled, value, onChange]
  );

  return [value, setValue];
}

// Usage in component
function Counter({ value: controlledValue, onChange, defaultValue = 0 }) {
  const [value, setValue] = useControlledState(
    controlledValue,
    defaultValue,
    onChange
  );

  return (
    <div>
      <button onClick={() => setValue((v) => v - 1)}>-</button>
      <span>{value}</span>
      <button onClick={() => setValue((v) => v + 1)}>+</button>
    </div>
  );
}
```

## Best Practices

### 1. Consistent Prop Naming

```javascript
// Good: Clear naming convention
function Component({
  value,           // Controlled value
  onChange,        // Controlled handler
  defaultValue,    // Uncontrolled initial value
}) {}

// Avoid: Confusing names
function Component({
  val,             // Unclear
  update,          // Not standard
  initialValue,    // Confusing with controlled
}) {}
```

### 2. Warning for Mixed Usage

```javascript
function Counter({ value, defaultValue, onChange }) {
  // Warn if both controlled and default value provided
  useEffect(() => {
    if (value !== undefined && defaultValue !== undefined) {
      console.warn(
        'Counter: You provided both value and defaultValue. ' +
        'Use either value for controlled or defaultValue for uncontrolled.'
      );
    }
  }, []);

  // Implementation...
}
```

### 3. Support Both Modes

```javascript
// Good: Works controlled or uncontrolled
function FlexibleComponent({ value, defaultValue, onChange }) {
  const [state, setState] = useControlledState(
    value,
    defaultValue,
    onChange
  );
  // Component works in both modes
}

// Bad: Only controlled
function ControlledOnly({ value, onChange }) {
  // Requires parent to manage state always
}
```

### 4. Notify on All Changes

```javascript
function Input({ value, onChange, defaultValue }) {
  const [state, setState] = useControlledState(
    value,
    defaultValue,
    onChange
  );

  const handleChange = (e) => {
    const newValue = e.target.value;
    setState(newValue);
    // onChange is called by useControlledState
  };

  // Good: Always call onChange, even in uncontrolled mode
}
```

## Common Mistakes

### 1. Switching Between Modes

```javascript
// Bad: Switching from uncontrolled to controlled
function BadComponent() {
  const [value, setValue] = useState(undefined);

  // Later, set value to controlled
  const handleClick = () => setValue(5);

  return <Counter value={value} onChange={setValue} />;
}

// Good: Choose one mode and stick with it
function GoodComponent() {
  const [value, setValue] = useState(0); // Always controlled

  return <Counter value={value} onChange={setValue} />;
}
```

### 2. Not Handling Undefined

```javascript
// Bad: Doesn't check for undefined
function Counter({ value, onChange }) {
  return <span>{value}</span>; // Breaks if value is undefined
}

// Good: Handle both modes
function Counter({ value, defaultValue = 0, onChange }) {
  const [state, setState] = useControlledState(
    value,
    defaultValue,
    onChange
  );
  return <span>{state}</span>;
}
```

### 3. Forgetting onChange

```javascript
// Bad: Required onChange not provided
<Counter value={count} />

// Good: Always provide onChange with value
<Counter value={count} onChange={setCount} />

// Or: Use uncontrolled mode
<Counter defaultValue={count} onChange={(v) => console.log(v)} />
```

## Performance Considerations

### Memoize Callbacks

```javascript
function Parent() {
  const [value, setValue] = useState(0);

  // Bad: New function on every render
  const handleChange = (newValue) => {
    console.log('Changing to:', newValue);
    setValue(newValue);
  };

  // Good: Memoized callback
  const handleChange = useCallback((newValue) => {
    console.log('Changing to:', newValue);
    setValue(newValue);
  }, []);

  return <Counter value={value} onChange={handleChange} />;
}
```

## Testing

```javascript
import { render, screen, fireEvent } from '@testing-library/react';

describe('Counter - Controlled Props', () => {
  test('works in controlled mode', () => {
    const handleChange = jest.fn();
    const { rerender } = render(
      <Counter value={0} onChange={handleChange} />
    );

    fireEvent.click(screen.getByText('+'));
    expect(handleChange).toHaveBeenCalledWith(1);

    // Parent updates value
    rerender(<Counter value={1} onChange={handleChange} />);
    expect(screen.getByText('1')).toBeInTheDocument();
  });

  test('works in uncontrolled mode', () => {
    const handleChange = jest.fn();
    render(<Counter defaultValue={5} onChange={handleChange} />);

    expect(screen.getByText('5')).toBeInTheDocument();

    fireEvent.click(screen.getByText('+'));
    expect(screen.getByText('6')).toBeInTheDocument();
    expect(handleChange).toHaveBeenCalledWith(6);
  });

  test('warns when both value and defaultValue provided', () => {
    const consoleSpy = jest.spyOn(console, 'warn').mockImplementation();

    render(
      <Counter value={0} defaultValue={5} onChange={() => {}} />
    );

    expect(consoleSpy).toHaveBeenCalledWith(
      expect.stringContaining('both value and defaultValue')
    );

    consoleSpy.mockRestore();
  });
});
```

## Key Takeaways

1. **Flexible Components**: Support both controlled and uncontrolled modes for maximum flexibility
2. **Consistent Naming**: Use value/onChange for controlled, defaultValue for uncontrolled
3. **One Mode**: Don't switch between controlled and uncontrolled during component lifetime
4. **Always Notify**: Call onChange even in uncontrolled mode to notify parent
5. **Type Safety**: Use TypeScript to enforce correct prop combinations
6. **Performance**: Memoize onChange callbacks to prevent unnecessary renders
7. **Testing**: Test both controlled and uncontrolled behaviors

## Additional Resources

- [React Controlled Components](https://react.dev/learn/sharing-state-between-components)
- [Controlled vs Uncontrolled](https://react.dev/learn/sharing-state-between-components#controlled-and-uncontrolled-components)
- [React Hook Form - Controlled Components](https://react-hook-form.com/get-started#IntegratingControlledInputs)
- [Radix UI - Controlled Components](https://www.radix-ui.com/docs/primitives/guides/composition)
