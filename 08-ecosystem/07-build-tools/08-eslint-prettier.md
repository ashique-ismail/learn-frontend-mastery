# ESLint + Prettier

## The Idea

**In plain English:** ESLint and Prettier are two tools that automatically check and tidy your code — ESLint acts like a proofreader that catches mistakes and bad habits, while Prettier acts like an auto-formatter that makes sure all your code looks neat and consistent without you having to think about it.

**Real-world analogy:** Imagine a school essay going through two separate helpers before it gets handed in. A grammar checker (like Grammarly) flags actual errors — run-on sentences, missing words, wrong tense — and warns you to fix them. Then a style formatter sets every paragraph to the same font, size, and margin so the whole document looks uniform.

- The grammar checker = ESLint (finds real problems: unused variables, missing dependencies, risky patterns)
- The style formatter = Prettier (enforces appearance: indentation, quote style, line length)
- The finished, corrected essay = your clean, consistent, error-free codebase

---

## ESLint: Linting (Code Quality)

ESLint enforces code quality rules — potential bugs, style conventions, best practices.

### Flat Config (ESLint v9+)

```js
// eslint.config.js (new flat config format)
import js from '@eslint/js';
import typescript from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';

export default [
  // Base JS rules
  js.configs.recommended,

  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      parser: tsParser,
      parserOptions: { project: './tsconfig.json' },
    },
    plugins: {
      '@typescript-eslint': typescript,
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      // TypeScript rules
      ...typescript.configs.recommended.rules,
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],

      // React hooks rules
      ...reactHooks.configs.recommended.rules,
      'react-hooks/exhaustive-deps': 'warn',

      // React Refresh (Vite HMR)
      'react-refresh/only-export-components': 'warn',
    },
  },

  // Ignore patterns
  { ignores: ['dist/**', 'node_modules/**', '*.config.js'] },
];
```

---

## Legacy Config Format (ESLint v8 and below)

```json
// .eslintrc.json (legacy)
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended"
  ],
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint", "react-hooks"],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error"
  }
}
```

---

## Prettier: Formatting

Prettier handles *how* code looks — indentation, quotes, semicolons, line length. It's opinionated and has few options by design.

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "arrowParens": "always"
}
```

---

## ESLint + Prettier Integration

ESLint and Prettier can conflict (ESLint has style rules, Prettier has style rules). Solution: disable ESLint's style rules that Prettier handles.

```bash
npm install -D eslint-config-prettier
```

```js
// eslint.config.js
import prettier from 'eslint-config-prettier';

export default [
  js.configs.recommended,
  // ... other configs
  prettier, // LAST — disables all ESLint rules that conflict with Prettier
];
```

**OR** use `eslint-plugin-prettier` to run Prettier as an ESLint rule (shows Prettier issues as ESLint errors). Less common now, as running them separately is cleaner.

---

## Key Plugin Ecosystem

```bash
# TypeScript
npm install -D @typescript-eslint/parser @typescript-eslint/eslint-plugin

# React
npm install -D eslint-plugin-react-hooks eslint-plugin-react
npm install -D eslint-plugin-react-refresh  # for Vite

# Imports
npm install -D eslint-plugin-import  # import ordering, unused imports
npm install -D eslint-import-resolver-typescript

# Accessibility
npm install -D eslint-plugin-jsx-a11y

# Testing
npm install -D eslint-plugin-vitest  # or eslint-plugin-jest
```

---

## Running Together

```json
// package.json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

---

## With lint-staged (Pre-commit)

```json
// .lintstagedrc.json — only lint/format staged files
{
  "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
  "*.{json,md,css}": ["prettier --write"]
}
```

```json
// package.json — husky hook
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  }
}
```

---

## Common Interview Questions

**Q: What's the difference between ESLint and Prettier?**
ESLint is a linter — it catches bugs and enforces code quality rules (no-unused-vars, react-hooks/exhaustive-deps). Prettier is a formatter — it handles code appearance (indentation, quotes, line breaks). They complement each other.

**Q: Why use `eslint-config-prettier`?**
Without it, ESLint's formatting rules and Prettier's output can conflict, causing ESLint to flag code that Prettier formats correctly. `eslint-config-prettier` turns off all ESLint rules that might conflict with Prettier.

**Q: What changed in ESLint v9 flat config?**
The new flat config (`eslint.config.js`) replaces the cascade of `.eslintrc` files. All configuration is in one array of config objects. Simpler override model, easier to understand what rules apply where. The old `extends` mechanism is replaced by spreading config arrays.
