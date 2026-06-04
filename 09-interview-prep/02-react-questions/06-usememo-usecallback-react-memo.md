# useMemo vs useCallback vs React.memo

## The Core Distinction

All three are memoization tools, but they operate at different levels:

| | What it memoizes | Returns |
|---|---|---|
| `useMemo` | The **result** of a computation | Computed value |
| `useCallback` | A **function reference** | Stable function |
| `React.memo` | A **component's render output** | Wrapped component |

---

## `useMemo` — Memoize Computed Values

```tsx
const filteredItems = useMemo(
  () => items.filter(item => item.active && item.name.includes(query)),
  [items, query]  // recompute only when these change
);
```

Use when:
- The computation is **expensive** (sorting/filtering large arrays, complex math)
- The value is used as a **dependency** in another hook or as a **prop** that would cause re-renders

When NOT to use:
- For cheap computations (`const x = a + b` — the memo overhead costs more than the computation)
- When the inputs change on every render anyway (memoization never hits)

---

## `useCallback` — Memoize Function References

```tsx
const handleSubmit = useCallback(
  async (data: FormData) => {
    await api.save(data);
    refetch();
  },
  [refetch]  // recreate only when refetch changes
);
```

The function body is the same every render, but without `useCallback` a **new function object** is created each time. This matters when the function is:
- Passed as a prop to `React.memo` children (new reference = re-render)
- Used in a `useEffect` dependency array
- Passed to a child that uses it in its own `useEffect`

```tsx
// Without useCallback — onDelete is a new function every render
// → ChildRow re-renders even when nothing changed
function ParentList({ items }) {
  const onDelete = (id) => deleteItem(id); // new ref every render
  return items.map(item => <ChildRow key={item.id} onDelete={onDelete} />);
}

// With useCallback — stable reference
function ParentList({ items }) {
  const onDelete = useCallback((id) => deleteItem(id), []);
  return items.map(item => <ChildRow key={item.id} onDelete={onDelete} />);
}
```

---

## `React.memo` — Memoize Component Renders

```tsx
const UserCard = React.memo(function UserCard({ user, onEdit }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={onEdit}>Edit</button>
    </div>
  );
});
```

`React.memo` wraps a component and performs **shallow prop comparison** before re-rendering. If props haven't changed (by reference), the component skips re-rendering.

```tsx
// Custom comparison function (when shallow compare isn't enough)
const UserCard = React.memo(
  function UserCard({ user }) { ... },
  (prevProps, nextProps) => prevProps.user.id === nextProps.user.id
);
```

---

## They Work Together

`React.memo` only helps if props are **stable**. Without `useCallback`/`useMemo`, parent re-renders create new prop references that defeat `React.memo`:

```tsx
function ParentList({ items }) {
  // ❌ New function every render → React.memo on ChildRow is useless
  const handleEdit = (id) => openEditor(id);

  // ✅ Stable reference → React.memo on ChildRow works
  const handleEdit = useCallback((id) => openEditor(id), []);

  return items.map(item => (
    <UserCard key={item.id} user={item} onEdit={handleEdit} />
  ));
}

const UserCard = React.memo(function UserCard({ user, onEdit }) { ... });
```

---

## When NOT to Memoize (Anti-Patterns)

```tsx
// ❌ Premature optimization — overhead > benefit for simple values
const doubled = useMemo(() => value * 2, [value]);
const greeting = useMemo(() => `Hello, ${name}!`, [name]);

// ❌ useCallback on a function that's always recreated anyway
const handler = useCallback(
  (e) => setValue(e.target.value),
  [setValue] // setValue changes every render (not from useState)
);

// ❌ React.memo on a component that always receives new object props
<MemoizedComponent config={{ debug: true }} /> // new object each render
```

---

## Common Interview Questions

**Q: Does `useCallback(fn, [])` make the function never re-created?**
Yes — empty dependency array means the function is created once on mount. Be careful: if the function closes over state or props, it will capture their initial values (stale closure).

**Q: If `useMemo` and `useCallback` have a cost, when do they pay off?**
`useMemo`: when the computation takes measurable time (>1ms) or the result is used in dependency arrays of other hooks. `useCallback`: primarily when passed to `React.memo` children or used in `useEffect` dependencies. Profile first — most components don't need memoization.

**Q: What's the difference between `useCallback(fn, deps)` and `useMemo(() => fn, deps)`?**
They're equivalent. `useCallback(fn, deps)` is syntactic sugar for `useMemo(() => fn, deps)`.
