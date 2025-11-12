# Questions 7-10: Advanced TypeScript Patterns

## Question 7: Explain TypeScript with React - Component typing, hooks, and props

### Answer:

**TypeScript with React** provides type safety for component props, state, hooks, and events. It catches errors at compile time and improves the development experience with better autocomplete and refactoring.

### Key Concepts:
1. **Component Props** - Typed props interfaces
2. **Hooks Typing** - useState, useEffect, custom hooks
3. **Event Handlers** - Typed event parameters
4. **Generic Components** - Reusable typed components
5. **Context API** - Typed context providers

### Banking Scenario: React TypeScript Banking Dashboard at Emirates NBD

```typescript
console.log('=== TypeScript with React - Emirates NBD ===\n');

// =============================================================================
// 1. COMPONENT PROPS TYPING
// =============================================================================

console.log('1. COMPONENT PROPS:\n');

// Functional component with typed props
interface AccountCardProps {
    accountNumber: string;
    accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
    balance: number;
    currency: string;
    onViewDetails: (accountNumber: string) => void;
    className?: string; // Optional prop
}

// Component definition
function AccountCard(props: AccountCardProps) {
    const { accountNumber, accountType, balance, currency, onViewDetails, className } = props;
    
    return `
    <div className="${className || 'account-card'}">
      <h3>${accountType} Account</h3>
      <p>Account: ${accountNumber}</p>
      <p>Balance: ${balance.toLocaleString()} ${currency}</p>
      <button onclick="handleClick">View Details</button>
    </div>
    `;
}

console.log('Functional Component with Props:\n');
console.log(AccountCard({
    accountNumber: 'ACC-123456789',
    accountType: 'SAVINGS',
    balance: 50000,
    currency: 'AED',
    onViewDetails: (acc) => console.log(`Viewing ${acc}`)
}));
console.log();

// With children prop
interface ButtonProps {
    children: React.ReactNode;
    variant?: 'primary' | 'secondary';
    onClick: () => void;
    disabled?: boolean;
}

console.log('Component with Children:');
console.log(`  Button<{ children: ReactNode, ... }>\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. HOOKS TYPING
// =============================================================================

console.log('2. REACT HOOKS:\n');

// useState with types
interface Account {
    id: string;
    accountNumber: string;
    balance: number;
    type: string;
}

console.log('useState<T>:');
console.log(`  const [account, setAccount] = useState<Account | null>(null);`);
console.log(`  const [accounts, setAccounts] = useState<Account[]>([]);`);
console.log(`  const [loading, setLoading] = useState<boolean>(false);\n`);

// useEffect typing
console.log('useEffect:');
console.log(`  useEffect(() => {
    // Effect logic
    return () => {
      // Cleanup
    };
  }, [dependencies]);\n`);

// useRef typing
console.log('useRef<T>:');
console.log(`  const inputRef = useRef<HTMLInputElement>(null);`);
console.log(`  const timerRef = useRef<NodeJS.Timeout | null>(null);\n`);

// Custom hook with types
interface UseAccountsReturn {
    accounts: Account[];
    loading: boolean;
    error: string | null;
    fetchAccounts: () => Promise<void>;
    createAccount: (data: Omit<Account, 'id'>) => Promise<void>;
}

function useAccounts(): UseAccountsReturn {
    // Simulated implementation
    return {
        accounts: [],
        loading: false,
        error: null,
        fetchAccounts: async () => {},
        createAccount: async (data) => {}
    };
}

console.log('Custom Hook:');
console.log(`  function useAccounts(): UseAccountsReturn { ... }`);
console.log(`  const { accounts, loading, fetchAccounts } = useAccounts();\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. EVENT HANDLERS
// =============================================================================

console.log('3. EVENT HANDLERS:\n');

interface TransferFormProps {
    onSubmit: (data: TransferData) => void;
}

interface TransferData {
    fromAccount: string;
    toAccount: string;
    amount: number;
}

console.log('Event Handler Types:\n');
console.log('Form Events:');
console.log(`  onChange: (event: React.ChangeEvent<HTMLInputElement>) => void`);
console.log(`  onSubmit: (event: React.FormEvent<HTMLFormElement>) => void\n`);

console.log('Mouse Events:');
console.log(`  onClick: (event: React.MouseEvent<HTMLButtonElement>) => void`);
console.log(`  onMouseEnter: (event: React.MouseEvent<HTMLDivElement>) => void\n`);

console.log('Keyboard Events:');
console.log(`  onKeyDown: (event: React.KeyboardEvent<HTMLInputElement>) => void\n`);

console.log('Example Implementation:');
console.log(`
function TransferForm({ onSubmit }: TransferFormProps) {
  const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    const formData = new FormData(event.currentTarget);
    onSubmit({
      fromAccount: formData.get('from') as string,
      toAccount: formData.get('to') as string,
      amount: Number(formData.get('amount'))
    });
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. GENERIC COMPONENTS
// =============================================================================

console.log('4. GENERIC COMPONENTS:\n');

// Generic list component
interface ListProps<T> {
    items: T[];
    renderItem: (item: T) => string;
    keyExtractor: (item: T) => string;
    emptyMessage?: string;
}

function List<T>(props: ListProps<T>): string {
    const { items, renderItem, keyExtractor, emptyMessage } = props;
    
    if (items.length === 0) {
        return emptyMessage || 'No items';
    }
    
    return items.map(item => renderItem(item)).join('\n');
}

// Usage with different types
const accounts: Account[] = [
    { id: '1', accountNumber: 'ACC-001', balance: 50000, type: 'SAVINGS' },
    { id: '2', accountNumber: 'ACC-002', balance: 30000, type: 'CURRENT' }
];

console.log('Generic List Component:\n');
console.log(List({
    items: accounts,
    renderItem: (account) => `  ${account.accountNumber}: ${account.balance} AED`,
    keyExtractor: (account) => account.id
}));
console.log();

// Generic form component
interface FormProps<T> {
    initialValues: T;
    onSubmit: (values: T) => void;
    validate?: (values: T) => Record<keyof T, string>;
}

console.log('Generic Form Component:');
console.log(`  <Form<TransferData> initialValues={...} onSubmit={...} />\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. CONTEXT API WITH TYPES
// =============================================================================

console.log('5. CONTEXT API:\n');

// Auth context
interface User {
    id: string;
    email: string;
    name: string;
    roles: string[];
}

interface AuthContextType {
    user: User | null;
    loading: boolean;
    login: (email: string, password: string) => Promise<void>;
    logout: () => void;
    hasRole: (role: string) => boolean;
}

console.log('Typed Context:\n');
console.log(`
interface AuthContextType {
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}\n`);

// Banking context
interface BankingContextType {
    accounts: Account[];
    selectedAccount: Account | null;
    selectAccount: (id: string) => void;
    refreshAccounts: () => Promise<void>;
}

console.log('Banking Context Example:');
console.log(`
function AccountDashboard() {
  const { accounts, selectedAccount, selectAccount } = useBanking();
  
  return (
    <div>
      {accounts.map(account => (
        <AccountCard
          key={account.id}
          {...account}
          onClick={() => selectAccount(account.id)}
        />
      ))}
    </div>
  );
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// REACT TYPESCRIPT SUMMARY
// =============================================================================

console.log('REACT TYPESCRIPT SUMMARY:\n');

console.log('Component Types:');
console.log('  • FC<Props> - Functional component');
console.log('  • ReactNode - Children elements');
console.log('  • ReactElement - Single element');
console.log('  • JSX.Element - JSX elements\n');

console.log('Props Patterns:');
console.log('  • Interface for props definition');
console.log('  • Optional props with ?');
console.log('  • Children prop for composition');
console.log('  • Generic props for flexibility\n');

console.log('Hooks:');
console.log('  • useState<T> - Typed state');
console.log('  • useEffect - Side effects');
console.log('  • useRef<T> - DOM references');
console.log('  • useContext<T> - Typed context');
console.log('  • Custom hooks with return types\n');

console.log('Events:');
console.log('  • ChangeEvent<HTMLInputElement>');
console.log('  • FormEvent<HTMLFormElement>');
console.log('  • MouseEvent<HTMLButtonElement>');
console.log('  • KeyboardEvent<HTMLInputElement>\n');

console.log('Best Practices:');
console.log('  ✓ Define interfaces for all props');
console.log('  ✓ Type all useState calls');
console.log('  ✓ Use generic components');
console.log('  ✓ Type context properly');
console.log('  ✓ Export prop interfaces');
console.log('  ✓ Use strict mode');

console.log();
```

---

## Question 8: How do you handle API types and data fetching in TypeScript?

### Answer:

**API Types** ensure type safety when communicating with backend services. TypeScript helps define request/response types, handle errors, and validate data at compile time.

### Key Patterns:
1. **API Client** - Typed HTTP client
2. **DTOs** - Data Transfer Objects
3. **Response Types** - Structured API responses
4. **Error Handling** - Typed error responses
5. **Type Guards** - Runtime validation

### Banking Scenario: API Integration at Emirates NBD

```typescript
console.log('=== API Types and Data Fetching - Emirates NBD ===\n');

// =============================================================================
// 1. API RESPONSE TYPES
// =============================================================================

console.log('1. API RESPONSE TYPES:\n');

// Generic API response wrapper
interface ApiResponse<T> {
    success: boolean;
    data?: T;
    error?: ApiError;
    metadata?: {
        timestamp: string;
        requestId: string;
    };
}

interface ApiError {
    code: string;
    message: string;
    details?: Record<string, any>;
}

// Paginated response
interface PaginatedResponse<T> {
    items: T[];
    pagination: {
        page: number;
        pageSize: number;
        total: number;
        totalPages: number;
    };
}

// Domain types
interface Account {
    id: string;
    accountNumber: string;
    accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
    balance: number;
    currency: string;
    status: 'ACTIVE' | 'FROZEN' | 'CLOSED';
    createdAt: string;
    updatedAt: string;
}

interface Transaction {
    id: string;
    accountId: string;
    type: 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER';
    amount: number;
    currency: string;
    status: 'PENDING' | 'COMPLETED' | 'FAILED';
    timestamp: string;
    description?: string;
}

console.log('Response Type Structure:');
console.log(`  ApiResponse<T> - Wraps all API responses`);
console.log(`  PaginatedResponse<T> - For list endpoints`);
console.log(`  Domain types - Account, Transaction, etc.\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. TYPED API CLIENT
// =============================================================================

console.log('2. TYPED API CLIENT:\n');

// HTTP client configuration
interface ApiConfig {
    baseURL: string;
    timeout: number;
    headers: Record<string, string>;
}

// Request options
interface RequestOptions {
    method?: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
    body?: any;
    headers?: Record<string, string>;
    params?: Record<string, string | number>;
}

class ApiClient {
    private config: ApiConfig;
    
    constructor(config: ApiConfig) {
        this.config = config;
    }
    
    private buildUrl(endpoint: string, params?: Record<string, string | number>): string {
        const url = `${this.config.baseURL}${endpoint}`;
        if (!params) return url;
        
        const queryString = Object.entries(params)
            .map(([key, value]) => `${key}=${encodeURIComponent(value)}`)
            .join('&');
        
        return `${url}?${queryString}`;
    }
    
    async request<T>(endpoint: string, options: RequestOptions = {}): Promise<ApiResponse<T>> {
        const url = this.buildUrl(endpoint, options.params);
        
        console.log(`  ${options.method || 'GET'} ${endpoint}`);
        
        // Simulate API call
        await new Promise(resolve => setTimeout(resolve, 100));
        
        // Mock response
        return {
            success: true,
            data: {} as T,
            metadata: {
                timestamp: new Date().toISOString(),
                requestId: `REQ-${Date.now()}`
            }
        };
    }
    
    async get<T>(endpoint: string, params?: Record<string, string | number>): Promise<T> {
        const response = await this.request<T>(endpoint, { method: 'GET', params });
        if (!response.success || !response.data) {
            throw new Error(response.error?.message || 'Request failed');
        }
        return response.data;
    }
    
    async post<T, B = any>(endpoint: string, body: B): Promise<T> {
        const response = await this.request<T>(endpoint, { method: 'POST', body });
        if (!response.success || !response.data) {
            throw new Error(response.error?.message || 'Request failed');
        }
        return response.data;
    }
    
    async put<T, B = any>(endpoint: string, body: B): Promise<T> {
        const response = await this.request<T>(endpoint, { method: 'PUT', body });
        if (!response.success || !response.data) {
            throw new Error(response.error?.message || 'Request failed');
        }
        return response.data;
    }
    
    async delete<T>(endpoint: string): Promise<T> {
        const response = await this.request<T>(endpoint, { method: 'DELETE' });
        if (!response.success || !response.data) {
            throw new Error(response.error?.message || 'Request failed');
        }
        return response.data;
    }
}

const apiClient = new ApiClient({
    baseURL: 'https://api.emiratesnbd.com/v1',
    timeout: 30000,
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer token123'
    }
});

console.log('API Client Features:');
console.log('  ✓ Generic request method');
console.log('  ✓ Type-safe get/post/put/delete');
console.log('  ✓ Automatic error handling');
console.log('  ✓ Query parameter support\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. API SERVICE LAYER
// =============================================================================

console.log('3. API SERVICE LAYER:\n');

// DTOs for requests
interface CreateAccountDto {
    accountNumber: string;
    accountType: 'SAVINGS' | 'CURRENT' | 'FIXED';
    initialBalance: number;
    currency: string;
}

interface TransferDto {
    fromAccountId: string;
    toAccountId: string;
    amount: number;
    description?: string;
}

// Service methods with typed responses
class AccountService {
    constructor(private api: ApiClient) {}
    
    async getAccounts(params?: {
        page?: number;
        pageSize?: number;
        status?: string;
    }): Promise<PaginatedResponse<Account>> {
        console.log('  Fetching accounts...');
        
        // Mock response
        return {
            items: [
                {
                    id: 'ACC-001',
                    accountNumber: 'ACC-123456789',
                    accountType: 'SAVINGS',
                    balance: 50000,
                    currency: 'AED',
                    status: 'ACTIVE',
                    createdAt: new Date().toISOString(),
                    updatedAt: new Date().toISOString()
                }
            ],
            pagination: {
                page: params?.page || 1,
                pageSize: params?.pageSize || 10,
                total: 1,
                totalPages: 1
            }
        };
    }
    
    async getAccount(id: string): Promise<Account> {
        console.log(`  Fetching account: ${id}`);
        return await this.api.get<Account>(`/accounts/${id}`);
    }
    
    async createAccount(dto: CreateAccountDto): Promise<Account> {
        console.log(`  Creating account: ${dto.accountNumber}`);
        return await this.api.post<Account, CreateAccountDto>('/accounts', dto);
    }
    
    async updateAccount(id: string, updates: Partial<Account>): Promise<Account> {
        console.log(`  Updating account: ${id}`);
        return await this.api.put<Account, Partial<Account>>(`/accounts/${id}`, updates);
    }
    
    async deleteAccount(id: string): Promise<void> {
        console.log(`  Deleting account: ${id}`);
        await this.api.delete<void>(`/accounts/${id}`);
    }
}

class TransactionService {
    constructor(private api: ApiClient) {}
    
    async getTransactions(accountId: string): Promise<Transaction[]> {
        console.log(`  Fetching transactions for: ${accountId}`);
        return await this.api.get<Transaction[]>(`/accounts/${accountId}/transactions`);
    }
    
    async transfer(dto: TransferDto): Promise<Transaction> {
        console.log(`  Processing transfer: ${dto.amount} AED`);
        return await this.api.post<Transaction, TransferDto>('/transactions/transfer', dto);
    }
}

// Usage
(async () => {
    const accountService = new AccountService(apiClient);
    const transactionService = new TransactionService(apiClient);
    
    console.log('Service Layer:\n');
    
    const accounts = await accountService.getAccounts({ page: 1, pageSize: 10 });
    console.log(`  Retrieved ${accounts.items.length} accounts\n`);
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 4. TYPE GUARDS FOR VALIDATION
    // =============================================================================
    
    console.log('4. RUNTIME VALIDATION:\n');
    
    // Type guard functions
    function isAccount(obj: any): obj is Account {
        return (
            typeof obj === 'object' &&
            typeof obj.id === 'string' &&
            typeof obj.accountNumber === 'string' &&
            typeof obj.balance === 'number' &&
            ['SAVINGS', 'CURRENT', 'FIXED'].includes(obj.accountType) &&
            ['ACTIVE', 'FROZEN', 'CLOSED'].includes(obj.status)
        );
    }
    
    function isTransaction(obj: any): obj is Transaction {
        return (
            typeof obj === 'object' &&
            typeof obj.id === 'string' &&
            typeof obj.amount === 'number' &&
            ['DEPOSIT', 'WITHDRAWAL', 'TRANSFER'].includes(obj.type)
        );
    }
    
    // Validation wrapper
    async function fetchAccountSafe(id: string): Promise<Account> {
        const response = await apiClient.get<unknown>(`/accounts/${id}`);
        
        if (!isAccount(response)) {
            throw new Error('Invalid account data received from API');
        }
        
        return response;
    }
    
    console.log('Type Guards:');
    console.log('  ✓ isAccount() - Validates account structure');
    console.log('  ✓ isTransaction() - Validates transaction structure');
    console.log('  ✓ Runtime validation of API responses\n');
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // 5. ERROR HANDLING PATTERNS
    // =============================================================================
    
    console.log('5. ERROR HANDLING:\n');
    
    // Custom error classes
    class ApiClientError extends Error {
        constructor(
            message: string,
            public statusCode: number,
            public code: string,
            public details?: any
        ) {
            super(message);
            this.name = 'ApiClientError';
        }
    }
    
    class NetworkError extends ApiClientError {
        constructor(message: string) {
            super(message, 0, 'NETWORK_ERROR');
            this.name = 'NetworkError';
        }
    }
    
    class ValidationError extends ApiClientError {
        constructor(message: string, details: any) {
            super(message, 400, 'VALIDATION_ERROR', details);
            this.name = 'ValidationError';
        }
    }
    
    // Error handler
    async function handleApiCall<T>(
        apiCall: () => Promise<T>
    ): Promise<{ data?: T; error?: ApiClientError }> {
        try {
            const data = await apiCall();
            return { data };
        } catch (error) {
            if (error instanceof ApiClientError) {
                return { error };
            }
            
            // Convert unknown errors
            return {
                error: new ApiClientError(
                    error instanceof Error ? error.message : 'Unknown error',
                    500,
                    'UNKNOWN_ERROR'
                )
            };
        }
    }
    
    console.log('Error Handling:');
    console.log('  • Custom error classes');
    console.log('  • Typed error responses');
    console.log('  • Error wrapper pattern');
    console.log('  • Result type pattern\n');
    
    // Usage example
    const { data, error } = await handleApiCall(() => 
        accountService.getAccount('ACC-001')
    );
    
    if (error) {
        console.log(`  Error: ${error.code} - ${error.message}`);
    } else {
        console.log(`  Success: Retrieved account data\n`);
    }
    
    console.log('='.repeat(70) + '\n');
    
    // =============================================================================
    // API TYPES SUMMARY
    // =============================================================================
    
    console.log('API TYPES SUMMARY:\n');
    
    console.log('Response Structures:');
    console.log('  • ApiResponse<T> - Standard wrapper');
    console.log('  • PaginatedResponse<T> - Lists with pagination');
    console.log('  • ApiError - Error details');
    console.log('  • Metadata - Request tracking\n');
    
    console.log('Type Safety:');
    console.log('  • Generic API client methods');
    console.log('  • Typed service layer');
    console.log('  • DTOs for requests');
    console.log('  • Type guards for validation\n');
    
    console.log('Error Handling:');
    console.log('  • Custom error classes');
    console.log('  • Result type pattern');
    console.log('  • Error code system');
    console.log('  • Detailed error information\n');
    
    console.log('Best Practices:');
    console.log('  ✓ Define all API types');
    console.log('  ✓ Use generic API client');
    console.log('  ✓ Validate responses at runtime');
    console.log('  ✓ Handle errors gracefully');
    console.log('  ✓ Use DTOs for requests');
    console.log('  ✓ Separate concerns (client/service)');
    
    console.log();
})();
```

---

## Question 9: Explain TypeScript module systems, namespaces, and project organization

### Answer:

**Module Organization** in TypeScript involves structuring code for maintainability and reusability. TypeScript supports ES6 modules, namespaces, and various import/export patterns.

### Key Concepts:
1. **ES6 Modules** - import/export syntax
2. **Module Resolution** - How TypeScript finds modules
3. **Barrel Exports** - index.ts pattern
4. **Path Mapping** - Absolute imports
5. **Declaration Files** - .d.ts files

### Banking Scenario: Module Organization at Emirates NBD

```typescript
console.log('=== TypeScript Module Organization - Emirates NBD ===\n');

// =============================================================================
// 1. ES6 MODULE PATTERNS
// =============================================================================

console.log('1. ES6 MODULES:\n');

console.log('Named Exports:\n');
console.log(`// account.service.ts
export class AccountService {
  async getAccount(id: string): Promise<Account> { ... }
}

export interface Account {
  id: string;
  balance: number;
}

export const MAX_BALANCE = 1000000;

// Usage
import { AccountService, Account, MAX_BALANCE } from './account.service';\n`);

console.log('Default Export:\n');
console.log(`// database.ts
export default class Database {
  connect() { ... }
}

// Usage
import Database from './database';
import DB from './database'; // Can rename\n`);

console.log('Mixed Exports:\n');
console.log(`// banking.ts
export default class BankingService { }
export { AccountService } from './account.service';
export { TransactionService } from './transaction.service';

// Usage
import BankingService, { AccountService } from './banking';\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. BARREL EXPORTS
// =============================================================================

console.log('2. BARREL EXPORTS:\n');

console.log('Project Structure:\n');
console.log(`src/
├── models/
│   ├── account.model.ts
│   ├── transaction.model.ts
│   └── index.ts           <- Barrel export
├── services/
│   ├── account.service.ts
│   ├── transaction.service.ts
│   └── index.ts           <- Barrel export
└── utils/
    ├── validator.ts
    ├── formatter.ts
    └── index.ts           <- Barrel export\n`);

console.log('Barrel Export Pattern:\n');
console.log(`// models/index.ts
export * from './account.model';
export * from './transaction.model';
export { Account as BankAccount } from './account.model'; // Re-export with alias

// Usage - Clean imports
import { Account, Transaction } from './models';
import { AccountService, TransactionService } from './services';\n`);

console.log('Benefits:');
console.log('  ✓ Cleaner import statements');
console.log('  ✓ Single source for exports');
console.log('  ✓ Easier refactoring');
console.log('  ✓ Better encapsulation\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. PATH MAPPING
// =============================================================================

console.log('3. PATH MAPPING:\n');

console.log('tsconfig.json:\n');
console.log(`{
  "compilerOptions": {
    "baseUrl": "./src",
    "paths": {
      "@models/*": ["models/*"],
      "@services/*": ["services/*"],
      "@utils/*": ["utils/*"],
      "@config/*": ["config/*"],
      "@types/*": ["types/*"]
    }
  }
}\n`);

console.log('Usage with Path Mapping:\n');
console.log(`// Instead of:
import { Account } from '../../../models/account.model';

// Use:
import { Account } from '@models/account.model';
import { AccountService } from '@services/account.service';
import { validateAccount } from '@utils/validator';\n`);

console.log('Benefits:');
console.log('  ✓ No relative path hell');
console.log('  ✓ Easier to move files');
console.log('  ✓ More readable imports');
console.log('  ✓ IDE autocomplete\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. MODULE RESOLUTION
// =============================================================================

console.log('4. MODULE RESOLUTION:\n');

console.log('Resolution Strategies:\n');
console.log('1. Classic (--moduleResolution classic):');
console.log('   • Looks for .ts, .tsx, .d.ts');
console.log('   • Relative imports only\n');

console.log('2. Node (--moduleResolution node):');
console.log('   • node_modules lookup');
console.log('   • package.json main field');
console.log('   • index.ts files\n');

console.log('Import Resolution Order:\n');
console.log(`import { Account } from './models/account';

1. ./models/account.ts
2. ./models/account.tsx
3. ./models/account.d.ts
4. ./models/account/package.json (main field)
5. ./models/account/index.ts
6. ./models/account/index.tsx
7. ./models/account/index.d.ts\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. PROJECT STRUCTURE
// =============================================================================

console.log('5. PROJECT STRUCTURE:\n');

console.log('Enterprise Banking Application:\n');
console.log(`
banking-app/
├── src/
│   ├── api/                      # API layer
│   │   ├── client.ts             # API client
│   │   ├── endpoints.ts          # API endpoints
│   │   └── index.ts
│   │
│   ├── models/                   # Domain models
│   │   ├── account.model.ts
│   │   ├── transaction.model.ts
│   │   ├── customer.model.ts
│   │   └── index.ts
│   │
│   ├── services/                 # Business logic
│   │   ├── account.service.ts
│   │   ├── transaction.service.ts
│   │   ├── notification.service.ts
│   │   └── index.ts
│   │
│   ├── repositories/             # Data access
│   │   ├── account.repository.ts
│   │   ├── transaction.repository.ts
│   │   └── index.ts
│   │
│   ├── dto/                      # Data Transfer Objects
│   │   ├── account.dto.ts
│   │   ├── transaction.dto.ts
│   │   └── index.ts
│   │
│   ├── types/                    # Type definitions
│   │   ├── api.types.ts
│   │   ├── common.types.ts
│   │   └── index.ts
│   │
│   ├── utils/                    # Utilities
│   │   ├── validator.ts
│   │   ├── formatter.ts
│   │   ├── logger.ts
│   │   └── index.ts
│   │
│   ├── config/                   # Configuration
│   │   ├── database.ts
│   │   ├── env.ts
│   │   └── index.ts
│   │
│   ├── middleware/               # Express middleware
│   │   ├── auth.middleware.ts
│   │   ├── validation.middleware.ts
│   │   └── index.ts
│   │
│   ├── constants/                # Constants
│   │   ├── account.constants.ts
│   │   ├── error.constants.ts
│   │   └── index.ts
│   │
│   └── server.ts                 # Entry point
│
├── tests/                        # Tests
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── dist/                         # Compiled output
├── node_modules/
├── tsconfig.json
├── package.json
└── README.md\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. DECLARATION FILES
// =============================================================================

console.log('6. DECLARATION FILES:\n');

console.log('Custom Type Declarations:\n');
console.log(`// types/global.d.ts
declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        email: string;
      };
    }
  }
}

export {};\n`);

console.log('Module Declarations:\n');
console.log(`// types/custom-module.d.ts
declare module 'banking-sdk' {
  export interface BankingClient {
    getAccounts(): Promise<Account[]>;
  }
  
  export function createClient(config: any): BankingClient;
}\n`);

console.log('Ambient Declarations:\n');
console.log(`// types/environment.d.ts
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: 'development' | 'production' | 'test';
    DATABASE_URL: string;
    JWT_SECRET: string;
    PORT: string;
  }
}\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// MODULE ORGANIZATION SUMMARY
// =============================================================================

console.log('MODULE ORGANIZATION SUMMARY:\n');

console.log('Module Patterns:');
console.log('  • Named exports - Multiple exports');
console.log('  • Default exports - Single export');
console.log('  • Re-exports - Barrel pattern');
console.log('  • Mixed exports - Flexible pattern\n');

console.log('Organization:');
console.log('  • Layer-based folders (models, services)');
console.log('  • Feature-based folders (account, transaction)');
console.log('  • Barrel exports (index.ts)');
console.log('  • Path mapping for clean imports\n');

console.log('Best Practices:');
console.log('  ✓ Use barrel exports');
console.log('  ✓ Configure path mapping');
console.log('  ✓ Consistent folder structure');
console.log('  ✓ One component per file');
console.log('  ✓ Logical layer separation');
console.log('  ✓ Type files separate from logic');
console.log('  ✓ Clear naming conventions\n');

console.log('Configuration:');
console.log('  • baseUrl - Base for relative imports');
console.log('  • paths - Path aliases');
console.log('  • moduleResolution - Resolution strategy');
console.log('  • rootDir - Source root');
console.log('  • outDir - Output directory');

console.log();
```

---

## Question 10: Explain TypeScript compilation, build process, and optimization

### Answer:

**TypeScript Compilation** transforms TypeScript code into JavaScript. Understanding the compilation process, configuration options, and optimization techniques is crucial for production applications.

### Key Topics:
1. **Compiler Options** - tsconfig.json settings
2. **Build Process** - Compilation workflow
3. **Source Maps** - Debugging support
4. **Type Checking** - Strict mode options
5. **Performance** - Build optimization

### Banking Scenario: Build Configuration at Emirates NBD

```typescript
console.log('=== TypeScript Compilation & Build - Emirates NBD ===\n');

// =============================================================================
// 1. COMPILER CONFIGURATION
// =============================================================================

console.log('1. TSCONFIG.JSON:\n');

const tsConfig = {
    compilerOptions: {
        // Target JavaScript version
        target: 'ES2020',
        module: 'commonjs',
        lib: ['ES2020'],
        
        // Output
        outDir: './dist',
        rootDir: './src',
        removeComments: true,
        sourceMap: true,
        declaration: true,
        declarationMap: true,
        
        // Strict Type Checking
        strict: true,
        noImplicitAny: true,
        strictNullChecks: true,
        strictFunctionTypes: true,
        strictBindCallApply: true,
        strictPropertyInitialization: true,
        noImplicitThis: true,
        alwaysStrict: true,
        
        // Additional Checks
        noUnusedLocals: true,
        noUnusedParameters: true,
        noImplicitReturns: true,
        noFallthroughCasesInSwitch: true,
        noUncheckedIndexedAccess: true,
        
        // Module Resolution
        moduleResolution: 'node',
        baseUrl: './src',
        paths: {
            '@models/*': ['models/*'],
            '@services/*': ['services/*'],
            '@utils/*': ['utils/*']
        },
        esModuleInterop: true,
        allowSyntheticDefaultImports: true,
        resolveJsonModule: true,
        
        // Advanced
        skipLibCheck: true,
        forceConsistentCasingInFileNames: true,
        experimentalDecorators: true,
        emitDecoratorMetadata: true,
        incremental: true,
        tsBuildInfoFile: './.tsbuildinfo'
    },
    include: ['src/**/*'],
    exclude: ['node_modules', 'dist', '**/*.spec.ts']
};

console.log('Complete tsconfig.json:');
console.log(JSON.stringify(tsConfig, null, 2));
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 2. BUILD SCRIPTS
// =============================================================================

console.log('2. BUILD SCRIPTS:\n');

const packageScripts = {
    // Development
    'dev': 'nodemon --exec ts-node --files src/server.ts',
    'dev:watch': 'tsc --watch',
    
    // Build
    'build': 'tsc',
    'build:prod': 'tsc --project tsconfig.prod.json',
    'clean': 'rm -rf dist',
    'prebuild': 'npm run clean',
    
    // Type checking
    'type-check': 'tsc --noEmit',
    'type-check:watch': 'tsc --noEmit --watch',
    
    // Linting
    'lint': 'eslint src/**/*.ts',
    'lint:fix': 'eslint src/**/*.ts --fix',
    
    // Testing
    'test': 'jest',
    'test:watch': 'jest --watch',
    'test:coverage': 'jest --coverage',
    
    // Start
    'start': 'node dist/server.js',
    'start:prod': 'NODE_ENV=production node dist/server.js'
};

console.log('package.json scripts:\n');
Object.entries(packageScripts).forEach(([key, value]) => {
    console.log(`  "${key}": "${value}"`);
});
console.log();

console.log('='.repeat(70) + '\n');

// =============================================================================
// 3. PRODUCTION BUILD OPTIMIZATION
// =============================================================================

console.log('3. PRODUCTION OPTIMIZATION:\n');

const prodConfig = {
    extends: './tsconfig.json',
    compilerOptions: {
        sourceMap: false,              // Disable for production
        declaration: false,             // No .d.ts files
        removeComments: true,           // Remove comments
        noEmitOnError: true,            // Fail on errors
        noUnusedLocals: true,           // No unused variables
        noUnusedParameters: true,       // No unused params
        incremental: false              // Clean build each time
    },
    exclude: [
        'node_modules',
        'dist',
        '**/*.spec.ts',
        '**/*.test.ts',
        'tests'
    ]
};

console.log('tsconfig.prod.json:\n');
console.log(JSON.stringify(prodConfig, null, 2));
console.log();

console.log('Production Optimizations:');
console.log('  ✓ No source maps (smaller bundle)');
console.log('  ✓ No declaration files');
console.log('  ✓ Remove comments');
console.log('  ✓ Exclude test files');
console.log('  ✓ Strict error checking\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 4. INCREMENTAL COMPILATION
// =============================================================================

console.log('4. INCREMENTAL COMPILATION:\n');

console.log('Enable Incremental Builds:\n');
console.log(`{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": "./.tsbuildinfo"
  }
}\n`);

console.log('Benefits:');
console.log('  • Only recompile changed files');
console.log('  • Faster subsequent builds');
console.log('  • Build cache in .tsbuildinfo');
console.log('  • Ideal for development\n');

console.log('Project References (Monorepo):');
console.log(`
banking-monorepo/
├── packages/
│   ├── models/
│   │   ├── src/
│   │   └── tsconfig.json
│   ├── services/
│   │   ├── src/
│   │   └── tsconfig.json
│   └── api/
│       ├── src/
│       └── tsconfig.json
└── tsconfig.json\n`);

console.log('Root tsconfig.json:');
console.log(`{
  "files": [],
  "references": [
    { "path": "./packages/models" },
    { "path": "./packages/services" },
    { "path": "./packages/api" }
  ]
}\n`);

console.log('Build command:');
console.log('  tsc --build --verbose\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 5. WEBPACK/BUNDLER INTEGRATION
// =============================================================================

console.log('5. WEBPACK INTEGRATION:\n');

console.log('webpack.config.js:\n');
console.log(`
const path = require('path');

module.exports = {
  mode: 'production',
  entry: './src/server.ts',
  target: 'node',
  module: {
    rules: [
      {
        test: /\\.ts$/,
        use: 'ts-loader',
        exclude: /node_modules/
      }
    ]
  },
  resolve: {
    extensions: ['.ts', '.js'],
    alias: {
      '@models': path.resolve(__dirname, 'src/models'),
      '@services': path.resolve(__dirname, 'src/services')
    }
  },
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  optimization: {
    minimize: true
  }
};\n`);

console.log('Dependencies:');
console.log('  npm install -D webpack webpack-cli ts-loader\n');

console.log('Build script:');
console.log('  webpack --config webpack.config.js\n');

console.log('='.repeat(70) + '\n');

// =============================================================================
// 6. CI/CD INTEGRATION
// =============================================================================

console.log('6. CI/CD PIPELINE:\n');

console.log('GitHub Actions (.github/workflows/build.yml):\n');
console.log(`
name: Build and Test

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Type check
        run: npm run type-check
        
      - name: Lint
        run: npm run lint
        
      - name: Test
        run: npm run test:coverage
        
      - name: Build
        run: npm run build:prod
        
      - name: Upload coverage
        uses: codecov/codecov-action@v2
        with:
          files: ./coverage/lcov.info\n`);

console.log('='.repeat(70) + '\n');

// =============================================================================
// COMPILATION SUMMARY
// =============================================================================

console.log('COMPILATION SUMMARY:\n');

console.log('Compiler Options:');
console.log('  • target - Output JavaScript version');
console.log('  • module - Module system (commonjs, es6)');
console.log('  • strict - Enable all strict checks');
console.log('  • outDir - Output directory');
console.log('  • sourceMap - Debug support\n');

console.log('Build Optimization:');
console.log('  • Incremental compilation');
console.log('  • Project references');
console.log('  • Remove comments and source maps');
console.log('  • Skip lib check');
console.log('  • Bundler integration\n');

console.log('Development Workflow:');
console.log('  1. ts-node for development');
console.log('  2. Watch mode for live reload');
console.log('  3. Type checking in CI/CD');
console.log('  4. Production build optimization');
console.log('  5. Automated testing\n');

console.log('Best Practices:');
console.log('  ✓ Enable strict mode');
console.log('  ✓ Use incremental builds');
console.log('  ✓ Separate dev/prod configs');
console.log('  ✓ Type check before build');
console.log('  ✓ Integrate with CI/CD');
console.log('  ✓ Monitor build times');
console.log('  ✓ Cache dependencies\n');

console.log('Performance Tips:');
console.log('  • Use skipLibCheck: true');
console.log('  • Enable incremental compilation');
console.log('  • Use project references for monorepos');
console.log('  • Exclude unnecessary files');
console.log('  • Use faster bundlers (esbuild, swc)');

console.log();
```

### Key Takeaways:

**React with TypeScript**:
- Typed component props and state
- Generic components for reusability
- Typed hooks (useState, useEffect, custom hooks)
- Event handler types
- Context API with types

**API Integration**:
- Generic API client with type safety
- DTOs for requests
- Structured response types
- Type guards for validation
- Custom error classes

**Module Organization**:
- ES6 modules with import/export
- Barrel exports for clean imports
- Path mapping for absolute imports
- Declaration files for types
- Logical project structure

**Compilation & Build**:
- Comprehensive tsconfig.json
- Development and production configs
- Incremental compilation
- Build optimization
- CI/CD integration

---

**End of Questions 7-10**
