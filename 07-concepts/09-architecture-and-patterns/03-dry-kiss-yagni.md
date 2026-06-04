# DRY, KISS, and YAGNI

## Overview

DRY (Don't Repeat Yourself), KISS (Keep It Simple, Stupid), and YAGNI (You Ain't Gonna Need It) are three of the most frequently cited software principles — and three of the most frequently misapplied. DRY is about authoritative knowledge, not eliminating all code that looks similar. KISS is about minimizing accidental complexity, not refusing to handle real complexity. YAGNI is about avoiding speculation, not being shortsighted. This guide explores each principle with frontend-specific examples and explains when each one is being violated — including violations in the over-application direction.

## DRY — Don't Repeat Yourself

**"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."** — The Pragmatic Programmer

DRY is about knowledge duplication, not code duplication. Two similar-looking functions that implement different business rules are NOT DRY violations.

### The Right Kind of DRY

```typescript
// ❌ Knowledge duplicated — business rule appears in three places
function calculateCartTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0) * 0.9; // 10% discount
}

function displayOrderSummary(items: CartItem[]): string {
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0) * 0.9; // 10% discount
  return `Total: $${total.toFixed(2)}`;
}

function validateOrderAmount(items: CartItem[]): boolean {
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0) * 0.9; // 10% discount
  return total > 0;
}
// When the discount rate changes from 10% to 15%, you must find and change 3 places
// (and likely miss one in a 50-file codebase)

// ✅ Single source of truth for the business rule
const STANDARD_DISCOUNT = 0.9; // 10% off

function getCartSubtotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

function applyStandardDiscount(amount: number): number {
  return amount * STANDARD_DISCOUNT;
}

function calculateCartTotal(items: CartItem[]): number {
  return applyStandardDiscount(getCartSubtotal(items));
}
// One change in applyStandardDiscount affects all uses
```

### The Wrong Kind of DRY — Accidental Similarity

```typescript
// ❌ Over-DRY: unifying code that LOOKS similar but implements different rules
// User registration form fields
function renderField(label: string, field: string) {
  if (field === 'email') return <input type="email" required />;
  if (field === 'username') return <input maxLength={20} pattern="[a-z0-9]+" />;
  if (field === 'password') return <input type="password" minLength={8} />;
  if (field === 'bio') return <textarea maxLength={500} />;
  // ...each new field type adds another branch to this function
}

// This "DRY" is actually worse than the duplication it replaced:
// Each field type has different rules — the abstraction is leaky
// Adding a new field type requires modifying the existing function
// Testing requires testing all branches together

// ✅ Embrace duplication when the rules are different
function EmailField() {
  return <input type="email" required aria-label="Email address" />;
}

function UsernameField() {
  return <input maxLength={20} pattern="[a-z0-9]+" aria-label="Username" />;
}

function PasswordField() {
  return <input type="password" minLength={8} aria-label="Password" />;
}
// Each field evolves independently — no shared fragile abstraction
```

### The Wrong Abstraction is Worse Than Duplication

```typescript
// Duplication is a local problem — it hurts only where it appears.
// A wrong abstraction is a global problem — it forces bad choices everywhere.

// ❌ Premature unification
interface InputFieldProps {
  type: 'text' | 'email' | 'password' | 'number' | 'date' | 'select' | 'checkbox';
  options?: SelectOption[];  // only for 'select'
  min?: number;              // only for 'number'
  max?: number;              // only for 'number'
  minDate?: Date;            // only for 'date'
  checked?: boolean;         // only for 'checkbox'
  rows?: number;             // oops, now need textarea
  // This keeps growing — the abstraction is fighting reality
}

// ✅ Separate types that know what they are
<TextInput label="Name" maxLength={100} />
<SelectInput label="Country" options={countries} />
<DateInput label="Birthday" minDate={MIN_DATE} />
<CheckboxInput label="Agree to terms" />
```

### DRY in CSS / Tailwind

```css
/* ❌ CSS DRY violation — styles duplicated in 15 components */
.primary-button {
  background: #3b82f6;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 500;
}

/* ✅ CSS variable / design token as single source */
:root {
  --color-primary: #3b82f6;
  --radius-md: 6px;
  --spacing-button: 8px 16px;
  --font-weight-medium: 500;
}

.primary-button {
  background: var(--color-primary);
  padding: var(--spacing-button);
  border-radius: var(--radius-md);
  font-weight: var(--font-weight-medium);
}
/* Change --color-primary once → updates everywhere */
```

## KISS — Keep It Simple, Stupid

**The simplest solution that correctly solves the problem is almost always the right one.** Complexity should be earned — it should exist only to handle actual complexity in the problem domain.

### Accidental vs Essential Complexity

```
Accidental complexity: complexity you introduced
  → 5 abstraction layers where 1 would do
  → Generic framework for one specific use case
  → Async where synchronous works
  → Config-driven behavior for one configuration
  → Abstract factory to create one concrete type

Essential complexity: complexity the problem actually requires
  → Real-time sync requires websockets (can't simplify away)
  → Financial calculations require decimal precision handling
  → Accessibility requires ARIA attributes
  → i18n requires locale-aware formatting
```

### KISS Violations in React

```tsx
// ❌ Over-engineered counter — unnecessary abstraction layer
class CounterState {
  private value: number = 0;
  private observers: Array<(v: number) => void> = [];

  subscribe(observer: (v: number) => void) {
    this.observers.push(observer);
    return () => { this.observers = this.observers.filter(o => o !== observer); };
  }

  increment() {
    this.value++;
    this.observers.forEach(o => o(this.value));
  }

  getValue() { return this.value; }
}

const counterState = new CounterState();

function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => counterState.subscribe(setCount), []);
  return <button onClick={() => counterState.increment()}>{count}</button>;
}

// ✅ KISS — it's just a counter
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

```tsx
// ❌ Generic "entity grid" that handles every possible entity type
function EntityGrid<T extends { id: string }>({
  entityType,
  renderItem,
  sortStrategies,
  filterStrategies,
  paginationStrategy,
  exportStrategy,
}: EntityGridProps<T>) {
  // 300 lines of generic logic
}

// Used once, for one entity:
<EntityGrid entityType="product" renderItem={renderProduct} ... />

// ✅ Just write ProductGrid
function ProductGrid({ products }: { products: Product[] }) {
  // 50 lines specifically for products
  // Clear, specific, easy to understand
}
// If you later need UserGrid, write UserGrid separately
// Then extract a shared abstraction if the similarity is real
```

### When Complexity Is the Right Answer

```tsx
// ✅ Complex solution for genuinely complex problem

// Real-time collaborative editor with conflict resolution:
// This IS complex — the problem is complex. KISS doesn't mean
// "simplify away real complexity."
function CollaborativeEditor({
  documentId,
  initialContent,
}: {
  documentId: string;
  initialContent: Y.Doc;
}) {
  const ydoc = useRef(initialContent);
  const provider = useRef(new WebrtcProvider(documentId, ydoc.current));

  // OT/CRDT, version vectors, sync protocol — all essential complexity
  // for real-time collaboration
  return <TipTapEditor collaboration={{ document: ydoc.current }} />;
}
```

## YAGNI — You Ain't Gonna Need It

**Don't implement functionality until it's needed.** Speculative generality adds maintenance burden, cognitive overhead, and often solves a problem you won't actually have.

### Common YAGNI Violations

```typescript
// ❌ Plugin system for a simple configuration loader
class ConfigLoader {
  private plugins: ConfigPlugin[] = [];

  registerPlugin(plugin: ConfigPlugin) { this.plugins.push(plugin); }

  load(path: string): Config {
    let config = parseYaml(readFile(path));
    for (const plugin of this.plugins) {
      config = plugin.transform(config);
    }
    return config;
  }
}

// There is ONE config file. It never changes. No plugin has ever been registered.
// YAGNI: just write parseConfig(path).

// ✅ Write the simplest thing that works
function loadConfig(path: string): Config {
  return parseYaml(readFile(path));
}
// If plugins are needed later, add them then — with a real use case to guide the design.
```

```typescript
// ❌ Building a full event system for "future flexibility"
class EventBus {
  private handlers: Map<string, Set<Function>> = new Map();

  on(event: string, handler: Function) { /* ... */ }
  off(event: string, handler: Function) { /* ... */ }
  emit(event: string, ...args: unknown[]) { /* ... */ }
  once(event: string, handler: Function) { /* ... */ }
  onAny(handler: Function) { /* ... */ }
}

// App currently has 2 event types: 'user:login' and 'user:logout'
// Both used by exactly one subscriber each
// A simple callback prop would have solved this entirely

// ✅ Use the simplest mechanism that works
interface AuthCallbacks {
  onLogin: (user: User) => void;
  onLogout: () => void;
}
```

### YAGNI and API Design

```typescript
// ❌ API with unused flexibility
interface FetchUserOptions {
  includeAddress?: boolean;
  includePaymentMethods?: boolean;
  includeOrderHistory?: boolean;
  includeWishlist?: boolean;
  maxOrderHistoryItems?: number;
  currency?: string;
  includeInactive?: boolean;
  // ... 8 more options "for future use"
}

async function fetchUser(id: string, options: FetchUserOptions = {}): Promise<User> {
  // Only includeOrderHistory has ever been used
}

// ✅ Start with what you need
async function fetchUser(id: string): Promise<User> {
  return api.get(`/users/${id}`);
}

// Add options when a concrete need exists:
async function fetchUserWithOrderHistory(id: string): Promise<UserWithOrders> {
  return api.get(`/users/${id}?include=orders`);
}
```

### When YAGNI Doesn't Apply

```
YAGNI is about IMPLEMENTATION, not design.

Design for extension (open/closed) even if you don't implement it:
  ✅ Define an interface with clear extension points
  ✅ Leave the architecture room to grow
  ✅ Don't hard-code limits that are hard to change later

But don't IMPLEMENT extensions until needed:
  ❌ Build the plugin system before any plugins exist
  ❌ Add configuration options for scenarios that haven't occurred
  ❌ Add "phase 2 features" to a phase 1 release
```

## Balancing the Principles Against Each Other

```
When principles conflict:

DRY vs KISS:
  Sometimes removing duplication INCREASES complexity.
  Two clear, simple functions vs one complex, generic function.
  Prefer KISS — live with the duplication.
  Rule: abstract only after the third occurrence, and only if the abstraction
        is clearer than the duplication.

DRY vs YAGNI:
  If you're abstracting speculatively, YAGNI wins.
  Abstract when you can see the existing duplication, not when you imagine future duplication.

YAGNI vs good architecture:
  Don't let YAGNI justify poor structure.
  Separating concerns (SRP) even in simple code is not YAGNI violation.
  YAGNI targets features, not structure.
```

### The Rule of Three

```
Abstract on the THIRD occurrence — not the first or second.

First occurrence: just write it.
Second occurrence: tempting to abstract. Resist — you don't know the pattern yet.
Third occurrence: now you see the real shape. Abstract at this point.

This avoids "premature DRY" — the wrong abstraction born from one example.
```

## Common Mistakes

### 1. Treating Code Similarity as DRY Violation

```typescript
// ❌ "These two look similar, must extract!"
async function fetchUser(id: string) { return api.get(`/users/${id}`); }
async function fetchPost(id: string) { return api.get(`/posts/${id}`); }

// "Let me DRY this up":
async function fetchEntity(type: 'users' | 'posts', id: string) {
  return api.get(`/${type}/${id}`);
}

// Problem: users and posts have DIFFERENT rules:
// - Users require auth on every request
// - Posts are public if published, auth'd if draft
// - User errors go to UserNotFound page, Post errors stay on page
// The abstraction doesn't hold
```

### 2. Applying YAGNI to Security and Data Integrity

```typescript
// ❌ "YAGNI — we won't need input validation"
// Input validation, security, error handling are REQUIREMENTS, not future features
// YAGNI doesn't exempt essential correctness

// ✅ Validate inputs from the start
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});
```

### 3. KISS as an Excuse for Sloppy Code

```typescript
// ❌ "KISS — no types needed"
function processData(data) { // any types, no validation
  return data.items.map(i => i.total).reduce((a, b) => a + b);
}

// KISS is about complexity, not correctness
// Type safety reduces complexity by catching errors at compile time
// ✅
function calculateItemsTotal(items: Array<{ total: number }>): number {
  return items.reduce((sum, item) => sum + item.total, 0);
}
```

## Interview Questions

### 1. What is the difference between DRY and avoiding code that looks similar?

**Answer:** DRY targets knowledge duplication — when the same fact or rule is encoded in multiple places, a change in that fact requires multiple updates, and you will miss some. It's about a single authoritative source for a business rule, algorithm, or invariant. Code that looks similar but implements different rules is NOT a DRY violation — it's two separate things that happen to look alike. The test: if the business rule changes, does it require changing both pieces of code? If yes, DRY applies. If the two pieces would change for different reasons (different business rules), they should stay separate. The "wrong abstraction" from premature DRY is often worse than the duplication.

### 2. What is YAGNI and how do you apply it without being shortsighted?

**Answer:** YAGNI says don't implement features or flexibility you don't currently need, because speculative implementation is usually wrong (the actual need turns out different), adds maintenance overhead, and makes code harder to understand. Apply it by building exactly what the current requirements ask for. The distinction: YAGNI targets implementation of speculative features, not architectural quality. Good separation of concerns, clean interfaces, and clear naming are not YAGNI violations — they make future changes easier without implementing the changes prematurely. YAGNI violations: plugin systems with no plugins, configuration options never used, "phase 2" code in a phase 1 release.

### 3. When does trying to follow DRY actually hurt your codebase?

**Answer:** When you abstract too early, before you understand the real shape of the duplication — the "rule of three" exists for this reason. When the two pieces of code that look similar are actually implementing different business rules that will diverge over time — a shared abstraction forces them to evolve together when they shouldn't. When the abstraction is more complex than the duplication — one generic function with 8 parameters for two use cases that each need 2 parameters. When the abstraction is at the wrong layer — DRY'ing presentation code across different domains creates coupling. The maxim "duplication is far cheaper than the wrong abstraction" captures the cost.

### 4. How do you explain KISS to someone who argues "simple is just not writing tests"?

**Answer:** KISS targets accidental complexity — complexity you introduced that isn't required by the problem. Tests, input validation, type safety, error handling, and security controls reduce accidental complexity by making code behavior predictable and catching mistakes early. They don't add complexity; they manage it. The complexity of a type error at runtime (stack trace, prod incident, debugging session) is far greater than the "complexity" of a TypeScript type definition. KISS means: don't add an event bus where a callback would do; don't add a plugin system for one use case; don't add a state machine for a simple boolean. It doesn't mean: skip correctness, safety, or testability.
