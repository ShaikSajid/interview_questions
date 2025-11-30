# Q10: Exception Handling

## Question: How does exception handling work in Python?

---

## Answer:

Python uses **try/except/else/finally** blocks to handle errors gracefully.

---

## 1. Basic try/except

```python
def process_payment(amount: float):
    try:
        if amount <= 0:
            raise ValueError("Amount must be positive")
        # Process payment
        return f"Payment of ${amount} successful"
    except ValueError as e:
        print(f"Error: {e}")
        return None

# Usage
result = process_payment(-100)  # Handles error gracefully
print(result)  # None
```

---

## 2. Multiple Exceptions

```python
def transfer_funds(from_acc: str, to_acc: str, amount: float):
    try:
        # Simulate various errors
        if amount <= 0:
            raise ValueError("Invalid amount")
        if from_acc == to_acc:
            raise ValueError("Cannot transfer to same account")
        # Simulated balance check
        balance = get_balance(from_acc)
        if balance < amount:
            raise InsufficientFundsError("Not enough balance")
        
        return "Transfer successful"
    
    except ValueError as e:
        return f"Validation error: {e}"
    except InsufficientFundsError as e:
        return f"Balance error: {e}"
    except Exception as e:
        return f"Unexpected error: {e}"

# Custom exception
class InsufficientFundsError(Exception):
    pass

def get_balance(acc):
    return 1000  # Simulated
```

---

## 3. try/except/else/finally

```python
def process_transaction(txn_id: str):
    file = None
    try:
        # Risky operation
        file = open(f'transaction_{txn_id}.txt', 'r')
        data = file.read()
        result = parse_transaction(data)
    
    except FileNotFoundError:
        print(f"Transaction file not found: {txn_id}")
        result = None
    
    except Exception as e:
        print(f"Error processing transaction: {e}")
        result = None
    
    else:
        # Runs only if NO exception occurred
        print("Transaction processed successfully")
    
    finally:
        # ALWAYS runs (cleanup)
        if file:
            file.close()
            print("File closed")
    
    return result

def parse_transaction(data):
    return {"status": "parsed"}
```

---

## 4. Custom Banking Exceptions

```python
class BankingException(Exception):
    """Base exception for banking operations"""
    pass

class InsufficientFundsError(BankingException):
    """Raised when account has insufficient funds"""
    pass

class InvalidAccountError(BankingException):
    """Raised when account is invalid"""
    pass

class DailyLimitExceededError(BankingException):
    """Raised when daily transaction limit exceeded"""
    pass

class BankAccount:
    def __init__(self, account_id: str, balance: float):
        self.account_id = account_id
        self.balance = balance
        self.daily_limit = 10000
        self.daily_total = 0
    
    def withdraw(self, amount: float):
        # Validation chain
        if amount <= 0:
            raise ValueError("Amount must be positive")
        
        if self.balance < amount:
            raise InsufficientFundsError(
                f"Insufficient funds. Balance: ${self.balance}, Requested: ${amount}"
            )
        
        if self.daily_total + amount > self.daily_limit:
            raise DailyLimitExceededError(
                f"Daily limit exceeded. Limit: ${self.daily_limit}"
            )
        
        # Process withdrawal
        self.balance -= amount
        self.daily_total += amount
        return f"Withdrawal successful. New balance: ${self.balance}"

# Usage
account = BankAccount("ACC001", 1000)

try:
    result = account.withdraw(1500)
    print(result)
except InsufficientFundsError as e:
    print(f"❌ {e}")
except DailyLimitExceededError as e:
    print(f"❌ {e}")
except ValueError as e:
    print(f"❌ Validation error: {e}")
```

---

## 5. Context Manager for Exception Handling

```python
from contextlib import contextmanager

@contextmanager
def transaction_scope(db_connection):
    """Auto-rollback on error"""
    try:
        print("BEGIN TRANSACTION")
        yield db_connection
        print("COMMIT")
    except Exception as e:
        print(f"ROLLBACK due to: {e}")
        raise

# Usage
class MockDB:
    def execute(self, sql):
        print(f"Executing: {sql}")
        if "ERROR" in sql:
            raise Exception("Database error")

db = MockDB()

try:
    with transaction_scope(db):
        db.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 'ACC001'")
        db.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 'ACC002'")
except Exception as e:
    print(f"Transaction failed: {e}")
```

---

## 6. Logging Exceptions

```python
import logging
import traceback

logging.basicConfig(level=logging.ERROR)
logger = logging.getLogger(__name__)

def process_payment_with_logging(amount: float):
    try:
        if amount <= 0:
            raise ValueError("Invalid amount")
        # Process payment
        return "Success"
    except Exception as e:
        # Log full stack trace
        logger.error(
            f"Payment processing failed: {e}",
            exc_info=True  # Includes stack trace
        )
        return None

process_payment_with_logging(-100)
```

---

## 7. Raise vs Raise From

```python
def fetch_account_data(account_id: str):
    try:
        # Simulate database error
        raise ConnectionError("Database connection failed")
    except ConnectionError as e:
        # Raise new exception while preserving original
        raise InvalidAccountError(f"Cannot fetch account {account_id}") from e

try:
    fetch_account_data("ACC001")
except InvalidAccountError as e:
    print(f"Error: {e}")
    print(f"Caused by: {e.__cause__}")  # Shows original ConnectionError
```

---

## Best Practices:

### ✅ DO:
- Catch specific exceptions (not bare `except`)
- Use custom exception classes for domain errors
- Log exceptions with full stack traces
- Clean up resources in `finally` or context managers
- Raise from original exception to preserve context

### ❌ DON'T:
- Use bare `except:` (catches everything)
- Silently swallow exceptions
- Use exceptions for control flow
- Catch Exception without re-raising or logging
- Forget to clean up resources

---

## Key Takeaways:

- **try/except** for error handling
- **finally** for guaranteed cleanup
- **Custom exceptions** for banking domain errors
- **Context managers** for automatic resource management
- **Banking use**: transaction rollbacks, validation, logging
