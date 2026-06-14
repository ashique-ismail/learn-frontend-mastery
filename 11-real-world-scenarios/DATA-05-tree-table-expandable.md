# Tree Table (Expandable Rows)

## The Idea

**In plain English:** A tree table is a data grid where each row can have child rows nested beneath it. Clicking an expand arrow reveals the children inline, indented under the parent. Children can themselves have children, making the structure arbitrarily deep. The visual hierarchy communicates parent-child relationships without navigating away from the page.

**Real-world analogy:** Think of a corporate org chart rendered as a spreadsheet.

- The **CEO row** is at level 0 — always visible, no indentation.
- Clicking the **expand arrow** on the CEO row reveals the VP rows beneath it, indented one level.
- Each **VP row** can be expanded to reveal their direct reports, indented another level.
- A **dotted-line relationship** = a row with lazy-loaded children that don't exist in the initial payload — you only fetch them when the user actually expands that row.
- **Collapsing a VP** hides all their reports, regardless of whether those reports were also expanded.
- The **row numbers in the spreadsheet** don't change — collapsed rows just vanish from the visual flow, but the data structure is unchanged.

The key insight: the visual indentation, the expand/collapse toggles, and the animated row transitions are all presentational concerns. The recursive data structure, which nodes are expanded, and lazy-fetch coordination are all state concerns. These two layers must stay cleanly separated.

---

## Learning Objectives

- Understand the recursive component pattern for rendering arbitrarily deep node trees
- Implement controlled vs uncontrolled expand state and know when each is appropriate
- Lazy-load children on first expand without re-fetching on subsequent expands
- Apply the ARIA `treegrid` pattern correctly: `role="treegrid"`, `role="row"`, `aria-level`, `aria-expanded`, `aria-posinset`, `aria-setsize`
- Drive indentation purely via CSS `padding-inline-start` without nested tables or extra wrapper elements
- Animate row expand/collapse with CSS without causing layout shift on the surrounding rows

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Indent rows by depth level | ✅ with CSS custom property `--depth` | — |
| Animate a single row expanding | ✅ with `grid-template-rows` trick | — |
| Track which nodes are expanded | ❌ | CSS has no Set or Map state |
| Recursively render child rows | ❌ | CSS cannot loop over or recurse into data |
| Lazy-fetch children on first expand | ❌ | CSS cannot call APIs |
| Collapse all descendants when parent collapses | ❌ | Requires walking a tree data structure |
| Update `aria-expanded` on toggle | ❌ | ARIA state toggling requires JS |
| Show a loading spinner while children fetch | ❌ | Requires async state |
| Synchronize controlled expand state from parent | ❌ | CSS has no prop-passing mechanism |

**Conclusion:** CSS owns indentation depth, the expand/collapse animation, and the loading spinner visual. JS/framework owns the expansion state map, the recursive render, lazy-fetch coordination, and all ARIA attribute management.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  role="treegrid" wraps the entire component.
  Each row is role="row" with aria-level, aria-expanded, aria-posinset, aria-setsize.
  The expand toggle cell is role="gridcell" and contains the actual button.
  Column headers use role="columnheader".
-->
<table role="treegrid" aria-label="File system" class="tree-table">
  <thead>
    <tr role="row">
      <th role="columnheader" scope="col" class="tree-table__name-col">Name</th>
      <th role="columnheader" scope="col">Size</th>
      <th role="columnheader" scope="col">Modified</th>
    </tr>
  </thead>
  <tbody>
    <!-- Level 0 — has children, collapsed -->
    <tr
      role="row"
      aria-level="1"
      aria-expanded="false"
      aria-posinset="1"
      aria-setsize="3"
      class="tree-table__row"
      style="--depth: 0"
    >
      <td role="gridcell" class="tree-table__name-cell">
        <button
          class="tree-table__toggle"
          aria-label="Expand src"
          tabindex="0"
        >
          <span class="tree-table__chevron" aria-hidden="true">▶</span>
        </button>
        <span class="tree-table__icon" aria-hidden="true">📁</span>
        src
      </td>
      <td role="gridcell">—</td>
      <td role="gridcell">2024-01-15</td>
    </tr>

    <!-- Level 1 — child row, hidden while parent collapsed -->
    <tr
      role="row"
      aria-level="2"
      aria-posinset="1"
      aria-setsize="2"
      class="tree-table__row tree-table__row--expanding"
      style="--depth: 1"
      hidden
    >
      <td role="gridcell" class="tree-table__name-cell">
        <!-- No toggle button: leaf node -->
        <span class="tree-table__leaf-spacer" aria-hidden="true"></span>
        <span class="tree-table__icon" aria-hidden="true">📄</span>
        index.ts
      </td>
      <td role="gridcell">4 KB</td>
      <td role="gridcell">2024-01-14</td>
    </tr>
  </tbody>
</table>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --tree-indent:        20px;   /* indent per depth level */
  --tree-row-height:    40px;
  --tree-transition:    0.2s cubic-bezier(0.4, 0, 0.2, 1);
  --tree-hover-bg:      rgba(0, 0, 0, 0.04);
  --tree-focus-ring:    2px solid #0066cc;
  --tree-toggle-size:   24px;
  --tree-chevron-color: #666;
}

/* ─── Table reset ─── */
.tree-table {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
}

.tree-table th,
.tree-table td {
  padding: 0 12px;
  height: var(--tree-row-height);
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  border-bottom: 1px solid #eee;
}

/* ─── Name cell: indent via CSS custom property ─── */
/*
  --depth is set inline per row (style="--depth: 2").
  We add (toggle-size + gap) as a base offset so leaf nodes
  align their text with branch nodes at the same level.
*/
.tree-table__name-cell {
  display: flex;
  align-items: center;
  gap: 6px;
  padding-inline-start: calc(
    var(--depth, 0) * var(--tree-indent) + 8px
  );
}

/* ─── Expand/collapse toggle ─── */
.tree-table__toggle {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width:  var(--tree-toggle-size);
  height: var(--tree-toggle-size);
  background: none;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  color: var(--tree-chevron-color);
  padding: 0;
}

.tree-table__toggle:focus-visible {
  outline: var(--tree-focus-ring);
  outline-offset: 1px;
}

/* Leaf node spacer: same width as the toggle so text aligns */
.tree-table__leaf-spacer {
  flex-shrink: 0;
  width: var(--tree-toggle-size);
}

/* ─── Chevron rotation ─── */
.tree-table__chevron {
  display: inline-block;
  font-size: 10px;
  transition: transform var(--tree-transition);
}

/* The row carries aria-expanded; style the chevron off that */
[aria-expanded="true"] .tree-table__chevron {
  transform: rotate(90deg);
}

/* ─── Row hover ─── */
.tree-table__row:hover td {
  background: var(--tree-hover-bg);
}

/* ─── Row expand/collapse animation ─── */
/*
  The "grid-template-rows: 0fr → 1fr" trick animates height
  without knowing the target height. Each animated row wraps
  its content in a div with overflow: hidden.
  Using a wrapper <div> inside <td> is necessary because
  <tr> itself cannot be animated with height reliably.
*/
.tree-table__row--expanding td > .tree-table__cell-inner {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows var(--tree-transition);
  overflow: hidden;
}

.tree-table__row--expanding.is-expanded td > .tree-table__cell-inner {
  grid-template-rows: 1fr;
}

.tree-table__row--expanding td > .tree-table__cell-inner > * {
  overflow: hidden;
  min-height: 0;
}

/* ─── Loading state ─── */
.tree-table__loading-row td {
  padding-inline-start: calc(var(--depth, 0) * var(--tree-indent) + 8px + var(--tree-toggle-size) + 6px);
  color: #999;
  font-style: italic;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .tree-table__chevron,
  .tree-table__row--expanding td > .tree-table__cell-inner {
    transition: none;
  }
}
```

**What CSS owns:** indentation depth via `--depth` custom property, chevron rotation keyed off `aria-expanded`, row hover highlight, expand/collapse animation via `grid-template-rows`, focus ring, reduced-motion fallback.

**What CSS cannot own:** which rows are expanded, recursive rendering of child rows, lazy-fetch timing, ARIA attribute management, collapsing all descendants when a parent collapses.

---

## React Implementation

### Data Shape

```tsx
// tree-table.types.ts
export interface TreeNode {
  id: string;
  name: string;
  size?: string;
  modified?: string;
  hasChildren: boolean;   // true even before children are fetched
  children?: TreeNode[];  // undefined = not yet loaded; [] = loaded, no children
}

// Simulated API
export async function fetchChildren(nodeId: string): Promise<TreeNode[]> {
  await new Promise(r => setTimeout(r, 400)); // simulate latency
  const map: Record<string, TreeNode[]> = {
    'src': [
      { id: 'index', name: 'index.ts',   size: '4 KB',  modified: '2024-01-14', hasChildren: false },
      { id: 'utils', name: 'utils',      modified: '2024-01-10', hasChildren: true },
    ],
    'utils': [
      { id: 'format', name: 'format.ts', size: '2 KB',  modified: '2024-01-10', hasChildren: false },
    ],
  };
  return map[nodeId] ?? [];
}
```

### The Hook — Expansion and Lazy-Load State

```tsx
// useTreeTable.ts
import { useState, useCallback } from 'react';
import type { TreeNode } from './tree-table.types';
import { fetchChildren } from './tree-table.types';

interface TreeState {
  nodes: Map<string, TreeNode>;       // flat map by id for O(1) updates
  expandedIds: Set<string>;
  loadingIds: Set<string>;
}

export function useTreeTable(initialNodes: TreeNode[]) {
  const [state, setState] = useState<TreeState>(() => ({
    nodes: new Map(flattenNodes(initialNodes)),
    expandedIds: new Set(),
    loadingIds: new Set(),
  }));

  const toggle = useCallback(async (nodeId: string) => {
    setState(prev => {
      const isExpanded = prev.expandedIds.has(nodeId);

      if (isExpanded) {
        // Collapse: remove this id AND all descendant ids from expandedIds
        const descendants = getDescendantIds(nodeId, prev.nodes);
        const nextExpanded = new Set(prev.expandedIds);
        nextExpanded.delete(nodeId);
        descendants.forEach(id => nextExpanded.delete(id));
        return { ...prev, expandedIds: nextExpanded };
      }

      // Expand: check if children are already loaded
      const node = prev.nodes.get(nodeId);
      if (!node || !node.hasChildren) return prev;

      if (node.children !== undefined) {
        // Already loaded — just expand
        const nextExpanded = new Set(prev.expandedIds);
        nextExpanded.add(nodeId);
        return { ...prev, expandedIds: nextExpanded };
      }

      // Need to fetch — mark as loading, don't expand yet
      const nextLoading = new Set(prev.loadingIds);
      nextLoading.add(nodeId);
      return { ...prev, loadingIds: nextLoading };
    });

    // Check if we need to fetch (outside setState to avoid closure issues)
    setState(prev => {
      const node = prev.nodes.get(nodeId);
      if (!node || node.children !== undefined || !prev.loadingIds.has(nodeId)) {
        return prev;
      }
      return prev; // actual fetch happens below
    });

    // Perform the fetch outside setState
    const currentNode = await new Promise<TreeNode | undefined>(resolve => {
      setState(prev => {
        resolve(prev.nodes.get(nodeId));
        return prev;
      });
    });

    if (!currentNode || currentNode.children !== undefined) return;

    try {
      const children = await fetchChildren(nodeId);

      setState(prev => {
        const nextNodes = new Map(prev.nodes);
        // Add children to the flat map
        children.forEach(child => nextNodes.set(child.id, child));
        // Update parent with loaded children
        nextNodes.set(nodeId, { ...currentNode, children });

        const nextLoading = new Set(prev.loadingIds);
        nextLoading.delete(nodeId);

        const nextExpanded = new Set(prev.expandedIds);
        nextExpanded.add(nodeId);

        return { nodes: nextNodes, expandedIds: nextExpanded, loadingIds: nextLoading };
      });
    } catch {
      setState(prev => {
        const nextLoading = new Set(prev.loadingIds);
        nextLoading.delete(nodeId);
        return { ...prev, loadingIds: nextLoading };
      });
    }
  }, []);

  return { state, toggle };
}

// ── Helpers ──

function flattenNodes(nodes: TreeNode[]): [string, TreeNode][] {
  const result: [string, TreeNode][] = [];
  function walk(list: TreeNode[]) {
    for (const node of list) {
      result.push([node.id, node]);
      if (node.children) walk(node.children);
    }
  }
  walk(nodes);
  return result;
}

function getDescendantIds(nodeId: string, nodes: Map<string, TreeNode>): string[] {
  const node = nodes.get(nodeId);
  if (!node?.children) return [];
  const result: string[] = [];
  function walk(children: TreeNode[]) {
    for (const child of children) {
      result.push(child.id);
      const childNode = nodes.get(child.id);
      if (childNode?.children) walk(childNode.children);
    }
  }
  walk(node.children);
  return result;
}
```

### The Recursive Row Component

```tsx
// TreeTable.tsx
import React from 'react';
import { useTreeTable } from './useTreeTable';
import type { TreeNode } from './tree-table.types';

interface TreeTableProps {
  nodes: TreeNode[];
  caption?: string;
}

export function TreeTable({ nodes, caption }: TreeTableProps) {
  const { state, toggle } = useTreeTable(nodes);

  // Build ordered list of visible rows by walking tree depth-first
  const visibleRows = buildVisibleRows(nodes, state.expandedIds, state.nodes, 0);

  return (
    <table
      role="treegrid"
      aria-label={caption ?? 'Tree table'}
      className="tree-table"
    >
      <thead>
        <tr role="row">
          <th role="columnheader" scope="col" className="tree-table__name-col">Name</th>
          <th role="columnheader" scope="col">Size</th>
          <th role="columnheader" scope="col">Modified</th>
        </tr>
      </thead>
      <tbody>
        {visibleRows.map(({ node, depth, posInSet, setSize }) => (
          <TreeRow
            key={node.id}
            node={node}
            depth={depth}
            posInSet={posInSet}
            setSize={setSize}
            isExpanded={state.expandedIds.has(node.id)}
            isLoading={state.loadingIds.has(node.id)}
            onToggle={toggle}
          />
        ))}
      </tbody>
    </table>
  );
}

// ── Single row ──

interface TreeRowProps {
  node: TreeNode;
  depth: number;
  posInSet: number;
  setSize: number;
  isExpanded: boolean;
  isLoading: boolean;
  onToggle: (id: string) => void;
}

function TreeRow({
  node, depth, posInSet, setSize, isExpanded, isLoading, onToggle,
}: TreeRowProps) {
  const isBranch = node.hasChildren;

  return (
    <tr
      role="row"
      aria-level={depth + 1}          // aria-level is 1-based
      aria-expanded={isBranch ? isExpanded : undefined}
      aria-posinset={posInSet}
      aria-setsize={setSize}
      className="tree-table__row"
      style={{ '--depth': depth } as React.CSSProperties}
    >
      <td role="gridcell" className="tree-table__name-cell">
        {isBranch ? (
          <button
            className="tree-table__toggle"
            aria-label={`${isExpanded ? 'Collapse' : 'Expand'} ${node.name}`}
            onClick={() => onToggle(node.id)}
            disabled={isLoading}
          >
            {isLoading
              ? <span className="tree-table__spinner" aria-hidden="true" />
              : <span className="tree-table__chevron" aria-hidden="true">▶</span>
            }
          </button>
        ) : (
          <span className="tree-table__leaf-spacer" aria-hidden="true" />
        )}
        <span aria-hidden="true">{isBranch ? '📁' : '📄'}</span>
        {node.name}
      </td>
      <td role="gridcell">{node.size ?? '—'}</td>
      <td role="gridcell">{node.modified ?? '—'}</td>
    </tr>
  );
}

// ── Walk tree to build a flat ordered list of visible rows ──

interface VisibleRow {
  node: TreeNode;
  depth: number;
  posInSet: number;
  setSize: number;
}

function buildVisibleRows(
  nodes: TreeNode[],
  expandedIds: Set<string>,
  nodeMap: Map<string, TreeNode>,
  depth: number,
): VisibleRow[] {
  const result: VisibleRow[] = [];
  nodes.forEach((rawNode, index) => {
    const node = nodeMap.get(rawNode.id) ?? rawNode;
    result.push({
      node,
      depth,
      posInSet: index + 1,
      setSize: nodes.length,
    });
    if (expandedIds.has(node.id) && node.children?.length) {
      result.push(...buildVisibleRows(node.children, expandedIds, nodeMap, depth + 1));
    }
  });
  return result;
}
```

### Controlled Variant (externally managed expand state)

```tsx
// Controlled: caller owns expandedIds and provides onToggle
interface ControlledTreeTableProps {
  nodes: TreeNode[];
  expandedIds: Set<string>;
  onToggle: (nodeId: string) => void;
  loadingIds?: Set<string>;
}

export function ControlledTreeTable({
  nodes, expandedIds, onToggle, loadingIds = new Set(),
}: ControlledTreeTableProps) {
  const nodeMap = new Map(flattenNodesExported(nodes));
  const visibleRows = buildVisibleRows(nodes, expandedIds, nodeMap, 0);

  return (
    <table role="treegrid" className="tree-table">
      <thead>
        <tr role="row">
          <th role="columnheader" scope="col">Name</th>
          <th role="columnheader" scope="col">Size</th>
          <th role="columnheader" scope="col">Modified</th>
        </tr>
      </thead>
      <tbody>
        {visibleRows.map(({ node, depth, posInSet, setSize }) => (
          <TreeRow
            key={node.id}
            node={node}
            depth={depth}
            posInSet={posInSet}
            setSize={setSize}
            isExpanded={expandedIds.has(node.id)}
            isLoading={loadingIds.has(node.id)}
            onToggle={onToggle}
          />
        ))}
      </tbody>
    </table>
  );
}

function flattenNodesExported(nodes: TreeNode[]): [string, TreeNode][] {
  const result: [string, TreeNode][] = [];
  function walk(list: TreeNode[]) {
    for (const node of list) {
      result.push([node.id, node]);
      if (node.children) walk(node.children);
    }
  }
  walk(nodes);
  return result;
}
```

---

## Angular Implementation

### Data Shape & Service

```typescript
// tree-table.model.ts
export interface TreeNode {
  id: string;
  name: string;
  size?: string;
  modified?: string;
  hasChildren: boolean;
  children?: TreeNode[];
}
```

```typescript
// tree-table.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import type { TreeNode } from './tree-table.model';

@Injectable({ providedIn: 'root' })
export class TreeTableService {
  private http = inject(HttpClient);

  private _nodes      = signal<TreeNode[]>([]);
  private _expanded   = signal<Set<string>>(new Set());
  private _loading    = signal<Set<string>>(new Set());
  private _nodeMap    = signal<Map<string, TreeNode>>(new Map());

  readonly expandedIds = this._expanded.asReadonly();
  readonly loadingIds  = this._loading.asReadonly();

  readonly visibleRows = computed(() => {
    return this.buildVisible(this._nodes(), this._expanded(), this._nodeMap(), 0);
  });

  init(nodes: TreeNode[]) {
    const map = new Map<string, TreeNode>();
    this.flatten(nodes, map);
    this._nodes.set(nodes);
    this._nodeMap.set(map);
  }

  async toggle(nodeId: string): Promise<void> {
    const expanded = this._expanded();

    if (expanded.has(nodeId)) {
      // Collapse — also remove all descendants
      const descendants = this.getDescendantIds(nodeId);
      const next = new Set(expanded);
      next.delete(nodeId);
      descendants.forEach(id => next.delete(id));
      this._expanded.set(next);
      return;
    }

    const node = this._nodeMap().get(nodeId);
    if (!node?.hasChildren) return;

    if (node.children !== undefined) {
      // Already loaded
      this._expanded.set(new Set([...expanded, nodeId]));
      return;
    }

    // Lazy load
    this._loading.set(new Set([...this._loading(), nodeId]));

    try {
      const children = await firstValueFrom(
        this.http.get<TreeNode[]>(`/api/nodes/${nodeId}/children`)
      );

      this._nodeMap.update(map => {
        const next = new Map(map);
        children.forEach(c => next.set(c.id, c));
        next.set(nodeId, { ...node, children });
        return next;
      });

      this._nodes.update(root => this.mergeChildren(root, nodeId, children));
      this._expanded.set(new Set([...this._expanded(), nodeId]));
    } finally {
      this._loading.update(set => {
        const next = new Set(set);
        next.delete(nodeId);
        return next;
      });
    }
  }

  private mergeChildren(nodes: TreeNode[], targetId: string, children: TreeNode[]): TreeNode[] {
    return nodes.map(node => {
      if (node.id === targetId) return { ...node, children };
      if (node.children) return { ...node, children: this.mergeChildren(node.children, targetId, children) };
      return node;
    });
  }

  private flatten(nodes: TreeNode[], map: Map<string, TreeNode>) {
    for (const node of nodes) {
      map.set(node.id, node);
      if (node.children) this.flatten(node.children, map);
    }
  }

  private getDescendantIds(nodeId: string): string[] {
    const node = this._nodeMap().get(nodeId);
    if (!node?.children) return [];
    const result: string[] = [];
    const walk = (children: TreeNode[]) => {
      for (const child of children) {
        result.push(child.id);
        const c = this._nodeMap().get(child.id);
        if (c?.children) walk(c.children);
      }
    };
    walk(node.children);
    return result;
  }

  private buildVisible(
    nodes: TreeNode[],
    expanded: Set<string>,
    nodeMap: Map<string, TreeNode>,
    depth: number,
  ): Array<{ node: TreeNode; depth: number; posInSet: number; setSize: number }> {
    const result: ReturnType<typeof this.buildVisible> = [];
    nodes.forEach((rawNode, i) => {
      const node = nodeMap.get(rawNode.id) ?? rawNode;
      result.push({ node, depth, posInSet: i + 1, setSize: nodes.length });
      if (expanded.has(node.id) && node.children?.length) {
        result.push(...this.buildVisible(node.children, expanded, nodeMap, depth + 1));
      }
    });
    return result;
  }
}
```

### Tree Table Component

```typescript
// tree-table.component.ts
import { Component, Input, OnInit, inject } from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { TreeTableService } from './tree-table.service';
import type { TreeNode } from './tree-table.model';

@Component({
  selector: 'app-tree-table',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <table
      role="treegrid"
      [attr.aria-label]="caption || 'Tree table'"
      class="tree-table"
    >
      <thead>
        <tr role="row">
          <th role="columnheader" scope="col">Name</th>
          <th role="columnheader" scope="col">Size</th>
          <th role="columnheader" scope="col">Modified</th>
        </tr>
      </thead>
      <tbody>
        <tr
          *ngFor="let row of svc.visibleRows(); trackBy: trackById"
          role="row"
          [attr.aria-level]="row.depth + 1"
          [attr.aria-expanded]="row.node.hasChildren ? svc.expandedIds().has(row.node.id) : null"
          [attr.aria-posinset]="row.posInSet"
          [attr.aria-setsize]="row.setSize"
          class="tree-table__row"
          [style.--depth]="row.depth"
        >
          <td role="gridcell" class="tree-table__name-cell">
            <ng-container *ngIf="row.node.hasChildren; else leafSpacer">
              <button
                class="tree-table__toggle"
                [attr.aria-label]="(svc.expandedIds().has(row.node.id) ? 'Collapse ' : 'Expand ') + row.node.name"
                [disabled]="svc.loadingIds().has(row.node.id)"
                (click)="svc.toggle(row.node.id)"
              >
                <span
                  *ngIf="!svc.loadingIds().has(row.node.id)"
                  class="tree-table__chevron"
                  aria-hidden="true"
                >▶</span>
                <span
                  *ngIf="svc.loadingIds().has(row.node.id)"
                  class="tree-table__spinner"
                  aria-hidden="true"
                ></span>
              </button>
            </ng-container>
            <ng-template #leafSpacer>
              <span class="tree-table__leaf-spacer" aria-hidden="true"></span>
            </ng-template>

            <span aria-hidden="true">{{ row.node.hasChildren ? '📁' : '📄' }}</span>
            {{ row.node.name }}
          </td>
          <td role="gridcell">{{ row.node.size ?? '—' }}</td>
          <td role="gridcell">{{ row.node.modified ?? '—' }}</td>
        </tr>
      </tbody>
    </table>
  `,
})
export class TreeTableComponent implements OnInit {
  @Input() nodes: TreeNode[] = [];
  @Input() caption?: string;

  svc = inject(TreeTableService);

  ngOnInit() {
    this.svc.init(this.nodes);
  }

  trackById(_: number, row: { node: TreeNode }) {
    return row.node.id;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Table has `role="treegrid"` | Applied to `<table>` element |
| Each data row has `role="row"` | Applied to every `<tr>` |
| Each cell has `role="gridcell"` | Applied to every `<td>`; headers use `role="columnheader"` |
| `aria-level` communicates depth | Set to `depth + 1` (1-based) on each `<tr>` |
| `aria-expanded` on branch rows only | `undefined` / omitted on leaf rows; `true`/`false` on branches |
| `aria-posinset` and `aria-setsize` | Computed during `buildVisibleRows` walk |
| Toggle button has descriptive label | `aria-label="Expand src"` / `aria-label="Collapse src"` |
| Loading state communicated | Button `disabled` + spinner replaces chevron; `aria-label` unchanged |
| Keyboard: Space/Enter toggles row | Inherits from `<button>` natively |
| Keyboard: arrow key navigation | `treegrid` pattern recommends Up/Down/Left/Right — implement via `keydown` handler on `<tbody>` |
| Focus remains on toggle after expand | Browser default preserves focus on the clicked button |
| Screen reader reads row in order | DOM order matches visual order; `aria-level` provides depth context |
| Reduced motion | `transition: none` via `prefers-reduced-motion` media query |

---

## Production Pitfalls

**1. Animating `<tr>` height directly doesn't work**
`<tr>` elements ignore `height` transitions in most browsers because table layout algorithm recalculates row heights synchronously. Fix: wrap cell content in a `<div>` and animate `grid-template-rows: 0fr → 1fr` on that wrapper. This animates the inner content height without touching the `<tr>` itself.

**2. Collapse must also collapse all descendants**
If a user expands A → B → C and then collapses A, B and C must be removed from `expandedIds`. Failing to do this causes ghost expansion: collapsing A, expanding A again, and watching B's children appear without B being explicitly expanded. Fix: walk the descendant tree on collapse and remove all descendant IDs from the expanded set.

**3. Lazy-fetched children can arrive out of order**
If the user rapidly expands two different nodes, both fetches run in parallel. The second fetch may resolve first. Fix: key fetched children to their parent node ID and merge them into the node map atomically in a single `setState` / `signal.update` call. Never merge by position.

**4. Flat DOM vs nested DOM — choose flat**
Nesting `<tbody>` inside `<tbody>` is invalid HTML. A recursive component that renders `<table>` inside a `<td>` will break screen reader table traversal. Fix: flatten the tree into a single ordered array of rows with depth metadata, and render all rows in one `<tbody>`. `aria-level` communicates hierarchy to assistive technology without nesting tables.

**5. `aria-level` must be 1-based, not 0-based**
ARIA requires `aria-level` to start at 1. Using `depth` directly (0-indexed) means root rows have `aria-level="0"`, which screen readers ignore or misinterpret. Fix: always pass `depth + 1`.

**6. Controlled expand state breaks with internal lazy-load logic**
If the consumer controls `expandedIds` but the component also fires the API fetch, there's a race: the consumer updates `expandedIds` before children arrive, causing the expand to happen with no children rendered. Fix: use a clear separation — either fully controlled (consumer manages fetch + state), or fully uncontrolled (component manages everything). Provide a prop like `mode="controlled" | "uncontrolled"` and document the contract explicitly.

**7. Large trees with thousands of rows tank rendering performance**
Rendering 5,000 `<tr>` elements at once stalls the main thread. Fix: add virtual scrolling using a windowing library (`@tanstack/react-virtual` or Angular CDK `ScrollingModule`). Only render rows that are within the viewport. The `buildVisibleRows` flat array maps perfectly to a virtualizer's item count.

---

## Interview Angle

**Q: "Walk me through how you would implement a tree table with lazy-loaded children."**

A strong answer covers four layers:

1. **Data model** — represent the tree as a flat `Map<id, node>` rather than a pure nested structure. This gives O(1) node lookup for updates and avoids deep cloning on every state change. The recursive tree structure is only used for building the ordered visible-rows array on each render.

2. **Expansion state** — use a `Set<string>` of expanded IDs rather than a boolean field on each node. This keeps the data immutable and makes collapse-all-descendants a clean set-difference operation. Distinguish between "not loaded" (`children === undefined`), "loaded but empty" (`children = []`), and "loaded with children" (`children.length > 0`).

3. **Lazy loading** — on first expand, if `children === undefined`, fire the fetch and track the ID in a `loadingIds` set. On fetch resolution, merge children into the node map and add the ID to `expandedIds` atomically. On subsequent expands, children are already present — no fetch needed. The `children !== undefined` check is the cache gate.

4. **ARIA** — use `role="treegrid"` on the table, `aria-level` (1-based), `aria-expanded` on branch rows only, `aria-posinset` / `aria-setsize` for position context, and a descriptive `aria-label` on each toggle button that announces the node name and expand/collapse intent.

**Follow-up: "Why not nest `<tbody>` elements recursively to reflect the hierarchy?"**

Nested `<tbody>` is invalid HTML — the spec allows only one `<tbody>` per `<table>`. Even where browsers tolerate it, screen readers count rows and columns across the entire table; nested tables inside cells break column alignment and produce incorrect cell counts for AT users. The flat-rows-with-depth-metadata approach renders all rows in a single `<tbody>` and uses `aria-level` to communicate hierarchy, which is exactly what the ARIA `treegrid` pattern specifies. This also makes virtual scrolling straightforward since the visible-rows array maps directly to a virtualizer's item list.
