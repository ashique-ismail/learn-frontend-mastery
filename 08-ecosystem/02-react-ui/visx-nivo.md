# Visx & Nivo — Advanced Chart Libraries

## Overview

Both are more powerful alternatives to Recharts for complex, customized data visualizations.

- **Visx** (Airbnb) — low-level, composable D3-backed primitives. Full control, more code.
- **Nivo** — higher-level, opinionated charts with good defaults. Less code, less flexibility.

---

## Visx — Low-Level D3 Primitives

Visx provides D3 functionality as React components. You compose primitives to build exactly the chart you need.

```bash
npm install @visx/group @visx/shape @visx/scale @visx/axis @visx/tooltip
```

```tsx
import { Group } from '@visx/group';
import { Bar } from '@visx/shape';
import { scaleBand, scaleLinear } from '@visx/scale';
import { AxisBottom, AxisLeft } from '@visx/axis';
import { useTooltip, TooltipWithBounds } from '@visx/tooltip';

const data = [
  { month: 'Jan', revenue: 4200 },
  { month: 'Feb', revenue: 5800 },
  { month: 'Mar', revenue: 4900 },
];

const width = 600;
const height = 400;
const margin = { top: 20, right: 20, bottom: 40, left: 60 };

const innerWidth = width - margin.left - margin.right;
const innerHeight = height - margin.top - margin.bottom;

function RevenueChart() {
  const { tooltipData, tooltipLeft, tooltipTop, showTooltip, hideTooltip } = useTooltip<typeof data[0]>();

  const xScale = scaleBand({
    domain: data.map(d => d.month),
    range: [0, innerWidth],
    padding: 0.3,
  });

  const yScale = scaleLinear({
    domain: [0, Math.max(...data.map(d => d.revenue))],
    range: [innerHeight, 0],
    nice: true,
  });

  return (
    <div style={{ position: 'relative' }}>
      <svg width={width} height={height}>
        <Group left={margin.left} top={margin.top}>
          {data.map(d => (
            <Bar
              key={d.month}
              x={xScale(d.month)!}
              y={yScale(d.revenue)}
              width={xScale.bandwidth()}
              height={innerHeight - yScale(d.revenue)}
              fill="#3b82f6"
              onMouseMove={event => {
                showTooltip({
                  tooltipData: d,
                  tooltipLeft: xScale(d.month)! + margin.left,
                  tooltipTop: yScale(d.revenue) + margin.top,
                });
              }}
              onMouseLeave={hideTooltip}
            />
          ))}
          <AxisBottom top={innerHeight} scale={xScale} />
          <AxisLeft scale={yScale} tickFormat={v => `$${Number(v) / 1000}k`} />
        </Group>
      </svg>

      {tooltipData && (
        <TooltipWithBounds left={tooltipLeft} top={tooltipTop}>
          <strong>{tooltipData.month}</strong>: ${tooltipData.revenue.toLocaleString()}
        </TooltipWithBounds>
      )}
    </div>
  );
}
```

**When to use Visx:**
- Custom chart types not covered by higher-level libraries
- Fine-grained control over every element (animations, interactions)
- Building a chart library or design system
- Need to combine multiple chart types in one SVG

---

## Nivo — Opinionated High-Level Charts

```bash
npm install @nivo/bar @nivo/line @nivo/pie
```

```tsx
import { ResponsiveBar } from '@nivo/bar';
import { ResponsiveLine } from '@nivo/line';

const data = [
  { month: 'Jan', revenue: 4200, expenses: 3100 },
  { month: 'Feb', revenue: 5800, expenses: 3800 },
  { month: 'Mar', revenue: 4900, expenses: 2900 },
];

function RevenueChart() {
  return (
    <div style={{ height: 400 }}>
      <ResponsiveBar
        data={data}
        keys={['revenue', 'expenses']}
        indexBy="month"
        margin={{ top: 50, right: 130, bottom: 50, left: 60 }}
        padding={0.3}
        groupMode="grouped"
        colors={{ scheme: 'nivo' }}
        borderColor={{ from: 'color', modifiers: [['darker', 1.6]] }}
        axisBottom={{ tickSize: 5, tickPadding: 5, legend: 'Month', legendPosition: 'middle' }}
        axisLeft={{ tickSize: 5, tickPadding: 5, format: v => `$${v / 1000}k`, legend: 'Amount' }}
        legends={[{
          dataFrom: 'keys',
          anchor: 'bottom-right',
          direction: 'column',
          itemWidth: 100,
          itemHeight: 20,
        }]}
        tooltip={({ id, value, color }) => (
          <div style={{ background: 'white', padding: '8px', border: `1px solid ${color}` }}>
            <strong>{String(id)}</strong>: ${value.toLocaleString()}
          </div>
        )}
      />
    </div>
  );
}
```

**Available chart types:** Bar, Line, Pie, Scatterplot, Heatmap, Chord, Treemap, Calendar, Funnel, Sankey, Radar, BumpChart, SwarmPlot, and more.

---

## Comparison

| | Recharts | Visx | Nivo |
|---|---|---|---|
| API level | High-level | Low-level (primitives) | High-level |
| Customization | Moderate | Total control | Config-based |
| Bundle size | ~90 KB | Per-package (~5-20 KB each) | Per-package (~30-80 KB each) |
| Animations | Basic | Manual (use framer-motion/spring) | Built-in (react-spring) |
| SVG vs Canvas | SVG | SVG | SVG + Canvas option |
| Learning curve | Low | High (requires D3 knowledge) | Low-medium |
| Best for | Standard charts fast | Custom/unusual charts | Polished charts with less code |

---

## Common Interview Questions

**Q: When would you choose Visx over Nivo or Recharts?**
When you need a chart type that doesn't exist in higher-level libraries, when you need pixel-perfect custom interactions, or when you're building a data visualization platform where every chart is unique. Visx requires knowing D3 concepts (scales, axes).

**Q: What's the performance difference between SVG and Canvas charts?**
SVG renders individual DOM elements (queryable, accessible, animatable with CSS). Canvas renders to a bitmap (no DOM, faster for thousands of data points). For <1000 data points, SVG is fine and more accessible. For >10,000 points, Canvas (or WebGL) is necessary.

**Q: How do you make charts accessible?**
Add `aria-label` to the SVG element describing the chart, use `<title>` and `<desc>` inside SVG, provide a data table as an alternative to the visual chart, ensure keyboard navigation for interactive elements.
