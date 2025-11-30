# React Interview Question 2: Components & Props
## Understanding React Components, Props, and Component Composition

---

## 📋 Summary of What Will Be Covered

### ✅ Question 2: Components & Props Architecture
- Function Components vs Class Components
- Props: Immutability and data flow
- Component composition patterns
- Props drilling and solutions
- Children prop and component slots
- Banking Example: Reusable account components with props

---

## ❓ Question 2: What are React Components and how do Props work?

### 📘 Comprehensive Explanation

**React Components**:

Components are the building blocks of React applications. They are independent, reusable pieces of UI that can be composed together to build complex interfaces.

**Two Types of Components**:

1. **Function Components** (Modern, Preferred)
   - JavaScript functions that return JSX
   - Can use Hooks for state and lifecycle
   - Simpler syntax, easier to test
   - Better performance optimization

2. **Class Components** (Legacy)
   - ES6 classes extending `React.Component`
   - Use `this.state` and lifecycle methods
   - More boilerplate code
   - Being phased out in favor of function components

**Props (Properties)**:

Props are arguments passed to React components. They work like function parameters, allowing parent components to pass data down to child components.

**Key Props Characteristics**:
- **Immutable**: Props cannot be modified by the receiving component
- **Unidirectional**: Data flows from parent to child (one-way data flow)
- **Read-only**: Child components can only read props, not write
- **Type-safe**: Can use PropTypes or TypeScript for validation

---

### 🏗️ Component Architecture Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   COMPONENT ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PARENT COMPONENT                                               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ function BankingDashboard() {                             │ │
│  │   const accountData = {                                   │ │
│  │     id: 'ACC001',                                         │ │
│  │     name: 'John Doe',                                     │ │
│  │     balance: 5000                                         │ │
│  │   };                                                      │ │
│  │                                                           │ │
│  │   return (                                                │ │
│  │     <AccountCard                                          │ │
│  │       accountId={accountData.id}    ← PROPS              │ │
│  │       accountName={accountData.name}                      │ │
│  │       balance={accountData.balance}                       │ │
│  │     />                                                    │ │
│  │   );                                                      │ │
│  │ }                                                         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│                       Props Object                              │
│                            ↓                                    │
│  {                                                              │
│    accountId: 'ACC001',                                         │
│    accountName: 'John Doe',                                     │
│    balance: 5000                                                │
│  }                                                              │
│                            ↓                                    │
│  CHILD COMPONENT                                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ function AccountCard(props) {                             │ │
│  │   // props = {                                            │ │
│  │   //   accountId: 'ACC001',                               │ │
│  │   //   accountName: 'John Doe',                           │ │
│  │   //   balance: 5000                                      │ │
│  │   // }                                                    │ │
│  │                                                           │ │
│  │   return (                                                │ │
│  │     <div className="account-card">                        │ │
│  │       <h2>{props.accountName}</h2>   ← READ ONLY          │ │
│  │       <p>Account: {props.accountId}</p>                   │ │
│  │       <p>Balance: ${props.balance}</p>                    │ │
│  │     </div>                                                │ │
│  │   );                                                      │ │
│  │ }                                                         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Props Flow: UNIDIRECTIONAL (Parent → Child)                   │
│  Props Mutability: IMMUTABLE (Read-only in child)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 💻 Function Components vs Class Components

#### Function Components (Modern Approach)

```jsx
// ============================================
// FUNCTION COMPONENT - Modern React
// ============================================

/**
 * Simple Function Component
 * - Clean syntax
 * - No 'this' keyword
 * - Can use Hooks
 */

function AccountCard({ accountId, accountName, balance }) {
  return (
    <div className="account-card">
      <h2>{accountName}</h2>
      <p className="account-id">Account: {accountId}</p>
      <p className="balance">Balance: ${balance.toFixed(2)}</p>
    </div>
  );
}

// Alternative: Arrow Function Component
const AccountCard = ({ accountId, accountName, balance }) => {
  return (
    <div className="account-card">
      <h2>{accountName}</h2>
      <p className="account-id">Account: {accountId}</p>
      <p className="balance">Balance: ${balance.toFixed(2)}</p>
    </div>
  );
};

// With default props using destructuring
const AccountCard = ({ 
  accountId = 'N/A', 
  accountName = 'Unknown', 
  balance = 0 
}) => {
  return (
    <div className="account-card">
      <h2>{accountName}</h2>
      <p className="account-id">Account: {accountId}</p>
      <p className="balance">Balance: ${balance.toFixed(2)}</p>
    </div>
  );
};

// With TypeScript (Type-Safe)
interface AccountCardProps {
  accountId: string;
  accountName: string;
  balance: number;
  currency?: string;
}

const AccountCard: React.FC<AccountCardProps> = ({ 
  accountId, 
  accountName, 
  balance,
  currency = 'USD'
}) => {
  return (
    <div className="account-card">
      <h2>{accountName}</h2>
      <p className="account-id">Account: {accountId}</p>
      <p className="balance">
        Balance: {currency} ${balance.toFixed(2)}
      </p>
    </div>
  );
};
```

#### Class Components (Legacy Approach)

```jsx
// ============================================
// CLASS COMPONENT - Legacy React
// ============================================

import React, { Component } from 'react';

/**
 * Class Component
 * - More boilerplate
 * - Uses 'this' keyword
 * - Lifecycle methods
 * - Legacy approach
 */

class AccountCard extends Component {
  // Default props (class property)
  static defaultProps = {
    accountId: 'N/A',
    accountName: 'Unknown',
    balance: 0,
    currency: 'USD'
  };

  // Constructor (optional if no state)
  constructor(props) {
    super(props);
    // Can initialize state here
  }

  // Render method (required)
  render() {
    // Destructure props
    const { accountId, accountName, balance, currency } = this.props;

    return (
      <div className="account-card">
        <h2>{accountName}</h2>
        <p className="account-id">Account: {accountId}</p>
        <p className="balance">
          Balance: {currency} ${balance.toFixed(2)}
        </p>
      </div>
    );
  }
}

export default AccountCard;
```

**Comparison Table**:

| Feature | Function Component | Class Component |
|---------|-------------------|-----------------|
| **Syntax** | Simple function | ES6 class |
| **State** | `useState` Hook | `this.state` |
| **Lifecycle** | `useEffect` Hook | Lifecycle methods |
| **`this` keyword** | ❌ Not needed | ✅ Required |
| **Performance** | ✅ Better | ⚠️ Slightly slower |
| **Hooks** | ✅ Can use | ❌ Cannot use |
| **Code length** | ✅ Shorter | ⚠️ More verbose |
| **Testing** | ✅ Easier | ⚠️ More complex |
| **React team recommendation** | ✅ Preferred | ⚠️ Legacy |

---

### 📦 Props Deep Dive

#### Props as Function Parameters

```jsx
/**
 * Props work exactly like function parameters
 * Parent passes data, child receives it
 */

// Parent Component
function BankingDashboard() {
  // Data to pass
  const account = {
    id: 'ACC001',
    name: 'John Doe',
    balance: 5000,
    type: 'SAVINGS'
  };

  // Pass props to child
  return (
    <div>
      <h1>Dashboard</h1>
      {/* Props passed as attributes */}
      <AccountCard
        accountId={account.id}
        accountName={account.name}
        balance={account.balance}
        accountType={account.type}
        isActive={true}
        onViewDetails={() => console.log('View details')}
      />
    </div>
  );
}

// Child Component receives props object
function AccountCard(props) {
  console.log('Props received:', props);
  // {
  //   accountId: 'ACC001',
  //   accountName: 'John Doe',
  //   balance: 5000,
  //   accountType: 'SAVINGS',
  //   isActive: true,
  //   onViewDetails: [Function]
  // }

  return (
    <div className="account-card">
      <h2>{props.accountName}</h2>
      <p>ID: {props.accountId}</p>
      <p>Type: {props.accountType}</p>
      <p>Balance: ${props.balance}</p>
      <p>Status: {props.isActive ? 'Active' : 'Inactive'}</p>
      <button onClick={props.onViewDetails}>View Details</button>
    </div>
  );
}

// Better: Props Destructuring
function AccountCard({ 
  accountId, 
  accountName, 
  balance, 
  accountType, 
  isActive, 
  onViewDetails 
}) {
  return (
    <div className="account-card">
      <h2>{accountName}</h2>
      <p>ID: {accountId}</p>
      <p>Type: {accountType}</p>
      <p>Balance: ${balance}</p>
      <p>Status: {isActive ? 'Active' : 'Inactive'}</p>
      <button onClick={onViewDetails}>View Details</button>
    </div>
  );
}
```

#### Props Types

```jsx
/**
 * Different types of props you can pass
 */

function PropsExamples() {
  return (
    <>
      {/* String props */}
      <Component title="Hello" />
      <Component title={'Hello'} />

      {/* Number props */}
      <Component count={42} />
      <Component price={99.99} />

      {/* Boolean props */}
      <Component isActive={true} />
      <Component isActive />  {/* Shorthand for true */}
      <Component disabled={false} />

      {/* Array props */}
      <Component items={[1, 2, 3]} />
      <Component transactions={['TXN001', 'TXN002']} />

      {/* Object props */}
      <Component user={{ name: 'John', age: 30 }} />
      <Component account={{ id: 'ACC001', balance: 5000 }} />

      {/* Function props (callbacks) */}
      <Component onClick={() => console.log('clicked')} />
      <Component onSubmit={handleSubmit} />

      {/* JSX/Component props */}
      <Component icon={<IconComponent />} />
      <Component header={<h1>Title</h1>} />

      {/* Spread props */}
      <Component {...allProps} />
    </>
  );
}
```

#### Props Immutability

```jsx
/**
 * Props are READ-ONLY
 * Components must not modify their props
 */

// ❌ BAD: Mutating props
function BadComponent(props) {
  // NEVER DO THIS!
  props.balance = 6000;  // ❌ ERROR: Cannot modify props
  props.accountName = 'Jane';  // ❌ Violation of React rules
  
  return <div>{props.balance}</div>;
}

// ✅ GOOD: Props are read-only
function GoodComponent({ balance, accountName }) {
  // Props are read, never written
  const displayBalance = balance.toFixed(2);
  const greeting = `Hello, ${accountName}`;
  
  return (
    <div>
      <p>{greeting}</p>
      <p>Balance: ${displayBalance}</p>
    </div>
  );
}

// ✅ GOOD: If you need to modify, use state
function ComponentWithState({ initialBalance }) {
  // Create state from prop (initial value only)
  const [balance, setBalance] = useState(initialBalance);
  
  const addMoney = () => {
    setBalance(prevBalance => prevBalance + 100);  // Modify state, not props
  };
  
  return (
    <div>
      <p>Balance: ${balance}</p>
      <button onClick={addMoney}>Add $100</button>
    </div>
  );
}
```

---

### 🎯 Banking Example: Complete Component Hierarchy

```jsx
/**
 * Real-world banking application component structure
 * Demonstrates props flow through multiple levels
 */

// ============================================
// TOP LEVEL: App Component
// ============================================

function BankingApp() {
  const [accounts, setAccounts] = useState([
    {
      id: 'ACC001',
      name: 'John Doe',
      type: 'SAVINGS',
      balance: 5000,
      currency: 'USD',
      isActive: true
    },
    {
      id: 'ACC002',
      name: 'Jane Smith',
      type: 'CHECKING',
      balance: 3500,
      currency: 'USD',
      isActive: true
    },
    {
      id: 'ACC003',
      name: 'Bob Johnson',
      type: 'SAVINGS',
      balance: 12000,
      currency: 'USD',
      isActive: false
    }
  ]);

  const [selectedAccountId, setSelectedAccountId] = useState(null);

  const handleAccountSelect = (accountId) => {
    console.log('Account selected:', accountId);
    setSelectedAccountId(accountId);
  };

  const handleDeposit = (accountId, amount) => {
    setAccounts(prevAccounts =>
      prevAccounts.map(account =>
        account.id === accountId
          ? { ...account, balance: account.balance + amount }
          : account
      )
    );
  };

  return (
    <div className="banking-app">
      <header>
        <h1>Banking Application</h1>
      </header>
      
      {/* Pass props to Dashboard */}
      <Dashboard
        accounts={accounts}
        selectedAccountId={selectedAccountId}
        onAccountSelect={handleAccountSelect}
        onDeposit={handleDeposit}
      />
    </div>
  );
}

// ============================================
// LEVEL 2: Dashboard Component
// ============================================

function Dashboard({ accounts, selectedAccountId, onAccountSelect, onDeposit }) {
  // Calculate totals
  const totalBalance = accounts.reduce((sum, acc) => sum + acc.balance, 0);
  const activeAccounts = accounts.filter(acc => acc.isActive).length;

  return (
    <div className="dashboard">
      {/* Summary Section */}
      <DashboardSummary
        totalBalance={totalBalance}
        totalAccounts={accounts.length}
        activeAccounts={activeAccounts}
      />

      {/* Account List */}
      <AccountList
        accounts={accounts}
        selectedAccountId={selectedAccountId}
        onAccountSelect={onAccountSelect}
        onDeposit={onDeposit}
      />
    </div>
  );
}

// ============================================
// LEVEL 3: Dashboard Summary Component
// ============================================

function DashboardSummary({ totalBalance, totalAccounts, activeAccounts }) {
  return (
    <div className="dashboard-summary">
      <SummaryCard
        title="Total Balance"
        value={`$${totalBalance.toFixed(2)}`}
        icon="💰"
      />
      <SummaryCard
        title="Total Accounts"
        value={totalAccounts}
        icon="🏦"
      />
      <SummaryCard
        title="Active Accounts"
        value={activeAccounts}
        icon="✅"
      />
    </div>
  );
}

// ============================================
// LEVEL 4: Summary Card Component (Reusable)
// ============================================

function SummaryCard({ title, value, icon }) {
  return (
    <div className="summary-card">
      <span className="icon">{icon}</span>
      <div className="content">
        <h3>{title}</h3>
        <p className="value">{value}</p>
      </div>
    </div>
  );
}

// ============================================
// LEVEL 3: Account List Component
// ============================================

function AccountList({ accounts, selectedAccountId, onAccountSelect, onDeposit }) {
  return (
    <div className="account-list">
      <h2>Your Accounts</h2>
      {accounts.length === 0 ? (
        <p className="empty-message">No accounts found</p>
      ) : (
        <ul>
          {accounts.map(account => (
            <AccountListItem
              key={account.id}
              account={account}
              isSelected={account.id === selectedAccountId}
              onSelect={onAccountSelect}
              onDeposit={onDeposit}
            />
          ))}
        </ul>
      )}
    </div>
  );
}

// ============================================
// LEVEL 4: Account List Item Component
// ============================================

function AccountListItem({ account, isSelected, onSelect, onDeposit }) {
  const [showDepositForm, setShowDepositForm] = useState(false);

  const handleClick = () => {
    onSelect(account.id);
  };

  const handleDeposit = (amount) => {
    onDeposit(account.id, amount);
    setShowDepositForm(false);
  };

  return (
    <li
      className={`account-list-item ${isSelected ? 'selected' : ''} ${!account.isActive ? 'inactive' : ''}`}
      onClick={handleClick}
    >
      {/* Account Info */}
      <AccountInfo account={account} />

      {/* Action Buttons */}
      <div className="actions">
        <button
          onClick={(e) => {
            e.stopPropagation();
            setShowDepositForm(true);
          }}
          disabled={!account.isActive}
        >
          Deposit
        </button>
      </div>

      {/* Deposit Form (conditional) */}
      {showDepositForm && (
        <DepositForm
          accountId={account.id}
          onDeposit={handleDeposit}
          onCancel={() => setShowDepositForm(false)}
        />
      )}
    </li>
  );
}

// ============================================
// LEVEL 5: Account Info Component
// ============================================

function AccountInfo({ account }) {
  const { name, id, type, balance, currency, isActive } = account;

  return (
    <div className="account-info">
      <div className="account-header">
        <h3>{name}</h3>
        <span className={`badge ${type.toLowerCase()}`}>{type}</span>
      </div>
      <div className="account-details">
        <p className="account-id">
          <strong>Account:</strong> {id}
        </p>
        <p className="account-balance">
          <strong>Balance:</strong> {currency} ${balance.toFixed(2)}
        </p>
        <p className="account-status">
          <strong>Status:</strong>{' '}
          <span className={isActive ? 'active' : 'inactive'}>
            {isActive ? 'Active' : 'Inactive'}
          </span>
        </p>
      </div>
    </div>
  );
}

// ============================================
// LEVEL 5: Deposit Form Component
// ============================================

function DepositForm({ accountId, onDeposit, onCancel }) {
  const [amount, setAmount] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();

    const depositAmount = parseFloat(amount);

    // Validation
    if (isNaN(depositAmount) || depositAmount <= 0) {
      setError('Please enter a valid amount');
      return;
    }

    if (depositAmount > 10000) {
      setError('Maximum deposit amount is $10,000');
      return;
    }

    // Clear error and submit
    setError('');
    onDeposit(depositAmount);
    setAmount('');
  };

  return (
    <form className="deposit-form" onSubmit={handleSubmit} onClick={(e) => e.stopPropagation()}>
      <h4>Deposit Money</h4>
      <div className="form-group">
        <label htmlFor={`amount-${accountId}`}>Amount:</label>
        <input
          id={`amount-${accountId}`}
          type="number"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          placeholder="Enter amount"
          step="0.01"
          min="0"
          autoFocus
        />
      </div>
      {error && <p className="error-message">{error}</p>}
      <div className="form-actions">
        <button type="submit" className="btn-primary">
          Deposit
        </button>
        <button type="button" className="btn-secondary" onClick={onCancel}>
          Cancel
        </button>
      </div>
    </form>
  );
}

export default BankingApp;
```

**Props Flow Diagram**:

```
BankingApp (State: accounts, selectedAccountId)
  ↓ Props: accounts, selectedAccountId, onAccountSelect, onDeposit
Dashboard
  ↓ Props: totalBalance, totalAccounts, activeAccounts
DashboardSummary
  ↓ Props: title, value, icon
SummaryCard (Reusable × 3)

  ↓ Props: accounts, selectedAccountId, onAccountSelect, onDeposit
AccountList
  ↓ Props: account, isSelected, onSelect, onDeposit (for each account)
AccountListItem (Repeated for each account)
  ↓ Props: account
AccountInfo
  
  ↓ Props: accountId, onDeposit, onCancel
DepositForm
```

**Data Flow**:
```
User clicks "Deposit" button
  → DepositForm receives onDeposit prop (function)
  → User enters $500, clicks "Deposit"
  → onDeposit(500) called
  → Bubbles up to AccountListItem
  → Calls parent's onDeposit with accountId
  → Bubbles up to AccountList
  → Bubbles up to Dashboard
  → Bubbles up to BankingApp
  → BankingApp updates state (accounts array)
  → React re-renders entire tree with new props
  → AccountListItem receives updated account prop
  → Balance displays new value: $5500
```

---

### 🔄 Component Composition Patterns

#### 1. Children Prop

```jsx
/**
 * Special 'children' prop
 * Allows wrapping components with content
 */

// Wrapper component receives children
function Card({ children, title }) {
  return (
    <div className="card">
      {title && <h3 className="card-title">{title}</h3>}
      <div className="card-content">
        {children}  {/* Content passed between tags */}
      </div>
    </div>
  );
}

// Usage: Content between opening/closing tags
function AccountDashboard() {
  return (
    <div>
      {/* Children: Everything between <Card> tags */}
      <Card title="Account Summary">
        <p>Account: ACC001</p>
        <p>Balance: $5000</p>
        <button>View Details</button>
      </Card>

      {/* Another card with different children */}
      <Card title="Recent Transactions">
        <ul>
          <li>Deposit: $500</li>
          <li>Withdrawal: $200</li>
        </ul>
      </Card>
    </div>
  );
}

// Children can be anything: strings, JSX, components
<Card>Just text</Card>
<Card><p>JSX element</p></Card>
<Card><TransactionList /></Card>
<Card>
  <h2>Multiple</h2>
  <p>Elements</p>
</Card>
```

#### 2. Render Props Pattern

```jsx
/**
 * Pass a function as prop that returns JSX
 * Allows sharing logic between components
 */

// Component with render prop
function DataFetcher({ url, render }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, [url]);

  // Call render function with data
  return render({ data, loading });
}

// Usage: Pass function that renders UI
function App() {
  return (
    <DataFetcher
      url="/api/accounts"
      render={({ data, loading }) => {
        if (loading) return <p>Loading...</p>;
        if (!data) return <p>No data</p>;
        
        return (
          <ul>
            {data.map(account => (
              <li key={account.id}>{account.name}: ${account.balance}</li>
            ))}
          </ul>
        );
      }}
    />
  );
}
```

#### 3. Component Slots Pattern

```jsx
/**
 * Named slots for flexible layouts
 * Similar to children but multiple slots
 */

function AccountLayout({ header, sidebar, content, footer }) {
  return (
    <div className="account-layout">
      <header className="layout-header">{header}</header>
      <aside className="layout-sidebar">{sidebar}</aside>
      <main className="layout-content">{content}</main>
      <footer className="layout-footer">{footer}</footer>
    </div>
  );
}

// Usage: Pass different content to each slot
function AccountPage() {
  return (
    <AccountLayout
      header={
        <div>
          <h1>Account Dashboard</h1>
          <button>Logout</button>
        </div>
      }
      sidebar={
        <nav>
          <ul>
            <li>Overview</li>
            <li>Transactions</li>
            <li>Settings</li>
          </ul>
        </nav>
      }
      content={
        <div>
          <AccountSummary />
          <TransactionList />
        </div>
      }
      footer={
        <p>&copy; 2024 Banking App</p>
      }
    />
  );
}
```

---

### 🚨 Props Drilling Problem & Solutions

#### The Problem: Props Drilling

```jsx
/**
 * Props drilling: Passing props through multiple levels
 * Problem: Middle components don't need the props
 */

// ❌ PROBLEM: Props drilling through 4 levels

function App() {
  const [user, setUser] = useState({ name: 'John', role: 'admin' });

  return <Level1 user={user} />;
}

function Level1({ user }) {
  // Level1 doesn't use 'user', just passes it down
  return <Level2 user={user} />;
}

function Level2({ user }) {
  // Level2 doesn't use 'user', just passes it down
  return <Level3 user={user} />;
}

function Level3({ user }) {
  // Level3 doesn't use 'user', just passes it down
  return <Level4 user={user} />;
}

function Level4({ user }) {
  // Finally! Level4 uses 'user'
  return <div>Welcome, {user.name}</div>;
}
```

**Problems with Props Drilling**:
1. **Tight Coupling**: Middle components depend on props they don't use
2. **Refactoring Hell**: Changing prop name requires updating all levels
3. **Hard to Maintain**: Difficult to track where props come from
4. **Performance**: Unnecessary re-renders of middle components

#### Solution 1: Component Composition

```jsx
/**
 * ✅ BETTER: Use component composition
 * Pass the final component as children
 */

function App() {
  const [user, setUser] = useState({ name: 'John', role: 'admin' });

  return (
    <Level1>
      <Level2>
        <Level3>
          <Level4 user={user} />  {/* Only Level4 gets user */}
        </Level3>
      </Level2>
    </Level1>
  );
}

function Level1({ children }) {
  return <div className="level1">{children}</div>;
}

function Level2({ children }) {
  return <div className="level2">{children}</div>;
}

function Level3({ children }) {
  return <div className="level3">{children}</div>;
}

function Level4({ user }) {
  return <div>Welcome, {user.name}</div>;
}
```

#### Solution 2: Context API

```jsx
/**
 * ✅ BEST: Use Context API for global data
 * No props drilling needed
 */

import { createContext, useContext, useState } from 'react';

// Create context
const UserContext = createContext(null);

// Provider component
function App() {
  const [user, setUser] = useState({ name: 'John', role: 'admin' });

  return (
    <UserContext.Provider value={user}>
      <Level1 />
    </UserContext.Provider>
  );
}

// No props passed through middle components
function Level1() {
  return <Level2 />;
}

function Level2() {
  return <Level3 />;
}

function Level3() {
  return <Level4 />;
}

// Level4 directly accesses user from context
function Level4() {
  const user = useContext(UserContext);  // Direct access!
  return <div>Welcome, {user.name}</div>;
}
```

#### Banking Example: Props Drilling Solutions

```jsx
/**
 * Real-world banking example
 * Comparing props drilling vs context
 */

// ============================================
// ❌ PROPS DRILLING APPROACH
// ============================================

function BankingAppDrilling() {
  const [userAuth, setUserAuth] = useState({
    userId: 'U001',
    name: 'John Doe',
    role: 'customer',
    token: 'abc123'
  });

  return (
    <Dashboard userAuth={userAuth} />
  );
}

function Dashboard({ userAuth }) {
  return (
    <div>
      <Header userAuth={userAuth} />  {/* Doesn't use it */}
      <AccountSection userAuth={userAuth} />  {/* Doesn't use it */}
    </div>
  );
}

function Header({ userAuth }) {
  return (
    <header>
      <Logo />
      <Navigation userAuth={userAuth} />  {/* Doesn't use it */}
    </header>
  );
}

function Navigation({ userAuth }) {
  return (
    <nav>
      <Menu />
      <UserMenu userAuth={userAuth} />  {/* Finally uses it! */}
    </nav>
  );
}

function UserMenu({ userAuth }) {
  // 4 levels deep to use this prop!
  return <div>Hello, {userAuth.name}</div>;
}

// ============================================
// ✅ CONTEXT API APPROACH
// ============================================

const AuthContext = createContext(null);

function BankingAppContext() {
  const [userAuth, setUserAuth] = useState({
    userId: 'U001',
    name: 'John Doe',
    role: 'customer',
    token: 'abc123'
  });

  return (
    <AuthContext.Provider value={{ userAuth, setUserAuth }}>
      <Dashboard />  {/* No props! */}
    </AuthContext.Provider>
  );
}

function Dashboard() {
  return (
    <div>
      <Header />  {/* No props! */}
      <AccountSection />
    </div>
  );
}

function Header() {
  return (
    <header>
      <Logo />
      <Navigation />  {/* No props! */}
    </header>
  );
}

function Navigation() {
  return (
    <nav>
      <Menu />
      <UserMenu />  {/* No props! */}
    </nav>
  );
}

function UserMenu() {
  // Direct access to auth data!
  const { userAuth } = useContext(AuthContext);
  
  return <div>Hello, {userAuth.name}</div>;
}

// Any component can access auth data
function AccountSection() {
  const { userAuth } = useContext(AuthContext);
  
  // Use auth to fetch account data
  useEffect(() => {
    fetchAccounts(userAuth.userId, userAuth.token);
  }, [userAuth]);
  
  return <div>Accounts for user {userAuth.userId}</div>;
}
```

---

### 📊 Props Validation with TypeScript

```typescript
/**
 * Type-safe props with TypeScript
 * Catch errors at compile time
 */

// ============================================
// BASIC PROPS INTERFACE
// ============================================

interface AccountCardProps {
  accountId: string;
  accountName: string;
  balance: number;
  accountType: 'SAVINGS' | 'CHECKING' | 'CREDIT';  // Union type
  isActive: boolean;
}

const AccountCard: React.FC<AccountCardProps> = ({
  accountId,
  accountName,
  balance,
  accountType,
  isActive
}) => {
  return (
    <div className="account-card">
      <h3>{accountName}</h3>
      <p>ID: {accountId}</p>
      <p>Type: {accountType}</p>
      <p>Balance: ${balance.toFixed(2)}</p>
      <p>Status: {isActive ? 'Active' : 'Inactive'}</p>
    </div>
  );
};

// Usage with type checking
<AccountCard
  accountId="ACC001"
  accountName="John Doe"
  balance={5000}
  accountType="SAVINGS"  // Must be one of: SAVINGS, CHECKING, CREDIT
  isActive={true}
/>

// ❌ TypeScript will error:
<AccountCard
  accountId="ACC001"
  accountName="John Doe"
  balance="5000"  // ❌ Error: string not assignable to number
  accountType="INVESTMENT"  // ❌ Error: not in union type
  isActive={true}
/>

// ============================================
// OPTIONAL PROPS
// ============================================

interface TransactionProps {
  id: string;
  amount: number;
  date: Date;
  description?: string;  // Optional prop (can be undefined)
  category?: string;
}

const Transaction: React.FC<TransactionProps> = ({
  id,
  amount,
  date,
  description = 'No description',  // Default value
  category
}) => {
  return (
    <div className="transaction">
      <p>ID: {id}</p>
      <p>Amount: ${amount}</p>
      <p>Date: {date.toLocaleDateString()}</p>
      <p>Description: {description}</p>
      {category && <p>Category: {category}</p>}
    </div>
  );
};

// Valid: Optional props can be omitted
<Transaction
  id="TXN001"
  amount={500}
  date={new Date()}
/>

// ============================================
// COMPLEX PROPS (NESTED OBJECTS)
// ============================================

interface Address {
  street: string;
  city: string;
  state: string;
  zipCode: string;
}

interface Account {
  id: string;
  name: string;
  balance: number;
  type: 'SAVINGS' | 'CHECKING';
}

interface CustomerProps {
  customerId: string;
  name: string;
  email: string;
  address: Address;  // Nested object
  accounts: Account[];  // Array of objects
  preferences?: {  // Optional nested object
    newsletter: boolean;
    notifications: boolean;
  };
}

const CustomerProfile: React.FC<CustomerProps> = ({
  customerId,
  name,
  email,
  address,
  accounts,
  preferences
}) => {
  return (
    <div className="customer-profile">
      <h2>{name}</h2>
      <p>ID: {customerId}</p>
      <p>Email: {email}</p>
      
      <div className="address">
        <h3>Address</h3>
        <p>{address.street}</p>
        <p>{address.city}, {address.state} {address.zipCode}</p>
      </div>

      <div className="accounts">
        <h3>Accounts ({accounts.length})</h3>
        {accounts.map(account => (
          <div key={account.id}>
            <p>{account.name}: ${account.balance}</p>
          </div>
        ))}
      </div>

      {preferences && (
        <div className="preferences">
          <p>Newsletter: {preferences.newsletter ? 'Yes' : 'No'}</p>
          <p>Notifications: {preferences.notifications ? 'Yes' : 'No'}</p>
        </div>
      )}
    </div>
  );
};

// ============================================
// FUNCTION PROPS (CALLBACKS)
// ============================================

interface ButtonProps {
  label: string;
  onClick: (event: React.MouseEvent<HTMLButtonElement>) => void;
  onHover?: () => void;
  disabled?: boolean;
}

const Button: React.FC<ButtonProps> = ({
  label,
  onClick,
  onHover,
  disabled = false
}) => {
  return (
    <button
      onClick={onClick}
      onMouseEnter={onHover}
      disabled={disabled}
    >
      {label}
    </button>
  );
};

// Usage
<Button
  label="Transfer Money"
  onClick={(e) => console.log('Button clicked', e)}
  onHover={() => console.log('Button hovered')}
/>

// ============================================
// CHILDREN PROP TYPES
// ============================================

interface CardProps {
  title: string;
  children: React.ReactNode;  // Can be anything renderable
}

const Card: React.FC<CardProps> = ({ title, children }) => {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div className="card-content">{children}</div>
    </div>
  );
};

// Children can be: string, number, JSX, component, array
<Card title="Summary">
  <p>Text child</p>
  <AccountCard {...accountProps} />
  {accounts.map(a => <div key={a.id}>{a.name}</div>)}
</Card>

// ============================================
// GENERIC PROPS
// ============================================

interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  );
}

// Usage with Account type
<List
  items={accounts}
  renderItem={(account) => (
    <div>{account.name}: ${account.balance}</div>
  )}
  keyExtractor={(account) => account.id}
/>

// Usage with Transaction type
<List
  items={transactions}
  renderItem={(txn) => (
    <div>{txn.description}: ${txn.amount}</div>
  )}
  keyExtractor={(txn) => txn.id}
/>

// ============================================
// EXTENDING HTML ELEMENT PROPS
// ============================================

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const Input: React.FC<InputProps> = ({ label, error, ...inputProps }) => {
  return (
    <div className="input-group">
      <label>{label}</label>
      <input {...inputProps} />  {/* All HTML input props */}
      {error && <span className="error">{error}</span>}
    </div>
  );
};

// Can use all standard input props + custom props
<Input
  label="Account Number"
  type="text"
  placeholder="Enter account number"
  maxLength={10}
  required
  error="Invalid account number"
  onChange={(e) => console.log(e.target.value)}
/>
```

---

### 🎨 Advanced Props Patterns

#### 1. Props Spreading

```jsx
/**
 * Spread operator with props
 * Pass all props at once
 */

function ParentComponent() {
  const accountProps = {
    accountId: 'ACC001',
    accountName: 'John Doe',
    balance: 5000,
    accountType: 'SAVINGS',
    isActive: true
  };

  // Spread all props
  return <AccountCard {...accountProps} />;

  // Equivalent to:
  return (
    <AccountCard
      accountId="ACC001"
      accountName="John Doe"
      balance={5000}
      accountType="SAVINGS"
      isActive={true}
    />
  );
}

// Override specific props
function ParentWithOverride() {
  const defaultProps = {
    accountId: 'ACC001',
    accountName: 'John Doe',
    balance: 5000
  };

  return (
    <AccountCard
      {...defaultProps}
      balance={6000}  {/* Overrides balance from spread */}
    />
  );
}

// Forward all props to child (common in wrapper components)
function Wrapper(props) {
  return (
    <div className="wrapper">
      <ChildComponent {...props} />  {/* Forward everything */}
    </div>
  );
}
```

#### 2. Rest Parameters

```jsx
/**
 * Destructure some props, collect rest
 */

function AccountCard({ accountId, accountName, ...otherProps }) {
  console.log('ID:', accountId);
  console.log('Name:', accountName);
  console.log('Other props:', otherProps);
  // otherProps = { balance: 5000, accountType: 'SAVINGS', isActive: true }

  return (
    <div className="account-card">
      <h3>{accountName}</h3>
      <p>ID: {accountId}</p>
      {/* Pass remaining props to child */}
      <AccountDetails {...otherProps} />
    </div>
  );
}

// Separate known props from HTML attributes
function Button({ variant, size, ...htmlProps }) {
  const className = `btn btn-${variant} btn-${size}`;
  
  // htmlProps contains: onClick, disabled, type, etc.
  return <button className={className} {...htmlProps} />;
}

<Button
  variant="primary"
  size="large"
  onClick={handleClick}
  disabled={false}
  type="submit"
/>
```

#### 3. Conditional Props

```jsx
/**
 * Pass props conditionally
 */

function AccountList({ accounts, isAdmin }) {
  return (
    <div>
      {accounts.map(account => (
        <AccountCard
          key={account.id}
          accountId={account.id}
          accountName={account.name}
          balance={account.balance}
          // Only pass onDelete if user is admin
          {...(isAdmin && { onDelete: handleDelete })}
          // Or
          onEdit={isAdmin ? handleEdit : undefined}
        />
      ))}
    </div>
  );
}

// Conditional prop spreading
function ConditionalProps({ showDetails, account }) {
  return (
    <AccountCard
      accountId={account.id}
      {...(showDetails && {
        balance: account.balance,
        transactions: account.transactions,
        history: account.history
      })}
    />
  );
}
```

#### 4. Props with Defaults (Compound Pattern)

```jsx
/**
 * Default props with multiple fallback levels
 */

interface AccountCardProps {
  account: {
    id: string;
    name?: string;
    balance?: number;
    type?: string;
  };
  showBalance?: boolean;
  currency?: string;
}

const AccountCard: React.FC<AccountCardProps> = ({
  account,
  showBalance = true,  // Component default
  currency = 'USD'     // Component default
}) => {
  // Property defaults
  const {
    id,
    name = 'Unknown Account',  // Property default
    balance = 0,                // Property default
    type = 'CHECKING'           // Property default
  } = account;

  return (
    <div className="account-card">
      <h3>{name}</h3>
      <p>ID: {id}</p>
      <p>Type: {type}</p>
      {showBalance && (
        <p>Balance: {currency} ${balance.toFixed(2)}</p>
      )}
    </div>
  );
};

// Usage: All props optional except id
<AccountCard
  account={{ id: 'ACC001' }}  // Uses all defaults
/>

<AccountCard
  account={{ id: 'ACC002', name: 'Savings' }}  // Partial overrides
  currency="EUR"
/>
```

---

### 💡 Complete Production Example: Banking Transaction Manager

```typescript
/**
 * Production-grade banking component
 * Demonstrates all props concepts
 */

// ============================================
// TYPE DEFINITIONS
// ============================================

type TransactionType = 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER';
type TransactionStatus = 'PENDING' | 'COMPLETED' | 'FAILED';
type AccountType = 'SAVINGS' | 'CHECKING' | 'CREDIT';

interface Transaction {
  id: string;
  type: TransactionType;
  amount: number;
  date: Date;
  description: string;
  status: TransactionStatus;
  fromAccount?: string;
  toAccount?: string;
}

interface Account {
  id: string;
  name: string;
  type: AccountType;
  balance: number;
  currency: string;
  isActive: boolean;
}

interface User {
  id: string;
  name: string;
  email: string;
  role: 'customer' | 'admin';
}

// ============================================
// CONTEXT FOR GLOBAL STATE
// ============================================

interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = React.createContext<AuthContextType | null>(null);

function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// ============================================
// TOP LEVEL: APP COMPONENT
// ============================================

function BankingApp() {
  const [user, setUser] = useState<User>({
    id: 'U001',
    name: 'John Doe',
    email: 'john@example.com',
    role: 'customer'
  });

  const [isAuthenticated, setIsAuthenticated] = useState(true);

  const login = async (email: string, password: string) => {
    // Simulate API call
    console.log('Logging in:', email);
    setIsAuthenticated(true);
  };

  const logout = () => {
    setUser(null);
    setIsAuthenticated(false);
  };

  return (
    <AuthContext.Provider value={{ user, isAuthenticated, login, logout }}>
      <div className="banking-app">
        <AppHeader />
        <TransactionManager userId={user.id} />
      </div>
    </AuthContext.Provider>
  );
}

// ============================================
// APP HEADER COMPONENT (Uses Context)
// ============================================

function AppHeader() {
  const { user, logout } = useAuth();

  return (
    <header className="app-header">
      <div className="header-content">
        <h1>🏦 Banking Portal</h1>
        <div className="user-info">
          <span>Welcome, {user?.name}</span>
          <button onClick={logout} className="btn-logout">
            Logout
          </button>
        </div>
      </div>
    </header>
  );
}

// ============================================
// TRANSACTION MANAGER (Main Component)
// ============================================

interface TransactionManagerProps {
  userId: string;
}

function TransactionManager({ userId }: TransactionManagerProps) {
  const [accounts, setAccounts] = useState<Account[]>([
    {
      id: 'ACC001',
      name: 'Primary Checking',
      type: 'CHECKING',
      balance: 5000,
      currency: 'USD',
      isActive: true
    },
    {
      id: 'ACC002',
      name: 'Savings Account',
      type: 'SAVINGS',
      balance: 12000,
      currency: 'USD',
      isActive: true
    },
    {
      id: 'ACC003',
      name: 'Credit Card',
      type: 'CREDIT',
      balance: -500,
      currency: 'USD',
      isActive: true
    }
  ]);

  const [transactions, setTransactions] = useState<Transaction[]>([
    {
      id: 'TXN001',
      type: 'DEPOSIT',
      amount: 1000,
      date: new Date('2024-01-15'),
      description: 'Salary deposit',
      status: 'COMPLETED',
      toAccount: 'ACC001'
    },
    {
      id: 'TXN002',
      type: 'WITHDRAWAL',
      amount: 200,
      date: new Date('2024-01-16'),
      description: 'ATM Withdrawal',
      status: 'COMPLETED',
      fromAccount: 'ACC001'
    },
    {
      id: 'TXN003',
      type: 'TRANSFER',
      amount: 500,
      date: new Date('2024-01-17'),
      description: 'Transfer to savings',
      status: 'PENDING',
      fromAccount: 'ACC001',
      toAccount: 'ACC002'
    }
  ]);

  const [selectedAccountId, setSelectedAccountId] = useState<string | null>('ACC001');
  const [filterType, setFilterType] = useState<TransactionType | 'ALL'>('ALL');
  const [showNewTransactionForm, setShowNewTransactionForm] = useState(false);

  // Handlers
  const handleAccountSelect = useCallback((accountId: string) => {
    setSelectedAccountId(accountId);
    console.log('Selected account:', accountId);
  }, []);

  const handleTransactionCreate = useCallback((transaction: Omit<Transaction, 'id' | 'date' | 'status'>) => {
    const newTransaction: Transaction = {
      ...transaction,
      id: `TXN${Date.now()}`,
      date: new Date(),
      status: 'PENDING'
    };

    setTransactions(prev => [newTransaction, ...prev]);
    setShowNewTransactionForm(false);

    // Update account balances
    if (transaction.type === 'DEPOSIT' && transaction.toAccount) {
      updateAccountBalance(transaction.toAccount, transaction.amount);
    } else if (transaction.type === 'WITHDRAWAL' && transaction.fromAccount) {
      updateAccountBalance(transaction.fromAccount, -transaction.amount);
    } else if (transaction.type === 'TRANSFER' && transaction.fromAccount && transaction.toAccount) {
      updateAccountBalance(transaction.fromAccount, -transaction.amount);
      updateAccountBalance(transaction.toAccount, transaction.amount);
    }
  }, []);

  const updateAccountBalance = (accountId: string, amountChange: number) => {
    setAccounts(prev =>
      prev.map(account =>
        account.id === accountId
          ? { ...account, balance: account.balance + amountChange }
          : account
      )
    );
  };

  const handleTransactionDelete = useCallback((transactionId: string) => {
    setTransactions(prev => prev.filter(t => t.id !== transactionId));
  }, []);

  // Filter transactions
  const filteredTransactions = useMemo(() => {
    let filtered = transactions;

    // Filter by selected account
    if (selectedAccountId) {
      filtered = filtered.filter(
        t => t.fromAccount === selectedAccountId || t.toAccount === selectedAccountId
      );
    }

    // Filter by type
    if (filterType !== 'ALL') {
      filtered = filtered.filter(t => t.type === filterType);
    }

    return filtered;
  }, [transactions, selectedAccountId, filterType]);

  // Calculate statistics
  const statistics = useMemo(() => {
    const totalDeposits = transactions
      .filter(t => t.type === 'DEPOSIT' && t.status === 'COMPLETED')
      .reduce((sum, t) => sum + t.amount, 0);

    const totalWithdrawals = transactions
      .filter(t => t.type === 'WITHDRAWAL' && t.status === 'COMPLETED')
      .reduce((sum, t) => sum + t.amount, 0);

    const pendingCount = transactions.filter(t => t.status === 'PENDING').length;

    return { totalDeposits, totalWithdrawals, pendingCount };
  }, [transactions]);

  return (
    <div className="transaction-manager">
      {/* Account Summary */}
      <AccountSummary
        accounts={accounts}
        selectedAccountId={selectedAccountId}
        onAccountSelect={handleAccountSelect}
      />

      {/* Statistics */}
      <StatisticsPanel
        totalDeposits={statistics.totalDeposits}
        totalWithdrawals={statistics.totalWithdrawals}
        pendingTransactions={statistics.pendingCount}
        totalAccounts={accounts.length}
      />

      {/* Controls */}
      <TransactionControls
        filterType={filterType}
        onFilterChange={setFilterType}
        onNewTransaction={() => setShowNewTransactionForm(true)}
      />

      {/* New Transaction Form (conditional) */}
      {showNewTransactionForm && (
        <NewTransactionForm
          accounts={accounts}
          onSubmit={handleTransactionCreate}
          onCancel={() => setShowNewTransactionForm(false)}
        />
      )}

      {/* Transaction List */}
      <TransactionList
        transactions={filteredTransactions}
        accounts={accounts}
        onTransactionDelete={handleTransactionDelete}
      />
    </div>
  );
}

// ============================================
// ACCOUNT SUMMARY COMPONENT
// ============================================

interface AccountSummaryProps {
  accounts: Account[];
  selectedAccountId: string | null;
  onAccountSelect: (accountId: string) => void;
}

const AccountSummary = React.memo<AccountSummaryProps>(({
  accounts,
  selectedAccountId,
  onAccountSelect
}) => {
  console.log('AccountSummary rendered');

  return (
    <div className="account-summary">
      <h2>Your Accounts</h2>
      <div className="account-grid">
        {accounts.map(account => (
          <AccountCard
            key={account.id}
            account={account}
            isSelected={account.id === selectedAccountId}
            onClick={() => onAccountSelect(account.id)}
          />
        ))}
      </div>
    </div>
  );
});

// ============================================
// ACCOUNT CARD COMPONENT (Reusable)
// ============================================

interface AccountCardProps {
  account: Account;
  isSelected: boolean;
  onClick: () => void;
}

const AccountCard = React.memo<AccountCardProps>(({
  account,
  isSelected,
  onClick
}) => {
  const { name, type, balance, currency, isActive } = account;

  const getTypeIcon = (type: AccountType) => {
    switch (type) {
      case 'CHECKING': return '💳';
      case 'SAVINGS': return '🏦';
      case 'CREDIT': return '💰';
    }
  };

  const getBalanceColor = (balance: number) => {
    if (balance < 0) return 'red';
    if (balance < 1000) return 'orange';
    return 'green';
  };

  return (
    <div
      className={`account-card ${isSelected ? 'selected' : ''} ${!isActive ? 'inactive' : ''}`}
      onClick={onClick}
    >
      <div className="account-card-header">
        <span className="account-icon">{getTypeIcon(type)}</span>
        <span className="account-type">{type}</span>
      </div>
      <h3 className="account-name">{name}</h3>
      <p className="account-balance" style={{ color: getBalanceColor(balance) }}>
        {currency} ${Math.abs(balance).toFixed(2)}
        {balance < 0 && ' (Credit)'}
      </p>
      <span className={`status-badge ${isActive ? 'active' : 'inactive'}`}>
        {isActive ? 'Active' : 'Inactive'}
      </span>
    </div>
  );
});

// ============================================
// STATISTICS PANEL COMPONENT
// ============================================

interface StatisticsPanelProps {
  totalDeposits: number;
  totalWithdrawals: number;
  pendingTransactions: number;
  totalAccounts: number;
}

const StatisticsPanel = React.memo<StatisticsPanelProps>(({
  totalDeposits,
  totalWithdrawals,
  pendingTransactions,
  totalAccounts
}) => {
  return (
    <div className="statistics-panel">
      <StatCard
        title="Total Deposits"
        value={`$${totalDeposits.toFixed(2)}`}
        icon="📈"
        color="green"
      />
      <StatCard
        title="Total Withdrawals"
        value={`$${totalWithdrawals.toFixed(2)}`}
        icon="📉"
        color="red"
      />
      <StatCard
        title="Pending"
        value={pendingTransactions}
        icon="⏳"
        color="orange"
      />
      <StatCard
        title="Accounts"
        value={totalAccounts}
        icon="🏦"
        color="blue"
      />
    </div>
  );
});

// ============================================
// STAT CARD COMPONENT (Reusable)
// ============================================

interface StatCardProps {
  title: string;
  value: string | number;
  icon: string;
  color: string;
}

const StatCard: React.FC<StatCardProps> = ({ title, value, icon, color }) => {
  return (
    <div className="stat-card" style={{ borderColor: color }}>
      <span className="stat-icon">{icon}</span>
      <div className="stat-content">
        <p className="stat-title">{title}</p>
        <p className="stat-value" style={{ color }}>
          {value}
        </p>
      </div>
    </div>
  );
};

// ============================================
// TRANSACTION CONTROLS COMPONENT
// ============================================

interface TransactionControlsProps {
  filterType: TransactionType | 'ALL';
  onFilterChange: (type: TransactionType | 'ALL') => void;
  onNewTransaction: () => void;
}

const TransactionControls: React.FC<TransactionControlsProps> = ({
  filterType,
  onFilterChange,
  onNewTransaction
}) => {
  return (
    <div className="transaction-controls">
      <div className="filter-buttons">
        <button
          className={filterType === 'ALL' ? 'active' : ''}
          onClick={() => onFilterChange('ALL')}
        >
          All
        </button>
        <button
          className={filterType === 'DEPOSIT' ? 'active' : ''}
          onClick={() => onFilterChange('DEPOSIT')}
        >
          Deposits
        </button>
        <button
          className={filterType === 'WITHDRAWAL' ? 'active' : ''}
          onClick={() => onFilterChange('WITHDRAWAL')}
        >
          Withdrawals
        </button>
        <button
          className={filterType === 'TRANSFER' ? 'active' : ''}
          onClick={() => onFilterChange('TRANSFER')}
        >
          Transfers
        </button>
      </div>
      <button className="btn-new-transaction" onClick={onNewTransaction}>
        + New Transaction
      </button>
    </div>
  );
};

// ============================================
// NEW TRANSACTION FORM COMPONENT
// ============================================

interface NewTransactionFormProps {
  accounts: Account[];
  onSubmit: (transaction: Omit<Transaction, 'id' | 'date' | 'status'>) => void;
  onCancel: () => void;
}

const NewTransactionForm: React.FC<NewTransactionFormProps> = ({
  accounts,
  onSubmit,
  onCancel
}) => {
  const [type, setType] = useState<TransactionType>('DEPOSIT');
  const [amount, setAmount] = useState('');
  const [description, setDescription] = useState('');
  const [fromAccount, setFromAccount] = useState('');
  const [toAccount, setToAccount] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    const amountNum = parseFloat(amount);

    // Validation
    if (isNaN(amountNum) || amountNum <= 0) {
      setError('Please enter a valid amount');
      return;
    }

    if (type === 'DEPOSIT' && !toAccount) {
      setError('Please select destination account');
      return;
    }

    if (type === 'WITHDRAWAL' && !fromAccount) {
      setError('Please select source account');
      return;
    }

    if (type === 'TRANSFER' && (!fromAccount || !toAccount)) {
      setError('Please select both accounts');
      return;
    }

    if (type === 'TRANSFER' && fromAccount === toAccount) {
      setError('Cannot transfer to the same account');
      return;
    }

    // Submit
    onSubmit({
      type,
      amount: amountNum,
      description: description || `${type} transaction`,
      fromAccount: fromAccount || undefined,
      toAccount: toAccount || undefined
    });
  };

  return (
    <div className="new-transaction-form-overlay">
      <form className="new-transaction-form" onSubmit={handleSubmit}>
        <h3>New Transaction</h3>

        {/* Transaction Type */}
        <div className="form-group">
          <label>Type:</label>
          <select value={type} onChange={(e) => setType(e.target.value as TransactionType)}>
            <option value="DEPOSIT">Deposit</option>
            <option value="WITHDRAWAL">Withdrawal</option>
            <option value="TRANSFER">Transfer</option>
          </select>
        </div>

        {/* Amount */}
        <div className="form-group">
          <label>Amount:</label>
          <input
            type="number"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
            placeholder="Enter amount"
            step="0.01"
            min="0"
          />
        </div>

        {/* Description */}
        <div className="form-group">
          <label>Description:</label>
          <input
            type="text"
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            placeholder="Enter description (optional)"
          />
        </div>

        {/* From Account (for WITHDRAWAL and TRANSFER) */}
        {(type === 'WITHDRAWAL' || type === 'TRANSFER') && (
          <div className="form-group">
            <label>From Account:</label>
            <select value={fromAccount} onChange={(e) => setFromAccount(e.target.value)}>
              <option value="">Select account</option>
              {accounts
                .filter(a => a.isActive)
                .map(account => (
                  <option key={account.id} value={account.id}>
                    {account.name} (${account.balance.toFixed(2)})
                  </option>
                ))}
            </select>
          </div>
        )}

        {/* To Account (for DEPOSIT and TRANSFER) */}
        {(type === 'DEPOSIT' || type === 'TRANSFER') && (
          <div className="form-group">
            <label>To Account:</label>
            <select value={toAccount} onChange={(e) => setToAccount(e.target.value)}>
              <option value="">Select account</option>
              {accounts
                .filter(a => a.isActive && a.id !== fromAccount)
                .map(account => (
                  <option key={account.id} value={account.id}>
                    {account.name} (${account.balance.toFixed(2)})
                  </option>
                ))}
            </select>
          </div>
        )}

        {/* Error Message */}
        {error && <p className="error-message">{error}</p>}

        {/* Actions */}
        <div className="form-actions">
          <button type="submit" className="btn-primary">
            Create Transaction
          </button>
          <button type="button" className="btn-secondary" onClick={onCancel}>
            Cancel
          </button>
        </div>
      </form>
    </div>
  );
};

// ============================================
// TRANSACTION LIST COMPONENT
// ============================================

interface TransactionListProps {
  transactions: Transaction[];
  accounts: Account[];
  onTransactionDelete: (transactionId: string) => void;
}

const TransactionList = React.memo<TransactionListProps>(({
  transactions,
  accounts,
  onTransactionDelete
}) => {
  console.log('TransactionList rendered with', transactions.length, 'transactions');

  const getAccountName = (accountId: string | undefined) => {
    if (!accountId) return 'N/A';
    const account = accounts.find(a => a.id === accountId);
    return account ? account.name : accountId;
  };

  if (transactions.length === 0) {
    return (
      <div className="transaction-list empty">
        <p>No transactions found</p>
      </div>
    );
  }

  return (
    <div className="transaction-list">
      <h2>Transactions ({transactions.length})</h2>
      <div className="transaction-items">
        {transactions.map(transaction => (
          <TransactionItem
            key={transaction.id}
            transaction={transaction}
            getAccountName={getAccountName}
            onDelete={() => onTransactionDelete(transaction.id)}
          />
        ))}
      </div>
    </div>
  );
});

// ============================================
// TRANSACTION ITEM COMPONENT
// ============================================

interface TransactionItemProps {
  transaction: Transaction;
  getAccountName: (accountId: string | undefined) => string;
  onDelete: () => void;
}

const TransactionItem = React.memo<TransactionItemProps>(({
  transaction,
  getAccountName,
  onDelete
}) => {
  const { id, type, amount, date, description, status, fromAccount, toAccount } = transaction;

  const getTypeIcon = (type: TransactionType) => {
    switch (type) {
      case 'DEPOSIT': return '📥';
      case 'WITHDRAWAL': return '📤';
      case 'TRANSFER': return '🔄';
    }
  };

  const getStatusColor = (status: TransactionStatus) => {
    switch (status) {
      case 'COMPLETED': return 'green';
      case 'PENDING': return 'orange';
      case 'FAILED': return 'red';
    }
  };

  return (
    <div className="transaction-item">
      <div className="transaction-icon">{getTypeIcon(type)}</div>
      <div className="transaction-details">
        <div className="transaction-header">
          <h4>{description}</h4>
          <span className={`status-badge ${status.toLowerCase()}`} style={{ backgroundColor: getStatusColor(status) }}>
            {status}
          </span>
        </div>
        <div className="transaction-info">
          <p className="transaction-type">{type}</p>
          <p className="transaction-date">{date.toLocaleDateString()}</p>
          {fromAccount && <p className="transaction-from">From: {getAccountName(fromAccount)}</p>}
          {toAccount && <p className="transaction-to">To: {getAccountName(toAccount)}</p>}
        </div>
      </div>
      <div className="transaction-amount">
        <p className={type === 'WITHDRAWAL' ? 'negative' : 'positive'}>
          {type === 'WITHDRAWAL' ? '-' : '+'}${amount.toFixed(2)}
        </p>
      </div>
      <button className="btn-delete" onClick={onDelete} title="Delete transaction">
        🗑️
      </button>
    </div>
  );
});

export default BankingApp;
```

**Component Hierarchy**:
```
BankingApp (State + Context)
├── AuthContext.Provider
│   ├── AppHeader (useContext)
│   └── TransactionManager (Props: userId)
│       ├── AccountSummary (Props: accounts, selectedAccountId, onAccountSelect)
│       │   └── AccountCard × N (Props: account, isSelected, onClick)
│       ├── StatisticsPanel (Props: totalDeposits, totalWithdrawals, etc.)
│       │   └── StatCard × 4 (Props: title, value, icon, color)
│       ├── TransactionControls (Props: filterType, onFilterChange, onNewTransaction)
│       ├── NewTransactionForm (Conditional, Props: accounts, onSubmit, onCancel)
│       └── TransactionList (Props: transactions, accounts, onTransactionDelete)
│           └── TransactionItem × N (Props: transaction, getAccountName, onDelete)
```

**Props Patterns Used**:
1. ✅ **Context API** - AuthContext for user authentication (no props drilling)
2. ✅ **TypeScript** - Complete type safety for all props
3. ✅ **Destructuring** - Clean prop extraction in all components
4. ✅ **Memoization** - React.memo on expensive list components
5. ✅ **Callbacks** - useCallback for stable function references
6. ✅ **Derived State** - useMemo for computed values (statistics, filtered lists)
7. ✅ **Conditional Rendering** - Form only shown when needed
8. ✅ **Reusable Components** - AccountCard, StatCard, TransactionItem
9. ✅ **Composition** - Children props for flexible layouts
10. ✅ **Spread Props** - Efficient prop passing
11. ✅ **Event Handlers** - onClick, onSubmit, onChange with proper typing
12. ✅ **Validation** - Form validation with error messages

---

### ✅ DO's - Best Practices

#### 1. ✅ DO: Always Destructure Props

```jsx
// ✅ GOOD: Clean and readable
function AccountCard({ accountId, accountName, balance }) {
  return (
    <div>
      <h3>{accountName}</h3>
      <p>Balance: ${balance}</p>
    </div>
  );
}

// ❌ BAD: Verbose and repetitive
function AccountCard(props) {
  return (
    <div>
      <h3>{props.accountName}</h3>
      <p>Balance: ${props.balance}</p>
    </div>
  );
}
```

#### 2. ✅ DO: Use TypeScript for Type Safety

```typescript
// ✅ GOOD: Type-safe props
interface AccountProps {
  accountId: string;
  balance: number;
  isActive: boolean;
}

const Account: React.FC<AccountProps> = ({ accountId, balance, isActive }) => {
  // Compiler ensures correct prop types
  return <div>Account {accountId}: ${balance.toFixed(2)}</div>;
};
```

#### 3. ✅ DO: Provide Default Props

```jsx
// ✅ GOOD: Safe defaults
function AccountCard({ 
  accountName = 'Unknown', 
  balance = 0, 
  currency = 'USD' 
}) {
  return (
    <div>
      <h3>{accountName}</h3>
      <p>{currency} ${balance.toFixed(2)}</p>
    </div>
  );
}
```

#### 4. ✅ DO: Use Prop Drilling Solutions (Context/Composition)

```jsx
// ✅ GOOD: Context API eliminates prop drilling
const UserContext = createContext();

function App() {
  const user = { name: 'John', role: 'admin' };
  return (
    <UserContext.Provider value={user}>
      <Dashboard />
    </UserContext.Provider>
  );
}

function ProfileButton() {
  const user = useContext(UserContext);  // Direct access
  return <button>{user.name}</button>;
}
```

#### 5. ✅ DO: Memoize Expensive Components

```jsx
// ✅ GOOD: Prevent unnecessary re-renders
const AccountList = React.memo(({ accounts }) => {
  console.log('AccountList rendered');
  return (
    <div>
      {accounts.map(account => (
        <AccountCard key={account.id} account={account} />
      ))}
    </div>
  );
});
```

#### 6. ✅ DO: Use Callback Props for Parent Communication

```jsx
// ✅ GOOD: Child communicates with parent
function AccountCard({ account, onSelect, onDelete }) {
  return (
    <div>
      <h3>{account.name}</h3>
      <button onClick={() => onSelect(account.id)}>Select</button>
      <button onClick={() => onDelete(account.id)}>Delete</button>
    </div>
  );
}

// Parent provides callbacks
function Dashboard() {
  const handleSelect = (id) => console.log('Selected:', id);
  const handleDelete = (id) => console.log('Deleted:', id);

  return (
    <AccountCard
      account={account}
      onSelect={handleSelect}
      onDelete={handleDelete}
    />
  );
}
```

#### 7. ✅ DO: Use Children Prop for Composition

```jsx
// ✅ GOOD: Flexible component composition
function Card({ title, children }) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div className="card-content">{children}</div>
    </div>
  );
}

// Usage: Can pass any content
<Card title="Account Summary">
  <p>Balance: $5000</p>
  <button>View Details</button>
</Card>
```

#### 8. ✅ DO: Spread Props When Forwarding

```jsx
// ✅ GOOD: Forward all HTML attributes
function Button({ variant, children, ...htmlProps }) {
  return (
    <button className={`btn btn-${variant}`} {...htmlProps}>
      {children}
    </button>
  );
}

// All HTML button props work
<Button variant="primary" onClick={handleClick} disabled type="submit">
  Submit
</Button>
```

#### 9. ✅ DO: Validate Props in Development

```jsx
// ✅ GOOD: PropTypes for runtime validation (JavaScript)
import PropTypes from 'prop-types';

function AccountCard({ accountId, balance }) {
  return <div>Account {accountId}: ${balance}</div>;
}

AccountCard.propTypes = {
  accountId: PropTypes.string.isRequired,
  balance: PropTypes.number.isRequired
};
```

#### 10. ✅ DO: Keep Components Pure

```jsx
// ✅ GOOD: Pure component (same props = same output)
function AccountBalance({ balance }) {
  const formatted = balance.toFixed(2);  // Pure calculation
  return <div>Balance: ${formatted}</div>;
}

// Component doesn't:
// - Mutate props
// - Make API calls directly
// - Access global state directly
// - Have side effects in render
```

---

### ❌ DON'Ts - Anti-Patterns

#### 1. ❌ DON'T: Mutate Props

```jsx
// ❌ BAD: Props are immutable!
function AccountCard(props) {
  props.balance = props.balance + 100;  // ❌ ERROR: Never mutate props
  return <div>Balance: ${props.balance}</div>;
}

// ✅ GOOD: Use state if you need to modify
function AccountCard({ initialBalance }) {
  const [balance, setBalance] = useState(initialBalance);
  
  const addMoney = () => {
    setBalance(prev => prev + 100);  // Modify state, not props
  };
  
  return (
    <div>
      <div>Balance: ${balance}</div>
      <button onClick={addMoney}>Add $100</button>
    </div>
  );
}
```

#### 2. ❌ DON'T: Create Components Inside Render

```jsx
// ❌ BAD: New component created on every render
function Dashboard({ accounts }) {
  // New AccountCard component on each render!
  const AccountCard = ({ account }) => <div>{account.name}</div>;
  
  return (
    <div>
      {accounts.map(account => (
        <AccountCard key={account.id} account={account} />
      ))}
    </div>
  );
}

// ✅ GOOD: Define component outside
const AccountCard = ({ account }) => <div>{account.name}</div>;

function Dashboard({ accounts }) {
  return (
    <div>
      {accounts.map(account => (
        <AccountCard key={account.id} account={account} />
      ))}
    </div>
  );
}
```

#### 3. ❌ DON'T: Pass Too Many Props (Code Smell)

```jsx
// ❌ BAD: Too many individual props
function AccountCard({
  accountId,
  accountName,
  accountType,
  balance,
  currency,
  isActive,
  createdAt,
  updatedAt,
  ownerId,
  ownerName,
  ownerEmail
}) {
  // 11 props is too many!
}

// ✅ GOOD: Group related props into objects
function AccountCard({ account, owner }) {
  const { accountId, accountName, accountType, balance, currency, isActive } = account;
  const { ownerId, ownerName, ownerEmail } = owner;
  
  return <div>{accountName}: ${balance}</div>;
}

// Usage
<AccountCard
  account={accountData}
  owner={ownerData}
/>
```

#### 4. ❌ DON'T: Use Inline Objects/Arrays as Props

```jsx
// ❌ BAD: New object created on every render
function Dashboard() {
  return (
    <AccountCard
      account={{ id: 'ACC001', name: 'Checking' }}  // ❌ New object every render
      styles={{ color: 'blue' }}                     // ❌ New object every render
    />
  );
}

// ✅ GOOD: Define outside or use useMemo
function Dashboard() {
  const account = useMemo(() => ({ 
    id: 'ACC001', 
    name: 'Checking' 
  }), []);
  
  const styles = useMemo(() => ({ 
    color: 'blue' 
  }), []);
  
  return <AccountCard account={account} styles={styles} />;
}
```

#### 5. ❌ DON'T: Drill Props Through Many Levels

```jsx
// ❌ BAD: Props drilling through 5 levels
function App() {
  const user = { name: 'John' };
  return <Level1 user={user} />;
}

function Level1({ user }) {
  return <Level2 user={user} />;  // Just passing through
}

function Level2({ user }) {
  return <Level3 user={user} />;  // Just passing through
}

function Level3({ user }) {
  return <Level4 user={user} />;  // Just passing through
}

function Level4({ user }) {
  return <Level5 user={user} />;  // Just passing through
}

function Level5({ user }) {
  return <div>{user.name}</div>;  // Finally uses it!
}

// ✅ GOOD: Use Context API
const UserContext = createContext();

function App() {
  const user = { name: 'John' };
  return (
    <UserContext.Provider value={user}>
      <Level1 />
    </UserContext.Provider>
  );
}

function Level5() {
  const user = useContext(UserContext);  // Direct access!
  return <div>{user.name}</div>;
}
```

#### 6. ❌ DON'T: Use Spread Props Blindly

```jsx
// ❌ BAD: Spreading unknown props can cause issues
function AccountCard({ account, ...rest }) {
  // What's in 'rest'? Unknown props passed down
  return <div {...rest}>{account.name}</div>;
}

// Might receive invalid HTML attributes
<AccountCard account={account} invalidProp="value" />

// ✅ GOOD: Be explicit about what you accept
function AccountCard({ account, className, onClick }) {
  return (
    <div className={className} onClick={onClick}>
      {account.name}
    </div>
  );
}
```

#### 7. ❌ DON'T: Ignore PropTypes/TypeScript Warnings

```jsx
// ❌ BAD: Ignoring type errors
// @ts-ignore
<AccountCard balance="5000" />  // Should be number, not string

// ✅ GOOD: Fix the type issue
<AccountCard balance={5000} />  // Correct type
```

#### 8. ❌ DON'T: Use Index as Key When Mapping with Props

```jsx
// ❌ BAD: Index as key causes issues when list changes
function AccountList({ accounts }) {
  return (
    <div>
      {accounts.map((account, index) => (
        <AccountCard key={index} account={account} />  // ❌ Bad key
      ))}
    </div>
  );
}

// ✅ GOOD: Use stable unique identifier
function AccountList({ accounts }) {
  return (
    <div>
      {accounts.map(account => (
        <AccountCard key={account.id} account={account} />  // ✅ Good key
      ))}
    </div>
  );
}
```

#### 9. ❌ DON'T: Pass Unstable Callbacks

```jsx
// ❌ BAD: New function created on every render
function Dashboard({ accounts }) {
  return (
    <div>
      {accounts.map(account => (
        <AccountCard
          key={account.id}
          account={account}
          onClick={() => console.log(account.id)}  // ❌ New function every render
        />
      ))}
    </div>
  );
}

// ✅ GOOD: Use useCallback for stable reference
function Dashboard({ accounts }) {
  const handleClick = useCallback((accountId) => {
    console.log(accountId);
  }, []);
  
  return (
    <div>
      {accounts.map(account => (
        <AccountCard
          key={account.id}
          account={account}
          onClick={() => handleClick(account.id)}
        />
      ))}
    </div>
  );
}
```

#### 10. ❌ DON'T: Use Props for Everything (Use State When Needed)

```jsx
// ❌ BAD: Trying to control everything with props
function AccountCard({ balance, onBalanceChange }) {
  // Component can't manage its own UI state
  return (
    <div>
      <div>Balance: ${balance}</div>
      <button onClick={() => onBalanceChange(balance + 100)}>
        Add $100
      </button>
    </div>
  );
}

// ✅ GOOD: Use local state for UI concerns
function AccountCard({ initialBalance, onSave }) {
  const [balance, setBalance] = useState(initialBalance);
  const [isEditing, setIsEditing] = useState(false);  // Local UI state
  
  const handleSave = () => {
    onSave(balance);  // Notify parent of final value
    setIsEditing(false);
  };
  
  return (
    <div>
      {isEditing ? (
        <>
          <input
            value={balance}
            onChange={(e) => setBalance(Number(e.target.value))}
          />
          <button onClick={handleSave}>Save</button>
        </>
      ) : (
        <>
          <div>Balance: ${balance}</div>
          <button onClick={() => setIsEditing(true)}>Edit</button>
        </>
      )}
    </div>
  );
}
```

---

### 🎯 Key Takeaways

#### **Component Fundamentals** (5 Points)
1. **Function vs Class**: Function components are simpler, support Hooks, and are recommended by React team
2. **Modern Approach**: Arrow functions with TypeScript provide clean, type-safe components
3. **Pure Components**: Components should be pure functions (same props → same output)
4. **Composition**: Build complex UIs by composing small, reusable components
5. **Lifecycle**: Function components use Hooks (useEffect) instead of class lifecycle methods

#### **Props Core Concepts** (5 Points)
6. **Props = Parameters**: Props work exactly like function parameters, passed from parent to child
7. **Immutability**: Props are READ-ONLY; components must never mutate their props
8. **Unidirectional Flow**: Data flows down from parent to child (one-way binding)
9. **Any Type**: Props can be strings, numbers, booleans, objects, arrays, functions, or JSX
10. **Destructuring**: Always destructure props for cleaner, more readable code

#### **Advanced Props Patterns** (5 Points)
11. **Children Prop**: Special prop that allows wrapping components with content
12. **Render Props**: Pass functions as props that return JSX for flexible logic sharing
13. **Spread Operator**: Use `{...props}` to pass multiple props efficiently
14. **Rest Parameters**: Destructure some props, collect rest with `...otherProps`
15. **Conditional Props**: Pass props conditionally using `&&` or ternary operators

#### **Props Drilling & Solutions** (5 Points)
16. **Props Drilling Problem**: Passing props through multiple levels that don't use them
17. **Context API Solution**: Best solution for global data (auth, theme, user)
18. **Composition Solution**: Pass components as children to avoid drilling
19. **Tight Coupling Issue**: Props drilling creates tight coupling between components
20. **Performance Impact**: Unnecessary props cause re-renders in middle components

#### **Type Safety** (5 Points)
21. **TypeScript**: Provides compile-time type checking for props
22. **Interfaces**: Define prop types with interfaces for complex objects
23. **Optional Props**: Use `?` for optional props with default values
24. **Union Types**: Restrict props to specific values (`'SAVINGS' | 'CHECKING'`)
25. **Generic Components**: Use generics for reusable components with any type

#### **Performance Optimization** (5 Points)
26. **React.memo**: Prevent re-renders when props haven't changed
27. **useCallback**: Stable function references for callback props
28. **useMemo**: Memoize expensive calculations and complex objects
29. **Avoid Inline Objects**: Don't create new objects in JSX (causes re-renders)
30. **Keys in Lists**: Always use stable unique keys when mapping arrays

#### **Best Practices** (5 Points)
31. **Default Props**: Provide sensible defaults for optional props
32. **Prop Validation**: Use TypeScript or PropTypes for runtime validation
33. **Meaningful Names**: Use descriptive prop names (onAccountSelect vs onClick)
34. **Group Related Props**: Pass objects for related data instead of many individual props
35. **Documentation**: Use JSDoc comments to document complex prop types

#### **Banking Application Insights** (5 Points)
36. **Account Hierarchy**: Top-level state for accounts, props flow down to display components
37. **Callback Communication**: Child components use callbacks to notify parent of actions
38. **Form State**: Use local state for forms, notify parent only on submit
39. **Context for Auth**: User authentication data shared via Context (no drilling)
40. **Memoization Critical**: Banking UIs update frequently; memoize lists and calculations

---

### 📚 Common Interview Questions

**Q1: What's the difference between props and state?**

**Answer**:
- **Props**: Data passed FROM parent TO child. Read-only, immutable, controlled by parent.
- **State**: Data managed WITHIN component. Mutable, triggers re-renders, controlled by component itself.

```jsx
// Props: Passed from parent
function Parent() {
  return <Child name="John" age={30} />;  // Props
}

function Child({ name, age }) {  // Receives props
  const [count, setCount] = useState(0);  // Internal state
  return <div>{name} clicked {count} times</div>;
}
```

**Q2: Can you modify props in a component?**

**Answer**: No! Props are READ-ONLY. Attempting to modify props violates React's unidirectional data flow principle and will cause errors.

```jsx
// ❌ WRONG
function Component(props) {
  props.value = 10;  // ❌ ERROR: Cannot assign to read-only property
}

// ✅ CORRECT: Use state
function Component({ initialValue }) {
  const [value, setValue] = useState(initialValue);
  const update = () => setValue(10);  // Modify state, not props
}
```

**Q3: How do you pass a function as a prop?**

**Answer**: Pass functions like any other prop. Child calls the function to communicate with parent.

```jsx
// Parent passes function
function Parent() {
  const handleClick = (id) => {
    console.log('Account selected:', id);
  };

  return <Child onSelect={handleClick} />;
}

// Child receives and calls function
function Child({ onSelect }) {
  return (
    <button onClick={() => onSelect('ACC001')}>
      Select Account
    </button>
  );
}
```

**Q4: What is props drilling and how do you solve it?**

**Answer**: Props drilling is passing props through multiple component levels that don't use them. Solutions:

1. **Context API** - Best for global data
2. **Component Composition** - Pass components as children
3. **State Management** - Redux, Zustand for complex apps

```jsx
// Context API solution
const UserContext = createContext();

function App() {
  return (
    <UserContext.Provider value={user}>
      <DeepChild />  {/* No props! */}
    </UserContext.Provider>
  );
}

function DeepChild() {
  const user = useContext(UserContext);  // Direct access
  return <div>{user.name}</div>;
}
```

**Q5: What are children props?**

**Answer**: `children` is a special prop that contains content between opening and closing tags of a component. Used for composition patterns.

```jsx
// Component receives children
function Card({ title, children }) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div>{children}</div>  {/* Content passed here */}
    </div>
  );
}

// Usage: Content between tags becomes children
<Card title="Account">
  <p>Balance: $5000</p>
  <button>View Details</button>
</Card>
```

---

### 🔗 Related Topics to Explore

1. **Q03: State Management (useState)** - Managing component internal state
2. **Q04: Lifecycle Methods** - Component mounting, updating, unmounting with useEffect
3. **Q05: React Hooks Basics** - useState, useEffect, useContext
4. **Q11: Context API** - Deep dive into global state management
5. **Q13: Custom Hooks** - Creating reusable stateful logic
6. **Q14: Performance Optimization** - useMemo, useCallback, React.memo deep dive

---

**End of Q02: Components & Props**

