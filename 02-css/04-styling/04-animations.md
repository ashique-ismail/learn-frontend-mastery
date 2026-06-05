# CSS Animations

## The Idea

**In plain English:** CSS animations let you make elements on a webpage move, fade, grow, or change over time automatically — you describe what the element should look like at different moments, and the browser smoothly plays it out like a mini movie clip.

**Real-world analogy:** Think of a flipbook — those small booklets where you draw a slightly different picture on each page, and when you flip through them fast, the drawings appear to move. Each page in the flipbook is a snapshot of the action at a specific moment.

- The flipbook pages = keyframes (the snapshots you define at 0%, 50%, 100%, etc.)
- The drawings on each page = CSS property values (position, opacity, size) at that moment
- Flipping through the pages = the browser playing the animation over its duration

---

## @keyframes Syntax

Define animation sequences with keyframes.

```css
/* Named animation */
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

/* Percentage keyframes */
@keyframes slideIn {
  0% {
    transform: translateX(-100%);
    opacity: 0;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    transform: translateX(0);
    opacity: 1;
  }
}

/* Multiple properties */
@keyframes bounce {
  0%, 100% {
    transform: translateY(0);
    animation-timing-function: ease-out;
  }
  50% {
    transform: translateY(-30px);
    animation-timing-function: ease-in;
  }
}

/* Reusable keyframes */
@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

/* Complex multi-stage animation */
@keyframes complexAnimation {
  0% {
    transform: scale(1) rotate(0deg);
    background-color: red;
  }
  25% {
    transform: scale(1.2) rotate(90deg);
    background-color: yellow;
  }
  50% {
    transform: scale(1.4) rotate(180deg);
    background-color: blue;
  }
  75% {
    transform: scale(1.2) rotate(270deg);
    background-color: green;
  }
  100% {
    transform: scale(1) rotate(360deg);
    background-color: red;
  }
}
```

## Animation Properties

### animation-name

```css
/* Single animation */
.element {
  animation-name: fadeIn;
}

/* Multiple animations */
.element {
  animation-name: fadeIn, slideIn;
}

/* No animation */
.element {
  animation-name: none;
}
```

### animation-duration

```css
/* Single duration */
.element {
  animation-name: fadeIn;
  animation-duration: 1s;
}

/* Multiple durations */
.element {
  animation-name: fadeIn, slideIn;
  animation-duration: 1s, 2s;
}

/* Milliseconds */
.element {
  animation-duration: 500ms;
}
```

### animation-timing-function

```css
/* Keyword values */
.ease { animation-timing-function: ease; }
.linear { animation-timing-function: linear; }
.ease-in { animation-timing-function: ease-in; }
.ease-out { animation-timing-function: ease-out; }
.ease-in-out { animation-timing-function: ease-in-out; }

/* Cubic bezier */
.custom {
  animation-timing-function: cubic-bezier(0.42, 0, 0.58, 1);
}

/* Common curves */
.swift {
  animation-timing-function: cubic-bezier(0.4, 0.0, 0.2, 1);
}

.bounce {
  animation-timing-function: cubic-bezier(0.68, -0.55, 0.265, 1.55);
}

/* Steps */
.steps {
  animation-timing-function: steps(4, end);
}

.step-start {
  animation-timing-function: step-start;
}

.step-end {
  animation-timing-function: step-end;
}

/* Per-keyframe timing (preferred for complex animations) */
@keyframes variableTiming {
  0% {
    transform: translateY(0);
    animation-timing-function: ease-out;
  }
  50% {
    transform: translateY(-100px);
    animation-timing-function: ease-in;
  }
  100% {
    transform: translateY(0);
  }
}
```

### animation-delay

```css
/* Delay start */
.element {
  animation-name: fadeIn;
  animation-duration: 1s;
  animation-delay: 0.5s; /* Wait 500ms */
}

/* Multiple delays */
.element {
  animation-name: fadeIn, slideIn;
  animation-duration: 1s, 2s;
  animation-delay: 0s, 0.5s;
}

/* Negative delay - start mid-animation */
.element {
  animation-duration: 2s;
  animation-delay: -1s; /* Start at 50% progress */
}

/* Staggered children */
.item:nth-child(1) { animation-delay: 0s; }
.item:nth-child(2) { animation-delay: 0.1s; }
.item:nth-child(3) { animation-delay: 0.2s; }
.item:nth-child(4) { animation-delay: 0.3s; }

/* Calculated delay */
.item {
  animation-delay: calc(var(--index) * 100ms);
}
```

### animation-iteration-count

```css
/* Run once */
.element {
  animation-iteration-count: 1; /* Default */
}

/* Run multiple times */
.element {
  animation-iteration-count: 3;
}

/* Loop forever */
.element {
  animation-iteration-count: infinite;
}

/* Partial iterations */
.element {
  animation-iteration-count: 2.5; /* 2.5 cycles */
}

/* Different counts for multiple animations */
.element {
  animation-name: fadeIn, pulse;
  animation-duration: 1s, 0.5s;
  animation-iteration-count: 1, infinite;
}
```

### animation-direction

```css
/* Normal - forward (default) */
.element {
  animation-direction: normal;
}

/* Reverse - backward */
.element {
  animation-direction: reverse;
}

/* Alternate - forward, then backward */
.element {
  animation-direction: alternate;
  animation-iteration-count: infinite;
}

/* Alternate reverse - backward, then forward */
.element {
  animation-direction: alternate-reverse;
  animation-iteration-count: infinite;
}

/* Example: smooth pendulum */
@keyframes swing {
  from { transform: rotate(-10deg); }
  to { transform: rotate(10deg); }
}

.pendulum {
  animation-name: swing;
  animation-duration: 1s;
  animation-direction: alternate;
  animation-iteration-count: infinite;
  animation-timing-function: ease-in-out;
}
```

### animation-fill-mode

Controls element state before/after animation.

```css
/* none - default, no styles applied before/after */
.element {
  animation-fill-mode: none;
}

/* forwards - retain final keyframe styles */
.element {
  animation-fill-mode: forwards;
}

@keyframes slideOut {
  to {
    transform: translateX(100%);
    opacity: 0;
  }
}

.slide-out {
  animation: slideOut 0.5s forwards;
  /* Stays at translateX(100%), opacity: 0 after animation */
}

/* backwards - apply first keyframe during delay */
.element {
  animation-fill-mode: backwards;
  animation-delay: 1s;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.fade-in {
  opacity: 1; /* Initial state */
  animation: fadeIn 1s backwards;
  animation-delay: 2s;
  /* During 2s delay, opacity is 0 (from keyframe) */
}

/* both - apply both forwards and backwards */
.element {
  animation-fill-mode: both;
}
```

### animation-play-state

Control animation playback.

```css
/* Running - play animation (default) */
.element {
  animation-play-state: running;
}

/* Paused - freeze animation */
.element {
  animation-play-state: paused;
}

/* Pause on hover */
.spinner {
  animation: spin 1s linear infinite;
}

.spinner:hover {
  animation-play-state: paused;
}

/* Toggle with JavaScript */
.element {
  animation: fadeIn 1s;
}

.element.paused {
  animation-play-state: paused;
}

/* Pause container to pause all child animations */
.container:hover .animated-child {
  animation-play-state: paused;
}
```

## Shorthand Syntax

```css
/* animation: name duration timing-function delay iteration-count direction fill-mode play-state */

/* Basic */
.element {
  animation: fadeIn 1s;
}

/* With timing function */
.element {
  animation: slideIn 0.5s ease-out;
}

/* With delay */
.element {
  animation: fadeIn 1s ease-in 0.5s;
}

/* Complete */
.element {
  animation: bounce 1s ease-in-out 0.5s infinite alternate both running;
}

/* Multiple animations */
.element {
  animation: 
    fadeIn 1s ease-out forwards,
    slideIn 0.5s ease-in 0.5s forwards;
}

/* Common patterns */
.fade {
  animation: fadeIn 0.3s ease-out;
}

.slide {
  animation: slideIn 0.5s cubic-bezier(0.4, 0, 0.2, 1);
}

.pulse {
  animation: pulse 2s ease-in-out infinite;
}

.spin {
  animation: spin 1s linear infinite;
}
```

## Percentage Keyframes

```css
/* Precise control at multiple points */
@keyframes progressBar {
  0% {
    width: 0%;
    background-color: red;
  }
  25% {
    background-color: orange;
  }
  50% {
    width: 50%;
    background-color: yellow;
  }
  75% {
    background-color: lightgreen;
  }
  100% {
    width: 100%;
    background-color: green;
  }
}

/* Multiple keyframes at same percentage */
@keyframes jump {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-50px);
  }
}

/* Hold at keyframes */
@keyframes blinkHold {
  0%, 49%, 51%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}

/* Elastic effect */
@keyframes elasticIn {
  0% {
    transform: scale(0);
  }
  55% {
    transform: scale(1.15);
  }
  70% {
    transform: scale(0.95);
  }
  85% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
}
```

## Animation Events

JavaScript events for animation lifecycle.

```html
<div class="animated-box"></div>

<script>
const box = document.querySelector('.animated-box');

// Animation starts (after delay)
box.addEventListener('animationstart', (e) => {
  console.log('Animation started:', e.animationName);
  console.log('Elapsed time:', e.elapsedTime);
});

// Animation iterates (loops)
box.addEventListener('animationiteration', (e) => {
  console.log('Animation iteration:', e.animationName);
  console.log('Elapsed time:', e.elapsedTime);
});

// Animation ends
box.addEventListener('animationend', (e) => {
  console.log('Animation ended:', e.animationName);
  console.log('Elapsed time:', e.elapsedTime);
  
  // Clean up or trigger next animation
  box.classList.remove('animated');
});

// Animation cancels
box.addEventListener('animationcancel', (e) => {
  console.log('Animation cancelled:', e.animationName);
});

// Event properties
box.addEventListener('animationend', (e) => {
  e.animationName;  // Name of animation
  e.elapsedTime;    // Duration in seconds
  e.pseudoElement;  // '::before', '::after', or ''
});
</script>
```

```css
/* Example: chain animations */
.box {
  animation: slideIn 0.5s forwards;
}

.box.phase-2 {
  animation: fadeOut 0.5s forwards;
}
```

```javascript
const box = document.querySelector('.box');

box.addEventListener('animationend', (e) => {
  if (e.animationName === 'slideIn') {
    box.classList.add('phase-2');
  }
});
```

## Performance Considerations

### GPU-Accelerated Properties

```css
/* BEST: Composite layer properties */
@keyframes performant {
  from {
    transform: translateX(0) scale(1);
    opacity: 1;
  }
  to {
    transform: translateX(100px) scale(1.2);
    opacity: 0;
  }
}

/* Use transform instead of position */
/* Bad */
@keyframes slowSlide {
  from { left: 0; }
  to { left: 100px; }
}

/* Good */
@keyframes fastSlide {
  from { transform: translateX(0); }
  to { transform: translateX(100px); }
}

/* Use transform: scale instead of width/height */
/* Bad */
@keyframes slowGrow {
  from { width: 100px; height: 100px; }
  to { width: 200px; height: 200px; }
}

/* Good */
@keyframes fastGrow {
  from { transform: scale(1); }
  to { transform: scale(2); }
}
```

### will-change Optimization

```css
/* Hint upcoming animation */
.element {
  will-change: transform, opacity;
}

.element.animating {
  animation: slideIn 0.5s;
}

/* Add before animation, remove after */
.element:hover {
  will-change: transform;
  animation: bounce 0.5s;
}

/* Remove after animation */
.element.animation-done {
  will-change: auto;
}
```

### Performance Tips

```css
/* 1. Stick to transform and opacity */
@keyframes good {
  from { transform: translateX(0); opacity: 0; }
  to { transform: translateX(100px); opacity: 1; }
}

/* 2. Avoid animating layout properties */
/* Bad: triggers layout */
@keyframes bad {
  from { width: 100px; margin: 0; }
  to { width: 200px; margin: 20px; }
}

/* 3. Use shorter durations for perceived performance */
.snappy {
  animation: fadeIn 0.2s; /* Feels responsive */
}

/* 4. Reduce animation complexity on mobile */
@media (max-width: 768px) {
  .complex-animation {
    animation: simpleVersion 0.3s;
  }
}

/* 5. Respect prefers-reduced-motion */
@media (prefers-reduced-motion: reduce) {
  .animated {
    animation: none;
  }
}

/* 6. Use requestAnimationFrame in JavaScript */
/* For JavaScript-driven animations */
```

## Animation vs Transition

### When to Use Animation

```css
/* Auto-play on load */
.loader {
  animation: spin 1s linear infinite;
}

/* Multiple keyframes */
@keyframes complexSequence {
  0% { /* state 1 */ }
  33% { /* state 2 */ }
  66% { /* state 3 */ }
  100% { /* state 4 */ }
}

/* Looping */
.pulse {
  animation: pulse 2s infinite;
}

/* Self-reversing */
.pendulum {
  animation: swing 1s alternate infinite;
}
```

### When to Use Transition

```css
/* State changes */
.button {
  transition: background-color 0.2s;
}

.button:hover {
  background-color: blue;
}

/* Simple A to B */
.modal {
  opacity: 0;
  transition: opacity 0.3s;
}

.modal.open {
  opacity: 1;
}

/* Bidirectional (automatically reverses) */
.card {
  transform: scale(1);
  transition: transform 0.2s;
}

.card:hover {
  transform: scale(1.05);
  /* Automatically reverses on unhover */
}
```

### Comparison

```css
/* Transition: Triggered by state change */
.element {
  opacity: 0;
  transition: opacity 0.3s;
}

.element.visible {
  opacity: 1; /* Transition from 0 to 1 */
}

/* Animation: Defined sequence, auto-plays */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.element {
  animation: fadeIn 0.3s forwards;
}
```

## Common Animation Patterns

```css
/* Fade in */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* Slide in from left */
@keyframes slideInLeft {
  from {
    transform: translateX(-100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

/* Bounce */
@keyframes bounce {
  0%, 20%, 50%, 80%, 100% {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-30px);
  }
  60% {
    transform: translateY(-15px);
  }
}

/* Spin */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

/* Pulse */
@keyframes pulse {
  0%, 100% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.05);
    opacity: 0.8;
  }
}

/* Shake */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  10%, 30%, 50%, 70%, 90% { transform: translateX(-10px); }
  20%, 40%, 60%, 80% { transform: translateX(10px); }
}

/* Wiggle */
@keyframes wiggle {
  0%, 100% { transform: rotate(0deg); }
  25% { transform: rotate(-5deg); }
  75% { transform: rotate(5deg); }
}

/* Heartbeat */
@keyframes heartbeat {
  0%, 100% { transform: scale(1); }
  10%, 30% { transform: scale(1.1); }
  20%, 40% { transform: scale(1); }
}

/* Skeleton loading */
@keyframes shimmer {
  0% {
    background-position: -1000px 0;
  }
  100% {
    background-position: 1000px 0;
  }
}

.skeleton {
  background: linear-gradient(
    90deg,
    #f0f0f0 25%,
    #e0e0e0 50%,
    #f0f0f0 75%
  );
  background-size: 1000px 100%;
  animation: shimmer 2s infinite;
}

/* Progress bar */
@keyframes progress {
  from { width: 0%; }
  to { width: 100%; }
}

.progress-bar {
  animation: progress 3s forwards;
}

/* Typing effect */
@keyframes typing {
  from { width: 0; }
  to { width: 100%; }
}

@keyframes blink {
  50% { border-color: transparent; }
}

.typewriter {
  width: 0;
  overflow: hidden;
  border-right: 2px solid;
  white-space: nowrap;
  animation: 
    typing 3s steps(30) forwards,
    blink 0.5s step-end infinite;
}
```

## Gotchas

1. **Animation doesn't restart on class re-add**
   ```css
   /* Problem: Re-adding same class doesn't restart animation */
   .animate {
     animation: fadeIn 1s;
   }
   ```
   
   ```javascript
   // Solution: Force reflow
   element.classList.remove('animate');
   void element.offsetWidth; // Trigger reflow
   element.classList.add('animate');
   
   // Or use animation-name toggle
   element.style.animationName = 'none';
   setTimeout(() => {
     element.style.animationName = '';
   }, 10);
   ```

2. **Animation-fill-mode: forwards keeps last state**
   ```css
   /* Element stays at final keyframe */
   .element {
     animation: slideOut 0.5s forwards;
   }
   
   /* To reset */
   .element.reset {
     animation: none;
     /* Reset properties manually */
     transform: translateX(0);
   }
   ```

3. **Animations can conflict with transitions**
   ```css
   /* Animation takes precedence */
   .element {
     transition: transform 0.3s;
     animation: bounce 1s;
     /* Transition is ignored during animation */
   }
   ```

4. **Steps timing requires careful calculation**
   ```css
   /* Sprite animation: 10 frames, 1 second */
   @keyframes sprite {
     to { background-position-x: -1000px; }
   }
   
   .sprite {
     width: 100px;
     height: 100px;
     background-image: url('sprite.png');
     animation: sprite 1s steps(10) infinite;
     /* 10 frames = 10 steps */
   }
   ```

5. **Negative delay confusing behavior**
   ```css
   /* Starts animation 1 second in */
   .element {
     animation: fadeIn 2s -1s;
     /* Appears 50% faded in immediately */
   }
   ```

6. **will-change overuse**
   ```css
   /* Bad: All animated elements always */
   .animated {
     will-change: transform; /* Wastes memory */
   }
   
   /* Good: Only when animating */
   .animated.playing {
     will-change: transform;
   }
   ```

## Interview Questions

**Q: What's the difference between CSS animations and transitions?**

A: 
- Transitions: Triggered by state change, A to B, implicit reverse, simpler
- Animations: Auto-play, multiple keyframes, looping, more control, explicit sequence

Use transitions for interactive state changes (hover, focus), animations for complex sequences, loading states, or self-playing effects.

**Q: Explain animation-fill-mode and its values.**

A: Controls element styling before/after animation:
- `none`: No styles applied outside animation
- `forwards`: Retains final keyframe styles after animation
- `backwards`: Applies first keyframe styles during delay
- `both`: Applies both forwards and backwards

Most common: `forwards` to keep element in final state (modal fade-in stays visible).

**Q: How do you create performant CSS animations?**

A: 
1. Only animate `transform` and `opacity` (GPU-accelerated)
2. Avoid layout properties (width, height, margin, padding, top, left)
3. Use `will-change` temporarily before animation
4. Keep animations short (< 500ms typically)
5. Reduce complexity on mobile
6. Respect `prefers-reduced-motion`
7. Use `animation-play-state` to pause when off-screen
8. Profile with DevTools Performance tab

Key: `transform` and `opacity` trigger compositing, not layout/paint.

**Q: What are animation events and how do you use them?**

A: JavaScript events for animation lifecycle:
- `animationstart`: Fires when animation begins (after delay)
- `animationiteration`: Fires each loop iteration
- `animationend`: Fires when animation completes
- `animationcancel`: Fires if animation is cancelled

Use cases:
- Chain animations
- Clean up classes
- Track completion
- Trigger JavaScript actions

Event object provides `animationName`, `elapsedTime`, `pseudoElement`.

**Q: How do you restart a CSS animation?**

A: Several methods:
```javascript
// 1. Remove and re-add class with reflow
element.classList.remove('animate');
void element.offsetWidth; // Force reflow
element.classList.add('animate');

// 2. Toggle animation-name
element.style.animationName = 'none';
setTimeout(() => element.style.animationName = '', 10);

// 3. Clone and replace element
const clone = element.cloneNode(true);
element.parentNode.replaceChild(clone, element);
```

Most reliable: Method 1 with forced reflow.

**Q: Explain the difference between animation-direction values.**

A: 
- `normal`: Play forward (0% → 100%)
- `reverse`: Play backward (100% → 0%)
- `alternate`: Forward then backward (0% → 100% → 0%)
- `alternate-reverse`: Backward then forward (100% → 0% → 100%)

`alternate` is useful for pendulum/ping-pong effects with `infinite` iteration. Each full cycle with alternate counts as 2 iterations (forward + backward).

**Q: When should you use steps() timing function?**

A: Use for discrete frame-by-frame animations:
- Sprite animations (walking character)
- Flip clock numbers
- Loading indicators with distinct states
- Pixelated effects
- Typewriter text reveal

`steps(n)` divides animation into n equal jumps. Example: 10-frame sprite needs `steps(10)`.

**Q: How do you handle animation accessibility?**

A: 
```css
@media (prefers-reduced-motion: reduce) {
  .animated {
    animation: none;
    /* Or reduce duration */
    animation-duration: 0.01s;
  }
}
```

Also consider:
- Avoid flashing animations (seizure risk, < 3 flashes/sec)
- Provide pause controls for infinite animations
- Don't convey critical info through animation alone
- Test with screen readers
- Avoid parallax on `prefers-reduced-motion`

**Q: What's the performance difference between animating left and transform: translateX?**

A: Huge difference:
- `left`: Triggers layout → paint → composite (slow, janky)
- `transform: translateX`: Triggers composite only (fast, smooth)

`transform` creates a compositing layer (GPU-accelerated), doesn't affect document flow. `left` changes element position in layout, forcing browser to recalculate all affected elements.

Always use `transform` for position changes, `scale` for size, `rotate` for rotation.
