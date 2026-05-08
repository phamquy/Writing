---
description: >-
  Similar concepts to Go channel in python, inter thread/process communication
  in python?
icon: pipe-valve
---

# Go channels in Python?

Go’s **channels** are built-in primitives for communication between goroutines.\
Python doesn’t have channels built into the language, but it has **similar constructs** in its standard library and ecosystem.

***

## 🔑 Channel-like Constructs in Python

### 1. `queue.Queue` (thread-safe, blocking)

* From `queue` module.
* Designed for communication between **threads**.
* Supports blocking `put()` and `get()`.

```python
import threading, queue

q = queue.Queue()

def worker():
    while True:
        item = q.get()
        if item is None:  # poison pill to stop
            break
        print(f"Processed {item}")
        q.task_done()

t = threading.Thread(target=worker)
t.start()

q.put(1)
q.put(2)
q.put(None)  # stop worker
t.join()
```

✅ Closest to Go channels for threads.

***

### 2. `multiprocessing.Queue`

* From `multiprocessing`.
* Works like `queue.Queue`, but for **separate processes**.

```python
from multiprocessing import Process, Queue

def worker(q):
    item = q.get()
    print(f"Got {item}")

q = Queue()
p = Process(target=worker, args=(q,))
p.start()
q.put("Hello from parent")
p.join()
```

***

### 3. `asyncio.Queue` (async/await world)

* Non-blocking channel-like structure for **coroutines**.
* Perfect for async tasks (like Go’s goroutines).

```python
import asyncio

async def producer(q):
    for i in range(3):
        await q.put(i)
    await q.put(None)  # signal end

async def consumer(q):
    while True:
        item = await q.get()
        if item is None:
            break
        print(f"Consumed {item}")

async def main():
    q = asyncio.Queue()
    await asyncio.gather(producer(q), consumer(q))

asyncio.run(main())
```

✅ This is the **closest match to Go’s channels** in Python (but async-based).

***

### 4. Third-party: `janus`

* Provides a **synchronous + async queue bridge**, handy when mixing threads and asyncio.

```bash
pip install janus
```

```python
import asyncio, janus

async def async_consumer(q):
    while True:
        item = await q.async_q.get()
        if item is None:
            break
        print(f"Consumed {item}")

async def main():
    q = janus.Queue()
    q.sync_q.put(1)   # from a thread or sync code
    q.sync_q.put(None)
    await async_consumer(q)

asyncio.run(main())
```

***

## ✅ Summary

* **Thread communication** → `queue.Queue`
* **Process communication** → `multiprocessing.Queue`
* **Async coroutines (closest to Go channels)** → `asyncio.Queue`
* **Hybrid (thread + async)** → `janus`

So yes — Python **does have channel-like constructs**, but depending on whether you’re working with threads, processes, or async coroutines, you’ll pick a different one.
