# Angular Forms Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

Angular provides two approaches to handling forms: Template-Driven Forms and Reactive Forms. Understanding when and how to use each is crucial for building robust form interactions.

### Form Approaches

**Template-Driven**: Simpler, directive-based, suitable for basic forms
**Reactive**: More powerful, model-driven, better for complex scenarios

### Key Features

- Two-way data binding
- Built-in and custom validation
- Form state management (pristine, dirty, valid, etc.)
- Dynamic form controls
- Cross-field validation
- Async validation

## Common Interview Questions

### Q1: Compare Template-Driven Forms and Reactive Forms. When would you use each?

**What the interviewer wants to know:**
- Understanding of both approaches
- Ability to choose appropriate solution
- Knowledge of trade-offs

**Strong Answer:**

Angular offers two fundamentally different approaches to forms, each with distinct advantages and use cases.

**Template-Driven Forms:**

```typescript
// Component
import { Component } from '@angular/core';

@Component({
  selector: 'app-template-form',
  template: `
    <form #userForm="ngForm" (ngSubmit)="onSubmit(userForm)">
      <div>
        <label>Name:</label>
        <input 
          type="text" 
          name="name" 
          [(ngModel)]="user.name"
          required
          minlength="3"
          #nameField="ngModel">
        
        <div *ngIf="nameField.invalid && nameField.touched">
          <span *ngIf="nameField.errors?.['required']">Name is required</span>
          <span *ngIf="nameField.errors?.['minlength']">Name must be at least 3 characters</span>
        </div>
      </div>
      
      <div>
        <label>Email:</label>
        <input 
          type="email" 
          name="email" 
          [(ngModel)]="user.email"
          required
          email>
      </div>
      
      <button type="submit" [disabled]="userForm.invalid">Submit</button>
    </form>
  `
})
export class TemplateFormComponent {
  user = {
    name: '',
    email: ''
  };
  
  onSubmit(form: NgForm): void {
    if (form.valid) {
      console.log('Form submitted:', this.user);
    }
  }
}
```

**Reactive Forms:**

```typescript
// Component
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-reactive-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <div>
        <label>Name:</label>
        <input 
          type="text" 
          formControlName="name">
        
        <div *ngIf="name.invalid && name.touched">
          <span *ngIf="name.errors?.['required']">Name is required</span>
          <span *ngIf="name.errors?.['minlength']">
            Name must be at least 3 characters
          </span>
        </div>
      </div>
      
      <div>
        <label>Email:</label>
        <input 
          type="email" 
          formControlName="email">
        
        <div *ngIf="email.invalid && email.touched">
          <span *ngIf="email.errors?.['required']">Email is required</span>
          <span *ngIf="email.errors?.['email']">Invalid email format</span>
        </div>
      </div>
      
      <button type="submit" [disabled]="userForm.invalid">Submit</button>
    </form>
    
    <div>
      <h3>Form State</h3>
      <p>Valid: {{ userForm.valid }}</p>
      <p>Dirty: {{ userForm.dirty }}</p>
      <p>Touched: {{ userForm.touched }}</p>
    </div>
  `
})
export class ReactiveFormComponent implements OnInit {
  userForm: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.userForm = this.fb.group({
      name: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]]
    });
    
    // Listen to value changes
    this.userForm.valueChanges.subscribe(value => {
      console.log('Form changed:', value);
    });
    
    // Listen to specific field changes
    this.name.valueChanges.subscribe(value => {
      console.log('Name changed:', value);
    });
  }
  
  get name() {
    return this.userForm.get('name')!;
  }
  
  get email() {
    return this.userForm.get('email')!;
  }
  
  onSubmit(): void {
    if (this.userForm.valid) {
      console.log('Form submitted:', this.userForm.value);
    }
  }
}
```

**Comparison Table:**

| Aspect | Template-Driven | Reactive |
|--------|----------------|----------|
| **Setup** | Simpler, less code | More setup, verbose |
| **Logic Location** | Template | Component |
| **Data Flow** | Asynchronous | Synchronous |
| **Validation** | Directives | Functions |
| **Testing** | Harder (needs DOM) | Easier (pure logic) |
| **Dynamic Forms** | Difficult | Easy |
| **Scalability** | Limited | Excellent |
| **Type Safety** | Weaker | Stronger |
| **RxJS Integration** | Limited | Full |

**When to Use Template-Driven:**

```typescript
// ✅ Good use cases:

// 1. Simple forms
@Component({
  template: `
    <form #loginForm="ngForm" (ngSubmit)="login(loginForm)">
      <input name="username" [(ngModel)]="credentials.username" required>
      <input name="password" [(ngModel)]="credentials.password" type="password" required>
      <button [disabled]="loginForm.invalid">Login</button>
    </form>
  `
})
export class SimpleLoginComponent {
  credentials = { username: '', password: '' };
  
  login(form: NgForm): void {
    if (form.valid) {
      this.authService.login(this.credentials);
    }
  }
}

// 2. Quick prototypes
// 3. Forms with minimal validation
// 4. Teams more comfortable with directive syntax
// 5. Migration from AngularJS
```

**When to Use Reactive:**

```typescript
// ✅ Good use cases:

// 1. Complex validation logic
export class ComplexFormComponent implements OnInit {
  form: FormGroup;
  
  ngOnInit(): void {
    this.form = this.fb.group({
      password: ['', [Validators.required, Validators.minLength(8)]],
      confirmPassword: ['', Validators.required]
    }, {
      validators: this.passwordMatchValidator // Cross-field validation
    });
    
    // Dynamic behavior
    this.form.get('password')!.valueChanges.subscribe(value => {
      if (value.length > 0) {
        this.form.get('confirmPassword')!.setValidators([
          Validators.required,
          this.matchPasswordValidator(value)
        ]);
      }
    });
  }
  
  passwordMatchValidator(group: AbstractControl): ValidationErrors | null {
    const password = group.get('password')?.value;
    const confirm = group.get('confirmPassword')?.value;
    return password === confirm ? null : { passwordMismatch: true };
  }
}

// 2. Dynamic forms (add/remove fields)
// 3. Complex data transformations
// 4. Heavy testing requirements
// 5. Observable-based workflows
// 6. Large enterprise applications
```

**Real-World Decision Example:**

```typescript
// Scenario: User profile form with different complexity levels

// Simple profile (Template-Driven)
@Component({
  template: `
    <form #profileForm="ngForm" (ngSubmit)="save(profileForm)">
      <input name="displayName" [(ngModel)]="profile.displayName" required>
      <input name="bio" [(ngModel)]="profile.bio">
      <button [disabled]="profileForm.invalid">Save</button>
    </form>
  `
})
export class SimpleProfileComponent {
  profile = { displayName: '', bio: '' };
  save(form: NgForm): void {}
}

// Complex profile (Reactive)
@Component({
  template: `
    <form [formGroup]="profileForm" (ngSubmit)="save()">
      <input formControlName="displayName">
      
      <div formGroupName="address">
        <input formControlName="street">
        <input formControlName="city">
        <input formControlName="country">
      </div>
      
      <div formArrayName="socialLinks">
        <div *ngFor="let link of socialLinks.controls; let i = index" [formGroupName]="i">
          <input formControlName="platform">
          <input formControlName="url">
          <button (click)="removeSocialLink(i)">Remove</button>
        </div>
      </div>
      
      <button (click)="addSocialLink()">Add Social Link</button>
      <button [disabled]="profileForm.invalid">Save</button>
    </form>
  `
})
export class ComplexProfileComponent implements OnInit {
  profileForm: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.profileForm = this.fb.group({
      displayName: ['', [Validators.required, Validators.minLength(3)]],
      address: this.fb.group({
        street: [''],
        city: ['', Validators.required],
        country: ['', Validators.required]
      }),
      socialLinks: this.fb.array([])
    });
  }
  
  get socialLinks(): FormArray {
    return this.profileForm.get('socialLinks') as FormArray;
  }
  
  addSocialLink(): void {
    this.socialLinks.push(this.fb.group({
      platform: ['', Validators.required],
      url: ['', [Validators.required, Validators.pattern(/^https?:\/\//)]]
    }));
  }
  
  removeSocialLink(index: number): void {
    this.socialLinks.removeAt(index);
  }
  
  save(): void {
    if (this.profileForm.valid) {
      console.log(this.profileForm.value);
    }
  }
}
```

**Hybrid Approach:**

```typescript
// You can mix both approaches in the same application

// Simple search form (Template-Driven)
@Component({
  selector: 'app-search',
  template: `<input name="query" [(ngModel)]="query" (ngModelChange)="search()">`
})
export class SearchComponent {
  query = '';
  search(): void {}
}

// Complex checkout form (Reactive)
@Component({
  selector: 'app-checkout'
})
export class CheckoutComponent {
  checkoutForm: FormGroup = this.fb.group({
    billing: this.fb.group({ /* ... */ }),
    shipping: this.fb.group({ /* ... */ }),
    payment: this.fb.group({ /* ... */ })
  });
  
  constructor(private fb: FormBuilder) {}
}
```

**Follow-up Questions:**

1. **Can you use Reactive Forms with ngModel?**
   - Technically yes, but not recommended
   - Creates confusion about single source of truth
   - Defeats purpose of reactive approach
   - Angular warns against mixing approaches

2. **How do you migrate from Template-Driven to Reactive?**
   - Start with FormBuilder to create FormGroup
   - Replace ngModel with formControlName
   - Move validation from directives to Validators
   - Transfer submit logic to component
   - Update tests to use form controls

3. **Which approach is better for performance?**
   - Reactive forms are slightly more performant
   - Template-driven has more digest cycles
   - Difference is negligible for most apps
   - Choose based on maintainability, not performance

---

### Q2: Explain FormBuilder, FormGroup, FormControl, and FormArray with examples.

**What the interviewer wants to know:**
- Understanding of Reactive Forms building blocks
- Ability to structure complex forms
- Knowledge of dynamic form patterns

**Strong Answer:**

Reactive Forms are built on four fundamental classes that work together to create powerful form structures.

**FormControl - Single Input Field:**

```typescript
// Basic FormControl
import { FormControl, Validators } from '@angular/forms';

// Simple control
const nameControl = new FormControl('');

// With initial value
const emailControl = new FormControl('user@example.com');

// With validators
const passwordControl = new FormControl('', [
  Validators.required,
  Validators.minLength(8)
]);

// With options
const ageControl = new FormControl(
  { value: 25, disabled: false }, // Initial state
  [Validators.required, Validators.min(18)], // Sync validators
  [this.ageValidator.bind(this)] // Async validators
);

// Usage in component
@Component({
  template: `
    <input [formControl]="nameControl" type="text">
    <p *ngIf="nameControl.invalid">Invalid name</p>
  `
})
export class ControlExampleComponent {
  nameControl = new FormControl('', Validators.required);
  
  ngOnInit(): void {
    // Listen to changes
    this.nameControl.valueChanges.subscribe(value => {
      console.log('Name changed:', value);
    });
    
    // Programmatic updates
    this.nameControl.setValue('John');
    
    // Patch value (silently)
    this.nameControl.patchValue('Jane', { emitEvent: false });
    
    // Check state
    console.log('Valid:', this.nameControl.valid);
    console.log('Dirty:', this.nameControl.dirty);
    console.log('Touched:', this.nameControl.touched);
    
    // Enable/disable
    this.nameControl.disable();
    this.nameControl.enable();
    
    // Reset
    this.nameControl.reset();
  }
}
```

**FormGroup - Multiple Controls:**

```typescript
// Basic FormGroup
import { FormGroup, FormControl } from '@angular/forms';

// Manual creation
const userForm = new FormGroup({
  name: new FormControl(''),
  email: new FormControl(''),
  age: new FormControl(0)
});

// Access controls
const nameControl = userForm.get('name');

// Get values
const formValue = userForm.value; // { name: '', email: '', age: 0 }

// Usage in component
@Component({
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="name">
      <input formControlName="email">
      <input formControlName="age" type="number">
      <button [disabled]="userForm.invalid">Submit</button>
    </form>
  `
})
export class GroupExampleComponent implements OnInit {
  userForm = new FormGroup({
    name: new FormControl('', Validators.required),
    email: new FormControl('', [Validators.required, Validators.email]),
    age: new FormControl(0, Validators.min(0))
  });
  
  ngOnInit(): void {
    // Listen to all changes
    this.userForm.valueChanges.subscribe(value => {
      console.log('Form changed:', value);
    });
    
    // Set all values
    this.userForm.setValue({
      name: 'John',
      email: 'john@example.com',
      age: 30
    });
    
    // Patch partial values
    this.userForm.patchValue({
      name: 'Jane' // Only updates name
    });
    
    // Validate entire form
    this.userForm.updateValueAndValidity();
  }
  
  onSubmit(): void {
    if (this.userForm.valid) {
      console.log('Submitted:', this.userForm.value);
    }
  }
}
```

**Nested FormGroups:**

```typescript
@Component({
  template: `
    <form [formGroup]="registrationForm">
      <div formGroupName="personalInfo">
        <input formControlName="firstName">
        <input formControlName="lastName">
      </div>
      
      <div formGroupName="address">
        <input formControlName="street">
        <input formControlName="city">
        <input formControlName="zipCode">
      </div>
      
      <div formGroupName="credentials">
        <input formControlName="username">
        <input formControlName="password" type="password">
      </div>
    </form>
  `
})
export class NestedFormComponent implements OnInit {
  registrationForm = new FormGroup({
    personalInfo: new FormGroup({
      firstName: new FormControl('', Validators.required),
      lastName: new FormControl('', Validators.required)
    }),
    address: new FormGroup({
      street: new FormControl(''),
      city: new FormControl('', Validators.required),
      zipCode: new FormControl('', [
        Validators.required,
        Validators.pattern(/^\d{5}$/)
      ])
    }),
    credentials: new FormGroup({
      username: new FormControl('', [
        Validators.required,
        Validators.minLength(3)
      ]),
      password: new FormControl('', [
        Validators.required,
        Validators.minLength(8)
      ])
    })
  });
  
  ngOnInit(): void {
    // Access nested controls
    const firstNameControl = this.registrationForm.get('personalInfo.firstName');
    const addressGroup = this.registrationForm.get('address') as FormGroup;
    
    // Get nested values
    const personalInfo = this.registrationForm.get('personalInfo')?.value;
    console.log(personalInfo); // { firstName: '', lastName: '' }
  }
}
```

**FormBuilder - Simplified Creation:**

```typescript
// FormBuilder reduces boilerplate
import { FormBuilder } from '@angular/forms';

@Component({})
export class BuilderExampleComponent {
  constructor(private fb: FormBuilder) {}
  
  // Without FormBuilder
  userFormManual = new FormGroup({
    name: new FormControl('', Validators.required),
    email: new FormControl('', [Validators.required, Validators.email])
  });
  
  // With FormBuilder (cleaner)
  userFormBuilder = this.fb.group({
    name: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]]
  });
  
  // Complex nested form with FormBuilder
  profileForm = this.fb.group({
    personalInfo: this.fb.group({
      firstName: ['', Validators.required],
      lastName: ['', Validators.required],
      dateOfBirth: [null]
    }),
    contactInfo: this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      phone: ['', Validators.pattern(/^\d{10}$/)],
      address: this.fb.group({
        street: [''],
        city: [''],
        state: [''],
        zipCode: ['']
      })
    }),
    preferences: this.fb.group({
      newsletter: [false],
      notifications: [true]
    })
  });
}
```

**FormArray - Dynamic List of Controls:**

```typescript
import { FormArray } from '@angular/forms';

@Component({
  template: `
    <form [formGroup]="surveyForm">
      <div formArrayName="questions">
        <div *ngFor="let question of questions.controls; let i = index" [formGroupName]="i">
          <h4>Question {{ i + 1 }}</h4>
          <input formControlName="text" placeholder="Question text">
          <input formControlName="answer" placeholder="Answer">
          <button (click)="removeQuestion(i)">Remove</button>
        </div>
      </div>
      
      <button (click)="addQuestion()">Add Question</button>
      <button (click)="submit()">Submit Survey</button>
    </form>
    
    <div>
      <h3>Form Value:</h3>
      <pre>{{ surveyForm.value | json }}</pre>
    </div>
  `
})
export class ArrayExampleComponent implements OnInit {
  surveyForm: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.surveyForm = this.fb.group({
      title: ['', Validators.required],
      questions: this.fb.array([
        this.createQuestion()
      ])
    });
  }
  
  get questions(): FormArray {
    return this.surveyForm.get('questions') as FormArray;
  }
  
  createQuestion(): FormGroup {
    return this.fb.group({
      text: ['', Validators.required],
      answer: ['']
    });
  }
  
  addQuestion(): void {
    this.questions.push(this.createQuestion());
  }
  
  removeQuestion(index: number): void {
    this.questions.removeAt(index);
  }
  
  submit(): void {
    console.log(this.surveyForm.value);
    // {
    //   title: 'My Survey',
    //   questions: [
    //     { text: 'Question 1', answer: 'Answer 1' },
    //     { text: 'Question 2', answer: 'Answer 2' }
    //   ]
    // }
  }
}
```

**Real-World FormArray Examples:**

**1. Shopping Cart Items:**

```typescript
@Component({
  template: `
    <form [formGroup]="orderForm">
      <div formArrayName="items">
        <div *ngFor="let item of items.controls; let i = index" [formGroupName]="i">
          <input formControlName="product" placeholder="Product">
          <input formControlName="quantity" type="number" min="1">
          <input formControlName="price" type="number" min="0" step="0.01">
          <span>Total: {{ calculateItemTotal(i) | currency }}</span>
          <button (click)="removeItem(i)">Remove</button>
        </div>
      </div>
      
      <button (click)="addItem()">Add Item</button>
      <p>Order Total: {{ calculateOrderTotal() | currency }}</p>
    </form>
  `
})
export class OrderFormComponent implements OnInit {
  orderForm: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.orderForm = this.fb.group({
      customerName: ['', Validators.required],
      items: this.fb.array([])
    });
    
    // Add initial item
    this.addItem();
  }
  
  get items(): FormArray {
    return this.orderForm.get('items') as FormArray;
  }
  
  addItem(): void {
    const itemGroup = this.fb.group({
      product: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]],
      price: [0, [Validators.required, Validators.min(0)]]
    });
    
    this.items.push(itemGroup);
  }
  
  removeItem(index: number): void {
    this.items.removeAt(index);
  }
  
  calculateItemTotal(index: number): number {
    const item = this.items.at(index).value;
    return item.quantity * item.price;
  }
  
  calculateOrderTotal(): number {
    return this.items.controls.reduce((total, control) => {
      const item = control.value;
      return total + (item.quantity * item.price);
    }, 0);
  }
}
```

**2. Skills/Tags Editor:**

```typescript
@Component({
  template: `
    <form [formGroup]="profileForm">
      <div formArrayName="skills">
        <div *ngFor="let skill of skills.controls; let i = index">
          <input [formControl]="skill">
          <button (click)="removeSkill(i)">×</button>
        </div>
      </div>
      
      <input #skillInput (keyup.enter)="addSkill(skillInput.value); skillInput.value = ''">
      <button (click)="addSkill(skillInput.value); skillInput.value = ''">Add Skill</button>
    </form>
  `
})
export class SkillsEditorComponent implements OnInit {
  profileForm: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.profileForm = this.fb.group({
      name: [''],
      skills: this.fb.array([])
    });
  }
  
  get skills(): FormArray {
    return this.profileForm.get('skills') as FormArray;
  }
  
  addSkill(skill: string): void {
    if (skill && skill.trim()) {
      this.skills.push(this.fb.control(skill.trim(), Validators.required));
    }
  }
  
  removeSkill(index: number): void {
    this.skills.removeAt(index);
  }
}
```

**3. Nested FormArray (Advanced):**

```typescript
// FormArray containing FormGroups with nested FormArrays
@Component({
  template: `
    <form [formGroup]="courseForm">
      <div formArrayName="modules">
        <div *ngFor="let module of modules.controls; let i = index" [formGroupName]="i">
          <h3>Module {{ i + 1 }}</h3>
          <input formControlName="title">
          
          <div formArrayName="lessons">
            <div *ngFor="let lesson of getModuleLessons(i).controls; let j = index" [formGroupName]="j">
              <input formControlName="name" placeholder="Lesson name">
              <input formControlName="duration" type="number" placeholder="Duration (minutes)">
              <button (click)="removeLesson(i, j)">Remove Lesson</button>
            </div>
          </div>
          
          <button (click)="addLesson(i)">Add Lesson</button>
          <button (click)="removeModule(i)">Remove Module</button>
        </div>
      </div>
      
      <button (click)="addModule()">Add Module</button>
    </form>
  `
})
export class CourseFormComponent implements OnInit {
  courseForm: FormGroup;
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.courseForm = this.fb.group({
      title: ['', Validators.required],
      modules: this.fb.array([])
    });
  }
  
  get modules(): FormArray {
    return this.courseForm.get('modules') as FormArray;
  }
  
  getModuleLessons(moduleIndex: number): FormArray {
    return this.modules.at(moduleIndex).get('lessons') as FormArray;
  }
  
  addModule(): void {
    const moduleGroup = this.fb.group({
      title: ['', Validators.required],
      lessons: this.fb.array([])
    });
    
    this.modules.push(moduleGroup);
  }
  
  removeModule(index: number): void {
    this.modules.removeAt(index);
  }
  
  addLesson(moduleIndex: number): void {
    const lessonGroup = this.fb.group({
      name: ['', Validators.required],
      duration: [0, Validators.min(1)]
    });
    
    this.getModuleLessons(moduleIndex).push(lessonGroup);
  }
  
  removeLesson(moduleIndex: number, lessonIndex: number): void {
    this.getModuleLessons(moduleIndex).removeAt(lessonIndex);
  }
}
```

**Follow-up Questions:**

1. **When would you use FormControl directly vs FormBuilder?**
   - Simple, single controls: either works
   - Complex nested forms: FormBuilder is cleaner
   - Dynamic forms: FormBuilder is more maintainable
   - FormBuilder is generally preferred

2. **How do you handle FormArray validation?**
   ```typescript
   // Validate individual items
   this.fb.array([], [Validators.required, Validators.minLength(1)])
   
   // Custom FormArray validator
   validateArray(array: FormArray): ValidationErrors | null {
     return array.length > 0 ? null : { empty: true };
   }
   ```

3. **Can you dynamically add validators to FormGroups?**
   ```typescript
   // Add validators
   this.userForm.setValidators([this.customValidator]);
   
   // Clear validators
   this.userForm.clearValidators();
   
   // Update validation
   this.userForm.updateValueAndValidity();
   ```

---

### Q3: Explain custom validators and async validators with real-world examples.

**What the interviewer wants to know:**
- Ability to create custom validation logic
- Understanding of synchronous vs asynchronous validation
- Knowledge of validation patterns

**Strong Answer:**

Angular's built-in validators cover common cases, but real applications often need custom validation logic. Understanding how to create both synchronous and asynchronous validators is essential.

**Synchronous Custom Validators:**

```typescript
// Simple custom validator function
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function forbiddenNameValidator(forbiddenName: string): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = control.value?.toLowerCase() === forbiddenName.toLowerCase();
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Usage
this.nameControl = new FormControl('', [
  Validators.required,
  forbiddenNameValidator('admin')
]);

// In template
<div *ngIf="nameControl.errors?.['forbiddenName']">
  Name "{{ nameControl.errors?.['forbiddenName'].value }}" is not allowed
</div>
```

**Real-World Sync Validators:**

**1. Password Strength Validator:**

```typescript
export function passwordStrengthValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value;
    
    if (!value) {
      return null; // Don't validate empty values
    }
    
    const hasUpperCase = /[A-Z]/.test(value);
    const hasLowerCase = /[a-z]/.test(value);
    const hasNumeric = /[0-9]/.test(value);
    const hasSpecial = /[!@#$%^&*(),.?":{}|<>]/.test(value);
    const isLongEnough = value.length >= 8;
    
    const passwordValid = hasUpperCase && hasLowerCase && hasNumeric && hasSpecial && isLongEnough;
    
    return !passwordValid ? {
      passwordStrength: {
        hasUpperCase,
        hasLowerCase,
        hasNumeric,
        hasSpecial,
        isLongEnough
      }
    } : null;
  };
}

// Usage in component
@Component({
  template: `
    <input [formControl]="passwordControl" type="password">
    
    <div *ngIf="passwordControl.errors?.['passwordStrength'] as errors">
      <p>Password must have:</p>
      <ul>
        <li [class.valid]="errors.hasUpperCase">Uppercase letter</li>
        <li [class.valid]="errors.hasLowerCase">Lowercase letter</li>
        <li [class.valid]="errors.hasNumeric">Number</li>
        <li [class.valid]="errors.hasSpecial">Special character</li>
        <li [class.valid]="errors.isLongEnough">At least 8 characters</li>
      </ul>
    </div>
  `
})
export class PasswordComponent {
  passwordControl = new FormControl('', [
    Validators.required,
    passwordStrengthValidator()
  ]);
}
```

**2. Date Range Validator:**

```typescript
export function dateRangeValidator(): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const startDate = group.get('startDate')?.value;
    const endDate = group.get('endDate')?.value;
    
    if (!startDate || !endDate) {
      return null;
    }
    
    const start = new Date(startDate);
    const end = new Date(endDate);
    
    if (start >= end) {
      return { dateRange: 'End date must be after start date' };
    }
    
    return null;
  };
}

// Usage
this.dateForm = this.fb.group({
  startDate: ['', Validators.required],
  endDate: ['', Validators.required]
}, {
  validators: dateRangeValidator()
});
```

**3. Credit Card Validator:**

```typescript
export function creditCardValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value?.replace(/\s/g, ''); // Remove spaces
    
    if (!value) {
      return null;
    }
    
    // Luhn algorithm
    let sum = 0;
    let isEven = false;
    
    for (let i = value.length - 1; i >= 0; i--) {
      let digit = parseInt(value[i], 10);
      
      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }
      
      sum += digit;
      isEven = !isEven;
    }
    
    const valid = sum % 10 === 0;
    
    return valid ? null : { invalidCreditCard: true };
  };
}

// Usage
this.cardControl = new FormControl('', [
  Validators.required,
  Validators.pattern(/^\d{13,19}$/), // 13-19 digits
  creditCardValidator()
]);
```

**Cross-Field Validators:**

```typescript
// Password confirmation validator
export function passwordMatchValidator(): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const password = group.get('password')?.value;
    const confirmPassword = group.get('confirmPassword')?.value;
    
    if (!password || !confirmPassword) {
      return null;
    }
    
    return password === confirmPassword ? null : { passwordMismatch: true };
  };
}

// Usage
this.registrationForm = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', [Validators.required, Validators.minLength(8)]],
  confirmPassword: ['', Validators.required]
}, {
  validators: passwordMatchValidator()
});

// In template
<div *ngIf="registrationForm.errors?.['passwordMismatch'] && registrationForm.get('confirmPassword')?.touched">
  Passwords do not match
</div>
```

**Asynchronous Validators:**

```typescript
// Check username availability (simulated)
export class UsernameValidator {
  static createValidator(userService: UserService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }
      
      return timer(500).pipe( // Debounce
        switchMap(() => userService.checkUsernameAvailability(control.value)),
        map(isAvailable => isAvailable ? null : { usernameTaken: true }),
        catchError(() => of(null)) // Ignore errors
      );
    };
  }
}

// Service
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}
  
  checkUsernameAvailability(username: string): Observable<boolean> {
    return this.http.get<{ available: boolean }>(`/api/check-username/${username}`)
      .pipe(
        map(response => response.available)
      );
  }
}

// Usage in component
@Component({
  template: `
    <input [formControl]="usernameControl">
    
    <span *ngIf="usernameControl.pending">Checking availability...</span>
    <span *ngIf="usernameControl.errors?.['usernameTaken']">Username is taken</span>
    <span *ngIf="usernameControl.valid">Username is available ✓</span>
  `
})
export class SignupComponent {
  usernameControl: FormControl;
  
  constructor(private userService: UserService) {
    this.usernameControl = new FormControl('', {
      validators: [Validators.required, Validators.minLength(3)],
      asyncValidators: [UsernameValidator.createValidator(this.userService)],
      updateOn: 'blur' // Only validate on blur to reduce API calls
    });
  }
}
```

**Real-World Async Validators:**

**1. Email Uniqueness:**

```typescript
export class EmailValidator {
  static createValidator(authService: AuthService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }
      
      return timer(300).pipe(
        switchMap(() => authService.checkEmailExists(control.value)),
        map(exists => exists ? { emailExists: true } : null),
        catchError(() => of(null))
      );
    };
  }
}

@Component({})
export class RegistrationComponent {
  registrationForm = this.fb.group({
    email: ['', {
      validators: [Validators.required, Validators.email],
      asyncValidators: [EmailValidator.createValidator(this.authService)],
      updateOn: 'blur'
    }]
  });
  
  constructor(
    private fb: FormBuilder,
    private authService: AuthService
  ) {}
}
```

**2. Domain Availability:**

```typescript
export class DomainValidator {
  static createValidator(domainService: DomainService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      const domain = control.value;
      
      if (!domain) {
        return of(null);
      }
      
      return timer(500).pipe(
        switchMap(() => domainService.checkDomainAvailability(domain)),
        map(result => {
          if (!result.available) {
            return { domainTaken: true };
          }
          if (result.price > 100) {
            return { domainTooExpensive: { price: result.price } };
          }
          return null;
        }),
        catchError(() => of({ domainCheckFailed: true }))
      );
    };
  }
}

@Component({
  template: `
    <input [formControl]="domainControl">
    
    <div *ngIf="domainControl.pending">
      Checking domain availability...
    </div>
    
    <div *ngIf="domainControl.errors as errors">
      <span *ngIf="errors['domainTaken']">Domain is not available</span>
      <span *ngIf="errors['domainTooExpensive']">
        Domain is too expensive: ${{ errors['domainTooExpensive'].price }}
      </span>
      <span *ngIf="errors['domainCheckFailed']">
        Could not check domain availability
      </span>
    </div>
  `
})
export class DomainSearchComponent {
  domainControl: FormControl;
  
  constructor(private domainService: DomainService) {
    this.domainControl = new FormControl('', {
      validators: [Validators.required],
      asyncValidators: [DomainValidator.createValidator(this.domainService)]
    });
  }
}
```

**3. Coupon Code Validator:**

```typescript
export class CouponValidator {
  static createValidator(checkoutService: CheckoutService): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      const code = control.value;
      
      if (!code) {
        return of(null);
      }
      
      return checkoutService.validateCoupon(code).pipe(
        map(result => {
          if (!result.valid) {
            return { invalidCoupon: true };
          }
          if (result.expired) {
            return { couponExpired: true };
          }
          if (result.minPurchase && result.minPurchase > this.getCartTotal()) {
            return { minPurchaseNotMet: { required: result.minPurchase } };
          }
          return null;
        }),
        catchError(() => of({ couponValidationFailed: true }))
      );
    };
  }
}
```

**Combining Sync and Async Validators:**

```typescript
@Component({})
export class ComprehensiveFormComponent {
  userForm = this.fb.group({
    username: ['', {
      validators: [
        Validators.required,
        Validators.minLength(3),
        Validators.maxLength(20),
        Validators.pattern(/^[a-zA-Z0-9_]+$/)
      ],
      asyncValidators: [
        UsernameValidator.createValidator(this.userService)
      ],
      updateOn: 'blur'
    }],
    email: ['', {
      validators: [
        Validators.required,
        Validators.email
      ],
      asyncValidators: [
        EmailValidator.createValidator(this.authService)
      ],
      updateOn: 'blur'
    }],
    password: ['', [
      Validators.required,
      passwordStrengthValidator()
    ]],
    confirmPassword: ['', Validators.required]
  }, {
    validators: passwordMatchValidator()
  });
  
  constructor(
    private fb: FormBuilder,
    private userService: UserService,
    private authService: AuthService
  ) {}
}
```

**Testing Custom Validators:**

```typescript
describe('Custom Validators', () => {
  describe('passwordStrengthValidator', () => {
    it('should return null for valid password', () => {
      const control = new FormControl('Password123!');
      const result = passwordStrengthValidator()(control);
      expect(result).toBeNull();
    });
    
    it('should return error for weak password', () => {
      const control = new FormControl('weak');
      const result = passwordStrengthValidator()(control);
      expect(result).toEqual({
        passwordStrength: jasmine.objectContaining({
          isLongEnough: false
        })
      });
    });
  });
  
  describe('UsernameValidator', () => {
    it('should return error when username is taken', fakeAsync(() => {
      const userService = jasmine.createSpyObj('UserService', ['checkUsernameAvailability']);
      userService.checkUsernameAvailability.and.returnValue(of(false));
      
      const control = new FormControl('john');
      const validator = UsernameValidator.createValidator(userService);
      
      let result: ValidationErrors | null = null;
      validator(control).subscribe(r => result = r);
      
      tick(500); // Wait for debounce
      
      expect(result).toEqual({ usernameTaken: true });
    }));
  });
});
```

**Follow-up Questions:**

1. **How do you optimize async validators to reduce API calls?**
   - Use updateOn: 'blur' instead of 'change'
   - Add debounce with timer()
   - Use switchMap to cancel previous requests
   - Cache validation results
   - Validate only when sync validators pass

2. **Can you have multiple async validators on one control?**
   - Yes, provide array of async validators
   - They run in sequence, not parallel
   - First error stops execution
   - Consider performance implications

3. **How do you handle validation errors from the server?**
   ```typescript
   onSubmit(): void {
     this.service.submit(this.form.value).subscribe({
       error: (response) => {
         if (response.errors) {
           Object.keys(response.errors).forEach(key => {
             this.form.get(key)?.setErrors({
               serverError: response.errors[key]
             });
           });
         }
       }
     });
   }
   ```

---

## Advanced Questions

### Q4: How do you implement dynamic forms that change based on user input or external data?

**What the interviewer wants to know:**
- Ability to build complex, adaptive forms
- Understanding of FormArray and dynamic controls
- Knowledge of real-world form patterns

**Strong Answer:**

Dynamic forms adapt their structure, validation, or behavior based on runtime conditions. This is common in configuration forms, surveys, and multi-step wizards.

**Pattern 1: Conditional Fields:**

```typescript
@Component({
  template: `
    <form [formGroup]="form">
      <select formControlName="country">
        <option value="">Select country</option>
        <option value="US">United States</option>
        <option value="CA">Canada</option>
        <option value="Other">Other</option>
      </select>
      
      <!-- Show state dropdown only for US -->
      <select *ngIf="showStateField" formControlName="state">
        <option *ngFor="let state of usStates" [value]="state">{{ state }}</option>
      </select>
      
      <!-- Show province field for Canada -->
      <input *ngIf="showProvinceField" formControlName="province" placeholder="Province">
      
      <!-- Show custom country field for Other -->
      <input *ngIf="showCustomCountryField" formControlName="customCountry" placeholder="Country name">
    </form>
  `
})
export class DynamicFieldsComponent implements OnInit {
  form: FormGroup;
  usStates = ['CA', 'NY', 'TX', 'FL'];
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.form = this.fb.group({
      country: ['', Validators.required]
    });
    
    // React to country changes
    this.form.get('country')!.valueChanges.subscribe(country => {
      this.updateFormBasedOnCountry(country);
    });
  }
  
  get showStateField(): boolean {
    return this.form.get('country')?.value === 'US';
  }
  
  get showProvinceField(): boolean {
    return this.form.get('country')?.value === 'CA';
  }
  
  get showCustomCountryField(): boolean {
    return this.form.get('country')?.value === 'Other';
  }
  
  private updateFormBasedOnCountry(country: string): void {
    // Remove all optional fields
    if (this.form.contains('state')) {
      this.form.removeControl('state');
    }
    if (this.form.contains('province')) {
      this.form.removeControl('province');
    }
    if (this.form.contains('customCountry')) {
      this.form.removeControl('customCountry');
    }
    
    // Add appropriate field based on country
    if (country === 'US') {
      this.form.addControl('state', this.fb.control('', Validators.required));
    } else if (country === 'CA') {
      this.form.addControl('province', this.fb.control('', Validators.required));
    } else if (country === 'Other') {
      this.form.addControl('customCountry', this.fb.control('', Validators.required));
    }
  }
}
```

**Pattern 2: Dynamic Validation:**

```typescript
@Component({})
export class DynamicValidationComponent implements OnInit {
  form: FormGroup;
  
  ngOnInit(): void {
    this.form = this.fb.group({
      shippingMethod: ['standard'],
      address: [''],
      email: ['', Validators.email],
      phone: ['']
    });
    
    // Update validation based on shipping method
    this.form.get('shippingMethod')!.valueChanges.subscribe(method => {
      const addressControl = this.form.get('address')!;
      const emailControl = this.form.get('email')!;
      const phoneControl = this.form.get('phone')!;
      
      if (method === 'pickup') {
        // Pickup: phone required, address optional
        addressControl.clearValidators();
        phoneControl.setValidators([Validators.required, Validators.pattern(/^\d{10}$/)]);
      } else if (method === 'delivery') {
        // Delivery: address required, phone optional
        addressControl.setValidators([Validators.required, Validators.minLength(10)]);
        phoneControl.clearValidators();
      } else if (method === 'email') {
        // Email: email required, others optional
        addressControl.clearValidators();
        phoneControl.clearValidators();
        emailControl.setValidators([Validators.required, Validators.email]);
      }
      
      // Update validity
      addressControl.updateValueAndValidity();
      phoneControl.updateValueAndValidity();
      emailControl.updateValueAndValidity();
    });
  }
}
```

**Pattern 3: Form from API/JSON:**

```typescript
interface FieldConfig {
  type: 'text' | 'number' | 'email' | 'select' | 'checkbox';
  name: string;
  label: string;
  value?: any;
  options?: { label: string; value: any }[];
  validators?: {
    required?: boolean;
    min?: number;
    max?: number;
    minLength?: number;
    maxLength?: number;
    pattern?: string;
  };
}

@Component({
  template: `
    <form [formGroup]="dynamicForm" (ngSubmit)="onSubmit()">
      <div *ngFor="let field of fields">
        <label>{{ field.label }}</label>
        
        <input *ngIf="field.type === 'text'" 
               [formControlName]="field.name"
               type="text">
        
        <input *ngIf="field.type === 'number'" 
               [formControlName]="field.name"
               type="number">
        
        <input *ngIf="field.type === 'email'" 
               [formControlName]="field.name"
               type="email">
        
        <select *ngIf="field.type === 'select'" 
                [formControlName]="field.name">
          <option *ngFor="let opt of field.options" [value]="opt.value">
            {{ opt.label }}
          </option>
        </select>
        
        <input *ngIf="field.type === 'checkbox'" 
               [formControlName]="field.name"
               type="checkbox">
        
        <div *ngIf="getControl(field.name)?.invalid && getControl(field.name)?.touched">
          Field is invalid
        </div>
      </div>
      
      <button [disabled]="dynamicForm.invalid">Submit</button>
    </form>
  `
})
export class DynamicFormComponent implements OnInit {
  fields: FieldConfig[] = [];
  dynamicForm: FormGroup;
  
  constructor(
    private fb: FormBuilder,
    private formService: FormService
  ) {}
  
  ngOnInit(): void {
    // Load form configuration from API
    this.formService.getFormConfig().subscribe(fields => {
      this.fields = fields;
      this.dynamicForm = this.createForm(fields);
    });
  }
  
  private createForm(fields: FieldConfig[]): FormGroup {
    const group: any = {};
    
    fields.forEach(field => {
      const validators = this.createValidators(field.validators || {});
      group[field.name] = [field.value || '', validators];
    });
    
    return this.fb.group(group);
  }
  
  private createValidators(config: any): ValidatorFn[] {
    const validators: ValidatorFn[] = [];
    
    if (config.required) {
      validators.push(Validators.required);
    }
    if (config.min !== undefined) {
      validators.push(Validators.min(config.min));
    }
    if (config.max !== undefined) {
      validators.push(Validators.max(config.max));
    }
    if (config.minLength) {
      validators.push(Validators.minLength(config.minLength));
    }
    if (config.maxLength) {
      validators.push(Validators.maxLength(config.maxLength));
    }
    if (config.pattern) {
      validators.push(Validators.pattern(config.pattern));
    }
    
    return validators;
  }
  
  getControl(name: string): AbstractControl | null {
    return this.dynamicForm.get(name);
  }
  
  onSubmit(): void {
    if (this.dynamicForm.valid) {
      console.log('Form submitted:', this.dynamicForm.value);
    }
  }
}
```

**Pattern 4: Multi-Step Wizard:**

```typescript
@Component({
  template: `
    <div class="wizard">
      <div class="steps">
        <div *ngFor="let step of steps; let i = index" 
             [class.active]="currentStep === i"
             [class.completed]="i < currentStep">
          Step {{ i + 1 }}: {{ step.title }}
        </div>
      </div>
      
      <form [formGroup]="getCurrentStepForm()">
        <div [ngSwitch]="currentStep">
          <div *ngSwitchCase="0">
            <h2>Personal Information</h2>
            <input formControlName="firstName" placeholder="First Name">
            <input formControlName="lastName" placeholder="Last Name">
            <input formControlName="email" type="email" placeholder="Email">
          </div>
          
          <div *ngSwitchCase="1">
            <h2>Address</h2>
            <input formControlName="street" placeholder="Street">
            <input formControlName="city" placeholder="City">
            <input formControlName="zipCode" placeholder="Zip Code">
          </div>
          
          <div *ngSwitchCase="2">
            <h2>Preferences</h2>
            <label>
              <input formControlName="newsletter" type="checkbox">
              Subscribe to newsletter
            </label>
            <label>
              <input formControlName="notifications" type="checkbox">
              Enable notifications
            </label>
          </div>
          
          <div *ngSwitchCase="3">
            <h2>Review</h2>
            <pre>{{ getAllFormData() | json }}</pre>
          </div>
        </div>
      </form>
      
      <div class="navigation">
        <button (click)="previousStep()" [disabled]="currentStep === 0">
          Previous
        </button>
        <button (click)="nextStep()" [disabled]="!canProceed()">
          {{ currentStep === steps.length - 1 ? 'Submit' : 'Next' }}
        </button>
      </div>
    </div>
  `
})
export class WizardFormComponent implements OnInit {
  currentStep = 0;
  
  steps = [
    { title: 'Personal Info', form: null as FormGroup | null },
    { title: 'Address', form: null as FormGroup | null },
    { title: 'Preferences', form: null as FormGroup | null },
    { title: 'Review', form: null as FormGroup | null }
  ];
  
  constructor(private fb: FormBuilder) {}
  
  ngOnInit(): void {
    this.steps[0].form = this.fb.group({
      firstName: ['', Validators.required],
      lastName: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    });
    
    this.steps[1].form = this.fb.group({
      street: ['', Validators.required],
      city: ['', Validators.required],
      zipCode: ['', [Validators.required, Validators.pattern(/^\d{5}$/)]]
    });
    
    this.steps[2].form = this.fb.group({
      newsletter: [false],
      notifications: [true]
    });
    
    this.steps[3].form = this.fb.group({});
  }
  
  getCurrentStepForm(): FormGroup {
    return this.steps[this.currentStep].form!;
  }
  
  canProceed(): boolean {
    return this.getCurrentStepForm().valid;
  }
  
  nextStep(): void {
    if (this.canProceed()) {
      if (this.currentStep === this.steps.length - 1) {
        this.submit();
      } else {
        this.currentStep++;
      }
    }
  }
  
  previousStep(): void {
    if (this.currentStep > 0) {
      this.currentStep--;
    }
  }
  
  getAllFormData(): any {
    const data: any = {};
    this.steps.forEach(step => {
      Object.assign(data, step.form?.value);
    });
    return data;
  }
  
  submit(): void {
    console.log('Wizard submitted:', this.getAllFormData());
  }
}
```

**Follow-up Questions:**

1. **How do you handle form state when adding/removing controls dynamically?**
   - Use addControl() and removeControl()
   - Call updateValueAndValidity() after changes
   - Consider using FormArray for collections
   - Reset touched/dirty state if needed

2. **What are the performance implications of dynamic forms?**
   - Adding/removing controls triggers change detection
   - Use OnPush strategy for large forms
   - Debounce valueChanges subscriptions
   - Avoid recreating entire form on changes

3. **How do you test dynamic forms?**
   - Test form creation logic separately
   - Mock form configuration data
   - Test each step/variant independently
   - Verify validators are applied correctly

---

## What Interviewers Look For

### Strong Signals

1. **Form Approach Understanding**: Knows when to use Template-Driven vs Reactive
2. **Reactive Forms Mastery**: Comfortable with FormBuilder, FormArray
3. **Validation Expertise**: Can create custom sync and async validators
4. **Dynamic Forms**: Experienced with conditional fields and validation
5. **Real-World Patterns**: Shares specific project examples
6. **Best Practices**: Proper error handling, state management
7. **Testing Knowledge**: Can test forms effectively

## Red Flags to Avoid

1. **"Always use Template-Driven" or "Always use Reactive"**: Lacks nuance
2. **Can't explain FormArray**: Limited experience with complex forms
3. **No custom validators**: Only used built-in validators
4. **Doesn't understand async validation**: Conceptual gap
5. **Poor error handling**: Doesn't show validation messages properly
6. **No dynamic form experience**: Limited to static forms
7. **Can't test forms**: Incomplete skill set

## Key Takeaways

### Essential Concepts

1. **Form Approaches**
   - Template-Driven: Simpler, directive-based
   - Reactive: Powerful, model-driven
   - Choose based on complexity

2. **Reactive Forms Building Blocks**
   - FormControl: Single field
   - FormGroup: Group of controls
   - FormArray: Dynamic list
   - FormBuilder: Simplified creation

3. **Validation**
   - Built-in validators
   - Custom sync validators
   - Async validators for API checks
   - Cross-field validation

4. **Dynamic Forms**
   - Conditional fields
   - Dynamic validation
   - API-driven forms
   - Multi-step wizards

### Best Practices

1. Use Reactive Forms for complex scenarios
2. Leverage FormBuilder for cleaner code
3. Create reusable custom validators
4. Use updateOn: 'blur' for async validators
5. Implement proper error messages
6. Test form logic separately from template
7. Use OnPush for large forms

### Interview Success Tips

1. **Start with basics**: Explain both approaches clearly
2. **Show practical experience**: Share real project examples
3. **Discuss trade-offs**: No perfect solution
4. **Demonstrate advanced features**: FormArray, custom validators
5. **Error handling**: Always show good UX
6. **Testing**: Mention testability
7. **Ask clarifying questions**: Understand requirements

### Quick Reference

```typescript
// Template-Driven
<input name="field" [(ngModel)]="model.field" required>

// Reactive Forms
this.form = this.fb.group({
  field: ['', Validators.required]
});

// FormArray
get items(): FormArray {
  return this.form.get('items') as FormArray;
}

// Custom validator
export function myValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    return control.value === 'invalid' ? { invalid: true } : null;
  };
}

// Async validator
export function asyncValidator(service: Service): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return service.check(control.value).pipe(
      map(valid => valid ? null : { invalid: true })
    );
  };
}
```

Remember: Forms are a critical part of user interaction. Master both approaches, understand when to use each, and always prioritize user experience with clear validation and error messages.
