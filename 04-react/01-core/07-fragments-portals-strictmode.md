# Fragments, Portals, and StrictMode

## Overview

React provides several special built-in components that solve specific problems: Fragments for grouping without DOM nodes, Portals for rendering outside the parent hierarchy, and StrictMode for highlighting potential issues. Understanding these tools helps you write better React applications.

## Fragments

Fragments let you group multiple elements without adding extra nodes to the DOM.

### The Problem

JSX requires a single root element:

```jsx
// Error: Adjacent JSX elements must be wrapped
function Component() {
  return (
    <h1>Title</h1>
    <p>Content</p>
  );
}

// Solution 1: Wrapper div (adds extra DOM node)
function Component() {
  return (
    <div>
      <h1>Title</h1>
      <p>Content</p>
    </div>
  );
}
```

**Problems with wrapper divs:**
- Extra unnecessary DOM nodes
- Can break CSS layouts (flexbox, grid)
- Makes DOM deeper and harder to debug
- Adds semantic meaninglessness

### Fragment Syntax

```jsx
// Long syntax
import { Fragment } from 'react';

function Component() {
  return (
    <Fragment>
      <h1>Title</h1>
      <p>Content</p>
    </Fragment>
  );
}

// Short syntax (most common)
function Component() {
  return (
    <>
      <h1>Title</h1>
      <p>Content</p>
    </>
  );
}
```

### Fragments with Keys

The short syntax cannot accept keys. Use the long syntax when mapping:

```jsx
function DescriptionList({ items }) {
  return (
    <dl>
      {items.map(item => (
        // Need explicit Fragment for key
        <Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.definition}</dd>
        </Fragment>
      ))}
    </dl>
  );
}

// This won't work
function BadExample({ items }) {
  return (
    <dl>
      {items.map(item => (
        <> {/* Error: Can't add key to short syntax */}
          <dt>{item.term}</dt>
          <dd>{item.definition}</dd>
        </>
      ))}
    </dl>
  );
}
```

### Common Use Cases

#### 1. Returning Multiple Elements

```jsx
function UserInfo({ user }) {
  return (
    <>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <p>{user.bio}</p>
    </>
  );
}
```

#### 2. Table Rows

```jsx
function TableRow({ data }) {
  return (
    <>
      <tr>
        <td>{data.name}</td>
        <td>{data.email}</td>
      </tr>
      <tr>
        <td colSpan="2">Additional info: {data.extra}</td>
      </tr>
    </>
  );
}

function Table({ rows }) {
  return (
    <table>
      <tbody>
        {rows.map(row => (
          <TableRow key={row.id} data={row} />
        ))}
      </tbody>
    </table>
  );
}
```

#### 3. Conditional Groups

```jsx
function ConditionalContent({ showDetails, data }) {
  return (
    <div>
      <h1>{data.title}</h1>
      {showDetails && (
        <>
          <p>{data.description}</p>
          <p>{data.date}</p>
          <p>{data.author}</p>
        </>
      )}
    </div>
  );
}
```

#### 4. List Items

```jsx
function Menu({ items }) {
  return (
    <ul>
      {items.map(item => (
        <Fragment key={item.id}>
          <li>{item.label}</li>
          {item.hasSubmenu && (
            <li className="submenu">
              <ul>
                {item.submenu.map(sub => (
                  <li key={sub.id}>{sub.label}</li>
                ))}
              </ul>
            </li>
          )}
        </Fragment>
      ))}
    </ul>
  );
}
```

### Fragments vs Divs

```jsx
// With div - creates extra DOM node
function WithDiv() {
  return (
    <div>
      <h1>Title</h1>
      <p>Content</p>
    </div>
  );
}
// DOM: <div><h1>Title</h1><p>Content</p></div>

// With Fragment - no extra DOM node
function WithFragment() {
  return (
    <>
      <h1>Title</h1>
      <p>Content</p>
    </>
  );
}
// DOM: <h1>Title</h1><p>Content</p>
```

### Why Fragments Matter

```jsx
// CSS Grid example
function GridLayout() {
  return (
    <div className="grid">
      {/* Fragment doesn't break grid layout */}
      <>
        <div className="item">1</div>
        <div className="item">2</div>
      </>
      <div className="item">3</div>
    </div>
  );
}

// If we used a div instead of fragment:
// <div class="grid">
//   <div>                    ← Extra wrapper breaks grid!
//     <div class="item">1</div>
//     <div class="item">2</div>
//   </div>
//   <div class="item">3</div>
// </div>
```

## Portals

Portals provide a way to render children into a DOM node that exists outside the parent component's DOM hierarchy.

### Creating a Portal

```jsx
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(
    <div className="modal">
      {children}
    </div>,
    document.getElementById('modal-root')
  );
}

// In your HTML, add a mount point
// <div id="root"></div>
// <div id="modal-root"></div>
```

### Why Use Portals?

**Common scenarios:**
- Modals that need to be on top of everything
- Tooltips that escape overflow: hidden
- Full-screen overlays
- Third-party widget containers

Without portals:

```jsx
function App() {
  return (
    <div style={{ overflow: 'hidden', position: 'relative' }}>
      <Modal>
        {/* Modal is clipped by parent's overflow: hidden */}
      </Modal>
    </div>
  );
}
```

With portals:

```jsx
function Modal({ children }) {
  return createPortal(
    children,
    document.body
  );
}

function App() {
  return (
    <div style={{ overflow: 'hidden', position: 'relative' }}>
      {/* Modal renders outside this div, not affected by overflow */}
      <Modal>Content</Modal>
    </div>
  );
}
```

### Complete Modal Example

```jsx
function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;
  
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <button className="modal-close" onClick={onClose}>×</button>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)}>
        <h2>Modal Title</h2>
        <p>Modal content here</p>
      </Modal>
    </div>
  );
}

// CSS
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  max-width: 500px;
  position: relative;
}
```

### Tooltip with Portal

```jsx
function Tooltip({ children, content, position = 'top' }) {
  const [isVisible, setIsVisible] = useState(false);
  const [coords, setCoords] = useState({ x: 0, y: 0 });
  const triggerRef = useRef();
  
  const showTooltip = () => {
    const rect = triggerRef.current.getBoundingClientRect();
    setCoords({
      x: rect.left + rect.width / 2,
      y: rect.top
    });
    setIsVisible(true);
  };
  
  return (
    <>
      <span
        ref={triggerRef}
        onMouseEnter={showTooltip}
        onMouseLeave={() => setIsVisible(false)}
      >
        {children}
      </span>
      
      {isVisible && createPortal(
        <div
          className={`tooltip tooltip-${position}`}
          style={{
            position: 'absolute',
            left: coords.x,
            top: coords.y
          }}
        >
          {content}
        </div>,
        document.body
      )}
    </>
  );
}

// Usage
<p>
  Hover over <Tooltip content="This is helpful info">this text</Tooltip> to see tooltip
</p>
```

### Event Bubbling with Portals

Events bubble through the React tree, not the DOM tree:

```jsx
function App() {
  const handleClick = () => {
    console.log('App clicked');
  };
  
  return (
    <div onClick={handleClick}>
      <Modal>
        {/* Click here bubbles to App, even though it's in a portal */}
        <button>Click me</button>
      </Modal>
    </div>
  );
}
```

This is useful because:
- Event handling works as expected in React
- You don't need to worry about DOM structure
- Portaled components behave like normal children

### Portal with Context

Portals maintain the React context:

```jsx
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <div>
        <Modal>
          {/* Can access ThemeContext even though rendered elsewhere */}
          <ThemedButton />
        </Modal>
      </div>
    </ThemeContext.Provider>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext); // Works!
  return <button className={theme}>Themed Button</button>;
}
```

### Creating Portal Container

```jsx
// Custom hook for portal container
function usePortal(id) {
  const rootRef = useRef(null);
  
  useEffect(() => {
    // Create or get existing portal root
    const existingRoot = document.getElementById(id);
    const root = existingRoot || document.createElement('div');
    
    if (!existingRoot) {
      root.setAttribute('id', id);
      document.body.appendChild(root);
    }
    
    rootRef.current = root;
    
    return () => {
      if (!existingRoot && root.parentNode) {
        root.parentNode.removeChild(root);
      }
    };
  }, [id]);
  
  return rootRef.current;
}

// Usage
function Modal({ children }) {
  const target = usePortal('modal-root');
  
  if (!target) return null;
  
  return createPortal(
    <div className="modal">{children}</div>,
    target
  );
}
```

## StrictMode

StrictMode is a development tool that helps identify potential problems.

### Basic Usage

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root'));
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

### What StrictMode Does

**1. Identifies unsafe lifecycles**

```jsx
// StrictMode warns about these deprecated lifecycles:
class OldComponent extends Component {
  componentWillMount() { }        // Warning!
  componentWillReceiveProps() { } // Warning!
  componentWillUpdate() { }       // Warning!
}
```

**2. Warns about legacy string refs**

```jsx
class Component extends React.Component {
  render() {
    // Warning: string ref
    return <input ref="myInput" />;
    
    // Better: callback ref or useRef
    return <input ref={node => this.myInput = node} />;
  }
}
```

**3. Warns about deprecated findDOMNode**

```jsx
import { findDOMNode } from 'react-dom';

class Component extends React.Component {
  componentDidMount() {
    // Warning: findDOMNode is deprecated
    const node = findDOMNode(this);
  }
}
```

**4. Detects unexpected side effects**

StrictMode intentionally double-invokes these functions (development only):
- Function component bodies
- useState, useMemo, useReducer initializers
- Class constructor, render, and shouldComponentUpdate

```jsx
function Component() {
  console.log('Rendering'); // Logs twice in StrictMode
  
  const [state] = useState(() => {
    console.log('State initializer'); // Runs twice
    return 0;
  });
  
  return <div>{state}</div>;
}
```

**Why?** To help you find bugs with impure components.

**5. Warns about legacy context**

```jsx
// Warning: Legacy context API
class Component extends React.Component {
  static contextTypes = {
    theme: PropTypes.string
  };
  
  render() {
    return <div>{this.context.theme}</div>;
  }
}

// Use modern Context API instead
const ThemeContext = createContext();
```

### Partial Activation

You can wrap only part of your app:

```jsx
function App() {
  return (
    <div>
      <Header /> {/* Not in StrictMode */}
      
      <StrictMode>
        <Dashboard /> {/* In StrictMode */}
        <Settings />  {/* In StrictMode */}
      </StrictMode>
      
      <Footer /> {/* Not in StrictMode */}
    </div>
  );
}
```

### Understanding Double Rendering

```jsx
// This component has a side effect (bad)
function BadComponent({ items }) {
  items.push('extra'); // Mutation!
  return <div>{items.length}</div>;
}

// StrictMode calls this twice:
// First render: items = [1, 2, 3] → pushes 'extra' → [1, 2, 3, 'extra']
// Second render: items = [1, 2, 3, 'extra'] → pushes 'extra' → [1, 2, 3, 'extra', 'extra']
// Helps you catch the bug!

// Fixed version (pure component)
function GoodComponent({ items }) {
  // Don't mutate props
  const itemsWithExtra = [...items, 'extra'];
  return <div>{itemsWithExtra.length}</div>;
}
```

### StrictMode and useEffect

useEffect is not double-invoked, but cleanup + setup is called twice:

```jsx
function Component() {
  useEffect(() => {
    console.log('Effect setup');
    
    return () => {
      console.log('Effect cleanup');
    };
  }, []);
  
  // In StrictMode (development):
  // 1. Effect setup
  // 2. Effect cleanup
  // 3. Effect setup
  
  // This helps catch missing cleanup logic
}
```

Example of bug StrictMode helps find:

```jsx
// Bug: Missing cleanup
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    // Missing cleanup!
  }, [roomId]);
  
  return <div>Chat</div>;
}

// StrictMode reveals the problem:
// 1. Connect to room1
// 2. (Unmount simulation) - no cleanup, still connected!
// 3. Connect to room1 again - now two connections!

// Fixed version
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    
    return () => {
      connection.disconnect(); // Proper cleanup
    };
  }, [roomId]);
  
  return <div>Chat</div>;
}
```

### Production Behavior

StrictMode has no effect in production:
- No double rendering
- No performance impact
- All checks are development-only
- Safe to leave in production code

```jsx
// This is fine to ship to production
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

## Best Practices

### Fragments

```jsx
// Good: Use short syntax when you don't need keys
function Component() {
  return (
    <>
      <Header />
      <Content />
    </>
  );
}

// Good: Use long syntax with keys
function List({ items }) {
  return items.map(item => (
    <Fragment key={item.id}>
      <dt>{item.term}</dt>
      <dd>{item.def}</dd>
    </Fragment>
  ));
}

// Bad: Unnecessary div wrapper
function Component() {
  return (
    <div>
      <Header />
      <Content />
    </div>
  );
}
```

### Portals

```jsx
// Good: Portal for modals
function Modal({ children }) {
  return createPortal(
    children,
    document.getElementById('modal-root')
  );
}

// Good: Clean up portal containers
useEffect(() => {
  const el = document.createElement('div');
  document.body.appendChild(el);
  
  return () => {
    document.body.removeChild(el);
  };
}, []);

// Bad: Modal in normal DOM (can be clipped)
function BadModal({ children }) {
  return <div className="modal">{children}</div>;
}
```

### StrictMode

```jsx
// Good: Enable in development
if (process.env.NODE_ENV === 'development') {
  root.render(
    <StrictMode>
      <App />
    </StrictMode>
  );
} else {
  root.render(<App />);
}

// Or just always use it (no production impact)
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);

// Good: Fix warnings, don't ignore them
// StrictMode warnings indicate real problems
```

## Summary

### Fragments
- Group elements without adding DOM nodes
- Use `<>...</>` for simple cases
- Use `<Fragment key={...}>...</Fragment>` when mapping
- Prevents extra DOM nodes and CSS layout issues
- Essential for clean, semantic HTML

### Portals
- Render components outside parent DOM hierarchy
- Use for modals, tooltips, dropdowns
- Created with `createPortal(children, domNode)`
- Events still bubble through React tree
- Context still works normally

### StrictMode
- Development-only tool for finding bugs
- Activates extra checks and warnings
- Intentionally double-invokes functions
- Helps find side effects and impure code
- No production impact
- Enable it to catch problems early

These three tools solve different problems: Fragments for cleaner DOM, Portals for DOM placement, and StrictMode for code quality. Use them appropriately to build better React applications.
