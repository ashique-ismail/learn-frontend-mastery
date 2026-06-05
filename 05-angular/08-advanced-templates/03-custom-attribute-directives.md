# Custom Attribute Directives

## The Idea

**In plain English:** A custom attribute directive is a reusable instruction you attach to any HTML element to change how it looks or behaves — like adding a special label to something that tells it to glow when you hover over it. The directive (the instruction) is defined once in code and can be slapped onto as many elements as you want.

**Real-world analogy:** Think of a coffee shop loyalty-card stamp. When a barista stamps your card, that stamp changes what the card "does" (gets you closer to a free drink) without replacing or removing the card itself. You can stamp any loyalty card — it works the same way every time.

- The loyalty card = the HTML element the directive is applied to
- The stamp = the directive attribute you add to the element
- The rule printed on the stamp = the directive's TypeScript class that defines the behavior
- Stamping different cards = applying the same directive to different elements

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding Attribute Directives](#understanding-attribute-directives)
3. [ElementRef and Renderer2](#elementref-and-renderer2)
4. [Creating Basic Attribute Directives](#creating-basic-attribute-directives)
5. [HostListener and HostBinding](#hostlistener-and-hostbinding)
6. [Directive Lifecycle Hooks](#directive-lifecycle-hooks)
7. [Advanced Directive Patterns](#advanced-directive-patterns)
8. [Directive Composition](#directive-composition)
9. [Testing Attribute Directives](#testing-attribute-directives)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)
13. [Key Takeaways](#key-takeaways)
14. [Resources](#resources)

## Introduction

Attribute directives modify the appearance or behavior of DOM elements, components, or other directives. Unlike structural directives that add or remove DOM elements, attribute directives change how existing elements look or behave. Angular provides built-in attribute directives like `ngClass`, `ngStyle`, and `ngModel`, but creating custom attribute directives allows you to encapsulate reusable behaviors and DOM manipulation logic.

This comprehensive guide covers everything from basic attribute directives to advanced patterns, including proper DOM manipulation techniques, event handling, and lifecycle management.

## Understanding Attribute Directives

Attribute directives are classes decorated with `@Directive` that can be applied to elements as attributes. They have access to the host element and can modify its properties, styles, attributes, and respond to events.

### Basic Structure

```typescript
import { Directive, ElementRef } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  constructor(private el: ElementRef) {
    // Direct access to native element
    this.el.nativeElement.style.backgroundColor = 'yellow';
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HighlightDirective],
  template: `<p appHighlight>Highlighted text</p>`
})
export class AppComponent {}
```

### Directive vs Component

```typescript
// Component: Has a template
@Component({
  selector: 'app-card',
  standalone: true,
  template: `<div class="card">Content</div>`
})
export class CardComponent {}

// Directive: No template, modifies host element
@Directive({
  selector: '[appCard]',
  standalone: true,
  host: {
    'class': 'card'
  }
})
export class CardDirective {}
```

## ElementRef and Renderer2

### ElementRef

`ElementRef` provides direct access to the host DOM element, but direct manipulation is not recommended for security and server-side rendering compatibility.

```typescript
import { Directive, ElementRef } from '@angular/core';

@Directive({
  selector: '[appElementRef]',
  standalone: true
})
export class ElementRefDirective {
  constructor(private el: ElementRef) {
    console.log('Native element:', this.el.nativeElement);
    console.log('Tag name:', this.el.nativeElement.tagName);
    
    // Direct manipulation (not recommended)
    this.el.nativeElement.style.color = 'red';
  }
}
```

### Renderer2

`Renderer2` is the recommended way to manipulate the DOM as it works with server-side rendering and is more secure.

```typescript
import { Directive, ElementRef, Renderer2, OnInit } from '@angular/core';

@Directive({
  selector: '[appRenderer]',
  standalone: true
})
export class RendererDirective implements OnInit {
  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  ngOnInit() {
    // Set styles
    this.renderer.setStyle(this.el.nativeElement, 'color', 'blue');
    this.renderer.setStyle(this.el.nativeElement, 'font-weight', 'bold');
    
    // Add/remove classes
    this.renderer.addClass(this.el.nativeElement, 'active');
    this.renderer.removeClass(this.el.nativeElement, 'inactive');
    
    // Set attributes
    this.renderer.setAttribute(this.el.nativeElement, 'data-id', '123');
    this.renderer.removeAttribute(this.el.nativeElement, 'disabled');
    
    // Set properties
    this.renderer.setProperty(this.el.nativeElement, 'textContent', 'New text');
    
    // Create and append elements
    const span = this.renderer.createElement('span');
    const text = this.renderer.createText('Added text');
    this.renderer.appendChild(span, text);
    this.renderer.appendChild(this.el.nativeElement, span);
  }
}
```

### Renderer2 Methods Reference

```typescript
import { Directive, ElementRef, Renderer2 } from '@angular/core';

@Directive({
  selector: '[appRendererMethods]',
  standalone: true
})
export class RendererMethodsDirective {
  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {
    const element = this.el.nativeElement;
    
    // Element manipulation
    this.renderer.createElement('div');
    this.renderer.createText('text');
    this.renderer.appendChild(element, childElement);
    this.renderer.insertBefore(element, newChild, refChild);
    this.renderer.removeChild(element, childElement);
    
    // Style manipulation
    this.renderer.setStyle(element, 'color', 'red');
    this.renderer.removeStyle(element, 'color');
    
    // Class manipulation
    this.renderer.addClass(element, 'active');
    this.renderer.removeClass(element, 'active');
    
    // Attribute manipulation
    this.renderer.setAttribute(element, 'id', 'myId');
    this.renderer.removeAttribute(element, 'id');
    
    // Property manipulation
    this.renderer.setProperty(element, 'value', 'text');
    
    // Event listeners
    const unlisten = this.renderer.listen(element, 'click', (event) => {
      console.log('Clicked', event);
    });
    // Call unlisten() to remove listener
    
    // Parent/sibling access
    this.renderer.parentNode(element);
    this.renderer.nextSibling(element);
  }
}
```

## Creating Basic Attribute Directives

### Simple Hover Directive

```typescript
import { Directive, ElementRef, Renderer2, HostListener } from '@angular/core';

@Directive({
  selector: '[appHover]',
  standalone: true
})
export class HoverDirective {
  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  @HostListener('mouseenter') onMouseEnter() {
    this.highlight('yellow');
  }

  @HostListener('mouseleave') onMouseLeave() {
    this.highlight('');
  }

  private highlight(color: string) {
    this.renderer.setStyle(this.el.nativeElement, 'backgroundColor', color);
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HoverDirective],
  template: `<div appHover>Hover over me!</div>`
})
export class AppComponent {}
```

### Configurable Highlight Directive

```typescript
import { Directive, ElementRef, Renderer2, Input, OnInit } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective implements OnInit {
  @Input() appHighlight = 'yellow';
  @Input() defaultColor = 'transparent';

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  ngOnInit() {
    this.highlight(this.defaultColor || this.appHighlight);
  }

  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.appHighlight);
  }

  @HostListener('mouseleave') onMouseLeave() {
    this.highlight(this.defaultColor);
  }

  private highlight(color: string) {
    this.renderer.setStyle(this.el.nativeElement, 'backgroundColor', color);
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HighlightDirective],
  template: `
    <!-- Default yellow -->
    <p appHighlight>Default highlight</p>
    
    <!-- Custom color -->
    <p [appHighlight]="'lightblue'">Blue highlight</p>
    
    <!-- Custom color with default -->
    <p [appHighlight]="'pink'" [defaultColor]="'lightgray'">
      Pink highlight with gray default
    </p>
  `
})
export class AppComponent {}
```

### Click Outside Directive

```typescript
import { 
  Directive, 
  ElementRef, 
  Output, 
  EventEmitter, 
  HostListener 
} from '@angular/core';

@Directive({
  selector: '[appClickOutside]',
  standalone: true
})
export class ClickOutsideDirective {
  @Output() clickOutside = new EventEmitter<void>();

  constructor(private elementRef: ElementRef) {}

  @HostListener('document:click', ['$event.target'])
  onDocumentClick(target: HTMLElement) {
    const clickedInside = this.elementRef.nativeElement.contains(target);
    if (!clickedInside) {
      this.clickOutside.emit();
    }
  }
}

// Usage
@Component({
  selector: 'app-dropdown',
  standalone: true,
  imports: [ClickOutsideDirective],
  template: `
    <div class="dropdown" (appClickOutside)="close()">
      <button (click)="toggle()">Toggle</button>
      <div class="menu" *ngIf="isOpen">
        Dropdown content
      </div>
    </div>
  `
})
export class DropdownComponent {
  isOpen = false;

  toggle() {
    this.isOpen = !this.isOpen;
  }

  close() {
    this.isOpen = false;
  }
}
```

### Copy to Clipboard Directive

```typescript
import { 
  Directive, 
  Input, 
  HostListener, 
  ElementRef,
  Renderer2 
} from '@angular/core';

@Directive({
  selector: '[appCopyToClipboard]',
  standalone: true
})
export class CopyToClipboardDirective {
  @Input() appCopyToClipboard = '';
  @Input() copySuccessMessage = 'Copied!';

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  @HostListener('click')
  async onClick() {
    try {
      await navigator.clipboard.writeText(this.appCopyToClipboard);
      this.showFeedback();
    } catch (err) {
      console.error('Failed to copy:', err);
    }
  }

  private showFeedback() {
    const originalText = this.el.nativeElement.textContent;
    this.renderer.setProperty(
      this.el.nativeElement, 
      'textContent', 
      this.copySuccessMessage
    );
    
    setTimeout(() => {
      this.renderer.setProperty(
        this.el.nativeElement, 
        'textContent', 
        originalText
      );
    }, 1000);
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [CopyToClipboardDirective],
  template: `
    <button [appCopyToClipboard]="code">
      Copy Code
    </button>
  `
})
export class AppComponent {
  code = 'const x = 10;';
}
```

## HostListener and HostBinding

### HostListener

`@HostListener` decorates a directive method to listen to events on the host element.

```typescript
import { Directive, HostListener, ElementRef } from '@angular/core';

@Directive({
  selector: '[appHostListenerExample]',
  standalone: true
})
export class HostListenerExampleDirective {
  constructor(private el: ElementRef) {}

  // Listen to simple events
  @HostListener('click')
  onClick() {
    console.log('Element clicked');
  }

  // Access event object
  @HostListener('mouseenter', ['$event'])
  onMouseEnter(event: MouseEvent) {
    console.log('Mouse entered at:', event.clientX, event.clientY);
  }

  // Multiple parameters
  @HostListener('keydown', ['$event.key', '$event.ctrlKey'])
  onKeyDown(key: string, ctrlKey: boolean) {
    if (ctrlKey && key === 's') {
      console.log('Ctrl+S pressed');
    }
  }

  // Window events
  @HostListener('window:resize', ['$event'])
  onWindowResize(event: Event) {
    console.log('Window resized');
  }

  // Document events
  @HostListener('document:click', ['$event.target'])
  onDocumentClick(target: HTMLElement) {
    console.log('Clicked on:', target);
  }
}
```

### HostBinding

`@HostBinding` decorates a directive property to bind it to a host element property.

```typescript
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appHostBindingExample]',
  standalone: true
})
export class HostBindingExampleDirective {
  // Bind to class
  @HostBinding('class.active') isActive = false;
  
  // Bind to style
  @HostBinding('style.backgroundColor') backgroundColor = 'white';
  @HostBinding('style.border') border = '1px solid black';
  
  // Bind to attribute
  @HostBinding('attr.role') role = 'button';
  @HostBinding('attr.aria-pressed') get ariaPressed() {
    return this.isActive.toString();
  }
  
  // Bind to property
  @HostBinding('disabled') isDisabled = false;

  @HostListener('click')
  onClick() {
    this.isActive = !this.isActive;
    this.backgroundColor = this.isActive ? 'lightblue' : 'white';
  }
}
```

### Combined HostListener and HostBinding Example

```typescript
import { Directive, HostBinding, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appButton]',
  standalone: true
})
export class ButtonDirective {
  @Input() appButton: 'primary' | 'secondary' | 'danger' = 'primary';
  
  @HostBinding('class') get classes() {
    return `btn btn-${this.appButton}`;
  }
  
  @HostBinding('class.pressed') isPressed = false;
  @HostBinding('class.disabled') @Input() disabled = false;
  
  @HostBinding('style.transform') get transform() {
    return this.isPressed ? 'scale(0.95)' : 'scale(1)';
  }
  
  @HostBinding('style.transition') transition = 'transform 0.1s';
  
  @HostBinding('attr.role') role = 'button';
  @HostBinding('attr.tabindex') tabindex = '0';

  @HostListener('mousedown')
  onMouseDown() {
    if (!this.disabled) {
      this.isPressed = true;
    }
  }

  @HostListener('mouseup')
  @HostListener('mouseleave')
  onMouseUp() {
    this.isPressed = false;
  }

  @HostListener('keydown.enter', ['$event'])
  @HostListener('keydown.space', ['$event'])
  onKeyDown(event: KeyboardEvent) {
    event.preventDefault();
    if (!this.disabled) {
      this.isPressed = true;
      setTimeout(() => this.isPressed = false, 100);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ButtonDirective],
  template: `
    <button appButton="primary">Primary</button>
    <button appButton="danger" [disabled]="true">Disabled</button>
  `
})
export class AppComponent {}
```

## Directive Lifecycle Hooks

### Available Lifecycle Hooks

```typescript
import { 
  Directive, 
  OnInit, 
  OnChanges, 
  DoCheck, 
  AfterContentInit,
  AfterContentChecked,
  AfterViewInit,
  AfterViewChecked,
  OnDestroy,
  SimpleChanges,
  Input
} from '@angular/core';

@Directive({
  selector: '[appLifecycle]',
  standalone: true
})
export class LifecycleDirective implements 
  OnInit, 
  OnChanges, 
  DoCheck, 
  AfterContentInit,
  AfterContentChecked,
  AfterViewInit,
  AfterViewChecked,
  OnDestroy 
{
  @Input() data: any;

  constructor() {
    console.log('Constructor called');
  }

  ngOnChanges(changes: SimpleChanges) {
    console.log('ngOnChanges:', changes);
  }

  ngOnInit() {
    console.log('ngOnInit called');
  }

  ngDoCheck() {
    console.log('ngDoCheck called');
  }

  ngAfterContentInit() {
    console.log('ngAfterContentInit called');
  }

  ngAfterContentChecked() {
    console.log('ngAfterContentChecked called');
  }

  ngAfterViewInit() {
    console.log('ngAfterViewInit called');
  }

  ngAfterViewChecked() {
    console.log('ngAfterViewChecked called');
  }

  ngOnDestroy() {
    console.log('ngOnDestroy called - cleanup here');
  }
}
```

### Practical Lifecycle Usage

```typescript
import { 
  Directive, 
  ElementRef, 
  Renderer2, 
  OnInit, 
  OnDestroy, 
  Input 
} from '@angular/core';

@Directive({
  selector: '[appAutoSave]',
  standalone: true
})
export class AutoSaveDirective implements OnInit, OnDestroy {
  @Input() autoSaveDelay = 2000;
  private timeoutId?: number;
  private unlistenFn?: () => void;

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  ngOnInit() {
    // Setup in ngOnInit, not constructor
    this.unlistenFn = this.renderer.listen(
      this.el.nativeElement,
      'input',
      (event) => this.onInput(event)
    );
  }

  onInput(event: Event) {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }

    this.timeoutId = window.setTimeout(() => {
      this.save((event.target as HTMLInputElement).value);
    }, this.autoSaveDelay);
  }

  save(value: string) {
    console.log('Auto-saving:', value);
    // Implement save logic
  }

  ngOnDestroy() {
    // Cleanup
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }
    if (this.unlistenFn) {
      this.unlistenFn();
    }
  }
}
```

## Advanced Directive Patterns

### Tooltip Directive

```typescript
import { 
  Directive, 
  Input, 
  ElementRef, 
  Renderer2, 
  HostListener,
  OnDestroy 
} from '@angular/core';

@Directive({
  selector: '[appTooltip]',
  standalone: true
})
export class TooltipDirective implements OnDestroy {
  @Input() appTooltip = '';
  @Input() tooltipPosition: 'top' | 'bottom' | 'left' | 'right' = 'top';
  @Input() tooltipDelay = 300;

  private tooltipElement?: HTMLElement;
  private showTimeoutId?: number;

  constructor(
    private el: ElementRef,
    private renderer: Renderer2
  ) {}

  @HostListener('mouseenter')
  onMouseEnter() {
    this.showTimeoutId = window.setTimeout(() => {
      this.show();
    }, this.tooltipDelay);
  }

  @HostListener('mouseleave')
  onMouseLeave() {
    if (this.showTimeoutId) {
      clearTimeout(this.showTimeoutId);
    }
    this.hide();
  }

  private show() {
    if (!this.appTooltip) return;

    this.tooltipElement = this.renderer.createElement('div');
    this.renderer.addClass(this.tooltipElement, 'tooltip');
    this.renderer.addClass(this.tooltipElement, `tooltip-${this.tooltipPosition}`);
    
    const text = this.renderer.createText(this.appTooltip);
    this.renderer.appendChild(this.tooltipElement, text);
    
    this.renderer.appendChild(document.body, this.tooltipElement);
    
    this.position();
  }

  private position() {
    if (!this.tooltipElement) return;

    const hostPos = this.el.nativeElement.getBoundingClientRect();
    const tooltipPos = this.tooltipElement.getBoundingClientRect();

    let top = 0;
    let left = 0;

    switch (this.tooltipPosition) {
      case 'top':
        top = hostPos.top - tooltipPos.height - 8;
        left = hostPos.left + (hostPos.width - tooltipPos.width) / 2;
        break;
      case 'bottom':
        top = hostPos.bottom + 8;
        left = hostPos.left + (hostPos.width - tooltipPos.width) / 2;
        break;
      case 'left':
        top = hostPos.top + (hostPos.height - tooltipPos.height) / 2;
        left = hostPos.left - tooltipPos.width - 8;
        break;
      case 'right':
        top = hostPos.top + (hostPos.height - tooltipPos.height) / 2;
        left = hostPos.right + 8;
        break;
    }

    this.renderer.setStyle(this.tooltipElement, 'top', `${top}px`);
    this.renderer.setStyle(this.tooltipElement, 'left', `${left}px`);
  }

  private hide() {
    if (this.tooltipElement) {
      this.renderer.removeChild(document.body, this.tooltipElement);
      this.tooltipElement = undefined;
    }
  }

  ngOnDestroy() {
    this.hide();
    if (this.showTimeoutId) {
      clearTimeout(this.showTimeoutId);
    }
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [TooltipDirective],
  template: `
    <button [appTooltip]="'This is a tooltip'" [tooltipPosition]="'top'">
      Hover me
    </button>
  `,
  styles: [`
    ::ng-deep .tooltip {
      position: fixed;
      background: #333;
      color: white;
      padding: 8px 12px;
      border-radius: 4px;
      font-size: 14px;
      z-index: 1000;
    }
  `]
})
export class AppComponent {}
```

### Debounce Directive

```typescript
import { 
  Directive, 
  Input, 
  Output, 
  EventEmitter, 
  OnInit, 
  OnDestroy 
} from '@angular/core';
import { Subject, Subscription } from 'rxjs';
import { debounceTime } from 'rxjs/operators';

@Directive({
  selector: '[appDebounce]',
  standalone: true
})
export class DebounceDirective implements OnInit, OnDestroy {
  @Input() debounceTime = 500;
  @Output() debounceEvent = new EventEmitter<Event>();

  private subject = new Subject<Event>();
  private subscription?: Subscription;

  ngOnInit() {
    this.subscription = this.subject
      .pipe(debounceTime(this.debounceTime))
      .subscribe(event => this.debounceEvent.emit(event));
  }

  @HostListener('input', ['$event'])
  onInput(event: Event) {
    this.subject.next(event);
  }

  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}

// Usage
@Component({
  selector: 'app-search',
  standalone: true,
  imports: [DebounceDirective],
  template: `
    <input 
      type="text" 
      [debounceTime]="300"
      (debounceEvent)="onSearch($event)"
      placeholder="Search..."
    >
  `
})
export class SearchComponent {
  onSearch(event: Event) {
    const value = (event.target as HTMLInputElement).value;
    console.log('Searching for:', value);
  }
}
```

### Lazy Load Image Directive

```typescript
import { 
  Directive, 
  ElementRef, 
  Input, 
  OnInit, 
  OnDestroy 
} from '@angular/core';

@Directive({
  selector: 'img[appLazyLoad]',
  standalone: true
})
export class LazyLoadImageDirective implements OnInit, OnDestroy {
  @Input() appLazyLoad = '';
  @Input() placeholder = 'data:image/svg+xml,...'; // Base64 placeholder

  private observer?: IntersectionObserver;

  constructor(private el: ElementRef<HTMLImageElement>) {}

  ngOnInit() {
    // Set placeholder
    this.el.nativeElement.src = this.placeholder;

    // Create intersection observer
    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          this.loadImage();
          this.observer?.disconnect();
        }
      },
      {
        rootMargin: '50px'
      }
    );

    this.observer.observe(this.el.nativeElement);
  }

  private loadImage() {
    const img = this.el.nativeElement;
    img.src = this.appLazyLoad;
    img.classList.add('lazy-loaded');
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}

// Usage
@Component({
  selector: 'app-gallery',
  standalone: true,
  imports: [LazyLoadImageDirective],
  template: `
    <img 
      [appLazyLoad]="imageSrc" 
      alt="Description"
      class="lazy-image"
    >
  `,
  styles: [`
    .lazy-image {
      opacity: 0;
      transition: opacity 0.3s;
    }
    .lazy-image.lazy-loaded {
      opacity: 1;
    }
  `]
})
export class GalleryComponent {
  imageSrc = 'https://example.com/large-image.jpg';
}
```

## Directive Composition

### Combining Multiple Directives

```typescript
// Base directives
@Directive({
  selector: '[appFocus]',
  standalone: true
})
export class FocusDirective implements AfterViewInit {
  constructor(private el: ElementRef) {}

  ngAfterViewInit() {
    this.el.nativeElement.focus();
  }
}

@Directive({
  selector: '[appValidate]',
  standalone: true
})
export class ValidateDirective {
  @HostBinding('class.invalid') isInvalid = false;

  @HostListener('blur')
  onBlur() {
    const value = this.el.nativeElement.value;
    this.isInvalid = value.length < 3;
  }

  constructor(private el: ElementRef) {}
}

// Usage: Apply multiple directives
@Component({
  selector: 'app-form',
  standalone: true,
  imports: [FocusDirective, ValidateDirective],
  template: `
    <input 
      appFocus 
      appValidate 
      type="text"
    >
  `
})
export class FormComponent {}
```

### Directive Inheritance

```typescript
// Base directive
@Directive()
export abstract class BaseButtonDirective {
  @HostBinding('class.btn') btnClass = true;
  @HostBinding('attr.role') role = 'button';
  @HostBinding('attr.tabindex') tabindex = '0';

  @HostListener('keydown.enter', ['$event'])
  @HostListener('keydown.space', ['$event'])
  onKeyDown(event: KeyboardEvent) {
    event.preventDefault();
    this.onClick();
  }

  abstract onClick(): void;
}

// Extended directive
@Directive({
  selector: '[appPrimaryButton]',
  standalone: true
})
export class PrimaryButtonDirective extends BaseButtonDirective {
  @HostBinding('class.btn-primary') primaryClass = true;
  @Output() buttonClick = new EventEmitter<void>();

  onClick() {
    this.buttonClick.emit();
  }
}
```

## Testing Attribute Directives

### Basic Directive Testing

```typescript
import { Component } from '@angular/core';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HighlightDirective } from './highlight.directive';

@Component({
  standalone: true,
  imports: [HighlightDirective],
  template: `<p appHighlight="yellow">Test</p>`
})
class TestComponent {}

describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestComponent>;
  let element: HTMLElement;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [TestComponent]
    });

    fixture = TestBed.createComponent(TestComponent);
    element = fixture.nativeElement.querySelector('p');
    fixture.detectChanges();
  });

  it('should highlight on hover', () => {
    element.dispatchEvent(new MouseEvent('mouseenter'));
    fixture.detectChanges();
    
    expect(element.style.backgroundColor).toBe('yellow');
  });

  it('should remove highlight on leave', () => {
    element.dispatchEvent(new MouseEvent('mouseenter'));
    element.dispatchEvent(new MouseEvent('mouseleave'));
    fixture.detectChanges();
    
    expect(element.style.backgroundColor).toBe('');
  });
});
```

## Common Mistakes

### 1. Direct DOM Manipulation

```typescript
// BAD: Direct manipulation
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective {
  constructor(private el: ElementRef) {
    this.el.nativeElement.style.color = 'red';
  }
}

// GOOD: Use Renderer2
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective {
  constructor(private el: ElementRef, private renderer: Renderer2) {
    this.renderer.setStyle(this.el.nativeElement, 'color', 'red');
  }
}
```

### 2. Not Cleaning Up Event Listeners

```typescript
// BAD: Memory leak
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective implements OnInit {
  ngOnInit() {
    document.addEventListener('click', () => {});
  }
}

// GOOD: Proper cleanup
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective implements OnInit, OnDestroy {
  private unlisten?: () => void;

  constructor(private renderer: Renderer2) {}

  ngOnInit() {
    this.unlisten = this.renderer.listen('document', 'click', () => {});
  }

  ngOnDestroy() {
    this.unlisten?.();
  }
}
```

### 3. Heavy Computation in HostListener

```typescript
// BAD: Heavy computation on every event
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective {
  @HostListener('mousemove', ['$event'])
  onMouseMove(event: MouseEvent) {
    // Heavy computation on every pixel movement
    this.expensiveCalculation();
  }
}

// GOOD: Throttle or debounce
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective implements OnInit, OnDestroy {
  private subject = new Subject<MouseEvent>();
  private subscription?: Subscription;

  ngOnInit() {
    this.subscription = this.subject
      .pipe(throttleTime(100))
      .subscribe(() => this.expensiveCalculation());
  }

  @HostListener('mousemove', ['$event'])
  onMouseMove(event: MouseEvent) {
    this.subject.next(event);
  }

  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}
```

## Best Practices

### 1. Use Meaningful Selector Names

```typescript
// GOOD: Clear, descriptive names
@Directive({ selector: '[appClickOutside]' })
@Directive({ selector: '[appAutoFocus]' })
@Directive({ selector: '[appLazyLoad]' })
```

### 2. Provide Configuration Options

```typescript
@Directive({
  selector: '[appTooltip]',
  standalone: true
})
export class TooltipDirective {
  @Input() appTooltip = '';
  @Input() tooltipPosition = 'top';
  @Input() tooltipDelay = 300;
  @Input() tooltipTheme = 'dark';
}
```

### 3. Use Type-Safe Event Parameters

```typescript
@Directive({
  selector: '[appKeyboard]',
  standalone: true
})
export class KeyboardDirective {
  @Output() enterPress = new EventEmitter<KeyboardEvent>();
  @Output() escapePress = new EventEmitter<KeyboardEvent>();

  @HostListener('keydown.enter', ['$event'])
  onEnter(event: KeyboardEvent) {
    this.enterPress.emit(event);
  }

  @HostListener('keydown.escape', ['$event'])
  onEscape(event: KeyboardEvent) {
    this.escapePress.emit(event);
  }
}
```

### 4. Document Your Directives

```typescript
/**
 * Highlights the host element on hover
 * 
 * @example
 * <div [appHighlight]="'yellow'">Content</div>
 * 
 * @example
 * <div [appHighlight]="color" [defaultColor]="'gray'">Content</div>
 */
@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {
  /** The highlight color to apply on hover */
  @Input() appHighlight = 'yellow';
  
  /** The default color when not hovering */
  @Input() defaultColor = 'transparent';
}
```

## Interview Questions

### Q1: What's the difference between ElementRef and Renderer2?

**Answer:** ElementRef provides direct access to the DOM element, but manipulating it directly bypasses Angular's abstraction and can cause security issues and SSR problems. Renderer2 is Angular's abstraction layer that safely manipulates the DOM and works in all environments.

### Q2: When should you use @HostListener vs Renderer2.listen?

**Answer:** Use `@HostListener` for simple event handling directly on the host element. Use `Renderer2.listen()` when you need to listen to events on other elements, document, or window, or when you need more control over listener lifecycle.

### Q3: What is the purpose of @HostBinding?

**Answer:** `@HostBinding` binds a directive class property to a host element property, attribute, or style. It's a declarative way to modify the host element without direct DOM manipulation.

### Q4: How do attribute directives differ from structural directives?

**Answer:** Attribute directives change the appearance or behavior of existing elements, while structural directives add or remove elements from the DOM. Attribute directives don't use `*` syntax and don't require TemplateRef/ViewContainerRef.

### Q5: Why is it important to implement OnDestroy in directives?

**Answer:** OnDestroy is crucial for cleanup - removing event listeners, unsubscribing from observables, clearing timers, and disconnecting observers. Without proper cleanup, directives can cause memory leaks.

### Q6: Can you apply multiple directives to the same element?

**Answer:** Yes, you can apply multiple directives to the same element. They'll all be instantiated and their logic will execute. However, be careful about conflicts if they modify the same properties.

### Q7: How do you test attribute directives?

**Answer:** Create a test component that uses the directive, then test the DOM changes or behavior it produces. Use ComponentFixture to access the element and trigger events with dispatchEvent.

### Q8: What's the difference between host metadata and @HostBinding/@HostListener?

**Answer:** The `host` metadata in `@Directive` decorator is static configuration, while `@HostBinding` and `@HostListener` are dynamic and can access component properties and methods. Use host metadata for simple static bindings.

## Key Takeaways

1. **Use Renderer2 for DOM manipulation** - never access nativeElement directly
2. **HostListener and HostBinding are declarative** alternatives to imperative DOM manipulation
3. **Always implement OnDestroy** for cleanup of listeners and subscriptions
4. **Attribute directives enhance existing elements** without changing DOM structure
5. **Type your event parameters** for better type safety and IDE support
6. **Use meaningful selector names** with the `app` prefix for custom directives
7. **Provide configuration through @Input** properties for reusable directives
8. **Test directives through host components** that use them
9. **Be mindful of performance** - throttle/debounce frequent events
10. **Document your directives** with JSDoc comments and usage examples

## Resources

### Official Documentation

- [Angular Attribute Directives](https://angular.dev/guide/directives/attribute-directives)
- [Renderer2 API](https://angular.dev/api/core/Renderer2)
- [ElementRef API](https://angular.dev/api/core/ElementRef)

### Articles

- "Angular Directives Guide" - Angular University
- "Creating Powerful Directives" - Thoughtram Blog
- "Directive Patterns" - Netanel Basal Blog

### Video Tutorials

- "Mastering Angular Directives" - ng-conf
- "Custom Directives Deep Dive" - Angular Connect
- "Building Reusable Directives" - Decoded Frontend

### Tools

- Angular DevTools - Inspect directive instances
- VS Code Angular Language Service - Directive intellisense
- Augury (deprecated but still useful) - Directive debugging
