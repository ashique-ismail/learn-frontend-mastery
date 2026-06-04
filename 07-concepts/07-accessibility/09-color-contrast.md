# Color Contrast

## Table of Contents
- [Introduction](#introduction)
- [Why Color Contrast Matters](#why-color-contrast-matters)
- [Understanding Contrast Ratios](#understanding-contrast-ratios)
- [WCAG Requirements](#wcag-requirements)
- [Color Blindness](#color-blindness)
- [Testing Tools](#testing-tools)
- [Accessible Color Palettes](#accessible-color-palettes)
- [Implementation Patterns](#implementation-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Color contrast is the difference in luminance between foreground content (usually text) and its background. Adequate contrast ensures that content is readable for users with low vision, color blindness, or those viewing content in challenging conditions (bright sunlight, low-quality displays, etc.).

```
Contrast Ratio Formula
======================

Contrast Ratio = (L1 + 0.05) / (L2 + 0.05)

Where:
- L1 = Relative luminance of lighter color
- L2 = Relative luminance of darker color
- Result ranges from 1:1 (no contrast) to 21:1 (maximum contrast)

Examples:
Black on White:   21:1  ✓ Excellent
Dark Gray on White: 7:1  ✓ Good
Light Gray on White: 2.5:1  ✗ Fail (< 4.5:1)
```

Approximately 8% of men and 0.5% of women have some form of color vision deficiency, making color contrast crucial for accessibility.

## Why Color Contrast Matters

### Legal and Compliance Requirements

- **ADA (Americans with Disabilities Act)**: Requires accessible websites
- **Section 508**: Federal accessibility standards
- **WCAG 2.1 AA**: Industry standard (4.5:1 for normal text)
- **WCAG 2.1 AAA**: Enhanced requirements (7:1 for normal text)
- **European Accessibility Act**: EU requirements
- **AODA (Ontario)**: Canadian provincial requirements

### User Impact

```typescript
// Example: Impact of Poor Contrast

// ❌ 2.1:1 ratio - Illegible for many users
<button style={{ color: '#777', backgroundColor: '#999' }}>
  Submit Form
</button>

// ✅ 7:1 ratio - Readable for all users
<button style={{ color: '#000', backgroundColor: '#fff' }}>
  Submit Form
</button>

/*
Users Affected by Poor Contrast:
- Low vision (253 million people worldwide)
- Color blindness (300 million people)
- Aging eyes (presbyopia affects 1.8 billion)
- Environmental factors (everyone on mobile in sunlight)
*/
```

## Understanding Contrast Ratios

### Luminance Calculation

```typescript
// React: Contrast ratio calculator utility
function getLuminance(r: number, g: number, b: number): number {
  // Convert RGB to sRGB
  const sRGB = [r, g, b].map((channel) => {
    const val = channel / 255;
    return val <= 0.03928
      ? val / 12.92
      : Math.pow((val + 0.055) / 1.055, 2.4);
  });

  // Calculate relative luminance
  return 0.2126 * sRGB[0] + 0.7152 * sRGB[1] + 0.0722 * sRGB[2];
}

function getContrastRatio(color1: string, color2: string): number {
  // Parse hex colors
  const hex1 = color1.replace('#', '');
  const hex2 = color2.replace('#', '');

  const rgb1 = {
    r: parseInt(hex1.substr(0, 2), 16),
    g: parseInt(hex1.substr(2, 2), 16),
    b: parseInt(hex1.substr(4, 2), 16)
  };

  const rgb2 = {
    r: parseInt(hex2.substr(0, 2), 16),
    g: parseInt(hex2.substr(2, 2), 16),
    b: parseInt(hex2.substr(4, 2), 16)
  };

  const lum1 = getLuminance(rgb1.r, rgb1.g, rgb1.b);
  const lum2 = getLuminance(rgb2.r, rgb2.g, rgb2.b);

  const lighter = Math.max(lum1, lum2);
  const darker = Math.min(lum1, lum2);

  return (lighter + 0.05) / (darker + 0.05);
}

// Usage
const ContrastChecker: React.FC = () => {
  const [foreground, setForeground] = React.useState('#000000');
  const [background, setBackground] = React.useState('#FFFFFF');
  const [ratio, setRatio] = React.useState(21);

  React.useEffect(() => {
    const calculatedRatio = getContrastRatio(foreground, background);
    setRatio(Math.round(calculatedRatio * 100) / 100);
  }, [foreground, background]);

  const passesAA = ratio >= 4.5;
  const passesAAA = ratio >= 7;
  const passesAALarge = ratio >= 3;

  return (
    <div className="contrast-checker">
      <h2>Contrast Ratio Checker</h2>

      <div className="color-inputs">
        <label>
          Foreground:
          <input
            type="color"
            value={foreground}
            onChange={(e) => setForeground(e.target.value)}
          />
          <input
            type="text"
            value={foreground}
            onChange={(e) => setForeground(e.target.value)}
          />
        </label>

        <label>
          Background:
          <input
            type="color"
            value={background}
            onChange={(e) => setBackground(e.target.value)}
          />
          <input
            type="text"
            value={background}
            onChange={(e) => setBackground(e.target.value)}
          />
        </label>
      </div>

      <div
        className="preview"
        style={{
          color: foreground,
          backgroundColor: background,
          padding: '20px',
          fontSize: '18px'
        }}
      >
        <p>Sample text at normal size (18px)</p>
        <p style={{ fontSize: '24px', fontWeight: 'bold' }}>
          Sample text at large size (24px bold)
        </p>
      </div>

      <div className="results">
        <h3>Contrast Ratio: {ratio.toFixed(2)}:1</h3>

        <div className="compliance">
          <div className={passesAA ? 'pass' : 'fail'}>
            <strong>WCAG AA (Normal Text):</strong>{' '}
            {passesAA ? '✓ Pass (≥4.5:1)' : '✗ Fail (<4.5:1)'}
          </div>

          <div className={passesAALarge ? 'pass' : 'fail'}>
            <strong>WCAG AA (Large Text):</strong>{' '}
            {passesAALarge ? '✓ Pass (≥3:1)' : '✗ Fail (<3:1)'}
          </div>

          <div className={passesAAA ? 'pass' : 'fail'}>
            <strong>WCAG AAA (Normal Text):</strong>{' '}
            {passesAAA ? '✓ Pass (≥7:1)' : '✗ Fail (<7:1)'}
          </div>
        </div>
      </div>
    </div>
  );
};
```

### Visual Representation

```
Contrast Ratio Scale
====================

21:1 ████████████████████ Black on White (Maximum)
18:1 ██████████████████   Very Dark Gray on White
12:1 ████████████         Dark Gray on White
7:1  ███████              WCAG AAA Threshold
4.5:1 █████               WCAG AA Threshold (Normal Text)
3:1   ███                 WCAG AA Threshold (Large Text)
2:1   ██                  Insufficient
1:1   █                   No Contrast
```

## WCAG Requirements

### Level AA Requirements (Minimum)

```typescript
// React: WCAG compliance constants
const WCAG_REQUIREMENTS = {
  AA: {
    normalText: 4.5,      // 18px or smaller, or 14px bold
    largeText: 3.0,       // 24px or larger, or 18px bold
    uiComponents: 3.0,    // Form inputs, icons, etc.
    graphicalObjects: 3.0 // Charts, diagrams, etc.
  },
  AAA: {
    normalText: 7.0,
    largeText: 4.5,
    uiComponents: 3.0,
    graphicalObjects: 3.0
  }
};

// Utility function to check compliance
function checkWCAGCompliance(
  ratio: number,
  fontSize: number,
  fontWeight: number = 400,
  level: 'AA' | 'AAA' = 'AA'
): { passes: boolean; level: string; requirement: number } {
  const isLargeText =
    (fontSize >= 24) ||
    (fontSize >= 18 && fontWeight >= 700);

  const requirement = isLargeText
    ? WCAG_REQUIREMENTS[level].largeText
    : WCAG_REQUIREMENTS[level].normalText;

  return {
    passes: ratio >= requirement,
    level: isLargeText ? 'Large Text' : 'Normal Text',
    requirement
  };
}

// Angular: WCAG compliance pipe
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'wcagCompliance' })
export class WcagCompliancePipe implements PipeTransform {
  transform(
    ratio: number,
    fontSize: number,
    fontWeight: number = 400,
    level: 'AA' | 'AAA' = 'AA'
  ): string {
    const result = checkWCAGCompliance(ratio, fontSize, fontWeight, level);
    
    return result.passes
      ? `✓ WCAG ${level} Pass (${ratio.toFixed(2)}:1 ≥ ${result.requirement}:1)`
      : `✗ WCAG ${level} Fail (${ratio.toFixed(2)}:1 < ${result.requirement}:1)`;
  }
}
```

### Exceptions to Contrast Requirements

```typescript
// React: Elements that don't require contrast
const ContrastExceptions: React.FC = () => {
  return (
    <div>
      {/* Logotypes and brand names */}
      <h1 className="logo" style={{ color: '#00A0E3' }}>
        Brand Name
        {/* Logo text is exempt from contrast requirements */}
      </h1>

      {/* Inactive/disabled UI components */}
      <button disabled style={{ color: '#999', backgroundColor: '#DDD' }}>
        Disabled Button
        {/* Disabled elements are exempt */}
      </button>

      {/* Incidental text */}
      <img src="photo.jpg" alt="Team photo with incidental text in background" />
      {/* Text in photographs is exempt */}

      {/* Decorative elements */}
      <div className="decorative-divider" style={{ background: '#EEE' }}>
        {/* Purely decorative elements without text are exempt */}
      </div>

      {/* Large-scale text and images */}
      <h1 style={{ fontSize: '48px', fontWeight: 'bold', color: '#777' }}>
        Large Heading
        {/* Large text has lower requirements (3:1 for AA) */}
      </h1>
    </div>
  );
};
```

## Color Blindness

### Types of Color Vision Deficiency

```typescript
// React: Color blindness simulation
const ColorBlindnessSimulation: React.FC = () => {
  const [mode, setMode] = React.useState<string>('normal');

  const filters = {
    normal: 'none',
    protanopia: 'url(#protanopia)',      // Red-blind
    deuteranopia: 'url(#deuteranopia)',  // Green-blind
    tritanopia: 'url(#tritanopia)',      // Blue-blind
    achromatopsia: 'grayscale(100%)'     // Complete color blindness
  };

  return (
    <div>
      <svg style={{ height: 0 }}>
        <defs>
          {/* Protanopia filter (red-blind) */}
          <filter id="protanopia">
            <feColorMatrix
              type="matrix"
              values="0.567, 0.433, 0,     0, 0
                      0.558, 0.442, 0,     0, 0
                      0,     0.242, 0.758, 0, 0
                      0,     0,     0,     1, 0"
            />
          </filter>

          {/* Deuteranopia filter (green-blind) */}
          <filter id="deuteranopia">
            <feColorMatrix
              type="matrix"
              values="0.625, 0.375, 0,   0, 0
                      0.7,   0.3,   0,   0, 0
                      0,     0.3,   0.7, 0, 0
                      0,     0,     0,   1, 0"
            />
          </filter>

          {/* Tritanopia filter (blue-blind) */}
          <filter id="tritanopia">
            <feColorMatrix
              type="matrix"
              values="0.95, 0.05,  0,     0, 0
                      0,    0.433, 0.567, 0, 0
                      0,    0.475, 0.525, 0, 0
                      0,    0,     0,     1, 0"
            />
          </filter>
        </defs>
      </svg>

      <div className="controls">
        <select value={mode} onChange={(e) => setMode(e.target.value)}>
          <option value="normal">Normal Vision</option>
          <option value="protanopia">Protanopia (Red-Blind)</option>
          <option value="deuteranopia">Deuteranopia (Green-Blind)</option>
          <option value="tritanopia">Tritanopia (Blue-Blind)</option>
          <option value="achromatopsia">Achromatopsia (No Color)</option>
        </select>
      </div>

      <div style={{ filter: filters[mode as keyof typeof filters] }}>
        <ColorPalette />
      </div>
    </div>
  );
};

/*
Color Blindness Statistics:
- Protanopia/Protanomaly (Red): ~1% of males
- Deuteranopia/Deuteranomaly (Green): ~6% of males
- Tritanopia/Tritanomaly (Blue): ~0.001% of population
- Complete (Achromatopsia): Very rare

Key Insight: Never rely on color alone to convey information!
*/
```

### Don't Rely on Color Alone

```typescript
// ❌ WRONG: Color-only status indication
const BadStatusIndicator: React.FC<{ status: string }> = ({ status }) => {
  const colors = {
    success: '#28A745',
    warning: '#FFC107',
    error: '#DC3545'
  };

  return (
    <div style={{ color: colors[status as keyof typeof colors] }}>
      Status: {status}
    </div>
  );
};

// ✅ CORRECT: Color + icon + text
const GoodStatusIndicator: React.FC<{ status: string }> = ({ status }) => {
  const statusConfig = {
    success: {
      color: '#28A745',
      icon: '✓',
      text: 'Success',
      ariaLabel: 'Success status'
    },
    warning: {
      color: '#FFC107',
      icon: '⚠',
      text: 'Warning',
      ariaLabel: 'Warning status'
    },
    error: {
      color: '#DC3545',
      icon: '✗',
      text: 'Error',
      ariaLabel: 'Error status'
    }
  };

  const config = statusConfig[status as keyof typeof statusConfig];

  return (
    <div
      style={{ color: config.color }}
      role="status"
      aria-label={config.ariaLabel}
    >
      <span aria-hidden="true">{config.icon}</span>
      {' '}Status: {config.text}
    </div>
  );
};
```

```typescript
// Angular: Accessible form validation
@Component({
  selector: 'app-form-field',
  template: `
    <div class="form-field" [class.error]="hasError" [class.success]="isValid">
      <label [for]="id">
        {{ label }}
        <span *ngIf="required" aria-label="required">*</span>
      </label>

      <input
        [id]="id"
        [type]="type"
        [formControl]="control"
        [attr.aria-invalid]="hasError"
        [attr.aria-describedby]="hasError ? id + '-error' : null"
      />

      <!-- Error state: Color + icon + text -->
      <div
        *ngIf="hasError"
        [id]="id + '-error'"
        class="error-message"
        role="alert"
      >
        <span class="icon" aria-hidden="true">⚠</span>
        <span class="text">{{ errorMessage }}</span>
      </div>

      <!-- Success state: Color + icon + text -->
      <div *ngIf="isValid" class="success-message">
        <span class="icon" aria-hidden="true">✓</span>
        <span class="text">Looks good!</span>
      </div>
    </div>
  `,
  styles: [`
    .error {
      border-color: #DC3545;
    }

    .error-message {
      color: #DC3545;
      display: flex;
      align-items: center;
      gap: 8px;
    }

    .success {
      border-color: #28A745;
    }

    .success-message {
      color: #28A745;
      display: flex;
      align-items: center;
      gap: 8px;
    }

    /* Ensure icons are visible even without color */
    .icon {
      font-weight: bold;
      font-size: 1.2em;
    }
  `]
})
export class FormFieldComponent {
  @Input() id!: string;
  @Input() label!: string;
  @Input() type: string = 'text';
  @Input() control!: FormControl;
  @Input() required: boolean = false;

  get hasError(): boolean {
    return this.control.invalid && this.control.touched;
  }

  get isValid(): boolean {
    return this.control.valid && this.control.touched;
  }

  get errorMessage(): string {
    // Return appropriate error message
    return 'This field is required';
  }
}
```

## Testing Tools

### Browser DevTools

```typescript
// React: Using Chrome DevTools for contrast testing
const ContrastTestingGuide: React.FC = () => {
  return (
    <div>
      <h2>Browser Testing Steps</h2>

      <section>
        <h3>Chrome DevTools</h3>
        <ol>
          <li>Right-click element → Inspect</li>
          <li>In Elements panel, find color values</li>
          <li>Click color swatch to open color picker</li>
          <li>Contrast ratio displayed automatically</li>
          <li>Checkmarks show AA/AAA compliance</li>
          <li>Use "Show more" for suggested colors</li>
        </ol>

        <h4>Rendering Emulation:</h4>
        <ol>
          <li>Open DevTools → More tools → Rendering</li>
          <li>Enable "Emulate vision deficiencies"</li>
          <li>Select: Protanopia, Deuteranopia, Tritanopia, or Achromatopsia</li>
          <li>View your site with simulated color blindness</li>
        </ol>
      </section>

      <section>
        <h3>Firefox DevTools</h3>
        <ol>
          <li>Right-click → Inspect Element</li>
          <li>Accessibility panel → Check for issues</li>
          <li>Contrast ratio shown for text elements</li>
          <li>Use Accessibility Simulator for vision deficiencies</li>
        </ol>
      </section>
    </div>
  );
};
```

### Automated Testing

```typescript
// React: Automated contrast testing with jest-axe
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('Button component', () => {
  it('should have sufficient color contrast', async () => {
    const { container } = render(
      <button style={{ color: '#000', backgroundColor: '#FFF' }}>
        Click Me
      </button>
    );

    const results = await axe(container, {
      rules: {
        'color-contrast': { enabled: true }
      }
    });

    expect(results).toHaveNoViolations();
  });

  it('should fail with insufficient contrast', async () => {
    const { container } = render(
      <button style={{ color: '#777', backgroundColor: '#999' }}>
        Click Me
      </button>
    );

    const results = await axe(container);

    // Expect violation
    expect(results.violations).toHaveLength(1);
    expect(results.violations[0].id).toBe('color-contrast');
  });
});
```

```typescript
// Angular: E2E contrast testing with Playwright
import { test, expect } from '@playwright/test';
import { injectAxe, checkA11y } from 'axe-playwright';

test.describe('Accessibility tests', () => {
  test('homepage should have no contrast violations', async ({ page }) => {
    await page.goto('http://localhost:4200');
    await injectAxe(page);

    await checkA11y(page, null, {
      detailedReport: true,
      detailedReportOptions: {
        html: true
      },
      rules: {
        'color-contrast': { enabled: true }
      }
    });
  });

  test('button colors should meet WCAG AA', async ({ page }) => {
    await page.goto('http://localhost:4200');

    const button = page.locator('button.primary');
    const color = await button.evaluate((el) => {
      const styles = window.getComputedStyle(el);
      return {
        color: styles.color,
        backgroundColor: styles.backgroundColor
      };
    });

    // Custom contrast check logic
    // (simplified - use actual contrast calculation)
    expect(color.color).not.toBe(color.backgroundColor);
  });
});
```

## Accessible Color Palettes

### Building an Accessible Palette

```typescript
// React: Accessible color system
const AccessibleColorSystem = {
  // Primary colors with accessible variations
  primary: {
    50: '#E3F2FD',   // Very light - background
    100: '#BBDEFB',  // Light - hover states
    200: '#90CAF9',  // Medium light
    300: '#64B5F6',  // Medium
    400: '#42A5F5',  // Medium dark
    500: '#2196F3',  // Base - sufficient contrast on white
    600: '#1E88E5',  // Dark
    700: '#1976D2',  // Darker
    800: '#1565C0',  // Very dark
    900: '#0D47A1'   // Darkest - sufficient contrast on light backgrounds
  },

  // Semantic colors (all WCAG AA compliant on white)
  semantic: {
    success: '#2E7D32',    // 4.5:1 on white
    warning: '#F57C00',    // 4.5:1 on white
    error: '#C62828',      // 6.4:1 on white
    info: '#1976D2'        // 5.3:1 on white
  },

  // Text colors
  text: {
    primary: '#212121',     // 16.1:1 on white (AAA)
    secondary: '#757575',   // 4.6:1 on white (AA)
    disabled: '#9E9E9E',    // 2.8:1 - okay for disabled
    hint: '#BDBDBD'         // For non-essential text
  },

  // Background colors
  background: {
    default: '#FFFFFF',
    paper: '#FAFAFA',
    elevated: '#F5F5F5'
  }
};

// Usage in component
const AccessibleButton: React.FC<{
  variant: 'primary' | 'success' | 'error';
}> = ({ variant, children }) => {
  const styles: Record<string, React.CSSProperties> = {
    primary: {
      backgroundColor: AccessibleColorSystem.primary[600],
      color: '#FFFFFF'  // 4.5:1+ on all primary shades
    },
    success: {
      backgroundColor: AccessibleColorSystem.semantic.success,
      color: '#FFFFFF'  // 5.8:1 ratio
    },
    error: {
      backgroundColor: AccessibleColorSystem.semantic.error,
      color: '#FFFFFF'  // 7.2:1 ratio
    }
  };

  return (
    <button style={styles[variant]}>
      {children}
    </button>
  );
};
```

### Dark Mode Considerations

```typescript
// React: Accessible dark mode implementation
const DarkModeColorSystem = {
  light: {
    background: '#FFFFFF',
    surface: '#F5F5F5',
    text: {
      primary: '#212121',    // 16.1:1 on white
      secondary: '#757575'   // 4.6:1 on white
    },
    primary: '#1976D2'       // 5.3:1 on white
  },
  dark: {
    background: '#121212',
    surface: '#1E1E1E',
    text: {
      primary: '#FFFFFF',    // 15.8:1 on #121212
      secondary: '#B3B3B3'   // 4.6:1 on #121212
    },
    primary: '#90CAF9'       // 6.5:1 on #121212
  }
};

const ThemedComponent: React.FC = () => {
  const [theme, setTheme] = React.useState<'light' | 'dark'>('light');
  const colors = DarkModeColorSystem[theme];

  return (
    <div
      style={{
        backgroundColor: colors.background,
        color: colors.text.primary,
        minHeight: '100vh',
        padding: '20px'
      }}
    >
      <button
        onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}
        style={{
          backgroundColor: colors.primary,
          color: theme === 'light' ? '#FFFFFF' : '#000000',
          padding: '10px 20px',
          border: 'none',
          borderRadius: '4px',
          cursor: 'pointer'
        }}
      >
        Toggle Theme
      </button>

      <h1>Primary Heading</h1>
      <p style={{ color: colors.text.secondary }}>
        Secondary text with proper contrast in both modes
      </p>
    </div>
  );
};
```

```typescript
// Angular: CSS custom properties for theme
@Component({
  selector: 'app-themed',
  template: `
    <div class="container" [attr.data-theme]="theme">
      <button (click)="toggleTheme()">Toggle Theme</button>
      <h1>Primary Heading</h1>
      <p class="secondary-text">Secondary text</p>
    </div>
  `,
  styles: [`
    .container {
      min-height: 100vh;
      padding: 20px;
      background-color: var(--bg-color);
      color: var(--text-primary);
    }

    .container[data-theme="light"] {
      --bg-color: #FFFFFF;
      --surface-color: #F5F5F5;
      --text-primary: #212121;
      --text-secondary: #757575;
      --primary-color: #1976D2;
      --primary-text: #FFFFFF;
    }

    .container[data-theme="dark"] {
      --bg-color: #121212;
      --surface-color: #1E1E1E;
      --text-primary: #FFFFFF;
      --text-secondary: #B3B3B3;
      --primary-color: #90CAF9;
      --primary-text: #000000;
    }

    button {
      background-color: var(--primary-color);
      color: var(--primary-text);
      padding: 10px 20px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }

    .secondary-text {
      color: var(--text-secondary);
    }
  `]
})
export class ThemedComponent {
  theme: 'light' | 'dark' = 'light';

  toggleTheme() {
    this.theme = this.theme === 'light' ? 'dark' : 'dark';
  }
}
```

## Implementation Patterns

### Dynamic Contrast Adjustment

```typescript
// React: Auto-adjusting text color based on background
function getContrastingTextColor(backgroundColor: string): string {
  // Parse hex color
  const hex = backgroundColor.replace('#', '');
  const r = parseInt(hex.substr(0, 2), 16);
  const g = parseInt(hex.substr(2, 2), 16);
  const b = parseInt(hex.substr(4, 2), 16);

  // Calculate relative luminance
  const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255;

  // Return black or white based on luminance
  return luminance > 0.5 ? '#000000' : '#FFFFFF';
}

const DynamicCard: React.FC<{ backgroundColor: string }> = ({
  backgroundColor,
  children
}) => {
  const textColor = getContrastingTextColor(backgroundColor);

  return (
    <div
      style={{
        backgroundColor,
        color: textColor,
        padding: '20px',
        borderRadius: '8px'
      }}
    >
      {children}
    </div>
  );
};

// Usage
<DynamicCard backgroundColor="#FF5733">
  Text automatically adjusts to white for readability
</DynamicCard>

<DynamicCard backgroundColor="#FFE5B4">
  Text automatically adjusts to black for readability
</DynamicCard>
```

### Link Contrast

```typescript
// React: Accessible link styling
const AccessibleLinks: React.FC = () => {
  return (
    <div style={{ padding: '20px', backgroundColor: '#FFFFFF' }}>
      <p>
        This is body text with a{' '}
        <a
          href="#"
          style={{
            color: '#0366D6',        // 5.0:1 on white (AA compliant)
            textDecoration: 'underline'  // Not relying on color alone
          }}
        >
          standard link
        </a>
        {' '}that meets WCAG AA.
      </p>

      <p>
        Links should be{' '}
        <a
          href="#"
          style={{
            color: '#0366D6',
            textDecoration: 'underline',
            fontWeight: 600  // Additional differentiation
          }}
        >
          distinguishable from body text
        </a>
        {' '}by more than just color.
      </p>

      {/* Focus and hover states */}
      <style>{`
        a:hover {
          background-color: #E6F2FF;  /* High contrast hover state */
          color: #044289;
        }

        a:focus {
          outline: 2px solid #0366D6;
          outline-offset: 2px;
          background-color: #E6F2FF;
        }
      `}</style>
    </div>
  );
};
```

## Common Mistakes

### 1. Low Contrast Text

```typescript
// ❌ WRONG: Insufficient contrast
<div style={{ backgroundColor: '#FFFFFF' }}>
  <p style={{ color: '#CCCCCC' }}>
    This text has only 1.6:1 contrast ratio - unreadable!
  </p>
</div>

// ✅ CORRECT: Sufficient contrast
<div style={{ backgroundColor: '#FFFFFF' }}>
  <p style={{ color: '#595959' }}>
    This text has 7.0:1 contrast ratio - highly readable!
  </p>
</div>
```

### 2. Transparent Overlays

```typescript
// ❌ WRONG: Text over image without ensuring contrast
const BadImageOverlay = () => (
  <div style={{ position: 'relative' }}>
    <img src="photo.jpg" alt="Background" />
    <div style={{
      position: 'absolute',
      top: 0,
      color: '#FFFFFF',  // May not have contrast on light images
      padding: '20px'
    }}>
      Overlay Text
    </div>
  </div>
);

// ✅ CORRECT: Ensure contrast with scrim/overlay
const GoodImageOverlay = () => (
  <div style={{ position: 'relative' }}>
    <img src="photo.jpg" alt="Background" />
    <div style={{
      position: 'absolute',
      top: 0,
      color: '#FFFFFF',
      padding: '20px',
      background: 'linear-gradient(to bottom, rgba(0,0,0,0.7), transparent)',
      // or
      textShadow: '2px 2px 4px rgba(0,0,0,0.8)'  // Ensures readability
    }}>
      Overlay Text
    </div>
  </div>
);
```

### 3. Placeholder Text Contrast

```typescript
// ❌ WRONG: Low contrast placeholders
<input
  type="text"
  placeholder="Enter your email"
  style={{
    '::placeholder': {
      color: '#CCCCCC'  // Too light!
    }
  }}
/>

// ✅ CORRECT: Accessible placeholder
const AccessibleInput = () => (
  <div>
    <label htmlFor="email" style={{ color: '#212121' }}>
      Email Address
    </label>
    <input
      id="email"
      type="email"
      placeholder="name@example.com"
      style={{
        color: '#212121'
      }}
    />
    <style>{`
      input::placeholder {
        color: #757575;  /* 4.6:1 contrast - AA compliant */
      }
    `}</style>
  </div>
);
```

### 4. Icon-Only Buttons Without Labels

```typescript
// ❌ WRONG: Color as only indicator
<button style={{ color: '#28A745' }}>
  ✓
</button>

// ✅ CORRECT: Text label + color
<button
  aria-label="Mark as complete"
  style={{
    color: '#28A745',
    backgroundColor: '#FFFFFF',
    border: '2px solid #28A745'
  }}
>
  <span aria-hidden="true">✓</span>
  <span className="sr-only">Mark as complete</span>
</button>
```

## Best Practices

### 1. Design with Contrast in Mind

```typescript
// React: Contrast-first design system
const DesignTokens = {
  // Define contrast-compliant colors from the start
  colors: {
    // Text on white background
    'text-primary-on-light': '#1A1A1A',      // 16.5:1
    'text-secondary-on-light': '#4A4A4A',    // 9.7:1
    'text-tertiary-on-light': '#737373',     // 4.7:1

    // Text on dark background
    'text-primary-on-dark': '#FFFFFF',       // 16.5:1
    'text-secondary-on-dark': '#D4D4D4',     // 9.7:1
    'text-tertiary-on-dark': '#A3A3A3',      // 4.7:1

    // Interactive elements
    'interactive-primary': '#0066CC',        // 7.3:1 on white
    'interactive-hover': '#0052A3',          // 9.2:1 on white
    'interactive-active': '#003D7A'          // 11.3:1 on white
  },

  // Pre-tested color combinations
  combinations: {
    primary: {
      background: '#FFFFFF',
      text: '#1A1A1A',
      link: '#0066CC',
      border: '#D4D4D4'
    },
    inverse: {
      background: '#1A1A1A',
      text: '#FFFFFF',
      link: '#66B3FF',
      border: '#4A4A4A'
    }
  }
};
```

### 2. Test Early and Often

```typescript
// React: Contrast testing in development
const ContrastWarning: React.FC<{
  foreground: string;
  background: string;
}> = ({ foreground, background }) => {
  const ratio = getContrastRatio(foreground, background);
  const isDevMode = process.env.NODE_ENV === 'development';

  if (isDevMode && ratio < 4.5) {
    console.warn(
      `Low contrast detected! ${foreground} on ${background} = ${ratio.toFixed(2)}:1` +
      `\nMinimum required: 4.5:1 for WCAG AA`
    );
  }

  return null;
};

// Usage in development
const Button = ({ style, children }) => (
  <>
    {process.env.NODE_ENV === 'development' && (
      <ContrastWarning
        foreground={style.color}
        background={style.backgroundColor}
      />
    )}
    <button style={style}>{children}</button>
  </>
);
```

### 3. Document Color Decisions

```typescript
// Document why certain colors were chosen
const ColorDocumentation = {
  primary: {
    value: '#0066CC',
    contrastOnWhite: '7.3:1',
    wcagLevel: 'AAA',
    usage: 'Primary CTA buttons, links, brand elements',
    accessible: true,
    notes: 'Tested against protanopia and deuteranopia'
  },
  success: {
    value: '#2E7D32',
    contrastOnWhite: '4.6:1',
    wcagLevel: 'AA',
    usage: 'Success messages, confirmation icons',
    accessible: true,
    notes: 'Paired with checkmark icon, not relying on color alone'
  }
};
```

## Interview Questions

### Junior Level

1. **What is color contrast and why does it matter?**
   - Difference between text and background colors
   - Ensures readability for all users
   - Required by WCAG standards
   - Critical for low vision users

2. **What is the minimum contrast ratio for WCAG AA compliance?**
   - 4.5:1 for normal text
   - 3:1 for large text (18pt+ or 14pt+ bold)
   - 3:1 for UI components

3. **What are the common types of color blindness?**
   - Protanopia (red-blind)
   - Deuteranopia (green-blind)
   - Tritanopia (blue-blind)
   - Achromatopsia (complete color blindness)

### Mid Level

4. **How would you test for color contrast issues?**
   - Browser DevTools (Chrome/Firefox)
   - Automated tools (axe, Lighthouse)
   - Manual testing with color picker
   - Color blindness simulators
   - Real user testing

5. **Why shouldn't you rely on color alone to convey information?**
   - Color blind users can't distinguish colors
   - Screen reader users don't see colors
   - Printed in black and white
   - Cultural color meanings vary
   - Always add icons, labels, or patterns

6. **How do you handle contrast in dark mode?**
   - Adjust color palette for dark backgrounds
   - Test new contrast ratios
   - Use lighter shades of colors
   - Maintain minimum 4.5:1 ratios
   - Test with actual users

### Senior Level

7. **Design a system for automatically ensuring color contrast across a design system**
   - Programmatic contrast calculation
   - Design token validation
   - CI/CD integration
   - Developer warnings in development
   - Automated testing in E2E tests
   - Documentation generation

8. **How would you handle dynamic user-generated colors while maintaining accessibility?**
   - Calculate contrast ratios in real-time
   - Auto-adjust text color based on background
   - Provide color picker with compliant options only
   - Show contrast warnings to users
   - Fallback to compliant alternatives
   - Save successful combinations

9. **Explain the relationship between color contrast and cognitive load**
   - Low contrast increases eye strain
   - Forces more mental effort to read
   - Reduces reading speed
   - Higher contrast = lower cognitive load
   - But too high contrast can also strain
   - Balance is key

10. **How would you approach accessibility for data visualizations?**
    - Don't rely on color alone for data
    - Use patterns, shapes, labels
    - Provide alternative text descriptions
    - Use colorblind-friendly palettes
    - Provide data tables as alternatives
    - Test with accessibility tools
    - Include chart descriptions

## Key Takeaways

1. **Minimum contrast ratios**: 4.5:1 (normal text), 3:1 (large text) for WCAG AA
2. **Never rely on color alone** to convey information
3. **Test with real tools** - browser DevTools, automated scanners, simulators
4. **Design systems should include** pre-tested, compliant color combinations
5. **Consider all users**: color blind, low vision, aging eyes, mobile in sunlight
6. **Dark mode requires different** contrast considerations
7. **Link colors must meet** contrast requirements against background
8. **Icons need text alternatives** - aria-label or sr-only text
9. **Test early and often** - don't wait until the end
10. **Document color decisions** - why colors were chosen, their contrast ratios

## Resources

### Tools
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- [Colorable](https://colorable.jxnblk.com/) - Color combination tester
- [Contrast Grid](https://contrast-grid.eightshapes.com/) - Test multiple combinations
- [Who Can Use](https://whocanuse.com/) - Real-world contrast testing
- [Coblis Color Blindness Simulator](https://www.color-blindness.com/coblis-color-blindness-simulator/)

### Color Palette Generators
- [Adobe Color](https://color.adobe.com/create/color-accessibility) - Accessible palette generator
- [Coolors](https://coolors.co/) - With contrast checker
- [Material Design Color Tool](https://material.io/resources/color/)

### Documentation
- [WCAG 2.1 - Contrast (Minimum)](https://www.w3.org/WAI/WCAG21/Understanding/contrast-minimum.html)
- [WCAG 2.1 - Contrast (Enhanced)](https://www.w3.org/WAI/WCAG21/Understanding/contrast-enhanced.html)
- [WebAIM: Contrast and Color Accessibility](https://webaim.org/articles/contrast/)

### Articles
- [A Comprehensive Guide to Color Accessibility](https://www.smashingmagazine.com/2022/09/guide-color-accessibility/)
- [Designing for Color Blindness](https://www.uxmatters.com/mt/archives/2020/07/designing-for-color-blindness.php)
- [The Myths of Color Contrast Accessibility](https://uxmovement.com/buttons/the-myths-of-color-contrast-accessibility/)
