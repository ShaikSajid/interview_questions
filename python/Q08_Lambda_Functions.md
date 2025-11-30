# Q08: Lambda Functions

## Question: What are lambda functions and when should you use them?

---

## Answer:

**Lambda** = Anonymous function defined inline using `lambda` keyword

```python
# Regular function
def add(x, y):
    return x + y

# Lambda equivalent
add_lambda = lambda x, y: x + y

print(add(5, 3))  # 8
print(add_lambda(5, 3))  # 8
```

---

## 1. Basic Lambda Syntax

```python
# Syntax: lambda arguments: expression

# Single argument
square = lambda x: x ** 2
print(square(5))  # 25

# Multiple arguments
multiply = lambda x, y: x * y
print(multiply(4, 5))  # 20

# No arguments
get_pi = lambda: 3.14159
print(get_pi())  # 3.14159
```

---

## 2. Banking Example: Sorting Transactions

```python
transactions = [
    {'id': 'TXN001', 'amount': 500, 'date': '2024-01-15'},
    {'id': 'TXN002', 'amount': 200, 'date': '2024-01-14'},
    {'id': 'TXN003', 'amount': 1500, 'date': '2024-01-16'},
]

# Sort by amount (using lambda)
sorted_by_amount = sorted(transactions, key=lambda txn: txn['amount'])
print([t['amount'] for t in sorted_by_amount])  # [200, 500, 1500]

# Sort by date (descending)
sorted_by_date = sorted(transactions, key=lambda txn: txn['date'], reverse=True)
print([t['date'] for t in sorted_by_date])  # ['2024-01-16', ...]

# Sort by multiple criteria
sorted_multi = sorted(
    transactions, 
    key=lambda txn: (txn['date'], txn['amount'])
)
```

---

## 3. Lambda with filter()

```python
transactions = [
    {'id': 'TXN001', 'amount': 100},
    {'id': 'TXN002', 'amount': 5000},
    {'id': 'TXN003', 'amount': 15000},
]

# Filter high-value transactions
high_value = list(filter(lambda txn: txn['amount'] > 1000, transactions))
print(high_value)  # TXN002 and TXN003

# ✅ More Pythonic with comprehension:
high_value = [txn for txn in transactions if txn['amount'] > 1000]
```

---

## 4. Lambda with map()

```python
amounts = [100, 200, 300, 400]

# Add fee to each amount
with_fee = list(map(lambda amount: amount + 10, amounts))
print(with_fee)  # [110, 210, 310, 410]

# ✅ More Pythonic:
with_fee = [amount + 10 for amount in amounts]
```

---

## 5. Lambda in Banking Calculations

```python
# Fee calculation functions
calculate_fees = {
    'domestic': lambda amount: amount * 0.01,  # 1%
    'international': lambda amount: amount * 0.025,  # 2.5%
    'premium': lambda amount: 0  # No fee
}

# Usage
amount = 1000
txn_type = 'international'
fee = calculate_fees[txn_type](amount)
print(f"Fee: ${fee}")  # $25.0
```

---

## 6. When to Use Lambda vs Regular Function

```python
# ✅ GOOD: Simple inline operations
accounts = sorted(accounts, key=lambda acc: acc['balance'])

# ❌ BAD: Complex logic (use regular function)
# Hard to read:
result = lambda x: x * 2 if x > 100 else x / 2 if x > 50 else x

# ✅ BETTER: Regular function
def calculate_adjusted_amount(x):
    if x > 100:
        return x * 2
    elif x > 50:
        return x / 2
    return x
```

---

## 7. Lambda Limitations

```python
# ❌ Cannot use statements (only expressions)
# lambda x: print(x)  # SyntaxError!

# ❌ Cannot use multiple statements
# lambda x: y = x + 1; return y  # SyntaxError!

# ✅ Use regular function for complex logic
def process_transaction(txn):
    validated = validate(txn)
    processed = apply_fees(validated)
    return save_to_db(processed)
```

---

## Best Practices:

### ✅ DO:
- Use lambda for simple, one-line operations
- Use with sort(), filter(), map()
- Keep lambda expressions readable
- Prefer comprehensions for simple transformations

### ❌ DON'T:
- Use lambda for complex logic
- Assign lambda to a variable (use `def` instead)
- Make lambda hard to understand
- Use when regular function is clearer

---

## Key Takeaways:

- **Lambda** = anonymous inline function
- **Best for**: simple operations, sorting, filtering
- **Limitations**: single expression only
- **Banking use**: sorting transactions, fee calculations
- **Readability matters**: use regular functions for complex logic
