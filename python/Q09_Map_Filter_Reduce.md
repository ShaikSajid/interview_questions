# Q09: Map, Filter, Reduce

## Question: How do map, filter, and reduce work in Python?

---

## Answer:

**`map()`**: Apply function to each item  
**`filter()`**: Keep items matching condition  
**`reduce()`**: Combine items into single value

---

## 1. map() - Transform Each Item

```python
amounts = [100, 200, 300, 400]

# Add 10% fee to each amount
with_fee = list(map(lambda x: x * 1.1, amounts))
print(with_fee)  # [110.0, 220.0, 330.0, 440.0]

# ✅ Pythonic alternative (comprehension)
with_fee = [x * 1.1 for x in amounts]
```

---

## 2. filter() - Keep Matching Items

```python
transactions = [
    {'id': 'TXN001', 'amount': 50},
    {'id': 'TXN002', 'amount': 1500},
    {'id': 'TXN003', 'amount': 500},
    {'id': 'TXN004', 'amount': 5000},
]

# Filter high-value transactions (> 1000)
high_value = list(filter(lambda txn: txn['amount'] > 1000, transactions))
print([t['id'] for t in high_value])  # ['TXN002', 'TXN004']

# ✅ Pythonic alternative
high_value = [txn for txn in transactions if txn['amount'] > 1000]
```

---

## 3. reduce() - Combine All Items

```python
from functools import reduce

amounts = [100, 200, 300, 400]

# Sum all amounts
total = reduce(lambda acc, x: acc + x, amounts)
print(total)  # 1000

# ✅ Better: Use built-in sum()
total = sum(amounts)  # More readable!

# Multiply all amounts
product = reduce(lambda acc, x: acc * x, amounts)
print(product)  # 2,400,000,000
```

---

## 4. Banking Example: Transaction Processing Pipeline

```python
from functools import reduce

transactions = [
    {'id': 'TXN001', 'amount': 100, 'status': 'pending'},
    {'id': 'TXN002', 'amount': 5000, 'status': 'completed'},
    {'id': 'TXN003', 'amount': 15000, 'status': 'completed'},
    {'id': 'TXN004', 'amount': 500, 'status': 'failed'},
]

# Step 1: Filter completed transactions
completed = filter(lambda txn: txn['status'] == 'completed', transactions)

# Step 2: Map to amounts only
amounts = map(lambda txn: txn['amount'], completed)

# Step 3: Reduce to total
total = reduce(lambda acc, x: acc + x, amounts, 0)

print(f"Total completed: ${total}")  # $20,000

# ✅ More Pythonic (and readable):
total = sum(txn['amount'] for txn in transactions if txn['status'] == 'completed')
```

---

## 5. Real-World Banking Example

```python
class TransactionProcessor:
    def __init__(self, transactions):
        self.transactions = transactions
    
    def calculate_daily_totals(self):
        """Group by date and sum amounts"""
        from collections import defaultdict
        
        # Group transactions by date
        daily = defaultdict(list)
        for txn in self.transactions:
            daily[txn['date']].append(txn['amount'])
        
        # Calculate totals using reduce
        return {
            date: reduce(lambda acc, x: acc + x, amounts, 0)
            for date, amounts in daily.items()
        }
    
    def apply_fees(self, fee_percent):
        """Apply fee to all transactions"""
        return list(map(
            lambda txn: {**txn, 'final_amount': txn['amount'] * (1 + fee_percent)},
            self.transactions
        ))
    
    def filter_large_transactions(self, threshold):
        """Filter transactions above threshold"""
        return list(filter(
            lambda txn: txn['amount'] > threshold,
            self.transactions
        ))

# Usage
txns = [
    {'id': 'TXN001', 'amount': 100, 'date': '2024-01-15'},
    {'id': 'TXN002', 'amount': 5000, 'date': '2024-01-15'},
    {'id': 'TXN003', 'amount': 15000, 'date': '2024-01-16'},
]

processor = TransactionProcessor(txns)
print(processor.calculate_daily_totals())
print(processor.apply_fees(0.01))  # 1% fee
print(processor.filter_large_transactions(1000))
```

---

## 6. Chaining map/filter/reduce

```python
transactions = [
    {'amount': 100, 'type': 'debit'},
    {'amount': 200, 'type': 'credit'},
    {'amount': 300, 'type': 'debit'},
    {'amount': 400, 'type': 'credit'},
]

# Chain operations
from functools import reduce

# Filter debits → map to amounts → reduce to sum
total_debits = reduce(
    lambda acc, x: acc + x,
    map(
        lambda txn: txn['amount'],
        filter(lambda txn: txn['type'] == 'debit', transactions)
    ),
    0
)
print(f"Total debits: ${total_debits}")  # $400

# ✅ More readable with comprehension:
total_debits = sum(txn['amount'] for txn in transactions if txn['type'] == 'debit')
```

---

## 7. Performance Comparison

```python
import time

data = list(range(1_000_000))

# Using map/filter
start = time.time()
result = list(map(lambda x: x * 2, filter(lambda x: x % 2 == 0, data)))
print(f"map/filter: {time.time() - start:.2f}s")

# Using comprehension
start = time.time()
result = [x * 2 for x in data if x % 2 == 0]
print(f"comprehension: {time.time() - start:.2f}s")
# Comprehensions are often slightly faster
```

---

## Best Practices:

### ✅ DO:
- Prefer comprehensions for readability
- Use built-in sum() instead of reduce for addition
- Use map/filter for functional programming style
- Chain operations for data pipelines

### ❌ DON'T:
- Overuse reduce (hard to read)
- Use map/filter when comprehensions are clearer
- Forget to convert to list if needed
- Ignore performance with large datasets

---

## Key Takeaways:

- **map()** transforms each item
- **filter()** selects items matching condition
- **reduce()** combines items into single value
- **Comprehensions often more Pythonic** than map/filter
- **Banking use**: processing transaction batches efficiently
