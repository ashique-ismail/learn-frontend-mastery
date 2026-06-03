# Picture element

## Overview

The `<picture>` element provides art direction and format flexibility for responsive images. It allows different images (not just different sizes) based on media conditions or format support.

## Basic structure

```html
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg" type="image/jpeg">
  <img src="image.jpg" alt="Fallback image">
</picture>
```

## Use cases

### Art direction

Different images for different screen sizes:

```html
<picture>
  <!-- Mobile: vertical crop -->
  <source 
    media="(max-width: 640px)"
    srcset="mobile-portrait.jpg">
  
  <!-- Tablet: square crop -->
  <source 
    media="(max-width: 1024px)"
    srcset="tablet-square.jpg">
  
  <!-- Desktop: horizontal crop -->
  <img 
    src="desktop-landscape.jpg" 
    alt="Responsive hero image">
</picture>
```

### Format fallback

Modern formats with fallbacks:

```html
<picture>
  <!-- Try AVIF first -->
  <source srcset="image.avif" type="image/avif">
  
  <!-- Then WebP -->
  <source srcset="image.webp" type="image/webp">
  
  <!-- Finally JPEG -->
  <img src="image.jpg" alt="Image with format fallback">
</picture>
```

### Combining format and art direction

```html
<picture>
  <!-- Mobile: AVIF -->
  <source 
    media="(max-width: 640px)"
    srcset="mobile.avif"
    type="image/avif">
  
  <!-- Mobile: WebP -->
  <source 
    media="(max-width: 640px)"
    srcset="mobile.webp"
    type="image/webp">
  
  <!-- Mobile: JPEG -->
  <source 
    media="(max-width: 640px)"
    srcset="mobile.jpg">
  
  <!-- Desktop: AVIF -->
  <source 
    srcset="desktop.avif"
    type="image/avif">
  
  <!-- Desktop: WebP -->
  <source 
    srcset="desktop.webp"
    type="image/webp">
  
  <!-- Desktop: JPEG fallback -->
  <img src="desktop.jpg" alt="Optimized responsive image">
</picture>
```

## srcset and sizes in picture

### With width descriptors

```html
<picture>
  <source 
    media="(max-width: 640px)"
    srcset="mobile-400.jpg 400w, mobile-800.jpg 800w"
    sizes="100vw">
  
  <source 
    media="(max-width: 1024px)"
    srcset="tablet-600.jpg 600w, tablet-1200.jpg 1200w"
    sizes="100vw">
  
  <img 
    src="desktop-1000.jpg"
    srcset="desktop-1000.jpg 1000w, desktop-2000.jpg 2000w"
    sizes="100vw"
    alt="Multi-resolution responsive image">
</picture>
```

### With density descriptors

```html
<picture>
  <source 
    media="(max-width: 640px)"
    srcset="mobile.jpg 1x, mobile@2x.jpg 2x">
  
  <img 
    src="desktop.jpg"
    srcset="desktop.jpg 1x, desktop@2x.jpg 2x"
    alt="High DPI responsive image">
</picture>
```

## Dark mode support

```html
<picture>
  <!-- Dark mode -->
  <source 
    srcset="logo-dark.svg"
    media="(prefers-color-scheme: dark)">
  
  <!-- Light mode -->
  <img src="logo-light.svg" alt="Company logo">
</picture>
```

## Orientation-based images

```html
<picture>
  <source 
    srcset="landscape.jpg"
    media="(orientation: landscape)">
  
  <source 
    srcset="portrait.jpg"
    media="(orientation: portrait)">
  
  <img src="square.jpg" alt="Orientation-aware image">
</picture>
```

## Resolution-based selection

```html
<picture>
  <!-- High resolution displays -->
  <source 
    srcset="high-quality.jpg"
    media="(min-resolution: 2dppx)">
  
  <!-- Standard displays -->
  <img src="standard-quality.jpg" alt="Resolution-optimized image">
</picture>
```

## Complete example: Hero image

```html
<picture>
  <!-- Mobile portrait: AVIF -->
  <source 
    media="(max-width: 640px)"
    srcset="hero-mobile-400.avif 400w,
            hero-mobile-800.avif 800w"
    sizes="100vw"
    type="image/avif">
  
  <!-- Mobile portrait: WebP -->
  <source 
    media="(max-width: 640px)"
    srcset="hero-mobile-400.webp 400w,
            hero-mobile-800.webp 800w"
    sizes="100vw"
    type="image/webp">
  
  <!-- Mobile portrait: JPEG -->
  <source 
    media="(max-width: 640px)"
    srcset="hero-mobile-400.jpg 400w,
            hero-mobile-800.jpg 800w"
    sizes="100vw">
  
  <!-- Tablet: AVIF -->
  <source 
    media="(max-width: 1024px)"
    srcset="hero-tablet-800.avif 800w,
            hero-tablet-1600.avif 1600w"
    sizes="100vw"
    type="image/avif">
  
  <!-- Tablet: WebP -->
  <source 
    media="(max-width: 1024px)"
    srcset="hero-tablet-800.webp 800w,
            hero-tablet-1600.webp 1600w"
    sizes="100vw"
    type="image/webp">
  
  <!-- Tablet: JPEG -->
  <source 
    media="(max-width: 1024px)"
    srcset="hero-tablet-800.jpg 800w,
            hero-tablet-1600.jpg 1600w"
    sizes="100vw">
  
  <!-- Desktop: AVIF -->
  <source 
    srcset="hero-desktop-1200.avif 1200w,
            hero-desktop-1600.avif 1600w,
            hero-desktop-2400.avif 2400w"
    sizes="100vw"
    type="image/avif">
  
  <!-- Desktop: WebP -->
  <source 
    srcset="hero-desktop-1200.webp 1200w,
            hero-desktop-1600.webp 1600w,
            hero-desktop-2400.webp 2400w"
    sizes="100vw"
    type="image/webp">
  
  <!-- Desktop fallback -->
  <img 
    src="hero-desktop-1200.jpg"
    srcset="hero-desktop-1200.jpg 1200w,
            hero-desktop-1600.jpg 1600w,
            hero-desktop-2400.jpg 2400w"
    sizes="100vw"
    alt="Hero banner"
    loading="eager"
    fetchpriority="high"
    width="1200"
    height="600">
</picture>
```

## Performance attributes

```html
<picture>
  <source srcset="image.webp" type="image/webp">
  <img 
    src="image.jpg" 
    alt="Lazy loaded image"
    loading="lazy"
    decoding="async"
    width="800"
    height="600">
</picture>
```

## Common patterns

### Blog post hero

```html
<picture>
  <source 
    media="(max-width: 640px)"
    srcset="post-hero-mobile.webp"
    type="image/webp">
  <source 
    media="(max-width: 640px)"
    srcset="post-hero-mobile.jpg">
  
  <source 
    srcset="post-hero-desktop.webp"
    type="image/webp">
  <img 
    src="post-hero-desktop.jpg" 
    alt="Article hero image"
    loading="eager"
    width="1200"
    height="630">
</picture>
```

### Product image

```html
<picture>
  <source 
    srcset="product.avif 1x, product@2x.avif 2x"
    type="image/avif">
  <source 
    srcset="product.webp 1x, product@2x.webp 2x"
    type="image/webp">
  <img 
    src="product.jpg"
    srcset="product.jpg 1x, product@2x.jpg 2x"
    alt="Product image"
    loading="lazy"
    width="600"
    height="600">
</picture>
```

### Avatar with fallback

```html
<picture>
  <source srcset="avatar.webp" type="image/webp">
  <img 
    src="avatar.jpg" 
    alt="User avatar"
    width="48"
    height="48"
    loading="lazy">
</picture>
```

## Browser behavior

### Source evaluation order

The browser evaluates `<source>` elements in order and uses the first match:

1. Checks media query (if present)
2. Checks type support (if present)
3. Uses first matching source
4. Falls back to `<img>` if no match

### Type checking

```html
<picture>
  <!-- Only used if browser supports AVIF -->
  <source srcset="image.avif" type="image/avif">
  
  <!-- Only used if browser supports WebP and no AVIF -->
  <source srcset="image.webp" type="image/webp">
  
  <!-- Used if neither AVIF nor WebP supported -->
  <img src="image.jpg" alt="Image">
</picture>
```

## Accessibility

The `alt` text goes on the `<img>` element:

```html
<picture>
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Descriptive alt text here">
</picture>
```

## When to use picture vs img

### Use picture when:
- Different images for different screens (art direction)
- Format fallbacks (AVIF, WebP, JPEG)
- Dark mode images
- Orientation-specific images

### Use img with srcset when:
- Same image, different sizes
- Density descriptors only
- Simple responsive images

## Best practices

1. Order sources from most to least specific
2. Always include fallback `<img>`
3. Put modern formats first (AVIF, then WebP, then JPEG)
4. Use `loading="lazy"` on below-fold images
5. Always include `width` and `height` on `<img>`
6. Include `alt` text on the `<img>` element
7. Test format support across browsers
8. Keep media queries consistent with CSS

## Format recommendations

### Modern stack (2025)
```html
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Image">
</picture>
```

### Maximum compatibility
```html
<picture>
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Image">
</picture>
```

## Key takeaways

- Use `<picture>` for art direction and format flexibility
- Use `<img>` with `srcset` for simple responsive images
- Order matters: most specific first, fallback last
- Modern formats first (AVIF, WebP), JPEG last
- All performance attributes go on `<img>` element
- Always include fallback `<img>` with `src` and `alt`
