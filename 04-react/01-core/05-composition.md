# Composition: Children, Render Props, and Component Slots

## Overview

Composition is one of React's most powerful features. Instead of using inheritance to share code between components, React uses composition - building complex UIs by combining simpler components. This document covers three key composition patterns: children, render props, and component slots.

## Children Composition

### Basic Children

The `children` prop is the foundation of React composition:

```jsx
function Card({ children }) {
  return (
    <div className="card">
      <div className="card-body">
        {children}
      </div>
    </div>
  );
}

// Usage
<Card>
  <h2>Title</h2>
  <p>Content goes here</p>
</Card>
```

### Why Children?

**Benefits:**
- Flexible and reusable components
- Clean, declarative syntax
- Natural HTML-like nesting
- Easy to understand and maintain

```jsx
// Without children - rigid
function Dialog({ title, content, actions }) {
  return (
    <div className="dialog">
      <h2>{title}</h2>
      <div>{content}</div>
      <div>{actions}</div>
    </div>
  );
}

// With children - flexible
function Dialog({ children }) {
  return <div className="dialog">{children}</div>;
}

// Can put anything inside
<Dialog>
  <CustomHeader />
  <ComplexContent />
  <DynamicActions />
</Dialog>
```

### Children Patterns

#### 1. Wrapper Components

```jsx
function Container({ children, maxWidth = '1200px' }) {
  return (
    <div style={{ maxWidth, margin: '0 auto', padding: '20px' }}>
      {children}
    </div>
  );
}

function Page() {
  return (
    <Container maxWidth="800px">
      <Header />
      <Content />
      <Footer />
    </Container>
  );
}
```

#### 2. Layout Components

```jsx
function TwoColumnLayout({ children }) {
  return (
    <div className="two-column">
      {children}
    </div>
  );
}

// Usage with any content
<TwoColumnLayout>
  <Sidebar />
  <MainContent />
</TwoColumnLayout>
```

#### 3. Provider Pattern

```jsx
function ThemeProvider({ children, theme }) {
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

// Wraps entire app
<ThemeProvider theme={darkTheme}>
  <App />
</ThemeProvider>
```

### Multiple Children Slots

Sometimes you need more control than a single `children` prop:

```jsx
// Named slots via props
function Layout({ header, sidebar, content, footer }) {
  return (
    <div className="layout">
      <header className="header">{header}</header>
      <div className="main">
        <aside className="sidebar">{sidebar}</aside>
        <main className="content">{content}</main>
      </div>
      <footer className="footer">{footer}</footer>
    </div>
  );
}

// Usage
<Layout
  header={<AppHeader />}
  sidebar={<Navigation />}
  content={<Dashboard />}
  footer={<AppFooter />}
/>
```

### Children as Functions

Also known as "function as children" or "render prop via children":

```jsx
function MouseTracker({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);
  
  // Call children as a function
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

### Manipulating Children

React provides utilities to work with children:

```jsx
import { Children, cloneElement, isValidElement } from 'react';

function List({ children }) {
  return (
    <ul>
      {Children.map(children, (child, index) => {
        // Add props to each child
        if (isValidElement(child)) {
          return cloneElement(child, {
            key: child.key || index,
            index
          });
        }
        return child;
      })}
    </ul>
  );
}

// Usage
<List>
  <ListItem>First</ListItem>
  <ListItem>Second</ListItem>
  <ListItem>Third</ListItem>
</List>
```

#### Children Utilities

```jsx
// Children.count - Count number of children
const count = Children.count(children);

// Children.only - Ensure single child
function SingleChild({ children }) {
  return Children.only(children);
}

// Children.toArray - Convert to flat array
const childArray = Children.toArray(children);

// Children.forEach - Iterate without returning
Children.forEach(children, (child, index) => {
  console.log(child, index);
});
```

### Restricting Children Types

```jsx
function Tabs({ children }) {
  // Only accept Tab components
  const tabs = Children.toArray(children).filter(
    child => isValidElement(child) && child.type === Tab
  );
  
  if (tabs.length === 0) {
    console.warn('Tabs component requires at least one Tab child');
  }
  
  return <div className="tabs">{tabs}</div>;
}

// Usage
<Tabs>
  <Tab label="First">Content 1</Tab>
  <Tab label="Second">Content 2</Tab>
</Tabs>
```

## Render Props

Render props is a pattern where a component accepts a function prop that returns React elements.

### Basic Render Prop

```jsx
function DataFetcher({ url, render }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
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
  
  return render({ data, loading, error });
}

// Usage
<DataFetcher
  url="/api/users"
  render={({ data, loading, error }) => {
    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;
    return <UserList users={data} />;
  }}
/>
```

### Why Render Props?

**Advantages:**
- Share stateful logic without wrapping components
- More flexible than HOCs
- Clear what data is being passed
- Can use multiple in same component

```jsx
// Multiple render props in one component
function Dashboard() {
  return (
    <div>
      <DataFetcher
        url="/api/users"
        render={({ data }) => <UserStats users={data} />}
      />
      
      <DataFetcher
        url="/api/posts"
        render={({ data }) => <RecentPosts posts={data} />}
      />
    </div>
  );
}
```

### Render Prop Variations

#### 1. Named Render Props

```jsx
function Dropdown({ renderTrigger, renderContent }) {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <div onClick={() => setIsOpen(!isOpen)}>
        {renderTrigger({ isOpen })}
      </div>
      {isOpen && (
        <div className="dropdown-content">
          {renderContent({ close: () => setIsOpen(false) })}
        </div>
      )}
    </div>
  );
}

// Usage
<Dropdown
  renderTrigger={({ isOpen }) => (
    <button>Menu {isOpen ? '▲' : '▼'}</button>
  )}
  renderContent={({ close }) => (
    <ul>
      <li onClick={close}>Item 1</li>
      <li onClick={close}>Item 2</li>
    </ul>
  )}
/>
```

#### 2. Children as Function (Render Prop)

```jsx
function Toggle({ children }) {
  const [isOn, setIsOn] = useState(false);
  
  return children({
    isOn,
    toggle: () => setIsOn(on => !on),
    setOn: () => setIsOn(true),
    setOff: () => setIsOn(false)
  });
}

// Usage
<Toggle>
  {({ isOn, toggle }) => (
    <div>
      <button onClick={toggle}>
        {isOn ? 'ON' : 'OFF'}
      </button>
      {isOn && <div>Content is visible</div>}
    </div>
  )}
</Toggle>
```

#### 3. Component Prop Pattern

```jsx
function List({ items, renderItem, renderEmpty }) {
  if (items.length === 0) {
    return renderEmpty ? renderEmpty() : <div>No items</div>;
  }
  
  return (
    <ul>
      {items.map((item, index) => (
        <li key={item.id}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// Usage
<List
  items={users}
  renderItem={(user, index) => (
    <div>
      {index + 1}. {user.name}
    </div>
  )}
  renderEmpty={() => <div>No users found</div>}
/>
```

### Render Props vs Hooks

Hooks often replace render props in modern React:

```jsx
// Old: Render prop
<MouseTracker>
  {({ x, y }) => <div>Position: {x}, {y}</div>}
</MouseTracker>

// Modern: Custom hook
function useMousePosition() {
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

function Component() {
  const { x, y } = useMousePosition();
  return <div>Position: {x}, {y}</div>;
}
```

**When to use render props:**
- Need JSX control in parent
- Visual component logic
- Multiple variations of rendering
- Third-party libraries that use them

**When to use hooks:**
- Share stateful logic
- Cleaner syntax
- Better composition
- Most modern cases

## Component Slots

Component slots provide named placeholders for content:

### Basic Slots

```jsx
function Modal({ title, content, actions, onClose }) {
  return (
    <div className="modal">
      <div className="modal-header">
        {title}
        <button onClick={onClose}>×</button>
      </div>
      <div className="modal-content">
        {content}
      </div>
      <div className="modal-actions">
        {actions}
      </div>
    </div>
  );
}

// Usage
<Modal
  title={<h2>Confirm Delete</h2>}
  content={<p>Are you sure you want to delete this item?</p>}
  actions={
    <>
      <button onClick={handleCancel}>Cancel</button>
      <button onClick={handleConfirm}>Delete</button>
    </>
  }
  onClose={handleClose}
/>
```

### Advanced Slot Pattern

```jsx
function Card({ header, body, footer, variant = 'default' }) {
  return (
    <div className={`card card-${variant}`}>
      {header && (
        <div className="card-header">{header}</div>
      )}
      {body && (
        <div className="card-body">{body}</div>
      )}
      {footer && (
        <div className="card-footer">{footer}</div>
      )}
    </div>
  );
}

// Usage with all slots
<Card
  header={<h3>User Profile</h3>}
  body={<UserDetails user={user} />}
  footer={<EditButton />}
  variant="primary"
/>

// Usage with some slots
<Card
  body={<UserDetails user={user} />}
/>
```

### Slot Components Pattern

Create explicit slot components:

```jsx
// Define slot components
function DialogTitle({ children }) {
  return <div className="dialog-title">{children}</div>;
}

function DialogContent({ children }) {
  return <div className="dialog-content">{children}</div>;
}

function DialogActions({ children }) {
  return <div className="dialog-actions">{children}</div>;
}

// Main component that accepts slots
function Dialog({ children }) {
  return <div className="dialog">{children}</div>;
}

// Attach slot components
Dialog.Title = DialogTitle;
Dialog.Content = DialogContent;
Dialog.Actions = DialogActions;

// Usage - very clear structure
<Dialog>
  <Dialog.Title>Confirm Action</Dialog.Title>
  <Dialog.Content>
    <p>Are you sure?</p>
  </Dialog.Content>
  <Dialog.Actions>
    <button>Cancel</button>
    <button>Confirm</button>
  </Dialog.Actions>
</Dialog>
```

### Extracting Slots from Children

```jsx
function Dialog({ children }) {
  const childrenArray = Children.toArray(children);
  
  const title = childrenArray.find(
    child => child.type === Dialog.Title
  );
  const content = childrenArray.find(
    child => child.type === Dialog.Content
  );
  const actions = childrenArray.find(
    child => child.type === Dialog.Actions
  );
  
  return (
    <div className="dialog">
      {title && <div className="dialog-header">{title}</div>}
      {content && <div className="dialog-body">{content}</div>}
      {actions && <div className="dialog-footer">{actions}</div>}
    </div>
  );
}
```

## Composition vs Inheritance

React recommends composition over inheritance:

### Don't Use Inheritance

```jsx
// Bad: Using inheritance
class SpecialButton extends Button {
  render() {
    return <button className="special">{this.props.children}</button>;
  }
}
```

### Use Composition Instead

```jsx
// Good: Using composition
function SpecialButton({ children, ...props }) {
  return (
    <Button {...props} className="special">
      {children}
    </Button>
  );
}
```

### Specialization via Composition

```jsx
// Generic component
function Dialog({ title, children }) {
  return (
    <div className="dialog">
      <h2>{title}</h2>
      <div>{children}</div>
    </div>
  );
}

// Specialized versions
function WelcomeDialog() {
  return (
    <Dialog title="Welcome">
      <p>Thank you for visiting!</p>
    </Dialog>
  );
}

function ConfirmDialog({ message, onConfirm, onCancel }) {
  return (
    <Dialog title="Confirm">
      <p>{message}</p>
      <button onClick={onCancel}>Cancel</button>
      <button onClick={onConfirm}>OK</button>
    </Dialog>
  );
}
```

## Advanced Composition Patterns

### 1. Compound Components

Components that work together:

```jsx
function Tabs({ children, defaultTab = 0 }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  
  return (
    <div className="tabs">
      {Children.map(children, (child, index) => {
        if (child.type === Tab) {
          return cloneElement(child, {
            isActive: index === activeTab,
            onClick: () => setActiveTab(index)
          });
        }
        return child;
      })}
    </div>
  );
}

function Tab({ label, isActive, onClick, children }) {
  return (
    <div>
      <button
        className={isActive ? 'active' : ''}
        onClick={onClick}
      >
        {label}
      </button>
      {isActive && <div className="tab-content">{children}</div>}
    </div>
  );
}

// Usage
<Tabs defaultTab={0}>
  <Tab label="First">Content 1</Tab>
  <Tab label="Second">Content 2</Tab>
  <Tab label="Third">Content 3</Tab>
</Tabs>
```

### 2. Flexible Composition

```jsx
function Page({ header, sidebar, content, footer }) {
  return (
    <div className="page">
      {header}
      <div className="page-body">
        {sidebar}
        {content}
      </div>
      {footer}
    </div>
  );
}

// Can compose in many ways
<Page
  header={<Header />}
  content={<Content />}
/>

<Page
  header={<Header />}
  sidebar={<Sidebar />}
  content={<Content />}
  footer={<Footer />}
/>

<Page content={<FullScreenContent />} />
```

### 3. Recursive Composition

```jsx
function TreeNode({ node, children }) {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div className="tree-node">
      <div onClick={() => setIsOpen(!isOpen)}>
        {node.hasChildren && (isOpen ? '▼' : '▶')}
        {node.label}
      </div>
      {isOpen && (
        <div className="tree-children">
          {children}
        </div>
      )}
    </div>
  );
}

function Tree({ data }) {
  const renderNode = (node) => (
    <TreeNode key={node.id} node={node}>
      {node.children?.map(renderNode)}
    </TreeNode>
  );
  
  return <div className="tree">{data.map(renderNode)}</div>;
}
```

## Best Practices

### 1. Favor Children for Flexibility

```jsx
// Less flexible
function Panel({ title, content, actions }) {
  return (
    <div>
      <h2>{title}</h2>
      <div>{content}</div>
      <div>{actions}</div>
    </div>
  );
}

// More flexible
function Panel({ children }) {
  return <div className="panel">{children}</div>;
}

function PanelHeader({ children }) {
  return <div className="panel-header">{children}</div>;
}

function PanelBody({ children }) {
  return <div className="panel-body">{children}</div>;
}

Panel.Header = PanelHeader;
Panel.Body = PanelBody;
```

### 2. Use Descriptive Prop Names

```jsx
// Unclear
<Component render={(data) => <div>{data}</div>} />

// Clear
<DataFetcher
  url="/api/users"
  renderData={(users) => <UserList users={users} />}
  renderLoading={() => <Spinner />}
  renderError={(error) => <Error message={error} />}
/>
```

### 3. Provide Sensible Defaults

```jsx
function List({ items, renderItem, renderEmpty }) {
  if (items.length === 0) {
    return renderEmpty ? renderEmpty() : <div>No items</div>;
  }
  
  return (
    <ul>
      {items.map((item, index) =>
        renderItem ? (
          renderItem(item, index)
        ) : (
          <li key={index}>{item}</li>
        )
      )}
    </ul>
  );
}
```

### 4. Document Composition APIs

```jsx
/**
 * A flexible dialog component that accepts multiple slots
 * 
 * @example
 * <Dialog>
 *   <Dialog.Title>My Dialog</Dialog.Title>
 *   <Dialog.Content>Content here</Dialog.Content>
 *   <Dialog.Actions>
 *     <button>OK</button>
 *   </Dialog.Actions>
 * </Dialog>
 */
function Dialog({ children }) {
  return <div className="dialog">{children}</div>;
}
```

## Common Patterns Comparison

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| Children | Generic containers, layouts | Simple, natural syntax | Less control over structure |
| Named Props | Fixed structure with slots | Clear expectations | Less flexible |
| Render Props | Share logic and state | Very flexible | More verbose |
| Slot Components | Complex component APIs | Clear structure | More setup |
| Hooks | Share stateful logic | Clean, composable | Can't control rendering |

## Summary

- Composition is React's way to build complex UIs from simple pieces
- Children provide flexible, natural composition
- Render props share logic while controlling rendering
- Component slots provide named placeholders for content
- Prefer composition over inheritance always
- Use hooks for logic sharing, composition for UI structure
- Compound components create cohesive component families
- Clear prop names and good defaults improve DX
- Choose the right pattern for your specific use case

Mastering composition is key to building flexible, reusable React components. Start with simple children composition, and graduate to more advanced patterns as your needs grow.
