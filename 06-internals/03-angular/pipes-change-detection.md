# Pipes and Change Detection: Pure vs Impure Execution

## Overview

Angular pipes are transformation functions used in templates to format, filter, and transform data before display. The distinction between pure and impure pipes significantly affects when they execute and how they interact with Angular's change detection system.

Understanding pipe execution timing is crucial for application performance, as impure pipes can cause performance bottlenecks if not used carefully, while pure pipes provide automatic memoization and optimization.

## Pure vs Impure Pipes

### Pure Pipes (Default)

```typescript
// Pure pipe - only executes when inputs change
@Pipe({
  name: 'multiply',
  pure: true  // default, can be omitted
})
export class MultiplyPipe implements PipeTransform {
  transform(value: number, factor: number): number {
    console.log('MultiplyPipe executed');
    return value * factor;
  }
}

// Usage in template
@Component({
  template: `
    <div>{{ value | multiply:2 }}</div>
    <button (click)="unrelatedChange()">Change</button>
  `
})
export class Component {
  value = 10;
  
  unrelatedChange() {
    // This does NOT trigger the pipe
    console.log('Button clicked');
  }
  
  changeValue() {
    // This DOES trigger the pipe
    this.value = 20;
  }
}

// Output on first render:
// MultiplyPipe executed
// 20

// Output on unrelatedChange():
// Button clicked
// (Pipe does NOT execute)

// Output on changeValue():
// MultiplyPipe executed
// 40
```

### Impure Pipes

```typescript
// Impure pipe - executes on EVERY change detection cycle
@Pipe({
  name: 'filterBy',
  pure: false
})
export class FilterByPipe implements PipeTransform {
  transform(items: any[], property: string, value: any): any[] {
    console.log('FilterByPipe executed');
    return items.filter(item => item[property] === value);
  }
}

// Usage
@Component({
  template: `
    <div *ngFor="let item of items | filterBy:'active':true">
      {{ item.name }}
    </div>
    <button (click)="unrelatedChange()">Change</button>
  `
})
export class Component {
  items = [
    { name: 'A', active: true },
    { name: 'B', active: false },
    { name: 'C', active: true }
  ];
  
  unrelatedChange() {
    // This DOES trigger the impure pipe
    console.log('Button clicked');
  }
}

// Output on every change detection:
// FilterByPipe executed
// (Even when nothing changed!)
```

## How Pure Pipes Work

### Memoization Implementation

```typescript
// Simplified implementation of pure pipe memoization
class PurePipeCache {
  private lastInput: any;
  private lastArgs: any[];
  private lastResult: any;
  
  transform(pipe: PipeTransform, input: any, args: any[]): any {
    // Check if input or arguments changed
    if (this.inputChanged(input, args)) {
      // Execute pipe transform
      this.lastResult = pipe.transform(input, ...args);
      this.lastInput = input;
      this.lastArgs = args;
    }
    
    // Return cached result
    return this.lastResult;
  }
  
  private inputChanged(input: any, args: any[]): boolean {
    // Check input reference
    if (input !== this.lastInput) {
      return true;
    }
    
    // Check argument references
    if (args.length !== this.lastArgs?.length) {
      return true;
    }
    
    for (let i = 0; i < args.length; i++) {
      if (args[i] !== this.lastArgs[i]) {
        return true;
      }
    }
    
    return false;
  }
}
```

### Generated Code for Pure Pipes

```typescript
// Source template
@Component({
  template: '{{ value | multiply:factor }}'
})
export class Component {
  value = 10;
  factor = 2;
}

// Generated Ivy code (simplified)
function Component_Template(rf, ctx) {
  if (rf & 1) {
    text(0);
  }
  if (rf & 2) {
    // Pure pipe with memoization
    textInterpolate(
      pipeBind2(
        1,           // Pipe index
        0,           // Slot for cache
        ctx.value,   // Input
        ctx.factor   // Argument
      )
    );
  }
}

// pipeBind2 implementation
function pipeBind2(index: number, slot: number, v1: any, v2: any): any {
  const pipe = getPipe(index);
  const cache = getCache(slot);
  
  // Check if inputs changed (reference comparison)
  if (cache.v1 !== v1 || cache.v2 !== v2) {
    // Execute pipe
    const result = pipe.transform(v1, v2);
    
    // Update cache
    cache.v1 = v1;
    cache.v2 = v2;
    cache.result = result;
    
    return result;
  }
  
  // Return cached result
  return cache.result;
}
```

## How Impure Pipes Work

### Execution on Every Change Detection

```typescript
// Impure pipes execute every time
function Component_Template(rf, ctx) {
  if (rf & 1) {
    text(0);
  }
  if (rf & 2) {
    // Impure pipe - NO memoization
    const pipe = getPipe('filterBy');
    const result = pipe.transform(ctx.items, 'active', true);
    textInterpolate(result);
  }
}

// Every change detection cycle runs the pipe
// Even if inputs haven't changed
```

### When to Use Impure Pipes

```typescript
// Valid use case: Async data that changes without reference change
@Pipe({
  name: 'async',
  pure: false
})
export class AsyncPipe implements PipeTransform {
  private subscription: Subscription;
  private latestValue: any = null;
  
  transform(obj: Observable<any> | Promise<any>): any {
    if (!this.subscription) {
      // Subscribe to observable
      this.subscription = this.subscribe(obj);
    }
    
    // Return latest emitted value
    return this.latestValue;
  }
  
  private subscribe(obj: Observable<any>): Subscription {
    return obj.subscribe(value => {
      this.latestValue = value;
      // Trigger change detection
      this.cdr.markForCheck();
    });
  }
  
  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}

// Angular's built-in async pipe is impure
// because Observable emission doesn't change reference
@Component({
  template: '{{ data$ | async }}'
})
export class Component {
  data$ = new BehaviorSubject('initial');
  
  updateData() {
    // Same observable reference, different value
    this.data$.next('updated');
    // async pipe detects this because it's impure
  }
}
```

## Common Pipe Patterns

### Array Transformation (Pure)

```typescript
// Pure pipe for array transformations
@Pipe({
  name: 'sort',
  pure: true
})
export class SortPipe implements PipeTransform {
  transform(array: any[], field: string): any[] {
    if (!array || !field) return array;
    
    // Create new array (immutability)
    return [...array].sort((a, b) => {
      if (a[field] < b[field]) return -1;
      if (a[field] > b[field]) return 1;
      return 0;
    });
  }
}

// Usage - MUST use immutable updates
@Component({
  template: `
    <div *ngFor="let item of items | sort:'name'">
      {{ item.name }}
    </div>
  `
})
export class Component {
  items = [
    { name: 'Charlie' },
    { name: 'Alice' },
    { name: 'Bob' }
  ];
  
  addItem(name: string) {
    // Correct: Creates new array reference
    this.items = [...this.items, { name }];
    // Pipe will execute because reference changed
  }
  
  wrongAddItem(name: string) {
    // Wrong: Mutates array, same reference
    this.items.push({ name });
    // Pipe will NOT execute!
  }
}
```

### Filter Pattern (Common Mistake)

```typescript
// Common mistake: impure filter pipe
@Pipe({
  name: 'filter',
  pure: false  // BAD: Performance killer
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], term: string): any[] {
    return items.filter(item => 
      item.name.toLowerCase().includes(term.toLowerCase())
    );
  }
}

// Better approach: Use component method or property
@Component({
  template: `
    <input [(ngModel)]="searchTerm">
    <div *ngFor="let item of filteredItems">
      {{ item.name }}
    </div>
  `
})
export class BetterComponent {
  items = [/* ... */];
  searchTerm = '';
  
  // Computed property with getter
  get filteredItems() {
    return this.items.filter(item =>
      item.name.toLowerCase().includes(this.searchTerm.toLowerCase())
    );
  }
}

// Best approach: Use signals (Angular 16+)
export class BestComponent {
  items = signal([/* ... */]);
  searchTerm = signal('');
  
  // Computed signal - memoized automatically
  filteredItems = computed(() => {
    const term = this.searchTerm().toLowerCase();
    return this.items().filter(item =>
      item.name.toLowerCase().includes(term)
    );
  });
}
```

### Stateful Pipes

```typescript
// Stateful pipe example: Counter
@Pipe({
  name: 'counter',
  pure: false
})
export class CounterPipe implements PipeTransform {
  private count = 0;
  
  transform(value: any): number {
    // Increments every time it runs
    return ++this.count;
  }
}

// Usage shows how often pipe executes
@Component({
  template: `
    <div>Pipe executed: {{ 'value' | counter }} times</div>
    <button (click)="doSomething()">Click</button>
  `
})
export class Component {
  doSomething() {
    // Every click triggers change detection
    // Counter increments
  }
}
```

## Performance Implications

### Benchmarking Pure vs Impure

```typescript
// Performance test setup
@Component({
  template: `
    <div *ngFor="let item of items | purePipe">{{ item }}</div>
    <div *ngFor="let item of items | impurePipe">{{ item }}</div>
    <button (click)="triggerChange()">Trigger CD</button>
  `
})
export class BenchmarkComponent {
  items = Array.from({ length: 1000 }, (_, i) => i);
  
  triggerChange() {
    // Triggers change detection without changing items
  }
}

@Pipe({ name: 'purePipe', pure: true })
export class PurePipe implements PipeTransform {
  transform(value: any[]): any[] {
    console.time('purePipe');
    const result = value.map(v => v * 2);
    console.timeEnd('purePipe');
    return result;
  }
}

@Pipe({ name: 'impurePipe', pure: false })
export class ImpurePipe implements PipeTransform {
  transform(value: any[]): any[] {
    console.time('impurePipe');
    const result = value.map(v => v * 2);
    console.timeEnd('impurePipe');
    return result;
  }
}

// Results after 10 button clicks:
// purePipe: 0ms (executed 1 time - on init)
// impurePipe: 45ms (executed 11 times - init + 10 CD cycles)
```

### Memory Considerations

```typescript
// Pure pipes with object inputs
@Pipe({ name: 'format', pure: true })
export class FormatPipe implements PipeTransform {
  transform(obj: any): string {
    return JSON.stringify(obj);
  }
}

// Problem: Object mutation doesn't trigger pipe
@Component({
  template: '{{ user | format }}'
})
export class Component {
  user = { name: 'John', age: 30 };
  
  updateAge() {
    // Pure pipe won't detect this
    this.user.age = 31;
  }
  
  correctUpdateAge() {
    // Creates new reference - pipe will execute
    this.user = { ...this.user, age: 31 };
  }
}
```

## Built-in Pipes

### Async Pipe (Impure but Optimized)

```typescript
// Simplified async pipe implementation
@Pipe({
  name: 'async',
  pure: false
})
export class AsyncPipe implements PipeTransform, OnDestroy {
  private latestValue: any = null;
  private subscription: Subscription | null = null;
  private obj: Observable<any> | Promise<any> | null = null;
  
  constructor(private cdr: ChangeDetectorRef) {}
  
  transform(obj: Observable<any> | Promise<any> | null): any {
    if (!obj) {
      return null;
    }
    
    // If observable changed, unsubscribe from old one
    if (obj !== this.obj) {
      this.dispose();
      this.obj = obj;
      this.subscribe(obj);
    }
    
    return this.latestValue;
  }
  
  private subscribe(obj: Observable<any> | Promise<any>): void {
    this.subscription = this.toObservable(obj).subscribe({
      next: (value) => {
        this.latestValue = value;
        // Manually trigger change detection
        this.cdr.markForCheck();
      }
    });
  }
  
  private toObservable(obj: Observable<any> | Promise<any>): Observable<any> {
    if (obj instanceof Observable) {
      return obj;
    }
    return from(obj);
  }
  
  private dispose(): void {
    this.subscription?.unsubscribe();
    this.subscription = null;
  }
  
  ngOnDestroy(): void {
    this.dispose();
  }
}

// Why async pipe is impure:
// The Observable reference doesn't change when it emits
// But the pipe needs to return the new emitted value
```

### Date Pipe (Pure)

```typescript
// Date pipe is pure - only re-formats when input changes
@Component({
  template: `
    <div>{{ currentDate | date:'short' }}</div>
    <button (click)="updateDate()">Update</button>
  `
})
export class Component {
  currentDate = new Date();
  
  updateDate() {
    // Creates new Date object
    this.currentDate = new Date();
    // DatePipe will execute
  }
  
  wrongUpdate() {
    // Mutates same Date object
    this.currentDate.setHours(12);
    // DatePipe will NOT execute (same reference)
  }
}
```

### Currency Pipe (Pure)

```typescript
// Currency pipe is pure and locale-aware
@Component({
  template: `
    <div>{{ price | currency:'USD':'symbol':'1.2-2' }}</div>
  `
})
export class Component {
  price = 1234.56;
  
  updatePrice(newPrice: number) {
    // Primitive value change triggers pipe
    this.price = newPrice;
  }
}

// Pipe parameters are also checked:
// {{ price | currency:currencyCode }}
// Changing currencyCode triggers pipe re-execution
```

## Advanced Patterns

### Chaining Pipes

```typescript
// Multiple pipes execute left to right
@Component({
  template: `
    {{ items | filter:'active' | sort:'name' | slice:0:10 }}
  `
})
export class Component {
  items = [/* array */];
}

// Execution order:
// 1. filter pipe
// 2. sort pipe (receives filtered array)
// 3. slice pipe (receives sorted array)

// Performance consideration:
// If items reference doesn't change:
// - Pure pipes: None execute
// - If items changes: All three execute in sequence
```

### Custom Memoization

```typescript
// Pipe with custom caching strategy
@Pipe({
  name: 'expensiveOperation',
  pure: true
})
export class ExpensivePipe implements PipeTransform {
  private cache = new Map<string, any>();
  
  transform(value: any, ...args: any[]): any {
    // Create cache key from input and args
    const cacheKey = this.getCacheKey(value, args);
    
    // Check cache
    if (this.cache.has(cacheKey)) {
      console.log('Cache hit');
      return this.cache.get(cacheKey);
    }
    
    // Expensive operation
    console.log('Cache miss - computing');
    const result = this.expensiveComputation(value, ...args);
    
    // Store in cache
    this.cache.set(cacheKey, result);
    
    // Limit cache size
    if (this.cache.size > 100) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    return result;
  }
  
  private getCacheKey(value: any, args: any[]): string {
    return JSON.stringify({ value, args });
  }
  
  private expensiveComputation(value: any, ...args: any[]): any {
    // Heavy computation here
    return /* result */;
  }
}
```

## Common Misconceptions

### "Pure pipes never re-execute"

**Reality**: Pure pipes re-execute when input or argument references change:

```typescript
items = [1, 2, 3];

// This triggers pipe:
this.items = [...this.items, 4];

// This doesn't:
this.items.push(4);
```

### "Impure pipes are always bad"

**Reality**: Impure pipes are necessary for async data (async pipe) and certain stateful operations. The key is using them judiciously.

### "Getters are the same as pipes"

**Reality**: Getters execute on every change detection like impure pipes, but without memoization:

```typescript
// Getter - executes every CD
get filtered() {
  return this.items.filter(/* ... */);
}

// Pipe - memoized if pure
{{ items | filter }}
```

## Performance Implications

### Pure Pipe Optimization

```
Scenario: 1000-item list with pure pipe
- Initial render: 1 pipe execution
- 100 change detection cycles: 0 additional executions
- Change list reference: 1 pipe execution
Total: 2 executions

Same scenario with impure pipe:
Total: 102 executions (50x more!)
```

### Guidelines

```typescript
// DO: Use pure pipes for transformations
{{ data | myPurePipe }}

// DON'T: Use impure pipes for filtering/sorting
{{ items | impureFilter }}  // Bad

// DO: Use component logic instead
get filteredItems() { return /* ... */; }

// BETTER: Use signals (Angular 16+)
filteredItems = computed(() => /* ... */);
```

## Interview Questions

### Q1: What's the difference between pure and impure pipes?

**Answer**: Pure pipes (default) only execute when input value or argument references change. They're memoized and use reference comparison for change detection. Impure pipes (`pure: false`) execute on every change detection cycle, regardless of whether inputs changed. Pure pipes are much more performant but require immutable data patterns. Impure pipes are necessary for async data (like the async pipe) or when pipe logic depends on external state that changes without reference changes.

### Q2: When would you use an impure pipe?

**Answer**: Use impure pipes sparingly, only when:
1. **Async data**: Observable/Promise values (async pipe)
2. **External state**: Pipe depends on data outside its inputs (e.g., locale changes, authentication state)
3. **Stateful operations**: Need to track internal state across executions

Most use cases should use pure pipes with immutable data patterns, or component logic/signals instead of pipes. Impure pipes have significant performance implications since they run on every change detection cycle.

### Q3: How does Angular optimize pure pipe execution?

**Answer**: Angular implements memoization for pure pipes using the `pipeBind` instructions. Generated code stores the last input and arguments in a cache slot. On each change detection, Angular compares new inputs with cached values using reference equality (`===`). If nothing changed, it returns the cached result without executing the pipe. This optimization is automatic and happens at compile time - Ivy generates different code for pure vs impure pipes.

### Q4: Why do pure pipes require immutable data patterns?

**Answer**: Pure pipes use reference comparison to detect changes. If you mutate an object/array, the reference stays the same, so the pipe won't re-execute:

```typescript
// Won't trigger pure pipe
this.items.push(newItem);

// Will trigger pure pipe
this.items = [...this.items, newItem];
```

This aligns with modern reactive programming patterns and enables Angular's optimization. It also makes code more predictable - you can see exactly when pipes will execute by looking at where references change.

### Q5: What happens if you use a getter in a template instead of a pipe?

**Answer**: Getters execute on every change detection cycle, like impure pipes, but without memoization:

```typescript
// Getter - runs every CD
get filtered() {
  console.log('Filtering');
  return this.items.filter(/* ... */);
}
// {{ filtered }}

// Pure pipe - memoized
// {{ items | filter }}
```

Getters are convenient but can cause performance issues. For expensive operations, use computed signals (Angular 16+) or pure pipes. Signals provide automatic memoization and only recompute when dependencies change, similar to pure pipes but more powerful.

### Q6: How does the async pipe work internally?

**Answer**: The async pipe subscribes to Observables/Promises and returns the latest emitted value:

1. On first execution, subscribes to the Observable
2. Stores subscription and latest value
3. When Observable emits, updates value and calls `ChangeDetectorRef.markForCheck()`
4. Returns latest value on each change detection
5. Unsubscribes on destroy

It's impure because the Observable reference doesn't change when it emits, but the pipe needs to return the new value. The pipe manually triggers change detection on emissions, making it efficient despite being impure.

### Q7: Can you chain multiple pipes? What's the performance impact?

**Answer**: Yes, pipes chain left to right:
```typescript
{{ items | filter:'active' | sort:'name' | slice:0:10 }}
```

Execution order: filter → sort → slice. Each pipe receives the previous pipe's output.

Performance: If all pipes are pure and the input reference doesn't change, NO pipes execute (memoization). If input changes, ALL pipes execute in sequence. Be cautious with expensive operations in chained pipes - consider computing in component logic or using signals instead. Each pipe adds overhead, so limit chains to 2-3 pipes maximum.

### Q8: Should you use pipes for filtering and sorting lists?

**Answer**: **Generally no**, especially not with impure pipes:

**Bad approach** (impure pipe):
```typescript
{{ items | filter:searchTerm }}  // Runs every CD cycle
```

**Better approaches**:
1. **Component method/getter**: Direct, no pipe overhead
2. **Computed signal** (best): Memoized, reactive
```typescript
filteredItems = computed(() => 
  this.items().filter(/* ... */)
);
```

3. **Pure pipe with immutability**: Works but requires discipline
```typescript
this.items = this.items.filter(/* ... */);
```

Pipes are great for formatting (date, currency, etc.) but filtering/sorting is better handled in component logic or with signals for optimal performance and maintainability.

## Key Takeaways

1. Pure pipes (default) only execute when input/argument references change
2. Impure pipes execute on every change detection cycle
3. Pure pipes are automatically memoized by Angular for performance
4. Pure pipes require immutable data patterns (new references for changes)
5. Async pipe is impure but necessary for reactive data streams
6. Avoid impure pipes for filtering/sorting - use component logic or signals
7. Chained pipes execute left to right, all re-execute if input changes
8. Getters execute like impure pipes but without memoization

## Resources

- [Angular Pipes Guide](https://angular.dev/guide/pipes)
- [Pure vs Impure Pipes](https://angular.io/guide/pipes#pure-and-impure-pipes)
- [Custom Pipes](https://angular.dev/guide/pipes/custom-data-transformation)
- [Async Pipe Source Code](https://github.com/angular/angular/blob/main/packages/common/src/pipes/async_pipe.ts)
- [Pipe Performance Optimization](https://blog.angular.io/angular-performance-guide-pipes-7c13e9c1c3d0)
- [Change Detection and Pipes](https://blog.angularindepth.com/angular-pipes-and-change-detection-8c5c5b8d7e7e)
- [When to Use Impure Pipes](https://stackoverflow.com/questions/34456430/what-are-impure-pipes-in-angular)
