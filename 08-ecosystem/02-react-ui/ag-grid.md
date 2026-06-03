# AG Grid

## What AG Grid Solves

AG Grid is a high-performance data grid built for enterprise use cases where standard HTML tables or lightweight libraries fall apart:

- **100k+ rows** in the DOM without freezing — achieved through row and column virtualisation
- **Complex interactions**: multi-column sort, pivot tables, tree data, row grouping, aggregation
- **Server-driven data**: built-in Server-Side Row Model with lazy loading and partial result sets
- **Rich editing**: inline cell editing, custom cell editors, undo/redo
- **Excel-grade exports** with formatting (Enterprise)
- **Built-in charts** from grid data (Enterprise)

The core promise: everything a data-heavy internal app needs, with a consistent API across React, Angular, and Vue.

---

## Community vs Enterprise

| Feature | Community | Enterprise |
|---|---|---|
| Sorting, filtering, pagination | Yes | Yes |
| Infinite / Server-Side Row Model | Yes | Yes |
| Row grouping & aggregation | No | Yes |
| Pivot tables | No | Yes |
| Master / Detail (expandable rows) | No | Yes |
| Excel export (`.xlsx`) | No | Yes |
| CSV export | Yes | Yes |
| Built-in charts | No | Yes |
| Column menu with filter panel | No | Yes |
| Set filter (multi-select filter) | No | Yes |
| Row dragging / ordering | Yes | Yes |
| License key required | No | Yes |

Enterprise is activated by registering a license key — the grid still works without one (watermark shown).

```tsx
import { LicenseManager } from 'ag-grid-enterprise';
LicenseManager.setLicenseKey('YOUR_LICENSE_KEY');
```

---

## React Integration

### Installation

```bash
# Community
npm install ag-grid-community ag-grid-react

# Enterprise (includes community)
npm install ag-grid-community ag-grid-enterprise ag-grid-react
```

### Module Registration (v30+)

AG Grid v30 introduced modular registration — import only what you use to keep bundle size down.

```tsx
import { ModuleRegistry, AllCommunityModule } from 'ag-grid-community';
import { RowGroupingModule, ExcelExportModule } from 'ag-grid-enterprise';

// Register once at app startup (e.g. main.tsx)
ModuleRegistry.registerModules([
  AllCommunityModule,
  RowGroupingModule,
  ExcelExportModule,
]);
```

### Basic AgGridReact Component

```tsx
import { AgGridReact } from 'ag-grid-react';
import { ColDef } from 'ag-grid-community';
import 'ag-grid-community/styles/ag-grid.css';
import 'ag-grid-community/styles/ag-theme-quartz.css';

interface Trade {
  id: string;
  ticker: string;
  quantity: number;
  price: number;
  status: 'open' | 'closed' | 'pending';
}

const columnDefs: ColDef<Trade>[] = [
  { field: 'ticker', headerName: 'Ticker', sortable: true, filter: true },
  { field: 'quantity', headerName: 'Qty', type: 'numericColumn' },
  { field: 'price', headerName: 'Price', valueFormatter: p => `$${p.value.toFixed(2)}` },
  { field: 'status', headerName: 'Status' },
];

const rowData: Trade[] = [
  { id: '1', ticker: 'AAPL', quantity: 100, price: 182.5, status: 'open' },
  { id: '2', ticker: 'MSFT', quantity: 200, price: 415.3, status: 'closed' },
];

export function TradeGrid() {
  return (
    <div className="ag-theme-quartz" style={{ height: 400, width: '100%' }}>
      <AgGridReact
        rowData={rowData}
        columnDefs={columnDefs}
        getRowId={p => p.data.id}   // stable identity for row diffing
      />
    </div>
  );
}
```

### Grid API via Ref

The `gridRef` gives you imperative access to the Grid API — used for programmatic sorting, filtering, exporting, selection, etc.

```tsx
import { useRef, useCallback } from 'react';
import { AgGridReact } from 'ag-grid-react';
import { GridApi, GridReadyEvent } from 'ag-grid-community';

export function TradeGrid() {
  const gridRef = useRef<AgGridReact>(null);

  const onGridReady = useCallback((event: GridReadyEvent) => {
    // event.api is also available here; same reference as gridRef.current.api
    event.api.sizeColumnsToFit();
  }, []);

  const exportCsv = useCallback(() => {
    gridRef.current?.api.exportDataAsCsv({ fileName: 'trades.csv' });
  }, []);

  const clearFilters = useCallback(() => {
    gridRef.current?.api.setFilterModel(null);
  }, []);

  return (
    <>
      <button onClick={exportCsv}>Export CSV</button>
      <button onClick={clearFilters}>Clear Filters</button>
      <div className="ag-theme-quartz" style={{ height: 400 }}>
        <AgGridReact
          ref={gridRef}
          rowData={rowData}
          columnDefs={columnDefs}
          onGridReady={onGridReady}
        />
      </div>
    </>
  );
}
```

---

## Column Definitions

### Core Properties

```tsx
const columnDefs: ColDef<Trade>[] = [
  {
    field: 'ticker',           // maps to rowData property
    headerName: 'Symbol',      // label shown in header
    width: 120,
    minWidth: 80,
    maxWidth: 200,
    flex: 1,                   // like CSS flex — shares available width
    pinned: 'left',            // freeze column: 'left' | 'right'
    lockPinned: true,          // prevent user from unpinning
    sortable: true,
    sort: 'asc',               // initial sort direction
    sortIndex: 0,              // order in multi-column sort
    filter: true,              // enable default filter for column type
    resizable: true,
    hide: false,
  },
];
```

### valueFormatter — Display Transformation

`valueFormatter` changes what is displayed without touching the underlying data. Used for numbers, dates, currency.

```tsx
{
  field: 'price',
  valueFormatter: (params) => {
    if (params.value == null) return '';
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
    }).format(params.value);
  },
}
```

### valueGetter — Computed Values

```tsx
{
  headerName: 'Total Value',
  valueGetter: (params) => params.data.quantity * params.data.price,
  valueFormatter: (params) => `$${params.value.toLocaleString()}`,
}
```

### cellRenderer — React Component as Cell Renderer

Pass a React component directly. AG Grid will mount it per visible cell.

```tsx
import { ICellRendererParams } from 'ag-grid-community';

interface StatusCellProps extends ICellRendererParams<Trade> {}

function StatusCell({ value }: StatusCellProps) {
  const colours = {
    open: 'bg-green-100 text-green-800',
    closed: 'bg-gray-100 text-gray-600',
    pending: 'bg-yellow-100 text-yellow-800',
  };
  return (
    <span className={`px-2 py-0.5 rounded text-xs font-medium ${colours[value]}`}>
      {value}
    </span>
  );
}

// In column def:
{
  field: 'status',
  cellRenderer: StatusCell,
  // cellRendererParams can pass static extra props:
  cellRendererParams: { someExtra: 'value' },
}
```

### cellRendererSelector — Different Renderers per Row

```tsx
{
  field: 'value',
  cellRendererSelector: (params) => {
    if (params.data.type === 'sparkline') {
      return { component: SparklineCell };
    }
    return { component: 'agAnimateShowChangeCellRenderer' }; // built-in
  },
}
```

---

## Row Models

AG Grid has four row models. Choose based on data volume and where logic lives.

### 1. Client-Side Row Model (default)

All data is loaded upfront in the browser. Sorting, filtering, pagination are done in-memory.

**Use when**: data fits in memory (up to ~100k rows is practical), or you want the simplest setup.

```tsx
<AgGridReact
  rowModelType="clientSide"   // default, can be omitted
  rowData={allRows}
  columnDefs={columnDefs}
/>
```

### 2. Infinite Scroll Row Model

Data is fetched in blocks as the user scrolls. The datasource fetches one block at a time.

**Use when**: read-only list that grows large; simple "load more as you scroll" pattern.

```tsx
import { IDatasource, IGetRowsParams } from 'ag-grid-community';

const datasource: IDatasource = {
  getRows(params: IGetRowsParams) {
    fetch(`/api/trades?start=${params.startRow}&end=${params.endRow}`)
      .then(r => r.json())
      .then(data => {
        params.successCallback(data.rows, data.totalCount);
      })
      .catch(() => params.failCallback());
  },
};

<AgGridReact
  rowModelType="infinite"
  datasource={datasource}
  cacheBlockSize={100}       // rows per block fetch
  maxBlocksInCache={10}      // how many blocks to keep in memory
  columnDefs={columnDefs}
/>
```

### 3. Server-Side Row Model (Enterprise)

The most powerful model. Supports server-driven sorting, filtering, grouping, and pagination. The grid tells the server exactly what slice of data is needed.

**Use when**: millions of rows, server-side pivoting/aggregation, or regulated data that must not be fully loaded client-side.

### 4. Viewport Row Model (Enterprise)

Only the visible viewport rows are maintained. Server always knows which rows are visible.

**Use when**: real-time data feeds (trading, monitoring) where you want the server to push updates only for visible rows.

---

## Server-Side Row Model Deep Dive

### IServerSideDatasource Interface

```tsx
import {
  IServerSideDatasource,
  IServerSideGetRowsParams,
  IServerSideGetRowsRequest,
} from 'ag-grid-community';

const serverDatasource: IServerSideDatasource = {
  getRows(params: IServerSideGetRowsParams) {
    const request: IServerSideGetRowsRequest = params.request;

    // request contains everything the server needs:
    // request.startRow, request.endRow
    // request.sortModel  — [{ colId: 'price', sort: 'desc' }]
    // request.filterModel — { ticker: { type: 'contains', filter: 'AA' } }
    // request.rowGroupCols, request.valueCols (for aggregation)
    // request.groupKeys — path of expanded group nodes

    fetch('/api/trades/ssrm', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(request),
    })
      .then(r => r.json())
      .then(response => {
        params.success({
          rowData: response.rows,
          rowCount: response.totalCount,  // omit if unknown (enables infinite scroll mode)
        });
      })
      .catch(() => params.fail());
  },
};

<AgGridReact
  rowModelType="serverSide"
  serverSideDatasource={serverDatasource}
  cacheBlockSize={50}
  columnDefs={columnDefs}
/>
```

### SSRM with Pagination

```tsx
<AgGridReact
  rowModelType="serverSide"
  serverSideDatasource={serverDatasource}
  pagination={true}
  paginationPageSize={25}
  cacheBlockSize={25}        // align with page size for clean fetches
  columnDefs={columnDefs}
/>
```

### SSRM with Row Grouping (Enterprise)

When the user expands a group, `params.request.groupKeys` contains the path of the expanded node. Your server needs to handle this.

```tsx
// Server receives groupKeys: [] for top level, ['AAPL'] for expanded AAPL group
// request.rowGroupCols: [{ id: 'ticker', field: 'ticker', displayName: 'Ticker' }]

const serverDatasource: IServerSideDatasource = {
  getRows(params) {
    const { groupKeys, rowGroupCols } = params.request;
    const isGroupLevel = rowGroupCols.length > groupKeys.length;

    if (isGroupLevel) {
      // return group summary rows
      fetchGroupSummaries(params.request).then(data =>
        params.success({ rowData: data.groups, rowCount: data.groups.length })
      );
    } else {
      // return leaf rows for the current group
      fetchLeafRows(params.request).then(data =>
        params.success({ rowData: data.rows, rowCount: data.total })
      );
    }
  },
};
```

---

## Sorting and Filtering

### Built-in Column Filters

```tsx
// Text filter (default for string columns)
{ field: 'ticker', filter: 'agTextColumnFilter' }

// Number filter
{ field: 'price', filter: 'agNumberColumnFilter' }

// Date filter
{ field: 'tradeDate', filter: 'agDateColumnFilter' }

// Set filter — multi-select (Enterprise)
{ field: 'status', filter: 'agSetColumnFilter' }

// Quick shorthand — AG Grid infers correct filter from column type:
{ field: 'price', filter: true }
```

### Filter Params

```tsx
{
  field: 'price',
  filter: 'agNumberColumnFilter',
  filterParams: {
    filterOptions: ['equals', 'greaterThan', 'lessThan', 'inRange'],
    defaultOption: 'greaterThan',
    buttons: ['reset', 'apply'],  // show Apply button instead of instant filter
    closeOnApply: true,
  },
}
```

### Custom Filter Component

```tsx
import { IFilterComp, IFilterParams, IDoesFilterPassParams } from 'ag-grid-community';

// AG Grid custom filters use class-based API (not hooks) as of v31
// For React functional component filters, use IFilterReactComp interface via forwardRef

import { forwardRef, useImperativeHandle, useState } from 'react';

const QuickStatusFilter = forwardRef((props: IFilterParams, ref) => {
  const [selected, setSelected] = useState<string | null>(null);

  useImperativeHandle(ref, () => ({
    isFilterActive: () => selected !== null,
    doesFilterPass: (params: IDoesFilterPassParams) =>
      selected === null || params.data.status === selected,
    getModel: () => (selected ? { value: selected } : null),
    setModel: (model: any) => setSelected(model?.value ?? null),
  }));

  return (
    <div className="p-2 flex flex-col gap-1">
      {['open', 'closed', 'pending'].map(s => (
        <label key={s} className="flex items-center gap-2 cursor-pointer">
          <input
            type="radio"
            name="status"
            checked={selected === s}
            onChange={() => {
              setSelected(s);
              props.filterChangedCallback();  // tell grid filter changed
            }}
          />
          {s}
        </label>
      ))}
      <button onClick={() => { setSelected(null); props.filterChangedCallback(); }}>
        Clear
      </button>
    </div>
  );
});

// In column def:
{ field: 'status', filter: QuickStatusFilter }
```

### External / Quick Filter

```tsx
const gridRef = useRef<AgGridReact>(null);
const [quickFilter, setQuickFilter] = useState('');

// The quickFilterText prop applies a cross-column text search
<input
  value={quickFilter}
  onChange={e => setQuickFilter(e.target.value)}
  placeholder="Search all columns..."
/>
<AgGridReact
  ref={gridRef}
  quickFilterText={quickFilter}
  // ...
/>
```

### Programmatic External Filter

```tsx
// When you need logic more complex than quickFilterText:
const [showOpenOnly, setShowOpenOnly] = useState(false);

function isExternalFilterPresent() {
  return showOpenOnly;
}

function doesExternalFilterPass(node) {
  return node.data.status === 'open';
}

// After toggling showOpenOnly, tell the grid to re-evaluate:
gridRef.current?.api.onFilterChanged();

<AgGridReact
  isExternalFilterPresent={isExternalFilterPresent}
  doesExternalFilterPass={doesExternalFilterPass}
  // ...
/>
```

---

## Row Selection

### Single Selection

```tsx
<AgGridReact
  rowSelection="single"
  onSelectionChanged={(event) => {
    const selectedRows = event.api.getSelectedRows();
    console.log(selectedRows[0]); // the selected Trade
  }}
/>
```

### Multi-Selection with Checkboxes

```tsx
// Add a checkbox column to a specific column def:
{
  checkboxSelection: true,
  headerCheckboxSelection: true,   // "select all" in header
  headerCheckboxSelectionFilteredOnly: true,  // select all only selects filtered rows
  width: 50,
  pinned: 'left',
}

<AgGridReact
  rowSelection="multiple"
  suppressRowClickSelection={true}  // only checkboxes trigger selection, not row click
  onSelectionChanged={(event) => {
    const selected = event.api.getSelectedRows();
    console.log(`${selected.length} rows selected`);
  }}
/>
```

### Programmatic Selection

```tsx
// Select by row node
gridRef.current?.api.forEachNode(node => {
  if (node.data.status === 'open') {
    node.setSelected(true);
  }
});

// Clear selection
gridRef.current?.api.deselectAll();

// Get selected rows
const rows = gridRef.current?.api.getSelectedRows();
```

---

## Cell Editing

### Enabling Editing

```tsx
// Entire grid editable
<AgGridReact editable={true} />

// Per column
{ field: 'quantity', editable: true }

// Conditional — callback form
{ field: 'price', editable: (params) => params.data.status === 'open' }
```

### Edit Events and Value Setter

```tsx
{
  field: 'quantity',
  editable: true,
  valueSetter: (params) => {
    const newVal = Number(params.newValue);
    if (isNaN(newVal) || newVal <= 0) return false;  // reject invalid value
    params.data.quantity = newVal;
    return true;  // signal that value changed
  },
}

<AgGridReact
  onCellValueChanged={(event) => {
    console.log('Changed:', event.colDef.field, event.oldValue, '->', event.newValue);
    // Persist to server here
    saveRow(event.data);
  }}
/>
```

### Custom Cell Editor (React component)

```tsx
import { ICellEditorParams } from 'ag-grid-community';
import { forwardRef, useImperativeHandle, useRef, useState } from 'react';

interface StatusEditorProps extends ICellEditorParams<Trade> {}

const StatusEditor = forwardRef((props: StatusEditorProps, ref) => {
  const [value, setValue] = useState(props.value);

  useImperativeHandle(ref, () => ({
    getValue: () => value,
    isCancelBeforeStart: () => false,
    isCancelAfterEnd: () => false,
  }));

  return (
    <select
      value={value}
      onChange={e => setValue(e.target.value)}
      autoFocus
      className="h-full w-full border-none outline-none"
    >
      <option value="open">Open</option>
      <option value="closed">Closed</option>
      <option value="pending">Pending</option>
    </select>
  );
});

// In column def:
{
  field: 'status',
  editable: true,
  cellEditor: StatusEditor,
  cellEditorPopup: false,  // render inline (true = floating popup)
}
```

### Single-Click vs Double-Click Editing

```tsx
<AgGridReact
  singleClickEdit={true}   // default is double-click
  // or per column:
  // { field: 'quantity', singleClickEdit: true }
/>
```

---

## Row Grouping and Aggregation (Enterprise)

### Basic Row Grouping

```tsx
// Declare which columns can be grouped
{
  field: 'ticker',
  rowGroup: true,          // automatically groups by this field on load
  hide: true,              // hide the column itself when grouped
}

<AgGridReact
  groupDisplayType="groupRows"  // 'groupRows' | 'multipleColumns' | 'singleColumn'
  groupDefaultExpanded={1}      // auto-expand first level
  animateRows={true}
/>
```

### Aggregation

```tsx
{
  field: 'quantity',
  aggFunc: 'sum',          // built-in: 'sum' | 'min' | 'max' | 'avg' | 'count' | 'first' | 'last'
}
```

### Custom Aggregation Function

```tsx
import { IAggFunc, IAggFuncParams } from 'ag-grid-community';

const weightedAvgPrice: IAggFunc = (params: IAggFuncParams) => {
  let totalValue = 0;
  let totalQty = 0;
  params.rowNode.allLeafChildren?.forEach(node => {
    totalValue += node.data.price * node.data.quantity;
    totalQty += node.data.quantity;
  });
  return totalQty === 0 ? 0 : totalValue / totalQty;
};

<AgGridReact
  aggFuncs={{ weightedAvgPrice }}
  columnDefs={[
    { field: 'price', aggFunc: 'weightedAvgPrice' },
  ]}
/>
```

### Pivot (Enterprise)

```tsx
const columnDefs: ColDef[] = [
  { field: 'ticker', rowGroup: true, hide: true },
  { field: 'status', pivot: true },     // columns are generated from distinct values
  { field: 'quantity', aggFunc: 'sum' },
];

<AgGridReact
  pivotMode={true}
  columnDefs={columnDefs}
/>
```

---

## Virtual Scrolling Internals

AG Grid renders only the rows and columns currently in the viewport — this is the key to handling 100k+ rows.

### Row Virtualisation

- The grid tracks a **row buffer** (default 10 rows) above and below the visible area.
- As the user scrolls, rows exiting the buffer are recycled/destroyed; rows entering are mounted.
- Row height can be fixed (`rowHeight={40}`) or variable (`getRowHeight`). Fixed height is faster because the grid can compute scroll position without layout.

```tsx
// Fixed height — fastest
<AgGridReact rowHeight={40} />

// Variable height — grid measures each row
<AgGridReact
  getRowHeight={(params) => params.data.hasDetails ? 80 : 40}
/>
```

### Column Virtualisation

- Only columns in (or near) the horizontal viewport are rendered.
- Pinned columns are always rendered (they sit outside the scroll container).
- `suppressColumnVirtualisation={true}` disables this — useful if you need all cells present for PDF/print export, but hurts performance on wide grids.

```tsx
<AgGridReact
  rowBuffer={20}                          // increase buffer for smoother fast scrolling
  suppressColumnVirtualisation={false}    // default: columns are virtualised
/>
```

### DOM Recycling

AG Grid reuses DOM row elements when scrolling. It swaps the data bound to existing row components rather than mounting/unmounting. This means:

- Cell renderers are **refreshed** (not remounted) when a row is recycled.
- Implement `refresh(params)` in a class-based renderer, or use `agReactUi` mode which re-renders React components.
- With `reactiveCustomComponents={true}` (v31+), React cell renderers re-render normally via props — no manual refresh needed.

---

## Performance Tips

### getRowId — Stable Row Identity

Without `getRowId`, AG Grid uses array index for identity. When row data changes, the entire grid re-renders. With `getRowId`, only changed rows update.

```tsx
<AgGridReact
  getRowId={(params) => params.data.id}  // unique, stable string
  rowData={trades}                        // grid diffs using getRowId
/>
```

### immutableData / Immutable Row Updates

When `getRowId` is set, passing new `rowData` triggers a diff. Only rows whose reference changed are updated — cells flash, animations run correctly.

```tsx
// Efficient: only the changed row triggers a re-render
setTrades(prev => prev.map(t =>
  t.id === updatedTrade.id ? { ...t, price: updatedTrade.price } : t
));
```

### rowBuffer

Default is 10 rows above and below the viewport. Increase for high-speed scroll, decrease for lower memory usage.

```tsx
<AgGridReact rowBuffer={20} />
```

### suppressColumnVirtualisation Trade-off

| | With Column Virtualisation (default) | Without (`suppressColumnVirtualisation={true}`) |
|---|---|---|
| Render time | Fast (only visible columns) | Slow on wide grids |
| Memory | Low | Higher |
| Export / print | May miss off-screen cells | All cells present |

### animateRows

```tsx
<AgGridReact animateRows={true} />  // smooth row reorder/sort animation; slight perf cost
```

### Large Dataset Tips Summary

```tsx
<AgGridReact
  getRowId={(p) => p.data.id}        // stable identity
  rowBuffer={15}                      // slightly larger buffer
  animateRows={false}                 // disable for very large grids
  suppressColumnVirtualisation={false}
  rowModelType="serverSide"           // don't load 1M rows client-side
  cacheBlockSize={100}
/>
```

---

## Theme Customisation

### Built-in Themes

AG Grid ships with several themes. Import the CSS at your app entry point.

```tsx
// Quartz (modern default, v31+)
import 'ag-grid-community/styles/ag-grid.css';
import 'ag-grid-community/styles/ag-theme-quartz.css';

// Alpine (clean, compact)
import 'ag-grid-community/styles/ag-theme-alpine.css';

// Balham (dense, spreadsheet-like)
import 'ag-grid-community/styles/ag-theme-balham.css';

// Apply theme via className on the wrapper div
<div className="ag-theme-quartz" style={{ height: 400 }}>
  <AgGridReact ... />
</div>
```

Dark variants: `ag-theme-quartz-dark`, `ag-theme-alpine-dark`, `ag-theme-balham-dark`.

### CSS Variable Overrides

AG Grid v31+ exposes the full design system as CSS custom properties.

```css
.ag-theme-quartz {
  --ag-background-color: #1a1a2e;
  --ag-header-background-color: #16213e;
  --ag-odd-row-background-color: #0f3460;
  --ag-row-hover-color: #e94560;
  --ag-border-color: #2a2a4a;
  --ag-font-size: 13px;
  --ag-grid-size: 6px;           /* base spacing unit, scales the whole grid */
  --ag-icon-size: 14px;
  --ag-row-height: 40px;
  --ag-header-height: 44px;
  --ag-cell-horizontal-padding: 12px;
  --ag-selected-row-background-color: rgba(233, 69, 96, 0.2);
}
```

### Programmatic Theme (v32+ Theming API)

```tsx
import { themeQuartz } from 'ag-grid-community';

const myTheme = themeQuartz.withParams({
  backgroundColor: '#1a1a2e',
  headerBackgroundColor: '#16213e',
  accentColor: '#e94560',
});

<AgGridReact theme={myTheme} ... />
```

### Per-Row and Per-Cell Styling

```tsx
// Row-level class
<AgGridReact
  getRowClass={(params) => {
    if (params.data.status === 'pending') return 'row-pending';
    if (params.data.pnl < 0) return 'row-loss';
  }}
/>

// Cell-level style
{
  field: 'pnl',
  cellStyle: (params) => ({
    color: params.value >= 0 ? '#22c55e' : '#ef4444',
    fontWeight: 'bold',
  }),
}
```

---

## AG Grid vs TanStack Table

This is a common interview comparison. The fundamental difference is the design philosophy.

| Dimension | AG Grid | TanStack Table v8 |
|---|---|---|
| Architecture | Batteries-included: full component with DOM | Headless: logic only, you own the DOM |
| UI control | Limited to theme/CSS customisation | Total — render whatever HTML you want |
| Bundle size | Larger (~200–500 KB min+gzip, varies by modules) | Small (~15 KB) |
| Out-of-box features | Massive: SSRM, pivoting, Excel export, charts | Core: sort, filter, pagination, grouping logic |
| Enterprise features | Row grouping, pivot, Excel (paid license) | All features free, community-driven |
| Virtualisation | Built-in, highly optimised | Bring your own (e.g. TanStack Virtual) |
| Cell editing | Built-in | Implement yourself |
| Learning curve | High (large API surface) | Moderate (concepts, but you control output) |
| When to pick | Internal tools, finance/analytics, complex requirements out of the box | Custom design systems, full control, smaller apps, open source |

### TanStack Table code for comparison

```tsx
// TanStack Table — you write the table HTML:
const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() });

return (
  <table>
    <thead>
      {table.getHeaderGroups().map(hg => (
        <tr key={hg.id}>
          {hg.headers.map(h => (
            <th key={h.id}>{flexRender(h.column.columnDef.header, h.getContext())}</th>
          ))}
        </tr>
      ))}
    </thead>
    <tbody>
      {table.getRowModel().rows.map(row => (
        <tr key={row.id}>
          {row.getVisibleCells().map(cell => (
            <td key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</td>
          ))}
        </tr>
      ))}
    </tbody>
  </table>
);
```

### AG Grid equivalent — no HTML table needed:

```tsx
// AG Grid — the grid renders everything:
<div className="ag-theme-quartz" style={{ height: 400 }}>
  <AgGridReact rowData={data} columnDefs={columns} />
</div>
```

**Interview answer**: Choose AG Grid when requirements include enterprise features (server-side row model, Excel export, pivot), the team can't build and maintain complex table UI, or you're building internal tooling fast. Choose TanStack Table when you have a design system that the table must conform to, the feature set is simpler, or bundle size is constrained.

---

## Interview Questions & Key Points

**Q: How does AG Grid handle 1 million rows?**
Use the Server-Side Row Model. The grid never holds all rows client-side — it requests blocks of rows from the server based on scroll position, sort, and filter state. Combine with `cacheBlockSize` to tune block size vs round-trip count.

**Q: What is getRowId used for?**
It gives AG Grid a stable identity for each row. Without it, AG Grid uses array index — when data changes, all rows re-render. With `getRowId`, only rows whose data changed are updated, and the grid can animate row changes correctly.

**Q: How do you pass data from a cell renderer back to the grid?**
Via the Grid API: call `params.api.applyTransaction()` or use an `onCellValueChanged` handler. Cell renderers should not mutate `params.data` directly — use `valueSetter` or dispatch to your state management layer.

**Q: What is the difference between valueFormatter and cellRenderer?**
`valueFormatter` transforms the display string — no React component, no DOM beyond the text node. It also applies in exports (CSV/Excel). `cellRenderer` replaces the entire cell content with a React component — full control but no automatic export support unless you implement `cellRendererParams.exportValueFormatter`.

**Q: How do you keep AG Grid in sync with React state?**
Pass `rowData` as a prop derived from your state. With `getRowId` set, AG Grid diffs the new array and only updates changed rows. For real-time updates, use `api.applyTransaction({ add, update, remove })` for surgical updates without diffing the full array.

```tsx
// Surgical update — more efficient than replacing rowData for single row updates
gridRef.current?.api.applyTransaction({
  update: [{ ...updatedTrade }],
});
```
