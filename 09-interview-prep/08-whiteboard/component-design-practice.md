# Whiteboard: Component Design Practice

## Overview

Practice designing component APIs, props interfaces, state management approaches, composition patterns, and discussing reusability and type safety during whiteboard interviews.

## Component Design Framework

### Step 1: Understand Requirements
- Clarify functionality and use cases
- Identify data inputs and outputs
- Consider edge cases and error states
- Understand performance requirements

### Step 2: Design API
- Define props interface
- Plan component composition
- Consider extensibility points
- Design event handlers

### Step 3: State Management
- Identify what state is needed
- Decide where state lives
- Plan state updates
- Consider derived state

### Step 4: Accessibility & UX
- Keyboard navigation
- ARIA attributes
- Focus management
- Loading and error states

## Example 1: Design a Dropdown/Select Component

### Requirements Clarification

**Questions to Ask:**
- Single or multi-select?
- Async data loading?
- Search/filter capability?
- Custom option rendering?
- Keyboard navigation required?
- Accessibility requirements?

### Component API Design

```typescript
// Props interface
interface DropdownProps<T> {
  // Required props
  options: DropdownOption<T>[];
  value: T | T[] | null;
  onChange: (value: T | T[]) => void;

  // Optional props
  placeholder?: string;
  disabled?: boolean;
  multiple?: boolean;
  searchable?: boolean;
  loading?: boolean;
  error?: string;
  maxHeight?: number;

  // Custom rendering
  renderOption?: (option: DropdownOption<T>) => React.ReactNode;
  renderValue?: (value: T) => React.ReactNode;

  // Callbacks
  onSearch?: (query: string) => void;
  onOpen?: () => void;
  onClose?: () => void;

  // Accessibility
  label?: string;
  'aria-label'?: string;
  'aria-describedby'?: string;
}

interface DropdownOption<T> {
  value: T;
  label: string;
  disabled?: boolean;
  group?: string;
}

// Example usage
<Dropdown
  options={[
    { value: 'us', label: 'United States' },
    { value: 'uk', label: 'United Kingdom' },
    { value: 'ca', label: 'Canada' }
  ]}
  value={selectedCountry}
  onChange={setSelectedCountry}
  placeholder="Select a country"
  searchable
  label="Country"
/>

// Advanced usage with custom rendering
<Dropdown
  options={users}
  value={selectedUser}
  onChange={setSelectedUser}
  renderOption={(option) => (
    <div className="user-option">
      <img src={option.avatar} alt="" />
      <div>
        <div>{option.label}</div>
        <div className="email">{option.email}</div>
      </div>
    </div>
  )}
/>
```

### State Management

```typescript
function Dropdown<T>(props: DropdownProps<T>) {
  // State
  const [isOpen, setIsOpen] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');
  const [highlightedIndex, setHighlightedIndex] = useState(0);
  const [focusedIndex, setFocusedIndex] = useState(-1);

  // Refs
  const dropdownRef = useRef<HTMLDivElement>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  // Derived state
  const filteredOptions = useMemo(() => {
    if (!searchQuery) return props.options;
    return props.options.filter(option =>
      option.label.toLowerCase().includes(searchQuery.toLowerCase())
    );
  }, [props.options, searchQuery]);

  const selectedOptions = useMemo(() => {
    if (props.multiple) {
      return props.options.filter(opt =>
        (props.value as T[]).includes(opt.value)
      );
    }
    return props.options.find(opt => opt.value === props.value);
  }, [props.value, props.options, props.multiple]);

  // Event handlers
  const handleToggle = () => {
    if (props.disabled) return;
    setIsOpen(!isOpen);
    if (!isOpen) {
      props.onOpen?.();
    } else {
      props.onClose?.();
    }
  };

  const handleSelect = (option: DropdownOption<T>) => {
    if (option.disabled) return;

    if (props.multiple) {
      const currentValue = (props.value as T[]) || [];
      const newValue = currentValue.includes(option.value)
        ? currentValue.filter(v => v !== option.value)
        : [...currentValue, option.value];
      props.onChange(newValue);
    } else {
      props.onChange(option.value);
      setIsOpen(false);
    }
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setHighlightedIndex(prev =>
          Math.min(prev + 1, filteredOptions.length - 1)
        );
        break;
      case 'ArrowUp':
        e.preventDefault();
        setHighlightedIndex(prev => Math.max(prev - 1, 0));
        break;
      case 'Enter':
        e.preventDefault();
        if (isOpen && filteredOptions[highlightedIndex]) {
          handleSelect(filteredOptions[highlightedIndex]);
        } else {
          setIsOpen(true);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        break;
    }
  };

  // Close on click outside
  useEffect(() => {
    const handleClickOutside = (e: MouseEvent) => {
      if (dropdownRef.current && !dropdownRef.current.contains(e.target as Node)) {
        setIsOpen(false);
      }
    };

    document.addEventListener('mousedown', handleClickOutside);
    return () => document.removeEventListener('mousedown', handleClickOutside);
  }, []);

  return (
    <div
      ref={dropdownRef}
      className="dropdown"
      onKeyDown={handleKeyDown}>
      {/* Trigger */}
      <button
        type="button"
        onClick={handleToggle}
        disabled={props.disabled}
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-label={props['aria-label']}
        className="dropdown-trigger">
        {/* Selected value display */}
        {selectedOptions ? renderSelectedValue() : props.placeholder}
        <span className="dropdown-arrow">▼</span>
      </button>

      {/* Dropdown menu */}
      {isOpen && (
        <div
          role="listbox"
          className="dropdown-menu"
          style={{ maxHeight: props.maxHeight }}>
          {/* Search input */}
          {props.searchable && (
            <input
              ref={inputRef}
              type="text"
              value={searchQuery}
              onChange={e => setSearchQuery(e.target.value)}
              placeholder="Search..."
              className="dropdown-search"
            />
          )}

          {/* Options */}
          {filteredOptions.map((option, index) => (
            <div
              key={String(option.value)}
              role="option"
              aria-selected={isSelected(option)}
              className={`dropdown-option ${
                highlightedIndex === index ? 'highlighted' : ''
              } ${isSelected(option) ? 'selected' : ''}`}
              onClick={() => handleSelect(option)}
              onMouseEnter={() => setHighlightedIndex(index)}>
              {props.renderOption ? props.renderOption(option) : option.label}
            </div>
          ))}

          {/* Empty state */}
          {filteredOptions.length === 0 && (
            <div className="dropdown-empty">No options found</div>
          )}

          {/* Loading state */}
          {props.loading && (
            <div className="dropdown-loading">Loading...</div>
          )}
        </div>
      )}

      {/* Error message */}
      {props.error && (
        <div className="dropdown-error">{props.error}</div>
      )}
    </div>
  );
}
```

## Example 2: Design a Modal/Dialog Component

### Requirements

- Support different sizes (small, medium, large, fullscreen)
- Dismissible or non-dismissible
- Custom header, content, footer
- Animations
- Focus trap
- Escape key to close
- Backdrop click to close
- Scroll locking

### Component API

```typescript
interface ModalProps {
  // State
  isOpen: boolean;
  onClose: () => void;

  // Content
  title?: React.ReactNode;
  children: React.ReactNode;
  footer?: React.ReactNode;

  // Behavior
  closeOnBackdropClick?: boolean;
  closeOnEscape?: boolean;
  showCloseButton?: boolean;

  // Styling
  size?: 'small' | 'medium' | 'large' | 'fullscreen';
  className?: string;

  // Callbacks
  onOpen?: () => void;
  onAfterClose?: () => void;

  // Accessibility
  'aria-label'?: string;
  'aria-describedby'?: string;
}

// Usage
<Modal
  isOpen={isModalOpen}
  onClose={() => setIsModalOpen(false)}
  title="Confirm Action"
  size="medium"
  closeOnBackdropClick
  closeOnEscape
  footer={
    <>
      <Button onClick={() => setIsModalOpen(false)}>Cancel</Button>
      <Button variant="primary" onClick={handleConfirm}>Confirm</Button>
    </>
  }>
  <p>Are you sure you want to proceed with this action?</p>
</Modal>
```

### Implementation Considerations

```typescript
function Modal(props: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousActiveElement = useRef<HTMLElement | null>(null);

  // Focus trap
  useEffect(() => {
    if (!props.isOpen) return;

    // Store previously focused element
    previousActiveElement.current = document.activeElement as HTMLElement;

    // Focus first focusable element in modal
    const focusableElements = modalRef.current?.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    if (focusableElements && focusableElements.length > 0) {
      (focusableElements[0] as HTMLElement).focus();
    }

    // Cleanup: restore focus
    return () => {
      previousActiveElement.current?.focus();
    };
  }, [props.isOpen]);

  // Escape key handler
  useEffect(() => {
    if (!props.isOpen || !props.closeOnEscape) return;

    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        props.onClose();
      }
    };

    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [props.isOpen, props.closeOnEscape, props.onClose]);

  // Body scroll lock
  useEffect(() => {
    if (props.isOpen) {
      document.body.style.overflow = 'hidden';
    } else {
      document.body.style.overflow = '';
    }

    return () => {
      document.body.style.overflow = '';
    };
  }, [props.isOpen]);

  if (!props.isOpen) return null;

  return createPortal(
    <div
      className="modal-backdrop"
      onClick={(e) => {
        if (props.closeOnBackdropClick && e.target === e.currentTarget) {
          props.onClose();
        }
      }}>
      <div
        ref={modalRef}
        role="dialog"
        aria-modal="true"
        aria-label={props['aria-label']}
        className={`modal modal-${props.size}`}>
        {/* Header */}
        <div className="modal-header">
          {props.title}
          {props.showCloseButton && (
            <button
              type="button"
              onClick={props.onClose}
              aria-label="Close">
              ×
            </button>
          )}
        </div>

        {/* Content */}
        <div className="modal-content">{props.children}</div>

        {/* Footer */}
        {props.footer && (
          <div className="modal-footer">{props.footer}</div>
        )}
      </div>
    </div>,
    document.body
  );
}
```

## Design Principles

### 1. Composition over Configuration

```typescript
// Bad: Configuration-heavy API
<Table
  data={data}
  columns={columns}
  sortable
  filterable
  expandable
  selectable
  pagination />

// Good: Composable API
<Table data={data}>
  <Table.Header>
    <Table.Column sortable>Name</Table.Column>
    <Table.Column filterable>Email</Table.Column>
    <Table.Column>Actions</Table.Column>
  </Table.Header>
  <Table.Body />
  <Table.Pagination />
</Table>
```

### 2. Controlled vs Uncontrolled

```typescript
// Controlled: Parent controls state
<Input
  value={inputValue}
  onChange={setInputValue}
/>

// Uncontrolled: Component controls state
<Input
  defaultValue="initial"
  onBlur={handleBlur}
/>

// Hybrid: Support both
function Input(props) {
  const [internalValue, setInternalValue] = useState(props.defaultValue);
  const isControlled = props.value !== undefined;
  const value = isControlled ? props.value : internalValue;

  const handleChange = (e) => {
    if (!isControlled) {
      setInternalValue(e.target.value);
    }
    props.onChange?.(e.target.value);
  };

  return <input value={value} onChange={handleChange} />;
}
```

### 3. Prop Naming Conventions

```typescript
interface ComponentProps {
  // Boolean props: use "is", "has", "should", "can"
  isOpen: boolean;
  hasError: boolean;
  shouldAutoFocus: boolean;
  canEdit: boolean;

  // Callbacks: use "on" prefix
  onOpen: () => void;
  onChange: (value: T) => void;
  onValidate: (value: T) => boolean;

  // Render props: use "render" prefix
  renderHeader: () => React.ReactNode;
  renderItem: (item: T) => React.ReactNode;

  // Data props: descriptive nouns
  items: T[];
  value: T;
  options: Option[];

  // Style/className props
  className?: string;
  style?: React.CSSProperties;
}
```

## Key Takeaways

1. **Start with Requirements:** Clarify use cases, constraints, and edge cases before designing.

2. **Design for Extensibility:** Use render props, composition, and slots for flexibility.

3. **Type Safety:** Leverage TypeScript generics for reusable, type-safe components.

4. **Accessibility First:** Include ARIA attributes, keyboard navigation, and focus management from the start.

5. **Controlled vs Uncontrolled:** Support both patterns or choose based on use case.

6. **Composition Over Configuration:** Prefer composable APIs over heavily configured single components.

7. **Error States:** Plan for loading, error, and empty states in the API.

8. **Performance:** Consider virtualization, memoization, and lazy loading for large datasets.

9. **Consistent Naming:** Follow conventions for props (on*, is*, render*, etc.).

10. **Document Decisions:** Explain trade-offs and reasoning during the design discussion.

## Whiteboard Tips

- Draw component hierarchy
- Sketch state flow diagrams
- List props interface first
- Discuss trade-offs openly
- Consider edge cases
- Think out loud
- Ask clarifying questions
- Iterate based on feedback
- Focus on API design over implementation details
- Discuss testing strategy
