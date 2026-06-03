# Keyboard Navigation

## Table of Contents
- [Introduction](#introduction)
- [Why Keyboard Navigation Matters](#why-keyboard-navigation-matters)
- [Tab Order and Focus Management](#tab-order-and-focus-management)
- [Skip Links](#skip-links)
- [Keyboard Shortcuts](#keyboard-shortcuts)
- [Focus Indicators](#focus-indicators)
- [Modal and Dialog Navigation](#modal-and-dialog-navigation)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Keyboard navigation is a fundamental aspect of web accessibility that enables users to interact with websites without using a mouse. Many users rely on keyboard navigation, including people with motor disabilities, blind users, power users who prefer keyboard shortcuts, and users with broken or missing pointing devices.

```
Keyboard Navigation Flow
========================

     [Tab]         [Shift+Tab]      [Enter/Space]
       |               |                  |
       v               v                  v
  Next Element <-> Previous Element -> Activate
       |               |                  |
       v               v                  v
  [Arrow Keys]    [Escape]           [Home/End]
       |               |                  |
       v               v                  v
  Navigate Menu -> Close Dialog -> Jump to Start/End
```

Proper keyboard navigation ensures that all interactive elements are accessible and that the focus order follows a logical, predictable pattern.

## Why Keyboard Navigation Matters

### User Groups That Depend on Keyboards

1. **Motor Disabilities**: Users who cannot use a mouse due to tremors, paralysis, or limited mobility
2. **Blind Users**: Screen reader users navigate primarily with keyboards
3. **Low Vision Users**: May use screen magnification and prefer keyboard navigation
4. **Power Users**: Prefer keyboard shortcuts for efficiency
5. **Temporary Disabilities**: Users with broken arms or RSI

### Legal and Standards Requirements

- **WCAG 2.1 Level A**: All functionality must be available via keyboard
- **Section 508**: Federal accessibility requirements in the US
- **ADA Compliance**: Americans with Disabilities Act requirements
- **EN 301 549**: European accessibility standard

## Tab Order and Focus Management

### Natural Tab Order

The browser's default tab order follows the DOM structure:

```typescript
// React: Natural Tab Order Example
import React from 'react';

const NaturalTabOrder: React.FC = () => {
  return (
    <div className="form-container">
      <h1>Registration Form</h1>
      
      {/* Tabindex 0 - Natural tab order */}
      <input type="text" placeholder="First Name" />
      <input type="text" placeholder="Last Name" />
      <input type="email" placeholder="Email" />
      <input type="password" placeholder="Password" />
      
      <button type="submit">Submit</button>
      <button type="button">Cancel</button>
    </div>
  );
};

export default NaturalTabOrder;
```

```typescript
// Angular: Natural Tab Order Component
import { Component } from '@angular/core';

@Component({
  selector: 'app-natural-tab-order',
  template: `
    <div class="form-container">
      <h1>Registration Form</h1>
      
      <input type="text" placeholder="First Name" />
      <input type="text" placeholder="Last Name" />
      <input type="email" placeholder="Email" />
      <input type="password" placeholder="Password" />
      
      <button type="submit">Submit</button>
      <button type="button">Cancel</button>
    </div>
  `
})
export class NaturalTabOrderComponent {}
```

### Managing Focus Programmatically

```typescript
// React: Focus Management Hook
import { useRef, useEffect } from 'react';

const useFocusManagement = (shouldFocus: boolean) => {
  const elementRef = useRef<HTMLElement>(null);

  useEffect(() => {
    if (shouldFocus && elementRef.current) {
      elementRef.current.focus();
    }
  }, [shouldFocus]);

  return elementRef;
};

// Usage Example
const DynamicContent: React.FC = () => {
  const [showContent, setShowContent] = React.useState(false);
  const headingRef = useFocusManagement(showContent);

  return (
    <div>
      <button onClick={() => setShowContent(true)}>
        Load Content
      </button>
      
      {showContent && (
        <div>
          <h2 ref={headingRef} tabIndex={-1}>
            New Content Loaded
          </h2>
          <p>This content appeared dynamically.</p>
        </div>
      )}
    </div>
  );
};
```

```typescript
// Angular: Focus Management Directive
import { Directive, ElementRef, Input, OnChanges } from '@angular/core';

@Directive({
  selector: '[appAutoFocus]'
})
export class AutoFocusDirective implements OnChanges {
  @Input() appAutoFocus: boolean = false;

  constructor(private elementRef: ElementRef) {}

  ngOnChanges() {
    if (this.appAutoFocus) {
      setTimeout(() => {
        this.elementRef.nativeElement.focus();
      }, 0);
    }
  }
}

// Usage
@Component({
  selector: 'app-dynamic-content',
  template: `
    <button (click)="showContent = true">Load Content</button>
    
    <div *ngIf="showContent">
      <h2 [appAutoFocus]="showContent" tabindex="-1">
        New Content Loaded
      </h2>
      <p>This content appeared dynamically.</p>
    </div>
  `
})
export class DynamicContentComponent {
  showContent = false;
}
```

### TabIndex Values

```typescript
// React: TabIndex Examples
const TabIndexExamples: React.FC = () => {
  return (
    <div>
      {/* tabIndex={0} - Natural tab order, focusable */}
      <div tabIndex={0} role="button" onClick={handleClick}>
        Custom Button (Focusable)
      </div>

      {/* tabIndex={-1} - Programmatically focusable only */}
      <h2 tabIndex={-1} id="section-heading">
        Section Heading (Focus Target)
      </h2>

      {/* AVOID: Positive tabIndex disrupts natural order */}
      {/* <input tabIndex={3} /> DO NOT DO THIS */}

      {/* Native elements are naturally focusable */}
      <button>Native Button</button>
      <a href="/page">Link</a>
      <input type="text" />
    </div>
  );
};
```

### Focus Trapping in Modals

```typescript
// React: Focus Trap Hook
import { useEffect, useRef } from 'react';

const useFocusTrap = (isActive: boolean) => {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isActive || !containerRef.current) return;

    const container = containerRef.current;
    const focusableElements = container.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;

    const handleTabKey = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        // Shift + Tab: Moving backwards
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement.focus();
        }
      } else {
        // Tab: Moving forwards
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement.focus();
        }
      }
    };

    // Focus first element
    firstElement?.focus();

    container.addEventListener('keydown', handleTabKey);
    return () => container.removeEventListener('keydown', handleTabKey);
  }, [isActive]);

  return containerRef;
};

// Modal with Focus Trap
const Modal: React.FC<{ isOpen: boolean; onClose: () => void }> = ({
  isOpen,
  onClose,
  children
}) => {
  const modalRef = useFocusTrap(isOpen);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Store previous focus
      previousFocusRef.current = document.activeElement as HTMLElement;
    } else if (previousFocusRef.current) {
      // Restore focus when modal closes
      previousFocusRef.current.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div
        ref={modalRef}
        className="modal-content"
        role="dialog"
        aria-modal="true"
        onClick={(e) => e.stopPropagation()}
      >
        <button
          className="modal-close"
          onClick={onClose}
          aria-label="Close modal"
        >
          ×
        </button>
        {children}
      </div>
    </div>
  );
};
```

```typescript
// Angular: Focus Trap Directive
import { Directive, ElementRef, OnInit, OnDestroy } from '@angular/core';

@Directive({
  selector: '[appFocusTrap]'
})
export class FocusTrapDirective implements OnInit, OnDestroy {
  private previousFocus: HTMLElement | null = null;

  constructor(private elementRef: ElementRef) {}

  ngOnInit() {
    this.previousFocus = document.activeElement as HTMLElement;
    this.trapFocus();
  }

  ngOnDestroy() {
    if (this.previousFocus) {
      this.previousFocus.focus();
    }
  }

  private trapFocus() {
    const element = this.elementRef.nativeElement;
    const focusableElements = element.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    const firstElement = focusableElements[0] as HTMLElement;
    const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;

    firstElement?.focus();

    element.addEventListener('keydown', (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;

      if (e.shiftKey) {
        if (document.activeElement === firstElement) {
          e.preventDefault();
          lastElement.focus();
        }
      } else {
        if (document.activeElement === lastElement) {
          e.preventDefault();
          firstElement.focus();
        }
      }
    });
  }
}
```

## Skip Links

Skip links allow keyboard users to bypass repetitive navigation and jump directly to main content.

```typescript
// React: Skip Link Implementation
const Layout: React.FC = ({ children }) => {
  return (
    <>
      {/* Skip Link - Must be first focusable element */}
      <a href="#main-content" className="skip-link">
        Skip to main content
      </a>

      <header>
        <nav>
          <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/about">About</a></li>
            <li><a href="/services">Services</a></li>
            <li><a href="/contact">Contact</a></li>
          </ul>
        </nav>
      </header>

      <main id="main-content" tabIndex={-1}>
        {children}
      </main>

      <footer>
        <p>&copy; 2024 Company Name</p>
      </footer>
    </>
  );
};

// CSS for Skip Link
const styles = `
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: #000;
    color: #fff;
    padding: 8px;
    text-decoration: none;
    z-index: 100;
  }

  .skip-link:focus {
    top: 0;
  }
`;
```

```typescript
// React: Multiple Skip Links
const AdvancedLayout: React.FC = ({ children }) => {
  return (
    <>
      <div className="skip-links">
        <a href="#main-content">Skip to main content</a>
        <a href="#navigation">Skip to navigation</a>
        <a href="#search">Skip to search</a>
        <a href="#footer">Skip to footer</a>
      </div>

      <header>
        <nav id="navigation">
          {/* Navigation content */}
        </nav>
        <div id="search">
          {/* Search form */}
        </div>
      </header>

      <main id="main-content" tabIndex={-1}>
        {children}
      </main>

      <footer id="footer">
        {/* Footer content */}
      </footer>
    </>
  );
};
```

```typescript
// Angular: Skip Link Component
@Component({
  selector: 'app-skip-links',
  template: `
    <div class="skip-links">
      <a href="#main-content" class="skip-link">
        Skip to main content
      </a>
      <a href="#navigation" class="skip-link">
        Skip to navigation
      </a>
    </div>
  `,
  styles: [`
    .skip-links {
      position: relative;
    }
    
    .skip-link {
      position: absolute;
      top: -40px;
      left: 0;
      background: #000;
      color: #fff;
      padding: 8px 16px;
      text-decoration: none;
      z-index: 1000;
    }
    
    .skip-link:focus {
      top: 0;
    }
  `]
})
export class SkipLinksComponent {}
```

## Keyboard Shortcuts

### Implementing Custom Keyboard Shortcuts

```typescript
// React: Keyboard Shortcut Hook
import { useEffect } from 'react';

interface KeyboardShortcut {
  key: string;
  ctrlKey?: boolean;
  shiftKey?: boolean;
  altKey?: boolean;
  metaKey?: boolean;
  callback: () => void;
}

const useKeyboardShortcut = (shortcuts: KeyboardShortcut[]) => {
  useEffect(() => {
    const handleKeyDown = (event: KeyboardEvent) => {
      for (const shortcut of shortcuts) {
        const modifiersMatch =
          (shortcut.ctrlKey === undefined || shortcut.ctrlKey === event.ctrlKey) &&
          (shortcut.shiftKey === undefined || shortcut.shiftKey === event.shiftKey) &&
          (shortcut.altKey === undefined || shortcut.altKey === event.altKey) &&
          (shortcut.metaKey === undefined || shortcut.metaKey === event.metaKey);

        if (event.key === shortcut.key && modifiersMatch) {
          event.preventDefault();
          shortcut.callback();
          break;
        }
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [shortcuts]);
};

// Usage Example
const TextEditor: React.FC = () => {
  const [content, setContent] = React.useState('');
  const [isBold, setIsBold] = React.useState(false);
  const [isItalic, setIsItalic] = React.useState(false);

  useKeyboardShortcut([
    {
      key: 'b',
      ctrlKey: true,
      callback: () => {
        setIsBold(!isBold);
        console.log('Toggle bold');
      }
    },
    {
      key: 'i',
      ctrlKey: true,
      callback: () => {
        setIsItalic(!isItalic);
        console.log('Toggle italic');
      }
    },
    {
      key: 's',
      ctrlKey: true,
      callback: () => {
        console.log('Save document');
      }
    },
    {
      key: '/',
      ctrlKey: true,
      callback: () => {
        console.log('Show keyboard shortcuts help');
      }
    }
  ]);

  return (
    <div>
      <div className="toolbar">
        <button onClick={() => setIsBold(!isBold)} title="Bold (Ctrl+B)">
          <strong>B</strong>
        </button>
        <button onClick={() => setIsItalic(!isItalic)} title="Italic (Ctrl+I)">
          <em>I</em>
        </button>
      </div>
      <textarea
        value={content}
        onChange={(e) => setContent(e.target.value)}
        style={{
          fontWeight: isBold ? 'bold' : 'normal',
          fontStyle: isItalic ? 'italic' : 'normal'
        }}
      />
      <div className="help-text">
        Press Ctrl+/ to view keyboard shortcuts
      </div>
    </div>
  );
};
```

```typescript
// Angular: Keyboard Shortcut Service
import { Injectable } from '@angular/core';
import { fromEvent, Subject } from 'rxjs';
import { filter, takeUntil } from 'rxjs/operators';

interface Shortcut {
  key: string;
  modifiers?: {
    ctrl?: boolean;
    shift?: boolean;
    alt?: boolean;
    meta?: boolean;
  };
  description: string;
}

@Injectable({
  providedIn: 'root'
})
export class KeyboardShortcutService {
  private shortcuts = new Map<string, Subject<void>>();
  private destroy$ = new Subject<void>();

  constructor() {
    this.initListener();
  }

  private initListener() {
    fromEvent<KeyboardEvent>(document, 'keydown')
      .pipe(takeUntil(this.destroy$))
      .subscribe((event) => {
        const shortcutKey = this.createShortcutKey(event);
        const subject = this.shortcuts.get(shortcutKey);
        
        if (subject) {
          event.preventDefault();
          subject.next();
        }
      });
  }

  register(shortcut: Shortcut): Subject<void> {
    const key = this.createKeyFromShortcut(shortcut);
    
    if (!this.shortcuts.has(key)) {
      this.shortcuts.set(key, new Subject<void>());
    }
    
    return this.shortcuts.get(key)!;
  }

  private createShortcutKey(event: KeyboardEvent): string {
    const parts = [];
    if (event.ctrlKey) parts.push('ctrl');
    if (event.shiftKey) parts.push('shift');
    if (event.altKey) parts.push('alt');
    if (event.metaKey) parts.push('meta');
    parts.push(event.key.toLowerCase());
    return parts.join('+');
  }

  private createKeyFromShortcut(shortcut: Shortcut): string {
    const parts = [];
    if (shortcut.modifiers?.ctrl) parts.push('ctrl');
    if (shortcut.modifiers?.shift) parts.push('shift');
    if (shortcut.modifiers?.alt) parts.push('alt');
    if (shortcut.modifiers?.meta) parts.push('meta');
    parts.push(shortcut.key.toLowerCase());
    return parts.join('+');
  }

  destroy() {
    this.destroy$.next();
    this.destroy$.complete();
    this.shortcuts.clear();
  }
}

// Usage in Component
@Component({
  selector: 'app-text-editor',
  template: `
    <div class="toolbar">
      <button (click)="toggleBold()" title="Bold (Ctrl+B)">
        <strong>B</strong>
      </button>
      <button (click)="toggleItalic()" title="Italic (Ctrl+I)">
        <em>I</em>
      </button>
    </div>
    <textarea
      [(ngModel)]="content"
      [style.font-weight]="isBold ? 'bold' : 'normal'"
      [style.font-style]="isItalic ? 'italic' : 'normal'"
    ></textarea>
  `
})
export class TextEditorComponent implements OnInit, OnDestroy {
  content = '';
  isBold = false;
  isItalic = false;

  constructor(private shortcutService: KeyboardShortcutService) {}

  ngOnInit() {
    this.shortcutService.register({
      key: 'b',
      modifiers: { ctrl: true },
      description: 'Toggle bold'
    }).subscribe(() => this.toggleBold());

    this.shortcutService.register({
      key: 'i',
      modifiers: { ctrl: true },
      description: 'Toggle italic'
    }).subscribe(() => this.toggleItalic());
  }

  toggleBold() {
    this.isBold = !this.isBold;
  }

  toggleItalic() {
    this.isItalic = !this.isItalic;
  }

  ngOnDestroy() {
    // Cleanup handled by service
  }
}
```

### Arrow Key Navigation

```typescript
// React: Arrow Key Navigation for Lists
const NavigableList: React.FC<{ items: string[] }> = ({ items }) => {
  const [focusedIndex, setFocusedIndex] = React.useState(0);
  const itemRefs = React.useRef<(HTMLLIElement | null)[]>([]);

  const handleKeyDown = (event: React.KeyboardEvent, index: number) => {
    let newIndex = index;

    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        newIndex = Math.min(index + 1, items.length - 1);
        break;
      case 'ArrowUp':
        event.preventDefault();
        newIndex = Math.max(index - 1, 0);
        break;
      case 'Home':
        event.preventDefault();
        newIndex = 0;
        break;
      case 'End':
        event.preventDefault();
        newIndex = items.length - 1;
        break;
      default:
        return;
    }

    setFocusedIndex(newIndex);
    itemRefs.current[newIndex]?.focus();
  };

  return (
    <ul role="listbox" aria-label="Navigable list">
      {items.map((item, index) => (
        <li
          key={item}
          ref={(el) => (itemRefs.current[index] = el)}
          role="option"
          tabIndex={index === focusedIndex ? 0 : -1}
          aria-selected={index === focusedIndex}
          onKeyDown={(e) => handleKeyDown(e, index)}
          onClick={() => setFocusedIndex(index)}
          className={index === focusedIndex ? 'focused' : ''}
        >
          {item}
        </li>
      ))}
    </ul>
  );
};
```

```typescript
// React: Roving TabIndex for Grid Navigation
const NavigableGrid: React.FC = () => {
  const [focusedCell, setFocusedCell] = React.useState({ row: 0, col: 0 });
  const gridData = Array(5).fill(null).map((_, i) => 
    Array(5).fill(null).map((_, j) => `Cell ${i}-${j}`)
  );
  const cellRefs = React.useRef<(HTMLButtonElement | null)[][]>(
    Array(5).fill(null).map(() => Array(5).fill(null))
  );

  const handleKeyDown = (
    event: React.KeyboardEvent,
    row: number,
    col: number
  ) => {
    let newRow = row;
    let newCol = col;

    switch (event.key) {
      case 'ArrowRight':
        event.preventDefault();
        newCol = Math.min(col + 1, 4);
        break;
      case 'ArrowLeft':
        event.preventDefault();
        newCol = Math.max(col - 1, 0);
        break;
      case 'ArrowDown':
        event.preventDefault();
        newRow = Math.min(row + 1, 4);
        break;
      case 'ArrowUp':
        event.preventDefault();
        newRow = Math.max(row - 1, 0);
        break;
      case 'Home':
        event.preventDefault();
        newCol = 0;
        break;
      case 'End':
        event.preventDefault();
        newCol = 4;
        break;
      default:
        return;
    }

    setFocusedCell({ row: newRow, col: newCol });
    cellRefs.current[newRow][newCol]?.focus();
  };

  return (
    <div role="grid" aria-label="Navigable grid">
      {gridData.map((row, rowIndex) => (
        <div key={rowIndex} role="row">
          {row.map((cell, colIndex) => (
            <button
              key={`${rowIndex}-${colIndex}`}
              ref={(el) => {
                if (!cellRefs.current[rowIndex]) {
                  cellRefs.current[rowIndex] = [];
                }
                cellRefs.current[rowIndex][colIndex] = el;
              }}
              role="gridcell"
              tabIndex={
                rowIndex === focusedCell.row && colIndex === focusedCell.col
                  ? 0
                  : -1
              }
              onKeyDown={(e) => handleKeyDown(e, rowIndex, colIndex)}
              onClick={() => setFocusedCell({ row: rowIndex, col: colIndex })}
              className="grid-cell"
            >
              {cell}
            </button>
          ))}
        </div>
      ))}
    </div>
  );
};
```

## Focus Indicators

### Visible Focus Styles

```typescript
// React: Custom Focus Indicators
const FocusExamples: React.FC = () => {
  return (
    <div className="focus-examples">
      {/* Default browser focus */}
      <button className="default-focus">
        Default Focus
      </button>

      {/* Custom focus ring */}
      <button className="custom-focus-ring">
        Custom Focus Ring
      </button>

      {/* Focus with offset */}
      <button className="focus-offset">
        Focus with Offset
      </button>

      {/* Focus visible (keyboard only) */}
      <button className="focus-visible-only">
        Focus Visible Only
      </button>
    </div>
  );
};

const styles = `
  /* Never remove focus outline without replacement! */
  /* WRONG: button:focus { outline: none; } */

  /* Good: Custom focus style */
  .custom-focus-ring:focus {
    outline: 3px solid #4A90E2;
    outline-offset: 2px;
  }

  /* Focus with background */
  .focus-offset:focus {
    outline: 2px solid #000;
    outline-offset: 4px;
    background-color: #FFF3CD;
  }

  /* Focus-visible: Style keyboard focus, not mouse clicks */
  .focus-visible-only:focus {
    outline: none;
  }

  .focus-visible-only:focus-visible {
    outline: 3px solid #4A90E2;
    outline-offset: 2px;
  }

  /* High contrast mode support */
  @media (prefers-contrast: high) {
    button:focus {
      outline: 3px solid currentColor;
    }
  }
`;
```

```typescript
// React: Focus Within Pattern
const CardWithFocusWithin: React.FC = () => {
  return (
    <div className="card">
      <h3>Interactive Card</h3>
      <p>Card with multiple interactive elements</p>
      <button>Action 1</button>
      <button>Action 2</button>
    </div>
  );
};

const focusWithinStyles = `
  .card {
    border: 2px solid transparent;
    padding: 16px;
    transition: border-color 0.2s;
  }

  /* Highlight card when any child has focus */
  .card:focus-within {
    border-color: #4A90E2;
    box-shadow: 0 0 0 3px rgba(74, 144, 226, 0.1);
  }

  .card button:focus {
    outline: 2px solid #4A90E2;
    outline-offset: 2px;
  }
`;
```

## Modal and Dialog Navigation

```typescript
// React: Accessible Modal with Full Keyboard Support
import React, { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

const AccessibleModal: React.FC<ModalProps> = ({
  isOpen,
  onClose,
  title,
  children
}) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (!isOpen) return;

    // Store previous focus
    previousFocusRef.current = document.activeElement as HTMLElement;

    // Focus modal
    modalRef.current?.focus();

    // Prevent body scroll
    document.body.style.overflow = 'hidden';

    // Handle escape key
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    document.addEventListener('keydown', handleEscape);

    return () => {
      document.removeEventListener('keydown', handleEscape);
      document.body.style.overflow = '';
      
      // Restore focus
      if (previousFocusRef.current) {
        previousFocusRef.current.focus();
      }
    };
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="modal-overlay"
      onClick={onClose}
      role="presentation"
    >
      <div
        ref={modalRef}
        className="modal"
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        onClick={(e) => e.stopPropagation()}
        tabIndex={-1}
      >
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            onClick={onClose}
            aria-label="Close dialog"
            className="modal-close"
          >
            ×
          </button>
        </div>
        <div className="modal-body">
          {children}
        </div>
        <div className="modal-footer">
          <button onClick={onClose}>Cancel</button>
          <button onClick={onClose} autoFocus>
            Confirm
          </button>
        </div>
      </div>
    </div>,
    document.body
  );
};
```

## Common Mistakes

### 1. Removing Focus Indicators

```typescript
// ❌ WRONG: Removing focus outline without replacement
const styles = `
  button:focus {
    outline: none; /* Never do this! */
  }
`;

// ✅ CORRECT: Replace with visible alternative
const correctStyles = `
  button:focus {
    outline: 2px solid #4A90E2;
    outline-offset: 2px;
  }
  
  /* Or use focus-visible for keyboard-only focus */
  button:focus {
    outline: none;
  }
  
  button:focus-visible {
    outline: 2px solid #4A90E2;
    outline-offset: 2px;
  }
`;
```

### 2. Positive TabIndex Values

```typescript
// ❌ WRONG: Using positive tabindex
<input tabIndex={1} />
<input tabIndex={3} />
<input tabIndex={2} />

// ✅ CORRECT: Use natural tab order or 0/-1
<input /> {/* Natural order */}
<input tabIndex={0} /> {/* Include in tab order */}
<div tabIndex={-1}>Programmatically focusable</div>
```

### 3. Non-Focusable Interactive Elements

```typescript
// ❌ WRONG: div with onClick, not keyboard accessible
const WrongButton = () => (
  <div onClick={handleClick}>
    Click me
  </div>
);

// ✅ CORRECT: Use button or add keyboard support
const CorrectButton = () => (
  <button onClick={handleClick}>
    Click me
  </button>
);

// ✅ ALTERNATIVE: Make div keyboard accessible
const AccessibleDiv = () => (
  <div
    role="button"
    tabIndex={0}
    onClick={handleClick}
    onKeyDown={(e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        handleClick();
      }
    }}
  >
    Click me
  </div>
);
```

### 4. Forgetting Focus Management After Route Changes

```typescript
// ❌ WRONG: No focus management on route change
const App = () => {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </Router>
  );
};

// ✅ CORRECT: Focus management on route change
const App = () => {
  const location = useLocation();
  const mainRef = useRef<HTMLElement>(null);

  useEffect(() => {
    mainRef.current?.focus();
  }, [location]);

  return (
    <Router>
      <main ref={mainRef} tabIndex={-1}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
        </Routes>
      </main>
    </Router>
  );
};
```

## Best Practices

### 1. Maintain Logical Tab Order

```typescript
// Visual order should match DOM order
const LogicalOrder: React.FC = () => {
  return (
    <form>
      {/* 1 */ <input type="text" placeholder="First Name" />}
      {/* 2 */ <input type="text" placeholder="Last Name" />}
      {/* 3 */ <input type="email" placeholder="Email" />}
      {/* 4 */ <button type="submit">Submit</button>}
    </form>
  );
};
```

### 2. Announce Dynamic Content

```typescript
// React: Live Region for Announcements
const SearchResults: React.FC = () => {
  const [results, setResults] = React.useState<string[]>([]);
  const [searchTerm, setSearchTerm] = React.useState('');

  return (
    <div>
      <input
        type="search"
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        aria-label="Search"
      />
      
      {/* Screen reader announcement */}
      <div role="status" aria-live="polite" aria-atomic="true">
        {results.length > 0 && `${results.length} results found`}
      </div>

      <ul>
        {results.map((result) => (
          <li key={result}>{result}</li>
        ))}
      </ul>
    </div>
  );
};
```

### 3. Provide Keyboard Shortcuts Documentation

```typescript
// React: Keyboard Shortcuts Help Modal
const KeyboardShortcutsHelp: React.FC = () => {
  const shortcuts = [
    { keys: 'Tab', description: 'Move to next element' },
    { keys: 'Shift + Tab', description: 'Move to previous element' },
    { keys: 'Enter', description: 'Activate button or link' },
    { keys: 'Space', description: 'Activate button or toggle checkbox' },
    { keys: 'Escape', description: 'Close modal or cancel' },
    { keys: 'Ctrl + S', description: 'Save document' },
    { keys: 'Ctrl + /', description: 'Show this help' }
  ];

  return (
    <div className="shortcuts-help">
      <h2>Keyboard Shortcuts</h2>
      <table>
        <thead>
          <tr>
            <th>Keys</th>
            <th>Action</th>
          </tr>
        </thead>
        <tbody>
          {shortcuts.map((shortcut) => (
            <tr key={shortcut.keys}>
              <td><kbd>{shortcut.keys}</kbd></td>
              <td>{shortcut.description}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};
```

### 4. Test with Keyboard Only

Always test your application using only the keyboard:
- Unplug your mouse
- Use Tab to navigate forward
- Use Shift+Tab to navigate backward
- Use Enter/Space to activate elements
- Ensure all functionality is accessible

## Interview Questions

### Junior Level

1. **What is keyboard navigation and why is it important?**
   - Navigation using only keyboard
   - Essential for accessibility
   - Required by WCAG
   - Used by screen reader users and motor-impaired users

2. **What keys are commonly used for keyboard navigation?**
   - Tab/Shift+Tab for focus movement
   - Enter/Space for activation
   - Arrow keys for menus and lists
   - Escape for closing modals

3. **What is the purpose of the tabindex attribute?**
   - Controls tab order
   - 0 = natural order, focusable
   - -1 = programmatically focusable
   - Positive values = disrupts natural order (avoid)

### Mid Level

4. **How do you implement a focus trap in a modal dialog?**
   - Track all focusable elements
   - Handle Tab and Shift+Tab
   - Cycle focus within modal
   - Return focus when closed

5. **What are skip links and when should you use them?**
   - Links to bypass repetitive content
   - Usually "Skip to main content"
   - First focusable element
   - Visible on focus

6. **Explain the difference between :focus and :focus-visible**
   - :focus = all focus (mouse and keyboard)
   - :focus-visible = keyboard focus only
   - Better UX for mouse users

### Senior Level

7. **How would you implement accessible keyboard shortcuts without conflicting with browser/screen reader shortcuts?**
   - Use modifier keys (Ctrl, Alt)
   - Avoid single-key shortcuts
   - Make shortcuts configurable
   - Document all shortcuts
   - Test with screen readers

8. **Describe strategies for managing focus in single-page applications**
   - Focus main heading on route change
   - Use tabIndex={-1} on containers
   - Announce route changes
   - Restore scroll position
   - Handle focus for dynamic content

9. **How do you handle keyboard navigation for complex widgets like data grids or tree views?**
   - Arrow keys for 2D navigation
   - Home/End for boundaries
   - Page Up/Page Down for sections
   - Type-ahead search
   - Follow ARIA authoring practices

10. **What is roving tabindex and when would you use it?**
    - Only one element in group has tabIndex={0}
    - Others have tabIndex={-1}
    - Manage focus with arrow keys
    - Used for toolbars, menus, grids
    - Reduces Tab stops

## Key Takeaways

1. **All functionality must be keyboard accessible** - WCAG requirement
2. **Never remove focus indicators** without visible replacement
3. **Avoid positive tabIndex** values - they disrupt natural order
4. **Implement focus traps** for modals and dialogs
5. **Provide skip links** to bypass repetitive content
6. **Test with keyboard only** - unplug your mouse
7. **Use semantic HTML** - native elements have built-in keyboard support
8. **Manage focus** for dynamic content and route changes
9. **Document keyboard shortcuts** for power users
10. **Follow ARIA authoring practices** for complex widgets

## Resources

### Official Documentation
- [WCAG 2.1 - Keyboard Accessible](https://www.w3.org/WAI/WCAG21/Understanding/keyboard-accessible)
- [MDN - Keyboard-navigable JavaScript widgets](https://developer.mozilla.org/en-US/docs/Web/Accessibility/Keyboard-navigable_JavaScript_widgets)
- [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/)

### Tools
- [Tab Order Visualizer Chrome Extension](https://chrome.google.com/webstore/detail/tab-order-visualizer)
- [Focus Indicator Chrome Extension](https://chrome.google.com/webstore/detail/focus-indicator)
- [Accessibility Insights](https://accessibilityinsights.io/)

### Articles
- [WebAIM: Keyboard Accessibility](https://webaim.org/articles/keyboard/)
- [The A11Y Project: Keyboard Navigation](https://www.a11yproject.com/posts/keyboard-navigation/)
- [Smashing Magazine: A Complete Guide To Accessible Front-End Components](https://www.smashingmagazine.com/2021/03/complete-guide-accessible-front-end-components/)

### Libraries
- [focus-trap-react](https://github.com/focus-trap/focus-trap-react)
- [react-hotkeys-hook](https://github.com/JohannesKlauss/react-hotkeys-hook)
- [@angular/cdk/a11y](https://material.angular.io/cdk/a11y/overview)
