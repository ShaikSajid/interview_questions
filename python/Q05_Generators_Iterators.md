# Q05: Generators and Iterators

## Question: What is the difference between generators and iterators?

---

## Answer:

**Iterators**: Objects with `__iter__()` and `__next__()` methods  
**Generators**: Functions that use `yield` to produce values lazily

---

## 1. Iterator Example

```python
class TransactionIterator:
    """Custom iterator for transactions"""
    
    def __init__(self, transactions: list):
        self.transactions = transactions
        self.index = 0
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.index >= len(self.transactions):
            raise StopIteration
        txn = self.transactions[self.index]
        self.index += 1
        return txn

# Usage
txns = [{'id': 1, 'amount': 100}, {'id': 2, 'amount': 200}]
iterator = TransactionIterator(txns)

for txn in iterator:
    print(f"Transaction {txn['id']}: ${txn['amount']}")
```

---

## 2. Generator Example (Simpler!)

```python
def transaction_generator(transactions: list):
    """Generator for transactions"""
    for txn in transactions:
        yield txn  # Pauses here and returns value

# Usage (same as iterator)
txns = [{'id': 1, 'amount': 100}, {'id': 2, 'amount': 200}]

for txn in transaction_generator(txns):
    print(f"Transaction {txn['id']}: ${txn['amount']}")
```

---

## 3. Key Differences

| Feature | Iterator | Generator |
|---------|----------|----------|
| **Implementation** | Class with `__iter__` and `__next__` | Function with `yield` |
| **Code** | More verbose | Concise |
| **State** | Manually tracked | Automatically tracked |
| **Memory** | Can store all data | Lazy evaluation |
| **Use case** | Complex iteration logic | Simple iteration |

---

## 4. Banking Example: Memory-Efficient Processing

```python
def read_large_transaction_file(filename: str):
    """Generator for processing large files"""
    with open(filename, 'r') as f:
        for line in f:
            # Process one line at a time (memory efficient!)
            parts = line.strip().split(',')
            yield {
                'id': parts[0],
                'amount': float(parts[1]),
                'account': parts[2]
            }

# Process millions of transactions without loading all into memory
for txn in read_large_transaction_file('transactions.csv'):
    if txn['amount'] > 10000:
        print(f"Large transaction: {txn}")
```

---

## 5. Generator Expressions (Like List Comprehensions)

```python
# List comprehension (loads all into memory)
amounts = [txn['amount'] for txn in transactions]  # Creates list

# Generator expression (lazy evaluation)
amounts_gen = (txn['amount'] for txn in transactions)  # Creates generator

# Example
transactions = [{'amount': i * 100} for i in range(1000000)]

# ❌ BAD: List comprehension uses lots of memory
total = sum([txn['amount'] for txn in transactions])

# ✅ GOOD: Generator expression is memory efficient
total = sum(txn['amount'] for txn in transactions)
```

---

## 6. Advanced: Generator with send()

```python
def transaction_processor():
    """Generator that receives and processes transactions"""
    total = 0
    while True:
        amount = yield total  # Receive value and return total
        if amount is None:
            break
        total += amount

# Usage
processor = transaction_processor()
next(processor)  # Initialize generator

processor.send(100)  # Send transaction amount
processor.send(200)
total = processor.send(300)
print(f"Total: ${total}")  # 600
```

---

## 7. Banking Example: Infinite Transaction Stream

```python
def transaction_stream():
    """Infinite generator for real-time transactions"""
    txn_id = 1
    while True:
        yield {
            'id': f'TXN{txn_id:06d}',
            'amount': txn_id * 10,
            'timestamp': time.time()
        }
        txn_id += 1

# Process first 10 transactions only
import itertools
for txn in itertools.islice(transaction_stream(), 10):
    print(txn)
```

---

## 8. Performance Comparison

```python
import sys

# List (loads all into memory)
txn_list = [i for i in range(1000000)]
print(f"List size: {sys.getsizeof(txn_list)} bytes")  # ~8 MB

# Generator (lazy evaluation)
txn_gen = (i for i in range(1000000))
print(f"Generator size: {sys.getsizeof(txn_gen)} bytes")  # ~128 bytes!
```

---

## Best Practices:

### ✅ DO:
- Use generators for large datasets
- Use generators for streaming data
- Use generator expressions instead of list comprehensions when possible
- Chain generators for data pipelines

### ❌ DON'T:
- Use generators when you need to access data multiple times
- Use generators when you need random access
- Forget generators are one-time use (can't rewind)
- Store generators expecting to reuse them

---

## 9. Real-World Scenario: Processing Large Transaction Files

```python
import csv
import gzip
from typing import Iterator, Dict
import time
import sys

class TransactionFileProcessor:
    """
    Process large transaction files (100GB+) without loading into memory
    Real banking scenario: End-of-day transaction processing
    """
    
    def read_transactions(self, filename: str) -> Iterator[Dict]:
        """
        Generator: Read transactions one at a time
        Memory efficient - processes 10M+ transactions
        """
        with open(filename, 'r') as f:
            reader = csv.DictReader(f)
            for row in reader:
                # Parse and validate one transaction at a time
                yield {
                    'transaction_id': row['id'],
                    'amount': float(row['amount']),
                    'account_id': row['account'],
                    'timestamp': row['timestamp']
                }
    
    def read_compressed_transactions(self, filename: str) -> Iterator[Dict]:
        """
        Handle compressed files (common in banking for archival)
        """
        with gzip.open(filename, 'rt') as f:
            reader = csv.DictReader(f)
            for row in reader:
                yield {
                    'transaction_id': row['id'],
                    'amount': float(row['amount']),
                    'account_id': row['account']
                }
    
    def filter_high_value(self, transactions: Iterator[Dict], 
                          threshold: float) -> Iterator[Dict]:
        """
        Generator pipeline: Filter high-value transactions
        """
        for txn in transactions:
            if txn['amount'] > threshold:
                yield txn
    
    def calculate_fees(self, transactions: Iterator[Dict]) -> Iterator[Dict]:
        """
        Generator pipeline: Add fee calculation
        """
        for txn in transactions:
            fee = txn['amount'] * 0.01  # 1% fee
            yield {
                **txn,
                'fee': fee,
                'total': txn['amount'] + fee
            }
    
    def batch_transactions(self, transactions: Iterator[Dict], 
                           batch_size: int) -> Iterator[list]:
        """
        Generator: Group transactions into batches
        Useful for bulk database inserts
        """
        batch = []
        for txn in transactions:
            batch.append(txn)
            if len(batch) >= batch_size:
                yield batch
                batch = []
        
        # Yield remaining transactions
        if batch:
            yield batch

# Demo: Processing pipeline
print("=== Transaction Processing Pipeline ===")

# Create sample file
import tempfile
import os

with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.csv') as f:
    f.write("id,amount,account,timestamp\n")
    for i in range(10000):
        f.write(f"TXN{i:05d},{i * 10},ACC{i % 100:03d},2024-01-01\n")
    temp_file = f.name

processor = TransactionFileProcessor()

# Pipeline: Read → Filter → Calculate Fees → Batch
start_time = time.time()
total_processed = 0
total_amount = 0

try:
    # Chain generators (no intermediate storage!)
    transactions = processor.read_transactions(temp_file)
    high_value = processor.filter_high_value(transactions, threshold=5000)
    with_fees = processor.calculate_fees(high_value)
    batches = processor.batch_transactions(with_fees, batch_size=100)
    
    # Process batches
    for batch_num, batch in enumerate(batches, 1):
        # Simulate batch database insert
        total_processed += len(batch)
        total_amount += sum(txn['total'] for txn in batch)
        print(f"  Batch {batch_num}: {len(batch)} transactions")
    
    duration = time.time() - start_time
    print(f"\nProcessed {total_processed} high-value transactions in {duration:.2f}s")
    print(f"Total amount: ${total_amount:,.2f}")
    print(f"Memory efficient: Only {sys.getsizeof(batch)} bytes in memory at peak")

finally:
    os.unlink(temp_file)
```

---

## 10. Scenario: Memory Comparison - List vs Generator

```python
import sys
import tracemalloc

def process_with_list(num_transactions: int):
    """
    ❌ BAD: Load all transactions into memory
    """
    transactions = []
    for i in range(num_transactions):
        transactions.append({
            'id': f'TXN{i:07d}',
            'amount': i * 100,
            'account': f'ACC{i % 1000:04d}'
        })
    
    # Process all transactions
    total = sum(txn['amount'] for txn in transactions)
    return total

def process_with_generator(num_transactions: int):
    """
    ✅ GOOD: Generate transactions on-the-fly
    """
    def transaction_generator():
        for i in range(num_transactions):
            yield {
                'id': f'TXN{i:07d}',
                'amount': i * 100,
                'account': f'ACC{i % 1000:04d}'
            }
    
    # Process one at a time
    total = sum(txn['amount'] for txn in transaction_generator())
    return total

# Compare memory usage
print("\n=== Memory Comparison ===")
num_txns = 1_000_000

# Test with list
tracemalloc.start()
process_with_list(num_txns)
current, peak = tracemalloc.get_traced_memory()
list_memory = peak / 1024 / 1024
tracemalloc.stop()

print(f"List approach: {list_memory:.2f} MB peak memory")

# Test with generator
tracemalloc.start()
process_with_generator(num_txns)
current, peak = tracemalloc.get_traced_memory()
gen_memory = peak / 1024 / 1024
tracemalloc.stop()

print(f"Generator approach: {gen_memory:.2f} MB peak memory")
print(f"Memory saved: {list_memory - gen_memory:.2f} MB ({(1 - gen_memory/list_memory)*100:.1f}% reduction)")
```

---

## 11. Scenario: Infinite Data Streams

```python
import random
import time
from datetime import datetime

def transaction_stream():
    """
    Infinite generator: Real-time transaction stream
    Use case: Processing transactions from message queue
    """
    transaction_id = 1
    while True:
        yield {
            'id': f'TXN{transaction_id:08d}',
            'amount': random.uniform(10, 10000),
            'timestamp': datetime.now().isoformat(),
            'account': f'ACC{random.randint(1, 1000):04d}'
        }
        transaction_id += 1
        time.sleep(0.01)  # Simulate real-time stream

def fraud_detector(transactions: Iterator[Dict], 
                   max_amount: float = 5000) -> Iterator[Dict]:
    """
    Monitor infinite stream for suspicious transactions
    """
    for txn in transactions:
        if txn['amount'] > max_amount:
            yield {
                **txn,
                'alert': 'HIGH_VALUE_TRANSACTION',
                'risk_level': 'HIGH'
            }

# Process real-time stream
print("\n=== Real-Time Fraud Detection ===")
print("Monitoring transaction stream (Ctrl+C to stop)...")

stream = transaction_stream()
alerts = fraud_detector(stream, max_amount=5000)

# Process first 10 suspicious transactions
import itertools
for alert in itertools.islice(alerts, 10):
    print(f"🚨 ALERT: {alert['id']} - ${alert['amount']:.2f} - {alert['alert']}")
```

---

## 12. Generator with .send() - Two-Way Communication

```python
def running_balance_calculator():
    """
    Generator that maintains running balance
    Receives transactions, yields current balance
    """
    balance = 0.0
    
    while True:
        # Receive transaction from sender
        transaction = yield balance
        
        if transaction is None:
            break
        
        # Update balance
        if transaction['type'] == 'DEPOSIT':
            balance += transaction['amount']
        elif transaction['type'] == 'WITHDRAWAL':
            balance -= transaction['amount']
        
        print(f"  {transaction['type']}: ${transaction['amount']:.2f} → Balance: ${balance:.2f}")

print("\n=== Running Balance with .send() ===")

# Create calculator
calc = running_balance_calculator()
next(calc)  # Prime the generator

# Send transactions
transactions = [
    {'type': 'DEPOSIT', 'amount': 1000.00},
    {'type': 'WITHDRAWAL', 'amount': 250.00},
    {'type': 'DEPOSIT', 'amount': 500.00},
    {'type': 'WITHDRAWAL', 'amount': 100.00},
]

for txn in transactions:
    balance = calc.send(txn)

final_balance = calc.send(None)  # Stop
print(f"\nFinal balance: ${final_balance:.2f}")
```

---

## 13. When to Use Generators vs Lists

```python
import time

# Scenario 1: Need to iterate multiple times
print("\n=== Scenario 1: Multiple Iterations ===")

# ❌ Generator - can only iterate once!
def get_accounts_gen():
    for i in range(5):
        yield f'ACC{i:03d}'

accounts_gen = get_accounts_gen()
print("First iteration:", list(accounts_gen))
print("Second iteration:", list(accounts_gen))  # Empty!

# ✅ List - can iterate multiple times
accounts_list = [f'ACC{i:03d}' for i in range(5)]
print("First iteration:", accounts_list)
print("Second iteration:", accounts_list)  # Works!

# Scenario 2: Need to know length or access by index
print("\n=== Scenario 2: Length and Index Access ===")

# ❌ Generator - can't get length or access by index
txns_gen = (i for i in range(100))
# print(len(txns_gen))  # Error!
# print(txns_gen[50])   # Error!

# ✅ List - supports length and indexing
txns_list = list(range(100))
print(f"Length: {len(txns_list)}")
print(f"Index 50: {txns_list[50]}")

# Scenario 3: Large dataset, single iteration
print("\n=== Scenario 3: Large Dataset Processing ===")

# ❌ List - loads everything into memory
start = time.time()
large_list = [i * 2 for i in range(10_000_000)]
total = sum(large_list)
list_time = time.time() - start
list_size = sys.getsizeof(large_list) / 1024 / 1024
print(f"List: {list_time:.2f}s, {list_size:.2f} MB")

# ✅ Generator - memory efficient
start = time.time()
large_gen = (i * 2 for i in range(10_000_000))
total = sum(large_gen)
gen_time = time.time() - start
gen_size = sys.getsizeof(large_gen) / 1024
print(f"Generator: {gen_time:.2f}s, {gen_size:.2f} KB")
```

---

## Key Takeaways:

### Generators vs Iterators:

| Feature | Iterator | Generator |
|---------|----------|----------|
| **Implementation** | Class with `__iter__` and `__next__` | Function with `yield` |
| **Code complexity** | ~20 lines | ~5 lines |
| **State management** | Manual (instance variables) | Automatic |
| **Memory** | Depends on implementation | Always lazy |
| **Best for** | Complex iteration logic | Simple, sequential iteration |

### When to Use Generators:

✅ **ALWAYS Use for:**
- **Large files** (CSV, logs, archives)
- **Database result sets** (millions of rows)
- **Real-time streams** (transactions, events, logs)
- **Expensive computations** (calculate on-demand)
- **Pipeline operations** (filter → transform → aggregate)
- **Infinite sequences** (counters, timestamps)
- **Memory-constrained environments**

❌ **DON'T Use for:**
- **Small datasets** (< 1000 items)
- **Multiple iterations needed** (can't rewind)
- **Random access required** (need indexing)
- **Need to know length** (len() doesn't work)
- **Sorting/reversing** (need full dataset)

### Generator Patterns:

```python
# 1. Simple generation
def numbers():
    for i in range(10):
        yield i

# 2. Filtering pipeline
def filter_chain(items):
    for item in items:
        if item > 100:  # Filter
            yield item * 2  # Transform

# 3. Batch processing
def batcher(items, size):
    batch = []
    for item in items:
        batch.append(item)
        if len(batch) >= size:
            yield batch
            batch = []
    if batch:
        yield batch

# 4. Infinite streams
def infinite_counter(start=0):
    n = start
    while True:
        yield n
        n += 1

# 5. Two-way communication
def accumulator():
    total = 0
    while True:
        value = yield total
        if value is None:
            break
        total += value
```

### Performance Numbers:

```python
# Processing 10M transactions:
# List: ~800 MB memory, 2.5s
# Generator: ~0.1 MB memory, 2.3s
# Winner: Generator (400x less memory, similar speed)

# Processing 100 transactions (multiple iterations):
# List: ~0.01 MB, instant access
# Generator: Need to regenerate each time
# Winner: List (convenience worth the memory)
```

### Banking Applications:

**Perfect for generators:**
- End-of-day transaction processing (10M+ transactions)
- Statement generation (millions of customers)
- Audit log analysis (GB of logs)
- Real-time fraud detection (infinite stream)
- Batch payment processing (millions of payments)
- Data migration/archival (TB of data)

**Use lists instead:**
- Current session transactions (< 1000)
- Active account list (hundreds)
- Recently viewed items (< 100)
- Search results (paginated, < 100 per page)
- UI data (needs sorting, filtering, multiple access)
- Cache data (frequently accessed)

---

## 14. Scenario: Custom Iterator with State Management

```python
class AccountStatementIterator:
    """
    Custom iterator for generating monthly account statements
    Maintains complex state: balance tracking, fee calculation, interest
    
    Use case: Generate statements for 10M accounts without loading all data
    """
    
    def __init__(self, account_id: str, start_date: str, end_date: str):
        self.account_id = account_id
        self.start_date = start_date
        self.end_date = end_date
        self.current_balance = self._get_starting_balance()
        self.current_index = 0
        self.transactions = self._fetch_transactions()
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current_index >= len(self.transactions):
            raise StopIteration
        
        txn = self.transactions[self.current_index]
        self.current_index += 1
        
        # Update running balance
        if txn['type'] == 'DEBIT':
            self.current_balance -= txn['amount']
        else:
            self.current_balance += txn['amount']
        
        # Add balance to transaction
        return {
            **txn,
            'running_balance': self.current_balance
        }
    
    def _get_starting_balance(self):
        # Simulate database query
        return 5000.00
    
    def _fetch_transactions(self):
        # Simulate fetching transactions from database
        return [
            {'date': '2024-01-01', 'type': 'CREDIT', 'amount': 1000, 'description': 'Salary'},
            {'date': '2024-01-05', 'type': 'DEBIT', 'amount': 200, 'description': 'Groceries'},
            {'date': '2024-01-10', 'type': 'DEBIT', 'amount': 100, 'description': 'Utilities'},
            {'date': '2024-01-15', 'type': 'CREDIT', 'amount': 500, 'description': 'Freelance'},
            {'date': '2024-01-20', 'type': 'DEBIT', 'amount': 50, 'description': 'Subscription'},
        ]

# Usage
print("\n=== Custom Iterator with State ===")
print(f"Account Statement for ACC001")
print(f"{'Date':<12} {'Type':<10} {'Amount':>10} {'Balance':>12} {'Description'}")
print("-" * 60)

statement = AccountStatementIterator('ACC001', '2024-01-01', '2024-01-31')
for txn in statement:
    print(f"{txn['date']:<12} {txn['type']:<10} ${txn['amount']:>9.2f} ${txn['running_balance']:>11.2f} {txn['description']}")
```

---

## 15. Scenario: Generator Chain for Data Pipeline

```python
from typing import Iterator, Dict
import time
import random

def fetch_transactions_from_db(batch_size: int = 1000) -> Iterator[Dict]:
    """
    Step 1: Fetch transactions from database in batches
    Simulates reading from PostgreSQL with cursor
    """
    print("📊 Fetching transactions from database...")
    for batch_num in range(5):  # Simulate 5 batches
        batch = []
        for i in range(batch_size):
            batch.append({
                'id': f'TXN{batch_num * batch_size + i:07d}',
                'amount': random.uniform(10, 10000),
                'currency': 'USD',
                'status': random.choice(['PENDING', 'COMPLETED', 'FAILED'])
            })
        
        # Yield one transaction at a time
        for txn in batch:
            yield txn
        
        print(f"  Batch {batch_num + 1}/5 fetched ({batch_size} transactions)")
        time.sleep(0.1)  # Simulate DB query time

def validate_transactions(transactions: Iterator[Dict]) -> Iterator[Dict]:
    """
    Step 2: Validate transactions
    Filter out invalid transactions
    """
    print("\n✅ Validating transactions...")
    valid_count = 0
    invalid_count = 0
    
    for txn in transactions:
        # Validation rules
        if txn['amount'] <= 0:
            invalid_count += 1
            continue
        
        if txn['status'] == 'FAILED':
            invalid_count += 1
            continue
        
        valid_count += 1
        yield txn
    
    print(f"  Valid: {valid_count}, Invalid: {invalid_count}")

def convert_currency(transactions: Iterator[Dict], 
                     target_currency: str = 'AED') -> Iterator[Dict]:
    """
    Step 3: Convert currency
    USD to AED (1 USD = 3.67 AED)
    """
    print(f"\n💱 Converting to {target_currency}...")
    conversion_rate = 3.67
    converted_count = 0
    
    for txn in transactions:
        if txn['currency'] != target_currency:
            txn['original_amount'] = txn['amount']
            txn['original_currency'] = txn['currency']
            txn['amount'] = txn['amount'] * conversion_rate
            txn['currency'] = target_currency
            converted_count += 1
        
        yield txn
    
    print(f"  Converted {converted_count} transactions")

def calculate_fees(transactions: Iterator[Dict]) -> Iterator[Dict]:
    """
    Step 4: Calculate transaction fees
    1% for amounts < 1000, 0.5% for amounts >= 1000
    """
    print("\n💰 Calculating fees...")
    total_fees = 0
    
    for txn in transactions:
        if txn['amount'] < 1000:
            fee = txn['amount'] * 0.01  # 1%
        else:
            fee = txn['amount'] * 0.005  # 0.5%
        
        txn['fee'] = fee
        txn['net_amount'] = txn['amount'] - fee
        total_fees += fee
        
        yield txn
    
    print(f"  Total fees: ${total_fees:.2f}")

def aggregate_by_status(transactions: Iterator[Dict]) -> Dict:
    """
    Step 5: Aggregate results
    Final step consumes the generator
    """
    print("\n📈 Aggregating results...")
    
    stats = {
        'PENDING': {'count': 0, 'total': 0},
        'COMPLETED': {'count': 0, 'total': 0}
    }
    
    for txn in transactions:
        status = txn['status']
        stats[status]['count'] += 1
        stats[status]['total'] += txn['net_amount']
    
    return stats

# Execute the pipeline
print("\n=== Transaction Processing Pipeline ===")
print("Processing 5000 transactions through 5-stage pipeline...\n")

start_time = time.time()

# Chain generators (no intermediate storage!)
transactions = fetch_transactions_from_db(batch_size=1000)
valid_txns = validate_transactions(transactions)
converted_txns = convert_currency(valid_txns, target_currency='AED')
with_fees = calculate_fees(converted_txns)
results = aggregate_by_status(with_fees)

duration = time.time() - start_time

print(f"\n🎯 Results:")
print(f"  PENDING: {results['PENDING']['count']} transactions, Total: AED {results['PENDING']['total']:,.2f}")
print(f"  COMPLETED: {results['COMPLETED']['count']} transactions, Total: AED {results['COMPLETED']['total']:,.2f}")
print(f"\n⚡ Pipeline completed in {duration:.2f}s")
print(f"💾 Memory efficient: Only 1 transaction in memory at a time")
```

---

## 16. Scenario: Generator vs List - Performance Benchmark

```python
import sys
import tracemalloc
import time
from typing import List, Iterator

def generate_transactions_list(count: int) -> List[Dict]:
    """Generate transactions as list (all in memory)"""
    return [
        {
            'id': f'TXN{i:08d}',
            'amount': i * 10.5,
            'account': f'ACC{i % 10000:05d}',
            'timestamp': f'2024-01-{(i % 28) + 1:02d}'
        }
        for i in range(count)
    ]

def generate_transactions_gen(count: int) -> Iterator[Dict]:
    """Generate transactions as generator (lazy)"""
    for i in range(count):
        yield {
            'id': f'TXN{i:08d}',
            'amount': i * 10.5,
            'account': f'ACC{i % 10000:05d}',
            'timestamp': f'2024-01-{(i % 28) + 1:02d}'
        }

def process_transactions(transactions) -> float:
    """Process transactions and return total"""
    return sum(txn['amount'] for txn in transactions)

print("\n=== Performance Benchmark: List vs Generator ===")

test_sizes = [10_000, 100_000, 1_000_000]

for size in test_sizes:
    print(f"\n📊 Testing with {size:,} transactions:")
    
    # Test List
    tracemalloc.start()
    start = time.time()
    txns_list = generate_transactions_list(size)
    total = process_transactions(txns_list)
    list_time = time.time() - start
    list_current, list_peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    
    # Test Generator
    tracemalloc.start()
    start = time.time()
    txns_gen = generate_transactions_gen(size)
    total = process_transactions(txns_gen)
    gen_time = time.time() - start
    gen_current, gen_peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    
    # Results
    list_mb = list_peak / 1024 / 1024
    gen_mb = gen_peak / 1024 / 1024
    memory_saving = ((list_mb - gen_mb) / list_mb) * 100
    time_diff = ((list_time - gen_time) / list_time) * 100
    
    print(f"  List:      {list_time:.3f}s, {list_mb:.2f} MB")
    print(f"  Generator: {gen_time:.3f}s, {gen_mb:.2f} MB")
    print(f"  💾 Memory saved: {memory_saving:.1f}%")
    print(f"  ⚡ Time difference: {abs(time_diff):.1f}% {'faster' if time_diff > 0 else 'slower'}")
```

---

## 17. When NOT to Use Generators - Common Mistakes

```python
print("\n=== When NOT to Use Generators ===")

# ❌ MISTAKE 1: Using generator when you need multiple iterations
print("\n❌ MISTAKE 1: Multiple iterations")

def get_account_balances():
    accounts = ['ACC001', 'ACC002', 'ACC003']
    for acc in accounts:
        yield {'account': acc, 'balance': 1000.00}

balances = get_account_balances()
print("First iteration:", list(balances))  # Works
print("Second iteration:", list(balances))  # Empty! Generator exhausted

# ✅ FIX: Use list for small datasets that need multiple access
balances_list = [{'account': acc, 'balance': 1000.00} 
                 for acc in ['ACC001', 'ACC002', 'ACC003']]
print("First iteration:", balances_list)
print("Second iteration:", balances_list)  # Still works!

# ❌ MISTAKE 2: Using generator when you need indexing
print("\n❌ MISTAKE 2: Need index access")

txns_gen = (i for i in range(100))
# print(txns_gen[50])  # TypeError: 'generator' object is not subscriptable
# print(len(txns_gen))  # TypeError: object of type 'generator' has no len()

# ✅ FIX: Convert to list or use list from start
txns_list = list(range(100))
print(f"Item at index 50: {txns_list[50]}")
print(f"Length: {len(txns_list)}")

# ❌ MISTAKE 3: Using generator for data that needs sorting
print("\n❌ MISTAKE 3: Need to sort data")

def unsorted_transactions():
    for i in [100, 50, 200, 25, 150]:
        yield {'amount': i}

txns = unsorted_transactions()
# sorted_txns = sorted(txns, key=lambda x: x['amount'])  # Works but defeats purpose!
# Generator consumed entirely to sort anyway

# ✅ FIX: Use list when you need sorting
txns_list = [{'amount': i} for i in [100, 50, 200, 25, 150]]
sorted_txns = sorted(txns_list, key=lambda x: x['amount'])
print("Sorted:", [t['amount'] for t in sorted_txns])

# ❌ MISTAKE 4: Premature optimization
print("\n❌ MISTAKE 4: Over-engineering for small data")

# Overkill for 10 items
def small_dataset_gen():
    for i in range(10):
        yield i

# ✅ FIX: Just use list for small datasets
small_dataset = list(range(10))
print(f"Small dataset: {small_dataset}")

# ✅ GOOD: Use generator for large datasets
def large_dataset_gen():
    for i in range(10_000_000):
        yield i

# Only loads one at a time
total = sum(large_dataset_gen())
print(f"Large dataset sum: {total}")
```
