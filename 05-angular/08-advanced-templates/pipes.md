# Pipes in Angular

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Pipes](#understanding-pipes)
3. [Built-in Pipes](#built-in-pipes)
4. [Pure vs Impure Pipes](#pure-vs-impure-pipes)
5. [Creating Custom Pipes](#creating-custom-pipes)
6. [Async Pipe Deep Dive](#async-pipe-deep-dive)
7. [Transform Chains](#transform-chains)
8. [Caching Strategies](#caching-strategies)
9. [Stateful Pipes](#stateful-pipes)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)
13. [Key Takeaways](#key-takeaways)
14. [Resources](#resources)

## Introduction

Pipes are a powerful feature in Angular that transform displayed values within a template. They take data as input and transform it into a desired output format. Pipes are ideal for formatting dates, numbers, currencies, and performing other data transformations without modifying the underlying data.

This comprehensive guide covers everything from basic pipe usage to advanced patterns including pure vs impure pipes, custom pipe creation, the async pipe, transform chains, and caching strategies.

## Understanding Pipes

Pipes transform template data using the pipe operator (`|`). They're pure functions that take an input value and return a transformed output.

### Basic Pipe Syntax

```typescript
import { Component } from '@angular/core';
import { UpperCasePipe, LowerCasePipe, DatePipe } from '@angular/common';

@Component({
  selector: 'app-basic-pipes',
  standalone: true,
  imports: [UpperCasePipe, LowerCasePipe, DatePipe],
  template: `
    <h2>Basic Pipe Usage</h2>
    
    <!-- Uppercase transformation -->
    <p>{{ 'hello world' | uppercase }}</p>
    <!-- Output: HELLO WORLD -->
    
    <!-- Lowercase transformation -->
    <p>{{ 'HELLO WORLD' | lowercase }}</p>
    <!-- Output: hello world -->
    
    <!-- Date formatting -->
    <p>{{ today | date }}</p>
    <!-- Output: Jan 1, 2026 -->
    
    <!-- Date with parameters -->
    <p>{{ today | date:'fullDate' }}</p>
    <!-- Output: Friday, January 1, 2026 -->
  `
})
export class BasicPipesComponent {
  today = new Date();
}
```

### Pipe Parameters

```typescript
import { Component } from '@angular/core';
import { DatePipe, CurrencyPipe, DecimalPipe } from '@angular/common';

@Component({
  selector: 'app-pipe-parameters',
  standalone: true,
  imports: [DatePipe, CurrencyPipe, DecimalPipe],
  template: `
    <!-- Single parameter -->
    <p>{{ price | currency:'EUR' }}</p>
    
    <!-- Multiple parameters -->
    <p>{{ price | currency:'EUR':'symbol':'1.2-2' }}</p>
    
    <!-- Date with format and timezone -->
    <p>{{ today | date:'short':'UTC' }}</p>
    
    <!-- Number formatting -->
    <p>{{ 1234.5678 | number:'1.2-4' }}</p>
    <!-- Output: 1,234.5678 -->
  `
})
export class PipeParametersComponent {
  price = 99.99;
  today = new Date();
}
```

## Built-in Pipes

### Common Pipes

```typescript
import { Component } from '@angular/core';
import { 
  DatePipe, 
  UpperCasePipe, 
  LowerCasePipe, 
  TitleCasePipe,
  CurrencyPipe, 
  DecimalPipe, 
  PercentPipe,
  JsonPipe,
  SlicePipe
} from '@angular/common';

@Component({
  selector: 'app-built-in-pipes',
  standalone: true,
  imports: [
    DatePipe,
    UpperCasePipe,
    LowerCasePipe,
    TitleCasePipe,
    CurrencyPipe,
    DecimalPipe,
    PercentPipe,
    JsonPipe,
    SlicePipe
  ],
  template: `
    <h2>Date Pipes</h2>
    <p>{{ today | date }}</p>
    <p>{{ today | date:'short' }}</p>
    <p>{{ today | date:'medium' }}</p>
    <p>{{ today | date:'long' }}</p>
    <p>{{ today | date:'full' }}</p>
    <p>{{ today | date:'shortDate' }}</p>
    <p>{{ today | date:'mediumTime' }}</p>
    <p>{{ today | date:'yyyy-MM-dd HH:mm:ss' }}</p>
    
    <h2>Case Pipes</h2>
    <p>{{ text | uppercase }}</p>
    <p>{{ text | lowercase }}</p>
    <p>{{ text | titlecase }}</p>
    
    <h2>Number Pipes</h2>
    <p>{{ price | currency }}</p>
    <p>{{ price | currency:'EUR' }}</p>
    <p>{{ price | currency:'GBP':'code' }}</p>
    <p>{{ 0.259 | percent }}</p>
    <p>{{ 0.259 | percent:'2.2-2' }}</p>
    <p>{{ 1234567.89 | number }}</p>
    <p>{{ 1234567.89 | number:'1.0-0' }}</p>
    
    <h2>Slice Pipe</h2>
    <p>{{ text | slice:0:5 }}</p>
    <p>{{ items | slice:1:3 | json }}</p>
    
    <h2>JSON Pipe</h2>
    <pre>{{ user | json }}</pre>
  `
})
export class BuiltInPipesComponent {
  today = new Date();
  text = 'hello world';
  price = 99.99;
  items = [1, 2, 3, 4, 5];
  user = { name: 'John', age: 30 };
}
```

### KeyValue Pipe

```typescript
import { Component } from '@angular/core';
import { KeyValuePipe } from '@angular/common';

@Component({
  selector: 'app-keyvalue-example',
  standalone: true,
  imports: [KeyValuePipe],
  template: `
    <h2>Object iteration</h2>
    <div *ngFor="let item of object | keyvalue">
      {{ item.key }}: {{ item.value }}
    </div>
    
    <h2>Map iteration</h2>
    <div *ngFor="let item of map | keyvalue">
      {{ item.key }}: {{ item.value }}
    </div>
    
    <h2>Custom key comparator</h2>
    <div *ngFor="let item of object | keyvalue:reverseSort">
      {{ item.key }}: {{ item.value }}
    </div>
  `
})
export class KeyValueExampleComponent {
  object = { z: 'last', a: 'first', m: 'middle' };
  map = new Map([
    ['key1', 'value1'],
    ['key2', 'value2'],
    ['key3', 'value3']
  ]);
  
  reverseSort = (a: any, b: any) => {
    return b.key.localeCompare(a.key);
  };
}
```

## Pure vs Impure Pipes

### Pure Pipes

Pure pipes only execute when Angular detects a pure change to the input value (primitive value change or object reference change). They're the default and most performant.

```typescript
import { Pipe, PipeTransform } from '@angular/core';

// Pure pipe (default)
@Pipe({
  name: 'purePipe',
  standalone: true,
  pure: true // This is the default
})
export class PurePipe implements PipeTransform {
  transform(value: string): string {
    console.log('Pure pipe executed');
    return value.toUpperCase();
  }
}

@Component({
  selector: 'app-pure-example',
  standalone: true,
  imports: [PurePipe],
  template: `
    <p>{{ text | purePipe }}</p>
    <button (click)="changeText()">Change Text</button>
    <button (click)="mutateText()">Mutate Text</button>
  `
})
export class PureExampleComponent {
  text = 'hello';
  
  changeText() {
    // Creates new reference - pipe executes
    this.text = 'world';
  }
  
  mutateText() {
    // Same reference - pipe doesn't execute
    this.text = this.text + '!';
  }
}
```

### Impure Pipes

Impure pipes execute on every change detection cycle, regardless of whether the input changed. Use sparingly as they can impact performance.

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'impurePipe',
  standalone: true,
  pure: false // Impure pipe
})
export class ImpurePipe implements PipeTransform {
  transform(value: any[]): any[] {
    console.log('Impure pipe executed');
    return value.filter(item => item.active);
  }
}

@Component({
  selector: 'app-impure-example',
  standalone: true,
  imports: [ImpurePipe],
  template: `
    <div *ngFor="let item of items | impurePipe">
      {{ item.name }}
    </div>
    <button (click)="addItem()">Add Item</button>
    <button (click)="toggleItem()">Toggle First Item</button>
  `
})
export class ImpureExampleComponent {
  items = [
    { name: 'Item 1', active: true },
    { name: 'Item 2', active: false },
    { name: 'Item 3', active: true }
  ];
  
  addItem() {
    // Impure pipe detects this
    this.items.push({ name: 'New Item', active: true });
  }
  
  toggleItem() {
    // Impure pipe detects this mutation
    this.items[0].active = !this.items[0].active;
  }
}
```

### Pure Pipe with Immutable Data

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'filterActive',
  standalone: true,
  pure: true // Pure pipe
})
export class FilterActivePipe implements PipeTransform {
  transform(items: any[]): any[] {
    console.log('Pure filter executed');
    return items.filter(item => item.active);
  }
}

@Component({
  selector: 'app-immutable-example',
  standalone: true,
  imports: [FilterActivePipe],
  template: `
    <div *ngFor="let item of items | filterActive">
      {{ item.name }}
    </div>
    <button (click)="addItem()">Add Item</button>
    <button (click)="toggleItem()">Toggle First Item</button>
  `
})
export class ImmutableExampleComponent {
  items = [
    { name: 'Item 1', active: true },
    { name: 'Item 2', active: false },
    { name: 'Item 3', active: true }
  ];
  
  addItem() {
    // Create new array reference
    this.items = [...this.items, { name: 'New Item', active: true }];
  }
  
  toggleItem() {
    // Create new array and object references
    this.items = [
      { ...this.items[0], active: !this.items[0].active },
      ...this.items.slice(1)
    ];
  }
}
```

## Creating Custom Pipes

### Simple Custom Pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'exponential',
  standalone: true
})
export class ExponentialPipe implements PipeTransform {
  transform(value: number, exponent = 1): number {
    return Math.pow(value, exponent);
  }
}

// Usage
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ExponentialPipe],
  template: `
    <p>{{ 2 | exponential }}</p>        <!-- 2^1 = 2 -->
    <p>{{ 2 | exponential:2 }}</p>      <!-- 2^2 = 4 -->
    <p>{{ 2 | exponential:3 }}</p>      <!-- 2^3 = 8 -->
  `
})
export class AppComponent {}
```

### String Manipulation Pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 50, trail = '...'): string {
    if (!value) return '';
    
    if (value.length <= limit) {
      return value;
    }
    
    return value.substring(0, limit) + trail;
  }
}

@Pipe({
  name: 'highlight',
  standalone: true
})
export class HighlightPipe implements PipeTransform {
  transform(value: string, search: string): string {
    if (!search) return value;
    
    const regex = new RegExp(search, 'gi');
    return value.replace(regex, (match) => `<mark>${match}</mark>`);
  }
}

@Pipe({
  name: 'initials',
  standalone: true
})
export class InitialsPipe implements PipeTransform {
  transform(value: string): string {
    return value
      .split(' ')
      .map(word => word[0])
      .join('')
      .toUpperCase();
  }
}

// Usage
@Component({
  selector: 'app-string-pipes',
  standalone: true,
  imports: [TruncatePipe, HighlightPipe, InitialsPipe],
  template: `
    <p>{{ longText | truncate:30 }}</p>
    <p [innerHTML]="text | highlight:searchTerm"></p>
    <p>{{ fullName | initials }}</p>
  `
})
export class StringPipesComponent {
  longText = 'This is a very long text that will be truncated';
  text = 'Hello World, this is a test';
  searchTerm = 'test';
  fullName = 'John Michael Doe';
}
```

### Array Manipulation Pipes

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'filter',
  standalone: true,
  pure: false // Impure to detect array mutations
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], searchText: string, property?: string): any[] {
    if (!items || !searchText) return items;
    
    searchText = searchText.toLowerCase();
    
    return items.filter(item => {
      const value = property ? item[property] : item;
      return String(value).toLowerCase().includes(searchText);
    });
  }
}

@Pipe({
  name: 'sort',
  standalone: true,
  pure: false
})
export class SortPipe implements PipeTransform {
  transform(items: any[], property: string, order: 'asc' | 'desc' = 'asc'): any[] {
    if (!items || !property) return items;
    
    return [...items].sort((a, b) => {
      const valueA = a[property];
      const valueB = b[property];
      
      let comparison = 0;
      if (valueA > valueB) {
        comparison = 1;
      } else if (valueA < valueB) {
        comparison = -1;
      }
      
      return order === 'desc' ? comparison * -1 : comparison;
    });
  }
}

@Pipe({
  name: 'groupBy',
  standalone: true
})
export class GroupByPipe implements PipeTransform {
  transform(items: any[], property: string): any {
    if (!items || !property) return items;
    
    return items.reduce((groups, item) => {
      const key = item[property];
      if (!groups[key]) {
        groups[key] = [];
      }
      groups[key].push(item);
      return groups;
    }, {});
  }
}

// Usage
@Component({
  selector: 'app-array-pipes',
  standalone: true,
  imports: [FilterPipe, SortPipe, GroupByPipe, KeyValuePipe],
  template: `
    <h2>Filter</h2>
    <input [(ngModel)]="searchTerm">
    <div *ngFor="let user of users | filter:searchTerm:'name'">
      {{ user.name }} - {{ user.age }}
    </div>
    
    <h2>Sort</h2>
    <button (click)="sortOrder = 'asc'">Ascending</button>
    <button (click)="sortOrder = 'desc'">Descending</button>
    <div *ngFor="let user of users | sort:'age':sortOrder">
      {{ user.name }} - {{ user.age }}
    </div>
    
    <h2>Group By</h2>
    <div *ngFor="let group of users | groupBy:'department' | keyvalue">
      <h3>{{ group.key }}</h3>
      <div *ngFor="let user of group.value">
        {{ user.name }}
      </div>
    </div>
  `
})
export class ArrayPipesComponent {
  searchTerm = '';
  sortOrder: 'asc' | 'desc' = 'asc';
  
  users = [
    { name: 'John', age: 30, department: 'IT' },
    { name: 'Jane', age: 25, department: 'HR' },
    { name: 'Bob', age: 35, department: 'IT' },
    { name: 'Alice', age: 28, department: 'HR' }
  ];
}
```

### Date and Time Pipes

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'timeAgo',
  standalone: true
})
export class TimeAgoPipe implements PipeTransform {
  transform(value: Date | string): string {
    const date = new Date(value);
    const now = new Date();
    const seconds = Math.floor((now.getTime() - date.getTime()) / 1000);
    
    const intervals = {
      year: 31536000,
      month: 2592000,
      week: 604800,
      day: 86400,
      hour: 3600,
      minute: 60,
      second: 1
    };
    
    for (const [unit, secondsInUnit] of Object.entries(intervals)) {
      const interval = Math.floor(seconds / secondsInUnit);
      if (interval >= 1) {
        return `${interval} ${unit}${interval > 1 ? 's' : ''} ago`;
      }
    }
    
    return 'just now';
  }
}

@Pipe({
  name: 'duration',
  standalone: true
})
export class DurationPipe implements PipeTransform {
  transform(seconds: number): string {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    
    const parts = [];
    if (hours > 0) parts.push(`${hours}h`);
    if (minutes > 0) parts.push(`${minutes}m`);
    if (secs > 0 || parts.length === 0) parts.push(`${secs}s`);
    
    return parts.join(' ');
  }
}

// Usage
@Component({
  selector: 'app-time-pipes',
  standalone: true,
  imports: [TimeAgoPipe, DurationPipe],
  template: `
    <p>{{ pastDate | timeAgo }}</p>
    <p>{{ 3665 | duration }}</p>
  `
})
export class TimePipesComponent {
  pastDate = new Date(Date.now() - 5 * 60 * 1000); // 5 minutes ago
}
```

## Async Pipe Deep Dive

The async pipe subscribes to Observables or Promises and returns the latest emitted value. It automatically unsubscribes when the component is destroyed.

### Basic Async Pipe

```typescript
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { Observable, of, interval } from 'rxjs';
import { map, take } from 'rxjs/operators';

@Component({
  selector: 'app-async-basic',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <h2>Observable with async pipe</h2>
    <p>{{ message$ | async }}</p>
    
    <h2>Counter</h2>
    <p>{{ counter$ | async }}</p>
    
    <h2>Promise</h2>
    <p>{{ promise | async }}</p>
  `
})
export class AsyncBasicComponent {
  message$: Observable<string> = of('Hello from Observable!');
  
  counter$ = interval(1000).pipe(
    take(10),
    map(n => n + 1)
  );
  
  promise = new Promise<string>(resolve => {
    setTimeout(() => resolve('Hello from Promise!'), 2000);
  });
}
```

### Async Pipe with ng-container

```typescript
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { Observable } from 'rxjs';
import { HttpClient } from '@angular/common/http';

interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-async-container',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <!-- Subscribe once, use multiple times -->
    <ng-container *ngIf="user$ | async as user">
      <h2>{{ user.name }}</h2>
      <p>Email: {{ user.email }}</p>
      <p>ID: {{ user.id }}</p>
    </ng-container>
    
    <!-- Without ng-container (subscribes 3 times!) -->
    <h2>{{ (user$ | async)?.name }}</h2>
    <p>Email: {{ (user$ | async)?.email }}</p>
    <p>ID: {{ (user$ | async)?.id }}</p>
  `
})
export class AsyncContainerComponent {
  user$: Observable<User>;
  
  constructor(private http: HttpClient) {
    this.user$ = this.http.get<User>('api/user/1');
  }
}
```

### Async Pipe with Loading and Error States

```typescript
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { Observable, of, throwError } from 'rxjs';
import { delay, catchError, map } from 'rxjs/operators';

interface AsyncState<T> {
  loading: boolean;
  data?: T;
  error?: string;
}

@Component({
  selector: 'app-async-states',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <ng-container *ngIf="state$ | async as state">
      <div *ngIf="state.loading">Loading...</div>
      <div *ngIf="state.error">Error: {{ state.error }}</div>
      <div *ngIf="state.data">
        <h2>{{ state.data.title }}</h2>
        <p>{{ state.data.content }}</p>
      </div>
    </ng-container>
  `
})
export class AsyncStatesComponent {
  state$: Observable<AsyncState<any>>;
  
  constructor() {
    this.state$ = this.loadData();
  }
  
  private loadData(): Observable<AsyncState<any>> {
    return of({ loading: true }).pipe(
      delay(1000),
      map(() => ({
        loading: false,
        data: { title: 'Success', content: 'Data loaded!' }
      })),
      catchError(error => of({
        loading: false,
        error: error.message
      }))
    );
  }
}
```

### Combining Multiple Async Pipes

```typescript
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { Observable, combineLatest, of } from 'rxjs';
import { map } from 'rxjs/operators';

@Component({
  selector: 'app-combine-async',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <!-- Method 1: Multiple async pipes -->
    <div>
      <p>User: {{ (user$ | async)?.name }}</p>
      <p>Settings: {{ (settings$ | async)?.theme }}</p>
    </div>
    
    <!-- Method 2: Combine observables -->
    <ng-container *ngIf="viewModel$ | async as vm">
      <p>User: {{ vm.user.name }}</p>
      <p>Settings: {{ vm.settings.theme }}</p>
    </ng-container>
  `
})
export class CombineAsyncComponent {
  user$ = of({ id: 1, name: 'John' });
  settings$ = of({ theme: 'dark', language: 'en' });
  
  viewModel$ = combineLatest([this.user$, this.settings$]).pipe(
    map(([user, settings]) => ({ user, settings }))
  );
}
```

## Transform Chains

Multiple pipes can be chained together, executing from left to right.

### Basic Chaining

```typescript
import { Component } from '@angular/core';
import { DatePipe, UpperCasePipe, SlicePipe } from '@angular/common';

@Component({
  selector: 'app-chain-basic',
  standalone: true,
  imports: [DatePipe, UpperCasePipe, SlicePipe],
  template: `
    <!-- Chain multiple transformations -->
    <p>{{ text | slice:0:5 | uppercase }}</p>
    <!-- Output: HELLO -->
    
    <p>{{ today | date:'fullDate' | uppercase }}</p>
    <!-- Output: FRIDAY, JANUARY 1, 2026 -->
    
    <p>{{ items | slice:1:4 | json }}</p>
    <!-- Output: [2, 3, 4] -->
  `
})
export class ChainBasicComponent {
  text = 'hello world';
  today = new Date();
  items = [1, 2, 3, 4, 5];
}
```

### Complex Chaining

```typescript
import { Component } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { Pipe, PipeTransform } from '@angular/core';
import { Observable, of } from 'rxjs';
import { map } from 'rxjs/operators';

@Pipe({ name: 'multiply', standalone: true })
export class MultiplyPipe implements PipeTransform {
  transform(value: number, factor: number): number {
    return value * factor;
  }
}

@Pipe({ name: 'round', standalone: true })
export class RoundPipe implements PipeTransform {
  transform(value: number, decimals = 0): number {
    return Number(value.toFixed(decimals));
  }
}

@Pipe({ name: 'prefix', standalone: true })
export class PrefixPipe implements PipeTransform {
  transform(value: any, prefix: string): string {
    return `${prefix}${value}`;
  }
}

@Component({
  selector: 'app-chain-complex',
  standalone: true,
  imports: [AsyncPipe, MultiplyPipe, RoundPipe, PrefixPipe],
  template: `
    <!-- Complex chain -->
    <p>{{ price | multiply:1.2 | round:2 | prefix:'$' }}</p>
    <!-- Output: $119.99 -->
    
    <!-- With async pipe -->
    <p>{{ price$ | async | multiply:1.2 | round:2 | prefix:'€' }}</p>
    <!-- Output: €119.99 -->
  `
})
export class ChainComplexComponent {
  price = 99.99;
  price$ = of(99.99);
}
```

## Caching Strategies

### Memoized Pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'memoized',
  standalone: true,
  pure: true
})
export class MemoizedPipe implements PipeTransform {
  private cache = new Map<string, any>();
  
  transform(value: any, ...args: any[]): any {
    const key = JSON.stringify([value, ...args]);
    
    if (this.cache.has(key)) {
      console.log('Returning cached result');
      return this.cache.get(key);
    }
    
    console.log('Computing result');
    const result = this.expensiveOperation(value, ...args);
    this.cache.set(key, result);
    
    return result;
  }
  
  private expensiveOperation(value: any, ...args: any[]): any {
    // Simulate expensive operation
    return value;
  }
}
```

### Cache with Size Limit

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'cachedFilter',
  standalone: true,
  pure: true
})
export class CachedFilterPipe implements PipeTransform {
  private cache = new Map<string, any[]>();
  private readonly maxCacheSize = 10;
  
  transform(items: any[], searchTerm: string): any[] {
    if (!items || !searchTerm) return items;
    
    const key = `${JSON.stringify(items)}_${searchTerm}`;
    
    if (this.cache.has(key)) {
      return this.cache.get(key)!;
    }
    
    const result = items.filter(item => 
      JSON.stringify(item).toLowerCase().includes(searchTerm.toLowerCase())
    );
    
    // LRU cache implementation
    if (this.cache.size >= this.maxCacheSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, result);
    return result;
  }
}
```

### Time-based Cache

```typescript
import { Pipe, PipeTransform } from '@angular/core';

interface CacheEntry {
  value: any;
  timestamp: number;
}

@Pipe({
  name: 'timedCache',
  standalone: true,
  pure: false // Needs to check time
})
export class TimedCachePipe implements PipeTransform {
  private cache = new Map<string, CacheEntry>();
  private readonly cacheDuration = 5000; // 5 seconds
  
  transform(value: any, ...args: any[]): any {
    const key = JSON.stringify([value, ...args]);
    const now = Date.now();
    
    const cached = this.cache.get(key);
    if (cached && (now - cached.timestamp) < this.cacheDuration) {
      console.log('Returning cached value');
      return cached.value;
    }
    
    console.log('Computing new value');
    const result = this.compute(value, ...args);
    
    this.cache.set(key, {
      value: result,
      timestamp: now
    });
    
    return result;
  }
  
  private compute(value: any, ...args: any[]): any {
    // Expensive computation
    return value;
  }
}
```

## Stateful Pipes

Stateful pipes maintain internal state and can produce different outputs for the same input based on their state.

### Counter Pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'counter',
  standalone: true,
  pure: false
})
export class CounterPipe implements PipeTransform {
  private count = 0;
  
  transform(value: any): string {
    this.count++;
    return `${value} (called ${this.count} times)`;
  }
}
```

### Stateful Filter Pipe

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'statefulFilter',
  standalone: true,
  pure: false
})
export class StatefulFilterPipe implements PipeTransform {
  private lastItems: any[] = [];
  private lastSearchTerm = '';
  private cachedResult: any[] = [];
  
  transform(items: any[], searchTerm: string): any[] {
    // Only recompute if inputs actually changed
    if (items === this.lastItems && searchTerm === this.lastSearchTerm) {
      return this.cachedResult;
    }
    
    this.lastItems = items;
    this.lastSearchTerm = searchTerm;
    
    if (!searchTerm) {
      this.cachedResult = items;
      return items;
    }
    
    this.cachedResult = items.filter(item =>
      JSON.stringify(item).toLowerCase().includes(searchTerm.toLowerCase())
    );
    
    return this.cachedResult;
  }
}
```

## Common Mistakes

### 1. Using Impure Pipes Unnecessarily

```typescript
// BAD: Impure pipe for simple transformation
@Pipe({ name: 'bad', pure: false })
export class BadPipe implements PipeTransform {
  transform(value: string): string {
    return value.toUpperCase();
  }
}

// GOOD: Pure pipe is sufficient
@Pipe({ name: 'good', pure: true })
export class GoodPipe implements PipeTransform {
  transform(value: string): string {
    return value.toUpperCase();
  }
}
```

### 2. Multiple Async Pipe Subscriptions

```typescript
// BAD: Subscribes 3 times
@Component({
  template: `
    <p>{{ user$ | async }}</p>
    <p>{{ user$ | async }}</p>
    <p>{{ user$ | async }}</p>
  `
})
export class BadComponent {}

// GOOD: Subscribe once
@Component({
  template: `
    <ng-container *ngIf="user$ | async as user">
      <p>{{ user }}</p>
      <p>{{ user }}</p>
      <p>{{ user }}</p>
    </ng-container>
  `
})
export class GoodComponent {}
```

### 3. Heavy Computation in Pipes

```typescript
// BAD: Expensive operation every time
@Pipe({ name: 'bad', pure: true })
export class BadPipe implements PipeTransform {
  transform(items: any[]): any[] {
    return items.filter(item => {
      // Expensive operation
      return this.expensiveCheck(item);
    });
  }
}

// GOOD: Cache results
@Pipe({ name: 'good', pure: true })
export class GoodPipe implements PipeTransform {
  private cache = new Map();
  
  transform(items: any[]): any[] {
    const key = JSON.stringify(items);
    if (this.cache.has(key)) {
      return this.cache.get(key);
    }
    
    const result = items.filter(item => this.expensiveCheck(item));
    this.cache.set(key, result);
    return result;
  }
}
```

## Best Practices

### 1. Keep Pipes Pure When Possible

```typescript
@Pipe({
  name: 'format',
  standalone: true,
  pure: true // Default, but explicit is better
})
export class FormatPipe implements PipeTransform {
  transform(value: string): string {
    return value.toUpperCase();
  }
}
```

### 2. Use Async Pipe for Observables

```typescript
// GOOD: Let Angular handle subscription
@Component({
  template: `<p>{{ data$ | async }}</p>`
})
export class GoodComponent {
  data$ = this.service.getData();
}

// AVOID: Manual subscription
@Component({
  template: `<p>{{ data }}</p>`
})
export class AvoidComponent implements OnInit, OnDestroy {
  data: any;
  subscription?: Subscription;
  
  ngOnInit() {
    this.subscription = this.service.getData().subscribe(
      data => this.data = data
    );
  }
  
  ngOnDestroy() {
    this.subscription?.unsubscribe();
  }
}
```

### 3. Document Pipe Parameters

```typescript
/**
 * Truncates a string to a specified length
 * @param value - The string to truncate
 * @param limit - Maximum length (default: 50)
 * @param trail - Trailing string (default: '...')
 * @example
 * {{ text | truncate:30:'...' }}
 */
@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 50, trail = '...'): string {
    // Implementation
  }
}
```

### 4. Use TypeScript for Type Safety

```typescript
@Pipe({ name: 'typed', standalone: true })
export class TypedPipe implements PipeTransform {
  transform(value: string, limit: number, suffix: string): string {
    // TypeScript ensures correct types
    return value.substring(0, limit) + suffix;
  }
}
```

## Interview Questions

### Q1: What's the difference between pure and impure pipes?

**Answer:** Pure pipes (default) only execute when input reference changes. Impure pipes execute on every change detection cycle. Pure pipes are more performant but don't detect mutations within objects or arrays.

### Q2: How does the async pipe prevent memory leaks?

**Answer:** The async pipe automatically subscribes to the Observable/Promise when the component is created and unsubscribes when the component is destroyed, preventing memory leaks from forgotten subscriptions.

### Q3: When should you use an impure pipe?

**Answer:** Use impure pipes sparingly, only when you need to detect mutations within objects or arrays. Consider using immutable data patterns with pure pipes instead for better performance.

### Q4: Can pipes have side effects?

**Answer:** Pipes should be pure functions without side effects. They should only transform data, not modify the input or trigger other operations. Side effects make pipes unpredictable and harder to test.

### Q5: How do you optimize pipe performance?

**Answer:** Use pure pipes when possible, implement caching for expensive operations, avoid multiple async pipe subscriptions to the same Observable, and consider memoization for complex transformations.

### Q6: What happens when you chain pipes?

**Answer:** Pipes execute left to right. Each pipe receives the output of the previous pipe as its input. The final transformed value is displayed in the template.

### Q7: Can you use pipes in component code?

**Answer:** Yes, inject the pipe in the constructor and call its transform method. However, this is generally discouraged - use pipes in templates and move complex logic to services.

### Q8: How do custom pipes handle null or undefined values?

**Answer:** Pipes should handle null/undefined inputs gracefully, either returning a default value or the input itself. Always add null checks to prevent errors.

## Key Takeaways

1. **Pipes transform template data** without modifying the original values
2. **Pure pipes are the default** and most performant option
3. **Impure pipes execute every change detection** cycle - use sparingly
4. **Async pipe automatically manages subscriptions** preventing memory leaks
5. **Chain pipes for complex transformations** - they execute left to right
6. **Cache expensive computations** in pipes for better performance
7. **Use ng-container with async** to avoid multiple subscriptions
8. **Pipes should be pure functions** without side effects
9. **Document pipe parameters** with JSDoc for better maintainability
10. **Type your pipes** with TypeScript for type safety

## Resources

### Official Documentation
- [Angular Pipes](https://angular.dev/guide/pipes)
- [Async Pipe API](https://angular.dev/api/common/AsyncPipe)
- [Creating Custom Pipes](https://angular.dev/guide/pipes/transform-data)

### Articles
- "Understanding Angular Pipes" - Angular University
- "Pure vs Impure Pipes Deep Dive" - Thoughtram
- "Async Pipe Best Practices" - Netanel Basal

### Video Tutorials
- "Mastering Angular Pipes" - ng-conf
- "Custom Pipes Patterns" - Angular Connect
- "Performance with Pipes" - Angular Denver

### Tools
- Angular DevTools - Profile pipe executions
- RxJS Marbles - Visualize async pipe behavior
- VS Code Angular Language Service - Pipe intellisense
