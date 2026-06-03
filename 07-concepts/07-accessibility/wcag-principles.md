# WCAG Principles and Conformance Levels

## Learning Objectives
- Understand the four foundational POUR principles of web accessibility
- Master WCAG 2.2 guidelines and success criteria
- Differentiate between conformance levels A, AA, and AAA
- Apply WCAG principles to real-world web development
- Evaluate web applications for accessibility compliance

## Introduction to WCAG

The Web Content Accessibility Guidelines (WCAG) are the international standard for web accessibility, developed by the World Wide Web Consortium (W3C) Web Accessibility Initiative (WAI). WCAG provides a comprehensive framework for making web content accessible to people with disabilities, including visual, auditory, physical, speech, cognitive, language, learning, and neurological disabilities.

WCAG 2.2, published in October 2023, builds upon WCAG 2.1 and 2.0, adding nine new success criteria focused on mobile accessibility, cognitive accessibility, and assistive technology support. The guidelines are organized around four foundational principles, known by the acronym POUR: Perceivable, Operable, Understandable, and Robust.

Understanding WCAG is not just about legal compliance—it's about creating inclusive digital experiences that work for everyone, regardless of ability. An estimated 15% of the world's population has some form of disability, representing over one billion people. Making your applications accessible expands your user base, improves usability for everyone, and often leads to cleaner, more maintainable code.

## The Four POUR Principles

### 1. Perceivable

Information and user interface components must be presentable to users in ways they can perceive. This means users must be able to perceive the information being presented—it can't be invisible to all of their senses.

**Key Guidelines:**
- **Text Alternatives**: Provide text alternatives for non-text content (images, videos, audio)
- **Time-based Media**: Provide alternatives for time-based media (captions, transcripts, audio descriptions)
- **Adaptable**: Create content that can be presented in different ways without losing information or structure
- **Distinguishable**: Make it easier for users to see and hear content, including separating foreground from background

```typescript
// React: Perceivable Image Example
interface ProductImageProps {
  src: string;
  alt: string;
  productName: string;
}

const ProductImage: React.FC<ProductImageProps> = ({ src, alt, productName }) => {
  return (
    <figure>
      <img 
        src={src} 
        alt={alt} // Descriptive alt text
        loading="lazy"
      />
      <figcaption className="sr-only">
        {productName} product image
      </figcaption>
    </figure>
  );
};

// Usage
<ProductImage 
  src="/products/laptop.jpg"
  alt="Silver laptop with 15-inch display, open at 90 degrees showing keyboard and screen"
  productName="UltraBook Pro 15"
/>
```

```typescript
// Angular: Perceivable Video with Captions
@Component({
  selector: 'app-accessible-video',
  template: `
    <div class="video-container">
      <video 
        #videoPlayer
        [attr.aria-label]="videoTitle"
        controls
        [poster]="posterImage">
        <source [src]="videoSrc" type="video/mp4">
        <track 
          kind="captions" 
          [src]="captionsSrc" 
          srclang="en" 
          label="English"
          default>
        <track 
          kind="descriptions" 
          [src]="descriptionsSrc" 
          srclang="en" 
          label="Audio descriptions">
        Your browser doesn't support HTML5 video.
      </video>
      <p class="video-description">{{ description }}</p>
    </div>
  `,
  styles: [`
    .video-container {
      position: relative;
      max-width: 100%;
    }
    
    video {
      width: 100%;
      height: auto;
    }
    
    .video-description {
      margin-top: 1rem;
      color: var(--text-secondary);
    }
  `]
})
export class AccessibleVideoComponent {
  @Input() videoSrc!: string;
  @Input() captionsSrc!: string;
  @Input() descriptionsSrc!: string;
  @Input() videoTitle!: string;
  @Input() description!: string;
  @Input() posterImage!: string;
}
```

```css
/* Distinguishable: High contrast and clear visual separation */
:root {
  /* WCAG AA compliant color contrast ratios (4.5:1 for normal text) */
  --background: #ffffff;
  --text-primary: #1a1a1a;
  --text-secondary: #4a4a4a;
  --link-color: #0066cc;
  --link-visited: #551a8b;
  --focus-outline: #005fcc;
}

/* Focus indicators for keyboard navigation */
a:focus, button:focus, input:focus, select:focus, textarea:focus {
  outline: 3px solid var(--focus-outline);
  outline-offset: 2px;
}

/* Ensure sufficient color contrast */
.button-primary {
  background: #0066cc; /* 4.5:1 contrast against white */
  color: #ffffff; /* 13:1 contrast */
  border: none;
  padding: 0.75rem 1.5rem;
  font-size: 1rem;
  font-weight: 600;
}

/* Don't rely on color alone */
.status-success {
  color: #0a6e0a;
  background: #e6f4e6;
  border-left: 4px solid #0a6e0a; /* Visual indicator beyond color */
}

.status-success::before {
  content: '✓ '; /* Icon provides additional context */
}
```

### 2. Operable

User interface components and navigation must be operable. This means users must be able to operate the interface—the interface cannot require interaction that a user cannot perform.

**Key Guidelines:**
- **Keyboard Accessible**: Make all functionality available from a keyboard
- **Enough Time**: Provide users enough time to read and use content
- **Seizures and Physical Reactions**: Don't design content that causes seizures or physical reactions
- **Navigable**: Provide ways to help users navigate, find content, and determine where they are
- **Input Modalities**: Make it easier for users to operate functionality through various inputs beyond keyboard

```typescript
// React: Operable Modal Dialog
import React, { useEffect, useRef, useState } from 'react';

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
  const closeButtonRef = useRef<HTMLButtonElement>(null);
  const [previousActiveElement, setPreviousActiveElement] = 
    useState<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Store previously focused element
      setPreviousActiveElement(document.activeElement as HTMLElement);
      
      // Focus first focusable element (close button)
      closeButtonRef.current?.focus();
      
      // Trap focus within modal
      const handleTabKey = (e: KeyboardEvent) => {
        if (e.key !== 'Tab') return;
        
        const focusableElements = modalRef.current?.querySelectorAll(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        );
        
        if (!focusableElements || focusableElements.length === 0) return;
        
        const firstElement = focusableElements[0] as HTMLElement;
        const lastElement = focusableElements[
          focusableElements.length - 1
        ] as HTMLElement;
        
        if (e.shiftKey && document.activeElement === firstElement) {
          lastElement.focus();
          e.preventDefault();
        } else if (!e.shiftKey && document.activeElement === lastElement) {
          firstElement.focus();
          e.preventDefault();
        }
      };
      
      // Close on Escape key
      const handleEscapeKey = (e: KeyboardEvent) => {
        if (e.key === 'Escape') {
          onClose();
        }
      };
      
      document.addEventListener('keydown', handleTabKey);
      document.addEventListener('keydown', handleEscapeKey);
      
      return () => {
        document.removeEventListener('keydown', handleTabKey);
        document.removeEventListener('keydown', handleEscapeKey);
      };
    } else {
      // Restore focus when modal closes
      previousActiveElement?.focus();
    }
  }, [isOpen, onClose, previousActiveElement]);

  if (!isOpen) return null;

  return (
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
      >
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            ref={closeButtonRef}
            onClick={onClose}
            aria-label="Close dialog"
            className="modal-close"
          >
            ×
          </button>
        </div>
        <div className="modal-content">
          {children}
        </div>
      </div>
    </div>
  );
};

// Usage
function App() {
  const [isModalOpen, setIsModalOpen] = useState(false);

  return (
    <>
      <button onClick={() => setIsModalOpen(true)}>
        Open Settings
      </button>
      
      <AccessibleModal
        isOpen={isModalOpen}
        onClose={() => setIsModalOpen(false)}
        title="Application Settings"
      >
        <form>
          <label htmlFor="username">Username</label>
          <input id="username" type="text" />
          <button type="submit">Save</button>
        </form>
      </AccessibleModal>
    </>
  );
}
```

```typescript
// Angular: Operable Dropdown Menu
import { Component, ElementRef, HostListener, ViewChild } from '@angular/core';

interface MenuItem {
  label: string;
  action: () => void;
  disabled?: boolean;
}

@Component({
  selector: 'app-accessible-dropdown',
  template: `
    <div class="dropdown" #dropdownContainer>
      <button
        #triggerButton
        type="button"
        [attr.aria-expanded]="isOpen"
        [attr.aria-haspopup]="true"
        aria-controls="dropdown-menu"
        (click)="toggleDropdown()"
        (keydown)="handleTriggerKeydown($event)"
        class="dropdown-trigger"
      >
        {{ label }}
        <span aria-hidden="true">▼</span>
      </button>
      
      <ul
        #menuList
        *ngIf="isOpen"
        id="dropdown-menu"
        role="menu"
        [attr.aria-labelledby]="'dropdown-trigger'"
        class="dropdown-menu"
      >
        <li 
          *ngFor="let item of items; let i = index"
          role="none"
        >
          <button
            role="menuitem"
            [attr.tabindex]="isOpen ? 0 : -1"
            [disabled]="item.disabled"
            (click)="selectItem(item)"
            (keydown)="handleMenuKeydown($event, i)"
            class="dropdown-item"
          >
            {{ item.label }}
          </button>
        </li>
      </ul>
    </div>
  `,
  styles: [`
    .dropdown {
      position: relative;
      display: inline-block;
    }
    
    .dropdown-trigger {
      padding: 0.5rem 1rem;
      background: white;
      border: 1px solid #ccc;
      cursor: pointer;
    }
    
    .dropdown-menu {
      position: absolute;
      top: 100%;
      left: 0;
      min-width: 200px;
      background: white;
      border: 1px solid #ccc;
      list-style: none;
      padding: 0;
      margin: 0;
      box-shadow: 0 2px 8px rgba(0,0,0,0.15);
    }
    
    .dropdown-item {
      width: 100%;
      padding: 0.5rem 1rem;
      border: none;
      background: none;
      text-align: left;
      cursor: pointer;
    }
    
    .dropdown-item:hover:not(:disabled) {
      background: #f0f0f0;
    }
    
    .dropdown-item:focus {
      outline: 2px solid #005fcc;
      outline-offset: -2px;
      background: #e6f2ff;
    }
    
    .dropdown-item:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  `]
})
export class AccessibleDropdownComponent {
  @Input() label: string = 'Menu';
  @Input() items: MenuItem[] = [];
  @ViewChild('triggerButton') triggerButton!: ElementRef<HTMLButtonElement>;
  @ViewChild('menuList') menuList!: ElementRef<HTMLUListElement>;
  
  isOpen = false;
  currentIndex = -1;

  toggleDropdown(): void {
    this.isOpen = !this.isOpen;
    if (this.isOpen) {
      setTimeout(() => this.focusFirstItem(), 0);
    }
  }

  selectItem(item: MenuItem): void {
    if (!item.disabled) {
      item.action();
      this.closeDropdown();
    }
  }

  closeDropdown(): void {
    this.isOpen = false;
    this.currentIndex = -1;
    this.triggerButton.nativeElement.focus();
  }

  handleTriggerKeydown(event: KeyboardEvent): void {
    switch (event.key) {
      case 'ArrowDown':
      case 'ArrowUp':
        event.preventDefault();
        this.isOpen = true;
        setTimeout(() => this.focusFirstItem(), 0);
        break;
      case 'Escape':
        this.closeDropdown();
        break;
    }
  }

  handleMenuKeydown(event: KeyboardEvent, index: number): void {
    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.focusNextItem(index);
        break;
      case 'ArrowUp':
        event.preventDefault();
        this.focusPreviousItem(index);
        break;
      case 'Home':
        event.preventDefault();
        this.focusFirstItem();
        break;
      case 'End':
        event.preventDefault();
        this.focusLastItem();
        break;
      case 'Escape':
        event.preventDefault();
        this.closeDropdown();
        break;
      case 'Tab':
        this.closeDropdown();
        break;
    }
  }

  private focusFirstItem(): void {
    this.focusItem(0);
  }

  private focusLastItem(): void {
    this.focusItem(this.items.length - 1);
  }

  private focusNextItem(currentIndex: number): void {
    const nextIndex = (currentIndex + 1) % this.items.length;
    this.focusItem(nextIndex);
  }

  private focusPreviousItem(currentIndex: number): void {
    const prevIndex = currentIndex === 0 ? this.items.length - 1 : currentIndex - 1;
    this.focusItem(prevIndex);
  }

  private focusItem(index: number): void {
    const items = this.menuList.nativeElement.querySelectorAll('button');
    items[index]?.focus();
    this.currentIndex = index;
  }

  @HostListener('document:click', ['$event'])
  onDocumentClick(event: MouseEvent): void {
    const target = event.target as HTMLElement;
    if (!this.triggerButton.nativeElement.contains(target) && 
        (!this.menuList || !this.menuList.nativeElement.contains(target))) {
      this.closeDropdown();
    }
  }
}
```

### 3. Understandable

Information and the operation of user interface must be understandable. This means users must be able to understand the information as well as the operation of the user interface.

**Key Guidelines:**
- **Readable**: Make text content readable and understandable
- **Predictable**: Make web pages appear and operate in predictable ways
- **Input Assistance**: Help users avoid and correct mistakes

```typescript
// React: Understandable Form with Validation
import React, { useState } from 'react';

interface FormErrors {
  email?: string;
  password?: string;
  confirmPassword?: string;
}

const RegistrationForm: React.FC = () => {
  const [formData, setFormData] = useState({
    email: '',
    password: '',
    confirmPassword: ''
  });
  
  const [errors, setErrors] = useState<FormErrors>({});
  const [touched, setTouched] = useState<Set<string>>(new Set());

  const validateEmail = (email: string): string | undefined => {
    if (!email) {
      return 'Email is required';
    }
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      return 'Please enter a valid email address (e.g., user@example.com)';
    }
    return undefined;
  };

  const validatePassword = (password: string): string | undefined => {
    if (!password) {
      return 'Password is required';
    }
    if (password.length < 8) {
      return 'Password must be at least 8 characters long';
    }
    if (!/[A-Z]/.test(password)) {
      return 'Password must contain at least one uppercase letter';
    }
    if (!/[0-9]/.test(password)) {
      return 'Password must contain at least one number';
    }
    return undefined;
  };

  const validateConfirmPassword = (
    password: string, 
    confirmPassword: string
  ): string | undefined => {
    if (!confirmPassword) {
      return 'Please confirm your password';
    }
    if (password !== confirmPassword) {
      return 'Passwords do not match';
    }
    return undefined;
  };

  const handleBlur = (field: string) => {
    setTouched(prev => new Set(prev).add(field));
    validateField(field);
  };

  const validateField = (field: string) => {
    const newErrors: FormErrors = { ...errors };
    
    switch (field) {
      case 'email':
        newErrors.email = validateEmail(formData.email);
        break;
      case 'password':
        newErrors.password = validatePassword(formData.password);
        if (touched.has('confirmPassword')) {
          newErrors.confirmPassword = validateConfirmPassword(
            formData.password, 
            formData.confirmPassword
          );
        }
        break;
      case 'confirmPassword':
        newErrors.confirmPassword = validateConfirmPassword(
          formData.password, 
          formData.confirmPassword
        );
        break;
    }
    
    setErrors(newErrors);
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    
    // Validate all fields
    const newErrors: FormErrors = {
      email: validateEmail(formData.email),
      password: validatePassword(formData.password),
      confirmPassword: validateConfirmPassword(
        formData.password, 
        formData.confirmPassword
      )
    };
    
    setErrors(newErrors);
    setTouched(new Set(['email', 'password', 'confirmPassword']));
    
    const hasErrors = Object.values(newErrors).some(error => error !== undefined);
    
    if (!hasErrors) {
      // Submit form
      console.log('Form submitted successfully');
    } else {
      // Focus first error field
      const firstErrorField = Object.keys(newErrors).find(
        key => newErrors[key as keyof FormErrors]
      );
      document.getElementById(firstErrorField || '')?.focus();
    }
  };

  const getFieldProps = (field: keyof typeof formData) => ({
    id: field,
    name: field,
    value: formData[field],
    onChange: (e: React.ChangeEvent<HTMLInputElement>) => {
      setFormData({ ...formData, [field]: e.target.value });
    },
    onBlur: () => handleBlur(field),
    'aria-invalid': touched.has(field) && !!errors[field],
    'aria-describedby': `${field}-error ${field}-hint`,
    'aria-required': 'true'
  });

  return (
    <form onSubmit={handleSubmit} noValidate>
      <h1>Create Account</h1>
      
      {/* Clear instructions */}
      <p className="form-description">
        All fields are required. Password must be at least 8 characters 
        and include uppercase letters and numbers.
      </p>
      
      {/* Email field */}
      <div className="form-field">
        <label htmlFor="email">
          Email Address
          <span aria-label="required">*</span>
        </label>
        <input
          type="email"
          {...getFieldProps('email')}
          autoComplete="email"
        />
        <div id="email-hint" className="field-hint">
          We'll use this to send you account notifications
        </div>
        {touched.has('email') && errors.email && (
          <div id="email-error" className="error-message" role="alert">
            {errors.email}
          </div>
        )}
      </div>
      
      {/* Password field */}
      <div className="form-field">
        <label htmlFor="password">
          Password
          <span aria-label="required">*</span>
        </label>
        <input
          type="password"
          {...getFieldProps('password')}
          autoComplete="new-password"
        />
        <div id="password-hint" className="field-hint">
          Must be at least 8 characters with uppercase and numbers
        </div>
        {touched.has('password') && errors.password && (
          <div id="password-error" className="error-message" role="alert">
            {errors.password}
          </div>
        )}
      </div>
      
      {/* Confirm password field */}
      <div className="form-field">
        <label htmlFor="confirmPassword">
          Confirm Password
          <span aria-label="required">*</span>
        </label>
        <input
          type="password"
          {...getFieldProps('confirmPassword')}
          autoComplete="new-password"
        />
        <div id="confirmPassword-hint" className="field-hint">
          Re-enter your password to confirm
        </div>
        {touched.has('confirmPassword') && errors.confirmPassword && (
          <div id="confirmPassword-error" className="error-message" role="alert">
            {errors.confirmPassword}
          </div>
        )}
      </div>
      
      <button type="submit" className="submit-button">
        Create Account
      </button>
    </form>
  );
};
```

### 4. Robust

Content must be robust enough that it can be interpreted reliably by a wide variety of user agents, including assistive technologies. This means users must be able to access the content as technologies advance.

**Key Guidelines:**
- **Compatible**: Maximize compatibility with current and future user agents, including assistive technologies
- **Valid HTML**: Use proper HTML structure and semantics
- **Progressive Enhancement**: Ensure baseline functionality works across all browsers

```typescript
// React: Robust Component with Progressive Enhancement
import React, { useState, useEffect } from 'react';

interface NotificationProps {
  message: string;
  type: 'success' | 'error' | 'warning' | 'info';
  duration?: number;
  onClose?: () => void;
}

const AccessibleNotification: React.FC<NotificationProps> = ({
  message,
  type,
  duration = 5000,
  onClose
}) => {
  const [isVisible, setIsVisible] = useState(true);
  const [supportsAnimation, setSupportsAnimation] = useState(true);

  useEffect(() => {
    // Feature detection for animations
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
    setSupportsAnimation(!mediaQuery.matches);

    // Auto-hide notification
    if (duration > 0) {
      const timer = setTimeout(() => {
        handleClose();
      }, duration);

      return () => clearTimeout(timer);
    }
  }, [duration]);

  const handleClose = () => {
    setIsVisible(false);
    onClose?.();
  };

  if (!isVisible) return null;

  const ariaLive = type === 'error' ? 'assertive' : 'polite';
  const roleAttr = type === 'error' ? 'alert' : 'status';

  return (
    <div
      className={`notification notification-${type} ${
        supportsAnimation ? 'animated' : ''
      }`}
      role={roleAttr}
      aria-live={ariaLive}
      aria-atomic="true"
    >
      <div className="notification-content">
        {/* Icon with proper alternative text */}
        <span className="notification-icon" aria-hidden="true">
          {type === 'success' && '✓'}
          {type === 'error' && '✕'}
          {type === 'warning' && '⚠'}
          {type === 'info' && 'ℹ'}
        </span>
        
        <span className="notification-message">{message}</span>
      </div>
      
      <button
        onClick={handleClose}
        aria-label="Close notification"
        className="notification-close"
      >
        ×
      </button>
    </div>
  );
};

// Progressive enhancement example with feature detection
const FeatureDetectionExample: React.FC = () => {
  const [features, setFeatures] = useState({
    hasLocalStorage: false,
    hasServiceWorker: false,
    hasIntersectionObserver: false
  });

  useEffect(() => {
    // Detect features and provide fallbacks
    setFeatures({
      hasLocalStorage: typeof window !== 'undefined' && 'localStorage' in window,
      hasServiceWorker: typeof window !== 'undefined' && 'serviceWorker' in navigator,
      hasIntersectionObserver: typeof window !== 'undefined' && 
        'IntersectionObserver' in window
    });
  }, []);

  const saveData = (key: string, value: string) => {
    if (features.hasLocalStorage) {
      try {
        localStorage.setItem(key, value);
      } catch (e) {
        // Fallback to session storage or cookies
        console.warn('localStorage not available, using fallback');
      }
    } else {
      // Alternative storage mechanism
      console.warn('Using alternative storage');
    }
  };

  return (
    <div>
      <h2>Feature Support Status</h2>
      <ul>
        <li>
          Local Storage: {features.hasLocalStorage ? '✓ Supported' : '✕ Not available'}
        </li>
        <li>
          Service Worker: {features.hasServiceWorker ? '✓ Supported' : '✕ Not available'}
        </li>
        <li>
          Intersection Observer: {features.hasIntersectionObserver ? 
            '✓ Supported' : '✕ Using fallback'}
        </li>
      </ul>
    </div>
  );
};
```

## WCAG Conformance Levels

WCAG defines three levels of conformance: A (minimum), AA (mid-range), and AAA (highest). Each level includes all the requirements from the previous levels.

```
Conformance Level Structure:
┌─────────────────────────────────────┐
│           Level AAA                  │
│  (Enhanced - Most Stringent)        │
│  ┌───────────────────────────────┐  │
│  │        Level AA               │  │
│  │  (Mid-range - Recommended)    │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │      Level A            │  │  │
│  │  │  (Minimum - Required)   │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Level A (Minimum Conformance)

Level A includes the most basic web accessibility features that must be in place. Without Level A compliance, some users will find it impossible to use your content.

**Key Level A Criteria:**
- Provide text alternatives for images (1.1.1)
- Provide captions for pre-recorded audio/video (1.2.1, 1.2.2)
- Content can be presented in different ways (1.3.1, 1.3.2, 1.3.3)
- Use color plus another indicator (1.4.1)
- Keyboard accessible (2.1.1, 2.1.2)
- No keyboard traps (2.1.2)
- Adjustable timing (2.2.1)
- No content flashes more than 3 times per second (2.3.1)
- Provide page titles (2.4.2)
- Logical focus order (2.4.3)
- Link purpose from context (2.4.4)
- Identify input purpose (1.3.5)
- Valid HTML (4.1.1)
- Name, role, value for components (4.1.2)

```typescript
// Level A compliance example
const LevelACompliantImage: React.FC = () => {
  return (
    <div>
      {/* 1.1.1: Non-text Content - Text alternative */}
      <img 
        src="/chart.png" 
        alt="Bar chart showing 40% increase in sales from Q1 to Q2 2024"
      />
      
      {/* 1.4.1: Use of Color - Don't rely on color alone */}
      <div className="status">
        <span className="success-icon" aria-hidden="true">✓</span>
        <span style={{ color: 'green' }}>Success</span>
      </div>
      
      {/* 2.4.2: Page Titled */}
      {/* (This would be in your document head) */}
      <title>Dashboard - Analytics App</title>
      
      {/* 2.4.4: Link Purpose - Clear link text */}
      <a href="/reports">View detailed sales reports</a>
      {/* NOT: <a href="/reports">Click here</a> */}
      
      {/* 4.1.2: Name, Role, Value */}
      <button 
        type="button"
        aria-label="Close dialog"
        aria-pressed="false"
      >
        ×
      </button>
    </div>
  );
};
```

### Level AA (Mid-Range Conformance)

Level AA is the recommended target for most websites and is often the legal requirement for government and public sector sites. It addresses the biggest and most common barriers for disabled users.

**Key Level AA Criteria (in addition to Level A):**
- Captions for live audio (1.2.4)
- Audio description for video (1.2.5)
- Contrast ratio of at least 4.5:1 (1.4.3)
- Text can be resized to 200% (1.4.4)
- Images of text avoided (1.4.5)
- Multiple ways to find pages (2.4.5)
- Headings and labels descriptive (2.4.6)
- Visible keyboard focus (2.4.7)
- Consistent navigation (3.2.3)
- Consistent identification (3.2.4)
- Error suggestions provided (3.3.3)
- Error prevention for legal/financial data (3.3.4)
- Status messages (4.1.3)

```typescript
// Level AA compliance example
const LevelAACompliantForm: React.FC = () => {
  const [email, setEmail] = useState('');
  const [error, setError] = useState('');
  const [status, setStatus] = useState('');

  const validateEmail = (value: string) => {
    if (!value) {
      return 'Email is required';
    }
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      // 3.3.3: Error Suggestion - Provide helpful guidance
      return 'Please enter a valid email address (e.g., name@example.com)';
    }
    return '';
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const validationError = validateEmail(email);
    
    if (validationError) {
      setError(validationError);
      setStatus('');
    } else {
      setError('');
      setStatus('Email saved successfully');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* 2.4.6: Headings and Labels - Descriptive */}
      <h1>Update Email Preferences</h1>
      
      <div className="form-field">
        <label htmlFor="email-input">
          Email Address
        </label>
        <input
          id="email-input"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          aria-invalid={!!error}
          aria-describedby={error ? 'email-error' : undefined}
          // 1.4.3: Contrast - Ensure 4.5:1 ratio in CSS
          style={{
            border: error ? '2px solid #c00' : '1px solid #666',
            fontSize: '1rem' // 1.4.4: Text can resize
          }}
        />
        
        {/* Error message with proper semantics */}
        {error && (
          <div 
            id="email-error" 
            className="error-message"
            role="alert" // 4.1.3: Status Messages
            style={{
              color: '#c00',
              background: '#fee',
              border: '1px solid #c00',
              padding: '0.5rem'
            }}
          >
            {error}
          </div>
        )}
        
        {/* Success message */}
        {status && (
          <div 
            className="success-message"
            role="status" // 4.1.3: Status Messages
            aria-live="polite"
            style={{
              color: '#0a6e0a',
              background: '#e6f4e6',
              border: '1px solid #0a6e0a',
              padding: '0.5rem'
            }}
          >
            {status}
          </div>
        )}
      </div>
      
      <button 
        type="submit"
        // 2.4.7: Focus Visible - Ensure visible focus in CSS
        style={{
          background: '#0066cc', // 4.5:1 contrast against white
          color: '#fff',
          padding: '0.75rem 1.5rem',
          border: 'none',
          fontSize: '1rem'
        }}
      >
        Save Email
      </button>
    </form>
  );
};
```

```css
/* Level AA CSS requirements */
:root {
  /* 1.4.3: Contrast Minimum - 4.5:1 for normal text, 3:1 for large text */
  --text-color: #1a1a1a; /* 14.5:1 against white */
  --link-color: #0066cc; /* 4.5:1 against white */
  --background: #ffffff;
}

/* 2.4.7: Focus Visible - Clear focus indicator */
*:focus {
  outline: 3px solid #005fcc;
  outline-offset: 2px;
}

/* 1.4.4: Resize Text - Support 200% zoom without loss of functionality */
html {
  font-size: 100%; /* User can adjust browser text size */
}

body {
  font-size: 1rem; /* Relative units for scalability */
  line-height: 1.5;
}

/* 1.4.5: Images of Text - Avoid when possible, use real text */
.heading {
  font-family: 'Roboto', sans-serif;
  font-size: 2rem;
  /* Better than an image of text */
}
```

### Level AAA (Enhanced Conformance)

Level AAA is the highest level of accessibility conformance. It's not recommended to require full AAA conformance for entire sites, as some content cannot meet all AAA criteria. However, individual AAA criteria can be targeted for specific content types.

**Key Level AAA Criteria (in addition to A and AA):**
- Sign language interpretation for pre-recorded audio (1.2.6)
- Extended audio description (1.2.7)
- Media alternative for video (1.2.8)
- Audio-only alternative for video (1.2.9)
- Contrast ratio of at least 7:1 (1.4.6)
- No background audio or can be turned off (1.4.7)
- Visual presentation customizable (1.4.8)
- Reading level lower secondary education or supplemental content (3.1.5)
- Context-sensitive help available (3.3.5)
- Error prevention for all user input (3.3.6)

```typescript
// Level AAA enhanced features
const LevelAAAEnhancedArticle: React.FC = () => {
  const [textSize, setTextSize] = useState(100);
  const [lineSpacing, setLineSpacing] = useState(1.5);
  const [readingMode, setReadingMode] = useState(false);

  return (
    <div className="enhanced-article">
      {/* 1.4.8: Visual Presentation - Customizable */}
      <div className="reading-controls" role="region" aria-label="Reading preferences">
        <fieldset>
          <legend>Customize Reading Experience</legend>
          
          <div>
            <label htmlFor="text-size">Text Size: {textSize}%</label>
            <input
              id="text-size"
              type="range"
              min="100"
              max="200"
              value={textSize}
              onChange={(e) => setTextSize(Number(e.target.value))}
              aria-valuemin={100}
              aria-valuemax={200}
              aria-valuenow={textSize}
            />
          </div>
          
          <div>
            <label htmlFor="line-spacing">Line Spacing: {lineSpacing}</label>
            <input
              id="line-spacing"
              type="range"
              min="1.5"
              max="3"
              step="0.1"
              value={lineSpacing}
              onChange={(e) => setLineSpacing(Number(e.target.value))}
            />
          </div>
          
          <div>
            <label>
              <input
                type="checkbox"
                checked={readingMode}
                onChange={(e) => setReadingMode(e.target.checked)}
              />
              Simplified Reading Mode
            </label>
          </div>
        </fieldset>
      </div>
      
      {/* Content with customizable presentation */}
      <article
        style={{
          fontSize: `${textSize}%`,
          lineHeight: lineSpacing,
          maxWidth: readingMode ? '60ch' : '100%', // 1.4.8: Line length
          color: '#000', // 1.4.6: 7:1 contrast
          background: '#fff'
        }}
      >
        <h1>Article Title</h1>
        
        {/* 3.1.5: Reading Level - Provide explanation for complex terms */}
        <p>
          This article discusses <dfn>accessibility</dfn> 
          <span className="definition-tooltip">
            (the practice of making websites usable by everyone)
          </span>.
        </p>
        
        {/* 3.3.5: Context-sensitive help */}
        <form>
          <div className="form-field">
            <label htmlFor="ssn">
              Social Security Number
              <button
                type="button"
                aria-label="Help: Why we need your SSN"
                className="help-button"
              >
                ?
              </button>
            </label>
            <input
              id="ssn"
              type="text"
              aria-describedby="ssn-help"
            />
            <div id="ssn-help" className="help-text">
              We need your SSN to verify your identity and process your application.
              Your information is encrypted and protected.
            </div>
          </div>
        </form>
      </article>
    </div>
  );
};
```

## WCAG 2.2 New Success Criteria

WCAG 2.2 introduced nine new success criteria:

1. **2.4.11 Focus Not Obscured (Minimum)** - AA: When a component receives keyboard focus, it's not entirely hidden by author-created content
2. **2.4.12 Focus Not Obscured (Enhanced)** - AAA: When a component receives focus, no part is hidden by author-created content
3. **2.4.13 Focus Appearance** - AAA: The focus indicator has sufficient size and contrast
4. **2.5.7 Dragging Movements** - AA: Functionality that uses dragging can be achieved by a single pointer without dragging
5. **2.5.8 Target Size (Minimum)** - AA: Touch targets are at least 24x24 CSS pixels
6. **3.2.6 Consistent Help** - A: Help mechanisms appear in the same relative order across pages
7. **3.3.7 Redundant Entry** - A: Information previously entered is auto-populated or selectable
8. **3.3.8 Accessible Authentication (Minimum)** - AA: Cognitive function tests not required for authentication
9. **3.3.9 Accessible Authentication (Enhanced)** - AAA: No cognitive function tests required for authentication

```typescript
// WCAG 2.2 compliance examples
const WCAG22Features: React.FC = () => {
  const [draggedItem, setDraggedItem] = useState<string | null>(null);

  // 2.5.7: Dragging Movements - Provide alternative to drag-and-drop
  const handleReorder = (item: string, direction: 'up' | 'down') => {
    // Allow reordering without dragging
    console.log(`Moving ${item} ${direction}`);
  };

  return (
    <div>
      {/* 2.4.11: Focus Not Obscured - Ensure sticky headers don't hide focus */}
      <style>{`
        .sticky-header {
          position: sticky;
          top: 0;
          z-index: 10;
        }
        
        /* Ensure focused items scroll into view with offset */
        *:focus {
          scroll-margin-top: 80px; /* Height of sticky header + buffer */
        }
      `}</style>
      
      {/* 2.5.8: Target Size (Minimum) - 24x24px minimum */}
      <button
        style={{
          minWidth: '44px', // Exceed 24px minimum
          minHeight: '44px',
          padding: '8px'
        }}
      >
        ×
      </button>
      
      {/* 2.5.7: Dragging Movements - Alternative to drag-and-drop */}
      <ul className="sortable-list">
        <li>
          <span>Item 1</span>
          <div className="reorder-buttons">
            <button
              onClick={() => handleReorder('Item 1', 'up')}
              aria-label="Move Item 1 up"
            >
              ↑
            </button>
            <button
              onClick={() => handleReorder('Item 1', 'down')}
              aria-label="Move Item 1 down"
            >
              ↓
            </button>
          </div>
        </li>
      </ul>
      
      {/* 3.3.7: Redundant Entry - Auto-populate previous entries */}
      <form>
        <label htmlFor="billing-address">Billing Address</label>
        <input id="billing-address" type="text" />
        
        <label>
          <input type="checkbox" id="same-as-billing" />
          Shipping address same as billing
        </label>
        
        <label htmlFor="shipping-address">Shipping Address</label>
        <input 
          id="shipping-address" 
          type="text"
          // Auto-populate if checkbox selected
        />
      </form>
      
      {/* 3.3.8: Accessible Authentication - No cognitive tests */}
      <div className="login-form">
        <p>
          We've sent a code to your email. 
          You can copy and paste it below.
        </p>
        {/* Allow paste, don't require memorization */}
        <input
          type="text"
          placeholder="Enter verification code"
          autoComplete="one-time-code"
        />
      </div>
    </div>
  );
};
```

## Common Mistakes

### Mistake 1: Adding ARIA When Semantic HTML Would Work
Using ARIA when native HTML elements provide the same functionality.

```typescript
// ❌ BAD: Unnecessary ARIA
<div role="button" tabIndex={0} onClick={handleClick}>
  Click me
</div>

// ✅ GOOD: Use semantic HTML
<button onClick={handleClick}>
  Click me
</button>
```

### Mistake 2: Missing Alt Text or Poor Alt Text
Not providing alternative text for images or writing poor descriptions.

```typescript
// ❌ BAD: Missing or redundant alt text
<img src="photo.jpg" alt="" />
<img src="chart.jpg" alt="chart" />
<img src="logo.jpg" alt="logo image" />

// ✅ GOOD: Descriptive alt text
<img src="photo.jpg" alt="Team celebrating project launch in office" />
<img src="chart.jpg" alt="Bar chart showing 25% revenue increase from Q1 to Q2" />
<img src="logo.jpg" alt="Acme Corporation" />
```

### Mistake 3: Removing Focus Outline
Removing focus indicators without providing an alternative.

```css
/* ❌ BAD: Removing focus outline without alternative */
*:focus {
  outline: none;
}

/* ✅ GOOD: Custom focus indicator with sufficient contrast */
*:focus {
  outline: 3px solid #005fcc;
  outline-offset: 2px;
}

/* ✅ GOOD: Respect user preferences */
*:focus-visible {
  outline: 3px solid #005fcc;
  outline-offset: 2px;
}
```

### Mistake 4: Insufficient Color Contrast
Using color combinations that don't meet minimum contrast ratios.

```typescript
// ❌ BAD: Insufficient contrast (2.5:1)
<button style={{ background: '#7fa8c9', color: '#ffffff' }}>
  Submit
</button>

// ✅ GOOD: Sufficient contrast (4.5:1 for AA, 7:1 for AAA)
<button style={{ background: '#005fcc', color: '#ffffff' }}>
  Submit
</button>
```

### Mistake 5: Inaccessible Form Labels
Not properly associating labels with form controls.

```typescript
// ❌ BAD: Label not associated with input
<div>
  <span>Email:</span>
  <input type="email" />
</div>

// ✅ GOOD: Properly associated label
<div>
  <label htmlFor="email-input">Email:</label>
  <input id="email-input" type="email" />
</div>
```

## Best Practices

### 1. Start with Semantic HTML
Always use the most appropriate HTML element for the job. Semantic HTML provides built-in accessibility features.

```typescript
// ✅ Use semantic elements
<nav>
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<main>
  <article>
    <h1>Article Title</h1>
    <p>Content...</p>
  </article>
</main>

<footer>
  <p>&copy; 2024 Company Name</p>
</footer>
```

### 2. Test with Keyboard Only
Ensure all functionality is accessible via keyboard before adding mouse support.

```typescript
// ✅ Keyboard accessible custom component
const AccessibleTab: React.FC = () => {
  const [activeTab, setActiveTab] = useState(0);
  
  const handleKeyDown = (e: React.KeyboardEvent, index: number) => {
    let newIndex = index;
    
    switch (e.key) {
      case 'ArrowRight':
        newIndex = (index + 1) % 3;
        break;
      case 'ArrowLeft':
        newIndex = index === 0 ? 2 : index - 1;
        break;
      case 'Home':
        newIndex = 0;
        break;
      case 'End':
        newIndex = 2;
        break;
      default:
        return;
    }
    
    e.preventDefault();
    setActiveTab(newIndex);
  };
  
  return (
    <div role="tablist">
      {[0, 1, 2].map(index => (
        <button
          key={index}
          role="tab"
          aria-selected={activeTab === index}
          aria-controls={`panel-${index}`}
          tabIndex={activeTab === index ? 0 : -1}
          onClick={() => setActiveTab(index)}
          onKeyDown={(e) => handleKeyDown(e, index)}
        >
          Tab {index + 1}
        </button>
      ))}
    </div>
  );
};
```

### 3. Provide Multiple Ways to Complete Tasks
Don't rely on a single input method—support keyboard, mouse, touch, and voice.

### 4. Test with Real Users and Assistive Technologies
Automated testing catches only about 30% of accessibility issues. Test with real screen readers and users with disabilities.

### 5. Make Accessibility Part of Your Definition of Done
Don't treat accessibility as a separate phase—build it in from the start.

## When to Target Each Conformance Level

### Target Level A When:
- Building internal tools with minimal accessibility requirements
- Creating prototypes or MVPs (but plan to improve)
- You have severe resource constraints (but this is not ideal)

### Target Level AA When:
- Building public-facing websites and applications
- Required by law (ADA, Section 508, European Accessibility Act)
- Serving diverse user bases
- Building enterprise applications
- Want to reach the widest possible audience

### Target Level AAA When:
- Building specialized applications for users with disabilities
- Creating government services
- Developing educational content
- Building healthcare applications
- Want to lead in accessibility best practices

## Interview Questions

**Q1: What are the four principles of WCAG (POUR) and what does each mean?**

**A:** WCAG is built on four principles:

1. **Perceivable**: Information and user interface components must be presentable to users in ways they can perceive. This means content can't be invisible to all of their senses. Examples include providing alt text for images, captions for videos, and sufficient color contrast.

2. **Operable**: User interface components and navigation must be operable. Users must be able to operate the interface—it cannot require interaction that a user cannot perform. This includes keyboard accessibility, providing enough time to complete tasks, and not designing content that causes seizures.

3. **Understandable**: Information and operation of the user interface must be understandable. Users must be able to understand both the information and how to operate the interface. This includes readable text, predictable behavior, and input assistance.

4. **Robust**: Content must be robust enough to be interpreted reliably by a wide variety of user agents, including assistive technologies. This means using valid, well-formed HTML and ensuring compatibility with current and future assistive technologies.

**Q2: What's the difference between WCAG conformance levels A, AA, and AAA?**

**A:** WCAG has three conformance levels, each building on the previous:

**Level A (Minimum)**: The most basic accessibility features. If these aren't met, some users will find it impossible to access content. Includes basics like text alternatives for images, keyboard accessibility, and no keyboard traps. Has 30 success criteria.

**Level AA (Recommended)**: Addresses the most common barriers for disabled users. This is the standard most organizations target and what's typically required by law. Includes all Level A criteria plus additional requirements like 4.5:1 contrast ratio, captions for live audio, and visible focus indicators. Has 20 additional criteria (50 total).

**Level AAA (Enhanced)**: The highest level of accessibility. Not recommended as a blanket requirement for entire sites since some content can't meet all AAA criteria. Includes enhanced requirements like 7:1 contrast ratio, sign language interpretation, and extended audio descriptions. Has 28 additional criteria (78 total).

**Q3: What are some new success criteria introduced in WCAG 2.2?**

**A:** WCAG 2.2 introduced nine new success criteria focused on mobile accessibility and cognitive disabilities:

- **2.4.11 Focus Not Obscured (Minimum)** - AA: Focused elements aren't completely hidden by sticky headers or other content
- **2.5.7 Dragging Movements** - AA: Functionality using dragging has single-pointer alternatives
- **2.5.8 Target Size (Minimum)** - AA: Touch targets are at least 24x24 CSS pixels
- **3.2.6 Consistent Help** - A: Help mechanisms appear in consistent locations
- **3.3.7 Redundant Entry** - A: Previously entered information is auto-populated
- **3.3.8 Accessible Authentication (Minimum)** - AA: No cognitive function tests required for authentication (but allows object recognition or personal content)

These additions particularly benefit mobile users and people with cognitive or motor disabilities.

**Q4: How do you test for WCAG compliance in a practical workflow?**

**A:** A comprehensive WCAG testing approach includes:

**Automated Testing (catches ~30% of issues):**
- Use tools like axe DevTools, Lighthouse, or WAVE
- Integrate into CI/CD with @axe-core/react or pa11y
- Run on every commit or pull request

**Manual Testing:**
- Keyboard-only testing: Tab through entire interface
- Screen reader testing: NVDA (Windows), JAWS (Windows), VoiceOver (Mac/iOS)
- Color contrast testing with tools like Contrast Checker
- Content zoom to 200% to verify layout doesn't break
- Test with different viewport sizes

**Real User Testing:**
- Involve people with disabilities in testing
- Use services like Fable or UserTesting with accessibility participants
- Conduct usability studies with assistive technology users

**Code Review:**
- Check for semantic HTML
- Verify ARIA usage is appropriate
- Ensure form labels are properly associated
- Review focus management in interactive components

**Continuous Monitoring:**
- Regular accessibility audits (quarterly or semi-annually)
- Monitor user feedback and bug reports
- Track accessibility metrics over time

**Q5: When should you use ARIA and when should you avoid it?**

**A:** The first rule of ARIA is: "Don't use ARIA." Use native HTML elements whenever possible because they have built-in accessibility features.

**Use ARIA When:**
- Native HTML doesn't provide the semantics you need (tabs, tree views, complex widgets)
- You need to expose dynamic content changes (aria-live regions)
- You need to provide additional context (aria-label, aria-describedby)
- Building custom interactive components not available in HTML

**Avoid ARIA When:**
- Native HTML element exists (`<button>` instead of `<div role="button">`)
- You're just adding it without testing with screen readers
- It duplicates native semantics (`<nav role="navigation">`)
- You're using it to fix invalid HTML (fix the HTML instead)

**Example of proper ARIA usage:**
```typescript
// ✅ GOOD: ARIA for custom widget not in HTML
<div
  role="tablist"
  aria-label="Account settings"
>
  <button
    role="tab"
    aria-selected="true"
    aria-controls="panel-profile"
  >
    Profile
  </button>
</div>

// ❌ BAD: Unnecessary ARIA when native element exists
<div 
  role="button" 
  tabIndex={0} 
  onClick={handleClick}
>
  Click me
</div>

// ✅ GOOD: Native button
<button onClick={handleClick}>
  Click me
</button>
```

**Q6: How do you ensure color contrast meets WCAG standards?**

**A:** WCAG requires minimum contrast ratios:
- **Level AA**: 4.5:1 for normal text, 3:1 for large text (18pt+ or 14pt+ bold)
- **Level AAA**: 7:1 for normal text, 4.5:1 for large text

**Practical approaches:**

1. **Use contrast checking tools:**
   - Browser DevTools color picker shows contrast ratios
   - WebAIM Contrast Checker
   - Stark plugin for design tools
   - Contrast Analyzer apps

2. **Design with accessibility in mind:**
```css
:root {
  /* AA compliant color palette */
  --text-primary: #1a1a1a;    /* 14.5:1 against white */
  --text-secondary: #4a4a4a;  /* 9:1 against white */
  --link-blue: #0066cc;       /* 4.5:1 against white (AA) */
  --link-blue-dark: #004499;  /* 7:1 against white (AAA) */
}

/* Ensure sufficient contrast for all states */
.button-primary {
  background: #0066cc;
  color: #ffffff;  /* High contrast */
}

.button-primary:hover {
  background: #0052a3;  /* Maintain contrast */
}

.button-primary:disabled {
  /* Disabled elements exempt from contrast requirements */
  opacity: 0.5;
}
```

3. **Test in different conditions:**
   - Grayscale mode to check reliance on color
   - Color blindness simulators
   - Different screen brightness settings
   - Various devices and screen types

4. **Don't rely on color alone:**
```typescript
// ✅ GOOD: Multiple indicators
<div className="status-error">
  <span className="icon" aria-hidden="true">⚠</span>
  <span style={{ color: '#c00' }}>Error occurred</span>
</div>
```

**Q7: How do you make single-page applications (SPAs) accessible?**

**A:** SPAs present unique accessibility challenges because content changes without page reloads:

**1. Manage Focus on Route Changes:**
```typescript
// React Router example
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

function App() {
  const location = useLocation();
  
  useEffect(() => {
    // Focus main content heading on route change
    const heading = document.querySelector('h1');
    if (heading) {
      heading.setAttribute('tabindex', '-1');
      heading.focus();
    }
  }, [location]);
  
  return <Routes>...</Routes>;
}
```

**2. Announce Route Changes:**
```typescript
// Route announcement component
const RouteAnnouncer: React.FC = () => {
  const location = useLocation();
  const [announcement, setAnnouncement] = useState('');
  
  useEffect(() => {
    // Announce new page
    const title = document.title;
    setAnnouncement(`Navigated to ${title}`);
  }, [location]);
  
  return (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className="sr-only"
    >
      {announcement}
    </div>
  );
};
```

**3. Manage Document Title:**
```typescript
useEffect(() => {
  document.title = `${pageTitle} - App Name`;
}, [pageTitle]);
```

**4. Handle Dynamic Content:**
```typescript
// Use aria-live for dynamic updates
<div aria-live="polite" aria-atomic="true">
  {statusMessage}
</div>
```

**5. Preserve Scroll Position:**
```typescript
// Scroll to top on navigation, but preserve on back button
useEffect(() => {
  if (location.action === 'PUSH') {
    window.scrollTo(0, 0);
  }
}, [location]);
```

**Q8: What are the most common WCAG violations you see in production applications?**

**A:** Based on WebAIM's annual accessibility analysis:

**1. Low Contrast Text (86.4% of homepages):**
- Text doesn't meet 4.5:1 contrast ratio
- Often affects secondary text, placeholders, disabled states

**2. Missing Alt Text (58.2% of homepages):**
- Images without alt attributes
- Decorative images not marked with empty alt
- Complex images without sufficient descriptions

**3. Empty Links (50.1% of homepages):**
- Links with no text content
- Icon-only links without labels
- Links with only images without alt text

**4. Missing Form Labels (45.9% of homepages):**
- Inputs without associated labels
- Placeholder used instead of label
- Visual labels not programmatically associated

**5. Empty Buttons (27.5% of homepages):**
- Buttons with no accessible text
- Icon buttons without aria-label
- Submit buttons with only icons

**6. Missing Document Language (17.1% of homepages):**
- No lang attribute on HTML element
- Incorrect language specified
- Language changes not marked

**Real-world example of fixing common violations:**
```typescript
// ❌ BEFORE: Multiple violations
<div style={{ color: '#999' }}>  {/* Low contrast */}
  <img src="logo.png" />  {/* Missing alt */}
  <a href="/home"><i className="icon-home"></i></a>  {/* Empty link */}
  <input placeholder="Email" />  {/* Missing label */}
  <button><i className="icon-search"></i></button>  {/* Empty button */}
</div>

// ✅ AFTER: WCAG compliant
<div style={{ color: '#666' }}>  {/* Sufficient contrast */}
  <img src="logo.png" alt="Company Name" />
  <a href="/home" aria-label="Home">
    <i className="icon-home" aria-hidden="true"></i>
  </a>
  <label htmlFor="email-input">Email</label>
  <input id="email-input" type="email" placeholder="name@example.com" />
  <button aria-label="Search">
    <i className="icon-search" aria-hidden="true"></i>
  </button>
</div>
```

## Key Takeaways

- WCAG is organized around four POUR principles: Perceivable, Operable, Understandable, and Robust
- Three conformance levels: A (minimum), AA (recommended standard), AAA (enhanced)
- WCAG 2.2 adds nine new criteria focused on mobile and cognitive accessibility
- Level AA is the legal standard for most jurisdictions and should be the minimum target
- Automated testing catches only ~30% of accessibility issues—manual testing is essential
- Start with semantic HTML before adding ARIA
- Test with keyboard only and screen readers regularly
- Color contrast ratios: 4.5:1 minimum for normal text (AA), 7:1 for AAA
- Most common violations: low contrast, missing alt text, unlabeled forms, empty links/buttons
- Build accessibility in from the start—it's harder and more expensive to retrofit
- Accessibility benefits everyone, not just users with disabilities
- SPAs require special attention to focus management and route announcements
- Document language must be specified for proper pronunciation by screen readers
- Make all interactive elements keyboard accessible with visible focus indicators
- Provide multiple ways to complete tasks—don't rely on a single input method

## Resources

### Official Specifications
- [WCAG 2.2 Guidelines](https://www.w3.org/TR/WCAG22/) - Official W3C specification
- [Understanding WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/) - Detailed explanations of success criteria
- [How to Meet WCAG (Quick Reference)](https://www.w3.org/WAI/WCAG22/quickref/) - Customizable quick reference
- [WCAG 2.2 What's New](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/) - Changes from 2.1 to 2.2

### Testing Tools
- [axe DevTools](https://www.deque.com/axe/devtools/) - Browser extension for automated testing
- [WAVE](https://wave.webaim.org/) - Web accessibility evaluation tool
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) - Color contrast testing
- [Lighthouse](https://developers.google.com/web/tools/lighthouse) - Automated auditing in Chrome DevTools

### Educational Resources
- [WebAIM Articles](https://webaim.org/articles/) - Comprehensive accessibility articles
- [A11y Project Checklist](https://www.a11yproject.com/checklist/) - Practical WCAG checklist
- [MDN Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility) - Technical documentation
- [Inclusive Components](https://inclusive-components.design/) - Pattern library for accessible components

### Books
- "Accessibility for Everyone" by Laura Kalbag
- "A Web for Everyone" by Sarah Horton and Whitney Quesenbery
- "Form Design Patterns" by Adam Silver (includes accessibility)
