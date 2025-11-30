# React.js Architecture & Internal Working

**Target Role:** Senior React Engineer (Full Stack/Frontend)  
**Experience Required:** 7+ years  
**Focus Areas:** Component Architecture, Reconciliation, Fiber, Virtual DOM, Performance

---

## 📋 Table of Contents

1. [React Architecture Overview](#react-architecture-overview)
2. [Virtual DOM & Reconciliation](#virtual-dom--reconciliation)
3. [React Fiber Architecture](#react-fiber-architecture)
4. [Component Lifecycle & Rendering](#component-lifecycle--rendering)
5. [Hooks Internal Implementation](#hooks-internal-implementation)
6. [State Management Architecture](#state-management-architecture)
7. [React Performance Optimization](#react-performance-optimization)
8. [Micro Frontend Architecture](#micro-frontend-architecture)

---

## React Architecture Overview

### Complete React Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        REACT ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                 APPLICATION LAYER (Your Code)                     │ │
│  │                                                                   │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │ │
│  │  │ Components  │  │ Custom Hooks │  │  Business Logic        │ │ │
│  │  │ (JSX)       │  │              │  │  (State Management)    │ │ │
│  │  └─────────────┘  └──────────────┘  └────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                            ↓                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                     REACT CORE API                                │ │
│  │                                                                   │ │
│  │  • React.createElement()     • Hooks API (useState, useEffect)  │ │
│  │  • React.Component           • Context API                       │ │
│  │  • React.memo()              • Suspense/Lazy Loading             │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                            ↓                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                   REACT RECONCILER (Fiber)                        │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  Fiber Tree Construction & Work Loop                        │ │ │
│  │  │  • BeginWork: Create fiber nodes                            │ │ │
│  │  │  • CompleteWork: Finalize fiber nodes                       │ │ │
│  │  │  • CommitWork: Apply changes to DOM                         │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  Reconciliation Algorithm (Diffing)                         │ │ │
│  │  │  • Compare Virtual DOM trees                                │ │ │
│  │  │  • Identify minimum changes needed                          │ │ │
│  │  │  • Prioritize updates (Concurrent Mode)                     │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  Scheduler (Priority-based)                                 │ │ │
│  │  │  • ImmediatePriority: Blocking user input                   │ │ │
│  │  │  • UserBlockingPriority: Click, scroll                      │ │ │
│  │  │  • NormalPriority: Data fetching                            │ │ │
│  │  │  • LowPriority: Analytics                                   │ │ │
│  │  │  • IdlePriority: Background tasks                           │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                            ↓                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    REACT RENDERERS                                │ │
│  │                                                                   │ │
│  │  ┌──────────────┐  ┌─────────────┐  ┌──────────────────────┐   │ │
│  │  │ ReactDOM     │  │ ReactNative │  │ Custom Renderers     │   │ │
│  │  │              │  │             │  │ (React-PDF, Ink)     │   │ │
│  │  │ • Browser    │  │ • iOS       │  │                      │   │ │
│  │  │   DOM APIs   │  │ • Android   │  │ • ReactThreeFiber    │   │ │
│  │  │              │  │   Native    │  │ • React-Canvas       │   │ │
│  │  └──────────────┘  └─────────────┘  └──────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                            ↓                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    BROWSER / PLATFORM                             │ │
│  │                                                                   │ │
│  │  • DOM API (Web)          • Native Components (Mobile)           │ │
│  │  • JavaScript Engine      • Platform APIs                        │ │
│  │  • Event System           • Hardware Acceleration                │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

```

---

## 🔍 Key Components Explained

### 1. React Core API Layer 🎯

**What it is**: The public-facing API developers interact with

**Responsibilities**:
- **JSX Transformation**: Converts JSX to `React.createElement()` calls
- **Component API**: Class components, Function components
- **Hooks System**: useState, useEffect, useContext, etc.
- **Context API**: Global state management
- **Suspense/Lazy**: Code splitting and async rendering

**JSX Transformation Example**:

```jsx
// What you write:
const element = <h1 className="greeting">Hello, {name}!</h1>;

// What Babel transforms it to:
const element = React.createElement(
  'h1',
  { className: 'greeting' },
  'Hello, ',
  name,
  '!'
);

// Internal React Element structure:
{
  type: 'h1',
  props: {
    className: 'greeting',
    children: ['Hello, ', 'John', '!']
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element')
}
```

**Banking Example**:
```jsx
// Component definition
function AccountBalance({ accountId }) {
  const [balance, setBalance] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchBalance(accountId).then(data => {
      setBalance(data.balance);
      setLoading(false);
    });
  }, [accountId]);
  
  if (loading) return <Spinner />;
  
  return (
    <div className="account-balance">
      <h2>Account: {accountId}</h2>
      <p>Balance: ${balance.toFixed(2)}</p>
    </div>
  );
}

// React.createElement() calls generated:
React.createElement(
  'div',
  { className: 'account-balance' },
  React.createElement('h2', null, 'Account: ', accountId),
  React.createElement('p', null, 'Balance: $', balance.toFixed(2))
);
```

---

### 2. React Reconciler (Fiber Architecture) 🔄

**What it is**: The core algorithm that determines what needs to update

**Fiber is**:
- A JavaScript object representing a component instance or DOM node
- Part of a linked list (tree structure)
- Contains information about component's state, props, and relationships

**Fiber Node Structure**:

```javascript
// Internal Fiber node structure
const fiberNode = {
  // Instance
  tag: 0,                    // Type of work (FunctionComponent, ClassComponent, etc.)
  type: AccountBalance,      // Component function/class
  stateNode: null,          // DOM node or component instance
  
  // Fiber relationships (Linked list structure)
  return: parentFiber,      // Parent fiber
  child: firstChildFiber,   // First child
  sibling: nextSiblingFiber, // Next sibling
  
  // Props and state
  pendingProps: { accountId: 'ACC001' },  // New props
  memoizedProps: { accountId: 'ACC001' },  // Previous props
  memoizedState: { balance: 5000 },        // Current state
  updateQueue: null,                        // Queue of state updates
  
  // Effects
  flags: 0,                  // Side effect flags (Update, Placement, Deletion)
  subtreeFlags: 0,          // Child effects
  deletions: null,          // Children to delete
  
  // Priority
  lanes: 0,                 // Priority lanes for this update
  childLanes: 0,            // Priority lanes for children
  
  // Work-in-progress
  alternate: currentFiber   // Alternate fiber (double buffering)
};
```

**Double Buffering** (Current vs Work-in-Progress):

```
┌──────────────────────────────────────────────────────────────┐
│                    DOUBLE BUFFERING                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────┐        ┌───────────────────────┐│
│  │   Current Tree         │◄──────►│ Work-in-Progress Tree ││
│  │   (On Screen)          │ alter- │ (Being Built)         ││
│  │                        │  nate  │                       ││
│  │  • App (Fiber)         │        │  • App (Fiber)        ││
│  │    ├─ Header           │        │    ├─ Header          ││
│  │    ├─ Dashboard        │        │    ├─ Dashboard       ││
│  │    │   ├─ Balance      │        │    │   ├─ Balance*    ││
│  │    │   └─ Transactions │        │    │   └─ Transactions││
│  │    └─ Footer           │        │    └─ Footer          ││
│  │                        │        │                       ││
│  │  * = Updated component │        │  * = Changes applied  ││
│  └────────────────────────┘        └───────────────────────┘│
│            ↑                                    │            │
│            │         Commit Phase               │            │
│            └────────────────────────────────────┘            │
│         (Swap pointers, WIP becomes Current)                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Key Insight**: React maintains TWO fiber trees:
- **Current Tree**: Currently rendered on screen
- **Work-in-Progress Tree**: Being constructed with updates
- After reconciliation completes, WIP tree becomes Current tree

**Fiber Work Loop**:

```javascript
/**
 * React Fiber Work Loop (Simplified)
 * This is the heart of React's rendering engine
 */

function workLoop(deadline) {
  let shouldYield = false;
  
  while (nextUnitOfWork && !shouldYield) {
    // Process one unit of work (one fiber node)
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    
    // Check if we should yield to browser
    shouldYield = deadline.timeRemaining() < 1;
  }
  
  if (nextUnitOfWork) {
    // More work to do, schedule continuation
    requestIdleCallback(workLoop);
  } else {
    // All work done, commit to DOM
    commitRoot();
  }
}

function performUnitOfWork(fiber) {
  // 1. BEGIN WORK: Process current fiber
  beginWork(fiber);
  
  // 2. Traverse down to children
  if (fiber.child) {
    return fiber.child;
  }
  
  // 3. COMPLETE WORK: No more children, finalize this fiber
  completeWork(fiber);
  
  // 4. Move to sibling or go back up
  if (fiber.sibling) {
    return fiber.sibling;
  }
  
  return fiber.return;  // Go back to parent
}

function beginWork(fiber) {
  // Based on fiber type, create/update child fibers
  switch (fiber.tag) {
    case FunctionComponent:
      // Call the function component
      const Component = fiber.type;
      const props = fiber.pendingProps;
      const children = Component(props);
      
      // Reconcile children
      reconcileChildren(fiber, children);
      break;
      
    case ClassComponent:
      // Call render() method
      const instance = fiber.stateNode;
      const children = instance.render();
      reconcileChildren(fiber, children);
      break;
      
    case HostComponent:  // DOM element
      // Create child fibers for DOM children
      const element = fiber.type;
      const props = fiber.pendingProps;
      reconcileChildren(fiber, props.children);
      break;
  }
}

function reconcileChildren(fiber, children) {
  // DIFFING ALGORITHM
  // Compare new children with old children
  // Mark fibers for Update, Placement, or Deletion
  
  let oldFiber = fiber.alternate?.child;
  let newChild = Array.isArray(children) ? children[0] : children;
  let previousNewFiber = null;
  
  while (oldFiber || newChild) {
    let newFiber = null;
    
    // Same type? Update
    if (oldFiber && newChild && oldFiber.type === newChild.type) {
      newFiber = {
        type: oldFiber.type,
        props: newChild.props,
        stateNode: oldFiber.stateNode,  // Reuse DOM node
        alternate: oldFiber,
        flags: Update,  // Mark for update
        return: fiber
      };
    }
    
    // New child? Placement
    if (!oldFiber && newChild) {
      newFiber = {
        type: newChild.type,
        props: newChild.props,
        stateNode: null,  // Will create DOM node
        alternate: null,
        flags: Placement,  // Mark for insertion
        return: fiber
      };
    }
    
    // Old child removed? Deletion
    if (oldFiber && !newChild) {
      oldFiber.flags = Deletion;  // Mark for deletion
      fiber.deletions.push(oldFiber);
    }
    
    // Link fibers
    if (previousNewFiber === null) {
      fiber.child = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    
    previousNewFiber = newFiber;
    oldFiber = oldFiber?.sibling;
    newChild = newChild?.sibling;
  }
}

function completeWork(fiber) {
  // Create or update DOM nodes
  if (fiber.tag === HostComponent) {
    if (!fiber.stateNode) {
      // Create DOM node
      fiber.stateNode = document.createElement(fiber.type);
    }
    
    // Update DOM properties
    updateDOMProperties(fiber.stateNode, fiber.memoizedProps, fiber.pendingProps);
  }
}

function commitRoot() {
  // COMMIT PHASE: Apply all changes to DOM
  // This happens synchronously (cannot be interrupted)
  
  // 1. Process deletions
  commitDeletions();
  
  // 2. Process placements and updates
  commitWork(workInProgressRoot.child);
  
  // 3. Swap current and work-in-progress trees
  currentRoot = workInProgressRoot;
  workInProgressRoot = null;
  
  // 4. Run useEffect callbacks
  commitEffects();
}
```

---

### 3. Reconciliation Algorithm (Diffing) 🔍

**What it is**: The algorithm that compares Virtual DOM trees to find minimum changes

**Key Rules**:
1. **Different types** → Replace entire subtree
2. **Same type** → Update props only
3. **Keys** → Identify which items changed in lists

**Diffing Example - Banking Transaction List**:

```jsx
// Before: 3 transactions
<ul>
  <li key="txn1">Transfer: $500</li>
  <li key="txn2">Deposit: $1000</li>
  <li key="txn3">Withdrawal: $200</li>
</ul>

// After: Added new transaction at position 2
<ul>
  <li key="txn1">Transfer: $500</li>
  <li key="txn4">Payment: $150</li>  // NEW
  <li key="txn2">Deposit: $1000</li>
  <li key="txn3">Withdrawal: $200</li>
</ul>

// WITHOUT KEYS - React's behavior:
// ❌ BAD: Assumes all 4 items changed
// Updates: txn1 (no change), txn2→txn4, txn3→txn2, txn4→txn3
// 4 DOM updates

// WITH KEYS - React's behavior:
// ✅ GOOD: Identifies txn4 is new, others stay same
// Insertion: txn4 at position 2
// 1 DOM insertion (3x faster!)
```

**Diffing Algorithm Steps**:

```
Step 1: Compare Root Elements
┌─────────────────────────────────────────┐
│ Old Tree          New Tree              │
│ ────────          ────────              │
│ <div>             <div>                 │
│   Same type → Continue to children      │
└─────────────────────────────────────────┘

Step 2: Compare Children (Keys)
┌─────────────────────────────────────────┐
│ Old: [txn1, txn2, txn3]                 │
│ New: [txn1, txn4, txn2, txn3]           │
│                                         │
│ Algorithm:                              │
│ 1. Build old children map by key       │
│    { txn1: fiber1, txn2: fiber2, ... } │
│ 2. Iterate new children                │
│    - txn1: Found in map → Update       │
│    - txn4: Not in map → Placement      │
│    - txn2: Found in map → Update       │
│    - txn3: Found in map → Update       │
│ 3. Result:                              │
│    - Reuse: txn1, txn2, txn3           │
│    - Insert: txn4                       │
└─────────────────────────────────────────┘

Step 3: Apply Changes (Commit Phase)
┌─────────────────────────────────────────┐
│ DOM Operations (Minimum changes):       │
│ 1. Create <li> for txn4                │
│ 2. Insert after txn1                   │
│ Done! (All other nodes reused)         │
└─────────────────────────────────────────┘
```

---

### 4. React Scheduler (Priority-based) ⏰

**What it is**: Manages when and in what order updates are processed

**Priority Levels**:

```javascript
// React Priority Lanes (32 lanes for granular control)
const priorities = {
  // Immediate: Must happen now (blocks everything)
  SyncLane: 0b0000000000000000000000000000001,
  
  // User-blocking: Click, input, scroll (high priority)
  InputContinuousLane: 0b0000000000000000000000000000100,
  
  // Default: Most updates (data fetching, state updates)
  DefaultLane: 0b0000000000000000000000000010000,
  
  // Transition: Can be interrupted (page navigation)
  TransitionLane1: 0b0000000000000000000001000000000,
  
  // Idle: Low priority (analytics, logging)
  IdleLane: 0b0100000000000000000000000000000,
};
```

**Banking Example - Priority Scheduling**:

```jsx
function BankingDashboard() {
  const [balance, setBalance] = useState(0);
  const [transactions, setTransactions] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  
  // HIGH PRIORITY: User typing in search
  const handleSearch = (query) => {
    // This update is urgent (user is waiting)
    startTransition(() => {
      setTransactions(filterTransactions(query));  // Can be interrupted
    });
  };
  
  // MEDIUM PRIORITY: Balance update from API
  useEffect(() => {
    fetchBalance().then(data => {
      setBalance(data);  // Normal priority
    });
  }, []);
  
  // LOW PRIORITY: Analytics calculation
  useEffect(() => {
    // Defer to idle time
    requestIdleCallback(() => {
      setAnalytics(calculateAnalytics(transactions));
    });
  }, [transactions]);
  
  return (
    <div>
      <h1>Balance: ${balance}</h1>  {/* High priority */}
      <SearchBar onChange={handleSearch} />  {/* Immediate priority */}
      <TransactionList items={transactions} />  {/* Transition priority */}
      <Analytics data={analytics} />  {/* Idle priority */}
    </div>
  );
}
```

**Scheduler Work Flow**:

```
Time →
─────────────────────────────────────────────────────────────────
Frame 1 (16ms):
  [████████████] User Input (Click) - Immediate Priority
  Main thread busy

Frame 2 (16ms):
  [███░░░░░░░░] Balance Update - Normal Priority
  [░░░░░░░░░░░] Idle time (5ms)
  
Frame 3 (16ms):
  [░░░░░░░░░░░] Idle time (16ms)
  [Analytics calculation runs]  // Low priority

Key:
  █ = React working
  ░ = Idle time
  
---

## Virtual DOM & Reconciliation

### What is Virtual DOM?

**Virtual DOM** is a lightweight JavaScript representation of the actual DOM. It's a tree of plain JavaScript objects that mirrors the structure of the real DOM.

**Why Virtual DOM?**
- **Performance**: Batch multiple changes before touching real DOM
- **Efficiency**: Calculate minimum changes needed (diffing)
- **Cross-platform**: Same code works for web, mobile, VR

**Real DOM vs Virtual DOM**:

```javascript
// Real DOM (Browser API - Slow)
const div = document.createElement('div');
div.className = 'account-card';
div.textContent = 'Balance: $5000';
document.body.appendChild(div);

// Later... update
div.textContent = 'Balance: $5500';  // Triggers reflow/repaint

// Virtual DOM (Plain JavaScript Object - Fast)
const vdom = {
  type: 'div',
  props: {
    className: 'account-card',
    children: 'Balance: $5000'
  }
};

// Later... update
const newVdom = {
  type: 'div',
  props: {
    className: 'account-card',
    children: 'Balance: $5500'  // Only this changed
  }
};

// React compares, finds only text changed
// Applies minimum change to real DOM
```

**Performance Comparison**:

```
Operation: Update 10 items in a list

REAL DOM (Direct manipulation):
  1. Find each element                    ~10ms
  2. Update innerHTML 10 times            ~50ms
  3. Browser reflows/repaints 10 times    ~100ms
  Total: ~160ms ❌

VIRTUAL DOM (React):
  1. Create new Virtual DOM tree          ~2ms
  2. Diff with old Virtual DOM            ~5ms
  3. Calculate minimum changes            ~3ms
  4. Batch update real DOM once           ~15ms
  5. Browser reflows/repaints once        ~10ms
  Total: ~35ms ✅ (4.5x faster!)
```

### Reconciliation Process (Step-by-Step)

**Banking Example**: User updates account balance

```jsx
// Initial render
function AccountCard({ accountId }) {
  const [balance, setBalance] = useState(5000);
  
  return (
    <div className="account-card">
      <h2>Account: {accountId}</h2>
      <p className="balance">Balance: ${balance}</p>
      <button onClick={() => setBalance(5500)}>Add $500</button>
    </div>
  );
}
```

**Reconciliation Steps**:

```
Step 1: State Update Triggered
─────────────────────────────────────────
User clicks button → setBalance(5500)
React schedules update
  └─ Creates "update" object
  └─ Adds to fiber's updateQueue
  └─ Schedules work

Step 2: Render Phase (Interruptible)
─────────────────────────────────────────
React calls AccountCard() function again
  ├─ Input: { accountId: 'ACC001' }
  ├─ useState(5000) → returns 5500 (new state)
  └─ Returns new JSX

New Virtual DOM tree created:
{
  type: 'div',
  props: {
    className: 'account-card',
    children: [
      {
        type: 'h2',
        children: 'Account: ACC001'  // Same
      },
      {
        type: 'p',
        props: { className: 'balance' },
        children: 'Balance: $5500'   // CHANGED!
      },
      {
        type: 'button',
        props: { onClick: [Function] },
        children: 'Add $500'          // Same
      }
    ]
  }
}

Step 3: Diffing (Compare Trees)
─────────────────────────────────────────
Old Virtual DOM ↔ New Virtual DOM

Compare root <div>:
  ✅ Same type ('div')
  ✅ Same className ('account-card')
  → Reuse existing DOM node, check children

Compare <h2>:
  ✅ Same type ('h2')
  ✅ Same content ('Account: ACC001')
  → No update needed

Compare <p>:
  ✅ Same type ('p')
  ✅ Same className ('balance')
  ❌ Different content: '$5000' → '$5500'
  → Mark for update (Update flag)

Compare <button>:
  ✅ Same type ('button')
  ✅ Same content ('Add $500')
  → No update needed

Result: Only <p> element needs updating!

Step 4: Commit Phase (Synchronous, Uninterruptible)
─────────────────────────────────────────
React applies changes to real DOM:

1. Find existing <p> DOM node
2. Update textContent: '$5000' → '$5500'
3. Browser repaints only that <p> element

Total DOM operations: 1 (extremely fast!)

Step 5: Effects Phase
─────────────────────────────────────────
Run useEffect callbacks (if any)
Cleanup previous effects
Run new effects
```

**Optimization - Keys in Lists**:

```jsx
// BAD: No keys (React re-renders all items)
function TransactionList({ transactions }) {
  return (
    <ul>
      {transactions.map(txn => (
        <li>{txn.description} - ${txn.amount}</li>
      ))}
    </ul>
  );
}

// Update from 3 to 4 transactions:
// React assumes all 4 items changed → 4 DOM updates ❌

// GOOD: With keys (React identifies which item is new)
function TransactionList({ transactions }) {
  return (
    <ul>
      {transactions.map(txn => (
        <li key={txn.id}>
          {txn.description} - ${txn.amount}
        </li>
      ))}
    </ul>
  );
}

// Update from 3 to 4 transactions:
// React: "txn1, txn2, txn3 exist, txn4 is new"
// Only txn4 inserted → 1 DOM operation ✅ (4x faster!)
```

---

## React Fiber Architecture

### Why Fiber?

**Problem with Old React (Stack Reconciler)**:
- Rendering was **synchronous** and **uninterruptible**
- Large updates blocked the main thread
- UI froze during heavy updates (janky animations)

**Solution: Fiber Architecture (React 16+)**:
- **Interruptible** rendering (can pause/resume)
- **Priority-based** scheduling (urgent updates first)
- **Concurrent** rendering (work on multiple updates)
- **Time-slicing** (split work into chunks)

**Stack Reconciler vs Fiber**:

```
STACK RECONCILER (Old React):
───────────────────────────────────────────────────
Frame 1 (16ms available):
  [████████████████████] Rendering (25ms)
  Frame dropped! ❌ Janky!

Frame 2:
  [████░░░░░░░░░░░░░░░] Finish rendering
  
User sees: Lag, frozen UI

FIBER RECONCILER (Modern React):
───────────────────────────────────────────────────
Frame 1 (16ms available):
  [███████████████] Render (15ms) - Pause here
  [░] 1ms idle

Frame 2 (16ms available):
  [██████████] Continue rendering (10ms)
  [░░░░░░] 6ms idle

Frame 3:
  [████] Finish rendering (4ms)
  [░░░░░░░░░░░] 12ms idle

User sees: Smooth 60fps! ✅
```

### Fiber Node Deep Dive

**Complete Fiber Structure**:

```javascript
// Banking Dashboard Component Fiber
const fiberNode = {
  // ==========================================
  // IDENTITY & TYPE
  // ==========================================
  tag: FunctionComponent,           // Type of work
  type: BankingDashboard,          // Component function
  key: null,                        // Key prop
  elementType: BankingDashboard,   // Original type
  
  // ==========================================
  // TREE STRUCTURE (Linked List)
  // ==========================================
  return: parentFiber,      // Points to <App> fiber (parent)
  child: headerFiber,       // Points to <Header> fiber (first child)
  sibling: null,            // No sibling components
  index: 0,                 // Position in parent's children
  
  // Child fiber also points to siblings:
  // <Header> → sibling: <AccountList>
  // <AccountList> → sibling: <Footer>
  
  // ==========================================
  // STATE & PROPS
  // ==========================================
  pendingProps: {           // New props from parent
    userId: 'USER123',
    theme: 'dark'
  },
  memoizedProps: {          // Props from last render
    userId: 'USER123',
    theme: 'light'          // Changed!
  },
  memoizedState: {          // Current state
    balance: 5000,
    transactions: [...],
    loading: false
  },
  updateQueue: {            // Pending state updates
    baseState: { balance: 5000 },
    firstUpdate: {
      action: (state) => ({ ...state, balance: 5500 }),
      next: null
    }
  },
  
  // ==========================================
  // HOOKS (Stored in memoizedState as linked list)
  // ==========================================
  // memoizedState structure for hooks:
  // {
  //   memoizedState: value,     // Hook value
  //   queue: updateQueue,       // Update queue
  //   next: nextHook            // Next hook in list
  // }
  
  // Example hook chain:
  // useState → useEffect → useContext
  
  // ==========================================
  // DOM & INSTANCE
  // ==========================================
  stateNode: domNode,       // Actual DOM node or class instance
  ref: null,                // Ref object
  
  // ==========================================
  // EFFECTS (Side effects to commit)
  // ==========================================
  flags: Update | Passive,  // Effect flags
  // Flags:
  // - Placement: Insert new DOM node
  // - Update: Update existing DOM node
  // - Deletion: Remove DOM node
  // - Passive: Has useEffect
  // - Layout: Has useLayoutEffect
  
  subtreeFlags: Placement,  // Child effects
  deletions: [oldChildFiber], // Children to delete
  
  nextEffect: null,         // Next fiber with effects
  firstEffect: null,        // First child with effects
  lastEffect: null,         // Last child with effects
  
  // ==========================================
  // PRIORITY & LANES
  // ==========================================
  lanes: SyncLane,          // Priority of this update
  childLanes: DefaultLane,  // Priority of children's updates
  
  // Lanes system (32 bits for fine-grained priority):
  // 0b00000001 = SyncLane (highest)
  // 0b00000010 = InputContinuous
  // 0b00010000 = DefaultLane
  // 0b01000000 = TransitionLane
  // 0b10000000 = IdleLane (lowest)
  
  // ==========================================
  // DOUBLE BUFFERING
  // ==========================================
  alternate: currentFiberNode,  // Points to alternate fiber
  // Current tree ↔ Work-in-progress tree
  
  // ==========================================
  // CONCURRENCY
  // ==========================================
  actualDuration: 5.23,     // Time spent rendering (ms)
  actualStartTime: 1234567, // When rendering started
  selfBaseDuration: 2.1,    // Time for this component only
  treeBaseDuration: 5.23    // Time for entire subtree
};
```

**Fiber Tree Example - Banking App**:

```
Banking Application Fiber Tree
═══════════════════════════════════════════════════

<App> (Fiber)
  ├─ type: App
  ├─ child: ────────────────────────┐
  ├─ sibling: null                  │
  └─ return: null (root)            │
                                    ↓
         <Header> (Fiber)
           ├─ type: Header
           ├─ child: ──────────────┐
           ├─ sibling: ───────────┼────────────────┐
           └─ return: <App>       │                │
                                  ↓                ↓
              <Logo> (Fiber)     <Nav> (Fiber)    <Dashboard> (Fiber)
                ├─ return: <Header>                ├─ type: Dashboard
                ├─ sibling: <Nav>                  ├─ child: ──────────────┐
                └─ memoizedState: null             ├─ sibling: <Footer>    │
                                                   └─ return: <App>        │
                                                                           ↓
                                                        <AccountCard> (Fiber)
                                                          ├─ type: AccountCard
                                                          ├─ memoizedState: { balance: 5000 }
                                                          ├─ child: ────────────┐
                                                          ├─ sibling: <TransactionList>
                                                          └─ return: <Dashboard>
                                                                                ↓
                                                              <div> (HostComponent Fiber)
                                                                ├─ type: 'div'
                                                                ├─ stateNode: [Actual DOM div]
                                                                ├─ child: ─────────────┐
                                                                └─ props: { className: 'card' }
                                                                                       ↓
                                                                    <h2> (HostComponent Fiber)
                                                                      ├─ stateNode: [DOM h2]
                                                                      ├─ sibling: <p>
                                                                      └─ children: 'Account'
```

**Fiber Traversal Algorithm**:

```javascript
/**
 * How React traverses the Fiber tree
 */

function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;
  
  // STEP 1: Begin work on this fiber
  let next = beginWork(current, unitOfWork, renderLanes);
  
  // Update memoized props
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  
  if (next === null) {
    // STEP 2: No child, complete this fiber
    completeUnitOfWork(unitOfWork);
  } else {
    // STEP 3: Move to child
    workInProgress = next;
  }
}

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;
  
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;
    
    // Complete this fiber
    completeWork(current, completedWork, renderLanes);
    
    // Check for sibling
    const siblingFiber = completedWork.sibling;
    
    if (siblingFiber !== null) {
      // Move to sibling
      workInProgress = siblingFiber;
      return;
    }
    
    // No sibling, move up to parent
    completedWork = returnFiber;
    workInProgress = completedWork;
    
  } while (completedWork !== null);
  
  // Reached root, work complete!
}
```

**Traversal Order Example**:

```
Tree:
    A
   / \
  B   C
 /   / \
D   E   F

Traversal order (Depth-First):
1. A (begin)
2. B (begin)
3. D (begin)
4. D (complete) - No children
5. B (complete) - No more children
6. C (begin)
7. E (begin)
8. E (complete) - No children
9. F (begin)
10. F (complete) - No children
11. C (complete) - No more children
12. A (complete) - Root complete!

Flow:
  A → B → D(✓) ← B(✓) ← A → C → E(✓) ← C → F(✓) ← C(✓) ← A(✓)
  
Phases:
  ───── Render Phase ─────┤├─ Commit Phase ─
  (Interruptible)          (Synchronous)
```

---

## Component Lifecycle & Rendering

### Class Component Lifecycle

```
Component Lifecycle Phases
═══════════════════════════════════════════════════════════

MOUNTING (Component created and inserted into DOM)
─────────────────────────────────────────────────────
  constructor()
    ↓
  static getDerivedStateFromProps()
    ↓
  render()  ← Returns React elements (Virtual DOM)
    ↓
  [React updates DOM]
    ↓
  componentDidMount()  ← Side effects, API calls
  
  
UPDATING (Props or state changed)
─────────────────────────────────────────────────────
  static getDerivedStateFromProps()
    ↓
  shouldComponentUpdate()  ← Return false to skip render
    ↓
  render()
    ↓
  getSnapshotBeforeUpdate()  ← Capture DOM state
    ↓
  [React updates DOM]
    ↓
  componentDidUpdate()  ← Side effects after update
  
  
UNMOUNTING (Component removed from DOM)
─────────────────────────────────────────────────────
  componentWillUnmount()  ← Cleanup (timers, subscriptions)


ERROR HANDLING (Error in child component)
─────────────────────────────────────────────────────
  static getDerivedStateFromError()
    ↓
  componentDidCatch()  ← Log error, show fallback UI
```

**Banking Example - Class Component**:

```jsx
class AccountDashboard extends React.Component {
  constructor(props) {
    super(props);
    // Initialize state
    this.state = {
      balance: 0,
      transactions: [],
      loading: true,
      error: null
    };
    
    console.log('1. Constructor: Component created');
  }
  
  static getDerivedStateFromProps(props, state) {
    // Sync state with props (rare use case)
    console.log('2. getDerivedStateFromProps: Props received', props);
    
    // Example: Reset error if accountId changed
    if (props.accountId !== state.prevAccountId) {
      return {
        error: null,
        prevAccountId: props.accountId
      };
    }
    
    return null;  // No state update needed
  }
  
  shouldComponentUpdate(nextProps, nextState) {
    // Performance optimization: Skip render if nothing changed
    console.log('3. shouldComponentUpdate: Should re-render?');
    
    // Only re-render if balance or transactions changed
    return (
      nextState.balance !== this.state.balance ||
      nextState.transactions.length !== this.state.transactions.length
    );
  }
  
  render() {
    console.log('4. render: Creating Virtual DOM');
    
    const { balance, transactions, loading, error } = this.state;
    
    if (loading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;
    
    return (
      <div className="dashboard">
        <h1>Account Balance</h1>
        <p className="balance">${balance.toFixed(2)}</p>
        <TransactionList transactions={transactions} />
      </div>
    );
  }
  
  getSnapshotBeforeUpdate(prevProps, prevState) {
    // Capture DOM state before update (e.g., scroll position)
    console.log('5. getSnapshotBeforeUpdate: Capture current DOM state');
    
    // Example: Save scroll position
    const list = document.getElementById('transaction-list');
    return list ? list.scrollTop : null;
  }
  
  componentDidMount() {
    console.log('6. componentDidMount: Component in DOM');
    
    // Fetch data (side effect)
    this.fetchAccountData();
    
    // Setup subscriptions
    this.subscription = accountWebSocket.subscribe(
      this.props.accountId,
      this.handleBalanceUpdate
    );
  }
  
  componentDidUpdate(prevProps, prevState, snapshot) {
    console.log('7. componentDidUpdate: Component updated in DOM');
    
    // Restore scroll position if needed
    if (snapshot !== null) {
      const list = document.getElementById('transaction-list');
      if (list) list.scrollTop = snapshot;
    }
    
    // Fetch new data if accountId changed
    if (prevProps.accountId !== this.props.accountId) {
      this.fetchAccountData();
    }
  }
  
  componentWillUnmount() {
    console.log('8. componentWillUnmount: Cleanup');
    
    // Cleanup subscriptions
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
    
    // Cancel pending requests
    if (this.abortController) {
      this.abortController.abort();
    }
  }
  
  static getDerivedStateFromError(error) {
    // Update state so next render shows fallback UI
    console.log('9. getDerivedStateFromError: Error caught');
    return { error: error.message };
  }
  
  componentDidCatch(error, errorInfo) {
    // Log error to service
    console.log('10. componentDidCatch: Log error');
    logErrorToService(error, errorInfo);
  }
  
  // Helper methods
  fetchAccountData = async () => {
    this.abortController = new AbortController();
    
    try {
      const response = await fetch(
        `/api/accounts/${this.props.accountId}`,
        { signal: this.abortController.signal }
      );
      
      const data = await response.json();
      
      this.setState({
        balance: data.balance,
        transactions: data.transactions,
        loading: false
      });
    } catch (error) {
      if (error.name !== 'AbortError') {
        this.setState({ error: error.message, loading: false });
      }
    }
  };
  
  handleBalanceUpdate = (newBalance) => {
    this.setState({ balance: newBalance });
  };
}
```

### Function Component Lifecycle (Hooks)

```
Function Component Lifecycle with Hooks
═══════════════════════════════════════════════════════════

RENDER PHASE (Interruptible)
─────────────────────────────────────────────────────
  Function Component Called
    ↓
  useState() / useReducer()  ← Get current state
    ↓
  useMemo()  ← Memoized calculations
    ↓
  useCallback()  ← Memoized functions
    ↓
  render (return JSX)  ← Create Virtual DOM
    ↓
  React Reconciliation (Fiber work)


COMMIT PHASE (Synchronous)
─────────────────────────────────────────────────────
  useLayoutEffect() cleanup  ← Synchronous cleanup
    ↓
  [React updates DOM]
    ↓
  useLayoutEffect() effect  ← Synchronous effects
    ↓
  [Browser paints]
    ↓
  useEffect() cleanup  ← Async cleanup (previous render)
    ↓
  useEffect() effect  ← Async effects (after paint)


UNMOUNT
─────────────────────────────────────────────────────
  useLayoutEffect() cleanup
    ↓
  useEffect() cleanup
```

**Banking Example - Function Component**:

```jsx
function AccountDashboard({ accountId }) {
  // STATE HOOKS
  const [balance, setBalance] = useState(0);
  const [transactions, setTransactions] = useState([]);
  const [loading, setLoading] = useState(true);
  
  console.log('1. Component function called (Render phase)');
  
  // MEMOIZATION HOOKS
  const sortedTransactions = useMemo(() => {
    console.log('2. useMemo: Calculating sorted transactions');
    return transactions.sort((a, b) => 
      new Date(b.date) - new Date(a.date)
    );
  }, [transactions]);
  
  const handleRefresh = useCallback(() => {
    console.log('3. useCallback: Memoized callback');
    setLoading(true);
    fetchAccountData();
  }, [accountId]);
  
  // EFFECT HOOKS
  
  // useLayoutEffect: Runs synchronously after DOM updates (before paint)
  useLayoutEffect(() => {
    console.log('4. useLayoutEffect: After DOM update, before paint');
    
    // Measure DOM (e.g., for animations)
    const element = document.getElementById('balance');
    if (element) {
      const height = element.offsetHeight;
      console.log('   Balance element height:', height);
    }
    
    return () => {
      console.log('4b. useLayoutEffect cleanup');
    };
  }, [balance]);
  
  // useEffect: Runs asynchronously after paint (doesn't block browser)
  useEffect(() => {
    console.log('5. useEffect: After paint (async)');
    
    const abortController = new AbortController();
    
    async function fetchData() {
      try {
        const response = await fetch(
          `/api/accounts/${accountId}`,
          { signal: abortController.signal }
        );
        
        const data = await response.json();
        setBalance(data.balance);
        setTransactions(data.transactions);
        setLoading(false);
        
        console.log('   Data fetched:', data);
      } catch (error) {
        if (error.name !== 'AbortError') {
          console.error('   Fetch error:', error);
        }
      }
    }
    
    fetchData();
    
    // Cleanup function (runs before next effect or unmount)
    return () => {
      console.log('5b. useEffect cleanup: Cancel fetch');
      abortController.abort();
    };
  }, [accountId]);  // Re-run when accountId changes
  
  // WebSocket subscription
  useEffect(() => {
    console.log('6. useEffect: Setup WebSocket');
    
    const ws = new WebSocket(`wss://api.bank.com/balance/${accountId}`);
    
    ws.onmessage = (event) => {
      const newBalance = JSON.parse(event.data).balance;
      setBalance(newBalance);
      console.log('   Balance updated via WebSocket:', newBalance);
    };
    
    return () => {
      console.log('6b. useEffect cleanup: Close WebSocket');
      ws.close();
    };
  }, [accountId]);
  
  // Logging effect (runs on every render)
  useEffect(() => {
    console.log('7. useEffect: Component rendered/updated');
    console.log('   Balance:', balance);
    console.log('   Transactions:', transactions.length);
  });  // No dependency array = runs every render
  
  if (loading) return <Spinner />;
  
  return (
    <div className="dashboard">
      <h1>Account Dashboard</h1>
      <p id="balance" className="balance">
        ${balance.toFixed(2)}
      </p>
      <button onClick={handleRefresh}>Refresh</button>
      <TransactionList transactions={sortedTransactions} />
    </div>
  );
}
```

**Execution Order - Complete Flow**:

```
Initial Mount:
══════════════════════════════════════════════════════════
1. Component function called (Render phase)
2. useMemo: Calculating sorted transactions
3. useCallback: Memoized callback
   [React creates Virtual DOM]
   [React commits to real DOM]
4. useLayoutEffect: After DOM update, before paint
   [Browser paints]
5. useEffect: After paint (async) - Fetch data
6. useEffect: Setup WebSocket
7. useEffect: Component rendered/updated

State Update (balance changes):
══════════════════════════════════════════════════════════
1. Component function called (Render phase)
2. useMemo: (skipped - transactions didn't change)
3. useCallback: (skipped - accountId didn't change)
   [React creates Virtual DOM]
   [React reconciles]
   [React commits changes to real DOM]
4b. useLayoutEffect cleanup  ← Previous effect
4. useLayoutEffect: After DOM update, before paint
   [Browser paints]
5b. useEffect cleanup (not run - accountId same)
6b. useEffect cleanup (not run - accountId same)
7. useEffect: Component rendered/updated

Unmount:
══════════════════════════════════════════════════════════
4b. useLayoutEffect cleanup
5b. useEffect cleanup: Cancel fetch
6b. useEffect cleanup: Close WebSocket
---

## Hooks Internal Implementation

### How Hooks Work Internally

**Key Concept**: Hooks are stored as a **linked list** on the fiber node

```javascript
// Fiber node's memoizedState for function components
fiber.memoizedState = {
  // Hook 1: useState
  memoizedState: 5000,  // Current balance value
  queue: {
    pending: null,  // Pending updates
    dispatch: setBalance  // setState function
  },
  next: {
    // Hook 2: useEffect
    memoizedState: {
      tag: HookHasEffect | HookPassive,
      create: () => { /* effect function */ },
      destroy: undefined,  // cleanup function
      deps: [accountId],   // dependencies
      next: null
    },
    next: {
      // Hook 3: useMemo
      memoizedState: [sortedTransactions, [transactions]],
      next: null
    }
  }
};
```

**Why Rules of Hooks Exist**:

```jsx
// BAD: Conditional hook (breaks linked list order)
function BadComponent({ showBalance }) {
  const [name, setName] = useState('John');
  
  if (showBalance) {
    const [balance, setBalance] = useState(0);  // ❌ ERROR!
  }
  
  const [transactions, setTransactions] = useState([]);
  
  return <div>...</div>;
}

// Problem:
// Render 1 (showBalance = true):
//   Hook order: name → balance → transactions
// Render 2 (showBalance = false):
//   Hook order: name → transactions
// React: "Where did balance go? Transactions is now hook #2!"
// Result: State mismatched! 💥

// GOOD: Always same hook order
function GoodComponent({ showBalance }) {
  const [name, setName] = useState('John');
  const [balance, setBalance] = useState(0);  // ✅ Always called
  const [transactions, setTransactions] = useState([]);
  
  return (
    <div>
      {name}
      {showBalance && <p>${balance}</p>}  {/* Conditional rendering */}
      <TransactionList items={transactions} />
    </div>
  );
}
```

### useState Implementation (Simplified)

```javascript
/**
 * Simplified useState implementation
 * Shows how React tracks state internally
 */

let currentFiber = null;  // Currently rendering fiber
let currentHookIndex = 0;  // Current hook position in list

function useState(initialState) {
  const fiber = currentFiber;
  const hooks = fiber.memoizedState || [];
  const hook = hooks[currentHookIndex];
  
  // First render: Initialize hook
  if (!hook) {
    const newHook = {
      memoizedState: initialState,  // Current state value
      queue: [],  // Queue of pending updates
      dispatch: null  // setState function
    };
    
    hooks[currentHookIndex] = newHook;
    fiber.memoizedState = hooks;
  }
  
  const currentHook = hooks[currentHookIndex];
  
  // Process queued updates
  if (currentHook.queue.length > 0) {
    currentHook.queue.forEach(update => {
      if (typeof update === 'function') {
        currentHook.memoizedState = update(currentHook.memoizedState);
      } else {
        currentHook.memoizedState = update;
      }
    });
    
    currentHook.queue = [];  // Clear queue
  }
  
  // Create setState function
  const setState = (update) => {
    // Add update to queue
    currentHook.queue.push(update);
    
    // Schedule re-render
    scheduleUpdateOnFiber(fiber);
  };
  
  currentHook.dispatch = setState;
  currentHookIndex++;
  
  return [currentHook.memoizedState, setState];
}

// Banking example using our useState
function AccountCard() {
  currentFiber = getCurrentFiber();
  currentHookIndex = 0;
  
  const [balance, setBalance] = useState(5000);  // Hook #0
  const [currency, setCurrency] = useState('USD');  // Hook #1
  
  // Later... user clicks button
  // setBalance(6000)
  //   → Adds 6000 to hook #0's queue
  //   → Schedules re-render
  
  // Next render:
  //   Hook #0: Process queue, memoizedState = 6000
  //   Hook #1: No updates, memoizedState = 'USD'
  
  return <div>${balance} {currency}</div>;
}
```

### useEffect Implementation (Simplified)

```javascript
/**
 * Simplified useEffect implementation
 */

const HookPassive = 0b00000001;  // Passive effect (useEffect)
const HookLayout = 0b00000010;   // Layout effect (useLayoutEffect)
const HookHasEffect = 0b00000100; // Has effect to run

function useEffect(create, deps) {
  const fiber = currentFiber;
  const hooks = fiber.memoizedState || [];
  const hook = hooks[currentHookIndex];
  
  let hasChanged = true;
  
  if (hook) {
    // Check if dependencies changed
    const prevDeps = hook.memoizedState.deps;
    
    if (deps && prevDeps) {
      hasChanged = !deps.every((dep, i) => Object.is(dep, prevDeps[i]));
    }
  }
  
  const effect = {
    tag: HookPassive | (hasChanged ? HookHasEffect : 0),
    create,  // Effect function
    destroy: hook?.memoizedState.destroy,  // Previous cleanup
    deps,
    next: null
  };
  
  if (!hooks[currentHookIndex]) {
    hooks[currentHookIndex] = {
      memoizedState: effect
    };
    fiber.memoizedState = hooks;
  } else {
    hooks[currentHookIndex].memoizedState = effect;
  }
  
  // Mark fiber as having effects
  if (hasChanged) {
    fiber.flags |= HookPassive;
  }
  
  currentHookIndex++;
}

// Effect execution (after render committed)
function commitPassiveEffects(fiber) {
  const hooks = fiber.memoizedState;
  
  if (!hooks) return;
  
  hooks.forEach(hook => {
    const effect = hook.memoizedState;
    
    if (effect.tag & HookHasEffect) {
      // Run cleanup from previous effect
      if (effect.destroy) {
        effect.destroy();
      }
      
      // Run new effect
      const cleanup = effect.create();
      
      // Store cleanup for next time
      effect.destroy = cleanup;
    }
  });
}

// Banking example
function BalanceMonitor({ accountId }) {
  currentFiber = getCurrentFiber();
  currentHookIndex = 0;
  
  useEffect(() => {
    console.log('Effect: Fetch balance');
    
    const ws = new WebSocket(`/balance/${accountId}`);
    
    ws.onmessage = (msg) => {
      console.log('Balance updated:', msg.data);
    };
    
    // Cleanup function
    return () => {
      console.log('Cleanup: Close WebSocket');
      ws.close();
    };
  }, [accountId]);  // Re-run only when accountId changes
  
  // Render 1 (accountId = 'ACC001'):
  //   Effect runs, WebSocket opened
  
  // Render 2 (accountId = 'ACC001'):
  //   Deps unchanged, effect skipped
  
  // Render 3 (accountId = 'ACC002'):
  //   Deps changed:
  //     1. Run cleanup (close old WebSocket)
  //     2. Run new effect (open new WebSocket)
  
  return <div>Monitoring...</div>;
}
```

### useMemo & useCallback Implementation

```javascript
/**
 * Simplified useMemo implementation
 */

function useMemo(create, deps) {
  const fiber = currentFiber;
  const hooks = fiber.memoizedState || [];
  const hook = hooks[currentHookIndex];
  
  if (!hook) {
    // First render: Calculate value
    const value = create();
    
    hooks[currentHookIndex] = {
      memoizedState: [value, deps]
    };
    fiber.memoizedState = hooks;
    
    currentHookIndex++;
    return value;
  }
  
  // Check if dependencies changed
  const [prevValue, prevDeps] = hook.memoizedState;
  
  if (deps && prevDeps) {
    const hasChanged = !deps.every((dep, i) => Object.is(dep, prevDeps[i]));
    
    if (!hasChanged) {
      // Deps unchanged, return cached value
      currentHookIndex++;
      return prevValue;
    }
  }
  
  // Deps changed, recalculate
  const nextValue = create();
  hook.memoizedState = [nextValue, deps];
  
  currentHookIndex++;
  return nextValue;
}

function useCallback(callback, deps) {
  // useCallback is just useMemo that returns the function
  return useMemo(() => callback, deps);
}

// Banking example with performance optimization
function TransactionList({ transactions, onTransactionClick }) {
  // Heavy calculation: Sort and filter transactions
  const processedTransactions = useMemo(() => {
    console.log('useMemo: Processing transactions...');
    
    return transactions
      .filter(txn => txn.status === 'COMPLETED')
      .sort((a, b) => new Date(b.date) - new Date(a.date))
      .map(txn => ({
        ...txn,
        formattedAmount: `$${txn.amount.toFixed(2)}`,
        formattedDate: new Date(txn.date).toLocaleDateString()
      }));
  }, [transactions]);  // Only recalculate when transactions array changes
  
  // Memoized callback to prevent child re-renders
  const handleClick = useCallback((txnId) => {
    console.log('useCallback: Transaction clicked', txnId);
    onTransactionClick(txnId);
  }, [onTransactionClick]);
  
  // Scenario 1: transactions unchanged
  //   → useMemo returns cached value (fast!)
  //   → useCallback returns same function reference
  //   → Child components don't re-render
  
  // Scenario 2: transactions changed
  //   → useMemo recalculates (necessary)
  //   → useCallback returns same function (onTransactionClick unchanged)
  //   → Only TransactionItem with new data re-renders
  
  return (
    <ul>
      {processedTransactions.map(txn => (
        <TransactionItem
          key={txn.id}
          transaction={txn}
          onClick={handleClick}  // Same reference = no re-render
        />
      ))}
    </ul>
  );
}
```

---

## State Management Architecture

### State Management Patterns in React

```
State Management Hierarchy
═══════════════════════════════════════════════════════════

LOCAL STATE (Component-specific)
─────────────────────────────────────────────────────
  • useState / useReducer
  • Managed by React component
  • Example: Form inputs, toggles, local UI state
  
  When to use:
    ✓ UI state (modal open/closed)
    ✓ Form inputs
    ✓ Component-specific flags
    
  Banking Example:
    const [showPassword, setShowPassword] = useState(false);
    const [accountFilter, setAccountFilter] = useState('all');

    
LIFTED STATE (Shared between siblings)
─────────────────────────────────────────────────────
  • State in common parent
  • Passed down via props
  • Callbacks passed down for updates
  
  When to use:
    ✓ State shared by 2-3 components
    ✓ Sibling communication
    
  Banking Example:
    function Dashboard() {
      const [selectedAccount, setSelectedAccount] = useState(null);
      
      return (
        <>
          <AccountSelector onSelect={setSelectedAccount} />
          <AccountDetails account={selectedAccount} />
        </>
      );
    }


CONTEXT (Subtree-wide state)
─────────────────────────────────────────────────────
  • Context API
  • Avoid prop drilling
  • Theme, auth, language
  
  When to use:
    ✓ Deep component trees
    ✓ Configuration (theme, locale)
    ✓ Authentication state
    
  Banking Example:
    const AuthContext = createContext();
    
    function App() {
      const [user, setUser] = useState(null);
      
      return (
        <AuthContext.Provider value={{ user, setUser }}>
          <Dashboard />  {/* Any depth can access */}
        </AuthContext.Provider>
      );
    }


GLOBAL STATE (Application-wide)
─────────────────────────────────────────────────────
  • Redux, Zustand, MobX
  • Single source of truth
  • Time-travel debugging
  • Middleware support
  
  When to use:
    ✓ Complex state logic
    ✓ Many components need same data
    ✓ Need debugging tools
    ✓ Server state caching
    
  Banking Example:
    // Redux store
    {
      accounts: [...],
      transactions: [...],
      user: {...},
      preferences: {...}
    }


SERVER STATE (Remote data)
─────────────────────────────────────────────────────
  • React Query, SWR, Apollo Client
  • Caching, refetching, synchronization
  • Optimistic updates
  
  When to use:
    ✓ API data
    ✓ Need caching
    ✓ Real-time updates
    ✓ Offline support
    
  Banking Example:
    const { data, isLoading } = useQuery(
      ['balance', accountId],
      () => fetchBalance(accountId),
      { staleTime: 60000 }  // Cache for 1 minute
    );
```

### Context API Deep Dive

**How Context Works Internally**:

```jsx
// Context creation
const BalanceContext = createContext(defaultValue);

// Internally:
{
  $$typeof: Symbol.for('react.context'),
  _currentValue: defaultValue,  // Current context value
  _currentValue2: defaultValue,  // For concurrent mode
  Provider: ContextProvider,
  Consumer: ContextConsumer
}
```

**Provider Implementation**:

```jsx
function BalanceProvider({ children }) {
  const [balance, setBalance] = useState(5000);
  const [currency, setCurrency] = useState('USD');
  
  const value = useMemo(
    () => ({
      balance,
      currency,
      setBalance,
      setCurrency
    }),
    [balance, currency]
  );
  
  return (
    <BalanceContext.Provider value={value}>
      {children}
    </BalanceContext.Provider>
  );
}

// Internally, Provider:
// 1. Stores value on context._currentValue
// 2. Marks all consumer fibers as needing update
// 3. Re-renders consumers when value changes
```

**Consumer Usage**:

```jsx
// Method 1: useContext hook (preferred)
function AccountCard() {
  const { balance, currency } = useContext(BalanceContext);
  
  return <div>{balance} {currency}</div>;
}

// Method 2: Consumer component (legacy)
function AccountCard() {
  return (
    <BalanceContext.Consumer>
      {({ balance, currency }) => (
        <div>{balance} {currency}</div>
      )}
    </BalanceContext.Consumer>
  );
}
```

**Context Update Flow**:

```
User clicks "Add $500" button
  ↓
setBalance(5500) called
  ↓
Provider re-renders
  ↓
context._currentValue = { balance: 5500, currency: 'USD' }
  ↓
React finds all consumers of this context
  ├─ AccountCard (consumer)
  ├─ BalanceDisplay (consumer)
  └─ TransactionSummary (consumer)
  ↓
Mark consumers as needing update
  ↓
Re-render consumers with new context value
  ↓
Only consumers re-render (siblings unaffected!)
```

**Performance Optimization - Context**:

```jsx
// BAD: Entire context changes on every state update
function BadProvider({ children }) {
  const [balance, setBalance] = useState(5000);
  const [transactions, setTransactions] = useState([]);
  
  const value = {
    balance,
    setBalance,
    transactions,
    setTransactions
  };  // ❌ New object on every render!
  
  return (
    <BankContext.Provider value={value}>
      {children}
    </BankContext.Provider>
  );
}

// Result: All consumers re-render even if they only use balance!

// GOOD: Split contexts by concern
function GoodProviders({ children }) {
  return (
    <BalanceProvider>
      <TransactionProvider>
        {children}
      </TransactionProvider>
    </BalanceProvider>
  );
}

function BalanceProvider({ children }) {
  const [balance, setBalance] = useState(5000);
  
  const value = useMemo(
    () => ({ balance, setBalance }),
    [balance]
  );  // ✅ Only changes when balance changes
  
  return (
    <BalanceContext.Provider value={value}>
      {children}
    </BalanceContext.Provider>
  );
}

function TransactionProvider({ children }) {
  const [transactions, setTransactions] = useState([]);
  
  const value = useMemo(
    () => ({ transactions, setTransactions }),
    [transactions]
  );
  
  return (
    <TransactionContext.Provider value={value}>
      {children}
    </TransactionContext.Provider>
  );
}

// Result:
//   AccountCard (uses BalanceContext) → Only re-renders when balance changes
//   TransactionList (uses TransactionContext) → Only re-renders when transactions change
```

---
