# Clarity Design System

## The Idea

**In plain English:** Clarity Design System is a ready-made collection of buttons, tables, forms, and navigation menus built specifically for business software — instead of designing every screen from scratch, developers plug in these pre-built, professionally styled pieces. A "design system" is just a shared rulebook plus a toolkit that keeps an app looking and behaving consistently everywhere.

**Real-world analogy:** Think of a large hospital that orders all its furniture and equipment from one approved medical supplier catalog. Every room gets the same style of bed, the same type of whiteboard, and the same branded signage — so any nurse who transfers to a new ward instantly knows where everything is and how it works.
- The hospital catalog = Clarity Design System (the central source of approved components)
- Each furniture item in the catalog = a Clarity component (datagrid, wizard, button, alert)
- The consistent look across every ward = the shared visual style and behavior rules enforced by Clarity
- A nurse moving between wards = a developer moving between different parts of the same enterprise app

---

## Overview

Clarity Design System is an open-source design system developed by VMware that provides a comprehensive set of UX guidelines, Angular components, web components, and design resources for building enterprise applications. Known for its focus on enterprise use cases, data-intensive interfaces, and accessibility, Clarity excels at creating professional, consistent user experiences for complex business applications.

## Installation

### Prerequisites

```bash
# Required dependencies
npm install @angular/core @angular/common @angular/animations
```

### Installation Steps

```bash
# Install Clarity Angular and Core
npm install @cds/angular @cds/core @clr/angular @clr/ui

# Install Clarity icons
npm install @cds/city
```

### Angular Configuration

**angular.json:**
```json
{
  "projects": {
    "your-app": {
      "architect": {
        "build": {
          "options": {
            "styles": [
              "node_modules/@clr/ui/clr-ui.min.css",
              "node_modules/@cds/core/global.min.css",
              "node_modules/@cds/core/styles/theme.dark.min.css",
              "src/styles.css"
            ],
            "scripts": []
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
import { ClarityModule } from '@clr/angular';
import { CdsModule } from '@cds/angular';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    ClarityModule,
    CdsModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

## Core Concepts

### 1. Design Principles

Clarity follows four core principles:

- **Efficiency**: Streamline workflows with clear information hierarchy
- **Clarity**: Remove ambiguity through explicit communication
- **Scalability**: Design patterns that work across applications
- **Accessibility**: WCAG 2.1 AA compliant components

### 2. Component Architecture

Clarity provides two component libraries:

- **@clr/angular**: Traditional Angular components
- **@cds/angular**: Web Components wrapped for Angular (Core Components)

## Component Examples

### 1. Datagrid - Enterprise Data Display

The Datagrid is Clarity's flagship component for displaying and managing large datasets.

**Basic Datagrid:**
```typescript
import { Component } from '@angular/core';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
  status: 'active' | 'inactive';
}

@Component({
  selector: 'app-user-list',
  template: `
    <clr-datagrid>
      <clr-dg-column [clrDgField]="'id'">
        <ng-container *clrDgHideableColumn="{hidden: false}">
          ID
        </ng-container>
      </clr-dg-column>
      
      <clr-dg-column [clrDgField]="'name'">
        <ng-container *clrDgHideableColumn="{hidden: false}">
          Name
        </ng-container>
      </clr-dg-column>
      
      <clr-dg-column [clrDgField]="'email'">Email</clr-dg-column>
      
      <clr-dg-column [clrDgField]="'role'">Role</clr-dg-column>
      
      <clr-dg-column [clrDgField]="'status'">Status</clr-dg-column>

      <clr-dg-row *clrDgItems="let user of users" [clrDgItem]="user">
        <clr-dg-cell>{{user.id}}</clr-dg-cell>
        <clr-dg-cell>{{user.name}}</clr-dg-cell>
        <clr-dg-cell>{{user.email}}</clr-dg-cell>
        <clr-dg-cell>{{user.role}}</clr-dg-cell>
        <clr-dg-cell>
          <span class="label" [class.label-success]="user.status === 'active'"
                [class.label-danger]="user.status === 'inactive'">
            {{user.status}}
          </span>
        </clr-dg-cell>
      </clr-dg-row>

      <clr-dg-footer>
        <clr-dg-pagination #pagination [clrDgPageSize]="10">
          <clr-dg-page-size [clrPageSizeOptions]="[10,20,50,100]">
            Users per page
          </clr-dg-page-size>
          {{pagination.firstItem + 1}} - {{pagination.lastItem + 1}}
          of {{pagination.totalItems}} users
        </clr-dg-pagination>
      </clr-dg-footer>
    </clr-datagrid>
  `
})
export class UserListComponent {
  users: User[] = [
    { id: 1, name: 'John Doe', email: 'john@example.com', role: 'Admin', status: 'active' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'User', status: 'active' },
    { id: 3, name: 'Bob Johnson', email: 'bob@example.com', role: 'User', status: 'inactive' }
  ];
}
```

**Advanced Datagrid with Server-Side Operations:**
```typescript
import { Component } from '@angular/core';
import { ClrDatagridStateInterface } from '@clr/angular';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-server-side-grid',
  template: `
    <clr-datagrid (clrDgRefresh)="refresh($event)" [clrDgLoading]="loading">
      <clr-dg-column [clrDgField]="'name'">
        <clr-dg-filter [clrDgFilter]="nameFilter">
          <app-name-filter #nameFilter></app-name-filter>
        </clr-dg-filter>
        Name
      </clr-dg-column>
      
      <clr-dg-column [clrDgField]="'email'">Email</clr-dg-column>
      <clr-dg-column [clrDgField]="'department'">Department</clr-dg-column>
      
      <clr-dg-column [clrDgField]="'salary'">
        <clr-dg-filter [clrDgFilter]="salaryFilter">
          <app-salary-filter #salaryFilter></app-salary-filter>
        </clr-dg-filter>
        Salary
      </clr-dg-column>

      <clr-dg-row *clrDgItems="let employee of employees" [clrDgItem]="employee">
        <clr-dg-action-overflow>
          <button class="action-item" (click)="onEdit(employee)">Edit</button>
          <button class="action-item" (click)="onDelete(employee)">Delete</button>
        </clr-dg-action-overflow>
        
        <clr-dg-cell>{{employee.name}}</clr-dg-cell>
        <clr-dg-cell>{{employee.email}}</clr-dg-cell>
        <clr-dg-cell>{{employee.department}}</clr-dg-cell>
        <clr-dg-cell>{{employee.salary | currency}}</clr-dg-cell>
      </clr-dg-row>

      <clr-dg-footer>
        <clr-dg-pagination #pagination [clrDgTotalItems]="total">
          {{pagination.firstItem + 1}} - {{pagination.lastItem + 1}}
          of {{total}} employees
        </clr-dg-pagination>
      </clr-dg-footer>
    </clr-datagrid>
  `
})
export class ServerSideGridComponent {
  employees: any[] = [];
  total: number = 0;
  loading: boolean = true;

  constructor(private employeeService: EmployeeService) {}

  refresh(state: ClrDatagridStateInterface) {
    this.loading = true;
    
    // Build query parameters from state
    const filters = state.filters || [];
    const sort = state.sort;
    const page = state.page;

    const params = {
      page: page?.current || 1,
      size: page?.size || 10,
      sortBy: sort?.by as string,
      sortReverse: sort?.reverse || false,
      filters: this.buildFilters(filters)
    };

    this.employeeService.getEmployees(params).subscribe({
      next: (result) => {
        this.employees = result.data;
        this.total = result.total;
        this.loading = false;
      },
      error: () => {
        this.loading = false;
      }
    });
  }

  private buildFilters(filters: any[]): any {
    return filters.reduce((acc, filter) => {
      if (filter.property) {
        acc[filter.property] = filter.value;
      }
      return acc;
    }, {});
  }

  onEdit(employee: any) {
    console.log('Edit employee:', employee);
  }

  onDelete(employee: any) {
    console.log('Delete employee:', employee);
  }
}
```

### 2. Wizard - Multi-Step Workflows

Clarity wizards guide users through complex multi-step processes.

**Basic Wizard:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-onboarding-wizard',
  template: `
    <clr-wizard #wizard [(clrWizardOpen)]="open" 
                [clrWizardSize]="'lg'"
                (clrWizardOnFinish)="onFinish()"
                (clrWizardOnCancel)="onCancel()">
      
      <clr-wizard-title>User Onboarding</clr-wizard-title>

      <clr-wizard-button [type]="'cancel'">Cancel</clr-wizard-button>
      <clr-wizard-button [type]="'previous'">Back</clr-wizard-button>
      <clr-wizard-button [type]="'next'">Next</clr-wizard-button>
      <clr-wizard-button [type]="'finish'">Complete</clr-wizard-button>

      <!-- Step 1: Personal Information -->
      <clr-wizard-page [clrWizardPageNextDisabled]="!personalInfoValid">
        <ng-template clrPageTitle>Personal Information</ng-template>
        
        <form clrForm>
          <clr-input-container>
            <label>First Name</label>
            <input clrInput 
                   name="firstName" 
                   [(ngModel)]="formData.firstName"
                   (ngModelChange)="validatePersonalInfo()"
                   required />
          </clr-input-container>

          <clr-input-container>
            <label>Last Name</label>
            <input clrInput 
                   name="lastName" 
                   [(ngModel)]="formData.lastName"
                   (ngModelChange)="validatePersonalInfo()"
                   required />
          </clr-input-container>

          <clr-input-container>
            <label>Email</label>
            <input clrInput 
                   type="email"
                   name="email" 
                   [(ngModel)]="formData.email"
                   (ngModelChange)="validatePersonalInfo()"
                   required />
          </clr-input-container>
        </form>
      </clr-wizard-page>

      <!-- Step 2: Account Settings -->
      <clr-wizard-page [clrWizardPageNextDisabled]="!accountSettingsValid">
        <ng-template clrPageTitle>Account Settings</ng-template>
        
        <form clrForm>
          <clr-input-container>
            <label>Username</label>
            <input clrInput 
                   name="username" 
                   [(ngModel)]="formData.username"
                   (ngModelChange)="validateAccountSettings()"
                   required />
          </clr-input-container>

          <clr-password-container>
            <label>Password</label>
            <input clrPassword 
                   name="password" 
                   [(ngModel)]="formData.password"
                   (ngModelChange)="validateAccountSettings()"
                   required />
          </clr-password-container>

          <clr-select-container>
            <label>Role</label>
            <select clrSelect 
                    name="role" 
                    [(ngModel)]="formData.role"
                    (ngModelChange)="validateAccountSettings()"
                    required>
              <option value="">Select role</option>
              <option value="user">User</option>
              <option value="admin">Admin</option>
              <option value="manager">Manager</option>
            </select>
          </clr-select-container>
        </form>
      </clr-wizard-page>

      <!-- Step 3: Preferences -->
      <clr-wizard-page>
        <ng-template clrPageTitle>Preferences</ng-template>
        
        <form clrForm>
          <clr-checkbox-container>
            <label>Notifications</label>
            <clr-checkbox-wrapper>
              <input type="checkbox" 
                     clrCheckbox 
                     name="emailNotifications"
                     [(ngModel)]="formData.emailNotifications" />
              <label>Email notifications</label>
            </clr-checkbox-wrapper>
            <clr-checkbox-wrapper>
              <input type="checkbox" 
                     clrCheckbox 
                     name="smsNotifications"
                     [(ngModel)]="formData.smsNotifications" />
              <label>SMS notifications</label>
            </clr-checkbox-wrapper>
          </clr-checkbox-container>

          <clr-radio-container>
            <label>Theme</label>
            <clr-radio-wrapper>
              <input type="radio" 
                     clrRadio 
                     name="theme" 
                     value="light"
                     [(ngModel)]="formData.theme" />
              <label>Light</label>
            </clr-radio-wrapper>
            <clr-radio-wrapper>
              <input type="radio" 
                     clrRadio 
                     name="theme" 
                     value="dark"
                     [(ngModel)]="formData.theme" />
              <label>Dark</label>
            </clr-radio-wrapper>
          </clr-radio-container>
        </form>
      </clr-wizard-page>

      <!-- Step 4: Review -->
      <clr-wizard-page>
        <ng-template clrPageTitle>Review</ng-template>
        
        <div class="clr-row">
          <div class="clr-col-12">
            <h4>Please review your information</h4>
            
            <dl class="clr-row">
              <dt class="clr-col-4">Name:</dt>
              <dd class="clr-col-8">{{formData.firstName}} {{formData.lastName}}</dd>
              
              <dt class="clr-col-4">Email:</dt>
              <dd class="clr-col-8">{{formData.email}}</dd>
              
              <dt class="clr-col-4">Username:</dt>
              <dd class="clr-col-8">{{formData.username}}</dd>
              
              <dt class="clr-col-4">Role:</dt>
              <dd class="clr-col-8">{{formData.role}}</dd>
              
              <dt class="clr-col-4">Theme:</dt>
              <dd class="clr-col-8">{{formData.theme}}</dd>
            </dl>
          </div>
        </div>
      </clr-wizard-page>
    </clr-wizard>

    <button class="btn btn-primary" (click)="open = true">Start Onboarding</button>
  `
})
export class OnboardingWizardComponent {
  open: boolean = false;
  personalInfoValid: boolean = false;
  accountSettingsValid: boolean = false;

  formData = {
    firstName: '',
    lastName: '',
    email: '',
    username: '',
    password: '',
    role: '',
    emailNotifications: true,
    smsNotifications: false,
    theme: 'light'
  };

  validatePersonalInfo() {
    this.personalInfoValid = !!(
      this.formData.firstName &&
      this.formData.lastName &&
      this.formData.email &&
      this.isValidEmail(this.formData.email)
    );
  }

  validateAccountSettings() {
    this.accountSettingsValid = !!(
      this.formData.username &&
      this.formData.password &&
      this.formData.password.length >= 8 &&
      this.formData.role
    );
  }

  isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  onFinish() {
    console.log('Wizard completed:', this.formData);
    this.open = false;
  }

  onCancel() {
    console.log('Wizard cancelled');
    this.open = false;
  }
}
```

### 3. Forms - Enterprise Form Controls

Clarity provides comprehensive form components with built-in validation and accessibility.

**Advanced Form Example:**
```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-employee-form',
  template: `
    <form clrForm [formGroup]="employeeForm" (ngSubmit)="onSubmit()">
      <clr-input-container>
        <label class="clr-required-mark">Employee ID</label>
        <input clrInput 
               formControlName="employeeId" 
               placeholder="Enter employee ID" />
        <clr-control-error *clrIfError="'required'">
          Employee ID is required
        </clr-control-error>
        <clr-control-error *clrIfError="'pattern'">
          Employee ID must be in format EMP-XXXX
        </clr-control-error>
      </clr-input-container>

      <clr-input-container>
        <label class="clr-required-mark">Full Name</label>
        <input clrInput 
               formControlName="fullName" 
               placeholder="Enter full name" />
        <clr-control-error *clrIfError="'required'">
          Name is required
        </clr-control-error>
        <clr-control-error *clrIfError="'minlength'">
          Name must be at least 3 characters
        </clr-control-error>
      </clr-input-container>

      <clr-input-container>
        <label class="clr-required-mark">Email</label>
        <input clrInput 
               type="email"
               formControlName="email" 
               placeholder="email@example.com" />
        <clr-control-helper>Use company email</clr-control-helper>
        <clr-control-error *clrIfError="'required'">
          Email is required
        </clr-control-error>
        <clr-control-error *clrIfError="'email'">
          Invalid email format
        </clr-control-error>
      </clr-input-container>

      <clr-date-container>
        <label class="clr-required-mark">Start Date</label>
        <input clrDate 
               formControlName="startDate" 
               placeholder="MM/DD/YYYY" />
        <clr-control-error *clrIfError="'required'">
          Start date is required
        </clr-control-error>
      </clr-date-container>

      <clr-select-container>
        <label class="clr-required-mark">Department</label>
        <select clrSelect formControlName="department">
          <option value="">Select department</option>
          <option *ngFor="let dept of departments" [value]="dept.id">
            {{dept.name}}
          </option>
        </select>
        <clr-control-error *clrIfError="'required'">
          Department is required
        </clr-control-error>
      </clr-select-container>

      <clr-combobox-container>
        <label class="clr-required-mark">Skills</label>
        <clr-combobox 
          formControlName="skills" 
          name="skills"
          [clrMulti]="true">
          <clr-options>
            <clr-option *clrOptionItems="let skill of allSkills" 
                        [clrValue]="skill">
              {{skill}}
            </clr-option>
          </clr-options>
        </clr-combobox>
        <clr-control-helper>
          Select all applicable skills
        </clr-control-helper>
      </clr-combobox-container>

      <clr-textarea-container>
        <label>Notes</label>
        <textarea clrTextarea 
                  formControlName="notes"
                  rows="4"></textarea>
        <clr-control-helper>
          {{employeeForm.get('notes')?.value?.length || 0}} / 500 characters
        </clr-control-helper>
      </clr-textarea-container>

      <clr-checkbox-container>
        <clr-checkbox-wrapper>
          <input type="checkbox" 
                 clrCheckbox 
                 formControlName="isActive" />
          <label>Active Employee</label>
        </clr-checkbox-wrapper>
      </clr-checkbox-container>

      <div class="btn-group">
        <button type="submit" 
                class="btn btn-primary"
                [disabled]="!employeeForm.valid || submitting">
          <clr-icon shape="check" *ngIf="!submitting"></clr-icon>
          <clr-spinner clrSmall *ngIf="submitting"></clr-spinner>
          {{submitting ? 'Saving...' : 'Save Employee'}}
        </button>
        <button type="button" 
                class="btn btn-outline"
                (click)="onReset()">
          Reset
        </button>
      </div>
    </form>
  `
})
export class EmployeeFormComponent implements OnInit {
  employeeForm!: FormGroup;
  submitting = false;

  departments = [
    { id: 'eng', name: 'Engineering' },
    { id: 'sales', name: 'Sales' },
    { id: 'hr', name: 'Human Resources' },
    { id: 'marketing', name: 'Marketing' }
  ];

  allSkills = [
    'JavaScript', 'TypeScript', 'Angular', 'React', 
    'Node.js', 'Python', 'Java', 'AWS', 'Docker'
  ];

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.employeeForm = this.fb.group({
      employeeId: ['', [
        Validators.required,
        Validators.pattern(/^EMP-\d{4}$/)
      ]],
      fullName: ['', [
        Validators.required,
        Validators.minLength(3)
      ]],
      email: ['', [
        Validators.required,
        Validators.email
      ]],
      startDate: ['', Validators.required],
      department: ['', Validators.required],
      skills: [[], Validators.required],
      notes: ['', Validators.maxLength(500)],
      isActive: [true]
    });
  }

  onSubmit() {
    if (this.employeeForm.valid) {
      this.submitting = true;
      console.log('Form submitted:', this.employeeForm.value);
      
      // Simulate API call
      setTimeout(() => {
        this.submitting = false;
        this.employeeForm.reset({ isActive: true });
      }, 2000);
    }
  }

  onReset() {
    this.employeeForm.reset({ isActive: true });
  }
}
```

### 4. Navigation and Layout

**Application Layout with Navigation:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-main-layout',
  template: `
    <clr-main-container>
      <!-- Header -->
      <clr-header class="header-6">
        <div class="branding">
          <a href="/" class="nav-link">
            <clr-icon shape="vm-bug"></clr-icon>
            <span class="title">Enterprise App</span>
          </a>
        </div>

        <div class="header-nav" [clr-nav-level]="1">
          <a class="nav-link" 
             routerLink="/dashboard" 
             routerLinkActive="active">
            <span class="nav-text">Dashboard</span>
          </a>
          <a class="nav-link" 
             routerLink="/users" 
             routerLinkActive="active">
            <span class="nav-text">Users</span>
          </a>
          <a class="nav-link" 
             routerLink="/reports" 
             routerLinkActive="active">
            <span class="nav-text">Reports</span>
          </a>
        </div>

        <div class="header-actions">
          <!-- Search -->
          <clr-dropdown>
            <button class="nav-icon" clrDropdownTrigger>
              <clr-icon shape="search"></clr-icon>
            </button>
            <clr-dropdown-menu clrPosition="bottom-right" *clrIfOpen>
              <input type="text" 
                     placeholder="Search..." 
                     class="clr-input" />
            </clr-dropdown-menu>
          </clr-dropdown>

          <!-- Notifications -->
          <clr-dropdown>
            <button class="nav-icon" clrDropdownTrigger>
              <clr-icon shape="bell"></clr-icon>
              <span class="badge">3</span>
            </button>
            <clr-dropdown-menu clrPosition="bottom-right" *clrIfOpen>
              <div class="dropdown-header">Notifications</div>
              <a clrDropdownItem>New user registered</a>
              <a clrDropdownItem>Report generated</a>
              <a clrDropdownItem>System update available</a>
            </clr-dropdown-menu>
          </clr-dropdown>

          <!-- User Menu -->
          <clr-dropdown>
            <button class="nav-icon" clrDropdownTrigger>
              <clr-icon shape="user"></clr-icon>
            </button>
            <clr-dropdown-menu clrPosition="bottom-right" *clrIfOpen>
              <div class="dropdown-header">{{currentUser?.name}}</div>
              <a clrDropdownItem routerLink="/profile">Profile</a>
              <a clrDropdownItem routerLink="/settings">Settings</a>
              <div class="dropdown-divider"></div>
              <a clrDropdownItem (click)="onLogout()">Logout</a>
            </clr-dropdown-menu>
          </clr-dropdown>
        </div>
      </clr-header>

      <!-- Sidebar Navigation -->
      <nav class="subnav" [clr-nav-level]="2">
        <ul class="nav">
          <li class="nav-item">
            <a class="nav-link" 
               routerLink="/dashboard/overview" 
               routerLinkActive="active">
              Overview
            </a>
          </li>
          <li class="nav-item">
            <a class="nav-link" 
               routerLink="/dashboard/analytics" 
               routerLinkActive="active">
              Analytics
            </a>
          </li>
          <li class="nav-item">
            <a class="nav-link" 
               routerLink="/dashboard/reports" 
               routerLinkActive="active">
              Reports
            </a>
          </li>
        </ul>
      </nav>

      <!-- Main Content Area -->
      <div class="content-container">
        <div class="content-area">
          <router-outlet></router-outlet>
        </div>

        <!-- Optional Sidenav -->
        <clr-vertical-nav 
          [clrVerticalNavCollapsible]="true"
          [(clrVerticalNavCollapsed)]="collapsed">
          
          <a clrVerticalNavLink 
             routerLink="/dashboard" 
             routerLinkActive="active">
            <clr-icon clrVerticalNavIcon shape="dashboard"></clr-icon>
            Dashboard
          </a>

          <clr-vertical-nav-group routerLinkActive="active">
            <clr-icon clrVerticalNavIcon shape="users"></clr-icon>
            Users
            <clr-vertical-nav-group-children>
              <a clrVerticalNavLink 
                 routerLink="/users/list" 
                 routerLinkActive="active">
                User List
              </a>
              <a clrVerticalNavLink 
                 routerLink="/users/roles" 
                 routerLinkActive="active">
                Roles & Permissions
              </a>
            </clr-vertical-nav-group-children>
          </clr-vertical-nav-group>

          <a clrVerticalNavLink 
             routerLink="/settings" 
             routerLinkActive="active">
            <clr-icon clrVerticalNavIcon shape="cog"></clr-icon>
            Settings
          </a>
        </clr-vertical-nav>
      </div>
    </clr-main-container>
  `,
  styles: [`
    .badge {
      position: absolute;
      top: 8px;
      right: 8px;
      background: #c92100;
      color: white;
      border-radius: 50%;
      padding: 2px 6px;
      font-size: 10px;
    }
  `]
})
export class MainLayoutComponent {
  collapsed = false;
  currentUser = { name: 'John Doe' };

  onLogout() {
    console.log('Logging out...');
  }
}
```

### 5. Alerts and Notifications

**Alert Management System:**
```typescript
import { Component } from '@angular/core';

interface Alert {
  id: number;
  type: 'info' | 'success' | 'warning' | 'danger';
  message: string;
  closeable: boolean;
}

@Component({
  selector: 'app-alert-manager',
  template: `
    <div class="alert-container">
      <!-- Standard Alerts -->
      <clr-alert 
        *ngFor="let alert of alerts"
        [clrAlertType]="alert.type"
        [clrAlertClosable]="alert.closeable"
        (clrAlertClosedChange)="onAlertClose(alert.id)">
        <clr-alert-item>
          <span class="alert-text">{{alert.message}}</span>
        </clr-alert-item>
      </clr-alert>

      <!-- App-Level Alert -->
      <clr-alert 
        [clrAlertType]="'warning'"
        [clrAlertAppLevel]="true"
        [(clrAlertClosed)]="appLevelClosed">
        <clr-alert-item>
          <span class="alert-text">
            System maintenance scheduled for tonight at 11 PM
          </span>
          <div class="alert-actions">
            <button class="btn btn-sm btn-link" 
                    (click)="onLearnMore()">
              Learn More
            </button>
          </div>
        </clr-alert-item>
      </clr-alert>

      <!-- Alert with Multiple Items -->
      <clr-alert 
        [clrAlertType]="'danger'"
        *ngIf="validationErrors.length > 0">
        <clr-alert-item *ngFor="let error of validationErrors">
          <span class="alert-text">{{error}}</span>
        </clr-alert-item>
      </clr-alert>

      <!-- Alert with Actions -->
      <clr-alert 
        [clrAlertType]="'info'"
        [(clrAlertClosed)]="updateAvailableClosed">
        <clr-alert-item>
          <span class="alert-text">
            A new version is available
          </span>
          <div class="alert-actions">
            <button class="btn btn-sm btn-primary" 
                    (click)="onUpdate()">
              Update Now
            </button>
            <button class="btn btn-sm btn-link" 
                    (click)="updateAvailableClosed = true">
              Dismiss
            </button>
          </div>
        </clr-alert-item>
      </clr-alert>
    </div>

    <!-- Alert Triggers -->
    <div class="btn-group">
      <button class="btn btn-primary" (click)="showSuccess()">
        Success
      </button>
      <button class="btn btn-warning" (click)="showWarning()">
        Warning
      </button>
      <button class="btn btn-danger" (click)="showError()">
        Error
      </button>
      <button class="btn btn-info" (click)="showInfo()">
        Info
      </button>
    </div>
  `
})
export class AlertManagerComponent {
  alerts: Alert[] = [];
  validationErrors: string[] = [];
  appLevelClosed = false;
  updateAvailableClosed = false;
  private alertIdCounter = 0;

  showSuccess() {
    this.addAlert('success', 'Operation completed successfully!', true);
  }

  showWarning() {
    this.addAlert('warning', 'This action cannot be undone.', true);
  }

  showError() {
    this.addAlert('danger', 'An error occurred while processing your request.', true);
  }

  showInfo() {
    this.addAlert('info', 'New features are now available.', true);
  }

  private addAlert(
    type: 'info' | 'success' | 'warning' | 'danger',
    message: string,
    closeable: boolean
  ) {
    this.alerts.push({
      id: ++this.alertIdCounter,
      type,
      message,
      closeable
    });

    // Auto-dismiss after 5 seconds
    if (closeable) {
      setTimeout(() => {
        this.onAlertClose(this.alertIdCounter);
      }, 5000);
    }
  }

  onAlertClose(id: number) {
    this.alerts = this.alerts.filter(alert => alert.id !== id);
  }

  onLearnMore() {
    console.log('Navigate to maintenance details');
  }

  onUpdate() {
    console.log('Starting update...');
    this.updateAvailableClosed = true;
  }
}
```

### 6. Modal Dialogs

**Modal Service with Different Sizes:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-modal-demo',
  template: `
    <!-- Small Modal -->
    <clr-modal [(clrModalOpen)]="smallModalOpen" [clrModalSize]="'sm'">
      <h3 class="modal-title">Confirm Action</h3>
      <div class="modal-body">
        <p>Are you sure you want to proceed?</p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-outline" (click)="smallModalOpen = false">
          Cancel
        </button>
        <button type="button" class="btn btn-primary" (click)="onConfirm()">
          Confirm
        </button>
      </div>
    </clr-modal>

    <!-- Large Modal with Form -->
    <clr-modal [(clrModalOpen)]="largeModalOpen" 
               [clrModalSize]="'lg'"
               [clrModalClosable]="false">
      <h3 class="modal-title">Create New Project</h3>
      <div class="modal-body">
        <form clrForm [formGroup]="projectForm">
          <clr-input-container>
            <label>Project Name</label>
            <input clrInput formControlName="name" />
          </clr-input-container>

          <clr-textarea-container>
            <label>Description</label>
            <textarea clrTextarea formControlName="description"></textarea>
          </clr-textarea-container>

          <clr-select-container>
            <label>Priority</label>
            <select clrSelect formControlName="priority">
              <option value="low">Low</option>
              <option value="medium">Medium</option>
              <option value="high">High</option>
            </select>
          </clr-select-container>
        </form>
      </div>
      <div class="modal-footer">
        <button type="button" 
                class="btn btn-outline" 
                (click)="onCancelProject()">
          Cancel
        </button>
        <button type="button" 
                class="btn btn-primary" 
                [disabled]="!projectForm.valid"
                (click)="onCreateProject()">
          Create
        </button>
      </div>
    </clr-modal>

    <!-- Full Page Modal -->
    <clr-modal [(clrModalOpen)]="fullPageModalOpen" [clrModalSize]="'xl'">
      <h3 class="modal-title">Document Viewer</h3>
      <div class="modal-body" style="height: 70vh; overflow-y: auto;">
        <div class="clr-row">
          <div class="clr-col-3">
            <nav class="sidenav">
              <section class="sidenav-content">
                <a class="nav-link active">Chapter 1</a>
                <a class="nav-link">Chapter 2</a>
                <a class="nav-link">Chapter 3</a>
              </section>
            </nav>
          </div>
          <div class="clr-col-9">
            <h4>Document Content</h4>
            <p>Lorem ipsum dolor sit amet...</p>
          </div>
        </div>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-primary" (click)="fullPageModalOpen = false">
          Close
        </button>
      </div>
    </clr-modal>

    <!-- Static Backdrop Modal -->
    <clr-modal [(clrModalOpen)]="staticModalOpen" 
               [clrModalStaticBackdrop]="true">
      <h3 class="modal-title">Unsaved Changes</h3>
      <div class="modal-body">
        <clr-alert [clrAlertType]="'warning'" [clrAlertClosable]="false">
          <clr-alert-item>
            <span class="alert-text">
              You have unsaved changes. Click outside will not close this dialog.
            </span>
          </clr-alert-item>
        </clr-alert>
        <p>Would you like to save your changes before leaving?</p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-outline" (click)="onDiscard()">
          Discard
        </button>
        <button type="button" class="btn btn-outline" (click)="staticModalOpen = false">
          Cancel
        </button>
        <button type="button" class="btn btn-primary" (click)="onSave()">
          Save Changes
        </button>
      </div>
    </clr-modal>

    <!-- Trigger Buttons -->
    <div class="btn-group">
      <button class="btn btn-primary" (click)="smallModalOpen = true">
        Small Modal
      </button>
      <button class="btn btn-primary" (click)="largeModalOpen = true">
        Large Modal
      </button>
      <button class="btn btn-primary" (click)="fullPageModalOpen = true">
        Full Page Modal
      </button>
      <button class="btn btn-primary" (click)="staticModalOpen = true">
        Static Modal
      </button>
    </div>
  `
})
export class ModalDemoComponent {
  smallModalOpen = false;
  largeModalOpen = false;
  fullPageModalOpen = false;
  staticModalOpen = false;

  projectForm = this.fb.group({
    name: ['', Validators.required],
    description: [''],
    priority: ['medium']
  });

  constructor(private fb: FormBuilder) {}

  onConfirm() {
    console.log('Confirmed');
    this.smallModalOpen = false;
  }

  onCreateProject() {
    if (this.projectForm.valid) {
      console.log('Creating project:', this.projectForm.value);
      this.largeModalOpen = false;
      this.projectForm.reset({ priority: 'medium' });
    }
  }

  onCancelProject() {
    this.largeModalOpen = false;
    this.projectForm.reset({ priority: 'medium' });
  }

  onSave() {
    console.log('Saving changes...');
    this.staticModalOpen = false;
  }

  onDiscard() {
    console.log('Discarding changes...');
    this.staticModalOpen = false;
  }
}
```

### 7. Cards and Layout

**Dashboard with Cards:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-dashboard',
  template: `
    <div class="clr-row">
      <!-- Metric Cards -->
      <div class="clr-col-lg-3 clr-col-md-6 clr-col-12">
        <div class="card">
          <div class="card-header">
            Total Users
          </div>
          <div class="card-block">
            <h1>12,453</h1>
            <p class="card-text">
              <clr-icon shape="arrow-up" class="is-success"></clr-icon>
              <span class="text-success">5.2%</span> from last month
            </p>
          </div>
        </div>
      </div>

      <div class="clr-col-lg-3 clr-col-md-6 clr-col-12">
        <div class="card">
          <div class="card-header">
            Active Sessions
          </div>
          <div class="card-block">
            <h1>1,847</h1>
            <p class="card-text">
              <clr-icon shape="arrow-down" class="is-danger"></clr-icon>
              <span class="text-danger">2.1%</span> from yesterday
            </p>
          </div>
        </div>
      </div>

      <!-- Chart Card -->
      <div class="clr-col-lg-6 clr-col-12">
        <div class="card">
          <div class="card-header">
            Revenue Trend
            <div class="card-header-actions">
              <clr-dropdown>
                <button class="btn btn-sm btn-link" clrDropdownTrigger>
                  <clr-icon shape="ellipsis-vertical"></clr-icon>
                </button>
                <clr-dropdown-menu *clrIfOpen clrPosition="bottom-right">
                  <a clrDropdownItem>Export</a>
                  <a clrDropdownItem>Refresh</a>
                </clr-dropdown-menu>
              </clr-dropdown>
            </div>
          </div>
          <div class="card-block">
            <div class="card-text">
              <!-- Chart component would go here -->
              <p>Chart placeholder (integrate with your chart library)</p>
            </div>
          </div>
        </div>
      </div>

      <!-- List Card -->
      <div class="clr-col-lg-4 clr-col-12">
        <div class="card">
          <div class="card-header">
            Recent Activities
          </div>
          <ul class="list">
            <li *ngFor="let activity of recentActivities">
              <span class="badge badge-info">{{activity.type}}</span>
              {{activity.description}}
              <span class="label label-light-gray">{{activity.time}}</span>
            </li>
          </ul>
        </div>
      </div>

      <!-- Media Card -->
      <div class="clr-col-lg-4 clr-col-12">
        <div class="card">
          <div class="card-img">
            <img src="placeholder.jpg" alt="Card image" />
          </div>
          <div class="card-block">
            <h4 class="card-title">Feature Announcement</h4>
            <p class="card-text">
              Check out our new dashboard features that will help you...
            </p>
          </div>
          <div class="card-footer">
            <button class="btn btn-sm btn-link">Learn More</button>
          </div>
        </div>
      </div>

      <!-- Clickable Card -->
      <div class="clr-col-lg-4 clr-col-12">
        <div class="card clickable" (click)="onCardClick()">
          <div class="card-block">
            <div class="card-media-block">
              <clr-icon shape="folder" size="48"></clr-icon>
              <div class="card-media-description">
                <h4 class="card-title">Project Files</h4>
                <p class="card-text">
                  125 files, 2.3 GB
                </p>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .card {
      margin-bottom: 24px;
    }
    
    .text-success {
      color: #62a420;
    }
    
    .text-danger {
      color: #c92100;
    }
    
    .is-success {
      color: #62a420;
    }
    
    .is-danger {
      color: #c92100;
    }
  `]
})
export class DashboardComponent {
  recentActivities = [
    { type: 'Login', description: 'User logged in', time: '2m ago' },
    { type: 'Create', description: 'New project created', time: '15m ago' },
    { type: 'Update', description: 'Profile updated', time: '1h ago' }
  ];

  onCardClick() {
    console.log('Card clicked');
  }
}
```

## Common Mistakes

### 1. Not Importing ClarityModule

**Wrong:**
```typescript
// Missing ClarityModule import
@NgModule({
  imports: [BrowserModule]
})
export class AppModule { }
```

**Correct:**
```typescript
import { ClarityModule } from '@clr/angular';

@NgModule({
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    ClarityModule
  ]
})
export class AppModule { }
```

### 2. Forgetting Animations Module

**Wrong:**
```typescript
// Missing BrowserAnimationsModule
@NgModule({
  imports: [BrowserModule, ClarityModule]
})
export class AppModule { }
```

**Correct:**
```typescript
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';

@NgModule({
  imports: [
    BrowserModule,
    BrowserAnimationsModule, // Required for Clarity
    ClarityModule
  ]
})
export class AppModule { }
```

### 3. Incorrect Datagrid State Handling

**Wrong:**
```typescript
// Not handling loading state properly
refresh(state: ClrDatagridStateInterface) {
  this.service.getData().subscribe(data => {
    this.data = data;
    // Forgot to set loading = false
  });
}
```

**Correct:**
```typescript
refresh(state: ClrDatagridStateInterface) {
  this.loading = true;
  this.service.getData(state).subscribe({
    next: (data) => {
      this.data = data;
      this.loading = false;
    },
    error: () => {
      this.loading = false;
    }
  });
}
```

### 4. Missing CSS Imports

**Wrong:**
```typescript
// Not including Clarity CSS files
```

**Correct:**
```json
// angular.json
"styles": [
  "node_modules/@clr/ui/clr-ui.min.css",
  "node_modules/@cds/core/global.min.css",
  "src/styles.css"
]
```

## Best Practices

### 1. Use Lazy Loading for Large Forms

```typescript
// Load Clarity forms module lazily
const routes: Routes = [
  {
    path: 'forms',
    loadChildren: () => import('./forms/forms.module').then(m => m.FormsModule)
  }
];
```

### 2. Implement Custom Filters

```typescript
import { ClrDatagridFilterInterface } from '@clr/angular';
import { Subject } from 'rxjs';

class CustomFilter implements ClrDatagridFilterInterface<any> {
  changes = new Subject<any>();
  
  isActive(): boolean {
    return this.value !== undefined;
  }
  
  accepts(item: any): boolean {
    return item.field.includes(this.value);
  }
  
  private _value: string;
  
  get value(): string {
    return this._value;
  }
  
  set value(val: string) {
    this._value = val;
    this.changes.next(val);
  }
}
```

### 3. Optimize Datagrid Performance

```typescript
// Use trackBy for better performance
<clr-dg-row *clrDgItems="let item of items; trackBy: trackByFn">
  <!-- content -->
</clr-dg-row>

trackByFn(index: number, item: any): any {
  return item.id;
}
```

### 4. Implement Proper Error Handling

```typescript
@Component({
  template: `
    <clr-input-container>
      <label>Email</label>
      <input clrInput formControlName="email" />
      <clr-control-error *clrIfError="'required'">
        Email is required
      </clr-control-error>
      <clr-control-error *clrIfError="'email'">
        Invalid email format
      </clr-control-error>
    </clr-input-container>
  `
})
class FormComponent { }
```

### 5. Use Theming System

```css
/* Custom theme variables */
:root {
  --clr-global-app-background: #fafafa;
  --clr-btn-primary-bg-color: #0072a3;
  --clr-btn-primary-hover-bg-color: #00597c;
}
```

## When to Use Clarity Design

### Use When:

- Building enterprise applications with complex data management
- Need comprehensive data grid functionality
- Require consistent VMware-aligned design language
- Building applications with wizards and multi-step processes
- Need strong accessibility compliance (WCAG 2.1 AA)
- Working on applications with heavy form requirements

### Avoid When:

- Building consumer-facing applications with custom branding
- Need extremely lightweight bundle size
- Require highly animated, modern UI components
- Building mobile-first applications (Clarity is desktop-focused)
- Need extensive chart/visualization components

## Interview Questions

### Q1: What makes Clarity Design System particularly suitable for enterprise applications?

**Answer:** Clarity is designed specifically for enterprise use cases with features like:

- **Advanced Datagrid**: Handles large datasets with server-side pagination, sorting, filtering, and column management
- **Wizard Component**: Guides users through complex multi-step workflows with validation
- **Form Components**: Comprehensive form controls with built-in validation and accessibility
- **Enterprise Patterns**: Navigation, layouts, and patterns optimized for business applications
- **Accessibility**: WCAG 2.1 AA compliant out of the box
- **VMware Design Language**: Consistent with enterprise software standards

### Q2: How do you implement server-side pagination in Clarity Datagrid?

**Answer:** Server-side pagination requires handling the `clrDgRefresh` event and managing state:

```typescript
refresh(state: ClrDatagridStateInterface) {
  this.loading = true;
  
  const params = {
    page: state.page?.current || 1,
    size: state.page?.size || 10,
    sortBy: state.sort?.by,
    reverse: state.sort?.reverse,
    filters: this.buildFilters(state.filters)
  };
  
  this.service.getData(params).subscribe({
    next: (result) => {
      this.items = result.data;
      this.total = result.total;
      this.loading = false;
    }
  });
}
```

Key points:
- Set `[clrDgLoading]` to show loading indicator
- Use `[clrDgTotalItems]` to bind total count
- Handle `clrDgRefresh` event for state changes
- Parse `ClrDatagridStateInterface` for pagination, sorting, and filtering

### Q3: Explain the difference between @clr/angular and @cds/angular packages.

**Answer:**

**@clr/angular (Traditional Clarity):**
- Traditional Angular components using Angular's component model
- Mature, feature-complete library
- Desktop-focused enterprise components
- Includes Datagrid, Wizard, forms, navigation

**@cds/angular (Core Components):**
- Web Components wrapped for Angular
- Modern, framework-agnostic architecture
- Better performance and smaller bundle sizes
- New component designs following latest standards
- Can be used alongside @clr/angular

Both packages can coexist, with Core Components being the future direction while traditional Clarity maintains existing enterprise features.

### Q4: How do you create a custom validator for Clarity forms?

**Answer:**

```typescript
// Custom validator function
function employeeIdValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value;
    if (!value) return null;
    
    const valid = /^EMP-\d{4}$/.test(value);
    return valid ? null : { invalidEmployeeId: { value } };
  };
}

// Usage in component
@Component({
  template: `
    <clr-input-container>
      <label>Employee ID</label>
      <input clrInput formControlName="employeeId" />
      <clr-control-error *clrIfError="'required'">
        Required
      </clr-control-error>
      <clr-control-error *clrIfError="'invalidEmployeeId'">
        Must be in format EMP-XXXX
      </clr-control-error>
    </clr-input-container>
  `
})
class FormComponent {
  form = this.fb.group({
    employeeId: ['', [Validators.required, employeeIdValidator()]]
  });
}
```

### Q5: What are the key considerations for optimizing Clarity Datagrid performance with large datasets?

**Answer:**

1. **Server-Side Operations**: Move pagination, sorting, and filtering to backend
2. **Virtual Scrolling**: Use for extremely large datasets
3. **TrackBy Function**: Implement trackBy for efficient DOM updates
4. **Lazy Loading**: Load data on-demand rather than all at once
5. **Hideable Columns**: Allow users to hide unused columns
6. **Debounce Filters**: Debounce filter inputs to reduce API calls
7. **Smart Refresh**: Only refresh when necessary

```typescript
// Example with optimization
@Component({
  template: `
    <clr-datagrid 
      (clrDgRefresh)="refresh($event)"
      [clrDgLoading]="loading">
      <clr-dg-row *clrDgItems="let item of items; trackBy: trackById">
        <!-- columns -->
      </clr-dg-row>
    </clr-datagrid>
  `
})
class OptimizedGrid {
  trackById(index: number, item: any) {
    return item.id; // Use unique identifier
  }
  
  refresh = debounce((state: ClrDatagridStateInterface) => {
    // Server-side data fetching
  }, 300);
}
```

## Key Takeaways

1. **Enterprise Focus**: Clarity is specifically designed for enterprise applications with complex data management needs
2. **Datagrid Excellence**: The Datagrid component is one of the most powerful in the ecosystem for handling large datasets
3. **Wizard Component**: Built-in wizard component makes multi-step workflows easy to implement
4. **Accessibility First**: WCAG 2.1 AA compliance built into all components
5. **VMware Design Language**: Follows established enterprise design patterns and standards
6. **Two Component Libraries**: @clr/angular (traditional) and @cds/angular (modern web components)
7. **Form Components**: Comprehensive form controls with built-in validation and error handling
8. **Server-Side Support**: Built-in support for server-side pagination, sorting, and filtering
9. **Navigation Patterns**: Multiple navigation patterns (header, subnav, vertical nav) for complex applications
10. **Extensible**: Components can be customized and extended for specific needs
11. **Performance**: Optimized for handling large amounts of data efficiently
12. **Documentation**: Extensive documentation and examples from VMware
13. **Community**: Strong enterprise community and support
14. **Theming**: CSS custom properties for theme customization
15. **Future-Proof**: Active development with focus on web standards and modern architecture

## Resources

- **Official Documentation**: https://clarity.design/
- **GitHub Repository**: https://github.com/vmware-clarity/ng-clarity
- **Core Components**: https://core.clarity.design/
- **Storybook**: https://clarity.design/storybook/core/
- **Design Files**: https://clarity.design/get-started/design/
- **Community**: https://github.com/vmware-clarity/ng-clarity/discussions
- **Migration Guide**: https://clarity.design/documentation/migration
- **Angular CLI Integration**: https://clarity.design/get-started/developing/angular/
- **Accessibility Guidelines**: https://clarity.design/foundation/accessibility/
- **VMware Design**: https://vmware.github.io/clarity/
