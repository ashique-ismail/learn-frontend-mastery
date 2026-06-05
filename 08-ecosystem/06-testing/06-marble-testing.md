# Marble Testing for RxJS

## The Idea

**In plain English:** Marble testing is a way to test code that produces values over time by drawing the timeline as a simple text diagram, where each character represents a tiny slice of time and letters represent the values that arrive. Instead of waiting for real time to pass, you describe what should happen and when, and the computer checks if your code matches.

**Real-world analogy:** Imagine a music teacher writing out a rhythm on paper using dots and dashes to show when notes should play — and then listening to a student play and checking if the timing matches the sheet exactly.

- The sheet of dots and dashes = the marble string (e.g. `'-a-b-c-|'`)
- Each dash on the sheet = one frame of virtual time passing with no note
- Each letter on the sheet = a value being emitted by the Observable at that moment
- The teacher's ear checking the student = the test assertion comparing actual vs expected output

---

## What Marble Testing Is

Marble testing is a way to visualize and test asynchronous Observable sequences using string diagrams. Each character represents a frame of virtual time.

```
'-a-b-c-|'
 ^ ^ ^ ^
 | | | └── | = Observable completed
 | | └──── c emitted at frame 5
 | └────── b emitted at frame 3
 └──────── - = one frame of time, 'a' emitted at frame 1
```

---

## Marble Syntax

| Symbol | Meaning |
|---|---|
| `-` | One frame of virtual time (10ms by default) |
| `a`, `b`, `c` | Value emission (letters represent values) |
| `|` | Observable completion |
| `#` | Observable error |
| `^` | Subscription point (for hot observables) |
| `!` | Unsubscription point |
| `(ab)` | Multiple values emitted in the same frame |
| `---#` | Error after 3 frames |
| `--5s--|` | 5 second wait (time progression shorthand) |

---

## Setup with TestScheduler

```ts
import { TestScheduler } from 'rxjs/testing';
import { map, filter, debounceTime, switchMap } from 'rxjs/operators';

describe('Observable marble tests', () => {
  let testScheduler: TestScheduler;

  beforeEach(() => {
    testScheduler = new TestScheduler((actual, expected) => {
      expect(actual).toEqual(expected);
    });
  });
```

---

## Basic Marble Test

```ts
it('should map values', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('-a-b-c-|', { a: 1, b: 2, c: 3 });
    const result$ = source$.pipe(map(x => x * 10));

    expectObservable(result$).toBe('-a-b-c-|', { a: 10, b: 20, c: 30 });
  });
});

it('should filter odd numbers', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a-b-c-d-|', { a: 1, b: 2, c: 3, d: 4 });
    const result$ = source$.pipe(filter(x => x % 2 === 0));

    expectObservable(result$).toBe('----b---d-|', { b: 2, d: 4 });
  });
});
```

---

## Testing debounceTime

```ts
it('should debounce rapid emissions', () => {
  testScheduler.run(({ cold, expectObservable }) => {
    const source$ = cold('--a-b-c---------d-|', { a: 'a', b: 'b', c: 'c', d: 'd' });
    const result$ = source$.pipe(debounceTime(50)); // 50ms debounce

    // Only c and d emit (a and b are cancelled by rapid succession)
    expectObservable(result$).toBe('-----------c-----d-|', { c: 'c', d: 'd' });
  });
});
```

---

## Testing NgRx Effects

```ts
import { provideMockActions } from '@ngrx/effects/testing';
import { cold, hot } from 'jasmine-marbles'; // jest-marbles or rxjs/testing

describe('UserEffects', () => {
  let actions$: Observable<any>;
  let effects: UserEffects;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        UserEffects,
        provideMockActions(() => actions$),
        { provide: UserService, useValue: jasmine.createSpyObj('UserService', ['getUser']) },
      ]
    });
    effects = TestBed.inject(UserEffects);
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });

  it('should dispatch loadUserSuccess on successful API call', () => {
    const user = { id: '1', name: 'Alice' };
    const action = UserActions.loadUser({ id: '1' });
    const outcome = UserActions.loadUserSuccess({ user });

    actions$ = hot('-a', { a: action });
    const response = cold('-b|', { b: user });
    userService.getUser.and.returnValue(response);

    const expected = cold('--c', { c: outcome });
    expect(effects.loadUser$).toBeObservable(expected);
  });

  it('should dispatch loadUserFailure on error', () => {
    const action = UserActions.loadUser({ id: '1' });
    const error = new Error('Network error');
    const outcome = UserActions.loadUserFailure({ error: error.message });

    actions$ = hot('-a', { a: action });
    const response = cold('-#', {}, error);
    userService.getUser.and.returnValue(response);

    const expected = cold('--c', { c: outcome });
    expect(effects.loadUser$).toBeObservable(expected);
  });
});
```

---

## Testing switchMap Cancellation

```ts
it('should cancel previous request on new action', () => {
  testScheduler.run(({ cold, hot, expectObservable }) => {
    const searchAction = (query: string) => SearchActions.query({ query });

    // Two quick searches — second should cancel first
    const actions$ = hot('-a--b-', {
      a: searchAction('react'),
      b: searchAction('angular'),
    });

    // First search would take 5 frames, but gets cancelled
    const searchResponse = cold('-----r|', { r: [] });
    mockSearchService.search.mockReturnValue(searchResponse);

    // Only 'angular' search completes (react is cancelled at frame 4)
    const expected = '---------r|';
    expectObservable(effects.search$).toBe(expected, { r: SearchActions.resultsLoaded({ results: [] }) });
  });
});
```

---

## `jest-marbles` Library

```bash
npm install -D jest-marbles
```

```ts
import { cold, hot } from 'jest-marbles';

it('using jest-marbles syntax', () => {
  const source$ = cold('--a--b-|', { a: 1, b: 2 });
  const expected = cold('--a--b-|', { a: 10, b: 20 });
  expect(source$.pipe(map(x => x * 10))).toBeObservable(expected);
});
```

---

## Common Interview Questions

**Q: Why use marble testing over `subscribe` + `done()`?**
Marble tests are synchronous (virtual time) and declarative — you express the expected sequence as a string. Subscribe-based tests require manual timeout handling, are asynchronous, and are harder to read. Marble tests also make cancellation and timing behavior explicit.

**Q: What does `hot` vs `cold` mean in marble testing?**
`cold` creates an Observable that starts when subscribed to (each subscriber gets its own timeline starting from `-`). `hot` creates an Observable that exists independently of subscriptions — subscribers joining in the middle miss earlier emissions. Use `cold` for HTTP responses, `hot` for actions stream, DOM events, Subjects.

**Q: How do you test an Observable that uses `delay` or `timer`?**
Use `testScheduler.run()` which virtualizes time. Inside it, `delay(1000)` runs in virtual time — the test completes synchronously without actually waiting 1 second. This is the core power of marble testing.
