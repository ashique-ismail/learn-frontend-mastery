# Keys & Reconciliation

## Overview

Keys are a special string attribute that help React identify which items in a list have changed, been added, or been removed. While they might seem like a simple concept, keys are fundamental to React's reconciliation algorithm and have profound implications for application performance, correctness, and user experience.

This guide explores the deep mechanics of how keys work, why they matter, common pitfalls, and advanced patterns for complex scenarios.

## Why Keys Exist: The List Reconciliation Problem

When React reconciles a list of children, it needs to match old children with new children to determine what changed. Without keys, React uses positional matching.

```javascript
// Example 1: The problem without keys
function TodoList() {
  const [todos, setTodos] = useState([
    { text: 'Buy milk', completed: false },
    { text: 'Walk dog', completed: false }
  ]);
  
  return (
    <ul>
      {todos.map((todo, index) => (
        // No key provided
        <TodoItem todo={todo} />
      ))}
    </ul>
  );
}

// Initial render creates:
// Position 0: <TodoItem todo="Buy milk" />
// Position 1: <TodoItem todo="Walk dog" />

// After adding "Take out trash" at the beginning:
const newTodos = [
  { text: 'Take out trash', completed: false },
  { text: 'Buy milk', completed: false },
  { text: 'Walk dog', completed: false }
];

// React reconciles by position:
// Position 0: "Buy milk" → "Take out trash" (UPDATE)
// Position 1: "Walk dog" → "Buy milk" (UPDATE)
// Position 2: undefined → "Walk dog" (INSERT)
//
// Result: React updates 2 existing items and inserts 1
// Problem: If TodoItem had internal state (expanded, focused),
// it would be incorrectly transferred
```

Keys solve this by giving React a way to track element identity:

```javascript
// Example 2: Solution with keys
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Buy milk', completed: false },
    { id: 2, text: 'Walk dog', completed: false }
  ]);
  
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

// After adding item at beginning:
const newTodos = [
  { id: 3, text: 'Take out trash', completed: false },
  { id: 1, text: 'Buy milk', completed: false },
  { id: 2, text: 'Walk dog', completed: false }
];

// React reconciles by key:
// key=1: Exists in old list, reuse component, move to position 1
// key=2: Exists in old list, reuse component, move to position 2
// key=3: New key, create new component at position 0
//
// Result: React reuses 2 components and creates 1 new one
// State is preserved correctly
```

## How React Uses Keys Internally

React builds a map of old children keyed by their keys, then processes new children:

```javascript
// Example 3: Simplified key reconciliation algorithm
function reconcileChildrenWithKeys(oldChildren, newChildren) {
  // Step 1: Build a map of old children by key
  const oldChildrenMap = new Map();
  oldChildren.forEach((child, index) => {
    const key = child.key ?? index; // Use index as fallback
    oldChildrenMap.set(key, child);
  });
  
  // Step 2: Process new children
  const result = [];
  const usedKeys = new Set();
  
  newChildren.forEach((newChild, index) => {
    const key = newChild.key ?? index;
    
    if (oldChildrenMap.has(key)) {
      // Reuse existing child
      const oldChild = oldChildrenMap.get(key);
      result.push({
        type: 'UPDATE',
        fiber: oldChild,
        newProps: newChild.props
      });
      usedKeys.add(key);
    } else {
      // Create new child
      result.push({
        type: 'INSERT',
        element: newChild
      });
    }
  });
  
  // Step 3: Delete unused old children
  oldChildren.forEach(child => {
    const key = child.key ?? child.index;
    if (!usedKeys.has(key)) {
      result.push({
        type: 'DELETE',
        fiber: child
      });
    }
  });
  
  return result;
}
```

## The Index-as-Key Anti-Pattern

Using array indices as keys is React's most common pitfall:

```javascript
// Example 4: Why index keys fail
function MessageList({ messages }) {
  const [localState] = useState({});
  
  return (
    <div>
      {messages.map((msg, index) => (
        <Message 
          key={index}  // BAD: Using index
          message={msg}
          onExpand={() => localState[index] = true}
        />
      ))}
    </div>
  );
}

// Initial messages:
// [0] { id: 'a', text: 'Hello' }
// [1] { id: 'b', text: 'World' }

// User expands message at index 1 (id='b')
// localState[1] = true

// After removing first message:
// [0] { id: 'b', text: 'World' }

// React reconciliation:
// key=0: "Hello" → "World" (UPDATE with wrong state)
// key=1: deleted
//
// Result: The wrong message appears expanded!
```

```javascript
// Example 5: Index keys with reordering
function DraggableList({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        <DraggableItem
          key={index}  // BAD
          item={item}
          dragging={false}  // internal state
        />
      ))}
    </div>
  );
}

// Initial: [A, B, C] with indices [0, 1, 2]
// User drags B to first position: [B, A, C]

// React reconciliation with index keys:
// key=0: A → B (updates content, but preserves A's component instance)
// key=1: B → A (updates content, but preserves B's component instance)
// key=2: C → C (no change)
//
// Problem: Drag state is attached to wrong items
// The component that was dragging (B at index 1) is now rendering A
```

```javascript
// Example 6: Correct - stable keys
function DraggableList({ items }) {
  return (
    <div>
      {items.map(item => (
        <DraggableItem
          key={item.id}  // GOOD
          item={item}
        />
      ))}
    </div>
  );
}

// After reorder [B, A, C]:
// key=B: Move component to position 0 (preserves B's state)
// key=A: Move component to position 1 (preserves A's state)
// key=C: Keep at position 2
//
// Drag state stays with correct component
```

## When Index Keys Are Acceptable

Despite the warnings, index keys are sometimes appropriate:

```javascript
// Example 7: Static list - OK to use index
function StaticLabels({ labels }) {
  // List never changes order, adds, or removes items
  return (
    <div>
      {labels.map((label, index) => (
        <span key={index}>{label}</span>
      ))}
    </div>
  );
}

// Example 8: Append-only list - OK to use index
function AppendOnlyLog({ entries }) {
  // New items only added at end, never reordered or removed
  return (
    <ul>
      {entries.map((entry, index) => (
        <li key={index}>{entry.message}</li>
      ))}
    </ul>
  );
}

// Example 9: Stateless children - OK to use index
function SimpleList({ items }) {
  return (
    <ul>
      {items.map((item, index) => (
        // No component state, refs, or side effects
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
}
```

The rule: Index keys are safe when the list is static OR append-only AND children have no state/refs.

## Key Requirements and Constraints

```javascript
// Example 10: Keys must be unique among siblings
function BadUniqueKeys({ items }) {
  return (
    <div>
      {items.map(item => (
        // BAD: category might repeat
        <Card key={item.category} item={item} />
      ))}
    </div>
  );
}

// React warning: "Encountered two children with the same key"
// Results in undefined behavior - React can't distinguish elements

// Example 11: Keys must be stable
function BadStableKeys({ items }) {
  return (
    <div>
      {items.map(item => (
        // BAD: Math.random() creates new key every render
        <Card key={Math.random()} item={item} />
      ))}
    </div>
  );
}

// Every render creates new keys
// React treats all children as new, causing full remount every time

// Example 12: Keys should be strings or numbers
function GoodKeyTypes({ items }) {
  return (
    <>
      {/* Good: number */}
      {items.map(item => <Card key={item.id} item={item} />)}
      
      {/* Good: string */}
      {items.map(item => <Card key={item.uuid} item={item} />)}
      
      {/* Acceptable: stringified compound */}
      {items.map(item => (
        <Card key={`${item.category}-${item.id}`} item={item} />
      ))}
    </>
  );
}
```

## Advanced Key Patterns

### Compound Keys for Nested Lists

```javascript
// Example 13: Nested lists need compound keys
function NestedList({ categories }) {
  return (
    <div>
      {categories.map(category => (
        <div key={category.id}>
          <h2>{category.name}</h2>
          <ul>
            {category.items.map(item => (
              // Compound key: category + item
              <li key={`${category.id}-${item.id}`}>
                {item.name}
              </li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
}
```

### Using Keys to Force Remount

```javascript
// Example 14: Keys for intentional remounting
function UserProfile({ userId }) {
  // Key changes when userId changes
  // Forces complete remount, resetting all state
  return <ProfileForm key={userId} userId={userId} />;
}

function ProfileForm({ userId }) {
  const [formData, setFormData] = useState({});
  
  // When key changes:
  // 1. Old ProfileForm unmounts (useEffect cleanup)
  // 2. New ProfileForm mounts (fresh state)
  // 3. Form data automatically resets
  
  return <form>...</form>;
}

// Example 15: Keys for animations
function AnimatedList({ items }) {
  return (
    <TransitionGroup>
      {items.map(item => (
        // Key enables exit animations
        <CSSTransition key={item.id} timeout={300}>
          <Card item={item} />
        </CSSTransition>
      ))}
    </TransitionGroup>
  );
}

// When item removed:
// 1. React keeps component mounted during exit animation
// 2. After timeout, unmounts component
// Without stable key, can't track which item is exiting
```

### Keys with Fragments

```javascript
// Example 16: Keyed fragments
function GroupedList({ groups }) {
  return (
    <dl>
      {groups.map(group => (
        <React.Fragment key={group.id}>
          <dt>{group.term}</dt>
          <dd>{group.definition}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}

// Fragments can have keys when mapping
// Allows reconciling multi-element groups

// Example 17: Short syntax doesn't support keys
function NoKeyFragment({ items }) {
  return (
    <>
      {items.map(item => (
        // ERROR: Short syntax doesn't support keys
        <>
          <div>{item.title}</div>
          <div>{item.content}</div>
        </>
      ))}
    </>
  );
}

// Must use explicit React.Fragment for keys
```

## Keys and Component State

```javascript
// Example 18: How keys affect state preservation
function Counter({ id, initialCount }) {
  const [count, setCount] = useState(initialCount);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

function App() {
  const [items, setItems] = useState([
    { id: 'a', count: 0 },
    { id: 'b', count: 5 }
  ]);
  
  return (
    <div>
      {items.map(item => (
        <Counter
          key={item.id}
          id={item.id}
          initialCount={item.count}
        />
      ))}
      <button onClick={() => setItems([items[1], items[0]])}>
        Swap
      </button>
    </div>
  );
}

// With key={item.id}:
// - User increments Counter A to 3
// - User increments Counter B to 8
// - Click Swap
// - Counter A (now second) still shows 3
// - Counter B (now first) still shows 8
// State preserved correctly

// Without keys (or with index keys):
// - After swap, first counter would show 3 (wrong!)
// - Second counter would show 8 (wrong!)
// State doesn't follow the logical item
```

## Keys and Refs

```javascript
// Example 19: Keys with refs
function FocusableList({ items }) {
  const inputRefs = useRef({});
  
  const focusItem = (id) => {
    inputRefs.current[id]?.focus();
  };
  
  return (
    <div>
      {items.map(item => (
        <input
          key={item.id}
          ref={el => inputRefs.current[item.id] = el}
          defaultValue={item.text}
        />
      ))}
    </div>
  );
}

// Stable keys ensure refs point to correct DOM nodes
// Even after reordering, inputRefs.current[id] stays accurate
```

## Performance Implications of Keys

```javascript
// Example 20: Key choice affects performance
function LargeList({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        <ExpensiveComponent
          key={item.id}  // vs key={index}
          item={item}
        />
      ))}
    </div>
  );
}

// Scenario: items = 10,000 elements, one item updated

// With stable ID keys:
// - React builds key map: O(n) - 10,000 operations
// - Reconciles list: O(n) - 10,000 comparisons
// - Finds 1 changed prop
// - Re-renders 1 component
// - Total: ~20,000 operations, 1 expensive render

// With index keys after reorder:
// - React compares by position: O(n) - 10,000 comparisons
// - Detects prop changes at every position
// - Re-renders all 10,000 components
// - Total: ~10,000 operations, 10,000 expensive renders
// Much slower!

// The key (pun intended) is minimizing renders, not comparisons
```

## Debugging Key Issues

```javascript
// Example 21: React DevTools shows key warnings
// Open console to see warnings

function BrokenKeys({ items }) {
  return (
    <div>
      {items.map(item => (
        // Duplicate keys
        <Card key={item.category} item={item} />
      ))}
    </div>
  );
}

// Console warning:
// "Warning: Encountered two children with the same key, `electronics`.
// Keys should be unique so that components maintain their identity across updates."

// Example 22: Detecting index key problems
function DebugKeys({ items }) {
  useEffect(() => {
    console.log('Component mounted');
    return () => console.log('Component unmounted');
  }, []);
  
  return <div>{items[0].text}</div>;
}

function Parent({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        <DebugKeys key={index} items={[item]} />
      ))}
    </div>
  );
}

// After reorder, watch console
// Unexpected unmount/mount pairs indicate key issues
```

## Common Misconceptions

1. **"Keys are only needed to avoid console warnings"** - False. Keys fundamentally change how React reconciles lists. Without proper keys, you'll get bugs, not just warnings.

2. **"Index keys are always bad"** - False. They're fine for static, append-only, or stateless lists.

3. **"Keys need to be globally unique"** - False. Keys only need to be unique among siblings. Different lists can have overlapping keys.

4. **"Changing a key is expensive"** - Partly true. Changing a key forces a remount, which is more expensive than an update, but sometimes that's exactly what you want.

5. **"Keys are a React-specific concept"** - False. The concept of stable identity in list reconciliation exists in many UI frameworks (Vue, Angular, SwiftUI, etc.).

## Performance Implications

1. **Key Stability** - Unstable keys (like Math.random()) cause complete remounts every render, destroying performance.

2. **Key Uniqueness** - Non-unique keys cause undefined behavior and can corrupt component state.

3. **Key Computation** - Complex key generation (hashing, JSON.stringify) adds overhead. Simple string/number keys are best.

4. **Reconciliation Efficiency** - Good keys enable O(n) reconciliation. Bad keys degrade to O(n²) in worst cases.

## Interview Questions

1. **Q: Explain why keys are necessary in React lists.**
   A: Keys give React a way to identify which items have changed, been added, or removed. Without keys, React uses positional matching, which breaks when lists reorder because component state gets associated with the wrong items. Keys enable efficient reconciliation by tracking element identity across renders.

2. **Q: What happens if you use non-unique keys?**
   A: React cannot distinguish between elements with the same key, leading to undefined behavior. Components may not update when they should, state may be shared incorrectly between components, and React will show console warnings. In essence, the reconciliation algorithm breaks down.

3. **Q: When is it acceptable to use index as a key?**
   A: Index keys are safe when (1) the list is static and never changes, (2) items are only appended to the end, or (3) children are stateless and have no refs. If the list can reorder or items can be inserted/removed at arbitrary positions, index keys will cause bugs.

4. **Q: How does React use keys during reconciliation?**
   A: React builds a map of old children indexed by key, then iterates through new children. For each new child, if its key exists in the old map, React reuses that component instance (potentially moving it). New keys result in new components being created. Keys not present in the new list result in unmounts.

5. **Q: Why would you intentionally change a component's key?**
   A: Changing a key forces React to unmount the old component and mount a new one, completely resetting state. This is useful for scenarios like switching between different users' forms, resetting animations, or clearing complex component state without manual cleanup.

6. **Q: What's the difference between a key and an id prop?**
   A: Keys are a special prop that React uses internally for reconciliation and never passes to components. An id prop is just a regular prop that the component receives. Keys affect React's behavior; id props affect your component's logic.

7. **Q: How do keys affect component performance?**
   A: Good keys (stable, unique) enable efficient reconciliation—React can quickly determine what changed and minimize DOM operations. Bad keys (index on dynamic lists, non-unique, or unstable) force unnecessary re-renders or full remounts, severely degrading performance.

8. **Q: Can fragments have keys?**
   A: Yes, but only when using the explicit React.Fragment syntax, not the short <> syntax. Keyed fragments are useful when mapping over data that produces multiple sibling elements, allowing React to reconcile groups of elements together.

## Key Takeaways

1. Keys provide stable identity for list items across renders
2. React uses keys to efficiently reconcile lists by tracking element identity
3. Index keys cause bugs in dynamic lists but are acceptable for static ones
4. Keys must be unique among siblings and stable across renders
5. Changing a component's key forces a complete remount and state reset
6. Non-unique or unstable keys break reconciliation and harm performance
7. Keys affect internal React behavior; they're not passed as props to components
8. Proper key selection is critical for correct behavior and optimal performance

## Resources

- [React Lists and Keys Documentation](https://react.dev/learn/rendering-lists)
- [Why Keys Matter - React Docs](https://react.dev/learn/preserving-and-resetting-state#option-2-resetting-state-with-a-key)
- [Index as Key is an Anti-Pattern](https://robinpokorny.com/blog/index-as-a-key-is-an-anti-pattern/)
- [Understanding React's key Prop](https://kentcdodds.com/blog/understanding-reacts-key-prop)
- [React Reconciliation Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactChildFiber.js)
