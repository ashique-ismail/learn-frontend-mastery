# CSS Transforms

## The Idea

**In plain English:** CSS transforms let you visually move, spin, resize, or tilt an element on the page without changing the layout around it. Think of it as picking up a sticker on a sheet of paper and repositioning or rotating it without shifting any of the other stickers.

**Real-world analogy:** Imagine you have a photo pinned to a corkboard. You can pull the pin out and slide the photo to a new spot, rotate it at an angle, stretch it bigger, or tilt it sideways — all without rearranging anything else on the board.

- The photo = the HTML element being transformed
- The corkboard = the page layout, which stays unchanged
- Sliding the photo = `translate()` (moving the element)
- Rotating the photo = `rotate()` (spinning the element)
- Stretching the photo = `scale()` (resizing the element)

---

## 2D Transforms

### translate()

Move elements horizontally and vertically.

```css
/* translateX - horizontal */
.element {
  transform: translateX(50px); /* Move 50px right */
  transform: translateX(-50px); /* Move 50px left */
  transform: translateX(50%); /* 50% of element's width */
}

/* translateY - vertical */
.element {
  transform: translateY(30px); /* Move 30px down */
  transform: translateY(-30px); /* Move 30px up */
  transform: translateY(50%); /* 50% of element's height */
}

/* translate - both axes */
.element {
  transform: translate(50px, 30px); /* x, y */
  transform: translate(50%, 50%); /* Percentages */
  transform: translate(-50%, -50%); /* Common centering trick */
}

/* Centering with translate */
.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* Responsive movement */
.responsive {
  transform: translateX(clamp(-100px, 5vw, 100px));
}

/* Performance: Use translate for position animation */
/* Bad */
.element {
  position: relative;
  left: 0;
  transition: left 0.3s;
}

.element:hover {
  left: 50px; /* Triggers layout */
}

/* Good */
.element {
  transform: translateX(0);
  transition: transform 0.3s;
}

.element:hover {
  transform: translateX(50px); /* Composite only */
}
```

### rotate()

Rotate elements around their origin.

```css
/* Degrees */
.element {
  transform: rotate(45deg);
  transform: rotate(-90deg); /* Counter-clockwise */
}

/* Turns */
.element {
  transform: rotate(0.25turn); /* 90 degrees */
  transform: rotate(1turn); /* 360 degrees */
}

/* Radians */
.element {
  transform: rotate(1.57rad); /* ~90 degrees */
}

/* Gradians */
.element {
  transform: rotate(100grad); /* 90 degrees */
}

/* Spinning animation */
@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.spinner {
  animation: spin 1s linear infinite;
}

/* Rotate on hover */
.icon {
  transition: transform 0.3s;
}

.icon:hover {
  transform: rotate(180deg);
}

/* Continuous rotation */
.rotate {
  transform: rotate(45deg);
  transform-origin: center; /* Rotation point */
}
```

### scale()

Change element size.

```css
/* Scale uniformly */
.element {
  transform: scale(1.5); /* 150% of original size */
  transform: scale(0.5); /* 50% of original size */
  transform: scale(2); /* Double size */
}

/* scaleX - horizontal only */
.element {
  transform: scaleX(2); /* Double width */
  transform: scaleX(0.5); /* Half width */
  transform: scaleX(-1); /* Mirror horizontally */
}

/* scaleY - vertical only */
.element {
  transform: scaleY(2); /* Double height */
  transform: scaleY(0.5); /* Half height */
  transform: scaleY(-1); /* Mirror vertically */
}

/* scale(x, y) - different axes */
.element {
  transform: scale(1.5, 0.8); /* Wider, shorter */
}

/* Zoom on hover */
.card {
  transition: transform 0.3s ease-out;
}

.card:hover {
  transform: scale(1.05);
}

/* Flip horizontal */
.flipped {
  transform: scaleX(-1);
}

/* Flip vertical */
.flipped {
  transform: scaleY(-1);
}

/* Pulse animation */
@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

.pulse {
  animation: pulse 1s infinite;
}

/* Performance: Use scale instead of width/height */
/* Bad */
.element {
  width: 100px;
  transition: width 0.3s;
}

.element:hover {
  width: 200px; /* Triggers layout */
}

/* Good */
.element {
  transition: transform 0.3s;
}

.element:hover {
  transform: scale(2); /* Composite only */
}
```

### skew()

Slant elements along axes.

```css
/* skewX - horizontal slant */
.element {
  transform: skewX(15deg); /* Slant right */
  transform: skewX(-15deg); /* Slant left */
}

/* skewY - vertical slant */
.element {
  transform: skewY(10deg); /* Slant down */
  transform: skewY(-10deg); /* Slant up */
}

/* skew(x, y) - both axes */
.element {
  transform: skew(15deg, 10deg);
}

/* Parallelogram effect */
.parallelogram {
  transform: skewX(-20deg);
}

/* Undo skew for child content */
.parallelogram-container {
  transform: skewX(-20deg);
}

.parallelogram-content {
  transform: skewX(20deg); /* Counter-skew */
}

/* Subtle perspective effect */
.card {
  transition: transform 0.3s;
}

.card:hover {
  transform: skewY(-2deg);
}
```

### Combining 2D Transforms

```css
/* Multiple transforms - order matters! */
.element {
  /* Applied right to left: scale, then rotate, then translate */
  transform: translate(50px, 0) rotate(45deg) scale(1.2);
}

/* Different order = different result */
.rotate-then-translate {
  transform: translate(50px, 0) rotate(45deg);
  /* Rotates, then moves 50px right */
}

.translate-then-rotate {
  transform: rotate(45deg) translate(50px, 0);
  /* Moves 50px right, then rotates (moves diagonally) */
}

/* Common pattern: scale + rotate */
.element:hover {
  transform: scale(1.1) rotate(5deg);
}

/* Lift and grow */
.card:hover {
  transform: translateY(-5px) scale(1.02);
}

/* Complex hover effect */
.button {
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.button:hover {
  transform: translateY(-2px) scale(1.05) rotate(-1deg);
}

/* Animation with multiple transforms */
@keyframes complexMove {
  0% {
    transform: translate(0, 0) rotate(0deg) scale(1);
  }
  50% {
    transform: translate(100px, -50px) rotate(180deg) scale(1.5);
  }
  100% {
    transform: translate(0, 0) rotate(360deg) scale(1);
  }
}
```

## 3D Transforms

### Perspective

Create 3D space depth.

```css
/* Parent perspective - affects all children */
.container {
  perspective: 1000px; /* Viewer distance (lower = more dramatic) */
}

.child {
  transform: rotateY(45deg); /* Uses parent's perspective */
}

/* Individual perspective - affects single element */
.element {
  transform: perspective(1000px) rotateY(45deg);
}

/* Perspective origin - viewer position */
.container {
  perspective: 1000px;
  perspective-origin: center center; /* Default */
  perspective-origin: top left; /* View from top-left */
  perspective-origin: 50% 50%; /* Center */
}

/* Common values */
.subtle-3d {
  perspective: 2000px; /* Subtle depth */
}

.dramatic-3d {
  perspective: 500px; /* Dramatic depth */
}

/* Card flip example */
.card-container {
  perspective: 1000px;
}

.card {
  transition: transform 0.6s;
  transform-style: preserve-3d;
}

.card-container:hover .card {
  transform: rotateY(180deg);
}
```

### translateZ()

Move elements in 3D space (depth).

```css
/* Move toward viewer */
.element {
  transform: translateZ(100px); /* Closer (larger) */
}

/* Move away from viewer */
.element {
  transform: translateZ(-100px); /* Farther (smaller) */
}

/* translate3d(x, y, z) */
.element {
  transform: translate3d(50px, 30px, 100px);
}

/* Force hardware acceleration */
.element {
  transform: translateZ(0); /* Creates compositing layer */
  /* or */
  transform: translate3d(0, 0, 0);
}

/* Layered card effect */
.container {
  perspective: 1000px;
}

.layer-1 {
  transform: translateZ(0px);
}

.layer-2 {
  transform: translateZ(50px);
}

.layer-3 {
  transform: translateZ(100px);
}

/* Parallax effect */
.parallax-container {
  perspective: 1px;
  height: 100vh;
  overflow-y: auto;
}

.parallax-layer {
  transform: translateZ(-2px) scale(3);
}
```

### rotateX() / rotateY() / rotateZ()

Rotate in 3D space.

```css
/* rotateX - around horizontal axis (pitch) */
.element {
  transform: rotateX(45deg); /* Flip forward */
  transform: rotateX(-45deg); /* Flip backward */
}

/* rotateY - around vertical axis (yaw) */
.element {
  transform: rotateY(45deg); /* Turn right */
  transform: rotateY(-45deg); /* Turn left */
}

/* rotateZ - around z-axis (roll) - same as rotate() */
.element {
  transform: rotateZ(45deg);
  /* Equivalent to rotate(45deg) */
}

/* rotate3d(x, y, z, angle) - custom axis */
.element {
  transform: rotate3d(1, 1, 0, 45deg); /* Diagonal axis */
}

/* Card flip animation */
@keyframes flipCard {
  from {
    transform: rotateY(0deg);
  }
  to {
    transform: rotateY(180deg);
  }
}

.card {
  transition: transform 0.6s;
  transform-style: preserve-3d;
}

.card:hover {
  animation: flipCard 0.6s forwards;
}

/* 3D cube faces */
.cube-face-front {
  transform: rotateY(0deg) translateZ(100px);
}

.cube-face-back {
  transform: rotateY(180deg) translateZ(100px);
}

.cube-face-right {
  transform: rotateY(90deg) translateZ(100px);
}

.cube-face-left {
  transform: rotateY(-90deg) translateZ(100px);
}

.cube-face-top {
  transform: rotateX(90deg) translateZ(100px);
}

.cube-face-bottom {
  transform: rotateX(-90deg) translateZ(100px);
}
```

### scale3d()

Scale in 3D space.

```css
/* scale3d(x, y, z) */
.element {
  transform: scale3d(1.5, 1.5, 2);
}

/* scaleZ - depth scaling (requires perspective) */
.element {
  transform: perspective(500px) scaleZ(2);
}

/* Combine with rotation for effects */
.element {
  transform: perspective(1000px) rotateX(45deg) scaleZ(2);
}
```

## transform-origin

Set the pivot point for transformations.

```css
/* Default: center center */
.element {
  transform-origin: center center; /* 50% 50% */
}

/* Keywords */
.element {
  transform-origin: top left;
  transform-origin: bottom right;
  transform-origin: center top;
}

/* Percentages */
.element {
  transform-origin: 50% 50%; /* Center */
  transform-origin: 0% 0%; /* Top-left */
  transform-origin: 100% 100%; /* Bottom-right */
}

/* Lengths */
.element {
  transform-origin: 20px 40px;
}

/* 3D origin */
.element {
  transform-origin: 50% 50% 0; /* x, y, z */
  transform-origin: center center 100px;
}

/* Rotate from different points */
.rotate-top-left {
  transform-origin: top left;
  transform: rotate(45deg);
}

.rotate-bottom-right {
  transform-origin: bottom right;
  transform: rotate(45deg);
}

/* Scale from corner */
.scale-corner {
  transform-origin: top left;
  transform: scale(1.5);
}

/* Door swing effect */
.door {
  transform-origin: left center; /* Hinge on left */
  transition: transform 0.5s;
}

.door.open {
  transform: rotateY(-90deg);
}

/* Clock hands */
.hour-hand {
  transform-origin: 50% 100%; /* Bottom center */
  transform: rotate(90deg);
}

.minute-hand {
  transform-origin: 50% 100%;
  transform: rotate(180deg);
}
```

## transform-style

Control 3D rendering for children.

```css
/* flat - children rendered in parent's plane (default) */
.element {
  transform-style: flat;
}

/* preserve-3d - children maintain 3D position */
.element {
  transform-style: preserve-3d;
}

/* Card flip example */
.card-container {
  perspective: 1000px;
}

.card {
  transform-style: preserve-3d; /* Children maintain 3D space */
  transition: transform 0.6s;
}

.card-front,
.card-back {
  backface-visibility: hidden;
}

.card-back {
  transform: rotateY(180deg);
}

.card-container:hover .card {
  transform: rotateY(180deg);
}

/* 3D cube requires preserve-3d */
.cube {
  transform-style: preserve-3d;
  animation: rotateCube 10s infinite linear;
}

@keyframes rotateCube {
  from { transform: rotateX(0deg) rotateY(0deg); }
  to { transform: rotateX(360deg) rotateY(360deg); }
}

/* Nested 3D transforms */
.parent {
  transform-style: preserve-3d;
  transform: rotateY(45deg);
}

.child {
  transform: rotateX(45deg); /* Combines with parent */
}
```

## backface-visibility

Control whether back of element is visible.

```css
/* visible - show back side (default) */
.element {
  backface-visibility: visible;
}

/* hidden - hide back side */
.element {
  backface-visibility: hidden;
}

/* Card flip - classic use case */
.card-front,
.card-back {
  backface-visibility: hidden;
  position: absolute;
  width: 100%;
  height: 100%;
}

.card-back {
  transform: rotateY(180deg);
}

.card.flipped {
  transform: rotateY(180deg);
}

/* Prevent flickering during rotation */
.rotating-element {
  backface-visibility: hidden;
  transform: rotateY(0deg);
  transition: transform 0.5s;
}

/* Performance optimization */
.element {
  backface-visibility: hidden; /* Can improve rendering */
}
```

## Matrix Functions

Low-level transform control (rarely used directly).

```css
/* matrix(a, b, c, d, tx, ty) - 2D transform */
.element {
  /* matrix(scaleX, skewY, skewX, scaleY, translateX, translateY) */
  transform: matrix(1.5, 0, 0, 1.5, 50, 30);
  /* Equivalent to: scale(1.5) translate(50px, 30px) */
}

/* matrix3d() - 3D transform (16 values) */
.element {
  transform: matrix3d(
    1, 0, 0, 0,
    0, 1, 0, 0,
    0, 0, 1, 0,
    0, 0, 0, 1
  );
  /* Identity matrix (no transformation) */
}

/* Usually generated by JavaScript or calculated */
/* Not typically written by hand */
```

## Performance Best Practices

```css
/* 1. Use transform instead of position properties */
/* Bad */
.element {
  position: relative;
  left: 0;
  transition: left 0.3s;
}

/* Good */
.element {
  transform: translateX(0);
  transition: transform 0.3s;
}

/* 2. Use will-change for complex animations */
.animating {
  will-change: transform;
}

/* 3. Force hardware acceleration */
.accelerated {
  transform: translateZ(0);
  /* or */
  transform: translate3d(0, 0, 0);
}

/* 4. Use transform over multiple properties */
/* Bad */
.element {
  transition: width 0.3s, height 0.3s;
}

/* Good */
.element {
  transition: transform 0.3s;
}

.element:hover {
  transform: scale(1.2);
}

/* 5. Combine transforms in single property */
/* Good */
.element {
  transform: translateX(50px) rotate(45deg) scale(1.2);
}

/* Bad (only last one applies) */
.element {
  transform: translateX(50px);
  transform: rotate(45deg); /* Overrides translateX */
}

/* 6. Use 3D transforms for smoother animation */
/* Better performance */
.element {
  transform: translate3d(50px, 0, 0);
}

/* vs 2D */
.element {
  transform: translateX(50px);
}
```

## Common Patterns

```css
/* Center element */
.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* Lift on hover */
.card:hover {
  transform: translateY(-5px);
}

/* Zoom on hover */
.image:hover {
  transform: scale(1.1);
}

/* Flip card */
.card-container {
  perspective: 1000px;
}

.card {
  transform-style: preserve-3d;
  transition: transform 0.6s;
}

.card-front,
.card-back {
  backface-visibility: hidden;
}

.card-back {
  transform: rotateY(180deg);
}

.card-container:hover .card {
  transform: rotateY(180deg);
}

/* 3D button press */
.button {
  transform: translateY(0);
  transition: transform 0.1s;
}

.button:active {
  transform: translateY(2px);
}

/* Spinning loader */
.loader {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Parallax layers */
.parallax-container {
  perspective: 1px;
  height: 100vh;
  overflow-y: auto;
}

.parallax-back {
  transform: translateZ(-1px) scale(2);
}

.parallax-front {
  transform: translateZ(0);
}

/* 3D cube */
.cube {
  width: 200px;
  height: 200px;
  position: relative;
  transform-style: preserve-3d;
  animation: rotateCube 10s infinite linear;
}

.cube-face {
  position: absolute;
  width: 200px;
  height: 200px;
}

.cube-front { transform: rotateY(0deg) translateZ(100px); }
.cube-back { transform: rotateY(180deg) translateZ(100px); }
.cube-right { transform: rotateY(90deg) translateZ(100px); }
.cube-left { transform: rotateY(-90deg) translateZ(100px); }
.cube-top { transform: rotateX(90deg) translateZ(100px); }
.cube-bottom { transform: rotateX(-90deg) translateZ(100px); }

@keyframes rotateCube {
  to { transform: rotateX(360deg) rotateY(360deg); }
}
```

## Gotchas

1. **Transform order matters**
   ```css
   /* Different results */
   .a { transform: rotate(45deg) translateX(100px); }
   .b { transform: translateX(100px) rotate(45deg); }
   /* Applied right to left, so order affects final position */
   ```

2. **Transforms create stacking context**
   ```css
   /* Creates new stacking context */
   .element {
     transform: translateZ(0);
     /* z-index now isolated from parent context */
   }
   ```

3. **Percentage translates based on element size**
   ```css
   /* translateX(50%) moves 50% of element's width, not container */
   .element {
     width: 100px;
     transform: translateX(50%); /* Moves 50px, not 50% of parent */
   }
   ```

4. **Transform overrides previous transform**
   ```css
   /* Only rotate applies */
   .element {
     transform: translateX(50px);
     transform: rotate(45deg); /* Overwrites translateX */
   }
   
   /* Combine in single property */
   .element {
     transform: translateX(50px) rotate(45deg);
   }
   ```

5. **Perspective must be on parent for child 3D transforms**
   ```css
   /* Doesn't work */
   .child {
     perspective: 1000px;
     transform: rotateY(45deg);
   }
   
   /* Works */
   .parent {
     perspective: 1000px;
   }
   
   .child {
     transform: rotateY(45deg);
   }
   
   /* Alternative: inline perspective */
   .child {
     transform: perspective(1000px) rotateY(45deg);
   }
   ```

6. **Text rendering with transforms can blur**
   ```css
   /* Can cause blurry text */
   .element {
     transform: translateZ(0);
   }
   
   /* Fix with */
   .element {
     transform: translateZ(0);
     -webkit-font-smoothing: antialiased;
   }
   ```

## Interview Questions

**Q: Why should you use transform instead of changing position properties like left/top?**

A: Transform is GPU-accelerated and triggers only compositing, while left/top trigger layout recalculation and paint. Transform changes happen on the compositor thread, avoiding main thread work. This results in smoother 60fps animations. Use `transform: translate()` instead of left/top, `scale()` instead of width/height.

**Q: Explain how transform-origin works.**

A: `transform-origin` sets the pivot point for transformations. Default is `center center` (50% 50%). For example, `rotate()` spins around this point. Setting `transform-origin: top left` makes rotation pivot from top-left corner. Useful for door swings (left/right), dropdown menus (top), clock hands (bottom center).

**Q: What's the difference between transform-style: flat and preserve-3d?**

A: 
- `flat` (default): Children are flattened into parent's 2D plane, losing 3D positioning
- `preserve-3d`: Children maintain their 3D transforms relative to parent

Required for nested 3D transforms like card flips with front/back faces or 3D cubes where each face needs independent 3D positioning.

**Q: How does perspective work in CSS 3D transforms?**

A: Perspective defines the viewer's distance from the z=0 plane, creating depth perception. Lower values (500px) create dramatic perspective, higher values (2000px) create subtle depth. Applied two ways:
1. Parent `perspective` property (affects all children)
2. Individual `transform: perspective()` function

Also controlled by `perspective-origin` to set viewer position.

**Q: What is backface-visibility and when do you use it?**

A: Controls whether the back face of a rotated element is visible. Values:
- `visible` (default): Show back (reversed)
- `hidden`: Hide back

Essential for card flip effects where you want front visible from 0-90deg, back visible from 90-180deg. Prevents seeing reversed text/images when element rotates past 90deg. Also prevents flickering in 3D animations.

**Q: How do you create a performant hover lift effect?**

A: Use `transform: translateY()` instead of `top/bottom`:
```css
.card {
  transition: transform 0.2s ease-out;
}

.card:hover {
  transform: translateY(-5px);
}
```

Benefits: GPU-accelerated, no layout recalculation, smooth 60fps animation. Add `will-change: transform` for complex animations.

**Q: What's the difference between rotate() and rotateZ()?**

A: They're equivalent. `rotate()` is 2D shorthand for `rotateZ()`. Both rotate around the z-axis (perpendicular to screen). For 3D rotations, use:
- `rotateX()`: Flip forward/backward (pitch)
- `rotateY()`: Turn left/right (yaw)  
- `rotateZ()` or `rotate()`: Spin clockwise/counter-clockwise (roll)

**Q: Why might translateZ(0) improve performance?**

A: `translateZ(0)` forces creation of a compositing layer (GPU-accelerated), even without actual z-axis movement. This moves the element to GPU for independent rendering, reducing main thread work. However, overusing wastes GPU memory. Only apply to elements that will animate or need smooth repainting.

**Q: How do percentage values work in translate()?**

A: Percentages are based on the element's own size, not the parent:
- `translateX(50%)`: Moves 50% of element's width
- `translateY(50%)`: Moves 50% of element's height

This is why `translate(-50%, -50%)` centers elements - it moves the element back by half its own dimensions. Differs from `left: 50%` which is based on parent size.

**Q: What happens when you combine multiple transform functions?**

A: They're applied right-to-left (composed):
```css
transform: translate(50px, 0) rotate(45deg) scale(1.5);
/* 1. Scale to 1.5x
   2. Rotate 45deg
   3. Move 50px right */
```

Order matters! `rotate(45deg) translateX(100px)` moves diagonally, while `translateX(100px) rotate(45deg)` moves horizontally then rotates. Each transform affects the coordinate system for subsequent transforms.
