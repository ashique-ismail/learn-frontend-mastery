# AutoAnimate

## What It Is

AutoAnimate is a zero-config animation library that adds smooth animations to list add/remove/move operations with a single line of code. No configuration, no keyframes, no animation states — just add the hook.

```bash
npm install @formkit/auto-animate
```

---

## React Hook

```tsx
import { useAutoAnimate } from '@formkit/auto-animate/react';

function TodoList() {
  const [items, setItems] = useState(['Buy groceries', 'Write tests', 'Ship feature']);
  const [parent] = useAutoAnimate();

  function addItem() {
    setItems(prev => [...prev, `New item ${prev.length + 1}`]);
  }

  function removeItem(index: number) {
    setItems(prev => prev.filter((_, i) => i !== index));
  }

  return (
    <div>
      <button onClick={addItem}>Add Item</button>
      <ul ref={parent}>   {/* ← just this ref */}
        {items.map((item, index) => (
          <li key={item}>
            {item}
            <button onClick={() => removeItem(index)}>✕</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

That's it. Adding, removing, and reordering items is now animated.

---

## Configuration Options

```tsx
const [parent] = useAutoAnimate({
  duration: 250,          // animation duration in ms (default: 250)
  easing: 'ease-in-out',  // CSS easing function
  disrespectUserMotionPreference: false, // respect prefers-reduced-motion
});

// Or use a custom animation function
const [parent] = useAutoAnimate((el, action, oldCoords, newCoords) => {
  let keyframes: Keyframe[];

  if (action === 'add') {
    keyframes = [
      { opacity: 0, transform: 'scale(0.5)' },
      { opacity: 1, transform: 'scale(1)' },
    ];
  } else if (action === 'remove') {
    keyframes = [
      { opacity: 1, transform: 'scale(1)' },
      { opacity: 0, transform: 'scale(0.5)' },
    ];
  } else {
    // move — position transition handled automatically
    keyframes = [
      { opacity: 1 },
      { opacity: 1 },
    ];
  }

  return new KeyframeEffect(el, keyframes, { duration: 300, easing: 'ease-out' });
});
```

---

## Enable/Disable Programmatically

```tsx
const [parent, enableAnimations] = useAutoAnimate();
const [isAnimating, setIsAnimating] = useState(true);

function toggleAnimations() {
  setIsAnimating(prev => {
    enableAnimations(!prev); // pass to AutoAnimate
    return !prev;
  });
}
```

---

## Vue / Vanilla JS Usage

```vue
<!-- Vue -->
<ul v-auto-animate>
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
</ul>
```

```js
// Vanilla JS
import autoAnimate from '@formkit/auto-animate';

const listEl = document.querySelector('ul');
autoAnimate(listEl);
// Now adding/removing children is animated
```

---

## How It Works

AutoAnimate uses the FLIP technique (First, Last, Invert, Play):
1. Before mutation: record element positions
2. After mutation: record new positions
3. Apply inverse transforms to "pretend" elements are in old positions
4. Animate to remove the inverse transforms → smooth transition

The library detects DOM mutations via `MutationObserver` and applies this technique automatically.

---

## Comparison with Framer Motion

| | AutoAnimate | Framer Motion |
|---|---|---|
| Setup | Single line (`ref={parent}`) | Wrap elements in `<motion.div>` |
| Bundle size | ~2 KB | ~50 KB |
| Control | Minimal | Full (variants, orchestration) |
| Layout animations | Yes (automatic) | Yes (layout prop) |
| Enter/exit animations | Yes | Yes (AnimatePresence) |
| Gestures | No | Yes (drag, hover, tap) |
| SVG animations | No | Yes |
| Complex choreography | No | Yes |

**Use AutoAnimate for:** Simple list animations, toggle visibility, reordering — "just make this smooth."

**Use Framer Motion for:** Complex enter/exit animations, gesture interactions, animated routes, intricate choreography.

---

## Common Interview Questions

**Q: What's the performance cost of AutoAnimate?**
Negligible. It uses CSS animations via the Web Animations API, which run on the compositor thread. The MutationObserver overhead is minimal. The 2 KB bundle is tiny compared to animation alternatives.

**Q: Does AutoAnimate work with `AnimatePresence` from Framer Motion?**
No — they're separate systems. AutoAnimate works by observing the DOM; Framer Motion wraps elements in motion components. If you're using Framer Motion for other animations, use its layout animations instead.

**Q: Does AutoAnimate respect `prefers-reduced-motion`?**
By default yes, when `disrespectUserMotionPreference` is `false` (the default). The animation is skipped for users who have set `prefers-reduced-motion: reduce` in their OS settings.
