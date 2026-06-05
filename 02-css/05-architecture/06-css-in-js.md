# CSS-in-JS

## The Idea

**In plain English:** CSS-in-JS is a technique where you write the visual styling rules for a webpage (colors, sizes, fonts) directly inside JavaScript code, instead of in a separate stylesheet file. Think of "CSS" as the rulebook for how things look, and "JS" (JavaScript) as the programming language that controls what the page does — CSS-in-JS merges the two into one place.

**Real-world analogy:** Imagine a costume designer on a film set who travels with each actor instead of keeping all costumes in a central wardrobe. Each actor carries their own outfit instructions tailored specifically to them, and the costume can even change on the spot depending on the scene being filmed.

- The actor = a UI component (a button, a card, a menu)
- The outfit instructions the actor carries = the CSS styles written inside the JavaScript component file
- The scene being filmed = the runtime data or props that can change how the styles look on the fly

---

## Overview

CSS-in-JS is an approach where CSS is composed using JavaScript instead of defined in external stylesheets. Styles are created, managed, and applied dynamically at runtime or build-time, enabling powerful features like dynamic styling, type safety, and component-level encapsulation.

## Philosophy

### Traditional Separation

```jsx
// Component.jsx
import './Component.css';

function Component() {
  return <div className="card">Content</div>;
}
```

```css
/* Component.css */
.card {
  background: white;
  padding: 20px;
}
```

### CSS-in-JS Approach

```jsx
import styled from 'styled-components';

const Card = styled.div`
  background: white;
  padding: 20px;
`;

function Component() {
  return <Card>Content</Card>;
}
```

### Core Benefits

1. **Colocation**: Styles live with components
2. **Dynamic**: Styles can use JavaScript logic
3. **Scoped**: No global namespace pollution
4. **Type-safe**: TypeScript support
5. **Dead Code Elimination**: Unused styles removed automatically

## Runtime CSS-in-JS

Styles are generated and injected at runtime in the browser.

### Styled-Components

Most popular CSS-in-JS library.

#### Basic Usage

```jsx
import styled from 'styled-components';

// Simple styled component
const Button = styled.button`
  background: #007bff;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 600;
  
  &:hover {
    background: #0056b3;
  }
  
  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

function App() {
  return (
    <>
      <Button>Primary Button</Button>
      <Button disabled>Disabled Button</Button>
    </>
  );
}
```

#### Props-Based Styling

```jsx
const Button = styled.button`
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  font-weight: 600;
  cursor: pointer;
  
  /* Dynamic based on props */
  background: ${props => {
    switch(props.variant) {
      case 'primary': return '#007bff';
      case 'secondary': return '#6c757d';
      case 'danger': return '#dc3545';
      default: return '#007bff';
    }
  }};
  
  color: ${props => props.variant === 'secondary' ? '#333' : 'white'};
  
  /* Size variations */
  padding: ${props => {
    switch(props.size) {
      case 'small': return '5px 10px';
      case 'large': return '15px 30px';
      default: return '10px 20px';
    }
  }};
  
  /* Conditional styles */
  ${props => props.fullWidth && `
    width: 100%;
    display: block;
  `}
  
  ${props => props.outlined && `
    background: transparent;
    border: 2px solid currentColor;
    color: ${props.variant === 'primary' ? '#007bff' : '#6c757d'};
  `}
`;

// Usage
<Button variant="primary">Primary</Button>
<Button variant="danger" size="large">Large Danger</Button>
<Button variant="secondary" outlined fullWidth>
  Full Width Outlined
</Button>
```

#### Extending Styles

```jsx
const Button = styled.button`
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
`;

const PrimaryButton = styled(Button)`
  background: #007bff;
  color: white;
`;

const DangerButton = styled(Button)`
  background: #dc3545;
  color: white;
`;

// Extend third-party components
import { Link } from 'react-router-dom';

const StyledLink = styled(Link)`
  color: #007bff;
  text-decoration: none;
  
  &:hover {
    text-decoration: underline;
  }
`;
```

#### Theming

```jsx
import { ThemeProvider, createGlobalStyle } from 'styled-components';

const lightTheme = {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff',
    text: '#333333',
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px',
  },
  borderRadius: '4px',
};

const darkTheme = {
  colors: {
    primary: '#375a7f',
    secondary: '#444444',
    background: '#1a1a1a',
    text: '#ffffff',
  },
  spacing: lightTheme.spacing,
  borderRadius: lightTheme.borderRadius,
};

const GlobalStyle = createGlobalStyle`
  body {
    background: ${props => props.theme.colors.background};
    color: ${props => props.theme.colors.text};
    margin: 0;
    font-family: system-ui, sans-serif;
  }
`;

const Button = styled.button`
  background: ${props => props.theme.colors.primary};
  color: white;
  padding: ${props => props.theme.spacing.medium};
  border: none;
  border-radius: ${props => props.theme.borderRadius};
`;

function App() {
  const [theme, setTheme] = useState('light');
  const currentTheme = theme === 'light' ? lightTheme : darkTheme;
  
  return (
    <ThemeProvider theme={currentTheme}>
      <GlobalStyle />
      <Button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </Button>
    </ThemeProvider>
  );
}
```

#### Animations

```jsx
import styled, { keyframes } from 'styled-components';

const fadeIn = keyframes`
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
`;

const spin = keyframes`
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
`;

const Card = styled.div`
  animation: ${fadeIn} 0.3s ease-out;
`;

const Spinner = styled.div`
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #007bff;
  border-radius: 50%;
  animation: ${spin} 1s linear infinite;
`;
```

#### Attrs Method

Set default attributes:

```jsx
const Input = styled.input.attrs(props => ({
  type: props.type || 'text',
  placeholder: props.placeholder || 'Enter text...',
}))`
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
  
  &:focus {
    outline: none;
    border-color: #007bff;
  }
`;

// Usage
<Input />  {/* type="text" placeholder="Enter text..." */}
<Input type="email" placeholder="Enter email..." />
```

### Emotion

Alternative to styled-components with similar API.

#### Styled API

```jsx
import styled from '@emotion/styled';

const Button = styled.button`
  background: #007bff;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  
  &:hover {
    background: #0056b3;
  }
`;
```

#### CSS Prop

```jsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

function Button() {
  return (
    <button
      css={css`
        background: #007bff;
        color: white;
        padding: 10px 20px;
        border: none;
        border-radius: 4px;
        
        &:hover {
          background: #0056b3;
        }
      `}
    >
      Click me
    </button>
  );
}
```

#### Object Styles

```jsx
import { css } from '@emotion/react';

const buttonStyles = css({
  background: '#007bff',
  color: 'white',
  padding: '10px 20px',
  border: 'none',
  borderRadius: '4px',
  '&:hover': {
    background: '#0056b3',
  },
});

function Button() {
  return <button css={buttonStyles}>Click me</button>;
}
```

#### Composition

```jsx
import { css } from '@emotion/react';

const baseButton = css`
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
`;

const primaryButton = css`
  ${baseButton}
  background: #007bff;
  color: white;
`;

const secondaryButton = css`
  ${baseButton}
  background: #6c757d;
  color: white;
`;
```

### Other Runtime Libraries

#### JSS

```jsx
import { createUseStyles } from 'react-jss';

const useStyles = createUseStyles({
  button: {
    background: '#007bff',
    color: 'white',
    padding: '10px 20px',
    border: 'none',
    borderRadius: '4px',
    '&:hover': {
      background: '#0056b3',
    },
  },
  primary: {
    background: '#007bff',
  },
  secondary: {
    background: '#6c757d',
  },
});

function Button({ variant = 'primary' }) {
  const classes = useStyles();
  return (
    <button className={`${classes.button} ${classes[variant]}`}>
      Click me
    </button>
  );
}
```

#### Aphrodite

```jsx
import { StyleSheet, css } from 'aphrodite';

const styles = StyleSheet.create({
  button: {
    background: '#007bff',
    color: 'white',
    padding: '10px 20px',
    border: 'none',
    borderRadius: '4px',
    ':hover': {
      background: '#0056b3',
    },
  },
});

function Button() {
  return (
    <button className={css(styles.button)}>
      Click me
    </button>
  );
}
```

## Zero-Runtime CSS-in-JS

Styles are extracted at build-time and served as static CSS, eliminating runtime overhead.

### Linaria

```jsx
import { styled } from '@linaria/react';

// Extracted to CSS at build time
const Button = styled.button`
  background: #007bff;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  
  &:hover {
    background: #0056b3;
  }
`;

// Dynamic values via CSS variables
const DynamicButton = styled.button`
  background: var(--button-bg);
  color: var(--button-color);
  padding: 10px 20px;
`;

function App() {
  return (
    <>
      <Button>Static</Button>
      <DynamicButton 
        style={{
          '--button-bg': '#dc3545',
          '--button-color': 'white',
        }}
      >
        Dynamic
      </DynamicButton>
    </>
  );
}
```

### Vanilla Extract

TypeScript-first zero-runtime CSS-in-JS.

```typescript
// Button.css.ts
import { style } from '@vanilla-extract/css';

export const button = style({
  background: '#007bff',
  color: 'white',
  padding: '10px 20px',
  border: 'none',
  borderRadius: '4px',
  selectors: {
    '&:hover': {
      background: '#0056b3',
    },
  },
});

export const primary = style({
  background: '#007bff',
});

export const secondary = style({
  background: '#6c757d',
});
```

```tsx
// Button.tsx
import * as styles from './Button.css';

function Button({ variant = 'primary' }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      Click me
    </button>
  );
}
```

#### Theming

```typescript
// theme.css.ts
import { createTheme, createThemeContract } from '@vanilla-extract/css';

export const vars = createThemeContract({
  colors: {
    primary: null,
    secondary: null,
    background: null,
    text: null,
  },
  spacing: {
    small: null,
    medium: null,
    large: null,
  },
});

export const lightTheme = createTheme(vars, {
  colors: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff',
    text: '#333333',
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px',
  },
});

export const darkTheme = createTheme(vars, {
  colors: {
    primary: '#375a7f',
    secondary: '#444444',
    background: '#1a1a1a',
    text: '#ffffff',
  },
  spacing: {
    small: '8px',
    medium: '16px',
    large: '24px',
  },
});
```

```typescript
// Button.css.ts
import { style } from '@vanilla-extract/css';
import { vars } from './theme.css';

export const button = style({
  background: vars.colors.primary,
  color: 'white',
  padding: vars.spacing.medium,
});
```

### Compiled

```jsx
import { styled } from '@compiled/react';

const Button = styled.button`
  background: #007bff;
  color: white;
  padding: 10px 20px;
  
  &:hover {
    background: #0056b3;
  }
`;

// With props (limited - extracted at build time)
const DynamicButton = styled.button`
  background: ${props => props.primary ? '#007bff' : '#6c757d'};
  color: white;
`;
```

### Comparison Table

| Library | Runtime | Bundle Impact | Dynamic Styling | Learning Curve |
|---------|---------|---------------|-----------------|----------------|
| styled-components | Yes | Medium-High | Full | Medium |
| Emotion | Yes | Medium | Full | Medium |
| JSS | Yes | Medium-High | Full | High |
| Linaria | No | Low | Limited (CSS vars) | Medium |
| Vanilla Extract | No | Low | Limited (CSS vars) | Medium-High |
| Compiled | No | Low | Limited | Medium |

## Pros and Cons

### Advantages

#### 1. Dynamic Styling

```jsx
const Card = styled.div`
  background: ${props => props.urgent ? '#dc3545' : 'white'};
  border: 2px solid ${props => props.urgent ? '#dc3545' : '#ddd'};
  padding: ${props => props.compact ? '10px' : '20px'};
  
  /* Complex logic */
  ${props => {
    if (props.status === 'success') return 'border-color: #28a745;';
    if (props.status === 'warning') return 'border-color: #ffc107;';
    if (props.status === 'error') return 'border-color: #dc3545;';
    return '';
  }}
`;

<Card urgent>Urgent card</Card>
<Card status="success" compact>Success card</Card>
```

#### 2. Colocated Styles

```jsx
// Everything in one file
const Card = styled.div`
  background: white;
  padding: 20px;
`;

const CardHeader = styled.div`
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
`;

function Card() {
  return (
    <Card>
      <CardHeader>
        <h2>Title</h2>
      </CardHeader>
      <p>Content</p>
    </Card>
  );
}
```

#### 3. Type Safety

```tsx
import styled from 'styled-components';

interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  fullWidth?: boolean;
}

const Button = styled.button<ButtonProps>`
  padding: ${props => {
    switch(props.size) {
      case 'small': return '5px 10px';
      case 'large': return '15px 30px';
      default: return '10px 20px';
    }
  }};
  
  ${props => props.fullWidth && 'width: 100%;'}
`;

// TypeScript enforces prop types
<Button variant="primary" size="large">Click</Button>
<Button variant="invalid" size="huge">  {/* TypeScript error */}
```

#### 4. No Naming Conflicts

```jsx
// Each component has unique generated class names
const Button = styled.button`
  background: blue;
`;
// Generates: .sc-bdVaJa { background: blue; }

// No collision with other Button components
```

#### 5. Automatic Critical CSS

```jsx
// Only styles for rendered components are injected
{showModal && <Modal />}  // Modal styles injected only when shown
```

#### 6. JavaScript Ecosystem

```jsx
import { lighten, darken } from 'polished';

const Button = styled.button`
  background: #007bff;
  
  &:hover {
    background: ${darken(0.1, '#007bff')};
  }
  
  &:active {
    background: ${darken(0.2, '#007bff')};
  }
`;

// Use npm packages for color manipulation, etc.
```

### Disadvantages

#### 1. Bundle Size

Runtime libraries add significant weight:

```
styled-components: ~16KB gzipped
Emotion: ~11KB gzipped
JSS: ~15KB gzipped

vs

CSS Modules: 0KB (static CSS)
Tailwind: ~5KB gzipped (purged)
```

#### 2. Runtime Performance Cost

```jsx
// Style generation happens at runtime
const Button = styled.button`
  background: ${props => props.color};
  padding: ${props => props.size}px;
`;

// On every render:
// 1. Parse template literal
// 2. Generate CSS
// 3. Inject into DOM
// 4. Apply class name
```

Especially bad with many dynamic styles:

```jsx
// Poor performance
{items.map(item => (
  <Card 
    color={item.color} 
    size={item.size}
    margin={item.margin}
  />
))}
// Each card generates unique styles at runtime
```

#### 3. SSR Complexity

```jsx
// Server-side rendering requires setup
import { ServerStyleSheet } from 'styled-components';

const sheet = new ServerStyleSheet();
try {
  const html = renderToString(sheet.collectStyles(<App />));
  const styleTags = sheet.getStyleTags();
  // Must inject styleTags into HTML
} catch (error) {
  console.error(error);
} finally {
  sheet.seal();
}
```

#### 4. DevTools Harder to Use

```html
<!-- Traditional CSS -->
<button class="btn-primary">Click</button>

<!-- CSS-in-JS -->
<button class="sc-bdVaJa eDVcSA">Click</button>
<!-- Hard to identify what component this is -->
```

#### 5. Can't Use Standard CSS Tools

- Can't use PostCSS plugins easily
- Can't use CSS linters (Stylelint)
- Can't use standard CSS build tools

#### 6. Learning Curve

Team needs to learn:
- Library API
- Template literal syntax
- Props-based styling patterns
- Theming system
- SSR setup

#### 7. Flash of Unstyled Content (FOUC)

```jsx
// On slow networks or slow devices
// Component renders before styles inject
// Brief flash of unstyled content

// Especially bad with code-splitting
const Modal = lazy(() => import('./Modal'));
// Modal styles load after Modal component
```

## When to Use CSS-in-JS

### Good Use Cases

#### 1. Highly Dynamic UIs

```jsx
// Styles change based on runtime data
<Chart 
  color={data.color} 
  height={data.height}
  borderWidth={data.borderWidth}
/>
```

#### 2. Component Libraries

```jsx
// Publish styled components for others to use
import { Button, Card, Modal } from 'my-ui-library';

<Button variant="primary">Click</Button>
```

#### 3. Theming Required

```jsx
// Easy theme switching
<ThemeProvider theme={isDark ? darkTheme : lightTheme}>
  <App />
</ThemeProvider>
```

#### 4. Strong TypeScript Integration Needed

```tsx
interface Props {
  variant: 'primary' | 'secondary';
}

const Button = styled.button<Props>`
  background: ${props => 
    props.variant === 'primary' ? '#007bff' : '#6c757d'
  };
`;
```

#### 5. Rapid Prototyping

```jsx
// Quick iteration without switching files
const Component = styled.div`
  display: flex;
  gap: 10px;
`;
```

### When to Consider Alternatives

#### 1. Performance-Critical Apps

Use zero-runtime CSS-in-JS or CSS Modules.

#### 2. Content-Heavy Sites

Static CSS loads faster (blogs, marketing sites).

#### 3. Simple Static Designs

No need for dynamic styling complexity.

#### 4. Large Teams New to React

Easier to start with traditional CSS or CSS Modules.

#### 5. SEO-Critical Apps

SSR complexity may not be worth it.

## Runtime vs Zero-Runtime

### Runtime (styled-components, Emotion)

**Pros:**
- Fully dynamic styling
- Easy to use
- Rich ecosystem

**Cons:**
- Bundle size overhead
- Runtime performance cost
- SSR complexity

**Best for:**
- Apps with heavy dynamic styling
- Component libraries
- Prototype/MVP development

### Zero-Runtime (Linaria, Vanilla Extract, Compiled)

**Pros:**
- No runtime overhead
- Smaller bundles
- Better performance
- Simpler SSR

**Cons:**
- Limited dynamic styling
- Smaller ecosystem
- More build configuration

**Best for:**
- Performance-sensitive apps
- Static designs with occasional dynamics
- Production applications at scale

## Migration Strategies

### From CSS Modules to CSS-in-JS

```jsx
// Before (CSS Modules)
import styles from './Button.module.css';
<button className={styles.button}>Click</button>

// After (Styled Components)
const Button = styled.button`
  /* paste styles from Button.module.css */
`;
<Button>Click</Button>
```

### From CSS-in-JS to Zero-Runtime

```jsx
// Before (styled-components)
const Button = styled.button`
  background: ${props => props.color};
`;

// After (Vanilla Extract)
// Button.css.ts
export const button = style({
  background: 'var(--button-color)',
});

// Button.tsx
<button 
  className={styles.button}
  style={{ '--button-color': color }}
>
```

## Best Practices

### 1. Extract Reusable Styled Components

```jsx
// Bad
function Component() {
  const Button = styled.button`...`;
  return <Button>Click</Button>;
}

// Good
const Button = styled.button`...`;

function Component() {
  return <Button>Click</Button>;
}
```

### 2. Use Transient Props (styled-components 5.1+)

```jsx
// Bad - `color` forwarded to DOM
<Button color="red">Click</Button>
// <button color="red">Click</button>

// Good - `$color` not forwarded
const Button = styled.button`
  background: ${props => props.$color};
`;
<Button $color="red">Click</Button>
// <button>Click</button>
```

### 3. Avoid Inline Styled Components

```jsx
// Bad - new component on every render
{items.map(item => {
  const Item = styled.div`...`;
  return <Item key={item.id}>{item.name}</Item>;
})}

// Good
const Item = styled.div`...`;

{items.map(item => (
  <Item key={item.id}>{item.name}</Item>
))}
```

### 4. Use Theme Consistently

```jsx
// Good
const Button = styled.button`
  background: ${props => props.theme.colors.primary};
  padding: ${props => props.theme.spacing.medium};
`;
```

### 5. Leverage shouldForwardProp (Emotion)

```jsx
import styled from '@emotion/styled';
import { shouldForwardProp } from '@emotion/react';

const Button = styled('button', {
  shouldForwardProp: prop => !['color', 'size'].includes(prop),
})`
  background: ${props => props.color};
  padding: ${props => props.size === 'large' ? '20px' : '10px'};
`;
```

## Conclusion

CSS-in-JS represents a paradigm shift in styling, offering powerful features like dynamic styling, colocation, and type safety. However, it comes with trade-offs in bundle size, runtime performance, and complexity.

### Decision Matrix

Choose **Runtime CSS-in-JS** (styled-components, Emotion) when:
- Highly dynamic styling requirements
- Building component libraries
- TypeScript integration is priority
- Team comfortable with the approach

Choose **Zero-Runtime CSS-in-JS** (Vanilla Extract, Linaria) when:
- Need CSS-in-JS benefits without runtime cost
- Performance is critical
- Styles are mostly static with occasional dynamics

Choose **Alternatives** (CSS Modules, Utility-First) when:
- Runtime overhead is unacceptable
- Team prefers traditional CSS
- Simple static designs
- Learning curve is a concern

The best choice depends on your project requirements, team expertise, and performance constraints. Many successful projects use a hybrid approach, leveraging CSS-in-JS for components needing dynamic styling while using CSS Modules or utilities for static styles.
