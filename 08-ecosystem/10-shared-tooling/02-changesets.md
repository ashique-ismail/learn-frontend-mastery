# Changesets

## What It Does

Changesets is a version management and changelog tool designed for JavaScript monorepos. It solves the problem of coordinating version bumps and changelogs across multiple packages that may be changed together.

---

## Core Workflow

```
Developer makes changes → adds a changeset → PR merges → CI bumps versions → publishes
```

1. Developer runs `npx changeset` to describe their changes
2. A `.changeset/*.md` file is created and committed
3. When ready to release, `npx changeset version` bumps versions and updates changelogs
4. `npx changeset publish` publishes to npm

---

## Setup

```bash
npm install -D @changesets/cli
npx changeset init
```

Creates `.changeset/config.json`:
```json
{
  "$schema": "https://unpkg.com/@changesets/config@latest/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "linked": [],
  "access": "restricted",    // or "public" for public packages
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

---

## Adding a Changeset

```bash
npx changeset
```

Interactive CLI:
1. Select which packages changed
2. Select bump type (major / minor / patch)
3. Write a summary

Creates `.changeset/shiny-dogs-jump.md`:
```markdown
---
"@acme/ui": minor
"@acme/utils": patch
---

Add Button component variants and fix color utility bug
```

This file is committed with your PR. It describes the intent, not the action — the actual version bump happens later.

---

## Versioning (on merge/release)

```bash
# Consume all pending changesets, bump versions, update CHANGELOGs
npx changeset version
```

This:
1. Reads all `.changeset/*.md` files
2. Determines the highest bump type for each package
3. Bumps `package.json` versions
4. Updates `CHANGELOG.md` for each changed package
5. Deletes the consumed `.changeset` files

---

## Publishing

```bash
# Build packages first
npm run build

# Publish changed packages to npm
npx changeset publish
```

Or with `--tag` for pre-releases:
```bash
npx changeset publish --tag beta
```

---

## GitHub Actions Integration

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci

      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          publish: npm run release
          version: npm run version
          commit: 'chore: release packages'
          title: 'Release: version bump'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**What this does:**
- If there are pending changesets: opens/updates a "Version Packages" PR
- If the "Version Packages" PR is merged: publishes to npm

---

## Pre-release Versions (Beta/Alpha)

```bash
# Enter pre-release mode
npx changeset pre enter beta

# Add changesets normally
npx changeset add

# Bump to pre-release version (e.g., 1.2.0-beta.0)
npx changeset version

# Publish with beta tag
npx changeset publish --tag beta

# Exit pre-release mode when ready for stable
npx changeset pre exit
```

---

## Comparison with semantic-release

| | Changesets | semantic-release |
|---|---|---|
| Monorepo support | First-class | Via plugin (multi-semantic-release) |
| Version determination | Manual (developer chooses) | Automatic (from commit messages) |
| Workflow | PR-based, explicit | CI-based, automated |
| Changelog quality | Developer-written summaries | Commit messages |
| Control | High | Lower (automatic) |
| Best for | Teams, libraries, monorepos | Solo projects, fast iteration |

---

## Common Interview Questions

**Q: What happens if two PRs add changesets for the same package with different bump types?**
Changesets takes the highest bump type. If PR A adds `minor` and PR B adds `patch` for the same package, the release will be a `minor` bump.

**Q: How do linked packages work?**
`"linked": [["@acme/ui", "@acme/icons"]]` — linked packages always release together with the same version. If either has a changeset, both are bumped to the same new version.

**Q: When would you choose Changesets over conventional commits + semantic-release?**
When you want human-written changelog entries (better user experience), when you have a monorepo with complex interdependencies, or when you want developers to be explicit about the significance of their changes rather than relying on commit message conventions.
