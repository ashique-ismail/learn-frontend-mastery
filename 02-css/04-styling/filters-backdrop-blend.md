# CSS Filters, Backdrop Filters, and Blend Modes

## Filter Functions

Apply graphical effects to elements.

### blur()

```css
/* Blur radius in pixels */
.element {
  filter: blur(5px);
}

/* No blur */
.element {
  filter: blur(0);
}

/* Strong blur */
.element {
  filter: blur(20px);
}

/* Blur on hover */
.image {
  filter: blur(0);
  transition: filter 0.3s;
}

.image:hover {
  filter: blur(5px);
}

/* Focus effect - blur background */
.modal-backdrop {
  filter: blur(10px);
}

/* Loading state blur */
.loading {
  filter: blur(3px);
  pointer-events: none;
}

/* Performance note: blur is expensive, use sparingly */
```

### brightness()

```css
/* Values: 0 (black) to 1 (original) to 2+ (brighter) */
.element {
  filter: brightness(1); /* Original */
  filter: brightness(0); /* Black */
  filter: brightness(0.5); /* 50% darker */
  filter: brightness(1.5); /* 50% brighter */
  filter: brightness(2); /* Double brightness */
}

/* Darken on hover */
.image:hover {
  filter: brightness(0.7);
}

/* Lighten on hover */
.button:hover {
  filter: brightness(1.2);
}

/* Night mode effect */
.night-mode {
  filter: brightness(0.8);
}

/* Highlight active item */
.item {
  filter: brightness(0.8);
  transition: filter 0.2s;
}

.item.active {
  filter: brightness(1);
}
```

### contrast()

```css
/* Values: 0 (gray) to 1 (original) to 2+ (higher contrast) */
.element {
  filter: contrast(1); /* Original */
  filter: contrast(0); /* Gray */
  filter: contrast(0.5); /* Low contrast */
  filter: contrast(1.5); /* High contrast */
  filter: contrast(2); /* Double contrast */
}

/* Make colors pop */
.vibrant {
  filter: contrast(1.3);
}

/* Accessibility: high contrast mode */
@media (prefers-contrast: high) {
  .element {
    filter: contrast(1.5);
  }
}

/* Washed out effect */
.faded {
  filter: contrast(0.7);
}
```

### grayscale()

```css
/* Values: 0 (original) to 1 (fully grayscale) */
.element {
  filter: grayscale(0); /* No grayscale */
  filter: grayscale(0.5); /* 50% grayscale */
  filter: grayscale(1); /* Fully grayscale */
  filter: grayscale(100%); /* Same as 1 */
}

/* Grayscale to color on hover */
.image {
  filter: grayscale(1);
  transition: filter 0.3s;
}

.image:hover {
  filter: grayscale(0);
}

/* Disabled state */
.button:disabled {
  filter: grayscale(1);
  opacity: 0.5;
}

/* Black and white effect */
.bw-photo {
  filter: grayscale(1) contrast(1.2);
}
```

### hue-rotate()

```css
/* Rotate colors around color wheel */
.element {
  filter: hue-rotate(0deg); /* Original */
  filter: hue-rotate(90deg); /* Shift colors */
  filter: hue-rotate(180deg); /* Opposite colors */
  filter: hue-rotate(360deg); /* Back to original */
}

/* Negative values */
.element {
  filter: hue-rotate(-90deg);
}

/* Color theme switcher */
.theme-red { filter: hue-rotate(0deg); }
.theme-blue { filter: hue-rotate(60deg); }
.theme-green { filter: hue-rotate(120deg); }

/* Animated rainbow effect */
@keyframes rainbow {
  from { filter: hue-rotate(0deg); }
  to { filter: hue-rotate(360deg); }
}

.rainbow {
  animation: rainbow 3s linear infinite;
}

/* Tint effect */
.sepia-tint {
  filter: sepia(1) hue-rotate(30deg);
}
```

### invert()

```css
/* Values: 0 (original) to 1 (inverted) */
.element {
  filter: invert(0); /* Original */
  filter: invert(0.5); /* 50% inverted */
  filter: invert(1); /* Fully inverted */
}

/* Dark mode inversion */
.dark-mode img {
  filter: invert(1);
}

/* Invert icons for contrast */
.icon-dark-bg {
  filter: invert(1);
}

/* Negative effect */
.negative {
  filter: invert(1);
}

/* Smart inversion (avoid inverting images) */
@media (prefers-color-scheme: dark) {
  body {
    filter: invert(1) hue-rotate(180deg);
  }
  
  img, video {
    filter: invert(1) hue-rotate(180deg); /* Un-invert */
  }
}
```

### saturate()

```css
/* Values: 0 (grayscale) to 1 (original) to 2+ (oversaturated) */
.element {
  filter: saturate(1); /* Original */
  filter: saturate(0); /* Grayscale */
  filter: saturate(0.5); /* Desaturated */
  filter: saturate(1.5); /* More vibrant */
  filter: saturate(2); /* Highly saturated */
}

/* Vibrant colors */
.vibrant {
  filter: saturate(1.5);
}

/* Desaturated look */
.muted {
  filter: saturate(0.5);
}

/* Instagram-like filters */
.filter-vivid {
  filter: saturate(1.5) contrast(1.2);
}

.filter-faded {
  filter: saturate(0.7) brightness(1.1);
}

/* Hover effect */
.image {
  filter: saturate(0.7);
  transition: filter 0.3s;
}

.image:hover {
  filter: saturate(1.2);
}
```

### sepia()

```css
/* Values: 0 (original) to 1 (fully sepia) */
.element {
  filter: sepia(0); /* Original */
  filter: sepia(0.5); /* 50% sepia */
  filter: sepia(1); /* Fully sepia */
}

/* Vintage photo effect */
.vintage {
  filter: sepia(0.8);
}

/* Old photograph */
.old-photo {
  filter: sepia(1) contrast(0.9) brightness(1.1);
}

/* Warm tone */
.warm {
  filter: sepia(0.3);
}
```

### drop-shadow()

```css
/* drop-shadow(offset-x offset-y blur-radius color) */
.element {
  filter: drop-shadow(2px 2px 4px rgba(0, 0, 0, 0.3));
}

/* Multiple shadows */
.element {
  filter: 
    drop-shadow(2px 2px 4px rgba(0, 0, 0, 0.3))
    drop-shadow(-2px -2px 4px rgba(255, 255, 255, 0.5));
}

/* Advantage over box-shadow: follows element outline */
.icon {
  /* Follows PNG transparency */
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.2));
}

/* vs box-shadow which is rectangular */
.icon {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2); /* Rectangle */
}

/* Glow effect */
.glow {
  filter: drop-shadow(0 0 10px #00f);
}

/* Lifted shadow */
.lifted {
  filter: drop-shadow(0 10px 20px rgba(0, 0, 0, 0.2));
}

/* Text shadow effect on clipped text */
.clipped-text {
  background: linear-gradient(to right, red, blue);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  filter: drop-shadow(2px 2px 4px rgba(0, 0, 0, 0.5));
}
```

## Combining Filters

```css
/* Multiple filters applied in order */
.element {
  filter: grayscale(0.5) brightness(1.2) contrast(1.3);
}

/* Instagram-like filters */
.filter-nashville {
  filter: sepia(0.2) contrast(1.2) brightness(1.05) saturate(1.2);
}

.filter-hudson {
  filter: brightness(1.2) contrast(0.9) saturate(1.1);
}

.filter-lomo {
  filter: contrast(1.5) brightness(0.9) saturate(1.5);
}

.filter-toaster {
  filter: contrast(1.5) brightness(0.9) sepia(0.2);
}

.filter-walden {
  filter: brightness(1.1) saturate(1.6) sepia(0.3) hue-rotate(-20deg);
}

/* Hover effect combining filters */
.image {
  filter: grayscale(1) brightness(0.8);
  transition: filter 0.3s;
}

.image:hover {
  filter: grayscale(0) brightness(1) contrast(1.2);
}

/* Animated filter */
@keyframes filterAnimation {
  0% {
    filter: hue-rotate(0deg) saturate(1);
  }
  50% {
    filter: hue-rotate(180deg) saturate(1.5);
  }
  100% {
    filter: hue-rotate(360deg) saturate(1);
  }
}

.animated {
  animation: filterAnimation 5s infinite;
}
```

## backdrop-filter

Apply filters to area behind element (requires semi-transparent background).

```css
/* Basic backdrop blur (frosted glass effect) */
.glass {
  background: rgba(255, 255, 255, 0.3);
  backdrop-filter: blur(10px);
}

/* All filter functions work */
.element {
  backdrop-filter: blur(5px);
  backdrop-filter: brightness(1.5);
  backdrop-filter: contrast(0.8);
  backdrop-filter: grayscale(0.5);
  backdrop-filter: hue-rotate(90deg);
  backdrop-filter: invert(0.3);
  backdrop-filter: saturate(1.8);
  backdrop-filter: sepia(0.5);
}

/* Combining backdrop filters */
.frosted-glass {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px) saturate(1.5) brightness(1.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

/* macOS Big Sur style blur */
.macos-blur {
  background: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(20px) saturate(1.8);
  -webkit-backdrop-filter: blur(20px) saturate(1.8);
}

/* Dark glass */
.dark-glass {
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
}

/* Modal backdrop */
.modal-backdrop {
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(5px);
}

/* Navigation bar with blur */
.nav {
  position: fixed;
  top: 0;
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(10px) saturate(1.5);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

/* Card with backdrop blur */
.card {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(15px);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

/* Browser support check */
@supports (backdrop-filter: blur(10px)) {
  .modern-blur {
    backdrop-filter: blur(10px);
    background: rgba(255, 255, 255, 0.7);
  }
}

@supports not (backdrop-filter: blur(10px)) {
  .fallback-blur {
    background: rgba(255, 255, 255, 0.95); /* More opaque fallback */
  }
}
```

## mix-blend-mode

Control how element blends with background.

```css
/* Blend modes */
.element {
  mix-blend-mode: normal; /* Default */
  mix-blend-mode: multiply;
  mix-blend-mode: screen;
  mix-blend-mode: overlay;
  mix-blend-mode: darken;
  mix-blend-mode: lighten;
  mix-blend-mode: color-dodge;
  mix-blend-mode: color-burn;
  mix-blend-mode: hard-light;
  mix-blend-mode: soft-light;
  mix-blend-mode: difference;
  mix-blend-mode: exclusion;
  mix-blend-mode: hue;
  mix-blend-mode: saturation;
  mix-blend-mode: color;
  mix-blend-mode: luminosity;
}

/* Text over image */
.hero-text {
  color: white;
  mix-blend-mode: difference; /* Always contrasts with background */
}

/* Duotone effect */
.duotone {
  position: relative;
}

.duotone::before,
.duotone::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.duotone::before {
  background: #00f;
  mix-blend-mode: darken;
}

.duotone::after {
  background: #f0f;
  mix-blend-mode: lighten;
}

/* Colorize image */
.colorize {
  position: relative;
}

.colorize::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #00f;
  mix-blend-mode: color;
}

/* Multiply blend for shadows */
.shadow-blend {
  mix-blend-mode: multiply;
}

/* Screen blend for highlights */
.highlight-blend {
  mix-blend-mode: screen;
}

/* Text knockout effect */
.knockout-text {
  background: url('image.jpg');
  color: white;
  mix-blend-mode: multiply;
}

/* Creative hover effect */
.blend-image {
  transition: mix-blend-mode 0.3s;
}

.blend-image:hover {
  mix-blend-mode: difference;
}
```

## isolation

Create new stacking context for blend modes.

```css
/* Isolate blend mode effects */
.container {
  isolation: isolate; /* Creates stacking context */
}

.container .blended-child {
  mix-blend-mode: multiply;
  /* Only blends with siblings in this container, not background */
}

/* Without isolation */
.no-isolation .blended {
  mix-blend-mode: multiply;
  /* Blends with everything behind, including body background */
}

/* With isolation */
.isolated {
  isolation: isolate;
}

.isolated .blended {
  mix-blend-mode: multiply;
  /* Only blends within .isolated context */
}

/* Practical example */
.card {
  isolation: isolate; /* Prevent blend modes from affecting page background */
  background: white;
}

.card-overlay {
  mix-blend-mode: overlay;
  /* Only affects elements inside .card */
}
```

## Complete Blend Modes Reference

```css
/* Darken modes */
.darken { mix-blend-mode: darken; }
.multiply { mix-blend-mode: multiply; } /* Darkens, like overlaying paint */
.color-burn { mix-blend-mode: color-burn; } /* Intense darkening */

/* Lighten modes */
.lighten { mix-blend-mode: lighten; }
.screen { mix-blend-mode: screen; } /* Lightens, opposite of multiply */
.color-dodge { mix-blend-mode: color-dodge; } /* Intense lightening */

/* Contrast modes */
.overlay { mix-blend-mode: overlay; } /* Multiply dark, screen light */
.soft-light { mix-blend-mode: soft-light; } /* Subtle overlay */
.hard-light { mix-blend-mode: hard-light; } /* Intense overlay */

/* Difference modes */
.difference { mix-blend-mode: difference; } /* Inverts based on brightness */
.exclusion { mix-blend-mode: exclusion; } /* Like difference but softer */

/* Component modes */
.hue { mix-blend-mode: hue; } /* Use hue, keep saturation/luminosity */
.saturation { mix-blend-mode: saturation; } /* Use saturation only */
.color { mix-blend-mode: color; } /* Use hue + saturation, keep luminosity */
.luminosity { mix-blend-mode: luminosity; } /* Use luminosity, keep hue/saturation */
```

## Performance Considerations

```css
/* 1. Filters are expensive - use sparingly */
/* Bad: Many filtered elements */
.many-blurs div {
  filter: blur(10px); /* Can cause performance issues */
}

/* Good: Filter parent or use backdrop-filter */
.parent-blur {
  filter: blur(10px);
}

/* 2. backdrop-filter is very expensive */
.expensive {
  backdrop-filter: blur(20px); /* Use only when necessary */
}

/* 3. Animate with will-change */
.animating-filter {
  will-change: filter;
  transition: filter 0.3s;
}

.animating-filter:hover {
  filter: brightness(1.2);
}

/* Remove will-change after */
.animating-filter:not(:hover) {
  will-change: auto;
}

/* 4. Prefer filter over SVG filters for simple effects */
/* Faster */
.element {
  filter: blur(5px);
}

/* Slower */
.element {
  filter: url(#svg-blur);
}

/* 5. Reduce filter complexity on mobile */
@media (max-width: 768px) {
  .complex-filter {
    filter: blur(5px); /* Simpler than blur(20px) brightness(1.2) contrast(1.3) */
  }
}

/* 6. Test with many elements */
.list-item {
  /* Be cautious with filters on many items */
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.1));
}
```

## Common Patterns

```css
/* Loading skeleton with shimmer */
@keyframes shimmer {
  0% {
    filter: brightness(1);
  }
  50% {
    filter: brightness(1.3);
  }
  100% {
    filter: brightness(1);
  }
}

.skeleton {
  background: #e0e0e0;
  animation: shimmer 2s infinite;
}

/* Disabled state */
.disabled {
  filter: grayscale(1) opacity(0.5);
  pointer-events: none;
}

/* Focus highlight */
.focusable:focus {
  filter: brightness(1.2) drop-shadow(0 0 0 3px rgba(0, 123, 255, 0.5));
}

/* Image hover zoom with brightness */
.image-zoom {
  transition: transform 0.3s, filter 0.3s;
}

.image-zoom:hover {
  transform: scale(1.1);
  filter: brightness(1.1);
}

/* Dark mode with inversion */
@media (prefers-color-scheme: dark) {
  body {
    filter: invert(1) hue-rotate(180deg);
  }
  
  img,
  video,
  [data-no-invert] {
    filter: invert(1) hue-rotate(180deg); /* Cancel inversion */
  }
}

/* Glassmorphism card */
.glass-card {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px) saturate(1.5);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

/* Text shadow with drop-shadow */
.text-icon {
  background: linear-gradient(45deg, #f00, #00f);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  filter: drop-shadow(2px 2px 4px rgba(0, 0, 0, 0.3));
}
```

## Gotchas

1. **backdrop-filter requires transparency**
   ```css
   /* Doesn't work - background is opaque */
   .no-blur {
     background: rgb(255, 255, 255);
     backdrop-filter: blur(10px);
   }
   
   /* Works - background is semi-transparent */
   .blur {
     background: rgba(255, 255, 255, 0.7);
     backdrop-filter: blur(10px);
   }
   ```

2. **Filters create stacking context**
   ```css
   /* Creates new stacking context */
   .filtered {
     filter: blur(5px);
     /* z-index now isolated */
   }
   ```

3. **Filter inheritance**
   ```css
   /* Filters don't inherit but affect children visually */
   .parent {
     filter: blur(5px); /* Blurs entire parent including children */
   }
   
   .child {
     /* Cannot "un-blur" */
   }
   ```

4. **mix-blend-mode affects all layers below**
   ```css
   /* Blends with everything behind */
   .blended {
     mix-blend-mode: multiply;
   }
   
   /* Limit with isolation */
   .container {
     isolation: isolate;
   }
   
   .blended-limited {
     mix-blend-mode: multiply; /* Only blends within container */
   }
   ```

5. **backdrop-filter browser support**
   ```css
   /* Not supported in Firefox (as of 2024) */
   @supports (backdrop-filter: blur(10px)) {
     .blur {
       backdrop-filter: blur(10px);
       background: rgba(255, 255, 255, 0.7);
     }
   }
   
   @supports not (backdrop-filter: blur(10px)) {
     .blur {
       background: rgba(255, 255, 255, 0.95); /* Fallback */
     }
   }
   ```

## Interview Questions

**Q: What's the difference between filter and backdrop-filter?**

A: 
- `filter`: Applies effects to the element and its contents
- `backdrop-filter`: Applies effects to the area behind the element (requires semi-transparent background)

Example: `filter: blur(10px)` blurs the element itself, while `backdrop-filter: blur(10px)` blurs what's behind it (frosted glass effect).

**Q: How does mix-blend-mode work and what are common use cases?**

A: `mix-blend-mode` controls how an element blends with layers behind it, like Photoshop blend modes. Common uses:
- `multiply`: Darken (overlapping colors)
- `screen`: Lighten (opposite of multiply)
- `difference`: Always contrasts (text that works on any background)
- `overlay`: Increase contrast
- `color`: Colorize grayscale images

Use `isolation: isolate` to limit blend scope.

**Q: Why is backdrop-filter considered expensive performance-wise?**

A: `backdrop-filter` requires:
1. Rendering everything behind the element
2. Applying the filter to that rendered area
3. Compositing the result with the element

This is more expensive than regular filters which only process the element itself. Use sparingly, especially with large blur radii or on many elements. Mobile devices particularly struggle with backdrop-filter.

**Q: How do you create a glassmorphism effect?**

A: Combine semi-transparent background with backdrop-filter:
```css
.glass {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px) saturate(1.5);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}
```

Key elements: low-opacity background, backdrop blur, subtle border, shadow for depth. Requires transparent background to see blur effect.

**Q: What's the advantage of drop-shadow() over box-shadow?**

A: `filter: drop-shadow()` follows the actual shape of the element (including transparent PNG areas), while `box-shadow` always creates a rectangular shadow. 

Example: PNG icon with transparency - drop-shadow follows the icon shape, box-shadow creates a rectangle. Drop-shadow also respects clipped elements and SVGs better.

**Q: How do you implement a dark mode using CSS filters?**

A: Use invert and hue-rotate on body, then counter-invert images:
```css
@media (prefers-color-scheme: dark) {
  body {
    filter: invert(1) hue-rotate(180deg);
  }
  
  img, video {
    filter: invert(1) hue-rotate(180deg); /* Cancel */
  }
}
```

Inverts colors but hue-rotate(180deg) preserves original hues. Quick but imperfect solution - better to use CSS custom properties for colors.

**Q: What does isolation: isolate do?**

A: Creates a new stacking context that isolates `mix-blend-mode` effects. Without isolation, blended elements affect all layers below including body background. With isolation, blending is limited to the isolated container.

Useful when you want blend modes to only affect siblings, not the entire page background.

**Q: Can you animate CSS filters and what's the performance impact?**

A: Yes, filters can be animated with transitions/animations:
```css
.element {
  filter: brightness(1);
  transition: filter 0.3s;
}

.element:hover {
  filter: brightness(1.2);
}
```

Performance: Filter animations are expensive as they require re-rendering. Use `will-change: filter` before animation, remove after. Prefer animating opacity/transform when possible. Keep filter changes simple on mobile.

**Q: How do you handle backdrop-filter browser support?**

A: Use feature queries with fallbacks:
```css
@supports (backdrop-filter: blur(10px)) {
  .element {
    background: rgba(255, 255, 255, 0.7);
    backdrop-filter: blur(10px);
  }
}

@supports not (backdrop-filter: blur(10px)) {
  .element {
    background: rgba(255, 255, 255, 0.95); /* More opaque */
  }
}
```

Or use webkit prefix: `-webkit-backdrop-filter` for Safari. Not supported in Firefox as of 2024.