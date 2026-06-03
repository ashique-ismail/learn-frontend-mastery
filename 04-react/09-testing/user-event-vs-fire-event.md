# userEvent vs fireEvent

## Overview

React Testing Library provides two ways to simulate user interactions: `userEvent` and `fireEvent`. While `fireEvent` directly dispatches DOM events, `userEvent` simulates complete user interactions with all their side effects, making tests more realistic and reliable.

## Table of Contents

1. [Fundamental Differences](#fundamental-differences)
2. [userEvent API](#userevent-api)
3. [fireEvent API](#fireevent-api)
4. [When to Use Each](#when-to-use-each)
5. [Testing Keyboard Navigation](#testing-keyboard-navigation)
6. [Testing Form Interactions](#testing-form-interactions)
7. [Testing Hover States](#testing-hover-states)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

## Fundamental Differences

### fireEvent (Low-Level)

```typescript
import { render, screen, fireEvent } from '@testing-library/react';

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}

test('fireEvent example', () => {
  render(<Counter />);
  
  const button = screen.getByRole('button');
  
  // Fires ONLY the click event
  // No mousedown, mouseup, focus, etc.
  fireEvent.click(button);
  
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### userEvent (High-Level)

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('userEvent example', async () => {
  const user = userEvent.setup();
  render(<Counter />);
  
  const button = screen.getByRole('button');
  
  // Simulates real user interaction:
  // 1. mousedown
  // 2. focus
  // 3. mouseup
  // 4. click
  await user.click(button);
  
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### Side Effects Comparison

```typescript
function InputWithSideEffects() {
  const [focused, setFocused] = useState(false);
  const [hovered, setHovered] = useState(false);
  
  return (
    <div>
      <input
        type="text"
        onFocus={() => setFocused(true)}
        onBlur={() => setFocused(false)}
        onMouseEnter={() => setHovered(true)}
        onMouseLeave={() => setHovered(false)}
      />
      {focused && <span>Focused</span>}
      {hovered && <span>Hovered</span>}
    </div>
  );
}

test('fireEvent - misses side effects', () => {
  render(<InputWithSideEffects />);
  const input = screen.getByRole('textbox');
  
  // Only fires focus event
  fireEvent.focus(input);
  
  expect(screen.getByText('Focused')).toBeInTheDocument();
  // Hover state NOT triggered
  expect(screen.queryByText('Hovered')).not.toBeInTheDocument();
});

test('userEvent - captures all side effects', async () => {
  const user = userEvent.setup();
  render(<InputWithSideEffects />);
  const input = screen.getByRole('textbox');
  
  // Simulates real click: hover + focus
  await user.click(input);
  
  expect(screen.getByText('Focused')).toBeInTheDocument();
  expect(screen.getByText('Hovered')).toBeInTheDocument();
});
```

## userEvent API

### userEvent.setup()

```typescript
import userEvent from '@testing-library/user-event';

test('userEvent setup', async () => {
  // Always call setup() for stateful interactions
  const user = userEvent.setup();
  
  render(<MyComponent />);
  
  // Use the user instance
  await user.click(screen.getByRole('button'));
  await user.type(screen.getByRole('textbox'), 'Hello');
});

test('multiple users', async () => {
  // Simulate multiple users
  const user1 = userEvent.setup();
  const user2 = userEvent.setup();
  
  render(<ChatApp />);
  
  const input1 = screen.getByTestId('user1-input');
  const input2 = screen.getByTestId('user2-input');
  
  // Independent interactions
  await user1.type(input1, 'Hello from user 1');
  await user2.type(input2, 'Hello from user 2');
});
```

### userEvent.click()

```typescript
import userEvent from '@testing-library/user-event';

test('single click', async () => {
  const handleClick = jest.fn();
  const user = userEvent.setup();
  
  render(<button onClick={handleClick}>Click me</button>);
  
  await user.click(screen.getByRole('button'));
  
  expect(handleClick).toHaveBeenCalledTimes(1);
});

test('double click', async () => {
  const handleDoubleClick = jest.fn();
  const user = userEvent.setup();
  
  render(<button onDoubleClick={handleDoubleClick}>Double click</button>);
  
  await user.dblClick(screen.getByRole('button'));
  
  expect(handleDoubleClick).toHaveBeenCalledTimes(1);
});

test('triple click (select all text)', async () => {
  const user = userEvent.setup();
  
  render(<input type="text" defaultValue="Select all this text" />);
  
  const input = screen.getByRole('textbox');
  
  await user.tripleClick(input);
  
  // Text should be selected
  expect(input).toHaveProperty('selectionStart', 0);
  expect(input).toHaveProperty('selectionEnd', 20);
});
```

### userEvent.type()

```typescript
import userEvent from '@testing-library/user-event';

test('type text', async () => {
  const user = userEvent.setup();
  
  render(<input type="text" />);
  
  const input = screen.getByRole('textbox');
  
  await user.type(input, 'Hello World');
  
  expect(input).toHaveValue('Hello World');
});

test('type with delay', async () => {
  const user = userEvent.setup({ delay: 100 }); // 100ms between keystrokes
  
  render(<input type="text" />);
  
  const startTime = Date.now();
  await user.type(screen.getByRole('textbox'), 'Hi');
  const duration = Date.now() - startTime;
  
  // Should take ~200ms (2 chars * 100ms)
  expect(duration).toBeGreaterThanOrEqual(200);
});

test('type with special keys', async () => {
  const user = userEvent.setup();
  
  render(<input type="text" />);
  
  const input = screen.getByRole('textbox');
  
  // Special keys
  await user.type(input, 'Hello{Enter}World');
  await user.type(input, '{Backspace}');
  await user.type(input, '{selectall}{delete}');
  await user.type(input, '{Control>}a{/Control}'); // Ctrl+A
});

test('type triggers events', async () => {
  const handleChange = jest.fn();
  const handleKeyDown = jest.fn();
  const handleKeyUp = jest.fn();
  const user = userEvent.setup();
  
  render(
    <input
      type="text"
      onChange={handleChange}
      onKeyDown={handleKeyDown}
      onKeyUp={handleKeyUp}
    />
  );
  
  await user.type(screen.getByRole('textbox'), 'Hi');
  
  // Each keystroke triggers keydown, keypress, keyup, input, change
  expect(handleKeyDown).toHaveBeenCalledTimes(2);
  expect(handleKeyUp).toHaveBeenCalledTimes(2);
  expect(handleChange).toHaveBeenCalledTimes(2);
});
```

### userEvent.clear()

```typescript
test('clear input', async () => {
  const user = userEvent.setup();
  
  render(<input type="text" defaultValue="Remove this" />);
  
  const input = screen.getByRole('textbox');
  
  await user.clear(input);
  
  expect(input).toHaveValue('');
});
```

### userEvent.keyboard()

```typescript
test('keyboard sequences', async () => {
  const user = userEvent.setup();
  
  render(<input type="text" />);
  
  const input = screen.getByRole('textbox');
  
  // Complex keyboard sequences
  await user.keyboard('Hello');
  expect(input).toHaveValue('Hello');
  
  // Hold shift and type
  await user.keyboard('{Shift>}world{/Shift}');
  expect(input).toHaveValue('HelloWORLD');
  
  // Select all and delete
  await user.keyboard('{Control>}a{/Control}');
  await user.keyboard('{Delete}');
  expect(input).toHaveValue('');
});
```

### userEvent.selectOptions()

```typescript
test('select dropdown option', async () => {
  const user = userEvent.setup();
  
  render(
    <select>
      <option value="">Select...</option>
      <option value="1">Option 1</option>
      <option value="2">Option 2</option>
      <option value="3">Option 3</option>
    </select>
  );
  
  const select = screen.getByRole('combobox');
  
  await user.selectOptions(select, '2');
  
  expect(select).toHaveValue('2');
});

test('select multiple options', async () => {
  const user = userEvent.setup();
  
  render(
    <select multiple>
      <option value="1">Option 1</option>
      <option value="2">Option 2</option>
      <option value="3">Option 3</option>
    </select>
  );
  
  const select = screen.getByRole('listbox');
  
  await user.selectOptions(select, ['1', '3']);
  
  expect(screen.getByRole('option', { name: 'Option 1' })).toHaveAttribute('selected');
  expect(screen.getByRole('option', { name: 'Option 3' })).toHaveAttribute('selected');
});
```

### userEvent.upload()

```typescript
test('file upload', async () => {
  const user = userEvent.setup();
  const handleChange = jest.fn();
  
  render(<input type="file" onChange={handleChange} />);
  
  const input = screen.getByRole('textbox', { hidden: true }); // file inputs are hidden
  
  const file = new File(['hello'], 'hello.txt', { type: 'text/plain' });
  
  await user.upload(input, file);
  
  expect(handleChange).toHaveBeenCalled();
  expect(input.files).toHaveLength(1);
  expect(input.files?.[0]).toBe(file);
});

test('upload multiple files', async () => {
  const user = userEvent.setup();
  
  render(<input type="file" multiple />);
  
  const input = screen.getByRole('textbox', { hidden: true });
  
  const files = [
    new File(['file1'], 'file1.txt', { type: 'text/plain' }),
    new File(['file2'], 'file2.txt', { type: 'text/plain' }),
  ];
  
  await user.upload(input, files);
  
  expect(input.files).toHaveLength(2);
});
```

### userEvent.hover() and userEvent.unhover()

```typescript
function TooltipButton() {
  const [showTooltip, setShowTooltip] = useState(false);
  
  return (
    <div>
      <button
        onMouseEnter={() => setShowTooltip(true)}
        onMouseLeave={() => setShowTooltip(false)}
      >
        Hover me
      </button>
      {showTooltip && <div role="tooltip">Helpful tooltip</div>}
    </div>
  );
}

test('hover shows tooltip', async () => {
  const user = userEvent.setup();
  render(<TooltipButton />);
  
  const button = screen.getByRole('button');
  
  // Tooltip not visible initially
  expect(screen.queryByRole('tooltip')).not.toBeInTheDocument();
  
  // Hover over button
  await user.hover(button);
  
  expect(screen.getByRole('tooltip')).toBeInTheDocument();
  
  // Unhover
  await user.unhover(button);
  
  expect(screen.queryByRole('tooltip')).not.toBeInTheDocument();
});
```

## fireEvent API

### Basic fireEvent

```typescript
import { render, screen, fireEvent } from '@testing-library/react';

test('fireEvent click', () => {
  const handleClick = jest.fn();
  
  render(<button onClick={handleClick}>Click</button>);
  
  fireEvent.click(screen.getByRole('button'));
  
  expect(handleClick).toHaveBeenCalled();
});

test('fireEvent change', () => {
  const handleChange = jest.fn();
  
  render(<input onChange={handleChange} />);
  
  const input = screen.getByRole('textbox');
  
  fireEvent.change(input, { target: { value: 'Hello' } });
  
  expect(handleChange).toHaveBeenCalled();
  expect(input).toHaveValue('Hello');
});
```

### Custom Events

```typescript
test('fireEvent with custom event data', () => {
  const handleKeyDown = jest.fn();
  
  render(<input onKeyDown={handleKeyDown} />);
  
  const input = screen.getByRole('textbox');
  
  fireEvent.keyDown(input, { key: 'Enter', code: 'Enter', keyCode: 13 });
  
  expect(handleKeyDown).toHaveBeenCalledWith(
    expect.objectContaining({
      key: 'Enter',
      code: 'Enter',
      keyCode: 13,
    })
  );
});
```

## When to Use Each

### Use userEvent (Default)

```typescript
// ✅ ALWAYS prefer userEvent
import userEvent from '@testing-library/user-event';

test('form submission', async () => {
  const user = userEvent.setup();
  const handleSubmit = jest.fn();
  
  render(
    <form onSubmit={handleSubmit}>
      <input type="text" name="username" />
      <button type="submit">Submit</button>
    </form>
  );
  
  // Real user interaction
  await user.type(screen.getByRole('textbox'), 'john');
  await user.click(screen.getByRole('button'));
  
  expect(handleSubmit).toHaveBeenCalled();
});
```

### Use fireEvent (Special Cases)

```typescript
// Use fireEvent only when:
// 1. Testing low-level events not supported by userEvent
// 2. Need to trigger events that don't happen through normal user interaction
// 3. Performance-critical tests (fireEvent is faster)

test('scroll event', () => {
  const handleScroll = jest.fn();
  
  render(<div onScroll={handleScroll} style={{ height: 100, overflow: 'auto' }} />);
  
  const scrollable = screen.getByRole('generic');
  
  // userEvent doesn't support scroll
  fireEvent.scroll(scrollable, { target: { scrollY: 100 } });
  
  expect(handleScroll).toHaveBeenCalled();
});

test('focus without user interaction', () => {
  render(<input autoFocus />);
  
  const input = screen.getByRole('textbox');
  
  // Testing programmatic focus, not user focus
  fireEvent.focus(input);
  
  expect(input).toHaveFocus();
});
```

## Testing Keyboard Navigation

### Tab Navigation

```typescript
function Form() {
  return (
    <form>
      <input type="text" placeholder="First" />
      <input type="text" placeholder="Second" />
      <button type="submit">Submit</button>
    </form>
  );
}

test('tab navigation', async () => {
  const user = userEvent.setup();
  render(<Form />);
  
  const firstInput = screen.getByPlaceholderText('First');
  const secondInput = screen.getByPlaceholderText('Second');
  const button = screen.getByRole('button');
  
  // Start at first input
  firstInput.focus();
  expect(firstInput).toHaveFocus();
  
  // Tab to second input
  await user.tab();
  expect(secondInput).toHaveFocus();
  
  // Tab to button
  await user.tab();
  expect(button).toHaveFocus();
  
  // Shift+Tab back
  await user.tab({ shift: true });
  expect(secondInput).toHaveFocus();
});
```

### Arrow Key Navigation

```typescript
function Menu() {
  const [selected, setSelected] = useState(0);
  const items = ['Item 1', 'Item 2', 'Item 3'];
  
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'ArrowDown') {
      setSelected(s => Math.min(s + 1, items.length - 1));
    } else if (e.key === 'ArrowUp') {
      setSelected(s => Math.max(s - 1, 0));
    }
  };
  
  return (
    <ul onKeyDown={handleKeyDown} tabIndex={0}>
      {items.map((item, index) => (
        <li key={item} data-selected={selected === index}>
          {item}
        </li>
      ))}
    </ul>
  );
}

test('arrow key navigation', async () => {
  const user = userEvent.setup();
  render(<Menu />);
  
  const menu = screen.getByRole('list');
  menu.focus();
  
  // Arrow down
  await user.keyboard('{ArrowDown}');
  expect(screen.getByText('Item 2')).toHaveAttribute('data-selected', 'true');
  
  // Arrow down again
  await user.keyboard('{ArrowDown}');
  expect(screen.getByText('Item 3')).toHaveAttribute('data-selected', 'true');
  
  // Arrow up
  await user.keyboard('{ArrowUp}');
  expect(screen.getByText('Item 2')).toHaveAttribute('data-selected', 'true');
});
```

### Enter and Escape Keys

```typescript
function Modal({ onClose }: { onClose: () => void }) {
  return (
    <div role="dialog" onKeyDown={(e) => e.key === 'Escape' && onClose()}>
      <button onClick={onClose}>Close</button>
    </div>
  );
}

test('escape key closes modal', async () => {
  const user = userEvent.setup();
  const handleClose = jest.fn();
  
  render(<Modal onClose={handleClose} />);
  
  const dialog = screen.getByRole('dialog');
  dialog.focus();
  
  await user.keyboard('{Escape}');
  
  expect(handleClose).toHaveBeenCalled();
});

test('enter key submits form', async () => {
  const user = userEvent.setup();
  const handleSubmit = jest.fn(e => e.preventDefault());
  
  render(
    <form onSubmit={handleSubmit}>
      <input type="text" />
    </form>
  );
  
  const input = screen.getByRole('textbox');
  
  await user.type(input, 'text{Enter}');
  
  expect(handleSubmit).toHaveBeenCalled();
});
```

## Testing Form Interactions

### Complete Form Flow

```typescript
function LoginForm({ onSubmit }: { onSubmit: (data: any) => void }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSubmit({ email, password });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      
      <label htmlFor="password">Password</label>
      <input
        id="password"
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      
      <button type="submit">Login</button>
    </form>
  );
}

test('complete form submission', async () => {
  const user = userEvent.setup();
  const handleSubmit = jest.fn();
  
  render(<LoginForm onSubmit={handleSubmit} />);
  
  // Type email
  await user.type(screen.getByLabelText('Email'), 'user@example.com');
  
  // Tab to password
  await user.tab();
  
  // Type password
  await user.type(screen.getByLabelText('Password'), 'password123');
  
  // Submit with Enter key
  await user.keyboard('{Enter}');
  
  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'user@example.com',
    password: 'password123',
  });
});
```

### Checkbox and Radio

```typescript
test('checkbox interaction', async () => {
  const user = userEvent.setup();
  const handleChange = jest.fn();
  
  render(
    <label>
      <input type="checkbox" onChange={handleChange} />
      Accept terms
    </label>
  );
  
  const checkbox = screen.getByRole('checkbox');
  
  await user.click(checkbox);
  expect(checkbox).toBeChecked();
  expect(handleChange).toHaveBeenCalledTimes(1);
  
  await user.click(checkbox);
  expect(checkbox).not.toBeChecked();
  expect(handleChange).toHaveBeenCalledTimes(2);
});

test('radio button interaction', async () => {
  const user = userEvent.setup();
  
  render(
    <fieldset>
      <legend>Choose option</legend>
      <label>
        <input type="radio" name="option" value="1" />
        Option 1
      </label>
      <label>
        <input type="radio" name="option" value="2" />
        Option 2
      </label>
    </fieldset>
  );
  
  const option1 = screen.getByLabelText('Option 1');
  const option2 = screen.getByLabelText('Option 2');
  
  await user.click(option1);
  expect(option1).toBeChecked();
  
  await user.click(option2);
  expect(option2).toBeChecked();
  expect(option1).not.toBeChecked();
});
```

## Testing Hover States

```typescript
function HoverCard() {
  const [isHovered, setIsHovered] = useState(false);
  
  return (
    <div
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
      style={{
        background: isHovered ? 'blue' : 'gray',
      }}
    >
      {isHovered ? 'Hovered' : 'Not hovered'}
    </div>
  );
}

test('hover changes background', async () => {
  const user = userEvent.setup();
  render(<HoverCard />);
  
  const card = screen.getByText('Not hovered');
  
  expect(card).toHaveStyle({ background: 'gray' });
  
  await user.hover(card);
  
  expect(screen.getByText('Hovered')).toHaveStyle({ background: 'blue' });
  
  await user.unhover(card);
  
  expect(screen.getByText('Not hovered')).toHaveStyle({ background: 'gray' });
});
```

## Common Mistakes

### 1. Not Awaiting userEvent

```typescript
// ❌ WRONG: Not awaiting
test('wrong', () => {
  const user = userEvent.setup();
  render(<input />);
  
  user.type(screen.getByRole('textbox'), 'Hello'); // Missing await!
});

// ✅ CORRECT: Await userEvent
test('correct', async () => {
  const user = userEvent.setup();
  render(<input />);
  
  await user.type(screen.getByRole('textbox'), 'Hello');
});
```

### 2. Using fireEvent for Complex Interactions

```typescript
// ❌ WRONG: Missing side effects
test('wrong', () => {
  render(<input />);
  fireEvent.change(screen.getByRole('textbox'), { target: { value: 'Hello' } });
  // Misses focus, blur, keydown, keyup events
});

// ✅ CORRECT: Complete interaction
test('correct', async () => {
  const user = userEvent.setup();
  render(<input />);
  
  await user.type(screen.getByRole('textbox'), 'Hello');
  // Triggers all events a real user would trigger
});
```

### 3. Not Calling userEvent.setup()

```typescript
// ❌ WRONG: Direct method calls (deprecated)
test('wrong', async () => {
  render(<input />);
  await userEvent.type(screen.getByRole('textbox'), 'Hello');
});

// ✅ CORRECT: Use setup()
test('correct', async () => {
  const user = userEvent.setup();
  render(<input />);
  
  await user.type(screen.getByRole('textbox'), 'Hello');
});
```

## Best Practices

### 1. Always Use userEvent

```typescript
// Default to userEvent for all interactions
const user = userEvent.setup();
await user.click(button);
await user.type(input, 'text');
```

### 2. Setup userEvent Once Per Test

```typescript
test('my test', async () => {
  const user = userEvent.setup();
  render(<MyComponent />);
  
  // Use user throughout the test
  await user.click(button1);
  await user.type(input, 'text');
  await user.click(button2);
});
```

### 3. Use fireEvent Only When Necessary

```typescript
// Only use fireEvent for:
// - Events not supported by userEvent (scroll, resize)
// - Testing programmatic events
// - Performance-critical tests
```

## Interview Questions

### Q1: What's the main difference between userEvent and fireEvent?

**Answer:** userEvent simulates complete user interactions with all side effects (hover, focus, keydown, keyup, click), while fireEvent only dispatches a single event. userEvent is more realistic and should be the default choice.

### Q2: Why must you await userEvent calls?

**Answer:** userEvent methods are asynchronous because they simulate realistic timing and event sequences. For example, typing triggers keydown, keypress, input, keyup, and change events in sequence with proper timing.

### Q3: When should you use fireEvent instead of userEvent?

**Answer:** Use fireEvent for events not supported by userEvent (scroll, resize), testing programmatic events (autofocus), or in performance-critical tests where the extra realism isn't needed.

### Q4: What is userEvent.setup()?

**Answer:** userEvent.setup() creates a user instance with stateful interaction tracking. It's required for proper event sequencing and is called once at the beginning of each test.

### Q5: How does userEvent.type() differ from fireEvent.change()?

**Answer:** userEvent.type() simulates each keystroke individually with keydown, keypress, input, keyup, and change events, while fireEvent.change() only triggers a single change event with the final value.

## Key Takeaways

1. Always prefer userEvent over fireEvent for realistic tests
2. userEvent simulates complete interactions with all side effects
3. fireEvent only dispatches single events without side effects
4. Always await userEvent calls (they're async)
5. Call userEvent.setup() once at the start of each test
6. Use fireEvent only for unsupported events or special cases
7. userEvent.type() simulates real typing, fireEvent.change() doesn't
8. userEvent handles focus, hover, and keyboard navigation automatically
9. userEvent.tab() for testing keyboard navigation
10. userEvent is slower but more accurate than fireEvent

## Resources

- [userEvent Documentation](https://testing-library.com/docs/user-event/intro)
- [fireEvent Documentation](https://testing-library.com/docs/dom-testing-library/api-events)
- [userEvent vs fireEvent](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library#not-using-testing-libraryuser-event)
- [Testing Library Best Practices](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
