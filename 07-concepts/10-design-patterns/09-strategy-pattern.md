# Strategy Pattern

## Overview

The Strategy pattern is a behavioral design pattern that defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it. This pattern promotes the Open/Closed Principle by allowing new strategies to be added without modifying existing code.

## Core Concepts

### Strategy Pattern Structure

```
┌─────────────────┐
│    Context      │
├─────────────────┤
│ - strategy      │◆─────────┐
│ + setStrategy() │          │
│ + execute()     │          │
└─────────────────┘          │
                             │
                    ┌────────▼───────┐
                    │   Strategy     │
                    │  «interface»   │
                    ├────────────────┤
                    │ + execute()    │
                    └────────┬───────┘
                             △
                    ┌────────┴────────┐
                    │                 │
          ┌─────────┴────────┐ ┌─────┴──────────┐
          │ ConcreteStrategyA│ │ConcreteStrategyB│
          ├──────────────────┤ ├────────────────┤
          │ + execute()      │ │ + execute()    │
          └──────────────────┘ └────────────────┘
```

### Key Components

1. **Strategy Interface**: Defines common interface for all strategies
2. **Concrete Strategies**: Implement different algorithms
3. **Context**: Maintains reference to strategy, delegates to it
4. **Client**: Creates and configures strategies

## Implementation Patterns

### Basic Strategy Pattern

```typescript
// Strategy interface
interface PaymentStrategy {
  pay(amount: number): Promise<PaymentResult>;
  validatePaymentDetails(): boolean;
}

interface PaymentResult {
  success: boolean;
  transactionId?: string;
  error?: string;
}

// Concrete strategies
class CreditCardStrategy implements PaymentStrategy {
  constructor(
    private cardNumber: string,
    private cvv: string,
    private expiryDate: string
  ) {}

  validatePaymentDetails(): boolean {
    return (
      this.cardNumber.length === 16 &&
      this.cvv.length === 3 &&
      /^\d{2}\/\d{2}$/.test(this.expiryDate)
    );
  }

  async pay(amount: number): Promise<PaymentResult> {
    if (!this.validatePaymentDetails()) {
      return { success: false, error: 'Invalid card details' };
    }

    console.log(`Processing credit card payment of $${amount}`);
    
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    return {
      success: true,
      transactionId: `CC-${Date.now()}`
    };
  }
}

class PayPalStrategy implements PaymentStrategy {
  constructor(
    private email: string,
    private password: string
  ) {}

  validatePaymentDetails(): boolean {
    return (
      this.email.includes('@') &&
      this.password.length >= 8
    );
  }

  async pay(amount: number): Promise<PaymentResult> {
    if (!this.validatePaymentDetails()) {
      return { success: false, error: 'Invalid PayPal credentials' };
    }

    console.log(`Processing PayPal payment of $${amount}`);
    
    await new Promise(resolve => setTimeout(resolve, 800));
    
    return {
      success: true,
      transactionId: `PP-${Date.now()}`
    };
  }
}

class CryptoStrategy implements PaymentStrategy {
  constructor(
    private walletAddress: string,
    private currency: 'BTC' | 'ETH'
  ) {}

  validatePaymentDetails(): boolean {
    return this.walletAddress.length >= 26;
  }

  async pay(amount: number): Promise<PaymentResult> {
    if (!this.validatePaymentDetails()) {
      return { success: false, error: 'Invalid wallet address' };
    }

    console.log(`Processing ${this.currency} payment of $${amount}`);
    
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    return {
      success: true,
      transactionId: `CRYPTO-${Date.now()}`
    };
  }
}

// Context
class PaymentProcessor {
  private strategy: PaymentStrategy;

  constructor(strategy: PaymentStrategy) {
    this.strategy = strategy;
  }

  setStrategy(strategy: PaymentStrategy): void {
    this.strategy = strategy;
  }

  async processPayment(amount: number): Promise<PaymentResult> {
    if (!this.strategy.validatePaymentDetails()) {
      return {
        success: false,
        error: 'Payment validation failed'
      };
    }

    return await this.strategy.pay(amount);
  }
}

// Usage
async function demonstratePayment() {
  const processor = new PaymentProcessor(
    new CreditCardStrategy('1234567890123456', '123', '12/25')
  );

  let result = await processor.processPayment(99.99);
  console.log('Credit Card Result:', result);

  // Switch strategy at runtime
  processor.setStrategy(
    new PayPalStrategy('user@example.com', 'securepass123')
  );

  result = await processor.processPayment(49.99);
  console.log('PayPal Result:', result);

  // Try another strategy
  processor.setStrategy(
    new CryptoStrategy('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', 'BTC')
  );

  result = await processor.processPayment(199.99);
  console.log('Crypto Result:', result);
}
```

### Form Validation Strategies (React)

```typescript
// validation-strategies.ts
export interface ValidationStrategy {
  validate(value: any): ValidationResult;
}

export interface ValidationResult {
  isValid: boolean;
  error?: string;
}

// Email validation strategy
export class EmailValidationStrategy implements ValidationStrategy {
  validate(value: string): ValidationResult {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    
    if (!value) {
      return { isValid: false, error: 'Email is required' };
    }
    
    if (!emailRegex.test(value)) {
      return { isValid: false, error: 'Invalid email format' };
    }
    
    return { isValid: true };
  }
}

// Password validation strategy
export class PasswordValidationStrategy implements ValidationStrategy {
  constructor(
    private minLength: number = 8,
    private requireSpecialChar: boolean = true
  ) {}

  validate(value: string): ValidationResult {
    if (!value) {
      return { isValid: false, error: 'Password is required' };
    }

    if (value.length < this.minLength) {
      return {
        isValid: false,
        error: `Password must be at least ${this.minLength} characters`
      };
    }

    if (this.requireSpecialChar && !/[!@#$%^&*]/.test(value)) {
      return {
        isValid: false,
        error: 'Password must contain a special character'
      };
    }

    if (!/\d/.test(value)) {
      return {
        isValid: false,
        error: 'Password must contain a number'
      };
    }

    if (!/[A-Z]/.test(value)) {
      return {
        isValid: false,
        error: 'Password must contain an uppercase letter'
      };
    }

    return { isValid: true };
  }
}

// Phone validation strategy
export class PhoneValidationStrategy implements ValidationStrategy {
  constructor(private countryCode: 'US' | 'UK' | 'IN' = 'US') {}

  validate(value: string): ValidationResult {
    if (!value) {
      return { isValid: false, error: 'Phone number is required' };
    }

    const patterns = {
      US: /^\(?([0-9]{3})\)?[-. ]?([0-9]{3})[-. ]?([0-9]{4})$/,
      UK: /^(\+44\s?7\d{3}|\(?07\d{3}\)?)\s?\d{3}\s?\d{3}$/,
      IN: /^(\+91[\-\s]?)?[0]?(91)?[789]\d{9}$/
    };

    if (!patterns[this.countryCode].test(value)) {
      return {
        isValid: false,
        error: `Invalid ${this.countryCode} phone number format`
      };
    }

    return { isValid: true };
  }
}

// Credit card validation strategy
export class CreditCardValidationStrategy implements ValidationStrategy {
  validate(value: string): ValidationResult {
    if (!value) {
      return { isValid: false, error: 'Card number is required' };
    }

    // Remove spaces and dashes
    const cardNumber = value.replace(/[\s-]/g, '');

    if (!/^\d+$/.test(cardNumber)) {
      return { isValid: false, error: 'Card number must contain only digits' };
    }

    if (cardNumber.length < 13 || cardNumber.length > 19) {
      return { isValid: false, error: 'Invalid card number length' };
    }

    // Luhn algorithm
    if (!this.luhnCheck(cardNumber)) {
      return { isValid: false, error: 'Invalid card number' };
    }

    return { isValid: true };
  }

  private luhnCheck(cardNumber: string): boolean {
    let sum = 0;
    let isEven = false;

    for (let i = cardNumber.length - 1; i >= 0; i--) {
      let digit = parseInt(cardNumber[i], 10);

      if (isEven) {
        digit *= 2;
        if (digit > 9) {
          digit -= 9;
        }
      }

      sum += digit;
      isEven = !isEven;
    }

    return sum % 10 === 0;
  }
}

// React component using validation strategies
import React, { useState } from 'react';

interface FormFieldProps {
  label: string;
  name: string;
  type?: string;
  validationStrategy: ValidationStrategy;
  value: string;
  onChange: (name: string, value: string) => void;
}

export const FormField: React.FC<FormFieldProps> = ({
  label,
  name,
  type = 'text',
  validationStrategy,
  value,
  onChange
}) => {
  const [touched, setTouched] = useState(false);
  const [validationResult, setValidationResult] = useState<ValidationResult>({
    isValid: true
  });

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    onChange(name, newValue);
    
    if (touched) {
      const result = validationStrategy.validate(newValue);
      setValidationResult(result);
    }
  };

  const handleBlur = () => {
    setTouched(true);
    const result = validationStrategy.validate(value);
    setValidationResult(result);
  };

  return (
    <div className="form-field">
      <label htmlFor={name}>{label}</label>
      <input
        id={name}
        name={name}
        type={type}
        value={value}
        onChange={handleChange}
        onBlur={handleBlur}
        className={!validationResult.isValid ? 'error' : ''}
      />
      {!validationResult.isValid && (
        <span className="error-message">{validationResult.error}</span>
      )}
    </div>
  );
};

// Form component
export const RegistrationForm: React.FC = () => {
  const [formData, setFormData] = useState({
    email: '',
    password: '',
    phone: '',
    cardNumber: ''
  });

  const handleFieldChange = (name: string, value: string) => {
    setFormData(prev => ({ ...prev, [name]: value }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log('Form submitted:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <FormField
        label="Email"
        name="email"
        type="email"
        validationStrategy={new EmailValidationStrategy()}
        value={formData.email}
        onChange={handleFieldChange}
      />

      <FormField
        label="Password"
        name="password"
        type="password"
        validationStrategy={new PasswordValidationStrategy(8, true)}
        value={formData.password}
        onChange={handleFieldChange}
      />

      <FormField
        label="Phone"
        name="phone"
        type="tel"
        validationStrategy={new PhoneValidationStrategy('US')}
        value={formData.phone}
        onChange={handleFieldChange}
      />

      <FormField
        label="Card Number"
        name="cardNumber"
        validationStrategy={new CreditCardValidationStrategy()}
        value={formData.cardNumber}
        onChange={handleFieldChange}
      />

      <button type="submit">Register</button>
    </form>
  );
};
```

### Sorting Strategies

```typescript
// Sorting strategies
interface SortStrategy<T> {
  sort(items: T[]): T[];
}

class BubbleSortStrategy<T> implements SortStrategy<T> {
  constructor(private compareFn: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    const arr = [...items];
    const n = arr.length;

    for (let i = 0; i < n - 1; i++) {
      for (let j = 0; j < n - i - 1; j++) {
        if (this.compareFn(arr[j], arr[j + 1]) > 0) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }

    return arr;
  }
}

class QuickSortStrategy<T> implements SortStrategy<T> {
  constructor(private compareFn: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    if (items.length <= 1) return items;

    const arr = [...items];
    return this.quickSort(arr, 0, arr.length - 1);
  }

  private quickSort(arr: T[], low: number, high: number): T[] {
    if (low < high) {
      const pi = this.partition(arr, low, high);
      this.quickSort(arr, low, pi - 1);
      this.quickSort(arr, pi + 1, high);
    }
    return arr;
  }

  private partition(arr: T[], low: number, high: number): number {
    const pivot = arr[high];
    let i = low - 1;

    for (let j = low; j < high; j++) {
      if (this.compareFn(arr[j], pivot) <= 0) {
        i++;
        [arr[i], arr[j]] = [arr[j], arr[i]];
      }
    }

    [arr[i + 1], arr[high]] = [arr[high], arr[i + 1]];
    return i + 1;
  }
}

class MergeSortStrategy<T> implements SortStrategy<T> {
  constructor(private compareFn: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    if (items.length <= 1) return items;
    return this.mergeSort([...items]);
  }

  private mergeSort(arr: T[]): T[] {
    if (arr.length <= 1) return arr;

    const mid = Math.floor(arr.length / 2);
    const left = this.mergeSort(arr.slice(0, mid));
    const right = this.mergeSort(arr.slice(mid));

    return this.merge(left, right);
  }

  private merge(left: T[], right: T[]): T[] {
    const result: T[] = [];
    let i = 0, j = 0;

    while (i < left.length && j < right.length) {
      if (this.compareFn(left[i], right[j]) <= 0) {
        result.push(left[i++]);
      } else {
        result.push(right[j++]);
      }
    }

    return result.concat(left.slice(i)).concat(right.slice(j));
  }
}

// Sorter context
class Sorter<T> {
  constructor(private strategy: SortStrategy<T>) {}

  setStrategy(strategy: SortStrategy<T>): void {
    this.strategy = strategy;
  }

  sort(items: T[]): T[] {
    const start = performance.now();
    const result = this.strategy.sort(items);
    const end = performance.now();
    
    console.log(`Sorting took ${(end - start).toFixed(2)}ms`);
    return result;
  }
}

// Usage
const numbers = [64, 34, 25, 12, 22, 11, 90];
const compareFn = (a: number, b: number) => a - b;

const sorter = new Sorter(new BubbleSortStrategy(compareFn));
console.log('Bubble Sort:', sorter.sort(numbers));

sorter.setStrategy(new QuickSortStrategy(compareFn));
console.log('Quick Sort:', sorter.sort(numbers));

sorter.setStrategy(new MergeSortStrategy(compareFn));
console.log('Merge Sort:', sorter.sort(numbers));

// Smart strategy selection based on data size
class AdaptiveSorter<T> extends Sorter<T> {
  constructor(private compareFn: (a: T, b: T) => number) {
    super(new QuickSortStrategy(compareFn));
  }

  sort(items: T[]): T[] {
    // Choose strategy based on array size
    if (items.length < 10) {
      this.setStrategy(new BubbleSortStrategy(this.compareFn));
    } else if (items.length < 1000) {
      this.setStrategy(new QuickSortStrategy(this.compareFn));
    } else {
      this.setStrategy(new MergeSortStrategy(this.compareFn));
    }

    return super.sort(items);
  }
}
```

### Compression Strategies

```typescript
// Compression strategies
interface CompressionStrategy {
  compress(data: string): string;
  decompress(data: string): string;
  getCompressionRatio(original: string, compressed: string): number;
}

class GzipCompressionStrategy implements CompressionStrategy {
  compress(data: string): string {
    // Simulated gzip compression
    console.log('Using GZIP compression');
    return `GZIP:${btoa(data)}`;
  }

  decompress(data: string): string {
    if (!data.startsWith('GZIP:')) {
      throw new Error('Invalid GZIP data');
    }
    return atob(data.substring(5));
  }

  getCompressionRatio(original: string, compressed: string): number {
    return (compressed.length / original.length) * 100;
  }
}

class BrotliCompressionStrategy implements CompressionStrategy {
  compress(data: string): string {
    // Simulated Brotli compression (usually better than gzip)
    console.log('Using Brotli compression');
    const compressed = btoa(data);
    // Simulate better compression
    return `BR:${compressed.substring(0, Math.floor(compressed.length * 0.8))}`;
  }

  decompress(data: string): string {
    if (!data.startsWith('BR:')) {
      throw new Error('Invalid Brotli data');
    }
    // Simulated decompression
    return atob(data.substring(3) + '==');
  }

  getCompressionRatio(original: string, compressed: string): number {
    return (compressed.length / original.length) * 100;
  }
}

class LZ4CompressionStrategy implements CompressionStrategy {
  compress(data: string): string {
    // Simulated LZ4 compression (faster, less compression)
    console.log('Using LZ4 compression');
    return `LZ4:${btoa(data)}`;
  }

  decompress(data: string): string {
    if (!data.startsWith('LZ4:')) {
      throw new Error('Invalid LZ4 data');
    }
    return atob(data.substring(4));
  }

  getCompressionRatio(original: string, compressed: string): number {
    return (compressed.length / original.length) * 100;
  }
}

// Compressor context
class DataCompressor {
  constructor(private strategy: CompressionStrategy) {}

  setStrategy(strategy: CompressionStrategy): void {
    this.strategy = strategy;
  }

  compress(data: string): { compressed: string; ratio: number } {
    const compressed = this.strategy.compress(data);
    const ratio = this.strategy.getCompressionRatio(data, compressed);
    
    return { compressed, ratio };
  }

  decompress(data: string): string {
    return this.strategy.decompress(data);
  }
}

// Usage with auto-selection
class SmartCompressor extends DataCompressor {
  compress(data: string): { compressed: string; ratio: number } {
    // Choose strategy based on data characteristics
    if (data.length < 1000) {
      // Small data - use fast LZ4
      this.setStrategy(new LZ4CompressionStrategy());
    } else if (data.includes('http') || data.includes('www')) {
      // Web content - use Brotli
      this.setStrategy(new BrotliCompressionStrategy());
    } else {
      // General data - use GZIP
      this.setStrategy(new GzipCompressionStrategy());
    }

    return super.compress(data);
  }
}
```

### Angular Service with Strategy Pattern

```typescript
// pricing.strategies.ts
export interface PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number;
  getDescription(): string;
}

export class RegularPricingStrategy implements PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number {
    return basePrice * quantity;
  }

  getDescription(): string {
    return 'Regular pricing';
  }
}

export class BulkDiscountStrategy implements PricingStrategy {
  constructor(
    private threshold: number,
    private discountPercent: number
  ) {}

  calculatePrice(basePrice: number, quantity: number): number {
    const total = basePrice * quantity;
    
    if (quantity >= this.threshold) {
      return total * (1 - this.discountPercent / 100);
    }
    
    return total;
  }

  getDescription(): string {
    return `${this.discountPercent}% off for ${this.threshold}+ items`;
  }
}

export class SeasonalPricingStrategy implements PricingStrategy {
  constructor(
    private seasonalMultiplier: number,
    private seasonName: string
  ) {}

  calculatePrice(basePrice: number, quantity: number): number {
    return basePrice * quantity * this.seasonalMultiplier;
  }

  getDescription(): string {
    const change = (this.seasonalMultiplier - 1) * 100;
    return `${this.seasonName} pricing (${change > 0 ? '+' : ''}${change.toFixed(0)}%)`;
  }
}

export class MembershipPricingStrategy implements PricingStrategy {
  constructor(
    private membershipLevel: 'bronze' | 'silver' | 'gold',
    private discounts: Record<string, number> = {
      bronze: 5,
      silver: 10,
      gold: 15
    }
  ) {}

  calculatePrice(basePrice: number, quantity: number): number {
    const total = basePrice * quantity;
    const discount = this.discounts[this.membershipLevel];
    return total * (1 - discount / 100);
  }

  getDescription(): string {
    return `${this.membershipLevel.toUpperCase()} member pricing`;
  }
}

// pricing.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { PricingStrategy, RegularPricingStrategy } from './pricing.strategies';

@Injectable({
  providedIn: 'root'
})
export class PricingService {
  private strategySubject = new BehaviorSubject<PricingStrategy>(
    new RegularPricingStrategy()
  );
  
  public strategy$: Observable<PricingStrategy> = this.strategySubject.asObservable();

  setStrategy(strategy: PricingStrategy): void {
    this.strategySubject.next(strategy);
  }

  calculatePrice(basePrice: number, quantity: number): number {
    const strategy = this.strategySubject.value;
    return strategy.calculatePrice(basePrice, quantity);
  }

  getCurrentStrategy(): PricingStrategy {
    return this.strategySubject.value;
  }

  getStrategyDescription(): string {
    return this.strategySubject.value.getDescription();
  }
}

// product.component.ts
import { Component, OnInit } from '@angular/core';
import { PricingService } from './pricing.service';
import {
  RegularPricingStrategy,
  BulkDiscountStrategy,
  SeasonalPricingStrategy,
  MembershipPricingStrategy
} from './pricing.strategies';

@Component({
  selector: 'app-product',
  template: `
    <div class="product">
      <h2>{{ productName }}</h2>
      <p>Base Price: {{ basePrice | currency }}</p>
      
      <div class="quantity">
        <label>Quantity:</label>
        <input type="number" [(ngModel)]="quantity" min="1" />
      </div>

      <div class="pricing-strategy">
        <label>Pricing Strategy:</label>
        <select (change)="onStrategyChange($event)">
          <option value="regular">Regular</option>
          <option value="bulk">Bulk Discount (10+ items, 15% off)</option>
          <option value="seasonal">Seasonal (Summer +20%)</option>
          <option value="membership">Gold Membership (15% off)</option>
        </select>
      </div>

      <div class="pricing-info">
        <p>Strategy: {{ strategyDescription }}</p>
        <p class="total">Total: {{ totalPrice | currency }}</p>
      </div>

      <button (click)="addToCart()">Add to Cart</button>
    </div>
  `
})
export class ProductComponent implements OnInit {
  productName = 'Premium Widget';
  basePrice = 29.99;
  quantity = 1;
  totalPrice = 0;
  strategyDescription = '';

  constructor(private pricingService: PricingService) {}

  ngOnInit(): void {
    this.updatePrice();
  }

  onStrategyChange(event: Event): void {
    const value = (event.target as HTMLSelectElement).value;

    switch (value) {
      case 'regular':
        this.pricingService.setStrategy(new RegularPricingStrategy());
        break;
      case 'bulk':
        this.pricingService.setStrategy(new BulkDiscountStrategy(10, 15));
        break;
      case 'seasonal':
        this.pricingService.setStrategy(new SeasonalPricingStrategy(1.2, 'Summer'));
        break;
      case 'membership':
        this.pricingService.setStrategy(new MembershipPricingStrategy('gold'));
        break;
    }

    this.updatePrice();
  }

  updatePrice(): void {
    this.totalPrice = this.pricingService.calculatePrice(this.basePrice, this.quantity);
    this.strategyDescription = this.pricingService.getStrategyDescription();
  }

  addToCart(): void {
    console.log(`Added ${this.quantity} items at ${this.totalPrice}`);
  }
}
```

### Route Navigation Strategy

```typescript
// navigation.strategies.ts
interface NavigationStrategy {
  navigate(from: string, to: string): void;
  canNavigate(from: string, to: string): boolean;
}

class DirectNavigationStrategy implements NavigationStrategy {
  navigate(from: string, to: string): void {
    console.log(`Direct navigation: ${from} → ${to}`);
    window.location.href = to;
  }

  canNavigate(from: string, to: string): boolean {
    return true;
  }
}

class ConfirmNavigationStrategy implements NavigationStrategy {
  constructor(private message: string = 'Leave this page?') {}

  navigate(from: string, to: string): void {
    if (window.confirm(this.message)) {
      console.log(`Confirmed navigation: ${from} → ${to}`);
      window.location.href = to;
    } else {
      console.log('Navigation cancelled');
    }
  }

  canNavigate(from: string, to: string): boolean {
    return window.confirm(this.message);
  }
}

class SaveBeforeNavigationStrategy implements NavigationStrategy {
  constructor(private saveCallback: () => Promise<boolean>) {}

  async navigate(from: string, to: string): Promise<void> {
    console.log('Saving before navigation...');
    const saved = await this.saveCallback();
    
    if (saved) {
      console.log(`Navigation after save: ${from} → ${to}`);
      window.location.href = to;
    } else {
      console.log('Save failed, navigation cancelled');
    }
  }

  canNavigate(from: string, to: string): boolean {
    // Synchronous check not possible, use navigate instead
    return false;
  }
}

class RoleBasedNavigationStrategy implements NavigationStrategy {
  constructor(
    private userRole: string,
    private allowedRoutes: Record<string, string[]>
  ) {}

  navigate(from: string, to: string): void {
    if (this.canNavigate(from, to)) {
      console.log(`Role-based navigation: ${from} → ${to}`);
      window.location.href = to;
    } else {
      console.log(`Access denied for ${this.userRole} to ${to}`);
      window.location.href = '/unauthorized';
    }
  }

  canNavigate(from: string, to: string): boolean {
    const allowed = this.allowedRoutes[this.userRole] || [];
    return allowed.includes(to) || allowed.includes('*');
  }
}

// Router context
class Router {
  private strategy: NavigationStrategy;
  private currentRoute: string = '/';

  constructor(strategy: NavigationStrategy) {
    this.strategy = strategy;
  }

  setStrategy(strategy: NavigationStrategy): void {
    this.strategy = strategy;
  }

  navigate(to: string): void {
    this.strategy.navigate(this.currentRoute, to);
    this.currentRoute = to;
  }

  canNavigate(to: string): boolean {
    return this.strategy.canNavigate(this.currentRoute, to);
  }
}

// Usage
const router = new Router(new DirectNavigationStrategy());
router.navigate('/dashboard');

// Form with unsaved changes
router.setStrategy(
  new ConfirmNavigationStrategy('You have unsaved changes. Leave anyway?')
);
router.navigate('/home');

// Auto-save before navigation
router.setStrategy(
  new SaveBeforeNavigationStrategy(async () => {
    // Simulate save
    await new Promise(resolve => setTimeout(resolve, 500));
    return true;
  })
);

// Role-based routing
router.setStrategy(
  new RoleBasedNavigationStrategy('user', {
    admin: ['*'],
    user: ['/home', '/profile', '/settings'],
    guest: ['/home', '/login']
  })
);
```

## Common Mistakes

### 1. Creating Too Many Strategies

```typescript
// BAD - Over-engineering with too many similar strategies
class AddTenStrategy { add(n: number) { return n + 10; } }
class AddTwentyStrategy { add(n: number) { return n + 20; } }
class AddThirtyStrategy { add(n: number) { return n + 30; } }

// GOOD - Parameterized strategy
class AddStrategy {
  constructor(private amount: number) {}
  add(n: number) { return n + this.amount; }
}
```

### 2. Not Using Dependency Injection

```typescript
// BAD - Hard-coded strategy creation
class PaymentProcessor {
  processPayment(type: string) {
    if (type === 'credit') {
      new CreditCardStrategy().pay();
    } else if (type === 'paypal') {
      new PayPalStrategy().pay();
    }
  }
}

// GOOD - Inject strategy
class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}
  
  processPayment() {
    this.strategy.pay();
  }
}
```

### 3. Exposing Strategy Implementation Details

```typescript
// BAD - Client knows too much
const strategy = new SortStrategy();
strategy.setComparator(compareNumbers);
strategy.setPivotSelection('median');
strategy.enableParallel(true);

// GOOD - Encapsulate configuration
const strategy = SortStrategyFactory.createQuickSort({
  comparator: compareNumbers,
  parallel: true
});
```

## Best Practices

1. **Define clear strategy interfaces** with consistent method signatures
2. **Use dependency injection** to provide strategies
3. **Make strategies stateless** when possible
4. **Provide factory methods** for strategy creation
5. **Use strategy pattern** with template method for common operations
6. **Document when to use** each strategy
7. **Consider default strategies** for common cases
8. **Validate strategy compatibility** before use
9. **Use generics** for type-safe strategies
10. **Combine with factory pattern** for strategy selection

## When to Use Strategy Pattern

### Good Use Cases

- Multiple algorithms for the same task (sorting, compression, validation)
- Runtime algorithm selection based on conditions
- Isolating algorithm implementation from usage
- Eliminating conditional statements for algorithm selection
- Family of related algorithms
- Different variants of an algorithm

### When to Avoid

- Only one algorithm is needed
- Algorithms are simple and unlikely to change
- Clients don't need to know about different strategies
- Performance overhead of abstraction is critical

## Interview Questions

### Q1: What's the difference between Strategy and State patterns?

**Answer:** 
- **Strategy**: Encapsulates algorithms, client chooses strategy
- **State**: Encapsulates state-dependent behavior, context changes state automatically

### Q2: How does Strategy pattern promote Open/Closed Principle?

**Answer:** New strategies can be added without modifying existing code. The context class is closed for modification but open for extension through new strategy implementations.

### Q3: When would you use Strategy pattern over a simple if-else?

**Answer:** When:
- Algorithms are complex and should be isolated
- Multiple related algorithms need to be interchangeable
- Algorithm selection logic is complex
- Algorithms may change frequently
- You want to test algorithms independently

### Q4: How do you handle strategy selection in a real application?

**Answer:** Common approaches:
- Factory pattern for creation
- Configuration-based selection
- Runtime conditions (data size, user role, etc.)
- Dependency injection
- Strategy registry pattern

### Q5: What are the performance implications of Strategy pattern?

**Answer:** 
- Pros: Optimized algorithms can be swapped in
- Cons: Extra abstraction layer, more objects created
- Mitigation: Use flyweight pattern for stateless strategies, cache strategy instances

### Q6: How would you combine Strategy with Template Method pattern?

**Answer:**
```typescript
abstract class SortTemplate {
  sort(items: any[]): any[] {
    this.preProcess(items);
    const result = this.doSort(items);
    this.postProcess(result);
    return result;
  }

  protected preProcess(items: any[]): void {}
  protected abstract doSort(items: any[]): any[];
  protected postProcess(items: any[]): void {}
}
```

### Q7: How do you test strategies effectively?

**Answer:** 
- Test each strategy independently
- Test strategy switching in context
- Use mock strategies in context tests
- Test edge cases specific to each strategy
- Verify strategy selection logic

### Q8: What's the relationship between Strategy and Dependency Injection?

**Answer:** DI is a great way to provide strategies to context objects, promoting loose coupling and testability. The strategy interface becomes a dependency that can be injected.

## Key Takeaways

1. Strategy pattern defines a family of interchangeable algorithms
2. Encapsulates algorithms separately from the context that uses them
3. Enables runtime algorithm selection and switching
4. Promotes Open/Closed Principle and Single Responsibility
5. Eliminates conditional logic for algorithm selection
6. Makes algorithms easier to test independently
7. Use dependency injection to provide strategies
8. Combine with factory pattern for strategy creation
9. Consider parameterized strategies over many similar strategies
10. Balance abstraction benefits against complexity costs

## Resources

- [Refactoring Guru: Strategy Pattern](https://refactoring.guru/design-patterns/strategy)
- [Gang of Four Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- [Martin Fowler: Replace Conditional with Polymorphism](https://refactoring.com/catalog/replaceConditionalWithPolymorphism.html)
- [TypeScript Design Patterns](https://www.typescriptlang.org/docs/handbook/patterns.html)
- [Angular Dependency Injection](https://angular.io/guide/dependency-injection)
- [React Design Patterns](https://react.dev/learn/thinking-in-react)
