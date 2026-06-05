# CSS Typography

## The Idea

**In plain English:** CSS typography is about controlling how text looks on a webpage — things like which font (typeface) is used, how bold or thin the letters are, and how the browser handles loading those fonts. A font is simply a specific visual style for letters, numbers, and symbols.

**Real-world analogy:** Think of a restaurant printing its menus. The manager tells the printer "use Garamond font, but if you don't have it, use Times New Roman, and if not that, any fancy serif style will do." While the menus are being printed, customers get a plain temporary menu so they're not left waiting with nothing.

- The list of preferred fonts in order = the font stack (primary font with fallbacks)
- The temporary plain menu given while waiting = the fallback font shown during font loading
- Swapping the plain menu for the fancy one when it arrives = `font-display: swap`

---

## Font Stacks

Font stacks provide fallback fonts when primary fonts are unavailable.

```css
/* System font stack (modern approach) */
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, 
    "Helvetica Neue", Arial, sans-serif;
}

/* Serif stack */
.serif {
  font-family: Georgia, "Times New Roman", Times, serif;
}

/* Monospace stack */
code {
  font-family: "SF Mono", Monaco, "Cascadia Code", "Roboto Mono", 
    Consolas, "Courier New", monospace;
}

/* Humanist sans-serif */
.humanist {
  font-family: "Gill Sans", "Gill Sans MT", Calibri, "Trebuchet MS", sans-serif;
}
```

## Web Fonts (@font-face)

```css
/* Basic @font-face declaration */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/customfont.woff2') format('woff2'),
       url('/fonts/customfont.woff') format('woff');
  font-weight: 400;
  font-style: normal;
}

/* Multiple weights and styles */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/customfont-bold.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
}

@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/customfont-italic.woff2') format('woff2');
  font-weight: 400;
  font-style: italic;
}

/* Unicode range subsetting */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/customfont-latin.woff2') format('woff2');
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, 
    U+02C6, U+02DA, U+02DC, U+2000-206F, U+2074, U+20AC, U+2122, 
    U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}

/* Local font with fallback */
@font-face {
  font-family: 'PreferLocal';
  src: local('Arial'),
       url('/fonts/fallback.woff2') format('woff2');
}
```

## font-display

Controls how font faces are displayed based on download status.

```css
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/customfont.woff2') format('woff2');
  font-display: swap; /* Most common - show fallback immediately */
}

/* Values explained:
   - auto: Browser default behavior
   - block: Short invisible period, then swap (FOIT)
   - swap: Use fallback immediately, swap when ready (FOUT)
   - fallback: Very short invisible period, swap if loaded quickly
   - optional: Very short invisible period, browser decides if to use
*/

/* Performance-focused approach */
@font-face {
  font-family: 'BodyFont';
  src: url('/fonts/body.woff2') format('woff2');
  font-display: swap; /* Content is king */
}

@font-face {
  font-family: 'HeadingFont';
  src: url('/fonts/heading.woff2') format('woff2');
  font-display: fallback; /* Less critical, avoid layout shift */
}

@font-face {
  font-family: 'IconFont';
  src: url('/fonts/icons.woff2') format('woff2');
  font-display: block; /* Icons should appear together */
}
```

## Variable Fonts

Single font file with multiple variations.

```css
/* Variable font with multiple axes */
@font-face {
  font-family: 'InterVariable';
  src: url('/fonts/inter-variable.woff2') format('woff2-variations');
  font-weight: 100 900; /* Weight range */
  font-display: swap;
}

/* Using font-variation-settings */
.text {
  font-family: 'InterVariable', sans-serif;
  
  /* Standard properties (preferred when available) */
  font-weight: 650;
  font-style: oblique 10deg;
  
  /* Custom axes via font-variation-settings */
  font-variation-settings: 
    'wght' 650,  /* Weight */
    'slnt' -10,  /* Slant */
    'wdth' 90;   /* Width */
}

/* Animation with variable fonts */
@keyframes weightPulse {
  0%, 100% { font-variation-settings: 'wght' 400; }
  50% { font-variation-settings: 'wght' 700; }
}

.animated-text {
  font-family: 'InterVariable';
  animation: weightPulse 2s infinite;
}

/* Responsive typography with variable fonts */
h1 {
  font-family: 'InterVariable';
  font-variation-settings: 
    'wght' clamp(500, 50vw, 800),
    'wdth' clamp(75, 10vw + 75, 100);
}
```

## font-feature-settings

Enable/disable OpenType features.

```css
/* Ligatures */
.ligatures {
  /* Standard ligatures (fi, fl) */
  font-feature-settings: 'liga' 1;
  
  /* Discretionary ligatures (ct, st) */
  font-feature-settings: 'dlig' 1;
  
  /* Disable all ligatures */
  font-feature-settings: 'liga' 0, 'dlig' 0;
  
  /* Better: use font-variant-ligatures */
  font-variant-ligatures: common-ligatures discretionary-ligatures;
}

/* Numeric features */
.numbers {
  /* Old-style numerals (3 descends below baseline) */
  font-feature-settings: 'onum' 1;
  
  /* Tabular numerals (fixed width) */
  font-feature-settings: 'tnum' 1;
  
  /* Fractions */
  font-feature-settings: 'frac' 1;
  
  /* Better: use font-variant-numeric */
  font-variant-numeric: oldstyle-nums tabular-nums diagonal-fractions;
}

/* Small caps */
.small-caps {
  font-feature-settings: 'smcp' 1;
  
  /* Better: use font-variant-caps */
  font-variant-caps: small-caps;
}

/* Stylistic sets */
.stylistic-set-1 {
  font-feature-settings: 'ss01' 1; /* Enable stylistic set 1 */
}

/* Kerning */
.kerning {
  font-feature-settings: 'kern' 1;
  
  /* Better: use font-kerning */
  font-kerning: normal;
}

/* Multiple features */
.advanced-typography {
  font-feature-settings: 
    'liga' 1,  /* Ligatures */
    'kern' 1,  /* Kerning */
    'onum' 1,  /* Old-style numerals */
    'pnum' 1;  /* Proportional numerals */
}
```

## Font Loading Strategies

### 1. FOUT (Flash of Unstyled Text)
Show fallback font immediately, swap when web font loads.

```css
/* Using font-display: swap */
@font-face {
  font-family: 'WebFont';
  src: url('/fonts/webfont.woff2') format('woff2');
  font-display: swap;
}

/* Minimize layout shift with fallback matching */
body {
  font-family: 'WebFont', Arial, sans-serif;
  font-size: 16px;
  
  /* Adjust fallback to match web font metrics */
  font-size-adjust: 0.5; /* Preserve x-height */
}
```

### 2. FOIT (Flash of Invisible Text)
Hide text until font loads (briefly).

```css
@font-face {
  font-family: 'WebFont';
  src: url('/fonts/webfont.woff2') format('woff2');
  font-display: block; /* 3s invisible period */
}
```

### 3. FOFT (Flash of Faux Text)
Load roman weight first, then other weights.

```html
<script>
// Load critical roman weight first
const roman = new FontFace('WebFont', 'url(/fonts/webfont-roman.woff2)', {
  weight: '400',
  style: 'normal'
});

roman.load().then(font => {
  document.fonts.add(font);
  document.documentElement.classList.add('fonts-loaded-roman');
  
  // Then load other weights
  const bold = new FontFace('WebFont', 'url(/fonts/webfont-bold.woff2)', {
    weight: '700'
  });
  
  bold.load().then(font => {
    document.fonts.add(font);
    document.documentElement.classList.add('fonts-loaded-all');
  });
});
</script>
```

```css
/* Progressive enhancement */
body {
  font-family: Arial, sans-serif;
}

.fonts-loaded-roman body {
  font-family: 'WebFont', Arial, sans-serif;
}

/* Faux bold until real bold loads */
.fonts-loaded-roman strong {
  font-weight: 600; /* Slightly lighter faux bold */
}

.fonts-loaded-all strong {
  font-weight: 700; /* Real bold */
}
```

### 4. JavaScript Font Loading API

```html
<script>
// Check if font is already cached
if (document.fonts.check('16px WebFont')) {
  document.documentElement.classList.add('fonts-loaded');
} else {
  // Load font
  document.fonts.load('16px WebFont').then(() => {
    document.documentElement.classList.add('fonts-loaded');
  });
  
  // Set timeout for fallback
  setTimeout(() => {
    document.documentElement.classList.add('fonts-failed');
  }, 3000);
}

// Listen to all fonts loaded
document.fonts.ready.then(() => {
  console.log('All fonts loaded');
});

// Monitor font loading
document.fonts.addEventListener('loadingdone', (event) => {
  event.fontfaces.forEach(fontFace => {
    console.log(`Loaded: ${fontFace.family} ${fontFace.weight}`);
  });
});
</script>
```

### 5. Preloading Fonts

```html
<!-- Preload critical fonts -->
<link rel="preload" 
      href="/fonts/webfont.woff2" 
      as="font" 
      type="font/woff2" 
      crossorigin>

<!-- Preconnect to font CDN -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

## Font Subsetting

Reduce file size by including only needed characters.

```bash
# Using pyftsubset (fonttools)
pyftsubset input.ttf \
  --output-file=output.woff2 \
  --flavor=woff2 \
  --layout-features=* \
  --unicodes=U+0020-007E,U+00A0-00FF

# Subset for specific language
pyftsubset input.ttf \
  --output-file=latin.woff2 \
  --flavor=woff2 \
  --unicodes-file=latin-unicodes.txt
```

```css
/* Use subsetted fonts */
@font-face {
  font-family: 'WebFont';
  src: url('/fonts/webfont-latin.woff2') format('woff2');
  unicode-range: U+0020-007E, U+00A0-00FF;
}

@font-face {
  font-family: 'WebFont';
  src: url('/fonts/webfont-extended.woff2') format('woff2');
  unicode-range: U+0100-017F;
}
```

## Performance Tips

```css
/* 1. Minimize font weights/styles */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
/* Only import what you need, not all weights */

/* 2. Use system fonts for better performance */
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  /* Zero network requests, instant rendering */
}

/* 3. Subset fonts aggressively */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom-subset.woff2') format('woff2');
  unicode-range: U+0020-007E; /* Basic Latin only */
}

/* 4. Optimize font-display */
@font-face {
  font-family: 'BodyFont';
  src: url('/fonts/body.woff2') format('woff2');
  font-display: swap; /* Prioritize content visibility */
}

/* 5. Match fallback font metrics */
@font-face {
  font-family: 'HeadingFont';
  src: url('/fonts/heading.woff2') format('woff2');
  font-display: swap;
  ascent-override: 90%;
  descent-override: 20%;
  line-gap-override: 0%;
  size-adjust: 105%;
}

/* 6. Use variable fonts to reduce requests */
@font-face {
  font-family: 'InterVar';
  src: url('/fonts/inter-var.woff2') format('woff2-variations');
  font-weight: 100 900; /* Single file for all weights */
}
```

## Common Patterns

```css
/* Responsive typography */
.responsive-text {
  font-size: clamp(1rem, 2vw + 0.5rem, 2rem);
  line-height: 1.5;
  letter-spacing: -0.01em;
}

/* Fluid typography with variable fonts */
.fluid-heading {
  font-family: 'InterVariable', sans-serif;
  font-size: clamp(2rem, 5vw, 4rem);
  font-variation-settings: 'wght' clamp(500, 50vw, 800);
}

/* Optical sizing for better readability */
.optical-sizing {
  font-optical-sizing: auto; /* Use optical size axis if available */
}

/* Prevent layout shift */
.font-loading {
  font-family: Arial, sans-serif;
  font-size: 16px;
}

.fonts-loaded .font-loading {
  font-family: 'WebFont', Arial, sans-serif;
  /* Match metrics to prevent shift */
}

/* Fallback for variable fonts */
.variable-font-text {
  font-family: 'InterVariable', -apple-system, sans-serif;
  font-weight: 450;
  
  /* Fallback for browsers without variable font support */
  @supports not (font-variation-settings: normal) {
    font-weight: 400;
  }
}
```

## Gotchas

1. **Font loading blocks render**
   ```css
   /* Bad: Blocks rendering for 3 seconds */
   @font-face {
     font-family: 'Heavy';
     src: url('/fonts/heavy.woff2');
     font-display: block;
   }
   
   /* Good: Shows content immediately */
   @font-face {
     font-family: 'Heavy';
     src: url('/fonts/heavy.woff2');
     font-display: swap;
   }
   ```

2. **Layout shift from font swap**
   ```css
   /* Problem: Different metrics cause layout shift */
   body { font-family: 'WebFont', Arial; }
   
   /* Solution: Match fallback metrics */
   @font-face {
     font-family: 'WebFont';
     src: url('/fonts/webfont.woff2');
     font-display: swap;
     size-adjust: 97%;
     ascent-override: 100%;
   }
   ```

3. **Missing unicode-range causes unnecessary downloads**
   ```css
   /* Bad: Downloads even if characters not used */
   @font-face {
     font-family: 'WebFont';
     src: url('/fonts/full.woff2');
   }
   
   /* Good: Only downloads when needed */
   @font-face {
     font-family: 'WebFont';
     src: url('/fonts/latin.woff2');
     unicode-range: U+0000-00FF;
   }
   ```

4. **font-variation-settings overrides standard properties**
   ```css
   /* font-weight is ignored */
   .element {
     font-weight: 700;
     font-variation-settings: 'wght' 400; /* Wins */
   }
   
   /* Solution: Use standard properties or variation settings, not both */
   .element {
     font-variation-settings: 'wght' 700;
   }
   ```

5. **CORS issues with fonts**
   ```html
   <!-- Fonts must be served with CORS headers -->
   <link rel="preload" 
         href="https://cdn.com/font.woff2" 
         as="font" 
         crossorigin> <!-- Required! -->
   ```

## Interview Questions

**Q: Explain the difference between FOUT, FOIT, and FOFT.**

A: 
- FOUT (Flash of Unstyled Text): Shows fallback font immediately, swaps to web font when loaded. Achieved with `font-display: swap`. Prioritizes content visibility but may cause layout shift.
- FOIT (Flash of Invisible Text): Hides text briefly while font loads. Achieved with `font-display: block`. Avoids layout shift but delays content visibility.
- FOFT (Flash of Faux Text): Loads critical roman weight first, shows content, then loads other weights. Combines benefits of both - fast initial render with progressive enhancement.

**Q: How does font-display work and which value should you use?**

A: `font-display` controls font rendering behavior during load:
- `swap`: Show fallback immediately, swap when loaded (most common)
- `block`: Short invisible period, then swap (3s max)
- `fallback`: Very short invisible (~100ms), swap only if fast
- `optional`: Very short invisible, browser decides based on connection
- `auto`: Browser default

Use `swap` for body text (content is priority), `fallback` for headings (reduce layout shift), `block` for icon fonts (avoid missing icons).

**Q: What are variable fonts and what are their benefits?**

A: Variable fonts contain multiple variations (weights, widths, slants) in a single file using interpolation. Benefits:
- Reduced HTTP requests (one file vs multiple)
- Smaller total file size
- Smooth animations between weights/widths
- Fine-grained control (font-weight: 450 instead of just 400/700)
- Responsive typography with custom axes

Use with `font-variation-settings` or standard properties like `font-weight`.

**Q: How do you optimize web font loading performance?**

A: Multiple strategies:
1. Use `font-display: swap` for critical fonts
2. Preload critical fonts: `<link rel="preload" as="font">`
3. Subset fonts to include only needed characters
4. Use WOFF2 format (best compression)
5. Use variable fonts to reduce requests
6. Implement font loading strategies (FOFT)
7. Match fallback font metrics to reduce layout shift
8. Use unicode-range for automatic subsetting
9. Limit number of font weights/styles
10. Consider system fonts for non-critical text

**Q: What is font-feature-settings and when should you use it?**

A: `font-feature-settings` enables/disables OpenType font features like ligatures, old-style numerals, kerning, small caps. Use for:
- Standard ligatures: `'liga' 1`
- Tabular numbers: `'tnum' 1`
- Fractions: `'frac' 1`
- Stylistic sets: `'ss01' 1`

However, prefer higher-level properties when available:
- `font-variant-ligatures` instead of `'liga'`
- `font-variant-numeric` instead of `'onum'`, `'tnum'`
- `font-variant-caps` instead of `'smcp'`

These are more maintainable and cascade better.

**Q: How do you prevent layout shift when web fonts load?**

A: Several approaches:
1. Match fallback font metrics using font descriptors:
   ```css
   @font-face {
     font-family: 'WebFont';
     ascent-override: 90%;
     descent-override: 20%;
     size-adjust: 105%;
   }
   ```
2. Use `font-size-adjust` to maintain x-height
3. Use FOIT strategy with short block period
4. Preload critical fonts
5. Inline critical font files
6. Use system fonts (no shift)
7. Test with tools like Cumulative Layout Shift (CLS)

**Q: What is unicode-range and how does it optimize font loading?**

A: `unicode-range` specifies which Unicode characters a font should be used for. Browser only downloads font if page contains those characters. Enables:
- Automatic subsetting by language/script
- Progressive enhancement (load extended characters only if needed)
- Reduced bandwidth for most users

Example:
```css
@font-face {
  font-family: 'WebFont';
  src: url('latin.woff2');
  unicode-range: U+0000-00FF; /* Only Latin */
}
@font-face {
  font-family: 'WebFont';
  src: url('extended.woff2');
  unicode-range: U+0100-017F; /* Extended Latin */
}
```

**Q: Explain the font loading timeline and blocking behavior.**

A: Browser font loading has three periods:
1. Block period (0-3s): Text invisible, waiting for font
2. Swap period (3s-infinite): Show fallback, swap when font loads
3. Failure period: Use fallback permanently if font fails

`font-display` controls these periods:
- `block`: 3s block, infinite swap
- `swap`: 0s block, infinite swap
- `fallback`: 100ms block, 3s swap
- `optional`: 100ms block, no swap (browser decides)

Critical to understand: fonts block rendering during block period, causing delayed First Contentful Paint (FCP).
