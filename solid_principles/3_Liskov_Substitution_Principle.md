# Liskov Substitution Principle (LSP)

## What is Liskov Substitution Principle?

**Definition**: Objects of a superclass should be replaceable with objects of its subclasses without breaking the application. Subtypes must be substitutable for their base types.

**Simple Explanation**: If class B is a subtype of class A, you should be able to use B anywhere you use A without causing errors or unexpected behavior. Child classes should enhance, not break, parent class behavior.

**Barbara Liskov's Original Definition (1987)**: 
> "If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program."

## Why is it Important?

### Benefits:
1. **Reliable Inheritance** - Subclasses work as expected
2. **Predictable Behavior** - No surprises when using derived classes
3. **Safe Polymorphism** - Can use base class references safely
4. **Maintainable Code** - Easy to add new subclasses
5. **Contract Compliance** - Subclasses honor parent contracts
6. **Reduced Bugs** - Fewer unexpected runtime errors

### Problems Without LSP:
- **Broken Polymorphism** - Can't use subclasses interchangeably
- **Type Checking** - Need instanceof checks everywhere
- **Runtime Errors** - Unexpected exceptions or behavior
- **Fragile Code** - Adding subclass breaks existing code
- **Violated Contracts** - Methods don't work as advertised

---

## Banking Domain Examples

### ❌ BAD Example - Violating LSP

```typescript
console.log('=== LSP Violation - Emirates NBD Banking ===\n');

// Base class - Regular bank account
class BankAccount {
    constructor(
        public accountNumber: string,
        protected balance: number
    ) {}
    
    deposit(amount: number): void {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        this.balance += amount;
        console.log(`Deposited ${amount} AED. New balance: ${this.balance} AED`);
    }
    
    withdraw(amount: number): void {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
        if (this.balance < amount) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
        console.log(`Withdrawn ${amount} AED. New balance: ${this.balance} AED`);
    }
    
    getBalance(): number {
        return this.balance;
    }
}

// ❌ VIOLATION 1: Fixed Deposit Account - Can't withdraw!
class FixedDepositAccount extends BankAccount {
    constructor(accountNumber: string, balance: number, public maturityDate: Date) {
        super(accountNumber, balance);
    }
    
    // ❌ Breaking parent's contract - withdraw should work!
    withdraw(amount: number): void {
        throw new Error('Cannot withdraw from fixed deposit before maturity!');
    }
    
    // Now we need a special method
    withdrawOnMaturity(amount: number): void {
        if (new Date() < this.maturityDate) {
            throw new Error('Account not matured yet');
        }
        super.withdraw(amount);
    }
}

// ❌ VIOLATION 2: ReadOnly Account - Can't deposit!
class ReadOnlyAccount extends BankAccount {
    // ❌ Breaking parent's contract - deposit should work!
    deposit(amount: number): void {
        throw new Error('This is a read-only account. Cannot deposit!');
    }
}

// ❌ VIOLATION 3: Savings Account with unreasonable restrictions
class RestrictedSavingsAccount extends BankAccount {
    private monthlyWithdrawals = 0;
    
    // ❌ Adding preconditions that parent doesn't have
    withdraw(amount: number): void {
        if (this.monthlyWithdrawals >= 3) {
            throw new Error('Monthly withdrawal limit exceeded!');
        }
        if (amount > 5000) {
            throw new Error('Single withdrawal cannot exceed 5000 AED!');
        }
        this.monthlyWithdrawals++;
        super.withdraw(amount);
    }
}

// ❌ VIOLATION 4: Returns different type
class DetailedAccount extends BankAccount {
    // ❌ Return type is incompatible (returning object instead of number)
    getBalance(): any {
        return {
            balance: this.balance,
            currency: 'AED',
            lastUpdated: new Date()
        };
    }
}

// Problems when using these classes:
console.log('Problems with LSP Violations:\n');

function processAccountOperations(account: BankAccount) {
    console.log(`Processing account: ${account.accountNumber}`);
    
    try {
        // These operations SHOULD work for ANY BankAccount
        account.deposit(1000);
        account.withdraw(500);
        
        const balance = account.getBalance();
        console.log(`Final balance: ${balance} AED`);
    } catch (error) {
        console.log(`❌ ERROR: ${(error as Error).message}`);
    }
    console.log();
}

// Regular account - works fine
const regular = new BankAccount('REG-001', 5000);
processAccountOperations(regular);

// Fixed deposit - BREAKS! Can't withdraw
const fixed = new FixedDepositAccount('FD-001', 10000, new Date('2026-01-01'));
processAccountOperations(fixed);

// Read-only - BREAKS! Can't deposit
const readonly = new ReadOnlyAccount('RO-001', 3000);
processAccountOperations(readonly);

// Restricted - MIGHT BREAK! Has hidden constraints
const restricted = new RestrictedSavingsAccount('SAV-001', 8000);
processAccountOperations(restricted);

console.log('='.repeat(70));
console.log('LSP Violations Summary:');
console.log('  ❌ FixedDepositAccount: Cannot substitute - withdraw() throws');
console.log('  ❌ ReadOnlyAccount: Cannot substitute - deposit() throws');
console.log('  ❌ RestrictedSavingsAccount: Adds preconditions parent doesn\'t have');
console.log('  ❌ DetailedAccount: Changes return type of getBalance()');
console.log('  ❌ Need instanceof checks everywhere = BAD DESIGN!');
console.log('='.repeat(70) + '\n');
```

---

### ✅ GOOD Example - Following LSP

```typescript
console.log('=== LSP Compliant Design - Emirates NBD Banking ===\n');

// ============================================================
// SOLUTION: Use Composition and Proper Abstractions
// ============================================================

// 1. Define proper interfaces for different capabilities
interface Depositable {
    deposit(amount: number): void;
}

interface Withdrawable {
    withdraw(amount: number): void;
}

interface BalanceQuery {
    getBalance(): number;
}

// 2. Base class only has common behavior
abstract class Account implements BalanceQuery {
    constructor(
        public readonly accountNumber: string,
        protected balance: number,
        public readonly accountType: string
    ) {}
    
    getBalance(): number {
        return this.balance;
    }
    
    protected validateAmount(amount: number): void {
        if (amount <= 0) {
            throw new Error('Amount must be positive');
        }
    }
    
    protected hassufficientFunds(amount: number): boolean {
        return this.balance >= amount;
    }
}

// ============================================================
// 3. Concrete implementations - Each follows its contract
// ============================================================

// ✅ Savings Account - Fully functional
class SavingsAccount extends Account implements Depositable, Withdrawable {
    constructor(accountNumber: string, balance: number) {
        super(accountNumber, balance, 'SAVINGS');
    }
    
    deposit(amount: number): void {
        this.validateAmount(amount);
        this.balance += amount;
        console.log(`  ✓ Deposited ${amount} AED to ${this.accountNumber}`);
    }
    
    withdraw(amount: number): void {
        this.validateAmount(amount);
        if (!this.hassufficientFunds(amount)) {
            throw new Error('Insufficient funds');
        }
        this.balance -= amount;
        console.log(`  ✓ Withdrawn ${amount} AED from ${this.accountNumber}`);
    }
}

// ✅ Current Account - Can have overdraft
class CurrentAccount extends Account implements Depositable, Withdrawable {
    constructor(
        accountNumber: string,
        balance: number,
        private overdraftLimit: number
    ) {
        super(accountNumber, balance, 'CURRENT');
    }
    
    deposit(amount: number): void {
        this.validateAmount(amount);
        this.balance += amount;
        console.log(`  ✓ Deposited ${amount} AED to ${this.accountNumber}`);
    }
    
    withdraw(amount: number): void {
        this.validateAmount(amount);
        const availableBalance = this.balance + this.overdraftLimit;
        if (amount > availableBalance) {
            throw new Error(`Insufficient funds. Available: ${availableBalance} AED`);
        }
        this.balance -= amount;
        console.log(`  ✓ Withdrawn ${amount} AED from ${this.accountNumber}`);
    }
    
    getAvailableBalance(): number {
        return this.balance + this.overdraftLimit;
    }
}

// ✅ Fixed Deposit - Only query balance, special withdrawal
class FixedDepositAccount extends Account {
    constructor(
        accountNumber: string,
        balance: number,
        public readonly maturityDate: Date,
        public readonly interestRate: number
    ) {
        super(accountNumber, balance, 'FIXED_DEPOSIT');
    }
    
    // ✅ Doesn't implement Withdrawable - clearly states it can't withdraw
    // Has its own specific method with clear constraints
    breakDeposit(): number {
        console.log(`  ⚠️  Breaking fixed deposit ${this.accountNumber}`);
        console.log(`     Penalty applied for early withdrawal`);
        
        const penalty = this.balance * 0.01; // 1% penalty
        const amountAfterPenalty = this.balance - penalty;
        
        console.log(`     Original: ${this.balance} AED`);
        console.log(`     Penalty: ${penalty} AED`);
        console.log(`     Net Amount: ${amountAfterPenalty} AED`);
        
        this.balance = 0;
        return amountAfterPenalty;
    }
    
    withdrawAtMaturity(): number {
        if (new Date() < this.maturityDate) {
            throw new Error('Cannot withdraw before maturity date');
        }
        
        const interest = this.balance * (this.interestRate / 100);
        const totalAmount = this.balance + interest;
        
        console.log(`  ✓ Matured withdrawal from ${this.accountNumber}`);
        console.log(`     Principal: ${this.balance} AED`);
        console.log(`     Interest: ${interest} AED`);
        console.log(`     Total: ${totalAmount} AED`);
        
        this.balance = 0;
        return totalAmount;
    }
}

// ✅ Investment Account - Deposit only (for systematic investment)
class InvestmentAccount extends Account implements Depositable {
    private investments: Array<{date: Date, amount: number}> = [];
    
    constructor(accountNumber: string, balance: number) {
        super(accountNumber, balance, 'INVESTMENT');
    }
    
    deposit(amount: number): void {
        this.validateAmount(amount);
        this.balance += amount;
        this.investments.push({ date: new Date(), amount });
        console.log(`  ✓ Invested ${amount} AED in ${this.accountNumber}`);
    }
    
    // ✅ Doesn't implement Withdrawable
    // Has specific redemption process
    redeemInvestment(units: number): number {
        console.log(`  ✓ Redeeming ${units} units from ${this.accountNumber}`);
        // Redemption logic here
        return units * 100; // Simplified
    }
    
    getInvestmentHistory(): Array<{date: Date, amount: number}> {
        return [...this.investments];
    }
}

// ============================================================
// 4. Use Interfaces for Operations
// ============================================================

class AccountOperations {
    // ✅ Only works with accounts that support deposits
    static processDeposit(account: Depositable, amount: number): void {
        account.deposit(amount);
    }
    
    // ✅ Only works with accounts that support withdrawals
    static processWithdrawal(account: Withdrawable, amount: number): void {
        account.withdraw(amount);
    }
    
    // ✅ Works with ANY account (all have balance)
    static checkBalance(account: BalanceQuery): void {
        console.log(`  Balance: ${account.getBalance()} AED`);
    }
}

// ============================================================
// 5. Type Guards for Safe Operations
// ============================================================

function canDeposit(account: Account): account is SavingsAccount | CurrentAccount | InvestmentAccount {
    return 'deposit' in account;
}

function canWithdraw(account: Account): account is SavingsAccount | CurrentAccount {
    return 'withdraw' in account;
}

function isFixedDeposit(account: Account): account is FixedDepositAccount {
    return account.accountType === 'FIXED_DEPOSIT';
}

// ============================================================
// DEMONSTRATION
// ============================================================

console.log('1️⃣  LSP Compliant Operations:\n');

const savings = new SavingsAccount('SAV-001', 5000);
const current = new CurrentAccount('CUR-001', 3000, 2000);
const fixed = new FixedDepositAccount('FD-001', 10000, new Date('2026-06-01'), 5.5);
const investment = new InvestmentAccount('INV-001', 0);

// ✅ All accounts can check balance (common interface)
console.log('Checking Balances:');
[savings, current, fixed, investment].forEach(account => {
    console.log(`${account.accountType} ${account.accountNumber}: ${account.getBalance()} AED`);
});

console.log('\n2️⃣  Type-Safe Deposits:\n');

// ✅ Compile-time safety - only depositable accounts
const depositableAccounts: Depositable[] = [savings, current, investment];
depositableAccounts.forEach((account) => {
    AccountOperations.processDeposit(account, 1000);
});

console.log('\n3️⃣  Type-Safe Withdrawals:\n');

// ✅ Compile-time safety - only withdrawable accounts
const withdrawableAccounts: Withdrawable[] = [savings, current];
withdrawableAccounts.forEach((account) => {
    AccountOperations.processWithdrawal(account, 500);
});

console.log('\n4️⃣  Special Account Operations:\n');

// ✅ Fixed deposit has its own methods
console.log('Fixed Deposit Operations:');
if (isFixedDeposit(fixed)) {
    // Option 1: Break early (with penalty)
    // const amount = fixed.breakDeposit();
    
    // Option 2: Wait for maturity
    console.log(`  Maturity Date: ${fixed.maturityDate.toLocaleDateString()}`);
    console.log(`  Interest Rate: ${fixed.interestRate}%`);
}

console.log('\n5️⃣  Runtime Type Checking (When Needed):\n');

function processAccountSafely(account: Account): void {
    console.log(`\nProcessing ${account.accountType} ${account.accountNumber}:`);
    
    // Always safe - all accounts have this
    AccountOperations.checkBalance(account);
    
    // Safe deposit if supported
    if (canDeposit(account)) {
        console.log('  ✓ Account supports deposits');
        AccountOperations.processDeposit(account, 500);
    } else {
        console.log('  ⚠️  Account does not support deposits');
    }
    
    // Safe withdrawal if supported
    if (canWithdraw(account)) {
        console.log('  ✓ Account supports withdrawals');
        AccountOperations.processWithdrawal(account, 200);
    } else {
        console.log('  ⚠️  Account does not support withdrawals');
    }
}

[savings, current, fixed, investment].forEach(processAccountSafely);

console.log('\n' + '='.repeat(70) + '\n');
```

---

## LSP Violation Patterns

```typescript
console.log('=== Common LSP Violation Patterns ===\n');

// ❌ PATTERN 1: Throwing exceptions in overridden methods
console.log('1. THROWING EXCEPTIONS:\n');

class Bird {
    fly(): void {
        console.log('Flying...');
    }
}

class Penguin extends Bird {
    // ❌ LSP Violation - parent says birds can fly!
    fly(): void {
        throw new Error('Penguins cannot fly!');
    }
}

console.log('  Problem: Code expecting Bird breaks with Penguin');
console.log('  Solution: Separate FlyingBird and FlightlessBird hierarchies\n');

// ✅ SOLUTION:
abstract class BirdBase {
    eat(): void {
        console.log('Eating...');
    }
}

class FlyingBird extends BirdBase {
    fly(): void {
        console.log('Flying...');
    }
}

class FlightlessBird extends BirdBase {
    swim(): void {
        console.log('Swimming...');
    }
}

class Eagle extends FlyingBird {}
class PenguinFixed extends FlightlessBird {}

console.log('  ✅ Now Eagle and Penguin have appropriate capabilities\n');

// ❌ PATTERN 2: Strengthening preconditions
console.log('2. STRENGTHENING PRECONDITIONS:\n');

class Rectangle {
    constructor(protected width: number, protected height: number) {}
    
    setWidth(width: number): void {
        this.width = width;
    }
    
    setHeight(height: number): void {
        this.height = height;
    }
    
    getArea(): number {
        return this.width * this.height;
    }
}

class Square extends Rectangle {
    // ❌ Strengthening preconditions - requires width === height
    setWidth(width: number): void {
        this.width = width;
        this.height = width; // Forcing constraint!
    }
    
    setHeight(height: number): void {
        this.width = height; // Forcing constraint!
        this.height = height;
    }
}

function testRectangle(rect: Rectangle): void {
    rect.setWidth(5);
    rect.setHeight(10);
    console.log(`  Area: ${rect.getArea()}`); // Expect 50
}

console.log('  Rectangle:');
testRectangle(new Rectangle(0, 0)); // 50 ✓

console.log('  Square:');
testRectangle(new Square(0, 0)); // 100! ❌ LSP violation!

console.log('\n  Problem: Square changes behavior unexpectedly');
console.log('  Solution: Don\'t make Square extend Rectangle\n');

// ❌ PATTERN 3: Weakening postconditions
console.log('3. WEAKENING POSTCONDITIONS:\n');

class PaymentProcessor {
    processPayment(amount: number): boolean {
        // Guarantees: Returns true if payment successful
        console.log(`  Processing payment: ${amount} AED`);
        return true; // Always succeeds in base class
    }
}

class UnreliablePaymentProcessor extends PaymentProcessor {
    processPayment(amount: number): boolean {
        // ❌ Sometimes returns false even for valid payments
        // Weakening the postcondition!
        const success = Math.random() > 0.3;
        console.log(`  Processing payment: ${amount} AED - ${success ? 'Success' : 'Failed'}`);
        return success;
    }
}

console.log('  Problem: Code expecting guaranteed success breaks');
console.log('  Solution: Be explicit about failure possibilities in base\n');

console.log('='.repeat(70) + '\n');
```

---

## Design by Contract

```typescript
console.log('=== Design by Contract & LSP ===\n');

// LSP is closely related to Design by Contract
// Subtypes must respect:
// 1. Preconditions cannot be strengthened
// 2. Postconditions cannot be weakened
// 3. Invariants must be preserved

console.log('Contract Rules:\n');

class AccountWithContract {
    constructor(protected balance: number) {}
    
    /**
     * PRECONDITION: amount > 0
     * POSTCONDITION: balance increases by amount
     * INVARIANT: balance >= 0
     */
    deposit(amount: number): void {
        // Precondition
        if (amount <= 0) {
            throw new Error('Precondition: amount must be positive');
        }
        
        const oldBalance = this.balance;
        this.balance += amount;
        
        // Postcondition
        if (this.balance !== oldBalance + amount) {
            throw new Error('Postcondition violated');
        }
        
        // Invariant
        if (this.balance < 0) {
            throw new Error('Invariant violated: balance cannot be negative');
        }
    }
}

class PremiumAccountWithContract extends AccountWithContract {
    /**
     * ✅ CORRECT: Can weaken preconditions (accept zero)
     * ✅ CORRECT: Can strengthen postconditions (add bonus)
     * ✅ CORRECT: Must maintain invariants
     */
    deposit(amount: number): void {
        // ✅ Weaker precondition (accepting zero is OK)
        if (amount < 0) {
            throw new Error('Amount cannot be negative');
        }
        
        if (amount === 0) {
            return; // Accept zero amount
        }
        
        // ✅ Stronger postcondition (add 1% bonus)
        const bonus = amount * 0.01;
        super.deposit(amount + bonus);
        
        console.log(`  ✓ Deposited ${amount} AED + ${bonus} AED bonus`);
    }
}

console.log('Testing Contract Compliance:');
const premium = new PremiumAccountWithContract(1000);
premium.deposit(1000); // Gets bonus!
console.log(`Final balance: ${premium.getBalance()} AED`);

console.log('\n' + '='.repeat(70) + '\n');
```

---

## Interview Questions & Answers

### Q1: What is Liskov Substitution Principle?
**Answer**: LSP states that objects of a subclass should be able to replace objects of the parent class without breaking the application. If you have `Account` as base class and `SavingsAccount` as subclass, you should be able to use `SavingsAccount` anywhere you use `Account` without unexpected behavior or errors.

### Q2: How do you identify LSP violations?
**Answer**: Look for:
1. **Throwing exceptions** in overridden methods that parent doesn't throw
2. **Type checking** (instanceof) before using subclass
3. **Empty implementations** or methods that do nothing
4. **Strengthened preconditions** (stricter requirements than parent)
5. **Weakened postconditions** (weaker guarantees than parent)
6. **Changed return types** (incompatible with parent)

### Q3: What's the classic Rectangle-Square LSP violation?
**Answer**: Square extending Rectangle violates LSP because:
- Rectangle allows independent width/height changes
- Square forces width === height
- Code like `rect.setWidth(5); rect.setHeight(10);` breaks with Square
- Expected area: 50, actual: 100
**Solution**: Don't make Square inherit from Rectangle; use common Shape interface.

### Q4: How does LSP relate to the "IS-A" relationship?
**Answer**: LSP clarifies "IS-A":
- **Naive**: Penguin IS-A Bird (both are birds)
- **LSP**: Penguin IS-NOT-A FlyingBird (penguin can't fly like birds should)

IS-A should be behavioral, not just conceptual:
- Square IS-A shape ✓
- Square IS-A rectangle (behavior-wise) ✗

### Q5: Give banking example of LSP violation
**Answer**: 
```typescript
// ❌ Violation
class BankAccount {
    withdraw(amount) { /* allows withdrawal */ }
}

class FixedDeposit extends BankAccount {
    withdraw(amount) {
        throw new Error("Can't withdraw!"); // Breaks LSP!
    }
}

// ✅ Solution
class BankAccount { /* common behavior */ }
class WithdrawableAccount extends BankAccount {
    withdraw(amount) { /* ... */ }
}
class FixedDeposit extends BankAccount {
    breakDeposit() { /* specific method */ }
}
```

### Q6: How to fix LSP violations?
**Answer**:
1. **Use interfaces** instead of inheritance
2. **Separate capabilities** (Depositable, Withdrawable interfaces)
3. **Favor composition** over inheritance
4. **Type guards** for safe runtime checks
5. **Design by Contract** - respect preconditions, postconditions
6. **Abstract properly** - only common behavior in base class

### Q7: What's the relationship between LSP and polymorphism?
**Answer**: 
- **Polymorphism** = Using base class reference for derived objects
- **LSP** = Ensures polymorphism works correctly

Without LSP, polymorphism breaks:
```typescript
Account account = getAccount(); // Could be any subtype
account.withdraw(100); // Might throw exception if FixedDeposit!
```

LSP guarantees polymorphism is safe and predictable.

---

## Key Takeaways

### ✅ DO:
- Design inheritance hierarchies carefully
- Use interfaces for different capabilities
- Respect method contracts (pre/post conditions)
- Make subclasses truly substitutable
- Use composition when inheritance doesn't fit
- Write clear contracts in base classes

### ❌ DON'T:
- Throw exceptions in overridden methods unexpectedly
- Strengthen preconditions in subclasses
- Weaken postconditions in subclasses
- Return incompatible types
- Use inheritance for code reuse (use composition)
- Force IS-A relationship when it doesn't fit behaviorally

### Remember:
> "Subclass objects should behave like base class objects"
> 
> If you need to check the actual type before calling a method, you're probably violating LSP!

---

## Summary

**Liskov Substitution Principle** ensures reliable inheritance in banking applications:

- ✅ Use interfaces to define capabilities (Depositable, Withdrawable)
- ✅ Don't force Fixed Deposit to implement withdraw()
- ✅ Subclasses should strengthen, not break, parent behavior
- ✅ Use type guards for safe polymorphism
- ✅ Design by contract - respect pre/post conditions

Following LSP makes your banking application:
- **Reliable** - Subclasses work as expected
- **Maintainable** - Safe to add new account types
- **Polymorphic** - Can use base class references safely
- **Testable** - Test base class contract once
- **Extensible** - Add features without breaking existing code

**The key question**: "Can I use this subclass anywhere I use the base class without surprises?"

If NO, redesign using:
- Separate interfaces for capabilities
- Composition instead of inheritance
- Proper abstractions
- Type guards for safety

**Real-world impact**:
- No runtime surprises when using polymorphism
- Safe to add new account types
- Code works with any account implementation
- Reliable contracts throughout the system
