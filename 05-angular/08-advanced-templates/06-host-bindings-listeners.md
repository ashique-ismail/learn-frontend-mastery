# Host Bindings and Listeners

## The Idea

**In plain English:** Host bindings and listeners are ways for an Angular component or directive to reach out and control the HTML element it lives on — changing its look or reacting when someone clicks, types, or scrolls on it. Think of the "host element" as the outer container tag that holds your component.

**Real-world analogy:** Imagine a hotel room (the host element) and a smart device panel installed on the wall inside it. The panel can change the room's lights and temperature (host bindings), and it can also detect when the door opens or someone presses a button (host listeners).

- The hotel room = the host element (the HTML tag your component sits on)
- Changing the room lights/temperature = @HostBinding (updating properties, styles, or classes on the host element)
- The door sensor / button detector = @HostListener (listening for events like clicks or keypresses on the host element)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding the Host Element](#understanding-the-host-element)
3. [HostBinding Decorator](#hostbinding-decorator)
4. [HostListener Decorator](#hostlistener-decorator)
5. [Host Metadata Configuration](#host-metadata-configuration)
6. [Dynamic Styles and Classes](#dynamic-styles-and-classes)
7. [Advanced Event Handling](#advanced-event-handling)
8. [Combining Host Decorators](#combining-host-decorators)
9. [Performance Optimization](#performance-optimization)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)
13. [Key Takeaways](#key-takeaways)
14. [Resources](#resources)

## Introduction

Host bindings and listeners are Angular's declarative way to interact with the host element of a directive or component. They provide a clean, type-safe alternative to imperative DOM manipulation through `ElementRef` and `Renderer2`. Understanding host bindings and listeners is essential for creating components and directives that properly encapsulate their behavior and integrate seamlessly with Angular's change detection.

This guide covers everything from basic `@HostBinding` and `@HostListener` usage to advanced patterns for dynamic styling, event handling, and performance optimization.

## Understanding the Host Element

The host element is the DOM element that hosts a directive or component. For components, it's the element matching the component's selector. For attribute directives, it's the element the directive is applied to.

### What is a Host Element?

```typescript
// Component example
@Component({
  selector: 'app-card',
  standalone: true,
  template: `<div class="content">Card content</div>`
})
export class CardComponent {}

// Usage: <app-card></app-card>
// Host element: The <app-card> element itself
```

```typescript
// Directive example
@Directive({
  selector: '[appHighlight]',
  standalone: true
})
export class HighlightDirective {}

// Usage: <div appHighlight>Text</div>
// Host element: The <div> element
```

### Accessing Host Element Without Decorators

```typescript
import { Component, ElementRef } from '@angular/core';

@Component({
  selector: 'app-example',
  standalone: true,
  template: `<p>Content</p>`
})
export class ExampleComponent {
  constructor(private elementRef: ElementRef) {
    // Direct access to host element (not recommended for manipulation)
    console.log('Host element:', this.elementRef.nativeElement);
  }
}
```

## HostBinding Decorator

`@HostBinding` binds a class property to a host element property, attribute, or style. It's evaluated during change detection and automatically updates the host element.

### Basic Property Binding

```typescript
import { Component, HostBinding } from '@angular/core';

@Component({
  selector: 'app-button',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class ButtonComponent {
  // Bind to property
  @HostBinding('disabled') isDisabled = true;
  
  // Bind to attribute
  @HostBinding('attr.role') role = 'button';
  
  // Bind to aria attribute
  @HostBinding('attr.aria-label') ariaLabel = 'Click me';
  
  // Bind to data attribute
  @HostBinding('attr.data-testid') testId = 'my-button';
}

// Rendered: <app-button disabled role="button" aria-label="Click me" data-testid="my-button">
```

### Class Binding

```typescript
import { Component, HostBinding, Input } from '@angular/core';

@Component({
  selector: 'app-badge',
  standalone: true,
  template: `<ng-content></ng-content>`,
  styles: [`
    :host.badge { padding: 4px 8px; border-radius: 4px; }
    :host.badge-primary { background: blue; color: white; }
    :host.badge-success { background: green; color: white; }
    :host.badge-danger { background: red; color: white; }
  `]
})
export class BadgeComponent {
  @Input() variant: 'primary' | 'success' | 'danger' = 'primary';
  
  // Single class binding
  @HostBinding('class.badge') badgeClass = true;
  
  // Conditional class binding
  @HostBinding('class.badge-primary') 
  get isPrimary() {
    return this.variant === 'primary';
  }
  
  @HostBinding('class.badge-success') 
  get isSuccess() {
    return this.variant === 'success';
  }
  
  @HostBinding('class.badge-danger') 
  get isDanger() {
    return this.variant === 'danger';
  }
}

// Usage
@Component({
  template: `
    <app-badge variant="success">New</app-badge>
  `
})
export class AppComponent {}
```

### Style Binding

```typescript
import { Component, HostBinding, Input } from '@angular/core';

@Component({
  selector: 'app-progress-bar',
  standalone: true,
  template: `<div class="progress"></div>`,
  styles: [`
    :host {
      display: block;
      width: 100%;
      height: 20px;
      background: #f0f0f0;
      border-radius: 10px;
      overflow: hidden;
    }
    .progress {
      height: 100%;
      transition: width 0.3s ease;
    }
  `]
})
export class ProgressBarComponent {
  @Input() progress = 0;
  @Input() color = '#4caf50';
  
  // Bind individual style properties
  @HostBinding('style.width') width = '100%';
  @HostBinding('style.display') display = 'block';
  
  // Bind with units
  @HostBinding('style.height.px') height = 20;
  @HostBinding('style.borderRadius.px') borderRadius = 10;
  
  // Dynamic style binding
  @HostBinding('style.opacity')
  get opacity() {
    return this.progress === 100 ? 0.7 : 1;
  }
}
```

### Multiple Bindings with Getters

```typescript
import { Directive, HostBinding, Input } from '@angular/core';

@Directive({
  selector: '[appResponsive]',
  standalone: true
})
export class ResponsiveDirective {
  @Input() breakpoint = 768;
  private windowWidth = window.innerWidth;
  
  @HostBinding('class.mobile')
  get isMobile() {
    return this.windowWidth < this.breakpoint;
  }
  
  @HostBinding('class.desktop')
  get isDesktop() {
    return this.windowWidth >= this.breakpoint;
  }
  
  @HostBinding('style.fontSize.px')
  get fontSize() {
    return this.isMobile ? 14 : 16;
  }
  
  @HostListener('window:resize')
  onResize() {
    this.windowWidth = window.innerWidth;
  }
}
```

### Complex Property Bindings

```typescript
import { Component, HostBinding, Input } from '@angular/core';

interface Theme {
  background: string;
  color: string;
  border: string;
}

@Component({
  selector: 'app-themed-card',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class ThemedCardComponent {
  @Input() theme: Theme = {
    background: '#ffffff',
    color: '#000000',
    border: '1px solid #ccc'
  };
  
  @HostBinding('style.backgroundColor')
  get backgroundColor() {
    return this.theme.background;
  }
  
  @HostBinding('style.color')
  get textColor() {
    return this.theme.color;
  }
  
  @HostBinding('style.border')
  get border() {
    return this.theme.border;
  }
  
  @HostBinding('style.padding') padding = '16px';
  @HostBinding('style.borderRadius') borderRadius = '8px';
}
```

## HostListener Decorator

`@HostListener` decorates a method to listen for events on the host element or global targets like window or document.

### Basic Event Listening

```typescript
import { Directive, HostListener } from '@angular/core';

@Directive({
  selector: '[appClickTracker]',
  standalone: true
})
export class ClickTrackerDirective {
  clickCount = 0;
  
  // Listen to click events
  @HostListener('click')
  onClick() {
    this.clickCount++;
    console.log('Clicked:', this.clickCount);
  }
  
  // Listen to double-click
  @HostListener('dblclick')
  onDoubleClick() {
    console.log('Double clicked!');
  }
  
  // Listen to mouse enter
  @HostListener('mouseenter')
  onMouseEnter() {
    console.log('Mouse entered');
  }
  
  // Listen to mouse leave
  @HostListener('mouseleave')
  onMouseLeave() {
    console.log('Mouse left');
  }
}
```

### Event Object Access

```typescript
import { Directive, HostListener } from '@angular/core';

@Directive({
  selector: '[appEventDetails]',
  standalone: true
})
export class EventDetailsDirective {
  // Access full event object
  @HostListener('click', ['$event'])
  onClick(event: MouseEvent) {
    console.log('Click position:', event.clientX, event.clientY);
    console.log('Target:', event.target);
    console.log('Ctrl key:', event.ctrlKey);
  }
  
  // Access specific event properties
  @HostListener('keydown', ['$event.key', '$event.ctrlKey', '$event.shiftKey'])
  onKeyDown(key: string, ctrlKey: boolean, shiftKey: boolean) {
    if (ctrlKey && key === 's') {
      console.log('Ctrl+S pressed');
    }
    if (shiftKey && key === 'Enter') {
      console.log('Shift+Enter pressed');
    }
  }
  
  // Access event target
  @HostListener('input', ['$event.target.value'])
  onInput(value: string) {
    console.log('Input value:', value);
  }
}
```

### Keyboard Event Handling

```typescript
import { Directive, HostListener, Output, EventEmitter } from '@angular/core';

@Directive({
  selector: '[appKeyboardShortcuts]',
  standalone: true
})
export class KeyboardShortcutsDirective {
  @Output() save = new EventEmitter<void>();
  @Output() cancel = new EventEmitter<void>();
  @Output() submit = new EventEmitter<void>();
  
  // Listen to specific keys
  @HostListener('keydown.enter')
  onEnter() {
    this.submit.emit();
  }
  
  @HostListener('keydown.escape')
  onEscape() {
    this.cancel.emit();
  }
  
  // Listen to key combinations
  @HostListener('keydown.control.s', ['$event'])
  @HostListener('keydown.meta.s', ['$event']) // Mac Command key
  onSave(event: KeyboardEvent) {
    event.preventDefault(); // Prevent browser save dialog
    this.save.emit();
  }
  
  // Complex key combinations
  @HostListener('keydown', ['$event'])
  onKeyDown(event: KeyboardEvent) {
    // Ctrl+Shift+K
    if (event.ctrlKey && event.shiftKey && event.key === 'K') {
      event.preventDefault();
      console.log('Custom shortcut triggered');
    }
  }
}

// Usage
@Component({
  template: `
    <div 
      appKeyboardShortcuts
      (save)="onSave()"
      (cancel)="onCancel()"
      (submit)="onSubmit()"
    >
      Press Ctrl+S to save, Esc to cancel, Enter to submit
    </div>
  `
})
export class FormComponent {
  onSave() { console.log('Saving...'); }
  onCancel() { console.log('Cancelling...'); }
  onSubmit() { console.log('Submitting...'); }
}
```

### Window and Document Events

```typescript
import { Directive, HostListener, HostBinding } from '@angular/core';

@Directive({
  selector: '[appScrollAware]',
  standalone: true
})
export class ScrollAwareDirective {
  @HostBinding('class.scrolled') isScrolled = false;
  
  // Listen to window scroll
  @HostListener('window:scroll')
  onWindowScroll() {
    this.isScrolled = window.scrollY > 100;
  }
  
  // Listen to window resize
  @HostListener('window:resize')
  onWindowResize() {
    console.log('Window resized:', window.innerWidth, window.innerHeight);
  }
  
  // Listen to document events
  @HostListener('document:click', ['$event.target'])
  onDocumentClick(target: HTMLElement) {
    console.log('Document clicked on:', target.tagName);
  }
  
  // Listen to before unload
  @HostListener('window:beforeunload', ['$event'])
  onBeforeUnload(event: BeforeUnloadEvent) {
    // Show confirmation dialog
    event.preventDefault();
    event.returnValue = '';
  }
}
```

### Touch and Gesture Events

```typescript
import { Directive, HostListener, Output, EventEmitter } from '@angular/core';

@Directive({
  selector: '[appSwipe]',
  standalone: true
})
export class SwipeDirective {
  @Output() swipeLeft = new EventEmitter<void>();
  @Output() swipeRight = new EventEmitter<void>();
  @Output() swipeUp = new EventEmitter<void>();
  @Output() swipeDown = new EventEmitter<void>();
  
  private touchStartX = 0;
  private touchStartY = 0;
  private minSwipeDistance = 50;
  
  @HostListener('touchstart', ['$event'])
  onTouchStart(event: TouchEvent) {
    this.touchStartX = event.touches[0].clientX;
    this.touchStartY = event.touches[0].clientY;
  }
  
  @HostListener('touchend', ['$event'])
  onTouchEnd(event: TouchEvent) {
    const touchEndX = event.changedTouches[0].clientX;
    const touchEndY = event.changedTouches[0].clientY;
    
    const deltaX = touchEndX - this.touchStartX;
    const deltaY = touchEndY - this.touchStartY;
    
    if (Math.abs(deltaX) > Math.abs(deltaY)) {
      // Horizontal swipe
      if (Math.abs(deltaX) > this.minSwipeDistance) {
        if (deltaX > 0) {
          this.swipeRight.emit();
        } else {
          this.swipeLeft.emit();
        }
      }
    } else {
      // Vertical swipe
      if (Math.abs(deltaY) > this.minSwipeDistance) {
        if (deltaY > 0) {
          this.swipeDown.emit();
        } else {
          this.swipeUp.emit();
        }
      }
    }
  }
}
```

## Host Metadata Configuration

The `host` property in the `@Component` or `@Directive` decorator provides static configuration for host bindings and listeners.

### Static Host Bindings

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-static-host',
  standalone: true,
  template: `<ng-content></ng-content>`,
  host: {
    // Static class
    'class': 'custom-component',
    
    // Static attribute
    'role': 'button',
    'tabindex': '0',
    
    // Static style
    'style': 'display: block; padding: 16px',
    
    // Conditional bindings
    '[class.active]': 'isActive',
    '[class.disabled]': 'isDisabled',
    
    // Property bindings
    '[attr.aria-label]': 'label',
    '[attr.aria-pressed]': 'isPressed',
    
    // Event listeners
    '(click)': 'onClick()',
    '(keydown.enter)': 'onEnter($event)',
    '(keydown.space)': 'onSpace($event)'
  }
})
export class StaticHostComponent {
  isActive = false;
  isDisabled = false;
  isPressed = false;
  label = 'Click me';
  
  onClick() {
    this.isPressed = !this.isPressed;
  }
  
  onEnter(event: KeyboardEvent) {
    event.preventDefault();
    this.onClick();
  }
  
  onSpace(event: KeyboardEvent) {
    event.preventDefault();
    this.onClick();
  }
}
```

### Combining Host Metadata with Decorators

```typescript
import { Component, HostBinding, HostListener, Input } from '@angular/core';

@Component({
  selector: 'app-hybrid',
  standalone: true,
  template: `<ng-content></ng-content>`,
  host: {
    // Static bindings
    'class': 'hybrid-component',
    'role': 'button',
    
    // Call methods
    '(click)': 'handleClick()'
  }
})
export class HybridComponent {
  @Input() type: 'primary' | 'secondary' = 'primary';
  
  // Dynamic bindings via decorator
  @HostBinding('class.primary') 
  get isPrimary() {
    return this.type === 'primary';
  }
  
  @HostBinding('class.secondary')
  get isSecondary() {
    return this.type === 'secondary';
  }
  
  @HostBinding('attr.aria-label') ariaLabel = 'Button';
  
  // Additional event listeners via decorator
  @HostListener('keydown.enter', ['$event'])
  onEnter(event: KeyboardEvent) {
    event.preventDefault();
    this.handleClick();
  }
  
  handleClick() {
    console.log('Clicked!');
  }
}
```

## Dynamic Styles and Classes

### Dynamic Class Management

```typescript
import { Component, HostBinding, Input } from '@angular/core';

@Component({
  selector: 'app-button',
  standalone: true,
  template: `<ng-content></ng-content>`,
  styles: [`
    :host {
      display: inline-block;
      padding: 8px 16px;
      border-radius: 4px;
      cursor: pointer;
      border: none;
      transition: all 0.2s;
    }
    :host.btn-sm { padding: 4px 8px; font-size: 12px; }
    :host.btn-md { padding: 8px 16px; font-size: 14px; }
    :host.btn-lg { padding: 12px 24px; font-size: 16px; }
    :host.btn-primary { background: blue; color: white; }
    :host.btn-secondary { background: gray; color: white; }
    :host.btn-disabled { opacity: 0.5; cursor: not-allowed; }
    :host:hover:not(.btn-disabled) { transform: translateY(-2px); }
  `]
})
export class ButtonComponent {
  @Input() size: 'sm' | 'md' | 'lg' = 'md';
  @Input() variant: 'primary' | 'secondary' = 'primary';
  @Input() disabled = false;
  
  // Dynamic class binding using string interpolation
  @HostBinding('class')
  get hostClasses(): string {
    return `btn-${this.size} btn-${this.variant} ${this.disabled ? 'btn-disabled' : ''}`;
  }
  
  // Alternative: object-based class binding
  @HostBinding('class')
  get hostClassesObject(): Record<string, boolean> {
    return {
      [`btn-${this.size}`]: true,
      [`btn-${this.variant}`]: true,
      'btn-disabled': this.disabled
    };
  }
}
```

### Dynamic Style Calculations

```typescript
import { Component, HostBinding, Input, OnInit } from '@angular/core';

@Component({
  selector: 'app-color-box',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class ColorBoxComponent implements OnInit {
  @Input() hue = 0;
  @Input() saturation = 50;
  @Input() lightness = 50;
  @Input() size = 100;
  
  @HostBinding('style.backgroundColor')
  get backgroundColor(): string {
    return `hsl(${this.hue}, ${this.saturation}%, ${this.lightness}%)`;
  }
  
  @HostBinding('style.width.px')
  @HostBinding('style.height.px')
  get dimensions(): number {
    return this.size;
  }
  
  @HostBinding('style.color')
  get textColor(): string {
    // Calculate contrasting text color
    return this.lightness > 50 ? '#000000' : '#ffffff';
  }
  
  @HostBinding('style.border')
  get border(): string {
    const borderColor = `hsl(${this.hue}, ${this.saturation}%, ${this.lightness - 20}%)`;
    return `2px solid ${borderColor}`;
  }
  
  @HostBinding('style.boxShadow')
  get boxShadow(): string {
    return `0 4px 8px hsla(${this.hue}, ${this.saturation}%, ${this.lightness}%, 0.3)`;
  }
  
  ngOnInit() {
    console.log('Color box initialized:', this.backgroundColor);
  }
}
```

### Animated Style Transitions

```typescript
import { Component, HostBinding, HostListener } from '@angular/core';

@Component({
  selector: 'app-animated-card',
  standalone: true,
  template: `<ng-content></ng-content>`,
  styles: [`
    :host {
      display: block;
      transition: all 0.3s ease-in-out;
    }
  `]
})
export class AnimatedCardComponent {
  private isHovered = false;
  private isPressed = false;
  
  @HostBinding('style.transform')
  get transform(): string {
    if (this.isPressed) {
      return 'scale(0.95) translateY(2px)';
    }
    if (this.isHovered) {
      return 'scale(1.05) translateY(-4px)';
    }
    return 'scale(1) translateY(0)';
  }
  
  @HostBinding('style.boxShadow')
  get boxShadow(): string {
    if (this.isHovered) {
      return '0 8px 16px rgba(0, 0, 0, 0.2)';
    }
    return '0 2px 4px rgba(0, 0, 0, 0.1)';
  }
  
  @HostBinding('style.opacity')
  get opacity(): number {
    return this.isPressed ? 0.9 : 1;
  }
  
  @HostListener('mouseenter')
  onMouseEnter() {
    this.isHovered = true;
  }
  
  @HostListener('mouseleave')
  onMouseLeave() {
    this.isHovered = false;
    this.isPressed = false;
  }
  
  @HostListener('mousedown')
  onMouseDown() {
    this.isPressed = true;
  }
  
  @HostListener('mouseup')
  onMouseUp() {
    this.isPressed = false;
  }
}
```

## Advanced Event Handling

### Event Delegation Pattern

```typescript
import { Directive, HostListener, Output, EventEmitter } from '@angular/core';

@Directive({
  selector: '[appDelegatedClick]',
  standalone: true
})
export class DelegatedClickDirective {
  @Output() itemClick = new EventEmitter<string>();
  
  @HostListener('click', ['$event'])
  onClick(event: MouseEvent) {
    const target = event.target as HTMLElement;
    
    // Find closest parent with data-id
    const clickedElement = target.closest('[data-id]') as HTMLElement;
    
    if (clickedElement) {
      const itemId = clickedElement.getAttribute('data-id');
      if (itemId) {
        this.itemClick.emit(itemId);
      }
    }
  }
}

// Usage
@Component({
  template: `
    <ul appDelegatedClick (itemClick)="onItemClick($event)">
      <li data-id="1">Item 1</li>
      <li data-id="2">Item 2</li>
      <li data-id="3">Item 3</li>
    </ul>
  `
})
export class ListComponent {
  onItemClick(id: string) {
    console.log('Clicked item:', id);
  }
}
```

### Throttled and Debounced Events

```typescript
import { 
  Directive, 
  HostListener, 
  Output, 
  EventEmitter, 
  OnInit, 
  OnDestroy 
} from '@angular/core';
import { Subject, Subscription } from 'rxjs';
import { throttleTime, debounceTime } from 'rxjs/operators';

@Directive({
  selector: '[appThrottledScroll]',
  standalone: true
})
export class ThrottledScrollDirective implements OnInit, OnDestroy {
  @Output() throttledScroll = new EventEmitter<Event>();
  
  private scrollSubject = new Subject<Event>();
  private subscription?: Subscription;
  
  ngOnInit() {
    this.subscription = this.scrollSubject
      .pipe(throttleTime(200))
      .subscribe(event => this.throttledScroll.emit(event));
  }
  
  @HostListener('scroll', ['$event'])
  onScroll(event: Event) {
    this.scrollSubject.next(event);
  }
  
  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}

@Directive({
  selector: '[appDebouncedInput]',
  standalone: true
})
export class DebouncedInputDirective implements OnInit, OnDestroy {
  @Output() debouncedInput = new EventEmitter<string>();
  
  private inputSubject = new Subject<string>();
  private subscription?: Subscription;
  
  ngOnInit() {
    this.subscription = this.inputSubject
      .pipe(debounceTime(300))
      .subscribe(value => this.debouncedInput.emit(value));
  }
  
  @HostListener('input', ['$event.target.value'])
  onInput(value: string) {
    this.inputSubject.next(value);
  }
  
  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}
```

### Long Press Directive

```typescript
import { 
  Directive, 
  HostListener, 
  Output, 
  EventEmitter, 
  OnDestroy 
} from '@angular/core';

@Directive({
  selector: '[appLongPress]',
  standalone: true
})
export class LongPressDirective implements OnDestroy {
  @Output() longPress = new EventEmitter<void>();
  private timeoutId?: number;
  private readonly longPressDelay = 500;
  
  @HostListener('mousedown')
  @HostListener('touchstart')
  onPressStart() {
    this.timeoutId = window.setTimeout(() => {
      this.longPress.emit();
    }, this.longPressDelay);
  }
  
  @HostListener('mouseup')
  @HostListener('mouseleave')
  @HostListener('touchend')
  @HostListener('touchcancel')
  onPressEnd() {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
      this.timeoutId = undefined;
    }
  }
  
  ngOnDestroy() {
    if (this.timeoutId) {
      clearTimeout(this.timeoutId);
    }
  }
}

// Usage
@Component({
  template: `
    <button appLongPress (longPress)="onLongPress()">
      Hold me
    </button>
  `
})
export class ButtonComponent {
  onLongPress() {
    console.log('Button long pressed!');
  }
}
```

## Combining Host Decorators

### Complete Interactive Component

```typescript
import { 
  Component, 
  HostBinding, 
  HostListener, 
  Input, 
  Output, 
  EventEmitter 
} from '@angular/core';

@Component({
  selector: 'app-interactive-card',
  standalone: true,
  template: `
    <div class="card-header">
      <ng-content select="[header]"></ng-content>
    </div>
    <div class="card-body">
      <ng-content></ng-content>
    </div>
  `,
  styles: [`
    :host {
      display: block;
      transition: all 0.3s;
    }
    .card-header { font-weight: bold; margin-bottom: 8px; }
    .card-body { padding: 16px; }
  `]
})
export class InteractiveCardComponent {
  @Input() elevation = 2;
  @Input() hoverable = true;
  @Input() clickable = false;
  @Input() disabled = false;
  
  @Output() cardClick = new EventEmitter<MouseEvent>();
  @Output() cardHover = new EventEmitter<boolean>();
  
  private isHovered = false;
  private isFocused = false;
  
  // Class bindings
  @HostBinding('class.card') cardClass = true;
  @HostBinding('class.card-hoverable') 
  get isHoverable() {
    return this.hoverable && !this.disabled;
  }
  
  @HostBinding('class.card-clickable')
  get isClickable() {
    return this.clickable && !this.disabled;
  }
  
  @HostBinding('class.card-disabled')
  get isDisabled() {
    return this.disabled;
  }
  
  @HostBinding('class.card-focused')
  get cardFocused() {
    return this.isFocused;
  }
  
  // Style bindings
  @HostBinding('style.boxShadow')
  get boxShadow(): string {
    const baseElevation = this.elevation;
    const hoverElevation = this.isHovered && this.hoverable ? 4 : 0;
    const totalElevation = baseElevation + hoverElevation;
    
    return `0 ${totalElevation}px ${totalElevation * 2}px rgba(0, 0, 0, 0.1)`;
  }
  
  @HostBinding('style.transform')
  get transform(): string {
    if (this.isHovered && this.hoverable) {
      return 'translateY(-4px)';
    }
    return 'translateY(0)';
  }
  
  @HostBinding('style.cursor')
  get cursor(): string {
    if (this.disabled) return 'not-allowed';
    if (this.clickable) return 'pointer';
    return 'default';
  }
  
  @HostBinding('style.opacity')
  get opacity(): number {
    return this.disabled ? 0.6 : 1;
  }
  
  // Attribute bindings
  @HostBinding('attr.role')
  get role(): string | null {
    return this.clickable ? 'button' : null;
  }
  
  @HostBinding('attr.tabindex')
  get tabindex(): number | null {
    return this.clickable && !this.disabled ? 0 : null;
  }
  
  @HostBinding('attr.aria-disabled')
  get ariaDisabled(): string {
    return this.disabled.toString();
  }
  
  // Event listeners
  @HostListener('mouseenter')
  onMouseEnter() {
    if (!this.disabled) {
      this.isHovered = true;
      this.cardHover.emit(true);
    }
  }
  
  @HostListener('mouseleave')
  onMouseLeave() {
    this.isHovered = false;
    this.cardHover.emit(false);
  }
  
  @HostListener('click', ['$event'])
  onClick(event: MouseEvent) {
    if (this.clickable && !this.disabled) {
      this.cardClick.emit(event);
    }
  }
  
  @HostListener('keydown.enter', ['$event'])
  @HostListener('keydown.space', ['$event'])
  onKeyPress(event: KeyboardEvent) {
    if (this.clickable && !this.disabled) {
      event.preventDefault();
      this.cardClick.emit(event as any);
    }
  }
  
  @HostListener('focus')
  onFocus() {
    if (!this.disabled) {
      this.isFocused = true;
    }
  }
  
  @HostListener('blur')
  onBlur() {
    this.isFocused = false;
  }
}
```

## Performance Optimization

### Avoiding Frequent Calculations

```typescript
import { Component, HostBinding, Input, OnChanges } from '@angular/core';

@Component({
  selector: 'app-optimized',
  standalone: true,
  template: `<ng-content></ng-content>`
})
export class OptimizedComponent implements OnChanges {
  @Input() data: any;
  
  // Cached computed value
  private cachedClasses = '';
  
  @HostBinding('class')
  get classes(): string {
    return this.cachedClasses;
  }
  
  ngOnChanges() {
    // Expensive computation only when inputs change
    this.cachedClasses = this.computeClasses();
  }
  
  private computeClasses(): string {
    // Expensive computation
    return 'computed-class';
  }
}
```

### Conditional Event Listeners

```typescript
import { 
  Directive, 
  Input, 
  Renderer2, 
  ElementRef, 
  OnChanges, 
  OnDestroy 
} from '@angular/core';

@Directive({
  selector: '[appConditionalListener]',
  standalone: true
})
export class ConditionalListenerDirective implements OnChanges, OnDestroy {
  @Input() enableListener = false;
  private unlistenFn?: () => void;
  
  constructor(
    private renderer: Renderer2,
    private el: ElementRef
  ) {}
  
  ngOnChanges() {
    if (this.enableListener && !this.unlistenFn) {
      // Add listener only when needed
      this.unlistenFn = this.renderer.listen(
        this.el.nativeElement,
        'click',
        () => console.log('Clicked')
      );
    } else if (!this.enableListener && this.unlistenFn) {
      // Remove listener when not needed
      this.unlistenFn();
      this.unlistenFn = undefined;
    }
  }
  
  ngOnDestroy() {
    this.unlistenFn?.();
  }
}
```

## Common Mistakes

### 1. Direct DOM Manipulation in HostListener

```typescript
// BAD: Direct manipulation
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective {
  constructor(private el: ElementRef) {}
  
  @HostListener('click')
  onClick() {
    this.el.nativeElement.style.color = 'red';
  }
}

// GOOD: Use HostBinding
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective {
  @HostBinding('style.color') color = 'black';
  
  @HostListener('click')
  onClick() {
    this.color = 'red';
  }
}
```

### 2. Not Cleaning Up Listeners

```typescript
// BAD: Memory leak
@Directive({ selector: '[appBad]', standalone: true })
export class BadDirective {
  @HostListener('window:scroll')
  onScroll() {
    // This listener is never removed
  }
}

// GOOD: Automatic cleanup
@Directive({ selector: '[appGood]', standalone: true })
export class GoodDirective implements OnDestroy {
  private unlisten?: () => void;
  
  constructor(private renderer: Renderer2) {
    this.unlisten = this.renderer.listen('window', 'scroll', () => {});
  }
  
  ngOnDestroy() {
    this.unlisten?.();
  }
}
```

### 3. Heavy Computation in Getters

```typescript
// BAD: Expensive computation on every check
@Component({ selector: 'app-bad', standalone: true, template: '' })
export class BadComponent {
  @Input() items: any[] = [];
  
  @HostBinding('class.has-items')
  get hasItems() {
    // Heavy computation every change detection cycle
    return this.items.filter(i => i.active).length > 0;
  }
}

// GOOD: Cache the result
@Component({ selector: 'app-good', standalone: true, template: '' })
export class GoodComponent implements OnChanges {
  @Input() items: any[] = [];
  private cachedHasItems = false;
  
  @HostBinding('class.has-items')
  get hasItems() {
    return this.cachedHasItems;
  }
  
  ngOnChanges() {
    this.cachedHasItems = this.items.filter(i => i.active).length > 0;
  }
}
```

## Best Practices

### 1. Prefer HostBinding Over Direct DOM Manipulation

```typescript
// Use HostBinding for declarative bindings
@HostBinding('class.active') isActive = true;
@HostBinding('style.color') textColor = 'blue';
```

### 2. Use Type-Safe Event Parameters

```typescript
@HostListener('click', ['$event'])
onClick(event: MouseEvent) {
  // TypeScript will catch errors
  console.log(event.clientX);
}
```

### 3. Document Complex Bindings

```typescript
/**
 * Dynamic class binding based on component state
 * Combines size, variant, and disabled state
 */
@HostBinding('class')
get hostClasses(): string {
  return `btn-${this.size} btn-${this.variant} ${this.disabled ? 'btn-disabled' : ''}`;
}
```

### 4. Use Host Metadata for Static Bindings

```typescript
@Component({
  selector: 'app-button',
  standalone: true,
  host: {
    'class': 'btn',
    'role': 'button',
    'tabindex': '0'
  }
})
export class ButtonComponent {}
```

## Interview Questions

### Q1: What's the difference between @HostBinding and ElementRef manipulation?

**Answer:** `@HostBinding` is declarative, integrates with change detection, and is type-safe. ElementRef manipulation is imperative, bypasses Angular's abstractions, can cause security issues, and doesn't work with SSR.

### Q2: When does @HostBinding get evaluated?

**Answer:** `@HostBinding` is evaluated during change detection. Getters are called every time change detection runs, which is why expensive computations should be cached.

### Q3: Can you listen to events outside the host element with @HostListener?

**Answer:** Yes, you can listen to window and document events using `@HostListener('window:event')` or `@HostListener('document:event')`.

### Q4: What happens if multiple HostBindings target the same property?

**Answer:** The last binding wins. If multiple directives or the component itself bind to the same property, the last one in the application order will be applied.

### Q5: How do you prevent memory leaks with @HostListener?

**Answer:** @HostListener decorators are automatically cleaned up when the directive/component is destroyed. However, if you manually add listeners using Renderer2, you must call the unlisten function in ngOnDestroy.

### Q6: Can you use @HostBinding with computed values?

**Answer:** Yes, using getters. The getter will be called during change detection to get the current value.

### Q7: What's the performance impact of @HostListener on scroll/mousemove?

**Answer:** High-frequency events can cause performance issues. Use throttling or debouncing with RxJS operators to reduce the frequency of handler execution.

### Q8: How do @HostBinding and ngClass differ?

**Answer:** `@HostBinding` is for directive/component classes and binds to the host element. `ngClass` is for templates and binds to elements within the template. @HostBinding is more efficient for host element bindings.

## Key Takeaways

1. **HostBinding provides declarative property binding** to the host element
2. **HostListener provides declarative event handling** with automatic cleanup
3. **Use host metadata for static bindings** that don't change
4. **Prefer decorators over direct DOM manipulation** for better SSR and security
5. **Cache expensive computations** in getters to avoid performance issues
6. **Window and document events** can be listened to with special syntax
7. **Getters in HostBinding** are called on every change detection cycle
8. **Clean up manual listeners** in ngOnDestroy to prevent memory leaks
9. **Throttle/debounce high-frequency events** to maintain performance
10. **Type event parameters** for better type safety and IDE support

## Resources

### Official Documentation

- [Host Element Interactions](https://angular.dev/guide/components/host-elements)
- [HostBinding API](https://angular.dev/api/core/HostBinding)
- [HostListener API](https://angular.dev/api/core/HostListener)

### Articles

- "Angular Host Bindings Deep Dive" - Angular University
- "Optimizing Host Listeners" - Netanel Basal
- "Host Element Best Practices" - Thoughtram

### Video Tutorials

- "Mastering Host Bindings" - ng-conf
- "Advanced Directive Patterns" - Angular Connect
- "Performance with Host Listeners" - Angular Denver

### Tools

- Angular DevTools - Inspect host bindings
- Chrome DevTools - Profile event listener performance
- VS Code Angular Language Service - Host decorator intellisense
