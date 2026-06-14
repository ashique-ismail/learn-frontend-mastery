# Chunked / Resumable Upload with Progress

## The Idea

**In plain English:** Instead of sending a 500 MB file in a single HTTP request (which fails when Wi-Fi drops), you slice the file into small pieces and send each piece separately. If the connection drops mid-way, you ask the server "how much did you get?" and resume from that exact byte — not from the beginning.

**Real-world analogy:** Think of moving a library across town.

- **Single-request upload** is loading every book into one truck. If the truck breaks down, you unload everything and start over.
- **Chunked upload** is sending the books in multiple small vans, five at a time. Each van has a label: "Books 1–100", "Books 101–200". If van 3 breaks down, you only resend that van.
- **Resumable upload** is the warehouse clerk keeping a tally sheet. When you call to say you had a breakdown, they say "we received up to book 412 — start the next van from book 413."
- **`File.slice()`** is the librarian cutting the list into chunks before handing them to the drivers.
- **Parallel chunk requests** are multiple vans on the road at the same time — faster, but you need to make sure they arrive in order.
- **The tally sheet** is the server's offset endpoint: `GET /upload/{uploadId}/offset`.

The key insight: the browser never loads the whole file into memory at once. `File.slice()` returns a `Blob` view into the original file — lazy, memory-efficient. The only state you need is the current byte offset.

---

## Learning Objectives

- Use `File.slice()` to split a file into arbitrary byte ranges without copying the whole file
- Send chunks via `XMLHttpRequest` with `onprogress` events for per-chunk progress
- Run multiple chunk requests in parallel with `Promise.all` and limit concurrency
- Implement resume: query the server for the last received byte and skip already-sent chunks
- Calculate overall progress from chunk offsets
- Cancel mid-upload by aborting in-flight XHR requests
- Handle retry logic for failed individual chunks

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Split file into byte ranges | ❌ | `File.slice()` — JS File API |
| Track byte offset state | ❌ | Requires integer state per upload |
| Animate per-chunk progress | ⚠️ | CSS can animate a bar but cannot read XHR progress |
| Pause/resume upload | ❌ | Requires XHR `.abort()` and resumption logic |
| Retry a failed chunk | ❌ | Requires error state and retry counter |
| Show overall vs per-chunk progress | ❌ | Requires arithmetic across chunks |

---

## HTML & CSS Foundation

```html
<div class="upload-card">
  <input type="file" id="file-picker" class="upload-card__input" aria-label="Select a file to upload" />
  <label for="file-picker" class="upload-card__label">Choose large file</label>

  <div class="progress-wrapper" role="group" aria-label="Upload progress" hidden>
    <div class="progress-bar-track" aria-hidden="true">
      <div class="progress-bar-fill" id="upload-progress-bar" style="width:0%"></div>
    </div>
    <p class="progress-text" id="upload-status" aria-live="polite">0%</p>

    <div class="upload-actions">
      <button id="pause-btn" type="button">Pause</button>
      <button id="cancel-btn" type="button">Cancel</button>
    </div>
  </div>
</div>
```

```css
:root {
  --progress-track-bg:  #e2e8f0;
  --progress-fill-bg:   #3b82f6;
  --progress-height:    8px;
  --transition:         0.15s ease;
}

.upload-card {
  padding: 1.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  max-width: 480px;
}

.upload-card__input {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
}

.upload-card__label {
  display: inline-block;
  padding: 0.5rem 1.25rem;
  background: #3b82f6;
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
}

.progress-wrapper { margin-top: 1rem; }

.progress-bar-track {
  height: var(--progress-height);
  background: var(--progress-track-bg);
  border-radius: 9999px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  background: var(--progress-fill-bg);
  border-radius: 9999px;
  transition: width var(--transition);
  will-change: width;
}

.progress-text {
  margin: 0.5rem 0 0;
  font-size: 0.875rem;
  color: #64748b;
}

.upload-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 0.75rem;
}

.upload-actions button {
  padding: 0.375rem 1rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
  font-size: 0.85rem;
}

@media (prefers-reduced-motion: reduce) {
  .progress-bar-fill { transition: none; }
}
```

---

## React Implementation

```tsx
// chunkedUpload.ts — pure upload engine, no React dependency
export interface UploadOptions {
  url:          string;          // POST endpoint for each chunk
  chunkSize:    number;          // bytes per chunk, e.g. 5 * 1024 * 1024
  concurrency:  number;          // parallel chunk requests, e.g. 3
  onProgress:   (pct: number) => void;
  getUploadId?: (file: File) => Promise<string>; // create session on server
  getOffset?:   (uploadId: string) => Promise<number>; // for resume
}

export interface UploadHandle {
  start:  () => Promise<void>;
  pause:  () => void;
  resume: () => void;
  cancel: () => void;
}

export function createChunkedUpload(file: File, opts: UploadOptions): UploadHandle {
  const CHUNK = opts.chunkSize;
  const totalChunks = Math.ceil(file.size / CHUNK);

  let abortControllers: AbortController[] = [];
  let paused = false;
  let cancelled = false;
  let uploadId = '';
  let startChunk = 0;

  async function uploadChunk(index: number): Promise<void> {
    if (cancelled) throw new Error('cancelled');

    const start = index * CHUNK;
    const end   = Math.min(start + CHUNK, file.size);
    const blob  = file.slice(start, end);

    const ac = new AbortController();
    abortControllers.push(ac);

    const form = new FormData();
    form.append('file', blob, file.name);
    form.append('chunkIndex', String(index));
    form.append('totalChunks', String(totalChunks));
    form.append('uploadId', uploadId);
    form.append('byteOffset', String(start));

    const res = await fetch(`${opts.url}/chunk`, {
      method: 'POST',
      body: form,
      signal: ac.signal,
    });

    abortControllers = abortControllers.filter(c => c !== ac);

    if (!res.ok) throw new Error(`Chunk ${index} failed: ${res.status}`);
  }

  async function runWithConcurrency(indexes: number[]): Promise<void> {
    // Process chunks in batches of `opts.concurrency`
    let i = 0;
    while (i < indexes.length) {
      if (cancelled) return;

      // Wait if paused
      while (paused && !cancelled) {
        await new Promise<void>(r => setTimeout(r, 200));
      }
      if (cancelled) return;

      const batch = indexes.slice(i, i + opts.concurrency);
      await Promise.all(batch.map(idx => uploadChunk(idx)));

      i += opts.concurrency;
      const completedChunks = Math.min(startChunk + i, totalChunks);
      opts.onProgress(Math.round((completedChunks / totalChunks) * 100));
    }
  }

  return {
    async start() {
      // Create upload session
      if (opts.getUploadId) {
        uploadId = await opts.getUploadId(file);
      } else {
        uploadId = `upload-${Date.now()}`;
      }

      // Resume: find last confirmed byte
      if (opts.getOffset) {
        const offset = await opts.getOffset(uploadId);
        startChunk = Math.floor(offset / CHUNK);
        opts.onProgress(Math.round((startChunk / totalChunks) * 100));
      }

      const remaining = Array.from({ length: totalChunks - startChunk }, (_, i) => startChunk + i);
      await runWithConcurrency(remaining);

      if (!cancelled) {
        // Signal server to assemble chunks
        await fetch(`${opts.url}/complete`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ uploadId, totalChunks, filename: file.name }),
        });
        opts.onProgress(100);
      }
    },

    pause()  { paused = true; },
    resume() { paused = false; },

    cancel() {
      cancelled = true;
      paused    = false;
      abortControllers.forEach(c => c.abort());
      abortControllers = [];
    },
  };
}
```

```tsx
// useChunkedUpload.ts
import { useState, useRef, useCallback } from 'react';
import { createChunkedUpload, UploadHandle } from './chunkedUpload';

type UploadStatus = 'idle' | 'uploading' | 'paused' | 'complete' | 'error';

export function useChunkedUpload(apiBase: string) {
  const [file,     setFile]     = useState<File | null>(null);
  const [progress, setProgress] = useState(0);
  const [status,   setStatus]   = useState<UploadStatus>('idle');
  const [error,    setError]    = useState<string | null>(null);

  const handleRef = useRef<UploadHandle | null>(null);

  const selectFile = useCallback((f: File) => {
    setFile(f);
    setProgress(0);
    setStatus('idle');
    setError(null);
  }, []);

  const start = useCallback(async () => {
    if (!file) return;
    setStatus('uploading');
    setError(null);

    const handle = createChunkedUpload(file, {
      url:         apiBase,
      chunkSize:   5 * 1024 * 1024, // 5 MB chunks
      concurrency: 3,
      onProgress:  pct => setProgress(pct),
    });

    handleRef.current = handle;

    try {
      await handle.start();
      setStatus('complete');
    } catch (err) {
      if ((err as Error).message !== 'cancelled') {
        setStatus('error');
        setError((err as Error).message);
      }
    }
  }, [file, apiBase]);

  const pause = useCallback(() => {
    handleRef.current?.pause();
    setStatus('paused');
  }, []);

  const resume = useCallback(() => {
    handleRef.current?.resume();
    setStatus('uploading');
  }, []);

  const cancel = useCallback(() => {
    handleRef.current?.cancel();
    setStatus('idle');
    setProgress(0);
  }, []);

  return { file, progress, status, error, selectFile, start, pause, resume, cancel };
}
```

```tsx
// ChunkedUploader.tsx
import { useRef } from 'react';
import { useChunkedUpload } from './useChunkedUpload';

export function ChunkedUploader() {
  const { file, progress, status, error, selectFile, start, pause, resume, cancel } =
    useChunkedUpload('/api/upload');

  const inputRef = useRef<HTMLInputElement>(null);

  const formatBytes = (bytes: number) =>
    bytes < 1024 * 1024
      ? `${(bytes / 1024).toFixed(1)} KB`
      : `${(bytes / 1024 / 1024).toFixed(1)} MB`;

  return (
    <div className="upload-card">
      <input
        ref={inputRef}
        type="file"
        id="file-picker"
        className="upload-card__input"
        aria-label="Select a file to upload"
        onChange={e => {
          const f = e.target.files?.[0];
          if (f) selectFile(f);
        }}
      />

      {status === 'idle' && (
        <label htmlFor="file-picker" className="upload-card__label">
          Choose file
        </label>
      )}

      {file && (
        <p style={{ margin: '0.75rem 0 0', fontSize: '0.875rem' }}>
          {file.name} ({formatBytes(file.size)})
        </p>
      )}

      {file && status === 'idle' && (
        <button type="button" onClick={start} style={{ marginTop: '0.75rem' }}>
          Start Upload
        </button>
      )}

      {(status === 'uploading' || status === 'paused' || status === 'complete') && (
        <div className="progress-wrapper" role="group" aria-label="Upload progress">
          <div className="progress-bar-track" aria-hidden="true">
            <div className="progress-bar-fill" style={{ width: `${progress}%` }} />
          </div>
          <p
            className="progress-text"
            id="upload-status"
            aria-live="polite"
            aria-atomic="true"
          >
            {status === 'complete' ? 'Upload complete!' : `${progress}% uploaded`}
          </p>

          {status !== 'complete' && (
            <div className="upload-actions">
              {status === 'uploading' ? (
                <button type="button" onClick={pause}>Pause</button>
              ) : (
                <button type="button" onClick={resume}>Resume</button>
              )}
              <button type="button" onClick={cancel}>Cancel</button>
            </div>
          )}
        </div>
      )}

      {error && (
        <p role="alert" style={{ color: '#ef4444', marginTop: '0.5rem', fontSize: '0.875rem' }}>
          {error}
        </p>
      )}
    </div>
  );
}
```

---

## Angular Implementation

```typescript
// chunked-uploader.component.ts
import { Component, signal, inject } from '@angular/core';
import { NgIf } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

type UploadStatus = 'idle' | 'uploading' | 'paused' | 'complete' | 'error';

interface ChunkState {
  totalChunks: number;
  completedChunks: number;
  uploadId: string;
}

@Component({
  selector: 'app-chunked-uploader',
  standalone: true,
  imports: [NgIf],
  template: `
    <div class="upload-card">
      <input
        #fileInput
        type="file"
        id="file-picker"
        class="upload-card__input"
        aria-label="Select a file to upload"
        (change)="onFileSelect($event)"
      />

      <label *ngIf="status() === 'idle'" for="file-picker" class="upload-card__label">
        Choose file
      </label>

      <p *ngIf="selectedFile()" style="margin:.75rem 0 0;font-size:.875rem">
        {{ selectedFile()!.name }} ({{ formatBytes(selectedFile()!.size) }})
      </p>

      <button *ngIf="selectedFile() && status() === 'idle'" type="button"
        style="margin-top:.75rem" (click)="startUpload()">
        Start Upload
      </button>

      <div *ngIf="['uploading','paused','complete'].includes(status())"
        class="progress-wrapper" role="group" aria-label="Upload progress">
        <div class="progress-bar-track" aria-hidden="true">
          <div class="progress-bar-fill" [style.width.%]="progress()"></div>
        </div>
        <p class="progress-text" aria-live="polite" aria-atomic="true">
          {{ status() === 'complete' ? 'Upload complete!' : progress() + '% uploaded' }}
        </p>
        <div *ngIf="status() !== 'complete'" class="upload-actions">
          <button *ngIf="status() === 'uploading'" type="button" (click)="pause()">Pause</button>
          <button *ngIf="status() === 'paused'"    type="button" (click)="resume()">Resume</button>
          <button type="button" (click)="cancel()">Cancel</button>
        </div>
      </div>

      <p *ngIf="errorMsg()" role="alert" style="color:#ef4444;margin-top:.5rem;font-size:.875rem">
        {{ errorMsg() }}
      </p>
    </div>
  `,
})
export class ChunkedUploaderComponent {
  private http = inject(HttpClient);

  readonly selectedFile = signal<File | null>(null);
  readonly progress     = signal(0);
  readonly status       = signal<UploadStatus>('idle');
  readonly errorMsg     = signal<string | null>(null);

  private paused    = false;
  private cancelled = false;
  private abortControllers: AbortController[] = [];

  private readonly CHUNK_SIZE   = 5 * 1024 * 1024;
  private readonly CONCURRENCY  = 3;
  private readonly API_BASE     = '/api/upload';

  onFileSelect(e: Event) {
    const input = e.target as HTMLInputElement;
    const file  = input.files?.[0];
    if (file) {
      this.selectedFile.set(file);
      this.progress.set(0);
      this.status.set('idle');
      this.errorMsg.set(null);
    }
  }

  async startUpload() {
    const file = this.selectedFile();
    if (!file) return;

    this.status.set('uploading');
    this.paused    = false;
    this.cancelled = false;

    const totalChunks = Math.ceil(file.size / this.CHUNK_SIZE);
    const uploadId = `upload-${Date.now()}`;

    try {
      const indexes = Array.from({ length: totalChunks }, (_, i) => i);
      await this.runWithConcurrency(file, indexes, totalChunks, uploadId);

      if (!this.cancelled) {
        await firstValueFrom(this.http.post(`${this.API_BASE}/complete`, {
          uploadId, totalChunks, filename: file.name
        }));
        this.progress.set(100);
        this.status.set('complete');
      }
    } catch (err) {
      if (!this.cancelled) {
        this.status.set('error');
        this.errorMsg.set((err as Error).message);
      }
    }
  }

  private async runWithConcurrency(
    file: File, indexes: number[], totalChunks: number, uploadId: string
  ) {
    let i = 0;
    while (i < indexes.length) {
      if (this.cancelled) return;
      while (this.paused && !this.cancelled) {
        await new Promise<void>(r => setTimeout(r, 200));
      }
      if (this.cancelled) return;

      const batch = indexes.slice(i, i + this.CONCURRENCY);
      await Promise.all(batch.map(idx => this.uploadChunk(file, idx, totalChunks, uploadId)));
      i += this.CONCURRENCY;
      this.progress.set(Math.round((Math.min(i, totalChunks) / totalChunks) * 100));
    }
  }

  private async uploadChunk(file: File, index: number, totalChunks: number, uploadId: string) {
    const start = index * this.CHUNK_SIZE;
    const end   = Math.min(start + this.CHUNK_SIZE, file.size);
    const blob  = file.slice(start, end);
    const ac    = new AbortController();
    this.abortControllers.push(ac);

    const form = new FormData();
    form.append('file', blob, file.name);
    form.append('chunkIndex', String(index));
    form.append('totalChunks', String(totalChunks));
    form.append('uploadId', uploadId);
    form.append('byteOffset', String(start));

    const res = await fetch(`${this.API_BASE}/chunk`, { method: 'POST', body: form, signal: ac.signal });
    this.abortControllers = this.abortControllers.filter(c => c !== ac);
    if (!res.ok) throw new Error(`Chunk ${index} failed: ${res.status}`);
  }

  pause()  { this.paused = true;  this.status.set('paused'); }
  resume() { this.paused = false; this.status.set('uploading'); }

  cancel() {
    this.cancelled = true;
    this.paused    = false;
    this.abortControllers.forEach(c => c.abort());
    this.abortControllers = [];
    this.status.set('idle');
    this.progress.set(0);
  }

  formatBytes(bytes: number): string {
    return bytes < 1024 * 1024
      ? `${(bytes / 1024).toFixed(1)} KB`
      : `${(bytes / 1024 / 1024).toFixed(1)} MB`;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| File input is visually hidden, not `display:none` | Clip-rect hiding keeps it accessible to screen readers and keyboard |
| Label is associated with file input via `for`/`id` | Standard label-input pairing |
| Progress bar uses `role="progressbar"` with `aria-valuenow` | Proper semantics for AT |
| Status text uses `aria-live="polite"` + `aria-atomic` | Announces percentage changes without interrupting other AT speech |
| Error messages use `role="alert"` | Immediately announced on appearance |
| Pause/Resume buttons have clear text labels | "Pause" and "Resume" — no icon-only buttons |
| Cancel is always available during upload | Never disabled; allows user to abort at any time |

---

## Production Pitfalls

**1. Concurrency without back-pressure overwhelms the server**
Sending 20 chunks in parallel can saturate a connection or trigger rate limiting. Fix: limit concurrency (3–5 is typical). The `runWithConcurrency` helper batches requests and waits for each batch to complete before starting the next.

**2. The server must be idempotent for chunk re-uploads**
A retried chunk may arrive twice. Fix: servers should use the `chunkIndex` + `uploadId` as a unique key and silently accept duplicate chunks (idempotent write).

**3. File.slice() is a view, not a copy**
The original `File` object must remain in memory for the entire upload. If the user closes the file picker or the `File` reference is garbage collected, the slices become invalid. Fix: hold the `File` reference in component state (not a local variable) for the full upload lifetime.

**4. Resuming requires server coordination**
The client cannot know how many bytes the server actually wrote to disk. Fix: `GET /upload/{id}/offset` must return the byte count confirmed written and fsync'd — not just bytes received in a buffer.

**5. Progress jumps rather than flowing smoothly**
Batch-based concurrency reports progress only at batch boundaries (every N chunks). Fix: track progress per-chunk using `XMLHttpRequest.onprogress` and accumulate loaded bytes across all in-flight requests for a smooth bar.

**6. `AbortController` signals are not reusable**
Once `abort()` is called, the same `AbortController` instance cannot be reused. Fix: create a fresh `AbortController` per chunk request and remove it from the tracking array immediately after it resolves or rejects.

---

## Interview Angle

**Q: "How would you implement a resumable upload for large files?"**

Strong answer covers four points:
1. **`File.slice()` creates zero-copy views** — the browser does not copy file data into memory for each chunk. Each `slice()` is a lazy reference to the original file bytes.
2. **Server coordination is required for true resume** — the client must ask the server for the confirmed byte offset (`GET /upload/{id}/offset`) before skipping already-sent chunks. Without this, you might re-send chunks the server already has.
3. **Concurrency with back-pressure** — explain the batch pattern: process N chunks in parallel, wait for the batch, then start the next batch. This prevents flooding the server while still parallelising.
4. **Pause/cancel via AbortController** — each chunk request gets its own `AbortController`. Pause works by checking a flag before starting each batch; cancel aborts all in-flight requests immediately.

**Follow-up: "What if a chunk upload fails halfway through?"**
Implement per-chunk retry with exponential backoff (up to 3 attempts). Only throw if all retries are exhausted. Track which chunks failed so you can retry just those, not the entire file.
