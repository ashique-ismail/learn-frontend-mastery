# Atomic Design

## Overview

Atomic Design is a methodology for creating design systems by breaking UI into five hierarchical levels: atoms, molecules, organisms, templates, and pages. Coined by Brad Frost, the metaphor comes from chemistry — elements combine into more complex structures. The methodology addresses the core challenge of building consistent, maintainable UIs at scale: understanding which level of the hierarchy a component belongs to and why.

## The Five Levels

```
Atomic Design hierarchy:

ATOMS                    smallest unit; can't be broken down further
  ├─ Button              Input, Label, Icon, Checkbox, Badge, Spinner
  ├─ Typography          Heading, Paragraph, Caption
  └─ Form elements       TextInput, Select, Textarea

MOLECULES               groups of atoms that function together as a unit
  ├─ SearchBar           = Input atom + Button atom
  ├─ FormField           = Label atom + Input atom + ErrorMessage atom
  └─ ProductCard         = Image + Heading + Price + Button atoms

ORGANISMS              larger sections; may contain molecules + atoms
  ├─ Header             = Logo + Navigation (molecules) + SearchBar (molecule)
  ├─ ProductGrid        = multiple ProductCard (molecules)
  └─ CheckoutForm       = multiple FormField (molecules) + SubmitButton

TEMPLATES             page-level layouts; real component structure, placeholder content
  ├─ BlogPostTemplate   = Header + ArticleContent + Sidebar + Footer
  └─ ProductPageTemplate = Header + ProductDetail + Reviews + Footer

PAGES                templates with real content; what users see
  ├─ BlogPost("How to code") = BlogPostTemplate + actual article data
  └─ ProductPage("iPhone 15") = ProductPageTemplate + real product data
```

## Atoms

Atoms are the smallest, most abstract, most reusable components. They should work in any context with no knowledge of their surroundings:

```typescript
// Button atom — knows nothing about where it's used
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
}

function Button({
  variant = 'primary',
  size = 'md',
  isLoading,
  children,
  disabled,
  ...props
}: ButtonProps) {
  return (
    <button
      disabled={disabled || isLoading}
      className={clsx(buttonVariants({ variant, size }))}
      {...props}
    >
      {isLoading ? <Spinner size="sm" /> : children}
    </button>
  );
}

// TextInput atom
interface TextInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  hasError?: boolean;
}

function TextInput({ hasError, className, ...props }: TextInputProps) {
  return (
    <input
      className={clsx(
        'base-input-styles',
        hasError && 'border-red-500',
        className
      )}
      aria-invalid={hasError}
      {...props}
    />
  );
}

// Badge atom
function Badge({ children, variant = 'default' }: { children: ReactNode; variant?: string }) {
  return (
    <span className={clsx('badge', `badge-${variant}`)}>
      {children}
    </span>
  );
}
```

## Molecules

Molecules combine atoms to form a simple, self-contained functional unit:

```typescript
// FormField molecule: Label + Input + ErrorMessage
interface FormFieldProps {
  label: string;
  name: string;
  error?: string;
  hint?: string;
  required?: boolean;
  inputProps?: React.InputHTMLAttributes<HTMLInputElement>;
}

function FormField({ label, name, error, hint, required, inputProps }: FormFieldProps) {
  const inputId = `field-${name}`;
  const errorId = `${inputId}-error`;
  const hintId = `${inputId}-hint`;

  return (
    <div className="form-field">
      {/* Label atom */}
      <label htmlFor={inputId} className={clsx(required && 'required')}>
        {label}
        {required && <span aria-hidden="true">*</span>}
      </label>

      {/* TextInput atom */}
      <TextInput
        id={inputId}
        name={name}
        hasError={!!error}
        aria-describedby={[error && errorId, hint && hintId].filter(Boolean).join(' ')}
        {...inputProps}
      />

      {/* Error text atom */}
      {hint && <p id={hintId} className="hint-text">{hint}</p>}
      {error && <p id={errorId} className="error-text" role="alert">{error}</p>}
    </div>
  );
}

// SearchBar molecule: Input + Button
function SearchBar({ onSearch, placeholder = 'Search...' }: {
  onSearch: (query: string) => void;
  placeholder?: string;
}) {
  const [query, setQuery] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSearch(query.trim());
  };

  return (
    <form onSubmit={handleSubmit} role="search">
      <TextInput
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder={placeholder}
        aria-label="Search"
      />
      <Button type="submit" variant="primary">
        <SearchIcon aria-hidden="true" /> Search
      </Button>
    </form>
  );
}

// ProductCard molecule
function ProductCard({ product, onAddToCart }: {
  product: Product;
  onAddToCart: (id: string) => void;
}) {
  return (
    <article className="product-card" aria-label={product.name}>
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <p className="price">{formatCurrency(product.price)}</p>
      <Badge variant={product.inStock ? 'success' : 'warning'}>
        {product.inStock ? 'In Stock' : 'Low Stock'}
      </Badge>
      <Button onClick={() => onAddToCart(product.id)} disabled={!product.inStock}>
        Add to Cart
      </Button>
    </article>
  );
}
```

## Organisms

Organisms are complex UI sections that compose molecules and atoms into a distinct, self-contained section:

```typescript
// Header organism: Logo + Navigation + SearchBar + UserMenu
function Header() {
  const { user } = useAuth();
  const navigate = useNavigate();

  return (
    <header className="site-header">
      <Logo />

      {/* NavigationMenu molecule */}
      <NavigationMenu
        items={mainNavItems}
        activeItem={useCurrentPath()}
      />

      {/* SearchBar molecule */}
      <SearchBar onSearch={(q) => navigate(`/search?q=${encodeURIComponent(q)}`)} />

      {/* UserMenu molecule or SignInButton atom */}
      {user ? <UserMenu user={user} /> : <Button href="/login">Sign in</Button>}
    </header>
  );
}

// ProductGrid organism: heading + filters + grid of ProductCard molecules
function ProductGrid({
  products,
  isLoading,
  onAddToCart,
}: {
  products: Product[];
  isLoading: boolean;
  onAddToCart: (id: string) => void;
}) {
  const [filter, setFilter] = useState<'all' | 'inStock'>('all');

  const filtered = filter === 'inStock'
    ? products.filter((p) => p.inStock)
    : products;

  return (
    <section aria-label="Products">
      {/* FilterTabs molecule */}
      <FilterTabs
        options={[
          { value: 'all', label: 'All Products' },
          { value: 'inStock', label: 'In Stock' },
        ]}
        value={filter}
        onChange={(v) => setFilter(v as typeof filter)}
      />

      {isLoading ? (
        <LoadingSkeleton count={6} />
      ) : (
        <ul className="product-grid">
          {filtered.map((product) => (
            <li key={product.id}>
              <ProductCard product={product} onAddToCart={onAddToCart} />
            </li>
          ))}
        </ul>
      )}

      {!isLoading && filtered.length === 0 && (
        <EmptyState message="No products found" />
      )}
    </section>
  );
}
```

## Templates

Templates define the page layout using organisms/molecules as placeholders — no real data:

```typescript
// Templates define layout and structure, not content

interface ProductPageTemplateProps {
  header: ReactNode;        // Header organism
  productDetail: ReactNode; // ProductDetail organism
  reviews: ReactNode;       // Reviews organism
  relatedProducts: ReactNode; // RelatedProducts organism
  footer: ReactNode;        // Footer organism
}

function ProductPageTemplate({
  header,
  productDetail,
  reviews,
  relatedProducts,
  footer,
}: ProductPageTemplateProps) {
  return (
    <div className="product-page">
      {header}

      <main className="product-main">
        <div className="product-layout">
          <div className="product-detail-area">
            {productDetail}
          </div>
          <aside className="product-sidebar">
            {/* Sidebar content */}
          </aside>
        </div>

        <section className="reviews-section">
          {reviews}
        </section>

        <section className="related-section">
          {relatedProducts}
        </section>
      </main>

      {footer}
    </div>
  );
}
```

## Pages

Pages are template instances with real data — the actual routes users visit:

```typescript
// Page: template + real data + coordination logic

// pages/ProductPage.tsx
function ProductPage() {
  const { productId } = useParams<{ productId: string }>();
  const { data: product, isLoading } = useProduct(productId!);
  const { mutate: addToCart } = useAddToCart();

  if (isLoading) return <ProductPageSkeleton />;
  if (!product) return <NotFoundPage />;

  return (
    <ProductPageTemplate
      header={<Header />}
      productDetail={
        <ProductDetailOrganism
          product={product}
          onAddToCart={(qty) => addToCart({ productId: product.id, quantity: qty })}
        />
      }
      reviews={<ReviewsOrganism productId={product.id} />}
      relatedProducts={
        <RelatedProductsOrganism categoryId={product.categoryId} />
      }
      footer={<Footer />}
    />
  );
}
```

## Project Structure

```
src/
├── components/
│   ├── atoms/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.stories.tsx
│   │   │   ├── Button.test.tsx
│   │   │   └── index.ts
│   │   ├── TextInput/
│   │   ├── Badge/
│   │   └── ...
│   ├── molecules/
│   │   ├── FormField/
│   │   ├── SearchBar/
│   │   ├── ProductCard/
│   │   └── ...
│   ├── organisms/
│   │   ├── Header/
│   │   ├── ProductGrid/
│   │   ├── CheckoutForm/
│   │   └── ...
│   └── templates/
│       ├── ProductPageTemplate/
│       ├── BlogPostTemplate/
│       └── ...
└── pages/
    ├── ProductPage/
    ├── HomePage/
    └── ...
```

## When Atomic Design Helps and When It Doesn't

### When It Helps

```
✓ Design system / component library development
  → Clear hierarchy makes components discoverable and consistent

✓ Large teams where multiple designers and developers collaborate
  → Common vocabulary: "this is an organism, not a molecule"

✓ Applications with many UI patterns shared across features
  → Atoms ensure visual consistency; any change propagates

✓ Storybook-driven development
  → Atomic hierarchy maps directly to Storybook stories structure

✓ White-label products where the same components need
  different themes/skins
  → Atoms are the theming points
```

### When It's Over-Engineering

```
✗ Small applications with few components (< 20 components)
  → The taxonomy overhead outweighs the benefit

✗ Rapid prototyping / MVPs
  → Premature structure slows iteration

✗ Applications where components don't need to be reused
  → Feature-based structure is simpler: components/ live next to features/

✗ When the boundaries are genuinely unclear
  → Is this a molecule or an organism? If the team argues about it,
    the abstraction may not match the domain well
```

## Common Mistakes

### 1. Treating Atomic Design as a Strict Hierarchy for ALL Components

```
❌ Every component must fit into one of the five levels

// Some components are contextual — they exist only in one organism
// and have no business being a standalone molecule
// Forcing them through the hierarchy creates unnecessary files and complexity

✅ Apply atomic design to the design system / shared components
   Use a simpler co-location pattern for feature-specific components
```

### 2. Putting Business Logic in Atoms

```typescript
// ❌ Atom with business logic
function PriceAtom({ productId }: { productId: string }) {
  const { data: product } = useProduct(productId); // business logic!
  return <span>{formatCurrency(product?.price)}</span>;
}

// ✅ Atoms are presentational — receive data, render it
function PriceAtom({ amount, currency = 'USD' }: { amount: number; currency?: string }) {
  return <span>{new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount)}</span>;
}
// Business logic (fetching product) lives in the organism or page
```

### 3. Over-Granularizing Atoms

```typescript
// ❌ Every HTML element is an atom — too granular
const Div = ({ children }) => <div>{children}</div>;
const H1 = ({ children }) => <h1>{children}</h1>;

// These add zero value over native elements

// ✅ Atoms add meaningful design-system constraints
function Heading({ level = 2, size = 'md', children }) {
  // Heading atom applies design token sizing, spacing, font-weight
  // Not just a thin wrapper around an HTML tag
}
```

## Interview Questions

### 1. What are the five levels of Atomic Design and how do they relate to each other?

**Answer:** Atoms are the smallest indivisible UI elements: Button, Input, Label, Badge. Molecules group atoms into a simple functional unit: FormField (Label + Input + Error), SearchBar (Input + Button). Organisms are complex UI sections that compose molecules and atoms: Header, ProductGrid, CheckoutForm. Templates define the page structure with real organisms but placeholder content — the layout without the data. Pages are template instances with real content — what users actually see. Each level adds composition and specificity. Atoms maximize reuse; pages minimize it (each is unique). The hierarchy is about dependency direction: pages → templates → organisms → molecules → atoms.

### 2. At what level should data fetching logic live?

**Answer:** At the organism and page level — never in atoms or molecules. Atoms and molecules are presentational components that receive data as props. This keeps them reusable, testable in isolation, and previewable in Storybook without real data. Organisms can own local state and may trigger data fetching for their specific domain (a ReviewsOrganism fetches reviews for the product it's displaying). Pages are the primary orchestration layer — they fetch the main data, handle loading/error states, and pass data down to organisms. This separation ensures that ProductCard molecule can be used in a SearchResultsOrganism, a FeaturedProductsOrganism, and a WishlistOrganism without any of them caring about where the data came from.

### 3. When would you NOT use Atomic Design?

**Answer:** For small-to-medium applications with fewer than ~20-30 components, the taxonomy overhead exceeds the benefit — a simpler flat `components/` directory or feature-based co-location works better. For feature-specific components that exist only in one place and will never be reused — forcing them through the hierarchy creates unnecessary ceremony. During early MVP/prototype phases where the UI is volatile — establishing atomic hierarchy too early means the structure changes with every design iteration. Atomic Design is most valuable for design systems shared across multiple products or applications, or for large teams where consistent vocabulary and discoverability are critical.

### 4. How does Atomic Design interact with Storybook?

**Answer:** They complement each other naturally — Storybook stories mirror the atomic hierarchy. Each atom gets stories for all its visual states (primary, secondary, disabled, loading). Molecules get stories for all meaningful combinations of their atom states (FormField: default, error, hint, required). Organisms get stories for their content states (ProductGrid: loading, empty, populated, filtered). Templates get one story per layout variant. This setup enables visual development from the bottom up — design the atom in Storybook before using it anywhere, catch regressions per component with visual testing, and give designers a living documentation of every component in every state.
