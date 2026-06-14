# High Contrast & Forced Colors Mode

## The Idea

**In plain English:** Windows High Contrast mode (and the CSS `forced-colors` media query that represents it) is an operating-system-level accessibility feature where the OS replaces every color on screen with a small palette chosen by the user — typically white text on black, or black text on white. Your carefully chosen brand colors, gradients, box-shadows, and background images are all thrown out. The OS owns the palette. Your job is to ensure your UI remains usable — that icons are still visible, interactive elements still look interactive, and focus indicators still appear — even when you have no control over color.

**Real-world analogy:** Imagine a photocopier set to black-and-white mode. Whatever color document you feed in — purple marketing brochure, green financial report — the output is always high-contrast black and white. If your design relies on a light grey border to separate two sections, it vanishes. If your "active" state is communicated only by a color change from grey to blue, those sections look identical. A well-built document uses bold text, borders, and underlines — not just color — so the photocopied version still makes structural sense. Forced colors is that photocopier.

The key insight: `forced-colors` is not a dark mode variant. It is a complete color override. Anything you draw with `background`, `border`, `box-shadow`, `color`, `fill`, and `stroke` will be replaced by system colors. Only the layout and geometry survives untouched.

---

## Learning Objectives

- Understand the difference between `prefers-color-scheme` and `forced-colors`
- Know which CSS properties are overridden and which are left alone
- Use `SystemColor` keywords to paint with the OS palette intentionally
- Apply the transparent border trick to keep layout-critical borders visible
- Ensure SVG icons remain visible in forced-colors environments
- Test in Windows High Contrast mode and with browser DevTools emulation
- Audit existing components and identify what breaks without any code changes

---

## Why CSS Alone Isn't Enough

CSS can style components for forced-colors environments, but several aspects of the problem require deliberate architectural decisions:

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Detect forced-colors at runtime | ✅ `@media (forced-colors: active)` | — |
| Use system palette colors intentionally | ✅ `SystemColor` keywords | — |
| Preserve layout-critical transparent borders | ✅ `border: 1px solid transparent` | — |
| Make CSS `box-shadow`-only focus rings visible | ❌ | `box-shadow` is suppressed in forced-colors; must use `outline` |
| Keep SVG icon fill visible when inline | ⚠️ | Needs `fill: currentColor` or explicit `forced-color-adjust` |
| Prevent the OS overriding a specific element's colors | ⚠️ | `forced-color-adjust: none` opts out but loses accessibility guarantees |
| Make Canvas- or WebGL-drawn UI accessible | ❌ | Canvas pixels are not CSS and will not be adjusted; requires a DOM fallback |
| Test without a Windows machine | ⚠️ | Chromium DevTools emulates it; Firefox does not yet |

**Conclusion:** CSS `@media (forced-colors: active)` combined with `SystemColor` keywords handles most cases. The real discipline is avoiding anti-patterns — shadow-only focus rings, color-only state indicators, icon-as-background-image — before forced-colors mode is ever involved.

---

## HTML & CSS Foundation

### SystemColor Keywords

The browser exposes the OS palette through a fixed set of CSS keywords that resolve to the correct color for the active high-contrast theme:

```css
/*
  SystemColor keyword reference — use these inside forced-colors blocks
  to paint with the OS palette rather than hardcoded values.

  ButtonFace       — background of buttons
  ButtonText       — foreground of buttons
  Canvas           — page/element background
  CanvasText       — default text on Canvas
  Field            — input background
  FieldText        — input foreground text
  GrayText         — disabled text
  Highlight        — selected item background
  HighlightText    — selected item foreground
  LinkText         — unvisited link color
  VisitedText      — visited link color
  Mark             — highlighted/marked text background
  MarkText         — highlighted/marked text foreground
  ActiveText       — active/pressed link
*/
```

### The Transparent Border Trick

In normal mode, many cards and panels use `box-shadow` or `background-color` differences to create visual separation. Both are suppressed in forced-colors. A `1px solid transparent` border costs nothing in normal mode but becomes a solid `CanvasText`-colored border in forced-colors, preserving the structural boundary:

```css
.card {
  background: var(--surface-color);
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);

  /* transparent in normal mode, visible in forced-colors */
  border: 1px solid transparent;
}

/*
  No @media block needed — the browser automatically makes the
  transparent border visible in forced-colors mode because the
  spec requires borders to resolve to a system color, not transparent.
  The border: transparent pattern is a universal fallback strategy.
*/
```

### Focus Indicators

`box-shadow`-based focus rings are the most common breakage point:

```css
/* BAD: box-shadow is suppressed in forced-colors */
.btn:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3px #4f9cf9;
}

/* GOOD: outline survives forced-colors */
.btn:focus-visible {
  outline: 3px solid Highlight;    /* system Highlight color in forced-colors */
  outline-offset: 2px;
}

/* BEST: combine both — custom style in normal mode, system color in forced */
.btn:focus-visible {
  outline: 3px solid var(--focus-ring-color, #4f9cf9);
  outline-offset: 2px;
  box-shadow: none;   /* don't double up in normal mode */
}

@media (forced-colors: active) {
  .btn:focus-visible {
    outline-color: Highlight;
  }
}
```

### Icon Visibility

SVG icons rendered as `<img>` tags lose all color information in forced-colors — the browser can only invert their luminance, which often produces an unrecognizable blob. Inline SVG with `fill: currentColor` inherits `CanvasText` automatically:

```css
/* Pattern 1: inline SVG inherits currentColor */
.icon {
  width: 20px;
  height: 20px;
  fill: currentColor;       /* inherits from parent color, which becomes CanvasText */
  flex-shrink: 0;
}

/* Pattern 2: <img> icon — must use forced-color-adjust to invert */
@media (forced-colors: active) {
  .icon-img {
    forced-color-adjust: none;   /* opt this element out of color forcing */
    filter: invert(1);           /* manually invert so it's visible on Canvas */
  }
}

/* Pattern 3: CSS background-image icon — becomes invisible in forced-colors */
/* Avoid this pattern for functional icons; use inline SVG or <img> with alt text */
.icon-bg {
  background-image: url('/icons/arrow.svg');
  /* forced-colors: background-image is suppressed — icon disappears */
}

@media (forced-colors: active) {
  /* Fallback: render a unicode glyph instead */
  .icon-bg::before {
    content: '→';
  }
  .icon-bg {
    background-image: none;
  }
}
```

### State Indicators Beyond Color

Any state communicated by color alone breaks in forced-colors:

```css
/* BAD: active tab is blue text, inactive is grey — looks identical in forced-colors */
.tab { color: #888; }
.tab.is-active { color: #2563eb; }

/* GOOD: combine color with a structural indicator */
.tab {
  color: GrayText;
  border-bottom: 3px solid transparent;
  padding-bottom: 4px;
}

.tab.is-active {
  color: CanvasText;     /* becomes CanvasText in forced-colors automatically */
  border-bottom-color: Highlight;
}

/* In forced-colors the border-bottom on the active tab remains visible
   because it uses a SystemColor keyword */
```

---

## React Implementation

### useHighContrast Hook

```tsx
// useHighContrast.ts
import { useState, useEffect } from 'react';

interface HighContrastState {
  isActive: boolean;
  isNone: boolean;       // forced-colors: none (normal mode)
}

export function useHighContrast(): HighContrastState {
  const query = '(forced-colors: active)';

  const [isActive, setIsActive] = useState<boolean>(
    () => typeof window !== 'undefined' && window.matchMedia(query).matches
  );

  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setIsActive(e.matches);

    mql.addEventListener('change', handler);
    return () => mql.removeEventListener('change', handler);
  }, []);

  return { isActive, isNone: !isActive };
}
```

### FocusRing Utility Component

```tsx
// FocusRing.tsx
// Wraps any interactive child to guarantee a visible focus ring in all modes.
import React from 'react';
import './FocusRing.css';

interface FocusRingProps {
  children: React.ReactElement;
  offset?: number;
}

export function FocusRing({ children, offset = 2 }: FocusRingProps) {
  return React.cloneElement(children, {
    className: [children.props.className, 'focus-ring-target'].filter(Boolean).join(' '),
    style: {
      ...children.props.style,
      '--focus-ring-offset': `${offset}px`,
    } as React.CSSProperties,
  });
}
```

```css
/* FocusRing.css */
.focus-ring-target:focus {
  /* Remove default outline so our custom one is the only one */
  outline: none;
}

.focus-ring-target:focus-visible {
  outline: 3px solid #4f9cf9;
  outline-offset: var(--focus-ring-offset, 2px);
}

@media (forced-colors: active) {
  .focus-ring-target:focus-visible {
    /* Highlight is the system selection/focus color */
    outline-color: Highlight;
  }
}
```

### StatusBadge Component

This component demonstrates how to communicate status without relying on color alone:

```tsx
// StatusBadge.tsx
import './StatusBadge.css';

type Status = 'success' | 'warning' | 'error' | 'info' | 'neutral';

interface StatusBadgeProps {
  status: Status;
  label: string;
}

// Each status has a symbol fallback so information is never color-only
const STATUS_META: Record<Status, { symbol: string; ariaLabel: string }> = {
  success: { symbol: '✓', ariaLabel: 'Success' },
  warning: { symbol: '!', ariaLabel: 'Warning' },
  error:   { symbol: '✕', ariaLabel: 'Error' },
  info:    { symbol: 'i', ariaLabel: 'Info' },
  neutral: { symbol: '–', ariaLabel: 'Neutral' },
};

export function StatusBadge({ status, label }: StatusBadgeProps) {
  const meta = STATUS_META[status];

  return (
    <span
      className={`status-badge status-badge--${status}`}
      role="status"
    >
      {/* Symbol is visible in forced-colors; color is the normal-mode signal */}
      <span className="status-badge__symbol" aria-hidden="true">
        {meta.symbol}
      </span>
      <span className="status-badge__label">{label}</span>
      {/* Screen reader gets both the status category and the label */}
      <span className="sr-only">{meta.ariaLabel}:</span>
    </span>
  );
}
```

```css
/* StatusBadge.css */
.status-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 0.8125rem;
  font-weight: 500;

  /* Transparent border becomes visible in forced-colors */
  border: 1px solid transparent;
}

.status-badge--success { background: #dcfce7; color: #166534; }
.status-badge--warning { background: #fef9c3; color: #854d0e; }
.status-badge--error   { background: #fee2e2; color: #991b1b; }
.status-badge--info    { background: #dbeafe; color: #1e40af; }
.status-badge--neutral { background: #f3f4f6; color: #374151; }

@media (forced-colors: active) {
  /* All color overrides are dropped by OS — reset our structural properties */
  .status-badge {
    border-color: CanvasText;   /* explicit border so boundary is always visible */
    background: Canvas;
    color: CanvasText;
  }

  /* The symbol already carries the semantic meaning, so no extra work needed */
  .status-badge__symbol {
    color: CanvasText;
  }
}
```

### Icon Component with Forced-Colors Handling

```tsx
// Icon.tsx
import { SVGProps } from 'react';
import './Icon.css';

interface IconProps extends SVGProps<SVGSVGElement> {
  name: 'arrow-right' | 'check' | 'warning' | 'close' | 'info';
  size?: number;
  label?: string;   // if provided, icon is not decorative
}

// Inline SVG paths — fill: currentColor inherits correctly in all modes
const PATHS: Record<IconProps['name'], string> = {
  'arrow-right': 'M5 12h14M12 5l7 7-7 7',
  'check':       'M20 6L9 17l-5-5',
  'warning':     'M12 9v4M12 17h.01M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z',
  'close':       'M18 6L6 18M6 6l12 12',
  'info':        'M12 16v-4M12 8h.01',
};

export function Icon({ name, size = 20, label, className = '', ...rest }: IconProps) {
  return (
    <svg
      xmlns="http://www.w3.org/2000/svg"
      width={size}
      height={size}
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"   /* currentColor = CanvasText in forced-colors */
      strokeWidth={2}
      strokeLinecap="round"
      strokeLinejoin="round"
      className={`icon ${className}`}
      aria-hidden={!label}
      aria-label={label}
      role={label ? 'img' : undefined}
      {...rest}
    >
      <path d={PATHS[name]} />
    </svg>
  );
}
```

---

## Angular Implementation

### ForcedColorsService

```typescript
// forced-colors.service.ts
import { Injectable, signal, OnDestroy } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ForcedColorsService implements OnDestroy {
  private readonly query = window.matchMedia('(forced-colors: active)');
  private readonly handler = (e: MediaQueryListEvent) =>
    this._isActive.set(e.matches);

  private _isActive = signal<boolean>(this.query.matches);
  readonly isActive = this._isActive.asReadonly();

  constructor() {
    this.query.addEventListener('change', this.handler);
  }

  ngOnDestroy(): void {
    this.query.removeEventListener('change', this.handler);
  }
}
```

### StatusBadge Component (Angular)

```typescript
// status-badge.component.ts
import { Component, Input } from '@angular/core';
import { NgClass } from '@angular/common';

type Status = 'success' | 'warning' | 'error' | 'info' | 'neutral';

@Component({
  selector: 'app-status-badge',
  standalone: true,
  imports: [NgClass],
  template: `
    <span
      class="status-badge"
      [ngClass]="'status-badge--' + status"
      role="status"
    >
      <span class="status-badge__symbol" aria-hidden="true">{{ symbol }}</span>
      <span class="status-badge__label">{{ label }}</span>
      <span class="sr-only">{{ ariaLabel }}:</span>
    </span>
  `,
  styleUrls: ['./status-badge.component.css'],
})
export class StatusBadgeComponent {
  @Input({ required: true }) status!: Status;
  @Input({ required: true }) label!: string;

  private static readonly META: Record<Status, { symbol: string; ariaLabel: string }> = {
    success: { symbol: '✓', ariaLabel: 'Success' },
    warning: { symbol: '!', ariaLabel: 'Warning' },
    error:   { symbol: '✕', ariaLabel: 'Error' },
    info:    { symbol: 'i', ariaLabel: 'Info' },
    neutral: { symbol: '–', ariaLabel: 'Neutral' },
  };

  get symbol(): string {
    return StatusBadgeComponent.META[this.status].symbol;
  }

  get ariaLabel(): string {
    return StatusBadgeComponent.META[this.status].ariaLabel;
  }
}
```

### HighContrastAwareButton

```typescript
// hc-button.component.ts
import { Component, Input, inject } from '@angular/core';
import { ForcedColorsService } from './forced-colors.service';

type ButtonVariant = 'primary' | 'secondary' | 'ghost';

@Component({
  selector: 'app-hc-button',
  standalone: true,
  template: `
    <button
      class="btn"
      [class.btn--primary]="variant === 'primary'"
      [class.btn--secondary]="variant === 'secondary'"
      [class.btn--ghost]="variant === 'ghost'"
      [disabled]="disabled"
    >
      <ng-content />
    </button>
  `,
  styles: [`
    /*
      Styles are co-located here for clarity.
      In a real project, move to a shared stylesheet.
    */
    .btn {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      padding: 10px 20px;
      min-height: 44px;
      font-size: 0.9375rem;
      font-weight: 600;
      border-radius: 6px;
      cursor: pointer;
      transition: background 0.15s, color 0.15s;

      /* Transparent border ensures a boundary in forced-colors */
      border: 2px solid transparent;
    }

    .btn--primary {
      background: #2563eb;
      color: #ffffff;
    }

    .btn--secondary {
      background: transparent;
      color: #2563eb;
      border-color: #2563eb;
    }

    .btn--ghost {
      background: transparent;
      color: #374151;
    }

    .btn:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    .btn:focus-visible {
      outline: 3px solid #4f9cf9;
      outline-offset: 2px;
    }

    @media (forced-colors: active) {
      .btn--primary {
        background: ButtonFace;
        color: ButtonText;
        border-color: ButtonText;
      }

      .btn--secondary {
        background: Canvas;
        color: ButtonText;
        border-color: ButtonText;
      }

      .btn--ghost {
        background: Canvas;
        color: ButtonText;
        border-color: ButtonText;
      }

      .btn:disabled {
        color: GrayText;
        border-color: GrayText;
        opacity: 1;   /* OS already dims it; double opacity makes it too faint */
      }

      .btn:focus-visible {
        outline-color: Highlight;
      }
    }
  `],
})
export class HcButtonComponent {
  @Input() variant: ButtonVariant = 'primary';
  @Input() disabled = false;

  readonly forcedColors = inject(ForcedColorsService);
}
```

### Form Input with Forced-Colors Guard

```typescript
// hc-input.component.ts
import { Component, Input, forwardRef } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-hc-input',
  standalone: true,
  imports: [ReactiveFormsModule],
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => HcInputComponent),
    multi: true,
  }],
  template: `
    <div class="field-wrapper">
      <label [for]="id" class="field-label">{{ label }}</label>
      <input
        [id]="id"
        class="field-input"
        [type]="type"
        [value]="value"
        [disabled]="isDisabled"
        (input)="onInput($event)"
        (blur)="onTouched()"
      />
    </div>
  `,
  styles: [`
    .field-wrapper { display: flex; flex-direction: column; gap: 4px; }

    .field-label {
      font-size: 0.875rem;
      font-weight: 500;
      color: CanvasText;   /* always correct in both normal and forced-colors */
    }

    .field-input {
      height: 44px;
      padding: 0 12px;
      border: 1.5px solid #d1d5db;
      border-radius: 6px;
      font-size: 1rem;
      background: #ffffff;
      color: #111827;
    }

    .field-input:focus-visible {
      outline: 3px solid #4f9cf9;
      outline-offset: 0;
      border-color: #2563eb;
    }

    .field-input:disabled {
      background: #f9fafb;
      color: #9ca3af;
      cursor: not-allowed;
    }

    @media (forced-colors: active) {
      .field-input {
        background: Field;
        color: FieldText;
        border-color: CanvasText;
      }

      .field-input:focus-visible {
        outline-color: Highlight;
        border-color: Highlight;
      }

      .field-input:disabled {
        background: Field;
        color: GrayText;
        border-color: GrayText;
      }
    }
  `],
})
export class HcInputComponent implements ControlValueAccessor {
  @Input({ required: true }) id!: string;
  @Input({ required: true }) label!: string;
  @Input() type: 'text' | 'email' | 'password' | 'number' = 'text';

  value = '';
  isDisabled = false;
  onChange: (v: string) => void = () => {};
  onTouched: () => void = () => {};

  onInput(event: Event): void {
    this.value = (event.target as HTMLInputElement).value;
    this.onChange(this.value);
  }

  writeValue(value: string): void { this.value = value ?? ''; }
  registerOnChange(fn: (v: string) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }
  setDisabledState(isDisabled: boolean): void { this.isDisabled = isDisabled; }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Focus rings visible in forced-colors | Use `outline` not `box-shadow`; set `outline-color: Highlight` in `@media (forced-colors: active)` |
| Cards and panels have a visible boundary | `border: 1px solid transparent` on every card — becomes visible automatically |
| Inline SVG icons use `fill: currentColor` or `stroke: currentColor` | Inherits `CanvasText` automatically; no `@media` block needed |
| Background-image icons have a forced-colors fallback | `::before` pseudo-element with a Unicode fallback character |
| Status communicated by symbol as well as color | Include a text symbol (`✓`, `!`, `✕`) alongside color-coded badges |
| Disabled state uses `GrayText` | Avoid `opacity: 0.5` in forced-colors — OS already provides disabled styling |
| Button and input borders are always explicit | `border: 2px solid ButtonText` inside the `forced-colors` block |
| `<img>` icons that must be preserved | `forced-color-adjust: none` + `filter: invert(1)` if background is Canvas |
| Link color distinct from body text | `LinkText` and `VisitedText` system keywords for anchor colors |
| Selected/highlighted state uses `Highlight` | Background `Highlight`, foreground `HighlightText` for selected items |
| Text on interactive elements uses `ButtonText` | Not `CanvasText` — ensures contrast against `ButtonFace` background |
| Testing done in actual Windows High Contrast mode | Chrome DevTools > Rendering > Emulate CSS media > `forced-colors: active` |

---

## Production Pitfalls

**1. `box-shadow` focus rings disappear completely**
The single most common failure. `box-shadow` is in the forced-colors suppression list — it is zeroed out by the browser before painting. A design system that replaced `outline` with `box-shadow` for aesthetic reasons will have invisible focus rings in Windows High Contrast mode. Fix: always use `outline` as the primary focus indicator. Use `box-shadow` as a decorative supplement only, and never set `outline: none` without a proper `outline` replacement.

**2. Transparent separators and dividers vanish**
`<hr>` elements and dividers that rely on `background-color: #eee` or `border-color: #eee` map to shades that become invisible against `Canvas` in most high-contrast themes. Fix: use `border: 1px solid CanvasText` inside a `@media (forced-colors: active)` block, or adopt the transparent border pattern universally so all structural boundaries survive.

**3. `forced-color-adjust: none` misused as a global fix**
Developers discover that `forced-color-adjust: none` opts an element out of color forcing and apply it broadly to "restore" their design. This defeats the purpose of the feature: users who need high contrast for medical or visual reasons will be unable to use those components. Reserve `forced-color-adjust: none` for elements where the color itself is content — a color picker swatch, a brand logo — not for decorative components.

**4. SVG icons embedded as `<img>` become invisible on white themes**
An SVG icon with a dark fill looks fine on the default black forced-colors theme, but on "High Contrast White" (white background, black text) the icon was designed with a dark fill that maps to Canvas (white) — invisible. The inline SVG with `stroke: currentColor` pattern avoids this entirely because it inherits the current text color regardless of the theme variant.

**5. Gradient and image backgrounds create invisible text**
A hero section with `background-image: linear-gradient(...)` or a background photo will have the image stripped and replaced with `Canvas`. If the text color was chosen to contrast against the image and was also stripped, both text and background become the same system color. Fix: never rely on background images for text legibility; ensure text has an explicit `color` that references a `CanvasText`-adjacent system keyword so it always contrasts against `Canvas`.

**6. `prefers-color-scheme` queries do not cover forced-colors**
A common assumption is that supporting dark mode covers high-contrast users. It does not. A user can run "High Contrast Black" (which looks dark) but `prefers-color-scheme: dark` will NOT match; only `forced-colors: active` applies. Build these as separate concerns: dark mode is a preference, forced-colors is an accessibility override that replaces your entire palette.

**7. Testing only in DevTools misses real Windows behavior**
Chromium's "Emulate CSS media feature forced-colors" in the Rendering panel accurately emulates the `@media` query and `SystemColor` resolution. However, it does not emulate every edge case of the Windows rendering pipeline, such as how the OS handles `<canvas>` elements, custom scrollbars, or native form controls. Always do a final verification pass in actual Windows High Contrast mode (Settings > Accessibility > Contrast themes) before shipping a high-visibility component.

**8. Skipping the audit on third-party component libraries**
UI libraries like MUI, Radix, or Ant Design have varying levels of forced-colors support. A custom design system built on top of them can inherit unfixed gaps. Fix: test the full component set with DevTools emulation as part of CI. A Playwright test with `forcedColors: 'active'` in the launch options can screenshot components for visual regression in forced-colors mode.

---

## Interview Angle

**Q: "What is the difference between `prefers-color-scheme: dark` and `forced-colors: active`, and why does supporting one not imply the other?"**

`prefers-color-scheme` is a user preference — it signals that the user would like a dark aesthetic, but your CSS still controls every color. You can style anything. `forced-colors: active` is an operating-system intervention — the OS intercepts the CSS painting step and replaces a defined set of CSS properties (background-color, color, border-color, fill, stroke, box-shadow, outline-color) with a small palette of system colors chosen by the user. Your design is not adjusted; it is replaced. A high-contrast black theme can look dark, but `prefers-color-scheme: dark` will not match because it is not a color scheme preference — it is an accessibility override. They must be authored as separate concerns.

**Follow-up Q: "How would you audit an existing component library for forced-colors compliance?"**

The fastest path is Chromium DevTools: open the Rendering panel, set "Emulate CSS media feature forced-colors" to `active`, then walk the component inventory. Every invisible focus ring, every vanished border, every color-only state indicator surfaces immediately. Then apply fixes in priority order: (1) replace `box-shadow`-only focus rings with `outline`, (2) add `border: 1px solid transparent` to all cards and panels, (3) convert icon `<img>` tags to inline SVG with `currentColor`, (4) add `SystemColor` keyword overrides for explicit state badges. For automation, add a Playwright test suite that launches with `{ forcedColors: 'active' }` and runs visual snapshots — any forced-colors regression appears as a diff in CI before it ships.
