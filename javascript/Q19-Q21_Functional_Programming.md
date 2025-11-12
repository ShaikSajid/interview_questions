# Questions 19-21: Functional Programming & Advanced Concepts

## Question 19: What is Functional Programming? Explain Pure Functions, Immutability, and Higher-Order Functions.

### Answer:

**Functional Programming (FP)** is a programming paradigm that treats computation as the evaluation of mathematical functions and avoids changing state and mutable data.

### Core Concepts:

1. **Pure Functions**: 
   - Same input always produces same output
   - No side effects (no external state modification)
   - Predictable and testable

2. **Immutability**:
   - Data cannot be changed after creation
   - Create new data structures instead of modifying existing ones
   - Prevents bugs from unexpected mutations

3. **Higher-Order Functions**:
   - Functions that take functions as arguments
   - Functions that return functions
   - Examples: map, filter, reduce, compose

4. **First-Class Functions**:
   - Functions are values (can be assigned to variables)
   - Can be passed as arguments
   - Can be returned from other functions

### Banking Scenario: Functional Transaction Processing at Emirates NBD

```javascript
// Pure Functions - No side effects, deterministic
class PureFunctions {
    // ✅ Pure: Same input → Same output, no side effects
    static calculateInterest(principal, rate, time) {
        return principal * rate * time;
    }

    // ✅ Pure: No external state modification
    static applyTransactionFee(amount, feePercent) {
        return amount * (1 - feePercent / 100);
    }

    // ✅ Pure: Creates new object, doesn't modify input
    static updateAccountBalance(account, transaction) {
        return {
            ...account,
            balance: account.balance + transaction.amount,
            transactions: [...account.transactions, transaction]
        };
    }

    // ❌ Impure: Modifies external state
    static impureUpdateBalance(account, amount) {
        account.balance += amount; // Mutation!
        console.log('Balance updated'); // Side effect!
        return account;
    }

    // ❌ Impure: Depends on external state
    static impureGetDiscountRate(amount) {
        return amount > window.globalDiscountThreshold ? 0.1 : 0;
    }
}

// Immutability Principles
class ImmutableOperations {
    // ✅ Immutable: Create new array instead of modifying
    static addTransaction(transactions, newTransaction) {
        // Don't use push() - it mutates!
        return [...transactions, newTransaction];
    }

    // ✅ Immutable: Create new object with updated properties
    static updateAccount(account, updates) {
        return {
            ...account,
            ...updates,
            updatedAt: new Date().toISOString()
        };
    }

    // ✅ Immutable: Filter without mutating
    static getApprovedTransactions(transactions) {
        return transactions.filter(t => t.status === 'APPROVED');
    }

    // ✅ Immutable: Deep clone for nested structures
    static updateNestedProperty(account, path, value) {
        // Use structuredClone for deep immutability
        const clone = structuredClone(account);
        
        // Navigate to the property
        const keys = path.split('.');
        let current = clone;
        for (let i = 0; i < keys.length - 1; i++) {
            current = current[keys[i]];
        }
        current[keys[keys.length - 1]] = value;
        
        return clone;
    }

    // Using Object.freeze for immutability
    static createImmutableAccount(accountData) {
        // Shallow freeze
        return Object.freeze({ ...accountData });
    }

    // Deep freeze for nested objects
    static deepFreeze(obj) {
        Object.freeze(obj);
        
        Object.keys(obj).forEach(key => {
            if (obj[key] !== null && typeof obj[key] === 'object') {
                ImmutableOperations.deepFreeze(obj[key]);
            }
        });
        
        return obj;
    }
}

// Higher-Order Functions
class HigherOrderFunctions {
    // Function that returns a function
    static createTransactionValidator(rules) {
        return (transaction) => {
            return rules.every(rule => rule(transaction));
        };
    }

    // Function that takes a function as argument
    static processTransactions(transactions, processor) {
        return transactions.map(processor);
    }

    // Composition: Combine multiple functions
    static compose(...fns) {
        return (value) => {
            return fns.reduceRight((acc, fn) => fn(acc), value);
        };
    }

    // Pipe: Like compose but left-to-right
    static pipe(...fns) {
        return (value) => {
            return fns.reduce((acc, fn) => fn(acc), value);
        };
    }

    // Currying: Transform function(a, b, c) to function(a)(b)(c)
    static curry(fn) {
        return function curried(...args) {
            if (args.length >= fn.length) {
                return fn.apply(this, args);
            } else {
                return function (...nextArgs) {
                    return curried.apply(this, [...args, ...nextArgs]);
                };
            }
        };
    }

    // Partial application
    static partial(fn, ...fixedArgs) {
        return function (...remainingArgs) {
            return fn(...fixedArgs, ...remainingArgs);
        };
    }
}

// Functional Banking Service
class FunctionalBankingService {
    // Pure transaction processing pipeline
    static processTransactionPipeline(transaction) {
        const validateAmount = (txn) => {
            if (txn.amount <= 0) throw new Error('Invalid amount');
            return txn;
        };

        const applyFee = (txn) => ({
            ...txn,
            fee: txn.amount * 0.01,
            netAmount: txn.amount * 0.99
        });

        const addTimestamp = (txn) => ({
            ...txn,
            processedAt: new Date().toISOString()
        });

        const addConfirmation = (txn) => ({
            ...txn,
            confirmationNumber: `CNF-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
        });

        // Use pipe to create processing pipeline
        const pipeline = HigherOrderFunctions.pipe(
            validateAmount,
            applyFee,
            addTimestamp,
            addConfirmation
        );

        return pipeline(transaction);
    }

    // Map: Transform each transaction
    static calculateTransactionFees(transactions) {
        return transactions.map(txn => ({
            ...txn,
            fee: txn.amount * 0.01,
            total: txn.amount * 1.01
        }));
    }

    // Filter: Get specific transactions
    static getHighValueTransactions(transactions, threshold = 10000) {
        return transactions.filter(txn => txn.amount >= threshold);
    }

    // Reduce: Aggregate transactions
    static calculateTotalBalance(transactions) {
        return transactions.reduce((total, txn) => {
            return txn.type === 'CREDIT' 
                ? total + txn.amount 
                : total - txn.amount;
        }, 0);
    }

    // Reduce to create summary object
    static getTransactionSummary(transactions) {
        return transactions.reduce((summary, txn) => {
            return {
                totalTransactions: summary.totalTransactions + 1,
                totalAmount: summary.totalAmount + txn.amount,
                byType: {
                    ...summary.byType,
                    [txn.type]: (summary.byType[txn.type] || 0) + 1
                },
                byStatus: {
                    ...summary.byStatus,
                    [txn.status]: (summary.byStatus[txn.status] || 0) + 1
                }
            };
        }, {
            totalTransactions: 0,
            totalAmount: 0,
            byType: {},
            byStatus: {}
        });
    }

    // Every: Check if all transactions meet criteria
    static allTransactionsApproved(transactions) {
        return transactions.every(txn => txn.status === 'APPROVED');
    }

    // Some: Check if any transaction meets criteria
    static hasHighValueTransaction(transactions, threshold = 50000) {
        return transactions.some(txn => txn.amount >= threshold);
    }

    // Find: Get first matching transaction
    static findTransactionById(transactions, id) {
        return transactions.find(txn => txn.id === id);
    }

    // FlatMap: Transform and flatten
    static getAllTransactionDetails(accounts) {
        return accounts.flatMap(account => 
            account.transactions.map(txn => ({
                accountNumber: account.accountNumber,
                accountName: account.name,
                ...txn
            }))
        );
    }
}

// Currying Example
class CurriedFunctions {
    // Regular function
    static calculateLoanPayment(principal, rate, years) {
        const monthlyRate = rate / 12 / 100;
        const numPayments = years * 12;
        
        return (principal * monthlyRate * Math.pow(1 + monthlyRate, numPayments)) /
               (Math.pow(1 + monthlyRate, numPayments) - 1);
    }

    // Curried version
    static curriedCalculateLoanPayment = HigherOrderFunctions.curry(
        CurriedFunctions.calculateLoanPayment
    );

    // Create specialized functions
    static create5YearLoanCalculator() {
        return CurriedFunctions.curriedCalculateLoanPayment(5);
    }

    static createStandardRateLoanCalculator() {
        return CurriedFunctions.curriedCalculateLoanPayment(5)(5); // 5% rate
    }
}

// Function Composition Example
class TransactionProcessor {
    // Individual transformation functions
    static sanitizeInput(transaction) {
        return {
            ...transaction,
            amount: Math.abs(parseFloat(transaction.amount) || 0),
            description: transaction.description?.trim() || ''
        };
    }

    static validateTransaction(transaction) {
        if (transaction.amount <= 0) {
            throw new Error('Amount must be positive');
        }
        if (!transaction.fromAccount || !transaction.toAccount) {
            throw new Error('Account numbers required');
        }
        return transaction;
    }

    static checkFraud(transaction) {
        // Simulate fraud check
        const isSuspicious = transaction.amount > 100000;
        
        return {
            ...transaction,
            fraudCheck: {
                passed: !isSuspicious,
                risk: isSuspicious ? 'HIGH' : 'LOW',
                checkedAt: new Date().toISOString()
            }
        };
    }

    static applyBusinessRules(transaction) {
        const fee = transaction.amount > 10000 ? 10 : 5;
        
        return {
            ...transaction,
            fee,
            netAmount: transaction.amount - fee
        };
    }

    static enrichWithMetadata(transaction) {
        return {
            ...transaction,
            id: `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
            timestamp: new Date().toISOString(),
            status: 'PENDING'
        };
    }

    // Compose all steps into single pipeline
    static createProcessingPipeline() {
        return HigherOrderFunctions.pipe(
            TransactionProcessor.sanitizeInput,
            TransactionProcessor.validateTransaction,
            TransactionProcessor.checkFraud,
            TransactionProcessor.applyBusinessRules,
            TransactionProcessor.enrichWithMetadata
        );
    }
}

// Demonstration
console.log('=== Functional Programming Demo - Emirates NBD ===\n');

// Demo 1: Pure Functions
console.log('1. Pure Functions:\n');

const principal = 100000;
const rate = 0.05;
const time = 10;

const interest1 = PureFunctions.calculateInterest(principal, rate, time);
const interest2 = PureFunctions.calculateInterest(principal, rate, time);

console.log(`Interest calculation 1: AED ${interest1.toFixed(2)}`);
console.log(`Interest calculation 2: AED ${interest2.toFixed(2)}`);
console.log(`✅ Always same result (deterministic): ${interest1 === interest2}\n`);

const account = {
    accountNumber: 'ACC-123456789',
    balance: 50000,
    transactions: []
};

const transaction = { id: 'TXN-001', amount: 5000, type: 'CREDIT' };
const updatedAccount = PureFunctions.updateAccountBalance(account, transaction);

console.log('Original account balance:', account.balance);
console.log('Updated account balance:', updatedAccount.balance);
console.log(`✅ Original unchanged (immutable): ${account.balance === 50000}\n`);

console.log('='.repeat(70) + '\n');

// Demo 2: Immutability
console.log('2. Immutability:\n');

const transactions = [
    { id: 'TXN-001', amount: 1000, status: 'APPROVED' },
    { id: 'TXN-002', amount: 2000, status: 'PENDING' }
];

const newTransactions = ImmutableOperations.addTransaction(
    transactions,
    { id: 'TXN-003', amount: 3000, status: 'APPROVED' }
);

console.log(`Original transactions count: ${transactions.length}`);
console.log(`New transactions count: ${newTransactions.length}`);
console.log(`✅ Original array unchanged: ${transactions.length === 2}\n`);

// Object.freeze example
const frozenAccount = ImmutableOperations.createImmutableAccount({
    accountNumber: 'ACC-999',
    balance: 10000
});

try {
    frozenAccount.balance = 99999; // This won't work
    console.log('Attempted to modify frozen object...');
    console.log(`Balance still: ${frozenAccount.balance}`);
    console.log('✅ Object.freeze prevented modification\n');
} catch (error) {
    console.log('Error:', error.message);
}

console.log('='.repeat(70) + '\n');

// Demo 3: Higher-Order Functions - Map, Filter, Reduce
console.log('3. Higher-Order Functions:\n');

const sampleTransactions = [
    { id: 'TXN-001', amount: 5000, type: 'CREDIT', status: 'APPROVED' },
    { id: 'TXN-002', amount: 15000, type: 'DEBIT', status: 'APPROVED' },
    { id: 'TXN-003', amount: 3000, type: 'CREDIT', status: 'PENDING' },
    { id: 'TXN-004', amount: 25000, type: 'CREDIT', status: 'APPROVED' },
    { id: 'TXN-005', amount: 8000, type: 'DEBIT', status: 'APPROVED' }
];

console.log('Map - Add fees to all transactions:');
const withFees = FunctionalBankingService.calculateTransactionFees(sampleTransactions);
console.log(`  First transaction: ${withFees[0].id} - Fee: AED ${withFees[0].fee.toFixed(2)}\n`);

console.log('Filter - Get high-value transactions (>= 10,000):');
const highValue = FunctionalBankingService.getHighValueTransactions(sampleTransactions, 10000);
console.log(`  Found ${highValue.length} high-value transactions`);
highValue.forEach(t => console.log(`    - ${t.id}: AED ${t.amount.toLocaleString()}`));
console.log();

console.log('Reduce - Calculate net balance:');
const netBalance = FunctionalBankingService.calculateTotalBalance(sampleTransactions);
console.log(`  Net Balance: AED ${netBalance.toLocaleString()}\n`);

console.log('Reduce - Create summary:');
const summary = FunctionalBankingService.getTransactionSummary(sampleTransactions);
console.log(`  Total Transactions: ${summary.totalTransactions}`);
console.log(`  Total Amount: AED ${summary.totalAmount.toLocaleString()}`);
console.log(`  By Type:`, summary.byType);
console.log(`  By Status:`, summary.byStatus);
console.log();

console.log('Every - All approved?:', 
    FunctionalBankingService.allTransactionsApproved(sampleTransactions) ? '❌ No' : '✅ Yes'
);

console.log('Some - Has high-value transaction?:', 
    FunctionalBankingService.hasHighValueTransaction(sampleTransactions, 20000) ? '✅ Yes' : '❌ No'
);

console.log();
console.log('='.repeat(70) + '\n');

// Demo 4: Function Composition and Pipe
console.log('4. Function Composition & Pipeline:\n');

const rawTransaction = {
    amount: '5000.50',
    description: '  Transfer to savings  ',
    fromAccount: 'ACC-123456789',
    toAccount: 'ACC-987654321'
};

console.log('Raw transaction:', rawTransaction);

const pipeline = TransactionProcessor.createProcessingPipeline();
const processedTransaction = pipeline(rawTransaction);

console.log('\nProcessed transaction:');
console.log('  ID:', processedTransaction.id);
console.log('  Amount:', processedTransaction.amount);
console.log('  Fee:', processedTransaction.fee);
console.log('  Net Amount:', processedTransaction.netAmount);
console.log('  Fraud Check:', processedTransaction.fraudCheck.passed ? '✅ Passed' : '❌ Failed');
console.log('  Status:', processedTransaction.status);
console.log();

console.log('='.repeat(70) + '\n');

// Demo 5: Currying and Partial Application
console.log('5. Currying & Partial Application:\n');

// Regular function call
const payment1 = CurriedFunctions.calculateLoanPayment(100000, 5, 5);
console.log(`Regular: AED ${payment1.toFixed(2)}/month for 100k loan at 5% for 5 years\n`);

// Curried function (can be called in steps)
const step1 = CurriedFunctions.curriedCalculateLoanPayment(100000);
const step2 = step1(5);
const payment2 = step2(5);
console.log(`Curried: AED ${payment2.toFixed(2)}/month (same result)\n`);

// Create specialized calculators
const loanCalc5Years = CurriedFunctions.curriedCalculateLoanPayment;
const for100k = loanCalc5Years(100000);
const at5Percent = for100k(5);

console.log('Specialized calculators:');
console.log(`  3 years: AED ${at5Percent(3).toFixed(2)}/month`);
console.log(`  5 years: AED ${at5Percent(5).toFixed(2)}/month`);
console.log(`  10 years: AED ${at5Percent(10).toFixed(2)}/month`);
console.log();

// Partial application
const calculateWithStandardRate = HigherOrderFunctions.partial(
    CurriedFunctions.calculateLoanPayment,
    // Fix rate at 5%
);

console.log('Partial application for standard 5% rate:');
console.log(`  50k for 3 years: AED ${CurriedFunctions.calculateLoanPayment(50000, 5, 3).toFixed(2)}/month`);
console.log(`  100k for 5 years: AED ${CurriedFunctions.calculateLoanPayment(100000, 5, 5).toFixed(2)}/month`);
console.log();

console.log('='.repeat(70) + '\n');

// Demo 6: Functional Programming Benefits
console.log('6. Functional Programming Benefits:\n');

console.log('✅ Benefits:');
console.log('  • Predictable: Pure functions always return same output');
console.log('  • Testable: Easy to unit test without mocks');
console.log('  • Composable: Combine small functions into complex operations');
console.log('  • Reusable: Pure functions work in any context');
console.log('  • Debuggable: No hidden state or side effects');
console.log('  • Parallel: Immutable data is thread-safe');

console.log('\n🔑 Key Principles:');
console.log('  1. Pure Functions: No side effects, deterministic');
console.log('  2. Immutability: Never modify data, create new copies');
console.log('  3. First-Class Functions: Functions as values');
console.log('  4. Higher-Order Functions: Functions that operate on functions');
console.log('  5. Function Composition: Combine simple functions');

console.log('\n📚 Common Patterns:');
console.log('  • map: Transform each element');
console.log('  • filter: Select elements by criteria');
console.log('  • reduce: Aggregate to single value');
console.log('  • compose/pipe: Chain transformations');
console.log('  • curry: Partial application of arguments');

console.log('\n⚠️  When to Avoid:');
console.log('  • Performance-critical loops (use for loops)');
console.log('  • Large data structures (memory overhead)');
console.log('  • DOM manipulation (inherently imperative)');
console.log('  • I/O operations (require side effects)');
```

### Key Takeaways:

**Pure Functions**:
- ✅ Same input → same output
- ✅ No side effects
- ✅ Don't modify external state
- ✅ Don't depend on external state

**Immutability**:
- ✅ Use spread operator `...` for arrays/objects
- ✅ Array methods: `map`, `filter`, `reduce` (return new arrays)
- ✅ `Object.freeze()` for shallow immutability
- ✅ Avoid: `push`, `pop`, `splice`, direct property assignment

**Higher-Order Functions**:
- ✅ Functions that take functions as arguments
- ✅ Functions that return functions
- ✅ Enable composition and reusability

---

## Question 20: What is Currying? How is it different from Partial Application?

### Answer:

**Currying** transforms a function with multiple arguments into a sequence of functions, each taking a single argument. **Partial Application** fixes some arguments of a function, producing a new function with fewer arguments.

### Banking Scenario: Flexible Loan Calculators at Emirates NBD

```javascript
// Currying: Transform f(a, b, c) → f(a)(b)(c)
class CurryingExamples {
    // Manual currying
    static calculateLoanInterest(principal) {
        return function(rate) {
            return function(years) {
                return principal * rate * years;
            };
        };
    }

    // Arrow function syntax (more concise)
    static calculateMonthlyPayment = (principal) => (rate) => (years) => {
        const monthlyRate = rate / 12 / 100;
        const numPayments = years * 12;
        return (principal * monthlyRate * Math.pow(1 + monthlyRate, numPayments)) /
               (Math.pow(1 + monthlyRate, numPayments) - 1);
    };

    // Generic curry helper
    static curry(fn) {
        return function curried(...args) {
            if (args.length >= fn.length) {
                return fn.apply(this, args);
            }
            return (...nextArgs) => curried(...args, ...nextArgs);
        };
    }

    // Transaction fee calculator (curried)
    static calculateFee = (baseRate) => (transactionType) => (amount) => {
        const multipliers = {
            'DOMESTIC': 1.0,
            'INTERNATIONAL': 1.5,
            'PREMIUM': 0.5
        };
        
        return amount * baseRate * (multipliers[transactionType] || 1.0);
    };
}

// Partial Application: Fix some arguments
class PartialApplicationExamples {
    // Manual partial application
    static createAccountValidator(accountType, minimumBalance) {
        return (currentBalance) => {
            return {
                valid: currentBalance >= minimumBalance,
                accountType,
                minimumBalance,
                currentBalance,
                deficit: currentBalance < minimumBalance ? minimumBalance - currentBalance : 0
            };
        };
    }

    // Generic partial application helper
    static partial(fn, ...fixedArgs) {
        return (...remainingArgs) => {
            return fn(...fixedArgs, ...remainingArgs);
        };
    }

    // Flexible partial (can specify which args to fix)
    static partialRight(fn, ...fixedArgs) {
        return (...remainingArgs) => {
            return fn(...remainingArgs, ...fixedArgs);
        };
    }

    // Transaction processor with partial application
    static processTransaction(transactionType, fee, taxRate, amount) {
        const subtotal = amount + fee;
        const tax = subtotal * taxRate;
        const total = subtotal + tax;
        
        return {
            transactionType,
            amount,
            fee,
            subtotal,
            tax,
            total
        };
    }
}

// Practical Examples
class BankingCalculators {
    // Curried loan calculator - allows building specialized calculators
    static loanCalculator = CurryingExamples.curry(
        (principal, annualRate, years, frequency) => {
            const rate = annualRate / frequency / 100;
            const periods = years * frequency;
            return (principal * rate * Math.pow(1 + rate, periods)) /
                   (Math.pow(1 + rate, periods) - 1);
        }
    );

    // Create specialized calculators using currying
    static createPersonalLoanCalculator() {
        // Personal loans: 5% rate, 12 monthly payments
        return BankingCalculators.loanCalculator()(5)()(12);
    }

    static createMortgageCalculator() {
        // Mortgages: 3.5% rate, 12 monthly payments
        return BankingCalculators.loanCalculator()(3.5)()(12);
    }

    // Partial application for standard transaction types
    static createDomesticTransferProcessor() {
        return PartialApplicationExamples.partial(
            PartialApplicationExamples.processTransaction,
            'DOMESTIC_TRANSFER',
            25,  // Fixed fee: AED 25
            0.05 // Fixed tax rate: 5%
        );
    }

    static createInternationalTransferProcessor() {
        return PartialApplicationExamples.partial(
            PartialApplicationExamples.processTransaction,
            'INTERNATIONAL_TRANSFER',
            50,  // Fixed fee: AED 50
            0.05 // Fixed tax rate: 5%
        );
    }
}

// Real-world use case: Transaction authorization
class AuthorizationService {
    // Curried authorization checker
    static checkAuthorization = (userRole) => (transactionType) => (amount) => {
        const limits = {
            'TELLER': {
                'WITHDRAWAL': 5000,
                'TRANSFER': 10000,
                'DEPOSIT': Infinity
            },
            'MANAGER': {
                'WITHDRAWAL': 50000,
                'TRANSFER': 100000,
                'DEPOSIT': Infinity
            },
            'ADMIN': {
                'WITHDRAWAL': Infinity,
                'TRANSFER': Infinity,
                'DEPOSIT': Infinity
            }
        };

        const limit = limits[userRole]?.[transactionType] ?? 0;
        
        return {
            authorized: amount <= limit,
            userRole,
            transactionType,
            amount,
            limit,
            exceeded: amount > limit ? amount - limit : 0
        };
    };

    // Create role-specific checkers using partial application
    static createTellerChecker() {
        const curried = AuthorizationService.checkAuthorization;
        return curried('TELLER');
    }

    static createManagerChecker() {
        const curried = AuthorizationService.checkAuthorization;
        return curried('MANAGER');
    }
}

// Interest rate calculator with currying
class InterestCalculator {
    // Base calculation
    static calculate(principal, rate, time, compound) {
        if (compound) {
            return principal * Math.pow(1 + rate, time) - principal;
        }
        return principal * rate * time;
    }

    // Curried version
    static curriedCalculate = CurryingExamples.curry(InterestCalculator.calculate);

    // Create specialized calculators
    static savings = InterestCalculator.curriedCalculate()(0.025); // 2.5% rate
    static fixedDeposit = InterestCalculator.curriedCalculate()(0.045); // 4.5% rate
    static loan = InterestCalculator.curriedCalculate()(0.08); // 8% rate
}

// Demonstration
console.log('=== Currying vs Partial Application - Emirates NBD ===\n');

// Demo 1: Currying Basics
console.log('1. CURRYING - Sequential Function Application:\n');

// Manual currying
const interest1 = CurryingExamples.calculateLoanInterest(100000);
const interest2 = interest1(0.05);
const interest3 = interest2(10);
console.log(`Curried interest calculation: AED ${interest3.toFixed(2)}`);

// All at once
const interestAllAtOnce = CurryingExamples.calculateLoanInterest(100000)(0.05)(10);
console.log(`Same result all at once: AED ${interestAllAtOnce.toFixed(2)}\n`);

// Arrow function syntax
const payment1 = CurryingExamples.calculateMonthlyPayment(100000);
const payment2 = payment1(5);
const payment3 = payment2(10);
console.log(`Monthly payment: AED ${payment3.toFixed(2)}\n`);

console.log('='.repeat(70) + '\n');

// Demo 2: Partial Application Basics
console.log('2. PARTIAL APPLICATION - Fixing Some Arguments:\n');

// Create specialized validators
const savingsValidator = PartialApplicationExamples.createAccountValidator('SAVINGS', 1000);
const currentValidator = PartialApplicationExamples.createAccountValidator('CURRENT', 5000);

console.log('Savings account validation (min AED 1,000):');
console.log('  Balance 1500:', savingsValidator(1500).valid ? '✅ Valid' : '❌ Invalid');
console.log('  Balance 500:', savingsValidator(500).valid ? '✅ Valid' : '❌ Invalid', 
    `(Deficit: AED ${savingsValidator(500).deficit})`);

console.log('\nCurrent account validation (min AED 5,000):');
console.log('  Balance 10000:', currentValidator(10000).valid ? '✅ Valid' : '❌ Invalid');
console.log('  Balance 3000:', currentValidator(3000).valid ? '✅ Valid' : '❌ Invalid',
    `(Deficit: AED ${currentValidator(3000).deficit})`);

console.log('\n' + '='.repeat(70) + '\n');

// Demo 3: Key Differences
console.log('3. CURRYING vs PARTIAL APPLICATION:\n');

console.log('CURRYING:');
console.log('  • Transforms f(a, b, c) → f(a)(b)(c)');
console.log('  • Always returns unary (single-argument) functions');
console.log('  • Each call returns a new function until all args provided');
console.log('  • More theoretical/mathematical');

const curriedFee = CurryingExamples.calculateFee(0.01);
const domestic = curriedFee('DOMESTIC');
const fee = domestic(5000);
console.log(`  Example: Fee = baseRate(0.01)('DOMESTIC')(5000) = AED ${fee.toFixed(2)}\n`);

console.log('PARTIAL APPLICATION:');
console.log('  • Fixes some arguments, returns function expecting rest');
console.log('  • Can fix any number of arguments');
console.log('  • More practical/flexible');
console.log('  • Creates specialized versions of functions');

const domesticProcessor = BankingCalculators.createDomesticTransferProcessor();
const result = domesticProcessor(5000);
console.log(`  Example: Domestic transfer of AED 5000:`);
console.log(`    Fee: AED ${result.fee}`);
console.log(`    Tax: AED ${result.tax.toFixed(2)}`);
console.log(`    Total: AED ${result.total.toFixed(2)}\n`);

console.log('='.repeat(70) + '\n');

// Demo 4: Practical Use Cases
console.log('4. PRACTICAL USE CASES:\n');

console.log('A) Authorization Checking (Currying):');
const tellerAuth = AuthorizationService.createTellerChecker();
const tellerWithdrawal = tellerAuth('WITHDRAWAL');
const tellerTransfer = tellerAuth('TRANSFER');

console.log('  Teller attempting withdrawal:');
console.log(`    AED 3,000: ${tellerWithdrawal(3000).authorized ? '✅ Authorized' : '❌ Denied'}`);
console.log(`    AED 10,000: ${tellerWithdrawal(10000).authorized ? '✅ Authorized' : '❌ Denied'} (Limit: AED 5,000)`);

console.log('\n  Teller attempting transfer:');
console.log(`    AED 8,000: ${tellerTransfer(8000).authorized ? '✅ Authorized' : '❌ Denied'}`);
console.log(`    AED 15,000: ${tellerTransfer(15000).authorized ? '✅ Authorized' : '❌ Denied'} (Limit: AED 10,000)\n`);

console.log('B) Interest Calculators (Currying):');
const savings1Year = InterestCalculator.savings(1)(true);
const savings5Year = InterestCalculator.savings(5)(true);

console.log('  Savings account (2.5% compound interest):');
console.log(`    AED 10,000 for 1 year: AED ${savings1Year(10000).toFixed(2)}`);
console.log(`    AED 10,000 for 5 years: AED ${savings5Year(10000).toFixed(2)}\n`);

const fd1Year = InterestCalculator.fixedDeposit(1)(true);
const fd3Year = InterestCalculator.fixedDeposit(3)(true);

console.log('  Fixed Deposit (4.5% compound interest):');
console.log(`    AED 50,000 for 1 year: AED ${fd1Year(50000).toFixed(2)}`);
console.log(`    AED 50,000 for 3 years: AED ${fd3Year(50000).toFixed(2)}\n`);

console.log('C) Transaction Processors (Partial Application):');
const intlProcessor = BankingCalculators.createInternationalTransferProcessor();

const txn1 = intlProcessor(10000);
const txn2 = intlProcessor(50000);

console.log('  International transfers (AED 50 fee, 5% tax):');
console.log(`    AED 10,000: Total = AED ${txn1.total.toFixed(2)}`);
console.log(`    AED 50,000: Total = AED ${txn2.total.toFixed(2)}\n`);

console.log('='.repeat(70) + '\n');

// Demo 5: Benefits Summary
console.log('5. BENEFITS:\n');

console.log('CURRYING Benefits:');
console.log('  ✓ Creates reusable function pipelines');
console.log('  ✓ Enables function composition');
console.log('  ✓ Better for functional programming patterns');
console.log('  ✓ Cleaner syntax with arrow functions');

console.log('\nPARTIAL APPLICATION Benefits:');
console.log('  ✓ More flexible (fix any arguments)');
console.log('  ✓ Better performance (fewer function calls)');
console.log('  ✓ More intuitive for most developers');
console.log('  ✓ Easier to create specialized functions');

console.log('\nWhen to use which:');
console.log('  • Currying: Functional composition, pipelines, mathematical operations');
console.log('  • Partial: Configuration, creating specialized versions, dependency injection');
```

### Key Differences:

| Aspect | Currying | Partial Application |
|--------|----------|---------------------|
| **Definition** | f(a, b, c) → f(a)(b)(c) | f(a, b, c) + fix(a) → f'(b, c) |
| **Arity** | Always unary functions | Any arity |
| **Flexibility** | Fixed sequence | Fix any arguments |
| **Return** | Always function (until complete) | Function with remaining args |
| **Use Case** | Composition, pipelines | Configuration, specialization |

---

## Question 21: What is the `this` keyword? How does it work in different contexts?

### Answer:

The `this` keyword refers to the context in which a function is executed. Its value depends on **how** the function is called, not where it's defined.

### Rules for `this`:

1. **Global Context**: `this` = global object (window/global)
2. **Object Method**: `this` = the object
3. **Constructor**: `this` = new instance
4. **Arrow Functions**: `this` = lexical (inherited from parent)
5. **Explicit Binding**: call/apply/bind set `this`
6. **Event Handlers**: `this` = element that received event

### Banking Scenario: Context Management at Emirates NBD

```javascript
// Global context (avoid in real code)
console.log('=== `this` Keyword - Emirates NBD ===\n');

// Demo 1: Object Methods
console.log('1. Object Method Context:\n');

const bankAccount = {
    accountNumber: 'ACC-123456789',
    balance: 50000,
    customerName: 'Ahmed Mohammed',

    // Regular method - `this` refers to bankAccount
    getBalance() {
        console.log(`Account: ${this.accountNumber}`);
        console.log(`Customer: ${this.customerName}`);
        return this.balance;
    },

    // Arrow function - `this` is lexically bound (from surrounding scope)
    getBalanceArrow: () => {
        // ⚠️ `this` is NOT bankAccount in arrow functions!
        console.log('Arrow function this:', this);
        // return this.balance; // Would be undefined
    },

    deposit(amount) {
        this.balance += amount;
        console.log(`Deposited AED ${amount}`);
        console.log(`New balance: AED ${this.balance.toLocaleString()}`);
        return this.balance;
    },

    // Method with nested function
    processTransactions(transactions) {
        console.log(`Processing ${transactions.length} transactions for ${this.accountNumber}`);
        
        // ❌ Problem: `this` is lost in nested function
        transactions.forEach(function(txn) {
            // `this` is undefined here (strict mode) or global object
            // this.balance += txn.amount; // Would fail
            console.log(`  - Transaction: ${txn.id}`);
        });

        // ✅ Solution 1: Arrow function (inherits `this`)
        transactions.forEach((txn) => {
            this.balance += txn.amount;
            console.log(`  - ${txn.id}: ${txn.type} AED ${txn.amount}`);
        });

        // ✅ Solution 2: Save `this` to variable
        const self = this;
        transactions.forEach(function(txn) {
            self.balance += txn.amount;
        });

        // ✅ Solution 3: Use bind
        transactions.forEach(function(txn) {
            this.balance += txn.amount;
        }.bind(this));
    }
};

console.log('Calling getBalance():');
const balance = bankAccount.getBalance();
console.log(`Balance: AED ${balance.toLocaleString()}\n`);

console.log('Calling deposit():');
bankAccount.deposit(10000);
console.log();

console.log('='.repeat(70) + '\n');

// Demo 2: Function Context Loss
console.log('2. Function Context Loss:\n');

const account2 = {
    accountNumber: 'ACC-987654321',
    balance: 75000,
    
    showBalance() {
        console.log(`[${this.accountNumber}] Balance: AED ${this.balance.toLocaleString()}`);
    }
};

console.log('Direct method call:');
account2.showBalance(); // ✅ `this` = account2

console.log('\nAssigned to variable (context lost):');
const showBalanceFunc = account2.showBalance;
try {
    // showBalanceFunc(); // ❌ `this` is undefined! Would throw error
    console.log('⚠️  Context lost when assigned to variable\n');
} catch (error) {
    console.log('Error:', error.message);
}

console.log('='.repeat(70) + '\n');

// Demo 3: Explicit Binding (call, apply, bind)
console.log('3. Explicit Binding (call, apply, bind):\n');

class TransactionService {
    constructor() {
        this.serviceName = 'Emirates NBD Transaction Service';
    }

    processTransfer(fromAccount, toAccount, amount) {
        console.log(`Service: ${this.serviceName}`);
        console.log(`Transfer: ${fromAccount} → ${toAccount}`);
        console.log(`Amount: AED ${amount.toLocaleString()}`);
        return {
            from: fromAccount,
            to: toAccount,
            amount,
            status: 'COMPLETED'
        };
    }

    processMultipleTransfers(transfers) {
        console.log(`Service: ${this.serviceName}`);
        console.log(`Processing ${transfers.length} transfers:`);
        transfers.forEach((t, i) => {
            console.log(`  ${i + 1}. ${t.from} → ${t.to}: AED ${t.amount}`);
        });
    }
}

const service = new TransactionService();

// call: Invoke immediately with individual arguments
console.log('Using call():');
service.processTransfer.call(
    { serviceName: 'Custom Service via call()' },
    'ACC-111',
    'ACC-222',
    5000
);

console.log('\nUsing apply():');
// apply: Invoke immediately with array of arguments
service.processTransfer.apply(
    { serviceName: 'Custom Service via apply()' },
    ['ACC-333', 'ACC-444', 10000]
);

console.log('\nUsing bind():');
// bind: Returns new function with fixed `this`
const boundProcessTransfer = service.processTransfer.bind(
    { serviceName: 'Bound Service' }
);
boundProcessTransfer('ACC-555', 'ACC-666', 15000);

console.log();
console.log('='.repeat(70) + '\n');

// Demo 4: Constructor Functions and Classes
console.log('4. Constructor Functions and Classes:\n');

// Constructor function (old way)
function BankAccount(accountNumber, initialBalance) {
    // `this` refers to the new instance
    this.accountNumber = accountNumber;
    this.balance = initialBalance;
    this.transactions = [];

    this.deposit = function(amount) {
        this.balance += amount;
        this.transactions.push({ type: 'DEPOSIT', amount });
        return this.balance;
    };
}

console.log('Constructor function:');
const acc1 = new BankAccount('ACC-CONST-001', 10000);
acc1.deposit(5000);
console.log(`Account: ${acc1.accountNumber}, Balance: AED ${acc1.balance.toLocaleString()}\n`);

// ES6 Class (modern way)
class ModernBankAccount {
    constructor(accountNumber, initialBalance) {
        // `this` refers to the new instance
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.transactions = [];
    }

    deposit(amount) {
        this.balance += amount;
        this.transactions.push({ type: 'DEPOSIT', amount });
        console.log(`Deposited AED ${amount.toLocaleString()}`);
        return this.balance;
    }

    // Arrow function as class field - `this` always bound to instance
    getAccountInfo = () => {
        return {
            accountNumber: this.accountNumber,
            balance: this.balance,
            transactionCount: this.transactions.length
        };
    };

    // Regular method
    getBalance() {
        return this.balance;
    }
}

console.log('ES6 Class:');
const acc2 = new ModernBankAccount('ACC-CLASS-001', 20000);
acc2.deposit(8000);
console.log(`Balance: AED ${acc2.getBalance().toLocaleString()}\n`);

// Arrow function maintains context
const getInfo = acc2.getAccountInfo;
console.log('Arrow function (context preserved):', getInfo());

// Regular method loses context when extracted
const getBalanceExtracted = acc2.getBalance;
try {
    // getBalanceExtracted(); // Would throw error
    console.log('⚠️  Regular method loses context when extracted\n');
} catch (error) {
    console.log('Error:', error.message);
}

console.log('='.repeat(70) + '\n');

// Demo 5: Event Handlers and Callbacks
console.log('5. Event Handlers and Callbacks:\n');

class DashboardWidget {
    constructor(accountNumber) {
        this.accountNumber = accountNumber;
        this.balance = 100000;
        this.updateCount = 0;
    }

    // ❌ Problem: Regular method loses context in callbacks
    updateBalanceRegular() {
        this.updateCount++;
        console.log(`[${this.accountNumber}] Update #${this.updateCount}: AED ${this.balance.toLocaleString()}`);
    }

    // ✅ Solution 1: Arrow function (always bound to instance)
    updateBalanceArrow = () => {
        this.updateCount++;
        console.log(`[${this.accountNumber}] Update #${this.updateCount}: AED ${this.balance.toLocaleString()}`);
    };

    // ✅ Solution 2: Bind in constructor
    constructor2(accountNumber) {
        this.accountNumber = accountNumber;
        this.balance = 100000;
        this.updateCount = 0;
        this.updateBalanceRegular = this.updateBalanceRegular.bind(this);
    }

    simulateUpdates() {
        console.log('Simulating dashboard updates...\n');

        // Using arrow function (context preserved)
        console.log('With arrow function:');
        setTimeout(this.updateBalanceArrow, 100);

        // Using regular function with bind (context preserved)
        console.log('With regular function + bind:');
        setTimeout(this.updateBalanceRegular.bind(this), 200);

        // ⚠️ Using regular function without bind (context lost)
        // setTimeout(this.updateBalanceRegular, 300); // Would fail
    }
}

const widget = new DashboardWidget('ACC-WIDGET-001');
widget.simulateUpdates();

// Wait for async operations
setTimeout(() => {
    console.log('\n' + '='.repeat(70) + '\n');

    // Demo 6: Practical Patterns
    console.log('6. PRACTICAL PATTERNS:\n');

    // Pattern 1: Method chaining with `this`
    class FluentAccount {
        constructor(accountNumber) {
            this.accountNumber = accountNumber;
            this.balance = 0;
            this.transactions = [];
        }

        deposit(amount) {
            this.balance += amount;
            this.transactions.push({ type: 'DEPOSIT', amount });
            return this; // Return `this` for chaining
        }

        withdraw(amount) {
            this.balance -= amount;
            this.transactions.push({ type: 'WITHDRAWAL', amount });
            return this; // Return `this` for chaining
        }

        addNote(note) {
            this.transactions[this.transactions.length - 1].note = note;
            return this; // Return `this` for chaining
        }

        logBalance() {
            console.log(`Account ${this.accountNumber}: AED ${this.balance.toLocaleString()}`);
            return this;
        }
    }

    console.log('Pattern 1: Method Chaining');
    const fluentAcc = new FluentAccount('ACC-FLUENT-001');
    fluentAcc
        .deposit(10000)
        .addNote('Initial deposit')
        .deposit(5000)
        .withdraw(2000)
        .logBalance();

    console.log();

    // Pattern 2: Factory with closures (avoiding `this`)
    function createAccount(accountNumber, initialBalance) {
        let balance = initialBalance;
        const transactions = [];

        return {
            getAccountNumber: () => accountNumber,
            getBalance: () => balance,
            deposit: (amount) => {
                balance += amount;
                transactions.push({ type: 'DEPOSIT', amount });
                return balance;
            },
            withdraw: (amount) => {
                if (balance >= amount) {
                    balance -= amount;
                    transactions.push({ type: 'WITHDRAWAL', amount });
                    return balance;
                }
                throw new Error('Insufficient funds');
            },
            getTransactions: () => [...transactions]
        };
    }

    console.log('Pattern 2: Factory Pattern (no `this` issues)');
    const factoryAcc = createAccount('ACC-FACTORY-001', 50000);
    factoryAcc.deposit(10000);
    console.log(`Balance: AED ${factoryAcc.getBalance().toLocaleString()}`);
    console.log('✓ No context issues - uses closures instead of `this`\n');

    console.log('='.repeat(70) + '\n');

    // Summary
    console.log('7. SUMMARY:\n');

    console.log('`this` Binding Rules:');
    console.log('  1. Default: global object (or undefined in strict mode)');
    console.log('  2. Implicit: object before the dot (obj.method())');
    console.log('  3. Explicit: call(), apply(), bind()');
    console.log('  4. new: newly created object');
    console.log('  5. Arrow functions: lexical (from enclosing scope)');

    console.log('\nCommon Pitfalls:');
    console.log('  ❌ Extracting methods (loses context)');
    console.log('  ❌ Passing methods as callbacks (loses context)');
    console.log('  ❌ Nested functions in methods (different context)');

    console.log('\nSolutions:');
    console.log('  ✓ Arrow functions (lexical binding)');
    console.log('  ✓ bind() method');
    console.log('  ✓ Store `this` in variable (const self = this)');
    console.log('  ✓ Use call/apply for one-time binding');
    console.log('  ✓ Class fields with arrow functions');

    console.log('\nBest Practices:');
    console.log('  • Use arrow functions for callbacks and event handlers');
    console.log('  • Use bind() in constructors for methods passed as callbacks');
    console.log('  • Avoid extracting methods without binding');
    console.log('  • Consider factory pattern to avoid `this` altogether');
    console.log('  • Use method chaining by returning `this`');

}, 500);
```

### `this` Binding Priority (highest to lowest):

1. **new binding**: `new Foo()` → new object
2. **Explicit binding**: `call`, `apply`, `bind` → specified object
3. **Implicit binding**: `obj.method()` → `obj`
4. **Default binding**: standalone function → global/undefined
5. **Arrow functions**: lexical → surrounding scope

### Key Takeaways:

- `this` is determined by **how** a function is called, not where it's defined
- Arrow functions inherit `this` from their lexical scope
- Use `bind()` to permanently fix `this` to an object
- Method chaining returns `this` for fluent APIs
- Factory pattern avoids `this` issues with closures

---

**End of Questions 19-21**
