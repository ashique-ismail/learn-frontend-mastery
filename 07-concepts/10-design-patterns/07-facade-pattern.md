# Facade Pattern

## The Idea

**In plain English:** A facade is a simplified control panel that hides all the complicated wiring behind the scenes, so you can get things done without knowing how every part works. In code, a "facade" is a class or function that bundles up many complex steps into one easy-to-use command.

**Real-world analogy:** Think of a hotel concierge desk. When you ask them to "arrange a night out," they quietly book a restaurant, call a taxi, and reserve theatre tickets — all without you contacting each service yourself.

- The concierge desk = the Facade class (your single point of contact)
- The restaurant, taxi company, and theatre = the subsystem classes (the complex parts doing the real work)
- You, the hotel guest = the client code (which only needs to speak to the facade)

---

## Overview

The Facade pattern is a structural design pattern that provides a simplified interface to a complex subsystem. It defines a higher-level interface that makes the subsystem easier to use by hiding its complexity and providing a clean, unified interface. The facade pattern is one of the most commonly used patterns in modern application development.

## Core Concepts

### Facade Pattern Structure

```
┌──────────────┐
│    Client    │
└──────┬───────┘
       │ uses
       ▼
┌──────────────┐
│    Facade    │
├──────────────┤
│ + operation()│────┐
└──────────────┘    │
                    │ delegates to
       ┌────────────┼────────────┐
       ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│SystemA  │  │SystemB  │  │SystemC  │
├─────────┤  ├─────────┤  ├─────────┤
│methodA()│  │methodB()│  │methodC()│
│methodA2()│ │methodB2()│ │methodC2()│
└─────────┘  └─────────┘  └─────────┘
     Complex Subsystem
```

### Key Components

1. **Facade**: Provides simple methods that delegate to subsystem classes
2. **Subsystem Classes**: Implement subsystem functionality, handle work assigned by facade
3. **Client**: Uses facade instead of calling subsystem classes directly

## Implementation Patterns

### Basic Facade Pattern

```typescript
// Complex subsystem classes
class CPU {
  freeze(): void {
    console.log('CPU: Freezing processor');
  }

  jump(position: number): void {
    console.log(`CPU: Jumping to position ${position}`);
  }

  execute(): void {
    console.log('CPU: Executing instructions');
  }
}

class Memory {
  load(position: number, data: string): void {
    console.log(`Memory: Loading data "${data}" at position ${position}`);
  }

  getData(position: number): string {
    console.log(`Memory: Getting data from position ${position}`);
    return 'boot_data';
  }
}

class HardDrive {
  read(sector: number, size: number): string {
    console.log(`HardDrive: Reading ${size} bytes from sector ${sector}`);
    return 'boot_sector_data';
  }
}

// Facade - simplifies the boot process
class ComputerFacade {
  private cpu: CPU;
  private memory: Memory;
  private hardDrive: HardDrive;

  constructor() {
    this.cpu = new CPU();
    this.memory = new Memory();
    this.hardDrive = new HardDrive();
  }

  start(): void {
    console.log('=== Starting computer ===');
    this.cpu.freeze();
    const bootData = this.hardDrive.read(0, 1024);
    this.memory.load(0, bootData);
    this.cpu.jump(0);
    this.cpu.execute();
    console.log('=== Computer started ===\n');
  }
}

// Client code
const computer = new ComputerFacade();
computer.start(); // Simple one-liner instead of complex subsystem calls
```

### E-commerce Checkout Facade

```typescript
// Subsystem: Payment Processing
class PaymentProcessor {
  validatePaymentInfo(cardNumber: string, cvv: string): boolean {
    console.log('Validating payment information...');
    return cardNumber.length === 16 && cvv.length === 3;
  }

  processPayment(amount: number, cardNumber: string): string {
    console.log(`Processing payment of $${amount}...`);
    return `PAY-${Date.now()}`;
  }

  refundPayment(transactionId: string): boolean {
    console.log(`Refunding transaction ${transactionId}...`);
    return true;
  }
}

// Subsystem: Inventory Management
class InventoryManager {
  checkStock(productId: string, quantity: number): boolean {
    console.log(`Checking stock for product ${productId}...`);
    return true; // Simplified
  }

  reserveItems(productId: string, quantity: number): string {
    console.log(`Reserving ${quantity} units of ${productId}...`);
    return `RES-${Date.now()}`;
  }

  releaseReservation(reservationId: string): void {
    console.log(`Releasing reservation ${reservationId}...`);
  }

  updateStock(productId: string, quantity: number): void {
    console.log(`Updating stock for ${productId}: -${quantity}`);
  }
}

// Subsystem: Shipping
class ShippingService {
  calculateShipping(address: Address, weight: number): number {
    console.log('Calculating shipping cost...');
    return 9.99;
  }

  createShipment(orderId: string, address: Address): string {
    console.log(`Creating shipment for order ${orderId}...`);
    return `SHIP-${Date.now()}`;
  }

  trackShipment(shipmentId: string): string {
    return 'In transit';
  }
}

// Subsystem: Notification
class NotificationService {
  sendOrderConfirmation(email: string, orderId: string): void {
    console.log(`Sending confirmation email to ${email} for order ${orderId}`);
  }

  sendShippingNotification(email: string, trackingNumber: string): void {
    console.log(`Sending shipping notification to ${email}`);
  }

  sendPaymentReceipt(email: string, amount: number): void {
    console.log(`Sending receipt to ${email} for $${amount}`);
  }
}

// Subsystem: Order Management
class OrderManager {
  createOrder(customerId: string, items: OrderItem[]): string {
    console.log(`Creating order for customer ${customerId}...`);
    return `ORD-${Date.now()}`;
  }

  updateOrderStatus(orderId: string, status: string): void {
    console.log(`Updating order ${orderId} status to ${status}`);
  }

  getOrderDetails(orderId: string): Order {
    return {
      id: orderId,
      status: 'pending',
      items: [],
      total: 0
    };
  }
}

// Types
interface Address {
  street: string;
  city: string;
  state: string;
  zip: string;
}

interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

interface Order {
  id: string;
  status: string;
  items: OrderItem[];
  total: number;
}

interface CheckoutResult {
  success: boolean;
  orderId?: string;
  error?: string;
}

// Facade - simplifies entire checkout process
class CheckoutFacade {
  private payment: PaymentProcessor;
  private inventory: InventoryManager;
  private shipping: ShippingService;
  private notification: NotificationService;
  private orderManager: OrderManager;

  constructor() {
    this.payment = new PaymentProcessor();
    this.inventory = new InventoryManager();
    this.shipping = new ShippingService();
    this.notification = new NotificationService();
    this.orderManager = new OrderManager();
  }

  async checkout(
    customerId: string,
    email: string,
    items: OrderItem[],
    shippingAddress: Address,
    cardNumber: string,
    cvv: string
  ): Promise<CheckoutResult> {
    console.log('\n=== Starting Checkout Process ===\n');

    try {
      // Step 1: Validate payment
      if (!this.payment.validatePaymentInfo(cardNumber, cvv)) {
        return { success: false, error: 'Invalid payment information' };
      }

      // Step 2: Check inventory
      const reservations: string[] = [];
      for (const item of items) {
        if (!this.inventory.checkStock(item.productId, item.quantity)) {
          // Release any reservations made so far
          reservations.forEach(res => this.inventory.releaseReservation(res));
          return { success: false, error: `Item ${item.productId} out of stock` };
        }
        const reservationId = this.inventory.reserveItems(item.productId, item.quantity);
        reservations.push(reservationId);
      }

      // Step 3: Calculate total
      const itemTotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
      const totalWeight = items.reduce((sum, item) => sum + item.quantity, 0);
      const shippingCost = this.shipping.calculateShipping(shippingAddress, totalWeight);
      const total = itemTotal + shippingCost;

      // Step 4: Process payment
      const transactionId = this.payment.processPayment(total, cardNumber);

      // Step 5: Create order
      const orderId = this.orderManager.createOrder(customerId, items);

      // Step 6: Update inventory
      items.forEach(item => {
        this.inventory.updateStock(item.productId, item.quantity);
      });

      // Step 7: Create shipment
      const shipmentId = this.shipping.createShipment(orderId, shippingAddress);

      // Step 8: Send notifications
      this.notification.sendOrderConfirmation(email, orderId);
      this.notification.sendPaymentReceipt(email, total);
      this.notification.sendShippingNotification(email, shipmentId);

      // Step 9: Update order status
      this.orderManager.updateOrderStatus(orderId, 'completed');

      console.log('\n=== Checkout Completed Successfully ===\n');
      return { success: true, orderId };

    } catch (error) {
      console.error('Checkout failed:', error);
      return { success: false, error: 'Checkout process failed' };
    }
  }

  async getOrderStatus(orderId: string): Promise<Order> {
    return this.orderManager.getOrderDetails(orderId);
  }

  async trackShipment(shipmentId: string): Promise<string> {
    return this.shipping.trackShipment(shipmentId);
  }
}

// Client usage - extremely simple!
async function demonstrateCheckout() {
  const checkout = new CheckoutFacade();

  const result = await checkout.checkout(
    'CUST-123',
    'customer@example.com',
    [
      { productId: 'PROD-001', quantity: 2, price: 29.99 },
      { productId: 'PROD-002', quantity: 1, price: 49.99 }
    ],
    {
      street: '123 Main St',
      city: 'Springfield',
      state: 'IL',
      zip: '62701'
    },
    '1234567890123456',
    '123'
  );

  if (result.success) {
    console.log(`Order placed successfully! Order ID: ${result.orderId}`);
  } else {
    console.log(`Order failed: ${result.error}`);
  }
}
```

### React API Facade

```typescript
// Complex API subsystems
class UserAPI {
  async fetchUser(id: string): Promise<any> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
  }

  async updateUser(id: string, data: any): Promise<any> {
    const response = await fetch(`/api/users/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    return response.json();
  }
}

class ProfileAPI {
  async fetchProfile(userId: string): Promise<any> {
    const response = await fetch(`/api/profiles/${userId}`);
    return response.json();
  }

  async uploadAvatar(userId: string, file: File): Promise<string> {
    const formData = new FormData();
    formData.append('avatar', file);
    const response = await fetch(`/api/profiles/${userId}/avatar`, {
      method: 'POST',
      body: formData
    });
    const data = await response.json();
    return data.avatarUrl;
  }
}

class PreferencesAPI {
  async fetchPreferences(userId: string): Promise<any> {
    const response = await fetch(`/api/preferences/${userId}`);
    return response.json();
  }

  async updatePreferences(userId: string, prefs: any): Promise<any> {
    const response = await fetch(`/api/preferences/${userId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(prefs)
    });
    return response.json();
  }
}

class ActivityAPI {
  async fetchRecentActivity(userId: string, limit: number = 10): Promise<any[]> {
    const response = await fetch(`/api/activity/${userId}?limit=${limit}`);
    return response.json();
  }
}

// Facade - unified user data interface
class UserDataFacade {
  private userAPI: UserAPI;
  private profileAPI: ProfileAPI;
  private preferencesAPI: PreferencesAPI;
  private activityAPI: ActivityAPI;

  constructor() {
    this.userAPI = new UserAPI();
    this.profileAPI = new ProfileAPI();
    this.preferencesAPI = new PreferencesAPI();
    this.activityAPI = new ActivityAPI();
  }

  async getCompleteUserData(userId: string): Promise<CompleteUserData> {
    // Fetch all user data in parallel
    const [user, profile, preferences, activity] = await Promise.all([
      this.userAPI.fetchUser(userId),
      this.profileAPI.fetchProfile(userId),
      this.preferencesAPI.fetchPreferences(userId),
      this.activityAPI.fetchRecentActivity(userId)
    ]);

    return {
      id: userId,
      name: user.name,
      email: user.email,
      avatarUrl: profile.avatarUrl,
      bio: profile.bio,
      theme: preferences.theme,
      notifications: preferences.notifications,
      recentActivity: activity
    };
  }

  async updateUserProfile(
    userId: string,
    updates: {
      name?: string;
      bio?: string;
      avatar?: File;
      theme?: string;
      notifications?: any;
    }
  ): Promise<void> {
    const promises: Promise<any>[] = [];

    if (updates.name) {
      promises.push(this.userAPI.updateUser(userId, { name: updates.name }));
    }

    if (updates.bio) {
      promises.push(this.profileAPI.fetchProfile(userId).then(() => {
        // Update profile with bio
      }));
    }

    if (updates.avatar) {
      promises.push(this.profileAPI.uploadAvatar(userId, updates.avatar));
    }

    if (updates.theme || updates.notifications) {
      promises.push(this.preferencesAPI.updatePreferences(userId, {
        theme: updates.theme,
        notifications: updates.notifications
      }));
    }

    await Promise.all(promises);
  }
}

interface CompleteUserData {
  id: string;
  name: string;
  email: string;
  avatarUrl: string;
  bio: string;
  theme: string;
  notifications: any;
  recentActivity: any[];
}

// React component using facade
import React, { useEffect, useState } from 'react';

const userDataFacade = new UserDataFacade();

export const UserDashboard: React.FC<{ userId: string }> = ({ userId }) => {
  const [userData, setUserData] = useState<CompleteUserData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadUserData();
  }, [userId]);

  const loadUserData = async () => {
    setLoading(true);
    try {
      // Simple one-liner thanks to facade!
      const data = await userDataFacade.getCompleteUserData(userId);
      setUserData(data);
    } catch (error) {
      console.error('Failed to load user data:', error);
    } finally {
      setLoading(false);
    }
  };

  const handleProfileUpdate = async (updates: any) => {
    try {
      // Another simple facade call
      await userDataFacade.updateUserProfile(userId, updates);
      await loadUserData(); // Refresh
    } catch (error) {
      console.error('Failed to update profile:', error);
    }
  };

  if (loading) return <div>Loading...</div>;
  if (!userData) return <div>No data</div>;

  return (
    <div className="user-dashboard">
      <div className="user-header">
        <img src={userData.avatarUrl} alt={userData.name} />
        <h1>{userData.name}</h1>
        <p>{userData.email}</p>
      </div>
      
      <div className="user-bio">
        <p>{userData.bio}</p>
      </div>

      <div className="user-activity">
        <h2>Recent Activity</h2>
        <ul>
          {userData.recentActivity.map((activity, i) => (
            <li key={i}>{activity.description}</li>
          ))}
        </ul>
      </div>

      <button onClick={() => handleProfileUpdate({ theme: 'dark' })}>
        Switch to Dark Mode
      </button>
    </div>
  );
};
```

### Angular Service Facade

```typescript
// Angular services (subsystems)
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ProductService {
  constructor(private http: HttpClient) {}

  getProducts(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products');
  }

  getProduct(id: string): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`);
  }
}

@Injectable({ providedIn: 'root' })
export class CartService {
  private items: CartItem[] = [];

  addItem(product: Product, quantity: number): void {
    const existing = this.items.find(item => item.product.id === product.id);
    if (existing) {
      existing.quantity += quantity;
    } else {
      this.items.push({ product, quantity });
    }
  }

  getItems(): CartItem[] {
    return this.items;
  }

  getTotal(): number {
    return this.items.reduce((sum, item) => 
      sum + item.product.price * item.quantity, 0
    );
  }

  clear(): void {
    this.items = [];
  }
}

@Injectable({ providedIn: 'root' })
export class OrderService {
  constructor(private http: HttpClient) {}

  createOrder(order: Order): Observable<Order> {
    return this.http.post<Order>('/api/orders', order);
  }

  getOrders(userId: string): Observable<Order[]> {
    return this.http.get<Order[]>(`/api/orders?userId=${userId}`);
  }
}

@Injectable({ providedIn: 'root' })
export class PaymentService {
  constructor(private http: HttpClient) {}

  processPayment(payment: PaymentInfo): Observable<PaymentResult> {
    return this.http.post<PaymentResult>('/api/payments', payment);
  }
}

// Facade service - simplifies complex operations
@Injectable({ providedIn: 'root' })
export class ShoppingFacade {
  constructor(
    private productService: ProductService,
    private cartService: CartService,
    private orderService: OrderService,
    private paymentService: PaymentService
  ) {}

  // Simplified method for adding product to cart
  async addToCart(productId: string, quantity: number): Promise<void> {
    const product = await this.productService.getProduct(productId).toPromise();
    if (product) {
      this.cartService.addItem(product, quantity);
    }
  }

  // Simplified checkout process
  async checkout(userId: string, paymentInfo: PaymentInfo): Promise<CheckoutResult> {
    try {
      // Get cart items
      const items = this.cartService.getItems();
      if (items.length === 0) {
        return { success: false, error: 'Cart is empty' };
      }

      // Calculate total
      const total = this.cartService.getTotal();

      // Process payment
      const paymentResult = await this.paymentService.processPayment({
        ...paymentInfo,
        amount: total
      }).toPromise();

      if (!paymentResult?.success) {
        return { success: false, error: 'Payment failed' };
      }

      // Create order
      const order: Order = {
        userId,
        items: items.map(item => ({
          productId: item.product.id,
          quantity: item.quantity,
          price: item.product.price
        })),
        total,
        paymentId: paymentResult.transactionId,
        status: 'confirmed'
      };

      const createdOrder = await this.orderService.createOrder(order).toPromise();

      // Clear cart
      this.cartService.clear();

      return {
        success: true,
        orderId: createdOrder?.id
      };

    } catch (error) {
      console.error('Checkout error:', error);
      return { success: false, error: 'Checkout failed' };
    }
  }

  // Get user's shopping overview
  async getShoppingOverview(userId: string): Promise<ShoppingOverview> {
    const [orders, cartItems] = await Promise.all([
      this.orderService.getOrders(userId).toPromise(),
      Promise.resolve(this.cartService.getItems())
    ]);

    return {
      recentOrders: orders || [],
      cartItems: cartItems,
      cartTotal: this.cartService.getTotal(),
      orderCount: orders?.length || 0
    };
  }
}

// Component using facade
@Component({
  selector: 'app-shopping',
  template: `
    <div class="shopping">
      <button (click)="addProduct('prod-123', 1)">
        Add to Cart
      </button>
      
      <button (click)="checkout()">
        Checkout
      </button>

      <div *ngIf="overview">
        <p>Cart Total: {{ overview.cartTotal | currency }}</p>
        <p>Total Orders: {{ overview.orderCount }}</p>
      </div>
    </div>
  `
})
export class ShoppingComponent implements OnInit {
  overview: ShoppingOverview | null = null;

  constructor(private shoppingFacade: ShoppingFacade) {}

  async ngOnInit() {
    this.overview = await this.shoppingFacade.getShoppingOverview('user-123');
  }

  async addProduct(productId: string, quantity: number) {
    await this.shoppingFacade.addToCart(productId, quantity);
    this.overview = await this.shoppingFacade.getShoppingOverview('user-123');
  }

  async checkout() {
    const result = await this.shoppingFacade.checkout('user-123', {
      cardNumber: '1234567890123456',
      cvv: '123'
    });

    if (result.success) {
      alert(`Order placed! Order ID: ${result.orderId}`);
    } else {
      alert(`Checkout failed: ${result.error}`);
    }
  }
}

// Types
interface Product {
  id: string;
  name: string;
  price: number;
}

interface CartItem {
  product: Product;
  quantity: number;
}

interface Order {
  id?: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
  paymentId: string;
  status: string;
}

interface PaymentInfo {
  cardNumber: string;
  cvv: string;
  amount?: number;
}

interface PaymentResult {
  success: boolean;
  transactionId: string;
}

interface CheckoutResult {
  success: boolean;
  orderId?: string;
  error?: string;
}

interface ShoppingOverview {
  recentOrders: Order[];
  cartItems: CartItem[];
  cartTotal: number;
  orderCount: number;
}
```

### Database Facade

```typescript
// Complex database operations
class DatabaseConnection {
  connect(connectionString: string): void {
    console.log('Connecting to database...');
  }

  disconnect(): void {
    console.log('Disconnecting from database...');
  }

  beginTransaction(): void {
    console.log('Beginning transaction...');
  }

  commit(): void {
    console.log('Committing transaction...');
  }

  rollback(): void {
    console.log('Rolling back transaction...');
  }
}

class QueryBuilder {
  private query: string = '';

  select(table: string, columns: string[]): this {
    this.query = `SELECT ${columns.join(', ')} FROM ${table}`;
    return this;
  }

  where(condition: string): this {
    this.query += ` WHERE ${condition}`;
    return this;
  }

  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.query += ` ORDER BY ${column} ${direction}`;
    return this;
  }

  limit(count: number): this {
    this.query += ` LIMIT ${count}`;
    return this;
  }

  build(): string {
    return this.query;
  }
}

class QueryExecutor {
  execute<T>(query: string): Promise<T[]> {
    console.log(`Executing: ${query}`);
    return Promise.resolve([]);
  }

  executeSingle<T>(query: string): Promise<T | null> {
    console.log(`Executing single: ${query}`);
    return Promise.resolve(null);
  }
}

class CacheManager {
  private cache = new Map<string, any>();

  get(key: string): any {
    return this.cache.get(key);
  }

  set(key: string, value: any, ttl: number = 60000): void {
    this.cache.set(key, value);
    setTimeout(() => this.cache.delete(key), ttl);
  }

  has(key: string): boolean {
    return this.cache.has(key);
  }

  clear(): void {
    this.cache.clear();
  }
}

// Facade - simplified database operations
class DatabaseFacade {
  private connection: DatabaseConnection;
  private queryBuilder: QueryBuilder;
  private executor: QueryExecutor;
  private cache: CacheManager;

  constructor(connectionString: string) {
    this.connection = new DatabaseConnection();
    this.queryBuilder = new QueryBuilder();
    this.executor = new QueryExecutor();
    this.cache = new CacheManager();
    
    this.connection.connect(connectionString);
  }

  async find<T>(
    table: string,
    condition: string,
    options?: { orderBy?: string; limit?: number }
  ): Promise<T[]> {
    const cacheKey = `${table}:${condition}:${JSON.stringify(options)}`;
    
    if (this.cache.has(cacheKey)) {
      console.log('Returning cached results');
      return this.cache.get(cacheKey);
    }

    let builder = this.queryBuilder.select(table, ['*']).where(condition);
    
    if (options?.orderBy) {
      builder = builder.orderBy(options.orderBy);
    }
    
    if (options?.limit) {
      builder = builder.limit(options.limit);
    }

    const query = builder.build();
    const results = await this.executor.execute<T>(query);
    
    this.cache.set(cacheKey, results);
    return results;
  }

  async findOne<T>(table: string, id: string): Promise<T | null> {
    const cacheKey = `${table}:${id}`;
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey);
    }

    const query = this.queryBuilder
      .select(table, ['*'])
      .where(`id = '${id}'`)
      .build();

    const result = await this.executor.executeSingle<T>(query);
    
    if (result) {
      this.cache.set(cacheKey, result);
    }
    
    return result;
  }

  async save<T extends { id?: string }>(table: string, data: T): Promise<T> {
    this.connection.beginTransaction();
    
    try {
      // Simplified save logic
      console.log(`Saving to ${table}:`, data);
      
      this.cache.clear(); // Invalidate cache
      this.connection.commit();
      
      return data;
    } catch (error) {
      this.connection.rollback();
      throw error;
    }
  }

  async delete(table: string, id: string): Promise<void> {
    this.connection.beginTransaction();
    
    try {
      console.log(`Deleting from ${table} where id = ${id}`);
      
      this.cache.clear(); // Invalidate cache
      this.connection.commit();
    } catch (error) {
      this.connection.rollback();
      throw error;
    }
  }

  close(): void {
    this.connection.disconnect();
    this.cache.clear();
  }
}

// Usage - simple and clean
const db = new DatabaseFacade('postgresql://localhost:5432/mydb');

// Simple queries without knowing complex subsystem details
const users = await db.find('users', "status = 'active'", {
  orderBy: 'created_at',
  limit: 10
});

const user = await db.findOne('users', '123');

await db.save('users', { id: '123', name: 'John Doe' });

await db.delete('users', '456');

db.close();
```

## Common Mistakes

### 1. Facade Doing Too Much Business Logic

```typescript
// BAD - Facade contains business logic
class BadFacade {
  processOrder(order: Order) {
    // Don't implement business rules in facade!
    if (order.total > 1000) {
      order.discount = 0.1;
    }
    // Facade should delegate, not implement
  }
}

// GOOD - Facade delegates to subsystems
class GoodFacade {
  processOrder(order: Order) {
    const discount = this.discountService.calculate(order);
    const validated = this.validator.validate(order);
    return this.orderService.create(validated);
  }
}
```

### 2. Tight Coupling to Subsystems

```typescript
// BAD - Facade tightly coupled
class BadFacade {
  method() {
    const result = new ConcreteSubsystem().doWork();
    return result.specificProperty; // Knows too much!
  }
}

// GOOD - Loose coupling through interfaces
class GoodFacade {
  constructor(private subsystem: SubsystemInterface) {}
  
  method() {
    return this.subsystem.doWork();
  }
}
```

### 3. Not Truly Simplifying

```typescript
// BAD - Facade as complicated as subsystems
class BadFacade {
  complexMethod(a, b, c, d, e, f) {
    // Just as complex as calling subsystems directly!
  }
}

// GOOD - Simplified interface
class GoodFacade {
  simpleMethod(config: SimpleConfig) {
    // Hides complexity, provides clear interface
    return this.subsystems.handleComplex(config);
  }
}
```

## Best Practices

1. **Provide simple, intuitive methods** that hide complexity
2. **Don't prevent direct subsystem access** if needed
3. **Keep facade focused** on coordination, not business logic
4. **Use dependency injection** for subsystems
5. **Document what facade does** and when to use it
6. **Make facade optional** - clients can bypass if needed
7. **Group related operations** logically
8. **Handle errors gracefully** and consistently
9. **Consider multiple facades** for different client needs
10. **Don't create god objects** - keep facades focused

## When to Use Facade Pattern

### Good Use Cases

- Simplifying complex library or framework usage
- Providing unified interface to set of interfaces
- Decoupling client code from subsystems
- Creating layers in application architecture
- Wrapping poorly designed APIs
- Reducing dependencies on external code
- Making code easier to test (mock the facade)

### When to Avoid

- Subsystem is already simple
- Clients need full control over subsystems
- Creates unnecessary abstraction layer
- When it becomes a bottleneck

## Interview Questions

### Q1: What's the difference between Facade and Adapter patterns?

**Answer:**
- **Facade**: Simplifies complex subsystem with new interface (many-to-one)
- **Adapter**: Makes incompatible interface compatible (one-to-one)

### Q2: Can clients still access subsystems directly when using Facade?

**Answer:** Yes! Facade doesn't prevent direct access. It provides a simpler interface but clients can bypass it when they need more control.

### Q3: How does Facade pattern support the Dependency Inversion Principle?

**Answer:** Facade creates an abstraction layer that high-level modules depend on, rather than depending directly on low-level subsystem modules.

### Q4: What's the difference between Facade and Mediator patterns?

**Answer:**
- **Facade**: Simplifies complex subsystem (unidirectional, facade knows subsystems)
- **Mediator**: Enables communication between colleagues (bidirectional, colleagues know mediator)

### Q5: Should a Facade contain business logic?

**Answer:** No. Facade should coordinate and delegate to subsystems, not implement business logic. Keep it thin and focused on simplification.

### Q6: How many facades should an application have?

**Answer:** As many as needed for different client contexts. Large systems might have multiple facades for different layers or client types.

### Q7: How do you test code that uses a facade?

**Answer:** Mock the facade interface in tests. The facade itself should have integration tests verifying it correctly coordinates subsystems.

### Q8: What's the relationship between Facade pattern and layered architecture?

**Answer:** Facades often define layer boundaries. Each layer might expose a facade interface to the layer above, hiding its internal complexity.

## Key Takeaways

1. Facade provides simplified interface to complex subsystem
2. Hides complexity but doesn't prevent direct subsystem access
3. Reduces coupling between clients and subsystems
4. Makes code easier to use, test, and maintain
5. Should coordinate and delegate, not implement logic
6. Different from Adapter (which adapts) and Mediator (which mediates)
7. Common in API wrappers and service layers
8. Supports loose coupling and dependency inversion
9. Can have multiple facades for different purposes
10. Essential for managing complexity in large applications

## Resources

- [Refactoring Guru: Facade Pattern](https://refactoring.guru/design-patterns/facade)
- [Gang of Four Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- [Martin Fowler: Gateway Pattern](https://martinfowler.com/eaaCatalog/gateway.html)
- [Source Making: Facade Pattern](https://sourcemaking.com/design_patterns/facade)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microsoft: Facade Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/facade)
