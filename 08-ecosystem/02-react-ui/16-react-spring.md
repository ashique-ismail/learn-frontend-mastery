# React Spring - Physics-Based Animation Library

## Table of Contents
- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
- [Hooks API](#hooks-api)
- [Animation Patterns](#animation-patterns)
- [Gestures Integration](#gestures-integration)
- [Advanced Features](#advanced-features)
- [Performance Optimization](#performance-optimization)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use](#when-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

React Spring is a spring-physics-based animation library for React that brings fluid, natural animations to your applications. Unlike traditional duration-based animations, React Spring uses physics to create more realistic and organic motion.

**Key Features:**
- Physics-based animations (spring dynamics)
- Hooks-based API
- Interpolation and chaining
- Gesture integration
- TypeScript support
- Cross-platform (React, React Native, web)

## Installation & Setup

```bash
# Install react-spring
npm install @react-spring/web

# For React Native
npm install @react-spring/native

# For gestures
npm install @use-gesture/react
```

### Basic Setup

```jsx
import { useSpring, animated } from '@react-spring/web';

function App() {
  const springs = useSpring({
    from: { opacity: 0 },
    to: { opacity: 1 },
  });

  return <animated.div style={springs}>Hello World</animated.div>;
}
```

## Core Concepts

### 1. Spring Physics

React Spring uses spring physics instead of durations:

```jsx
import { useSpring, animated } from '@react-spring/web';

function SpringExample() {
  const [isToggled, setIsToggled] = useState(false);

  const springs = useSpring({
    x: isToggled ? 100 : 0,
    config: {
      tension: 170,  // Spring tension (higher = faster)
      friction: 26,  // Spring friction (higher = slower)
      mass: 1,       // Spring mass (higher = heavier)
    },
  });

  return (
    <>
      <animated.div
        style={{
          transform: springs.x.to(x => `translateX(${x}px)`),
        }}
      >
        Box
      </animated.div>
      <button onClick={() => setIsToggled(!isToggled)}>
        Toggle
      </button>
    </>
  );
}
```

### 2. Animated Components

Use `animated` components to animate DOM elements:

```jsx
import { useSpring, animated } from '@react-spring/web';

function AnimatedExample() {
  const springs = useSpring({
    from: { scale: 0, rotate: 0 },
    to: { scale: 1, rotate: 360 },
  });

  return (
    <animated.div
      style={{
        transform: springs.scale.to(
          s => `scale(${s}) rotate(${springs.rotate.get()}deg)`
        ),
      }}
    >
      Animated Box
    </animated.div>
  );
}
```

### 3. Configuration Presets

React Spring provides preset configurations:

```jsx
import { useSpring, animated, config } from '@react-spring/web';

function PresetExample() {
  const springs = useSpring({
    from: { y: -100 },
    to: { y: 0 },
    config: config.wobbly, // Available presets: default, gentle, wobbly, stiff, slow, molasses
  });

  return (
    <animated.div style={{ transform: springs.y.to(y => `translateY(${y}px)`) }}>
      Bouncy Box
    </animated.div>
  );
}
```

## Hooks API

### 1. useSpring Hook

Animate a single set of values:

```jsx
import { useSpring, animated } from '@react-spring/web';

function UseSpringExample() {
  const [isHovered, setIsHovered] = useState(false);

  const springs = useSpring({
    scale: isHovered ? 1.2 : 1,
    backgroundColor: isHovered ? '#ff6b6b' : '#4ecdc4',
    config: config.wobbly,
  });

  return (
    <animated.div
      style={{
        width: 100,
        height: 100,
        ...springs,
      }}
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
    />
  );
}
```

### 2. useSpring with Imperative API

Control animations programmatically:

```jsx
import { useSpring, animated } from '@react-spring/web';

function ImperativeExample() {
  const [springs, api] = useSpring(() => ({
    x: 0,
    y: 0,
  }));

  const handleClick = () => {
    api.start({
      x: Math.random() * 200,
      y: Math.random() * 200,
      config: config.gentle,
    });
  };

  return (
    <>
      <animated.div
        style={{
          position: 'absolute',
          transform: springs.x.to((x, y) => `translate(${x}px, ${springs.y.get()}px)`),
        }}
      >
        Click to move me
      </animated.div>
      <button onClick={handleClick}>Animate</button>
    </>
  );
}
```

### 3. useSprings Hook

Animate multiple items:

```jsx
import { useSprings, animated } from '@react-spring/web';

function UseSpringsExample() {
  const items = ['Item 1', 'Item 2', 'Item 3', 'Item 4'];

  const springs = useSprings(
    items.length,
    items.map((_, index) => ({
      from: { opacity: 0, x: -100 },
      to: { opacity: 1, x: 0 },
      delay: index * 100, // Stagger animation
    }))
  );

  return (
    <div>
      {springs.map((spring, index) => (
        <animated.div key={index} style={spring}>
          {items[index]}
        </animated.div>
      ))}
    </div>
  );
}
```

### 4. useTrail Hook

Create trailing animations:

```jsx
import { useTrail, animated } from '@react-spring/web';

function TrailExample() {
  const items = ['Hello', 'World', 'From', 'React', 'Spring'];
  const [isOpen, setIsOpen] = useState(false);

  const trail = useTrail(items.length, {
    opacity: isOpen ? 1 : 0,
    x: isOpen ? 0 : 20,
    height: isOpen ? 80 : 0,
    from: { opacity: 0, x: 20, height: 0 },
  });

  return (
    <>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      <div>
        {trail.map((style, index) => (
          <animated.div
            key={index}
            style={{
              ...style,
              transform: style.x.to(x => `translateX(${x}px)`),
            }}
          >
            {items[index]}
          </animated.div>
        ))}
      </div>
    </>
  );
}
```

### 5. useTransition Hook

Animate component mounting/unmounting:

```jsx
import { useTransition, animated } from '@react-spring/web';

function TransitionExample() {
  const [items, setItems] = useState([]);

  const transitions = useTransition(items, {
    from: { opacity: 0, transform: 'translateY(-20px)' },
    enter: { opacity: 1, transform: 'translateY(0px)' },
    leave: { opacity: 0, transform: 'translateY(20px)' },
    keys: item => item.id,
  });

  const addItem = () => {
    setItems([...items, { id: Date.now(), text: `Item ${items.length + 1}` }]);
  };

  const removeItem = (id) => {
    setItems(items.filter(item => item.id !== id));
  };

  return (
    <>
      <button onClick={addItem}>Add Item</button>
      <div>
        {transitions((style, item) => (
          <animated.div style={style}>
            {item.text}
            <button onClick={() => removeItem(item.id)}>Remove</button>
          </animated.div>
        ))}
      </div>
    </>
  );
}
```

### 6. useChain Hook

Chain multiple animations:

```jsx
import { useSpring, useTransition, useChain, useSpringRef, animated } from '@react-spring/web';

function ChainExample() {
  const [isOpen, setIsOpen] = useState(false);
  const items = ['A', 'B', 'C', 'D'];

  const springRef = useSpringRef();
  const springs = useSpring({
    ref: springRef,
    width: isOpen ? 200 : 80,
    height: isOpen ? 200 : 80,
  });

  const transitionRef = useSpringRef();
  const transitions = useTransition(isOpen ? items : [], {
    ref: transitionRef,
    from: { opacity: 0, scale: 0 },
    enter: { opacity: 1, scale: 1 },
    leave: { opacity: 0, scale: 0 },
  });

  // Chain animations: spring first, then transitions
  useChain(isOpen ? [springRef, transitionRef] : [transitionRef, springRef], [
    0,
    isOpen ? 0.1 : 0.3, // Timing offsets
  ]);

  return (
    <>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      <animated.div style={{ ...springs, position: 'relative' }}>
        {transitions((style, item) => (
          <animated.div style={{ ...style, position: 'absolute' }}>
            {item}
          </animated.div>
        ))}
      </animated.div>
    </>
  );
}
```

## Animation Patterns

### 1. Interpolation

Transform animated values:

```jsx
import { useSpring, animated } from '@react-spring/web';

function InterpolationExample() {
  const [isActive, setIsActive] = useState(false);

  const { x } = useSpring({
    x: isActive ? 1 : 0,
  });

  return (
    <animated.div
      style={{
        opacity: x.to([0, 0.5, 1], [0, 1, 0.5]), // Input range -> Output range
        transform: x.to(val => `scale(${1 + val * 0.5}) rotate(${val * 360}deg)`),
        backgroundColor: x.to(
          val => `rgb(${Math.round(val * 255)}, 100, ${Math.round((1 - val) * 255)})`
        ),
      }}
      onClick={() => setIsActive(!isActive)}
    >
      Click me
    </animated.div>
  );
}
```

### 2. Loop Animations

Create infinite loops:

```jsx
import { useSpring, animated } from '@react-spring/web';

function LoopExample() {
  const { rotate } = useSpring({
    from: { rotate: 0 },
    to: { rotate: 360 },
    loop: true,
    config: { duration: 2000 },
  });

  return (
    <animated.div
      style={{
        transform: rotate.to(r => `rotate(${r}deg)`),
      }}
    >
      🌀
    </animated.div>
  );
}
```

### 3. Parallel Animations

Run multiple animations simultaneously:

```jsx
import { useSpring, animated } from '@react-spring/web';

function ParallelExample() {
  const [isActive, setIsActive] = useState(false);

  const springs = useSpring({
    to: [
      { opacity: 1, color: '#ff6b6b' },
      { opacity: 0.5, color: '#4ecdc4' },
      { opacity: 1, color: '#ffe66d' },
    ],
    from: { opacity: 0, color: '#000000' },
    config: { duration: 1000 },
    reset: isActive,
  });

  return (
    <>
      <animated.div style={springs}>
        Multi-step Animation
      </animated.div>
      <button onClick={() => setIsActive(!isActive)}>Animate</button>
    </>
  );
}
```

### 4. Delay and Pause

Control animation timing:

```jsx
import { useSpring, animated } from '@react-spring/web';

function DelayExample() {
  const [springs, api] = useSpring(() => ({
    x: 0,
    opacity: 1,
  }));

  const animate = async () => {
    // Delay before starting
    await api.start({ x: 100, delay: 500 });
    
    // Pause
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // Continue
    await api.start({ x: 0, opacity: 0.5 });
  };

  return (
    <>
      <animated.div style={springs}>
        Delayed Animation
      </animated.div>
      <button onClick={animate}>Start</button>
    </>
  );
}
```

## Gestures Integration

### 1. Drag Gesture

```jsx
import { useSpring, animated } from '@react-spring/web';
import { useDrag } from '@use-gesture/react';

function DragExample() {
  const [{ x, y }, api] = useSpring(() => ({ x: 0, y: 0 }));

  const bind = useDrag(({ offset: [ox, oy], down }) => {
    api.start({
      x: ox,
      y: oy,
      immediate: down, // No spring when dragging
    });
  });

  return (
    <animated.div
      {...bind()}
      style={{
        x,
        y,
        touchAction: 'none',
        cursor: 'grab',
      }}
    >
      Drag me!
    </animated.div>
  );
}
```

### 2. Hover and Click

```jsx
import { useSpring, animated } from '@react-spring/web';
import { useHover } from '@use-gesture/react';

function HoverExample() {
  const [springs, api] = useSpring(() => ({
    scale: 1,
    rotateZ: 0,
  }));

  const bind = useHover(({ hovering }) => {
    api.start({
      scale: hovering ? 1.1 : 1,
      rotateZ: hovering ? 5 : 0,
    });
  });

  return (
    <animated.div
      {...bind()}
      style={{
        ...springs,
        width: 100,
        height: 100,
        backgroundColor: '#4ecdc4',
      }}
    >
      Hover me
    </animated.div>
  );
}
```

### 3. Swipe Gesture

```jsx
import { useSpring, animated } from '@react-spring/web';
import { useGesture } from '@use-gesture/react';

function SwipeExample() {
  const [springs, api] = useSpring(() => ({
    x: 0,
    opacity: 1,
  }));

  const bind = useGesture({
    onDrag: ({ down, movement: [mx], velocity: [vx], direction: [dx] }) => {
      if (!down && Math.abs(vx) > 0.5) {
        // Swipe detected
        api.start({
          x: dx > 0 ? 300 : -300,
          opacity: 0,
          config: { velocity: vx },
        });
      } else {
        api.start({
          x: down ? mx : 0,
          opacity: down ? 0.8 : 1,
          immediate: down,
        });
      }
    },
  });

  return (
    <animated.div
      {...bind()}
      style={{
        ...springs,
        touchAction: 'none',
      }}
    >
      Swipe me left or right
    </animated.div>
  );
}
```

## Advanced Features

### 1. Custom Springs

Create reusable spring configurations:

```jsx
const customConfig = {
  tension: 280,
  friction: 60,
  clamp: true, // Prevents overshoot
  precision: 0.01,
  velocity: 0,
};

function CustomSpringExample() {
  const springs = useSpring({
    from: { x: 0 },
    to: { x: 100 },
    config: customConfig,
  });

  return <animated.div style={springs}>Custom Spring</animated.div>;
}
```

### 2. Spring Events

Listen to animation lifecycle events:

```jsx
function EventsExample() {
  const springs = useSpring({
    from: { x: 0 },
    to: { x: 100 },
    onStart: () => console.log('Animation started'),
    onChange: (result) => console.log('Animation changed', result.value),
    onRest: () => console.log('Animation completed'),
  });

  return <animated.div style={springs}>Event-driven Animation</animated.div>;
}
```

### 3. Conditional Animations

Animate based on conditions:

```jsx
function ConditionalExample() {
  const [state, setState] = useState('idle');

  const springs = useSpring({
    to: async (next) => {
      if (state === 'loading') {
        while (state === 'loading') {
          await next({ rotate: 360 });
          await next({ rotate: 0 });
        }
      } else {
        await next({ rotate: 0, scale: 1 });
      }
    },
    from: { rotate: 0, scale: 1 },
    config: { duration: 1000 },
  });

  return (
    <>
      <animated.div style={springs}>
        {state === 'loading' ? '⏳' : '✅'}
      </animated.div>
      <button onClick={() => setState(state === 'idle' ? 'loading' : 'idle')}>
        Toggle
      </button>
    </>
  );
}
```

### 4. SVG Animations

Animate SVG elements:

```jsx
import { useSpring, animated } from '@react-spring/web';

function SVGExample() {
  const [isActive, setIsActive] = useState(false);

  const { strokeDashoffset, scale } = useSpring({
    strokeDashoffset: isActive ? 0 : 150,
    scale: isActive ? 1 : 0.8,
    config: config.slow,
  });

  return (
    <svg width="100" height="100" onClick={() => setIsActive(!isActive)}>
      <animated.circle
        cx="50"
        cy="50"
        r="40"
        stroke="#4ecdc4"
        strokeWidth="4"
        fill="none"
        strokeDasharray="150"
        strokeDashoffset={strokeDashoffset}
        transform={scale.to(s => `scale(${s})`)}
        style={{ transformOrigin: 'center' }}
      />
    </svg>
  );
}
```

## Performance Optimization

### 1. Avoid Re-renders

Use imperative API to prevent re-renders:

```jsx
function PerformanceExample() {
  const [springs, api] = useSpring(() => ({ x: 0 }));

  // This doesn't cause re-renders
  const handleMove = (e) => {
    api.start({ x: e.clientX });
  };

  return (
    <div onMouseMove={handleMove}>
      <animated.div style={springs}>High Performance</animated.div>
    </div>
  );
}
```

### 2. Use Immediate for Dragging

Disable spring physics during interaction:

```jsx
function ImmediateExample() {
  const [{ x }, api] = useSpring(() => ({ x: 0 }));

  const bind = useDrag(({ down, movement: [mx] }) => {
    api.start({
      x: down ? mx : 0,
      immediate: down, // No spring calculation while dragging
    });
  });

  return <animated.div {...bind()} style={{ x }}>Smooth Drag</animated.div>;
}
```

### 3. Optimize Multiple Animations

Use useSprings efficiently:

```jsx
function OptimizedListExample({ items }) {
  const [springs, api] = useSprings(
    items.length,
    index => ({
      opacity: 1,
      x: 0,
    }),
    [items.length] // Only recreate when length changes
  );

  // Update specific items without recreating all springs
  const updateItem = (index) => {
    api.start(i => {
      if (i === index) {
        return { x: 100, opacity: 0.5 };
      }
      return {};
    });
  };

  return springs.map((spring, index) => (
    <animated.div key={items[index].id} style={spring} onClick={() => updateItem(index)}>
      {items[index].text}
    </animated.div>
  ));
}
```

## Common Mistakes

### 1. Forgetting Animated Components

```jsx
// ❌ Wrong - won't animate
<div style={springs}>Content</div>

// ✅ Correct - use animated
<animated.div style={springs}>Content</animated.div>
```

### 2. Incorrect Interpolation

```jsx
// ❌ Wrong - accessing value directly
<animated.div style={{ transform: `translateX(${springs.x}px)` }}>

// ✅ Correct - use .to() for interpolation
<animated.div style={{ transform: springs.x.to(x => `translateX(${x}px)`) }}>
```

### 3. Missing Keys in Transitions

```jsx
// ❌ Wrong - transitions without keys
const transitions = useTransition(items, {
  from: { opacity: 0 },
  enter: { opacity: 1 },
});

// ✅ Correct - always provide keys
const transitions = useTransition(items, {
  keys: item => item.id,
  from: { opacity: 0 },
  enter: { opacity: 1 },
});
```

### 4. Overusing Reset

```jsx
// ❌ Wrong - causes unnecessary re-animations
const springs = useSpring({
  x: value,
  reset: true, // Resets on every render
});

// ✅ Correct - reset conditionally
const springs = useSpring({
  x: value,
  reset: shouldReset,
});
```

### 5. Not Handling Touch Actions

```jsx
// ❌ Wrong - gestures might conflict with scroll
<animated.div {...bind()}>

// ✅ Correct - disable touch actions
<animated.div {...bind()} style={{ touchAction: 'none' }}>
```

## Best Practices

### 1. Use Config Presets

```jsx
import { config } from '@react-spring/web';

// Start with presets, then customize
const springs = useSpring({
  x: 100,
  config: config.gentle, // default, gentle, wobbly, stiff, slow, molasses
});
```

### 2. Memoize Gesture Handlers

```jsx
const bind = useMemo(
  () => useDrag(({ offset: [x, y] }) => {
    api.start({ x, y });
  }),
  [api]
);
```

### 3. Clean Up Animations

```jsx
useEffect(() => {
  const api = createSpringAPI();
  
  return () => {
    api.stop(); // Clean up on unmount
  };
}, []);
```

### 4. Use Refs for Chaining

```jsx
const springRef = useSpringRef();
const transitionRef = useSpringRef();

useChain([springRef, transitionRef], [0, 0.2]);
```

### 5. Optimize Re-renders

```jsx
// Extract animation logic to custom hooks
function useCardAnimation(isOpen) {
  return useSpring({
    transform: isOpen ? 'scale(1.1)' : 'scale(1)',
    config: config.wobbly,
  });
}
```

## When to Use

### Use React Spring When

1. **Natural motion** - Need physics-based, organic animations
2. **Interactive animations** - Dragging, swiping, gestures
3. **Complex sequences** - Chaining and orchestrating multiple animations
4. **Performance critical** - Smooth 60fps animations
5. **Interpolation needs** - Complex value transformations

### Alternatives

- **Framer Motion** - More declarative API, better for variants
- **CSS Transitions** - Simple state-based animations
- **CSS Animations** - Keyframe-based animations
- **GSAP** - Complex timeline animations
- **Anime.js** - SVG and timeline animations

## Interview Questions

### 1. How does React Spring differ from CSS animations?

**Answer:** React Spring uses spring physics instead of duration/easing curves. Springs feel more natural because they calculate motion based on physical properties (tension, friction, mass). CSS animations use fixed durations and easing curves. React Spring also allows:
- Interruption and reversal mid-animation
- Value interpolation and transformation
- Gesture integration
- Imperative control without re-renders

### 2. Explain the difference between useSpring and useSprings.

**Answer:** 
- `useSpring`: Animates a single set of values. Returns one spring object.
- `useSprings`: Animates multiple items with independent springs. Returns an array of spring objects, one per item. Useful for lists where each item animates separately but can be orchestrated together.

```jsx
// useSpring - one item
const spring = useSpring({ x: 100 });

// useSprings - multiple items
const springs = useSprings(5, items.map(item => ({ x: item.x })));
```

### 3. How do you optimize performance with React Spring?

**Answer:**
1. **Use imperative API** to avoid re-renders
2. **Enable immediate** during drag interactions
3. **Memoize gesture handlers**
4. **Use refs** for chaining animations
5. **Batch updates** when possible
6. **Limit spring calculations** with precision and clamp
7. **Use native driver** for React Native

### 4. What is interpolation and why is it useful?

**Answer:** Interpolation transforms animated values into different output ranges or formats. It's useful for:
- **Range mapping**: `x.to([0, 1], [0, 100])` maps 0-1 to 0-100
- **Complex transforms**: `x.to(val => `scale(${val}) rotate(${val * 360}deg)`)`
- **Color transitions**: `x.to(v => `rgb(${v * 255}, 0, 0)`)`
- **Non-linear mapping**: Different rates for different ranges
- **Performance**: Calculated outside render cycle

### 5. How do you create staggered animations in React Spring?

**Answer:** Multiple approaches:
1. **Delay in useSprings**: Add incremental delays
2. **useTrail**: Built-in staggering hook
3. **useChain**: Chain multiple animation refs
4. **useTransition**: With enter/leave delays

```jsx
// Using useSprings with delay
const springs = useSprings(
  items.length,
  items.map((_, i) => ({
    from: { opacity: 0 },
    to: { opacity: 1 },
    delay: i * 100,
  }))
);

// Using useTrail
const trail = useTrail(items.length, {
  from: { opacity: 0 },
  to: { opacity: 1 },
});
```

### 6. How do you handle animation cleanup?

**Answer:**
```jsx
function Component() {
  const [springs, api] = useSpring(() => ({ x: 0 }));

  useEffect(() => {
    // Start animation
    api.start({ x: 100 });

    return () => {
      // Stop animation on unmount
      api.stop();
    };
  }, [api]);
}
```

Also use `cancel` on unmount and handle cleanup in gesture handlers.

### 7. Explain the spring configuration parameters.

**Answer:**
- **tension**: Spring stiffness (default 170). Higher = faster, snappier
- **friction**: Spring damping (default 26). Higher = slower, less bouncy
- **mass**: Spring mass (default 1). Higher = heavier, more momentum
- **clamp**: Prevents overshoot (default false)
- **precision**: Stop threshold (default 0.01)
- **velocity**: Initial velocity (default 0)
- **duration**: Override spring physics with fixed duration

### 8. How do you create infinite loop animations?

**Answer:**
```jsx
// Method 1: loop option
const springs = useSpring({
  from: { rotate: 0 },
  to: { rotate: 360 },
  loop: true,
  config: { duration: 2000 },
});

// Method 2: async to with while loop
const springs = useSpring({
  to: async (next) => {
    while (true) {
      await next({ rotate: 360 });
      await next({ rotate: 0 });
    }
  },
});

// Method 3: loop with reverse
const springs = useSpring({
  from: { scale: 1 },
  to: { scale: 1.2 },
  loop: { reverse: true },
});
```

### 9. How do you integrate gestures with React Spring?

**Answer:** Use `@use-gesture/react` library:
```jsx
import { useDrag } from '@use-gesture/react';

const [{ x, y }, api] = useSpring(() => ({ x: 0, y: 0 }));

const bind = useDrag(({ offset: [ox, oy], down }) => {
  api.start({
    x: ox,
    y: oy,
    immediate: down, // No spring while dragging
  });
});

return <animated.div {...bind()} style={{ x, y, touchAction: 'none' }} />;
```

### 10. What are the differences between declarative and imperative APIs?

**Answer:**
- **Declarative (useSpring with dependencies)**: Component re-renders trigger animations. Simple, reactive.
- **Imperative (api.start)**: Direct control without re-renders. Better performance, more control.

```jsx
// Declarative
const springs = useSpring({ x: isOpen ? 100 : 0 });

// Imperative
const [springs, api] = useSpring(() => ({ x: 0 }));
const toggle = () => api.start({ x: isOpen ? 100 : 0 });
```

## Key Takeaways

1. **React Spring uses spring physics** instead of duration-based animations for natural motion
2. **useSpring for single animations**, useSprings for multiple items, useTrail for staggering
3. **Interpolation transforms values** outside the render cycle for complex transformations
4. **Imperative API (api.start)** prevents re-renders and improves performance
5. **useTransition handles mount/unmount** animations with enter/leave/update lifecycle
6. **Gestures integrate seamlessly** with @use-gesture/react for drag, swipe, hover
7. **Config presets available** (gentle, wobbly, stiff) or customize tension/friction/mass
8. **useChain orchestrates** multiple animations in sequence with timing control
9. **animated components required** to apply spring values to DOM elements
10. **Performance optimization** through immediate mode, memoization, and imperative updates
11. **SVG animation support** for strokeDashoffset, transforms, and other properties
12. **Loop animations** with loop option or async to functions
13. **TypeScript support** with full type inference for spring values
14. **Cross-platform** - works with React DOM, React Native, and more

## Resources

### Official Documentation
- [React Spring Docs](https://www.react-spring.dev/)
- [API Reference](https://www.react-spring.dev/docs/api)
- [Examples](https://www.react-spring.dev/examples)

### Tutorials
- [React Spring Visualizer](https://react-spring-visualizer.com/)
- [Alligator.io React Spring Guide](https://alligator.io/react/react-spring/)
- [CSS-Tricks React Spring](https://css-tricks.com/animating-react-components-with-react-spring/)

### Libraries
- [@use-gesture/react](https://use-gesture.netlify.app/) - Gesture library
- [@react-spring/three](https://www.npmjs.com/package/@react-spring/three) - Three.js integration
- [@react-spring/native](https://www.npmjs.com/package/@react-spring/native) - React Native

### Tools
- [React Spring Playground](https://codesandbox.io/s/react-spring-playground)
- [Spring Config Visualizer](https://chenglou.github.io/react-motion/demos/demo5-spring-parameters-chooser/)

### Community
- [GitHub Repository](https://github.com/pmndrs/react-spring)
- [Discord Community](https://discord.gg/poimandres)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/react-spring)
