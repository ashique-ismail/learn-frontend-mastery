# Mantine

## Introduction

Mantine is a fully featured React component library with 100+ customizable components and 50+ hooks. It provides exceptional developer experience with TypeScript support, built-in dark theme, accessibility features, and a powerful theming system. Mantine emphasizes flexibility, allowing developers to build everything from simple websites to complex applications.

## Installation and Setup

### Basic Installation

```bash
npm install @mantine/core @mantine/hooks

# or with yarn
yarn add @mantine/core @mantine/hooks

# or with pnpm
pnpm add @mantine/core @mantine/hooks
```

### Additional Packages

```bash
# Forms
npm install @mantine/form

# Notifications
npm install @mantine/notifications

# Date picker
npm install @mantine/dates dayjs

# Rich text editor
npm install @mantine/tiptap @tiptap/react @tiptap/starter-kit

# Code highlight
npm install @mantine/code-highlight
```

### App Setup

```tsx
// app/layout.tsx or _app.tsx
import '@mantine/core/styles.css';
import { MantineProvider, ColorSchemeScript } from '@mantine/core';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <head>
        <ColorSchemeScript />
      </head>
      <body>
        <MantineProvider>{children}</MantineProvider>
      </body>
    </html>
  );
}
```

### Custom Theme Configuration

```tsx
import { MantineProvider, createTheme } from '@mantine/core';

const theme = createTheme({
  primaryColor: 'blue',
  defaultRadius: 'md',
  fontFamily: 'Inter, sans-serif',
  headings: {
    fontFamily: 'Poppins, sans-serif',
    fontWeight: '700',
    sizes: {
      h1: { fontSize: '2.5rem', lineHeight: '1.2' },
      h2: { fontSize: '2rem', lineHeight: '1.3' },
      h3: { fontSize: '1.5rem', lineHeight: '1.4' },
    },
  },
  colors: {
    brand: [
      '#e3f2fd',
      '#bbdefb',
      '#90caf9',
      '#64b5f6',
      '#42a5f5',
      '#2196f3',
      '#1e88e5',
      '#1976d2',
      '#1565c0',
      '#0d47a1',
    ],
  },
  shadows: {
    sm: '0 1px 3px rgba(0, 0, 0, 0.05)',
    md: '0 4px 6px rgba(0, 0, 0, 0.1)',
    lg: '0 10px 15px rgba(0, 0, 0, 0.1)',
    xl: '0 20px 25px rgba(0, 0, 0, 0.15)',
  },
  spacing: {
    xs: '0.5rem',
    sm: '0.75rem',
    md: '1rem',
    lg: '1.5rem',
    xl: '2rem',
  },
});

function App() {
  return (
    <MantineProvider theme={theme} defaultColorScheme="light">
      {/* Your app */}
    </MantineProvider>
  );
}

export default App;
```

## Core Components

### Button and ActionIcon

```tsx
import {
  Button,
  Group,
  Stack,
  ActionIcon,
  Tooltip,
} from '@mantine/core';
import {
  IconTrash,
  IconEdit,
  IconDownload,
  IconHeart,
} from '@tabler/icons-react';

function ButtonExamples() {
  return (
    <Stack gap="md">
      {/* Button Variants */}
      <Group>
        <Button variant="filled">Filled</Button>
        <Button variant="light">Light</Button>
        <Button variant="outline">Outline</Button>
        <Button variant="subtle">Subtle</Button>
        <Button variant="default">Default</Button>
        <Button variant="white">White</Button>
      </Group>

      {/* Button Colors */}
      <Group>
        <Button color="blue">Blue</Button>
        <Button color="red">Red</Button>
        <Button color="green">Green</Button>
        <Button color="brand">Brand</Button>
      </Group>

      {/* Button Sizes */}
      <Group>
        <Button size="xs">Extra Small</Button>
        <Button size="sm">Small</Button>
        <Button size="md">Medium</Button>
        <Button size="lg">Large</Button>
        <Button size="xl">Extra Large</Button>
      </Group>

      {/* Loading States */}
      <Group>
        <Button loading>Loading</Button>
        <Button loading loaderProps={{ type: 'dots' }}>
          Loading Dots
        </Button>
      </Group>

      {/* With Icons */}
      <Group>
        <Button leftSection={<IconDownload size={16} />}>
          Download
        </Button>
        <Button rightSection={<IconHeart size={16} />}>
          Like
        </Button>
      </Group>

      {/* Action Icons */}
      <Group>
        <Tooltip label="Edit">
          <ActionIcon variant="filled" color="blue">
            <IconEdit size={18} />
          </ActionIcon>
        </Tooltip>
        <Tooltip label="Delete">
          <ActionIcon variant="filled" color="red">
            <IconTrash size={18} />
          </ActionIcon>
        </Tooltip>
      </Group>
    </Stack>
  );
}

export default ButtonExamples;
```

### Input Components

```tsx
import {
  TextInput,
  PasswordInput,
  Textarea,
  Select,
  MultiSelect,
  NumberInput,
  Checkbox,
  Radio,
  Switch,
  Stack,
  Group,
} from '@mantine/core';
import { IconMail, IconLock, IconUser } from '@tabler/icons-react';
import { useState } from 'react';

function InputExamples() {
  const [value, setValue] = useState('');
  const [checked, setChecked] = useState(false);

  return (
    <Stack gap="md">
      {/* Text Inputs */}
      <TextInput
        label="Email"
        placeholder="your@email.com"
        description="We'll never share your email"
        withAsterisk
        leftSection={<IconMail size={16} />}
      />

      <PasswordInput
        label="Password"
        placeholder="Your password"
        description="Must be at least 8 characters"
        withAsterisk
        leftSection={<IconLock size={16} />}
      />

      <TextInput
        label="Username"
        placeholder="Username"
        leftSection={<IconUser size={16} />}
        rightSection={
          <Tooltip label="Username must be unique">
            <IconInfoCircle size={16} />
          </Tooltip>
        }
      />

      {/* Textarea */}
      <Textarea
        label="Bio"
        placeholder="Tell us about yourself"
        description="Maximum 500 characters"
        minRows={4}
        maxRows={8}
        autosize
      />

      {/* Select */}
      <Select
        label="Country"
        placeholder="Pick one"
        data={[
          { value: 'usa', label: 'United States' },
          { value: 'uk', label: 'United Kingdom' },
          { value: 'canada', label: 'Canada' },
          { value: 'australia', label: 'Australia' },
        ]}
        searchable
        clearable
      />

      {/* MultiSelect */}
      <MultiSelect
        label="Skills"
        placeholder="Pick all that apply"
        data={[
          { value: 'react', label: 'React' },
          { value: 'angular', label: 'Angular' },
          { value: 'vue', label: 'Vue' },
          { value: 'svelte', label: 'Svelte' },
        ]}
        searchable
        clearable
      />

      {/* Number Input */}
      <NumberInput
        label="Age"
        placeholder="Your age"
        min={18}
        max={120}
        clampBehavior="strict"
      />

      {/* Checkbox */}
      <Checkbox
        label="I agree to terms and conditions"
        description="You must agree to continue"
      />

      <Checkbox.Group
        label="Select your preferences"
        description="Choose all that apply"
        withAsterisk
      >
        <Group mt="xs">
          <Checkbox value="newsletter" label="Newsletter" />
          <Checkbox value="updates" label="Product Updates" />
          <Checkbox value="marketing" label="Marketing" />
        </Group>
      </Checkbox.Group>

      {/* Radio */}
      <Radio.Group
        label="Select your plan"
        description="Choose the plan that fits you best"
        withAsterisk
      >
        <Group mt="xs">
          <Radio value="free" label="Free" />
          <Radio value="pro" label="Pro" />
          <Radio value="enterprise" label="Enterprise" />
        </Group>
      </Radio.Group>

      {/* Switch */}
      <Switch
        label="Enable notifications"
        description="Receive email notifications"
        checked={checked}
        onChange={(event) => setChecked(event.currentTarget.checked)}
      />
    </Stack>
  );
}

export default InputExamples;
```

### Form with Validation

```tsx
import { useForm } from '@mantine/form';
import {
  TextInput,
  PasswordInput,
  Button,
  Group,
  Box,
  NumberInput,
  Select,
} from '@mantine/core';
import { notifications } from '@mantine/notifications';

interface FormValues {
  email: string;
  password: string;
  confirmPassword: string;
  age: number | string;
  country: string;
}

function FormExample() {
  const form = useForm<FormValues>({
    initialValues: {
      email: '',
      password: '',
      confirmPassword: '',
      age: '',
      country: '',
    },

    validate: {
      email: (value) => (/^\S+@\S+$/.test(value) ? null : 'Invalid email'),
      password: (value) =>
        value.length < 8 ? 'Password must be at least 8 characters' : null,
      confirmPassword: (value, values) =>
        value !== values.password ? 'Passwords do not match' : null,
      age: (value) => {
        const age = Number(value);
        if (isNaN(age)) return 'Age must be a number';
        if (age < 18) return 'You must be at least 18 years old';
        if (age > 120) return 'Please enter a valid age';
        return null;
      },
      country: (value) => (!value ? 'Please select a country' : null),
    },

    transformValues: (values) => ({
      ...values,
      age: Number(values.age),
    }),
  });

  const handleSubmit = (values: typeof form.values) => {
    console.log('Form submitted:', values);
    notifications.show({
      title: 'Success',
      message: 'Form submitted successfully!',
      color: 'green',
    });
    form.reset();
  };

  return (
    <Box maw={400} mx="auto">
      <form onSubmit={form.onSubmit(handleSubmit)}>
        <TextInput
          withAsterisk
          label="Email"
          placeholder="your@email.com"
          {...form.getInputProps('email')}
          mb="md"
        />

        <PasswordInput
          withAsterisk
          label="Password"
          placeholder="Password"
          {...form.getInputProps('password')}
          mb="md"
        />

        <PasswordInput
          withAsterisk
          label="Confirm Password"
          placeholder="Confirm password"
          {...form.getInputProps('confirmPassword')}
          mb="md"
        />

        <NumberInput
          withAsterisk
          label="Age"
          placeholder="Your age"
          {...form.getInputProps('age')}
          mb="md"
        />

        <Select
          withAsterisk
          label="Country"
          placeholder="Select country"
          data={[
            { value: 'usa', label: 'United States' },
            { value: 'uk', label: 'United Kingdom' },
            { value: 'canada', label: 'Canada' },
          ]}
          {...form.getInputProps('country')}
          mb="md"
        />

        <Group justify="flex-end" mt="md">
          <Button type="submit">Submit</Button>
        </Group>
      </form>
    </Box>
  );
}

export default FormExample;
```

### Modal and Drawer

```tsx
import { useDisclosure } from '@mantine/hooks';
import {
  Modal,
  Drawer,
  Button,
  Stack,
  Text,
  TextInput,
  Group,
} from '@mantine/core';

function ModalDrawerExample() {
  const [modalOpened, { open: openModal, close: closeModal }] = useDisclosure(false);
  const [drawerOpened, { open: openDrawer, close: closeDrawer }] = useDisclosure(false);

  return (
    <Stack gap="md">
      <Group>
        <Button onClick={openModal}>Open Modal</Button>
        <Button onClick={openDrawer}>Open Drawer</Button>
      </Group>

      {/* Modal */}
      <Modal
        opened={modalOpened}
        onClose={closeModal}
        title="Create New User"
        centered
        size="lg"
      >
        <Stack gap="md">
          <TextInput label="Name" placeholder="Enter name" />
          <TextInput label="Email" placeholder="Enter email" />
          <Group justify="flex-end">
            <Button variant="subtle" onClick={closeModal}>
              Cancel
            </Button>
            <Button onClick={closeModal}>Create</Button>
          </Group>
        </Stack>
      </Modal>

      {/* Drawer */}
      <Drawer
        opened={drawerOpened}
        onClose={closeDrawer}
        title="Settings"
        position="right"
        size="md"
      >
        <Stack gap="md">
          <TextInput label="Display Name" placeholder="Enter display name" />
          <TextInput label="Bio" placeholder="Tell us about yourself" />
          <Button onClick={closeDrawer}>Save Changes</Button>
        </Stack>
      </Drawer>
    </Stack>
  );
}

export default ModalDrawerExample;
```

### Notifications

```tsx
import { Button, Group, Stack } from '@mantine/core';
import { notifications } from '@mantine/notifications';
import { IconCheck, IconX, IconInfoCircle } from '@tabler/icons-react';

function NotificationExamples() {
  const showSuccess = () => {
    notifications.show({
      title: 'Success',
      message: 'Operation completed successfully',
      color: 'green',
      icon: <IconCheck size={18} />,
    });
  };

  const showError = () => {
    notifications.show({
      title: 'Error',
      message: 'Something went wrong',
      color: 'red',
      icon: <IconX size={18} />,
    });
  };

  const showInfo = () => {
    notifications.show({
      title: 'Information',
      message: 'This is an informational message',
      color: 'blue',
      icon: <IconInfoCircle size={18} />,
      autoClose: 5000,
      withCloseButton: true,
    });
  };

  const showLoading = () => {
    const id = notifications.show({
      loading: true,
      title: 'Loading',
      message: 'Please wait...',
      autoClose: false,
      withCloseButton: false,
    });

    setTimeout(() => {
      notifications.update({
        id,
        color: 'green',
        title: 'Complete',
        message: 'Operation completed!',
        icon: <IconCheck size={18} />,
        loading: false,
        autoClose: 2000,
      });
    }, 2000);
  };

  const showCustom = () => {
    notifications.show({
      title: 'Custom Notification',
      message: 'With custom styling and position',
      color: 'grape',
      position: 'bottom-right',
      autoClose: 3000,
      withBorder: true,
    });
  };

  return (
    <Stack gap="md">
      <Group>
        <Button onClick={showSuccess} color="green">
          Success
        </Button>
        <Button onClick={showError} color="red">
          Error
        </Button>
        <Button onClick={showInfo} color="blue">
          Info
        </Button>
        <Button onClick={showLoading} color="orange">
          Loading
        </Button>
        <Button onClick={showCustom} color="grape">
          Custom
        </Button>
      </Group>
    </Stack>
  );
}

export default NotificationExamples;
```

### Data Table

```tsx
import { Table, Badge, Group, ActionIcon, Text } from '@mantine/core';
import { IconEdit, IconTrash, IconEye } from '@tabler/icons-react';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
  status: 'active' | 'inactive';
}

function DataTable() {
  const users: User[] = [
    {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com',
      role: 'Admin',
      status: 'active',
    },
    {
      id: 2,
      name: 'Jane Smith',
      email: 'jane@example.com',
      role: 'User',
      status: 'active',
    },
    {
      id: 3,
      name: 'Bob Johnson',
      email: 'bob@example.com',
      role: 'User',
      status: 'inactive',
    },
  ];

  const rows = users.map((user) => (
    <Table.Tr key={user.id}>
      <Table.Td>{user.name}</Table.Td>
      <Table.Td>{user.email}</Table.Td>
      <Table.Td>{user.role}</Table.Td>
      <Table.Td>
        <Badge color={user.status === 'active' ? 'green' : 'gray'}>
          {user.status}
        </Badge>
      </Table.Td>
      <Table.Td>
        <Group gap="xs">
          <ActionIcon variant="subtle" color="blue">
            <IconEye size={16} />
          </ActionIcon>
          <ActionIcon variant="subtle" color="yellow">
            <IconEdit size={16} />
          </ActionIcon>
          <ActionIcon variant="subtle" color="red">
            <IconTrash size={16} />
          </ActionIcon>
        </Group>
      </Table.Td>
    </Table.Tr>
  ));

  return (
    <Table striped highlightOnHover withTableBorder withColumnBorders>
      <Table.Thead>
        <Table.Tr>
          <Table.Th>Name</Table.Th>
          <Table.Th>Email</Table.Th>
          <Table.Th>Role</Table.Th>
          <Table.Th>Status</Table.Th>
          <Table.Th>Actions</Table.Th>
        </Table.Tr>
      </Table.Thead>
      <Table.Tbody>{rows}</Table.Tbody>
    </Table>
  );
}

export default DataTable;
```

## Mantine Hooks

### useForm Hook

```tsx
import { useForm, zodResolver } from '@mantine/form';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(2, { message: 'Name must be at least 2 characters' }),
  email: z.string().email({ message: 'Invalid email' }),
  age: z.number().min(18, { message: 'You must be at least 18' }),
});

function ZodValidationForm() {
  const form = useForm({
    initialValues: {
      name: '',
      email: '',
      age: 0,
    },
    validate: zodResolver(schema),
  });

  return (
    <form onSubmit={form.onSubmit(console.log)}>
      <TextInput {...form.getInputProps('name')} label="Name" />
      <TextInput {...form.getInputProps('email')} label="Email" />
      <NumberInput {...form.getInputProps('age')} label="Age" />
      <Button type="submit">Submit</Button>
    </form>
  );
}
```

### useDisclosure Hook

```tsx
import { useDisclosure } from '@mantine/hooks';
import { Button, Collapse, Paper, Text } from '@mantine/core';

function DisclosureExample() {
  const [opened, { toggle, open, close }] = useDisclosure(false);

  return (
    <>
      <Button onClick={toggle}>Toggle Content</Button>
      <Collapse in={opened}>
        <Paper p="md" mt="md" withBorder>
          <Text>This content can be toggled</Text>
        </Paper>
      </Collapse>
    </>
  );
}
```

### useHover Hook

```tsx
import { useHover } from '@mantine/hooks';
import { Paper, Text } from '@mantine/core';

function HoverExample() {
  const { hovered, ref } = useHover();

  return (
    <Paper
      ref={ref}
      p="xl"
      bg={hovered ? 'blue' : 'gray'}
      c="white"
      style={{ transition: 'background-color 0.2s' }}
    >
      <Text>{hovered ? 'Hovering!' : 'Hover over me'}</Text>
    </Paper>
  );
}
```

### useLocalStorage Hook

```tsx
import { useLocalStorage } from '@mantine/hooks';
import { Button, Stack, Text } from '@mantine/core';

function LocalStorageExample() {
  const [value, setValue] = useLocalStorage({
    key: 'my-key',
    defaultValue: 0,
  });

  return (
    <Stack>
      <Text>Value: {value}</Text>
      <Button onClick={() => setValue((current) => current + 1)}>
        Increment
      </Button>
    </Stack>
  );
}
```

### useMediaQuery Hook

```tsx
import { useMediaQuery } from '@mantine/hooks';
import { Text } from '@mantine/core';

function MediaQueryExample() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  const isDesktop = useMediaQuery('(min-width: 1025px)');

  return (
    <Text>
      {isMobile && 'Mobile view'}
      {isTablet && 'Tablet view'}
      {isDesktop && 'Desktop view'}
    </Text>
  );
}
```

## Advanced Patterns

### Color Scheme Toggle

```tsx
import {
  useMantineColorScheme,
  ActionIcon,
  Group,
} from '@mantine/core';
import { IconSun, IconMoon } from '@tabler/icons-react';

function ColorSchemeToggle() {
  const { colorScheme, setColorScheme } = useMantineColorScheme();
  const isDark = colorScheme === 'dark';

  return (
    <Group>
      <ActionIcon
        variant="outline"
        color={isDark ? 'yellow' : 'blue'}
        onClick={() => setColorScheme(isDark ? 'light' : 'dark')}
        title="Toggle color scheme"
      >
        {isDark ? <IconSun size={18} /> : <IconMoon size={18} />}
      </ActionIcon>
    </Group>
  );
}

export default ColorSchemeToggle;
```

### Custom Component with Polymorphic Prop

```tsx
import { Box, BoxProps, ElementProps } from '@mantine/core';

interface CustomBoxProps extends BoxProps, ElementProps<'div'> {
  highlight?: boolean;
}

function CustomBox({ highlight, children, ...others }: CustomBoxProps) {
  return (
    <Box
      p="md"
      bg={highlight ? 'blue.1' : undefined}
      style={{
        border: highlight ? '2px solid var(--mantine-color-blue-5)' : undefined,
      }}
      {...others}
    >
      {children}
    </Box>
  );
}

// Usage with polymorphism
<CustomBox component="a" href="#" highlight>
  This is a link
</CustomBox>
```

## Common Mistakes

1. **Not importing styles**: Always import @mantine/core/styles.css
```tsx
// Bad - Missing styles import
import { Button } from '@mantine/core';

// Good
import '@mantine/core/styles.css';
import { Button } from '@mantine/core';
```

2. **Forgetting MantineProvider**: Required for theme and context
```tsx
// Bad
<App />

// Good
<MantineProvider>
  <App />
</MantineProvider>
```

3. **Using incorrect prop names**: Mantine uses specific naming conventions
```tsx
// Bad
<Button className="my-button">Click</Button>

// Good
<Button classNames={{ root: 'my-button' }}>Click</Button>
```

4. **Not using form.getInputProps()**: Manual binding is error-prone
```tsx
// Bad
<TextInput
  value={form.values.name}
  onChange={(e) => form.setFieldValue('name', e.target.value)}
  error={form.errors.name}
/>

// Good
<TextInput {...form.getInputProps('name')} />
```

5. **Missing notifications package**: Required for notifications.show()
```tsx
// Install first
npm install @mantine/notifications

// Then import styles
import '@mantine/notifications/styles.css';
```

## Best Practices

1. **Use TypeScript**: Mantine has excellent TypeScript support
2. **Leverage Mantine hooks**: They solve common UI problems elegantly
3. **Customize theme globally**: Use MantineProvider for consistent styling
4. **Use form validation**: @mantine/form provides robust validation
5. **Optimize bundle size**: Import only components you need
6. **Follow accessibility guidelines**: Mantine components are accessible by default
7. **Use polymorphic components**: component prop allows semantic HTML
8. **Implement proper error handling**: Use error boundaries
9. **Test with different themes**: Ensure components work in light/dark modes
10. **Use Mantine's spacing system**: Consistent spacing with gap/p/m props

## When to Use Mantine

### Use Mantine When:
- You want a modern, comprehensive component library
- Developer experience and TypeScript support are priorities
- You need built-in hooks for common patterns
- Dark mode support is required
- You value flexibility and customization
- Building complex forms with validation
- You want excellent documentation and examples

### Consider Alternatives When:
- You need a specific design system (Material UI, Ant Design)
- Minimal styling/headless components preferred (Radix UI)
- You're working with utility-first CSS (Tailwind components)
- Bundle size is critical (consider lighter alternatives)
- You need enterprise-specific features (Ant Design)

## Interview Questions

1. **Q: What makes Mantine different from other component libraries?**
   A: Mantine focuses on developer experience with 100+ components, 50+ hooks, excellent TypeScript support, built-in dark theme, and a flexible theming system. It's more modern than Material UI and more comprehensive than Chakra UI.

2. **Q: How does theming work in Mantine?**
   A: Mantine uses CSS variables with the createTheme function. Themes define colors, spacing, fonts, shadows, and component-specific styles. The MantineProvider applies themes globally with support for color scheme switching.

3. **Q: Explain the useForm hook.**
   A: useForm manages form state, validation, and submission. It supports Zod/Yup validation, field dependencies, async validation, and provides getInputProps for easy input binding.

4. **Q: What are polymorphic components in Mantine?**
   A: Components accepting the component prop can render as any HTML element or React component while maintaining Mantine styling and TypeScript types. Example: <Button component="a" href="#">Link</Button>

5. **Q: How do you optimize Mantine bundle size?**
   A: Import specific components, use tree-shaking, avoid importing entire packages, lazy load heavy components, and use @mantine/core/styles.css instead of individual component styles.

## Key Takeaways

1. Mantine provides 100+ components and 50+ hooks for comprehensive UI development
2. Excellent TypeScript support with full type inference
3. Built-in dark mode with seamless color scheme switching
4. Powerful form management with @mantine/form
5. Flexible theming system using CSS variables
6. Polymorphic components for semantic HTML
7. Great documentation with interactive examples
8. Active development and responsive community
9. Hooks library solves common UI patterns elegantly
10. Balance between flexibility and out-of-the-box functionality

## Resources

- **Official Documentation**: https://mantine.dev/
- **GitHub Repository**: https://github.com/mantinedev/mantine
- **Component Demos**: https://mantine.dev/core/button/
- **Hooks Documentation**: https://mantine.dev/hooks/use-disclosure/
- **Form Library**: https://mantine.dev/form/use-form/
- **Template Gallery**: https://mantine.dev/templates/
- **Discord Community**: https://discord.gg/wbH82zuWMN
- **NPM Package**: https://www.npmjs.com/package/@mantine/core
- **Figma UI Kit**: https://www.figma.com/community/file/1293863610787190027
- **YouTube Tutorials**: Search for "Mantine React" for video guides
