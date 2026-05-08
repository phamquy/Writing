---
icon: arrow-progress
---

# Python Multiprocessing (Fork vs. Spawn)

When you hit a `TypeError: cannot pickle 'mmap.mmap' object` on macOS or Windows in Python multi process application, it can be confusing. Why does a child process need to "pickle" something that's supposed to be shared memory? The answer lies in how different operating systems create new processes.

### Two Ways to Start a Process

Python's `multiprocessing` module uses different **start methods** depending on the OS:

| Method      | Default On     | How It Works                                                                        |
| ----------- | -------------- | ----------------------------------------------------------------------------------- |
| **`fork`**  | Linux          | Creates an **exact clone** of the parent's memory. The child "inherits" everything. |
| **`spawn`** | macOS, Windows | Starts a **brand new Python interpreter** and requires serialization of arguments.  |

***

### How `fork` Works (Linux)

```
                    Parent Process
                    ┌───────────────────────────┐
                    │  Memory:                  │
                    │  - globals, variables     │
                    │  - mmap object (handle)   │
                    │  - open files             │
                    └─────────────┬─────────────┘
                                  │
                              fork()
                                  │
              ┌───────────────────┴───────────────────┐
              │                                       │
              ▼                                       ▼
     Parent (continues)                     Child (clone of parent)
     ┌────────────────────┐                 ┌────────────────────┐
     │  Same memory       │                 │  Copy of memory    │
     │  (mmap still works)│                 │  (mmap still works)│
     └────────────────────┘                 └────────────────────┘
```

When the OS **forks** a process:

1. The child gets an **exact copy** of the parent's address space (using Copy-on-Write for efficiency).
2. Open file descriptors, memory-mapped regions, and most kernel resources are **inherited**.
3. No serialization is needed—the child already "has" everything.

This is why `mmap` works seamlessly on Linux: the child inherits the mapping directly from the parent.

***

### How `spawn` Works (macOS/Windows)

```
                    Parent Process
                    ┌───────────────────────────┐
                    │  Memory:                  │
                    │  - globals, variables     │
                    │  - mmap object (handle)   │
                    └─────────────┬─────────────┘
                                  │
                  pickle(target, args) ────────┐
                                  │            │  (pipe)
                              spawn()          │
                                  │            │
                                  ▼            ▼
                           New Python Process
                           ┌────────────────────┐
                           │  Fresh memory      │
                           │  - Empty state     │
                           │  - Reads from pipe │
                           │  - unpickle(...)   │
                           └────────────────────┘
```

When the OS **spawns** a process:

1. A completely **new Python interpreter** is started from scratch.
2. The child has **no knowledge** of the parent's memory, file handles, or kernel resources.
3. Python must **serialize (pickle)** the target function and its arguments, send them over a pipe, and **deserialize** them in the child.

This is where the problem arises: an `mmap` object is just a pointer/handle to a kernel-managed memory region. Pickling it would only capture a meaningless address—the child process has no way to use it.

***

### The Solution: Named Resources

Python provides primitives that work around this limitation by using **named kernel resources**:

| Object                  | How It Works Across `spawn`                             |
| ----------------------- | ------------------------------------------------------- |
| `multiprocessing.Value` | Creates a named shared memory segment; passes the name. |
| `multiprocessing.Array` | Same as `Value`, for arrays.                            |
| `SharedMemory`          | Explicitly named; child attaches by name.               |
| `Lock`, `Queue`, `Pipe` | Use named semaphores/FIFOs managed by the kernel.       |

```python
from multiprocessing import Process
from multiprocessing.shared_memory import SharedMemory

def worker(shm_name):
    # Attach to existing shared memory BY NAME
    existing_shm = SharedMemory(name=shm_name)
    print(bytes(existing_shm.buf[:5]))
    existing_shm.close()

if __name__ == '__main__':
    shm = SharedMemory(create=True, size=1024)
    shm.buf[:5] = b"Hello"
    
    # Pass the NAME (a pickleable string), not the object
    p = Process(target=worker, args=(shm.name,))
    p.start()
    p.join()
    
    shm.close()
    shm.unlink()
```

***

### Key Takeaways

1. **`fork` clones memory**; `spawn` starts fresh.
2. On macOS/Windows, arguments must be **pickleable** because they are sent through a pipe to a new interpreter.
3. **Raw `mmap` objects cannot be pickled**—they are just local handles to kernel resources.
4. Use **`multiprocessing.shared_memory.SharedMemory`** for cross-process byte sharing on all platforms.
5. Always **pass names (strings)** instead of kernel objects when using `spawn`.

***

### When to Use What

| Use Case                         | Recommended Approach                         |
| -------------------------------- | -------------------------------------------- |
| Share simple values (int, float) | `multiprocessing.Value`                      |
| Share arrays of C-types          | `multiprocessing.Array`                      |
| Share raw bytes (flexible)       | `multiprocessing.shared_memory.SharedMemory` |
| Cross-platform compatibility     | Always assume `spawn` behavior               |
