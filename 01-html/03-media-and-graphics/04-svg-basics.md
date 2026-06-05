# SVG basics

## The Idea

**In plain English:** SVG (Scalable Vector Graphics) is a way to draw shapes, icons, and illustrations directly in code using mathematical instructions instead of pixels — so they stay perfectly sharp no matter how big or small you make them.

**Real-world analogy:** Think of an architect's blueprint versus a photograph. A photograph is made of tiny dots (pixels), so blowing it up makes it blurry. A blueprint is a set of instructions ("draw a line from point A to point B, then a circle here") — you can print it at any size and it stays crisp. SVG works exactly like a blueprint for your browser.

- The blueprint's instruction sheet = the SVG file (a list of drawing commands in code)
- The "draw a circle at position X, radius Y" instruction = an SVG element like `<circle cx="100" cy="100" r="50"/>`
- The finished printed blueprint at any size = the browser rendering the SVG perfectly sharp on any screen

---

## Overview

Scalable Vector Graphics (SVG) is an XML-based vector image format for 2D graphics. SVGs are resolution-independent, accessible, and can be styled with CSS and manipulated with JavaScript.

## Basic SVG structure

```html
<svg width="200" height="200" xmlns="http://www.w3.org/2000/svg">
  <!-- SVG content goes here -->
</svg>
```

## Basic shapes

### Rectangle

```html
<svg width="200" height="200">
  <rect x="10" y="10" width="180" height="80" fill="blue"/>
</svg>
```

### Rectangle with stroke and rounded corners

```html
<svg width="200" height="200">
  <rect x="10" y="10" width="180" height="80" 
        fill="lightblue" 
        stroke="blue" 
        stroke-width="3"
        rx="10" 
        ry="10"/>
</svg>
```

### Circle

```html
<svg width="200" height="200">
  <circle cx="100" cy="100" r="80" fill="red"/>
</svg>
```

### Ellipse

```html
<svg width="200" height="200">
  <ellipse cx="100" cy="100" rx="90" ry="60" fill="green"/>
</svg>
```

### Line

```html
<svg width="200" height="200">
  <line x1="10" y1="10" x2="190" y2="190" 
        stroke="black" 
        stroke-width="2"/>
</svg>
```

### Polyline

```html
<svg width="200" height="200">
  <polyline points="10,10 50,80 90,20 130,90 170,30" 
            fill="none" 
            stroke="purple" 
            stroke-width="2"/>
</svg>
```

### Polygon

```html
<svg width="200" height="200">
  <polygon points="100,10 150,90 50,90" 
           fill="orange" 
           stroke="darkorange" 
           stroke-width="2"/>
</svg>
```

## Path element

The most powerful SVG element for creating complex shapes.

### Basic path commands

```html
<svg width="200" height="200">
  <!-- M = moveTo, L = lineTo, Z = closePath -->
  <path d="M 10 10 L 90 90 L 10 90 Z" fill="blue"/>
</svg>
```

### Path command reference

- `M x y` - Move to (absolute)
- `m dx dy` - Move to (relative)
- `L x y` - Line to (absolute)
- `l dx dy` - Line to (relative)
- `H x` - Horizontal line to
- `V y` - Vertical line to
- `C x1 y1 x2 y2 x y` - Cubic Bezier curve
- `S x2 y2 x y` - Smooth cubic Bezier
- `Q x1 y1 x y` - Quadratic Bezier curve
- `T x y` - Smooth quadratic Bezier
- `A rx ry rotation large-arc sweep x y` - Arc
- `Z` - Close path

### Complex path example

```html
<svg width="200" height="200">
  <!-- Heart shape -->
  <path d="M 100,150 
           C 100,150 90,130 75,120 
           C 60,110 50,115 50,130 
           C 50,145 60,155 100,180 
           C 140,155 150,145 150,130 
           C 150,115 140,110 125,120 
           C 110,130 100,150 100,150 Z" 
        fill="red"/>
</svg>
```

### Curved paths

```html
<svg width="200" height="200">
  <!-- Quadratic curve -->
  <path d="M 10 80 Q 95 10 180 80" 
        fill="none" 
        stroke="blue" 
        stroke-width="2"/>
  
  <!-- Cubic curve -->
  <path d="M 10 130 C 40 80, 160 80, 190 130" 
        fill="none" 
        stroke="red" 
        stroke-width="2"/>
</svg>
```

## Fill and stroke

### Fill properties

```html
<svg width="300" height="200">
  <!-- Solid fill -->
  <rect x="10" y="10" width="80" height="80" fill="blue"/>
  
  <!-- No fill -->
  <rect x="110" y="10" width="80" height="80" 
        fill="none" stroke="black" stroke-width="2"/>
  
  <!-- Semi-transparent fill -->
  <rect x="210" y="10" width="80" height="80" 
        fill="red" fill-opacity="0.5"/>
</svg>
```

### Stroke properties

```html
<svg width="400" height="300">
  <!-- Basic stroke -->
  <line x1="10" y1="50" x2="390" y2="50" 
        stroke="black" stroke-width="2"/>
  
  <!-- Dashed stroke -->
  <line x1="10" y1="100" x2="390" y2="100" 
        stroke="blue" stroke-width="3" 
        stroke-dasharray="10,5"/>
  
  <!-- Dotted stroke -->
  <line x1="10" y1="150" x2="390" y2="150" 
        stroke="red" stroke-width="3" 
        stroke-dasharray="2,5"/>
  
  <!-- Line caps -->
  <line x1="50" y1="200" x2="150" y2="200" 
        stroke="green" stroke-width="10" stroke-linecap="butt"/>
  <line x1="170" y1="200" x2="270" y2="200" 
        stroke="green" stroke-width="10" stroke-linecap="round"/>
  <line x1="290" y1="200" x2="390" y2="200" 
        stroke="green" stroke-width="10" stroke-linecap="square"/>
</svg>
```

## ViewBox

ViewBox defines the coordinate system and enables responsive SVGs.

### Basic viewBox

```html
<!-- SVG that scales to container -->
<svg viewBox="0 0 100 100" width="400" height="400">
  <circle cx="50" cy="50" r="40" fill="blue"/>
</svg>
```

### Responsive SVG

```html
<svg viewBox="0 0 200 100" style="width: 100%; height: auto;">
  <rect x="10" y="10" width="180" height="80" fill="lightblue"/>
  <text x="100" y="55" text-anchor="middle" font-size="20">
    Responsive
  </text>
</svg>
```

## Grouping with g

```html
<svg width="200" height="200">
  <g fill="blue" stroke="black" stroke-width="2">
    <circle cx="50" cy="50" r="30"/>
    <circle cx="150" cy="50" r="30"/>
  </g>
  
  <g fill="red" stroke="darkred" stroke-width="2">
    <rect x="20" y="120" width="60" height="60"/>
    <rect x="120" y="120" width="60" height="60"/>
  </g>
</svg>
```

## Text in SVG

### Basic text

```html
<svg width="300" height="200">
  <text x="10" y="40" font-family="Arial" font-size="24" fill="black">
    Hello SVG!
  </text>
</svg>
```

### Text styling

```html
<svg width="400" height="300">
  <text x="200" y="50" text-anchor="middle" 
        font-size="32" font-weight="bold" fill="blue">
    Title
  </text>
  
  <text x="200" y="100" text-anchor="middle" 
        font-size="16" font-style="italic" fill="gray">
    Subtitle
  </text>
  
  <text x="200" y="150" text-anchor="middle" 
        font-family="monospace" font-size="14" 
        letter-spacing="2" fill="black">
    MONOSPACE TEXT
  </text>
</svg>
```

### Text on path

```html
<svg width="400" height="200">
  <defs>
    <path id="curve" d="M 50,150 Q 200,50 350,150"/>
  </defs>
  
  <use href="#curve" fill="none" stroke="lightgray" stroke-width="1"/>
  
  <text font-size="20" fill="blue">
    <textPath href="#curve">
      Text follows the curve!
    </textPath>
  </text>
</svg>
```

## Gradients

### Linear gradient

```html
<svg width="200" height="200">
  <defs>
    <linearGradient id="grad1" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" style="stop-color:rgb(255,255,0);stop-opacity:1" />
      <stop offset="100%" style="stop-color:rgb(255,0,0);stop-opacity:1" />
    </linearGradient>
  </defs>
  
  <rect width="200" height="200" fill="url(#grad1)"/>
</svg>
```

### Radial gradient

```html
<svg width="200" height="200">
  <defs>
    <radialGradient id="grad2">
      <stop offset="0%" style="stop-color:white;stop-opacity:1" />
      <stop offset="100%" style="stop-color:blue;stop-opacity:1" />
    </radialGradient>
  </defs>
  
  <circle cx="100" cy="100" r="90" fill="url(#grad2)"/>
</svg>
```

## Patterns

```html
<svg width="400" height="200">
  <defs>
    <pattern id="dots" x="0" y="0" width="20" height="20" patternUnits="userSpaceOnUse">
      <circle cx="10" cy="10" r="3" fill="white"/>
    </pattern>
  </defs>
  
  <rect width="400" height="200" fill="blue"/>
  <rect width="400" height="200" fill="url(#dots)"/>
</svg>
```

## Filters

### Blur effect

```html
<svg width="200" height="200">
  <defs>
    <filter id="blur">
      <feGaussianBlur in="SourceGraphic" stdDeviation="5"/>
    </filter>
  </defs>
  
  <circle cx="100" cy="100" r="50" fill="red" filter="url(#blur)"/>
</svg>
```

### Drop shadow

```html
<svg width="200" height="200">
  <defs>
    <filter id="shadow" x="-50%" y="-50%" width="200%" height="200%">
      <feDropShadow dx="3" dy="3" stdDeviation="3" flood-color="rgba(0,0,0,0.5)"/>
    </filter>
  </defs>
  
  <rect x="50" y="50" width="100" height="100" 
        fill="blue" filter="url(#shadow)"/>
</svg>
```

## Clipping and masking

### Clip path

```html
<svg width="200" height="200">
  <defs>
    <clipPath id="clip">
      <circle cx="100" cy="100" r="80"/>
    </clipPath>
  </defs>
  
  <rect width="200" height="200" fill="blue" clip-path="url(#clip)"/>
</svg>
```

### Mask

```html
<svg width="200" height="200">
  <defs>
    <mask id="mask">
      <circle cx="100" cy="100" r="80" fill="white"/>
      <circle cx="100" cy="100" r="40" fill="black"/>
    </mask>
  </defs>
  
  <rect width="200" height="200" fill="blue" mask="url(#mask)"/>
</svg>
```

## Reusable elements with defs and use

```html
<svg width="400" height="200">
  <defs>
    <g id="star">
      <polygon points="50,5 61,40 96,40 68,61 79,96 50,75 21,96 32,61 4,40 39,40"/>
    </g>
  </defs>
  
  <use href="#star" x="0" y="0" fill="gold"/>
  <use href="#star" x="100" y="0" fill="silver"/>
  <use href="#star" x="200" y="0" fill="bronze"/>
  <use href="#star" x="300" y="0" fill="blue"/>
</svg>
```

## Transforms

```html
<svg width="400" height="300">
  <!-- Translate -->
  <rect x="0" y="0" width="50" height="50" fill="red" 
        transform="translate(50, 50)"/>
  
  <!-- Rotate -->
  <rect x="150" y="50" width="50" height="50" fill="blue" 
        transform="rotate(45 175 75)"/>
  
  <!-- Scale -->
  <rect x="250" y="50" width="50" height="50" fill="green" 
        transform="scale(1.5)"/>
  
  <!-- Skew -->
  <rect x="50" y="150" width="50" height="50" fill="orange" 
        transform="skewX(20)"/>
</svg>
```

## Symbols

```html
<svg width="400" height="200">
  <symbol id="icon" viewBox="0 0 100 100">
    <circle cx="50" cy="50" r="40" fill="blue"/>
    <text x="50" y="60" text-anchor="middle" 
          font-size="40" fill="white">i</text>
  </symbol>
  
  <use href="#icon" width="50" height="50"/>
  <use href="#icon" x="100" width="100" height="100"/>
  <use href="#icon" x="250" width="150" height="150"/>
</svg>
```

## Accessibility

```html
<svg role="img" aria-labelledby="title desc">
  <title id="title">Chart Title</title>
  <desc id="desc">Detailed description of the chart contents</desc>
  
  <rect x="10" y="10" width="100" height="50" fill="blue">
    <title>Data point 1</title>
  </rect>
</svg>
```

## Inline vs external SVG

### Inline SVG

```html
<svg width="100" height="100">
  <circle cx="50" cy="50" r="40" fill="blue"/>
</svg>
```

### External SVG file

```html
<img src="image.svg" alt="Description">
```

### SVG as background

```css
.element {
  background-image: url('image.svg');
}
```

### SVG with object

```html
<object data="image.svg" type="image/svg+xml">
  Fallback content
</object>
```

## Icon example

```html
<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
        d="M4 6h16M4 12h16M4 18h16"/>
</svg>
```

## Logo example

```html
<svg viewBox="0 0 200 100" width="200" height="100">
  <defs>
    <linearGradient id="logo-grad" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#4F46E5"/>
      <stop offset="100%" style="stop-color:#7C3AED"/>
    </linearGradient>
  </defs>
  
  <circle cx="50" cy="50" r="40" fill="url(#logo-grad)"/>
  <text x="110" y="60" font-family="Arial" font-size="32" 
        font-weight="bold" fill="#1F2937">
    Logo
  </text>
</svg>
```

## Best practices

1. Use viewBox for responsive SVGs
2. Optimize SVG files (remove unnecessary code)
3. Use semantic grouping with `<g>`
4. Define reusable elements in `<defs>`
5. Include accessibility attributes
6. Use currentColor for theme-able icons
7. Minimize path complexity
8. Use appropriate coordinate precision
9. Consider file size for complex graphics
10. Test across browsers

## Key takeaways

- SVG is XML-based vector graphics format
- Resolution-independent and scalable
- Basic shapes: rect, circle, ellipse, line, polyline, polygon
- Path element for complex shapes
- ViewBox enables responsive SVGs
- Can be styled with CSS and animated
- Group elements with `<g>` and reuse with `<use>`
- Support gradients, patterns, filters, clipping, masking
- Always include accessibility attributes
- Choose inline SVG for CSS/JS manipulation, external for caching
