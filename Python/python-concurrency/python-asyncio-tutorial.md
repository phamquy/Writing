---
icon: rotate-exclamation
---

# Python Asyncio Tutorial

`asyncio` is Python's built-in library for writing **concurrent code** using the **async/await** syntax. It's ideal for **I/O-bound** and high-level **structured network code**.

### Key Concepts

| Concept        | Description                                                             |
| -------------- | ----------------------------------------------------------------------- |
| **Coroutine**  | A function defined with `async def` that can be paused and resumed      |
| **Event Loop** | The core that runs async tasks, handles I/O, and manages callbacks      |
| **Task**       | A wrapper around a coroutine that schedules it to run on the event loop |
| **Future**     | A low-level awaitable object representing an eventual result            |
| **await**      | Keyword to pause coroutine execution until the awaited object completes |

### When to Use asyncio

✅ **Good for:**

* Network I/O (HTTP requests, database queries, websockets)
* File I/O operations
* High-concurrency scenarios (thousands of connections)
* Web servers and API clients

❌ **Not ideal for:**

* CPU-bound tasks (use `multiprocessing` instead)
* Simple sequential code

### 1. Basic Coroutines and `async`/`await`

A coroutine is defined with `async def` and must be awaited to execute.

```python
import asyncio

# Define a simple coroutine
async def say_hello(name: str, delay: float) -> str:
    """A coroutine that waits and then returns a greeting."""
    print(f"Starting to greet {name}...")
    await asyncio.sleep(delay)  # Non-blocking sleep
    message = f"Hello, {name}!"
    print(message)
    return message

# Run a single coroutine
# In Jupyter, we can use await directly (top-level await)
result = await say_hello("World", 1)
print(f"Returned: {result}")
```

```
Starting to greet World...
Hello, World!
Returned: Hello, World!
```

### 2. Running Multiple Coroutines Concurrently

#### Using `asyncio.gather()` - Run multiple coroutines and collect results

```python
import time

async def fetch_data(source: str, delay: float) -> dict:
    """Simulate fetching data from a source."""
    print(f"Fetching from {source}...")
    await asyncio.sleep(delay)
    return {"source": source, "data": f"Data from {source}"}

# Run multiple coroutines concurrently with gather()
start = time.perf_counter()

results = await asyncio.gather(
    fetch_data("Database", 2),
    fetch_data("API", 1),
    fetch_data("Cache", 0.5),
)

elapsed = time.perf_counter() - start
print(f"\nAll results: {results}")
print(f"Total time: {elapsed:.2f}s (not 3.5s because they ran concurrently!)")
```

```
Fetching from Database...
Fetching from API...
Fetching from Cache...

All results: [{'source': 'Database', 'data': 'Data from Database'}, {'source': 'API', 'data': 'Data from API'}, {'source': 'Cache', 'data': 'Data from Cache'}]
Total time: 2.00s (not 3.5s because they ran concurrently!)
```

#### Using `asyncio.create_task()` - Schedule coroutines as Tasks

Tasks allow you to schedule coroutines to run "in the background" while you do other work.

```python
async def background_task(name: str, duration: float):
    """A task that runs in the background."""
    print(f"Task {name} started")
    await asyncio.sleep(duration)
    print(f"Task {name} completed")
    return f"Result from {name}"

async def main_with_tasks():
    # Create tasks - they start running immediately
    task1 = asyncio.create_task(background_task("A", 2))
    task2 = asyncio.create_task(background_task("B", 1))
    
    # Do some other work while tasks run
    print("Main: doing some work...")
    await asyncio.sleep(0.5)
    print("Main: still working...")
    
    # Wait for tasks to complete
    result1 = await task1
    result2 = await task2
    
    print(f"Results: {result1}, {result2}")

await main_with_tasks()
```

```
Main: doing some work...
Task A started
Task B started
Main: still working...
Task B completed
Task A completed
Results: Result from A, Result from B
```

#### Using `asyncio.TaskGroup` (Python 3.11+) - Structured Concurrency

TaskGroup provides better error handling and ensures all tasks are properly cleaned up.

```python
async def worker(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    print(f"Worker {name} done")
    return f"Result-{name}"

async def main_with_taskgroup():
    results = []
    
    # TaskGroup ensures all tasks complete or are cancelled on error
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(worker("X", 1))
        task2 = tg.create_task(worker("Y", 0.5))
        task3 = tg.create_task(worker("Z", 1.5))
    
    # All tasks are guaranteed to be done here
    print(f"Results: {task1.result()}, {task2.result()}, {task3.result()}")

await main_with_taskgroup()
```

```
Worker Y done
Worker X done
Worker Z done
Results: Result-X, Result-Y, Result-Z
```

### 3. Error Handling in Async Code

```python
async def might_fail(name: str, should_fail: bool = False):
    await asyncio.sleep(0.5)
    if should_fail:
        raise ValueError(f"Task {name} failed!")
    return f"Success from {name}"

# Using gather with return_exceptions=True
results = await asyncio.gather(
    might_fail("A", should_fail=False),
    might_fail("B", should_fail=True),
    might_fail("C", should_fail=False),
    return_exceptions=True  # Don't raise, return exceptions as results
)

for i, result in enumerate(results):
    if isinstance(result, Exception):
        print(f"Task {i}: ERROR - {result}")
    else:
        print(f"Task {i}: {result}")
```

```
Task 0: Success from A
Task 1: ERROR - Task B failed!
Task 2: Success from C
```

### 4. Timeouts and Cancellation

```python
async def slow_operation():
    print("Starting slow operation...")
    await asyncio.sleep(10)
    return "Done"

# Using asyncio.timeout (Python 3.11+)
try:
    async with asyncio.timeout(2):
        result = await slow_operation()
except TimeoutError:
    print("Operation timed out!")

# Using asyncio.wait_for (works in older Python versions too)
try:
    result = await asyncio.wait_for(slow_operation(), timeout=1.5)
except asyncio.TimeoutError:
    print("wait_for: Operation timed out!")
```

```
Starting slow operation...
Operation timed out!
Starting slow operation...
wait_for: Operation timed out!
```

```python
# Manual task cancellation
async def cancellable_task():
    try:
        print("Task started, will run for 10 seconds...")
        await asyncio.sleep(10)
        return "Completed"
    except asyncio.CancelledError:
        print("Task was cancelled! Cleaning up...")
        raise  # Re-raise to propagate cancellation

async def main_cancel():
    task = asyncio.create_task(cancellable_task())
    
    # Let it run for a bit
    await asyncio.sleep(1)
    
    # Cancel the task
    task.cancel()
    
    try:
        await task
    except asyncio.CancelledError:
        print("Main: Confirmed task was cancelled")

await main_cancel()
```

```
Task started, will run for 10 seconds...
Task was cancelled! Cleaning up...
Main: Confirmed task was cancelled
```

### 5. Async Iterators and Generators

#### Async for loops with `async for`

```python
# Async generator - yields values asynchronously
async def async_range(start: int, stop: int, delay: float = 0.5):
    """Async generator that yields numbers with a delay."""
    for i in range(start, stop):
        await asyncio.sleep(delay)
        yield i

# Consume with async for
async def consume_async_generator():
    print("Consuming async generator:")
    async for num in async_range(1, 5, delay=0.3):
        print(f"  Got: {num}")

await consume_async_generator()
```

```
Consuming async generator:
  Got: 1
  Got: 2
  Got: 3
  Got: 4
```

### 6. Synchronization Primitives

#### asyncio.Lock - Mutual Exclusion

```python
# Shared resource that needs protection
shared_counter = 0
lock = asyncio.Lock()

async def increment_with_lock(name: str, times: int):
    global shared_counter
    for _ in range(times):
        async with lock:  # Acquire lock
            current = shared_counter
            await asyncio.sleep(0.01)  # Simulate some work
            shared_counter = current + 1
    print(f"{name} done")

async def demo_lock():
    global shared_counter
    shared_counter = 0
    
    await asyncio.gather(
        increment_with_lock("Task1", 5),
        increment_with_lock("Task2", 5),
        increment_with_lock("Task3", 5),
    )
    
    print(f"Final counter value: {shared_counter} (expected: 15)")

await demo_lock()
```

```
Task1 done
Task2 done
Task3 done
Final counter value: 15 (expected: 15)
```

#### asyncio.Semaphore - Limit Concurrent Access

```python
# Limit concurrent API calls (e.g., rate limiting)
semaphore = asyncio.Semaphore(3)  # Allow max 3 concurrent operations

async def rate_limited_api_call(request_id: int):
    async with semaphore:
        print(f"Request {request_id}: Starting (semaphore acquired)")
        await asyncio.sleep(1)  # Simulate API call
        print(f"Request {request_id}: Done")
        return f"Response-{request_id}"

async def demo_semaphore():
    # Try to make 10 requests, but only 3 can run at a time
    tasks = [rate_limited_api_call(i) for i in range(10)]
    results = await asyncio.gather(*tasks)
    print(f"\nAll {len(results)} requests completed")

await demo_semaphore()
```

```
Request 0: Starting (semaphore acquired)
Request 1: Starting (semaphore acquired)
Request 2: Starting (semaphore acquired)
Request 0: Done
Request 1: Done
Request 2: Done
Request 3: Starting (semaphore acquired)
Request 4: Starting (semaphore acquired)
Request 5: Starting (semaphore acquired)
Request 3: Done
Request 4: Done
Request 5: Done
Request 6: Starting (semaphore acquired)
Request 7: Starting (semaphore acquired)
Request 8: Starting (semaphore acquired)
Request 6: Done
Request 7: Done
Request 8: Done
Request 9: Starting (semaphore acquired)
Request 9: Done

All 10 requests completed
```

#### asyncio.Event - Signal Between Coroutines

```python
event = asyncio.Event()

async def waiter(name: str):
    print(f"{name}: Waiting for event...")
    await event.wait()
    print(f"{name}: Event received! Proceeding...")

async def setter():
    print("Setter: Doing some initialization...")
    await asyncio.sleep(2)
    print("Setter: Setting event!")
    event.set()

async def demo_event():
    event.clear()  # Reset event
    await asyncio.gather(
        waiter("Waiter-1"),
        waiter("Waiter-2"),
        setter(),
    )

await demo_event()
```

```
Waiter-1: Waiting for event...
Waiter-2: Waiting for event...
Setter: Doing some initialization...
Setter: Setting event!
Waiter-1: Event received! Proceeding...
Waiter-2: Event received! Proceeding...
```

### 7. Queues for Producer-Consumer Pattern

```python
async def producer(queue: asyncio.Queue, n_items: int):
    """Produce items and put them in the queue."""
    for i in range(n_items):
        item = f"item-{i}"
        await asyncio.sleep(0.2)  # Simulate producing
        await queue.put(item)
        print(f"Producer: Added {item}")
    
    # Signal end of production with None
    await queue.put(None)
    print("Producer: Done")

async def consumer(queue: asyncio.Queue, name: str):
    """Consume items from the queue."""
    while True:
        item = await queue.get()
        if item is None:
            # Put sentinel back for other consumers
            await queue.put(None)
            break
        print(f"Consumer {name}: Processing {item}")
        await asyncio.sleep(0.3)  # Simulate processing
        queue.task_done()
    print(f"Consumer {name}: Done")

async def demo_queue():
    queue = asyncio.Queue(maxsize=5)  # Bounded queue
    
    await asyncio.gather(
        producer(queue, 8),
        consumer(queue, "A"),
        consumer(queue, "B"),
    )

await demo_queue()
```

```
Producer: Added item-0
Consumer A: Processing item-0
Producer: Added item-1
Consumer B: Processing item-1
Producer: Added item-2
Consumer A: Processing item-2
Producer: Added item-3
Consumer B: Processing item-3
Producer: Added item-4
Consumer A: Processing item-4
Producer: Added item-5
Consumer B: Processing item-5
Producer: Added item-6
Consumer A: Processing item-6
Producer: Added item-7
Producer: Done
Consumer B: Processing item-7
Consumer A: Done
Consumer B: Done
```

### 8. Running Blocking Code in Async Context

Use `asyncio.to_thread()` (Python 3.9+) to run blocking code without blocking the event loop.

```python
import hashlib

def blocking_cpu_work(data: str) -> str:
    """A blocking CPU-bound function (not async)."""
    # Simulate CPU-intensive work
    for _ in range(100000):
        hashlib.sha256(data.encode()).hexdigest()
    return f"Processed: {data}"

async def demo_to_thread():
    print("Starting blocking work in thread pool...")
    start = time.perf_counter()
    
    # Run blocking functions concurrently in threads
    results = await asyncio.gather(
        asyncio.to_thread(blocking_cpu_work, "data1"),
        asyncio.to_thread(blocking_cpu_work, "data2"),
        asyncio.to_thread(blocking_cpu_work, "data3"),
    )
    
    elapsed = time.perf_counter() - start
    print(f"Results: {results}")
    print(f"Total time: {elapsed:.2f}s")

await demo_to_thread()
```

```
Starting blocking work in thread pool...
Results: ['Processed: data1', 'Processed: data2', 'Processed: data3']
Total time: 0.08s
```

### 9. Async Context Managers

Use `async with` for resources that need async setup/teardown.

```python
class AsyncDatabaseConnection:
    """Example async context manager for database connection."""
    
    def __init__(self, db_name: str):
        self.db_name = db_name
        self.connected = False
    
    async def __aenter__(self):
        print(f"Connecting to {self.db_name}...")
        await asyncio.sleep(0.5)  # Simulate connection
        self.connected = True
        print(f"Connected to {self.db_name}")
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print(f"Closing connection to {self.db_name}...")
        await asyncio.sleep(0.2)  # Simulate cleanup
        self.connected = False
        print(f"Connection closed")
        return False  # Don't suppress exceptions
    
    async def query(self, sql: str):
        if not self.connected:
            raise RuntimeError("Not connected")
        await asyncio.sleep(0.1)
        return f"Result of: {sql}"

# Using the async context manager
async with AsyncDatabaseConnection("mydb") as db:
    result = await db.query("SELECT * FROM users")
    print(f"Query result: {result}")
```

```
Connecting to mydb...
Connected to mydb
Query result: Result of: SELECT * FROM users
Closing connection to mydb...
Connection closed
```

### 10. Real-World Example: Async HTTP Client

Using `aiohttp` for concurrent HTTP requests (common pattern).

```python
# Note: This requires aiohttp to be installed: pip install aiohttp
# Simulated example without actual HTTP calls

async def fetch_url(url: str, delay: float = 0.5) -> dict:
    """Simulate fetching a URL asynchronously."""
    await asyncio.sleep(delay)
    return {"url": url, "status": 200, "content_length": 1024}

async def fetch_all_urls():
    urls = [
        "https://api.example.com/users",
        "https://api.example.com/products",
        "https://api.example.com/orders",
        "https://api.example.com/inventory",
        "https://api.example.com/analytics",
    ]
    
    # Rate limit with semaphore
    semaphore = asyncio.Semaphore(3)
    
    async def fetch_with_limit(url):
        async with semaphore:
            return await fetch_url(url)
    
    start = time.perf_counter()
    results = await asyncio.gather(*[fetch_with_limit(url) for url in urls])
    elapsed = time.perf_counter() - start
    
    print(f"Fetched {len(results)} URLs in {elapsed:.2f}s")
    for r in results:
        print(f"  {r['url']}: {r['status']}")

await fetch_all_urls()
```

```
Fetched 5 URLs in 1.00s
  https://api.example.com/users: 200
  https://api.example.com/products: 200
  https://api.example.com/orders: 200
  https://api.example.com/inventory: 200
  https://api.example.com/analytics: 200
```

### 11. Running asyncio from Regular Python Scripts

In regular Python scripts (not Jupyter), you need to explicitly run the event loop.

```python
# Example of a complete async script structure
# (This would be in a .py file, not Jupyter)

script_example = '''
import asyncio

async def main():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# The recommended way to run async code in a script
if __name__ == "__main__":
    asyncio.run(main())
'''

print(script_example)
```

```
import asyncio

async def main():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# The recommended way to run async code in a script
if __name__ == "__main__":
    asyncio.run(main())
```

### 12. Understanding Event Loop Creation

**Important:** Python does NOT automatically create an async runtime when you use `await`. You must explicitly create one.

#### Why can't you just use `await` at the top level?

```python
# ❌ This causes SyntaxError in a regular Python script
await some_coroutine()  # SyntaxError: 'await' outside async function
```

#### Event Loop Creation by Context

| Context                       | Event Loop Creation                         |
| ----------------------------- | ------------------------------------------- |
| Regular `.py` script          | Must use `asyncio.run()`                    |
| Inside `async def`            | Already running, use `await` directly       |
| Jupyter Notebook              | Loop is pre-created (top-level await works) |
| REPL with `python -m asyncio` | Loop is provided                            |

```python
# What asyncio.run() actually does under the hood:

import asyncio

async def example_coroutine():
    print("Running coroutine")
    await asyncio.sleep(0.1)
    return "Done"

# asyncio.run(example_coroutine()) is roughly equivalent to:
def manual_run():
    loop = asyncio.new_event_loop()      # 1. Create a new event loop
    asyncio.set_event_loop(loop)          # 2. Set it as the current loop
    try:
        return loop.run_until_complete(example_coroutine())  # 3. Run until done
    finally:
        loop.close()                       # 4. Clean up

# In Jupyter, we can't use asyncio.run() because a loop is already running
# That's why we use await directly here
result = await example_coroutine()
print(f"Result: {result}")
```

```
Running coroutine
Result: Done
```

#### Why Jupyter is Special

Jupyter/IPython runs its own event loop in the background, which enables "top-level await". This is **NOT** standard Python behavior - it's a special feature of the IPython kernel.

```python
# In a regular .py file, this would fail:
result = await some_async_function()  # ❌ SyntaxError

# You must wrap it:
async def main():
    result = await some_async_function()  # ✅ Works
    return result

asyncio.run(main())  # ✅ This creates and runs the event loop
```

### 13. Python vs Rust: Async Runtime Comparison

Unlike Rust, Python does **NOT** support pluggable async runtimes. Python has a single built-in runtime (`asyncio`).

#### Comparison Table

| Aspect                    | Rust                               | Python                                      |
| ------------------------- | ---------------------------------- | ------------------------------------------- |
| **Runtime**               | Pluggable (Tokio, async-std, smol) | Single built-in (`asyncio`)                 |
| **Event Loop**            | Provided by runtime crate          | Built into `asyncio` module                 |
| **Customization**         | Full control, choose your runtime  | Limited, can swap event loop implementation |
| **Zero-cost abstraction** | Yes                                | No (overhead exists)                        |

#### Rust Approach (for comparison)

```rust
// Rust - you pick your runtime
#[tokio::main]  // or async-std, smol, etc.
async fn main() {
    // Your async code
}
```

#### What You CAN Customize in Python

1. **Event Loop Implementation** - Use a faster loop like `uvloop`
2. **Alternative Libraries** - Use `trio` or `curio` (different APIs, not drop-in replacements)

```python
# Using uvloop - a faster event loop implementation (drop-in replacement)
# pip install uvloop

uvloop_example = '''
import asyncio
import uvloop

async def main():
    await asyncio.sleep(1)
    print("Done with uvloop!")

# Option 1: Install as default policy (Python 3.11+)
uvloop.install()
asyncio.run(main())

# Option 2: Set event loop policy manually
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
asyncio.run(main())
'''

print("Example: Using uvloop for better performance")
print(uvloop_example)
```

```
Example: Using uvloop for better performance

import asyncio
import uvloop

async def main():
    await asyncio.sleep(1)
    print("Done with uvloop!")

# Option 1: Install as default policy (Python 3.11+)
uvloop.install()
asyncio.run(main())

# Option 2: Set event loop policy manually
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
asyncio.run(main())
```

#### Alternative Async Libraries

| Library    | Description                                  | Compatible with asyncio? |
| ---------- | -------------------------------------------- | ------------------------ |
| **uvloop** | Faster event loop (libuv-based)              | ✅ Yes (drop-in)          |
| **trio**   | Different API, better structured concurrency | ❌ No                     |
| **curio**  | Alternative async library                    | ❌ No                     |
| **anyio**  | Abstraction layer for asyncio/trio           | ✅ Works with both        |

**Bottom line:** Python has ONE async ecosystem centered on `asyncio`. You can optimize the event loop (`uvloop`) or use alternative libraries (`trio`), but you can't plug in completely different runtimes like in Rust.

### 14. Common Pitfall: Calling Async Function Without `await`

When you call an async function **without `await`**, you get a **coroutine object** that is **never executed**.

#### What Happens

| Scenario       | Result                                           |
| -------------- | ------------------------------------------------ |
| No `await`     | Returns coroutine object, code **never runs**    |
| Python warning | `RuntimeWarning: coroutine was never awaited`    |
| Side effects   | Database writes, API calls, etc. **never occur** |
| Resources      | Connections stay open, cleanup doesn't happen    |

```python
import warnings

# Demo: What happens when you forget await
async def my_async_func():
    print("This should print if executed!")
    return 42

# ❌ WRONG - calling without await
print("Calling async function WITHOUT await:")
coro = my_async_func()  # Creates coroutine object, does NOT run!
print(f"Result: {coro}")
print(f"Type: {type(coro)}")
print("Notice: 'This should print' was NEVER printed!\n")

# The coroutine object is just sitting there, never executed
# Python will warn when the coroutine is garbage collected
```

```
Calling async function WITHOUT await:
Result: <coroutine object my_async_func at 0x10bbe2200>
Type: <class 'coroutine'>
Notice: 'This should print' was NEVER printed!
```

```python
# ✅ CORRECT - using await
print("Calling async function WITH await:")
result = await my_async_func()  # Actually executes!
print(f"Result: {result}")
print("Now 'This should print' WAS printed above!")
```

```
Calling async function WITH await:
This should print if executed!
Result: 42
Now 'This should print' WAS printed above!
```

#### How to Schedule Without Immediately Awaiting

If you want to start a coroutine running without blocking, use `create_task()`:

```python
async def long_running_task():
    print("Task: Starting...")
    await asyncio.sleep(1)
    print("Task: Finished!")
    return "task result"

async def demo_create_task():
    # ❌ WRONG - just creates coroutine, doesn't run
    # coro = long_running_task()  # Does nothing!
    
    # ✅ CORRECT - create_task schedules it to run
    task = asyncio.create_task(long_running_task())
    print("Main: Task is now running in background")
    
    # Do other work while task runs
    await asyncio.sleep(0.5)
    print("Main: Still doing other work...")
    
    # Later, get the result
    result = await task
    print(f"Main: Got result: {result}")

await demo_create_task()
```

```
Main: Task is now running in background
Task: Starting...
Main: Still doing other work...
Task: Finished!
Main: Got result: task result
```

### 15. Creating Awaitable Objects Without `async def`

You can create awaitable/coroutine-like objects manually by implementing the `__await__()` method.

#### The Awaitable Protocol

| Method        | Purpose                                     |
| ------------- | ------------------------------------------- |
| `__await__()` | Returns an iterator, makes object awaitable |
| `send(value)` | Send value into coroutine                   |
| `throw(exc)`  | Throw exception into coroutine              |
| `close()`     | Close the coroutine                         |

```python
# Method 1: Implement __await__() to make any class awaitable

class MyAwaitable:
    """A custom awaitable object - no async keyword needed!"""
    
    def __init__(self, value):
        self.value = value
    
    def __await__(self):
        # Delegate to an existing awaitable (asyncio.sleep)
        yield from asyncio.sleep(0.5).__await__()
        return self.value

# Usage - works with await!
result = await MyAwaitable(42)
print(f"Got result from custom awaitable: {result}")
```

```
Got result from custom awaitable: 42
```

```python
# Practical Example: Custom Sleep with logging

class LoggedSleep:
    """Custom awaitable that logs when sleep starts and ends."""
    
    def __init__(self, seconds: float, label: str = ""):
        self.seconds = seconds
        self.label = label
    
    def __await__(self):
        print(f"[{self.label}] Starting sleep for {self.seconds}s")
        yield from asyncio.sleep(self.seconds).__await__()
        print(f"[{self.label}] Woke up!")
        return self.seconds

# Usage
elapsed = await LoggedSleep(0.5, "Timer1")
print(f"Slept for {elapsed} seconds")
```

```
[Timer1] Starting sleep for 0.5s
[Timer1] Woke up!
Slept for 0.5 seconds
```

```python
# Method 2: Using types.coroutine decorator (lower-level)
import types

@types.coroutine
def manual_coroutine():
    """A coroutine created without async keyword."""
    print("Manual coroutine: starting")
    yield from asyncio.sleep(0.3).__await__()
    print("Manual coroutine: done")
    return "manual result"

result = await manual_coroutine()
print(f"Result: {result}")
```

```
Manual coroutine: starting
Manual coroutine: done
Result: manual result
```

```python
# How Python expands async def to vanilla Python (conceptually)
# ================================================================

# ORIGINAL: async function using async/await syntax
# -------------------------------------------------
async def fetch_user(user_id: int) -> dict:
    print(f"Fetching user {user_id}")
    await asyncio.sleep(1)  # Simulate I/O
    return {"id": user_id, "name": "Alice"}


# EXPANDED: What Python conceptually creates under the hood
# ---------------------------------------------------------
import types

class _FetchUserCoroutine:
    """
    This is roughly what Python creates when you define an async function.
    It's a state machine that tracks where execution paused.
    """
    def __init__(self, user_id: int):
        self.user_id = user_id
        self._state = 0  # Track which "await" we're at
        self._result = None
        
    def send(self, value):
        """Resume execution, called by the event loop."""
        if self._state == 0:
            # First part: before first await
            print(f"Fetching user {self.user_id}")
            self._state = 1
            # Return the awaitable we need to wait for
            return asyncio.sleep(1)
        elif self._state == 1:
            # After first await completed, return final result
            self._state = 2
            raise StopIteration({"id": self.user_id, "name": "Alice"})
        else:
            raise StopIteration(self._result)
    
    def throw(self, exc):
        """Inject an exception into the coroutine."""
        raise exc
    
    def close(self):
        """Clean up the coroutine."""
        pass
    
    def __await__(self):
        return self
    
    def __iter__(self):
        return self
    
    def __next__(self):
        return self.send(None)

def fetch_user_vanilla(user_id: int):
    """This function returns our coroutine object."""
    return _FetchUserCoroutine(user_id)

# Demonstration: both work the same way!
print("=== Using async def ===")
result1 = await fetch_user(42)
print(f"Result: {result1}\n")

print("=== Conceptual expansion (simplified) ===")
# Note: This simplified version won't work exactly like real coroutines
# but shows the concept of state machine + iterator protocol
```

```
=== Using async def ===
Fetching user 42
Result: {'id': 42, 'name': 'Alice'}

=== Conceptual expansion (simplified) ===
```

#### Why Create Custom Awaitables?

| Use Case                  | Example                                   |
| ------------------------- | ----------------------------------------- |
| **Wrap async operations** | Add retry logic, timing, logging          |
| **Lazy evaluation**       | Defer computation until awaited           |
| **Interop**               | Bridge between sync/async code            |
| **Framework internals**   | How `asyncio.Future` works under the hood |

#### Key Insight

`async def` is **syntactic sugar** that creates a coroutine function. Under the hood, Python:

1. Creates a generator-like object
2. Implements the coroutine protocol (`__await__`, `send`, `throw`, `close`)
3. Returns a coroutine object when called

### Summary: Key asyncio Functions

| Function                          | Purpose                                      |
| --------------------------------- | -------------------------------------------- |
| `asyncio.run(coro)`               | Run a coroutine from sync code (entry point) |
| `await coro`                      | Wait for a coroutine to complete             |
| `asyncio.create_task(coro)`       | Schedule a coroutine as a Task               |
| `asyncio.gather(*coros)`          | Run multiple coroutines concurrently         |
| `asyncio.TaskGroup()`             | Structured concurrency (Python 3.11+)        |
| `asyncio.sleep(seconds)`          | Non-blocking sleep                           |
| `asyncio.wait_for(coro, timeout)` | Run with timeout                             |
| `asyncio.timeout(seconds)`        | Timeout context manager (3.11+)              |
| `asyncio.to_thread(func, *args)`  | Run blocking code in thread                  |

### Best Practices

1. **Don't mix sync and async** - Choose one paradigm per codebase section
2. **Use `async with` and `async for`** for proper resource management
3. **Prefer `TaskGroup`** over raw `create_task` for error handling
4. **Use semaphores** for rate limiting concurrent operations
5. **Handle `CancelledError`** properly for graceful shutdown
6. **Avoid blocking calls** in async code - use `to_thread()` if needed
7. **Use `gather(return_exceptions=True)`** to handle partial failures
