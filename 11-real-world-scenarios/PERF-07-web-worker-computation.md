# Web Worker for Heavy Computation

## The Idea

**In plain English:** JavaScript is single-threaded. When you run a slow calculation on the main thread — parsing a 50,000-row CSV, running an image convolution, encrypting a large file — the browser freezes. No scroll, no clicks, no animations until the loop finishes. A Web Worker moves that computation onto a separate OS thread. The UI stays fluid while the worker grinds through the data. Comlink makes calling worker functions feel like calling a local `async` function instead of passing raw message envelopes.

**Real-world analogy:** Think of a restaurant kitchen.

- The **main thread** is the head waiter — they greet guests, take orders, and serve dishes. If they had to cook every meal themselves, the restaurant grinds to a halt.
- The **Web Worker** is the back-of-house kitchen. The waiter passes an order ticket (`postMessage`), the kitchen cooks it (heavy computation), then slides the dish back (`postMessage` response). The waiter keeps running the floor the whole time.
- **Comlink** is a headset system. Instead of manually passing tickets back and forth, the waiter just says "cook the steak and call me when done" — a normal function call that returns a Promise.
- **Transferable objects** (ArrayBuffer, ImageBitmap) = sliding the physical ticket across the pass rather than photocopying it. Ownership transfers instantly — zero bytes copied — which is critical for 100 MB binary payloads.
- **Terminating the worker on unmount** = telling the kitchen to stop if the customer already left the building. Zombie workers burn CPU and memory with no benefit.

The key insight: the main thread owns the DOM and reactivity; the worker owns raw computation. Data flows between them via structured-clone (copy) or transferables (zero-copy move).

---

## Learning Objectives

- Understand why long-running synchronous code blocks the browser event loop
- Spin up a Web Worker using the Vite / Webpack `new URL(…, import.meta.url)` pattern
- Use the Comlink library to expose a worker module as an RPC interface
- Send large binary data via `Comlink.transfer` to avoid the copy overhead
- Proxy a progress callback from the main thread into the worker with `Comlink.proxy`
- Terminate the worker cleanly when the React component unmounts or Angular service is destroyed
- Implement both React (hooks) and Angular (signal-based service) patterns

---

## Why CSS Alone Isn't Enough

This topic is entirely a JavaScript and threading concern — CSS only styles the progress indicator. The table below shows what the browser can and cannot do without moving work off the main thread.

| Requirement | CSS can do it? | Why not |
| --- | --- | --- |
| Style a progress bar | ✅ | — |
| Animate a spinner while work runs | ⚠️ CSS animation pauses | JS-heavy main thread starves the compositor |
| Run 50,000-row CSV parse without freezing | ❌ | CSS has no computation model |
| Report progress mid-computation | ❌ | Requires a worker `postMessage` |
| Prevent UI jank during image processing | ❌ | Requires off-main-thread execution |
| Transfer binary data without copying | ❌ | Requires transferable objects + `postMessage` |

**Conclusion:** CSS owns the visual loading state. Web Workers own the computation. Without a worker, even a CSS spinner freezes because the compositor thread is starved when the main thread is saturated.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- File chooser — visually hidden, triggered by label -->
<input type="file" id="file-input" accept=".csv" class="file-input" />
<label for="file-input" class="btn-upload">Choose CSV file</label>

<!-- Progress region — shown while worker is processing -->
<div class="worker-progress" hidden aria-live="polite">
  <div
    class="progress-bar-wrapper"
    role="progressbar"
    aria-valuenow="0"
    aria-valuemin="0"
    aria-valuemax="100"
    aria-label="Processing progress"
  >
    <div class="progress-bar-fill"></div>
  </div>
  <p class="progress-label">Processing… <span class="progress-pct">0</span>%</p>
</div>

<!-- Result region — shown when done -->
<div class="worker-result" hidden role="region" aria-label="Processing result" aria-live="polite">
</div>

<!-- Error region -->
<p class="worker-error" hidden role="alert"></p>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --progress-track:  #e5e7eb;
  --progress-fill:   #3b82f6;
  --progress-height: 10px;
}

/* ─── Visually-hidden file input ─── */
.file-input {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}

.btn-upload {
  display: inline-flex;
  align-items: center;
  padding: 0.625rem 1.25rem;
  background: var(--progress-fill);
  color: #fff;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background 0.15s;
}
.btn-upload:hover { background: #2563eb; }

/* Focus ring for keyboard users who Tab to the label */
.file-input:focus-visible + .btn-upload {
  outline: 2px solid #2563eb;
  outline-offset: 2px;
}

/* ─── Progress bar ─── */
.progress-bar-wrapper {
  height: var(--progress-height);
  background: var(--progress-track);
  border-radius: 999px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  width: 0%;
  background: linear-gradient(90deg, var(--progress-fill), #60a5fa);
  border-radius: 999px;
  transition: width 0.1s ease;
  will-change: width;    /* hint compositor for smooth animation */
}

.progress-label {
  font-size: 0.875rem;
  color: #6b7280;
  margin: 0.375rem 0 0;
}

/* ─── Result ─── */
.worker-result {
  padding: 1rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  font-size: 0.9rem;
}

/* ─── Error ─── */
.worker-error { color: #dc2626; font-size: 0.9rem; }

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .progress-bar-fill { transition: none; }
}
```

**What CSS owns:** progress bar fill animation, button hover state, focus ring, reduced-motion opt-out.

**What CSS cannot own:** triggering the worker, streaming progress updates, transferring binary data, terminating the worker on unmount.

---

## React Implementation

### The Worker File

```typescript
// workers/csv-processor.worker.ts
// IMPORTANT: Never import React, DOM APIs, or any module that references `window`.
// This file executes on a WorkerGlobalScope, not in the browser window.

import * as Comlink from 'comlink';

export interface ProcessResult {
  rowCount: number;
  columns:  string[];
  sample:   Record<string, string>[];
  stats:    Record<string, { min: number; max: number; mean: number }>;
}

/**
 * Process a CSV file passed as an ArrayBuffer.
 *
 * @param buffer      - Transferred (zero-copy) ArrayBuffer of the file bytes.
 * @param onProgress  - Comlink.proxy() callback; called on the main thread with 0–100.
 */
async function processCSV(
  buffer: ArrayBuffer,
  onProgress: (percent: number) => void
): Promise<ProcessResult> {
  const text    = new TextDecoder().decode(buffer);
  const lines   = text.split('\n').filter(l => l.trim().length > 0);

  if (lines.length < 2) throw new Error('File is empty or has no data rows.');

  const columns = lines[0].split(',').map(c => c.trim().replace(/^"|"$/g, ''));
  const rows: Record<string, string>[] = [];
  const total = lines.length - 1;

  for (let i = 1; i <= total; i++) {
    const cells = lines[i].split(',').map(c => c.trim().replace(/^"|"$/g, ''));
    const row: Record<string, string> = {};
    columns.forEach((col, j) => { row[col] = cells[j] ?? ''; });
    rows.push(row);

    // Report progress every 500 rows, not every row — cross-thread messages are not free.
    if (i % 500 === 0) {
      onProgress(Math.round((i / total) * 100));
      // Yield to allow other messages (e.g. terminate) to be processed.
      await new Promise<void>(resolve => setTimeout(resolve, 0));
    }
  }

  // Compute numeric statistics per column.
  const stats: ProcessResult['stats'] = {};
  for (const col of columns) {
    const nums = rows.map(r => parseFloat(r[col])).filter(v => !isNaN(v));
    if (nums.length > 0) {
      stats[col] = {
        min:  Math.min(...nums),
        max:  Math.max(...nums),
        mean: nums.reduce((a, b) => a + b, 0) / nums.length,
      };
    }
  }

  onProgress(100);

  return {
    rowCount: rows.length,
    columns,
    sample: rows.slice(0, 5),
    stats,
  };
}

// Expose the module via Comlink so the main thread can call it like an async function.
Comlink.expose({ processCSV });
```

### The Hook

```tsx
// useCSVWorker.ts
import { useRef, useState, useEffect, useCallback } from 'react';
import * as Comlink from 'comlink';
import type { ProcessResult } from './workers/csv-processor.worker';

type WorkerAPI = {
  processCSV: (
    buffer: ArrayBuffer,
    onProgress: (percent: number) => void
  ) => Promise<ProcessResult>;
};

type WorkerState =
  | { status: 'idle' }
  | { status: 'processing'; progress: number }
  | { status: 'done';       result: ProcessResult }
  | { status: 'error';      error: string };

export function useCSVWorker() {
  const [state, setState] = useState<WorkerState>({ status: 'idle' });

  const rawWorkerRef = useRef<Worker | null>(null);
  const apiRef       = useRef<Comlink.Remote<WorkerAPI> | null>(null);

  useEffect(() => {
    // Vite / Webpack syntax: the bundler emits the worker as a separate chunk.
    const worker = new Worker(
      new URL('./workers/csv-processor.worker.ts', import.meta.url),
      { type: 'module' }
    );
    rawWorkerRef.current = worker;
    apiRef.current       = Comlink.wrap<WorkerAPI>(worker);

    // Terminate the worker when the component unmounts.
    // Skipping this leaks a background OS thread for the lifetime of the page.
    return () => {
      worker.terminate();
      rawWorkerRef.current = null;
      apiRef.current       = null;
    };
  }, []);

  const process = useCallback(async (file: File) => {
    const api = apiRef.current;
    if (!api) return;

    setState({ status: 'processing', progress: 0 });

    // file.arrayBuffer() reads the file in the main thread first.
    // For very large files consider a streaming approach.
    const buffer = await file.arrayBuffer();

    try {
      const result = await api.processCSV(
        // Comlink.transfer(value, transferList): transfers ownership of `buffer`
        // to the worker thread. After this call, `buffer` on this side is detached
        // (byteLength === 0). No bytes are copied — O(1) regardless of file size.
        Comlink.transfer(buffer, [buffer]),

        // Comlink.proxy(fn): wraps the callback so the worker can call it across
        // the thread boundary. Without this, functions cannot be serialised.
        Comlink.proxy((percent: number) => {
          setState({ status: 'processing', progress: percent });
        })
      );
      setState({ status: 'done', result });
    } catch (err) {
      setState({
        status: 'error',
        error: err instanceof Error ? err.message : 'Unknown processing error',
      });
    }
  }, []);

  const reset = useCallback(() => setState({ status: 'idle' }), []);

  return { state, process, reset };
}
```

### The Component

```tsx
// CSVProcessor.tsx
import { useRef } from 'react';
import { useCSVWorker } from './useCSVWorker';

export function CSVProcessor() {
  const { state, process, reset } = useCSVWorker();
  const inputRef = useRef<HTMLInputElement>(null);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) process(file);
  };

  return (
    <div className="worker-demo" style={{ display: 'flex', flexDirection: 'column', gap: '1rem', maxWidth: 480 }}>

      {state.status === 'idle' && (
        <>
          <input
            ref={inputRef}
            type="file"
            id="file-input"
            accept=".csv"
            className="file-input"
            onChange={handleChange}
          />
          <label htmlFor="file-input" className="btn-upload">
            Choose CSV file
          </label>
        </>
      )}

      {state.status === 'processing' && (
        <div className="worker-progress" aria-live="polite">
          <div
            role="progressbar"
            aria-valuenow={state.progress}
            aria-valuemin={0}
            aria-valuemax={100}
            aria-label="Processing progress"
            className="progress-bar-wrapper"
          >
            <div className="progress-bar-fill" style={{ width: `${state.progress}%` }} />
          </div>
          <p className="progress-label">Processing… {state.progress}%</p>
        </div>
      )}

      {state.status === 'done' && (
        <div className="worker-result" role="region" aria-label="Processing result" aria-live="polite">
          <p><strong>{state.result.rowCount.toLocaleString()}</strong> rows processed</p>
          <p><strong>Columns:</strong> {state.result.columns.join(', ')}</p>
          {Object.keys(state.result.stats).length > 0 && (
            <details>
              <summary>Numeric stats</summary>
              <pre style={{ fontSize: '0.75rem', overflow: 'auto', maxHeight: 180 }}>
                {JSON.stringify(state.result.stats, null, 2)}
              </pre>
            </details>
          )}
          <details>
            <summary>Sample rows (first 5)</summary>
            <pre style={{ fontSize: '0.75rem', overflow: 'auto', maxHeight: 180 }}>
              {JSON.stringify(state.result.sample, null, 2)}
            </pre>
          </details>
          <button onClick={reset} style={{ marginTop: '0.75rem' }}>
            Process another file
          </button>
        </div>
      )}

      {state.status === 'error' && (
        <p role="alert" className="worker-error">{state.error}</p>
      )}

    </div>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// csv-worker.service.ts
import { Injectable, OnDestroy, signal } from '@angular/core';
import * as Comlink from 'comlink';

export interface ProcessResult {
  rowCount: number;
  columns:  string[];
  sample:   Record<string, string>[];
  stats:    Record<string, { min: number; max: number; mean: number }>;
}

type WorkerAPI = {
  processCSV: (
    buffer: ArrayBuffer,
    onProgress: (percent: number) => void
  ) => Promise<ProcessResult>;
};

@Injectable({ providedIn: 'root' })
export class CsvWorkerService implements OnDestroy {
  // Signal-based state — templates bind directly without AsyncPipe.
  readonly status   = signal<'idle' | 'processing' | 'done' | 'error'>('idle');
  readonly progress = signal(0);
  readonly result   = signal<ProcessResult | null>(null);
  readonly error    = signal<string | null>(null);

  private rawWorker: Worker | null = null;
  private api: Comlink.Remote<WorkerAPI> | null = null;

  constructor() {
    this.rawWorker = new Worker(
      new URL('./workers/csv-processor.worker.ts', import.meta.url),
      { type: 'module' }
    );
    this.api = Comlink.wrap<WorkerAPI>(this.rawWorker);
  }

  async processFile(file: File): Promise<void> {
    if (!this.api) return;

    this.status.set('processing');
    this.progress.set(0);
    this.error.set(null);
    this.result.set(null);

    try {
      const buffer = await file.arrayBuffer();
      const res = await this.api.processCSV(
        Comlink.transfer(buffer, [buffer]),
        Comlink.proxy((p: number) => this.progress.set(p))
      );
      this.result.set(res);
      this.status.set('done');
    } catch (err) {
      this.error.set(err instanceof Error ? err.message : 'Processing failed');
      this.status.set('error');
    }
  }

  reset(): void {
    this.status.set('idle');
    this.result.set(null);
    this.error.set(null);
    this.progress.set(0);
  }

  ngOnDestroy(): void {
    // Terminate the worker when the Angular application shuts down
    // or when a component-scoped provider is destroyed.
    this.rawWorker?.terminate();
    this.rawWorker = null;
    this.api       = null;
  }
}
```

### Standalone Component

```typescript
// csv-processor.component.ts
import { Component, inject } from '@angular/core';
import { DecimalPipe } from '@angular/common';
import { CsvWorkerService } from './csv-worker.service';

@Component({
  selector: 'app-csv-processor',
  standalone: true,
  imports: [DecimalPipe],
  template: `
    <div class="worker-demo">

      @switch (svc.status()) {

        @case ('idle') {
          <input
            type="file"
            id="file-input"
            accept=".csv"
            class="file-input"
            (change)="onFile($event)"
          />
          <label for="file-input" class="btn-upload">Choose CSV file</label>
        }

        @case ('processing') {
          <div class="worker-progress" aria-live="polite">
            <div
              role="progressbar"
              [attr.aria-valuenow]="svc.progress()"
              aria-valuemin="0"
              aria-valuemax="100"
              aria-label="Processing progress"
              class="progress-bar-wrapper"
            >
              <div class="progress-bar-fill" [style.width.%]="svc.progress()"></div>
            </div>
            <p class="progress-label">Processing… {{ svc.progress() }}%</p>
          </div>
        }

        @case ('done') {
          <div class="worker-result" role="region" aria-label="Processing result" aria-live="polite">
            <p>
              <strong>{{ svc.result()?.rowCount | number }}</strong> rows processed
            </p>
            <p><strong>Columns:</strong> {{ svc.result()?.columns?.join(', ') }}</p>
            <button (click)="svc.reset()">Process another file</button>
          </div>
        }

        @case ('error') {
          <p role="alert" class="worker-error">{{ svc.error() }}</p>
        }

      }

    </div>
  `,
})
export class CsvProcessorComponent {
  svc = inject(CsvWorkerService);

  onFile(e: Event): void {
    const file = (e.target as HTMLInputElement).files?.[0];
    if (file) this.svc.processFile(file);
  }
}
```

### Component-Scoped Worker (for multiple isolated instances)

```typescript
// When each component instance needs its own worker (e.g. parallel uploads),
// provide the service at component level rather than root.

@Component({
  selector: 'app-csv-processor',
  standalone: true,
  providers: [CsvWorkerService],   // <-- scoped to this component tree
  // ...
})
export class CsvProcessorComponent implements OnDestroy {
  svc = inject(CsvWorkerService);

  ngOnDestroy(): void {
    // Angular calls svc.ngOnDestroy() automatically because
    // the provider is component-scoped and destroyed with the component.
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
| --- | --- |
| Progress bar has semantic role | `role="progressbar"` on the track element |
| Progress value exposed to AT | `aria-valuenow`, `aria-valuemin`, `aria-valuemax` updated as work proceeds |
| Progress bar is labelled | `aria-label="Processing progress"` |
| Status updates announced without interrupting | `aria-live="polite"` on the progress container |
| Error announced immediately | `role="alert"` causes screen readers to interrupt and read the error |
| Result region is identifiable | `role="region"` + `aria-label="Processing result"` |
| File input is keyboard accessible | Native `<input type="file">` — visually hidden but reachable; `<label>` triggers it |
| Focus ring visible on label | `:focus-visible` ring applied when the hidden input is focused via keyboard |
| Spinner / animation respects motion preference | `transition: none` under `prefers-reduced-motion: reduce` |

---

## Production Pitfalls

**1. Importing DOM or framework APIs inside the worker**
Workers run on `WorkerGlobalScope` — `window`, `document`, React, and Angular are all undefined. Any transitive import that references these crashes the worker silently. Keep worker files pure: imports should only reach utility libraries (parsers, crypto, WASM) with no DOM dependencies. Use `eslint-plugin-no-restricted-globals` to enforce this.

**2. Forgetting to terminate the worker on unmount**
Workers keep their OS thread alive indefinitely after the component destroys. On pages that mount and unmount the same component repeatedly (route changes, tab switching), you accumulate zombie workers that consume CPU and memory with no benefit. Always pair worker creation with `worker.terminate()` in the cleanup (`useEffect` return / `ngOnDestroy`).

**3. Not using transferable objects for binary payloads**
`postMessage` serialises data via structured clone — copying a 100 MB ArrayBuffer doubles peak memory usage. Passing `Comlink.transfer(buffer, [buffer])` moves ownership in O(1) without any copy. The trade-off: after transfer the original reference is detached (`byteLength === 0`). Read the buffer's size before transferring if you need it.

**4. Calling `onProgress` on every single iteration**
Each `Comlink.proxy()` callback invocation sends a cross-thread `postMessage`. At 50,000 rows, calling `onProgress(i)` every row sends 50,000 messages and saturates the message queue. Batch to every 500–1,000 rows and yield with `await new Promise(r => setTimeout(r, 0))` periodically to allow the worker to process terminate requests.

**5. Worker errors are silent without explicit handling**
An uncaught exception inside the worker fires the worker's `onerror` event, which Comlink surfaces as a rejected Promise. If the caller does not `try/catch` around `await api.processCSV(…)`, the rejection is unhandled and the UI stays frozen on the processing state forever. Always wrap the awaited call in try/catch and set an error state.

**6. Sharing one global worker instance across concurrent operations**
A `providedIn: 'root'` service with a single worker instance serialises all calls. Two simultaneous file uploads block each other. For concurrent use cases, either create one worker per component instance (component-level provider) or build a worker pool — an array of N workers with a round-robin or least-busy dispatch strategy.

**7. Bundler misconfiguration silently falls back to main thread**
If the worker URL pattern (`new URL('./worker.ts', import.meta.url)`) is not recognised by the bundler, the import may resolve to undefined or throw at runtime. Verify with Vite's `?worker` suffix alternative or Webpack's `worker-loader`. Check that the worker chunk appears in the network tab as a separate file, not inlined into the main bundle.

---

## Interview Angle

### Q: "Why does a long-running computation freeze the browser, and how do you solve it without breaking the UI?"

The browser event loop is single-threaded: JavaScript, style recalculation, layout, and paint all share the same thread. A `for` loop over 100,000 rows occupies the thread for hundreds of milliseconds, preventing the loop from processing any other tasks — scroll events, `requestAnimationFrame` callbacks, CSS transitions. The fix is a Web Worker: a separate OS thread with its own event loop. The main thread stays free to handle all UI work while the worker computes. Communication is asynchronous via `postMessage`. Comlink wraps this in Promise-based RPC so you write `await worker.processCSV(buffer, onProgress)` instead of managing raw message envelopes.

#### Follow-up: "What are transferable objects and when do you reach for them?"

By default `postMessage` serialises data using structured clone, which copies every byte — a 100 MB ArrayBuffer becomes 200 MB peak. Transferable objects (ArrayBuffer, ImageBitmap, OffscreenCanvas, MessagePort, ReadableStream) transfer ownership between threads in O(1) time with no copy at all. You declare them in the transfer list: `postMessage(data, [arrayBuffer])`. After transfer, the ArrayBuffer on the sending side is detached — its `byteLength` becomes 0. This is the right tool whenever you are moving file data, video frames, or large typed arrays between threads. The only constraint is that you must not use the reference after transferring it.

#### Follow-up: "How do you report progress from a worker without overwhelming the message channel?"

Progress callbacks cross the thread boundary as `postMessage` calls, which have real overhead. Calling back on every row at 50,000 rows is equivalent to firing 50,000 events per second — the message queue saturates. The pattern is to batch updates: report every N rows (500–1,000 is typical), and intersperse `await new Promise(r => setTimeout(r, 0))` to yield control so terminate requests are processed. In Comlink, wrap the callback with `Comlink.proxy()` on the sending side and call it like a normal function on the worker side — Comlink handles the serialisation transparently.
