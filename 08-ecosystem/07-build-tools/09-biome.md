# Biome

## The Idea

**In plain English:** Biome is a tool that automatically checks your code for mistakes and keeps it neatly formatted — like a spell-checker and auto-formatter rolled into one, but for JavaScript and TypeScript. It runs incredibly fast because it is built with a programming language called Rust, which is known for speed.

**Real-world analogy:** Imagine a restaurant kitchen where one inspector walks through, checks that every dish follows the health code (no raw chicken next to salad), and simultaneously arranges every plate so it looks presentable before it leaves the kitchen — all in a single pass, in seconds.

- The inspector = Biome (one tool doing both jobs)
- Checking health-code violations = the linter (finding code mistakes and bad patterns)
- Arranging plates consistently = the formatter (making code look uniform and readable)
- Doing it all in one pass = sharing the same internal code representation (AST) so nothing conflicts

---

## What It Is

Biome is an all-in-one toolchain for JavaScript/TypeScript — **linter + formatter** in a single fast Rust-based tool. Replaces both ESLint and Prettier.

---

## Key Value Proposition

- **10-100x faster** than ESLint + Prettier (Rust vs JavaScript)
- **Single tool** — one config, one install, one command
- **No configuration needed** for basic use
- **Consistent** — formatter and linter share AST (no parse conflicts)
- **No Prettier-ESLint conflicts** (same tool handles both)

---

## Setup

```bash
npm install -D @biomejs/biome
npx biome init  # generates biome.json
```

```json
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.8.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noExtraBooleanCast": "error"
      },
      "correctness": {
        "noUnusedVariables": "warn",
        "useExhaustiveDependencies": "warn"  // like react-hooks/exhaustive-deps
      },
      "security": {
        "noDangerouslySetInnerHtml": "warn"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "trailingCommas": "es5",
      "semicolons": "always"
    }
  },
  "files": {
    "ignore": ["dist/**", "node_modules/**", "*.config.js"]
  }
}
```

---

## Usage

```bash
# Check (lint + format check, no changes)
npx biome check .

# Apply fixes
npx biome check --apply .

# Format only
npx biome format --write .

# Lint only
npx biome lint .

# CI (exits with error code on issues)
npx biome ci .
```

---

## Available Rules

Biome has 200+ lint rules organized into categories:

- **correctness** — likely bugs (`noUnusedVariables`, `noArrayIndexKey`, `useExhaustiveDependencies`)
- **suspicious** — potentially problematic patterns (`noDoubleEquals`, `noExplicitAny`)
- **complexity** — unnecessary complexity (`noExtraBooleanCast`, `useFlatMap`)
- **performance** — performance anti-patterns (`noDelete`, `noAccumulatingSpread`)
- **style** — code style preferences (`useConst`, `noVar`, `useNamingConvention`)
- **security** — security issues (`noDangerouslySetInnerHtml`, `noGlobalEval`)
- **a11y** — accessibility (`useAltText`, `useButtonType`, `noAccessKey`)

---

## Comparison with ESLint + Prettier

| | Biome | ESLint + Prettier |
|---|---|---|
| Speed | 10-100x faster | Slower |
| Config | Single `biome.json` | Multiple files (`.eslintrc`, `.prettierrc`) |
| Setup | Minimal | Complex (plugins, extends, compatibility) |
| Ecosystem | Growing (200+ rules) | Massive (thousands of plugins) |
| TypeScript rules | Built-in | Requires `@typescript-eslint` |
| Import sorting | Built-in | Requires plugin |
| Custom rules | Not supported | Fully supported |
| ESLint plugin compatibility | Not compatible | Full ecosystem |

---

## Migration from ESLint + Prettier

```bash
# Biome provides a migration command
npx biome migrate eslint --write
npx biome migrate prettier --write
# Converts .eslintrc and .prettierrc rules to biome.json equivalents
```

---

## IDE Integration

```json
// .vscode/extensions.json
{ "recommendations": ["biomejs.biome"] }

// .vscode/settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "[javascript][typescript][typescriptreact]": {
    "editor.defaultFormatter": "biomejs.biome"
  }
}
```

---

## Common Interview Questions

**Q: Why is Biome faster than ESLint?**
Biome is written in Rust and operates on a single AST pass. ESLint is JavaScript and runs multiple passes per plugin. Biome also doesn't need a separate parse step for the formatter since linting and formatting share the AST.

**Q: Can Biome fully replace ESLint?**
For most projects, yes. Biome has most of the commonly used ESLint rules. The gap is custom/plugin rules — Biome doesn't support custom rule plugins. For projects needing specialized rules (e.g., company-specific patterns), ESLint may still be needed.

**Q: Does Biome support JSX and TypeScript?**
Yes — both are supported natively. No parser plugins required (unlike ESLint which needs `@typescript-eslint/parser` and React support).
