# ARIA Roles, States, and Properties

## The Idea

**In plain English:** ARIA (Accessible Rich Internet Applications) is a set of extra labels you can add to HTML elements to describe what they are and what they're doing — so that assistive tools like screen readers (software that reads the screen aloud for people who are blind) can understand and explain your web page correctly.

**Real-world analogy:** Imagine a busy airport where every area has a sign overhead — "Check-in", "Security", "Gate B7", "Restrooms". Now picture a blind traveler with a cane who also has a small earpiece that reads those signs aloud. Without the signs, every area looks like the same blank floor. With them, the traveler knows exactly where they are and what they can do in each zone.
- The blank floor areas = plain `<div>` elements with no meaning
- The overhead signs = ARIA roles (e.g., `role="navigation"`, `role="dialog"`)
- The earpiece reading the signs = a screen reader announcing ARIA to the user
- A sign that flips between "Gate Open" and "Boarding Now" = an ARIA state (e.g., `aria-expanded="true"`)

---

## Learning Objectives
- Understand what ARIA is and when to use it
- Master ARIA roles for defining element types and purposes
- Learn ARIA states and properties for describing element behavior
- Know when to use native HTML vs when ARIA is necessary
- Apply ARIA correctly to custom interactive components

## Introduction to ARIA

ARIA (Accessible Rich Internet Applications) is a set of attributes that define ways to make web content and web applications more accessible to people with disabilities. ARIA supplements HTML by providing additional semantics and metadata that assistive technologies can use to better communicate with users.

ARIA was created to bridge the gap between what modern web applications need to convey and what HTML alone can express. While HTML provides semantic elements for many common patterns (buttons, links, forms), it doesn't have built-in elements for complex widgets like tabs, tree views, autocomplete fields, or application-specific controls. ARIA fills this gap by allowing developers to indicate roles, states, and properties that aren't available in HTML.

However, ARIA comes with an important principle: **The First Rule of ARIA is "Don't use ARIA."** This seemingly contradictory rule means: if you can use a native HTML element or attribute with the semantics and behavior you require already built in, then do so. ARIA should be your solution when HTML doesn't provide what you need.

## The Three Categories of ARIA

ARIA attributes fall into three categories:

```
ARIA Categories:
┌────────────────────────────────────────┐
│            ARIA Roles                   │
│  Define what an element is             │
│  Examples: button, dialog, tab         │
├────────────────────────────────────────┤
│          ARIA Properties               │
│  Define characteristics (relatively    │
│  static)                               │
│  Examples: aria-label, aria-required   │
├────────────────────────────────────────┤
│           ARIA States                  │
│  Define current conditions (dynamic)   │
│  Examples: aria-expanded, aria-checked │
└────────────────────────────────────────┘
```

## ARIA Roles

Roles define what an element is or does. There are four categories of ARIA roles:

### 1. Landmark Roles

Landmark roles identify regions of a page, allowing users to quickly navigate to different sections.

```typescript
// React: Landmark roles (most have HTML5 equivalents)
const LandmarkExamples: React.FC = () => {
  return (
    <div>
      {/* ✅ GOOD: Use semantic HTML when possible */}
      <header> {/* Implicit role="banner" */}
        <h1>Site Title</h1>
      </header>
      
      <nav> {/* Implicit role="navigation" */}
        <ul>
          <li><a href="/">Home</a></li>
        </ul>
      </nav>
      
      <main> {/* Implicit role="main" */}
        <h1>Main Content</h1>
      </main>
      
      <aside> {/* Implicit role="complementary" */}
        <h2>Related</h2>
      </aside>
      
      <footer> {/* Implicit role="contentinfo" */}
        <p>&copy; 2024</p>
      </footer>
      
      {/* When you need explicit role (no HTML5 equivalent) */}
      <div role="search">
        <form>
          <input type="search" aria-label="Search site" />
          <button type="submit">Search</button>
        </form>
      </div>
      
      {/* Multiple landmarks of same type need labels */}
      <nav aria-label="Primary navigation">
        <a href="/">Home</a>
      </nav>
      
      <nav aria-label="Footer navigation">
        <a href="/privacy">Privacy</a>
      </nav>
      
      {/* Generic region needs a label to be a landmark */}
      <section aria-labelledby="products-heading" role="region">
        <h2 id="products-heading">Featured Products</h2>
      </section>
    </div>
  );
};
```

**Available Landmark Roles:**
- `banner` - Site header (use `<header>` at top level)
- `navigation` - Navigation links (use `<nav>`)
- `main` - Primary content (use `<main>`)
- `complementary` - Supporting content (use `<aside>`)
- `contentinfo` - Site footer (use `<footer>` at top level)
- `search` - Search functionality (no HTML equivalent)
- `form` - Form landmark when labeled (use `<form>` with aria-label)
- `region` - Generic landmark when labeled (use `<section>` with aria-labelledby)

### 2. Widget Roles

Widget roles define interactive elements, especially custom controls.

```typescript
// React: Custom tab widget with widget roles
import React, { useState } from 'react';

const TabWidget: React.FC = () => {
  const [activeTab, setActiveTab] = useState(0);
  
  const tabs = [
    { label: 'Personal Info', content: 'Personal information form...' },
    { label: 'Account', content: 'Account settings...' },
    { label: 'Preferences', content: 'User preferences...' }
  ];
  
  const handleKeyDown = (e: React.KeyboardEvent, index: number) => {
    let newIndex = index;
    
    switch (e.key) {
      case 'ArrowRight':
        newIndex = (index + 1) % tabs.length;
        break;
      case 'ArrowLeft':
        newIndex = index === 0 ? tabs.length - 1 : index - 1;
        break;
      case 'Home':
        newIndex = 0;
        break;
      case 'End':
        newIndex = tabs.length - 1;
        break;
      default:
        return;
    }
    
    e.preventDefault();
    setActiveTab(newIndex);
    
    // Focus the newly activated tab
    setTimeout(() => {
      document.getElementById(`tab-${newIndex}`)?.focus();
    }, 0);
  };

  return (
    <div className="tabs">
      {/* tablist: Container for tabs */}
      <div role="tablist" aria-label="Account settings">
        {tabs.map((tab, index) => (
          <button
            key={index}
            id={`tab-${index}`}
            role="tab"
            aria-selected={activeTab === index}
            aria-controls={`panel-${index}`}
            tabIndex={activeTab === index ? 0 : -1}
            onClick={() => setActiveTab(index)}
            onKeyDown={(e) => handleKeyDown(e, index)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      
      {/* tabpanel: Content area for each tab */}
      {tabs.map((tab, index) => (
        <div
          key={index}
          id={`panel-${index}`}
          role="tabpanel"
          aria-labelledby={`tab-${index}`}
          hidden={activeTab !== index}
          tabIndex={0}
        >
          {activeTab === index && <p>{tab.content}</p>}
        </div>
      ))}
    </div>
  );
};

// Angular: Custom slider widget
@Component({
  selector: 'app-slider',
  template: `
    <div class="slider">
      <label [attr.id]="labelId">{{ label }}</label>
      
      <div
        role="slider"
        [attr.aria-labelledby]="labelId"
        [attr.aria-valuemin]="min"
        [attr.aria-valuemax]="max"
        [attr.aria-valuenow]="value"
        [attr.aria-valuetext]="valueText"
        [attr.tabindex]="0"
        (keydown)="handleKeyDown($event)"
        (mousedown)="startDrag($event)"
        class="slider-track"
      >
        <div 
          class="slider-thumb"
          [style.left.%]="percentage"
        ></div>
      </div>
      
      <output [attr.for]="labelId">{{ valueText }}</output>
    </div>
  `,
  styles: [`
    .slider-track {
      position: relative;
      width: 100%;
      height: 8px;
      background: #ddd;
      border-radius: 4px;
      cursor: pointer;
    }
    
    .slider-thumb {
      position: absolute;
      top: 50%;
      transform: translate(-50%, -50%);
      width: 20px;
      height: 20px;
      background: #005fcc;
      border-radius: 50%;
      cursor: grab;
    }
    
    [role="slider"]:focus {
      outline: 2px solid #005fcc;
      outline-offset: 2px;
    }
  `]
})
export class SliderComponent {
  @Input() label: string = 'Slider';
  @Input() min: number = 0;
  @Input() max: number = 100;
  @Input() step: number = 1;
  @Input() value: number = 50;
  @Input() formatValue?: (value: number) => string;
  @Output() valueChange = new EventEmitter<number>();
  
  labelId = `slider-${Math.random().toString(36).substr(2, 9)}`;
  
  get percentage(): number {
    return ((this.value - this.min) / (this.max - this.min)) * 100;
  }
  
  get valueText(): string {
    return this.formatValue ? this.formatValue(this.value) : `${this.value}`;
  }
  
  handleKeyDown(event: KeyboardEvent): void {
    let newValue = this.value;
    
    switch (event.key) {
      case 'ArrowRight':
      case 'ArrowUp':
        newValue = Math.min(this.value + this.step, this.max);
        break;
      case 'ArrowLeft':
      case 'ArrowDown':
        newValue = Math.max(this.value - this.step, this.min);
        break;
      case 'Home':
        newValue = this.min;
        break;
      case 'End':
        newValue = this.max;
        break;
      case 'PageUp':
        newValue = Math.min(this.value + this.step * 10, this.max);
        break;
      case 'PageDown':
        newValue = Math.max(this.value - this.step * 10, this.min);
        break;
      default:
        return;
    }
    
    event.preventDefault();
    this.setValue(newValue);
  }
  
  startDrag(event: MouseEvent): void {
    // Implementation for mouse drag
    event.preventDefault();
  }
  
  private setValue(newValue: number): void {
    this.value = newValue;
    this.valueChange.emit(newValue);
  }
}
```

**Common Widget Roles:**
- `button` - Clickable button (use `<button>` instead)
- `checkbox` - Checkbox (use `<input type="checkbox">` instead)
- `radio` - Radio button (use `<input type="radio">` instead)
- `textbox` - Text input (use `<input>` or `<textarea>` instead)
- `slider` - Range selector (use `<input type="range">` or custom)
- `switch` - On/off switch (use `<input type="checkbox" role="switch">`)
- `tab` - Tab in a tablist
- `tabpanel` - Content panel for a tab
- `tablist` - Container for tabs
- `menu` - Menu widget
- `menuitem` - Item in a menu
- `menubar` - Menu bar
- `tree` - Tree view
- `treeitem` - Item in a tree
- `grid` - Interactive grid
- `gridcell` - Cell in a grid
- `listbox` - Listbox widget
- `option` - Option in a listbox
- `combobox` - Combo box (combination of textbox and listbox)
- `dialog` - Dialog or modal (use `<dialog>` when appropriate)
- `alertdialog` - Alert dialog requiring user response
- `tooltip` - Tooltip

### 3. Document Structure Roles

Document structure roles describe the structure of content sections.

```typescript
// React: Document structure roles
const DocumentStructure: React.FC = () => {
  return (
    <article>
      <h1>Understanding ARIA</h1>
      
      {/* heading: Heading at specific level */}
      <div role="heading" aria-level={2}>
        Introduction
      </div>
      {/* Better: <h2>Introduction</h2> */}
      
      {/* list and listitem: List structure */}
      <div role="list">
        <div role="listitem">First item</div>
        <div role="listitem">Second item</div>
      </div>
      {/* Better: <ul><li>...</li></ul> */}
      
      {/* table structure roles */}
      <div role="table" aria-label="User data">
        <div role="rowgroup">
          <div role="row">
            <div role="columnheader">Name</div>
            <div role="columnheader">Email</div>
          </div>
        </div>
        <div role="rowgroup">
          <div role="row">
            <div role="cell">John Doe</div>
            <div role="cell">john@example.com</div>
          </div>
        </div>
      </div>
      {/* Better: Use actual <table> */}
      
      {/* group: Generic group of elements */}
      <div role="group" aria-labelledby="radio-legend">
        <div id="radio-legend">Choose size:</div>
        <label><input type="radio" name="size" /> Small</label>
        <label><input type="radio" name="size" /> Medium</label>
      </div>
      {/* Better: <fieldset><legend>...</legend></fieldset> */}
      
      {/* separator: Visual or semantic separator */}
      <hr /> {/* Has implicit role="separator" */}
      
      {/* img: Image */}
      <div role="img" aria-label="Company logo">
        <svg>...</svg>
      </div>
      {/* Better: <img alt="Company logo" /> */}
      
      {/* article, section have implicit roles */}
      <section aria-labelledby="features">
        <h2 id="features">Features</h2>
        <p>Content...</p>
      </section>
    </article>
  );
};
```

**Document Structure Roles:**
- `article` - Self-contained content (use `<article>`)
- `definition` - Definition of a term (use `<dd>`)
- `directory` - List of references (deprecated, use list)
- `document` - Document content
- `feed` - Scrollable list of articles
- `figure` - Image with caption (use `<figure>`)
- `group` - Set of related elements
- `heading` - Heading (use `<h1>`-`<h6>`)
- `img` - Image container (use `<img>`)
- `list` - List (use `<ul>`, `<ol>`)
- `listitem` - List item (use `<li>`)
- `math` - Mathematical expression
- `note` - Parenthetic content
- `presentation` / `none` - Remove semantics
- `row` - Row in table/grid (use `<tr>`)
- `rowgroup` - Group of rows (use `<thead>`, `<tbody>`)
- `separator` - Separator (use `<hr>`)
- `table` - Table (use `<table>`)
- `term` - Term being defined (use `<dt>`)
- `toolbar` - Toolbar of controls

### 4. Live Region Roles

Live region roles announce dynamic content changes to screen readers.

```typescript
// React: Live region roles
import React, { useState } from 'react';

const LiveRegionExamples: React.FC = () => {
  const [status, setStatus] = useState('');
  const [alert, setAlert] = useState('');
  const [log, setLog] = useState<string[]>([]);
  const [timer, setTimer] = useState(60);

  const saveData = () => {
    setStatus('Saving...');
    setTimeout(() => {
      setStatus('Changes saved successfully');
    }, 1000);
  };

  const showError = () => {
    setAlert('Error: Invalid email address format');
  };

  const addLogEntry = (message: string) => {
    setLog(prev => [...prev, `${new Date().toLocaleTimeString()}: ${message}`]);
  };

  return (
    <div>
      <h1>Live Region Examples</h1>
      
      {/* alert: Important, time-sensitive message */}
      {alert && (
        <div role="alert">
          {alert}
        </div>
      )}
      
      {/* status: Advisory, non-critical status */}
      <div role="status" aria-live="polite" aria-atomic="true">
        {status}
      </div>
      
      {/* log: Sequential information */}
      <div role="log" aria-live="polite" aria-atomic="false">
        {log.map((entry, index) => (
          <div key={index}>{entry}</div>
        ))}
      </div>
      
      {/* timer: Countdown or elapsed time */}
      <div role="timer" aria-live="off" aria-atomic="true">
        Time remaining: {timer} seconds
      </div>
      
      {/* marquee: Scrolling information (rarely used) */}
      <div role="marquee" aria-live="off">
        Breaking news: ...
      </div>
      
      <button onClick={saveData}>Save Changes</button>
      <button onClick={showError}>Show Error</button>
      <button onClick={() => addLogEntry('Action performed')}>
        Add Log Entry
      </button>
    </div>
  );
};

// Angular: Status messages and notifications
@Component({
  selector: 'app-form-with-feedback',
  template: `
    <form (submit)="handleSubmit($event)">
      <div>
        <label for="email">Email</label>
        <input
          type="email"
          id="email"
          [(ngModel)]="email"
          name="email"
          [attr.aria-invalid]="hasError"
          [attr.aria-describedby]="hasError ? 'email-error' : null"
        />
        
        <!-- Error message with alert role -->
        <div
          *ngIf="hasError"
          id="email-error"
          role="alert"
          class="error"
        >
          {{ errorMessage }}
        </div>
      </div>
      
      <button type="submit" [disabled]="isSubmitting">
        {{ isSubmitting ? 'Submitting...' : 'Submit' }}
      </button>
      
      <!-- Status message for submission feedback -->
      <div
        *ngIf="statusMessage"
        role="status"
        aria-live="polite"
        class="status"
      >
        {{ statusMessage }}
      </div>
    </form>
    
    <!-- Progress updates -->
    <div
      *ngIf="uploadProgress > 0"
      role="status"
      aria-live="polite"
      aria-atomic="true"
    >
      Upload progress: {{ uploadProgress }}%
    </div>
  `,
  styles: [`
    .error {
      color: #c00;
      margin-top: 0.25rem;
    }
    
    .status {
      color: #0a6e0a;
      margin-top: 1rem;
    }
  `]
})
export class FormWithFeedbackComponent {
  email = '';
  hasError = false;
  errorMessage = '';
  statusMessage = '';
  isSubmitting = false;
  uploadProgress = 0;
  
  handleSubmit(event: Event): void {
    event.preventDefault();
    
    if (!this.validateEmail(this.email)) {
      this.hasError = true;
      this.errorMessage = 'Please enter a valid email address';
      return;
    }
    
    this.hasError = false;
    this.isSubmitting = true;
    this.statusMessage = 'Submitting form...';
    
    // Simulate submission
    setTimeout(() => {
      this.isSubmitting = false;
      this.statusMessage = 'Form submitted successfully!';
      
      // Clear status after 3 seconds
      setTimeout(() => {
        this.statusMessage = '';
      }, 3000);
    }, 1500);
  }
  
  private validateEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

**Live Region Roles:**
- `alert` - Important, time-sensitive message (implicit aria-live="assertive")
- `status` - Advisory information (implicit aria-live="polite")
- `log` - Sequential information like chat logs (implicit aria-live="polite")
- `marquee` - Non-essential scrolling information (implicit aria-live="off")
- `timer` - Countdown or elapsed time (implicit aria-live="off")

## ARIA Properties

ARIA properties describe characteristics of elements. They're relatively static (though they can change).

### Labeling Properties

```typescript
// React: Labeling properties
const LabelingExamples: React.FC = () => {
  return (
    <div>
      {/* aria-label: Direct label for element */}
      <button aria-label="Close dialog">
        ×
      </button>
      
      {/* aria-labelledby: References element(s) that label this element */}
      <section aria-labelledby="products-heading">
        <h2 id="products-heading">Featured Products</h2>
        <p>Our best sellers...</p>
      </section>
      
      {/* aria-describedby: References element(s) that describe this element */}
      <input
        type="password"
        id="password"
        aria-describedby="password-requirements"
      />
      <div id="password-requirements">
        Password must be at least 8 characters and include a number.
      </div>
      
      {/* Multiple references with space-separated IDs */}
      <button
        aria-labelledby="button-icon button-text"
        aria-describedby="button-hint"
      >
        <span id="button-icon" aria-hidden="true">💾</span>
        <span id="button-text">Save</span>
      </button>
      <div id="button-hint">Saves your current progress</div>
    </div>
  );
};
```

### Relationship Properties

```typescript
// React: Relationship properties
const RelationshipExamples: React.FC = () => {
  const [isExpanded, setIsExpanded] = useState(false);

  return (
    <div>
      {/* aria-controls: Element(s) controlled by this element */}
      <button
        aria-expanded={isExpanded}
        aria-controls="dropdown-menu"
        onClick={() => setIsExpanded(!isExpanded)}
      >
        Options
      </button>
      <ul id="dropdown-menu" hidden={!isExpanded}>
        <li>Option 1</li>
        <li>Option 2</li>
      </ul>
      
      {/* aria-owns: Defines parent-child relationship */}
      <div role="tree" aria-owns="tree-child-1 tree-child-2">
        <div role="treeitem">Parent Item</div>
      </div>
      <div id="tree-child-1" role="treeitem">Child 1</div>
      <div id="tree-child-2" role="treeitem">Child 2</div>
      
      {/* aria-activedescendant: Current active child in composite widget */}
      <div
        role="listbox"
        tabIndex={0}
        aria-activedescendant="option-2"
      >
        <div role="option" id="option-1">Option 1</div>
        <div role="option" id="option-2">Option 2 (active)</div>
        <div role="option" id="option-3">Option 3</div>
      </div>
      
      {/* aria-flowto: Suggests reading order */}
      <div id="content-1" aria-flowto="content-3">
        First section
      </div>
      <div id="content-2">
        Sidebar
      </div>
      <div id="content-3">
        Second section (continues from first)
      </div>
    </div>
  );
};
```

### Widget Properties

```typescript
// React: Widget properties
const WidgetProperties: React.FC = () => {
  return (
    <div>
      {/* aria-autocomplete: Type of autocomplete */}
      <input
        type="text"
        role="combobox"
        aria-autocomplete="list"
        aria-controls="suggestions"
      />
      
      {/* aria-haspopup: Indicates popup element type */}
      <button aria-haspopup="menu">
        File Menu
      </button>
      
      <button aria-haspopup="dialog">
        Open Settings
      </button>
      
      {/* aria-level: Hierarchical level */}
      <div role="heading" aria-level={2}>
        Subheading
      </div>
      {/* Better: <h2>Subheading</h2> */}
      
      {/* aria-multiselectable: Multiple selection allowed */}
      <ul role="listbox" aria-multiselectable="true">
        <li role="option" aria-selected="true">Item 1</li>
        <li role="option" aria-selected="true">Item 2</li>
        <li role="option">Item 3</li>
      </ul>
      
      {/* aria-orientation: Orientation of widget */}
      <div role="tablist" aria-orientation="vertical">
        <button role="tab">Tab 1</button>
        <button role="tab">Tab 2</button>
      </div>
      
      {/* aria-placeholder: Placeholder text */}
      <div
        role="textbox"
        contentEditable
        aria-placeholder="Enter your comment"
      />
      {/* Better: <input placeholder="..." /> */}
      
      {/* aria-readonly: Element is not editable */}
      <input
        type="text"
        value="Read-only value"
        aria-readonly="true"
        readOnly
      />
      
      {/* aria-required: Input is required */}
      <input
        type="text"
        aria-required="true"
        required
      />
      
      {/* aria-valuemin, aria-valuemax, aria-valuenow, aria-valuetext */}
      <div
        role="slider"
        aria-valuemin={0}
        aria-valuemax={100}
        aria-valuenow={50}
        aria-valuetext="50 percent"
        tabIndex={0}
      />
      
      {/* aria-multiline: Textbox accepts multiple lines */}
      <div
        role="textbox"
        aria-multiline="true"
        contentEditable
      />
      {/* Better: <textarea /> */}
    </div>
  );
};
```

### Other Properties

```typescript
// React: Miscellaneous properties
const OtherProperties: React.FC = () => {
  return (
    <div>
      {/* aria-atomic: Announce entire region on change */}
      <div role="status" aria-live="polite" aria-atomic="true">
        Items 1-10 of 50
      </div>
      
      {/* aria-busy: Element/region is being updated */}
      <div aria-busy="true">
        Loading content...
      </div>
      
      {/* aria-live: How to announce live region changes */}
      <div aria-live="polite">
        Status updates appear here
      </div>
      
      <div aria-live="assertive">
        Critical alerts appear here
      </div>
      
      {/* aria-relevant: What changes to announce */}
      <div
        role="log"
        aria-live="polite"
        aria-relevant="additions text"
      >
        Log entries
      </div>
      
      {/* aria-dropeffect: Drag-and-drop operation (deprecated in ARIA 1.1) */}
      <div aria-dropeffect="move">
        Drop zone
      </div>
      
      {/* aria-grabbed: Element is grabbed for drag (deprecated in ARIA 1.1) */}
      <div aria-grabbed="false" draggable="true">
        Draggable item
      </div>
      
      {/* aria-hidden: Hide from accessibility tree */}
      <span aria-hidden="true">★</span>
      <span className="sr-only">5 star rating</span>
      
      {/* aria-keyshortcuts: Keyboard shortcuts */}
      <button aria-keyshortcuts="Control+S">
        Save (Ctrl+S)
      </button>
      
      {/* aria-roledescription: Human-readable role */}
      <div role="article" aria-roledescription="blog post">
        Blog content
      </div>
    </div>
  );
};
```

## ARIA States

ARIA states describe the current condition of elements. They change frequently in response to user interaction.

```typescript
// React: ARIA states
import React, { useState } from 'react';

const ARIAStates: React.FC = () => {
  const [isChecked, setIsChecked] = useState(false);
  const [isExpanded, setIsExpanded] = useState(false);
  const [isPressedToggle, setIsPressedToggle] = useState(false);
  const [isSelectedTab, setIsSelectedTab] = useState(0);
  const [isDisabled, setIsDisabled] = useState(false);
  const [hasError, setHasError] = useState(false);

  return (
    <div>
      {/* aria-checked: Checked state of checkbox/radio/switch */}
      <div
        role="checkbox"
        aria-checked={isChecked}
        tabIndex={0}
        onClick={() => setIsChecked(!isChecked)}
      >
        {isChecked ? '☑' : '☐'} Custom Checkbox
      </div>
      {/* Better: <input type="checkbox" checked={isChecked} /> */}
      
      {/* aria-checked with "mixed" state for indeterminate */}
      <div role="checkbox" aria-checked="mixed">
        ☒ Partially selected
      </div>
      
      {/* aria-current: Current item in set */}
      <nav>
        <a href="/page1" aria-current="page">Current Page</a>
        <a href="/page2">Other Page</a>
      </nav>
      
      {/* aria-disabled: Element is disabled */}
      <button aria-disabled={isDisabled} onClick={() => {}}>
        {isDisabled ? 'Disabled' : 'Enabled'} Button
      </button>
      
      {/* aria-expanded: Expandable element state */}
      <button
        aria-expanded={isExpanded}
        aria-controls="expandable-content"
        onClick={() => setIsExpanded(!isExpanded)}
      >
        {isExpanded ? 'Collapse' : 'Expand'}
      </button>
      <div id="expandable-content" hidden={!isExpanded}>
        Expandable content
      </div>
      
      {/* aria-hidden: Hidden from accessibility tree */}
      <div>
        <span aria-hidden="true">→</span>
        <span className="sr-only">Next</span>
      </div>
      
      {/* aria-invalid: Validation error state */}
      <input
        type="email"
        aria-invalid={hasError}
        aria-describedby={hasError ? 'email-error' : undefined}
      />
      {hasError && (
        <div id="email-error" role="alert">
          Please enter a valid email
        </div>
      )}
      
      {/* aria-pressed: Toggle button state */}
      <button
        aria-pressed={isPressedToggle}
        onClick={() => setIsPressedToggle(!isPressedToggle)}
      >
        {isPressedToggle ? 'On' : 'Off'}
      </button>
      
      {/* aria-selected: Selected state in widget */}
      <div role="tablist">
        <button
          role="tab"
          aria-selected={isSelectedTab === 0}
          onClick={() => setIsSelectedTab(0)}
        >
          Tab 1
        </button>
        <button
          role="tab"
          aria-selected={isSelectedTab === 1}
          onClick={() => setIsSelectedTab(1)}
        >
          Tab 2
        </button>
      </div>
    </div>
  );
};
```

**All ARIA States:**
- `aria-busy` - Element is being updated
- `aria-checked` - Checked state (true/false/mixed)
- `aria-current` - Current item in a set
- `aria-disabled` - Element is disabled
- `aria-expanded` - Element is expanded or collapsed
- `aria-grabbed` - Element is grabbed for drag (deprecated)
- `aria-hidden` - Element is hidden from accessibility tree
- `aria-invalid` - Input has validation error
- `aria-pressed` - Toggle button state
- `aria-selected` - Element is selected

## When to Use ARIA

### Use ARIA When:

```typescript
// 1. Custom widgets without native HTML equivalent
const CustomTreeView: React.FC = () => {
  return (
    <ul role="tree">
      <li role="treeitem" aria-expanded="true">
        <span>Folder 1</span>
        <ul role="group">
          <li role="treeitem">File 1.1</li>
        </ul>
      </li>
    </ul>
  );
};

// 2. Dynamic content changes
const DynamicNotification: React.FC = () => {
  const [message, setMessage] = useState('');
  
  return (
    <div role="status" aria-live="polite" aria-atomic="true">
      {message}
    </div>
  );
};

// 3. Providing additional context
const IconButton: React.FC = () => {
  return (
    <button aria-label="Delete item">
      <span aria-hidden="true">🗑️</span>
    </button>
  );
};

// 4. Complex relationships
const ComboBox: React.FC = () => {
  return (
    <>
      <input
        role="combobox"
        aria-expanded="true"
        aria-controls="listbox-1"
        aria-activedescendant="option-1"
      />
      <ul id="listbox-1" role="listbox">
        <li role="option" id="option-1">Option 1</li>
      </ul>
    </>
  );
};
```

### Don't Use ARIA When:

```typescript
// ❌ BAD: Native HTML element exists
<div role="button" tabIndex={0} onClick={handler}>
  Click me
</div>

// ✅ GOOD: Use native button
<button onClick={handler}>Click me</button>

// ❌ BAD: Redundant ARIA
<nav role="navigation">
  <ul role="list">
    <li role="listitem">
      <a href="/" role="link">Home</a>
    </li>
  </ul>
</nav>

// ✅ GOOD: Semantic HTML provides roles
<nav>
  <ul>
    <li><a href="/">Home</a></li>
  </ul>
</nav>

// ❌ BAD: ARIA to fix invalid HTML
<div>
  <div role="row">
    <div role="cell">Data</div>
  </div>
</div>

// ✅ GOOD: Use proper HTML
<table>
  <tr>
    <td>Data</td>
  </tr>
</table>
```

## Common Mistakes

### Mistake 1: Conflicting Roles

```typescript
// ❌ BAD: Overriding native semantics
<button role="link">This is confusing</button>

// ✅ GOOD: Use the right element
<a href="/page">Link</a>
<button onClick={handler}>Button</button>
```

### Mistake 2: Missing Keyboard Support

```typescript
// ❌ BAD: ARIA without keyboard support
<div role="button" onClick={handler}>
  Click me
</div>

// ✅ GOOD: Full keyboard support
<div
  role="button"
  tabIndex={0}
  onClick={handler}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      handler();
    }
  }}
>
  Click me
</div>

// ✅ BEST: Native button
<button onClick={handler}>Click me</button>
```

### Mistake 3: Incorrect aria-label Placement

```typescript
// ❌ BAD: aria-label on non-labelable element
<div aria-label="Section">
  <p>Content</p>
</div>

// ✅ GOOD: Use heading or section with role
<section aria-labelledby="section-heading">
  <h2 id="section-heading">Section</h2>
  <p>Content</p>
</section>
```

### Mistake 4: Not Updating Dynamic States

```typescript
// ❌ BAD: Static aria-expanded
<button aria-expanded="false" onClick={toggle}>
  Toggle
</button>

// ✅ GOOD: Dynamic state
const [isExpanded, setIsExpanded] = useState(false);
<button
  aria-expanded={isExpanded}
  onClick={() => setIsExpanded(!isExpanded)}
>
  Toggle
</button>
```

## Best Practices

1. **Follow the first rule**: Don't use ARIA if semantic HTML suffices
2. **Don't override native semantics**: Avoid conflicting roles
3. **All interactive ARIA controls must be keyboard accessible**: Add keyboard handlers
4. **Provide accessible names**: Use aria-label or aria-labelledby
5. **Update states dynamically**: Keep ARIA states synchronized with UI
6. **Test with screen readers**: Automated tools miss ARIA issues
7. **Use ARIA landmarks**: But prefer HTML5 semantic elements
8. **Hide decorative content**: Use aria-hidden="true" for icons
9. **Provide descriptions**: Use aria-describedby for additional context
10. **Validate ARIA**: Use accessibility linters and validators

## Interview Questions

**Q1: What is ARIA and when should you use it?**

**A:** ARIA (Accessible Rich Internet Applications) is a set of attributes that provide additional semantics and metadata to make web applications accessible to assistive technologies. ARIA defines roles, states, and properties that describe the behavior and relationships of interface elements.

**When to use ARIA:**
- Building custom widgets without native HTML equivalents (tabs, tree views, sliders)
- Creating dynamic content that changes without page reload (live regions)
- Providing additional context not conveyed by HTML alone (aria-label for icon buttons)
- Defining complex relationships between elements (aria-controls, aria-owns)

**The First Rule of ARIA is "Don't use ARIA"** - meaning you should use native HTML semantic elements with built-in accessibility features whenever possible. ARIA is necessary only when HTML doesn't provide the semantics or behavior you need.

**Q2: What's the difference between aria-label, aria-labelledby, and aria-describedby?**

**A:**

**aria-label**: Provides a direct text label for an element. The text is not visible on screen but is announced by screen readers.
```typescript
<button aria-label="Close dialog">×</button>
```
Use when: The element has no visible label or the visible content isn't descriptive enough.

**aria-labelledby**: References one or more elements whose text content serves as the label. Multiple IDs can be space-separated.
```typescript
<section aria-labelledby="section-title">
  <h2 id="section-title">Products</h2>
</section>
```
Use when: Visible text already exists that can serve as the label.

**aria-describedby**: References one or more elements that provide additional description or context.
```typescript
<input
  type="password"
  aria-describedby="password-requirements"
/>
<div id="password-requirements">
  Must be 8+ characters with a number
</div>
```
Use when: Additional information exists that explains or provides context for the element.

**Priority order**: aria-labelledby overrides aria-label, which overrides native labels. aria-describedby is supplementary and doesn't override labels.

**Q3: What are ARIA live regions and when should you use them?**

**A:** ARIA live regions announce dynamic content changes to screen readers without requiring user focus to move. They're essential for single-page applications where content updates without page reload.

**Types of live regions:**

1. **`role="alert"`**: For important, time-sensitive messages (implicit aria-live="assertive")
```typescript
<div role="alert">Error: Form submission failed</div>
```

2. **`role="status"`**: For advisory, non-critical updates (implicit aria-live="polite")
```typescript
<div role="status">Changes saved</div>
```

3. **`role="log"`**: For sequential information like chat messages
```typescript
<div role="log" aria-live="polite">
  <div>User joined</div>
  <div>User sent message</div>
</div>
```

4. **`aria-live="polite"`**: Announces when screen reader is idle
5. **`aria-live="assertive"`**: Interrupts current announcements (use sparingly)
6. **`aria-live="off"`**: No announcements (default)

**Additional properties:**
- **`aria-atomic="true"`**: Announce entire region vs just changes
- **`aria-relevant`**: What changes to announce (additions, removals, text, all)

**Use live regions when:**
- Form submission results need announcing
- Loading states change
- Search results update
- Notifications appear
- Timer/countdown updates
- Chat messages arrive

**Don't overuse**: Too many live region announcements create noise and confusion.

**Q4: How do you make a custom select/dropdown accessible?**

**A:** A custom select requires specific ARIA patterns and keyboard support:

```typescript
const AccessibleSelect: React.FC = () => {
  const [isOpen, setIsOpen] = useState(false);
  const [selected, setSelected] = useState(0);
  const [activeOption, setActiveOption] = useState(0);
  const options = ['Option 1', 'Option 2', 'Option 3'];

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        if (isOpen) {
          setActiveOption((activeOption + 1) % options.length);
        } else {
          setIsOpen(true);
        }
        break;
      case 'ArrowUp':
        e.preventDefault();
        if (isOpen) {
          setActiveOption(
            activeOption === 0 ? options.length - 1 : activeOption - 1
          );
        }
        break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        if (isOpen) {
          setSelected(activeOption);
          setIsOpen(false);
        } else {
          setIsOpen(true);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        break;
    }
  };

  return (
    <div>
      <label id="select-label">Choose option:</label>
      <div
        role="combobox"
        aria-labelledby="select-label"
        aria-expanded={isOpen}
        aria-controls="listbox-1"
        aria-activedescendant={`option-${activeOption}`}
        tabIndex={0}
        onKeyDown={handleKeyDown}
        onClick={() => setIsOpen(!isOpen)}
      >
        {options[selected]}
      </div>
      
      {isOpen && (
        <ul id="listbox-1" role="listbox" tabIndex={-1}>
          {options.map((option, index) => (
            <li
              key={index}
              id={`option-${index}`}
              role="option"
              aria-selected={activeOption === index}
              onClick={() => {
                setSelected(index);
                setIsOpen(false);
              }}
            >
              {option}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

**Required elements:**
- `role="combobox"` on trigger
- `aria-expanded` to indicate state
- `aria-controls` pointing to listbox
- `aria-activedescendant` for keyboard navigation
- `role="listbox"` on container
- `role="option"` on items
- `aria-selected` on active option
- Label via `aria-labelledby` or `aria-label`

**Keyboard support:**
- Down Arrow: Open listbox or move to next option
- Up Arrow: Move to previous option
- Enter/Space: Select option and close
- Escape: Close listbox
- Type ahead (optional): Jump to matching option

**Q5: What's the difference between aria-hidden="true" and display: none?**

**A:**

**`aria-hidden="true"`**:
- Removes element from accessibility tree only
- Element remains visible on screen
- Still in document flow
- Receives mouse clicks
- Can be focused with mouse (if focusable)
- Use for decorative content that's visible but shouldn't be announced

```typescript
// Icon that's visible but redundant for screen readers
<button>
  <span aria-hidden="true">🗑️</span>
  Delete
</button>
```

**`display: none` or `hidden` attribute**:
- Removes element from both visual and accessibility tree
- Not visible on screen
- Not in document flow
- Cannot receive any interactions
- Cannot be focused
- Use for truly hidden content

```typescript
// Content that shouldn't be visible or announced
<div hidden={!isOpen}>
  Conditional content
</div>
```

**Important warning**: Don't use `aria-hidden="true"` on focusable elements:
```typescript
// ❌ BAD: Focusable but hidden from screen readers
<button aria-hidden="true">Click me</button>

// This creates a confusing experience where keyboard users can
// focus an element but screen readers don't announce it
```

**Best practices:**
- Use `aria-hidden="true"` for decorative icons with adjacent text
- Use `display: none` for truly hidden content
- Never hide focusable content from screen readers
- Don't put interactive content inside `aria-hidden` containers

**Q6: When should you use role="presentation" or role="none"?**

**A:** `role="presentation"` and `role="none"` (synonyms) remove the semantic meaning of an element while keeping it in the DOM and accessibility tree as generic content.

**Use cases:**

1. **Layout tables** (not data tables):
```typescript
<table role="presentation">
  <tr>
    <td>Left column</td>
    <td>Right column</td>
  </tr>
</table>
```

2. **Removing list semantics** when styled as navigation or buttons:
```typescript
<ul role="presentation">
  <li><button>Action 1</button></li>
  <li><button>Action 2</button></li>
</ul>
```

3. **Image containers** where the image itself has proper alt text:
```typescript
<div role="presentation">
  <img src="photo.jpg" alt="Descriptive text" />
</div>
```

**What it does:**
- Removes the element's implicit role
- Child elements still have their semantics
- Element still visible and in DOM
- Different from `aria-hidden="true"` which removes entire subtree

**What it doesn't do:**
- Doesn't hide focusable elements
- Doesn't prevent keyboard interaction
- Doesn't hide child elements

**When NOT to use:**
- Real data tables (use proper table markup)
- Semantic lists of related items
- When you want to hide content (use `aria-hidden` or `display: none`)
- On interactive elements that need their role

**Better alternatives often exist:**
```typescript
// ❌ Removing semantics
<ul role="presentation">
  <li><button>Action</button></li>
</ul>

// ✅ Better: Use appropriate container
<div>
  <button>Action</button>
</div>
```

## Key Takeaways

- ARIA provides roles, states, and properties to make custom widgets accessible
- First rule of ARIA: "Don't use ARIA" - prefer native HTML semantic elements
- ARIA has four role categories: landmark, widget, document structure, live region
- Roles define what an element is (relatively static)
- States describe current conditions (change frequently with interaction)
- Properties describe characteristics (mostly static)
- All custom interactive ARIA controls must be keyboard accessible
- ARIA live regions announce dynamic changes without moving focus
- aria-label, aria-labelledby, and aria-describedby serve different labeling purposes
- Never override native semantics with conflicting ARIA roles
- Test with actual screen readers - automated tools miss many ARIA issues
- aria-hidden="true" hides from screen readers but keeps visible
- Update ARIA states dynamically as UI changes
- Provide keyboard support matching expected patterns for each widget type
- ARIA is necessary for SPAs with dynamic content and custom widgets

## Resources

### Official Specifications
- [WAI-ARIA 1.2](https://www.w3.org/TR/wai-aria-1.2/) - Official W3C specification
- [ARIA Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/) - Patterns and examples
- [ARIA in HTML](https://www.w3.org/TR/html-aria/) - When to use ARIA in HTML
- [Using ARIA](https://www.w3.org/TR/using-aria/) - Practical guide

### Pattern Libraries
- [Inclusive Components](https://inclusive-components.design/) - Accessible component patterns
- [A11y Style Guide](https://a11y-style-guide.com/style-guide/) - Accessible component examples
- [Reach UI](https://reach.tech/) - Accessible React components
- [Radix UI](https://www.radix-ui.com/) - Unstyled accessible components

### Testing Tools
- [axe DevTools](https://www.deque.com/axe/devtools/) - ARIA testing
- [ARIA DevTools](https://chrome.google.com/webstore/detail/aria-devtools) - Visualize ARIA
- [Accessibility Insights](https://accessibilityinsights.io/) - Microsoft's testing tool

### Learning Resources
- [MDN ARIA Guide](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA) - Comprehensive reference
- [The A11Y Project](https://www.a11yproject.com/) - Accessibility resources
- [WebAIM Articles](https://webaim.org/articles/) - In-depth articles
- [Deque University](https://dequeuniversity.com/) - Training courses
