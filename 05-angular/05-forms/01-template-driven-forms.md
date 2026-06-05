# Template-Driven Forms in Angular

## The Idea

**In plain English:** A template-driven form is a way to build an input form (like a sign-up page) in Angular where you describe all the rules — what fields are required, what format they need — directly in the HTML, and Angular automatically keeps track of what the user typed and whether it is valid.

**Real-world analogy:** Think of a paper job application form at a store. The form itself has printed instructions like "required" next to certain boxes, and a manager checks it before accepting it.
- The printed form (HTML template) = the Angular template where you write `ngModel` and validation rules
- Each blank box on the form = an input field bound with `[(ngModel)]`
- The "required" label printed next to a box = a built-in HTML validator like `required` or `email`
- The manager reviewing the completed form = Angular's `NgForm` directive that tracks overall validity before allowing submission

---

## Table of Contents
- [Introduction](#introduction)
- [Setting Up Template-Driven Forms](#setting-up-template-driven-forms)
- [ngModel Directive](#ngmodel-directive)
- [Two-Way Data Binding](#two-way-data-binding)
- [Form Validation](#form-validation)
- [Displaying Validation Errors](#displaying-validation-errors)
- [Form Submission](#form-submission)
- [ngForm and ngModelGroup](#ngform-and-ngmodelgroup)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Template-driven forms in Angular are forms where the form logic lives primarily in the template (HTML) rather than in the component class. They use directives like `ngModel`, `ngForm`, and validators to create and manage forms. This approach is similar to AngularJS forms and is best suited for simple forms with straightforward validation requirements.

Template-driven forms are ideal for:
- Simple forms with basic validation
- Rapid prototyping
- Forms that closely mirror the data model
- Scenarios where you want minimal component code

## Setting Up Template-Driven Forms

### Import FormsModule

```typescript
// app.config.ts (Standalone)
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { FormsModule } from '@angular/forms';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    // FormsModule is imported at component level for standalone
  ]
};

// For standalone components
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-contact-form',
  standalone: true,
  imports: [FormsModule, CommonModule],
  templateUrl: './contact-form.component.html'
})
export class ContactFormComponent {}
```

### Module-Based Setup

```typescript
// app.module.ts (Module-based)
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    FormsModule  // Import FormsModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

## ngModel Directive

The `ngModel` directive creates a `FormControl` instance and binds it to a form control element.

### Basic ngModel

```typescript
@Component({
  selector: 'app-simple-form',
  standalone: true,
  imports: [FormsModule],
  template: `
    <form>
      <label for="username">Username:</label>
      <input 
        type="text" 
        id="username"
        name="username"
        [(ngModel)]="username"
      >
      <p>You entered: {{ username }}</p>
    </form>
  `
})
export class SimpleFormComponent {
  username = '';
}
```

### ngModel with Different Input Types

```typescript
@Component({
  selector: 'app-input-types',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form>
      <!-- Text input -->
      <input type="text" [(ngModel)]="user.name" name="name">
      
      <!-- Email input -->
      <input type="email" [(ngModel)]="user.email" name="email">
      
      <!-- Number input -->
      <input type="number" [(ngModel)]="user.age" name="age">
      
      <!-- Checkbox -->
      <input type="checkbox" [(ngModel)]="user.subscribe" name="subscribe">
      
      <!-- Radio buttons -->
      <label>
        <input type="radio" [(ngModel)]="user.gender" name="gender" value="male">
        Male
      </label>
      <label>
        <input type="radio" [(ngModel)]="user.gender" name="gender" value="female">
        Female
      </label>
      
      <!-- Select dropdown -->
      <select [(ngModel)]="user.country" name="country">
        <option value="">Select Country</option>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="ca">Canada</option>
      </select>
      
      <!-- Textarea -->
      <textarea [(ngModel)]="user.bio" name="bio"></textarea>
      
      <!-- Multiple select -->
      <select [(ngModel)]="user.interests" name="interests" multiple>
        <option value="sports">Sports</option>
        <option value="music">Music</option>
        <option value="art">Art</option>
      </select>
    </form>

    <pre>{{ user | json }}</pre>
  `
})
export class InputTypesComponent {
  user = {
    name: '',
    email: '',
    age: null,
    subscribe: false,
    gender: '',
    country: '',
    bio: '',
    interests: []
  };
}
```

## Two-Way Data Binding

The `[(ngModel)]` syntax combines property binding `[ngModel]` and event binding `(ngModelChange)`.

### Understanding Two-Way Binding

```typescript
// [(ngModel)]="username" is shorthand for:
<input 
  [ngModel]="username"
  (ngModelChange)="username = $event"
>

// Expanded form
@Component({
  selector: 'app-binding-demo',
  standalone: true,
  imports: [FormsModule],
  template: `
    <!-- Two-way binding (shorthand) -->
    <input [(ngModel)]="username" name="username">
    
    <!-- Same as above (expanded) -->
    <input 
      [ngModel]="username"
      (ngModelChange)="username = $event"
      name="username"
    >
    
    <!-- With custom logic on change -->
    <input 
      [ngModel]="username"
      (ngModelChange)="onUsernameChange($event)"
      name="username"
    >
  `
})
export class BindingDemoComponent {
  username = '';

  onUsernameChange(value: string) {
    this.username = value.toLowerCase(); // Transform to lowercase
  }
}
```

### One-Way Data Flow

```typescript
@Component({
  selector: 'app-one-way',
  standalone: true,
  imports: [FormsModule],
  template: `
    <!-- One-way binding (property) -->
    <input [ngModel]="username" name="username">
    <!-- Value changes in component update input, but not vice versa -->
    
    <!-- One-way binding (event) -->
    <input (ngModelChange)="username = $event" name="username">
    <!-- Input changes update component, but component changes don't update input -->
  `
})
export class OneWayComponent {
  username = 'initial value';
}
```

## Form Validation

Template-driven forms support built-in HTML5 validators and custom validators through directives.

### Built-in Validators

```typescript
@Component({
  selector: 'app-validation-form',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form #userForm="ngForm">
      <!-- Required -->
      <input 
        type="text" 
        [(ngModel)]="user.username" 
        name="username"
        required
        #username="ngModel"
      >
      <div *ngIf="username.invalid && username.touched">
        Username is required
      </div>

      <!-- Required + minlength -->
      <input 
        type="password" 
        [(ngModel)]="user.password" 
        name="password"
        required
        minlength="8"
        #password="ngModel"
      >
      <div *ngIf="password.invalid && password.touched">
        <div *ngIf="password.errors?.['required']">Password is required</div>
        <div *ngIf="password.errors?.['minlength']">
          Password must be at least 8 characters
        </div>
      </div>

      <!-- Email validation -->
      <input 
        type="email" 
        [(ngModel)]="user.email" 
        name="email"
        required
        email
        #email="ngModel"
      >
      <div *ngIf="email.invalid && email.touched">
        <div *ngIf="email.errors?.['required']">Email is required</div>
        <div *ngIf="email.errors?.['email']">Invalid email format</div>
      </div>

      <!-- Pattern validation -->
      <input 
        type="text" 
        [(ngModel)]="user.phone" 
        name="phone"
        pattern="[0-9]{3}-[0-9]{3}-[0-9]{4}"
        #phone="ngModel"
      >
      <div *ngIf="phone.invalid && phone.touched">
        Phone must be in format: 123-456-7890
      </div>

      <!-- Min and max -->
      <input 
        type="number" 
        [(ngModel)]="user.age" 
        name="age"
        min="18"
        max="100"
        #age="ngModel"
      >
      <div *ngIf="age.invalid && age.touched">
        Age must be between 18 and 100
      </div>

      <button type="submit" [disabled]="userForm.invalid">
        Submit
      </button>
    </form>
  `
})
export class ValidationFormComponent {
  user = {
    username: '',
    password: '',
    email: '',
    phone: '',
    age: null
  };
}
```

### Validation States

```typescript
@Component({
  selector: 'app-validation-states',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form>
      <input 
        type="email" 
        [(ngModel)]="email" 
        name="email"
        required
        email
        #emailField="ngModel"
      >

      <!-- Validation state properties -->
      <div>Valid: {{ emailField.valid }}</div>
      <div>Invalid: {{ emailField.invalid }}</div>
      <div>Pristine: {{ emailField.pristine }}</div>
      <div>Dirty: {{ emailField.dirty }}</div>
      <div>Touched: {{ emailField.touched }}</div>
      <div>Untouched: {{ emailField.untouched }}</div>
      
      <!-- Errors object -->
      <div>Errors: {{ emailField.errors | json }}</div>
      
      <!-- CSS classes applied automatically -->
      <!-- 
        .ng-valid / .ng-invalid
        .ng-pristine / .ng-dirty
        .ng-touched / .ng-untouched
      -->
    </form>
  `,
  styles: [`
    input.ng-invalid.ng-touched {
      border: 2px solid red;
    }
    
    input.ng-valid.ng-touched {
      border: 2px solid green;
    }
  `]
})
export class ValidationStatesComponent {
  email = '';
}
```

## Displaying Validation Errors

### Error Messages with ngIf

```typescript
@Component({
  selector: 'app-error-display',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form>
      <div class="form-group">
        <label for="email">Email</label>
        <input 
          type="email" 
          id="email"
          [(ngModel)]="email" 
          name="email"
          required
          email
          #emailField="ngModel"
          class="form-control"
        >
        
        <!-- Show errors only when field is touched -->
        <div class="error-messages" *ngIf="emailField.touched">
          <div class="error" *ngIf="emailField.errors?.['required']">
            Email is required
          </div>
          <div class="error" *ngIf="emailField.errors?.['email']">
            Please enter a valid email address
          </div>
        </div>
      </div>
    </form>
  `,
  styles: [`
    .error {
      color: red;
      font-size: 0.875rem;
      margin-top: 0.25rem;
    }
    
    .form-control.ng-invalid.ng-touched {
      border-color: red;
    }
  `]
})
export class ErrorDisplayComponent {
  email = '';
}
```

### Reusable Error Component

```typescript
// error-message.component.ts
@Component({
  selector: 'app-error-message',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="error-message" *ngIf="shouldShowError()">
      <ng-content></ng-content>
    </div>
  `,
  styles: [`
    .error-message {
      color: #dc3545;
      font-size: 0.875rem;
      margin-top: 0.25rem;
    }
  `]
})
export class ErrorMessageComponent {
  @Input() control: any;
  @Input() error: string = '';

  shouldShowError(): boolean {
    return this.control?.touched && this.control?.errors?.[this.error];
  }
}

// Usage
@Component({
  selector: 'app-form-with-errors',
  standalone: true,
  imports: [FormsModule, CommonModule, ErrorMessageComponent],
  template: `
    <form>
      <input 
        type="email" 
        [(ngModel)]="email" 
        name="email"
        required
        email
        #emailField="ngModel"
      >
      
      <app-error-message [control]="emailField" error="required">
        Email is required
      </app-error-message>
      
      <app-error-message [control]="emailField" error="email">
        Invalid email format
      </app-error-message>
    </form>
  `
})
export class FormWithErrorsComponent {
  email = '';
}
```

## Form Submission

### Basic Form Submission

```typescript
@Component({
  selector: 'app-contact-form',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form #contactForm="ngForm" (ngSubmit)="onSubmit(contactForm)">
      <input 
        type="text" 
        [(ngModel)]="contact.name" 
        name="name"
        required
        #name="ngModel"
      >
      
      <input 
        type="email" 
        [(ngModel)]="contact.email" 
        name="email"
        required
        email
        #email="ngModel"
      >
      
      <textarea 
        [(ngModel)]="contact.message" 
        name="message"
        required
        #message="ngModel"
      ></textarea>

      <button type="submit" [disabled]="contactForm.invalid">
        Submit
      </button>
    </form>

    <div *ngIf="submitted">
      <h3>Form Submitted!</h3>
      <pre>{{ contact | json }}</pre>
    </div>
  `
})
export class ContactFormComponent {
  contact = {
    name: '',
    email: '',
    message: ''
  };

  submitted = false;

  onSubmit(form: NgForm) {
    if (form.valid) {
      console.log('Form submitted:', this.contact);
      this.submitted = true;
      
      // Reset form after submission
      form.reset();
    }
  }
}
```

### Form Reset and State Management

```typescript
@Component({
  selector: 'app-managed-form',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form #userForm="ngForm" (ngSubmit)="onSubmit(userForm)">
      <input 
        type="text" 
        [(ngModel)]="user.username" 
        name="username"
        required
      >
      
      <input 
        type="email" 
        [(ngModel)]="user.email" 
        name="email"
        required
        email
      >

      <div class="actions">
        <button type="submit" [disabled]="userForm.invalid">
          Submit
        </button>
        <button type="button" (click)="onReset(userForm)">
          Reset
        </button>
        <button type="button" (click)="onClear()">
          Clear
        </button>
      </div>
    </form>

    <div class="debug-info">
      <h4>Form State:</h4>
      <p>Valid: {{ userForm.valid }}</p>
      <p>Pristine: {{ userForm.pristine }}</p>
      <p>Submitted: {{ userForm.submitted }}</p>
    </div>
  `
})
export class ManagedFormComponent {
  user = {
    username: '',
    email: ''
  };

  onSubmit(form: NgForm) {
    if (form.valid) {
      console.log('Submitted:', this.user);
      // API call here
      form.resetForm(); // Reset form to pristine state
    }
  }

  onReset(form: NgForm) {
    form.resetForm(); // Reset form and validation states
  }

  onClear() {
    this.user = { username: '', email: '' }; // Clear model only
  }
}
```

## ngForm and ngModelGroup

### ngForm Directive

```typescript
@Component({
  selector: 'app-ngform-demo',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <!-- ngForm is automatically applied to <form> -->
    <form #myForm="ngForm">
      <input type="text" [(ngModel)]="name" name="name" required>
      
      <!-- Access form properties -->
      <p>Form valid: {{ myForm.valid }}</p>
      <p>Form value: {{ myForm.value | json }}</p>
      <p>Form submitted: {{ myForm.submitted }}</p>
    </form>
  `
})
export class NgFormDemoComponent {
  name = '';
}
```

### ngModelGroup for Nested Forms

```typescript
@Component({
  selector: 'app-nested-form',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form #userForm="ngForm" (ngSubmit)="onSubmit()">
      <!-- Personal Info Group -->
      <fieldset ngModelGroup="personalInfo" #personalInfo="ngModelGroup">
        <legend>Personal Information</legend>
        
        <input 
          type="text" 
          [(ngModel)]="user.personalInfo.firstName" 
          name="firstName"
          required
        >
        
        <input 
          type="text" 
          [(ngModel)]="user.personalInfo.lastName" 
          name="lastName"
          required
        >
        
        <input 
          type="date" 
          [(ngModel)]="user.personalInfo.birthDate" 
          name="birthDate"
        >

        <div *ngIf="personalInfo.invalid && personalInfo.touched">
          Personal information is incomplete
        </div>
      </fieldset>

      <!-- Address Group -->
      <fieldset ngModelGroup="address" #address="ngModelGroup">
        <legend>Address</legend>
        
        <input 
          type="text" 
          [(ngModel)]="user.address.street" 
          name="street"
          required
        >
        
        <input 
          type="text" 
          [(ngModel)]="user.address.city" 
          name="city"
          required
        >
        
        <input 
          type="text" 
          [(ngModel)]="user.address.zip" 
          name="zip"
          required
          pattern="[0-9]{5}"
        >

        <div *ngIf="address.invalid && address.touched">
          Address is incomplete
        </div>
      </fieldset>

      <button type="submit" [disabled]="userForm.invalid">
        Submit
      </button>
    </form>

    <div class="debug">
      <h4>Form Value:</h4>
      <pre>{{ userForm.value | json }}</pre>
    </div>
  `
})
export class NestedFormComponent {
  user = {
    personalInfo: {
      firstName: '',
      lastName: '',
      birthDate: ''
    },
    address: {
      street: '',
      city: '',
      zip: ''
    }
  };

  onSubmit() {
    console.log('User data:', this.user);
  }
}
```

### Dynamic Form Groups

```typescript
@Component({
  selector: 'app-dynamic-groups',
  standalone: true,
  imports: [FormsModule, CommonModule],
  template: `
    <form #orderForm="ngForm">
      <div *ngFor="let item of orderItems; let i = index">
        <fieldset [ngModelGroup]="'item' + i">
          <legend>Item {{ i + 1 }}</legend>
          
          <input 
            type="text" 
            [(ngModel)]="item.product" 
            [name]="'product' + i"
            required
          >
          
          <input 
            type="number" 
            [(ngModel)]="item.quantity" 
            [name]="'quantity' + i"
            required
            min="1"
          >
          
          <button type="button" (click)="removeItem(i)">
            Remove
          </button>
        </fieldset>
      </div>

      <button type="button" (click)="addItem()">
        Add Item
      </button>

      <button type="submit" [disabled]="orderForm.invalid">
        Place Order
      </button>
    </form>
  `
})
export class DynamicGroupsComponent {
  orderItems = [
    { product: '', quantity: 1 }
  ];

  addItem() {
    this.orderItems.push({ product: '', quantity: 1 });
  }

  removeItem(index: number) {
    this.orderItems.splice(index, 1);
  }
}
```

## Common Mistakes

### 1. Forgetting name Attribute

```typescript
// WRONG: Missing name attribute
<input type="text" [(ngModel)]="username">
// Error: ngModel requires a name attribute!

// RIGHT: Include name attribute
<input type="text" [(ngModel)]="username" name="username">
```

### 2. Not Importing FormsModule

```typescript
// WRONG: FormsModule not imported
@Component({
  standalone: true,
  imports: [CommonModule], // Missing FormsModule
  template: `<input [(ngModel)]="name">`
})

// RIGHT: Import FormsModule
@Component({
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `<input [(ngModel)]="name" name="name">`
})
```

### 3. Submitting Invalid Forms

```typescript
// WRONG: No validation check
onSubmit(form: NgForm) {
  // Submits even if invalid!
  this.saveData(form.value);
}

// RIGHT: Check validity
onSubmit(form: NgForm) {
  if (form.valid) {
    this.saveData(form.value);
  } else {
    // Mark all fields as touched to show errors
    Object.keys(form.controls).forEach(key => {
      form.controls[key].markAsTouched();
    });
  }
}
```

### 4. Not Using Template Reference Variables

```typescript
// WRONG: Can't access validation state
<input type="email" [(ngModel)]="email" name="email" required email>
<div>Email is invalid</div> <!-- Always shows -->

// RIGHT: Use template reference variable
<input 
  type="email" 
  [(ngModel)]="email" 
  name="email" 
  required 
  email
  #emailField="ngModel"
>
<div *ngIf="emailField.invalid && emailField.touched">
  Email is invalid
</div>
```

## Best Practices

### 1. Use for Simple Forms

```typescript
// Template-driven is good for simple forms
@Component({
  template: `
    <form #loginForm="ngForm" (ngSubmit)="onLogin(loginForm)">
      <input [(ngModel)]="username" name="username" required>
      <input type="password" [(ngModel)]="password" name="password" required>
      <button [disabled]="loginForm.invalid">Login</button>
    </form>
  `
})
```

### 2. Validate on Touch, Not on Pristine

```typescript
// Show errors only after user interaction
<div *ngIf="field.invalid && field.touched">
  <!-- Errors here -->
</div>
```

### 3. Disable Submit Button for Invalid Forms

```typescript
<button type="submit" [disabled]="myForm.invalid">
  Submit
</button>
```

### 4. Reset Form After Successful Submission

```typescript
onSubmit(form: NgForm) {
  if (form.valid) {
    this.apiService.save(form.value).subscribe({
      next: () => {
        form.resetForm(); // Reset to pristine state
        this.showSuccessMessage();
      }
    });
  }
}
```

## Interview Questions

### Q1: What's the difference between template-driven and reactive forms?

**Answer:**
- **Template-driven**: Form logic in template, uses directives (ngModel), simpler for basic forms, less testing-friendly
- **Reactive**: Form logic in component, uses FormControl/FormGroup, more powerful, better for complex validation and testing

### Q2: Why is the name attribute required with ngModel?

**Answer:** The name attribute is required so Angular can register the control with the parent form. It's used as the key in the form's value object and to track the control's state within the form.

### Q3: How do you display validation errors only after user interaction?

**Answer:** Check both `invalid` and `touched` states:
```typescript
<div *ngIf="field.invalid && field.touched">
  Error message
</div>
```
This ensures errors don't show immediately, only after the user has interacted with the field.

## Key Takeaways

1. Template-driven forms use directives in templates
2. Import **FormsModule** to use template-driven forms
3. **ngModel** requires a **name** attribute
4. Use **[(ngModel)]** for two-way binding
5. Access form state with template reference variables
6. Validate with HTML5 validators and directives
7. Show errors conditionally based on touched/invalid state
8. Use **ngModelGroup** for nested form structures
9. Disable submit button when form is invalid
10. Best for simple forms; use reactive forms for complex scenarios

## Resources

- [Angular Documentation - Template-Driven Forms](https://angular.io/guide/forms)
- [Angular Forms Guide](https://angular.io/guide/forms-overview)
- [Angular University - Template-Driven Forms](https://blog.angular-university.io/introduction-to-angular-2-forms-template-driven-vs-model-driven/)
- [MDN - HTML5 Form Validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation)
