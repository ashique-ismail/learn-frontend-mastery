# Editable Table Accessibility

## The Idea

**In plain English:** You have a data table where users can click into a cell to edit its value in place — no separate form, no navigation away. The challenge is that a plain `<table>` with contenteditable cells is invisible to screen readers as an interactive widget. You must apply the ARIA `grid` role pattern to communicate the widget's two-dimensional keyboard model, manage focus explicitly via roving tabindex, and announce mode transitions (read mode vs. edit mode) so screen reader users know when they can type and when they cannot.

**Real-world analogy:** Think of a spreadsheet application like Google Sheets.

- The **spreadsheet grid** = a `role="grid"` container. Screen readers understand this as a navigable two-dimensional widget, not just a static table.
- **Navigating with arrow keys between cells** = the roving tabindex pattern. Only one cell holds `tabindex="0"` at a time; all others are `tabindex="-1"`. Arrow keys move the "live" tabindex to the next cell.
- **Pressing Enter or F2 to start editing** = activating edit mode on a gridcell. The cell transitions from read mode to a text input, and a screen reader announcement says "Editing: Product Name".
- **Pressing Escape to cancel** = restoring the original value and returning to navigation mode. The cell becomes a gridcell again and screen reader focus returns to it.
- **The formula bar showing the current cell address** = the `aria-label` and live region that tells screen reader users exactly which cell is focused and what mode they are in.
- **Column headers that sort when clicked** = `role="columnheader"` with `aria-sort` toggling between `ascending`, `descending`, and `none`.

The key insight: a visual table and an ARIA grid are structurally identical in the DOM — the ARIA roles are an accessibility overlay that changes how assistive technology interprets and interacts with the same HTML.

---

## Learning Objectives

- Understand the ARIA grid role contract and how it differs from `role="table"`
- Implement roving tabindex correctly so only one cell owns focus at a time
- Build Enter-to-edit and Escape-to-cancel with full keyboard navigation
- Announce edit mode activation and cancellation to screen readers via live regions
- Handle column headers with `aria-sort` for sortable columns
- Avoid the common production pitfalls: focus loss on re-render, stale cell refs, and double-announcing

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Style a cell to look editable on focus | Yes | — |
| Show an input inside a cell | Yes (via `:focus-within` sibling tricks) | — |
| Track which cell is currently focused | No | CSS has no concept of the "active" cell in a grid |
| Move focus on arrow key press | No | Requires `keydown` handler and `element.focus()` |
| Roving tabindex management | No | `tabindex` attribute must be set by JS |
| Announce "edit mode" to screen readers | No | `aria-live` region content must be updated by JS |
| Revert value on Escape | No | Requires storing the pre-edit value in state |
| Mark a column as sorted ascending/descending | No | `aria-sort` must be toggled by JS |
| Prevent arrow keys from moving the caret inside an input | No | `event.stopPropagation()` must be called by JS |

**Conclusion:** CSS handles all visual states (focused cell highlight, edit-mode input styling, sort indicator icons). JS owns the interactive contract: focus routing, ARIA attribute toggling, value state, and screen reader announcements.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  role="grid" identifies the widget as a two-dimensional interactive grid.
  aria-label provides an accessible name for the whole widget.
  aria-rowcount / aria-colcount can be added for virtual scroll scenarios.
-->
<table
  role="grid"
  aria-label="Product inventory"
  class="editable-grid"
>
  <thead>
    <tr role="row">
      <!--
        role="columnheader" + aria-sort for sortable columns.
        aria-sort values: "none" | "ascending" | "descending" | "other"
      -->
      <th role="columnheader" aria-sort="ascending" scope="col">
        <button class="sort-btn" aria-label="Sort by Name, currently ascending">
          Name
          <span class="sort-icon" aria-hidden="true">▲</span>
        </button>
      </th>
      <th role="columnheader" aria-sort="none" scope="col">
        <button class="sort-btn" aria-label="Sort by Price, not sorted">
          Price
          <span class="sort-icon" aria-hidden="true"></span>
        </button>
      </th>
      <th role="columnheader" scope="col">Category</th>
    </tr>
  </thead>

  <tbody>
    <tr role="row">
      <!--
        role="gridcell" on every data cell.
        tabindex="0"  on the single currently-focused cell.
        tabindex="-1" on every other cell.
        aria-readonly="false" signals the cell is editable.
      -->
      <td
        role="gridcell"
        tabindex="0"
        aria-readonly="false"
        aria-label="Name: Wireless Keyboard, row 1"
        class="grid-cell grid-cell--focused"
      >
        <span class="cell-value">Wireless Keyboard</span>
      </td>
      <td
        role="gridcell"
        tabindex="-1"
        aria-readonly="false"
        aria-label="Price: 49.99, row 1"
        class="grid-cell"
      >
        <span class="cell-value">49.99</span>
      </td>
      <td
        role="gridcell"
        tabindex="-1"
        aria-readonly="false"
        class="grid-cell"
      >
        <span class="cell-value">Electronics</span>
      </td>
    </tr>
  </tbody>
</table>

<!-- Off-screen live region for mode announcements -->
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
  class="sr-only"
  id="grid-announcer"
></div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --cell-height: 44px;
  --cell-padding: 0 0.75rem;
  --focus-ring: 0 0 0 2px #0066cc;
  --edit-bg: #fffbea;
  --edit-border: #f59e0b;
}

/* ─── Grid container ─── */
.editable-grid {
  border-collapse: collapse;
  width: 100%;
  font-size: 0.9375rem;
}

/* ─── Column headers ─── */
.editable-grid thead th {
  background: #f1f5f9;
  border-bottom: 2px solid #cbd5e1;
  text-align: left;
  white-space: nowrap;
}

.sort-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  background: none;
  border: none;
  font: inherit;
  font-weight: 600;
  padding: var(--cell-padding);
  height: var(--cell-height);
  cursor: pointer;
  width: 100%;
}

.sort-btn:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: -2px;
}

/* Sort icon states driven by aria-sort on the th */
th[aria-sort="ascending"]  .sort-icon::after { content: ' ▲'; }
th[aria-sort="descending"] .sort-icon::after { content: ' ▼'; }
th[aria-sort="none"]       .sort-icon::after { content: ' ⇅'; color: #94a3b8; }

/* ─── Gridcells ─── */
.grid-cell {
  border: 1px solid #e2e8f0;
  height: var(--cell-height);
  padding: var(--cell-padding);
  position: relative;
  outline: none;            /* we draw our own focus ring */
  vertical-align: middle;
  cursor: default;
}

/* Read mode: focused cell */
.grid-cell:focus,
.grid-cell--focused {
  box-shadow: var(--focus-ring);
  z-index: 1;              /* keep ring on top of adjacent cell borders */
}

/* Edit mode: cell containing an active input */
.grid-cell--editing {
  background: var(--edit-bg);
  box-shadow: 0 0 0 2px var(--edit-border);
  z-index: 2;
  padding: 0;              /* input fills the full cell */
}

/* The edit input fills the cell exactly */
.cell-input {
  display: block;
  width: 100%;
  height: 100%;
  border: none;
  background: transparent;
  font: inherit;
  padding: var(--cell-padding);
  outline: none;
}

/* ─── Utility ─── */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .grid-cell,
  .grid-cell--editing {
    transition: none;
  }
}
```

**What CSS owns:** focus ring appearance, edit-mode background, sort indicator glyphs via `attr()` / `content`, tap-target sizing.

**What CSS cannot own:** which cell holds `tabindex="0"`, the `aria-sort` toggle, the edit/read mode ARIA state, or the live region text.

---

## React Implementation

### Types

```tsx
// editableGrid.types.ts
export interface Column<T> {
  key: keyof T;
  label: string;
  sortable?: boolean;
}

export type SortDir = 'ascending' | 'descending' | 'none';

export interface SortState<T> {
  key: keyof T | null;
  dir: SortDir;
}

export interface CellPosition {
  row: number;
  col: number;
}
```

### The Hook — Grid Keyboard Navigation

```tsx
// useGridNavigation.ts
import { useState, useCallback, useRef } from 'react';
import type { CellPosition } from './editableGrid.types';

interface UseGridNavigationOptions {
  rowCount: number;
  colCount: number;
  onAnnounce: (message: string) => void;
}

export function useGridNavigation({
  rowCount,
  colCount,
  onAnnounce,
}: UseGridNavigationOptions) {
  // The single cell that holds tabindex="0"
  const [focused, setFocused] = useState<CellPosition>({ row: 0, col: 0 });
  // Which cell (if any) is in edit mode
  const [editing, setEditing] = useState<CellPosition | null>(null);
  // Store pre-edit values so Escape can revert
  const preEditValues = useRef<Map<string, string>>(new Map());

  const cellKey = (pos: CellPosition) => `${pos.row}:${pos.col}`;

  const moveFocus = useCallback(
    (next: CellPosition) => {
      const clamped: CellPosition = {
        row: Math.max(0, Math.min(next.row, rowCount - 1)),
        col: Math.max(0, Math.min(next.col, colCount - 1)),
      };
      setFocused(clamped);
    },
    [rowCount, colCount]
  );

  const startEditing = useCallback(
    (pos: CellPosition, currentValue: string, colLabel: string) => {
      preEditValues.current.set(cellKey(pos), currentValue);
      setEditing(pos);
      onAnnounce(`Editing ${colLabel}: ${currentValue}. Press Escape to cancel.`);
    },
    [onAnnounce]
  );

  const cancelEditing = useCallback(
    (colLabel: string): string | null => {
      if (!editing) return null;
      const original = preEditValues.current.get(cellKey(editing)) ?? null;
      preEditValues.current.delete(cellKey(editing));
      setEditing(null);
      onAnnounce(`Cancelled. ${colLabel} restored.`);
      return original;
    },
    [editing, onAnnounce]
  );

  const commitEditing = useCallback(
    (newValue: string, colLabel: string) => {
      if (!editing) return;
      preEditValues.current.delete(cellKey(editing));
      setEditing(null);
      onAnnounce(`${colLabel} updated to: ${newValue}`);
    },
    [editing, onAnnounce]
  );

  const isFocused = (pos: CellPosition) =>
    pos.row === focused.row && pos.col === focused.col;

  const isEditing = (pos: CellPosition) =>
    editing !== null && editing.row === pos.row && editing.col === pos.col;

  return {
    focused,
    editing,
    moveFocus,
    startEditing,
    cancelEditing,
    commitEditing,
    isFocused,
    isEditing,
  };
}
```

### The Full Component

```tsx
// EditableGrid.tsx
import { useState, useRef, useCallback, useEffect } from 'react';
import { useGridNavigation } from './useGridNavigation';
import type { Column, SortDir, SortState } from './editableGrid.types';

interface Props<T extends Record<string, string | number>> {
  data: T[];
  columns: Column<T>[];
  onChange: (rowIndex: number, key: keyof T, value: string) => void;
}

export function EditableGrid<T extends Record<string, string | number>>({
  data,
  columns,
  onChange,
}: Props<T>) {
  // ── Announcer ──
  const announcerRef = useRef<HTMLDivElement>(null);
  const announce = useCallback((msg: string) => {
    if (!announcerRef.current) return;
    // Clear then set forces re-announcement even for identical strings
    announcerRef.current.textContent = '';
    requestAnimationFrame(() => {
      if (announcerRef.current) announcerRef.current.textContent = msg;
    });
  }, []);

  // ── Sorting ──
  const [sort, setSort] = useState<SortState<T>>({ key: null, dir: 'none' });

  const sortedData = [...data].sort((a, b) => {
    if (!sort.key || sort.dir === 'none') return 0;
    const av = a[sort.key];
    const bv = b[sort.key];
    const cmp = av < bv ? -1 : av > bv ? 1 : 0;
    return sort.dir === 'ascending' ? cmp : -cmp;
  });

  const handleSort = (key: keyof T, label: string) => {
    setSort(prev => {
      const next: SortDir =
        prev.key !== key ? 'ascending'
        : prev.dir === 'ascending' ? 'descending'
        : prev.dir === 'descending' ? 'none'
        : 'ascending';
      announce(
        next === 'none'
          ? `${label}: sort cleared`
          : `${label}: sorted ${next}`
      );
      return { key, dir: next };
    });
  };

  // ── Grid navigation ──
  const {
    focused,
    moveFocus,
    startEditing,
    cancelEditing,
    commitEditing,
    isFocused,
    isEditing,
  } = useGridNavigation({
    rowCount: sortedData.length,
    colCount: columns.length,
    onAnnounce: announce,
  });

  // Sync DOM focus whenever focused cell changes
  const cellRefs = useRef<Map<string, HTMLTableCellElement>>(new Map());
  const inputRefs = useRef<Map<string, HTMLInputElement>>(new Map());

  useEffect(() => {
    const key = `${focused.row}:${focused.col}`;
    cellRefs.current.get(key)?.focus();
  }, [focused]);

  // When a cell enters edit mode, focus its input
  useEffect(() => {
    // editing state is derived inside the hook, but we need a way to focus
    // the input. We watch the editing row/col via a separate effect.
  }, []);

  // ── Per-cell edit value (uncontrolled-like) ──
  const [editValues, setEditValues] = useState<Record<string, string>>({});

  // ── Cell keyboard handler (read mode) ──
  const handleCellKeyDown = useCallback(
    (
      e: React.KeyboardEvent<HTMLTableCellElement>,
      rowIdx: number,
      colIdx: number
    ) => {
      const col = columns[colIdx];
      const value = String(sortedData[rowIdx][col.key] ?? '');

      switch (e.key) {
        case 'ArrowRight':
          e.preventDefault();
          moveFocus({ row: rowIdx, col: colIdx + 1 });
          break;
        case 'ArrowLeft':
          e.preventDefault();
          moveFocus({ row: rowIdx, col: colIdx - 1 });
          break;
        case 'ArrowDown':
          e.preventDefault();
          moveFocus({ row: rowIdx + 1, col: colIdx });
          break;
        case 'ArrowUp':
          e.preventDefault();
          moveFocus({ row: rowIdx - 1, col: colIdx });
          break;
        case 'Enter':
        case 'F2': {
          e.preventDefault();
          const cellKey = `${rowIdx}:${colIdx}`;
          setEditValues(prev => ({ ...prev, [cellKey]: value }));
          startEditing({ row: rowIdx, col: colIdx }, value, col.label);
          // Focus the input after state settles
          requestAnimationFrame(() => {
            inputRefs.current.get(cellKey)?.focus();
          });
          break;
        }
        // Typing immediately starts edit mode with the typed char
        default: {
          if (e.key.length === 1 && !e.ctrlKey && !e.metaKey && !e.altKey) {
            const cellKey = `${rowIdx}:${colIdx}`;
            setEditValues(prev => ({ ...prev, [cellKey]: e.key }));
            startEditing({ row: rowIdx, col: colIdx }, value, col.label);
            requestAnimationFrame(() => {
              inputRefs.current.get(cellKey)?.focus();
            });
          }
        }
      }
    },
    [columns, sortedData, moveFocus, startEditing]
  );

  // ── Input keyboard handler (edit mode) ──
  const handleInputKeyDown = useCallback(
    (
      e: React.KeyboardEvent<HTMLInputElement>,
      rowIdx: number,
      colIdx: number
    ) => {
      const col = columns[colIdx];
      const cellKey = `${rowIdx}:${colIdx}`;

      if (e.key === 'Escape') {
        e.preventDefault();
        e.stopPropagation();
        const original = cancelEditing(col.label);
        if (original !== null) {
          setEditValues(prev => ({ ...prev, [cellKey]: original }));
        }
        // Return focus to the cell
        requestAnimationFrame(() => {
          cellRefs.current.get(cellKey)?.focus();
        });
      } else if (e.key === 'Enter' || e.key === 'Tab') {
        e.preventDefault();
        const newValue = editValues[cellKey] ?? '';
        commitEditing(newValue, col.label);
        onChange(rowIdx, col.key, newValue);
        requestAnimationFrame(() => {
          cellRefs.current.get(cellKey)?.focus();
        });
        if (e.key === 'Tab') {
          moveFocus({ row: rowIdx, col: colIdx + (e.shiftKey ? -1 : 1) });
        }
      }
      // Arrow keys inside an input should NOT move cell focus
      if (['ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown'].includes(e.key)) {
        e.stopPropagation();
      }
    },
    [columns, editValues, cancelEditing, commitEditing, onChange, moveFocus]
  );

  return (
    <>
      {/* Off-screen announcer */}
      <div
        ref={announcerRef}
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="sr-only"
        id="grid-announcer"
      />

      <table
        role="grid"
        aria-label="Product inventory"
        aria-rowcount={sortedData.length + 1}
        aria-colcount={columns.length}
        className="editable-grid"
      >
        <thead>
          <tr role="row">
            {columns.map((col, ci) => {
              const sortDir: SortDir =
                sort.key === col.key ? sort.dir : 'none';
              return (
                <th
                  key={String(col.key)}
                  role="columnheader"
                  aria-sort={col.sortable ? sortDir : undefined}
                  scope="col"
                >
                  {col.sortable ? (
                    <button
                      className="sort-btn"
                      onClick={() => handleSort(col.key, col.label)}
                      aria-label={`Sort by ${col.label}${sortDir !== 'none' ? `, currently ${sortDir}` : ''}`}
                    >
                      {col.label}
                      <span className="sort-icon" aria-hidden="true" />
                    </button>
                  ) : (
                    col.label
                  )}
                </th>
              );
            })}
          </tr>
        </thead>
        <tbody>
          {sortedData.map((row, ri) => (
            <tr key={ri} role="row" aria-rowindex={ri + 2}>
              {columns.map((col, ci) => {
                const pos = { row: ri, col: ci };
                const cellKey = `${ri}:${ci}`;
                const focused = isFocused(pos);
                const editing = isEditing(pos);
                const displayValue = String(row[col.key] ?? '');

                return (
                  <td
                    key={String(col.key)}
                    ref={el => {
                      if (el) cellRefs.current.set(cellKey, el);
                      else cellRefs.current.delete(cellKey);
                    }}
                    role="gridcell"
                    tabIndex={focused && !editing ? 0 : -1}
                    aria-readonly={false}
                    aria-selected={focused}
                    aria-colindex={ci + 1}
                    aria-label={`${col.label}: ${displayValue}, row ${ri + 1}`}
                    className={[
                      'grid-cell',
                      focused && !editing ? 'grid-cell--focused' : '',
                      editing ? 'grid-cell--editing' : '',
                    ]
                      .filter(Boolean)
                      .join(' ')}
                    onKeyDown={
                      !editing
                        ? e => handleCellKeyDown(e, ri, ci)
                        : undefined
                    }
                    onFocus={() => moveFocus(pos)}
                  >
                    {editing ? (
                      <input
                        ref={el => {
                          if (el) inputRefs.current.set(cellKey, el);
                          else inputRefs.current.delete(cellKey);
                        }}
                        className="cell-input"
                        value={editValues[cellKey] ?? displayValue}
                        aria-label={`Edit ${col.label}`}
                        onChange={e =>
                          setEditValues(prev => ({
                            ...prev,
                            [cellKey]: e.target.value,
                          }))
                        }
                        onKeyDown={e => handleInputKeyDown(e, ri, ci)}
                        onBlur={() => {
                          // Commit on blur (click outside)
                          const newValue = editValues[cellKey] ?? displayValue;
                          commitEditing(newValue, col.label);
                          onChange(ri, col.key, newValue);
                        }}
                      />
                    ) : (
                      <span className="cell-value">{displayValue}</span>
                    )}
                  </td>
                );
              })}
            </tr>
          ))}
        </tbody>
      </table>
    </>
  );
}
```

---

## Angular Implementation

### Model & Service

```typescript
// editable-grid.model.ts
export interface GridColumn<T> {
  key: keyof T;
  label: string;
  sortable?: boolean;
}

export type SortDir = 'ascending' | 'descending' | 'none';

export interface CellPos {
  row: number;
  col: number;
}
```

```typescript
// editable-grid.service.ts
import { Injectable, signal, computed } from '@angular/core';
import type { CellPos } from './editable-grid.model';

@Injectable()
export class EditableGridService {
  private _focused  = signal<CellPos>({ row: 0, col: 0 });
  private _editing  = signal<CellPos | null>(null);
  private _preEditValues = new Map<string, string>();

  readonly focused  = this._focused.asReadonly();
  readonly editing  = this._editing.asReadonly();

  readonly isEditing = computed(() => this._editing() !== null);

  private key(pos: CellPos) { return `${pos.row}:${pos.col}`; }

  focus(pos: CellPos) { this._focused.set(pos); }

  isFocused(pos: CellPos): boolean {
    const f = this._focused();
    return f.row === pos.row && f.col === pos.col;
  }

  isCellEditing(pos: CellPos): boolean {
    const e = this._editing();
    return e !== null && e.row === pos.row && e.col === pos.col;
  }

  startEdit(pos: CellPos, currentValue: string) {
    this._preEditValues.set(this.key(pos), currentValue);
    this._editing.set({ ...pos });
  }

  cancelEdit(): string | null {
    const pos = this._editing();
    if (!pos) return null;
    const original = this._preEditValues.get(this.key(pos)) ?? null;
    this._preEditValues.delete(this.key(pos));
    this._editing.set(null);
    return original;
  }

  commitEdit() {
    const pos = this._editing();
    if (!pos) return;
    this._preEditValues.delete(this.key(pos));
    this._editing.set(null);
  }

  moveFocus(delta: { dr: number; dc: number }, maxRow: number, maxCol: number) {
    const cur = this._focused();
    this._focused.set({
      row: Math.max(0, Math.min(cur.row + delta.dr, maxRow - 1)),
      col: Math.max(0, Math.min(cur.col + delta.dc, maxCol - 1)),
    });
  }
}
```

### Standalone Component

```typescript
// editable-grid.component.ts
import {
  Component, Input, Output, EventEmitter, OnChanges,
  inject, ElementRef, ViewChildren, QueryList,
  AfterViewInit, effect, signal
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { EditableGridService } from './editable-grid.service';
import type { GridColumn, SortDir, CellPos } from './editable-grid.model';

@Component({
  selector: 'app-editable-grid',
  standalone: true,
  imports: [NgFor, NgIf, FormsModule],
  providers: [EditableGridService],   // scoped per-instance
  template: `
    <!-- Off-screen announcer -->
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      class="sr-only"
      #announcer
    ></div>

    <table
      role="grid"
      [attr.aria-label]="gridLabel"
      [attr.aria-rowcount]="rows.length + 1"
      [attr.aria-colcount]="columns.length"
      class="editable-grid"
    >
      <thead>
        <tr role="row">
          <th
            *ngFor="let col of columns; let ci = index"
            role="columnheader"
            [attr.aria-sort]="col.sortable ? getSortDir(col.key) : null"
            scope="col"
          >
            <button
              *ngIf="col.sortable"
              class="sort-btn"
              (click)="onSort(col)"
              [attr.aria-label]="'Sort by ' + col.label + getSortLabel(col.key)"
            >
              {{ col.label }}
              <span class="sort-icon" aria-hidden="true"></span>
            </button>
            <ng-container *ngIf="!col.sortable">{{ col.label }}</ng-container>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr
          *ngFor="let row of sortedRows; let ri = index"
          role="row"
          [attr.aria-rowindex]="ri + 2"
        >
          <td
            *ngFor="let col of columns; let ci = index"
            #cellEl
            role="gridcell"
            [tabindex]="svc.isFocused({row: ri, col: ci}) && !svc.isCellEditing({row: ri, col: ci}) ? 0 : -1"
            [attr.aria-readonly]="false"
            [attr.aria-selected]="svc.isFocused({row: ri, col: ci})"
            [attr.aria-colindex]="ci + 1"
            [attr.aria-label]="col.label + ': ' + row[col.key] + ', row ' + (ri + 1)"
            [class.grid-cell]="true"
            [class.grid-cell--focused]="svc.isFocused({row: ri, col: ci}) && !svc.isCellEditing({row: ri, col: ci})"
            [class.grid-cell--editing]="svc.isCellEditing({row: ri, col: ci})"
            (keydown)="!svc.isCellEditing({row: ri, col: ci}) && onCellKeyDown($event, ri, ci)"
            (focus)="svc.focus({row: ri, col: ci})"
          >
            <input
              *ngIf="svc.isCellEditing({row: ri, col: ci})"
              class="cell-input"
              [ngModel]="editBuffer"
              (ngModelChange)="editBuffer = $event"
              [attr.aria-label]="'Edit ' + col.label"
              (keydown)="onInputKeyDown($event, ri, ci, col)"
              (blur)="onInputBlur(ri, ci, col)"
              #inputEl
            />
            <span
              *ngIf="!svc.isCellEditing({row: ri, col: ci})"
              class="cell-value"
            >{{ row[col.key] }}</span>
          </td>
        </tr>
      </tbody>
    </table>
  `,
})
export class EditableGridComponent<T extends Record<string, string | number>>
  implements OnChanges, AfterViewInit
{
  @Input() rows: T[] = [];
  @Input() columns: GridColumn<T>[] = [];
  @Input() gridLabel = 'Editable data grid';
  @Output() rowChange = new EventEmitter<{ index: number; key: keyof T; value: string }>();

  @ViewChildren('cellEl') cellEls!: QueryList<ElementRef<HTMLTableCellElement>>;
  @ViewChildren('inputEl') inputEls!: QueryList<ElementRef<HTMLInputElement>>;
  @ViewChildren('announcer') announcerEl!: QueryList<ElementRef<HTMLDivElement>>;

  svc = inject(EditableGridService);

  sortKey: keyof T | null = null;
  sortDir: SortDir = 'none';
  sortedRows: T[] = [];
  editBuffer = '';

  ngOnChanges() {
    this.applySorting();
  }

  ngAfterViewInit() {
    // Focus the first cell on init
    this.focusCellAt({ row: 0, col: 0 });
  }

  getSortDir(key: keyof T): SortDir {
    return this.sortKey === key ? this.sortDir : 'none';
  }

  getSortLabel(key: keyof T): string {
    const dir = this.getSortDir(key);
    return dir !== 'none' ? `, currently ${dir}` : '';
  }

  onSort(col: GridColumn<T>) {
    if (this.sortKey !== col.key) {
      this.sortKey = col.key;
      this.sortDir = 'ascending';
    } else {
      this.sortDir =
        this.sortDir === 'ascending' ? 'descending'
        : this.sortDir === 'descending' ? 'none'
        : 'ascending';
      if (this.sortDir === 'none') this.sortKey = null;
    }
    this.applySorting();
    this.announce(
      this.sortDir === 'none'
        ? `${col.label}: sort cleared`
        : `${col.label}: sorted ${this.sortDir}`
    );
  }

  private applySorting() {
    if (!this.sortKey || this.sortDir === 'none') {
      this.sortedRows = [...this.rows];
      return;
    }
    const key = this.sortKey;
    const dir = this.sortDir;
    this.sortedRows = [...this.rows].sort((a, b) => {
      const av = a[key], bv = b[key];
      const cmp = av < bv ? -1 : av > bv ? 1 : 0;
      return dir === 'ascending' ? cmp : -cmp;
    });
  }

  onCellKeyDown(e: KeyboardEvent, ri: number, ci: number) {
    const col = this.columns[ci];
    const value = String(this.sortedRows[ri]?.[col.key] ?? '');

    const moves: Record<string, { dr: number; dc: number }> = {
      ArrowRight: { dr: 0, dc: 1 },
      ArrowLeft:  { dr: 0, dc: -1 },
      ArrowDown:  { dr: 1, dc: 0 },
      ArrowUp:    { dr: -1, dc: 0 },
    };

    if (moves[e.key]) {
      e.preventDefault();
      this.svc.moveFocus(moves[e.key], this.sortedRows.length, this.columns.length);
      const f = this.svc.focused();
      this.focusCellAt(f);
      return;
    }

    if (e.key === 'Enter' || e.key === 'F2') {
      e.preventDefault();
      this.beginEdit(ri, ci, value);
      return;
    }

    if (e.key.length === 1 && !e.ctrlKey && !e.metaKey && !e.altKey) {
      this.editBuffer = e.key;
      this.beginEdit(ri, ci, value);
    }
  }

  private beginEdit(ri: number, ci: number, currentValue: string) {
    this.editBuffer = currentValue;
    this.svc.startEdit({ row: ri, col: ci }, currentValue);
    const col = this.columns[ci];
    this.announce(`Editing ${col.label}: ${currentValue}. Press Escape to cancel.`);
    setTimeout(() => {
      const input = this.inputEls.first?.nativeElement;
      input?.focus();
      input?.select();
    }, 0);
  }

  onInputKeyDown(e: KeyboardEvent, ri: number, ci: number, col: GridColumn<T>) {
    if (e.key === 'Escape') {
      e.preventDefault();
      e.stopPropagation();
      const original = this.svc.cancelEdit();
      if (original !== null) this.editBuffer = original;
      this.announce(`Cancelled. ${col.label} restored.`);
      setTimeout(() => this.focusCellAt({ row: ri, col: ci }), 0);
    } else if (e.key === 'Enter' || e.key === 'Tab') {
      e.preventDefault();
      this.commitAndReturn(ri, ci, col, e.shiftKey && e.key === 'Tab' ? -1 : e.key === 'Tab' ? 1 : 0);
    }
    // Stop arrow keys from propagating to cell handler while editing
    if (['ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown'].includes(e.key)) {
      e.stopPropagation();
    }
  }

  onInputBlur(ri: number, ci: number, col: GridColumn<T>) {
    if (this.svc.isCellEditing({ row: ri, col: ci })) {
      this.commitAndReturn(ri, ci, col, 0);
    }
  }

  private commitAndReturn(ri: number, ci: number, col: GridColumn<T>, dcOffset: number) {
    this.rowChange.emit({ index: ri, key: col.key, value: this.editBuffer });
    this.svc.commitEdit();
    this.announce(`${col.label} updated to: ${this.editBuffer}`);
    setTimeout(() => {
      if (dcOffset !== 0) {
        this.svc.moveFocus({ dr: 0, dc: dcOffset }, this.sortedRows.length, this.columns.length);
        this.focusCellAt(this.svc.focused());
      } else {
        this.focusCellAt({ row: ri, col: ci });
      }
    }, 0);
  }

  private focusCellAt(pos: CellPos) {
    const index = pos.row * this.columns.length + pos.col;
    const cells = this.cellEls.toArray();
    cells[index]?.nativeElement.focus();
  }

  private announce(msg: string) {
    const el = this.announcerEl.first?.nativeElement;
    if (!el) return;
    el.textContent = '';
    requestAnimationFrame(() => { el.textContent = msg; });
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Table container uses `role="grid"` | Applied to `<table>` element; distinguishes interactive widget from static `role="table"` |
| Each data row uses `role="row"` | Applied to `<tr>` elements in `<tbody>` |
| Each data cell uses `role="gridcell"` | Applied to `<td>` elements; columnheaders use `role="columnheader"` |
| Roving tabindex | `tabindex="0"` on the single focused cell; `tabindex="-1"` on all others; updated on every navigation |
| Arrow key navigation | `keydown` handler on gridcells moves focus; clamped to valid row/col bounds |
| Enter / F2 activates edit mode | Replaces static span with a text input; announces "Editing [col]: [value]" |
| Typing a character starts edit mode | Single-character keydown on a focused cell seeds `editBuffer` and starts editing |
| Escape cancels edit | Reverts to pre-edit value, removes input, returns focus to cell, announces cancellation |
| Tab commits and moves | Commits edit, moves focus to next or previous cell, announces update |
| Screen reader announcement on mode change | `role="status"` + `aria-live="polite"` off-screen div; cleared then repopulated to force re-read |
| Column sort state communicated | `aria-sort="ascending\|descending\|none"` on `<th role="columnheader">` |
| Cell context in announcement | `aria-label` on each gridcell includes column name, current value, and row number |
| `aria-selected` on focused cell | Tells screen readers which cell is the "current" cell in the grid |
| `aria-readonly="false"` on editable cells | Explicitly signals cells can be edited, not just read |
| `aria-rowcount` / `aria-colcount` | Set on `role="grid"` for accurate announcement in virtual scroll scenarios |

---

## Production Pitfalls

**1. Re-renders destroy the focused cell's DOM node**
In React, if the sorted data array changes during an edit (e.g., a live data poll), the component re-renders and the `<input>` inside the editing cell may be destroyed. The workaround is to stabilise the row key — use a unique row ID as the React key, not the array index. With Angular's `trackBy`, pass the row's `id` field. Never use array index as a key in an editable grid; a sort or insert will cause the wrong cell to appear in edit mode.

**2. `requestAnimationFrame` race with focus**
Calling `element.focus()` immediately after setting state that triggers a re-render will focus a stale DOM node that is about to be replaced. Always schedule `.focus()` inside `requestAnimationFrame` (React) or `setTimeout(0)` (Angular `ngAfterViewChecked` timing). The announcer also needs two frames: clear the text in frame N, set the new text in frame N+1 — otherwise identical consecutive messages are swallowed by screen readers.

**3. Arrow keys move the text cursor inside the input**
When a cell is in edit mode, `ArrowLeft` / `ArrowRight` should move the caret inside the text, not navigate to the adjacent cell. The cell's `keydown` handler must be disabled while editing (`!svc.isCellEditing` guard in Angular, `undefined` handler in React). The input's own `keydown` handler must call `e.stopPropagation()` for arrow keys so they never bubble up to the cell.

**4. `aria-live` announces the wrong message after a sort**
Sorting changes `sortedRows`, which re-renders every cell. If each `aria-label` attribute change triggers a mutation observer inside the screen reader, the user hears dozens of cell announcements instead of just "Name: sorted ascending". Fix: batch sort + announce in a single tick. Use a dedicated announcer element, not `aria-label` changes, as the announcement channel.

**5. `tabindex` out of sync after programmatic row insertion**
If a row is prepended to the data array, all row indices shift by one. A cell stored as `focused = { row: 2, col: 1 }` now points at the wrong row. The grid service must receive the updated row count and clamp the focused position. In both React and Angular, fire a `moveFocus({ dr: 0, dc: 0 }, newRowCount, colCount)` clamp after any data mutation to keep the focused state valid.

**6. Focus trap conflicts with the browser's native grid navigation**
Screen readers on Windows (NVDA/JAWS) have their own table navigation keys (Ctrl+Alt+Arrow) that operate independently of the DOM focus. Do not intercept these — they work at the virtual buffer level and cannot be overridden from JS. Your `keydown` handler only needs to manage DOM focus for sighted keyboard users. Screen reader users with `role="grid"` will get NVDA/JAWS navigation for free without any extra work from you.

---

## Interview Angle

**Q: "What is the difference between `role="table"` and `role="grid"`, and when do you use each?"**

`role="table"` is a static, read-only structure. Screen readers treat it like an HTML table — arrow keys do not navigate cells, and there is no concept of a "selected" cell or interactive mode. `role="grid"` is a composite interactive widget. It signals to screen readers that the widget has its own keyboard navigation contract: arrow keys move between cells, Enter/Space activate cells, and one cell is always the "active" cell managed by the application. You use `role="table"` for presentational data (reports, comparison tables). You use `role="grid"` whenever any cell is focusable, editable, selectable, or participates in a keyboard navigation pattern. Using `role="table"` on an editable grid is a critical accessibility bug — screen reader users will be unable to keyboard-navigate to cells that are technically in the tab order but not part of any announced interactive widget.

**Follow-up: "A QA tester reports that NVDA reads out every cell label every time the table re-sorts. How do you debug and fix that?"**

NVDA watches the live DOM for `aria-live` region mutations and for changes inside `role="grid"`. When sorting re-renders all rows, every `aria-label` on every `<td>` changes simultaneously — NVDA queues all of them. The fix has two parts: first, ensure every row has a stable `key` (row ID, not index) so React/Angular reuses DOM nodes rather than replacing them, which means `aria-label` attributes on unchanged rows do not mutate. Second, route all intentional announcements through a single dedicated `role="status"` `aria-live="polite"` off-screen div rather than relying on attribute mutations. The test is to sort the table with a screen reader and confirm exactly one announcement fires.
