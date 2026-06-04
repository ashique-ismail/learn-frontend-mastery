# Component Composition

## Overview

Component Composition is a fundamental React pattern for building complex UIs from smaller, reusable pieces. Instead of using inheritance or monolithic components, React favors composition - combining simple components to create more complex functionality. This approach leads to more maintainable, testable, and flexible code.

Composition is at the heart of React's design philosophy: "Composition is how you write good React code."

## Core Concepts

### What is Component Composition?

Component composition involves:

1. **Building small, focused components** that do one thing well
2. **Combining components** to create more complex functionality
3. **Passing components as props** (including children)
4. **Creating flexible interfaces** through composition rather than configuration

### Composition vs Inheritance

```javascript
// Bad: Inheritance (not idiomatic React)
class BaseButton extends React.Component {
  handleClick() {
    console.log('clicked');
  }
}

class PrimaryButton extends BaseButton {
  render() {
    return <button onClick={this.handleClick}>Click me</button>;
  }
}

// Good: Composition
function Button({ onClick, children, variant = 'default' }) {
  const styles = {
    default: 'btn-default',
    primary: 'btn-primary',
    danger: 'btn-danger',
  };

  return (
    <button onClick={onClick} className={styles[variant]}>
      {children}
    </button>
  );
}

// Usage
<Button variant="primary" onClick={() => console.log('clicked')}>
  Click me
</Button>
```

## Composition Patterns

### 1. Children Prop

The most basic form of composition uses the children prop:

```javascript
function Card({ children }) {
  return <div className="card">{children}</div>;
}

function App() {
  return (
    <Card>
      <h2>Card Title</h2>
      <p>Card content goes here</p>
    </Card>
  );
}
```

### 2. Slot Pattern (Multiple Children)

```javascript
function Layout({ header, sidebar, content, footer }) {
  return (
    <div className="layout">
      <header>{header}</header>
      <div className="main">
        <aside>{sidebar}</aside>
        <main>{content}</main>
      </div>
      <footer>{footer}</footer>
    </div>
  );
}

// Usage
function App() {
  return (
    <Layout
      header={<Header />}
      sidebar={<Sidebar />}
      content={<MainContent />}
      footer={<Footer />}
    />
  );
}
```

### 3. Render Props

```javascript
function MouseTracker({ render }) {
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
    <MouseTracker
      render={({ x, y }) => (
        <div>
          Mouse position: {x}, {y}
        </div>
      )}
    />
  );
}

// Or using children as function
function MouseTracker({ children }) {
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
<MouseTracker>
  {({ x, y }) => (
    <div>
      Mouse position: {x}, {y}
    </div>
  )}
</MouseTracker>
```

### 4. Compound Components

```javascript
const TabsContext = createContext();

function Tabs({ children, defaultActive = 0 }) {
  const [activeIndex, setActiveIndex] = useState(defaultActive);

  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div className="tab-list">{children}</div>;
}

function Tab({ index, children }) {
  const { activeIndex, setActiveIndex } = useContext(TabsContext);
  const isActive = index === activeIndex;

  return (
    <button
      className={`tab ${isActive ? 'active' : ''}`}
      onClick={() => setActiveIndex(index)}
    >
      {children}
    </button>
  );
}

function TabPanels({ children }) {
  const { activeIndex } = useContext(TabsContext);
  return <div className="tab-panels">{children[activeIndex]}</div>;
}

function TabPanel({ children }) {
  return <div className="tab-panel">{children}</div>;
}

// Export as compound component
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// Usage
function App() {
  return (
    <Tabs defaultActive={0}>
      <Tabs.List>
        <Tabs.Tab index={0}>Tab 1</Tabs.Tab>
        <Tabs.Tab index={1}>Tab 2</Tabs.Tab>
        <Tabs.Tab index={2}>Tab 3</Tabs.Tab>
      </Tabs.List>
      <Tabs.Panels>
        <Tabs.Panel>Content 1</Tabs.Panel>
        <Tabs.Panel>Content 2</Tabs.Panel>
        <Tabs.Panel>Content 3</Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

## Advanced Composition Patterns

### Container/Presentational Pattern

```javascript
// Presentational Component (Pure UI)
function UserList({ users, onUserClick, loading, error }) {
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id} onClick={() => onUserClick(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}

// Container Component (Logic)
function UserListContainer() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/users')
      .then((res) => res.json())
      .then((data) => {
        setUsers(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  const handleUserClick = (user) => {
    console.log('Clicked:', user);
  };

  return (
    <UserList
      users={users}
      loading={loading}
      error={error}
      onUserClick={handleUserClick}
    />
  );
}
```

### HOC Composition

```javascript
// Higher Order Components
function withLoading(Component) {
  return function WithLoadingComponent({ loading, ...props }) {
    if (loading) return <div>Loading...</div>;
    return <Component {...props} />;
  };
}

function withError(Component) {
  return function WithErrorComponent({ error, ...props }) {
    if (error) return <div>Error: {error}</div>;
    return <Component {...props} />;
  };
}

// Base component
function UserList({ users, onUserClick }) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id} onClick={() => onUserClick(user)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}

// Compose HOCs
const EnhancedUserList = withError(withLoading(UserList));

// Usage
function App() {
  const { users, loading, error } = useUsers();

  return (
    <EnhancedUserList
      users={users}
      loading={loading}
      error={error}
      onUserClick={(user) => console.log(user)}
    />
  );
}
```

### Custom Hooks Composition

```javascript
// Composable hooks
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

function useSearch() {
  const [query, setQuery] = useLocalStorage('searchQuery', '');
  const debouncedQuery = useDebounce(query, 500);
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`)
        .then((res) => res.json())
        .then(setResults);
    }
  }, [debouncedQuery]);

  return { query, setQuery, results };
}

// Usage
function SearchComponent() {
  const { query, setQuery, results } = useSearch();

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {results.map((result) => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Real-World Examples

### Modal Composition

```javascript
function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

function ModalHeader({ children, onClose }) {
  return (
    <div className="modal-header">
      {children}
      <button onClick={onClose} className="close-btn">
        ×
      </button>
    </div>
  );
}

function ModalBody({ children }) {
  return <div className="modal-body">{children}</div>;
}

function ModalFooter({ children }) {
  return <div className="modal-footer">{children}</div>;
}

// Export as compound component
Modal.Header = ModalHeader;
Modal.Body = ModalBody;
Modal.Footer = ModalFooter;

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)}>
        <Modal.Header onClose={() => setIsOpen(false)}>
          <h2>Confirm Action</h2>
        </Modal.Header>
        <Modal.Body>
          <p>Are you sure you want to proceed?</p>
        </Modal.Body>
        <Modal.Footer>
          <button onClick={() => setIsOpen(false)}>Cancel</button>
          <button onClick={() => setIsOpen(false)}>Confirm</button>
        </Modal.Footer>
      </Modal>
    </>
  );
}
```

### Form Composition

```javascript
const FormContext = createContext();

function Form({ onSubmit, children, initialValues = {} }) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});

  const setValue = (name, value) => {
    setValues((prev) => ({ ...prev, [name]: value }));
    // Clear error when value changes
    if (errors[name]) {
      setErrors((prev) => {
        const newErrors = { ...prev };
        delete newErrors[name];
        return newErrors;
      });
    }
  };

  const setError = (name, error) => {
    setErrors((prev) => ({ ...prev, [name]: error }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit(values, { setError });
  };

  return (
    <FormContext.Provider
      value={{ values, errors, setValue, setError }}
    >
      <form onSubmit={handleSubmit}>{children}</form>
    </FormContext.Provider>
  );
}

function FormField({ name, label, type = 'text', validate }) {
  const { values, errors, setValue } = useContext(FormContext);
  const value = values[name] || '';
  const error = errors[name];

  const handleChange = (e) => {
    setValue(name, e.target.value);
  };

  const handleBlur = () => {
    if (validate) {
      const error = validate(value);
      if (error) {
        setValue(name, value); // Trigger error display
      }
    }
  };

  return (
    <div className="form-field">
      <label htmlFor={name}>{label}</label>
      <input
        id={name}
        name={name}
        type={type}
        value={value}
        onChange={handleChange}
        onBlur={handleBlur}
      />
      {error && <span className="error">{error}</span>}
    </div>
  );
}

function SubmitButton({ children }) {
  return <button type="submit">{children}</button>;
}

Form.Field = FormField;
Form.Submit = SubmitButton;

// Usage
function LoginForm() {
  const handleSubmit = (values, { setError }) => {
    if (!values.email) {
      setError('email', 'Email is required');
      return;
    }
    if (!values.password) {
      setError('password', 'Password is required');
      return;
    }
    console.log('Submitting:', values);
  };

  return (
    <Form onSubmit={handleSubmit}>
      <Form.Field
        name="email"
        label="Email"
        type="email"
        validate={(value) =>
          !value ? 'Required' : !value.includes('@') ? 'Invalid email' : null
        }
      />
      <Form.Field
        name="password"
        label="Password"
        type="password"
        validate={(value) =>
          !value
            ? 'Required'
            : value.length < 8
            ? 'Min 8 characters'
            : null
        }
      />
      <Form.Submit>Login</Form.Submit>
    </Form>
  );
}
```

### List with Composition

```javascript
function List({ items, renderItem, emptyMessage = 'No items' }) {
  if (items.length === 0) {
    return <div className="empty-state">{emptyMessage}</div>;
  }

  return (
    <ul className="list">
      {items.map((item, index) => (
        <li key={item.id || index}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

function VirtualList({ items, renderItem, height = 400, itemHeight = 50 }) {
  const [scrollTop, setScrollTop] = useState(0);
  
  const startIndex = Math.floor(scrollTop / itemHeight);
  const endIndex = Math.ceil((scrollTop + height) / itemHeight);
  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <div
      style={{ height, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.target.scrollTop)}
    >
      <div style={{ height: items.length * itemHeight, position: 'relative' }}>
        {visibleItems.map((item, index) => (
          <div
            key={item.id || startIndex + index}
            style={{
              position: 'absolute',
              top: (startIndex + index) * itemHeight,
              left: 0,
              right: 0,
              height: itemHeight,
            }}
          >
            {renderItem(item, startIndex + index)}
          </div>
        ))}
      </div>
    </div>
  );
}

// Usage
function App() {
  const users = [
    { id: 1, name: 'John', email: 'john@example.com' },
    { id: 2, name: 'Jane', email: 'jane@example.com' },
  ];

  return (
    <>
      <List
        items={users}
        renderItem={(user) => (
          <div>
            <strong>{user.name}</strong>
            <br />
            <small>{user.email}</small>
          </div>
        )}
        emptyMessage="No users found"
      />

      <VirtualList
        items={Array.from({ length: 10000 }, (_, i) => ({
          id: i,
          name: `User ${i}`,
        }))}
        renderItem={(user) => <div>{user.name}</div>}
        height={400}
        itemHeight={50}
      />
    </>
  );
}
```

## Best Practices

### 1. Single Responsibility

```javascript
// Bad: Component does too much
function UserDashboard() {
  const [users, setUsers] = useState([]);
  const [selectedUser, setSelectedUser] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    // Fetch users
  }, []);

  return (
    <div>
      {/* User list */}
      {/* User details */}
      {/* Edit form */}
    </div>
  );
}

// Good: Composed from smaller components
function UserDashboard() {
  const [selectedUser, setSelectedUser] = useState(null);

  return (
    <div>
      <UserList onSelect={setSelectedUser} />
      {selectedUser && <UserDetails user={selectedUser} />}
      {selectedUser && <UserEditForm user={selectedUser} />}
    </div>
  );
}
```

### 2. Flexible Children

```javascript
// Good: Accept any children
function Container({ children }) {
  return <div className="container">{children}</div>;
}

// Good: Accept render prop for flexibility
function DataFetcher({ url, children }) {
  const { data, loading, error } = useFetch(url);
  return children({ data, loading, error });
}
```

### 3. Composition Over Props

```javascript
// Bad: Too many props
function Button({
  text,
  icon,
  iconPosition,
  badge,
  badgeColor,
  tooltip,
  onClick,
}) {
  return <button onClick={onClick}>{/* Complex logic */}</button>;
}

// Good: Composition
function Button({ children, onClick }) {
  return <button onClick={onClick}>{children}</button>;
}

// Usage with composition
<Button onClick={handleClick}>
  <Icon name="save" />
  Save
  <Badge color="red">3</Badge>
</Button>
```

## Common Mistakes

### 1. Prop Drilling

```javascript
// Bad: Passing props through many levels
function App() {
  const [user, setUser] = useState(null);
  return <Layout user={user} setUser={setUser} />;
}

function Layout({ user, setUser }) {
  return <Sidebar user={user} setUser={setUser} />;
}

function Sidebar({ user, setUser }) {
  return <UserMenu user={user} setUser={setUser} />;
}

// Good: Use Context
const UserContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserMenu() {
  const { user, setUser } = useContext(UserContext);
  // Use directly
}
```

### 2. Over-composition

```javascript
// Bad: Too many tiny components
function UserCard({ user }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>
          <CardTitleText>{user.name}</CardTitleText>
        </CardTitle>
      </CardHeader>
      <CardBody>
        <CardBodyText>{user.email}</CardBodyText>
      </CardBody>
    </Card>
  );
}

// Good: Balance composition with simplicity
function UserCard({ user }) {
  return (
    <Card>
      <Card.Header>
        <h3>{user.name}</h3>
      </Card.Header>
      <Card.Body>
        <p>{user.email}</p>
      </Card.Body>
    </Card>
  );
}
```

## Testing Composed Components

```javascript
import { render, screen } from '@testing-library/react';

describe('Modal Composition', () => {
  test('renders composed modal', () => {
    render(
      <Modal isOpen={true} onClose={() => {}}>
        <Modal.Header onClose={() => {}}>
          <h2>Title</h2>
        </Modal.Header>
        <Modal.Body>Content</Modal.Body>
        <Modal.Footer>Footer</Modal.Footer>
      </Modal>
    );

    expect(screen.getByText('Title')).toBeInTheDocument();
    expect(screen.getByText('Content')).toBeInTheDocument();
    expect(screen.getByText('Footer')).toBeInTheDocument();
  });

  test('composition works without optional parts', () => {
    render(
      <Modal isOpen={true} onClose={() => {}}>
        <Modal.Body>Just body content</Modal.Body>
      </Modal>
    );

    expect(screen.getByText('Just body content')).toBeInTheDocument();
  });
});
```

## Key Takeaways

1. **Prefer Composition**: Build complex UIs from simple, reusable components
2. **Single Responsibility**: Each component should do one thing well
3. **Flexible APIs**: Use children, render props, and compound components for flexibility
4. **Context for Shared State**: Avoid prop drilling with Context API
5. **Custom Hooks**: Compose logic with custom hooks
6. **Test Components in Isolation**: Test composed components independently
7. **Balance**: Don't over-compose - find the right level of granularity

## Additional Resources

- [React Composition vs Inheritance](https://react.dev/learn/composition-vs-inheritance)
- [Compound Components Pattern](https://kentcdodds.com/blog/compound-components-with-react-hooks)
- [Render Props](https://react.dev/reference/react/cloneElement#passing-data-with-a-render-prop)
- [Advanced React Patterns](https://javascript.plainenglish.io/5-advanced-react-patterns-a6b7624267a6)
