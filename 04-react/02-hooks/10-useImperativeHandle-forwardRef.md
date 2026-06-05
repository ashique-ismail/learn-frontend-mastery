# useImperativeHandle + forwardRef

## The Idea

**In plain English:** Sometimes a parent part of your app needs to directly tell a child component to do something (like "focus yourself" or "play now") without the child asking for instructions first. `forwardRef` lets a child component receive a special "remote control" handle from its parent, and `useImperativeHandle` lets the child decide exactly which buttons that remote control has.

**Real-world analogy:** Imagine a TV remote control and a TV. The TV manufacturer decides which buttons the remote will have — power, volume, channel — but does NOT give you access to the internal wiring. The parent (you) uses the remote to issue commands, but the TV controls exactly what each button does internally.

- The remote control = the `ref` object passed from the parent
- The TV manufacturer choosing which buttons exist = `useImperativeHandle` defining which methods are exposed
- The act of passing the remote to you = `forwardRef` allowing the parent to receive the ref

---

## Overview

`useImperativeHandle` and `forwardRef` are advanced React APIs that customize the instance value exposed to parent components when using refs. They're essential for building reusable component libraries where you need fine-grained control over what imperative methods/values are exposed to consumers.

## Why These APIs Exist

### The Ref Forwarding Problem

By default, function components don't accept refs like DOM elements do:

```jsx
// This DOESN'T work
function Input(props) {
  return <input {...props} />;
}

function Parent() {
  const inputRef = useRef();
  // Error: Function components cannot be given refs
  return <Input ref={inputRef} />;
}
```

### The Solution: forwardRef

`forwardRef` allows function components to accept refs and forward them:

```jsx
const Input = forwardRef((props, ref) => {
  return <input ref={ref} {...props} />;
});

function Parent() {
  const inputRef = useRef();
  // Now this works!
  return <Input ref={inputRef} />;
}
```

## forwardRef Deep Dive

### Basic Syntax

```jsx
const MyComponent = forwardRef((props, ref) => {
  // ref is the second parameter
  return <div ref={ref}>Content</div>;
});
```

### Forwarding to DOM Elements

```jsx
const FancyButton = forwardRef((props, ref) => {
  return (
    <button ref={ref} className="fancy-button">
      {props.children}
    </button>
  );
});

// Usage
function App() {
  const buttonRef = useRef();
  
  useEffect(() => {
    buttonRef.current.focus(); // Direct DOM access
  }, []);
  
  return <FancyButton ref={buttonRef}>Click me</FancyButton>;
}
```

### With TypeScript

```tsx
interface InputProps {
  label: string;
  placeholder?: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, ...props }, ref) => {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} {...props} />
      </div>
    );
  }
);

// Usage with full type safety
function App() {
  const inputRef = useRef<HTMLInputElement>(null);
  
  return <Input ref={inputRef} label="Name" placeholder="Enter name" />;
}
```

## useImperativeHandle Deep Dive

### The Problem with Raw Refs

When you forward a ref to a DOM element, the parent gets access to ALL DOM methods:

```jsx
const Input = forwardRef((props, ref) => {
  return <input ref={ref} />;
});

// Parent has access to everything
function Parent() {
  const ref = useRef();
  
  ref.current.focus();         // ✓ Intended
  ref.current.blur();          // ✓ Intended
  ref.current.value = 'hack';  // ✗ Bypasses React
  ref.current.remove();        // ✗ Dangerous!
}
```

### The Solution: Custom Interface

`useImperativeHandle` lets you expose only specific methods:

```jsx
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  
  useImperativeHandle(ref, () => ({
    // Only expose these methods
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ''; }
  }));
  
  return <input ref={inputRef} />;
});

// Parent can only use exposed methods
function Parent() {
  const ref = useRef();
  
  ref.current.focus();  // ✓ Works
  ref.current.clear();  // ✓ Works
  ref.current.value;    // ✗ undefined - not exposed
}
```

### Syntax

```jsx
useImperativeHandle(ref, createHandle, dependencies?)
```

- **ref**: The ref forwarded from parent
- **createHandle**: Function returning the handle object
- **dependencies**: Optional dependency array (like useEffect)

## Practical Examples

### Example 1: Controlled Focus Management

```jsx
const TextInput = forwardRef(({ label, onEnter }, ref) => {
  const inputRef = useRef();
  const [value, setValue] = useState('');
  
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    blur: () => inputRef.current.blur(),
    clear: () => setValue(''),
    getValue: () => value,
    setValue: (newValue) => setValue(newValue)
  }));
  
  const handleKeyDown = (e) => {
    if (e.key === 'Enter' && onEnter) {
      onEnter(value);
    }
  };
  
  return (
    <div>
      <label>{label}</label>
      <input
        ref={inputRef}
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onKeyDown={handleKeyDown}
      />
    </div>
  );
});

// Usage
function Form() {
  const nameRef = useRef();
  const emailRef = useRef();
  
  const handleSubmit = () => {
    console.log('Name:', nameRef.current.getValue());
    console.log('Email:', emailRef.current.getValue());
    
    nameRef.current.clear();
    emailRef.current.clear();
  };
  
  const handleNameEnter = () => {
    emailRef.current.focus(); // Move to next field
  };
  
  return (
    <>
      <TextInput ref={nameRef} label="Name" onEnter={handleNameEnter} />
      <TextInput ref={emailRef} label="Email" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

### Example 2: Video Player Control

```jsx
const VideoPlayer = forwardRef(({ src }, ref) => {
  const videoRef = useRef();
  const [isPlaying, setIsPlaying] = useState(false);
  
  useImperativeHandle(ref, () => ({
    play: () => {
      videoRef.current.play();
      setIsPlaying(true);
    },
    pause: () => {
      videoRef.current.pause();
      setIsPlaying(false);
    },
    seek: (time) => {
      videoRef.current.currentTime = time;
    },
    getCurrentTime: () => videoRef.current.currentTime,
    getDuration: () => videoRef.current.duration,
    isPlaying: () => isPlaying
  }));
  
  return (
    <video
      ref={videoRef}
      src={src}
      onPlay={() => setIsPlaying(true)}
      onPause={() => setIsPlaying(false)}
    />
  );
});

// Usage
function VideoControls() {
  const playerRef = useRef();
  const [currentTime, setCurrentTime] = useState(0);
  
  const handlePlayPause = () => {
    if (playerRef.current.isPlaying()) {
      playerRef.current.pause();
    } else {
      playerRef.current.play();
    }
  };
  
  const handleSeek = (seconds) => {
    const current = playerRef.current.getCurrentTime();
    playerRef.current.seek(current + seconds);
  };
  
  return (
    <>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={handlePlayPause}>Play/Pause</button>
      <button onClick={() => handleSeek(-10)}>-10s</button>
      <button onClick={() => handleSeek(10)}>+10s</button>
    </>
  );
}
```

### Example 3: Modal with Imperative API

```jsx
const Modal = forwardRef(({ children, title }, ref) => {
  const [isOpen, setIsOpen] = useState(false);
  const dialogRef = useRef();
  
  useImperativeHandle(ref, () => ({
    open: () => {
      setIsOpen(true);
      // Focus trap, lock scroll, etc.
    },
    close: () => {
      setIsOpen(false);
    },
    isOpen: () => isOpen
  }));
  
  if (!isOpen) return null;
  
  return createPortal(
    <div className="modal-overlay" onClick={() => setIsOpen(false)}>
      <div 
        ref={dialogRef}
        className="modal-content"
        onClick={(e) => e.stopPropagation()}
      >
        <h2>{title}</h2>
        {children}
      </div>
    </div>,
    document.body
  );
});

// Usage
function App() {
  const confirmModalRef = useRef();
  const alertModalRef = useRef();
  
  const handleDelete = () => {
    confirmModalRef.current.open();
  };
  
  const confirmDelete = () => {
    // Delete logic
    confirmModalRef.current.close();
    alertModalRef.current.open();
  };
  
  return (
    <>
      <button onClick={handleDelete}>Delete</button>
      
      <Modal ref={confirmModalRef} title="Confirm">
        <p>Are you sure?</p>
        <button onClick={confirmDelete}>Yes</button>
        <button onClick={() => confirmModalRef.current.close()}>No</button>
      </Modal>
      
      <Modal ref={alertModalRef} title="Success">
        <p>Item deleted!</p>
      </Modal>
    </>
  );
}
```

### Example 4: Complex Form Field

```jsx
const DateRangePicker = forwardRef((props, ref) => {
  const [startDate, setStartDate] = useState(null);
  const [endDate, setEndDate] = useState(null);
  const [error, setError] = useState(null);
  
  useImperativeHandle(ref, () => ({
    getValue: () => ({ startDate, endDate }),
    setValue: ({ startDate: start, endDate: end }) => {
      setStartDate(start);
      setEndDate(end);
    },
    validate: () => {
      if (!startDate || !endDate) {
        setError('Both dates required');
        return false;
      }
      if (startDate > endDate) {
        setError('Start date must be before end date');
        return false;
      }
      setError(null);
      return true;
    },
    reset: () => {
      setStartDate(null);
      setEndDate(null);
      setError(null);
    },
    getError: () => error
  }));
  
  return (
    <div>
      <input
        type="date"
        value={startDate || ''}
        onChange={(e) => setStartDate(e.target.value)}
      />
      <input
        type="date"
        value={endDate || ''}
        onChange={(e) => setEndDate(e.target.value)}
      />
      {error && <span className="error">{error}</span>}
    </div>
  );
});

// Usage
function BookingForm() {
  const dateRangeRef = useRef();
  
  const handleSubmit = () => {
    if (!dateRangeRef.current.validate()) {
      return;
    }
    
    const { startDate, endDate } = dateRangeRef.current.getValue();
    // Submit booking
  };
  
  const handleReset = () => {
    dateRangeRef.current.reset();
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <DateRangePicker ref={dateRangeRef} />
      <button type="submit">Book</button>
      <button type="button" onClick={handleReset}>Reset</button>
    </form>
  );
}
```

## TypeScript Best Practices

### Defining Handle Type

```tsx
interface VideoPlayerHandle {
  play: () => void;
  pause: () => void;
  seek: (time: number) => void;
  getCurrentTime: () => number;
  getDuration: () => number;
}

interface VideoPlayerProps {
  src: string;
  autoPlay?: boolean;
}

const VideoPlayer = forwardRef<VideoPlayerHandle, VideoPlayerProps>(
  ({ src, autoPlay }, ref) => {
    const videoRef = useRef<HTMLVideoElement>(null);
    
    useImperativeHandle(ref, () => ({
      play: () => videoRef.current?.play(),
      pause: () => videoRef.current?.pause(),
      seek: (time) => {
        if (videoRef.current) {
          videoRef.current.currentTime = time;
        }
      },
      getCurrentTime: () => videoRef.current?.currentTime ?? 0,
      getDuration: () => videoRef.current?.duration ?? 0
    }));
    
    return <video ref={videoRef} src={src} autoPlay={autoPlay} />;
  }
);

// Usage with type safety
function App() {
  const playerRef = useRef<VideoPlayerHandle>(null);
  
  const handlePlay = () => {
    playerRef.current?.play(); // Type-safe!
  };
  
  return <VideoPlayer ref={playerRef} src="/video.mp4" />;
}
```

### Generic Forwarded Component

```tsx
interface ListHandle<T> {
  scrollToItem: (index: number) => void;
  getSelectedItems: () => T[];
  clearSelection: () => void;
}

interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
}

function ListInner<T>(
  { items, renderItem }: ListProps<T>,
  ref: ForwardedRef<ListHandle<T>>
) {
  const [selectedItems, setSelectedItems] = useState<T[]>([]);
  const listRef = useRef<HTMLDivElement>(null);
  
  useImperativeHandle(ref, () => ({
    scrollToItem: (index) => {
      // Scroll logic
    },
    getSelectedItems: () => selectedItems,
    clearSelection: () => setSelectedItems([])
  }));
  
  return (
    <div ref={listRef}>
      {items.map(renderItem)}
    </div>
  );
}

const List = forwardRef(ListInner) as <T>(
  props: ListProps<T> & { ref?: ForwardedRef<ListHandle<T>> }
) => ReactElement;
```

## Dependency Array

### When to Use Dependencies

```jsx
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  const [value, setValue] = useState('');
  
  // Bad: No dependencies - handle recreated every render
  useImperativeHandle(ref, () => ({
    getValue: () => value
  }));
  
  // Good: Include dependencies for stable identity
  useImperativeHandle(ref, () => ({
    getValue: () => value
  }), [value]);
  
  // Better: For simple cases, omit value from closure
  useImperativeHandle(ref, () => ({
    getValue: () => inputRef.current.value
  }), []); // No dependencies needed
  
  return (
    <input
      ref={inputRef}
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
});
```

## Common Patterns

### Pattern 1: Focus Management Chain

```jsx
const FormField = forwardRef(({ label, nextRef, ...props }, ref) => {
  const inputRef = useRef();
  
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    focusNext: () => nextRef?.current?.focus()
  }));
  
  const handleKeyDown = (e) => {
    if (e.key === 'Enter') {
      nextRef?.current?.focus();
    }
  };
  
  return (
    <div>
      <label>{label}</label>
      <input
        ref={inputRef}
        onKeyDown={handleKeyDown}
        {...props}
      />
    </div>
  );
});

function MultiFieldForm() {
  const field1Ref = useRef();
  const field2Ref = useRef();
  const field3Ref = useRef();
  
  return (
    <>
      <FormField ref={field1Ref} nextRef={field2Ref} label="Field 1" />
      <FormField ref={field2Ref} nextRef={field3Ref} label="Field 2" />
      <FormField ref={field3Ref} label="Field 3" />
    </>
  );
}
```

### Pattern 2: Validation API

```jsx
const ValidatedInput = forwardRef(({ validation, ...props }, ref) => {
  const inputRef = useRef();
  const [error, setError] = useState(null);
  
  useImperativeHandle(ref, () => ({
    validate: () => {
      const value = inputRef.current.value;
      const error = validation(value);
      setError(error);
      return !error;
    },
    reset: () => {
      inputRef.current.value = '';
      setError(null);
    },
    focus: () => inputRef.current.focus()
  }));
  
  return (
    <div>
      <input ref={inputRef} {...props} />
      {error && <span className="error">{error}</span>}
    </div>
  );
});

function Form() {
  const emailRef = useRef();
  const passwordRef = useRef();
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    const isEmailValid = emailRef.current.validate();
    const isPasswordValid = passwordRef.current.validate();
    
    if (!isEmailValid) {
      emailRef.current.focus();
      return;
    }
    
    if (!isPasswordValid) {
      passwordRef.current.focus();
      return;
    }
    
    // Submit
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <ValidatedInput
        ref={emailRef}
        validation={(v) => !v.includes('@') ? 'Invalid email' : null}
      />
      <ValidatedInput
        ref={passwordRef}
        type="password"
        validation={(v) => v.length < 8 ? 'Too short' : null}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Pattern 3: Composite Component

```jsx
const Tabs = forwardRef(({ children }, ref) => {
  const [activeIndex, setActiveIndex] = useState(0);
  const tabRefs = useRef([]);
  
  useImperativeHandle(ref, () => ({
    selectTab: (index) => setActiveIndex(index),
    nextTab: () => setActiveIndex((i) => (i + 1) % children.length),
    prevTab: () => setActiveIndex((i) => (i - 1 + children.length) % children.length),
    getActiveIndex: () => activeIndex
  }));
  
  return (
    <div>
      <div role="tablist">
        {React.Children.map(children, (child, index) => (
          <button
            role="tab"
            aria-selected={index === activeIndex}
            onClick={() => setActiveIndex(index)}
          >
            {child.props.label}
          </button>
        ))}
      </div>
      <div role="tabpanel">
        {children[activeIndex]}
      </div>
    </div>
  );
});

// Usage
function App() {
  const tabsRef = useRef();
  
  useEffect(() => {
    const handleKeyDown = (e) => {
      if (e.key === 'ArrowRight') {
        tabsRef.current.nextTab();
      } else if (e.key === 'ArrowLeft') {
        tabsRef.current.prevTab();
      }
    };
    
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []);
  
  return (
    <Tabs ref={tabsRef}>
      <div label="Tab 1">Content 1</div>
      <div label="Tab 2">Content 2</div>
      <div label="Tab 3">Content 3</div>
    </Tabs>
  );
}
```

## Common Mistakes

### Mistake 1: Overusing Imperative Handles

```jsx
// Bad: Everything is imperative
const Form = forwardRef((props, ref) => {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  
  useImperativeHandle(ref, () => ({
    getName: () => name,
    setName,
    getEmail: () => email,
    setEmail,
    submit: () => { /* ... */ }
  }));
  
  return (/* JSX */);
});

// Good: Keep it declarative
function Form({ onSubmit }) {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  
  const handleSubmit = () => {
    onSubmit({ name, email });
  };
  
  return (/* JSX */);
}
```

### Mistake 2: Exposing Internal State

```jsx
// Bad: Exposing setState directly
const Counter = forwardRef((props, ref) => {
  const [count, setCount] = useState(0);
  
  useImperativeHandle(ref, () => ({
    count,
    setCount  // Dangerous! Parent can break component
  }));
  
  return <div>{count}</div>;
});

// Good: Expose controlled operations
const Counter = forwardRef((props, ref) => {
  const [count, setCount] = useState(0);
  
  useImperativeHandle(ref, () => ({
    increment: () => setCount(c => c + 1),
    decrement: () => setCount(c => c - 1),
    reset: () => setCount(0),
    getCount: () => count
  }));
  
  return <div>{count}</div>;
});
```

### Mistake 3: Not Using Callback Refs

```jsx
// Problem: Can't imperatively control nested components
function Parent() {
  const modalRef = useRef();
  
  return (
    <Modal ref={modalRef}>
      <ComplexForm /> {/* Can't access this from Parent */}
    </Modal>
  );
}

// Solution: Expose multiple handles
const Modal = forwardRef(({ children }, ref) => {
  const contentRef = useRef();
  
  useImperativeHandle(ref, () => ({
    open: () => { /* ... */ },
    close: () => { /* ... */ },
    getContentHandle: () => contentRef.current
  }));
  
  return (
    <div>
      {React.cloneElement(children, { ref: contentRef })}
    </div>
  );
});
```

## When to Use

### Use Cases (Good)

1. **Focus management** - Programmatic focus control
2. **Media controls** - Video/audio playback
3. **Animations** - Triggering imperative animations
4. **Third-party integration** - Wrapping non-React libraries
5. **Form validation** - Complex multi-step validation
6. **Scrolling/virtualization** - Programmatic scroll control

### Anti-Patterns (Bad)

1. **Data flow** - Use props/state instead
2. **Communication** - Use callbacks/context instead
3. **State sync** - Use controlled components instead
4. **Event handling** - Use regular event props

## Performance Considerations

### Memoization

```jsx
const ExpensiveComponent = forwardRef((props, ref) => {
  const internalRef = useRef();
  
  // Memoize handle object to prevent parent re-renders
  const handle = useMemo(() => ({
    doSomething: () => {
      // Expensive operation
    }
  }), []); // Empty deps = stable identity
  
  useImperativeHandle(ref, () => handle, [handle]);
  
  return <div ref={internalRef}>Content</div>;
});
```

### Lazy Initialization

```jsx
const Chart = forwardRef((props, ref) => {
  const chartInstanceRef = useRef(null);
  
  useImperativeHandle(ref, () => ({
    export: () => {
      if (!chartInstanceRef.current) {
        // Lazy init
        chartInstanceRef.current = createChartInstance();
      }
      return chartInstanceRef.current.export();
    }
  }));
  
  return <canvas />;
});
```

## Testing

```jsx
import { render } from '@testing-library/react';
import { createRef } from 'react';

test('exposes imperative methods', () => {
  const ref = createRef();
  
  render(<VideoPlayer ref={ref} src="/video.mp4" />);
  
  // Test exposed methods
  expect(typeof ref.current.play).toBe('function');
  expect(typeof ref.current.pause).toBe('function');
  
  // Test method behavior
  ref.current.seek(30);
  expect(ref.current.getCurrentTime()).toBe(30);
});

test('validates form field', () => {
  const ref = createRef();
  
  render(
    <ValidatedInput
      ref={ref}
      validation={(v) => !v ? 'Required' : null}
    />
  );
  
  expect(ref.current.validate()).toBe(false);
  
  // Simulate user input
  ref.current.setValue('test');
  expect(ref.current.validate()).toBe(true);
});
```

## Migration Path

### From Class Components

```jsx
// Old: Class with ref
class Input extends React.Component {
  focus() {
    this.input.focus();
  }
  
  render() {
    return <input ref={el => this.input = el} />;
  }
}

// New: Function with forwardRef + useImperativeHandle
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus()
  }));
  
  return <input ref={inputRef} />;
});
```

## Resources

- [React Docs: forwardRef](https://react.dev/reference/react/forwardRef)
- [React Docs: useImperativeHandle](https://react.dev/reference/react/useImperativeHandle)
- [When to use refs](https://react.dev/learn/manipulating-the-dom-with-refs)
- [TypeScript with forwardRef](https://fettblog.eu/typescript-react-generic-forward-refs/)
