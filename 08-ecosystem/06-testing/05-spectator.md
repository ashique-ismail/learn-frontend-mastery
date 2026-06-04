# Spectator (Angular Testing)

## What It Is

Spectator is a testing library for Angular that reduces boilerplate in unit and integration tests. It wraps `TestBed` with a more ergonomic API.

```bash
npm install -D @ngneat/spectator
```

---

## Component Testing

```ts
import { createComponentFactory, Spectator } from '@ngneat/spectator';
import { UserCardComponent } from './user-card.component';
import { UserService } from './user.service';

describe('UserCardComponent', () => {
  let spectator: Spectator<UserCardComponent>;

  const createComponent = createComponentFactory({
    component: UserCardComponent,
    providers: [mockProvider(UserService)],  // auto-mock
    imports: [CommonModule],
    detectChanges: false, // manual control
  });

  beforeEach(() => {
    spectator = createComponent({
      props: { user: { id: '1', name: 'Alice', email: 'alice@example.com' } }
    });
    spectator.detectChanges();
  });

  it('displays user name', () => {
    expect(spectator.query('.user-name')).toHaveText('Alice');
  });

  it('emits on delete click', () => {
    const onDelete = jest.fn();
    spectator.component.deleteClicked.subscribe(onDelete);

    spectator.click('.delete-btn');

    expect(onDelete).toHaveBeenCalled();
  });

  it('calls service on save', () => {
    const userService = spectator.inject(UserService);
    spectator.click('.save-btn');
    expect(userService.save).toHaveBeenCalledWith(spectator.component.user);
  });
});
```

---

## Query Helpers

```ts
// Single element query
spectator.query('.btn');                          // by CSS selector
spectator.query(UserAvatarComponent);             // by component type
spectator.query<HTMLInputElement>('#email');      // typed
spectator.query(byText('Submit'));                 // by text content

// Multiple elements
spectator.queryAll('.item');
spectator.queryAll(ListItemComponent);

// Debug if not found
expect(spectator.query('.header')).toBeTruthy();
expect(spectator.query('.hidden-section')).toBeNull();
```

---

## Interaction Helpers

```ts
// Click
spectator.click('.submit-btn');
spectator.click(spectator.query(ButtonComponent));

// Type in input
spectator.typeInElement('john@example.com', '#email');

// Keyboard events
spectator.keyboard.pressEnter('#search');
spectator.keyboard.pressEscape();

// Focus/Blur
spectator.focus('#input');
spectator.blur('#input');

// Dispatch custom event
spectator.dispatchMouseEvent('.draggable', 'mousedown');
spectator.dispatchKeyboardEvent('#input', 'keydown', 'ArrowDown');
```

---

## Service Testing

```ts
import { createServiceFactory, SpectatorService } from '@ngneat/spectator';
import { HttpClientTestingModule } from '@angular/common/http/testing';

describe('UserService', () => {
  let spectator: SpectatorService<UserService>;

  const createService = createServiceFactory({
    service: UserService,
    imports: [HttpClientTestingModule],
  });

  beforeEach(() => { spectator = createService(); });

  it('fetches users', () => {
    const http = spectator.inject(HttpTestingController);
    let result: User[];

    spectator.service.getUsers().subscribe(users => { result = users; });

    const req = http.expectOne('/api/users');
    req.flush([{ id: '1', name: 'Alice' }]);

    expect(result!).toHaveLength(1);
    expect(result![0].name).toBe('Alice');

    http.verify(); // ensure no unexpected requests
  });
});
```

---

## Directive Testing

```ts
import { createDirectiveFactory } from '@ngneat/spectator';
import { HighlightDirective } from './highlight.directive';

describe('HighlightDirective', () => {
  const createDirective = createDirectiveFactory({
    directive: HighlightDirective,
    template: `<div highlight color="yellow">Hello</div>`,
  });

  it('applies background color', () => {
    const spectator = createDirective();
    expect(spectator.element).toHaveStyle({ backgroundColor: 'yellow' });
  });
});
```

---

## `mockProvider` — Auto-mocking

```ts
import { mockProvider } from '@ngneat/spectator';
import { jest } from '@jest/globals';

// Creates a mock where all methods are jest.fn()
const createComponent = createComponentFactory({
  component: MyComponent,
  providers: [
    mockProvider(UserService),          // all methods are jest.fn()
    mockProvider(RouterService, {       // with custom implementation
      navigate: jest.fn().mockResolvedValue(true)
    }),
  ],
});

// Access the mock in tests
const userService = spectator.inject(UserService);
userService.getUser.mockReturnValue(of({ name: 'Alice' }));
```

---

## Custom Matchers

Spectator adds jest-dom-like matchers:

```ts
expect(spectator.query('.btn')).toHaveText('Submit');
expect(spectator.query('.btn')).toHaveClass('active');
expect(spectator.query('.btn')).toHaveAttribute('disabled');
expect(spectator.query('.btn')).toBeDisabled();
expect(spectator.query('.section')).toBeVisible();
expect(spectator.query('.hidden')).toBeHidden();
expect(spectator.query<HTMLInputElement>('#name')).toHaveValue('Alice');
```

---

## Comparison with Plain TestBed

```ts
// Without Spectator (verbose)
beforeEach(() => {
  TestBed.configureTestingModule({
    declarations: [UserCardComponent],
    providers: [{ provide: UserService, useClass: MockUserService }],
  });
  fixture = TestBed.createComponent(UserCardComponent);
  component = fixture.componentInstance;
  fixture.detectChanges();
});

const button = fixture.debugElement.query(By.css('.delete-btn'));
button.triggerEventHandler('click', {});

// With Spectator (concise)
spectator.click('.delete-btn');
```

---

## Common Interview Questions

**Q: When would you use Spectator vs plain TestBed?**
Spectator for any non-trivial component or service test — the query helpers, interaction shortcuts, and auto-mocking save significant boilerplate. Plain TestBed for micro-tests or when you need very precise control over the test environment.

**Q: How does `mockProvider` work?**
It uses `jest-auto-spies` under the hood to create a mock class where every method is replaced with a `jest.fn()`. Methods that return Observables get mocks that return `EMPTY` by default. You can override in the factory options or in individual tests.
