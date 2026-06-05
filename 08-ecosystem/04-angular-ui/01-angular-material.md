# Angular Material

## The Idea

**In plain English:** Angular Material is a ready-made set of polished website building blocks — things like buttons, menus, and pop-up forms — that you can drop into an Angular app without having to design or style them yourself. "Angular" is a framework (a structured toolkit) for building websites, and "Material" refers to Google's visual design style that makes things look clean and modern.

**Real-world analogy:** Imagine you are building a house and instead of cutting every piece of wood yourself, you go to a store that sells pre-made doors, windows, staircases, and cabinets — all designed to the same style and guaranteed to fit standard wall openings. You just pick the piece you need and install it.

- The pre-made doors and windows = Angular Material components (buttons, dialogs, tables)
- The matching style all pieces share = Material Design (Google's visual design system)
- The standard wall openings they fit into = Angular's component system that accepts these pieces

---

## Overview

Angular Material is the official Material Design component library for Angular, developed and maintained by the Angular team. It provides a comprehensive set of high-quality, reusable UI components following Google's Material Design specifications. Angular Material includes theming, accessibility features, responsive design, internationalization support, and seamless integration with Angular's reactive forms and change detection.

The library is production-ready, well-tested, and widely adopted in enterprise Angular applications. It comes with the Component Dev Kit (CDK), a collection of behavior primitives for building custom components.

## Installation and Setup

```bash
ng add @angular/material
```

The `ng add` command will:

- Install Angular Material and CDK
- Add Material icons
- Configure animations
- Offer theme selection
- Set up global styles

### Manual Installation

```bash
npm install @angular/material @angular/cdk @angular/animations
```

```typescript
// app.config.ts (Standalone)
import { ApplicationConfig } from '@angular/core';
import { provideAnimations } from '@angular/platform-browser/animations';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations()
  ]
};

// Or for modules
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';

@NgModule({
  imports: [BrowserAnimationsModule]
})
export class AppModule {}
```

### Import Material Icons

```html
<!-- index.html -->
<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
```

```scss
// styles.scss
@import '@angular/material/prebuilt-themes/indigo-pink.css';
```

## Core Concepts

### 1. Basic Components

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { MatButtonModule } from '@angular/material/button';
import { MatCardModule } from '@angular/material/card';
import { MatIconModule } from '@angular/material/icon';

@Component({
  selector: 'app-basic-example',
  standalone: true,
  imports: [
    CommonModule,
    MatButtonModule,
    MatCardModule,
    MatIconModule
  ],
  template: `
    <mat-card>
      <mat-card-header>
        <mat-card-title>Welcome</mat-card-title>
        <mat-card-subtitle>Material Design Components</mat-card-subtitle>
      </mat-card-header>
      
      <mat-card-content>
        <p>Angular Material provides beautiful, accessible components.</p>
      </mat-card-content>
      
      <mat-card-actions>
        <button mat-raised-button color="primary">
          <mat-icon>favorite</mat-icon>
          Like
        </button>
        <button mat-button>Share</button>
      </mat-card-actions>
    </mat-card>
  `
})
export class BasicExampleComponent {}
```

### 2. Form Controls

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatSelectModule } from '@angular/material/select';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatRadioModule } from '@angular/material/radio';
import { MatDatepickerModule } from '@angular/material/datepicker';
import { MatNativeDateModule } from '@angular/material/core';

@Component({
  selector: 'app-form-example',
  standalone: true,
  imports: [
    ReactiveFormsModule,
    MatFormFieldModule,
    MatInputModule,
    MatSelectModule,
    MatCheckboxModule,
    MatRadioModule,
    MatDatepickerModule,
    MatNativeDateModule
  ],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <!-- Text Input -->
      <mat-form-field appearance="fill">
        <mat-label>Name</mat-label>
        <input matInput formControlName="name" required>
        <mat-error *ngIf="form.get('name')?.hasError('required')">
          Name is required
        </mat-error>
      </mat-form-field>

      <!-- Email Input -->
      <mat-form-field appearance="outline">
        <mat-label>Email</mat-label>
        <input matInput type="email" formControlName="email">
        <mat-icon matPrefix>email</mat-icon>
        <mat-hint>Enter your email address</mat-hint>
        <mat-error *ngIf="form.get('email')?.hasError('email')">
          Please enter a valid email
        </mat-error>
      </mat-form-field>

      <!-- Select -->
      <mat-form-field>
        <mat-label>Country</mat-label>
        <mat-select formControlName="country">
          <mat-option value="us">United States</mat-option>
          <mat-option value="uk">United Kingdom</mat-option>
          <mat-option value="ca">Canada</mat-option>
        </mat-select>
      </mat-form-field>

      <!-- Checkbox -->
      <mat-checkbox formControlName="subscribe">
        Subscribe to newsletter
      </mat-checkbox>

      <!-- Radio Group -->
      <mat-radio-group formControlName="gender">
        <mat-radio-button value="male">Male</mat-radio-button>
        <mat-radio-button value="female">Female</mat-radio-button>
        <mat-radio-button value="other">Other</mat-radio-button>
      </mat-radio-group>

      <!-- Datepicker -->
      <mat-form-field>
        <mat-label>Birth Date</mat-label>
        <input matInput [matDatepicker]="picker" formControlName="birthDate">
        <mat-datepicker-toggle matSuffix [for]="picker"></mat-datepicker-toggle>
        <mat-datepicker #picker></mat-datepicker>
      </mat-form-field>

      <button mat-raised-button color="primary" type="submit">
        Submit
      </button>
    </form>
  `
})
export class FormExampleComponent {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
      country: [''],
      subscribe: [false],
      gender: [''],
      birthDate: [null]
    });
  }

  onSubmit(): void {
    if (this.form.valid) {
      console.log(this.form.value);
    }
  }
}
```

### 3. Data Tables

```typescript
import { Component, ViewChild } from '@angular/core';
import { MatTableModule, MatTableDataSource } from '@angular/material/table';
import { MatPaginatorModule, MatPaginator } from '@angular/material/paginator';
import { MatSortModule, MatSort } from '@angular/material/sort';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

@Component({
  selector: 'app-table-example',
  standalone: true,
  imports: [
    MatTableModule,
    MatPaginatorModule,
    MatSortModule,
    MatInputModule,
    MatFormFieldModule
  ],
  template: `
    <mat-form-field>
      <mat-label>Filter</mat-label>
      <input matInput (keyup)="applyFilter($event)" placeholder="Search">
    </mat-form-field>

    <table mat-table [dataSource]="dataSource" matSort>
      <!-- ID Column -->
      <ng-container matColumnDef="id">
        <th mat-header-cell *matHeaderCellDef mat-sort-header> ID </th>
        <td mat-cell *matCellDef="let user"> {{user.id}} </td>
      </ng-container>

      <!-- Name Column -->
      <ng-container matColumnDef="name">
        <th mat-header-cell *matHeaderCellDef mat-sort-header> Name </th>
        <td mat-cell *matCellDef="let user"> {{user.name}} </td>
      </ng-container>

      <!-- Email Column -->
      <ng-container matColumnDef="email">
        <th mat-header-cell *matHeaderCellDef mat-sort-header> Email </th>
        <td mat-cell *matCellDef="let user"> {{user.email}} </td>
      </ng-container>

      <!-- Role Column -->
      <ng-container matColumnDef="role">
        <th mat-header-cell *matHeaderCellDef mat-sort-header> Role </th>
        <td mat-cell *matCellDef="let user"> {{user.role}} </td>
      </ng-container>

      <!-- Actions Column -->
      <ng-container matColumnDef="actions">
        <th mat-header-cell *matHeaderCellDef> Actions </th>
        <td mat-cell *matCellDef="let user">
          <button mat-icon-button (click)="editUser(user)">
            <mat-icon>edit</mat-icon>
          </button>
          <button mat-icon-button (click)="deleteUser(user)">
            <mat-icon>delete</mat-icon>
          </button>
        </td>
      </ng-container>

      <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
      <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>
    </table>

    <mat-paginator
      [pageSizeOptions]="[5, 10, 25, 100]"
      showFirstLastButtons
    ></mat-paginator>
  `
})
export class TableExampleComponent {
  displayedColumns: string[] = ['id', 'name', 'email', 'role', 'actions'];
  dataSource: MatTableDataSource<User>;

  @ViewChild(MatPaginator) paginator!: MatPaginator;
  @ViewChild(MatSort) sort!: MatSort;

  constructor() {
    const users: User[] = [
      { id: 1, name: 'John Doe', email: 'john@example.com', role: 'Admin' },
      { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'User' },
      // ... more users
    ];

    this.dataSource = new MatTableDataSource(users);
  }

  ngAfterViewInit() {
    this.dataSource.paginator = this.paginator;
    this.dataSource.sort = this.sort;
  }

  applyFilter(event: Event) {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();

    if (this.dataSource.paginator) {
      this.dataSource.paginator.firstPage();
    }
  }

  editUser(user: User): void {
    console.log('Edit user:', user);
  }

  deleteUser(user: User): void {
    console.log('Delete user:', user);
  }
}
```

### 4. Dialogs

```typescript
import { Component, Inject } from '@angular/core';
import { MatDialogModule, MatDialog, MAT_DIALOG_DATA, MatDialogRef } from '@angular/material/dialog';
import { MatButtonModule } from '@angular/material/button';

// Dialog Component
@Component({
  selector: 'app-confirmation-dialog',
  standalone: true,
  imports: [MatDialogModule, MatButtonModule],
  template: `
    <h2 mat-dialog-title>{{ data.title }}</h2>
    <mat-dialog-content>
      <p>{{ data.message }}</p>
    </mat-dialog-content>
    <mat-dialog-actions align="end">
      <button mat-button (click)="onCancel()">Cancel</button>
      <button mat-raised-button color="warn" (click)="onConfirm()">
        {{ data.confirmText }}
      </button>
    </mat-dialog-actions>
  `
})
export class ConfirmationDialogComponent {
  constructor(
    public dialogRef: MatDialogRef<ConfirmationDialogComponent>,
    @Inject(MAT_DIALOG_DATA) public data: {
      title: string;
      message: string;
      confirmText: string;
    }
  ) {}

  onCancel(): void {
    this.dialogRef.close(false);
  }

  onConfirm(): void {
    this.dialogRef.close(true);
  }
}

// Parent Component
@Component({
  selector: 'app-dialog-example',
  standalone: true,
  imports: [MatButtonModule, MatDialogModule],
  template: `
    <button mat-raised-button (click)="openDialog()">
      Delete Item
    </button>
  `
})
export class DialogExampleComponent {
  constructor(private dialog: MatDialog) {}

  openDialog(): void {
    const dialogRef = this.dialog.open(ConfirmationDialogComponent, {
      width: '400px',
      data: {
        title: 'Confirm Delete',
        message: 'Are you sure you want to delete this item?',
        confirmText: 'Delete'
      }
    });

    dialogRef.afterClosed().subscribe(result => {
      if (result) {
        console.log('Item deleted');
      }
    });
  }
}
```

### 5. Theming System

```scss
// custom-theme.scss
@use '@angular/material' as mat;

// Include the common styles
@include mat.core();

// Define custom palettes
$my-primary: mat.define-palette(mat.$indigo-palette);
$my-accent: mat.define-palette(mat.$pink-palette, A200, A100, A400);
$my-warn: mat.define-palette(mat.$red-palette);

// Create the theme
$my-theme: mat.define-light-theme((
  color: (
    primary: $my-primary,
    accent: $my-accent,
    warn: $my-warn,
  ),
  typography: mat.define-typography-config(),
  density: 0,
));

// Apply the theme
@include mat.all-component-themes($my-theme);

// Dark theme
$my-dark-theme: mat.define-dark-theme((
  color: (
    primary: $my-primary,
    accent: $my-accent,
    warn: $my-warn,
  )
));

// Apply dark theme to .dark-theme class
.dark-theme {
  @include mat.all-component-colors($my-dark-theme);
}
```

```typescript
// Dynamic theme switching
@Component({
  selector: 'app-root',
  template: `
    <div [class.dark-theme]="isDarkTheme">
      <button mat-raised-button (click)="toggleTheme()">
        Toggle Theme
      </button>
      <router-outlet></router-outlet>
    </div>
  `
})
export class AppComponent {
  isDarkTheme = false;

  toggleTheme(): void {
    this.isDarkTheme = !this.isDarkTheme;
  }
}
```

### 6. Navigation Components

```typescript
import { Component } from '@angular/core';
import { MatSidenavModule } from '@angular/material/sidenav';
import { MatToolbarModule } from '@angular/material/toolbar';
import { MatListModule } from '@angular/material/list';
import { MatIconModule } from '@angular/material/icon';
import { MatButtonModule } from '@angular/material/button';

@Component({
  selector: 'app-navigation',
  standalone: true,
  imports: [
    MatSidenavModule,
    MatToolbarModule,
    MatListModule,
    MatIconModule,
    MatButtonModule
  ],
  template: `
    <mat-sidenav-container class="sidenav-container">
      <mat-sidenav #sidenav mode="side" opened>
        <mat-nav-list>
          <a mat-list-item routerLink="/dashboard">
            <mat-icon matListItemIcon>dashboard</mat-icon>
            <span matListItemTitle>Dashboard</span>
          </a>
          <a mat-list-item routerLink="/users">
            <mat-icon matListItemIcon>people</mat-icon>
            <span matListItemTitle>Users</span>
          </a>
          <a mat-list-item routerLink="/settings">
            <mat-icon matListItemIcon>settings</mat-icon>
            <span matListItemTitle>Settings</span>
          </a>
        </mat-nav-list>
      </mat-sidenav>

      <mat-sidenav-content>
        <mat-toolbar color="primary">
          <button mat-icon-button (click)="sidenav.toggle()">
            <mat-icon>menu</mat-icon>
          </button>
          <span>My Application</span>
          <span class="spacer"></span>
          <button mat-icon-button>
            <mat-icon>notifications</mat-icon>
          </button>
          <button mat-icon-button>
            <mat-icon>account_circle</mat-icon>
          </button>
        </mat-toolbar>

        <div class="content">
          <router-outlet></router-outlet>
        </div>
      </mat-sidenav-content>
    </mat-sidenav-container>
  `,
  styles: [`
    .sidenav-container {
      height: 100vh;
    }
    .spacer {
      flex: 1 1 auto;
    }
    .content {
      padding: 20px;
    }
  `]
})
export class NavigationComponent {}
```

### 7. Snackbar Notifications

```typescript
import { Component } from '@angular/core';
import { MatSnackBar, MatSnackBarModule } from '@angular/material/snack-bar';
import { MatButtonModule } from '@angular/material/button';

@Component({
  selector: 'app-snackbar-example',
  standalone: true,
  imports: [MatSnackBarModule, MatButtonModule],
  template: `
    <button mat-raised-button (click)="showSuccess()">
      Success Message
    </button>
    <button mat-raised-button (click)="showError()">
      Error Message
    </button>
    <button mat-raised-button (click)="showWithAction()">
      Message with Action
    </button>
  `
})
export class SnackbarExampleComponent {
  constructor(private snackBar: MatSnackBar) {}

  showSuccess(): void {
    this.snackBar.open('Operation successful!', 'Close', {
      duration: 3000,
      horizontalPosition: 'end',
      verticalPosition: 'top',
      panelClass: ['success-snackbar']
    });
  }

  showError(): void {
    this.snackBar.open('Error occurred!', 'Close', {
      duration: 5000,
      panelClass: ['error-snackbar']
    });
  }

  showWithAction(): void {
    const snackBarRef = this.snackBar.open(
      'Item deleted',
      'Undo',
      { duration: 5000 }
    );

    snackBarRef.onAction().subscribe(() => {
      console.log('Undo action triggered');
    });
  }
}
```

### 8. CDK Utilities

```typescript
import { Component } from '@angular/core';
import { CdkDragDrop, DragDropModule, moveItemInArray } from '@angular/cdk/drag-drop';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-drag-drop-example',
  standalone: true,
  imports: [CommonModule, DragDropModule],
  template: `
    <div cdkDropList (cdkDropListDropped)="drop($event)">
      <div
        *ngFor="let item of items"
        cdkDrag
        class="drag-item"
      >
        <div class="drag-handle" cdkDragHandle>
          <mat-icon>drag_indicator</mat-icon>
        </div>
        {{ item }}
      </div>
    </div>
  `,
  styles: [`
    .drag-item {
      padding: 16px;
      border: 1px solid #ccc;
      margin-bottom: 8px;
      background: white;
      cursor: move;
      display: flex;
      align-items: center;
    }
    .drag-handle {
      margin-right: 8px;
      color: #999;
    }
    .cdk-drag-preview {
      box-shadow: 0 5px 10px rgba(0,0,0,0.3);
    }
    .cdk-drag-animating {
      transition: transform 250ms cubic-bezier(0, 0, 0.2, 1);
    }
  `]
})
export class DragDropExampleComponent {
  items = ['Item 1', 'Item 2', 'Item 3', 'Item 4', 'Item 5'];

  drop(event: CdkDragDrop<string[]>): void {
    moveItemInArray(this.items, event.previousIndex, event.currentIndex);
  }
}
```

### 9. Autocomplete

```typescript
import { Component, OnInit } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { MatAutocompleteModule } from '@angular/material/autocomplete';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';
import { Observable } from 'rxjs';
import { map, startWith } from 'rxjs/operators';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-autocomplete-example',
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    MatAutocompleteModule,
    MatInputModule,
    MatFormFieldModule
  ],
  template: `
    <mat-form-field>
      <mat-label>Search</mat-label>
      <input
        type="text"
        matInput
        [formControl]="searchControl"
        [matAutocomplete]="auto"
      >
      <mat-autocomplete #auto="matAutocomplete">
        <mat-option
          *ngFor="let option of filteredOptions$ | async"
          [value]="option"
        >
          {{ option }}
        </mat-option>
      </mat-autocomplete>
    </mat-form-field>
  `
})
export class AutocompleteExampleComponent implements OnInit {
  searchControl = new FormControl('');
  options: string[] = ['Apple', 'Banana', 'Cherry', 'Date', 'Elderberry'];
  filteredOptions$!: Observable<string[]>;

  ngOnInit(): void {
    this.filteredOptions$ = this.searchControl.valueChanges.pipe(
      startWith(''),
      map(value => this._filter(value || ''))
    );
  }

  private _filter(value: string): string[] {
    const filterValue = value.toLowerCase();
    return this.options.filter(option =>
      option.toLowerCase().includes(filterValue)
    );
  }
}
```

### 10. Stepper

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { MatStepperModule } from '@angular/material/stepper';
import { MatInputModule } from '@angular/material/input';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatButtonModule } from '@angular/material/button';

@Component({
  selector: 'app-stepper-example',
  standalone: true,
  imports: [
    ReactiveFormsModule,
    MatStepperModule,
    MatFormFieldModule,
    MatInputModule,
    MatButtonModule
  ],
  template: `
    <mat-stepper [linear]="true" #stepper>
      <mat-step [stepControl]="personalInfoForm">
        <form [formGroup]="personalInfoForm">
          <ng-template matStepLabel>Personal Information</ng-template>
          
          <mat-form-field>
            <mat-label>Name</mat-label>
            <input matInput formControlName="name" required>
          </mat-form-field>

          <mat-form-field>
            <mat-label>Email</mat-label>
            <input matInput formControlName="email" type="email" required>
          </mat-form-field>

          <div>
            <button mat-button matStepperNext>Next</button>
          </div>
        </form>
      </mat-step>

      <mat-step [stepControl]="addressForm">
        <form [formGroup]="addressForm">
          <ng-template matStepLabel>Address</ng-template>
          
          <mat-form-field>
            <mat-label>Street</mat-label>
            <input matInput formControlName="street" required>
          </mat-form-field>

          <mat-form-field>
            <mat-label>City</mat-label>
            <input matInput formControlName="city" required>
          </mat-form-field>

          <div>
            <button mat-button matStepperPrevious>Back</button>
            <button mat-button matStepperNext>Next</button>
          </div>
        </form>
      </mat-step>

      <mat-step>
        <ng-template matStepLabel>Done</ng-template>
        <p>You are now done.</p>
        <div>
          <button mat-button matStepperPrevious>Back</button>
          <button mat-button (click)="stepper.reset()">Reset</button>
        </div>
      </mat-step>
    </mat-stepper>
  `
})
export class StepperExampleComponent {
  personalInfoForm: FormGroup;
  addressForm: FormGroup;

  constructor(private fb: FormBuilder) {
    this.personalInfoForm = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]]
    });

    this.addressForm = this.fb.group({
      street: ['', Validators.required],
      city: ['', Validators.required]
    });
  }
}
```

## Common Mistakes

1. **Not importing BrowserAnimationsModule**: Material components require animations.

2. **Forgetting to import individual modules**: Each component needs its own module import.

3. **Improper theme configuration**: Must use Sass/SCSS and include theme properly.

4. **Not using mat-form-field**: Input components must be wrapped in mat-form-field.

5. **Ignoring accessibility**: Material is accessible by default, don't remove ARIA attributes.

## Best Practices

1. **Use standalone components**: Import only needed Material modules
2. **Leverage CDK**: Use CDK primitives for custom component behavior
3. **Custom themes**: Create branded themes matching your design system
4. **Typography configuration**: Configure typography for consistent text styles
5. **Responsive design**: Use Material's breakpoint observer for responsive layouts
6. **Accessibility**: Test with screen readers and keyboard navigation
7. **Performance**: Tree-shake unused components
8. **Consistent spacing**: Use Material's spacing system

## When to Use Angular Material

**Use Angular Material when:**

- Building Angular applications
- Want Material Design aesthetics
- Need enterprise-ready components
- Require accessibility out of the box
- Want official Angular support
- Building admin dashboards or internal tools

**Consider alternatives when:**

- Need completely custom design system
- Want lighter bundle size
- Prefer other design languages (Bootstrap, Ant Design)
- Mobile-first applications (consider Ionic)

## Interview Questions

1. **Q: What is the CDK and why is it important?**
   A: The Component Dev Kit (CDK) provides behavior primitives (drag-drop, overlay, a11y, etc.) for building custom components without opinionated styles. It's the foundation Material components are built on.

2. **Q: How does Material theming work?**
   A: Material uses Sass mixins to generate CSS from theme definitions. You define color palettes (primary, accent, warn), typography, and density, then apply them using `@include mat.all-component-themes()`.

3. **Q: What's the difference between mat-button variants?**
   A: `mat-button` is flat, `mat-raised-button` has elevation, `mat-flat-button` has background without elevation, `mat-stroked-button` has border, `mat-icon-button` is circular for icons, `mat-fab` is floating action button.

4. **Q: How do you customize Material component styles?**
   A: Use Angular's view encapsulation strategies, global styles with `::ng-deep` (deprecated), or theme mixins. Best practice: create custom themes or use CSS variables.

5. **Q: What are Material table features?**
   A: Sorting, pagination, filtering, row selection, sticky headers/columns, column resizing, inline editing, expandable rows. Powered by `MatTableDataSource`.

6. **Q: How do dialogs communicate with parent components?**
   A: Data is passed via MAT_DIALOG_DATA injection token, and returned through `dialogRef.close(result)`. Parent receives result from `afterClosed()` observable.

## Key Takeaways

1. Angular Material is the official Material Design library for Angular
2. CDK provides behavior primitives for custom components
3. Theming uses Sass mixins with customizable palettes
4. All components follow accessibility and internationalization standards
5. Forms integrate seamlessly with Angular's reactive forms
6. MatTableDataSource provides powerful data table features
7. Dialogs use injection tokens for data passing
8. Navigation components support responsive layouts
9. Standalone imports enable tree-shaking for smaller bundles
10. Material is ideal for enterprise Angular applications

## Resources

- [Angular Material Documentation](https://material.angular.io/)
- [CDK Documentation](https://material.angular.io/cdk/categories)
- [Material Design Guidelines](https://m3.material.io/)
- [Theming Guide](https://material.angular.io/guide/theming)
- [Component Dev Kit](https://material.angular.io/cdk/categories)
- [Accessibility Guide](https://material.angular.io/guide/accessibility)
