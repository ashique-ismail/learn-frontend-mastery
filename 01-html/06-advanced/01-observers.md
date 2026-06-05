# Observers

## The Idea

**In plain English:** Observers are built-in browser helpers that watch specific things on a webpage — like whether an element has scrolled into view, changed size, or been modified — and automatically run your code when that happens. Instead of your code constantly asking "has anything changed yet?", an observer taps your shoulder the moment something does.

**Real-world analogy:** Imagine a security guard watching a set of doors in a building. You give the guard a list of doors to watch and a note saying "call me if any of these open." The guard stays alert without you having to run back every few seconds to check yourself.

- The guard = the Observer (e.g., IntersectionObserver)
- The doors being watched = the HTML elements you call `.observe()` on
- Your note with instructions = the callback function you pass to the Observer

---

## Overview

Observer APIs allow you to watch for changes in the DOM, element visibility, size, and more without constantly polling. They use callbacks and are highly performant.

## Intersection Observer

Observes when elements enter or leave the viewport.

### Basic usage

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      console.log('Element is visible:', entry.target);
    } else {
      console.log('Element is not visible:', entry.target);
    }
  });
});

const element = document.getElementById('my-element');
observer.observe(element);
```

### Options

```javascript
const options = {
  root: null,           // viewport (default) or specific element
  rootMargin: '0px',    // margin around root
  threshold: 0.5        // 0 to 1, or array [0, 0.25, 0.5, 0.75, 1]
};

const observer = new IntersectionObserver(callback, options);
```

### Lazy loading images

```html
<img class="lazy" data-src="image.jpg" alt="Description">
<img class="lazy" data-src="image2.jpg" alt="Description">
<img class="lazy" data-src="image3.jpg" alt="Description">

<script>
const lazyImages = document.querySelectorAll('.lazy');

const imageObserver = new IntersectionObserver((entries, observer) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      img.classList.remove('lazy');
      observer.unobserve(img); // Stop observing once loaded
    }
  });
});

lazyImages.forEach(img => imageObserver.observe(img));
</script>
```

### Infinite scroll

```html
<div id="content">
  <!-- Content items -->
</div>
<div id="sentinel"></div>

<script>
let page = 1;
let loading = false;

const sentinel = document.getElementById('sentinel');
const content = document.getElementById('content');

const observer = new IntersectionObserver(async (entries) => {
  if (entries[0].isIntersecting && !loading) {
    loading = true;
    
    // Load more content
    const newItems = await loadPage(page++);
    content.insertAdjacentHTML('beforeend', newItems);
    
    loading = false;
  }
});

observer.observe(sentinel);

async function loadPage(page) {
  const response = await fetch(`/api/items?page=${page}`);
  const items = await response.json();
  return items.map(item => `<div class="item">${item.title}</div>`).join('');
}
</script>
```

### Fade in on scroll

```html
<div class="fade-in">Content 1</div>
<div class="fade-in">Content 2</div>
<div class="fade-in">Content 3</div>

<style>
.fade-in {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity 0.6s ease, transform 0.6s ease;
}

.fade-in.visible {
  opacity: 1;
  transform: translateY(0);
}
</style>

<script>
const fadeElements = document.querySelectorAll('.fade-in');

const fadeObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('visible');
      fadeObserver.unobserve(entry.target);
    }
  });
}, {
  threshold: 0.1
});

fadeElements.forEach(el => fadeObserver.observe(el));
</script>
```

### Tracking element visibility

```javascript
const tracker = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    const ratio = entry.intersectionRatio;
    const visible = entry.isIntersecting;
    
    console.log(`Element ${visible ? 'visible' : 'hidden'}, ${(ratio * 100).toFixed(0)}% shown`);
    
    // Track in analytics
    if (visible && ratio > 0.5) {
      trackView(entry.target.id);
    }
  });
}, {
  threshold: [0, 0.25, 0.5, 0.75, 1]
});

const ads = document.querySelectorAll('.advertisement');
ads.forEach(ad => tracker.observe(ad));
```

## Mutation Observer

Observes changes to the DOM tree.

### Basic usage

```javascript
const observer = new MutationObserver((mutations) => {
  mutations.forEach(mutation => {
    console.log('Type:', mutation.type);
    console.log('Target:', mutation.target);
    
    if (mutation.type === 'childList') {
      console.log('Added:', mutation.addedNodes);
      console.log('Removed:', mutation.removedNodes);
    }
    
    if (mutation.type === 'attributes') {
      console.log('Attribute:', mutation.attributeName);
      console.log('Old value:', mutation.oldValue);
    }
  });
});

const config = {
  childList: true,      // Observe child additions/removals
  attributes: true,     // Observe attribute changes
  characterData: true,  // Observe text changes
  subtree: true,        // Observe descendants
  attributeOldValue: true,  // Record old attribute values
  characterDataOldValue: true  // Record old text values
};

const target = document.getElementById('container');
observer.observe(target, config);

// Stop observing
observer.disconnect();
```

### Detecting new elements

```javascript
const observer = new MutationObserver((mutations) => {
  mutations.forEach(mutation => {
    mutation.addedNodes.forEach(node => {
      if (node.nodeType === Node.ELEMENT_NODE) {
        if (node.matches('.dynamic-content')) {
          initializeDynamicContent(node);
        }
      }
    });
  });
});

observer.observe(document.body, {
  childList: true,
  subtree: true
});
```

### Form validation on attribute change

```html
<form>
  <input type="email" id="email" aria-invalid="false">
  <span id="error-message"></span>
</form>

<script>
const input = document.getElementById('email');
const errorMessage = document.getElementById('error-message');

const observer = new MutationObserver((mutations) => {
  mutations.forEach(mutation => {
    if (mutation.attributeName === 'aria-invalid') {
      const isInvalid = input.getAttribute('aria-invalid') === 'true';
      errorMessage.textContent = isInvalid ? 'Invalid email' : '';
    }
  });
});

observer.observe(input, {
  attributes: true,
  attributeFilter: ['aria-invalid']
});

// Validation logic
input.addEventListener('blur', () => {
  const isValid = input.validity.valid;
  input.setAttribute('aria-invalid', !isValid);
});
</script>
```

### Detecting text changes

```javascript
const textObserver = new MutationObserver((mutations) => {
  mutations.forEach(mutation => {
    if (mutation.type === 'characterData') {
      console.log('Text changed from:', mutation.oldValue);
      console.log('Text changed to:', mutation.target.textContent);
    }
  });
});

const textNode = document.getElementById('editable');
textObserver.observe(textNode, {
  characterData: true,
  characterDataOldValue: true,
  subtree: true
});
```

## Resize Observer

Observes changes to element size.

### Basic usage

```javascript
const observer = new ResizeObserver((entries) => {
  entries.forEach(entry => {
    const width = entry.contentRect.width;
    const height = entry.contentRect.height;
    
    console.log(`Size: ${width}x${height}`);
    
    // Respond to size change
    if (width < 600) {
      entry.target.classList.add('compact');
    } else {
      entry.target.classList.remove('compact');
    }
  });
});

const element = document.getElementById('responsive-element');
observer.observe(element);
```

### Responsive components

```javascript
class ResponsiveCard extends HTMLElement {
  connectedCallback() {
    this.resizeObserver = new ResizeObserver((entries) => {
      const width = entries[0].contentRect.width;
      
      if (width < 300) {
        this.classList.add('small');
        this.classList.remove('medium', 'large');
      } else if (width < 600) {
        this.classList.add('medium');
        this.classList.remove('small', 'large');
      } else {
        this.classList.add('large');
        this.classList.remove('small', 'medium');
      }
    });
    
    this.resizeObserver.observe(this);
  }
  
  disconnectedCallback() {
    this.resizeObserver.disconnect();
  }
}

customElements.define('responsive-card', ResponsiveCard);
```

### Chart resizing

```javascript
const chartContainer = document.getElementById('chart');
const chart = createChart(chartContainer);

const resizeObserver = new ResizeObserver((entries) => {
  const { width, height } = entries[0].contentRect;
  chart.resize(width, height);
});

resizeObserver.observe(chartContainer);
```

### Textarea auto-resize

```html
<textarea id="auto-resize" rows="3"></textarea>

<script>
const textarea = document.getElementById('auto-resize');

const resizeObserver = new ResizeObserver(() => {
  textarea.style.height = 'auto';
  textarea.style.height = textarea.scrollHeight + 'px';
});

resizeObserver.observe(textarea);

textarea.addEventListener('input', () => {
  textarea.style.height = 'auto';
  textarea.style.height = textarea.scrollHeight + 'px';
});
</script>
```

## Performance Observer

Observes performance metrics.

### Basic usage

```javascript
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log('Entry:', entry.name);
    console.log('Type:', entry.entryType);
    console.log('Start:', entry.startTime);
    console.log('Duration:', entry.duration);
  });
});

// Observe specific types
observer.observe({ entryTypes: ['measure', 'mark', 'resource'] });
```

### Measuring performance

```javascript
// Mark points in time
performance.mark('start-loading');

// ... do something ...

performance.mark('end-loading');

// Measure between marks
performance.measure('loading-time', 'start-loading', 'end-loading');

// Observe measurements
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    if (entry.name === 'loading-time') {
      console.log(`Loading took ${entry.duration}ms`);
    }
  });
});

observer.observe({ entryTypes: ['measure'] });
```

### Monitoring resource loading

```javascript
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    if (entry.entryType === 'resource') {
      console.log(`Resource: ${entry.name}`);
      console.log(`Duration: ${entry.duration}ms`);
      console.log(`Size: ${entry.transferSize} bytes`);
    }
  });
});

observer.observe({ entryTypes: ['resource'] });
```

### Largest Contentful Paint (LCP)

```javascript
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  
  console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime);
  console.log('Element:', lastEntry.element);
});

observer.observe({ entryTypes: ['largest-contentful-paint'] });
```

### First Input Delay (FID)

```javascript
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    const fid = entry.processingStart - entry.startTime;
    console.log('FID:', fid, 'ms');
  });
});

observer.observe({ entryTypes: ['first-input'] });
```

## Complete example: Image gallery with lazy loading and analytics

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Observer Demo</title>
  <style>
    .gallery {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
      gap: 16px;
      padding: 16px;
    }
    
    .image-container {
      aspect-ratio: 16/9;
      background: #f0f0f0;
      border-radius: 8px;
      overflow: hidden;
      position: relative;
    }
    
    .image-container img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      opacity: 0;
      transition: opacity 0.3s ease;
    }
    
    .image-container img.loaded {
      opacity: 1;
    }
    
    .image-container.viewing {
      outline: 3px solid blue;
    }
  </style>
</head>
<body>
  <div class="gallery" id="gallery">
    <!-- Images will be added here -->
  </div>
  
  <script>
    // Generate image placeholders
    const gallery = document.getElementById('gallery');
    for (let i = 1; i <= 50; i++) {
      gallery.innerHTML += `
        <div class="image-container" data-id="${i}">
          <img data-src="https://picsum.photos/600/400?random=${i}" alt="Image ${i}">
        </div>
      `;
    }
    
    // Lazy loading observer
    const lazyImages = document.querySelectorAll('.image-container img');
    
    const lazyObserver = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const img = entry.target;
          img.src = img.dataset.src;
          
          img.addEventListener('load', () => {
            img.classList.add('loaded');
          });
          
          lazyObserver.unobserve(img);
        }
      });
    }, {
      rootMargin: '100px' // Start loading before entering viewport
    });
    
    lazyImages.forEach(img => lazyObserver.observe(img));
    
    // Visibility tracking observer
    const containers = document.querySelectorAll('.image-container');
    
    const trackingObserver = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting && entry.intersectionRatio > 0.5) {
          entry.target.classList.add('viewing');
          
          // Track impression
          const imageId = entry.target.dataset.id;
          console.log(`Image ${imageId} viewed`);
          trackImpression(imageId);
        } else {
          entry.target.classList.remove('viewing');
        }
      });
    }, {
      threshold: 0.5
    });
    
    containers.forEach(container => trackingObserver.observe(container));
    
    // Resize observer for responsive behavior
    const resizeObserver = new ResizeObserver((entries) => {
      entries.forEach(entry => {
        const width = entry.contentRect.width;
        
        if (width < 250) {
          entry.target.classList.add('compact');
        } else {
          entry.target.classList.remove('compact');
        }
      });
    });
    
    containers.forEach(container => resizeObserver.observe(container));
    
    // Analytics function (mock)
    function trackImpression(imageId) {
      // Send to analytics service
      console.log('Tracking impression:', imageId);
    }
  </script>
</body>
</html>
```

## Best practices

### 1. Clean up observers

```javascript
// Store observer reference
this.observer = new IntersectionObserver(callback);

// Disconnect when done
this.observer.disconnect();

// Or unobserve specific elements
this.observer.unobserve(element);
```

### 2. Use appropriate thresholds

```javascript
// Lazy loading: trigger early
new IntersectionObserver(callback, {
  rootMargin: '100px'
});

// Visibility tracking: require mostly visible
new IntersectionObserver(callback, {
  threshold: 0.75
});
```

### 3. Debounce rapid changes

```javascript
let timeout;

const resizeObserver = new ResizeObserver((entries) => {
  clearTimeout(timeout);
  timeout = setTimeout(() => {
    handleResize(entries);
  }, 100);
});
```

### 4. Check for support

```javascript
if ('IntersectionObserver' in window) {
  // Use IntersectionObserver
} else {
  // Use fallback
  window.addEventListener('scroll', handleScroll);
}
```

## Browser support

- **IntersectionObserver**: All modern browsers
- **MutationObserver**: All modern browsers
- **ResizeObserver**: Chrome 64+, Firefox 69+, Safari 13.1+
- **PerformanceObserver**: All modern browsers

## Key takeaways

- **IntersectionObserver**: Element visibility in viewport
- **MutationObserver**: DOM changes
- **ResizeObserver**: Element size changes
- **PerformanceObserver**: Performance metrics
- More efficient than polling
- Always clean up observers
- Use appropriate thresholds and margins
- Check browser support
- Common uses: lazy loading, infinite scroll, animations, analytics
- Combine observers for complex functionality
