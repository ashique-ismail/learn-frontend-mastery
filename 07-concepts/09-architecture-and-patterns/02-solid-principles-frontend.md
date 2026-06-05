# SOLID Principles Applied to Frontend

## The Idea

**In plain English:** SOLID is a set of five rules that help you write code that stays easy to change and fix over time — like building with well-designed Lego pieces instead of hot glue. Each rule stops a different way that code can become a tangled mess.

**Real-world analogy:** Think of a restaurant kitchen. Each station (grill, salad, pastry) has one job, uses shared equipment through standard hooks, and can swap out a chef without disrupting the others.

- The separate kitchen stations = the Single Responsibility Principle (each piece of code does one job)
- The standard hooks and rails that let you hang any pan = the Open/Closed Principle (you can add new tools without rebuilding the kitchen)
- Any trained chef being able to fill any station = the Liskov Substitution Principle (replacements must work the same way as the original)

---

## Overview

SOLID is a set of five design principles coined by Robert Martin that guide object-oriented design toward maintainable, extensible code. While originally framed for class-based OOP, all five principles apply directly to frontend development — React components, Angular services, TypeScript modules, and pure functions all benefit from SOLID thinking. This guide translates each principle into concrete frontend patterns.

## S — Single Responsibility Principle

**A module should have one reason to change.**

A component or function should do one thing. Multiple reasons to change is a sign that concerns are mixed.

### Violation in React

```tsx
// ❌ UserProfile does too much: data fetching, formatting, rendering, analytics
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((data) => {
        setUser(data);
        // Analytics logic mixed in
        window.gtag('event', 'profile_view', { user_id: userId });
      })
      .catch(() => setError('Failed to load'))
      .finally(() => setLoading(false));
  }, [userId]);

  const formattedJoinDate = user
    ? new Intl.DateTimeFormat('en-US').format(new Date(user.joinedAt))
    : null;

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <div>
      <h1>{user!.name}</h1>
      <p>Joined: {formattedJoinDate}</p>
      <p>Email: {user!.email}</p>
    </div>
  );
}
```

### Applying SRP

```tsx
// ✅ Separated concerns

// 1. Data fetching — useUser hook
function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then((r) => r.json()),
  });
}

// 2. Analytics — custom hook
function useProfileViewTracking(userId: string) {
  useEffect(() => {
    window.gtag('event', 'profile_view', { user_id: userId });
  }, [userId]);
}

// 3. Date formatting — pure utility
function formatJoinDate(isoDate: string): string {
  return new Intl.DateTimeFormat('en-US').format(new Date(isoDate));
}

// 4. Display — pure presentational component
function UserProfileCard({ user }: { user: User }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Joined: {formatJoinDate(user.joinedAt)}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}

// 5. Composition — orchestrates the above
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, isError } = useUser(userId);
  useProfileViewTracking(userId);

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage message="Failed to load" />;
  return <UserProfileCard user={user!} />;
}
```

## O — Open/Closed Principle

**Open for extension, closed for modification.**

Add new behavior without modifying existing code. In frontend, this means designing components and functions that can be extended through composition, props, or configuration — not if/else chains.

### Violation

```tsx
// ❌ Adding a new variant requires modifying the Button component
function Button({ variant, children }: { variant: 'primary' | 'secondary' | 'danger'; children: ReactNode }) {
  if (variant === 'primary') {
    return <button className="bg-blue-500 text-white">{children}</button>;
  } else if (variant === 'secondary') {
    return <button className="bg-gray-200 text-gray-800">{children}</button>;
  } else if (variant === 'danger') {
    return <button className="bg-red-500 text-white">{children}</button>;
  }
  // Adding 'ghost' variant requires editing this file
}
```

### Applying OCP

```tsx
// ✅ Closed for modification — extend by passing className or styles

// Option A: className extension
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
}

const variantClasses: Record<NonNullable<ButtonProps['variant']>, string> = {
  primary: 'bg-blue-500 text-white hover:bg-blue-600',
  secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
  danger: 'bg-red-500 text-white hover:bg-red-600',
  ghost: 'bg-transparent border border-gray-300 hover:bg-gray-50',
};

function Button({ variant = 'primary', className = '', ...props }: ButtonProps) {
  return (
    <button
      className={`base-button-styles ${variantClasses[variant]} ${className}`}
      {...props}
    />
  );
}

// New variant via className prop — no modification to Button needed
<Button className="bg-gradient-to-r from-purple-500 to-pink-500 text-white">
  Special
</Button>

// Option B: Compound components / slots — even more extensible
function Card({ children }: { children: ReactNode }) {
  return <div className="card-base">{children}</div>;
}

Card.Header = ({ children }: { children: ReactNode }) => (
  <div className="card-header">{children}</div>
);

Card.Body = ({ children }: { children: ReactNode }) => (
  <div className="card-body">{children}</div>
);

// Consumers extend Card with new layouts without touching the base
```

## L — Liskov Substitution Principle

**Subtypes must be substitutable for their base types without altering correctness.**

In TypeScript/React: if a component accepts `ComponentA`, and `ComponentB` extends it, `ComponentB` should be usable everywhere `ComponentA` is used without breaking anything.

### Violation

```tsx
// ❌ EnhancedInput removes the onChange prop — breaks substitution
interface InputProps {
  value: string;
  onChange: (value: string) => void;
}

// Subtype REMOVES behavior instead of extending it
interface EnhancedInputProps extends InputProps {
  onChange?: never; // Breaking LSP — callers that depend on onChange break!
}
```

### Applying LSP

```tsx
// ✅ Subtypes extend, never restrict

interface BaseInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

// Plain input — satisfies BaseInputProps
function TextInput({ label, error, ...inputProps }: BaseInputProps) {
  return (
    <label>
      {label}
      <input {...inputProps} aria-invalid={!!error} />
      {error && <span role="alert">{error}</span>}
    </label>
  );
}

// AutocompleteInput — EXTENDS TextInput; still satisfies BaseInputProps
interface AutocompleteInputProps extends BaseInputProps {
  suggestions: string[];  // NEW capability — doesn't remove anything
}

function AutocompleteInput({ suggestions, ...props }: AutocompleteInputProps) {
  return (
    <div>
      <TextInput {...props} />  {/* Can use base anywhere */}
      <ul>
        {suggestions.map((s) => <li key={s}>{s}</li>)}
      </ul>
    </div>
  );
}

// Can use AutocompleteInput anywhere BaseInputProps is expected
function FormField({ Input = TextInput }: { Input: React.ComponentType<BaseInputProps> }) {
  return <Input label="Email" />;
}
// FormField can receive AutocompleteInput — substitution works
```

## I — Interface Segregation Principle

**Clients should not be forced to depend on interfaces they don't use.**

Avoid "fat interfaces" — split large prop interfaces into smaller, focused ones. Components shouldn't receive props they never use.

### Violation

```tsx
// ❌ MonsterProps — forces all consumers to handle irrelevant props
interface MonsterProps {
  user: User;
  onLogout: () => void;
  onAvatarChange: (file: File) => void;
  onEmailChange: (email: string) => void;
  onPasswordChange: (password: string) => void;
  notificationCount: number;
  onNotificationClick: () => void;
  sidebarOpen: boolean;
  onSidebarToggle: () => void;
}

function UserAvatarDisplay({ user }: MonsterProps) {
  // Only uses user.avatarUrl and user.name
  // But must receive (and forward) the entire MonsterProps
  return <img src={user.avatarUrl} alt={user.name} />;
}
```

### Applying ISP

```tsx
// ✅ Split into focused interfaces

interface UserIdentity {
  id: string;
  name: string;
  avatarUrl: string;
}

interface AuthActions {
  onLogout: () => void;
}

interface ProfileEditActions {
  onAvatarChange: (file: File) => void;
  onEmailChange: (email: string) => void;
  onPasswordChange: (password: string) => void;
}

interface NotificationProps {
  count: number;
  onClick: () => void;
}

// Each component takes only what it needs
function UserAvatar({ user }: { user: UserIdentity }) {
  return <img src={user.avatarUrl} alt={user.name} />;
}

function UserMenu({ user, onLogout }: { user: UserIdentity } & AuthActions) {
  return (
    <div>
      <UserAvatar user={user} />
      <button onClick={onLogout}>Sign out</button>
    </div>
  );
}

function ProfileEditor({ onAvatarChange, onEmailChange, onPasswordChange }: ProfileEditActions) {
  // Only receives what it needs
}
```

## D — Dependency Inversion Principle

**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

In frontend, this means components should depend on interfaces/types, not concrete implementations. Inject dependencies (data fetching, logging, navigation) through props or context rather than importing them directly.

### Violation

```tsx
// ❌ Component directly imports and uses a concrete service
import { userService } from '../services/UserService'; // concrete dependency
import { analyticsClient } from '../utils/analytics'; // another concrete dependency

function DeleteAccountButton({ userId }: { userId: string }) {
  const handleDelete = async () => {
    await userService.deleteUser(userId);              // hard-coded implementation
    analyticsClient.track('account_deleted', { userId }); // hard-coded
    window.location.href = '/goodbye';                 // hard-coded navigation
  };

  return <button onClick={handleDelete}>Delete Account</button>;
}
// Cannot test without calling real userService and real analytics
```

### Applying DIP

```tsx
// ✅ Depend on abstractions — inject concrete implementations

interface DeleteAccountButtonProps {
  userId: string;
  onDelete: (userId: string) => Promise<void>;    // abstraction
  onTrack: (event: string, data: Record<string, unknown>) => void; // abstraction
  onSuccess: () => void;                          // abstraction
}

function DeleteAccountButton({
  userId,
  onDelete,
  onTrack,
  onSuccess,
}: DeleteAccountButtonProps) {
  const handleDelete = async () => {
    await onDelete(userId);
    onTrack('account_deleted', { userId });
    onSuccess();
  };

  return <button onClick={handleDelete}>Delete Account</button>;
}

// Wire up concrete implementations at the app/page level
function AccountSettings() {
  const navigate = useNavigate();
  const { mutateAsync: deleteUser } = useMutation({ mutationFn: api.deleteUser });

  return (
    <DeleteAccountButton
      userId={currentUser.id}
      onDelete={deleteUser}
      onTrack={analytics.track}
      onSuccess={() => navigate('/goodbye')}
    />
  );
}

// Test with mocks — no real services needed
test('DeleteAccountButton calls onDelete and onSuccess', async () => {
  const onDelete = jest.fn().mockResolvedValue(undefined);
  const onTrack = jest.fn();
  const onSuccess = jest.fn();

  render(
    <DeleteAccountButton
      userId="u1"
      onDelete={onDelete}
      onTrack={onTrack}
      onSuccess={onSuccess}
    />
  );

  fireEvent.click(screen.getByRole('button'));
  await waitFor(() => expect(onSuccess).toHaveBeenCalled());
});
```

### DIP with Angular Services

```typescript
// ✅ Depend on interfaces, not concrete classes

// Abstract interface
abstract class Logger {
  abstract log(level: 'info' | 'warn' | 'error', message: string, data?: unknown): void;
}

// Concrete implementation
@Injectable()
class DatadogLogger extends Logger {
  log(level: 'info' | 'warn' | 'error', message: string, data?: unknown) {
    window.DD_RUM?.addAction(message, { level, ...data });
  }
}

// Alternative implementation (for testing)
@Injectable()
class ConsoleLogger extends Logger {
  log(level: 'info' | 'warn' | 'error', message: string, data?: unknown) {
    console[level](message, data);
  }
}

// Component depends on abstract Logger — not DatadogLogger
@Component({ ... })
class UserService {
  constructor(private logger: Logger) {} // abstraction

  async deleteUser(id: string) {
    this.logger.log('info', 'Deleting user', { id }); // works with any Logger
  }
}

// AppModule: wire concrete implementation
providers: [
  { provide: Logger, useClass: DatadogLogger },
]

// TestingModule: wire mock
providers: [
  { provide: Logger, useClass: ConsoleLogger },
]
```

## Common Mistakes

### 1. Treating SRP as "One Method Per File"

```
SRP is about "one reason to change" — not "only one function."
A 500-line module with 30 related helper functions can have one reason to change
if all 30 functions serve the same concern (e.g., date formatting utilities).
A 20-line component that mixes data fetching, business logic, and presentation
has multiple reasons to change and violates SRP.
```

### 2. Over-Applying OCP — Premature Abstraction

```tsx
// ❌ Building an extension point before it's needed
// "What if we need 17 variants someday?"
interface ButtonVariantStrategy {
  getClassName(): string;
  getIconPosition(): 'left' | 'right';
  shouldShowBorder(): boolean;
}

// ✅ Wait for the second variant before abstracting
// YAGNI — You Ain't Gonna Need It until you do
function Button({ variant = 'primary' }) {
  // Simple — add complexity when there are 2+ real variants
}
```

### 3. Confusing ISP with Making Everything Tiny

```
ISP says: "don't force clients to depend on what they don't use"
It doesn't say: "every interface must have one prop"

A UserProfile interface with 10 user-related props is fine.
The same 10 props forced onto a UserAvatar that only uses 2 is ISP violation.
```

## Interview Questions

### 1. How does the Single Responsibility Principle apply to React hooks?

**Answer:** A custom hook should encapsulate one concern. `useUser(userId)` fetches and caches user data. `useAnalytics()` provides tracking functions. `useDebounce(value, delay)` manages debounced state. When a hook handles data fetching, error formatting, analytics, AND manages a side effect — it has multiple reasons to change. Split it: the data hook changes when the API changes; the analytics hook changes when the tracking library changes. SRP makes hooks independently testable and composable — `useUserProfile` can compose `useUser` + `useAnalytics` without them knowing about each other.

### 2. How would you apply the Open/Closed Principle to a growing set of button variants?

**Answer:** Design the Button component to accept a `className` or `style` prop that passes through to the underlying element — this is the "open for extension" surface. Define a map of variant classes that's easy to add to without modifying the rendering logic. Use CVA (class-variance-authority) or similar utilities to define a type-safe variant system. When a new variant is needed, add it to the config object — no conditional logic modification. The more powerful form is the compound component pattern: `Button.Icon`, `Button.Label`, `Button.Badge` slots that consumers can compose, extending behavior without modifying the base.

### 3. What does the Dependency Inversion Principle mean for testing React components?

**Answer:** Components that import concrete dependencies (a specific API client, `localStorage`, `Date.now()`) are hard to test in isolation — you must mock the module. Components that receive dependencies as props or via context test with zero mocking: pass a jest.fn() directly. The practical pattern: define the interface (what the component needs), inject the implementation at the page/app level where the real services are available, and inject mocks in tests. This is why `onDelete`, `onTrack`, `onNavigate` props make components more testable than calling `api.deleteUser` directly — the test gives the component a spy, not a real network call.

### 4. How does the Interface Segregation Principle affect prop design in large component libraries?

**Answer:** A "god props" component that receives 40+ props and uses only 5-10 per use case violates ISP — every consumer gets all 40 props in their TypeScript types even if they're irrelevant. Split the interface by concern: `BaseInputProps` for the minimal input behavior, `ValidationInputProps extends BaseInputProps` for validation-aware inputs, `AsyncInputProps extends BaseInputProps` for async-loading variants. Consumers import only the interface they need. This also applies to context: one massive React context forces every consumer to re-render when any part changes — split into multiple focused contexts (UserContext, ThemeContext, NotificationContext) so consumers only subscribe to what they use.
