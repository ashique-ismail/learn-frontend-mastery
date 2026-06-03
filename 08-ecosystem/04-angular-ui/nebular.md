# Nebular - Customizable Angular UI Library

## Overview

Nebular is a customizable Angular UI Library with a focus on beautiful design and a comprehensive set of Angular components, themes, and modules. Developed by Akveo, Nebular provides not just UI components, but also includes authentication (@nebular/auth), security (@nebular/security), and theming modules, making it a complete solution for building modern Angular applications with consistent, customizable design.

## Installation

### Prerequisites

```bash
# Required dependencies
npm install @angular/core @angular/common @angular/animations
```

### Installation Steps

```bash
# Install Nebular theme and UI components
npm install @nebular/theme @angular/cdk

# Install Eva Icons (required for Nebular)
npm install @eva-design/eva

# Optional: Authentication module
npm install @nebular/auth

# Optional: Security module
npm install @nebular/security
```

### Setup

**angular.json:**
```json
{
  "projects": {
    "your-app": {
      "architect": {
        "build": {
          "options": {
            "styles": [
              "node_modules/@nebular/theme/styles/prebuilt/default.css",
              "src/styles.scss"
            ]
          }
        }
      }
    }
  }
}
```

**app.module.ts:**
```typescript
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
import { 
  NbThemeModule, 
  NbLayoutModule,
  NbButtonModule,
  NbCardModule,
  NbMenuModule,
  NbSidebarModule
} from '@nebular/theme';
import { NbEvaIconsModule } from '@nebular/eva-icons';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    NbThemeModule.forRoot({ name: 'default' }),
    NbLayoutModule,
    NbEvaIconsModule,
    NbButtonModule,
    NbCardModule,
    NbMenuModule.forRoot(),
    NbSidebarModule.forRoot()
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

**styles.scss:**
```scss
@import '@nebular/theme/styles/globals';

@include nb-install() {
  @include nb-theme-global();
}
```

## Core Concepts

### 1. Theme System

Nebular's powerful theming system allows complete customization of colors, typography, and component styles.

**Built-in Themes:**
- `default`: Light theme with blue accents
- `dark`: Dark theme
- `cosmic`: Dark theme with purple accents
- `corporate`: Professional corporate theme

### 2. Layout System

Nebular provides a flexible layout system with header, footer, sidebar, and content areas.

### 3. Component Architecture

Components are highly customizable with:
- Status colors (primary, success, info, warning, danger)
- Sizes (tiny, small, medium, large, giant)
- Appearances (filled, outline, ghost, hero)

## Component Examples

### 1. Layout and Navigation

**Complete Application Layout:**
```typescript
import { Component } from '@angular/core';
import { NbMenuItem } from '@nebular/theme';

@Component({
  selector: 'app-root',
  template: `
    <nb-layout>
      <!-- Header -->
      <nb-layout-header fixed>
        <div class="header-container">
          <button nbButton ghost (click)="toggleSidebar()">
            <nb-icon icon="menu-outline"></nb-icon>
          </button>
          
          <div class="logo">
            <h4>My Application</h4>
          </div>

          <div class="header-actions">
            <!-- Search -->
            <nb-search 
              type="rotate-layout"
              placeholder="Search...">
            </nb-search>

            <!-- Notifications -->
            <nb-actions size="small">
              <nb-action 
                icon="bell-outline"
                [badgeText]="'5'"
                badgeStatus="danger"
                (click)="showNotifications()">
              </nb-action>

              <nb-action>
                <nb-user 
                  [nbContextMenu]="userMenu"
                  [name]="user.name"
                  [title]="user.role"
                  [picture]="user.avatar">
                </nb-user>
              </nb-action>
            </nb-actions>
          </div>
        </div>
      </nb-layout-header>

      <!-- Sidebar -->
      <nb-sidebar class="menu-sidebar" tag="menu-sidebar" responsive>
        <div class="sidebar-header">
          <nb-icon icon="grid-outline"></nb-icon>
          <span>Menu</span>
        </div>
        <nb-menu [items]="menuItems" tag="main-menu"></nb-menu>
      </nb-sidebar>

      <!-- Content -->
      <nb-layout-column>
        <router-outlet></router-outlet>
      </nb-layout-column>

      <!-- Footer -->
      <nb-layout-footer fixed>
        <span>&copy; 2026 My Company. All rights reserved.</span>
      </nb-layout-footer>
    </nb-layout>
  `,
  styles: [`
    .header-container {
      display: flex;
      align-items: center;
      width: 100%;
    }

    .logo {
      margin-left: 1rem;
    }

    .logo h4 {
      margin: 0;
    }

    .header-actions {
      margin-left: auto;
      display: flex;
      align-items: center;
      gap: 1rem;
    }

    .sidebar-header {
      padding: 1.25rem;
      display: flex;
      align-items: center;
      gap: 0.5rem;
      font-weight: 600;
    }
  `]
})
export class AppComponent {
  user = {
    name: 'John Doe',
    role: 'Administrator',
    avatar: 'assets/avatar.jpg'
  };

  menuItems: NbMenuItem[] = [
    {
      title: 'Dashboard',
      icon: 'home-outline',
      link: '/dashboard',
      home: true
    },
    {
      title: 'Users',
      icon: 'people-outline',
      children: [
        {
          title: 'User List',
          link: '/users/list'
        },
        {
          title: 'Roles & Permissions',
          link: '/users/roles'
        },
        {
          title: 'Add User',
          link: '/users/add'
        }
      ]
    },
    {
      title: 'Projects',
      icon: 'folder-outline',
      link: '/projects',
      badge: {
        text: '12',
        status: 'primary'
      }
    },
    {
      title: 'Reports',
      icon: 'bar-chart-outline',
      children: [
        {
          title: 'Analytics',
          link: '/reports/analytics'
        },
        {
          title: 'Performance',
          link: '/reports/performance'
        }
      ]
    },
    {
      title: 'Settings',
      icon: 'settings-outline',
      link: '/settings'
    }
  ];

  userMenu: NbMenuItem[] = [
    { title: 'Profile', icon: 'person-outline' },
    { title: 'Settings', icon: 'settings-outline' },
    { title: 'Logout', icon: 'log-out-outline' }
  ];

  constructor(private sidebarService: NbSidebarService) {}

  toggleSidebar() {
    this.sidebarService.toggle(false, 'menu-sidebar');
  }

  showNotifications() {
    console.log('Show notifications');
  }
}
```

### 2. Cards and Data Display

**Dashboard with Various Card Types:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-dashboard',
  template: `
    <div class="row">
      <!-- Status Cards -->
      <div class="col-lg-3 col-md-6">
        <nb-card status="primary" accent="primary">
          <nb-card-body>
            <div class="icon-container">
              <nb-icon icon="people-outline" status="primary"></nb-icon>
            </div>
            <div class="details">
              <div class="value">12,453</div>
              <div class="title">Total Users</div>
            </div>
          </nb-card-body>
        </nb-card>
      </div>

      <div class="col-lg-3 col-md-6">
        <nb-card status="success" accent="success">
          <nb-card-body>
            <div class="icon-container">
              <nb-icon icon="trending-up-outline" status="success"></nb-icon>
            </div>
            <div class="details">
              <div class="value">$45,678</div>
              <div class="title">Revenue</div>
            </div>
          </nb-card-body>
        </nb-card>
      </div>

      <div class="col-lg-3 col-md-6">
        <nb-card status="info" accent="info">
          <nb-card-body>
            <div class="icon-container">
              <nb-icon icon="shopping-cart-outline" status="info"></nb-icon>
            </div>
            <div class="details">
              <div class="value">1,847</div>
              <div class="title">Orders</div>
            </div>
          </nb-card-body>
        </nb-card>
      </div>

      <div class="col-lg-3 col-md-6">
        <nb-card status="warning" accent="warning">
          <nb-card-body>
            <div class="icon-container">
              <nb-icon icon="alert-triangle-outline" status="warning"></nb-icon>
            </div>
            <div class="details">
              <div class="value">23</div>
              <div class="title">Pending Issues</div>
            </div>
          </nb-card-body>
        </nb-card>
      </div>

      <!-- Chart Card -->
      <div class="col-lg-8">
        <nb-card>
          <nb-card-header>
            <div class="card-header-content">
              <span>Sales Overview</span>
              <nb-select size="small" [(selected)]="selectedPeriod">
                <nb-option value="week">Last Week</nb-option>
                <nb-option value="month">Last Month</nb-option>
                <nb-option value="year">Last Year</nb-option>
              </nb-select>
            </div>
          </nb-card-header>
          <nb-card-body>
            <!-- Chart component -->
            <div class="chart-container">
              Chart goes here
            </div>
          </nb-card-body>
        </nb-card>
      </div>

      <!-- Activity Feed Card -->
      <div class="col-lg-4">
        <nb-card>
          <nb-card-header>Recent Activities</nb-card-header>
          <nb-list>
            <nb-list-item *ngFor="let activity of activities">
              <nb-user 
                [name]="activity.user"
                [title]="activity.action"
                size="medium">
              </nb-user>
              <div class="timestamp">{{activity.time}}</div>
            </nb-list-item>
          </nb-list>
        </nb-card>
      </div>

      <!-- Interactive Card with Tabs -->
      <div class="col-lg-6">
        <nb-card>
          <nb-card-header>
            <nb-tabset fullWidth>
              <nb-tab tabTitle="Overview">
                <div class="tab-content">
                  <p>Overview content</p>
                </div>
              </nb-tab>
              <nb-tab tabTitle="Details">
                <div class="tab-content">
                  <p>Details content</p>
                </div>
              </nb-tab>
              <nb-tab tabTitle="Settings">
                <div class="tab-content">
                  <p>Settings content</p>
                </div>
              </nb-tab>
            </nb-tabset>
          </nb-card-header>
        </nb-card>
      </div>

      <!-- Flip Card -->
      <div class="col-lg-6">
        <nb-flip-card [flipped]="flipped" [showToggleButton]="true">
          <nb-card-front>
            <nb-card accent="info">
              <nb-card-header>Front Card</nb-card-header>
              <nb-card-body>
                <p>This is the front side of the flip card.</p>
                <button nbButton status="info" (click)="toggleFlip()">
                  Flip to Back
                </button>
              </nb-card-body>
            </nb-card>
          </nb-card-front>
          <nb-card-back>
            <nb-card accent="success">
              <nb-card-header>Back Card</nb-card-header>
              <nb-card-body>
                <p>This is the back side of the flip card.</p>
                <button nbButton status="success" (click)="toggleFlip()">
                  Flip to Front
                </button>
              </nb-card-body>
            </nb-card>
          </nb-card-back>
        </nb-flip-card>
      </div>
    </div>
  `,
  styles: [`
    .icon-container {
      font-size: 3rem;
      margin-bottom: 1rem;
    }

    .details .value {
      font-size: 2rem;
      font-weight: bold;
      margin-bottom: 0.5rem;
    }

    .details .title {
      color: nb-theme(text-hint-color);
      text-transform: uppercase;
      font-size: 0.875rem;
    }

    .card-header-content {
      display: flex;
      justify-content: space-between;
      align-items: center;
      width: 100%;
    }

    .chart-container {
      height: 300px;
      display: flex;
      align-items: center;
      justify-content: center;
    }

    .timestamp {
      font-size: 0.75rem;
      color: nb-theme(text-hint-color);
    }

    .tab-content {
      padding: 1rem;
    }
  `]
})
export class DashboardComponent {
  selectedPeriod = 'month';
  flipped = false;

  activities = [
    { user: 'John Doe', action: 'Created new project', time: '2m ago' },
    { user: 'Jane Smith', action: 'Updated profile', time: '15m ago' },
    { user: 'Bob Johnson', action: 'Completed task', time: '1h ago' }
  ];

  toggleFlip() {
    this.flipped = !this.flipped;
  }
}
```

### 3. Forms and Inputs

**Advanced Form with Nebular Components:**
```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-user-form',
  template: `
    <nb-card>
      <nb-card-header>
        <h5>User Registration</h5>
      </nb-card-header>
      <nb-card-body>
        <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
          <!-- Text Input -->
          <div class="form-group">
            <label class="label">Full Name</label>
            <input 
              type="text"
              nbInput
              fullWidth
              formControlName="fullName"
              placeholder="Enter your full name"
              [status]="getFieldStatus('fullName')">
            <div class="caption status-danger" 
                 *ngIf="userForm.get('fullName')?.invalid && 
                        userForm.get('fullName')?.touched">
              Full name is required (min 3 characters)
            </div>
          </div>

          <!-- Email Input with Icon -->
          <div class="form-group">
            <label class="label">Email</label>
            <nb-form-field>
              <input 
                type="email"
                nbInput
                fullWidth
                formControlName="email"
                placeholder="Enter your email"
                [status]="getFieldStatus('email')">
              <button nbSuffix nbButton ghost>
                <nb-icon icon="email-outline"></nb-icon>
              </button>
            </nb-form-field>
            <div class="caption status-danger" 
                 *ngIf="userForm.get('email')?.invalid && 
                        userForm.get('email')?.touched">
              Valid email is required
            </div>
          </div>

          <!-- Password Input -->
          <div class="form-group">
            <label class="label">Password</label>
            <nb-form-field>
              <input 
                [type]="showPassword ? 'text' : 'password'"
                nbInput
                fullWidth
                formControlName="password"
                placeholder="Enter password"
                [status]="getFieldStatus('password')">
              <button 
                nbSuffix 
                nbButton 
                ghost
                type="button"
                (click)="togglePassword()">
                <nb-icon 
                  [icon]="showPassword ? 'eye-off-outline' : 'eye-outline'">
                </nb-icon>
              </button>
            </nb-form-field>
            <div class="caption" [class.status-danger]="passwordStrength < 3">
              Password strength: {{getPasswordStrengthText()}}
            </div>
          </div>

          <!-- Select -->
          <div class="form-group">
            <label class="label">Role</label>
            <nb-select 
              fullWidth
              formControlName="role"
              placeholder="Select role"
              [status]="getFieldStatus('role')">
              <nb-option value="user">User</nb-option>
              <nb-option value="admin">Admin</nb-option>
              <nb-option value="manager">Manager</nb-option>
              <nb-option value="guest">Guest</nb-option>
            </nb-select>
          </div>

          <!-- Radio Buttons -->
          <div class="form-group">
            <label class="label">Account Type</label>
            <nb-radio-group formControlName="accountType">
              <nb-radio value="personal">Personal</nb-radio>
              <nb-radio value="business">Business</nb-radio>
            </nb-radio-group>
          </div>

          <!-- Checkbox -->
          <div class="form-group">
            <nb-checkbox formControlName="newsletter">
              Subscribe to newsletter
            </nb-checkbox>
          </div>

          <!-- Toggle -->
          <div class="form-group">
            <nb-toggle 
              formControlName="notifications"
              labelPosition="left">
              Enable notifications
            </nb-toggle>
          </div>

          <!-- Date Picker -->
          <div class="form-group">
            <label class="label">Birth Date</label>
            <input 
              nbInput
              fullWidth
              placeholder="Select date"
              formControlName="birthDate"
              [nbDatepicker]="datepicker"
              [status]="getFieldStatus('birthDate')">
            <nb-datepicker #datepicker></nb-datepicker>
          </div>

          <!-- Date Range Picker -->
          <div class="form-group">
            <label class="label">Active Period</label>
            <input 
              nbInput
              fullWidth
              placeholder="Select date range"
              formControlName="activePeriod"
              [nbDatepicker]="rangepicker">
            <nb-rangepicker #rangepicker></nb-rangepicker>
          </div>

          <!-- Autocomplete -->
          <div class="form-group">
            <label class="label">Country</label>
            <input 
              type="text"
              nbInput
              fullWidth
              placeholder="Type to search..."
              formControlName="country"
              [nbAutocomplete]="auto">
            <nb-autocomplete #auto>
              <nb-option 
                *ngFor="let country of filteredCountries$ | async" 
                [value]="country">
                {{country}}
              </nb-option>
            </nb-autocomplete>
          </div>

          <!-- Tag Input -->
          <div class="form-group">
            <label class="label">Skills</label>
            <nb-tag-list>
              <nb-tag 
                *ngFor="let skill of skills"
                [text]="skill"
                removable
                (remove)="removeSkill(skill)">
              </nb-tag>
              <input 
                type="text"
                nbTagInput
                placeholder="Add skill..."
                (tagAdd)="addSkill($event)"
                fullWidth>
            </nb-tag-list>
          </div>

          <!-- Slider -->
          <div class="form-group">
            <label class="label">
              Experience Level: {{userForm.get('experience')?.value}} years
            </label>
            <input 
              type="range"
              formControlName="experience"
              min="0"
              max="20"
              step="1"
              nbInput>
          </div>

          <!-- Buttons -->
          <div class="form-group">
            <button 
              nbButton
              status="primary"
              type="submit"
              [disabled]="!userForm.valid || submitting"
              fullWidth>
              <nb-icon icon="checkmark-outline" *ngIf="!submitting"></nb-icon>
              {{submitting ? 'Creating Account...' : 'Create Account'}}
            </button>
          </div>

          <div class="form-group">
            <button 
              nbButton
              status="basic"
              type="button"
              (click)="onReset()"
              fullWidth>
              Reset Form
            </button>
          </div>
        </form>
      </nb-card-body>
    </nb-card>
  `,
  styles: [`
    .form-group {
      margin-bottom: 1.5rem;
    }

    .label {
      display: block;
      margin-bottom: 0.5rem;
      font-weight: 600;
    }

    .caption {
      margin-top: 0.25rem;
      font-size: 0.875rem;
    }

    .status-danger {
      color: nb-theme(color-danger-500);
    }
  `]
})
export class UserFormComponent implements OnInit {
  userForm!: FormGroup;
  submitting = false;
  showPassword = false;
  passwordStrength = 0;
  skills: string[] = [];
  
  countries = ['USA', 'UK', 'Canada', 'Australia', 'Germany', 'France'];
  filteredCountries$!: Observable<string[]>;

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.userForm = this.fb.group({
      fullName: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(8)]],
      role: ['', Validators.required],
      accountType: ['personal'],
      newsletter: [false],
      notifications: [true],
      birthDate: [''],
      activePeriod: [''],
      country: [''],
      experience: [5]
    });

    // Watch password changes for strength indicator
    this.userForm.get('password')?.valueChanges.subscribe(password => {
      this.passwordStrength = this.calculatePasswordStrength(password);
    });

    // Setup autocomplete filtering
    this.filteredCountries$ = this.userForm.get('country')!.valueChanges.pipe(
      startWith(''),
      map(value => this.filterCountries(value))
    );
  }

  getFieldStatus(fieldName: string): string {
    const field = this.userForm.get(fieldName);
    if (!field) return 'basic';
    
    if (field.invalid && field.touched) return 'danger';
    if (field.valid && field.touched) return 'success';
    return 'basic';
  }

  togglePassword() {
    this.showPassword = !this.showPassword;
  }

  calculatePasswordStrength(password: string): number {
    let strength = 0;
    if (password.length >= 8) strength++;
    if (/[a-z]/.test(password) && /[A-Z]/.test(password)) strength++;
    if (/\d/.test(password)) strength++;
    if (/[^a-zA-Z\d]/.test(password)) strength++;
    return strength;
  }

  getPasswordStrengthText(): string {
    const labels = ['Weak', 'Fair', 'Good', 'Strong'];
    return labels[this.passwordStrength - 1] || 'Weak';
  }

  filterCountries(value: string): string[] {
    return this.countries.filter(country =>
      country.toLowerCase().includes(value.toLowerCase())
    );
  }

  addSkill(event: { value: string }) {
    if (event.value && !this.skills.includes(event.value)) {
      this.skills.push(event.value);
    }
  }

  removeSkill(skill: string) {
    this.skills = this.skills.filter(s => s !== skill);
  }

  onSubmit() {
    if (this.userForm.valid) {
      this.submitting = true;
      const formData = {
        ...this.userForm.value,
        skills: this.skills
      };
      console.log('Form submitted:', formData);
      
      setTimeout(() => {
        this.submitting = false;
        // Reset or navigate
      }, 2000);
    }
  }

  onReset() {
    this.userForm.reset({
      accountType: 'personal',
      notifications: true,
      experience: 5
    });
    this.skills = [];
  }
}
```

### 4. Dialogs and Modals

**Dialog Service Usage:**
```typescript
import { Component } from '@angular/core';
import { NbDialogService, NbDialogRef } from '@nebular/theme';

@Component({
  selector: 'app-dialog-demo',
  template: `
    <nb-card>
      <nb-card-header>Dialog Examples</nb-card-header>
      <nb-card-body>
        <div class="button-group">
          <button nbButton status="primary" (click)="openDialog()">
            Open Dialog
          </button>
          <button nbButton status="info" (click)="openDialogWithData()">
            Dialog with Data
          </button>
          <button nbButton status="success" (click)="openCustomDialog()">
            Custom Dialog
          </button>
          <button nbButton status="warning" (click)="openConfirmDialog()">
            Confirm Dialog
          </button>
        </div>
      </nb-card-body>
    </nb-card>
  `,
  styles: [`
    .button-group {
      display: flex;
      gap: 1rem;
      flex-wrap: wrap;
    }
  `]
})
export class DialogDemoComponent {
  constructor(private dialogService: NbDialogService) {}

  openDialog() {
    this.dialogService.open(SimpleDialogComponent);
  }

  openDialogWithData() {
    this.dialogService.open(DataDialogComponent, {
      context: {
        title: 'User Details',
        data: { name: 'John Doe', email: 'john@example.com' }
      }
    });
  }

  openCustomDialog() {
    this.dialogService.open(CustomDialogComponent, {
      closeOnBackdropClick: false,
      hasBackdrop: true,
      hasScroll: true
    });
  }

  openConfirmDialog() {
    this.dialogService.open(ConfirmDialogComponent, {
      context: {
        message: 'Are you sure you want to delete this item?'
      }
    }).onClose.subscribe(confirmed => {
      if (confirmed) {
        console.log('Action confirmed');
      }
    });
  }
}

// Simple Dialog Component
@Component({
  selector: 'app-simple-dialog',
  template: `
    <nb-card>
      <nb-card-header>
        Simple Dialog
        <button nbButton ghost (click)="dismiss()">
          <nb-icon icon="close-outline"></nb-icon>
        </button>
      </nb-card-header>
      <nb-card-body>
        <p>This is a simple dialog with basic content.</p>
      </nb-card-body>
      <nb-card-footer>
        <button nbButton status="primary" (click)="dismiss()">
          Close
        </button>
      </nb-card-footer>
    </nb-card>
  `
})
export class SimpleDialogComponent {
  constructor(protected ref: NbDialogRef<SimpleDialogComponent>) {}

  dismiss() {
    this.ref.close();
  }
}

// Dialog with Data
@Component({
  selector: 'app-data-dialog',
  template: `
    <nb-card>
      <nb-card-header>{{title}}</nb-card-header>
      <nb-card-body>
        <nb-list>
          <nb-list-item>
            <strong>Name:</strong> {{data.name}}
          </nb-list-item>
          <nb-list-item>
            <strong>Email:</strong> {{data.email}}
          </nb-list-item>
        </nb-list>
      </nb-card-body>
      <nb-card-footer>
        <button nbButton (click)="ref.close()">Close</button>
      </nb-card-footer>
    </nb-card>
  `
})
export class DataDialogComponent {
  title: string = '';
  data: any = {};

  constructor(protected ref: NbDialogRef<DataDialogComponent>) {}
}

// Confirmation Dialog
@Component({
  selector: 'app-confirm-dialog',
  template: `
    <nb-card accent="warning">
      <nb-card-header>
        Confirm Action
      </nb-card-header>
      <nb-card-body>
        <p>{{message}}</p>
      </nb-card-body>
      <nb-card-footer>
        <div class="button-group">
          <button nbButton status="basic" (click)="ref.close(false)">
            Cancel
          </button>
          <button nbButton status="danger" (click)="ref.close(true)">
            Confirm
          </button>
        </div>
      </nb-card-footer>
    </nb-card>
  `,
  styles: [`
    .button-group {
      display: flex;
      gap: 0.5rem;
      justify-content: flex-end;
    }
  `]
})
export class ConfirmDialogComponent {
  message: string = '';

  constructor(protected ref: NbDialogRef<ConfirmDialogComponent>) {}
}

// Custom Dialog with Form
@Component({
  selector: 'app-custom-dialog',
  template: `
    <nb-card>
      <nb-card-header>Create Task</nb-card-header>
      <nb-card-body>
        <form [formGroup]="taskForm">
          <div class="form-group">
            <label class="label">Task Title</label>
            <input nbInput fullWidth formControlName="title">
          </div>
          <div class="form-group">
            <label class="label">Description</label>
            <textarea nbInput fullWidth formControlName="description"></textarea>
          </div>
          <div class="form-group">
            <label class="label">Priority</label>
            <nb-select fullWidth formControlName="priority">
              <nb-option value="low">Low</nb-option>
              <nb-option value="medium">Medium</nb-option>
              <nb-option value="high">High</nb-option>
            </nb-select>
          </div>
        </form>
      </nb-card-body>
      <nb-card-footer>
        <div class="button-group">
          <button nbButton status="basic" (click)="ref.close()">
            Cancel
          </button>
          <button 
            nbButton 
            status="primary" 
            [disabled]="!taskForm.valid"
            (click)="submit()">
            Create
          </button>
        </div>
      </nb-card-footer>
    </nb-card>
  `,
  styles: [`
    .form-group {
      margin-bottom: 1rem;
    }

    .label {
      display: block;
      margin-bottom: 0.5rem;
    }

    .button-group {
      display: flex;
      gap: 0.5rem;
      justify-content: flex-end;
    }
  `]
})
export class CustomDialogComponent {
  taskForm = this.fb.group({
    title: ['', Validators.required],
    description: [''],
    priority: ['medium']
  });

  constructor(
    protected ref: NbDialogRef<CustomDialogComponent>,
    private fb: FormBuilder
  ) {}

  submit() {
    if (this.taskForm.valid) {
      this.ref.close(this.taskForm.value);
    }
  }
}
```

### 5. Notifications and Toasts

**Toast Service Usage:**
```typescript
import { Component } from '@angular/core';
import { NbToastrService, NbGlobalPosition, NbGlobalPhysicalPosition } from '@nebular/theme';

@Component({
  selector: 'app-toast-demo',
  template: `
    <nb-card>
      <nb-card-header>Toast Notifications</nb-card-header>
      <nb-card-body>
        <div class="button-grid">
          <button nbButton status="primary" (click)="showInfo()">
            Info Toast
          </button>
          <button nbButton status="success" (click)="showSuccess()">
            Success Toast
          </button>
          <button nbButton status="warning" (click)="showWarning()">
            Warning Toast
          </button>
          <button nbButton status="danger" (click)="showError()">
            Error Toast
          </button>
          <button nbButton status="info" (click)="showCustom()">
            Custom Toast
          </button>
          <button nbButton status="basic" (click)="showWithAction()">
            Toast with Action
          </button>
          <button nbButton status="primary" (click)="showDifferentPosition()">
            Different Position
          </button>
          <button nbButton status="success" (click)="showPersistent()">
            Persistent Toast
          </button>
        </div>
      </nb-card-body>
    </nb-card>
  `,
  styles: [`
    .button-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 1rem;
    }
  `]
})
export class ToastDemoComponent {
  constructor(private toastrService: NbToastrService) {}

  showInfo() {
    this.toastrService.info(
      'This is an informational message',
      'Info',
      { duration: 3000 }
    );
  }

  showSuccess() {
    this.toastrService.success(
      'Your changes have been saved successfully!',
      'Success'
    );
  }

  showWarning() {
    this.toastrService.warning(
      'Please review the form for errors',
      'Warning',
      { duration: 5000 }
    );
  }

  showError() {
    this.toastrService.danger(
      'An error occurred while processing your request',
      'Error',
      { duration: 0 } // Won't auto-close
    );
  }

  showCustom() {
    this.toastrService.show(
      'This is a custom toast with icon',
      'Custom',
      {
        status: 'primary',
        icon: 'star-outline',
        duration: 4000,
        preventDuplicates: true
      }
    );
  }

  showWithAction() {
    const toast = this.toastrService.info(
      'Click the action button to undo',
      'Action Required',
      {
        duration: 5000,
        destroyByClick: false
      }
    );

    // Can manually close the toast
    setTimeout(() => {
      toast.close();
    }, 10000);
  }

  showDifferentPosition() {
    this.toastrService.success(
      'This toast appears in bottom-right',
      'Position',
      {
        position: NbGlobalPhysicalPosition.BOTTOM_RIGHT,
        duration: 3000
      }
    );
  }

  showPersistent() {
    this.toastrService.warning(
      'This toast will not close automatically',
      'Persistent',
      {
        duration: 0,
        destroyByClick: true,
        preventDuplicates: false
      }
    );
  }
}
```

### 6. Theming System

**Custom Theme Configuration:**
```typescript
// app.module.ts
import { NbThemeModule } from '@nebular/theme';

@NgModule({
  imports: [
    NbThemeModule.forRoot({ 
      name: 'custom',
    }, [
      {
        name: 'custom',
        base: 'default',
        variables: {
          // Color variables
          colorPrimary500: '#3366ff',
          colorSuccess500: '#00d68f',
          colorInfo500: '#0095ff',
          colorWarning500: '#ffaa00',
          colorDanger500: '#ff3d71',
          
          // Typography
          fontFamilyPrimary: 'Inter, sans-serif',
          fontSizeBase: '1rem',
          lineHeightBase: '1.5',
          
          // Spacing
          paddingBase: '1rem',
          marginBase: '1rem',
          
          // Border radius
          borderRadiusBase: '0.375rem'
        }
      }
    ])
  ]
})
export class AppModule { }
```

**Dynamic Theme Switching:**
```typescript
import { Component } from '@angular/core';
import { NbThemeService } from '@nebular/theme';

@Component({
  selector: 'app-theme-switcher',
  template: `
    <nb-card>
      <nb-card-header>Theme Settings</nb-card-header>
      <nb-card-body>
        <div class="theme-options">
          <button 
            nbButton 
            [status]="currentTheme === 'default' ? 'primary' : 'basic'"
            (click)="changeTheme('default')">
            <nb-icon icon="sun-outline"></nb-icon>
            Light
          </button>
          <button 
            nbButton 
            [status]="currentTheme === 'dark' ? 'primary' : 'basic'"
            (click)="changeTheme('dark')">
            <nb-icon icon="moon-outline"></nb-icon>
            Dark
          </button>
          <button 
            nbButton 
            [status]="currentTheme === 'cosmic' ? 'primary' : 'basic'"
            (click)="changeTheme('cosmic')">
            <nb-icon icon="star-outline"></nb-icon>
            Cosmic
          </button>
          <button 
            nbButton 
            [status]="currentTheme === 'corporate' ? 'primary' : 'basic'"
            (click)="changeTheme('corporate')">
            <nb-icon icon="briefcase-outline"></nb-icon>
            Corporate
          </button>
        </div>
      </nb-card-body>
    </nb-card>
  `,
  styles: [`
    .theme-options {
      display: flex;
      gap: 1rem;
      flex-wrap: wrap;
    }
  `]
})
export class ThemeSwitcherComponent {
  currentTheme = 'default';

  constructor(private themeService: NbThemeService) {
    this.themeService.onThemeChange()
      .subscribe((theme: any) => {
        this.currentTheme = theme.name;
      });
  }

  changeTheme(themeName: string) {
    this.themeService.changeTheme(themeName);
  }
}
```

## Authentication Module (@nebular/auth)

**Setup Authentication:**
```typescript
import { NgModule } from '@angular/core';
import { NbAuthModule, NbPasswordAuthStrategy } from '@nebular/auth';
import { NbSecurityModule, NbRoleProvider } from '@nebular/security';

@NgModule({
  imports: [
    NbAuthModule.forRoot({
      strategies: [
        NbPasswordAuthStrategy.setup({
          name: 'email',
          baseEndpoint: 'http://localhost:3000',
          login: {
            endpoint: '/auth/login',
            method: 'post',
          },
          register: {
            endpoint: '/auth/register',
            method: 'post',
          },
          logout: {
            endpoint: '/auth/logout',
            method: 'post',
          },
          requestPass: {
            endpoint: '/auth/request-pass',
            method: 'post',
          },
          resetPass: {
            endpoint: '/auth/reset-pass',
            method: 'post',
          },
        }),
      ],
      forms: {
        login: {
          redirectDelay: 500,
          strategy: 'email',
          rememberMe: true,
          showMessages: {
            success: true,
            error: true,
          },
        },
        register: {
          redirectDelay: 500,
          strategy: 'email',
          showMessages: {
            success: true,
            error: true,
          },
          terms: true,
        },
        requestPassword: {
          redirectDelay: 500,
          strategy: 'email',
          showMessages: {
            success: true,
            error: true,
          },
        },
        resetPassword: {
          redirectDelay: 500,
          strategy: 'email',
          showMessages: {
            success: true,
            error: true,
          },
        },
        logout: {
          redirectDelay: 500,
          strategy: 'email',
        },
      },
    }),
    NbSecurityModule.forRoot({
      accessControl: {
        guest: {
          view: '*',
        },
        user: {
          parent: 'guest',
          create: '*',
          edit: '*',
          remove: '*',
        },
        admin: {
          parent: 'user',
          create: '*',
          edit: '*',
          remove: '*',
        },
      },
    }),
  ],
})
export class AuthModule {}
```

**Login Component:**
```typescript
import { Component } from '@angular/core';
import { NbAuthService, NbAuthResult } from '@nebular/auth';
import { Router } from '@angular/router';

@Component({
  selector: 'app-login',
  template: `
    <nb-layout>
      <nb-layout-column>
        <nb-card>
          <nb-card-header>
            <h3>Login</h3>
          </nb-card-header>
          <nb-card-body>
            <form (ngSubmit)="login()" #form="ngForm">
              <div class="form-group">
                <label class="label">Email</label>
                <input 
                  nbInput
                  fullWidth
                  name="email"
                  [(ngModel)]="user.email"
                  placeholder="Email"
                  type="email"
                  required>
              </div>

              <div class="form-group">
                <label class="label">Password</label>
                <input 
                  nbInput
                  fullWidth
                  name="password"
                  [(ngModel)]="user.password"
                  placeholder="Password"
                  type="password"
                  required>
              </div>

              <nb-checkbox name="rememberMe" [(ngModel)]="rememberMe">
                Remember me
              </nb-checkbox>

              <button 
                nbButton
                fullWidth
                status="primary"
                [disabled]="!form.valid || loading">
                {{loading ? 'Logging in...' : 'Login'}}
              </button>
            </form>

            <div class="links">
              <a routerLink="/auth/register">Create Account</a>
              <a routerLink="/auth/request-password">Forgot Password?</a>
            </div>
          </nb-card-body>
        </nb-card>
      </nb-layout-column>
    </nb-layout>
  `,
  styles: [`
    nb-card {
      max-width: 400px;
      margin: 2rem auto;
    }

    .form-group {
      margin-bottom: 1rem;
    }

    .label {
      display: block;
      margin-bottom: 0.5rem;
    }

    .links {
      display: flex;
      justify-content: space-between;
      margin-top: 1rem;
    }
  `]
})
export class LoginComponent {
  user = {
    email: '',
    password: ''
  };
  rememberMe = false;
  loading = false;

  constructor(
    private authService: NbAuthService,
    private router: Router
  ) {}

  login() {
    this.loading = true;
    this.authService.authenticate('email', this.user)
      .subscribe((result: NbAuthResult) => {
        this.loading = false;
        if (result.isSuccess()) {
          this.router.navigate(['/dashboard']);
        } else {
          // Handle error
          console.error(result.getErrors());
        }
      });
  }
}
```

## Common Mistakes

### 1. Forgetting BrowserAnimationsModule

**Wrong:**
```typescript
@NgModule({
  imports: [BrowserModule, NbThemeModule.forRoot()]
})
export class AppModule { }
```

**Correct:**
```typescript
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';

@NgModule({
  imports: [
    BrowserModule,
    BrowserAnimationsModule, // Required
    NbThemeModule.forRoot()
  ]
})
export class AppModule { }
```

### 2. Not Including Eva Icons

**Wrong:**
```typescript
// Icons won't display without Eva Icons
<nb-icon icon="home-outline"></nb-icon>
```

**Correct:**
```typescript
// app.module.ts
import { NbEvaIconsModule } from '@nebular/eva-icons';

@NgModule({
  imports: [NbEvaIconsModule]
})
export class AppModule { }
```

### 3. Incorrect Theme Import

**Wrong:**
```scss
// styles.scss
@import '@nebular/theme/styles/prebuilt/default.css';
```

**Correct:**
```scss
// Use SCSS mixins for better customization
@import '@nebular/theme/styles/globals';

@include nb-install() {
  @include nb-theme-global();
}
```

### 4. Not Using fullWidth Properly

**Wrong:**
```html
<input nbInput style="width: 100%">
```

**Correct:**
```html
<input nbInput fullWidth>
```

## Best Practices

### 1. Use Feature Modules

```typescript
// feature.module.ts
@NgModule({
  imports: [
    CommonModule,
    NbCardModule,
    NbButtonModule,
    // Only import what you need
  ]
})
export class FeatureModule { }
```

### 2. Implement Theme Customization

```scss
// custom-theme.scss
@import '@nebular/theme/styles/theming';
@import '@nebular/theme/styles/themes/default';

$nb-themes: nb-register-theme((
  color-primary-100: #your-color,
  // Customize other variables
), custom, default);
```

### 3. Use Status Colors Consistently

```typescript
// Use semantic statuses
<button nbButton status="primary">Primary</button>
<button nbButton status="success">Success</button>
<button nbButton status="info">Info</button>
<button nbButton status="warning">Warning</button>
<button nbButton status="danger">Danger</button>
```

### 4. Optimize Bundle Size

```typescript
// Import only needed modules
import { NbButtonModule } from '@nebular/theme';
// Instead of importing entire NbThemeModule everywhere
```

### 5. Use Security Module for Access Control

```typescript
import { NbAccessChecker } from '@nebular/security';

constructor(private accessChecker: NbAccessChecker) {}

canEdit() {
  this.accessChecker.isGranted('edit', 'resource')
    .subscribe(granted => {
      // Handle access
    });
}
```

## When to Use Nebular

### Use When:

- Building modern Angular applications with consistent design
- Need a complete UI solution with auth and security
- Want extensive theming capabilities
- Building admin dashboards or SaaS applications
- Need beautiful, pre-designed components
- Want mobile-responsive components out of the box

### Avoid When:

- Need extremely lightweight solution
- Building simple landing pages
- Require extensive custom styling that conflicts with Nebular's design
- Need extensive documentation (Nebular docs can be sparse)
- Working with older Angular versions (< 12)

## Interview Questions

### Q1: What are the core modules in Nebular and their purposes?

**Answer:**

1. **@nebular/theme**: Core UI components and theming system
   - Layout components (header, sidebar, footer)
   - Form controls
   - Navigation components
   - Data display components

2. **@nebular/auth**: Authentication module
   - Login/register/logout flows
   - Password reset
   - Multiple authentication strategies
   - Token management

3. **@nebular/security**: Access control module
   - Role-based access control
   - Permission checking
   - Access control lists (ACL)

4. **@nebular/eva-icons**: Icon package
   - 480+ icons
   - Eva Design System icons

### Q2: How does Nebular's theming system work?

**Answer:** Nebular uses CSS custom properties and SCSS mixins for theming:

1. **Built-in Themes**: Provides 4 pre-built themes (default, dark, cosmic, corporate)

2. **Theme Structure**: Themes define color palettes, typography, spacing, and component styles

3. **Runtime Switching**: Can switch themes dynamically using `NbThemeService`

4. **Customization**: Create custom themes by extending existing ones:

```scss
$nb-themes: nb-register-theme((
  color-primary-500: #your-color,
  // Override variables
), custom, default);
```

5. **Component Theming**: All components automatically adapt to active theme

### Q3: Explain the status colors system in Nebular.

**Answer:** Nebular uses status colors for semantic meaning:

- **primary**: Main brand color, primary actions
- **success**: Successful operations, positive states
- **info**: Informational messages, neutral actions
- **warning**: Cautionary messages, potentially dangerous actions
- **danger**: Errors, destructive actions
- **basic**: Neutral, default styling

Each status has multiple shades (100-900) for different contexts. Components support status via `status` attribute:

```html
<button nbButton status="primary">Primary</button>
<nb-card accent="success">Success Card</nb-card>
```

### Q4: How do you implement authentication with @nebular/auth?

**Answer:**

1. **Install and Configure**:
```typescript
NbAuthModule.forRoot({
  strategies: [
    NbPasswordAuthStrategy.setup({
      name: 'email',
      baseEndpoint: 'http://api.example.com',
      login: { endpoint: '/auth/login' },
      // Configure endpoints
    })
  ]
})
```

2. **Use Auth Service**:
```typescript
constructor(private authService: NbAuthService) {}

login() {
  this.authService.authenticate('email', credentials)
    .subscribe((result: NbAuthResult) => {
      if (result.isSuccess()) {
        // Handle success
      }
    });
}
```

3. **Protect Routes**:
```typescript
{
  path: 'dashboard',
  canActivate: [NbAuthGuard],
  component: DashboardComponent
}
```

4. **Access Token**:
```typescript
this.authService.getToken()
  .subscribe((token: NbAuthToken) => {
    const tokenValue = token.getValue();
  });
```

### Q5: What are the key differences between Nebular and Material Design?

**Answer:**

**Nebular:**
- Eva Design System based
- More opinionated design language
- Includes auth and security modules
- Better theming system
- Focused on admin/SaaS applications
- Smaller component library
- Beautiful out-of-the-box design

**Angular Material:**
- Material Design based
- More flexible, less opinionated
- Larger component library
- Better documentation
- More widely adopted
- CDK for low-level functionality
- More accessibility features

**Choice depends on**:
- Design requirements
- Need for built-in auth
- Theming requirements
- Component needs
- Community/ecosystem preferences

## Key Takeaways

1. **Complete Solution**: Provides UI, auth, and security in one package
2. **Beautiful Design**: Eva Design System creates modern, attractive interfaces
3. **Powerful Theming**: Comprehensive theming system with runtime switching
4. **Status System**: Semantic color system for consistent component styling
5. **Layout Components**: Built-in layout system for application structure
6. **Auth Integration**: @nebular/auth provides complete authentication flows
7. **Security Module**: Built-in access control and permissions
8. **Eva Icons**: 480+ icons included
9. **Mobile Responsive**: All components are responsive by default
10. **Form Components**: Rich set of form controls with consistent styling
11. **Customizable**: Highly customizable through SCSS variables
12. **Size Variants**: Components support multiple sizes (tiny, small, medium, large, giant)
13. **Appearance Variants**: Multiple appearances (filled, outline, ghost, hero)
14. **Active Development**: Regularly updated by Akveo
15. **Good for Admin Panels**: Particularly well-suited for admin dashboards and SaaS applications

## Resources

- **Official Documentation**: https://akveo.github.io/nebular/
- **GitHub Repository**: https://github.com/akveo/nebular
- **Eva Design System**: https://eva.design/
- **Eva Icons**: https://akveo.github.io/eva-icons/
- **Nebular Auth Docs**: https://akveo.github.io/nebular/docs/auth/introduction
- **Nebular Security**: https://akveo.github.io/nebular/docs/security/introduction
- **Themes Documentation**: https://akveo.github.io/nebular/docs/design-system/eva-design-system-theme
- **Component Examples**: https://akveo.github.io/nebular/docs/components/components-overview
- **Starter Kits**: https://github.com/akveo/ngx-admin
- **Community**: https://github.com/akveo/nebular/discussions