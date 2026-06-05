# TanStack Table v8

## The Idea

**In plain English:** TanStack Table is a tool that handles all the smart behavior behind a data table — like sorting rows, filtering results, and flipping through pages — without deciding what it looks like. You give it your data and rules, and it tells you exactly what to display; you draw the table yourself.

**Real-world analogy:** Think of a school librarian who manages a giant card catalog. You ask her to sort books by author, filter to only sci-fi, and show you page 3 of the results. She hands you a neat, ordered list — but she never builds the bookshelf or prints the shelf labels. That is your job.

- The librarian = TanStack Table (handles all sorting, filtering, and pagination logic)
- The card catalog rules = column definitions (describe what each column is and how to sort/filter it)
- The bookshelf and labels you build = the HTML table elements (`<table>`, `<th>`, `<td>`) you render yourself

---

## What It Is

TanStack Table (formerly React Table) is a **headless** table library — it provides all the logic for sorting, filtering, pagination, row selection, and more, but zero UI. You render the table HTML yourself.

---

## Core Concept: Headless

```
TanStack Table provides:         You provide:
- Column definition logic    →   <th>, <td> HTML elements
- Sorting state/handlers     →   Sort button UI
- Filter state/handlers      →   Filter input UI
- Pagination state/handlers  →   Page buttons UI
- Row selection logic        →   Checkboxes UI
```

---

## Basic Setup

```tsx
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  ColumnDef,
  flexRender,
} from '@tanstack/react-table';

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
}

const columns: ColumnDef<User>[] = [
  {
    id: 'select',
    header: ({ table }) => (
      <input
        type="checkbox"
        checked={table.getIsAllRowsSelected()}
        onChange={table.getToggleAllRowsSelectedHandler()}
      />
    ),
    cell: ({ row }) => (
      <input
        type="checkbox"
        checked={row.getIsSelected()}
        onChange={row.getToggleSelectedHandler()}
      />
    ),
  },
  {
    accessorKey: 'name',
    header: 'Name',
    cell: ({ row }) => <strong>{row.original.name}</strong>,
  },
  {
    accessorKey: 'email',
    header: 'Email',
  },
  {
    accessorKey: 'role',
    header: 'Role',
    cell: ({ getValue }) => <Badge>{getValue<string>()}</Badge>,
    filterFn: 'equals',
  },
  {
    accessorFn: row => row.createdAt,
    id: 'createdAt',
    header: 'Created',
    cell: ({ getValue }) => getValue<Date>().toLocaleDateString(),
    sortingFn: 'datetime',
  },
];

function UserTable({ data }: { data: User[] }) {
  const [sorting, setSorting] = useState<SortingState>([]);
  const [filtering, setFiltering] = useState('');
  const [rowSelection, setRowSelection] = useState<RowSelectionState>({});

  const table = useReactTable({
    data,
    columns,
    state: { sorting, globalFilter: filtering, rowSelection },
    onSortingChange: setSorting,
    onGlobalFilterChange: setFiltering,
    onRowSelectionChange: setRowSelection,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    enableRowSelection: true,
    initialState: { pagination: { pageSize: 20 } },
  });

  return (
    <div>
      <input
        value={filtering}
        onChange={e => setFiltering(e.target.value)}
        placeholder="Search..."
      />

      <table>
        <thead>
          {table.getHeaderGroups().map(headerGroup => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map(header => (
                <th
                  key={header.id}
                  onClick={header.column.getToggleSortingHandler()}
                  style={{ cursor: header.column.getCanSort() ? 'pointer' : 'default' }}
                >
                  {flexRender(header.column.columnDef.header, header.getContext())}
                  {{ asc: ' 🔼', desc: ' 🔽' }[header.column.getIsSorted() as string] ?? ''}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map(row => (
            <tr key={row.id} data-selected={row.getIsSelected()}>
              {row.getVisibleCells().map(cell => (
                <td key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      <div>
        <button onClick={() => table.previousPage()} disabled={!table.getCanPreviousPage()}>
          Previous
        </button>
        <span>Page {table.getState().pagination.pageIndex + 1} of {table.getPageCount()}</span>
        <button onClick={() => table.nextPage()} disabled={!table.getCanNextPage()}>
          Next
        </button>
      </div>

      <p>{table.getFilteredSelectedRowModel().rows.length} of {table.getFilteredRowModel().rows.length} rows selected</p>
    </div>
  );
}
```

---

## Server-Side Data

For large datasets, use manual mode — table manages state, you fetch data based on that state:

```tsx
function ServerSideTable() {
  const [pagination, setPagination] = useState({ pageIndex: 0, pageSize: 20 });
  const [sorting, setSorting] = useState<SortingState>([]);

  const { data, isLoading } = useQuery({
    queryKey: ['users', pagination, sorting],
    queryFn: () => fetchUsers({
      page: pagination.pageIndex,
      pageSize: pagination.pageSize,
      sortBy: sorting[0]?.id,
      sortDir: sorting[0]?.desc ? 'desc' : 'asc',
    }),
  });

  const table = useReactTable({
    data: data?.rows ?? [],
    columns,
    rowCount: data?.total, // total count for pagination
    state: { pagination, sorting },
    onPaginationChange: setPagination,
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    manualPagination: true, // tell table we're handling pagination
    manualSorting: true,    // tell table we're handling sorting
  });
}
```

---

## Column Pinning

```tsx
const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),
  initialState: {
    columnPinning: {
      left: ['select', 'name'],    // pin these to the left
      right: ['actions'],          // pin these to the right
    },
  },
});

// In cell rendering
<td
  style={{
    position: 'sticky',
    left: cell.column.getIsPinned() === 'left' ? `${cell.column.getStart('left')}px` : undefined,
    zIndex: cell.column.getIsPinned() ? 1 : 0,
  }}
>
```

---

## Integration with TanStack Virtual

For large datasets with virtualization:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualTable({ data }: { data: User[] }) {
  const table = useReactTable({ data, columns, getCoreRowModel: getCoreRowModel() });
  const { rows } = table.getRowModel();

  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => {
          const row = rows[virtualRow.index];
          return (
            <div
              key={row.id}
              style={{ position: 'absolute', top: `${virtualRow.start}px`, height: '48px' }}
            >
              {row.getVisibleCells().map(cell => (
                <span key={cell.id}>
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </span>
              ))}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## Common Interview Questions

**Q: Why headless instead of a pre-built table component?**
Complete control over markup, styling, and accessibility. No fighting the library's opinionated HTML structure. Works with any CSS framework (Tailwind, CSS Modules, Material UI).

**Q: How does TanStack Table handle millions of rows?**
Use manual server-side mode (pagination, sorting done on the server) or combine with TanStack Virtual for client-side virtualization. The table itself doesn't limit row count — the bottleneck is DOM rendering.

**Q: What's `accessorKey` vs `accessorFn`?**
`accessorKey: 'name'` — accesses `row.name` (string path). `accessorFn: row => row.nested.deeply.value` — arbitrary function for computed or nested values. Use `accessorFn` when the value isn't a direct key.
