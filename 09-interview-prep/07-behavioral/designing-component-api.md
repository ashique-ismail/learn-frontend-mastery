# How Do You Design a Component API?

## What Interviewers Want to See

Design thinking: you balance flexibility, simplicity, and consistency. They want to know you think about the component's consumers, not just its internal implementation.

---

## Core Principles

### 1. Start With the Consumer's Point of View

Write the usage code first (API-first design), then implement:

```tsx
// Before writing the component, write how you want to use it:
<DataTable
  data={users}
  columns={columnDefs}
  sortable
  onRowClick={handleRowClick}
  loading={isFetching}
  emptyState={<EmptyUsers />}
/>
```

If the API looks awkward to write, it'll be awkward to use. Fix it before implementation.

### 2. Prop Design: The Three Questions

For every prop, ask:
- **Is it required or optional?** Avoid requiring props that have sensible defaults.
- **What type should it be?** Prefer specific types over `any` or `object`.
- **Does it belong here?** Could the consumer compute it themselves? Could the component handle it internally?

```tsx
// ❌ Too many required props — forces callers to compute things
<Button
  backgroundColor="blue"
  textColor="white"
  hoverBackgroundColor="darkblue"
  isDisabled={false}
  spinnerVisible={false}
/>

// ✅ Abstracted to semantic variants
<Button variant="primary" size="md" loading={false}>Submit</Button>
```

### 3. Composition Over Configuration

Prefer composing components over a single mega-component with many boolean props:

```tsx
// ❌ Boolean props that multiply combinatorially
<Modal
  hasHeader
  hasFooter
  hasCloseButton
  hasBackdrop
  isScrollable
  headerContent={<h2>Title</h2>}
  footerContent={<Button>OK</Button>}
/>

// ✅ Composable slots
<Modal>
  <ModalHeader>
    <h2>Title</h2>
    <ModalCloseButton />
  </ModalHeader>
  <ModalBody>Content here</ModalBody>
  <ModalFooter>
    <Button>OK</Button>
  </ModalFooter>
</Modal>
```

Each sub-component is independently testable, and consumers only include what they need.

### 4. Escape Hatches

Always provide a way out of the component's opinions:

```tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  className?: string;   // ← escape hatch for custom styles
  style?: CSSProperties; // ← escape hatch for inline styles
  as?: ElementType;    // ← escape hatch for rendering as a different element
}
```

### 5. Render Props / Children Flexibility

```tsx
// Flexible: let consumer decide what to render
<Select options={items} renderOption={(item) => <CustomOption item={item} />} />

// Or via children for simpler cases
<Tabs>
  <Tab label="Overview"><OverviewContent /></Tab>
  <Tab label="Details"><DetailsContent /></Tab>
</Tabs>
```

---

## Example: Designing a `Table` Component

**Requirements:**
- Display tabular data with sorting, pagination, optional selection
- Must work for simple cases without configuration
- Must be customizable for complex cases

**API Design Process:**

```tsx
// Step 1: Simple case (must work with minimal config)
<Table data={rows} columns={columns} />

// Step 2: Add sorting
<Table data={rows} columns={columns} sortable />

// Step 3: Column-level control (some columns sortable, some have custom render)
const columns: ColumnDef[] = [
  { key: 'name', header: 'Name', sortable: true },
  { key: 'email', header: 'Email', sortable: true },
  { key: 'status', header: 'Status', cell: (row) => <StatusBadge status={row.status} /> },
];

// Step 4: Pagination
<Table
  data={rows}
  columns={columns}
  pagination={{ pageSize: 20, total: 500, onPageChange: setPage }}
/>

// Step 5: Row selection
<Table
  data={rows}
  columns={columns}
  selectable
  onSelectionChange={setSelectedIds}
/>
```

Each step adds a new capability without breaking previous usage.

---

## Common Mistakes

```tsx
// ❌ Mixing concerns — component doing too much
<UserCard
  userId="123"        // fetches its own data (side effect)
  onSave={save}       // handles its own mutations
  validationRules={rules} // owns its own validation
/>

// ✅ Presentational + separation of concerns
const { user, save } = useUser('123');
<UserCard
  user={user}         // receives data
  onSave={save}       // receives action
  errors={errors}     // receives validation state
/>
```

---

## Example Answer

"When I designed our notification component library, I started by writing every usage scenario first — inline, toast, banner, dialog. I noticed they all needed a title, optional description, optional action button, and a severity level. Rather than four separate components with duplicated logic, I designed one `Notification` component with a `variant` prop and composable content slots.

The key decision was whether to accept a plain string `message` or allow `children`. We chose `children` because it allows rich content (formatted text, links) without us needing to anticipate every use case. The `className` escape hatch handles the rare cases where our design doesn't match the context.

I also wrote detailed usage examples in Storybook before the PR review — the examples caught two props that were confusingly named and one case where the API forced too much computation onto the consumer."

---

## Common Follow-Up Questions

**"How do you balance flexibility and simplicity?"**
"I aim for the 90% case to be simple (3-4 props max) and the 10% edge cases to have escape hatches (className, render props, ref forwarding). I don't design for every hypothetical — I design for what consumers are actually asking for."

**"How do you handle breaking changes in a component API?"**
"Deprecation warnings first, then removal in the next major version. Document migration paths. For internal components, I colocate the migration as a codemod if the change affects many call sites."
