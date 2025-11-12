# Questions 18-20: TypeScript Advanced Practices & Migration

## Question 18: How do you migrate JavaScript projects to TypeScript?

### Answer:

**Migration to TypeScript** is a gradual process that involves converting JavaScript code while maintaining functionality. TypeScript supports incremental migration, allowing mixed JS/TS codebases.

### Key Steps:
1. **Initial Setup** - Configure TypeScript
2. **Gradual Conversion** - File-by-file migration
3. **Type Definitions** - Add types incrementally
4. **Refactoring** - Improve code quality
5. **Testing** - Ensure functionality

### Banking Scenario: Migration at Emirates NBD

```typescript
console.log('=== JavaScript to TypeScript Migration - Emirates NBD ===\n');

// =============================================================================
// 1. MIGRATION SETUP
// =============================================================================

console.log('1. MIGRATION SETUP:\n');

console.log('Step 1: Install TypeScript:\n');
console.log(`npm install --save-dev typescript @types/node
npm install --save-dev @types/express @types/jest
npm install --save-dev ts-node\n`);

console.log('Step 2: Initial tsconfig.json (Permissive):\n');
console.log(`{
  "compilerOptions": {
    // Allow JavaScript files
    "allowJs": true,
    "checkJs": false,  // Don't type-check JS files initially
    
    // Output
    "outDir": "./dist",
    "rootDir": "./src",
    
    // Target
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    
    // Module resolution
    "moduleResolution": "node",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    
    // Permissive settings for migration
    "strict": false,           // Enable later
    "noImplicitAny": false,    // Enable later
    "strictNullChecks": false, // Enable later
    
    // Source maps for debugging
    "sourceMap": true,
    
    // Skip lib checking for faster builds
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}\n`);

console.log('Step 3: Update package.json:\n');
console.log(`{
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "dev": "ts-node src/server.ts",
    "start": "node dist/server.js",
    
    "type-check": "tsc --noEmit",
    "type-check:watch": "tsc --noEmit --watch"
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. GRADUAL CONVERSION STRATEGY
// =============================================================================

console.log('2. CONVERSION STRATEGY:\n');

console.log('Migration Order (Bottom-Up Approach):\n');
console.log(`
1. Utility functions and helpers
   └─ Pure functions with clear inputs/outputs
   
2. Models and types
   └─ Data structures and interfaces
   
3. Services and business logic
   └─ Core functionality
   
4. Controllers and routes
   └─ API endpoints
   
5. Middleware
   └─ Request handling logic
   
6. Application entry point
   └─ Server setup\n`);

console.log('Example: Converting a utility file\n');

console.log('❌ Before (JavaScript - utils/validator.js):\n');
console.log(`
// utils/validator.js
function validateEmail(email) {
  const regex = /^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/;
  return regex.test(email);
}

function validateAccountNumber(accountNumber) {
  return accountNumber && accountNumber.startsWith('ACC-');
}

function validateAmount(amount) {
  return typeof amount === 'number' && amount > 0;
}

module.exports = {
  validateEmail,
  validateAccountNumber,
  validateAmount
};\n`);

console.log('✅ After (TypeScript - utils/validator.ts):\n');
console.log(`
// utils/validator.ts
export function validateEmail(email: string): boolean {
  const regex = /^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/;
  return regex.test(email);
}

export function validateAccountNumber(accountNumber: string): boolean {
  return accountNumber.startsWith('ACC-');
}

export function validateAmount(amount: number): boolean {
  return amount > 0;
}

// Add type guards
export function isValidEmail(email: unknown): email is string {
  return typeof email === 'string' && validateEmail(email);
}

export function isValidAmount(amount: unknown): amount is number {
  return typeof amount === 'number' && validateAmount(amount);
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. ADDING TYPES INCREMENTALLY
// =============================================================================

console.log('3. ADDING TYPES:\n');

console.log('Step 1: Define types (types/account.ts):\n');
console.log(`
// types/account.ts
export interface Account {
  id: string;
  accountNumber: string;
  customerId: string;
  accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
  balance: number;
  currency: string;
  status: 'ACTIVE' | 'FROZEN' | 'CLOSED';
  createdAt: Date;
  updatedAt: Date;
}

export interface Transaction {
  id: string;
  accountId: string;
  type: 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER';
  amount: number;
  description?: string;
  timestamp: Date;
}

export interface CreateAccountDto {
  accountNumber: string;
  customerId: string;
  accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
  initialBalance: number;
}\n`);

console.log('Step 2: Convert service\n');

console.log('❌ Before (JavaScript):\n');
console.log(`
// services/accountService.js
class AccountService {
  constructor(repository) {
    this.repository = repository;
  }
  
  async getAccount(id) {
    return this.repository.findById(id);
  }
  
  async createAccount(data) {
    const account = {
      id: generateId(),
      ...data,
      balance: data.initialBalance,
      status: 'ACTIVE',
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    return this.repository.save(account);
  }
  
  async deposit(accountId, amount) {
    const account = await this.getAccount(accountId);
    if (!account) throw new Error('Account not found');
    
    account.balance += amount;
    account.updatedAt = new Date();
    
    return this.repository.save(account);
  }
}

module.exports = AccountService;\n`);

console.log('✅ After (TypeScript):\n');
console.log(`
// services/accountService.ts
import { Account, CreateAccountDto } from '../types/account';
import { AccountRepository } from '../repositories/accountRepository';

export class AccountService {
  constructor(private repository: AccountRepository) {}
  
  async getAccount(id: string): Promise<Account | null> {
    return this.repository.findById(id);
  }
  
  async createAccount(data: CreateAccountDto): Promise<Account> {
    const account: Account = {
      id: this.generateId(),
      accountNumber: data.accountNumber,
      customerId: data.customerId,
      accountType: data.accountType,
      balance: data.initialBalance,
      currency: 'AED',
      status: 'ACTIVE',
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    return this.repository.save(account);
  }
  
  async deposit(accountId: string, amount: number): Promise<Account> {
    const account = await this.getAccount(accountId);
    if (!account) {
      throw new Error('Account not found');
    }
    
    account.balance += amount;
    account.updatedAt = new Date();
    
    return this.repository.save(account);
  }
  
  private generateId(): string {
    return \`ACC-\${Date.now()}-\${Math.random().toString(36).substr(2, 9)}\`;
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. HANDLING COMMON MIGRATION CHALLENGES
// =============================================================================

console.log('4. MIGRATION CHALLENGES:\n');

console.log('Challenge 1: Dynamic Properties\n');

console.log('❌ JavaScript Pattern:\n');
console.log(`
function enrichAccount(account, userData) {
  account.userName = userData.name;
  account.userEmail = userData.email;
  account.lastAccess = new Date();
  return account;
}\n`);

console.log('✅ TypeScript Solutions:\n');
console.log(`
// Option 1: Extend the type
interface EnrichedAccount extends Account {
  userName: string;
  userEmail: string;
  lastAccess: Date;
}

function enrichAccount(
  account: Account,
  userData: { name: string; email: string }
): EnrichedAccount {
  return {
    ...account,
    userName: userData.name,
    userEmail: userData.email,
    lastAccess: new Date()
  };
}

// Option 2: Use generics for flexibility
function enrichAccount<T extends Account, U>(
  account: T,
  enrichData: U
): T & U {
  return { ...account, ...enrichData };
}\n`);

console.log('Challenge 2: Callback Hell\n');

console.log('❌ JavaScript:\n');
console.log(`
function transfer(from, to, amount, callback) {
  getAccount(from, (err, fromAccount) => {
    if (err) return callback(err);
    
    getAccount(to, (err, toAccount) => {
      if (err) return callback(err);
      
      if (fromAccount.balance < amount) {
        return callback(new Error('Insufficient funds'));
      }
      
      updateBalance(from, -amount, (err) => {
        if (err) return callback(err);
        
        updateBalance(to, amount, (err) => {
          if (err) return callback(err);
          callback(null, { success: true });
        });
      });
    });
  });
}\n`);

console.log('✅ TypeScript with async/await:\n');
console.log(`
async function transfer(
  fromId: string,
  toId: string,
  amount: number
): Promise<{ success: boolean; transactionId: string }> {
  const fromAccount = await getAccount(fromId);
  if (!fromAccount) {
    throw new Error('Source account not found');
  }
  
  const toAccount = await getAccount(toId);
  if (!toAccount) {
    throw new Error('Destination account not found');
  }
  
  if (fromAccount.balance < amount) {
    throw new InsufficientFundsError(fromId, amount, fromAccount.balance);
  }
  
  // Use transaction for atomicity
  const result = await database.transaction(async (tx) => {
    await tx.updateBalance(fromId, -amount);
    await tx.updateBalance(toId, amount);
    
    const transactionId = await tx.createTransaction({
      fromAccount: fromId,
      toAccount: toId,
      amount,
      type: 'TRANSFER'
    });
    
    return { success: true, transactionId };
  });
  
  return result;
}\n`);

console.log('Challenge 3: Loose Typing\n');

console.log('❌ JavaScript:\n');
console.log(`
function processAccount(account) {
  // account could be anything
  if (account.type === 'SAVINGS') {
    return account.interestRate || 0;
  }
  return 0;
}\n`);

console.log('✅ TypeScript with discriminated unions:\n');
console.log(`
interface SavingsAccount {
  type: 'SAVINGS';
  accountNumber: string;
  balance: number;
  interestRate: number;
}

interface CurrentAccount {
  type: 'CURRENT';
  accountNumber: string;
  balance: number;
  overdraftLimit: number;
}

type Account = SavingsAccount | CurrentAccount;

function processAccount(account: Account): number {
  switch (account.type) {
    case 'SAVINGS':
      return account.interestRate; // Type-safe access
    case 'CURRENT':
      return 0;
    default:
      const _exhaustive: never = account;
      return _exhaustive;
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PROGRESSIVE STRICTNESS
// =============================================================================

console.log('5. PROGRESSIVE STRICTNESS:\n');

console.log('Migration Timeline:\n');
console.log(`
Phase 1: Initial Setup (Week 1-2)
  {
    "allowJs": true,
    "strict": false,
    "noImplicitAny": false
  }
  → Convert utility functions and types

Phase 2: Core Conversion (Week 3-4)
  {
    "allowJs": true,
    "strict": false,
    "noImplicitAny": true  // Enable this
  }
  → Convert services and repositories
  → Add explicit types

Phase 3: Strict Checking (Week 5-6)
  {
    "allowJs": true,
    "strict": false,
    "noImplicitAny": true,
    "strictNullChecks": true  // Enable this
  }
  → Handle null/undefined properly
  → Add null checks

Phase 4: Full Strict Mode (Week 7-8)
  {
    "allowJs": false,  // Disable JS files
    "strict": true     // Full strict mode
  }
  → Convert remaining JS files
  → Fix all strict mode errors\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// MIGRATION SUMMARY
// =============================================================================

console.log('MIGRATION SUMMARY:\n');

console.log('Migration Strategy:');
console.log('  1. Start with permissive settings');
console.log('  2. Convert files bottom-up');
console.log('  3. Add types incrementally');
console.log('  4. Enable strict checks gradually');
console.log('  5. Test thoroughly at each step\n');

console.log('Best Practices:');
console.log('  ✓ Use allowJs during migration');
console.log('  ✓ Rename .js to .ts incrementally');
console.log('  ✓ Define types in separate files');
console.log('  ✓ Use type guards for validation');
console.log('  ✓ Leverage VS Code quick fixes');
console.log('  ✓ Run tests after each conversion');
console.log('  ✓ Enable strict mode gradually');
console.log('  ✓ Document type decisions\n');

console.log('Common Pitfalls:');
console.log('  ✗ Converting too many files at once');
console.log('  ✗ Using "any" everywhere');
console.log('  ✗ Ignoring TypeScript errors');
console.log('  ✗ Not testing after conversion');
console.log('  ✗ Enabling strict mode too early');

console.log();
```

---

## Question 19: What are TypeScript utility types and how do you create custom ones?

### Answer:

**Utility Types** are built-in TypeScript types that facilitate common type transformations. Custom utility types can be created for domain-specific transformations.

### Key Utility Types:
1. **Built-in Utilities** - Partial, Pick, Omit, etc.
2. **Mapped Types** - Transform properties
3. **Conditional Types** - Type logic
4. **Template Literals** - String manipulation
5. **Custom Utilities** - Domain-specific types

### Banking Scenario: Utility Types at Emirates NBD

```typescript
console.log('=== TypeScript Utility Types - Emirates NBD ===\n');

// =============================================================================
// 1. BUILT-IN UTILITY TYPES
// =============================================================================

console.log('1. BUILT-IN UTILITY TYPES:\n');

interface Account {
  id: string;
  accountNumber: string;
  customerId: string;
  accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
  balance: number;
  currency: string;
  status: 'ACTIVE' | 'FROZEN' | 'CLOSED';
  interestRate?: number;
  createdAt: Date;
  updatedAt: Date;
}

console.log('Partial<T> - Makes all properties optional:\n');
console.log(`
type PartialAccount = Partial<Account>;
// All properties are optional

function updateAccount(id: string, updates: Partial<Account>) {
  // Can update any subset of properties
}\n`);

console.log('Required<T> - Makes all properties required:\n');
console.log(`
type RequiredAccount = Required<Account>;
// interestRate is now required instead of optional\n`);

console.log('Readonly<T> - Makes all properties readonly:\n');
console.log(`
type ReadonlyAccount = Readonly<Account>;

const account: ReadonlyAccount = { /* ... */ };
// account.balance = 5000; // Error: Cannot assign to readonly property\n`);

console.log('Pick<T, K> - Select specific properties:\n');
console.log(`
type AccountSummary = Pick<Account, 'id' | 'accountNumber' | 'balance'>;
// Only these 3 properties

const summary: AccountSummary = {
  id: 'ACC-001',
  accountNumber: 'ACC-123456',
  balance: 50000
};\n`);

console.log('Omit<T, K> - Remove specific properties:\n');
console.log(`
type AccountWithoutTimestamps = Omit<Account, 'createdAt' | 'updatedAt'>;
// All properties except createdAt and updatedAt

type CreateAccountDto = Omit<Account, 'id' | 'createdAt' | 'updatedAt'>;\n`);

console.log('Record<K, T> - Create object type with specific keys:\n');
console.log(`
type AccountsByType = Record<
  'SAVINGS' | 'CURRENT' | 'FIXED',
  Account[]
>;

const groupedAccounts: AccountsByType = {
  SAVINGS: [/* savings accounts */],
  CURRENT: [/* current accounts */],
  FIXED: [/* fixed deposit accounts */]
};\n`);

console.log('Extract<T, U> - Extract types from union:\n');
console.log(`
type AccountType = Account['accountType'];
// 'SAVINGS' | 'CURRENT' | 'FIXED'

type ActiveStatus = Extract<Account['status'], 'ACTIVE'>;
// Only 'ACTIVE'\n`);

console.log('Exclude<T, U> - Exclude types from union:\n');
console.log(`
type InactiveStatus = Exclude<Account['status'], 'ACTIVE'>;
// 'FROZEN' | 'CLOSED'\n`);

console.log('NonNullable<T> - Remove null and undefined:\n');
console.log(`
type MaybeAccount = Account | null | undefined;
type DefiniteAccount = NonNullable<MaybeAccount>;
// Only Account, no null or undefined\n`);

console.log('ReturnType<T> - Extract return type of function:\n');
console.log(`
function getAccount(id: string): Promise<Account | null> {
  // implementation
  return Promise.resolve(null);
}

type GetAccountReturn = ReturnType<typeof getAccount>;
// Promise<Account | null>\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. CUSTOM MAPPED TYPES
// =============================================================================

console.log('2. CUSTOM MAPPED TYPES:\n');

console.log('Nullable - Make properties nullable:\n');
console.log(`
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableAccount = Nullable<Account>;
// All properties can be null

const account: NullableAccount = {
  id: 'ACC-001',
  accountNumber: null,
  balance: 50000,
  // ... other properties
};\n`);

console.log('DeepPartial - Recursively partial:\n');
console.log(`
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object
    ? DeepPartial<T[P]>
    : T[P];
};

interface Customer {
  id: string;
  name: string;
  address: {
    street: string;
    city: string;
    country: string;
  };
  account: Account;
}

type PartialCustomer = DeepPartial<Customer>;
// All nested properties are optional\n`);

console.log('Mutable - Remove readonly:\n');
console.log(`
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};

type MutableAccount = Mutable<Readonly<Account>>;
// All properties are now mutable\n`);

console.log('PickByType - Pick properties by their type:\n');
console.log(`
type PickByType<T, U> = {
  [P in keyof T as T[P] extends U ? P : never]: T[P];
};

type AccountNumbers = PickByType<Account, number>;
// { balance: number }

type AccountStrings = PickByType<Account, string>;
// { id: string; accountNumber: string; customerId: string; currency: string; ... }\n`);

console.log('RequireAtLeastOne - Require at least one property:\n');
console.log(`
type RequireAtLeastOne<T, Keys extends keyof T = keyof T> = 
  Pick<T, Exclude<keyof T, Keys>> & 
  {
    [K in Keys]-?: Required<Pick<T, K>> & Partial<Pick<T, Exclude<Keys, K>>>
  }[Keys];

interface SearchCriteria {
  accountNumber?: string;
  customerId?: string;
  email?: string;
}

type ValidSearch = RequireAtLeastOne<SearchCriteria>;
// Must have at least one of the properties\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. CONDITIONAL TYPES
// =============================================================================

console.log('3. CONDITIONAL TYPES:\n');

console.log('UnwrapPromise - Extract Promise type:\n');
console.log(`
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

type AsyncResult = Promise<Account>;
type SyncResult = UnwrapPromise<AsyncResult>;
// Account\n`);

console.log('IsArray - Check if type is array:\n');
console.log(`
type IsArray<T> = T extends any[] ? true : false;

type Test1 = IsArray<Account[]>;  // true
type Test2 = IsArray<Account>;    // false\n`);

console.log('FunctionProperties - Extract function properties:\n');
console.log(`
type FunctionProperties<T> = {
  [P in keyof T]: T[P] extends Function ? P : never;
}[keyof T];

interface AccountService {
  accounts: Account[];
  getAccount(id: string): Promise<Account>;
  createAccount(data: any): Promise<Account>;
  deleteAccount(id: string): Promise<void>;
}

type ServiceMethods = FunctionProperties<AccountService>;
// 'getAccount' | 'createAccount' | 'deleteAccount'\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. TEMPLATE LITERAL TYPES
// =============================================================================

console.log('4. TEMPLATE LITERAL TYPES:\n');

console.log('Event Names:\n');
console.log(`
type EventName = 'account' | 'transaction' | 'customer';
type EventAction = 'created' | 'updated' | 'deleted';

type DomainEvent = \`\${EventName}:\${EventAction}\`;
// 'account:created' | 'account:updated' | 'account:deleted' |
// 'transaction:created' | ... (9 combinations)\n`);

console.log('HTTP Methods:\n');
console.log(`
type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Endpoint = '/accounts' | '/transactions';

type RouteDefinition = \`\${HTTPMethod} \${Endpoint}\`;
// 'GET /accounts' | 'POST /accounts' | 'PUT /accounts' | ...\n`);

console.log('Uppercase/Lowercase:\n');
console.log(`
type Status = 'active' | 'frozen' | 'closed';
type UpperStatus = Uppercase<Status>;
// 'ACTIVE' | 'FROZEN' | 'CLOSED'

type API_STATUS = 'PENDING' | 'SUCCESS' | 'ERROR';
type LowerStatus = Lowercase<API_STATUS>;
// 'pending' | 'success' | 'error'\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. REAL-WORLD CUSTOM UTILITIES
// =============================================================================

console.log('5. BANKING UTILITIES:\n');

console.log('AuditableEntity - Add audit fields:\n');
console.log(`
type AuditableEntity<T> = T & {
  createdAt: Date;
  createdBy: string;
  updatedAt: Date;
  updatedBy: string;
  version: number;
};

interface BaseAccount {
  id: string;
  accountNumber: string;
  balance: number;
}

type AuditedAccount = AuditableEntity<BaseAccount>;
// Has all BaseAccount properties plus audit fields\n`);

console.log('ApiResponse - Wrap responses:\n');
console.log(`
type ApiResponse<T, E = Error> = 
  | { success: true; data: T; error?: never }
  | { success: false; data?: never; error: E };

type AccountResponse = ApiResponse<Account>;

// Type-safe usage
const response: AccountResponse = { success: true, data: account };

if (response.success) {
  console.log(response.data.accountNumber); // Type-safe
} else {
  console.log(response.error.message);
}\n`);

console.log('PaginatedResponse - Add pagination:\n');
console.log(`
type PaginatedResponse<T> = {
  items: T[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
  };
  links: {
    first: string;
    last: string;
    prev?: string;
    next?: string;
  };
};

type AccountListResponse = PaginatedResponse<Account>;\n`);

console.log('TypedEventEmitter - Type-safe events:\n');
console.log(`
type EventMap = {
  'account:created': { account: Account };
  'account:updated': { account: Account; changes: Partial<Account> };
  'account:deleted': { accountId: string };
  'transaction:completed': { transaction: Transaction };
};

interface TypedEventEmitter<T extends Record<string, any>> {
  on<K extends keyof T>(event: K, handler: (data: T[K]) => void): void;
  emit<K extends keyof T>(event: K, data: T[K]): void;
  off<K extends keyof T>(event: K, handler: (data: T[K]) => void): void;
}

const emitter: TypedEventEmitter<EventMap> = createEmitter();

// Type-safe event handling
emitter.on('account:created', (data) => {
  console.log(data.account.accountNumber); // Typed!
});

emitter.emit('account:created', { account });\n`);

console.log('StateTransition - Enforce valid state changes:\n');
console.log(`
type AccountStatus = 'PENDING' | 'ACTIVE' | 'FROZEN' | 'CLOSED';

type ValidTransition = 
  | { from: 'PENDING'; to: 'ACTIVE' | 'CLOSED' }
  | { from: 'ACTIVE'; to: 'FROZEN' | 'CLOSED' }
  | { from: 'FROZEN'; to: 'ACTIVE' | 'CLOSED' };

function transitionStatus(transition: ValidTransition): void {
  // Only valid transitions allowed
}

// Valid
transitionStatus({ from: 'PENDING', to: 'ACTIVE' });

// Invalid - TypeScript error
// transitionStatus({ from: 'CLOSED', to: 'ACTIVE' });\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// UTILITY TYPES SUMMARY
// =============================================================================

console.log('UTILITY TYPES SUMMARY:\n');

console.log('Built-in Utilities:');
console.log('  • Partial<T> - All optional');
console.log('  • Required<T> - All required');
console.log('  • Readonly<T> - All readonly');
console.log('  • Pick<T, K> - Select properties');
console.log('  • Omit<T, K> - Remove properties');
console.log('  • Record<K, T> - Key-value mapping\n');

console.log('Custom Patterns:');
console.log('  • Mapped types for transformations');
console.log('  • Conditional types for logic');
console.log('  • Template literals for strings');
console.log('  • Recursive types for deep operations\n');

console.log('Best Practices:');
console.log('  ✓ Use built-in utilities first');
console.log('  ✓ Create reusable custom utilities');
console.log('  ✓ Document complex types');
console.log('  ✓ Test utility types');
console.log('  ✓ Keep types simple when possible');
console.log('  ✓ Use type aliases for readability');

console.log();
```

---

## Question 20: What are TypeScript best practices and common patterns for enterprise applications?

### Answer:

**Enterprise Best Practices** involve patterns, conventions, and architectural decisions that ensure maintainability, scalability, and code quality in large TypeScript applications.

### Key Areas:
1. **Project Structure** - Organization
2. **Type System** - Effective typing
3. **Error Handling** - Robust errors
4. **Code Quality** - Standards and tooling
5. **Architecture** - Scalable design

### Banking Scenario: Best Practices at Emirates NBD

```typescript
console.log('=== TypeScript Best Practices - Emirates NBD ===\n');

// =============================================================================
// 1. PROJECT STRUCTURE
// =============================================================================

console.log('1. ENTERPRISE PROJECT STRUCTURE:\n');

console.log(`
banking-platform/
├── src/
│   ├── modules/                   # Feature modules
│   │   ├── accounts/
│   │   │   ├── controllers/
│   │   │   │   └── account.controller.ts
│   │   │   ├── services/
│   │   │   │   └── account.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── account.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-account.dto.ts
│   │   │   │   └── update-account.dto.ts
│   │   │   ├── entities/
│   │   │   │   └── account.entity.ts
│   │   │   ├── interfaces/
│   │   │   │   └── account.interface.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── transactions/
│   │   ├── customers/
│   │   └── auth/
│   │
│   ├── shared/                    # Shared code
│   │   ├── decorators/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── filters/
│   │   ├── pipes/
│   │   └── utils/
│   │
│   ├── common/                    # Common utilities
│   │   ├── constants/
│   │   ├── enums/
│   │   ├── interfaces/
│   │   └── types/
│   │
│   ├── config/                    # Configuration
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   └── app.config.ts
│   │
│   ├── infrastructure/            # External services
│   │   ├── database/
│   │   ├── cache/
│   │   ├── queue/
│   │   └── messaging/
│   │
│   └── main.ts                    # Entry point
│
├── tests/                         # Tests mirror src/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docs/                          # Documentation
├── scripts/                       # Build/deployment scripts
├── .github/workflows/             # CI/CD
├── tsconfig.json
├── tsconfig.prod.json
├── package.json
└── README.md\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPE SYSTEM BEST PRACTICES
// =============================================================================

console.log('2. TYPE SYSTEM PATTERNS:\n');

console.log('✅ Use Strict Mode:\n');
console.log(`
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}\n`);

console.log('✅ Prefer Interfaces for Objects:\n');
console.log(`
// Good - Interface for object shapes
interface Account {
  id: string;
  balance: number;
}

// Use type for unions, intersections, tuples
type Status = 'ACTIVE' | 'FROZEN';
type Result = Success | Error;
type Coordinates = [number, number];\n`);

console.log('✅ Use Readonly for Immutability:\n');
console.log(`
interface ImmutableAccount {
  readonly id: string;
  readonly accountNumber: string;
  balance: number; // Can change
}

// For arrays
const transactions: readonly Transaction[] = [...];
// transactions.push() // Error\n`);

console.log('✅ Avoid "any" - Use "unknown":\n');
console.log(`
// Bad
function processData(data: any) {
  return data.value; // No type safety
}

// Good
function processData(data: unknown) {
  if (isAccount(data)) {
    return data.balance; // Type-safe
  }
  throw new Error('Invalid data');
}\n`);

console.log('✅ Use Discriminated Unions:\n');
console.log(`
type ApiResult<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }
  | { status: 'loading' };

function handleResult(result: ApiResult<Account>) {
  switch (result.status) {
    case 'success':
      console.log(result.data.accountNumber);
      break;
    case 'error':
      console.error(result.error.message);
      break;
    case 'loading':
      console.log('Loading...');
      break;
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. ERROR HANDLING PATTERNS
// =============================================================================

console.log('3. ERROR HANDLING:\n');

console.log('Custom Error Hierarchy:\n');
console.log(`
// Base error
abstract class ApplicationError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      code: this.code,
      message: this.message,
      statusCode: this.statusCode
    };
  }
}

// Domain errors
class NotFoundError extends ApplicationError {
  readonly statusCode = 404;
  readonly code = 'NOT_FOUND';
}

class ValidationError extends ApplicationError {
  readonly statusCode = 400;
  readonly code = 'VALIDATION_ERROR';
  
  constructor(
    message: string,
    public readonly fields: Record<string, string>
  ) {
    super(message);
  }
}

class InsufficientFundsError extends ApplicationError {
  readonly statusCode = 400;
  readonly code = 'INSUFFICIENT_FUNDS';
  
  constructor(
    public readonly accountId: string,
    public readonly required: number,
    public readonly available: number
  ) {
    super(\`Insufficient funds: need \${required}, have \${available}\`);
  }
}\n`);

console.log('Result Pattern:\n');
console.log(`
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function transfer(
  from: string,
  to: string,
  amount: number
): Promise<Result<Transaction, ApplicationError>> {
  try {
    // Business logic
    const transaction = await performTransfer(from, to, amount);
    return { ok: true, value: transaction };
  } catch (error) {
    if (error instanceof ApplicationError) {
      return { ok: false, error };
    }
    return { ok: false, error: new ApplicationError('Unknown error') };
  }
}

// Usage
const result = await transfer('ACC-1', 'ACC-2', 1000);

if (result.ok) {
  console.log('Transfer successful:', result.value.id);
} else {
  console.error('Transfer failed:', result.error.message);
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. CODE QUALITY
// =============================================================================

console.log('4. CODE QUALITY TOOLS:\n');

console.log('ESLint Configuration (.eslintrc.json):\n');
console.log(`
{
  "parser": "@typescript-eslint/parser",
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking",
    "prettier"
  ],
  "parserOptions": {
    "project": "./tsconfig.json"
  },
  "rules": {
    "@typescript-eslint/explicit-function-return-type": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/strict-boolean-expressions": "error",
    "@typescript-eslint/no-floating-promises": "error",
    "no-console": ["warn", { "allow": ["warn", "error"] }]
  }
}\n`);

console.log('Prettier Configuration (.prettierrc):\n');
console.log(`
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always"
}\n`);

console.log('Husky Pre-commit Hook (.husky/pre-commit):\n');
console.log(`
#!/bin/sh
npm run lint
npm run type-check
npm run test\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. ARCHITECTURAL PATTERNS
// =============================================================================

console.log('5. ARCHITECTURAL PATTERNS:\n');

console.log('Dependency Injection:\n');
console.log(`
// Define interfaces
interface IAccountRepository {
  findById(id: string): Promise<Account | null>;
  save(account: Account): Promise<Account>;
}

interface INotificationService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

// Implementation with dependencies injected
class AccountService {
  constructor(
    private readonly repository: IAccountRepository,
    private readonly notificationService: INotificationService
  ) {}
  
  async createAccount(data: CreateAccountDto): Promise<Account> {
    const account = await this.repository.save(data);
    
    await this.notificationService.sendEmail(
      data.email,
      'Account Created',
      \`Your account \${account.accountNumber} has been created\`
    );
    
    return account;
  }
}

// Dependency injection container
class Container {
  private services = new Map<string, any>();
  
  register<T>(key: string, factory: () => T): void {
    this.services.set(key, factory);
  }
  
  resolve<T>(key: string): T {
    const factory = this.services.get(key);
    if (!factory) throw new Error(\`Service \${key} not found\`);
    return factory();
  }
}

const container = new Container();
container.register('AccountRepository', () => new AccountRepository(db));
container.register('NotificationService', () => new EmailService());
container.register('AccountService', () => 
  new AccountService(
    container.resolve('AccountRepository'),
    container.resolve('NotificationService')
  )
);\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// BEST PRACTICES SUMMARY
// =============================================================================

console.log('BEST PRACTICES SUMMARY:\n');

console.log('Type System:');
console.log('  ✓ Enable strict mode');
console.log('  ✓ Avoid "any", use "unknown"');
console.log('  ✓ Prefer interfaces for objects');
console.log('  ✓ Use discriminated unions');
console.log('  ✓ Leverage utility types');
console.log('  ✓ Make immutability explicit\n');

console.log('Code Organization:');
console.log('  ✓ Feature-based folder structure');
console.log('  ✓ Separate concerns (controller/service/repository)');
console.log('  ✓ Use barrel exports (index.ts)');
console.log('  ✓ Path mapping for imports');
console.log('  ✓ Keep files focused and small\n');

console.log('Error Handling:');
console.log('  ✓ Custom error classes');
console.log('  ✓ Result pattern for operations');
console.log('  ✓ Never swallow errors');
console.log('  ✓ Type-safe error handling');
console.log('  ✓ Meaningful error messages\n');

console.log('Code Quality:');
console.log('  ✓ ESLint for code standards');
console.log('  ✓ Prettier for formatting');
console.log('  ✓ Pre-commit hooks');
console.log('  ✓ Automated testing');
console.log('  ✓ Code reviews\n');

console.log('Architecture:');
console.log('  ✓ Dependency injection');
console.log('  ✓ SOLID principles');
console.log('  ✓ Repository pattern');
console.log('  ✓ Service layer');
console.log('  ✓ Clear boundaries\n');

console.log('Performance:');
console.log('  • Incremental compilation');
console.log('  • Skip lib check');
console.log('  • Project references');
console.log('  • Efficient types');
console.log('  • Tree shaking\n');

console.log('Documentation:');
console.log('  • JSDoc comments for public APIs');
console.log('  • README for each module');
console.log('  • Architecture decisions documented');
console.log('  • Type definitions self-documenting');

console.log();
```

### Final Summary: Complete TypeScript Interview Preparation

**Questions 18-20 Coverage**:

**Migration Strategy**:
- Gradual JavaScript to TypeScript conversion
- Progressive strictness enabling
- Handling common migration challenges
- Bottom-up conversion approach
- Testing and validation throughout

**Utility Types**:
- Built-in TypeScript utilities (Partial, Pick, Omit, etc.)
- Custom mapped types for domain needs
- Conditional types for complex logic
- Template literal types for string manipulation
- Real-world banking utility patterns

**Best Practices**:
- Enterprise project structure
- Type system patterns (strict mode, immutability)
- Robust error handling with custom classes
- Code quality tools (ESLint, Prettier, Husky)
- Architectural patterns (DI, SOLID principles)
- Performance optimization strategies

---

**🎉 COMPLETE: All 20 TypeScript Questions Finished! 🎉**

### Full Coverage (Q1-Q20):

1. **Q1-Q3**: TypeScript Fundamentals (Types, Generics, Advanced Types)
2. **Q4-Q6**: TypeScript Practice (Decorators, Node.js/Express, Edge Cases)
3. **Q7-Q10**: Advanced Patterns (React, APIs, Modules, Compilation)
4. **Q11-Q14**: Advanced Topics (Design Patterns, Async, Testing, Performance)
5. **Q15-Q17**: Ecosystem & Tools (ORMs, Middleware, Configuration)
6. **Q18-Q20**: Migration & Best Practices (JS→TS Migration, Utility Types, Enterprise Practices)

**All questions feature**:
- Emirates NBD banking scenarios
- Executable TypeScript code examples
- Comprehensive explanations
- Real-world patterns
- Best practices and summaries

---

**End of Questions 18-20 - TypeScript Interview Preparation Complete!**
