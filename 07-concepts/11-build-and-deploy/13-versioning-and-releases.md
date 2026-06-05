# Versioning and Releases

## The Idea

**In plain English:** Versioning is a way of giving every update to a piece of software a unique number so that you and others always know exactly which copy you are using. A release is when you officially publish one of those numbered versions for others to use.

**Real-world analogy:** Think of a cookbook that gets updated over time. The publisher prints "Edition 2.4.1" on the cover so readers and bookstores always know which copy they have. When a new edition adds a whole new chapter (a big deal), the first number goes up. When a few new recipes are added (smaller addition), the middle number goes up. When a typo is corrected (tiny fix), the last number goes up.

- The edition number on the cover = the version number in software (e.g., 2.4.1)
- Publishing a new edition for sale = making a release
- A brand-new chapter that changes how the whole book is organised = a major version bump (breaking change)
- Adding new recipes without removing old ones = a minor version bump (new feature, backward compatible)
- Fixing a typo = a patch version bump (bug fix)

---

## Overview

Versioning and release management are critical practices for tracking changes, communicating updates, and maintaining software stability. Semantic Versioning (SemVer) provides a standardized approach to version numbering, while proper release processes ensure smooth deployments and clear communication with users and stakeholders.

## Table of Contents

- [Semantic Versioning (SemVer)](#semantic-versioning-semver)
- [Version Numbering Schemes](#version-numbering-schemes)
- [Git Tags and Branches](#git-tags-and-branches)
- [Changelogs](#changelogs)
- [Release Automation](#release-automation)
- [Version Bumping](#version-bumping)
- [Pre-release and Build Metadata](#pre-release-and-build-metadata)
- [Release Strategies](#release-strategies)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [When to Use/Not to Use](#when-to-usenot-to-use)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Semantic Versioning (SemVer)

### Understanding SemVer

Semantic Versioning uses a three-part version number: `MAJOR.MINOR.PATCH`

```
Version: 2.4.7
         │ │ │
         │ │ └── PATCH: Bug fixes (backward compatible)
         │ └──── MINOR: New features (backward compatible)
         └────── MAJOR: Breaking changes (not backward compatible)
```

### SemVer Rules

```javascript
// package.json
{
  "name": "my-library",
  "version": "1.5.3",
  "description": "Version format: MAJOR.MINOR.PATCH"
}

// Version increment examples:
// 1.5.3 → 1.5.4  (Bug fix, backward compatible)
// 1.5.4 → 1.6.0  (New feature, backward compatible)
// 1.6.0 → 2.0.0  (Breaking change)

// Pre-release versions:
// 1.0.0-alpha.1
// 1.0.0-beta.2
// 1.0.0-rc.1

// Build metadata:
// 1.0.0+20230615
// 1.0.0+build.123
```

### Version Comparison

```javascript
// version-compare.js
class Version {
  constructor(version) {
    const match = version.match(/^(\d+)\.(\d+)\.(\d+)(?:-([^+]+))?(?:\+(.+))?$/);
    if (!match) throw new Error('Invalid version format');
    
    this.major = parseInt(match[1], 10);
    this.minor = parseInt(match[2], 10);
    this.patch = parseInt(match[3], 10);
    this.prerelease = match[4] || '';
    this.buildMetadata = match[5] || '';
  }
  
  isGreaterThan(other) {
    if (this.major !== other.major) return this.major > other.major;
    if (this.minor !== other.minor) return this.minor > other.minor;
    if (this.patch !== other.patch) return this.patch > other.patch;
    
    // No pre-release is greater than pre-release
    if (!this.prerelease && other.prerelease) return true;
    if (this.prerelease && !other.prerelease) return false;
    
    return this.prerelease > other.prerelease;
  }
  
  toString() {
    let version = `${this.major}.${this.minor}.${this.patch}`;
    if (this.prerelease) version += `-${this.prerelease}`;
    if (this.buildMetadata) version += `+${this.buildMetadata}`;
    return version;
  }
}

// Usage
const v1 = new Version('1.5.3');
const v2 = new Version('2.0.0');
console.log(v1.isGreaterThan(v2)); // false

const v3 = new Version('1.0.0-alpha.1');
const v4 = new Version('1.0.0');
console.log(v3.isGreaterThan(v4)); // false (pre-release < release)
```

### Implementing SemVer in Projects

```typescript
// version.ts
export interface VersionInfo {
  version: string;
  major: number;
  minor: number;
  patch: number;
  prerelease?: string;
  buildNumber?: string;
  gitHash?: string;
  buildDate?: string;
}

export class VersionManager {
  private version: VersionInfo;
  
  constructor() {
    const pkg = require('../package.json');
    const version = this.parseVersion(pkg.version);
    
    this.version = {
      ...version,
      buildNumber: process.env.BUILD_NUMBER,
      gitHash: process.env.GIT_COMMIT?.substring(0, 7),
      buildDate: new Date().toISOString(),
    };
  }
  
  private parseVersion(versionString: string): Omit<VersionInfo, 'buildNumber' | 'gitHash' | 'buildDate'> {
    const match = versionString.match(/^(\d+)\.(\d+)\.(\d+)(?:-([^+]+))?$/);
    if (!match) throw new Error(`Invalid version: ${versionString}`);
    
    return {
      version: versionString,
      major: parseInt(match[1], 10),
      minor: parseInt(match[2], 10),
      patch: parseInt(match[3], 10),
      prerelease: match[4],
    };
  }
  
  getVersion(): VersionInfo {
    return this.version;
  }
  
  getVersionString(): string {
    let version = `${this.version.major}.${this.version.minor}.${this.version.patch}`;
    if (this.version.prerelease) version += `-${this.version.prerelease}`;
    if (this.version.gitHash) version += `+${this.version.gitHash}`;
    return version;
  }
  
  isCompatible(requiredVersion: string): boolean {
    const required = this.parseVersion(requiredVersion);
    
    // Major version must match
    if (this.version.major !== required.major) return false;
    
    // Minor version must be >= required
    if (this.version.minor < required.minor) return false;
    
    // If minor versions match, patch must be >= required
    if (this.version.minor === required.minor && this.version.patch < required.patch) {
      return false;
    }
    
    return true;
  }
}

// Usage in React
import React from 'react';
import { VersionManager } from './version';

const versionManager = new VersionManager();

export const VersionDisplay: React.FC = () => {
  const version = versionManager.getVersion();
  
  return (
    <div>
      <p>Version: {version.version}</p>
      <p>Build: {version.buildNumber}</p>
      <p>Commit: {version.gitHash}</p>
      <p>Date: {version.buildDate}</p>
    </div>
  );
};

// Usage in Angular
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class VersionService {
  private versionManager = new VersionManager();
  
  getVersion(): VersionInfo {
    return this.versionManager.getVersion();
  }
  
  getVersionString(): string {
    return this.versionManager.getVersionString();
  }
  
  checkCompatibility(requiredVersion: string): boolean {
    return this.versionManager.isCompatible(requiredVersion);
  }
}
```

## Version Numbering Schemes

### Calendar Versioning (CalVer)

```javascript
// CalVer formats:
// YYYY.MM.DD - Ubuntu (20.04, 22.04)
// YYYY.0M.0D - Date-based
// YY.MM.MICRO - Twisted

class CalVer {
  static generate(format = 'YYYY.0M.MICRO') {
    const now = new Date();
    const year = now.getFullYear();
    const month = String(now.getMonth() + 1).padStart(2, '0');
    const day = String(now.getDate()).padStart(2, '0');
    
    switch (format) {
      case 'YYYY.MM.DD':
        return `${year}.${month}.${day}`;
      case 'YYYY.0M.MICRO':
        return `${year}.${month}.0`;
      case 'YY.MM.MICRO':
        return `${String(year).slice(-2)}.${month}.0`;
      default:
        throw new Error(`Unknown CalVer format: ${format}`);
    }
  }
}

// Usage
console.log(CalVer.generate('YYYY.MM.DD')); // "2024.06.15"

// package.json with CalVer
{
  "version": "2024.06.0"
}
```

### Other Versioning Schemes

```javascript
// Sequential versioning (simple incrementing)
// v1, v2, v3, v4...

// Date-based versioning
const dateVersion = () => {
  const now = new Date();
  return `${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}`;
};
console.log(dateVersion()); // "20240615"

// Git commit-based versioning
// v1.2.3-45-gd5a2b1c
// Format: <tag>-<commits-since-tag>-g<commit-hash>

// Marketing versions (decoupled from technical versions)
// Technical: 3.14.5
// Marketing: "Summer 2024 Release"
```

## Git Tags and Branches

### Creating Git Tags

```bash
# Lightweight tag (just a pointer to commit)
git tag v1.0.0

# Annotated tag (recommended - includes metadata)
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag with detailed message
git tag -a v1.0.0 -m "Release v1.0.0

Features:
- Added user authentication
- Improved performance
- Fixed security vulnerabilities"

# Tag specific commit
git tag -a v1.0.0 9fceb02 -m "Release version 1.0.0"

# Push tags to remote
git push origin v1.0.0

# Push all tags
git push origin --tags

# List tags
git tag
git tag -l "v1.*"

# View tag details
git show v1.0.0

# Delete tag
git tag -d v1.0.0
git push origin --delete v1.0.0

# Check out specific tag
git checkout v1.0.0
```

### Release Branches

```bash
# Git Flow branching model
main (production)
  ├── develop (integration)
  │   ├── feature/user-auth
  │   ├── feature/new-dashboard
  │   └── bugfix/login-issue
  └── release/v1.2.0
      └── hotfix/security-patch

# Creating release branch
git checkout develop
git checkout -b release/v1.2.0

# Prepare release (bump version, update changelog)
npm version minor
git add .
git commit -m "Bump version to 1.2.0"

# Merge to main and tag
git checkout main
git merge release/v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"

# Merge back to develop
git checkout develop
git merge release/v1.2.0

# Delete release branch
git branch -d release/v1.2.0

# Hotfix workflow
git checkout main
git checkout -b hotfix/v1.2.1
# Fix bug
git commit -m "Fix critical security issue"
git checkout main
git merge hotfix/v1.2.1
git tag -a v1.2.1 -m "Hotfix version 1.2.1"
git checkout develop
git merge hotfix/v1.2.1
```

### Automated Tagging Script

```javascript
// scripts/release.js
const { execSync } = require('child_process');
const fs = require('fs');

function exec(command) {
  return execSync(command, { encoding: 'utf8' }).trim();
}

function getCurrentVersion() {
  const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
  return pkg.version;
}

function hasUncommittedChanges() {
  const status = exec('git status --porcelain');
  return status.length > 0;
}

function tagExists(tag) {
  try {
    exec(`git rev-parse ${tag}`);
    return true;
  } catch {
    return false;
  }
}

function createRelease(version, message) {
  const tag = `v${version}`;
  
  // Validate
  if (hasUncommittedChanges()) {
    throw new Error('Uncommitted changes detected. Commit or stash changes first.');
  }
  
  if (tagExists(tag)) {
    throw new Error(`Tag ${tag} already exists.`);
  }
  
  // Create annotated tag
  exec(`git tag -a ${tag} -m "${message}"`);
  console.log(`Created tag ${tag}`);
  
  // Push tag
  exec(`git push origin ${tag}`);
  console.log(`Pushed tag ${tag} to origin`);
  
  // Push branch
  const branch = exec('git rev-parse --abbrev-ref HEAD');
  exec(`git push origin ${branch}`);
  console.log(`Pushed ${branch} to origin`);
}

// Usage
const version = getCurrentVersion();
const message = process.argv[2] || `Release version ${version}`;

try {
  createRelease(version, message);
  console.log('Release created successfully!');
} catch (error) {
  console.error('Release failed:', error.message);
  process.exit(1);
}
```

## Changelogs

### Conventional Changelog Format

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New user profile customization options

### Changed
- Updated dashboard layout for better UX

### Deprecated
- Old API endpoints (will be removed in v3.0.0)

## [1.2.0] - 2024-06-15

### Added
- User authentication with OAuth2
- Dark mode support
- Real-time notifications
- Export to CSV functionality

### Changed
- Improved search performance by 50%
- Updated UI components to Material Design 3
- Refactored API error handling

### Fixed
- Fixed memory leak in dashboard
- Resolved timezone issues in date picker
- Fixed broken links in documentation

### Security
- Patched XSS vulnerability in user input
- Updated dependencies with security fixes

## [1.1.5] - 2024-06-01

### Fixed
- Critical bug in payment processing
- Login redirect issue on mobile devices

## [1.1.0] - 2024-05-15

### Added
- Multi-language support (EN, ES, FR, DE)
- Advanced filtering options
- Bulk operations

### Changed
- Migrated to React 18
- Updated database schema

### Removed
- Deprecated legacy API endpoints

## [1.0.0] - 2024-04-01

### Added
- Initial release
- User management system
- Dashboard analytics
- RESTful API

[Unreleased]: https://github.com/user/repo/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/user/repo/compare/v1.1.5...v1.2.0
[1.1.5]: https://github.com/user/repo/compare/v1.1.0...v1.1.5
[1.1.0]: https://github.com/user/repo/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/user/repo/releases/tag/v1.0.0
```

### Automated Changelog Generation

```javascript
// scripts/generate-changelog.js
const { execSync } = require('child_process');
const fs = require('fs');

function exec(command) {
  return execSync(command, { encoding: 'utf8' }).trim();
}

function getCommitsSinceTag(tag) {
  try {
    const commits = exec(`git log ${tag}..HEAD --pretty=format:"%H|%s|%an|%ae|%ad" --date=short`);
    return commits.split('\n').map(line => {
      const [hash, subject, author, email, date] = line.split('|');
      return { hash, subject, author, email, date };
    });
  } catch {
    // No previous tag, get all commits
    const commits = exec('git log --pretty=format:"%H|%s|%an|%ae|%ad" --date=short');
    return commits.split('\n').map(line => {
      const [hash, subject, author, email, date] = line.split('|');
      return { hash, subject, author, email, date };
    });
  }
}

function categorizeCommits(commits) {
  const categories = {
    features: [],
    fixes: [],
    breaking: [],
    docs: [],
    chores: [],
    other: [],
  };
  
  commits.forEach(commit => {
    const subject = commit.subject.toLowerCase();
    
    if (subject.startsWith('feat:') || subject.startsWith('feature:')) {
      categories.features.push(commit);
    } else if (subject.startsWith('fix:')) {
      categories.fixes.push(commit);
    } else if (subject.includes('breaking change') || subject.startsWith('breaking:')) {
      categories.breaking.push(commit);
    } else if (subject.startsWith('docs:')) {
      categories.docs.push(commit);
    } else if (subject.startsWith('chore:') || subject.startsWith('refactor:')) {
      categories.chores.push(commit);
    } else {
      categories.other.push(commit);
    }
  });
  
  return categories;
}

function generateChangelogSection(version, date, categories) {
  let changelog = `\n## [${version}] - ${date}\n`;
  
  if (categories.breaking.length > 0) {
    changelog += '\n### Breaking Changes\n';
    categories.breaking.forEach(commit => {
      changelog += `- ${commit.subject.replace(/^breaking:\s*/i, '')} (${commit.hash.substring(0, 7)})\n`;
    });
  }
  
  if (categories.features.length > 0) {
    changelog += '\n### Added\n';
    categories.features.forEach(commit => {
      changelog += `- ${commit.subject.replace(/^feat(ure)?:\s*/i, '')} (${commit.hash.substring(0, 7)})\n`;
    });
  }
  
  if (categories.fixes.length > 0) {
    changelog += '\n### Fixed\n';
    categories.fixes.forEach(commit => {
      changelog += `- ${commit.subject.replace(/^fix:\s*/i, '')} (${commit.hash.substring(0, 7)})\n`;
    });
  }
  
  return changelog;
}

function updateChangelog(newVersion) {
  const latestTag = exec('git describe --tags --abbrev=0 2>/dev/null || echo ""');
  const commits = getCommitsSinceTag(latestTag);
  
  if (commits.length === 0) {
    console.log('No commits since last release');
    return;
  }
  
  const categories = categorizeCommits(commits);
  const date = new Date().toISOString().split('T')[0];
  const newSection = generateChangelogSection(newVersion, date, categories);
  
  const changelogPath = 'CHANGELOG.md';
  let changelog = '';
  
  if (fs.existsSync(changelogPath)) {
    changelog = fs.readFileSync(changelogPath, 'utf8');
    // Insert after header
    const headerEnd = changelog.indexOf('\n## ');
    if (headerEnd !== -1) {
      changelog = changelog.slice(0, headerEnd) + newSection + changelog.slice(headerEnd);
    } else {
      changelog += newSection;
    }
  } else {
    changelog = `# Changelog\n\nAll notable changes to this project will be documented in this file.\n${newSection}`;
  }
  
  fs.writeFileSync(changelogPath, changelog);
  console.log('CHANGELOG.md updated successfully');
}

// Usage
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
updateChangelog(pkg.version);
```

## Release Automation

### GitHub Actions Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
      
      - name: Get version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Generate changelog
        id: changelog
        run: |
          CHANGELOG=$(git log $(git describe --tags --abbrev=0 HEAD^)..HEAD --pretty=format:"- %s (%h)" --no-merges)
          echo "CHANGELOG<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ steps.version.outputs.VERSION }}
          body: |
            ## Changes in this Release
            ${{ steps.changelog.outputs.CHANGELOG }}
            
            ## Installation
            ```bash
            npm install my-package@${{ steps.version.outputs.VERSION }}
            ```
          draft: false
          prerelease: ${{ contains(github.ref, '-') }}
      
      - name: Publish to NPM
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
      
      - name: Deploy to production
        run: |
          echo "Deploying version ${{ steps.version.outputs.VERSION }}"
          # Add deployment commands here
```

### Semantic Release Configuration

```javascript
// .releaserc.js
module.exports = {
  branches: ['main', 'next', { name: 'beta', prerelease: true }],
  plugins: [
    // Analyze commits to determine version bump
    ['@semantic-release/commit-analyzer', {
      preset: 'angular',
      releaseRules: [
        { type: 'docs', scope: 'README', release: 'patch' },
        { type: 'refactor', release: 'patch' },
        { type: 'style', release: 'patch' },
        { scope: 'no-release', release: false },
      ],
    }],
    
    // Generate release notes
    ['@semantic-release/release-notes-generator', {
      preset: 'angular',
      writerOpts: {
        commitsSort: ['subject', 'scope'],
      },
    }],
    
    // Update CHANGELOG.md
    ['@semantic-release/changelog', {
      changelogFile: 'CHANGELOG.md',
    }],
    
    // Update package.json version
    ['@semantic-release/npm', {
      npmPublish: true,
    }],
    
    // Commit updated files
    ['@semantic-release/git', {
      assets: ['CHANGELOG.md', 'package.json', 'package-lock.json'],
      message: 'chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}',
    }],
    
    // Create GitHub release
    ['@semantic-release/github', {
      assets: [
        { path: 'dist/**/*.js', label: 'JS Distribution' },
        { path: 'dist/**/*.css', label: 'CSS Distribution' },
      ],
    }],
  ],
};

// package.json
{
  "scripts": {
    "semantic-release": "semantic-release"
  },
  "devDependencies": {
    "@semantic-release/changelog": "^6.0.0",
    "@semantic-release/commit-analyzer": "^9.0.0",
    "@semantic-release/git": "^10.0.0",
    "@semantic-release/github": "^8.0.0",
    "@semantic-release/npm": "^9.0.0",
    "@semantic-release/release-notes-generator": "^10.0.0",
    "semantic-release": "^19.0.0"
  }
}
```

### Release Script

```bash
#!/bin/bash
# scripts/release.sh

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to print colored output
print_info() {
  echo -e "${GREEN}[INFO]${NC} $1"
}

print_warning() {
  echo -e "${YELLOW}[WARNING]${NC} $1"
}

print_error() {
  echo -e "${RED}[ERROR]${NC} $1"
}

# Check for uncommitted changes
if [[ -n $(git status -s) ]]; then
  print_error "Uncommitted changes detected. Commit or stash changes first."
  exit 1
fi

# Ensure we're on main branch
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [[ "$CURRENT_BRANCH" != "main" ]]; then
  print_error "Releases must be created from 'main' branch. Current branch: $CURRENT_BRANCH"
  exit 1
fi

# Pull latest changes
print_info "Pulling latest changes from origin..."
git pull origin main

# Get current version
CURRENT_VERSION=$(node -p "require('./package.json').version")
print_info "Current version: $CURRENT_VERSION"

# Prompt for version bump type
echo ""
echo "Select version bump type:"
echo "1) Patch (bug fixes): $CURRENT_VERSION -> $(npm version patch --no-git-tag-version --dry-run | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')"
echo "2) Minor (new features): $CURRENT_VERSION -> $(npm version minor --no-git-tag-version --dry-run | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')"
echo "3) Major (breaking changes): $CURRENT_VERSION -> $(npm version major --no-git-tag-version --dry-run | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')"
echo "4) Custom version"
read -p "Enter choice (1-4): " CHOICE

case $CHOICE in
  1)
    BUMP_TYPE="patch"
    ;;
  2)
    BUMP_TYPE="minor"
    ;;
  3)
    BUMP_TYPE="major"
    ;;
  4)
    read -p "Enter custom version (e.g., 1.2.3): " CUSTOM_VERSION
    BUMP_TYPE="$CUSTOM_VERSION"
    ;;
  *)
    print_error "Invalid choice"
    exit 1
    ;;
esac

# Bump version
print_info "Bumping version..."
if [[ "$BUMP_TYPE" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
  npm version "$BUMP_TYPE" --no-git-tag-version
else
  npm version "$BUMP_TYPE" --no-git-tag-version
fi

NEW_VERSION=$(node -p "require('./package.json').version")
print_info "New version: $NEW_VERSION"

# Generate changelog
print_info "Generating changelog..."
node scripts/generate-changelog.js

# Run tests
print_info "Running tests..."
npm test

# Build project
print_info "Building project..."
npm run build

# Commit version bump and changelog
print_info "Committing changes..."
git add package.json package-lock.json CHANGELOG.md
git commit -m "chore(release): $NEW_VERSION"

# Create git tag
print_info "Creating git tag v$NEW_VERSION..."
git tag -a "v$NEW_VERSION" -m "Release version $NEW_VERSION"

# Push changes
print_info "Pushing to origin..."
git push origin main
git push origin "v$NEW_VERSION"

print_info "Release $NEW_VERSION created successfully!"
print_info "GitHub Actions will now build and publish the release."
```

## Version Bumping

### NPM Version Command

```bash
# Bump version automatically
npm version patch  # 1.0.0 -> 1.0.1
npm version minor  # 1.0.0 -> 1.1.0
npm version major  # 1.0.0 -> 2.0.0

# Pre-release versions
npm version prepatch  # 1.0.0 -> 1.0.1-0
npm version preminor  # 1.0.0 -> 1.1.0-0
npm version premajor  # 1.0.0 -> 2.0.0-0
npm version prerelease  # 1.0.0-0 -> 1.0.0-1

# Specific version
npm version 2.3.4

# Skip git tag creation
npm version patch --no-git-tag-version

# Custom commit message
npm version patch -m "Upgrade to %s for reasons"
```

### Custom Version Bumping Script

```javascript
// scripts/bump-version.js
const fs = require('fs');
const path = require('path');

function bumpVersion(type, currentVersion) {
  const parts = currentVersion.split('-')[0].split('.');
  let [major, minor, patch] = parts.map(Number);
  
  switch (type) {
    case 'major':
      major += 1;
      minor = 0;
      patch = 0;
      break;
    case 'minor':
      minor += 1;
      patch = 0;
      break;
    case 'patch':
      patch += 1;
      break;
    default:
      throw new Error(`Unknown bump type: ${type}`);
  }
  
  return `${major}.${minor}.${patch}`;
}

function updatePackageJson(newVersion) {
  const pkgPath = path.join(process.cwd(), 'package.json');
  const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
  pkg.version = newVersion;
  fs.writeFileSync(pkgPath, JSON.stringify(pkg, null, 2) + '\n');
}

function updateVersionFile(newVersion) {
  const versionPath = path.join(process.cwd(), 'src', 'version.ts');
  const content = `// Auto-generated file - do not edit
export const VERSION = '${newVersion}';
export const BUILD_DATE = '${new Date().toISOString()}';
`;
  fs.writeFileSync(versionPath, content);
}

function updateLockFiles(newVersion) {
  // Update package-lock.json
  const lockPath = path.join(process.cwd(), 'package-lock.json');
  if (fs.existsSync(lockPath)) {
    const lock = JSON.parse(fs.readFileSync(lockPath, 'utf8'));
    lock.version = newVersion;
    if (lock.packages && lock.packages['']) {
      lock.packages[''].version = newVersion;
    }
    fs.writeFileSync(lockPath, JSON.stringify(lock, null, 2) + '\n');
  }
}

// Main execution
const bumpType = process.argv[2] || 'patch';
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
const currentVersion = pkg.version;
const newVersion = bumpVersion(bumpType, currentVersion);

console.log(`Bumping version from ${currentVersion} to ${newVersion}`);

updatePackageJson(newVersion);
updateVersionFile(newVersion);
updateLockFiles(newVersion);

console.log('Version bumped successfully!');
console.log(`Run: git add . && git commit -m "chore: bump version to ${newVersion}"`);
```

## Pre-release and Build Metadata

### Pre-release Versions

```javascript
// Pre-release version patterns
const preReleaseVersions = {
  alpha: '1.0.0-alpha.1',     // Early development
  beta: '1.0.0-beta.2',       // Feature complete, testing
  rc: '1.0.0-rc.3',           // Release candidate
  snapshot: '1.0.0-snapshot', // Daily builds
};

// Incrementing pre-release versions
function incrementPreRelease(version) {
  const match = version.match(/^(\d+\.\d+\.\d+)-([a-z]+)\.(\d+)$/);
  if (!match) throw new Error('Invalid pre-release version');
  
  const [, base, identifier, number] = match;
  const newNumber = parseInt(number, 10) + 1;
  
  return `${base}-${identifier}.${newNumber}`;
}

console.log(incrementPreRelease('1.0.0-alpha.1')); // "1.0.0-alpha.2"
console.log(incrementPreRelease('1.0.0-beta.5')); // "1.0.0-beta.6"

// Promoting pre-release to release
function promoteToRelease(version) {
  return version.split('-')[0];
}

console.log(promoteToRelease('1.0.0-rc.3')); // "1.0.0"
```

### Build Metadata

```javascript
// Build metadata (doesn't affect version precedence)
const buildVersions = {
  withBuildNumber: '1.0.0+build.123',
  withCommitHash: '1.0.0+sha.5114f85',
  withTimestamp: '1.0.0+20240615',
  combined: '1.0.0-beta.1+build.456.sha.abc123',
};

// Generating build metadata
function generateBuildMetadata() {
  const buildNumber = process.env.BUILD_NUMBER || 'local';
  const gitHash = require('child_process')
    .execSync('git rev-parse --short HEAD')
    .toString()
    .trim();
  const timestamp = new Date().toISOString().split('T')[0].replace(/-/g, '');
  
  return `build.${buildNumber}.${gitHash}.${timestamp}`;
}

// Adding build metadata to version
function addBuildMetadata(version) {
  const metadata = generateBuildMetadata();
  return `${version}+${metadata}`;
}

console.log(addBuildMetadata('1.2.3')); 
// "1.2.3+build.456.abc1234.20240615"
```

## Release Strategies

### Release Flow Diagrams

```
Trunk-Based Development:
────────────────────────────────────────────
main:     ●──●──●──●──●──●──●──●──●──●
          ↓     ↓        ↓        ↓
        v1.0  v1.1     v1.2     v1.3


Git Flow:
────────────────────────────────────────────
main:     ●────────●────────●────────●
          │      v1.0     v1.1     v1.2
develop:  ●──●──●──●──●──●──●──●──●──●
             │     │     │
feature:     ●──●  │     │
                   │     │
release:           ●──●  │
                         │
hotfix:                  ●──●


GitHub Flow:
────────────────────────────────────────────
main:     ●──────●──────●──────●──────●
             │      │      │      │
feature-1:   ●──●   │      │      │
                    │      │      │
feature-2:          ●──●   │      │
                           │      │
bugfix:                    ●──●   │
                                  │
feature-3:                        ●──●──●
```

### Continuous Deployment

```yaml
# .github/workflows/cd.yml
name: Continuous Deployment

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Determine environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "ENVIRONMENT=production" >> $GITHUB_OUTPUT
            echo "VERSION_SUFFIX=" >> $GITHUB_OUTPUT
          else
            echo "ENVIRONMENT=staging" >> $GITHUB_OUTPUT
            echo "VERSION_SUFFIX=-dev.${{ github.run_number }}" >> $GITHUB_OUTPUT
          fi
      
      - name: Generate version
        id: version
        run: |
          BASE_VERSION=$(node -p "require('./package.json').version")
          VERSION="${BASE_VERSION}${{ steps.env.outputs.VERSION_SUFFIX }}"
          echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT
          echo "Version: ${VERSION}"
      
      - name: Build and deploy
        env:
          VERSION: ${{ steps.version.outputs.VERSION }}
          ENVIRONMENT: ${{ steps.env.outputs.ENVIRONMENT }}
        run: |
          echo "Deploying version ${VERSION} to ${ENVIRONMENT}"
          npm ci
          npm run build
          # Deploy commands here
```

## Common Mistakes

### 1. Inconsistent Versioning

```javascript
// Bad: Random version numbers
"1.0.0" -> "1.5.0" -> "1.3.2" -> "2.0.1"

// Good: Follow SemVer consistently
"1.0.0" -> "1.0.1" -> "1.1.0" -> "2.0.0"
```

### 2. Missing Changelog Updates

```bash
# Bad: Tag without changelog
git tag v1.2.0
git push --tags

# Good: Update changelog first
npm run changelog
git add CHANGELOG.md
git commit -m "docs: update changelog for v1.2.0"
git tag -a v1.2.0 -m "Release v1.2.0"
git push --tags
```

### 3. Breaking Changes in Minor Versions

```javascript
// Bad: Breaking change in minor version
// v1.2.0 -> v1.3.0 with breaking API changes

// Good: Breaking change in major version
// v1.2.0 -> v2.0.0 with breaking API changes
```

### 4. Not Testing Before Release

```bash
# Bad: Tag and release without testing
git tag v1.2.0
npm publish

# Good: Test, then release
npm test
npm run build
npm run e2e
git tag v1.2.0
npm publish
```

### 5. Forgetting to Sync Version Numbers

```javascript
// Bad: package.json version doesn't match git tag
// package.json: "version": "1.2.0"
// Git tag: v1.3.0

// Good: Keep them in sync
npm version minor  // Updates package.json AND creates git tag
```

## Best Practices

### 1. Automate Version Bumping

```json
{
  "scripts": {
    "release:patch": "npm run test && npm version patch && git push --follow-tags",
    "release:minor": "npm run test && npm version minor && git push --follow-tags",
    "release:major": "npm run test && npm version major && git push --follow-tags"
  }
}
```

### 2. Use Conventional Commits

```bash
# Commit message format: <type>(<scope>): <subject>

feat(auth): add OAuth2 login support
fix(api): resolve timeout issue in user endpoints
docs(readme): update installation instructions
chore(deps): upgrade React to v18
refactor(components): simplify button component
test(auth): add unit tests for login flow
perf(dashboard): optimize data fetching

# Breaking changes
feat(api)!: remove deprecated v1 endpoints

BREAKING CHANGE: v1 API endpoints have been removed.
Migrate to v2 endpoints documented at /docs/api-v2
```

### 3. Maintain Detailed Changelogs

```markdown
## [1.2.0] - 2024-06-15

### Added
- **Authentication**: OAuth2 login support (#123)
  - Google Sign-In
  - GitHub Sign-In
  - Facebook Sign-In
- **Dashboard**: Real-time notifications (#145)
- **Export**: CSV export for all data tables (#156)

### Changed
- **Performance**: Improved search speed by 50% (#134)
- **UI**: Updated to Material Design 3 (#142)
- **API**: Enhanced error messages with more context (#151)

### Fixed
- **Dashboard**: Memory leak in chart component (#128)
- **Mobile**: Fixed responsive layout issues on iOS (#136)
- **Timezone**: Resolved date picker timezone bugs (#147)

### Security
- **XSS**: Patched vulnerability in user input fields (CVE-2024-12345)
- **Dependencies**: Updated all packages with known vulnerabilities

### Migration Guide
If upgrading from v1.1.x:
1. Update API base URL from `/api/v1` to `/api/v2`
2. Replace `user.getName()` with `user.fullName` property
3. Run database migration: `npm run migrate`
```

### 4. Use Protected Branches and Release Branches

```bash
# GitHub branch protection settings
# Settings -> Branches -> Branch protection rules

# Protect main branch:
# ☑ Require pull request reviews before merging
# ☑ Require status checks to pass before merging
# ☑ Require conversation resolution before merging
# ☑ Include administrators
# ☑ Restrict who can push to matching branches (only CI/CD)

# Release workflow
git checkout main
git pull
git checkout -b release/v1.2.0
# Make release preparations
npm version minor
npm run changelog
git push origin release/v1.2.0
# Create PR to main
# After merge, CI/CD creates tag and release
```

### 5. Implement Version Compatibility Checks

```typescript
// version-check.ts
export class VersionChecker {
  static isCompatible(requiredVersion: string, currentVersion: string): boolean {
    const required = this.parseVersion(requiredVersion);
    const current = this.parseVersion(currentVersion);
    
    // Major versions must match
    if (required.major !== current.major) {
      return false;
    }
    
    // Current minor must be >= required minor
    if (current.minor < required.minor) {
      return false;
    }
    
    // If minor versions match, current patch must be >= required patch
    if (current.minor === required.minor && current.patch < required.patch) {
      return false;
    }
    
    return true;
  }
  
  private static parseVersion(version: string) {
    const match = version.match(/^(\d+)\.(\d+)\.(\d+)/);
    if (!match) throw new Error(`Invalid version: ${version}`);
    
    return {
      major: parseInt(match[1], 10),
      minor: parseInt(match[2], 10),
      patch: parseInt(match[3], 10),
    };
  }
}

// Usage in app initialization
const APP_VERSION = '2.1.5';
const REQUIRED_API_VERSION = '2.0.0';

if (!VersionChecker.isCompatible(REQUIRED_API_VERSION, APP_VERSION)) {
  throw new Error(`App version ${APP_VERSION} is not compatible with API version ${REQUIRED_API_VERSION}`);
}
```

## When to Use/Not to Use

### When to Use Semantic Versioning

1. **Libraries and Packages**: Public APIs consumed by other developers
2. **Versioned APIs**: RESTful APIs with breaking changes
3. **Distributed Applications**: Multiple teams coordinating releases
4. **Open Source Projects**: Clear communication with community
5. **Long-term Projects**: Maintenance over months/years
6. **Dependencies**: Packages with dependency management
7. **Backward Compatibility Matters**: Need to communicate breaking changes

### When to Use Calendar Versioning

1. **Time-Based Releases**: Regular release schedules (Ubuntu 22.04)
2. **Marketing-Driven**: Version numbers tied to release dates
3. **Internal Tools**: Less focus on API compatibility
4. **Rapid Releases**: Frequent updates where SemVer is cumbersome
5. **User-Facing Software**: Version communicates recency

### When to Use Other Schemes

1. **Sequential Numbering**: Very simple projects, internal tools
2. **Git Commit-Based**: Pre-release builds, CI/CD artifacts
3. **Marketing Versions**: User-facing names ("Windows 11", "macOS Ventura")

## Interview Questions

### 1. Explain Semantic Versioning and when you would bump each number.

**Answer**: Semantic Versioning uses MAJOR.MINOR.PATCH format. Bump MAJOR for breaking changes that break backward compatibility (API changes, removed features). Bump MINOR for new features that are backward compatible. Bump PATCH for bug fixes that don't add features or break compatibility. Pre-release versions use suffixes like `-alpha.1`, `-beta.2`, `-rc.1`.

### 2. How would you implement automated version bumping in a CI/CD pipeline?

**Answer**: Use tools like semantic-release or standard-version that analyze commit messages following Conventional Commits format. Configure CI/CD to run these tools on main branch merges. They automatically determine version bump type, update package.json, generate changelog, create git tag, and publish release. Can be integrated with GitHub Actions, GitLab CI, or other CI/CD tools.

### 3. What's the difference between a git tag and a release branch?

**Answer**: A git tag is a pointer to a specific commit, marking a release point (lightweight or annotated). A release branch is a long-lived branch for preparing and maintaining a release, allowing bug fixes without incorporating new features from develop. Tags are immutable snapshots; release branches allow ongoing work. Git Flow uses both: release branches for preparation, tags for marking final releases.

### 4. How do you handle hotfixes in a production release?

**Answer**: Create a hotfix branch from the production tag/commit, fix the bug, bump the PATCH version, test thoroughly, merge to main and tag as a patch release (e.g., v1.2.3 -> v1.2.4), then merge back to develop to ensure the fix is included in future releases. Deploy the hotfix to production immediately. Document in changelog under "Fixed" or "Security" section.

### 5. What should be included in a good changelog?

**Answer**: Follow Keep a Changelog format with categories: Added, Changed, Deprecated, Removed, Fixed, Security. Each entry should be human-readable, link to issues/PRs, group related changes, highlight breaking changes prominently, include migration guides for major versions, and date each release. Maintain an Unreleased section for upcoming changes. Use semantic grouping and clear, non-technical language when possible.

### 6. How do you manage dependencies with different major versions?

**Answer**: Use package managers' version resolution (npm's semver ranges, yarn resolutions). For breaking changes, maintain multiple major versions in separate branches if needed. Use peer dependencies to force version alignment. Document compatibility requirements. Consider using tools like Renovate or Dependabot for automated dependency updates. Implement version compatibility checks in code.

### 7. What's the difference between pre-release versions and build metadata?

**Answer**: Pre-release versions (-alpha.1, -beta.2) affect version precedence and indicate stability level (1.0.0-alpha < 1.0.0). Build metadata (+build.123) does NOT affect version precedence, only provides additional information about the build. Pre-release is for users; build metadata is for developers and CI/CD systems. Both can be combined: 1.0.0-beta.1+build.456.

### 8. How would you implement feature flags tied to versions?

**Answer**: Store feature flag configuration versioned alongside code. Use semantic versioning to communicate when features are available. Implement runtime checks: `if (version >= '2.1.0') { enableFeature(); }`. Consider using feature flag services (LaunchDarkly) for dynamic control. Use environment-based flags for gradual rollouts. Document feature availability in changelogs with version requirements.

## Key Takeaways

1. **Semantic Versioning is the industry standard**: MAJOR.MINOR.PATCH communicates intent clearly
2. **Automate version management**: Use tools to reduce human error and ensure consistency
3. **Changelogs are essential**: Keep a human-readable record of changes between versions
4. **Git tags mark releases**: Use annotated tags for releases with metadata
5. **Follow Conventional Commits**: Enables automated tooling and clear history
6. **Test before releasing**: Never tag/publish without testing
7. **Pre-release versions for testing**: Use alpha, beta, rc for non-stable releases
8. **Keep versions synchronized**: package.json, git tags, and documentation should match
9. **Breaking changes need major bumps**: Respect backward compatibility
10. **Document migration paths**: Help users upgrade smoothly between major versions

## Resources

### Documentation
- [Semantic Versioning Specification](https://semver.org/)
- [Keep a Changelog](https://keepachangelog.com/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Calendar Versioning](https://calver.org/)

### Tools
- [semantic-release](https://github.com/semantic-release/semantic-release) - Automated versioning
- [standard-version](https://github.com/conventional-changelog/standard-version) - Version management
- [release-it](https://github.com/release-it/release-it) - Release automation
- [np](https://github.com/sindresorhus/np) - Better npm publish
- [commitizen](https://github.com/commitizen/cz-cli) - Conventional commits helper

### Git Flow
- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [GitLab Flow](https://docs.gitlab.com/ee/topics/gitlab_flow.html)

### Articles
- [NPM Semantic Versioning](https://docs.npmjs.com/about-semantic-versioning)
- [The Art of Versioning](https://medium.com/@jameshamann/the-art-of-versioning-9a2a5e7a7f5f)
- [Version Control Best Practices](https://www.git-tower.com/learn/git/ebook/en/command-line/appendix/best-practices)
