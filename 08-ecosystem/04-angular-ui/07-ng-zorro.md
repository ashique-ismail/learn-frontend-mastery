# NG-ZORRO (Ant Design Angular)

## Overview

NG-ZORRO is the Angular implementation of Ant Design, an enterprise-class UI design language and React UI library created by Alibaba. It provides 60+ high-quality Angular components out of the box, following Ant Design specifications with a focus on enterprise applications, data-intensive interfaces, and administrative dashboards.

NG-ZORRO offers comprehensive components including advanced data tables, complex forms, charts, layouts, and navigation with built-in internationalization (i18n), theming, TypeScript support, and accessibility features. It's widely adopted in enterprise Angular applications, particularly in admin panels and business applications.

## Installation and Setup

```bash
ng add ng-zorro-antd
```

The `ng add` command will:
- Install ng-zorro-antd package
- Import modules automatically
- Add styles to angular.json
- Configure animations

### Manual Installation

```bash
npm install ng-zorro-antd
```

```typescript
// app.config.ts (Standalone)
import { ApplicationConfig } from '@angular/core';
import { provideAnimations } from '@angular/platform-browser/animations';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations(),
    provideHttpClient()
  ]
};
```

```scss
// styles.scss
@import "~ng-zorro-antd/ng-zorro-antd.min.css";
```

### Module Setup

```typescript
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
import { NzButtonModule } from 'ng-zorro-antd/button';

@NgModule({
  imports: [
    BrowserAnimationsModule,
    NzButtonModule
  ]
})
export class AppModule {}
```

## Core Concepts

### 1. Basic Components

```typescript
import { Component } from '@angular/core';
import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzCardModule } from 'ng-zorro-antd/card';
import { NzIconModule } from 'ng-zorro-antd/icon';
import { NzSpaceModule } from 'ng-zorro-antd/space';

@Component({
  selector: 'app-basic-example',
  standalone: true,
  imports: [
    NzButtonModule,
    NzCardModule,
    NzIconModule,
    NzSpaceModule
  ],
  template: `
    <nz-card
      nzTitle="Welcome to NG-ZORRO"
      [nzExtra]="extraTemplate"
      [nzActions]="[actionSetting, actionEdit, actionEllipsis]"
    >
      <p>Ant Design components for Angular</p>
      
      <nz-space>
        <button *nzSpaceItem nz-button nzType="primary">
          <span nz-icon nzType="download"></span>
          Primary
        </button>
        <button *nzSpaceItem nz-button nzType="default">
          Default
        </button>
        <button *nzSpaceItem nz-button nzType="dashed">
          Dashed
        </button>
        <button *nzSpaceItem nz-button nzType="link">
          Link
        </button>
        <button *nzSpaceItem nz-button nzType="text">
          Text
        </button>
        <button *nzSpaceItem nz-button nzType="primary" nzDanger>
          Danger
        </button>
      </nz-space>
    </nz-card>

    <ng-template #extraTemplate>
      <a>More</a>
    </ng-template>

    <ng-template #actionSetting>
      <span nz-icon nzType="setting"></span>
    </ng-template>
    <ng-template #actionEdit>
      <span nz-icon nzType="edit"></span>
    </ng-template>
    <ng-template #actionEllipsis>
      <span nz-icon nzType="ellipsis"></span>
    </ng-template>
  `
})
export class BasicExampleComponent {}
```

### 2. Advanced Table

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { NzTableModule } from 'ng-zorro-antd/table';
import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzIconModule } from 'ng-zorro-antd/icon';
import { NzInputModule } from 'ng-zorro-antd/input';
import { NzSelectModule } from 'ng-zorro-antd/select';
import { NzTagModule } from 'ng-zorro-antd/tag';
import { FormsModule } from '@angular/forms';

interface DataItem {
  id: number;
  name: string;
  age: number;
  address: string;
  tags: string[];
}

@Component({
  selector: 'app-table-example',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule,
    NzTableModule,
    NzButtonModule,
    NzIconModule,
    NzInputModule,
    NzSelectModule,
    NzTagModule
  ],
  template: `
    <nz-table
      #basicTable
      [nzData]="listOfData"
      [nzPageSize]="10"
      [nzPageSizeOptions]="[10, 20, 50]"
      [nzShowSizeChanger]="true"
      [nzShowPagination]="true"
      nzTableLayout="fixed"
    >
      <thead>
        <tr>
          <th
            nzColumnKey="name"
            [nzSortFn]="sortByName"
            [nzFilterMultiple]="false"
          >
            Name
          </th>
          <th
            nzColumnKey="age"
            [nzSortFn]="sortByAge"
          >
            Age
          </th>
          <th>Address</th>
          <th
            nzColumnKey="tags"
            [nzFilters]="tagFilters"
            [nzFilterFn]="filterByTags"
          >
            Tags
          </th>
          <th nzWidth="200px">Action</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let data of basicTable.data">
          <td>{{ data.name }}</td>
          <td>{{ data.age }}</td>
          <td>{{ data.address }}</td>
          <td>
            <nz-tag *ngFor="let tag of data.tags" [nzColor]="getTagColor(tag)">
              {{ tag }}
            </nz-tag>
          </td>
          <td>
            <button nz-button nzType="link" (click)="editRow(data)">
              <span nz-icon nzType="edit"></span>
              Edit
            </button>
            <button nz-button nzType="link" nzDanger (click)="deleteRow(data)">
              <span nz-icon nzType="delete"></span>
              Delete
            </button>
          </td>
        </tr>
      </tbody>
    </nz-table>
  `
})
export class TableExampleComponent implements OnInit {
  listOfData: DataItem[] = [];
  
  tagFilters = [
    { text: 'Nice', value: 'nice' },
    { text: 'Developer', value: 'developer' },
    { text: 'Cool', value: 'cool' }
  ];

  ngOnInit(): void {
    this.listOfData = [
      {
        id: 1,
        name: 'John Brown',
        age: 32,
        address: 'New York No. 1 Lake Park',
        tags: ['nice', 'developer']
      },
      {
        id: 2,
        name: 'Jim Green',
        age: 42,
        address: 'London No. 1 Lake Park',
        tags: ['cool']
      },
      {
        id: 3,
        name: 'Joe Black',
        age: 32,
        address: 'Sidney No. 1 Lake Park',
        tags: ['cool', 'teacher']
      }
    ];
  }

  sortByName = (a: DataItem, b: DataItem) => a.name.localeCompare(b.name);
  sortByAge = (a: DataItem, b: DataItem) => a.age - b.age;
  
  filterByTags = (tags: string[], item: DataItem) =>
    item.tags.some(tag => tags.includes(tag));

  getTagColor(tag: string): string {
    const colorMap: Record<string, string> = {
      'nice': 'green',
      'developer': 'blue',
      'cool': 'cyan',
      'teacher': 'purple'
    };
    return colorMap[tag] || 'default';
  }

  editRow(data: DataItem): void {
    console.log('Edit:', data);
  }

  deleteRow(data: DataItem): void {
    this.listOfData = this.listOfData.filter(d => d.id !== data.id);
  }
}
```

### 3. Form Components

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { NzFormModule } from 'ng-zorro-antd/form';
import { NzInputModule } from 'ng-zorro-antd/input';
import { NzSelectModule } from 'ng-zorro-antd/select';
import { NzDatePickerModule } from 'ng-zorro-antd/date-picker';
import { NzSwitchModule } from 'ng-zorro-antd/switch';
import { NzCheckboxModule } from 'ng-zorro-antd/checkbox';
import { NzRadioModule } from 'ng-zorro-antd/radio';
import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzInputNumberModule } from 'ng-zorro-antd/input-number';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-form-example',
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    NzFormModule,
    NzInputModule,
    NzSelectModule,
    NzDatePickerModule,
    NzSwitchModule,
    NzCheckboxModule,
    NzRadioModule,
    NzButtonModule,
    NzInputNumberModule
  ],
  template: `
    <form nz-form [formGroup]="validateForm" (ngSubmit)="submitForm()">
      <!-- Text Input -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6" nzRequired>Username</nz-form-label>
        <nz-form-control [nzSpan]="14" nzErrorTip="Please input your username!">
          <input nz-input formControlName="userName" placeholder="Username" />
        </nz-form-control>
      </nz-form-item>

      <!-- Email -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6" nzRequired>Email</nz-form-label>
        <nz-form-control [nzSpan]="14" nzErrorTip="Please input a valid email!">
          <input nz-input formControlName="email" type="email" placeholder="Email" />
        </nz-form-control>
      </nz-form-item>

      <!-- Password -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6" nzRequired>Password</nz-form-label>
        <nz-form-control [nzSpan]="14" nzErrorTip="Please input your password!">
          <input
            nz-input
            type="password"
            formControlName="password"
            placeholder="Password"
          />
        </nz-form-control>
      </nz-form-item>

      <!-- Select -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6" nzRequired>Country</nz-form-label>
        <nz-form-control [nzSpan]="14">
          <nz-select formControlName="country" nzPlaceHolder="Select a country">
            <nz-option nzValue="us" nzLabel="United States"></nz-option>
            <nz-option nzValue="uk" nzLabel="United Kingdom"></nz-option>
            <nz-option nzValue="ca" nzLabel="Canada"></nz-option>
          </nz-select>
        </nz-form-control>
      </nz-form-item>

      <!-- Date Picker -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6">Birth Date</nz-form-label>
        <nz-form-control [nzSpan]="14">
          <nz-date-picker formControlName="birthDate" nzPlaceHolder="Select date"></nz-date-picker>
        </nz-form-control>
      </nz-form-item>

      <!-- Number Input -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6">Age</nz-form-label>
        <nz-form-control [nzSpan]="14">
          <nz-input-number
            formControlName="age"
            [nzMin]="1"
            [nzMax]="120"
            [nzStep]="1"
          ></nz-input-number>
        </nz-form-control>
      </nz-form-item>

      <!-- Switch -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6">Notifications</nz-form-label>
        <nz-form-control [nzSpan]="14">
          <nz-switch formControlName="notifications"></nz-switch>
        </nz-form-control>
      </nz-form-item>

      <!-- Checkbox -->
      <nz-form-item>
        <nz-form-control [nzSpan]="14" [nzOffset]="6">
          <label nz-checkbox formControlName="agree">
            I have read and agree to the terms
          </label>
        </nz-form-control>
      </nz-form-item>

      <!-- Radio Group -->
      <nz-form-item>
        <nz-form-label [nzSpan]="6">Gender</nz-form-label>
        <nz-form-control [nzSpan]="14">
          <nz-radio-group formControlName="gender">
            <label nz-radio nzValue="male">Male</label>
            <label nz-radio nzValue="female">Female</label>
            <label nz-radio nzValue="other">Other</label>
          </nz-radio-group>
        </nz-form-control>
      </nz-form-item>

      <!-- Submit Button -->
      <nz-form-item>
        <nz-form-control [nzSpan]="14" [nzOffset]="6">
          <button nz-button nzType="primary" [disabled]="!validateForm.valid">
            Submit
          </button>
          <button nz-button (click)="resetForm()" type="button" class="ml-2">
            Reset
          </button>
        </nz-form-control>
      </nz-form-item>
    </form>
  `
})
export class FormExampleComponent {
  validateForm: FormGroup;

  constructor(private fb: FormBuilder) {
    this.validateForm = this.fb.group({
      userName: ['', [Validators.required]],
      email: ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(6)]],
      country: [null, [Validators.required]],
      birthDate: [null],
      age: [18],
      notifications: [true],
      agree: [false],
      gender: ['male']
    });
  }

  submitForm(): void {
    if (this.validateForm.valid) {
      console.log('Form values:', this.validateForm.value);
    } else {
      Object.values(this.validateForm.controls).forEach(control => {
        if (control.invalid) {
          control.markAsDirty();
          control.updateValueAndValidity({ onlySelf: true });
        }
      });
    }
  }

  resetForm(): void {
    this.validateForm.reset();
  }
}
```

### 4. Modal Dialogs

```typescript
import { Component } from '@angular/core';
import { NzModalModule, NzModalService } from 'ng-zorro-antd/modal';
import { NzButtonModule } from 'ng-zorro-antd/button';

@Component({
  selector: 'app-modal-content',
  standalone: true,
  template: `
    <p>Modal Content</p>
    <p>Data: {{ data }}</p>
  `
})
export class ModalContentComponent {
  data: string = '';
}

@Component({
  selector: 'app-modal-example',
  standalone: true,
  imports: [NzModalModule, NzButtonModule],
  template: `
    <button nz-button nzType="primary" (click)="showModal()">
      Open Modal
    </button>
    
    <button nz-button (click)="showConfirm()">
      Confirm Modal
    </button>
    
    <button nz-button (click)="showInfo()">
      Info Modal
    </button>

    <button nz-button nzDanger (click)="showDelete()">
      Delete Confirm
    </button>
  `
})
export class ModalExampleComponent {
  constructor(private modal: NzModalService) {}

  showModal(): void {
    const modal = this.modal.create({
      nzTitle: 'Modal Title',
      nzContent: ModalContentComponent,
      nzFooter: [
        {
          label: 'Cancel',
          onClick: () => modal.destroy()
        },
        {
          label: 'OK',
          type: 'primary',
          onClick: () => {
            console.log('OK clicked');
            modal.destroy();
          }
        }
      ]
    });

    modal.componentInstance!.data = 'Custom data';
  }

  showConfirm(): void {
    this.modal.confirm({
      nzTitle: 'Do you confirm?',
      nzContent: 'When clicked the OK button, this dialog will be closed',
      nzOnOk: () => console.log('OK')
    });
  }

  showInfo(): void {
    this.modal.info({
      nzTitle: 'This is a notification message',
      nzContent: 'Some descriptions'
    });
  }

  showDelete(): void {
    this.modal.confirm({
      nzTitle: 'Are you sure delete this task?',
      nzContent: '<b>This action cannot be undone</b>',
      nzOkText: 'Yes',
      nzOkType: 'primary',
      nzOkDanger: true,
      nzOnOk: () => console.log('Deleted'),
      nzCancelText: 'No',
      nzOnCancel: () => console.log('Cancel')
    });
  }
}
```

### 5. Layout Components

```typescript
import { Component } from '@angular/core';
import { NzLayoutModule } from 'ng-zorro-antd/layout';
import { NzMenuModule } from 'ng-zorro-antd/menu';
import { NzBreadCrumbModule } from 'ng-zorro-antd/breadcrumb';
import { NzIconModule } from 'ng-zorro-antd/icon';

@Component({
  selector: 'app-layout-example',
  standalone: true,
  imports: [
    NzLayoutModule,
    NzMenuModule,
    NzBreadCrumbModule,
    NzIconModule
  ],
  template: `
    <nz-layout class="layout">
      <nz-header>
        <div class="logo"></div>
        <ul nz-menu nzTheme="dark" nzMode="horizontal">
          <li nz-menu-item nzSelected>
            <span nz-icon nzType="home"></span>
            Home
          </li>
          <li nz-menu-item>
            <span nz-icon nzType="user"></span>
            Users
          </li>
          <li nz-menu-item>
            <span nz-icon nzType="setting"></span>
            Settings
          </li>
        </ul>
      </nz-header>
      
      <nz-content style="padding:0 50px;">
        <nz-breadcrumb style="margin:16px 0;">
          <nz-breadcrumb-item>Home</nz-breadcrumb-item>
          <nz-breadcrumb-item>List</nz-breadcrumb-item>
          <nz-breadcrumb-item>App</nz-breadcrumb-item>
        </nz-breadcrumb>
        
        <div class="inner-content">
          <p>Content</p>
        </div>
      </nz-content>
      
      <nz-footer style="text-align: center;">
        Ant Design ©2023 Implement By Angular
      </nz-footer>
    </nz-layout>
  `,
  styles: [`
    .layout {
      min-height: 100vh;
    }
    .logo {
      width: 120px;
      height: 31px;
      background: rgba(255, 255, 255, 0.2);
      margin: 16px 24px 16px 0;
      float: left;
    }
    .inner-content {
      background: #fff;
      padding: 24px;
      min-height: 380px;
    }
  `]
})
export class LayoutExampleComponent {}
```

### 6. Message and Notification

```typescript
import { Component } from '@angular/core';
import { NzButtonModule } from 'ng-zorro-antd/button';
import { NzMessageService } from 'ng-zorro-antd/message';
import { NzNotificationService } from 'ng-zorro-antd/notification';

@Component({
  selector: 'app-message-example',
  standalone: true,
  imports: [NzButtonModule],
  template: `
    <h3>Messages</h3>
    <button nz-button (click)="createBasicMessage()">Display normal message</button>
    <button nz-button (click)="createSuccessMessage()">Success</button>
    <button nz-button (click)="createErrorMessage()">Error</button>
    <button nz-button (click)="createWarningMessage()">Warning</button>

    <h3 class="mt-4">Notifications</h3>
    <button nz-button (click)="createBasicNotification()">Basic</button>
    <button nz-button (click)="createSuccessNotification()">Success</button>
    <button nz-button (click)="createErrorNotification()">Error</button>
    <button nz-button (click)="createWarningNotification()">Warning</button>
  `
})
export class MessageExampleComponent {
  constructor(
    private message: NzMessageService,
    private notification: NzNotificationService
  ) {}

  createBasicMessage(): void {
    this.message.info('This is a normal message');
  }

  createSuccessMessage(): void {
    this.message.success('This is a success message');
  }

  createErrorMessage(): void {
    this.message.error('This is an error message');
  }

  createWarningMessage(): void {
    this.message.warning('This is a warning message');
  }

  createBasicNotification(): void {
    this.notification.blank(
      'Notification Title',
      'This is the content of the notification.'
    );
  }

  createSuccessNotification(): void {
    this.notification.success(
      'Success',
      'Operation completed successfully!'
    );
  }

  createErrorNotification(): void {
    this.notification.error(
      'Error',
      'There was an error processing your request.'
    );
  }

  createWarningNotification(): void {
    this.notification.warning(
      'Warning',
      'Please review your input before proceeding.'
    );
  }
}
```

### 7. Internationalization (i18n)

```typescript
import { Component } from '@angular/core';
import { en_US, zh_CN, NzI18nService } from 'ng-zorro-antd/i18n';
import { NzSelectModule } from 'ng-zorro-antd/select';
import { NzDatePickerModule } from 'ng-zorro-antd/date-picker';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-i18n-example',
  standalone: true,
  imports: [
    FormsModule,
    NzSelectModule,
    NzDatePickerModule
  ],
  template: `
    <div>
      <label>Switch Language: </label>
      <nz-select [(ngModel)]="selectedLang" (ngModelChange)="changeLanguage()">
        <nz-option nzValue="en" nzLabel="English"></nz-option>
        <nz-option nzValue="zh" nzLabel="中文"></nz-option>
      </nz-select>
    </div>

    <div class="mt-3">
      <nz-date-picker></nz-date-picker>
    </div>
  `
})
export class I18nExampleComponent {
  selectedLang = 'en';

  constructor(private i18n: NzI18nService) {}

  changeLanguage(): void {
    this.i18n.setLocale(this.selectedLang === 'en' ? en_US : zh_CN);
  }
}
```

### 8. Theming

```scss
// Custom theme variables
@import "~ng-zorro-antd/ng-zorro-antd.less";

// Override theme variables
@primary-color: #1890ff; // primary color for all components
@link-color: #1890ff; // link color
@success-color: #52c41a; // success state color
@warning-color: #faad14; // warning state color
@error-color: #f5222d; // error state color
@font-size-base: 14px; // major text font size
@heading-color: rgba(0, 0, 0, 0.85); // heading text color
@text-color: rgba(0, 0, 0, 0.65); // major text color
@text-color-secondary: rgba(0, 0, 0, 0.45); // secondary text color
@disabled-color: rgba(0, 0, 0, 0.25); // disable state color
@border-radius-base: 2px; // major border radius
@border-color-base: #d9d9d9; // major border color
@box-shadow-base: 0 3px 6px -4px rgba(0, 0, 0, 0.12);
```

```typescript
// Dynamic theme switching
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <button (click)="toggleTheme()">Toggle Theme</button>
  `
})
export class AppComponent {
  isDark = false;

  toggleTheme(): void {
    this.isDark = !this.isDark;
    const theme = this.isDark ? 'dark' : 'default';
    // Load theme CSS dynamically
    const link = document.getElementById('theme-css') as HTMLLinkElement;
    link.href = `assets/themes/${theme}.css`;
  }
}
```

## Common Mistakes

1. **Not importing required modules**: Each component needs its specific module.

2. **Forgetting animations**: NG-ZORRO requires BrowserAnimationsModule.

3. **Improper form validation**: Must mark controls as dirty to show errors.

4. **Not handling i18n**: Import and configure NzI18nModule for datepickers.

5. **CSS conflicts**: Ensure proper CSS specificity when overriding styles.

## Best Practices

1. **Use standalone components**: Import only needed modules
2. **Leverage theming**: Customize theme variables for brand consistency
3. **Implement i18n**: Configure internationalization for global apps
4. **Accessibility**: Test with keyboard and screen readers
5. **Performance**: Use OnPush change detection where possible
6. **Type safety**: Use provided TypeScript interfaces
7. **Responsive design**: Use Grid and Layout components
8. **Consistent icons**: Use Ant Design icon library

## When to Use NG-ZORRO

**Use NG-ZORRO when:**
- Building enterprise Angular applications
- Want Ant Design aesthetics
- Need comprehensive component library
- Building admin dashboards
- Require i18n support out of the box
- Want professional, enterprise-grade UI

**Consider alternatives when:**
- Want Material Design (use Angular Material)
- Need Bootstrap design (use ng-bootstrap)
- Mobile-first applications (use Ionic)
- Lightweight requirements

## Interview Questions

1. **Q: What makes NG-ZORRO different from other Angular UI libraries?**
   A: NG-ZORRO implements Ant Design specifications, focuses on enterprise applications, provides 60+ components, has built-in i18n support, and offers comprehensive theming with less variables. It's particularly strong for data-intensive admin interfaces.

2. **Q: How does NG-ZORRO handle internationalization?**
   A: Uses NzI18nService with pre-built locale files. Call `setLocale()` to change language. Affects built-in components like DatePicker, Pagination, Table. Custom translations need separate implementation.

3. **Q: What are the advantages of NG-ZORRO's Table component?**
   A: Built-in sorting, filtering, pagination, selection, fixed columns, expandable rows, tree data support, virtual scrolling, customizable templates, and excellent performance with large datasets.

4. **Q: How do you customize NG-ZORRO themes?**
   A: Override less variables in angular.json or use ng-alain schematics for theme generation. Can modify primary colors, fonts, spacing, shadows, and component-specific styles.

5. **Q: What's the difference between Message and Notification?**
   A: Message is lightweight feedback at top of screen, auto-dismisses, for simple info. Notification is more prominent, appears in corner, can include actions, better for important updates requiring attention.

## Key Takeaways

1. NG-ZORRO is Angular implementation of Ant Design specification
2. Provides 60+ enterprise-grade components
3. Built-in internationalization with multiple locale support
4. Comprehensive theming system using less variables
5. Advanced Table component for data-intensive applications
6. Modal service for programmatic dialog management
7. Layout components for responsive admin interfaces
8. Form components integrate with Angular's reactive forms
9. Icon library with 700+ Ant Design icons
10. Ideal for enterprise applications and admin dashboards

## Resources

- [NG-ZORRO Documentation](https://ng.ant.design/)
- [NG-ZORRO GitHub](https://github.com/NG-ZORRO/ng-zorro-antd)
- [Ant Design Guidelines](https://ant.design/)
- [NG-ALAIN Scaffold](https://ng-alain.com/)
- [Theming Guide](https://ng.ant.design/docs/customize-theme/en)
- [Internationalization](https://ng.ant.design/docs/i18n/en)
