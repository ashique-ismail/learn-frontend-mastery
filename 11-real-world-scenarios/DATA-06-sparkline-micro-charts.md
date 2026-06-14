# Sparkline & Inline Micro Charts

## The Idea

**In plain English:** A sparkline is a tiny chart — typically 60-120px wide, 30-40px tall — embedded inline with text or inside a table cell to show a trend at a glance without axes, labels, or the visual weight of a full chart. You generate an SVG `<polyline>` by mapping a raw data array to x/y pixel coordinates, then render it directly in the DOM. No external chart library is needed, no canvas, no WebGL.

**Real-world analogy:** Think of a stock price dashboard in a financial terminal.

- The **full-page chart** with axes, zoom controls, and a date legend = a Chart.js or D3 bar chart. It tells the whole story.
- The **tiny line next to a ticker symbol** (AAPL ▲ $189.23 `~~~~`) = the sparkline. It answers *one question only*: is this thing trending up, down, or flat over the last N periods?
- The **table of 50 stocks**, each with its own sparkline = why you cannot use a full chart library per row. At 50 chart instances the page becomes unresponsive.
- **The SVG polyline path** you compute yourself = a phone screen printed on a receipt. It has just enough ink to convey the trend and no more.
- **The tooltip** that appears on hover = the waiter who, when you point at a line item on the receipt, tells you the exact value.

The key insight: a sparkline is data visualization stripped to its mathematical minimum — a coordinate transform from a data domain to a pixel range, expressed as a single SVG element.

---

## Learning Objectives

- Understand the coordinate transform from data domain to SVG viewBox pixel space
- Build a headless sparkline component that owns zero styling opinions
- Generate SVG `<polyline>` point strings and area `<path>` fill from a raw number array
- Implement responsive sizing using `viewBox` + `preserveAspectRatio` instead of fixed pixel dimensions
- Render a positioned tooltip on hover without a library, correctly handling edge cases at the chart boundary
- Build both a React hook-based and an Angular signal-based implementation
- Know the accessibility requirements for decorative versus informative sparklines

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Draw a line through arbitrary data points | ❌ | CSS cannot compute coordinate transforms from a data array |
| Scale the line to fit any container size | ❌ | Scaling requires reading container dimensions and remapping data points |
| Fill the area under the line | ❌ | Closed path generation requires knowing the first and last x coordinate |
| Show tooltip with the exact value at the hovered point | ❌ | Requires identifying which point is nearest the cursor (hit-test logic) |
| Change line color based on trend direction (up/down) | ⚠️ | CSS can apply a class, but JS must compute whether the trend is positive |
| Render from a dynamic API data array | ❌ | CSS has no iteration or data binding |
| Animate the line drawing on mount | ⚠️ | CSS `stroke-dashoffset` animation works, but JS must first measure the path `getTotalLength()` |

**Conclusion:** CSS owns color tokens, the tooltip bubble's visual style, and animation easing. JS owns every coordinate, every path string, every hover hit-test, and every trend computation.

---

## HTML & CSS Foundation

### The SVG Structure

```html
<!-- Sparkline wrapper — inline-block so it sits next to text -->
<span class="sparkline-wrapper" aria-hidden="true">
  <svg
    class="sparkline"
    viewBox="0 0 100 32"
    preserveAspectRatio="none"
    width="100"
    height="32"
    role="img"
    aria-label="Price trend over 30 days"
  >
    <!-- Area fill under the line -->
    <path
      class="sparkline__area"
      d="M0,28 L10,20 L20,22 L30,10 L40,14 L50,8 L60,12 L70,6 L80,10 L90,4 L100,28 Z"
      fill="rgba(34,197,94,0.15)"
      stroke="none"
    />
    <!-- The actual trend line -->
    <polyline
      class="sparkline__line"
      points="0,28 10,20 20,22 30,10 40,14 50,8 60,12 70,6 80,10 90,4"
      fill="none"
      stroke="#22c55e"
      stroke-width="1.5"
      stroke-linejoin="round"
      stroke-linecap="round"
    />
    <!-- Hover target dots — one per data point, transparent until hovered -->
    <circle class="sparkline__dot" cx="90" cy="4" r="3" fill="#22c55e" />
  </svg>
</span>

<!-- Tooltip — positioned absolutely by JS -->
<div class="sparkline-tooltip" role="tooltip" aria-live="polite" hidden>
  <span class="sparkline-tooltip__value">$189.23</span>
  <span class="sparkline-tooltip__label">Day 30</span>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --spark-color-up:   #22c55e;
  --spark-color-down: #ef4444;
  --spark-color-flat: #94a3b8;
  --spark-area-alpha: 0.15;
  --spark-stroke-width: 1.5px;
  --spark-tooltip-bg: #1e293b;
  --spark-tooltip-radius: 4px;
}

/* ─── Wrapper ─── */
.sparkline-wrapper {
  position: relative;   /* tooltip positions against this */
  display: inline-block;
  vertical-align: middle;
  line-height: 0;       /* prevent extra space below SVG */
}

/* ─── SVG ─── */
.sparkline {
  display: block;
  overflow: visible;    /* dots at the edge must not be clipped */
}

/* Trend color applied by JS via data attribute */
.sparkline-wrapper[data-trend="up"]   .sparkline__line { stroke: var(--spark-color-up);   }
.sparkline-wrapper[data-trend="down"] .sparkline__line { stroke: var(--spark-color-down); }
.sparkline-wrapper[data-trend="flat"] .sparkline__line { stroke: var(--spark-color-flat); }

.sparkline-wrapper[data-trend="up"]   .sparkline__area { fill: color-mix(in srgb, var(--spark-color-up)   20%, transparent); }
.sparkline-wrapper[data-trend="down"] .sparkline__area { fill: color-mix(in srgb, var(--spark-color-down) 20%, transparent); }
.sparkline-wrapper[data-trend="flat"] .sparkline__area { fill: color-mix(in srgb, var(--spark-color-flat) 20%, transparent); }

/* ─── Hover dot ─── */
.sparkline__dot {
  opacity: 0;
  transition: opacity 0.1s;
  pointer-events: none;   /* hit-testing done on invisible rects, not the dot itself */
}

/* ─── Tooltip ─── */
.sparkline-tooltip {
  position: absolute;
  bottom: calc(100% + 6px);   /* default: above the sparkline */
  left: 50%;
  transform: translateX(-50%);
  background: var(--spark-tooltip-bg);
  color: #fff;
  font-size: 0.75rem;
  line-height: 1.3;
  padding: 4px 8px;
  border-radius: var(--spark-tooltip-radius);
  white-space: nowrap;
  pointer-events: none;
  z-index: 10;
}

.sparkline-tooltip[hidden] {
  display: none;
}

/* Tooltip arrow */
.sparkline-tooltip::after {
  content: '';
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  border: 5px solid transparent;
  border-top-color: var(--spark-tooltip-bg);
}

/* ─── Reduced motion: disable draw-on animation ─── */
@media (prefers-reduced-motion: reduce) {
  .sparkline__line {
    stroke-dasharray: none !important;
    stroke-dashoffset: none !important;
    animation: none !important;
  }
}
```

**What CSS owns:** color theming via `data-trend` attribute, tooltip bubble appearance, dot fade transition, reduced-motion override.

**What CSS cannot own:** computing the `points` attribute, mapping data to pixel coordinates, identifying the nearest hovered point, setting `left` on the tooltip to follow the cursor.

---

## React Implementation

### Coordinate Utilities

```tsx
// sparkline-utils.ts

export interface SparkPoint {
  x: number;
  y: number;
  value: number;
  index: number;
}

export type Trend = 'up' | 'down' | 'flat';

const PADDING = 3;   // px — prevents dots from touching the SVG edge

/**
 * Maps a data array to SVG coordinate points within a given viewBox.
 * All computation is pure — no DOM access, fully testable.
 */
export function toSparkPoints(
  data: number[],
  viewWidth: number,
  viewHeight: number
): SparkPoint[] {
  if (data.length < 2) return [];

  const min = Math.min(...data);
  const max = Math.max(...data);
  const range = max - min || 1;   // avoid division by zero for flat data

  const usableW = viewWidth  - PADDING * 2;
  const usableH = viewHeight - PADDING * 2;

  return data.map((value, index) => ({
    value,
    index,
    x: PADDING + (index / (data.length - 1)) * usableW,
    // SVG y-axis is inverted: higher value = lower y number
    y: PADDING + (1 - (value - min) / range) * usableH,
  }));
}

/** Converts SparkPoint array to SVG polyline points string */
export function toPolylinePoints(points: SparkPoint[]): string {
  return points.map(p => `${p.x.toFixed(2)},${p.y.toFixed(2)}`).join(' ');
}

/** Builds a closed SVG path for the area fill under the line */
export function toAreaPath(
  points: SparkPoint[],
  viewHeight: number
): string {
  if (points.length < 2) return '';
  const baseline = viewHeight - PADDING;
  const linePoints = points.map(p => `${p.x.toFixed(2)},${p.y.toFixed(2)}`).join(' L ');
  const first = points[0];
  const last  = points[points.length - 1];
  // Move to bottom-left, line up to first point, trace the line, drop back down
  return `M ${first.x.toFixed(2)},${baseline} L ${linePoints} L ${last.x.toFixed(2)},${baseline} Z`;
}

/** Determines trend from first to last value */
export function computeTrend(data: number[], threshold = 0.01): Trend {
  if (data.length < 2) return 'flat';
  const delta = (data[data.length - 1] - data[0]) / (Math.abs(data[0]) || 1);
  if (delta >  threshold) return 'up';
  if (delta < -threshold) return 'down';
  return 'flat';
}

/** Returns the point nearest to a given SVG x coordinate */
export function nearestPoint(points: SparkPoint[], svgX: number): SparkPoint | null {
  if (!points.length) return null;
  return points.reduce((nearest, p) =>
    Math.abs(p.x - svgX) < Math.abs(nearest.x - svgX) ? p : nearest
  );
}
```

### The Headless Hook

```tsx
// useSparkline.ts
import { useState, useCallback, useRef, useMemo } from 'react';
import {
  toSparkPoints, toPolylinePoints, toAreaPath,
  computeTrend, nearestPoint,
  type SparkPoint, type Trend
} from './sparkline-utils';

interface UseSparklineOptions {
  data: number[];
  viewWidth?: number;
  viewHeight?: number;
  formatValue?: (value: number) => string;
  formatLabel?: (index: number) => string;
}

interface TooltipState {
  visible: boolean;
  value: string;
  label: string;
  x: number;      // pixel offset relative to the wrapper, for CSS left
}

export interface UseSparklineReturn {
  points: SparkPoint[];
  polylinePoints: string;
  areaPath: string;
  trend: Trend;
  activePoint: SparkPoint | null;
  tooltip: TooltipState;
  svgProps: React.SVGProps<SVGSVGElement>;
  wrapperProps: React.HTMLAttributes<HTMLSpanElement>;
}

export function useSparkline({
  data,
  viewWidth  = 100,
  viewHeight = 32,
  formatValue = v => v.toFixed(2),
  formatLabel = i => `Point ${i + 1}`,
}: UseSparklineOptions): UseSparklineReturn {
  const [activePoint, setActivePoint] = useState<SparkPoint | null>(null);
  const svgRef = useRef<SVGSVGElement | null>(null);

  const points        = useMemo(() => toSparkPoints(data, viewWidth, viewHeight), [data, viewWidth, viewHeight]);
  const polylinePoints = useMemo(() => toPolylinePoints(points), [points]);
  const areaPath      = useMemo(() => toAreaPath(points, viewHeight), [points, viewHeight]);
  const trend         = useMemo(() => computeTrend(data), [data]);

  // Convert a mouse event's clientX to SVG viewBox x coordinate
  const clientXToSvgX = useCallback((clientX: number): number => {
    if (!svgRef.current) return 0;
    const rect  = svgRef.current.getBoundingClientRect();
    const ratio = viewWidth / rect.width;
    return (clientX - rect.left) * ratio;
  }, [viewWidth]);

  const handleMouseMove = useCallback((e: React.MouseEvent<SVGSVGElement>) => {
    const svgX   = clientXToSvgX(e.clientX);
    const nearest = nearestPoint(points, svgX);
    setActivePoint(nearest);
  }, [points, clientXToSvgX]);

  const handleMouseLeave = useCallback(() => {
    setActivePoint(null);
  }, []);

  // Compute tooltip left offset in rendered pixels (not SVG units)
  const tooltipX = useMemo((): number => {
    if (!activePoint || !svgRef.current) return 0;
    const rect  = svgRef.current.getBoundingClientRect();
    return rect.left + (activePoint.x / viewWidth) * rect.width;
  }, [activePoint, viewWidth]);

  const tooltip: TooltipState = {
    visible: activePoint !== null,
    value:   activePoint ? formatValue(activePoint.value) : '',
    label:   activePoint ? formatLabel(activePoint.index) : '',
    x:       tooltipX,
  };

  return {
    points,
    polylinePoints,
    areaPath,
    trend,
    activePoint,
    tooltip,
    svgProps: {
      ref: svgRef as React.RefObject<SVGSVGElement>,
      viewBox: `0 0 ${viewWidth} ${viewHeight}`,
      preserveAspectRatio: 'none',
      onMouseMove: handleMouseMove,
      onMouseLeave: handleMouseLeave,
    },
    wrapperProps: {
      'data-trend': trend,
    } as React.HTMLAttributes<HTMLSpanElement>,
  };
}
```

### Sparkline Component

```tsx
// Sparkline.tsx
import { useRef, useEffect } from 'react';
import { useSparkline } from './useSparkline';

interface SparklineProps {
  data: number[];
  width?: number;
  height?: number;
  viewWidth?: number;
  viewHeight?: number;
  label?: string;
  formatValue?: (v: number) => string;
  formatLabel?: (i: number) => string;
  animate?: boolean;
}

export function Sparkline({
  data,
  width    = 100,
  height   = 32,
  viewWidth  = 100,
  viewHeight = 32,
  label    = 'Value trend',
  formatValue,
  formatLabel,
  animate  = true,
}: SparklineProps) {
  const lineRef = useRef<SVGPolylineElement>(null);

  const {
    points,
    polylinePoints,
    areaPath,
    trend,
    activePoint,
    tooltip,
    svgProps,
    wrapperProps,
  } = useSparkline({ data, viewWidth, viewHeight, formatValue, formatLabel });

  // Draw-on animation using stroke-dashoffset trick
  useEffect(() => {
    if (!animate || !lineRef.current) return;
    const line   = lineRef.current;
    const length = line.getTotalLength();
    line.style.strokeDasharray  = String(length);
    line.style.strokeDashoffset = String(length);
    // Force reflow so the initial state is painted before the transition starts
    void line.getBoundingClientRect();
    line.style.transition       = 'stroke-dashoffset 0.6s ease-out';
    line.style.strokeDashoffset = '0';
  }, [data, animate]);

  const strokeColor =
    trend === 'up'   ? 'var(--spark-color-up)'   :
    trend === 'down' ? 'var(--spark-color-down)' :
                       'var(--spark-color-flat)';

  return (
    <span className="sparkline-wrapper" style={{ position: 'relative' }} {...wrapperProps}>
      <svg
        className="sparkline"
        width={width}
        height={height}
        role="img"
        aria-label={label}
        {...svgProps}
      >
        {/* Area fill */}
        <path
          className="sparkline__area"
          d={areaPath}
          fill={strokeColor}
          fillOpacity={0.15}
          stroke="none"
        />

        {/* Trend line */}
        <polyline
          ref={lineRef}
          className="sparkline__line"
          points={polylinePoints}
          fill="none"
          stroke={strokeColor}
          strokeWidth={1.5}
          strokeLinejoin="round"
          strokeLinecap="round"
        />

        {/* Active dot — follows hovered point */}
        {activePoint && (
          <circle
            className="sparkline__dot"
            cx={activePoint.x}
            cy={activePoint.y}
            r={3}
            fill={strokeColor}
            style={{ opacity: 1 }}
          />
        )}
      </svg>

      {/* Tooltip */}
      {tooltip.visible && (
        <div
          className="sparkline-tooltip"
          role="tooltip"
          style={{
            position:  'absolute',
            left:      `${(activePoint!.x / viewWidth) * 100}%`,
            transform: 'translateX(-50%)',
            bottom:    'calc(100% + 6px)',
          }}
        >
          <strong>{tooltip.value}</strong>
          <br />
          <span style={{ opacity: 0.7 }}>{tooltip.label}</span>
        </div>
      )}
    </span>
  );
}
```

### Usage in a Data Table

```tsx
// StockRow.tsx — one row in a 50-row table, no chart library
interface StockRowProps {
  ticker:    string;
  price:     number;
  change:    number;
  history:   number[];   // last 30 closing prices
}

export function StockRow({ ticker, price, change, history }: StockRowProps) {
  const isPositive = change >= 0;
  return (
    <tr>
      <td>{ticker}</td>
      <td style={{ fontVariantNumeric: 'tabular-nums' }}>
        ${price.toFixed(2)}
      </td>
      <td style={{ color: isPositive ? 'var(--spark-color-up)' : 'var(--spark-color-down)' }}>
        {isPositive ? '+' : ''}{change.toFixed(2)}%
      </td>
      <td>
        <Sparkline
          data={history}
          width={80}
          height={28}
          label={`${ticker} 30-day price trend`}
          formatValue={v => `$${v.toFixed(2)}`}
          formatLabel={i => `Day ${i + 1}`}
        />
      </td>
    </tr>
  );
}
```

---

## Angular Implementation

### Utilities (shared, same pure functions)

```typescript
// sparkline.utils.ts — identical pure functions to the React side
// toSparkPoints, toPolylinePoints, toAreaPath, computeTrend, nearestPoint
// (same implementation — omitted for brevity, fully reusable)
export * from './sparkline-utils';
```

### Sparkline Component with Signals

```typescript
// sparkline.component.ts
import {
  Component, input, computed, signal,
  ElementRef, viewChild, effect,
  ChangeDetectionStrategy
} from '@angular/core';
import {
  toSparkPoints, toPolylinePoints, toAreaPath,
  computeTrend, nearestPoint,
  type SparkPoint, type Trend
} from './sparkline.utils';

interface TooltipState {
  visible: boolean;
  value:   string;
  label:   string;
  leftPct: number;   // percentage, for CSS left
}

@Component({
  selector: 'app-sparkline',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <span
      class="sparkline-wrapper"
      [attr.data-trend]="trend()"
      style="position: relative; display: inline-block; line-height: 0;"
    >
      <svg
        #svgEl
        class="sparkline"
        [attr.width]="width()"
        [attr.height]="height()"
        [attr.viewBox]="viewBox()"
        preserveAspectRatio="none"
        role="img"
        [attr.aria-label]="label()"
        (mousemove)="onMouseMove($event)"
        (mouseleave)="onMouseLeave()"
      >
        <!-- Area fill -->
        <path
          class="sparkline__area"
          [attr.d]="areaPath()"
          [attr.fill]="strokeColor()"
          fill-opacity="0.15"
          stroke="none"
        />

        <!-- Trend line -->
        <polyline
          #lineEl
          class="sparkline__line"
          [attr.points]="polylinePoints()"
          fill="none"
          [attr.stroke]="strokeColor()"
          stroke-width="1.5"
          stroke-linejoin="round"
          stroke-linecap="round"
        />

        <!-- Active hover dot -->
        @if (activePoint()) {
          <circle
            class="sparkline__dot"
            [attr.cx]="activePoint()!.x"
            [attr.cy]="activePoint()!.y"
            r="3"
            [attr.fill]="strokeColor()"
            style="opacity: 1"
          />
        }
      </svg>

      <!-- Tooltip -->
      @if (tooltip().visible) {
        <div
          class="sparkline-tooltip"
          role="tooltip"
          [style.left.%]="tooltip().leftPct"
          style="position: absolute; transform: translateX(-50%); bottom: calc(100% + 6px);"
        >
          <strong>{{ tooltip().value }}</strong><br />
          <span style="opacity: 0.7">{{ tooltip().label }}</span>
        </div>
      }
    </span>
  `,
})
export class SparklineComponent {
  // Inputs as signals (Angular 17+)
  data        = input.required<number[]>();
  width       = input<number>(100);
  height      = input<number>(32);
  viewWidth   = input<number>(100);
  viewHeight  = input<number>(32);
  label       = input<string>('Value trend');
  formatValue = input<(v: number) => string>(v => v.toFixed(2));
  formatLabel = input<(i: number) => string>(i => `Point ${i + 1}`);
  animate     = input<boolean>(true);

  private svgEl  = viewChild<ElementRef<SVGSVGElement>>('svgEl');
  private lineEl = viewChild<ElementRef<SVGPolylineElement>>('lineEl');

  // Derived signals — zero subscriptions needed
  points        = computed(() => toSparkPoints(this.data(), this.viewWidth(), this.viewHeight()));
  polylinePoints = computed(() => toPolylinePoints(this.points()));
  areaPath      = computed(() => toAreaPath(this.points(), this.viewHeight()));
  trend         = computed<Trend>(() => computeTrend(this.data()));
  viewBox       = computed(() => `0 0 ${this.viewWidth()} ${this.viewHeight()}`);

  strokeColor = computed(() => {
    switch (this.trend()) {
      case 'up':   return 'var(--spark-color-up)';
      case 'down': return 'var(--spark-color-down)';
      default:     return 'var(--spark-color-flat)';
    }
  });

  // Hover state
  activePoint = signal<SparkPoint | null>(null);

  tooltip = computed<TooltipState>(() => {
    const p = this.activePoint();
    if (!p) return { visible: false, value: '', label: '', leftPct: 0 };
    return {
      visible:  true,
      value:    this.formatValue()(p.value),
      label:    this.formatLabel()(p.index),
      leftPct:  (p.x / this.viewWidth()) * 100,
    };
  });

  constructor() {
    // Draw-on animation: triggers whenever data changes
    effect(() => {
      if (!this.animate()) return;
      const line = this.lineEl()?.nativeElement;
      if (!line) return;
      // effect re-runs after view update, so the element is already painted
      const length = line.getTotalLength();
      line.style.strokeDasharray  = String(length);
      line.style.strokeDashoffset = String(length);
      void line.getBoundingClientRect();
      line.style.transition       = 'stroke-dashoffset 0.6s ease-out';
      line.style.strokeDashoffset = '0';
    });
  }

  onMouseMove(event: MouseEvent) {
    const svg = this.svgEl()?.nativeElement;
    if (!svg) return;
    const rect  = svg.getBoundingClientRect();
    const ratio = this.viewWidth() / rect.width;
    const svgX  = (event.clientX - rect.left) * ratio;
    this.activePoint.set(nearestPoint(this.points(), svgX));
  }

  onMouseLeave() {
    this.activePoint.set(null);
  }
}
```

### Usage in an Angular Table

```typescript
// portfolio-table.component.ts
import { Component } from '@angular/core';
import { SparklineComponent } from './sparkline.component';

interface Holding {
  ticker:  string;
  price:   number;
  change:  number;
  history: number[];
}

@Component({
  selector: 'app-portfolio-table',
  standalone: true,
  imports: [SparklineComponent],
  template: `
    <table>
      <thead>
        <tr>
          <th>Ticker</th><th>Price</th><th>Change</th><th>30d Trend</th>
        </tr>
      </thead>
      <tbody>
        @for (holding of holdings; track holding.ticker) {
          <tr>
            <td>{{ holding.ticker }}</td>
            <td>{{ holding.price | currency }}</td>
            <td [style.color]="holding.change >= 0 ? 'var(--spark-color-up)' : 'var(--spark-color-down)'">
              {{ holding.change >= 0 ? '+' : '' }}{{ holding.change.toFixed(2) }}%
            </td>
            <td>
              <app-sparkline
                [data]="holding.history"
                [width]="80"
                [height]="28"
                [label]="holding.ticker + ' 30-day trend'"
                [formatValue]="dollarFormat"
                [formatLabel]="dayLabel"
              />
            </td>
          </tr>
        }
      </tbody>
    </table>
  `,
})
export class PortfolioTableComponent {
  holdings: Holding[] = [/* injected from service */];

  dollarFormat = (v: number) => `$${v.toFixed(2)}`;
  dayLabel     = (i: number) => `Day ${i + 1}`;
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Decorative sparkline hidden from screen readers | `aria-hidden="true"` on the wrapper when the adjacent text already conveys the trend |
| Informative sparkline described to screen readers | `role="img"` + `aria-label` with a meaningful description on the `<svg>` element |
| Tooltip content readable by assistive technology | `role="tooltip"` on the tooltip div; hover interaction via `aria-describedby` linking the SVG to the tooltip id |
| Keyboard users can access data values | Provide a `<details>/<summary>` or visually-hidden `<table>` with the raw data as a progressive disclosure alternative |
| Tooltip visible text meets 4.5:1 contrast ratio | Dark background token `--spark-tooltip-bg: #1e293b` against white text |
| Color is not the only encoding for trend direction | Trend also encoded as text label in `aria-label` ("upward trend", "downward trend") |
| Reduced motion respected | `@media (prefers-reduced-motion: reduce)` disables the stroke-dashoffset animation |
| Focus visible on interactive elements | SVG itself is not focusable; interactive version wraps in a `<button>` or uses `tabIndex="0"` with keyboard event handlers |

---

## Production Pitfalls

**1. `preserveAspectRatio="none"` distorts dot radius and stroke width**
When the SVG scales non-uniformly (width and height scale by different ratios), a `stroke-width` of 1.5 viewBox units will look thicker on one axis than the other, and a circle `r="3"` will become an ellipse. Fix: use `vector-effect="non-scaling-stroke"` on the `<polyline>` and `<circle>` elements. This attribute instructs the renderer to apply the stroke in screen pixels, not viewBox units.

**2. `getTotalLength()` returns 0 on first render**
In React, calling `getTotalLength()` in a `useEffect` with an empty dependency array can execute before the browser has painted the SVG. The polyline `points` attribute is set, but the layout pass has not yet computed the path length. Fix: use a `useLayoutEffect` instead of `useEffect` for the animation setup, which fires synchronously after DOM mutations but before the browser paints.

**3. Tooltip overflows the viewport at edge data points**
When the hovered point is the first or last in the series, centering the tooltip on its x coordinate causes it to overflow the left or right viewport edge. Fix: clamp the tooltip's `left` value: `Math.max(tooltipHalfWidth, Math.min(leftPx, containerWidth - tooltipHalfWidth))`. Measure the tooltip's `offsetWidth` after render using a ref.

**4. 50 sparklines in a table cause layout thrashing on hover**
If each `mousemove` triggers a React `setState` or Angular `signal.set`, and each re-render forces the parent table to reconcile, the page drops frames. Fix: use `pointer-events: none` on the SVG and instead add a single delegated `mousemove` listener on the `<table>` element, identifying which sparkline is active by `closest('[data-sparkline-id]')`.

**5. Flat data (all identical values) renders as a horizontal line at the bottom**
When `min === max`, the range is zero and the division `(value - min) / range` produces `NaN`. The SVG renders nothing. Fix: guard with `const range = max - min || 1` so flat data maps to the vertical midpoint. Also add a CSS min-height to the wrapper so the empty SVG does not collapse the table row.

**6. The draw-on animation replays on every re-render**
If the parent component re-renders for unrelated state changes (e.g., a sibling counter), the `useEffect` with `[data]` dependency fires again only if `data` reference changes. But if the parent creates a new array literal on every render (`history={[1,2,3]}`), the animation replays continuously. Fix: memoize the data array at the call site with `useMemo` in React or define it as a signal-derived value in Angular so referential equality is preserved.

---

## Interview Angle

**Q: "How would you build a sparkline chart for a data table with 200 rows, without a chart library?"**

A strong answer has four components. First, explain the coordinate transform: map the data domain `[min, max]` to the SVG viewBox pixel range `[padding, viewHeight - padding]`, noting that SVG's y-axis is inverted. Express this as a pure function so it is testable and framework-agnostic. Second, explain `preserveAspectRatio="none"` — the viewBox defines an internal coordinate system, and the `width`/`height` attributes (or CSS) stretch it to fit the container, making the sparkline fully responsive without JavaScript resize listeners. Third, address the 200-row performance constraint: a single SVG polyline per row is O(n) DOM nodes, far cheaper than any canvas or WebGL-based chart library's initialization overhead. The real risk is per-row `mousemove` listeners; solve this with event delegation on the table. Fourth, cover accessibility: purely decorative sparklines get `aria-hidden="true"` and the adjacent cell text already conveys the trend; informative standalone sparklines get `role="img"` and a descriptive `aria-label`.

**Follow-up: "What does `vector-effect='non-scaling-stroke'` solve, and when would you use it?"**
When `preserveAspectRatio="none"` scales the SVG's x and y axes by different ratios (e.g., the viewBox is 100x32 but the rendered size is 80x28), stroke widths and circle radii defined in viewBox units scale non-uniformly. A 1.5-unit stroke looks visually thicker along one axis. `vector-effect="non-scaling-stroke"` tells the browser to render the stroke in screen pixels regardless of the current transform matrix, so the line always appears the same visual weight. Use it any time you use `preserveAspectRatio="none"` with strokes or radii you want to remain visually consistent.
