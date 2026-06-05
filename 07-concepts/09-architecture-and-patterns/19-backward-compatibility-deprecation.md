# Backward Compatibility and Deprecation Strategy

## The Idea

**In plain English:** Backward compatibility means keeping old features working even after you upgrade or change software, so people who relied on the old version do not suddenly have broken code. Deprecation is a polite warning that says "this old feature still works today, but we plan to remove it soon — please switch to the new way."

**Real-world analogy:** Imagine a school cafeteria that decides to rename "Lunch Combo A" to "Meal Deal 1" on the menu. Instead of ripping out Combo A overnight and leaving students confused, the cafeteria runs both names for a whole semester, posting a sign that says "Combo A is going away — please start saying Meal Deal 1." Students who haven't updated their order habit still get their food, but they are nudged to change. At the end of the semester, Combo A disappears entirely.

- The old menu name (Combo A) = the deprecated API or prop name that still works temporarily
- The notice sign warning students = the deprecation warning printed in the developer console
- The end-of-semester removal = the breaking change shipped in the next major version

---

## Overview

Every API you ship — whether a REST endpoint, a component prop, or a function signature — is a promise to everyone who depends on it. Breaking that promise without warning destroys trust and costs teams hours of emergency fixes. A principled deprecation strategy lets you evolve APIs while honoring existing consumers.

---

## Component API Versioning

Unlike REST APIs, component libraries don't have URL versioning. You manage compatibility through design choices.

### Additive vs Breaking Changes

```tsx
// ADDITIVE — safe, backward compatible
// Before
<Button onClick={handleClick}>Save</Button>

// After: new optional prop, default preserves old behavior
<Button onClick={handleClick} size="medium">Save</Button>
//                            ^^^^^^^^^^^^^ new optional prop with default
// All existing usages still work — no change required

// BREAKING — requires a migration
// Before
<Button onClick={handleClick}>Save</Button>

// After: renamed required prop
<Button onPress={handleClick}>Save</Button>
//      ^^^^^^^ renamed from onClick
// Every consumer must update — 500 teams' code breaks silently
```

### The Deprecation Lifecycle

```
1. Deprecated   → old API still works, console warning, migration guide
2. Legacy       → old API still works, no warning (consumers had time to migrate)
3. Removed      → old API gone (semver major version bump)

Minimum time between Deprecated → Removed: one major version
Recommended time: 6–12 months
```

### Implementing Deprecation Warnings

```tsx
// React component: warn on deprecated prop usage
function Button({ onClick, onPress, children, ...props }) {
  if (onClick && !onPress) {
    if (process.env.NODE_ENV !== 'production') {
      console.warn(
        '[Button] The `onClick` prop is deprecated. ' +
        'Use `onPress` instead. ' +
        'See migration guide: https://design.acme.com/button#migration'
      );
    }
  }

  // Support both during deprecation period
  const handler = onPress ?? onClick;

  return <button onClick={handler} {...props}>{children}</button>;
}
```

```typescript
// TypeScript: mark deprecated in type system
interface ButtonProps {
  /** @deprecated Use `onPress` instead. Will be removed in v4. */
  onClick?: (event: MouseEvent) => void;
  onPress?: (event: MouseEvent) => void;
}
// IDEs show a strikethrough on deprecated props — visual warning before runtime
```

---

## Codemods: Automating Migrations

A codemod is a script that automatically updates consumer code when an API changes. Without codemods, consumers must manually update every usage — which is why breaking changes are so costly.

### Writing a Codemod with jscodeshift

```bash
npm install -g jscodeshift
```

```javascript
// codemods/rename-onclick-to-onpress.js
// Transforms: <Button onClick={...}> → <Button onPress={...}>

module.exports = function(fileInfo, api) {
  const j = api.jscodeshift;
  const root = j(fileInfo.source);

  // Find all JSX attributes named 'onClick' on Button components
  root
    .find(j.JSXOpeningElement, { name: { name: 'Button' } })
    .find(j.JSXAttribute, { name: { name: 'onClick' } })
    .forEach(path => {
      // Rename onClick → onPress
      path.node.name.name = 'onPress';
    });

  return root.toSource();
};
```

```bash
# Run across the entire codebase
jscodeshift -t codemods/rename-onclick-to-onpress.js src/ --extensions=tsx,ts,jsx,js

# Dry run first
jscodeshift -t codemods/rename-onclick-to-onpress.js src/ --dry --print
```

**What a good codemod ships with:**
1. The transform script
2. Tests (input → expected output)
3. A migration guide (for cases the codemod can't handle automatically)
4. A script to verify no old API usage remains: `grep -r "Button.*onClick" src/`

---

## localStorage / IndexedDB Schema Migration

Data persisted on the client outlives your code. A user who last opened the app 2 years ago may have data in a schema from v1 that your current v5 code can't read.

### Versioned Migration Pattern

```typescript
const SCHEMA_VERSION = 5;

interface StoredData_v1 { theme: string; fontSize: number }
interface StoredData_v3 { theme: { mode: string; accent: string }; fontSize: number }
interface StoredData_v5 { theme: { mode: string; accent: string }; fontSize: number; lang: string }

function loadSettings(): StoredData_v5 {
  const raw = localStorage.getItem('settings');
  if (!raw) return defaultSettings();

  let data = JSON.parse(raw) as { __v?: number; [key: string]: unknown };
  const version = data.__v ?? 0;

  // Chain migrations: each migrates from its version to the next
  if (version < 2) data = migrateV1toV2(data as StoredData_v1);
  if (version < 3) data = migrateV2toV3(data);
  if (version < 5) data = migrateV4toV5(data as StoredData_v3);

  // Always re-save with current version after migration
  const migrated = { ...data, __v: SCHEMA_VERSION } as StoredData_v5;
  localStorage.setItem('settings', JSON.stringify(migrated));
  return migrated;
}

function migrateV1toV2(old: StoredData_v1) {
  return {
    __v: 2,
    theme: { mode: old.theme, accent: 'blue' }, // string → object
    fontSize: old.fontSize,
  };
}

function migrateV4toV5(old: StoredData_v3) {
  return {
    ...old,
    __v: 5,
    lang: navigator.language ?? 'en', // new field with sensible default
  };
}
```

### IndexedDB: Schema Versioning

```typescript
// IDB has built-in version migration via onupgradeneeded
const request = indexedDB.open('app-db', 5); // increment version to trigger migration

request.onupgradeneeded = (event) => {
  const db = (event.target as IDBOpenDBRequest).result;
  const oldVersion = event.oldVersion;

  // Run only the migrations needed
  if (oldVersion < 2) {
    // v1 → v2: add 'tags' field to documents store
    const store = db.createObjectStore('documents', { keyPath: 'id' });
    store.createIndex('tags', 'tags', { multiEntry: true });
  }

  if (oldVersion < 3) {
    // v2 → v3: add 'archived' field
    // Can't alter existing store structure — must recreate or add via cursor
    const transaction = (event.target as IDBOpenDBRequest).transaction!;
    const store = transaction.objectStore('documents');
    store.openCursor().onsuccess = (e) => {
      const cursor = (e.target as IDBRequest<IDBCursorWithValue>).result;
      if (cursor) {
        const doc = cursor.value;
        cursor.update({ ...doc, archived: false }); // add default
        cursor.continue();
      }
    };
  }

  if (oldVersion < 5) {
    // v4 → v5: new object store
    db.createObjectStore('attachments', { keyPath: 'id', autoIncrement: true });
  }
};
```

---

## REST API Versioning and Breaking Change Policy

### Versioning Strategies

```
URL versioning:     /api/v1/users → /api/v2/users
Header versioning:  Accept: application/vnd.api+json;version=2
Query param:        /api/users?version=2

Recommendation: URL versioning for public APIs (visible, cacheable, debuggable)
```

### What Constitutes a Breaking Change

```
BREAKING (requires version bump):
  - Removing a field
  - Renaming a field
  - Changing a field type (string → number)
  - Changing error response shape
  - Removing an endpoint
  - Changing authentication requirements

NON-BREAKING (backward compatible):
  - Adding an optional field
  - Adding a new endpoint
  - Adding an optional query parameter
  - Expanding an enum (adding a new value)
```

### Frontend Defense: Unknown Field Tolerance

```typescript
// BAD: assumes exact API shape — breaks when new fields are added
function processUser(data: { name: string; email: string }) {
  return { name: data.name, email: data.email }; // works
}

// GOOD: accept and pass through unknown fields
function processUser(data: User & Record<string, unknown>) {
  const { name, email, ...rest } = data;
  return { name, email }; // consume what you need, ignore the rest
}

// Even better: use Zod with .passthrough()
const UserSchema = z.object({
  name: z.string(),
  email: z.string(),
}).passthrough(); // unknown fields allowed, preserved

const user = UserSchema.parse(apiResponse); // doesn't throw on new fields
```

---

## Feature Flags as a Deprecation Tool

Feature flags let you run old and new implementations simultaneously, gradually shifting traffic:

```typescript
// Old API: Button with onClick
// New API: Button with onPress
// Feature flag gates which version ships to which users

function Button({ onClick, onPress, ...props }) {
  const useNewApi = useFeatureFlag('button-new-api');

  if (useNewApi) {
    // New implementation — onPress
    return <NewButton onPress={onPress ?? onClick} {...props} />;
  }

  // Old implementation — onClick
  return <OldButton onClick={onClick ?? onPress} {...props} />;
}

// Rollout plan:
// Week 1: 5% of users → new API
// Week 2: 25% → measure metrics
// Week 3: 100% → if metrics good
// Week 4: remove flag, remove old code
```

---

## Interview Questions

**Q: "We need to rename a prop on a component used by 50 teams. How do you do it without breaking everyone?"**
A: Four-step process: (1) Add the new prop while keeping the old one — both work, old one logs a deprecation warning in development; (2) Write a codemod that automatically transforms `oldProp` to `newProp` across all usages; (3) Announce the deprecation with a migration guide, give teams 1–2 sprints to run the codemod; (4) In the next major version, remove the old prop. The key is the codemod — without it, asking 50 teams to manually update their code creates 50 breaking changes across 50 repositories.

**Q: "A user hasn't opened the app in 18 months. They upgrade now. Their localStorage data is v1, your code expects v5. What happens?"**
A: Without a migration strategy: `TypeError` as v1 fields don't match the v5 schema — the app crashes or silently corrupts state. With versioned migrations: on load, you detect the `__v` field (or absence of it), run migration functions for each version step in sequence (v1→v2→v3→v4→v5), save the migrated data, and proceed normally. The user sees their preferences preserved as much as the schema allows, with sensible defaults for fields that didn't exist in their version.

**Q: "What is a codemod and when would you write one?"**
A: A codemod is a script that automatically transforms source code — typically using jscodeshift, which parses code into an AST and lets you find and replace patterns programmatically. I'd write one when making a mechanical, large-scale change: renaming a prop, updating an import path after a package rename, migrating from one hook to another. If a human can do the transformation with find-and-replace (with enough context), a codemod can do it faster and more reliably across 1,000 files.
