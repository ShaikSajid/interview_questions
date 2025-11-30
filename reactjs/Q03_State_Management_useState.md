# React Interview Question 3: State Management with useState
## Understanding React State, useState Hook, and State Management Patterns

---

## 📋 Summary of What Will Be Covered

### ✅ Question 3: State Management with useState Hook
- What is state and why it's needed
- useState Hook fundamentals
- State updates and re-renders
- Functional updates for state
- Multiple state variables vs single object
- State initialization and lazy initialization
- Banking Example: Account balance management with transactions

---

## ❓ Question 3: What is React State and how does the useState Hook work?

### 📘 Comprehensive Explanation

**React State**:

State is data that changes over time and affects what gets rendered on the screen. When state changes, React re-renders the component to reflect the new data.

**Key Concepts**:
- State is **component-specific** (each instance has its own state)
- State changes trigger **re-renders**
- State updates are **asynchronous**
- State should be treated as **immutable**
- State persists between re-renders (unlike regular variables)

**useState Hook**:

The `useState` Hook is the primary way to add state to function components. It returns a pair: the current state value and a function to update it.

```javascript
const [state, setState] = useState(initialValue);
```

---

### 🏗️ State Management Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                   REACT STATE LIFECYCLE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: Component Initial Render                               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ function AccountBalance() {                               │ │
│  │   const [balance, setBalance] = useState(5000);           │ │
│  │   //     ↑         ↑                       ↑              │ │
│  │   //   current   updater              initial value       │ │
│  │   //   state     function                                 │ │
│  │                                                           │ │
│  │   return <div>Balance: ${balance}</div>;                  │ │
│  │ }                                                         │ │
│  │                                                           │ │
│  │ React internal state: { balance: 5000 }                   │ │
│  │ DOM renders: <div>Balance: $5000</div>                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  STEP 2: User Action (setState called)                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ <button onClick={() => setBalance(6000)}>                 │ │
│  │   Add $1000                                               │ │
│  │ </button>                                                 │ │
│  │                                                           │ │
│  │ User clicks button                                        │ │
│  │ → setBalance(6000) called                                 │ │
│  │ → React schedules state update                            │ │
│  │ → React marks component as "needs update"                 │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  STEP 3: React Processes Update (Reconciliation)               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ React internal state: { balance: 5000 } → { balance: 6000 }│ │
│  │                                                           │ │
│  │ React compares:                                           │ │
│  │   Old state: 5000                                         │ │
│  │   New state: 6000                                         │ │
│  │   States are different? ✅ YES                            │ │
│  │   → Component needs re-render                             │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  STEP 4: Component Re-renders                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ function AccountBalance() {                               │ │
│  │   const [balance, setBalance] = useState(5000);           │ │
│  │   //                                      ↑               │ │
│  │   // useState returns NEW value: 6000 (not 5000!)         │ │
│  │   // balance = 6000                                       │ │
│  │                                                           │ │
│  │   return <div>Balance: ${balance}</div>;                  │ │
│  │   //              ↓                                       │ │
│  │   // Renders: <div>Balance: $6000</div>                   │ │
│  │ }                                                         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  STEP 5: Virtual DOM Diff & Commit to Real DOM                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Old Virtual DOM:                                          │ │
│  │   <div>Balance: $5000</div>                               │ │
│  │                                                           │ │
│  │ New Virtual DOM:                                          │ │
│  │   <div>Balance: $6000</div>                               │ │
│  │                                                           │ │
│  │ Diff: Text content changed                                │ │
│  │ Patch: Update text from "$5000" to "$6000"                │ │
│  │                                                           │ │
│  │ Real DOM updated: <div>Balance: $6000</div>               │ │
│  │ User sees: Balance: $6000 ✅                              │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Total time: ~5-15ms (React is fast!)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 💻 useState Basics - From Simple to Advanced

#### Example 1: Basic useState Usage

```jsx
/**
 * Simple counter example to understand useState fundamentals
 */

import React, { useState } from 'react';

function Counter() {
  // Declare state variable
  const [count, setCount] = useState(0);
  //     ↑       ↑                    ↑
  //  current  updater           initial value
  //   value   function

  console.log('Component rendered, count:', count);

  const increment = () => {
    setCount(count + 1);  // Schedule update with new value
    console.log('After setCount, count is still:', count);  // Still old value!
    // State updates are asynchronous!
  };

  const decrement = () => {
    setCount(count - 1);
  };

  const reset = () => {
    setCount(0);
  };

  return (
    <div className="counter">
      <h2>Count: {count}</h2>
      <div className="button-group">
        <button onClick={decrement}>-</button>
        <button onClick={reset}>Reset</button>
        <button onClick={increment}>+</button>
      </div>
    </div>
  );
}

export default Counter;
```

**Console Output**:
```
Initial render:
  Component rendered, count: 0

User clicks "+" button:
  After setCount, count is still: 0  ← Still old value!
  Component rendered, count: 1       ← Re-render with new value

User clicks "+" again:
  After setCount, count is still: 1
  Component rendered, count: 2
```

**Key Observations**:
1. **State is preserved** between re-renders (doesn't reset to 0)
2. **setState is asynchronous** (new value not available immediately)
3. **Each setState triggers re-render**
4. **Regular variables would reset** on every render

---

#### Example 2: State vs Regular Variables

```jsx
/**
 * Comparison: State vs Regular Variables
 */

function StateVsVariable() {
  // Regular variable (resets on every render)
  let regularCount = 0;

  // State variable (persists between renders)
  const [stateCount, setStateCount] = useState(0);

  const incrementRegular = () => {
    regularCount = regularCount + 1;  // Changes variable
    console.log('Regular count:', regularCount);  // Logs new value
    // But component doesn't re-render!
  };

  const incrementState = () => {
    setStateCount(stateCount + 1);  // Triggers re-render
  };

  return (
    <div className="comparison">
      <h2>Regular Variable vs State</h2>
      
      <div className="variable-demo">
        <h3>Regular Variable</h3>
        <p>Value: {regularCount}</p>  {/* Always shows 0! */}
        <button onClick={incrementRegular}>
          Increment Regular (doesn't work)
        </button>
        <p className="note">
          ⚠️ Increments in memory but UI doesn't update
        </p>
      </div>

      <div className="state-demo">
        <h3>State Variable</h3>
        <p>Value: {stateCount}</p>  {/* Updates on screen! */}
        <button onClick={incrementState}>
          Increment State (works!)
        </button>
        <p className="note">
          ✅ Updates UI on every click
        </p>
      </div>
    </div>
  );
}

// Expected behavior:
// - Regular count: Increments in console but UI stuck at 0
// - State count: Updates UI on every click
```

**Why State is Different**:

```javascript
// What happens with regular variable:
function Component() {
  let count = 0;  // ← Reset to 0 on every render!
  
  const increment = () => {
    count = count + 1;  // Variable changes
    // But no re-render triggered, so change not visible
  };
  
  return <div>{count}</div>;  // Always shows 0
}

// Next render:
//   count = 0 (reset!)

// What happens with state:
function Component() {
  const [count, setCount] = useState(0);  // ← Stored by React!
  
  const increment = () => {
    setCount(count + 1);  // Tells React to re-render
  };
  
  return <div>{count}</div>;  // Shows current state value
}

// Next render:
//   count = 1 (preserved by React!)
```

---

#### Example 3: Multiple State Variables

```jsx
/**
 * Managing multiple independent state variables
 */

function BankingForm() {
  // Multiple state variables (one per field)
  const [accountName, setAccountName] = useState('');
  const [accountType, setAccountType] = useState('SAVINGS');
  const [initialDeposit, setInitialDeposit] = useState('');
  const [agreeTerms, setAgreeTerms] = useState(false);
  const [errors, setErrors] = useState({});

  const handleSubmit = (e) => {
    e.preventDefault();

    // Validation
    const newErrors = {};
    
    if (!accountName.trim()) {
      newErrors.accountName = 'Account name is required';
    }
    
    const deposit = parseFloat(initialDeposit);
    if (isNaN(deposit) || deposit < 100) {
      newErrors.initialDeposit = 'Minimum deposit is $100';
    }
    
    if (!agreeTerms) {
      newErrors.agreeTerms = 'You must agree to terms';
    }

    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }

    // Submit form
    console.log('Form submitted:', {
      accountName,
      accountType,
      initialDeposit: deposit,
      agreeTerms
    });

    // Reset form
    setAccountName('');
    setAccountType('SAVINGS');
    setInitialDeposit('');
    setAgreeTerms(false);
    setErrors({});
  };

  return (
    <form onSubmit={handleSubmit} className="banking-form">
      <h2>Open New Account</h2>

      {/* Account Name */}
      <div className="form-group">
        <label htmlFor="accountName">Account Name:</label>
        <input
          type="text"
          id="accountName"
          value={accountName}
          onChange={(e) => setAccountName(e.target.value)}
          placeholder="Enter account name"
        />
        {errors.accountName && (
          <span className="error">{errors.accountName}</span>
        )}
      </div>

      {/* Account Type */}
      <div className="form-group">
        <label htmlFor="accountType">Account Type:</label>
        <select
          id="accountType"
          value={accountType}
          onChange={(e) => setAccountType(e.target.value)}
        >
          <option value="SAVINGS">Savings Account</option>
          <option value="CHECKING">Checking Account</option>
          <option value="BUSINESS">Business Account</option>
        </select>
      </div>

      {/* Initial Deposit */}
      <div className="form-group">
        <label htmlFor="initialDeposit">Initial Deposit:</label>
        <input
          type="number"
          id="initialDeposit"
          value={initialDeposit}
          onChange={(e) => setInitialDeposit(e.target.value)}
          placeholder="Minimum $100"
          step="0.01"
        />
        {errors.initialDeposit && (
          <span className="error">{errors.initialDeposit}</span>
        )}
      </div>

      {/* Terms Checkbox */}
      <div className="form-group checkbox">
        <label>
          <input
            type="checkbox"
            checked={agreeTerms}
            onChange={(e) => setAgreeTerms(e.target.checked)}
          />
          I agree to the terms and conditions
        </label>
        {errors.agreeTerms && (
          <span className="error">{errors.agreeTerms}</span>
        )}
      </div>

      <button type="submit" className="btn-primary">
        Open Account
      </button>
    </form>
  );
}

export default BankingForm;
```

**When to Use Multiple State Variables**:
- ✅ **Independent values** (name, type, deposit are separate concerns)
- ✅ **Different update patterns** (name updates on every keystroke, type on selection)
- ✅ **Clearer code** (easy to see what each piece of state represents)

---

#### Example 4: Single Object State

```jsx
/**
 * Managing related data as a single object
 */

function AccountProfile() {
  // Single state object for related data
  const [account, setAccount] = useState({
    id: 'ACC001',
    name: 'John Doe',
    email: 'john@example.com',
    phone: '555-1234',
    type: 'SAVINGS',
    balance: 5000,
    isActive: true
  });

  // Update single field (must spread existing state!)
  const updateName = (newName) => {
    setAccount({
      ...account,           // ← Spread existing properties
      name: newName         // ← Override name
    });
  };

  const updateEmail = (newEmail) => {
    setAccount(prevAccount => ({
      ...prevAccount,       // ← Use previous state
      email: newEmail
    }));
  };

  const updateBalance = (amount) => {
    setAccount(prev => ({
      ...prev,
      balance: prev.balance + amount  // ← Calculate from previous
    }));
  };

  const toggleActive = () => {
    setAccount(prev => ({
      ...prev,
      isActive: !prev.isActive
    }));
  };

  return (
    <div className="account-profile">
      <h2>Account Profile</h2>

      <div className="profile-info">
        <div className="info-row">
          <label>Account ID:</label>
          <span>{account.id}</span>
        </div>

        <div className="info-row">
          <label>Name:</label>
          <input
            value={account.name}
            onChange={(e) => updateName(e.target.value)}
          />
        </div>

        <div className="info-row">
          <label>Email:</label>
          <input
            value={account.email}
            onChange={(e) => updateEmail(e.target.value)}
          />
        </div>

        <div className="info-row">
          <label>Phone:</label>
          <input
            value={account.phone}
            onChange={(e) => setAccount({ ...account, phone: e.target.value })}
          />
        </div>

        <div className="info-row">
          <label>Type:</label>
          <span>{account.type}</span>
        </div>

        <div className="info-row">
          <label>Balance:</label>
          <span className={account.balance >= 0 ? 'positive' : 'negative'}>
            ${account.balance.toFixed(2)}
          </span>
        </div>

        <div className="info-row">
          <label>Status:</label>
          <span className={account.isActive ? 'active' : 'inactive'}>
            {account.isActive ? 'Active' : 'Inactive'}
          </span>
          <button onClick={toggleActive}>
            {account.isActive ? 'Deactivate' : 'Activate'}
          </button>
        </div>
      </div>

      <div className="actions">
        <button onClick={() => updateBalance(100)}>
          Deposit $100
        </button>
        <button onClick={() => updateBalance(-50)}>
          Withdraw $50
        </button>
      </div>
    </div>
  );
}

export default AccountProfile;
```

**When to Use Object State**:
- ✅ **Related data** (all account properties belong together)
- ✅ **API responses** (server returns object, store it as-is)
- ✅ **Fewer state updates** (update multiple fields at once)

**⚠️ Important**: Always spread existing state when updating!

```jsx
// ❌ BAD: Loses other properties
setAccount({ name: 'Jane' });  
// Result: { name: 'Jane' } (email, phone, etc. GONE!)

// ✅ GOOD: Keeps other properties
setAccount({ ...account, name: 'Jane' });
// Result: { name: 'Jane', email: 'john@example.com', phone: '555-1234', ... }
```

---

#### Example 5: Functional Updates (Critical for Correctness)

```jsx
/**
 * Why functional updates matter
 */

function TransactionCounter() {
  const [balance, setBalance] = useState(1000);

  // ❌ PROBLEM: Multiple rapid updates
  const addMultipleTransactionsBad = () => {
    setBalance(balance + 100);  // balance = 1000 + 100 = 1100
    setBalance(balance + 100);  // balance = 1000 + 100 = 1100 (still!)
    setBalance(balance + 100);  // balance = 1000 + 100 = 1100 (still!)
    // Expected: 1300, Actual: 1100 ❌
  };

  // ✅ SOLUTION: Functional updates
  const addMultipleTransactionsGood = () => {
    setBalance(prev => prev + 100);  // 1000 + 100 = 1100
    setBalance(prev => prev + 100);  // 1100 + 100 = 1200
    setBalance(prev => prev + 100);  // 1200 + 100 = 1300
    // Result: 1300 ✅
  };

  return (
    <div className="transaction-counter">
      <h2>Balance: ${balance}</h2>

      <div className="demo">
        <button onClick={addMultipleTransactionsBad}>
          Bad: Add 3 × $100 (will only add once!)
        </button>
        <button onClick={addMultipleTransactionsGood}>
          Good: Add 3 × $100 (works correctly!)
        </button>
        <button onClick={() => setBalance(1000)}>
          Reset
        </button>
      </div>

      <div className="explanation">
        <h3>Why This Happens:</h3>
        <pre>
{`// Direct value (bad):
const addMultiple = () => {
  setBalance(balance + 100);  // balance captured as 1000
  setBalance(balance + 100);  // balance still 1000!
  setBalance(balance + 100);  // balance still 1000!
};

// Functional update (good):
const addMultiple = () => {
  setBalance(prev => prev + 100);  // prev = 1000 → 1100
  setBalance(prev => prev + 100);  // prev = 1100 → 1200
  setBalance(prev => prev + 100);  // prev = 1200 → 1300
};`}
        </pre>
      </div>
    </div>
  );
}

export default TransactionCounter;
```

**When to Use Functional Updates**:

```jsx
// ✅ Use functional update when:
// 1. New state depends on previous state
setBalance(prev => prev + 100);

// 2. Multiple updates in quick succession
setBalance(prev => prev + 100);
setBalance(prev => prev + 50);

// 3. Updates in async callbacks
setTimeout(() => {
  setBalance(prev => prev + 100);  // Uses latest value
}, 1000);

// ❌ Direct value is OK when:
// 1. New value doesn't depend on old value
setBalance(0);  // Reset to 0

// 2. Setting from user input
setName(e.target.value);  // User typed something

// 3. Setting from API response
setUsers(response.data);  // Got data from server
```

---

### 🎯 Banking Example: Complete Account Manager

```typescript
/**
 * Production-ready banking account manager
 * Demonstrates all useState patterns
 */

import React, { useState, useCallback } from 'react';

// ============================================
// TYPE DEFINITIONS
// ============================================

type TransactionType = 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER' | 'FEE';

interface Transaction {
  id: string;
  type: TransactionType;
  amount: number;
  date: Date;
  description: string;
  balanceAfter: number;
}

interface AccountState {
  accountId: string;
  accountName: string;
  balance: number;
  currency: string;
  isLocked: boolean;
}

// ============================================
// MAIN COMPONENT
// ============================================

function BankingAccountManager() {
  // ============================================
  // STATE: Account Information
  // ============================================
  const [account, setAccount] = useState<AccountState>({
    accountId: 'ACC001',
    accountName: 'Primary Checking',
    balance: 5000,
    currency: 'USD',
    isLocked: false
  });

  // ============================================
  // STATE: Transaction List
  // ============================================
  const [transactions, setTransactions] = useState<Transaction[]>([
    {
      id: 'TXN001',
      type: 'DEPOSIT',
      amount: 5000,
      date: new Date('2024-11-01'),
      description: 'Initial deposit',
      balanceAfter: 5000
    }
  ]);

  // ============================================
  // STATE: UI State
  // ============================================
  const [amount, setAmount] = useState<string>('');
  const [description, setDescription] = useState<string>('');
  const [selectedType, setSelectedType] = useState<TransactionType>('DEPOSIT');
  const [error, setError] = useState<string>('');
  const [successMessage, setSuccessMessage] = useState<string>('');

  // ============================================
  // STATE: Statistics (Derived State)
  // ============================================
  const totalDeposits = transactions
    .filter(t => t.type === 'DEPOSIT')
    .reduce((sum, t) => sum + t.amount, 0);

  const totalWithdrawals = transactions
    .filter(t => t.type === 'WITHDRAWAL' || t.type === 'FEE')
    .reduce((sum, t) => sum + t.amount, 0);

  const transactionCount = transactions.length;

  // ============================================
  // HANDLERS: Transaction Operations
  // ============================================

  const handleDeposit = useCallback(() => {
    const depositAmount = parseFloat(amount);

    // Validation
    if (isNaN(depositAmount) || depositAmount <= 0) {
      setError('Please enter a valid amount');
      return;
    }

    if (depositAmount > 50000) {
      setError('Maximum deposit amount is $50,000');
      return;
    }

    // Clear errors
    setError('');

    // Update balance (functional update!)
    setAccount(prevAccount => ({
      ...prevAccount,
      balance: prevAccount.balance + depositAmount
    }));

    // Add transaction
    const newTransaction: Transaction = {
      id: `TXN${Date.now()}`,
      type: 'DEPOSIT',
      amount: depositAmount,
      date: new Date(),
      description: description || `Deposit of $${depositAmount.toFixed(2)}`,
      balanceAfter: account.balance + depositAmount
    };

    setTransactions(prevTransactions => [newTransaction, ...prevTransactions]);

    // Success message
    setSuccessMessage(`Successfully deposited $${depositAmount.toFixed(2)}`);
    setTimeout(() => setSuccessMessage(''), 3000);

    // Reset form
    setAmount('');
    setDescription('');
  }, [amount, description, account.balance]);

  const handleWithdrawal = useCallback(() => {
    const withdrawalAmount = parseFloat(amount);

    // Validation
    if (isNaN(withdrawalAmount) || withdrawalAmount <= 0) {
      setError('Please enter a valid amount');
      return;
    }

    if (withdrawalAmount > account.balance) {
      setError('Insufficient funds');
      return;
    }

    if (account.isLocked) {
      setError('Account is locked. Cannot withdraw.');
      return;
    }

    // Clear errors
    setError('');

    // Update balance
    setAccount(prevAccount => ({
      ...prevAccount,
      balance: prevAccount.balance - withdrawalAmount
    }));

    // Add transaction
    const newTransaction: Transaction = {
      id: `TXN${Date.now()}`,
      type: 'WITHDRAWAL',
      amount: withdrawalAmount,
      date: new Date(),
      description: description || `Withdrawal of $${withdrawalAmount.toFixed(2)}`,
      balanceAfter: account.balance - withdrawalAmount
    };

    setTransactions(prevTransactions => [newTransaction, ...prevTransactions]);

    // Success message
    setSuccessMessage(`Successfully withdrew $${withdrawalAmount.toFixed(2)}`);
    setTimeout(() => setSuccessMessage(''), 3000);

    // Reset form
    setAmount('');
    setDescription('');
  }, [amount, description, account.balance, account.isLocked]);

  const handleTransaction = () => {
    if (selectedType === 'DEPOSIT') {
      handleDeposit();
    } else if (selectedType === 'WITHDRAWAL') {
      handleWithdrawal();
    }
  };

  const toggleLock = () => {
    setAccount(prev => ({
      ...prev,
      isLocked: !prev.isLocked
    }));
  };

  // ============================================
  // RENDER
  // ============================================

  return (
    <div className="banking-account-manager">
      {/* Account Header */}
      <div className="account-header">
        <h1>{account.accountName}</h1>
        <p className="account-id">Account: {account.accountId}</p>
        <div className={`balance ${account.balance < 0 ? 'negative' : 'positive'}`}>
          <span className="currency">{account.currency}</span>
          <span className="amount">${Math.abs(account.balance).toFixed(2)}</span>
          {account.balance < 0 && <span className="overdraft">(Overdraft)</span>}
        </div>
        <button 
          onClick={toggleLock}
          className={`lock-button ${account.isLocked ? 'locked' : 'unlocked'}`}
        >
          {account.isLocked ? '🔒 Locked' : '🔓 Unlocked'}
        </button>
      </div>

      {/* Statistics */}
      <div className="statistics-grid">
        <div className="stat-card">
          <h3>Total Deposits</h3>
          <p className="stat-value positive">${totalDeposits.toFixed(2)}</p>
        </div>
        <div className="stat-card">
          <h3>Total Withdrawals</h3>
          <p className="stat-value negative">${totalWithdrawals.toFixed(2)}</p>
        </div>
        <div className="stat-card">
          <h3>Transactions</h3>
          <p className="stat-value">{transactionCount}</p>
        </div>
      </div>

      {/* Transaction Form */}
      <div className="transaction-form">
        <h2>New Transaction</h2>

        {error && (
          <div className="alert alert-error">
            ⚠️ {error}
          </div>
        )}

        {successMessage && (
          <div className="alert alert-success">
            ✅ {successMessage}
          </div>
        )}

        <div className="form-row">
          <div className="form-group">
            <label>Transaction Type:</label>
            <select
              value={selectedType}
              onChange={(e) => setSelectedType(e.target.value as TransactionType)}
              disabled={account.isLocked}
            >
              <option value="DEPOSIT">Deposit</option>
              <option value="WITHDRAWAL">Withdrawal</option>
            </select>
          </div>

          <div className="form-group">
            <label>Amount:</label>
            <input
              type="number"
              value={amount}
              onChange={(e) => setAmount(e.target.value)}
              placeholder="Enter amount"
              step="0.01"
              min="0"
              disabled={account.isLocked}
            />
          </div>
        </div>

        <div className="form-group">
          <label>Description (optional):</label>
          <input
            type="text"
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            placeholder="e.g., Salary, Rent, etc."
            disabled={account.isLocked}
          />
        </div>

        <button
          onClick={handleTransaction}
          className="btn-primary"
          disabled={account.isLocked || !amount}
        >
          {selectedType === 'DEPOSIT' ? 'Deposit' : 'Withdraw'} ${amount || '0.00'}
        </button>
      </div>

      {/* Transaction History */}
      <div className="transaction-history">
        <h2>Transaction History</h2>
        {transactions.length === 0 ? (
          <p className="empty-state">No transactions yet</p>
        ) : (
          <ul className="transaction-list">
            {transactions.map(txn => (
              <li key={txn.id} className={`transaction-item ${txn.type.toLowerCase()}`}>
                <div className="transaction-icon">
                  {txn.type === 'DEPOSIT' ? '💰' :
                   txn.type === 'WITHDRAWAL' ? '💸' :
                   txn.type === 'FEE' ? '📝' : '🔄'}
                </div>
                <div className="transaction-details">
                  <strong>{txn.description}</strong>
                  <span className="transaction-date">
                    {txn.date.toLocaleDateString()} {txn.date.toLocaleTimeString()}
                  </span>
                </div>
                <div className="transaction-amount">
                  <span className={txn.type === 'DEPOSIT' ? 'positive' : 'negative'}>
                    {txn.type === 'DEPOSIT' ? '+' : '-'}${txn.amount.toFixed(2)}
                  </span>
                  <span className="balance-after">
                    Balance: ${txn.balanceAfter.toFixed(2)}
                  </span>
                </div>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}

export default BankingAccountManager;
```

**State Management Patterns Demonstrated**:

1. ✅ **Object State** - `account` object with multiple related properties
2. ✅ **Array State** - `transactions` list
3. ✅ **Multiple State Variables** - `amount`, `description`, `error`, etc.
4. ✅ **Functional Updates** - `setAccount(prev => ({ ...prev, balance: ... }))`
5. ✅ **Derived State** - `totalDeposits`, `totalWithdrawals` calculated from transactions
6. ✅ **Controlled Inputs** - Form inputs controlled by state
7. ✅ **Conditional Rendering** - Error/success messages, empty states
8. ✅ **State Reset** - Clearing form after submission

---

### 🚀 Advanced useState Patterns

#### Pattern 1: Lazy Initialization

```jsx
/**
 * Lazy initialization for expensive initial state
 */

// ❌ BAD: Function runs on every render
function BadComponent() {
  const [data, setData] = useState(expensiveComputation());
  //                                 ↑
  //                          Runs every render! ❌
  
  return <div>{data}</div>;
}

// ✅ GOOD: Function runs only once (initial render)
function GoodComponent() {
  const [data, setData] = useState(() => expensiveComputation());
  //                                ↑
  //                         Function called only once ✅
  
  return <div>{data}</div>;
}

// Real-world example: Loading from localStorage
function BankingDashboard() {
  // ❌ BAD: Reads localStorage on every render
  const [accounts, setAccounts] = useState(
    JSON.parse(localStorage.getItem('accounts') || '[]')
  );

  // ✅ GOOD: Reads localStorage only once
  const [accounts, setAccounts] = useState(() => {
    console.log('Reading from localStorage (only on mount)');
    const saved = localStorage.getItem('accounts');
    return saved ? JSON.parse(saved) : [];
  });

  return (
    <div>
      <h2>Your Accounts ({accounts.length})</h2>
      {accounts.map(account => (
        <div key={account.id}>{account.name}</div>
      ))}
    </div>
  );
}

function expensiveComputation() {
  console.log('Expensive computation running...');
  let result = 0;
  for (let i = 0; i < 1000000; i++) {
    result += Math.sqrt(i);
  }
  return result;
}
```

**When to Use Lazy Initialization**:
- ✅ Reading from localStorage/sessionStorage
- ✅ Complex calculations for initial state
- ✅ Parsing large JSON data
- ✅ Computing default values from props

---

#### Pattern 2: State Batching (Automatic in React 18+)

```jsx
/**
 * React automatically batches state updates
 */

function StateBatchingDemo() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  const [text, setText] = useState('');

  console.log('Component rendered');

  // React 18+: All updates are batched
  const handleClick = () => {
    console.log('Button clicked');
    
    setCount(c => c + 1);      // Update 1
    setFlag(f => !f);          // Update 2
    setText('Updated');        // Update 3
    
    // In React 18+: Only ONE re-render for all 3 updates! ✅
    // In React 17: Would cause 3 re-renders ❌
  };

  // Even in async code (React 18+)
  const handleAsyncClick = () => {
    setTimeout(() => {
      setCount(c => c + 1);    // Batched
      setFlag(f => !f);        // Batched
      setText('Async Update'); // Batched
      // Still only ONE re-render! ✅
    }, 1000);
  };

  return (
    <div>
      <h2>State Batching Demo</h2>
      <p>Count: {count}</p>
      <p>Flag: {flag ? 'true' : 'false'}</p>
      <p>Text: {text}</p>
      <button onClick={handleClick}>Update All (Sync)</button>
      <button onClick={handleAsyncClick}>Update All (Async)</button>
      <p className="note">
        ℹ️ Check console: Only 1 render per button click!
      </p>
    </div>
  );
}

// Expected console output:
// Component rendered (initial)
// User clicks button:
//   Button clicked
//   Component rendered (all 3 updates batched!)
```

---

#### Pattern 3: State Reducer Pattern (useState + switch)

```jsx
/**
 * Complex state logic with useState (before useReducer)
 */

function TransactionManager() {
  const [state, setState] = useState({
    transactions: [],
    balance: 0,
    filter: 'ALL',
    sortBy: 'date',
    isLoading: false,
    error: null
  });

  // Actions as functions
  const addTransaction = (transaction) => {
    setState(prev => ({
      ...prev,
      transactions: [transaction, ...prev.transactions],
      balance: transaction.type === 'DEPOSIT' 
        ? prev.balance + transaction.amount 
        : prev.balance - transaction.amount
    }));
  };

  const setFilter = (filter) => {
    setState(prev => ({ ...prev, filter }));
  };

  const setSortBy = (sortBy) => {
    setState(prev => ({ ...prev, sortBy }));
  };

  const setLoading = (isLoading) => {
    setState(prev => ({ ...prev, isLoading }));
  };

  const setError = (error) => {
    setState(prev => ({ ...prev, error, isLoading: false }));
  };

  // Filtered and sorted transactions
  const processedTransactions = state.transactions
    .filter(t => state.filter === 'ALL' || t.type === state.filter)
    .sort((a, b) => {
      if (state.sortBy === 'date') {
        return b.date.getTime() - a.date.getTime();
      } else if (state.sortBy === 'amount') {
        return b.amount - a.amount;
      }
      return 0;
    });

  return (
    <div>
      <h2>Balance: ${state.balance.toFixed(2)}</h2>
      
      {state.isLoading && <p>Loading...</p>}
      {state.error && <p className="error">{state.error}</p>}
      
      <div className="controls">
        <select value={state.filter} onChange={e => setFilter(e.target.value)}>
          <option value="ALL">All</option>
          <option value="DEPOSIT">Deposits</option>
          <option value="WITHDRAWAL">Withdrawals</option>
        </select>
        
        <select value={state.sortBy} onChange={e => setSortBy(e.target.value)}>
          <option value="date">Date</option>
          <option value="amount">Amount</option>
        </select>
      </div>

      <ul>
        {processedTransactions.map(txn => (
          <li key={txn.id}>
            {txn.type}: ${txn.amount}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

#### Pattern 4: Previous State Pattern

```jsx
/**
 * Keeping track of previous state value
 */

function AccountBalanceTracker() {
  const [balance, setBalance] = useState(1000);
  const [previousBalance, setPreviousBalance] = useState(1000);

  const updateBalance = (newBalance: number) => {
    setPreviousBalance(balance);  // Save current as previous
    setBalance(newBalance);       // Set new balance
  };

  const change = balance - previousBalance;
  const percentChange = ((change / previousBalance) * 100).toFixed(2);

  return (
    <div className="balance-tracker">
      <h2>Current Balance</h2>
      <p className="balance">${balance.toFixed(2)}</p>

      <div className="change-indicator">
        <p>Previous: ${previousBalance.toFixed(2)}</p>
        <p className={change >= 0 ? 'positive' : 'negative'}>
          Change: {change >= 0 ? '+' : ''}${change.toFixed(2)}
          ({change >= 0 ? '+' : ''}{percentChange}%)
        </p>
      </div>

      <div className="actions">
        <button onClick={() => updateBalance(balance + 500)}>
          Deposit $500
        </button>
        <button onClick={() => updateBalance(balance - 200)}>
          Withdraw $200
        </button>
        <button onClick={() => updateBalance(previousBalance)}>
          Undo Last Change
        </button>
      </div>
    </div>
  );
}
```

---

#### Pattern 5: State Synchronization with Props

```jsx
/**
 * Syncing state with props (derived state pattern)
 */

interface AccountCardProps {
  initialBalance: number;
  accountId: string;
}

function AccountCard({ initialBalance, accountId }: AccountCardProps) {
  // Initialize state from props
  const [balance, setBalance] = useState(initialBalance);
  const [lastSyncedBalance, setLastSyncedBalance] = useState(initialBalance);

  // Sync state when prop changes
  if (initialBalance !== lastSyncedBalance) {
    setBalance(initialBalance);
    setLastSyncedBalance(initialBalance);
  }

  // OR use useEffect (better for side effects)
  // useEffect(() => {
  //   setBalance(initialBalance);
  // }, [initialBalance]);

  return (
    <div className="account-card">
      <h3>Account {accountId}</h3>
      <p>Server Balance: ${initialBalance.toFixed(2)}</p>
      <p>Local Balance: ${balance.toFixed(2)}</p>
      
      {balance !== initialBalance && (
        <p className="warning">
          ⚠️ Local changes not saved
        </p>
      )}

      <button onClick={() => setBalance(balance + 100)}>
        Add $100 (Local)
      </button>
      <button onClick={() => setBalance(initialBalance)}>
        Reset to Server Value
      </button>
    </div>
  );
}
```

---

### 🔍 useState Internals (How It Works)

```javascript
/**
 * Simplified implementation of useState
 * (Not actual React code, but shows the concept)
 */

// React's internal state storage
let componentState = [];  // Array of state values
let currentStateIndex = 0;  // Which state we're on

function useState(initialValue) {
  // Get current state slot
  const stateIndex = currentStateIndex;
  
  // Initialize state if first render
  if (componentState[stateIndex] === undefined) {
    componentState[stateIndex] = 
      typeof initialValue === 'function' 
        ? initialValue()  // Lazy initialization
        : initialValue;
  }
  
  // Get current state value
  const state = componentState[stateIndex];
  
  // Create setState function
  const setState = (newValue) => {
    // Calculate new state value
    const nextState = 
      typeof newValue === 'function'
        ? newValue(componentState[stateIndex])  // Functional update
        : newValue;
    
    // Only update if value changed
    if (nextState !== componentState[stateIndex]) {
      componentState[stateIndex] = nextState;
      
      // Schedule re-render
      scheduleRerender();
    }
  };
  
  // Move to next state slot
  currentStateIndex++;
  
  // Return state and updater
  return [state, setState];
}

function Component() {
  const [count, setCount] = useState(0);        // Index 0
  const [name, setName] = useState('John');     // Index 1
  const [flag, setFlag] = useState(false);      // Index 2
  
  // After first render:
  // componentState = [0, 'John', false]
  // currentStateIndex = 3
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Before render:
currentStateIndex = 0;

// Render component:
Component();  // Calls useState 3 times, index moves 0→1→2→3

// After render:
currentStateIndex = 0;  // Reset for next render
```

**Why Hooks Must Be Called in Same Order**:

```jsx
// ✅ CORRECT: Same order every render
function Component({ showName }) {
  const [count, setCount] = useState(0);     // Always index 0
  const [name, setName] = useState('John');  // Always index 1
  const [flag, setFlag] = useState(false);   // Always index 2
  
  return <div>...</div>;
}

// ❌ WRONG: Conditional hook
function Component({ showName }) {
  const [count, setCount] = useState(0);  // Index 0
  
  if (showName) {  // ❌ Conditional!
    const [name, setName] = useState('John');  // Sometimes index 1, sometimes skipped
  }
  
  const [flag, setFlag] = useState(false);  // Index 1 or 2 (inconsistent!)
  
  return <div>...</div>;
}

// First render (showName = true):
//   componentState = [0, 'John', false]
//                     ↑    ↑       ↑
//                   count  name   flag
//
// Second render (showName = false):
//   componentState = [0, false, ???]
//                     ↑    ↑
//                   count  flag (WRONG POSITION!)
//
// React gets confused! 💥
```

---

### 📊 State Update Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                   STATE UPDATE LIFECYCLE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User Action                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ <button onClick={() => setCount(count + 1)}>+</button>    │ │
│  │                                                           │ │
│  │ User clicks button                                        │ │
│  │ → setCount(5) called                                      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  2. React Schedules Update                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ React's internal update queue:                            │ │
│  │   [{                                                      │ │
│  │     component: Component,                                 │ │
│  │     stateIndex: 0,                                        │ │
│  │     newValue: 5,                                          │ │
│  │     oldValue: 4                                           │ │
│  │   }]                                                      │ │
│  │                                                           │ │
│  │ Update is ASYNCHRONOUS (not immediate)                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  3. React Batches Updates (if multiple)                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ If multiple setStates called in same event:               │ │
│  │   setCount(5)                                             │ │
│  │   setName('Jane')                                         │ │
│  │   setFlag(true)                                           │ │
│  │                                                           │ │
│  │ React batches all updates → ONE re-render                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  4. React Compares States                                       │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Old state: 4                                              │ │
│  │ New state: 5                                              │ │
│  │                                                           │ │
│  │ Are they different? YES                                   │ │
│  │ → Proceed with re-render                                  │ │
│  │                                                           │ │
│  │ (If same, React skips re-render - optimization)          │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  5. Component Re-renders                                        │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ function Component() {                                    │ │
│  │   const [count, setCount] = useState(0);                  │ │
│  │   //                                 ↑                    │ │
│  │   // Initial value ignored (not first render)            │ │
│  │   // Returns stored value: 5                             │ │
│  │                                                           │ │
│  │   console.log('Rendering with count:', count); // 5       │ │
│  │                                                           │ │
│  │   return <div>Count: {count}</div>;                       │ │
│  │ }                                                         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                    │
│  6. Virtual DOM Diff & Commit                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Old: <div>Count: 4</div>                                  │ │
│  │ New: <div>Count: 5</div>                                  │ │
│  │                                                           │ │
│  │ Diff: Text changed from "4" to "5"                        │ │
│  │ Commit: Update DOM text node                             │ │
│  │                                                           │ │
│  │ Browser displays: Count: 5 ✅                             │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### ✅ DO's - Best Practices

#### 1. ✅ DO: Use Functional Updates When New State Depends on Old

```jsx
// ✅ GOOD: Functional update
setBalance(prevBalance => prevBalance + 100);

// ❌ BAD: Direct value (can cause bugs)
setBalance(balance + 100);

// Why? Multiple updates in quick succession:
const handleMultiple = () => {
  // ❌ BAD: All use stale closure value
  setBalance(balance + 100);  // balance = 1000
  setBalance(balance + 100);  // balance = 1000 (still!)
  setBalance(balance + 100);  // balance = 1000 (still!)
  // Result: 1100 (expected 1300) ❌

  // ✅ GOOD: Each uses latest value
  setBalance(prev => prev + 100);  // prev = 1000 → 1100
  setBalance(prev => prev + 100);  // prev = 1100 → 1200
  setBalance(prev => prev + 100);  // prev = 1200 → 1300
  // Result: 1300 ✅
};
```

#### 2. ✅ DO: Initialize Complex State with Function

```jsx
// ✅ GOOD: Function called only once
const [data, setData] = useState(() => {
  const saved = localStorage.getItem('data');
  return saved ? JSON.parse(saved) : defaultData;
});

// ❌ BAD: Expression evaluated every render
const [data, setData] = useState(
  JSON.parse(localStorage.getItem('data') || '{}')
);
```

#### 3. ✅ DO: Keep State Minimal and Derived

```jsx
// ✅ GOOD: Store only source data, derive the rest
function TransactionList({ transactions }) {
  const [filter, setFilter] = useState('ALL');
  
  // Derive filtered list (don't store in state!)
  const filtered = transactions.filter(t => 
    filter === 'ALL' || t.type === filter
  );
  
  return <ul>{filtered.map(...)}</ul>;
}

// ❌ BAD: Storing derived data in state
function TransactionList({ transactions }) {
  const [filter, setFilter] = useState('ALL');
  const [filtered, setFiltered] = useState(transactions);  // ❌ Redundant!
  
  // Now you have to keep them in sync!
  useEffect(() => {
    setFiltered(transactions.filter(t => 
      filter === 'ALL' || t.type === filter
    ));
  }, [transactions, filter]);
  
  return <ul>{filtered.map(...)}</ul>;
}
```

#### 4. ✅ DO: Use Objects for Related State

```jsx
// ✅ GOOD: Related data in one object
const [account, setAccount] = useState({
  id: 'ACC001',
  name: 'Checking',
  balance: 5000,
  type: 'CHECKING'
});

// Update one field
setAccount(prev => ({ ...prev, balance: 6000 }));

// ❌ AVOID: Many separate state variables for related data
const [accountId, setAccountId] = useState('ACC001');
const [accountName, setAccountName] = useState('Checking');
const [accountBalance, setAccountBalance] = useState(5000);
const [accountType, setAccountType] = useState('CHECKING');
```

#### 5. ✅ DO: Reset State When Prop Changes (if needed)

```jsx
function AccountDetails({ accountId }) {
  const [details, setDetails] = useState(null);
  const [loading, setLoading] = useState(false);

  // Reset state when accountId changes
  useEffect(() => {
    setDetails(null);
    setLoading(true);
    
    fetchAccountDetails(accountId).then(data => {
      setDetails(data);
      setLoading(false);
    });
  }, [accountId]);  // ✅ Dependency on accountId

  return <div>...</div>;
}
```

#### 6. ✅ DO: Use Separate State for Independent Values

```jsx
// ✅ GOOD: Independent concerns
const [username, setUsername] = useState('');
const [email, setEmail] = useState('');
const [isSubmitting, setIsSubmitting] = useState(false);

// These update independently, so separate state makes sense
```

#### 7. ✅ DO: Handle Loading and Error States

```jsx
// ✅ GOOD: Complete state management
function AccountList() {
  const [accounts, setAccounts] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    setError(null);
    
    fetchAccounts()
      .then(data => {
        setAccounts(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  if (loading) return <Spinner />;
  if (error) return <Error message={error} />;
  return <ul>{accounts.map(...)}</ul>;
}
```

#### 8. ✅ DO: Use TypeScript for Type Safety

```typescript
// ✅ GOOD: Type-safe state
interface Account {
  id: string;
  balance: number;
  type: 'SAVINGS' | 'CHECKING';
}

const [account, setAccount] = useState<Account>({
  id: 'ACC001',
  balance: 5000,
  type: 'SAVINGS'
});

// TypeScript prevents errors
setAccount({ ...account, balance: '5000' });  // ❌ Error: string not assignable to number
```

#### 9. ✅ DO: Clear Timers/Subscriptions in Cleanup

```jsx
// ✅ GOOD: Cleanup to prevent memory leaks
function AutoRefreshBalance() {
  const [balance, setBalance] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      fetchBalance().then(setBalance);
    }, 5000);

    // Cleanup on unmount
    return () => clearInterval(interval);
  }, []);

  return <div>Balance: ${balance}</div>;
}
```

#### 10. ✅ DO: Use Memoization for Expensive Calculations

```jsx
// ✅ GOOD: Memoize expensive derived state
function TransactionSummary({ transactions }) {
  const [filter, setFilter] = useState('ALL');

  const statistics = useMemo(() => {
    console.log('Calculating statistics...');
    
    const filtered = transactions.filter(t => 
      filter === 'ALL' || t.type === filter
    );
    
    return {
      total: filtered.reduce((sum, t) => sum + t.amount, 0),
      count: filtered.length,
      average: filtered.length > 0 
        ? filtered.reduce((sum, t) => sum + t.amount, 0) / filtered.length 
        : 0
    };
  }, [transactions, filter]);  // Only recalculate when these change

  return <div>Total: ${statistics.total}</div>;
}
```

---

### ❌ DON'Ts - Anti-Patterns to Avoid

#### 1. ❌ DON'T: Mutate State Directly

```jsx
// ❌ BAD: Mutating state object
const [account, setAccount] = useState({ balance: 5000 });

account.balance = 6000;  // ❌ MUTATION! React won't detect change
setAccount(account);     // ❌ Same reference, no re-render

// ✅ GOOD: Create new object
setAccount({ ...account, balance: 6000 });  // ✅ New reference

// ❌ BAD: Mutating array
const [transactions, setTransactions] = useState([]);

transactions.push(newTransaction);  // ❌ MUTATION!
setTransactions(transactions);      // ❌ Same reference

// ✅ GOOD: Create new array
setTransactions([...transactions, newTransaction]);  // ✅ New reference
```

#### 2. ❌ DON'T: Use State for Derived Values

```jsx
// ❌ BAD: Storing derived value in state
function TransactionList({ transactions }) {
  const [total, setTotal] = useState(0);

  useEffect(() => {
    const sum = transactions.reduce((acc, t) => acc + t.amount, 0);
    setTotal(sum);  // ❌ Unnecessary state!
  }, [transactions]);

  return <div>Total: ${total}</div>;
}

// ✅ GOOD: Calculate on render
function TransactionList({ transactions }) {
  const total = transactions.reduce((acc, t) => acc + t.amount, 0);
  
  return <div>Total: ${total}</div>;
}
```

#### 3. ❌ DON'T: Call setState in Render

```jsx
// ❌ BAD: setState during render (infinite loop!)
function Component({ value }) {
  const [state, setState] = useState(0);
  
  if (value > 10) {
    setState(value);  // ❌ Sets state during render! Infinite loop!
  }
  
  return <div>{state}</div>;
}

// ✅ GOOD: Use useEffect for side effects
function Component({ value }) {
  const [state, setState] = useState(0);
  
  useEffect(() => {
    if (value > 10) {
      setState(value);  // ✅ Effect runs after render
    }
  }, [value]);
  
  return <div>{state}</div>;
}
```

#### 4. ❌ DON'T: Forget to Spread When Updating Objects/Arrays

```jsx
// ❌ BAD: Loses other properties
const [user, setUser] = useState({
  name: 'John',
  email: 'john@example.com',
  age: 30
});

setUser({ name: 'Jane' });  // ❌ email and age are GONE!

// ✅ GOOD: Spread existing properties
setUser({ ...user, name: 'Jane' });  // ✅ Keeps email and age
```

#### 5. ❌ DON'T: Depend on State Immediately After setState

```jsx
// ❌ BAD: State not updated yet!
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(5);
    console.log(count);  // ❌ Still 0! (not 5)
    
    if (count === 5) {   // ❌ Will be false!
      doSomething();
    }
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// ✅ GOOD: Use the value you're setting
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    const newCount = 5;
    setCount(newCount);
    console.log(newCount);  // ✅ Correct: 5
    
    if (newCount === 5) {   // ✅ Works correctly
      doSomething();
    }
  };
  
  return <button onClick={handleClick}>Click</button>;
}
```

#### 6. ❌ DON'T: Use Index as Key with Dynamic State Lists

```jsx
// ❌ BAD: Index as key with reorderable list
function TodoList() {
  const [todos, setTodos] = useState([
    { text: 'Learn React', done: false },
    { text: 'Build app', done: false }
  ]);

  return (
    <ul>
      {todos.map((todo, index) => (
        <TodoItem 
          key={index}  // ❌ Bad key! Causes bugs when list reorders
          todo={todo} 
        />
      ))}
    </ul>
  );
}

// ✅ GOOD: Unique ID as key
function TodoList() {
  const [todos, setTodos] = useState([
    { id: '1', text: 'Learn React', done: false },
    { id: '2', text: 'Build app', done: false }
  ]);

  return (
    <ul>
      {todos.map(todo => (
        <TodoItem 
          key={todo.id}  // ✅ Stable unique key
          todo={todo} 
        />
      ))}
    </ul>
  );
}
```

#### 7. ❌ DON'T: Create New Functions in setState

```jsx
// ❌ BAD: Creating functions in state
const [callback, setCallback] = useState(() => () => console.log('Hi'));
//                                         ↑    ↑
//                                    initializer  actual value (function)

// Problem: Confusing and causes issues

// ✅ GOOD: Store non-function values, use refs for functions
const callbackRef = useRef(() => console.log('Hi'));

// Or if you really need state:
const [callback, setCallback] = useState(() => {
  return () => console.log('Hi');  // ✅ Clear initializer
});
```

#### 8. ❌ DON'T: Use Too Many State Variables

```jsx
// ❌ BAD: Too many related state variables
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  const [address, setAddress] = useState('');
  const [city, setCity] = useState('');
  const [state, setState] = useState('');
  const [zip, setZip] = useState('');
  // ... 20 more fields
  
  return <form>...</form>;
}

// ✅ GOOD: Group related data
function Form() {
  const [formData, setFormData] = useState({
    firstName: '',
    lastName: '',
    email: '',
    phone: '',
    address: {
      street: '',
      city: '',
      state: '',
      zip: ''
    }
  });
  
  const updateField = (field, value) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };
  
  return <form>...</form>;
}
```

#### 9. ❌ DON'T: Ignore Stale Closure Issues

```jsx
// ❌ BAD: Stale closure in setTimeout
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setTimeout(() => {
      setCount(count + 1);  // ❌ 'count' is stale after 3 seconds
    }, 3000);
  };
  
  // Click twice quickly:
  //   First click: setTimeout with count=0
  //   Second click: setTimeout with count=0 (still!)
  //   After 3 seconds: both set count to 1 ❌
  
  return <button onClick={handleClick}>+</button>;
}

// ✅ GOOD: Functional update
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setTimeout(() => {
      setCount(prev => prev + 1);  // ✅ Uses latest value
    }, 3000);
  };
  
  // Click twice quickly:
  //   First click: setTimeout with updater
  //   Second click: setTimeout with updater
  //   After 3 seconds: count becomes 2 ✅
  
  return <button onClick={handleClick}>+</button>;
}
```

#### 10. ❌ DON'T: Forget State is Asynchronous

```jsx
// ❌ BAD: Expecting immediate state update
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);
    
    // ❌ count is still old value!
    console.log('New count:', count);  // Still 0!
    
    // ❌ This won't work as expected
    if (count > 5) {
      alert('Count exceeded 5');
    }
  };
  
  return <button onClick={handleClick}>+</button>;
}

// ✅ GOOD: Use the value you're setting
function Component() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    const newCount = count + 1;
    setCount(newCount);
    
    // ✅ Use newCount variable
    console.log('New count:', newCount);  // Correct!
    
    if (newCount > 5) {
      alert('Count exceeded 5');  // ✅ Works correctly
    }
  };
  
  return <button onClick={handleClick}>+</button>;
}

// ✅ BETTER: Use useEffect to react to state changes
function Component() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    if (count > 5) {
      alert('Count exceeded 5');  // ✅ Runs after state updates
    }
  }, [count]);
  
  return <button onClick={() => setCount(count + 1)}>+</button>;
}
```

---

### 🎯 Key Takeaways

#### **useState Fundamentals** (5 Points)
1. **State persists** between re-renders (unlike regular variables)
2. **setState triggers re-render** to update UI with new state
3. **State updates are asynchronous** (not immediate)
4. **useState returns array** `[currentValue, updaterFunction]`
5. **Initial value** used only on first render, ignored afterwards

#### **State Update Patterns** (5 Points)
6. **Functional updates** (`setState(prev => prev + 1)`) ensure correctness
7. **Multiple setState calls batched** into single re-render (React 18+)
8. **Always spread objects** when updating (`{ ...prev, field: value }`)
9. **Never mutate state** directly (create new object/array)
10. **Lazy initialization** (`useState(() => expensive())`) for performance

#### **State Organization** (5 Points)
11. **Multiple state variables** for independent concerns
12. **Single object state** for related data
13. **Minimize state** - derive values instead of storing
14. **Co-locate state** close to where it's used
15. **Lift state up** when multiple components need it

#### **Common Patterns** (5 Points)
16. **Loading state** - track async operations
17. **Error state** - handle failures gracefully
18. **Form state** - controlled inputs with state
19. **Previous state tracking** - keep history for undo
20. **State synchronization** - sync with props when needed

#### **Performance** (5 Points)
21. **Lazy initialization** prevents expensive recalculations
22. **Batching** reduces unnecessary re-renders
23. **useMemo** for expensive derived state
24. **Functional updates** avoid closure issues
25. **Minimal state** reduces complexity and re-renders

#### **Banking Application Insights** (5 Points)
26. **Balance state** requires functional updates for correctness
27. **Transaction list** stored as array state
28. **Form inputs** controlled by state for validation
29. **Error handling** with dedicated error state
30. **Account locking** managed with boolean state

---

### 📚 Common Interview Questions

**Q1: What's the difference between state and props?**

**Answer**:
- **State**: Managed WITHIN component, mutable (via setState), triggers re-renders
- **Props**: Passed FROM parent, immutable (read-only), controlled by parent

```jsx
// State: Component controls its own data
function Counter() {
  const [count, setCount] = useState(0);  // Internal state
  return <div>{count}</div>;
}

// Props: Parent controls data
function Display({ count }) {  // Receives from parent
  return <div>{count}</div>;
}
```

**Q2: Why use functional updates?**

**Answer**: Functional updates ensure you always work with the latest state value, especially important with multiple updates or async operations.

```jsx
// ❌ Wrong: Uses stale closure value
setCount(count + 1);
setCount(count + 1);  // count still has old value!

// ✅ Correct: Uses latest state
setCount(prev => prev + 1);
setCount(prev => prev + 1);  // prev has updated value
```

**Q3: Can you update state directly?**

**Answer**: No! State must be immutable. Always use setState to update.

```jsx
// ❌ Wrong: Direct mutation
state.count = 5;  // React won't detect change

// ✅ Correct: Create new state
setState({ ...state, count: 5 });
```

**Q4: When do you use object state vs multiple state variables?**

**Answer**:
- **Object state**: Related data that updates together
- **Multiple state**: Independent values that update separately

```jsx
// Related data → Object
const [account, setAccount] = useState({
  id: 'ACC001',
  balance: 5000,
  type: 'SAVINGS'
});

// Independent values → Separate
const [username, setUsername] = useState('');
const [isLoading, setIsLoading] = useState(false);
```

**Q5: What happens when you call setState multiple times?**

**Answer**: React batches updates into single re-render for performance (React 18+).

```jsx
function handleClick() {
  setCount(1);
  setName('John');
  setFlag(true);
  // Only ONE re-render for all three updates! ✅
}
```

---

### 🔗 Related Topics to Explore

1. **Q04: useEffect Hook** - Side effects and lifecycle
2. **Q05: React Hooks Deep Dive** - All hooks explained
3. **Q11: Context API** - Global state management
4. **Q12: useReducer Hook** - Complex state logic
5. **Q13: Custom Hooks** - Reusable stateful logic
6. **Q21: Redux Fundamentals** - Enterprise state management

---

**End of Q03: State Management with useState** ✅

