# CSS Transitions

## Transition Properties

### transition-property

Specifies which CSS property should be animated.

```css
/* Single property */
.box {
  transition-property: background-color;
}

/* Multiple properties */
.box {
  transition-property: background-color, transform, opacity;
}

/* All animatable properties */
.box {
  transition-property: all; /* Use sparingly - performance impact */
}

/* No transition */
.box {
  transition-property: none;
}

/* Custom properties (CSS variables) */
.box {
  --color: red;
  transition-property: --color;
  background-color: var(--color);
}
```

### transition-duration

How long the transition takes.

```css
/* Single duration */
.box {
  transition-property: opacity;
  transition-duration: 0.3s; /* 300 milliseconds */
}

/* Multiple durations for multiple properties */
.box {
  transition-property: opacity, transform;
  transition-duration: 0.3s, 0.5s;
  /* opacity: 0.3s, transform: 0.5s */
}

/* Milliseconds */
.box {
  transition-duration: 300ms;
}

/* Different units */
.box {
  transition-property: width, height;
  transition-duration: 0.5s, 500ms; /* Same duration */
}
```

### transition-timing-function

Controls acceleration/deceleration during transition.

```css
/* Keyword values */
.ease { transition-timing-function: ease; } /* Slow start, fast middle, slow end (default) */
.linear { transition-timing-function: linear; } /* Constant speed */
.ease-in { transition-timing-function: ease-in; } /* Slow start */
.ease-out { transition-timing-function: ease-out; } /* Slow end */
.ease-in-out { transition-timing-function: ease-in-out; } /* Slow start and end */

/* Cubic bezier - custom curves */
.custom {
  /* cubic-bezier(x1, y1, x2, y2) */
  transition-timing-function: cubic-bezier(0.4, 0.0, 0.2, 1);
}

/* Common custom curves */
.swift-out {
  transition-timing-function: cubic-bezier(0.55, 0, 0.1, 1);
}

.bounce {
  transition-timing-function: cubic-bezier(0.68, -0.55, 0.265, 1.55);
}

/* Steps - discrete transitions */
.steps {
  transition-timing-function: steps(4); /* 4 equal steps */
}

.steps-start {
  transition-timing-function: steps(4, start); /* Jump at start */
}

.steps-end {
  transition-timing-function: steps(4, end); /* Jump at end (default) */
}

.step-start {
  transition-timing-function: step-start; /* Instant change at start */
}

.step-end {
  transition-timing-function: step-end; /* Instant change at end */
}

/* Multiple timing functions */
.multi {
  transition-property: opacity, transform;
  transition-duration: 0.3s, 0.5s;
  transition-timing-function: ease-out, cubic-bezier(0.4, 0, 0.2, 1);
}
```

### transition-delay

Delay before transition starts.

```css
/* Single delay */
.box {
  transition-property: opacity;
  transition-duration: 0.3s;
  transition-delay: 0.1s; /* Wait 100ms before starting */
}

/* Multiple delays */
.box {
  transition-property: opacity, transform;
  transition-duration: 0.3s, 0.5s;
  transition-delay: 0s, 0.2s;
  /* opacity: immediate, transform: after 200ms */
}

/* Negative delay - starts mid-transition */
.box {
  transition-duration: 1s;
  transition-delay: -0.5s; /* Starts at 50% progress */
}

/* Staggered effects */
.item:nth-child(1) { transition-delay: 0s; }
.item:nth-child(2) { transition-delay: 0.1s; }
.item:nth-child(3) { transition-delay: 0.2s; }
.item:nth-child(4) { transition-delay: 0.3s; }

/* Calculated delay */
.item {
  transition-delay: calc(var(--index) * 0.1s);
}
```

## Shorthand Syntax

```css
/* transition: property duration timing-function delay */

/* Basic */
.box {
  transition: opacity 0.3s ease-out;
}

/* With delay */
.box {
  transition: opacity 0.3s ease-out 0.1s;
}

/* Multiple transitions */
.box {
  transition: 
    opacity 0.3s ease-out,
    transform 0.5s cubic-bezier(0.4, 0, 0.2, 1) 0.1s;
}

/* All properties (use sparingly) */
.box {
  transition: all 0.3s ease;
}

/* Common pattern: specific properties for performance */
.box {
  transition: 
    opacity 0.3s ease-out,
    transform 0.3s ease-out;
}
```

## Animatable Properties

### High Performance (GPU-accelerated)

```css
/* BEST: These trigger compositing, not layout/paint */
.performant {
  transition: transform 0.3s, opacity 0.3s;
}

/* transform */
.box { transform: translateX(100px); }
.box { transform: scale(1.2); }
.box { transform: rotate(45deg); }

/* opacity */
.box { opacity: 0.5; }
```

### Medium Performance (Paint only)

```css
/* Triggers paint, not layout */
.medium {
  transition: background-color 0.3s, color 0.3s;
}

/* Colors */
.box { background-color: blue; color: red; }

/* Shadows */
.box { box-shadow: 0 2px 4px rgba(0,0,0,0.2); }
```

### Low Performance (Layout + Paint)

```css
/* AVOID: Triggers layout recalculation */
.slow {
  transition: width 0.3s, height 0.3s, top 0.3s, left 0.3s;
}

/* These cause reflow */
width, height, padding, margin, border-width,
top, right, bottom, left,
font-size, line-height

/* Scale with transform instead */
/* Bad */
.box:hover { width: 200px; height: 200px; }

/* Good */
.box:hover { transform: scale(2); }
```

### Complete List of Animatable Properties

```css
/* Visual */
opacity, color, background-color, border-color, outline-color

/* Transform */
transform, transform-origin

/* Position & Size (avoid if possible) */
top, right, bottom, left, width, height

/* Spacing (avoid if possible) */
margin, padding

/* Border */
border-width, border-radius, border-color

/* Shadow */
box-shadow, text-shadow

/* Filters */
filter, backdrop-filter

/* Text */
font-size, font-weight (with variable fonts), letter-spacing, word-spacing, line-height

/* Visibility */
visibility (discrete animation)

/* Flex & Grid (avoid) */
flex-grow, flex-shrink, flex-basis, gap

/* Other */
z-index, scroll-margin, scroll-padding
```

## Performance Optimization

### will-change

Hint to browser that property will change.

```css
/* Inform browser to optimize for changes */
.box {
  will-change: transform, opacity;
  transition: transform 0.3s, opacity 0.3s;
}

/* Add only when needed, remove after */
.box:hover {
  will-change: transform;
}

.box.animating {
  will-change: transform;
}

/* Remove when done */
.box.animation-done {
  will-change: auto;
}

/* Multiple properties */
.box {
  will-change: transform, opacity, filter;
}

/* Common pitfall: don't overuse */
/* Bad: All elements, always */
* {
  will-change: transform; /* Wastes memory */
}

/* Good: Only on interactive elements, temporarily */
.interactive {
  transition: transform 0.3s;
}

.interactive:hover,
.interactive:focus {
  will-change: transform;
  transform: scale(1.05);
}
```

### Best Practices

```css
/* 1. Stick to transform and opacity */
.performant-box {
  transition: transform 0.3s ease-out, opacity 0.3s ease-out;
}

.performant-box:hover {
  transform: translateY(-5px) scale(1.02);
  opacity: 0.9;
}

/* 2. Use specific properties, not 'all' */
/* Bad */
.box {
  transition: all 0.3s;
}

/* Good */
.box {
  transition: transform 0.3s, opacity 0.3s;
}

/* 3. Avoid layout-triggering properties */
/* Bad: Triggers layout */
.box:hover {
  width: 200px;
  height: 200px;
  font-size: 20px;
}

/* Good: Composite only */
.box:hover {
  transform: scale(1.5);
}

/* 4. Use shorter durations for better perceived performance */
.snappy {
  transition: transform 0.15s ease-out; /* Feels responsive */
}

.slow {
  transition: transform 0.8s ease-out; /* Feels sluggish */
}

/* 5. Match easing to motion direction */
.enter {
  transition: transform 0.3s ease-out; /* Decelerates entering screen */
}

.exit {
  transition: transform 0.2s ease-in; /* Accelerates leaving screen */
}
```

### Hardware Acceleration

```css
/* Force GPU acceleration (use sparingly) */
.accelerated {
  transform: translateZ(0); /* Creates compositing layer */
  /* or */
  will-change: transform;
  /* or */
  transform: translate3d(0, 0, 0);
}

/* When to use hardware acceleration */
.large-image {
  /* Large image being moved frequently */
  transform: translateZ(0);
  transition: transform 0.3s;
}

/* When NOT to use */
.static-element {
  /* Don't force acceleration on static elements */
  /* Wastes GPU memory */
}
```

## Common Patterns

```css
/* Smooth color transition */
.button {
  background-color: #3b82f6;
  color: white;
  transition: background-color 0.2s ease-in-out;
}

.button:hover {
  background-color: #2563eb;
}

/* Lift on hover */
.card {
  transition: transform 0.2s ease-out, box-shadow 0.2s ease-out;
}

.card:hover {
  transform: translateY(-4px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
}

/* Expand/collapse */
.expandable {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease-out;
}

.expandable.open {
  max-height: 500px; /* Set to max expected height */
}

/* Fade in/out */
.fade {
  opacity: 0;
  transition: opacity 0.3s ease-in-out;
}

.fade.visible {
  opacity: 1;
}

/* Slide in */
.slide {
  transform: translateX(-100%);
  transition: transform 0.3s ease-out;
}

.slide.visible {
  transform: translateX(0);
}

/* Scale on hover */
.zoom {
  transition: transform 0.2s ease-out;
}

.zoom:hover {
  transform: scale(1.1);
}

/* Rotate */
.rotate-icon {
  transition: transform 0.3s ease-in-out;
}

.open .rotate-icon {
  transform: rotate(180deg);
}

/* Underline animation */
.link {
  position: relative;
  text-decoration: none;
}

.link::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 0;
  height: 2px;
  background-color: currentColor;
  transition: width 0.3s ease-out;
}

.link:hover::after {
  width: 100%;
}

/* Staggered list animation */
.list-item {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.3s ease-out, transform 0.3s ease-out;
}

.list-item.visible {
  opacity: 1;
  transform: translateY(0);
}

.list-item:nth-child(1) { transition-delay: 0s; }
.list-item:nth-child(2) { transition-delay: 0.1s; }
.list-item:nth-child(3) { transition-delay: 0.2s; }

/* Responsive transition */
.responsive {
  transition: transform 0.3s ease-out;
}

@media (prefers-reduced-motion: reduce) {
  .responsive {
    transition: none; /* Respect user preference */
  }
}
```

## Gotchas

1. **transition: all is expensive**
   ```css
   /* Bad: Transitions every property change */
   .box {
     transition: all 0.3s;
   }
   
   /* Good: Only transition what changes */
   .box {
     transition: transform 0.3s, opacity 0.3s;
   }
   ```

2. **Auto values can't be transitioned**
   ```css
   /* Doesn't work */
   .box {
     height: auto;
     transition: height 0.3s;
   }
   
   .box.expanded {
     height: auto; /* Can't calculate in-between values */
   }
   
   /* Workaround: max-height */
   .box {
     max-height: 0;
     overflow: hidden;
     transition: max-height 0.3s;
   }
   
   .box.expanded {
     max-height: 500px; /* Must be larger than content */
   }
   ```

3. **Display: none breaks transitions**
   ```css
   /* Doesn't work */
   .box {
     display: none;
     opacity: 0;
     transition: opacity 0.3s;
   }
   
   .box.visible {
     display: block;
     opacity: 1; /* Instant change, no transition */
   }
   
   /* Solution: Use visibility or pointer-events */
   .box {
     visibility: hidden;
     opacity: 0;
     transition: opacity 0.3s, visibility 0s 0.3s;
   }
   
   .box.visible {
     visibility: visible;
     opacity: 1;
     transition-delay: 0s, 0s;
   }
   ```

4. **Transform origin matters**
   ```css
   /* Default: center */
   .box {
     transform-origin: center center;
     transition: transform 0.3s;
   }
   
   .box:hover {
     transform: scale(1.2); /* Scales from center */
   }
   
   /* Custom origin */
   .box {
     transform-origin: top left;
   }
   
   .box:hover {
     transform: scale(1.2); /* Scales from top-left */
   }
   ```

5. **Transitions don't work on initial render**
   ```css
   /* Element appears instantly on load */
   .box {
     opacity: 0;
     transition: opacity 0.3s;
   }
   
   /* Need to add class after render */
   /* JavaScript: setTimeout(() => box.classList.add('visible'), 10) */
   .box.visible {
     opacity: 1;
   }
   ```

6. **will-change overuse**
   ```css
   /* Bad: Wastes memory on all divs */
   div {
     will-change: transform;
   }
   
   /* Good: Only on interactive elements, temporarily */
   .interactive:hover {
     will-change: transform;
   }
   ```

## Interview Questions

**Q: What's the difference between transition and animation?**

A: 
- Transitions: Trigger on state change (hover, class toggle), go from A to B, run once per trigger, simpler syntax
- Animations: Can auto-play, define multiple keyframes, loop/repeat, more control (play/pause, direction), don't need trigger

Use transitions for simple state changes, animations for complex multi-step sequences.

**Q: Which CSS properties can be transitioned and which should you avoid for performance?**

A: All numeric/color properties can transition, but performance varies:

Best (GPU-accelerated):
- `transform` (translate, scale, rotate)
- `opacity`

Medium (paint only):
- `background-color`, `color`
- `box-shadow`, `text-shadow`

Avoid (trigger layout):
- `width`, `height`, `top`, `left`
- `margin`, `padding`, `font-size`

Always prefer `transform: scale()` over width/height, `transform: translate()` over top/left.

**Q: Explain transition-timing-function and common easing curves.**

A: Controls acceleration during transition:
- `ease` (default): Slow start, fast middle, slow end - natural feeling
- `linear`: Constant speed - mechanical, use for opacity/colors
- `ease-in`: Slow start - use for exits
- `ease-out`: Slow end - use for entrances
- `ease-in-out`: Slow start/end - use for loops
- `cubic-bezier(x1,y1,x2,y2)`: Custom curves
- `steps(n)`: Discrete jumps - use for sprite animations

Choose based on motion: ease-out for entrances (decelerate into view), ease-in for exits (accelerate away).

**Q: What is will-change and when should you use it?**

A: `will-change` hints to browser that property will change, allowing optimization (create compositing layer, allocate GPU memory). 

Use when:
- Element will animate frequently (drag-and-drop, games)
- Complex animations that need smooth performance
- Large elements being transformed

Don't use:
- On all elements (wastes memory)
- Permanently (defeats optimization)
- For simple hover effects

Best practice: Add on hover/before animation, remove after.

**Q: How do you create a performant hover transition?**

A: 
```css
.card {
  /* Only transition compositor properties */
  transition: transform 0.2s ease-out, opacity 0.2s ease-out;
  
  /* Optional: hint optimization */
  will-change: transform;
}

.card:hover {
  /* Use transform instead of layout properties */
  transform: translateY(-4px) scale(1.02);
  opacity: 0.95;
}
```

Key principles:
1. Use transform/opacity only
2. Keep duration short (0.15-0.3s)
3. Use ease-out for hover-in, ease-in for hover-out
4. Avoid transitioning width, height, margins
5. Add will-change only on hover state

**Q: Why can't you transition to height: auto and what's the workaround?**

A: Browser can't calculate intermediate values between fixed height and auto (unknown until layout). 

Workarounds:
1. Use `max-height` (set larger than content):
   ```css
   .box { max-height: 0; transition: max-height 0.3s; }
   .box.open { max-height: 500px; }
   ```
2. Use JavaScript to get computed height
3. Use CSS Grid with `grid-template-rows: 0fr` to `1fr`
4. Use transform: scaleY (but clips content)

Most common: max-height trick, but timing varies with content height.

**Q: How do you handle accessibility with transitions?**

A: Respect `prefers-reduced-motion`:
```css
.element {
  transition: transform 0.3s ease-out;
}

@media (prefers-reduced-motion: reduce) {
  .element {
    transition-duration: 0.01s; /* Nearly instant */
    /* or */
    transition: none;
  }
}
```

Users with vestibular disorders need reduced motion. Always test with this preference enabled. Some frameworks (Tailwind) auto-handle this with motion-safe/motion-reduce prefixes.

**Q: What's the difference between transition-delay and animation-delay?**

A: Both delay start, but:
- `transition-delay`: Delays transition on state change. Useful for staggered effects. Can be negative to start mid-transition.
- `animation-delay`: Delays animation start after page load or when applied. Negative values also start mid-animation.

Transition delay is per-property and re-triggers on each state change. Animation delay typically runs once (unless looping).

**Q: Explain how stacking context affects transitions.**

A: Transitions don't create stacking contexts unless you:
1. Use `transform` (any value except none)
2. Use `opacity` < 1
3. Use `will-change: transform/opacity`

This can cause z-index issues:
```css
/* Creates stacking context, may affect layering */
.box {
  transform: translateZ(0); /* Hardware acceleration */
  transition: transform 0.3s;
}
```

Elements with transforms create new stacking contexts, isolating them from parent's z-index hierarchy. Test layering when transitioning transforms.