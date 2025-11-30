# Q06: List Comprehensions vs Generator Expressions

## Question: What are the differences between list comprehensions and generator expressions?

---

## Answer:

**List Comprehension**: Creates a list in memory immediately  
**Generator Expression**: Creates a generator (lazy evaluation)

---

## 1. Basic Syntax

```python
# List comprehension []
transaction_amounts = [txn['amount'] for txn in transactions]

# Generator expression ()
transaction_amounts_gen = (txn['amount'] for txn in transactions)

# Key difference
print(type([x for x in range(5)]))  # <class 'list'>
print(type((x for x in range(5))))  # <class 'generator'>
```

---

## 2. Memory Comparison

```python
import sys

transactions = [{'amount': i} for i in range(1_000_000)]

# List comprehension - loads all into memory
amounts_list = [txn['amount'] for txn in transactions]
print(f"List: {sys.getsizeof(amounts_list):,} bytes")  # ~8 MB

# Generator expression - lazy evaluation
amounts_gen = (txn['amount'] for txn in transactions)
print(f"Generator: {sys.getsizeof(amounts_gen):,} bytes")  # ~128 bytes

# ✅ Use generator when you only iterate once
total = sum(txn['amount'] for txn in transactions)  # Memory efficient!
```

---

## 3. Banking Example: Filtering Transactions

```python
transactions = [
    {'id': 'TXN001', 'amount': 100, 'status': 'completed'},
    {'id': 'TXN002', 'amount': 5000, 'status': 'pending'},
    {'id': 'TXN003', 'amount': 15000, 'status': 'completed'},
]

# List comprehension with condition
high_value = [txn for txn in transactions if txn['amount'] > 1000]
print(high_value)  # Returns list immediately

# Generator expression with condition
high_value_gen = (txn for txn in transactions if txn['amount'] > 1000)
print(list(high_value_gen))  # Convert to list to see results
```

---

## 4. Nested Comprehensions

```python
# Flatten nested transaction data
accounts = [
    {'id': 'ACC001', 'transactions': [100, 200, 300]},
    {'id': 'ACC002', 'transactions': [400, 500]}
]

# List comprehension (nested)
all_amounts = [amount 
               for account in accounts 
               for amount in account['transactions']]
print(all_amounts)  # [100, 200, 300, 400, 500]

# With condition
large_amounts = [amount 
                 for account in accounts 
                 for amount in account['transactions'] 
                 if amount > 250]
print(large_amounts)  # [300, 400, 500]
```

---

## 5. Dictionary & Set Comprehensions

```python
transactions = [
    {'id': 'TXN001', 'amount': 100},
    {'id': 'TXN002', 'amount': 200},
]

# Dictionary comprehension
txn_dict = {txn['id']: txn['amount'] for txn in transactions}
print(txn_dict)  # {'TXN001': 100, 'TXN002': 200}

# Set comprehension (unique amounts)
amounts_set = {txn['amount'] for txn in transactions + transactions}
print(amounts_set)  # {100, 200} - duplicates removed
```

---

## 6. When to Use Which?

```python
# ✅ Use List Comprehension when:
# - Need to access data multiple times
# - Need to sort/index the results
# - Dataset is small

active_accounts = [acc for acc in accounts if acc['status'] == 'active']
active_accounts.sort(key=lambda x: x['balance'])  # Can sort list
first = active_accounts[0]  # Can index

# ✅ Use Generator Expression when:
# - Large dataset
# - Only iterate once
# - Pipeline operations

# Memory efficient for large files
total = sum(txn['amount'] for txn in read_transactions('large_file.csv'))
```

---

## 7. Performance Example

```python
import time

def measure_time(func):
    start = time.time()
    result = func()
    return time.time() - start, result

transactions = [{'amount': i} for i in range(10_000_000)]

# List comprehension
time_list, _ = measure_time(lambda: [txn['amount'] for txn in transactions])
print(f"List: {time_list:.2f}s")

# Generator expression with sum
time_gen, _ = measure_time(lambda: sum(txn['amount'] for txn in transactions))
print(f"Generator: {time_gen:.2f}s")  # Faster and less memory!
```

---

## Best Practices:

### ✅ DO:
- Use generators for large datasets or one-time iteration
- Use list comprehensions for small datasets needing multiple access
- Use comprehensions instead of map/filter for readability
- Keep comprehensions simple and readable

### ❌ DON'T:
- Make comprehensions too complex (use regular loops)
- Use list comprehension just to iterate once
- Nest more than 2-3 levels deep
- Ignore memory constraints with large data

---

## 8. Real-World Scenario: Processing Large Transaction Files

```python
import time
import sys
import tracemalloc
from typing import List, Dict

def process_transactions_list(file_path: str) -> List[Dict]:
    """
    ❌ List comprehension: Loads entire file into memory
    Problem: 10GB transaction file causes memory issues
    """
    # Read all transactions at once
    transactions = [
        {
            'id': f'TXN{i:08d}',
            'amount': i * 10.5,
            'account': f'ACC{i % 10000:05d}',
            'fee': i * 0.01
        }
        for i in range(5_000_000)  # 5 million transactions
    ]
    
    # Filter and transform
    high_value = [
        {'id': txn['id'], 'total': txn['amount'] + txn['fee']}
        for txn in transactions
        if txn['amount'] > 1000
    ]
    
    return high_value

def process_transactions_generator(file_path: str):
    """
    ✅ Generator expression: Memory efficient
    Solution: Process one transaction at a time
    """
    # Generate transactions on-the-fly
    transactions = (
        {
            'id': f'TXN{i:08d}',
            'amount': i * 10.5,
            'account': f'ACC{i % 10000:05d}',
            'fee': i * 0.01
        }
        for i in range(5_000_000)
    )
    
    # Filter and transform (still lazy)
    high_value = (
        {'id': txn['id'], 'total': txn['amount'] + txn['fee']}
        for txn in transactions
        if txn['amount'] > 1000
    )
    
    return high_value

# Performance comparison
print("\n=== Performance Benchmark ===")

# Test List Comprehension
print("\n1️⃣ List Comprehension (loads all in memory):")
tracemalloc.start()
start = time.time()
txns_list = process_transactions_list('transactions.csv')
list_time = time.time() - start
list_current, list_peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

list_mb = list_peak / 1024 / 1024
print(f"   Time: {list_time:.2f}s")
print(f"   Memory: {list_mb:.2f} MB")
print(f"   Count: {len(txns_list):,} high-value transactions")

# Test Generator Expression
print("\n2️⃣ Generator Expression (lazy evaluation):")
tracemalloc.start()
start = time.time()
txns_gen = process_transactions_generator('transactions.csv')
count = sum(1 for _ in txns_gen)  # Consume generator
gen_time = time.time() - start
gen_current, gen_peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

gen_mb = gen_peak / 1024 / 1024
print(f"   Time: {gen_time:.2f}s")
print(f"   Memory: {gen_mb:.2f} MB")
print(f"   Count: {count:,} high-value transactions")

# Results
memory_saved = list_mb - gen_mb
memory_percent = (memory_saved / list_mb) * 100
time_diff = ((list_time - gen_time) / list_time) * 100

print(f"\n📊 Results:")
print(f"   💾 Memory saved: {memory_saved:.2f} MB ({memory_percent:.1f}%)")
print(f"   ⚡ Time difference: {abs(time_diff):.1f}% {'faster' if time_diff > 0 else 'slower'}")
```

---

## 9. Scenario: Nested Comprehensions for Complex Data

```python
# Banking scenario: Process transactions for multiple accounts
accounts_data = [
    {
        'account_id': 'ACC001',
        'customer': 'John Doe',
        'transactions': [
            {'date': '2024-01-01', 'amount': 100, 'type': 'DEBIT'},
            {'date': '2024-01-05', 'amount': 500, 'type': 'CREDIT'},
            {'date': '2024-01-10', 'amount': 50, 'type': 'DEBIT'},
        ]
    },
    {
        'account_id': 'ACC002',
        'customer': 'Jane Smith',
        'transactions': [
            {'date': '2024-01-02', 'amount': 1000, 'type': 'CREDIT'},
            {'date': '2024-01-08', 'amount': 200, 'type': 'DEBIT'},
            {'date': '2024-01-15', 'amount': 300, 'type': 'DEBIT'},
        ]
    },
    {
        'account_id': 'ACC003',
        'customer': 'Bob Johnson',
        'transactions': [
            {'date': '2024-01-03', 'amount': 5000, 'type': 'CREDIT'},
            {'date': '2024-01-12', 'amount': 100, 'type': 'DEBIT'},
        ]
    }
]

print("\n=== Nested Comprehensions ===")

# 1. Flatten all transactions with account info
all_transactions = [
    {
        'account_id': account['account_id'],
        'customer': account['customer'],
        **transaction
    }
    for account in accounts_data
    for transaction in account['transactions']
]

print(f"\n1️⃣ All transactions: {len(all_transactions)}")
for txn in all_transactions[:3]:
    print(f"   {txn['account_id']} - {txn['customer']}: {txn['type']} ${txn['amount']}")

# 2. Get all CREDIT amounts > 500
high_credits = [
    {
        'account': account['account_id'],
        'amount': txn['amount'],
        'date': txn['date']
    }
    for account in accounts_data
    for txn in account['transactions']
    if txn['type'] == 'CREDIT' and txn['amount'] > 500
]

print(f"\n2️⃣ High-value credits (> $500): {len(high_credits)}")
for credit in high_credits:
    print(f"   {credit['account']}: ${credit['amount']} on {credit['date']}")

# 3. Calculate total debits per account
account_debits = [
    {
        'account_id': account['account_id'],
        'customer': account['customer'],
        'total_debits': sum(
            txn['amount']
            for txn in account['transactions']
            if txn['type'] == 'DEBIT'
        )
    }
    for account in accounts_data
]

print(f"\n3️⃣ Total debits per account:")
for acc in account_debits:
    print(f"   {acc['account_id']} ({acc['customer']}): ${acc['total_debits']}")

# 4. Create matrix of daily totals
from collections import defaultdict

# Group by date and account
daily_matrix = defaultdict(lambda: defaultdict(float))
for account in accounts_data:
    for txn in account['transactions']:
        daily_matrix[txn['date']][account['account_id']] = txn['amount']

print(f"\n4️⃣ Daily transaction matrix:")
for date in sorted(daily_matrix.keys())[:3]:
    print(f"   {date}: {dict(daily_matrix[date])}")
```

---

## 10. Scenario: Dictionary & Set Comprehensions

```python
transactions = [
    {'id': 'TXN001', 'account': 'ACC001', 'amount': 100, 'type': 'DEBIT'},
    {'id': 'TXN002', 'account': 'ACC002', 'amount': 500, 'type': 'CREDIT'},
    {'id': 'TXN003', 'account': 'ACC001', 'amount': 200, 'type': 'DEBIT'},
    {'id': 'TXN004', 'account': 'ACC003', 'amount': 1000, 'type': 'CREDIT'},
    {'id': 'TXN005', 'account': 'ACC002', 'amount': 50, 'type': 'DEBIT'},
]

print("\n=== Dictionary & Set Comprehensions ===")

# 1. Dictionary comprehension: Transaction lookup
txn_lookup = {txn['id']: txn for txn in transactions}
print(f"\n1️⃣ Transaction lookup dictionary:")
print(f"   TXN001: {txn_lookup['TXN001']}")
print(f"   Quick lookup: O(1) time complexity")

# 2. Dictionary: Account balance summary
from collections import defaultdict
account_balances = defaultdict(float)
for txn in transactions:
    if txn['type'] == 'CREDIT':
        account_balances[txn['account']] += txn['amount']
    else:
        account_balances[txn['account']] -= txn['amount']

# Convert to regular dict with comprehension
balances = {acc: bal for acc, bal in account_balances.items()}
print(f"\n2️⃣ Account balances:")
for acc, bal in sorted(balances.items()):
    print(f"   {acc}: ${bal:+.2f}")

# 3. Dictionary: Group transactions by account
account_txns = {
    account_id: [txn for txn in transactions if txn['account'] == account_id]
    for account_id in {txn['account'] for txn in transactions}
}

print(f"\n3️⃣ Transactions grouped by account:")
for acc, txns in sorted(account_txns.items()):
    print(f"   {acc}: {len(txns)} transactions")

# 4. Set comprehension: Unique accounts
unique_accounts = {txn['account'] for txn in transactions}
print(f"\n4️⃣ Unique accounts: {sorted(unique_accounts)}")

# 5. Set comprehension: High-value transaction IDs
high_value_ids = {txn['id'] for txn in transactions if txn['amount'] > 100}
print(f"\n5️⃣ High-value transaction IDs (> $100): {sorted(high_value_ids)}")

# 6. Dictionary: Conditional mapping
# Map account to transaction count, only if count > 1
active_accounts = {
    acc: len(txns)
    for acc, txns in account_txns.items()
    if len(txns) > 1
}
print(f"\n6️⃣ Active accounts (> 1 transaction): {active_accounts}")
```

---

## 11. When to Use List vs Generator - Decision Matrix

```python
import sys

print("\n=== Decision Matrix: List vs Generator ===")

# Scenario 1: Need multiple iterations
print("\n1️⃣ Scenario: Multiple iterations needed")

# ❌ Generator - can only iterate once
def get_accounts_gen():
    return (f'ACC{i:03d}' for i in range(5))

accounts_gen = get_accounts_gen()
print("   Generator - First pass:", list(accounts_gen))
print("   Generator - Second pass:", list(accounts_gen))  # Empty!

# ✅ List - iterate multiple times
accounts_list = [f'ACC{i:03d}' for i in range(5)]
print("   List - First pass:", accounts_list)
print("   List - Second pass:", accounts_list)  # Still works!

print("   ✅ Winner: List (reusable)")

# Scenario 2: Large dataset, single iteration
print("\n2️⃣ Scenario: Large dataset (10M items), single iteration")

# List approach
large_list = [i for i in range(10_000_000)]
list_size = sys.getsizeof(large_list) / 1024 / 1024
print(f"   List: {list_size:.2f} MB in memory")

# Generator approach
large_gen = (i for i in range(10_000_000))
gen_size = sys.getsizeof(large_gen) / 1024
print(f"   Generator: {gen_size:.2f} KB in memory")

print(f"   ✅ Winner: Generator ({list_size/gen_size*1024:.0f}x less memory)")

# Scenario 3: Need indexing or length
print("\n3️⃣ Scenario: Need random access, sorting, or length")

txns_list = [i * 100 for i in range(1000)]
print(f"   List - Length: {len(txns_list)}")
print(f"   List - Item 500: {txns_list[500]}")
print(f"   List - Sorted: {sorted(txns_list, reverse=True)[:3]}")

txns_gen = (i * 100 for i in range(1000))
# print(len(txns_gen))  # ❌ TypeError!
# print(txns_gen[500])  # ❌ TypeError!

print("   ✅ Winner: List (supports indexing, len, sorting)")

# Scenario 4: Chaining operations
print("\n4️⃣ Scenario: Chaining multiple operations")

data = range(1_000_000)

# ✅ Generator chain (memory efficient)
result = (
    x * 2
    for x in (
        x
        for x in data
        if x % 2 == 0
    )
    if x > 1000
)

print(f"   Generator chain: ~{sys.getsizeof(result)} bytes")
print(f"   Result (first 5): {list(next(result) for _ in range(5))}")
print("   ✅ Winner: Generator (lazy evaluation)")

print("\n📊 Summary:")
print("""
   Use LIST when:
   • Need multiple iterations
   • Need indexing or length
   • Need to sort or reverse
   • Small dataset (< 10,000 items)
   • Need to modify elements
   
   Use GENERATOR when:
   • Large dataset (> 100,000 items)
   • Single iteration
   • Memory constrained
   • Infinite sequences
   • Pipeline operations
""")
```

---

## 12. Performance: Readability vs Efficiency

```python
import time

print("\n=== Performance vs Readability ===")

transactions = [
    {'id': f'TXN{i:05d}', 'amount': i * 10, 'type': 'DEBIT' if i % 2 else 'CREDIT'}
    for i in range(100_000)
]

# Test 1: Traditional loop
print("\n1️⃣ Traditional for loop:")
start = time.time()
result1 = []
for txn in transactions:
    if txn['type'] == 'DEBIT' and txn['amount'] > 5000:
        result1.append(txn['amount'] * 1.01)
loop_time = time.time() - start
print(f"   Time: {loop_time:.4f}s")
print(f"   Results: {len(result1)}")

# Test 2: List comprehension
print("\n2️⃣ List comprehension:")
start = time.time()
result2 = [
    txn['amount'] * 1.01
    for txn in transactions
    if txn['type'] == 'DEBIT' and txn['amount'] > 5000
]
comp_time = time.time() - start
print(f"   Time: {comp_time:.4f}s")
print(f"   Results: {len(result2)}")
print(f"   ⚡ {((loop_time - comp_time) / loop_time * 100):.1f}% faster than loop")

# Test 3: Generator expression
print("\n3️⃣ Generator expression:")
start = time.time()
result3 = (
    txn['amount'] * 1.01
    for txn in transactions
    if txn['type'] == 'DEBIT' and txn['amount'] > 5000
)
result3_list = list(result3)  # Consume generator
gen_time = time.time() - start
print(f"   Time: {gen_time:.4f}s")
print(f"   Results: {len(result3_list)}")
print(f"   ⚡ {((loop_time - gen_time) / loop_time * 100):.1f}% faster than loop")

# Test 4: Complex nested comprehension (readability suffers)
print("\n4️⃣ Complex nested comprehension:")
# ❌ BAD: Hard to read
result4 = [
    sum([txn['amount'] for txn in transactions if txn['type'] == t and txn['amount'] > threshold])
    for t in ['DEBIT', 'CREDIT']
    for threshold in [1000, 5000, 10000]
]

print(f"   Result: {result4[:3]}...")
print("   ❌ Hard to understand! Use regular loop instead.")

# ✅ BETTER: Break it down
print("\n5️⃣ Readable alternative:")
results = []
for txn_type in ['DEBIT', 'CREDIT']:
    for threshold in [1000, 5000, 10000]:
        total = sum(
            txn['amount']
            for txn in transactions
            if txn['type'] == txn_type and txn['amount'] > threshold
        )
        results.append(total)

print(f"   Result: {results[:3]}...")
print("   ✅ Much clearer!")

print("\n📊 Key Insight:")
print("   • List comprehensions are 15-30% faster than loops")
print("   • Generators have similar speed but save memory")
print("   • Readability matters: Don't nest > 2 levels")
print("   • Use regular loops for complex logic")
```

---

## Key Takeaways:

### List Comprehensions vs Generator Expressions:

| Feature | List `[]` | Generator `()` |
|---------|-----------|----------------|
| **Memory** | All in memory | One item at a time |
| **Speed** | Fast access | Lazy evaluation |
| **Reusable** | Yes ✅ | No ❌ (one-time) |
| **Indexing** | Yes ✅ | No ❌ |
| **Length** | Yes `len()` ✅ | No ❌ |
| **Sorting** | Yes ✅ | Must convert first |
| **Best for** | Small-medium data | Large datasets |

### When to Use List Comprehensions:

✅ **ALWAYS Use for:**
- **Small datasets** (< 10,000 items)
- **Multiple iterations** needed
- **Need indexing** or slicing
- **Need length** (`len()`)
- **Need sorting/reversing**
- **Modifying elements**
- **UI display** (need all data)
- **Caching results**

### When to Use Generator Expressions:

✅ **ALWAYS Use for:**
- **Large datasets** (> 100,000 items)
- **Memory constraints**
- **Single iteration**
- **Pipeline operations**
- **Infinite sequences**
- **Streaming data**
- **File processing** (line by line)

### Performance Numbers:

```python
# 1 million transactions:
# List: ~85 MB memory, 0.15s
# Generator: ~0.1 MB memory, 0.14s
# Winner: Generator (850x less memory, similar speed)

# 1000 transactions (multiple access):
# List: ~0.1 MB, instant random access
# Generator: Must regenerate each time
# Winner: List (convenience)
```

### Banking Applications:

**Use List Comprehensions:**
- Dashboard data (current transactions)
- Account lists (hundreds of accounts)
- Recent activity (last 50 transactions)
- Search results (paginated)
- Dropdown options
- Form validation data

**Use Generator Expressions:**
- End-of-day batch processing (millions)
- Statement generation (all customers)
- Audit log analysis (GB of logs)
- Data migration (millions of records)
- CSV export (large files)
- Real-time data streams

### Common Mistakes:

```python
# ❌ Using list for large one-time iteration
all_txns = [txn for txn in read_10_million_transactions()]
total = sum(txn['amount'] for txn in all_txns)  # Wasteful!

# ✅ Use generator directly
total = sum(txn['amount'] for txn in read_10_million_transactions())

# ❌ Too complex comprehension
result = [[y*2 for y in [x for x in data if x > 0]] for data in datasets if data]

# ✅ Break it down
result = []
for data in datasets:
    if data:
        filtered = [x for x in data if x > 0]
        doubled = [y * 2 for y in filtered]
        result.append(doubled)
```
