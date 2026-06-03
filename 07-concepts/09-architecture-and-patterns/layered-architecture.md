# Layered Architecture

## Table of Contents
1. [Introduction](#introduction)
2. [Core Layers](#core-layers)
3. [Layer Dependencies](#layer-dependencies)
4. [Implementation Strategies](#implementation-strategies)
5. [Implementation Examples](#implementation-examples)
6. [Common Mistakes](#common-mistakes)
7. [Best Practices](#best-practices)
8. [When to Use/Not to Use](#when-to-usenot-to-use)
9. [Interview Questions](#interview-questions)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## Introduction

Layered Architecture (also known as N-Tier Architecture) is one of the most common architectural patterns. It organizes code into horizontal layers, where each layer has a specific role and responsibility. Layers depend only on layers below them, creating a unidirectional dependency flow.

This pattern promotes separation of concerns, making applications easier to understand, test, and maintain.

## Core Layers

### Standard 3-Layer Architecture

```
┌─────────────────────────────────────────────────┐
│          Presentation Layer                      │
│  (UI Components, Controllers, Views)             │
└─────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│          Business Logic Layer                    │
│  (Services, Domain Logic, Validation)            │
└─────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│          Data Access Layer                       │
│  (Repositories, Database, External APIs)         │
└─────────────────────────────────────────────────┘
```

### Extended 4-Layer Architecture

```
┌─────────────────────────────────────────────────┐
│          Presentation Layer                      │
│  (React Components, Angular Components)          │
└─────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│          Application Layer                       │
│  (Use Cases, Application Services)               │
└─────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│          Domain Layer                            │
│  (Business Entities, Domain Services)            │
└─────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│          Infrastructure Layer                    │
│  (Database, APIs, External Services)             │
└─────────────────────────────────────────────────┘
```

## Layer Dependencies

### Strict Layering

Each layer only depends on the layer directly below it:

```
Presentation → Business Logic → Data Access
```

```typescript
// Strict layering example

// DATA ACCESS LAYER
class UserRepository {
  async findById(id: string): Promise<UserData | null> {
    return await db.users.findOne({ id });
  }

  async save(user: UserData): Promise<void> {
    await db.users.upsert(user);
  }
}

// BUSINESS LOGIC LAYER (depends only on Data Access)
class UserService {
  constructor(private userRepository: UserRepository) {}

  async getUserProfile(id: string): Promise<UserProfile> {
    const userData = await this.userRepository.findById(id);
    if (!userData) {
      throw new Error('User not found');
    }

    // Business logic
    return {
      id: userData.id,
      name: `${userData.firstName} ${userData.lastName}`,
      email: userData.email,
      memberSince: userData.createdAt
    };
  }

  async updateUserEmail(id: string, newEmail: string): Promise<void> {
    // Business validation
    if (!this.isValidEmail(newEmail)) {
      throw new Error('Invalid email format');
    }

    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new Error('User not found');
    }

    user.email = newEmail;
    user.updatedAt = new Date();
    
    await this.userRepository.save(user);
  }

  private isValidEmail(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

// PRESENTATION LAYER (depends only on Business Logic)
function UserProfilePage({ userId }: Props) {
  const userService = useUserService();
  const [profile, setProfile] = useState<UserProfile | null>(null);

  useEffect(() => {
    userService.getUserProfile(userId).then(setProfile);
  }, [userId, userService]);

  if (!profile) return <div>Loading...</div>;

  return (
    <div>
      <h1>{profile.name}</h1>
      <p>{profile.email}</p>
      <p>Member since: {profile.memberSince.toLocaleDateString()}</p>
    </div>
  );
}
```

### Relaxed Layering

Layers can depend on any layer below them:

```
Presentation → Business Logic → Data Access
     ↓                ↓
     └────────────────┘
```

```typescript
// Relaxed layering example

// PRESENTATION LAYER
function UserListPage() {
  const userService = useUserService();
  const userRepository = useUserRepository(); // Can access Data Access directly

  // Quick read from repository (bypassing service)
  const [users, setUsers] = useState([]);
  
  useEffect(() => {
    userRepository.findAll().then(setUsers);
  }, [userRepository]);

  // Write through service (business logic)
  const handleActivateUser = async (userId: string) => {
    await userService.activateUser(userId);
  };

  return (
    <div>
      {users.map(user => (
        <UserCard
          key={user.id}
          user={user}
          onActivate={() => handleActivateUser(user.id)}
        />
      ))}
    </div>
  );
}
```

## Implementation Strategies

### Strategy 1: Classic 3-Layer with React

```typescript
// ============================================
// DATA ACCESS LAYER
// ============================================

interface ProductData {
  id: string;
  name: string;
  price: number;
  stock: number;
}

class ProductRepository {
  async findAll(): Promise<ProductData[]> {
    const response = await fetch('/api/products');
    return response.json();
  }

  async findById(id: string): Promise<ProductData | null> {
    const response = await fetch(`/api/products/${id}`);
    if (!response.ok) return null;
    return response.json();
  }

  async save(product: ProductData): Promise<void> {
    await fetch(`/api/products/${product.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(product)
    });
  }

  async create(product: Omit<ProductData, 'id'>): Promise<ProductData> {
    const response = await fetch('/api/products', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(product)
    });
    return response.json();
  }

  async delete(id: string): Promise<void> {
    await fetch(`/api/products/${id}`, { method: 'DELETE' });
  }
}

// ============================================
// BUSINESS LOGIC LAYER
// ============================================

interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
  inStock: boolean;
  displayPrice: string;
}

class ProductService {
  constructor(private repository: ProductRepository) {}

  async getAllProducts(): Promise<Product[]> {
    const productsData = await this.repository.findAll();
    return productsData.map(data => this.enrichProduct(data));
  }

  async getProduct(id: string): Promise<Product> {
    const data = await this.repository.findById(id);
    if (!data) {
      throw new Error('Product not found');
    }
    return this.enrichProduct(data);
  }

  async createProduct(name: string, price: number, stock: number): Promise<Product> {
    // Business validation
    this.validateProduct(name, price, stock);

    const data = await this.repository.create({ name, price, stock });
    return this.enrichProduct(data);
  }

  async updateStock(id: string, quantity: number): Promise<void> {
    if (quantity < 0) {
      throw new Error('Stock cannot be negative');
    }

    const product = await this.repository.findById(id);
    if (!product) {
      throw new Error('Product not found');
    }

    product.stock = quantity;
    await this.repository.save(product);
  }

  async purchaseProduct(id: string, quantity: number): Promise<void> {
    const product = await this.repository.findById(id);
    if (!product) {
      throw new Error('Product not found');
    }

    // Business logic
    if (product.stock < quantity) {
      throw new Error('Insufficient stock');
    }

    product.stock -= quantity;
    await this.repository.save(product);
  }

  private enrichProduct(data: ProductData): Product {
    return {
      ...data,
      inStock: data.stock > 0,
      displayPrice: `$${data.price.toFixed(2)}`
    };
  }

  private validateProduct(name: string, price: number, stock: number): void {
    if (!name || name.trim().length === 0) {
      throw new Error('Product name is required');
    }
    if (price <= 0) {
      throw new Error('Price must be positive');
    }
    if (stock < 0) {
      throw new Error('Stock cannot be negative');
    }
  }
}

// ============================================
// PRESENTATION LAYER
// ============================================

// Context for dependency injection
const ServiceContext = createContext<{
  productService: ProductService;
} | null>(null);

export function ServiceProvider({ children }: { children: ReactNode }) {
  const services = useMemo(() => ({
    productService: new ProductService(new ProductRepository())
  }), []);

  return (
    <ServiceContext.Provider value={services}>
      {children}
    </ServiceContext.Provider>
  );
}

function useProductService() {
  const context = useContext(ServiceContext);
  if (!context) {
    throw new Error('useProductService must be used within ServiceProvider');
  }
  return context.productService;
}

// Components
function ProductList() {
  const productService = useProductService();
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    productService.getAllProducts()
      .then(setProducts)
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, [productService]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="product-list">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}

function ProductCard({ product }: { product: Product }) {
  const productService = useProductService();
  const [purchasing, setPurchasing] = useState(false);

  const handlePurchase = async () => {
    setPurchasing(true);
    try {
      await productService.purchaseProduct(product.id, 1);
      alert('Purchase successful!');
    } catch (error) {
      alert(`Error: ${error.message}`);
    } finally {
      setPurchasing(false);
    }
  };

  return (
    <div className="product-card">
      <h3>{product.name}</h3>
      <p className="price">{product.displayPrice}</p>
      <p className={product.inStock ? 'in-stock' : 'out-of-stock'}>
        {product.inStock ? `${product.stock} in stock` : 'Out of stock'}
      </p>
      <button
        onClick={handlePurchase}
        disabled={!product.inStock || purchasing}
      >
        {purchasing ? 'Processing...' : 'Buy Now'}
      </button>
    </div>
  );
}

function CreateProductForm() {
  const productService = useProductService();
  const [name, setName] = useState('');
  const [price, setPrice] = useState('');
  const [stock, setStock] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setSubmitting(true);

    try {
      await productService.createProduct(
        name,
        parseFloat(price),
        parseInt(stock)
      );
      
      setName('');
      setPrice('');
      setStock('');
      alert('Product created!');
    } catch (error) {
      alert(`Error: ${error.message}`);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="create-product-form">
      <input
        type="text"
        value={name}
        onChange={e => setName(e.target.value)}
        placeholder="Product name"
        required
      />
      <input
        type="number"
        value={price}
        onChange={e => setPrice(e.target.value)}
        placeholder="Price"
        step="0.01"
        required
      />
      <input
        type="number"
        value={stock}
        onChange={e => setStock(e.target.value)}
        placeholder="Stock quantity"
        required
      />
      <button type="submit" disabled={submitting}>
        {submitting ? 'Creating...' : 'Create Product'}
      </button>
    </form>
  );
}
```

### Strategy 2: Classic 3-Layer with Angular

```typescript
// ============================================
// DATA ACCESS LAYER
// ============================================

export interface ProductData {
  id: string;
  name: string;
  price: number;
  stock: number;
}

@Injectable({ providedIn: 'root' })
export class ProductRepository {
  constructor(private http: HttpClient) {}

  findAll(): Observable<ProductData[]> {
    return this.http.get<ProductData[]>('/api/products');
  }

  findById(id: string): Observable<ProductData> {
    return this.http.get<ProductData>(`/api/products/${id}`);
  }

  save(product: ProductData): Observable<void> {
    return this.http.put<void>(`/api/products/${product.id}`, product);
  }

  create(product: Omit<ProductData, 'id'>): Observable<ProductData> {
    return this.http.post<ProductData>('/api/products', product);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`/api/products/${id}`);
  }
}

// ============================================
// BUSINESS LOGIC LAYER
// ============================================

export interface Product {
  id: string;
  name: string;
  price: number;
  stock: number;
  inStock: boolean;
  displayPrice: string;
}

@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(private repository: ProductRepository) {}

  getAllProducts(): Observable<Product[]> {
    return this.repository.findAll().pipe(
      map(productsData => productsData.map(data => this.enrichProduct(data)))
    );
  }

  getProduct(id: string): Observable<Product> {
    return this.repository.findById(id).pipe(
      map(data => this.enrichProduct(data))
    );
  }

  createProduct(name: string, price: number, stock: number): Observable<Product> {
    this.validateProduct(name, price, stock);

    return this.repository.create({ name, price, stock }).pipe(
      map(data => this.enrichProduct(data))
    );
  }

  updateStock(id: string, quantity: number): Observable<void> {
    if (quantity < 0) {
      return throwError(() => new Error('Stock cannot be negative'));
    }

    return this.repository.findById(id).pipe(
      switchMap(product => {
        product.stock = quantity;
        return this.repository.save(product);
      })
    );
  }

  purchaseProduct(id: string, quantity: number): Observable<void> {
    return this.repository.findById(id).pipe(
      switchMap(product => {
        if (product.stock < quantity) {
          return throwError(() => new Error('Insufficient stock'));
        }

        product.stock -= quantity;
        return this.repository.save(product);
      })
    );
  }

  private enrichProduct(data: ProductData): Product {
    return {
      ...data,
      inStock: data.stock > 0,
      displayPrice: `$${data.price.toFixed(2)}`
    };
  }

  private validateProduct(name: string, price: number, stock: number): void {
    if (!name || name.trim().length === 0) {
      throw new Error('Product name is required');
    }
    if (price <= 0) {
      throw new Error('Price must be positive');
    }
    if (stock < 0) {
      throw new Error('Stock cannot be negative');
    }
  }
}

// ============================================
// PRESENTATION LAYER
// ============================================

@Component({
  selector: 'app-product-list',
  template: `
    <div class="product-list">
      <div *ngIf="loading">Loading...</div>
      <div *ngIf="error" class="error">Error: {{ error }}</div>

      <div *ngIf="!loading && !error" class="products">
        <app-product-card
          *ngFor="let product of products"
          [product]="product"
          (purchase)="handlePurchase($event)"
        ></app-product-card>
      </div>
    </div>
  `
})
export class ProductListComponent implements OnInit {
  products: Product[] = [];
  loading = true;
  error: string | null = null;

  constructor(private productService: ProductService) {}

  ngOnInit(): void {
    this.productService.getAllProducts().subscribe({
      next: (products) => {
        this.products = products;
        this.loading = false;
      },
      error: (error) => {
        this.error = error.message;
        this.loading = false;
      }
    });
  }

  handlePurchase(productId: string): void {
    this.productService.purchaseProduct(productId, 1).subscribe({
      next: () => {
        alert('Purchase successful!');
        this.ngOnInit(); // Reload products
      },
      error: (error) => {
        alert(`Error: ${error.message}`);
      }
    });
  }
}

@Component({
  selector: 'app-product-card',
  template: `
    <div class="product-card">
      <h3>{{ product.name }}</h3>
      <p class="price">{{ product.displayPrice }}</p>
      <p [class]="product.inStock ? 'in-stock' : 'out-of-stock'">
        {{ product.inStock ? product.stock + ' in stock' : 'Out of stock' }}
      </p>
      <button
        (click)="onPurchaseClick()"
        [disabled]="!product.inStock || purchasing"
      >
        {{ purchasing ? 'Processing...' : 'Buy Now' }}
      </button>
    </div>
  `
})
export class ProductCardComponent {
  @Input() product!: Product;
  @Output() purchase = new EventEmitter<string>();
  
  purchasing = false;

  onPurchaseClick(): void {
    this.purchasing = true;
    this.purchase.emit(this.product.id);
    // Reset in parent component callback
    setTimeout(() => this.purchasing = false, 1000);
  }
}

@Component({
  selector: 'app-create-product-form',
  template: `
    <form [formGroup]="productForm" (ngSubmit)="handleSubmit()" class="create-product-form">
      <input
        formControlName="name"
        placeholder="Product name"
      />
      <input
        type="number"
        formControlName="price"
        placeholder="Price"
        step="0.01"
      />
      <input
        type="number"
        formControlName="stock"
        placeholder="Stock quantity"
      />
      <button type="submit" [disabled]="!productForm.valid || submitting">
        {{ submitting ? 'Creating...' : 'Create Product' }}
      </button>
    </form>
  `
})
export class CreateProductFormComponent {
  productForm = this.fb.group({
    name: ['', Validators.required],
    price: [0, [Validators.required, Validators.min(0.01)]],
    stock: [0, [Validators.required, Validators.min(0)]]
  });

  submitting = false;

  constructor(
    private fb: FormBuilder,
    private productService: ProductService
  ) {}

  handleSubmit(): void {
    if (!this.productForm.valid) return;

    this.submitting = true;
    const { name, price, stock } = this.productForm.value;

    this.productService.createProduct(name!, price!, stock!).subscribe({
      next: () => {
        this.productForm.reset();
        alert('Product created!');
        this.submitting = false;
      },
      error: (error) => {
        alert(`Error: ${error.message}`);
        this.submitting = false;
      }
    });
  }
}
```

### Strategy 3: 4-Layer Architecture with Domain Layer

```typescript
// ============================================
// INFRASTRUCTURE LAYER (Database)
// ============================================

class OrderRepository {
  async save(order: Order): Promise<void> {
    const data = this.toDatabase(order);
    await db.orders.upsert(data);
  }

  async findById(id: string): Promise<Order | null> {
    const data = await db.orders.findOne({ id });
    return data ? this.toDomain(data) : null;
  }

  private toDatabase(order: Order): any {
    return {
      id: order.getId(),
      customerId: order.getCustomerId(),
      items: order.getItems(),
      status: order.getStatus(),
      total: order.getTotal()
    };
  }

  private toDomain(data: any): Order {
    const order = new Order(data.id, data.customerId);
    data.items.forEach((item: any) => {
      order.addItem(item.productId, item.quantity, item.price);
    });
    return order;
  }
}

// ============================================
// DOMAIN LAYER (Business Entities)
// ============================================

class Order {
  private items: OrderItem[] = [];
  private status: OrderStatus = OrderStatus.Draft;

  constructor(
    private readonly id: string,
    private readonly customerId: string
  ) {}

  addItem(productId: string, quantity: number, price: number): void {
    if (this.status !== OrderStatus.Draft) {
      throw new Error('Cannot modify confirmed order');
    }

    const existingItem = this.items.find(item => item.productId === productId);
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.push(new OrderItem(productId, quantity, price));
    }
  }

  confirm(): void {
    if (this.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    this.status = OrderStatus.Confirmed;
  }

  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.getTotal(), 0);
  }

  getId(): string {
    return this.id;
  }

  getCustomerId(): string {
    return this.customerId;
  }

  getItems(): OrderItem[] {
    return [...this.items];
  }

  getStatus(): OrderStatus {
    return this.status;
  }
}

class OrderItem {
  constructor(
    public readonly productId: string,
    private quantity: number,
    private readonly price: number
  ) {}

  increaseQuantity(amount: number): void {
    this.quantity += amount;
  }

  getTotal(): number {
    return this.quantity * this.price;
  }
}

enum OrderStatus {
  Draft = 'draft',
  Confirmed = 'confirmed',
  Shipped = 'shipped',
  Delivered = 'delivered'
}

// ============================================
// APPLICATION LAYER (Use Cases)
// ============================================

class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository
  ) {}

  async execute(customerId: string, items: Array<{ productId: string; quantity: number }>): Promise<string> {
    const order = new Order(this.generateId(), customerId);

    for (const item of items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) {
        throw new Error(`Product ${item.productId} not found`);
      }

      order.addItem(item.productId, item.quantity, product.price);
    }

    await this.orderRepository.save(order);
    return order.getId();
  }

  private generateId(): string {
    return `order-${Date.now()}`;
  }
}

class ConfirmOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private paymentService: PaymentService
  ) {}

  async execute(orderId: string): Promise<void> {
    const order = await this.orderRepository.findById(orderId);
    if (!order) {
      throw new Error('Order not found');
    }

    // Process payment
    const paymentResult = await this.paymentService.processPayment(
      order.getCustomerId(),
      order.getTotal()
    );

    if (!paymentResult.success) {
      throw new Error('Payment failed');
    }

    // Confirm order (domain logic)
    order.confirm();

    await this.orderRepository.save(order);
  }
}

// ============================================
// PRESENTATION LAYER (React)
// ============================================

function CreateOrderPage() {
  const createOrderUseCase = useCreateOrderUseCase();
  const [items, setItems] = useState<Array<{ productId: string; quantity: number }>>([]);

  const handleSubmit = async () => {
    try {
      const orderId = await createOrderUseCase.execute('customer-id', items);
      alert(`Order created: ${orderId}`);
    } catch (error) {
      alert(`Error: ${error.message}`);
    }
  };

  return (
    <div>
      {/* Order form */}
      <button onClick={handleSubmit}>Create Order</button>
    </div>
  );
}
```

## Common Mistakes

### 1. Layer Bypass

```typescript
// BAD: Presentation layer directly accessing data layer
function ProductList() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    // Bypassing business logic layer!
    fetch('/api/products')
      .then(res => res.json())
      .then(setProducts);
  }, []);
}

// GOOD: Going through proper layers
function ProductList() {
  const productService = useProductService();
  const [products, setProducts] = useState([]);

  useEffect(() => {
    productService.getAllProducts().then(setProducts);
  }, [productService]);
}
```

### 2. Business Logic in Presentation Layer

```typescript
// BAD: Validation in component
function CreateProductForm() {
  const handleSubmit = (name: string, price: number) => {
    // Business logic in presentation layer!
    if (!name || name.trim().length === 0) {
      alert('Name is required');
      return;
    }
    if (price <= 0) {
      alert('Price must be positive');
      return;
    }
    
    // Save product...
  };
}

// GOOD: Validation in business logic layer
class ProductService {
  createProduct(name: string, price: number): Promise<Product> {
    this.validateProduct(name, price);
    return this.repository.create({ name, price });
  }

  private validateProduct(name: string, price: number): void {
    if (!name || name.trim().length === 0) {
      throw new Error('Name is required');
    }
    if (price <= 0) {
      throw new Error('Price must be positive');
    }
  }
}
```

### 3. Circular Dependencies

```typescript
// BAD: Layers depend on each other
class ProductService {
  constructor(private component: ProductListComponent) {} // Wrong!
}

class ProductListComponent {
  constructor(private service: ProductService) {}
}

// GOOD: Unidirectional dependencies
class ProductService {
  // No knowledge of presentation layer
}

class ProductListComponent {
  constructor(private service: ProductService) {} // Correct direction
}
```

## Best Practices

### 1. Keep Layers Thin

```typescript
// Each layer should have minimal code

// Data Access: Just data operations
class ProductRepository {
  findAll(): Promise<Product[]> {
    return fetch('/api/products').then(res => res.json());
  }
}

// Business Logic: Just business rules
class ProductService {
  constructor(private repo: ProductRepository) {}
  
  async getAvailableProducts(): Promise<Product[]> {
    const products = await this.repo.findAll();
    return products.filter(p => p.stock > 0);
  }
}

// Presentation: Just rendering
function ProductList() {
  const service = useProductService();
  const [products] = useAsync(() => service.getAvailableProducts());
  return <div>{products.map(p => <ProductCard product={p} />)}</div>;
}
```

### 2. Use Dependency Injection

```typescript
// Define interfaces in each layer
interface IProductRepository {
  findAll(): Promise<Product[]>;
}

class ProductService {
  // Depend on interface, not implementation
  constructor(private repository: IProductRepository) {}
}

// Easy to test with mocks
const mockRepo: IProductRepository = {
  findAll: jest.fn().mockResolvedValue([])
};
const service = new ProductService(mockRepo);
```

### 3. Define Clear Boundaries

```typescript
// Each layer has clear responsibilities

// Data Access: Database operations
class OrderRepository {
  async save(order: Order): Promise<void> { }
  async findById(id: string): Promise<Order | null> { }
}

// Business Logic: Business rules
class OrderService {
  async placeOrder(items: OrderItem[]): Promise<Order> {
    // Validate, calculate, apply business rules
  }
}

// Presentation: User interaction
function OrderForm() {
  // Handle user input, display results
}
```

### 4. Handle Cross-Cutting Concerns Properly

```typescript
// Logging, authentication, etc.

// As middleware/interceptors in data layer
class LoggingRepository implements IProductRepository {
  constructor(
    private inner: IProductRepository,
    private logger: Logger
  ) {}

  async findAll(): Promise<Product[]> {
    this.logger.log('Fetching all products');
    const products = await this.inner.findAll();
    this.logger.log(`Found ${products.length} products`);
    return products;
  }
}

// Or as decorators in business layer
class LoggingProductService {
  constructor(
    private inner: ProductService,
    private logger: Logger
  ) {}

  async getProducts(): Promise<Product[]> {
    this.logger.log('Getting products');
    try {
      return await this.inner.getProducts();
    } catch (error) {
      this.logger.error('Error getting products', error);
      throw error;
    }
  }
}
```

## When to Use/Not to Use

### Use Layered Architecture When:

1. **Traditional Business Applications**: CRUD operations, forms, reports
2. **Team Experience**: Team familiar with layered approach
3. **Clear Separation Needed**: Distinct presentation, business, data concerns
4. **Multiple Clients**: Same business logic used by web, mobile, API

### Don't Use When:

1. **Complex Domain Logic**: Consider DDD or Clean Architecture instead
2. **Microservices**: Each service might have simpler internal structure
3. **Highly Distributed Systems**: Layers become artificial boundaries
4. **Event-Driven Systems**: Consider different patterns

## Interview Questions

### Junior Level

**Q: What is layered architecture?**

A: An architectural pattern that organizes code into horizontal layers, where each layer has specific responsibilities. Common layers include Presentation, Business Logic, and Data Access.

**Q: What is the dependency rule in layered architecture?**

A: Layers should only depend on layers below them, creating a unidirectional dependency flow from top (presentation) to bottom (data access).

### Mid Level

**Q: What's the difference between strict and relaxed layering?**

A: Strict layering means each layer can only depend on the layer directly below it. Relaxed layering allows layers to depend on any layer below them, potentially skipping intermediate layers.

**Q: How do you handle cross-cutting concerns like logging?**

A: Use decorators, interceptors, or middleware that wrap layer components. Alternatively, use aspect-oriented programming or dependency injection to provide cross-cutting services.

### Senior Level

**Q: How does layered architecture compare to Clean Architecture?**

A: Traditional layered architecture has database at the bottom with dependencies flowing down. Clean Architecture inverts this, placing domain at the center with dependencies pointing inward, making the domain independent of infrastructure.

**Q: How do you prevent layer violations in a large team?**

A: Use:
- Module boundaries and folder structure
- Linting rules (e.g., eslint-plugin-import with restrictions)
- Architecture decision records (ADRs)
- Code reviews focused on architectural compliance
- Automated tests checking dependencies

```typescript
// Example: ESLint rule to prevent layer violations
{
  "rules": {
    "import/no-restricted-paths": ["error", {
      "zones": [
        {
          "target": "./src/data-access",
          "from": "./src/business-logic"
        },
        {
          "target": "./src/business-logic",
          "from": "./src/presentation"
        }
      ]
    }]
  }
}
```

## Key Takeaways

1. Layered architecture organizes code into horizontal layers with specific responsibilities
2. Standard layers: Presentation, Business Logic, Data Access
3. Dependencies flow downward (or inward in some variants)
4. Each layer should be replaceable without affecting other layers
5. Strict layering restricts dependencies to immediate layer below
6. Relaxed layering allows bypassing intermediate layers
7. Keep layers thin and focused on single responsibility
8. Use dependency injection for flexibility and testability
9. Handle cross-cutting concerns with decorators or middleware
10. Best for traditional business applications with clear separation of concerns

## Resources

### Books
- "Patterns of Enterprise Application Architecture" by Martin Fowler
- "Clean Architecture" by Robert C. Martin
- "Domain-Driven Design" by Eric Evans

### Articles
- Layered Architecture by Microsoft: https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ee658109(v=pandp.10)
- N-Tier Architecture: https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/

### Examples
- Spring Framework (Java): Layered architecture by default
- ASP.NET Core: Recommended layered structure
- Angular Style Guide: Recommended folder structure

### Tools
- ESLint import rules: Enforce layer boundaries
- ArchUnit: Architecture testing
- NDepend: .NET architecture analysis
