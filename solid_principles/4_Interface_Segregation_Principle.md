# Interface Segregation Principle (ISP)

## What is Interface Segregation Principle?

**Definition**: No client should be forced to depend on methods it does not use. Instead of one fat interface, create multiple smaller, specific interfaces.

**Simple Explanation**: Don't force classes to implement methods they don't need. Split large interfaces into smaller, focused ones so clients only depend on what they actually use.

**Robert C. Martin's Definition**:
> "Clients should not be forced to depend upon interfaces that they do not use."

## Why is it Important?

### Benefits:
1. **No Unnecessary Dependencies** - Classes only implement what they need
2. **Easier to Understand** - Small, focused interfaces are clear
3. **Better Maintainability** - Changes don't affect unrelated clients
4. **Flexible Design** - Mix and match interfaces as needed
5. **Prevents Interface Pollution** - No bloated interfaces
6. **Easier Testing** - Mock only what's needed

### Problems Without ISP:
- **Fat Interfaces** - Huge interfaces with many methods
- **Dummy Implementations** - Empty methods or throwing exceptions
- **Tight Coupling** - Clients depend on methods they don't use
- **Difficult to Implement** - Classes forced to implement irrelevant methods
- **Ripple Effects** - Interface changes affect all implementers

---

## Banking Domain Examples

### ❌ BAD Example - Violating ISP

```typescript
console.log('=== ISP Violation - Emirates NBD Banking ===\n');

// ❌ FAT INTERFACE - Trying to cover all account operations
interface IBankAccountOperations {
    // Balance operations
    getBalance(): number;
    deposit(amount: number): void;
    withdraw(amount: number): void;
    
    // Interest operations
    calculateInterest(): number;
    applyInterest(): void;
    
    // Loan operations
    getLoanBalance(): number;
    payLoan(amount: number): void;
    calculateLoanEMI(): number;
    
    // Investment operations
    buyUnits(amount: number): void;
    sellUnits(units: number): void;
    getPortfolioValue(): number;
    
    // Card operations
    blockCard(): void;
    unblockCard(): void;
    setCardLimit(limit: number): void;
    
    // Overdraft operations
    getOverdraftLimit(): number;
    setOverdraftLimit(limit: number): void;
    
    // Fixed deposit operations
    breakDeposit(): number;
    getMaturityDate(): Date;
    extendMaturity(months: number): void;
}

// Now let's try to implement different account types...

// ❌ PROBLEM 1: Savings Account doesn't need loan/investment/card operations!
class ViolatedSavingsAccount implements IBankAccountOperations {
    constructor(private balance: number) {}
    
    // ✓ Needs these
    getBalance(): number { return this.balance; }
    deposit(amount: number): void { this.balance += amount; }
    withdraw(amount: number): void { this.balance -= amount; }
    calculateInterest(): number { return this.balance * 0.03; }
    applyInterest(): void { this.balance += this.calculateInterest(); }
    
    // ❌ Doesn't need these - forced to implement!
    getLoanBalance(): number {
        throw new Error('Savings account does not have loan!');
    }
    payLoan(amount: number): void {
        throw new Error('Savings account does not have loan!');
    }
    calculateLoanEMI(): number {
        throw new Error('Savings account does not have loan!');
    }
    buyUnits(amount: number): void {
        throw new Error('Savings account is not investment account!');
    }
    sellUnits(units: number): void {
        throw new Error('Savings account is not investment account!');
    }
    getPortfolioValue(): number {
        throw new Error('Savings account is not investment account!');
    }
    blockCard(): void {
        throw new Error('Savings account does not have card operations!');
    }
    unblockCard(): void {
        throw new Error('Savings account does not have card operations!');
    }
    setCardLimit(limit: number): void {
        throw new Error('Savings account does not have card operations!');
    }
    getOverdraftLimit(): number {
        throw new Error('Savings account does not have overdraft!');
    }
    setOverdraftLimit(limit: number): void {
        throw new Error('Savings account does not have overdraft!');
    }
    breakDeposit(): number {
        throw new Error('Not a fixed deposit!');
    }
    getMaturityDate(): Date {
        throw new Error('Not a fixed deposit!');
    }
    extendMaturity(months: number): void {
        throw new Error('Not a fixed deposit!');
    }
}

// ❌ PROBLEM 2: Fixed Deposit doesn't even allow deposit/withdraw!
class ViolatedFixedDepositAccount implements IBankAccountOperations {
    constructor(
        private balance: number,
        private maturityDate: Date
    ) {}
    
    // ✓ Needs only these
    getBalance(): number { return this.balance; }
    getMaturityDate(): Date { return this.maturityDate; }
    breakDeposit(): number {
        const penalty = this.balance * 0.01;
        return this.balance - penalty;
    }
    calculateInterest(): number { return this.balance * 0.06; }
    applyInterest(): void { this.balance += this.calculateInterest(); }
    extendMaturity(months: number): void {
        this.maturityDate = new Date(this.maturityDate.getTime() + months * 30 * 24 * 60 * 60 * 1000);
    }
    
    // ❌ Forced to implement 11 irrelevant methods!
    deposit(amount: number): void {
        throw new Error('Cannot deposit to fixed deposit!');
    }
    withdraw(amount: number): void {
        throw new Error('Cannot withdraw from fixed deposit!');
    }
    getLoanBalance(): number { throw new Error('Not applicable'); }
    payLoan(amount: number): void { throw new Error('Not applicable'); }
    calculateLoanEMI(): number { throw new Error('Not applicable'); }
    buyUnits(amount: number): void { throw new Error('Not applicable'); }
    sellUnits(units: number): void { throw new Error('Not applicable'); }
    getPortfolioValue(): number { throw new Error('Not applicable'); }
    blockCard(): void { throw new Error('Not applicable'); }
    unblockCard(): void { throw new Error('Not applicable'); }
    setCardLimit(limit: number): void { throw new Error('Not applicable'); }
    getOverdraftLimit(): number { throw new Error('Not applicable'); }
    setOverdraftLimit(limit: number): void { throw new Error('Not applicable'); }
}

// ❌ PROBLEM 3: Credit Card doesn't need most of these!
class ViolatedCreditCardAccount implements IBankAccountOperations {
    constructor(
        private outstandingBalance: number,
        private creditLimit: number
    ) {}
    
    // ✓ Needs only these
    getBalance(): number { return -this.outstandingBalance; }
    payLoan(amount: number): void { this.outstandingBalance -= amount; }
    getLoanBalance(): number { return this.outstandingBalance; }
    calculateLoanEMI(): number { return this.outstandingBalance / 12; }
    blockCard(): void { console.log('Card blocked'); }
    unblockCard(): void { console.log('Card unblocked'); }
    setCardLimit(limit: number): void { this.creditLimit = limit; }
    
    // ❌ Forced to implement 10 irrelevant methods!
    deposit(amount: number): void { throw new Error('Cannot deposit to credit card'); }
    withdraw(amount: number): void { throw new Error('Use card for purchases'); }
    calculateInterest(): number { throw new Error('Not applicable'); }
    applyInterest(): void { throw new Error('Not applicable'); }
    buyUnits(amount: number): void { throw new Error('Not applicable'); }
    sellUnits(units: number): void { throw new Error('Not applicable'); }
    getPortfolioValue(): number { throw new Error('Not applicable'); }
    getOverdraftLimit(): number { throw new Error('Not applicable'); }
    setOverdraftLimit(limit: number): void { throw new Error('Not applicable'); }
    breakDeposit(): number { throw new Error('Not applicable'); }
    getMaturityDate(): Date { throw new Error('Not applicable'); }
    extendMaturity(months: number): void { throw new Error('Not applicable'); }
}

console.log('Problems with Fat Interface:\n');
console.log('  ❌ Savings: Implements 17 methods, needs only 5 (12 throw errors)');
console.log('  ❌ Fixed Deposit: Implements 17 methods, needs only 6 (11 throw errors)');
console.log('  ❌ Credit Card: Implements 17 methods, needs only 7 (10 throw errors)');
console.log('  ❌ Lots of dummy implementations throwing exceptions');
console.log('  ❌ Hard to understand what each account actually supports');
console.log('  ❌ Changes to interface affect ALL account types');
console.log('  ❌ Tight coupling - everything depends on everything');
console.log('\n' + '='.repeat(70) + '\n');
```

---

### ✅ GOOD Example - Following ISP

```typescript
console.log('=== ISP Compliant Design - Emirates NBD Banking ===\n');

// ============================================================
// SOLUTION: Split into focused, role-specific interfaces
// ============================================================

// 1. Core interfaces - Everyone needs balance
interface IBalanceQuery {
    getBalance(): number;
    getAccountNumber(): string;
}

// 2. Transaction interfaces - For transactional accounts
interface IDepositable {
    deposit(amount: number): void;
}

interface IWithdrawable {
    withdraw(amount: number): void;
}

// 3. Interest interfaces - For interest-bearing accounts
interface IInterestBearing {
    calculateInterest(): number;
    applyInterest(): void;
    getInterestRate(): number;
}

// 4. Loan interfaces - For loan/credit accounts
interface ILoanAccount {
    getLoanBalance(): number;
    payLoan(amount: number): void;
    calculateEMI(): number;
}

// 5. Investment interfaces - For investment accounts
interface IInvestmentAccount {
    buyUnits(amount: number): void;
    sellUnits(units: number): void;
    getPortfolioValue(): number;
    getUnitsHeld(): number;
}

// 6. Card interfaces - For card-linked accounts
interface ICardOperations {
    blockCard(): void;
    unblockCard(): void;
    setCardLimit(limit: number): void;
    getCardLimit(): number;
}

// 7. Overdraft interfaces - For overdraft facilities
interface IOverdraftFacility {
    getOverdraftLimit(): number;
    setOverdraftLimit(limit: number): void;
    getAvailableOverdraft(): number;
}

// 8. Fixed deposit interfaces - For term deposits
interface IFixedDepositOperations {
    getMaturityDate(): Date;
    breakDeposit(): number;
    extendMaturity(months: number): void;
}

// ============================================================
// IMPLEMENTATIONS - Each class implements only what it needs
// ============================================================

// ✅ Savings Account - Simple and clean
class SavingsAccount implements 
    IBalanceQuery, 
    IDepositable, 
    IWithdrawable, 
    IInterestBearing {
    
    constructor(
        private accountNumber: string,
        private balance: number,
        private interestRate: number = 3.0
    ) {}
    
    // IBalanceQuery
    getAccountNumber(): string { return this.accountNumber; }
    getBalance(): number { return this.balance; }
    
    // IDepositable
    deposit(amount: number): void {
        this.balance += amount;
        console.log(`  ✓ Deposited ${amount} AED to ${this.accountNumber}`);
    }
    
    // IWithdrawable
    withdraw(amount: number): void {
        if (this.balance >= amount) {
            this.balance -= amount;
            console.log(`  ✓ Withdrawn ${amount} AED from ${this.accountNumber}`);
        } else {
            throw new Error('Insufficient funds');
        }
    }
    
    // IInterestBearing
    getInterestRate(): number { return this.interestRate; }
    calculateInterest(): number {
        return this.balance * (this.interestRate / 100);
    }
    applyInterest(): void {
        const interest = this.calculateInterest();
        this.balance += interest;
        console.log(`  ✓ Applied interest ${interest} AED to ${this.accountNumber}`);
    }
}

// ✅ Current Account - With overdraft
class CurrentAccount implements 
    IBalanceQuery, 
    IDepositable, 
    IWithdrawable, 
    IOverdraftFacility {
    
    constructor(
        private accountNumber: string,
        private balance: number,
        private overdraftLimit: number = 5000
    ) {}
    
    getAccountNumber(): string { return this.accountNumber; }
    getBalance(): number { return this.balance; }
    
    deposit(amount: number): void {
        this.balance += amount;
        console.log(`  ✓ Deposited ${amount} AED to ${this.accountNumber}`);
    }
    
    withdraw(amount: number): void {
        const available = this.balance + this.overdraftLimit;
        if (available >= amount) {
            this.balance -= amount;
            console.log(`  ✓ Withdrawn ${amount} AED from ${this.accountNumber}`);
        } else {
            throw new Error(`Insufficient funds. Available: ${available} AED`);
        }
    }
    
    // IOverdraftFacility
    getOverdraftLimit(): number { return this.overdraftLimit; }
    setOverdraftLimit(limit: number): void {
        this.overdraftLimit = limit;
        console.log(`  ✓ Overdraft limit set to ${limit} AED`);
    }
    getAvailableOverdraft(): number {
        return this.balance < 0 ? this.overdraftLimit + this.balance : this.overdraftLimit;
    }
}

// ✅ Fixed Deposit - Only what it needs
class FixedDepositAccount implements 
    IBalanceQuery, 
    IInterestBearing, 
    IFixedDepositOperations {
    
    constructor(
        private accountNumber: string,
        private balance: number,
        private maturityDate: Date,
        private interestRate: number = 6.0
    ) {}
    
    getAccountNumber(): string { return this.accountNumber; }
    getBalance(): number { return this.balance; }
    
    getInterestRate(): number { return this.interestRate; }
    calculateInterest(): number {
        return this.balance * (this.interestRate / 100);
    }
    applyInterest(): void {
        const interest = this.calculateInterest();
        this.balance += interest;
        console.log(`  ✓ Applied interest ${interest} AED to ${this.accountNumber}`);
    }
    
    // IFixedDepositOperations
    getMaturityDate(): Date { return this.maturityDate; }
    
    breakDeposit(): number {
        console.log(`  ⚠️  Breaking deposit ${this.accountNumber} early`);
        const penalty = this.balance * 0.01; // 1% penalty
        const amount = this.balance - penalty;
        this.balance = 0;
        console.log(`  Penalty: ${penalty} AED, Amount: ${amount} AED`);
        return amount;
    }
    
    extendMaturity(months: number): void {
        const newDate = new Date(this.maturityDate);
        newDate.setMonth(newDate.getMonth() + months);
        this.maturityDate = newDate;
        console.log(`  ✓ Maturity extended to ${this.maturityDate.toLocaleDateString()}`);
    }
}

// ✅ Credit Card - Card and loan operations
class CreditCardAccount implements 
    IBalanceQuery, 
    ILoanAccount, 
    ICardOperations {
    
    constructor(
        private accountNumber: string,
        private outstandingBalance: number,
        private creditLimit: number,
        private cardBlocked: boolean = false
    ) {}
    
    getAccountNumber(): string { return this.accountNumber; }
    getBalance(): number { return -this.outstandingBalance; } // Negative balance
    
    // ILoanAccount
    getLoanBalance(): number { return this.outstandingBalance; }
    
    payLoan(amount: number): void {
        this.outstandingBalance -= amount;
        console.log(`  ✓ Paid ${amount} AED to ${this.accountNumber}`);
        console.log(`  Outstanding: ${this.outstandingBalance} AED`);
    }
    
    calculateEMI(): number {
        const months = 12;
        const interest = this.outstandingBalance * 0.18; // 18% annual
        return (this.outstandingBalance + interest) / months;
    }
    
    // ICardOperations
    blockCard(): void {
        this.cardBlocked = true;
        console.log(`  🔒 Card ${this.accountNumber} blocked`);
    }
    
    unblockCard(): void {
        this.cardBlocked = false;
        console.log(`  🔓 Card ${this.accountNumber} unblocked`);
    }
    
    setCardLimit(limit: number): void {
        this.creditLimit = limit;
        console.log(`  ✓ Card limit set to ${limit} AED`);
    }
    
    getCardLimit(): number { return this.creditLimit; }
}

// ✅ Investment Account - Investment operations
class InvestmentAccount implements 
    IBalanceQuery, 
    IInvestmentAccount {
    
    private units: number = 0;
    private navPerUnit: number = 100; // Net Asset Value
    
    constructor(
        private accountNumber: string,
        private cashBalance: number
    ) {}
    
    getAccountNumber(): string { return this.accountNumber; }
    getBalance(): number { return this.cashBalance; }
    
    // IInvestmentAccount
    buyUnits(amount: number): void {
        if (this.cashBalance >= amount) {
            const unitsToBuy = amount / this.navPerUnit;
            this.units += unitsToBuy;
            this.cashBalance -= amount;
            console.log(`  ✓ Bought ${unitsToBuy.toFixed(2)} units for ${amount} AED`);
        } else {
            throw new Error('Insufficient cash balance');
        }
    }
    
    sellUnits(units: number): void {
        if (this.units >= units) {
            const amount = units * this.navPerUnit;
            this.units -= units;
            this.cashBalance += amount;
            console.log(`  ✓ Sold ${units.toFixed(2)} units for ${amount} AED`);
        } else {
            throw new Error('Insufficient units');
        }
    }
    
    getPortfolioValue(): number {
        return this.units * this.navPerUnit + this.cashBalance;
    }
    
    getUnitsHeld(): number { return this.units; }
}

// ============================================================
// DEMONSTRATION
// ============================================================

console.log('1️⃣  Savings Account Operations:\n');
const savings = new SavingsAccount('SAV-001', 10000, 3.5);
savings.deposit(5000);
savings.withdraw(2000);
savings.applyInterest();
console.log(`Final Balance: ${savings.getBalance()} AED\n`);

console.log('2️⃣  Current Account with Overdraft:\n');
const current = new CurrentAccount('CUR-001', 3000, 5000);
current.deposit(2000);
current.withdraw(7000); // Uses overdraft
console.log(`Balance: ${current.getBalance()} AED`);
console.log(`Available Overdraft: ${current.getAvailableOverdraft()} AED\n`);

console.log('3️⃣  Fixed Deposit Operations:\n');
const fixed = new FixedDepositAccount(
    'FD-001', 
    50000, 
    new Date('2026-12-31'), 
    6.5
);
console.log(`Maturity: ${fixed.getMaturityDate().toLocaleDateString()}`);
fixed.applyInterest();
fixed.extendMaturity(6);
console.log();

console.log('4️⃣  Credit Card Operations:\n');
const creditCard = new CreditCardAccount('CC-001', 15000, 50000);
console.log(`Outstanding: ${creditCard.getLoanBalance()} AED`);
console.log(`Monthly EMI: ${creditCard.calculateEMI().toFixed(2)} AED`);
creditCard.payLoan(5000);
creditCard.blockCard();
creditCard.unblockCard();
console.log();

console.log('5️⃣  Investment Account Operations:\n');
const investment = new InvestmentAccount('INV-001', 100000);
investment.buyUnits(50000);
console.log(`Units Held: ${investment.getUnitsHeld().toFixed(2)}`);
console.log(`Portfolio Value: ${investment.getPortfolioValue()} AED`);
investment.sellUnits(100);
console.log(`Cash Balance: ${investment.getBalance()} AED\n`);

// ============================================================
// BENEFITS - Functions work with specific interfaces
// ============================================================

console.log('6️⃣  Interface-Based Operations:\n');

function processDeposit(account: IDepositable, amount: number): void {
    account.deposit(amount);
}

function applyInterestToAll(accounts: IInterestBearing[]): void {
    console.log('Applying interest to all eligible accounts:');
    accounts.forEach(account => account.applyInterest());
}

function manageCards(accounts: ICardOperations[]): void {
    console.log('Managing card operations:');
    accounts.forEach(account => {
        account.setCardLimit(75000);
    });
}

// Only accounts that support deposits
const depositableAccounts: IDepositable[] = [savings, current];
depositableAccounts.forEach(acc => processDeposit(acc, 1000));

console.log();

// Only accounts that earn interest
const interestAccounts: IInterestBearing[] = [savings, fixed];
applyInterestToAll(interestAccounts);

console.log('\n' + '='.repeat(70));
console.log('ISP Benefits Summary:');
console.log('  ✅ SavingsAccount: 4 interfaces, ~40 lines (was 17 methods, 80+ lines)');
console.log('  ✅ FixedDeposit: 3 interfaces, ~35 lines (was 17 methods, 80+ lines)');
console.log('  ✅ CreditCard: 3 interfaces, ~40 lines (was 17 methods, 80+ lines)');
console.log('  ✅ No dummy implementations or exceptions');
console.log('  ✅ Clear what each account supports');
console.log('  ✅ Easy to add new interfaces without affecting existing code');
console.log('  ✅ Loose coupling - depend only on what you need');
console.log('='.repeat(70) + '\n');
```

---

## More Real-World Examples

```typescript
console.log('=== More ISP Examples ===\n');

// ❌ BAD: Fat Interface for all document operations
console.log('1. DOCUMENT MANAGEMENT:\n');

interface IFatDocument {
    open(): void;
    close(): void;
    save(): void;
    print(): void;
    scan(): void;
    fax(): void;
    email(): void;
    encrypt(): void;
    decrypt(): void;
}

// ❌ Simple document doesn't need all these!
class SimpleDocument implements IFatDocument {
    open(): void { console.log('Opening...'); }
    close(): void { console.log('Closing...'); }
    save(): void { console.log('Saving...'); }
    
    // ❌ Forced to implement these
    print(): void { throw new Error('Not supported'); }
    scan(): void { throw new Error('Not supported'); }
    fax(): void { throw new Error('Not supported'); }
    email(): void { throw new Error('Not supported'); }
    encrypt(): void { throw new Error('Not supported'); }
    decrypt(): void { throw new Error('Not supported'); }
}

console.log('  ❌ SimpleDocument forced to implement 6 irrelevant methods\n');

// ✅ GOOD: Segregated interfaces
interface IOpenable { open(): void; }
interface ICloseable { close(): void; }
interface ISaveable { save(): void; }
interface IPrintable { print(): void; }
interface IScannabled { scan(): void; }
interface IFaxable { fax(): void; }
interface IEmailable { email(): void; }
interface IEncryptable { encrypt(): void; decrypt(): void; }

class GoodSimpleDocument implements IOpenable, ICloseable, ISaveable {
    open(): void { console.log('  ✓ Opening...'); }
    close(): void { console.log('  ✓ Closing...'); }
    save(): void { console.log('  ✓ Saving...'); }
}

class SecureDocument extends GoodSimpleDocument implements IEncryptable {
    encrypt(): void { console.log('  ✓ Encrypting...'); }
    decrypt(): void { console.log('  ✓ Decrypting...'); }
}

console.log('  ✅ GoodSimpleDocument: Only 3 methods needed');
console.log('  ✅ SecureDocument: Adds encryption on top\n');

// ❌ BAD: Worker interface that assumes all workers are robots
console.log('2. WORKER MANAGEMENT:\n');

interface IFatWorker {
    work(): void;
    eat(): void;
    sleep(): void;
    charge(): void; // For robots
    maintenance(): void; // For robots
}

class Robot implements IFatWorker {
    work(): void { console.log('Robot working...'); }
    charge(): void { console.log('Charging battery...'); }
    maintenance(): void { console.log('Running diagnostics...'); }
    
    // ❌ Robots don't eat or sleep!
    eat(): void { throw new Error('Robots don\'t eat'); }
    sleep(): void { throw new Error('Robots don\'t sleep'); }
}

console.log('  ❌ Robot forced to implement eat() and sleep()\n');

// ✅ GOOD: Segregated worker interfaces
interface IWorkable { work(): void; }
interface IFeedable { eat(): void; }
interface ISleepable { sleep(): void; }
interface IChargeable { charge(): void; }
interface IMaintainable { maintenance(): void; }

class GoodRobot implements IWorkable, IChargeable, IMaintainable {
    work(): void { console.log('  ✓ Robot working...'); }
    charge(): void { console.log('  ✓ Charging...'); }
    maintenance(): void { console.log('  ✓ Maintenance...'); }
}

class Human implements IWorkable, IFeedable, ISleepable {
    work(): void { console.log('  ✓ Human working...'); }
    eat(): void { console.log('  ✓ Eating...'); }
    sleep(): void { console.log('  ✓ Sleeping...'); }
}

console.log('  ✅ Robot: 3 relevant methods');
console.log('  ✅ Human: 3 relevant methods');
console.log('  ✅ No dummy implementations\n');

console.log('='.repeat(70) + '\n');
```

---

## Interview Questions & Answers

### Q1: What is Interface Segregation Principle?
**Answer**: ISP states that clients should not be forced to depend on interfaces they don't use. Instead of one large interface with many methods, create multiple smaller, focused interfaces. Classes implement only the interfaces they need.

Example: Instead of `IBankAccount` with 20 methods, split into `IDepositable`, `IWithdrawable`, `IInterestBearing`, etc.

### Q2: How do you identify ISP violations?
**Answer**: Look for:
1. **Fat interfaces** - Many methods in one interface
2. **Empty implementations** - Methods that do nothing
3. **Exception throwing** - Methods that throw "Not supported"
4. **Comments** - "Not applicable" or "Not used"
5. **Dummy returns** - Returning null or default values
6. **Many implementers** - But each uses only a subset of methods

### Q3: Give a banking example of ISP violation
**Answer**:
```typescript
// ❌ Violation - Fat interface
interface IBankAccount {
    deposit(); withdraw(); // All accounts
    calculateInterest(); // Only some
    payLoan(); // Only credit
    buyUnits(); // Only investment
    blockCard(); // Only card accounts
}

// Savings forced to implement loan/investment methods!
class Savings implements IBankAccount {
    deposit() { /*...*/ }
    withdraw() { /*...*/ }
    calculateInterest() { /*...*/ }
    payLoan() { throw new Error("No loan!"); } // ❌
    buyUnits() { throw new Error("No investment!"); } // ❌
    blockCard() { throw new Error("No card!"); } // ❌
}

// ✅ Solution - Segregated interfaces
interface IDepositable { deposit(); }
interface IWithdrawable { withdraw(); }
interface IInterestBearing { calculateInterest(); }
interface ILoanAccount { payLoan(); }

class GoodSavings implements IDepositable, IWithdrawable, IInterestBearing {
    // Only implements what it needs!
}
```

### Q4: What's the difference between ISP and SRP?
**Answer**:
- **SRP**: About classes - A class should have one reason to change
- **ISP**: About interfaces - Split interfaces so clients depend only on what they use

**Example**:
```typescript
// SRP: AccountService does ONE thing (account management)
class AccountService {
    createAccount() {}
    closeAccount() {}
}

// ISP: Split interface so clients use only what they need
interface IAccountCreation { createAccount(); }
interface IAccountClosure { closeAccount(); }
```

Both reduce coupling but at different levels.

### Q5: How does ISP improve testability?
**Answer**:
1. **Smaller mocks** - Mock only what's used
2. **Focused tests** - Test one capability at a time
3. **Clear dependencies** - Know exactly what's needed

```typescript
// ❌ Without ISP - Must mock huge interface
function processPayment(account: IBankAccountFat) {
    account.withdraw(100); // Only uses this method
}
// Mock needs all 20 methods!

// ✅ With ISP - Mock only what's needed
function processPayment(account: IWithdrawable) {
    account.withdraw(100);
}
// Mock needs only 1 method!
```

### Q6: When should you split an interface?
**Answer**: Split when:
1. **Different clients** use different subsets of methods
2. **Some implementations** throw "Not supported"
3. **Adding methods** affects unrelated clients
4. **Interface has** unrelated responsibilities
5. **Naming is hard** - Interface does too many things

Don't split too early - Wait for actual need (YAGNI).

### Q7: How to refactor fat interfaces?
**Answer**:
**Step 1**: Identify method groups
```typescript
// Fat interface has: balance, transactions, interest, loans
interface IFat { getBalance(); deposit(); withdraw(); 
                 calculateInterest(); payLoan(); }
```

**Step 2**: Create focused interfaces
```typescript
interface IBalanceQuery { getBalance(); }
interface ITransactions { deposit(); withdraw(); }
interface IInterest { calculateInterest(); }
interface ILoan { payLoan(); }
```

**Step 3**: Compose as needed
```typescript
class Savings implements IBalanceQuery, ITransactions, IInterest {}
class CreditCard implements IBalanceQuery, ILoan {}
```

**Step 4**: Update clients to use specific interfaces
```typescript
// Before: function process(account: IFat)
// After: function process(account: ITransactions)
```

---

## Key Takeaways

### ✅ DO:
- Create small, focused interfaces
- One role or responsibility per interface
- Compose interfaces to build complex types
- Use interface names that clearly state purpose
- Keep interfaces cohesive
- Design interfaces from client's perspective

### ❌ DON'T:
- Create "god interfaces" with many methods
- Force classes to implement unused methods
- Make interfaces depend on implementation details
- Add methods just because it's convenient
- Create one interface per method (too granular)
- Throw "Not supported" exceptions

### Remember:
> "Many client-specific interfaces are better than one general-purpose interface"
> 
> If a class implements an interface but throws exceptions or has empty methods, the interface is too fat!

---

## Summary

**Interface Segregation Principle** keeps interfaces focused in banking applications:

- ✅ Split `IBankAccount` into `IDepositable`, `IWithdrawable`, `IInterestBearing`, etc.
- ✅ Savings implements only deposit, withdraw, interest operations
- ✅ Fixed Deposit implements only balance, interest, maturity operations
- ✅ No dummy implementations or "Not supported" exceptions
- ✅ Clients depend only on methods they actually use

Following ISP makes your banking application:
- **Flexible** - Mix and match capabilities
- **Maintainable** - Changes don't ripple everywhere
- **Testable** - Mock only what's needed
- **Clear** - Obvious what each account supports
- **Decoupled** - Loose coupling between components

**The key question**: "Does every implementer use every method in this interface?"

If NO, split the interface:
- Group related methods
- Create focused interfaces
- Compose interfaces as needed
- Update clients to use specific interfaces

**Real-world impact**:
- New account types easier to add
- Testing requires fewer mocks
- Clear contracts between components
- Changes isolated to relevant clients
- No "Not applicable" methods
