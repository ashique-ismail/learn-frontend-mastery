# Chakra UI

## The Idea

**In plain English:** Chakra UI is a ready-made collection of styled building blocks (buttons, forms, menus, etc.) for React apps, so instead of designing everything from scratch you just pick a piece, drop it in, and it already looks good and works correctly. "Accessible" means it also works for people using keyboards or screen readers, without extra effort on your part.

**Real-world analogy:** Think of a flat-pack furniture store (like IKEA) where every item already comes in a matching style, fits together perfectly, and has clear assembly instructions. You pick the pieces you need and assemble your room.

- The furniture store catalog = Chakra UI's component library
- Each furniture piece (chair, table, shelf) = a Chakra component (Button, Input, Modal)
- The shared color and style guide across all pieces = the Chakra theme (colors, spacing, fonts)

---

## Introduction

Chakra UI is a simple, modular, and accessible component library that provides building blocks for React applications. It emphasizes developer experience with sensible defaults, excellent TypeScript support, and a powerful theming system based on design tokens and style props.

## Installation and Setup

### Basic Installation

```bash
npm install @chakra-ui/react @emotion/react @emotion/styled framer-motion

# or with yarn
yarn add @chakra-ui/react @emotion/react @emotion/styled framer-motion

# or with pnpm
pnpm add @chakra-ui/react @emotion/react @emotion/styled framer-motion
```

### Provider Setup

```tsx
// app/providers.tsx
'use client';

import { ChakraProvider } from '@chakra-ui/react';

export function Providers({ children }: { children: React.ReactNode }) {
  return <ChakraProvider>{children}</ChakraProvider>;
}

// app/layout.tsx
import { Providers } from './providers';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### Custom Theme Setup

```tsx
// theme.ts
import { extendTheme, type ThemeConfig } from '@chakra-ui/react';

const config: ThemeConfig = {
  initialColorMode: 'light',
  useSystemColorMode: false,
};

const colors = {
  brand: {
    50: '#e3f2fd',
    100: '#bbdefb',
    200: '#90caf9',
    300: '#64b5f6',
    400: '#42a5f5',
    500: '#2196f3',
    600: '#1e88e5',
    700: '#1976d2',
    800: '#1565c0',
    900: '#0d47a1',
  },
};

const fonts = {
  heading: `'Poppins', sans-serif`,
  body: `'Inter', sans-serif`,
};

const styles = {
  global: (props: any) => ({
    body: {
      bg: props.colorMode === 'dark' ? 'gray.900' : 'white',
      color: props.colorMode === 'dark' ? 'white' : 'gray.800',
    },
  }),
};

export const theme = extendTheme({ config, colors, fonts, styles });

// app/providers.tsx
import { theme } from './theme';

export function Providers({ children }: { children: React.ReactNode }) {
  return <ChakraProvider theme={theme}>{children}</ChakraProvider>;
}
```

## Core Concepts

### Style Props

```tsx
import { Box, Text } from '@chakra-ui/react';

function StylePropsExample() {
  return (
    <Box
      bg="brand.500"
      w="100%"
      p={4}
      color="white"
      borderRadius="lg"
      boxShadow="xl"
      _hover={{ bg: 'brand.600', transform: 'translateY(-2px)' }}
      _active={{ transform: 'translateY(0)' }}
      transition="all 0.2s"
    >
      <Text fontSize="2xl" fontWeight="bold">
        Style Props Demo
      </Text>
      <Text mt={2} fontSize="md" opacity={0.9}>
        Chakra UI allows inline styling with type-safe props
      </Text>
    </Box>
  );
}
```

### Responsive Styles

```tsx
import { Box, Stack, Heading } from '@chakra-ui/react';

function ResponsiveExample() {
  return (
    <Box
      width={{ base: '100%', md: '50%', lg: '33.33%' }}
      padding={{ base: 4, md: 6, lg: 8 }}
      bg={{ base: 'blue.100', md: 'green.100', lg: 'purple.100' }}
    >
      <Heading
        fontSize={{ base: 'xl', md: '2xl', lg: '3xl' }}
        textAlign={{ base: 'center', md: 'left' }}
      >
        Responsive Heading
      </Heading>
      
      <Stack
        direction={{ base: 'column', md: 'row' }}
        spacing={{ base: 2, md: 4, lg: 6 }}
        mt={4}
      >
        <Box flex={1} bg="red.200" p={4}>Item 1</Box>
        <Box flex={1} bg="yellow.200" p={4}>Item 2</Box>
      </Stack>
    </Box>
  );
}
```

### Color Mode

```tsx
import {
  Box,
  Button,
  useColorMode,
  useColorModeValue,
  IconButton,
} from '@chakra-ui/react';
import { MoonIcon, SunIcon } from '@chakra-ui/icons';

function ColorModeExample() {
  const { colorMode, toggleColorMode } = useColorMode();
  const bg = useColorModeValue('white', 'gray.800');
  const color = useColorModeValue('gray.800', 'white');

  return (
    <Box bg={bg} color={color} p={8} borderRadius="md">
      <IconButton
        aria-label="Toggle color mode"
        icon={colorMode === 'light' ? <MoonIcon /> : <SunIcon />}
        onClick={toggleColorMode}
        size="lg"
        mb={4}
      />
      
      <Button
        colorScheme="brand"
        variant={useColorModeValue('solid', 'outline')}
      >
        Current mode: {colorMode}
      </Button>
    </Box>
  );
}
```

## Component Examples

### Layout Components

```tsx
import {
  Container,
  Box,
  Grid,
  GridItem,
  Flex,
  Spacer,
  Stack,
  VStack,
  HStack,
} from '@chakra-ui/react';

function LayoutExample() {
  return (
    <Container maxW="container.xl" py={8}>
      {/* Flexbox Layout */}
      <Flex mb={8}>
        <Box bg="brand.500" p={4} color="white">Left</Box>
        <Spacer />
        <Box bg="brand.600" p={4} color="white">Right</Box>
      </Flex>

      {/* Grid Layout */}
      <Grid
        templateColumns={{ base: '1fr', md: 'repeat(3, 1fr)' }}
        gap={6}
        mb={8}
      >
        <GridItem bg="teal.100" p={4}>Grid Item 1</GridItem>
        <GridItem bg="teal.200" p={4}>Grid Item 2</GridItem>
        <GridItem bg="teal.300" p={4}>Grid Item 3</GridItem>
      </Grid>

      {/* Stack Layout */}
      <VStack spacing={4} align="stretch">
        <Box bg="purple.100" p={4}>Vertical Stack 1</Box>
        <Box bg="purple.200" p={4}>Vertical Stack 2</Box>
        
        <HStack spacing={4}>
          <Box bg="orange.100" p={4} flex={1}>Horizontal 1</Box>
          <Box bg="orange.200" p={4} flex={1}>Horizontal 2</Box>
        </HStack>
      </VStack>
    </Container>
  );
}
```

### Form Components

```tsx
import {
  FormControl,
  FormLabel,
  FormErrorMessage,
  FormHelperText,
  Input,
  InputGroup,
  InputLeftElement,
  InputRightElement,
  Button,
  VStack,
  useToast,
} from '@chakra-ui/react';
import { EmailIcon, LockIcon, ViewIcon, ViewOffIcon } from '@chakra-ui/icons';
import { useState } from 'react';

function FormExample() {
  const [show, setShow] = useState(false);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState({ email: '', password: '' });
  const toast = useToast();

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const newErrors = { email: '', password: '' };

    if (!email) {
      newErrors.email = 'Email is required';
    } else if (!/\S+@\S+\.\S+/.test(email)) {
      newErrors.email = 'Email is invalid';
    }

    if (!password) {
      newErrors.password = 'Password is required';
    } else if (password.length < 8) {
      newErrors.password = 'Password must be at least 8 characters';
    }

    setErrors(newErrors);

    if (!newErrors.email && !newErrors.password) {
      toast({
        title: 'Form submitted',
        description: 'Your credentials have been validated',
        status: 'success',
        duration: 3000,
        isClosable: true,
      });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <VStack spacing={4} align="stretch">
        <FormControl isInvalid={!!errors.email}>
          <FormLabel>Email</FormLabel>
          <InputGroup>
            <InputLeftElement pointerEvents="none">
              <EmailIcon color="gray.300" />
            </InputLeftElement>
            <Input
              type="email"
              placeholder="Enter your email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
            />
          </InputGroup>
          {errors.email ? (
            <FormErrorMessage>{errors.email}</FormErrorMessage>
          ) : (
            <FormHelperText>We'll never share your email.</FormHelperText>
          )}
        </FormControl>

        <FormControl isInvalid={!!errors.password}>
          <FormLabel>Password</FormLabel>
          <InputGroup>
            <InputLeftElement pointerEvents="none">
              <LockIcon color="gray.300" />
            </InputLeftElement>
            <Input
              type={show ? 'text' : 'password'}
              placeholder="Enter password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
            />
            <InputRightElement width="4.5rem">
              <Button
                h="1.75rem"
                size="sm"
                onClick={() => setShow(!show)}
                variant="ghost"
              >
                {show ? <ViewOffIcon /> : <ViewIcon />}
              </Button>
            </InputRightElement>
          </InputGroup>
          {errors.password && (
            <FormErrorMessage>{errors.password}</FormErrorMessage>
          )}
        </FormControl>

        <Button type="submit" colorScheme="brand" size="lg">
          Sign In
        </Button>
      </VStack>
    </form>
  );
}
```

### Modal and Disclosure

```tsx
import {
  Modal,
  ModalOverlay,
  ModalContent,
  ModalHeader,
  ModalFooter,
  ModalBody,
  ModalCloseButton,
  Button,
  useDisclosure,
  Text,
} from '@chakra-ui/react';

function ModalExample() {
  const { isOpen, onOpen, onClose } = useDisclosure();

  return (
    <>
      <Button onClick={onOpen} colorScheme="brand">
        Open Modal
      </Button>

      <Modal isOpen={isOpen} onClose={onClose} size="xl" isCentered>
        <ModalOverlay
          bg="blackAlpha.300"
          backdropFilter="blur(10px)"
        />
        <ModalContent>
          <ModalHeader>Modal Title</ModalHeader>
          <ModalCloseButton />
          <ModalBody>
            <Text>
              This is a modal dialog with backdrop blur effect. Chakra UI
              provides excellent accessibility features out of the box,
              including focus management and keyboard navigation.
            </Text>
          </ModalBody>

          <ModalFooter>
            <Button variant="ghost" mr={3} onClick={onClose}>
              Cancel
            </Button>
            <Button colorScheme="brand" onClick={onClose}>
              Confirm
            </Button>
          </ModalFooter>
        </ModalContent>
      </Modal>
    </>
  );
}
```

### Menu and Dropdown

```tsx
import {
  Menu,
  MenuButton,
  MenuList,
  MenuItem,
  MenuGroup,
  MenuDivider,
  Button,
  IconButton,
  Avatar,
} from '@chakra-ui/react';
import { ChevronDownIcon, HamburgerIcon } from '@chakra-ui/icons';

function MenuExample() {
  return (
    <>
      {/* Simple Menu */}
      <Menu>
        <MenuButton as={Button} rightIcon={<ChevronDownIcon />}>
          Actions
        </MenuButton>
        <MenuList>
          <MenuItem>Download</MenuItem>
          <MenuItem>Create a Copy</MenuItem>
          <MenuItem>Mark as Draft</MenuItem>
          <MenuItem>Delete</MenuItem>
        </MenuList>
      </Menu>

      {/* Grouped Menu */}
      <Menu>
        <MenuButton
          as={IconButton}
          aria-label="Options"
          icon={<HamburgerIcon />}
          variant="outline"
        />
        <MenuList>
          <MenuGroup title="Profile">
            <MenuItem>My Account</MenuItem>
            <MenuItem>Payments</MenuItem>
          </MenuGroup>
          <MenuDivider />
          <MenuGroup title="Help">
            <MenuItem>Docs</MenuItem>
            <MenuItem>FAQ</MenuItem>
          </MenuGroup>
        </MenuList>
      </Menu>

      {/* User Menu */}
      <Menu>
        <MenuButton>
          <Avatar
            size="sm"
            name="John Doe"
            src="https://bit.ly/dan-abramov"
          />
        </MenuButton>
        <MenuList>
          <MenuItem>Profile</MenuItem>
          <MenuItem>Settings</MenuItem>
          <MenuDivider />
          <MenuItem color="red.500">Sign Out</MenuItem>
        </MenuList>
      </Menu>
    </>
  );
}
```

### Toast Notifications

```tsx
import {
  Button,
  useToast,
  VStack,
  HStack,
  Box,
  Text,
  CloseButton,
} from '@chakra-ui/react';

function ToastExample() {
  const toast = useToast();

  const showToast = (status: 'success' | 'error' | 'warning' | 'info') => {
    toast({
      title: `${status} toast`,
      description: `This is a ${status} message`,
      status,
      duration: 3000,
      isClosable: true,
      position: 'top-right',
    });
  };

  const showCustomToast = () => {
    toast({
      position: 'bottom-left',
      render: ({ onClose }) => (
        <Box
          bg="brand.500"
          color="white"
          p={4}
          borderRadius="md"
          boxShadow="lg"
        >
          <HStack justify="space-between">
            <VStack align="start" spacing={0}>
              <Text fontWeight="bold">Custom Toast</Text>
              <Text fontSize="sm">With custom styling</Text>
            </VStack>
            <CloseButton onClick={onClose} />
          </HStack>
        </Box>
      ),
    });
  };

  return (
    <VStack spacing={4}>
      <HStack spacing={4}>
        <Button onClick={() => showToast('success')} colorScheme="green">
          Success
        </Button>
        <Button onClick={() => showToast('error')} colorScheme="red">
          Error
        </Button>
        <Button onClick={() => showToast('warning')} colorScheme="orange">
          Warning
        </Button>
        <Button onClick={() => showToast('info')} colorScheme="blue">
          Info
        </Button>
      </HStack>
      <Button onClick={showCustomToast} colorScheme="brand">
        Custom Toast
      </Button>
    </VStack>
  );
}
```

### Card Component

```tsx
import {
  Card,
  CardHeader,
  CardBody,
  CardFooter,
  Heading,
  Text,
  Button,
  Image,
  Stack,
  Divider,
  ButtonGroup,
} from '@chakra-ui/react';

function CardExample() {
  return (
    <Stack spacing={8} direction={{ base: 'column', md: 'row' }}>
      {/* Simple Card */}
      <Card maxW="sm">
        <CardBody>
          <Image
            src="https://images.unsplash.com/photo-1555041469-a586c61ea9bc"
            alt="Product"
            borderRadius="lg"
          />
          <Stack mt={6} spacing={3}>
            <Heading size="md">Living room Sofa</Heading>
            <Text>
              This sofa is perfect for modern tropical spaces, baroque inspired
              spaces.
            </Text>
            <Text color="brand.600" fontSize="2xl">
              $450
            </Text>
          </Stack>
        </CardBody>
        <Divider />
        <CardFooter>
          <ButtonGroup spacing={2}>
            <Button variant="solid" colorScheme="brand">
              Buy now
            </Button>
            <Button variant="ghost" colorScheme="brand">
              Add to cart
            </Button>
          </ButtonGroup>
        </CardFooter>
      </Card>

      {/* Card with Header */}
      <Card maxW="sm">
        <CardHeader>
          <Heading size="md">User Profile</Heading>
        </CardHeader>
        <CardBody>
          <Stack spacing={4}>
            <Text>
              <strong>Name:</strong> John Doe
            </Text>
            <Text>
              <strong>Email:</strong> john@example.com
            </Text>
            <Text>
              <strong>Role:</strong> Developer
            </Text>
          </Stack>
        </CardBody>
        <Divider />
        <CardFooter>
          <Button colorScheme="brand" w="full">
            Edit Profile
          </Button>
        </CardFooter>
      </Card>
    </Stack>
  );
}
```

## Advanced Patterns

### Custom Component with Theme

```tsx
// components/CustomButton.tsx
import { Button, ButtonProps, useStyleConfig } from '@chakra-ui/react';

interface CustomButtonProps extends ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
}

export function CustomButton({ variant = 'primary', ...props }: CustomButtonProps) {
  const styles = useStyleConfig('CustomButton', { variant });

  return <Button __css={styles} {...props} />;
}

// theme.ts
import { defineStyleConfig } from '@chakra-ui/react';

const CustomButton = defineStyleConfig({
  baseStyle: {
    fontWeight: 'bold',
    borderRadius: 'full',
    _focus: {
      boxShadow: 'outline',
    },
  },
  variants: {
    primary: {
      bg: 'brand.500',
      color: 'white',
      _hover: {
        bg: 'brand.600',
      },
    },
    secondary: {
      bg: 'gray.200',
      color: 'gray.800',
      _hover: {
        bg: 'gray.300',
      },
    },
    danger: {
      bg: 'red.500',
      color: 'white',
      _hover: {
        bg: 'red.600',
      },
    },
  },
  sizes: {
    sm: {
      fontSize: 'sm',
      px: 4,
      py: 2,
    },
    md: {
      fontSize: 'md',
      px: 6,
      py: 3,
    },
    lg: {
      fontSize: 'lg',
      px: 8,
      py: 4,
    },
  },
  defaultProps: {
    size: 'md',
    variant: 'primary',
  },
});

export const theme = extendTheme({
  components: {
    CustomButton,
  },
});
```

### Data Table Component

```tsx
import {
  Table,
  Thead,
  Tbody,
  Tr,
  Th,
  Td,
  TableContainer,
  Badge,
  Avatar,
  HStack,
  Text,
  IconButton,
  Menu,
  MenuButton,
  MenuList,
  MenuItem,
} from '@chakra-ui/react';
import { ChevronDownIcon } from '@chakra-ui/icons';

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
  status: 'active' | 'inactive';
  avatar: string;
}

function DataTable() {
  const users: User[] = [
    {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com',
      role: 'Admin',
      status: 'active',
      avatar: 'https://bit.ly/dan-abramov',
    },
    {
      id: 2,
      name: 'Jane Smith',
      email: 'jane@example.com',
      role: 'User',
      status: 'active',
      avatar: 'https://bit.ly/code-beast',
    },
    {
      id: 3,
      name: 'Bob Johnson',
      email: 'bob@example.com',
      role: 'User',
      status: 'inactive',
      avatar: 'https://bit.ly/ryan-florence',
    },
  ];

  return (
    <TableContainer>
      <Table variant="simple">
        <Thead>
          <Tr>
            <Th>User</Th>
            <Th>Role</Th>
            <Th>Status</Th>
            <Th>Actions</Th>
          </Tr>
        </Thead>
        <Tbody>
          {users.map((user) => (
            <Tr key={user.id}>
              <Td>
                <HStack spacing={3}>
                  <Avatar size="sm" name={user.name} src={user.avatar} />
                  <VStack align="start" spacing={0}>
                    <Text fontWeight="medium">{user.name}</Text>
                    <Text fontSize="sm" color="gray.500">
                      {user.email}
                    </Text>
                  </VStack>
                </HStack>
              </Td>
              <Td>{user.role}</Td>
              <Td>
                <Badge
                  colorScheme={user.status === 'active' ? 'green' : 'gray'}
                >
                  {user.status}
                </Badge>
              </Td>
              <Td>
                <Menu>
                  <MenuButton
                    as={IconButton}
                    icon={<ChevronDownIcon />}
                    variant="ghost"
                    size="sm"
                  />
                  <MenuList>
                    <MenuItem>Edit</MenuItem>
                    <MenuItem>View Details</MenuItem>
                    <MenuItem color="red.500">Delete</MenuItem>
                  </MenuList>
                </Menu>
              </Td>
            </Tr>
          ))}
        </Tbody>
      </Table>
    </TableContainer>
  );
}
```

### Custom Hook for Breakpoints

```tsx
import { useBreakpointValue } from '@chakra-ui/react';

function useResponsiveValue() {
  const columns = useBreakpointValue({
    base: 1,
    md: 2,
    lg: 3,
    xl: 4,
  });

  const isMobile = useBreakpointValue({ base: true, md: false });
  const isTablet = useBreakpointValue({ base: false, md: true, lg: false });
  const isDesktop = useBreakpointValue({ base: false, lg: true });

  return { columns, isMobile, isTablet, isDesktop };
}

function ResponsiveComponent() {
  const { columns, isMobile } = useResponsiveValue();

  return (
    <Box>
      <Text>Current columns: {columns}</Text>
      {isMobile && <Text>Mobile view active</Text>}
    </Box>
  );
}
```

## Accessibility Features

### Focus Management

```tsx
import {
  Button,
  Drawer,
  DrawerBody,
  DrawerFooter,
  DrawerHeader,
  DrawerOverlay,
  DrawerContent,
  DrawerCloseButton,
  useDisclosure,
  Input,
} from '@chakra-ui/react';
import { useRef } from 'react';

function AccessibleDrawer() {
  const { isOpen, onOpen, onClose } = useDisclosure();
  const btnRef = useRef(null);

  return (
    <>
      <Button ref={btnRef} colorScheme="brand" onClick={onOpen}>
        Open Drawer
      </Button>
      <Drawer
        isOpen={isOpen}
        placement="right"
        onClose={onClose}
        finalFocusRef={btnRef}
      >
        <DrawerOverlay />
        <DrawerContent>
          <DrawerCloseButton />
          <DrawerHeader>Create your account</DrawerHeader>

          <DrawerBody>
            <Input placeholder="Type here..." autoFocus />
          </DrawerBody>

          <DrawerFooter>
            <Button variant="outline" mr={3} onClick={onClose}>
              Cancel
            </Button>
            <Button colorScheme="brand">Save</Button>
          </DrawerFooter>
        </DrawerContent>
      </Drawer>
    </>
  );
}
```

### Skip Navigation

```tsx
import { Box, Link, VisuallyHidden } from '@chakra-ui/react';

function AccessibleLayout() {
  return (
    <>
      <Link
        href="#main-content"
        position="absolute"
        left="-9999px"
        zIndex="9999"
        _focus={{
          left: 0,
          top: 0,
          bg: 'brand.500',
          color: 'white',
          p: 4,
        }}
      >
        Skip to main content
      </Link>

      <Box as="nav" aria-label="Main navigation">
        {/* Navigation items */}
      </Box>

      <Box id="main-content" as="main">
        {/* Main content */}
      </Box>
    </>
  );
}
```

## Common Mistakes

1. **Not using semantic HTML**: Chakra components accept `as` prop for semantic markup
```tsx
// Bad
<Box>Title</Box>

// Good
<Box as="h1">Title</Box>
// or better
<Heading as="h1">Title</Heading>
```

2. **Overusing inline styles**: Use theme tokens instead
```tsx
// Bad
<Box bg="#2196f3" />

// Good
<Box bg="brand.500" />
```

3. **Ignoring responsive values**: Always consider mobile-first design
```tsx
// Bad
<Box width="500px" />

// Good
<Box width={{ base: '100%', md: '500px' }} />
```

4. **Not leveraging color mode**: Design for both light and dark modes
```tsx
// Bad
<Box bg="white" color="black" />

// Good
const bg = useColorModeValue('white', 'gray.800');
const color = useColorModeValue('gray.800', 'white');
<Box bg={bg} color={color} />
```

5. **Missing accessibility attributes**: Always provide proper ARIA labels
```tsx
// Bad
<IconButton icon={<SearchIcon />} onClick={search} />

// Good
<IconButton
  aria-label="Search database"
  icon={<SearchIcon />}
  onClick={search}
/>
```

## Best Practices

1. **Use composition**: Build complex components from simple ones
2. **Leverage style props**: Use Chakra's style props instead of CSS
3. **Create custom theme**: Define your design tokens upfront
4. **Use semantic components**: Prefer specific components over generic Box
5. **Implement proper focus management**: Use `finalFocusRef` in modals
6. **Design for accessibility**: Test with keyboard navigation and screen readers
7. **Optimize performance**: Use `shouldForwardProp` for styled components
8. **Use TypeScript**: Take advantage of excellent type support
9. **Create reusable components**: Build your component library on top of Chakra
10. **Test color modes**: Always test both light and dark themes

## When to Use Chakra UI

### Use Chakra UI When:
- You need rapid prototyping with good defaults
- Accessibility is a priority
- You want excellent TypeScript support
- You prefer component-based styling over CSS files
- You need built-in dark mode support
- You want a comprehensive component library
- You value developer experience and documentation

### Consider Alternatives When:
- You need extremely custom designs (consider Tailwind CSS)
- You require minimal bundle size (use Radix UI with custom styling)
- You prefer CSS-in-JS alternatives (consider vanilla-extract)
- You need specific component libraries (Material UI for Material Design)

## Interview Questions

1. **Q: How does Chakra UI implement responsive styles?**
   A: Chakra uses an object or array syntax for responsive values mapped to breakpoints (base, sm, md, lg, xl, 2xl). These are transformed into media queries at build time.

2. **Q: Explain color mode implementation in Chakra UI.**
   A: Chakra uses `ColorModeProvider` with local storage persistence, CSS variables for theme tokens, and hooks like `useColorMode` and `useColorModeValue` for runtime theme access.

3. **Q: What is the difference between Box and Flex components?**
   A: Box is a generic container with display: block by default. Flex extends Box with display: flex and provides flex-specific props like justify, align, and direction.

4. **Q: How do you extend Chakra's theme?**
   A: Use `extendTheme` utility to merge custom config, colors, fonts, components, and other theme values with Chakra's default theme.

5. **Q: What are style props and how do they work?**
   A: Style props are component props that map to CSS properties, allowing inline styling with type safety and theme-aware values (e.g., `bg="brand.500"` maps to `background-color: var(--chakra-colors-brand-500)`).

## Key Takeaways

1. Chakra UI provides an excellent balance between flexibility and structure
2. Style props enable rapid development without leaving JSX
3. Built-in accessibility features save significant development time
4. The theming system based on design tokens ensures consistency
5. Responsive design is simple with object/array syntax
6. Color mode support is seamless and well-implemented
7. TypeScript support is exceptional throughout the library
8. Component composition patterns enable building complex UIs
9. Focus management and ARIA attributes are handled automatically
10. The library strikes a good balance between bundle size and features

## Resources

- **Official Documentation**: https://chakra-ui.com/docs/getting-started
- **GitHub Repository**: https://github.com/chakra-ui/chakra-ui
- **Component Examples**: https://chakra-ui.com/docs/components
- **Theme Tools**: https://chakra-ui.com/docs/styled-system/theme
- **Chakra Templates**: https://chakra-templates.dev/
- **Pro Components**: https://pro.chakra-ui.com/
- **Discord Community**: https://discord.gg/chakra-ui
- **Awesome Chakra UI**: https://github.com/chakra-ui/awesome-chakra-ui
- **Figma Kit**: https://www.figma.com/community/file/971408767069651759
- **YouTube Tutorials**: Search for "Chakra UI" on YouTube for video guides
