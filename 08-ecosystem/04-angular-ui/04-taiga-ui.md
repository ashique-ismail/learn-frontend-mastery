# Taiga UI - Lightweight Angular UI Kit

## Overview

Taiga UI is a lightweight, customizable Angular UI Kit with a focus on performance, accessibility, and developer experience. Developed by Tinkoff, Taiga UI provides a comprehensive set of components, directives, and utilities with extensive customization options, design tokens, and excellent mobile support. It emphasizes small bundle sizes, tree-shaking, and modern Angular features.

## Installation

### Prerequisites

```bash
# Required dependencies
npm install @angular/core @angular/common @angular/forms
```

### Installation Steps

```bash
# Install Taiga UI core and components
npm install @taiga-ui/cdk @taiga-ui/core @taiga-ui/kit

# Install icons
npm install @taiga-ui/icons

# Optional: Install additional packages
npm install @taiga-ui/addon-commerce
npm install @taiga-ui/addon-table
npm install @taiga-ui/addon-charts
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
              "node_modules/@taiga-ui/core/styles/taiga-ui-theme.less",
              "node_modules/@taiga-ui/core/styles/taiga-ui-fonts.less",
              "src/styles.less"
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
import { TuiRootModule, TuiDialogModule, TuiAlertModule, TUI_SANITIZER } from '@taiga-ui/core';
import { TuiInputModule, TuiIslandModule } from '@taiga-ui/kit';
import { NgDompurifySanitizer } from '@taiga-ui/dompurify';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    TuiRootModule,
    TuiDialogModule,
    TuiAlertModule,
    TuiInputModule,
    TuiIslandModule
  ],
  providers: [
    { provide: TUI_SANITIZER, useClass: NgDompurifySanitizer }
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

**app.component.html:**
```html
<tui-root>
  <router-outlet></router-outlet>
</tui-root>
```

## Core Concepts

### 1. Package Structure

Taiga UI is split into focused packages:

- **@taiga-ui/cdk**: Core Development Kit (utilities, directives)
- **@taiga-ui/core**: Core components (buttons, inputs, layout)
- **@taiga-ui/kit**: Extended component kit
- **@taiga-ui/icons**: Icon set
- **@taiga-ui/addon-***: Optional feature addons

### 2. Design Tokens

Taiga UI uses CSS custom properties for theming:

```less
:root {
  --tui-primary: #526ed3;
  --tui-accent: #ff7043;
  --tui-success: #3aa981;
  --tui-error: #dd4c1e;
}
```

### 3. Size System

Components support standardized sizes: `s`, `m`, `l`, `xl`

## Component Examples

### 1. Form Components

**Comprehensive Form Example:**
```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { TuiDay, TuiTime } from '@taiga-ui/cdk';

@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()" class="form">
      <!-- Text Input -->
      <tui-input formControlName="name" [tuiTextfieldLabelOutside]="true">
        Full Name
        <input tuiTextfield placeholder="Enter your name" />
      </tui-input>

      <!-- Email Input with Icon -->
      <tui-input 
        formControlName="email"
        [tuiTextfieldLabelOutside]="true">
        Email
        <input 
          tuiTextfield 
          type="email"
          placeholder="email@example.com" />
        <tui-svg 
          tuiTextfieldIcon
          src="tuiIconMail">
        </tui-svg>
      </tui-input>

      <!-- Password Input -->
      <tui-input-password
        formControlName="password"
        [tuiTextfieldLabelOutside]="true">
        Password
        <input tuiTextfield placeholder="Enter password" />
      </tui-input-password>

      <!-- Phone Input -->
      <tui-input-phone
        formControlName="phone"
        [tuiTextfieldLabelOutside]="true"
        [phoneMaskAfterCountryCode]="phoneMask">
        Phone Number
      </tui-input-phone>

      <!-- Number Input with Slider -->
      <tui-input-slider
        formControlName="age"
        [min]="18"
        [max]="100"
        [tuiTextfieldLabelOutside]="true">
        Age
      </tui-input-slider>

      <!-- Select -->
      <tui-select
        formControlName="country"
        [tuiTextfieldLabelOutside]="true"
        [valueContent]="countryContent">
        Country
        <tui-data-list-wrapper
          *tuiDataList
          [items]="countries">
        </tui-data-list-wrapper>
      </tui-select>

      <ng-template #countryContent let-data>
        <div class="country-item">
          <tui-flag [countryCode]="data.code"></tui-flag>
          <span>{{data.name}}</span>
        </div>
      </ng-template>

      <!-- Multi Select -->
      <tui-multi-select
        formControlName="interests"
        [tuiTextfieldLabelOutside]="true">
        Interests
        <tui-data-list-wrapper
          *tuiDataList
          [items]="interests">
        </tui-data-list-wrapper>
      </tui-multi-select>

      <!-- Date Picker -->
      <tui-input-date
        formControlName="birthDate"
        [tuiTextfieldLabelOutside]="true"
        [max]="maxDate">
        Birth Date
      </tui-input-date>

      <!-- Date Range Picker -->
      <tui-input-date-range
        formControlName="dateRange"
        [tuiTextfieldLabelOutside]="true">
        Date Range
      </tui-input-date-range>

      <!-- Time Input -->
      <tui-input-time
        formControlName="time"
        [tuiTextfieldLabelOutside]="true"
        [items]="timeItems">
        Preferred Time
      </tui-input-time>

      <!-- Text Area -->
      <tui-text-area
        formControlName="bio"
        [tuiTextfieldLabelOutside]="true"
        [expandable]="true"
        [rows]="3">
        Biography
        <textarea tuiTextfield></textarea>
      </tui-text-area>

      <!-- Checkbox Group -->
      <div class="form-group">
        <label tuiLabel>Preferences</label>
        <tui-checkbox-labeled formControlName="newsletter">
          Subscribe to newsletter
        </tui-checkbox-labeled>
        <tui-checkbox-labeled formControlName="notifications">
          Enable notifications
        </tui-checkbox-labeled>
      </div>

      <!-- Radio Group -->
      <div class="form-group">
        <label tuiLabel>Account Type</label>
        <tui-radio-list
          formControlName="accountType"
          [items]="accountTypes"
          [orientation]="'horizontal'">
        </tui-radio-list>
      </div>

      <!-- Toggle -->
      <tui-toggle formControlName="isActive" class="toggle">
        Active Account
      </tui-toggle>

      <!-- File Input -->
      <tui-input-files
        formControlName="avatar"
        [accept]="'image/*'"
        [maxFileSize]="5000000"
        [tuiTextfieldLabelOutside]="true">
        Profile Picture
      </tui-input-files>

      <!-- Color Picker -->
      <tui-input-color
        formControlName="favoriteColor"
        [tuiTextfieldLabelOutside]="true">
        Favorite Color
      </tui-input-color>

      <!-- Range Slider -->
      <div class="form-group">
        <label tuiLabel>
          Price Range: {{userForm.value.priceRange?.join(' - ')}}
        </label>
        <tui-range formControlName="priceRange" [min]="0" [max]="1000">
        </tui-range>
      </div>

      <!-- Buttons -->
      <div class="button-group">
        <button
          tuiButton
          type="submit"
          [disabled]="!userForm.valid || submitting"
          appearance="primary"
          size="l">
          <tui-svg src="tuiIconCheck"></tui-svg>
          {{submitting ? 'Submitting...' : 'Submit'}}
        </button>
        <button
          tuiButton
          type="button"
          appearance="secondary"
          size="l"
          (click)="onReset()">
          Reset
        </button>
      </div>
    </form>
  `,
  styles: [`
    .form {
      max-width: 600px;
      margin: 2rem auto;
      display: flex;
      flex-direction: column;
      gap: 1.5rem;
    }

    .form-group {
      display: flex;
      flex-direction: column;
      gap: 0.5rem;
    }

    .button-group {
      display: flex;
      gap: 1rem;
      margin-top: 1rem;
    }

    .country-item {
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }

    .toggle {
      display: flex;
      align-items: center;
    }
  `]
})
export class UserFormComponent implements OnInit {
  userForm!: FormGroup;
  submitting = false;
  maxDate = TuiDay.currentLocal();
  phoneMask = '(###) ###-##-##';

  countries = [
    { name: 'United States', code: 'US' },
    { name: 'United Kingdom', code: 'GB' },
    { name: 'Canada', code: 'CA' },
    { name: 'Australia', code: 'AU' }
  ];

  interests = [
    'Technology', 'Sports', 'Music', 'Travel', 
    'Reading', 'Gaming', 'Cooking', 'Art'
  ];

  accountTypes = ['Personal', 'Business'];

  timeItems = [
    TuiTime.fromString('09:00'),
    TuiTime.fromString('12:00'),
    TuiTime.fromString('15:00'),
    TuiTime.fromString('18:00')
  ];

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.userForm = this.fb.group({
      name: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(8)]],
      phone: ['', Validators.required],
      age: [25, [Validators.min(18), Validators.max(100)]],
      country: [null, Validators.required],
      interests: [[], Validators.required],
      birthDate: [null, Validators.required],
      dateRange: [null],
      time: [null],
      bio: [''],
      newsletter: [false],
      notifications: [true],
      accountType: ['Personal'],
      isActive: [true],
      avatar: [null],
      favoriteColor: ['#526ed3'],
      priceRange: [[0, 500]]
    });
  }

  onSubmit() {
    if (this.userForm.valid) {
      this.submitting = true;
      console.log('Form submitted:', this.userForm.value);
      
      setTimeout(() => {
        this.submitting = false;
      }, 2000);
    }
  }

  onReset() {
    this.userForm.reset({
      age: 25,
      notifications: true,
      accountType: 'Personal',
      isActive: true,
      favoriteColor: '#526ed3',
      priceRange: [0, 500]
    });
  }
}
```

### 2. Islands (Cards)

**Dashboard with Islands:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <!-- Simple Island -->
      <tui-island class="metric-card">
        <h3 class="tui-island__title">Total Users</h3>
        <div class="metric-value">12,453</div>
        <div class="metric-change positive">
          <tui-svg src="tuiIconTrendingUp"></tui-svg>
          <span>+5.2% from last month</span>
        </div>
      </tui-island>

      <!-- Island with Badge -->
      <tui-island class="metric-card">
        <div class="tui-island__header">
          <h3 class="tui-island__title">Active Sessions</h3>
          <tui-badge status="success" value="Live"></tui-badge>
        </div>
        <div class="metric-value">1,847</div>
        <tui-line-clamp [lineHeight]="20" [linesLimit]="2">
          Current active user sessions across all platforms
        </tui-line-clamp>
      </tui-island>

      <!-- Island with Actions -->
      <tui-island class="action-card">
        <tui-svg 
          src="tuiIconFile" 
          class="card-icon">
        </tui-svg>
        <h3 class="tui-island__title">Documents</h3>
        <p class="tui-island__paragraph">
          125 files, 2.3 GB
        </p>
        <div class="card-actions">
          <button tuiButton appearance="flat" size="s">
            View All
          </button>
          <button tuiIconButton appearance="icon" size="s">
            <tui-svg src="tuiIconMoreHorizontal"></tui-svg>
          </button>
        </div>
      </tui-island>

      <!-- Hoverable Island -->
      <tui-island 
        [hoverable]="true" 
        class="clickable-card"
        (click)="onCardClick()">
        <tui-svg 
          src="tuiIconSettings" 
          class="card-icon">
        </tui-svg>
        <h3 class="tui-island__title">Settings</h3>
        <p class="tui-island__paragraph">
          Configure your application preferences
        </p>
        <tui-svg 
          src="tuiIconChevronRight"
          class="card-arrow">
        </tui-svg>
      </tui-island>

      <!-- Island with Avatar -->
      <tui-island class="profile-card">
        <tui-avatar
          [avatarUrl]="'assets/avatar.jpg'"
          [text]="'John Doe'"
          size="l">
        </tui-avatar>
        <div class="profile-info">
          <h3 class="tui-island__title">John Doe</h3>
          <p class="tui-island__paragraph">Administrator</p>
          <div class="profile-stats">
            <div class="stat">
              <span class="stat-value">156</span>
              <span class="stat-label">Posts</span>
            </div>
            <div class="stat">
              <span class="stat-value">1.2K</span>
              <span class="stat-label">Followers</span>
            </div>
          </div>
        </div>
      </tui-island>

      <!-- Island with Progress -->
      <tui-island class="progress-card">
        <h3 class="tui-island__title">Project Progress</h3>
        <p class="tui-island__paragraph">Development Phase</p>
        <tui-progress
          [value]="progressValue"
          [max]="100"
          size="m"
          class="progress-bar">
        </tui-progress>
        <div class="progress-label">
          {{progressValue}}% Complete
        </div>
      </tui-island>

      <!-- Island with Accordion -->
      <tui-island class="accordion-card">
        <tui-accordion>
          <tui-accordion-item>
            <ng-template tuiAccordionItemContent>
              <div class="accordion-header">
                <h4>Recent Activity</h4>
                <tui-badge status="info" value="3"></tui-badge>
              </div>
            </ng-template>
            <div class="accordion-body">
              <div *ngFor="let activity of activities" class="activity-item">
                <tui-svg [src]="activity.icon"></tui-svg>
                <div class="activity-details">
                  <div class="activity-title">{{activity.title}}</div>
                  <div class="activity-time">{{activity.time}}</div>
                </div>
              </div>
            </div>
          </tui-accordion-item>
        </tui-accordion>
      </tui-island>

      <!-- Island with Tabs -->
      <tui-island class="tabbed-card">
        <tui-tabs [(activeItemIndex)]="activeTabIndex">
          <button tuiTab>Overview</button>
          <button tuiTab>Details</button>
          <button tuiTab>Settings</button>
        </tui-tabs>
        <div class="tab-content">
          <div *ngIf="activeTabIndex === 0">
            <h4>Overview Content</h4>
            <p>This is the overview tab content.</p>
          </div>
          <div *ngIf="activeTabIndex === 1">
            <h4>Details Content</h4>
            <p>This is the details tab content.</p>
          </div>
          <div *ngIf="activeTabIndex === 2">
            <h4>Settings Content</h4>
            <p>This is the settings tab content.</p>
          </div>
        </div>
      </tui-island>
    </div>
  `,
  styles: [`
    .dashboard {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
      gap: 1.5rem;
      padding: 2rem;
    }

    .metric-card {
      padding: 1.5rem;
    }

    .metric-value {
      font-size: 2.5rem;
      font-weight: bold;
      margin: 1rem 0;
    }

    .metric-change {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      font-size: 0.875rem;
    }

    .metric-change.positive {
      color: var(--tui-success-fill);
    }

    .action-card,
    .clickable-card {
      padding: 1.5rem;
      display: flex;
      flex-direction: column;
      gap: 0.5rem;
    }

    .card-icon {
      width: 48px;
      height: 48px;
      margin-bottom: 0.5rem;
    }

    .card-actions {
      display: flex;
      gap: 0.5rem;
      margin-top: 1rem;
    }

    .clickable-card {
      cursor: pointer;
      position: relative;
    }

    .card-arrow {
      position: absolute;
      top: 1rem;
      right: 1rem;
    }

    .profile-card {
      padding: 1.5rem;
      display: flex;
      align-items: center;
      gap: 1rem;
    }

    .profile-info {
      flex: 1;
    }

    .profile-stats {
      display: flex;
      gap: 2rem;
      margin-top: 0.5rem;
    }

    .stat {
      display: flex;
      flex-direction: column;
    }

    .stat-value {
      font-size: 1.25rem;
      font-weight: bold;
    }

    .stat-label {
      font-size: 0.75rem;
      color: var(--tui-text-03);
    }

    .progress-card {
      padding: 1.5rem;
    }

    .progress-bar {
      margin: 1rem 0;
    }

    .progress-label {
      text-align: center;
      font-size: 0.875rem;
    }

    .accordion-card,
    .tabbed-card {
      padding: 1.5rem;
    }

    .accordion-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .accordion-body {
      display: flex;
      flex-direction: column;
      gap: 1rem;
      padding-top: 1rem;
    }

    .activity-item {
      display: flex;
      gap: 1rem;
      align-items: start;
    }

    .activity-details {
      flex: 1;
    }

    .activity-title {
      font-weight: 500;
    }

    .activity-time {
      font-size: 0.75rem;
      color: var(--tui-text-03);
    }

    .tab-content {
      padding: 1rem 0;
    }
  `]
})
export class DashboardComponent {
  progressValue = 67;
  activeTabIndex = 0;

  activities = [
    { icon: 'tuiIconUser', title: 'New user registered', time: '2m ago' },
    { icon: 'tuiIconFile', title: 'Document uploaded', time: '15m ago' },
    { icon: 'tuiIconSettings', title: 'Settings updated', time: '1h ago' }
  ];

  onCardClick() {
    console.log('Card clicked');
  }
}
```

### 3. Dialogs and Notifications

**Dialog Service:**
```typescript
import { Component, Inject, Injector } from '@angular/core';
import { TuiDialogService } from '@taiga-ui/core';
import { PolymorpheusComponent } from '@tinkoff/ng-polymorpheus';

@Component({
  selector: 'app-dialog-demo',
  template: `
    <div class="demo-container">
      <button tuiButton appearance="primary" (click)="showDialog()">
        Show Dialog
      </button>
      <button tuiButton appearance="secondary" (click)="showConfirmDialog()">
        Confirm Dialog
      </button>
      <button tuiButton appearance="accent" (click)="showCustomDialog()">
        Custom Dialog
      </button>
      <button tuiButton appearance="flat" (click)="showNotification()">
        Show Notification
      </button>
    </div>
  `,
  styles: [`
    .demo-container {
      display: flex;
      gap: 1rem;
      padding: 2rem;
    }
  `]
})
export class DialogDemoComponent {
  constructor(
    @Inject(TuiDialogService) private readonly dialogService: TuiDialogService,
    @Inject(Injector) private readonly injector: Injector
  ) {}

  showDialog() {
    this.dialogService.open(
      'This is a simple dialog message',
      {
        label: 'Information',
        size: 's',
        closeable: true,
        dismissible: true
      }
    ).subscribe();
  }

  showConfirmDialog() {
    this.dialogService.open(
      'Are you sure you want to proceed with this action?',
      {
        label: 'Confirm',
        size: 's',
        data: { button: 'Yes, proceed' }
      }
    ).subscribe(result => {
      console.log('Dialog result:', result);
    });
  }

  showCustomDialog() {
    this.dialogService.open(
      new PolymorpheusComponent(CustomDialogComponent, this.injector),
      {
        label: 'Custom Dialog',
        size: 'l',
        closeable: true,
        dismissible: false
      }
    ).subscribe(result => {
      console.log('Custom dialog result:', result);
    });
  }

  showNotification() {
    this.dialogService.open(
      'Your changes have been saved successfully',
      {
        label: 'Success',
        status: 'success',
        size: 's',
        autoClose: 3000
      }
    ).subscribe();
  }
}

@Component({
  selector: 'app-custom-dialog',
  template: `
    <div class="dialog-content">
      <form [formGroup]="form" (ngSubmit)="onSubmit()">
        <tui-input formControlName="title" [tuiTextfieldLabelOutside]="true">
          Title
          <input tuiTextfield placeholder="Enter title" />
        </tui-input>

        <tui-text-area
          formControlName="description"
          [tuiTextfieldLabelOutside]="true"
          [rows]="4">
          Description
          <textarea tuiTextfield></textarea>
        </tui-text-area>

        <tui-select formControlName="priority" [tuiTextfieldLabelOutside]="true">
          Priority
          <tui-data-list-wrapper
            *tuiDataList
            [items]="priorities">
          </tui-data-list-wrapper>
        </tui-select>

        <div class="button-group">
          <button tuiButton type="button" appearance="secondary" (click)="onCancel()">
            Cancel
          </button>
          <button tuiButton type="submit" appearance="primary" [disabled]="!form.valid">
            Save
          </button>
        </div>
      </form>
    </div>
  `,
  styles: [`
    .dialog-content {
      display: flex;
      flex-direction: column;
      gap: 1.5rem;
      padding: 1rem;
    }

    .button-group {
      display: flex;
      gap: 1rem;
      justify-content: flex-end;
      margin-top: 1rem;
    }
  `]
})
export class CustomDialogComponent {
  form = this.fb.group({
    title: ['', Validators.required],
    description: [''],
    priority: ['medium']
  });

  priorities = ['low', 'medium', 'high'];

  constructor(
    @Inject(TuiDialogService) private readonly dialogService: TuiDialogService,
    private fb: FormBuilder
  ) {}

  onSubmit() {
    if (this.form.valid) {
      // Return form value to parent
      this.dialogService.open(this.form.value).subscribe();
    }
  }

  onCancel() {
    // Close dialog without result
    this.dialogService.open(null).subscribe();
  }
}
```

**Alerts Service:**
```typescript
import { Component, Inject } from '@angular/core';
import { TuiAlertService, TuiNotification } from '@taiga-ui/core';

@Component({
  selector: 'app-alerts-demo',
  template: `
    <div class="button-grid">
      <button tuiButton appearance="primary" (click)="showInfo()">
        Info Alert
      </button>
      <button tuiButton appearance="primary" (click)="showSuccess()">
        Success Alert
      </button>
      <button tuiButton appearance="primary" (click)="showWarning()">
        Warning Alert
      </button>
      <button tuiButton appearance="primary" (click)="showError()">
        Error Alert
      </button>
      <button tuiButton appearance="secondary" (click)="showCustom()">
        Custom Alert
      </button>
      <button tuiButton appearance="accent" (click)="showWithAction()">
        Alert with Action
      </button>
    </div>
  `,
  styles: [`
    .button-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 1rem;
      padding: 2rem;
    }
  `]
})
export class AlertsDemoComponent {
  constructor(
    @Inject(TuiAlertService) private readonly alertService: TuiAlertService
  ) {}

  showInfo() {
    this.alertService.open('This is an informational message', {
      label: 'Info',
      status: TuiNotification.Info,
      autoClose: 3000
    }).subscribe();
  }

  showSuccess() {
    this.alertService.open('Operation completed successfully!', {
      label: 'Success',
      status: TuiNotification.Success,
      autoClose: 3000
    }).subscribe();
  }

  showWarning() {
    this.alertService.open('Please review your input', {
      label: 'Warning',
      status: TuiNotification.Warning,
      autoClose: 5000
    }).subscribe();
  }

  showError() {
    this.alertService.open('An error occurred while processing your request', {
      label: 'Error',
      status: TuiNotification.Error,
      autoClose: 0 // Won't auto-close
    }).subscribe();
  }

  showCustom() {
    this.alertService.open(
      new PolymorpheusComponent(CustomAlertComponent, this.injector),
      {
        label: 'Custom Alert',
        status: TuiNotification.Info,
        hasCloseButton: true
      }
    ).subscribe();
  }

  showWithAction() {
    this.alertService.open('Your session is about to expire', {
      label: 'Session Expiring',
      status: TuiNotification.Warning,
      autoClose: false,
      hasCloseButton: true,
      data: {
        button: 'Extend Session'
      }
    }).subscribe(action => {
      if (action) {
        console.log('Session extended');
      }
    });
  }
}
```

### 4. Tables

**Advanced Table with Taiga UI:**
```typescript
import { Component } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
  status: 'active' | 'inactive';
  joinDate: Date;
}

@Component({
  selector: 'app-data-table',
  template: `
    <tui-island>
      <div class="table-header">
        <h3>User Management</h3>
        <div class="table-actions">
          <tui-input [(ngModel)]="searchTerm" [tuiTextfieldLabelOutside]="true">
            <input tuiTextfield placeholder="Search users..." />
            <tui-svg tuiTextfieldIcon src="tuiIconSearch"></tui-svg>
          </tui-input>
          <button tuiButton appearance="primary" (click)="onAddUser()">
            <tui-svg src="tuiIconPlus"></tui-svg>
            Add User
          </button>
        </div>
      </div>

      <table tuiTable [columns]="columns" class="table">
        <thead>
          <tr tuiThGroup>
            <th *tuiHead="'id'" tuiTh [sorter]="null">ID</th>
            <th *tuiHead="'name'" tuiTh [sorter]="null">Name</th>
            <th *tuiHead="'email'" tuiTh [sorter]="null">Email</th>
            <th *tuiHead="'role'" tuiTh>Role</th>
            <th *tuiHead="'status'" tuiTh>Status</th>
            <th *tuiHead="'joinDate'" tuiTh [sorter]="null">Join Date</th>
            <th *tuiHead="'actions'" tuiTh>Actions</th>
          </tr>
        </thead>
        <tbody tuiTbody [data]="filteredUsers">
          <tr *tuiRow="let user of filteredUsers" tuiTr>
            <td *tuiCell="'id'" tuiTd>{{user.id}}</td>
            <td *tuiCell="'name'" tuiTd>
              <div class="user-cell">
                <tui-avatar
                  [text]="user.name"
                  size="s">
                </tui-avatar>
                <span>{{user.name}}</span>
              </div>
            </td>
            <td *tuiCell="'email'" tuiTd>{{user.email}}</td>
            <td *tuiCell="'role'" tuiTd>
              <tui-badge [value]="user.role"></tui-badge>
            </td>
            <td *tuiCell="'status'" tuiTd>
              <tui-badge
                [value]="user.status"
                [status]="user.status === 'active' ? 'success' : 'error'">
              </tui-badge>
            </td>
            <td *tuiCell="'joinDate'" tuiTd>
              {{user.joinDate | date:'mediumDate'}}
            </td>
            <td *tuiCell="'actions'" tuiTd>
              <div class="action-buttons">
                <button
                  tuiIconButton
                  appearance="icon"
                  size="s"
                  (click)="onEdit(user)">
                  <tui-svg src="tuiIconEdit"></tui-svg>
                </button>
                <button
                  tuiIconButton
                  appearance="icon"
                  size="s"
                  (click)="onDelete(user)">
                  <tui-svg src="tuiIconTrash"></tui-svg>
                </button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>

      <tui-pagination
        [length]="totalPages"
        [(index)]="currentPage"
        (indexChange)="onPageChange($event)"
        class="pagination">
      </tui-pagination>
    </tui-island>
  `,
  styles: [`
    .table-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1.5rem;
      padding: 1rem;
    }

    .table-actions {
      display: flex;
      gap: 1rem;
      align-items: center;
    }

    .table {
      width: 100%;
    }

    .user-cell {
      display: flex;
      align-items: center;
      gap: 0.5rem;
    }

    .action-buttons {
      display: flex;
      gap: 0.5rem;
    }

    .pagination {
      margin-top: 1rem;
      padding: 1rem;
    }
  `]
})
export class DataTableComponent {
  columns = ['id', 'name', 'email', 'role', 'status', 'joinDate', 'actions'];
  searchTerm = '';
  currentPage = 0;
  totalPages = 5;

  users: User[] = [
    {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com',
      role: 'Admin',
      status: 'active',
      joinDate: new Date('2024-01-15')
    },
    {
      id: 2,
      name: 'Jane Smith',
      email: 'jane@example.com',
      role: 'User',
      status: 'active',
      joinDate: new Date('2024-02-20')
    },
    {
      id: 3,
      name: 'Bob Johnson',
      email: 'bob@example.com',
      role: 'User',
      status: 'inactive',
      joinDate: new Date('2024-03-10')
    }
  ];

  get filteredUsers(): User[] {
    if (!this.searchTerm) return this.users;
    
    return this.users.filter(user =>
      user.name.toLowerCase().includes(this.searchTerm.toLowerCase()) ||
      user.email.toLowerCase().includes(this.searchTerm.toLowerCase())
    );
  }

  onAddUser() {
    console.log('Add user');
  }

  onEdit(user: User) {
    console.log('Edit user:', user);
  }

  onDelete(user: User) {
    console.log('Delete user:', user);
  }

  onPageChange(page: number) {
    console.log('Page changed:', page);
  }
}
```

### 5. Mobile-First Components

**Mobile-Optimized Layout:**
```typescript
import { Component } from '@angular/core';
import { TuiSwipeService, TuiSwipeDirection } from '@taiga-ui/cdk';

@Component({
  selector: 'app-mobile-layout',
  template: `
    <div class="mobile-container">
      <!-- Pull to Refresh -->
      <tui-pull-to-refresh
        [loading]="isRefreshing"
        (pulled)="onRefresh()">
        <div class="content">
          <!-- Mobile-Friendly Cards -->
          <tui-island
            *ngFor="let item of items"
            class="mobile-card"
            (tuiSwipe)="onSwipe($event, item)">
            <div class="card-header">
              <h4>{{item.title}}</h4>
              <button tuiIconButton appearance="icon" size="s">
                <tui-svg src="tuiIconMoreVertical"></tui-svg>
              </button>
            </div>
            <p>{{item.description}}</p>
            <div class="card-footer">
              <span class="timestamp">{{item.timestamp}}</span>
              <tui-badge [value]="item.category"></tui-badge>
            </div>
          </tui-island>
        </div>
      </tui-pull-to-refresh>

      <!-- Bottom Navigation -->
      <tui-tabs class="bottom-nav">
        <button tuiTab>
          <tui-svg src="tuiIconHome"></tui-svg>
          <span>Home</span>
        </button>
        <button tuiTab>
          <tui-svg src="tuiIconSearch"></tui-svg>
          <span>Search</span>
        </button>
        <button tuiTab>
          <tui-svg src="tuiIconPlus"></tui-svg>
          <span>Add</span>
        </button>
        <button tuiTab>
          <tui-svg src="tuiIconBell"></tui-svg>
          <span>Notifications</span>
        </button>
        <button tuiTab>
          <tui-svg src="tuiIconUser"></tui-svg>
          <span>Profile</span>
        </button>
      </tui-tabs>

      <!-- Floating Action Button -->
      <button
        tuiButton
        appearance="primary"
        size="l"
        class="fab"
        (click)="onFabClick()">
        <tui-svg src="tuiIconPlus"></tui-svg>
      </button>
    </div>
  `,
  styles: [`
    .mobile-container {
      min-height: 100vh;
      padding-bottom: 60px; /* Space for bottom nav */
    }

    .content {
      padding: 1rem;
      display: flex;
      flex-direction: column;
      gap: 1rem;
    }

    .mobile-card {
      padding: 1rem;
    }

    .card-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 0.5rem;
    }

    .card-footer {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 1rem;
    }

    .timestamp {
      font-size: 0.75rem;
      color: var(--tui-text-03);
    }

    .bottom-nav {
      position: fixed;
      bottom: 0;
      left: 0;
      right: 0;
      background: var(--tui-base-01);
      border-top: 1px solid var(--tui-base-03);
      padding: 0.5rem;
      display: flex;
      justify-content: space-around;
      z-index: 100;
    }

    .bottom-nav button {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 0.25rem;
      font-size: 0.75rem;
    }

    .fab {
      position: fixed;
      bottom: 80px;
      right: 1rem;
      border-radius: 50%;
      width: 56px;
      height: 56px;
      box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
      z-index: 99;
    }
  `]
})
export class MobileLayoutComponent {
  isRefreshing = false;

  items = [
    {
      title: 'Item 1',
      description: 'Description for item 1',
      timestamp: '2m ago',
      category: 'News'
    },
    {
      title: 'Item 2',
      description: 'Description for item 2',
      timestamp: '15m ago',
      category: 'Updates'
    }
  ];

  onRefresh() {
    this.isRefreshing = true;
    setTimeout(() => {
      this.isRefreshing = false;
      console.log('Content refreshed');
    }, 2000);
  }

  onSwipe(direction: TuiSwipeDirection, item: any) {
    console.log('Swiped:', direction, item);
  }

  onFabClick() {
    console.log('FAB clicked');
  }
}
```

## Common Mistakes

### 1. Not Wrapping App in TuiRoot

**Wrong:**
```html
<router-outlet></router-outlet>
```

**Correct:**
```html
<tui-root>
  <router-outlet></router-outlet>
</tui-root>
```

### 2. Missing Sanitizer Provider

**Wrong:**
```typescript
// No sanitizer configured
@NgModule({
  providers: []
})
```

**Correct:**
```typescript
import { TUI_SANITIZER } from '@taiga-ui/core';
import { NgDompurifySanitizer } from '@taiga-ui/dompurify';

@NgModule({
  providers: [
    { provide: TUI_SANITIZER, useClass: NgDompurifySanitizer }
  ]
})
```

### 3. Incorrect Textfield Usage

**Wrong:**
```html
<tui-input>
  <input type="text" />
</tui-input>
```

**Correct:**
```html
<tui-input>
  <input tuiTextfield />
</tui-input>
```

### 4. Not Using Data List Wrapper

**Wrong:**
```html
<tui-select>
  <option *ngFor="let item of items">{{item}}</option>
</tui-select>
```

**Correct:**
```html
<tui-select>
  <tui-data-list-wrapper
    *tuiDataList
    [items]="items">
  </tui-data-list-wrapper>
</tui-select>
```

## Best Practices

### 1. Tree-Shake Unused Components

```typescript
// Import only what you need
import { TuiInputModule } from '@taiga-ui/kit';
// Instead of importing everything
```

### 2. Use Design Tokens for Theming

```less
// custom-theme.less
:root {
  --tui-primary: #your-brand-color;
  --tui-primary-hover: #your-hover-color;
}
```

### 3. Leverage CDK Utilities

```typescript
import { tuiPure } from '@taiga-ui/cdk';

@Component({...})
class MyComponent {
  @tuiPure
  expensiveCalculation(data: any[]): any {
    // Will be memoized
    return data.reduce(...);
  }
}
```

### 4. Use Polymorpheus for Dynamic Content

```typescript
import { PolymorpheusComponent } from '@tinkoff/ng-polymorpheus';

const content = new PolymorpheusComponent(MyComponent, injector);
this.dialogService.open(content);
```

### 5. Optimize Bundle Size

```typescript
// Use providedIn for tree-shakeable services
@Injectable({ providedIn: 'root' })
class MyService {}
```

## When to Use Taiga UI

### Use When:

- Building lightweight, performant Angular applications
- Need excellent mobile support and touch interactions
- Want extensive customization with design tokens
- Building modern, minimalist interfaces
- Need comprehensive form components
- Want smaller bundle sizes
- Building Progressive Web Apps (PWAs)

### Avoid When:

- Need extensive pre-built complex components
- Require Material Design specifically
- Need extensive ecosystem/community support
- Building applications requiring IE11 support
- Need extensive charting/visualization components (though addon available)

## Interview Questions

### Q1: What makes Taiga UI different from other Angular UI libraries?

**Answer:** Taiga UI distinguishes itself through:

1. **Lightweight**: Smaller bundle sizes through tree-shaking
2. **Performance**: Optimized for performance with CDK utilities
3. **Mobile-First**: Excellent touch support and mobile components
4. **Customization**: Extensive design token system
5. **Modern**: Uses latest Angular features and best practices
6. **Modular**: Split into focused packages (@taiga-ui/cdk, core, kit)
7. **Developer Experience**: Great TypeScript support and API design

It's developed by Tinkoff, a major Russian fintech company, and reflects real-world production requirements.

### Q2: How does Taiga UI's theming system work with design tokens?

**Answer:** Taiga UI uses CSS custom properties (design tokens) for theming:

```less
:root {
  --tui-primary: #526ed3;
  --tui-primary-hover: #6c86e2;
  --tui-text-01: #333333;
  --tui-base-01: #ffffff;
}
```

**Benefits:**
- Runtime theme switching
- No rebuild required for theme changes
- Easy customization per component
- Consistent design system
- Better performance than SCSS variables

**Usage:**
```less
.my-component {
  color: var(--tui-primary);
  background: var(--tui-base-01);
}
```

### Q3: Explain the package structure of Taiga UI.

**Answer:**

**@taiga-ui/cdk (CDK - Component Dev Kit):**
- Low-level utilities and directives
- No visual components
- Utilities like `tuiPure`, `tuiZonefree`
- Services and tokens
- Tree-shakeable

**@taiga-ui/core:**
- Core UI components (buttons, inputs, dialogs)
- Layout components
- Essential services (dialog, alert)
- Base styling and themes

**@taiga-ui/kit:**
- Extended component set
- Complex form controls
- Data display components
- Islands, tabs, accordions

**@taiga-ui/icons:**
- Icon set
- SVG-based icons

**@taiga-ui/addon-***:
- Optional feature packages
- Commerce, table, charts, etc.
- Can be excluded if not needed

### Q4: How do you implement responsive, mobile-friendly interfaces with Taiga UI?

**Answer:**

1. **Touch Gestures:**
```typescript
import { TuiSwipeService } from '@taiga-ui/cdk';

(tuiSwipe)="onSwipe($event)"
```

2. **Pull to Refresh:**
```html
<tui-pull-to-refresh
  [loading]="loading"
  (pulled)="onRefresh()">
  <!-- content -->
</tui-pull-to-refresh>
```

3. **Responsive Components:**
```html
<tui-island [hoverable]="!isMobile">
  <!-- Disable hover on mobile -->
</tui-island>
```

4. **Bottom Navigation:**
```html
<tui-tabs class="bottom-nav">
  <!-- Mobile-friendly bottom tabs -->
</tui-tabs>
```

5. **Viewport Service:**
```typescript
inject(TuiMobileDialogService)
```

### Q5: What are the performance optimization features in Taiga UI?

**Answer:**

1. **Tree-Shaking**: Modular architecture allows unused code elimination

2. **@tuiPure Decorator**: Memoizes pure function results
```typescript
@tuiPure
calculateTotal(items: Item[]): number {
  // Cached based on items reference
}
```

3. **Zone-Free Operations**: `tuiZonefree` for operations outside Angular zones

4. **Lazy Loading**: Components can be lazy-loaded

5. **Small Bundle Size**: Optimized component implementations

6. **CDK Utilities**: Low-level utilities don't carry component weight

7. **No Dependencies**: Minimal external dependencies

8. **OnPush Strategy**: Components use OnPush change detection

## Key Takeaways

1. **Lightweight**: Smaller bundle sizes through modular architecture
2. **Mobile-First**: Excellent touch and mobile support
3. **Design Tokens**: Comprehensive CSS custom property theming
4. **Performance**: Built-in performance optimizations
5. **Modular Packages**: Split into CDK, core, kit, and addons
6. **Tree-Shakeable**: Unused components excluded from bundle
7. **Developer Experience**: Great TypeScript support and API
8. **Modern Angular**: Uses latest Angular features
9. **Customizable**: Extensive customization options
10. **Form Components**: Rich set of form controls
11. **No External Dependencies**: Minimal dependencies
12. **OnPush Strategy**: Optimized change detection
13. **Polymorpheus**: Dynamic content rendering
14. **Active Development**: Regular updates from Tinkoff
15. **Production-Ready**: Used in large-scale fintech applications

## Resources

- **Official Documentation**: https://taiga-ui.dev/
- **GitHub Repository**: https://github.com/Tinkoff/taiga-ui
- **Stackblitz Examples**: https://stackblitz.com/@taiga-ui
- **Design Tokens**: https://taiga-ui.dev/theme
- **CDK Documentation**: https://taiga-ui.dev/cdk
- **Icon Set**: https://taiga-ui.dev/icons
- **Changelog**: https://github.com/Tinkoff/taiga-ui/releases
- **Contributing Guide**: https://github.com/Tinkoff/taiga-ui/blob/main/CONTRIBUTING.md
- **Discord Community**: https://discord.gg/taiga-ui
- **Tinkoff Open Source**: https://github.com/Tinkoff