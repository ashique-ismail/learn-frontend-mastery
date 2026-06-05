# Synthetic Events

## The Idea

**In plain English:** When you click a button on a webpage, the browser creates a "click event" — a signal that says something happened. React wraps that signal in its own package called a synthetic event, which looks and works the same no matter which browser (Chrome, Firefox, Safari) you are using.

**Real-world analogy:** Imagine a large hotel where every guest complaint goes to the front desk first, not directly to the housekeeper, chef, or manager. The front desk translates the complaint into a standard hotel form and routes it to the right department.

- The front desk = React's root event listener (one central point that catches all events)
- The standard hotel form = the synthetic event (a consistent, normalized wrapper)
- The individual departments receiving the form = the React component handlers (onClick, onChange, etc.)

---

## Overview

React's Synthetic Event system is a cross-browser wrapper around native browser events. It provides a consistent API across different browsers, implements event delegation at the root level, and integrates with React's reconciliation and batching mechanisms. Understanding synthetic events reveals how React optimizes event handling and why certain patterns work better than others.

This guide explores event delegation, event pooling (legacy), cross-browser normalization, and the event system's evolution in React 17+.

## Event Delegation

React attaches event listeners at the root level, not on individual elements:

```javascript
// Example 1: Traditional DOM event handling
// TRADITIONAL (without React):
document.getElementById('button1').addEventListener('click', handler1);
document.getElementById('button2').addEventListener('click', handler2);
document.getElementById('button3').addEventListener('click', handler3);
// Three separate event listeners

// REACT (with delegation):
function App() {
  return (
    <div>
      <button onClick={handler1}>Button 1</button>
      <button onClick={handler2}>Button 2</button>
      <button onClick={handler3}>Button 3</button>
    </div>
  );
}

// React attaches ONE listener at root:
// rootElement.addEventListener('click', reactClickHandler);
// Then dispatches to appropriate handler based on target
```

### Event Delegation Implementation

```javascript
// Example 2: Simplified delegation mechanism (React 16 and earlier)
function attachEventToRoot(rootElement) {
  rootElement.addEventListener('click', (nativeEvent) => {
    // Get fiber node from DOM node
    const fiber = getClosestFiber(nativeEvent.target);
    
    // Create synthetic event
    const syntheticEvent = createSyntheticEvent(nativeEvent);
    
    // Collect handlers from fiber tree
    const handlers = [];
    let currentFiber = fiber;
    
    while (currentFiber) {
      if (currentFiber.props && currentFiber.props.onClick) {
        handlers.push({
          fiber: currentFiber,
          handler: currentFiber.props.onClick
        });
      }
      currentFiber = currentFiber.return; // parent
    }
    
    // Execute handlers in capture/bubble order
    executeHandlers(handlers, syntheticEvent);
  });
}

// Example 3: React 17+ delegation change
// React 16 and earlier:
// Event listeners attached to document
// document.addEventListener('click', ...)

// React 17+:
// Event listeners attached to root container
const root = ReactDOM.createRoot(document.getElementById('root'));
// root.container.addEventListener('click', ...)

// Benefits:
// - Multiple React roots can coexist
// - Gradual migration from React 16 → 17
// - Better integration with other libraries
```

## Synthetic Event Structure

```javascript
// Example 4: Synthetic event object
function handleClick(event) {
  console.log(event);
  // SyntheticEvent {
  //   bubbles: true,
  //   cancelable: true,
  //   currentTarget: DOMElement,
  //   target: DOMElement,
  //   type: 'click',
  //   nativeEvent: MouseEvent, // Original browser event
  //   preventDefault: function,
  //   stopPropagation: function,
  //   ...
  // }
}

// Example 5: Cross-browser normalization
function SyntheticEvent(nativeEvent) {
  // Normalize across browsers
  this.nativeEvent = nativeEvent;
  this.type = nativeEvent.type;
  this.target = nativeEvent.target || nativeEvent.srcElement;
  this.currentTarget = nativeEvent.currentTarget;
  
  // Normalize button property (different in IE)
  this.button = normalizeButton(nativeEvent.button);
  
  // Normalize key codes
  this.key = normalizeKey(nativeEvent.key, nativeEvent.keyCode);
  
  // Normalize touch events
  this.touches = normalizeTouches(nativeEvent.touches);
  
  // Add React-specific properties
  this._dispatchListeners = [];
  this._dispatchInstances = [];
}

// Example 6: Accessing native event
function handleMouseMove(event) {
  // Synthetic event (normalized)
  console.log(event.clientX); // Works in all browsers
  
  // Native event (browser-specific)
  const nativeEvent = event.nativeEvent;
  console.log(nativeEvent.mozPressure); // Firefox-specific
  console.log(nativeEvent.webkitForce); // Safari-specific
}
```

## Event Pooling (Legacy - Pre React 17)

In React 16 and earlier, synthetic events were pooled for performance:

```javascript
// Example 7: Event pooling problem (React 16)
function OldBehavior() {
  const handleClick = (event) => {
    console.log(event.type); // 'click'
    
    setTimeout(() => {
      console.log(event.type); // null in React 16!
      // Event has been recycled
    }, 100);
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// Example 8: Persisting events (React 16)
function PersistExample() {
  const handleClick = (event) => {
    event.persist(); // Prevent pooling
    
    setTimeout(() => {
      console.log(event.type); // 'click' (works now)
    }, 100);
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// Example 9: React 17+ (no pooling)
function NewBehavior() {
  const handleClick = (event) => {
    console.log(event.type); // 'click'
    
    setTimeout(() => {
      console.log(event.type); // 'click' (still works!)
      // No pooling, event not recycled
    }, 100);
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// Pooling removed because:
// 1. Modern browsers are fast enough
// 2. Caused confusion and bugs
// 3. Minimal performance impact
```

## Event Phases

React implements both capture and bubble phases:

```javascript
// Example 10: Capture vs bubble phase
function EventPhases() {
  return (
    <div
      onClickCapture={() => console.log('1. Outer capture')}
      onClick={() => console.log('4. Outer bubble')}
    >
      <div
        onClickCapture={() => console.log('2. Middle capture')}
        onClick={() => console.log('3. Middle bubble')}
      >
        <button onClick={() => console.log('Target')}>
          Click Me
        </button>
      </div>
    </div>
  );
}

// Click button output:
// 1. Outer capture (top-down)
// 2. Middle capture (top-down)
// Target (event target)
// 3. Middle bubble (bottom-up)
// 4. Outer bubble (bottom-up)

// Example 11: Stopping propagation
function StopPropagation() {
  return (
    <div onClick={() => console.log('Outer')}>
      <div onClick={(e) => {
        console.log('Middle');
        e.stopPropagation(); // Stops here
      }}>
        <button onClick={() => console.log('Button')}>
          Click
        </button>
      </div>
    </div>
  );
}

// Output:
// Button
// Middle
// (Outer doesn't fire)
```

## Event Handler Patterns

```javascript
// Example 12: Inline handlers
function InlineHandlers() {
  return (
    // Creates new function every render
    <button onClick={() => console.log('Clicked')}>
      Click Me
    </button>
  );
}

// Example 13: Method handlers
function MethodHandlers() {
  const handleClick = () => {
    console.log('Clicked');
  };
  
  // handleClick recreated every render
  return <button onClick={handleClick}>Click Me</button>;
}

// Example 14: Memoized handlers
function MemoizedHandlers() {
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // Stable reference
  
  return <button onClick={handleClick}>Click Me</button>;
}

// Example 15: Passing data to handlers
function DataHandlers() {
  const items = ['a', 'b', 'c'];
  
  // WRONG: Creates new function per item
  return items.map(item => (
    <button key={item} onClick={() => handleClick(item)}>
      {item}
    </button>
  ));
  
  // BETTER: Use data attributes
  return items.map(item => (
    <button
      key={item}
      data-id={item}
      onClick={(e) => handleClick(e.currentTarget.dataset.id)}
    >
      {item}
    </button>
  ));
  
  // BEST: Separate component
  return items.map(item => (
    <ItemButton key={item} id={item} onClick={handleClick} />
  ));
}

const ItemButton = React.memo(({ id, onClick }) => {
  return (
    <button onClick={() => onClick(id)}>
      {id}
    </button>
  );
});
```

## Native vs Synthetic Events

```javascript
// Example 16: Mixing native and synthetic events
function MixedEvents() {
  const buttonRef = useRef();
  
  useEffect(() => {
    const button = buttonRef.current;
    
    // Native event listener
    const nativeHandler = (e) => {
      console.log('Native event');
      // Fires BEFORE React synthetic event
    };
    
    button.addEventListener('click', nativeHandler);
    
    return () => {
      button.removeEventListener('click', nativeHandler);
    };
  }, []);
  
  // Synthetic event
  const syntheticHandler = (e) => {
    console.log('Synthetic event');
  };
  
  return <button ref={buttonRef} onClick={syntheticHandler}>Click</button>;
}

// Click order:
// 1. Native event (captures at element)
// 2. Event bubbles to React root
// 3. React dispatches synthetic event
// 4. Synthetic event fires

// Example 17: Preventing native vs synthetic
function PreventionOrder() {
  const divRef = useRef();
  
  useEffect(() => {
    // Native listener on document
    const handler = (e) => {
      console.log('Document clicked');
    };
    
    document.addEventListener('click', handler);
    
    return () => document.removeEventListener('click', handler);
  }, []);
  
  return (
    <div ref={divRef} onClick={(e) => {
      console.log('React handler');
      e.stopPropagation(); // Only stops React events!
      // Native document listener still fires
    }}>
      Click me
    </div>
  );
}

// To stop native events:
function StopNativeEvents() {
  const divRef = useRef();
  
  useEffect(() => {
    const handler = (e) => {
      console.log('Document clicked');
    };
    
    document.addEventListener('click', handler);
    return () => document.removeEventListener('click', handler);
  }, []);
  
  return (
    <div onClick={(e) => {
      console.log('React handler');
      e.nativeEvent.stopImmediatePropagation();
      // Stops all events, including native
    }}>
      Click me
    </div>
  );
}
```

## Special Event Cases

```javascript
// Example 18: onChange in React vs DOM
// DOM onchange: Fires when input loses focus
// React onChange: Fires on every keystroke

function OnChangeExample() {
  return (
    <input
      // React onChange = DOM oninput (mostly)
      onChange={(e) => console.log(e.target.value)}
      // Fires on every character
    />
  );
}

// To get DOM onChange behavior:
function DOMOnChange() {
  return (
    <input
      onBlur={(e) => console.log('Lost focus:', e.target.value)}
    />
  );
}

// Example 19: onInput in React
function OnInputExample() {
  return (
    <input
      // onInput also fires on every keystroke
      // Similar to onChange in most cases
      onInput={(e) => console.log(e.target.value)}
    />
  );
}

// Example 20: Form events
function FormEvents() {
  const handleSubmit = (e) => {
    e.preventDefault(); // Prevent page reload
    console.log('Form submitted');
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="username" />
      <button type="submit">Submit</button>
    </form>
  );
}

// Example 21: Focus events
function FocusEvents() {
  return (
    <div>
      <input
        onFocus={() => console.log('Focus')}
        onBlur={() => console.log('Blur')}
      />
      
      {/* FocusIn/FocusOut bubble (unlike focus/blur) */}
      <div
        onFocusCapture={() => console.log('Child focused (capture)')}
        onBlur={() => console.log('Child blurred')}
      >
        <input placeholder="Child input" />
      </div>
    </div>
  );
}
```

## Event Handler Binding

```javascript
// Example 22: Class component binding patterns
class ClassComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    
    // Bind in constructor (good)
    this.handleClick1 = this.handleClick1.bind(this);
  }
  
  handleClick1() {
    // 'this' is bound correctly
    this.setState({ count: this.state.count + 1 });
  }
  
  // Arrow function property (good)
  handleClick2 = () => {
    this.setState({ count: this.state.count + 1 });
  }
  
  handleClick3() {
    // Not bound, 'this' is undefined
    this.setState({ count: this.state.count + 1 }); // Error
  }
  
  render() {
    return (
      <div>
        <button onClick={this.handleClick1}>Method 1</button>
        <button onClick={this.handleClick2}>Method 2</button>
        
        {/* Inline bind (bad - creates new function every render) */}
        <button onClick={this.handleClick3.bind(this)}>Method 3</button>
        
        {/* Inline arrow (bad - creates new function every render) */}
        <button onClick={() => this.handleClick3()}>Method 4</button>
      </div>
    );
  }
}

// Example 23: Function component (no binding needed)
function FunctionComponent() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);
  };
  
  return <button onClick={handleClick}>Count: {count}</button>;
}
```

## Performance Considerations

```javascript
// Example 24: Event handler performance
function ExpensiveHandlers({ items }) {
  // WORST: Inline arrow with closure over item
  return items.map(item => (
    <button
      key={item.id}
      onClick={() => {
        // New function per render per item
        console.log(item.name);
      }}
    >
      {item.name}
    </button>
  ));
}

// BETTER: Separate component
const ItemButton = React.memo(({ item, onClick }) => {
  return (
    <button onClick={() => onClick(item)}>
      {item.name}
    </button>
  );
});

function BetterHandlers({ items }) {
  const handleClick = useCallback((item) => {
    console.log(item.name);
  }, []);
  
  return items.map(item => (
    <ItemButton key={item.id} item={item} onClick={handleClick} />
  ));
}

// Example 25: Debouncing events
function DebouncedInput() {
  const [value, setValue] = useState('');
  
  // Debounce to avoid excessive re-renders
  const debouncedSetValue = useMemo(
    () => debounce(setValue, 300),
    []
  );
  
  return (
    <input
      onChange={(e) => {
        const val = e.target.value;
        debouncedSetValue(val);
      }}
    />
  );
}

function debounce(fn, ms) {
  let timer;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}
```

## Common Misconceptions

1. **"React attaches listeners to each element"** - False. React uses event delegation, attaching listeners at the root level.

2. **"Synthetic events are slower than native events"** - Mostly false. The overhead is minimal, and delegation often makes React faster for many elements.

3. **"event.persist() is needed in async code"** - False in React 17+. Event pooling was removed.

4. **"stopPropagation stops all events"** - Partially false. It stops React synthetic events but not native events unless you use nativeEvent.stopImmediatePropagation().

5. **"React's onChange is the same as DOM onchange"** - False. React's onChange fires on every input change, like DOM's oninput.

## Performance Implications

1. **Event Delegation** - Reduces memory usage (one listener vs thousands) and improves initial render time (fewer DOM operations).

2. **Handler Recreation** - Inline arrow functions create new functions every render. For simple cases, this is fine. For lists or React.memo'd children, use useCallback.

3. **Passive Event Listeners** - React automatically uses passive listeners for scroll/touch events where possible, improving scroll performance.

4. **Event Pooling Removal** - React 17+ removed pooling. Slight memory increase, but eliminates confusion and bugs.

## Interview Questions

1. **Q: How does React's event delegation work?**
   A: React attaches event listeners at the root container, not on individual elements. When an event occurs, React's listener catches it, creates a synthetic event, walks up the fiber tree collecting handlers, and dispatches them in the correct order (capture then bubble).

2. **Q: What are synthetic events and why do they exist?**
   A: Synthetic events are React's cross-browser wrapper around native events. They provide a consistent API across browsers, normalize event properties, and integrate with React's batching and concurrent features.

3. **Q: What changed in React 17's event system?**
   A: React 17 moved event delegation from document to the root container, enabling multiple React versions on one page. It also removed event pooling, making async access to event properties safe without calling persist().

4. **Q: Why was event pooling removed?**
   A: Event pooling was a performance optimization from when object creation was expensive. Modern browsers are fast enough that pooling's benefits don't outweigh the confusion and bugs it caused when developers accessed events in async code.

5. **Q: How do you access the native event from a synthetic event?**
   A: Use event.nativeEvent to access the underlying browser event. This is useful for browser-specific properties or methods not normalized by React.

6. **Q: What's the difference between stopPropagation and nativeEvent.stopImmediatePropagation?**
   A: stopPropagation stops the synthetic event from bubbling to parent React handlers. nativeEvent.stopImmediatePropagation stops all events including native listeners and other React roots.

7. **Q: Why does React's onChange differ from DOM onchange?**
   A: React's onChange fires on every input change for consistent behavior across all form elements. DOM's onchange fires only when the element loses focus. React's behavior matches DOM's oninput more closely.

8. **Q: How do event handlers affect React.memo optimization?**
   A: If you pass inline arrow functions or recreated handlers to React.memo'd components, they'll re-render every time because the handler reference changes. Use useCallback to maintain stable references for optimized children.

## Key Takeaways

1. React uses event delegation, attaching listeners at the root container
2. Synthetic events provide cross-browser compatibility and React integration
3. Event pooling was removed in React 17 for better developer experience
4. React 17 moved delegation from document to root for better multi-root support
5. stopPropagation only affects React events, not native listeners
6. React's onChange fires on input, not on blur like DOM onchange
7. Avoid inline handlers in lists or with React.memo for performance
8. Access native events via event.nativeEvent for browser-specific features

## Resources

- [Handling Events in React](https://react.dev/learn/responding-to-events)
- [SyntheticEvent API](https://legacy.reactjs.org/docs/events.html)
- [React 17 Event Delegation](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#changes-to-event-delegation)
- [Event System Source Code](https://github.com/facebook/react/tree/main/packages/react-dom/src/events)
- [Why Event Pooling Was Removed](https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#no-event-pooling)
