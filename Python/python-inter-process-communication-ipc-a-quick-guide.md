---
icon: bars-staggered
---

# Python Inter-Process Communication (IPC): A Quick Guide

Processes in Python have **isolated memory spaces** by design. Unlike threads, which share the same heap, processes cannot directly access each other's variables. This guide covers all the mechanisms Python provides to move data between processes.

***

### Table of Contents

1. Why IPC is Needed
2. Message Passing: Queue and Pipe
3. Shared Memory: Value, Array, and SharedMemory
4. Manager: Shared Python Objects
5. Synchronization Primitives
6. fork vs. spawn: The Pickling Requirement
7. Comparison Table

***

### Why IPC is Needed

```
┌─────────────────────┐      ┌─────────────────────┐
│   Process A         │      │   Process B         │
│   ┌───────────────┐ │      │ ┌───────────────┐   │
│   │  Memory Space │ │  ??  │ │  Memory Space │   │
│   │  x = 42       │ │◄────►│ │  ???          │   │
│   └───────────────┘ │      │ └───────────────┘   │
└─────────────────────┘      └─────────────────────┘
```

Each Python process runs in its own **virtual address space**. Variables, objects, and file handles are completely private. To share data, you need explicit IPC mechanisms.

***

### Message Passing: Queue and Pipe

#### `multiprocessing.Queue`

A **thread-safe, process-safe FIFO queue**. Best for producer-consumer patterns with multiple workers.

```python
from multiprocessing import Process, Queue

def producer(q):
    q.put("Hello from producer!")

def consumer(q):
    print(q.get())

if __name__ == '__main__':
    q = Queue()
    p1 = Process(target=producer, args=(q,))
    p2 = Process(target=consumer, args=(q,))
    p1.start(); p2.start()
    p1.join(); p2.join()
```

**Under the hood**: Built on pipes + locks/semaphores.

***

#### `multiprocessing.Pipe`

A **direct connection between two endpoints**. Faster than Queue for 1:1 communication.

```python
from multiprocessing import Process, Pipe

def child(conn):
    conn.send("Message from child")
    print(f"Child got: {conn.recv()}")

if __name__ == '__main__':
    parent_conn, child_conn = Pipe()
    p = Process(target=child, args=(child_conn,))
    p.start()
    
    print(f"Parent got: {parent_conn.recv()}")
    parent_conn.send("Reply from parent")
    p.join()
```

**Duplex vs Simplex**: By default, both ends can send/receive. Use `Pipe(duplex=False)` for one-way.

***

### Shared Memory: Value, Array, and SharedMemory

#### `multiprocessing.Value` and `multiprocessing.Array`

High-level wrappers for sharing **C-style typed data** in shared memory.

```python
from multiprocessing import Process, Value, Array

def worker(num, arr):
    num.value = 3.14
    for i in range(len(arr)):
        arr[i] *= 2

if __name__ == '__main__':
    num = Value('d', 0.0)       # 'd' = double (float)
    arr = Array('i', [1, 2, 3]) # 'i' = int

    p = Process(target=worker, args=(num, arr))
    p.start()
    p.join()

    print(num.value)  # 3.14
    print(arr[:])     # [2, 4, 6]
```

**Supported Type Codes**

| Code          | C Type               | Python Type |
| ------------- | -------------------- | ----------- |
| `'b'` / `'B'` | signed/unsigned char | int         |
| `'h'` / `'H'` | short                | int         |
| `'i'` / `'I'` | int                  | int         |
| `'l'` / `'L'` | long                 | int         |
| `'q'` / `'Q'` | long long            | int         |
| `'f'`         | float                | float       |
| `'d'`         | double               | float       |

***

#### `multiprocessing.shared_memory.SharedMemory` (Python 3.8+)

Low-level **raw byte buffer** for maximum flexibility and performance.

```python
from multiprocessing import Process
from multiprocessing.shared_memory import SharedMemory

def worker(shm_name):
    shm = SharedMemory(name=shm_name)
    print(bytes(shm.buf[:5]))
    shm.buf[:5] = b"CHILD"
    shm.close()

if __name__ == '__main__':
    shm = SharedMemory(create=True, size=1024)
    shm.buf[:5] = b"START"

    p = Process(target=worker, args=(shm.name,))
    p.start()
    p.join()

    print(bytes(shm.buf[:5]))  # b'CHILD'
    shm.close()
    shm.unlink()  # Clean up OS resource
```

**Key Point**: Pass the **name** (a string), not the object. This works with `spawn`.

***

### Manager: Shared Python Objects

For **complex Python types** (dicts, lists, Namespace), use a `Manager`. It runs a server process that proxies access.

```python
from multiprocessing import Process, Manager

def worker(d, l):
    d["key"] = "value"
    l.append(42)

if __name__ == '__main__':
    with Manager() as manager:
        d = manager.dict()
        l = manager.list()

        p = Process(target=worker, args=(d, l))
        p.start()
        p.join()

        print(d)  # {'key': 'value'}
        print(l)  # [42]
```

**Trade-off**: More flexible but **slower** than Value/Array (involves IPC to the manager server).

***

### Synchronization Primitives

Shared memory requires synchronization to avoid race conditions.

| Primitive   | Purpose                                                   |
| ----------- | --------------------------------------------------------- |
| `Lock`      | Mutual exclusion (only one process at a time)             |
| `RLock`     | Re-entrant lock (same process can acquire multiple times) |
| `Semaphore` | Limit concurrent access (e.g., pool of N resources)       |
| `Event`     | Binary signal (set/clear/wait)                            |
| `Condition` | Wait for complex conditions with notify                   |
| `Barrier`   | Wait for N processes to reach a sync point                |

#### Example: Lock with Shared Value

```python
from multiprocessing import Process, Value, Lock

def increment(counter, lock):
    for _ in range(10000):
        with lock:
            counter.value += 1

if __name__ == '__main__':
    counter = Value('i', 0)
    lock = Lock()

    procs = [Process(target=increment, args=(counter, lock)) for _ in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()

    print(counter.value)  # 40000 (correct with lock)
```

***

### fork vs. spawn: The Pickling Requirement

#### Why This Matters

| Start Method | Default On     | Behavior                                             |
| ------------ | -------------- | ---------------------------------------------------- |
| `fork`       | Linux          | Child copies parent's memory. Objects are inherited. |
| `spawn`      | macOS, Windows | New interpreter. Arguments must be **pickled**.      |

On `spawn`, when you pass an argument to a child process:

1. Python **serializes (pickles)** the object.
2. Sends it through a pipe to the new process.
3. The child **deserializes** it.

**Problem**: Some objects (like `mmap.mmap`) can't be pickled—they're just handles to kernel resources.

**Solution**: Use **named resources** (`SharedMemory`, `Lock`, etc.) and pass the **name** instead.

***

### Comparison Table

| Mechanism       | Speed     | Data Types     | Ease of Use | Best For                   |
| --------------- | --------- | -------------- | ----------- | -------------------------- |
| `Queue`         | Medium    | Any pickleable | Easy        | Multi-producer/consumer    |
| `Pipe`          | Fast      | Any pickleable | Easy        | 1:1 communication          |
| `Value`/`Array` | Very Fast | C-types only   | Medium      | Shared counters/arrays     |
| `SharedMemory`  | Fastest   | Raw bytes      | Hard        | High-performance, flexible |
| `Manager`       | Slow      | Python objects | Easy        | Complex data structures    |

***

### Best Practices

1. **Always use `if __name__ == '__main__':`** on macOS/Windows.
2. **Always call `p.join()`** to prevent resource cleanup before children finish.
3. **Use Locks** with shared memory to prevent race conditions.
4. **Prefer `Queue`** for most use cases—it's safe and simple.
5. **Use `SharedMemory`** when you need maximum performance with raw bytes.
6. **Pass names, not objects** when writing cross-platform code (assume `spawn`).
