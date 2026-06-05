# Lists and Keys: Stable Identity Rules

## The Idea

**In plain English:** When React shows a list of items on screen, it needs a way to tell each item apart so it knows what changed, what was added, and what was removed. A "key" is a unique label you give each item — like a name tag — so React never gets them confused.

**Real-world analogy:** Imagine a teacher handing back graded tests to a class of students. Each test has the student's name written on it, so the teacher knows exactly whose test is whose — even if students change seats.

- The student's name on the test = the `key` prop on each list item
- Each graded test = each React element in the list
- The teacher handing back tests = React updating the screen

---

## Overview

Rendering lists is a fundamental operation in React applications. Keys are special attributes that help React identify which items have changed, been added, or removed. Understanding how keys work is crucial for writing performant and bug-free React applications.

## Rendering Lists

### Basic List Rendering

Use JavaScript's `map()` to transform arrays into lists of elements:

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.text}
        </li>
      ))}
    </ul>
  );
}

// Usage
const todos = [
  { id: 1, text: 'Learn React' },
  { id: 2, text: 'Build an app' },
  { id: 3, text: 'Deploy to production' }
];

<TodoList todos={todos} />
```

### Extracting List Items

Extract list items into separate components for clarity:

```jsx
function TodoItem({ todo }) {
  return (
    <li className="todo-item">
      <span>{todo.text}</span>
      {todo.completed && <span className="check">✓</span>}
    </li>
  );
}

function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}
```

### Complex List Items

```jsx
function UserCard({ user }) {
  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <div className="user-info">
        <h3>{user.name}</h3>
        <p>{user.email}</p>
        <p>Joined: {user.joinDate}</p>
      </div>
      <button>View Profile</button>
    </div>
  );
}

function UserList({ users }) {
  return (
    <div className="user-list">
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}
```

## The Key Prop

### Why Keys Matter

Keys help React identify which items have changed:

```jsx
// Without proper keys, React may:
// 1. Re-render unnecessarily
// 2. Lose component state
// 3. Apply updates to wrong elements
// 4. Have performance issues

// Initial render
<ul>
  <li key="1">Apple</li>
  <li key="2">Banana</li>
  <li key="3">Cherry</li>
</ul>

// After adding item at beginning
<ul>
  <li key="0">Apricot</li>  // New item
  <li key="1">Apple</li>     // Same key, React knows it moved
  <li key="2">Banana</li>
  <li key="3">Cherry</li>
</ul>
```

### Where Keys Are Needed

```jsx
// In map()
{items.map(item => <Item key={item.id} {...item} />)}

// In fragments within map()
{items.map(item => (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </React.Fragment>
))}

// Multiple children that are dynamically generated
<ul>
  {items.map(item => <li key={item.id}>{item.name}</li>)}
</ul>
```

### Where Keys Are NOT Needed

```jsx
// Static children don't need keys
<div>
  <Header />
  <Content />
  <Footer />
</div>

// Short fragment syntax (but can't use keys)
<>
  <Header />
  <Content />
</>
```

## Choosing Keys

### 1. Use Stable IDs (Best)

```jsx
// Backend-provided IDs
function ProductList({ products }) {
  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

// Database IDs, UUIDs, etc.
const products = [
  { id: 'prod_1234', name: 'Laptop' },
  { id: 'prod_5678', name: 'Mouse' }
];
```

### 2. Generate Stable IDs

```jsx
// Use a library like uuid
import { v4 as uuidv4 } from 'uuid';

function TodoForm() {
  const [todos, setTodos] = useState([]);
  
  const addTodo = (text) => {
    setTodos([
      ...todos,
      { id: uuidv4(), text, completed: false }
    ]);
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}
```

### 3. Use Index (Last Resort)

```jsx
// ONLY use index when:
// - List is static (never reordered, filtered, or modified)
// - Items have no IDs
// - Items are simple values

function StaticList({ items }) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
}

// Example: Days of week (never changes)
const DAYS = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'];

function WeekDays() {
  return (
    <ul>
      {DAYS.map((day, index) => (
        <li key={index}>{day}</li>
      ))}
    </ul>
  );
}
```

## Problems with Index as Key

### Example 1: Component State Loss

```jsx
// Using index as key
function BadTodoList() {
  const [todos, setTodos] = useState([
    { text: 'Learn React' },
    { text: 'Build app' }
  ]);
  
  const [editingIndex, setEditingIndex] = useState(null);
  
  const addTodo = () => {
    // Add at beginning
    setTodos([{ text: 'New todo' }, ...todos]);
  };
  
  return (
    <div>
      <button onClick={addTodo}>Add</button>
      {todos.map((todo, index) => (
        // Problem: index changes when item is added
        <TodoItem
          key={index}
          todo={todo}
          isEditing={editingIndex === index}
        />
      ))}
    </div>
  );
}

// After adding item:
// - Old index 0 becomes 1
// - EditingIndex still points to 0
// - Wrong item is now in edit mode!
```

### Example 2: Input State Loss

```jsx
function BadExample() {
  const [items, setItems] = useState(['A', 'B', 'C']);
  
  const reverse = () => setItems([...items].reverse());
  
  return (
    <div>
      <button onClick={reverse}>Reverse</button>
      {items.map((item, index) => (
        <div key={index}>
          <span>{item}</span>
          {/* Input value doesn't follow item when reordered */}
          <input placeholder={`Input for ${item}`} />
        </div>
      ))}
    </div>
  );
}

// Before reverse: Input in first position
// After reverse: Input stays in first position but different item!
```

### Example 3: Animation Issues

```jsx
// Index keys break animations
function AnimatedList({ items }) {
  return (
    <div>
      {items.map((item, index) => (
        // Animation classes get confused
        <div
          key={index}
          className="item-enter"
          style={{ animationDelay: `${index * 100}ms` }}
        >
          {item}
        </div>
      ))}
    </div>
  );
}
```

### Correct Solutions

```jsx
// Solution: Use stable IDs
function GoodTodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn React' },
    { id: 2, text: 'Build app' }
  ]);
  
  const [editingId, setEditingId] = useState(null);
  
  const addTodo = () => {
    const newId = Math.max(...todos.map(t => t.id)) + 1;
    setTodos([{ id: newId, text: 'New todo' }, ...todos]);
  };
  
  return (
    <div>
      <button onClick={addTodo}>Add</button>
      {todos.map(todo => (
        // Stable ID maintains state correctly
        <TodoItem
          key={todo.id}
          todo={todo}
          isEditing={editingId === todo.id}
        />
      ))}
    </div>
  );
}
```

## Key Rules and Best Practices

### 1. Keys Must Be Unique Among Siblings

```jsx
// Correct: Unique among siblings
<ul>
  <li key="1">Item 1</li>
  <li key="2">Item 2</li>
</ul>

// Different lists can have same keys
<div>
  <ul>
    <li key="1">List 1 - Item 1</li>
  </ul>
  <ul>
    <li key="1">List 2 - Item 1</li>  {/* OK: Different list */}
  </ul>
</div>

// Wrong: Duplicate keys in same list
<ul>
  <li key="1">Item 1</li>
  <li key="1">Item 2</li>  {/* ERROR: Duplicate key */}
</ul>
```

### 2. Keys Must Be Stable

```jsx
// Wrong: Random keys change every render
{items.map(item => (
  <Item key={Math.random()} item={item} />
))}

// Wrong: Generating keys in render
{items.map(item => (
  <Item key={uuidv4()} item={item} />
))}

// Correct: Stable ID from data
{items.map(item => (
  <Item key={item.id} item={item} />
))}
```

### 3. Don't Use Spread Index

```jsx
// Wrong: Index-based key with spread
{items.map((item, index) => (
  <Item key={index} {...item} />
))}

// If item has an id property, it won't be used as key!

// Correct: Use item's ID
{items.map(item => (
  <Item key={item.id} {...item} />
))}
```

### 4. Keys Are Not Props

```jsx
function Item({ key, name }) {
  // key is undefined here - React doesn't pass it as prop
  console.log(key); // undefined
  return <div>{name}</div>;
}

// If you need the ID, pass it separately
function Item({ id, name }) {
  console.log(id); // Works
  return <div data-id={id}>{name}</div>;
}

<Item key={item.id} id={item.id} name={item.name} />
```

### 5. Fragments with Keys

```jsx
// Short syntax can't accept keys
{items.map(item => (
  <>  {/* Error: Can't add key to short syntax */}
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </>
))}

// Use explicit React.Fragment
{items.map(item => (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </React.Fragment>
))}
```

## Advanced Key Patterns

### 1. Composite Keys

When items don't have unique IDs:

```jsx
function UserPosts({ userId, posts }) {
  return (
    <div>
      {posts.map(post => (
        // Combine user and post ID for unique key
        <Post key={`${userId}-${post.id}`} post={post} />
      ))}
    </div>
  );
}
```

### 2. Nested Lists

```jsx
function CategoryList({ categories }) {
  return (
    <div>
      {categories.map(category => (
        <div key={category.id}>
          <h2>{category.name}</h2>
          <ul>
            {category.items.map(item => (
              // Can reuse IDs across different lists
              <li key={item.id}>{item.name}</li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
}
```

### 3. Conditional Lists

```jsx
function FilteredList({ items, showCompleted }) {
  const filteredItems = showCompleted
    ? items
    : items.filter(item => !item.completed);
  
  // Keys remain stable even when filtered
  return (
    <ul>
      {filteredItems.map(item => (
        <li key={item.id}>{item.text}</li>
      ))}
    </ul>
  );
}
```

### 4. Dynamic Reordering

```jsx
function SortableList({ items, sortBy }) {
  const sortedItems = [...items].sort((a, b) => {
    return a[sortBy] > b[sortBy] ? 1 : -1;
  });
  
  // Keys keep component state when reordering
  return (
    <ul>
      {sortedItems.map(item => (
        <SortableItem key={item.id} item={item} />
      ))}
    </ul>
  );
}
```

## Performance Considerations

### 1. Key Stability Affects Performance

```jsx
// Bad: React re-renders everything
function BadList({ items }) {
  return (
    <ul>
      {items.map((item, index) => (
        <ExpensiveItem key={Math.random()} item={item} />
      ))}
    </ul>
  );
}

// Good: React reuses existing elements
function GoodList({ items }) {
  return (
    <ul>
      {items.map(item => (
        <ExpensiveItem key={item.id} item={item} />
      ))}
    </ul>
  );
}
```

### 2. Memoization with Keys

```jsx
// Memoized component only re-renders when props change
const TodoItem = memo(function TodoItem({ todo }) {
  return <li>{todo.text}</li>;
});

function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        // Stable key + memo = optimal performance
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}
```

### 3. Virtual Scrolling

For very large lists, use virtualization:

```jsx
import { FixedSizeList } from 'react-window';

function LargeList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {/* Keys handled internally by react-window */}
      {items[index].name}
    </div>
  );
  
  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

## Common Patterns

### 1. Adding Items

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'First' },
    { id: 2, text: 'Second' }
  ]);
  
  const addTodo = (text) => {
    const newTodo = {
      id: Date.now(), // Simple ID generation
      text,
      completed: false
    };
    setTodos([...todos, newTodo]);
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

### 2. Removing Items

```jsx
function TodoList() {
  const [todos, setTodos] = useState([...]);
  
  const removeTodo = (id) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.text}
          <button onClick={() => removeTodo(todo.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

### 3. Updating Items

```jsx
function TodoList() {
  const [todos, setTodos] = useState([...]);
  
  const toggleTodo = (id) => {
    setTodos(
      todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  };
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

### 4. Grouped Lists

```jsx
function GroupedList({ items }) {
  const grouped = items.reduce((acc, item) => {
    const group = item.category;
    if (!acc[group]) acc[group] = [];
    acc[group].push(item);
    return acc;
  }, {});
  
  return (
    <div>
      {Object.entries(grouped).map(([category, items]) => (
        <div key={category}>
          <h2>{category}</h2>
          <ul>
            {items.map(item => (
              <li key={item.id}>{item.name}</li>
            ))}
          </ul>
        </div>
      ))}
    </div>
  );
}
```

## Debugging Key Issues

### React DevTools

React warns about key issues in console:

```text
Warning: Each child in a list should have a unique "key" prop.
```

```text
Warning: Encountered two children with the same key, `1`.
Keys should be unique so that components maintain their identity across updates.
```

### Finding Key Issues

```jsx
// Add this to help debug
function DebugList({ items }) {
  useEffect(() => {
    const keys = items.map(item => item.id);
    const unique = new Set(keys);
    
    if (keys.length !== unique.size) {
      console.warn('Duplicate keys detected!', keys);
    }
  }, [items]);
  
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.text}</li>
      ))}
    </ul>
  );
}
```

## Summary

- Keys help React identify which items changed, added, or removed
- Use stable, unique IDs as keys whenever possible
- Never use random values or values generated in render as keys
- Use index as key only for static lists that never change
- Keys must be unique among siblings, not globally
- Keys are not passed as props to components
- Use React.Fragment when mapping items that need multiple elements
- Proper keys are crucial for performance and correctness
- Index as key can cause state loss, wrong updates, and animation issues
- When items don't have IDs, generate stable IDs when creating items

Understanding keys is essential for building correct, performant React applications. Always use stable IDs from your data, and only fall back to index as a last resort for static lists.
