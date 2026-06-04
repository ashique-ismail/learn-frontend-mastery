# Compound Components Pattern

## Table of Contents
- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Implementation Patterns](#implementation-patterns)
- [Context-based Compound Components](#context-based-compound-components)
- [Advanced Techniques](#advanced-techniques)
- [TypeScript Implementation](#typescript-implementation)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

The Compound Components pattern is a React design pattern where components work together as a cohesive unit, sharing implicit state and behavior. Think of HTML elements like `<select>` and `<option>` - they work together naturally.

### Why Compound Components?

```javascript
// ❌ Without Compound Components: Prop explosion
<Accordion
  title="Section 1"
  content="Content 1"
  isOpen={isOpen1}
  onToggle={handleToggle1}
  icon={<ChevronIcon />}
  headerClassName="custom-header"
  contentClassName="custom-content"
/>

// ✅ With Compound Components: Flexible and composable
<Accordion>
  <Accordion.Header>
    <Accordion.Title>Section 1</Accordion.Title>
    <Accordion.Icon />
  </Accordion.Header>
  <Accordion.Content>
    Content 1
  </Accordion.Content>
</Accordion>
```

### Benefits

1. **Flexible API**: Users can arrange and customize sub-components
2. **Implicit State Sharing**: Components share state without prop drilling
3. **Inversion of Control**: Users control rendering and layout
4. **Separation of Concerns**: Each component has a single responsibility
5. **Better Developer Experience**: Natural, intuitive API

## Basic Concepts

### Simple Compound Component

```javascript
// Basic structure
function Select({ children, value, onChange }) {
  return (
    <select value={value} onChange={onChange}>
      {children}
    </select>
  );
}

Select.Option = function Option({ value, children }) {
  return <option value={value}>{children}</option>;
};

// Usage
function App() {
  const [value, setValue] = useState('');
  
  return (
    <Select value={value} onChange={(e) => setValue(e.target.value)}>
      <Select.Option value="apple">Apple</Select.Option>
      <Select.Option value="banana">Banana</Select.Option>
      <Select.Option value="orange">Orange</Select.Option>
    </Select>
  );
}
```

### Visual Representation

```
Parent Component (Accordion)
├─ Shared State (isOpen, toggle)
├─ Context Provider
│
├─ Child: Accordion.Header
│  ├─ Accesses: toggle function
│  └─ Renders: trigger element
│
├─ Child: Accordion.Title
│  ├─ Accesses: isOpen state
│  └─ Renders: title with icon
│
└─ Child: Accordion.Content
   ├─ Accesses: isOpen state
   └─ Renders: conditionally based on isOpen
```

## Implementation Patterns

### Pattern 1: Static Properties

```javascript
// Parent component
function Tabs({ children, defaultTab = 0 }) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  return (
    <div className="tabs">
      {React.Children.map(children, (child, index) => {
        return React.cloneElement(child, {
          isActive: index === activeTab,
          onSelect: () => setActiveTab(index),
          index
        });
      })}
    </div>
  );
}

// Child components as static properties
Tabs.Tab = function Tab({ children, isActive, onSelect }) {
  return (
    <button
      className={`tab ${isActive ? 'active' : ''}`}
      onClick={onSelect}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function Panel({ children, isActive }) {
  if (!isActive) return null;
  return <div className="panel">{children}</div>;
};

// Usage
function App() {
  return (
    <Tabs defaultTab={0}>
      <Tabs.Tab>Tab 1</Tabs.Tab>
      <Tabs.Tab>Tab 2</Tabs.Tab>
      <Tabs.Tab>Tab 3</Tabs.Tab>
      
      <Tabs.Panel>Content 1</Tabs.Panel>
      <Tabs.Panel>Content 2</Tabs.Panel>
      <Tabs.Panel>Content 3</Tabs.Panel>
    </Tabs>
  );
}
```

### Pattern 2: React.Children with cloneElement

```javascript
function RadioGroup({ children, name, value, onChange }) {
  return (
    <div className="radio-group">
      {React.Children.map(children, (child) => {
        if (React.isValidElement(child)) {
          return React.cloneElement(child, {
            name,
            checked: child.props.value === value,
            onChange: () => onChange(child.props.value)
          });
        }
        return child;
      })}
    </div>
  );
}

RadioGroup.Option = function RadioOption({ 
  value, 
  children, 
  name,
  checked,
  onChange 
}) {
  return (
    <label className="radio-option">
      <input
        type="radio"
        name={name}
        value={value}
        checked={checked}
        onChange={onChange}
      />
      <span>{children}</span>
    </label>
  );
};

// Usage
function Form() {
  const [selected, setSelected] = useState('option1');

  return (
    <RadioGroup 
      name="options" 
      value={selected} 
      onChange={setSelected}
    >
      <RadioGroup.Option value="option1">
        Option 1
      </RadioGroup.Option>
      <RadioGroup.Option value="option2">
        Option 2
      </RadioGroup.Option>
      <RadioGroup.Option value="option3">
        Option 3
      </RadioGroup.Option>
    </RadioGroup>
  );
}
```

## Context-based Compound Components

### Basic Context Pattern

```javascript
// Create context
const AccordionContext = createContext(null);

// Parent component
function Accordion({ children, allowMultiple = false }) {
  const [openSections, setOpenSections] = useState([]);

  const toggleSection = useCallback((id) => {
    setOpenSections(prev => {
      if (allowMultiple) {
        return prev.includes(id)
          ? prev.filter(sectionId => sectionId !== id)
          : [...prev, id];
      } else {
        return prev.includes(id) ? [] : [id];
      }
    });
  }, [allowMultiple]);

  const isOpen = useCallback((id) => {
    return openSections.includes(id);
  }, [openSections]);

  const value = useMemo(() => ({
    toggleSection,
    isOpen
  }), [toggleSection, isOpen]);

  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

// Custom hook for accessing context
function useAccordion() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('Accordion components must be used within Accordion');
  }
  return context;
}

// Child components
Accordion.Section = function AccordionSection({ id, children }) {
  return (
    <div className="accordion-section">
      {children}
    </div>
  );
};

Accordion.Header = function AccordionHeader({ id, children }) {
  const { toggleSection, isOpen } = useAccordion();
  const expanded = isOpen(id);

  return (
    <button
      className="accordion-header"
      onClick={() => toggleSection(id)}
      aria-expanded={expanded}
    >
      {children}
      <span className={`icon ${expanded ? 'expanded' : ''}`}>
        ▼
      </span>
    </button>
  );
};

Accordion.Content = function AccordionContent({ id, children }) {
  const { isOpen } = useAccordion();
  const expanded = isOpen(id);

  return (
    <div 
      className={`accordion-content ${expanded ? 'open' : 'closed'}`}
      aria-hidden={!expanded}
    >
      {expanded && children}
    </div>
  );
};

// Usage
function App() {
  return (
    <Accordion allowMultiple={true}>
      <Accordion.Section id="section1">
        <Accordion.Header id="section1">
          Section 1
        </Accordion.Header>
        <Accordion.Content id="section1">
          Content for section 1
        </Accordion.Content>
      </Accordion.Section>

      <Accordion.Section id="section2">
        <Accordion.Header id="section2">
          Section 2
        </Accordion.Header>
        <Accordion.Content id="section2">
          Content for section 2
        </Accordion.Content>
      </Accordion.Section>
    </Accordion>
  );
}
```

### Advanced Context Pattern

```javascript
// Multiple contexts for separation of concerns
const TabsStateContext = createContext(null);
const TabsDispatchContext = createContext(null);

function Tabs({ children, defaultTab = 0, onChange }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  const tabsRef = useRef([]);
  const panelsRef = useRef([]);

  // Register tabs and panels
  const registerTab = useCallback((id, index) => {
    tabsRef.current[index] = id;
  }, []);

  const registerPanel = useCallback((id, index) => {
    panelsRef.current[index] = id;
  }, []);

  // Handle tab change
  const selectTab = useCallback((index) => {
    setActiveTab(index);
    onChange?.(index);
  }, [onChange]);

  // Keyboard navigation
  const handleKeyDown = useCallback((e, currentIndex) => {
    let newIndex = currentIndex;

    switch (e.key) {
      case 'ArrowLeft':
        newIndex = currentIndex > 0 ? currentIndex - 1 : tabsRef.current.length - 1;
        break;
      case 'ArrowRight':
        newIndex = currentIndex < tabsRef.current.length - 1 ? currentIndex + 1 : 0;
        break;
      case 'Home':
        newIndex = 0;
        break;
      case 'End':
        newIndex = tabsRef.current.length - 1;
        break;
      default:
        return;
    }

    e.preventDefault();
    selectTab(newIndex);
  }, [selectTab]);

  const state = useMemo(() => ({
    activeTab,
    tabsRef,
    panelsRef
  }), [activeTab]);

  const dispatch = useMemo(() => ({
    selectTab,
    registerTab,
    registerPanel,
    handleKeyDown
  }), [selectTab, registerTab, registerPanel, handleKeyDown]);

  return (
    <TabsStateContext.Provider value={state}>
      <TabsDispatchContext.Provider value={dispatch}>
        <div className="tabs">{children}</div>
      </TabsDispatchContext.Provider>
    </TabsStateContext.Provider>
  );
}

// Custom hooks
function useTabsState() {
  const context = useContext(TabsStateContext);
  if (!context) {
    throw new Error('useTabsState must be used within Tabs');
  }
  return context;
}

function useTabsDispatch() {
  const context = useContext(TabsDispatchContext);
  if (!context) {
    throw new Error('useTabsDispatch must be used within Tabs');
  }
  return context;
}

// Child components
Tabs.List = function TabsList({ children }) {
  return (
    <div role="tablist" className="tabs-list">
      {children}
    </div>
  );
};

Tabs.Tab = function Tab({ index, children }) {
  const { activeTab } = useTabsState();
  const { selectTab, registerTab, handleKeyDown } = useTabsDispatch();
  const isActive = activeTab === index;
  const id = `tab-${index}`;
  const panelId = `panel-${index}`;

  useEffect(() => {
    registerTab(id, index);
  }, [registerTab, id, index]);

  return (
    <button
      role="tab"
      id={id}
      aria-controls={panelId}
      aria-selected={isActive}
      tabIndex={isActive ? 0 : -1}
      onClick={() => selectTab(index)}
      onKeyDown={(e) => handleKeyDown(e, index)}
      className={`tab ${isActive ? 'active' : ''}`}
    >
      {children}
    </button>
  );
};

Tabs.Panels = function TabsPanels({ children }) {
  return (
    <div className="tabs-panels">
      {children}
    </div>
  );
};

Tabs.Panel = function TabPanel({ index, children }) {
  const { activeTab } = useTabsState();
  const { registerPanel } = useTabsDispatch();
  const isActive = activeTab === index;
  const id = `panel-${index}`;
  const tabId = `tab-${index}`;

  useEffect(() => {
    registerPanel(id, index);
  }, [registerPanel, id, index]);

  if (!isActive) return null;

  return (
    <div
      role="tabpanel"
      id={id}
      aria-labelledby={tabId}
      tabIndex={0}
      className="tab-panel"
    >
      {children}
    </div>
  );
};

// Usage
function App() {
  return (
    <Tabs defaultTab={0} onChange={(index) => console.log('Tab', index)}>
      <Tabs.List>
        <Tabs.Tab index={0}>Tab 1</Tabs.Tab>
        <Tabs.Tab index={1}>Tab 2</Tabs.Tab>
        <Tabs.Tab index={2}>Tab 3</Tabs.Tab>
      </Tabs.List>

      <Tabs.Panels>
        <Tabs.Panel index={0}>
          <h2>Panel 1</h2>
          <p>Content for tab 1</p>
        </Tabs.Panel>
        <Tabs.Panel index={1}>
          <h2>Panel 2</h2>
          <p>Content for tab 2</p>
        </Tabs.Panel>
        <Tabs.Panel index={2}>
          <h2>Panel 3</h2>
          <p>Content for tab 3</p>
        </Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

## Advanced Techniques

### Controlled vs Uncontrolled

```javascript
function Dropdown({ 
  value: controlledValue,
  defaultValue,
  onChange,
  children 
}) {
  const [uncontrolledValue, setUncontrolledValue] = useState(defaultValue);
  
  // Determine if controlled
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : uncontrolledValue;

  const handleChange = useCallback((newValue) => {
    if (!isControlled) {
      setUncontrolledValue(newValue);
    }
    onChange?.(newValue);
  }, [isControlled, onChange]);

  const contextValue = useMemo(() => ({
    value,
    onChange: handleChange
  }), [value, handleChange]);

  return (
    <DropdownContext.Provider value={contextValue}>
      <div className="dropdown">{children}</div>
    </DropdownContext.Provider>
  );
}

// Usage - Uncontrolled
<Dropdown defaultValue="option1">
  <Dropdown.Option value="option1">Option 1</Dropdown.Option>
</Dropdown>

// Usage - Controlled
const [value, setValue] = useState('option1');
<Dropdown value={value} onChange={setValue}>
  <Dropdown.Option value="option1">Option 1</Dropdown.Option>
</Dropdown>
```

### Composition with Slots

```javascript
const CardContext = createContext(null);

function Card({ children, variant = 'default' }) {
  const value = useMemo(() => ({ variant }), [variant]);

  return (
    <CardContext.Provider value={value}>
      <div className={`card card-${variant}`}>
        {children}
      </div>
    </CardContext.Provider>
  );
}

Card.Header = function CardHeader({ children, actions }) {
  return (
    <div className="card-header">
      <div className="card-header-content">{children}</div>
      {actions && (
        <div className="card-header-actions">{actions}</div>
      )}
    </div>
  );
};

Card.Title = function CardTitle({ children }) {
  return <h3 className="card-title">{children}</h3>;
};

Card.Description = function CardDescription({ children }) {
  return <p className="card-description">{children}</p>;
};

Card.Content = function CardContent({ children }) {
  return <div className="card-content">{children}</div>;
};

Card.Footer = function CardFooter({ children }) {
  return <div className="card-footer">{children}</div>;
};

// Usage with flexible composition
function App() {
  return (
    <Card variant="elevated">
      <Card.Header 
        actions={
          <>
            <button>Edit</button>
            <button>Delete</button>
          </>
        }
      >
        <Card.Title>Card Title</Card.Title>
        <Card.Description>Card description</Card.Description>
      </Card.Header>
      
      <Card.Content>
        <p>Main content goes here</p>
      </Card.Content>
      
      <Card.Footer>
        <button>Cancel</button>
        <button>Save</button>
      </Card.Footer>
    </Card>
  );
}
```

### Validation and Type Checking

```javascript
function validateChildren(children, allowedTypes) {
  React.Children.forEach(children, (child) => {
    if (!React.isValidElement(child)) return;
    
    const childType = child.type.displayName || child.type.name;
    
    if (!allowedTypes.includes(childType)) {
      throw new Error(
        `Invalid child component: ${childType}. ` +
        `Only ${allowedTypes.join(', ')} are allowed.`
      );
    }
  });
}

function Menu({ children }) {
  // Validate that only allowed components are children
  if (process.env.NODE_ENV !== 'production') {
    validateChildren(children, ['MenuItem', 'MenuDivider', 'MenuGroup']);
  }

  return <div className="menu">{children}</div>;
}

Menu.Item = function MenuItem({ children, onClick, disabled }) {
  return (
    <button
      className="menu-item"
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
};
Menu.Item.displayName = 'MenuItem';

Menu.Divider = function MenuDivider() {
  return <div className="menu-divider" />;
};
Menu.Divider.displayName = 'MenuDivider';

Menu.Group = function MenuGroup({ label, children }) {
  return (
    <div className="menu-group">
      {label && <div className="menu-group-label">{label}</div>}
      {children}
    </div>
  );
};
Menu.Group.displayName = 'MenuGroup';
```

## TypeScript Implementation

```typescript
import { createContext, useContext, useState, ReactNode } from 'react';

// Type definitions
interface AccordionContextValue {
  openItems: string[];
  toggleItem: (id: string) => void;
  isOpen: (id: string) => boolean;
}

interface AccordionProps {
  children: ReactNode;
  allowMultiple?: boolean;
  defaultOpen?: string[];
}

interface AccordionItemProps {
  id: string;
  children: ReactNode;
}

interface AccordionHeaderProps {
  id: string;
  children: ReactNode;
}

interface AccordionContentProps {
  id: string;
  children: ReactNode;
}

// Context
const AccordionContext = createContext<AccordionContextValue | null>(null);

// Custom hook with type safety
function useAccordion(): AccordionContextValue {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('Accordion components must be used within Accordion');
  }
  return context;
}

// Parent component
export function Accordion({ 
  children, 
  allowMultiple = false,
  defaultOpen = []
}: AccordionProps) {
  const [openItems, setOpenItems] = useState<string[]>(defaultOpen);

  const toggleItem = (id: string) => {
    setOpenItems(prev => {
      if (allowMultiple) {
        return prev.includes(id)
          ? prev.filter(item => item !== id)
          : [...prev, id];
      } else {
        return prev.includes(id) ? [] : [id];
      }
    });
  };

  const isOpen = (id: string) => openItems.includes(id);

  return (
    <AccordionContext.Provider value={{ openItems, toggleItem, isOpen }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

// Child components
Accordion.Item = function AccordionItem({ id, children }: AccordionItemProps) {
  return <div className="accordion-item" data-item-id={id}>{children}</div>;
};

Accordion.Header = function AccordionHeader({ 
  id, 
  children 
}: AccordionHeaderProps) {
  const { toggleItem, isOpen } = useAccordion();
  const expanded = isOpen(id);

  return (
    <button
      className="accordion-header"
      onClick={() => toggleItem(id)}
      aria-expanded={expanded}
    >
      {children}
    </button>
  );
};

Accordion.Content = function AccordionContent({ 
  id, 
  children 
}: AccordionContentProps) {
  const { isOpen } = useAccordion();
  const expanded = isOpen(id);

  if (!expanded) return null;

  return (
    <div className="accordion-content">
      {children}
    </div>
  );
};
```

## Common Mistakes

### 1. Not Validating Context

```javascript
// ❌ WRONG: No validation
function AccordionHeader() {
  const context = useContext(AccordionContext);
  // context might be null!
  return <button onClick={context.toggle}>...</button>;
}

// ✅ CORRECT: Validate context
function AccordionHeader() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('AccordionHeader must be used within Accordion');
  }
  return <button onClick={context.toggle}>...</button>;
}
```

### 2. Creating Context in Render

```javascript
// ❌ WRONG: New context every render
function Parent({ children }) {
  const value = { data: 'value' }; // New object every render!
  return (
    <MyContext.Provider value={value}>
      {children}
    </MyContext.Provider>
  );
}

// ✅ CORRECT: Memoize context value
function Parent({ children }) {
  const value = useMemo(() => ({ data: 'value' }), []);
  return (
    <MyContext.Provider value={value}>
      {children}
    </MyContext.Provider>
  );
}
```

### 3. Breaking Component Hierarchy

```javascript
// ❌ WRONG: Components used incorrectly
<Tabs>
  <Tabs.Panel>Panel</Tabs.Panel>  {/* No Tab! */}
</Tabs>

// ✅ CORRECT: Proper hierarchy
<Tabs>
  <Tabs.Tab>Tab</Tabs.Tab>
  <Tabs.Panel>Panel</Tabs.Panel>
</Tabs>
```

## Best Practices

### 1. Provide Clear Error Messages

```javascript
function useAccordion() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error(
      'Accordion compound components (Accordion.Header, Accordion.Content) ' +
      'must be used within an <Accordion> component.'
    );
  }
  return context;
}
```

### 2. Export Sub-components

```javascript
// ✅ GOOD: Named exports for tree-shaking
export { Accordion, AccordionItem, AccordionHeader, AccordionContent };

// Also attach to parent for compound pattern
Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
Accordion.Content = AccordionContent;
```

### 3. Document Usage

```javascript
/**
 * Accordion component for collapsible content sections
 * 
 * @example
 * <Accordion allowMultiple>
 *   <Accordion.Item id="item1">
 *     <Accordion.Header id="item1">Header</Accordion.Header>
 *     <Accordion.Content id="item1">Content</Accordion.Content>
 *   </Accordion.Item>
 * </Accordion>
 */
function Accordion({ children, allowMultiple = false }) {
  // Implementation
}
```

### 4. Support Controlled and Uncontrolled

```javascript
function Component({ value, defaultValue, onChange }) {
  const isControlled = value !== undefined;
  const [internalValue, setInternalValue] = useState(defaultValue);
  
  const actualValue = isControlled ? value : internalValue;
  const actualOnChange = isControlled ? onChange : setInternalValue;
  
  // Use actualValue and actualOnChange
}
```

## Real-World Examples

### Modal Dialog System

```javascript
const ModalContext = createContext(null);

function Modal({ children, isOpen, onClose }) {
  const modalRef = useRef(null);
  const [focusTrap, setFocusTrap] = useState(null);

  useEffect(() => {
    if (!isOpen) return;

    const previousFocus = document.activeElement;
    
    // Focus first focusable element
    const focusableElements = modalRef.current?.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    if (focusableElements?.length > 0) {
      focusableElements[0].focus();
    }

    // Handle Escape key
    const handleEscape = (e) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };

    document.addEventListener('keydown', handleEscape);

    return () => {
      document.removeEventListener('keydown', handleEscape);
      previousFocus?.focus();
    };
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  const value = useMemo(() => ({ onClose }), [onClose]);

  return createPortal(
    <ModalContext.Provider value={value}>
      <div className="modal-overlay" onClick={onClose}>
        <div 
          ref={modalRef}
          className="modal"
          onClick={(e) => e.stopPropagation()}
          role="dialog"
          aria-modal="true"
        >
          {children}
        </div>
      </div>
    </ModalContext.Provider>,
    document.body
  );
}

function useModal() {
  const context = useContext(ModalContext);
  if (!context) {
    throw new Error('Modal components must be used within Modal');
  }
  return context;
}

Modal.Header = function ModalHeader({ children }) {
  return (
    <div className="modal-header">
      {children}
    </div>
  );
};

Modal.Title = function ModalTitle({ children }) {
  return <h2 className="modal-title">{children}</h2>;
};

Modal.CloseButton = function ModalCloseButton() {
  const { onClose } = useModal();
  return (
    <button 
      className="modal-close"
      onClick={onClose}
      aria-label="Close"
    >
      ×
    </button>
  );
};

Modal.Body = function ModalBody({ children }) {
  return <div className="modal-body">{children}</div>;
};

Modal.Footer = function ModalFooter({ children }) {
  return <div className="modal-footer">{children}</div>;
};

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>

      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)}>
        <Modal.Header>
          <Modal.Title>Confirm Action</Modal.Title>
          <Modal.CloseButton />
        </Modal.Header>

        <Modal.Body>
          <p>Are you sure you want to proceed?</p>
        </Modal.Body>

        <Modal.Footer>
          <button onClick={() => setIsOpen(false)}>Cancel</button>
          <button onClick={() => { /* confirm */ }}>Confirm</button>
        </Modal.Footer>
      </Modal>
    </div>
  );
}
```

## Interview Questions

### Q1: What are compound components?

**Answer:** Compound components are a React pattern where multiple components work together to form a complete UI element, sharing implicit state through context. They provide a flexible, composable API similar to HTML elements like `<select>` and `<option>`.

### Q2: When should you use compound components?

**Answer:** Use when:
- Components are tightly coupled and always used together
- You want to provide flexible composition
- Internal state sharing is needed
- You want to avoid prop drilling
- API needs to feel natural and intuitive

Examples: Tabs, Accordions, Dropdowns, Modals

### Q3: What are the advantages over prop-based APIs?

**Answer:**
- **Flexibility**: Users control layout and composition
- **Separation**: Each component has single responsibility
- **Extensibility**: Easy to add new sub-components
- **No prop explosion**: Avoid passing many props
- **Better DX**: Natural, intuitive API

### Q4: How do compound components share state?

**Answer:** Through React Context:
1. Parent creates context with shared state
2. Parent provides context value
3. Children consume context with useContext
4. Custom hook ensures components are used correctly

## Key Takeaways

1. **Flexible composition** - Users control structure and layout
2. **Implicit state sharing** - Via context, no prop drilling
3. **Better DX** - Intuitive, natural API
4. **Separation of concerns** - Each component focused
5. **Extensible** - Easy to add new sub-components
6. **Type-safe** - Works great with TypeScript
7. **Context is key** - For state sharing between components

## Resources

### Official Documentation
- [Composition vs Inheritance](https://react.dev/learn/passing-props-to-a-component)
- [Context API](https://react.dev/learn/passing-data-deeply-with-context)

### Articles
- [Compound Components Pattern](https://kentcdodds.com/blog/compound-components-with-react-hooks)
- [React Component Patterns](https://www.patterns.dev/posts/compound-pattern)

### Examples
- [Reach UI](https://reach.tech/) - Great examples of compound components
- [Radix UI](https://www.radix-ui.com/) - Modern accessible components
- [Headless UI](https://headlessui.com/) - Unstyled compound components

---

**Next Steps:** Learn about render props pattern, higher-order components, and custom hooks for complementary composition techniques.
