# SOLID Principles - Complete Reference

## Overview

**SOLID** is an acronym for five design principles that make software designs more understandable, flexible, and maintainable. These principles were introduced by Robert C. Martin (Uncle Bob) and are fundamental to object-oriented programming and design.

---

## The Five SOLID Principles

### 1. **Single Responsibility Principle (SRP)** 📦
A class should have only ONE reason to change - it should have only ONE responsibility.

**Example**: A `User` class should only handle user data, not send emails or log to database.

[→ Read Detailed SRP Guide](./1_Single_Responsibility_Principle.md)

---

### 2. **Open/Closed Principle (OCP)** 🔓
Software entities should be OPEN for extension but CLOSED for modification.

**Example**: Add new payment methods without modifying existing payment processing code.

[→ Read Detailed OCP Guide](./2_Open_Closed_Principle.md)

---

### 3. **Liskov Substitution Principle (LSP)** 🔄
Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.

**Example**: If a function works with `Bird`, it should work with `Sparrow` (subclass) without issues.

[→ Read Detailed LSP Guide](./3_Liskov_Substitution_Principle.md)

---

### 4. **Interface Segregation Principle (ISP)** ✂️
Clients should not be forced to depend on interfaces they don't use.

**Example**: Don't force a `PrinterScanner` interface on a simple `Printer` that can't scan.

[→ Read Detailed ISP Guide](./4_Interface_Segregation_Principle.md)

---

### 5. **Dependency Inversion Principle (DIP)** ⬆️
High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Example**: Business logic should depend on interfaces, not concrete database implementations.

[→ Read Detailed DIP Guide](./5_Dependency_Inversion_Principle.md)

---

## SOLID at a Glance

| Principle | Question | Solution |
|-----------|----------|----------|
| **SRP** | Does this class do too many things? | One class = One responsibility |
| **OCP** | Can I add features without modifying code? | Use abstractions & inheritance |
| **LSP** | Can I substitute parent with child? | Follow contract, don't break behavior |
| **ISP** | Are there unused methods in interface? | Split into smaller interfaces |
| **DIP** | Does business logic depend on details? | Depend on abstractions, not concrete |

---

## Real-World E-Commerce Example

Let's see how ALL SOLID principles work together in a real e-commerce system:

```typescript
// ============================================
// 1. SINGLE RESPONSIBILITY PRINCIPLE (SRP)
// ============================================
// Each class has ONE responsibility

// ✅ GOOD: Separate responsibilities
class Order {
  constructor(
    public orderId: string,
    public items: OrderItem[],
    public customerId: string,
    public status: OrderStatus
  ) {}
  
  calculateTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

class OrderRepository {
  async save(order: Order): Promise<void> {
    // Only responsible for data persistence
    await db.collection('orders').insertOne(order);
  }
  
  async findById(orderId: string): Promise<Order | null> {
    return await db.collection('orders').findOne({ orderId });
  }
}

class OrderNotificationService {
  async sendOrderConfirmation(order: Order): Promise<void> {
    // Only responsible for notifications
    await emailService.send({
      to: order.customerId,
      subject: 'Order Confirmation',
      body: `Your order ${order.orderId} has been confirmed`
    });
  }
}

// ============================================
// 2. OPEN/CLOSED PRINCIPLE (OCP)
// ============================================
// Open for extension, closed for modification

// ✅ GOOD: Use interfaces and strategy pattern
interface PaymentProcessor {
  processPayment(amount: number, orderId: string): Promise<PaymentResult>;
}

class CreditCardProcessor implements PaymentProcessor {
  async processPayment(amount: number, orderId: string): Promise<PaymentResult> {
    // Credit card logic
    return { success: true, transactionId: 'CC-123' };
  }
}

class PayPalProcessor implements PaymentProcessor {
  async processPayment(amount: number, orderId: string): Promise<PaymentResult> {
    // PayPal logic
    return { success: true, transactionId: 'PP-456' };
  }
}

class CryptoProcessor implements PaymentProcessor {
  async processPayment(amount: number, orderId: string): Promise<PaymentResult> {
    // Cryptocurrency logic
    return { success: true, transactionId: 'BTC-789' };
  }
}

// Payment service is CLOSED for modification but OPEN for extension
class PaymentService {
  constructor(private processor: PaymentProcessor) {}
  
  async processOrderPayment(order: Order): Promise<PaymentResult> {
    const total = order.calculateTotal();
    return await this.processor.processPayment(total, order.orderId);
  }
}

// ============================================
// 3. LISKOV SUBSTITUTION PRINCIPLE (LSP)
// ============================================
// Subtypes must be substitutable for their base types

// ✅ GOOD: All discounts follow the same contract
abstract class Discount {
  abstract calculate(total: number): number;
  
  // Contract: discount must be between 0 and total
  protected validateDiscount(discount: number, total: number): number {
    if (discount < 0) return 0;
    if (discount > total) return total;
    return discount;
  }
}

class PercentageDiscount extends Discount {
  constructor(private percentage: number) {
    super();
  }
  
  calculate(total: number): number {
    const discount = total * (this.percentage / 100);
    return this.validateDiscount(discount, total);
  }
}

class FixedAmountDiscount extends Discount {
  constructor(private amount: number) {
    super();
  }
  
  calculate(total: number): number {
    return this.validateDiscount(this.amount, total);
  }
}

// Can use any Discount subclass interchangeably
class OrderPricing {
  applyDiscount(order: Order, discount: Discount): number {
    const total = order.calculateTotal();
    const discountAmount = discount.calculate(total);
    return total - discountAmount;
  }
}

// ============================================
// 4. INTERFACE SEGREGATION PRINCIPLE (ISP)
// ============================================
// Many specific interfaces > One general interface

// ✅ GOOD: Segregated interfaces
interface Readable {
  read(): Promise<any>;
}

interface Writable {
  write(data: any): Promise<void>;
}

interface Deletable {
  delete(id: string): Promise<void>;
}

// Read-only repository only implements Readable
class ProductCatalogRepository implements Readable {
  async read(): Promise<Product[]> {
    return await db.collection('products').find({ available: true }).toArray();
  }
}

// Full CRUD repository implements all three
class AdminProductRepository implements Readable, Writable, Deletable {
  async read(): Promise<Product[]> {
    return await db.collection('products').find().toArray();
  }
  
  async write(product: Product): Promise<void> {
    await db.collection('products').insertOne(product);
  }
  
  async delete(id: string): Promise<void> {
    await db.collection('products').deleteOne({ productId: id });
  }
}

// ============================================
// 5. DEPENDENCY INVERSION PRINCIPLE (DIP)
// ============================================
// Depend on abstractions, not concretions

// ✅ GOOD: High-level module depends on abstraction
interface IOrderRepository {
  save(order: Order): Promise<void>;
  findById(orderId: string): Promise<Order | null>;
}

interface IPaymentGateway {
  charge(amount: number): Promise<string>;
}

interface INotificationService {
  notify(message: string): Promise<void>;
}

// High-level business logic depends on abstractions
class OrderService {
  constructor(
    private orderRepo: IOrderRepository,
    private paymentGateway: IPaymentGateway,
    private notificationService: INotificationService
  ) {}
  
  async placeOrder(order: Order): Promise<void> {
    // Business logic doesn't care about implementation details
    await this.orderRepo.save(order);
    await this.paymentGateway.charge(order.calculateTotal());
    await this.notificationService.notify(`Order ${order.orderId} placed`);
  }
}

// Low-level implementations
class MongoOrderRepository implements IOrderRepository {
  async save(order: Order): Promise<void> {
    await mongoClient.db('shop').collection('orders').insertOne(order);
  }
  
  async findById(orderId: string): Promise<Order | null> {
    return await mongoClient.db('shop').collection('orders').findOne({ orderId });
  }
}

class StripePaymentGateway implements IPaymentGateway {
  async charge(amount: number): Promise<string> {
    const charge = await stripe.charges.create({ amount, currency: 'usd' });
    return charge.id;
  }
}

class EmailNotificationService implements INotificationService {
  async notify(message: string): Promise<void> {
    await emailClient.send({ subject: 'Order Update', body: message });
  }
}

// Dependency injection (Composition Root)
const orderService = new OrderService(
  new MongoOrderRepository(),
  new StripePaymentGateway(),
  new EmailNotificationService()
);
```

---

## Benefits of Following SOLID

### Code Quality Improvements:
1. ✅ **Maintainability** - Easier to update and modify
2. ✅ **Testability** - Simpler to write unit tests
3. ✅ **Flexibility** - Easy to add new features
4. ✅ **Reusability** - Components can be reused
5. ✅ **Readability** - Code is clearer and more understandable
6. ✅ **Scalability** - System grows without becoming complex
7. ✅ **Reduced Bugs** - Fewer side effects from changes

### Team Benefits:
1. ✅ Easier code reviews
2. ✅ Better collaboration
3. ✅ Faster onboarding of new developers
4. ✅ Less technical debt
5. ✅ Predictable codebase

---

## When to Apply SOLID Principles

### ✅ Apply SOLID When:
- Building enterprise applications
- Working on long-term projects
- Team collaboration is required
- Code will be maintained for years
- System needs to be extensible
- High code quality is important

### ⚠️ Can Relax SOLID When:
- Prototyping or proof of concepts
- Simple scripts or one-off tools
- Tight deadlines with throwaway code
- Very small applications (< 100 lines)
- Performance is absolutely critical (rare cases)

**Note**: Even in relaxed scenarios, some SOLID principles (especially SRP) provide value.

---

## Common Anti-Patterns (SOLID Violations)

### 1. God Object (Violates SRP)
```typescript
// ❌ BAD: Class does everything
class Application {
  connectDatabase() { }
  authenticateUser() { }
  processPayment() { }
  sendEmail() { }
  generateReport() { }
  logErrors() { }
  // ... 50 more methods
}
```

### 2. Tight Coupling (Violates DIP & OCP)
```typescript
// ❌ BAD: Tightly coupled to MySQL
class UserService {
  private db = new MySQLConnection(); // Hard dependency!
  
  getUser(id: string) {
    return this.db.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}
```

### 3. Fat Interface (Violates ISP)
```typescript
// ❌ BAD: Forces unused methods
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Robot implements Worker {
  work() { /* OK */ }
  eat() { throw new Error('Robots don\'t eat!'); } // ❌ Forced to implement
  sleep() { throw new Error('Robots don\'t sleep!'); } // ❌ Forced to implement
}
```

### 4. Inheritance Misuse (Violates LSP)
```typescript
// ❌ BAD: Square breaks Rectangle's behavior
class Rectangle {
  constructor(protected width: number, protected height: number) {}
  
  setWidth(width: number) { this.width = width; }
  setHeight(height: number) { this.height = height; }
  
  getArea(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  setWidth(width: number) {
    this.width = width;
    this.height = width; // ❌ Changes both dimensions
  }
  
  setHeight(height: number) {
    this.width = height; // ❌ Changes both dimensions
    this.height = height;
  }
}

// Breaks expectation
const rect: Rectangle = new Square(5, 5);
rect.setWidth(10);
rect.setHeight(5);
console.log(rect.getArea()); // Expected: 50, Actual: 25 ❌
```

---

## SOLID in Different Programming Paradigms

### Object-Oriented (TypeScript/Java)
```typescript
// Traditional OOP approach
interface PaymentMethod {
  process(amount: number): Promise<boolean>;
}

class CreditCard implements PaymentMethod {
  async process(amount: number): Promise<boolean> {
    return true;
  }
}
```

### Functional (TypeScript)
```typescript
// Functional approach (still follows SOLID concepts)
type PaymentProcessor = (amount: number) => Promise<boolean>;

const creditCardProcessor: PaymentProcessor = async (amount) => {
  // Process credit card
  return true;
};

const processPayment = (processor: PaymentProcessor) => async (amount: number) => {
  return await processor(amount);
};
```

### Dependency Injection (NestJS)
```typescript
// Modern framework with built-in DI
@Injectable()
class OrderService {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly paymentService: PaymentService
  ) {}
  
  async createOrder(data: CreateOrderDto): Promise<Order> {
    // Business logic
  }
}
```

---

## Testing with SOLID Principles

```typescript
// Easy to test when following SOLID
describe('OrderService', () => {
  let orderService: OrderService;
  let mockOrderRepo: jest.Mocked<IOrderRepository>;
  let mockPaymentGateway: jest.Mocked<IPaymentGateway>;
  
  beforeEach(() => {
    // Mock dependencies (thanks to DIP!)
    mockOrderRepo = {
      save: jest.fn(),
      findById: jest.fn()
    };
    
    mockPaymentGateway = {
      charge: jest.fn().mockResolvedValue('txn-123')
    };
    
    orderService = new OrderService(
      mockOrderRepo,
      mockPaymentGateway,
      new MockNotificationService()
    );
  });
  
  it('should place order successfully', async () => {
    const order = new Order('ORD-1', [], 'CUST-1', 'PENDING');
    
    await orderService.placeOrder(order);
    
    expect(mockOrderRepo.save).toHaveBeenCalledWith(order);
    expect(mockPaymentGateway.charge).toHaveBeenCalled();
  });
  
  it('should rollback on payment failure', async () => {
    mockPaymentGateway.charge.mockRejectedValue(new Error('Payment failed'));
    
    const order = new Order('ORD-1', [], 'CUST-1', 'PENDING');
    
    await expect(orderService.placeOrder(order)).rejects.toThrow('Payment failed');
  });
});
```

---

## SOLID Principles Checklist

### Before Writing Code:
- [ ] Is this class doing one thing? (SRP)
- [ ] Can I extend without modifying? (OCP)
- [ ] Are my interfaces small and focused? (ISP)
- [ ] Am I depending on abstractions? (DIP)

### During Code Review:
- [ ] Does each class have a single responsibility?
- [ ] Are concrete implementations hidden behind interfaces?
- [ ] Can subclasses replace parent classes safely?
- [ ] Are interfaces minimal and cohesive?
- [ ] Is dependency injection used properly?

### Refactoring Indicators:
- [ ] Class has too many methods (SRP violation)
- [ ] Lots of if/switch statements (OCP violation)
- [ ] Subclasses break parent's behavior (LSP violation)
- [ ] Interfaces with unused methods (ISP violation)
- [ ] Direct instantiation of dependencies (DIP violation)

---

## Real-World System Design Example

```typescript
// Complete system following ALL SOLID principles

// ============================================
// DOMAIN MODELS (SRP - Single Responsibility)
// ============================================
class User {
  constructor(
    public userId: string,
    public email: string,
    public name: string
  ) {}
}

class Product {
  constructor(
    public productId: string,
    public name: string,
    public price: number,
    public stock: number
  ) {}
}

class Order {
  constructor(
    public orderId: string,
    public userId: string,
    public items: OrderItem[],
    public status: OrderStatus
  ) {}
  
  getTotal(): number {
    return this.items.reduce((sum, item) => 
      sum + item.price * item.quantity, 0
    );
  }
}

// ============================================
// ABSTRACTIONS (DIP - Dependency Inversion)
// ============================================
interface IUserRepository {
  findById(userId: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

interface IProductRepository {
  findById(productId: string): Promise<Product | null>;
  updateStock(productId: string, quantity: number): Promise<void>;
}

interface IOrderRepository {
  save(order: Order): Promise<void>;
  findById(orderId: string): Promise<Order | null>;
}

interface IPaymentGateway {
  processPayment(amount: number, userId: string): Promise<PaymentResult>;
}

interface INotificationService {
  sendOrderConfirmation(order: Order): Promise<void>;
}

interface IInventoryService {
  reserveStock(items: OrderItem[]): Promise<boolean>;
  releaseStock(items: OrderItem[]): Promise<void>;
}

// ============================================
// STRATEGY PATTERN (OCP - Open/Closed)
// ============================================
interface DiscountStrategy {
  calculate(total: number): number;
}

class NoDiscount implements DiscountStrategy {
  calculate(total: number): number {
    return 0;
  }
}

class PercentageDiscount implements DiscountStrategy {
  constructor(private percentage: number) {}
  
  calculate(total: number): number {
    return total * (this.percentage / 100);
  }
}

class BuyOneGetOne implements DiscountStrategy {
  calculate(total: number): number {
    return total * 0.5;
  }
}

// ============================================
// BUSINESS LOGIC (Uses all principles)
// ============================================
class OrderService {
  constructor(
    private orderRepo: IOrderRepository,
    private productRepo: IProductRepository,
    private inventoryService: IInventoryService,
    private paymentGateway: IPaymentGateway,
    private notificationService: INotificationService
  ) {}
  
  async placeOrder(
    userId: string,
    items: OrderItem[],
    discountStrategy: DiscountStrategy = new NoDiscount()
  ): Promise<Order> {
    // 1. Create order
    const order = new Order(
      this.generateOrderId(),
      userId,
      items,
      'PENDING'
    );
    
    // 2. Reserve inventory
    const reserved = await this.inventoryService.reserveStock(items);
    if (!reserved) {
      throw new Error('Insufficient stock');
    }
    
    try {
      // 3. Calculate total with discount
      const total = order.getTotal();
      const discount = discountStrategy.calculate(total);
      const finalAmount = total - discount;
      
      // 4. Process payment
      const paymentResult = await this.paymentGateway.processPayment(
        finalAmount,
        userId
      );
      
      if (!paymentResult.success) {
        throw new Error('Payment failed');
      }
      
      // 5. Update order status
      order.status = 'CONFIRMED';
      await this.orderRepo.save(order);
      
      // 6. Send notification
      await this.notificationService.sendOrderConfirmation(order);
      
      return order;
      
    } catch (error) {
      // Rollback: Release reserved stock
      await this.inventoryService.releaseStock(items);
      throw error;
    }
  }
  
  private generateOrderId(): string {
    return `ORD-${Date.now()}`;
  }
}

// ============================================
// CONCRETE IMPLEMENTATIONS
// ============================================
class MongoUserRepository implements IUserRepository {
  async findById(userId: string): Promise<User | null> {
    const doc = await db.collection('users').findOne({ userId });
    return doc ? new User(doc.userId, doc.email, doc.name) : null;
  }
  
  async save(user: User): Promise<void> {
    await db.collection('users').insertOne(user);
  }
}

class StripePaymentGateway implements IPaymentGateway {
  async processPayment(amount: number, userId: string): Promise<PaymentResult> {
    const charge = await stripe.charges.create({
      amount: amount * 100,
      currency: 'usd',
      customer: userId
    });
    
    return {
      success: charge.status === 'succeeded',
      transactionId: charge.id
    };
  }
}

// ============================================
// COMPOSITION ROOT (Dependency Injection)
// ============================================
class ApplicationComposer {
  static composeOrderService(): OrderService {
    const orderRepo = new MongoOrderRepository();
    const productRepo = new MongoProductRepository();
    const inventoryService = new InventoryService(productRepo);
    const paymentGateway = new StripePaymentGateway();
    const notificationService = new EmailNotificationService();
    
    return new OrderService(
      orderRepo,
      productRepo,
      inventoryService,
      paymentGateway,
      notificationService
    );
  }
}

// ============================================
// USAGE
// ============================================
const orderService = ApplicationComposer.composeOrderService();

// Easy to swap discount strategies (OCP)
await orderService.placeOrder(
  'USER-123',
  [{ productId: 'PROD-1', quantity: 2, price: 50 }],
  new PercentageDiscount(10) // 10% off
);

// Or use different strategy
await orderService.placeOrder(
  'USER-456',
  [{ productId: 'PROD-2', quantity: 4, price: 25 }],
  new BuyOneGetOne() // BOGO offer
);
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                   SOLID QUICK REFERENCE                      │
├─────────────────────────────────────────────────────────────┤
│ SINGLE RESPONSIBILITY PRINCIPLE (SRP)                        │
│  ✓ One class = One responsibility                           │
│  ✓ Only one reason to change                                │
│  ✓ High cohesion, low coupling                              │
├─────────────────────────────────────────────────────────────┤
│ OPEN/CLOSED PRINCIPLE (OCP)                                  │
│  ✓ Open for extension                                       │
│  ✓ Closed for modification                                  │
│  ✓ Use interfaces & inheritance                             │
├─────────────────────────────────────────────────────────────┤
│ LISKOV SUBSTITUTION PRINCIPLE (LSP)                          │
│  ✓ Subtypes must be substitutable                           │
│  ✓ Follow parent's contract                                 │
│  ✓ Don't strengthen preconditions                           │
│  ✓ Don't weaken postconditions                              │
├─────────────────────────────────────────────────────────────┤
│ INTERFACE SEGREGATION PRINCIPLE (ISP)                        │
│  ✓ Many specific interfaces                                 │
│  ✓ Don't force unused methods                               │
│  ✓ Client-specific interfaces                               │
├─────────────────────────────────────────────────────────────┤
│ DEPENDENCY INVERSION PRINCIPLE (DIP)                         │
│  ✓ Depend on abstractions                                   │
│  ✓ Not on concretions                                       │
│  ✓ Use dependency injection                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Preparation Checklist

- [ ] Explain each SOLID principle with examples
- [ ] Describe benefits of following SOLID
- [ ] Identify SOLID violations in code
- [ ] Know when to apply each principle
- [ ] Understand dependency injection
- [ ] Explain strategy pattern (OCP)
- [ ] Describe interface vs abstract class (ISP/LSP)
- [ ] Know common anti-patterns
- [ ] Practice refactoring to SOLID
- [ ] Explain testability benefits

---

## Additional Resources

### Detailed Guides
- [1. Single Responsibility Principle](./1_Single_Responsibility_Principle.md)
- [2. Open/Closed Principle](./2_Open_Closed_Principle.md)
- [3. Liskov Substitution Principle](./3_Liskov_Substitution_Principle.md)
- [4. Interface Segregation Principle](./4_Interface_Segregation_Principle.md)
- [5. Dependency Inversion Principle](./5_Dependency_Inversion_Principle.md)

### Related Topics
- Design Patterns (Strategy, Factory, Observer)
- Dependency Injection
- Clean Architecture
- Domain-Driven Design (DDD)
- Test-Driven Development (TDD)

---

## Summary

**SOLID** principles are foundational guidelines for writing maintainable, scalable, and testable code:

1. **Single Responsibility**: One class, one job - keep it focused
2. **Open/Closed**: Extend behavior without modifying existing code
3. **Liskov Substitution**: Subclasses should work wherever parent works
4. **Interface Segregation**: Many small interfaces better than one large
5. **Dependency Inversion**: Depend on abstractions, inject dependencies

**Key Takeaway**: SOLID principles work together to create flexible, maintainable code. They reduce coupling, increase cohesion, and make code easier to test, extend, and modify. While they require more upfront design, they save significant time and effort in the long run, especially for large applications and teams.

**When in doubt**: Start with SRP (single responsibility), use interfaces (DIP), and keep your code open for extension but closed for modification (OCP). The other principles will naturally follow!
