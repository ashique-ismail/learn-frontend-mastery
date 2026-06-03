# Framer Motion

## Introduction

Framer Motion is a production-ready motion library for React that makes creating animations simple. It provides a declarative API for animations, gestures, and layout animations with excellent performance. Created by Framer, it's widely used for creating smooth, professional animations in React applications.

## Installation and Setup

### Basic Installation

```bash
npm install framer-motion

# or with yarn
yarn add framer-motion

# or with pnpm
pnpm add framer-motion
```

### Basic Setup

```tsx
import { motion } from 'framer-motion';

function App() {
  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: 0.5 }}
    >
      Hello World
    </motion.div>
  );
}

export default App;
```

## Core Concepts

### Basic Animation

```tsx
import { motion } from 'framer-motion';

function BasicAnimation() {
  return (
    <div className="space-y-8 p-8">
      {/* Fade in */}
      <motion.div
        initial={{ opacity: 0 }}
        animate={{ opacity: 1 }}
        transition={{ duration: 1 }}
        className="w-24 h-24 bg-blue-500 rounded-lg"
      />

      {/* Slide in */}
      <motion.div
        initial={{ x: -100 }}
        animate={{ x: 0 }}
        transition={{ type: 'spring', stiffness: 100 }}
        className="w-24 h-24 bg-green-500 rounded-lg"
      />

      {/* Scale and rotate */}
      <motion.div
        initial={{ scale: 0, rotate: -180 }}
        animate={{ scale: 1, rotate: 0 }}
        transition={{ duration: 0.5 }}
        className="w-24 h-24 bg-purple-500 rounded-lg"
      />

      {/* Multiple properties */}
      <motion.div
        initial={{ opacity: 0, y: 50, scale: 0.5 }}
        animate={{ opacity: 1, y: 0, scale: 1 }}
        transition={{ duration: 0.8, ease: 'easeOut' }}
        className="w-24 h-24 bg-red-500 rounded-lg"
      />
    </div>
  );
}

export default BasicAnimation;
```

### Hover and Tap Animations

```tsx
import { motion } from 'framer-motion';

function InteractiveAnimations() {
  return (
    <div className="flex gap-4 p-8">
      {/* Hover scale */}
      <motion.button
        whileHover={{ scale: 1.1 }}
        whileTap={{ scale: 0.9 }}
        className="px-6 py-3 bg-blue-600 text-white rounded-lg"
      >
        Hover Me
      </motion.button>

      {/* Hover rotate */}
      <motion.div
        whileHover={{ rotate: 180 }}
        transition={{ duration: 0.3 }}
        className="w-16 h-16 bg-green-500 rounded-lg cursor-pointer"
      />

      {/* Complex hover effect */}
      <motion.button
        whileHover={{
          scale: 1.05,
          boxShadow: '0px 10px 30px rgba(0, 0, 0, 0.2)',
        }}
        whileTap={{ scale: 0.95 }}
        transition={{ type: 'spring', stiffness: 400 }}
        className="px-6 py-3 bg-purple-600 text-white rounded-lg"
      >
        Press Me
      </motion.button>

      {/* Hover with color change */}
      <motion.div
        whileHover={{
          backgroundColor: '#ff6b6b',
          borderRadius: '50%',
        }}
        className="w-16 h-16 bg-indigo-500 cursor-pointer"
      />
    </div>
  );
}

export default InteractiveAnimations;
```

### Variants

```tsx
import { motion } from 'framer-motion';

const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      delayChildren: 0.3,
      staggerChildren: 0.2,
    },
  },
};

const itemVariants = {
  hidden: { y: 20, opacity: 0 },
  visible: {
    y: 0,
    opacity: 1,
  },
};

function VariantsExample() {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      animate="visible"
      className="space-y-4 p-8"
    >
      {[1, 2, 3, 4].map((item) => (
        <motion.li
          key={item}
          variants={itemVariants}
          className="p-4 bg-blue-500 text-white rounded-lg"
        >
          Item {item}
        </motion.li>
      ))}
    </motion.ul>
  );
}

export default VariantsExample;
```

### Layout Animations

```tsx
import { motion, AnimatePresence } from 'framer-motion';
import { useState } from 'react';

function LayoutAnimations() {
  const [isExpanded, setIsExpanded] = useState(false);
  const [items, setItems] = useState([1, 2, 3]);

  return (
    <div className="p-8 space-y-8">
      {/* Layout animation */}
      <motion.div
        layout
        onClick={() => setIsExpanded(!isExpanded)}
        className="bg-blue-500 text-white p-4 rounded-lg cursor-pointer"
        style={{
          width: isExpanded ? 300 : 150,
          height: isExpanded ? 200 : 100,
        }}
      >
        <motion.p layout>
          Click to {isExpanded ? 'collapse' : 'expand'}
        </motion.p>
      </motion.div>

      {/* Reorder list */}
      <div>
        <button
          onClick={() => setItems([...items].reverse())}
          className="mb-4 px-4 py-2 bg-green-600 text-white rounded"
        >
          Shuffle
        </button>
        <AnimatePresence>
          {items.map((item) => (
            <motion.div
              key={item}
              layout
              initial={{ opacity: 0, x: -50 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: 50 }}
              className="p-4 mb-2 bg-purple-500 text-white rounded-lg"
            >
              Item {item}
            </motion.div>
          ))}
        </AnimatePresence>
      </div>
    </div>
  );
}

export default LayoutAnimations;
```

### AnimatePresence (Enter/Exit)

```tsx
import { motion, AnimatePresence } from 'framer-motion';
import { useState } from 'react';

function AnimatePresenceExample() {
  const [isVisible, setIsVisible] = useState(true);
  const [items, setItems] = useState([1, 2, 3, 4]);

  const removeItem = (id: number) => {
    setItems(items.filter((item) => item !== id));
  };

  return (
    <div className="p-8 space-y-8">
      {/* Toggle visibility */}
      <div>
        <button
          onClick={() => setIsVisible(!isVisible)}
          className="mb-4 px-4 py-2 bg-blue-600 text-white rounded"
        >
          Toggle
        </button>
        <AnimatePresence>
          {isVisible && (
            <motion.div
              initial={{ opacity: 0, scale: 0 }}
              animate={{ opacity: 1, scale: 1 }}
              exit={{ opacity: 0, scale: 0 }}
              transition={{ duration: 0.3 }}
              className="p-8 bg-blue-500 text-white rounded-lg"
            >
              I can be toggled!
            </motion.div>
          )}
        </AnimatePresence>
      </div>

      {/* List with exit animations */}
      <div>
        <AnimatePresence>
          {items.map((item) => (
            <motion.div
              key={item}
              initial={{ opacity: 0, height: 0 }}
              animate={{ opacity: 1, height: 'auto' }}
              exit={{ opacity: 0, height: 0 }}
              transition={{ duration: 0.3 }}
              className="mb-2 p-4 bg-green-500 text-white rounded-lg flex justify-between items-center overflow-hidden"
            >
              <span>Item {item}</span>
              <button
                onClick={() => removeItem(item)}
                className="px-3 py-1 bg-red-600 rounded"
              >
                Remove
              </button>
            </motion.div>
          ))}
        </AnimatePresence>
      </div>
    </div>
  );
}

export default AnimatePresenceExample;
```

### Gestures (Drag, Pan)

```tsx
import { motion } from 'framer-motion';

function GesturesExample() {
  return (
    <div className="p-8 space-y-8">
      {/* Basic drag */}
      <motion.div
        drag
        dragConstraints={{ left: 0, right: 200, top: 0, bottom: 200 }}
        className="w-24 h-24 bg-blue-500 rounded-lg cursor-grab active:cursor-grabbing"
      >
        <p className="text-white text-center pt-8 text-sm">Drag me</p>
      </motion.div>

      {/* Drag with elastic */}
      <motion.div
        drag
        dragElastic={0.2}
        dragConstraints={{ left: -100, right: 100, top: -100, bottom: 100 }}
        className="w-24 h-24 bg-green-500 rounded-lg cursor-grab"
      >
        <p className="text-white text-center pt-8 text-sm">Elastic</p>
      </motion.div>

      {/* Drag only horizontally */}
      <motion.div
        drag="x"
        dragConstraints={{ left: 0, right: 300 }}
        className="w-24 h-24 bg-purple-500 rounded-lg cursor-grab"
      >
        <p className="text-white text-center pt-8 text-sm">Horizontal</p>
      </motion.div>

      {/* Drag with snap back */}
      <motion.div
        drag
        dragSnapToOrigin
        className="w-24 h-24 bg-red-500 rounded-lg cursor-grab"
      >
        <p className="text-white text-center pt-8 text-sm">Snap back</p>
      </motion.div>
    </div>
  );
}

export default GesturesExample;
```

### Scroll Animations

```tsx
import { motion, useScroll, useTransform } from 'framer-motion';
import { useRef } from 'react';

function ScrollAnimations() {
  const ref = useRef<HTMLDivElement>(null);
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start end', 'end start'],
  });

  const opacity = useTransform(scrollYProgress, [0, 0.5, 1], [0, 1, 0]);
  const scale = useTransform(scrollYProgress, [0, 0.5, 1], [0.8, 1, 0.8]);
  const y = useTransform(scrollYProgress, [0, 1], [100, -100]);

  return (
    <div className="h-[200vh]">
      {/* Progress bar */}
      <motion.div
        className="fixed top-0 left-0 right-0 h-2 bg-blue-600 origin-left z-50"
        style={{ scaleX: scrollYProgress }}
      />

      {/* Scroll-triggered animation */}
      <motion.div
        ref={ref}
        style={{ opacity, scale, y }}
        className="mt-[50vh] mx-auto w-64 h-64 bg-gradient-to-r from-purple-500 to-pink-500 rounded-lg flex items-center justify-center"
      >
        <p className="text-white text-xl font-bold">Scroll to animate</p>
      </motion.div>
    </div>
  );
}

export default ScrollAnimations;
```

### SVG Path Animation

```tsx
import { motion } from 'framer-motion';

function SVGAnimation() {
  const icon = {
    hidden: {
      opacity: 0,
      pathLength: 0,
      fill: 'rgba(255, 255, 255, 0)',
    },
    visible: {
      opacity: 1,
      pathLength: 1,
      fill: 'rgba(255, 255, 255, 1)',
    },
  };

  return (
    <div className="p-8 flex justify-center">
      <svg
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 100 100"
        className="w-48 h-48"
      >
        <motion.path
          d="M 10 50 L 40 80 L 90 20"
          stroke="#0099ff"
          strokeWidth="4"
          fill="none"
          variants={icon}
          initial="hidden"
          animate="visible"
          transition={{
            default: { duration: 2, ease: 'easeInOut' },
            fill: { duration: 2, ease: [1, 0, 0.8, 1] },
          }}
        />
      </svg>
    </div>
  );
}

export default SVGAnimation;
```

### Keyframes

```tsx
import { motion } from 'framer-motion';

function KeyframesAnimation() {
  return (
    <div className="p-8">
      <motion.div
        animate={{
          scale: [1, 2, 2, 1, 1],
          rotate: [0, 0, 180, 180, 0],
          borderRadius: ['0%', '0%', '50%', '50%', '0%'],
        }}
        transition={{
          duration: 2,
          ease: 'easeInOut',
          times: [0, 0.2, 0.5, 0.8, 1],
          repeat: Infinity,
          repeatDelay: 1,
        }}
        className="w-24 h-24 bg-gradient-to-r from-blue-500 to-purple-500"
      />
    </div>
  );
}

export default KeyframesAnimation;
```

## Advanced Patterns

### useAnimation Hook

```tsx
import { motion, useAnimation } from 'framer-motion';
import { useEffect } from 'react';

function UseAnimationExample() {
  const controls = useAnimation();

  useEffect(() => {
    const sequence = async () => {
      await controls.start({ x: 100, transition: { duration: 1 } });
      await controls.start({ y: 100, transition: { duration: 1 } });
      await controls.start({ x: 0, y: 0, transition: { duration: 1 } });
    };
    
    sequence();
  }, [controls]);

  return (
    <motion.div
      animate={controls}
      className="w-24 h-24 bg-blue-500 rounded-lg"
    />
  );
}

export default UseAnimationExample;
```

### Shared Layout Animations

```tsx
import { motion, AnimateSharedLayout } from 'framer-motion';
import { useState } from 'react';

function SharedLayoutExample() {
  const [selected, setSelected] = useState<number | null>(null);

  const items = [1, 2, 3, 4];

  return (
    <div className="p-8">
      <div className="grid grid-cols-2 gap-4">
        {items.map((item) => (
          <motion.div
            key={item}
            layoutId={`item-${item}`}
            onClick={() => setSelected(item)}
            className="p-8 bg-blue-500 rounded-lg cursor-pointer"
          >
            Item {item}
          </motion.div>
        ))}
      </div>

      {selected && (
        <motion.div
          layoutId={`item-${selected}`}
          className="fixed inset-0 bg-blue-500 flex items-center justify-center"
          onClick={() => setSelected(null)}
        >
          <motion.h2 className="text-white text-4xl">
            Item {selected}
          </motion.h2>
        </motion.div>
      )}
    </div>
  );
}

export default SharedLayoutExample;
```

### Custom Spring Animation

```tsx
import { motion } from 'framer-motion';

function SpringAnimation() {
  return (
    <div className="p-8 space-y-4">
      {/* Stiff spring */}
      <motion.div
        initial={{ x: -100 }}
        animate={{ x: 0 }}
        transition={{ type: 'spring', stiffness: 500, damping: 30 }}
        className="w-24 h-24 bg-blue-500 rounded-lg"
      />

      {/* Soft spring */}
      <motion.div
        initial={{ x: -100 }}
        animate={{ x: 0 }}
        transition={{ type: 'spring', stiffness: 50, damping: 10 }}
        className="w-24 h-24 bg-green-500 rounded-lg"
      />

      {/* Bouncy spring */}
      <motion.div
        initial={{ x: -100 }}
        animate={{ x: 0 }}
        transition={{ type: 'spring', stiffness: 300, damping: 5 }}
        className="w-24 h-24 bg-purple-500 rounded-lg"
      />
    </div>
  );
}

export default SpringAnimation;
```

### Page Transitions

```tsx
import { motion, AnimatePresence } from 'framer-motion';
import { useState } from 'react';

const pageVariants = {
  initial: {
    opacity: 0,
    x: '-100vw',
  },
  in: {
    opacity: 1,
    x: 0,
  },
  out: {
    opacity: 0,
    x: '100vw',
  },
};

const pageTransition = {
  type: 'tween',
  ease: 'anticipate',
  duration: 0.5,
};

function PageTransitions() {
  const [page, setPage] = useState(1);

  return (
    <div className="p-8">
      <div className="mb-4 space-x-2">
        <button
          onClick={() => setPage(1)}
          className="px-4 py-2 bg-blue-600 text-white rounded"
        >
          Page 1
        </button>
        <button
          onClick={() => setPage(2)}
          className="px-4 py-2 bg-green-600 text-white rounded"
        >
          Page 2
        </button>
        <button
          onClick={() => setPage(3)}
          className="px-4 py-2 bg-purple-600 text-white rounded"
        >
          Page 3
        </button>
      </div>

      <AnimatePresence mode="wait">
        <motion.div
          key={page}
          initial="initial"
          animate="in"
          exit="out"
          variants={pageVariants}
          transition={pageTransition}
          className="p-12 bg-gray-100 rounded-lg"
        >
          <h1 className="text-4xl font-bold">Page {page}</h1>
          <p className="mt-4">This is the content of page {page}</p>
        </motion.div>
      </AnimatePresence>
    </div>
  );
}

export default PageTransitions;
```

## Common Mistakes

1. **Not using AnimatePresence for exit animations**
```tsx
// Bad - No exit animation
{show && <motion.div exit={{ opacity: 0 }} />}

// Good
<AnimatePresence>
  {show && <motion.div exit={{ opacity: 0 }} />}
</AnimatePresence>
```

2. **Animating expensive properties**
```tsx
// Bad - Causes layout recalculation
<motion.div animate={{ width: 200 }} />

// Good - Uses transform
<motion.div animate={{ scaleX: 2 }} />
```

3. **Not providing keys in lists**
```tsx
// Bad
{items.map(item => <motion.div />)}

// Good
{items.map(item => <motion.div key={item.id} />)}
```

4. **Over-animating**
```tsx
// Bad - Too many simultaneous animations
<motion.div animate={{ x: 100, y: 100, rotate: 360, scale: 2, opacity: 0.5 }} />

// Good - Keep it simple
<motion.div animate={{ scale: 1.1 }} whileHover={{ scale: 1.2 }} />
```

5. **Not optimizing layout animations**
```tsx
// Bad - Animates position directly
<motion.div animate={{ left: 100 }} />

// Good - Uses transform for GPU acceleration
<motion.div animate={{ x: 100 }} />
```

## Best Practices

1. **Use transforms for performance**: x, y, scale, rotate are GPU-accelerated
2. **Minimize layout thrashing**: Avoid animating width, height, top, left
3. **Use variants for complex animations**: Cleaner code and better orchestration
4. **Implement loading states**: Show animation feedback
5. **Test on mobile devices**: Ensure smooth performance
6. **Use will-change carefully**: Only when needed for performance
7. **Leverage AnimatePresence**: For enter/exit animations
8. **Use layout prop wisely**: Can be expensive, use sparingly
9. **Implement proper error boundaries**: Catch animation errors
10. **Profile performance**: Use React DevTools and Chrome DevTools

## When to Use Framer Motion

### Use Framer Motion When:
- Need professional, smooth animations in React
- Building interactive UIs with gestures
- Want declarative animation syntax
- Need layout animations
- Building prototypes or MVPs quickly
- Want excellent TypeScript support
- Need scroll-triggered animations
- Building design-heavy applications

### Consider Alternatives When:
- Need only CSS animations (use CSS or Tailwind)
- Building simple static sites
- Bundle size is critical (consider react-spring)
- Need physics-based animations only (react-spring is lighter)
- Working with WebGL/3D (use Three.js with react-three-fiber)

## Interview Questions

1. **Q: How does Framer Motion optimize performance?**
   A: Uses transform and opacity (GPU-accelerated properties), implements automatic hardware acceleration, provides layout animations via FLIP technique, and batches animations to minimize repaints.

2. **Q: Explain variants in Framer Motion.**
   A: Variants are predefined animation states that can be orchestrated. They support parent-child propagation, staggered animations via staggerChildren, and cleaner code by separating animation logic from JSX.

3. **Q: What is AnimatePresence used for?**
   A: AnimatePresence enables exit animations by keeping components mounted until animations complete. It's required for any component that needs to animate out of the DOM.

4. **Q: How do layout animations work?**
   A: Layout animations use the FLIP technique (First, Last, Invert, Play). Framer Motion calculates start and end positions, inverts the transform, then animates back to natural position.

5. **Q: What's the difference between initial and animate props?**
   A: initial defines the starting state before animation, animate defines the target state. Animations run when animate values change. Both accept objects or variant names.

## Key Takeaways

1. Framer Motion provides declarative animation syntax for React
2. Uses GPU-accelerated properties for optimal performance
3. Variants enable complex, orchestrated animations
4. AnimatePresence is required for exit animations
5. Layout animations use FLIP technique for smooth transitions
6. Gestures (drag, pan) are built-in with simple API
7. Scroll animations with useScroll and useTransform hooks
8. Spring physics provide natural-feeling animations
9. Excellent TypeScript support throughout
10. Production-ready with extensive documentation and examples

## Resources

- **Official Documentation**: https://www.framer.com/motion/
- **GitHub Repository**: https://github.com/framer/motion
- **API Reference**: https://www.framer.com/motion/component/
- **Animation Examples**: https://www.framer.com/motion/examples/
- **Interactive Tutorial**: https://www.framer.com/motion/introduction/
- **Community Examples**: https://codesandbox.io/search?query=framer-motion
- **YouTube Channel**: Framer official channel
- **Discord Community**: https://discord.gg/framer
- **NPM Package**: https://www.npmjs.com/package/framer-motion
- **Best Practices Guide**: https://www.framer.com/motion/animation/##performance
