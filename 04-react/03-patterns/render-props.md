# Render Props Pattern

## Overview

The **Render Props** pattern is a technique for sharing code between React components using a prop whose value is a function. The component calls this function instead of implementing its own render logic.

```jsx
<DataProvider render={data => <Display data={data} />} />
```

## Basic Concept

### Simple Example

```jsx
// Component with render prop
function Mouse({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return render(position);
}

// Usage
function App() {
  return (
    <Mouse
      render={({ x, y }) => (
        <div>Mouse position: {x}, {y}</div>
      )}
    />
  );
}
```

### Using children as Function

```jsx
// Alternative: children prop
function Mouse({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return children(position);
}

// Usage
function App() {
  return (
    <Mouse>
      {({ x, y }) => (
        <div>Mouse position: {x}, {y}</div>
      )}
    </Mouse>
  );
}
```

## Practical Examples

### Example 1: Toggle Logic

```jsx
function Toggle({ children }) {
  const [isOn, setIsOn] = useState(false);
  
  const toggle = () => setIsOn(prev => !prev);
  const on = () => setIsOn(true);
  const off = () => setIsOn(false);
  
  return children({
    isOn,
    toggle,
    on,
    off
  });
}

// Usage 1: Simple toggle
function SimpleToggle() {
  return (
    <Toggle>
      {({ isOn, toggle }) => (
        <button onClick={toggle}>
          {isOn ? 'ON' : 'OFF'}
        </button>
      )}
    </Toggle>
  );
}

// Usage 2: Modal
function ModalExample() {
  return (
    <Toggle>
      {({ isOn, on, off }) => (
        <>
          <button onClick={on}>Open Modal</button>
          {isOn && (
            <div className="modal">
              <p>Modal content</p>
              <button onClick={off}>Close</button>
            </div>
          )}
        </>
      )}
    </Toggle>
  );
}

// Usage 3: Accordion
function AccordionExample() {
  return (
    <div className="accordion">
      <Toggle>
        {({ isOn, toggle }) => (
          <>
            <button onClick={toggle}>
              Section 1 {isOn ? '▼' : '▶'}
            </button>
            {isOn && <div>Section 1 content</div>}
          </>
        )}
      </Toggle>
      
      <Toggle>
        {({ isOn, toggle }) => (
          <>
            <button onClick={toggle}>
              Section 2 {isOn ? '▼' : '▶'}
            </button>
            {isOn && <div>Section 2 content</div>}
          </>
        )}
      </Toggle>
    </div>
  );
}
```

### Example 2: Data Fetching

```jsx
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      })
      .catch(error => {
        setError(error);
        setLoading(false);
      });
  }, [url]);
  
  return children({ data, loading, error });
}

// Usage
function UserProfile() {
  return (
    <DataFetcher url="/api/user">
      {({ data, loading, error }) => {
        if (loading) return <Spinner />;
        if (error) return <Error message={error.message} />;
        return (
          <div>
            <h1>{data.name}</h1>
            <p>{data.email}</p>
          </div>
        );
      }}
    </DataFetcher>
  );
}

// Multiple data sources
function Dashboard() {
  return (
    <div>
      <DataFetcher url="/api/user">
        {({ data: user, loading: userLoading }) => (
          userLoading ? <Skeleton /> : <UserInfo user={user} />
        )}
      </DataFetcher>
      
      <DataFetcher url="/api/stats">
        {({ data: stats, loading: statsLoading }) => (
          statsLoading ? <Skeleton /> : <Stats data={stats} />
        )}
      </DataFetcher>
    </div>
  );
}
```

### Example 3: Form State Management

```jsx
function Form({ initialValues, onSubmit, children }) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  
  const handleChange = (name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
  };
  
  const handleBlur = (name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
  };
  
  const handleSubmit = (e) => {
    e.preventDefault();
    
    // Mark all as touched
    const allTouched = Object.keys(values).reduce(
      (acc, key) => ({ ...acc, [key]: true }),
      {}
    );
    setTouched(allTouched);
    
    // Validate and submit
    const validationErrors = validate(values);
    if (Object.keys(validationErrors).length === 0) {
      onSubmit(values);
    } else {
      setErrors(validationErrors);
    }
  };
  
  const validate = (vals) => {
    // Simple validation example
    const errs = {};
    if (!vals.email) errs.email = 'Required';
    if (!vals.password) errs.password = 'Required';
    return errs;
  };
  
  return children({
    values,
    errors,
    touched,
    handleChange,
    handleBlur,
    handleSubmit
  });
}

// Usage
function LoginForm() {
  return (
    <Form
      initialValues={{ email: '', password: '' }}
      onSubmit={(values) => console.log('Submit:', values)}
    >
      {({ values, errors, touched, handleChange, handleBlur, handleSubmit }) => (
        <form onSubmit={handleSubmit}>
          <div>
            <label>Email</label>
            <input
              type="email"
              value={values.email}
              onChange={(e) => handleChange('email', e.target.value)}
              onBlur={() => handleBlur('email')}
            />
            {touched.email && errors.email && (
              <span className="error">{errors.email}</span>
            )}
          </div>
          
          <div>
            <label>Password</label>
            <input
              type="password"
              value={values.password}
              onChange={(e) => handleChange('password', e.target.value)}
              onBlur={() => handleBlur('password')}
            />
            {touched.password && errors.password && (
              <span className="error">{errors.password}</span>
            )}
          </div>
          
          <button type="submit">Login</button>
        </form>
      )}
    </Form>
  );
}
```

### Example 4: Intersection Observer

```jsx
function Intersection({ threshold = 0, children }) {
  const [isIntersecting, setIsIntersecting] = useState(false);
  const [entry, setEntry] = useState(null);
  const targetRef = useRef(null);
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        setIsIntersecting(entry.isIntersecting);
        setEntry(entry);
      },
      { threshold }
    );
    
    const current = targetRef.current;
    if (current) {
      observer.observe(current);
    }
    
    return () => {
      if (current) {
        observer.unobserve(current);
      }
    };
  }, [threshold]);
  
  return children({
    ref: targetRef,
    isIntersecting,
    entry
  });
}

// Usage: Lazy load image
function LazyImage({ src, alt }) {
  return (
    <Intersection threshold={0.5}>
      {({ ref, isIntersecting }) => (
        <div ref={ref}>
          {isIntersecting ? (
            <img src={src} alt={alt} />
          ) : (
            <div className="placeholder">Loading...</div>
          )}
        </div>
      )}
    </Intersection>
  );
}

// Usage: Reveal on scroll animation
function RevealOnScroll({ children }) {
  return (
    <Intersection threshold={0.3}>
      {({ ref, isIntersecting, entry }) => (
        <div
          ref={ref}
          style={{
            opacity: isIntersecting ? 1 : 0,
            transform: isIntersecting ? 'translateY(0)' : 'translateY(50px)',
            transition: 'all 0.5s'
          }}
        >
          {children}
        </div>
      )}
    </Intersection>
  );
}
```

### Example 5: Media Query

```jsx
function MediaQuery({ query, children }) {
  const [matches, setMatches] = useState(false);
  
  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    
    setMatches(mediaQuery.matches);
    
    const handler = (e) => setMatches(e.matches);
    mediaQuery.addEventListener('change', handler);
    
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);
  
  return children(matches);
}

// Usage
function ResponsiveLayout() {
  return (
    <>
      <MediaQuery query="(max-width: 768px)">
        {(isMobile) => (
          isMobile ? <MobileNav /> : <DesktopNav />
        )}
      </MediaQuery>
      
      <MediaQuery query="(prefers-color-scheme: dark)">
        {(isDark) => (
          <div className={isDark ? 'dark-theme' : 'light-theme'}>
            Content
          </div>
        )}
      </MediaQuery>
    </>
  );
}
```

### Example 6: Debounced Input

```jsx
function DebouncedInput({ delay = 500, children }) {
  const [value, setValue] = useState('');
  const [debouncedValue, setDebouncedValue] = useState('');
  
  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return children({
    value,
    debouncedValue,
    onChange: (e) => setValue(e.target.value),
    setValue
  });
}

// Usage: Search with debounce
function Search() {
  return (
    <DebouncedInput delay={300}>
      {({ value, debouncedValue, onChange }) => (
        <>
          <input
            type="text"
            value={value}
            onChange={onChange}
            placeholder="Search..."
          />
          <SearchResults query={debouncedValue} />
        </>
      )}
    </DebouncedInput>
  );
}

function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    if (query) {
      fetchSearchResults(query).then(setResults);
    }
  }, [query]);
  
  return (
    <ul>
      {results.map(result => (
        <li key={result.id}>{result.title}</li>
      ))}
    </ul>
  );
}
```

### Example 7: Pagination

```jsx
function Pagination({ items, pageSize = 10, children }) {
  const [currentPage, setCurrentPage] = useState(1);
  
  const totalPages = Math.ceil(items.length / pageSize);
  const startIndex = (currentPage - 1) * pageSize;
  const endIndex = startIndex + pageSize;
  const currentItems = items.slice(startIndex, endIndex);
  
  const goToPage = (page) => {
    setCurrentPage(Math.max(1, Math.min(page, totalPages)));
  };
  
  const nextPage = () => goToPage(currentPage + 1);
  const prevPage = () => goToPage(currentPage - 1);
  
  return children({
    currentItems,
    currentPage,
    totalPages,
    goToPage,
    nextPage,
    prevPage,
    hasNext: currentPage < totalPages,
    hasPrev: currentPage > 1
  });
}

// Usage
function UserList({ users }) {
  return (
    <Pagination items={users} pageSize={5}>
      {({
        currentItems,
        currentPage,
        totalPages,
        nextPage,
        prevPage,
        hasNext,
        hasPrev
      }) => (
        <>
          <ul>
            {currentItems.map(user => (
              <li key={user.id}>{user.name}</li>
            ))}
          </ul>
          
          <div className="pagination">
            <button onClick={prevPage} disabled={!hasPrev}>
              Previous
            </button>
            <span>
              Page {currentPage} of {totalPages}
            </span>
            <button onClick={nextPage} disabled={!hasNext}>
              Next
            </button>
          </div>
        </>
      )}
    </Pagination>
  );
}
```

## Advanced Patterns

### Multiple Render Props

```jsx
function MultiRenderProp({ renderHeader, renderBody, renderFooter }) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  return (
    <div>
      {renderHeader && renderHeader(data)}
      {renderBody && renderBody(data)}
      {renderFooter && renderFooter(data)}
    </div>
  );
}

// Usage
<MultiRenderProp
  renderHeader={(data) => <h1>{data?.title}</h1>}
  renderBody={(data) => <p>{data?.content}</p>}
  renderFooter={(data) => <small>{data?.date}</small>}
/>
```

### Render Props with Hooks

```jsx
// Combine render props with hooks
function useDataFetcher(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, [url]);
  
  return { data, loading };
}

function DataRenderer({ url, render }) {
  const { data, loading } = useDataFetcher(url);
  return render({ data, loading });
}
```

### Composing Render Props

```jsx
function App() {
  return (
    <Toggle>
      {({ isOn, toggle }) => (
        <DataFetcher url="/api/data">
          {({ data, loading }) => (
            <div>
              <button onClick={toggle}>Toggle</button>
              {isOn && !loading && <Display data={data} />}
            </div>
          )}
        </DataFetcher>
      )}
    </Toggle>
  );
}
```

## TypeScript

```tsx
interface RenderPropProps<T> {
  children: (data: T) => ReactNode;
}

function DataFetcher<T>({ url, children }: { 
  url: string; 
  children: (state: { 
    data: T | null; 
    loading: boolean; 
    error: Error | null;
  }) => ReactNode;
}) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err);
        setLoading(false);
      });
  }, [url]);
  
  return children({ data, loading, error });
}

// Usage with type safety
interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile() {
  return (
    <DataFetcher<User> url="/api/user">
      {({ data, loading, error }) => {
        if (loading) return <div>Loading...</div>;
        if (error) return <div>Error: {error.message}</div>;
        if (!data) return null;
        
        return (
          <div>
            <h1>{data.name}</h1>
            <p>{data.email}</p>
          </div>
        );
      }}
    </DataFetcher>
  );
}
```

## Advantages

1. **Flexibility**: Consumer decides what to render
2. **Reusability**: Logic shared across different UIs
3. **Separation**: Logic separate from presentation
4. **Composability**: Can nest multiple render props
5. **Type Safety**: Full TypeScript support

## Disadvantages

1. **Callback Hell**: Deep nesting can be hard to read
2. **Performance**: Creates new function on each render
3. **Verbosity**: More code than alternatives
4. **Debugging**: Stack traces can be confusing

## Modern Alternatives

### Custom Hooks (Preferred in Modern React)

```jsx
// Render prop
<Mouse render={({ x, y }) => <div>{x}, {y}</div>} />

// Custom hook (simpler)
function App() {
  const { x, y } = useMouse();
  return <div>{x}, {y}</div>;
}

function useMouse() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return position;
}
```

## When to Use

### Use Render Props When:
- Building a library that needs maximum flexibility
- Multiple consumers need different UIs for same logic
- Need to render different things based on state

### Use Custom Hooks When:
- Building application code
- Logic is primary concern
- Want simpler, cleaner code
- No need for different render implementations

## Resources

- [React Docs: Render Props](https://legacy.reactjs.org/docs/render-props.html)
- [Use a Render Prop!](https://cdb.reacttraining.com/use-a-render-prop-50de598f11ce)
- [Custom Hooks vs Render Props](https://kentcdodds.com/blog/react-hooks-whats-going-to-happen-to-render-props)
