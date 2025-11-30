# Q07: *args and **kwargs

## Question: What are *args and **kwargs in Python?

---

## Answer:

**`*args`**: Accepts variable number of **positional** arguments (tuple)  
**`**kwargs`**: Accepts variable number of **keyword** arguments (dict)

---

## 1. Basic *args Example

```python
def calculate_total(*amounts):
    """Sum any number of transaction amounts"""
    return sum(amounts)

# Works with any number of arguments
print(calculate_total(100))  # 100
print(calculate_total(100, 200, 300))  # 600
print(calculate_total(10, 20, 30, 40, 50))  # 150

# *args is a tuple
def show_args(*args):
    print(f"Type: {type(args)}")  # <class 'tuple'>
    print(f"Values: {args}")

show_args(1, 2, 3)  # (1, 2, 3)
```

---

## 2. Basic **kwargs Example

```python
def create_transaction(**details):
    """Create transaction with any number of fields"""
    print(f"Type: {type(details)}")  # <class 'dict'>
    for key, value in details.items():
        print(f"{key}: {value}")

# Works with any keyword arguments
create_transaction(id='TXN001', amount=100, status='completed')
create_transaction(id='TXN002', amount=500)  # Fewer arguments OK
```

---

## 3. Banking Example: Flexible Transaction Processing

```python
def process_transaction(txn_id, amount, *fees, **metadata):
    """
    txn_id: required positional
    amount: required positional
    *fees: optional additional fees
    **metadata: optional transaction metadata
    """
    total = amount + sum(fees)
    
    print(f"Transaction: {txn_id}")
    print(f"Base amount: ${amount}")
    print(f"Fees: ${sum(fees)}")
    print(f"Total: ${total}")
    print(f"Metadata: {metadata}")

# Usage
process_transaction(
    'TXN001', 
    1000,  # amount
    10, 5, 2,  # fees (*args)
    category='transfer',  # metadata (**kwargs)
    urgent=True,
    destination='ACC002'
)
```

---

## 4. Unpacking with * and **

```python
# Unpacking lists/tuples with *
amounts = [100, 200, 300]
print(calculate_total(*amounts))  # Unpacks to: 100, 200, 300

# Unpacking dictionaries with **
txn_details = {
    'id': 'TXN001',
    'amount': 500,
    'category': 'payment'
}
create_transaction(**txn_details)  # Unpacks to keyword arguments
```

---

## 5. Combining Regular Args with *args and **kwargs

```python
def transfer_funds(from_account, to_account, *amounts, **options):
    """
    Order must be: regular args, *args, **kwargs
    """
    total = sum(amounts)
    
    print(f"From: {from_account}")
    print(f"To: {to_account}")
    print(f"Amounts: {amounts}")
    print(f"Total: ${total}")
    print(f"Options: {options}")

transfer_funds(
    'ACC001',  # from_account
    'ACC002',  # to_account
    100, 200, 300,  # *amounts
    urgent=True,  # **options
    notify=True
)
```

---

## 6. Real-World Banking Example

```python
class TransactionProcessor:
    def execute(self, txn_type, amount, *accounts, **flags):
        """Flexible transaction execution"""
        result = {
            'type': txn_type,
            'amount': amount,
            'accounts': accounts,
            'flags': flags,
            'status': 'pending'
        }
        
        # Process based on flags
        if flags.get('urgent'):
            result['priority'] = 'high'
        if flags.get('notify'):
            self.send_notification(accounts, amount)
        
        return result
    
    def send_notification(self, accounts, amount):
        print(f"Notification sent to {len(accounts)} accounts: ${amount}")

# Usage
processor = TransactionProcessor()

result = processor.execute(
    'transfer',  # txn_type
    1000,  # amount
    'ACC001', 'ACC002', 'ACC003',  # *accounts
    urgent=True,  # **flags
    notify=True,
    audit=True
)
print(result)
```

---

## 7. Function Signature Order (Important!)

```python
def func(pos1, pos2, *args, kwonly, **kwargs):
    """
    Correct order:
    1. Regular positional arguments
    2. *args (variable positional)
    3. Keyword-only arguments
    4. **kwargs (variable keyword)
    """
    pass

# ❌ This won't work:
# def bad_func(*args, pos1, **kwargs):  # Error!
#     pass
```

---

## Best Practices:

### ✅ DO:
- Use *args for variable positional arguments
- Use **kwargs for optional configuration
- Combine with regular args for flexibility
- Unpack with * and ** when calling functions

### ❌ DON'T:
- Overuse - makes functions hard to understand
- Forget the correct parameter order
- Use when fixed parameters are more appropriate
- Ignore type hints for *args and **kwargs

---

## 8. Real-World Scenario: Flexible Banking API

```python
from typing import Any, Dict, List
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class BankingAPI:
    """
    Production banking API with flexible parameter handling
    Supports various transaction types with different parameter requirements
    """
    
    def process_transaction(self, 
                           transaction_type: str,
                           amount: float,
                           *accounts: str,
                           **options: Any) -> Dict:
        """
        Flexible transaction processing
        
        Args:
            transaction_type: Type of transaction (TRANSFER, DEPOSIT, WITHDRAW)
            amount: Transaction amount
            *accounts: Variable number of account IDs
            **options: Additional options (urgent, notify, audit, etc.)
        
        Returns:
            Transaction result dictionary
        """
        result = {
            'transaction_type': transaction_type,
            'amount': amount,
            'accounts': accounts,
            'options': options,
            'timestamp': time.time(),
            'status': 'PENDING'
        }
        
        logger.info(f"Processing {transaction_type} of ${amount}")
        logger.info(f"  Accounts: {accounts}")
        logger.info(f"  Options: {options}")
        
        # Handle different transaction types
        if transaction_type == 'TRANSFER':
            if len(accounts) < 2:
                raise ValueError("TRANSFER requires at least 2 accounts")
            result['from_account'] = accounts[0]
            result['to_account'] = accounts[1]
        
        elif transaction_type == 'DEPOSIT':
            if len(accounts) != 1:
                raise ValueError("DEPOSIT requires exactly 1 account")
            result['to_account'] = accounts[0]
        
        elif transaction_type == 'WITHDRAW':
            if len(accounts) != 1:
                raise ValueError("WITHDRAW requires exactly 1 account")
            result['from_account'] = accounts[0]
        
        # Process options
        if options.get('urgent'):
            result['priority'] = 'HIGH'
            logger.info("  ⚡ Urgent transaction - high priority")
        
        if options.get('notify'):
            self._send_notification(accounts, amount, transaction_type)
        
        if options.get('audit'):
            self._log_audit(result)
        
        if options.get('retry_count'):
            result['retry_count'] = options['retry_count']
        
        result['status'] = 'COMPLETED'
        return result
    
    def _send_notification(self, accounts, amount, txn_type):
        logger.info(f"  📧 Notification sent to {len(accounts)} account(s)")
    
    def _log_audit(self, result):
        logger.info(f"  📝 Audit log created for transaction")

# Usage Examples
print("\n=== Flexible Banking API ===")

api = BankingAPI()

# Example 1: Simple deposit
print("\n1️⃣ Simple Deposit:")
result = api.process_transaction(
    'DEPOSIT',
    1000.00,
    'ACC001'
)
print(f"   Result: {result['status']} - ${result['amount']} to {result['to_account']}")

# Example 2: Transfer with options
print("\n2️⃣ Urgent Transfer with Notification:")
result = api.process_transaction(
    'TRANSFER',
    5000.00,
    'ACC001', 'ACC002',  # Multiple accounts via *args
    urgent=True,         # Options via **kwargs
    notify=True,
    audit=True,
    notes="Salary payment"
)
print(f"   Result: {result['status']} - {result['from_account']} → {result['to_account']}")
print(f"   Priority: {result.get('priority', 'NORMAL')}")

# Example 3: Multi-account transfer
print("\n3️⃣ Multi-Account Transfer:")
result = api.process_transaction(
    'TRANSFER',
    100.00,
    'ACC001', 'ACC002', 'ACC003',  # Can handle any number
    split_evenly=True,
    notify=True
)
print(f"   Accounts involved: {len(result['accounts'])}")
```

---

## 9. Scenario: Function Decorator with Flexible Arguments

```python
import functools
import time
from typing import Callable

def log_transaction(*log_args, level='INFO', include_result=True, **log_kwargs):
    """
    Flexible decorator that accepts any number of arguments
    Can configure logging behavior with keyword arguments
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # Log function call
            func_name = func.__name__
            logger.log(
                getattr(logging, level),
                f"Calling {func_name} with args={args}, kwargs={kwargs}"
            )
            
            # Log decorator arguments
            if log_args:
                logger.info(f"  Decorator args: {log_args}")
            if log_kwargs:
                logger.info(f"  Decorator kwargs: {log_kwargs}")
            
            # Execute function
            start = time.time()
            try:
                result = func(*args, **kwargs)
                duration = time.time() - start
                
                if include_result:
                    logger.info(f"  Result: {result}")
                logger.info(f"  Duration: {duration:.3f}s")
                
                return result
            except Exception as e:
                logger.error(f"  Error: {e}")
                raise
        
        return wrapper
    return decorator

# Usage
print("\n=== Flexible Decorator ===")

@log_transaction('AUDIT', 'CRITICAL', level='WARNING', include_result=True)
def transfer_funds(from_account: str, to_account: str, amount: float):
    """Transfer funds between accounts"""
    time.sleep(0.1)  # Simulate processing
    return {'status': 'SUCCESS', 'amount': amount}

print("\n1️⃣ Decorated Function:")
result = transfer_funds('ACC001', 'ACC002', 1000.00)
print(f"   Final result: {result}")

@log_transaction(level='DEBUG', include_result=False)
def calculate_interest(balance: float, rate: float = 0.05):
    """Calculate interest"""
    return balance * rate

print("\n2️⃣ Another Decorated Function:")
interest = calculate_interest(10000.00)
print(f"   Interest: ${interest:.2f}")
```

---

## 10. Scenario: Function Factory with *args/**kwargs

```python
from typing import Callable

def create_fee_calculator(*fixed_fees: float, **percentage_fees: float) -> Callable:
    """
    Factory function that creates fee calculators
    Accepts fixed fees as positional args and percentage fees as kwargs
    
    Example:
        calc = create_fee_calculator(5.00, 2.50, processing=0.01, network=0.005)
    """
    def calculator(transaction_amount: float) -> Dict:
        """Calculate total fees for transaction"""
        # Calculate fixed fees
        total_fixed = sum(fixed_fees)
        
        # Calculate percentage fees
        total_percentage = sum(
            transaction_amount * rate
            for rate in percentage_fees.values()
        )
        
        total_fees = total_fixed + total_percentage
        
        return {
            'amount': transaction_amount,
            'fixed_fees': total_fixed,
            'percentage_fees': total_percentage,
            'total_fees': total_fees,
            'net_amount': transaction_amount + total_fees,
            'breakdown': {
                'fixed': list(fixed_fees),
                'percentage': percentage_fees
            }
        }
    
    return calculator

print("\n=== Function Factory ===")

# Create domestic calculator: $5 flat fee + 1% processing
domestic_calc = create_fee_calculator(5.00, processing=0.01)

print("\n1️⃣ Domestic Transaction ($1000):")
result = domestic_calc(1000.00)
print(f"   Amount: ${result['amount']:.2f}")
print(f"   Fixed fees: ${result['fixed_fees']:.2f}")
print(f"   Percentage fees: ${result['percentage_fees']:.2f}")
print(f"   Total fees: ${result['total_fees']:.2f}")
print(f"   Net amount: ${result['net_amount']:.2f}")

# Create international calculator: $5 + $10 + 2.5% processing + 1% network
international_calc = create_fee_calculator(
    5.00, 10.00,                    # Fixed fees
    processing=0.025, network=0.01  # Percentage fees
)

print("\n2️⃣ International Transaction ($1000):")
result = international_calc(1000.00)
print(f"   Amount: ${result['amount']:.2f}")
print(f"   Fixed fees: ${result['fixed_fees']:.2f}")
print(f"   Percentage fees: ${result['percentage_fees']:.2f}")
print(f"   Total fees: ${result['total_fees']:.2f}")
print(f"   Net amount: ${result['net_amount']:.2f}")

# Create premium calculator: No fixed fees, only 0.5% processing
premium_calc = create_fee_calculator(processing=0.005)

print("\n3️⃣ Premium Transaction ($1000):")
result = premium_calc(1000.00)
print(f"   Amount: ${result['amount']:.2f}")
print(f"   Total fees: ${result['total_fees']:.2f}")
print(f"   Net amount: ${result['net_amount']:.2f}")
```

---

## 11. Scenario: Argument Unpacking in Practice

```python
from typing import Dict, List

class TransactionProcessor:
    """
    Process transactions with flexible argument patterns
    """
    
    def batch_transfer(self, *transfers: Dict):
        """
        Process multiple transfers at once
        Each transfer is a dictionary with transaction details
        """
        results = []
        
        print(f"\n💼 Processing {len(transfers)} transfers...")
        
        for i, transfer in enumerate(transfers, 1):
            print(f"\n  Transfer {i}:")
            # Unpack dictionary as keyword arguments
            result = self.single_transfer(**transfer)
            results.append(result)
        
        return results
    
    def single_transfer(self, from_account: str, to_account: str, 
                       amount: float, **metadata):
        """Process single transfer"""
        print(f"    From: {from_account}")
        print(f"    To: {to_account}")
        print(f"    Amount: ${amount}")
        
        if metadata:
            print(f"    Metadata: {metadata}")
        
        return {
            'from': from_account,
            'to': to_account,
            'amount': amount,
            'status': 'COMPLETED',
            **metadata  # Include metadata in result
        }

print("\n=== Argument Unpacking ===")

processor = TransactionProcessor()

# Define transfers as dictionaries
transfer1 = {
    'from_account': 'ACC001',
    'to_account': 'ACC002',
    'amount': 1000.00,
    'reference': 'INV-001',
    'urgent': True
}

transfer2 = {
    'from_account': 'ACC003',
    'to_account': 'ACC004',
    'amount': 500.00,
    'reference': 'INV-002'
}

transfer3 = {
    'from_account': 'ACC005',
    'to_account': 'ACC006',
    'amount': 2000.00,
    'reference': 'INV-003',
    'notify': True,
    'audit': True
}

# Process all transfers by unpacking list
results = processor.batch_transfer(transfer1, transfer2, transfer3)

print(f"\n✅ Completed {len(results)} transfers")

# Alternative: Unpack from list
print("\n=== Unpacking from List ===")
transfers_list = [transfer1, transfer2, transfer3]
results = processor.batch_transfer(*transfers_list)  # Unpack list
print(f"\n✅ Completed {len(results)} transfers from list")
```

---

## 12. Common Mistakes and Best Practices

```python
print("\n=== Common Mistakes ===")

# ❌ MISTAKE 1: Wrong parameter order
print("\n1️⃣ Mistake: Wrong parameter order")
try:
    # def wrong_order(*args, required, **kwargs):  # ❌ SyntaxError!
    #     pass
    
    # Correct order:
    def correct_order(required, *args, keyword_only=None, **kwargs):
        return required, args, keyword_only, kwargs
    
    result = correct_order(1, 2, 3, keyword_only=4, extra=5)
    print(f"   ✅ Correct: required={result[0]}, args={result[1]}, kwonly={result[2]}, kwargs={result[3]}")
except Exception as e:
    print(f"   ❌ Error: {e}")

# ❌ MISTAKE 2: Modifying *args or **kwargs
print("\n2️⃣ Mistake: Modifying mutable defaults")

def bad_function(*args, default_accounts=['ACC001']):
    """❌ BAD: Mutable default argument"""
    default_accounts.append('ACC002')  # Modifies shared list!
    return args, default_accounts

print("   First call:", bad_function())
print("   Second call:", bad_function())  # List keeps growing!

def good_function(*args, default_accounts=None):
    """✅ GOOD: Use None as default"""
    if default_accounts is None:
        default_accounts = ['ACC001']
    default_accounts = default_accounts.copy()  # Work with copy
    default_accounts.append('ACC002')
    return args, default_accounts

print("   First call:", good_function())
print("   Second call:", good_function())  # Fresh list each time

# ❌ MISTAKE 3: Overusing *args/**kwargs
print("\n3️⃣ Mistake: Overusing flexibility")

def too_flexible(*args, **kwargs):
    """❌ BAD: No idea what parameters are expected"""
    pass

def clear_interface(from_account: str, to_account: str, amount: float,
                    *additional_accounts: str, **options: Any):
    """✅ GOOD: Clear required parameters, flexible extras"""
    return {
        'from': from_account,
        'to': to_account,
        'amount': amount,
        'additional': additional_accounts,
        'options': options
    }

result = clear_interface('ACC001', 'ACC002', 1000.00, 'ACC003', urgent=True)
print(f"   ✅ Clear interface: {result}")

# ✅ BEST PRACTICE: Type hints
print("\n4️⃣ Best Practice: Use type hints")

def well_documented(required: str,
                    *optional_args: str,
                    flag: bool = False,
                    **metadata: Any) -> Dict[str, Any]:
    """
    ✅ BEST: Clear types for all parameters
    
    Args:
        required: Required string parameter
        *optional_args: Variable number of strings
        flag: Optional boolean flag
        **metadata: Additional metadata as key-value pairs
    
    Returns:
        Result dictionary
    """
    return {
        'required': required,
        'optional': optional_args,
        'flag': flag,
        'metadata': metadata
    }

result = well_documented('test', 'arg1', 'arg2', flag=True, key='value')
print(f"   ✅ Well documented: {result}")
```

---

## Key Takeaways:

### Parameter Order (CRITICAL):

```python
def function(
    required_positional,           # 1. Required positional
    optional_positional=default,   # 2. Optional positional
    *args,                        # 3. Variable positional
    keyword_only,                 # 4. Keyword-only (after *args)
    keyword_only_with_default=x,  # 5. Keyword-only with default
    **kwargs                      # 6. Variable keyword
):
    pass
```

### When to Use *args:

✅ **GOOD use cases:**
- **Variable number of similar items** (accounts, transaction IDs)
- **Wrapper functions** (decorators, middleware)
- **Combining lists** (concatenation, merging)
- **Mathematical operations** (sum, product)
- **Logging functions** (multiple messages)

❌ **BAD use cases:**
- **When parameters have different types**
- **When order matters but isn't obvious**
- **When you need named parameters**
- **When API clarity is important**

### When to Use **kwargs:

✅ **GOOD use cases:**
- **Optional configuration** (flags, settings)
- **Metadata** (tags, labels, annotations)
- **Extensibility** (future parameters)
- **Forwarding to other functions**
- **Dynamic APIs** (flexible endpoints)

❌ **BAD use cases:**
- **Required parameters**
- **When parameter names are unclear**
- **When type safety is critical**
- **When validation is needed**

### Production Patterns:

```python
# Pattern 1: Flexible API endpoint
def process_transaction(txn_type: str, amount: float, 
                       *accounts: str, **options: Any):
    """Handles any transaction type with flexible options"""
    pass

# Pattern 2: Configuration builder
def configure_service(**settings: Any):
    """Accept any configuration setting"""
    pass

# Pattern 3: Decorator pattern
def decorator(*dec_args, **dec_kwargs):
    def wrapper(func):
        @functools.wraps(func)
        def inner(*args, **kwargs):
            # Use all arguments
            return func(*args, **kwargs)
        return inner
    return wrapper

# Pattern 4: Function factory
def create_calculator(*fixed: float, **variable: float):
    def calculate(amount: float):
        return amount + sum(fixed) + sum(variable.values())
    return calculate
```

### Performance Impact:

- **Unpacking overhead**: ~0.1-0.5 microseconds per call
- **Negligible for I/O operations**
- **Matters in tight loops** (millions of iterations)
- **Dictionary unpacking** slightly slower than positional

### Banking Applications:

```python
# Transaction API (flexible parameters)
api.process('TRANSFER', 1000, 'ACC001', 'ACC002', urgent=True)

# Batch operations (variable accounts)
api.distribute(1000, 'ACC001', 'ACC002', 'ACC003', 'ACC004')

# Configuration (metadata)
api.configure(timeout=30, retry=3, log_level='DEBUG')

# Fee calculation (flexible fees)
calc = create_calculator(5.00, 10.00, processing=0.01, network=0.005)
```
