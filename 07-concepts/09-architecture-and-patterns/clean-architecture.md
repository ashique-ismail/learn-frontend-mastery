# Clean Architecture

## Table of Contents
1. [Introduction](#introduction)
2. [Core Principles](#core-principles)
3. [Architecture Layers](#architecture-layers)
4. [The Dependency Rule](#the-dependency-rule)
5. [Implementation Examples](#implementation-examples)
6. [Common Mistakes](#common-mistakes)
7. [Best Practices](#best-practices)
8. [When to Use/Not to Use](#when-to-usenot-to-use)
9. [Interview Questions](#interview-questions)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## Introduction

Clean Architecture is a software design philosophy introduced by Robert C. Martin (Uncle Bob) that emphasizes separation of concerns and independence of frameworks, UI, databases, and external agencies. The architecture organizes code into concentric layers, with business logic at the center and infrastructure concerns at the edges.

The primary goal is to create systems that are independent of frameworks, testable, independent of UI, independent of databases, and independent of any external agency.

## Core Principles

### The Dependency Rule

Dependencies point inward. Inner layers know nothing about outer layers:

```
┌─────────────────────────────────────────────────────┐
│              Frameworks & Drivers                    │
│  ┌───────────────────────────────────────────────┐  │
│  │     Interface Adapters (Controllers, etc.)    │  │
│  │  ┌─────────────────────────────────────────┐ │  │
│  │  │    Application Business Rules (Use Cases)│ │  │
│  │  │  ┌───────────────────────────────────┐  │ │  │
│  │  │  │ Enterprise Business Rules (Entities)│ │ │  │
│  │  │  │                                     │  │ │  │
│  │  │  │  ← Dependencies point inward        │  │ │  │
│  │  │  └───────────────────────────────────┘  │ │  │
│  │  └─────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Layer Independence

Each layer can be developed, tested, and maintained independently:

```typescript
// Inner layers define interfaces
// Outer layers implement them

// Domain Layer (innermost)
interface PaymentGateway {
  charge(amount: Money, method: PaymentMethod): Promise<PaymentResult>;
}

// Use Case Layer
class ProcessOrderUseCase {
  constructor(private paymentGateway: PaymentGateway) {}
  // Uses interface, doesn't know about implementation
}

// Infrastructure Layer (outermost)
class StripePaymentGateway implements PaymentGateway {
  async charge(amount: Money, method: PaymentMethod): Promise<PaymentResult> {
    // Stripe-specific implementation
  }
}
```

## Architecture Layers

### Layer 1: Entities (Enterprise Business Rules)

The innermost layer contains enterprise-wide business rules and core domain logic:

```typescript
// Pure business entities with no framework dependencies

export class Order {
  private items: OrderItem[] = [];
  private status: OrderStatus = OrderStatus.Draft;
  private readonly createdAt: Date;

  constructor(
    private readonly id: string,
    private readonly customerId: string
  ) {
    this.createdAt = new Date();
  }

  // Business rules encoded in the entity
  addItem(productId: string, quantity: number, unitPrice: Money): void {
    if (this.status !== OrderStatus.Draft) {
      throw new OrderNotModifiableError('Cannot modify non-draft order');
    }

    const existingItem = this.findItem(productId);
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.push(new OrderItem(productId, quantity, unitPrice));
    }
  }

  confirm(): void {
    if (this.items.length === 0) {
      throw new EmptyOrderError('Cannot confirm order without items');
    }
    if (this.status !== OrderStatus.Draft) {
      throw new OrderNotModifiableError('Order already confirmed');
    }

    this.status = OrderStatus.Confirmed;
  }

  cancel(): void {
    if (this.status === OrderStatus.Delivered) {
      throw new OrderNotCancellableError('Cannot cancel delivered order');
    }
    this.status = OrderStatus.Cancelled;
  }

  getTotalAmount(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.getTotalPrice()),
      new Money(0, 'USD')
    );
  }

  private findItem(productId: string): OrderItem | undefined {
    return this.items.find(item => item.productId === productId);
  }
}

export class OrderItem {
  constructor(
    public readonly productId: string,
    private quantity: number,
    private readonly unitPrice: Money
  ) {
    if (quantity <= 0) {
      throw new InvalidQuantityError('Quantity must be positive');
    }
  }

  increaseQuantity(amount: number): void {
    if (amount <= 0) {
      throw new InvalidQuantityError('Amount must be positive');
    }
    this.quantity += amount;
  }

  getTotalPrice(): Money {
    return this.unitPrice.multiply(this.quantity);
  }
}

// Value Objects (part of entity layer)
export class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string
  ) {
    if (amount < 0) {
      throw new InvalidMoneyError('Amount cannot be negative');
    }
  }

  add(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  private ensureSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new CurrencyMismatchError('Cannot operate on different currencies');
    }
  }
}
```

### Layer 2: Use Cases (Application Business Rules)

Use cases orchestrate the flow of data to and from entities:

```typescript
// Use Case Interface
export interface UseCase<Request, Response> {
  execute(request: Request): Promise<Response>;
}

// ============================================
// Create Order Use Case
// ============================================

export interface CreateOrderRequest {
  customerId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
}

export interface CreateOrderResponse {
  orderId: string;
  success: boolean;
  error?: string;
}

export class CreateOrderUseCase implements UseCase<CreateOrderRequest, CreateOrderResponse> {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository,
    private orderFactory: OrderFactory
  ) {}

  async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    try {
      // 1. Create new order
      const order = this.orderFactory.createOrder(request.customerId);

      // 2. Add items to order
      for (const itemRequest of request.items) {
        const product = await this.productRepository.findById(itemRequest.productId);
        
        if (!product) {
          return {
            orderId: '',
            success: false,
            error: `Product ${itemRequest.productId} not found`
          };
        }

        order.addItem(
          product.getId(),
          itemRequest.quantity,
          product.getPrice()
        );
      }

      // 3. Save order
      await this.orderRepository.save(order);

      return {
        orderId: order.getId(),
        success: true
      };
    } catch (error) {
      return {
        orderId: '',
        success: false,
        error: error.message
      };
    }
  }
}

// ============================================
// Confirm Order Use Case
// ============================================

export interface ConfirmOrderRequest {
  orderId: string;
  paymentMethod: PaymentMethod;
}

export interface ConfirmOrderResponse {
  success: boolean;
  error?: string;
}

export class ConfirmOrderUseCase implements UseCase<ConfirmOrderRequest, ConfirmOrderResponse> {
  constructor(
    private orderRepository: OrderRepository,
    private paymentGateway: PaymentGateway,
    private inventoryService: InventoryService,
    private emailService: EmailService
  ) {}

  async execute(request: ConfirmOrderRequest): Promise<ConfirmOrderResponse> {
    try {
      // 1. Retrieve order
      const order = await this.orderRepository.findById(request.orderId);
      if (!order) {
        return { success: false, error: 'Order not found' };
      }

      // 2. Check inventory
      const inventoryCheck = await this.inventoryService.checkAvailability(order);
      if (!inventoryCheck.available) {
        return { success: false, error: 'Insufficient inventory' };
      }

      // 3. Process payment
      const totalAmount = order.getTotalAmount();
      const paymentResult = await this.paymentGateway.charge(
        totalAmount,
        request.paymentMethod
      );

      if (!paymentResult.success) {
        return { success: false, error: 'Payment failed' };
      }

      // 4. Confirm order (domain logic)
      order.confirm();

      // 5. Reserve inventory
      await this.inventoryService.reserve(order);

      // 6. Save order
      await this.orderRepository.save(order);

      // 7. Send confirmation email
      await this.emailService.sendOrderConfirmation(order);

      return { success: true };
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
}

// ============================================
// Query Use Case (CQRS Pattern)
// ============================================

export interface GetOrderDetailsRequest {
  orderId: string;
}

export interface OrderDetailsDTO {
  id: string;
  customerId: string;
  items: OrderItemDTO[];
  totalAmount: number;
  status: string;
  createdAt: Date;
}

export class GetOrderDetailsUseCase implements UseCase<GetOrderDetailsRequest, OrderDetailsDTO> {
  constructor(private orderRepository: OrderRepository) {}

  async execute(request: GetOrderDetailsRequest): Promise<OrderDetailsDTO> {
    const order = await this.orderRepository.findById(request.orderId);
    
    if (!order) {
      throw new OrderNotFoundError(`Order ${request.orderId} not found`);
    }

    // Convert entity to DTO
    return {
      id: order.getId(),
      customerId: order.getCustomerId(),
      items: order.getItems().map(item => ({
        productId: item.productId,
        quantity: item.quantity,
        unitPrice: item.unitPrice.amount,
        totalPrice: item.getTotalPrice().amount
      })),
      totalAmount: order.getTotalAmount().amount,
      status: order.getStatus(),
      createdAt: order.getCreatedAt()
    };
  }
}
```

### Layer 3: Interface Adapters

Adapters convert data between use cases and external systems:

```typescript
// ============================================
// Controllers (convert external requests to use cases)
// ============================================

// React Controller
export class OrderController {
  constructor(
    private createOrderUseCase: CreateOrderUseCase,
    private confirmOrderUseCase: ConfirmOrderUseCase,
    private getOrderDetailsUseCase: GetOrderDetailsUseCase
  ) {}

  async createOrder(formData: OrderFormData): Promise<void> {
    const request: CreateOrderRequest = {
      customerId: formData.customerId,
      items: formData.items.map(item => ({
        productId: item.productId,
        quantity: parseInt(item.quantity)
      }))
    };

    const response = await this.createOrderUseCase.execute(request);

    if (!response.success) {
      throw new Error(response.error);
    }

    return response.orderId;
  }

  async confirmOrder(orderId: string, paymentData: PaymentFormData): Promise<void> {
    const request: ConfirmOrderRequest = {
      orderId,
      paymentMethod: {
        type: paymentData.type,
        cardNumber: paymentData.cardNumber,
        expiryDate: paymentData.expiryDate
      }
    };

    const response = await this.confirmOrderUseCase.execute(request);

    if (!response.success) {
      throw new Error(response.error);
    }
  }

  async getOrderDetails(orderId: string): Promise<OrderDetailsDTO> {
    return await this.getOrderDetailsUseCase.execute({ orderId });
  }
}

// Angular Controller
@Injectable({ providedIn: 'root' })
export class OrderControllerService {
  constructor(
    private createOrderUseCase: CreateOrderUseCase,
    private confirmOrderUseCase: ConfirmOrderUseCase,
    private getOrderDetailsUseCase: GetOrderDetailsUseCase
  ) {}

  createOrder(formData: OrderFormData): Observable<string> {
    const request: CreateOrderRequest = {
      customerId: formData.customerId,
      items: formData.items
    };

    return from(this.createOrderUseCase.execute(request)).pipe(
      map(response => {
        if (!response.success) {
          throw new Error(response.error);
        }
        return response.orderId;
      })
    );
  }

  confirmOrder(orderId: string, paymentData: PaymentFormData): Observable<void> {
    const request: ConfirmOrderRequest = {
      orderId,
      paymentMethod: this.mapPaymentMethod(paymentData)
    };

    return from(this.confirmOrderUseCase.execute(request)).pipe(
      map(response => {
        if (!response.success) {
          throw new Error(response.error);
        }
      })
    );
  }

  getOrderDetails(orderId: string): Observable<OrderDetailsDTO> {
    return from(this.getOrderDetailsUseCase.execute({ orderId }));
  }

  private mapPaymentMethod(data: PaymentFormData): PaymentMethod {
    return {
      type: data.type,
      cardNumber: data.cardNumber,
      expiryDate: data.expiryDate
    };
  }
}

// ============================================
// Presenters (format data for UI)
// ============================================

export class OrderPresenter {
  present(orderDetails: OrderDetailsDTO): OrderViewModel {
    return {
      orderId: orderDetails.id,
      items: orderDetails.items.map(item => ({
        productId: item.productId,
        quantity: item.quantity,
        unitPrice: this.formatMoney(item.unitPrice),
        totalPrice: this.formatMoney(item.totalPrice)
      })),
      total: this.formatMoney(orderDetails.totalAmount),
      status: this.formatStatus(orderDetails.status),
      createdAt: this.formatDate(orderDetails.createdAt)
    };
  }

  private formatMoney(amount: number): string {
    return `$${amount.toFixed(2)}`;
  }

  private formatStatus(status: string): string {
    return status.charAt(0).toUpperCase() + status.slice(1).toLowerCase();
  }

  private formatDate(date: Date): string {
    return date.toLocaleDateString();
  }
}

// ============================================
// Gateways (adapt to external systems)
// ============================================

// Payment Gateway Interface (defined in use case layer)
export interface PaymentGateway {
  charge(amount: Money, method: PaymentMethod): Promise<PaymentResult>;
}

// Payment Gateway Implementation (infrastructure layer)
export class StripePaymentGateway implements PaymentGateway {
  constructor(private stripeClient: StripeClient) {}

  async charge(amount: Money, method: PaymentMethod): Promise<PaymentResult> {
    try {
      // Convert domain model to Stripe format
      const stripeCharge = await this.stripeClient.charges.create({
        amount: Math.round(amount.amount * 100), // Convert to cents
        currency: amount.currency.toLowerCase(),
        source: method.token,
        description: 'Order payment'
      });

      // Convert Stripe response to domain model
      return {
        success: stripeCharge.status === 'succeeded',
        transactionId: stripeCharge.id,
        message: stripeCharge.status
      };
    } catch (error) {
      return {
        success: false,
        transactionId: '',
        message: error.message
      };
    }
  }
}
```

### Layer 4: Frameworks & Drivers

The outermost layer contains frameworks, tools, and glue code:

```typescript
// ============================================
// React Application Setup
// ============================================

// Dependency Injection Container
class DIContainer {
  private instances = new Map<string, any>();

  register<T>(key: string, factory: () => T): void {
    this.instances.set(key, factory());
  }

  resolve<T>(key: string): T {
    const instance = this.instances.get(key);
    if (!instance) {
      throw new Error(`No instance registered for key: ${key}`);
    }
    return instance;
  }
}

// Container Setup
const container = new DIContainer();

// Register infrastructure
container.register('httpClient', () => new HttpClient());
container.register('database', () => new DatabaseClient());

// Register repositories
container.register('orderRepository', () => 
  new OrderRepositoryImpl(container.resolve('database'))
);
container.register('productRepository', () => 
  new ProductRepositoryImpl(container.resolve('database'))
);

// Register gateways
container.register('paymentGateway', () => 
  new StripePaymentGateway(container.resolve('stripeClient'))
);
container.register('emailService', () => 
  new EmailServiceImpl(container.resolve('httpClient'))
);
container.register('inventoryService', () => 
  new InventoryServiceImpl(container.resolve('httpClient'))
);

// Register use cases
container.register('createOrderUseCase', () => 
  new CreateOrderUseCase(
    container.resolve('orderRepository'),
    container.resolve('productRepository'),
    new OrderFactory()
  )
);
container.register('confirmOrderUseCase', () => 
  new ConfirmOrderUseCase(
    container.resolve('orderRepository'),
    container.resolve('paymentGateway'),
    container.resolve('inventoryService'),
    container.resolve('emailService')
  )
);

// Register controllers
container.register('orderController', () => 
  new OrderController(
    container.resolve('createOrderUseCase'),
    container.resolve('confirmOrderUseCase'),
    container.resolve('getOrderDetailsUseCase')
  )
);

// React Context Provider
const DIContext = createContext<DIContainer | null>(null);

export function DIProvider({ children }: { children: ReactNode }) {
  return (
    <DIContext.Provider value={container}>
      {children}
    </DIContext.Provider>
  );
}

export function useDI() {
  const context = useContext(DIContext);
  if (!context) {
    throw new Error('useDI must be used within DIProvider');
  }
  return context;
}

// React Component using Clean Architecture
function CreateOrderForm() {
  const di = useDI();
  const controller = di.resolve<OrderController>('orderController');
  const [items, setItems] = useState<OrderFormItem[]>([]);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setLoading(true);

    try {
      const orderId = await controller.createOrder({
        customerId: 'current-user-id',
        items
      });
      
      alert(`Order created: ${orderId}`);
    } catch (error) {
      alert(`Error: ${error.message}`);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create Order'}
      </button>
    </form>
  );
}

// ============================================
// Angular Application Setup
// ============================================

// Angular Module with DI
@NgModule({
  providers: [
    // Infrastructure
    { provide: 'HttpClient', useClass: HttpClient },
    { provide: 'Database', useClass: DatabaseClient },
    
    // Repositories
    {
      provide: OrderRepository,
      useFactory: (db: DatabaseClient) => new OrderRepositoryImpl(db),
      deps: ['Database']
    },
    {
      provide: ProductRepository,
      useFactory: (db: DatabaseClient) => new ProductRepositoryImpl(db),
      deps: ['Database']
    },
    
    // Gateways
    {
      provide: PaymentGateway,
      useFactory: (stripe: StripeClient) => new StripePaymentGateway(stripe),
      deps: [StripeClient]
    },
    
    // Use Cases
    {
      provide: CreateOrderUseCase,
      useFactory: (
        orderRepo: OrderRepository,
        productRepo: ProductRepository
      ) => new CreateOrderUseCase(orderRepo, productRepo, new OrderFactory()),
      deps: [OrderRepository, ProductRepository]
    },
    {
      provide: ConfirmOrderUseCase,
      useFactory: (
        orderRepo: OrderRepository,
        payment: PaymentGateway,
        inventory: InventoryService,
        email: EmailService
      ) => new ConfirmOrderUseCase(orderRepo, payment, inventory, email),
      deps: [OrderRepository, PaymentGateway, InventoryService, EmailService]
    },
    
    // Controllers
    OrderControllerService
  ]
})
export class OrderModule {}

// Angular Component
@Component({
  selector: 'app-create-order',
  template: `
    <form (ngSubmit)="handleSubmit()">
      <!-- Form fields -->
      <button type="submit" [disabled]="loading">
        {{ loading ? 'Creating...' : 'Create Order' }}
      </button>
    </form>
  `
})
export class CreateOrderComponent {
  items: OrderFormItem[] = [];
  loading = false;

  constructor(private controller: OrderControllerService) {}

  handleSubmit(): void {
    this.loading = true;

    this.controller.createOrder({
      customerId: 'current-user-id',
      items: this.items
    }).subscribe({
      next: (orderId) => {
        alert(`Order created: ${orderId}`);
        this.loading = false;
      },
      error: (error) => {
        alert(`Error: ${error.message}`);
        this.loading = false;
      }
    });
  }
}
```

## Implementation Examples

### Complete Feature with All Layers

```typescript
// ============================================
// LAYER 1: ENTITIES
// ============================================

class Product {
  constructor(
    private readonly id: string,
    private name: string,
    private price: Money,
    private stockQuantity: number
  ) {}

  getId(): string {
    return this.id;
  }

  getPrice(): Money {
    return this.price;
  }

  isInStock(quantity: number): boolean {
    return this.stockQuantity >= quantity;
  }

  reserveStock(quantity: number): void {
    if (!this.isInStock(quantity)) {
      throw new InsufficientStockError('Not enough stock available');
    }
    this.stockQuantity -= quantity;
  }
}

// ============================================
// LAYER 2: USE CASES
// ============================================

// Repository Interfaces (defined in use case layer)
interface ProductRepository {
  findById(id: string): Promise<Product | null>;
  save(product: Product): Promise<void>;
}

interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

// Use Case
class AddItemToOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository
  ) {}

  async execute(request: AddItemRequest): Promise<AddItemResponse> {
    // 1. Get order
    const order = await this.orderRepository.findById(request.orderId);
    if (!order) {
      return { success: false, error: 'Order not found' };
    }

    // 2. Get product
    const product = await this.productRepository.findById(request.productId);
    if (!product) {
      return { success: false, error: 'Product not found' };
    }

    // 3. Check stock
    if (!product.isInStock(request.quantity)) {
      return { success: false, error: 'Insufficient stock' };
    }

    // 4. Add item to order (domain logic)
    try {
      order.addItem(
        product.getId(),
        request.quantity,
        product.getPrice()
      );
    } catch (error) {
      return { success: false, error: error.message };
    }

    // 5. Save order
    await this.orderRepository.save(order);

    return { success: true };
  }
}

// ============================================
// LAYER 3: INTERFACE ADAPTERS
// ============================================

// HTTP Controller
class OrderHttpController {
  constructor(
    private addItemUseCase: AddItemToOrderUseCase,
    private presenter: OrderPresenter
  ) {}

  async addItem(req: HttpRequest, res: HttpResponse): Promise<void> {
    try {
      const request: AddItemRequest = {
        orderId: req.params.orderId,
        productId: req.body.productId,
        quantity: parseInt(req.body.quantity)
      };

      const response = await this.addItemUseCase.execute(request);

      if (response.success) {
        res.status(200).json({ message: 'Item added successfully' });
      } else {
        res.status(400).json({ error: response.error });
      }
    } catch (error) {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
}

// Repository Implementation
class OrderRepositoryImpl implements OrderRepository {
  constructor(private db: Database) {}

  async save(order: Order): Promise<void> {
    const data = {
      id: order.getId(),
      customerId: order.getCustomerId(),
      items: order.getItems().map(item => ({
        productId: item.productId,
        quantity: item.quantity,
        unitPrice: item.unitPrice.amount
      })),
      status: order.getStatus()
    };

    await this.db.orders.upsert(data);
  }

  async findById(id: string): Promise<Order | null> {
    const data = await this.db.orders.findOne({ id });
    if (!data) return null;

    const order = new Order(data.id, data.customerId);
    data.items.forEach((item: any) => {
      order.addItem(
        item.productId,
        item.quantity,
        new Money(item.unitPrice, 'USD')
      );
    });

    return order;
  }
}

// ============================================
// LAYER 4: FRAMEWORKS & DRIVERS
// ============================================

// Express.js Setup
const app = express();

// Setup DI
const db = new DatabaseClient();
const orderRepo = new OrderRepositoryImpl(db);
const productRepo = new ProductRepositoryImpl(db);
const addItemUseCase = new AddItemToOrderUseCase(orderRepo, productRepo);
const orderPresenter = new OrderPresenter();
const orderController = new OrderHttpController(addItemUseCase, orderPresenter);

// Route
app.post('/api/orders/:orderId/items', (req, res) => 
  orderController.addItem(req, res)
);

// React Component
function AddItemToOrderButton({ orderId, productId }: Props) {
  const di = useDI();
  const controller = di.resolve<OrderController>('orderController');

  const handleClick = async () => {
    try {
      await controller.addItemToOrder(orderId, productId, 1);
      alert('Item added!');
    } catch (error) {
      alert(`Error: ${error.message}`);
    }
  };

  return <button onClick={handleClick}>Add to Order</button>;
}
```

### Testing with Clean Architecture

```typescript
// Testing is easy because layers are independent

// ============================================
// Testing Entities (no dependencies)
// ============================================

describe('Order Entity', () => {
  it('should add item to order', () => {
    const order = new Order('order-1', 'customer-1');
    const price = new Money(100, 'USD');

    order.addItem('product-1', 2, price);

    expect(order.getItems()).toHaveLength(1);
    expect(order.getTotalAmount().amount).toBe(200);
  });

  it('should not allow modifying confirmed order', () => {
    const order = new Order('order-1', 'customer-1');
    order.addItem('product-1', 1, new Money(100, 'USD'));
    order.confirm();

    expect(() => {
      order.addItem('product-2', 1, new Money(50, 'USD'));
    }).toThrow(OrderNotModifiableError);
  });
});

// ============================================
// Testing Use Cases (mock dependencies)
// ============================================

describe('CreateOrderUseCase', () => {
  let useCase: CreateOrderUseCase;
  let mockOrderRepo: jest.Mocked<OrderRepository>;
  let mockProductRepo: jest.Mocked<ProductRepository>;
  let mockOrderFactory: jest.Mocked<OrderFactory>;

  beforeEach(() => {
    mockOrderRepo = {
      save: jest.fn(),
      findById: jest.fn()
    };
    mockProductRepo = {
      findById: jest.fn()
    };
    mockOrderFactory = {
      createOrder: jest.fn()
    };

    useCase = new CreateOrderUseCase(
      mockOrderRepo,
      mockProductRepo,
      mockOrderFactory
    );
  });

  it('should create order successfully', async () => {
    const mockOrder = new Order('order-1', 'customer-1');
    const mockProduct = new Product(
      'product-1',
      'Test Product',
      new Money(100, 'USD'),
      10
    );

    mockOrderFactory.createOrder.mockReturnValue(mockOrder);
    mockProductRepo.findById.mockResolvedValue(mockProduct);
    mockOrderRepo.save.mockResolvedValue(undefined);

    const request: CreateOrderRequest = {
      customerId: 'customer-1',
      items: [{ productId: 'product-1', quantity: 2 }]
    };

    const response = await useCase.execute(request);

    expect(response.success).toBe(true);
    expect(mockOrderRepo.save).toHaveBeenCalledWith(mockOrder);
  });

  it('should fail when product not found', async () => {
    const mockOrder = new Order('order-1', 'customer-1');
    mockOrderFactory.createOrder.mockReturnValue(mockOrder);
    mockProductRepo.findById.mockResolvedValue(null);

    const request: CreateOrderRequest = {
      customerId: 'customer-1',
      items: [{ productId: 'invalid-product', quantity: 1 }]
    };

    const response = await useCase.execute(request);

    expect(response.success).toBe(false);
    expect(response.error).toContain('not found');
    expect(mockOrderRepo.save).not.toHaveBeenCalled();
  });
});

// ============================================
// Testing Controllers (mock use cases)
// ============================================

describe('OrderController (React)', () => {
  let controller: OrderController;
  let mockCreateUseCase: jest.Mocked<CreateOrderUseCase>;

  beforeEach(() => {
    mockCreateUseCase = {
      execute: jest.fn()
    } as any;

    controller = new OrderController(
      mockCreateUseCase,
      null as any,
      null as any
    );
  });

  it('should create order successfully', async () => {
    mockCreateUseCase.execute.mockResolvedValue({
      orderId: 'order-1',
      success: true
    });

    const formData: OrderFormData = {
      customerId: 'customer-1',
      items: [{ productId: 'product-1', quantity: '2' }]
    };

    const orderId = await controller.createOrder(formData);

    expect(orderId).toBe('order-1');
    expect(mockCreateUseCase.execute).toHaveBeenCalledWith({
      customerId: 'customer-1',
      items: [{ productId: 'product-1', quantity: 2 }]
    });
  });
});
```

## Common Mistakes

### 1. Breaking the Dependency Rule

```typescript
// BAD: Inner layer depends on outer layer
class Order {
  async save(): Promise<void> {
    await fetch('/api/orders', { // Domain depends on HTTP!
      method: 'POST',
      body: JSON.stringify(this)
    });
  }
}

// GOOD: Outer layer depends on inner layer
interface OrderRepository {
  save(order: Order): Promise<void>;
}

class Order {
  // No infrastructure dependencies
}

class HttpOrderRepository implements OrderRepository {
  async save(order: Order): Promise<void> {
    await fetch('/api/orders', {
      method: 'POST',
      body: JSON.stringify(this.serialize(order))
    });
  }
}
```

### 2. Fat Use Cases

```typescript
// BAD: Use case contains business logic
class CreateOrderUseCase {
  async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    const order = new Order(request.customerId);
    
    // Business logic in use case!
    if (request.items.length === 0) {
      throw new Error('Cannot create empty order');
    }
    
    for (const item of request.items) {
      if (item.quantity <= 0) {
        throw new Error('Invalid quantity');
      }
      order.items.push(item);
    }
    
    await this.repository.save(order);
  }
}

// GOOD: Business logic in entity
class Order {
  private items: OrderItem[] = [];
  
  addItem(productId: string, quantity: number, price: Money): void {
    if (quantity <= 0) {
      throw new InvalidQuantityError('Quantity must be positive');
    }
    this.items.push(new OrderItem(productId, quantity, price));
  }
  
  confirm(): void {
    if (this.items.length === 0) {
      throw new EmptyOrderError('Cannot confirm empty order');
    }
    this.status = OrderStatus.Confirmed;
  }
}

class CreateOrderUseCase {
  async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    const order = this.factory.createOrder(request.customerId);
    
    for (const item of request.items) {
      const product = await this.productRepo.findById(item.productId);
      order.addItem(item.productId, item.quantity, product.getPrice());
    }
    
    await this.orderRepo.save(order);
  }
}
```

### 3. Passing Entities Across Boundaries

```typescript
// BAD: Exposing entities to UI
function OrderDisplay({ order }: { order: Order }) {
  return (
    <div>
      {order.getItems().map(item => /* ... */)}
    </div>
  );
}

// GOOD: Use DTOs/ViewModels
interface OrderViewModel {
  id: string;
  items: OrderItemViewModel[];
  total: string;
}

function OrderDisplay({ order }: { order: OrderViewModel }) {
  return (
    <div>
      {order.items.map(item => /* ... */)}
    </div>
  );
}

class OrderPresenter {
  present(order: Order): OrderViewModel {
    return {
      id: order.getId(),
      items: order.getItems().map(item => this.presentItem(item)),
      total: this.formatMoney(order.getTotalAmount())
    };
  }
}
```

## Best Practices

### 1. Define Interfaces in Inner Layers

```typescript
// Use case layer defines what it needs
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
}

interface PaymentGateway {
  charge(amount: Money, method: PaymentMethod): Promise<PaymentResult>;
}

// Infrastructure layer implements these interfaces
class StripePaymentGateway implements PaymentGateway {
  // Implementation
}
```

### 2. Use Dependency Injection

```typescript
// Constructor injection makes dependencies explicit
class ConfirmOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private paymentGateway: PaymentGateway,
    private emailService: EmailService
  ) {}
}

// Easy to test with mocks
const mockRepo = { save: jest.fn() };
const mockGateway = { charge: jest.fn() };
const mockEmail = { send: jest.fn() };
const useCase = new ConfirmOrderUseCase(mockRepo, mockGateway, mockEmail);
```

### 3. Keep Entities Framework-Agnostic

```typescript
// AVOID: Framework-specific decorators
class Order {
  @Column()
  id: string;
  
  @Observable()
  status: string;
}

// PREFER: Pure TypeScript classes
class Order {
  constructor(
    private readonly id: string,
    private status: OrderStatus
  ) {}
}
```

### 4. Use DTOs for Communication

```typescript
// Request DTO
interface CreateOrderRequest {
  customerId: string;
  items: OrderItemRequest[];
}

// Response DTO
interface CreateOrderResponse {
  orderId: string;
  success: boolean;
  error?: string;
}

// Use case works with DTOs
class CreateOrderUseCase {
  async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    // Map DTO to domain model
    const order = this.mapToDomain(request);
    
    // Domain logic
    await this.repository.save(order);
    
    // Map domain model to DTO
    return this.mapToResponse(order);
  }
}
```

## When to Use/Not to Use

### Use Clean Architecture When:

1. **Long-Term Projects**: System will be maintained for years
2. **Complex Business Logic**: Domain logic is sophisticated
3. **Multiple Interfaces**: Same logic used by web, mobile, API
4. **Testability is Critical**: Need high test coverage

```typescript
// Good fit: Complex e-commerce system
class Order {
  applyDiscount(rules: DiscountRules): void {
    // Complex business rules
  }
  
  calculateTax(jurisdiction: TaxJurisdiction): Money {
    // Complex tax calculation
  }
}
```

### Don't Use Clean Architecture When:

1. **Simple CRUD Applications**: Minimal business logic
2. **Prototypes/MVPs**: Speed over structure
3. **Small Team Without Experience**: Learning curve too steep
4. **Tight Coupling to Framework**: Framework IS the architecture (e.g., WordPress plugin)

```typescript
// Poor fit: Simple todo app
interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

// Just use simple services
@Injectable()
class TodoService {
  async create(title: string): Promise<Todo> {
    return this.http.post('/api/todos', { title });
  }
}
```

## Interview Questions

### Junior Level

**Q: What is Clean Architecture?**

A: Clean Architecture is a software design approach that separates concerns into layers, with business logic at the center and infrastructure at the edges. Dependencies point inward, making the system independent of frameworks, UI, and databases.

**Q: What is the Dependency Rule?**

A: The Dependency Rule states that source code dependencies must point only inward, toward higher-level policies. Inner layers should not know about outer layers.

### Mid Level

**Q: How does Clean Architecture differ from traditional layered architecture?**

A: Traditional layered architecture often has database at the bottom with dependencies pointing down. Clean Architecture inverts this: domain is at the center, and all dependencies point inward. Infrastructure depends on domain, not vice versa.

**Q: How do you handle cross-cutting concerns like logging in Clean Architecture?**

A: Use dependency injection to provide cross-cutting concerns to use cases. Define interfaces in the use case layer and implement them in the infrastructure layer.

```typescript
interface Logger {
  log(message: string): void;
  error(message: string, error: Error): void;
}

class CreateOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private logger: Logger
  ) {}

  async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    this.logger.log(`Creating order for customer ${request.customerId}`);
    // ...
  }
}
```

### Senior Level

**Q: How do you implement Clean Architecture with CQRS?**

A: Separate command use cases (write operations) from query use cases (read operations). Commands work with domain entities, queries return DTOs from optimized read models.

```typescript
// Command
class CreateOrderUseCase {
  async execute(command: CreateOrderCommand): Promise<void> {
    const order = this.factory.create(command);
    await this.writeRepo.save(order);
    await this.eventBus.publish(new OrderCreatedEvent(order.id));
  }
}

// Query
class GetOrderListUseCase {
  async execute(query: GetOrderListQuery): Promise<OrderListDTO[]> {
    return await this.readRepo.query(query);
  }
}
```

**Q: How do you handle transactions across multiple aggregates in Clean Architecture?**

A: Use domain events and eventual consistency, or implement a transaction management service at the use case level that coordinates multiple repository operations.

```typescript
class ConfirmOrderUseCase {
  async execute(request: ConfirmOrderRequest): Promise<void> {
    await this.transactionManager.executeInTransaction(async () => {
      const order = await this.orderRepo.findById(request.orderId);
      order.confirm();
      await this.orderRepo.save(order);
      
      await this.inventoryRepo.reserve(order.getItems());
      await this.paymentRepo.processPayment(order.getTotalAmount());
    });
  }
}
```

## Key Takeaways

1. Clean Architecture separates concerns into concentric layers
2. Dependencies always point inward (Dependency Rule)
3. Business logic is independent of frameworks, UI, and databases
4. Entities contain enterprise business rules
5. Use cases contain application business rules
6. Interface adapters convert data between layers
7. Frameworks and drivers are in the outermost layer
8. Use dependency injection and interfaces to maintain independence
9. DTOs/ViewModels cross boundaries, not entities
10. Architecture enables independent testability of each layer

## Resources

### Books
- "Clean Architecture" by Robert C. Martin
- "Get Your Hands Dirty on Clean Architecture" by Tom Hombergs
- "Implementing Domain-Driven Design" by Vaughn Vernon

### Articles
- Clean Architecture by Uncle Bob: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- The Clean Architecture: https://khalilstemmler.com/articles/software-design-architecture/organizing-app-logic/

### Examples
- Clean Architecture React: https://github.com/eduardomoroni/react-clean-architecture
- Clean Architecture TypeScript: https://github.com/thebrodmann/ts-clean-architecture
- Angular Clean Architecture: https://github.com/im-konge/ng-clean-architecture

### Tools
- NestJS: Framework with built-in Clean Architecture support
- InversifyJS: Dependency injection for TypeScript
- TypeDI: Dependency injection container
