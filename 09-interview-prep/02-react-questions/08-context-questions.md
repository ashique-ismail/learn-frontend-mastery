# React Context API Interview Questions

## The Idea

**In plain English:** React Context is a way for different parts of your app to share information without having to pass it through every level in between. Think of it as a shared bulletin board that any component in your app can read from, instead of handing a note down a long chain of people one by one.

**Real-world analogy:** Imagine a school building with a public announcement speaker system. The principal (the Provider) broadcasts a message once, and every classroom (component) that has a speaker (uses the context) can hear it directly — without the message having to be passed from the office to the hallway monitor to the classroom door to each student one at a time.

- The principal = the Context Provider (holds and broadcasts the data)
- The announcement = the context value (the shared data, like a logged-in user or theme)
- Each classroom speaker = a component calling `useContext` (subscribes to receive the data)

---

## Overview

Context questions test your understanding of React's built-in state management solution. Knowing when to use Context, how to optimize it, and its trade-offs versus props or external state management is crucial for intermediate to senior roles.

---

## Core Context Questions

### Q1: What is React Context and when should you use it?

**Answer:**

Context provides a way to pass data through the component tree without manually passing props at every level. It's designed to share data that's "global" for a tree of components.

**The Props Drilling Problem:**

```javascript
// Without Context: Props drilling through multiple levels
function App() {
  const [user, setUser] = useState({ name: 'Alice', role: 'admin' });
  
  return <Layout user={user} />;
}

function Layout({ user }) {
  return (
    <div>
      <Header user={user} />
      <Sidebar user={user} />
      <Content user={user} />
    </div>
  );
}

function Header({ user }) {
  return <Navigation user={user} />;
}

function Navigation({ user }) {
  return <UserMenu user={user} />;
}

function UserMenu({ user }) {
  return <div>Welcome, {user.name}</div>;
}

// user prop passed through 5 levels, even though only UserMenu needs it!
```

**With Context:**

```javascript
// Create Context
const UserContext = createContext();

// Provider at the top
function App() {
  const [user, setUser] = useState({ name: 'Alice', role: 'admin' });
  
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

// No props drilling!
function Layout() {
  return (
    <div>
      <Header />
      <Sidebar />
      <Content />
    </div>
  );
}

function Header() {
  return <Navigation />;
}

function Navigation() {
  return <UserMenu />;
}

// Consume directly where needed
function UserMenu() {
  const user = useContext(UserContext);
  return <div>Welcome, {user.name}</div>;
}
```

**When to Use Context:**

**Good Use Cases:**
1. **Theme/Appearance:**
```javascript
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

function Button() {
  const { theme } = useContext(ThemeContext);
  return <button className={`btn-${theme}`}>Click</button>;
}
```

2. **User Authentication:**
```javascript
const AuthContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  
  const login = async (credentials) => {
    const userData = await authenticateUser(credentials);
    setUser(userData);
  };
  
  const logout = () => {
    setUser(null);
  };
  
  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      <Router />
    </AuthContext.Provider>
  );
}

function ProfilePage() {
  const { user, logout } = useContext(AuthContext);
  
  if (!user) return <Redirect to="/login" />;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

3. **Localization/i18n:**
```javascript
const LocaleContext = createContext();

function App() {
  const [locale, setLocale] = useState('en');
  const translations = getTranslations(locale);
  
  return (
    <LocaleContext.Provider value={{ locale, setLocale, t: translations }}>
      <Layout />
    </LocaleContext.Provider>
  );
}

function Greeting() {
  const { t } = useContext(LocaleContext);
  return <h1>{t('greeting')}</h1>;
}
```

4. **UI State (modals, sidebars):**
```javascript
const UIContext = createContext();

function App() {
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const [modalContent, setModalContent] = useState(null);
  
  return (
    <UIContext.Provider value={{
      sidebarOpen,
      setSidebarOpen,
      modalContent,
      setModalContent
    }}>
      <Layout />
    </UIContext.Provider>
  );
}
```

**When NOT to Use Context:**

**1. Frequent Updates:**
```javascript
// BAD: Context for high-frequency updates
const MouseContext = createContext();

function App() {
  const [mousePosition, setMousePosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setMousePosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return (
    <MouseContext.Provider value={mousePosition}>
      <Layout /> {/* Entire tree re-renders on every mouse move! */}
    </MouseContext.Provider>
  );
}

// BETTER: Use a state management library or component state
```

**2. Component Can Pass Props Directly:**
```javascript
// BAD: Context for nearby components
<UserContext.Provider value={user}>
  <UserProfile />
</UserContext.Provider>

// GOOD: Just use props
<UserProfile user={user} />
```

**3. Complex State Logic:**
```javascript
// BAD: Complex state in Context
const AppContext = createContext();

function App() {
  const [state, setState] = useState({
    users: [],
    posts: [],
    comments: [],
    filters: {},
    sortOrder: 'asc',
    // ... 20 more fields
  });
  
  // 50+ lines of state update logic
  
  return (
    <AppContext.Provider value={{ state, setState }}>
      <Layout />
    </AppContext.Provider>
  );
}

// BETTER: Use Redux, Zustand, or other state management
```

**Decision Tree:**

```
Do you need to pass data through multiple levels?
├─ No → Use props
└─ Yes → Continue
    
    Is the data needed by most components?
    ├─ No → Use props or component composition
    └─ Yes → Continue
        
        Does it update frequently?
        ├─ Yes → Use state management library
        └─ No → Use Context ✓
```

**What Interviewers Look For:**
- Understanding of props drilling problem
- Knowledge of appropriate use cases
- Awareness of Context limitations
- Ability to identify when NOT to use Context

---

### Q2: How do you optimize Context to prevent unnecessary re-renders?

**Answer:**

Context has a performance pitfall: all consumers re-render when the Context value changes. Optimization is crucial.

**Problem: Everything Re-renders:**

```javascript
const AppContext = createContext();

function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  const [theme, setTheme] = useState('light');
  
  // New object every render!
  const value = { user, theme, setUser, setTheme };
  
  return (
    <AppContext.Provider value={value}>
      <Header />
      <Content />
      <Footer />
    </AppContext.Provider>
  );
}

// Problem: When theme changes, ALL three components re-render
// Even if Header only uses user!
```

**Solution 1: useMemo the Context Value:**

```javascript
function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  const [theme, setTheme] = useState('light');
  
  // Stable object reference
  const value = useMemo(
    () => ({ user, theme, setUser, setTheme }),
    [user, theme]
  );
  
  return (
    <AppContext.Provider value={value}>
      <Header />
      <Content />
      <Footer />
    </AppContext.Provider>
  );
}

// Now value only changes when user or theme actually changes
```

**Solution 2: Split Contexts:**

```javascript
// Separate contexts for separate concerns
const UserContext = createContext();
const ThemeContext = createContext();

function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  const [theme, setTheme] = useState('light');
  
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <ThemeContext.Provider value={{ theme, setTheme }}>
        <Header />   {/* Only re-renders when theme changes */}
        <Content />  {/* Only re-renders when user changes */}
        <Footer />   {/* Only re-renders when theme changes */}
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}

function Header() {
  const { theme } = useContext(ThemeContext); // Only subscribes to theme
  return <header className={theme}>Header</header>;
}

function Content() {
  const { user } = useContext(UserContext); // Only subscribes to user
  return <div>Welcome, {user.name}</div>;
}
```

**Solution 3: Split State and Setters:**

```javascript
// State context
const UserStateContext = createContext();
// Setters context
const UserDispatchContext = createContext();

function UserProvider({ children }) {
  const [user, setUser] = useState({ name: 'Alice' });
  
  // Setters never change
  const dispatch = useMemo(
    () => ({
      updateName: (name) => setUser(u => ({ ...u, name })),
      updateEmail: (email) => setUser(u => ({ ...u, email }))
    }),
    []
  );
  
  return (
    <UserStateContext.Provider value={user}>
      <UserDispatchContext.Provider value={dispatch}>
        {children}
      </UserDispatchContext.Provider>
    </UserStateContext.Provider>
  );
}

// Read user (re-renders on user changes)
function DisplayUser() {
  const user = useContext(UserStateContext);
  return <div>{user.name}</div>;
}

// Update user (never re-renders!)
function UpdateUser() {
  const { updateName } = useContext(UserDispatchContext);
  return <button onClick={() => updateName('Bob')}>Change Name</button>;
}
```

**Solution 4: Context Selectors (Manual):**

```javascript
// Create a selector hook
function useUserSelector(selector) {
  const user = useContext(UserContext);
  const selectedValue = selector(user);
  
  // Memoize to prevent unnecessary re-renders
  return useMemo(() => selectedValue, [selectedValue]);
}

// Usage: Only re-renders when selected value changes
function UserName() {
  const name = useUserSelector(user => user.name);
  return <div>{name}</div>;
}

function UserEmail() {
  const email = useUserSelector(user => user.email);
  return <div>{email}</div>;
}

// If user.age changes, neither component re-renders!
```

**Solution 5: React.memo with Context:**

```javascript
const ThemeContext = createContext();

function App() {
  const [theme, setTheme] = useState('light');
  const [count, setCount] = useState(0);
  
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ThemedComponent />
    </ThemeContext.Provider>
  );
}

// Without memo: Re-renders when App re-renders (count changes)
function ThemedComponent() {
  const theme = useContext(ThemeContext);
  console.log('Render');
  return <div className={theme}>Themed</div>;
}

// With memo: Only re-renders when theme changes
const ThemedComponent = React.memo(function ThemedComponent() {
  const theme = useContext(ThemeContext);
  console.log('Render');
  return <div className={theme}>Themed</div>;
});
```

**Solution 6: Composition (Avoid Context Entirely):**

```javascript
// Instead of Context:
function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Layout />
    </ThemeContext.Provider>
  );
}

function Layout() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>{/* content */}</div>;
}

// Use composition:
function App() {
  const [theme, setTheme] = useState('light');
  
  return <Layout theme={theme} />;
}

function Layout({ theme }) {
  return <div className={theme}>{/* content */}</div>;
}

// Simpler and more performant!
```

**Complete Optimized Example:**

```javascript
// Split contexts
const TodoStateContext = createContext();
const TodoDispatchContext = createContext();

function TodoProvider({ children }) {
  const [todos, setTodos] = useState([]);
  
  // Memoized dispatch object (stable reference)
  const dispatch = useMemo(
    () => ({
      addTodo: (text) => {
        setTodos(todos => [...todos, { id: Date.now(), text, done: false }]);
      },
      toggleTodo: (id) => {
        setTodos(todos =>
          todos.map(t => t.id === id ? { ...t, done: !t.done } : t)
        );
      },
      deleteTodo: (id) => {
        setTodos(todos => todos.filter(t => t.id !== id));
      }
    }),
    []
  );
  
  return (
    <TodoStateContext.Provider value={todos}>
      <TodoDispatchContext.Provider value={dispatch}>
        {children}
      </TodoDispatchContext.Provider>
    </TodoStateContext.Provider>
  );
}

// Hook for reading state (re-renders on todos change)
function useTodos() {
  return useContext(TodoStateContext);
}

// Hook for actions (never re-renders)
function useTodoActions() {
  return useContext(TodoDispatchContext);
}

// Components
function TodoList() {
  const todos = useTodos(); // Re-renders when todos change
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

const TodoItem = React.memo(function TodoItem({ todo }) {
  const { toggleTodo, deleteTodo } = useTodoActions(); // Never re-renders from context
  
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.done}
        onChange={() => toggleTodo(todo.id)}
      />
      {todo.text}
      <button onClick={() => deleteTodo(todo.id)}>Delete</button>
    </li>
  );
});

function AddTodo() {
  const { addTodo } = useTodoActions(); // Never re-renders from context
  const [text, setText] = useState('');
  
  const handleSubmit = (e) => {
    e.preventDefault();
    addTodo(text);
    setText('');
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button>Add</button>
    </form>
  );
}
```

**What Interviewers Look For:**
- Understanding of Context re-render behavior
- Knowledge of useMemo for Context values
- Ability to split contexts effectively
- Awareness of state/dispatch separation pattern

---

### Q3: What's the difference between Context and prop drilling? When is prop drilling actually better?

**Answer:**

This question tests whether you blindly reach for Context or understand trade-offs.

**Prop Drilling Definition:**

Passing props through intermediate components that don't use them.

```javascript
// Prop drilling example
function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  
  return <PageLayout user={user} setUser={setUser} />;
}

function PageLayout({ user, setUser }) {
  return (
    <div>
      <Sidebar user={user} setUser={setUser} />
      <MainContent user={user} setUser={setUser} />
    </div>
  );
}

function Sidebar({ user, setUser }) {
  return (
    <aside>
      <UserWidget user={user} setUser={setUser} />
    </aside>
  );
}

function UserWidget({ user, setUser }) {
  return <div>{user.name}</div>;
}

// user passed through PageLayout and Sidebar, which don't use it
```

**When Prop Drilling is Better:**

**1. Shallow Component Trees (2-3 levels):**

```javascript
// OVERKILL to use Context here
function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Button />
    </ThemeContext.Provider>
  );
}

// BETTER: Just pass props
function App() {
  const [theme, setTheme] = useState('light');
  return <Button theme={theme} />;
}
```

**2. Explicit Dependencies:**

```javascript
// With props: Clear what component needs
function UserCard({ user, onUpdate, onDelete }) {
  // I know exactly what this component depends on
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={onUpdate}>Update</button>
      <button onClick={onDelete}>Delete</button>
    </div>
  );
}

// With Context: Hidden dependencies
function UserCard({ user }) {
  const { updateUser, deleteUser } = useContext(UserContext);
  // What else is in UserContext? Have to check the provider
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={() => updateUser(user.id)}>Update</button>
      <button onClick={() => deleteUser(user.id)}>Delete</button>
    </div>
  );
}
```

**3. Component Reusability:**

```javascript
// Props: Component is reusable anywhere
function DataTable({ data, columns, onRowClick }) {
  return (
    <table>
      {data.map(row => (
        <tr key={row.id} onClick={() => onRowClick(row)}>
          {columns.map(col => (
            <td key={col}>{row[col]}</td>
          ))}
        </tr>
      ))}
    </table>
  );
}

// Can use in any context:
<DataTable data={users} columns={['name', 'email']} onRowClick={handleUserClick} />
<DataTable data={products} columns={['title', 'price']} onRowClick={handleProductClick} />

// Context: Tied to specific context
function DataTable() {
  const { data, columns, onRowClick } = useContext(DataTableContext);
  // Only works within DataTableContext.Provider
  // Less reusable
}
```

**4. Testing:**

```javascript
// Props: Easy to test
test('UserCard displays user name', () => {
  render(<UserCard user={{ name: 'Alice' }} onUpdate={jest.fn()} />);
  expect(screen.getByText('Alice')).toBeInTheDocument();
});

// Context: Need to wrap in provider
test('UserCard displays user name', () => {
  render(
    <UserContext.Provider value={{ user: { name: 'Alice' }, updateUser: jest.fn() }}>
      <UserCard />
    </UserContext.Provider>
  );
  expect(screen.getByText('Alice')).toBeInTheDocument();
});
```

**5. Type Safety:**

```javascript
// Props: TypeScript tracks prop flow
interface UserCardProps {
  user: User;
  onUpdate: (id: string) => void;
}

function UserCard({ user, onUpdate }: UserCardProps) {
  // TypeScript ensures correct usage
}

// Context: Types can be harder to track
function UserCard() {
  const context = useContext(UserContext);
  // What if context is null? Need runtime checks
  if (!context) throw new Error('Must be used within UserProvider');
  
  const { user, onUpdate } = context;
}
```

**When Context is Better:**

**1. Deep Component Trees:**

```javascript
// Props drilling through 6+ levels is painful
<App>
  <Layout>
    <Dashboard>
      <Panel>
        <Section>
          <Widget>
            <UserInfo user={user} /> {/* Finally! */}
          </Widget>
        </Section>
      </Panel>
    </Dashboard>
  </Layout>
</App>

// Context is cleaner
<UserContext.Provider value={user}>
  <App>
    <Layout>
      <Dashboard>
        <Panel>
          <Section>
            <Widget>
              <UserInfo /> {/* useContext(UserContext) */}
            </Widget>
          </Section>
        </Panel>
      </Dashboard>
    </Layout>
  </App>
</UserContext.Provider>
```

**2. Many Components Need Same Data:**

```javascript
// Theme needed by 50+ components throughout the app
// Props drilling would be insane

<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>

// Any component can access theme
function AnyComponent() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>Content</div>;
}
```

**3. Cross-Cutting Concerns:**

```javascript
// Authentication, localization, theme - used everywhere
<AuthProvider>
  <LocaleProvider>
    <ThemeProvider>
      <App />
    </ThemeProvider>
  </LocaleProvider>
</AuthProvider>
```

**Hybrid Approach (Best of Both):**

```javascript
// Use Context for global concerns
<ThemeContext.Provider value={theme}>
  <AuthContext.Provider value={auth}>
    <App />
  </AuthContext.Provider>
</ThemeContext.Provider>

// Use props for local data flow
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </ul>
  );
}

function UserCard({ user }) {
  // Global from Context
  const theme = useContext(ThemeContext);
  const { deleteUser } = useContext(AuthContext);
  
  // Local from props
  return (
    <div className={theme}>
      <h3>{user.name}</h3>
      <button onClick={() => deleteUser(user.id)}>Delete</button>
    </div>
  );
}
```

**What Interviewers Look For:**
- Understanding of trade-offs
- Not blindly using Context everywhere
- Awareness of testability and reusability
- Balanced judgment

**Red Flags:**
- "Always use Context instead of props"
- "Prop drilling is always bad"
- Not considering component reusability
- Ignoring testing complexity

---

## Key Takeaways

**Essential Concepts:**
1. Context solves props drilling for global data
2. All consumers re-render when Context value changes
3. useMemo Context values to prevent unnecessary re-renders
4. Split contexts for different concerns
5. Separate state and dispatch contexts

**Best Practices:**
- Use Context for truly global data (theme, auth, locale)
- Memoize Context values with useMemo
- Split contexts to minimize re-renders
- Consider props for shallow trees
- Keep Context values stable

**Common Mistakes:**
- Creating new objects in Provider value every render
- Using Context for frequently updating data
- Not splitting contexts by concern
- Over-using Context when props would be simpler
- Forgetting to memoize dispatch functions

**Interview Tips:**
- Explain props drilling problem
- Show optimization techniques
- Discuss trade-offs vs props
- Demonstrate Context splitting
- Mention when NOT to use Context

**Red Flags to Avoid:**
- Using Context for everything
- Not understanding re-render behavior
- Ignoring performance implications
- Can't explain when props are better
- Creating new Context for every feature
