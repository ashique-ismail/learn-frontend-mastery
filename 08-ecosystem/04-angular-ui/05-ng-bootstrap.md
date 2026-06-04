# ng-bootstrap

## Overview

ng-bootstrap is a library that provides Bootstrap 4/5 components as native Angular directives and components, built from the ground up using Bootstrap's CSS and Angular's features. Unlike traditional Bootstrap that relies on jQuery, ng-bootstrap is purely Angular-based, providing type-safe, template-driven components with full accessibility support.

The library includes modals, tooltips, popovers, datepickers, typeaheads, carousels, accordions, and more, all implemented as Angular components without jQuery dependencies. It's maintained by the Angular UI team and widely adopted in the Angular community.

## Installation and Setup

```bash
npm install @ng-bootstrap/ng-bootstrap bootstrap
```

### Standalone Setup (Angular 15+)

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideAnimations } from '@angular/platform-browser/animations';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAnimations()
  ]
};

// Component
import { NgbModule } from '@ng-bootstrap/ng-bootstrap';

@Component({
  standalone: true,
  imports: [NgbModule],
  // ...
})
export class AppComponent {}
```

### Module Setup

```typescript
import { NgModule } from '@angular/core';
import { NgbModule } from '@ng-bootstrap/ng-bootstrap';

@NgModule({
  imports: [NgbModule],
  // ...
})
export class AppModule {}
```

### Add Bootstrap CSS

```scss
// styles.scss
@import "bootstrap/scss/bootstrap";

// Or use CDN in index.html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
```

## Core Concepts

### 1. Modal Dialogs

```typescript
import { Component, inject } from '@angular/core';
import { NgbModal, NgbModalModule } from '@ng-bootstrap/ng-bootstrap';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-modal-content',
  standalone: true,
  template: `
    <div class="modal-header">
      <h4 class="modal-title">Modal Title</h4>
      <button
        type="button"
        class="btn-close"
        aria-label="Close"
        (click)="activeModal.dismiss('Cross click')"
      ></button>
    </div>
    <div class="modal-body">
      <p>Modal body content goes here.</p>
      <p>{{ data }}</p>
    </div>
    <div class="modal-footer">
      <button
        type="button"
        class="btn btn-secondary"
        (click)="activeModal.close('Close click')"
      >
        Close
      </button>
      <button
        type="button"
        class="btn btn-primary"
        (click)="activeModal.close('Save click')"
      >
        Save
      </button>
    </div>
  `
})
export class ModalContentComponent {
  activeModal = inject(NgbActiveModal);
  data: string = '';
}

@Component({
  selector: 'app-modal-example',
  standalone: true,
  imports: [CommonModule, NgbModalModule],
  template: `
    <button class="btn btn-primary" (click)="open()">
      Open Modal
    </button>
    <button class="btn btn-secondary" (click)="openLarge()">
      Large Modal
    </button>
    <button class="btn btn-warning" (click)="openCentered()">
      Centered Modal
    </button>
  `
})
export class ModalExampleComponent {
  private modalService = inject(NgbModal);

  open() {
    const modalRef = this.modalService.open(ModalContentComponent);
    modalRef.componentInstance.data = 'Custom data passed to modal';

    modalRef.result.then(
      (result) => {
        console.log('Closed with:', result);
      },
      (reason) => {
        console.log('Dismissed with:', reason);
      }
    );
  }

  openLarge() {
    this.modalService.open(ModalContentComponent, {
      size: 'lg',
      backdrop: 'static',
      keyboard: false
    });
  }

  openCentered() {
    this.modalService.open(ModalContentComponent, {
      centered: true,
      scrollable: true
    });
  }
}
```

### 2. Tooltips and Popovers

```typescript
import { Component } from '@angular/core';
import { NgbTooltipModule, NgbPopoverModule } from '@ng-bootstrap/ng-bootstrap';

@Component({
  selector: 'app-tooltip-example',
  standalone: true,
  imports: [NgbTooltipModule, NgbPopoverModule],
  template: `
    <!-- Basic Tooltip -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      ngbTooltip="Nice tooltip!"
    >
      Hover me
    </button>

    <!-- Tooltip with custom trigger -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      ngbTooltip="You see, I show up on click!"
      triggers="click"
    >
      Click me
    </button>

    <!-- Tooltip with position -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      ngbTooltip="Top tooltip"
      placement="top"
    >
      Top
    </button>

    <!-- Tooltip with custom class -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      ngbTooltip="Fancy tooltip"
      tooltipClass="custom-tooltip"
    >
      Custom style
    </button>

    <!-- Popover -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      ngbPopover="Vivamus sagittis lacus vel augue laoreet rutrum faucibus."
      popoverTitle="Popover on top"
    >
      Click for popover
    </button>

    <!-- Popover with HTML -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      [ngbPopover]="popContent"
      popoverTitle="HTML Content"
    >
      HTML Popover
    </button>

    <ng-template #popContent>
      <div>
        <strong>Bold text</strong>
        <p class="text-muted">Some muted text</p>
      </div>
    </ng-template>

    <!-- Auto-close popover -->
    <button
      type="button"
      class="btn btn-outline-secondary"
      ngbPopover="Auto-closes when clicked outside"
      autoClose="outside"
    >
      Click me
    </button>
  `
})
export class TooltipExampleComponent {}
```

### 3. Datepicker

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { NgbDatepickerModule, NgbDateStruct } from '@ng-bootstrap/ng-bootstrap';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-datepicker-example',
  standalone: true,
  imports: [CommonModule, FormsModule, NgbDatepickerModule],
  template: `
    <div class="container">
      <!-- Basic Datepicker -->
      <div class="row mb-3">
        <div class="col">
          <ngb-datepicker
            [(ngModel)]="model"
            (ngModelChange)="onDateChange($event)"
          ></ngb-datepicker>
        </div>
      </div>

      <!-- Date Input with Popup -->
      <div class="row mb-3">
        <div class="col">
          <div class="input-group">
            <input
              class="form-control"
              placeholder="yyyy-mm-dd"
              name="dp"
              [(ngModel)]="model2"
              ngbDatepicker
              #d="ngbDatepicker"
            />
            <button
              class="btn btn-outline-secondary"
              (click)="d.toggle()"
              type="button"
            >
              <i class="bi bi-calendar"></i>
            </button>
          </div>
        </div>
      </div>

      <!-- Range Selection -->
      <div class="row mb-3">
        <div class="col">
          <ngb-datepicker
            #dp
            [(ngModel)]="modelRange"
            [displayMonths]="2"
            [dayTemplate]="t"
            outsideDays="hidden"
          ></ngb-datepicker>

          <ng-template #t let-date let-focused="focused">
            <span
              class="custom-day"
              [class.focused]="focused"
              [class.range]="isRange(date)"
              [class.faded]="isHovered(date) || isInside(date)"
              (mouseenter)="hoveredDate = date"
              (mouseleave)="hoveredDate = null"
            >
              {{ date.day }}
            </span>
          </ng-template>
        </div>
      </div>

      <!-- Custom Day Template -->
      <div class="row mb-3">
        <div class="col">
          <ngb-datepicker
            [(ngModel)]="model3"
            [dayTemplate]="customDay"
            [markDisabled]="isDisabled"
          ></ngb-datepicker>

          <ng-template #customDay let-date let-currentMonth="currentMonth">
            <span
              class="custom-day"
              [class.weekend]="isWeekend(date)"
              [class.other-month]="date.month !== currentMonth"
            >
              {{ date.day }}
            </span>
          </ng-template>
        </div>
      </div>

      <!-- Min/Max Dates -->
      <div class="row mb-3">
        <div class="col">
          <ngb-datepicker
            [(ngModel)]="model4"
            [minDate]="minDate"
            [maxDate]="maxDate"
          ></ngb-datepicker>
        </div>
      </div>

      <div class="row">
        <div class="col">
          <p>Selected: {{ model | json }}</p>
          <p>Range: {{ modelRange | json }}</p>
        </div>
      </div>
    </div>
  `,
  styles: [`
    .custom-day {
      text-align: center;
      padding: 0.185rem 0.25rem;
      display: inline-block;
      height: 2rem;
      width: 2rem;
    }
    .custom-day.focused {
      background-color: #e6e6e6;
    }
    .custom-day.range,
    .custom-day:hover {
      background-color: rgb(2, 117, 216);
      color: white;
    }
    .custom-day.weekend {
      color: #dc3545;
    }
  `]
})
export class DatepickerExampleComponent {
  model: NgbDateStruct | null = null;
  model2: NgbDateStruct | null = null;
  modelRange: NgbDateStruct | null = null;
  model3: NgbDateStruct | null = null;
  model4: NgbDateStruct | null = null;
  
  hoveredDate: NgbDateStruct | null = null;
  
  minDate: NgbDateStruct = { year: 2023, month: 1, day: 1 };
  maxDate: NgbDateStruct = { year: 2025, month: 12, day: 31 };

  onDateChange(date: NgbDateStruct) {
    console.log('Date changed:', date);
  }

  isRange(date: NgbDateStruct): boolean {
    // Custom range logic
    return false;
  }

  isHovered(date: NgbDateStruct): boolean {
    return this.hoveredDate !== null &&
           date.year === this.hoveredDate.year &&
           date.month === this.hoveredDate.month &&
           date.day === this.hoveredDate.day;
  }

  isInside(date: NgbDateStruct): boolean {
    // Custom inside range logic
    return false;
  }

  isWeekend(date: NgbDateStruct): boolean {
    const d = new Date(date.year, date.month - 1, date.day);
    return d.getDay() === 0 || d.getDay() === 6;
  }

  isDisabled = (date: NgbDateStruct) => {
    const d = new Date(date.year, date.month - 1, date.day);
    return d.getDay() === 0 || d.getDay() === 6;
  };
}
```

### 4. Typeahead (Autocomplete)

```typescript
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { NgbTypeaheadModule } from '@ng-bootstrap/ng-bootstrap';
import { Observable, of, OperatorFunction } from 'rxjs';
import { debounceTime, distinctUntilChanged, map, switchMap } from 'rxjs/operators';
import { CommonModule } from '@angular/common';

interface Country {
  name: string;
  code: string;
}

@Component({
  selector: 'app-typeahead-example',
  standalone: true,
  imports: [CommonModule, FormsModule, NgbTypeaheadModule],
  template: `
    <!-- Basic Typeahead -->
    <div class="mb-3">
      <label for="typeahead-basic">Search for a state:</label>
      <input
        id="typeahead-basic"
        type="text"
        class="form-control"
        [(ngModel)]="model"
        [ngbTypeahead]="search"
        placeholder="Type to search..."
      />
    </div>

    <!-- Typeahead with HTTP -->
    <div class="mb-3">
      <label for="typeahead-http">Wikipedia search:</label>
      <input
        id="typeahead-http"
        type="text"
        class="form-control"
        [(ngModel)]="model2"
        [ngbTypeahead]="searchWikipedia"
        placeholder="Search Wikipedia..."
      />
    </div>

    <!-- Typeahead with Custom Template -->
    <div class="mb-3">
      <label for="typeahead-template">Search countries:</label>
      <ng-template #rt let-r="result" let-t="term">
        <img
          [src]="'https://flagcdn.com/16x12/' + r.code.toLowerCase() + '.png'"
          class="me-1"
          style="width: 16px"
        />
        <ngb-highlight [result]="r.name" [term]="t"></ngb-highlight>
      </ng-template>
      <input
        id="typeahead-template"
        type="text"
        class="form-control"
        [(ngModel)]="model3"
        [ngbTypeahead]="searchCountries"
        [resultTemplate]="rt"
        [inputFormatter]="formatter"
      />
    </div>

    <!-- Typeahead with Focus -->
    <div class="mb-3">
      <label for="typeahead-focus">Click to see suggestions:</label>
      <input
        id="typeahead-focus"
        type="text"
        class="form-control"
        [(ngModel)]="model4"
        [ngbTypeahead]="search"
        (focus)="focus$.next($any($event).target.value)"
        (click)="click$.next($any($event).target.value)"
        placeholder="Type or click..."
      />
    </div>

    <div>
      <p>Selected: {{ model }}</p>
      <p>Country: {{ model3 | json }}</p>
    </div>
  `
})
export class TypeaheadExampleComponent {
  model: any;
  model2: any;
  model3: any;
  model4: any;

  states = [
    'Alabama', 'Alaska', 'Arizona', 'Arkansas', 'California',
    'Colorado', 'Connecticut', 'Delaware', 'Florida', 'Georgia'
  ];

  countries: Country[] = [
    { name: 'United States', code: 'US' },
    { name: 'United Kingdom', code: 'GB' },
    { name: 'Canada', code: 'CA' },
    { name: 'Germany', code: 'DE' },
    { name: 'France', code: 'FR' }
  ];

  focus$ = new Subject<string>();
  click$ = new Subject<string>();

  search: OperatorFunction<string, readonly string[]> = (text$: Observable<string>) =>
    text$.pipe(
      debounceTime(200),
      distinctUntilChanged(),
      map(term => term.length < 2 ? []
        : this.states.filter(v => v.toLowerCase().indexOf(term.toLowerCase()) > -1).slice(0, 10))
    );

  searchWikipedia: OperatorFunction<string, readonly string[]> = (text$: Observable<string>) =>
    text$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(term =>
        term.length < 2 ? of([]) :
        // In real app, would call HTTP service here
        of(this.states.filter(v => v.toLowerCase().includes(term.toLowerCase())))
      )
    );

  searchCountries: OperatorFunction<string, readonly Country[]> = (text$: Observable<string>) =>
    text$.pipe(
      debounceTime(200),
      map(term => term === '' ? []
        : this.countries.filter(v => v.name.toLowerCase().indexOf(term.toLowerCase()) > -1).slice(0, 10))
    );

  formatter = (country: Country) => country.name;
}
```

### 5. Accordion

```typescript
import { Component } from '@angular/core';
import { NgbAccordionModule } from '@ng-bootstrap/ng-bootstrap';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-accordion-example',
  standalone: true,
  imports: [CommonModule, NgbAccordionModule],
  template: `
    <!-- Basic Accordion -->
    <ngb-accordion #acc="ngbAccordion">
      <ngb-panel id="panel-1">
        <ng-template ngbPanelTitle>
          <span>First panel</span>
        </ng-template>
        <ng-template ngbPanelContent>
          Content for first panel goes here.
        </ng-template>
      </ngb-panel>
      
      <ngb-panel id="panel-2">
        <ng-template ngbPanelTitle>
          <span>Second panel</span>
        </ng-template>
        <ng-template ngbPanelContent>
          Content for second panel goes here.
        </ng-template>
      </ngb-panel>
      
      <ngb-panel id="panel-3" [disabled]="true">
        <ng-template ngbPanelTitle>
          <span>Disabled panel</span>
        </ng-template>
        <ng-template ngbPanelContent>
          This panel is disabled.
        </ng-template>
      </ngb-panel>
    </ngb-accordion>

    <div class="mt-3">
      <button class="btn btn-sm btn-outline-primary me-2" (click)="acc.toggle('panel-1')">
        Toggle first
      </button>
      <button class="btn btn-sm btn-outline-primary me-2" (click)="acc.expand('panel-2')">
        Expand second
      </button>
      <button class="btn btn-sm btn-outline-primary me-2" (click)="acc.collapseAll()">
        Collapse all
      </button>
    </div>

    <!-- Custom Accordion -->
    <ngb-accordion
      [closeOthers]="true"
      activeIds="custom-panel-1"
      class="mt-3"
    >
      <ngb-panel
        *ngFor="let panel of panels"
        [id]="panel.id"
        [disabled]="panel.disabled"
      >
        <ng-template ngbPanelHeader>
          <div class="d-flex align-items-center justify-content-between">
            <button ngbPanelToggle class="btn btn-link p-0">
              {{ panel.title }}
            </button>
            <span class="badge bg-primary">{{ panel.badge }}</span>
          </div>
        </ng-template>
        <ng-template ngbPanelContent>
          {{ panel.content }}
        </ng-template>
      </ngb-panel>
    </ngb-accordion>
  `
})
export class AccordionExampleComponent {
  panels = [
    {
      id: 'custom-panel-1',
      title: 'Panel 1',
      content: 'Content for panel 1',
      badge: '5',
      disabled: false
    },
    {
      id: 'custom-panel-2',
      title: 'Panel 2',
      content: 'Content for panel 2',
      badge: '3',
      disabled: false
    }
  ];
}
```

### 6. Carousel

```typescript
import { Component } from '@angular/core';
import { NgbCarouselModule } from '@ng-bootstrap/ng-bootstrap';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-carousel-example',
  standalone: true,
  imports: [CommonModule, NgbCarouselModule],
  template: `
    <!-- Basic Carousel -->
    <ngb-carousel>
      <ng-template ngbSlide>
        <div class="picsum-img-wrapper">
          <img src="https://picsum.photos/900/500?random=1" alt="Random first slide" />
        </div>
        <div class="carousel-caption">
          <h3>First slide label</h3>
          <p>First slide description</p>
        </div>
      </ng-template>
      <ng-template ngbSlide>
        <div class="picsum-img-wrapper">
          <img src="https://picsum.photos/900/500?random=2" alt="Random second slide" />
        </div>
        <div class="carousel-caption">
          <h3>Second slide label</h3>
          <p>Second slide description</p>
        </div>
      </ng-template>
      <ng-template ngbSlide>
        <div class="picsum-img-wrapper">
          <img src="https://picsum.photos/900/500?random=3" alt="Random third slide" />
        </div>
        <div class="carousel-caption">
          <h3>Third slide label</h3>
          <p>Third slide description</p>
        </div>
      </ng-template>
    </ngb-carousel>

    <!-- Custom Carousel with Controls -->
    <ngb-carousel
      #carousel
      [interval]="3000"
      [pauseOnHover]="true"
      [pauseOnFocus]="true"
      (slide)="onSlide($event)"
      class="mt-4"
    >
      <ng-template ngbSlide *ngFor="let image of images; let i = index">
        <div class="picsum-img-wrapper">
          <img [src]="image.src" [alt]="image.alt" />
        </div>
        <div class="carousel-caption">
          <h3>{{ image.title }}</h3>
          <p>{{ image.description }}</p>
        </div>
      </ng-template>
    </ngb-carousel>

    <div class="mt-3">
      <button class="btn btn-sm btn-outline-primary me-2" (click)="carousel.prev()">
        Previous
      </button>
      <button class="btn btn-sm btn-outline-primary me-2" (click)="carousel.next()">
        Next
      </button>
      <button class="btn btn-sm btn-outline-primary me-2" (click)="carousel.pause()">
        Pause
      </button>
      <button class="btn btn-sm btn-outline-primary" (click)="carousel.cycle()">
        Cycle
      </button>
    </div>
  `,
  styles: [`
    .picsum-img-wrapper {
      position: relative;
      height: 500px;
    }
    .picsum-img-wrapper img {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
  `]
})
export class CarouselExampleComponent {
  images = [
    {
      src: 'https://picsum.photos/900/500?random=4',
      alt: 'Image 1',
      title: 'Slide 1',
      description: 'Description for slide 1'
    },
    {
      src: 'https://picsum.photos/900/500?random=5',
      alt: 'Image 2',
      title: 'Slide 2',
      description: 'Description for slide 2'
    }
  ];

  onSlide(event: any) {
    console.log('Slide event:', event);
  }
}
```

### 7. Dropdown

```typescript
import { Component } from '@angular/core';
import { NgbDropdownModule } from '@ng-bootstrap/ng-bootstrap';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-dropdown-example',
  standalone: true,
  imports: [CommonModule, NgbDropdownModule],
  template: `
    <!-- Basic Dropdown -->
    <div ngbDropdown class="d-inline-block">
      <button class="btn btn-outline-primary" id="dropdownBasic1" ngbDropdownToggle>
        Toggle dropdown
      </button>
      <div ngbDropdownMenu aria-labelledby="dropdownBasic1">
        <button ngbDropdownItem>Action - 1</button>
        <button ngbDropdownItem>Another Action</button>
        <button ngbDropdownItem>Something else</button>
      </div>
    </div>

    <!-- Manual Trigger -->
    <div ngbDropdown class="d-inline-block ms-2" #myDrop="ngbDropdown">
      <button class="btn btn-outline-primary" id="dropdownManual" ngbDropdownAnchor (click)="myDrop.toggle()">
        Manual trigger
      </button>
      <div ngbDropdownMenu aria-labelledby="dropdownManual">
        <button ngbDropdownItem>Action</button>
        <button ngbDropdownItem (click)="$event.stopPropagation()">
          Action (no close)
        </button>
      </div>
    </div>

    <!-- Split Button -->
    <div ngbDropdown class="d-inline-block ms-2">
      <button class="btn btn-primary">
        Split button
      </button>
      <button class="btn btn-primary" ngbDropdownToggle></button>
      <div ngbDropdownMenu>
        <button ngbDropdownItem>Action</button>
        <button ngbDropdownItem>Another Action</button>
      </div>
    </div>

    <!-- Dropup -->
    <div ngbDropdown class="d-inline-block ms-2" placement="top">
      <button class="btn btn-outline-primary" ngbDropdownToggle>
        Dropup
      </button>
      <div ngbDropdownMenu>
        <button ngbDropdownItem>Action</button>
        <button ngbDropdownItem>Another Action</button>
      </div>
    </div>
  `
})
export class DropdownExampleComponent {}
```

## Common Mistakes

1. **Forgetting Bootstrap CSS**: ng-bootstrap requires Bootstrap CSS to be imported.

2. **Using jQuery**: ng-bootstrap doesn't need or use jQuery - remove it if present.

3. **Not handling modal promises**: Modal.result returns a promise that should be handled.

4. **Improper date handling**: NgbDateStruct uses {year, month, day} format, not JavaScript Date.

5. **Not debouncing typeahead**: Always debounce and use distinctUntilChanged for typeahead.

## Best Practices

1. **Use standalone components**: Import specific modules instead of NgbModule
2. **Handle modal cleanup**: Always handle both resolve and reject cases
3. **Accessibility**: Test with keyboard navigation and screen readers
4. **Custom templates**: Leverage ng-template for customization
5. **Lazy loading**: Load ng-bootstrap modules only where needed
6. **Type safety**: Use provided TypeScript interfaces
7. **Responsive design**: Use Bootstrap's grid system
8. **Testing**: ng-bootstrap components are easier to test than jQuery-based Bootstrap

## When to Use ng-bootstrap

**Use ng-bootstrap when:**
- Building Angular applications with Bootstrap design
- Want native Angular components without jQuery
- Need type-safe, template-driven components
- Require accessibility support
- Want official Angular integration
- Need modals, datepickers, tooltips

**Consider alternatives when:**
- Want Material Design (use Angular Material)
- Need more enterprise features (use PrimeNG)
- Building mobile apps (use Ionic)
- Want completely custom design system

## Interview Questions

1. **Q: How does ng-bootstrap differ from regular Bootstrap?**
   A: ng-bootstrap is pure Angular without jQuery dependency, provides type-safe components, uses Angular templates, has better change detection integration, and leverages Angular's dependency injection.

2. **Q: How do you pass data to ng-bootstrap modals?**
   A: Use `componentInstance` property of the modal reference to set properties: `modalRef.componentInstance.propertyName = value`. Access via `@Input()` or direct property in modal component.

3. **Q: How does NgbDatepicker handle dates?**
   A: Uses `NgbDateStruct` interface with {year, month, day} instead of JavaScript Date. This avoids timezone issues and provides better control. Convert with `NgbDate` utility methods.

4. **Q: What's the benefit of typeahead's OperatorFunction approach?**
   A: Provides full RxJS pipeline control for debouncing, HTTP calls, error handling, and transformation. More flexible than simple filtering and integrates with Angular's reactive patterns.

5. **Q: How do you customize ng-bootstrap component styles?**
   A: Override Bootstrap CSS variables, use component-specific classes, leverage ng-template for structure customization, or use ViewEncapsulation.None for deep styling.

## Key Takeaways

1. ng-bootstrap provides Bootstrap components as native Angular directives
2. No jQuery dependency - pure Angular implementation
3. Type-safe with full TypeScript support
4. Modals use promises and component injection for data passing
5. Datepicker uses NgbDateStruct for timezone-safe date handling
6. Typeahead supports async operations with RxJS operators
7. All components support custom templates via ng-template
8. Accessibility features built-in with ARIA support
9. Works with Bootstrap 4 and Bootstrap 5
10. Ideal for Angular apps following Bootstrap design patterns

## Resources

- [ng-bootstrap Documentation](https://ng-bootstrap.github.io/)
- [ng-bootstrap GitHub](https://github.com/ng-bootstrap/ng-bootstrap)
- [Bootstrap Documentation](https://getbootstrap.com/)
- [Migration Guide](https://ng-bootstrap.github.io/#/getting-started)
- [API Reference](https://ng-bootstrap.github.io/#/components/)
