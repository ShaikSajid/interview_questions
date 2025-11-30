# Q02: GIL (Global Interpreter Lock)

## Question: What is the Global Interpreter Lock and how does it affect Python threading?

---

## Answer:

The **GIL** is a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecode simultaneously.

---

## 1. What is GIL?

```python
# Only ONE thread can execute Python code at a time
# Even on multi-core systems!

import threading
import time

def cpu_bound_task():
    """Simulates CPU-intensive work"""
    count = 0
    for i in range(10_000_000):
        count += 1
    return count

# Single-threaded execution
start = time.time()
cpu_bound_task()
cpu_bound_task()
end = time.time()
print(f"Single-threaded: {end - start:.2f}s")

# Multi-threaded execution (NO speedup due to GIL!)
start = time.time()
threads = [
    threading.Thread(target=cpu_bound_task),
    threading.Thread(target=cpu_bound_task)
]
for t in threads:
    t.start()
for t in threads:
    t.join()
end = time.time()
print(f"Multi-threaded: {end - start:.2f}s")
# Similar time due to GIL!
```

---

## 2. When GIL Matters vs Doesn't Matter

| Scenario | GIL Impact | Solution |
|----------|------------|----------|
| **CPU-bound tasks** (calculations) | ❌ Slows down | Use **multiprocessing** |
| **I/O-bound tasks** (network, files) | ✅ No impact | Use **threading** |
| **Async operations** | ✅ No impact | Use **asyncio** |

---

## 3. Banking Example: I/O-Bound (Threading Works!)

```python
import threading
import time
import requests

def fetch_account_balance(account_id: str) -> dict:
    """Fetch balance from external API (I/O-bound)"""
    # Simulated API call
    time.sleep(0.5)  # I/O wait - GIL released!
    return {'account_id': account_id, 'balance': 1000.0}

# ✅ GOOD: Threading for I/O-bound tasks
def check_multiple_accounts_threaded(account_ids: list) -> list:
    results = []
    threads = []
    
    def fetch_and_store(acc_id):
        results.append(fetch_account_balance(acc_id))
    
    for acc_id in account_ids:
        t = threading.Thread(target=fetch_and_store, args=(acc_id,))
        threads.append(t)
        t.start()
    
    for t in threads:
        t.join()
    
    return results

# Test
accounts = [f"ACC{i:03d}" for i in range(10)]
start = time.time()
balances = check_multiple_accounts_threaded(accounts)
print(f"Threading: {time.time() - start:.2f}s")  # ~0.5s (parallel)
```

---

## 4. Banking Example: CPU-Bound (Use Multiprocessing!)

```python
from multiprocessing import Pool
import time

def calculate_loan_eligibility(customer_data: dict) -> dict:
    """CPU-intensive credit scoring (CPU-bound)"""
    score = 0
    # Simulate heavy computation
    for _ in range(1_000_000):
        score += customer_data['income'] * 0.001
    
    return {
        'customer_id': customer_data['id'],
        'eligible': score > 50000
    }

# ❌ BAD: Threading for CPU-bound (no speedup)
def process_customers_threaded(customers: list) -> list:
    results = []
    threads = []
    
    def calc_and_store(cust):
        results.append(calculate_loan_eligibility(cust))
    
    for cust in customers:
        t = threading.Thread(target=calc_and_store, args=(cust,))
        threads.append(t)
        t.start()
    
    for t in threads:
        t.join()
    
    return results

# ✅ GOOD: Multiprocessing for CPU-bound
def process_customers_multiprocess(customers: list) -> list:
    with Pool(processes=4) as pool:
        return pool.map(calculate_loan_eligibility, customers)

# Test
customers = [{'id': i, 'income': 5000} for i in range(8)]

start = time.time()
process_customers_threaded(customers)
print(f"Threading: {time.time() - start:.2f}s")  # Slow

start = time.time()
process_customers_multiprocess(customers)
print(f"Multiprocessing: {time.time() - start:.2f}s")  # Fast!
```

---

## 5. Solution: Use Asyncio for Concurrent I/O

```python
import asyncio
import aiohttp

async def fetch_transaction_async(txn_id: str) -> dict:
    """Async I/O - no GIL issues"""
    await asyncio.sleep(0.1)  # Simulated API call
    return {'id': txn_id, 'status': 'completed'}

async def process_transactions(txn_ids: list) -> list:
    """Process multiple transactions concurrently"""
    tasks = [fetch_transaction_async(txn_id) for txn_id in txn_ids]
    return await asyncio.gather(*tasks)

# Usage
txn_ids = [f"TXN{i:04d}" for i in range(100)]
results = asyncio.run(process_transactions(txn_ids))
print(f"Processed {len(results)} transactions concurrently")
```

---

## 6. Decision Matrix

```python
# Choose the right approach for banking tasks:

def choose_concurrency_model(task_type: str):
    """
    I/O-bound (API calls, DB queries, file reads):
        → Use threading or asyncio
    
    CPU-bound (risk calculations, data processing):
        → Use multiprocessing
    
    Mixed workload:
        → Use asyncio + ThreadPoolExecutor for I/O
        → Use ProcessPoolExecutor for CPU tasks
    """
    pass
```

---

## Best Practices:

### ✅ DO:
- Use **threading** for I/O-bound tasks (database, API calls)
- Use **multiprocessing** for CPU-bound tasks (calculations)
- Use **asyncio** for high-concurrency I/O operations
- Profile your code to identify bottlenecks

### ❌ DON'T:
- Use threading for CPU-intensive calculations
- Assume threading will speed up all tasks
- Ignore GIL limitations in production
- Mix threading and multiprocessing without understanding GIL

---

## 7. Real-World Scenario: API Gateway Processing

```python
import threading
import time
import requests
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import multiprocessing

# Scenario: Banking API Gateway handling 1000 customer requests

def fetch_customer_data(customer_id: int) -> dict:
    """
    I/O-bound: Fetch customer data from external service
    Network call releases GIL - threading is perfect!
    """
    # Simulated API call (I/O wait)
    time.sleep(0.1)  # GIL released during sleep!
    return {
        'customer_id': customer_id,
        'name': f'Customer {customer_id}',
        'balance': customer_id * 1000
    }

def calculate_credit_score(customer_data: dict) -> int:
    """
    CPU-bound: Complex mathematical calculation
    Heavy CPU work - GIL becomes a bottleneck!
    """
    score = 0
    # Simulated heavy computation
    for i in range(5_000_000):
        score += (customer_data['balance'] * 0.001) / (i + 1)
    return int(score)

# Test 1: I/O-bound with Threading (GOOD!)
print("=== Test 1: I/O-Bound (API Calls) ===")
customer_ids = list(range(100))

# Sequential
start = time.time()
for cid in customer_ids:
    fetch_customer_data(cid)
seq_time = time.time() - start
print(f"Sequential: {seq_time:.2f}s")

# Threading (FAST!)
start = time.time()
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(fetch_customer_data, customer_ids))
thread_time = time.time() - start
print(f"Threading: {thread_time:.2f}s")
print(f"Speedup: {seq_time/thread_time:.2f}x faster!")

# Test 2: CPU-bound with Threading (BAD!)
print("\n=== Test 2: CPU-Bound (Credit Scoring) ===")
test_customers = [{'customer_id': i, 'balance': i * 1000} for i in range(4)]

# Sequential
start = time.time()
for customer in test_customers:
    calculate_credit_score(customer)
seq_time = time.time() - start
print(f"Sequential: {seq_time:.2f}s")

# Threading (NO SPEEDUP due to GIL!)
start = time.time()
with ThreadPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(calculate_credit_score, test_customers))
thread_time = time.time() - start
print(f"Threading: {thread_time:.2f}s")
print(f"Speedup: {seq_time/thread_time:.2f}x (almost no improvement!)")

# Multiprocessing (FAST!)
start = time.time()
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(calculate_credit_score, test_customers))
process_time = time.time() - start
print(f"Multiprocessing: {process_time:.2f}s")
print(f"Speedup: {seq_time/process_time:.2f}x faster!")
```

---

## 8. Scenario: When GIL Doesn't Matter

```python
import threading
import time
import asyncio
import aiohttp

# Scenario 1: Database Queries (I/O-bound - GIL released)
def query_transactions(account_id: str) -> list:
    """Database query - GIL released during I/O"""
    time.sleep(0.5)  # Simulated DB query
    return [f"TXN{i}" for i in range(100)]

# ✅ Threading works great here
accounts = [f"ACC{i:03d}" for i in range(10)]

start = time.time()
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(query_transactions, accounts))
print(f"10 DB queries in parallel: {time.time() - start:.2f}s")  # ~0.5s

# Scenario 2: File I/O (GIL released)
def process_transaction_file(filename: str) -> int:
    """Read and process file - GIL released during I/O"""
    with open(filename, 'r') as f:
        lines = f.readlines()  # I/O operation
        return len(lines)

# ✅ Threading is fine for file operations

# Scenario 3: Network Requests (GIL released)
import urllib.request

def fetch_exchange_rate(currency: str) -> float:
    """Fetch exchange rate from API"""
    # Network I/O - GIL released
    time.sleep(0.2)  # Simulated API call
    return 1.0 + hash(currency) % 100 / 100

currencies = ['USD', 'EUR', 'GBP', 'JPY', 'AUD']

start = time.time()
with ThreadPoolExecutor(max_workers=5) as executor:
    rates = list(executor.map(fetch_exchange_rate, currencies))
print(f"Fetched {len(rates)} exchange rates: {time.time() - start:.2f}s")
```

---

## 9. Scenario: When GIL Becomes a Problem

```python
import threading
import multiprocessing
import time

# Scenario: Fraud Detection Algorithm (CPU-intensive)
def detect_fraud(transactions: list) -> list:
    """
    CPU-intensive fraud detection
    GIL prevents parallel execution!
    """
    suspicious = []
    for txn in transactions:
        # Complex pattern matching (pure CPU work)
        risk_score = 0
        for i in range(1_000_000):  # Heavy computation
            risk_score += txn['amount'] * 0.001 / (i + 1)
        
        if risk_score > 50:
            suspicious.append(txn)
    
    return suspicious

# Generate test data
all_transactions = [
    [{'id': f'TXN{j:04d}', 'amount': j * 100} for j in range(i*100, (i+1)*100)]
    for i in range(4)  # 4 batches of 100 transactions
]

print("=== Fraud Detection Performance ===")

# Sequential
start = time.time()
for batch in all_transactions:
    detect_fraud(batch)
seq_time = time.time() - start
print(f"Sequential: {seq_time:.2f}s")

# Threading (NO SPEEDUP!)
start = time.time()
threads = []
for batch in all_transactions:
    t = threading.Thread(target=detect_fraud, args=(batch,))
    threads.append(t)
    t.start()
for t in threads:
    t.join()
thread_time = time.time() - start
print(f"Threading: {thread_time:.2f}s (similar to sequential!)")

# Multiprocessing (REAL SPEEDUP!)
start = time.time()
with multiprocessing.Pool(processes=4) as pool:
    pool.map(detect_fraud, all_transactions)
process_time = time.time() - start
print(f"Multiprocessing: {process_time:.2f}s")
print(f"\nSpeedup with multiprocessing: {seq_time/process_time:.2f}x")
```

---

## 10. Decision Tree for Concurrency

```python
def choose_concurrency_strategy(task_description: str):
    """
    Decision tree for choosing the right concurrency approach
    """
    print(f"\nTask: {task_description}")
    print("Decision Process:")
    
    # Question 1: CPU-bound or I/O-bound?
    print("\n1. Is this task CPU-bound or I/O-bound?")
    print("   CPU-bound: Heavy calculations, data processing")
    print("   I/O-bound: Network calls, database queries, file operations")
    
    # I/O-bound path
    print("\n   If I/O-bound:")
    print("   • Use threading.Thread or ThreadPoolExecutor")
    print("   • Or use asyncio for high concurrency")
    print("   • GIL doesn't matter (released during I/O)")
    
    # CPU-bound path
    print("\n   If CPU-bound:")
    print("   • Use multiprocessing.Process or ProcessPoolExecutor")
    print("   • Each process has its own GIL")
    print("   • True parallel execution on multiple cores")
    
    # Question 2: How many concurrent operations?
    print("\n2. How many concurrent operations?")
    print("   • 10-100: Threading or asyncio")
    print("   • 1000s: asyncio (more efficient)")
    print("   • CPU-bound: multiprocessing (# of CPU cores)")

# Examples
print("="*60)
choose_concurrency_strategy("Fetch account balances from 50 accounts (API calls)")
print("\n✅ Recommendation: Use threading.ThreadPoolExecutor(max_workers=10)")

print("\n" + "="*60)
choose_concurrency_strategy("Calculate risk scores for 10,000 loan applications")
print("\n✅ Recommendation: Use multiprocessing.Pool(processes=cpu_count())")

print("\n" + "="*60)
choose_concurrency_strategy("Process 1,000 webhook notifications")
print("\n✅ Recommendation: Use asyncio with aiohttp")
```

---

## 11. Production Example: Mixed Workload

```python
import asyncio
import aiohttp
from concurrent.futures import ProcessPoolExecutor
import multiprocessing

class BankingService:
    """
    Real-world banking service handling mixed workloads
    """
    
    def __init__(self):
        # Process pool for CPU-intensive tasks
        self.process_pool = ProcessPoolExecutor(
            max_workers=multiprocessing.cpu_count()
        )
    
    async def process_loan_application(self, application: dict) -> dict:
        """
        Mixed workload: I/O + CPU
        """
        # Step 1: Fetch customer data (I/O-bound - use async)
        async with aiohttp.ClientSession() as session:
            customer_data = await self.fetch_customer_data(session, application['customer_id'])
        
        # Step 2: Calculate credit score (CPU-bound - use multiprocessing)
        loop = asyncio.get_event_loop()
        credit_score = await loop.run_in_executor(
            self.process_pool,
            self.calculate_credit_score,
            customer_data
        )
        
        # Step 3: Store result (I/O-bound - use async)
        result = await self.store_result(application['id'], credit_score)
        
        return result
    
    async def fetch_customer_data(self, session, customer_id: str) -> dict:
        """Async I/O operation"""
        await asyncio.sleep(0.1)  # Simulated API call
        return {'customer_id': customer_id, 'income': 50000}
    
    def calculate_credit_score(self, customer_data: dict) -> int:
        """CPU-intensive calculation in separate process"""
        score = 0
        for i in range(5_000_000):
            score += customer_data['income'] * 0.001 / (i + 1)
        return int(score)
    
    async def store_result(self, app_id: str, score: int) -> dict:
        """Async I/O operation"""
        await asyncio.sleep(0.05)  # Simulated DB write
        return {'application_id': app_id, 'score': score, 'status': 'completed'}
    
    async def process_batch(self, applications: list) -> list:
        """Process multiple applications concurrently"""
        tasks = [self.process_loan_application(app) for app in applications]
        return await asyncio.gather(*tasks)

# Usage
async def main():
    service = BankingService()
    
    applications = [
        {'id': f'APP{i:03d}', 'customer_id': f'CUST{i:03d}'}
        for i in range(10)
    ]
    
    print("Processing 10 loan applications...")
    start = time.time()
    results = await service.process_batch(applications)
    duration = time.time() - start
    
    print(f"Processed {len(results)} applications in {duration:.2f}s")
    print(f"Average: {duration/len(results):.2f}s per application")

# asyncio.run(main())
```

---

## Key Takeaways:

### Understanding GIL:
- **GIL = Mutex**: Only one thread executes Python bytecode at a time
- **Not a Python limitation**: It's a CPython implementation detail
- **GIL is released**: During I/O operations (network, files, DB)
- **GIL is NOT released**: During pure Python CPU work

### Decision Matrix:

| Task Type | Concurrency | Why |
|-----------|-------------|-----|
| **API calls, DB queries** | Threading | I/O-bound, GIL released |
| **File operations** | Threading | I/O-bound, GIL released |
| **Web scraping** | Threading/Asyncio | I/O-bound |
| **Data processing** | Multiprocessing | CPU-bound, bypasses GIL |
| **Heavy calculations** | Multiprocessing | CPU-bound |
| **Mixed workload** | Asyncio + ProcessPool | Best of both |

### Banking Applications:

✅ **Use Threading/Asyncio for:**
- Fetching account balances from multiple sources
- Processing payment webhooks
- Sending transaction notifications
- Database queries across multiple tables
- API integrations (payment gateways, credit bureaus)

✅ **Use Multiprocessing for:**
- Credit score calculations
- Fraud detection algorithms
- Risk analysis for loan portfolios
- Large dataset processing (end-of-day reports)
- Machine learning inference

❌ **Don't Use Threading for:**
- Heavy mathematical computations
- Data encryption/decryption at scale
- Image processing (check signatures, documents)
- Complex algorithm execution

### Performance Numbers (Typical):

```python
# I/O-bound task (API call)
# Sequential: 10 requests × 0.5s = 5.0s
# Threading (10 workers): 0.5s (10x speedup!)
# Multiprocessing: 0.5s (same as threading, more overhead)

# CPU-bound task (calculation)
# Sequential: 4 tasks × 2s = 8.0s
# Threading (4 workers): 8.0s (NO speedup due to GIL!)
# Multiprocessing (4 workers): 2.0s (4x speedup!)
```

### When GIL Matters:
- Only for **CPU-bound** tasks in **multi-threaded** code
- Doesn't affect single-threaded code
- Doesn't affect I/O-bound operations
- Doesn't affect multiprocessing
