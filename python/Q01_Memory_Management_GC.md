# Q01: Python Memory Management & Garbage Collection

## Question: How does Python manage memory and what is the garbage collection mechanism?

---

## Answer:

Python uses **automatic memory management** with two key mechanisms:

1. **Reference Counting** - Primary mechanism
2. **Garbage Collection** - Handles circular references

---

## 1. Reference Counting

Every object has a reference count. When count reaches 0, memory is freed immediately.

```python
import sys

# Reference counting example
x = [1, 2, 3]  # refcount = 1
print(sys.getrefcount(x))  # 2 (includes getrefcount's reference)

y = x  # refcount = 2
z = x  # refcount = 3

del y  # refcount = 2
del z  # refcount = 1
del x  # refcount = 0, memory freed
```

---

## 2. Circular References & Garbage Collector

**Problem**: Reference counting can't handle circular references.

```python
import gc

class Transaction:
    def __init__(self, id):
        self.id = id
        self.related = None

# Create circular reference
txn1 = Transaction("TXN001")
txn2 = Transaction("TXN002")
txn1.related = txn2  # txn1 → txn2
txn2.related = txn1  # txn2 → txn1 (circular!)

# Even after deletion, objects remain in memory
del txn1
del txn2

# Garbage collector handles this
collected = gc.collect()  # Force GC
print(f"Collected {collected} objects")
```

---

## 3. Banking Example: Memory-Efficient Transaction Processing

```python
from typing import List, Iterator

class TransactionProcessor:
    """Memory-efficient transaction processing for banking"""
    
    def process_large_file(self, filename: str) -> Iterator[dict]:
        """Process transactions without loading entire file into memory"""
        # ✅ GOOD: Uses generator (memory efficient)
        with open(filename, 'r') as f:
            for line in f:
                yield self.parse_transaction(line)
    
    def process_large_file_bad(self, filename: str) -> List[dict]:
        """❌ BAD: Loads all transactions into memory"""
        with open(filename, 'r') as f:
            return [self.parse_transaction(line) for line in f]
    
    def parse_transaction(self, line: str) -> dict:
        parts = line.strip().split(',')
        return {
            'id': parts[0],
            'amount': float(parts[1]),
            'account': parts[2]
        }

# Usage
processor = TransactionProcessor()

# Memory efficient - processes one at a time
for txn in processor.process_large_file('transactions.csv'):
    # Process each transaction
    print(f"Processing {txn['id']}: ${txn['amount']}")
```

---

## 4. Monitoring Memory Usage

```python
import sys
import gc
import tracemalloc

class MemoryMonitor:
    """Monitor memory usage for banking application"""
    
    @staticmethod
    def get_object_size(obj) -> int:
        """Get size of object in bytes"""
        return sys.getsizeof(obj)
    
    @staticmethod
    def monitor_memory():
        """Monitor current memory usage"""
        tracemalloc.start()
        
        # Simulate processing
        transactions = [{'id': i, 'amount': i * 100} for i in range(10000)]
        
        current, peak = tracemalloc.get_traced_memory()
        print(f"Current: {current / 1024 / 1024:.2f} MB")
        print(f"Peak: {peak / 1024 / 1024:.2f} MB")
        
        tracemalloc.stop()
    
    @staticmethod
    def gc_stats():
        """Print garbage collection statistics"""
        print(f"GC thresholds: {gc.get_threshold()}")
        print(f"GC counts: {gc.get_count()}")

# Usage
monitor = MemoryMonitor()
monitor.monitor_memory()
monitor.gc_stats()
```

---

## 5. Best Practices

### ✅ DO:
- Use generators for large datasets
- Close files and connections explicitly
- Use context managers (`with` statement)
- Delete large objects when done
- Use `__slots__` for classes with many instances

### ❌ DON'T:
- Load entire large files into memory
- Create unnecessary circular references
- Ignore memory leaks in long-running processes
- Keep references to large objects unnecessarily

---

## 6. Optimization Example

```python
class OptimizedAccount:
    """Memory-efficient account class"""
    __slots__ = ['account_id', 'balance']  # Saves memory
    
    def __init__(self, account_id: str, balance: float):
        self.account_id = account_id
        self.balance = balance

# Compare memory usage
import sys

class RegularAccount:
    def __init__(self, account_id: str, balance: float):
        self.account_id = account_id
        self.balance = balance

regular = RegularAccount("ACC001", 1000.0)
optimized = OptimizedAccount("ACC001", 1000.0)

print(f"Regular: {sys.getsizeof(regular.__dict__)} bytes")
print(f"Optimized: {sys.getsizeof(optimized)} bytes")
# Optimized uses ~40% less memory
```

---

## 7. Real-World Scenario: Memory Leak in Production

```python
import weakref
import gc
from typing import Dict, List

class TransactionCache:
    """
    Scenario: Banking app caching transactions for quick lookup
    Problem: Cache grows indefinitely causing memory leak
    """
    
    def __init__(self):
        # ❌ BAD: Strong references keep objects in memory forever
        self.cache_bad: Dict[str, dict] = {}
        
        # ✅ GOOD: Weak references allow garbage collection
        self.cache_good: weakref.WeakValueDictionary = weakref.WeakValueDictionary()
    
    def add_transaction_bad(self, txn_id: str, data: dict):
        """Memory leak - objects never freed"""
        self.cache_bad[txn_id] = data
        # Even if data is deleted elsewhere, cache keeps reference!
    
    def add_transaction_good(self, txn_id: str, data: dict):
        """Allows garbage collection when data deleted elsewhere"""
        # Wrap in a class to make it weakly referenceable
        class TxnData:
            def __init__(self, d):
                self.data = d
        
        self.cache_good[txn_id] = TxnData(data)

# Demo the problem
cache = TransactionCache()

# Add 1 million transactions
for i in range(1_000_000):
    txn = {'id': f'TXN{i:07d}', 'amount': i * 100}
    cache.add_transaction_bad(f'TXN{i:07d}', txn)

print(f"Cache size: {len(cache.cache_bad):,}")  # 1,000,000

# Even if we try to clear old transactions, they stay in cache!
del txn  # This doesn't help
gc.collect()
print(f"Cache size after GC: {len(cache.cache_bad):,}")  # Still 1,000,000!

# Solution: Clear cache explicitly or use weak references
cache.cache_bad.clear()
print(f"After clear: {len(cache.cache_bad)}"  # 0
```

---

## 8. Scenario: When to Use `__slots__`

```python
import sys
from dataclasses import dataclass

# Scenario: Storing 10 million customer accounts in memory

class AccountRegular:
    """Regular class with __dict__"""
    def __init__(self, account_id: str, balance: float, customer_name: str):
        self.account_id = account_id
        self.balance = balance
        self.customer_name = customer_name

class AccountOptimized:
    """Memory-efficient with __slots__"""
    __slots__ = ['account_id', 'balance', 'customer_name']
    
    def __init__(self, account_id: str, balance: float, customer_name: str):
        self.account_id = account_id
        self.balance = balance
        self.customer_name = customer_name

# Memory comparison
regular = AccountRegular("ACC001", 1000.0, "John Doe")
optimized = AccountOptimized("ACC001", 1000.0, "John Doe")

print(f"Regular account: {sys.getsizeof(regular.__dict__)} bytes")
print(f"Optimized account: {sys.getsizeof(optimized)} bytes")
print(f"Savings per object: ~{sys.getsizeof(regular.__dict__) - sys.getsizeof(optimized)} bytes")

# For 10 million accounts:
accounts_count = 10_000_000
regular_memory = sys.getsizeof(regular.__dict__) * accounts_count / (1024**2)
optimized_memory = sys.getsizeof(optimized) * accounts_count / (1024**2)

print(f"\n10 Million Accounts:")
print(f"Regular: ~{regular_memory:.2f} MB")
print(f"Optimized: ~{optimized_memory:.2f} MB")
print(f"Memory saved: ~{regular_memory - optimized_memory:.2f} MB")

# ✅ When to use __slots__:
# - Creating millions of instances
# - Fixed set of attributes
# - Memory is a constraint

# ❌ When NOT to use __slots__:
# - Need dynamic attributes
# - Using multiple inheritance (complex)
# - Only a few instances
```

---

## 9. Scenario: Reference Counting in Action

```python
import sys

class BankAccount:
    def __init__(self, account_id: str):
        self.account_id = account_id
        print(f"Account {account_id} created")
    
    def __del__(self):
        print(f"Account {self.account_id} destroyed")

# Reference counting demo
print("\n=== Reference Counting Demo ===")

# Create account (refcount = 1)
account = BankAccount("ACC001")
print(f"Refcount: {sys.getrefcount(account) - 1}")  # -1 for getrefcount's own ref

# Add another reference (refcount = 2)
account_ref = account
print(f"Refcount: {sys.getrefcount(account) - 1}")

# Add to list (refcount = 3)
accounts_list = [account]
print(f"Refcount: {sys.getrefcount(account) - 1}")

# Remove references
del account_ref
print(f"After del account_ref: {sys.getrefcount(account) - 1}")

accounts_list.clear()
print(f"After clear list: {sys.getrefcount(account) - 1}")

# Delete last reference - object destroyed immediately!
del account
print("Object destroyed (you should see destruction message above)")
```

---

## 10. Scenario: Garbage Collection for Circular References

```python
import gc
import weakref

class Transaction:
    """Transaction with related transactions"""
    def __init__(self, txn_id: str):
        self.txn_id = txn_id
        self.related_transactions = []  # Can cause circular refs!
    
    def __repr__(self):
        return f"Transaction({self.txn_id})"
    
    def __del__(self):
        print(f"Transaction {self.txn_id} deleted")

# Scenario: Creating circular references
print("\n=== Circular Reference Problem ===")

# Create two transactions that reference each other
txn1 = Transaction("TXN001")
txn2 = Transaction("TXN002")

# Create circular reference
txn1.related_transactions.append(txn2)  # txn1 → txn2
txn2.related_transactions.append(txn1)  # txn2 → txn1 (circular!)

print(f"TXN001 refcount: {sys.getrefcount(txn1) - 1}")
print(f"TXN002 refcount: {sys.getrefcount(txn2) - 1}")

# Delete references
del txn1
del txn2

print("\nDeleted txn1 and txn2, but objects still in memory!")
print("(No deletion messages printed)")

# Force garbage collection to clean up circular refs
print("\nRunning garbage collector...")
collected = gc.collect()
print(f"Collected {collected} objects")
print("(Now you see deletion messages)")

# ✅ Solution: Use weak references for related transactions
print("\n=== Solution with Weak References ===")

class TransactionSafe:
    def __init__(self, txn_id: str):
        self.txn_id = txn_id
        # Use weak references to avoid circular refs
        self.related_transactions = []  # Store weak refs
    
    def add_related(self, txn):
        # Store weak reference instead of strong reference
        self.related_transactions.append(weakref.ref(txn))
    
    def get_related(self):
        # Dereference weak refs (may return None if object deleted)
        return [ref() for ref in self.related_transactions if ref() is not None]
    
    def __del__(self):
        print(f"TransactionSafe {self.txn_id} deleted immediately")

txn1_safe = TransactionSafe("TXN001")
txn2_safe = TransactionSafe("TXN002")

txn1_safe.add_related(txn2_safe)
txn2_safe.add_related(txn1_safe)

print("Deleting safe transactions...")
del txn1_safe
del txn2_safe
print("Deleted immediately! (No GC needed)")
```

---

## 11. Production Monitoring Example

```python
import tracemalloc
import time
from typing import List

class MemoryMonitor:
    """
    Production-ready memory monitoring for banking application
    """
    
    def __init__(self, threshold_mb: float = 100.0):
        self.threshold_mb = threshold_mb
        self.snapshots = []
    
    def start(self):
        """Start monitoring"""
        tracemalloc.start()
        print("Memory monitoring started")
    
    def take_snapshot(self, label: str = ""):
        """Take memory snapshot"""
        snapshot = tracemalloc.take_snapshot()
        current, peak = tracemalloc.get_traced_memory()
        
        current_mb = current / 1024 / 1024
        peak_mb = peak / 1024 / 1024
        
        self.snapshots.append({
            'label': label,
            'current_mb': current_mb,
            'peak_mb': peak_mb,
            'snapshot': snapshot
        })
        
        print(f"\n[{label}]")
        print(f"Current: {current_mb:.2f} MB")
        print(f"Peak: {peak_mb:.2f} MB")
        
        if current_mb > self.threshold_mb:
            print(f"⚠️  WARNING: Memory usage exceeds {self.threshold_mb} MB!")
            self.analyze_top_allocations(snapshot)
        
        return current_mb, peak_mb
    
    def analyze_top_allocations(self, snapshot, top=10):
        """Show top memory allocations"""
        print(f"\nTop {top} memory allocations:")
        top_stats = snapshot.statistics('lineno')
        
        for index, stat in enumerate(top_stats[:top], 1):
            print(f"{index}. {stat}")
    
    def compare_snapshots(self, snapshot1_idx: int, snapshot2_idx: int):
        """Compare two snapshots"""
        if len(self.snapshots) < 2:
            print("Need at least 2 snapshots to compare")
            return
        
        snap1 = self.snapshots[snapshot1_idx]
        snap2 = self.snapshots[snapshot2_idx]
        
        diff_mb = snap2['current_mb'] - snap1['current_mb']
        
        print(f"\nMemory change from '{snap1['label']}' to '{snap2['label']}':")
        print(f"Difference: {diff_mb:+.2f} MB")
        
        if diff_mb > 10:
            print("⚠️  Memory increased significantly!")

# Real-world usage
monitor = MemoryMonitor(threshold_mb=50.0)
monitor.start()

# Initial snapshot
monitor.take_snapshot("Application Start")

# Simulate processing transactions
print("\nProcessing 1 million transactions...")
transactions = []
for i in range(1_000_000):
    transactions.append({
        'id': f'TXN{i:07d}',
        'amount': i * 100,
        'timestamp': time.time()
    })

monitor.take_snapshot("After Loading Transactions")

# Process transactions
print("\nProcessing transactions...")
total = sum(txn['amount'] for txn in transactions)

monitor.take_snapshot("After Processing")

# Clear transactions
print("\nClearing transactions...")
transactions.clear()
transactions = None

import gc
gc.collect()

monitor.take_snapshot("After Cleanup")

# Compare
monitor.compare_snapshots(0, 1)
monitor.compare_snapshots(1, 3)
```

---

## Key Takeaways:

### Memory Management:
- **Reference Counting**: Automatic but can't handle circular references
- **Garbage Collection**: Runs periodically to clean circular refs
- **Weak References**: Prevent circular reference memory leaks
- **`__slots__`**: Save 40%+ memory for millions of objects

### When to Optimize:
✅ **DO Optimize When:**
- Handling millions of objects (accounts, transactions)
- Long-running processes (services, APIs)
- Memory-constrained environments (containers with limits)
- Caching large datasets

❌ **DON'T Optimize When:**
- Premature optimization (< 10,000 objects)
- Short-lived scripts
- Memory is abundant
- Code readability suffers significantly

### Banking Context:
- **Critical for**: Processing millions of daily transactions
- **Monitoring**: Essential in production to detect leaks
- **Resource limits**: Docker containers, cloud instances
- **Cost impact**: Memory = Money in cloud environments
