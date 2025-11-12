# Open/Closed Principle (OCP)

## What is Open/Closed Principle?

**Definition**: Software entities (classes, modules, functions) should be OPEN for extension but CLOSED for modification.

**Simple Explanation**: You should be able to add new functionality WITHOUT changing existing code. Design your code so new features can be added by extending, not by modifying working code.

## Why is it Important?

### Benefits:
1. **Prevents Bugs** - Don't touch working code, less chance of breaking it
2. **Easier Maintenance** - Add features without modifying stable code
3. **Backward Compatibility** - Existing functionality remains unchanged
4. **Reduced Testing** - No need to retest unchanged code
5. **Scalability** - Easy to add new features
6. **Team Safety** - Multiple developers can extend safely

### Problems Without OCP:
- **Fragile Code** - Every new feature risks breaking existing features
- **Regression Bugs** - Changes cause unexpected failures
- **Difficult Testing** - Must retest everything with each change
- **Fear of Change** - Developers afraid to modify code
- **Tight Coupling** - Code components highly dependent on each other

---

## Banking Domain Examples

### ❌ BAD Example - Violating OCP

```typescript
console.log('=== OCP Violation - Emirates NBD Banking ===\n');

// This design violates OCP - must modify class for each new account type!
class AccountInterestCalculator {
    calculateInterest(accountType: string, balance: number): number {
        // Every new account type requires modifying this method!
        if (accountType === 'SAVINGS') {
            return balance * 0.025; // 2.5%
        } else if (accountType === 'CURRENT') {
            return 0; // No interest
        } else if (accountType === 'FIXED_DEPOSIT') {
            return balance * 0.055; // 5.5%
        } else if (accountType === 'PREMIUM_SAVINGS') {
            return balance * 0.035; // 3.5%
        } else if (accountType === 'SENIOR_CITIZEN') {
            return balance * 0.045; // 4.5%
        } else if (accountType === 'STUDENT') {
            return balance * 0.015; // 1.5%
        } else {
            return 0;
        }
        // Need to add CORPORATE account? Must modify this method! ❌
        // Need to add ISLAMIC account? Must modify this method! ❌
    }
}

// Transaction fee calculation violating OCP
class TransactionFeeCalculator {
    calculateFee(transactionType: string, amount: number): number {
        if (transactionType === 'DOMESTIC_TRANSFER') {
            return 5; // Fixed 5 AED
        } else if (transactionType === 'INTERNATIONAL_TRANSFER') {
            return amount * 0.02; // 2% of amount
        } else if (transactionType === 'ATM_WITHDRAWAL') {
            return 2; // 2 AED
        } else if (transactionType === 'CASH_DEPOSIT') {
            return 0; // Free
        } else if (transactionType === 'CHEQUE_PROCESSING') {
            return 10; // 10 AED
        } else {
            return 0;
        }
        // New fee structure? Must modify this method! ❌
    }
}

// Problems with this design:
console.log('Problems with these classes:');
console.log('  ❌ Must modify calculateInterest() for each new account type');
console.log('  ❌ Must modify calculateFee() for each new transaction type');
console.log('  ❌ Growing if-else chains (hard to maintain)');
console.log('  ❌ Risk of breaking existing calculations');
console.log('  ❌ Must retest everything after adding new type');
console.log('  ❌ Violates OCP - closed for modification required!\n');

// Usage
const badCalculator = new AccountInterestCalculator();
console.log('Savings Interest:', badCalculator.calculateInterest('SAVINGS', 10000), 'AED');
console.log('Fixed Deposit Interest:', badCalculator.calculateInterest('FIXED_DEPOSIT', 10000), 'AED');

const feeCalculator = new TransactionFeeCalculator();
console.log('Domestic Transfer Fee:', feeCalculator.calculateFee('DOMESTIC_TRANSFER', 5000), 'AED');
console.log('International Transfer Fee:', feeCalculator.calculateFee('INTERNATIONAL_TRANSFER', 5000), 'AED');

console.log('\n❌ What happens when we add new account types?');
console.log('  • Corporate Account with 4% interest?');
console.log('  • Islamic Account with profit-sharing?');
console.log('  • VIP Account with tiered interest?');
console.log('  👉 Must modify the existing calculateInterest() method!');
console.log('  👉 Risk breaking existing account calculations!');
console.log('  👉 Must retest all account types!\n');

console.log('='.repeat(70) + '\n');
```

---

### ✅ GOOD Example - Following OCP

```typescript
console.log('=== OCP Compliant Design - Emirates NBD Banking ===\n');

// ============================================================
// 1. INTEREST CALCULATION - Open for Extension
// ============================================================

// Strategy interface - defines the contract
interface InterestStrategy {
    calculateInterest(balance: number): number;
    getAccountType(): string;
}

// Concrete strategies - can add NEW ones without modifying existing code
class SavingsInterestStrategy implements InterestStrategy {
    calculateInterest(balance: number): number {
        return balance * 0.025; // 2.5% interest
    }
    
    getAccountType(): string {
        return 'Savings Account';
    }
}

class FixedDepositInterestStrategy implements InterestStrategy {
    constructor(private months: number) {}
    
    calculateInterest(balance: number): number {
        const baseRate = 0.055; // 5.5% base
        const bonus = this.months >= 12 ? 0.005 : 0; // +0.5% if 12+ months
        return balance * (baseRate + bonus);
    }
    
    getAccountType(): string {
        return `Fixed Deposit (${this.months} months)`;
    }
}

class CurrentAccountInterestStrategy implements InterestStrategy {
    calculateInterest(balance: number): number {
        return 0; // No interest on current accounts
    }
    
    getAccountType(): string {
        return 'Current Account';
    }
}

class PremiumSavingsInterestStrategy implements InterestStrategy {
    calculateInterest(balance: number): number {
        // Tiered interest rates
        if (balance >= 100000) {
            return balance * 0.045; // 4.5% for high balance
        } else if (balance >= 50000) {
            return balance * 0.035; // 3.5% for medium balance
        } else {
            return balance * 0.025; // 2.5% for lower balance
        }
    }
    
    getAccountType(): string {
        return 'Premium Savings Account';
    }
}

// ✅ NEW ACCOUNT TYPE - Just add new class, don't modify existing code!
class SeniorCitizenInterestStrategy implements InterestStrategy {
    calculateInterest(balance: number): number {
        return balance * 0.045; // 4.5% for senior citizens
    }
    
    getAccountType(): string {
        return 'Senior Citizen Account';
    }
}

// ✅ NEW ACCOUNT TYPE - Islamic banking
class IslamicProfitSharingStrategy implements InterestStrategy {
    constructor(private profitRatio: number = 0.03) {}
    
    calculateInterest(balance: number): number {
        // Profit sharing instead of interest
        return balance * this.profitRatio;
    }
    
    getAccountType(): string {
        return 'Islamic Profit-Sharing Account';
    }
}

// ✅ NEW ACCOUNT TYPE - Corporate account
class CorporateAccountInterestStrategy implements InterestStrategy {
    calculateInterest(balance: number): number {
        // Negotiated rates based on balance
        if (balance >= 1000000) {
            return balance * 0.04; // 4% for 1M+
        } else if (balance >= 500000) {
            return balance * 0.035; // 3.5% for 500K+
        } else {
            return balance * 0.03; // 3% standard
        }
    }
    
    getAccountType(): string {
        return 'Corporate Account';
    }
}

// Account class - Open for extension, Closed for modification
class BankAccount {
    constructor(
        public accountNumber: string,
        public balance: number,
        private interestStrategy: InterestStrategy
    ) {}
    
    calculateInterest(): number {
        return this.interestStrategy.calculateInterest(this.balance);
    }
    
    getAccountInfo(): string {
        return `${this.interestStrategy.getAccountType()} - ${this.accountNumber}`;
    }
    
    // Can change strategy at runtime!
    setInterestStrategy(strategy: InterestStrategy): void {
        this.interestStrategy = strategy;
    }
}

// ============================================================
// 2. TRANSACTION FEE CALCULATION - Open for Extension
// ============================================================

interface FeeCalculationStrategy {
    calculateFee(amount: number): number;
    getTransactionType(): string;
}

class DomesticTransferFeeStrategy implements FeeCalculationStrategy {
    calculateFee(amount: number): number {
        return 5; // Flat fee
    }
    
    getTransactionType(): string {
        return 'Domestic Transfer';
    }
}

class InternationalTransferFeeStrategy implements FeeCalculationStrategy {
    calculateFee(amount: number): number {
        const percentage = 0.02; // 2%
        const minFee = 25;
        const maxFee = 200;
        
        const calculatedFee = amount * percentage;
        return Math.min(Math.max(calculatedFee, minFee), maxFee);
    }
    
    getTransactionType(): string {
        return 'International Transfer';
    }
}

class ATMWithdrawalFeeStrategy implements FeeCalculationStrategy {
    constructor(private isOwnBank: boolean) {}
    
    calculateFee(amount: number): number {
        return this.isOwnBank ? 0 : 2; // Free for own bank ATMs
    }
    
    getTransactionType(): string {
        return this.isOwnBank ? 'Own Bank ATM' : 'Other Bank ATM';
    }
}

// ✅ NEW FEE TYPE - Premium member gets discount
class PremiumMemberTransferFeeStrategy implements FeeCalculationStrategy {
    constructor(private baseFeeStrategy: FeeCalculationStrategy) {}
    
    calculateFee(amount: number): number {
        const baseFee = this.baseFeeStrategy.calculateFee(amount);
        return baseFee * 0.5; // 50% discount for premium members
    }
    
    getTransactionType(): string {
        return `Premium - ${this.baseFeeStrategy.getTransactionType()}`;
    }
}

// ✅ NEW FEE TYPE - Express transfer with higher fee
class ExpressTransferFeeStrategy implements FeeCalculationStrategy {
    constructor(private baseFeeStrategy: FeeCalculationStrategy) {}
    
    calculateFee(amount: number): number {
        const baseFee = this.baseFeeStrategy.calculateFee(amount);
        return baseFee + 20; // Additional 20 AED for express service
    }
    
    getTransactionType(): string {
        return `Express - ${this.baseFeeStrategy.getTransactionType()}`;
    }
}

class Transaction {
    constructor(
        public transactionId: string,
        public amount: number,
        private feeStrategy: FeeCalculationStrategy
    ) {}
    
    calculateTotalCost(): number {
        return this.amount + this.feeStrategy.calculateFee(this.amount);
    }
    
    getFeeBreakdown(): string {
        const fee = this.feeStrategy.calculateFee(this.amount);
        return `Amount: ${this.amount} AED + Fee: ${fee} AED = Total: ${this.calculateTotalCost()} AED`;
    }
}

// ============================================================
// 3. NOTIFICATION SYSTEM - Open for Extension
// ============================================================

interface NotificationMethod {
    send(recipient: string, message: string): void;
    getChannelName(): string;
}

class EmailNotification implements NotificationMethod {
    send(email: string, message: string): void {
        console.log(`  📧 Email to ${email}: ${message}`);
    }
    
    getChannelName(): string {
        return 'Email';
    }
}

class SMSNotification implements NotificationMethod {
    send(phone: string, message: string): void {
        console.log(`  📱 SMS to ${phone}: ${message}`);
    }
    
    getChannelName(): string {
        return 'SMS';
    }
}

// ✅ NEW NOTIFICATION CHANNEL - Just add new class!
class WhatsAppNotification implements NotificationMethod {
    send(phone: string, message: string): void {
        console.log(`  💬 WhatsApp to ${phone}: ${message}`);
    }
    
    getChannelName(): string {
        return 'WhatsApp';
    }
}

// ✅ NEW NOTIFICATION CHANNEL - Push notifications
class PushNotification implements NotificationMethod {
    send(deviceId: string, message: string): void {
        console.log(`  🔔 Push to device ${deviceId}: ${message}`);
    }
    
    getChannelName(): string {
        return 'Push Notification';
    }
}

// Notification service - doesn't need modification for new channels
class NotificationService {
    private methods: NotificationMethod[] = [];
    
    addNotificationMethod(method: NotificationMethod): void {
        this.methods.push(method);
        console.log(`  ✓ Added notification channel: ${method.getChannelName()}`);
    }
    
    notifyCustomer(recipient: string, message: string): void {
        this.methods.forEach(method => method.send(recipient, message));
    }
}

// ============================================================
// DEMONSTRATION
// ============================================================

console.log('1️⃣  Interest Calculation - Multiple Account Types:\n');

const accounts = [
    new BankAccount('SAV-001', 10000, new SavingsInterestStrategy()),
    new BankAccount('FD-001', 50000, new FixedDepositInterestStrategy(12)),
    new BankAccount('CUR-001', 25000, new CurrentAccountInterestStrategy()),
    new BankAccount('PREM-001', 75000, new PremiumSavingsInterestStrategy()),
    new BankAccount('SR-001', 30000, new SeniorCitizenInterestStrategy()),
    new BankAccount('ISL-001', 40000, new IslamicProfitSharingStrategy()),
    new BankAccount('CORP-001', 1500000, new CorporateAccountInterestStrategy())
];

accounts.forEach(account => {
    const interest = account.calculateInterest();
    console.log(`${account.getAccountInfo()}`);
    console.log(`  Balance: ${account.balance} AED → Interest: ${interest.toFixed(2)} AED`);
});

console.log('\n' + '='.repeat(70) + '\n');

console.log('2️⃣  Transaction Fee Calculation:\n');

const transactions = [
    new Transaction('TXN-001', 5000, new DomesticTransferFeeStrategy()),
    new Transaction('TXN-002', 10000, new InternationalTransferFeeStrategy()),
    new Transaction('TXN-003', 2000, new ATMWithdrawalFeeStrategy(true)),
    new Transaction('TXN-004', 2000, new ATMWithdrawalFeeStrategy(false)),
    new Transaction('TXN-005', 5000, new PremiumMemberTransferFeeStrategy(new DomesticTransferFeeStrategy())),
    new Transaction('TXN-006', 10000, new ExpressTransferFeeStrategy(new InternationalTransferFeeStrategy()))
];

transactions.forEach(txn => {
    console.log(`${txn.transactionId}: ${txn.getFeeBreakdown()}`);
});

console.log('\n' + '='.repeat(70) + '\n');

console.log('3️⃣  Notification System - Multiple Channels:\n');

const notificationService = new NotificationService();
notificationService.addNotificationMethod(new EmailNotification());
notificationService.addNotificationMethod(new SMSNotification());
notificationService.addNotificationMethod(new WhatsAppNotification());
notificationService.addNotificationMethod(new PushNotification());

console.log('\nSending notification:');
notificationService.notifyCustomer(
    'customer@example.com',
    'Your transaction of 5000 AED was successful!'
);

console.log('\n' + '='.repeat(70) + '\n');
```

---

## Benefits Comparison

```typescript
console.log('=== OCP Benefits Comparison ===\n');

console.log('WITHOUT OCP (Bad Design):');
console.log('  ❌ Modify calculateInterest() for each new account type');
console.log('  ❌ Growing if-else chains (10+ conditions)');
console.log('  ❌ Risk breaking existing calculations');
console.log('  ❌ Must retest all account types');
console.log('  ❌ Code becomes harder to read and maintain');
console.log('  ❌ Merge conflicts when multiple devs add features\n');

console.log('WITH OCP (Good Design):');
console.log('  ✅ Add new account type = Create new strategy class');
console.log('  ✅ No if-else chains');
console.log('  ✅ Existing code untouched (no breaking risk)');
console.log('  ✅ Only test new strategy class');
console.log('  ✅ Code remains clean and readable');
console.log('  ✅ Parallel development without conflicts\n');

console.log('Real-World Scenarios:\n');

console.log('Scenario 1: Add Student Account (0.5% interest)');
console.log('  Without OCP:');
console.log('    1. Modify calculateInterest() method ❌');
console.log('    2. Add new if-else condition ❌');
console.log('    3. Risk typo breaking other accounts ❌');
console.log('    4. Retest all 7 existing account types ❌');
console.log('  With OCP:');
console.log('    1. Create StudentInterestStrategy class ✅');
console.log('    2. Implement calculateInterest() ✅');
console.log('    3. Existing code unchanged ✅');
console.log('    4. Only test new strategy ✅\n');

console.log('Scenario 2: Add Telegram Notifications');
console.log('  Without OCP:');
console.log('    1. Modify NotificationService class ❌');
console.log('    2. Add telegram-specific logic ❌');
console.log('    3. Risk breaking email/SMS ❌');
console.log('  With OCP:');
console.log('    1. Create TelegramNotification class ✅');
console.log('    2. Implement send() method ✅');
console.log('    3. Add to notification service ✅\n');

console.log('Scenario 3: Change International Transfer Fees');
console.log('  Without OCP:');
console.log('    1. Find and modify if-else block ❌');
console.log('    2. Hope you don\'t break other fees ❌');
console.log('  With OCP:');
console.log('    1. Modify only InternationalTransferFeeStrategy ✅');
console.log('    2. Other fees guaranteed safe ✅\n');

console.log('='.repeat(70) + '\n');
```

---

## Design Patterns Supporting OCP

```typescript
console.log('=== Design Patterns that Enable OCP ===\n');

// 1. Strategy Pattern - Most common for OCP
console.log('1. STRATEGY PATTERN:');
console.log('   • Define family of algorithms');
console.log('   • Make them interchangeable');
console.log('   • Example: Different interest calculation strategies\n');

// 2. Template Method Pattern
console.log('2. TEMPLATE METHOD PATTERN:');
console.log('   • Define skeleton of algorithm');
console.log('   • Let subclasses override specific steps');
console.log('   • Example: Account opening process\n');

abstract class AccountOpeningProcess {
    // Template method - defines the workflow
    public openAccount(): void {
        this.verifyCustomer();
        this.createAccount();
        this.setInitialSettings();
        this.sendWelcomeMessage();
    }
    
    protected abstract verifyCustomer(): void;
    protected abstract setInitialSettings(): void;
    
    protected createAccount(): void {
        console.log('  • Creating account in database');
    }
    
    protected sendWelcomeMessage(): void {
        console.log('  • Sending welcome message');
    }
}

class SavingsAccountOpening extends AccountOpeningProcess {
    protected verifyCustomer(): void {
        console.log('  • Basic verification for savings account');
    }
    
    protected setInitialSettings(): void {
        console.log('  • Setting 2.5% interest rate');
        console.log('  • Setting minimum balance: 1000 AED');
    }
}

class CorporateAccountOpening extends AccountOpeningProcess {
    protected verifyCustomer(): void {
        console.log('  • Business verification');
        console.log('  • Trade license verification');
    }
    
    protected setInitialSettings(): void {
        console.log('  • Setting negotiated interest rate');
        console.log('  • Setting credit limit');
        console.log('  • Assigning relationship manager');
    }
}

console.log('Opening Savings Account:');
new SavingsAccountOpening().openAccount();

console.log('\nOpening Corporate Account:');
new CorporateAccountOpening().openAccount();

console.log('\n' + '='.repeat(70) + '\n');

// 3. Decorator Pattern
console.log('3. DECORATOR PATTERN:');
console.log('   • Add responsibilities dynamically');
console.log('   • Wrap objects with new behavior');
console.log('   • Example: Adding features to accounts\n');

interface AccountFeature {
    getDescription(): string;
    getCost(): number;
}

class BasicAccount implements AccountFeature {
    getDescription(): string {
        return 'Basic Account';
    }
    
    getCost(): number {
        return 0; // Free
    }
}

class OverdraftProtection implements AccountFeature {
    constructor(private account: AccountFeature) {}
    
    getDescription(): string {
        return this.account.getDescription() + ' + Overdraft Protection';
    }
    
    getCost(): number {
        return this.account.getCost() + 15; // 15 AED/month
    }
}

class InsuranceCoverage implements AccountFeature {
    constructor(private account: AccountFeature) {}
    
    getDescription(): string {
        return this.account.getDescription() + ' + Insurance';
    }
    
    getCost(): number {
        return this.account.getCost() + 25; // 25 AED/month
    }
}

class PremiumCard implements AccountFeature {
    constructor(private account: AccountFeature) {}
    
    getDescription(): string {
        return this.account.getDescription() + ' + Premium Card';
    }
    
    getCost(): number {
        return this.account.getCost() + 50; // 50 AED/month
    }
}

// Build account with features dynamically
let account: AccountFeature = new BasicAccount();
console.log(`${account.getDescription()}: ${account.getCost()} AED/month`);

account = new OverdraftProtection(account);
console.log(`${account.getDescription()}: ${account.getCost()} AED/month`);

account = new InsuranceCoverage(account);
console.log(`${account.getDescription()}: ${account.getCost()} AED/month`);

account = new PremiumCard(account);
console.log(`${account.getDescription()}: ${account.getCost()} AED/month`);

console.log('\n✅ Added features without modifying BasicAccount class!');

console.log('\n' + '='.repeat(70) + '\n');
```

---

## Interview Questions & Answers

### Q1: What is Open/Closed Principle?
**Answer**: OCP states that software entities should be OPEN for extension but CLOSED for modification. You should be able to add new functionality by extending the code (adding new classes) rather than modifying existing code. This prevents bugs in stable code and makes systems more maintainable.

### Q2: How do you achieve OCP in practice?
**Answer**: 
1. **Use abstractions** (interfaces/abstract classes)
2. **Dependency Injection** - depend on abstractions, not concrete classes
3. **Strategy Pattern** - encapsulate algorithms in separate classes
4. **Polymorphism** - runtime selection of behavior
5. **Composition over Inheritance** - build features by composing objects

### Q3: Give a banking example where OCP is crucial
**Answer**: Interest calculation for different account types. Instead of one method with multiple if-else:
```typescript
// ❌ Closed for extension, requires modification
if (type === 'SAVINGS') return balance * 0.025;
else if (type === 'FIXED') return balance * 0.055;

// ✅ Open for extension, closed for modification
interface InterestStrategy {
    calculate(balance: number): number;
}
class SavingsStrategy implements InterestStrategy { ... }
class FixedStrategy implements InterestStrategy { ... }
```

### Q4: What's the difference between OCP and just adding new code?
**Answer**: 
- **Adding new code** = Creating new methods/classes
- **OCP** = Adding new behavior WITHOUT modifying existing tested code

OCP ensures:
- Existing code stays stable (no regression bugs)
- New features don't break old features
- No need to retest existing functionality

### Q5: How does OCP relate to the Strategy pattern?
**Answer**: Strategy pattern is the PRIMARY way to implement OCP:
- Define interface for behavior (InterestStrategy)
- Create concrete strategies (SavingsStrategy, FixedStrategy)
- Add new strategies without modifying existing code
- Context class works with interface, not concrete classes

This makes the system open for extension (new strategies) but closed for modification (existing strategies unchanged).

### Q6: What are signs of OCP violation?
**Answer**:
1. **Long if-else or switch statements** based on type
2. **Modifying same method** for each new feature
3. **Fear of adding features** (might break existing code)
4. **Frequent regression bugs** after adding features
5. **Cannot add features** without source code access

### Q7: Can OCP be applied to existing legacy code?
**Answer**: Yes, through **refactoring**:
1. Identify variation points (what changes frequently)
2. Extract interface/abstract class
3. Create concrete implementations
4. Replace if-else with polymorphism
5. Use dependency injection

Example: Refactor AccountManager with if-else to strategy-based design.

---

## Key Takeaways

### ✅ DO:
- Design with extension in mind
- Use interfaces and abstract classes
- Apply Strategy, Template Method, Decorator patterns
- Think "How would I add new feature without changing code?"
- Use dependency injection
- Write modular, composable code

### ❌ DON'T:
- Use type checking (instanceof, typeof) to vary behavior
- Write long if-else or switch on type
- Modify existing methods to add features
- Hardcode dependencies
- Fear creating new classes

### Remember:
> "Software should be open for extension, closed for modification"
> 
> If adding a new account type requires modifying calculateInterest(), you're violating OCP!

---

## Summary

**Open/Closed Principle** ensures stable, extensible banking applications:

- ✅ Add new account types without touching existing code
- ✅ Add new notification channels by creating new classes
- ✅ Extend fee calculations without breaking existing fees
- ✅ Use Strategy pattern for interchangeable algorithms
- ✅ Apply Decorator pattern for optional features

Following OCP makes your banking application:
- **Safer** - Changes don't break existing features
- **More maintainable** - Easy to add features
- **More testable** - Only test new code
- **More flexible** - Easy to customize
- **Team-friendly** - Parallel development safe

**The key question**: "Can I add this feature by creating new code, or must I modify existing code?"
If you must modify existing code, find a way to make it extensible (extract interface, use strategy, etc.)!

**Real-world impact**:
- Add 10 new account types → 10 new classes (no risk to existing)
- Add 5 notification channels → 5 new classes (zero regression risk)
- Modify fee structure → Only touch one strategy class (safe!)
