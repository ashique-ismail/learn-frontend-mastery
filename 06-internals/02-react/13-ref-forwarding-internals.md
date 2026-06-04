# Ref Forwarding Internals

## Overview

Refs provide a way to access DOM nodes or React component instances directly. Ref forwarding allows components to expose refs to their children, enabling parent components to access nested DOM elements. Understanding ref implementation, forwarding mechanics, and useImperativeHandle reveals how React manages references and exposes controlled APIs.

This guide explores ref internals, the forwardRef pattern, useImperativeHandle customization, and advanced ref patterns.

## Ref Implementation

Refs are implemented as mutable containers:

```javascript
// Example 1: useRef implementation (simplified)
function useRef(initialValue) {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}

// On updates:
function updateRef(initialValue) {
  const hook = updateWorkInProgressHook();
  // Return same ref object
  return hook.memoizedState;
}

// Example 2: Ref object structure
const ref = useRef(null);
// ref = {
//   current: null  // Mutable property
// }

// React doesn't track mutations to ref.current
ref.current = document.getElementById('test'); // No re-render
```

### Ref Assignment

```javascript
// Example 3: How refs are assigned during commit
function commitAttachRef(fiber) {
  const ref = fiber.ref;
  if (ref !== null) {
    const instance = fiber.stateNode;
    
    // Assign DOM node or component instance
    if (typeof ref === 'function') {
      // Callback ref
      ref(instance);
    } else {
      // Object ref
      ref.current = instance;
    }
  }
}

// Example 4: Ref cleanup during unmount
function commitDetachRef(fiber) {
  const ref = fiber.ref;
  if (ref !== null) {
    if (typeof ref === 'function') {
      // Call callback with null
      ref(null);
    } else {
      // Clear object ref
      ref.current = null;
    }
  }
}
```

## forwardRef Pattern

forwardRef enables ref forwarding through component boundaries:

```javascript
// Example 5: Without forwardRef (doesn't work)
function Input(props) {
  // Can't receive ref as regular prop
  return <input {...props} />;
}

function Parent() {
  const inputRef = useRef();
  
  // This won't work - ref doesn't reach the input
  return <Input ref={inputRef} />;
}

// Example 6: With forwardRef (works)
const Input = forwardRef((props, ref) => {
  // ref is second parameter
  return <input ref={ref} {...props} />;
});

function Parent() {
  const inputRef = useRef();
  
  // Now ref correctly points to the input element
  return <Input ref={inputRef} />;
}

// Example 7: forwardRef implementation (simplified)
function forwardRef(render) {
  const elementType = {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render
  };
  return elementType;
}

// During rendering:
function updateForwardRef(current, workInProgress, Component, nextProps, renderLanes) {
  const ref = workInProgress.ref;
  const render = Component.render;
  
  // Call render with props and ref
  const nextChildren = render(nextProps, ref);
  
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

### Forwarding to Nested Elements

```javascript
// Example 8: Forwarding to nested DOM element
const FancyInput = forwardRef((props, ref) => {
  return (
    <div className="fancy-input">
      <label>{props.label}</label>
      <input ref={ref} {...props} />
    </div>
  );
});

// Usage:
function Form() {
  const inputRef = useRef();
  
  const focusInput = () => {
    inputRef.current.focus();
  };
  
  return (
    <>
      <FancyInput ref={inputRef} label="Name:" />
      <button onClick={focusInput}>Focus Input</button>
    </>
  );
}

// Example 9: Forwarding through multiple layers
const InnerInput = forwardRef((props, ref) => {
  return <input ref={ref} {...props} />;
});

const MiddleInput = forwardRef((props, ref) => {
  return (
    <div>
      <InnerInput ref={ref} {...props} />
    </div>
  );
});

const OuterInput = forwardRef((props, ref) => {
  return <MiddleInput ref={ref} {...props} />;
});

// ref flows through all layers to the actual input
```

## useImperativeHandle

useImperativeHandle customizes the ref value exposed to parent:

```javascript
// Example 10: Basic useImperativeHandle
const Input = forwardRef((props, ref) => {
  const inputRef = useRef();
  
  useImperativeHandle(ref, () => ({
    // Only expose these methods, not the whole element
    focus: () => {
      inputRef.current.focus();
    },
    clear: () => {
      inputRef.current.value = '';
    }
  }));
  
  return <input ref={inputRef} {...props} />;
});

// Usage:
function Parent() {
  const inputRef = useRef();
  
  const handleClick = () => {
    inputRef.current.focus(); // Works
    inputRef.current.clear(); // Works
    inputRef.current.value = 'x'; // Doesn't work - not exposed
  };
  
  return (
    <>
      <Input ref={inputRef} />
      <button onClick={handleClick}>Focus and Clear</button>
    </>
  );
}

// Example 11: useImperativeHandle implementation (simplified)
function useImperativeHandle(ref, createHandle, deps) {
  useLayoutEffect(() => {
    if (ref !== null) {
      const handle = createHandle();
      
      if (typeof ref === 'function') {
        ref(handle);
        return () => ref(null);
      } else {
        ref.current = handle;
        return () => {
          ref.current = null;
        };
      }
    }
  }, deps);
}
```

### Advanced useImperativeHandle Patterns

```javascript
// Example 12: Exposing complex API
const VideoPlayer = forwardRef((props, ref) => {
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
    getDuration: () => {
      return videoRef.current.duration;
    },
    getCurrentTime: () => {
      return videoRef.current.currentTime;
    },
    isPlaying: () => isPlaying
  }), [isPlaying]);
  
  return <video ref={videoRef} {...props} />;
});

// Example 13: Conditional API based on props
const ConfigurableInput = forwardRef((props, ref) => {
  const inputRef = useRef();
  const { allowClear, allowValidation } = props;
  
  useImperativeHandle(ref, () => {
    const api = {
      focus: () => inputRef.current.focus(),
      getValue: () => inputRef.current.value
    };
    
    if (allowClear) {
      api.clear = () => {
        inputRef.current.value = '';
      };
    }
    
    if (allowValidation) {
      api.validate = () => {
        return inputRef.current.validity.valid;
      };
    }
    
    return api;
  }, [allowClear, allowValidation]);
  
  return <input ref={inputRef} {...props} />;
});
```

## Callback Refs

Refs can be functions instead of objects:

```javascript
// Example 14: Callback ref
function CallbackRefExample() {
  const [element, setElement] = useState(null);
  
  const callbackRef = (node) => {
    if (node !== null) {
      console.log('Element attached:', node);
      setElement(node);
    } else {
      console.log('Element detached');
      setElement(null);
    }
  };
  
  return <input ref={callbackRef} />;
}

// Callback called:
// - When component mounts (with node)
// - When component unmounts (with null)
// - When ref changes (old with null, new with node)

// Example 15: Callback ref for measurements
function MeasureExample() {
  const [height, setHeight] = useState(0);
  
  const measuredRef = useCallback(node => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  }, []);
  
  return (
    <>
      <h1 ref={measuredRef}>Hello</h1>
      <p>Height: {height}px</p>
    </>
  );
}

// Example 16: Combining object and callback refs
function CombinedRefs() {
  const inputRef = useRef();
  
  const callbackRef = useCallback((node) => {
    // Set object ref
    inputRef.current = node;
    
    // Also do measurement
    if (node) {
      console.log('Width:', node.offsetWidth);
    }
  }, []);
  
  return <input ref={callbackRef} />;
}
```

## Ref Merging

Sometimes you need to use multiple refs on one element:

```javascript
// Example 17: Manual ref merging
function mergeRefs(...refs) {
  return (value) => {
    refs.forEach(ref => {
      if (typeof ref === 'function') {
        ref(value);
      } else if (ref != null) {
        ref.current = value;
      }
    });
  };
}

const Component = forwardRef((props, ref) => {
  const internalRef = useRef();
  
  useEffect(() => {
    // Use internal ref
    internalRef.current.focus();
  }, []);
  
  // Merge external and internal refs
  return <input ref={mergeRefs(ref, internalRef)} {...props} />;
});

// Example 18: useMergedRefs hook
function useMergedRefs(...refs) {
  return useMemo(
    () => {
      if (refs.every(ref => ref == null)) {
        return null;
      }
      return (value) => {
        refs.forEach(ref => {
          if (typeof ref === 'function') {
            ref(value);
          } else if (ref != null) {
            ref.current = value;
          }
        });
      };
    },
    refs // eslint-disable-line react-hooks/exhaustive-deps
  );
}

const Component = forwardRef((props, ref) => {
  const internalRef = useRef();
  const mergedRef = useMergedRefs(ref, internalRef);
  
  return <input ref={mergedRef} {...props} />;
});
```

## Ref and Effects

```javascript
// Example 19: Accessing refs in effects
function EffectWithRef() {
  const divRef = useRef();
  
  useEffect(() => {
    // Ref is available in effects
    const div = divRef.current;
    
    if (div) {
      console.log('Div dimensions:', div.offsetWidth, div.offsetHeight);
      
      // Set up observer
      const observer = new ResizeObserver(entries => {
        console.log('Div resized:', entries[0].contentRect);
      });
      
      observer.observe(div);
      
      return () => {
        observer.disconnect();
      };
    }
  }, []); // Empty deps - run once
  
  return <div ref={divRef}>Content</div>;
}

// Example 20: Ref callback timing
function RefCallbackTiming() {
  const [show, setShow] = useState(false);
  
  const ref = useCallback((node) => {
    if (node) {
      console.log('Mounted:', node);
      // Can access node immediately
    } else {
      console.log('Unmounted');
    }
  }, []);
  
  useEffect(() => {
    console.log('Effect runs');
  });
  
  return (
    <>
      <button onClick={() => setShow(!show)}>Toggle</button>
      {show && <div ref={ref}>Content</div>}
    </>
  );
}

// Toggle on:
// 1. Ref callback with node
// 2. Effect runs
//
// Toggle off:
// 1. Ref callback with null
// 2. Effect cleanup runs
```

## Refs with Portals

```javascript
// Example 21: Refs through portals
function PortalWithRef() {
  const contentRef = useRef();
  
  useEffect(() => {
    if (contentRef.current) {
      console.log('Portal content:', contentRef.current);
      // Can access element even though it's portaled elsewhere
    }
  });
  
  return ReactDOM.createPortal(
    <div ref={contentRef}>Portal content</div>,
    document.getElementById('portal-root')
  );
}

// Ref works normally even though element is in different DOM location
```

## Class Components and Refs

```javascript
// Example 22: Refs to class components
class ClassComponent extends React.Component {
  state = { count: 0 };
  
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  getCount = () => {
    return this.state.count;
  };
  
  render() {
    return <div>Count: {this.state.count}</div>;
  }
}

function Parent() {
  const componentRef = useRef();
  
  const handleClick = () => {
    // Can access class instance
    componentRef.current.increment();
    console.log('Count:', componentRef.current.getCount());
    console.log('State:', componentRef.current.state);
  };
  
  return (
    <>
      <ClassComponent ref={componentRef} />
      <button onClick={handleClick}>Increment from parent</button>
    </>
  );
}

// Example 23: Class component with forwardRef
class InnerComponent extends React.Component {
  render() {
    return <input {...this.props} />;
  }
}

const WrappedComponent = forwardRef((props, ref) => {
  // Can't forward directly to class component
  // Wrap in DOM element
  return <InnerComponent {...props} />;
});
```

## Advanced Patterns

```javascript
// Example 24: Ref as dependency
function RefAsDependency() {
  const inputRef = useRef();
  const [value, setValue] = useState('');
  
  useEffect(() => {
    // Don't use ref as dependency!
    // Ref object never changes
    if (inputRef.current) {
      inputRef.current.focus();
    }
  }, [inputRef]); // useless dependency
  
  // Better: use callback ref
  const callbackRef = useCallback(node => {
    if (node) {
      node.focus();
    }
  }, []);
  
  return <input ref={callbackRef} value={value} onChange={e => setValue(e.target.value)} />;
}

// Example 25: Lazy initialization with ref
function LazyInitRef() {
  const expensiveRef = useRef();
  
  if (!expensiveRef.current) {
    // Only compute once
    expensiveRef.current = createExpensiveObject();
  }
  
  return <div>Using: {expensiveRef.current.name}</div>;
}

// Example 26: Previous value tracking
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  
  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

## Common Misconceptions

1. **"Refs cause re-renders when changed"** - False. Mutating ref.current doesn't trigger re-renders. Refs are escape hatches from React's reactive system.

2. **"You can pass refs as regular props"** - False. Refs are special and extracted during element creation. Use forwardRef to pass them through components.

3. **"useImperativeHandle is for all ref forwarding"** - False. It's only for customizing the exposed API. Simple forwarding doesn't need it.

4. **"Refs are available immediately after render"** - Partially false. They're available after commit (useLayoutEffect), not after the render phase returns.

5. **"Callback refs and object refs work the same"** - Mostly true functionally, but callback refs give you more control and timing flexibility.

## Performance Implications

1. **Ref Access Cost** - Reading/writing ref.current is very fast—just property access. No React overhead.

2. **Callback Ref Cost** - Callback refs are called during commit. Expensive operations in callbacks can delay painting.

3. **useImperativeHandle Cost** - Uses useLayoutEffect internally, running synchronously after commit. Keep returned objects lightweight.

4. **Ref Changes Don't Optimize** - Changing ref callbacks or objects doesn't prevent re-renders. They're processed every render during reconciliation.

## Interview Questions

1. **Q: How do refs work internally in React?**
   A: Refs are plain JavaScript objects with a mutable current property. React creates them once and returns the same object on every render. During commit, React assigns DOM nodes or component instances to ref.current, providing direct access outside the normal React flow.

2. **Q: What is forwardRef and when would you use it?**
   A: forwardRef is a higher-order function that allows components to receive ref as a second parameter and forward it to a child element. Use it when building reusable components that need to expose their inner DOM elements to parents, like form inputs or custom buttons.

3. **Q: Explain useImperativeHandle and its purpose.**
   A: useImperativeHandle customizes the value exposed through a forwarded ref. Instead of exposing the entire DOM element, you can provide a specific API with only the methods parents should access. This encapsulates implementation details and provides a cleaner interface.

4. **Q: What's the difference between object refs and callback refs?**
   A: Object refs are { current: value } objects that React mutates. Callback refs are functions React calls with the node on mount and null on unmount. Callback refs offer more flexibility—they run on every ref change and allow custom logic during attachment/detachment.

5. **Q: Why don't ref changes trigger re-renders?**
   A: Refs are intentionally designed as an escape hatch from React's reactive system. They're for accessing imperative APIs (focus, scroll, measurements) that shouldn't trigger re-renders. If changing a ref caused renders, it would create infinite loops and defeat the purpose.

6. **Q: When are refs available in useEffect?**
   A: Refs are available in all effects because effects run after commit, when React has already assigned refs. In useLayoutEffect, refs are definitely available before paint. In useEffect, they're available but after the browser has painted.

7. **Q: How do you merge multiple refs on one element?**
   A: Create a callback ref that calls all refs. For function refs, invoke them with the value. For object refs, set their current property. Libraries like react-merge-refs help, or implement a custom mergeRefs utility.

8. **Q: Can you use refs with function components?**
   A: You can't attach refs directly to function components because they don't have instances. Use forwardRef to forward the ref to an inner element. Or use useImperativeHandle to expose a custom API that makes sense for the component.

## Key Takeaways

1. Refs are mutable objects that don't trigger re-renders when changed
2. forwardRef enables components to pass refs through to children
3. useImperativeHandle customizes the ref value exposed to parents
4. Callback refs provide more control than object refs
5. Refs are assigned during commit, after render but before paint (layout effects)
6. Ref changes don't prevent reconciliation or trigger effects
7. Merge multiple refs with a custom callback that updates all of them
8. Use refs for imperative operations, not data that should trigger renders

## Resources

- [Refs and the DOM](https://react.dev/learn/manipulating-the-dom-with-refs)
- [forwardRef API](https://react.dev/reference/react/forwardRef)
- [useImperativeHandle API](https://react.dev/reference/react/useImperativeHandle)
- [Forwarding Refs](https://legacy.reactjs.org/docs/forwarding-refs.html)
- [Ref Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCommitWork.js)
