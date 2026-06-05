# semantic-release

## The Idea

**In plain English:** semantic-release is a tool that automatically figures out what version number your software should get and publishes it — all based on the descriptions you wrote when you saved your code changes. A "version number" is like an edition label (e.g., 1.0.0) that tells users how much has changed since the last release.

**Real-world analogy:** Imagine a newspaper that automatically decides its own edition number based on the type of news submitted by reporters. A reporter files a "BREAKING NEWS" story, and the paper goes from Edition 1 to Edition 2. A reporter files a small correction, and the paper goes from Edition 1.0 to Edition 1.1. The paper also writes its own table of contents and sends itself to subscribers — no editor has to do it manually.

- The reporters filing stories = developers writing commit messages
- The "BREAKING NEWS" vs "correction" labels = commit types like `feat`, `fix`, or `BREAKING CHANGE`
- The edition number = the version number (e.g., 2.0.0, 1.1.0, 1.0.1)

---

## What It Does

semantic-release fully automates the version management and package publishing workflow by analyzing commit messages. It determines version bumps, generates changelogs, creates GitHub releases, and publishes to npm — all in CI, with no manual intervention.

---

## How It Works

1. Analyzes commits since last release (conventional commit format)
2. Determines version bump: `feat` → minor, `fix` → patch, `BREAKING CHANGE` → major
3. Generates `CHANGELOG.md` entry
4. Bumps `package.json` version
5. Creates a git tag
6. Creates a GitHub release
7. Publishes to npm

---

## Setup

```bash
npm install -D semantic-release @semantic-release/changelog @semantic-release/git @semantic-release/github
```

```json
// .releaserc.json (or release.config.js)
{
  "branches": ["main", { "name": "next", "prerelease": true }],
  "plugins": [
    "@semantic-release/commit-analyzer",    // determine version bump
    "@semantic-release/release-notes-generator", // generate changelog content
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    "@semantic-release/npm",                // publish to npm
    ["@semantic-release/git", {            // commit version bump
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }],
    "@semantic-release/github"              // create GitHub release
  ]
}
```

---

## GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main, next]

permissions:
  contents: write
  issues: write
  pull-requests: write
  id-token: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # required for full git history
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci

      - run: npm run build

      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## Commit Message Convention

```
feat: add user authentication        → MINOR bump (0.1.0 → 0.2.0)
fix: resolve login timeout bug       → PATCH bump (0.1.0 → 0.1.1)
docs: update API documentation       → no release
style: format code                   → no release
refactor: restructure auth module    → no release

# BREAKING CHANGE → MAJOR bump (1.0.0 → 2.0.0)
feat!: redesign authentication API

# Or in commit body:
feat: redesign authentication API

BREAKING CHANGE: The `login()` method now requires `{ username, password }` object instead of positional parameters.
```

---

## Release Channels

```json
{
  "branches": [
    "main",                            // → stable (1.2.3)
    { "name": "next", "prerelease": true },      // → 1.3.0-next.1
    { "name": "beta", "prerelease": true },      // → 1.3.0-beta.1
    { "name": "maintenance/1.x", "range": "1.x", "channel": "1.x" }
  ]
}
```

---

## Monorepo with multi-semantic-release

```bash
npm install -D multi-semantic-release
```

```json
// root package.json
{
  "scripts": {
    "release": "multi-semantic-release"
  }
}
```

Runs semantic-release for each package in the workspace, respecting internal dependencies (if `@acme/utils` is released, packages depending on it get their ranges updated).

---

## Comparison with Changesets

| | semantic-release | Changesets |
|---|---|---|
| Version determination | Automatic (commit analysis) | Manual (developer chooses) |
| Release trigger | Push to main | Merge of Version PR |
| Changelog quality | Commit message summaries | Developer-written notes |
| Monorepo support | Via plugin (complex) | First-class |
| Control | Less (automated) | More (explicit) |
| Setup complexity | Moderate | Low |
| Best for | Solo projects, fast CI/CD | Libraries, teams, monorepos |

---

## Common Interview Questions

**Q: What happens if a commit doesn't follow the conventional format?**
semantic-release ignores it for version determination. Non-conventional commits don't cause releases or changelog entries. Teams typically enforce conventional commits via `commitlint` and Husky.

**Q: How do you release a major version?**
Include `BREAKING CHANGE:` in the commit footer, or use the `!` shorthand: `feat!: redesign API`. semantic-release detects this and releases a major version.

**Q: Can semantic-release release without publishing to npm?**
Yes — remove `@semantic-release/npm` from plugins. You can use it only for GitHub releases, git tags, and changelogs without npm publishing.

**Q: What does `[skip ci]` in the release commit message do?**
semantic-release creates a commit to bump version and update the changelog. `[skip ci]` in the message tells your CI system not to run pipelines for that commit — preventing an infinite release loop.
