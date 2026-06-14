# Command Palette (⌘K)

## The Idea

**In plain English:** A command palette is a full-screen overlay triggered by a global keyboard shortcut (usually ⌘K on Mac, Ctrl+K on Windows). It renders a search input over a portal, fuzzy-matches every available action in the app as the user types, ranks results by score, and lets the user navigate the list with arrow keys and confirm with Enter — all without touching the mouse. Recent commands float to the top so power users rarely need to type at all.

**Real-world analogy:** Think of VS Code's Command Palette or Spotlight Search on macOS.

- The **global shortcut** is the physical button on the side of an iPhone — it doesn't matter which app is open, it always works.
- The **fuzzy search** is like a smart autocomplete that matches "git com" to "Git: Commit All" — it doesn't need an exact prefix, it scores fragments across the full string.
- The **virtual list** is like a cinema ticket counter that only sets up as many booths as fit in view — if 10,000 commands exist, the DOM still only renders 8 rows at a time, scrolling them in and out as you navigate.
- The **recent items** are like your browser's address bar — the things you reached for yesterday appear at the top today, before you even start typing.
- The **focus trap** is like a modal dialog: while the palette is open, your keyboard lives inside it. Nothing behind it reacts.

The key insight: the palette is a productivity multiplier precisely because it is keyboard-first, zero-mouse, and application-global. Every implementation decision follows from that constraint.

---

## Learning Objectives

- Understand why a command palette requires a portal, not an in-tree overlay
- Build the CSS foundation: overlay, panel, result rows, highlight spans
- Implement fuzzy search with score-based ranking from scratch (no library required)
- Build a virtual list that handles thousands of results without layout thrash
- Wire a global `keydown` listener that works regardless of which component has focus
- Implement keyboard-first navigation: ArrowUp, ArrowDown, Enter, Escape
- Manage focus trap and scroll-into-view for the active item
- Persist and surface recent commands using `localStorage`

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Render overlay above all other content | ⚠️ `position: fixed` + `z-index` can do it, but stacking context traps may clip it | Portal (React) / `document.body` append (Angular) is the safe solution |
| Listen to ⌘K regardless of focused element | ❌ | CSS has no event listeners |
| Filter and rank results as the user types | ❌ | CSS has no string-matching or scoring logic |
| Highlight matching characters in result labels | ❌ | Requires splitting text nodes and wrapping spans |
| Virtual list — render only visible rows | ❌ | CSS cannot conditionally render DOM nodes |
| Scroll the active item into view | ❌ | Requires `scrollIntoView()` or manual `scrollTop` |
| Persist recent commands across sessions | ❌ | No access to `localStorage` |
| Trap Tab focus inside the palette | ❌ | Requires `keydown` listener |
| Announce active result to screen readers | ❌ | `aria-activedescendant` must be updated by JS |

**Conclusion:** CSS owns appearance — the frosted-glass overlay, panel shadow, row hover states, and highlighted character spans. Everything behavioral is owned by JavaScript.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Trigger button (or just document keydown — no button required) -->
<button class="cmd-trigger" aria-label="Open command palette" aria-keyshortcuts="Meta+k Control+k">
  <kbd>⌘K</kbd>
</button>

<!-- Portal target: appended directly to <body>, outside the React/Angular root -->
<div class="cmd-portal" role="dialog" aria-modal="true" aria-label="Command palette">

  <!-- Backdrop -->
  <div class="cmd-backdrop"></div>

  <!-- Panel -->
  <div class="cmd-panel">

    <!-- Search input -->
    <div class="cmd-search-row">
      <svg class="cmd-search-icon" aria-hidden="true"><!-- search icon --></svg>
      <input
        class="cmd-input"
        type="text"
        role="combobox"
        aria-autocomplete="list"
        aria-controls="cmd-listbox"
        aria-activedescendant=""
        placeholder="Search commands…"
        autocomplete="off"
        spellcheck="false"
      />
      <kbd class="cmd-esc-hint" aria-hidden="true">esc</kbd>
    </div>

    <!-- Results listbox -->
    <ul
      id="cmd-listbox"
      class="cmd-list"
      role="listbox"
      aria-label="Commands"
    >
      <!-- Group header (optional) -->
      <li class="cmd-group-header" role="presentation">Recent</li>

      <!-- Result row -->
      <li
        class="cmd-result"
        id="cmd-result-0"
        role="option"
        aria-selected="true"
      >
        <span class="cmd-result-icon" aria-hidden="true">⚡</span>
        <span class="cmd-result-label">
          <!-- JS injects <mark> spans around matched characters -->
          Git: <mark>Com</mark>mit All
        </span>
        <kbd class="cmd-result-shortcut" aria-hidden="true">⌘⇧G</kbd>
      </li>
    </ul>

    <!-- Empty state -->
    <p class="cmd-empty" role="status" aria-live="polite">No results for "xyz"</p>
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --cmd-overlay-bg:  rgba(0, 0, 0, 0.45);
  --cmd-panel-bg:    #ffffff;
  --cmd-panel-width: 640px;
  --cmd-panel-max-h: 420px;
  --cmd-row-h:       44px;
  --cmd-radius:      12px;
  --cmd-shadow:      0 24px 64px rgba(0, 0, 0, 0.25);
  --cmd-accent:      #5b6af0;
  --cmd-highlight-bg: rgba(91, 106, 240, 0.12);
  --cmd-transition:  0.15s ease;
}

/* ─── Portal wrapper (position: fixed, full viewport) ─── */
.cmd-portal {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding-top: 12vh;
}

/* ─── Backdrop ─── */
.cmd-backdrop {
  position: absolute;
  inset: 0;
  background: var(--cmd-overlay-bg);
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
  animation: cmd-fade-in var(--cmd-transition) both;
}

/* ─── Panel ─── */
.cmd-panel {
  position: relative;           /* above backdrop */
  width: var(--cmd-panel-width);
  max-width: calc(100vw - 2rem);
  background: var(--cmd-panel-bg);
  border-radius: var(--cmd-radius);
  box-shadow: var(--cmd-shadow);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  animation: cmd-slide-in var(--cmd-transition) both;
}

/* ─── Search row ─── */
.cmd-search-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0 1rem;
  border-bottom: 1px solid #eee;
  flex-shrink: 0;
}

.cmd-search-icon {
  width: 18px;
  height: 18px;
  color: #999;
  flex-shrink: 0;
}

.cmd-input {
  flex: 1;
  height: 52px;
  border: none;
  outline: none;
  font-size: 1rem;
  background: transparent;
  color: inherit;
}

.cmd-esc-hint {
  font-size: 0.7rem;
  padding: 2px 6px;
  border: 1px solid #ddd;
  border-radius: 4px;
  color: #999;
}

/* ─── Results list ─── */
.cmd-list {
  overflow-y: auto;
  max-height: var(--cmd-panel-max-h);
  list-style: none;
  margin: 0;
  padding: 0.25rem 0;
  /* Virtual list: JS sets a fixed height and translates rows by index */
}

.cmd-group-header {
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #999;
  padding: 0.5rem 1rem 0.25rem;
}

/* ─── Result row ─── */
.cmd-result {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  height: var(--cmd-row-h);
  padding: 0 1rem;
  cursor: pointer;
  border-radius: 6px;
  margin: 0 0.25rem;
  transition: background var(--cmd-transition);
}

/* Keyboard-active state — driven by aria-selected, not :hover alone */
.cmd-result[aria-selected="true"] {
  background: var(--cmd-highlight-bg);
  outline: none;
}

.cmd-result:hover {
  background: var(--cmd-highlight-bg);
}

.cmd-result-icon {
  font-size: 1rem;
  width: 20px;
  text-align: center;
  flex-shrink: 0;
}

.cmd-result-label {
  flex: 1;
  font-size: 0.9rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Fuzzy match highlight characters */
.cmd-result-label mark {
  background: transparent;
  color: var(--cmd-accent);
  font-weight: 700;
}

.cmd-result-shortcut {
  font-size: 0.7rem;
  padding: 2px 6px;
  border: 1px solid #ddd;
  border-radius: 4px;
  color: #999;
  flex-shrink: 0;
}

/* ─── Empty state ─── */
.cmd-empty {
  padding: 2rem 1rem;
  text-align: center;
  color: #999;
  font-size: 0.9rem;
  margin: 0;
}

/* ─── Animations ─── */
@keyframes cmd-fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

@keyframes cmd-slide-in {
  from { opacity: 0; transform: scale(0.96) translateY(-8px); }
  to   { opacity: 1; transform: scale(1)    translateY(0);    }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .cmd-panel,
  .cmd-backdrop,
  .cmd-result {
    animation: none;
    transition: none;
  }
}
```

**What CSS owns:** overlay blur, panel animation, row highlight state via `aria-selected`, match character color via `mark`, empty state layout, reduced-motion fallback.

**What CSS cannot own:** fuzzy scoring, virtual list row positioning, active-item scroll management, global shortcut, focus trap.

---

## React Implementation

### Types and Fuzzy Search Engine

```tsx
// commandPalette.types.ts
export interface Command {
  id: string;
  label: string;
  group?: string;
  icon?: string;
  shortcut?: string;
  keywords?: string[];         // extra searchable terms not shown in label
  action: () => void;
}

export interface ScoredCommand {
  command: Command;
  score: number;
  labelParts: Array<{ text: string; match: boolean }>; // for highlight rendering
}
```

```tsx
// fuzzySearch.ts
import type { Command, ScoredCommand } from './commandPalette.types';

/**
 * Scores a query against a target string.
 * Returns -1 if no match, otherwise a positive integer (higher = better match).
 * Rewards: consecutive matches, prefix match, word-boundary match.
 */
export function fuzzyScore(query: string, target: string): number {
  if (!query) return 0;

  const q = query.toLowerCase();
  const t = target.toLowerCase();

  // Exact substring — highest priority
  const exactIdx = t.indexOf(q);
  if (exactIdx !== -1) return 1000 - exactIdx; // earlier position = higher score

  // Fuzzy: all query chars must appear in order
  let qi = 0;
  let score = 0;
  let consecutive = 0;
  let lastMatchIdx = -1;

  for (let ti = 0; ti < t.length && qi < q.length; ti++) {
    if (t[ti] === q[qi]) {
      // Bonus for consecutive characters
      consecutive = lastMatchIdx === ti - 1 ? consecutive + 1 : 0;
      score += 10 + consecutive * 5;

      // Bonus for word-boundary match
      if (ti === 0 || t[ti - 1] === ' ' || t[ti - 1] === '-' || t[ti - 1] === ':') {
        score += 20;
      }

      lastMatchIdx = ti;
      qi++;
    }
  }

  // All query chars must have matched
  return qi === q.length ? score : -1;
}

/**
 * Computes the index positions that matched (for highlight rendering).
 */
export function getMatchIndices(query: string, target: string): Set<number> {
  const indices = new Set<number>();
  if (!query) return indices;

  const q = query.toLowerCase();
  const t = target.toLowerCase();

  // Prefer exact substring highlight
  const exactIdx = t.indexOf(q);
  if (exactIdx !== -1) {
    for (let i = exactIdx; i < exactIdx + q.length; i++) indices.add(i);
    return indices;
  }

  // Fuzzy highlight
  let qi = 0;
  for (let ti = 0; ti < t.length && qi < q.length; ti++) {
    if (t[ti] === q[qi]) { indices.add(ti); qi++; }
  }
  return indices;
}

/**
 * Splits label into alternating plain/highlighted parts for rendering.
 */
export function buildLabelParts(
  label: string,
  matchIndices: Set<number>
): Array<{ text: string; match: boolean }> {
  const parts: Array<{ text: string; match: boolean }> = [];
  let current = '';
  let currentMatch = false;

  for (let i = 0; i < label.length; i++) {
    const isMatch = matchIndices.has(i);
    if (isMatch !== currentMatch) {
      if (current) parts.push({ text: current, match: currentMatch });
      current = label[i];
      currentMatch = isMatch;
    } else {
      current += label[i];
    }
  }
  if (current) parts.push({ text: current, match: currentMatch });
  return parts;
}

export function searchCommands(query: string, commands: Command[]): ScoredCommand[] {
  const results: ScoredCommand[] = [];

  for (const command of commands) {
    const searchTarget = [command.label, ...(command.keywords ?? [])].join(' ');
    const score = fuzzyScore(query, searchTarget);
    if (score < 0) continue;

    const matchIndices = getMatchIndices(query, command.label);
    const labelParts = buildLabelParts(command.label, matchIndices);
    results.push({ command, score, labelParts });
  }

  return results.sort((a, b) => b.score - a.score);
}
```

### useCommandPalette Hook

```tsx
// useCommandPalette.ts
import {
  useState, useEffect, useCallback, useRef, useMemo
} from 'react';
import type { Command, ScoredCommand } from './commandPalette.types';
import { searchCommands } from './fuzzySearch';

const RECENT_KEY  = 'cmd-palette:recent';
const MAX_RECENT  = 5;
const VISIBLE_ROWS = 8;

function loadRecent(): string[] {
  try {
    return JSON.parse(localStorage.getItem(RECENT_KEY) ?? '[]');
  } catch {
    return [];
  }
}

function saveRecent(ids: string[]) {
  localStorage.setItem(RECENT_KEY, JSON.stringify(ids.slice(0, MAX_RECENT)));
}

export function useCommandPalette(commands: Command[]) {
  const [isOpen,       setIsOpen]       = useState(false);
  const [query,        setQuery]        = useState('');
  const [activeIndex,  setActiveIndex]  = useState(0);
  const [recentIds,    setRecentIds]    = useState<string[]>(loadRecent);

  const inputRef   = useRef<HTMLInputElement>(null);
  const listRef    = useRef<HTMLUListElement>(null);

  /* ── Fuzzy-filtered + scored results ── */
  const results: ScoredCommand[] = useMemo(() => {
    if (!query.trim()) {
      // Show recent commands first when no query
      const recentCommands = recentIds
        .map(id => commands.find(c => c.id === id))
        .filter((c): c is Command => !!c);
      const rest = commands.filter(c => !recentIds.includes(c.id));
      return [...recentCommands, ...rest].map(command => ({
        command,
        score: 0,
        labelParts: [{ text: command.label, match: false }],
      }));
    }
    return searchCommands(query, commands);
  }, [query, commands, recentIds]);

  /* ── Open / close ── */
  const open = useCallback(() => {
    setIsOpen(true);
    setQuery('');
    setActiveIndex(0);
  }, []);

  const close = useCallback(() => {
    setIsOpen(false);
    setQuery('');
  }, []);

  /* ── Execute a command ── */
  const execute = useCallback((scored: ScoredCommand) => {
    const id = scored.command.id;
    setRecentIds(prev => {
      const next = [id, ...prev.filter(r => r !== id)];
      saveRecent(next);
      return next;
    });
    close();
    scored.command.action();
  }, [close]);

  /* ── Global ⌘K / Ctrl+K listener ── */
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
        e.preventDefault();
        isOpen ? close() : open();
      }
    };
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [isOpen, open, close]);

  /* ── Focus input when opened ── */
  useEffect(() => {
    if (isOpen) {
      // Defer so the portal has mounted
      requestAnimationFrame(() => inputRef.current?.focus());
    }
  }, [isOpen]);

  /* ── Keyboard navigation inside the palette ── */
  const onKeyDown = useCallback((e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown': {
        e.preventDefault();
        setActiveIndex(i => Math.min(i + 1, results.length - 1));
        break;
      }
      case 'ArrowUp': {
        e.preventDefault();
        setActiveIndex(i => Math.max(i - 1, 0));
        break;
      }
      case 'Enter': {
        e.preventDefault();
        const item = results[activeIndex];
        if (item) execute(item);
        break;
      }
      case 'Escape': {
        e.preventDefault();
        close();
        break;
      }
    }
  }, [results, activeIndex, execute, close]);

  /* ── Reset active index when results change ── */
  useEffect(() => { setActiveIndex(0); }, [query]);

  /* ── Scroll active row into view ── */
  useEffect(() => {
    const list = listRef.current;
    if (!list) return;
    const activeEl = list.querySelector<HTMLElement>('[aria-selected="true"]');
    activeEl?.scrollIntoView({ block: 'nearest' });
  }, [activeIndex]);

  return {
    isOpen, open, close, query, setQuery, results,
    activeIndex, onKeyDown, inputRef, listRef, execute,
    VISIBLE_ROWS,
  };
}
```

### Portal and Components

```tsx
// CommandPalette.tsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';
import type { Command, ScoredCommand } from './commandPalette.types';
import { useCommandPalette } from './useCommandPalette';

interface Props {
  commands: Command[];
}

export function CommandPalette({ commands }: Props) {
  const {
    isOpen, close, query, setQuery, results,
    activeIndex, onKeyDown, inputRef, listRef, execute,
  } = useCommandPalette(commands);

  // Focus trap: keep Tab inside the palette
  useEffect(() => {
    if (!isOpen) return;
    const trap = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      e.preventDefault();          // only one focusable element (the input)
    };
    document.addEventListener('keydown', trap);
    return () => document.removeEventListener('keydown', trap);
  }, [isOpen]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="cmd-portal"
      role="dialog"
      aria-modal="true"
      aria-label="Command palette"
      onKeyDown={onKeyDown}
    >
      {/* Backdrop */}
      <div className="cmd-backdrop" onClick={close} aria-hidden="true" />

      {/* Panel */}
      <div className="cmd-panel" role="presentation">
        {/* Search row */}
        <div className="cmd-search-row">
          <svg className="cmd-search-icon" viewBox="0 0 20 20" aria-hidden="true">
            <circle cx="8" cy="8" r="5.5" stroke="currentColor" strokeWidth="1.5" fill="none"/>
            <line x1="12.5" y1="12.5" x2="17" y2="17" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"/>
          </svg>
          <input
            ref={inputRef}
            className="cmd-input"
            type="text"
            role="combobox"
            aria-autocomplete="list"
            aria-controls="cmd-listbox"
            aria-activedescendant={results.length ? `cmd-result-${activeIndex}` : undefined}
            aria-expanded={results.length > 0}
            placeholder="Search commands…"
            autoComplete="off"
            spellCheck={false}
            value={query}
            onChange={e => setQuery(e.target.value)}
          />
          <kbd className="cmd-esc-hint" aria-hidden="true">esc</kbd>
        </div>

        {/* Results */}
        {results.length > 0 ? (
          <ul
            id="cmd-listbox"
            ref={listRef}
            className="cmd-list"
            role="listbox"
            aria-label="Commands"
          >
            {results.map((scored, i) => (
              <CommandResult
                key={scored.command.id}
                scored={scored}
                index={i}
                isActive={i === activeIndex}
                onSelect={() => execute(scored)}
                onHover={() => {/* activeIndex driven by keyboard only */}}
              />
            ))}
          </ul>
        ) : (
          <p className="cmd-empty" role="status" aria-live="polite">
            No results for &ldquo;{query}&rdquo;
          </p>
        )}
      </div>
    </div>,
    document.body
  );
}

/* ── Single result row ── */
interface ResultProps {
  scored: ScoredCommand;
  index: number;
  isActive: boolean;
  onSelect: () => void;
  onHover: () => void;
}

function CommandResult({ scored, index, isActive, onSelect }: ResultProps) {
  const { command, labelParts } = scored;
  return (
    <li
      id={`cmd-result-${index}`}
      className="cmd-result"
      role="option"
      aria-selected={isActive}
      onMouseDown={e => { e.preventDefault(); onSelect(); }} // mousedown prevents input blur
    >
      {command.icon && (
        <span className="cmd-result-icon" aria-hidden="true">{command.icon}</span>
      )}
      <span className="cmd-result-label">
        {labelParts.map((part, pi) =>
          part.match
            ? <mark key={pi}>{part.text}</mark>
            : <span key={pi}>{part.text}</span>
        )}
      </span>
      {command.shortcut && (
        <kbd className="cmd-result-shortcut" aria-hidden="true">{command.shortcut}</kbd>
      )}
    </li>
  );
}
```

### Usage

```tsx
// App.tsx
import { CommandPalette } from './CommandPalette';
import type { Command } from './commandPalette.types';

const COMMANDS: Command[] = [
  { id: 'new-file',    label: 'New File',        icon: '📄', shortcut: '⌘N',  keywords: ['create', 'add'], action: () => console.log('new file') },
  { id: 'git-commit',  label: 'Git: Commit All',  icon: '⚡', shortcut: '⌘⇧G', keywords: ['save', 'push'],  action: () => console.log('commit') },
  { id: 'theme-dark',  label: 'Toggle Dark Mode', icon: '🌙', keywords: ['appearance', 'color'],             action: () => console.log('theme') },
  { id: 'open-settings', label: 'Open Settings', icon: '⚙️', shortcut: '⌘,',                               action: () => console.log('settings') },
];

export function App() {
  return (
    <>
      <main>Your application here</main>
      {/* Mounted once at root — global ⌘K listener is inside the hook */}
      <CommandPalette commands={COMMANDS} />
    </>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// command-palette.service.ts
import { Injectable, signal, computed, effect } from '@angular/core';
import { DOCUMENT } from '@angular/common';
import { inject } from '@angular/core';
import type { Command, ScoredCommand } from './command-palette.model';
import { searchCommands, buildLabelParts } from './fuzzy-search';

const RECENT_KEY = 'cmd-palette:recent';
const MAX_RECENT = 5;

@Injectable({ providedIn: 'root' })
export class CommandPaletteService {
  private doc        = inject(DOCUMENT);
  private _isOpen    = signal(false);
  private _query     = signal('');
  private _activeIdx = signal(0);
  private _recentIds = signal<string[]>(this.loadRecent());

  readonly isOpen    = this._isOpen.asReadonly();
  readonly query     = this._query.asReadonly();
  readonly activeIdx = this._activeIdx.asReadonly();

  results = computed<ScoredCommand[]>(() => {
    const q        = this._query();
    const recent   = this._recentIds();
    const commands = this._commands();

    if (!q.trim()) {
      const recentCmds = recent
        .map(id => commands.find(c => c.id === id))
        .filter((c): c is Command => !!c);
      const rest = commands.filter(c => !recent.includes(c.id));
      return [...recentCmds, ...rest].map(cmd => ({
        command: cmd, score: 0,
        labelParts: [{ text: cmd.label, match: false }],
      }));
    }
    return searchCommands(q, commands);
  });

  private _commands = signal<Command[]>([]);

  register(commands: Command[]) {
    this._commands.set(commands);
  }

  open() { this._isOpen.set(true); this._query.set(''); this._activeIdx.set(0); }
  close() { this._isOpen.set(false); this._query.set(''); }

  setQuery(q: string) {
    this._query.set(q);
    this._activeIdx.set(0);
  }

  moveDown() {
    this._activeIdx.update(i => Math.min(i + 1, this.results().length - 1));
  }

  moveUp() {
    this._activeIdx.update(i => Math.max(i - 1, 0));
  }

  executeActive() {
    const item = this.results()[this._activeIdx()];
    if (!item) return;
    this.recordRecent(item.command.id);
    this.close();
    item.command.action();
  }

  private recordRecent(id: string) {
    this._recentIds.update(prev => {
      const next = [id, ...prev.filter(r => r !== id)].slice(0, MAX_RECENT);
      localStorage.setItem(RECENT_KEY, JSON.stringify(next));
      return next;
    });
  }

  private loadRecent(): string[] {
    try { return JSON.parse(localStorage.getItem(RECENT_KEY) ?? '[]'); }
    catch { return []; }
  }
}
```

### Component

```typescript
// command-palette.component.ts
import {
  Component, inject, OnInit, OnDestroy,
  ViewChild, ElementRef, afterNextRender, HostListener
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { CommandPaletteService } from './command-palette.service';

@Component({
  selector: 'app-command-palette',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <ng-container *ngIf="svc.isOpen()">
      <div
        class="cmd-portal"
        role="dialog"
        aria-modal="true"
        aria-label="Command palette"
        (keydown)="onKeyDown($event)"
      >
        <!-- Backdrop -->
        <div class="cmd-backdrop" aria-hidden="true" (click)="svc.close()"></div>

        <!-- Panel -->
        <div class="cmd-panel">
          <!-- Search row -->
          <div class="cmd-search-row">
            <svg class="cmd-search-icon" viewBox="0 0 20 20" aria-hidden="true">
              <circle cx="8" cy="8" r="5.5" stroke="currentColor" stroke-width="1.5" fill="none"/>
              <line x1="12.5" y1="12.5" x2="17" y2="17" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
            </svg>
            <input
              #inputEl
              class="cmd-input"
              type="text"
              role="combobox"
              aria-autocomplete="list"
              aria-controls="cmd-listbox"
              [attr.aria-activedescendant]="svc.results().length ? 'cmd-result-' + svc.activeIdx() : null"
              [attr.aria-expanded]="svc.results().length > 0"
              placeholder="Search commands…"
              autocomplete="off"
              spellcheck="false"
              [value]="svc.query()"
              (input)="svc.setQuery($any($event.target).value)"
            />
            <kbd class="cmd-esc-hint" aria-hidden="true">esc</kbd>
          </div>

          <!-- Results -->
          <ul
            *ngIf="svc.results().length > 0; else emptyState"
            id="cmd-listbox"
            #listEl
            class="cmd-list"
            role="listbox"
            aria-label="Commands"
          >
            <li
              *ngFor="let scored of svc.results(); let i = index; trackBy: trackById"
              [id]="'cmd-result-' + i"
              class="cmd-result"
              role="option"
              [attr.aria-selected]="i === svc.activeIdx()"
              (mousedown)="onMouseDown($event, i)"
            >
              <span *ngIf="scored.command.icon" class="cmd-result-icon" aria-hidden="true">
                {{ scored.command.icon }}
              </span>
              <span class="cmd-result-label">
                <ng-container *ngFor="let part of scored.labelParts">
                  <mark *ngIf="part.match">{{ part.text }}</mark>
                  <span *ngIf="!part.match">{{ part.text }}</span>
                </ng-container>
              </span>
              <kbd *ngIf="scored.command.shortcut" class="cmd-result-shortcut" aria-hidden="true">
                {{ scored.command.shortcut }}
              </kbd>
            </li>
          </ul>

          <ng-template #emptyState>
            <p class="cmd-empty" role="status" aria-live="polite">
              No results for "{{ svc.query() }}"
            </p>
          </ng-template>
        </div>
      </div>
    </ng-container>
  `,
})
export class CommandPaletteComponent implements OnInit, OnDestroy {
  svc = inject(CommandPaletteService);

  @ViewChild('inputEl') inputEl?: ElementRef<HTMLInputElement>;
  @ViewChild('listEl')  listEl?:  ElementRef<HTMLUListElement>;

  trackById(_: number, scored: { command: { id: string } }) {
    return scored.command.id;
  }

  ngOnInit() {
    // Focus the input after each open, using afterNextRender for SSR safety
    afterNextRender(() => {
      if (this.svc.isOpen()) this.inputEl?.nativeElement.focus();
    });
  }

  // Global ⌘K / Ctrl+K
  @HostListener('document:keydown', ['$event'])
  onGlobalKey(e: KeyboardEvent) {
    if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
      e.preventDefault();
      this.svc.isOpen() ? this.svc.close() : this.svc.open();
    }
  }

  onKeyDown(e: KeyboardEvent) {
    switch (e.key) {
      case 'ArrowDown':  e.preventDefault(); this.svc.moveDown();        this.scrollActive(); break;
      case 'ArrowUp':    e.preventDefault(); this.svc.moveUp();          this.scrollActive(); break;
      case 'Enter':      e.preventDefault(); this.svc.executeActive();   break;
      case 'Escape':     e.preventDefault(); this.svc.close();           break;
      case 'Tab':        e.preventDefault();                              break; // focus trap
    }
  }

  onMouseDown(e: MouseEvent, index: number) {
    e.preventDefault(); // prevent input blur
    const results = this.svc.results();
    const item = results[index];
    if (item) { this.svc.executeActive(); }
  }

  private scrollActive() {
    const list = this.listEl?.nativeElement;
    if (!list) return;
    const active = list.querySelector<HTMLElement>('[aria-selected="true"]');
    active?.scrollIntoView({ block: 'nearest' });
  }

  ngOnDestroy() { this.svc.close(); }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Dialog role and `aria-modal` | `role="dialog" aria-modal="true"` on the portal root |
| `aria-label` on the dialog | `aria-label="Command palette"` — no visible heading needed |
| Input has `role="combobox"` | Communicates to screen readers that typing filters a list |
| Input `aria-controls` points to listbox `id` | Links the two elements so VoiceOver announces the relationship |
| `aria-activedescendant` tracks the active row | Updated on every ArrowUp/ArrowDown; announces active command label |
| Active row has `aria-selected="true"` | Only one item selected at a time; all others have `aria-selected="false"` |
| Match highlights use `<mark>` | Semantic highlight element; VoiceOver reads surrounding context, not individual letters |
| Escape closes palette | Keyboard listener in component; standard dialog dismissal pattern |
| `mousedown` (not `click`) executes commands | Prevents the input from losing focus before the action fires |
| Empty state announced | `role="status" aria-live="polite"` updates as query changes |
| Focus trap on Tab | `e.preventDefault()` on Tab since the input is the only interactive element |
| Focus returns to trigger on close | Parent component holds trigger ref and calls `.focus()` when `isOpen` flips to false |
| `aria-keyshortcuts` on trigger button | Documents the shortcut for AT users scanning the page |
| `prefers-reduced-motion` respected | Entry animations disabled via `@media` query |

---

## Production Pitfalls

**1. The portal renders inside a stacking context and gets clipped**
If the palette is rendered inside a component that has `transform`, `filter`, or `will-change` applied, `position: fixed` coordinates are relative to that element, not the viewport. The panel appears at the wrong position or clips behind a sibling. Fix: always render via a React portal to `document.body` or use Angular CDK's `Overlay` service which appends to a dedicated `cdk-overlay-container` at the body level.

**2. `mousedown` vs `click` causes the input to lose focus**
A `click` event on a result fires `mousedown` first, which blurs the input, which fires a `blur` event, which may close the palette before the `click` fires. Fix: always handle result selection on `mousedown` and call `e.preventDefault()` — this suppresses the focus transfer while still registering the interaction.

**3. Virtual list is not needed until it is too late**
Rendering 500+ rows with DOM elements causes layout thrash during every ArrowDown keystroke because the browser must recalculate the scroll container height. Fix: if the command registry can exceed ~150 items, implement a virtual list. Cap the visible window to `VISIBLE_ROWS` rows (8 is conventional) and use `scrollIntoView({ block: 'nearest' })` rather than computing `scrollTop` manually — `nearest` only scrolls when the item is actually off-screen.

**4. Fuzzy search blocks the main thread on large registries**
Scoring 5,000 commands synchronously on every keystroke can drop below 60 fps. Fix: debounce the query update by 50–80 ms, or move the scoring loop into a Web Worker and `postMessage` the results back. For most apps the synchronous path is fine up to ~2,000 commands.

**5. Recent items grow stale after commands are removed**
If a user records "deploy:production" in recents and you remove that command in the next release, the palette will silently skip the id without showing an error — but the slot is wasted. Fix: filter recents against the live command registry when computing results (the `find()` call in the `results` computed already handles this — ensure it is always present).

**6. Global shortcut conflicts with browser or OS shortcuts**
⌘K is already "insert link" in many rich text editors (Notion, Google Docs, Slack). Registering it globally with `e.preventDefault()` will break those. Fix: only intercept ⌘K when the active element is not a `contenteditable`, `[role="textbox"]`, or `<textarea>` — or follow VS Code's approach and use ⌘⇧P as the palette shortcut instead, which has fewer conflicts.

**7. Overlay body scroll is not locked**
Unlike a full modal, a command palette usually does not need body scroll lock — the overlay backdrop covers everything. But if the app has fixed-position sidebars or floating chat widgets with their own scroll containers, the backdrop `z-index` must exceed those containers. Maintain a z-index scale (e.g., dialog = 9999) documented in design tokens to avoid arms races.

---

## Interview Angle

**Q: "Walk me through how you'd build a ⌘K command palette from scratch — what are the hard parts?"**

Strong answers identify four non-obvious challenges:
1. **Portal placement** — the overlay must escape any CSS stacking context in the component tree. `createPortal(el, document.body)` in React and Angular CDK Overlay both guarantee this. Explain what happens if you skip this (fixed positioning breaks inside `transform` parents).
2. **Fuzzy search scoring** — a naive prefix filter feels broken to power users. Explain that you score on consecutive character runs, word-boundary matches, and exact substring presence. The score determines sort order; the matched indices drive the `<mark>` highlight spans. Never use a regex for fuzzy matching — it cannot produce a score.
3. **Keyboard contract** — ArrowUp/Down move the active index, Enter fires the active command, Escape closes. The `aria-activedescendant` attribute must mirror the active index so VoiceOver announces the item as the user navigates. `mousedown` (not `click`) is required to prevent input blur from racing against selection.
4. **Recent items** — persist to `localStorage` keyed by command id; float them to the top when the query is empty; always filter against the live command registry to discard stale ids. This turns a useful feature into a productivity multiplier for daily users.

**Follow-up: "How would you handle a registry of 10,000 commands without dropping frames?"**

Debounce the query signal by 60 ms to reduce scoring frequency, then cap DOM output to ~8 visible rows (virtual list). For genuine 10k+ registries, move scoring to a Web Worker with `postMessage` — the worker receives `{ query, commands }` and returns `ScoredCommand[]`. Use `useDeferredValue` (React 18) or Angular's `asyncPipe` with an `Observable` + `debounceTime` to keep the input responsive while the worker computes in the background.
