# State Reducer Pattern

## Overview

The State Reducer Pattern is an advanced React pattern that gives users of your component complete control over its internal state management. It inverts control by allowing consumers to intercept and modify state changes before they're applied, making components extremely flexible while maintaining encapsulation.

This pattern is particularly useful when building reusable component libraries where different consumers need different state management behaviors.

## Core Concepts

### What is the State Reducer Pattern?

The State Reducer Pattern allows component consumers to:

1. **Intercept state changes** before they happen
2. **Modify or prevent** state transitions
3. **Add custom logic** to state management
4. **Override default behavior** without forking the component

Think of it as middleware for component state - every state change passes through a reducer function that the consumer can customize.

### When to Use This Pattern

Use the State Reducer Pattern when:

- Building component libraries that need maximum flexibility
- Users need to override default state behavior
- Complex state logic needs to be customizable
- You want to prevent certain state transitions
- Multiple consumers need different behaviors from the same component

## Basic Implementation

### Simple Counter with State Reducer

```javascript
import { useReducer } from 'react';

// Action types
const actionTypes = {
  INCREMENT: 'INCREMENT',
  DECREMENT: 'DECREMENT',
  RESET: 'RESET',
};

// Default reducer
function defaultReducer(state, action) {
  switch (action.type) {
    case actionTypes.INCREMENT:
      return { count: state.count + 1 };
    case actionTypes.DECREMENT:
      return { count: Math.max(0, state.count - 1) };
    case actionTypes.RESET:
      return { count: 0 };
    default:
      return state;
  }
}

// Custom hook with state reducer pattern
function useCounter({
  initialCount = 0,
  reducer = defaultReducer,
} = {}) {
  const [state, dispatch] = useReducer(reducer, {
    count: initialCount,
  });

  const increment = () => dispatch({ type: actionTypes.INCREMENT });
  const decrement = () => dispatch({ type: actionTypes.DECREMENT });
  const reset = () => dispatch({ type: actionTypes.RESET });

  return {
    count: state.count,
    increment,
    decrement,
    reset,
  };
}

// Usage: Default behavior
function BasicCounter() {
  const { count, increment, decrement, reset } = useCounter();

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}

// Usage: Custom behavior - max value
function LimitedCounter() {
  const customReducer = (state, action) => {
    const changes = defaultReducer(state, action);
    // Prevent count from going above 10
    return {
      ...changes,
      count: Math.min(changes.count, 10),
    };
  };

  const { count, increment, decrement, reset } = useCounter({
    reducer: customReducer,
  });

  return (
    <div>
      <p>Count (max 10): {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

## Advanced Examples

### Toggle Component with State Reducer

```javascript
const toggleActionTypes = {
  TOGGLE: 'TOGGLE',
  ON: 'ON',
  OFF: 'OFF',
};

function toggleReducer(state, action) {
  switch (action.type) {
    case toggleActionTypes.TOGGLE:
      return { on: !state.on };
    case toggleActionTypes.ON:
      return { on: true };
    case toggleActionTypes.OFF:
      return { on: false };
    default:
      return state;
  }
}

function useToggle({
  initialOn = false,
  reducer = toggleReducer,
  onChange,
} = {}) {
  const [state, dispatch] = useReducer(reducer, { on: initialOn });

  const toggle = () => {
    dispatch({ type: toggleActionTypes.TOGGLE });
    onChange?.(state.on);
  };

  const setOn = () => {
    dispatch({ type: toggleActionTypes.ON });
    onChange?.(true);
  };

  const setOff = () => {
    dispatch({ type: toggleActionTypes.OFF });
    onChange?.(false);
  };

  return {
    on: state.on,
    toggle,
    setOn,
    setOff,
  };
}

// Usage: Prevent toggling off after certain time
function TimedToggle() {
  const [lockedOn, setLockedOn] = React.useState(false);

  const customReducer = (state, action) => {
    // If locked on, prevent turning off
    if (lockedOn && action.type === toggleActionTypes.OFF) {
      return state;
    }
    return toggleReducer(state, action);
  };

  const { on, toggle, setOn, setOff } = useToggle({
    reducer: customReducer,
    onChange: (isOn) => {
      if (isOn) {
        // Lock it on for 3 seconds
        setLockedOn(true);
        setTimeout(() => setLockedOn(false), 3000);
      }
    },
  });

  return (
    <div>
      <p>Status: {on ? 'ON' : 'OFF'}</p>
      <p>{lockedOn && 'Locked for 3 seconds!'}</p>
      <button onClick={toggle}>Toggle</button>
      <button onClick={setOn}>Set On</button>
      <button onClick={setOff}>Set Off</button>
    </div>
  );
}
```

### Dropdown with State Reducer

```javascript
const dropdownActionTypes = {
  OPEN: 'OPEN',
  CLOSE: 'CLOSE',
  TOGGLE: 'TOGGLE',
  SELECT_ITEM: 'SELECT_ITEM',
  HIGHLIGHT_ITEM: 'HIGHLIGHT_ITEM',
};

function dropdownReducer(state, action) {
  switch (action.type) {
    case dropdownActionTypes.OPEN:
      return { ...state, isOpen: true };
    case dropdownActionTypes.CLOSE:
      return { ...state, isOpen: false, highlightedIndex: -1 };
    case dropdownActionTypes.TOGGLE:
      return { ...state, isOpen: !state.isOpen };
    case dropdownActionTypes.SELECT_ITEM:
      return {
        ...state,
        selectedItem: action.item,
        isOpen: false,
        highlightedIndex: -1,
      };
    case dropdownActionTypes.HIGHLIGHT_ITEM:
      return { ...state, highlightedIndex: action.index };
    default:
      return state;
  }
}

function useDropdown({
  items = [],
  initialSelectedItem = null,
  reducer = dropdownReducer,
  onSelect,
} = {}) {
  const [state, dispatch] = useReducer(reducer, {
    isOpen: false,
    selectedItem: initialSelectedItem,
    highlightedIndex: -1,
  });

  const open = () => dispatch({ type: dropdownActionTypes.OPEN });
  const close = () => dispatch({ type: dropdownActionTypes.CLOSE });
  const toggle = () => dispatch({ type: dropdownActionTypes.TOGGLE });

  const selectItem = (item) => {
    dispatch({ type: dropdownActionTypes.SELECT_ITEM, item });
    onSelect?.(item);
  };

  const highlightItem = (index) => {
    dispatch({ type: dropdownActionTypes.HIGHLIGHT_ITEM, index });
  };

  const handleKeyDown = (e) => {
    if (!state.isOpen && e.key !== 'Enter' && e.key !== ' ') return;

    switch (e.key) {
      case 'Enter':
      case ' ':
        if (state.isOpen && state.highlightedIndex >= 0) {
          selectItem(items[state.highlightedIndex]);
        } else {
          toggle();
        }
        break;
      case 'Escape':
        close();
        break;
      case 'ArrowDown':
        e.preventDefault();
        if (!state.isOpen) {
          open();
        } else {
          highlightItem(
            Math.min(state.highlightedIndex + 1, items.length - 1)
          );
        }
        break;
      case 'ArrowUp':
        e.preventDefault();
        highlightItem(Math.max(state.highlightedIndex - 1, 0));
        break;
    }
  };

  return {
    isOpen: state.isOpen,
    selectedItem: state.selectedItem,
    highlightedIndex: state.highlightedIndex,
    open,
    close,
    toggle,
    selectItem,
    highlightItem,
    handleKeyDown,
  };
}

// Usage: Prevent deselection
function Dropdown({ items }) {
  const customReducer = (state, action) => {
    // Prevent closing if nothing selected
    if (
      action.type === dropdownActionTypes.CLOSE &&
      !state.selectedItem
    ) {
      return state;
    }
    return dropdownReducer(state, action);
  };

  const {
    isOpen,
    selectedItem,
    highlightedIndex,
    toggle,
    selectItem,
    handleKeyDown,
  } = useDropdown({
    items,
    reducer: customReducer,
    onSelect: (item) => console.log('Selected:', item),
  });

  return (
    <div onKeyDown={handleKeyDown} tabIndex={0}>
      <button onClick={toggle}>
        {selectedItem || 'Select an item'} ▼
      </button>
      {isOpen && (
        <ul>
          {items.map((item, index) => (
            <li
              key={item}
              onClick={() => selectItem(item)}
              style={{
                background:
                  index === highlightedIndex ? '#e0e0e0' : 'white',
                cursor: 'pointer',
              }}
            >
              {item}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## Real-World Pattern: Form with Validation

```javascript
const formActionTypes = {
  SET_FIELD: 'SET_FIELD',
  SET_ERROR: 'SET_ERROR',
  SUBMIT: 'SUBMIT',
  SUBMIT_SUCCESS: 'SUBMIT_SUCCESS',
  SUBMIT_ERROR: 'SUBMIT_ERROR',
  RESET: 'RESET',
};

function formReducer(state, action) {
  switch (action.type) {
    case formActionTypes.SET_FIELD:
      return {
        ...state,
        values: {
          ...state.values,
          [action.field]: action.value,
        },
        errors: {
          ...state.errors,
          [action.field]: null,
        },
      };
    case formActionTypes.SET_ERROR:
      return {
        ...state,
        errors: {
          ...state.errors,
          ...action.errors,
        },
      };
    case formActionTypes.SUBMIT:
      return {
        ...state,
        isSubmitting: true,
        submitError: null,
      };
    case formActionTypes.SUBMIT_SUCCESS:
      return {
        ...state,
        isSubmitting: false,
        submitCount: state.submitCount + 1,
      };
    case formActionTypes.SUBMIT_ERROR:
      return {
        ...state,
        isSubmitting: false,
        submitError: action.error,
      };
    case formActionTypes.RESET:
      return {
        values: action.initialValues || {},
        errors: {},
        isSubmitting: false,
        submitError: null,
        submitCount: 0,
      };
    default:
      return state;
  }
}

function useForm({
  initialValues = {},
  validate,
  onSubmit,
  reducer = formReducer,
} = {}) {
  const [state, dispatch] = useReducer(reducer, {
    values: initialValues,
    errors: {},
    isSubmitting: false,
    submitError: null,
    submitCount: 0,
  });

  const setFieldValue = (field, value) => {
    dispatch({ type: formActionTypes.SET_FIELD, field, value });
  };

  const setErrors = (errors) => {
    dispatch({ type: formActionTypes.SET_ERROR, errors });
  };

  const handleSubmit = async (e) => {
    e?.preventDefault();

    // Validate
    if (validate) {
      const errors = validate(state.values);
      if (Object.keys(errors).length > 0) {
        setErrors(errors);
        return;
      }
    }

    dispatch({ type: formActionTypes.SUBMIT });

    try {
      await onSubmit?.(state.values);
      dispatch({ type: formActionTypes.SUBMIT_SUCCESS });
    } catch (error) {
      dispatch({
        type: formActionTypes.SUBMIT_ERROR,
        error: error.message,
      });
    }
  };

  const reset = (newInitialValues) => {
    dispatch({
      type: formActionTypes.RESET,
      initialValues: newInitialValues || initialValues,
    });
  };

  return {
    values: state.values,
    errors: state.errors,
    isSubmitting: state.isSubmitting,
    submitError: state.submitError,
    submitCount: state.submitCount,
    setFieldValue,
    setErrors,
    handleSubmit,
    reset,
  };
}

// Usage: Add confirmation on submit
function ConfirmationForm() {
  const customReducer = (state, action) => {
    // Prevent submit if value hasn't changed
    if (action.type === formActionTypes.SUBMIT) {
      if (state.submitCount > 0 && !state.values.confirmed) {
        return {
          ...state,
          errors: {
            ...state.errors,
            confirmed: 'Please confirm your changes',
          },
        };
      }
    }
    return formReducer(state, action);
  };

  const {
    values,
    errors,
    isSubmitting,
    setFieldValue,
    handleSubmit,
  } = useForm({
    initialValues: { email: '', confirmed: false },
    validate: (values) => {
      const errors = {};
      if (!values.email) errors.email = 'Required';
      if (values.email && !values.email.includes('@')) {
        errors.email = 'Invalid email';
      }
      return errors;
    },
    onSubmit: async (values) => {
      await new Promise((resolve) => setTimeout(resolve, 1000));
      console.log('Submitted:', values);
    },
    reducer: customReducer,
  });

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          type="email"
          value={values.email}
          onChange={(e) => setFieldValue('email', e.target.value)}
          placeholder="Email"
        />
        {errors.email && <span>{errors.email}</span>}
      </div>

      <div>
        <label>
          <input
            type="checkbox"
            checked={values.confirmed}
            onChange={(e) =>
              setFieldValue('confirmed', e.target.checked)
            }
          />
          I confirm this change
        </label>
        {errors.confirmed && <span>{errors.confirmed}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

## Integration with Other Patterns

### State Reducer + Compound Components

```javascript
const TabsContext = React.createContext();

const tabsActionTypes = {
  SELECT_TAB: 'SELECT_TAB',
  NEXT_TAB: 'NEXT_TAB',
  PREV_TAB: 'PREV_TAB',
};

function tabsReducer(state, action) {
  switch (action.type) {
    case tabsActionTypes.SELECT_TAB:
      return { ...state, activeIndex: action.index };
    case tabsActionTypes.NEXT_TAB:
      return {
        ...state,
        activeIndex: Math.min(state.activeIndex + 1, state.tabCount - 1),
      };
    case tabsActionTypes.PREV_TAB:
      return {
        ...state,
        activeIndex: Math.max(state.activeIndex - 1, 0),
      };
    default:
      return state;
  }
}

function Tabs({ children, defaultIndex = 0, reducer = tabsReducer }) {
  const tabCount = React.Children.count(children);
  const [state, dispatch] = useReducer(reducer, {
    activeIndex: defaultIndex,
    tabCount,
  });

  const selectTab = (index) => {
    dispatch({ type: tabsActionTypes.SELECT_TAB, index });
  };

  const nextTab = () => {
    dispatch({ type: tabsActionTypes.NEXT_TAB });
  };

  const prevTab = () => {
    dispatch({ type: tabsActionTypes.PREV_TAB });
  };

  return (
    <TabsContext.Provider
      value={{
        activeIndex: state.activeIndex,
        selectTab,
        nextTab,
        prevTab,
      }}
    >
      <div>{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ index, children }) {
  const { activeIndex, selectTab } = React.useContext(TabsContext);
  const isActive = index === activeIndex;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      onClick={() => selectTab(index)}
      style={{
        fontWeight: isActive ? 'bold' : 'normal',
      }}
    >
      {children}
    </button>
  );
}

function TabPanels({ children }) {
  const { activeIndex } = React.useContext(TabsContext);
  return <div>{React.Children.toArray(children)[activeIndex]}</div>;
}

function TabPanel({ children }) {
  return <div role="tabpanel">{children}</div>;
}

// Usage with custom reducer
function CustomTabs() {
  // Prevent going to certain tabs
  const customReducer = (state, action) => {
    if (
      action.type === tabsActionTypes.SELECT_TAB &&
      action.index === 2
    ) {
      // Skip tab 2
      return state;
    }
    return tabsReducer(state, action);
  };

  return (
    <Tabs reducer={customReducer}>
      <TabList>
        <Tab index={0}>Tab 1</Tab>
        <Tab index={1}>Tab 2</Tab>
        <Tab index={2}>Tab 3 (Disabled)</Tab>
      </TabList>
      <TabPanels>
        <TabPanel>Content 1</TabPanel>
        <TabPanel>Content 2</TabPanel>
        <TabPanel>Content 3</TabPanel>
      </TabPanels>
    </Tabs>
  );
}
```

## Best Practices

### 1. Export Action Types

```javascript
// Always export action types for consumers
export const counterActionTypes = {
  INCREMENT: 'INCREMENT',
  DECREMENT: 'DECREMENT',
  RESET: 'RESET',
};

// Consumers can use them in their reducers
import { counterActionTypes } from './useCounter';

function customReducer(state, action) {
  if (action.type === counterActionTypes.INCREMENT) {
    // Custom logic
  }
  return defaultReducer(state, action);
}
```

### 2. Call Default Reducer First

```javascript
// Good: Call default reducer, then modify
function customReducer(state, action) {
  const changes = defaultReducer(state, action);
  return {
    ...changes,
    // Add custom modifications
    count: Math.min(changes.count, 100),
  };
}

// Also good: Modify specific actions
function customReducer(state, action) {
  if (action.type === actionTypes.INCREMENT) {
    // Custom logic for specific action
    return { count: state.count + 2 };
  }
  // Fall back to default for others
  return defaultReducer(state, action);
}
```

### 3. Provide Type Safety

```typescript
type State = {
  count: number;
};

type Action =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' };

type Reducer = (state: State, action: Action) => State;

function useCounter({
  initialCount = 0,
  reducer = defaultReducer,
}: {
  initialCount?: number;
  reducer?: Reducer;
} = {}) {
  // Implementation
}
```

### 4. Document State Changes

```javascript
/**
 * Default reducer for counter
 * @param {Object} state - Current state
 * @param {Object} action - Action to perform
 * @returns {Object} New state
 *
 * Handles:
 * - INCREMENT: Increases count by 1
 * - DECREMENT: Decreases count by 1 (min 0)
 * - RESET: Sets count to 0
 */
function defaultReducer(state, action) {
  // Implementation
}
```

## Common Mistakes

### 1. Mutating State

```javascript
// Bad: Mutating state
function badReducer(state, action) {
  state.count++; // Mutation!
  return state;
}

// Good: Return new object
function goodReducer(state, action) {
  return { ...state, count: state.count + 1 };
}
```

### 2. Forgetting to Return State

```javascript
// Bad: No return for default case
function badReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    // Missing default case!
  }
}

// Good: Always return state
function goodReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    default:
      return state; // Important!
  }
}
```

### 3. Not Exposing Action Types

```javascript
// Bad: Action types hidden in component
function useCounter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  // No way for consumers to know action types
}

// Good: Export action types
export const actionTypes = { INCREMENT: 'INCREMENT' };
export function useCounter({ reducer } = {}) {
  // Now consumers can reference actionTypes
}
```

## Performance Considerations

### Memoize Reducer

```javascript
function MyComponent() {
  // Bad: New reducer function on every render
  const customReducer = (state, action) => {
    return defaultReducer(state, action);
  };

  const { count } = useCounter({ reducer: customReducer });

  // Good: Memoize or define outside
  const memoizedReducer = React.useCallback(
    (state, action) => defaultReducer(state, action),
    []
  );

  const { count: count2 } = useCounter({ reducer: memoizedReducer });
}

// Best: Define outside component
const customReducer = (state, action) => {
  return defaultReducer(state, action);
};

function MyComponent() {
  const { count } = useCounter({ reducer: customReducer });
}
```

## Testing

```javascript
import { renderHook, act } from '@testing-library/react';

describe('useCounter with state reducer', () => {
  test('uses default reducer by default', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  test('allows custom reducer', () => {
    const customReducer = (state, action) => {
      const changes = defaultReducer(state, action);
      // Max value of 5
      return { ...changes, count: Math.min(changes.count, 5) };
    };

    const { result } = renderHook(() =>
      useCounter({ reducer: customReducer })
    );

    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.increment();
      result.current.increment();
      result.current.increment();
      result.current.increment(); // Should stop at 5
    });

    expect(result.current.count).toBe(5);
  });

  test('custom reducer can prevent actions', () => {
    const customReducer = (state, action) => {
      // Prevent decrement
      if (action.type === actionTypes.DECREMENT) {
        return state;
      }
      return defaultReducer(state, action);
    };

    const { result } = renderHook(() =>
      useCounter({ initialCount: 5, reducer: customReducer })
    );

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(5); // Unchanged
  });
});
```

## Key Takeaways

1. **Inversion of Control**: State Reducer Pattern gives consumers complete control over state management
2. **Export Action Types**: Always export action types so consumers can reference them
3. **Call Default First**: Custom reducers should typically call the default reducer first
4. **Immutability**: Always return new state objects, never mutate
5. **Type Safety**: Use TypeScript or JSDoc for better developer experience
6. **Performance**: Memoize or define reducers outside components
7. **Testing**: Test both default and custom reducer behaviors

## Additional Resources

- [Kent C. Dodds - State Reducer Pattern](https://kentcdodds.com/blog/the-state-reducer-pattern-with-react-hooks)
- [Downshift Documentation](https://www.downshift-js.com/)
- [Advanced React Patterns](https://javascript.plainenglish.io/5-advanced-react-patterns-a6b7624267a6)
- [React useReducer Hook](https://react.dev/reference/react/useReducer)
