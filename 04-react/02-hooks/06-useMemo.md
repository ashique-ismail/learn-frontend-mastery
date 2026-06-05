# useMemo Hook - Optimizing Expensive Computations

## The Idea

**In plain English:** `useMemo` is a way to tell React "remember the answer to this calculation, and only redo it if the inputs change." Instead of running the same heavy math or search every time the screen updates, React keeps the saved result and reuses it when nothing relevant has changed.

**Real-world analogy:** Imagine a librarian who looks up a list of all books by a specific author. The first time you ask, she searches through thousands of cards — that takes a while. But she writes the result on a sticky note. The next time someone asks the same question, she just reads the sticky note instead of searching again. She only throws out the sticky note and searches fresh when the library gets new books added.

- The sticky note = the cached (memoized) value React stores
- Searching through the card catalog = the expensive computation that runs inside `useMemo`
- New books being added = a dependency changing, which tells React to recalculate

---

## Table of Contents

- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Core API](#core-api)
- [When to Use useMemo](#when-to-use-usememo)
- [Common Patterns](#common-patterns)
- [Performance Analysis](#performance-analysis)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

`useMemo` is a React Hook that memoizes the result of an expensive computation, recalculating only when dependencies change. It's used to optimize performance by avoiding unnecessary calculations on every render.

### The Problem useMemo Solves

```javascript
// ❌ Problem: Expensive computation on every render
function ProductList({ products, filters }) {
  // This runs on EVERY render, even if products/filters haven't changed!
  const filteredProducts = products.filter(product => {
    return Object.keys(filters).every(key => 
      product[key] === filters[key]
    );
  }).sort((a, b) => b.rating - a.rating);
  
  return (
    <div>
      {filteredProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// ✅ Solution: Memoize expensive computation
function ProductList({ products, filters }) {
  // Only recalculates when products or filters change
  const filteredProducts = useMemo(() => {
    return products
      .filter(product => 
        Object.keys(filters).every(key => 
          product[key] === filters[key]
        )
      )
      .sort((a, b) => b.rating - a.rating);
  }, [products, filters]);
  
  return (
    <div>
      {filteredProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

### Visual Representation

```
Without useMemo:                  With useMemo:
┌──────────────┐                 ┌──────────────┐
│Component     │                 │Component     │
│Renders       │                 │Renders       │
└──────┬───────┘                 └──────┬───────┘
       │                                │
       │                                ├─→ Check deps
       ↓                                │   Changed?
┌──────────────┐                       │   
│Run expensive │                       ├─→ No: Return
│computation   │                       │   cached value
└──────────────┘                       │
                                       └─→ Yes: Recompute
                                           and cache
```

## Basic Concepts

### Syntax

```javascript
const memoizedValue = useMemo(
  computeFunction,
  dependencies
);
```

**Parameters:**
- `computeFunction`: Function that returns the value to memoize
- `dependencies`: Array of values that trigger recalculation

**Returns:**
- Cached result of the computation

### How It Works

```javascript
function Component({ data }) {
  // Computation runs only when 'data' changes
  const processedData = useMemo(() => {
    console.log('Computing...');
    return expensiveProcessing(data);
  }, [data]);
  
  // On subsequent renders with same 'data':
  // - Function doesn't run
  // - Cached value is returned
  // - Console log doesn't appear
  
  return <div>{processedData}</div>;
}
```

## Core API

### Basic Usage

```javascript
import { useState, useMemo } from 'react';

function DataAnalyzer({ data }) {
  const [sortBy, setSortBy] = useState('name');

  // Expensive sorting operation
  const sortedData = useMemo(() => {
    console.log('Sorting data...');
    return [...data].sort((a, b) => {
      if (a[sortBy] < b[sortBy]) return -1;
      if (a[sortBy] > b[sortBy]) return 1;
      return 0;
    });
  }, [data, sortBy]);

  const statistics = useMemo(() => {
    console.log('Calculating statistics...');
    return {
      total: sortedData.length,
      average: sortedData.reduce((sum, item) => 
        sum + item.value, 0) / sortedData.length,
      max: Math.max(...sortedData.map(item => item.value)),
      min: Math.min(...sortedData.map(item => item.value))
    };
  }, [sortedData]);

  return (
    <div>
      <select value={sortBy} onChange={e => setSortBy(e.target.value)}>
        <option value="name">Name</option>
        <option value="value">Value</option>
      </select>
      
      <div>
        <p>Total: {statistics.total}</p>
        <p>Average: {statistics.average}</p>
        <p>Max: {statistics.max}</p>
        <p>Min: {statistics.min}</p>
      </div>
      
      <ul>
        {sortedData.map(item => (
          <li key={item.id}>{item.name}: {item.value}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Filtering and Sorting

```javascript
function ProductCatalog({ products }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [category, setCategory] = useState('all');
  const [sortBy, setSortBy] = useState('name');

  // Filter by search and category
  const filteredProducts = useMemo(() => {
    return products.filter(product => {
      const matchesSearch = product.name
        .toLowerCase()
        .includes(searchTerm.toLowerCase());
      const matchesCategory = 
        category === 'all' || product.category === category;
      return matchesSearch && matchesCategory;
    });
  }, [products, searchTerm, category]);

  // Sort filtered results
  const sortedProducts = useMemo(() => {
    return [...filteredProducts].sort((a, b) => {
      switch (sortBy) {
        case 'name':
          return a.name.localeCompare(b.name);
        case 'price':
          return a.price - b.price;
        case 'rating':
          return b.rating - a.rating;
        default:
          return 0;
      }
    });
  }, [filteredProducts, sortBy]);

  return (
    <div>
      <input
        type="search"
        value={searchTerm}
        onChange={e => setSearchTerm(e.target.value)}
        placeholder="Search products..."
      />
      
      <select value={category} onChange={e => setCategory(e.target.value)}>
        <option value="all">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>
      
      <select value={sortBy} onChange={e => setSortBy(e.target.value)}>
        <option value="name">Sort by Name</option>
        <option value="price">Sort by Price</option>
        <option value="rating">Sort by Rating</option>
      </select>
      
      <div className="results-count">
        Showing {sortedProducts.length} of {products.length} products
      </div>
      
      <div className="product-grid">
        {sortedProducts.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

### Complex Computations

```javascript
function ChartComponent({ dataPoints }) {
  // Calculate chart dimensions and scales
  const chartConfig = useMemo(() => {
    const values = dataPoints.map(d => d.value);
    const dates = dataPoints.map(d => new Date(d.date));
    
    return {
      width: 800,
      height: 400,
      padding: 40,
      minValue: Math.min(...values),
      maxValue: Math.max(...values),
      minDate: Math.min(...dates),
      maxDate: Math.max(...dates),
      xScale: (date) => {
        const range = dates[dates.length - 1] - dates[0];
        return ((date - dates[0]) / range) * 720 + 40;
      },
      yScale: (value) => {
        const range = Math.max(...values) - Math.min(...values);
        return 360 - ((value - Math.min(...values)) / range) * 320 + 40;
      }
    };
  }, [dataPoints]);

  // Generate SVG path
  const pathData = useMemo(() => {
    return dataPoints
      .map((point, index) => {
        const x = chartConfig.xScale(new Date(point.date));
        const y = chartConfig.yScale(point.value);
        return `${index === 0 ? 'M' : 'L'} ${x} ${y}`;
      })
      .join(' ');
  }, [dataPoints, chartConfig]);

  return (
    <svg width={chartConfig.width} height={chartConfig.height}>
      <path d={pathData} stroke="blue" fill="none" strokeWidth={2} />
      {/* More chart elements */}
    </svg>
  );
}
```

## When to Use useMemo

### Decision Matrix

| Scenario | Use useMemo | Don't Use |
|----------|-------------|-----------|
| Expensive computations | ✅ | ❌ |
| Large array operations | ✅ | ❌ |
| Complex object creation | ✅ | ❌ |
| Simple calculations | ❌ | ✅ |
| Primitive values | ❌ | ✅ |
| Prevent child re-renders | ✅ | ❌ |

### Good Use Cases

```javascript
// ✅ GOOD: Expensive filtering/sorting
const sortedUsers = useMemo(() => {
  return users
    .filter(user => user.active)
    .sort((a, b) => a.name.localeCompare(b.name));
}, [users]);

// ✅ GOOD: Complex calculations
const statistics = useMemo(() => {
  return calculateStatistics(data); // Expensive operation
}, [data]);

// ✅ GOOD: Creating objects/arrays for props
const config = useMemo(() => ({
  options: { theme: 'dark', locale: 'en' },
  callbacks: { onSave, onCancel }
}), [onSave, onCancel]);

// ✅ GOOD: Derived state
const fullName = useMemo(() => {
  return `${firstName} ${lastName}`.trim();
}, [firstName, lastName]);
```

### Bad Use Cases

```javascript
// ❌ BAD: Simple arithmetic
const total = useMemo(() => price + tax, [price, tax]);
// Better: const total = price + tax;

// ❌ BAD: String concatenation
const greeting = useMemo(() => `Hello, ${name}`, [name]);
// Better: const greeting = `Hello, ${name}`;

// ❌ BAD: Single array access
const firstItem = useMemo(() => items[0], [items]);
// Better: const firstItem = items[0];

// ❌ BAD: Cheap computations
const doubled = useMemo(() => count * 2, [count]);
// Better: const doubled = count * 2;
```

## Common Patterns

### Referential Equality

```javascript
// Problem: New object on every render causes child to re-render
function Parent() {
  const [count, setCount] = useState(0);
  
  // ❌ New object every render
  const theme = {
    primary: 'blue',
    secondary: 'green'
  };
  
  return <Child theme={theme} />;
  // Child re-renders even if count changes!
}

// Solution: Memoize object
function Parent() {
  const [count, setCount] = useState(0);
  
  // ✅ Same object reference when possible
  const theme = useMemo(() => ({
    primary: 'blue',
    secondary: 'green'
  }), []);
  
  return <Child theme={theme} />;
}

const Child = React.memo(({ theme }) => {
  console.log('Child rendered');
  return <div style={{ color: theme.primary }}>Content</div>;
});
```

### Chaining Memoized Values

```javascript
function DataDashboard({ rawData }) {
  // Step 1: Clean data
  const cleanedData = useMemo(() => {
    return rawData.filter(item => item.valid);
  }, [rawData]);

  // Step 2: Transform data (depends on cleaned)
  const transformedData = useMemo(() => {
    return cleanedData.map(item => ({
      ...item,
      computed: item.value * 2
    }));
  }, [cleanedData]);

  // Step 3: Aggregate (depends on transformed)
  const aggregatedData = useMemo(() => {
    return transformedData.reduce((acc, item) => {
      const key = item.category;
      acc[key] = (acc[key] || 0) + item.computed;
      return acc;
    }, {});
  }, [transformedData]);

  return (
    <div>
      <div>Total Items: {cleanedData.length}</div>
      <div>Transformed: {transformedData.length}</div>
      <div>Categories: {Object.keys(aggregatedData).length}</div>
    </div>
  );
}
```

### With Custom Hooks

```javascript
function useFilteredAndSorted(items, filters, sortConfig) {
  const filteredItems = useMemo(() => {
    return items.filter(item => {
      return Object.entries(filters).every(([key, value]) => {
        if (!value) return true;
        return item[key]?.toString().toLowerCase()
          .includes(value.toLowerCase());
      });
    });
  }, [items, filters]);

  const sortedItems = useMemo(() => {
    if (!sortConfig.key) return filteredItems;
    
    return [...filteredItems].sort((a, b) => {
      const aVal = a[sortConfig.key];
      const bVal = b[sortConfig.key];
      
      if (aVal < bVal) return sortConfig.direction === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortConfig.direction === 'asc' ? 1 : -1;
      return 0;
    });
  }, [filteredItems, sortConfig]);

  return sortedItems;
}

// Usage
function DataTable({ data }) {
  const [filters, setFilters] = useState({});
  const [sortConfig, setSortConfig] = useState({ 
    key: null, 
    direction: 'asc' 
  });

  const processedData = useFilteredAndSorted(data, filters, sortConfig);

  return <Table data={processedData} />;
}
```

### Memoizing JSX

```javascript
function Sidebar({ items, currentPath }) {
  // Memoize expensive JSX generation
  const navigation = useMemo(() => {
    return items.map(item => {
      const isActive = currentPath.startsWith(item.path);
      const Icon = item.icon;
      
      return (
        <li key={item.id} className={isActive ? 'active' : ''}>
          <Link to={item.path}>
            <Icon />
            <span>{item.label}</span>
          </Link>
          {item.children && (
            <ul>
              {item.children.map(child => (
                <li key={child.id}>
                  <Link to={child.path}>{child.label}</Link>
                </li>
              ))}
            </ul>
          )}
        </li>
      );
    });
  }, [items, currentPath]);

  return <nav><ul>{navigation}</ul></nav>;
}
```

## Performance Analysis

### Measuring Cost

```javascript
function ExpensiveComponent({ data }) {
  // Measure without useMemo
  console.time('computation');
  const result = expensiveComputation(data);
  console.timeEnd('computation');
  
  // Measure with useMemo
  const memoizedResult = useMemo(() => {
    console.time('memoized computation');
    const result = expensiveComputation(data);
    console.timeEnd('memoized computation');
    return result;
  }, [data]);
  
  return <div>{result}</div>;
}
```

### Profiling

```javascript
import { Profiler } from 'react';

function App() {
  const onRender = (id, phase, actualDuration, baseDuration) => {
    console.log({
      id,
      phase,
      actualDuration,
      baseDuration
    });
  };

  return (
    <Profiler id="ExpensiveComponent" onRender={onRender}>
      <ExpensiveComponent />
    </Profiler>
  );
}
```

### Benchmark Comparison

```javascript
// Without useMemo: ~50ms per render
function SlowComponent({ data }) {
  const processed = data
    .filter(x => x.active)
    .map(x => ({ ...x, computed: heavyComputation(x) }))
    .sort((a, b) => b.score - a.score);
  
  return <List items={processed} />;
}

// With useMemo: ~50ms first render, ~0.1ms subsequent renders
function FastComponent({ data }) {
  const processed = useMemo(() => 
    data
      .filter(x => x.active)
      .map(x => ({ ...x, computed: heavyComputation(x) }))
      .sort((a, b) => b.score - a.score),
    [data]
  );
  
  return <List items={processed} />;
}
```

## Common Mistakes

### 1. Missing Dependencies

```javascript
// ❌ WRONG: Missing 'multiplier' dependency
function Component({ data, multiplier }) {
  const computed = useMemo(() => {
    return data.map(x => x * multiplier); // Uses multiplier
  }, [data]); // But doesn't include it!
  
  // Result: Stale calculations when multiplier changes
}

// ✅ CORRECT: Include all dependencies
function Component({ data, multiplier }) {
  const computed = useMemo(() => {
    return data.map(x => x * multiplier);
  }, [data, multiplier]);
}
```

### 2. Overusing useMemo

```javascript
// ❌ BAD: Premature optimization
function Component() {
  const a = useMemo(() => 1 + 1, []);
  const b = useMemo(() => 'Hello' + 'World', []);
  const c = useMemo(() => props.value * 2, [props.value]);
  
  // Overhead of useMemo > cost of simple operations
}

// ✅ GOOD: Only for expensive operations
function Component() {
  const a = 1 + 1;
  const b = 'HelloWorld';
  const c = props.value * 2;
  
  // Only memoize when actually expensive
  const expensive = useMemo(() => {
    return data.map(item => {
      return heavyProcessing(item); // Actually expensive
    });
  }, [data]);
}
```

### 3. Unnecessary Object Creation

```javascript
// ❌ WRONG: Creating objects in dependencies
function Component({ user }) {
  const greeting = useMemo(() => {
    return `Hello, ${user.name}`;
  }, [{ name: user.name }]); // New object every render!
  
  // useMemo always recalculates
}

// ✅ CORRECT: Use primitive values
function Component({ user }) {
  const greeting = useMemo(() => {
    return `Hello, ${user.name}`;
  }, [user.name]); // Primitive value
}
```

### 4. Not Memoizing Related Values

```javascript
// ❌ BAD: Child still re-renders
function Parent() {
  const [count, setCount] = useState(0);
  
  const expensiveValue = useMemo(() => {
    return computeExpensive(count);
  }, [count]);
  
  // But config is new object every render!
  const config = { value: expensiveValue };
  
  return <MemoizedChild config={config} />;
}

// ✅ GOOD: Memoize all props
function Parent() {
  const [count, setCount] = useState(0);
  
  const expensiveValue = useMemo(() => {
    return computeExpensive(count);
  }, [count]);
  
  const config = useMemo(() => ({
    value: expensiveValue
  }), [expensiveValue]);
  
  return <MemoizedChild config={config} />;
}
```

## Best Practices

### 1. Profile Before Optimizing

```javascript
// Measure actual impact before adding useMemo
function Component({ data }) {
  // Add console.time to measure
  console.time('filterData');
  const filtered = data.filter(item => item.active);
  console.timeEnd('filterData');
  
  // If < 1ms, probably not worth memoizing
  // If > 50ms, definitely worth memoizing
}
```

### 2. Use ESLint Rule

```javascript
// .eslintrc.js
{
  "rules": {
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

### 3. Document Expensive Operations

```javascript
function Component({ data }) {
  // useMemo: Filters 10,000+ items and performs heavy computation
  // Measured at ~200ms without memoization
  const processed = useMemo(() => {
    return data
      .filter(item => item.valid)
      .map(item => heavyProcessing(item));
  }, [data]);
}
```

### 4. Consider useMemo for Referential Equality

```javascript
// Even if computation isn't expensive,
// useMemo prevents child re-renders
function Parent() {
  const [count, setCount] = useState(0);
  
  // Not expensive, but prevents MemoizedChild re-render
  const options = useMemo(() => ({
    theme: 'dark',
    locale: 'en'
  }), []);
  
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>
        Count: {count}
      </button>
      <MemoizedChild options={options} />
    </div>
  );
}
```

## Real-World Examples

### Data Grid with Filtering and Sorting

```javascript
function DataGrid({ data, columns }) {
  const [filters, setFilters] = useState({});
  const [sortBy, setSortBy] = useState(null);
  const [sortDir, setSortDir] = useState('asc');

  // Apply filters
  const filteredData = useMemo(() => {
    return data.filter(row => {
      return Object.entries(filters).every(([key, filterValue]) => {
        if (!filterValue) return true;
        const cellValue = row[key]?.toString().toLowerCase();
        return cellValue?.includes(filterValue.toLowerCase());
      });
    });
  }, [data, filters]);

  // Apply sorting
  const sortedData = useMemo(() => {
    if (!sortBy) return filteredData;
    
    return [...filteredData].sort((a, b) => {
      const aVal = a[sortBy];
      const bVal = b[sortBy];
      
      let comparison = 0;
      if (aVal < bVal) comparison = -1;
      if (aVal > bVal) comparison = 1;
      
      return sortDir === 'asc' ? comparison : -comparison;
    });
  }, [filteredData, sortBy, sortDir]);

  // Calculate page data
  const [page, setPage] = useState(1);
  const pageSize = 50;
  
  const paginatedData = useMemo(() => {
    const start = (page - 1) * pageSize;
    return sortedData.slice(start, start + pageSize);
  }, [sortedData, page]);

  const totalPages = Math.ceil(sortedData.length / pageSize);

  return (
    <div>
      <div className="filters">
        {columns.map(col => (
          <input
            key={col.key}
            placeholder={`Filter ${col.label}`}
            onChange={e => setFilters(prev => ({
              ...prev,
              [col.key]: e.target.value
            }))}
          />
        ))}
      </div>

      <table>
        <thead>
          <tr>
            {columns.map(col => (
              <th
                key={col.key}
                onClick={() => {
                  if (sortBy === col.key) {
                    setSortDir(dir => dir === 'asc' ? 'desc' : 'asc');
                  } else {
                    setSortBy(col.key);
                    setSortDir('asc');
                  }
                }}
              >
                {col.label}
                {sortBy === col.key && (sortDir === 'asc' ? ' ↑' : ' ↓')}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {paginatedData.map((row, idx) => (
            <tr key={idx}>
              {columns.map(col => (
                <td key={col.key}>{row[col.key]}</td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      <div className="pagination">
        <button 
          disabled={page === 1}
          onClick={() => setPage(p => p - 1)}
        >
          Previous
        </button>
        <span>Page {page} of {totalPages}</span>
        <button 
          disabled={page === totalPages}
          onClick={() => setPage(p => p + 1)}
        >
          Next
        </button>
      </div>

      <div>
        Showing {paginatedData.length} of {sortedData.length} items
        {Object.keys(filters).some(k => filters[k]) && 
          ` (filtered from ${data.length} total)`
        }
      </div>
    </div>
  );
}
```

### Search with Highlighting

```javascript
function SearchableList({ items }) {
  const [searchTerm, setSearchTerm] = useState('');

  const searchResults = useMemo(() => {
    if (!searchTerm.trim()) return items;
    
    const term = searchTerm.toLowerCase();
    return items.filter(item => 
      item.title.toLowerCase().includes(term) ||
      item.description.toLowerCase().includes(term)
    );
  }, [items, searchTerm]);

  const highlightedResults = useMemo(() => {
    if (!searchTerm.trim()) {
      return searchResults.map(item => ({
        ...item,
        titleHTML: item.title,
        descHTML: item.description
      }));
    }

    const regex = new RegExp(`(${searchTerm})`, 'gi');
    
    return searchResults.map(item => ({
      ...item,
      titleHTML: item.title.replace(regex, '<mark>$1</mark>'),
      descHTML: item.description.replace(regex, '<mark>$1</mark>')
    }));
  }, [searchResults, searchTerm]);

  return (
    <div>
      <input
        type="search"
        value={searchTerm}
        onChange={e => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      
      <p>{highlightedResults.length} results</p>
      
      <ul>
        {highlightedResults.map(item => (
          <li key={item.id}>
            <h3 dangerouslySetInnerHTML={{ __html: item.titleHTML }} />
            <p dangerouslySetInnerHTML={{ __html: item.descHTML }} />
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Interview Questions

### Q1: What's the difference between useMemo and useCallback?

**Answer:**
```javascript
// useMemo - memoizes a VALUE
const value = useMemo(() => {
  return expensiveComputation(a, b);
}, [a, b]);

// useCallback - memoizes a FUNCTION
const callback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);

// They're equivalent in this way:
useCallback(fn, deps) === useMemo(() => fn, deps)
```

### Q2: When should you use useMemo?

**Answer:** Use useMemo when:
1. Computation is expensive (>50ms)
2. Working with large data sets (filtering/sorting)
3. Need referential equality for child props
4. Creating objects/arrays passed to memoized components

Don't use for simple calculations - the overhead isn't worth it.

### Q3: What happens if you omit dependencies?

**Answer:**
```javascript
// Empty array: Computed once, never updates
const value = useMemo(() => compute(data), []);
// Dangerous if 'data' changes!

// No array: Computed every render
const value = useMemo(() => compute(data));
// Defeats the purpose!

// Correct: Include all dependencies
const value = useMemo(() => compute(data), [data]);
```

## Key Takeaways

1. **useMemo caches computed values** - Recalculates only when dependencies change
2. **Profile before optimizing** - Measure actual performance impact
3. **Use for expensive operations** - Filtering, sorting, complex calculations
4. **Include all dependencies** - Use ESLint exhaustive-deps rule
5. **Consider referential equality** - Prevents unnecessary child re-renders
6. **Don't overuse** - Simple computations don't need memoization
7. **Combine with React.memo** - Maximum optimization for child components

## Resources

### Official Documentation
- [React useMemo Docs](https://react.dev/reference/react/useMemo)
- [You Might Not Need useMemo](https://react.dev/learn/you-might-not-need-an-effect)

### Articles
- [When to useMemo and useCallback](https://kentcdodds.com/blog/usememo-and-usecallback)
- [React useMemo Hook Ultimate Guide](https://www.robinwieruch.de/react-usememo-hook)

### Tools
- [React DevTools Profiler](https://react.dev/learn/react-developer-tools)
- [Why Did You Render](https://github.com/welldone-software/why-did-you-render)

---

**Next Steps:** Learn about useRef for mutable values and DOM manipulation, then explore React.memo for component-level optimization.
