# JavaScript Interview Questions (Q1-Q3): Core Fundamentals

## Q1: Explain closures in JavaScript with a banking scenario example

**Answer:**

A closure is a function that has access to variables in its outer (enclosing) function's scope, even after the outer function has returned. This is fundamental for data encapsulation and privacy.

**Banking Account Example:**

```javascript
// Banking Account with Private Balance using Closures
function createBankAccount(accountHolder, initialBalance) {
    // Private variables - only accessible through closures
    let balance = initialBalance;
    let transactionHistory = [];
    const accountNumber = `ENBD${Math.random().toString(36).substr(2, 9).toUpperCase()}`;
    
    // Private helper function
    function recordTransaction(type, amount, description) {
        transactionHistory.push({
            id: transactionHistory.length + 1,
            type,
            amount,
            description,
            balance: balance,
            timestamp: new Date().toISOString()
        });
    }
    
    // Public methods (closures) that access private variables
    return {
        getAccountNumber: function() {
            return accountNumber;
        },
        
        getAccountHolder: function() {
            return accountHolder;
        },
        
        getBalance: function() {
            console.log(`Account ${accountNumber}: AED ${balance.toFixed(2)}`);
            return balance;
        },
        
        deposit: function(amount) {
            if (amount <= 0) {
                throw new Error('Deposit amount must be positive');
            }
            
            balance += amount;
            recordTransaction('CREDIT', amount, 'Cash Deposit');
            console.log(`Deposited AED ${amount}. New balance: AED ${balance.toFixed(2)}`);
            return balance;
        },
        
        withdraw: function(amount) {
            if (amount <= 0) {
                throw new Error('Withdrawal amount must be positive');
            }
            
            if (amount > balance) {
                throw new Error('Insufficient funds');
            }
            
            balance -= amount;
            recordTransaction('DEBIT', amount, 'Cash Withdrawal');
            console.log(`Withdrew AED ${amount}. New balance: AED ${balance.toFixed(2)}`);
            return balance;
        },
        
        transfer: function(targetAccount, amount) {
            if (amount <= 0) {
                throw new Error('Transfer amount must be positive');
            }
            
            if (amount > balance) {
                throw new Error('Insufficient funds for transfer');
            }
            
            balance -= amount;
            recordTransaction('DEBIT', amount, `Transfer to ${targetAccount.getAccountNumber()}`);
            targetAccount.deposit(amount);
            
            console.log(`Transferred AED ${amount} to ${targetAccount.getAccountNumber()}`);
            return balance;
        },
        
        getTransactionHistory: function() {
            // Return a copy to prevent external modification
            return transactionHistory.map(t => ({...t}));
        },
        
        getMiniStatement: function(limit = 5) {
            const recent = transactionHistory.slice(-limit);
            console.log(`\n=== Mini Statement for ${accountNumber} ===`);
            console.log(`Account Holder: ${accountHolder}`);
            console.log(`Current Balance: AED ${balance.toFixed(2)}\n`);
            
            recent.forEach(txn => {
                const sign = txn.type === 'CREDIT' ? '+' : '-';
                console.log(`${txn.timestamp} | ${txn.type} | ${sign}${txn.amount} | ${txn.description} | Balance: ${txn.balance}`);
            });
            
            return recent;
        }
    };
}

// Usage Example
const ahmedAccount = createBankAccount('Ahmed Al Mansouri', 10000);
const fatimaAccount = createBankAccount('Fatima Hassan', 5000);

console.log(ahmedAccount.getAccountNumber()); // ENBD...
console.log(ahmedAccount.getBalance()); // 10000

ahmedAccount.deposit(2500); // Deposited AED 2500. New balance: AED 12500
ahmedAccount.withdraw(1000); // Withdrew AED 1000. New balance: AED 11500
ahmedAccount.transfer(fatimaAccount, 3000); // Transferred AED 3000

ahmedAccount.getMiniStatement(3);
// Mini Statement shows last 3 transactions

// Attempting to access private variables directly fails
console.log(ahmedAccount.balance); // undefined - private variable protected
console.log(ahmedAccount.transactionHistory); // undefined - private variable protected

// This demonstrates closure: inner functions maintain access to outer scope
// even after createBankAccount has returned
```

**Advanced Closure Example - Interest Calculator:**

```javascript
// Closure for compound interest calculation
function createInterestCalculator(accountType) {
    const interestRates = {
        'savings': 0.025,      // 2.5% annual
        'fixed_deposit_1y': 0.04,  // 4% for 1 year
        'fixed_deposit_3y': 0.055, // 5.5% for 3 years
        'current': 0.001       // 0.1% for current accounts
    };
    
    const rate = interestRates[accountType] || 0;
    
    return {
        calculateSimpleInterest: function(principal, years) {
            const interest = principal * rate * years;
            return {
                principal,
                rate: rate * 100,
                years,
                interest,
                totalAmount: principal + interest
            };
        },
        
        calculateCompoundInterest: function(principal, years, frequency = 12) {
            // Compound interest: A = P(1 + r/n)^(nt)
            const n = frequency; // monthly = 12
            const amount = principal * Math.pow((1 + rate / n), n * years);
            const interest = amount - principal;
            
            return {
                principal,
                rate: rate * 100,
                years,
                frequency: `${frequency}x per year`,
                interest,
                totalAmount: amount
            };
        },
        
        getMonthlyInterest: function(principal) {
            const monthlyRate = rate / 12;
            const monthlyInterest = principal * monthlyRate;
            
            return {
                monthlyRate: monthlyRate * 100,
                monthlyInterest,
                annualProjection: monthlyInterest * 12
            };
        }
    };
}

// Usage
const savingsCalculator = createInterestCalculator('savings');
const fixedDepositCalculator = createInterestCalculator('fixed_deposit_3y');

console.log(savingsCalculator.calculateCompoundInterest(50000, 5));
// Output: 
// {
//   principal: 50000,
//   rate: 2.5,
//   years: 5,
//   frequency: '12x per year',
//   interest: 6630.45,
//   totalAmount: 56630.45
// }

console.log(fixedDepositCalculator.calculateSimpleInterest(100000, 3));
// Output:
// {
//   principal: 100000,
//   rate: 5.5,
//   years: 3,
//   interest: 16500,
//   totalAmount: 116500
// }
```

---

## Q2: Explain the difference between `var`, `let`, and `const` with practical banking examples

**Answer:**

The main differences are in scope, hoisting, and reassignment capabilities.

**Comparison Table:**

| Feature | var | let | const |
|---------|-----|-----|-------|
| **Scope** | Function-scoped | Block-scoped | Block-scoped |
| **Hoisting** | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| **Reassignment** | Yes | Yes | No |
| **Redeclaration** | Yes | No | No |
| **Best Practice** | Avoid | Use for variables | Use for constants |

**Banking Transaction Processing Example:**

```javascript
// Transaction Processing System
class TransactionProcessor {
    processTransactions(transactions) {
        // VAR - Function scoped (problematic)
        console.log('\n=== VAR Example (Function Scope - Problematic) ===');
        for (var i = 0; i < transactions.length; i++) {
            var status = 'processing'; // var is function-scoped
            
            setTimeout(function() {
                // BUG: 'i' will be transactions.length for all callbacks
                // because var is function-scoped and loop completes before timeouts
                console.log(`Transaction ${i}: ${status}`); // Wrong index!
            }, 100);
        }
        
        console.log(`After loop, i = ${i}`); // i is accessible - function scope
        console.log(`After loop, status = ${status}`); // status is accessible
        
        // LET - Block scoped (correct)
        console.log('\n=== LET Example (Block Scope - Correct) ===');
        for (let j = 0; j < transactions.length; j++) {
            let txnStatus = 'processing'; // let is block-scoped
            
            setTimeout(function() {
                // Correct: each iteration has its own 'j'
                console.log(`Transaction ${j}: ${transactions[j].amount} AED`);
            }, 200);
        }
        
        // console.log(j); // ReferenceError: j is not defined
        // console.log(txnStatus); // ReferenceError: txnStatus is not defined
    }
}

// Practical Example: Account Management
function processAccountOperations() {
    // CONST - Immutable reference (best practice for objects)
    const accountConfig = {
        minBalance: 1000,
        maxDailyWithdrawal: 10000,
        currency: 'AED'
    };
    
    // accountConfig = {}; // TypeError: Assignment to constant variable
    
    // But we can modify properties (object is mutable)
    accountConfig.minBalance = 500; // This works!
    console.log(accountConfig); // { minBalance: 500, maxDailyWithdrawal: 10000, currency: 'AED' }
    
    // To make object truly immutable
    const immutableConfig = Object.freeze({
        bankName: 'Emirates NBD',
        swiftCode: 'EBILAEAD',
        branches: ['Dubai', 'Abu Dhabi', 'Sharjah']
    });
    
    // immutableConfig.bankName = 'Other Bank'; // Fails silently in non-strict mode
    // immutableConfig.branches.push('Ajman'); // This still works - shallow freeze
    
    // Deep freeze for nested objects
    const deepFreeze = (obj) => {
        Object.freeze(obj);
        Object.keys(obj).forEach(key => {
            if (typeof obj[key] === 'object' && obj[key] !== null) {
                deepFreeze(obj[key]);
            }
        });
        return obj;
    };
    
    const trulyImmutableConfig = deepFreeze({
        bank: {
            name: 'Emirates NBD',
            code: 'ENBD'
        },
        limits: {
            daily: 50000,
            monthly: 200000
        }
    });
}

// Real Banking Scenario: Daily Transaction Limit Checker
class DailyLimitChecker {
    constructor(dailyLimit) {
        // Use const for values that shouldn't change
        const BANK_NAME = 'Emirates NBD';
        const MAX_ATTEMPTS = 3;
        
        // Use let for values that will change
        let transactionsToday = 0;
        let totalAmountToday = 0;
        let failedAttempts = 0;
        
        this.checkTransaction = function(amount) {
            // Block scope with let
            if (amount <= 0) {
                throw new Error('Invalid transaction amount');
            }
            
            // Increment attempts
            failedAttempts++;
            
            if (failedAttempts > MAX_ATTEMPTS) {
                return {
                    approved: false,
                    reason: 'Too many failed attempts. Card blocked.',
                    bank: BANK_NAME
                };
            }
            
            // Check if transaction exceeds daily limit
            if (totalAmountToday + amount > dailyLimit) {
                return {
                    approved: false,
                    reason: `Daily limit exceeded. Limit: AED ${dailyLimit}, Used: AED ${totalAmountToday}`,
                    remainingLimit: dailyLimit - totalAmountToday
                };
            }
            
            // Approve transaction
            transactionsToday++;
            totalAmountToday += amount;
            failedAttempts = 0; // Reset on success
            
            return {
                approved: true,
                transactionNumber: transactionsToday,
                totalToday: totalAmountToday,
                remainingLimit: dailyLimit - totalAmountToday,
                bank: BANK_NAME
            };
        };
        
        this.getDailySummary = function() {
            return {
                bank: BANK_NAME,
                dailyLimit: dailyLimit,
                transactionsCount: transactionsToday,
                totalSpent: totalAmountToday,
                remaining: dailyLimit - totalAmountToday
            };
        };
        
        this.resetDaily = function() {
            transactionsToday = 0;
            totalAmountToday = 0;
            failedAttempts = 0;
            console.log('Daily limits reset');
        };
    }
}

// Usage
const checker = new DailyLimitChecker(10000);

console.log(checker.checkTransaction(3000));
// { approved: true, transactionNumber: 1, totalToday: 3000, remainingLimit: 7000 }

console.log(checker.checkTransaction(8000));
// { approved: false, reason: 'Daily limit exceeded...', remainingLimit: 7000 }

console.log(checker.checkTransaction(5000));
// { approved: true, transactionNumber: 2, totalToday: 8000, remainingLimit: 2000 }

console.log(checker.getDailySummary());
// { bank: 'Emirates NBD', dailyLimit: 10000, transactionsCount: 2, totalSpent: 8000, remaining: 2000 }
```

**Hoisting Demonstration:**

```javascript
// Hoisting behavior comparison
function demonstrateHoisting() {
    console.log('\n=== Hoisting Behavior ===');
    
    // VAR - hoisted with undefined
    console.log(accountType); // undefined (not ReferenceError)
    var accountType = 'savings';
    console.log(accountType); // 'savings'
    
    // LET - Temporal Dead Zone (TDZ)
    try {
        console.log(customerName); // ReferenceError
        let customerName = 'Ahmed';
    } catch (error) {
        console.log('Error:', error.message);
    }
    
    // CONST - Temporal Dead Zone (TDZ)
    try {
        console.log(BANK_CODE); // ReferenceError
        const BANK_CODE = 'ENBD';
    } catch (error) {
        console.log('Error:', error.message);
    }
    
    // Function hoisting (works)
    console.log(calculateInterest(10000, 0.05)); // 500
    
    function calculateInterest(principal, rate) {
        return principal * rate;
    }
}

demonstrateHoisting();
```

---

## Q3: Explain Promises and async/await with a banking API example

**Answer:**

Promises represent the eventual completion (or failure) of an asynchronous operation. Async/await is syntactic sugar for working with Promises more elegantly.

**Banking API Service Example:**

```javascript
// Banking API Service with Promises and Async/Await
class BankingAPIService {
    constructor(baseURL) {
        this.baseURL = baseURL || 'https://api.emiratesnbd.com/v1';
        this.timeout = 5000;
    }
    
    // Promise-based fetch with timeout
    fetchWithTimeout(url, options = {}) {
        return Promise.race([
            fetch(url, options),
            new Promise((_, reject) => 
                setTimeout(() => reject(new Error('Request timeout')), this.timeout)
            )
        ]);
    }
    
    // Promise-based method: Get account balance
    getAccountBalance(accountId) {
        return new Promise((resolve, reject) => {
            console.log(`Fetching balance for account ${accountId}...`);
            
            this.fetchWithTimeout(`${this.baseURL}/accounts/${accountId}/balance`, {
                method: 'GET',
                headers: {
                    'Authorization': `Bearer ${this.getToken()}`,
                    'Content-Type': 'application/json'
                }
            })
            .then(response => {
                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
                }
                return response.json();
            })
            .then(data => {
                resolve({
                    accountId: data.accountId,
                    balance: data.balance,
                    currency: data.currency,
                    timestamp: new Date().toISOString()
                });
            })
            .catch(error => {
                reject(new Error(`Failed to fetch balance: ${error.message}`));
            });
        });
    }
    
    // Async/await method: Transfer funds
    async transferFunds(fromAccount, toAccount, amount) {
        try {
            console.log(`Initiating transfer of AED ${amount}...`);
            
            // Step 1: Validate accounts
            const [sourceAccount, destinationAccount] = await Promise.all([
                this.validateAccount(fromAccount),
                this.validateAccount(toAccount)
            ]);
            
            console.log('Accounts validated');
            
            // Step 2: Check balance
            const balance = await this.getAccountBalance(fromAccount);
            
            if (balance.balance < amount) {
                throw new Error(`Insufficient funds. Available: AED ${balance.balance}`);
            }
            
            console.log('Balance check passed');
            
            // Step 3: Create transaction
            const transaction = await this.createTransaction({
                type: 'TRANSFER',
                fromAccount,
                toAccount,
                amount,
                currency: 'AED'
            });
            
            console.log(`Transaction created: ${transaction.transactionId}`);
            
            // Step 4: Process transfer
            const result = await this.processTransfer(transaction.transactionId);
            
            // Step 5: Send notifications (don't wait)
            this.sendNotification(fromAccount, 'DEBIT', amount).catch(console.error);
            this.sendNotification(toAccount, 'CREDIT', amount).catch(console.error);
            
            return {
                success: true,
                transactionId: result.transactionId,
                fromAccount,
                toAccount,
                amount,
                status: result.status,
                timestamp: result.timestamp
            };
            
        } catch (error) {
            console.error('Transfer failed:', error.message);
            
            // Log error to monitoring system
            await this.logError(error, { fromAccount, toAccount, amount });
            
            return {
                success: false,
                error: error.message,
                timestamp: new Date().toISOString()
            };
        }
    }
    
    // Async helper: Validate account
    async validateAccount(accountId) {
        const response = await this.fetchWithTimeout(
            `${this.baseURL}/accounts/${accountId}/validate`,
            {
                method: 'GET',
                headers: {
                    'Authorization': `Bearer ${this.getToken()}`,
                    'Content-Type': 'application/json'
                }
            }
        );
        
        if (!response.ok) {
            throw new Error(`Invalid account: ${accountId}`);
        }
        
        return response.json();
    }
    
    // Async helper: Create transaction
    async createTransaction(transactionData) {
        const response = await this.fetchWithTimeout(
            `${this.baseURL}/transactions`,
            {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${this.getToken()}`,
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(transactionData)
            }
        );
        
        if (!response.ok) {
            throw new Error('Failed to create transaction');
        }
        
        return response.json();
    }
    
    // Async helper: Process transfer
    async processTransfer(transactionId) {
        const response = await this.fetchWithTimeout(
            `${this.baseURL}/transactions/${transactionId}/process`,
            {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${this.getToken()}`,
                    'Content-Type': 'application/json'
                }
            }
        );
        
        if (!response.ok) {
            throw new Error('Failed to process transfer');
        }
        
        return response.json();
    }
    
    // Async notification (fire and forget)
    async sendNotification(accountId, type, amount) {
        try {
            await this.fetchWithTimeout(
                `${this.baseURL}/notifications`,
                {
                    method: 'POST',
                    headers: {
                        'Authorization': `Bearer ${this.getToken()}`,
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({ accountId, type, amount })
                }
            );
            console.log(`Notification sent to account ${accountId}`);
        } catch (error) {
            console.error('Notification failed:', error.message);
        }
    }
    
    // Async error logging
    async logError(error, context) {
        try {
            await this.fetchWithTimeout(
                `${this.baseURL}/errors/log`,
                {
                    method: 'POST',
                    headers: {
                        'Authorization': `Bearer ${this.getToken()}`,
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({
                        error: error.message,
                        stack: error.stack,
                        context,
                        timestamp: new Date().toISOString()
                    })
                }
            );
        } catch (logError) {
            console.error('Failed to log error:', logError.message);
        }
    }
    
    getToken() {
        return 'mock-jwt-token-12345';
    }
}

// Usage Example
async function performBankingOperations() {
    const bankingAPI = new BankingAPIService();
    
    try {
        console.log('\n=== Banking Operations Demo ===\n');
        
        // 1. Get balance (Promise-based)
        const balance = await bankingAPI.getAccountBalance('ACC123456');
        console.log('Balance:', balance);
        
        // 2. Transfer funds (async/await)
        const transferResult = await bankingAPI.transferFunds(
            'ACC123456',
            'ACC789012',
            5000
        );
        
        console.log('\nTransfer Result:', transferResult);
        
    } catch (error) {
        console.error('Operation failed:', error.message);
    }
}

// Promise chaining example
function promiseChainingExample() {
    const bankingAPI = new BankingAPIService();
    
    console.log('\n=== Promise Chaining Example ===\n');
    
    bankingAPI.getAccountBalance('ACC123456')
        .then(balance => {
            console.log('Current balance:', balance);
            return bankingAPI.transferFunds('ACC123456', 'ACC789012', 1000);
        })
        .then(transferResult => {
            console.log('Transfer completed:', transferResult);
            return bankingAPI.getAccountBalance('ACC123456');
        })
        .then(newBalance => {
            console.log('New balance:', newBalance);
        })
        .catch(error => {
            console.error('Error in chain:', error.message);
        })
        .finally(() => {
            console.log('Operation chain completed');
        });
}

// Parallel operations with Promise.all
async function batchAccountOperations(accountIds) {
    const bankingAPI = new BankingAPIService();
    
    console.log('\n=== Batch Operations with Promise.all ===\n');
    
    try {
        // Fetch all balances in parallel
        const balancePromises = accountIds.map(id => 
            bankingAPI.getAccountBalance(id)
        );
        
        const balances = await Promise.all(balancePromises);
        
        const totalBalance = balances.reduce((sum, acc) => sum + acc.balance, 0);
        
        console.log('All Balances:', balances);
        console.log('Total Balance:', totalBalance);
        
        return { balances, totalBalance };
        
    } catch (error) {
        console.error('Batch operation failed:', error.message);
        throw error;
    }
}

// Promise.allSettled for error tolerance
async function batchOperationsWithErrorHandling(operations) {
    console.log('\n=== Promise.allSettled for Error Tolerance ===\n');
    
    const results = await Promise.allSettled(operations);
    
    const successful = results.filter(r => r.status === 'fulfilled');
    const failed = results.filter(r => r.status === 'rejected');
    
    console.log(`Successful: ${successful.length}, Failed: ${failed.length}`);
    
    return {
        successful: successful.map(r => r.value),
        failed: failed.map(r => r.reason)
    };
}

// Execute examples
performBankingOperations();
```

---
