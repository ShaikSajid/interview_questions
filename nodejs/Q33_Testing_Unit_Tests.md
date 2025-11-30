# Q33: Testing - Unit Tests with Jest

## 📋 Summary
Unit testing ensures individual functions and modules work correctly in isolation. This guide covers **Jest testing framework**, mocking strategies, test coverage, TDD principles, and comprehensive banking examples testing transaction validators, account operations, and business logic.

---

## 🎯 What You'll Learn
- **Jest Basics**: Setup, test structure, matchers, async testing
- **Mocking**: Functions, modules, timers, external dependencies
- **Spies**: Tracking function calls, arguments, return values
- **Test Coverage**: Code coverage reports, 80%+ coverage targets
- **TDD**: Test-Driven Development workflow
- **Banking Examples**: Transaction validators, account balance calculations, payment processing

---

## 📖 Comprehensive Explanation

### What is Unit Testing?

**Unit testing** tests individual functions or classes in isolation, without dependencies on external systems (databases, APIs, file systems).

**Benefits**:
- ✅ Catch bugs early (before integration)
- ✅ Document expected behavior
- ✅ Enable refactoring with confidence
- ✅ Fast execution (no I/O operations)
- ✅ Prevent regressions

**Unit Test Characteristics**:
- **Fast**: Run in milliseconds
- **Isolated**: No external dependencies
- **Repeatable**: Same result every time
- **Self-validating**: Pass or fail automatically
- **Timely**: Written before or with code (TDD)

---

## 🧪 Jest Testing Framework

### Why Jest?

**Jest** is the most popular JavaScript testing framework:
- Zero configuration
- Built-in mocking and spies
- Code coverage included
- Snapshot testing
- Parallel test execution
- Great error messages

### Jest Test Structure

```javascript
// Basic test structure
describe('Feature Name', () => {
  // Setup
  beforeAll(() => {
    // Runs once before all tests
  });

  beforeEach(() => {
    // Runs before each test
  });

  afterEach(() => {
    // Runs after each test
  });

  afterAll(() => {
    // Runs once after all tests
  });

  test('should do something', () => {
    // Arrange
    const input = 'data';

    // Act
    const result = functionUnderTest(input);

    // Assert
    expect(result).toBe('expected');
  });

  it('alternative syntax for test', () => {
    // Same as test()
  });
});
```

### Common Jest Matchers

```javascript
// Equality
expect(value).toBe(4);                    // Strict equality (===)
expect(value).toEqual({ a: 1, b: 2 });    // Deep equality
expect(value).not.toBe(5);                // Negation

// Truthiness
expect(value).toBeTruthy();               // Boolean true
expect(value).toBeFalsy();                // Boolean false
expect(value).toBeNull();                 // null
expect(value).toBeUndefined();            // undefined
expect(value).toBeDefined();              // Not undefined

// Numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeLessThan(5);
expect(value).toBeCloseTo(0.3);           // Floating point

// Strings
expect('team').toMatch(/tea/);
expect('team').toContain('ea');

// Arrays and iterables
expect(['a', 'b', 'c']).toContain('b');
expect(arr).toHaveLength(3);

// Objects
expect(obj).toHaveProperty('key', 'value');
expect(obj).toMatchObject({ a: 1 });      // Partial match

// Exceptions
expect(() => { throw new Error('oops'); }).toThrow();
expect(() => func()).toThrow('error message');

// Async
await expect(promise).resolves.toBe('value');
await expect(promise).rejects.toThrow();
```

---

## 🎭 Mocking and Spies

### Mock Functions

```javascript
// Create mock function
const mockFn = jest.fn();

// Mock implementation
mockFn.mockImplementation((x) => x * 2);
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue('async result');
mockFn.mockRejectedValue(new Error('async error'));

// Call mock
mockFn(21); // Returns 84

// Assert calls
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(1);
expect(mockFn).toHaveBeenCalledWith(21);
expect(mockFn).toHaveBeenLastCalledWith(21);

// Access call data
const calls = mockFn.mock.calls;           // [[21]]
const results = mockFn.mock.results;       // [{ value: 84 }]
```

### Spy on Methods

```javascript
const obj = {
  method: (x) => x * 2
};

// Spy on existing method
const spy = jest.spyOn(obj, 'method');

obj.method(5); // Still works normally

expect(spy).toHaveBeenCalledWith(5);

// Restore original
spy.mockRestore();
```

### Mock Modules

```javascript
// Mock entire module
jest.mock('./database');
const db = require('./database');

db.query.mockResolvedValue({ rows: [] });

// Mock specific functions
jest.mock('./utils', () => ({
  formatCurrency: jest.fn((amount) => `$${amount}`),
  validateEmail: jest.fn(() => true)
}));
```

---

## 📝 Example 1: Banking Transaction Validator Unit Tests

### Transaction Validator Implementation

```javascript
// src/validators/transactionValidator.js

class TransactionValidator {
  /**
   * Validate transfer amount
   */
  validateAmount(amount) {
    if (typeof amount !== 'number') {
      throw new Error('Amount must be a number');
    }

    if (isNaN(amount)) {
      throw new Error('Amount cannot be NaN');
    }

    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }

    if (amount > 1000000) {
      throw new Error('Amount exceeds maximum limit of $1,000,000');
    }

    // Check decimal places (max 2 for currency)
    const decimalPlaces = (amount.toString().split('.')[1] || '').length;
    if (decimalPlaces > 2) {
      throw new Error('Amount cannot have more than 2 decimal places');
    }

    return true;
  }

  /**
   * Validate account number format
   */
  validateAccountNumber(accountNumber) {
    if (typeof accountNumber !== 'string') {
      throw new Error('Account number must be a string');
    }

    const pattern = /^ACC-\d{9}$/;
    if (!pattern.test(accountNumber)) {
      throw new Error('Invalid account number format. Expected: ACC-XXXXXXXXX');
    }

    return true;
  }

  /**
   * Validate transaction description
   */
  validateDescription(description) {
    if (description && typeof description !== 'string') {
      throw new Error('Description must be a string');
    }

    if (description && description.length > 200) {
      throw new Error('Description cannot exceed 200 characters');
    }

    return true;
  }

  /**
   * Validate complete transaction
   */
  validateTransaction({ fromAccount, toAccount, amount, description }) {
    const errors = [];

    try {
      this.validateAccountNumber(fromAccount);
    } catch (error) {
      errors.push(`From account: ${error.message}`);
    }

    try {
      this.validateAccountNumber(toAccount);
    } catch (error) {
      errors.push(`To account: ${error.message}`);
    }

    if (fromAccount === toAccount) {
      errors.push('Cannot transfer to the same account');
    }

    try {
      this.validateAmount(amount);
    } catch (error) {
      errors.push(`Amount: ${error.message}`);
    }

    try {
      this.validateDescription(description);
    } catch (error) {
      errors.push(`Description: ${error.message}`);
    }

    if (errors.length > 0) {
      throw new Error(`Validation failed:\n${errors.join('\n')}`);
    }

    return true;
  }
}

module.exports = TransactionValidator;
```

### Complete Jest Test Suite

```javascript
// __tests__/validators/transactionValidator.test.js

const TransactionValidator = require('../../src/validators/transactionValidator');

describe('TransactionValidator', () => {
  let validator;

  beforeEach(() => {
    validator = new TransactionValidator();
  });

  describe('validateAmount', () => {
    test('should accept valid positive amounts', () => {
      expect(validator.validateAmount(100)).toBe(true);
      expect(validator.validateAmount(0.01)).toBe(true);
      expect(validator.validateAmount(999999.99)).toBe(true);
    });

    test('should accept amounts with up to 2 decimal places', () => {
      expect(validator.validateAmount(100.50)).toBe(true);
      expect(validator.validateAmount(100.99)).toBe(true);
    });

    test('should reject non-number types', () => {
      expect(() => validator.validateAmount('100')).toThrow('Amount must be a number');
      expect(() => validator.validateAmount(null)).toThrow('Amount must be a number');
      expect(() => validator.validateAmount(undefined)).toThrow('Amount must be a number');
      expect(() => validator.validateAmount({})).toThrow('Amount must be a number');
    });

    test('should reject NaN', () => {
      expect(() => validator.validateAmount(NaN)).toThrow('Amount cannot be NaN');
    });

    test('should reject zero and negative amounts', () => {
      expect(() => validator.validateAmount(0)).toThrow('Amount must be positive');
      expect(() => validator.validateAmount(-10)).toThrow('Amount must be positive');
      expect(() => validator.validateAmount(-0.01)).toThrow('Amount must be positive');
    });

    test('should reject amounts exceeding maximum limit', () => {
      expect(() => validator.validateAmount(1000001)).toThrow('Amount exceeds maximum limit of $1,000,000');
      expect(() => validator.validateAmount(9999999)).toThrow('Amount exceeds maximum limit');
    });

    test('should reject amounts with more than 2 decimal places', () => {
      expect(() => validator.validateAmount(100.123)).toThrow('Amount cannot have more than 2 decimal places');
      expect(() => validator.validateAmount(0.001)).toThrow('Amount cannot have more than 2 decimal places');
    });
  });

  describe('validateAccountNumber', () => {
    test('should accept valid account number format', () => {
      expect(validator.validateAccountNumber('ACC-123456789')).toBe(true);
      expect(validator.validateAccountNumber('ACC-000000000')).toBe(true);
      expect(validator.validateAccountNumber('ACC-999999999')).toBe(true);
    });

    test('should reject non-string types', () => {
      expect(() => validator.validateAccountNumber(123456789)).toThrow('Account number must be a string');
      expect(() => validator.validateAccountNumber(null)).toThrow('Account number must be a string');
      expect(() => validator.validateAccountNumber({})).toThrow('Account number must be a string');
    });

    test('should reject invalid account number formats', () => {
      expect(() => validator.validateAccountNumber('123456789')).toThrow('Invalid account number format');
      expect(() => validator.validateAccountNumber('ACC-12345678')).toThrow('Invalid account number format'); // Too short
      expect(() => validator.validateAccountNumber('ACC-1234567890')).toThrow('Invalid account number format'); // Too long
      expect(() => validator.validateAccountNumber('ACC-12345678A')).toThrow('Invalid account number format'); // Letter
      expect(() => validator.validateAccountNumber('acc-123456789')).toThrow('Invalid account number format'); // Lowercase
    });
  });

  describe('validateDescription', () => {
    test('should accept valid descriptions', () => {
      expect(validator.validateDescription('Payment for services')).toBe(true);
      expect(validator.validateDescription('A'.repeat(200))).toBe(true); // Exactly 200 chars
    });

    test('should accept empty or undefined descriptions', () => {
      expect(validator.validateDescription('')).toBe(true);
      expect(validator.validateDescription(undefined)).toBe(true);
      expect(validator.validateDescription(null)).toBe(true);
    });

    test('should reject non-string descriptions', () => {
      expect(() => validator.validateDescription(123)).toThrow('Description must be a string');
      expect(() => validator.validateDescription({})).toThrow('Description must be a string');
    });

    test('should reject descriptions exceeding 200 characters', () => {
      const longDescription = 'A'.repeat(201);
      expect(() => validator.validateDescription(longDescription)).toThrow('Description cannot exceed 200 characters');
    });
  });

  describe('validateTransaction', () => {
    const validTransaction = {
      fromAccount: 'ACC-111111111',
      toAccount: 'ACC-222222222',
      amount: 500.50,
      description: 'Test payment'
    };

    test('should accept valid transaction', () => {
      expect(validator.validateTransaction(validTransaction)).toBe(true);
    });

    test('should reject transaction with same from and to accounts', () => {
      const transaction = {
        ...validTransaction,
        toAccount: 'ACC-111111111' // Same as fromAccount
      };

      expect(() => validator.validateTransaction(transaction)).toThrow('Cannot transfer to the same account');
    });

    test('should collect multiple validation errors', () => {
      const invalidTransaction = {
        fromAccount: 'INVALID',
        toAccount: 'INVALID',
        amount: -100,
        description: 'A'.repeat(300)
      };

      expect(() => validator.validateTransaction(invalidTransaction)).toThrow('Validation failed');
      
      try {
        validator.validateTransaction(invalidTransaction);
      } catch (error) {
        expect(error.message).toContain('From account');
        expect(error.message).toContain('To account');
        expect(error.message).toContain('Amount');
        expect(error.message).toContain('Description');
      }
    });

    test('should handle missing fields gracefully', () => {
      const incompleteTransaction = {
        fromAccount: 'ACC-111111111'
        // Missing toAccount, amount, description
      };

      expect(() => validator.validateTransaction(incompleteTransaction)).toThrow();
    });
  });
});
```

### Running Tests

```bash
# Install Jest
npm install --save-dev jest

# Add to package.json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}

# Run tests
npm test

# Run with coverage
npm run test:coverage

# Watch mode (re-run on file changes)
npm run test:watch
```

### Expected Output

```
 PASS  __tests__/validators/transactionValidator.test.js
  TransactionValidator
    validateAmount
      ✓ should accept valid positive amounts (3 ms)
      ✓ should accept amounts with up to 2 decimal places (1 ms)
      ✓ should reject non-number types (2 ms)
      ✓ should reject NaN (1 ms)
      ✓ should reject zero and negative amounts (1 ms)
      ✓ should reject amounts exceeding maximum limit (1 ms)
      ✓ should reject amounts with more than 2 decimal places (1 ms)
    validateAccountNumber
      ✓ should accept valid account number format (1 ms)
      ✓ should reject non-string types (1 ms)
      ✓ should reject invalid account number formats (2 ms)
    validateDescription
      ✓ should accept valid descriptions (1 ms)
      ✓ should accept empty or undefined descriptions (1 ms)
      ✓ should reject non-string descriptions (1 ms)
      ✓ should reject descriptions exceeding 200 characters (1 ms)
    validateTransaction
      ✓ should accept valid transaction (1 ms)
      ✓ should reject transaction with same from and to accounts (1 ms)
      ✓ should collect multiple validation errors (2 ms)
      ✓ should handle missing fields gracefully (1 ms)

Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Time:        0.5 s

Coverage:
  TransactionValidator | 100% | 100% | 100% | 100%
```

---

## 📝 Example 2: Account Service with Mocking

### Account Service Implementation

```javascript
// src/services/accountService.js

class AccountService {
  constructor(database, cacheService, notificationService) {
    this.db = database;
    this.cache = cacheService;
    this.notifications = notificationService;
  }

  async getBalance(accountId) {
    // Try cache first
    const cached = await this.cache.get(`balance:${accountId}`);
    if (cached) {
      return JSON.parse(cached);
    }

    // Query database
    const result = await this.db.query(
      'SELECT balance FROM accounts WHERE account_id = $1',
      [accountId]
    );

    if (result.rows.length === 0) {
      throw new Error(`Account ${accountId} not found`);
    }

    const balance = parseFloat(result.rows[0].balance);

    // Cache result
    await this.cache.set(`balance:${accountId}`, JSON.stringify(balance), 300);

    return balance;
  }

  async updateBalance(accountId, newBalance) {
    // Validate
    if (newBalance < 0) {
      throw new Error('Balance cannot be negative');
    }

    // Update database
    const result = await this.db.query(
      'UPDATE accounts SET balance = $1 WHERE account_id = $2 RETURNING balance',
      [newBalance, accountId]
    );

    if (result.rows.length === 0) {
      throw new Error(`Account ${accountId} not found`);
    }

    // Invalidate cache
    await this.cache.del(`balance:${accountId}`);

    // Send notification
    await this.notifications.send(accountId, {
      type: 'balance_updated',
      newBalance,
      timestamp: new Date()
    });

    return parseFloat(result.rows[0].balance);
  }

  calculateInterest(principal, rate, years) {
    // Simple interest: P * r * t
    return principal * rate * years;
  }
}

module.exports = AccountService;
```

### Complete Test Suite with Mocking

```javascript
// __tests__/services/accountService.test.js

const AccountService = require('../../src/services/accountService');

// Mock dependencies
const mockDatabase = {
  query: jest.fn()
};

const mockCache = {
  get: jest.fn(),
  set: jest.fn(),
  del: jest.fn()
};

const mockNotifications = {
  send: jest.fn()
};

describe('AccountService', () => {
  let accountService;

  beforeEach(() => {
    // Create fresh instance with mocks
    accountService = new AccountService(mockDatabase, mockCache, mockNotifications);

    // Clear all mocks
    jest.clearAllMocks();
  });

  describe('getBalance', () => {
    const accountId = 'ACC-123456789';

    test('should return cached balance if available', async () => {
      // Arrange
      mockCache.get.mockResolvedValue(JSON.stringify(1000));

      // Act
      const balance = await accountService.getBalance(accountId);

      // Assert
      expect(balance).toBe(1000);
      expect(mockCache.get).toHaveBeenCalledWith(`balance:${accountId}`);
      expect(mockDatabase.query).not.toHaveBeenCalled(); // DB not queried
    });

    test('should query database if cache miss', async () => {
      // Arrange
      mockCache.get.mockResolvedValue(null); // Cache miss
      mockDatabase.query.mockResolvedValue({
        rows: [{ balance: '2500.50' }]
      });

      // Act
      const balance = await accountService.getBalance(accountId);

      // Assert
      expect(balance).toBe(2500.50);
      expect(mockCache.get).toHaveBeenCalledWith(`balance:${accountId}`);
      expect(mockDatabase.query).toHaveBeenCalledWith(
        'SELECT balance FROM accounts WHERE account_id = $1',
        [accountId]
      );
    });

    test('should cache database result', async () => {
      // Arrange
      mockCache.get.mockResolvedValue(null);
      mockDatabase.query.mockResolvedValue({
        rows: [{ balance: '3000' }]
      });

      // Act
      await accountService.getBalance(accountId);

      // Assert
      expect(mockCache.set).toHaveBeenCalledWith(
        `balance:${accountId}`,
        JSON.stringify(3000),
        300
      );
    });

    test('should throw error if account not found', async () => {
      // Arrange
      mockCache.get.mockResolvedValue(null);
      mockDatabase.query.mockResolvedValue({ rows: [] }); // No results

      // Act & Assert
      await expect(accountService.getBalance(accountId))
        .rejects
        .toThrow(`Account ${accountId} not found`);
    });

    test('should handle database errors', async () => {
      // Arrange
      mockCache.get.mockResolvedValue(null);
      mockDatabase.query.mockRejectedValue(new Error('Database connection failed'));

      // Act & Assert
      await expect(accountService.getBalance(accountId))
        .rejects
        .toThrow('Database connection failed');
    });
  });

  describe('updateBalance', () => {
    const accountId = 'ACC-123456789';

    test('should update balance successfully', async () => {
      // Arrange
      const newBalance = 5000;
      mockDatabase.query.mockResolvedValue({
        rows: [{ balance: '5000' }]
      });

      // Act
      const result = await accountService.updateBalance(accountId, newBalance);

      // Assert
      expect(result).toBe(5000);
      expect(mockDatabase.query).toHaveBeenCalledWith(
        'UPDATE accounts SET balance = $1 WHERE account_id = $2 RETURNING balance',
        [newBalance, accountId]
      );
    });

    test('should invalidate cache after update', async () => {
      // Arrange
      mockDatabase.query.mockResolvedValue({
        rows: [{ balance: '5000' }]
      });

      // Act
      await accountService.updateBalance(accountId, 5000);

      // Assert
      expect(mockCache.del).toHaveBeenCalledWith(`balance:${accountId}`);
    });

    test('should send notification after update', async () => {
      // Arrange
      const newBalance = 5000;
      mockDatabase.query.mockResolvedValue({
        rows: [{ balance: '5000' }]
      });

      // Act
      await accountService.updateBalance(accountId, newBalance);

      // Assert
      expect(mockNotifications.send).toHaveBeenCalledWith(
        accountId,
        expect.objectContaining({
          type: 'balance_updated',
          newBalance: 5000,
          timestamp: expect.any(Date)
        })
      );
    });

    test('should reject negative balance', async () => {
      // Act & Assert
      await expect(accountService.updateBalance(accountId, -100))
        .rejects
        .toThrow('Balance cannot be negative');

      // Ensure no database call made
      expect(mockDatabase.query).not.toHaveBeenCalled();
    });

    test('should throw error if account not found', async () => {
      // Arrange
      mockDatabase.query.mockResolvedValue({ rows: [] });

      // Act & Assert
      await expect(accountService.updateBalance(accountId, 1000))
        .rejects
        .toThrow(`Account ${accountId} not found`);
    });
  });

  describe('calculateInterest', () => {
    test('should calculate simple interest correctly', () => {
      expect(accountService.calculateInterest(1000, 0.05, 1)).toBe(50);
      expect(accountService.calculateInterest(5000, 0.03, 2)).toBe(300);
      expect(accountService.calculateInterest(10000, 0.04, 5)).toBe(2000);
    });

    test('should handle zero principal', () => {
      expect(accountService.calculateInterest(0, 0.05, 1)).toBe(0);
    });

    test('should handle zero rate', () => {
      expect(accountService.calculateInterest(1000, 0, 1)).toBe(0);
    });

    test('should handle zero years', () => {
      expect(accountService.calculateInterest(1000, 0.05, 0)).toBe(0);
    });
  });
});
```

### Test Coverage Report

```bash
npm run test:coverage
```

```
-------------------|---------|----------|---------|---------|-------------------
File               | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
-------------------|---------|----------|---------|---------|-------------------
All files          |   98.5  |   95.2   |  100    |   98.5  |
accountService.js  |   98.5  |   95.2   |  100    |   98.5  | 45
-------------------|---------|----------|---------|---------|-------------------
```

---

## 🎯 Key Takeaways

### Unit Testing Best Practices

1. **Test one thing per test**
   ```javascript
   // ❌ Bad: Testing multiple things
   test('should validate everything', () => {
     expect(validateAmount(100)).toBe(true);
     expect(validateAccount('ACC-123')).toBe(true);
     expect(processPayment()).toBe('success');
   });
   
   // ✅ Good: One test per concept
   test('should accept valid amounts', () => {
     expect(validateAmount(100)).toBe(true);
   });
   ```

2. **Use descriptive test names**
   ```javascript
   // ❌ Bad
   test('test1', () => {});
   
   // ✅ Good
   test('should reject negative transfer amounts', () => {});
   ```

3. **Follow AAA pattern** (Arrange, Act, Assert)
   ```javascript
   test('should calculate fee correctly', () => {
     // Arrange
     const amount = 1000;
     
     // Act
     const fee = calculateFee(amount);
     
     // Assert
     expect(fee).toBe(10);
   });
   ```

4. **Mock external dependencies**
   - Database calls
   - API requests
   - File system
   - Time/dates
   - Random numbers

5. **Test edge cases**
   - Empty inputs
   - Null/undefined
   - Boundary values
   - Error conditions

### Coverage Targets

- **Statements**: 80%+
- **Branches**: 75%+
- **Functions**: 80%+
- **Lines**: 80%+

**Priority**: Focus on business-critical code (payment processing, validation)

### TDD Workflow

1. **Red**: Write failing test
2. **Green**: Write minimum code to pass
3. **Refactor**: Improve code while keeping tests green

```javascript
// 1. Red: Write test first
test('should calculate discount', () => {
  expect(calculateDiscount(100, 10)).toBe(90);
});

// 2. Green: Implement function
function calculateDiscount(amount, percentage) {
  return amount * (1 - percentage / 100);
}

// 3. Refactor: Improve code
function calculateDiscount(amount, percentage) {
  validateAmount(amount);
  validatePercentage(percentage);
  return parseFloat((amount * (1 - percentage / 100)).toFixed(2));
}
```

---

**File**: `Q33_Testing_Unit_Tests.md`  
**Status**: ✅ Complete with Jest examples, mocking strategies, and banking test suites
