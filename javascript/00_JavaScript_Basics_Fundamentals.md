# JavaScript Basics & Fundamentals

## Table of Contents
1. [Data Types](#data-types)
2. [Variables & Scoping](#variables--scoping)
3. [Type Coercion & Conversion](#type-coercion--conversion)
4. [Operators](#operators)
5. [Conditionals & Loops](#conditionals--loops)
6. [Functions](#functions)
7. [Objects & Arrays](#objects--arrays)
8. [Event Emitter Pattern](#event-emitter-pattern)
9. [Prototypes & Inheritance](#prototypes--inheritance)
10. [this Keyword](#this-keyword)
11. [Closures](#closures)
12. [Error Handling](#error-handling)

---

## Data Types

JavaScript has **8 data types**: 7 primitive types and 1 reference type (Object).

### Primitive Data Types

```javascript
// 1. String - Text data
const name = "John";
const greeting = 'Hello';
const template = `Hello ${name}`; // Template literals

// 2. Number - Integers and floating-point numbers
const age = 25;
const price = 19.99;
const negative = -10;
const infinity = Infinity;
const notANumber = NaN;

// 3. BigInt - Large integers (beyond Number.MAX_SAFE_INTEGER)
const bigNumber = 9007199254740991n;
const anotherBig = BigInt("9007199254740991");

// 4. Boolean - true or false
const isActive = true;
const isCompleted = false;

// 5. Undefined - Variable declared but not assigned
let undefinedVar;
console.log(undefinedVar); // undefined

// 6. Null - Intentional absence of value
const emptyValue = null;

// 7. Symbol - Unique identifier (ES6)
const sym1 = Symbol('description');
const sym2 = Symbol('description');
console.log(sym1 === sym2); // false - each Symbol is unique
```

### 🏦 Banking Example: Account Balance System

```javascript
// Banking application demonstrating data types

class BankAccount {
  constructor(accountNumber, customerName, initialBalance) {
    // String: Account identifier
    this.accountNumber = accountNumber; // "ACC-123456789"
    
    // String: Customer information
    this.customerName = customerName;
    
    // Number: Financial amounts (careful with precision!)
    this.balance = initialBalance; // 1500.50
    
    // BigInt: For international transfers (large amounts in cents)
    this.balanceInCents = BigInt(Math.round(initialBalance * 100));
    
    // Boolean: Account status flags
    this.isActive = true;
    this.isVerified = false;
    this.hasFraudAlert = false;
    
    // Null: Intentional absence (no overdraft limit set)
    this.overdraftLimit = null;
    
    // Undefined: Not yet assigned
    let lastTransactionDate; // undefined until first transaction
    
    // Symbol: Unique transaction IDs
    this.accountId = Symbol('accountId');
    
    // Date object for tracking
    this.createdAt = new Date();
    this.lastActivity = null;
  }
  
  // Method demonstrating number precision issues
  deposit(amount) {
    // ⚠️ JavaScript number precision issue
    console.log(0.1 + 0.2); // 0.30000000000000004 (not 0.3!)
    
    // Better: Work with cents (integers)
    const amountInCents = Math.round(amount * 100);
    this.balanceInCents += BigInt(amountInCents);
    this.balance = Number(this.balanceInCents) / 100;
    
    this.lastActivity = new Date();
    return this.balance;
  }
  
  // Checking data types
  checkAccountData() {
    console.log('Account Number:', typeof this.accountNumber); // "string"
    console.log('Balance:', typeof this.balance);              // "number"
    console.log('Balance (cents):', typeof this.balanceInCents); // "bigint"
    console.log('Is Active:', typeof this.isActive);           // "boolean"
    console.log('Overdraft Limit:', typeof this.overdraftLimit); // "object" (null quirk)
    console.log('Is null?:', this.overdraftLimit === null);    // true
    console.log('Account ID:', typeof this.accountId);         // "symbol"
    console.log('Created At:', typeof this.createdAt);         // "object"
    console.log('Is Date?:', this.createdAt instanceof Date);  // true
  }
  
  // Handling NaN (Not a Number)
  withdraw(amount) {
    // Validate input is a number
    if (isNaN(amount)) {
      throw new Error('Invalid amount: Not a Number');
    }
    
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    
    if (amount > this.balance) {
      throw new Error('Insufficient funds');
    }
    
    const amountInCents = Math.round(amount * 100);
    this.balanceInCents -= BigInt(amountInCents);
    this.balance = Number(this.balanceInCents) / 100;
    
    return this.balance;
  }
  
  // Demonstrating Infinity
  calculateInterest(years) {
    const rate = 0.05; // 5% annual rate
    
    // Compound interest formula: A = P(1 + r)^t
    const futureValue = this.balance * Math.pow(1 + rate, years);
    
    // Handle infinity for extremely long periods
    if (futureValue === Infinity) {
      return 'Amount too large to calculate';
    }
    
    return futureValue.toFixed(2);
  }
  
  // Using BigInt for large transaction amounts
  processInternationalTransfer(amountInCents) {
    // BigInt can handle very large numbers accurately
    const transferAmount = BigInt(amountInCents);
    
    if (transferAmount > this.balanceInCents) {
      throw new Error('Insufficient funds for international transfer');
    }
    
    this.balanceInCents -= transferAmount;
    this.balance = Number(this.balanceInCents) / 100;
    
    return {
      success: true,
      newBalance: this.balance,
      transferredCents: transferAmount.toString()
    };
  }
}

// Usage Example
const account = new BankAccount('ACC-987654321', 'Alice Johnson', 5000);

console.log('=== Banking Data Types Demo ===\n');

// Check types
account.checkAccountData();

console.log('\n=== Precision Issues ===');
console.log('Standard JS addition: 0.1 + 0.2 =', 0.1 + 0.2);
console.log('Depositing $0.10:');
account.deposit(0.10);
console.log('New balance:', account.balance);
console.log('Balance in cents (accurate):', account.balanceInCents.toString());

console.log('\n=== Number Validation ===');
try {
  account.withdraw('not a number'); // Will throw error
} catch (error) {
  console.log('Error:', error.message);
}

try {
  account.withdraw('100'); // String "100" converts to number 100
  console.log('Withdrawal successful (type coercion)');
} catch (error) {
  console.log('Error:', error.message);
}

console.log('\n=== Interest Calculation ===');
console.log('Balance in 10 years:', account.calculateInterest(10));
console.log('Balance in 1000 years:', account.calculateInterest(1000)); // Might return Infinity

console.log('\n=== BigInt for Large Transactions ===');
// Transfer $50,000.00 (5,000,000 cents)
const result = account.processInternationalTransfer(5000000n);
console.log('Transfer result:', result);

console.log('\n=== Null vs Undefined ===');
console.log('Overdraft limit:', account.overdraftLimit);        // null (intentionally not set)
console.log('Non-existent property:', account.creditScore);     // undefined (doesn't exist)
console.log('Is overdraft null?', account.overdraftLimit === null);
console.log('Is creditScore undefined?', account.creditScore === undefined);

// Practical Banking Considerations
console.log('\n=== Practical Banking Tips ===');
console.log('✅ Use integers (cents) for accurate money calculations');
console.log('✅ Use BigInt for large international transfers');
console.log('✅ Validate all numeric inputs with isNaN()');
console.log('✅ Use null for "not set" vs undefined for "doesn\'t exist"');
console.log('✅ Always round money calculations: Math.round(amount * 100) / 100');
console.log('❌ Never use floating-point for money: 0.1 + 0.2 !== 0.3');
```

### Key Banking Takeaways

1. **Money Precision**: Always use integers (cents) or BigInt for financial calculations
   ```javascript
   // ❌ Wrong - floating point errors
   let balance = 10.50;
   balance += 0.1 + 0.2; // 10.80000000000000004
   
   // ✅ Correct - use cents
   let balanceInCents = 1050; // $10.50
   balanceInCents += 30;      // +$0.30
   const balance = balanceInCents / 100; // $10.80
   ```

2. **BigInt for Large Transfers**: International transfers can exceed `Number.MAX_SAFE_INTEGER`
   ```javascript
   const largeTransfer = 9007199254740992; // Loses precision!
   const accurateTransfer = 9007199254740992n; // ✅ Accurate with BigInt
   ```

3. **Type Validation**: Always validate financial inputs
   ```javascript
   function processPayment(amount) {
     if (typeof amount !== 'number' || isNaN(amount)) {
       throw new Error('Invalid amount');
     }
     // Process payment
   }
   ```

4. **Null vs Undefined in Banking**:
   - `null`: Customer opted out of overdraft protection (intentional)
   - `undefined`: Credit score not yet fetched from bureau (doesn't exist yet)

### Reference Type

```javascript
// 8. Object - Collection of key-value pairs
const person = {
  name: "Alice",
  age: 30,
  address: {
    city: "New York",
    country: "USA"
  }
};

// Arrays (special type of Object)
const numbers = [1, 2, 3, 4, 5];
const mixed = [1, "hello", true, null, { key: "value" }];

// Functions (special type of Object)
function greet() {
  return "Hello!";
}

// Date (special type of Object)
const now = new Date();

// RegExp (special type of Object)
const pattern = /[a-z]+/gi;
```

### Type Checking

```javascript
// typeof operator
console.log(typeof "hello");        // "string"
console.log(typeof 42);             // "number"
console.log(typeof true);           // "boolean"
console.log(typeof undefined);      // "undefined"
console.log(typeof null);           // "object" (JavaScript quirk!)
console.log(typeof Symbol());       // "symbol"
console.log(typeof 100n);           // "bigint"
console.log(typeof {});             // "object"
console.log(typeof []);             // "object"
console.log(typeof function() {}); // "function"

// Better type checking for arrays and null
Array.isArray([]);                 // true
Array.isArray({});                 // false

const value = null;
value === null;                    // true

// instanceof for complex types
const date = new Date();
console.log(date instanceof Date); // true
console.log(date instanceof Object); // true
```

---

## Variables & Scoping

### Variable Declarations

```javascript
// var - Function scoped, can be redeclared and updated (avoid using)
var x = 10;
var x = 20; // Redeclaration allowed
x = 30;     // Update allowed

// let - Block scoped, cannot be redeclared, can be updated (ES6)
let y = 10;
// let y = 20; // ❌ Error: Cannot redeclare
y = 20;        // ✅ Update allowed

// const - Block scoped, cannot be redeclared or reassigned (ES6)
const z = 10;
// const z = 20; // ❌ Error: Cannot redeclare
// z = 20;       // ❌ Error: Cannot reassign

// Note: const objects/arrays can have their contents modified
const user = { name: "John" };
user.name = "Jane";  // ✅ Allowed
user.age = 30;       // ✅ Allowed
// user = {};        // ❌ Error: Cannot reassign

const arr = [1, 2, 3];
arr.push(4);         // ✅ Allowed
arr[0] = 10;         // ✅ Allowed
// arr = [];         // ❌ Error: Cannot reassign
```

### 🏦 Banking Example: Transaction Processing with Proper Scoping

```javascript
// Banking Transaction System demonstrating variable scoping

class TransactionProcessor {
  constructor() {
    // const for configuration (never changes)
    const MAX_DAILY_LIMIT = 10000;
    const MIN_TRANSACTION_AMOUNT = 0.01;
    
    // Using const for objects that won't be reassigned
    this.config = {
      maxDailyLimit: MAX_DAILY_LIMIT,
      minAmount: MIN_TRANSACTION_AMOUNT,
      supportedCurrencies: ['USD', 'EUR', 'GBP']
    };
    
    // let for values that will change
    this.transactions = [];
    this.dailyTotal = 0;
  }
  
  // Demonstrating block scope
  processTransaction(amount, type, category) {
    // const for values that won't change in this function
    const transactionId = `TXN-${Date.now()}`;
    const timestamp = new Date();
    
    // let for values that may change
    let status = 'pending';
    let fee = 0;
    
    // Block scope example
    if (type === 'international') {
      // Variables declared here are only accessible in this block
      const exchangeRate = 1.1;
      const baseAmount = amount;
      let convertedAmount = baseAmount * exchangeRate;
      
      // Apply international transfer fee
      const feePercentage = 0.03; // 3%
      fee = convertedAmount * feePercentage;
      convertedAmount += fee;
      
      console.log(`Converting $${baseAmount} to $${convertedAmount.toFixed(2)}`);
      // exchangeRate is only accessible here
    }
    // console.log(exchangeRate); // ❌ Error: exchangeRate not defined (block scoped)
    
    // Validation using block scope
    if (amount < this.config.minAmount) {
      const errorMessage = `Amount below minimum: $${this.config.minAmount}`;
      throw new Error(errorMessage);
      // errorMessage is only accessible in this block
    }
    
    // Check daily limit
    if (this.dailyTotal + amount > this.config.maxDailyLimit) {
      const remaining = this.config.maxDailyLimit - this.dailyTotal;
      throw new Error(`Daily limit exceeded. Remaining: $${remaining.toFixed(2)}`);
    }
    
    // Process transaction
    status = 'completed';
    this.dailyTotal += amount;
    
    // Create transaction record (const object, but properties can be modified)
    const transaction = {
      id: transactionId,
      amount,
      fee,
      type,
      category,
      status,
      timestamp
    };
    
    this.transactions.push(transaction);
    return transaction;
  }
  
  // Demonstrating function scope vs block scope
  generateDailyReport() {
    // var is function scoped (avoid this!)
    var totalDeposits = 0;    // ❌ Bad practice
    let totalWithdrawals = 0; // ✅ Good practice
    const reportDate = new Date(); // ✅ Won't change
    
    // Loop demonstrating scoping issues with var
    for (var i = 0; i < this.transactions.length; i++) {
      const txn = this.transactions[i];
      
      if (txn.type === 'deposit') {
        totalDeposits += txn.amount;
      } else {
        totalWithdrawals += txn.amount;
      }
    }
    
    console.log(i); // ⚠️ i is accessible here with var (function scoped)
    // This is why we should use let instead
    
    // Better loop with let
    for (let j = 0; j < this.transactions.length; j++) {
      // j is block scoped
    }
    // console.log(j); // ❌ Error: j not defined (block scoped with let)
    
    return {
      date: reportDate,
      totalDeposits,
      totalWithdrawals,
      netChange: totalDeposits - totalWithdrawals,
      transactionCount: this.transactions.length
    };
  }
  
  // Demonstrating hoisting issues
  checkBalance(accountId) {
    // ❌ Bad: var is hoisted (value is undefined until assignment)
    console.log(balance); // undefined (var is hoisted)
    var balance = 1000;
    console.log(balance); // 1000
    
    // ✅ Better: let/const throw error if accessed before declaration
    // console.log(amount); // ❌ ReferenceError: Cannot access before initialization
    let amount = 500;
    console.log(amount); // 500
    
    return { balance, amount };
  }
  
  // Closure example: Maintaining private state
  createAccount(initialBalance) {
    // Private variables (closure)
    let balance = initialBalance;
    const accountNumber = `ACC-${Math.random().toString(36).substr(2, 9)}`;
    let transactionHistory = [];
    
    // Return public interface
    return {
      deposit(amount) {
        if (amount <= 0) {
          throw new Error('Amount must be positive');
        }
        balance += amount;
        transactionHistory.push({ type: 'deposit', amount, balance, date: new Date() });
        return balance;
      },
      
      withdraw(amount) {
        if (amount > balance) {
          throw new Error('Insufficient funds');
        }
        balance -= amount;
        transactionHistory.push({ type: 'withdraw', amount, balance, date: new Date() });
        return balance;
      },
      
      getBalance() {
        return balance; // Access to private variable via closure
      },
      
      getAccountNumber() {
        return accountNumber;
      },
      
      getHistory() {
        // Return copy to prevent external modification
        return [...transactionHistory];
      }
    };
  }
}

// Usage Examples
console.log('=== Transaction Processing with Scoping ===\n');

const processor = new TransactionProcessor();

// Process various transactions
try {
  const txn1 = processor.processTransaction(1500, 'domestic', 'groceries');
  console.log('✅ Transaction 1:', txn1.id, '-', `$${txn1.amount}`);
  
  const txn2 = processor.processTransaction(3000, 'international', 'travel');
  console.log('✅ Transaction 2:', txn2.id, '-', `$${txn2.amount}`, `(Fee: $${txn2.fee.toFixed(2)})`);
  
  // This will fail - below minimum
  const txn3 = processor.processTransaction(0.001, 'domestic', 'test');
} catch (error) {
  console.log('❌ Transaction 3 failed:', error.message);
}

console.log('\n=== Daily Report ===');
const report = processor.generateDailyReport();
console.log('Total Deposits:', `$${report.totalDeposits.toFixed(2)}`);
console.log('Total Withdrawals:', `$${report.totalWithdrawals.toFixed(2)}`);
console.log('Net Change:', `$${report.netChange.toFixed(2)}`);

console.log('\n=== Private Account with Closures ===');
const account = processor.createAccount(5000);
console.log('Account Number:', account.getAccountNumber());
console.log('Initial Balance:', `$${account.getBalance()}`);

account.deposit(1000);
console.log('After deposit:', `$${account.getBalance()}`);

account.withdraw(500);
console.log('After withdrawal:', `$${account.getBalance()}`);

// Cannot access private variables
console.log('Can access balance directly?', account.balance); // undefined
console.log('Transaction history:', account.getHistory().length, 'transactions');

// Demonstrating the problem with var in loops
console.log('\n=== var vs let in Loops (Common Interview Question) ===');

// Problem with var
console.log('Using var (all functions reference same i):');
const badTimeouts = [];
for (var i = 1; i <= 3; i++) {
  badTimeouts.push(() => console.log(`Transaction ${i}`));
}
badTimeouts.forEach(fn => fn()); // Prints "Transaction 4" three times!

// Fixed with let
console.log('\nUsing let (each function has its own i):');
const goodTimeouts = [];
for (let i = 1; i <= 3; i++) {
  goodTimeouts.push(() => console.log(`Transaction ${i}`));
}
goodTimeouts.forEach(fn => fn()); // Prints "Transaction 1", "Transaction 2", "Transaction 3"

// Practical banking scenario
console.log('\n=== Real Banking Scenario: Batch Processing ===');

function processBatchTransactions(transactions) {
  // Use const for values that won't be reassigned
  const batchId = `BATCH-${Date.now()}`;
  const startTime = Date.now();
  
  // Use let for counters and accumulators
  let successCount = 0;
  let failCount = 0;
  let totalAmount = 0;
  
  // Use const in loop (each iteration gets new binding)
  for (const transaction of transactions) {
    try {
      // Validate transaction (const for immutable check)
      const isValid = transaction.amount > 0 && transaction.amount <= 10000;
      
      if (!isValid) {
        throw new Error('Invalid transaction amount');
      }
      
      // Process (let for mutable state)
      let processedAmount = transaction.amount;
      
      // Apply business rules in block scope
      if (transaction.type === 'international') {
        const conversionFee = processedAmount * 0.03;
        processedAmount += conversionFee;
      }
      
      totalAmount += processedAmount;
      successCount++;
      
    } catch (error) {
      failCount++;
      console.log(`❌ Failed: ${transaction.id} - ${error.message}`);
    }
  }
  
  const endTime = Date.now();
  const duration = endTime - startTime;
  
  return {
    batchId,
    duration: `${duration}ms`,
    successCount,
    failCount,
    totalAmount: totalAmount.toFixed(2),
    successRate: `${((successCount / transactions.length) * 100).toFixed(1)}%`
  };
}

const testTransactions = [
  { id: 'T1', amount: 100, type: 'domestic' },
  { id: 'T2', amount: 500, type: 'international' },
  { id: 'T3', amount: 15000, type: 'domestic' }, // Will fail
  { id: 'T4', amount: 250, type: 'domestic' }
];

const batchResult = processBatchTransactions(testTransactions);
console.log('Batch Result:', batchResult);
```

### Key Banking Scoping Takeaways

1. **Use `const` by default** for:
   - Configuration values: `const MAX_DAILY_LIMIT = 10000`
   - Transaction IDs: `const transactionId = generateId()`
   - Objects that won't be reassigned: `const config = {}`

2. **Use `let` for** values that change:
   - Counters: `let balance = 0`
   - Loop variables: `for (let i = 0; i < length; i++)`
   - Accumulator variables: `let totalAmount = 0`

3. **Avoid `var`** in modern code:
   - Function-scoped (causes bugs in loops)
   - Hoisted (can access before declaration = undefined)
   - Can be redeclared (error-prone)

4. **Block scope** prevents bugs:
   ```javascript
   if (amount > 1000) {
     const highValueFee = amount * 0.05;
     // highValueFee only exists in this block
   }
   // console.log(highValueFee); // ❌ Error (prevents accidental access)
   ```

5. **Closures for private data**:
   ```javascript
   function createSecureAccount(balance) {
     // Private variable (not accessible outside)
     let accountBalance = balance;
     
     return {
       getBalance() { return accountBalance; },
       deposit(amt) { accountBalance += amt; }
     };
   }
   ```

### Scoping

```javascript
// Global Scope
const globalVar = "I'm global";

function outerFunction() {
  // Function Scope
  var functionVar = "I'm function scoped";
  
  if (true) {
    // Block Scope
    let blockVar = "I'm block scoped";
    const blockConst = "I'm also block scoped";
    var notBlockScoped = "I'm function scoped, not block!";
    
    console.log(blockVar);        // ✅ Accessible
    console.log(functionVar);     // ✅ Accessible
  }
  
  // console.log(blockVar);         // ❌ Error: blockVar not defined
  console.log(notBlockScoped);     // ✅ Accessible (var is function scoped)
  console.log(functionVar);        // ✅ Accessible
}

// Hoisting
console.log(hoistedVar);  // undefined (var is hoisted)
var hoistedVar = "I'm hoisted";

// console.log(notHoisted); // ❌ Error: Cannot access before initialization
let notHoisted = "I'm not hoisted the same way";

// Function hoisting
sayHello(); // ✅ Works! Function declarations are fully hoisted

function sayHello() {
  console.log("Hello!");
}

// sayGoodbye(); // ❌ Error: Cannot access before initialization
const sayGoodbye = function() {
  console.log("Goodbye!");
};
```

### Temporal Dead Zone (TDZ)

```javascript
// The time between entering scope and variable declaration
{
  // TDZ starts
  // console.log(value); // ❌ ReferenceError: Cannot access before initialization
  
  let value = 42; // TDZ ends
  console.log(value); // ✅ 42
}
```

---

## Type Coercion & Conversion

### Implicit Coercion (Automatic)

```javascript
// String coercion
console.log("5" + 3);        // "53" (number to string)
console.log("Hello" + true); // "Hellotrue"

// Number coercion
console.log("5" - 3);        // 2 (string to number)
console.log("5" * "2");      // 10
console.log("10" / "2");     // 5
console.log("10" % "3");     // 1

// Boolean coercion
console.log(true + true);    // 2 (true = 1, false = 0)
console.log(true + false);   // 1

// Comparison coercion
console.log("5" == 5);       // true (loose equality)
console.log("5" === 5);      // false (strict equality)
console.log(null == undefined);  // true
console.log(null === undefined); // false
```

### Explicit Conversion

```javascript
// To String
String(123);                 // "123"
String(true);                // "true"
String(null);                // "null"
String(undefined);           // "undefined"
(123).toString();            // "123"
123 + "";                    // "123"

// To Number
Number("123");               // 123
Number("123.45");            // 123.45
Number("123abc");            // NaN
Number(true);                // 1
Number(false);               // 0
Number(null);                // 0
Number(undefined);           // NaN
parseInt("123");             // 123
parseInt("123.45");          // 123 (removes decimal)
parseFloat("123.45");        // 123.45
+"123";                      // 123 (unary plus)

// To Boolean
Boolean(1);                  // true
Boolean(0);                  // false
Boolean("");                 // false
Boolean("hello");            // true
Boolean(null);               // false
Boolean(undefined);          // false
Boolean({});                 // true
Boolean([]);                 // true
!!value;                     // Double negation converts to boolean
```

### Truthy and Falsy Values

```javascript
// Falsy values (only 8 in JavaScript)
false
0
-0
0n (BigInt zero)
"" (empty string)
null
undefined
NaN

// Everything else is truthy!
true
42
"0"
"false"
[]
{}
function() {}
```

---

## Operators

### Arithmetic Operators

```javascript
let a = 10, b = 3;

console.log(a + b);   // 13 (Addition)
console.log(a - b);   // 7  (Subtraction)
console.log(a * b);   // 30 (Multiplication)
console.log(a / b);   // 3.333... (Division)
console.log(a % b);   // 1  (Modulus/Remainder)
console.log(a ** b);  // 1000 (Exponentiation - ES7)

// Increment/Decrement
let x = 5;
console.log(x++);     // 5 (post-increment, returns then increments)
console.log(x);       // 6
console.log(++x);     // 7 (pre-increment, increments then returns)
console.log(x--);     // 7 (post-decrement)
console.log(--x);     // 5 (pre-decrement)
```

### Comparison Operators

```javascript
console.log(5 == "5");    // true (loose equality)
console.log(5 === "5");   // false (strict equality)
console.log(5 != "5");    // false
console.log(5 !== "5");   // true

console.log(5 > 3);       // true
console.log(5 < 3);       // false
console.log(5 >= 5);      // true
console.log(5 <= 3);      // false
```

### Logical Operators

```javascript
// AND (&&) - Returns first falsy value or last value
console.log(true && true);     // true
console.log(true && false);    // false
console.log("hello" && 42);    // 42
console.log(0 && "hello");     // 0

// OR (||) - Returns first truthy value or last value
console.log(true || false);    // true
console.log(false || false);   // false
console.log(0 || "hello");     // "hello"
console.log("" || 0);          // 0

// NOT (!) - Inverts boolean
console.log(!true);            // false
console.log(!false);           // true
console.log(!"hello");         // false
console.log(!0);               // true

// Nullish Coalescing (??) - Returns right side if left is null/undefined
console.log(null ?? "default");      // "default"
console.log(undefined ?? "default"); // "default"
console.log(0 ?? "default");         // 0 (not null/undefined)
console.log("" ?? "default");        // "" (not null/undefined)
```

### Other Operators

```javascript
// Ternary Operator
const age = 18;
const canVote = age >= 18 ? "Yes" : "No";

// Optional Chaining (?.)
const user = { name: "John", address: { city: "NY" } };
console.log(user?.address?.city);        // "NY"
console.log(user?.contact?.phone);       // undefined (no error)

// Spread Operator (...)
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];            // [1, 2, 3, 4, 5]

const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 };          // { a: 1, b: 2, c: 3 }

// Rest Parameters
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3, 4));            // 10
```

---

## Conditionals & Loops

### Conditionals

```javascript
// if...else if...else
const score = 85;

if (score >= 90) {
  console.log("A");
} else if (score >= 80) {
  console.log("B");
} else if (score >= 70) {
  console.log("C");
} else {
  console.log("F");
}

// switch statement
const day = "Monday";

switch (day) {
  case "Monday":
    console.log("Start of the week");
    break;
  case "Friday":
    console.log("End of the week");
    break;
  case "Saturday":
  case "Sunday":
    console.log("Weekend!");
    break;
  default:
    console.log("Midweek");
}
```

### Loops

```javascript
// for loop
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// while loop
let count = 0;
while (count < 5) {
  console.log(count);
  count++;
}

// do...while loop (executes at least once)
let num = 0;
do {
  console.log(num);
  num++;
} while (num < 5);

// for...of (iterates over values)
const fruits = ["apple", "banana", "orange"];
for (const fruit of fruits) {
  console.log(fruit);
}

// for...in (iterates over keys/indices)
const person = { name: "John", age: 30 };
for (const key in person) {
  console.log(`${key}: ${person[key]}`);
}

// Array methods (forEach, map, filter, etc.)
fruits.forEach((fruit, index) => {
  console.log(`${index}: ${fruit}`);
});

// break and continue
for (let i = 0; i < 10; i++) {
  if (i === 3) continue;  // Skip 3
  if (i === 7) break;     // Stop at 7
  console.log(i);         // 0, 1, 2, 4, 5, 6
}
```

---

## Functions

### Function Declaration

```javascript
// Regular function
function add(a, b) {
  return a + b;
}

// Function with default parameters
function greet(name = "Guest") {
  return `Hello, ${name}!`;
}

// Function with rest parameters
function sum(...numbers) {
  return numbers.reduce((total, num) => total + num, 0);
}
```

### 🏦 Banking Example: Payment Processing Functions

```javascript
// Banking Payment System demonstrating different function patterns

// 1. FUNCTION DECLARATIONS - Traditional approach
function processPayment(amount, accountNumber) {
  if (amount <= 0) {
    throw new Error('Amount must be positive');
  }
  
  const transactionId = `PAY-${Date.now()}`;
  const fee = calculateFee(amount);
  const totalAmount = amount + fee;
  
  return {
    transactionId,
    amount,
    fee,
    totalAmount,
    accountNumber,
    status: 'completed',
    timestamp: new Date()
  };
}

// Function with default parameters
function createTransferRequest(from, to, amount, currency = 'USD', priority = 'normal') {
  // Default parameters make API calls more flexible
  return {
    from,
    to,
    amount,
    currency,
    priority,
    requestId: `REQ-${Date.now()}`,
    createdAt: new Date()
  };
}

// Function with rest parameters (unlimited arguments)
function calculateTotalBalance(...accounts) {
  // Accepts any number of account objects
  return accounts.reduce((total, account) => {
    return total + (account.balance || 0);
  }, 0);
}

// 2. FUNCTION EXPRESSIONS
const validateAccount = function(accountNumber) {
  // Named function expression
  const pattern = /^ACC-\d{9}$/;
  return pattern.test(accountNumber);
};

const calculateFee = function(amount) {
  // Fee structure
  if (amount < 100) return 0;
  if (amount < 1000) return amount * 0.01; // 1%
  if (amount < 10000) return amount * 0.005; // 0.5%
  return 50; // Flat fee for large amounts
};

// 3. ARROW FUNCTIONS
// Concise syntax for simple operations
const formatCurrency = (amount) => `$${amount.toFixed(2)}`;

const isLargeTransaction = (amount) => amount > 10000;

const applyDiscount = (amount, percentage) => amount * (1 - percentage / 100);

// Arrow function with multiple statements
const processRefund = (transactionId, amount, reason) => {
  const refundId = `REF-${Date.now()}`;
  const processingFee = amount * 0.01;
  const refundAmount = amount - processingFee;
  
  return {
    refundId,
    transactionId,
    requestedAmount: amount,
    processingFee,
    refundAmount,
    reason,
    status: 'pending'
  };
};

// 4. HIGHER-ORDER FUNCTIONS
// Functions that take other functions as arguments

function executeWithRetry(operation, maxAttempts = 3) {
  // Takes a function and executes it with retry logic
  let attempts = 0;
  
  while (attempts < maxAttempts) {
    try {
      return operation();
    } catch (error) {
      attempts++;
      if (attempts >= maxAttempts) {
        throw new Error(`Failed after ${maxAttempts} attempts: ${error.message}`);
      }
      console.log(`Attempt ${attempts} failed, retrying...`);
    }
  }
}

function applyTransactionFee(feeCalculator) {
  // Returns a function that applies a custom fee calculator
  return function(transaction) {
    const fee = feeCalculator(transaction.amount);
    return {
      ...transaction,
      fee,
      totalAmount: transaction.amount + fee
    };
  };
}

// 5. CALLBACK FUNCTIONS
function fetchAccountData(accountId, onSuccess, onError) {
  // Simulate async operation
  setTimeout(() => {
    const success = Math.random() > 0.2; // 80% success rate
    
    if (success) {
      const data = {
        accountId,
        balance: Math.random() * 10000,
        currency: 'USD',
        status: 'active'
      };
      onSuccess(data);
    } else {
      onError(new Error('Failed to fetch account data'));
    }
  }, 1000);
}

// 6. IIFE (Immediately Invoked Function Expression)
const bankingModule = (function() {
  // Private variables
  let transactionCount = 0;
  const MAX_DAILY_TRANSACTIONS = 100;
  
  // Private function
  function incrementCounter() {
    transactionCount++;
  }
  
  // Public API
  return {
    processTransaction(amount) {
      if (transactionCount >= MAX_DAILY_TRANSACTIONS) {
        throw new Error('Daily transaction limit reached');
      }
      incrementCounter();
      return {
        id: `TXN-${transactionCount}`,
        amount,
        status: 'completed'
      };
    },
    
    getTransactionCount() {
      return transactionCount;
    },
    
    resetCounter() {
      transactionCount = 0;
    }
  };
})();

// 7. FUNCTION COMPOSITION
// Build complex operations from simple functions

const addFee = (amount) => amount * 1.02; // 2% fee
const applyTax = (amount) => amount * 1.05; // 5% tax
const roundAmount = (amount) => Math.round(amount * 100) / 100;

// Compose functions
const calculateFinalAmount = (amount) => {
  return roundAmount(applyTax(addFee(amount)));
};

// Or use a compose helper
const compose = (...fns) => (value) => fns.reduceRight((acc, fn) => fn(acc), value);

const calculateTotal = compose(
  roundAmount,
  applyTax,
  addFee
);

// 8. CURRYING
// Transform a function with multiple arguments into a sequence of functions

function createTransfer(fromAccount) {
  return function(toAccount) {
    return function(amount) {
      return {
        from: fromAccount,
        to: toAccount,
        amount,
        id: `TRANSFER-${Date.now()}`
      };
    };
  };
}

// Arrow function version
const createTransferArrow = (fromAccount) => (toAccount) => (amount) => ({
  from: fromAccount,
  to: toAccount,
  amount,
  id: `TRANSFER-${Date.now()}`
});

// 9. MEMOIZATION (Caching function results)
function memoize(fn) {
  const cache = new Map();
  
  return function(...args) {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      console.log('Returning cached result');
      return cache.get(key);
    }
    
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// Expensive calculation
const calculateCompoundInterest = memoize((principal, rate, years) => {
  console.log('Calculating...');
  return principal * Math.pow(1 + rate, years);
});

// Usage Examples
console.log('=== Banking Functions Demo ===\n');

// Basic function usage
console.log('1. Process Payment:');
const payment = processPayment(500, 'ACC-123456789');
console.log(payment);

console.log('\n2. Transfer Request with Defaults:');
const transfer1 = createTransferRequest('ACC-111', 'ACC-222', 1000);
console.log(transfer1);
const transfer2 = createTransferRequest('ACC-111', 'ACC-222', 1000, 'EUR', 'high');
console.log(transfer2);

console.log('\n3. Calculate Total Balance (Rest Parameters):');
const accounts = [
  { id: 1, balance: 1000 },
  { id: 2, balance: 2500 },
  { id: 3, balance: 750 }
];
const totalBalance = calculateTotalBalance(...accounts);
console.log(`Total: ${formatCurrency(totalBalance)}`);

console.log('\n4. Arrow Functions:');
console.log('Format:', formatCurrency(1234.567));
console.log('Is large?', isLargeTransaction(15000));
console.log('After discount:', formatCurrency(applyDiscount(1000, 10)));

console.log('\n5. Higher-Order Function (Retry Logic):');
let attempts = 0;
const unreliableOperation = () => {
  attempts++;
  if (attempts < 3) throw new Error('Temporary failure');
  return 'Success!';
};

try {
  const result = executeWithRetry(unreliableOperation);
  console.log('Result:', result);
} catch (error) {
  console.log('Error:', error.message);
}

console.log('\n6. Custom Fee Calculator:');
const premiumFeeCalculator = (amount) => amount * 0.005; // 0.5%
const applyPremiumFee = applyTransactionFee(premiumFeeCalculator);

const txn = { amount: 1000, type: 'transfer' };
const txnWithFee = applyPremiumFee(txn);
console.log('Transaction:', txnWithFee);

console.log('\n7. Callback Function (Async):');
fetchAccountData('ACC-999', 
  (data) => {
    console.log('✅ Account data:', data);
  },
  (error) => {
    console.log('❌ Error:', error.message);
  }
);

console.log('\n8. IIFE Module:');
console.log('Process 3 transactions...');
bankingModule.processTransaction(100);
bankingModule.processTransaction(200);
bankingModule.processTransaction(300);
console.log('Transaction count:', bankingModule.getTransactionCount());

console.log('\n9. Function Composition:');
const baseAmount = 100;
console.log(`Base: ${formatCurrency(baseAmount)}`);
console.log(`With fee: ${formatCurrency(addFee(baseAmount))}`);
console.log(`With fee + tax: ${formatCurrency(calculateFinalAmount(baseAmount))}`);

console.log('\n10. Currying:');
const fromAccount123 = createTransferArrow('ACC-123');
const toAccount456 = fromAccount123('ACC-456');
const transfer = toAccount456(500);
console.log('Transfer:', transfer);

// Or chain it
const quickTransfer = createTransferArrow('ACC-111')('ACC-222')(250);
console.log('Quick transfer:', quickTransfer);

console.log('\n11. Memoization (Performance):');
console.log('First call:');
console.log(calculateCompoundInterest(1000, 0.05, 10));
console.log('Second call (cached):');
console.log(calculateCompoundInterest(1000, 0.05, 10));
console.log('Different args:');
console.log(calculateCompoundInterest(2000, 0.05, 10));
```

### Real-World Banking Function Patterns

```javascript
// Complete payment processing system

class PaymentProcessor {
  constructor() {
    this.processors = new Map();
    this.middleware = [];
  }
  
  // Register payment method handler
  registerProcessor(type, handler) {
    this.processors.set(type, handler);
  }
  
  // Add middleware (functions that run before/after payment)
  use(middlewareFn) {
    this.middleware.push(middlewareFn);
  }
  
  // Process payment through middleware chain
  async process(paymentData) {
    let data = { ...paymentData };
    
    // Run middleware (before processing)
    for (const middleware of this.middleware) {
      data = await middleware(data);
    }
    
    // Get appropriate processor
    const processor = this.processors.get(data.type);
    if (!processor) {
      throw new Error(`No processor for type: ${data.type}`);
    }
    
    // Process payment
    const result = await processor(data);
    
    return result;
  }
}

// Payment processor setup
const paymentSystem = new PaymentProcessor();

// Register processors (each payment type has its own handler)
paymentSystem.registerProcessor('credit_card', async (data) => {
  return {
    ...data,
    status: 'completed',
    processor: 'Stripe',
    transactionId: `CC-${Date.now()}`
  };
});

paymentSystem.registerProcessor('bank_transfer', async (data) => {
  return {
    ...data,
    status: 'pending',
    processor: 'ACH',
    transactionId: `ACH-${Date.now()}`
  };
});

// Add middleware (logging, validation, fraud detection)
paymentSystem.use(async (data) => {
  console.log('🔍 Validating payment:', data.amount);
  if (data.amount <= 0) {
    throw new Error('Invalid amount');
  }
  return data;
});

paymentSystem.use(async (data) => {
  console.log('🛡️  Fraud check:', data.amount);
  if (data.amount > 10000) {
    data.requiresVerification = true;
  }
  return data;
});

paymentSystem.use(async (data) => {
  console.log('💰 Calculating fees:', data.amount);
  data.fee = data.amount * 0.029 + 0.30; // 2.9% + $0.30
  data.total = data.amount + data.fee;
  return data;
});

// Usage
console.log('\n=== Payment Processor System ===\n');

(async () => {
  try {
    const payment1 = await paymentSystem.process({
      type: 'credit_card',
      amount: 500,
      customer: 'CUST-123'
    });
    console.log('✅ Payment 1:', payment1);
    
    const payment2 = await paymentSystem.process({
      type: 'bank_transfer',
      amount: 15000,
      customer: 'CUST-456'
    });
    console.log('✅ Payment 2:', payment2);
  } catch (error) {
    console.log('❌ Error:', error.message);
  }
})();
```

### Key Function Takeaways for Banking

1. **Use arrow functions** for simple operations (formatters, validators)
2. **Use function declarations** for main business logic (processPayment)
3. **Default parameters** make APIs flexible (optional currency, priority)
4. **Higher-order functions** enable reusable patterns (retry logic, middleware)
5. **Currying** creates specialized functions (createTransferFrom(account))
6. **Memoization** caches expensive calculations (compound interest)
7. **Function composition** builds complex operations from simple ones
8. **IIFE** creates private scope for modules (encapsulation)
9. **Callbacks** handle async operations (fetchAccountData)
10. **Middleware pattern** chains processing steps (validation → fraud → fees)

### Function Expression

```javascript
// Named function expression
const multiply = function mult(a, b) {
  return a * b;
};

// Anonymous function expression
const subtract = function(a, b) {
  return a - b;
};
```

### Arrow Functions

```javascript
// Basic arrow function
const divide = (a, b) => a / b;

// With single parameter (parentheses optional)
const square = x => x * x;

// With multiple statements (need curly braces and return)
const calculate = (a, b) => {
  const sum = a + b;
  const product = a * b;
  return { sum, product };
};

// Implicit return of object (need parentheses)
const createUser = (name, age) => ({ name, age });
```

### Immediately Invoked Function Expression (IIFE)

```javascript
// IIFE - Executes immediately
(function() {
  console.log("I run immediately!");
})();

// IIFE with parameters
(function(name) {
  console.log(`Hello, ${name}!`);
})("John");

// Arrow function IIFE
(() => {
  console.log("Arrow IIFE");
})();
```

### Higher-Order Functions

```javascript
// Function that takes another function as argument
function executeOperation(a, b, operation) {
  return operation(a, b);
}

const result = executeOperation(5, 3, (x, y) => x + y); // 8

// Function that returns a function
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
```

### Callback Functions

```javascript
// Synchronous callback
function processArray(arr, callback) {
  const result = [];
  for (const item of arr) {
    result.push(callback(item));
  }
  return result;
}

const numbers = [1, 2, 3, 4, 5];
const squared = processArray(numbers, x => x * x);

// Asynchronous callback
function fetchData(callback) {
  setTimeout(() => {
    callback({ data: "Some data" });
  }, 1000);
}

fetchData((result) => {
  console.log(result);
});
```

---

## Objects & Arrays

### Objects

```javascript
// Object creation
const person = {
  name: "Alice",
  age: 30,
  city: "New York",
  greet: function() {
    return `Hello, I'm ${this.name}`;
  }
};

// Accessing properties
console.log(person.name);           // Dot notation
console.log(person["age"]);         // Bracket notation

// Adding/Modifying properties
person.email = "alice@example.com";
person.age = 31;

// Deleting properties
delete person.city;

// Checking property existence
console.log("name" in person);      // true
console.log(person.hasOwnProperty("age")); // true

// Object methods
const keys = Object.keys(person);           // Array of keys
const values = Object.values(person);       // Array of values
const entries = Object.entries(person);     // Array of [key, value] pairs

// Object destructuring
const { name, age } = person;
const { name: userName, age: userAge } = person; // Rename

// Object spread
const newPerson = { ...person, country: "USA" };

// Object.assign (copy/merge objects)
const merged = Object.assign({}, person, { role: "developer" });

// Object.freeze (make immutable)
const frozen = Object.freeze({ value: 42 });
// frozen.value = 100; // Silently fails (or throws error in strict mode)

// Object.seal (prevent adding/removing properties)
const sealed = Object.seal({ value: 42 });
sealed.value = 100; // ✅ Allowed
// sealed.newProp = "test"; // ❌ Silently fails
```

### 🏦 Banking Example: Customer Account Management System

```javascript
// Complete banking system using objects and arrays

class BankingSystem {
  constructor() {
    // Using Map for O(1) lookups
    this.accounts = new Map();
    this.transactions = [];
    this.customers = new Map();
  }
  
  // Create customer with nested objects
  createCustomer(customerData) {
    const customer = {
      id: `CUST-${Date.now()}`,
      personalInfo: {
        firstName: customerData.firstName,
        lastName: customerData.lastName,
        email: customerData.email,
        phone: customerData.phone,
        dateOfBirth: new Date(customerData.dateOfBirth)
      },
      address: {
        street: customerData.street,
        city: customerData.city,
        state: customerData.state,
        zipCode: customerData.zipCode,
        country: customerData.country || 'USA'
      },
      accounts: [], // Array of account IDs
      kyc: {
        verified: false,
        documents: [],
        verificationDate: null
      },
      creditScore: null,
      createdAt: new Date(),
      lastLogin: null
    };
    
    this.customers.set(customer.id, customer);
    return customer;
  }
  
  // Create account using object destructuring
  createAccount({ customerId, type = 'checking', initialDeposit = 0 }) {
    // Validate customer exists
    const customer = this.customers.get(customerId);
    if (!customer) {
      throw new Error('Customer not found');
    }
    
    // Account object with computed properties
    const account = {
      accountNumber: `ACC-${Date.now()}`,
      customerId,
      type, // checking, savings, credit
      balance: initialDeposit,
      currency: 'USD',
      status: 'active',
      
      // Nested objects for limits and settings
      limits: {
        daily: type === 'checking' ? 5000 : 2000,
        monthly: type === 'checking' ? 50000 : 20000,
        perTransaction: 10000
      },
      
      settings: {
        overdraftProtection: type === 'checking',
        alerts: {
          lowBalance: true,
          largeTransactions: true,
          foreignTransactions: true
        },
        autoPayEnabled: false
      },
      
      // Interest rates (different for account types)
      interestRate: type === 'savings' ? 0.025 : 0.001,
      
      metadata: {
        createdAt: new Date(),
        lastActivity: new Date(),
        totalTransactions: 0,
        totalDeposits: initialDeposit,
        totalWithdrawals: 0
      }
    };
    
    // Add account to system
    this.accounts.set(account.accountNumber, account);
    
    // Link account to customer
    customer.accounts.push(account.accountNumber);
    
    return account;
  }
  
  // Object destructuring in parameters
  processTransaction({ from, to, amount, type, description = '' }) {
    const transaction = {
      id: `TXN-${Date.now()}`,
      type, // transfer, deposit, withdrawal
      from,
      to,
      amount,
      description,
      
      // Nested fee structure
      fees: {
        processing: this.calculateProcessingFee(amount),
        wire: type === 'wire' ? 25 : 0,
        international: to?.country !== 'USA' ? amount * 0.03 : 0
      },
      
      status: 'pending',
      timestamp: new Date(),
      metadata: {
        ip: '192.168.1.1',
        device: 'mobile',
        location: 'New York, NY'
      }
    };
    
    // Calculate total fees using Object.values
    transaction.totalFees = Object.values(transaction.fees).reduce((sum, fee) => sum + fee, 0);
    transaction.totalAmount = amount + transaction.totalFees;
    
    this.transactions.push(transaction);
    return transaction;
  }
  
  calculateProcessingFee(amount) {
    if (amount < 100) return 0;
    if (amount < 1000) return 1;
    if (amount < 10000) return amount * 0.001;
    return 10;
  }
  
  // Object.entries for iteration
  getAccountSummary(accountNumber) {
    const account = this.accounts.get(accountNumber);
    if (!account) return null;
    
    // Using Object.entries to transform nested objects
    const limitsArray = Object.entries(account.limits).map(([key, value]) => ({
      type: key,
      limit: value,
      used: 0 // Would calculate from transactions
    }));
    
    return {
      accountNumber: account.accountNumber,
      type: account.type,
      balance: account.balance,
      limits: limitsArray,
      settings: account.settings
    };
  }
  
  // Object spread and merging
  updateAccountSettings(accountNumber, newSettings) {
    const account = this.accounts.get(accountNumber);
    if (!account) throw new Error('Account not found');
    
    // Merge settings using spread operator (shallow merge)
    account.settings = {
      ...account.settings,
      ...newSettings,
      // Deep merge for nested objects
      alerts: {
        ...account.settings.alerts,
        ...(newSettings.alerts || {})
      }
    };
    
    return account;
  }
  
  // Object.freeze for immutable configuration
  getImmutableConfig() {
    return Object.freeze({
      maxDailyTransactions: 100,
      maxAccountsPerCustomer: 5,
      supportedCurrencies: Object.freeze(['USD', 'EUR', 'GBP']),
      feeStructure: Object.freeze({
        wire: 25,
        international: 0.03,
        atm: 2.5
      })
    });
  }
  
  // Object.keys, Object.values, Object.entries
  getStatistics() {
    const stats = {
      totalCustomers: this.customers.size,
      totalAccounts: this.accounts.size,
      totalTransactions: this.transactions.length,
      
      // Count accounts by type
      accountsByType: {},
      
      // Total balances
      totalDeposits: 0,
      
      // Average balance
      averageBalance: 0
    };
    
    // Iterate over accounts
    for (const account of this.accounts.values()) {
      // Count by type
      stats.accountsByType[account.type] = (stats.accountsByType[account.type] || 0) + 1;
      
      // Sum balances
      stats.totalDeposits += account.balance;
    }
    
    stats.averageBalance = stats.totalDeposits / stats.totalAccounts;
    
    return stats;
  }
  
  // Optional chaining for safe property access
  getCustomerEmail(customerId) {
    // Safe navigation through nested objects
    return this.customers.get(customerId)?.personalInfo?.email || 'N/A';
  }
  
  // Nullish coalescing for default values
  getAccountLimit(accountNumber, limitType) {
    const account = this.accounts.get(accountNumber);
    // Return limit or default to 1000
    return account?.limits?.[limitType] ?? 1000;
  }
}

// Usage Examples
console.log('=== Banking System with Objects ===\n');

const bank = new BankingSystem();

// 1. Create customer
const customer = bank.createCustomer({
  firstName: 'John',
  lastName: 'Doe',
  email: 'john.doe@example.com',
  phone: '555-0100',
  dateOfBirth: '1990-01-15',
  street: '123 Main St',
  city: 'New York',
  state: 'NY',
  zipCode: '10001'
});

console.log('Customer created:', customer.id);
console.log('Full name:', `${customer.personalInfo.firstName} ${customer.personalInfo.lastName}`);

// 2. Create accounts with object destructuring
const checkingAccount = bank.createAccount({
  customerId: customer.id,
  type: 'checking',
  initialDeposit: 5000
});

const savingsAccount = bank.createAccount({
  customerId: customer.id,
  type: 'savings',
  initialDeposit: 10000
});

console.log('\nAccounts created:');
console.log('- Checking:', checkingAccount.accountNumber, `($${checkingAccount.balance})`);
console.log('- Savings:', savingsAccount.accountNumber, `($${savingsAccount.balance})`);

// 3. Process transaction
const transaction = bank.processTransaction({
  from: checkingAccount.accountNumber,
  to: 'ACC-EXTERNAL',
  amount: 1000,
  type: 'wire',
  description: 'Payment to vendor'
});

console.log('\nTransaction processed:');
console.log('ID:', transaction.id);
console.log('Amount:', `$${transaction.amount}`);
console.log('Fees:', `$${transaction.totalFees}`);
console.log('Total:', `$${transaction.totalAmount}`);

// 4. Object destructuring from nested objects
const { personalInfo: { email, phone }, address: { city, state } } = customer;
console.log(`\nContact: ${email}, ${phone}`);
console.log(`Location: ${city}, ${state}`);

// 5. Update settings with spread operator
bank.updateAccountSettings(checkingAccount.accountNumber, {
  overdraftProtection: false,
  alerts: {
    lowBalance: false
  }
});

console.log('\nUpdated settings:', checkingAccount.settings);

// 6. Object.keys, Object.values, Object.entries
const stats = bank.getStatistics();
console.log('\nBank Statistics:');
Object.entries(stats).forEach(([key, value]) => {
  if (typeof value === 'object') {
    console.log(`${key}:`, JSON.stringify(value));
  } else {
    console.log(`${key}: ${value}`);
  }
});

// 7. Optional chaining
console.log('\nSafe property access:');
console.log('Customer email:', bank.getCustomerEmail(customer.id));
console.log('Non-existent customer:', bank.getCustomerEmail('INVALID'));

// 8. Immutable configuration
const config = bank.getImmutableConfig();
console.log('\nImmutable config:', config);
// config.maxDailyTransactions = 200; // ❌ Silently fails (frozen)

// 9. Object comparison (shallow vs deep)
const account1 = { accountNumber: 'ACC-1', balance: 1000 };
const account2 = { accountNumber: 'ACC-1', balance: 1000 };
const account3 = account1;

console.log('\nObject comparison:');
console.log('account1 === account2:', account1 === account2); // false (different references)
console.log('account1 === account3:', account1 === account3); // true (same reference)

// Deep equality check
const deepEqual = JSON.stringify(account1) === JSON.stringify(account2);
console.log('Deep equal:', deepEqual); // true

// 10. Cloning objects
const shallowClone = { ...checkingAccount };
const deepClone = JSON.parse(JSON.stringify(checkingAccount));

shallowClone.balance = 9999;
console.log('\nOriginal balance:', checkingAccount.balance); // Unchanged
console.log('Shallow clone balance:', shallowClone.balance); // 9999



let employee = {
    eid: "E102",
    ename: "Jack",
    eaddress: "New York",
    salary: 50000
};

console.log("Employee=> ", employee);

// Shallow copy
let newEmployee = { ...employee };    
console.log("New Employee=> ", newEmployee);

console.log("---------After modification----------");
newEmployee.ename = "Beck";

console.log("Employee=> ", employee);        
console.log("New Employee=> ", newEmployee);


let employee = {
    eid: "E102",
    ename: "Jack",
    eaddress: "New York",
    salary: 50000
}
console.log("=========Deep Copy========");
let newEmployee = JSON.parse(JSON.stringify(employee));
console.log("Employee=> ", employee);
console.log("New Employee=> ", newEmployee);
console.log("---------After modification---------");
newEmployee.ename = "Beck";
newEmployee.salary = 70000;
console.log("Employee=> ", employee);
console.log("New Employee=> ", newEmployee);



const lodash = require('lodash');
let employee = {
    eid: "E102",
    ename: "Jack",
    eaddress: "New York",
    salary: 50000,
    details: function () {
        return "Employee Name: " 
            + this.ename + "-->Salary: " 
            + this.salary;
    }
}

let deepCopy = lodash.cloneDeep(employee);
console.log("Original Employee Object");
console.log(employee);
console.log("Deep Copied Employee Object");
console.log(deepCopy);
deepCopy.eid = "E103";
deepCopy.ename = "Beck";
deepCopy.details = function () {
    return "Employee ID: " + this.eid 
        + "-->Salary: " + this.salary;
}
console.log("----------After Modification----------");
console.log("Original Employee Object");
console.log(employee);
console.log("Deep Copied Employee Object");
console.log(deepCopy);
console.log(employee.details());
console.log(deepCopy.details());
```

### Arrays

```javascript
// Array creation
const numbers = [1, 2, 3, 4, 5];
const mixed = [1, "hello", true, null, { key: "value" }];
const arrayFromConstructor = new Array(5); // Creates array with 5 empty slots

// Accessing elements
console.log(numbers[0]);        // 1 (first element)
console.log(numbers[numbers.length - 1]); // 5 (last element)

// Modifying arrays
numbers.push(6);                // Add to end
numbers.pop();                  // Remove from end
numbers.unshift(0);             // Add to beginning
numbers.shift();                // Remove from beginning

// Array methods
const fruits = ["apple", "banana", "orange"];

// map - Transform each element
const uppercased = fruits.map(fruit => fruit.toUpperCase());

// filter - Keep elements that pass test
const longFruits = fruits.filter(fruit => fruit.length > 5);

// reduce - Reduce to single value
const nums = [1, 2, 3, 4, 5];
const sum = nums.reduce((acc, num) => acc + num, 0);

// find - Find first element that matches
const found = fruits.find(fruit => fruit.startsWith("b")); // "banana"

// findIndex - Find index of first match
const index = fruits.findIndex(fruit => fruit === "orange"); // 2

// some - Check if at least one element passes test
const hasLongFruit = fruits.some(fruit => fruit.length > 6); // false

// every - Check if all elements pass test
const allStrings = fruits.every(fruit => typeof fruit === "string"); // true

// includes - Check if array contains value
console.log(fruits.includes("apple")); // true

// indexOf / lastIndexOf
console.log(fruits.indexOf("banana")); // 1

// slice - Create shallow copy of portion
const sliced = fruits.slice(1, 3); // ["banana", "orange"]

// splice - Modify array (add/remove elements)
const removed = fruits.splice(1, 1, "grape"); // Removes "banana", adds "grape"

// concat - Merge arrays
const moreFruits = fruits.concat(["mango", "kiwi"]);

// join - Convert to string
console.log(fruits.join(", ")); // "apple, grape, orange"

// reverse - Reverse array in place
fruits.reverse();

// sort - Sort array in place
numbers.sort((a, b) => a - b); // Ascending
numbers.sort((a, b) => b - a); // Descending

// flat - Flatten nested arrays
const nested = [1, [2, [3, [4]]]];
console.log(nested.flat());      // [1, 2, [3, [4]]]
console.log(nested.flat(2));     // [1, 2, 3, [4]]
console.log(nested.flat(Infinity)); // [1, 2, 3, 4]

// flatMap - Map then flatten
const arr = [1, 2, 3];
const doubled = arr.flatMap(x => [x, x * 2]); // [1, 2, 2, 4, 3, 6]

// Array destructuring
const [first, second, ...rest] = numbers;

// Array spread
const combined = [...fruits, ...moreFruits];
```

### 🏦 Banking Example: Transaction Processing with Arrays

```javascript
// Banking transaction processing using array methods

class TransactionAnalyzer {
  constructor() {
    // Sample transaction data
    this.transactions = [
      { id: 'T1', type: 'deposit', amount: 1000, category: 'salary', date: new Date('2024-01-01'), status: 'completed' },
      { id: 'T2', type: 'withdrawal', amount: 50, category: 'groceries', date: new Date('2024-01-02'), status: 'completed' },
      { id: 'T3', type: 'withdrawal', amount: 200, category: 'entertainment', date: new Date('2024-01-03'), status: 'completed' },
      { id: 'T4', type: 'deposit', amount: 500, category: 'freelance', date: new Date('2024-01-04'), status: 'completed' },
      { id: 'T5', type: 'withdrawal', amount: 1500, category: 'rent', date: new Date('2024-01-05'), status: 'pending' },
      { id: 'T6', type: 'withdrawal', amount: 75, category: 'utilities', date: new Date('2024-01-06'), status: 'completed' },
      { id: 'T7', type: 'deposit', amount: 2000, category: 'bonus', date: new Date('2024-01-07'), status: 'completed' },
      { id: 'T8', type: 'withdrawal', amount: 30, category: 'groceries', date: new Date('2024-01-08'), status: 'failed' }
    ];
  }
  
  // filter() - Find completed deposits
  getCompletedDeposits() {
    return this.transactions.filter(txn => 
      txn.type === 'deposit' && txn.status === 'completed'
    );
  }
  
  // map() - Extract just amounts and format currency
  getFormattedAmounts() {
    return this.transactions.map(txn => ({
      id: txn.id,
      amount: `$${txn.amount.toFixed(2)}`,
      type: txn.type
    }));
  }
  
  // reduce() - Calculate total balance
  calculateBalance() {
    return this.transactions.reduce((balance, txn) => {
      if (txn.status !== 'completed') return balance;
      
      return txn.type === 'deposit' 
        ? balance + txn.amount 
        : balance - txn.amount;
    }, 0);
  }
  
  // find() - Find specific transaction
  findTransaction(id) {
    return this.transactions.find(txn => txn.id === id);
  }
  
  // findIndex() - Find position of transaction
  findTransactionIndex(id) {
    return this.transactions.findIndex(txn => txn.id === id);
  }
  
  // some() - Check if any high-value transactions exist
  hasLargeTransactions(threshold = 1000) {
    return this.transactions.some(txn => txn.amount > threshold);
  }
  
  // every() - Check if all transactions are completed
  allCompleted() {
    return this.transactions.every(txn => txn.status === 'completed');
  }
  
  // sort() - Sort by amount (descending)
  getSortedByAmount() {
    return [...this.transactions].sort((a, b) => b.amount - a.amount);
  }
  
  // sort() - Sort by date (most recent first)
  getSortedByDate() {
    return [...this.transactions].sort((a, b) => b.date - a.date);
  }
  
  // reduce() - Group transactions by category
  groupByCategory() {
    return this.transactions.reduce((groups, txn) => {
      const category = txn.category;
      if (!groups[category]) {
        groups[category] = [];
      }
      groups[category].push(txn);
      return groups;
    }, {});
  }
  
  // reduce() - Sum by category
  getTotalsByCategory() {
    return this.transactions
      .filter(txn => txn.status === 'completed')
      .reduce((totals, txn) => {
        const category = txn.category;
        totals[category] = (totals[category] || 0) + 
          (txn.type === 'withdrawal' ? txn.amount : 0);
        return totals;
      }, {});
  }
  
  // flatMap() - Get all categories (with duplicates flattened)
  getAllCategories() {
    return this.transactions.flatMap(txn => [txn.category]);
  }
  
  // Set for unique categories
  getUniqueCategories() {
    return [...new Set(this.transactions.map(txn => txn.category))];
  }
  
  // slice() - Get recent transactions
  getRecentTransactions(count = 5) {
    return this.transactions
      .sort((a, b) => b.date - a.date)
      .slice(0, count);
  }
  
  // forEach() - Apply pending status updates
  updatePendingTransactions() {
    this.transactions.forEach(txn => {
      if (txn.status === 'pending') {
        txn.status = 'completed';
        txn.completedAt = new Date();
      }
    });
  }
  
  // includes() - Check if transaction exists
  hasTransaction(id) {
    return this.transactions.map(t => t.id).includes(id);
  }
  
  // Chaining multiple methods
  getMonthlySummary() {
    return this.transactions
      .filter(txn => txn.status === 'completed') // Only completed
      .filter(txn => txn.type === 'withdrawal')  // Only withdrawals
      .map(txn => txn.amount)                    // Get amounts
      .reduce((sum, amount) => sum + amount, 0); // Sum
  }
  
  // Complex analysis with multiple array methods
  getSpendingInsights() {
    const withdrawals = this.transactions
      .filter(txn => txn.type === 'withdrawal' && txn.status === 'completed');
    
    const totalSpending = withdrawals
      .reduce((sum, txn) => sum + txn.amount, 0);
    
    const averageTransaction = totalSpending / withdrawals.length;
    
    const categoryTotals = withdrawals
      .reduce((acc, txn) => {
        acc[txn.category] = (acc[txn.category] || 0) + txn.amount;
        return acc;
      }, {});
    
    const topCategory = Object.entries(categoryTotals)
      .sort((a, b) => b[1] - a[1])[0];
    
    return {
      totalSpending,
      averageTransaction,
      transactionCount: withdrawals.length,
      topCategory: topCategory ? {
        name: topCategory[0],
        amount: topCategory[1]
      } : null
    };
  }
}

// Usage Examples
console.log('=== Transaction Analysis with Arrays ===\n');

const analyzer = new TransactionAnalyzer();

// 1. Filter completed deposits
const deposits = analyzer.getCompletedDeposits();
console.log('Completed deposits:', deposits.length);
deposits.forEach(d => console.log(`  - ${d.id}: $${d.amount} (${d.category})`));

// 2. Map to formatted amounts
console.log('\nFormatted transactions:');
const formatted = analyzer.getFormattedAmounts().slice(0, 3);
formatted.forEach(f => console.log(`  ${f.id}: ${f.amount} (${f.type})`));

// 3. Reduce to calculate balance
const balance = analyzer.calculateBalance();
console.log('\nCurrent balance:', `$${balance.toFixed(2)}`);

// 4. Find specific transaction
const txn = analyzer.findTransaction('T3');
console.log('\nFound transaction T3:', txn);

// 5. Some and every
console.log('\nHas large transactions (>$1000)?', analyzer.hasLargeTransactions());
console.log('All transactions completed?', analyzer.allCompleted());

// 6. Sort by amount
console.log('\nTop 3 transactions by amount:');
const topTransactions = analyzer.getSortedByAmount().slice(0, 3);
topTransactions.forEach(t => 
  console.log(`  ${t.id}: $${t.amount} (${t.category})`)
);

// 7. Group by category
const grouped = analyzer.groupByCategory();
console.log('\nTransactions by category:');
Object.entries(grouped).forEach(([category, txns]) => {
  console.log(`  ${category}: ${txns.length} transactions`);
});

// 8. Category spending totals
const categoryTotals = analyzer.getTotalsByCategory();
console.log('\nSpending by category:');
Object.entries(categoryTotals)
  .sort((a, b) => b[1] - a[1])
  .forEach(([category, total]) => {
    console.log(`  ${category}: $${total.toFixed(2)}`);
  });

// 9. Unique categories
const categories = analyzer.getUniqueCategories();
console.log('\nUnique categories:', categories.join(', '));

// 10. Recent transactions
console.log('\nRecent transactions:');
const recent = analyzer.getRecentTransactions(3);
recent.forEach(t => 
  console.log(`  ${t.date.toLocaleDateString()}: ${t.type} $${t.amount}`)
);

// 11. Complex insights
const insights = analyzer.getSpendingInsights();
console.log('\nSpending Insights:');
console.log('  Total spending:', `$${insights.totalSpending.toFixed(2)}`);
console.log('  Average transaction:', `$${insights.averageTransaction.toFixed(2)}`);
console.log('  Transaction count:', insights.transactionCount);
if (insights.topCategory) {
  console.log('  Top category:', `${insights.topCategory.name} ($${insights.topCategory.amount})`);
}

// 12. Array destructuring in banking
const [firstTxn, secondTxn, ...restTxns] = analyzer.transactions;
console.log('\nArray destructuring:');
console.log('First transaction:', firstTxn.id);
console.log('Remaining transactions:', restTxns.length);

// 13. Spread operator for copying
const transactionsCopy = [...analyzer.transactions];
console.log('\nCopied transactions:', transactionsCopy.length);

// 14. Array.from for creating ranges
const monthlyTargets = Array.from({ length: 12 }, (_, i) => ({
  month: i + 1,
  savingsTarget: 1000 + (i * 50)
}));
console.log('\nMonthly savings targets:', monthlyTargets.slice(0, 3));
```

### Key Banking Array Takeaways

1. **filter()** - Find specific transactions (completed, by type, by amount)
2. **map()** - Transform data (format currency, extract fields)
3. **reduce()** - Aggregate data (calculate balance, sum by category)
4. **sort()** - Order transactions (by date, amount, category)
5. **find()/findIndex()** - Locate specific records
6. **some()/every()** - Validation checks
7. **slice()** - Pagination (get recent N transactions)
8. **flatMap()** - Flatten nested arrays
9. **Chaining** - Combine operations for complex queries
10. **Spread/Destructuring** - Copy arrays, extract values safely

---

## Event Emitter Pattern

The Event Emitter pattern is fundamental in Node.js and useful for creating event-driven architectures.

### Basic Event Emitter Implementation

```javascript
// Simple EventEmitter class
class EventEmitter {
  constructor() {
    this.events = {};
  }
  
  // Subscribe to an event
  on(eventName, callback) {
    if (!this.events[eventName]) {
      this.events[eventName] = [];
    }
    this.events[eventName].push(callback);
    return this;
  }
  
  // Subscribe to event (fires only once)
  once(eventName, callback) {
    const wrapper = (...args) => {
      callback(...args);
      this.off(eventName, wrapper);
    };
    this.on(eventName, wrapper);
    return this;
  }
  
  // Emit an event
  emit(eventName, ...args) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      callbacks.forEach(callback => callback(...args));
    }
    return this;
  }
  
  // Unsubscribe from event
  off(eventName, callback) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      this.events[eventName] = callbacks.filter(cb => cb !== callback);
    }
    return this;
  }
  
  // Remove all listeners for event
  removeAllListeners(eventName) {
    if (eventName) {
      delete this.events[eventName];
    } else {
      this.events = {};
    }
    return this;
  }
  
  // Get listener count
  listenerCount(eventName) {
    const callbacks = this.events[eventName];
    return callbacks ? callbacks.length : 0;
  }
}

// Usage Example
const emitter = new EventEmitter();

// Subscribe to events
emitter.on('data', (data) => {
  console.log('Received data:', data);
});

emitter.on('data', (data) => {
  console.log('Another listener:', data);
});

// Emit event
emitter.emit('data', { value: 42 });
// Output:
// Received data: { value: 42 }
// Another listener: { value: 42 }

// Once listener
emitter.once('connect', () => {
  console.log('Connected! (fires only once)');
});

emitter.emit('connect'); // "Connected! (fires only once)"
emitter.emit('connect'); // Nothing happens

// Unsubscribe
const errorHandler = (error) => {
  console.error('Error:', error);
};

emitter.on('error', errorHandler);
emitter.off('error', errorHandler);
```

### Real-World Event Emitter Example

```javascript
// User Authentication System
class AuthSystem extends EventEmitter {
  constructor() {
    super();
    this.users = new Map();
  }
  
  register(username, password) {
    if (this.users.has(username)) {
      this.emit('register:error', { message: 'User already exists' });
      return false;
    }
    
    this.users.set(username, { password, loginAttempts: 0 });
    this.emit('register:success', { username });
    return true;
  }
  
  login(username, password) {
    const user = this.users.get(username);
    
    if (!user) {
      this.emit('login:error', { message: 'User not found' });
      return false;
    }
    
    if (user.loginAttempts >= 3) {
      this.emit('login:locked', { username });
      return false;
    }
    
    if (user.password !== password) {
      user.loginAttempts++;
      this.emit('login:failed', { username, attempts: user.loginAttempts });
      return false;
    }
    
    user.loginAttempts = 0;
    this.emit('login:success', { username });
    return true;
  }
}

// Usage
const auth = new AuthSystem();

// Listen to events
auth.on('register:success', ({ username }) => {
  console.log(`✅ User ${username} registered successfully`);
});

auth.on('login:success', ({ username }) => {
  console.log(`✅ User ${username} logged in`);
});

auth.on('login:failed', ({ username, attempts }) => {
  console.log(`❌ Login failed for ${username}. Attempts: ${attempts}`);
});

auth.on('login:locked', ({ username }) => {
  console.log(`🔒 Account locked for ${username}`);
});

// Test
auth.register('john', 'password123');
auth.login('john', 'wrongpassword');
auth.login('john', 'wrongpassword');
auth.login('john', 'wrongpassword');
auth.login('john', 'password123'); // Locked!
```

### 🏦 Banking Example: Event-Driven Banking System

```javascript
// Comprehensive banking system using event emitters

// Extend the basic EventEmitter
class BankEventEmitter extends EventEmitter {
  constructor() {
    super();
    this.eventLog = [];
  }
  
  // Override emit to log all events
  emit(eventName, ...args) {
    this.eventLog.push({
      event: eventName,
      data: args,
      timestamp: new Date()
    });
    return super.emit(eventName, ...args);
  }
  
  getEventLog() {
    return [...this.eventLog];
  }
}

// Banking Account with Events
class BankingAccount extends BankEventEmitter {
  constructor(accountNumber, customerName, initialBalance) {
    super();
    this.accountNumber = accountNumber;
    this.customerName = customerName;
    this.balance = initialBalance;
    this.dailyLimit = 5000;
    this.dailyUsed = 0;
    this.transactions = [];
    
    // Emit account creation event
    this.emit('account:created', {
      accountNumber,
      customerName,
      initialBalance
    });
  }
  
  deposit(amount) {
    this.emit('transaction:initiated', {
      type: 'deposit',
      amount,
      accountNumber: this.accountNumber
    });
    
    if (amount <= 0) {
      this.emit('transaction:failed', {
        type: 'deposit',
        amount,
        reason: 'Invalid amount'
      });
      throw new Error('Amount must be positive');
    }
    
    // Check for large deposits (anti-money laundering)
    if (amount > 10000) {
      this.emit('compliance:large-deposit', {
        accountNumber: this.accountNumber,
        amount,
        customer: this.customerName
      });
    }
    
    this.balance += amount;
    
    this.transactions.push({
      type: 'deposit',
      amount,
      balance: this.balance,
      timestamp: new Date()
    });
    
    this.emit('transaction:completed', {
      type: 'deposit',
      amount,
      newBalance: this.balance
    });
    
    this.emit('balance:updated', {
      accountNumber: this.accountNumber,
      oldBalance: this.balance - amount,
      newBalance: this.balance,
      change: amount
    });
    
    return this.balance;
  }
  
  withdraw(amount) {
    this.emit('transaction:initiated', {
      type: 'withdrawal',
      amount,
      accountNumber: this.accountNumber
    });
    
    if (amount <= 0) {
      this.emit('transaction:failed', {
        type: 'withdrawal',
        amount,
        reason: 'Invalid amount'
      });
      throw new Error('Amount must be positive');
    }
    
    // Check daily limit
    if (this.dailyUsed + amount > this.dailyLimit) {
      this.emit('limit:exceeded', {
        accountNumber: this.accountNumber,
        attempted: amount,
        limit: this.dailyLimit,
        used: this.dailyUsed
      });
      throw new Error('Daily limit exceeded');
    }
    
    // Check sufficient funds
    if (amount > this.balance) {
      this.emit('transaction:failed', {
        type: 'withdrawal',
        amount,
        reason: 'Insufficient funds',
        available: this.balance
      });
      throw new Error('Insufficient funds');
    }
    
    const oldBalance = this.balance;
    this.balance -= amount;
    this.dailyUsed += amount;
    
    this.transactions.push({
      type: 'withdrawal',
      amount,
      balance: this.balance,
      timestamp: new Date()
    });
    
    this.emit('transaction:completed', {
      type: 'withdrawal',
      amount,
      newBalance: this.balance
    });
    
    this.emit('balance:updated', {
      accountNumber: this.accountNumber,
      oldBalance,
      newBalance: this.balance,
      change: -amount
    });
    
    // Check if balance is getting low
    if (this.balance < 100) {
      this.emit('balance:low', {
        accountNumber: this.accountNumber,
        currentBalance: this.balance,
        customer: this.customerName
      });
    }
    
    return this.balance;
  }
  
  transfer(toAccount, amount) {
    this.emit('transfer:initiated', {
      from: this.accountNumber,
      to: toAccount.accountNumber,
      amount
    });
    
    try {
      // Withdraw from this account
      this.withdraw(amount);
      
      // Deposit to recipient account
      toAccount.deposit(amount);
      
      this.emit('transfer:completed', {
        from: this.accountNumber,
        to: toAccount.accountNumber,
        amount
      });
      
      return true;
    } catch (error) {
      this.emit('transfer:failed', {
        from: this.accountNumber,
        to: toAccount.accountNumber,
        amount,
        reason: error.message
      });
      throw error;
    }
  }
}

// Fraud Detection System (listener)
class FraudDetectionSystem {
  constructor() {
    this.suspiciousActivities = [];
    this.blockedAccounts = new Set();
  }
  
  monitorAccount(account) {
    // Monitor large withdrawals
    account.on('transaction:initiated', (data) => {
      if (data.type === 'withdrawal' && data.amount > 5000) {
        console.log('🔍 Fraud Detection: Monitoring large withdrawal:', data.amount);
      }
    });
    
    // Detect rapid transactions
    let transactionCount = 0;
    let resetTimer;
    
    account.on('transaction:completed', (data) => {
      transactionCount++;
      
      if (transactionCount > 10) {
        this.suspiciousActivities.push({
          type: 'rapid_transactions',
          accountNumber: data.accountNumber || account.accountNumber,
          count: transactionCount,
          timestamp: new Date()
        });
        
        account.emit('fraud:suspected', {
          accountNumber: account.accountNumber,
          reason: 'Too many rapid transactions',
          count: transactionCount
        });
      }
      
      // Reset counter after 1 minute
      clearTimeout(resetTimer);
      resetTimer = setTimeout(() => {
        transactionCount = 0;
      }, 60000);
    });
    
    // Monitor suspicious patterns
    account.on('transaction:failed', (data) => {
      if (data.reason === 'Insufficient funds') {
        console.log('⚠️ Fraud Detection: Multiple failed attempts detected');
      }
    });
  }
  
  getSuspiciousActivities() {
    return [...this.suspiciousActivities];
  }
}

// Notification System (listener)
class NotificationSystem {
  constructor() {
    this.notifications = [];
  }
  
  subscribeToAccount(account, email, phone) {
    // Email notifications for deposits
    account.on('transaction:completed', (data) => {
      if (data.type === 'deposit') {
        const notification = {
          to: email,
          subject: 'Deposit Confirmed',
          message: `$${data.amount} has been deposited. New balance: $${data.newBalance}`,
          timestamp: new Date()
        };
        this.notifications.push(notification);
        console.log(`📧 Email sent to ${email}: ${notification.message}`);
      }
    });
    
    // SMS for withdrawals
    account.on('transaction:completed', (data) => {
      if (data.type === 'withdrawal') {
        const notification = {
          to: phone,
          message: `Withdrawal: $${data.amount}. Balance: $${data.newBalance}`,
          timestamp: new Date()
        };
        this.notifications.push(notification);
        console.log(`📱 SMS sent to ${phone}: ${notification.message}`);
      }
    });
    
    // Alert for low balance
    account.on('balance:low', (data) => {
      const notification = {
        to: email,
        priority: 'HIGH',
        subject: 'Low Balance Alert',
        message: `Your balance is low: $${data.currentBalance}`,
        timestamp: new Date()
      };
      this.notifications.push(notification);
      console.log(`⚠️ ALERT sent to ${email}: Low balance!`);
    });
    
    // Alert for fraud
    account.on('fraud:suspected', (data) => {
      const notification = {
        to: phone,
        priority: 'URGENT',
        message: `FRAUD ALERT: ${data.reason}. Contact bank immediately.`,
        timestamp: new Date()
      };
      this.notifications.push(notification);
      console.log(`🚨 URGENT SMS to ${phone}: Fraud suspected!`);
    });
  }
  
  getNotifications() {
    return [...this.notifications];
  }
}

// Compliance System (listener)
class ComplianceSystem {
  constructor() {
    this.reports = [];
  }
  
  monitor(account) {
    // Monitor large deposits
    account.on('compliance:large-deposit', (data) => {
      const report = {
        type: 'LARGE_DEPOSIT',
        accountNumber: data.accountNumber,
        customer: data.customer,
        amount: data.amount,
        timestamp: new Date(),
        status: 'PENDING_REVIEW'
      };
      
      this.reports.push(report);
      console.log(`📋 Compliance Report: Large deposit of $${data.amount} by ${data.customer}`);
      
      // Automatically flag if very large
      if (data.amount > 50000) {
        account.emit('compliance:flagged', {
          reportId: this.reports.length - 1,
          reason: 'Deposit exceeds $50,000 threshold'
        });
      }
    });
    
    // Monitor transfers
    account.on('transfer:completed', (data) => {
      if (data.amount > 10000) {
        const report = {
          type: 'LARGE_TRANSFER',
          from: data.from,
          to: data.to,
          amount: data.amount,
          timestamp: new Date(),
          status: 'LOGGED'
        };
        this.reports.push(report);
        console.log(`📋 Compliance: Large transfer logged: $${data.amount}`);
      }
    });
  }
  
  getReports() {
    return [...this.reports];
  }
}

// Analytics System (listener)
class AnalyticsSystem {
  constructor() {
    this.stats = {
      totalDeposits: 0,
      totalWithdrawals: 0,
      transactionCount: 0,
      averageTransaction: 0
    };
  }
  
  track(account) {
    account.on('transaction:completed', (data) => {
      this.stats.transactionCount++;
      
      if (data.type === 'deposit') {
        this.stats.totalDeposits += data.amount;
      } else if (data.type === 'withdrawal') {
        this.stats.totalWithdrawals += data.amount;
      }
      
      this.stats.averageTransaction = 
        (this.stats.totalDeposits + this.stats.totalWithdrawals) / 
        this.stats.transactionCount;
    });
  }
  
  getStats() {
    return { ...this.stats };
  }
}

// Usage Examples
console.log('=== Event-Driven Banking System ===\n');

// Create accounts
const account1 = new BankingAccount('ACC-001', 'Alice Johnson', 1000);
const account2 = new BankingAccount('ACC-002', 'Bob Wilson', 2000);

// Create systems
const fraudDetection = new FraudDetectionSystem();
const notifications = new NotificationSystem();
const compliance = new ComplianceSystem();
const analytics = new AnalyticsSystem();

// Setup monitoring
console.log('1. Setting up monitoring systems...\n');
fraudDetection.monitorAccount(account1);
notifications.subscribeToAccount(account1, 'alice@example.com', '555-0100');
compliance.monitor(account1);
analytics.track(account1);
analytics.track(account2);

// 2. Normal deposit
console.log('2. Normal Deposit:');
account1.deposit(500);

// 3. Large deposit (triggers compliance)
console.log('\n3. Large Deposit (Compliance Alert):');
account1.deposit(15000);

// 4. Withdrawal with notification
console.log('\n4. Withdrawal:');
account1.withdraw(200);

// 5. Multiple withdrawals to test fraud detection
console.log('\n5. Multiple Rapid Transactions (Fraud Detection):');
for (let i = 0; i < 12; i++) {
  try {
    account1.withdraw(10);
  } catch (error) {
    console.log(`   Transaction ${i + 1} failed:`, error.message);
  }
}

// 6. Low balance alert
console.log('\n6. Withdraw Until Low Balance:');
try {
  account1.withdraw(15500); // Will leave small balance
} catch (error) {
  console.log('   Error:', error.message);
}

// 7. Transfer between accounts
console.log('\n7. Transfer:');
account2.deposit(5000); // Give account2 more funds
account2.transfer(account1, 1000);

// 8. View analytics
console.log('\n8. Analytics Summary:');
const stats = analytics.getStats();
console.log('   Total deposits:', `$${stats.totalDeposits.toFixed(2)}`);
console.log('   Total withdrawals:', `$${stats.totalWithdrawals.toFixed(2)}`);
console.log('   Transaction count:', stats.transactionCount);
console.log('   Average transaction:', `$${stats.averageTransaction.toFixed(2)}`);

// 9. View fraud detection reports
console.log('\n9. Fraud Detection:');
const suspicious = fraudDetection.getSuspiciousActivities();
console.log(`   Suspicious activities detected: ${suspicious.length}`);
suspicious.forEach(activity => {
  console.log(`   - ${activity.type}: ${activity.reason} (${activity.count} transactions)`);
});

// 10. View compliance reports
console.log('\n10. Compliance Reports:');
const reports = compliance.getReports();
console.log(`   Total reports: ${reports.length}`);
reports.forEach((report, idx) => {
  console.log(`   ${idx + 1}. ${report.type}: $${report.amount} - ${report.status}`);
});

// 11. View notification history
console.log('\n11. Notifications Sent:');
const notifs = notifications.getNotifications();
console.log(`   Total notifications: ${notifs.length}`);
notifs.slice(0, 5).forEach(n => {
  console.log(`   - ${n.priority || 'NORMAL'}: ${n.subject || n.message}`);
});

// 12. Event log
console.log('\n12. Event Log (Last 5 events):');
const eventLog = account1.getEventLog();
console.log(`   Total events: ${eventLog.length}`);
eventLog.slice(-5).forEach(log => {
  console.log(`   - ${log.event} at ${log.timestamp.toLocaleTimeString()}`);
});

// 13. Once event (fires only once)
console.log('\n13. One-Time Event Listener:');
account1.once('transaction:completed', (data) => {
  console.log('   🎉 Special one-time notification:', data.type);
});

account1.deposit(100); // Triggers once listener
account1.deposit(100); // Does not trigger (already fired)

// 14. Remove listener
console.log('\n14. Removing Listeners:');
const tempListener = (data) => {
  console.log('Temporary listener:', data);
};

account1.on('balance:updated', tempListener);
console.log('   Listener count before removal:', account1.listenerCount('balance:updated'));

account1.off('balance:updated', tempListener);
console.log('   Listener count after removal:', account1.listenerCount('balance:updated'));

// 15. Error event
console.log('\n15. Error Handling with Events:');
account1.on('error', (error) => {
  console.log('   ❌ Error event caught:', error.message);
});

try {
  account1.withdraw(999999);
} catch (error) {
  account1.emit('error', error);
}
```

### Key Event Emitter Takeaways

1. **Decoupling** - Separate concerns (account, notifications, fraud detection)
2. **Observer Pattern** - Multiple listeners for same event
3. **Event Naming** - Use namespaces (`transaction:completed`, `balance:low`)
4. **once()** - Listen to event only once
5. **Event Data** - Pass relevant context with each event
6. **Error Events** - Special handling for errors
7. **Logging** - Track all events for audit trail
8. **Real-time Monitoring** - Systems react immediately to events
9. **Scalability** - Easy to add new listeners without modifying core logic
10. **Async Coordination** - Events help coordinate async operations

---

## Prototypes & Inheritance

### Prototype Chain

```javascript
// Every object has a prototype
const obj = {};
console.log(obj.__proto__ === Object.prototype); // true

// Constructor function
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// Add method to prototype
Person.prototype.greet = function() {
  return `Hello, I'm ${this.name}`;
};

Person.prototype.birthday = function() {
  this.age++;
};

// Create instances
const john = new Person("John", 30);
const jane = new Person("Jane", 25);

console.log(john.greet());    // "Hello, I'm John"
john.birthday();
console.log(john.age);        // 31

// Prototype chain
console.log(john.__proto__ === Person.prototype);           // true
console.log(Person.prototype.__proto__ === Object.prototype); // true
console.log(Object.prototype.__proto__);                    // null
```

### Prototypal Inheritance

```javascript
// Parent constructor
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  return `${this.name} makes a sound`;
};

// Child constructor
function Dog(name, breed) {
  Animal.call(this, name); // Call parent constructor
  this.breed = breed;
}

// Set up inheritance
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

// Add dog-specific method
Dog.prototype.bark = function() {
  return `${this.name} barks!`;
};

// Override parent method
Dog.prototype.speak = function() {
  return `${this.name} barks loudly!`;
};

const dog = new Dog("Buddy", "Golden Retriever");
console.log(dog.speak());  // "Buddy barks loudly!"
console.log(dog.bark());   // "Buddy barks!"
```

### ES6 Classes (Syntactic Sugar)

```javascript
// ES6 class syntax
class Animal {
  constructor(name) {
    this.name = name;
  }
  
  speak() {
    return `${this.name} makes a sound`;
  }
  
  static info() {
    return "Animals are living creatures";
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // Call parent constructor
    this.breed = breed;
  }
  
  speak() {
    return `${this.name} barks!`;
  }
  
  fetch() {
    return `${this.name} fetches the ball`;
  }
}

const dog = new Dog("Max", "Labrador");
console.log(dog.speak());     // "Max barks!"
console.log(dog.fetch());     // "Max fetches the ball"
console.log(Animal.info());   // "Animals are living creatures"
```

### 🏦 Banking Example: Account Inheritance Hierarchy

```javascript
// Complete banking account hierarchy using prototypes and ES6 classes

// Base Account class
class BankAccount {
  // Static properties
  static accountCount = 0;
  static minBalance = 0;
  
  constructor(accountNumber, customerName, initialBalance) {
    this.accountNumber = accountNumber;
    this.customerName = customerName;
    this.balance = initialBalance;
    this.transactions = [];
    this.createdAt = new Date();
    this.status = 'active';
    
    BankAccount.accountCount++;
  }
  
  // Common methods for all accounts
  deposit(amount) {
    if (amount <= 0) {
      throw new Error('Deposit amount must be positive');
    }
    
    this.balance += amount;
    this._recordTransaction('deposit', amount);
    return this.balance;
  }
  
  withdraw(amount) {
    if (amount <= 0) {
      throw new Error('Withdrawal amount must be positive');
    }
    
    if (amount > this.balance) {
      throw new Error('Insufficient funds');
    }
    
    this.balance -= amount;
    this._recordTransaction('withdrawal', amount);
    return this.balance;
  }
  
  getBalance() {
    return this.balance;
  }
  
  getTransactionHistory() {
    return [...this.transactions]; // Return copy
  }
  
  // Protected method (convention: _ prefix)
  _recordTransaction(type, amount) {
    this.transactions.push({
      type,
      amount,
      balance: this.balance,
      timestamp: new Date()
    });
  }
  
  // Method to be overridden
  calculateInterest() {
    return 0; // Base account has no interest
  }
  
  // Method to be overridden
  getAccountType() {
    return 'Basic Account';
  }
  
  // Static method
  static getTotalAccounts() {
    return BankAccount.accountCount;
  }
  
  // Display account info
  displayInfo() {
    return `${this.getAccountType()} #${this.accountNumber}
Customer: ${this.customerName}
Balance: $${this.balance.toFixed(2)}
Status: ${this.status}
Created: ${this.createdAt.toLocaleDateString()}`;
  }
}

// Checking Account extends BankAccount
class CheckingAccount extends BankAccount {
  static minBalance = 100; // Minimum balance required
  
  constructor(accountNumber, customerName, initialBalance, overdraftLimit = 500) {
    super(accountNumber, customerName, initialBalance);
    this.overdraftLimit = overdraftLimit;
    this.monthlyFee = 10;
    this.freeTransactions = 10;
    this.transactionCount = 0;
  }
  
  // Override withdraw to allow overdraft
  withdraw(amount) {
    if (amount <= 0) {
      throw new Error('Withdrawal amount must be positive');
    }
    
    const availableFunds = this.balance + this.overdraftLimit;
    
    if (amount > availableFunds) {
      throw new Error(`Insufficient funds. Available: $${availableFunds}`);
    }
    
    this.balance -= amount;
    this.transactionCount++;
    
    // Charge fee if over free transaction limit
    if (this.transactionCount > this.freeTransactions) {
      const fee = 1.5;
      this.balance -= fee;
      this._recordTransaction('fee', fee);
    }
    
    this._recordTransaction('withdrawal', amount);
    return this.balance;
  }
  
  // Checking accounts have minimal interest
  calculateInterest() {
    const rate = 0.001; // 0.1% annual
    return this.balance * rate / 12; // Monthly
  }
  
  // Override method
  getAccountType() {
    return 'Checking Account';
  }
  
  // Checking-specific method
  writeCheck(payee, amount) {
    console.log(`Writing check to ${payee} for $${amount}`);
    return this.withdraw(amount);
  }
  
  // Apply monthly fee
  applyMonthlyFee() {
    if (this.balance >= CheckingAccount.minBalance) {
      return; // Fee waived for maintaining minimum balance
    }
    
    this.balance -= this.monthlyFee;
    this._recordTransaction('monthly_fee', this.monthlyFee);
  }
  
  // Reset transaction count monthly
  resetTransactionCount() {
    this.transactionCount = 0;
  }
}

// Savings Account extends BankAccount
class SavingsAccount extends BankAccount {
  static minBalance = 500;
  static maxWithdrawals = 6; // Federal regulation
  
  constructor(accountNumber, customerName, initialBalance, interestRate = 0.025) {
    super(accountNumber, customerName, initialBalance);
    this.interestRate = interestRate; // 2.5% annual
    this.withdrawalsThisMonth = 0;
    this.lastInterestDate = new Date();
  }
  
  // Override withdraw with withdrawal limits
  withdraw(amount) {
    if (this.withdrawalsThisMonth >= SavingsAccount.maxWithdrawals) {
      throw new Error(`Maximum ${SavingsAccount.maxWithdrawals} withdrawals per month exceeded`);
    }
    
    if (amount <= 0) {
      throw new Error('Withdrawal amount must be positive');
    }
    
    if (this.balance - amount < SavingsAccount.minBalance) {
      throw new Error(`Cannot withdraw. Minimum balance of $${SavingsAccount.minBalance} required`);
    }
    
    this.balance -= amount;
    this.withdrawalsThisMonth++;
    this._recordTransaction('withdrawal', amount);
    return this.balance;
  }
  
  // Savings accounts have higher interest
  calculateInterest() {
    const monthlyRate = this.interestRate / 12;
    return this.balance * monthlyRate;
  }
  
  // Apply interest monthly
  applyInterest() {
    const interest = this.calculateInterest();
    this.balance += interest;
    this._recordTransaction('interest', interest);
    this.lastInterestDate = new Date();
    return interest;
  }
  
  // Override method
  getAccountType() {
    return 'Savings Account';
  }
  
  // Reset monthly withdrawal count
  resetWithdrawalCount() {
    this.withdrawalsThisMonth = 0;
  }
  
  // Savings-specific method
  getProjectedBalance(months) {
    let projectedBalance = this.balance;
    const monthlyRate = this.interestRate / 12;
    
    for (let i = 0; i < months; i++) {
      projectedBalance += projectedBalance * monthlyRate;
    }
    
    return projectedBalance;
  }
}

// Credit Account extends BankAccount
class CreditAccount extends BankAccount {
  constructor(accountNumber, customerName, creditLimit, apr = 0.18) {
    super(accountNumber, customerName, 0); // Credit starts at 0
    this.creditLimit = creditLimit;
    this.apr = apr; // Annual Percentage Rate (18%)
    this.availableCredit = creditLimit;
    this.minimumPaymentPercent = 0.02; // 2% of balance
    this.lastPaymentDate = null;
  }
  
  // For credit cards, deposit means making a payment
  deposit(amount) {
    if (amount <= 0) {
      throw new Error('Payment amount must be positive');
    }
    
    this.balance -= amount; // Paying down debt
    this.availableCredit = this.creditLimit - Math.abs(this.balance);
    this._recordTransaction('payment', amount);
    this.lastPaymentDate = new Date();
    return this.balance;
  }
  
  // For credit cards, withdraw means making a purchase
  withdraw(amount) {
    if (amount <= 0) {
      throw new Error('Purchase amount must be positive');
    }
    
    if (amount > this.availableCredit) {
      throw new Error(`Insufficient credit. Available: $${this.availableCredit}`);
    }
    
    this.balance -= amount; // Balance goes negative
    this.availableCredit = this.creditLimit - Math.abs(this.balance);
    this._recordTransaction('purchase', amount);
    return this.balance;
  }
  
  // Charge means making a purchase
  charge(amount, merchant) {
    console.log(`Charging $${amount} at ${merchant}`);
    return this.withdraw(amount);
  }
  
  // Calculate interest on carried balance
  calculateInterest() {
    if (this.balance >= 0) return 0; // No interest if paid in full
    
    const dailyRate = this.apr / 365;
    const daysInMonth = 30;
    return Math.abs(this.balance) * dailyRate * daysInMonth;
  }
  
  // Apply monthly interest
  applyInterest() {
    const interest = this.calculateInterest();
    if (interest > 0) {
      this.balance -= interest; // Adds to debt
      this.availableCredit = this.creditLimit - Math.abs(this.balance);
      this._recordTransaction('interest_charge', interest);
    }
    return interest;
  }
  
  // Calculate minimum payment due
  getMinimumPayment() {
    const percentagePayment = Math.abs(this.balance) * this.minimumPaymentPercent;
    const minimumFloor = 25;
    return Math.max(percentagePayment, minimumFloor);
  }
  
  // Override method
  getAccountType() {
    return 'Credit Card';
  }
  
  // Override balance display to show available credit
  getBalance() {
    return {
      currentBalance: this.balance,
      availableCredit: this.availableCredit,
      creditLimit: this.creditLimit,
      utilizationRate: (Math.abs(this.balance) / this.creditLimit * 100).toFixed(2) + '%'
    };
  }
}

// Premium Account with additional features
class PremiumAccount extends CheckingAccount {
  constructor(accountNumber, customerName, initialBalance) {
    super(accountNumber, customerName, initialBalance, 2000); // Higher overdraft
    this.cashbackRate = 0.02; // 2% cashback on purchases
    this.monthlyFee = 25; // Higher fee but more benefits
    this.rewardPoints = 0;
    this.personalBanker = null;
  }
  
  // Override withdraw to earn rewards
  withdraw(amount) {
    const balance = super.withdraw(amount);
    
    // Earn cashback on withdrawals (purchases)
    const cashback = amount * this.cashbackRate;
    this.rewardPoints += Math.floor(cashback * 100); // Convert to points
    
    return balance;
  }
  
  // Override to waive fees (premium perk)
  applyMonthlyFee() {
    // Premium accounts don't pay fees if balance > $5000
    if (this.balance >= 5000) {
      return;
    }
    
    super.applyMonthlyFee();
  }
  
  // Premium-specific method
  redeemPoints(points) {
    if (points > this.rewardPoints) {
      throw new Error('Insufficient reward points');
    }
    
    const cashValue = points / 100; // 100 points = $1
    this.rewardPoints -= points;
    this.balance += cashValue;
    this._recordTransaction('rewards_redemption', cashValue);
    
    return cashValue;
  }
  
  assignPersonalBanker(bankerName) {
    this.personalBanker = bankerName;
  }
  
  getAccountType() {
    return 'Premium Account';
  }
}

// Usage Examples
console.log('=== Banking Account Hierarchy ===\n');

// 1. Create different account types
const checking = new CheckingAccount('CHK-001', 'Alice Johnson', 1000);
const savings = new SavingsAccount('SAV-001', 'Bob Smith', 5000, 0.03);
const credit = new CreditAccount('CRD-001', 'Charlie Brown', 10000, 0.15);
const premium = new PremiumAccount('PRM-001', 'Diana Prince', 10000);

console.log('Created accounts:');
console.log(`- ${checking.getAccountType()}: ${checking.accountNumber}`);
console.log(`- ${savings.getAccountType()}: ${savings.accountNumber}`);
console.log(`- ${credit.getAccountType()}: ${credit.accountNumber}`);
console.log(`- ${premium.getAccountType()}: ${premium.accountNumber}`);
console.log(`Total accounts: ${BankAccount.getTotalAccounts()}\n`);

// 2. Polymorphism - same method, different behavior
console.log('=== Polymorphism Example ===');
const accounts = [checking, savings, credit, premium];

accounts.forEach(account => {
  console.log(`\n${account.getAccountType()}:`);
  console.log(`Interest: $${account.calculateInterest().toFixed(2)}`);
});

// 3. Checking account operations
console.log('\n=== Checking Account Operations ===');
checking.deposit(500);
console.log('After deposit:', `$${checking.getBalance()}`);

checking.writeCheck('Electric Company', 150);
console.log('After check:', `$${checking.getBalance()}`);

// Try to overdraw
try {
  checking.withdraw(2000); // Will use overdraft
  console.log('After overdraft withdrawal:', `$${checking.getBalance()}`);
} catch (error) {
  console.log('Withdrawal failed:', error.message);
}

// 4. Savings account with restrictions
console.log('\n=== Savings Account Operations ===');
savings.deposit(1000);
console.log('After deposit:', `$${savings.getBalance()}`);

// Make several withdrawals
for (let i = 1; i <= 3; i++) {
  savings.withdraw(100);
  console.log(`Withdrawal ${i}:`, `$${savings.getBalance()}`);
}

// Try to exceed withdrawal limit
try {
  for (let i = 4; i <= 7; i++) {
    savings.withdraw(50);
  }
} catch (error) {
  console.log('Withdrawal blocked:', error.message);
}

// Apply interest
const interest = savings.applyInterest();
console.log(`Interest applied: $${interest.toFixed(2)}`);
console.log(`New balance: $${savings.getBalance()}`);

// Project future balance
const projected = savings.getProjectedBalance(12);
console.log(`Projected balance in 12 months: $${projected.toFixed(2)}`);

// 5. Credit card operations
console.log('\n=== Credit Card Operations ===');
console.log('Initial:', credit.getBalance());

credit.charge(500, 'Amazon');
credit.charge(200, 'Restaurant');
credit.charge(300, 'Gas Station');
console.log('\nAfter purchases:', credit.getBalance());

// Make payment
credit.deposit(400);
console.log('\nAfter payment:', credit.getBalance());

// Apply interest on remaining balance
const creditInterest = credit.applyInterest();
console.log(`\nInterest charged: $${creditInterest.toFixed(2)}`);
console.log('Balance after interest:', credit.getBalance());

const minPayment = credit.getMinimumPayment();
console.log(`\nMinimum payment due: $${minPayment.toFixed(2)}`);

// 6. Premium account features
console.log('\n=== Premium Account Features ===');
premium.assignPersonalBanker('Michael Scott');
console.log(`Personal banker assigned: ${premium.personalBanker}`);

// Earn rewards
premium.withdraw(100); // Purchase
premium.withdraw(200); // Purchase
console.log(`Reward points earned: ${premium.rewardPoints}`);

// Redeem rewards
const redeemed = premium.redeemPoints(600);
console.log(`Redeemed $${redeemed.toFixed(2)} from rewards`);
console.log(`Balance after redemption: $${premium.getBalance()}`);

// 7. Inheritance chain verification
console.log('\n=== Inheritance Verification ===');
console.log('premium instanceof PremiumAccount:', premium instanceof PremiumAccount);
console.log('premium instanceof CheckingAccount:', premium instanceof CheckingAccount);
console.log('premium instanceof BankAccount:', premium instanceof BankAccount);
console.log('premium instanceof Object:', premium instanceof Object);

// 8. Method overriding demonstration
console.log('\n=== Method Overriding ===');
console.log('Checking displayInfo:');
console.log(checking.displayInfo());

console.log('\nCredit displayInfo:');
console.log(credit.displayInfo());

// 9. Static methods
console.log('\n=== Static Methods ===');
console.log('Total accounts created:', BankAccount.getTotalAccounts());
console.log('Checking min balance:', CheckingAccount.minBalance);
console.log('Savings min balance:', SavingsAccount.minBalance);

// 10. Transaction history (inherited method)
console.log('\n=== Transaction History ===');
const history = checking.getTransactionHistory();
console.log(`${checking.getAccountType()} has ${history.length} transactions:`);
history.forEach((txn, idx) => {
  console.log(`${idx + 1}. ${txn.type}: $${txn.amount.toFixed(2)} (Balance: $${txn.balance.toFixed(2)})`);
});
```

### Key Prototypes & Inheritance Takeaways

1. **Base Class** - `BankAccount` provides common functionality
2. **Inheritance** - `extends` keyword creates prototype chain
3. **super()** - Calls parent constructor and methods
4. **Method Overriding** - Child classes customize behavior
5. **Polymorphism** - Same method name, different implementations
6. **Static Members** - Shared across all instances
7. **instanceof** - Check inheritance chain
8. **Protected Methods** - Underscore convention (\_method)
9. **Abstract Methods** - Methods meant to be overridden
10. **Composition** - Complex objects from simple building blocks

---

## this Keyword

The `this` keyword refers to the context in which a function is executed.

```javascript
// Global context
console.log(this); // Window (browser) or global (Node.js)

// Object method
const user = {
  name: "Alice",
  greet: function() {
    console.log(this.name); // "Alice"
  }
};

user.greet();

// Constructor function
function Person(name) {
  this.name = name;
  this.greet = function() {
    console.log(this.name);
  };
}

const person = new Person("Bob");
person.greet(); // "Bob"

// Arrow functions (lexical this)
const obj = {
  name: "Charlie",
  regularFunc: function() {
    console.log(this.name); // "Charlie"
  },
  arrowFunc: () => {
    console.log(this.name); // undefined (inherits from outer scope)
  }
};

// call, apply, bind
function introduce(greeting, punctuation) {
  return `${greeting}, I'm ${this.name}${punctuation}`;
}

const user1 = { name: "David" };

// call - invoke immediately with arguments
console.log(introduce.call(user1, "Hello", "!"));  // "Hello, I'm David!"

// apply - invoke immediately with array of arguments
console.log(introduce.apply(user1, ["Hi", "."]));  // "Hi, I'm David."

// bind - create new function with bound this
const boundFunc = introduce.bind(user1);
console.log(boundFunc("Hey", "...")); // "Hey, I'm David..."

// this in event handlers
const button = document.createElement('button');
button.addEventListener('click', function() {
  console.log(this); // button element
});

button.addEventListener('click', () => {
  console.log(this); // lexical scope (not button)
});
```

### 🏦 Banking Example: Understanding `this` Context

```javascript
// Banking system demonstrating all aspects of `this` keyword

class Account {
  constructor(accountNumber, customerName, balance) {
    this.accountNumber = accountNumber;
    this.customerName = customerName;
    this.balance = balance;
    this.transactions = [];
  }
  
  // Regular method - this refers to the account instance
  deposit(amount) {
    this.balance += amount;
    this.transactions.push({
      type: 'deposit',
      amount,
      balance: this.balance,
      timestamp: new Date()
    });
    return this.balance;
  }
  
  // Method that returns this for chaining
  withdraw(amount) {
    if (amount > this.balance) {
      throw new Error('Insufficient funds');
    }
    this.balance -= amount;
    this.transactions.push({
      type: 'withdrawal',
      amount,
      balance: this.balance,
      timestamp: new Date()
    });
    return this; // Return this for method chaining
  }
  
  // Arrow function as property - `this` is lexically bound
  getInfo = () => {
    // `this` always refers to the Account instance, even if extracted
    return `Account: ${this.accountNumber}, Balance: $${this.balance}`;
  }
  
  // Regular method for comparison
  getInfoRegular() {
    return `Account: ${this.accountNumber}, Balance: $${this.balance}`;
  }
  
  // Method that uses a callback
  processTransactions(callback) {
    this.transactions.forEach((txn, index) => {
      // Using arrow function preserves `this` context
      callback.call(this, txn, index);
    });
  }
  
  // Method that loses `this` context if not careful
  scheduleTransaction(amount, delay) {
    // ❌ Wrong - this will be undefined or window in setTimeout
    // setTimeout(function() {
    //   this.deposit(amount);
    // }, delay);
    
    // ✅ Correct - arrow function preserves this
    setTimeout(() => {
      this.deposit(amount);
      console.log(`Scheduled deposit processed: $${amount}`);
    }, delay);
  }
}

// Banking service object with various this contexts
const BankingService = {
  serviceName: 'Premier Banking',
  accounts: new Map(),
  
  // Regular method - this refers to BankingService
  createAccount(accountNumber, customerName, initialBalance) {
    const account = new Account(accountNumber, customerName, initialBalance);
    this.accounts.set(accountNumber, account);
    console.log(`${this.serviceName}: Account created for ${customerName}`);
    return account;
  },
  
  // Method with nested function - this context issue
  processMultipleDeposits(accountNumber, deposits) {
    const account = this.accounts.get(accountNumber);
    
    // ❌ Problem: regular function loses this context
    // deposits.forEach(function(amount) {
    //   console.log(this.serviceName); // undefined!
    // });
    
    // ✅ Solution 1: Arrow function
    deposits.forEach(amount => {
      console.log(`${this.serviceName}: Processing $${amount}`);
      account.deposit(amount);
    });
    
    // ✅ Solution 2: bind this
    // deposits.forEach(function(amount) {
    //   console.log(this.serviceName);
    // }.bind(this));
    
    return account;
  },
  
  // Arrow function as method - this does NOT refer to BankingService!
  arrowMethod: () => {
    // ⚠️ Arrow functions don't have their own this
    // this will refer to the outer scope (global/window)
    console.log('Arrow method this:', this);
    // console.log(this.serviceName); // undefined
  }
};

// Transaction processor with bind, call, apply
const TransactionProcessor = {
  processorName: 'SecureProcessor',
  feeRate: 0.02,
  
  calculateFee(amount) {
    return amount * this.feeRate;
  },
  
  processPayment(account, amount, merchant) {
    const fee = this.calculateFee(amount);
    const total = amount + fee;
    
    console.log(`${this.processorName}: Processing $${amount} + $${fee} fee = $${total}`);
    account.withdraw(total);
    
    return {
      amount,
      fee,
      total,
      processor: this.processorName
    };
  }
};

// Alternative processor with different fee rate
const PremiumProcessor = {
  processorName: 'PremiumProcessor',
  feeRate: 0.01, // Lower fee rate
};

// Usage Examples
console.log('=== Understanding `this` in Banking Context ===\n');

// 1. Create account - this in object method
console.log('1. Object Method Context:');
const account1 = BankingService.createAccount('ACC-001', 'John Doe', 1000);
const account2 = BankingService.createAccount('ACC-002', 'Jane Smith', 2000);

// 2. Method chaining - returning this
console.log('\n2. Method Chaining (returning this):');
account1
  .withdraw(100)
  .withdraw(50);
console.log('Balance after chaining:', `$${account1.balance}`);

// 3. Extracted method - context loss with regular function
console.log('\n3. Context Loss with Extracted Methods:');

// Extract regular method - LOSES context
const getInfoRegular = account1.getInfoRegular;
try {
  console.log(getInfoRegular()); // ❌ Error: Cannot read properties of undefined
} catch (error) {
  console.log('Error with regular function:', error.message);
}

// Extract arrow function - PRESERVES context
const getInfoArrow = account1.getInfo;
console.log(getInfoArrow()); // ✅ Works! Arrow function keeps original this

// 4. Using bind to fix context
console.log('\n4. Using bind() to Fix Context:');
const boundGetInfo = account1.getInfoRegular.bind(account1);
console.log(boundGetInfo()); // ✅ Now works!

// 5. Using call and apply
console.log('\n5. Using call() and apply():');

// Use call to invoke with specific context
const result1 = TransactionProcessor.processPayment.call(
  TransactionProcessor,
  account1,
  100,
  'Amazon'
);
console.log('Payment result:', result1);

// Use call to "borrow" method with different context
const result2 = TransactionProcessor.processPayment.call(
  PremiumProcessor, // Different this context!
  account2,
  200,
  'Walmart'
);
console.log('Premium payment result:', result2);

// apply is same as call but takes array of arguments
const result3 = TransactionProcessor.processPayment.apply(
  TransactionProcessor,
  [account1, 50, 'Gas Station']
);
console.log('Apply result:', result3);

// 6. Callback with this context
console.log('\n6. Callbacks and this Context:');

BankingService.processMultipleDeposits('ACC-001', [100, 200, 300]);
console.log('Account 1 balance after deposits:', `$${account1.balance}`);

// 7. processTransactions with callback
console.log('\n7. Process Transactions with Callback:');

account1.processTransactions(function(txn, index) {
  // `this` here is bound by call() in the method
  console.log(`Transaction ${index + 1}: ${txn.type} $${txn.amount}`);
});

// 8. setTimeout and this context
console.log('\n8. Async Operations (setTimeout):');
account2.scheduleTransaction(500, 1000);
console.log('Scheduled transaction will process in 1 second...');

// 9. Arrow function behavior in object
console.log('\n9. Arrow Function as Object Method:');
BankingService.arrowMethod(); // this is NOT BankingService!

// 10. Explicit binding with bind
console.log('\n10. Creating Bound Functions:');

// Create a bound version of processPayment for specific processor
const processWithStandardFee = TransactionProcessor.processPayment.bind(
  TransactionProcessor
);

const processWithPremiumFee = TransactionProcessor.processPayment.bind(
  PremiumProcessor
);

console.log('Standard processor:');
processWithStandardFee(account1, 150, 'Restaurant');

console.log('\nPremium processor:');
processWithPremiumFee(account2, 150, 'Restaurant');

// 11. this in constructor functions
console.log('\n11. Constructor Function Context:');

function BankAccount(number, name) {
  // `this` refers to the new instance being created
  this.number = number;
  this.name = name;
  this.balance = 0;
  
  this.deposit = function(amount) {
    this.balance += amount;
  };
}

const oldStyleAccount = new BankAccount('OLD-001', 'Bob Johnson');
oldStyleAccount.deposit(500);
console.log(`Old style account balance: $${oldStyleAccount.balance}`);

// 12. this in nested objects
console.log('\n12. Nested Objects and this:');

const ComplexAccount = {
  accountNumber: 'COMPLEX-001',
  customer: {
    name: 'Alice Wonder',
    getAccountInfo: function() {
      // ❌ this refers to customer object, not ComplexAccount
      console.log('Customer this:', this);
      // console.log(this.accountNumber); // undefined
    },
    getAccountInfoArrow: () => {
      // Arrow function inherits this from outer scope
      console.log('Arrow this:', this);
    }
  },
  getCustomerName() {
    // Access nested property
    return this.customer.name;
  }
};

ComplexAccount.customer.getAccountInfo();
console.log('Customer name via outer method:', ComplexAccount.getCustomerName());

// 13. Practical example: Event handlers simulation
console.log('\n13. Event Handler Simulation:');

class ATM {
  constructor(accountNumber) {
    this.accountNumber = accountNumber;
    this.balance = 1000;
  }
  
  // Simulate button click handlers
  setupWithdrawButton() {
    // ❌ Wrong way - this will be undefined in callback
    const wrongHandler = function() {
      console.log('Wrong this:', this); // undefined or window
      // this.withdraw(100); // Error!
    };
    
    // ✅ Correct way 1 - arrow function
    const correctHandler = () => {
      this.balance -= 100;
      console.log(`ATM ${this.accountNumber}: Withdrew $100, Balance: $${this.balance}`);
    };
    
    // ✅ Correct way 2 - bind
    const boundHandler = function() {
      this.balance -= 100;
      console.log(`ATM ${this.accountNumber}: Withdrew $100, Balance: $${this.balance}`);
    }.bind(this);
    
    // Simulate clicking button
    console.log('Simulating button clicks:');
    correctHandler();
    boundHandler();
  }
}

const atm = new ATM('ATM-001');
atm.setupWithdrawButton();

// 14. Method borrowing
console.log('\n14. Method Borrowing:');

const account3 = {
  accountNumber: 'ACC-003',
  balance: 5000
};

// Borrow the getInfo method from account1
const info = account1.getInfoRegular.call(account3);
console.log('Borrowed method result:', info);

// Final summary
console.log('\n=== Summary of `this` Contexts ===');
console.log('✅ Regular method: this = object that calls it');
console.log('✅ Arrow function: this = lexical scope (where defined)');
console.log('✅ Constructor: this = new instance');
console.log('✅ call/apply: this = explicitly set context');
console.log('✅ bind: creates new function with fixed this');
console.log('⚠️  Extracted method: loses context (use bind/arrow)');
console.log('⚠️  setTimeout callback: use arrow function or bind');
```

### Key `this` Keyword Takeaways

1. **Object Method** - `this` refers to the object
2. **Arrow Functions** - Lexically bind `this` from outer scope
3. **Extracted Methods** - Regular functions lose context
4. **call()** - Invoke with specific `this`, individual arguments
5. **apply()** - Invoke with specific `this`, array of arguments
6. **bind()** - Create new function with permanent `this`
7. **Callbacks** - Use arrow functions to preserve `this`
8. **setTimeout/setInterval** - Always use arrow functions or bind
9. **Event Handlers** - Regular functions get element as `this`
10. **Method Chaining** - Return `this` to enable chaining

---

## Closures

A closure is a function that has access to variables in its outer (enclosing) scope, even after the outer function has returned.

```javascript
// Basic closure
function outer() {
  const message = "Hello";
  
  function inner() {
    console.log(message); // Has access to outer variable
  }
  
  return inner;
}

const greet = outer();
greet(); // "Hello" - closure remembers 'message'

// Practical example: Counter
function createCounter() {
  let count = 0;
  
  return {
    increment: function() {
      count++;
      return count;
    },
    decrement: function() {
      count--;
      return count;
    },
    getCount: function() {
      return count;
    }
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount());  // 2
console.log(counter.decrement()); // 1

// Private variables with closures
function BankAccount(initialBalance) {
  let balance = initialBalance; // Private variable
  
  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount <= balance) {
        balance -= amount;
        return balance;
      }
      throw new Error("Insufficient funds");
    },
    getBalance() {
      return balance;
    }
  };
}

const account = new BankAccount(1000);
console.log(account.deposit(500));   // 1500
console.log(account.withdraw(200));  // 1300
console.log(account.getBalance());   // 1300
// console.log(account.balance);     // undefined (private)

// Function factory with closures
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15

// Module pattern
const calculator = (function() {
  let result = 0;
  
  return {
    add(x) {
      result += x;
      return this;
    },
    subtract(x) {
      result -= x;
      return this;
    },
    multiply(x) {
      result *= x;
      return this;
    },
    divide(x) {
      result /= x;
      return this;
    },
    getResult() {
      return result;
    },
    clear() {
      result = 0;
      return this;
    }
  };
})();

calculator.add(10).multiply(2).subtract(5);
console.log(calculator.getResult()); // 15
```

### 🏦 Banking Example: Advanced Closure Patterns

```javascript
// Comprehensive banking system using closures for data privacy and encapsulation

// 1. Secure Bank Account with Private Data
function createSecureBankAccount(accountNumber, customerName, initialBalance, pin) {
  // All these variables are private (only accessible via closures)
  let balance = initialBalance;
  let transactionHistory = [];
  let isLocked = false;
  let failedAttempts = 0;
  const MAX_ATTEMPTS = 3;
  const correctPin = pin;
  
  // Private helper functions (closures)
  function recordTransaction(type, amount, success = true) {
    transactionHistory.push({
      type,
      amount,
      balance,
      success,
      timestamp: new Date()
    });
  }
  
  function validatePin(inputPin) {
    if (isLocked) {
      throw new Error('Account is locked due to multiple failed attempts');
    }
    
    if (inputPin !== correctPin) {
      failedAttempts++;
      if (failedAttempts >= MAX_ATTEMPTS) {
        isLocked = true;
        throw new Error('Account locked: Too many failed PIN attempts');
      }
      throw new Error(`Invalid PIN. ${MAX_ATTEMPTS - failedAttempts} attempts remaining`);
    }
    
    failedAttempts = 0; // Reset on successful validation
    return true;
  }
  
  // Public interface (these functions form closures)
  return {
    deposit(amount, inputPin) {
      validatePin(inputPin);
      
      if (amount <= 0) {
        throw new Error('Deposit amount must be positive');
      }
      
      balance += amount;
      recordTransaction('deposit', amount);
      return {
        success: true,
        newBalance: balance,
        message: `Deposited $${amount}`
      };
    },
    
    withdraw(amount, inputPin) {
      validatePin(inputPin);
      
      if (amount <= 0) {
        throw new Error('Withdrawal amount must be positive');
      }
      
      if (amount > balance) {
        recordTransaction('withdrawal', amount, false);
        throw new Error('Insufficient funds');
      }
      
      balance -= amount;
      recordTransaction('withdrawal', amount);
      return {
        success: true,
        newBalance: balance,
        message: `Withdrew $${amount}`
      };
    },
    
    getBalance(inputPin) {
      validatePin(inputPin);
      return balance;
    },
    
    getTransactionHistory(inputPin) {
      validatePin(inputPin);
      // Return a copy to prevent external modification
      return transactionHistory.map(t => ({...t}));
    },
    
    getAccountInfo(inputPin) {
      validatePin(inputPin);
      return {
        accountNumber,
        customerName,
        balance,
        transactionCount: transactionHistory.length,
        isLocked
      };
    },
    
    changePin(oldPin, newPin) {
      validatePin(oldPin);
      
      if (newPin.length !== 4) {
        throw new Error('PIN must be 4 digits');
      }
      
      // This creates a new closure over the new PIN
      correctPin = newPin; // This won't work as correctPin is const
      // In real implementation, we'd need a different approach
      recordTransaction('pin_change', 0);
      return { success: true, message: 'PIN changed successfully' };
    }
  };
}

// 2. Interest Calculator Factory with Closures
function createInterestCalculator(accountType) {
  // Private rate configuration
  const rates = {
    savings: 0.025,    // 2.5%
    checking: 0.001,   // 0.1%
    premium: 0.035,    // 3.5%
    business: 0.02     // 2.0%
  };
  
  const rate = rates[accountType] || 0;
  
  // Closure remembers the rate
  return {
    calculateMonthly(balance) {
      return balance * (rate / 12);
    },
    
    calculateYearly(balance) {
      return balance * rate;
    },
    
    calculateCompound(principal, years) {
      // Compound interest: A = P(1 + r/n)^(nt), where n=12 (monthly)
      return principal * Math.pow(1 + rate / 12, 12 * years);
    },
    
    getRate() {
      return rate;
    },
    
    getAccountType() {
      return accountType;
    }
  };
}

// 3. Transaction Processor with Rate Limiting (Closure + Timing)
function createTransactionProcessor(maxPerMinute = 10) {
  // Private state
  let transactions = [];
  let processingCount = 0;
  
  // Private function to clean old transactions
  function cleanOldTransactions() {
    const oneMinuteAgo = Date.now() - 60000;
    transactions = transactions.filter(t => t.timestamp > oneMinuteAgo);
  }
  
  return {
    processTransaction(accountId, amount, type) {
      cleanOldTransactions();
      
      // Rate limiting check
      if (transactions.length >= maxPerMinute) {
        throw new Error('Rate limit exceeded. Please try again later.');
      }
      
      // Simulate processing
      const transaction = {
        id: `TXN-${Date.now()}-${processingCount++}`,
        accountId,
        amount,
        type,
        timestamp: Date.now(),
        status: 'processed'
      };
      
      transactions.push(transaction);
      return transaction;
    },
    
    getRecentTransactions() {
      cleanOldTransactions();
      return [...transactions]; // Return copy
    },
    
    getRateLimitStatus() {
      cleanOldTransactions();
      return {
        used: transactions.length,
        limit: maxPerMinute,
        remaining: maxPerMinute - transactions.length
      };
    }
  };
}

// 4. Account Factory with Multiple Closures
function createBankingAccountFactory() {
  // Shared private state across all accounts
  let accountCounter = 1000;
  const allAccounts = new Map();
  
  return {
    createAccount(customerName, accountType, initialBalance, pin) {
      const accountNumber = `ACC-${++accountCounter}`;
      
      // Create account with its own closures
      const account = createSecureBankAccount(
        accountNumber,
        customerName,
        initialBalance,
        pin
      );
      
      // Add interest calculator
      const interestCalc = createInterestCalculator(accountType);
      
      // Enhanced account with additional closures
      const enhancedAccount = {
        ...account,
        
        calculateInterest(inputPin) {
          const balance = account.getBalance(inputPin);
          return {
            monthly: interestCalc.calculateMonthly(balance),
            yearly: interestCalc.calculateYearly(balance),
            projectedIn5Years: interestCalc.calculateCompound(balance, 5)
          };
        },
        
        getAccountType() {
          return accountType;
        }
      };
      
      // Store in factory's private map
      allAccounts.set(accountNumber, enhancedAccount);
      
      return {
        accountNumber,
        account: enhancedAccount
      };
    },
    
    getAccount(accountNumber) {
      return allAccounts.get(accountNumber);
    },
    
    getTotalAccounts() {
      return allAccounts.size;
    },
    
    getAllAccountNumbers() {
      return Array.from(allAccounts.keys());
    }
  };
}

// 5. Memoization with Closures (Performance Optimization)
function createMemoizedFeeCalculator() {
  // Cache stored in closure
  const cache = new Map();
  let cacheHits = 0;
  let cacheMisses = 0;
  
  return {
    calculateFee(amount, transactionType) {
      const cacheKey = `${amount}-${transactionType}`;
      
      // Check cache
      if (cache.has(cacheKey)) {
        cacheHits++;
        return cache.get(cacheKey);
      }
      
      // Calculate fee (expensive operation simulation)
      cacheMisses++;
      let fee = 0;
      
      if (transactionType === 'wire') {
        fee = Math.max(25, amount * 0.02);
      } else if (transactionType === 'international') {
        fee = Math.max(35, amount * 0.03);
      } else if (transactionType === 'standard') {
        fee = amount > 1000 ? amount * 0.001 : 1;
      }
      
      // Store in cache
      cache.set(cacheKey, fee);
      return fee;
    },
    
    getCacheStats() {
      return {
        size: cache.size,
        hits: cacheHits,
        misses: cacheMisses,
        hitRate: cacheHits / (cacheHits + cacheMisses)
      };
    },
    
    clearCache() {
      cache.clear();
      cacheHits = 0;
      cacheMisses = 0;
    }
  };
}

// 6. Partial Application with Closures
function createTransferFunction(fromAccount) {
  // First closure captures fromAccount
  return function(toAccount) {
    // Second closure captures toAccount
    return function(amount, pin) {
      // Final closure performs the transfer
      console.log(`Transferring $${amount} from ${fromAccount} to ${toAccount}`);
      
      // Simulate withdrawal and deposit
      return {
        from: fromAccount,
        to: toAccount,
        amount,
        timestamp: new Date(),
        status: 'completed'
      };
    };
  };
}

// Usage Examples
console.log('=== Advanced Closures in Banking ===\n');

// 1. Secure account with private data
console.log('1. Secure Account with Closures:');
const myAccount = createSecureBankAccount('ACC-001', 'John Doe', 1000, '1234');

try {
  const result = myAccount.deposit(500, '1234');
  console.log(result.message, '- Balance:', `$${result.newBalance}`);
  
  const balance = myAccount.getBalance('1234');
  console.log('Current balance:', `$${balance}`);
  
  // Try to access private data directly
  console.log('Direct balance access:', myAccount.balance); // undefined
  console.log('Private data is truly private! ✅');
} catch (error) {
  console.log('Error:', error.message);
}

// 2. Failed PIN attempts
console.log('\n2. PIN Validation with Closures:');
const lockedAccount = createSecureBankAccount('ACC-002', 'Jane Smith', 2000, '5678');

try {
  lockedAccount.getBalance('0000'); // Wrong PIN
} catch (error) {
  console.log('Attempt 1:', error.message);
}

try {
  lockedAccount.getBalance('1111'); // Wrong PIN
} catch (error) {
  console.log('Attempt 2:', error.message);
}

try {
  lockedAccount.getBalance('2222'); // Wrong PIN
} catch (error) {
  console.log('Attempt 3:', error.message);
}

try {
  lockedAccount.getBalance('5678'); // Correct PIN but account locked
} catch (error) {
  console.log('Account locked:', error.message);
}

// 3. Interest calculator factory
console.log('\n3. Interest Calculator Factory:');
const savingsCalc = createInterestCalculator('savings');
const checkingCalc = createInterestCalculator('checking');
const premiumCalc = createInterestCalculator('premium');

const principal = 10000;
console.log(`Principal: $${principal}`);
console.log(`Savings (${(savingsCalc.getRate() * 100).toFixed(2)}%): $${savingsCalc.calculateYearly(principal).toFixed(2)}/year`);
console.log(`Checking (${(checkingCalc.getRate() * 100).toFixed(2)}%): $${checkingCalc.calculateYearly(principal).toFixed(2)}/year`);
console.log(`Premium (${(premiumCalc.getRate() * 100).toFixed(2)}%): $${premiumCalc.calculateYearly(principal).toFixed(2)}/year`);

// 4. Transaction processor with rate limiting
console.log('\n4. Rate Limiting with Closures:');
const processor = createTransactionProcessor(5); // Max 5 per minute

for (let i = 1; i <= 7; i++) {
  try {
    const txn = processor.processTransaction('ACC-001', 100, 'deposit');
    console.log(`Transaction ${i}: ${txn.id} - ${txn.status}`);
  } catch (error) {
    console.log(`Transaction ${i}: ${error.message}`);
  }
}

const status = processor.getRateLimitStatus();
console.log(`Rate limit status: ${status.used}/${status.limit} (${status.remaining} remaining)`);

// 5. Account factory with shared state
console.log('\n5. Account Factory with Shared Closures:');
const factory = createBankingAccountFactory();

const acc1 = factory.createAccount('Alice Johnson', 'savings', 5000, '1111');
const acc2 = factory.createAccount('Bob Wilson', 'premium', 15000, '2222');
const acc3 = factory.createAccount('Carol Davis', 'checking', 3000, '3333');

console.log(`Created ${factory.getTotalAccounts()} accounts:`);
factory.getAllAccountNumbers().forEach(num => console.log(`  - ${num}`));

// Use accounts
const alice = acc1.account;
alice.deposit(1000, '1111');
const aliceBalance = alice.getBalance('1111');
console.log(`\nAlice's balance: $${aliceBalance}`);

const interest = alice.calculateInterest('1111');
console.log('Alice\'s projected interest:');
console.log(`  Monthly: $${interest.monthly.toFixed(2)}`);
console.log(`  Yearly: $${interest.yearly.toFixed(2)}`);
console.log(`  In 5 years: $${interest.projectedIn5Years.toFixed(2)}`);

// 6. Memoized fee calculator
console.log('\n6. Memoization with Closures:');
const feeCalc = createMemoizedFeeCalculator();

// Calculate fees (cache miss)
console.log('First calculations (cache misses):');
console.log('Wire $1000:', `$${feeCalc.calculateFee(1000, 'wire').toFixed(2)}`);
console.log('International $5000:', `$${feeCalc.calculateFee(5000, 'international').toFixed(2)}`);
console.log('Standard $500:', `$${feeCalc.calculateFee(500, 'standard').toFixed(2)}`);

// Calculate same fees again (cache hit)
console.log('\nSecond calculations (cache hits):');
console.log('Wire $1000:', `$${feeCalc.calculateFee(1000, 'wire').toFixed(2)}`);
console.log('International $5000:', `$${feeCalc.calculateFee(5000, 'international').toFixed(2)}`);

const stats = feeCalc.getCacheStats();
console.log(`\nCache stats: ${stats.hits} hits, ${stats.misses} misses, ${(stats.hitRate * 100).toFixed(1)}% hit rate`);

// 7. Partial application with closures (currying)
console.log('\n7. Partial Application (Currying):');
const transferFromAlice = createTransferFunction(acc1.accountNumber);
const transferFromAliceToBob = transferFromAlice(acc2.accountNumber);
const result = transferFromAliceToBob(500, '1111');
console.log(`Transfer: ${result.from} → ${result.to}: $${result.amount}`);

// Can reuse partially applied functions
const transferFromAliceToCarol = transferFromAlice(acc3.accountNumber);
const result2 = transferFromAliceToCarol(300, '1111');
console.log(`Transfer: ${result2.from} → ${result2.to}: $${result2.amount}`);

// 8. Module pattern with private state
console.log('\n8. Module Pattern with IIFE:');
const BankingModule = (function() {
  // Private state and functions
  let totalTransactions = 0;
  const transactionLog = [];
  
  function logTransaction(type, details) {
    totalTransactions++;
    transactionLog.push({
      id: totalTransactions,
      type,
      details,
      timestamp: new Date()
    });
  }
  
  // Public API
  return {
    processDeposit(account, amount) {
      logTransaction('deposit', { account, amount });
      return { success: true, transactionId: totalTransactions };
    },
    
    processWithdrawal(account, amount) {
      logTransaction('withdrawal', { account, amount });
      return { success: true, transactionId: totalTransactions };
    },
    
    getTransactionCount() {
      return totalTransactions;
    },
    
    getTransactionLog() {
      return [...transactionLog]; // Return copy
    }
  };
})();

BankingModule.processDeposit('ACC-001', 100);
BankingModule.processWithdrawal('ACC-001', 50);
BankingModule.processDeposit('ACC-002', 200);

console.log(`Total transactions: ${BankingModule.getTransactionCount()}`);
console.log('Cannot access private data:', typeof BankingModule.transactionLog); // undefined

// Final demonstration: Closures remember their scope
console.log('\n9. Closure Scope Chain:');
function createAccountManager(bankName) {
  return function(branchName) {
    return function(accountType) {
      return function(customerName) {
        // All outer variables are accessible via closure chain
        return `${customerName}'s ${accountType} account at ${bankName} - ${branchName} branch`;
      };
    };
  };
}

const boaManager = createAccountManager('Bank of America');
const nyBranch = boaManager('New York');
const savingsAccount = nyBranch('Savings');
const customerAccount = savingsAccount('John Smith');

console.log(customerAccount);
```

### Key Closures Takeaways

1. **Data Privacy** - Variables in outer scope are private
2. **State Retention** - Closures remember their environment
3. **Factory Functions** - Create multiple instances with private state
4. **Module Pattern** - IIFE + closures = private data + public API
5. **Memoization** - Cache results using closure-scoped Map
6. **Partial Application** - Create specialized functions from general ones
7. **Event Handlers** - Closures capture context for async operations
8. **Rate Limiting** - Track state over time with closures
9. **Encapsulation** - Hide implementation details
10. **Function Composition** - Chain functions while preserving scope

---

## Error Handling

### try...catch...finally

```javascript
// Basic error handling
try {
  // Code that might throw an error
  const result = riskyOperation();
  console.log(result);
} catch (error) {
  // Handle the error
  console.error("An error occurred:", error.message);
} finally {
  // Always executes (cleanup code)
  console.log("Cleanup complete");
}

// Throwing custom errors
function divide(a, b) {
  if (b === 0) {
    throw new Error("Division by zero");
  }
  return a / b;
}

try {
  const result = divide(10, 0);
} catch (error) {
  console.error(error.message); // "Division by zero"
}

// Custom error types
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = "ValidationError";
  }
}

class DatabaseError extends Error {
  constructor(message) {
    super(message);
    this.name = "DatabaseError";
  }
}

function validateUser(user) {
  if (!user.email) {
    throw new ValidationError("Email is required");
  }
  if (!user.password || user.password.length < 8) {
    throw new ValidationError("Password must be at least 8 characters");
  }
}

try {
  validateUser({ email: "test@example.com", password: "123" });
} catch (error) {
  if (error instanceof ValidationError) {
    console.error("Validation failed:", error.message);
  } else if (error instanceof DatabaseError) {
    console.error("Database error:", error.message);
  } else {
    console.error("Unknown error:", error);
  }
}

// Async error handling
async function fetchData(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Fetch failed:", error);
    throw error; // Re-throw if needed
  }
}

// Promise error handling
fetchData('https://api.example.com/data')
  .then(data => console.log(data))
  .catch(error => console.error(error))
  .finally(() => console.log("Request complete"));
```

### 🏦 Banking Example: Comprehensive Error Handling System

```javascript
// Custom banking error hierarchy

// Base banking error
class BankingError extends Error {
  constructor(message, code, details = {}) {
    super(message);
    this.name = 'BankingError';
    this.code = code;
    this.details = details;
    this.timestamp = new Date();
    
    // Capture stack trace
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      details: this.details,
      timestamp: this.timestamp
    };
  }
}

// Specific error types
class InsufficientFundsError extends BankingError {
  constructor(requested, available, accountNumber) {
    super(
      `Insufficient funds. Requested: $${requested}, Available: $${available}`,
      'ERR_INSUFFICIENT_FUNDS',
      { requested, available, accountNumber }
    );
    this.name = 'InsufficientFundsError';
  }
}

class InvalidAmountError extends BankingError {
  constructor(amount, reason) {
    super(
      `Invalid amount: ${reason}`,
      'ERR_INVALID_AMOUNT',
      { amount, reason }
    );
    this.name = 'InvalidAmountError';
  }
}

class AccountLockedError extends BankingError {
  constructor(accountNumber, reason) {
    super(
      `Account ${accountNumber} is locked: ${reason}`,
      'ERR_ACCOUNT_LOCKED',
      { accountNumber, reason }
    );
    this.name = 'AccountLockedError';
  }
}

class DailyLimitExceededError extends BankingError {
  constructor(attempted, limit, used) {
    super(
      `Daily limit exceeded. Attempted: $${attempted}, Limit: $${limit}, Already used: $${used}`,
      'ERR_DAILY_LIMIT',
      { attempted, limit, used, remaining: limit - used }
    );
    this.name = 'DailyLimitExceededError';
  }
}

class FraudDetectedError extends BankingError {
  constructor(transactionId, reason) {
    super(
      `Potential fraud detected: ${reason}`,
      'ERR_FRAUD_DETECTED',
      { transactionId, reason, severity: 'HIGH' }
    );
    this.name = 'FraudDetectedError';
  }
}

class AuthenticationError extends BankingError {
  constructor(attemptsRemaining) {
    super(
      `Authentication failed. ${attemptsRemaining} attempts remaining`,
      'ERR_AUTH_FAILED',
      { attemptsRemaining }
    );
    this.name = 'AuthenticationError';
  }
}

class NetworkError extends BankingError {
  constructor(operation, originalError) {
    super(
      `Network error during ${operation}`,
      'ERR_NETWORK',
      { operation, originalError: originalError.message }
    );
    this.name = 'NetworkError';
  }
}

class ValidationError extends BankingError {
  constructor(field, message) {
    super(
      `Validation error for ${field}: ${message}`,
      'ERR_VALIDATION',
      { field, validation: message }
    );
    this.name = 'ValidationError';
  }
}

// Banking transaction processor with comprehensive error handling
class TransactionProcessor {
  constructor(accountNumber, initialBalance) {
    this.accountNumber = accountNumber;
    this.balance = initialBalance;
    this.dailyLimit = 5000;
    this.dailyUsed = 0;
    this.isLocked = false;
    this.failedAuthAttempts = 0;
    this.transactionHistory = [];
  }
  
  // Validate transaction amount
  validateAmount(amount, operation) {
    if (typeof amount !== 'number') {
      throw new InvalidAmountError(amount, 'Amount must be a number');
    }
    
    if (isNaN(amount)) {
      throw new InvalidAmountError(amount, 'Amount is not a valid number');
    }
    
    if (amount <= 0) {
      throw new InvalidAmountError(amount, 'Amount must be positive');
    }
    
    if (amount > 1000000) {
      throw new InvalidAmountError(amount, 'Amount exceeds maximum transaction limit');
    }
    
    // Check for precision issues (banking requires exact amounts)
    if (!Number.isInteger(amount * 100)) {
      throw new InvalidAmountError(amount, 'Amount has more than 2 decimal places');
    }
  }
  
  // Check account status
  checkAccountStatus() {
    if (this.isLocked) {
      throw new AccountLockedError(
        this.accountNumber,
        'Multiple failed authentication attempts'
      );
    }
  }
  
  // Authenticate user
  authenticate(pin) {
    const correctPin = '1234'; // In real system, this would be hashed
    
    if (pin !== correctPin) {
      this.failedAuthAttempts++;
      
      if (this.failedAuthAttempts >= 3) {
        this.isLocked = true;
        throw new AccountLockedError(
          this.accountNumber,
          'Too many failed authentication attempts'
        );
      }
      
      throw new AuthenticationError(3 - this.failedAuthAttempts);
    }
    
    this.failedAuthAttempts = 0; // Reset on successful auth
  }
  
  // Fraud detection
  detectFraud(amount, location) {
    // Simple fraud detection rules
    if (amount > 10000 && location !== 'USA') {
      throw new FraudDetectedError(
        `TXN-${Date.now()}`,
        'Large international transaction'
      );
    }
    
    // Check for rapid successive transactions
    const recentTransactions = this.transactionHistory.filter(
      txn => Date.now() - txn.timestamp < 60000 // Last minute
    );
    
    if (recentTransactions.length > 5) {
      throw new FraudDetectedError(
        `TXN-${Date.now()}`,
        'Too many transactions in short time'
      );
    }
  }
  
  // Check daily limit
  checkDailyLimit(amount) {
    if (this.dailyUsed + amount > this.dailyLimit) {
      throw new DailyLimitExceededError(
        amount,
        this.dailyLimit,
        this.dailyUsed
      );
    }
  }
  
  // Record transaction
  recordTransaction(type, amount, status, error = null) {
    this.transactionHistory.push({
      type,
      amount,
      status,
      error: error ? error.message : null,
      balance: this.balance,
      timestamp: Date.now()
    });
  }
  
  // Withdrawal with comprehensive error handling
  withdraw(amount, pin, location = 'USA') {
    try {
      // Step 1: Authenticate
      this.checkAccountStatus();
      this.authenticate(pin);
      
      // Step 2: Validate amount
      this.validateAmount(amount, 'withdrawal');
      
      // Step 3: Fraud detection
      this.detectFraud(amount, location);
      
      // Step 4: Check daily limit
      this.checkDailyLimit(amount);
      
      // Step 5: Check sufficient funds
      if (amount > this.balance) {
        throw new InsufficientFundsError(
          amount,
          this.balance,
          this.accountNumber
        );
      }
      
      // Step 6: Process withdrawal
      this.balance -= amount;
      this.dailyUsed += amount;
      
      // Step 7: Record success
      this.recordTransaction('withdrawal', amount, 'success');
      
      return {
        success: true,
        newBalance: this.balance,
        transactionId: `TXN-${Date.now()}`,
        message: `Successfully withdrew $${amount}`
      };
      
    } catch (error) {
      // Record failure
      this.recordTransaction('withdrawal', amount, 'failed', error);
      
      // Re-throw for caller to handle
      throw error;
    }
  }
  
  // Deposit with error handling
  deposit(amount, pin) {
    try {
      this.checkAccountStatus();
      this.authenticate(pin);
      this.validateAmount(amount, 'deposit');
      
      // Check for suspicious deposits
      if (amount > 50000) {
        console.warn('Large deposit requires additional verification');
      }
      
      this.balance += amount;
      this.recordTransaction('deposit', amount, 'success');
      
      return {
        success: true,
        newBalance: this.balance,
        transactionId: `TXN-${Date.now()}`
      };
      
    } catch (error) {
      this.recordTransaction('deposit', amount, 'failed', error);
      throw error;
    }
  }
  
  // Transfer with error handling
  async transfer(toAccount, amount, pin) {
    let transactionStarted = false;
    
    try {
      // Validate and authenticate
      this.checkAccountStatus();
      this.authenticate(pin);
      this.validateAmount(amount, 'transfer');
      this.checkDailyLimit(amount);
      
      if (amount > this.balance) {
        throw new InsufficientFundsError(amount, this.balance, this.accountNumber);
      }
      
      // Start transaction
      transactionStarted = true;
      this.balance -= amount;
      
      // Simulate network call to credit receiving account
      try {
        await this.simulateNetworkCall(toAccount, amount);
      } catch (networkError) {
        // Rollback on network error
        this.balance += amount;
        transactionStarted = false;
        throw new NetworkError('transfer', networkError);
      }
      
      // Complete transaction
      this.dailyUsed += amount;
      this.recordTransaction('transfer', amount, 'success');
      
      return {
        success: true,
        from: this.accountNumber,
        to: toAccount,
        amount,
        newBalance: this.balance
      };
      
    } catch (error) {
      // Rollback if transaction was started
      if (transactionStarted) {
        this.balance += amount;
      }
      
      this.recordTransaction('transfer', amount, 'failed', error);
      throw error;
    } finally {
      // Finally block always executes (good for cleanup)
      console.log('Transaction processing complete');
    }
  }
  
  // Simulate network call
  async simulateNetworkCall(toAccount, amount) {
    return new Promise((resolve, reject) => {
      // Simulate random network failure
      setTimeout(() => {
        if (Math.random() < 0.1) { // 10% failure rate
          reject(new Error('Network timeout'));
        } else {
          resolve();
        }
      }, 100);
    });
  }
  
  // Get transaction history
  getHistory() {
    return [...this.transactionHistory];
  }
}

// Error handling utility functions
class ErrorHandler {
  static handle(error) {
    if (error instanceof InsufficientFundsError) {
      return {
        userMessage: 'You don\'t have enough funds for this transaction.',
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: false
      };
    }
    
    if (error instanceof InvalidAmountError) {
      return {
        userMessage: 'Please enter a valid amount.',
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: true
      };
    }
    
    if (error instanceof AccountLockedError) {
      return {
        userMessage: 'Your account has been locked. Please contact support.',
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: false,
        requiresSupport: true
      };
    }
    
    if (error instanceof DailyLimitExceededError) {
      return {
        userMessage: `Daily limit exceeded. You can still spend $${error.details.remaining}.`,
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: false
      };
    }
    
    if (error instanceof FraudDetectedError) {
      return {
        userMessage: 'This transaction has been flagged for review.',
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: false,
        requiresVerification: true
      };
    }
    
    if (error instanceof AuthenticationError) {
      return {
        userMessage: 'Incorrect PIN. Please try again.',
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: true
      };
    }
    
    if (error instanceof NetworkError) {
      return {
        userMessage: 'Network error. Please try again later.',
        technicalMessage: error.message,
        code: error.code,
        details: error.details,
        retry: true
      };
    }
    
    // Unknown error
    return {
      userMessage: 'An unexpected error occurred. Please contact support.',
      technicalMessage: error.message,
      code: 'ERR_UNKNOWN',
      retry: false
    };
  }
  
  static log(error) {
    // In production, this would send to logging service
    console.error('[ERROR LOG]', {
      name: error.name,
      message: error.message,
      code: error.code,
      details: error.details,
      stack: error.stack,
      timestamp: new Date()
    });
  }
}

// Usage Examples
console.log('=== Banking Error Handling Examples ===\n');

const account = new TransactionProcessor('ACC-12345', 1000);

// 1. Successful transaction
console.log('1. Successful Transaction:');
try {
  const result = account.withdraw(100, '1234');
  console.log('✅', result.message);
  console.log('   New balance:', `$${result.newBalance}`);
} catch (error) {
  const handled = ErrorHandler.handle(error);
  console.log('❌', handled.userMessage);
}

// 2. Insufficient funds error
console.log('\n2. Insufficient Funds:');
try {
  account.withdraw(2000, '1234');
} catch (error) {
  console.log('❌ Error caught:', error.name);
  console.log('   Message:', error.message);
  console.log('   Code:', error.code);
  console.log('   Details:', error.details);
  
  const handled = ErrorHandler.handle(error);
  console.log('   User sees:', handled.userMessage);
  console.log('   Can retry?', handled.retry);
}

// 3. Invalid amount error
console.log('\n3. Invalid Amount:');
try {
  account.withdraw(-50, '1234');
} catch (error) {
  console.log('❌ Error:', error.name);
  console.log('   Details:', error.details);
  
  const handled = ErrorHandler.handle(error);
  console.log('   User message:', handled.userMessage);
}

// 4. Multiple failed authentication attempts
console.log('\n4. Failed Authentication:');
for (let i = 1; i <= 4; i++) {
  try {
    account.withdraw(50, 'wrong-pin');
  } catch (error) {
    console.log(`   Attempt ${i}: ${error.name}`);
    
    if (error instanceof AccountLockedError) {
      console.log('   ⚠️ Account is now locked!');
      const handled = ErrorHandler.handle(error);
      console.log('   User message:', handled.userMessage);
      console.log('   Requires support?', handled.requiresSupport);
      break;
    }
  }
}

// 5. Daily limit exceeded
console.log('\n5. Daily Limit Check:');
const account2 = new TransactionProcessor('ACC-67890', 10000);
try {
  // Try to withdraw more than daily limit
  account2.withdraw(6000, '1234');
} catch (error) {
  console.log('❌', error.name);
  const handled = ErrorHandler.handle(error);
  console.log('   User message:', handled.userMessage);
  console.log('   Remaining today:', `$${error.details.remaining}`);
}

// 6. Fraud detection
console.log('\n6. Fraud Detection:');
const account3 = new TransactionProcessor('ACC-11111', 50000);
try {
  // Large international transaction
  account3.withdraw(15000, '1234', 'Russia');
} catch (error) {
  console.log('❌', error.name);
  console.log('   Severity:', error.details.severity);
  const handled = ErrorHandler.handle(error);
  console.log('   User message:', handled.userMessage);
  console.log('   Requires verification?', handled.requiresVerification);
}

// 7. Async transfer with error handling
console.log('\n7. Async Transfer (with network simulation):');
const account4 = new TransactionProcessor('ACC-22222', 5000);

(async () => {
  // Try multiple transfers to test network errors
  for (let i = 1; i <= 3; i++) {
    try {
      console.log(`\n   Attempt ${i}:`);
      const result = await account4.transfer('ACC-99999', 100, '1234');
      console.log('   ✅ Transfer successful!');
      console.log('   New balance:', `$${result.newBalance}`);
      break; // Success, exit loop
    } catch (error) {
      if (error instanceof NetworkError) {
        console.log('   ⚠️ Network error, retrying...');
        const handled = ErrorHandler.handle(error);
        if (handled.retry && i < 3) {
          continue; // Retry
        }
      } else {
        console.log('   ❌', error.name);
        const handled = ErrorHandler.handle(error);
        console.log('   User message:', handled.userMessage);
        break; // Don't retry for non-network errors
      }
    }
  }
  
  // 8. Precise error logging
  console.log('\n8. Error Logging:');
  try {
    account4.withdraw(10000, '1234'); // Will fail
  } catch (error) {
    ErrorHandler.log(error);
    console.log('   Error has been logged for monitoring');
  }
  
  // 9. Transaction history with errors
  console.log('\n9. Transaction History:');
  const history = account4.getHistory();
  console.log(`   Total transactions: ${history.length}`);
  
  const failed = history.filter(t => t.status === 'failed');
  console.log(`   Failed transactions: ${failed.length}`);
  
  failed.forEach(txn => {
    console.log(`   - ${txn.type}: $${txn.amount} - ${txn.error}`);
  });
  
  // 10. Custom error with JSON serialization
  console.log('\n10. Error Serialization:');
  try {
    throw new InsufficientFundsError(500, 100, 'ACC-TEST');
  } catch (error) {
    const json = error.toJSON();
    console.log('   Error as JSON:', JSON.stringify(json, null, 2));
  }
})();
```

### Key Error Handling Takeaways

1. **Custom Error Classes** - Create specific error types for different scenarios
2. **Error Hierarchy** - Base error class with specialized children
3. **Error Codes** - Use codes for programmatic error handling
4. **Error Details** - Include context (amounts, limits, etc.)
5. **try...catch...finally** - finally always executes (cleanup)
6. **Error Recovery** - Rollback on failure (transactions)
7. **Error Logging** - Track errors for monitoring
8. **User-Friendly Messages** - Translate technical errors
9. **Retry Logic** - Determine if operation can be retried
10. **Async Error Handling** - Use try/catch with async/await

---

## Key Takeaways

1. ✅ JavaScript has **8 data types**: 7 primitives + Object
2. ✅ Use `const` by default, `let` when reassignment needed, avoid `var`
3. ✅ Understand **truthy/falsy** values and type coercion
4. ✅ **Arrow functions** don't have their own `this` (lexical binding)
5. ✅ **Event Emitter** pattern enables event-driven programming
6. ✅ **Prototypes** are the foundation of inheritance in JavaScript
7. ✅ **Closures** allow functions to access outer scope variables
8. ✅ The **`this`** keyword depends on how a function is called
9. ✅ Use **try...catch** for error handling
10. ✅ Modern JavaScript offers powerful array/object methods

---

## Interview Questions

### Q1: What's the difference between `==` and `===`?
**A**: `==` (loose equality) performs type coercion before comparison, while `===` (strict equality) compares both value and type without coercion.

```javascript
5 == "5"   // true (coercion)
5 === "5"  // false (different types)
```

### Q2: Explain closures with an example.
**A**: A closure is when an inner function has access to outer function's variables even after the outer function returns.

```javascript
function outer(x) {
  return function(y) {
    return x + y; // Has access to 'x'
  };
}
const addFive = outer(5);
console.log(addFive(3)); // 8
```

### Q3: What is the Event Loop?
**A**: The Event Loop manages asynchronous operations in JavaScript. It continuously checks the call stack and callback queue, moving callbacks to the stack when it's empty.

### Q4: Explain `this` in different contexts.
**A**: `this` refers to:
- Global scope: global object
- Object method: the object
- Constructor: new instance
- Arrow function: lexical `this` (outer scope)
- Event handler: the element (regular function)

### Q5: What's the difference between `null` and `undefined`?
**A**: 
- `undefined`: Variable declared but not assigned
- `null`: Intentional absence of value (explicitly set)

```javascript
let x;              // undefined
let y = null;       // null
```

This covers the fundamental concepts of JavaScript! Practice these concepts with real examples to solidify your understanding.
