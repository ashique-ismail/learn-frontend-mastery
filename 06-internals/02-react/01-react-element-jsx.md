# React Element & JSX

## The Idea

**In plain English:** JSX is a shorthand way to describe what you want to appear on screen, and React turns that description into a small JavaScript object called a "React element" — think of it as a recipe card that tells React what to cook, without actually cooking it yet.

**Real-world analogy:** Imagine ordering food at a restaurant. You write your order on a ticket (JSX), the kitchen receives a printed receipt describing the dish (the React element object), and only then does the chef actually prepare the real meal (the DOM node on screen).

- The order ticket you write = JSX
- The printed kitchen receipt = the React element (a plain JavaScript object)
- The actual meal served = the real DOM node rendered in the browser

---

## Overview

JSX is syntactic sugar that compiles to React.createElement() calls, which create plain JavaScript objects called React elements. Understanding this transformation, the structure of React elements, and how they differ from components and DOM elements is fundamental to understanding React's architecture.

This guide explores JSX compilation, React element structure, element properties, and the journey from JSX to actual DOM nodes.

## JSX to React.createElement

JSX transforms into function calls during compilation:

```javascript
// Example 1: Basic JSX transformation
// JSX:
const element = <div className="container">Hello</div>;

// Compiles to:
const element = React.createElement(
  'div',
  { className: 'container' },
  'Hello'
);

// Example 2: Nested elements
// JSX:
const nested = (
  <div>
    <h1>Title</h1>
    <p>Content</p>
  </div>
);

// Compiles to:
const nested = React.createElement(
  'div',
  null,
  React.createElement('h1', null, 'Title'),
  React.createElement('p', null, 'Content')
);

// Example 3: Component elements
// JSX:
const app = <App title="Hello" count={42} />;

// Compiles to:
const app = React.createElement(
  App,
  { title: 'Hello', count: 42 },
  null
);
```

### JSX Transform Evolution

```javascript
// Example 4: Classic transform (React 16 and earlier)
// Requires React in scope
import React from 'react';

function App() {
  return <div>Hello</div>;
  // Transforms to React.createElement
}

// Example 5: Automatic transform (React 17+)
// No React import needed
// import { jsx } from 'react/jsx-runtime'; (auto-imported)

function App() {
  return <div>Hello</div>;
  // Transforms to jsx('div', { children: 'Hello' })
}

// Automatic transform benefits:
// - No need to import React
// - Smaller bundle (tree-shakes unused React APIs)
// - Faster compile (simpler output)
```

## React Element Structure

React elements are plain JavaScript objects:

```javascript
// Example 6: React element structure
const element = React.createElement(
  'div',
  { className: 'container', id: 'main' },
  'Hello',
  <span>World</span>
);

// Results in object:
{
  $$typeof: Symbol.for('react.element'),
  type: 'div',
  key: null,
  ref: null,
  props: {
    className: 'container',
    id: 'main',
    children: ['Hello', {
      $$typeof: Symbol.for('react.element'),
      type: 'span',
      key: null,
      ref: null,
      props: {
        children: 'World'
      }
    }]
  },
  _owner: null,
  _store: {}
}

// Example 7: createElement implementation (simplified)
function createElement(type, config, ...children) {
  const props = {};
  
  let key = null;
  let ref = null;
  
  if (config != null) {
    if (config.ref !== undefined) {
      ref = config.ref;
    }
    if (config.key !== undefined) {
      key = '' + config.key;
    }
    
    // Copy props except key and ref
    for (const propName in config) {
      if (
        config.hasOwnProperty(propName) &&
        propName !== 'key' &&
        propName !== 'ref'
      ) {
        props[propName] = config[propName];
      }
    }
  }
  
  // Handle children
  if (children.length === 1) {
    props.children = children[0];
  } else if (children.length > 1) {
    props.children = children;
  }
  
  // Add default props
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (const propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }
  
  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type,
    key,
    ref,
    props,
    _owner: ReactCurrentOwner.current
  };
}
```

### The $$typeof Symbol

```javascript
// Example 8: Security with $$typeof
const REACT_ELEMENT_TYPE = Symbol.for('react.element');

function isValidElement(object) {
  return (
    typeof object === 'object' &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}

// Example 9: Why $$typeof exists (XSS protection)
// Malicious JSON from server:
const maliciousJSON = {
  type: 'div',
  props: {
    dangerouslySetInnerHTML: {
      __html: '<script>alert("XSS")</script>'
    }
  }
};

// Without $$typeof:
// React might render this as valid element → XSS attack

// With $$typeof:
// JSON.parse cannot create Symbols
// React rejects object without $$typeof
// Attack prevented

// Example 10: Creating elements safely
// WRONG: Creating element object manually
const unsafeElement = {
  type: 'div',
  props: { children: 'Hello' }
};

// React won't render this (no $$typeof)

// RIGHT: Use createElement or JSX
const safeElement = React.createElement('div', null, 'Hello');
// or
const safeJSX = <div>Hello</div>;
```

## Element Types

Different types of elements:

```javascript
// Example 11: Host elements (DOM)
const div = <div />;
// type: 'div' (string)

const span = <span />;
// type: 'span' (string)

// Example 12: Function component elements
function Welcome({ name }) {
  return <div>Hello {name}</div>;
}

const welcome = <Welcome name="World" />;
// type: Welcome (function reference)

// Example 13: Class component elements
class Greeting extends React.Component {
  render() {
    return <div>Hello {this.props.name}</div>;
  }
}

const greeting = <Greeting name="World" />;
// type: Greeting (class reference)

// Example 14: Fragment elements
const fragment = (
  <>
    <div>First</div>
    <div>Second</div>
  </>
);

// Compiles to:
React.createElement(
  React.Fragment,
  null,
  React.createElement('div', null, 'First'),
  React.createElement('div', null, 'Second')
);

// type: Symbol.for('react.fragment')

// Example 15: Special element types
const portal = ReactDOM.createPortal(
  <div>Portal content</div>,
  document.getElementById('portal-root')
);
// type: Symbol.for('react.portal')

const context = (
  <ThemeContext.Provider value="dark">
    <App />
  </ThemeContext.Provider>
);
// type: Symbol.for('react.provider')

const suspense = (
  <Suspense fallback={<Loading />}>
    <LazyComponent />
  </Suspense>
);
// type: Symbol.for('react.suspense')
```

## Props and Children

```javascript
// Example 16: Children handling
// Single child
<div>Hello</div>
// props.children = 'Hello'

// Multiple children
<div>
  <span>One</span>
  <span>Two</span>
</div>
// props.children = [element1, element2]

// No children
<div />
// props.children = undefined

// Example 17: Children prop vs JSX children
// These are equivalent:
<div children="Hello" />
<div>Hello</div>

// But JSX children take precedence:
<div children="Ignored">Rendered</div>
// Renders "Rendered", not "Ignored"

// Example 18: Children utilities
import { Children } from 'react';

function List({ children }) {
  // Count children (handles arrays, fragments, etc.)
  const count = Children.count(children);
  
  // Map over children
  const items = Children.map(children, (child, index) => {
    return <li key={index}>{child}</li>;
  });
  
  // Convert to array
  const array = Children.toArray(children);
  
  // Get only child (throws if multiple)
  const only = Children.only(children);
  
  return <ul>{items}</ul>;
}

// Example 19: Cloning elements
function WrapWithClass({ children }) {
  return Children.map(children, child => {
    if (!React.isValidElement(child)) {
      return child;
    }
    
    // Clone element with additional props
    return React.cloneElement(child, {
      className: `${child.props.className || ''} wrapped`
    });
  });
}

// Usage:
<WrapWithClass>
  <div>One</div>
  <div className="existing">Two</div>
</WrapWithClass>

// Renders:
// <div class="wrapped">One</div>
// <div class="existing wrapped">Two</div>
```

## Keys and Refs

```javascript
// Example 20: Keys are not props
function List() {
  const items = ['a', 'b', 'c'];
  
  return items.map(item => (
    <Item key={item} id={item} />
  ));
}

function Item({ id }) {
  // key is not in props
  console.log(id);    // 'a', 'b', or 'c'
  // console.log(key); // undefined
  
  return <div>{id}</div>;
}

// Example 21: Refs are not props
function Parent() {
  const ref = useRef();
  
  return <Child ref={ref} />;
}

const Child = forwardRef((props, ref) => {
  // ref is separate parameter, not in props
  console.log(props.ref); // undefined
  
  return <div ref={ref}>Child</div>;
});

// Example 22: Accessing key and ref (not recommended)
function debugElement(element) {
  // Access key through element object
  console.log('Key:', element.key);
  
  // Access ref through element object
  console.log('Ref:', element.ref);
  
  // These are on element, not element.props
}
```

## Element Immutability

React elements are immutable:

```javascript
// Example 23: Elements are immutable
const element = <div className="container">Hello</div>;

// WRONG: Can't mutate element
element.props.className = 'new-class'; // Error or ignored
element.props.children = 'Goodbye';    // Error or ignored

// RIGHT: Create new element
const newElement = <div className="new-class">Goodbye</div>;

// Example 24: Cloning for modification
const original = <div className="original">Content</div>;

// Clone with new props
const modified = React.cloneElement(original, {
  className: 'modified'
});

// original unchanged
console.log(original.props.className); // 'original'
console.log(modified.props.className); // 'modified'

// Example 25: Why immutability matters
function BrokenComponent() {
  const element = <div>Content</div>;
  
  // Later in code...
  element.props.children = 'Modified'; // Doesn't work
  
  // React cached element, won't see change
  return element; // Still renders "Content"
}
```

## From Element to DOM

The journey from React element to actual DOM:

```javascript
// Example 26: Rendering pipeline
// 1. JSX
<div className="app">
  <Header title="Hello" />
  <Main />
</div>

// 2. React Element
{
  type: 'div',
  props: {
    className: 'app',
    children: [
      {
        type: Header,
        props: { title: 'Hello' }
      },
      {
        type: Main,
        props: {}
      }
    ]
  }
}

// 3. Fiber Tree
// React creates fiber nodes for reconciliation
{
  tag: HostComponent, // div
  type: 'div',
  props: { className: 'app', children: [...] },
  child: /* fiber for Header */,
  sibling: null,
  return: /* parent fiber */,
  stateNode: /* actual DOM node */
}

// 4. DOM Node
<div class="app">
  <div>Header content</div>
  <div>Main content</div>
</div>

// Example 27: Component rendering process
function MyComponent({ value }) {
  // Component is called
  return <div>{value}</div>;
  // Returns React element
}

const element = <MyComponent value={42} />;
// Creates element with type: MyComponent

// During render:
// 1. React sees type is function
// 2. Calls MyComponent({ value: 42 })
// 3. Receives React element { type: 'div', props: { children: 42 } }
// 4. Creates fiber for div
// 5. Creates DOM node <div>42</div>
```

## Advanced Patterns

```javascript
// Example 28: Element factory
function createButton(text, onClick) {
  return React.createElement(
    'button',
    { onClick },
    text
  );
}

const button = createButton('Click me', () => console.log('Clicked'));

// Example 29: Dynamic element type
function DynamicComponent({ tag, children }) {
  return React.createElement(tag, null, children);
}

<DynamicComponent tag="h1">Heading</DynamicComponent>
// Renders <h1>Heading</h1>

<DynamicComponent tag="p">Paragraph</DynamicComponent>
// Renders <p>Paragraph</p>

// Example 30: Element inspection
function inspectElement(element) {
  if (!React.isValidElement(element)) {
    console.log('Not a valid element');
    return;
  }
  
  console.log('Type:', element.type);
  console.log('Props:', element.props);
  console.log('Key:', element.key);
  
  // Check element type
  if (typeof element.type === 'string') {
    console.log('Host element (DOM)');
  } else if (typeof element.type === 'function') {
    console.log('Component element');
  } else {
    console.log('Special element (Fragment, Portal, etc.)');
  }
}
```

## JSX Compilation Options

```javascript
// Example 31: JSX pragma (custom createElement)
/** @jsx h */
import { h } from 'preact';

function App() {
  return <div>Hello</div>;
}

// Compiles to:
// h('div', null, 'Hello')

// Example 32: JSX Fragment pragma
/** @jsxFrag Fragment */
import { Fragment } from 'react';

function App() {
  return (
    <>
      <div>One</div>
      <div>Two</div>
    </>
  );
}

// Compiles to:
// React.createElement(Fragment, null, ...)

// Example 33: TypeScript JSX configuration
// tsconfig.json
{
  "compilerOptions": {
    "jsx": "react-jsx",        // Automatic transform
    // or
    "jsx": "react",            // Classic transform
    "jsxImportSource": "react" // Where to import jsx from
  }
}
```

## Common Misconceptions

1. **"JSX is required to use React"** - False. JSX compiles to createElement calls, which you can write directly. JSX is just syntactic sugar.

2. **"JSX is HTML in JavaScript"** - False. JSX is XML-like syntax that compiles to JavaScript. It's similar to HTML but has differences (className vs class, htmlFor vs for, etc.).

3. **"React elements are DOM elements"** - False. React elements are plain JavaScript objects describing what to render. React creates actual DOM elements from these descriptions.

4. **"You can modify element props"** - False. React elements are immutable. You must create new elements or use cloneElement to "modify" them.

5. **"Components are the same as elements"** - False. Components are functions or classes that return elements. Elements are objects describing what to render.

## Performance Implications

1. **Element Creation Cost** - Creating React elements is cheap—just object creation. Don't worry about creating many elements.

2. **Cloning Elements** - React.cloneElement creates a new object. In hot paths (like render loops), this adds overhead. Prefer composition over cloning.

3. **Children Array Allocation** - JSX with multiple children creates arrays. In performance-critical code, consider single-child patterns.

4. **Key Prop Overhead** - Keys are just strings/numbers. No significant performance cost, but necessary for correct reconciliation.

## Interview Questions

1. **Q: What is a React element?**
   A: A React element is a plain JavaScript object that describes what should appear on the screen. It has a type (string for DOM elements, function/class for components), props, key, ref, and $$typeof symbol. Elements are immutable and cheap to create.

2. **Q: How does JSX transform to JavaScript?**
   A: JSX transforms to React.createElement() calls (classic transform) or jsx() calls (automatic transform). These functions create React element objects with type, props, and children.

3. **Q: What is the $$typeof property and why does it exist?**
   A: $$typeof is a Symbol that identifies valid React elements. It exists for security—JSON.parse can't create Symbols, so React can reject potentially malicious objects from server responses, preventing XSS attacks.

4. **Q: What's the difference between a React element and a component?**
   A: A component is a function or class that returns elements. An element is a plain object describing what to render. Components are reusable blueprints; elements are instances of those blueprints.

5. **Q: Why are keys not accessible in props?**
   A: Keys are used internally by React for reconciliation, not by the component itself. React extracts key during element creation and uses it to track element identity across renders. Keeping it separate from props makes this clear.

6. **Q: Can you modify a React element after creation?**
   A: No. React elements are immutable. You must create a new element or use React.cloneElement to create a modified copy. Immutability enables efficient change detection and safe concurrent rendering.

7. **Q: How do React elements become DOM nodes?**
   A: React creates fiber nodes from elements during reconciliation. For host elements (like 'div'), React creates actual DOM nodes and stores references in fiber.stateNode. For components, React calls them to get more elements, recursively processing until reaching host elements.

8. **Q: What's the difference between createElement and cloneElement?**
   A: createElement creates a new element from scratch given type, props, and children. cloneElement creates a new element based on an existing one, merging in new props and children while preserving the original type and key.

## Key Takeaways

1. JSX transforms to React.createElement or jsx function calls
2. React elements are immutable plain JavaScript objects
3. Elements have type, props, key, ref, and $$typeof symbol
4. $$typeof prevents XSS attacks from malicious JSON
5. Elements are cheap to create; components are expensive to render
6. Keys and refs are extracted from props during element creation
7. Children can be single values, arrays, or undefined
8. Elements must be created through official APIs for security

## Resources

- [JSX In Depth](https://react.dev/learn/writing-markup-with-jsx)
- [React.createElement API](https://react.dev/reference/react/createElement)
- [Introducing JSX](https://legacy.reactjs.org/docs/introducing-jsx.html)
- [JSX Transform RFC](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md)
- [React Element Source Code](https://github.com/facebook/react/blob/main/packages/react/src/ReactElement.js)
