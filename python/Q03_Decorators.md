# Q03: Decorators - Fundamentals

## Question: What are decorators in Python and how do you implement them?

---

## Answer:

Decorators are **functions that modify other functions or classes** without changing their code.

---

## 1. Basic Function Decorator

```python
def log_transaction(func):
    """Decorator to log transaction execution"""
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Completed {func.__name__}")
        return result
    return wrapper

@log_transaction
def process_payment(amount: float):
    print(f"Processing ${amount}")
    return f"Payment of ${amount} successful"

# Usage
process_payment(100.0)
# Output:
# Calling process_payment
# Processing $100.0
# Completed process_payment
```

---

## 2. Decorator with Arguments

```python
from functools import wraps
import time

def audit_log(transaction_type: str):
    """Audit decorator with transaction type"""
    def decorator(func):
        @wraps(func)  # Preserves function metadata
        def wrapper(*args, **kwargs):
            print(f"[AUDIT] {transaction_type} - {func.__name__}")
            start = time.time()
            result = func(*args, **kwargs)
            duration = time.time() - start
            print(f"[AUDIT] Completed in {duration:.2f}s")
            return result
        return wrapper
    return decorator

@audit_log("MONEY_TRANSFER")
def transfer_funds(from_acc: str, to_acc: str, amount: float):
    time.sleep(0.1)  # Simulate processing
    return f"Transferred ${amount} from {from_acc} to {to_acc}"

# Usage
transfer_funds("ACC001", "ACC002", 500.0)
```

---

## 3. Banking Example: Multiple Decorators

```python
def validate_amount(func):
    """Validate transaction amount"""
    def wrapper(amount, *args, **kwargs):
        if amount <= 0:
            raise ValueError("Amount must be positive")
        if amount > 1_000_000:
            raise ValueError("Amount exceeds limit")
        return func(amount, *args, **kwargs)
    return wrapper

def check_balance(func):
    """Check if balance is sufficient"""
    def wrapper(amount, account_balance, *args, **kwargs):
        if amount > account_balance:
            raise ValueError("Insufficient balance")
        return func(amount, account_balance, *args, **kwargs)
    return wrapper

@audit_log("WITHDRAWAL")
@validate_amount
@check_balance
def withdraw(amount: float, account_balance: float):
    new_balance = account_balance - amount
    return f"Withdrawal successful. New balance: ${new_balance}"

# Decorators execute from bottom to top!
# Order: check_balance → validate_amount → audit_log

try:
    result = withdraw(200.0, 1000.0)
    print(result)
except ValueError as e:
    print(f"Error: {e}")
```

---

## 4. Class Decorators

```python
def add_transaction_id(cls):
    """Add automatic transaction ID to class"""
    import uuid
    original_init = cls.__init__
    
    def new_init(self, *args, **kwargs):
        self.transaction_id = str(uuid.uuid4())
        original_init(self, *args, **kwargs)
    
    cls.__init__ = new_init
    return cls

@add_transaction_id
class Transaction:
    def __init__(self, amount: float, account: str):
        self.amount = amount
        self.account = account

# Usage
txn = Transaction(100.0, "ACC001")
print(f"ID: {txn.transaction_id}")
print(f"Amount: ${txn.amount}")
```

---

## 5. Property Decorator

```python
class BankAccount:
    def __init__(self, balance: float):
        self._balance = balance
    
    @property
    def balance(self) -> float:
        """Getter for balance"""
        return self._balance
    
    @balance.setter
    def balance(self, value: float):
        """Setter with validation"""
        if value < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = value

account = BankAccount(1000.0)
print(account.balance)  # Uses getter
account.balance = 1500.0  # Uses setter
# account.balance = -100  # Raises ValueError
```

---

## Best Practices:

### ✅ DO:
- Use `@wraps` to preserve function metadata
- Keep decorators simple and focused
- Use decorators for cross-cutting concerns (logging, auth, validation)
- Chain decorators for complex behavior

### ❌ DON'T:
- Make decorators too complex
- Forget that decorator order matters
- Use decorators for business logic
- Ignore performance impact

---

## 7. Real-World Scenario: Complete Banking Audit System

```python
import functools
import time
import json
from datetime import datetime
from typing import Any, Callable
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AuditLog:
    """Centralized audit logging for banking operations"""
    
    def __init__(self):
        self.logs = []
    
    def log(self, entry: dict):
        entry['timestamp'] = datetime.now().isoformat()
        self.logs.append(entry)
        logger.info(json.dumps(entry))
    
    def get_logs(self, transaction_type: str = None) -> list:
        if transaction_type:
            return [log for log in self.logs if log.get('type') == transaction_type]
        return self.logs

# Global audit log instance
audit_log = AuditLog()

def audit_transaction(transaction_type: str):
    """
    Decorator for comprehensive transaction auditing
    Logs: function name, arguments, return value, duration, errors
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            # Extract account info from arguments
            func_args = ', '.join([repr(arg) for arg in args])
            func_kwargs = ', '.join([f"{k}={repr(v)}" for k, v in kwargs.items()])
            all_args = ', '.join(filter(None, [func_args, func_kwargs]))
            
            # Start audit entry
            audit_entry = {
                'type': transaction_type,
                'function': func.__name__,
                'arguments': {'args': args, 'kwargs': kwargs},
                'status': 'started'
            }
            
            start_time = time.time()
            
            try:
                # Execute function
                result = func(*args, **kwargs)
                
                # Success audit
                duration = time.time() - start_time
                audit_entry.update({
                    'status': 'success',
                    'duration_ms': round(duration * 1000, 2),
                    'result': str(result)[:100]  # Truncate long results
                })
                
                audit_log.log(audit_entry)
                return result
                
            except Exception as e:
                # Error audit
                duration = time.time() - start_time
                audit_entry.update({
                    'status': 'error',
                    'duration_ms': round(duration * 1000, 2),
                    'error': str(e),
                    'error_type': type(e).__name__
                })
                
                audit_log.log(audit_entry)
                raise  # Re-raise the exception
        
        return wrapper
    return decorator

def validate_positive_amount(func: Callable) -> Callable:
    """Validate that amount is positive"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Find 'amount' in args or kwargs
        amount = None
        
        # Check kwargs first
        if 'amount' in kwargs:
            amount = kwargs['amount']
        # Check positional args (assume second argument is amount)
        elif len(args) >= 2:
            amount = args[1]
        
        if amount is not None and amount <= 0:
            raise ValueError(f"Amount must be positive, got {amount}")
        
        return func(*args, **kwargs)
    return wrapper

def check_account_exists(func: Callable) -> Callable:
    """Validate that account exists"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Extract account_id (assume first argument)
        account_id = args[0] if args else kwargs.get('account_id')
        
        # Simulated account check
        valid_accounts = ['ACC001', 'ACC002', 'ACC003']
        if account_id not in valid_accounts:
            raise ValueError(f"Account {account_id} does not exist")
        
        return func(*args, **kwargs)
    return wrapper

def require_authentication(func: Callable) -> Callable:
    """Check if user is authenticated"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # In real app, check session/token
        user_authenticated = kwargs.get('authenticated', True)
        
        if not user_authenticated:
            raise PermissionError("Authentication required")
        
        return func(*args, **kwargs)
    return wrapper

def rate_limit(max_calls: int, time_window: int):
    """
    Rate limiting decorator
    max_calls: Maximum number of calls
    time_window: Time window in seconds
    """
    calls = []
    
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.time()
            
            # Remove old calls outside time window
            calls[:] = [call_time for call_time in calls if now - call_time < time_window]
            
            if len(calls) >= max_calls:
                raise Exception(f"Rate limit exceeded: {max_calls} calls per {time_window}s")
            
            calls.append(now)
            return func(*args, **kwargs)
        
        return wrapper
    return decorator

# Banking Service with Multiple Decorators
class BankingService:
    """
    Complete banking service with decorated methods
    """
    
    @audit_transaction('WITHDRAWAL')
    @validate_positive_amount
    @check_account_exists
    @require_authentication
    @rate_limit(max_calls=5, time_window=60)  # Max 5 withdrawals per minute
    def withdraw(self, account_id: str, amount: float, **kwargs) -> dict:
        """Withdraw money from account"""
        time.sleep(0.1)  # Simulate processing
        return {
            'status': 'success',
            'account_id': account_id,
            'amount': amount,
            'new_balance': 1000 - amount  # Simulated
        }
    
    @audit_transaction('TRANSFER')
    @validate_positive_amount
    @check_account_exists
    @require_authentication
    def transfer(self, from_account: str, to_account: str, amount: float, **kwargs) -> dict:
        """Transfer money between accounts"""
        time.sleep(0.1)  # Simulate processing
        return {
            'status': 'success',
            'from': from_account,
            'to': to_account,
            'amount': amount
        }
    
    @audit_transaction('DEPOSIT')
    @validate_positive_amount
    @check_account_exists
    def deposit(self, account_id: str, amount: float, **kwargs) -> dict:
        """Deposit money to account"""
        return {
            'status': 'success',
            'account_id': account_id,
            'amount': amount
        }

# Usage Examples
print("=== Banking Service with Decorators ===")

service = BankingService()

# Example 1: Successful withdrawal
print("\n1. Successful Withdrawal:")
try:
    result = service.withdraw('ACC001', 500.0, authenticated=True)
    print(f"✅ {result}")
except Exception as e:
    print(f"❌ {e}")

# Example 2: Invalid amount (caught by validator)
print("\n2. Invalid Amount:")
try:
    result = service.withdraw('ACC001', -100.0, authenticated=True)
    print(f"✅ {result}")
except ValueError as e:
    print(f"❌ {e}")

# Example 3: Invalid account
print("\n3. Invalid Account:")
try:
    result = service.withdraw('ACC999', 100.0, authenticated=True)
    print(f"✅ {result}")
except ValueError as e:
    print(f"❌ {e}")

# Example 4: Not authenticated
print("\n4. Not Authenticated:")
try:
    result = service.withdraw('ACC001', 100.0, authenticated=False)
    print(f"✅ {result}")
except PermissionError as e:
    print(f"❌ {e}")

# Example 5: Rate limiting
print("\n5. Rate Limiting Test:")
for i in range(7):
    try:
        result = service.withdraw('ACC001', 10.0, authenticated=True)
        print(f"  Withdrawal {i+1}: ✅ Success")
    except Exception as e:
        print(f"  Withdrawal {i+1}: ❌ {e}")

# View audit logs
print("\n=== Audit Logs ===")
logs = audit_log.get_logs()
for log in logs[-5:]:  # Show last 5 logs
    print(f"{log['timestamp']}: {log['type']} - {log['status']} ({log.get('duration_ms', 0)}ms)")
```

---

## 8. Scenario: When to Use Decorators vs Regular Functions

```python
import functools
import time

# ❌ BAD: Using decorator for one-time use
@functools.lru_cache(maxsize=128)
def calculate_one_time_value(x):
    """This is only called once - decorator is overhead"""
    return x * 2

result = calculate_one_time_value(5)  # Caching pointless here

# ✅ GOOD: Using decorator for repeated calls
@functools.lru_cache(maxsize=128)
def get_exchange_rate(currency: str) -> float:
    """Called many times with same currencies - caching helps!"""
    time.sleep(0.5)  # Simulate API call
    return 1.0 + hash(currency) % 100 / 100

# Called multiple times
for _ in range(100):
    rate = get_exchange_rate('USD')  # Only makes 1 API call!

# ❌ BAD: Complex logic in decorator (hard to test/debug)
def overly_complex_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # 50 lines of complex business logic here
        # Validations, transformations, logging, etc.
        # This should be in a separate service class!
        result = func(*args, **kwargs)
        # More complex logic
        return result
    return wrapper

# ✅ GOOD: Simple, focused decorator
def simple_timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__}: {time.time() - start:.2f}s")
        return result
    return wrapper

# ❌ BAD: Decorator modifying core business logic
def bad_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Changing business rules inside decorator - BAD!
        result = func(*args, **kwargs)
        result['amount'] *= 1.1  # Adding fee in decorator?
        return result
    return wrapper

# ✅ GOOD: Decorator for cross-cutting concerns only
def good_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Logging, monitoring, validation - OK!
        logger.info(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        logger.info(f"Completed {func.__name__}")
        return result  # Don't modify business result
    return wrapper
```

---

## 9. Decorator Order Matters!

```python
import functools

def decorator_a(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("A: Before")
        result = func(*args, **kwargs)
        print("A: After")
        return result
    return wrapper

def decorator_b(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("B: Before")
        result = func(*args, **kwargs)
        print("B: After")
        return result
    return wrapper

# Order: bottom to top (B wraps func, A wraps B)
@decorator_a
@decorator_b
def my_function():
    print("Function executing")

print("Execution order:")
my_function()

# Output:
# A: Before
# B: Before
# Function executing
# B: After
# A: After

# Banking Example: Correct Order
print("\n=== Banking Example: Decorator Order ===")

@audit_transaction('TRANSFER')      # 4. Log everything (outermost)
@require_authentication            # 3. Check authentication
@validate_positive_amount          # 2. Validate amount
@check_account_exists             # 1. Check account exists (innermost)
def transfer_funds(from_acc: str, to_acc: str, amount: float, **kwargs):
    return f"Transferred ${amount}"

# Execution order:
# 1. check_account_exists runs first
# 2. Then validate_positive_amount
# 3. Then require_authentication
# 4. Finally audit_transaction (logs everything)
```

---

## Key Takeaways:

### When to Use Decorators:

✅ **GOOD Use Cases:**
- **Logging and auditing** (transaction logs, security logs)
- **Authentication and authorization** (checking permissions)
- **Validation** (input validation, business rules)
- **Caching** (expensive function results)
- **Rate limiting** (API throttling)
- **Timing and profiling** (performance monitoring)
- **Retry logic** (network calls, external services)
- **Error handling** (converting exceptions)

❌ **BAD Use Cases:**
- **Core business logic** (should be in main function)
- **One-time operations** (decorator overhead not worth it)
- **Complex transformations** (hard to test/debug)
- **State management** (use classes instead)
- **Conditional logic** (if/else belongs in function)

### Decorator Best Practices:

```python
# 1. Always use @functools.wraps
@functools.wraps(func)  # Preserves __name__, __doc__, etc.

# 2. Keep decorators simple and focused
# One decorator = one responsibility

# 3. Make decorators reusable
# Should work with any function, not specific to one

# 4. Document decorator behavior
def my_decorator(func):
    """Document what this decorator does"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Implementation
        pass
    return wrapper

# 5. Consider decorator order
# Most specific first, most general last
```

### Banking Context:

**Critical decorators for banking:**
- `@audit_transaction` - Regulatory compliance
- `@require_authentication` - Security
- `@validate_amount` - Data integrity
- `@rate_limit` - Fraud prevention
- `@retry` - Reliability for external services
- `@cache` - Performance (exchange rates, etc.)

**Performance impact:**
- Decorators add ~0.1-1ms overhead per call
- Acceptable for most banking operations
- Avoid in high-frequency trading systems
- Measure actual impact in production
