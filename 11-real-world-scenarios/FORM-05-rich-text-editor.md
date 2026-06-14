# Rich Text Editor Integration (Tiptap / Quill)

## The Idea

**In plain English:** A rich text editor (RTE) is a `<div contenteditable="true">` that a library has wired up with its own internal state machine. The library owns the document model — a tree of nodes representing paragraphs, bold marks, links, images — and you access or mutate that model through an imperative API (`editor.getHTML()`, `editor.commands.toggleBold()`). You do not bind to it the same way you bind to an `<input>`. React and Angular's controlled-input pattern breaks down here because the editor controls its own DOM, and fighting that causes cursor-jumping and lost selections.

**Real-world analogy:** Think of a contract at a law firm.

- The **editable contract** sitting on a lawyer's desk is the `contenteditable` div. It is a physical object — you can touch it, annotate it, flip its pages.
- The **law firm's document management system** (file number, version history, tracked changes) is the editor's internal model. You do not rewrite the contract by changing what is in the document management system; the system and the physical paper stay in sync automatically.
- When the **client needs the final text** (to send via email), the lawyer calls the assistant and says "give me an HTML export of file #42." That is `editor.getHTML()` — an imperative read, not a live binding.
- The **toolbar buttons** (Bold, Italic, Link) are not directly modifying the paper. They fire commands at the document management system, which then reflowspages. That is `editor.commands.toggleBold()`.
- The **client asking "is this section bold right now?"** is your toolbar UI reading `editor.isActive('bold')` on every selection change event.

The key insight: you own the container element. The editor library owns everything inside it. Your job is to set up the bridge — initialise the editor, read from it imperatively on submit, and sync toolbar state by subscribing to selection-change events, not by diffing the DOM.

---

## Learning Objectives

- Understand why the controlled-component pattern fails for `contenteditable` and what to use instead
- Initialise Tiptap in React with `useEditor` and access content imperatively via `editor.getHTML()`
- Build a toolbar that reflects editor state using the `onUpdate` / `onSelectionUpdate` callbacks
- Initialise Quill in Angular using `AfterViewInit` and an imperative ref, never `ngModel`
- Wire an Angular service to bridge editor events to the rest of the form
- Handle the three hardest edge cases: paste, cursor restoration after re-render, and SSR
- Apply the correct ARIA pattern for a toolbar + editing region
- Recognise the five most common production bugs and know how to fix each one

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Display text as bold, italic, underline | Yes, via inline styles the editor injects | — |
| Show a floating toolbar at the current selection | No | Requires JS to read `window.getSelection()` for position |
| Keep toolbar button state in sync with cursor position | No | CSS has no concept of "is the cursor inside a bold span?" |
| Serialize editor content to HTML or JSON | No | Requires an imperative method call on the editor instance |
| Paste plain text and strip unsafe HTML | No | Requires a `paste` event handler and DOMParser |
| Undo/redo history | No | Requires a command stack maintained in JS |
| Mention suggestions (@user) as the user types | No | Requires keydown interception and async data fetch |
| Prevent form submission with empty editor | No | `required` does not work on `contenteditable` |

---

## HTML & CSS Foundation

### The Minimal Structure

```html
<!--
  The editor mounts inside #editor-container.
  The library replaces the inner div with contenteditable.
  The toolbar is a separate landmark, linked by aria-controls.
-->
<div class="rte-wrapper">

  <!-- Toolbar: role="toolbar" with aria-label -->
  <div
    role="toolbar"
    aria-label="Text formatting"
    aria-controls="editor-region"
    class="rte-toolbar"
  >
    <button type="button" data-command="bold"   aria-pressed="false" aria-label="Bold">   <strong>B</strong></button>
    <button type="button" data-command="italic" aria-pressed="false" aria-label="Italic"> <em>I</em></button>
    <button type="button" data-command="strike" aria-pressed="false" aria-label="Strikethrough"><s>S</s></button>

    <span role="separator" aria-orientation="vertical" class="rte-divider"></span>

    <button type="button" data-command="bulletList" aria-pressed="false" aria-label="Bullet list">&#8226;&#8226;</button>
    <button type="button" data-command="orderedList" aria-pressed="false" aria-label="Numbered list">1.</button>
  </div>

  <!-- Editing region: receives aria-multiline, role="textbox" from the library or manually -->
  <div
    id="editor-region"
    class="rte-content"
    aria-multiline="true"
    aria-label="Document body"
  >
    <!-- contenteditable div is injected here by the library -->
  </div>

  <!-- Hidden input carries the value for the surrounding form -->
  <input type="hidden" name="body" id="body-hidden-input" />

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --rte-border:         1px solid #d0d0d0;
  --rte-radius:         6px;
  --rte-toolbar-bg:     #f5f5f5;
  --rte-content-min-h:  180px;
  --rte-focus-ring:     0 0 0 3px rgba(66, 153, 225, 0.5);
  --rte-btn-active-bg:  #d1e9ff;
  --rte-btn-active-clr: #0050b3;
}

/* ─── Wrapper ─── */
.rte-wrapper {
  border: var(--rte-border);
  border-radius: var(--rte-radius);
  overflow: hidden;           /* clips toolbar and content to the same rounded border */
  display: flex;
  flex-direction: column;
}

/* ─── Toolbar ─── */
.rte-toolbar {
  display: flex;
  align-items: center;
  gap: 2px;
  padding: 6px 8px;
  background: var(--rte-toolbar-bg);
  border-bottom: var(--rte-border);
  flex-wrap: wrap;
}

.rte-toolbar button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 32px;
  min-height: 32px;
  padding: 0 6px;
  border: 1px solid transparent;
  border-radius: 4px;
  background: none;
  cursor: pointer;
  font-size: 0.875rem;
  color: #333;
  transition: background 0.15s, color 0.15s;
}

.rte-toolbar button:hover {
  background: #e8e8e8;
}

/* Active state is toggled by JS setting aria-pressed="true" */
.rte-toolbar button[aria-pressed="true"] {
  background: var(--rte-btn-active-bg);
  color:      var(--rte-btn-active-clr);
  border-color: #90c5ff;
}

.rte-toolbar button:focus-visible {
  outline: none;
  box-shadow: var(--rte-focus-ring);
}

.rte-divider {
  display: inline-block;
  width: 1px;
  height: 20px;
  background: #ccc;
  margin: 0 4px;
}

/* ─── Content region ─── */
.rte-content {
  min-height: var(--rte-content-min-h);
  padding: 12px 16px;
  font-size: 1rem;
  line-height: 1.6;
  outline: none;             /* suppress browser default; handle focus ring below */
}

/* Focus ring on the content div when the editor is focused */
.rte-content:focus-within {
  box-shadow: inset var(--rte-focus-ring);
}

/* Tiptap / ProseMirror resets */
.rte-content .ProseMirror {
  outline: none;
  min-height: var(--rte-content-min-h);
}

/* Placeholder text */
.rte-content .ProseMirror p.is-editor-empty:first-child::before {
  content: attr(data-placeholder);
  color: #aaa;
  pointer-events: none;
  float: left;
  height: 0;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .rte-toolbar button { transition: none; }
}
```

**What CSS owns:** toolbar layout, active-state styling (driven by `aria-pressed`), focus rings, placeholder text via `::before`.

**What CSS cannot own:** which toolbar buttons are active (depends on cursor position), serializing content to HTML, validating that the editor is not empty.

---

## React Implementation

### Types

```tsx
// rte.types.ts
export interface RteProps {
  initialContent?: string;   // HTML string to pre-populate
  placeholder?:    string;
  onChange?:       (html: string) => void;   // optional live callback
}

export interface RteRef {
  getHTML:  () => string;
  setHTML:  (html: string) => void;
  clear:    () => void;
  focus:    () => void;
}
```

### The Editor Hook

```tsx
// useRichTextEditor.ts
import { useEditor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Placeholder from '@tiptap/extension-placeholder';
import { useCallback } from 'react';

interface Options {
  initialContent: string;
  placeholder:    string;
  onChange?:      (html: string) => void;
}

export function useRichTextEditor({ initialContent, placeholder, onChange }: Options) {
  const editor = useEditor({
    extensions: [
      StarterKit,
      Placeholder.configure({ placeholder }),
    ],
    content: initialContent,

    /*
     * onUpdate fires after every keystroke.
     * Do NOT call editor.getHTML() on every render — only here.
     * If no onChange is provided, we skip serialization entirely
     * because the consumer will call getHTML() imperatively on submit.
     */
    onUpdate({ editor: e }) {
      onChange?.(e.getHTML());
    },
  });

  // Stable imperative handles — safe to put in a ref
  const getHTML  = useCallback(() => editor?.getHTML() ?? '',      [editor]);
  const setHTML  = useCallback((html: string) => {
    editor?.commands.setContent(html, /* emitUpdate */ false);
  }, [editor]);
  const clear    = useCallback(() => {
    editor?.commands.clearContent(/* emitUpdate */ false);
  }, [editor]);
  const focus    = useCallback(() => editor?.commands.focus(),      [editor]);

  return { editor, getHTML, setHTML, clear, focus };
}
```

### The Toolbar Component

```tsx
// RteToolbar.tsx
import type { Editor } from '@tiptap/react';

interface ToolbarProps {
  editor: Editor | null;
}

type FormatCommand =
  | 'toggleBold'
  | 'toggleItalic'
  | 'toggleStrike'
  | 'toggleBulletList'
  | 'toggleOrderedList';

interface ToolbarButton {
  command:    FormatCommand;
  isActive:   (e: Editor) => boolean;
  label:      string;
  display:    React.ReactNode;
}

const BUTTONS: ToolbarButton[] = [
  { command: 'toggleBold',        isActive: e => e.isActive('bold'),        label: 'Bold',           display: <strong>B</strong> },
  { command: 'toggleItalic',      isActive: e => e.isActive('italic'),      label: 'Italic',         display: <em>I</em>         },
  { command: 'toggleStrike',      isActive: e => e.isActive('strike'),      label: 'Strikethrough',  display: <s>S</s>           },
  { command: 'toggleBulletList',  isActive: e => e.isActive('bulletList'),  label: 'Bullet list',    display: <span>&#8226;&#8226;</span> },
  { command: 'toggleOrderedList', isActive: e => e.isActive('orderedList'), label: 'Numbered list',  display: <span>1.</span>    },
];

export function RteToolbar({ editor }: ToolbarProps) {
  if (!editor) return null;

  return (
    <div role="toolbar" aria-label="Text formatting" aria-controls="rte-region" className="rte-toolbar">
      {BUTTONS.map(({ command, isActive, label, display }, i) => (
        <button
          key={command}
          type="button"
          aria-pressed={isActive(editor)}
          aria-label={label}
          onMouseDown={e => {
            /*
             * Use onMouseDown + preventDefault instead of onClick.
             * onClick fires after onBlur, which moves focus away from
             * the editor. preventDefault keeps the editor focused so
             * the cursor position is preserved when the command runs.
             */
            e.preventDefault();
            editor.chain().focus()[command]().run();
          }}
        >
          {display}
        </button>
      ))}
    </div>
  );
}
```

### The Full RTE Component with Imperative Ref

```tsx
// RichTextEditor.tsx
import { forwardRef, useImperativeHandle } from 'react';
import { EditorContent } from '@tiptap/react';
import { useRichTextEditor } from './useRichTextEditor';
import { RteToolbar } from './RteToolbar';
import type { RteProps, RteRef } from './rte.types';

/*
 * forwardRef + useImperativeHandle is the standard pattern for
 * uncontrolled components that need to expose an API.
 * The parent calls editorRef.current.getHTML() on form submit —
 * it does NOT maintain any state for the editor content.
 */
export const RichTextEditor = forwardRef<RteRef, RteProps>(function RichTextEditor(
  { initialContent = '', placeholder = 'Start writing…', onChange },
  ref,
) {
  const { editor, getHTML, setHTML, clear, focus } = useRichTextEditor({
    initialContent,
    placeholder,
    onChange,
  });

  // Expose the imperative API to the parent via ref
  useImperativeHandle(ref, () => ({ getHTML, setHTML, clear, focus }), [
    getHTML, setHTML, clear, focus,
  ]);

  return (
    <div className="rte-wrapper">
      <RteToolbar editor={editor} />
      <div className="rte-content" id="rte-region">
        <EditorContent editor={editor} aria-multiline="true" aria-label="Document body" />
      </div>
    </div>
  );
});
```

### Consuming the RTE in a Form

```tsx
// DocumentForm.tsx
import { useRef } from 'react';
import { RichTextEditor } from './RichTextEditor';
import type { RteRef } from './rte.types';

export function DocumentForm() {
  const editorRef = useRef<RteRef>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    // Imperative read — no state involved
    const html = editorRef.current?.getHTML() ?? '';

    // Validate empty content: Tiptap returns '<p></p>' for an empty editor
    if (!html || html === '<p></p>') {
      alert('Document body is required.');
      editorRef.current?.focus();
      return;
    }

    fetch('/api/documents', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ body: html }),
    });
  };

  return (
    <form onSubmit={handleSubmit} noValidate>
      <label htmlFor="rte-region">Document Body</label>
      <RichTextEditor
        ref={editorRef}
        placeholder="Start writing your document…"
        initialContent="<p>Hello world</p>"
      />
      <button type="submit">Save</button>
    </form>
  );
}
```

---

## Angular Implementation

### Types and Service

```typescript
// rich-text-editor.model.ts
export interface RteState {
  isBold:        boolean;
  isItalic:      boolean;
  isStrike:      boolean;
  isBulletList:  boolean;
  isOrderedList: boolean;
  isEmpty:       boolean;
}

export const RTE_STATE_INIT: RteState = {
  isBold: false, isItalic: false, isStrike: false,
  isBulletList: false, isOrderedList: false, isEmpty: true,
};
```

```typescript
// rich-text-editor.service.ts
import { Injectable, signal, computed } from '@angular/core';
import Quill, { type Range } from 'quill';
import type { RteState } from './rich-text-editor.model';
import { RTE_STATE_INIT } from './rich-text-editor.model';

/*
 * The service holds the Quill instance and exposes:
 *   - signals for toolbar state (read by the toolbar component)
 *   - imperative methods (called by the toolbar and the form)
 *
 * The component that owns the DOM container calls init() from AfterViewInit.
 */
@Injectable()   // providedIn component — not a singleton, each editor gets its own instance
export class RichTextEditorService {
  private quill: Quill | null = null;

  private _state = signal<RteState>(RTE_STATE_INIT);
  readonly state = this._state.asReadonly();

  readonly isBold        = computed(() => this._state().isBold);
  readonly isItalic      = computed(() => this._state().isItalic);
  readonly isStrike      = computed(() => this._state().isStrike);
  readonly isBulletList  = computed(() => this._state().isBulletList);
  readonly isOrderedList = computed(() => this._state().isOrderedList);
  readonly isEmpty       = computed(() => this._state().isEmpty);

  init(container: HTMLElement, initialContent = '') {
    this.quill = new Quill(container, {
      theme: 'bubble',     // headless — we provide our own toolbar CSS
      modules: {
        toolbar: false,    // disable Quill's built-in toolbar; we render our own
      },
    });

    if (initialContent) {
      // setContents expects a Delta; fromHTML converts an HTML string
      const delta = this.quill.clipboard.convert({ html: initialContent });
      this.quill.setContents(delta, 'silent');
    }

    /*
     * selection-change fires on cursor move.
     * text-change fires on any keystroke.
     * Both need to refresh toolbar state.
     */
    this.quill.on('selection-change', () => this.syncState());
    this.quill.on('text-change',      () => this.syncState());
  }

  private syncState() {
    if (!this.quill) return;
    const format = this.quill.getFormat();
    const text   = this.quill.getText().trim();
    this._state.set({
      isBold:        !!format['bold'],
      isItalic:      !!format['italic'],
      isStrike:      !!format['strike'],
      isBulletList:  format['list'] === 'bullet',
      isOrderedList: format['list'] === 'ordered',
      isEmpty:       text.length === 0,
    });
  }

  // ── Imperative commands ──

  format(name: string, value: unknown) {
    this.quill?.format(name, value, 'user');
  }

  toggleBold()        { this.format('bold',   !this._state().isBold);        }
  toggleItalic()      { this.format('italic', !this._state().isItalic);      }
  toggleStrike()      { this.format('strike', !this._state().isStrike);      }
  toggleBulletList()  {
    this.format('list', this._state().isBulletList  ? false : 'bullet');
  }
  toggleOrderedList() {
    this.format('list', this._state().isOrderedList ? false : 'ordered');
  }

  // ── Read methods for form submission ──

  getHTML(): string {
    if (!this.quill) return '';
    const delta = this.quill.getContents();
    // quill.root.innerHTML is the fastest way to get the current HTML
    return this.quill.root.innerHTML;
  }

  setHTML(html: string) {
    if (!this.quill) return;
    const delta = this.quill.clipboard.convert({ html });
    this.quill.setContents(delta, 'silent');
  }

  clear() {
    this.quill?.setText('', 'silent');
  }

  focus() {
    this.quill?.focus();
  }

  destroy() {
    this.quill = null;
  }
}
```

### The Toolbar Component

```typescript
// rte-toolbar.component.ts
import { Component, inject } from '@angular/core';
import { RichTextEditorService } from './rich-text-editor.service';

@Component({
  selector: 'app-rte-toolbar',
  standalone: true,
  template: `
    <div role="toolbar" aria-label="Text formatting" aria-controls="rte-region" class="rte-toolbar">

      <button type="button"
        [attr.aria-pressed]="svc.isBold()"
        aria-label="Bold"
        (mousedown)="prevent($event); svc.toggleBold()"
      ><strong>B</strong></button>

      <button type="button"
        [attr.aria-pressed]="svc.isItalic()"
        aria-label="Italic"
        (mousedown)="prevent($event); svc.toggleItalic()"
      ><em>I</em></button>

      <button type="button"
        [attr.aria-pressed]="svc.isStrike()"
        aria-label="Strikethrough"
        (mousedown)="prevent($event); svc.toggleStrike()"
      ><s>S</s></button>

      <span role="separator" aria-orientation="vertical" class="rte-divider"></span>

      <button type="button"
        [attr.aria-pressed]="svc.isBulletList()"
        aria-label="Bullet list"
        (mousedown)="prevent($event); svc.toggleBulletList()"
      >&#8226;&#8226;</button>

      <button type="button"
        [attr.aria-pressed]="svc.isOrderedList()"
        aria-label="Numbered list"
        (mousedown)="prevent($event); svc.toggleOrderedList()"
      >1.</button>

    </div>
  `,
})
export class RteToolbarComponent {
  svc = inject(RichTextEditorService);

  prevent(e: MouseEvent) {
    // Same trick as the React version — prevents blur before the command fires
    e.preventDefault();
  }
}
```

### The Editor Component

```typescript
// rich-text-editor.component.ts
import {
  Component, AfterViewInit, OnDestroy,
  ElementRef, ViewChild, Input, Output, EventEmitter,
  inject
} from '@angular/core';
import { RichTextEditorService } from './rich-text-editor.service';
import { RteToolbarComponent } from './rte-toolbar.component';

@Component({
  selector: 'app-rich-text-editor',
  standalone: true,
  imports: [RteToolbarComponent],
  providers: [RichTextEditorService],   // one service instance per editor instance
  template: `
    <div class="rte-wrapper">
      <app-rte-toolbar />
      <div class="rte-content" id="rte-region" #editorContainer
           aria-multiline="true" aria-label="Document body">
      </div>
    </div>
  `,
})
export class RichTextEditorComponent implements AfterViewInit, OnDestroy {
  @Input() initialContent = '';
  @Input() placeholder    = 'Start writing…';
  @Output() contentChange = new EventEmitter<string>();

  @ViewChild('editorContainer') private containerRef!: ElementRef<HTMLDivElement>;

  svc = inject(RichTextEditorService);

  ngAfterViewInit() {
    // NEVER initialise Quill/Tiptap in the constructor or ngOnInit —
    // the DOM element does not exist until AfterViewInit.
    this.svc.init(this.containerRef.nativeElement, this.initialContent);
  }

  // Public API for the parent form to call via ViewChild
  getHTML()            { return this.svc.getHTML(); }
  setHTML(html: string){ this.svc.setHTML(html); }
  clear()              { this.svc.clear(); }
  focus()              { this.svc.focus(); }

  ngOnDestroy() {
    this.svc.destroy();
  }
}
```

### Consuming the Editor in a Form

```typescript
// document-form.component.ts
import { Component, ViewChild } from '@angular/core';
import { RichTextEditorComponent } from './rich-text-editor.component';

@Component({
  selector: 'app-document-form',
  standalone: true,
  imports: [RichTextEditorComponent],
  template: `
    <form (ngSubmit)="submit()" novalidate>
      <label for="rte-region">Document Body</label>

      <app-rich-text-editor
        #rte
        initialContent="<p>Hello world</p>"
        placeholder="Start writing your document…"
      />

      <p *ngIf="validationError" role="alert" class="error">{{ validationError }}</p>

      <button type="submit">Save</button>
    </form>
  `,
})
export class DocumentFormComponent {
  @ViewChild('rte') rte!: RichTextEditorComponent;
  validationError = '';

  submit() {
    const html = this.rte.getHTML();

    // Quill returns '<p><br></p>' for an empty editor
    const isEmpty = !html || html === '<p><br></p>';
    if (isEmpty) {
      this.validationError = 'Document body is required.';
      this.rte.focus();
      return;
    }

    this.validationError = '';
    // POST html to your API
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Toolbar has `role="toolbar"` and `aria-label` | Declared on the toolbar wrapper element |
| Toolbar is linked to the editing region | `aria-controls="rte-region"` on the toolbar |
| Each toolbar button has `aria-label` | Explicit label on every `<button>` — icon-only buttons have no visible text |
| Active format buttons reflect state | `aria-pressed="true/false"` toggled on every selection-change event |
| Editing region has `aria-multiline="true"` | Tells screen readers to announce line breaks |
| Editing region has `aria-label` | Required because `contenteditable` has no associated `<label>` by default |
| Toolbar buttons do not steal editor focus | `e.preventDefault()` on `mousedown` prevents blur before the command fires |
| Placeholder text is accessible | Rendered via CSS `::before` with `pointer-events: none`; a visible label is always present above the editor |
| Empty-content validation error is announced | `role="alert"` on the validation error paragraph triggers an immediate announcement |
| Keyboard navigation within toolbar | Toolbar buttons are sequential tab stops; `Home`/`End` not required for a simple toolbar |
| Undo/redo is available | Browser `Ctrl/Cmd+Z` works natively inside `contenteditable`; Tiptap/Quill intercept and use their own history |

---

## Production Pitfalls

**1. Using `onChange` + `setState` / `ngModel` causes cursor-jumping**
If you store the HTML string in React state on every keystroke and pass it back as `content` or `value`, the editor unmounts and remounts, resetting the cursor to position 0. Fix: treat the editor as an uncontrolled component. Only read content imperatively (`editor.getHTML()`) on submit or on an explicit save event. If you need a live callback for a character count or autosave, use the `onUpdate` callback — never use it to drive a state variable that feeds back into the editor.

**2. Toolbar commands fire after editor loses focus**
`onClick` on a toolbar button fires `blur` on the editor first, which in some browser/library combinations deselects the text. The command then has no selection to act on. Fix: always use `onMouseDown` + `e.preventDefault()` on toolbar buttons. `preventDefault` blocks the default action of `mousedown`, which is moving focus away from the active element.

**3. Empty editor HTML is not an empty string**
Tiptap returns `'<p></p>'`. Quill returns `'<p><br></p>'`. Neither is a falsy string, so `if (!html)` silently passes. Fix: define an `isEmpty` check specific to your library. For Tiptap: `editor.isEmpty`. For Quill: `quill.getText().trim().length === 0`. Never rely on the raw HTML string for emptiness checks.

**4. Paste from Word or Google Docs injects inline styles, `<span>` soup, and `<font>` tags**
The editor's default paste handler strips some unsafe HTML but preserves many styles that break your site's design. Fix: configure a custom paste handler. In Tiptap, extend the `ClipboardTextSerializer` or add the `transformPastedHTML` option. In Quill, intercept `'paste'` and run the content through `DOMParser` + a sanitiser (`DOMPurify`) before handing the delta to the editor. Always test paste from all three: plain text, Word, and Google Docs.

**5. Server-side rendering (SSR / Next.js App Router) crashes on import**
Both Tiptap and Quill access `document` and `window` at module load time. Importing them at the top of an SSR component throws `ReferenceError: document is not defined`. Fix: in Next.js, use `dynamic(() => import('./RichTextEditor'), { ssr: false })`. In Angular Universal, guard the `init()` call with `isPlatformBrowser(this.platformId)` and inject `PLATFORM_ID`. Never initialise the editor in a constructor or in `ngOnInit` when SSR is enabled — only in `ngAfterViewInit` inside a platform-browser guard.

---

## Interview Angle

**Q: "Why can't you use a controlled input pattern for a rich text editor?"**

The controlled-input pattern requires that the component reflects the value of a state variable on every render. For a plain `<input>`, the browser can replace the element's value and restore cursor position because the cursor is a simple integer offset. For `contenteditable`, the cursor is a `(node, offset)` pair inside a nested DOM tree. When React or Angular re-renders the inner DOM to match new HTML state, the cursor anchor node may no longer exist, causing the cursor to jump to position 0 or to throw. Additionally, rich text editors maintain internal ProseMirror/Delta models that are authoritative — re-rendering from serialized HTML requires a full parse and model rebuild, which destroys undo history and is orders of magnitude slower than a direct command. The correct pattern is uncontrolled: let the library own its DOM, access content imperatively on demand, and subscribe to events for side effects like character counts or autosave.

**Follow-up: "How do you handle form validation for an editor that isn't wired to a form control?"**

Three approaches, depending on strictness. First, on submit, call `editor.getHTML()` or `editor.isEmpty`, and if invalid, set an error message in state and call `editor.commands.focus()` to return the user to the editor. Second, for React Hook Form or Angular Reactive Forms, write a custom field using `useController` (React) or `ControlValueAccessor` (Angular) that subscribes to `onUpdate`/`text-change` and calls `onChange` with the serialized value — this plugs the editor into the form's validation engine without making it controlled. Third, sync the value to a hidden `<input>` inside the same `<form>` on every `onUpdate`, and apply `required` to the hidden input — the browser's native constraint validation then handles it. The hidden-input approach is the simplest for basic required-field validation but does not support min-length or pattern rules without custom code.
