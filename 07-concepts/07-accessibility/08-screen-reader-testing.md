# Screen Reader Testing

## Table of Contents
- [Introduction](#introduction)
- [Why Screen Reader Testing Matters](#why-screen-reader-testing-matters)
- [Major Screen Readers](#major-screen-readers)
- [Screen Reader Basics](#screen-reader-basics)
- [Testing Strategies](#testing-strategies)
- [Common Accessibility Patterns](#common-accessibility-patterns)
- [Testing Checklist](#testing-checklist)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Screen reader testing is essential for ensuring your web application is accessible to blind and visually impaired users. Screen readers convert digital text and interface elements into speech or braille, allowing users to navigate and interact with web content without seeing it.

```
Screen Reader User Journey
===========================

HTML Element -> Screen Reader -> User
     |              |              |
     v              v              v
Semantic HTML -> Parse + ARIA -> Speech/Braille
     |              |              |
     v              v              v
DOM Structure -> Accessibility -> Navigation
                    Tree
```

Approximately 2.2 billion people worldwide have some form of vision impairment, making screen reader accessibility crucial for inclusive web design.

## Why Screen Reader Testing Matters

### Legal and Ethical Obligations

1. **ADA Compliance**: Americans with Disabilities Act requirements
2. **Section 508**: US federal agency requirements
3. **WCAG Guidelines**: International accessibility standards
4. **Legal Precedents**: Increasing lawsuits for inaccessible websites

### User Impact

```typescript
// Example: How screen readers interpret content

// ❌ Poor implementation - confusing to screen readers
<div class="button" onclick="submit()">Submit</div>
// Announced as: "Submit, text"

// ✅ Good implementation - clear semantics
<button type="submit">Submit</button>
// Announced as: "Submit, button"
```

## Major Screen Readers

### 1. NVDA (NonVisual Desktop Access)

**Platform**: Windows  
**Cost**: Free and open source  
**Market Share**: ~40% of screen reader users  
**Best For**: Testing on Windows

```
NVDA Basic Commands
===================

Navigation:
- Down Arrow: Next line
- Up Arrow: Previous line
- Tab: Next focusable element
- Shift+Tab: Previous focusable element
- H: Next heading
- Shift+H: Previous heading
- K: Next link
- Shift+K: Previous link

Reading:
- Insert+Down Arrow: Read all (continuous)
- Insert+Up Arrow: Current line
- Insert+T: Read window title
- Insert+F7: Elements list

Modes:
- Insert+Space: Toggle focus/browse mode
- Insert+Z: Toggle browse mode
```

#### NVDA Installation and Setup

```bash
# Download from: https://www.nvaccess.org/download/
# Run installer
# Basic configuration for testing

# Key settings for developers:
# 1. Speech viewer (visual feedback)
# 2. Highlight focus
# 3. Debug logging
```

```typescript
// React: Testing component with NVDA
const AccessibleButton: React.FC = () => {
  return (
    <button
      aria-label="Submit form"
      aria-describedby="submit-help"
    >
      Submit
      <span id="submit-help" className="sr-only">
        This will submit your application
      </span>
    </button>
  );
};

/*
NVDA Announcement:
"Submit form, button, This will submit your application"
*/
```

### 2. JAWS (Job Access With Speech)

**Platform**: Windows  
**Cost**: $95/year (license)  
**Market Share**: ~30% of screen reader users  
**Best For**: Enterprise testing

```
JAWS Basic Commands
===================

Navigation:
- Down Arrow: Next line
- Up Arrow: Previous line
- T: Next table
- Shift+T: Previous table
- F: Next form field
- Shift+F: Previous form field
- R: Next region/landmark
- Shift+R: Previous region/landmark

Reading:
- Insert+Down Arrow: Say all
- Insert+Up Arrow: Current line
- Insert+F5: Form fields list
- Insert+F6: Headings list
- Insert+F7: Links list

Control:
- Insert+Z: Toggle virtual cursor
- Insert+3: Toggle forms mode
```

```typescript
// Angular: JAWS-optimized form
@Component({
  selector: 'app-accessible-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <fieldset>
        <legend>Personal Information</legend>
        
        <div class="form-group">
          <label for="firstName">
            First Name
            <span aria-label="required">*</span>
          </label>
          <input
            id="firstName"
            type="text"
            formControlName="firstName"
            [attr.aria-invalid]="firstName.invalid && firstName.touched"
            [attr.aria-describedby]="firstName.invalid ? 'firstName-error' : null"
            required
          />
          <div
            id="firstName-error"
            role="alert"
            *ngIf="firstName.invalid && firstName.touched"
          >
            First name is required
          </div>
        </div>
      </fieldset>
      
      <button type="submit" [disabled]="form.invalid">
        Submit Application
      </button>
    </form>
  `
})
export class AccessibleFormComponent {
  form = this.fb.group({
    firstName: ['', Validators.required]
  });

  get firstName() {
    return this.form.get('firstName')!;
  }

  constructor(private fb: FormBuilder) {}

  onSubmit() {
    // Form submission
  }
}

/*
JAWS Announcements:
- "Personal Information, grouping"
- "First Name, asterisk required, edit, blank"
- "First name is required" (on blur if invalid)
- "Submit Application, button, unavailable" (when form invalid)
*/
```

### 3. VoiceOver

**Platform**: macOS, iOS  
**Cost**: Free (built-in)  
**Market Share**: ~25% of screen reader users  
**Best For**: Apple device testing

```
VoiceOver Commands (macOS)
===========================

Basic:
- Cmd+F5: Toggle VoiceOver
- VO = Control+Option

Navigation:
- VO+Right Arrow: Next item
- VO+Left Arrow: Previous item
- VO+A: Read all
- VO+H H: Next heading
- VO+Shift+H H: Previous heading

Rotor (VO+U):
- Left/Right Arrow: Switch rotor type
- Up/Down Arrow: Navigate items
- Enter: Go to item

Web:
- Tab: Next focusable element
- VO+Space: Activate
- VO+Shift+Down: Interact with element
- VO+Shift+Up: Stop interacting
```

```typescript
// React: VoiceOver-optimized navigation
const AccessibleNav: React.FC = () => {
  return (
    <nav aria-label="Main navigation">
      <ul>
        <li>
          <a href="/" aria-current="page">
            Home
          </a>
        </li>
        <li>
          <a href="/products">Products</a>
        </li>
        <li>
          <a href="/about">About</a>
        </li>
        <li>
          <a href="/contact">Contact</a>
        </li>
      </ul>
    </nav>
  );
};

/*
VoiceOver Announcements:
- "Main navigation, navigation"
- "Home, current page, link"
- "Products, link"
- etc.
*/
```

### 4. TalkBack

**Platform**: Android  
**Cost**: Free (built-in)  
**Market Share**: Mobile users  
**Best For**: Mobile testing

```
TalkBack Gestures
=================

Navigation:
- Swipe Right: Next item
- Swipe Left: Previous item
- Swipe Down then Right: Reading controls
- Swipe Down then Left: Navigation menu

Actions:
- Double Tap: Activate
- Swipe Up then Right: Context menu
- Swipe Down then Up: Read from top
- Swipe Up then Down: Read from current

Settings:
- Swipe Down then Left: TalkBack menu
```

```typescript
// React Native: TalkBack-optimized component
import { View, Text, TouchableOpacity } from 'react-native';

const AccessibleCard = () => {
  return (
    <TouchableOpacity
      accessible={true}
      accessibilityRole="button"
      accessibilityLabel="Product: Blue Cotton T-Shirt"
      accessibilityHint="Double tap to view product details"
      accessibilityState={{ selected: false }}
    >
      <View>
        <Text>Blue Cotton T-Shirt</Text>
        <Text>$29.99</Text>
        <Text>4.5 stars</Text>
      </View>
    </TouchableOpacity>
  );
};

/*
TalkBack Announcement:
"Product: Blue Cotton T-Shirt, button. Double tap to view product details"
*/
```

## Screen Reader Basics

### How Screen Readers Work

```
Screen Reader Processing Pipeline
==================================

1. HTML/DOM
   └─> Parsed by browser

2. Accessibility Tree
   └─> Built from semantic HTML + ARIA
   
3. Screen Reader Buffer
   └─> Virtual representation of page

4. User Navigation
   └─> Commands to move through content

5. Output
   └─> Speech synthesis or braille display
```

### The Accessibility Tree

```typescript
// React: Understanding the Accessibility Tree
const ComponentStructure: React.FC = () => {
  return (
    <article aria-labelledby="article-title">
      <header>
        <h2 id="article-title">Article Title</h2>
        <p>
          By <span>Author Name</span> on{' '}
          <time dateTime="2024-01-15">January 15, 2024</time>
        </p>
      </header>
      <div>
        <p>Article content goes here...</p>
      </div>
      <footer>
        <button aria-label="Like article">👍</button>
        <button aria-label="Share article">📤</button>
      </footer>
    </article>
  );
};

/*
Accessibility Tree Structure:
article (Article Title)
├─ group (header)
│  ├─ heading level 2 "Article Title"
│  └─ text "By Author Name on January 15, 2024"
├─ text "Article content goes here..."
└─ group (footer)
   ├─ button "Like article"
   └─ button "Share article"
*/
```

### Browse vs Focus Mode

```typescript
// React: Understanding modes
const FormExample: React.FC = () => {
  return (
    <div>
      {/* Browse Mode: Read-only navigation */}
      <h1>Register for Event</h1>
      <p>Please fill out the form below to register.</p>

      {/* Focus Mode: Triggered by form fields */}
      <form>
        <label htmlFor="email">Email:</label>
        <input
          id="email"
          type="email"
          placeholder="your@email.com"
          aria-describedby="email-help"
        />
        <div id="email-help" className="help-text">
          We'll never share your email
        </div>

        <button type="submit">Register</button>
      </form>
    </div>
  );
};

/*
Screen Reader Behavior:
- Browse mode for heading and paragraph
- Switches to focus mode when entering input
- User can type without mode conflicts
- Back to browse mode after leaving form
*/
```

## Testing Strategies

### 1. Comprehensive Testing Checklist

```typescript
// React: Testable Component with Accessibility Features
import React, { useState } from 'react';

const AccessibleDataTable: React.FC = () => {
  const [sortBy, setSortBy] = useState<string>('name');
  const [sortOrder, setSortOrder] = useState<'asc' | 'desc'>('asc');

  const data = [
    { id: 1, name: 'Alice', role: 'Developer', status: 'Active' },
    { id: 2, name: 'Bob', role: 'Designer', status: 'Active' },
    { id: 3, name: 'Charlie', role: 'Manager', status: 'Inactive' }
  ];

  const handleSort = (column: string) => {
    if (sortBy === column) {
      setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc');
    } else {
      setSortBy(column);
      setSortOrder('asc');
    }
  };

  return (
    <div>
      <h2 id="table-title">Employee Directory</h2>
      
      {/* Screen reader-only summary */}
      <p id="table-description" className="sr-only">
        Table with {data.length} employees, sortable by name, role, and status
      </p>

      <table
        aria-labelledby="table-title"
        aria-describedby="table-description"
      >
        <thead>
          <tr>
            <th scope="col">
              <button
                onClick={() => handleSort('name')}
                aria-sort={
                  sortBy === 'name'
                    ? sortOrder === 'asc'
                      ? 'ascending'
                      : 'descending'
                    : 'none'
                }
              >
                Name
                {sortBy === 'name' && (
                  <span aria-hidden="true">
                    {sortOrder === 'asc' ? ' ↑' : ' ↓'}
                  </span>
                )}
              </button>
            </th>
            <th scope="col">Role</th>
            <th scope="col">Status</th>
            <th scope="col">Actions</th>
          </tr>
        </thead>
        <tbody>
          {data.map((employee) => (
            <tr key={employee.id}>
              <td>{employee.name}</td>
              <td>{employee.role}</td>
              <td>
                <span
                  className={`status ${employee.status.toLowerCase()}`}
                  role="img"
                  aria-label={`Status: ${employee.status}`}
                >
                  {employee.status}
                </span>
              </td>
              <td>
                <button
                  aria-label={`Edit ${employee.name}'s profile`}
                >
                  Edit
                </button>
                <button
                  aria-label={`Delete ${employee.name}'s profile`}
                >
                  Delete
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

/*
Testing Checklist for this Component:
✓ Table has caption/aria-labelledby
✓ Headers use scope attribute
✓ Sortable columns announce sort state
✓ Status indicators have text alternatives
✓ Action buttons have descriptive labels
✓ Navigate with table navigation commands (T, Ctrl+Alt+arrows in JAWS/NVDA)
*/
```

### 2. Live Region Testing

```typescript
// React: Live Region Patterns
const LiveRegionExamples: React.FC = () => {
  const [notifications, setNotifications] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);
  const [items, setItems] = useState<string[]>([]);

  const addNotification = (message: string) => {
    setNotifications([...notifications, message]);
  };

  return (
    <div>
      {/* Polite: Non-urgent updates */}
      <div role="status" aria-live="polite" aria-atomic="true">
        {notifications[notifications.length - 1]}
      </div>

      {/* Assertive: Urgent updates */}
      <div role="alert" aria-live="assertive" aria-atomic="true">
        {loading && 'Loading, please wait...'}
      </div>

      {/* Off: No announcements */}
      <div aria-live="off">
        <p>This content changes but is not announced</p>
      </div>

      <button
        onClick={() => {
          setLoading(true);
          setTimeout(() => {
            setLoading(false);
            addNotification('Items loaded successfully');
          }, 2000);
        }}
      >
        Load Items
      </button>

      {/* Results list with count announcement */}
      <div
        role="region"
        aria-label="Search results"
        aria-live="polite"
        aria-atomic="false"
      >
        <p>{items.length} items found</p>
        <ul>
          {items.map((item, index) => (
            <li key={index}>{item}</li>
          ))}
        </ul>
      </div>
    </div>
  );
};

/*
Screen Reader Announcements:
- aria-live="polite": Announced after current speech
- aria-live="assertive": Interrupts current speech
- aria-atomic="true": Announces entire region
- aria-atomic="false": Announces only changes
*/
```

### 3. Form Testing

```typescript
// Angular: Comprehensive Form Accessibility
@Component({
  selector: 'app-accessible-registration-form',
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <h2 id="form-title">Account Registration</h2>
      
      <!-- Form-level error summary -->
      <div
        *ngIf="formSubmitted && registrationForm.invalid"
        role="alert"
        aria-labelledby="error-summary-title"
        class="error-summary"
      >
        <h3 id="error-summary-title">Please correct the following errors:</h3>
        <ul>
          <li *ngIf="username.invalid">
            <a href="#username">Username is required</a>
          </li>
          <li *ngIf="email.invalid">
            <a href="#email">Valid email is required</a>
          </li>
          <li *ngIf="password.invalid">
            <a href="#password">Password must be at least 8 characters</a>
          </li>
        </ul>
      </div>

      <!-- Username field -->
      <div class="form-group">
        <label for="username">
          Username
          <span aria-label="required" class="required">*</span>
        </label>
        <input
          id="username"
          type="text"
          formControlName="username"
          [attr.aria-invalid]="username.invalid && username.touched"
          [attr.aria-describedby]="getDescribedBy('username')"
          aria-required="true"
        />
        <div id="username-help" class="help-text">
          Choose a unique username (3-20 characters)
        </div>
        <div
          id="username-error"
          role="alert"
          *ngIf="username.invalid && username.touched"
          class="error-message"
        >
          <span *ngIf="username.errors?.['required']">
            Username is required
          </span>
          <span *ngIf="username.errors?.['minlength']">
            Username must be at least 3 characters
          </span>
        </div>
      </div>

      <!-- Email field with validation -->
      <div class="form-group">
        <label for="email">
          Email Address
          <span aria-label="required" class="required">*</span>
        </label>
        <input
          id="email"
          type="email"
          formControlName="email"
          [attr.aria-invalid]="email.invalid && email.touched"
          [attr.aria-describedby]="getDescribedBy('email')"
          aria-required="true"
        />
        <div
          id="email-error"
          role="alert"
          *ngIf="email.invalid && email.touched"
          class="error-message"
        >
          Please enter a valid email address
        </div>
      </div>

      <!-- Password with strength indicator -->
      <div class="form-group">
        <label for="password">
          Password
          <span aria-label="required" class="required">*</span>
        </label>
        <input
          id="password"
          type="password"
          formControlName="password"
          [attr.aria-invalid]="password.invalid && password.touched"
          [attr.aria-describedby]="getDescribedBy('password')"
          aria-required="true"
        />
        <div
          id="password-strength"
          role="status"
          aria-live="polite"
          *ngIf="password.value"
        >
          Password strength: {{ getPasswordStrength() }}
        </div>
        <div id="password-help" class="help-text">
          Must be at least 8 characters
        </div>
      </div>

      <!-- Submit button -->
      <button
        type="submit"
        [disabled]="registrationForm.invalid"
        [attr.aria-disabled]="registrationForm.invalid"
      >
        Create Account
      </button>

      <!-- Success message -->
      <div
        role="alert"
        aria-live="polite"
        *ngIf="submissionSuccess"
      >
        Account created successfully! Redirecting...
      </div>
    </form>
  `
})
export class AccessibleRegistrationFormComponent {
  formSubmitted = false;
  submissionSuccess = false;

  registrationForm = this.fb.group({
    username: ['', [Validators.required, Validators.minLength(3)]],
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]]
  });

  get username() { return this.registrationForm.get('username')!; }
  get email() { return this.registrationForm.get('email')!; }
  get password() { return this.registrationForm.get('password')!; }

  constructor(private fb: FormBuilder) {}

  getDescribedBy(fieldName: string): string {
    const control = this.registrationForm.get(fieldName);
    const describedBy = [`${fieldName}-help`];
    
    if (control?.invalid && control?.touched) {
      describedBy.push(`${fieldName}-error`);
    }
    
    if (fieldName === 'password' && this.password.value) {
      describedBy.push('password-strength');
    }
    
    return describedBy.join(' ');
  }

  getPasswordStrength(): string {
    const pwd = this.password.value || '';
    if (pwd.length < 8) return 'Too short';
    if (pwd.length < 12) return 'Weak';
    if (!/\d/.test(pwd)) return 'Moderate';
    return 'Strong';
  }

  onSubmit() {
    this.formSubmitted = true;
    
    if (this.registrationForm.valid) {
      // Submit form
      this.submissionSuccess = true;
    }
  }
}

/*
Screen Reader Testing Checklist:
✓ Labels properly associated with inputs
✓ Required fields announced
✓ Error messages announced immediately
✓ Error summary with links to fields
✓ aria-invalid toggled appropriately
✓ aria-describedby includes all relevant text
✓ Password strength announced as it changes
✓ Success message announced
*/
```

### 4. Modal Dialog Testing

```typescript
// React: Fully Accessible Modal
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
  const closeButtonRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    if (!isOpen) return;

    // Store and manage focus
    previousFocusRef.current = document.activeElement as HTMLElement;
    closeButtonRef.current?.focus();

    // Prevent background scrolling
    document.body.style.overflow = 'hidden';

    // Announce modal opening
    const announcement = document.createElement('div');
    announcement.setAttribute('role', 'status');
    announcement.setAttribute('aria-live', 'polite');
    announcement.textContent = `${title} dialog opened`;
    announcement.className = 'sr-only';
    document.body.appendChild(announcement);

    // Escape key handler
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    document.addEventListener('keydown', handleEscape);

    return () => {
      document.removeEventListener('keydown', handleEscape);
      document.body.style.overflow = '';
      announcement.remove();
      
      // Restore focus
      if (previousFocusRef.current) {
        previousFocusRef.current.focus();
      }
    };
  }, [isOpen, onClose, title]);

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
        aria-describedby="modal-description"
        onClick={(e) => e.stopPropagation()}
      >
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            ref={closeButtonRef}
            onClick={onClose}
            aria-label={`Close ${title} dialog`}
            className="modal-close"
          >
            <span aria-hidden="true">×</span>
          </button>
        </div>
        
        <div id="modal-description" className="modal-body">
          {children}
        </div>
        
        <div className="modal-footer">
          <button onClick={onClose}>Cancel</button>
          <button onClick={onClose} className="primary">
            Confirm
          </button>
        </div>
      </div>
    </div>,
    document.body
  );
};

/*
Screen Reader Testing:
✓ "Dialog opened" announced when modal appears
✓ Focus moves to close button
✓ Dialog role announced
✓ aria-modal prevents background interaction
✓ Title announced with dialog
✓ Escape closes dialog
✓ Focus returns to trigger element on close
✓ Background content hidden from screen reader
*/
```

## Testing Checklist

### Essential Tests for Every Component

```typescript
// React: Component Testing Checklist
const ComponentTestingGuide = () => {
  return (
    <div>
      <h1>Screen Reader Testing Checklist</h1>

      <section>
        <h2>1. Semantic Structure</h2>
        <ul>
          <li>✓ Proper heading hierarchy (h1-h6)</li>
          <li>✓ Landmarks (header, nav, main, aside, footer)</li>
          <li>✓ Lists for groups of items</li>
          <li>✓ Buttons vs links used correctly</li>
        </ul>
      </section>

      <section>
        <h2>2. Keyboard Navigation</h2>
        <ul>
          <li>✓ All interactive elements focusable</li>
          <li>✓ Logical tab order</li>
          <li>✓ Focus indicators visible</li>
          <li>✓ No keyboard traps</li>
        </ul>
      </section>

      <section>
        <h2>3. Labels and Descriptions</h2>
        <ul>
          <li>✓ All form fields have labels</li>
          <li>✓ Buttons have descriptive text</li>
          <li>✓ Images have alt text</li>
          <li>✓ Icons have text alternatives</li>
        </ul>
      </section>

      <section>
        <h2>4. ARIA Usage</h2>
        <ul>
          <li>✓ ARIA roles when semantic HTML insufficient</li>
          <li>✓ aria-label/aria-labelledby for context</li>
          <li>✓ aria-describedby for additional info</li>
          <li>✓ aria-live for dynamic content</li>
          <li>✓ aria-expanded for expandable elements</li>
          <li>✓ aria-current for current page/item</li>
        </ul>
      </section>

      <section>
        <h2>5. Dynamic Content</h2>
        <ul>
          <li>✓ Loading states announced</li>
          <li>✓ Error messages announced</li>
          <li>✓ Success messages announced</li>
          <li>✓ Content changes announced</li>
          <li>✓ Focus managed appropriately</li>
        </ul>
      </section>

      <section>
        <h2>6. Screen Reader Specific</h2>
        <ul>
          <li>✓ Test with NVDA (Windows)</li>
          <li>✓ Test with JAWS (Windows)</li>
          <li>✓ Test with VoiceOver (macOS/iOS)</li>
          <li>✓ Test with TalkBack (Android)</li>
          <li>✓ Navigate with SR shortcuts</li>
          <li>✓ Check element list (headings, landmarks, links)</li>
        </ul>
      </section>
    </div>
  );
};
```

## Common Mistakes

### 1. Using ARIA When Semantic HTML Would Suffice

```typescript
// ❌ WRONG: Unnecessary ARIA
<div role="button" tabIndex={0} onClick={handleClick}>
  Submit
</div>

// ✅ CORRECT: Use semantic HTML
<button onClick={handleClick}>
  Submit
</button>
```

### 2. Missing or Incorrect Alt Text

```typescript
// ❌ WRONG: Various alt text mistakes
<img src="logo.png" alt="logo.png" /> {/* Filename */}
<img src="photo.jpg" alt="image" /> {/* Generic */}
<img src="chart.png" alt="Chart showing sales data" /> {/* Not descriptive enough */}
<img src="icon.png" /> {/* Missing alt */}

// ✅ CORRECT: Appropriate alt text
<img src="logo.png" alt="Company Name" />
<img src="photo.jpg" alt="Team celebrating project completion" />
<img src="chart.png" alt="Bar chart showing 25% increase in Q4 sales compared to Q3" />
<img src="decorative.png" alt="" /> {/* Decorative image */}
```

### 3. Inaccessible Custom Components

```typescript
// ❌ WRONG: No keyboard or screen reader support
const Dropdown = () => {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <div onClick={() => setIsOpen(!isOpen)}>
        Select Option
      </div>
      {isOpen && (
        <div>
          <div onClick={() => console.log('Option 1')}>Option 1</div>
          <div onClick={() => console.log('Option 2')}>Option 2</div>
        </div>
      )}
    </div>
  );
};

// ✅ CORRECT: Accessible dropdown
const AccessibleDropdown = () => {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedIndex, setSelectedIndex] = useState(0);
  
  return (
    <div className="dropdown">
      <button
        id="dropdown-button"
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        onClick={() => setIsOpen(!isOpen)}
      >
        Select Option
      </button>
      
      {isOpen && (
        <ul
          role="listbox"
          aria-labelledby="dropdown-button"
        >
          <li
            role="option"
            aria-selected={selectedIndex === 0}
            onClick={() => setSelectedIndex(0)}
          >
            Option 1
          </li>
          <li
            role="option"
            aria-selected={selectedIndex === 1}
            onClick={() => setSelectedIndex(1)}
          >
            Option 2
          </li>
        </ul>
      )}
    </div>
  );
};
```

### 4. Not Announcing Dynamic Changes

```typescript
// ❌ WRONG: No announcement of loading state
const DataLoader = () => {
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState(null);
  
  return (
    <div>
      <button onClick={loadData}>Load Data</button>
      {loading && <div>Loading...</div>}
      {data && <div>{data}</div>}
    </div>
  );
};

// ✅ CORRECT: Announce state changes
const AccessibleDataLoader = () => {
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  
  return (
    <div>
      <button onClick={loadData}>Load Data</button>
      
      <div role="status" aria-live="polite" aria-atomic="true">
        {loading && 'Loading data, please wait...'}
      </div>
      
      <div role="alert" aria-live="assertive">
        {error && `Error: ${error}`}
      </div>
      
      {data && (
        <div>
          <div role="status" aria-live="polite">
            Data loaded successfully
          </div>
          <div>{data}</div>
        </div>
      )}
    </div>
  );
};
```

## Best Practices

### 1. Use Screen Reader Only Text

```typescript
// React: Screen reader only utility class
const srOnlyStyles = `
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }
`;

// Usage
const IconButton: React.FC = () => (
  <button aria-label="Save document">
    <span className="sr-only">Save document</span>
    <SaveIcon aria-hidden="true" />
  </button>
);
```

### 2. Progressive Enhancement

```typescript
// Angular: Progressively enhanced component
@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs">
      <div role="tablist" aria-label="Content sections">
        <button
          *ngFor="let tab of tabs; let i = index"
          [id]="'tab-' + i"
          role="tab"
          [attr.aria-selected]="selectedTab === i"
          [attr.aria-controls]="'panel-' + i"
          [tabIndex]="selectedTab === i ? 0 : -1"
          (click)="selectTab(i)"
          (keydown)="handleKeyDown($event, i)"
        >
          {{ tab.label }}
        </button>
      </div>

      <div
        *ngFor="let tab of tabs; let i = index"
        [id]="'panel-' + i"
        role="tabpanel"
        [attr.aria-labelledby]="'tab-' + i"
        [hidden]="selectedTab !== i"
        [tabIndex]="0"
      >
        {{ tab.content }}
      </div>
    </div>
  `
})
export class TabsComponent {
  tabs = [
    { label: 'Tab 1', content: 'Content 1' },
    { label: 'Tab 2', content: 'Content 2' },
    { label: 'Tab 3', content: 'Content 3' }
  ];
  
  selectedTab = 0;

  selectTab(index: number) {
    this.selectedTab = index;
  }

  handleKeyDown(event: KeyboardEvent, index: number) {
    let newIndex = index;

    switch (event.key) {
      case 'ArrowRight':
        newIndex = (index + 1) % this.tabs.length;
        break;
      case 'ArrowLeft':
        newIndex = (index - 1 + this.tabs.length) % this.tabs.length;
        break;
      case 'Home':
        newIndex = 0;
        break;
      case 'End':
        newIndex = this.tabs.length - 1;
        break;
      default:
        return;
    }

    event.preventDefault();
    this.selectTab(newIndex);
    
    // Focus new tab
    setTimeout(() => {
      document.getElementById(`tab-${newIndex}`)?.focus();
    }, 0);
  }
}
```

### 3. Test Early and Often

```typescript
// React: Accessibility testing utilities
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('AccessibleComponent', () => {
  it('should have no accessibility violations', async () => {
    const { container } = render(<AccessibleComponent />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('should announce loading state', () => {
    const { getByRole } = render(<AccessibleComponent />);
    const status = getByRole('status');
    expect(status).toHaveTextContent('Loading...');
  });

  it('should have proper ARIA labels', () => {
    const { getByLabelText } = render(<AccessibleComponent />);
    expect(getByLabelText('Submit form')).toBeInTheDocument();
  });
});
```

## Interview Questions

### Junior Level

1. **What is a screen reader and who uses it?**
   - Software that reads digital content aloud
   - Used by blind and visually impaired users
   - Converts text and UI elements to speech/braille

2. **Name three popular screen readers**
   - NVDA (Windows, free)
   - JAWS (Windows, commercial)
   - VoiceOver (macOS/iOS, built-in)
   - TalkBack (Android, built-in)

3. **What is alt text and why is it important?**
   - Alternative text for images
   - Announced by screen readers
   - Describes image content
   - Required for accessibility

### Mid Level

4. **Explain the difference between aria-label and aria-labelledby**
   - aria-label: Direct text label
   - aria-labelledby: References another element by ID
   - aria-labelledby overrides native labels
   - Use aria-labelledby for existing text

5. **What is a live region and when would you use it?**
   - Region that announces dynamic changes
   - aria-live="polite" or "assertive"
   - Used for status updates, errors, notifications
   - Doesn't require focus

6. **How do screen readers handle modals?**
   - aria-modal="true" restricts navigation
   - Focus moves to modal
   - Background content inert
   - Focus returns on close

### Senior Level

7. **Describe the accessibility tree and how it differs from the DOM**
   - Simplified version of DOM
   - Contains only semantically relevant information
   - Built from HTML elements and ARIA
   - What screen readers actually navigate
   - Excludes decorative elements

8. **How would you test a complex data table for screen reader accessibility?**
   - Use caption or aria-labelledby
   - scope on header cells
   - Test with table navigation commands
   - Verify row/column headers announced
   - Test sorting announcements
   - Verify cell associations

9. **What strategies would you use to make a single-page application accessible?**
   - Announce route changes
   - Manage focus on navigation
   - Use aria-live for dynamic updates
   - Implement skip links
   - Maintain focus visibility
   - Test with screen readers

10. **Explain the difference between browse mode and focus mode in screen readers**
    - Browse mode: Document navigation
    - Focus mode: Form/widget interaction
    - Automatic switching for form fields
    - Different keyboard commands available
    - Important for custom widgets

## Key Takeaways

1. **Test with actual screen readers** - automated tools catch only 30-40% of issues
2. **Semantic HTML first** - use native elements before ARIA
3. **ARIA is a supplement** - not a replacement for proper HTML
4. **Test on multiple platforms** - NVDA, JAWS, VoiceOver minimum
5. **Focus management is critical** - especially for SPAs and modals
6. **Announce dynamic changes** - use live regions appropriately
7. **Labels are non-negotiable** - all form fields and buttons need them
8. **Test without seeing the screen** - close your eyes and navigate
9. **User test with real users** - screen reader users when possible
10. **Make it part of development** - not an afterthought

## Resources

### Screen Reader Downloads
- [NVDA](https://www.nvaccess.org/download/) - Free Windows screen reader
- [JAWS Trial](https://www.freedomscientific.com/downloads/jaws) - Commercial Windows screen reader
- [VoiceOver](https://support.apple.com/guide/voiceover/) - Built into macOS/iOS

### Documentation
- [NVDA User Guide](https://www.nvaccess.org/files/nvda/documentation/userGuide.html)
- [JAWS Documentation](https://www.freedomscientific.com/training/jaws/)
- [VoiceOver User Guide](https://support.apple.com/guide/voiceover/)
- [WebAIM: Screen Reader Testing](https://webaim.org/articles/screenreader_testing/)

### Testing Tools
- [Screen Reader User Survey](https://webaim.org/projects/screenreadersurvey9/) - Usage statistics
- [Accessibility Insights](https://accessibilityinsights.io/) - Testing extension
- [axe DevTools](https://www.deque.com/axe/devtools/) - Automated testing

### Learning Resources
- [A11ycasts YouTube Series](https://www.youtube.com/playlist?list=PLNYkxOF6rcICWx0C9LVWWVqvHlYJyqw7g)
- [WebAIM: Using NVDA to Evaluate Accessibility](https://webaim.org/articles/nvda/)
- [Deque University](https://dequeuniversity.com/) - Paid courses
