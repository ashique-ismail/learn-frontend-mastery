# Recharts — Composable React Charts

## Philosophy

Recharts is a charting library built on top of D3 and React's SVG rendering. Its core idea: **every chart is a composition of declarative React components**. You don't call D3 imperatively — instead you nest `<XAxis>`, `<Tooltip>`, `<Line>` inside a `<LineChart>` and the library wires up scales, domains, and event handling for you.

Key design choices:
- Components map 1-to-1 with chart concepts (axes, grids, tooltips, series)
- Data flows through a top-level chart container via `dataKey` props
- SVG-based (DOM-accessible, CSS-animatable, but not Canvas)
- D3 is used internally for scales and math — you never call D3 yourself

```bash
npm install recharts
```

---

## Data Format

All Recharts charts consume an array of plain objects. Each object is one data point. You reference keys via `dataKey`.

```ts
const data = [
  { month: 'Jan', revenue: 4200, expenses: 3100, profit: 1100 },
  { month: 'Feb', revenue: 5800, expenses: 3800, profit: 2000 },
  { month: 'Mar', revenue: 4900, expenses: 2900, profit: 2000 },
  { month: 'Apr', revenue: 6200, expenses: 4100, profit: 2100 },
  { month: 'May', revenue: 7100, expenses: 4400, profit: 2700 },
  { month: 'Jun', revenue: 6800, expenses: 4200, profit: 2600 },
];
```

- `dataKey="month"` on XAxis → categories on the x-axis
- `dataKey="revenue"` on a `<Line>` or `<Bar>` → which numeric key drives that series
- Recharts auto-infers the y-axis domain from all series values unless you override it

---

## LineChart

```tsx
import {
  LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend,
  ResponsiveContainer,
} from 'recharts';

const data = [
  { month: 'Jan', revenue: 4200, expenses: 3100 },
  { month: 'Feb', revenue: 5800, expenses: 3800 },
  { month: 'Mar', revenue: 4900, expenses: 2900 },
  { month: 'Apr', revenue: 6200, expenses: 4100 },
  { month: 'May', revenue: 7100, expenses: 4400 },
];

export function RevenueLineChart() {
  return (
    <ResponsiveContainer width="100%" height={320}>
      <LineChart data={data} margin={{ top: 8, right: 24, bottom: 8, left: 0 }}>
        <CartesianGrid strokeDasharray="3 3" stroke="#e5e7eb" />
        <XAxis dataKey="month" tick={{ fontSize: 12 }} />
        <YAxis tickFormatter={(v) => `$${(v / 1000).toFixed(0)}k`} />
        <Tooltip formatter={(value: number) => [`$${value.toLocaleString()}`, '']} />
        <Legend />
        <Line
          type="monotone"
          dataKey="revenue"
          stroke="#3b82f6"
          strokeWidth={2}
          dot={{ r: 4 }}
          activeDot={{ r: 6 }}
        />
        <Line
          type="monotone"
          dataKey="expenses"
          stroke="#f97316"
          strokeWidth={2}
          strokeDasharray="5 5"
          dot={false}
        />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

Key `<Line>` props:
- `type` — curve interpolation: `"monotone"` (smooth), `"linear"`, `"step"`, `"stepAfter"`, `"basis"`, `"natural"`
- `dot` — `false` hides dots, or pass an object `{ r: 4, fill: '#fff' }` or a custom render function
- `activeDot` — dot on hover; `{ r: 8, strokeWidth: 2 }`
- `strokeDasharray` — CSS stroke dash pattern e.g. `"5 5"` for dashed lines
- `connectNulls` — draw line through null/missing data points when `true`

---

## BarChart

```tsx
import {
  BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend,
  ResponsiveContainer, Cell,
} from 'recharts';

const data = [
  { quarter: 'Q1', revenue: 14200, target: 13000 },
  { quarter: 'Q2', revenue: 18900, target: 17000 },
  { quarter: 'Q3', revenue: 16800, target: 18000 },
  { quarter: 'Q4', revenue: 22100, target: 20000 },
];

export function QuarterlyBarChart() {
  return (
    <ResponsiveContainer width="100%" height={320}>
      <BarChart data={data} margin={{ top: 8, right: 24, bottom: 8, left: 0 }}>
        <CartesianGrid strokeDasharray="3 3" vertical={false} />
        <XAxis dataKey="quarter" />
        <YAxis tickFormatter={(v) => `$${v / 1000}k`} />
        <Tooltip
          cursor={{ fill: '#f3f4f6' }}
          formatter={(value: number, name: string) => [
            `$${value.toLocaleString()}`,
            name.charAt(0).toUpperCase() + name.slice(1),
          ]}
        />
        <Legend />
        <Bar dataKey="revenue" fill="#3b82f6" radius={[4, 4, 0, 0]}>
          {data.map((entry) => (
            <Cell
              key={entry.quarter}
              fill={entry.revenue >= entry.target ? '#22c55e' : '#ef4444'}
            />
          ))}
        </Bar>
        <Bar dataKey="target" fill="#d1d5db" radius={[4, 4, 0, 0]} />
      </BarChart>
    </ResponsiveContainer>
  );
}
```

Key `<Bar>` props:
- `radius` — corner rounding `[topLeft, topRight, bottomRight, bottomLeft]`
- `stackId` — bars with the same `stackId` stack on top of each other
- `layout` — `"horizontal"` (default) or `"vertical"` bars (pair with `layout` on `BarChart`)
- `maxBarSize` — cap bar width in pixels

Grouped vs stacked:
```tsx
{/* Grouped — default, no stackId */}
<Bar dataKey="revenue" fill="#3b82f6" />
<Bar dataKey="expenses" fill="#f97316" />

{/* Stacked — same stackId */}
<Bar dataKey="revenue" stackId="a" fill="#3b82f6" />
<Bar dataKey="expenses" stackId="a" fill="#f97316" />
```

---

## AreaChart

```tsx
import {
  AreaChart, Area, XAxis, YAxis, CartesianGrid, Tooltip,
  ResponsiveContainer,
} from 'recharts';

const data = [
  { date: '2024-01', users: 1200 },
  { date: '2024-02', users: 1900 },
  { date: '2024-03', users: 2400 },
  { date: '2024-04', users: 2200 },
  { date: '2024-05', users: 3100 },
  { date: '2024-06', users: 3800 },
];

export function UserGrowthAreaChart() {
  return (
    <ResponsiveContainer width="100%" height={280}>
      <AreaChart data={data} margin={{ top: 8, right: 24, bottom: 8, left: 0 }}>
        <defs>
          {/* Gradient fill */}
          <linearGradient id="usersGradient" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%" stopColor="#3b82f6" stopOpacity={0.3} />
            <stop offset="95%" stopColor="#3b82f6" stopOpacity={0} />
          </linearGradient>
        </defs>
        <CartesianGrid strokeDasharray="3 3" stroke="#e5e7eb" />
        <XAxis dataKey="date" />
        <YAxis />
        <Tooltip />
        <Area
          type="monotone"
          dataKey="users"
          stroke="#3b82f6"
          strokeWidth={2}
          fill="url(#usersGradient)"
        />
      </AreaChart>
    </ResponsiveContainer>
  );
}
```

Stacked areas:
```tsx
<Area type="monotone" dataKey="revenue" stackId="1" stroke="#3b82f6" fill="#3b82f6" fillOpacity={0.6} />
<Area type="monotone" dataKey="expenses" stackId="1" stroke="#f97316" fill="#f97316" fillOpacity={0.6} />
```

---

## PieChart

```tsx
import {
  PieChart, Pie, Cell, Tooltip, Legend, ResponsiveContainer,
} from 'recharts';

const data = [
  { name: 'Direct', value: 4200 },
  { name: 'Organic', value: 3100 },
  { name: 'Referral', value: 1800 },
  { name: 'Social', value: 900 },
];

const COLORS = ['#3b82f6', '#22c55e', '#f97316', '#a855f7'];

const RADIAN = Math.PI / 180;

function renderCustomLabel({
  cx, cy, midAngle, innerRadius, outerRadius, percent,
}: any) {
  const radius = innerRadius + (outerRadius - innerRadius) * 0.5;
  const x = cx + radius * Math.cos(-midAngle * RADIAN);
  const y = cy + radius * Math.sin(-midAngle * RADIAN);
  return (
    <text x={x} y={y} fill="white" textAnchor="middle" dominantBaseline="central" fontSize={12}>
      {`${(percent * 100).toFixed(0)}%`}
    </text>
  );
}

export function TrafficPieChart() {
  return (
    <ResponsiveContainer width="100%" height={320}>
      <PieChart>
        <Pie
          data={data}
          cx="50%"
          cy="50%"
          outerRadius={120}
          innerRadius={60}       // makes it a donut chart
          dataKey="value"
          labelLine={false}
          label={renderCustomLabel}
        >
          {data.map((entry, index) => (
            <Cell key={`cell-${index}`} fill={COLORS[index % COLORS.length]} />
          ))}
        </Pie>
        <Tooltip formatter={(value: number) => [value.toLocaleString(), '']} />
        <Legend />
      </PieChart>
    </ResponsiveContainer>
  );
}
```

Key `<Pie>` props:
- `innerRadius` — `0` for full pie, positive number for donut
- `paddingAngle` — gap between slices in degrees e.g. `paddingAngle={3}`
- `startAngle` / `endAngle` — `startAngle={90} endAngle={-270}` starts from 12 o'clock
- `activeIndex` + `onMouseEnter` — programmatically highlight a slice

---

## ComposedChart

ComposedChart lets you mix Bar, Line, and Area in a single coordinate system. All series share the same x-axis domain.

```tsx
import {
  ComposedChart, Bar, Line, Area, XAxis, YAxis, CartesianGrid,
  Tooltip, Legend, ResponsiveContainer,
} from 'recharts';

const data = [
  { month: 'Jan', revenue: 4200, expenses: 3100, margin: 26 },
  { month: 'Feb', revenue: 5800, expenses: 3800, margin: 34 },
  { month: 'Mar', revenue: 4900, expenses: 2900, margin: 41 },
  { month: 'Apr', revenue: 6200, expenses: 4100, margin: 34 },
  { month: 'May', revenue: 7100, expenses: 4400, margin: 38 },
];

export function FinancialComposedChart() {
  return (
    <ResponsiveContainer width="100%" height={360}>
      <ComposedChart data={data} margin={{ top: 8, right: 24, bottom: 8, left: 0 }}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="month" />
        {/* Left axis for dollar values */}
        <YAxis yAxisId="left" tickFormatter={(v) => `$${v / 1000}k`} />
        {/* Right axis for percentage */}
        <YAxis yAxisId="right" orientation="right" tickFormatter={(v) => `${v}%`} />
        <Tooltip />
        <Legend />
        {/* Stacked bars on left axis */}
        <Bar yAxisId="left" dataKey="expenses" fill="#fca5a5" stackId="a" />
        <Bar yAxisId="left" dataKey="revenue" fill="#3b82f6" stackId="a" />
        {/* Margin % as line on right axis */}
        <Line yAxisId="right" type="monotone" dataKey="margin" stroke="#f97316" strokeWidth={2} dot={{ r: 4 }} />
      </ComposedChart>
    </ResponsiveContainer>
  );
}
```

Important: when using dual y-axes, **every** `<Bar>`, `<Line>`, and `<Area>` must declare `yAxisId` to avoid misplotted data.

---

## Essential Cartesian Components

### XAxis & YAxis

```tsx
<XAxis
  dataKey="month"
  type="category"          // "category" (default) | "number"
  tickLine={false}         // hide tick marks
  axisLine={{ stroke: '#e5e7eb' }}
  tick={{ fill: '#6b7280', fontSize: 12 }}
  tickFormatter={(v) => v.slice(0, 3)}   // "January" → "Jan"
  interval={0}             // show every tick, no auto-skip
  angle={-30}              // rotate labels
  textAnchor="end"
  height={48}              // extra room for rotated labels
/>

<YAxis
  width={64}               // override default width
  domain={[0, 'auto']}     // explicit domain; 'auto' lets recharts calculate
  allowDataOverflow={false}
  tickCount={6}
  tickFormatter={(v) => `${v}%`}
  label={{ value: 'Revenue ($)', angle: -90, position: 'insideLeft' }}
/>
```

For time-series data use `type="number"` with epoch milliseconds:
```tsx
<XAxis
  dataKey="timestamp"
  type="number"
  scale="time"
  domain={['auto', 'auto']}
  tickFormatter={(ms) => new Date(ms).toLocaleDateString('en-US', { month: 'short', day: 'numeric' })}
/>
```

### CartesianGrid

```tsx
<CartesianGrid
  strokeDasharray="3 3"    // "3 3" dashes, "0" solid, "" no dash
  stroke="#f0f0f0"
  horizontal={true}        // show horizontal lines
  vertical={false}         // hide vertical lines (common for bar charts)
/>
```

### Tooltip

```tsx
<Tooltip
  cursor={{ stroke: '#d1d5db', strokeWidth: 1 }}  // crosshair line
  contentStyle={{ borderRadius: 8, border: '1px solid #e5e7eb' }}
  labelStyle={{ fontWeight: 600 }}
  formatter={(value: number, name: string) => [
    `$${value.toLocaleString()}`,   // formatted value
    name.replace(/_/g, ' '),        // formatted series name
  ]}
  labelFormatter={(label) => `Month: ${label}`}
/>
```

### Legend

```tsx
<Legend
  verticalAlign="top"           // "top" | "middle" | "bottom"
  align="right"                 // "left" | "center" | "right"
  iconType="circle"             // "line" | "square" | "rect" | "circle" | "cross" | "diamond"
  iconSize={12}
  formatter={(value) => <span style={{ color: '#374151' }}>{value}</span>}
  wrapperStyle={{ paddingBottom: 16 }}
/>
```

---

## ResponsiveContainer

Recharts charts have a fixed pixel width by default. `ResponsiveContainer` measures its parent's dimensions and re-renders the chart at that size.

```tsx
{/* Standard responsive pattern */}
<div style={{ width: '100%' }}>
  <ResponsiveContainer width="100%" height={320}>
    <LineChart data={data}>
      {/* ... */}
    </LineChart>
  </ResponsiveContainer>
</div>
```

Rules:
- The parent element **must have a defined width** — `100%` needs an ancestor with pixels
- `height` is almost always a fixed number (not percentage), because height `100%` requires the parent to have explicit height
- Use `aspect` instead of `height` to maintain a ratio: `<ResponsiveContainer width="100%" aspect={16/9}>`
- `debounce={50}` prop throttles resize recalculations

---

## Custom Tooltip

Build a fully styled tooltip by passing a component to the `content` prop.

```tsx
import { TooltipProps } from 'recharts';
import { ValueType, NameType } from 'recharts/types/component/DefaultTooltipContent';

interface CustomTooltipProps extends TooltipProps<ValueType, NameType> {}

function CustomTooltip({ active, payload, label }: CustomTooltipProps) {
  if (!active || !payload || payload.length === 0) return null;

  return (
    <div
      style={{
        background: 'white',
        border: '1px solid #e5e7eb',
        borderRadius: 8,
        padding: '10px 14px',
        boxShadow: '0 4px 12px rgba(0,0,0,0.08)',
        minWidth: 160,
      }}
    >
      <p style={{ margin: '0 0 6px', fontWeight: 600, color: '#111827' }}>{label}</p>
      {payload.map((entry) => (
        <div
          key={entry.dataKey as string}
          style={{ display: 'flex', justifyContent: 'space-between', gap: 16, fontSize: 13 }}
        >
          <span style={{ color: entry.color }}>{entry.name}</span>
          <span style={{ fontWeight: 500, color: '#1f2937' }}>
            ${(entry.value as number).toLocaleString()}
          </span>
        </div>
      ))}
    </div>
  );
}

// Usage:
<Tooltip content={<CustomTooltip />} />
```

The `payload` array contains one entry per visible series. Each entry has:
- `entry.dataKey` — the series key
- `entry.name` — display name (defaults to dataKey)
- `entry.value` — numeric value for this data point
- `entry.color` — the series stroke/fill color
- `entry.payload` — the full original data object for this x-position

---

## Custom Shapes and Cells

### Per-bar colors with Cell

```tsx
import { BarChart, Bar, Cell } from 'recharts';

const data = [
  { region: 'North', growth: 12 },
  { region: 'South', growth: -3 },
  { region: 'East',  growth: 8 },
  { region: 'West',  growth: -7 },
];

<BarChart data={data}>
  <Bar dataKey="growth">
    {data.map((entry, index) => (
      <Cell
        key={`cell-${index}`}
        fill={entry.growth >= 0 ? '#22c55e' : '#ef4444'}
        fillOpacity={Math.abs(entry.growth) / 15 + 0.4}  // intensity by magnitude
      />
    ))}
  </Bar>
</BarChart>
```

### Custom dot shape on Line

```tsx
function CustomDot(props: any) {
  const { cx, cy, value } = props;
  if (value > 5000) {
    // Star for high values
    return <polygon points={`${cx},${cy-8} ${cx+3},${cy} ${cx+8},${cy} ${cx+4},${cy+5} ${cx+5},${cy+10} ${cx},${cy+7} ${cx-5},${cy+10} ${cx-4},${cy+5} ${cx-8},${cy} ${cx-3},${cy}`} fill="#f97316" />;
  }
  return <circle cx={cx} cy={cy} r={4} fill="#3b82f6" stroke="white" strokeWidth={1} />;
}

<Line dataKey="revenue" dot={<CustomDot />} activeDot={false} />
```

### Custom bar shape

```tsx
function RoundedBar(props: any) {
  const { x, y, width, height, fill } = props;
  const radius = 4;
  return (
    <path
      d={`M ${x},${y + height} L ${x},${y + radius} Q ${x},${y} ${x + radius},${y} L ${x + width - radius},${y} Q ${x + width},${y} ${x + width},${y + radius} L ${x + width},${y + height} Z`}
      fill={fill}
    />
  );
}

<Bar dataKey="revenue" shape={<RoundedBar />} />
```

---

## Reference Lines and Reference Areas

```tsx
import {
  LineChart, Line, XAxis, YAxis, Tooltip, ReferenceLine, ReferenceArea,
  ResponsiveContainer,
} from 'recharts';

<ResponsiveContainer width="100%" height={320}>
  <LineChart data={data}>
    <XAxis dataKey="month" />
    <YAxis />
    <Tooltip />

    {/* Horizontal threshold line */}
    <ReferenceLine y={5000} stroke="#ef4444" strokeDasharray="4 4" label={{ value: 'Target', position: 'right', fill: '#ef4444', fontSize: 12 }} />

    {/* Vertical event marker */}
    <ReferenceLine x="Apr" stroke="#6366f1" label={{ value: 'Launch', position: 'top', fill: '#6366f1', fontSize: 12 }} />

    {/* Shaded region — e.g. a recession period */}
    <ReferenceArea
      x1="Feb"
      x2="Mar"
      fill="#fef9c3"
      fillOpacity={0.6}
      stroke="#fbbf24"
      strokeOpacity={0.5}
      label={{ value: 'Dip', position: 'insideTop', fill: '#92400e', fontSize: 11 }}
    />

    <Line type="monotone" dataKey="revenue" stroke="#3b82f6" strokeWidth={2} />
  </LineChart>
</ResponsiveContainer>
```

`ReferenceArea` accepts `y1`/`y2` for horizontal bands (e.g. normal ranges in health charts).

---

## Animations

Recharts animates series on mount and on data change by default.

```tsx
<Line
  dataKey="revenue"
  isAnimationActive={true}         // true by default; set false to disable
  animationBegin={0}               // delay before animation starts (ms)
  animationDuration={1200}         // total animation duration (ms)
  animationEasing="ease-out"       // CSS easing: "ease", "ease-in", "ease-out", "linear", "spring"
/>

<Bar
  dataKey="revenue"
  isAnimationActive={true}
  animationDuration={800}
  animationEasing="ease-in-out"
/>
```

Disable animations for performance (large datasets) or during testing:
```tsx
// Disable globally by adding to every series component
isAnimationActive={process.env.NODE_ENV !== 'test'}
```

For animated data updates (streaming/live data), Recharts auto-transitions when `data` prop changes — no extra code needed. Keep the same array shape; changing `dataKey` names resets animation.

---

## Synchronised Charts

`syncId` links hover interactions across multiple independent charts. When you hover over one, the tooltip and crosshair appear at the same x-position in all charts sharing the same `syncId`.

```tsx
const data1 = [
  { month: 'Jan', revenue: 4200 },
  { month: 'Feb', revenue: 5800 },
  { month: 'Mar', revenue: 4900 },
];

const data2 = [
  { month: 'Jan', users: 820 },
  { month: 'Feb', users: 1140 },
  { month: 'Mar', users: 980 },
];

export function SyncedDashboard() {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 16 }}>
      <ResponsiveContainer width="100%" height={200}>
        <LineChart data={data1} syncId="dashboard">
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="month" />
          <YAxis />
          <Tooltip />
          <Line type="monotone" dataKey="revenue" stroke="#3b82f6" />
        </LineChart>
      </ResponsiveContainer>

      <ResponsiveContainer width="100%" height={200}>
        <BarChart data={data2} syncId="dashboard">
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="month" />
          <YAxis />
          <Tooltip />
          <Bar dataKey="users" fill="#22c55e" />
        </BarChart>
      </ResponsiveContainer>
    </div>
  );
}
```

`syncId` works across `LineChart`, `BarChart`, `AreaChart`, and `ComposedChart`. It requires all charts to share the same x-axis domain values.

---

## Performance: Large Datasets

SVG renders one DOM element per data point. For 10,000+ points this becomes slow.

### Strategy 1 — Data sampling

```tsx
import { useMemo } from 'react';

function sampleEveryN<T>(data: T[], n: number): T[] {
  return data.filter((_, i) => i % n === 0);
}

function LargeDatasetChart({ rawData }: { rawData: DataPoint[] }) {
  // Sample on the client before passing to Recharts
  const displayData = useMemo(
    () => (rawData.length > 500 ? sampleEveryN(rawData, Math.ceil(rawData.length / 500)) : rawData),
    [rawData]
  );

  return (
    <ResponsiveContainer width="100%" height={320}>
      <LineChart data={displayData}>
        <Line type="monotone" dataKey="value" dot={false} isAnimationActive={false} />
        <XAxis dataKey="timestamp" />
        <YAxis />
        <Tooltip />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

### Strategy 2 — LTTB (Largest Triangle Three Buckets)

For time-series with visual fidelity requirements, use the LTTB downsampling algorithm which preserves peaks and troughs:

```bash
npm install d3-array
```

```ts
import { leastIndex } from 'd3-array';

// Or use a dedicated package:
// npm install @d3fc/d3fc-sample
import { largestTriangleThreeBucket } from '@d3fc/d3fc-sample';

function downsample(data: { x: number; y: number }[], targetCount: number) {
  const sampler = largestTriangleThreeBucket();
  sampler.x((d) => d.x).y((d) => d.y).bucketSize(data.length / targetCount);
  return sampler(data);
}
```

### Strategy 3 — Memoize data transformations

```tsx
const chartData = useMemo(() => {
  return rawData
    .filter((d) => d.status === 'active')
    .map((d) => ({
      date: d.timestamp,
      value: d.metricA / d.metricB,   // avoid recomputing every render
    }));
}, [rawData]);
```

### Strategy 4 — Disable dots and animations

```tsx
<Line
  dot={false}                  // removes per-point SVG circle elements
  isAnimationActive={false}    // skips animation on large re-renders
/>
```

For truly massive datasets (100k+ points), switch to a Canvas-based library: Visx with `@visx/canvas`, or ECharts.

---

## Comparison: Recharts vs Visx vs Nivo

| Criterion | Recharts | Visx | Nivo |
|---|---|---|---|
| API level | High-level components | Low-level D3 primitives | High-level config objects |
| Customization | Moderate — extend via props and render functions | Total — every pixel via code | Config-based — limited beyond what props expose |
| Bundle size | ~90 KB gzipped | Tree-shakeable, ~5–20 KB per package | Tree-shakeable, ~30–80 KB per chart |
| Setup speed | Fast — works in minutes | Slow — requires D3 knowledge | Fast — plug-and-play |
| Animations | Built-in, simple | Manual (Framer Motion / react-spring) | Built-in via react-spring |
| SSR support | Partial (layout issues) | Works with care | Full SSR support (Nivo explicitly supports it) |
| Chart types | Standard (Line, Bar, Area, Pie, Radar, Scatter) | Any — you compose it | Extensive (Sankey, Chord, Heatmap, Calendar, Treemap, etc.) |
| D3 knowledge needed | None | Yes — scales, paths, layouts | Minimal |
| Accessibility | Manual `aria-*` | Manual | Some built-in |
| Best for | Standard charts, quick delivery | Custom / unusual charts, design systems | Polished charts, uncommon types, SSR |

### When to choose Recharts

- You need standard chart types (line, bar, area, pie, scatter, radar) and good defaults
- Team velocity matters — Recharts is the fastest to go from zero to a working chart
- Charts are moderately complex (custom tooltips, dual axes, reference lines) but not pixel-perfect bespoke
- The app renders in the browser (not SSR-critical)
- You don't need exotic chart types like Sankey or Chord diagrams

### When to choose Visx instead

- Chart type doesn't exist in any library (bespoke, domain-specific)
- Building a shared chart library or design system that must match exact brand specifications
- Need full control over SVG output, animations, and interactions
- Team has D3 experience

### When to choose Nivo instead

- Need unusual chart types: Chord, Sankey, Calendar heatmap, Voronoi, Waffle
- SSR is required (Next.js App Router, etc.) — Nivo handles this cleanly
- Want a polished, consistent look with minimal configuration

---

## Common Interview Questions

**Q: How does Recharts know the domain/scale of each axis?**
Recharts iterates over the `data` array and all `dataKey` values mapped to that axis to compute `min`/`max`. You can override with `domain={[0, 100]}` or `domain={['auto', 'dataMax + 20']}` using string expressions. For categorical XAxis it derives categories in array order.

**Q: How would you add a second y-axis?**
Add two `<YAxis>` components with distinct `yAxisId` props (`"left"` and `"right"`), set `orientation="right"` on the second one, then attach each series component with the matching `yAxisId`. See the ComposedChart example above.

**Q: Why does my chart not fill its container?**
`ResponsiveContainer` needs the parent element to have a defined pixel width. A `div` with `width: 100%` inside a flex parent works. Also ensure `height` is a fixed number — percentage heights require the parent to have an explicit height set.

**Q: How do you handle click events on chart elements?**
Every series component (`Bar`, `Line`, `Pie`, etc.) accepts `onClick`. The callback receives the data point object and event:
```tsx
<Bar dataKey="revenue" onClick={(data, index) => console.log(data.month, data.revenue)} />
<Pie dataKey="value" onClick={(data, index) => setSelected(data)} />
```

**Q: How do you render a chart server-side (SSR)?**
Recharts relies on `window` and DOM measurements for `ResponsiveContainer`, which breaks SSR. Solutions:
1. Use `next/dynamic` with `ssr: false` to client-render the chart component
2. Provide fixed `width` and `height` directly on the chart (skip `ResponsiveContainer`) — this works in SSR but isn't responsive
3. Switch to Nivo which has full SSR support

**Q: How would you animate a chart when new data arrives?**
Simply update the `data` prop — Recharts auto-animates the transition using its built-in animation system. To control duration: set `animationDuration` on each series component. To trigger a fresh enter animation, change the chart's `key` prop to force unmount/remount.

**Q: What is `syncId` and when is it useful?**
`syncId` is a string that links hover state across multiple charts. When set to the same value on two or more charts, moving the cursor over one shows the tooltip and crosshair indicator on all of them at the same x-position. This is useful in dashboards where related metrics are displayed in separate charts stacked vertically.
