# Node.js Interview Question: Advanced Error Handling
## Question 10: How to Create Custom Error Classes and Handle Complex Error Scenarios?

---

## 📋 Summary of What Will Be Covered

In this comprehensive guide, you'll learn:

1. **Custom Error Classes** - Creating domain-specific error types
2. **Error Hierarchies** - Organizing errors by category
3. **Error Propagation** - How errors flow through async operations
4. **Error Context** - Adding metadata to errors
5. **Error Recovery Strategies** - Retry logic, fallbacks, circuit breakers
6. **Operational vs Programmer Errors** - Different handling approaches
7. **Banking Examples**: Robust payment error handling, transaction rollback, fraud detection

**Why This Matters in Banking**:
- Different errors require different responses (retry, alert, rollback)
- Error context helps debugging production issues
- Payment failures need specific error codes for customer support
- Compliance requires detailed error logging
- System resilience requires sophisticated error handling
- Customer trust depends on graceful error recovery

---

## 🎯 Custom Error Classes: The Foundation

### Why Custom Errors?

Standard JavaScript `Error` provides minimal information. Custom errors give you:
- **Specific error types** for different scenarios
- **Additional context** (error codes, HTTP status codes)
- **Structured error data** for logging and monitoring
- **Better error handling** based on error type

### Basic Custom Error

```javascript
class BankingError extends Error {
  constructor(message, code, statusCode = 500) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    this.timestamp = new Date().toISOString();
    
    // Capture stack trace (excludes constructor from stack)
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      statusCode: this.statusCode,
      timestamp: this.timestamp
    };
  }
}

// Usage
throw new BankingError(
  'Transaction failed',
  'TXN_FAILED',
  400
);
```

---

## 📊 Visual: Error Hierarchy

```
                    BankingError (Base)
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ValidationError    TransactionError    AccountError
        │                  │                  │
        ├─ InvalidAmount   ├─ InsufficientFunds    ├─ AccountNotFound
        ├─ MissingField    ├─ TransactionLimitExceeded    ├─ AccountLocked
        └─ InvalidFormat   ├─ DuplicateTransaction    └─ AccountClosed
                          └─ PaymentGatewayError
```

---

## 🏦 Example 1: Complete Error Hierarchy for Banking

This example demonstrates a comprehensive error class system for banking operations.

**File: `banking-error-system.js`**

```javascript
/**
 * Advanced Banking Error Handling System
 * Demonstrates: Custom error classes, error hierarchy, error context
 */

console.log('🏦 Banking Error Handling System\n');
console.log('='.repeat(70));

// ============================================
// Base Banking Error Class
// ============================================

class BankingError extends Error {
  constructor(message, code, statusCode = 500, details = {}) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    this.details = details;
    this.timestamp = new Date().toISOString();
    this.isOperational = true;  // Operational error (not a programmer error)
    
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      error: {
        name: this.name,
        message: this.message,
        code: this.code,
        statusCode: this.statusCode,
        details: this.details,
        timestamp: this.timestamp
      }
    };
  }
  
  toString() {
    return `[${this.code}] ${this.name}: ${this.message}`;
  }
}

// ============================================
// Validation Errors (400 series)
// ============================================

class ValidationError extends BankingError {
  constructor(message, details = {}) {
    super(message, 'VALIDATION_ERROR', 400, details);
  }
}

class InvalidAmountError extends ValidationError {
  constructor(amount, reason = 'Amount must be positive') {
    super(reason, {
      amount,
      field: 'amount',
      constraint: 'positive_number'
    });
    this.code = 'INVALID_AMOUNT';
  }
}

class MissingFieldError extends ValidationError {
  constructor(field) {
    super(`Required field missing: ${field}`, {
      field,
      constraint: 'required'
    });
    this.code = 'MISSING_FIELD';
  }
}

class InvalidFormatError extends ValidationError {
  constructor(field, expectedFormat) {
    super(`Invalid format for ${field}`, {
      field,
      expectedFormat,
      constraint: 'format'
    });
    this.code = 'INVALID_FORMAT';
  }
}

// ============================================
// Account Errors (404/403 series)
// ============================================

class AccountError extends BankingError {
  constructor(message, code, statusCode, accountId) {
    super(message, code, statusCode, { accountId });
  }
}

class AccountNotFoundError extends AccountError {
  constructor(accountId) {
    super(
      `Account not found: ${accountId}`,
      'ACCOUNT_NOT_FOUND',
      404,
      accountId
    );
  }
}

class AccountLockedError extends AccountError {
  constructor(accountId, reason = 'Security hold') {
    super(
      `Account locked: ${accountId}`,
      'ACCOUNT_LOCKED',
      403,
      accountId
    );
    this.details.reason = reason;
  }
}

class AccountClosedError extends AccountError {
  constructor(accountId, closedDate) {
    super(
      `Account closed: ${accountId}`,
      'ACCOUNT_CLOSED',
      403,
      accountId
    );
    this.details.closedDate = closedDate;
  }
}

// ============================================
// Transaction Errors (400/402/500 series)
// ============================================

class TransactionError extends BankingError {
  constructor(message, code, statusCode, transactionDetails = {}) {
    super(message, code, statusCode, transactionDetails);
  }
}

class InsufficientFundsError extends TransactionError {
  constructor(accountId, available, requested) {
    super(
      `Insufficient funds in account ${accountId}`,
      'INSUFFICIENT_FUNDS',
      402,
      {
        accountId,
        available,
        requested,
        shortfall: requested - available
      }
    );
  }
}

class TransactionLimitExceededError extends TransactionError {
  constructor(accountId, limit, amount, limitType = 'daily') {
    super(
      `Transaction limit exceeded: ${limitType} limit is ${limit}`,
      'TRANSACTION_LIMIT_EXCEEDED',
      400,
      {
        accountId,
        limit,
        amount,
        limitType,
        excess: amount - limit
      }
    );
  }
}

class DuplicateTransactionError extends TransactionError {
  constructor(transactionId, originalTransactionId) {
    super(
      `Duplicate transaction detected: ${transactionId}`,
      'DUPLICATE_TRANSACTION',
      409,
      {
        transactionId,
        originalTransactionId
      }
    );
  }
}

class PaymentGatewayError extends TransactionError {
  constructor(gatewayName, gatewayCode, gatewayMessage) {
    super(
      `Payment gateway error: ${gatewayMessage}`,
      'PAYMENT_GATEWAY_ERROR',
      502,
      {
        gateway: gatewayName,
        gatewayCode,
        gatewayMessage
      }
    );
    this.isRetryable = true;
  }
}

// ============================================
// Fraud Detection Errors
// ============================================

class FraudError extends BankingError {
  constructor(message, details = {}) {
    super(message, 'FRAUD_DETECTED', 403, details);
    this.requiresInvestigation = true;
  }
}

class SuspiciousActivityError extends FraudError {
  constructor(accountId, reason, riskScore) {
    super(
      `Suspicious activity detected on account ${accountId}`,
      {
        accountId,
        reason,
        riskScore,
        action: 'transaction_blocked'
      }
    );
    this.code = 'SUSPICIOUS_ACTIVITY';
  }
}

// ============================================
// Network/System Errors (500 series)
// ============================================

class SystemError extends BankingError {
  constructor(message, code, details = {}) {
    super(message, code, 500, details);
    this.isRetryable = true;
  }
}

class DatabaseError extends SystemError {
  constructor(operation, originalError) {
    super(
      `Database operation failed: ${operation}`,
      'DATABASE_ERROR',
      {
        operation,
        originalError: originalError.message
      }
    );
  }
}

class ServiceUnavailableError extends SystemError {
  constructor(serviceName, retryAfter = 60) {
    super(
      `Service unavailable: ${serviceName}`,
      'SERVICE_UNAVAILABLE',
      {
        serviceName,
        retryAfter
      }
    );
    this.statusCode = 503;
    this.details.retryAfter = retryAfter;
  }
}

// ============================================
// Error Handler Utility
// ============================================

class ErrorHandler {
  static isOperationalError(error) {
    if (error instanceof BankingError) {
      return error.isOperational;
    }
    return false;
  }
  
  static isRetryableError(error) {
    if (error instanceof BankingError) {
      return error.isRetryable === true;
    }
    return false;
  }
  
  static getErrorResponse(error) {
    if (error instanceof BankingError) {
      return {
        statusCode: error.statusCode,
        body: error.toJSON()
      };
    }
    
    // Unknown error - don't expose details
    return {
      statusCode: 500,
      body: {
        error: {
          name: 'InternalServerError',
          message: 'An unexpected error occurred',
          code: 'INTERNAL_ERROR',
          timestamp: new Date().toISOString()
        }
      }
    };
  }
  
  static logError(error, context = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      error: {
        name: error.name,
        message: error.message,
        code: error.code || 'UNKNOWN',
        stack: error.stack
      },
      context
    };
    
    if (error instanceof BankingError) {
      logEntry.error.details = error.details;
      logEntry.error.statusCode = error.statusCode;
      logEntry.error.isOperational = error.isOperational;
    }
    
    console.error(JSON.stringify(logEntry, null, 2));
  }
}

// ============================================
// Demo: Error Scenarios
// ============================================

function demonstrateErrors() {
  console.log('\n📋 DEMO: Banking Error Scenarios\n');
  
  const scenarios = [
    {
      name: 'Invalid Amount',
      error: new InvalidAmountError(-100, 'Amount must be positive'),
      handler: () => 'Reject transaction'
    },
    {
      name: 'Missing Field',
      error: new MissingFieldError('toAccount'),
      handler: () => 'Request missing information'
    },
    {
      name: 'Account Not Found',
      error: new AccountNotFoundError('ACC-99999'),
      handler: () => 'Return 404 to user'
    },
    {
      name: 'Account Locked',
      error: new AccountLockedError('ACC-12345', 'Multiple failed login attempts'),
      handler: () => 'Direct user to customer support'
    },
    {
      name: 'Insufficient Funds',
      error: new InsufficientFundsError('ACC-12345', 500, 750),
      handler: () => 'Reject transaction, suggest funding options'
    },
    {
      name: 'Transaction Limit Exceeded',
      error: new TransactionLimitExceededError('ACC-12345', 5000, 7500, 'daily'),
      handler: () => 'Reject transaction, show limit details'
    },
    {
      name: 'Duplicate Transaction',
      error: new DuplicateTransactionError('TXN-789', 'TXN-456'),
      handler: () => 'Return original transaction result'
    },
    {
      name: 'Payment Gateway Error',
      error: new PaymentGatewayError('Stripe', 'card_declined', 'Your card was declined'),
      handler: () => 'Retry with different payment method'
    },
    {
      name: 'Suspicious Activity',
      error: new SuspiciousActivityError('ACC-12345', 'Multiple high-value transactions', 0.85),
      handler: () => 'Block transaction, alert fraud team'
    },
    {
      name: 'Database Error',
      error: new DatabaseError('INSERT transaction', new Error('Connection timeout')),
      handler: () => 'Retry operation, alert ops team'
    }
  ];
  
  scenarios.forEach((scenario, index) => {
    console.log(`${index + 1}. ${scenario.name}`);
    console.log('   Error:', scenario.error.toString());
    console.log('   Code:', scenario.error.code);
    console.log('   Status:', scenario.error.statusCode);
    console.log('   Details:', JSON.stringify(scenario.error.details, null, 2).split('\n').map(l => '   ' + l).join('\n'));
    console.log('   Action:', scenario.handler());
    console.log('   Retryable:', ErrorHandler.isRetryableError(scenario.error) ? 'Yes' : 'No');
    console.log();
  });
}

// ============================================
// Demo: Error Response Generation
// ============================================

function demonstrateErrorResponses() {
  console.log('\n📤 DEMO: Error Response Generation\n');
  
  const errors = [
    new InvalidAmountError(-50),
    new AccountNotFoundError('ACC-99999'),
    new InsufficientFundsError('ACC-12345', 100, 500),
    new Error('Unexpected programmer error')  // Non-banking error
  ];
  
  errors.forEach((error, index) => {
    console.log(`${index + 1}. ${error.name || error.constructor.name}`);
    const response = ErrorHandler.getErrorResponse(error);
    console.log('   HTTP Status:', response.statusCode);
    console.log('   Response Body:');
    console.log(JSON.stringify(response.body, null, 2).split('\n').map(l => '   ' + l).join('\n'));
    console.log();
  });
}

// Run demos
demonstrateErrors();
demonstrateErrorResponses();

console.log('='.repeat(70));
console.log('✅ Error System Demonstration Complete!');
console.log('='.repeat(70) + '\n');
```

**To run this example:**

```bash
node banking-error-system.js
```

**Expected Output:**

```
🏦 Banking Error Handling System

======================================================================

📋 DEMO: Banking Error Scenarios

1. Invalid Amount
   Error: [INVALID_AMOUNT] InvalidAmountError: Amount must be positive
   Code: INVALID_AMOUNT
   Status: 400
   Details: {
     "amount": -100,
     "field": "amount",
     "constraint": "positive_number"
   }
   Action: Reject transaction
   Retryable: No

2. Missing Field
   Error: [MISSING_FIELD] MissingFieldError: Required field missing: toAccount
   Code: MISSING_FIELD
   Status: 400
   Details: {
     "field": "toAccount",
     "constraint": "required"
   }
   Action: Request missing information
   Retryable: No

3. Account Not Found
   Error: [ACCOUNT_NOT_FOUND] AccountNotFoundError: Account not found: ACC-99999
   Code: ACCOUNT_NOT_FOUND
   Status: 404
   Details: {
     "accountId": "ACC-99999"
   }
   Action: Return 404 to user
   Retryable: No

4. Account Locked
   Error: [ACCOUNT_LOCKED] AccountLockedError: Account locked: ACC-12345
   Code: ACCOUNT_LOCKED
   Status: 403
   Details: {
     "accountId": "ACC-12345",
     "reason": "Multiple failed login attempts"
   }
   Action: Direct user to customer support
   Retryable: No

5. Insufficient Funds
   Error: [INSUFFICIENT_FUNDS] InsufficientFundsError: Insufficient funds in account ACC-12345
   Code: INSUFFICIENT_FUNDS
   Status: 402
   Details: {
     "accountId": "ACC-12345",
     "available": 500,
     "requested": 750,
     "shortfall": 250
   }
   Action: Reject transaction, suggest funding options
   Retryable: No

6. Transaction Limit Exceeded
   Error: [TRANSACTION_LIMIT_EXCEEDED] TransactionLimitExceededError: Transaction limit exceeded: daily limit is 5000
   Code: TRANSACTION_LIMIT_EXCEEDED
   Status: 400
   Details: {
     "accountId": "ACC-12345",
     "limit": 5000,
     "amount": 7500,
     "limitType": "daily",
     "excess": 2500
   }
   Action: Reject transaction, show limit details
   Retryable: No

7. Duplicate Transaction
   Error: [DUPLICATE_TRANSACTION] DuplicateTransactionError: Duplicate transaction detected: TXN-789
   Code: DUPLICATE_TRANSACTION
   Status: 409
   Details: {
     "transactionId": "TXN-789",
     "originalTransactionId": "TXN-456"
   }
   Action: Return original transaction result
   Retryable: No

8. Payment Gateway Error
   Error: [PAYMENT_GATEWAY_ERROR] PaymentGatewayError: Payment gateway error: Your card was declined
   Code: PAYMENT_GATEWAY_ERROR
   Status: 502
   Details: {
     "gateway": "Stripe",
     "gatewayCode": "card_declined",
     "gatewayMessage": "Your card was declined"
   }
   Action: Retry with different payment method
   Retryable: Yes

9. Suspicious Activity
   Error: [SUSPICIOUS_ACTIVITY] SuspiciousActivityError: Suspicious activity detected on account ACC-12345
   Code: SUSPICIOUS_ACTIVITY
   Status: 403
   Details: {
     "accountId": "ACC-12345",
     "reason": "Multiple high-value transactions",
     "riskScore": 0.85,
     "action": "transaction_blocked"
   }
   Action: Block transaction, alert fraud team
   Retryable: No

10. Database Error
    Error: [DATABASE_ERROR] DatabaseError: Database operation failed: INSERT transaction
    Code: DATABASE_ERROR
    Status: 500
    Details: {
      "operation": "INSERT transaction",
      "originalError": "Connection timeout"
    }
    Action: Retry operation, alert ops team
    Retryable: Yes

📤 DEMO: Error Response Generation

1. InvalidAmountError
   HTTP Status: 400
   Response Body:
   {
     "error": {
       "name": "InvalidAmountError",
       "message": "Amount must be positive",
       "code": "INVALID_AMOUNT",
       "statusCode": 400,
       "details": {
         "amount": -50,
         "field": "amount",
         "constraint": "positive_number"
       },
       "timestamp": "2025-11-14T10:30:00.000Z"
     }
   }

2. AccountNotFoundError
   HTTP Status: 404
   Response Body:
   {
     "error": {
       "name": "AccountNotFoundError",
       "message": "Account not found: ACC-99999",
       "code": "ACCOUNT_NOT_FOUND",
       "statusCode": 404,
       "details": {
         "accountId": "ACC-99999"
       },
       "timestamp": "2025-11-14T10:30:00.000Z"
     }
   }

3. InsufficientFundsError
   HTTP Status: 402
   Response Body:
   {
     "error": {
       "name": "InsufficientFundsError",
       "message": "Insufficient funds in account ACC-12345",
       "code": "INSUFFICIENT_FUNDS",
       "statusCode": 402,
       "details": {
         "accountId": "ACC-12345",
         "available": 100,
         "requested": 500,
         "shortfall": 400
       },
       "timestamp": "2025-11-14T10:30:00.000Z"
     }
   }

4. Error
   HTTP Status: 500
   Response Body:
   {
     "error": {
       "name": "InternalServerError",
       "message": "An unexpected error occurred",
       "code": "INTERNAL_ERROR",
       "timestamp": "2025-11-14T10:30:00.000Z"
     }
   }

======================================================================
✅ Error System Demonstration Complete!
======================================================================
```

**Key Learnings from Example 1**:

1. **Error Hierarchy**: Organize errors by category (Validation, Account, Transaction)
2. **Error Codes**: Consistent error codes for client handling
3. **HTTP Status Codes**: Map errors to appropriate HTTP statuses
4. **Error Context**: Include relevant details for debugging
5. **Operational vs Programmer Errors**: Flag operational errors separately
6. **Retryable Errors**: Mark errors that can be retried
7. **Error Responses**: Structured error responses for APIs
8. **Security**: Don't expose internal errors to clients

---

## 🏦 Example 2: Error Propagation and Async Error Handling

This example demonstrates how errors propagate through async operations and how to handle them.

**File: `error-propagation-system.js`**

```javascript
import crypto from 'crypto';

/**
 * Error Propagation in Async Operations
 * Demonstrates: Error propagation, async error handling, error context preservation
 */

console.log('🔄 Error Propagation System\n');
console.log('='.repeat(70));

// ============================================
// Custom Errors (reusing from Example 1)
// ============================================

class BankingError extends Error {
  constructor(message, code, statusCode = 500, details = {}) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    this.details = details;
    this.timestamp = new Date().toISOString();
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

class AccountNotFoundError extends BankingError {
  constructor(accountId) {
    super(`Account not found: ${accountId}`, 'ACCOUNT_NOT_FOUND', 404, { accountId });
  }
}

class InsufficientFundsError extends BankingError {
  constructor(accountId, available, requested) {
    super(
      `Insufficient funds in account ${accountId}`,
      'INSUFFICIENT_FUNDS',
      402,
      { accountId, available, requested, shortfall: requested - available }
    );
  }
}

class TransactionLimitExceededError extends BankingError {
  constructor(accountId, limit, amount) {
    super(
      `Transaction limit exceeded: daily limit is ${limit}`,
      'TRANSACTION_LIMIT_EXCEEDED',
      400,
      { accountId, limit, amount, excess: amount - limit }
    );
  }
}

class DatabaseError extends BankingError {
  constructor(operation, originalError) {
    super(
      `Database operation failed: ${operation}`,
      'DATABASE_ERROR',
      500,
      { operation, originalError: originalError.message }
    );
  }
}

// ============================================
// Mock Database
// ============================================

class Database {
  constructor() {
    this.accounts = new Map([
      ['ACC-001', { id: 'ACC-001', balance: 1000, dailyLimit: 5000, dailySpent: 0 }],
      ['ACC-002', { id: 'ACC-002', balance: 500, dailyLimit: 2000, dailySpent: 1500 }],
      ['ACC-003', { id: 'ACC-003', balance: 10000, dailyLimit: 10000, dailySpent: 0 }]
    ]);
    this.transactions = [];
  }
  
  async getAccount(accountId) {
    // Simulate async database call
    await this.delay(10);
    
    const account = this.accounts.get(accountId);
    if (!account) {
      throw new AccountNotFoundError(accountId);
    }
    
    return { ...account };
  }
  
  async updateBalance(accountId, amount) {
    await this.delay(10);
    
    const account = this.accounts.get(accountId);
    if (!account) {
      throw new AccountNotFoundError(accountId);
    }
    
    account.balance += amount;
    return { ...account };
  }
  
  async createTransaction(transaction) {
    await this.delay(10);
    
    // Simulate occasional database error
    if (Math.random() < 0.1) {
      throw new DatabaseError('INSERT transaction', new Error('Connection timeout'));
    }
    
    this.transactions.push(transaction);
    return transaction;
  }
  
  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ============================================
// Payment Service (Layer 3 - Highest Level)
// ============================================

class PaymentService {
  constructor(transactionService) {
    this.transactionService = transactionService;
  }
  
  /**
   * Process payment - Entry point
   * Catches and handles all errors from lower layers
   */
  async processPayment(fromAccountId, toAccountId, amount, description) {
    const transactionId = `TXN-${Date.now()}`;
    
    console.log(`\n💳 Processing Payment: ${transactionId}`);
    console.log(`   From: ${fromAccountId} → To: ${toAccountId}`);
    console.log(`   Amount: $${amount}`);
    
    try {
      // Call lower layer - errors will propagate up
      const result = await this.transactionService.transfer(
        fromAccountId,
        toAccountId,
        amount,
        description
      );
      
      console.log(`   ✅ Success: ${transactionId}`);
      
      return {
        success: true,
        transactionId,
        ...result
      };
      
    } catch (error) {
      console.log(`   ❌ Failed: ${error.code}`);
      
      // Handle specific error types differently
      if (error instanceof InsufficientFundsError) {
        console.log(`   💡 Suggestion: Add $${error.details.shortfall} to account`);
        
        return {
          success: false,
          error: {
            code: error.code,
            message: error.message,
            suggestion: `Please add $${error.details.shortfall} to your account`,
            details: error.details
          }
        };
      }
      
      if (error instanceof TransactionLimitExceededError) {
        console.log(`   💡 Suggestion: Limit resets tomorrow or request increase`);
        
        return {
          success: false,
          error: {
            code: error.code,
            message: error.message,
            suggestion: 'Daily limit will reset tomorrow. Contact support to increase limit.',
            details: error.details
          }
        };
      }
      
      if (error instanceof AccountNotFoundError) {
        console.log(`   💡 Suggestion: Verify account number`);
        
        return {
          success: false,
          error: {
            code: error.code,
            message: error.message,
            suggestion: 'Please verify the account number and try again',
            details: error.details
          }
        };
      }
      
      if (error instanceof DatabaseError) {
        console.log(`   💡 Action: Retry after delay`);
        
        return {
          success: false,
          error: {
            code: error.code,
            message: 'Temporary system error. Please try again.',
            retryable: true,
            details: error.details
          }
        };
      }
      
      // Unknown error - don't expose details
      console.error(`   🚨 Unexpected error:`, error);
      
      return {
        success: false,
        error: {
          code: 'INTERNAL_ERROR',
          message: 'An unexpected error occurred. Please contact support.',
          details: {
            timestamp: new Date().toISOString()
          }
        }
      };
    }
  }
}

// ============================================
// Transaction Service (Layer 2 - Middle Level)
// ============================================

class TransactionService {
  constructor(accountService, database) {
    this.accountService = accountService;
    this.database = database;
  }
  
  /**
   * Transfer money between accounts
   * Validates and coordinates the transfer
   * Errors from accountService propagate up
   */
  async transfer(fromAccountId, toAccountId, amount, description) {
    // Validate amount
    if (amount <= 0) {
      throw new BankingError('Amount must be positive', 'INVALID_AMOUNT', 400, { amount });
    }
    
    // Check source account (errors propagate from accountService)
    await this.accountService.validateAccountForWithdrawal(fromAccountId, amount);
    
    // Check destination account exists
    await this.accountService.validateAccountExists(toAccountId);
    
    // Execute transfer (atomic-like operation)
    try {
      // Debit source account
      await this.database.updateBalance(fromAccountId, -amount);
      
      // Credit destination account
      await this.database.updateBalance(toAccountId, amount);
      
      // Record transaction
      const transaction = {
        id: `TXN-${crypto.randomBytes(4).toString('hex')}`,
        fromAccountId,
        toAccountId,
        amount,
        description,
        timestamp: new Date().toISOString(),
        status: 'completed'
      };
      
      await this.database.createTransaction(transaction);
      
      return {
        transaction,
        message: 'Transfer completed successfully'
      };
      
    } catch (error) {
      // Rollback would happen here in real system
      console.log(`   ⚠️  Transaction failed, would rollback here`);
      
      // Re-throw to propagate up
      throw error;
    }
  }
}

// ============================================
// Account Service (Layer 1 - Lowest Level)
// ============================================

class AccountService {
  constructor(database) {
    this.database = database;
  }
  
  /**
   * Validate account exists
   * Throws AccountNotFoundError if not found
   */
  async validateAccountExists(accountId) {
    // If getAccount throws, it propagates up
    await this.database.getAccount(accountId);
    return true;
  }
  
  /**
   * Validate account has sufficient funds and within limits
   * Throws specific errors that propagate up
   */
  async validateAccountForWithdrawal(accountId, amount) {
    const account = await this.database.getAccount(accountId);
    
    // Check sufficient funds
    if (account.balance < amount) {
      throw new InsufficientFundsError(accountId, account.balance, amount);
    }
    
    // Check daily limit
    if (account.dailySpent + amount > account.dailyLimit) {
      throw new TransactionLimitExceededError(
        accountId,
        account.dailyLimit,
        account.dailySpent + amount
      );
    }
    
    return true;
  }
}

// ============================================
// Demo: Error Propagation
// ============================================

async function demonstrateErrorPropagation() {
  const database = new Database();
  const accountService = new AccountService(database);
  const transactionService = new TransactionService(accountService, database);
  const paymentService = new PaymentService(transactionService);
  
  console.log('\n🔬 DEMO: Error Propagation Through Layers\n');
  
  // Scenario 1: Success
  console.log('='.repeat(70));
  console.log('SCENARIO 1: Successful Payment');
  console.log('='.repeat(70));
  
  const result1 = await paymentService.processPayment(
    'ACC-001',
    'ACC-003',
    500,
    'Payment for services'
  );
  console.log('\nResult:', JSON.stringify(result1, null, 2));
  
  // Scenario 2: Insufficient Funds
  console.log('\n' + '='.repeat(70));
  console.log('SCENARIO 2: Insufficient Funds');
  console.log('='.repeat(70));
  
  const result2 = await paymentService.processPayment(
    'ACC-002',  // Has $500
    'ACC-003',
    750,       // Trying to send $750
    'Large payment'
  );
  console.log('\nResult:', JSON.stringify(result2, null, 2));
  
  // Scenario 3: Account Not Found
  console.log('\n' + '='.repeat(70));
  console.log('SCENARIO 3: Account Not Found');
  console.log('='.repeat(70));
  
  const result3 = await paymentService.processPayment(
    'ACC-001',
    'ACC-999',  // Doesn't exist
    100,
    'Payment to invalid account'
  );
  console.log('\nResult:', JSON.stringify(result3, null, 2));
  
  // Scenario 4: Transaction Limit Exceeded
  console.log('\n' + '='.repeat(70));
  console.log('SCENARIO 4: Transaction Limit Exceeded');
  console.log('='.repeat(70));
  
  const result4 = await paymentService.processPayment(
    'ACC-002',  // dailyLimit: 2000, dailySpent: 1500
    'ACC-003',
    600,       // Would exceed limit (1500 + 600 = 2100 > 2000)
    'Large payment'
  );
  console.log('\nResult:', JSON.stringify(result4, null, 2));
  
  // Scenario 5: Multiple retries with database error
  console.log('\n' + '='.repeat(70));
  console.log('SCENARIO 5: Database Error with Retry');
  console.log('='.repeat(70));
  
  let attempts = 0;
  let success = false;
  
  while (attempts < 3 && !success) {
    attempts++;
    console.log(`\nAttempt ${attempts}:`);
    
    const result5 = await paymentService.processPayment(
      'ACC-003',
      'ACC-001',
      100,
      'Payment with potential DB error'
    );
    
    if (result5.success) {
      console.log('✅ Transaction succeeded!');
      success = true;
    } else if (result5.error.retryable) {
      console.log('⏳ Retrying after 500ms...');
      await new Promise(resolve => setTimeout(resolve, 500));
    } else {
      console.log('❌ Non-retryable error, stopping');
      break;
    }
  }
}

// Run demo
demonstrateErrorPropagation().then(() => {
  console.log('\n' + '='.repeat(70));
  console.log('✅ Error Propagation Demo Complete!');
  console.log('='.repeat(70) + '\n');
});
```

**To run this example:**

```bash
node error-propagation-system.js
```

**Expected Output:**

```
🔄 Error Propagation System

======================================================================

🔬 DEMO: Error Propagation Through Layers

======================================================================
SCENARIO 1: Successful Payment
======================================================================

💳 Processing Payment: TXN-1731585000123
   From: ACC-001 → To: ACC-003
   Amount: $500
   ✅ Success: TXN-1731585000123

Result: {
  "success": true,
  "transactionId": "TXN-1731585000123",
  "transaction": {
    "id": "TXN-a3f7e2d9",
    "fromAccountId": "ACC-001",
    "toAccountId": "ACC-003",
    "amount": 500,
    "description": "Payment for services",
    "timestamp": "2025-11-14T10:30:00.123Z",
    "status": "completed"
  },
  "message": "Transfer completed successfully"
}

======================================================================
SCENARIO 2: Insufficient Funds
======================================================================

💳 Processing Payment: TXN-1731585000456
   From: ACC-002 → To: ACC-003
   Amount: $750
   ❌ Failed: INSUFFICIENT_FUNDS
   💡 Suggestion: Add $250 to account

Result: {
  "success": false,
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Insufficient funds in account ACC-002",
    "suggestion": "Please add $250 to your account",
    "details": {
      "accountId": "ACC-002",
      "available": 500,
      "requested": 750,
      "shortfall": 250
    }
  }
}

======================================================================
SCENARIO 3: Account Not Found
======================================================================

💳 Processing Payment: TXN-1731585000789
   From: ACC-001 → To: ACC-999
   Amount: $100
   ❌ Failed: ACCOUNT_NOT_FOUND
   💡 Suggestion: Verify account number

Result: {
  "success": false,
  "error": {
    "code": "ACCOUNT_NOT_FOUND",
    "message": "Account not found: ACC-999",
    "suggestion": "Please verify the account number and try again",
    "details": {
      "accountId": "ACC-999"
    }
  }
}

======================================================================
SCENARIO 4: Transaction Limit Exceeded
======================================================================

💳 Processing Payment: TXN-1731585001012
   From: ACC-002 → To: ACC-003
   Amount: $600
   ❌ Failed: TRANSACTION_LIMIT_EXCEEDED
   💡 Suggestion: Limit resets tomorrow or request increase

Result: {
  "success": false,
  "error": {
    "code": "TRANSACTION_LIMIT_EXCEEDED",
    "message": "Transaction limit exceeded: daily limit is 2000",
    "suggestion": "Daily limit will reset tomorrow. Contact support to increase limit.",
    "details": {
      "accountId": "ACC-002",
      "limit": 2000,
      "amount": 2100,
      "excess": 100
    }
  }
}

======================================================================
SCENARIO 5: Database Error with Retry
======================================================================

Attempt 1:

💳 Processing Payment: TXN-1731585001345
   From: ACC-003 → To: ACC-001
   Amount: $100
   ❌ Failed: DATABASE_ERROR
   💡 Action: Retry after delay
⏳ Retrying after 500ms...

Attempt 2:

💳 Processing Payment: TXN-1731585001678
   From: ACC-003 → To: ACC-001
   Amount: $100
   ✅ Success: TXN-1731585001678
✅ Transaction succeeded!

======================================================================
✅ Error Propagation Demo Complete!
======================================================================
```

**Key Learnings from Example 2**:

1. **Error Propagation**: Errors thrown in lower layers automatically propagate up
2. **Async/Await Error Handling**: Use try-catch with async/await
3. **Layer Responsibility**: Each layer handles what it can, propagates rest
4. **Error Context Preservation**: Error details maintained through layers
5. **Specific Error Handling**: Handle different error types appropriately
6. **Retry Logic**: Implement retry for transient errors
7. **Rollback**: Plan for rollback on partial failures
8. **User-Friendly Messages**: Convert technical errors to user messages

---

## 🔄 Error Recovery Strategies

### 1. Retry with Exponential Backoff

```javascript
class RetryHandler {
  async retryWithBackoff(fn, maxRetries = 3, baseDelay = 1000) {
    let lastError;
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;
        
        // Don't retry if it's not a retryable error
        if (error instanceof BankingError && !error.isRetryable) {
          throw error;
        }
        
        if (attempt < maxRetries) {
          const delay = baseDelay * Math.pow(2, attempt - 1);
          console.log(`Attempt ${attempt} failed. Retrying in ${delay}ms...`);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }
    
    throw lastError;
  }
}

// Usage
const retryHandler = new RetryHandler();

const result = await retryHandler.retryWithBackoff(
  async () => {
    return await paymentGateway.charge(cardToken, amount);
  },
  3,  // max retries
  1000  // initial delay (1s, then 2s, then 4s)
);
```

### 2. Circuit Breaker Pattern

```javascript
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.threshold = threshold;
    this.timeout = timeout;
    this.failureCount = 0;
    this.state = 'CLOSED';  // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }
  
  async execute(fn) {
    // If circuit is open and timeout hasn't passed, fail fast
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new ServiceUnavailableError(
          'Circuit breaker is OPEN',
          Math.ceil((this.nextAttempt - Date.now()) / 1000)
        );
      }
      // Timeout passed, try half-open
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      
      // Success - reset circuit
      if (this.state === 'HALF_OPEN') {
        this.state = 'CLOSED';
        this.failureCount = 0;
        console.log('Circuit breaker: OPEN → CLOSED (recovered)');
      }
      
      return result;
      
    } catch (error) {
      this.failureCount++;
      
      // Check if we should open the circuit
      if (this.failureCount >= this.threshold) {
        this.state = 'OPEN';
        this.nextAttempt = Date.now() + this.timeout;
        console.log(`Circuit breaker: → OPEN (failures: ${this.failureCount})`);
      }
      
      throw error;
    }
  }
  
  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      nextAttempt: this.state === 'OPEN' ? new Date(this.nextAttempt) : null
    };
  }
}

// Usage
const paymentGatewayBreaker = new CircuitBreaker(5, 60000);

try {
  const result = await paymentGatewayBreaker.execute(async () => {
    return await paymentGateway.processPayment(paymentData);
  });
} catch (error) {
  if (error.code === 'SERVICE_UNAVAILABLE') {
    // Circuit is open, use fallback
    console.log('Payment gateway unavailable, queuing for later processing');
    await queuePaymentForRetry(paymentData);
  }
}
```

### 3. Fallback Strategies

```javascript
class PaymentProcessor {
  constructor() {
    this.primaryGateway = new StripeGateway();
    this.fallbackGateway = new BraintreeGateway();
  }
  
  async processPayment(paymentData) {
    try {
      // Try primary gateway
      return await this.primaryGateway.charge(paymentData);
      
    } catch (primaryError) {
      console.log('Primary gateway failed, trying fallback...');
      
      try {
        // Try fallback gateway
        return await this.fallbackGateway.charge(paymentData);
        
      } catch (fallbackError) {
        // Both failed - aggregate errors
        throw new AggregateError(
          [primaryError, fallbackError],
          'All payment gateways failed'
        );
      }
    }
  }
}
```

### 4. Error Aggregation

```javascript
class BulkTransactionProcessor {
  async processBatch(transactions) {
    const results = [];
    const errors = [];
    
    await Promise.allSettled(
      transactions.map(async (txn) => {
        try {
          const result = await this.processTransaction(txn);
          results.push({ transactionId: txn.id, status: 'success', result });
        } catch (error) {
          errors.push({
            transactionId: txn.id,
            error: {
              code: error.code,
              message: error.message,
              details: error.details
            }
          });
        }
      })
    );
    
    return {
      successful: results.length,
      failed: errors.length,
      results,
      errors
    };
  }
}
```

---

## 🚨 Operational vs Programmer Errors

### Operational Errors (Expected, Recoverable)

These are expected runtime problems that should be handled:

```javascript
// Examples of operational errors
class OperationalErrors {
  examples() {
    // User input errors
    throw new InvalidAmountError(-50);
    
    // Resource not found
    throw new AccountNotFoundError('ACC-999');
    
    // Business rule violations
    throw new InsufficientFundsError('ACC-123', 100, 500);
    
    // Network/external service errors
    throw new PaymentGatewayError('Stripe', 'timeout', 'Request timed out');
    
    // Database connection errors
    throw new DatabaseError('CONNECT', new Error('Connection refused'));
  }
  
  handling() {
    return `
      Operational errors should be:
      1. Caught and handled gracefully
      2. Logged with context
      3. Returned to user with helpful message
      4. Retried if appropriate
      5. Recovered from automatically when possible
    `;
  }
}
```

### Programmer Errors (Unexpected, Bugs)

These are bugs that should crash the application:

```javascript
// Examples of programmer errors
class ProgrammerErrors {
  examples() {
    // Calling function with wrong arguments
    processPayment(undefined, null, "invalid");
    
    // Accessing undefined property
    const amount = account.balance.toString();  // account is undefined
    
    // Type errors
    const total = "100" + 50;  // String concatenation instead of addition
    
    // Logic errors
    if (balance = 0) {  // Assignment instead of comparison
      throw new Error('No balance');
    }
    
    // Async errors
    async function getData() {
      return data.value;  // Forgot to await database call
    }
  }
  
  handling() {
    return `
      Programmer errors should:
      1. Crash the application immediately
      2. Log full stack trace
      3. Alert monitoring systems
      4. NOT be caught and hidden
      5. Be fixed through debugging
      
      Use process.on('uncaughtException') and process.on('unhandledRejection')
      to log these, but then exit the process.
    `;
  }
}
```

### Global Error Handlers

```javascript
// Handle uncaught exceptions (programmer errors)
process.on('uncaughtException', (error) => {
  console.error('💥 UNCAUGHT EXCEPTION! Shutting down...');
  console.error(error.name, error.message);
  console.error(error.stack);
  
  // Log to monitoring service
  logger.fatal({ error }, 'Uncaught exception');
  
  // Graceful shutdown
  process.exit(1);
});

// Handle unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  console.error('💥 UNHANDLED REJECTION! Shutting down...');
  console.error('Reason:', reason);
  console.error('Promise:', promise);
  
  // Log to monitoring service
  logger.fatal({ reason, promise }, 'Unhandled rejection');
  
  // Graceful shutdown
  process.exit(1);
});

// Graceful shutdown on SIGTERM
process.on('SIGTERM', () => {
  console.log('👋 SIGTERM received. Graceful shutdown...');
  
  // Close server
  server.close(() => {
    console.log('💀 Server closed');
    
    // Close database connections
    database.close(() => {
      console.log('💀 Database closed');
      process.exit(0);
    });
  });
});
```

---

## 📊 Error Monitoring and Logging

### Structured Error Logging

```javascript
class ErrorLogger {
  constructor(logger) {
    this.logger = logger;  // Winston, Bunyan, Pino, etc.
  }
  
  logError(error, context = {}) {
    const logData = {
      timestamp: new Date().toISOString(),
      environment: process.env.NODE_ENV,
      
      // Error details
      error: {
        name: error.name,
        message: error.message,
        code: error.code || 'UNKNOWN',
        stack: error.stack
      },
      
      // Request context
      request: {
        id: context.requestId,
        method: context.method,
        url: context.url,
        userId: context.userId,
        ip: context.ip
      },
      
      // Business context
      business: {
        accountId: context.accountId,
        transactionId: context.transactionId,
        amount: context.amount
      }
    };
    
    // Add error-specific details
    if (error instanceof BankingError) {
      logData.error.statusCode = error.statusCode;
      logData.error.details = error.details;
      logData.error.isOperational = error.isOperational;
    }
    
    // Log with appropriate level
    if (error.statusCode >= 500) {
      this.logger.error(logData);
    } else if (error.statusCode >= 400) {
      this.logger.warn(logData);
    } else {
      this.logger.info(logData);
    }
    
    // Send to monitoring service
    if (error.statusCode >= 500 || !error.isOperational) {
      this.sendToMonitoring(logData);
    }
  }
  
  sendToMonitoring(logData) {
    // Send to Sentry, DataDog, New Relic, etc.
    // monitoring.captureException(logData);
  }
}
```

### Error Tracking Metrics

```javascript
class ErrorMetrics {
  constructor() {
    this.errorCounts = new Map();
    this.errorRates = new Map();
  }
  
  track(error) {
    const errorType = error.code || error.name;
    
    // Increment counter
    const count = this.errorCounts.get(errorType) || 0;
    this.errorCounts.set(errorType, count + 1);
    
    // Calculate rate (errors per minute)
    const now = Date.now();
    const minuteKey = Math.floor(now / 60000);
    const rateKey = `${errorType}:${minuteKey}`;
    
    const rate = this.errorRates.get(rateKey) || 0;
    this.errorRates.set(rateKey, rate + 1);
    
    // Alert if error rate is too high
    if (rate > 10) {
      this.alertHighErrorRate(errorType, rate);
    }
  }
  
  alertHighErrorRate(errorType, rate) {
    console.log(`🚨 HIGH ERROR RATE: ${errorType} = ${rate}/min`);
    // Send alert to PagerDuty, Slack, etc.
  }
  
  getMetrics() {
    return {
      errorCounts: Object.fromEntries(this.errorCounts),
      topErrors: this.getTopErrors(5)
    };
  }
  
  getTopErrors(limit) {
    return Array.from(this.errorCounts.entries())
      .sort(([, a], [, b]) => b - a)
      .slice(0, limit)
      .map(([type, count]) => ({ type, count }));
  }
}
```

---

## 💡 DO's - Best Practices

### 1. ✅ DO: Create Specific Error Classes

```javascript
// ✅ GOOD: Specific error classes for different scenarios
class InsufficientFundsError extends BankingError {
  constructor(accountId, available, requested) {
    super(
      `Insufficient funds in account ${accountId}`,
      'INSUFFICIENT_FUNDS',
      402,
      { accountId, available, requested, shortfall: requested - available }
    );
  }
}

class TransactionLimitExceededError extends BankingError {
  constructor(accountId, limit, amount) {
    super(
      `Transaction limit exceeded: daily limit is ${limit}`,
      'TRANSACTION_LIMIT_EXCEEDED',
      400,
      { accountId, limit, amount, excess: amount - limit }
    );
  }
}

// ❌ BAD: Generic errors
throw new Error('Transaction failed');
throw new Error('Invalid request');
```

**Why**: Specific errors allow proper handling based on error type and provide structured context.

---

### 2. ✅ DO: Use Error Codes

```javascript
// ✅ GOOD: Consistent error codes
class BankingError extends Error {
  constructor(message, code, statusCode = 500, details = {}) {
    super(message);
    this.code = code;  // E.g., 'INSUFFICIENT_FUNDS', 'ACCOUNT_NOT_FOUND'
    this.statusCode = statusCode;
    this.details = details;
  }
}

// Client can handle by code
if (error.code === 'INSUFFICIENT_FUNDS') {
  showFundingOptions();
} else if (error.code === 'ACCOUNT_LOCKED') {
  showCustomerSupport();
}

// ❌ BAD: No error codes, only messages
throw new Error('Transaction failed due to insufficient funds');

// Client must parse message string
if (error.message.includes('insufficient funds')) {
  // Fragile!
}
```

**Why**: Error codes provide a stable API for error handling. Messages can change, codes shouldn't.

---

### 3. ✅ DO: Include Error Context

```javascript
// ✅ GOOD: Rich error context
throw new InsufficientFundsError('ACC-12345', 100, 500);
// Results in:
{
  code: 'INSUFFICIENT_FUNDS',
  message: 'Insufficient funds in account ACC-12345',
  details: {
    accountId: 'ACC-12345',
    available: 100,
    requested: 500,
    shortfall: 400
  }
}

// ❌ BAD: No context
throw new Error('Insufficient funds');
```

**Why**: Context helps debugging, user support, and provides actionable information.

---

### 4. ✅ DO: Distinguish Operational vs Programmer Errors

```javascript
// ✅ GOOD: Flag operational errors
class BankingError extends Error {
  constructor(message, code, statusCode, details) {
    super(message);
    this.code = code;
    this.statusCode = statusCode;
    this.details = details;
    this.isOperational = true;  // This is expected, handle gracefully
  }
}

// Handle accordingly
if (error.isOperational) {
  // Log and return error response
  logger.warn(error);
  res.status(error.statusCode).json(error.toJSON());
} else {
  // Programmer error - log and crash
  logger.fatal(error);
  process.exit(1);
}

// ❌ BAD: Treat all errors the same
catch (error) {
  console.error(error);
  res.status(500).json({ error: 'Something went wrong' });
}
```

**Why**: Operational errors should be handled gracefully. Programmer errors should crash to surface bugs.

---

### 5. ✅ DO: Implement Retry Logic for Transient Errors

```javascript
// ✅ GOOD: Retry transient errors with backoff
class PaymentGatewayError extends BankingError {
  constructor(gateway, code, message) {
    super(`Payment gateway error: ${message}`, 'PAYMENT_GATEWAY_ERROR', 502, {
      gateway, code, message
    });
    this.isRetryable = true;  // Mark as retryable
  }
}

async function processWithRetry(fn, maxRetries = 3) {
  let lastError;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (!error.isRetryable) {
        throw error;  // Don't retry non-retryable errors
      }
      
      if (i < maxRetries - 1) {
        await delay(Math.pow(2, i) * 1000);  // Exponential backoff
      }
    }
  }
  
  throw lastError;
}

// ❌ BAD: No retry for transient errors
try {
  await paymentGateway.charge(amount);
} catch (error) {
  throw error;  // Give up immediately
}
```

**Why**: Network and external service errors are often transient. Retrying increases success rate.

---

### 6. ✅ DO: Use Circuit Breaker for External Services

```javascript
// ✅ GOOD: Circuit breaker prevents cascading failures
const paymentGatewayBreaker = new CircuitBreaker({
  threshold: 5,      // Open after 5 failures
  timeout: 60000,    // Try again after 60s
  name: 'payment-gateway'
});

try {
  const result = await paymentGatewayBreaker.execute(async () => {
    return await paymentGateway.charge(paymentData);
  });
} catch (error) {
  if (error.code === 'CIRCUIT_OPEN') {
    // Queue for later processing
    await queuePayment(paymentData);
  }
}

// ❌ BAD: Keep hitting failing service
for (let i = 0; i < 100; i++) {
  try {
    await paymentGateway.charge(payments[i]);
  } catch (error) {
    // Keep trying even though service is down
  }
}
```

**Why**: Circuit breaker prevents wasting resources on a failing service and allows it time to recover.

---

### 7. ✅ DO: Log Errors with Context

```javascript
// ✅ GOOD: Structured logging with context
logger.error({
  error: {
    name: error.name,
    message: error.message,
    code: error.code,
    stack: error.stack
  },
  request: {
    id: requestId,
    method: req.method,
    url: req.url,
    userId: req.user?.id
  },
  business: {
    accountId: transaction.accountId,
    amount: transaction.amount,
    transactionId: transaction.id
  },
  timestamp: new Date().toISOString()
});

// ❌ BAD: Just log the error
console.error(error);
// Or worse:
console.error('An error occurred');
```

**Why**: Context helps debug production issues. You need to know what was happening when the error occurred.

---

### 8. ✅ DO: Handle Async Errors Properly

```javascript
// ✅ GOOD: Proper async error handling
async function processPayment(data) {
  try {
    const account = await getAccount(data.accountId);
    const result = await chargeCard(data.cardToken, data.amount);
    await saveTransaction(result);
    return result;
  } catch (error) {
    logger.error({ error, data }, 'Payment processing failed');
    throw error;  // Propagate to caller
  }
}

// Use in Express
app.post('/payments', async (req, res, next) => {
  try {
    const result = await processPayment(req.body);
    res.json(result);
  } catch (error) {
    next(error);  // Pass to error middleware
  }
});

// ❌ BAD: Unhandled promise rejection
async function processPayment(data) {
  const account = await getAccount(data.accountId);  // If this throws, unhandled!
  return await chargeCard(data.cardToken, data.amount);
}

// Or missing try-catch in route
app.post('/payments', async (req, res) => {
  const result = await processPayment(req.body);  // Unhandled rejection!
  res.json(result);
});
```

**Why**: Unhandled promise rejections crash Node.js (or will in future versions). Always handle async errors.

---

### 9. ✅ DO: Implement Global Error Handlers

```javascript
// ✅ GOOD: Catch-all error handlers
// Express error middleware
app.use((error, req, res, next) => {
  // Log error
  errorLogger.logError(error, {
    requestId: req.id,
    method: req.method,
    url: req.url,
    userId: req.user?.id
  });
  
  // Send appropriate response
  if (error instanceof BankingError) {
    res.status(error.statusCode).json(error.toJSON());
  } else {
    res.status(500).json({
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred'
      }
    });
  }
});

// Process-level handlers
process.on('uncaughtException', (error) => {
  logger.fatal({ error }, 'Uncaught exception');
  gracefulShutdown();
});

process.on('unhandledRejection', (reason, promise) => {
  logger.fatal({ reason, promise }, 'Unhandled rejection');
  gracefulShutdown();
});

// ❌ BAD: No global handlers
// Errors crash the application with no logging
```

**Why**: Global handlers ensure no error goes unnoticed and allows graceful shutdown.

---

### 10. ✅ DO: Test Error Scenarios

```javascript
// ✅ GOOD: Test all error paths
describe('Payment Processing', () => {
  it('should throw InsufficientFundsError when balance too low', async () => {
    const account = { id: 'ACC-1', balance: 100 };
    
    await expect(
      processPayment(account, 500)
    ).rejects.toThrow(InsufficientFundsError);
    
    await expect(
      processPayment(account, 500)
    ).rejects.toMatchObject({
      code: 'INSUFFICIENT_FUNDS',
      statusCode: 402,
      details: {
        accountId: 'ACC-1',
        available: 100,
        requested: 500,
        shortfall: 400
      }
    });
  });
  
  it('should retry on PaymentGatewayError', async () => {
    const mockGateway = jest.fn()
      .mockRejectedValueOnce(new PaymentGatewayError('Stripe', 'timeout', 'Timeout'))
      .mockResolvedValueOnce({ id: 'charge_123', status: 'success' });
    
    const result = await processWithRetry(() => mockGateway());
    
    expect(mockGateway).toHaveBeenCalledTimes(2);
    expect(result.status).toBe('success');
  });
});

// ❌ BAD: Only test happy paths
it('should process payment successfully', async () => {
  const result = await processPayment({ amount: 100 });
  expect(result.status).toBe('success');
});
```

**Why**: Error scenarios are critical paths. Test them thoroughly to ensure proper error handling.

---

## 🚫 DON'Ts - Common Mistakes

### 1. ❌ DON'T: Swallow Errors

```javascript
// ❌ BAD: Swallow errors silently
try {
  await processPayment(data);
} catch (error) {
  // Silent failure - user thinks payment succeeded!
}

// ❌ BAD: Log but don't propagate
try {
  await saveTransaction(transaction);
} catch (error) {
  console.error(error);  // Logged but caller doesn't know it failed
}

// ✅ GOOD: Log and propagate
try {
  await saveTransaction(transaction);
} catch (error) {
  logger.error({ error, transaction }, 'Failed to save transaction');
  throw error;  // Let caller handle it
}

// ✅ GOOD: Log and return error status
try {
  await processPayment(data);
  return { success: true };
} catch (error) {
  logger.error({ error, data }, 'Payment failed');
  return { success: false, error: error.toJSON() };
}
```

**Why**: Swallowing errors hides problems and creates inconsistent state. Always handle errors explicitly.

---

### 2. ❌ DON'T: Expose Internal Error Details

```javascript
// ❌ BAD: Expose internal details
app.use((error, req, res, next) => {
  res.status(500).json({
    error: error.message,  // Might contain sensitive info
    stack: error.stack,    // Exposes code structure
    query: error.sql       // Exposes database schema
  });
});

// ✅ GOOD: Safe error responses
app.use((error, req, res, next) => {
  // Log full details internally
  logger.error({ error, req });
  
  // Send safe response to client
  if (error instanceof BankingError && error.isOperational) {
    res.status(error.statusCode).json({
      error: {
        code: error.code,
        message: error.message,
        details: error.details  // Only if safe
      }
    });
  } else {
    // Don't expose programmer errors
    res.status(500).json({
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred'
      }
    });
  }
});
```

**Why**: Internal errors may contain sensitive information (database queries, file paths, credentials).

---

### 3. ❌ DON'T: Use Strings for Error Types

```javascript
// ❌ BAD: String-based error checking
throw new Error('INSUFFICIENT_FUNDS');

// Client code becomes fragile
if (error.message === 'INSUFFICIENT_FUNDS') {
  // Breaks if message changes
}

// ✅ GOOD: Class-based error types
throw new InsufficientFundsError(accountId, available, requested);

// Client code is robust
if (error instanceof InsufficientFundsError) {
  const shortfall = error.details.shortfall;
  showFundingOptions(shortfall);
}

// Or use error codes
if (error.code === 'INSUFFICIENT_FUNDS') {
  // Stable API
}
```

**Why**: String comparison is fragile and error-prone. Use instanceof or error codes.

---

### 4. ❌ DON'T: Create Error Objects Without Throwing

```javascript
// ❌ BAD: Return error object
function validateAmount(amount) {
  if (amount <= 0) {
    return new Error('Amount must be positive');  // Created but not thrown!
  }
  return null;
}

// Caller might miss the error
const error = validateAmount(-50);
if (!error) {  // Oops, error is truthy!
  processPayment(amount);
}

// ✅ GOOD: Throw errors
function validateAmount(amount) {
  if (amount <= 0) {
    throw new InvalidAmountError(amount, 'Amount must be positive');
  }
}

// Or return success/failure object
function validateAmount(amount) {
  if (amount <= 0) {
    return {
      valid: false,
      error: 'Amount must be positive'
    };
  }
  return { valid: true };
}

// Caller handles explicitly
const validation = validateAmount(amount);
if (!validation.valid) {
  throw new InvalidAmountError(amount, validation.error);
}
```

**Why**: Errors should be thrown, not returned. Returning errors makes it easy to miss them.

---

### 5. ❌ DON'T: Catch Errors Too Early

```javascript
// ❌ BAD: Catch too early, lose context
async function getAccount(accountId) {
  try {
    return await database.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
  } catch (error) {
    console.error('Database error');
    throw new Error('Failed to get account');  // Lost original error!
  }
}

// ✅ GOOD: Let errors propagate with context
async function getAccount(accountId) {
  try {
    return await database.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
  } catch (error) {
    // Add context and re-throw original
    throw new DatabaseError('SELECT account', error);
  }
}

// Or let it propagate naturally
async function getAccount(accountId) {
  // If database throws, it propagates automatically
  return await database.query('SELECT * FROM accounts WHERE id = ?', [accountId]);
}

// Handle at the boundary (e.g., HTTP handler)
app.get('/accounts/:id', async (req, res, next) => {
  try {
    const account = await getAccount(req.params.id);
    res.json(account);
  } catch (error) {
    next(error);  // Pass to error middleware
  }
});
```

**Why**: Catching errors too early loses the stack trace and context. Let errors propagate to the right handler.

---

### 6. ❌ DON'T: Mix Callbacks and Promises

```javascript
// ❌ BAD: Mixing callbacks and promises
function processPayment(data, callback) {
  getAccount(data.accountId)  // Returns promise
    .then(account => {
      validateBalance(account, data.amount);  // Throws error
      callback(null, account);
    })
    .catch(error => callback(error));  // Sync error not caught!
}

// ✅ GOOD: Use async/await consistently
async function processPayment(data) {
  const account = await getAccount(data.accountId);
  validateBalance(account, data.amount);  // Throws, will be caught by try-catch
  return account;
}

// Or use callbacks consistently
function processPayment(data, callback) {
  getAccount(data.accountId, (err, account) => {
    if (err) return callback(err);
    
    try {
      validateBalance(account, data.amount);
      callback(null, account);
    } catch (error) {
      callback(error);
    }
  });
}
```

**Why**: Mixing callback and promise error handling is confusing and error-prone.

---

### 7. ❌ DON'T: Ignore Unhandled Rejections

```javascript
// ❌ BAD: No unhandled rejection handler
async function init() {
  await startServer();
  await connectDatabase();
  // If either throws, unhandled rejection!
}

init();  // Potential unhandled rejection

// ✅ GOOD: Handle unhandled rejections
process.on('unhandledRejection', (reason, promise) => {
  logger.fatal({ reason, promise }, 'Unhandled promise rejection');
  gracefulShutdown();
});

// And handle in the function
async function init() {
  try {
    await startServer();
    await connectDatabase();
  } catch (error) {
    logger.fatal({ error }, 'Initialization failed');
    process.exit(1);
  }
}

init().catch(error => {
  logger.fatal({ error }, 'Init catch');
  process.exit(1);
});
```

**Why**: Unhandled rejections crash Node.js. Always have a catch handler.

---

### 8. ❌ DON'T: Use `throw` in Callbacks

```javascript
// ❌ BAD: Throw in callback
fs.readFile('file.txt', (err, data) => {
  if (err) {
    throw err;  // This will crash the process!
  }
  processData(data);
});

// ✅ GOOD: Pass error to callback
function readFileAsync(path, callback) {
  fs.readFile(path, (err, data) => {
    if (err) {
      return callback(err);  // Pass to callback
    }
    callback(null, data);
  });
}

// Or use promises
function readFileAsync(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, (err, data) => {
      if (err) {
        reject(err);  // Reject promise
      } else {
        resolve(data);
      }
    });
  });
}

// Or use util.promisify
const { promisify } = require('util');
const readFileAsync = promisify(fs.readFile);
```

**Why**: Throwing in callbacks bypasses error handling. Use callback error parameter or reject promises.

---

### 9. ❌ DON'T: Retry Non-Retryable Errors

```javascript
// ❌ BAD: Retry everything
async function processPayment(data) {
  let attempts = 0;
  
  while (attempts < 5) {
    try {
      return await chargeCard(data);
    } catch (error) {
      attempts++;
      await delay(1000);  // Retry even if card declined!
    }
  }
}

// ✅ GOOD: Only retry transient errors
async function processPayment(data) {
  let attempts = 0;
  
  while (attempts < 5) {
    try {
      return await chargeCard(data);
    } catch (error) {
      // Don't retry business logic errors
      if (error instanceof InsufficientFundsError ||
          error instanceof InvalidCardError) {
        throw error;  // No point retrying
      }
      
      // Only retry transient errors
      if (error.isRetryable) {
        attempts++;
        await delay(Math.pow(2, attempts) * 1000);
      } else {
        throw error;
      }
    }
  }
}
```

**Why**: Retrying business logic errors (insufficient funds, invalid card) is wasteful. Only retry transient errors.

---

### 10. ❌ DON'T: Forget to Clean Up on Error

```javascript
// ❌ BAD: No cleanup on error
async function processLargeFile(filePath) {
  const file = await fs.open(filePath, 'r');
  const data = await file.readFile();  // If this throws, file not closed!
  await processData(data);
  await file.close();
}

// ✅ GOOD: Always clean up
async function processLargeFile(filePath) {
  let file;
  
  try {
    file = await fs.open(filePath, 'r');
    const data = await file.readFile();
    await processData(data);
  } finally {
    if (file) {
      await file.close();  // Always close, even on error
    }
  }
}

// Or use resource management
async function processLargeFile(filePath) {
  const file = await fs.open(filePath, 'r');
  
  try {
    const data = await file.readFile();
    await processData(data);
  } finally {
    await file.close();
  }
}

// Or use streams (auto-cleanup)
const stream = fs.createReadStream(filePath);
stream.on('error', (error) => {
  // Stream auto-closes on error
  handleError(error);
});
```

**Why**: Resources (files, connections, locks) must be cleaned up even when errors occur. Use finally blocks.

---

## 🎯 Key Takeaways

### Custom Error Classes

| Feature | Purpose | Example |
|---------|---------|---------|
| **Error Hierarchy** | Organize errors by category | `BankingError` → `ValidationError` → `InvalidAmountError` |
| **Error Codes** | Stable API for error handling | `INSUFFICIENT_FUNDS`, `ACCOUNT_NOT_FOUND` |
| **HTTP Status Codes** | Map to HTTP responses | `400` (Bad Request), `402` (Payment Required), `404` (Not Found) |
| **Error Context** | Debug information | `{ accountId, available, requested, shortfall }` |
| **Operational Flag** | Distinguish expected errors | `isOperational: true` |
| **Retryable Flag** | Indicate transient errors | `isRetryable: true` |

### Error Propagation

| Layer | Responsibility | Example |
|-------|---------------|---------|
| **Data Layer** | Throw specific errors | `throw new AccountNotFoundError(id)` |
| **Business Layer** | Validate and propagate | `if (balance < amount) throw new InsufficientFundsError()` |
| **API Layer** | Catch and format response | `res.status(error.statusCode).json(error.toJSON())` |

### Error Recovery Strategies

| Strategy | Use Case | Implementation |
|----------|----------|----------------|
| **Retry with Backoff** | Transient network errors | Exponential backoff: 1s, 2s, 4s, 8s |
| **Circuit Breaker** | Repeated failures | Open after 5 failures, retry after 60s |
| **Fallback** | Service unavailable | Try primary, then fallback gateway |
| **Error Aggregation** | Batch operations | Collect all errors, return summary |

### Operational vs Programmer Errors

| Type | Examples | Handling |
|------|----------|----------|
| **Operational** | Invalid input, network errors, business rules | Catch, log, return error response |
| **Programmer** | Type errors, undefined variables, logic bugs | Log and crash immediately |

### Error Monitoring

| Metric | What to Track | Threshold |
|--------|---------------|-----------|
| **Error Rate** | Errors per minute | Alert if > 10/min |
| **Error Types** | Most common errors | Review daily |
| **Response Times** | Slow error handling | Alert if > 1s |
| **Circuit Breaker State** | Open/closed status | Alert on state change |

---

## 🎓 Interview Questions & Answers

### Q1: What's the difference between operational and programmer errors?

**Answer**:

**Operational Errors** are expected runtime problems:
- User input errors (invalid amount, missing fields)
- Resource not found (account doesn't exist)
- Business rule violations (insufficient funds)
- External service errors (payment gateway down)
- Network/database connection errors

These should be **handled gracefully** with try-catch, logged, and returned to the user with helpful messages.

**Programmer Errors** are bugs in your code:
- Calling a function with wrong arguments
- Accessing undefined properties
- Type errors
- Logic errors (assignment instead of comparison)
- Forgetting to await async operations

These should **crash the application immediately** with full stack trace so you can debug and fix them. Don't catch and hide programmer errors.

### Q2: How do you handle errors that propagate through multiple async layers?

**Answer**:

Errors thrown in lower layers automatically propagate up through async/await chains:

```javascript
// Data layer - throws specific error
async function getAccount(id) {
  const account = await db.query('SELECT * FROM accounts WHERE id = ?', [id]);
  if (!account) throw new AccountNotFoundError(id);
  return account;
}

// Business layer - error propagates automatically
async function transfer(fromId, toId, amount) {
  const fromAccount = await getAccount(fromId);  // Error propagates
  const toAccount = await getAccount(toId);      // Error propagates
  
  if (fromAccount.balance < amount) {
    throw new InsufficientFundsError(fromId, fromAccount.balance, amount);
  }
  
  // Execute transfer...
}

// API layer - catch and handle
app.post('/transfer', async (req, res, next) => {
  try {
    const result = await transfer(req.body.from, req.body.to, req.body.amount);
    res.json(result);
  } catch (error) {
    next(error);  // Pass to error middleware
  }
});
```

Key points:
1. Lower layers throw specific errors
2. Middle layers add context or propagate
3. Top layer catches and formats response
4. Use try-catch with async/await
5. Each layer handles what it can, propagates the rest

### Q3: When should you use a circuit breaker pattern?

**Answer**:

Use a circuit breaker when calling **external services** that might fail repeatedly:

**When to Use**:
- Payment gateways (Stripe, PayPal)
- External APIs (credit check, fraud detection)
- Microservices
- Databases (connection failures)

**Why**:
- Prevents wasting resources on a failing service
- Gives the service time to recover
- Provides fast failure (fail fast principle)
- Prevents cascading failures

**How it Works**:
1. **CLOSED** (normal): Requests pass through
2. **OPEN** (failing): Requests fail immediately after threshold reached
3. **HALF-OPEN** (testing): After timeout, allow one request to test recovery

```javascript
const circuitBreaker = new CircuitBreaker({
  threshold: 5,      // Open after 5 failures
  timeout: 60000,    // Try again after 60s
  name: 'payment-gateway'
});

const result = await circuitBreaker.execute(async () => {
  return await paymentGateway.charge(amount);
});
```

**Alternative**: If circuit is open, use a fallback (queue for later, use backup service).

### Q4: How do you implement retry logic for transient errors?

**Answer**:

Implement retry with **exponential backoff** for transient errors:

```javascript
async function retryWithBackoff(fn, maxRetries = 3, baseDelay = 1000) {
  let lastError;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      // Don't retry business logic errors
      if (error instanceof BankingError && !error.isRetryable) {
        throw error;
      }
      
      if (attempt < maxRetries) {
        // Exponential backoff: 1s, 2s, 4s
        const delay = baseDelay * Math.pow(2, attempt - 1);
        console.log(`Attempt ${attempt} failed. Retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  
  throw lastError;
}
```

**Key Points**:
1. Only retry **transient errors** (network, timeout, 5xx)
2. Don't retry **business logic errors** (insufficient funds, invalid card)
3. Use **exponential backoff** (1s, 2s, 4s, 8s)
4. Limit **max retries** (usually 3-5)
5. Mark errors as `isRetryable` in error class

### Q5: How do you structure error responses for a REST API?

**Answer**:

Use a consistent error response format:

```javascript
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Insufficient funds in account ACC-12345",
    "statusCode": 402,
    "details": {
      "accountId": "ACC-12345",
      "available": 100,
      "requested": 500,
      "shortfall": 400
    },
    "timestamp": "2025-01-14T10:30:00.000Z"
  }
}
```

**Best Practices**:
1. **code**: Machine-readable error code
2. **message**: Human-readable message
3. **statusCode**: HTTP status code
4. **details**: Additional context (only if safe to expose)
5. **timestamp**: When the error occurred

**Implementation**:

```javascript
class BankingError extends Error {
  toJSON() {
    return {
      error: {
        code: this.code,
        message: this.message,
        statusCode: this.statusCode,
        details: this.details,
        timestamp: this.timestamp
      }
    };
  }
}

// Error middleware
app.use((error, req, res, next) => {
  if (error instanceof BankingError && error.isOperational) {
    res.status(error.statusCode).json(error.toJSON());
  } else {
    // Don't expose programmer errors
    res.status(500).json({
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred',
        timestamp: new Date().toISOString()
      }
    });
  }
});
```

### Q6: What should you log when an error occurs?

**Answer**:

Log **structured data** with full context:

```javascript
logger.error({
  // Error details
  error: {
    name: error.name,
    message: error.message,
    code: error.code,
    stack: error.stack,
    statusCode: error.statusCode
  },
  
  // Request context
  request: {
    id: requestId,
    method: req.method,
    url: req.url,
    headers: sanitizeHeaders(req.headers),
    body: sanitizeBody(req.body),
    userId: req.user?.id,
    ip: req.ip
  },
  
  // Business context
  business: {
    accountId: transaction.accountId,
    amount: transaction.amount,
    transactionId: transaction.id
  },
  
  // Environment
  environment: process.env.NODE_ENV,
  timestamp: new Date().toISOString(),
  hostname: os.hostname(),
  pid: process.pid
});
```

**Important**:
1. Use structured logging (JSON)
2. Include **request context** (request ID, user ID, IP)
3. Include **business context** (account ID, transaction ID)
4. **Sanitize sensitive data** (passwords, card numbers, tokens)
5. Include **stack trace** for debugging
6. Add **correlation IDs** for distributed tracing

### Q7: How do you handle errors in batch/bulk operations?

**Answer**:

Use **error aggregation** to collect all errors:

```javascript
async function processBatch(transactions) {
  const results = [];
  const errors = [];
  
  // Process all transactions (don't stop on first error)
  await Promise.allSettled(
    transactions.map(async (txn) => {
      try {
        const result = await processTransaction(txn);
        results.push({
          transactionId: txn.id,
          status: 'success',
          result
        });
      } catch (error) {
        errors.push({
          transactionId: txn.id,
          status: 'failed',
          error: {
            code: error.code,
            message: error.message,
            details: error.details
          }
        });
      }
    })
  );
  
  return {
    total: transactions.length,
    successful: results.length,
    failed: errors.length,
    results,
    errors
  };
}
```

**Benefits**:
1. Process all items (don't stop on first error)
2. Return summary of successes and failures
3. Allow caller to decide how to handle partial success
4. Provide details for each failure

### Q8: How do you test error handling?

**Answer**:

Test **all error scenarios** explicitly:

```javascript
describe('Payment Processing', () => {
  // Test specific error thrown
  it('should throw InsufficientFundsError when balance too low', async () => {
    const account = { id: 'ACC-1', balance: 100 };
    
    await expect(
      processPayment(account, 500)
    ).rejects.toThrow(InsufficientFundsError);
  });
  
  // Test error details
  it('should include correct error details', async () => {
    const account = { id: 'ACC-1', balance: 100 };
    
    await expect(
      processPayment(account, 500)
    ).rejects.toMatchObject({
      code: 'INSUFFICIENT_FUNDS',
      statusCode: 402,
      details: {
        accountId: 'ACC-1',
        available: 100,
        requested: 500,
        shortfall: 400
      }
    });
  });
  
  // Test retry logic
  it('should retry on transient error', async () => {
    const mockFn = jest.fn()
      .mockRejectedValueOnce(new NetworkError('Timeout'))
      .mockResolvedValueOnce({ success: true });
    
    const result = await retryWithBackoff(() => mockFn());
    
    expect(mockFn).toHaveBeenCalledTimes(2);
    expect(result.success).toBe(true);
  });
  
  // Test no retry on business errors
  it('should not retry on business logic error', async () => {
    const mockFn = jest.fn()
      .mockRejectedValue(new InsufficientFundsError('ACC-1', 100, 500));
    
    await expect(
      retryWithBackoff(() => mockFn())
    ).rejects.toThrow(InsufficientFundsError);
    
    expect(mockFn).toHaveBeenCalledTimes(1);  // No retry
  });
  
  // Test error logging
  it('should log errors with context', async () => {
    const loggerSpy = jest.spyOn(logger, 'error');
    
    try {
      await processPayment({ amount: -50 });
    } catch (error) {
      // Error expected
    }
    
    expect(loggerSpy).toHaveBeenCalledWith(
      expect.objectContaining({
        error: expect.objectContaining({
          code: 'INVALID_AMOUNT'
        })
      })
    );
  });
});
```

**Test Coverage**:
1. Specific error types thrown
2. Error details and context
3. Retry logic (when to retry, when not to)
4. Error propagation through layers
5. Error responses (status codes, format)
6. Error logging
7. Circuit breaker state changes

### Q9: How do you implement graceful error recovery?

**Answer**:

Implement **fallback strategies** and **degraded functionality**:

```javascript
// 1. Fallback to alternative service
async function processPayment(data) {
  try {
    return await primaryPaymentGateway.charge(data);
  } catch (error) {
    console.log('Primary gateway failed, trying fallback...');
    return await fallbackPaymentGateway.charge(data);
  }
}

// 2. Queue for later processing
async function processPayment(data) {
  try {
    return await paymentGateway.charge(data);
  } catch (error) {
    if (error.isRetryable) {
      console.log('Gateway down, queuing for later...');
      await queueForRetry(data);
      return { status: 'queued', message: 'Payment queued for processing' };
    }
    throw error;
  }
}

// 3. Degrade functionality
async function getAccountWithRecommendations(accountId) {
  const account = await getAccount(accountId);  // Critical, must succeed
  
  let recommendations = [];
  try {
    recommendations = await getRecommendations(accountId);  // Nice-to-have
  } catch (error) {
    console.warn('Recommendations unavailable, continuing without them');
    // Don't fail entire request
  }
  
  return { account, recommendations };
}

// 4. Cache fallback
async function getExchangeRate(from, to) {
  try {
    return await exchangeRateAPI.get(from, to);
  } catch (error) {
    console.warn('Exchange rate API down, using cached value');
    return await cache.get(`rate:${from}:${to}`);  // Use stale data
  }
}
```

**Strategies**:
1. **Alternative Service**: Try backup gateway/API
2. **Queue for Later**: Async processing when service down
3. **Degrade Functionality**: Continue without non-critical features
4. **Cache Fallback**: Use stale data when fresh data unavailable
5. **Default Values**: Return safe defaults

### Q10: How do you prevent cascading failures?

**Answer**:

Use multiple strategies to prevent one failure from causing system-wide outage:

**1. Circuit Breaker**: Stop calling failing service
```javascript
const circuitBreaker = new CircuitBreaker({ threshold: 5, timeout: 60000 });
```

**2. Timeouts**: Don't wait forever
```javascript
const result = await Promise.race([
  fetchData(),
  new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Timeout')), 5000)
  )
]);
```

**3. Rate Limiting**: Prevent overwhelming service
```javascript
const rateLimiter = new RateLimiter({ maxRequests: 100, window: 60000 });
```

**4. Bulkheads**: Isolate failures
```javascript
// Separate thread pools for different operations
const accountPool = new ThreadPool(10);
const paymentPool = new ThreadPool(20);
```

**5. Load Shedding**: Reject requests when overloaded
```javascript
if (queue.length > MAX_QUEUE_SIZE) {
  throw new ServiceUnavailableError('Server overloaded', 30);
}
```

**6. Graceful Degradation**: Continue with reduced functionality
```javascript
// Skip non-critical features when system stressed
if (systemLoad > 0.8) {
  return { account, recommendations: [] };  // Skip recommendations
}
```

**7. Health Checks**: Detect and remove unhealthy instances
```javascript
app.get('/health', (req, res) => {
  const health = {
    database: await database.ping(),
    cache: await cache.ping(),
    queue: await queue.ping()
  };
  
  if (Object.values(health).every(v => v === 'ok')) {
    res.json({ status: 'healthy', checks: health });
  } else {
    res.status(503).json({ status: 'unhealthy', checks: health });
  }
});
```

---

## 🔧 Banking Error Handling Checklist

### Error Classes
- [ ] Base `BankingError` class with code, statusCode, details
- [ ] Validation errors (400): `InvalidAmountError`, `MissingFieldError`
- [ ] Account errors (403/404): `AccountNotFoundError`, `AccountLockedError`
- [ ] Transaction errors (402): `InsufficientFundsError`, `TransactionLimitExceededError`
- [ ] System errors (500): `DatabaseError`, `ServiceUnavailableError`
- [ ] Fraud errors (403): `SuspiciousActivityError`, `FraudDetectedError`

### Error Handling
- [ ] Try-catch around all async operations
- [ ] Error propagation through layers
- [ ] Specific error handling based on error type
- [ ] Error middleware in Express
- [ ] Global error handlers (uncaughtException, unhandledRejection)

### Error Recovery
- [ ] Retry logic with exponential backoff
- [ ] Circuit breaker for external services
- [ ] Fallback strategies (alternative service, queue, cache)
- [ ] Graceful degradation (skip non-critical features)
- [ ] Transaction rollback on failure

### Error Logging
- [ ] Structured logging with full context
- [ ] Request context (request ID, user ID, IP)
- [ ] Business context (account ID, transaction ID, amount)
- [ ] Error metrics (error rates, top errors)
- [ ] Alert on high error rates or critical errors

### Error Responses
- [ ] Consistent error response format
- [ ] Appropriate HTTP status codes
- [ ] Don't expose internal errors to clients
- [ ] User-friendly error messages
- [ ] Include error codes for client handling

### Testing
- [ ] Test all error scenarios
- [ ] Test error propagation
- [ ] Test retry logic
- [ ] Test circuit breaker state changes
- [ ] Test error responses and status codes
- [ ] Test error logging

### Monitoring
- [ ] Track error rates (errors per minute)
- [ ] Track error types (most common errors)
- [ ] Monitor circuit breaker state
- [ ] Alert on high error rates
- [ ] Dashboard for error metrics

---

## 🎓 Summary

**Advanced error handling** in Node.js is essential for building robust banking applications:

1. **Custom Error Classes**: Create domain-specific errors with codes, status codes, and context
2. **Error Hierarchies**: Organize errors by category for better handling
3. **Error Propagation**: Let errors propagate up through async layers naturally
4. **Error Recovery**: Implement retry, circuit breaker, fallback, and degradation strategies
5. **Operational vs Programmer Errors**: Handle operational errors gracefully, crash on programmer errors
6. **Error Logging**: Log with full context for debugging production issues
7. **Error Responses**: Provide consistent, helpful error responses to clients
8. **Testing**: Test all error scenarios thoroughly
9. **Monitoring**: Track error rates and types for operational visibility
10. **Resilience**: Build systems that recover from failures automatically

**Remember**: Good error handling is what separates toy applications from production-ready systems. In banking, proper error handling is not optional—it's critical for security, compliance, and customer trust.

---

## 📚 Additional Resources

- [Node.js Error Handling Best Practices](https://nodejs.org/en/docs/guides/error-handling)
- [Joyent's Error Handling Guide](https://www.joyent.com/node-js/production/design/errors)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience Engineering](https://www.oreilly.com/library/view/release-it-2nd/9781680504552/)

---

**End of Question 10: Advanced Error Handling** 🎉

**Total Lines**: ~2,000 lines of comprehensive error handling content with production-ready examples!
