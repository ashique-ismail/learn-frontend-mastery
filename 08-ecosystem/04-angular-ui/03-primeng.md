# PrimeNG

## The Idea

**In plain English:** PrimeNG is a ready-made collection of over 90 polished website building blocks (like buttons, tables, and date pickers) that you can drop straight into an Angular app — think of it as a giant LEGO set for web pages, where each piece already looks professional and works out of the box.

**Real-world analogy:** Imagine walking into a fully stocked kitchen supply store where every tool — pots, pans, spatulas, timers — is already designed, labelled, and ready to cook with, instead of you having to forge each utensil yourself.

- The store = PrimeNG (the library that holds all the ready-made pieces)
- Each kitchen tool = a UI component (a button, a table, a calendar widget, etc.)
- You picking a tool off the shelf and using it = importing and adding a component tag to your Angular template

---

## Overview

PrimeNG is a comprehensive UI component library for Angular applications, offering over 90 rich components with a focus on enterprise-grade features, extensive customization, and professional design. Developed by PrimeTek, it provides data tables, charts, forms, overlays, menus, and more with built-in theming, accessibility support, and responsive design patterns.

PrimeNG is widely adopted in enterprise applications for its feature-rich components, especially the advanced DataTable with filtering, sorting, pagination, lazy loading, and export capabilities. It also includes PrimeFlex for utility CSS and PrimeIcons for icons.

## Installation and Setup

```bash
npm install primeng primeicons
```

### Basic Configuration

```typescript
// app.config.ts (Standalone)
import { ApplicationConfig } from '@angular/core';
import { provideAnimations } from '@angular/platform-browser/animations';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations()
  ]
};
```

```scss
// styles.scss
@import "primeng/resources/themes/lara-light-blue/theme.css";
@import "primeng/resources/primeng.css";
@import "primeicons/primeicons.css";
```

### Module-based Setup

```typescript
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';

@NgModule({
  imports: [BrowserAnimationsModule]
})
export class AppModule {}
```

## Core Concepts

### 1. Basic Components

```typescript
import { Component } from '@angular/core';
import { ButtonModule } from 'primeng/button';
import { CardModule } from 'primeng/card';
import { InputTextModule } from 'primeng/inputtext';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-basic-example',
  standalone: true,
  imports: [
    ButtonModule,
    CardModule,
    InputTextModule,
    FormsModule
  ],
  template: `
    <p-card header="Welcome to PrimeNG" subheader="Enterprise UI Components">
      <ng-template pTemplate="header">
        <img alt="Card" src="assets/card-header.jpg" />
      </ng-template>
      
      <p>
        PrimeNG provides a comprehensive suite of UI components
        for Angular applications.
      </p>
      
      <ng-template pTemplate="footer">
        <p-button label="Save" icon="pi pi-check"></p-button>
        <p-button
          label="Cancel"
          icon="pi pi-times"
          styleClass="p-button-secondary"
        ></p-button>
      </ng-template>
    </p-card>

    <div class="p-field p-mt-3">
      <label for="username">Username</label>
      <input
        pInputText
        id="username"
        [(ngModel)]="username"
        placeholder="Enter username"
      />
    </div>
  `
})
export class BasicExampleComponent {
  username: string = '';
}
```

### 2. Advanced DataTable

```typescript
import { Component, OnInit } from '@angular/core';
import { TableModule } from 'primeng/table';
import { ButtonModule } from 'primeng/button';
import { InputTextModule } from 'primeng/inputtext';
import { DropdownModule } from 'primeng/dropdown';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';

interface Product {
  id: string;
  code: string;
  name: string;
  category: string;
  quantity: number;
  price: number;
  inventoryStatus: string;
}

@Component({
  selector: 'app-datatable-example',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule,
    TableModule,
    ButtonModule,
    InputTextModule,
    DropdownModule
  ],
  template: `
    <p-table
      [value]="products"
      [paginator]="true"
      [rows]="10"
      [rowsPerPageOptions]="[5, 10, 20]"
      [globalFilterFields]="['name', 'category', 'code']"
      [tableStyle]="{ 'min-width': '50rem' }"
      [resizableColumns]="true"
      [reorderableColumns]="true"
      [selectionMode]="'multiple'"
      [(selection)]="selectedProducts"
      dataKey="id"
    >
      <ng-template pTemplate="caption">
        <div class="flex">
          <span class="p-input-icon-left ml-auto">
            <i class="pi pi-search"></i>
            <input
              pInputText
              type="text"
              (input)="onGlobalFilter($event)"
              placeholder="Search keyword"
            />
          </span>
        </div>
      </ng-template>

      <ng-template pTemplate="header">
        <tr>
          <th style="width: 4rem">
            <p-tableHeaderCheckbox></p-tableHeaderCheckbox>
          </th>
          <th pSortableColumn="code" pReorderableColumn>
            Code <p-sortIcon field="code"></p-sortIcon>
          </th>
          <th pSortableColumn="name" pReorderableColumn>
            Name <p-sortIcon field="name"></p-sortIcon>
          </th>
          <th pSortableColumn="category" pReorderableColumn>
            Category <p-sortIcon field="category"></p-sortIcon>
          </th>
          <th pSortableColumn="quantity" pReorderableColumn>
            Quantity <p-sortIcon field="quantity"></p-sortIcon>
          </th>
          <th pSortableColumn="price" pReorderableColumn>
            Price <p-sortIcon field="price"></p-sortIcon>
          </th>
          <th>Status</th>
          <th>Actions</th>
        </tr>
        <tr>
          <th></th>
          <th>
            <input
              pInputText
              type="text"
              (input)="onFilter($event, 'code', 'contains')"
              placeholder="Search by code"
            />
          </th>
          <th>
            <input
              pInputText
              type="text"
              (input)="onFilter($event, 'name', 'contains')"
              placeholder="Search by name"
            />
          </th>
          <th>
            <p-dropdown
              [options]="categories"
              (onChange)="onFilter($event, 'category', 'equals')"
              placeholder="Select"
              [showClear]="true"
            ></p-dropdown>
          </th>
          <th></th>
          <th></th>
          <th></th>
          <th></th>
        </tr>
      </ng-template>

      <ng-template pTemplate="body" let-product>
        <tr>
          <td>
            <p-tableCheckbox [value]="product"></p-tableCheckbox>
          </td>
          <td>{{ product.code }}</td>
          <td>{{ product.name }}</td>
          <td>{{ product.category }}</td>
          <td>{{ product.quantity }}</td>
          <td>{{ product.price | currency }}</td>
          <td>
            <span
              [class]="'badge badge-' + product.inventoryStatus.toLowerCase()"
            >
              {{ product.inventoryStatus }}
            </span>
          </td>
          <td>
            <p-button
              icon="pi pi-pencil"
              styleClass="p-button-rounded p-button-success p-mr-2"
              (click)="editProduct(product)"
            ></p-button>
            <p-button
              icon="pi pi-trash"
              styleClass="p-button-rounded p-button-warning"
              (click)="deleteProduct(product)"
            ></p-button>
          </td>
        </tr>
      </ng-template>

      <ng-template pTemplate="emptymessage">
        <tr>
          <td colspan="8">No products found.</td>
        </tr>
      </ng-template>
    </p-table>
  `
})
export class DatatableExampleComponent implements OnInit {
  products: Product[] = [];
  selectedProducts: Product[] = [];
  categories: string[] = ['Electronics', 'Clothing', 'Books'];

  ngOnInit() {
    this.products = [
      {
        id: '1',
        code: 'P001',
        name: 'Product 1',
        category: 'Electronics',
        quantity: 10,
        price: 99.99,
        inventoryStatus: 'INSTOCK'
      },
      // ... more products
    ];
  }

  onGlobalFilter(event: Event) {
    const table = (event.target as any)?.closest('p-table');
    if (table) {
      const value = (event.target as HTMLInputElement).value;
      table.filterGlobal(value, 'contains');
    }
  }

  onFilter(event: any, field: string, matchMode: string) {
    // Custom filter logic
  }

  editProduct(product: Product) {
    console.log('Edit:', product);
  }

  deleteProduct(product: Product) {
    console.log('Delete:', product);
  }
}
```

### 3. Form Components

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { InputTextModule } from 'primeng/inputtext';
import { InputTextareaModule } from 'primeng/inputtextarea';
import { DropdownModule } from 'primeng/dropdown';
import { CalendarModule } from 'primeng/calendar';
import { CheckboxModule } from 'primeng/checkbox';
import { RadioButtonModule } from 'primeng/radiobutton';
import { InputNumberModule } from 'primeng/inputnumber';
import { MultiSelectModule } from 'primeng/multiselect';
import { ButtonModule } from 'primeng/button';

@Component({
  selector: 'app-form-example',
  standalone: true,
  imports: [
    ReactiveFormsModule,
    InputTextModule,
    InputTextareaModule,
    DropdownModule,
    CalendarModule,
    CheckboxModule,
    RadioButtonModule,
    InputNumberModule,
    MultiSelectModule,
    ButtonModule
  ],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div class="p-fluid">
        <!-- Text Input -->
        <div class="p-field">
          <label for="name">Name</label>
          <input
            pInputText
            id="name"
            formControlName="name"
            placeholder="Enter your name"
          />
          <small *ngIf="form.get('name')?.invalid && form.get('name')?.touched" class="p-error">
            Name is required
          </small>
        </div>

        <!-- Email Input -->
        <div class="p-field">
          <label for="email">Email</label>
          <input
            pInputText
            id="email"
            type="email"
            formControlName="email"
            placeholder="Enter your email"
          />
          <small *ngIf="form.get('email')?.invalid && form.get('email')?.touched" class="p-error">
            Valid email is required
          </small>
        </div>

        <!-- Dropdown -->
        <div class="p-field">
          <label for="country">Country</label>
          <p-dropdown
            id="country"
            [options]="countries"
            formControlName="country"
            placeholder="Select a Country"
            optionLabel="name"
            [filter]="true"
            filterBy="name"
          ></p-dropdown>
        </div>

        <!-- Calendar -->
        <div class="p-field">
          <label for="birthdate">Birth Date</label>
          <p-calendar
            id="birthdate"
            formControlName="birthdate"
            dateFormat="mm/dd/yy"
            [showIcon]="true"
          ></p-calendar>
        </div>

        <!-- Number Input -->
        <div class="p-field">
          <label for="age">Age</label>
          <p-inputNumber
            id="age"
            formControlName="age"
            [min]="0"
            [max]="120"
          ></p-inputNumber>
        </div>

        <!-- Multi Select -->
        <div class="p-field">
          <label for="skills">Skills</label>
          <p-multiSelect
            id="skills"
            [options]="skills"
            formControlName="skills"
            placeholder="Select Skills"
            [filter]="true"
          ></p-multiSelect>
        </div>

        <!-- Textarea -->
        <div class="p-field">
          <label for="description">Description</label>
          <textarea
            pInputTextarea
            id="description"
            formControlName="description"
            rows="5"
            placeholder="Enter description"
          ></textarea>
        </div>

        <!-- Checkbox -->
        <div class="p-field-checkbox">
          <p-checkbox
            formControlName="agree"
            binary="true"
            inputId="agree"
          ></p-checkbox>
          <label for="agree">I agree to the terms and conditions</label>
        </div>

        <!-- Radio Buttons -->
        <div class="p-field">
          <label>Gender</label>
          <div class="p-formgrid p-grid">
            <div class="p-field-radiobutton p-col-6">
              <p-radioButton
                formControlName="gender"
                value="male"
                inputId="male"
              ></p-radioButton>
              <label for="male">Male</label>
            </div>
            <div class="p-field-radiobutton p-col-6">
              <p-radioButton
                formControlName="gender"
                value="female"
                inputId="female"
              ></p-radioButton>
              <label for="female">Female</label>
            </div>
          </div>
        </div>

        <!-- Submit Button -->
        <p-button
          label="Submit"
          type="submit"
          [disabled]="form.invalid"
        ></p-button>
      </div>
    </form>
  `
})
export class FormExampleComponent {
  form: FormGroup;
  countries = [
    { name: 'United States', code: 'US' },
    { name: 'United Kingdom', code: 'UK' },
    { name: 'Canada', code: 'CA' }
  ];
  skills = ['Angular', 'React', 'Vue', 'Node.js', 'Python'];

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
      country: [null],
      birthdate: [null],
      age: [null],
      skills: [[]],
      description: [''],
      agree: [false],
      gender: ['male']
    });
  }

  onSubmit() {
    if (this.form.valid) {
      console.log(this.form.value);
    }
  }
}
```

### 4. Charts

```typescript
import { Component, OnInit } from '@angular/core';
import { ChartModule } from 'primeng/chart';

@Component({
  selector: 'app-chart-example',
  standalone: true,
  imports: [ChartModule],
  template: `
    <div class="p-grid">
      <div class="p-col-12 p-md-6">
        <p-chart type="bar" [data]="barData" [options]="barOptions"></p-chart>
      </div>
      
      <div class="p-col-12 p-md-6">
        <p-chart type="line" [data]="lineData" [options]="lineOptions"></p-chart>
      </div>
      
      <div class="p-col-12 p-md-6">
        <p-chart type="pie" [data]="pieData" [options]="pieOptions"></p-chart>
      </div>
      
      <div class="p-col-12 p-md-6">
        <p-chart type="doughnut" [data]="doughnutData"></p-chart>
      </div>
    </div>
  `
})
export class ChartExampleComponent implements OnInit {
  barData: any;
  barOptions: any;
  lineData: any;
  lineOptions: any;
  pieData: any;
  pieOptions: any;
  doughnutData: any;

  ngOnInit() {
    this.barData = {
      labels: ['January', 'February', 'March', 'April', 'May', 'June'],
      datasets: [
        {
          label: 'Sales',
          backgroundColor: '#42A5F5',
          borderColor: '#1E88E5',
          data: [65, 59, 80, 81, 56, 55]
        },
        {
          label: 'Revenue',
          backgroundColor: '#9CCC65',
          borderColor: '#7CB342',
          data: [28, 48, 40, 19, 86, 27]
        }
      ]
    };

    this.barOptions = {
      plugins: {
        legend: {
          labels: {
            color: '#495057'
          }
        }
      },
      scales: {
        x: {
          ticks: {
            color: '#495057'
          },
          grid: {
            color: '#ebedef'
          }
        },
        y: {
          ticks: {
            color: '#495057'
          },
          grid: {
            color: '#ebedef'
          }
        }
      }
    };

    this.lineData = {
      labels: ['January', 'February', 'March', 'April', 'May', 'June', 'July'],
      datasets: [
        {
          label: 'First Dataset',
          data: [65, 59, 80, 81, 56, 55, 40],
          fill: false,
          borderColor: '#42A5F5',
          tension: 0.4
        },
        {
          label: 'Second Dataset',
          data: [28, 48, 40, 19, 86, 27, 90],
          fill: false,
          borderColor: '#FFA726',
          tension: 0.4
        }
      ]
    };

    this.pieData = {
      labels: ['A', 'B', 'C'],
      datasets: [
        {
          data: [300, 50, 100],
          backgroundColor: ['#FF6384', '#36A2EB', '#FFCE56']
        }
      ]
    };

    this.doughnutData = {
      labels: ['A', 'B', 'C'],
      datasets: [
        {
          data: [300, 50, 100],
          backgroundColor: ['#FF6384', '#36A2EB', '#FFCE56']
        }
      ]
    };
  }
}
```

### 5. Dialog and Overlays

```typescript
import { Component } from '@angular/core';
import { DialogModule } from 'primeng/dialog';
import { ButtonModule } from 'primeng/button';
import { OverlayPanelModule } from 'primeng/overlaypanel';
import { TooltipModule } from 'primeng/tooltip';
import { ConfirmDialogModule } from 'primeng/confirmdialog';
import { ConfirmationService } from 'primeng/api';

@Component({
  selector: 'app-overlay-example',
  standalone: true,
  imports: [
    DialogModule,
    ButtonModule,
    OverlayPanelModule,
    TooltipModule,
    ConfirmDialogModule
  ],
  providers: [ConfirmationService],
  template: `
    <!-- Dialog -->
    <p-button
      label="Show Dialog"
      (click)="displayDialog = true"
    ></p-button>

    <p-dialog
      header="Dialog Header"
      [(visible)]="displayDialog"
      [modal]="true"
      [style]="{ width: '50vw' }"
      [draggable]="false"
      [resizable]="false"
    >
      <p>Dialog content goes here.</p>
      <ng-template pTemplate="footer">
        <p-button
          label="Cancel"
          icon="pi pi-times"
          (click)="displayDialog = false"
          styleClass="p-button-text"
        ></p-button>
        <p-button
          label="Save"
          icon="pi pi-check"
          (click)="displayDialog = false"
        ></p-button>
      </ng-template>
    </p-dialog>

    <!-- Overlay Panel -->
    <p-button
      type="button"
      label="Toggle Overlay"
      (click)="op.toggle($event)"
    ></p-button>

    <p-overlayPanel #op>
      <div>
        <h4>Overlay Content</h4>
        <p>This is overlay panel content.</p>
      </div>
    </p-overlayPanel>

    <!-- Tooltip -->
    <p-button
      label="Hover Me"
      pTooltip="This is a tooltip"
      tooltipPosition="top"
    ></p-button>

    <!-- Confirm Dialog -->
    <p-button
      label="Delete"
      (click)="confirm()"
      styleClass="p-button-danger"
    ></p-button>

    <p-confirmDialog></p-confirmDialog>
  `
})
export class OverlayExampleComponent {
  displayDialog = false;

  constructor(private confirmationService: ConfirmationService) {}

  confirm() {
    this.confirmationService.confirm({
      message: 'Are you sure you want to delete this item?',
      header: 'Confirmation',
      icon: 'pi pi-exclamation-triangle',
      accept: () => {
        console.log('Item deleted');
      },
      reject: () => {
        console.log('Deletion cancelled');
      }
    });
  }
}
```

### 6. Menu Components

```typescript
import { Component } from '@angular/core';
import { MenubarModule } from 'primeng/menubar';
import { MenuModule } from 'primeng/menu';
import { MegaMenuModule } from 'primeng/megamenu';
import { MenuItem } from 'primeng/api';

@Component({
  selector: 'app-menu-example',
  standalone: true,
  imports: [
    MenubarModule,
    MenuModule,
    MegaMenuModule
  ],
  template: `
    <!-- Menubar -->
    <p-menubar [model]="items">
      <ng-template pTemplate="start">
        <img src="assets/logo.png" height="40" class="p-mr-2" />
      </ng-template>
      <ng-template pTemplate="end">
        <input type="text" pInputText placeholder="Search" />
      </ng-template>
    </p-menubar>

    <!-- Vertical Menu -->
    <p-menu [model]="menuItems" [style]="{ width: '250px' }"></p-menu>

    <!-- Mega Menu -->
    <p-megaMenu [model]="megaMenuItems"></p-megaMenu>
  `
})
export class MenuExampleComponent {
  items: MenuItem[] = [
    {
      label: 'File',
      icon: 'pi pi-fw pi-file',
      items: [
        {
          label: 'New',
          icon: 'pi pi-fw pi-plus',
          command: () => this.onNew()
        },
        {
          label: 'Open',
          icon: 'pi pi-fw pi-folder-open'
        }
      ]
    },
    {
      label: 'Edit',
      icon: 'pi pi-fw pi-pencil',
      items: [
        { label: 'Copy', icon: 'pi pi-fw pi-copy' },
        { label: 'Delete', icon: 'pi pi-fw pi-trash' }
      ]
    },
    {
      label: 'Users',
      icon: 'pi pi-fw pi-user',
      routerLink: ['/users']
    },
    {
      label: 'Quit',
      icon: 'pi pi-fw pi-power-off'
    }
  ];

  menuItems: MenuItem[] = [
    {
      label: 'Documents',
      items: [
        { label: 'New', icon: 'pi pi-plus' },
        { label: 'Search', icon: 'pi pi-search' }
      ]
    },
    {
      label: 'Profile',
      items: [
        { label: 'Settings', icon: 'pi pi-cog' },
        { label: 'Logout', icon: 'pi pi-sign-out' }
      ]
    }
  ];

  megaMenuItems: MenuItem[] = [
    {
      label: 'Products',
      items: [
        [
          {
            label: 'Category 1',
            items: [
              { label: 'Product 1.1' },
              { label: 'Product 1.2' }
            ]
          }
        ],
        [
          {
            label: 'Category 2',
            items: [
              { label: 'Product 2.1' },
              { label: 'Product 2.2' }
            ]
          }
        ]
      ]
    }
  ];

  onNew() {
    console.log('New action');
  }
}
```

### 7. Toast Notifications

```typescript
import { Component } from '@angular/core';
import { ToastModule } from 'primeng/toast';
import { MessageService } from 'primeng/api';
import { ButtonModule } from 'primeng/button';

@Component({
  selector: 'app-toast-example',
  standalone: true,
  imports: [ToastModule, ButtonModule],
  providers: [MessageService],
  template: `
    <p-toast></p-toast>
    
    <div class="p-d-flex p-flex-column p-ai-center">
      <p-button
        label="Success"
        (click)="showSuccess()"
        styleClass="p-mb-2"
      ></p-button>
      
      <p-button
        label="Info"
        (click)="showInfo()"
        styleClass="p-button-info p-mb-2"
      ></p-button>
      
      <p-button
        label="Warning"
        (click)="showWarn()"
        styleClass="p-button-warning p-mb-2"
      ></p-button>
      
      <p-button
        label="Error"
        (click)="showError()"
        styleClass="p-button-danger p-mb-2"
      ></p-button>
      
      <p-button
        label="Multiple"
        (click)="showMultiple()"
        styleClass="p-button-secondary"
      ></p-button>
    </div>
  `
})
export class ToastExampleComponent {
  constructor(private messageService: MessageService) {}

  showSuccess() {
    this.messageService.add({
      severity: 'success',
      summary: 'Success',
      detail: 'Operation completed successfully'
    });
  }

  showInfo() {
    this.messageService.add({
      severity: 'info',
      summary: 'Info',
      detail: 'This is an informational message'
    });
  }

  showWarn() {
    this.messageService.add({
      severity: 'warn',
      summary: 'Warning',
      detail: 'This action requires caution'
    });
  }

  showError() {
    this.messageService.add({
      severity: 'error',
      summary: 'Error',
      detail: 'Operation failed',
      life: 5000
    });
  }

  showMultiple() {
    this.messageService.addAll([
      { severity: 'success', summary: 'Message 1', detail: 'Success' },
      { severity: 'info', summary: 'Message 2', detail: 'Info' },
      { severity: 'warn', summary: 'Message 3', detail: 'Warning' }
    ]);
  }
}
```

### 8. File Upload

```typescript
import { Component } from '@angular/core';
import { FileUploadModule } from 'primeng/fileupload';
import { MessageService } from 'primeng/api';

@Component({
  selector: 'app-upload-example',
  standalone: true,
  imports: [FileUploadModule],
  providers: [MessageService],
  template: `
    <p-fileUpload
      name="files[]"
      url="./upload"
      [multiple]="true"
      accept="image/*"
      maxFileSize="1000000"
      (onUpload)="onUpload($event)"
      (onSelect)="onSelect($event)"
      [showUploadButton]="false"
      [showCancelButton]="false"
    >
      <ng-template pTemplate="content">
        <ul *ngIf="uploadedFiles.length">
          <li *ngFor="let file of uploadedFiles">
            {{ file.name }} - {{ file.size }} bytes
          </li>
        </ul>
      </ng-template>
    </p-fileUpload>
  `
})
export class UploadExampleComponent {
  uploadedFiles: any[] = [];

  constructor(private messageService: MessageService) {}

  onUpload(event: any) {
    for (let file of event.files) {
      this.uploadedFiles.push(file);
    }

    this.messageService.add({
      severity: 'info',
      summary: 'File Uploaded',
      detail: ''
    });
  }

  onSelect(event: any) {
    console.log('Files selected:', event.files);
  }
}
```

## Theming

### Custom Theme

```scss
// custom-theme.scss
$primaryColor: #007bff;
$primaryDarkColor: #0056b3;

@import "primeng/resources/themes/lara-light-blue/theme.css";

// Override variables
:root {
  --primary-color: #{$primaryColor};
  --primary-color-text: #ffffff;
}
```

### Using PrimeFlex

```bash
npm install primeflex
```

```scss
@import "primeflex/primeflex.css";
```

```html
<div class="p-grid">
  <div class="p-col-12 p-md-6 p-lg-4">
    Column 1
  </div>
  <div class="p-col-12 p-md-6 p-lg-4">
    Column 2
  </div>
</div>
```

## Common Mistakes

1. **Not importing animations module**: PrimeNG requires animations.

2. **Forgetting to add styles**: Must import theme and icons CSS.

3. **Not using MessageService/ConfirmationService**: Required for toasts and confirms.

4. **Improper data binding**: Use correct property names (e.g., `[value]` not `value`).

5. **Not handling table events**: DataTable requires event handlers for custom logic.

## Best Practices

1. **Use standalone components**: Import only needed modules
2. **Leverage templates**: Use pTemplate for customization
3. **Consistent theming**: Create custom themes for brand consistency
4. **Accessibility**: Test with keyboard navigation
5. **Performance**: Use lazy loading for large datasets
6. **Responsive design**: Use PrimeFlex utilities
7. **Type safety**: Define interfaces for data structures
8. **Error handling**: Implement proper validation and error messages

## When to Use PrimeNG

**Use PrimeNG when:**

- Building enterprise Angular applications
- Need feature-rich data tables
- Want extensive component library
- Require professional design
- Need charts and data visualization
- Building admin dashboards

**Consider alternatives when:**

- Want Material Design (use Angular Material)
- Need lighter bundle size
- Mobile-first applications
- Simple CRUD applications

## Interview Questions

1. **Q: What makes PrimeNG DataTable special?**
   A: Advanced features like filtering, sorting, pagination, lazy loading, row selection, column reordering, resizing, frozen columns, row expansion, export, and inline editing all built-in.

2. **Q: How does PrimeNG theming work?**
   A: PrimeNG provides pre-built themes based on CSS variables. You can customize by overriding CSS variables or creating custom themes using Sass variables.

3. **Q: What is the purpose of pTemplate?**
   A: pTemplate is a directive for customizing component content. It allows you to define custom templates for headers, footers, content, and other sections of components.

4. **Q: How do you handle file uploads in PrimeNG?**
   A: Use FileUpload component with url for automatic upload or manual mode with (onSelect) event to handle files programmatically. Supports multiple files, validation, and progress tracking.

5. **Q: What's the difference between Dialog and OverlayPanel?**
   A: Dialog is a modal window with header/footer that blocks interaction with page. OverlayPanel is a non-modal popup that appears relative to a target element and allows page interaction.

## Key Takeaways

1. PrimeNG provides 90+ enterprise-grade UI components for Angular
2. DataTable is one of the most feature-rich table components available
3. Built-in theming with customizable CSS variables
4. PrimeFlex provides utility CSS for layouts and spacing
5. Charts powered by Chart.js with PrimeNG wrapper
6. Toast and ConfirmDialog require service providers
7. Templates (pTemplate) enable extensive customization
8. File upload supports validation, progress, and multiple files
9. Menu components support hierarchical navigation
10. Excellent for enterprise applications and admin dashboards

## Resources

- [PrimeNG Documentation](https://primeng.org/)
- [PrimeNG GitHub](https://github.com/primefaces/primeng)
- [PrimeFlex Utilities](https://www.primefaces.org/primeflex/)
- [PrimeIcons](https://www.primefaces.org/primeicons/)
- [Theming Guide](https://primeng.org/theming)
- [Templates Showcase](https://www.primefaces.org/layouts/apollo-ng)
