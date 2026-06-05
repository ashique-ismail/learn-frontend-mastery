# useLayoutEffect Hook - Synchronous Side Effects

## The Idea

**In plain English:** `useLayoutEffect` is a way to run some code in your React app right after the screen updates internally, but before the user actually sees anything new. Think of it as a chance to quietly adjust things behind the curtain before the audience looks up.

**Real-world analogy:** Imagine a theater stage crew that rushes on stage to reposition props the instant a scene ends, but before the curtain rises for the audience. They measure the space, move furniture into exactly the right spots, and step back — all before any viewer can see the stage.

- The stage crew rushing on = the `useLayoutEffect` callback running
- Measuring and repositioning props = reading DOM dimensions and updating state/styles
- The curtain staying closed until they finish = the browser holding off on painting the screen
- The audience finally seeing the perfectly arranged stage = the browser painting the final, flicker-free result

---

## Table of Contents

- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [useEffect vs useLayoutEffect](#useeffect-vs-uselayouteffect)
- [When to Use](#when-to-use)
- [Common Use Cases](#common-use-cases)
- [Performance Implications](#performance-implications)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

`useLayoutEffect` is a React Hook that fires synchronously after all DOM mutations but before the browser has painted. It's identical to `useEffect` in signature but different in timing, making it crucial for DOM measurements and synchronous updates.

### The Timing Difference

```
useEffect Timeline:
┌─────────────┐    ┌────────────┐    ┌──────────────┐    ┌─────────────┐
│   Render    │ -> │  Paint to  │ -> │  useEffect   │ -> │   Cleanup   │
│  Component  │    │   Screen   │    │    Runs      │    │  (if needed)│
└─────────────┘    └────────────┘    └──────────────┘    └─────────────┘

useLayoutEffect Timeline:
┌─────────────┐    ┌──────────────────┐    ┌────────────┐    ┌─────────────┐
│   Render    │ -> │ useLayoutEffect  │ -> │  Paint to  │ -> │   Cleanup   │
│  Component  │    │      Runs        │    │   Screen   │    │  (if needed)│
└─────────────┘    └──────────────────┘    └────────────┘    └─────────────┘
                   (Before paint!)
```

### Basic Example

```javascript
import { useLayoutEffect, useRef, useState } from 'react';

// ❌ Problem with useEffect: Flicker visible
function TooltipWithEffect() {
  const [tooltipHeight, setTooltipHeight] = useState(0);
  const tooltipRef = useRef(null);

  useEffect(() => {
    const height = tooltipRef.current.offsetHeight;
    setTooltipHeight(height);
    // Browser already painted, then updates again - flicker!
  }, []);

  return (
    <div 
      ref={tooltipRef}
      style={{ top: `${-tooltipHeight}px` }}
    >
      Tooltip
    </div>
  );
}

// ✅ Solution with useLayoutEffect: No flicker
function TooltipWithLayoutEffect() {
  const [tooltipHeight, setTooltipHeight] = useState(0);
  const tooltipRef = useRef(null);

  useLayoutEffect(() => {
    const height = tooltipRef.current.offsetHeight;
    setTooltipHeight(height);
    // Runs before paint - no flicker!
  }, []);

  return (
    <div 
      ref={tooltipRef}
      style={{ top: `${-tooltipHeight}px` }}
    >
      Tooltip
    </div>
  );
}
```

## Basic Concepts

### Syntax

```javascript
useLayoutEffect(
  setup,
  dependencies?
);
```

**Parameters:**
- `setup`: Function with effect logic, can return cleanup function
- `dependencies`: Optional array of reactive values

**Returns:** `undefined`

### Key Characteristics

```javascript
function Component() {
  console.log('1. Render');
  
  useLayoutEffect(() => {
    console.log('2. useLayoutEffect runs (before paint)');
    
    return () => {
      console.log('Cleanup');
    };
  });
  
  useEffect(() => {
    console.log('3. useEffect runs (after paint)');
  });
  
  return <div>4. Browser paints this</div>;
}

// Output order:
// 1. Render
// 2. useLayoutEffect runs (before paint)
// 4. Browser paints this (visible to user)
// 3. useEffect runs (after paint)
```

## useEffect vs useLayoutEffect

### Visual Comparison

```javascript
// Demo showing the difference
function FlickerDemo() {
  const [show, setShow] = useState(false);

  return (
    <div>
      <button onClick={() => setShow(!show)}>Toggle</button>
      
      <h3>With useEffect (may flicker):</h3>
      <BoxWithEffect show={show} />
      
      <h3>With useLayoutEffect (no flicker):</h3>
      <BoxWithLayoutEffect show={show} />
    </div>
  );
}

function BoxWithEffect({ show }) {
  const ref = useRef(null);
  const [height, setHeight] = useState(0);

  useEffect(() => {
    if (show && ref.current) {
      // Runs AFTER paint - causes flicker
      setHeight(ref.current.offsetHeight);
    }
  }, [show]);

  return (
    <div 
      ref={ref}
      style={{ 
        height: show ? `${height}px` : '0px',
        overflow: 'hidden'
      }}
    >
      Content here
    </div>
  );
}

function BoxWithLayoutEffect({ show }) {
  const ref = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    if (show && ref.current) {
      // Runs BEFORE paint - no flicker
      setHeight(ref.current.offsetHeight);
    }
  }, [show]);

  return (
    <div 
      ref={ref}
      style={{ 
        height: show ? `${height}px` : '0px',
        overflow: 'hidden'
      }}
    >
      Content here
    </div>
  );
}
```

### Decision Matrix

| Scenario | useEffect | useLayoutEffect |
|----------|-----------|-----------------|
| Data fetching | ✅ | ❌ |
| Event listeners | ✅ | ❌ |
| Subscriptions | ✅ | ❌ |
| DOM measurements | ❌ | ✅ |
| Animations | ❌ | ✅ |
| Scroll position | ❌ | ✅ |
| Focus management | ❌ | ✅ |
| Prevent visual flicker | ❌ | ✅ |

## When to Use

### Good Use Cases

```javascript
// ✅ 1. Measuring DOM elements
function useElementSize() {
  const ref = useRef(null);
  const [size, setSize] = useState({ width: 0, height: 0 });

  useLayoutEffect(() => {
    if (ref.current) {
      const { width, height } = ref.current.getBoundingClientRect();
      setSize({ width, height });
    }
  }, []);

  return [ref, size];
}

// ✅ 2. Reading layout before paint
function Tooltip({ children, content }) {
  const targetRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (targetRef.current) {
      const rect = targetRef.current.getBoundingClientRect();
      setPosition({
        top: rect.top - 50, // Position above element
        left: rect.left + rect.width / 2
      });
    }
  }, []);

  return (
    <>
      <div ref={targetRef}>{children}</div>
      <div 
        style={{
          position: 'fixed',
          top: position.top,
          left: position.left
        }}
      >
        {content}
      </div>
    </>
  );
}

// ✅ 3. Scrolling to element
function AutoScroll({ shouldScroll }) {
  const ref = useRef(null);

  useLayoutEffect(() => {
    if (shouldScroll && ref.current) {
      ref.current.scrollIntoView({ behavior: 'smooth' });
    }
  }, [shouldScroll]);

  return <div ref={ref}>Content</div>;
}
```

### Bad Use Cases

```javascript
// ❌ 1. Data fetching (blocks paint)
function BadDataFetch() {
  useLayoutEffect(() => {
    // Blocks painting until fetch completes!
    fetch('/api/data')
      .then(res => res.json())
      .then(setData);
  }, []);
}

// ✅ Use useEffect instead
function GoodDataFetch() {
  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData);
  }, []);
}

// ❌ 2. Heavy computations (blocks paint)
function BadComputation() {
  useLayoutEffect(() => {
    // Expensive calculation blocks rendering
    const result = heavyComputation();
    setState(result);
  }, []);
}

// ✅ Use useEffect or useMemo
function GoodComputation() {
  useEffect(() => {
    const result = heavyComputation();
    setState(result);
  }, []);
}
```

## Common Use Cases

### 1. Tooltips and Popovers

```javascript
function Popover({ anchor, children }) {
  const popoverRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (!anchor || !popoverRef.current) return;

    const anchorRect = anchor.getBoundingClientRect();
    const popoverRect = popoverRef.current.getBoundingClientRect();

    // Calculate position
    let top = anchorRect.bottom + 8; // Below anchor
    let left = anchorRect.left;

    // Check if popover goes off-screen
    if (top + popoverRect.height > window.innerHeight) {
      // Position above if no room below
      top = anchorRect.top - popoverRect.height - 8;
    }

    if (left + popoverRect.width > window.innerWidth) {
      // Align to right edge if no room on right
      left = window.innerWidth - popoverRect.width - 8;
    }

    setPosition({ top, left });
  }, [anchor]);

  return (
    <div
      ref={popoverRef}
      style={{
        position: 'fixed',
        top: position.top,
        left: position.left
      }}
    >
      {children}
    </div>
  );
}
```

### 2. Measuring Elements

```javascript
function ResizablePanel({ children }) {
  const ref = useRef(null);
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  useLayoutEffect(() => {
    const measure = () => {
      if (ref.current) {
        const rect = ref.current.getBoundingClientRect();
        setDimensions({
          width: rect.width,
          height: rect.height
        });
      }
    };

    measure(); // Initial measurement

    const resizeObserver = new ResizeObserver(measure);
    if (ref.current) {
      resizeObserver.observe(ref.current);
    }

    return () => resizeObserver.disconnect();
  }, []);

  return (
    <div ref={ref}>
      <div className="dimensions-display">
        {dimensions.width}px × {dimensions.height}px
      </div>
      {children}
    </div>
  );
}
```

### 3. Animation Initialization

```javascript
function AnimatedElement() {
  const ref = useRef(null);
  const [isVisible, setIsVisible] = useState(false);

  useLayoutEffect(() => {
    if (ref.current) {
      // Set initial state before first paint
      ref.current.style.opacity = '0';
      ref.current.style.transform = 'translateY(20px)';

      // Trigger animation on next frame
      requestAnimationFrame(() => {
        requestAnimationFrame(() => {
          ref.current.style.transition = 'all 0.3s';
          ref.current.style.opacity = '1';
          ref.current.style.transform = 'translateY(0)';
          setIsVisible(true);
        });
      });
    }
  }, []);

  return (
    <div ref={ref}>
      Animated content
    </div>
  );
}
```

### 4. Focus Management

```javascript
function FocusTrap({ children, isActive }) {
  const containerRef = useRef(null);
  const previousFocusRef = useRef(null);

  useLayoutEffect(() => {
    if (!isActive) return;

    // Store currently focused element
    previousFocusRef.current = document.activeElement;

    // Find all focusable elements
    const focusableElements = containerRef.current.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    if (focusableElements.length > 0) {
      // Focus first element before paint
      focusableElements[0].focus();
    }

    return () => {
      // Restore focus on cleanup
      previousFocusRef.current?.focus();
    };
  }, [isActive]);

  return (
    <div ref={containerRef}>
      {children}
    </div>
  );
}
```

### 5. Scroll Position Restoration

```javascript
function ScrollRestoration() {
  const contentRef = useRef(null);
  const scrollPosRef = useRef(0);

  useLayoutEffect(() => {
    // Restore scroll position before paint
    if (contentRef.current && scrollPosRef.current > 0) {
      contentRef.current.scrollTop = scrollPosRef.current;
    }

    return () => {
      // Save scroll position
      if (contentRef.current) {
        scrollPosRef.current = contentRef.current.scrollTop;
      }
    };
  });

  return (
    <div ref={contentRef} style={{ height: '400px', overflow: 'auto' }}>
      {/* Long content */}
    </div>
  );
}
```

## Performance Implications

### Blocking Nature

```javascript
// ⚠️ WARNING: useLayoutEffect blocks painting
function BlockingComponent() {
  useLayoutEffect(() => {
    // This blocks the browser from painting!
    const start = Date.now();
    while (Date.now() - start < 1000) {
      // Blocking for 1 second
    }
  }, []);

  return <div>This appears after 1 second delay</div>;
}

// ✅ BETTER: Use useEffect for non-critical updates
function NonBlockingComponent() {
  useEffect(() => {
    // This doesn't block painting
    const start = Date.now();
    while (Date.now() - start < 1000) {
      // Work happens after paint
    }
  }, []);

  return <div>This appears immediately</div>;
}
```

### Server-Side Rendering

```javascript
// ⚠️ useLayoutEffect doesn't run on server
function SSRComponent() {
  const [height, setHeight] = useState(0);
  const ref = useRef(null);

  useLayoutEffect(() => {
    // This won't run during SSR
    if (ref.current) {
      setHeight(ref.current.offsetHeight);
    }
  }, []);

  return (
    <div ref={ref}>
      Height: {height}px
    </div>
  );
}

// ✅ BETTER: Use useIsomorphicLayoutEffect
function useIsomorphicLayoutEffect(callback, deps) {
  const useEffectFunc = typeof window !== 'undefined'
    ? useLayoutEffect
    : useEffect;
  
  return useEffectFunc(callback, deps);
}

function SSRSafeComponent() {
  const [height, setHeight] = useState(0);
  const ref = useRef(null);

  useIsomorphicLayoutEffect(() => {
    if (ref.current) {
      setHeight(ref.current.offsetHeight);
    }
  }, []);

  return (
    <div ref={ref}>
      Height: {height}px
    </div>
  );
}
```

## Common Mistakes

### 1. Using for Async Operations

```javascript
// ❌ WRONG: Async operation in useLayoutEffect
function BadAsync() {
  useLayoutEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData);
    // Doesn't wait for fetch to complete!
  }, []);
}

// ✅ CORRECT: Use useEffect for async
function GoodAsync() {
  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData);
  }, []);
}
```

### 2. Forgetting Cleanup

```javascript
// ❌ WRONG: No cleanup
function BadCleanup() {
  useLayoutEffect(() => {
    window.addEventListener('resize', handleResize);
    // Memory leak!
  }, []);
}

// ✅ CORRECT: Proper cleanup
function GoodCleanup() {
  useLayoutEffect(() => {
    window.addEventListener('resize', handleResize);
    
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
}
```

### 3. Overusing useLayoutEffect

```javascript
// ❌ WRONG: Unnecessary useLayoutEffect
function OveruseLayoutEffect() {
  const [count, setCount] = useState(0);

  useLayoutEffect(() => {
    // No DOM reading/writing - useEffect would be fine
    console.log('Count changed:', count);
  }, [count]);
}

// ✅ CORRECT: Use useEffect when DOM sync isn't needed
function ProperUseEffect() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('Count changed:', count);
  }, [count]);
}
```

## Best Practices

### 1. Prefer useEffect by Default

```javascript
// ✅ Start with useEffect
useEffect(() => {
  // Most side effects
}, []);

// ⚠️ Only use useLayoutEffect when needed
useLayoutEffect(() => {
  // DOM measurements, preventing flicker
}, []);
```

### 2. Keep it Fast

```javascript
// ✅ GOOD: Quick synchronous work
useLayoutEffect(() => {
  const rect = ref.current.getBoundingClientRect();
  setPosition({ x: rect.left, y: rect.top });
}, []);

// ❌ BAD: Slow operations
useLayoutEffect(() => {
  const data = expensiveComputation(); // Blocks painting!
  setState(data);
}, []);
```

### 3. Use for Visual Consistency

```javascript
// ✅ Prevent layout shift
function PreventShift() {
  const ref = useRef(null);
  const [height, setHeight] = useState('auto');

  useLayoutEffect(() => {
    if (ref.current) {
      // Set height before paint to prevent shift
      setHeight(ref.current.scrollHeight);
    }
  }, []);

  return (
    <div ref={ref} style={{ height }}>
      Dynamic content
    </div>
  );
}
```

### 4. Handle SSR Properly

```javascript
// ✅ SSR-safe helper
const useIsomorphicLayoutEffect = 
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;

function Component() {
  useIsomorphicLayoutEffect(() => {
    // Safe in both SSR and CSR
  }, []);
}
```

## Real-World Examples

### Dropdown Positioning

```javascript
function Dropdown({ trigger, items }) {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    if (!isOpen || !dropdownRef.current || !trigger) return;

    const triggerRect = trigger.getBoundingClientRect();
    const dropdownRect = dropdownRef.current.getBoundingClientRect();

    let top = triggerRect.bottom + 4;
    let left = triggerRect.left;

    // Check bottom overflow
    if (top + dropdownRect.height > window.innerHeight) {
      top = triggerRect.top - dropdownRect.height - 4;
    }

    // Check right overflow
    if (left + dropdownRect.width > window.innerWidth) {
      left = window.innerWidth - dropdownRect.width - 8;
    }

    // Check left overflow
    if (left < 0) {
      left = 8;
    }

    setPosition({ top, left });
  }, [isOpen, trigger]);

  if (!isOpen) return null;

  return (
    <div
      ref={dropdownRef}
      style={{
        position: 'fixed',
        top: position.top,
        left: position.left,
        zIndex: 1000
      }}
    >
      {items.map((item, index) => (
        <div key={index} onClick={item.onClick}>
          {item.label}
        </div>
      ))}
    </div>
  );
}
```

### Smooth Height Animation

```javascript
function Accordion({ title, children, isExpanded }) {
  const contentRef = useRef(null);
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    if (!contentRef.current) return;

    if (isExpanded) {
      const contentHeight = contentRef.current.scrollHeight;
      setHeight(contentHeight);
    } else {
      setHeight(0);
    }
  }, [isExpanded]);

  return (
    <div>
      <div className="accordion-header">{title}</div>
      <div
        style={{
          height: `${height}px`,
          overflow: 'hidden',
          transition: 'height 0.3s ease'
        }}
      >
        <div ref={contentRef}>{children}</div>
      </div>
    </div>
  );
}
```

## Interview Questions

### Q1: What's the difference between useEffect and useLayoutEffect?

**Answer:**
- **useEffect**: Runs asynchronously after paint, doesn't block browser
- **useLayoutEffect**: Runs synchronously before paint, blocks browser

Use useLayoutEffect only when you need to read/write DOM before paint to prevent visual flicker.

### Q2: When should you use useLayoutEffect?

**Answer:** Use when:
1. Reading DOM measurements (getBoundingClientRect, offsetHeight)
2. Preventing visual flicker
3. Scroll position management
4. Tooltip/popover positioning
5. Focus management

Don't use for: data fetching, subscriptions, or anything async.

### Q3: Does useLayoutEffect run on the server?

**Answer:** No. During SSR, useLayoutEffect is skipped. For SSR-compatible code, use:

```javascript
const useIsomorphicLayoutEffect = 
  typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

### Q4: Can useLayoutEffect cause performance issues?

**Answer:** Yes! It blocks painting. If you perform expensive operations in useLayoutEffect, the browser can't paint until they complete, causing lag. Keep useLayoutEffect code fast and synchronous.

## Key Takeaways

1. **Runs before paint** - Synchronous timing after DOM mutations
2. **Blocks rendering** - Browser waits for completion before painting
3. **Use sparingly** - Only when preventing visual flicker
4. **Perfect for DOM measurements** - Read layout before user sees
5. **Not for async** - Use useEffect for data fetching
6. **SSR incompatible** - Doesn't run on server
7. **Performance sensitive** - Keep operations fast

## Resources

### Official Documentation
- [useLayoutEffect Reference](https://react.dev/reference/react/useLayoutEffect)
- [useEffect vs useLayoutEffect](https://kentcdodds.com/blog/useeffect-vs-uselayouteffect)

### Articles
- [When to useLayoutEffect Instead of useEffect](https://daveceddia.com/useeffect-vs-uselayouteffect/)
- [Understanding useLayoutEffect](https://blog.logrocket.com/useeffect-vs-uselayouteffect/)

### Videos
- [useLayoutEffect Explained](https://www.youtube.com/watch?v=wU57kvYOFkU)

---

**Next Steps:** Learn about useImperativeHandle for exposing component APIs, and explore React 18's new concurrent features.
