# Server-Side Sortable / Filterable Table

## The Idea

**In plain English:** You have a table of records that lives on the server — too many rows to load all at once. The user can click a column header to sort, type in a search box to filter, and change pages. Every action triggers a new server request. The challenge is keeping the URL in sync (so the page is bookmarkable and shareable), showing a loading state without jarring layout shifts, and making sort feel instant even before the server responds.

**Real-world analogy:** Think of a hotel's front desk reservation system.

- The **server is the back-office database** — it holds all 10,000 reservations. The clerk (browser) only sees a printed page of 20 at a time.
- **Sorting by check-in date** is telling the back office "re-print the list, sorted by date". The clerk slides the current list under a "fetching…" overlay while the new one prints.
- **Filtering by guest name** is telling the back office "only print guests whose name contains 'Smith'".
- The **URL** is the request slip — if you photocopy it, you get exactly the same list. This is why the sort/filter/page state lives in the URL, not in component state.
- **Optimistic sort** is the clerk flipping the current list upside down immediately (before the printed copy arrives), so the user sees something change right away.

---

## Learning Objectives

- Sync sort, filter, and page state to URL search parameters
- Debounce the search input before triggering a fetch
- Show a loading overlay that does not cause layout shift
- Implement optimistic column-header sort (reorder visible rows instantly)
- Handle error and empty states
- Replicate the pattern in Angular using signals and `ActivatedRoute`

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show/hide loading overlay | ✅ with class toggle | — |
| Sort indicator on column header | ✅ with `aria-sort` + CSS `::after` | — |
| Fetch new data on sort/filter change | ❌ | Requires JS `fetch` |
| Sync state to URL query params | ❌ | Requires `history.pushState` / router |
| Debounce search input | ❌ | Requires JS timer |
| Optimistic reorder of rows | ❌ | Requires JS array manipulation |

---

## HTML & CSS Foundation

```html
<div class="table-container">
  <!-- Filter bar -->
  <div class="table-toolbar">
    <input
      type="search"
      class="table-search"
      placeholder="Search…"
      aria-label="Filter table"
    />
    <span class="table-count" aria-live="polite">Showing 1–20 of 480 rows</span>
  </div>

  <!-- Loading overlay — absolute-positioned over the table -->
  <div class="table-wrapper" aria-busy="false">
    <div class="table-overlay" aria-hidden="true" hidden></div>
    <table class="data-table" aria-label="Users">
      <thead>
        <tr>
          <th scope="col">
            <button class="sort-btn" aria-sort="none">Name</button>
          </th>
          <th scope="col">
            <button class="sort-btn" aria-sort="ascending">Email</button>
          </th>
          <th scope="col">
            <button class="sort-btn" aria-sort="none">Created</button>
          </th>
        </tr>
      </thead>
      <tbody>
        <!-- Rows injected by JS -->
      </tbody>
    </table>
  </div>

  <!-- Pagination -->
  <nav class="pagination" aria-label="Table pagination">
    <button class="page-btn" aria-label="Previous page">‹</button>
    <span class="page-info">Page 1 of 24</span>
    <button class="page-btn" aria-label="Next page">›</button>
  </nav>
</div>
```

```css
.table-container { display: flex; flex-direction: column; gap: 0.75rem; }

.table-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.table-search {
  padding: 0.5rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.9rem;
  width: 260px;
  outline: none;
}
.table-search:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59,130,246,0.25);
}

.table-count { color: #6b7280; font-size: 0.875rem; }

/* Wrapper holds the overlay + table */
.table-wrapper {
  position: relative;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
}

/* Semi-transparent overlay while fetching */
.table-overlay {
  position: absolute;
  inset: 0;
  background: rgba(255, 255, 255, 0.65);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2;
}
.table-overlay::after {
  content: '';
  width: 24px;
  height: 24px;
  border: 3px solid #e5e7eb;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9rem;
}

.data-table th, .data-table td {
  padding: 0.625rem 0.875rem;
  text-align: left;
  border-bottom: 1px solid #f3f4f6;
}

.data-table th { background: #f9fafb; font-weight: 600; }

/* Sort button inside <th> */
.sort-btn {
  background: none;
  border: none;
  font: inherit;
  font-weight: 600;
  cursor: pointer;
  padding: 0;
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
}

/* Arrow indicator via CSS using aria-sort */
.sort-btn::after {
  content: '↕';
  font-size: 0.75rem;
  color: #9ca3af;
}
[aria-sort="ascending"]  .sort-btn::after { content: '↑'; color: #3b82f6; }
[aria-sort="descending"] .sort-btn::after { content: '↓'; color: #3b82f6; }

.pagination {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.page-btn {
  padding: 0.375rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
}
.page-btn:disabled { opacity: 0.4; cursor: not-allowed; }
.page-info { font-size: 0.875rem; color: #6b7280; }
```

---

## React Implementation

### Types and API Layer

```tsx
// types.ts
export type SortDir = 'asc' | 'desc' | null;

export interface SortState {
  column: string | null;
  direction: SortDir;
}

export interface TableParams {
  search: string;
  sort: string | null;
  dir: SortDir;
  page: number;
}

export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

export interface PagedResponse<T> {
  rows: T[];
  total: number;
  page: number;
  pageSize: number;
}
```

### URL-Synced Table Hook

```tsx
// useTableParams.ts
import { useCallback } from 'react';
import { useSearchParams } from 'react-router-dom';
import type { TableParams, SortDir } from './types';

export function useTableParams(): [TableParams, (partial: Partial<TableParams>) => void] {
  const [params, setParams] = useSearchParams();

  const tableParams: TableParams = {
    search: params.get('search') ?? '',
    sort:   params.get('sort')   ?? null,
    dir:    (params.get('dir')   ?? null) as SortDir,
    page:   Number(params.get('page') ?? '1'),
  };

  const update = useCallback((partial: Partial<TableParams>) => {
    setParams(prev => {
      const next = new URLSearchParams(prev);

      if (partial.search !== undefined) {
        partial.search ? next.set('search', partial.search) : next.delete('search');
        next.set('page', '1'); // reset to page 1 on search change
      }
      if (partial.sort !== undefined) {
        partial.sort ? next.set('sort', partial.sort) : next.delete('sort');
      }
      if (partial.dir !== undefined) {
        partial.dir ? next.set('dir', partial.dir) : next.delete('dir');
      }
      if (partial.page !== undefined) {
        next.set('page', String(partial.page));
      }

      return next;
    }, { replace: true }); // replace, not push — avoids polluting browser history
  }, [setParams]);

  return [tableParams, update];
}
```

### Table Component

```tsx
// UserTable.tsx
import { useState, useEffect, useRef, useCallback, useMemo } from 'react';
import { useTableParams } from './useTableParams';
import type { User, PagedResponse, SortDir } from './types';

const PAGE_SIZE = 20;

async function fetchUsers(
  params: URLSearchParams,
  signal: AbortSignal
): Promise<PagedResponse<User>> {
  const res = await fetch(`/api/users?${params}`, { signal });
  if (!res.ok) throw new Error(`Fetch failed: ${res.status}`);
  return res.json();
}

export function UserTable() {
  const [tableParams, updateParams] = useTableParams();
  const [data, setData]       = useState<PagedResponse<User> | null>(null);
  const [isLoading, setLoading] = useState(false);
  const [error, setError]     = useState<string | null>(null);
  // Optimistic rows: reorder current rows immediately on sort click
  const [optimisticRows, setOptimisticRows] = useState<User[] | null>(null);

  const abortRef = useRef<AbortController | null>(null);
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const load = useCallback((params: URLSearchParams) => {
    if (abortRef.current) abortRef.current.abort();
    const controller = new AbortController();
    abortRef.current = controller;

    setLoading(true);
    setError(null);

    fetchUsers(params, controller.signal)
      .then(result => {
        if (abortRef.current === controller) {
          setData(result);
          setOptimisticRows(null); // server result replaces optimistic sort
          setLoading(false);
        }
      })
      .catch(err => {
        if (err.name === 'AbortError') return;
        if (abortRef.current === controller) {
          setError(err.message);
          setLoading(false);
        }
      });
  }, []);

  // Trigger fetch whenever URL params change
  useEffect(() => {
    const params = new URLSearchParams({
      ...(tableParams.search && { search: tableParams.search }),
      ...(tableParams.sort   && { sort: tableParams.sort }),
      ...(tableParams.dir    && { dir:  tableParams.dir  }),
      page: String(tableParams.page),
      pageSize: String(PAGE_SIZE),
    });

    load(params);
    return () => { abortRef.current?.abort(); };
  }, [tableParams.search, tableParams.sort, tableParams.dir, tableParams.page, load]);

  // Cleanup on unmount
  useEffect(() => () => {
    abortRef.current?.abort();
    if (debounceRef.current) clearTimeout(debounceRef.current);
  }, []);

  const handleSearchChange = (value: string) => {
    if (debounceRef.current) clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      updateParams({ search: value });
    }, 300);
  };

  const handleSort = (column: string) => {
    // Toggle direction or start ascending
    const newDir: SortDir =
      tableParams.sort === column && tableParams.dir === 'asc' ? 'desc' : 'asc';

    // Optimistic sort: reorder current rows immediately
    if (data) {
      const sorted = [...data.rows].sort((a, b) => {
        const aVal = (a as Record<string, string>)[column] ?? '';
        const bVal = (b as Record<string, string>)[column] ?? '';
        return newDir === 'asc'
          ? aVal.localeCompare(bVal)
          : bVal.localeCompare(aVal);
      });
      setOptimisticRows(sorted);
    }

    updateParams({ sort: column, dir: newDir, page: 1 });
  };

  const displayRows = optimisticRows ?? data?.rows ?? [];
  const totalPages  = data ? Math.ceil(data.total / PAGE_SIZE) : 1;

  const columns: { key: keyof User; label: string }[] = [
    { key: 'name',      label: 'Name'    },
    { key: 'email',     label: 'Email'   },
    { key: 'createdAt', label: 'Created' },
  ];

  return (
    <div className="table-container">
      <div className="table-toolbar">
        <input
          type="search"
          className="table-search"
          placeholder="Search…"
          aria-label="Filter table"
          defaultValue={tableParams.search}
          onChange={e => handleSearchChange(e.target.value)}
        />
        {data && (
          <span className="table-count" aria-live="polite">
            Showing {(tableParams.page - 1) * PAGE_SIZE + 1}–
            {Math.min(tableParams.page * PAGE_SIZE, data.total)} of {data.total} rows
          </span>
        )}
      </div>

      <div className="table-wrapper" aria-busy={isLoading}>
        {isLoading && <div className="table-overlay" aria-hidden="true" />}

        {error ? (
          <p role="alert" style={{ padding: '1rem', color: '#dc2626' }}>{error}</p>
        ) : (
          <table className="data-table" aria-label="Users">
            <thead>
              <tr>
                {columns.map(col => (
                  <th
                    key={col.key}
                    scope="col"
                    aria-sort={
                      tableParams.sort === col.key
                        ? tableParams.dir === 'asc' ? 'ascending' : 'descending'
                        : 'none'
                    }
                  >
                    <button
                      className="sort-btn"
                      onClick={() => handleSort(col.key)}
                    >
                      {col.label}
                    </button>
                  </th>
                ))}
              </tr>
            </thead>
            <tbody>
              {displayRows.length === 0 && !isLoading ? (
                <tr>
                  <td colSpan={3} style={{ textAlign: 'center', padding: '2rem', color: '#9ca3af' }}>
                    No results found
                  </td>
                </tr>
              ) : (
                displayRows.map(row => (
                  <tr key={row.id}>
                    <td>{row.name}</td>
                    <td>{row.email}</td>
                    <td>{new Date(row.createdAt).toLocaleDateString()}</td>
                  </tr>
                ))
              )}
            </tbody>
          </table>
        )}
      </div>

      <nav className="pagination" aria-label="Table pagination">
        <button
          className="page-btn"
          aria-label="Previous page"
          disabled={tableParams.page <= 1}
          onClick={() => updateParams({ page: tableParams.page - 1 })}
        >
          ‹
        </button>
        <span className="page-info">
          Page {tableParams.page} of {totalPages}
        </span>
        <button
          className="page-btn"
          aria-label="Next page"
          disabled={tableParams.page >= totalPages}
          onClick={() => updateParams({ page: tableParams.page + 1 })}
        >
          ›
        </button>
      </nav>
    </div>
  );
}
```

---

## Angular Implementation

```typescript
// user-table.component.ts
import {
  Component, OnInit, inject, signal, computed, DestroyRef
} from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { HttpClient } from '@angular/common/http';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import {
  switchMap, debounceTime, distinctUntilChanged, catchError, of, tap, startWith, map
} from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DatePipe, NgFor, NgIf } from '@angular/common';

interface User { id: string; name: string; email: string; createdAt: string; }
interface PagedResponse { rows: User[]; total: number; page: number; pageSize: number; }

@Component({
  selector: 'app-user-table',
  standalone: true,
  imports: [ReactiveFormsModule, NgFor, NgIf, DatePipe],
  template: `
    <div class="table-container">
      <div class="table-toolbar">
        <input type="search" class="table-search" placeholder="Search…"
               aria-label="Filter table" [formControl]="searchControl" />
        <span class="table-count" aria-live="polite">
          Showing {{ rangeLabel() }} of {{ total() }} rows
        </span>
      </div>

      <div class="table-wrapper" [attr.aria-busy]="isLoading()">
        @if (isLoading()) {
          <div class="table-overlay" aria-hidden="true"></div>
        }
        @if (error()) {
          <p role="alert" style="padding:1rem;color:#dc2626">{{ error() }}</p>
        } @else {
          <table class="data-table" aria-label="Users">
            <thead>
              <tr>
                @for (col of columns; track col.key) {
                  <th scope="col"
                      [attr.aria-sort]="ariaSortFor(col.key)">
                    <button class="sort-btn" (click)="sortBy(col.key)">
                      {{ col.label }}
                    </button>
                  </th>
                }
              </tr>
            </thead>
            <tbody>
              @for (row of displayRows(); track row.id) {
                <tr>
                  <td>{{ row.name }}</td>
                  <td>{{ row.email }}</td>
                  <td>{{ row.createdAt | date:'mediumDate' }}</td>
                </tr>
              } @empty {
                <tr>
                  <td colspan="3" style="text-align:center;padding:2rem;color:#9ca3af">
                    No results found
                  </td>
                </tr>
              }
            </tbody>
          </table>
        }
      </div>

      <nav class="pagination" aria-label="Table pagination">
        <button class="page-btn" aria-label="Previous page"
                [disabled]="page() <= 1" (click)="goToPage(page() - 1)">‹</button>
        <span class="page-info">Page {{ page() }} of {{ totalPages() }}</span>
        <button class="page-btn" aria-label="Next page"
                [disabled]="page() >= totalPages()" (click)="goToPage(page() + 1)">›</button>
      </nav>
    </div>
  `,
})
export class UserTableComponent implements OnInit {
  private route    = inject(ActivatedRoute);
  private router   = inject(Router);
  private http     = inject(HttpClient);
  private destroyRef = inject(DestroyRef);

  searchControl = new FormControl('', { nonNullable: true });
  columns = [
    { key: 'name',      label: 'Name'    },
    { key: 'email',     label: 'Email'   },
    { key: 'createdAt', label: 'Created' },
  ];

  rows         = signal<User[]>([]);
  optimistic   = signal<User[] | null>(null);
  displayRows  = computed(() => this.optimistic() ?? this.rows());
  total        = signal(0);
  page         = signal(1);
  sortCol      = signal<string | null>(null);
  sortDir      = signal<'asc' | 'desc' | null>(null);
  isLoading    = signal(false);
  error        = signal<string | null>(null);

  readonly PAGE_SIZE = 20;

  totalPages  = computed(() => Math.ceil(this.total() / this.PAGE_SIZE));
  rangeLabel  = computed(() => {
    const start = (this.page() - 1) * this.PAGE_SIZE + 1;
    const end   = Math.min(this.page() * this.PAGE_SIZE, this.total());
    return `${start}–${end}`;
  });

  ariaSortFor(col: string): string {
    if (this.sortCol() !== col) return 'none';
    return this.sortDir() === 'asc' ? 'ascending' : 'descending';
  }

  ngOnInit() {
    // Read initial values from URL
    this.route.queryParamMap.pipe(
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(params => {
      this.searchControl.setValue(params.get('search') ?? '', { emitEvent: false });
      this.sortCol.set(params.get('sort'));
      this.sortDir.set((params.get('dir') as 'asc' | 'desc') ?? null);
      this.page.set(Number(params.get('page') ?? 1));
    });

    // React to search input changes
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(search => this.updateUrl({ search, page: 1 }));

    // React to URL changes and fetch data
    this.route.queryParamMap.pipe(
      debounceTime(0),   // microtask — lets all param changes settle
      switchMap(params => {
        this.isLoading.set(true);
        this.error.set(null);
        const p = new URLSearchParams({
          ...(params.get('search') && { search: params.get('search')! }),
          ...(params.get('sort')   && { sort:   params.get('sort')!   }),
          ...(params.get('dir')    && { dir:    params.get('dir')!    }),
          page:     params.get('page')     ?? '1',
          pageSize: String(this.PAGE_SIZE),
        });
        return this.http.get<PagedResponse>(`/api/users?${p}`).pipe(
          catchError(err => { this.error.set(err.message); return of(null); })
        );
      }),
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(result => {
      this.isLoading.set(false);
      if (result) {
        this.rows.set(result.rows);
        this.total.set(result.total);
        this.optimistic.set(null);
      }
    });
  }

  sortBy(col: string) {
    const newDir: 'asc' | 'desc' =
      this.sortCol() === col && this.sortDir() === 'asc' ? 'desc' : 'asc';

    // Optimistic sort
    const sorted = [...this.rows()].sort((a, b) => {
      const aVal = (a as Record<string, string>)[col] ?? '';
      const bVal = (b as Record<string, string>)[col] ?? '';
      return newDir === 'asc' ? aVal.localeCompare(bVal) : bVal.localeCompare(aVal);
    });
    this.optimistic.set(sorted);

    this.updateUrl({ sort: col, dir: newDir, page: 1 });
  }

  goToPage(page: number) {
    this.updateUrl({ page });
  }

  private updateUrl(partial: Record<string, string | number>) {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParamsHandling: 'merge',
      queryParams: partial,
      replaceUrl: true,
    });
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Sort direction announced to screen readers | `aria-sort="ascending"` / `"descending"` / `"none"` on `<th>` |
| Table has accessible name | `aria-label="Users"` on `<table>` |
| Loading state communicated | `aria-busy="true"` on the table wrapper |
| Count updates announced | `aria-live="polite"` on the count span |
| Error announced immediately | `role="alert"` on error paragraph |
| Empty state is communicated | Visible cell spanning all columns |
| Pagination buttons labeled | `aria-label="Previous page"` / `"Next page"` |
| Disabled buttons convey state | `disabled` attribute prevents click and sets `aria-disabled` implicitly |

---

## Production Pitfalls

**1. Pushing a history entry on every sort/filter change**
This creates a huge back-button stack. Always use `replace: true` / `replaceUrl: true` so that only intentional navigation (new routes) creates history entries.

**2. Optimistic sort diverges from server sort for edge cases**
Server sort may use collation, null handling, or locale-specific rules the client cannot replicate. The optimistic sort is only a visual hint — the server result must always overwrite it.

**3. Debounce and URL sync fighting each other**
If you both debounce the search input AND react to URL changes, you may trigger two fetches. The Angular implementation uses `debounceTime(0)` on the `queryParamMap` stream to coalesce multiple simultaneous URL param changes (sort + page reset) into a single fetch.

**4. No `aria-sort="none"` on unsorted columns**
Screen readers need `aria-sort="none"` to indicate a column is sortable but not currently sorted. Omitting it leaves the column's sortability undiscoverable.

**5. Race condition on rapid page changes**
Clicking Next five times rapidly fires five requests. `switchMap` (Angular) and `AbortController` (React) cancel previous requests, ensuring only the last response updates the UI.

---

## Interview Angle

**Q: "How do you implement a server-side sortable table so the URL is bookmarkable?"**

Strong answer: All table state (search, sort column, sort direction, page) lives in URL query parameters — not in component state. On any user action, update the URL with `replace: true` (avoiding history pollution). React to URL changes with `useSearchParams` (React Router) or `ActivatedRoute.queryParamMap` (Angular), and fire a new fetch. This means the back button, sharing a URL, and browser refresh all work correctly.

**Follow-up: "How do you make sorting feel instant?"**

Optimistic sort: on column-header click, immediately reorder the currently visible rows client-side using `Array.sort`, then fire the server request. When the server responds, replace the optimistically sorted rows with the authoritative result. The user sees instant feedback even on a slow connection.
