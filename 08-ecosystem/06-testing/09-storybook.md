# Storybook - Component Documentation

## The Idea

**In plain English:** Storybook is a tool that lets developers build and showcase individual website pieces (called components, like buttons or forms) in a separate mini-app, so you can test and demo each piece on its own without loading the whole website.

**Real-world analogy:** Imagine a car showroom where each car part — the steering wheel, seats, and dashboard — is displayed on its own stand so customers can inspect and test each part independently before seeing the full assembled car.

- The showroom = Storybook (the separate environment for viewing parts)
- Each display stand = a "story" (an isolated view of one component)
- The car part on the stand = the UI component (a button, card, form, etc.)

---

## Overview

Storybook is an open-source tool for developing and documenting UI components in isolation. It provides a sandbox environment for building, testing, and showcasing components outside your application. Storybook supports React, Vue, Angular, and many other frameworks, offering a powerful ecosystem of addons for accessibility testing, visual regression, and interactive documentation.

## Installation and Setup

```bash
# Initialize Storybook (auto-detects your framework)
npx storybook@latest init

# Or install manually
npm install -D @storybook/react @storybook/react-vite storybook

# Start Storybook
npm run storybook

# Build static Storybook
npm run build-storybook
```

### Configuration

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.mdx', '../src/**/*.stories.@(js|jsx|ts|tsx)'],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y',
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {},
  },
  docs: {
    autodocs: 'tag',
  },
};

export default config;
```

```typescript
// .storybook/preview.ts
import type { Preview } from '@storybook/react';
import '../src/index.css';

const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/,
      },
    },
  },
};

export default preview;
```

## Writing Stories

### Basic Stories (CSF 3.0)

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'danger'],
    },
    size: {
      control: 'radio',
      options: ['small', 'medium', 'large'],
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

// Primary story
export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Button',
  },
};

// Secondary story
export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Button',
  },
};

// Large button
export const Large: Story = {
  args: {
    size: 'large',
    children: 'Large Button',
  },
};

// With icon
export const WithIcon: Story = {
  args: {
    children: (
      <>
        <span>🚀</span> Launch
      </>
    ),
  },
};
```

### Template Pattern

```typescript
// Card.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Card } from './Card';

const meta: Meta<typeof Card> = {
  title: 'Components/Card',
  component: Card,
};

export default meta;
type Story = StoryObj<typeof Card>;

// Template for reuse
const Template: Story = {
  render: (args) => <Card {...args} />,
};

export const Default: Story = {
  ...Template,
  args: {
    title: 'Card Title',
    content: 'This is card content',
  },
};

export const WithImage: Story = {
  ...Template,
  args: {
    title: 'Card with Image',
    content: 'This card has an image',
    image: 'https://via.placeholder.com/300x200',
  },
};

export const Loading: Story = {
  ...Template,
  args: {
    isLoading: true,
  },
};
```

### Decorators

```typescript
// LoginForm.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { LoginForm } from './LoginForm';

const meta: Meta<typeof LoginForm> = {
  title: 'Forms/LoginForm',
  component: LoginForm,
  decorators: [
    (Story) => (
      <div style={{ padding: '3rem', maxWidth: '400px' }}>
        <Story />
      </div>
    ),
  ],
};

export default meta;
type Story = StoryObj<typeof LoginForm>;

export const Default: Story = {};

export const WithError: Story = {
  args: {
    error: 'Invalid credentials',
  },
};

// Story-specific decorator
export const InModal: Story = {
  decorators: [
    (Story) => (
      <div
        style={{
          background: 'rgba(0,0,0,0.5)',
          padding: '2rem',
          borderRadius: '8px',
        }}
      >
        <Story />
      </div>
    ),
  ],
};
```

### Play Functions (Interactions)

```typescript
// SearchBar.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { userEvent, within, expect } from '@storybook/test';
import { SearchBar } from './SearchBar';

const meta: Meta<typeof SearchBar> = {
  title: 'Components/SearchBar',
  component: SearchBar,
};

export default meta;
type Story = StoryObj<typeof SearchBar>;

export const Default: Story = {};

export const WithQuery: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const searchInput = canvas.getByPlaceholderText('Search...');

    await userEvent.type(searchInput, 'React');
    await expect(searchInput).toHaveValue('React');
  },
};

export const SubmitSearch: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    
    await userEvent.type(canvas.getByPlaceholderText('Search...'), 'Testing');
    await userEvent.click(canvas.getByRole('button', { name: /search/i }));

    // Verify search was triggered
    await expect(canvas.getByText(/showing results/i)).toBeInTheDocument();
  },
};
```

## Advanced Patterns

### Args and Controls

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    // Select control
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'danger', 'ghost'],
      description: 'Button variant style',
      table: {
        defaultValue: { summary: 'primary' },
      },
    },
    // Radio control
    size: {
      control: 'radio',
      options: ['small', 'medium', 'large'],
    },
    // Boolean control
    disabled: {
      control: 'boolean',
    },
    // Color control
    backgroundColor: {
      control: 'color',
    },
    // Number control
    borderRadius: {
      control: { type: 'number', min: 0, max: 50, step: 1 },
    },
    // Text control
    label: {
      control: 'text',
    },
    // Object control
    style: {
      control: 'object',
    },
    // Function control (actions)
    onClick: { action: 'clicked' },
    onHover: { action: 'hovered' },
    // Disable control
    children: {
      table: {
        disable: true,
      },
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Playground: Story = {
  args: {
    variant: 'primary',
    size: 'medium',
    disabled: false,
    backgroundColor: '#007bff',
    borderRadius: 4,
    label: 'Button',
  },
};
```

### Multiple Components

```typescript
// UserCard.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { UserCard } from './UserCard';
import { Avatar } from './Avatar';
import { Badge } from './Badge';

const meta: Meta<typeof UserCard> = {
  title: 'Components/UserCard',
  component: UserCard,
  subcomponents: { Avatar, Badge },
};

export default meta;
type Story = StoryObj<typeof UserCard>;

export const Default: Story = {
  args: {
    user: {
      name: 'John Doe',
      email: 'john@example.com',
      avatar: 'https://i.pravatar.cc/150?img=1',
      role: 'Admin',
    },
  },
};

export const WithBadge: Story = {
  render: (args) => (
    <UserCard {...args}>
      <Badge variant="success">Active</Badge>
    </UserCard>
  ),
  args: {
    user: {
      name: 'Jane Doe',
      email: 'jane@example.com',
      avatar: 'https://i.pravatar.cc/150?img=2',
    },
  },
};
```

### Context and Providers

```typescript
// ThemeButton.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { ThemeProvider } from '../context/ThemeContext';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/ThemeButton',
  component: Button,
  decorators: [
    (Story, context) => (
      <ThemeProvider theme={context.globals.theme || 'light'}>
        <Story />
      </ThemeProvider>
    ),
  ],
};

export default meta;
type Story = StoryObj<typeof Button>;

export const LightTheme: Story = {
  args: {
    children: 'Button',
  },
  globals: {
    theme: 'light',
  },
};

export const DarkTheme: Story = {
  args: {
    children: 'Button',
  },
  globals: {
    theme: 'dark',
  },
};
```

## Documentation

### MDX Documentation

```mdx
{/* Button.mdx */}
import { Meta, Story, Canvas, ArgsTable } from '@storybook/blocks';
import * as ButtonStories from './Button.stories';

<Meta of={ButtonStories} />

# Button Component

The Button component is a versatile, accessible button that supports multiple variants and sizes.

## Usage

```tsx
import { Button } from './components/Button';

function App() {
  return (
    <Button variant="primary" onClick={() => alert('Clicked!')}>
      Click me
    </Button>
  );
}
```

## Variants

<Canvas>
  <Story of={ButtonStories.Primary} />
  <Story of={ButtonStories.Secondary} />
  <Story of={ButtonStories.Danger} />
</Canvas>

## Sizes

<Canvas>
  <Story of={ButtonStories.Small} />
  <Story of={ButtonStories.Medium} />
  <Story of={ButtonStories.Large} />
</Canvas>

## Props

<ArgsTable of={ButtonStories} />

## Accessibility

The Button component follows ARIA best practices:

- Uses semantic `<button>` element
- Supports keyboard navigation
- Includes focus indicators
- Provides proper ARIA labels

## Best Practices

- Use descriptive button text
- Provide `aria-label` for icon-only buttons
- Don't disable buttons unnecessarily
- Use appropriate variant for context
```

### Inline Documentation

```typescript
// Input.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Input } from './Input';

const meta: Meta<typeof Input> = {
  title: 'Forms/Input',
  component: Input,
  parameters: {
    docs: {
      description: {
        component: 'A flexible input component with validation support.',
      },
    },
  },
  argTypes: {
    type: {
      description: 'HTML input type',
      table: {
        type: { summary: 'string' },
        defaultValue: { summary: 'text' },
      },
    },
    error: {
      description: 'Error message to display',
      table: {
        type: { summary: 'string' },
      },
    },
  },
};

export default meta;
type Story = StoryObj<typeof Input>;

export const Default: Story = {
  args: {
    placeholder: 'Enter text...',
  },
  parameters: {
    docs: {
      description: {
        story: 'Basic input field without any special styling.',
      },
    },
  },
};

export const WithError: Story = {
  args: {
    value: 'invalid@',
    error: 'Invalid email format',
  },
  parameters: {
    docs: {
      description: {
        story: 'Input showing validation error state.',
      },
    },
  },
};
```

## Testing in Storybook

### Accessibility Testing

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  parameters: {
    a11y: {
      config: {
        rules: [
          {
            id: 'color-contrast',
            enabled: true,
          },
        ],
      },
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Accessible: Story = {
  args: {
    children: 'Accessible Button',
    'aria-label': 'Submit form',
  },
};

export const PoorContrast: Story = {
  args: {
    children: 'Low Contrast',
    style: { backgroundColor: '#eee', color: '#ddd' },
  },
  parameters: {
    a11y: {
      config: {
        rules: [{ id: 'color-contrast', enabled: false }],
      },
    },
  },
};
```

### Visual Testing

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
};

export default meta;
type Story = StoryObj<typeof Button>;

export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: '1rem', flexDirection: 'column' }}>
      <Button variant="primary">Primary</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="danger">Danger</Button>
      <Button disabled>Disabled</Button>
    </div>
  ),
  parameters: {
    chromatic: { 
      viewports: [320, 1200],
      delay: 300,
    },
  },
};
```

## Addons

### Essential Addons

```typescript
// .storybook/main.ts
const config: StorybookConfig = {
  addons: [
    '@storybook/addon-links', // Navigation between stories
    '@storybook/addon-essentials', // Core addons bundle
    '@storybook/addon-interactions', // Interaction testing
    '@storybook/addon-a11y', // Accessibility checking
    '@storybook/addon-coverage', // Code coverage
    '@storybook/addon-themes', // Theme switching
    'storybook-dark-mode', // Dark mode toggle
  ],
};
```

### Custom Addon Configuration

```typescript
// .storybook/preview.ts
import { withThemeFromJSXProvider } from '@storybook/addon-themes';
import { ThemeProvider } from '../src/theme';
import { lightTheme, darkTheme } from '../src/themes';

export const decorators = [
  withThemeFromJSXProvider({
    themes: {
      light: lightTheme,
      dark: darkTheme,
    },
    defaultTheme: 'light',
    Provider: ThemeProvider,
  }),
];
```

## Real-World Patterns

### Component Library

```typescript
// index.stories.tsx
import type { Meta } from '@storybook/react';

const meta: Meta = {
  title: 'Introduction',
  parameters: {
    previewTabs: {
      'storybook/docs/panel': { index: -1 },
    },
  },
};

export default meta;

export const Welcome = {
  render: () => (
    <div>
      <h1>Welcome to Our Component Library</h1>
      <p>Browse components in the sidebar.</p>
      <h2>Getting Started</h2>
      <code>npm install @company/components</code>
      <h2>Usage</h2>
      <pre>
{`import { Button } from '@company/components';

function App() {
  return <Button variant="primary">Click me</Button>;
}`}
      </pre>
    </div>
  ),
};
```

### Design System

```typescript
// Colors.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';

const meta: Meta = {
  title: 'Design System/Colors',
  parameters: {
    layout: 'padded',
  },
};

export default meta;

const colors = {
  primary: '#007bff',
  secondary: '#6c757d',
  success: '#28a745',
  danger: '#dc3545',
  warning: '#ffc107',
  info: '#17a2b8',
};

export const Palette: StoryObj = {
  render: () => (
    <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: '1rem' }}>
      {Object.entries(colors).map(([name, color]) => (
        <div key={name}>
          <div
            style={{
              backgroundColor: color,
              height: '100px',
              borderRadius: '8px',
              marginBottom: '0.5rem',
            }}
          />
          <div>
            <strong>{name}</strong>
            <div style={{ fontSize: '0.875rem', color: '#666' }}>{color}</div>
          </div>
        </div>
      ))}
    </div>
  ),
};
```

## Common Mistakes

1. **Not organizing stories**: Putting all stories at root instead of using hierarchical titles.

2. **Poor naming**: Using unclear story names instead of descriptive ones.

3. **Missing args**: Hardcoding props instead of using args for interactive controls.

4. **No documentation**: Not adding descriptions or usage examples.

5. **Ignoring accessibility**: Not testing with a11y addon or considering keyboard navigation.

6. **Incomplete coverage**: Only showing happy path, missing error and edge cases.

7. **Not using decorators**: Duplicating provider/wrapper code across stories.

8. **Missing interactions**: Not using play functions to demonstrate component behavior.

9. **Poor TypeScript usage**: Not typing stories properly for better DX.

10. **Not building for production**: Not deploying Storybook for team visibility.

## Best Practices

1. **Organize by hierarchy**: Use clear, hierarchical story titles (e.g., 'Components/Forms/Input').

2. **Write comprehensive stories**: Cover all variants, states, and edge cases.

3. **Add rich documentation**: Include MDX docs with usage examples and guidelines.

4. **Use play functions**: Demonstrate interactions and user flows.

5. **Test accessibility**: Enable a11y addon and fix issues.

6. **Leverage args**: Make stories interactive with controls.

7. **Add decorators**: Wrap stories with necessary providers and styling.

8. **Document props**: Use argTypes to describe props clearly.

9. **Deploy Storybook**: Build and deploy for team access and reviews.

10. **Integrate with CI/CD**: Run interaction tests and visual regression in pipeline.

## When to Use Storybook

**Use Storybook when:**
- Building component library or design system
- Want isolated component development
- Need living documentation
- Want to showcase components to stakeholders
- Testing component variations
- Need accessibility testing
- Building design system

**Consider alternatives when:**
- Only need simple documentation (use TypeDoc)
- Building small projects
- Need E2E testing (use Playwright/Cypress)
- Want API documentation (use Swagger)

## Interview Questions

### Q1: What problem does Storybook solve?

**Answer**: Storybook solves the problem of developing and testing UI components in isolation. Without Storybook, developers must navigate through the full application to reach specific component states, which is time-consuming and limits testing edge cases. Storybook provides a sandbox where components can be developed independently with all their variations, documented, and tested without running the full application.

### Q2: What are CSF 3.0 stories and how do they differ from previous versions?

**Answer**: Component Story Format (CSF) 3.0 is the latest story format that uses object notation instead of templates. Stories are defined as simple objects with args, making them more concise. Example: `export const Primary: Story = { args: { variant: 'primary' } }` vs older template pattern. CSF 3.0 offers better TypeScript support, cleaner syntax, and better composition.

### Q3: What are decorators in Storybook and when should you use them?

**Answer**: Decorators are wrapper components that add context or styling to stories. Use them to: 1) Wrap stories with providers (theme, router, redux), 2) Add consistent spacing/styling, 3) Mock global dependencies. Decorators can be defined globally in preview.ts or per-story. They're essential for components that require context to function properly.

### Q4: How do play functions work and what are they used for?

**Answer**: Play functions are async functions that run after a story renders, simulating user interactions. They use Testing Library and user-event to interact with components. Use cases: 1) Demonstrate component behavior, 2) Test user flows, 3) Capture interaction states for visual testing. Example: typing in inputs, clicking buttons, verifying results. They enable interaction testing directly in Storybook.

### Q5: How do you handle components that require context in Storybook?

**Answer**: Use decorators to wrap stories with context providers:

```typescript
decorators: [
  (Story) => (
    <ThemeProvider>
      <RouterProvider>
        <Story />
      </RouterProvider>
    </ThemeProvider>
  ),
]
```

Or use addon-themes/contexts addons for dynamic context switching.

### Q6: What's the difference between args and parameters in Storybook?

**Answer**: Args are component props that can be modified via controls panel, making stories interactive. Parameters are configuration options that control Storybook's behavior (layout, viewport, a11y config). Args affect component rendering, parameters affect story presentation. Use args for component inputs, parameters for Storybook settings.

### Q7: How do you test accessibility in Storybook?

**Answer**: Use @storybook/addon-a11y which runs automated accessibility checks using axe-core. It appears as a panel showing violations. Configure rules per story:

```typescript
parameters: {
  a11y: {
    config: {
      rules: [{ id: 'color-contrast', enabled: true }],
    },
  },
}
```

Combine with manual testing using keyboard navigation and screen readers.

## Key Takeaways

1. Storybook provides isolated component development environment with live reloading and hot module replacement.

2. Component Story Format (CSF) 3.0 uses object-based stories with args for interactive controls.

3. Decorators wrap stories with providers, context, or styling needed for components to function.

4. Play functions enable interaction testing by simulating user actions after story renders.

5. Args make stories interactive through controls panel, while parameters configure Storybook behavior.

6. MDX documentation combines stories with rich prose documentation for comprehensive guides.

7. Accessibility addon (a11y) runs automated accessibility checks using axe-core on every story.

8. Visual testing integrations enable catching visual regressions through snapshot comparisons.

9. Story organization uses hierarchical titles for clear navigation in large component libraries.

10. Production builds create static sites for deployment, enabling team collaboration and stakeholder reviews.

## Resources

- [Official Documentation](https://storybook.js.org/docs)
- [Component Story Format](https://storybook.js.org/docs/react/api/csf)
- [Addons Catalog](https://storybook.js.org/addons)
- [Play Functions](https://storybook.js.org/docs/react/writing-stories/play-function)
- [Accessibility Testing](https://storybook.js.org/docs/react/writing-tests/accessibility-testing)
- [Visual Testing](https://storybook.js.org/docs/react/writing-tests/visual-testing)
- [Tutorials](https://storybook.js.org/tutorials/)
