# Image srcset, sizes, and loading

## Overview

Modern image elements support responsive images through `srcset` and `sizes` attributes, and performance optimization through `loading` and `decoding` attributes.

## srcset attribute

Provides multiple image sources for different display conditions.

### Density descriptors (x)

For different pixel densities:

```html
<img 
  src="image.jpg" 
  srcset="image.jpg 1x, image@2x.jpg 2x, image@3x.jpg 3x"
  alt="Responsive image">
```

### Width descriptors (w)

For different viewport widths:

```html
<img 
  src="image-800.jpg"
  srcset="image-400.jpg 400w,
          image-800.jpg 800w,
          image-1200.jpg 1200w,
          image-1600.jpg 1600w"
  sizes="(max-width: 600px) 100vw,
         (max-width: 1000px) 50vw,
         800px"
  alt="Responsive image">
```

## sizes attribute

Tells the browser how much space the image will take at different viewport sizes.

### Basic syntax

```html
<img 
  srcset="small.jpg 400w, medium.jpg 800w, large.jpg 1200w"
  sizes="(max-width: 600px) 100vw, 50vw"
  src="medium.jpg"
  alt="Image">
```

### Complex layout example

```html
<img 
  srcset="hero-400.jpg 400w,
          hero-800.jpg 800w,
          hero-1200.jpg 1200w,
          hero-1600.jpg 1600w,
          hero-2000.jpg 2000w"
  sizes="(max-width: 640px) 100vw,
         (max-width: 1024px) calc(100vw - 40px),
         (max-width: 1280px) 960px,
         1200px"
  src="hero-800.jpg"
  alt="Hero image">
```

## loading attribute

Controls when images load.

### Lazy loading

```html
<img src="image.jpg" loading="lazy" alt="Lazy loaded image">
```

### Eager loading (default)

```html
<img src="image.jpg" loading="eager" alt="Immediately loaded image">
```

### Best practices

```html
<!-- Above-the-fold: eager loading -->
<img src="hero.jpg" loading="eager" alt="Hero image">

<!-- Below-the-fold: lazy loading -->
<img src="gallery-1.jpg" loading="lazy" alt="Gallery image 1">
<img src="gallery-2.jpg" loading="lazy" alt="Gallery image 2">
<img src="gallery-3.jpg" loading="lazy" alt="Gallery image 3">

<!-- Critical images: fetchpriority -->
<img src="lcp-image.jpg" loading="eager" fetchpriority="high" alt="LCP image">
```

## decoding attribute

Controls image decode timing.

```html
<!-- Async decoding (default for most browsers) -->
<img src="image.jpg" decoding="async" alt="Image">

<!-- Sync decoding -->
<img src="critical.jpg" decoding="sync" alt="Critical image">

<!-- Auto (browser decides) -->
<img src="image.jpg" decoding="auto" alt="Image">
```

## fetchpriority attribute

Hints at the relative priority of fetching.

```html
<!-- High priority (LCP image) -->
<img src="hero.jpg" fetchpriority="high" alt="Hero">

<!-- Low priority -->
<img src="footer-logo.jpg" fetchpriority="low" alt="Logo">

<!-- Auto (default) -->
<img src="content.jpg" fetchpriority="auto" alt="Content">
```

## Complete responsive image example

```html
<img 
  src="product-800.jpg"
  srcset="product-400.jpg 400w,
          product-800.jpg 800w,
          product-1200.jpg 1200w,
          product-1600.jpg 1600w"
  sizes="(max-width: 640px) 100vw,
         (max-width: 1024px) 50vw,
         33vw"
  alt="Product showcase"
  loading="lazy"
  decoding="async"
  width="800"
  height="600">
```

## Art direction with picture

When you need different images (not just sizes):

```html
<picture>
  <source 
    media="(max-width: 640px)"
    srcset="mobile-400.jpg 400w, mobile-800.jpg 800w"
    sizes="100vw">
  <source 
    media="(max-width: 1024px)"
    srcset="tablet-800.jpg 800w, tablet-1600.jpg 1600w"
    sizes="100vw">
  <img 
    src="desktop-1200.jpg"
    srcset="desktop-1200.jpg 1200w, desktop-2400.jpg 2400w"
    sizes="100vw"
    alt="Responsive hero"
    loading="lazy">
</picture>
```

## Performance patterns

### Above-the-fold hero

```html
<img 
  src="hero-1200.jpg"
  srcset="hero-800.jpg 800w,
          hero-1200.jpg 1200w,
          hero-1600.jpg 1600w,
          hero-2400.jpg 2400w"
  sizes="100vw"
  alt="Hero image"
  loading="eager"
  fetchpriority="high"
  decoding="async"
  width="1200"
  height="600">
```

### Gallery images

```html
<img 
  src="gallery-400.jpg"
  srcset="gallery-400.jpg 400w,
          gallery-800.jpg 800w"
  sizes="(max-width: 640px) 50vw, 25vw"
  alt="Gallery item"
  loading="lazy"
  decoding="async"
  width="400"
  height="300">
```

### Thumbnail grid

```html
<img 
  src="thumb-200.jpg"
  srcset="thumb-200.jpg 1x, thumb-400.jpg 2x"
  alt="Thumbnail"
  loading="lazy"
  width="200"
  height="200">
```

## Browser selection algorithm

The browser selects an image based on:

1. Device pixel ratio
2. Viewport size
3. sizes attribute
4. Network conditions (in some browsers)
5. User preferences (data saver mode)

## Common patterns

### Full-width hero

```html
<img 
  srcset="hero-640.jpg 640w,
          hero-1024.jpg 1024w,
          hero-1920.jpg 1920w,
          hero-2560.jpg 2560w"
  sizes="100vw"
  src="hero-1024.jpg"
  alt="Hero">
```

### Sidebar image

```html
<img 
  srcset="sidebar-300.jpg 300w,
          sidebar-600.jpg 600w"
  sizes="(max-width: 1024px) 100vw, 300px"
  src="sidebar-300.jpg"
  alt="Sidebar">
```

### Article image

```html
<img 
  srcset="article-600.jpg 600w,
          article-1200.jpg 1200w"
  sizes="(max-width: 768px) 100vw,
         (max-width: 1200px) 80vw,
         800px"
  src="article-600.jpg"
  alt="Article">
```

## Key takeaways

- Use `srcset` with `w` descriptors for different sizes
- Use `sizes` to tell browser how much space image takes
- Use `loading="lazy"` for below-the-fold images
- Use `fetchpriority="high"` for LCP images
- Always include fallback `src`
- Always include `width` and `height` to prevent layout shift
- Use `decoding="async"` for non-critical images
- Combine with `<picture>` for art direction
