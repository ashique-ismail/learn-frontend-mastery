# Builder Pattern

## The Idea

**In plain English:** The Builder pattern is a way to put together a complicated thing one piece at a time, instead of trying to cram every detail into a single overwhelming instruction. Think of it as filling out a form step by step rather than blurting out all the answers at once.

**Real-world analogy:** Ordering a custom sandwich at a deli counter. You tell the person behind the counter each choice one at a time: the bread, then the filling, then the toppings, then the sauce. Only when you say "that's it" do they hand it over.

- The deli worker taking your order = the Builder object collecting your choices
- Each topping or filling you call out = a method call on the builder (like `.method('POST')` or `.retry(3)`)
- Saying "that's it, wrap it up" = calling `.build()` to produce the finished object

---

## Overview

The Builder pattern constructs complex objects step by step, separating the construction process from the final representation. It's most valuable when an object requires many optional parameters, when the creation order matters, or when you want to prevent partially-initialized objects. In TypeScript and frontend development, the builder pattern produces fluent, readable APIs for constructing complex configurations, test fixtures, and query objects.

## When to Use Builder

```
Builder is appropriate when:

✓ Object has many optional parameters (5+ constructor args is a smell)
✓ Different orderings of setup steps produce valid objects
✓ Construction must enforce invariants (A must be set before B)
✓ Multiple representations from the same build process
✓ Readable step-by-step construction > single complex constructor

Builder over alternatives:
  vs Object literal:   Builder enforces structure and can validate on .build()
  vs Many overloads:   Builder scales to N options without N! overloads
  vs Config object:    Builder enables method chaining + step validation
```

## Basic Builder

```typescript
// ❌ Before: constructor with too many parameters
class Request {
  constructor(
    url: string,
    method: string,
    headers: Record<string, string>,
    body: unknown,
    timeout: number,
    retries: number,
    auth: { type: string; token: string } | null,
    cache: boolean
  ) { /* ... */ }
}

// Callers must remember argument order:
new Request('/api/users', 'POST', {}, userData, 5000, 3, null, false);
// What does `false` mean? You have to look at the signature.

// ✅ After: fluent builder
class RequestBuilder {
  private _url: string;
  private _method: string = 'GET';
  private _headers: Record<string, string> = {};
  private _body: unknown = null;
  private _timeout: number = 10_000;
  private _retries: number = 0;
  private _auth: { type: string; token: string } | null = null;
  private _cache: boolean = false;

  constructor(url: string) {
    this._url = url;
  }

  method(method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE'): this {
    this._method = method;
    return this; // return this for chaining
  }

  header(name: string, value: string): this {
    this._headers[name] = value;
    return this;
  }

  json(body: unknown): this {
    this._body = body;
    this._headers['Content-Type'] = 'application/json';
    return this;
  }

  timeout(ms: number): this {
    if (ms <= 0) throw new Error('Timeout must be positive');
    this._timeout = ms;
    return this;
  }

  retry(count: number): this {
    this._retries = count;
    return this;
  }

  bearerAuth(token: string): this {
    this._auth = { type: 'Bearer', token };
    return this;
  }

  cache(): this {
    this._cache = true;
    return this;
  }

  build(): Request {
    // Validate invariants before producing the object
    if (this._method !== 'GET' && !this._body && !this._cache) {
      // warning: non-GET with no body is unusual
    }
    return new Request(
      this._url,
      this._method,
      this._headers,
      this._body,
      this._timeout,
      this._retries,
      this._auth,
      this._cache
    );
  }
}

// Usage — self-documenting
const request = new RequestBuilder('/api/users')
  .method('POST')
  .json({ name: 'Alice', email: 'alice@example.com' })
  .bearerAuth(token)
  .timeout(5000)
  .retry(3)
  .build();
```

## TypeScript Builder with Compile-Time Enforcement

Use TypeScript's type system to make required fields mandatory at compile time:

```typescript
// Type-safe builder: required fields enforced by types
type Required = { url: string; method: string };
type Optional = {
  headers: Record<string, string>;
  body: unknown;
  timeout: number;
};

// Builder that tracks which required fields have been set
class TypedRequestBuilder<TSet extends Partial<Required> = {}> {
  private config: Partial<Required & Optional> = {
    headers: {},
    timeout: 10_000,
  };

  url(url: string): TypedRequestBuilder<TSet & { url: string }> {
    this.config.url = url;
    return this as any;
  }

  method(m: string): TypedRequestBuilder<TSet & { method: string }> {
    this.config.method = m;
    return this as any;
  }

  header(name: string, value: string): this {
    this.config.headers![name] = value;
    return this;
  }

  // .build() only available when both url and method are set
  build(this: TypedRequestBuilder<Required>): Request {
    return new Request(this.config as Required & Optional);
  }
}

// ✅ Compiles
const r1 = new TypedRequestBuilder()
  .url('/api/posts')
  .method('GET')
  .build(); // OK — both required fields set

// ❌ Compile error: .build() not on type where 'method' is missing
const r2 = new TypedRequestBuilder()
  .url('/api/posts')
  .build(); // Error: Property 'build' does not exist
```

## Builder for Test Fixtures

One of the most practical frontend uses of Builder is creating test data with sensible defaults:

```typescript
// Test fixture builder
interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'editor' | 'viewer';
  plan: 'free' | 'pro' | 'enterprise';
  createdAt: Date;
  isActive: boolean;
}

class UserBuilder {
  private user: User = {
    id: 'user-1',
    email: 'test@example.com',
    name: 'Test User',
    role: 'viewer',
    plan: 'free',
    createdAt: new Date('2024-01-01'),
    isActive: true,
  };

  withId(id: string): this {
    this.user.id = id;
    return this;
  }

  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  asAdmin(): this {
    this.user.role = 'admin';
    return this;
  }

  asEditor(): this {
    this.user.role = 'editor';
    return this;
  }

  onProPlan(): this {
    this.user.plan = 'pro';
    return this;
  }

  inactive(): this {
    this.user.isActive = false;
    return this;
  }

  build(): User {
    // Return a copy — prevent mutation of the builder's internal state
    return { ...this.user };
  }

  buildMany(count: number, modifier?: (b: this, i: number) => void): User[] {
    return Array.from({ length: count }, (_, i) => {
      if (modifier) modifier(this, i);
      return this.withId(`user-${i + 1}`).build();
    });
  }
}

// In tests — readable, minimal, focused on what matters
test('admin can access settings', () => {
  const admin = new UserBuilder().asAdmin().build();
  expect(canAccess(admin, 'settings')).toBe(true);
});

test('inactive user cannot login', () => {
  const inactiveUser = new UserBuilder().inactive().build();
  expect(canLogin(inactiveUser)).toBe(false);
});

test('shows upgrade prompt for free plan users', () => {
  const freeUser = new UserBuilder().build(); // defaults to free
  render(<Dashboard user={freeUser} />);
  expect(screen.getByText(/upgrade/i)).toBeInTheDocument();
});

// Creating multiple related test objects
const [alice, bob, carol] = new UserBuilder().buildMany(3);
```

## Builder for Query/Filter Construction

```typescript
// Database query builder pattern
interface QueryConfig {
  table: string;
  conditions: string[];
  orderBy: string | null;
  limit: number | null;
  offset: number;
  fields: string[];
}

class QueryBuilder {
  private config: QueryConfig;

  constructor(table: string) {
    this.config = {
      table,
      conditions: [],
      orderBy: null,
      limit: null,
      offset: 0,
      fields: ['*'],
    };
  }

  select(...fields: string[]): this {
    this.config.fields = fields;
    return this;
  }

  where(condition: string): this {
    this.config.conditions.push(condition);
    return this;
  }

  orderBy(field: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.config.orderBy = `${field} ${direction}`;
    return this;
  }

  limit(n: number): this {
    this.config.limit = n;
    return this;
  }

  offset(n: number): this {
    this.config.offset = n;
    return this;
  }

  build(): string {
    const { table, conditions, orderBy, limit, offset, fields } = this.config;
    let sql = `SELECT ${fields.join(', ')} FROM ${table}`;
    if (conditions.length > 0) {
      sql += ` WHERE ${conditions.join(' AND ')}`;
    }
    if (orderBy) sql += ` ORDER BY ${orderBy}`;
    if (limit !== null) sql += ` LIMIT ${limit}`;
    if (offset > 0) sql += ` OFFSET ${offset}`;
    return sql;
  }
}

// Usage
const query = new QueryBuilder('products')
  .select('id', 'name', 'price')
  .where('inStock = true')
  .where('price < 100')
  .orderBy('price', 'ASC')
  .limit(20)
  .offset(40)
  .build();
// SELECT id, name, price FROM products WHERE inStock = true AND price < 100 ORDER BY price ASC LIMIT 20 OFFSET 40
```

## Builder for React Component Configuration

```typescript
// Builder for building complex form configurations
interface FieldConfig {
  name: string;
  label: string;
  type: 'text' | 'email' | 'select' | 'checkbox' | 'date';
  required: boolean;
  validators: Array<(value: unknown) => string | null>;
  options?: Array<{ value: string; label: string }>;
  placeholder?: string;
  hint?: string;
}

class FieldBuilder {
  private config: FieldConfig;

  constructor(name: string, label: string) {
    this.config = {
      name,
      label,
      type: 'text',
      required: false,
      validators: [],
    };
  }

  type(t: FieldConfig['type']): this {
    this.config.type = t;
    return this;
  }

  required(message = 'This field is required'): this {
    this.config.required = true;
    this.config.validators.push((v) => (!v ? message : null));
    return this;
  }

  email(): this {
    this.config.type = 'email';
    this.config.validators.push((v) =>
      typeof v === 'string' && !/\S+@\S+\.\S+/.test(v)
        ? 'Invalid email address'
        : null
    );
    return this;
  }

  minLength(n: number): this {
    this.config.validators.push((v) =>
      typeof v === 'string' && v.length < n
        ? `Must be at least ${n} characters`
        : null
    );
    return this;
  }

  options(opts: Array<{ value: string; label: string }>): this {
    this.config.type = 'select';
    this.config.options = opts;
    return this;
  }

  hint(text: string): this {
    this.config.hint = text;
    return this;
  }

  build(): FieldConfig {
    return { ...this.config, validators: [...this.config.validators] };
  }
}

// Usage in form schema definition
const registrationFields = [
  new FieldBuilder('email', 'Email Address')
    .email()
    .required()
    .build(),

  new FieldBuilder('password', 'Password')
    .type('text')
    .required()
    .minLength(8)
    .hint('Must be at least 8 characters')
    .build(),

  new FieldBuilder('country', 'Country')
    .options([
      { value: 'us', label: 'United States' },
      { value: 'gb', label: 'United Kingdom' },
    ])
    .required()
    .build(),
];
```

## Director Pattern (Optional)

A Director encapsulates common construction sequences:

```typescript
class RequestBuilder {
  // ... (from earlier example)
  static jsonPost(url: string, body: unknown, token: string): Request {
    return new RequestBuilder(url)
      .method('POST')
      .json(body)
      .bearerAuth(token)
      .timeout(5000)
      .retry(2)
      .build();
  }

  static authenticatedGet(url: string, token: string): Request {
    return new RequestBuilder(url)
      .method('GET')
      .bearerAuth(token)
      .cache()
      .build();
  }
}

// Common construction via static factory methods
const createPostRequest = RequestBuilder.jsonPost('/api/posts', postData, authToken);
const getUserRequest = RequestBuilder.authenticatedGet('/api/me', authToken);
```

## Common Mistakes

### 1. Not Returning `this` from Builder Methods

```typescript
// ❌ Methods return void — no chaining
class BadBuilder {
  setName(name: string): void {   // returns void!
    this.name = name;
  }
}

// ✅ Return this (or a new builder instance for immutable builders)
class GoodBuilder {
  setName(name: string): this {   // returns this
    this.name = name;
    return this;
  }
}
```

### 2. Mutating the Builder's Internal State in build()

```typescript
// ❌ build() mutates — calling build() twice produces inconsistent results
class BadBuilder {
  build(): Product {
    this.config.id = uuid(); // each call generates new id!
    return new Product(this.config);
  }
}

// ✅ build() produces a copy; builder state is immutable after build()
class GoodBuilder {
  build(): Product {
    return new Product({ ...this.config }); // spread — copy, not mutation
  }
}
```

### 3. Building Invalid Objects

```typescript
// ❌ No validation — can build incomplete objects
class ProductBuilder {
  build(): Product {
    return new Product(this.config); // config.price might be undefined!
  }
}

// ✅ Validate on build()
class ProductBuilder {
  build(): Product {
    if (!this.config.name) throw new Error('Product name is required');
    if (this.config.price === undefined) throw new Error('Price is required');
    if (this.config.price < 0) throw new Error('Price cannot be negative');
    return new Product(this.config);
  }
}
```

## Interview Questions

### 1. When should you use the Builder pattern versus a plain object literal?

**Answer:** Builder when: (1) the object has more than 4-5 optional parameters where a plain object literal would have too many `undefined` fields; (2) construction order matters or there are dependencies between fields (setting `type: 'email'` should also set a validator); (3) you need to validate invariants at construction time (`build()` throws if required fields are missing); (4) you want to reuse common construction sequences (test fixture builders). Plain object literal when: the structure is simple, all fields are required and obviously named, and there's no complex construction logic. The Builder pattern is especially valuable for test fixtures and configuration objects where the fluent API reads like a sentence describing the object.

### 2. How does the Builder pattern support the Open/Closed Principle?

**Answer:** Adding a new option to a Builder doesn't require changing existing callers — you add a new method without touching the existing API. Callers that don't need the new option continue to work unchanged. Contrast this with adding a parameter to a constructor: every single call site must be updated. Builder isolates change — new capabilities are new methods. Callers opt-in to new behavior by calling the new method; old callers are untouched. This also makes the builder itself open for extension — subclasses can add methods while inheriting the base fluent API.

### 3. How do you implement a type-safe Builder in TypeScript where some fields are required?

**Answer:** Use generic type parameters to track which required fields have been set. Each setter method returns a new type that includes the field it just set in its type parameter. The `build()` method is only available when the type parameter satisfies the required fields constraint. This is achieved with intersection types and conditional return types. The trick is that the runtime behavior is identical — it's the same class instance — but TypeScript sees different types for different call chains. Required fields missing → `build()` not on that type → compile error. This prevents runtime errors from partially-configured objects at the cost of some type gymnastics.
