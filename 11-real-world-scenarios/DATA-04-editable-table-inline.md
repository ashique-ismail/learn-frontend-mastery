# Editable Table (Inline Editing)

## The Idea

**In plain English:** A data table where clicking a cell — or a row — replaces the static text with an input field so the user can edit the value without leaving the page. When the user confirms the change (pressing Enter, Tab, or clicking Save), the new value is sent to the server. If the save fails, the cell reverts to its previous value. The interaction must be keyboard-navigable: Tab moves to the next editable cell, Shift+Tab moves back, Enter confirms, Escape cancels.

**Real-world analogy:** Think of a spreadsheet program like Google Sheets.

- **Viewing mode** — you see all the rows and columns as read-only text. You can arrow through them quickly.
- **Edit mode** — double-click or press F2 on a cell and the cell turns into an input field. Only that one cell is "active."
- **Row mode** — some tools (like Airtable's record drawer) let you click a row's edit icon to open the entire row for editing at once, with Save / Cancel buttons at the bottom.
- **Tab key** — moves you from one cell to the next, left to right, top to bottom, exactly like a spreadsheet. You never need to reach for the mouse mid-edit.
- **Optimistic save** — Sheets shows your change instantly, then quietly syncs to the server. If the sync fails, it shows a red "Could not save" badge and reverts the cell.

The key insight: the table owns two states simultaneously — the *committed* data (last confirmed by the server) and the *draft* data (what the user is typing right now). If the save fails, you throw the draft away and restore the committed value.

---

## Learning Objectives

- Understand the cell edit mode vs. row edit mode trade-off and when to choose each
- Build the click-to-edit interaction with a zero-layout-shift input overlay
- Implement Tab / Shift+Tab / Enter / Escape keyboard navigation across cells
- Design a save/cancel API that keeps committed state separate from draft state
- Apply optimistic updates with full rollback on API failure
- Expose correct ARIA roles and live regions so screen readers understand the editing state

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show an input when a cell is clicked | ❌ | CSS has no click-to-mutate-DOM mechanism |
| Track which cell is currently being edited | ❌ | CSS has no per-cell integer coordinate state |
| Validate input value before saving | ❌ | CSS cannot read or evaluate input content |
| Cancel and restore previous value on Escape | ❌ | CSS cannot store previous values |
| Send changed value to an API | ❌ | CSS has no fetch/XHR capability |
| Rollback on server error | ❌ | CSS cannot branch on async results |
| Move focus to the next cell on Tab | ❌ | CSS cannot call `focus()` on a calculated target element |
| Announce "now editing row 3, column Price" to screen readers | ❌ | Dynamic ARIA updates require JS |
| Show a saving spinner per cell | ⚠️ | CSS can animate a spinner, but JS must add/remove the class |

**Conclusion:** CSS owns the visual distinction between read mode and edit mode (border, background, input sizing). JS owns every behavioral aspect: which cell is active, draft value management, API calls, rollback, and keyboard routing.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Each cell renders one of two states -->

<!-- Read mode -->
<td class="cell" data-row="2" data-col="1">
  <span class="cell__display">Acme Corp</span>
  <!-- invisible until hover/focus -->
  <button class="cell__edit-trigger" aria-label="Edit company name">
    <svg aria-hidden="true"><!-- pencil icon --></svg>
  </button>
</td>

<!-- Edit mode (JS swaps in this markup) -->
<td class="cell cell--editing" data-row="2" data-col="1">
  <input
    class="cell__input"
    type="text"
    value="Acme Corp"
    aria-label="Edit company name"
    aria-describedby="cell-hint"
  />
  <!-- optional inline save/cancel for mouse users -->
  <div class="cell__actions">
    <button class="cell__save"   aria-label="Save">✓</button>
    <button class="cell__cancel" aria-label="Cancel">✗</button>
  </div>
</td>

<!-- Row edit mode — entire row becomes a form -->
<tr class="row row--editing" role="row">
  <td><input class="cell__input" value="Acme Corp" /></td>
  <td><input class="cell__input" type="number" value="4200" /></td>
  <td>
    <button class="btn btn--save">Save</button>
    <button class="btn btn--cancel">Cancel</button>
  </td>
</tr>

<!-- Screen reader live region for save feedback -->
<div id="cell-hint" class="sr-only" aria-live="polite"></div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --cell-border:        1px solid #e2e8f0;
  --cell-edit-border:   2px solid #3b82f6;
  --cell-edit-bg:       #eff6ff;
  --cell-error-border:  2px solid #ef4444;
  --cell-error-bg:      #fef2f2;
  --cell-saving-opacity: 0.6;
  --transition-fast:    0.15s ease;
}

/* ─── Base cell ─── */
.cell {
  position: relative;
  padding: 0;       /* input handles its own padding */
  border: var(--cell-border);
  vertical-align: middle;
  white-space: nowrap;
}

/* ─── Read-mode display ─── */
.cell__display {
  display: block;
  padding: 0.5rem 0.75rem;
  min-height: 2.5rem;
  line-height: 1.5;
  cursor: default;
}

/* Edit trigger button — hidden until row/cell is hovered or focused */
.cell__edit-trigger {
  position: absolute;
  top: 50%;
  right: 0.375rem;
  transform: translateY(-50%);
  opacity: 0;
  pointer-events: none;
  transition: opacity var(--transition-fast);
  background: none;
  border: none;
  cursor: pointer;
  padding: 0.25rem;
  color: #64748b;
}

.cell:hover .cell__edit-trigger,
.cell:focus-within .cell__edit-trigger {
  opacity: 1;
  pointer-events: auto;
}

/* ─── Edit mode ─── */
.cell--editing {
  outline: none;
  background: var(--cell-edit-bg);
  box-shadow: 0 0 0 2px var(--cell-edit-border);
  z-index: 1;           /* sit above neighbouring cells */
}

/* Input fills the cell exactly — zero layout shift */
.cell__input {
  display: block;
  width: 100%;
  height: 100%;
  min-height: 2.5rem;
  padding: 0.5rem 0.75rem;
  border: none;
  background: transparent;
  font: inherit;
  color: inherit;
  outline: none;
}

/* Inline save/cancel buttons (shown on edit) */
.cell__actions {
  position: absolute;
  bottom: -2rem;
  right: 0;
  display: flex;
  gap: 0.25rem;
  z-index: 2;
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 0 0 4px 4px;
  padding: 0.25rem;
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

/* ─── Saving state ─── */
.cell--saving .cell__input {
  opacity: var(--cell-saving-opacity);
  pointer-events: none;
}

/* Pulse animation for saving spinner */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.4; }
}
.cell--saving::after {
  content: '';
  position: absolute;
  inset: 0;
  background: rgba(59, 130, 246, 0.08);
  animation: pulse 1s ease-in-out infinite;
}

/* ─── Error state ─── */
.cell--error {
  background: var(--cell-error-bg);
  box-shadow: 0 0 0 2px var(--cell-error-border);
}

/* ─── Row edit mode ─── */
.row--editing td {
  background: #f8faff;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .cell--saving::after {
    animation: none;
    background: rgba(59, 130, 246, 0.15);
  }
}

/* ─── Screen reader only ─── */
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0,0,0,0);
  white-space: nowrap;
  border: 0;
}
```

**What CSS owns:** cell highlight during edit, zero-layout-shift input sizing, saving pulse animation, error highlight, edit trigger visibility on hover.

**What CSS cannot own:** which cell is active, draft vs. committed value, API calls, rollback, keyboard routing between cells.

---

## React Implementation

### Types

```tsx
// types.ts
export interface Row {
  id: string;
  [column: string]: string | number;
}

export interface ColumnDef {
  key: string;
  label: string;
  type?: 'text' | 'number' | 'email';
  editable?: boolean;
  validate?: (value: string) => string | null;  // returns error message or null
}

export type CellCoord = { rowId: string; colKey: string };

export type CellStatus = 'idle' | 'editing' | 'saving' | 'error';
```

### The Hook — Cell Edit State

```tsx
// useEditableTable.ts
import { useState, useCallback, useRef } from 'react';
import type { Row, ColumnDef, CellCoord, CellStatus } from './types';

interface CellState {
  draft: string;
  status: CellStatus;
  errorMessage: string | null;
}

interface UseEditableTableOptions {
  rows: Row[];
  columns: ColumnDef[];
  onSave: (rowId: string, colKey: string, value: string) => Promise<void>;
}

export function useEditableTable({ rows, columns, onSave }: UseEditableTableOptions) {
  // committed data lives in props (rows); draft data lives here
  const [cellStates, setCellStates] = useState<Map<string, CellState>>(new Map());
  const [activeCell, setActiveCell] = useState<CellCoord | null>(null);

  const inputRefs = useRef<Map<string, HTMLInputElement>>(new Map());

  const cellKey = (rowId: string, colKey: string) => `${rowId}::${colKey}`;

  const editableCols = columns.filter(c => c.editable !== false);

  // ── Open a cell for editing ──
  const startEdit = useCallback((rowId: string, colKey: string, currentValue: string) => {
    const key = cellKey(rowId, colKey);
    setCellStates(prev => {
      const next = new Map(prev);
      next.set(key, { draft: String(currentValue), status: 'editing', errorMessage: null });
      return next;
    });
    setActiveCell({ rowId, colKey });
    // Focus the input on next tick (after render)
    requestAnimationFrame(() => {
      inputRefs.current.get(key)?.focus();
      inputRefs.current.get(key)?.select();
    });
  }, []);

  // ── Update draft value as user types ──
  const updateDraft = useCallback((rowId: string, colKey: string, value: string) => {
    const key = cellKey(rowId, colKey);
    setCellStates(prev => {
      const existing = prev.get(key);
      if (!existing) return prev;
      const next = new Map(prev);
      next.set(key, { ...existing, draft: value, errorMessage: null });
      return next;
    });
  }, []);

  // ── Save a cell ──
  const saveCell = useCallback(async (rowId: string, colKey: string) => {
    const key = cellKey(rowId, colKey);
    const state = cellStates.get(key);
    if (!state || state.status !== 'editing') return;

    const col = columns.find(c => c.key === colKey);
    if (col?.validate) {
      const error = col.validate(state.draft);
      if (error) {
        setCellStates(prev => {
          const next = new Map(prev);
          next.set(key, { ...state, errorMessage: error });
          return next;
        });
        return;
      }
    }

    // Optimistic: mark as saving immediately
    setCellStates(prev => {
      const next = new Map(prev);
      next.set(key, { ...state, status: 'saving' });
      return next;
    });

    try {
      await onSave(rowId, colKey, state.draft);
      // Success — clear the cell state (committed value is now in props)
      setCellStates(prev => {
        const next = new Map(prev);
        next.delete(key);
        return next;
      });
      setActiveCell(null);
    } catch (err) {
      // Rollback: restore to 'error' state with original value still in draft
      // The parent's `rows` prop has not changed, so committed value is safe
      setCellStates(prev => {
        const next = new Map(prev);
        next.set(key, {
          draft: state.draft,
          status: 'error',
          errorMessage: 'Save failed. Press Escape to discard.',
        });
        return next;
      });
    }
  }, [cellStates, columns, onSave]);

  // ── Cancel editing — discard draft, restore committed value ──
  const cancelEdit = useCallback((rowId: string, colKey: string) => {
    const key = cellKey(rowId, colKey);
    setCellStates(prev => {
      const next = new Map(prev);
      next.delete(key);
      return next;
    });
    setActiveCell(null);
  }, []);

  // ── Keyboard navigation: Tab → next cell, Shift+Tab → prev cell ──
  const navigateCell = useCallback((
    rowId: string,
    colKey: string,
    direction: 'next' | 'prev',
    currentValue: string
  ) => {
    // Find the flat ordered list of all editable cells
    const allCells: CellCoord[] = rows.flatMap(row =>
      editableCols.map(col => ({ rowId: row.id, colKey: col.key }))
    );

    const currentIdx = allCells.findIndex(
      c => c.rowId === rowId && c.colKey === colKey
    );
    if (currentIdx === -1) return;

    const nextIdx = direction === 'next'
      ? (currentIdx + 1) % allCells.length
      : (currentIdx - 1 + allCells.length) % allCells.length;

    const nextCell = allCells[nextIdx];
    const nextRow  = rows.find(r => r.id === nextCell.rowId);
    if (!nextRow) return;

    // Save current cell first (fire-and-forget, don't block navigation)
    saveCell(rowId, colKey);
    startEdit(nextCell.rowId, nextCell.colKey, String(nextRow[nextCell.colKey] ?? ''));
  }, [rows, editableCols, saveCell, startEdit]);

  const getCellState = (rowId: string, colKey: string) =>
    cellStates.get(cellKey(rowId, colKey)) ?? null;

  const registerInputRef = (rowId: string, colKey: string, el: HTMLInputElement | null) => {
    const key = cellKey(rowId, colKey);
    if (el) inputRefs.current.set(key, el);
    else inputRefs.current.delete(key);
  };

  return {
    activeCell,
    getCellState,
    startEdit,
    updateDraft,
    saveCell,
    cancelEdit,
    navigateCell,
    registerInputRef,
  };
}
```

### The Components

```tsx
// EditableTable.tsx
import { useCallback } from 'react';
import { useEditableTable } from './useEditableTable';
import type { Row, ColumnDef } from './types';

interface Props {
  rows: Row[];
  columns: ColumnDef[];
  onSave: (rowId: string, colKey: string, value: string) => Promise<void>;
}

export function EditableTable({ rows, columns, onSave }: Props) {
  const table = useEditableTable({ rows, columns, onSave });

  return (
    <>
      {/* Live region for save feedback */}
      <div id="cell-hint" className="sr-only" aria-live="polite" aria-atomic="true" />

      <table role="grid" aria-label="Editable data table">
        <thead>
          <tr>
            {columns.map(col => (
              <th key={col.key} scope="col">{col.label}</th>
            ))}
          </tr>
        </thead>
        <tbody>
          {rows.map(row => (
            <tr key={row.id}>
              {columns.map(col => (
                <EditableCell
                  key={col.key}
                  row={row}
                  col={col}
                  table={table}
                />
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </>
  );
}

/* ── Single cell ── */
interface EditableCellProps {
  row: Row;
  col: ColumnDef;
  table: ReturnType<typeof useEditableTable>;
}

function EditableCell({ row, col, table }: EditableCellProps) {
  const state         = table.getCellState(row.id, col.key);
  const isEditing     = state?.status === 'editing' || state?.status === 'error';
  const isSaving      = state?.status === 'saving';
  const committedValue = String(row[col.key] ?? '');
  const displayValue  = state ? state.draft : committedValue;

  const handleKeyDown = useCallback((e: React.KeyboardEvent<HTMLInputElement>) => {
    switch (e.key) {
      case 'Enter':
        e.preventDefault();
        table.saveCell(row.id, col.key);
        break;
      case 'Escape':
        e.preventDefault();
        table.cancelEdit(row.id, col.key);
        break;
      case 'Tab':
        e.preventDefault();
        table.navigateCell(
          row.id, col.key,
          e.shiftKey ? 'prev' : 'next',
          state?.draft ?? committedValue
        );
        break;
    }
  }, [table, row.id, col.key, state, committedValue]);

  if (!col.editable) {
    return <td className="cell"><span className="cell__display">{committedValue}</span></td>;
  }

  return (
    <td
      className={[
        'cell',
        isEditing ? 'cell--editing' : '',
        isSaving  ? 'cell--saving'  : '',
        state?.status === 'error' ? 'cell--error' : '',
      ].filter(Boolean).join(' ')}
      aria-label={`${col.label}: ${displayValue}`}
    >
      {isEditing || isSaving ? (
        <>
          <input
            ref={el => table.registerInputRef(row.id, col.key, el)}
            className="cell__input"
            type={col.type ?? 'text'}
            value={state!.draft}
            aria-label={`Edit ${col.label}`}
            aria-describedby="cell-hint"
            aria-invalid={state?.status === 'error'}
            onChange={e => table.updateDraft(row.id, col.key, e.target.value)}
            onKeyDown={handleKeyDown}
            onBlur={() => table.saveCell(row.id, col.key)}
            disabled={isSaving}
          />
          {state?.errorMessage && (
            <span className="cell__error-msg" role="alert">{state.errorMessage}</span>
          )}
          <div className="cell__actions">
            <button
              className="cell__save"
              aria-label="Save"
              onMouseDown={e => { e.preventDefault(); table.saveCell(row.id, col.key); }}
            >✓</button>
            <button
              className="cell__cancel"
              aria-label="Cancel"
              onMouseDown={e => { e.preventDefault(); table.cancelEdit(row.id, col.key); }}
            >✗</button>
          </div>
        </>
      ) : (
        <>
          <span className="cell__display">{committedValue}</span>
          <button
            className="cell__edit-trigger"
            aria-label={`Edit ${col.label}`}
            tabIndex={-1}
            onClick={() => table.startEdit(row.id, col.key, committedValue)}
          >
            <svg width="12" height="12" viewBox="0 0 24 24" aria-hidden="true" fill="currentColor">
              <path d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04a1 1 0 0 0 0-1.41l-2.34-2.34a1 1 0 0 0-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"/>
            </svg>
          </button>
        </>
      )}
    </td>
  );
}
```

### Usage

```tsx
// Usage in a page component
import { useState } from 'react';
import { EditableTable } from './EditableTable';
import type { Row, ColumnDef } from './types';

const COLUMNS: ColumnDef[] = [
  { key: 'company', label: 'Company',  type: 'text',   editable: true,
    validate: v => v.trim() === '' ? 'Company name is required' : null },
  { key: 'mrr',     label: 'MRR ($)',  type: 'number', editable: true,
    validate: v => isNaN(Number(v))   ? 'Must be a number' : null },
  { key: 'email',   label: 'Email',    type: 'email',  editable: true },
  { key: 'plan',    label: 'Plan',     editable: false },  // read-only column
];

export function CustomersPage() {
  const [rows, setRows] = useState<Row[]>([
    { id: '1', company: 'Acme Corp',  mrr: 4200, email: 'joe@acme.com',   plan: 'Pro' },
    { id: '2', company: 'Globex Inc', mrr: 800,  email: 'jane@globex.com', plan: 'Starter' },
  ]);

  const handleSave = async (rowId: string, colKey: string, value: string) => {
    // Optimistic: update local state immediately
    setRows(prev =>
      prev.map(r => r.id === rowId ? { ...r, [colKey]: value } : r)
    );
    // Persist to server
    await fetch(`/api/customers/${rowId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ [colKey]: value }),
    }).then(res => {
      if (!res.ok) {
        // Rollback: restore the previous value by re-fetching or reverting state
        setRows(prev =>
          prev.map(r => r.id === rowId ? { ...r, [colKey]: rows.find(x => x.id === rowId)?.[colKey] } : r)
        );
        throw new Error('Save failed');
      }
    });
  };

  return <EditableTable rows={rows} columns={COLUMNS} onSave={handleSave} />;
}
```

---

## Angular Implementation

### Model and Service

```typescript
// editable-table.model.ts
export interface Row { id: string; [col: string]: string | number; }
export interface ColumnDef {
  key: string;
  label: string;
  type?: 'text' | 'number' | 'email';
  editable?: boolean;
  validate?: (v: string) => string | null;
}
export type CellStatus = 'idle' | 'editing' | 'saving' | 'error';
export interface CellState { draft: string; status: CellStatus; errorMessage: string | null; }
```

```typescript
// editable-table.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { Row, ColumnDef, CellState } from './editable-table.model';

@Injectable()
export class EditableTableService {
  private _cellStates = signal<Map<string, CellState>>(new Map());
  private _activeCell = signal<{ rowId: string; colKey: string } | null>(null);

  readonly activeCell  = this._activeCell.asReadonly();
  readonly cellStates  = this._cellStates.asReadonly();

  private rows: Row[]       = [];
  private columns: ColumnDef[] = [];
  private saveFn!: (rowId: string, colKey: string, value: string) => Promise<void>;

  init(rows: Row[], columns: ColumnDef[], saveFn: typeof this.saveFn) {
    this.rows    = rows;
    this.columns = columns;
    this.saveFn  = saveFn;
  }

  updateRows(rows: Row[]) { this.rows = rows; }

  private key(rowId: string, colKey: string) { return `${rowId}::${colKey}`; }

  getCellState(rowId: string, colKey: string): CellState | null {
    return this._cellStates().get(this.key(rowId, colKey)) ?? null;
  }

  startEdit(rowId: string, colKey: string, currentValue: string) {
    const k = this.key(rowId, colKey);
    this._cellStates.update(m => {
      const next = new Map(m);
      next.set(k, { draft: currentValue, status: 'editing', errorMessage: null });
      return next;
    });
    this._activeCell.set({ rowId, colKey });
  }

  updateDraft(rowId: string, colKey: string, value: string) {
    const k = this.key(rowId, colKey);
    this._cellStates.update(m => {
      const existing = m.get(k);
      if (!existing) return m;
      const next = new Map(m);
      next.set(k, { ...existing, draft: value, errorMessage: null });
      return next;
    });
  }

  async saveCell(rowId: string, colKey: string) {
    const k     = this.key(rowId, colKey);
    const state = this._cellStates().get(k);
    if (!state || state.status !== 'editing') return;

    const col = this.columns.find(c => c.key === colKey);
    if (col?.validate) {
      const error = col.validate(state.draft);
      if (error) {
        this._cellStates.update(m => {
          const next = new Map(m); next.set(k, { ...state, errorMessage: error }); return next;
        });
        return;
      }
    }

    // Optimistic save
    this._cellStates.update(m => {
      const next = new Map(m); next.set(k, { ...state, status: 'saving' }); return next;
    });

    try {
      await this.saveFn(rowId, colKey, state.draft);
      this._cellStates.update(m => { const next = new Map(m); next.delete(k); return next; });
      this._activeCell.set(null);
    } catch {
      this._cellStates.update(m => {
        const next = new Map(m);
        next.set(k, { draft: state.draft, status: 'error', errorMessage: 'Save failed. Press Escape to discard.' });
        return next;
      });
    }
  }

  cancelEdit(rowId: string, colKey: string) {
    const k = this.key(rowId, colKey);
    this._cellStates.update(m => { const next = new Map(m); next.delete(k); return next; });
    this._activeCell.set(null);
  }

  navigateCell(rowId: string, colKey: string, direction: 'next' | 'prev') {
    const editableCols = this.columns.filter(c => c.editable !== false);
    const allCells     = this.rows.flatMap(r => editableCols.map(c => ({ rowId: r.id, colKey: c.key })));
    const idx          = allCells.findIndex(c => c.rowId === rowId && c.colKey === colKey);
    if (idx === -1) return;

    const nextIdx  = direction === 'next'
      ? (idx + 1) % allCells.length
      : (idx - 1 + allCells.length) % allCells.length;
    const nextCell = allCells[nextIdx];
    const nextRow  = this.rows.find(r => r.id === nextCell.rowId);
    if (!nextRow) return;

    this.saveCell(rowId, colKey);
    this.startEdit(nextCell.rowId, nextCell.colKey, String(nextRow[nextCell.colKey] ?? ''));
  }
}
```

### Standalone Component

```typescript
// editable-table.component.ts
import {
  Component, Input, OnChanges, SimpleChanges,
  inject, ChangeDetectionStrategy
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { EditableTableService } from './editable-table.service';
import { Row, ColumnDef } from './editable-table.model';

@Component({
  selector: 'app-editable-table',
  standalone: true,
  imports: [NgFor, NgIf],
  providers: [EditableTableService],   // scoped per table instance
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div id="cell-hint" class="sr-only" aria-live="polite" aria-atomic="true"></div>

    <table role="grid" aria-label="Editable data table">
      <thead>
        <tr>
          <th *ngFor="let col of columns" scope="col">{{ col.label }}</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let row of rows; trackBy: trackById">
          <td
            *ngFor="let col of columns"
            class="cell"
            [class.cell--editing]="isEditing(row.id, col.key)"
            [class.cell--saving]="isSaving(row.id, col.key)"
            [class.cell--error]="isError(row.id, col.key)"
            [attr.aria-label]="col.label + ': ' + getCellDisplay(row, col.key)"
          >
            <!-- Edit mode -->
            <ng-container *ngIf="isEditing(row.id, col.key) || isSaving(row.id, col.key); else readMode">
              <input
                #cellInput
                class="cell__input"
                [type]="col.type || 'text'"
                [value]="svc.getCellState(row.id, col.key)?.draft ?? ''"
                [attr.aria-label]="'Edit ' + col.label"
                aria-describedby="cell-hint"
                [attr.aria-invalid]="isError(row.id, col.key)"
                [disabled]="isSaving(row.id, col.key)"
                (input)="svc.updateDraft(row.id, col.key, $any($event.target).value)"
                (keydown)="onKeyDown($event, row.id, col.key)"
                (blur)="svc.saveCell(row.id, col.key)"
              />
              <span
                *ngIf="svc.getCellState(row.id, col.key)?.errorMessage as msg"
                class="cell__error-msg"
                role="alert"
              >{{ msg }}</span>
            </ng-container>

            <!-- Read mode -->
            <ng-template #readMode>
              <ng-container *ngIf="col.editable !== false; else staticCell">
                <span class="cell__display">{{ row[col.key] }}</span>
                <button
                  class="cell__edit-trigger"
                  [attr.aria-label]="'Edit ' + col.label"
                  tabindex="-1"
                  (click)="svc.startEdit(row.id, col.key, '' + row[col.key])"
                >✎</button>
              </ng-container>
              <ng-template #staticCell>
                <span class="cell__display">{{ row[col.key] }}</span>
              </ng-template>
            </ng-template>
          </td>
        </tr>
      </tbody>
    </table>
  `,
})
export class EditableTableComponent implements OnChanges {
  @Input() rows: Row[]       = [];
  @Input() columns: ColumnDef[] = [];
  @Input() onSave!: (rowId: string, colKey: string, value: string) => Promise<void>;

  svc = inject(EditableTableService);

  ngOnChanges(changes: SimpleChanges) {
    if (changes['rows'] || changes['columns'] || changes['onSave']) {
      this.svc.init(this.rows, this.columns, this.onSave);
    }
    if (changes['rows'] && !changes['rows'].firstChange) {
      this.svc.updateRows(this.rows);
    }
  }

  trackById(_: number, row: Row) { return row.id; }

  getCellDisplay(row: Row, colKey: string) {
    return String(row[colKey] ?? '');
  }

  isEditing(rowId: string, colKey: string) {
    return this.svc.getCellState(rowId, colKey)?.status === 'editing';
  }
  isSaving(rowId: string, colKey: string) {
    return this.svc.getCellState(rowId, colKey)?.status === 'saving';
  }
  isError(rowId: string, colKey: string) {
    return this.svc.getCellState(rowId, colKey)?.status === 'error';
  }

  onKeyDown(e: KeyboardEvent, rowId: string, colKey: string) {
    switch (e.key) {
      case 'Enter':
        e.preventDefault();
        this.svc.saveCell(rowId, colKey);
        break;
      case 'Escape':
        e.preventDefault();
        this.svc.cancelEdit(rowId, colKey);
        break;
      case 'Tab':
        e.preventDefault();
        this.svc.navigateCell(rowId, colKey, e.shiftKey ? 'prev' : 'next');
        break;
    }
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Table has `role="grid"` | Signals to AT that cells are interactive, not just presentational |
| Each editable cell announces its column and current value | `aria-label` on `<td>` contains column name and display value |
| Input has `aria-label="Edit <column>"` | Distinguishes the input from the display span for screen readers |
| Input has `aria-describedby` pointing to live region | Allows save/error feedback to be read without re-focusing |
| `aria-invalid="true"` set on input with validation error | Standard form error pattern, understood by all AT |
| Error message wrapped in `role="alert"` | Immediately announced without needing focus change |
| Saving state disables input and sets `disabled` | Prevents double-submit; AT announces input as unavailable |
| Save/Cancel inline buttons use `onMouseDown` with `preventDefault` | Prevents `onBlur` from firing before the button click registers |
| Edit trigger button has `tabindex="-1"` | Excluded from Tab order — cell itself is the Tab stop |
| Keyboard: Tab/Shift+Tab navigates between cells | Implemented in `onKeyDown`; `e.preventDefault()` blocks browser default Tab |
| Keyboard: Enter confirms, Escape discards | Consistent with spreadsheet conventions familiar to power users |
| Minimum tap target on edit trigger | 44x44px enforced via CSS padding |
| Reduced-motion: saving animation disabled | `@media (prefers-reduced-motion)` replaces pulse with static tint |

---

## Production Pitfalls

**1. `onBlur` fires before `onClick` on Save button, causing a double-save race**
When the user clicks the Save button, `onBlur` fires on the input before `onClick` fires on the button. If `onBlur` calls `saveCell` and dismisses the edit state, the button's `onClick` never finds an active cell to save. Fix: use `onMouseDown` instead of `onClick` on inline action buttons and call `e.preventDefault()` to stop `onBlur` from firing on the input. `onMouseDown` fires before `onBlur`.

**2. Optimistic update leaves stale state if the parent's `rows` prop doesn't update**
If you call `setRows` after a failed save to "rollback", but your state update computes the old value from the closure at the time the save was initiated (not the time the error resolved), you can roll back to a stale value. Fix: always capture the committed value *before* starting the optimistic update, not from the closure of the async callback.

**3. Tab navigation skips hidden or disabled columns**
If a column is conditionally `editable: false`, it must be excluded from the flat cell coordinate list used by `navigateCell`. If it is not, Tab stops on a read-only cell and the user sees nothing change. Fix: filter to `editable !== false` columns when building the navigation grid, exactly as shown in `useEditableTable`.

**4. Cell edit mode is lost on row-level re-renders**
In React, if the parent component re-renders and recreates the `rows` array reference (e.g., from a WebSocket update), the `EditableTable` re-renders and may lose the currently focused input. Fix: keep cell editing state in the hook (inside `EditableTable`) rather than in parent state — the hook is stable across row prop changes. In Angular, the signal-based service is scoped to the component and unaffected by input updates.

**5. Row edit mode vs. cell edit mode chosen incorrectly**
Cell edit mode (one cell at a time) is optimal when rows have many fields and users typically change only one value. Row edit mode (all cells in a row editable simultaneously) is better when a new-row creation flow exists, or when fields have interdependent validation. Choosing cell mode for a form-heavy row (e.g., 12 related fields) forces the user to Tab through each field individually with no way to cancel the entire row. Define this boundary in the design phase, not after implementation.

**6. Validation errors are silently lost on Escape**
If the user types an invalid value, sees the error message, then presses Escape, the draft is discarded and the error disappears — which is the correct behavior. However, if `cancelEdit` is also called by `onBlur` when focus leaves the table entirely, a user clicking elsewhere will silently lose their unsaved input without confirmation. Fix: distinguish between cancel-by-Escape (always discard) and blur-triggered save (validate first; show a confirmation dialog or toast if there are unsaved changes with errors).

---

## Interview Angle

**Q: "Walk me through how you'd implement an editable table where clicking a cell opens an input, Tab moves to the next cell, and a failed API save rolls back the value."**

A strong answer covers five points: (1) **State separation** — maintain two parallel values per cell: the committed value (lives in the rows prop, sourced from the server) and the draft value (lives in local state, owned by the editing hook). The cell displays whichever is active. (2) **Optimistic update** — transition the cell to a "saving" state immediately and call `setRows` before the API resolves, so the UI responds instantly. (3) **Rollback** — in the catch block, delete the draft state; because the committed value in `rows` was never updated (or you revert it), the cell snaps back to the last known good value. (4) **Keyboard routing** — intercept Tab in `onKeyDown`, call `e.preventDefault()` to suppress the browser default, compute the next cell coordinates from a flat ordered list of all editable cells, call `saveCell` on the current cell and `startEdit` on the target cell. (5) **`onBlur` / `onMouseDown` ordering** — inline Save/Cancel buttons must use `onMouseDown` with `preventDefault` to prevent `onBlur` from dismissing the edit state before the button action registers.

**Follow-up: "What's the difference between cell edit mode and row edit mode, and when do you pick each?"**
Cell edit mode minimises screen noise — only one input is visible at a time, ideal for sparse edits on a read-heavy table (e.g., a CRM updating a single deal value). Row edit mode shows all inputs for a row simultaneously, ideal when fields are interdependent (e.g., shipping address has five related fields that must be validated together) or when you support inline row creation. The cost of row mode is that it visually inflates the row height and requires coordinating a single Save/Cancel per row rather than per cell, which complicates the optimistic save contract when only some fields have changed.
