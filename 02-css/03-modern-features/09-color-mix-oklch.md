# Modern Color Functions: color-mix() and OKLCH

## Overview

Modern CSS introduces powerful color capabilities:
- **`color-mix()`**: Blend colors in any color space
- **OKLCH/OKLAB**: Perceptually uniform color spaces
- **Relative colors**: Modify existing colors
- **Wide gamut**: Display-P3, Rec2020 for modern displays

These replace older color functions (lighten/darken from Sass) with more accurate, perceptually uniform alternatives.

## color-mix() Function

Mix two colors with optional percentages in a specified color space.

### Basic Syntax

```css
color-mix(in <colorspace>, <color1> <percentage>?, <color2> <percentage>?)
```

### Simple Examples

```css
.button {
  /* 50/50 mix in sRGB */
  background: color-mix(in srgb, blue, white);
  /* Result: light blue */
  
  /* 75% blue, 25% white */
  background: color-mix(in srgb, blue 75%, white);
  
  /* Explicit percentages */
  background: color-mix(in srgb, blue 60%, white 40%);
}
```

### Color Spaces

```css
/* Different color spaces produce different results */
.demo {
  /* sRGB (default web color space) */
  --mix-srgb: color-mix(in srgb, blue, yellow);
  /* Result: grayish (passes through gray) */
  
  /* HSL (hue-based interpolation) */
  --mix-hsl: color-mix(in hsl, blue, yellow);
  /* Result: greenish (goes through green on hue wheel) */
  
  /* OKLCH (perceptually uniform) */
  --mix-oklch: color-mix(in oklch, blue, yellow);
  /* Result: more natural looking */
  
  /* LAB (perceptually uniform) */
  --mix-lab: color-mix(in lab, blue, yellow);
}
```

## Color Spaces Comparison

| Color Space | Type | Perceptually Uniform | Use Case |
|-------------|------|---------------------|----------|
| `srgb` | RGB | No | Simple mixing, backward compat |
| `srgb-linear` | RGB | No | Physically accurate light mixing |
| `hsl` | Cylindrical | No | Hue-based interpolation |
| `hwb` | Cylindrical | No | Whiteness/blackness mixing |
| `lab` | Perceptual | Yes | Accurate lightness mixing |
| `oklab` | Perceptual | Yes | Better hue uniformity than LAB |
| `lch` | Perceptual | Yes | LAB in cylindrical form |
| `oklch` | Perceptual | Yes | **Best overall choice** |
| `xyz` | CIE | No | Color science calculations |
| `display-p3` | RGB | No | Wide gamut displays |
| `rec2020` | RGB | No | Ultra-wide gamut (future) |

## Practical color-mix() Use Cases

### Creating Tints and Shades

```css
:root {
  --primary: #0066cc;
  
  /* Tints (mixing with white) */
  --primary-100: color-mix(in oklch, var(--primary) 10%, white);
  --primary-200: color-mix(in oklch, var(--primary) 20%, white);
  --primary-300: color-mix(in oklch, var(--primary) 40%, white);
  --primary-400: color-mix(in oklch, var(--primary) 60%, white);
  --primary-500: var(--primary);
  
  /* Shades (mixing with black) */
  --primary-600: color-mix(in oklch, var(--primary) 80%, black);
  --primary-700: color-mix(in oklch, var(--primary) 60%, black);
  --primary-800: color-mix(in oklch, var(--primary) 40%, black);
  --primary-900: color-mix(in oklch, var(--primary) 20%, black);
}

/* Old way (inaccurate) */
.old {
  --tint: rgba(0, 102, 204, 0.5); /* Just transparency, not mixing */
}
```

### Hover States

```css
.button {
  background: var(--primary);
  
  /* Darken on hover */
  &:hover {
    background: color-mix(in oklch, var(--primary) 90%, black);
  }
  
  /* Lighten on hover (for dark backgrounds) */
  &:hover {
    background: color-mix(in oklch, var(--primary) 90%, white);
  }
}
```

### Transparent Overlays

```css
.modal-backdrop {
  /* Mix color with transparent */
  background: color-mix(in srgb, black 50%, transparent);
  /* Better than rgba(0, 0, 0, 0.5) for consistency */
}

.glass-effect {
  background: color-mix(in oklch, white 20%, transparent);
  backdrop-filter: blur(10px);
}
```

### Theme Variations

```css
:root {
  --base-color: #0066cc;
}

/* Automatically generate complementary colors */
.theme {
  --primary: var(--base-color);
  --primary-hover: color-mix(in oklch, var(--primary) 85%, black);
  --primary-active: color-mix(in oklch, var(--primary) 70%, black);
  --primary-light: color-mix(in oklch, var(--primary) 30%, white);
  --primary-lighter: color-mix(in oklch, var(--primary) 15%, white);
  
  /* Background colors */
  --bg-primary: color-mix(in oklch, var(--primary) 10%, white);
  --bg-hover: color-mix(in oklch, var(--primary) 15%, white);
}
```

## OKLCH Color Space

**OKLCH** = **O**ptimized **L**ightness **C**hroma **H**ue

The best modern color space for web design.

### OKLCH Syntax

```css
oklch(L C H / A)
/* L: Lightness (0-1 or 0%-100%) */
/* C: Chroma/saturation (0-0.4, unbounded) */
/* H: Hue (0-360 degrees) */
/* A: Alpha (0-1, optional) */

.example {
  /* Syntax variations */
  color: oklch(0.6 0.15 250);           /* No alpha */
  color: oklch(60% 0.15 250);           /* Lightness as percentage */
  color: oklch(0.6 0.15 250 / 0.5);     /* With alpha */
  color: oklch(60% 0.15 250 / 50%);     /* Percentages */
}
```

### OKLCH Components

```css
/* Lightness: 0 (black) to 1 (white) */
.lightness-demo {
  --very-dark: oklch(0.2 0.1 250);    /* 20% lightness */
  --medium: oklch(0.5 0.1 250);       /* 50% lightness */
  --very-light: oklch(0.9 0.1 250);   /* 90% lightness */
}

/* Chroma: 0 (gray) to 0.4+ (vivid) */
.chroma-demo {
  --gray: oklch(0.6 0 250);           /* No color */
  --muted: oklch(0.6 0.05 250);       /* Subtle color */
  --vibrant: oklch(0.6 0.15 250);     /* Vivid color */
  --ultra: oklch(0.6 0.3 250);        /* Ultra vivid (may be out of gamut) */
}

/* Hue: 0-360 degrees */
.hue-demo {
  --red: oklch(0.6 0.15 30);          /* ~30° red */
  --yellow: oklch(0.6 0.15 90);       /* ~90° yellow */
  --green: oklch(0.6 0.15 140);       /* ~140° green */
  --cyan: oklch(0.6 0.15 200);        /* ~200° cyan */
  --blue: oklch(0.6 0.15 250);        /* ~250° blue */
  --magenta: oklch(0.6 0.15 330);     /* ~330° magenta */
}
```

## Why OKLCH is Better

### Problem with HSL

```css
/* HSL: not perceptually uniform */
.hsl-problem {
  --yellow: hsl(60, 100%, 50%);   /* Looks very bright */
  --blue: hsl(240, 100%, 50%);    /* Looks much darker */
  /* Same L=50% but drastically different perceived brightness! */
}

/* OKLCH: perceptually uniform */
.oklch-solution {
  --yellow: oklch(0.9 0.2 100);   /* Actually bright */
  --blue: oklch(0.5 0.2 250);     /* Actually dark */
  /* Lightness matches perception */
}
```

### Consistent Color Palettes

```css
/* OKLCH makes it easy to create consistent palettes */
:root {
  /* Same lightness, different hues - all feel equally bright */
  --red: oklch(0.55 0.22 30);
  --orange: oklch(0.55 0.22 60);
  --yellow: oklch(0.55 0.22 90);
  --green: oklch(0.55 0.22 140);
  --cyan: oklch(0.55 0.22 200);
  --blue: oklch(0.55 0.22 250);
  --purple: oklch(0.55 0.22 300);
  --pink: oklch(0.55 0.22 330);
}
```

### Creating Color Scales

```css
/* Gray scale: vary only lightness */
:root {
  --gray-50: oklch(0.95 0.02 250);
  --gray-100: oklch(0.90 0.02 250);
  --gray-200: oklch(0.80 0.02 250);
  --gray-300: oklch(0.70 0.02 250);
  --gray-400: oklch(0.60 0.02 250);
  --gray-500: oklch(0.50 0.02 250);
  --gray-600: oklch(0.40 0.02 250);
  --gray-700: oklch(0.30 0.02 250);
  --gray-800: oklch(0.20 0.02 250);
  --gray-900: oklch(0.15 0.02 250);
}

/* Color scale: vary lightness and chroma together */
:root {
  --blue-50: oklch(0.95 0.02 250);  /* Very light, low chroma */
  --blue-100: oklch(0.90 0.04 250);
  --blue-200: oklch(0.85 0.06 250);
  --blue-300: oklch(0.75 0.08 250);
  --blue-400: oklch(0.65 0.12 250);
  --blue-500: oklch(0.55 0.15 250); /* Base color */
  --blue-600: oklch(0.45 0.15 250);
  --blue-700: oklch(0.35 0.13 250);
  --blue-800: oklch(0.25 0.10 250);
  --blue-900: oklch(0.20 0.08 250); /* Very dark, lower chroma */
}
```

## OKLAB Color Space

OKLAB is the rectangular (non-cylindrical) version of OKLCH.

```css
oklab(L a b / A)
/* L: Lightness (0-1) */
/* a: green-red axis (-0.4 to 0.4) */
/* b: blue-yellow axis (-0.4 to 0.4) */

.oklab-example {
  color: oklab(0.6 -0.1 0.1);        /* Greenish-yellow */
  color: oklab(60% -0.1 0.1 / 50%);  /* With alpha */
}
```

**When to use OKLAB vs OKLCH:**
- **OKLCH**: Color manipulation, palettes, UI design (easier to work with)
- **OKLAB**: Color calculations, interpolation, gradients (mathematically simpler)

## Relative Colors

Modify existing colors by adjusting individual channels.

### Basic Syntax

```css
/* Relative color syntax */
color: oklch(from <basecolor> <L> <C> <H> / <A>)
```

### Adjusting Lightness

```css
:root {
  --primary: oklch(0.55 0.2 250);
}

.variations {
  /* Lighten by increasing L */
  --lighter: oklch(from var(--primary) calc(l + 0.2) c h);
  
  /* Darken by decreasing L */
  --darker: oklch(from var(--primary) calc(l - 0.2) c h);
  
  /* Set specific lightness */
  --light: oklch(from var(--primary) 0.8 c h);
  --dark: oklch(from var(--primary) 0.3 c h);
}
```

### Adjusting Chroma (Saturation)

```css
.saturation {
  /* More saturated */
  --vibrant: oklch(from var(--primary) l calc(c + 0.1) h);
  
  /* Less saturated (more muted) */
  --muted: oklch(from var(--primary) l calc(c - 0.1) h);
  
  /* Fully desaturated (grayscale) */
  --gray: oklch(from var(--primary) l 0 h);
}
```

### Adjusting Hue

```css
.hue-shift {
  /* Rotate hue */
  --complement: oklch(from var(--primary) l c calc(h + 180));
  --analogous-1: oklch(from var(--primary) l c calc(h + 30));
  --analogous-2: oklch(from var(--primary) l c calc(h - 30));
  --triadic-1: oklch(from var(--primary) l c calc(h + 120));
  --triadic-2: oklch(from var(--primary) l c calc(h + 240));
}
```

### Adjusting Alpha

```css
.transparency {
  /* Set alpha */
  --semi: oklch(from var(--primary) l c h / 0.5);
  
  /* Adjust alpha */
  --more-opaque: oklch(from var(--primary) l c h / calc(alpha + 0.2));
  --more-transparent: oklch(from var(--primary) l c h / calc(alpha - 0.2));
}
```

### Converting Between Spaces

```css
/* Convert RGB to OKLCH */
.convert {
  --rgb-color: #0066cc;
  --as-oklch: oklch(from var(--rgb-color) l c h);
  
  /* Now can manipulate in OKLCH */
  --lighter: oklch(from var(--rgb-color) calc(l + 0.2) c h);
}
```

## Wide Gamut Colors

Modern displays support colors beyond sRGB.

### Display-P3

```css
/* Display-P3: ~25% more colors than sRGB */
.wide-gamut {
  /* P3 red (more vivid than sRGB red) */
  color: color(display-p3 1 0 0);
  
  /* P3 cyan (more vibrant than sRGB cyan) */
  background: color(display-p3 0 1 1);
  
  /* With alpha */
  border-color: color(display-p3 0.5 0.2 0.9 / 0.8);
}

/* Feature detection */
@media (color-gamut: p3) {
  .vibrant {
    /* Use P3 colors for capable displays */
    color: color(display-p3 1 0.5 0);
  }
}

@media (color-gamut: srgb) {
  .vibrant {
    /* Fallback to sRGB */
    color: rgb(255, 128, 0);
  }
}
```

### Rec2020 (Ultra-Wide Gamut)

```css
/* Rec2020: even wider than P3 (rare support) */
.future {
  color: color(rec2020 1 0 0);
}

@media (color-gamut: rec2020) {
  /* Only for high-end displays */
}
```

### OKLCH with Display-P3

```css
/* OKLCH can represent P3 colors */
.p3-blue {
  /* More vivid blue than sRGB can display */
  color: oklch(0.5 0.3 250); /* High chroma may be P3 */
}

/* Browsers automatically clamp to display capabilities */
```

## Practical Complete Example

```css
:root {
  /* Base brand color in OKLCH */
  --brand-hue: 250;
  --brand-base: oklch(0.55 0.2 var(--brand-hue));
  
  /* Automatic palette generation */
  --brand-50: oklch(0.97 0.01 var(--brand-hue));
  --brand-100: oklch(0.93 0.03 var(--brand-hue));
  --brand-200: oklch(0.85 0.06 var(--brand-hue));
  --brand-300: oklch(0.75 0.10 var(--brand-hue));
  --brand-400: oklch(0.65 0.15 var(--brand-hue));
  --brand-500: var(--brand-base);
  --brand-600: oklch(0.45 0.18 var(--brand-hue));
  --brand-700: oklch(0.35 0.15 var(--brand-hue));
  --brand-800: oklch(0.28 0.12 var(--brand-hue));
  --brand-900: oklch(0.22 0.09 var(--brand-hue));
  
  /* Derived colors with color-mix */
  --brand-hover: color-mix(in oklch, var(--brand-base) 85%, black);
  --brand-active: color-mix(in oklch, var(--brand-base) 70%, black);
  --brand-light-bg: color-mix(in oklch, var(--brand-base) 10%, white);
  
  /* Relative colors for states */
  --brand-focus: oklch(from var(--brand-base) l c h / 0.3);
  --brand-disabled: oklch(from var(--brand-base) l 0.02 h);
  
  /* Complementary colors */
  --accent: oklch(from var(--brand-base) l c calc(h + 180));
  --accent-hover: color-mix(in oklch, var(--accent) 85%, black);
}

/* Usage in components */
.button {
  background: var(--brand-500);
  color: white;
  
  &:hover {
    background: var(--brand-hover);
  }
  
  &:active {
    background: var(--brand-active);
  }
  
  &:focus-visible {
    outline: 2px solid var(--brand-base);
    outline-offset: 2px;
  }
  
  &[disabled] {
    background: var(--brand-disabled);
    cursor: not-allowed;
  }
}

.badge {
  background: var(--brand-light-bg);
  color: var(--brand-800);
  border: 1px solid color-mix(in oklch, var(--brand-base) 30%, white);
}
```

## Browser Support

```css
/* Feature detection for color-mix */
@supports (background: color-mix(in oklch, red, blue)) {
  .modern {
    background: color-mix(in oklch, var(--primary), white);
  }
}

/* Feature detection for OKLCH */
@supports (color: oklch(0.5 0.2 250)) {
  .oklch-colors {
    color: oklch(0.6 0.15 250);
  }
}

/* Fallback */
@supports not (color: oklch(0.5 0.2 250)) {
  .oklch-colors {
    color: #4169e1; /* RGB fallback */
  }
}
```

## Common Gotchas

### 1. Out of Gamut Colors

```css
/* OKLCH can describe colors outside sRGB/display capabilities */
.danger {
  /* Very high chroma may be out of gamut */
  color: oklch(0.6 0.5 30);
  /* Browser will clamp to displayable color */
}

/* Check with DevTools or color picker */
```

### 2. Chroma Varies by Lightness

```css
/* Maximum displayable chroma varies by lightness */
.blue {
  /* At L=0.5, max chroma ~0.3 */
  color: oklch(0.5 0.3 250);
  
  /* At L=0.9, max chroma ~0.05 */
  color: oklch(0.9 0.05 250);
  /* Higher chroma would be clamped */
}
```

### 3. Percentage Mixing

```css
/* Percentages must add to 100% (or omit one) */
color-mix(in oklch, blue 60%, white 40%)  /* OK */
color-mix(in oklch, blue 60%, white)      /* OK - white is 40% */
color-mix(in oklch, blue 60%, white 50%)  /* Invalid - total is 110% */
```

### 4. Color Space Matters

```css
/* Different spaces = different results */
.a { color: color-mix(in srgb, blue, yellow); }    /* Gray/brown */
.b { color: color-mix(in hsl, blue, yellow); }     /* Green */
.c { color: color-mix(in oklch, blue, yellow); }   /* Natural mix */

/* Use OKLCH for most UI work */
```

## Performance

Modern color functions have **no runtime performance penalty**. They're computed at style computation time, not per-frame.

## Interview Questions

**Q1: What problem does OKLCH solve compared to HSL?**

**A:** HSL is not perceptually uniform - colors with the same L value (lightness) appear dramatically different in brightness to humans (yellow looks much brighter than blue at 50% L). OKLCH is perceptually uniform, meaning colors with the same L value appear equally bright. This makes it much easier to create consistent color palettes and accessible color combinations.

**Q2: When would you use `color-mix()` vs relative colors?**

**A:** Use `color-mix()` for blending two colors (tints, shades, transparency, hover states). Use relative colors for modifying a single color's channels (adjusting lightness, rotating hue, changing saturation). Example: `color-mix(in oklch, blue, white)` creates a tint, while `oklch(from blue calc(l + 0.2) c h)` lightens blue specifically.

**Q3: What are the components of OKLCH?**

**A:** 
- **L (Lightness)**: 0-1 or 0%-100%, perceptually uniform brightness
- **C (Chroma)**: 0-0.4+, color intensity (0 is gray, higher is more vivid)
- **H (Hue)**: 0-360 degrees, color angle (red ~30°, blue ~250°)
- **A (Alpha)**: 0-1, opacity (optional)

**Q4: How do you create a consistent color scale with OKLCH?**

**A:** Keep chroma and hue constant, vary only lightness for gray scales. For color scales, vary lightness and reduce chroma slightly at the extremes (very light and very dark). Example:
```css
--blue-100: oklch(0.95 0.02 250);  /* Very light, low chroma */
--blue-500: oklch(0.55 0.15 250);  /* Mid, full chroma */
--blue-900: oklch(0.20 0.08 250);  /* Very dark, reduced chroma */
```

**Q5: What's the difference between OKLCH and OKLAB?**

**A:** Both are perceptually uniform. OKLCH is cylindrical (Lightness, Chroma, Hue) - easier for humans to understand and manipulate. OKLAB is rectangular (Lightness, a-axis, b-axis) - simpler for mathematical operations. Use OKLCH for design work, OKLAB for color calculations if needed.

**Q6: How do you handle browser support for modern color functions?**

**A:** Use `@supports` for feature detection and provide RGB/hex fallbacks:
```css
.element {
  background: #0066cc; /* Fallback */
  
  @supports (background: oklch(0.5 0.2 250)) {
    background: oklch(0.55 0.2 250);
  }
}
```

**Q7: What's Display-P3 and when should you use it?**

**A:** Display-P3 is a wide color gamut ~25% larger than sRGB, supported by modern displays (iPhone, MacBook Pro, etc.). Use P3 colors for more vibrant UI on capable displays, with sRGB fallbacks. Check with `@media (color-gamut: p3)` and use high-chroma OKLCH values or `color(display-p3 ...)` syntax.

## Resources

- [MDN: color-mix()](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/color-mix)
- [MDN: oklch()](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/oklch)
- [OKLCH Color Picker](https://oklch.com/)
- [Lea Verou: OKLCH in CSS](https://lea.verou.me/2020/04/lch-colors-in-css-what-why-and-how/)
- [Evil Martians: OKLCH Guide](https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl)
- [CSS Color Module Level 4 Spec](https://www.w3.org/TR/css-color-4/)
- [Can I Use: color-mix()](https://caniuse.com/mdn-css_types_color_color-mix)
- [Can I Use: OKLCH](https://caniuse.com/mdn-css_types_color_oklch)

## Summary

Modern CSS color functions transform color manipulation. Use `color-mix()` in OKLCH color space for blending colors (tints, shades, hover states). Use OKLCH for defining colors with perceptually uniform lightness, making consistent palettes easy. Leverage relative colors to modify individual channels. Wide gamut support (Display-P3) enables more vibrant colors on modern displays. Together, these features replace preprocessor color functions with more accurate, perceptually uniform, and powerful native CSS capabilities.
