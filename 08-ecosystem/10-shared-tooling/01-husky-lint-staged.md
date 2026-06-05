# Husky + lint-staged + commitlint

## The Idea

**In plain English:** Husky, lint-staged, and commitlint are tools that automatically check your code and commit messages for problems the moment you try to save your work to git — think of them as a quality gate that runs before your changes are accepted. "Git" is a tool developers use to save and track changes to code over time.

**Real-world analogy:** Imagine a school that checks students' homework before the teacher collects it. A classroom monitor stops each student at the door, quickly scans only the pages they added today (not the whole notebook), and rejects any paper that isn't titled correctly.

- The classroom monitor stopping students = Husky (intercepts git commits via hooks)
- Scanning only today's added pages = lint-staged (checks only the files you changed, not everything)
- The rule that papers must be titled correctly = commitlint (enforces a standard format for commit messages)

---

## What They Do

- **Husky** — runs shell commands on git hooks (pre-commit, commit-msg, pre-push, etc.)
- **lint-staged** — runs linters/formatters only on *staged* files (not the whole codebase)
- **commitlint** — enforces commit message format (conventional commits)

Together: enforce code quality before code enters the repository.

---

## Husky v9 Setup

```bash
npm install -D husky
npx husky init  # creates .husky/ directory with a sample pre-commit hook
```

```bash
# .husky/pre-commit
npx lint-staged

# .husky/commit-msg
npx --no -- commitlint --edit $1
```

---

## lint-staged

Run linters only on changed files — much faster than linting the whole project:

```bash
npm install -D lint-staged
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,css,yml}": [
      "prettier --write"
    ]
  }
}
```

Or as a separate file:

```js
// .lintstagedrc.mjs
export default {
  '*.{ts,tsx}': (filenames) => [
    `eslint --fix ${filenames.join(' ')}`,
    `prettier --write ${filenames.join(' ')}`,
  ],
  '*.{json,md}': 'prettier --write',
};
```

**Why lint-staged over running on all files?**
A codebase with 1000 files takes 30s to lint. lint-staged runs on only the 3 files you changed — takes 1s.

---

## commitlint — Conventional Commits

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
```

```js
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'chore', 'revert', 'ci'
    ]],
    'scope-case': [2, 'always', 'lower-case'],
    'subject-case': [2, 'always', 'sentence-case'],
    'header-max-length': [2, 'always', 72],
  }
};
```

**Conventional Commits format:**
```
type(scope): subject

feat(auth): Add OAuth2 login with GitHub
fix(checkout): Prevent duplicate order submission
docs(api): Update authentication endpoint examples
chore(deps): Bump React from 18.2 to 18.3
```

---

## CI Integration

Husky hooks only run locally. To enforce the same rules in CI:

```yaml
# .github/workflows/ci.yml
- name: Lint
  run: npx eslint . --max-warnings=0

- name: Type check
  run: npx tsc --noEmit

- name: Format check
  run: npx prettier --check .

- name: Commit message lint (for PRs)
  run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to HEAD --verbose
```

---

## Monorepo Setup

```bash
# Root .husky/pre-commit
cd apps/web && npx lint-staged || exit 1
cd apps/mobile && npx lint-staged || exit 1
```

Or use Turborepo to run lint-staged across workspaces:

```bash
# .husky/pre-commit
npx turbo run lint-staged --filter='[HEAD]'
```

---

## Bypass Hooks (Emergency)

```bash
# Skip pre-commit hook (use sparingly)
git commit --no-verify -m "fix: Emergency hotfix"

# Skip commit-msg hook
git commit --no-verify -m "WIP"
```

Document when bypassing is acceptable in team guidelines.

---

## Common Interview Questions

**Q: Why use lint-staged instead of running ESLint on all files in the pre-commit hook?**
Speed. Running ESLint on 500 TypeScript files takes 20-30 seconds. lint-staged only processes the files you're committing — typically 2-10 files — taking under a second. Fast hooks get ignored less.

**Q: What are conventional commits and why do they matter?**
A structured commit message format: `type(scope): description`. Enables automated changelog generation, semantic versioning (tools like `semantic-release` or `changesets` parse commit types to determine version bumps), and readable git history.

**Q: What happens if a lint-staged fix modifies a file? Does it auto-stage the fix?**
In most configurations, no — lint-staged runs the linter/formatter but doesn't automatically re-stage the modified files. The commit fails, you see the changes, then `git add` and commit again. Some setups use `--fix` and then `git add` in the lint-staged config, but this can be risky.
