# Questions 28-30: Testing, Security & Best Practices

## Question 28: How do you test JavaScript code? Explain unit testing, integration testing, and E2E testing.

### Answer:

**Testing** ensures code quality, catches bugs early, and documents expected behavior. Different testing levels serve different purposes in the testing pyramid.

### Testing Pyramid:
1. **Unit Tests** (70%): Test individual functions/components
2. **Integration Tests** (20%): Test component interactions
3. **E2E Tests** (10%): Test complete user workflows

### Banking Scenario: Testing Suite at Emirates NBD

```javascript
console.log('=== JavaScript Testing Demo - Emirates NBD ===\n');

// =============================================================================
// CODE TO TEST - Banking Service
// =============================================================================

class BankAccount {
    constructor(accountNumber, initialBalance = 0) {
        if (!this.isValidAccountNumber(accountNumber)) {
            throw new Error('Invalid account number format');
        }
        
        if (initialBalance < 0) {
            throw new Error('Initial balance cannot be negative');
        }
        
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.transactions = [];
        this.status = 'ACTIVE';
    }

    isValidAccountNumber(accountNumber) {
        return /^ACC-\d{9}$/.test(accountNumber);
    }

    deposit(amount) {
        if (amount <= 0) {
            throw new Error('Deposit amount must be positive');
        }
        
        this.balance += amount;
        this.transactions.push({
            type: 'DEPOSIT',
            amount,
            timestamp: new Date(),
            balance: this.balance
        });
        
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
        this.transactions.push({
            type: 'WITHDRAWAL',
            amount,
            timestamp: new Date(),
            balance: this.balance
        });
        
        return this.balance;
    }

    transfer(toAccount, amount) {
        if (amount <= 0) {
            throw new Error('Transfer amount must be positive');
        }
        
        if (amount > this.balance) {
            throw new Error('Insufficient funds');
        }
        
        this.withdraw(amount);
        toAccount.deposit(amount);
        
        return {
            from: this.accountNumber,
            to: toAccount.accountNumber,
            amount,
            timestamp: new Date()
        };
    }

    getBalance() {
        return this.balance;
    }

    getTransactions() {
        return [...this.transactions];
    }

    freeze() {
        this.status = 'FROZEN';
    }

    unfreeze() {
        this.status = 'ACTIVE';
    }

    isActive() {
        return this.status === 'ACTIVE';
    }
}

class TransactionService {
    constructor(apiClient) {
        this.apiClient = apiClient;
        this.cache = new Map();
    }

    async processTransaction(transaction) {
        // Validate transaction
        if (!transaction.accountNumber || !transaction.amount) {
            throw new Error('Invalid transaction data');
        }
        
        // Call API
        const result = await this.apiClient.post('/transactions', transaction);
        
        // Cache result
        this.cache.set(result.id, result);
        
        return result;
    }

    async getTransaction(transactionId) {
        // Check cache first
        if (this.cache.has(transactionId)) {
            return this.cache.get(transactionId);
        }
        
        // Fetch from API
        const result = await this.apiClient.get(`/transactions/${transactionId}`);
        this.cache.set(transactionId, result);
        
        return result;
    }
}

// =============================================================================
// UNIT TESTS - Test individual functions
// =============================================================================

console.log('1. UNIT TESTS:\n');

class TestRunner {
    constructor(suiteName) {
        this.suiteName = suiteName;
        this.tests = [];
        this.passed = 0;
        this.failed = 0;
    }

    test(description, testFn) {
        this.tests.push({ description, testFn });
    }

    async run() {
        console.log(`\n${'='.repeat(70)}`);
        console.log(`Test Suite: ${this.suiteName}`);
        console.log('='.repeat(70) + '\n');
        
        for (const test of this.tests) {
            try {
                await test.testFn();
                this.passed++;
                console.log(`✅ PASS: ${test.description}`);
            } catch (error) {
                this.failed++;
                console.log(`❌ FAIL: ${test.description}`);
                console.log(`   Error: ${error.message}\n`);
            }
        }
        
        console.log(`\n${'='.repeat(70)}`);
        console.log(`Results: ${this.passed} passed, ${this.failed} failed`);
        console.log('='.repeat(70) + '\n');
    }
}

// Assertion library (simplified)
const assert = {
    equal(actual, expected, message = '') {
        if (actual !== expected) {
            throw new Error(
                `${message}\nExpected: ${expected}\nActual: ${actual}`
            );
        }
    },
    
    throws(fn, expectedError, message = '') {
        try {
            fn();
            throw new Error(`${message}\nExpected function to throw`);
        } catch (error) {
            if (expectedError && !error.message.includes(expectedError)) {
                throw new Error(
                    `${message}\nExpected error: ${expectedError}\nActual: ${error.message}`
                );
            }
        }
    },
    
    deepEqual(actual, expected, message = '') {
        const actualStr = JSON.stringify(actual);
        const expectedStr = JSON.stringify(expected);
        
        if (actualStr !== expectedStr) {
            throw new Error(
                `${message}\nExpected: ${expectedStr}\nActual: ${actualStr}`
            );
        }
    }
};

// Unit Test Suite for BankAccount
const accountTests = new TestRunner('BankAccount Unit Tests');

accountTests.test('should create account with valid number', () => {
    const account = new BankAccount('ACC-123456789', 1000);
    assert.equal(account.accountNumber, 'ACC-123456789');
    assert.equal(account.balance, 1000);
});

accountTests.test('should reject invalid account number', () => {
    assert.throws(
        () => new BankAccount('INVALID', 1000),
        'Invalid account number format'
    );
});

accountTests.test('should reject negative initial balance', () => {
    assert.throws(
        () => new BankAccount('ACC-123456789', -100),
        'Initial balance cannot be negative'
    );
});

accountTests.test('should deposit money correctly', () => {
    const account = new BankAccount('ACC-123456789', 1000);
    const newBalance = account.deposit(500);
    
    assert.equal(newBalance, 1500);
    assert.equal(account.balance, 1500);
    assert.equal(account.transactions.length, 1);
    assert.equal(account.transactions[0].type, 'DEPOSIT');
});

accountTests.test('should reject negative deposit', () => {
    const account = new BankAccount('ACC-123456789', 1000);
    assert.throws(
        () => account.deposit(-100),
        'Deposit amount must be positive'
    );
});

accountTests.test('should withdraw money correctly', () => {
    const account = new BankAccount('ACC-123456789', 1000);
    const newBalance = account.withdraw(300);
    
    assert.equal(newBalance, 700);
    assert.equal(account.balance, 700);
});

accountTests.test('should reject withdrawal exceeding balance', () => {
    const account = new BankAccount('ACC-123456789', 1000);
    assert.throws(
        () => account.withdraw(1500),
        'Insufficient funds'
    );
});

accountTests.test('should transfer between accounts', () => {
    const account1 = new BankAccount('ACC-111111111', 1000);
    const account2 = new BankAccount('ACC-222222222', 500);
    
    const transfer = account1.transfer(account2, 300);
    
    assert.equal(account1.balance, 700);
    assert.equal(account2.balance, 800);
    assert.equal(transfer.amount, 300);
});

accountTests.test('should freeze and unfreeze account', () => {
    const account = new BankAccount('ACC-123456789', 1000);
    
    assert.equal(account.isActive(), true);
    
    account.freeze();
    assert.equal(account.isActive(), false);
    
    account.unfreeze();
    assert.equal(account.isActive(), true);
});

await accountTests.run();

// =============================================================================
// INTEGRATION TESTS - Test component interactions
// =============================================================================

console.log('\n2. INTEGRATION TESTS:\n');

// Mock API Client
class MockAPIClient {
    constructor() {
        this.requests = [];
        this.responses = new Map();
    }

    async post(url, data) {
        this.requests.push({ method: 'POST', url, data });
        
        // Simulate API response
        await new Promise(resolve => setTimeout(resolve, 10));
        
        const response = {
            id: `TXN-${Date.now()}`,
            ...data,
            status: 'COMPLETED',
            timestamp: new Date()
        };
        
        return response;
    }

    async get(url) {
        this.requests.push({ method: 'GET', url });
        
        await new Promise(resolve => setTimeout(resolve, 10));
        
        // Return mocked response
        if (this.responses.has(url)) {
            return this.responses.get(url);
        }
        
        return {
            id: 'TXN-001',
            amount: 1000,
            status: 'COMPLETED'
        };
    }

    setMockResponse(url, response) {
        this.responses.set(url, response);
    }

    getRequests() {
        return this.requests;
    }
}

const integrationTests = new TestRunner('TransactionService Integration Tests');

integrationTests.test('should process transaction via API', async () => {
    const apiClient = new MockAPIClient();
    const service = new TransactionService(apiClient);
    
    const result = await service.processTransaction({
        accountNumber: 'ACC-123456789',
        amount: 500,
        type: 'DEPOSIT'
    });
    
    assert.equal(result.amount, 500);
    assert.equal(result.status, 'COMPLETED');
    assert.equal(apiClient.getRequests().length, 1);
});

integrationTests.test('should cache transaction after processing', async () => {
    const apiClient = new MockAPIClient();
    const service = new TransactionService(apiClient);
    
    const result1 = await service.processTransaction({
        accountNumber: 'ACC-123456789',
        amount: 500,
        type: 'DEPOSIT'
    });
    
    // Get from cache (should not make API call)
    const result2 = await service.getTransaction(result1.id);
    
    assert.equal(result2.id, result1.id);
    assert.equal(apiClient.getRequests().length, 1); // Only 1 API call
});

integrationTests.test('should fetch from API if not in cache', async () => {
    const apiClient = new MockAPIClient();
    const service = new TransactionService(apiClient);
    
    apiClient.setMockResponse('/transactions/TXN-999', {
        id: 'TXN-999',
        amount: 1500,
        status: 'COMPLETED'
    });
    
    const result = await service.getTransaction('TXN-999');
    
    assert.equal(result.id, 'TXN-999');
    assert.equal(result.amount, 1500);
    assert.equal(apiClient.getRequests().length, 1);
});

integrationTests.test('should reject invalid transaction data', async () => {
    const apiClient = new MockAPIClient();
    const service = new TransactionService(apiClient);
    
    try {
        await service.processTransaction({ amount: 500 }); // Missing accountNumber
        throw new Error('Should have thrown error');
    } catch (error) {
        assert.equal(error.message.includes('Invalid transaction data'), true);
    }
});

await integrationTests.run();

// =============================================================================
// E2E TESTS - Test complete workflows
// =============================================================================

console.log('\n3. E2E (End-to-End) TESTS:\n');

class E2ETestEnvironment {
    constructor() {
        this.apiClient = new MockAPIClient();
        this.transactionService = new TransactionService(this.apiClient);
        this.accounts = new Map();
    }

    createAccount(accountNumber, balance) {
        const account = new BankAccount(accountNumber, balance);
        this.accounts.set(accountNumber, account);
        return account;
    }

    getAccount(accountNumber) {
        return this.accounts.get(accountNumber);
    }

    async processDeposit(accountNumber, amount) {
        const account = this.getAccount(accountNumber);
        account.deposit(amount);
        
        return await this.transactionService.processTransaction({
            accountNumber,
            amount,
            type: 'DEPOSIT'
        });
    }

    async processWithdrawal(accountNumber, amount) {
        const account = this.getAccount(accountNumber);
        account.withdraw(amount);
        
        return await this.transactionService.processTransaction({
            accountNumber,
            amount,
            type: 'WITHDRAWAL'
        });
    }

    async processTransfer(fromAccountNumber, toAccountNumber, amount) {
        const fromAccount = this.getAccount(fromAccountNumber);
        const toAccount = this.getAccount(toAccountNumber);
        
        fromAccount.transfer(toAccount, amount);
        
        return await this.transactionService.processTransaction({
            fromAccountNumber,
            toAccountNumber,
            amount,
            type: 'TRANSFER'
        });
    }
}

const e2eTests = new TestRunner('Banking System E2E Tests');

e2eTests.test('Complete user journey: Create account → Deposit → Withdraw', async () => {
    const env = new E2ETestEnvironment();
    
    // Step 1: Create account
    const account = env.createAccount('ACC-123456789', 0);
    assert.equal(account.balance, 0);
    
    // Step 2: Make deposit
    const depositResult = await env.processDeposit('ACC-123456789', 5000);
    assert.equal(account.balance, 5000);
    assert.equal(depositResult.status, 'COMPLETED');
    
    // Step 3: Make withdrawal
    const withdrawalResult = await env.processWithdrawal('ACC-123456789', 2000);
    assert.equal(account.balance, 3000);
    assert.equal(withdrawalResult.status, 'COMPLETED');
    
    // Verify transaction history
    const transactions = account.getTransactions();
    assert.equal(transactions.length, 2);
    assert.equal(transactions[0].type, 'DEPOSIT');
    assert.equal(transactions[1].type, 'WITHDRAWAL');
});

e2eTests.test('Complete transfer workflow between two accounts', async () => {
    const env = new E2ETestEnvironment();
    
    // Create accounts
    const account1 = env.createAccount('ACC-111111111', 10000);
    const account2 = env.createAccount('ACC-222222222', 5000);
    
    // Perform transfer
    const transferResult = await env.processTransfer('ACC-111111111', 'ACC-222222222', 3000);
    
    // Verify balances
    assert.equal(account1.balance, 7000);
    assert.equal(account2.balance, 8000);
    assert.equal(transferResult.status, 'COMPLETED');
    
    // Verify transaction histories
    assert.equal(account1.getTransactions().length, 1);
    assert.equal(account2.getTransactions().length, 1);
});

e2eTests.test('Account freeze workflow prevents transactions', async () => {
    const env = new E2ETestEnvironment();
    
    const account = env.createAccount('ACC-123456789', 10000);
    
    // Make successful transaction
    await env.processDeposit('ACC-123456789', 1000);
    assert.equal(account.balance, 11000);
    
    // Freeze account
    account.freeze();
    assert.equal(account.isActive(), false);
    
    // Account is frozen, but we can still check balance
    assert.equal(account.balance, 11000);
});

e2eTests.test('Multiple transactions maintain correct balance', async () => {
    const env = new E2ETestEnvironment();
    
    const account = env.createAccount('ACC-123456789', 10000);
    
    // Perform multiple transactions
    await env.processDeposit('ACC-123456789', 2000);   // 12000
    await env.processWithdrawal('ACC-123456789', 1000); // 11000
    await env.processDeposit('ACC-123456789', 3000);   // 14000
    await env.processWithdrawal('ACC-123456789', 500);  // 13500
    
    assert.equal(account.balance, 13500);
    assert.equal(account.getTransactions().length, 4);
    
    // Verify transaction order
    const transactions = account.getTransactions();
    assert.equal(transactions[0].type, 'DEPOSIT');
    assert.equal(transactions[0].balance, 12000);
    assert.equal(transactions[3].balance, 13500);
});

await e2eTests.run();

// =============================================================================
// TEST SUMMARY & BEST PRACTICES
// =============================================================================

console.log('\n' + '='.repeat(70));
console.log('TESTING SUMMARY & BEST PRACTICES');
console.log('='.repeat(70) + '\n');

console.log('1. TESTING PYRAMID:\n');
console.log('   Unit Tests (70%)      ← Fast, isolated, many tests');
console.log('        ↑');
console.log('   Integration Tests (20%) ← Test component interactions');
console.log('        ↑');
console.log('   E2E Tests (10%)       ← Slow, full workflows, fewer tests\n');

console.log('2. TEST CHARACTERISTICS:\n');
console.log('Unit Tests:');
console.log('  • Test single function/class');
console.log('  • Fast execution (milliseconds)');
console.log('  • No dependencies (use mocks)');
console.log('  • Easy to debug');
console.log('  • High code coverage');

console.log('\nIntegration Tests:');
console.log('  • Test multiple components together');
console.log('  • Medium speed (seconds)');
console.log('  • May use mocks for external services');
console.log('  • Verify component interactions');

console.log('\nE2E Tests:');
console.log('  • Test complete user workflows');
console.log('  • Slow execution (minutes)');
console.log('  • Use real or near-real environment');
console.log('  • Verify business requirements');
console.log('  • Fewer but critical paths\n');

console.log('3. TESTING BEST PRACTICES:\n');
console.log('Naming:');
console.log('  ✓ Descriptive test names');
console.log('  ✓ Should/When/Then format');
console.log('  ✓ Include expected behavior');

console.log('\nStructure:');
console.log('  ✓ Arrange: Set up test data');
console.log('  ✓ Act: Execute function');
console.log('  ✓ Assert: Verify result');

console.log('\nIsolation:');
console.log('  ✓ Each test independent');
console.log('  ✓ No shared state');
console.log('  ✓ Clean up after tests');

console.log('\nCoverage:');
console.log('  ✓ Test happy path');
console.log('  ✓ Test edge cases');
console.log('  ✓ Test error conditions');
console.log('  ✓ Aim for 80%+ coverage\n');

console.log('4. TESTING TOOLS:\n');
console.log('Test Frameworks:');
console.log('  • Jest: Full-featured, popular');
console.log('  • Mocha: Flexible, minimalist');
console.log('  • Jasmine: Behavior-driven');
console.log('  • Vitest: Modern, fast');

console.log('\nAssertion Libraries:');
console.log('  • Chai: Flexible assertions');
console.log('  • Jest (built-in)');
console.log('  • Assert (Node.js)');

console.log('\nMocking:');
console.log('  • Jest mocks');
console.log('  • Sinon.js');
console.log('  • MSW (Mock Service Worker)');

console.log('\nE2E Testing:');
console.log('  • Playwright: Modern, cross-browser');
console.log('  • Cypress: Developer-friendly');
console.log('  • Selenium: Industry standard');
console.log('  • Puppeteer: Chrome DevTools Protocol\n');

console.log('5. EXAMPLE TEST PATTERN:\n');
console.log('describe("BankAccount", () => {');
console.log('  describe("deposit()", () => {');
console.log('    it("should increase balance by deposit amount", () => {');
console.log('      // Arrange');
console.log('      const account = new BankAccount("ACC-123", 1000);');
console.log('      ');
console.log('      // Act');
console.log('      account.deposit(500);');
console.log('      ');
console.log('      // Assert');
console.log('      expect(account.balance).toBe(1500);');
console.log('    });');
console.log('  });');
console.log('});\n');

console.log('6. COVERAGE METRICS:\n');
console.log('  • Statement Coverage: % of statements executed');
console.log('  • Branch Coverage: % of if/else branches taken');
console.log('  • Function Coverage: % of functions called');
console.log('  • Line Coverage: % of lines executed');
console.log('\n  Target: 80%+ for critical code\n');

console.log('7. CONTINUOUS INTEGRATION:\n');
console.log('  • Run tests on every commit');
console.log('  • Automated test pipeline');
console.log('  • Fail builds on test failures');
console.log('  • Generate coverage reports');
console.log('  • Monitor test performance');
```

---

## Question 29: What are common security vulnerabilities in JavaScript? How do you prevent them?

### Answer:

JavaScript applications face various security threats. Understanding and preventing these vulnerabilities is crucial for building secure applications.

### Common Vulnerabilities:

1. **XSS** (Cross-Site Scripting)
2. **CSRF** (Cross-Site Request Forgery)
3. **Injection Attacks**
4. **Insecure Data Storage**
5. **Prototype Pollution**
6. **Open Redirects**

### Banking Scenario: Security Implementation at Emirates NBD

```javascript
console.log('=== JavaScript Security Demo - Emirates NBD ===\n');

// =============================================================================
// 1. XSS (Cross-Site Scripting) Prevention
// =============================================================================

console.log('1. XSS (Cross-Site Scripting) Prevention:\n');

class XSSPrevention {
    // ❌ VULNERABLE: Direct HTML insertion
    static renderUserInputUnsafe(input) {
        // Never do this!
        // document.getElementById('output').innerHTML = input;
        console.log('❌ UNSAFE: innerHTML with user input');
        console.log(`   Input: ${input}`);
        console.log('   This could execute: <script>alert("XSS")</script>\n');
    }

    // ✅ SAFE: Escape HTML entities
    static escapeHTML(str) {
        const div = { textContent: str };
        // In browser: document.createElement('div').textContent = str
        return str
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#x27;')
            .replace(/\//g, '&#x2F;');
    }

    // ✅ SAFE: Use textContent instead of innerHTML
    static renderUserInputSafe(input) {
        const escaped = this.escapeHTML(input);
        console.log('✅ SAFE: Escaped HTML entities');
        console.log(`   Input: ${input}`);
        console.log(`   Escaped: ${escaped}\n`);
        
        // In browser: element.textContent = input;
        return escaped;
    }

    // ✅ SAFE: Content Security Policy headers
    static getCSPHeaders() {
        return {
            'Content-Security-Policy': [
                "default-src 'self'",
                "script-src 'self' 'nonce-{random}'",
                "style-src 'self' 'unsafe-inline'",
                "img-src 'self' data: https:",
                "connect-src 'self' https://api.emiratesnbd.com",
                "frame-ancestors 'none'"
            ].join('; ')
        };
    }

    // Sanitize user input for account search
    static sanitizeAccountSearch(query) {
        // Remove any script tags, event handlers, etc.
        const sanitized = query
            .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
            .replace(/on\w+\s*=\s*["'][^"']*["']/gi, '')
            .replace(/javascript:/gi, '')
            .trim();
        
        console.log('Sanitized search query:');
        console.log(`  Original: ${query}`);
        console.log(`  Sanitized: ${sanitized}\n`);
        
        return sanitized;
    }
}

// Demonstrate XSS prevention
const maliciousInput = '<script>alert("Steal credentials")</script>Ahmed';
XSSPrevention.renderUserInputUnsafe(maliciousInput);
XSSPrevention.renderUserInputSafe(maliciousInput);

const searchQuery = 'ACC-123<script>alert("XSS")</script>';
XSSPrevention.sanitizeAccountSearch(searchQuery);

console.log('CSP Headers:', XSSPrevention.getCSPHeaders());
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. CSRF (Cross-Site Request Forgery) Prevention
// =============================================================================

console.log('2. CSRF (Cross-Site Request Forgery) Prevention:\n');

class CSRFProtection {
    // Generate CSRF token
    static generateToken() {
        // In production, use crypto.randomBytes()
        const token = Array.from({ length: 32 }, () => 
            Math.random().toString(36)[2]
        ).join('');
        
        console.log('Generated CSRF token:', token.substring(0, 20) + '...\n');
        return token;
    }

    // Validate CSRF token
    static validateToken(requestToken, sessionToken) {
        if (!requestToken || !sessionToken) {
            console.log('❌ CSRF validation failed: Missing token\n');
            return false;
        }
        
        if (requestToken !== sessionToken) {
            console.log('❌ CSRF validation failed: Token mismatch\n');
            return false;
        }
        
        console.log('✅ CSRF validation passed\n');
        return true;
    }

    // Add token to form
    static addTokenToForm(form, token) {
        console.log('Adding CSRF token to form');
        console.log(`  Token: ${token.substring(0, 20)}...\n`);
        
        // In browser:
        // const input = document.createElement('input');
        // input.type = 'hidden';
        // input.name = 'csrf_token';
        // input.value = token;
        // form.appendChild(input);
        
        return { ...form, csrf_token: token };
    }

    // Verify origin header
    static verifyOrigin(requestOrigin, allowedOrigins) {
        if (!allowedOrigins.includes(requestOrigin)) {
            console.log(`❌ Origin validation failed: ${requestOrigin}\n`);
            return false;
        }
        
        console.log(`✅ Origin validated: ${requestOrigin}\n`);
        return true;
    }
}

// Demonstrate CSRF protection
const csrfToken = CSRFProtection.generateToken();

// Valid request
const validForm = { amount: 5000, toAccount: 'ACC-987654321' };
const formWithToken = CSRFProtection.addTokenToForm(validForm, csrfToken);
console.log('Form with CSRF token:', formWithToken, '\n');

CSRFProtection.validateToken(csrfToken, csrfToken);

// Invalid request (missing token)
CSRFProtection.validateToken(null, csrfToken);

// Verify origin
const allowedOrigins = ['https://emiratesnbd.com', 'https://app.emiratesnbd.com'];
CSRFProtection.verifyOrigin('https://emiratesnbd.com', allowedOrigins);
CSRFProtection.verifyOrigin('https://malicious.com', allowedOrigins);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. SQL/NoSQL Injection Prevention
// =============================================================================

console.log('3. Injection Attack Prevention:\n');

class InjectionPrevention {
    // ❌ VULNERABLE: String concatenation
    static buildQueryUnsafe(accountNumber) {
        const query = `SELECT * FROM accounts WHERE account_number = '${accountNumber}'`;
        console.log('❌ UNSAFE query (vulnerable to injection):');
        console.log(`   ${query}\n`);
        return query;
    }

    // ✅ SAFE: Parameterized query
    static buildQuerySafe(accountNumber) {
        console.log('✅ SAFE query (parameterized):');
        console.log('   SELECT * FROM accounts WHERE account_number = ?');
        console.log(`   Parameters: ["${accountNumber}"]\n`);
        
        return {
            text: 'SELECT * FROM accounts WHERE account_number = $1',
            values: [accountNumber]
        };
    }

    // Sanitize input
    static sanitizeInput(input) {
        // Remove SQL injection attempts
        const dangerous = [
            '--', ';', '/*', '*/', 'xp_', 'sp_', 
            'DROP', 'DELETE', 'INSERT', 'UPDATE',
            'EXEC', 'EXECUTE', 'SCRIPT'
        ];
        
        let sanitized = input;
        dangerous.forEach(pattern => {
            const regex = new RegExp(pattern, 'gi');
            sanitized = sanitized.replace(regex, '');
        });
        
        console.log('Sanitized input:');
        console.log(`  Original: ${input}`);
        console.log(`  Sanitized: ${sanitized}\n`);
        
        return sanitized;
    }

    // Validate input format
    static validateAccountNumber(accountNumber) {
        const pattern = /^ACC-\d{9}$/;
        
        if (!pattern.test(accountNumber)) {
            console.log(`❌ Invalid account number format: ${accountNumber}\n`);
            return false;
        }
        
        console.log(`✅ Valid account number format: ${accountNumber}\n`);
        return true;
    }
}

// Demonstrate injection prevention
const maliciousAccount = "ACC-123' OR '1'='1";
InjectionPrevention.buildQueryUnsafe(maliciousAccount);
InjectionPrevention.buildQuerySafe('ACC-123456789');

const sqlInjection = "'; DROP TABLE accounts; --";
InjectionPrevention.sanitizeInput(sqlInjection);

InjectionPrevention.validateAccountNumber('ACC-123456789');
InjectionPrevention.validateAccountNumber("ACC-123' OR '1'='1");

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. Secure Data Storage
// =============================================================================

console.log('4. Secure Data Storage:\n');

class SecureStorage {
    // ❌ NEVER store sensitive data in localStorage
    static storeDataInsecure(key, value) {
        console.log('❌ INSECURE: Storing in localStorage');
        console.log(`   ${key}: ${value}`);
        console.log('   Accessible via JavaScript & XSS attacks\n');
        
        // localStorage.setItem(key, value); // DON'T DO THIS
    }

    // ✅ Use httpOnly cookies for sensitive data
    static setSecureCookie(name, value, options = {}) {
        console.log('✅ SECURE: httpOnly cookie');
        console.log(`   Name: ${name}`);
        console.log('   Attributes:');
        console.log('     - HttpOnly: true (no JavaScript access)');
        console.log('     - Secure: true (HTTPS only)');
        console.log('     - SameSite: Strict (CSRF protection)');
        console.log(`     - Max-Age: ${options.maxAge || 3600}s\n`);
        
        // Server-side:
        // res.cookie(name, value, {
        //     httpOnly: true,
        //     secure: true,
        //     sameSite: 'strict',
        //     maxAge: 3600000
        // });
    }

    // Encrypt sensitive data
    static async encryptData(data, key) {
        console.log('Encrypting sensitive data...');
        
        // In production, use Web Crypto API or similar
        const encrypted = Buffer.from(data).toString('base64');
        
        console.log(`  Original: ${data.substring(0, 20)}...`);
        console.log(`  Encrypted: ${encrypted.substring(0, 30)}...\n`);
        
        return encrypted;
    }

    // Hash passwords
    static async hashPassword(password) {
        console.log('Hashing password...');
        
        // In production, use bcrypt or argon2
        // const hash = await bcrypt.hash(password, 12);
        
        const mockHash = 'protected$2b$12$' + Buffer.from(password).toString('base64');
        
        console.log(`  Password: ${password.replace(/./g, '*')}`);
        console.log(`  Hash: ${mockHash.substring(0, 40)}...\n`);
        
        return mockHash;
    }
}

// Demonstrate secure storage
SecureStorage.storeDataInsecure('pin', '1234'); // NEVER do this
SecureStorage.setSecureCookie('session_token', 'abc123xyz', { maxAge: 3600 });

await SecureStorage.encryptData('ACC-123456789:BALANCE:50000', 'encryption-key');
await SecureStorage.hashPassword('UserPassword123!');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. Prototype Pollution Prevention
// =============================================================================

console.log('5. Prototype Pollution Prevention:\n');

class PrototypePollutionPrevention {
    // ❌ VULNERABLE: Unsafe merge
    static mergeUnsafe(target, source) {
        console.log('❌ UNSAFE merge (vulnerable to prototype pollution)');
        
        for (let key in source) {
            if (source.hasOwnProperty(key)) {
                target[key] = source[key];
            }
        }
        
        return target;
    }

    // ✅ SAFE: Validate keys
    static mergeSafe(target, source) {
        console.log('✅ SAFE merge (protected against prototype pollution)\n');
        
        const dangerousKeys = ['__proto__', 'constructor', 'prototype'];
        
        for (let key in source) {
            if (source.hasOwnProperty(key) && !dangerousKeys.includes(key)) {
                target[key] = source[key];
            }
        }
        
        return target;
    }

    // Use Object.create(null) for dictionaries
    static createSafeDictionary() {
        // Object without prototype chain
        const dict = Object.create(null);
        
        console.log('Created safe dictionary (no prototype):');
        console.log(`  dict.__proto__: ${dict.__proto__}\n`);
        
        return dict;
    }

    // Freeze objects to prevent modification
    static freezeConfig(config) {
        Object.freeze(config);
        
        console.log('Froze configuration object');
        console.log('  Attempts to modify will fail silently or throw in strict mode\n');
        
        return config;
    }
}

// Demonstrate prototype pollution prevention
const maliciousPayload = {
    accountNumber: 'ACC-123456789',
    __proto__: { isAdmin: true }
};

console.log('Attempting prototype pollution...\n');

const account1 = {};
// PrototypePollutionPrevention.mergeUnsafe(account1, maliciousPayload);

const account2 = {};
PrototypePollutionPrevention.mergeSafe(account2, maliciousPayload);

const safeDict = PrototypePollutionPrevention.createSafeDictionary();
safeDict.accountNumber = 'ACC-123456789';
console.log('Safe dictionary:', safeDict);
console.log();

const config = { apiUrl: 'https://api.emiratesnbd.com', timeout: 30000 };
PrototypePollutionPrevention.freezeConfig(config);

console.log('='.repeat(70) + '\n');

// =============================================================================
// SECURITY CHECKLIST
// =============================================================================

console.log('SECURITY CHECKLIST:\n');

console.log('✅ Input Validation:');
console.log('  • Validate all user inputs');
console.log('  • Whitelist allowed characters');
console.log('  • Use regex patterns for format validation');
console.log('  • Sanitize before processing');

console.log('\n✅ Output Encoding:');
console.log('  • Escape HTML entities');
console.log('  • Use textContent instead of innerHTML');
console.log('  • Implement Content Security Policy');
console.log('  • Validate URLs before redirects');

console.log('\n✅ Authentication & Authorization:');
console.log('  • Use strong password hashing (bcrypt, argon2)');
console.log('  • Implement MFA (Multi-Factor Authentication)');
console.log('  • Use secure session management');
console.log('  • Validate permissions on every request');

console.log('\n✅ Data Protection:');
console.log('  • Use HTTPS everywhere');
console.log('  • Encrypt sensitive data at rest');
console.log('  • Use httpOnly cookies for tokens');
console.log('  • Never log sensitive information');

console.log('\n✅ API Security:');
console.log('  • Implement rate limiting');
console.log('  • Use CSRF tokens');
console.log('  • Validate origin headers');
console.log('  • Use parameterized queries');

console.log('\n✅ Dependency Management:');
console.log('  • Keep dependencies updated');
console.log('  • Run security audits (npm audit)');
console.log('  • Use tools like Snyk or Dependabot');
console.log('  • Review third-party code');

console.log('\n✅ Error Handling:');
console.log('  • Don't expose stack traces');
console.log('  • Use generic error messages');
console.log('  • Log errors securely');
console.log('  • Implement monitoring');
```

---

## Question 30: What are JavaScript best practices and coding standards?

### Answer:

Following best practices and coding standards ensures maintainable, performant, and secure code.

### Banking Scenario: Code Standards at Emirates NBD

```javascript
console.log('=== JavaScript Best Practices - Emirates NBD ===\n');

// =============================================================================
// 1. NAMING CONVENTIONS
// =============================================================================

console.log('1. NAMING CONVENTIONS:\n');

// ✅ GOOD: Clear, descriptive names
class BankAccount {
    constructor(accountNumber, initialBalance) {
        this.accountNumber = accountNumber;  // camelCase for variables
        this.balance = initialBalance;
        this.createdAt = new Date();
    }
    
    // camelCase for methods
    calculateInterest(rate, years) {
        return this.balance * rate * years;
    }
}

// Constants in UPPER_SNAKE_CASE
const MAX_TRANSFER_AMOUNT = 100000;
const MIN_ACCOUNT_BALANCE = 1000;
const API_TIMEOUT_MS = 30000;

// Private fields with # prefix
class SecureAccount {
    #pin;  // Private field
    #balance;
    
    constructor(pin, balance) {
        this.#pin = pin;
        this.#balance = balance;
    }
    
    verifyPin(inputPin) {
        return this.#pin === inputPin;
    }
}

console.log('✅ Use descriptive names');
console.log('✅ camelCase for variables and functions');
console.log('✅ PascalCase for classes');
console.log('✅ UPPER_SNAKE_CASE for constants');
console.log('✅ # prefix for private fields\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. CODE ORGANIZATION
// =============================================================================

console.log('2. CODE ORGANIZATION:\n');

// ✅ GOOD: Single Responsibility Principle
class AccountService {
    async getAccount(accountNumber) {
        // Only handles account retrieval
        return { accountNumber, balance: 50000 };
    }
}

class TransactionService {
    async createTransaction(transaction) {
        // Only handles transaction creation
        return { id: 'TXN-001', ...transaction };
    }
}

class NotificationService {
    async sendNotification(userId, message) {
        // Only handles notifications
        console.log(`Notification sent to ${userId}: ${message}`);
    }
}

// ✅ GOOD: Module pattern
const BankingModule = (() => {
    // Private
    let accounts = [];
    
    // Public API
    return {
        addAccount(account) {
            accounts.push(account);
        },
        getAccountCount() {
            return accounts.length;
        }
    };
})();

console.log('✅ Single Responsibility Principle');
console.log('✅ Separation of Concerns');
console.log('✅ Modular code structure');
console.log('✅ Clear file organization\n');

console.log('Suggested folder structure:');
console.log('src/');
console.log('  models/        - Data models');
console.log('  services/      - Business logic');
console.log('  controllers/   - Request handlers');
console.log('  utils/         - Helper functions');
console.log('  config/        - Configuration');
console.log('  tests/         - Test files\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. ERROR HANDLING
// =============================================================================

console.log('3. ERROR HANDLING:\n');

// ✅ GOOD: Specific error classes
class BankingError extends Error {
    constructor(message, code) {
        super(message);
        this.name = 'BankingError';
        this.code = code;
    }
}

class InsufficientFundsError extends BankingError {
    constructor(balance, requested) {
        super(
            `Insufficient funds: Balance ${balance}, Requested ${requested}`,
            'INSUFFICIENT_FUNDS'
        );
        this.name = 'InsufficientFundsError';
    }
}

// ✅ GOOD: Try-catch with specific handling
async function processWithdrawal(account, amount) {
    try {
        if (account.balance < amount) {
            throw new InsufficientFundsError(account.balance, amount);
        }
        
        account.balance -= amount;
        return { success: true, newBalance: account.balance };
        
    } catch (error) {
        if (error instanceof InsufficientFundsError) {
            console.error(`Withdrawal failed: ${error.message}`);
            return { success: false, error: error.code };
        }
        
        // Unknown error
        console.error('Unexpected error:', error);
        throw error;
    }
}

// ✅ GOOD: Input validation
function validateTransferAmount(amount) {
    if (typeof amount !== 'number') {
        throw new TypeError('Amount must be a number');
    }
    
    if (amount <= 0) {
        throw new RangeError('Amount must be positive');
    }
    
    if (amount > MAX_TRANSFER_AMOUNT) {
        throw new RangeError(`Amount exceeds maximum: ${MAX_TRANSFER_AMOUNT}`);
    }
    
    return true;
}

console.log('✅ Use custom error classes');
console.log('✅ Handle errors appropriately');
console.log('✅ Validate inputs early');
console.log('✅ Provide meaningful error messages');
console.log('✅ Log errors properly\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. ASYNC/AWAIT BEST PRACTICES
// =============================================================================

console.log('4. ASYNC/AWAIT BEST PRACTICES:\n');

// ✅ GOOD: Proper error handling with async/await
async function fetchAccountData(accountNumber) {
    try {
        const account = await getAccount(accountNumber);
        const transactions = await getTransactions(accountNumber);
        
        return { account, transactions };
    } catch (error) {
        console.error('Failed to fetch account data:', error);
        throw error;
    }
}

// ✅ GOOD: Parallel async operations
async function fetchMultipleAccounts(accountNumbers) {
    try {
        // Execute in parallel
        const accounts = await Promise.all(
            accountNumbers.map(num => getAccount(num))
        );
        
        return accounts;
    } catch (error) {
        console.error('Failed to fetch accounts:', error);
        throw error;
    }
}

// ✅ GOOD: Timeout for async operations
async function fetchWithTimeout(promise, timeoutMs = 5000) {
    const timeout = new Promise((_, reject) => {
        setTimeout(() => reject(new Error('Operation timed out')), timeoutMs);
    });
    
    return Promise.race([promise, timeout]);
}

// Mock functions for demonstration
async function getAccount(accountNumber) {
    return { accountNumber, balance: 50000 };
}

async function getTransactions(accountNumber) {
    return [{ id: 'TXN-001', amount: 5000 }];
}

console.log('✅ Always handle promise rejections');
console.log('✅ Use Promise.all() for parallel operations');
console.log('✅ Implement timeouts for network calls');
console.log('✅ Avoid blocking the event loop\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PERFORMANCE OPTIMIZATION
// =============================================================================

console.log('5. PERFORMANCE OPTIMIZATION:\n');

// ✅ GOOD: Debouncing expensive operations
function debounce(func, delay = 300) {
    let timeoutId;
    return function (...args) {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => func.apply(this, args), delay);
    };
}

// ✅ GOOD: Memoization for expensive calculations
function memoize(fn) {
    const cache = new Map();
    return function (...args) {
        const key = JSON.stringify(args);
        if (cache.has(key)) return cache.get(key);
        
        const result = fn.apply(this, args);
        cache.set(key, result);
        return result;
    };
}

// ✅ GOOD: Use const/let instead of var
const accountNumber = 'ACC-123456789';  // Block-scoped, immutable reference
let balance = 50000;  // Block-scoped, mutable

// ✅ GOOD: Avoid unnecessary DOM operations
// Cache DOM references
const accountElement = {}; // document.getElementById('account');
// Batch DOM updates
// Use DocumentFragment for multiple insertions

console.log('✅ Use debouncing/throttling');
console.log('✅ Implement caching/memoization');
console.log('✅ Minimize DOM operations');
console.log('✅ Use const/let instead of var');
console.log('✅ Lazy load resources\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// BEST PRACTICES SUMMARY
// =============================================================================

console.log('BEST PRACTICES SUMMARY:\n');

console.log('Code Quality:');
console.log('  ✓ Write clear, self-documenting code');
console.log('  ✓ Follow consistent naming conventions');
console.log('  ✓ Keep functions small and focused');
console.log('  ✓ Use meaningful variable names');
console.log('  ✓ Comment complex logic, not obvious code');

console.log('\nCode Structure:');
console.log('  ✓ Organize code into modules');
console.log('  ✓ Follow Single Responsibility Principle');
console.log('  ✓ Use ES6+ features (classes, arrow functions, modules)');
console.log('  ✓ Prefer composition over inheritance');
console.log('  ✓ Keep dependencies minimal');

console.log('\nError Handling:');
console.log('  ✓ Always handle errors explicitly');
console.log('  ✓ Use custom error classes');
console.log('  ✓ Validate inputs early');
console.log('  ✓ Fail fast, fail loudly');
console.log('  ✓ Log errors appropriately');

console.log('\nAsynchronous Code:');
console.log('  ✓ Use async/await over callbacks');
console.log('  ✓ Handle promise rejections');
console.log('  ✓ Implement timeouts');
console.log('  ✓ Use Promise.all() for parallel ops');
console.log('  ✓ Avoid blocking operations');

console.log('\nSecurity:');
console.log('  ✓ Validate and sanitize inputs');
console.log('  ✓ Escape output');
console.log('  ✓ Use parameterized queries');
console.log('  ✓ Implement CSRF protection');
console.log('  ✓ Keep dependencies updated');

console.log('\nPerformance:');
console.log('  ✓ Debounce/throttle expensive operations');
console.log('  ✓ Implement caching strategies');
console.log('  ✓ Minimize DOM operations');
console.log('  ✓ Use lazy loading');
console.log('  ✓ Profile and optimize hot paths');

console.log('\nTesting:');
console.log('  ✓ Write unit tests');
console.log('  ✓ Aim for 80%+ code coverage');
console.log('  ✓ Test edge cases and errors');
console.log('  ✓ Use continuous integration');
console.log('  ✓ Keep tests fast and isolated');

console.log('\nTools & Linting:');
console.log('  ✓ Use ESLint for linting');
console.log('  ✓ Use Prettier for formatting');
console.log('  ✓ Configure pre-commit hooks');
console.log('  ✓ Use TypeScript for type safety');
console.log('  ✓ Run security audits regularly');
```

### Key Takeaways:

**Testing**:
- Unit tests: Test individual functions (70%)
- Integration tests: Test interactions (20%)
- E2E tests: Test workflows (10%)
- Use Jest, Mocha, or similar frameworks
- Aim for 80%+ code coverage

**Security**:
- Prevent XSS: Escape HTML, use CSP
- Prevent CSRF: Use tokens, validate origin
- Prevent injection: Parameterized queries
- Secure storage: httpOnly cookies, encryption
- Regular security audits

**Best Practices**:
- Clear naming conventions
- Single Responsibility Principle
- Proper error handling
- Async/await best practices
- Performance optimization
- Regular code reviews

---

**End of Questions 28-30**
