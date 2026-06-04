# Render Props vs Hooks

## What Render Props Are

Render props is a pattern where a component receives a function as a prop and calls it to determine what to render — sharing logic via the render prop callback.

```tsx
// A component that tracks mouse position via render prop
function MouseTracker({ render }: { render: (x: number, y: number) => ReactNode }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  return (
    <div onMouseMove={e => setPosition({ x: e.clientX, y: e.clientY })}>
      {render(position.x, position.y)}
    </div>
  );
}

// Usage
<MouseTracker render={(x, y) => <p>Mouse at {x}, {y}</p>} />

// Or via children prop (most common form)
<MouseTracker>
  {(x, y) => <p>Mouse at {x}, {y}</p>}
</MouseTracker>
```

---

## The Same Logic as a Custom Hook

```tsx
// Custom hook version of MouseTracker
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    function handleMove(e: MouseEvent) {
      setPosition({ x: e.clientX, y: e.clientY });
    }
    document.addEventListener('mousemove', handleMove);
    return () => document.removeEventListener('mousemove', handleMove);
  }, []);

  return position;
}

// Usage — much simpler
function Cursor() {
  const { x, y } = useMousePosition();
  return <p>Mouse at {x}, {y}</p>;
}
```

---

## Comparison

| | Render Props | Custom Hook |
|---|---|---|
| Pattern level | Component | Function |
| Extra DOM layer | Sometimes yes | No |
| Composability | Nesting required | Flat composition |
| TypeScript complexity | Moderate | Simple |
| Access to component lifecycle | Yes | Yes (via hooks) |
| Conditional use | Cannot conditionally render (hooks rules) | Cannot call conditionally (same rules) |
| Readability | Can be verbose | Generally cleaner |

---

## When Render Props Are Still Useful

**1. Sharing rendering logic, not just data**

When the component also controls *what gets rendered*:

```tsx
// Downshift (autocomplete library) uses render props
// because it needs to control focus management, ARIA attributes, and list rendering
<Downshift
  onChange={selection => setItem(selection)}
  itemToString={item => (item ? item.value : '')}
>
  {({ getInputProps, getItemProps, getMenuProps, isOpen, inputValue }) => (
    <div>
      <input {...getInputProps()} />
      <ul {...getMenuProps()}>
        {isOpen && items
          .filter(item => !inputValue || item.value.includes(inputValue))
          .map((item, index) => (
            <li {...getItemProps({ key: item.id, index, item })}>
              {item.value}
            </li>
          ))}
      </ul>
    </div>
  )}
</Downshift>
```

**2. Inversion of control** — let the consumer control what's rendered:

```tsx
// DataList exposes data and lets consumer decide rendering
<DataList queryKey="users" fetchFn={fetchUsers}>
  {({ data, loading, error }) => (
    loading ? <Skeleton /> :
    error ? <ErrorView error={error} /> :
    <UserGrid users={data} />
  )}
</DataList>
```

**3. Libraries that predate hooks**: react-motion, Downshift, react-router's `<Route render>` (legacy) — still use render props for backwards compatibility.

---

## Modern Hybrid: Hook + Component

The modern pattern combines both — hook for logic, component for rendering:

```tsx
// Hook handles the data
function useDataList<T>(queryKey: string, fetchFn: () => Promise<T[]>) {
  return useQuery({ queryKey: [queryKey], queryFn: fetchFn });
}

// Component handles conditional rendering
function DataList<T>({ queryKey, fetchFn, children }: {
  queryKey: string;
  fetchFn: () => Promise<T[]>;
  children: (data: T[]) => ReactNode;
}) {
  const { data, isLoading, error } = useDataList(queryKey, fetchFn);
  if (isLoading) return <Skeleton />;
  if (error) return <ErrorView />;
  return <>{children(data!)}</>;
}
```

---

## Common Interview Questions

**Q: Why did the React community move from render props to hooks?**
Render props cause "callback hell" when composing multiple logic concerns. Hooks compose flatly — `useX(); useY(); useZ()` vs nested callbacks. Hooks also have better TypeScript inference and are easier to test.

**Q: Are render props an anti-pattern now?**
Not entirely. They're still the right tool when a component needs to control *rendering* (not just share data). Libraries like Downshift, react-aria, and headless UI components use render props because they need to inject attributes into the consumer's elements.

**Q: What's the difference between render props and children-as-function?**
Syntactically different, functionally the same pattern. `render={fn}` is an explicit prop; `{children}` called as a function is `children` being used as a function (children-as-function / function-as-child). Both are "render props pattern."
