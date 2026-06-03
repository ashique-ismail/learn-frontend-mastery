# Font Optimization

## Overview

Font optimization is critical for web performance, as web fonts can significantly impact page load time and rendering. Unoptimized fonts can cause Flash of Invisible Text (FOIT), Flash of Unstyled Text (FOUT), or layout shifts, degrading user experience. This guide covers modern font loading strategies, font-display values, subsetting, variable fonts, and preloading techniques.

## Font Loading Phases

Browsers go through distinct phases when loading web fonts:

```
Timeline:
┌─────────────────────────────────────────────────────────────┐
│  Block Period  │  Swap Period  │   Failure Period           │
│   (0-3s)       │   (3s-∞)      │   (fallback forever)       │
├────────────────┼───────────────┼────────────────────────────┤
│  FOIT          │  Show fallback│   Use fallback font        │
│  (invisible)   │  then swap    │   permanently              │
└─────────────────────────────────────────────────────────────┘
```

**Three main behaviors:**

1. **FOIT (Flash of Invisible Text)**: Text is invisible during block period
2. **FOUT (Flash of Unstyled Text)**: Text shows immediately with fallback, then swaps
3. **FOFT (Flash of Faux Text)**: Progressive enhancement with staged font loading

## font-display Property

The `font-display` CSS descriptor controls font loading behavior:

### font-display Values

```css
/* 1. auto - Browser default (usually similar to block) */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: auto;
}

/* 2. block - Short block period (3s), infinite swap */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: block; /* Good for icons/critical branding */
}

/* 3. swap - No block period, immediate fallback, infinite swap */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* RECOMMENDED for body text */
}

/* 4. fallback - Short block (100ms), short swap (3s) */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: fallback; /* Balance performance/aesthetics */
}

/* 5. optional - Short block (100ms), no swap period */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: optional; /* Best for performance, user-first */
}
```

### Decision Matrix

```
┌──────────────┬────────────┬────────────┬─────────────────────┐
│ font-display │ Block Time │ Swap Time  │ Use Case            │
├──────────────┼────────────┼────────────┼─────────────────────┤
│ auto         │ 3s         │ Infinite   │ Default             │
│ block        │ 3s         │ Infinite   │ Icons, branding     │
│ swap         │ 0ms        │ Infinite   │ Body text (COMMON)  │
│ fallback     │ 100ms      │ 3s         │ Balanced approach   │
│ optional     │ 100ms      │ 0s         │ Performance-first   │
└──────────────┴────────────┴────────────┴─────────────────────┘
```

## Font Subsetting

Subsetting reduces font file size by including only required glyphs:

### Manual Subsetting with pyftsubset

```bash
# Install fonttools
pip install fonttools brotli

# Subset font to Latin characters only
pyftsubset CustomFont.ttf \
  --output-file=CustomFont-latin.woff2 \
  --flavor=woff2 \
  --layout-features=* \
  --unicodes=U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD

# Subset to specific characters
pyftsubset CustomFont.ttf \
  --output-file=CustomFont-numbers.woff2 \
  --flavor=woff2 \
  --text="0123456789"
```

### Unicode Ranges in CSS

```css
/* Latin subset */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom-latin.woff2') format('woff2');
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+02C6, U+02DA, U+02DC, U+2000-206F, U+2074, U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

/* Cyrillic subset */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom-cyrillic.woff2') format('woff2');
  font-display: swap;
  unicode-range: U+0400-045F, U+0490-0491, U+04B0-04B1, U+2116;
}

/* Greek subset */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom-greek.woff2') format('woff2');
  font-display: swap;
  unicode-range: U+0370-03FF;
}
```

### React Font Subsetting Component

```typescript
// FontProvider.tsx
import React from 'react';

interface FontConfig {
  family: string;
  weights: number[];
  display: 'auto' | 'block' | 'swap' | 'fallback' | 'optional';
  subsets: string[];
}

export const FontProvider: React.FC<{ config: FontConfig }> = ({ config, children }) => {
  React.useEffect(() => {
    const style = document.createElement('style');
    
    const fontFaces = config.subsets.flatMap(subset => 
      config.weights.map(weight => {
        const unicodeRange = getUnicodeRange(subset);
        return `
          @font-face {
            font-family: '${config.family}';
            font-weight: ${weight};
            font-display: ${config.display};
            src: url('/fonts/${config.family}-${subset}-${weight}.woff2') format('woff2');
            unicode-range: ${unicodeRange};
          }
        `;
      })
    ).join('\n');
    
    style.textContent = fontFaces;
    document.head.appendChild(style);
    
    return () => {
      document.head.removeChild(style);
    };
  }, [config]);
  
  return <>{children}</>;
};

function getUnicodeRange(subset: string): string {
  const ranges: Record<string, string> = {
    latin: 'U+0000-00FF, U+0131, U+0152-0153',
    'latin-ext': 'U+0100-024F, U+0259, U+1E00-1EFF, U+2020, U+20A0-20AB',
    cyrillic: 'U+0400-045F, U+0490-0491, U+04B0-04B1, U+2116',
    greek: 'U+0370-03FF',
    vietnamese: 'U+0102-0103, U+0110-0111, U+0128-0129, U+0168-0169',
  };
  return ranges[subset] || ranges.latin;
}

// Usage
const fontConfig: FontConfig = {
  family: 'Inter',
  weights: [400, 600, 700],
  display: 'swap',
  subsets: ['latin', 'latin-ext']
};

function App() {
  return (
    <FontProvider config={fontConfig}>
      <div>Your app content</div>
    </FontProvider>
  );
}
```

## Variable Fonts

Variable fonts contain multiple font variations in a single file:

### Variable Font Declaration

```css
/* Traditional approach: Multiple files */
@font-face {
  font-family: 'Inter';
  font-weight: 400;
  src: url('/fonts/inter-400.woff2') format('woff2');
}
@font-face {
  font-family: 'Inter';
  font-weight: 600;
  src: url('/fonts/inter-600.woff2') format('woff2');
}
@font-face {
  font-family: 'Inter';
  font-weight: 700;
  src: url('/fonts/inter-700.woff2') format('woff2');
}

/* Variable font: Single file */
@font-face {
  font-family: 'InterVariable';
  font-weight: 100 900; /* Weight range */
  font-display: swap;
  src: url('/fonts/inter-variable.woff2') format('woff2-variations');
}

/* Use any weight in the range */
h1 {
  font-family: 'InterVariable', sans-serif;
  font-weight: 650; /* Any value between 100-900 */
}

h2 {
  font-family: 'InterVariable', sans-serif;
  font-weight: 550;
}
```

### Variable Font Axes

```css
/* Multi-axis variable font */
@font-face {
  font-family: 'Recursive';
  src: url('/fonts/recursive-variable.woff2') format('woff2-variations');
  font-weight: 300 1000;
  font-display: swap;
}

/* Standard axes (predefined) */
.standard-axes {
  font-family: 'Recursive', sans-serif;
  font-weight: 600;           /* wght axis */
  font-style: oblique 10deg;  /* slnt axis */
  font-stretch: 125%;         /* wdth axis */
  font-optical-sizing: auto;  /* opsz axis */
}

/* Custom axes (font-specific) */
.custom-axes {
  font-family: 'Recursive', sans-serif;
  font-variation-settings: 
    'MONO' 1,      /* Monospace: 0 (sans) to 1 (mono) */
    'CASL' 0.5,    /* Casual: 0 (linear) to 1 (casual) */
    'wght' 700,    /* Weight */
    'slnt' -15;    /* Slant */
}

/* Animation with variable fonts */
@keyframes weight-pulse {
  0%, 100% { font-variation-settings: 'wght' 400; }
  50% { font-variation-settings: 'wght' 700; }
}

.animated-text {
  animation: weight-pulse 2s ease-in-out infinite;
}
```

### React Variable Font Component

```typescript
// VariableText.tsx
import React, { CSSProperties } from 'react';

interface VariableTextProps {
  children: React.ReactNode;
  weight?: number;
  width?: number;
  slant?: number;
  customAxes?: Record<string, number>;
  className?: string;
}

export const VariableText: React.FC<VariableTextProps> = ({
  children,
  weight = 400,
  width = 100,
  slant = 0,
  customAxes = {},
  className
}) => {
  const style: CSSProperties = {
    fontFamily: 'InterVariable, sans-serif',
    fontWeight: weight,
    fontStretch: `${width}%`,
    fontStyle: slant !== 0 ? `oblique ${slant}deg` : 'normal',
    fontVariationSettings: Object.entries(customAxes)
      .map(([axis, value]) => `'${axis}' ${value}`)
      .join(', ')
  };
  
  return (
    <span className={className} style={style}>
      {children}
    </span>
  );
};

// Usage
function Example() {
  return (
    <>
      <VariableText weight={300}>Light text</VariableText>
      <VariableText weight={700}>Bold text</VariableText>
      <VariableText weight={650} width={110}>
        Custom weight and width
      </VariableText>
    </>
  );
}
```

## Font Preloading

Preloading fonts reduces FOIT/FOUT by starting font downloads earlier:

### Basic Preload

```html
<!-- Preload critical fonts -->
<link
  rel="preload"
  href="/fonts/inter-latin-400.woff2"
  as="font"
  type="font/woff2"
  crossorigin="anonymous"
>

<!-- Preload multiple weights -->
<link rel="preload" href="/fonts/inter-400.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/fonts/inter-700.woff2" as="font" type="font/woff2" crossorigin>
```

### React Preload Component

```typescript
// FontPreload.tsx
import React from 'react';
import { Helmet } from 'react-helmet-async';

interface FontPreloadProps {
  fonts: Array<{
    href: string;
    type?: string;
  }>;
}

export const FontPreload: React.FC<FontPreloadProps> = ({ fonts }) => {
  return (
    <Helmet>
      {fonts.map((font, index) => (
        <link
          key={index}
          rel="preload"
          href={font.href}
          as="font"
          type={font.type || 'font/woff2'}
          crossOrigin="anonymous"
        />
      ))}
    </Helmet>
  );
};

// Usage
function App() {
  const criticalFonts = [
    { href: '/fonts/inter-latin-400.woff2' },
    { href: '/fonts/inter-latin-700.woff2' }
  ];
  
  return (
    <>
      <FontPreload fonts={criticalFonts} />
      <div>App content</div>
    </>
  );
}
```

### Angular Font Preload Service

```typescript
// font-preload.service.ts
import { Injectable, Inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';

interface FontDescriptor {
  href: string;
  type?: string;
}

@Injectable({
  providedIn: 'root'
})
export class FontPreloadService {
  private preloadedFonts = new Set<string>();
  
  constructor(@Inject(DOCUMENT) private document: Document) {}
  
  preloadFonts(fonts: FontDescriptor[]): void {
    fonts.forEach(font => {
      if (!this.preloadedFonts.has(font.href)) {
        this.createPreloadLink(font);
        this.preloadedFonts.add(font.href);
      }
    });
  }
  
  private createPreloadLink(font: FontDescriptor): void {
    const link = this.document.createElement('link');
    link.rel = 'preload';
    link.href = font.href;
    link.as = 'font';
    link.type = font.type || 'font/woff2';
    link.crossOrigin = 'anonymous';
    
    this.document.head.appendChild(link);
  }
  
  preloadFont(href: string, type = 'font/woff2'): void {
    this.preloadFonts([{ href, type }]);
  }
}

// app.component.ts
import { Component, OnInit } from '@angular/core';
import { FontPreloadService } from './services/font-preload.service';

@Component({
  selector: 'app-root',
  template: '<router-outlet></router-outlet>'
})
export class AppComponent implements OnInit {
  constructor(private fontPreload: FontPreloadService) {}
  
  ngOnInit(): void {
    this.fontPreload.preloadFonts([
      { href: '/fonts/inter-400.woff2' },
      { href: '/fonts/inter-700.woff2' }
    ]);
  }
}
```

## Font Loading API

The CSS Font Loading API provides programmatic control over font loading:

### Basic Font Loading

```typescript
// Check if font is loaded
if (document.fonts.check('1rem Inter')) {
  console.log('Inter font is loaded');
}

// Load font programmatically
const font = new FontFace(
  'CustomFont',
  'url(/fonts/custom.woff2)',
  {
    weight: '400',
    style: 'normal',
    display: 'swap'
  }
);

font.load().then(loadedFont => {
  document.fonts.add(loadedFont);
  console.log('Font loaded successfully');
});

// Wait for multiple fonts
Promise.all([
  new FontFace('Inter', 'url(/fonts/inter-400.woff2)').load(),
  new FontFace('Inter', 'url(/fonts/inter-700.woff2)', { weight: '700' }).load()
]).then(fonts => {
  fonts.forEach(font => document.fonts.add(font));
  document.body.classList.add('fonts-loaded');
});
```

### React Font Loading Hook

```typescript
// useFontLoader.ts
import { useEffect, useState } from 'react';

interface FontDescriptor {
  family: string;
  source: string;
  descriptors?: FontFaceDescriptors;
}

export function useFontLoader(fonts: FontDescriptor[]) {
  const [loaded, setLoaded] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const loadFonts = async () => {
      try {
        const fontFaces = fonts.map(
          font => new FontFace(font.family, font.source, font.descriptors)
        );
        
        const loadedFonts = await Promise.all(
          fontFaces.map(font => font.load())
        );
        
        loadedFonts.forEach(font => {
          document.fonts.add(font);
        });
        
        setLoaded(true);
      } catch (err) {
        setError(err as Error);
      }
    };
    
    loadFonts();
  }, [fonts]);
  
  return { loaded, error };
}

// Usage
function App() {
  const { loaded, error } = useFontLoader([
    {
      family: 'Inter',
      source: 'url(/fonts/inter-400.woff2)',
      descriptors: { weight: '400', display: 'swap' }
    },
    {
      family: 'Inter',
      source: 'url(/fonts/inter-700.woff2)',
      descriptors: { weight: '700', display: 'swap' }
    }
  ]);
  
  if (error) return <div>Font loading error: {error.message}</div>;
  
  return (
    <div className={loaded ? 'fonts-loaded' : 'fonts-loading'}>
      <h1>Content</h1>
    </div>
  );
}
```

### Angular Font Loading Directive

```typescript
// font-loader.directive.ts
import { Directive, Input, OnInit, ElementRef, Renderer2 } from '@angular/core';

interface FontConfig {
  family: string;
  source: string;
  weight?: string;
  style?: string;
}

@Directive({
  selector: '[appFontLoader]'
})
export class FontLoaderDirective implements OnInit {
  @Input() appFontLoader: FontConfig[] = [];
  @Input() loadingClass = 'fonts-loading';
  @Input() loadedClass = 'fonts-loaded';
  
  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}
  
  ngOnInit(): void {
    this.renderer.addClass(this.el.nativeElement, this.loadingClass);
    this.loadFonts();
  }
  
  private async loadFonts(): Promise<void> {
    try {
      const fontPromises = this.appFontLoader.map(config => {
        const font = new FontFace(config.family, config.source, {
          weight: config.weight || 'normal',
          style: config.style || 'normal',
          display: 'swap'
        });
        return font.load();
      });
      
      const loadedFonts = await Promise.all(fontPromises);
      loadedFonts.forEach(font => document.fonts.add(font));
      
      this.renderer.removeClass(this.el.nativeElement, this.loadingClass);
      this.renderer.addClass(this.el.nativeElement, this.loadedClass);
    } catch (error) {
      console.error('Font loading failed:', error);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  template: `
    <div [appFontLoader]="fonts">
      <h1>Content</h1>
    </div>
  `
})
export class AppComponent {
  fonts = [
    { family: 'Inter', source: 'url(/fonts/inter-400.woff2)', weight: '400' },
    { family: 'Inter', source: 'url(/fonts/inter-700.woff2)', weight: '700' }
  ];
}
```

## FOFT Strategy (Progressive Font Loading)

Flash of Faux Text loads fonts in stages for optimal performance:

```typescript
// FOFT implementation
class FOFTLoader {
  async loadFonts() {
    // Stage 1: Load Roman (regular) weight first
    const roman = new FontFace(
      'CustomFont',
      'url(/fonts/custom-400.woff2)',
      { weight: '400', style: 'normal', display: 'swap' }
    );
    
    await roman.load();
    document.fonts.add(roman);
    document.documentElement.classList.add('fonts-stage-1');
    
    // Stage 2: Load other weights/styles
    const boldItalic = await Promise.all([
      new FontFace('CustomFont', 'url(/fonts/custom-700.woff2)', {
        weight: '700', display: 'swap'
      }).load(),
      new FontFace('CustomFont', 'url(/fonts/custom-400i.woff2)', {
        weight: '400', style: 'italic', display: 'swap'
      }).load()
    ]);
    
    boldItalic.forEach(font => document.fonts.add(font));
    document.documentElement.classList.add('fonts-stage-2');
  }
}

// CSS for FOFT
const styles = `
  /* Fallback */
  body {
    font-family: Arial, sans-serif;
  }
  
  /* Stage 1: Roman loaded, use for everything */
  .fonts-stage-1 body {
    font-family: 'CustomFont', Arial, sans-serif;
  }
  
  /* Stage 1: Synthesize bold/italic */
  .fonts-stage-1 strong,
  .fonts-stage-1 b {
    font-weight: 600; /* Slightly synthesized */
  }
  
  .fonts-stage-1 em,
  .fonts-stage-1 i {
    font-style: italic; /* Browser synthesizes */
  }
  
  /* Stage 2: True fonts loaded */
  .fonts-stage-2 strong,
  .fonts-stage-2 b {
    font-weight: 700; /* True bold */
  }
  
  .fonts-stage-2 em,
  .fonts-stage-2 i {
    font-style: italic; /* True italic */
  }
`;
```

### React FOFT Component

```typescript
// FOFTProvider.tsx
import React, { useEffect, useState } from 'react';

interface FOFTStage {
  fonts: Array<{
    family: string;
    source: string;
    descriptors?: FontFaceDescriptors;
  }>;
}

interface FOFTConfig {
  stages: FOFTStage[];
}

export const FOFTProvider: React.FC<{
  config: FOFTConfig;
  children: React.ReactNode;
}> = ({ config, children }) => {
  const [stage, setStage] = useState(0);
  
  useEffect(() => {
    const loadStage = async (stageIndex: number) => {
      if (stageIndex >= config.stages.length) return;
      
      const fonts = config.stages[stageIndex].fonts.map(
        font => new FontFace(font.family, font.source, font.descriptors)
      );
      
      const loaded = await Promise.all(fonts.map(f => f.load()));
      loaded.forEach(font => document.fonts.add(font));
      
      document.documentElement.classList.add(`fonts-stage-${stageIndex + 1}`);
      setStage(stageIndex + 1);
      
      // Load next stage
      loadStage(stageIndex + 1);
    };
    
    loadStage(0);
  }, [config]);
  
  return <>{children}</>;
};

// Usage
const foftConfig: FOFTConfig = {
  stages: [
    {
      // Stage 1: Critical roman font
      fonts: [
        {
          family: 'Inter',
          source: 'url(/fonts/inter-400.woff2)',
          descriptors: { weight: '400', display: 'swap' }
        }
      ]
    },
    {
      // Stage 2: Other weights
      fonts: [
        {
          family: 'Inter',
          source: 'url(/fonts/inter-700.woff2)',
          descriptors: { weight: '700', display: 'swap' }
        },
        {
          family: 'Inter',
          source: 'url(/fonts/inter-400i.woff2)',
          descriptors: { weight: '400', style: 'italic', display: 'swap' }
        }
      ]
    }
  ]
};

function App() {
  return (
    <FOFTProvider config={foftConfig}>
      <div>Your content</div>
    </FOFTProvider>
  );
}
```

## Self-Hosted vs CDN Fonts

### Self-Hosted Fonts

```typescript
// Self-hosted font setup
const selfHostedFonts = `
  @font-face {
    font-family: 'Inter';
    src: url('/fonts/inter-400.woff2') format('woff2');
    font-weight: 400;
    font-display: swap;
  }
  
  @font-face {
    font-family: 'Inter';
    src: url('/fonts/inter-700.woff2') format('woff2');
    font-weight: 700;
    font-display: swap;
  }
`;

// Pros:
// - Full control over caching
// - No external DNS lookup
// - No privacy concerns
// - Can use HTTP/2 push
// - Guaranteed availability

// Cons:
// - Manual updates needed
// - No global CDN cache
// - Bandwidth costs
```

### Google Fonts (CDN)

```html
<!-- Traditional approach -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">

<!-- Optimized approach: Self-host Google Fonts -->
```

### Download Google Fonts Script

```bash
#!/bin/bash
# download-google-fonts.sh

FONT_URL="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap"
OUTPUT_DIR="./public/fonts"

mkdir -p "$OUTPUT_DIR"

# Get CSS file
curl -s "$FONT_URL" \
  -H "User-Agent: Mozilla/5.0" \
  > "$OUTPUT_DIR/fonts.css"

# Extract and download font files
grep -oP "https://[^)]*\.woff2" "$OUTPUT_DIR/fonts.css" | while read url; do
  filename=$(basename "$url" | cut -d'?' -f1)
  curl -s "$url" -o "$OUTPUT_DIR/$filename"
  
  # Update CSS to use local files
  sed -i "s|$url|/fonts/$filename|g" "$OUTPUT_DIR/fonts.css"
done

echo "Fonts downloaded to $OUTPUT_DIR"
```

## Common Mistakes

### 1. Not Using font-display

```typescript
// ❌ Bad: No font-display (causes FOIT)
const badFontFace = `
  @font-face {
    font-family: 'CustomFont';
    src: url('/fonts/custom.woff2') format('woff2');
    /* Missing font-display! */
  }
`;

// ✅ Good: Use font-display: swap
const goodFontFace = `
  @font-face {
    font-family: 'CustomFont';
    src: url('/fonts/custom.woff2') format('woff2');
    font-display: swap;
  }
`;
```

### 2. Preloading Too Many Fonts

```html
<!-- ❌ Bad: Preloading all fonts -->
<link rel="preload" href="/fonts/inter-100.woff2" as="font" crossorigin>
<link rel="preload" href="/fonts/inter-200.woff2" as="font" crossorigin>
<link rel="preload" href="/fonts/inter-300.woff2" as="font" crossorigin>
<link rel="preload" href="/fonts/inter-400.woff2" as="font" crossorigin>
<link rel="preload" href="/fonts/inter-500.woff2" as="font" crossorigin>
<!-- etc... delays other critical resources -->

<!-- ✅ Good: Only preload critical fonts -->
<link rel="preload" href="/fonts/inter-400.woff2" as="font" crossorigin>
<link rel="preload" href="/fonts/inter-700.woff2" as="font" crossorigin>
```

### 3. Missing crossorigin Attribute

```html
<!-- ❌ Bad: Missing crossorigin -->
<link rel="preload" href="/fonts/custom.woff2" as="font">

<!-- ✅ Good: Include crossorigin -->
<link rel="preload" href="/fonts/custom.woff2" as="font" crossorigin="anonymous">
```

### 4. Not Subsetting Fonts

```typescript
// ❌ Bad: Full font with all glyphs (500KB+)
const fullFont = `
  @font-face {
    font-family: 'Roboto';
    src: url('/fonts/roboto-full.woff2') format('woff2');
  }
`;

// ✅ Good: Subset to required characters (50KB)
const subsetFont = `
  @font-face {
    font-family: 'Roboto';
    src: url('/fonts/roboto-latin.woff2') format('woff2');
    unicode-range: U+0000-00FF;
  }
`;
```

### 5. Wrong Font Format Order

```css
/* ❌ Bad: Wrong order */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.ttf') format('truetype'),
       url('/fonts/custom.woff2') format('woff2');
}

/* ✅ Good: WOFF2 first (smallest, best supported) */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2'),
       url('/fonts/custom.woff') format('woff'),
       url('/fonts/custom.ttf') format('truetype');
}
```

## Best Practices

### 1. Use font-display: swap for Body Text

```css
@font-face {
  font-family: 'BodyFont';
  src: url('/fonts/body.woff2') format('woff2');
  font-display: swap; /* Show text immediately */
}
```

### 2. Subset Fonts Aggressively

```bash
# Only include characters you need
pyftsubset font.ttf \
  --output-file=font-subset.woff2 \
  --flavor=woff2 \
  --unicodes=U+0020-007E # Basic ASCII
```

### 3. Preload Only Critical Fonts

```html
<!-- Only above-the-fold, high-priority fonts -->
<link rel="preload" href="/fonts/heading-bold.woff2" as="font" crossorigin>
```

### 4. Use Variable Fonts When Possible

```css
/* One file instead of multiple */
@font-face {
  font-family: 'InterVar';
  src: url('/fonts/inter-variable.woff2') format('woff2-variations');
  font-weight: 100 900;
}
```

### 5. Implement FOFT for Custom Fonts

Load roman weight first, then other styles progressively.

### 6. Self-Host Google Fonts

Avoid external DNS lookups and privacy concerns.

### 7. Use WOFF2 Format

WOFF2 provides best compression (30% better than WOFF).

### 8. Set Proper Cache Headers

```nginx
# nginx config
location ~* \.(woff2|woff)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

### 9. Match Fallback Font Metrics

```css
/* Minimize layout shift */
body {
  font-family: 'CustomFont', Arial, sans-serif;
  /* Adjust fallback to match custom font metrics */
  font-size-adjust: 0.5;
}
```

### 10. Monitor Font Performance

```typescript
// Track font loading time
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name.includes('.woff2')) {
      console.log(`Font loaded: ${entry.name} in ${entry.duration}ms`);
    }
  }
});

observer.observe({ entryTypes: ['resource'] });
```

## When to Use Each Strategy

### Use font-display: swap

- **Body text**: Always
- **Headings**: Usually
- **Content-first sites**: Always

### Use font-display: optional

- **Performance-critical sites**
- **Slow networks** (3G)
- **Mobile-first** apps

### Use font-display: block

- **Icon fonts** (must render correctly)
- **Brand logos**
- **Specialized symbols**

### Use Variable Fonts

- **Multiple weights needed**
- **Interactive typography**
- **Modern browser targets**

### Use Font Subsetting

- **Non-Latin content** (separate subsets)
- **Large font files** (>100KB)
- **Limited character sets** (numbers only, etc.)

### Use FOFT

- **Custom brand fonts**
- **Multiple font styles**
- **Content-rich sites**

## Interview Questions

### 1. Explain the difference between FOIT and FOUT. Which is better for UX?

**Answer**: FOIT (Flash of Invisible Text) hides text until fonts load, creating invisible text periods. FOUT (Flash of Unstyled Text) shows fallback fonts immediately, then swaps to web fonts. FOUT is generally better for UX because users can read content immediately, though it causes a visual "flash." Modern best practice is `font-display: swap` for FOUT, or `font-display: optional` for performance-critical sites.

### 2. What is font-display and what are its values?

**Answer**: `font-display` controls font loading behavior:
- `auto`: Browser default (usually block)
- `block`: Short invisible period (3s), then swap
- `swap`: Immediate fallback, swap when ready (best for body text)
- `fallback`: Very short block (100ms), limited swap window (3s)
- `optional`: Very short block (100ms), no swap (best for performance)

### 3. What is font subsetting and why is it important?

**Answer**: Font subsetting removes unused glyphs from font files to reduce file size. A full font might contain 10,000+ glyphs (all Unicode characters), but most sites only need 100-500 (Latin characters). Subsetting can reduce font size from 500KB to 50KB. Use `unicode-range` in CSS to load different subsets for different languages.

### 4. How do variable fonts improve performance?

**Answer**: Variable fonts contain multiple font variations (weights, widths, styles) in a single file. Instead of loading separate files for weights 400, 600, 700 (3 files, ~150KB total), you load one variable font file (~80KB) that includes the entire weight range (100-900). This reduces HTTP requests and total file size, while enabling any intermediate weight value.

### 5. Why is the crossorigin attribute required for font preloading?

**Answer**: Fonts are fetched in anonymous CORS mode, even for same-origin requests. Without `crossorigin="anonymous"` on the preload link, the browser will fetch the font twice: once for the preload (non-CORS) and again for the actual @font-face (CORS mode). The crossorigin attribute ensures the preloaded font matches the CORS mode of the actual request.

### 6. What is FOFT (Flash of Faux Text) and when should you use it?

**Answer**: FOFT is a progressive font loading strategy that loads the roman (regular) weight first, then loads bold/italic/other weights. Stage 1 shows content with the base font (browser-synthesized bold/italic if needed). Stage 2 replaces synthesized styles with true fonts. This provides faster initial rendering while ensuring content is readable. Use FOFT for custom brand fonts where you need multiple styles but want to prioritize content display.

### 7. How would you optimize Google Fonts for production?

**Answer**: Instead of loading from Google's CDN:
1. Download font files locally using a script
2. Subset fonts to required characters
3. Convert to WOFF2 format
4. Self-host with proper cache headers
5. Preload critical fonts
6. Use font-display: swap
This eliminates external DNS lookup, improves privacy, gives full cache control, and reduces network latency.

### 8. What metrics should you monitor for font performance?

**Answer**:
- Font load time (Resource Timing API)
- CLS (Cumulative Layout Shift) from font swapping
- FCP (First Contentful Paint) - impacted by font blocking
- Time to fonts visible (custom metric)
- Font file sizes and download times
- Percentage of users experiencing FOIT vs FOUT
Monitor using Real User Monitoring (RUM) and tools like web-vitals library.

## Key Takeaways

1. **Always use font-display: swap** for body text to prevent invisible text and improve UX
2. **Preload only critical fonts** (1-2 files) to avoid delaying other resources
3. **Subset fonts aggressively** to reduce file size by 80-90% for Latin-only content
4. **Use variable fonts** when you need multiple weights to reduce HTTP requests and file size
5. **Self-host fonts** instead of using CDNs to eliminate DNS lookup and improve privacy
6. **WOFF2 is the standard** - best compression, supported by all modern browsers
7. **Include crossorigin attribute** on font preload links to avoid double-fetching
8. **Implement FOFT strategy** for custom fonts to progressively enhance typography
9. **Match fallback font metrics** to minimize CLS when fonts swap
10. **Monitor font performance** with RUM to track real-world impact on Core Web Vitals

## Resources

- [Web Font Optimization (Google)](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/webfont-optimization)
- [CSS Font Loading API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Font_Loading_API)
- [font-display for the Masses](https://css-tricks.com/font-display-masses/)
- [Variable Fonts Guide](https://variablefonts.io/)
- [Font Subsetting (Google Fonts)](https://developers.google.com/fonts/docs/getting_started#specifying_font_families_and_styles_in_a_stylesheet_url)
- [glyphhanger (subsetting tool)](https://github.com/zachleat/glyphhanger)
- [A Comprehensive Guide to Font Loading Strategies](https://www.zachleat.com/web/comprehensive-webfonts/)
- [System Font Stack](https://systemfontstack.com/)
