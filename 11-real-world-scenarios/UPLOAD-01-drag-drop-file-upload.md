# Drag-and-Drop File Upload with Preview

## The Idea

**In plain English:** Users drag files from their desktop into a browser zone (or click to browse), see thumbnail previews instantly before uploading, and get validation feedback if a file type or size is wrong — all before a single byte hits the server.

**Real-world analogy:** Think of a physical mail room.

- The **drop zone** is the mail slot in the door — files go in here.
- **Dragenter/dragleave** is the flap on the slot swinging open and closed — visual feedback that something is about to go through.
- The **File API** is the mail room clerk who opens each envelope, reads the label (MIME type), checks the weight (file size), and stamps it "accepted" or "rejected" before it ever reaches the destination.
- **createObjectURL** is the clerk making a photocopy and pinning it to the wall so you can see what arrived — the copy lives in the building's memory (the browser tab) until you leave (unmount), at which point it gets shredded (URL revoked).
- The **fallback `<input type="file">`** is the regular mail drop box next to the slot — for anyone who can't or won't use the fancy slot (keyboard users, assistive technology).

The key insight: previews are free (in-memory blob URLs) and disappear automatically if you revoke them. The drag events are noisy — `dragover` fires dozens of times per second — so the drop zone state lives in React/Angular state, not CSS `:hover`.

---

## Learning Objectives

- Implement drag-and-drop using the HTML5 Drag and Drop API (`dragenter`, `dragleave`, `dragover`, `drop`)
- Create instant previews using `URL.createObjectURL()` and revoke them on unmount
- Validate MIME types and file sizes before adding to the upload queue
- Build an accessible fallback `<input type="file">` that shares the same state
- Handle multiple files, duplicate detection, and remove-from-queue
- Avoid the common drag counter bug that makes drop zones flicker

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Highlight drop zone on drag hover | ⚠️ `:hover` doesn't fire for drag | Drag events don't trigger CSS `:hover` |
| Show per-file thumbnails | ❌ | Requires `createObjectURL()` — JS only |
| Validate MIME type / size | ❌ | Requires reading `file.type` and `file.size` |
| Track drag-enter counter (prevent flicker) | ❌ | Requires integer counter state |
| Remove individual files from queue | ❌ | Requires mutable array state |
| Revoke object URLs on unmount | ❌ | Requires `useEffect` cleanup / `ngOnDestroy` |
| Show upload progress per file | ❌ | Requires XHR `onprogress` events |

---

## HTML & CSS Foundation

```html
<!-- The drop zone wraps a hidden file input for accessibility -->
<div
  class="drop-zone"
  role="region"
  aria-label="File upload drop zone"
  tabindex="0"
>
  <input
    type="file"
    id="file-input"
    class="drop-zone__input"
    multiple
    accept="image/*,application/pdf"
    aria-label="Choose files to upload"
  />
  <label for="file-input" class="drop-zone__label">
    <span class="drop-zone__icon" aria-hidden="true">📁</span>
    <span>Drag files here or <strong>click to browse</strong></span>
    <span class="drop-zone__hint">PNG, JPG, PDF up to 10 MB each</span>
  </label>
</div>

<!-- Preview grid rendered below the zone -->
<ul class="preview-grid" aria-label="Selected files" role="list"></ul>
```

```css
:root {
  --drop-idle-border:   #cbd5e1;
  --drop-active-border: #3b82f6;
  --drop-active-bg:     #eff6ff;
  --drop-error-border:  #ef4444;
  --preview-size:       120px;
  --transition:         0.2s ease;
}

/* ── Drop zone ── */
.drop-zone {
  position: relative;
  border: 2px dashed var(--drop-idle-border);
  border-radius: 12px;
  padding: 2.5rem 1.5rem;
  text-align: center;
  cursor: pointer;
  transition: border-color var(--transition), background-color var(--transition);
  outline-offset: 4px;
}

.drop-zone:focus-visible {
  outline: 3px solid var(--drop-active-border);
}

/* State classes toggled by JS */
.drop-zone.is-over {
  border-color: var(--drop-active-border);
  border-style: solid;
  background-color: var(--drop-active-bg);
}

.drop-zone.has-error {
  border-color: var(--drop-error-border);
}

/* Hide the native input visually, keep it accessible */
.drop-zone__input {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
}

.drop-zone__label {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  pointer-events: none; /* let clicks pass through to the input */
  font-size: 0.95rem;
  color: #64748b;
}

.drop-zone__icon { font-size: 2.5rem; }

.drop-zone__hint {
  font-size: 0.8rem;
  color: #94a3b8;
}

/* ── Preview grid ── */
.preview-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(var(--preview-size), 1fr));
  gap: 1rem;
  margin-top: 1.5rem;
  list-style: none;
  padding: 0;
}

.preview-item {
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid #e2e8f0;
  aspect-ratio: 1;
}

.preview-item__img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.preview-item__icon {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2.5rem;
  background: #f8fafc;
}

.preview-item__remove {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: none;
  background: rgba(0,0,0,0.6);
  color: #fff;
  font-size: 0.75rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

.preview-item__name {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 0.25rem 0.375rem;
  background: rgba(0,0,0,0.5);
  color: #fff;
  font-size: 0.7rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* ── Progress bar ── */
.preview-item__progress {
  position: absolute;
  bottom: 0;
  left: 0;
  height: 3px;
  background: #3b82f6;
  transition: width 0.1s;
}

@media (prefers-reduced-motion: reduce) {
  .drop-zone,
  .preview-item__progress { transition: none; }
}
```

---

## React Implementation

```tsx
// useFileDropzone.ts
import { useState, useCallback, useRef, useEffect } from 'react';

export interface FileEntry {
  id:       string;
  file:     File;
  previewUrl: string | null;  // null for non-image files
  progress: number;           // 0–100
  error:    string | null;
}

interface DropzoneConfig {
  accept:   string[];         // e.g. ['image/jpeg', 'image/png', 'application/pdf']
  maxBytes: number;           // e.g. 10 * 1024 * 1024
  maxFiles: number;
}

export function useFileDropzone(config: DropzoneConfig) {
  const [files,     setFiles]     = useState<FileEntry[]>([]);
  const [isDragOver, setIsDragOver] = useState(false);
  const [error,     setError]     = useState<string | null>(null);

  // Drag-enter counter prevents flicker when cursor moves over child elements
  const dragCounter = useRef(0);

  // Revoke object URLs when entries are removed or component unmounts
  const revokeUrl = useCallback((entry: FileEntry) => {
    if (entry.previewUrl) URL.revokeObjectURL(entry.previewUrl);
  }, []);

  useEffect(() => {
    return () => {
      // Revoke all URLs on unmount
      setFiles(prev => {
        prev.forEach(revokeUrl);
        return [];
      });
    };
  }, [revokeUrl]);

  const validateFile = useCallback((file: File): string | null => {
    if (!config.accept.includes(file.type)) {
      return `${file.name}: unsupported file type (${file.type})`;
    }
    if (file.size > config.maxBytes) {
      const mb = (config.maxBytes / 1024 / 1024).toFixed(0);
      return `${file.name}: exceeds ${mb} MB limit`;
    }
    return null;
  }, [config]);

  const addFiles = useCallback((incoming: File[]) => {
    setError(null);

    const errors: string[] = [];
    const valid: FileEntry[] = [];

    for (const file of incoming) {
      const err = validateFile(file);
      if (err) { errors.push(err); continue; }

      const isImage = file.type.startsWith('image/');
      valid.push({
        id:         `${file.name}-${file.size}-${Date.now()}`,
        file,
        previewUrl: isImage ? URL.createObjectURL(file) : null,
        progress:   0,
        error:      null,
      });
    }

    if (errors.length) setError(errors.join(' | '));

    setFiles(prev => {
      // Deduplicate by name+size
      const existingKeys = new Set(prev.map(e => `${e.file.name}-${e.file.size}`));
      const novel = valid.filter(e => !existingKeys.has(`${e.file.name}-${e.file.size}`));

      const combined = [...prev, ...novel];
      if (combined.length > config.maxFiles) {
        // Revoke excess
        combined.slice(config.maxFiles).forEach(revokeUrl);
        return combined.slice(0, config.maxFiles);
      }
      return combined;
    });
  }, [validateFile, config.maxFiles, revokeUrl]);

  const removeFile = useCallback((id: string) => {
    setFiles(prev => {
      const entry = prev.find(e => e.id === id);
      if (entry) revokeUrl(entry);
      return prev.filter(e => e.id !== id);
    });
  }, [revokeUrl]);

  const updateProgress = useCallback((id: string, progress: number) => {
    setFiles(prev => prev.map(e => e.id === id ? { ...e, progress } : e));
  }, []);

  // Drag event handlers
  const onDragEnter = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    dragCounter.current++;
    if (dragCounter.current === 1) setIsDragOver(true);
  }, []);

  const onDragLeave = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    dragCounter.current--;
    if (dragCounter.current === 0) setIsDragOver(false);
  }, []);

  const onDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault(); // required to allow drop
    e.dataTransfer.dropEffect = 'copy';
  }, []);

  const onDrop = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    dragCounter.current = 0;
    setIsDragOver(false);
    const droppedFiles = Array.from(e.dataTransfer.files);
    addFiles(droppedFiles);
  }, [addFiles]);

  const onInputChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const selected = Array.from(e.target.files ?? []);
    addFiles(selected);
    // Reset input so the same file can be re-selected
    e.target.value = '';
  }, [addFiles]);

  return {
    files, isDragOver, error,
    removeFile, updateProgress, addFiles,
    dropZoneProps: { onDragEnter, onDragLeave, onDragOver, onDrop },
    inputProps: { onChange: onInputChange },
  };
}
```

```tsx
// FileDropzone.tsx
import { useRef } from 'react';
import { useFileDropzone, FileEntry } from './useFileDropzone';

const ACCEPTED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
const MAX_BYTES = 10 * 1024 * 1024; // 10 MB
const MAX_FILES = 10;

export function FileDropzone() {
  const {
    files, isDragOver, error,
    removeFile, dropZoneProps, inputProps,
  } = useFileDropzone({ accept: ACCEPTED_TYPES, maxBytes: MAX_BYTES, maxFiles: MAX_FILES });

  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <div>
      {/* Drop zone */}
      <div
        className={`drop-zone ${isDragOver ? 'is-over' : ''} ${error ? 'has-error' : ''}`}
        role="region"
        aria-label="File upload drop zone"
        aria-describedby={error ? 'drop-error' : undefined}
        tabIndex={0}
        onKeyDown={e => { if (e.key === 'Enter' || e.key === ' ') inputRef.current?.click(); }}
        {...dropZoneProps}
      >
        <input
          ref={inputRef}
          type="file"
          id="file-input"
          className="drop-zone__input"
          multiple
          accept={ACCEPTED_TYPES.join(',')}
          aria-label="Choose files to upload"
          {...inputProps}
        />
        <label htmlFor="file-input" className="drop-zone__label">
          <span className="drop-zone__icon" aria-hidden="true">
            {isDragOver ? '📂' : '📁'}
          </span>
          <span>
            {isDragOver ? 'Drop files here' : (
              <>Drag files here or <strong>click to browse</strong></>
            )}
          </span>
          <span className="drop-zone__hint">PNG, JPG, WebP, PDF up to 10 MB</span>
        </label>
      </div>

      {error && (
        <p id="drop-error" role="alert" style={{ color: '#ef4444', marginTop: '0.5rem', fontSize: '0.875rem' }}>
          {error}
        </p>
      )}

      {/* Preview grid */}
      {files.length > 0 && (
        <ul className="preview-grid" aria-label={`${files.length} file${files.length !== 1 ? 's' : ''} selected`}>
          {files.map(entry => (
            <FilePreviewItem
              key={entry.id}
              entry={entry}
              onRemove={() => removeFile(entry.id)}
            />
          ))}
        </ul>
      )}

      {/* Upload button */}
      {files.length > 0 && (
        <button
          type="button"
          style={{ marginTop: '1rem' }}
          onClick={() => console.log('Upload', files.map(f => f.file))}
        >
          Upload {files.length} file{files.length !== 1 ? 's' : ''}
        </button>
      )}
    </div>
  );
}

function FilePreviewItem({ entry, onRemove }: { entry: FileEntry; onRemove: () => void }) {
  const sizeLabel = entry.file.size < 1024 * 1024
    ? `${(entry.file.size / 1024).toFixed(1)} KB`
    : `${(entry.file.size / 1024 / 1024).toFixed(1)} MB`;

  return (
    <li className="preview-item" aria-label={`${entry.file.name}, ${sizeLabel}`}>
      {entry.previewUrl ? (
        <img
          className="preview-item__img"
          src={entry.previewUrl}
          alt={`Preview of ${entry.file.name}`}
          loading="lazy"
        />
      ) : (
        <div className="preview-item__icon" aria-hidden="true">📄</div>
      )}

      <button
        className="preview-item__remove"
        aria-label={`Remove ${entry.file.name}`}
        onClick={onRemove}
      >
        ✕
      </button>

      <span className="preview-item__name">{entry.file.name}</span>

      {entry.progress > 0 && entry.progress < 100 && (
        <div
          className="preview-item__progress"
          role="progressbar"
          aria-valuenow={entry.progress}
          aria-valuemin={0}
          aria-valuemax={100}
          aria-label={`Upload progress for ${entry.file.name}`}
          style={{ width: `${entry.progress}%` }}
        />
      )}
    </li>
  );
}
```

---

## Angular Implementation

```typescript
// file-dropzone.component.ts
import {
  Component, signal, computed, inject, OnDestroy,
  HostListener, ElementRef
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';

interface FileEntry {
  id:         string;
  file:       File;
  previewUrl: string | null;
  progress:   number;
  error:      string | null;
}

const ACCEPTED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
const MAX_BYTES = 10 * 1024 * 1024;
const MAX_FILES = 10;

@Component({
  selector: 'app-file-dropzone',
  standalone: true,
  imports: [NgFor, NgIf],
  template: `
    <div
      class="drop-zone"
      [class.is-over]="isDragOver()"
      [class.has-error]="!!uploadError()"
      role="region"
      aria-label="File upload drop zone"
      tabindex="0"
      (dragenter)="onDragEnter($event)"
      (dragleave)="onDragLeave($event)"
      (dragover)="onDragOver($event)"
      (drop)="onDrop($event)"
      (keydown.enter)="fileInput.click()"
      (keydown.space)="fileInput.click()"
    >
      <input
        #fileInput
        type="file"
        class="drop-zone__input"
        [accept]="acceptAttr"
        multiple
        aria-label="Choose files to upload"
        (change)="onInputChange($event)"
      />
      <label class="drop-zone__label" (click)="fileInput.click()">
        <span class="drop-zone__icon" aria-hidden="true">
          {{ isDragOver() ? '📂' : '📁' }}
        </span>
        <span *ngIf="!isDragOver()">
          Drag files here or <strong>click to browse</strong>
        </span>
        <span *ngIf="isDragOver()">Drop files here</span>
        <span class="drop-zone__hint">PNG, JPG, WebP, PDF up to 10 MB</span>
      </label>
    </div>

    <p *ngIf="uploadError()" role="alert" style="color:#ef4444;margin-top:.5rem;font-size:.875rem">
      {{ uploadError() }}
    </p>

    <ul *ngIf="files().length" class="preview-grid" [attr.aria-label]="files().length + ' files selected'">
      <li
        *ngFor="let entry of files(); trackBy: trackById"
        class="preview-item"
        [attr.aria-label]="entry.file.name"
      >
        <img *ngIf="entry.previewUrl" class="preview-item__img"
          [src]="entry.previewUrl"
          [alt]="'Preview of ' + entry.file.name"
        />
        <div *ngIf="!entry.previewUrl" class="preview-item__icon" aria-hidden="true">📄</div>

        <button
          class="preview-item__remove"
          [attr.aria-label]="'Remove ' + entry.file.name"
          (click)="removeFile(entry.id)"
        >✕</button>

        <span class="preview-item__name">{{ entry.file.name }}</span>

        <div *ngIf="entry.progress > 0 && entry.progress < 100"
          class="preview-item__progress"
          role="progressbar"
          [attr.aria-valuenow]="entry.progress"
          aria-valuemin="0" aria-valuemax="100"
          [style.width.%]="entry.progress"
        ></div>
      </li>
    </ul>

    <button *ngIf="files().length" type="button" style="margin-top:1rem" (click)="upload()">
      Upload {{ files().length }} file{{ files().length !== 1 ? 's' : '' }}
    </button>
  `,
})
export class FileDropzoneComponent implements OnDestroy {
  private dragCounter = 0;

  readonly isDragOver  = signal(false);
  readonly uploadError = signal<string | null>(null);
  readonly files       = signal<FileEntry[]>([]);

  readonly acceptAttr = ACCEPTED_TYPES.join(',');

  trackById(_: number, entry: FileEntry) { return entry.id; }

  onDragEnter(e: DragEvent) {
    e.preventDefault();
    this.dragCounter++;
    if (this.dragCounter === 1) this.isDragOver.set(true);
  }

  onDragLeave(e: DragEvent) {
    e.preventDefault();
    this.dragCounter--;
    if (this.dragCounter === 0) this.isDragOver.set(false);
  }

  onDragOver(e: DragEvent) {
    e.preventDefault();
    if (e.dataTransfer) e.dataTransfer.dropEffect = 'copy';
  }

  onDrop(e: DragEvent) {
    e.preventDefault();
    this.dragCounter = 0;
    this.isDragOver.set(false);
    const dropped = Array.from(e.dataTransfer?.files ?? []);
    this.addFiles(dropped);
  }

  onInputChange(e: Event) {
    const input = e.target as HTMLInputElement;
    this.addFiles(Array.from(input.files ?? []));
    input.value = ''; // reset so same file can be re-selected
  }

  private addFiles(incoming: File[]) {
    this.uploadError.set(null);
    const errors: string[] = [];
    const valid: FileEntry[] = [];

    for (const file of incoming) {
      const err = this.validate(file);
      if (err) { errors.push(err); continue; }
      valid.push({
        id:         `${file.name}-${file.size}-${Date.now()}`,
        file,
        previewUrl: file.type.startsWith('image/') ? URL.createObjectURL(file) : null,
        progress:   0,
        error:      null,
      });
    }

    if (errors.length) this.uploadError.set(errors.join(' | '));

    this.files.update(prev => {
      const existingKeys = new Set(prev.map(e => `${e.file.name}-${e.file.size}`));
      const novel = valid.filter(e => !existingKeys.has(`${e.file.name}-${e.file.size}`));
      const combined = [...prev, ...novel];
      if (combined.length > MAX_FILES) {
        combined.slice(MAX_FILES).forEach(e => { if (e.previewUrl) URL.revokeObjectURL(e.previewUrl); });
        return combined.slice(0, MAX_FILES);
      }
      return combined;
    });
  }

  removeFile(id: string) {
    this.files.update(prev => {
      const entry = prev.find(e => e.id === id);
      if (entry?.previewUrl) URL.revokeObjectURL(entry.previewUrl);
      return prev.filter(e => e.id !== id);
    });
  }

  private validate(file: File): string | null {
    if (!ACCEPTED_TYPES.includes(file.type)) return `${file.name}: unsupported type`;
    if (file.size > MAX_BYTES) return `${file.name}: exceeds 10 MB`;
    return null;
  }

  upload() {
    console.log('Upload', this.files().map(e => e.file));
  }

  ngOnDestroy() {
    this.files().forEach(e => { if (e.previewUrl) URL.revokeObjectURL(e.previewUrl); });
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Drop zone is keyboard accessible | `tabindex="0"`, Enter/Space triggers file input click |
| Native `<input type="file">` present | Accessible fallback; screen readers interact with it directly |
| Drop zone has `role="region"` and `aria-label` | Identifies purpose to assistive technology |
| Validation errors use `role="alert"` | Screen readers announce errors immediately |
| Each preview item has a descriptive `aria-label` | Name + size read aloud |
| Remove buttons have `aria-label` with file name | Avoids "X" announced with no context |
| Progress bars use `role="progressbar"` with `aria-valuenow/min/max` | Proper progress semantics |
| File input `accept` attribute set | Filters the native file picker OS-level |
| Drag state changes do not rely on CSS `:hover` | `dragenter`/`dragleave` events used instead |
| Reduced motion respected | Transitions removed via `@media (prefers-reduced-motion)` |

---

## Production Pitfalls

**1. Drag-leave fires when cursor moves over a child element**
`dragleave` fires when the cursor exits the drop zone *or* enters any child element. This causes the "is-over" highlight to flicker. Fix: use a counter (`dragCounter`). Increment on `dragenter`, decrement on `dragleave`. Only change visual state when the counter reaches 0.

**2. Object URLs leak memory if never revoked**
`URL.createObjectURL()` creates a URL that holds a reference to the file in memory. If you never call `URL.revokeObjectURL()`, the memory is not freed until the page is closed. Fix: revoke in `removeFile` and in `useEffect` cleanup / `ngOnDestroy`.

**3. `dragover` must call `preventDefault()` or drop is ignored**
The browser's default for `dragover` is to show a "not allowed" cursor and block the `drop` event. Fix: always call `e.preventDefault()` in `onDragOver`.

**4. MIME type validation is spoofable client-side**
`file.type` is derived from the file extension, not the actual file header. A user can rename `malware.exe` to `image.png` and `file.type` will say `image/png`. Fix: validate MIME type client-side for UX only. Always validate server-side with a magic-bytes check (e.g., Sharp for images, pdf-parse for PDFs).

**5. Large images block the main thread during `createObjectURL`**
`URL.createObjectURL()` itself is fast (it just creates a pointer), but displaying a 20 MB raw JPEG in an `<img>` will cause layout and paint jank. Fix: never render the full-resolution file directly. Use `<canvas>` to draw a downscaled thumbnail, or accept the preview delay and show a spinner.

**6. Input `value` not reset after selection**
If the user selects a file, removes it, then tries to select the same file again, the `onChange` event does not fire because the input value hasn't changed. Fix: reset `e.target.value = ''` in the `onChange` handler after processing.

---

## Interview Angle

**Q: "How would you implement a file upload drop zone with previews?"**

Strong answer covers four points:
1. **Two entry points, one code path** — a styled `div` handles drag events; a hidden `<input type="file">` handles click/keyboard. Both call the same `addFiles()` function so the file queue stays unified.
2. **The drag counter pattern** — explain why `dragenter`/`dragleave` fire on children and how a counter avoids visual flicker. This is a common interview follow-up.
3. **Object URL lifecycle** — `createObjectURL` creates a live pointer to the file in memory. It must be revoked on item removal and on component teardown. Forgetting this is a memory leak.
4. **Client-side validation is UX only** — MIME type and size checks give instant feedback but can be bypassed. The real gate is the server.

**Follow-up: "What breaks in production that a demo wouldn't show?"**
Memory leaks from un-revoked URLs, drag-leave flicker from child elements, same-file re-select not triggering `onChange`, and MIME spoofing — the pitfalls above cover each.
