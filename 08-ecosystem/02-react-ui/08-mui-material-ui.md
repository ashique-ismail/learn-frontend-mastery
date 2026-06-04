# Material-UI (MUI) - React Component Library

## Overview

Material-UI is a comprehensive React UI framework implementing Google's Material Design. It provides a robust set of customizable components, powerful theming capabilities, and accessibility features out of the box.

### Key Features

- **Material Design**: Google's design system
- **Rich component library**: 50+ components
- **Theming system**: Powerful customization
- **TypeScript support**: Full type safety
- **Accessibility**: ARIA compliant
- **Responsive**: Mobile-first approach
- **Server-side rendering**: SSR compatible

## Installation

```bash
npm install @mui/material @emotion/react @emotion/styled
```

## Basic Setup

```typescript
import { ThemeProvider, createTheme } from '@mui/material/styles';
import CssBaseline from '@mui/material/CssBaseline';

const theme = createTheme();

function App() {
  return (
    <ThemeProvider theme={theme}>
      <CssBaseline />
      <YourApp />
    </ThemeProvider>
  );
}
```

## Core Components

### Buttons

```typescript
import Button from '@mui/material/Button';
import IconButton from '@mui/material/IconButton';
import DeleteIcon from '@mui/icons-material/Delete';

function ButtonExamples() {
  return (
    <div>
      <Button variant="contained">Contained</Button>
      <Button variant="outlined">Outlined</Button>
      <Button variant="text">Text</Button>
      <Button variant="contained" color="primary">Primary</Button>
      <Button variant="contained" color="secondary">Secondary</Button>
      <Button variant="contained" disabled>Disabled</Button>
      <Button variant="contained" size="small">Small</Button>
      <Button variant="contained" size="large">Large</Button>
      <IconButton aria-label="delete">
        <DeleteIcon />
      </IconButton>
    </div>
  );
}
```

### Form Controls

```typescript
import TextField from '@mui/material/TextField';
import Select from '@mui/material/Select';
import MenuItem from '@mui/material/MenuItem';
import Checkbox from '@mui/material/Checkbox';
import Radio from '@mui/material/Radio';
import RadioGroup from '@mui/material/RadioGroup';
import FormControlLabel from '@mui/material/FormControlLabel';

function FormExample() {
  const [value, setValue] = React.useState('');

  return (
    <form>
      <TextField label="Name" variant="outlined" fullWidth margin="normal" />
      <TextField label="Email" type="email" variant="filled" fullWidth />
      <TextField label="Message" multiline rows={4} fullWidth />
      
      <Select value={value} onChange={(e) => setValue(e.target.value)}>
        <MenuItem value="option1">Option 1</MenuItem>
        <MenuItem value="option2">Option 2</MenuItem>
      </Select>

      <FormControlLabel control={<Checkbox />} label="Accept terms" />
      
      <RadioGroup>
        <FormControlLabel value="yes" control={<Radio />} label="Yes" />
        <FormControlLabel value="no" control={<Radio />} label="No" />
      </RadioGroup>
    </form>
  );
}
```

## Theming

```typescript
import { createTheme, ThemeProvider } from '@mui/material/styles';

const theme = createTheme({
  palette: {
    primary: {
      main: '#1976d2',
      light: '#42a5f5',
      dark: '#1565c0',
    },
    secondary: {
      main: '#9c27b0',
    },
    mode: 'light',
  },
  typography: {
    fontFamily: '"Roboto", "Helvetica", "Arial", sans-serif',
    h1: {
      fontSize: '2.5rem',
      fontWeight: 500,
    },
  },
  components: {
    MuiButton: {
      styleOverrides: {
        root: {
          textTransform: 'none',
        },
      },
    },
  },
});
```

## Common Mistakes

1. **Not using ThemeProvider**: Missing theme context
2. **Inline styles over sx prop**: Use sx for better performance
3. **Missing CssBaseline**: Inconsistent baseline styles
4. **Over-customizing**: Use theme tokens instead
5. **Not leveraging Grid system**: Manual layout calculations

## Best Practices

1. **Use theme tokens**: Consistent design system
2. **Leverage sx prop**: Performance and convenience
3. **Component composition**: Reuse and extend components
4. **Accessibility**: Use semantic HTML and ARIA
5. **Responsive design**: Use breakpoints consistently
6. **TypeScript**: Full type safety
7. **Tree shaking**: Import specific components

## When to Use MUI

### Good Use Cases
- Enterprise applications
- Admin dashboards
- Material Design preference
- Need comprehensive component library
- Rapid prototyping

### Not Ideal For
- Custom design systems (use headless UI)
- Minimal bundle size requirements
- Non-Material Design aesthetics

## Resources

- [Official Documentation](https://mui.com/)
- [Component API](https://mui.com/material-ui/api/button/)
- [Theming Guide](https://mui.com/material-ui/customization/theming/)
