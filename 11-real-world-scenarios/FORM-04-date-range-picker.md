# Date Range Picker

## The Idea

**In plain English:** A date range picker is a UI that lets users select a start date and an end date from a visual calendar grid. The two dates share state — choosing the second date is meaningless without knowing the first — which is what makes this a compound component problem.

**Real-world analogy:** Think of booking a hotel room on a travel site.

- You first click "Check-in" — the calendar highlights today and future dates as selectable
- You click a date, say June 15 — it becomes the **start anchor**
- As you hover over future dates, the calendar **shades the range** between June 15 and wherever your cursor is
- You click June 20 — the **end anchor** is set, the range June 15–20 is confirmed
- The two input fields ("Check-in" / "Check-out") are visually separate but share the same calendar and the same selection state
- Clicking a date that is before June 15 **resets** the start date rather than creating an invalid range
- The calendar also greys out **unavailable dates** (rooms already booked)

The key insight: the two date inputs are siblings, but the calendar is the single source of truth. This is the compound component pattern — multiple UI pieces, one shared state.

---

## Learning Objectives

- Implement the compound component pattern with shared context state
- Build range selection logic: anchor → hover preview → confirm
- Handle keyboard navigation within a calendar grid (arrow keys, Enter, Escape)
- Make the date grid accessible per the ARIA grid pattern
- Avoid invalid ranges and handle locale-aware date formatting

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Render a calendar grid for any month | ❌ | Requires computing day offsets from JS Date |
| Track start/end anchor dates | ❌ | State, not style |
| Shade the in-range dates on hover | ❌ | CSS can't compare dates |
| Disable dates before today or unavailable dates | ❌ | Logic depends on data |
| Navigate months (prev/next) | ❌ | Month offset is integer state |
| Announce selected range to screen readers | ❌ | aria-label updates require JS |

---

## HTML & CSS Foundation

```html
<!-- Trigger inputs -->
<div class="date-range-wrapper">
  <div class="date-input-group">
    <label for="start-date">Check-in</label>
    <input
      id="start-date"
      type="text"
      readonly
      aria-haspopup="dialog"
      aria-expanded="false"
      placeholder="DD / MM / YYYY"
      class="date-input"
    />
  </div>
  <span class="date-range-sep" aria-hidden="true">→</span>
  <div class="date-input-group">
    <label for="end-date">Check-out</label>
    <input
      id="end-date"
      type="text"
      readonly
      aria-haspopup="dialog"
      aria-expanded="false"
      placeholder="DD / MM / YYYY"
      class="date-input"
    />
  </div>
</div>

<!-- Calendar panel (opened as a dialog) -->
<div
  role="dialog"
  aria-modal="true"
  aria-label="Select date range"
  class="calendar-panel"
  hidden
>
  <div class="calendar-header">
    <button aria-label="Previous month" class="cal-nav-btn">‹</button>
    <span class="cal-month-label" aria-live="polite">June 2026</span>
    <button aria-label="Next month" class="cal-nav-btn">›</button>
  </div>

  <!-- Grid of day cells -->
  <table role="grid" aria-label="June 2026" class="cal-grid">
    <thead>
      <tr>
        <th scope="col" abbr="Monday">Mo</th>
        <!-- … -->
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>
          <button
            class="cal-day"
            aria-label="Monday, June 1, 2026"
            aria-pressed="false"
            aria-disabled="false"
          >1</button>
        </td>
        <!-- … -->
      </tr>
    </tbody>
  </table>
</div>
```

```css
:root {
  --cal-accent: #1a73e8;
  --cal-range-bg: #e8f0fe;
  --cal-today: #fbbc04;
  --cal-disabled: #ccc;
  --cal-cell-size: 40px;
}

.date-range-wrapper {
  display: inline-flex;
  align-items: center;
  gap: 12px;
}

.date-input {
  padding: 10px 14px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 1rem;
  cursor: pointer;
  min-width: 160px;
  caret-color: transparent;
}

.date-input:focus { outline: 2px solid var(--cal-accent); outline-offset: 2px; }

.calendar-panel {
  position: absolute;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 10px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.14);
  padding: 16px;
  z-index: 300;
}

.calendar-panel[hidden] { display: none; }

.calendar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.cal-nav-btn {
  width: 36px; height: 36px;
  border-radius: 50%;
  border: none;
  background: none;
  font-size: 1.2rem;
  cursor: pointer;
}
.cal-nav-btn:hover { background: #f5f5f5; }

.cal-grid { border-collapse: collapse; }
.cal-grid th { font-size: 0.8rem; color: #888; padding: 4px 0; text-align: center; }

.cal-day {
  width: var(--cal-cell-size);
  height: var(--cal-cell-size);
  border: none;
  border-radius: 50%;
  background: none;
  font-size: 0.9rem;
  cursor: pointer;
  position: relative;
}

.cal-day:hover:not([aria-disabled="true"]) { background: #f0f0f0; }
.cal-day:focus { outline: 2px solid var(--cal-accent); outline-offset: 2px; }
.cal-day[aria-pressed="true"] {
  background: var(--cal-accent);
  color: #fff;
  border-radius: 50%;
}
.cal-day.in-range {
  background: var(--cal-range-bg);
  border-radius: 0;
}
.cal-day.range-start { border-radius: 50% 0 0 50%; }
.cal-day.range-end   { border-radius: 0 50% 50% 0; }
.cal-day[aria-disabled="true"] { color: var(--cal-disabled); cursor: not-allowed; }
.cal-day.is-today::after {
  content: '';
  display: block;
  position: absolute;
  bottom: 4px;
  left: 50%; transform: translateX(-50%);
  width: 4px; height: 4px;
  border-radius: 50%;
  background: var(--cal-today);
}

@media (prefers-reduced-motion: reduce) { .calendar-panel { transition: none; } }
```

---

## React Implementation

### Context & State

```tsx
// DateRangeContext.tsx
import { createContext, useContext, useState, useCallback, ReactNode } from 'react';

interface DateRange { start: Date | null; end: Date | null; }

interface DateRangeContextValue {
  range: DateRange;
  hoverDate: Date | null;
  viewMonth: Date;
  selectDate: (d: Date) => void;
  setHoverDate: (d: Date | null) => void;
  prevMonth: () => void;
  nextMonth: () => void;
  isInRange: (d: Date) => boolean;
  isStart: (d: Date) => boolean;
  isEnd: (d: Date) => boolean;
  isDisabled: (d: Date) => boolean;
}

const Ctx = createContext<DateRangeContextValue | null>(null);
export const useDateRange = () => {
  const ctx = useContext(Ctx);
  if (!ctx) throw new Error('useDateRange must be used within DateRangeProvider');
  return ctx;
};

interface Props {
  children: ReactNode;
  minDate?: Date;
  disabledDates?: Date[];
  onChange?: (range: DateRange) => void;
}

export function DateRangeProvider({ children, minDate, disabledDates = [], onChange }: Props) {
  const [range, setRange] = useState<DateRange>({ start: null, end: null });
  const [hoverDate, setHoverDate] = useState<Date | null>(null);
  const [viewMonth, setViewMonth] = useState(() => {
    const d = new Date();
    d.setDate(1);
    return d;
  });

  const isSameDay = (a: Date, b: Date) =>
    a.getFullYear() === b.getFullYear() &&
    a.getMonth() === b.getMonth() &&
    a.getDate() === b.getDate();

  const isDisabled = useCallback(
    (d: Date) => {
      if (minDate && d < minDate) return true;
      return disabledDates.some(dd => isSameDay(dd, d));
    },
    [minDate, disabledDates]
  );

  const selectDate = useCallback(
    (d: Date) => {
      if (isDisabled(d)) return;

      setRange(prev => {
        // If no start, or if both set (restart selection), set start
        if (!prev.start || (prev.start && prev.end)) {
          const next = { start: d, end: null };
          onChange?.(next);
          return next;
        }
        // Start is set, no end yet
        if (d < prev.start) {
          // Clicked before start: make new start
          const next = { start: d, end: null };
          onChange?.(next);
          return next;
        }
        // Valid end
        const next = { start: prev.start, end: d };
        onChange?.(next);
        return next;
      });
    },
    [isDisabled, onChange]
  );

  const effectiveEnd = range.end ?? hoverDate;

  const isInRange = useCallback(
    (d: Date) => {
      if (!range.start || !effectiveEnd) return false;
      const [lo, hi] =
        range.start <= effectiveEnd
          ? [range.start, effectiveEnd]
          : [effectiveEnd, range.start];
      return d > lo && d < hi;
    },
    [range.start, effectiveEnd]
  );

  const isStart = useCallback(
    (d: Date) => !!range.start && isSameDay(d, range.start),
    [range.start]
  );
  const isEnd = useCallback(
    (d: Date) => !!effectiveEnd && isSameDay(d, effectiveEnd),
    [effectiveEnd]
  );

  const prevMonth = () =>
    setViewMonth(m => new Date(m.getFullYear(), m.getMonth() - 1, 1));
  const nextMonth = () =>
    setViewMonth(m => new Date(m.getFullYear(), m.getMonth() + 1, 1));

  return (
    <Ctx.Provider
      value={{
        range, hoverDate, viewMonth,
        selectDate, setHoverDate,
        prevMonth, nextMonth,
        isInRange, isStart, isEnd, isDisabled,
      }}
    >
      {children}
    </Ctx.Provider>
  );
}
```

### Calendar Grid Component

```tsx
// CalendarGrid.tsx
import { useMemo } from 'react';
import { useDateRange } from './DateRangeContext';

function buildCalendarDays(year: number, month: number): (Date | null)[] {
  const firstDay = new Date(year, month, 1).getDay();
  const daysInMonth = new Date(year, month + 1, 0).getDate();
  const offset = (firstDay + 6) % 7; // Monday-first offset
  const cells: (Date | null)[] = Array(offset).fill(null);
  for (let d = 1; d <= daysInMonth; d++) {
    cells.push(new Date(year, month, d));
  }
  return cells;
}

const DAY_LABELS = ['Mo', 'Tu', 'We', 'Th', 'Fr', 'Sa', 'Su'];
const MONTH_NAMES = [
  'January','February','March','April','May','June',
  'July','August','September','October','November','December'
];

export function CalendarGrid() {
  const {
    viewMonth, selectDate, setHoverDate,
    prevMonth, nextMonth, isInRange, isStart, isEnd, isDisabled,
  } = useDateRange();

  const year  = viewMonth.getFullYear();
  const month = viewMonth.getMonth();
  const days  = useMemo(() => buildCalendarDays(year, month), [year, month]);
  const today = new Date();

  const isSameDay = (a: Date, b: Date) =>
    a.getFullYear() === b.getFullYear() &&
    a.getMonth() === b.getMonth() &&
    a.getDate() === b.getDate();

  return (
    <div className="calendar-panel">
      <div className="calendar-header">
        <button className="cal-nav-btn" aria-label="Previous month" onClick={prevMonth}>‹</button>
        <span className="cal-month-label" aria-live="polite">
          {MONTH_NAMES[month]} {year}
        </span>
        <button className="cal-nav-btn" aria-label="Next month" onClick={nextMonth}>›</button>
      </div>

      <table role="grid" aria-label={`${MONTH_NAMES[month]} ${year}`} className="cal-grid">
        <thead>
          <tr>{DAY_LABELS.map(d => <th key={d} scope="col" abbr={d}>{d}</th>)}</tr>
        </thead>
        <tbody>
          {Array.from({ length: Math.ceil(days.length / 7) }, (_, row) => (
            <tr key={row}>
              {days.slice(row * 7, row * 7 + 7).map((date, col) => (
                <td key={col}>
                  {date ? (
                    <button
                      className={[
                        'cal-day',
                        isInRange(date) ? 'in-range' : '',
                        isStart(date)   ? 'range-start' : '',
                        isEnd(date)     ? 'range-end' : '',
                        isSameDay(date, today) ? 'is-today' : '',
                      ].filter(Boolean).join(' ')}
                      aria-label={date.toLocaleDateString('en-GB', {
                        weekday: 'long', year: 'numeric', month: 'long', day: 'numeric',
                      })}
                      aria-pressed={isStart(date) || isEnd(date)}
                      aria-disabled={isDisabled(date)}
                      tabIndex={isDisabled(date) ? -1 : 0}
                      onClick={() => selectDate(date)}
                      onMouseEnter={() => setHoverDate(date)}
                      onMouseLeave={() => setHoverDate(null)}
                    >
                      {date.getDate()}
                    </button>
                  ) : (
                    <span aria-hidden="true" />
                  )}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### Compound Component Entry Point

```tsx
// DateRangePicker.tsx
import { useState, useRef, useEffect } from 'react';
import { DateRangeProvider, useDateRange } from './DateRangeContext';
import { CalendarGrid } from './CalendarGrid';

function Trigger() {
  const { range } = useDateRange();
  const fmt = (d: Date | null) =>
    d ? d.toLocaleDateString('en-GB') : 'DD / MM / YYYY';

  return (
    <div className="date-range-wrapper">
      <div className="date-input-group">
        <label htmlFor="start-date">Check-in</label>
        <input id="start-date" className="date-input" readOnly value={fmt(range.start)} />
      </div>
      <span className="date-range-sep" aria-hidden="true">→</span>
      <div className="date-input-group">
        <label htmlFor="end-date">Check-out</label>
        <input id="end-date" className="date-input" readOnly value={fmt(range.end)} />
      </div>
    </div>
  );
}

interface DateRangePickerProps {
  minDate?: Date;
  disabledDates?: Date[];
  onChange?: (range: { start: Date | null; end: Date | null }) => void;
}

export function DateRangePicker({ minDate, disabledDates, onChange }: DateRangePickerProps) {
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (!ref.current?.contains(e.target as Node)) setOpen(false);
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  return (
    <DateRangeProvider minDate={minDate} disabledDates={disabledDates} onChange={onChange}>
      <div ref={ref} style={{ position: 'relative', display: 'inline-block' }}>
        <div onClick={() => setOpen(o => !o)}>
          <Trigger />
        </div>
        {open && <CalendarGrid />}
      </div>
    </DateRangeProvider>
  );
}
```

---

## Angular Implementation

```typescript
// date-range-picker.component.ts
import {
  Component, Input, Output, EventEmitter,
  signal, computed, ChangeDetectionStrategy
} from '@angular/core';
import { NgFor, NgIf, NgClass } from '@angular/common';

export interface DateRange { start: Date | null; end: Date | null; }

@Component({
  selector: 'app-date-range-picker',
  standalone: true,
  imports: [NgFor, NgIf, NgClass],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="date-range-wrapper">
      <input class="date-input" readonly [value]="fmt(range().start)" (click)="open.set(true)" />
      <span aria-hidden="true">→</span>
      <input class="date-input" readonly [value]="fmt(range().end)" (click)="open.set(true)" />
    </div>

    <div *ngIf="open()" class="calendar-panel" role="dialog" aria-modal="true" aria-label="Select date range">
      <div class="calendar-header">
        <button class="cal-nav-btn" aria-label="Previous month" (click)="changeMonth(-1)">‹</button>
        <span aria-live="polite">{{ monthLabel() }}</span>
        <button class="cal-nav-btn" aria-label="Next month" (click)="changeMonth(1)">›</button>
      </div>

      <table role="grid" [attr.aria-label]="monthLabel()" class="cal-grid">
        <thead>
          <tr><th *ngFor="let d of dayLabels" scope="col">{{ d }}</th></tr>
        </thead>
        <tbody>
          <tr *ngFor="let week of weeks()">
            <td *ngFor="let date of week">
              <button
                *ngIf="date; else empty"
                class="cal-day"
                [ngClass]="{
                  'in-range': date && isInRange(date),
                  'range-start': date && isStart(date),
                  'range-end': date && isEnd(date)
                }"
                [attr.aria-label]="date | date:'EEEE, MMMM d, y'"
                [attr.aria-pressed]="isStart(date) || isEnd(date)"
                [attr.aria-disabled]="isDisabled(date)"
                (click)="selectDate(date)"
                (mouseenter)="hoverDate.set(date)"
                (mouseleave)="hoverDate.set(null)"
              >{{ date.getDate() }}</button>
              <ng-template #empty><span aria-hidden="true"></span></ng-template>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  `,
})
export class DateRangePickerComponent {
  @Input() minDate?: Date;
  @Output() rangeChange = new EventEmitter<DateRange>();

  open      = signal(false);
  range     = signal<DateRange>({ start: null, end: null });
  hoverDate = signal<Date | null>(null);
  viewMonth = signal(new Date(new Date().getFullYear(), new Date().getMonth(), 1));

  dayLabels = ['Mo', 'Tu', 'We', 'Th', 'Fr', 'Sa', 'Su'];

  monthLabel = computed(() =>
    this.viewMonth().toLocaleDateString('en-US', { month: 'long', year: 'numeric' })
  );

  weeks = computed(() => {
    const m = this.viewMonth();
    const first = new Date(m.getFullYear(), m.getMonth(), 1).getDay();
    const days  = new Date(m.getFullYear(), m.getMonth() + 1, 0).getDate();
    const offset = (first + 6) % 7;
    const cells: (Date | null)[] = Array(offset).fill(null);
    for (let d = 1; d <= days; d++)
      cells.push(new Date(m.getFullYear(), m.getMonth(), d));
    const result: (Date | null)[][] = [];
    for (let i = 0; i < cells.length; i += 7) result.push(cells.slice(i, i + 7));
    return result;
  });

  fmt(d: Date | null) { return d ? d.toLocaleDateString('en-GB') : 'DD/MM/YYYY'; }

  isSameDay(a: Date, b: Date) {
    return a.getFullYear() === b.getFullYear() &&
           a.getMonth()    === b.getMonth()    &&
           a.getDate()     === b.getDate();
  }

  isDisabled(d: Date) { return !!(this.minDate && d < this.minDate); }
  isStart(d: Date) { return !!(this.range().start && this.isSameDay(d, this.range().start!)); }
  isEnd(d: Date) {
    const end = this.range().end ?? this.hoverDate();
    return !!(end && this.isSameDay(d, end));
  }
  isInRange(d: Date) {
    const { start } = this.range();
    const end = this.range().end ?? this.hoverDate();
    if (!start || !end) return false;
    return d > start && d < end;
  }

  selectDate(d: Date) {
    if (this.isDisabled(d)) return;
    const { start, end } = this.range();
    if (!start || (start && end)) {
      this.range.set({ start: d, end: null });
    } else if (d < start) {
      this.range.set({ start: d, end: null });
    } else {
      const next = { start, end: d };
      this.range.set(next);
      this.rangeChange.emit(next);
      this.open.set(false);
    }
  }

  changeMonth(delta: number) {
    const m = this.viewMonth();
    this.viewMonth.set(new Date(m.getFullYear(), m.getMonth() + delta, 1));
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Calendar opened as `role="dialog"` | Communicates modal overlay to screen readers |
| `aria-modal="true"` on panel | Prevents virtual cursor from leaving the calendar |
| Calendar table has `role="grid"` | Correct ARIA semantics for interactive grid |
| Each day cell is a `<button>` (not a `<td>`) | Buttons are keyboard-focusable; `<td>` is not |
| Day buttons have descriptive `aria-label` | "Monday, June 15, 2026" — not just "15" |
| `aria-pressed` on selected start/end days | Communicates selection state |
| `aria-disabled` on unavailable dates | Distinguishes from truly disabled (`disabled` attr) |
| Month navigation announced via `aria-live` | Screen reader hears month name on navigation |
| In-range days get no extra ARIA noise | Visual-only distinction; do not aria-label every in-range cell |
| Escape closes the calendar | Standard dialog keyboard contract |

---

## Production Pitfalls

**1. `Date` arithmetic creates off-by-one bugs with timezones**
`new Date('2026-06-15')` parses as UTC midnight, which in UTC-5 becomes June 14th at 7 PM. Always construct dates with `new Date(year, month, day)` (local time constructor), never from ISO strings, inside picker logic.

**2. Hover range state causes excessive re-renders**
Updating `hoverDate` on every `mousemove` can trigger dozens of re-renders per second across the full grid. Fix: use `onMouseEnter`/`onMouseLeave` (not `onMouseMove`) on day cells, and memoize `isInRange` with `useMemo`/`computed`.

**3. Keyboard navigation within the grid**
The ARIA grid pattern requires ArrowLeft/Right/Up/Down to move between cells. A naive tab-order approach is unusable (28+ tab stops per month). Fix: use `roving tabindex` — only the focused cell has `tabIndex={0}`, all others have `tabIndex={-1}`.

**4. Two-month side-by-side picker breaks on mobile**
Many designs show two months side by side on desktop. On mobile, this is too cramped. Fix: conditionally render one or two month grids based on viewport width, or use a single scrollable month-list.

**5. Disabled dates that are inside a selected range**
If a user selects June 1–30 and June 15 is unavailable, the range is invalid. Fix: when the user selects an end date, scan for disabled dates between start and end and reject the range with a visible error message.

---

## Interview Angle

**Q: "How would you design the state architecture for a date range picker?"**

The key design decision is where state lives. A compound component pattern works well: a shared context (React Context or Angular service) holds `start`, `end`, `hoverDate`, and `viewMonth`. The trigger inputs and the calendar grid are siblings that both consume this context. The calendar writes selection; the inputs read it. This avoids prop-drilling and keeps both pieces in sync without lifting state to a parent that shouldn't know about calendar internals.

**Q: "How do you handle the 'hover preview' of the range without causing performance issues?"**

Store `hoverDate` in state and derive the in-range visual by comparing each cell's date to `[start, hoverDate]` inside a `computed` / `useMemo`. The key performance fix is to update `hoverDate` on `mouseenter` (per cell), not on document `mousemove`. This limits updates to N events per month grid navigation rather than continuous pointer events.

**Q: "What's wrong with building a date picker with `<input type='date'>`?"**

Native `<input type="date">` has no range selection, inconsistent cross-browser UI, no ability to disable specific dates, and cannot show two months side by side. It is appropriate for simple single-date forms but insufficient for booking-style range UX.
