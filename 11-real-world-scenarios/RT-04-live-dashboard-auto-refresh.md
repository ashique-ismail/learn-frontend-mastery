# Live Dashboard with Auto-Refresh

## The Idea

**In plain English:** You have a dashboard — metrics, stats, charts — that must stay fresh without the user manually reloading the page. You poll an API on a timer. When the API fails, you back off exponentially so you don't hammer a struggling server. While the tab is hidden you pause polling entirely. When the tab becomes visible again you resume immediately. You show the user a "last updated" timestamp so they always know how stale the data is. There is also a manual refresh button for impatient users.

**Real-world analogy:** Think of a stock ticker board at an airport currency exchange desk.

- The board updates every 30 seconds automatically — that is your **polling interval**.
- If the data feed goes down, the staff don't keep hammering the supplier every second. They wait 1 minute, then 2, then 4, doubling each time — that is **exponential backoff**.
- When the airport closes at night (tab hidden), the board goes dark — **pausing polling**. When it opens in the morning (tab visible again) it immediately refreshes before returning to its normal schedule — **resume on focus**.
- A small timestamp in the corner reads "Last updated: 14:32" — that is the **stale-data indicator**.
- There is also a physical refresh button the staff can hit manually — that is the **manual refresh trigger**.
- The key insight: polling is not just "call `setInterval` and forget". It is a state machine: polling, paused, backing-off, error, and idle.

---

## Learning Objectives

- Understand why naive `setInterval` is dangerous for production polling
- Implement exponential backoff with jitter on API failure
- Use the `visibilitychange` event to pause and resume polling correctly
- Build a stale-data indicator that accurately tracks the last successful fetch
- Wire a manual refresh button that resets the backoff state
- Know the React `useInterval` hook pattern and why `useEffect` + `setTimeout` beats `setInterval`
- Implement the same pattern in Angular using RxJS `timer`, `retryWhen`, and `takeUntil`

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Poll an API on a timer | ❌ | CSS has no networking or async capability |
| Exponential backoff on failure | ❌ | Requires stateful retry logic |
| Detect tab visibility changes | ❌ | Requires the `visibilitychange` DOM event |
| Display time elapsed since last update | ❌ | Requires JS date arithmetic |
| Cancel in-flight requests on unmount | ❌ | Requires `AbortController` in JS |
| Show stale indicator after N seconds | ⚠️ | CSS `@keyframes` can animate, but cannot know if data is actually stale |

---

## HTML & CSS Foundation

### Markup Structure

```html
<!-- Dashboard shell -->
<section class="dashboard" aria-label="Live metrics dashboard">

  <!-- Header bar with status and controls -->
  <header class="dashboard-header">
    <h1 class="dashboard-title">System Metrics</h1>
    <div class="dashboard-status" role="status" aria-live="polite">
      <!-- JS populates: "Updated 12s ago" or "Retrying in 30s…" -->
    </div>
    <button
      class="refresh-btn"
      aria-label="Refresh dashboard data"
      type="button"
    >
      <svg class="refresh-icon" aria-hidden="true" viewBox="0 0 24 24" width="18" height="18">
        <path d="M17.65 6.35A8 8 0 1 0 19.73 14H17.65A6 6 0 1 1 12 6a5.92 5.92 0 0 1 4.24 1.76L13 11h7V4l-2.35 2.35z"/>
      </svg>
      Refresh
    </button>
  </header>

  <!-- Stale overlay: shown when data is older than threshold -->
  <div class="stale-banner" role="alert" aria-live="assertive" hidden>
    Data may be outdated. Last update was over 2 minutes ago.
  </div>

  <!-- Metric cards — populated by JS -->
  <div class="metrics-grid">
    <article class="metric-card" aria-labelledby="metric-cpu-label">
      <h2 id="metric-cpu-label" class="metric-label">CPU Usage</h2>
      <p class="metric-value" aria-live="polite">—</p>
    </article>
    <article class="metric-card" aria-labelledby="metric-mem-label">
      <h2 id="metric-mem-label" class="metric-label">Memory</h2>
      <p class="metric-value" aria-live="polite">—</p>
    </article>
    <article class="metric-card" aria-labelledby="metric-rps-label">
      <h2 id="metric-rps-label" class="metric-label">Req / s</h2>
      <p class="metric-value" aria-live="polite">—</p>
    </article>
  </div>

</section>
```

### CSS

```css
/* ─── Tokens ─── */
:root {
  --card-bg: #ffffff;
  --card-border: #e2e8f0;
  --stale-color: #b45309;
  --error-color: #dc2626;
  --ok-color: #16a34a;
  --transition: 0.2s ease;
}

/* ─── Dashboard shell ─── */
.dashboard {
  display: grid;
  gap: 1.5rem;
  padding: 1.5rem;
  max-width: 1200px;
  margin: 0 auto;
}

/* ─── Header ─── */
.dashboard-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.dashboard-title {
  font-size: 1.25rem;
  font-weight: 700;
  margin: 0;
  flex: 1;
}

/* Status pill */
.dashboard-status {
  font-size: 0.8125rem;
  color: #64748b;
  display: flex;
  align-items: center;
  gap: 0.375rem;
}

/* Pulsing dot — green when fresh, amber when stale, red on error */
.dashboard-status::before {
  content: '';
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--ok-color);
  flex-shrink: 0;
}

.dashboard-status[data-state="stale"]::before  { background: var(--stale-color); }
.dashboard-status[data-state="error"]::before  { background: var(--error-color); }
.dashboard-status[data-state="polling"]::before {
  background: var(--ok-color);
  animation: pulse 1.2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.3; }
}

/* ─── Refresh button ─── */
.refresh-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.375rem;
  padding: 0.4rem 0.875rem;
  font-size: 0.875rem;
  background: #f1f5f9;
  border: 1px solid var(--card-border);
  border-radius: 6px;
  cursor: pointer;
  transition: background var(--transition), transform var(--transition);
}

.refresh-btn:hover { background: #e2e8f0; }

/* Spin the icon during loading */
.refresh-btn[data-loading="true"] .refresh-icon {
  animation: spin 0.8s linear infinite;
}

.refresh-btn[data-loading="true"] {
  pointer-events: none;
  opacity: 0.7;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* ─── Stale banner ─── */
.stale-banner {
  padding: 0.75rem 1rem;
  background: #fef3c7;
  border: 1px solid #fcd34d;
  border-radius: 6px;
  color: var(--stale-color);
  font-size: 0.875rem;
}

.stale-banner[hidden] { display: none; }

/* ─── Metrics grid ─── */
.metrics-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem;
}

.metric-card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: 10px;
  padding: 1.25rem;
  transition: opacity var(--transition);
}

/* Dim cards while a refresh is in flight */
.metric-card[data-loading="true"] {
  opacity: 0.5;
}

.metric-label {
  font-size: 0.8125rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: #64748b;
  margin: 0 0 0.5rem;
}

.metric-value {
  font-size: 2rem;
  font-weight: 700;
  margin: 0;
  font-variant-numeric: tabular-nums;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .refresh-icon,
  .dashboard-status::before {
    animation: none !important;
  }
}
```

**What CSS owns:** card layout, status indicator color and pulse animation, spinner on the refresh button, stale banner visibility, reduced-motion fallback.

**What CSS cannot own:** when to show/hide the stale banner, what the status text says, polling timing, backoff logic, or any data fetching.

---

## React Implementation

### Types

```tsx
// dashboard.types.ts
export interface MetricSnapshot {
  cpu: number;       // percentage 0–100
  memory: number;    // percentage 0–100
  requestsPerSec: number;
  fetchedAt: Date;
}

export type PollingState =
  | { status: 'idle' }
  | { status: 'polling' }
  | { status: 'loading' }
  | { status: 'error'; retryAt: Date; attempt: number }
  | { status: 'stale' };
```

### The Core Hook — `useDashboardPolling`

```tsx
// useDashboardPolling.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import type { MetricSnapshot, PollingState } from './dashboard.types';

const POLL_INTERVAL_MS     = 30_000;   // 30 s between successful polls
const STALE_THRESHOLD_MS   = 120_000;  // 2 min without update = stale
const BASE_BACKOFF_MS      = 5_000;    // 5 s initial retry delay
const MAX_BACKOFF_MS        = 300_000; // cap at 5 min

function exponentialBackoff(attempt: number): number {
  // Full jitter: random value in [0, min(cap, base * 2^attempt)]
  const ceiling = Math.min(MAX_BACKOFF_MS, BASE_BACKOFF_MS * 2 ** attempt);
  return Math.random() * ceiling;
}

async function fetchMetrics(signal: AbortSignal): Promise<MetricSnapshot> {
  const res = await fetch('/api/metrics', { signal });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const data = await res.json();
  return { ...data, fetchedAt: new Date() };
}

export function useDashboardPolling() {
  const [data,         setData]         = useState<MetricSnapshot | null>(null);
  const [pollingState, setPollingState] = useState<PollingState>({ status: 'idle' });

  // Refs so callbacks always see latest values without re-creating effects
  const attemptRef    = useRef(0);
  const timerRef      = useRef<ReturnType<typeof setTimeout> | null>(null);
  const abortRef      = useRef<AbortController | null>(null);
  const isMountedRef  = useRef(true);

  const clearTimer = () => {
    if (timerRef.current !== null) {
      clearTimeout(timerRef.current);
      timerRef.current = null;
    }
  };

  const poll = useCallback(async () => {
    if (!isMountedRef.current) return;

    // Cancel any in-flight request
    abortRef.current?.abort();
    abortRef.current = new AbortController();

    setPollingState({ status: 'loading' });

    try {
      const snapshot = await fetchMetrics(abortRef.current.signal);

      if (!isMountedRef.current) return;

      attemptRef.current = 0;          // reset backoff on success
      setData(snapshot);
      setPollingState({ status: 'polling' });

      // Schedule next regular poll
      clearTimer();
      timerRef.current = setTimeout(poll, POLL_INTERVAL_MS);

    } catch (err) {
      if (!isMountedRef.current) return;
      // Ignore AbortError — component unmounted or manual refresh triggered
      if (err instanceof Error && err.name === 'AbortError') return;

      const attempt = attemptRef.current;
      const delay   = exponentialBackoff(attempt);
      attemptRef.current = attempt + 1;

      const retryAt = new Date(Date.now() + delay);
      setPollingState({ status: 'error', retryAt, attempt });

      clearTimer();
      timerRef.current = setTimeout(poll, delay);
    }
  }, []);

  // Manual refresh: abort current backoff timer and poll immediately
  const refresh = useCallback(() => {
    clearTimer();
    attemptRef.current = 0;
    poll();
  }, [poll]);

  // Pause / resume based on tab visibility
  useEffect(() => {
    const onVisibilityChange = () => {
      if (document.visibilityState === 'visible') {
        // Resume: poll immediately on tab focus
        clearTimer();
        attemptRef.current = 0;
        poll();
      } else {
        // Pause: cancel pending timer and in-flight request
        clearTimer();
        abortRef.current?.abort();
        setPollingState({ status: 'idle' });
      }
    };

    document.addEventListener('visibilitychange', onVisibilityChange);
    return () => document.removeEventListener('visibilitychange', onVisibilityChange);
  }, [poll]);

  // Stale indicator: re-evaluate every 15 s
  useEffect(() => {
    const staleCheck = setInterval(() => {
      if (!data) return;
      const age = Date.now() - data.fetchedAt.getTime();
      if (age > STALE_THRESHOLD_MS) {
        setPollingState(prev =>
          prev.status === 'error' ? prev : { status: 'stale' }
        );
      }
    }, 15_000);
    return () => clearInterval(staleCheck);
  }, [data]);

  // Start polling on mount; clean up on unmount
  useEffect(() => {
    isMountedRef.current = true;
    if (document.visibilityState === 'visible') poll();

    return () => {
      isMountedRef.current = false;
      clearTimer();
      abortRef.current?.abort();
    };
  }, [poll]);

  return { data, pollingState, refresh };
}
```

### Dashboard Component

```tsx
// LiveDashboard.tsx
import { useEffect, useRef } from 'react';
import { useDashboardPolling } from './useDashboardPolling';
import type { MetricSnapshot, PollingState } from './dashboard.types';

function statusLabel(state: PollingState, data: MetricSnapshot | null): string {
  switch (state.status) {
    case 'idle':
      return 'Paused (tab hidden)';
    case 'loading':
      return data ? 'Refreshing…' : 'Loading…';
    case 'polling': {
      const secs = Math.round((Date.now() - (data?.fetchedAt.getTime() ?? 0)) / 1000);
      return `Updated ${secs}s ago`;
    }
    case 'error': {
      const secsUntil = Math.max(
        0,
        Math.round((state.retryAt.getTime() - Date.now()) / 1000)
      );
      return `Error — retrying in ${secsUntil}s (attempt ${state.attempt + 1})`;
    }
    case 'stale':
      return 'Data may be outdated';
  }
}

export function LiveDashboard() {
  const { data, pollingState, refresh } = useDashboardPolling();
  const isLoading = pollingState.status === 'loading';
  const isStale   = pollingState.status === 'stale';
  const isError   = pollingState.status === 'error';

  // Force status label to re-render every second for live "Xs ago" display
  const [, forceUpdate] = useReducerTick();

  const statusEl = statusLabel(pollingState, data);

  return (
    <section className="dashboard" aria-label="Live metrics dashboard">
      <header className="dashboard-header">
        <h1 className="dashboard-title">System Metrics</h1>

        <div
          className="dashboard-status"
          role="status"
          aria-live="polite"
          data-state={isError ? 'error' : isStale ? 'stale' : 'polling'}
        >
          {statusEl}
        </div>

        <button
          className="refresh-btn"
          aria-label="Refresh dashboard data"
          data-loading={isLoading ? 'true' : undefined}
          onClick={refresh}
          disabled={isLoading}
        >
          <svg className="refresh-icon" aria-hidden="true" viewBox="0 0 24 24" width="18" height="18">
            <path d="M17.65 6.35A8 8 0 1 0 19.73 14H17.65A6 6 0 1 1 12 6a5.92 5.92 0 0 1 4.24 1.76L13 11h7V4l-2.35 2.35z"/>
          </svg>
          Refresh
        </button>
      </header>

      {/* Stale banner */}
      {isStale && (
        <div className="stale-banner" role="alert" aria-live="assertive">
          Data may be outdated. Last update was over 2 minutes ago.
        </div>
      )}

      {/* Metric cards */}
      <div className="metrics-grid">
        <MetricCard
          id="metric-cpu"
          label="CPU Usage"
          value={data ? `${data.cpu}%` : '—'}
          loading={isLoading}
        />
        <MetricCard
          id="metric-mem"
          label="Memory"
          value={data ? `${data.memory}%` : '—'}
          loading={isLoading}
        />
        <MetricCard
          id="metric-rps"
          label="Req / s"
          value={data ? `${data.requestsPerSec}` : '—'}
          loading={isLoading}
        />
      </div>
    </section>
  );
}

/* ── Metric card ── */
interface MetricCardProps {
  id: string;
  label: string;
  value: string;
  loading: boolean;
}

function MetricCard({ id, label, value, loading }: MetricCardProps) {
  return (
    <article className="metric-card" aria-labelledby={`${id}-label`} data-loading={loading ? 'true' : undefined}>
      <h2 id={`${id}-label`} className="metric-label">{label}</h2>
      <p className="metric-value" aria-live="polite">{value}</p>
    </article>
  );
}

/* ── Utility: force re-render every second for live timestamps ── */
function useReducerTick(): [number, () => void] {
  const [tick, setTick] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setTick(t => t + 1), 1000);
    return () => clearInterval(id);
  }, []);
  return [tick, () => setTick(t => t + 1)];
}

// Missing import
import { useState } from 'react';
```

---

## Angular Implementation

### Model & Service

```typescript
// dashboard.model.ts
export interface MetricSnapshot {
  cpu: number;
  memory: number;
  requestsPerSec: number;
  fetchedAt: Date;
}

export type DashboardStatus = 'idle' | 'loading' | 'polling' | 'error' | 'stale';
```

```typescript
// dashboard.service.ts
import { Injectable, signal, computed, inject, OnDestroy } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import {
  Subject, Subscription, timer, EMPTY, fromEvent
} from 'rxjs';
import {
  switchMap, catchError, retry, tap, takeUntil
} from 'rxjs/operators';
import type { MetricSnapshot, DashboardStatus } from './dashboard.model';

const POLL_INTERVAL_MS   = 30_000;
const STALE_THRESHOLD_MS = 120_000;

@Injectable({ providedIn: 'root' })
export class DashboardService implements OnDestroy {
  private http     = inject(HttpClient);
  private destroy$ = new Subject<void>();
  private refresh$ = new Subject<void>();

  // ── Signals ──────────────────────────────────────────────────────────────
  private _data     = signal<MetricSnapshot | null>(null);
  private _status   = signal<DashboardStatus>('idle');
  private _attempt  = signal(0);
  private _retryAt  = signal<Date | null>(null);

  readonly data     = this._data.asReadonly();
  readonly status   = this._status.asReadonly();
  readonly attempt  = this._attempt.asReadonly();
  readonly retryAt  = this._retryAt.asReadonly();

  readonly isStale  = computed(() => this._status() === 'stale');
  readonly isError  = computed(() => this._status() === 'error');

  private staleCheckSub?: Subscription;
  private pollSub?: Subscription;

  startPolling() {
    this.startStaleCheck();
    this.subscribeToVisibility();
    if (document.visibilityState === 'visible') this.triggerPoll();
  }

  manualRefresh() {
    this._attempt.set(0);
    this.refresh$.next();
    this.triggerPoll();
  }

  private triggerPoll() {
    this.pollSub?.unsubscribe();

    this.pollSub = timer(0, POLL_INTERVAL_MS).pipe(
      // Manual refresh restarts the entire timer stream
      switchMap(() =>
        this.http.get<Omit<MetricSnapshot, 'fetchedAt'>>('/api/metrics').pipe(
          tap(raw => {
            const snapshot: MetricSnapshot = { ...raw, fetchedAt: new Date() };
            this._data.set(snapshot);
            this._status.set('polling');
            this._attempt.set(0);
          }),
          catchError((err, caught$) => {
            const attempt = this._attempt();
            const delay   = this.backoff(attempt);
            this._attempt.update(a => a + 1);
            this._retryAt.set(new Date(Date.now() + delay));
            this._status.set('error');
            // Retry after computed delay — RxJS retryWhen equivalent via timer
            return timer(delay).pipe(switchMap(() => caught$));
          })
        )
      ),
      takeUntil(this.destroy$)
    ).subscribe();
  }

  private subscribeToVisibility() {
    fromEvent(document, 'visibilitychange').pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => {
      if (document.visibilityState === 'visible') {
        this._attempt.set(0);
        this.triggerPoll();
      } else {
        this.pollSub?.unsubscribe();
        this._status.set('idle');
      }
    });
  }

  private startStaleCheck() {
    this.staleCheckSub = timer(15_000, 15_000).pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => {
      const snapshot = this._data();
      if (!snapshot) return;
      const age = Date.now() - snapshot.fetchedAt.getTime();
      if (age > STALE_THRESHOLD_MS && this._status() === 'polling') {
        this._status.set('stale');
      }
    });
  }

  private backoff(attempt: number): number {
    const ceiling = Math.min(300_000, 5_000 * 2 ** attempt);
    return Math.random() * ceiling;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### Dashboard Component

```typescript
// live-dashboard.component.ts
import {
  Component, OnInit, inject, ChangeDetectionStrategy,
  signal, computed, effect
} from '@angular/core';
import { AsyncPipe, DatePipe } from '@angular/common';
import { DashboardService } from './dashboard.service';

@Component({
  selector: 'app-live-dashboard',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [DatePipe],
  template: `
    <section class="dashboard" aria-label="Live metrics dashboard">

      <header class="dashboard-header">
        <h1 class="dashboard-title">System Metrics</h1>

        <div
          class="dashboard-status"
          role="status"
          aria-live="polite"
          [attr.data-state]="svc.isError() ? 'error' : svc.isStale() ? 'stale' : 'polling'"
        >
          {{ statusLabel() }}
        </div>

        <button
          class="refresh-btn"
          aria-label="Refresh dashboard data"
          [attr.data-loading]="svc.status() === 'loading' ? 'true' : null"
          [disabled]="svc.status() === 'loading'"
          (click)="svc.manualRefresh()"
        >
          <svg class="refresh-icon" aria-hidden="true" viewBox="0 0 24 24" width="18" height="18">
            <path d="M17.65 6.35A8 8 0 1 0 19.73 14H17.65A6 6 0 1 1 12 6a5.92 5.92 0 0 1 4.24 1.76L13 11h7V4l-2.35 2.35z"/>
          </svg>
          Refresh
        </button>
      </header>

      @if (svc.isStale()) {
        <div class="stale-banner" role="alert" aria-live="assertive">
          Data may be outdated. Last update was over 2 minutes ago.
        </div>
      }

      <div class="metrics-grid">
        <article class="metric-card"
          aria-labelledby="metric-cpu-label"
          [attr.data-loading]="svc.status() === 'loading' ? 'true' : null"
        >
          <h2 id="metric-cpu-label" class="metric-label">CPU Usage</h2>
          <p class="metric-value" aria-live="polite">
            {{ svc.data()?.cpu != null ? svc.data()!.cpu + '%' : '—' }}
          </p>
        </article>

        <article class="metric-card"
          aria-labelledby="metric-mem-label"
          [attr.data-loading]="svc.status() === 'loading' ? 'true' : null"
        >
          <h2 id="metric-mem-label" class="metric-label">Memory</h2>
          <p class="metric-value" aria-live="polite">
            {{ svc.data()?.memory != null ? svc.data()!.memory + '%' : '—' }}
          </p>
        </article>

        <article class="metric-card"
          aria-labelledby="metric-rps-label"
          [attr.data-loading]="svc.status() === 'loading' ? 'true' : null"
        >
          <h2 id="metric-rps-label" class="metric-label">Req / s</h2>
          <p class="metric-value" aria-live="polite">
            {{ svc.data()?.requestsPerSec ?? '—' }}
          </p>
        </article>
      </div>

    </section>
  `,
})
export class LiveDashboardComponent implements OnInit {
  svc = inject(DashboardService);

  // Tick signal — incremented every second to keep "Xs ago" live
  private tick = signal(0);

  constructor() {
    setInterval(() => this.tick.update(t => t + 1), 1000);
  }

  readonly statusLabel = computed(() => {
    void this.tick();   // declare dependency so computed re-evaluates each tick
    const status   = this.svc.status();
    const data     = this.svc.data();
    const retryAt  = this.svc.retryAt();
    const attempt  = this.svc.attempt();

    switch (status) {
      case 'idle':    return 'Paused (tab hidden)';
      case 'loading': return data ? 'Refreshing…' : 'Loading…';
      case 'polling': {
        const secs = Math.round((Date.now() - (data?.fetchedAt.getTime() ?? 0)) / 1000);
        return `Updated ${secs}s ago`;
      }
      case 'error': {
        const secsUntil = retryAt
          ? Math.max(0, Math.round((retryAt.getTime() - Date.now()) / 1000))
          : 0;
        return `Error — retrying in ${secsUntil}s (attempt ${attempt + 1})`;
      }
      case 'stale': return 'Data may be outdated';
    }
  });

  ngOnInit() {
    this.svc.startPolling();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Status bar announced to screen readers on every change | `role="status"` with `aria-live="polite"` — screen reader reads "Updated 12s ago" on each poll |
| Stale / critical errors use assertive live region | `role="alert"` with `aria-live="assertive"` on the stale banner |
| Individual metric values announced when they change | `aria-live="polite"` on each `.metric-value` paragraph |
| Refresh button is keyboard accessible and has a clear label | `<button>` element with `aria-label="Refresh dashboard data"` |
| Refresh button disabled and non-interactive while loading | `disabled` attribute prevents double-click polling; CSS hides it visually |
| No ARIA role on the metrics grid (decorative layout) | Grid uses no ARIA roles — screen reader traverses cards by heading hierarchy |
| Each metric card has a labelled region | `<article aria-labelledby>` points to the `<h2>` inside the card |
| Status dot color is never the sole indicator | Text description accompanies the colored dot at all times |
| Reduced motion: pulse and spin animations disabled | `@media (prefers-reduced-motion: reduce)` removes all animations |
| Tab hidden pausing does not surprise screen reader users | Status area announces "Paused (tab hidden)" when polling stops |

---

## Production Pitfalls

**1. Using `setInterval` directly instead of chained `setTimeout`**
`setInterval` fires on a fixed clock regardless of how long each fetch takes. If a request takes 25 s and your interval is 30 s, you get only 5 s of "breathing room" — or worse, two requests overlap. Use `setTimeout` scheduled after each response resolves. The React hook and Angular service above both use this pattern.

**2. Not aborting in-flight requests on unmount**
If the component unmounts while a fetch is in-flight, the `then()` callback fires and calls `setState` on an unmounted component (React), leaking memory and causing warnings. Always create a fresh `AbortController` per fetch and call `abort()` in the cleanup function. The `destroy$` Subject in the Angular service does the same via `takeUntil`.

**3. Exponential backoff without jitter causes thundering herd**
If ten dashboard instances all fail at the same time (e.g., a brief network blip) and all retry after exactly 5 s, they hammer the server together again. Adding full jitter — random delay in `[0, cap]` — desynchronises retries. The `exponentialBackoff()` helper above implements this.

**4. Polling continues in hidden tabs draining batteries and server budget**
Mobile browsers throttle `setInterval` in background tabs to every 1+ minute, but they do not stop it. Pausing explicitly on `visibilitychange` is cleaner and reduces unnecessary load on your API. Always resume immediately on tab focus to avoid the user seeing stale data after switching back.

**5. Stale-data indicator based only on a timer, not actual data age**
A timer-based stale check that runs every 15 s is fine, but it must compare `Date.now()` against `data.fetchedAt`, not against when the timer started. If the user was on a different tab for 10 minutes, the very first stale check after focus must catch the staleness immediately.

**6. Status label causes layout shift when text length changes**
"Updated 3s ago" is shorter than "Error — retrying in 127s (attempt 4)". If the header is not flex-wrapped, this can push the refresh button around. Reserve minimum width on the status container or use a fixed-width monospace number format for countdowns.

**7. Manual refresh does not reset backoff attempt counter**
If the user hits refresh while in an error/backoff state, the next failure should start backoff from attempt 0, not continue from where it was. Always reset `attemptRef.current = 0` (React) or `this._attempt.set(0)` (Angular) before kicking off a manual refresh.

---

## Interview Angle

**Q: "How would you implement a live dashboard that polls an API, handles errors gracefully, and doesn't waste resources when the user isn't looking at the page?"**

A complete answer covers four layers. First, the polling loop itself: prefer chained `setTimeout` over `setInterval` so each request fully completes before the next one is scheduled, preventing overlapping requests. Second, error resilience: on failure, implement exponential backoff with full jitter — doubling the delay cap each attempt and randomising within it — to avoid thundering herd when a server recovers and many clients retry simultaneously. Third, tab awareness: listen to `document.visibilitychange`; pause the loop and abort in-flight requests when the tab is hidden, resume and poll immediately when it becomes visible again. Finally, the user experience layer: track `fetchedAt` on each successful response to drive a live "Updated Xs ago" label and a stale banner after a configurable threshold, with a manual refresh button that resets the backoff state.

**Follow-up: "What breaks in production that a demo wouldn't reveal?"**
Three things surface in production but not in demos. First, requests that overlap when `setInterval` drift is not accounted for on slow networks. Second, memory leaks in React from `setState` calls after unmount when `AbortController` cleanup is skipped. Third, the thundering herd problem on server recovery — if you used a fixed retry delay instead of jittered backoff, all clients retry simultaneously and you experience the same failure immediately again.
