# Angular Template Syntax

## Learning Objectives
- Master interpolation and template expression syntax in Angular
- Understand property, attribute, class, and style bindings
- Implement event binding and custom event handlers
- Use two-way data binding with [(ngModel)] and other form elements
- Work with template reference variables and their scope
- Apply safe navigation operator and nullish coalescing in templates
- Understand template expression context and limitations
- Debug common template syntax errors and binding issues

## Introduction

Angular's template syntax is a powerful extension of HTML that enables you to bind data from your component class to the view and respond to user interactions. Unlike traditional HTML, Angular templates are dynamic and reactive, automatically updating the DOM when underlying data changes. The template syntax provides multiple binding mechanisms, each optimized for specific use cases, from simple text interpolation to complex two-way data flow.

Understanding Angular's template syntax is fundamental to building interactive applications. The syntax is designed to be intuitive for developers familiar with HTML while providing the power needed for complex data binding scenarios. Angular's compiler processes these templates at build time, generating efficient JavaScript code that updates the DOM with minimal overhead.

## Interpolation

Interpolation is the simplest form of data binding in Angular, allowing you to embed component property values directly into your template using double curly braces `{{ }}`.

### Basic Interpolation

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  standalone: true,
  template: `
    <div class="profile">
      <h1>{{ userName }}</h1>
      <p>Email: {{ userEmail }}</p>
      <p>Member since: {{ memberSince }}</p>
      <p>Posts: {{ postCount }}</p>
    </div>
  `,
  styles: [`
    .profile {
      padding: 20px;
      border: 1px solid #ddd;
      border-radius: 8px;
    }
  `]
})
export class UserProfileComponent {
  userName = 'John Doe';
  userEmail = 'john.doe@example.com';
  memberSince = '2024-01-15';
  postCount = 42;
}
```

### Template Expressions

Interpolation can contain any valid template expression, which is a subset of JavaScript:

```typescript
@Component({
  selector: 'app-calculations',
  standalone: true,
  template: `
    <div>
      <!-- Arithmetic operations -->
      <p>Total: {{ price * quantity }}</p>
      <p>Tax: {{ price * quantity * 0.08 }}</p>
      <p>Grand Total: {{ (price * quantity) * 1.08 }}</p>

      <!-- String concatenation -->
      <p>Full Name: {{ firstName + ' ' + lastName }}</p>
      
      <!-- Ternary operator -->
      <p>Status: {{ isActive ? 'Active' : 'Inactive' }}</p>
      
      <!-- Method calls -->
      <p>Formatted Date: {{ formatDate(currentDate) }}</p>
      
      <!-- Logical operations -->
      <p>{{ hasPermission && isLoggedIn ? 'Welcome' : 'Please log in' }}</p>
      
      <!-- Nullish coalescing -->
      <p>Display Name: {{ nickname ?? userName }}</p>
    </div>
  `
})
export class CalculationsComponent {
  price = 99.99;
  quantity = 3;
  firstName = 'Jane';
  lastName = 'Smith';
  isActive = true;
  currentDate = new Date();
  hasPermission = true;
  isLoggedIn = true;
  nickname: string | null = null;
  userName = 'jsmith';

  formatDate(date: Date): string {
    return date.toLocaleDateString('en-US', {
      year: 'numeric',
      month: 'long',
      day: 'numeric'
    });
  }
}
```

### Template Expression Limitations

Template expressions have intentional limitations for security and performance:

```typescript
@Component({
  selector: 'app-expression-demo',
  standalone: true,
  template: `
    <div>
      <!-- ✅ ALLOWED: Simple property access -->
      <p>{{ user.name }}</p>
      
      <!-- ✅ ALLOWED: Method calls -->
      <p>{{ getFullName() }}</p>
      
      <!-- ✅ ALLOWED: Safe operators -->
      <p>{{ user?.address?.city }}</p>
      
      <!-- ❌ NOT ALLOWED: Assignments -->
      <!-- <p>{{ user.name = 'New Name' }}</p> -->
      
      <!-- ❌ NOT ALLOWED: new, typeof, instanceof -->
      <!-- <p>{{ new Date() }}</p> -->
      
      <!-- ❌ NOT ALLOWED: Chaining with ; or , -->
      <!-- <p>{{ firstName = 'John'; lastName = 'Doe' }}</p> -->
      
      <!-- ❌ NOT ALLOWED: Increment/Decrement -->
      <!-- <p>{{ counter++ }}</p> -->
      
      <!-- ❌ NOT ALLOWED: Bitwise operators -->
      <!-- <p>{{ value | 0 }}</p> -->
    </div>
  `
})
export class ExpressionDemoComponent {
  user = {
    name: 'Alice',
    address: {
      city: 'New York'
    }
  };

  getFullName(): string {
    return `${this.user.name} (NYC)`;
  }
}
```

## Property Binding

Property binding sets an element property to a component property value using square brackets `[property]="expression"`.

### DOM Property Binding

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-property-binding',
  standalone: true,
  template: `
    <div>
      <!-- Binding to element properties -->
      <img [src]="imageUrl" [alt]="imageAlt" [width]="imageWidth">
      
      <!-- Binding to input element properties -->
      <input 
        [value]="searchTerm" 
        [placeholder]="searchPlaceholder"
        [disabled]="isSearchDisabled"
        [readonly]="isReadOnly">
      
      <!-- Binding to boolean properties -->
      <button [disabled]="!canSubmit">Submit</button>
      <input type="checkbox" [checked]="isChecked">
      
      <!-- Binding to element dimensions -->
      <div 
        [style.width.px]="containerWidth"
        [style.height.px]="containerHeight">
        Content
      </div>
      
      <!-- Binding to component inputs (covered in detail later) -->
      <app-child [childData]="parentData"></app-child>
    </div>
  `
})
export class PropertyBindingComponent {
  imageUrl = 'https://example.com/avatar.jpg';
  imageAlt = 'User Avatar';
  imageWidth = 200;
  
  searchTerm = 'Angular';
  searchPlaceholder = 'Search documentation...';
  isSearchDisabled = false;
  isReadOnly = true;
  
  canSubmit = true;
  isChecked = false;
  
  containerWidth = 400;
  containerHeight = 300;
  
  parentData = { message: 'Hello from parent' };
}
```

### Attribute Binding

Some HTML attributes don't have corresponding DOM properties. Use attribute binding with `[attr.attribute-name]`:

```typescript
@Component({
  selector: 'app-attribute-binding',
  standalone: true,
  template: `
    <div>
      <!-- ARIA attributes -->
      <button 
        [attr.aria-label]="buttonLabel"
        [attr.aria-pressed]="isPressed"
        [attr.aria-disabled]="isDisabled">
        {{ buttonText }}
      </button>
      
      <!-- Data attributes -->
      <div 
        [attr.data-user-id]="userId"
        [attr.data-role]="userRole">
        User Info
      </div>
      
      <!-- Colspan/Rowspan -->
      <table>
        <tr>
          <td [attr.colspan]="columnSpan">Wide Cell</td>
        </tr>
        <tr>
          <td [attr.rowspan]="rowSpan">Tall Cell</td>
          <td>Regular Cell</td>
        </tr>
      </table>
      
      <!-- SVG attributes -->
      <svg>
        <circle 
          [attr.cx]="circleX"
          [attr.cy]="circleY"
          [attr.r]="circleRadius"
          [attr.fill]="circleColor">
        </circle>
      </svg>
    </div>
  `
})
export class AttributeBindingComponent {
  buttonLabel = 'Submit form';
  isPressed = false;
  isDisabled = false;
  buttonText = 'Click Me';
  
  userId = 12345;
  userRole = 'admin';
  
  columnSpan = 3;
  rowSpan = 2;
  
  circleX = 50;
  circleY = 50;
  circleRadius = 40;
  circleColor = '#3498db';
}
```

### Class Binding

Angular provides multiple ways to bind CSS classes dynamically:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-class-binding',
  standalone: true,
  template: `
    <div>
      <!-- Single class binding -->
      <div [class.active]="isActive">Active State</div>
      <div [class.highlighted]="isHighlighted">Highlighted</div>
      
      <!-- Multiple classes with string -->
      <div [class]="currentClasses">Dynamic Classes</div>
      
      <!-- Multiple classes with object -->
      <div [class]="{
        'card': true,
        'card-elevated': isElevated,
        'card-primary': isPrimary,
        'card-disabled': isDisabled
      }">
        Card Content
      </div>
      
      <!-- Multiple classes with array -->
      <div [class]="['base-class', dynamicClass, conditionalClass]">
        Array Classes
      </div>
      
      <!-- Combining class attribute and binding -->
      <div class="static-class" [class.dynamic]="isDynamic">
        Mixed Classes
      </div>
    </div>
  `,
  styles: [`
    .active { color: green; }
    .highlighted { background-color: yellow; }
    .card { padding: 20px; border-radius: 8px; }
    .card-elevated { box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
    .card-primary { border: 2px solid #3498db; }
    .card-disabled { opacity: 0.5; pointer-events: none; }
  `]
})
export class ClassBindingComponent {
  isActive = true;
  isHighlighted = false;
  
  currentClasses = 'base-class theme-dark';
  
  isElevated = true;
  isPrimary = false;
  isDisabled = false;
  
  dynamicClass = 'custom-class';
  conditionalClass = this.isActive ? 'active-class' : 'inactive-class';
  isDynamic = true;
}
```

### Style Binding

Bind inline styles with single or multiple style properties:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-style-binding',
  standalone: true,
  template: `
    <div>
      <!-- Single style property -->
      <div [style.color]="textColor">Colored Text</div>
      <div [style.font-size.px]="fontSize">Dynamic Font Size</div>
      <div [style.width.%]="widthPercent">Dynamic Width</div>
      
      <!-- Multiple styles with object -->
      <div [style]="{
        'background-color': bgColor,
        'padding': '20px',
        'border-radius': '8px',
        'box-shadow': '0 2px 4px rgba(0,0,0,0.1)'
      }">
        Styled Container
      </div>
      
      <!-- Style with units -->
      <div 
        [style.width.px]="boxWidth"
        [style.height.px]="boxHeight"
        [style.transform]="'rotate(' + rotation + 'deg)'">
        Transformed Box
      </div>
      
      <!-- Conditional styles -->
      <div 
        [style.display]="isVisible ? 'block' : 'none'"
        [style.opacity]="isEnabled ? 1 : 0.5">
        Conditional Display
      </div>
      
      <!-- CSS custom properties -->
      <div 
        [style.--primary-color]="primaryColor"
        [style.--secondary-color]="secondaryColor">
        Custom Properties
      </div>
    </div>
  `
})
export class StyleBindingComponent {
  textColor = '#e74c3c';
  fontSize = 18;
  widthPercent = 75;
  
  bgColor = '#ecf0f1';
  
  boxWidth = 200;
  boxHeight = 200;
  rotation = 45;
  
  isVisible = true;
  isEnabled = true;
  
  primaryColor = '#3498db';
  secondaryColor = '#2ecc71';
}
```

## Event Binding

Event binding allows you to respond to user actions like clicks, keypresses, and mouse movements using parentheses `(event)="handler()"`.

### Basic Event Binding

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-event-binding',
  standalone: true,
  template: `
    <div>
      <!-- Click events -->
      <button (click)="handleClick()">Click Me</button>
      <button (click)="counter = counter + 1">Increment: {{ counter }}</button>
      
      <!-- Input events -->
      <input 
        (input)="handleInput($event)"
        (focus)="handleFocus()"
        (blur)="handleBlur()"
        [value]="inputValue">
      <p>Input value: {{ inputValue }}</p>
      
      <!-- Keyboard events -->
      <input 
        (keydown)="handleKeyDown($event)"
        (keyup)="handleKeyUp($event)"
        (keydown.enter)="handleEnter()"
        (keydown.escape)="handleEscape()"
        placeholder="Press keys...">
      
      <!-- Mouse events -->
      <div 
        (mouseenter)="handleMouseEnter()"
        (mouseleave)="handleMouseLeave()"
        (mousemove)="handleMouseMove($event)"
        style="width: 200px; height: 200px; border: 1px solid #ddd;">
        Mouse Area: {{ mousePosition }}
      </div>
      
      <!-- Form events -->
      <form (submit)="handleSubmit($event)">
        <input name="username" required>
        <button type="submit">Submit</button>
      </form>
      
      <!-- Custom component events -->
      <app-child (childEvent)="handleChildEvent($event)"></app-child>
    </div>
  `
})
export class EventBindingComponent {
  counter = 0;
  inputValue = '';
  mousePosition = 'x: 0, y: 0';

  handleClick(): void {
    console.log('Button clicked!');
    alert('Button was clicked');
  }

  handleInput(event: Event): void {
    const target = event.target as HTMLInputElement;
    this.inputValue = target.value;
    console.log('Input changed:', this.inputValue);
  }

  handleFocus(): void {
    console.log('Input focused');
  }

  handleBlur(): void {
    console.log('Input blurred');
  }

  handleKeyDown(event: KeyboardEvent): void {
    console.log('Key down:', event.key);
  }

  handleKeyUp(event: KeyboardEvent): void {
    console.log('Key up:', event.key);
  }

  handleEnter(): void {
    console.log('Enter key pressed');
  }

  handleEscape(): void {
    console.log('Escape key pressed');
  }

  handleMouseEnter(): void {
    console.log('Mouse entered');
  }

  handleMouseLeave(): void {
    console.log('Mouse left');
  }

  handleMouseMove(event: MouseEvent): void {
    this.mousePosition = `x: ${event.offsetX}, y: ${event.offsetY}`;
  }

  handleSubmit(event: Event): void {
    event.preventDefault();
    console.log('Form submitted');
  }

  handleChildEvent(data: any): void {
    console.log('Received from child:', data);
  }
}
```

### Event Object and $event

The special `$event` variable provides access to the DOM event or custom event data:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-event-object',
  standalone: true,
  template: `
    <div>
      <!-- Accessing event properties -->
      <button (click)="logEventDetails($event)">
        Click for Event Details
      </button>
      
      <!-- Input with event -->
      <input 
        (input)="onValueChange($event)"
        placeholder="Type something...">
      
      <!-- Checkbox with event -->
      <input 
        type="checkbox"
        (change)="onCheckboxChange($event)">
      
      <!-- Select with event -->
      <select (change)="onSelectChange($event)">
        <option value="option1">Option 1</option>
        <option value="option2">Option 2</option>
        <option value="option3">Option 3</option>
      </select>
      
      <!-- File input with event -->
      <input 
        type="file"
        (change)="onFileChange($event)">
      
      <!-- Drag and drop events -->
      <div 
        (dragover)="onDragOver($event)"
        (drop)="onDrop($event)"
        style="width: 200px; height: 100px; border: 2px dashed #ddd;">
        Drop Zone
      </div>
    </div>
  `
})
export class EventObjectComponent {
  logEventDetails(event: MouseEvent): void {
    console.log('Event type:', event.type);
    console.log('Target element:', event.target);
    console.log('Current target:', event.currentTarget);
    console.log('Mouse position:', event.clientX, event.clientY);
    console.log('Button pressed:', event.button);
    console.log('Modifier keys:', {
      shift: event.shiftKey,
      ctrl: event.ctrlKey,
      alt: event.altKey,
      meta: event.metaKey
    });
  }

  onValueChange(event: Event): void {
    const input = event.target as HTMLInputElement;
    console.log('New value:', input.value);
  }

  onCheckboxChange(event: Event): void {
    const checkbox = event.target as HTMLInputElement;
    console.log('Checkbox checked:', checkbox.checked);
  }

  onSelectChange(event: Event): void {
    const select = event.target as HTMLSelectElement;
    console.log('Selected value:', select.value);
  }

  onFileChange(event: Event): void {
    const input = event.target as HTMLInputElement;
    const files = input.files;
    if (files && files.length > 0) {
      console.log('Selected file:', files[0].name);
      console.log('File size:', files[0].size, 'bytes');
      console.log('File type:', files[0].type);
    }
  }

  onDragOver(event: DragEvent): void {
    event.preventDefault();
    console.log('Dragging over');
  }

  onDrop(event: DragEvent): void {
    event.preventDefault();
    const files = event.dataTransfer?.files;
    if (files && files.length > 0) {
      console.log('Dropped file:', files[0].name);
    }
  }
}
```

### Event Filtering and Key Modifiers

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-event-filtering',
  standalone: true,
  template: `
    <div>
      <!-- Key-specific events -->
      <input 
        (keydown.enter)="onEnter()"
        (keydown.escape)="onEscape()"
        (keydown.space)="onSpace()"
        (keydown.tab)="onTab()"
        placeholder="Press Enter, Esc, Space, or Tab">
      
      <!-- Key combinations with modifiers -->
      <input 
        (keydown.control.s)="onCtrlS($event)"
        (keydown.shift.enter)="onShiftEnter()"
        (keydown.alt.n)="onAltN()"
        placeholder="Try Ctrl+S, Shift+Enter, Alt+N">
      
      <!-- Arrow keys -->
      <div 
        (keydown.arrowUp)="onArrowUp()"
        (keydown.arrowDown)="onArrowDown()"
        (keydown.arrowLeft)="onArrowLeft()"
        (keydown.arrowRight)="onArrowRight()"
        tabindex="0"
        style="padding: 20px; border: 1px solid #ddd;">
        Use arrow keys: {{ direction }}
      </div>
      
      <!-- Mouse button filtering -->
      <div 
        (click)="onLeftClick()"
        (contextmenu)="onRightClick($event)"
        (auxclick)="onMiddleClick($event)"
        style="padding: 20px; border: 1px solid #ddd;">
        Try different mouse buttons
      </div>
    </div>
  `
})
export class EventFilteringComponent {
  direction = 'none';

  onEnter(): void {
    console.log('Enter pressed');
  }

  onEscape(): void {
    console.log('Escape pressed');
  }

  onSpace(): void {
    console.log('Space pressed');
  }

  onTab(): void {
    console.log('Tab pressed');
  }

  onCtrlS(event: KeyboardEvent): void {
    event.preventDefault(); // Prevent browser save dialog
    console.log('Ctrl+S - Save action');
  }

  onShiftEnter(): void {
    console.log('Shift+Enter - New line');
  }

  onAltN(): void {
    console.log('Alt+N - New action');
  }

  onArrowUp(): void {
    this.direction = 'up';
  }

  onArrowDown(): void {
    this.direction = 'down';
  }

  onArrowLeft(): void {
    this.direction = 'left';
  }

  onArrowRight(): void {
    this.direction = 'right';
  }

  onLeftClick(): void {
    console.log('Left click');
  }

  onRightClick(event: MouseEvent): void {
    event.preventDefault();
    console.log('Right click');
  }

  onMiddleClick(event: MouseEvent): void {
    console.log('Middle click');
  }
}
```

## Two-Way Binding

Two-way binding combines property binding and event binding to create a bidirectional data flow using `[(ngModel)]` syntax (banana in a box).

### NgModel Two-Way Binding

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-two-way-binding',
  standalone: true,
  imports: [FormsModule],
  template: `
    <div>
      <!-- Basic ngModel usage -->
      <input [(ngModel)]="username" placeholder="Username">
      <p>Username: {{ username }}</p>
      
      <!-- Multiple inputs bound to same property -->
      <input [(ngModel)]="email" type="email">
      <input [(ngModel)]="email" type="email">
      <p>Email: {{ email }}</p>
      
      <!-- Checkbox two-way binding -->
      <label>
        <input type="checkbox" [(ngModel)]="isSubscribed">
        Subscribe to newsletter
      </label>
      <p>Subscribed: {{ isSubscribed }}</p>
      
      <!-- Radio buttons -->
      <label>
        <input type="radio" [(ngModel)]="gender" value="male">
        Male
      </label>
      <label>
        <input type="radio" [(ngModel)]="gender" value="female">
        Female
      </label>
      <p>Gender: {{ gender }}</p>
      
      <!-- Select dropdown -->
      <select [(ngModel)]="selectedCountry">
        <option value="">Select a country</option>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="ca">Canada</option>
      </select>
      <p>Selected: {{ selectedCountry }}</p>
      
      <!-- Textarea -->
      <textarea 
        [(ngModel)]="description"
        rows="4"
        placeholder="Enter description...">
      </textarea>
      <p>Description length: {{ description.length }}</p>
    </div>
  `
})
export class TwoWayBindingComponent {
  username = '';
  email = '';
  isSubscribed = false;
  gender = '';
  selectedCountry = '';
  description = '';
}
```

### Expanded Two-Way Binding Syntax

Two-way binding is syntactic sugar for property and event binding:

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-expanded-binding',
  standalone: true,
  imports: [FormsModule],
  template: `
    <div>
      <!-- These two are equivalent -->
      
      <!-- Shorthand: two-way binding -->
      <input [(ngModel)]="name">
      
      <!-- Expanded: property + event binding -->
      <input 
        [ngModel]="name"
        (ngModelChange)="name = $event">
      
      <!-- With custom logic on change -->
      <input 
        [ngModel]="searchTerm"
        (ngModelChange)="onSearchChange($event)">
      <p>Search: {{ searchTerm }} ({{ searchResults.length }} results)</p>
      
      <!-- Debounced input -->
      <input 
        [ngModel]="debouncedValue"
        (ngModelChange)="onDebouncedChange($event)">
      <p>Debounced: {{ debouncedValue }}</p>
    </div>
  `
})
export class ExpandedBindingComponent {
  name = '';
  searchTerm = '';
  searchResults: string[] = [];
  debouncedValue = '';
  private debounceTimer: any;

  onSearchChange(value: string): void {
    this.searchTerm = value;
    // Simulate search
    this.searchResults = value 
      ? ['Result 1', 'Result 2', 'Result 3']
      : [];
  }

  onDebouncedChange(value: string): void {
    clearTimeout(this.debounceTimer);
    this.debounceTimer = setTimeout(() => {
      this.debouncedValue = value;
      console.log('Debounced value updated:', value);
    }, 500);
  }
}
```

### Custom Two-Way Binding

Create custom two-way binding for your components:

```typescript
import { Component, Input, Output, EventEmitter } from '@angular/core';

// Child component with custom two-way binding
@Component({
  selector: 'app-custom-input',
  standalone: true,
  template: `
    <div class="custom-input">
      <button (click)="decrement()">-</button>
      <span>{{ value }}</span>
      <button (click)="increment()">+</button>
    </div>
  `,
  styles: [`
    .custom-input {
      display: flex;
      gap: 10px;
      align-items: center;
    }
    button {
      padding: 5px 15px;
    }
  `]
})
export class CustomInputComponent {
  @Input() value: number = 0;
  @Output() valueChange = new EventEmitter<number>();

  increment(): void {
    this.value++;
    this.valueChange.emit(this.value);
  }

  decrement(): void {
    this.value--;
    this.valueChange.emit(this.value);
  }
}

// Parent component using custom two-way binding
@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [CustomInputComponent],
  template: `
    <div>
      <!-- Using custom two-way binding -->
      <app-custom-input [(value)]="count"></app-custom-input>
      <p>Count: {{ count }}</p>
      
      <!-- Expanded syntax -->
      <app-custom-input 
        [value]="anotherCount"
        (valueChange)="anotherCount = $event">
      </app-custom-input>
      <p>Another count: {{ anotherCount }}</p>
    </div>
  `
})
export class ParentComponent {
  count = 5;
  anotherCount = 10;
}
```

## Template Reference Variables

Template reference variables provide direct access to DOM elements or component instances using the `#` syntax.

### Basic Template Variables

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-template-variables',
  standalone: true,
  template: `
    <div>
      <!-- Reference to input element -->
      <input #nameInput placeholder="Enter name">
      <button (click)="logValue(nameInput.value)">
        Log Value
      </button>
      <button (click)="nameInput.focus()">
        Focus Input
      </button>
      
      <!-- Reference to form element -->
      <form #myForm>
        <input name="email" required>
        <button type="button" (click)="logForm(myForm)">
          Check Form
        </button>
      </form>
      
      <!-- Reference in same template -->
      <input #phone placeholder="Phone">
      <p>Phone value: {{ phone.value }}</p>
      <p>Input type: {{ phone.type }}</p>
      
      <!-- Reference to component -->
      <app-child #childComponent></app-child>
      <button (click)="childComponent.someMethod()">
        Call Child Method
      </button>
      
      <!-- Multiple references to same element -->
      <input #input1 #input2 placeholder="Multiple refs">
      <button (click)="input1.value = 'Updated'">Update via ref1</button>
      <button (click)="input2.value = 'Changed'">Update via ref2</button>
      
      <!-- Reference with ng-template -->
      <ng-template #templateRef>
        <p>Template content</p>
      </ng-template>
    </div>
  `
})
export class TemplateVariablesComponent {
  logValue(value: string): void {
    console.log('Input value:', value);
  }

  logForm(form: HTMLFormElement): void {
    console.log('Form element:', form);
    console.log('Form validity:', form.checkValidity());
  }
}
```

### Advanced Template Variable Usage

```typescript
import { Component, ViewChild, ElementRef, AfterViewInit } from '@angular/core';

@Component({
  selector: 'app-advanced-template-vars',
  standalone: true,
  template: `
    <div>
      <!-- Video player control -->
      <video #videoPlayer width="400" controls>
        <source src="sample.mp4" type="video/mp4">
      </video>
      <div>
        <button (click)="videoPlayer.play()">Play</button>
        <button (click)="videoPlayer.pause()">Pause</button>
        <button (click)="videoPlayer.currentTime = 0">Reset</button>
        <p>Duration: {{ videoPlayer.duration }}</p>
      </div>
      
      <!-- Canvas manipulation -->
      <canvas #myCanvas width="400" height="300"></canvas>
      <button (click)="drawOnCanvas(myCanvas)">Draw</button>
      <button (click)="clearCanvas(myCanvas)">Clear</button>
      
      <!-- Measuring element dimensions -->
      <div #measuredDiv style="padding: 20px; background: #f0f0f0;">
        Content to measure
      </div>
      <button (click)="measureElement(measuredDiv)">
        Measure Element
      </button>
      
      <!-- Form validation with references -->
      <form #registrationForm>
        <input 
          #usernameInput
          name="username"
          required
          minlength="3"
          placeholder="Username">
        <div *ngIf="usernameInput.validity.valid">
          ✓ Valid username
        </div>
        
        <input 
          #passwordInput
          type="password"
          name="password"
          required
          minlength="8">
        <button 
          type="submit"
          [disabled]="!registrationForm.checkValidity()">
          Register
        </button>
      </form>
    </div>
  `
})
export class AdvancedTemplateVarsComponent implements AfterViewInit {
  @ViewChild('videoPlayer') videoPlayer!: ElementRef<HTMLVideoElement>;
  @ViewChild('myCanvas') canvas!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit(): void {
    // Can access elements after view initialization
    console.log('Video element:', this.videoPlayer.nativeElement);
  }

  drawOnCanvas(canvas: HTMLCanvasElement): void {
    const ctx = canvas.getContext('2d');
    if (ctx) {
      ctx.fillStyle = '#3498db';
      ctx.fillRect(50, 50, 200, 100);
      ctx.fillStyle = '#fff';
      ctx.font = '20px Arial';
      ctx.fillText('Hello Canvas!', 90, 105);
    }
  }

  clearCanvas(canvas: HTMLCanvasElement): void {
    const ctx = canvas.getContext('2d');
    if (ctx) {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    }
  }

  measureElement(element: HTMLElement): void {
    const rect = element.getBoundingClientRect();
    console.log('Element dimensions:', {
      width: rect.width,
      height: rect.height,
      top: rect.top,
      left: rect.left
    });
  }
}
```

## Safe Navigation and Nullish Coalescing

Angular provides operators to safely handle null and undefined values in templates.

### Safe Navigation Operator (?.)

```typescript
import { Component } from '@angular/core';

interface User {
  name: string;
  email?: string;
  address?: {
    street?: string;
    city?: string;
    country?: string;
  };
  preferences?: {
    theme?: string;
    notifications?: {
      email?: boolean;
      push?: boolean;
    };
  };
}

@Component({
  selector: 'app-safe-navigation',
  standalone: true,
  template: `
    <div>
      <!-- Without safe navigation (would throw error if user is null) -->
      <!-- <p>{{ user.name }}</p> --> <!-- ❌ Error if user is null -->
      
      <!-- With safe navigation -->
      <p>Name: {{ user?.name }}</p>
      <p>Email: {{ user?.email }}</p>
      
      <!-- Nested safe navigation -->
      <p>Street: {{ user?.address?.street }}</p>
      <p>City: {{ user?.address?.city }}</p>
      <p>Country: {{ user?.address?.country }}</p>
      
      <!-- Deep nested access -->
      <p>Email notifications: {{ user?.preferences?.notifications?.email }}</p>
      <p>Push notifications: {{ user?.preferences?.notifications?.push }}</p>
      
      <!-- Safe navigation with method calls -->
      <p>Upper name: {{ user?.name?.toUpperCase() }}</p>
      <p>Email domain: {{ user?.email?.split('@')[1] }}</p>
      
      <!-- Safe navigation with arrays -->
      <p>First hobby: {{ hobbies?.[0] }}</p>
      <p>Second hobby: {{ hobbies?.[1] }}</p>
      
      <!-- Combined with other operators -->
      <p>Theme: {{ user?.preferences?.theme ?? 'default' }}</p>
      <p>Status: {{ user?.email ? 'Active' : 'Inactive' }}</p>
    </div>
  `
})
export class SafeNavigationComponent {
  user: User | null = {
    name: 'John Doe',
    email: 'john@example.com',
    address: {
      city: 'New York'
      // street and country are undefined
    },
    preferences: {
      notifications: {
        email: true
      }
    }
  };

  hobbies: string[] | null = ['Reading', 'Coding', 'Gaming'];

  // Simulate loading state
  loadingUser: User | null = null;
}
```

### Nullish Coalescing Operator (??)

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-nullish-coalescing',
  standalone: true,
  template: `
    <div>
      <!-- Nullish coalescing: use default if null/undefined -->
      <p>Username: {{ username ?? 'Anonymous' }}</p>
      <p>Age: {{ age ?? 'Not specified' }}</p>
      <p>Score: {{ score ?? 0 }}</p>
      
      <!-- Difference from || operator -->
      <p>Count with ??: {{ count ?? 10 }}</p> <!-- Shows 0 -->
      <p>Count with ||: {{ count || 10 }}</p> <!-- Shows 10 -->
      
      <p>Empty string with ??: {{ emptyString ?? 'default' }}</p> <!-- Shows '' -->
      <p>Empty string with ||: {{ emptyString || 'default' }}</p> <!-- Shows 'default' -->
      
      <!-- Combined with safe navigation -->
      <p>City: {{ user?.address?.city ?? 'Unknown city' }}</p>
      <p>Theme: {{ settings?.theme ?? 'light' }}</p>
      
      <!-- Chaining nullish coalescing -->
      <p>Display: {{ nickname ?? username ?? email ?? 'User' }}</p>
      
      <!-- With expressions -->
      <p>Status: {{ (isActive ?? false) ? 'Active' : 'Inactive' }}</p>
      <p>Total: {{ (price ?? 0) * (quantity ?? 1) }}</p>
    </div>
  `
})
export class NullishCoalescingComponent {
  username: string | null = null;
  age: number | undefined = undefined;
  score: number | null = null;
  
  count = 0; // Falsy but not nullish
  emptyString = ''; // Falsy but not nullish
  
  user = {
    address: {
      city: undefined
    }
  };
  
  settings: { theme?: string } | null = null;
  
  nickname: string | null = null;
  email = 'user@example.com';
  
  isActive: boolean | null = null;
  price: number | undefined = undefined;
  quantity: number | null = null;
}
```

### Non-Null Assertion Operator (!)

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-non-null-assertion',
  standalone: true,
  template: `
    <div>
      <!-- Use when you know value is not null -->
      <!-- Be careful: this disables type checking -->
      
      <!-- Safe: value is initialized -->
      <p>Initialized: {{ initializedValue!.toUpperCase() }}</p>
      
      <!-- Dangerous: might be null at runtime -->
      <!-- <p>{{ nullableValue!.toUpperCase() }}</p> --> <!-- ❌ Runtime error -->
      
      <!-- Better alternatives -->
      <p>Safe: {{ nullableValue?.toUpperCase() }}</p>
      <p>With default: {{ nullableValue?.toUpperCase() ?? 'N/A' }}</p>
      
      <!-- Non-null with array access -->
      <p>First item: {{ items![0] }}</p>
      
      <!-- Use in expressions -->
      <p>Length: {{ text!.length }}</p>
    </div>
  `
})
export class NonNullAssertionComponent {
  initializedValue: string = 'hello';
  nullableValue: string | null = null;
  items: string[] = ['Item 1', 'Item 2'];
  text: string = 'Sample text';
}
```

## Common Mistakes

### Mistake 1: Using Assignment in Template Expressions

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- This will cause an error -->
    <button (click)="count = count + 1">{{ count }}</button>
    <p>{{ name = 'John' }}</p>
  `
})
export class WrongComponent {
  count = 0;
  name = '';
}
```

✅ **Correct:**
```typescript
@Component({
  template: `
    <button (click)="increment()">{{ count }}</button>
    <p>{{ name }}</p>
  `
})
export class CorrectComponent {
  count = 0;
  name = '';

  increment(): void {
    this.count++;
  }

  ngOnInit(): void {
    this.name = 'John';
  }
}
```

### Mistake 2: Confusing Property and Attribute Binding

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- colspan is an attribute, not a property -->
    <td [colspan]="3">Cell</td>
    
    <!-- value is a property, not an attribute -->
    <input attr.value="text">
  `
})
```

✅ **Correct:**
```typescript
@Component({
  template: `
    <!-- Use attribute binding for attributes -->
    <td [attr.colspan]="3">Cell</td>
    
    <!-- Use property binding for properties -->
    <input [value]="'text'">
  `
})
```

### Mistake 3: Forgetting FormsModule for NgModel

❌ **Wrong:**
```typescript
@Component({
  standalone: true,
  // Missing FormsModule import
  template: `
    <input [(ngModel)]="value">
  `
})
export class WrongComponent {
  value = '';
}
```

✅ **Correct:**
```typescript
import { FormsModule } from '@angular/forms';

@Component({
  standalone: true,
  imports: [FormsModule], // Required for ngModel
  template: `
    <input [(ngModel)]="value">
  `
})
export class CorrectComponent {
  value = '';
}
```

### Mistake 4: Not Preventing Default on Form Submit

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- Form will cause page reload -->
    <form (submit)="handleSubmit()">
      <button type="submit">Submit</button>
    </form>
  `
})
export class WrongComponent {
  handleSubmit(): void {
    console.log('Submitted');
    // Page reloads before this completes
  }
}
```

✅ **Correct:**
```typescript
@Component({
  template: `
    <form (submit)="handleSubmit($event)">
      <button type="submit">Submit</button>
    </form>
  `
})
export class CorrectComponent {
  handleSubmit(event: Event): void {
    event.preventDefault();
    console.log('Submitted');
  }
}
```

### Mistake 5: Misusing Safe Navigation with Methods

❌ **Wrong:**
```typescript
@Component({
  template: `
    <!-- Safe navigation doesn't work with method calls this way -->
    <p>{{ user?.getName?.() }}</p>
  `
})
```

✅ **Correct:**
```typescript
@Component({
  template: `
    <!-- Call method in component -->
    <p>{{ getUserName() }}</p>
  `
})
export class CorrectComponent {
  user: User | null = null;

  getUserName(): string {
    return this.user?.getName() ?? 'Unknown';
  }
}
```

## Best Practices

### 1. Keep Template Logic Simple

✅ **Good:**
```typescript
@Component({
  template: `
    <p>{{ fullName }}</p>
    <p>{{ isEligible ? 'Eligible' : 'Not eligible' }}</p>
  `
})
export class GoodComponent {
  firstName = 'John';
  lastName = 'Doe';
  age = 25;

  get fullName(): string {
    return `${this.firstName} ${this.lastName}`;
  }

  get isEligible(): boolean {
    return this.age >= 18;
  }
}
```

### 2. Use Appropriate Binding Types

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Property binding for DOM properties -->
    <input [value]="text" [disabled]="isDisabled">
    
    <!-- Attribute binding for HTML attributes -->
    <td [attr.colspan]="columns">
    
    <!-- Class binding for CSS classes -->
    <div [class.active]="isActive">
    
    <!-- Style binding for inline styles -->
    <div [style.color]="textColor">
    
    <!-- Event binding for events -->
    <button (click)="handleClick()">
  `
})
```

### 3. Prefer Safe Navigation Over Null Checks

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Concise and safe -->
    <p>{{ user?.profile?.displayName ?? 'Anonymous' }}</p>
  `
})
```

❌ **Avoid:**
```typescript
@Component({
  template: `
    <!-- Verbose and error-prone -->
    <p>{{ user && user.profile && user.profile.displayName ? user.profile.displayName : 'Anonymous' }}</p>
  `
})
```

### 4. Use Template Reference Variables for DOM Access

✅ **Good:**
```typescript
@Component({
  template: `
    <input #searchInput>
    <button (click)="search(searchInput.value)">Search</button>
  `
})
export class GoodComponent {
  search(term: string): void {
    console.log('Searching for:', term);
  }
}
```

### 5. Leverage Key Event Filtering

✅ **Good:**
```typescript
@Component({
  template: `
    <input 
      (keydown.enter)="submit()"
      (keydown.escape)="cancel()"
      (keydown.control.s)="save($event)">
  `
})
```

### 6. Use Nullish Coalescing for Default Values

✅ **Good:**
```typescript
@Component({
  template: `
    <!-- Preserves falsy values except null/undefined -->
    <p>Count: {{ count ?? 0 }}</p>
    <p>Name: {{ name ?? 'Guest' }}</p>
  `
})
export class GoodComponent {
  count: number | null = null;
  name: string | undefined;
}
```

### 7. Combine Bindings Effectively

✅ **Good:**
```typescript
@Component({
  template: `
    <div 
      class="card"
      [class.card-highlighted]="isHighlighted"
      [style.background-color]="bgColor"
      (click)="handleClick()">
      Content
    </div>
  `
})
```

### 8. Use Two-Way Binding Sparingly

✅ **Good:**
```typescript
// Use for simple form inputs
@Component({
  template: `
    <input [(ngModel)]="searchTerm">
  `
})
```

✅ **Better for complex scenarios:**
```typescript
// Use expanded syntax when you need custom logic
@Component({
  template: `
    <input 
      [ngModel]="searchTerm"
      (ngModelChange)="onSearchChange($event)">
  `
})
export class BetterComponent {
  searchTerm = '';

  onSearchChange(value: string): void {
    this.searchTerm = value;
    // Custom logic here
    this.performSearch(value);
  }

  performSearch(term: string): void {
    // Search implementation
  }
}
```

## Interview Questions

**Q1: What is the difference between property binding and attribute binding in Angular?**

**A:** Property binding (`[property]`) sets DOM element properties, while attribute binding (`[attr.attribute]`) sets HTML attributes. Key differences:

1. **Property binding** targets DOM object properties:
```typescript
<input [value]="text">        // Sets input.value property
<button [disabled]="isDisabled"> // Sets button.disabled property
```

2. **Attribute binding** targets HTML attributes:
```typescript
<td [attr.colspan]="3">       // Sets colspan attribute
<button [attr.aria-label]="label"> // Sets aria-label attribute
```

Most HTML attributes have corresponding DOM properties, so property binding is preferred. Use attribute binding when:
- No corresponding DOM property exists (ARIA attributes, data attributes)
- Working with SVG attributes
- Setting colspan, rowspan, etc.

**Q2: How does two-way binding work in Angular? What is it syntactic sugar for?**

**A:** Two-way binding (`[(ngModel)]`) is syntactic sugar that combines property binding and event binding:

```typescript
// Two-way binding
<input [(ngModel)]="value">

// Expands to:
<input 
  [ngModel]="value"
  (ngModelChange)="value = $event">
```

It works through:
1. **Property binding** (`[ngModel]`) - passes component value to the view
2. **Event binding** (`(ngModelChange)`) - updates component when view changes
3. **Naming convention** - the event must be named `propertyChange` where `property` is the input name

For custom components:
```typescript
@Component({
  selector: 'app-custom',
  template: `<input [value]="value" (input)="onChange($event)">`
})
export class CustomComponent {
  @Input() value = '';
  @Output() valueChange = new EventEmitter<string>();

  onChange(event: Event): void {
    const newValue = (event.target as HTMLInputElement).value;
    this.valueChange.emit(newValue);
  }
}

// Usage:
<app-custom [(value)]="myValue"></app-custom>
```

**Q3: What are template reference variables and when should you use them?**

**A:** Template reference variables (created with `#varName`) provide direct access to DOM elements or component instances within the template:

```typescript
@Component({
  template: `
    <!-- Reference to DOM element -->
    <input #nameInput placeholder="Name">
    <button (click)="nameInput.focus()">Focus</button>
    <p>{{ nameInput.value }}</p>

    <!-- Reference to component -->
    <app-child #childComp></app-child>
    <button (click)="childComp.childMethod()">Call Child</button>

    <!-- Reference to directive -->
    <form #myForm="ngForm">
      <input ngModel name="username">
    </form>
  `
})
```

**Use cases:**
1. **Direct DOM manipulation** - focus, scroll, measure dimensions
2. **Access element values** - without binding to component properties
3. **Call component methods** - from parent template
4. **Form validation** - access form/control state
5. **Passing elements** - to methods as parameters

**Avoid:**
- Don't overuse for state management (use component properties instead)
- Don't use for complex DOM manipulation (use Renderer2 or directives)
- Don't access in class before `AfterViewInit` (they're undefined until then)

**Q4: Explain the safe navigation operator (?.) and nullish coalescing operator (??). When should you use each?**

**A:** Both operators handle null/undefined values safely, but serve different purposes:

**Safe Navigation Operator (?.):**
- Prevents errors when accessing properties of null/undefined objects
- Short-circuits evaluation and returns undefined if the left side is null/undefined
```typescript
// Without safe navigation - throws error if user is null
<p>{{ user.name }}</p>

// With safe navigation - returns undefined safely
<p>{{ user?.name }}</p>

// Chaining
<p>{{ user?.address?.city }}</p>
```

**Nullish Coalescing Operator (??):**
- Provides default values for null/undefined (but not other falsy values)
- Returns right side only if left side is null or undefined
```typescript
// Returns 'Guest' only if username is null/undefined
<p>{{ username ?? 'Guest' }}</p>

// Key difference from || operator
count = 0;
<p>{{ count ?? 10 }}</p>  // Shows 0
<p>{{ count || 10 }}</p>  // Shows 10

emptyString = '';
<p>{{ emptyString ?? 'default' }}</p>  // Shows ''
<p>{{ emptyString || 'default' }}</p>  // Shows 'default'
```

**Combined usage:**
```typescript
// Safe navigation with default value
<p>{{ user?.profile?.displayName ?? 'Anonymous' }}</p>

// Safe array access with default
<p>{{ items?.[0]?.name ?? 'No items' }}</p>
```

**Q5: What are the limitations of template expressions in Angular?**

**A:** Template expressions are intentionally restricted for security and predictability:

**Not Allowed:**
1. **Assignments** - can't use `=`, `+=`, `-=`, etc.
```typescript
// ❌ Wrong
<p>{{ count = count + 1 }}</p>
```

2. **new, typeof, instanceof operators**
```typescript
// ❌ Wrong
<p>{{ new Date() }}</p>
<p>{{ typeof value }}</p>
```

3. **Chaining with semicolons or commas**
```typescript
// ❌ Wrong
<p>{{ firstName = 'John'; lastName = 'Doe' }}</p>
```

4. **Increment/Decrement operators**
```typescript
// ❌ Wrong
<p>{{ counter++ }}</p>
<p>{{ --index }}</p>
```

5. **Bitwise operators** - `|`, `&`, `^`

6. **Global namespace access** - window, document, console, etc.
```typescript
// ❌ Wrong
<p>{{ window.location.href }}</p>
```

**Allowed:**
1. **Property access** - `user.name`, `items[0]`
2. **Method calls** - `getFullName()`
3. **Simple operators** - `+`, `-`, `*`, `/`, `%`, `&&`, `||`, `!`
4. **Ternary operator** - `condition ? true : false`
5. **Optional chaining** - `user?.name`
6. **Nullish coalescing** - `value ?? default`
7. **Pipe operator** - `value | pipeName`

**Why these restrictions?**
- **Security** - prevents code injection
- **Predictability** - no side effects in templates
- **Performance** - expressions should be fast
- **Maintainability** - complex logic belongs in component class

**Q6: How do you handle form submission properly in Angular?**

**A:** Proper form submission handling prevents page reloads and provides better UX:

**Method 1: Prevent default on submit event**
```typescript
@Component({
  template: `
    <form (submit)="onSubmit($event)">
      <input [(ngModel)]="username" name="username">
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent {
  username = '';

  onSubmit(event: Event): void {
    event.preventDefault(); // Crucial!
    console.log('Submitted:', this.username);
    // Handle form submission
  }
}
```

**Method 2: Use ngSubmit event (recommended)**
```typescript
@Component({
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="username" name="username">
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent {
  username = '';

  onSubmit(): void {
    // No need for preventDefault with ngSubmit
    console.log('Submitted:', this.username);
  }
}
```

**Method 3: Template-driven forms with validation**
```typescript
@Component({
  template: `
    <form #form="ngForm" (ngSubmit)="onSubmit(form)">
      <input 
        [(ngModel)]="username" 
        name="username"
        required
        minlength="3">
      <button 
        type="submit"
        [disabled]="!form.valid">
        Submit
      </button>
    </form>
  `
})
export class FormComponent {
  username = '';

  onSubmit(form: NgForm): void {
    if (form.valid) {
      console.log('Valid submission:', this.username);
    }
  }
}
```

**Method 4: Reactive forms**
```typescript
@Component({
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="username">
      <button 
        type="submit"
        [disabled]="!form.valid">
        Submit
      </button>
    </form>
  `
})
export class FormComponent {
  form = new FormGroup({
    username: new FormControl('', [
      Validators.required,
      Validators.minLength(3)
    ])
  });

  onSubmit(): void {
    if (this.form.valid) {
      console.log('Submitted:', this.form.value);
    }
  }
}
```

**Best practices:**
1. Always use `ngSubmit` instead of `submit` for Angular forms
2. Disable submit button when form is invalid
3. Show validation errors
4. Handle loading states during submission
5. Reset form after successful submission if needed

**Q7: What's the difference between (keydown), (keypress), and (keyup) events?**

**A:** These keyboard events fire at different stages of a key press:

**1. keydown**
- Fires when key is first pressed down
- Fires continuously if key is held
- Fires before the input value changes
- Best for: keyboard shortcuts, preventing default behavior

```typescript
@Component({
  template: `
    <input 
      (keydown)="onKeyDown($event)"
      (keydown.enter)="onEnter()"
      (keydown.control.s)="onSave($event)">
  `
})
export class KeyEventsComponent {
  onKeyDown(event: KeyboardEvent): void {
    console.log('Key down:', event.key);
    // Input value not yet updated
  }

  onEnter(): void {
    console.log('Enter pressed');
  }

  onSave(event: KeyboardEvent): void {
    event.preventDefault(); // Prevent browser save
    console.log('Save triggered');
  }
}
```

**2. keypress (deprecated)**
- Fires between keydown and keyup
- Only fires for character keys (not Shift, Ctrl, etc.)
- **Deprecated** - use keydown instead
- Not fired for special keys

```typescript
// ⚠️ Deprecated - don't use
(keypress)="onKeyPress($event)"
```

**3. keyup**
- Fires when key is released
- Fires after input value has changed
- Best for: reading input values, text processing

```typescript
@Component({
  template: `
    <input 
      (keyup)="onKeyUp($event)"
      (keyup.escape)="onEscape()">
  `
})
export class KeyEventsComponent {
  onKeyUp(event: KeyboardEvent): void {
    const input = event.target as HTMLInputElement;
    console.log('Current value:', input.value);
    // Input value is updated
  }

  onEscape(): void {
    console.log('Escape pressed');
  }
}
```

**Event order:**
```
User presses 'A' key:
1. keydown (fires)
2. keypress (deprecated, fires)
3. input value changes
4. keyup (fires)
```

**Use cases:**
- **keydown**: Keyboard shortcuts, preventing input, game controls
- **keyup**: Search as you type, character counting, validation
- **keypress**: Don't use (deprecated)

**Event filtering:**
```typescript
@Component({
  template: `
    <!-- Specific keys -->
    <input (keydown.enter)="submit()">
    <input (keydown.escape)="cancel()">
    
    <!-- With modifiers -->
    <input (keydown.control.s)="save($event)">
    <input (keydown.shift.enter)="newLine()">
    
    <!-- Arrow keys -->
    <div 
      (keydown.arrowUp)="moveUp()"
      (keydown.arrowDown)="moveDown()"
      tabindex="0">
    </div>
  `
})
```

**Q8: How do you implement debouncing for user input in templates?**

**A:** There are multiple approaches to debounce input in Angular templates:

**Method 1: Expanded two-way binding with timer**
```typescript
import { Component } from '@angular/core';

@Component({
  template: `
    <input 
      [ngModel]="searchTerm"
      (ngModelChange)="onSearchChange($event)"
      placeholder="Search...">
    <p>Searching for: {{ debouncedSearchTerm }}</p>
  `
})
export class DebounceComponent {
  searchTerm = '';
  debouncedSearchTerm = '';
  private debounceTimer: any;

  onSearchChange(value: string): void {
    this.searchTerm = value;
    
    clearTimeout(this.debounceTimer);
    this.debounceTimer = setTimeout(() => {
      this.debouncedSearchTerm = value;
      this.performSearch(value);
    }, 500);
  }

  performSearch(term: string): void {
    console.log('Searching for:', term);
    // API call here
  }
}
```

**Method 2: RxJS debounceTime (recommended)**
```typescript
import { Component, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Component({
  template: `
    <input 
      (input)="searchSubject.next($any($event.target).value)"
      placeholder="Search...">
  `
})
export class DebounceComponent implements OnDestroy {
  searchSubject = new Subject<string>();

  constructor() {
    this.searchSubject.pipe(
      debounceTime(500),
      distinctUntilChanged()
    ).subscribe(searchTerm => {
      this.performSearch(searchTerm);
    });
  }

  performSearch(term: string): void {
    console.log('Searching for:', term);
  }

  ngOnDestroy(): void {
    this.searchSubject.complete();
  }
}
```

**Method 3: Custom debounce directive**
```typescript
import { Directive, EventEmitter, HostListener, Input, Output } from '@angular/core';
import { Subject } from 'rxjs';
import { debounceTime } from 'rxjs/operators';

@Directive({
  selector: '[appDebounce]',
  standalone: true
})
export class DebounceDirective {
  @Input() debounceTime = 500;
  @Output() debounced = new EventEmitter<Event>();
  private subject = new Subject<Event>();

  constructor() {
    this.subject
      .pipe(debounceTime(this.debounceTime))
      .subscribe(event => this.debounced.emit(event));
  }

  @HostListener('input', ['$event'])
  onInput(event: Event): void {
    this.subject.next(event);
  }
}

// Usage:
@Component({
  imports: [DebounceDirective],
  template: `
    <input 
      appDebounce
      [debounceTime]="300"
      (debounced)="onSearch($event)">
  `
})
```

**Method 4: FormControl with RxJS**
```typescript
import { Component, OnInit } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Component({
  imports: [ReactiveFormsModule],
  template: `
    <input [formControl]="searchControl" placeholder="Search...">
  `
})
export class DebounceComponent implements OnInit {
  searchControl = new FormControl('');

  ngOnInit(): void {
    this.searchControl.valueChanges
      .pipe(
        debounceTime(500),
        distinctUntilChanged()
      )
      .subscribe(value => {
        this.performSearch(value ?? '');
      });
  }

  performSearch(term: string): void {
    console.log('Searching for:', term);
  }
}
```

**Best approach:** Method 4 (FormControl with RxJS) is recommended because:
- Clean separation of concerns
- Built-in RxJS operators
- Easy to add more operators (distinctUntilChanged, switchMap, etc.)
- No manual cleanup needed with async pipe
- Type-safe with typed forms

## Key Takeaways

- **Interpolation** (`{{ }}`) is the simplest way to display component data in templates
- **Property binding** (`[property]`) sets DOM properties, while **attribute binding** (`[attr.name]`) sets HTML attributes
- **Event binding** (`(event)`) responds to user actions and passes the `$event` object
- **Two-way binding** (`[(ngModel)]`) combines property and event binding for bidirectional data flow
- **Template reference variables** (`#varName`) provide direct access to DOM elements and components
- **Safe navigation** (`?.`) prevents errors when accessing properties of null/undefined objects
- **Nullish coalescing** (`??`) provides default values only for null/undefined (not other falsy values)
- Template expressions are intentionally limited to prevent side effects and security issues
- Use **ngSubmit** instead of submit for proper form handling without page reloads
- Keep template logic simple; move complex operations to component methods or getters
- Prefer **keydown** over deprecated keypress for keyboard event handling
- Use **RxJS debounceTime** for efficient input debouncing instead of manual timers

## Resources

- [Angular Template Syntax - Official Docs](https://angular.io/guide/template-syntax)
- [Angular Binding Syntax - Official Docs](https://angular.io/guide/binding-syntax)
- [Property Binding - Official Guide](https://angular.io/guide/property-binding)
- [Event Binding - Official Guide](https://angular.io/guide/event-binding)
- [Two-Way Binding - Official Guide](https://angular.io/guide/two-way-binding)
- [Template Reference Variables - Official Docs](https://angular.io/guide/template-reference-variables)
- [Template Expression Operators - Official Guide](https://angular.io/guide/template-expression-operators)
- [Angular FormsModule API](https://angular.io/api/forms/FormsModule)
- [Understanding Angular Template Syntax - Angular.io Blog](https://blog.angular.io/tagged/templates)
- [Template Syntax Best Practices - Angular Style Guide](https://angular.io/guide/styleguide)
