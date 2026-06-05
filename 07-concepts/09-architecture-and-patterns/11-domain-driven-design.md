# Domain-Driven Design (DDD)

## The Idea

**In plain English:** Domain-Driven Design is a way of building software by first deeply understanding the real-world problem you are solving (the "domain") and then making your code mirror the language and structure of that world. A "domain" just means the subject area your app is about — like banking, shipping, or healthcare.

**Real-world analogy:** Think of a hospital. The doctors, nurses, and receptionists all have different jobs, but they share a common language — words like "patient," "prescription," and "ward" mean the same thing to everyone. Each department (Emergency, Radiology, Billing) handles its own version of a "patient" record with only the details it needs, and a head doctor (the attending physician) is the only one who can approve changes to a patient's treatment plan.

- The shared hospital vocabulary = the Ubiquitous Language (the common words developers and business people agree to use)
- Each department's separate patient record = a Bounded Context (an area of the codebase with its own model and rules)
- The attending physician who controls treatment decisions = the Aggregate Root (the one object that controls all changes to a group of related data)

---

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Strategic Design](#strategic-design)
4. [Tactical Design](#tactical-design)
5. [Implementation Examples](#implementation-examples)
6. [Common Mistakes](#common-mistakes)
7. [Best Practices](#best-practices)
8. [When to Use/Not to Use](#when-to-usenot-to-use)
9. [Interview Questions](#interview-questions)
10. [Key Takeaways](#key-takeaways)
11. [Resources](#resources)

## Introduction

Domain-Driven Design (DDD) is a software development approach that emphasizes collaboration between technical and domain experts to create a shared understanding of the problem space. It focuses on modeling the core business domain and its logic, using a common language (Ubiquitous Language) that bridges the gap between developers and domain experts.

DDD is particularly valuable in complex business domains where the business logic is intricate and constantly evolving. It provides patterns and principles for organizing code around business concepts rather than technical concerns.

## Core Concepts

### Ubiquitous Language

The foundation of DDD is creating a common language shared by all team members:

```typescript
// BAD: Technical jargon disconnected from business
class DataProcessor {
  executeOperation(data: any[]): void {
    // What does this do in business terms?
  }
}

// GOOD: Business-focused terminology
class OrderFulfillment {
  processOrderForShipment(order: Order): void {
    // Clear business intent
  }
}
```

### Domain Model

The domain model represents the core business concepts and their relationships:

```
┌─────────────────────────────────────────────────────────┐
│                    Domain Model                          │
│                                                          │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐       │
│  │ Entity   │────▶│ Value    │     │ Aggregate│       │
│  │          │     │ Object   │     │  Root    │       │
│  └──────────┘     └──────────┘     └──────────┘       │
│                                                          │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐       │
│  │ Service  │     │Repository│     │ Factory  │       │
│  │          │     │          │     │          │       │
│  └──────────┘     └──────────┘     └──────────┘       │
└─────────────────────────────────────────────────────────┘
```

## Strategic Design

### Bounded Contexts

Bounded Contexts define explicit boundaries around models, ensuring each context has its own consistent vocabulary:

```typescript
// E-commerce System with Multiple Bounded Contexts

// ============================================
// SALES CONTEXT
// ============================================
namespace SalesContext {
  // In Sales, Customer is about purchasing
  export class Customer {
    constructor(
      public readonly id: string,
      public readonly email: string,
      public readonly loyaltyPoints: number,
      public readonly preferredPaymentMethod: string
    ) {}

    canPlaceOrder(): boolean {
      return this.loyaltyPoints >= 0;
    }
  }

  export class Order {
    constructor(
      public readonly id: string,
      public readonly customerId: string,
      public readonly items: OrderItem[],
      private status: OrderStatus
    ) {}

    getTotalAmount(): number {
      return this.items.reduce((sum, item) => sum + item.getTotalPrice(), 0);
    }
  }
}

// ============================================
// SHIPPING CONTEXT
// ============================================
namespace ShippingContext {
  // In Shipping, Customer is about delivery
  export class Customer {
    constructor(
      public readonly id: string,
      public readonly shippingAddresses: Address[],
      public readonly preferredCarrier: string
    ) {}

    getDefaultShippingAddress(): Address | null {
      return this.shippingAddresses.find(addr => addr.isDefault) || null;
    }
  }

  export class Shipment {
    constructor(
      public readonly id: string,
      public readonly orderId: string,
      public readonly destination: Address,
      private trackingNumber?: string
    ) {}

    ship(carrier: string): void {
      this.trackingNumber = this.generateTrackingNumber(carrier);
    }

    private generateTrackingNumber(carrier: string): string {
      return `${carrier}-${Date.now()}`;
    }
  }
}

// ============================================
// SUPPORT CONTEXT
// ============================================
namespace SupportContext {
  // In Support, Customer is about service history
  export class Customer {
    constructor(
      public readonly id: string,
      public readonly email: string,
      public readonly tickets: SupportTicket[],
      public readonly satisfactionRating: number
    ) {}

    hasActiveTickets(): boolean {
      return this.tickets.some(ticket => ticket.isOpen);
    }
  }
}
```

### Context Mapping

Context mapping defines relationships between bounded contexts:

```typescript
// Context Map Patterns

// 1. SHARED KERNEL: Shared domain model between contexts
namespace SharedKernel {
  export class CustomerId {
    constructor(public readonly value: string) {
      if (!value || value.length < 3) {
        throw new Error('Invalid customer ID');
      }
    }

    equals(other: CustomerId): boolean {
      return this.value === other.value;
    }
  }

  export class Money {
    constructor(
      public readonly amount: number,
      public readonly currency: string
    ) {}

    add(other: Money): Money {
      if (this.currency !== other.currency) {
        throw new Error('Currency mismatch');
      }
      return new Money(this.amount + other.amount, this.currency);
    }
  }
}

// 2. ANTICORRUPTION LAYER: Translate between contexts
class OrderToShipmentAdapter {
  // Protects Shipping context from Sales context changes
  toShipmentRequest(order: SalesContext.Order): ShippingContext.ShipmentRequest {
    return {
      orderId: order.id,
      items: order.items.map(item => ({
        sku: item.productId,
        quantity: item.quantity,
        weight: this.getProductWeight(item.productId)
      })),
      priority: this.calculateShippingPriority(order)
    };
  }

  private calculateShippingPriority(order: SalesContext.Order): string {
    return order.getTotalAmount() > 1000 ? 'express' : 'standard';
  }

  private getProductWeight(productId: string): number {
    // Fetch from product catalog
    return 1.0;
  }
}

// 3. CUSTOMER-SUPPLIER: Downstream depends on upstream
interface SalesOrderService {
  // Upstream (Sales) defines the contract
  getOrder(orderId: string): Promise<SalesContext.Order>;
  getOrdersByCustomer(customerId: string): Promise<SalesContext.Order[]>;
}

class ShippingService {
  // Downstream (Shipping) consumes the contract
  constructor(private salesService: SalesOrderService) {}

  async createShipmentForOrder(orderId: string): Promise<void> {
    const order = await this.salesService.getOrder(orderId);
    // Create shipment based on order
  }
}
```

## Tactical Design

### Entities

Entities have a unique identity that persists over time:

```typescript
// TypeScript Entity Example
export class Order {
  private items: OrderItem[] = [];
  private status: OrderStatus = OrderStatus.Draft;
  private readonly createdAt: Date;
  private updatedAt: Date;

  constructor(
    private readonly id: string,
    private readonly customerId: string
  ) {
    this.createdAt = new Date();
    this.updatedAt = new Date();
  }

  // Identity
  getId(): string {
    return this.id;
  }

  equals(other: Order): boolean {
    return this.id === other.id;
  }

  // Business methods
  addItem(product: Product, quantity: number): void {
    if (this.status !== OrderStatus.Draft) {
      throw new Error('Cannot modify confirmed order');
    }

    const existingItem = this.items.find(
      item => item.productId === product.getId()
    );

    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.push(new OrderItem(product.getId(), quantity, product.getPrice()));
    }

    this.updatedAt = new Date();
  }

  confirm(): void {
    if (this.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    if (this.status !== OrderStatus.Draft) {
      throw new Error('Order already confirmed');
    }

    this.status = OrderStatus.Confirmed;
    this.updatedAt = new Date();
  }

  getTotalAmount(): number {
    return this.items.reduce((sum, item) => sum + item.getTotalPrice(), 0);
  }
}

// Angular Entity with RxJS
@Injectable()
export class OrderEntity {
  private readonly id: string;
  private readonly customerId: string;
  private itemsSubject = new BehaviorSubject<OrderItem[]>([]);
  private statusSubject = new BehaviorSubject<OrderStatus>(OrderStatus.Draft);

  items$ = this.itemsSubject.asObservable();
  status$ = this.statusSubject.asObservable();
  totalAmount$ = this.items$.pipe(
    map(items => items.reduce((sum, item) => sum + item.getTotalPrice(), 0))
  );

  constructor(id: string, customerId: string) {
    this.id = id;
    this.customerId = customerId;
  }

  addItem(product: Product, quantity: number): void {
    const currentItems = this.itemsSubject.value;
    const currentStatus = this.statusSubject.value;

    if (currentStatus !== OrderStatus.Draft) {
      throw new Error('Cannot modify confirmed order');
    }

    const existingItemIndex = currentItems.findIndex(
      item => item.productId === product.getId()
    );

    if (existingItemIndex >= 0) {
      currentItems[existingItemIndex].increaseQuantity(quantity);
    } else {
      currentItems.push(new OrderItem(product.getId(), quantity, product.getPrice()));
    }

    this.itemsSubject.next([...currentItems]);
  }

  confirm(): void {
    if (this.itemsSubject.value.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    this.statusSubject.next(OrderStatus.Confirmed);
  }
}
```

### Value Objects

Value Objects are immutable and defined by their attributes rather than identity:

```typescript
// Value Object Examples

export class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string
  ) {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    if (!currency || currency.length !== 3) {
      throw new Error('Invalid currency code');
    }
  }

  // Value objects are compared by value, not identity
  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  // Value objects are immutable - operations return new instances
  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('Cannot add money with different currencies');
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  isGreaterThan(other: Money): boolean {
    if (this.currency !== other.currency) {
      throw new Error('Cannot compare money with different currencies');
    }
    return this.amount > other.amount;
  }

  format(): string {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }
}

export class EmailAddress {
  private readonly value: string;

  constructor(email: string) {
    if (!this.isValid(email)) {
      throw new Error('Invalid email address');
    }
    this.value = email.toLowerCase();
  }

  private isValid(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }

  equals(other: EmailAddress): boolean {
    return this.value === other.value;
  }

  getValue(): string {
    return this.value;
  }

  getDomain(): string {
    return this.value.split('@')[1];
  }
}

export class Address {
  constructor(
    public readonly street: string,
    public readonly city: string,
    public readonly state: string,
    public readonly zipCode: string,
    public readonly country: string
  ) {
    this.validate();
  }

  private validate(): void {
    if (!this.street || !this.city || !this.state || !this.zipCode || !this.country) {
      throw new Error('All address fields are required');
    }
  }

  equals(other: Address): boolean {
    return (
      this.street === other.street &&
      this.city === other.city &&
      this.state === other.state &&
      this.zipCode === other.zipCode &&
      this.country === other.country
    );
  }

  format(): string {
    return `${this.street}, ${this.city}, ${this.state} ${this.zipCode}, ${this.country}`;
  }
}

// React Component using Value Objects
function PriceDisplay({ price }: { price: Money }) {
  return (
    <div className="price">
      <span className="currency">{price.currency}</span>
      <span className="amount">{price.amount.toFixed(2)}</span>
    </div>
  );
}

function CheckoutForm() {
  const [subtotal] = useState(new Money(100, 'USD'));
  const [tax] = useState(new Money(10, 'USD'));
  const total = subtotal.add(tax);

  return (
    <div>
      <div>Subtotal: {subtotal.format()}</div>
      <div>Tax: {tax.format()}</div>
      <div>Total: {total.format()}</div>
    </div>
  );
}
```

### Aggregates

Aggregates are clusters of domain objects treated as a single unit, with one entity serving as the aggregate root:

```typescript
// Aggregate Root: Order
export class Order {
  private readonly id: string;
  private readonly customerId: string;
  private items: OrderItem[] = [];
  private shippingAddress: Address | null = null;
  private status: OrderStatus = OrderStatus.Draft;
  private paymentInfo: PaymentInfo | null = null;

  constructor(id: string, customerId: string) {
    this.id = id;
    this.customerId = customerId;
  }

  // ============================================
  // AGGREGATE ROOT CONTROLS ALL MODIFICATIONS
  // ============================================

  addItem(productId: string, quantity: number, unitPrice: Money): void {
    this.ensureDraft();

    const existingItem = this.items.find(item => item.productId === productId);
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      const item = new OrderItem(productId, quantity, unitPrice);
      this.items.push(item);
    }
  }

  removeItem(productId: string): void {
    this.ensureDraft();
    this.items = this.items.filter(item => item.productId !== productId);
  }

  setShippingAddress(address: Address): void {
    this.ensureDraft();
    this.shippingAddress = address;
  }

  setPaymentInfo(paymentInfo: PaymentInfo): void {
    this.ensureDraft();
    this.paymentInfo = paymentInfo;
  }

  // Business invariants enforced by aggregate
  confirm(): void {
    if (this.status !== OrderStatus.Draft) {
      throw new Error('Order can only be confirmed from draft status');
    }
    if (this.items.length === 0) {
      throw new Error('Cannot confirm order without items');
    }
    if (!this.shippingAddress) {
      throw new Error('Shipping address is required');
    }
    if (!this.paymentInfo) {
      throw new Error('Payment information is required');
    }
    if (!this.hasValidInventory()) {
      throw new Error('Insufficient inventory for order items');
    }

    this.status = OrderStatus.Confirmed;
  }

  ship(trackingNumber: string): void {
    if (this.status !== OrderStatus.Confirmed) {
      throw new Error('Can only ship confirmed orders');
    }
    this.status = OrderStatus.Shipped;
  }

  cancel(): void {
    if (this.status === OrderStatus.Delivered) {
      throw new Error('Cannot cancel delivered orders');
    }
    this.status = OrderStatus.Cancelled;
  }

  // Aggregate exposes only necessary information
  getTotalAmount(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.getTotalPrice()),
      new Money(0, 'USD')
    );
  }

  getItems(): ReadonlyArray<OrderItem> {
    return [...this.items]; // Return copy to prevent external modification
  }

  private ensureDraft(): void {
    if (this.status !== OrderStatus.Draft) {
      throw new Error('Cannot modify order that is not in draft status');
    }
  }

  private hasValidInventory(): boolean {
    // Domain service would be called here
    return true;
  }
}

// OrderItem is part of the Order aggregate
class OrderItem {
  constructor(
    public readonly productId: string,
    private quantity: number,
    private readonly unitPrice: Money
  ) {
    if (quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
  }

  increaseQuantity(amount: number): void {
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    this.quantity += amount;
  }

  getTotalPrice(): Money {
    return this.unitPrice.multiply(this.quantity);
  }
}

// React Component for Aggregate
function OrderManagement({ orderId }: { orderId: string }) {
  const [order, setOrder] = useState<Order | null>(null);

  const addItem = (productId: string, quantity: number, price: Money) => {
    if (order) {
      order.addItem(productId, quantity, price);
      setOrder(new Order(order.getId(), order.getCustomerId()));
    }
  };

  const confirmOrder = () => {
    if (order) {
      try {
        order.confirm();
        setOrder(new Order(order.getId(), order.getCustomerId()));
      } catch (error) {
        alert(error.message);
      }
    }
  };

  return (
    <div>
      {order && (
        <>
          <OrderItemList items={order.getItems()} />
          <div>Total: {order.getTotalAmount().format()}</div>
          <button onClick={confirmOrder}>Confirm Order</button>
        </>
      )}
    </div>
  );
}
```

### Domain Services

Domain services contain business logic that doesn't naturally fit within an entity or value object:

```typescript
// Domain Service: Pricing Service
export class PricingService {
  calculateOrderTotal(order: Order, customer: Customer): Money {
    const subtotal = order.getTotalAmount();
    const discount = this.calculateDiscount(subtotal, customer);
    const tax = this.calculateTax(subtotal.add(discount.multiply(-1)));

    return subtotal.add(discount.multiply(-1)).add(tax);
  }

  private calculateDiscount(subtotal: Money, customer: Customer): Money {
    if (customer.isVIP()) {
      return subtotal.multiply(0.1); // 10% discount
    }
    if (customer.getLoyaltyPoints() > 1000) {
      return subtotal.multiply(0.05); // 5% discount
    }
    return new Money(0, subtotal.currency);
  }

  private calculateTax(amount: Money): Money {
    return amount.multiply(0.08); // 8% tax
  }
}

// Domain Service: Inventory Service
export class InventoryService {
  constructor(private inventoryRepository: InventoryRepository) {}

  async reserveItems(order: Order): Promise<ReservationResult> {
    const items = order.getItems();
    const reservations: ItemReservation[] = [];

    for (const item of items) {
      const available = await this.inventoryRepository.getAvailableQuantity(
        item.productId
      );

      if (available < item.quantity) {
        // Rollback previous reservations
        await this.releaseReservations(reservations);
        return ReservationResult.insufficient(item.productId);
      }

      const reservation = await this.inventoryRepository.reserve(
        item.productId,
        item.quantity
      );
      reservations.push(reservation);
    }

    return ReservationResult.success(reservations);
  }

  private async releaseReservations(
    reservations: ItemReservation[]
  ): Promise<void> {
    for (const reservation of reservations) {
      await this.inventoryRepository.release(reservation);
    }
  }
}

// Angular Service as Domain Service
@Injectable({ providedIn: 'root' })
export class OrderFulfillmentService {
  constructor(
    private pricingService: PricingService,
    private inventoryService: InventoryService,
    private paymentService: PaymentService,
    private shippingService: ShippingService
  ) {}

  async processOrder(order: Order, customer: Customer): Promise<OrderResult> {
    // 1. Calculate final price
    const totalAmount = this.pricingService.calculateOrderTotal(order, customer);

    // 2. Reserve inventory
    const reservationResult = await this.inventoryService.reserveItems(order);
    if (!reservationResult.isSuccess()) {
      return OrderResult.failure('Insufficient inventory');
    }

    // 3. Process payment
    try {
      const paymentResult = await this.paymentService.charge(
        customer,
        totalAmount
      );

      if (!paymentResult.isSuccess()) {
        await this.inventoryService.releaseReservations(
          reservationResult.reservations
        );
        return OrderResult.failure('Payment failed');
      }
    } catch (error) {
      await this.inventoryService.releaseReservations(
        reservationResult.reservations
      );
      throw error;
    }

    // 4. Confirm order
    order.confirm();

    // 5. Schedule shipment
    await this.shippingService.scheduleShipment(order);

    return OrderResult.success(order);
  }
}
```

### Repositories

Repositories provide collection-like interfaces for accessing aggregates:

```typescript
// Repository Interface (domain layer)
export interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  delete(id: string): Promise<void>;
}

// Repository Implementation (infrastructure layer)
export class OrderRepositoryImpl implements OrderRepository {
  constructor(private db: Database) {}

  async save(order: Order): Promise<void> {
    const data = this.toDatabase(order);
    await this.db.orders.upsert(data);
  }

  async findById(id: string): Promise<Order | null> {
    const data = await this.db.orders.findOne({ id });
    return data ? this.toDomain(data) : null;
  }

  async findByCustomerId(customerId: string): Promise<Order[]> {
    const dataList = await this.db.orders.find({ customerId });
    return dataList.map(data => this.toDomain(data));
  }

  async delete(id: string): Promise<void> {
    await this.db.orders.delete({ id });
  }

  private toDatabase(order: Order): any {
    return {
      id: order.getId(),
      customerId: order.getCustomerId(),
      items: order.getItems().map(item => ({
        productId: item.productId,
        quantity: item.quantity,
        unitPrice: {
          amount: item.unitPrice.amount,
          currency: item.unitPrice.currency
        }
      })),
      status: order.getStatus(),
      createdAt: order.getCreatedAt(),
      updatedAt: order.getUpdatedAt()
    };
  }

  private toDomain(data: any): Order {
    const order = new Order(data.id, data.customerId);
    data.items.forEach((item: any) => {
      order.addItem(
        item.productId,
        item.quantity,
        new Money(item.unitPrice.amount, item.unitPrice.currency)
      );
    });
    return order;
  }
}

// React Hook for Repository
function useOrderRepository() {
  const db = useDatabase();

  const repository = useMemo(
    () => new OrderRepositoryImpl(db),
    [db]
  );

  const findById = useCallback(
    async (id: string) => {
      return await repository.findById(id);
    },
    [repository]
  );

  const save = useCallback(
    async (order: Order) => {
      await repository.save(order);
    },
    [repository]
  );

  return { findById, save };
}

// Angular Repository Service
@Injectable({ providedIn: 'root' })
export class OrderRepositoryService implements OrderRepository {
  constructor(private http: HttpClient) {}

  async save(order: Order): Promise<void> {
    await firstValueFrom(
      this.http.post('/api/orders', this.serialize(order))
    );
  }

  async findById(id: string): Promise<Order | null> {
    const data = await firstValueFrom(
      this.http.get<any>(`/api/orders/${id}`)
    );
    return this.deserialize(data);
  }

  async findByCustomerId(customerId: string): Promise<Order[]> {
    const dataList = await firstValueFrom(
      this.http.get<any[]>(`/api/customers/${customerId}/orders`)
    );
    return dataList.map(data => this.deserialize(data));
  }

  async delete(id: string): Promise<void> {
    await firstValueFrom(
      this.http.delete(`/api/orders/${id}`)
    );
  }

  private serialize(order: Order): any {
    // Convert domain model to API format
    return {
      id: order.getId(),
      customerId: order.getCustomerId(),
      items: order.getItems()
    };
  }

  private deserialize(data: any): Order {
    // Convert API format to domain model
    const order = new Order(data.id, data.customerId);
    // Reconstruct order from data
    return order;
  }
}
```

### Factories

Factories encapsulate complex object creation logic:

```typescript
// Factory for creating Orders
export class OrderFactory {
  createNewOrder(customerId: string): Order {
    const orderId = this.generateOrderId();
    return new Order(orderId, customerId);
  }

  createFromData(data: OrderData): Order {
    const order = new Order(data.id, data.customerId);

    data.items.forEach(itemData => {
      const price = new Money(itemData.unitPrice.amount, itemData.unitPrice.currency);
      order.addItem(itemData.productId, itemData.quantity, price);
    });

    if (data.shippingAddress) {
      const address = new Address(
        data.shippingAddress.street,
        data.shippingAddress.city,
        data.shippingAddress.state,
        data.shippingAddress.zipCode,
        data.shippingAddress.country
      );
      order.setShippingAddress(address);
    }

    return order;
  }

  private generateOrderId(): string {
    return `ORD-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Complex Factory with Dependencies
export class CustomerFactory {
  constructor(
    private addressValidator: AddressValidator,
    private emailValidator: EmailValidator
  ) {}

  async createCustomer(data: CustomerRegistrationData): Promise<Customer> {
    // Validate email
    const email = new EmailAddress(data.email);
    const isEmailUnique = await this.emailValidator.isUnique(email);
    if (!isEmailUnique) {
      throw new Error('Email already in use');
    }

    // Validate and create address
    const addressValid = await this.addressValidator.validate(data.address);
    if (!addressValid) {
      throw new Error('Invalid address');
    }

    const address = new Address(
      data.address.street,
      data.address.city,
      data.address.state,
      data.address.zipCode,
      data.address.country
    );

    // Create customer with validated data
    const customerId = this.generateCustomerId();
    const customer = new Customer(
      customerId,
      email,
      data.firstName,
      data.lastName,
      address
    );

    return customer;
  }

  private generateCustomerId(): string {
    return `CUST-${Date.now()}`;
  }
}

// React Hook Factory
function useOrderFactory() {
  const factory = useMemo(() => new OrderFactory(), []);

  const createNewOrder = useCallback(
    (customerId: string) => {
      return factory.createNewOrder(customerId);
    },
    [factory]
  );

  return { createNewOrder };
}
```

## Implementation Examples

### Complete DDD Example in React

```typescript
// ============================================
// DOMAIN LAYER
// ============================================

// Value Objects
class ProductId {
  constructor(public readonly value: string) {}
  equals(other: ProductId): boolean {
    return this.value === other.value;
  }
}

class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string
  ) {}

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('Currency mismatch');
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }
}

// Entity
class CartItem {
  constructor(
    public readonly productId: ProductId,
    private quantity: number,
    private readonly unitPrice: Money
  ) {}

  increaseQuantity(amount: number): void {
    this.quantity += amount;
  }

  getTotalPrice(): Money {
    return this.unitPrice.multiply(this.quantity);
  }
}

// Aggregate Root
class ShoppingCart {
  private items: Map<string, CartItem> = new Map();

  constructor(private readonly customerId: string) {}

  addItem(productId: ProductId, quantity: number, unitPrice: Money): void {
    const key = productId.value;
    const existingItem = this.items.get(key);

    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.set(key, new CartItem(productId, quantity, unitPrice));
    }
  }

  removeItem(productId: ProductId): void {
    this.items.delete(productId.value);
  }

  getTotal(): Money {
    let total = new Money(0, 'USD');
    for (const item of this.items.values()) {
      total = total.add(item.getTotalPrice());
    }
    return total;
  }

  getItems(): CartItem[] {
    return Array.from(this.items.values());
  }

  clear(): void {
    this.items.clear();
  }
}

// Domain Service
class CheckoutService {
  constructor(
    private inventoryService: InventoryService,
    private paymentService: PaymentService
  ) {}

  async checkout(cart: ShoppingCart, paymentMethod: PaymentMethod): Promise<Order> {
    // Verify inventory
    for (const item of cart.getItems()) {
      const available = await this.inventoryService.checkAvailability(item.productId);
      if (!available) {
        throw new Error(`Product ${item.productId.value} not available`);
      }
    }

    // Process payment
    const total = cart.getTotal();
    const paymentResult = await this.paymentService.charge(paymentMethod, total);

    if (!paymentResult.isSuccess()) {
      throw new Error('Payment failed');
    }

    // Create order (would use OrderFactory)
    const order = this.createOrder(cart, paymentResult);
    
    return order;
  }

  private createOrder(cart: ShoppingCart, paymentResult: PaymentResult): Order {
    // Implementation
    return new Order();
  }
}

// ============================================
// APPLICATION LAYER (React)
// ============================================

// Context for Domain
const ShoppingCartContext = createContext<{
  cart: ShoppingCart;
  addItem: (productId: string, quantity: number, price: number) => void;
  removeItem: (productId: string) => void;
  checkout: () => Promise<void>;
} | null>(null);

function ShoppingCartProvider({ children }: { children: ReactNode }) {
  const [cart] = useState(() => new ShoppingCart('customer-123'));
  const checkoutService = useMemo(
    () => new CheckoutService(new InventoryService(), new PaymentService()),
    []
  );

  const addItem = useCallback(
    (productId: string, quantity: number, price: number) => {
      cart.addItem(
        new ProductId(productId),
        quantity,
        new Money(price, 'USD')
      );
    },
    [cart]
  );

  const removeItem = useCallback(
    (productId: string) => {
      cart.removeItem(new ProductId(productId));
    },
    [cart]
  );

  const checkout = useCallback(async () => {
    try {
      const paymentMethod = new PaymentMethod(); // Get from form
      await checkoutService.checkout(cart, paymentMethod);
      cart.clear();
    } catch (error) {
      console.error('Checkout failed:', error);
    }
  }, [cart, checkoutService]);

  return (
    <ShoppingCartContext.Provider value={{ cart, addItem, removeItem, checkout }}>
      {children}
    </ShoppingCartContext.Provider>
  );
}

function CartDisplay() {
  const context = useContext(ShoppingCartContext);
  if (!context) throw new Error('Must be used within ShoppingCartProvider');

  const { cart, removeItem, checkout } = context;
  const items = cart.getItems();
  const total = cart.getTotal();

  return (
    <div>
      {items.map(item => (
        <div key={item.productId.value}>
          <span>{item.productId.value}</span>
          <span>{item.getTotalPrice().amount}</span>
          <button onClick={() => removeItem(item.productId.value)}>Remove</button>
        </div>
      ))}
      <div>Total: {total.amount}</div>
      <button onClick={checkout}>Checkout</button>
    </div>
  );
}
```

### Complete DDD Example in Angular

```typescript
// ============================================
// DOMAIN LAYER
// ============================================

// (Same value objects, entities, aggregates as above)

// ============================================
// APPLICATION LAYER (Angular)
// ============================================

// Domain State Service
@Injectable({ providedIn: 'root' })
export class ShoppingCartService {
  private cart: ShoppingCart;
  private cartSubject = new BehaviorSubject<ShoppingCart | null>(null);

  cart$ = this.cartSubject.asObservable();
  items$ = this.cart$.pipe(
    filter(cart => cart !== null),
    map(cart => cart!.getItems())
  );
  total$ = this.cart$.pipe(
    filter(cart => cart !== null),
    map(cart => cart!.getTotal())
  );

  constructor() {
    this.cart = new ShoppingCart('customer-123');
    this.cartSubject.next(this.cart);
  }

  addItem(productId: string, quantity: number, price: number): void {
    this.cart.addItem(
      new ProductId(productId),
      quantity,
      new Money(price, 'USD')
    );
    this.cartSubject.next(this.cart);
  }

  removeItem(productId: string): void {
    this.cart.removeItem(new ProductId(productId));
    this.cartSubject.next(this.cart);
  }

  clear(): void {
    this.cart.clear();
    this.cartSubject.next(this.cart);
  }
}

// Domain Service
@Injectable({ providedIn: 'root' })
export class CheckoutService {
  constructor(
    private inventoryService: InventoryService,
    private paymentService: PaymentService,
    private cartService: ShoppingCartService
  ) {}

  async checkout(paymentMethod: PaymentMethod): Promise<Order> {
    const cart = await firstValueFrom(this.cartService.cart$);
    if (!cart) throw new Error('No cart');

    // Verify inventory
    for (const item of cart.getItems()) {
      const available = await this.inventoryService.checkAvailability(item.productId);
      if (!available) {
        throw new Error(`Product ${item.productId.value} not available`);
      }
    }

    // Process payment
    const total = cart.getTotal();
    const paymentResult = await this.paymentService.charge(paymentMethod, total);

    if (!paymentResult.isSuccess()) {
      throw new Error('Payment failed');
    }

    const order = this.createOrder(cart, paymentResult);
    this.cartService.clear();

    return order;
  }

  private createOrder(cart: ShoppingCart, paymentResult: PaymentResult): Order {
    return new Order();
  }
}

// Component
@Component({
  selector: 'app-shopping-cart',
  template: `
    <div class="cart">
      <div *ngFor="let item of items$ | async" class="cart-item">
        <span>{{ item.productId.value }}</span>
        <span>{{ item.getTotalPrice().amount | currency }}</span>
        <button (click)="removeItem(item.productId.value)">Remove</button>
      </div>
      <div class="cart-total">
        Total: {{ (total$ | async)?.amount | currency }}
      </div>
      <button (click)="checkout()">Checkout</button>
    </div>
  `
})
export class ShoppingCartComponent {
  items$ = this.cartService.items$;
  total$ = this.cartService.total$;

  constructor(
    private cartService: ShoppingCartService,
    private checkoutService: CheckoutService
  ) {}

  removeItem(productId: string): void {
    this.cartService.removeItem(productId);
  }

  async checkout(): Promise<void> {
    try {
      const paymentMethod = new PaymentMethod(); // Get from form
      await this.checkoutService.checkout(paymentMethod);
      alert('Order placed successfully!');
    } catch (error) {
      alert(`Checkout failed: ${error.message}`);
    }
  }
}
```

## Common Mistakes

### 1. Anemic Domain Model

```typescript
// BAD: Anemic model (just data holders)
class Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  status: string;
  total: number;
}

class OrderService {
  calculateTotal(order: Order): number {
    return order.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  validateOrder(order: Order): boolean {
    return order.items.length > 0 && order.total > 0;
  }

  confirmOrder(order: Order): void {
    if (this.validateOrder(order)) {
      order.status = 'confirmed';
    }
  }
}

// GOOD: Rich domain model
class Order {
  private items: OrderItem[] = [];
  private status: OrderStatus = OrderStatus.Draft;

  addItem(productId: string, quantity: number, unitPrice: Money): void {
    // Business logic in the domain
  }

  getTotalAmount(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.getTotalPrice()),
      new Money(0, 'USD')
    );
  }

  confirm(): void {
    if (this.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    this.status = OrderStatus.Confirmed;
  }
}
```

### 2. Leaky Abstractions

```typescript
// BAD: Database concerns leak into domain
class Order {
  @Column()
  id: string;

  @ManyToOne(() => Customer)
  customer: Customer;

  save(): Promise<void> {
    return database.orders.save(this);
  }
}

// GOOD: Pure domain model
class Order {
  constructor(
    private readonly id: string,
    private readonly customerId: string
  ) {}

  // Only business logic, no infrastructure concerns
  confirm(): void {
    this.status = OrderStatus.Confirmed;
  }
}

// Infrastructure in separate layer
class OrderRepository {
  async save(order: Order): Promise<void> {
    await this.db.orders.save(this.toDatabase(order));
  }
}
```

### 3. Ignoring Aggregate Boundaries

```typescript
// BAD: Directly modifying entities within aggregate
const order = await orderRepository.findById(orderId);
const item = order.items[0];
item.quantity = 5; // Bypassing aggregate root
await orderItemRepository.save(item);

// GOOD: All modifications through aggregate root
const order = await orderRepository.findById(orderId);
order.updateItemQuantity(item.productId, 5);
await orderRepository.save(order);
```

### 4. Not Using Value Objects

```typescript
// BAD: Primitive obsession
class Order {
  customerEmail: string;
  totalAmount: number;
  currency: string;
}

function sendConfirmation(email: string) {
  // No validation, easy to pass invalid email
}

// GOOD: Value objects with validation
class Order {
  constructor(
    private customerEmail: EmailAddress,
    private total: Money
  ) {}
}

class EmailAddress {
  constructor(private value: string) {
    if (!this.isValid(value)) {
      throw new Error('Invalid email');
    }
  }
}
```

## Best Practices

### 1. Start with Ubiquitous Language

```typescript
// Work with domain experts to establish terminology
// Document terms in a glossary

/**
 * UBIQUITOUS LANGUAGE GLOSSARY
 * 
 * Order: A customer's request to purchase products
 * Order Item: A line item within an order representing a product and quantity
 * Fulfillment: The process of preparing and shipping an order
 * Backorder: An order for a product currently out of stock
 * Drop Shipment: Order fulfilled directly by supplier
 */

// Use these exact terms in code
class Order {
  fulfill(): void { }
  backorder(): void { }
}

class Fulfillment {
  prepareForShipment(): void { }
}
```

### 2. Keep Aggregates Small

```typescript
// AVOID: Large aggregate with too many responsibilities
class Order {
  customer: Customer; // Full customer aggregate
  items: OrderItem[];
  payments: Payment[];
  shipments: Shipment[];
  reviews: Review[];
}

// PREFER: Smaller aggregate with references
class Order {
  customerId: string; // Reference, not full aggregate
  items: OrderItem[];
  
  // Other aggregates referenced by ID
  getShipments(): Promise<Shipment[]> {
    // Query separate aggregate
  }
}
```

### 3. Use Domain Events

```typescript
// Domain events for cross-aggregate communication
class OrderConfirmedEvent {
  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly totalAmount: Money,
    public readonly occurredAt: Date
  ) {}
}

class Order {
  private events: DomainEvent[] = [];

  confirm(): void {
    this.status = OrderStatus.Confirmed;
    this.events.push(
      new OrderConfirmedEvent(
        this.id,
        this.customerId,
        this.getTotalAmount(),
        new Date()
      )
    );
  }

  getEvents(): DomainEvent[] {
    return [...this.events];
  }

  clearEvents(): void {
    this.events = [];
  }
}

// Event handler in separate bounded context
@Injectable()
export class InventoryEventHandler {
  @OnEvent('order.confirmed')
  async handleOrderConfirmed(event: OrderConfirmedEvent): Promise<void> {
    // Reserve inventory in Inventory context
    await this.inventoryService.reserve(event.orderId);
  }
}
```

### 4. Separate Read and Write Models

```typescript
// Write model (domain aggregate)
class Order {
  // Rich behavior, enforces invariants
  addItem(productId: string, quantity: number, price: Money): void { }
  confirm(): void { }
}

// Read model (query/view model)
interface OrderListItem {
  orderId: string;
  customerName: string;
  totalAmount: number;
  status: string;
  createdAt: Date;
}

interface OrderDetails {
  order: OrderListItem;
  items: OrderItemView[];
  shippingAddress: AddressView;
}

// Separate query service
@Injectable()
export class OrderQueryService {
  async getOrderList(customerId: string): Promise<OrderListItem[]> {
    // Optimized read query
    return this.db.query(`
      SELECT o.id, c.name, o.total, o.status, o.created_at
      FROM orders o
      JOIN customers c ON o.customer_id = c.id
      WHERE o.customer_id = ?
    `, [customerId]);
  }
}
```

## When to Use/Not to Use

### Use DDD When:

1. **Complex Business Logic**: The domain has intricate business rules and workflows
2. **Long-Term Projects**: The system will be maintained and evolved over years
3. **Domain Expert Collaboration**: You have access to domain experts
4. **Core Business Differentiator**: The software is a competitive advantage

```typescript
// Good fit: Complex e-commerce with business rules
class Order {
  applyPromotion(code: string): void {
    // Complex promotion logic
  }

  calculateDynamicPricing(customer: Customer): Money {
    // Complex pricing based on customer tier, volume, time, etc.
  }
}
```

### Don't Use DDD When:

1. **Simple CRUD Applications**: Basic data entry with minimal logic
2. **Tight Deadlines**: Learning curve is too steep for project timeline
3. **Small Team Without Buy-In**: Team doesn't understand or support DDD
4. **No Domain Complexity**: The domain is straightforward

```typescript
// Poor fit: Simple CRUD
interface User {
  id: string;
  name: string;
  email: string;
}

// Just use simple services
@Injectable()
export class UserService {
  async create(user: User): Promise<User> {
    return this.db.users.create(user);
  }
}
```

## Interview Questions

### Junior Level

**Q: What is a Value Object and how does it differ from an Entity?**

A: A Value Object is defined by its attributes and has no unique identity, while an Entity has a unique identity that persists over time. Value Objects are immutable and compared by value.

```typescript
// Value Object: compared by value
class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: string
  ) {}

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }
}

// Entity: compared by identity
class Order {
  constructor(private readonly id: string) {}

  equals(other: Order): boolean {
    return this.id === other.id; // Identity comparison
  }
}
```

**Q: What is an Aggregate in DDD?**

A: An Aggregate is a cluster of domain objects (entities and value objects) treated as a single unit, with one entity serving as the Aggregate Root. All external access to the aggregate goes through the root, which enforces business invariants.

### Mid Level

**Q: Explain Bounded Contexts and why they're important.**

A: Bounded Contexts are explicit boundaries around models where specific terms and concepts have precise meaning. They prevent ambiguity and allow different parts of the system to use the same terms differently based on their context.

**Q: How do you handle cross-aggregate consistency?**

A: Use eventual consistency with domain events. Aggregates should be designed to maintain consistency boundaries within themselves. For cross-aggregate operations, use events to propagate changes asynchronously.

```typescript
class Order {
  confirm(): void {
    this.status = OrderStatus.Confirmed;
    this.addEvent(new OrderConfirmedEvent(this.id));
  }
}

// Event handler updates other aggregate eventually
@OnEvent('order.confirmed')
async handleOrderConfirmed(event: OrderConfirmedEvent) {
  await this.inventoryService.reserveItems(event.orderId);
}
```

### Senior Level

**Q: How do you implement CQRS with DDD and what are the tradeoffs?**

A: CQRS (Command Query Responsibility Segregation) separates write models (commands that change state) from read models (queries). This allows optimization of each side independently but adds complexity in synchronization and consistency.

```typescript
// Command side: domain model
class CreateOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly items: OrderItemData[]
  ) {}
}

class OrderCommandHandler {
  async handle(command: CreateOrderCommand): Promise<void> {
    const order = this.orderFactory.create(command);
    await this.orderRepository.save(order);
    await this.eventBus.publish(new OrderCreatedEvent(order.id));
  }
}

// Query side: optimized read model
interface OrderListQuery {
  customerId?: string;
  status?: string;
  limit: number;
}

class OrderQueryHandler {
  async handle(query: OrderListQuery): Promise<OrderView[]> {
    // Optimized query against denormalized read database
    return this.readDatabase.orders.find(query);
  }
}
```

**Q: How do you handle integration with external systems in DDD?**

A: Use an Anticorruption Layer (ACL) to translate between your domain model and external systems. This protects your domain from external changes and maintains your ubiquitous language.

```typescript
// External system API
interface ExternalPaymentAPI {
  processPayment(data: any): Promise<any>;
}

// Anticorruption Layer
class PaymentGatewayAdapter {
  constructor(private externalAPI: ExternalPaymentAPI) {}

  async processPayment(payment: Payment): Promise<PaymentResult> {
    // Translate domain model to external format
    const externalRequest = {
      amount: payment.amount.amount * 100, // Convert to cents
      currency: payment.amount.currency.toLowerCase(),
      customer_id: payment.customerId
    };

    const externalResponse = await this.externalAPI.processPayment(externalRequest);

    // Translate external response to domain model
    return new PaymentResult(
      externalResponse.success,
      externalResponse.transaction_id
    );
  }
}
```

## Key Takeaways

1. DDD is about collaboration and shared understanding, not just code patterns
2. Ubiquitous Language bridges the gap between technical and domain experts
3. Bounded Contexts provide clear boundaries where terms have specific meanings
4. Entities have identity; Value Objects are defined by their attributes
5. Aggregates enforce consistency boundaries and business invariants
6. Domain Services contain logic that doesn't fit in entities or value objects
7. Repositories provide collection-like interfaces for aggregates
8. Keep aggregates small and use domain events for cross-aggregate communication
9. Strategic design (contexts, relationships) is as important as tactical patterns
10. DDD adds complexity; use it when domain complexity justifies the investment

## Resources

### Official Documentation
- Eric Evans: Domain-Driven Design (The Blue Book)
- Vaughn Vernon: Implementing Domain-Driven Design (The Red Book)
- Martin Fowler: Domain-Driven Design https://martinfowler.com/tags/domain%20driven%20design.html

### Books
- "Domain-Driven Design Distilled" by Vaughn Vernon
- "Patterns, Principles, and Practices of Domain-Driven Design" by Scott Millett

### Online Resources
- DDD Community: https://www.dddcommunity.org/
- Context Mapping: https://github.com/ddd-crew/context-mapping
- Aggregate Design Canvas: https://github.com/ddd-crew/aggregate-design-canvas

### Tools
- EventStorming: Workshop technique for domain exploration
- Domain Storytelling: Visual technique for requirements
- Context Mapper: DSL for modeling strategic DDD patterns

### Videos
- "Domain-Driven Design: The Good Parts" by Jimmy Bogard
- "Implementing DDD" series by Vaughn Vernon
- "Strategic Domain-Driven Design" by Nick Tune
