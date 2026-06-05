# Emotion & styled-components

## The Idea

**In plain English:** Emotion and styled-components are tools that let you write the visual styling of a webpage (colors, sizes, spacing) right inside the same file as your website's building blocks, and those styles can automatically change based on what's happening in your app — like a button turning blue when it's the "main" action.

**Real-world analogy:** Think of a costume designer on a movie set who sews a unique outfit for each actor and adjusts it on the fly based on the scene (daytime scene = bright outfit, nighttime scene = dark outfit). The designer travels with the actor instead of working in a separate wardrobe department.

- The actor = a UI component (like a Button)
- The outfit sewn onto the actor = the CSS styles written inside the same file as the component
- Changing the outfit based on the scene = styles that update automatically based on props or app state

---

## What They Are

Both are **CSS-in-JS** libraries that let you write CSS inside JavaScript/TypeScript files. Styles are colocated with components, scoped automatically, and can use JavaScript variables and logic.

---

## styled-components

```tsx
import styled from 'styled-components';

const Button = styled.button<{ $variant?: 'primary' | 'ghost' }>`
  display: inline-flex;
  align-items: center;
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: 600;

  background: ${props => props.$variant === 'primary' ? '#3b82f6' : 'transparent'};
  color: ${props => props.$variant === 'primary' ? 'white' : '#374151'};

  &:hover {
    background: ${props => props.$variant === 'primary' ? '#2563eb' : '#f3f4f6'};
  }
`;

// Usage
<Button $variant="primary">Submit</Button>
<Button>Cancel</Button>
```

Note: The `$` prefix (styled-components v6+) prevents props from being forwarded to the DOM element.

### Extending Styles

```tsx
const BaseButton = styled.button`padding: 8px 16px; border-radius: 4px;`;
const PrimaryButton = styled(BaseButton)`background: #3b82f6; color: white;`;
const IconButton = styled(BaseButton)`padding: 8px; border-radius: 50%;`;
```

### `ThemeProvider`

```tsx
const theme = {
  colors: { primary: '#3b82f6', text: '#111827' },
  spacing: { sm: '8px', md: '16px' },
};

// Wrap app
<ThemeProvider theme={theme}>
  <App />
</ThemeProvider>

// Access in any styled component
const Card = styled.div`
  padding: ${props => props.theme.spacing.md};
  color: ${props => props.theme.colors.text};
`;
```

---

## Emotion

Emotion offers two APIs:

### `styled` (same as styled-components)

```tsx
import styled from '@emotion/styled';

const Button = styled.button`
  background: #3b82f6;
  color: white;
`;
```

### `css` prop (Emotion's unique feature)

```tsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

function Button({ variant }) {
  return (
    <button
      css={css`
        background: ${variant === 'primary' ? '#3b82f6' : 'transparent'};
        color: white;
        padding: 8px 16px;
      `}
    >
      Click
    </button>
  );
}
```

The `css` prop lets you write one-off styles directly on any element without creating a new component. Requires the JSX pragma or Babel plugin.

---

## Runtime vs Build Time

Both styled-components and Emotion are **runtime CSS-in-JS** — they inject `<style>` tags into the DOM at runtime:

```
Component renders → CSS string computed (with JS values) → style injected into <head>
```

This means:
- ✅ Fully dynamic styles (use any JS expression)
- ✅ Scoped automatically (hashed class names)
- ❌ Style recalculation happens in JS → performance cost on large apps
- ❌ Flash of unstyled content in SSR without careful hydration setup
- ❌ Extra bundle weight (~15-30 KB for runtime)

---

## SSR Considerations

Both require special setup for server-side rendering to avoid hydration mismatches:

```tsx
// styled-components: ServerStyleSheet
import { ServerStyleSheet } from 'styled-components';
const sheet = new ServerStyleSheet();
const html = renderToString(sheet.collectStyles(<App />));
const styleTags = sheet.getStyleTags();
// Inject styleTags into HTML head

// Emotion: Global cache
import createEmotionServer from '@emotion/server/create-instance';
const cache = createCache({ key: 'css' });
const { extractCriticalToChunks } = createEmotionServer(cache);
```

---

## Comparison

| | styled-components | Emotion |
|---|---|---|
| Bundle size | ~15 KB gzipped | ~8 KB gzipped |
| `css` prop | No (requires a component) | Yes (unique feature) |
| Performance | Slightly slower | Generally faster |
| TypeScript | Good | Good |
| Theming | ThemeProvider | ThemeProvider or CSS vars |
| SSR | ServerStyleSheet | extractCritical |
| Community | Larger | Growing |
| Adopted by | Large existing codebases | MUI v5+, many new projects |

---

## Comparison with Alternatives

| | Emotion/styled-components | vanilla-extract | Tailwind CSS |
|---|---|---|---|
| Runtime | Yes (style injection) | No (build time CSS) | No (utility classes) |
| Dynamic styles | Full (any JS) | Via CSS vars only | Via CVA/conditional classes |
| Bundle | +15-30 KB JS | 0 KB JS | Utility CSS only |
| SSR | Requires setup | Built-in | Built-in |

---

## Common Interview Questions

**Q: What's the performance cost of CSS-in-JS at runtime?**
On every render, Emotion/styled-components serialize the CSS template string, hash it, check if it's been injected, and inject if not. For static styles, this is cached. For dynamic styles (changing props), this runs on every render. In MUI v5, this caused measurable performance issues for large component trees — which is why newer libraries like vanilla-extract choose zero-runtime.

**Q: Why did MUI switch from JSS to Emotion?**
JSS (another CSS-in-JS library) had performance issues and a less developer-friendly API. Emotion was the most popular runtime CSS-in-JS with similar performance characteristics but better DX. MUI v5+ uses Emotion by default with an option to use styled-components.

**Q: When would you still choose Emotion/styled-components today?**
When you need fully dynamic styles based on runtime JavaScript values (complex animations driven by state, theme values from an API). For most cases, vanilla-extract or Tailwind with CVA provide better performance without sacrificing DX.
