# Function Components and Props

## Overview

Components are the building blocks of React applications. They let you split the UI into independent, reusable pieces. Function components are JavaScript functions that accept props and return React elements. They're simpler than class components and are now the recommended way to write React components.

## Function Components

### Basic Function Component

```jsx
function Welcome() {
  return <h1>Hello, World!</h1>;
}

// Arrow function syntax
const Welcome = () => {
  return <h1>Hello, World!</h1>;
};

// Implicit return (one-line)
const Welcome = () => <h1>Hello, World!</h1>;
```

### Component Naming

**Convention: PascalCase**

```jsx
// Correct
function UserProfile() { }
function TodoList() { }
function API_Client() { } // Acceptable for acronyms

// Wrong - lowercase components are treated as HTML tags
function userProfile() { } // React thinks this is <userprofile>
```

### Using Components

```jsx
function App() {
  return (
    <div>
      <Welcome />
      <Welcome />
      <Welcome />
    </div>
  );
}
```

## Props (Properties)

Props are arguments passed to React components. They're read-only and flow down from parent to child (unidirectional data flow).

### Basic Props

```jsx
function Welcome(props) {
  return <h1>Hello, {props.name}!</h1>;
}

// Usage
<Welcome name="Alice" />
<Welcome name="Bob" />
```

### Destructuring Props

Most common pattern in modern React:

```jsx
function Welcome({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Multiple props
function UserCard({ name, email, age }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{email}</p>
      <p>Age: {age}</p>
    </div>
  );
}

// Usage
<UserCard 
  name="Alice" 
  email="alice@example.com" 
  age={30} 
/>
```

### Destructuring with Default Values

```jsx
function Button({ text = 'Click me', variant = 'primary' }) {
  return (
    <button className={`btn btn-${variant}`}>
      {text}
    </button>
  );
}

// Uses defaults
<Button />
// Output: <button class="btn btn-primary">Click me</button>

// Override defaults
<Button text="Submit" variant="success" />
// Output: <button class="btn btn-success">Submit</button>
```

### Props Types

Props can be any JavaScript value:

```jsx
function Component({
  // Primitives
  text,           // string
  count,          // number
  isActive,       // boolean
  value,          // any primitive
  
  // Complex types
  user,           // object
  items,          // array
  onClick,        // function
  children,       // React nodes
  
  // JSX elements
  icon,           // <Icon />
  renderHeader,   // function that returns JSX
}) {
  return <div>...</div>;
}

// Usage
<Component
  text="Hello"
  count={42}
  isActive={true}
  user={{ name: 'Alice', age: 30 }}
  items={[1, 2, 3]}
  onClick={() => console.log('clicked')}
  icon={<Icon name="star" />}
  renderHeader={() => <h1>Title</h1>}
>
  <p>Child content</p>
</Component>
```

### Props Are Read-Only

Props must never be modified:

```jsx
// WRONG - Never modify props
function Component(props) {
  props.name = 'Modified'; // Error!
  return <div>{props.name}</div>;
}

// WRONG - Don't mutate object/array props
function Component({ user }) {
  user.name = 'Modified'; // Mutates parent's data!
  return <div>{user.name}</div>;
}

// CORRECT - Create new values
function Component({ user }) {
  const modifiedUser = { ...user, name: 'Modified' };
  return <div>{modifiedUser.name}</div>;
}
```

**Why are props read-only?**
- Ensures predictable data flow
- Makes debugging easier
- Prevents accidental side effects
- Enables React's optimization strategies

## The children Prop

The `children` prop contains content between component tags:

```jsx
function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  );
}

// Usage
<Card>
  <h2>Title</h2>
  <p>Content here</p>
</Card>
```

### Children Types

```jsx
function Container({ children }) {
  return <div>{children}</div>;
}

// String
<Container>Hello</Container>

// Number
<Container>{42}</Container>

// Element
<Container>
  <h1>Title</h1>
</Container>

// Multiple elements
<Container>
  <h1>Title</h1>
  <p>Content</p>
</Container>

// Expression
<Container>
  {user.isLoggedIn ? <Dashboard /> : <Login />}
</Container>

// Array
<Container>
  {items.map(item => <Item key={item.id} {...item} />)}
</Container>
```

### Manipulating Children

React provides utilities for working with children:

```jsx
import { Children, cloneElement } from 'react';

function List({ children }) {
  // Count children
  const count = Children.count(children);
  
  // Map over children
  const items = Children.map(children, (child, index) => {
    // Clone and add props
    return cloneElement(child, {
      key: index,
      index: index
    });
  });
  
  return <ul>{items}</ul>;
}

// Usage
<List>
  <li>Item 1</li>
  <li>Item 2</li>
  <li>Item 3</li>
</List>
```

### Render Props via Children

```jsx
function MouseTracker({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  // Children as function
  return children(position);
}

// Usage
<MouseTracker>
  {({ x, y }) => (
    <div>Mouse position: {x}, {y}</div>
  )}
</MouseTracker>
```

## Props Patterns

### 1. Rest Props

Collect remaining props with rest syntax:

```jsx
function Button({ variant, children, ...rest }) {
  return (
    <button className={`btn btn-${variant}`} {...rest}>
      {children}
    </button>
  );
}

// All additional props are passed to button
<Button 
  variant="primary" 
  onClick={handleClick}
  disabled
  aria-label="Submit form"
>
  Submit
</Button>
```

### 2. Prop Spreading

```jsx
const buttonProps = {
  onClick: handleClick,
  disabled: true,
  'aria-label': 'Submit'
};

// Spread props
<button {...buttonProps}>Submit</button>

// Override specific props
<button {...buttonProps} disabled={false}>
  Submit
</button>
```

### 3. Conditional Props

```jsx
function Input({ error, ...props }) {
  return (
    <div>
      <input
        className={error ? 'input-error' : 'input'}
        {...props}
      />
      {error && <span className="error">{error}</span>}
    </div>
  );
}
```

### 4. Props with Validation

While TypeScript provides compile-time validation, you can also use PropTypes for runtime validation:

```jsx
import PropTypes from 'prop-types';

function UserCard({ name, age, email, onEdit }) {
  return <div>...</div>;
}

UserCard.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
  email: PropTypes.string.isRequired,
  onEdit: PropTypes.func
};

UserCard.defaultProps = {
  age: 0,
  onEdit: () => {}
};
```

**Note:** TypeScript is preferred over PropTypes in modern React:

```typescript
interface UserCardProps {
  name: string;
  age?: number;
  email: string;
  onEdit?: () => void;
}

function UserCard({ name, age = 0, email, onEdit }: UserCardProps) {
  return <div>...</div>;
}
```

## Component Composition

### Basic Composition

```jsx
function Avatar({ src, alt }) {
  return <img src={src} alt={alt} className="avatar" />;
}

function UserInfo({ name, email }) {
  return (
    <div>
      <h3>{name}</h3>
      <p>{email}</p>
    </div>
  );
}

function UserCard({ user }) {
  return (
    <div className="user-card">
      <Avatar src={user.avatar} alt={user.name} />
      <UserInfo name={user.name} email={user.email} />
    </div>
  );
}
```

### Specialized Components

```jsx
function Dialog({ title, children }) {
  return (
    <div className="dialog">
      <h2>{title}</h2>
      <div className="content">{children}</div>
    </div>
  );
}

// Specialized versions
function ConfirmDialog({ children }) {
  return (
    <Dialog title="Confirm Action">
      {children}
      <div className="actions">
        <button>Cancel</button>
        <button>Confirm</button>
      </div>
    </Dialog>
  );
}

function WelcomeDialog() {
  return (
    <Dialog title="Welcome">
      <p>Thank you for joining!</p>
    </Dialog>
  );
}
```

### Container and Presentation Pattern

```jsx
// Presentational component (pure UI)
function UserList({ users, onUserClick }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id} onClick={() => onUserClick(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}

// Container component (logic and state)
function UserListContainer() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchUsers().then(data => {
      setUsers(data);
      setLoading(false);
    });
  }, []);
  
  const handleUserClick = (user) => {
    console.log('Clicked:', user);
  };
  
  if (loading) return <div>Loading...</div>;
  
  return <UserList users={users} onUserClick={handleUserClick} />;
}
```

## Common Patterns

### 1. Computed Values from Props

```jsx
function PriceDisplay({ price, taxRate }) {
  // Compute derived values
  const tax = price * taxRate;
  const total = price + tax;
  
  return (
    <div>
      <div>Subtotal: ${price.toFixed(2)}</div>
      <div>Tax: ${tax.toFixed(2)}</div>
      <div>Total: ${total.toFixed(2)}</div>
    </div>
  );
}
```

### 2. Conditional Rendering Based on Props

```jsx
function Message({ type, text }) {
  const styles = {
    success: 'bg-green text-white',
    error: 'bg-red text-white',
    warning: 'bg-yellow text-black',
    info: 'bg-blue text-white'
  };
  
  const icons = {
    success: '✓',
    error: '✗',
    warning: '⚠',
    info: 'ℹ'
  };
  
  return (
    <div className={styles[type]}>
      <span>{icons[type]}</span>
      <span>{text}</span>
    </div>
  );
}
```

### 3. Props as Configuration

```jsx
function DataTable({ 
  data,
  columns,
  sortable = false,
  filterable = false,
  paginated = false,
  pageSize = 10 
}) {
  // Implementation based on configuration
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => (
            <th key={col.key}>
              {col.label}
              {sortable && <SortIcon />}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id}>
            {columns.map(col => (
              <td key={col.key}>{row[col.key]}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

### 4. Callback Props

```jsx
function SearchBox({ onSearch, onClear, placeholder }) {
  const [value, setValue] = useState('');
  
  const handleSubmit = (e) => {
    e.preventDefault();
    onSearch(value);
  };
  
  const handleClear = () => {
    setValue('');
    onClear?.(); // Optional callback
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder={placeholder}
      />
      <button type="submit">Search</button>
      {value && <button type="button" onClick={handleClear}>Clear</button>}
    </form>
  );
}

// Usage
<SearchBox
  onSearch={(query) => console.log('Searching:', query)}
  onClear={() => console.log('Cleared')}
  placeholder="Search users..."
/>
```

## Advanced Patterns

### 1. Render Props

```jsx
function WithData({ url, render }) {
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
  
  return render({ data, loading });
}

// Usage
<WithData
  url="/api/users"
  render={({ data, loading }) => (
    loading ? <div>Loading...</div> : <UserList users={data} />
  )}
/>
```

### 2. Component as Prop

```jsx
function Layout({ Header, Sidebar, Content, Footer }) {
  return (
    <div className="layout">
      <header>{Header}</header>
      <div className="main">
        <aside>{Sidebar}</aside>
        <main>{Content}</main>
      </div>
      <footer>{Footer}</footer>
    </div>
  );
}

// Usage
<Layout
  Header={<AppHeader user={currentUser} />}
  Sidebar={<Navigation items={navItems} />}
  Content={<Dashboard />}
  Footer={<AppFooter />}
/>
```

### 3. Props Getter Pattern

```jsx
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  
  const getTogglerProps = (props = {}) => ({
    ...props,
    onClick: () => {
      setValue(v => !v);
      props.onClick?.();
    }
  });
  
  return { value, getTogglerProps };
}

function Component() {
  const { value, getTogglerProps } = useToggle(false);
  
  return (
    <div>
      <button {...getTogglerProps()}>Toggle</button>
      {value && <div>Visible content</div>}
    </div>
  );
}
```

## Performance Considerations

### 1. Avoid Creating Objects in Render

```jsx
// Bad - Creates new object every render
function Component({ onClick }) {
  return <Child data={{ name: 'Alice' }} onClick={() => onClick()} />;
}

// Good - Stable references
function Component({ onClick }) {
  const data = useMemo(() => ({ name: 'Alice' }), []);
  const handleClick = useCallback(() => onClick(), [onClick]);
  
  return <Child data={data} onClick={handleClick} />;
}

// Or move outside component if static
const DATA = { name: 'Alice' };

function Component({ onClick }) {
  return <Child data={DATA} onClick={onClick} />;
}
```

### 2. Props Destructuring and Memoization

```jsx
// If Child is memoized, only re-renders when name changes
const Child = memo(function Child({ name, age }) {
  return <div>{name}, {age}</div>;
});

function Parent({ user }) {
  // Destructure to pass only needed props
  return <Child name={user.name} age={user.age} />;
}
```

## Common Mistakes

### 1. Mutating Props

```jsx
// Wrong
function Component({ items }) {
  items.push('new item'); // Mutates parent's data!
  return <List items={items} />;
}

// Correct
function Component({ items }) {
  const newItems = [...items, 'new item'];
  return <List items={newItems} />;
}
```

### 2. Using Props Directly in State

```jsx
// Anti-pattern: Props in state
function Component({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  // If initialCount prop changes, state doesn't update!
  
  return <div>{count}</div>;
}

// Better: Use the prop directly or useEffect
function Component({ count }) {
  return <div>{count}</div>;
}

// Or if you need to initialize and then diverge
function Component({ initialCount }) {
  const [count, setCount] = useState(initialCount);
  
  useEffect(() => {
    setCount(initialCount);
  }, [initialCount]);
  
  return <div>{count}</div>;
}
```

### 3. Forgetting Key Prop

```jsx
// Wrong - Missing keys
items.map(item => <Item data={item} />)

// Correct
items.map(item => <Item key={item.id} data={item} />)
```

### 4. Passing Functions Incorrectly

```jsx
// Wrong - Calls function immediately
<button onClick={handleClick()}>Click</button>

// Correct - Passes function reference
<button onClick={handleClick}>Click</button>

// Correct - Passes arrow function
<button onClick={() => handleClick(arg)}>Click</button>
```

## Best Practices

### 1. Naming Props

```jsx
// Good names
<Button onClick={handleClick} disabled={isDisabled} variant="primary" />
<UserCard user={currentUser} onEdit={handleEdit} showAvatar />

// Bad names
<Button click={handleClick} dis={isDisabled} type="primary" />
<UserCard data={currentUser} edit={handleEdit} avatar />
```

### 2. Boolean Props

```jsx
// Presence means true
<Button disabled />
<Input required autoFocus />

// Explicit for clarity
<Button disabled={isLoading} />
<Input required={isRequired} />

// Prefix with "is", "has", "should", etc.
<Modal isOpen={isModalOpen} />
<List hasMore={hasMoreItems} />
<Form shouldValidate={shouldValidate} />
```

### 3. Keep Components Focused

```jsx
// Too many responsibilities
function UserDashboard({ user }) {
  return (
    <div>
      <Avatar src={user.avatar} />
      <UserInfo name={user.name} email={user.email} />
      <ActivityFeed items={user.activities} />
      <Recommendations items={user.recommendations} />
      <Settings config={user.settings} />
    </div>
  );
}

// Better: Break into smaller components
function UserDashboard({ user }) {
  return (
    <div>
      <UserHeader user={user} />
      <UserContent user={user} />
      <UserSidebar user={user} />
    </div>
  );
}
```

### 4. Document Complex Props

```jsx
/**
 * Displays a user profile card with avatar and information
 * 
 * @param {Object} user - User object
 * @param {string} user.name - Full name
 * @param {string} user.email - Email address
 * @param {string} user.avatar - Avatar URL
 * @param {Function} onEdit - Called when edit button is clicked
 * @param {boolean} showActions - Whether to show action buttons
 */
function UserCard({ user, onEdit, showActions = true }) {
  return <div>...</div>;
}
```

## Summary

- Function components are JavaScript functions that return JSX
- Props are read-only data passed from parent to child
- Use destructuring for cleaner prop access
- The children prop contains content between component tags
- Components should be pure functions of their props
- Composition is preferred over inheritance
- Use TypeScript or PropTypes for prop validation
- Avoid creating objects/functions in render for better performance
- Keep components focused and props well-named
- Props flow down (unidirectional data flow) for predictable behavior

Understanding components and props is fundamental to React. Master these concepts and you'll be able to build complex, maintainable applications through composition.
