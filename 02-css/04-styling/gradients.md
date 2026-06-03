# CSS Gradients

## Linear Gradients

Gradients that transition colors along a straight line.

### Basic Syntax

```css
/* Default: top to bottom */
.element {
  background: linear-gradient(red, blue);
}

/* Two colors */
.element {
  background: linear-gradient(#ff0000, #0000ff);
}

/* Three or more colors */
.element {
  background: linear-gradient(red, yellow, green, blue);
}

/* With transparency */
.element {
  background: linear-gradient(
    rgba(255, 0, 0, 1),
    rgba(255, 0, 0, 0)
  );
}
```

### Direction Keywords

```css
/* To direction */
.to-right {
  background: linear-gradient(to right, red, blue);
}

.to-left {
  background: linear-gradient(to left, red, blue);
}

.to-top {
  background: linear-gradient(to top, red, blue);
}

.to-bottom {
  background: linear-gradient(to bottom, red, blue); /* Default */
}

/* Diagonal */
.to-top-right {
  background: linear-gradient(to top right, red, blue);
}

.to-bottom-left {
  background: linear-gradient(to bottom left, red, blue);
}

.to-top-left {
  background: linear-gradient(to top left, red, blue);
}

.to-bottom-right {
  background: linear-gradient(to bottom right, red, blue);
}
```

### Angle Values

```css
/* Degrees - 0deg is bottom to top, 90deg is left to right */
.element {
  background: linear-gradient(0deg, red, blue); /* Bottom to top */
  background: linear-gradient(45deg, red, blue); /* Diagonal */
  background: linear-gradient(90deg, red, blue); /* Left to right */
  background: linear-gradient(180deg, red, blue); /* Top to bottom */
  background: linear-gradient(270deg, red, blue); /* Right to left */
}

/* Negative angles */
.element {
  background: linear-gradient(-45deg, red, blue);
}

/* Other units */
.element {
  background: linear-gradient(0.25turn, red, blue); /* 90deg */
  background: linear-gradient(1.57rad, red, blue); /* ~90deg */
  background: linear-gradient(100grad, red, blue); /* 90deg */
}

/* Common angles */
.diagonal-45 {
  background: linear-gradient(45deg, #667eea, #764ba2);
}

.steep {
  background: linear-gradient(60deg, #f093fb, #f5576c);
}
```

### Color Stops

Control where colors begin and end.

```css
/* Percentage stops */
.element {
  background: linear-gradient(
    red 0%,
    yellow 50%,
    blue 100%
  );
}

/* Pixel stops */
.element {
  background: linear-gradient(
    red 0px,
    yellow 100px,
    blue 200px
  );
}

/* Multiple stops per color */
.element {
  background: linear-gradient(
    red 0%,
    red 25%,
    blue 25%,
    blue 50%,
    yellow 50%,
    yellow 75%,
    green 75%,
    green 100%
  );
}

/* Hard color stops (no blend) */
.sharp {
  background: linear-gradient(
    red 50%,
    blue 50%
  ); /* Sharp line at 50% */
}

/* Smooth transition */
.smooth {
  background: linear-gradient(
    red 0%,
    blue 100%
  );
}

/* Uneven distribution */
.uneven {
  background: linear-gradient(
    red 0%,
    red 10%,
    yellow 10%,
    yellow 90%,
    blue 90%,
    blue 100%
  );
}

/* Overlapping stops for blending zones */
.blend-zones {
  background: linear-gradient(
    red 0%,
    red 30%,
    blue 70%,
    blue 100%
  ); /* Blend in middle 30-70% */
}
```

## Radial Gradients

Gradients that radiate from a center point.

### Basic Syntax

```css
/* Default: circle from center */
.element {
  background: radial-gradient(red, blue);
}

/* Multiple colors */
.element {
  background: radial-gradient(red, yellow, green, blue);
}
```

### Shape

```css
/* Circle */
.circle {
  background: radial-gradient(circle, red, blue);
}

/* Ellipse (default) */
.ellipse {
  background: radial-gradient(ellipse, red, blue);
}

/* Explicit dimensions */
.sized {
  background: radial-gradient(circle 100px, red, blue);
  background: radial-gradient(ellipse 200px 100px, red, blue);
}
```

### Size Keywords

```css
/* closest-side - gradient ends at closest edge */
.closest-side {
  background: radial-gradient(
    circle closest-side,
    red, blue
  );
}

/* closest-corner - gradient ends at closest corner */
.closest-corner {
  background: radial-gradient(
    circle closest-corner,
    red, blue
  );
}

/* farthest-side - gradient ends at farthest edge */
.farthest-side {
  background: radial-gradient(
    circle farthest-side,
    red, blue
  );
}

/* farthest-corner - gradient ends at farthest corner (default) */
.farthest-corner {
  background: radial-gradient(
    circle farthest-corner,
    red, blue
  );
}

/* Common use: spotlight effect */
.spotlight {
  background: radial-gradient(
    circle closest-side,
    rgba(255, 255, 255, 0.8),
    rgba(255, 255, 255, 0)
  );
}
```

### Position

```css
/* Position with keywords */
.element {
  background: radial-gradient(
    circle at center,
    red, blue
  ); /* Default */
}

.top-left {
  background: radial-gradient(
    circle at top left,
    red, blue
  );
}

.bottom-right {
  background: radial-gradient(
    circle at bottom right,
    red, blue
  );
}

/* Position with percentages */
.element {
  background: radial-gradient(
    circle at 50% 50%,
    red, blue
  );
}

.custom-position {
  background: radial-gradient(
    circle at 30% 70%,
    red, blue
  );
}

/* Position with lengths */
.element {
  background: radial-gradient(
    circle at 100px 50px,
    red, blue
  );
}

/* Combine size and position */
.spotlight {
  background: radial-gradient(
    circle 200px at center,
    white, transparent
  );
}

.corner-glow {
  background: radial-gradient(
    circle 300px at top left,
    rgba(255, 0, 0, 0.3),
    transparent
  );
}
```

### Color Stops

```css
/* With color stops */
.element {
  background: radial-gradient(
    circle,
    red 0%,
    yellow 50%,
    blue 100%
  );
}

/* Hard stops */
.concentric {
  background: radial-gradient(
    circle,
    red 0%,
    red 30%,
    yellow 30%,
    yellow 60%,
    blue 60%,
    blue 100%
  );
}

/* Glow effect */
.glow {
  background: radial-gradient(
    circle at center,
    rgba(255, 255, 255, 1) 0%,
    rgba(255, 255, 255, 0.5) 30%,
    rgba(255, 255, 255, 0) 70%
  );
}
```

## Conic Gradients

Gradients that rotate around a center point.

### Basic Syntax

```css
/* Default: starts at top, goes clockwise */
.element {
  background: conic-gradient(red, yellow, green, blue, red);
}

/* Two colors */
.element {
  background: conic-gradient(red, blue);
}
```

### Starting Angle

```css
/* from angle */
.element {
  background: conic-gradient(
    from 0deg,
    red, blue
  ); /* Start at top (default) */
}

.rotated {
  background: conic-gradient(
    from 90deg,
    red, blue
  ); /* Start at right */
}

.custom-angle {
  background: conic-gradient(
    from 45deg,
    red, yellow, green, blue, red
  );
}
```

### Position

```css
/* at position */
.element {
  background: conic-gradient(
    at center,
    red, blue
  ); /* Default */
}

.off-center {
  background: conic-gradient(
    at 30% 70%,
    red, blue
  );
}

.corner {
  background: conic-gradient(
    at top left,
    red, blue
  );
}

/* Combine angle and position */
.custom {
  background: conic-gradient(
    from 45deg at 30% 30%,
    red, yellow, green, blue, red
  );
}
```

### Color Stops

```css
/* Angle stops */
.element {
  background: conic-gradient(
    red 0deg,
    yellow 90deg,
    green 180deg,
    blue 270deg,
    red 360deg
  );
}

/* Percentage stops */
.element {
  background: conic-gradient(
    red 0%,
    yellow 25%,
    green 50%,
    blue 75%,
    red 100%
  );
}

/* Turn stops */
.element {
  background: conic-gradient(
    red 0turn,
    yellow 0.25turn,
    green 0.5turn,
    blue 0.75turn,
    red 1turn
  );
}

/* Hard stops */
.pie-chart {
  background: conic-gradient(
    red 0deg,
    red 120deg,
    yellow 120deg,
    yellow 240deg,
    blue 240deg,
    blue 360deg
  );
}

/* Checkerboard pattern */
.checkerboard {
  background: conic-gradient(
    from 45deg,
    black 0deg,
    black 90deg,
    white 90deg,
    white 180deg,
    black 180deg,
    black 270deg,
    white 270deg,
    white 360deg
  );
}
```

## Repeating Gradients

### repeating-linear-gradient()

```css
/* Basic stripes */
.stripes {
  background: repeating-linear-gradient(
    45deg,
    red,
    red 10px,
    white 10px,
    white 20px
  );
}

/* Vertical stripes */
.vertical-stripes {
  background: repeating-linear-gradient(
    90deg,
    #667eea 0px,
    #667eea 20px,
    #764ba2 20px,
    #764ba2 40px
  );
}

/* Diagonal pattern */
.diagonal {
  background: repeating-linear-gradient(
    45deg,
    transparent,
    transparent 10px,
    rgba(0, 0, 0, 0.1) 10px,
    rgba(0, 0, 0, 0.1) 20px
  );
}

/* Barber pole */
.barber-pole {
  background: repeating-linear-gradient(
    -45deg,
    red 0px,
    red 20px,
    white 20px,
    white 40px,
    blue 40px,
    blue 60px
  );
}

/* Animated stripes */
@keyframes move-stripes {
  to {
    background-position: 40px 0;
  }
}

.animated-stripes {
  background: repeating-linear-gradient(
    45deg,
    red 0px,
    red 10px,
    white 10px,
    white 20px
  );
  background-size: 40px 40px;
  animation: move-stripes 1s linear infinite;
}
```

### repeating-radial-gradient()

```css
/* Concentric circles */
.circles {
  background: repeating-radial-gradient(
    circle,
    red,
    red 10px,
    white 10px,
    white 20px
  );
}

/* Target pattern */
.target {
  background: repeating-radial-gradient(
    circle at center,
    red 0px,
    red 20px,
    white 20px,
    white 40px
  );
}

/* Ripple effect */
.ripple {
  background: repeating-radial-gradient(
    circle at 50% 50%,
    rgba(0, 0, 0, 0.05),
    rgba(0, 0, 0, 0.05) 10px,
    transparent 10px,
    transparent 20px
  );
}
```

### repeating-conic-gradient()

```css
/* Spinning pattern */
.spokes {
  background: repeating-conic-gradient(
    black 0deg,
    black 10deg,
    white 10deg,
    white 20deg
  );
}

/* Color wheel sections */
.color-wheel {
  background: repeating-conic-gradient(
    from 0deg,
    red 0deg,
    red 30deg,
    yellow 30deg,
    yellow 60deg
  );
}

/* Starburst */
.starburst {
  background: repeating-conic-gradient(
    from 0deg,
    rgba(255, 255, 255, 0.1) 0deg,
    rgba(255, 255, 255, 0.1) 5deg,
    transparent 5deg,
    transparent 10deg
  );
}
```

## Multiple Gradients

```css
/* Layered gradients */
.layered {
  background:
    linear-gradient(
      to right,
      rgba(255, 0, 0, 0.5),
      rgba(0, 0, 255, 0.5)
    ),
    linear-gradient(
      to bottom,
      rgba(0, 255, 0, 0.5),
      rgba(255, 255, 0, 0.5)
    );
}

/* Gradient over image */
.image-overlay {
  background:
    linear-gradient(
      to bottom,
      rgba(0, 0, 0, 0),
      rgba(0, 0, 0, 0.7)
    ),
    url('image.jpg');
  background-size: cover;
}

/* Multiple color overlays */
.complex {
  background:
    radial-gradient(
      circle at top left,
      rgba(255, 0, 0, 0.3),
      transparent
    ),
    radial-gradient(
      circle at bottom right,
      rgba(0, 0, 255, 0.3),
      transparent
    ),
    linear-gradient(
      to bottom,
      #f0f0f0,
      #ffffff
    );
}

/* Spotlight effect */
.spotlights {
  background:
    radial-gradient(
      circle 200px at 20% 30%,
      rgba(255, 255, 255, 0.3),
      transparent
    ),
    radial-gradient(
      circle 150px at 80% 70%,
      rgba(255, 255, 255, 0.2),
      transparent
    ),
    #1a1a1a;
}
```

## Color Spaces in Gradients

```css
/* sRGB (default) */
.srgb {
  background: linear-gradient(red, blue);
}

/* oklch - perceptually uniform */
.oklch {
  background: linear-gradient(
    in oklch,
    oklch(0.6 0.2 0),
    oklch(0.6 0.2 180)
  );
}

/* oklab - perceptually uniform */
.oklab {
  background: linear-gradient(
    in oklab,
    oklab(0.6 0.2 0),
    oklab(0.6 -0.2 0)
  );
}

/* hsl - by hue */
.hsl {
  background: linear-gradient(
    in hsl,
    hsl(0 100% 50%),
    hsl(180 100% 50%)
  );
}

/* Longer/shorter hue interpolation */
.longer-hue {
  background: linear-gradient(
    in hsl longer hue,
    red, blue
  );
}

.shorter-hue {
  background: linear-gradient(
    in hsl shorter hue,
    red, blue
  );
}
```

## Gradient Patterns

```css
/* Stripes */
.stripes {
  background: linear-gradient(
    90deg,
    #fff 0%,
    #fff 50%,
    #f0f0f0 50%,
    #f0f0f0 100%
  );
  background-size: 40px 100%;
}

/* Checkerboard */
.checkerboard {
  background:
    linear-gradient(45deg, #ddd 25%, transparent 25%),
    linear-gradient(-45deg, #ddd 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, #ddd 75%),
    linear-gradient(-45deg, transparent 75%, #ddd 75%);
  background-size: 20px 20px;
  background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
}

/* Dots */
.dots {
  background:
    radial-gradient(circle, black 1px, transparent 1px);
  background-size: 20px 20px;
}

/* Grid */
.grid {
  background:
    linear-gradient(rgba(0, 0, 0, 0.1) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 0, 0, 0.1) 1px, transparent 1px);
  background-size: 20px 20px;
}

/* Carbon fiber */
.carbon {
  background:
    radial-gradient(circle, transparent 20%, black 20%, black 80%, transparent 80%),
    radial-gradient(circle, transparent 20%, black 20%, black 80%, transparent 80%) 25px 25px,
    linear-gradient(#1a1a1a 2px, transparent 2px) 0 -1px,
    linear-gradient(90deg, #1a1a1a 2px, transparent 2px) -1px 0;
  background-color: #282828;
  background-size: 50px 50px, 50px 50px, 25px 25px, 25px 25px;
}

/* Wave pattern */
.waves {
  background:
    repeating-linear-gradient(
      45deg,
      transparent,
      transparent 10px,
      rgba(0, 0, 0, 0.05) 10px,
      rgba(0, 0, 0, 0.05) 20px
    );
}
```

## Common Use Cases

```css
/* Hero section overlay */
.hero {
  background:
    linear-gradient(
      to bottom,
      rgba(0, 0, 0, 0.3),
      rgba(0, 0, 0, 0.7)
    ),
    url('hero.jpg') center/cover;
}

/* Gradient text */
.gradient-text {
  background: linear-gradient(
    45deg,
    #667eea,
    #764ba2
  );
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

/* Button gradient */
.gradient-button {
  background: linear-gradient(
    135deg,
    #667eea 0%,
    #764ba2 100%
  );
  transition: opacity 0.3s;
}

.gradient-button:hover {
  opacity: 0.9;
}

/* Loading bar */
@keyframes loading {
  0% {
    background-position: 0 0;
  }
  100% {
    background-position: 40px 0;
  }
}

.loading-bar {
  background: repeating-linear-gradient(
    -45deg,
    #667eea,
    #667eea 10px,
    #764ba2 10px,
    #764ba2 20px
  );
  animation: loading 1s linear infinite;
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

/* Mesh gradient background */
.mesh-gradient {
  background:
    radial-gradient(at 0% 0%, #667eea 0px, transparent 50%),
    radial-gradient(at 100% 0%, #764ba2 0px, transparent 50%),
    radial-gradient(at 100% 100%, #f093fb 0px, transparent 50%),
    radial-gradient(at 0% 100%, #4facfe 0px, transparent 50%);
  background-color: #fff;
}

/* Glass morphism with gradient border */
.glass-card {
  background: linear-gradient(
    135deg,
    rgba(255, 255, 255, 0.1),
    rgba(255, 255, 255, 0.05)
  );
  backdrop-filter: blur(10px);
  border: 1px solid;
  border-image: linear-gradient(
    135deg,
    rgba(255, 255, 255, 0.3),
    rgba(255, 255, 255, 0.1)
  ) 1;
}
```

## Performance Tips

```css
/* 1. Use solid colors when possible */
/* Faster */
.solid {
  background: #667eea;
}

/* Slower */
.gradient {
  background: linear-gradient(#667eea, #764ba2);
}

/* 2. Limit gradient complexity */
/* Simple gradient - better performance */
.simple {
  background: linear-gradient(#667eea, #764ba2);
}

/* Complex gradient - worse performance */
.complex {
  background: linear-gradient(
    45deg,
    red 0%, orange 10%, yellow 20%, green 30%,
    blue 40%, indigo 50%, violet 60%, red 70%
  );
}

/* 3. Use background-size for patterns instead of many stops */
.efficient-stripes {
  background: linear-gradient(90deg, #fff 50%, #000 50%);
  background-size: 20px 100%;
}

/* 4. Avoid animating gradients directly */
/* Bad - very expensive */
@keyframes badGradient {
  from { background: linear-gradient(red, blue); }
  to { background: linear-gradient(blue, red); }
}

/* Good - animate position or opacity */
@keyframes goodGradient {
  from { background-position: 0 0; }
  to { background-position: 100% 0; }
}

/* 5. Use CSS custom properties for dynamic gradients */
.dynamic {
  --color-start: #667eea;
  --color-end: #764ba2;
  background: linear-gradient(var(--color-start), var(--color-end));
}
```

## Gotchas

1. **Gradient direction vs angle**
   ```css
   /* Not the same! */
   background: linear-gradient(to right, red, blue);
   background: linear-gradient(90deg, red, blue);
   /* to right = 90deg, but to top = 0deg (not 270deg) */
   ```

2. **Color stops need units**
   ```css
   /* Invalid */
   background: linear-gradient(red 50, blue 100);
   
   /* Valid */
   background: linear-gradient(red 50%, blue 100%);
   ```

3. **Repeating gradients need defined size**
   ```css
   /* Doesn't repeat - no stop distances */
   background: repeating-linear-gradient(red, blue);
   
   /* Repeats correctly */
   background: repeating-linear-gradient(
     red 0px,
     blue 10px
   );
   ```

4. **Transparent in gradients**
   ```css
   /* Fades through gray in some browsers */
   background: linear-gradient(red, transparent);
   
   /* Better: use rgba */
   background: linear-gradient(
     rgba(255, 0, 0, 1),
     rgba(255, 0, 0, 0)
   );
   ```

5. **Multiple backgrounds layer order**
   ```css
   /* First listed is on top */
   background:
     linear-gradient(red, blue), /* Top */
     url('image.jpg'); /* Bottom */
   ```

## Interview Questions

**Q: Explain the difference between linear-gradient direction keywords and angle values.**

A: Direction keywords (`to right`, `to bottom`) are relative to the element's edges. Angles are absolute:
- `0deg` = bottom to top (up)
- `90deg` = left to right
- `180deg` = top to bottom (down)
- `270deg` = right to left

Note: `to right` equals `90deg`, but `to top` equals `0deg` (not 270deg). Keywords are often clearer for simple directions.

**Q: How do you create a hard color stop vs a smooth transition?**

A: Place two color stops at the same position for hard stops:
```css
/* Hard stop */
background: linear-gradient(
  red 50%,
  blue 50%
);

/* Smooth transition */
background: linear-gradient(
  red 0%,
  blue 100%
);
```

Hard stops create sharp lines, smooth transitions blend between positions.

**Q: What's the difference between radial-gradient size keywords?**

A: 
- `closest-side`: Ends at nearest edge
- `closest-corner`: Ends at nearest corner
- `farthest-side`: Ends at farthest edge
- `farthest-corner`: Ends at farthest corner (default)

Affects gradient spread. `closest-side` creates compact gradients, `farthest-corner` extends to fill entire element.

**Q: How do conic gradients work and what are common use cases?**

A: Conic gradients rotate around a center point. Syntax: `conic-gradient(from angle at position, colors)`.

Use cases:
- Pie charts (hard color stops at specific angles)
- Color wheels (smooth hue transitions)
- Loading spinners
- Starburst/sunburst patterns

Example: `conic-gradient(red 0deg, red 120deg, blue 120deg, blue 240deg, yellow 240deg)`

**Q: How do you create gradient text in CSS?**

A: Use background gradient with background-clip:
```css
.gradient-text {
  background: linear-gradient(45deg, red, blue);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
```

The gradient fills the background, then `background-clip: text` restricts it to text shape, and `text-fill-color: transparent` makes text transparent to show gradient.

**Q: What's the performance impact of gradients?**

A: Gradients are more expensive than solid colors:
- Simple 2-color gradients: Minimal impact
- Complex multi-color gradients: Moderate impact
- Animating gradient properties: Very expensive
- Multiple layered gradients: Can be expensive

Best practices: Keep gradients simple, animate `background-position` instead of gradient itself, use CSS custom properties for dynamic changes.

**Q: How do you create animated gradient backgrounds?**

A: Don't animate the gradient itself. Instead, animate `background-position`:
```css
.animated {
  background: linear-gradient(
    90deg,
    red, blue, red
  );
  background-size: 200% 100%;
  animation: slide 3s infinite;
}

@keyframes slide {
  to { background-position: -200% 0; }
}
```

This is more performant than animating gradient colors.

**Q: Why might transparent in gradients cause unexpected gray colors?**

A: `transparent` is treated as `rgba(0, 0, 0, 0)` (transparent black). Transitioning from a color to transparent can interpolate through black/gray.

Solution: Use `rgba()` with the same color but 0 opacity:
```css
/* Bad */
background: linear-gradient(red, transparent);

/* Good */
background: linear-gradient(
  rgba(255, 0, 0, 1),
  rgba(255, 0, 0, 0)
);
```

**Q: How do you layer multiple gradients?**

A: List gradients separated by commas, first listed appears on top:
```css
background:
  radial-gradient(...), /* Top layer */
  linear-gradient(...), /* Middle */
  url('image.jpg'); /* Bottom */
```

Each layer needs transparency for lower layers to show through. Commonly used for gradient overlays on images or combining radial spotlights with linear backgrounds.