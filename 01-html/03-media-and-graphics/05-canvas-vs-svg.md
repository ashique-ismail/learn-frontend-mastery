# Canvas vs SVG

## The Idea

**In plain English:** Canvas and SVG are two different ways to draw graphics on a webpage. Canvas works like a whiteboard where you paint pixels directly and the browser forgets what you drew, while SVG works like a set of sticky notes where every shape is remembered as its own object you can move or click on later.

**Real-world analogy:** Think of painting a mural vs. arranging magnetic shapes on a fridge. With a mural (Canvas) you apply paint stroke by stroke — once it's on the wall, the wall only knows about colored paint, not individual brush strokes. With magnetic shapes (SVG) each piece stays as its own object you can slide around, remove, or tap individually.
- The painted wall = the Canvas element (a grid of pixels, no memory of shapes)
- Each brush stroke = a JavaScript draw call (executes once, leaves pixels behind)
- Each magnetic shape on the fridge = an SVG element in the DOM (a remembered, clickable object)

---

## Overview

Canvas and SVG are two different approaches to graphics on the web. Canvas is raster-based (bitmap) and immediate-mode, while SVG is vector-based and retained-mode.

## Canvas

### What is Canvas?

- Raster graphics (bitmap, pixel-based)
- Immediate mode rendering
- JavaScript API for drawing
- Resolution-dependent
- No DOM representation of drawn objects
- Better for high-performance graphics

### Basic canvas setup

```html
<canvas id="myCanvas" width="800" height="600">
  Fallback content for browsers that don't support canvas.
</canvas>

<script>
  const canvas = document.getElementById('myCanvas');
  const ctx = canvas.getContext('2d');
  
  // Draw a rectangle
  ctx.fillStyle = 'blue';
  ctx.fillRect(50, 50, 200, 100);
</script>
```

### Canvas characteristics

#### Pros:
- Fast for complex scenes with many objects
- Pixel manipulation
- Good for games and animations
- Better performance for thousands of objects
- Can export to image formats easily

#### Cons:
- Not accessible by default
- Resolution-dependent (pixelated when scaled)
- No built-in event handling on shapes
- Must redraw entire canvas for updates
- Not searchable or indexable

### Canvas use cases

```html
<!-- Game graphics -->
<canvas id="game" width="1920" height="1080"></canvas>

<!-- Data visualization with many points -->
<canvas id="chart" width="1000" height="600"></canvas>

<!-- Image manipulation -->
<canvas id="photo-editor" width="1200" height="800"></canvas>

<!-- Particle effects -->
<canvas id="particles" width="800" height="600"></canvas>
```

### Canvas drawing examples

```javascript
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');

// Rectangle
ctx.fillStyle = 'red';
ctx.fillRect(10, 10, 100, 50);

// Circle
ctx.beginPath();
ctx.arc(200, 100, 50, 0, Math.PI * 2);
ctx.fillStyle = 'blue';
ctx.fill();

// Line
ctx.beginPath();
ctx.moveTo(300, 50);
ctx.lineTo(400, 150);
ctx.strokeStyle = 'green';
ctx.lineWidth = 5;
ctx.stroke();

// Text
ctx.font = '24px Arial';
ctx.fillStyle = 'black';
ctx.fillText('Hello Canvas!', 450, 100);

// Image
const img = new Image();
img.onload = function() {
  ctx.drawImage(img, 0, 200, 200, 150);
};
img.src = 'image.jpg';
```

### Canvas animation

```javascript
const canvas = document.getElementById('animation');
const ctx = canvas.getContext('2d');

let x = 0;

function animate() {
  // Clear canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  
  // Draw moving circle
  ctx.beginPath();
  ctx.arc(x, 100, 30, 0, Math.PI * 2);
  ctx.fillStyle = 'blue';
  ctx.fill();
  
  // Update position
  x += 2;
  if (x > canvas.width) x = 0;
  
  // Continue animation
  requestAnimationFrame(animate);
}

animate();
```

## SVG

### What is SVG?

- Vector graphics (mathematical shapes)
- Retained mode rendering
- XML-based markup
- Resolution-independent
- DOM representation of each shape
- Better for interactive graphics

### Basic SVG

```html
<svg width="800" height="600" xmlns="http://www.w3.org/2000/svg">
  <!-- Rectangle -->
  <rect x="50" y="50" width="200" height="100" fill="blue"/>
  
  <!-- Circle -->
  <circle cx="200" cy="200" r="50" fill="red"/>
  
  <!-- Line -->
  <line x1="300" y1="50" x2="400" y2="150" 
        stroke="green" stroke-width="5"/>
  
  <!-- Text -->
  <text x="450" y="100" font-size="24" fill="black">
    Hello SVG!
  </text>
</svg>
```

### SVG characteristics

#### Pros:
- Resolution-independent (scales without quality loss)
- Accessible (DOM-based)
- CSS styling and animations
- Event handling on individual shapes
- Searchable and indexable
- Smaller file size for simple graphics

#### Cons:
- Slower for complex scenes (many objects)
- DOM overhead
- Not suitable for pixel manipulation
- Performance degrades with many elements
- Larger file size for detailed images

### SVG use cases

```html
<!-- Icons and logos -->
<svg width="100" height="100">
  <path d="M50 10 L90 90 L10 90 Z" fill="blue"/>
</svg>

<!-- Charts and diagrams -->
<svg width="600" height="400">
  <rect x="0" y="300" width="100" height="100" fill="blue"/>
  <rect x="120" y="200" width="100" height="200" fill="red"/>
  <rect x="240" y="150" width="100" height="250" fill="green"/>
</svg>

<!-- Interactive maps -->
<svg width="800" height="600">
  <g id="region-1" class="clickable">
    <path d="..." fill="lightblue"/>
  </g>
</svg>

<!-- Illustrations -->
<svg viewBox="0 0 100 100" width="400" height="400">
  <circle cx="50" cy="50" r="40" fill="yellow"/>
  <circle cx="35" cy="45" r="5" fill="black"/>
  <circle cx="65" cy="45" r="5" fill="black"/>
  <path d="M 30 65 Q 50 75 70 65" stroke="black" 
        stroke-width="2" fill="none"/>
</svg>
```

### SVG with CSS

```html
<style>
  .my-circle {
    fill: blue;
    transition: fill 0.3s;
  }
  
  .my-circle:hover {
    fill: red;
  }
  
  @keyframes pulse {
    0%, 100% { transform: scale(1); }
    50% { transform: scale(1.2); }
  }
  
  .animated {
    animation: pulse 2s infinite;
    transform-origin: center;
  }
</style>

<svg width="400" height="400">
  <circle class="my-circle animated" cx="200" cy="200" r="50"/>
</svg>
```

### SVG with JavaScript

```html
<svg id="interactive" width="400" height="400">
  <circle id="myCircle" cx="200" cy="200" r="50" fill="blue"/>
</svg>

<script>
  const circle = document.getElementById('myCircle');
  
  // Event handling
  circle.addEventListener('click', function() {
    this.setAttribute('fill', 'red');
  });
  
  // Animation
  let size = 50;
  let growing = true;
  
  setInterval(() => {
    if (growing) {
      size += 1;
      if (size >= 100) growing = false;
    } else {
      size -= 1;
      if (size <= 50) growing = true;
    }
    circle.setAttribute('r', size);
  }, 50);
</script>
```

## Side-by-side comparison

| Feature | Canvas | SVG |
|---------|--------|-----|
| Type | Raster (bitmap) | Vector |
| Mode | Immediate | Retained |
| DOM | No | Yes |
| Accessibility | Poor | Good |
| Scalability | Pixelated | Perfect |
| Performance (many objects) | Better | Worse |
| Performance (few objects) | Similar | Similar |
| Event handling | Manual | Built-in |
| CSS styling | No | Yes |
| Animation | JavaScript | CSS or JavaScript |
| File size (complex) | N/A | Large |
| Pixel manipulation | Yes | No |
| Best for | Games, complex graphics | Icons, charts, UI |

## When to use Canvas

1. High-performance games
2. Pixel manipulation (photo editing)
3. Complex scenes with thousands of objects
4. Real-time data visualization
5. Particle systems
6. Video processing
7. Dynamic image generation
8. Performance-critical animations

```html
<!-- Game example -->
<canvas id="game" width="1920" height="1080"></canvas>
<script>
  const ctx = document.getElementById('game').getContext('2d');
  
  function gameLoop() {
    ctx.clearRect(0, 0, 1920, 1080);
    
    // Draw thousands of particles
    for (let i = 0; i < 10000; i++) {
      ctx.fillRect(
        Math.random() * 1920,
        Math.random() * 1080,
        2, 2
      );
    }
    
    requestAnimationFrame(gameLoop);
  }
  
  gameLoop();
</script>
```

## When to use SVG

1. Icons and logos
2. Charts and graphs
3. Interactive diagrams
4. Maps
5. UI elements
6. Illustrations
7. Infographics
8. Responsive graphics

```html
<!-- Interactive chart example -->
<svg width="600" height="400" viewBox="0 0 600 400">
  <style>
    .bar { transition: fill 0.3s; }
    .bar:hover { fill: orange; }
  </style>
  
  <rect class="bar" x="50" y="200" width="80" height="150" fill="blue"/>
  <rect class="bar" x="150" y="150" width="80" height="200" fill="blue"/>
  <rect class="bar" x="250" y="100" width="80" height="250" fill="blue"/>
  <rect class="bar" x="350" y="50" width="80" height="300" fill="blue"/>
  
  <text x="300" y="30" text-anchor="middle" font-size="20">
    Sales Chart
  </text>
</svg>
```

## Combining Canvas and SVG

You can use both together:

```html
<!-- SVG for UI, Canvas for graphics -->
<div style="position: relative;">
  <!-- Canvas for game graphics -->
  <canvas id="game" width="800" height="600" 
          style="position: absolute; top: 0; left: 0;"></canvas>
  
  <!-- SVG for UI overlay -->
  <svg width="800" height="600" 
       style="position: absolute; top: 0; left: 0; pointer-events: none;">
    <text x="400" y="50" text-anchor="middle" font-size="24" fill="white">
      Score: 1000
    </text>
    <circle cx="50" cy="50" r="20" fill="red" style="pointer-events: auto;"/>
  </svg>
</div>
```

## Accessibility considerations

### Canvas accessibility

```html
<canvas id="chart" width="600" height="400" role="img" 
        aria-label="Bar chart showing sales data">
  <!-- Fallback content -->
  <table>
    <caption>Sales Data</caption>
    <tr><th>Quarter</th><th>Sales</th></tr>
    <tr><td>Q1</td><td>$50k</td></tr>
    <tr><td>Q2</td><td>$75k</td></tr>
    <tr><td>Q3</td><td>$100k</td></tr>
    <tr><td>Q4</td><td>$125k</td></tr>
  </table>
</canvas>
```

### SVG accessibility

```html
<svg role="img" aria-labelledby="chart-title chart-desc">
  <title id="chart-title">Sales Chart</title>
  <desc id="chart-desc">
    Bar chart showing quarterly sales increasing from $50k in Q1 to $125k in Q4
  </desc>
  
  <rect x="50" y="200" width="80" height="150" fill="blue">
    <title>Q1: $50k</title>
  </rect>
  <!-- More bars... -->
</svg>
```

## Performance tips

### Canvas
- Clear only necessary areas, not entire canvas
- Use offscreen canvases for complex objects
- Batch drawing operations
- Avoid state changes
- Use `requestAnimationFrame` for animations

### SVG
- Limit number of DOM elements
- Use CSS transforms instead of attribute changes
- Group related elements with `<g>`
- Use `<use>` for repeated elements
- Consider `will-change` for animations

## Key takeaways

- Canvas: raster, immediate-mode, better for complex/many objects
- SVG: vector, retained-mode, better for simple/few objects
- Canvas: games, pixel manipulation, high-performance graphics
- SVG: icons, charts, UI elements, accessible graphics
- Canvas scales poorly, SVG scales perfectly
- SVG is accessible and stylable with CSS
- Canvas requires manual accessibility implementation
- Choose based on use case, not preference
- Can combine both for best of both worlds
