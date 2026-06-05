# Image Optimization

## The Idea

**In plain English:** Image optimization is the process of making image files as small as possible without making them look noticeably worse, so websites load faster. An "image file" is just data stored on a computer that tells your screen what colors to show — optimization squeezes out the parts your eyes won't miss.

**Real-world analogy:** Imagine you're sending a physical photo album through the mail. Instead of mailing the original enormous hardcover book, you photocopy only the pages the recipient needs, shrink each photo to fit the page size they're actually using, and use a compact envelope instead of a giant box. The recipient gets the same photos, but shipping is much faster and cheaper.

- The original photo album = the full-resolution, uncompressed image file on your server
- Choosing which pages to include = selecting the right image format (JPEG, WebP, AVIF) for the content
- Shrinking photos to fit the page = generating multiple sizes so each device downloads only what fits its screen
- The compact envelope = compression that removes invisible detail to reduce file size

---

## Overview

Images typically account for 50-70% of a web page's total size, making image optimization one of the most impactful performance improvements you can make. Proper image optimization involves choosing the right format, size, compression level, and delivery method. Modern techniques include responsive images, next-generation formats (WebP, AVIF), lazy loading, and CDN delivery with automatic optimization.

## Image Format Comparison

```text
Image Format Selection Guide
=============================

Format  | Best For              | Compression | Transparency | Animation | Avg Size
--------|----------------------|-------------|--------------|-----------|----------
JPEG    | Photos, complex      | Lossy       | No           | No        | Baseline
        | color images         | Good (60-85)| |            |           |
--------|----------------------|-------------|--------------|-----------|----------
PNG     | Graphics, logos,     | Lossless    | Yes          | No        | 2-3x JPEG
        | transparency needed  |             |              |           |
--------|----------------------|-------------|--------------|-----------|----------
WebP    | Modern alternative   | Both        | Yes          | Yes       | 25-35%
        | to JPEG/PNG          | Better      |              |           | smaller
--------|----------------------|-------------|--------------|-----------|----------
AVIF    | Next-gen format,     | Both        | Yes          | Yes       | 50%
        | best compression     | Best        |              |           | smaller
--------|----------------------|-------------|--------------|-----------|----------
SVG     | Icons, logos,        | N/A         | Yes          | Yes       | Variable
        | vector graphics      | (Vector)    |              |           | (tiny)
--------|----------------------|-------------|--------------|-----------|----------
GIF     | Simple animations    | Lossless    | Yes (1-bit)  | Yes       | Poor
        | (prefer WebP/MP4)    | Poor        |              |           |


Typical Savings (from 100KB JPEG):
┌─────────────────────────────────┐
│ JPEG:  ████████████  100KB      │
│ PNG:   ██████████████████ 250KB │
│ WebP:  ████████  65KB (-35%)    │
│ AVIF:  ██████  50KB (-50%)      │
│ SVG:   █  5KB (vectors only)    │
└─────────────────────────────────┘
```

## Responsive Images with srcset and sizes

```typescript
// React: Complete responsive image implementation
import { useState, useEffect } from 'react';

interface ResponsiveImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  sizes?: string;
  priority?: boolean;
}

function ResponsiveImage({
  src,
  alt,
  width,
  height,
  sizes,
  priority = false
}: ResponsiveImageProps) {
  const basePath = src.replace(/\.[^/.]+$/, ''); // Remove extension
  
  return (
    <img
      src={`${basePath}.jpg`}
      srcSet={`
        ${basePath}-320w.jpg 320w,
        ${basePath}-640w.jpg 640w,
        ${basePath}-960w.jpg 960w,
        ${basePath}-1280w.jpg 1280w,
        ${basePath}-1920w.jpg 1920w
      `}
      sizes={sizes || `
        (max-width: 320px) 280px,
        (max-width: 640px) 600px,
        (max-width: 960px) 920px,
        (max-width: 1280px) 1200px,
        1920px
      `}
      alt={alt}
      width={width}
      height={height}
      loading={priority ? 'eager' : 'lazy'}
      fetchPriority={priority ? 'high' : 'auto'}
      style={{ maxWidth: '100%', height: 'auto' }}
    />
  );
}

// Modern formats with fallbacks using picture element
function ModernImageFormats({ src, alt, width, height }: any) {
  const basePath = src.replace(/\.[^/.]+$/, '');
  
  return (
    <picture>
      {/* AVIF - best compression, newest format */}
      <source
        type="image/avif"
        srcSet={`
          ${basePath}-320w.avif 320w,
          ${basePath}-640w.avif 640w,
          ${basePath}-960w.avif 960w,
          ${basePath}-1280w.avif 1280w,
          ${basePath}-1920w.avif 1920w
        `}
        sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 33vw"
      />
      
      {/* WebP - good compression, widely supported */}
      <source
        type="image/webp"
        srcSet={`
          ${basePath}-320w.webp 320w,
          ${basePath}-640w.webp 640w,
          ${basePath}-960w.webp 960w,
          ${basePath}-1280w.webp 1280w,
          ${basePath}-1920w.webp 1920w
        `}
        sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 33vw"
      />
      
      {/* JPEG fallback for older browsers */}
      <img
        src={`${basePath}.jpg`}
        srcSet={`
          ${basePath}-320w.jpg 320w,
          ${basePath}-640w.jpg 640w,
          ${basePath}-960w.jpg 960w,
          ${basePath}-1280w.jpg 1280w,
          ${basePath}-1920w.jpg 1920w
        `}
        sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 33vw"
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
        style={{ maxWidth: '100%', height: 'auto' }}
      />
    </picture>
  );
}

// Art direction - different images for different screen sizes
function ArtDirectedImage() {
  return (
    <picture>
      {/* Mobile - portrait crop */}
      <source
        media="(max-width: 640px)"
        srcSet="/images/hero-mobile.jpg"
      />
      
      {/* Tablet - square crop */}
      <source
        media="(max-width: 1024px)"
        srcSet="/images/hero-tablet.jpg"
      />
      
      {/* Desktop - landscape crop */}
      <img
        src="/images/hero-desktop.jpg"
        alt="Hero image"
        width={1920}
        height={1080}
      />
    </picture>
  );
}

// Next.js Image component (built-in optimization)
import Image from 'next/image';

function OptimizedNextImage() {
  return (
    <Image
      src="/images/photo.jpg"
      alt="Optimized photo"
      width={1920}
      height={1080}
      quality={85}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
      priority={false}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    />
  );
}

// Custom hook for device-aware image loading
function useDeviceAwareImage(imageSizes: any[]) {
  const [imageSrc, setImageSrc] = useState(imageSizes[0].url);

  useEffect(() => {
    const updateImage = () => {
      const width = window.innerWidth;
      const dpr = window.devicePixelRatio || 1;
      const targetWidth = width * dpr;

      const optimalImage = imageSizes.reduce((prev, curr) => {
        const prevDiff = Math.abs(prev.width - targetWidth);
        const currDiff = Math.abs(curr.width - targetWidth);
        return currDiff < prevDiff ? curr : prev;
      });

      setImageSrc(optimalImage.url);
    };

    updateImage();
    window.addEventListener('resize', updateImage);
    return () => window.removeEventListener('resize', updateImage);
  }, [imageSizes]);

  return imageSrc;
}

// Usage
function SmartImage() {
  const imageSrc = useDeviceAwareImage([
    { width: 320, url: '/images/photo-320w.jpg' },
    { width: 640, url: '/images/photo-640w.jpg' },
    { width: 1280, url: '/images/photo-1280w.jpg' },
    { width: 1920, url: '/images/photo-1920w.jpg' }
  ]);

  return <img src={imageSrc} alt="Smart responsive image" loading="lazy" />;
}
```

**Angular Implementation:**

```typescript
// responsive-image.component.ts
import { Component, Input, OnInit } from '@angular/core';

interface ImageSize {
  width: number;
  format: string;
}

@Component({
  selector: 'app-responsive-image',
  template: `
    <picture>
      <!-- AVIF sources -->
      <source
        *ngIf="supportsAvif"
        [srcset]="generateSrcSet('avif')"
        [sizes]="sizes"
        type="image/avif"
      />
      
      <!-- WebP sources -->
      <source
        [srcset]="generateSrcSet('webp')"
        [sizes]="sizes"
        type="image/webp"
      />
      
      <!-- JPEG fallback -->
      <img
        [src]="fallbackSrc"
        [srcset]="generateSrcSet('jpg')"
        [sizes]="sizes"
        [alt]="alt"
        [width]="width"
        [height]="height"
        loading="lazy"
        class="responsive-image"
      />
    </picture>
  `,
  styles: [`
    .responsive-image {
      max-width: 100%;
      height: auto;
      display: block;
    }
  `]
})
export class ResponsiveImageComponent implements OnInit {
  @Input() basePath!: string;
  @Input() alt!: string;
  @Input() width!: number;
  @Input() height!: number;
  @Input() sizes = '100vw';

  supportsAvif = false;
  fallbackSrc = '';
  imageSizes = [320, 640, 960, 1280, 1920];

  ngOnInit(): void {
    this.checkAvifSupport();
    this.fallbackSrc = `${this.basePath}.jpg`;
  }

  generateSrcSet(format: string): string {
    return this.imageSizes
      .map(width => `${this.basePath}-${width}w.${format} ${width}w`)
      .join(', ');
  }

  async checkAvifSupport(): Promise<void> {
    const avif = 'data:image/avif;base64,AAAAIGZ0eXBhdmlmAAAAAGF2aWZtaWYxbWlhZk1BMUIAAADybWV0YQAAAAAAAAAoaGRscgAAAAAAAAAAcGljdAAAAAAAAAAAAAAAAGxpYmF2aWYAAAAADnBpdG0AAAAAAAEAAAAeaWxvYwAAAABEAAABAAEAAAABAAABGgAAAB0AAAAoaWluZgAAAAAAAQAAABppbmZlAgAAAAABAABhdjAxQ29sb3IAAAAAamlwcnAAAABLaXBjbwAAABRpc3BlAAAAAAAAAAEAAAABAAAAEHBpeGkAAAAAAwgICAAAAAxhdjFDgQ0MAAAAABNjb2xybmNseAACAAIAAYAAAAAXaXBtYQAAAAAAAAABAAEEAQKDBAAAACVtZGF0EgAKCBgANogQEAwgMg8f8D///8WfhwB8+ErK42A=';
    
    const img = new Image();
    img.src = avif;
    
    try {
      await img.decode();
      this.supportsAvif = true;
    } catch {
      this.supportsAvif = false;
    }
  }
}

// image-optimizer.service.ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class ImageOptimizerService {
  private imageCache = new Map<string, HTMLImageElement>();

  preloadImage(src: string): Promise<void> {
    if (this.imageCache.has(src)) {
      return Promise.resolve();
    }

    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => {
        this.imageCache.set(src, img);
        resolve();
      };
      img.onerror = reject;
      img.src = src;
    });
  }

  preloadImages(urls: string[]): Promise<void[]> {
    return Promise.all(urls.map(url => this.preloadImage(url)));
  }

  getCachedImage(src: string): HTMLImageElement | undefined {
    return this.imageCache.get(src);
  }

  clearCache(): void {
    this.imageCache.clear();
  }
}
```

## Build-Time Image Optimization

```javascript
// imageOptimizer.js - Node.js script for batch optimization
const sharp = require('sharp');
const fs = require('fs').promises;
const path = require('path');

const SIZES = [320, 640, 960, 1280, 1920];
const QUALITY = {
  jpeg: 85,
  webp: 85,
  avif: 75
};

async function optimizeImage(inputPath, outputDir) {
  const filename = path.parse(inputPath).name;
  const ext = path.parse(inputPath).ext;

  console.log(`Optimizing ${filename}${ext}...`);

  // Create output directory
  await fs.mkdir(outputDir, { recursive: true });

  const optimizations = [];

  for (const width of SIZES) {
    const image = sharp(inputPath).resize(width, null, {
      withoutEnlargement: true,
      fit: 'inside'
    });

    // Generate JPEG
    optimizations.push(
      image
        .clone()
        .jpeg({ quality: QUALITY.jpeg, progressive: true, mozjpeg: true })
        .toFile(path.join(outputDir, `${filename}-${width}w.jpg`))
    );

    // Generate WebP
    optimizations.push(
      image
        .clone()
        .webp({ quality: QUALITY.webp, effort: 6 })
        .toFile(path.join(outputDir, `${filename}-${width}w.webp`))
    );

    // Generate AVIF
    optimizations.push(
      image
        .clone()
        .avif({ quality: QUALITY.avif, effort: 4 })
        .toFile(path.join(outputDir, `${filename}-${width}w.avif`))
    );
  }

  // Generate LQIP (Low Quality Image Placeholder)
  const lqipBuffer = await sharp(inputPath)
    .resize(20)
    .jpeg({ quality: 50 })
    .toBuffer();

  const lqipBase64 = `data:image/jpeg;base64,${lqipBuffer.toString('base64')}`;
  
  optimizations.push(
    fs.writeFile(
      path.join(outputDir, `${filename}-lqip.txt`),
      lqipBase64
    )
  );

  // Generate metadata
  const metadata = await sharp(inputPath).metadata();
  const stats = await fs.stat(inputPath);
  
  optimizations.push(
    fs.writeFile(
      path.join(outputDir, `${filename}-meta.json`),
      JSON.stringify({
        originalSize: stats.size,
        width: metadata.width,
        height: metadata.height,
        format: metadata.format,
        generated: new Date().toISOString()
      }, null, 2)
    )
  );

  await Promise.all(optimizations);
  console.log(`✓ Optimized ${filename}`);
}

async function optimizeDirectory(inputDir, outputDir) {
  const files = await fs.readdir(inputDir);
  const imageFiles = files.filter(file =>
    /\.(jpe?g|png|webp)$/i.test(file)
  );

  console.log(`Found ${imageFiles.length} images to optimize\n`);

  for (const file of imageFiles) {
    try {
      await optimizeImage(
        path.join(inputDir, file),
        outputDir
      );
    } catch (error) {
      console.error(`Failed to optimize ${file}:`, error.message);
    }
  }

  console.log(`\n✓ All images optimized!`);
}

// CLI usage
const inputDir = process.argv[2] || './images/source';
const outputDir = process.argv[3] || './public/images/optimized';

optimizeDirectory(inputDir, outputDir)
  .catch(console.error);

// Webpack configuration for automatic optimization
// webpack.config.js
const ImageMinimizerPlugin = require('image-minimizer-webpack-plugin');

module.exports = {
  module: {
    rules: [
      {
        test: /\.(jpe?g|png|gif|svg)$/i,
        type: 'asset'
      }
    ]
  },
  plugins: [
    new ImageMinimizerPlugin({
      minimizer: {
        implementation: ImageMinimizerPlugin.sharpMinify,
        options: {
          encodeOptions: {
            jpeg: { quality: 85, progressive: true },
            webp: { quality: 85 },
            avif: { quality: 75 },
            png: { compressionLevel: 9 }
          }
        }
      },
      generator: [
        {
          preset: 'webp',
          implementation: ImageMinimizerPlugin.sharpGenerate,
          options: {
            encodeOptions: {
              webp: { quality: 85 }
            }
          }
        },
        {
          preset: 'avif',
          implementation: ImageMinimizerPlugin.sharpGenerate,
          options: {
            encodeOptions: {
              avif: { quality: 75 }
            }
          }
        }
      ]
    })
  ]
};
```

## Progressive Image Loading

```typescript
// React: Progressive image with blur-up effect
import { useState, useEffect, useRef } from 'react';

interface ProgressiveImageProps {
  src: string;
  placeholder: string;
  alt: string;
  className?: string;
}

function ProgressiveImage({
  src,
  placeholder,
  alt,
  className
}: ProgressiveImageProps) {
  const [currentSrc, setCurrentSrc] = useState(placeholder);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(false);

  useEffect(() => {
    const img = new Image();
    img.src = src;
    
    img.onload = () => {
      setCurrentSrc(src);
      setLoading(false);
    };
    
    img.onerror = () => {
      setError(true);
      setLoading(false);
    };

    return () => {
      img.onload = null;
      img.onerror = null;
    };
  }, [src]);

  if (error) {
    return (
      <div className="image-error">
        <span>Failed to load image</span>
      </div>
    );
  }

  return (
    <div className={`progressive-image ${className || ''}`}>
      <img
        src={currentSrc}
        alt={alt}
        className={loading ? 'loading' : 'loaded'}
        style={{
          filter: loading ? 'blur(20px)' : 'none',
          transform: loading ? 'scale(1.05)' : 'scale(1)',
          transition: 'filter 0.4s ease-out, transform 0.4s ease-out'
        }}
      />
      {loading && (
        <div className="loading-indicator">
          <div className="spinner" />
        </div>
      )}
    </div>
  );
}

// Intersection Observer + Progressive Loading
function LazyProgressiveImage({ src, placeholder, alt }: any) {
  const [isVisible, setIsVisible] = useState(false);
  const [imageSrc, setImageSrc] = useState(placeholder);
  const [loaded, setLoaded] = useState(false);
  const imgRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { rootMargin: '100px' }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  useEffect(() => {
    if (isVisible && !loaded) {
      const img = new Image();
      img.src = src;
      img.onload = () => {
        setImageSrc(src);
        setLoaded(true);
      };
    }
  }, [isVisible, src, loaded]);

  return (
    <div ref={imgRef} className="lazy-progressive-image">
      <img
        src={imageSrc}
        alt={alt}
        style={{
          filter: loaded ? 'none' : 'blur(20px)',
          opacity: loaded ? 1 : 0.8,
          transition: 'filter 0.3s, opacity 0.3s'
        }}
      />
    </div>
  );
}

// Advanced: Dominant color placeholder
function DominantColorImage({ src, dominantColor, alt }: any) {
  const [loaded, setLoaded] = useState(false);

  return (
    <div
      style={{
        backgroundColor: dominantColor,
        position: 'relative',
        overflow: 'hidden'
      }}
    >
      <img
        src={src}
        alt={alt}
        onLoad={() => setLoaded(true)}
        style={{
          opacity: loaded ? 1 : 0,
          transition: 'opacity 0.3s'
        }}
      />
    </div>
  );
}
```

## CDN Integration

```typescript
// Cloudinary helper
class CloudinaryImage {
  private cloudName: string;
  private baseUrl: string;

  constructor(cloudName: string) {
    this.cloudName = cloudName;
    this.baseUrl = `https://res.cloudinary.com/${cloudName}/image/upload`;
  }

  getUrl(publicId: string, transformations: any = {}): string {
    const transforms = [
      transformations.width && `w_${transformations.width}`,
      transformations.height && `h_${transformations.height}`,
      transformations.crop && `c_${transformations.crop}`,
      transformations.quality && `q_${transformations.quality}`,
      'f_auto', // Auto format
      'dpr_auto' // Auto DPR
    ].filter(Boolean).join(',');

    return `${this.baseUrl}/${transforms}/${publicId}`;
  }

  getResponsiveSrcSet(publicId: string, widths: number[]): string {
    return widths
      .map(width => `${this.getUrl(publicId, { width })} ${width}w`)
      .join(', ');
  }
}

// React component
function CloudinaryResponsiveImage({ publicId, alt }: any) {
  const cloudinary = new CloudinaryImage('your-cloud-name');
  const widths = [320, 640, 960, 1280, 1920];

  return (
    <img
      src={cloudinary.getUrl(publicId, { width: 1280, quality: 'auto' })}
      srcSet={cloudinary.getResponsiveSrcSet(publicId, widths)}
      sizes="(max-width: 640px) 100vw, (max-width: 1280px) 50vw, 33vw"
      alt={alt}
      loading="lazy"
    />
  );
}

// Imgix helper
class ImgixImage {
  private domain: string;

  constructor(domain: string) {
    this.domain = domain;
  }

  getUrl(path: string, params: any = {}): string {
    const queryParams = new URLSearchParams({
      auto: 'format,compress',
      ...params
    });

    return `https://${this.domain}/${path}?${queryParams}`;
  }

  getSrcSet(path: string, params: any = {}): string {
    const dprs = [1, 2, 3];
    return dprs
      .map(dpr => `${this.getUrl(path, { ...params, dpr })} ${dpr}x`)
      .join(', ');
  }
}
```

## Image Optimization Workflow

```
Image Processing Pipeline
==========================

    Original (3.2MB)
          │
          ▼
    ┌─────────────┐
    │  1. Resize  │  Generate multiple sizes
    │             │  320w, 640w, 960w, 1280w, 1920w
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  2. Format  │  Convert to modern formats
    │             │  JPEG, WebP, AVIF
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  3. Compress│  Optimize quality
    │             │  JPEG: 85%, WebP: 85%, AVIF: 75%
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  4. LQIP    │  Generate placeholder
    │             │  20px blur-up preview
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  5. CDN     │  Upload to CDN
    │             │  Global distribution
    └──────┬──────┘
           │
           ▼
    Optimized Images
    └─ 15 files (5 sizes × 3 formats)
    └─ Average: 180KB (94% reduction)
    └─ Formats: AVIF, WebP, JPEG
    └─ Responsive delivery
```

## Common Mistakes

1. **Not specifying dimensions**
   - Causes layout shift (CLS)
   - Always set width/height

2. **Using PNG for photos**
   - 2-3x larger than JPEG
   - Use JPEG/WebP for photos

3. **Serving same image to all devices**
   - Wastes bandwidth
   - Use responsive images

4. **Not compressing before upload**
   - Unnecessary file sizes
   - Compress during build

5. **Ignoring modern formats**
   - Missing 30-50% savings
   - Support WebP/AVIF

6. **No lazy loading**
   - Slows initial load
   - Lazy load below fold

7. **No progressive loading**
   - Poor perceived performance
   - Use blur-up technique

8. **Using images for text**
   - Not accessible
   - Use CSS/SVG instead

## Best Practices

1. **Use modern formats**: WebP (30% smaller), AVIF (50% smaller)
2. **Implement responsive images**: srcset and sizes attributes
3. **Compress aggressively**: 85% JPEG quality is usually fine
4. **Lazy load**: Load images as they enter viewport
5. **Specify dimensions**: Prevent layout shifts (CLS)
6. **Use CDN**: Global distribution, automatic optimization
7. **Progressive loading**: Blur-up for better perceived performance
8. **Automate optimization**: Use build tools (Sharp, ImageMagick)
9. **Monitor performance**: Track image metrics
10. **Test on real devices**: Verify on slow networks

## When to Use

**Optimize images for:**
- All production websites
- Mobile-first applications
- E-commerce product pages
- Image galleries/portfolios
- Blog post images
- Landing pages

**Less critical for:**
- Internal tools
- Development environments
- Admin dashboards
- Placeholder images

## Interview Questions

1. **What are the benefits of WebP over JPEG?**
   - 25-35% smaller file size
   - Supports transparency (unlike JPEG)
   - Supports animation
   - Both lossy and lossless compression

2. **Explain srcset and sizes attributes.**
   - srcset: List of image sources at different widths
   - sizes: Media queries defining image display size
   - Browser chooses optimal image based on viewport
   - Saves bandwidth on smaller screens

3. **What is LQIP and why use it?**
   - Low Quality Image Placeholder
   - Small blurred preview (20px)
   - Shows immediately while full image loads
   - Improves perceived performance
   - Prevents empty space/layout shifts

4. **How does lazy loading improve performance?**
   - Reduces initial page weight
   - Loads images only when needed
   - Improves LCP and FCP metrics
   - Saves bandwidth for unseen images

5. **When would you use the picture element vs srcset?**
   - picture: Art direction (different crops/images per breakpoint)
   - srcset: Same image, different sizes
   - picture: Multiple format fallbacks
   - srcset: Simpler, automatic selection

6. **How do you prevent CLS from images?**
   - Always specify width/height attributes
   - Use aspect-ratio CSS property
   - Reserve space with padding-top hack
   - Provide placeholder with dimensions

7. **What's the trade-off between image quality and file size?**
   - 85% JPEG quality vs 100%: 50% size reduction, imperceptible quality loss
   - Diminishing returns above 85%
   - Test with target audience
   - Mobile users tolerate lower quality for speed

8. **How would you implement image optimization in CI/CD?**
   - Automated compression with Sharp/ImageMagick
   - Generate multiple sizes and formats
   - Upload to CDN
   - Set size budgets
   - Fail build if images exceed limits

## Key Takeaways

1. Images account for 50-70% of page weight - biggest optimization opportunity
2. WebP provides 25-35% savings, AVIF provides 50% savings vs JPEG
3. Use srcset and sizes for responsive images across devices
4. Always specify image dimensions to prevent layout shifts
5. Lazy load images below the fold for faster initial page load
6. Implement progressive loading with blur-up for better UX
7. Compress images at 85% quality for optimal size/quality balance
8. Use CDN for automatic format selection and global distribution
9. Generate multiple sizes (320w, 640w, 960w, 1280w, 1920w)
10. Automate optimization in build pipeline for consistency

## Resources

- **Official Documentation**
  - [Responsive Images](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images)
  - [WebP Format](https://developers.google.com/speed/webp)
  - [AVIF Format](https://avif.io/)
  - [Picture Element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/picture)

- **Tools**
  - [Sharp](https://sharp.pixelplumbing.com/) - Node.js image processing
  - [Squoosh](https://squoosh.app/) - Online optimizer
  - [ImageOptim](https://imageoptim.com/) - Mac app
  - [TinyPNG](https://tinypng.com/) - Online compression
  - [Cloudinary](https://cloudinary.com/) - Image CDN
  - [Imgix](https://imgix.com/) - Image optimization service

- **Articles**
  - [Optimize Images - web.dev](https://web.dev/fast/#optimize-your-images)
  - [Serve Responsive Images](https://web.dev/serve-responsive-images/)
  - [Use WebP Images](https://web.dev/serve-images-webp/)
  - [Image Optimization](https://web.dev/image-optimization/)
