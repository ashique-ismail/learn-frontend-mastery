# Reflow and Repaint

## Table of Contents
- [Introduction](#introduction)
- [What Triggers Reflow](#what-triggers-reflow)
- [What Triggers Repaint](#what-triggers-repaint)
- [Composite-Only Changes](#composite-only-changes)
- [Layout Thrashing](#layout-thrashing)
- [Optimizing Reflow and Repaint](#optimizing-reflow-and-repaint)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Reflow and repaint are two critical browser operations that affect rendering performance. Understanding what triggers these operations and how to minimize them is essential for building fast, smooth web applications.

Key concepts:
- **Reflow (Layout)**: Recalculating element positions and sizes
- **Repaint (Paint)**: Redrawing pixels without layout changes
- **Composite**: Combining layers (cheapest operation)
- **Layout thrashing**: Repeated reflow/repaint in quick succession
- **Optimization**: Batching DOM changes, using requestAnimationFrame

## What Triggers Reflow

### Geometry-Affecting Changes

Operations that trigger layout recalculation:

```javascript
// Properties that trigger REFLOW
const reflowProperties = {
  // Dimensions
  width: '100px',
  height: '100px',
  padding: '10px',
  margin: '10px',
  border: '1px solid black',
  
  // Positioning
  position: 'absolute',
  top: '10px',
  left: '10px',
  right: '10px',
  bottom: '10px',
  
  // Display
  display: 'block',
  float: 'left',
  clear: 'both',
  
  // Flexbox
  'flex-direction': 'column',
  'flex-wrap': 'wrap',
  'justify-content': 'center',
  'align-items': 'center',
  
  // Grid
  'grid-template-columns': '1fr 1fr',
  'grid-gap': '10px',
  
  // Text
  'font-size': '16px',
  'font-family': 'Arial',
  'font-weight': 'bold',
  'line-height': '1.5',
  'text-align': 'center',
  'vertical-align': 'middle',
  'white-space': 'nowrap',
  'word-wrap': 'break-word'
};

// Example: All these trigger reflow
element.style.width = '200px';        // Reflow
element.style.padding = '20px';       // Reflow
element.style.fontSize = '18px';      // Reflow
element.style.display = 'inline';     // Reflow
```

### DOM Modifications

Adding, removing, or moving elements:

```javascript
// DOM operations that trigger reflow
class ReflowTriggers {
  // Adding elements
  addElement() {
    const div = document.createElement('div');
    div.textContent = 'New element';
    document.body.appendChild(div); // REFLOW
  }

  // Removing elements
  removeElement(element) {
    element.remove(); // REFLOW
  }

  // Moving elements
  moveElement(element, newParent) {
    newParent.appendChild(element); // REFLOW
  }

  // Changing content
  changeContent(element) {
    element.textContent = 'New content'; // REFLOW (if it affects size)
    element.innerHTML = '<span>HTML</span>'; // REFLOW
  }

  // Changing classes
  changeClass(element) {
    element.className = 'new-class'; // REFLOW (if styles affect layout)
    element.classList.add('active'); // REFLOW (if styles affect layout)
  }

  // Manipulating styles
  changeStyles(element) {
    element.style.width = '100px'; // REFLOW
    element.style.height = '100px'; // REFLOW
    element.style.margin = '10px'; // REFLOW
  }
}
```

### Reading Layout Properties

Accessing certain properties forces immediate layout:

```javascript
// Properties that FORCE synchronous reflow
const forcedReflowProperties = [
  // Position and dimensions
  'offsetTop',
  'offsetLeft',
  'offsetWidth',
  'offsetHeight',
  'offsetParent',
  
  // Client dimensions
  'clientTop',
  'clientLeft',
  'clientWidth',
  'clientHeight',
  
  // Scroll information
  'scrollTop',
  'scrollLeft',
  'scrollWidth',
  'scrollHeight',
  
  // Computed styles
  'getComputedStyle()',
  
  // Bounding rectangles
  'getBoundingClientRect()',
  'getClientRects()',
  
  // Scroll methods
  'scrollIntoView()',
  'scrollTo()',
  
  // Selection
  'getSelection()',
  
  // Focus
  'focus()'
];

// Example: Forced synchronous layout
function forcedReflow() {
  const element = document.querySelector('.box');
  
  // This forces immediate layout calculation
  const width = element.offsetWidth; // FORCED REFLOW
  
  console.log('Width:', width);
}

// Batching to avoid forced reflow
function optimizedReflow() {
  const elements = document.querySelectorAll('.box');
  
  // Batch all reads first
  const widths = Array.from(elements).map(el => el.offsetWidth);
  // Only ONE reflow for all reads
  
  // Then batch all writes
  elements.forEach((el, i) => {
    el.style.width = widths[i] * 2 + 'px';
  });
  // Only ONE more reflow for all writes
}
```

### Window Resizing

Browser resize triggers reflow:

```javascript
// Window resize handling
class ResizeHandler {
  constructor() {
    this.resizeTimeout = null;
    this.setupListeners();
  }

  setupListeners() {
    // BAD: Reflow on every pixel change
    window.addEventListener('resize', () => {
      this.handleResize(); // Called hundreds of times!
    });

    // GOOD: Debounced resize
    window.addEventListener('resize', () => {
      clearTimeout(this.resizeTimeout);
      this.resizeTimeout = setTimeout(() => {
        this.handleResize(); // Called once after resizing stops
      }, 250);
    });

    // BETTER: ResizeObserver (modern)
    const observer = new ResizeObserver(entries => {
      // Called when size actually changes
      for (const entry of entries) {
        this.handleElementResize(entry);
      }
    });

    observer.observe(document.body);
  }

  handleResize() {
    // Resize logic that triggers reflow
    console.log('Window resized');
  }

  handleElementResize(entry) {
    const width = entry.contentRect.width;
    const height = entry.contentRect.height;
    console.log(`Element resized: ${width}x${height}`);
  }
}
```

## What Triggers Repaint

### Visual-Only Changes

Properties that affect appearance but not layout:

```javascript
// Properties that trigger REPAINT (but not reflow)
const repaintOnlyProperties = {
  // Colors
  color: 'red',
  'background-color': 'blue',
  'border-color': 'green',
  'outline-color': 'yellow',
  
  // Visual effects
  'box-shadow': '0 0 10px rgba(0,0,0,0.5)',
  'border-radius': '5px',
  'border-style': 'dashed',
  
  // Visibility
  visibility: 'hidden', // Takes space but not painted
  
  // Outline
  outline: '2px solid red',
  'outline-width': '2px',
  'outline-style': 'solid',
  
  // Text decoration
  'text-decoration': 'underline',
  
  // Cursor
  cursor: 'pointer'
};

// Example: Repaint-only changes
function repaintExample() {
  const element = document.querySelector('.box');
  
  // These trigger REPAINT only (no reflow)
  element.style.color = 'red';              // Repaint
  element.style.backgroundColor = 'blue';   // Repaint
  element.style.boxShadow = '0 0 10px #000'; // Repaint
  element.style.borderRadius = '5px';       // Repaint
  
  // These are cheaper than reflow but still cost performance
}
```

### Visibility Changes

Hiding and showing elements:

```javascript
// Different ways to hide elements
class VisibilityControl {
  // Option 1: display: none
  hideWithDisplay(element) {
    element.style.display = 'none';
    // Triggers: REFLOW (element removed from layout)
    // Element takes no space
    // Child elements also hidden
  }

  showWithDisplay(element) {
    element.style.display = 'block';
    // Triggers: REFLOW (element added back to layout)
  }

  // Option 2: visibility: hidden
  hideWithVisibility(element) {
    element.style.visibility = 'hidden';
    // Triggers: REPAINT (no reflow)
    // Element still takes space
    // Child elements can be visible if set
  }

  showWithVisibility(element) {
    element.style.visibility = 'visible';
    // Triggers: REPAINT
  }

  // Option 3: opacity: 0
  hideWithOpacity(element) {
    element.style.opacity = '0';
    // Triggers: COMPOSITE (if on own layer)
    // Element still takes space
    // Still responds to events
  }

  showWithOpacity(element) {
    element.style.opacity = '1';
    // Triggers: COMPOSITE (cheapest!)
  }

  // Performance comparison
  comparePerformance() {
    const iterations = 1000;

    // display: none/block
    console.time('display');
    for (let i = 0; i < iterations; i++) {
      element.style.display = i % 2 ? 'none' : 'block';
    }
    console.timeEnd('display'); // Slowest (reflow)

    // visibility: hidden/visible
    console.time('visibility');
    for (let i = 0; i < iterations; i++) {
      element.style.visibility = i % 2 ? 'hidden' : 'visible';
    }
    console.timeEnd('visibility'); // Medium (repaint)

    // opacity: 0/1
    console.time('opacity');
    for (let i = 0; i < iterations; i++) {
      element.style.opacity = i % 2 ? '0' : '1';
    }
    console.timeEnd('opacity'); // Fastest (composite)
  }
}
```

### Background and Border Changes

Visual property updates:

```javascript
// Repaint-triggering visual changes
class VisualUpdates {
  // Background changes
  updateBackground(element) {
    element.style.backgroundColor = '#' + Math.random().toString(16).substr(2, 6);
    // Triggers: REPAINT
  }

  updateBackgroundImage(element) {
    element.style.backgroundImage = 'url(new-image.jpg)';
    // Triggers: REPAINT
  }

  // Border changes (not affecting size)
  updateBorderColor(element) {
    element.style.borderColor = 'red';
    // Triggers: REPAINT
  }

  updateBorderStyle(element) {
    element.style.borderStyle = 'dashed';
    // Triggers: REPAINT
  }

  // Box shadow
  updateBoxShadow(element) {
    element.style.boxShadow = '0 0 20px rgba(0,0,0,0.5)';
    // Triggers: REPAINT
  }

  // Multiple visual updates (batched)
  batchVisualUpdates(element) {
    // Use CSS class for batching
    element.classList.add('highlighted');
    // Single REPAINT for all visual changes in class
  }
}
```

## Composite-Only Changes

### Transform and Opacity

Properties that only affect compositing:

```javascript
// Properties that trigger COMPOSITE only (no reflow/repaint)
const compositeOnlyProperties = {
  // Transform (2D and 3D)
  transform: 'translateX(100px)',
  transform: 'translateY(50px)',
  transform: 'translateZ(0)',
  transform: 'scale(1.5)',
  transform: 'rotate(45deg)',
  transform: 'skew(10deg)',
  
  // Opacity
  opacity: '0.5',
  
  // Filters (on composited layer)
  filter: 'blur(5px)',
  'backdrop-filter': 'blur(10px)'
};

// Example: Composite-only animations (fastest)
class CompositeAnimations {
  // Transform animation (GPU accelerated)
  animatePosition(element) {
    element.style.transform = 'translateX(0)';
    
    // Animate with transform (composite only)
    element.style.transition = 'transform 0.3s';
    element.style.transform = 'translateX(100px)';
    // Silky smooth 60fps animation
  }

  // Opacity animation
  animateOpacity(element) {
    element.style.opacity = '1';
    
    element.style.transition = 'opacity 0.3s';
    element.style.opacity = '0';
    // Smooth composite-only change
  }

  // BAD: Left animation (triggers reflow every frame)
  animateLeftBad(element) {
    element.style.left = '0px';
    
    element.style.transition = 'left 0.3s';
    element.style.left = '100px';
    // Janky animation (reflow per frame)
  }

  // GOOD: Transform animation
  animateLeftGood(element) {
    element.style.transform = 'translateX(0)';
    
    element.style.transition = 'transform 0.3s';
    element.style.transform = 'translateX(100px)';
    // Smooth GPU-accelerated animation
  }

  // will-change hint
  prepareForAnimation(element) {
    // Tell browser to create layer
    element.style.willChange = 'transform, opacity';
    
    // Clean up after animation
    setTimeout(() => {
      element.style.willChange = 'auto';
    }, 1000);
  }
}
```

### Layer Promotion

Creating compositor layers:

```javascript
// Techniques to promote elements to layers
class LayerPromotion {
  // Method 1: 3D transform
  promoteWith3D(element) {
    element.style.transform = 'translateZ(0)';
    // Creates layer, enables GPU acceleration
  }

  // Method 2: will-change
  promoteWithWillChange(element) {
    element.style.willChange = 'transform';
    // Hints browser to create layer
    // Use sparingly - costs memory
  }

  // Method 3: Opacity < 1
  promoteWithOpacity(element) {
    element.style.opacity = '0.99';
    // Creates layer (only if needed for blending)
  }

  // Method 4: CSS filter
  promoteWithFilter(element) {
    element.style.filter = 'blur(0)';
    // Creates layer
  }

  // Checking layer creation
  checkLayers() {
    // Chrome DevTools: Layers panel
    // Shows: layer tree, memory usage, repaint count

    const elements = document.querySelectorAll('.animated');
    
    elements.forEach(el => {
      const styles = getComputedStyle(el);
      const hasLayer = (
        styles.transform !== 'none' ||
        styles.willChange !== 'auto' ||
        parseFloat(styles.opacity) < 1 ||
        styles.filter !== 'none'
      );

      console.log('Has layer:', hasLayer, el);
    });
  }

  // Avoiding over-promotion
  avoidOverPromotion() {
    // DON'T promote everything!
    // Each layer costs memory
    // Too many layers = poor performance

    // GOOD: Promote animated elements
    const animatedElement = document.querySelector('.animated');
    animatedElement.style.willChange = 'transform';

    // BAD: Promote static elements
    const staticElements = document.querySelectorAll('.static');
    staticElements.forEach(el => {
      el.style.transform = 'translateZ(0)'; // Unnecessary!
    });
  }
}
```

## Layout Thrashing

### Read-Write Cycles

The performance killer:

```javascript
// Layout thrashing example (BAD)
function layoutThrashing() {
  const elements = document.querySelectorAll('.box');

  elements.forEach(el => {
    // Read: forces layout
    const width = el.offsetWidth;
    
    // Write: invalidates layout
    el.style.width = width + 10 + 'px';
    
    // Read: forces layout AGAIN
    const height = el.offsetHeight;
    
    // Write: invalidates layout AGAIN
    el.style.height = height + 10 + 'px';
  });

  // Result: 2 * elements.length reflows!
  // With 100 elements = 200 reflows!
}

// Optimized version (GOOD)
function optimizedLayout() {
  const elements = document.querySelectorAll('.box');

  // Phase 1: Batch all reads
  const dimensions = Array.from(elements).map(el => ({
    width: el.offsetWidth,
    height: el.offsetHeight
  }));
  // Only ONE reflow for all reads

  // Phase 2: Batch all writes
  elements.forEach((el, i) => {
    el.style.width = dimensions[i].width + 10 + 'px';
    el.style.height = dimensions[i].height + 10 + 'px';
  });
  // Only ONE reflow for all writes

  // Result: Only 2 reflows total!
}
```

### FastDOM Library

Using FastDOM to prevent thrashing:

```javascript
// FastDOM: Batches reads and writes
import fastdom from 'fastdom';

class FastDOMExample {
  // Schedule reads
  scheduleReads() {
    const element = document.querySelector('.box');

    fastdom.measure(() => {
      // All reads batched together
      const width = element.offsetWidth;
      const height = element.offsetHeight;
      
      console.log('Dimensions:', width, height);

      // Schedule writes based on reads
      fastdom.mutate(() => {
        element.style.width = width * 2 + 'px';
        element.style.height = height * 2 + 'px';
      });
    });
  }

  // Multiple reads and writes
  complexLayout() {
    const elements = document.querySelectorAll('.item');

    elements.forEach(el => {
      // Each read scheduled for batch execution
      fastdom.measure(() => {
        const rect = el.getBoundingClientRect();

        // Each write scheduled after all reads
        fastdom.mutate(() => {
          el.style.transform = `translateY(${rect.top}px)`;
        });
      });
    });

    // FastDOM ensures:
    // 1. All measure() callbacks run together
    // 2. Then all mutate() callbacks run together
    // 3. No thrashing!
  }

  // Clear scheduled tasks
  clearScheduled() {
    const readId = fastdom.measure(() => {
      console.log('This might not run');
    });

    // Cancel if needed
    fastdom.clear(readId);
  }
}
```

### RequestAnimationFrame

Coordinating updates with rendering:

```javascript
// Using requestAnimationFrame to batch updates
class RAFScheduler {
  constructor() {
    this.reads = [];
    this.writes = [];
    this.scheduled = false;
  }

  // Schedule a read
  read(callback) {
    this.reads.push(callback);
    this.scheduleFrame();
  }

  // Schedule a write
  write(callback) {
    this.writes.push(callback);
    this.scheduleFrame();
  }

  scheduleFrame() {
    if (this.scheduled) return;
    
    this.scheduled = true;
    requestAnimationFrame(() => this.flush());
  }

  flush() {
    // Execute all reads first
    const reads = this.reads;
    this.reads = [];
    reads.forEach(read => read());

    // Then all writes
    const writes = this.writes;
    this.writes = [];
    writes.forEach(write => write());

    this.scheduled = false;
  }
}

// Usage
const scheduler = new RAFScheduler();

// Schedule reads
scheduler.read(() => {
  const width = element.offsetWidth;
  console.log('Width:', width);
});

scheduler.read(() => {
  const height = element.offsetHeight;
  console.log('Height:', height);
});

// Schedule writes
scheduler.write(() => {
  element.style.width = '200px';
});

scheduler.write(() => {
  element.style.height = '200px';
});

// All reads execute together, then all writes
// Only 2 reflows total (read batch + write batch)
```

## Optimizing Reflow and Repaint

### Batching DOM Changes

Minimize reflow/repaint operations:

```javascript
// Optimization strategies
class DOMOptimizer {
  // 1. Use document fragments
  addManyElementsOptimized() {
    const fragment = document.createDocumentFragment();

    // Add to fragment (no reflow)
    for (let i = 0; i < 1000; i++) {
      const div = document.createElement('div');
      div.textContent = `Item ${i}`;
      fragment.appendChild(div); // No reflow yet
    }

    // Single reflow when appending fragment
    document.body.appendChild(fragment); // ONE reflow
  }

  // 2. Clone and replace
  modifyManyProperties() {
    const element = document.querySelector('.box');
    
    // Clone element
    const clone = element.cloneNode(true);

    // Modify clone (offline, no reflow)
    clone.style.width = '200px';
    clone.style.height = '200px';
    clone.style.padding = '20px';
    clone.style.margin = '10px';

    // Replace original (single reflow)
    element.parentNode.replaceChild(clone, element);
  }

  // 3. Use CSS classes
  updateStylesWithClass() {
    const element = document.querySelector('.box');

    // BAD: Multiple reflows
    element.style.width = '200px';    // Reflow
    element.style.height = '200px';   // Reflow
    element.style.padding = '20px';   // Reflow

    // GOOD: Single reflow
    element.classList.add('large');   // ONE reflow
    // .large { width: 200px; height: 200px; padding: 20px; }
  }

  // 4. Take element offline
  modifyOffline() {
    const element = document.querySelector('.list');

    // Remove from DOM
    const parent = element.parentNode;
    parent.removeChild(element); // Reflow

    // Modify offline (no reflow)
    for (let i = 0; i < 100; i++) {
      const item = document.createElement('li');
      item.textContent = `Item ${i}`;
      element.appendChild(item); // No reflow
    }

    // Add back to DOM
    parent.appendChild(element); // Reflow

    // Total: 2 reflows (vs 100 if done while in DOM)
  }

  // 5. Cache layout properties
  cacheLayoutProperties() {
    const element = document.querySelector('.box');

    // BAD: Multiple reads
    for (let i = 0; i < 100; i++) {
      const width = element.offsetWidth; // Reflow each time!
      console.log(width);
    }

    // GOOD: Cache the value
    const width = element.offsetWidth; // ONE reflow
    for (let i = 0; i < 100; i++) {
      console.log(width); // No reflow
    }
  }
}
```

### Debouncing and Throttling

Limit reflow frequency:

```javascript
// Debounce: Execute after quiet period
function debounce(func, wait) {
  let timeout;
  
  return function executedFunction(...args) {
    const later = () => {
      clearTimeout(timeout);
      func(...args);
    };

    clearTimeout(timeout);
    timeout = setTimeout(later, wait);
  };
}

// Throttle: Execute at most once per period
function throttle(func, limit) {
  let inThrottle;
  
  return function executedFunction(...args) {
    if (!inThrottle) {
      func(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Usage examples
class EventOptimization {
  constructor() {
    this.setupListeners();
  }

  setupListeners() {
    // Debounced resize (execute once after resizing stops)
    window.addEventListener('resize', debounce(() => {
      this.handleResize();
    }, 250));

    // Throttled scroll (execute at most every 100ms)
    window.addEventListener('scroll', throttle(() => {
      this.handleScroll();
    }, 100));

    // Debounced input (execute after user stops typing)
    const input = document.querySelector('#search');
    input.addEventListener('input', debounce((e) => {
      this.handleSearch(e.target.value);
    }, 300));
  }

  handleResize() {
    // Expensive layout operations
    console.log('Resize handled');
  }

  handleScroll() {
    // Scroll-based updates
    console.log('Scroll handled');
  }

  handleSearch(query) {
    // API calls, filtering, etc.
    console.log('Search:', query);
  }
}
```

### Virtual Scrolling

Rendering only visible items:

```javascript
// Virtual scrolling implementation
class VirtualScroller {
  constructor(container, items, itemHeight) {
    this.container = container;
    this.items = items;
    this.itemHeight = itemHeight;
    this.visibleCount = Math.ceil(container.clientHeight / itemHeight);
    
    this.render();
    this.setupScrollListener();
  }

  render() {
    const scrollTop = this.container.scrollTop;
    const startIndex = Math.floor(scrollTop / this.itemHeight);
    const endIndex = startIndex + this.visibleCount;

    // Only render visible items
    const visibleItems = this.items.slice(startIndex, endIndex);

    // Clear container
    this.container.innerHTML = '';

    // Create spacer for items above viewport
    const topSpacer = document.createElement('div');
    topSpacer.style.height = startIndex * this.itemHeight + 'px';
    this.container.appendChild(topSpacer);

    // Render visible items
    visibleItems.forEach((item, index) => {
      const element = this.createItemElement(item);
      element.style.height = this.itemHeight + 'px';
      this.container.appendChild(element);
    });

    // Create spacer for items below viewport
    const bottomSpacer = document.createElement('div');
    const remainingItems = this.items.length - endIndex;
    bottomSpacer.style.height = remainingItems * this.itemHeight + 'px';
    this.container.appendChild(bottomSpacer);
  }

  createItemElement(item) {
    const div = document.createElement('div');
    div.textContent = item.name;
    div.className = 'item';
    return div;
  }

  setupScrollListener() {
    this.container.addEventListener('scroll', () => {
      requestAnimationFrame(() => this.render());
    });
  }
}

// Usage: Render 10,000 items but only paint ~20 visible
const items = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Item ${i}`
}));

const scroller = new VirtualScroller(
  document.querySelector('.scroll-container'),
  items,
  50 // Item height
);
```

## Common Misconceptions

### Misconception 1: "display: none doesn't affect performance"

**Reality**: While it skips layout/paint, it's still in DOM and affects memory.

```javascript
// display: none:
// ✓ In DOM (memory cost)
// ✓ Styles calculated
// ✗ No layout (no reflow cost)
// ✗ No paint (no repaint cost)

// For truly unused content, remove from DOM:
element.remove(); // Better for memory

// Or use content-visibility: auto (modern)
element.style.contentVisibility = 'auto';
// Browser skips rendering work for off-screen content
```

### Misconception 2: "All CSS changes cause reflow"

**Reality**: Only geometry-affecting properties cause reflow.

```javascript
// These cause REFLOW:
element.style.width = '100px';
element.style.padding = '10px';
element.style.display = 'inline';

// These cause only REPAINT:
element.style.color = 'red';
element.style.backgroundColor = 'blue';
element.style.visibility = 'hidden';

// These cause only COMPOSITE:
element.style.transform = 'translateX(100px)';
element.style.opacity = '0.5';
```

### Misconception 3: "Reading layout properties is free"

**Reality**: Reading forces immediate layout if it's invalidated.

```javascript
// Forced synchronous layout
element.style.width = '100px'; // Layout invalidated

const width = element.offsetWidth; // Forces immediate layout!
// Browser must recalculate layout NOW to return accurate value

// Avoid by batching reads before writes
```

## Performance Implications

### Measuring Reflow/Repaint

Tracking layout performance:

```javascript
class LayoutPerformanceMonitor {
  measureLayoutTime(callback) {
    const start = performance.now();
    
    callback();
    
    // Force layout if needed
    document.body.offsetHeight;
    
    const end = performance.now();
    const duration = end - start;
    
    console.log(`Layout time: ${duration.toFixed(2)}ms`);
    
    if (duration > 16.67) {
      console.warn('Layout took longer than one frame (16.67ms)!');
    }
    
    return duration;
  }

  profileLayout() {
    // Chrome DevTools: Performance tab
    // Shows: Layout events, timing, call stacks

    console.profile('Layout Profile');
    
    // Code that triggers layout
    this.expensiveLayoutOperation();
    
    console.profileEnd('Layout Profile');
  }

  detectLayoutThrashing() {
    const layoutEvents = [];
    
    // Mock: In reality, use Performance Observer
    const observer = new PerformanceObserver(list => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'measure') {
          layoutEvents.push({
            name: entry.name,
            duration: entry.duration,
            startTime: entry.startTime
          });
        }
      }
    });

    observer.observe({ entryTypes: ['measure'] });

    // Detect rapid layout events (thrashing)
    const threshold = 16.67; // One frame
    const recentEvents = layoutEvents.filter(e => 
      e.startTime > performance.now() - threshold
    );

    if (recentEvents.length > 3) {
      console.warn('Possible layout thrashing detected!');
      console.log('Layout events:', recentEvents);
    }
  }

  expensiveLayoutOperation() {
    // Simulate expensive operation
    const elements = document.querySelectorAll('.item');
    
    elements.forEach(el => {
      const width = el.offsetWidth;
      el.style.width = width + 1 + 'px';
    });
  }
}
```

### Performance Budget

Setting limits:

```javascript
class PerformanceBudget {
  constructor() {
    this.budgets = {
      layoutTime: 10,      // Max 10ms per layout
      paintTime: 5,        // Max 5ms per paint
      compositeTime: 2,    // Max 2ms per composite
      totalFrameTime: 16.67 // 60fps = 16.67ms per frame
    };
  }

  checkBudget(operation, duration) {
    const budget = this.budgets[operation];
    
    if (duration > budget) {
      console.warn(`Budget exceeded for ${operation}:`);
      console.warn(`  Budget: ${budget}ms`);
      console.warn(`  Actual: ${duration.toFixed(2)}ms`);
      console.warn(`  Overage: ${(duration - budget).toFixed(2)}ms`);
      
      // Send to analytics
      this.reportViolation(operation, duration, budget);
    }
  }

  reportViolation(operation, duration, budget) {
    // Report to performance monitoring service
    console.log('Sending performance violation to analytics');
  }

  measureOperation(name, callback) {
    const start = performance.now();
    
    callback();
    
    const end = performance.now();
    const duration = end - start;
    
    this.checkBudget(name, duration);
    
    return duration;
  }
}

// Usage
const budget = new PerformanceBudget();

budget.measureOperation('layoutTime', () => {
  // Layout-triggering code
  elements.forEach(el => {
    el.style.width = '100px';
  });
});
```

## Interview Questions

### Question 1: What's the difference between reflow and repaint?

**Answer**:

**Reflow (Layout)**:
- Recalculates positions and sizes
- Triggered by: geometry changes (width, height, padding, etc.)
- Expensive: affects layout tree
- Example: `element.style.width = '100px'`

**Repaint (Paint)**:
- Redraws pixels without layout changes
- Triggered by: visual changes (color, background, etc.)
- Less expensive: only affected elements repainted
- Example: `element.style.color = 'red'`

**Neither (Composite)**:
- Changes composite layers only
- Properties: transform, opacity
- Cheapest: GPU-accelerated
- Example: `element.style.transform = 'translateX(100px)'`

### Question 2: What is layout thrashing and how do you avoid it?

**Answer**: Layout thrashing occurs when you repeatedly read and write layout properties, forcing multiple synchronous layouts:

```javascript
// BAD: Layout thrashing
elements.forEach(el => {
  const width = el.offsetWidth;  // Read: forces layout
  el.style.width = width + 'px'; // Write: invalidates layout
}); // Reflows: elements.length

// GOOD: Batch reads and writes
const widths = elements.map(el => el.offsetWidth); // All reads
elements.forEach((el, i) => {
  el.style.width = widths[i] + 'px'; // All writes
}); // Reflows: 2
```

**Prevention**:
- Batch all reads, then all writes
- Use requestAnimationFrame
- Use libraries like FastDOM
- Cache layout values

### Question 3: Which CSS properties cause reflow vs repaint vs composite?

**Answer**:

**Reflow**: width, height, padding, margin, position, display, float, font-size, line-height

**Repaint**: color, background-color, border-color, box-shadow, border-radius, visibility

**Composite**: transform, opacity, filter (on composited layer)

**Rule**: Use transform/opacity for animations for best performance.

### Question 4: How do you optimize animations for 60fps?

**Answer**:

1. **Use composite-only properties**: transform, opacity
2. **Avoid layout properties**: Don't animate width, height, top, left
3. **Create layers**: Use will-change or translateZ(0)
4. **Use requestAnimationFrame**: Sync with browser rendering
5. **CSS animations/transitions**: Browser can optimize

```javascript
// BAD: Animating left (reflow every frame)
element.style.left = '100px';

// GOOD: Animating transform (composite only)
element.style.transform = 'translateX(100px)';
```

### Question 5: What properties force synchronous layout?

**Answer**: Reading these properties forces immediate layout calculation if layout is invalidated:

- offsetTop, offsetLeft, offsetWidth, offsetHeight
- clientTop, clientLeft, clientWidth, clientHeight
- scrollTop, scrollLeft, scrollWidth, scrollHeight
- getBoundingClientRect(), getClientRects()
- getComputedStyle()

**Impact**: If you write styles then read these properties, browser must recalculate layout immediately, blocking execution.

### Question 6: How does virtual scrolling improve performance?

**Answer**: Virtual scrolling renders only visible items:

**Without virtual scrolling** (10,000 items):
- 10,000 DOM nodes created
- 10,000 layout calculations
- Heavy memory usage
- Slow scrolling

**With virtual scrolling**:
- Only ~20 DOM nodes (visible items)
- Layout only for visible items
- Low memory usage
- Smooth scrolling

**Trade-off**: More complex implementation, but essential for large lists.

### Question 7: When should you use will-change?

**Answer**: Use `will-change` to hint browser about upcoming changes:

```css
.animated {
  will-change: transform, opacity;
}
```

**When to use**:
- Before starting animation
- For elements that will animate frequently
- When you need immediate layer creation

**When NOT to use**:
- On too many elements (memory cost)
- On static elements
- As a permanent style

**Best practice**: Add before animation, remove after.

### Question 8: How do you debug reflow/repaint issues?

**Answer**: Tools and techniques:

1. **Chrome DevTools Performance Tab**:
   - Record timeline
   - Look for purple (layout) and green (paint) bars
   - Check for long tasks (>50ms)

2. **Rendering Tab**:
   - Paint flashing: Shows repainted areas
   - Layout shift regions: Shows reflows
   - FPS meter: Monitor frame rate

3. **Code profiling**:
   ```javascript
   console.time('operation');
   expensiveOperation();
   console.timeEnd('operation');
   ```

4. **Performance Observer**:
   ```javascript
   new PerformanceObserver(list => {
     for (const entry of list.getEntries()) {
       if (entry.duration > 16) {
         console.warn('Long task:', entry);
       }
     }
   }).observe({ entryTypes: ['measure'] });
   ```

## Key Takeaways

1. **Reflow recalculates layout** (expensive), repaint redraws pixels (cheaper), composite combines layers (cheapest)
2. **Layout thrashing kills performance** - batch all reads before writes
3. **Use transform and opacity** for animations - they're composite-only
4. **Reading layout properties can force synchronous layout** - cache values when possible
5. **Virtual scrolling essential for long lists** - render only visible items
6. **will-change creates layers** but costs memory - use sparingly
7. **Debounce/throttle event handlers** to limit reflow frequency
8. **Use Chrome DevTools** to identify and fix layout/paint issues

## Resources

### Official Documentation
- [Chrome's Rendering Performance](https://web.dev/rendering-performance/)
- [MDN: Reflow and Repaint](https://developer.mozilla.org/en-US/docs/Glossary/Reflow)
- [Paul Irish: What Forces Layout](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)

### Articles & Tutorials
- "Avoiding Layout Thrashing" by Wilson Page
- "High Performance Animations" by Paul Lewis
- "Optimize JavaScript Execution" by Addy Osmani

### Videos
- "Rendering Performance" (Chrome Dev Summit)
- "60fps on the Mobile Web" by Flipboard
- "Jank Free: Chrome Rendering Performance" by Paul Irish

### Tools
- Chrome DevTools Performance Tab
- Chrome DevTools Rendering Tab
- Lighthouse Performance Audits
- FastDOM library

### Books
- "High Performance Browser Networking" by Ilya Grigorik
- "Even Faster Web Sites" by Steve Souders
