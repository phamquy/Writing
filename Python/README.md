---
icon: microchip
---

# What Happens If a Python Program Spawns More Processes Than CPU Cores?

Spawning more processes than available CPU cores is perfectly valid in Python — **but it comes with trade-offs**. Here's what happens and what you should consider:

***

#### ✅ It Works — But With Limits

Python’s `multiprocessing` module can spawn any number of processes, regardless of the number of physical or logical CPU cores.

For example:

```python
import multiprocessing

def worker(i):
    print(f"Worker {i} starting")
    while True:
        pass  # Simulate heavy CPU work

if __name__ == "__main__":
    for i in range(20):  # More than most CPU core counts
        multiprocessing.Process(target=worker, args=(i,)).start()
```

***

#### 🔄 When Processes > CPU Cores

| Aspect                | What Happens                                      |
| --------------------- | ------------------------------------------------- |
| **OS Scheduling**     | The OS time-slices processes onto available cores |
| **Context Switching** | More frequent, adds overhead                      |
| **CPU-bound Tasks**   | Slower overall performance, contention for CPU    |
| **IO-bound Tasks**    | Still fine, processes wait on IO, not CPU         |
| **Power Usage**       | Higher power usage and CPU temperature            |

***

#### 🧠 Why Spawn More Processes?

* **Good for IO-bound tasks**: Many processes may be idle (waiting for network, disk, etc.), so oversubscribing can help utilize the CPU better.
* **Bad for CPU-bound tasks**: If all processes are fighting for CPU time, performance drops.

***

#### 🧪 Observing This in Action

You can monitor this with:

```bash
htop        # On Unix-based systems
Task Manager  # On Windows
```

You’ll see CPU cores max out, with context switching increasing as more processes are added.

***

#### 🧮 Best Practice

| Workload Type | Ideal Strategy                              |
| ------------- | ------------------------------------------- |
| CPU-bound     | Use `multiprocessing.cpu_count()` processes |
| IO-bound      | You can spawn **many more** than CPU count  |
| Mixed         | Profile and tune based on task behavior     |

Example:

```python
# Optimal for CPU-bound:
pool = multiprocessing.Pool(processes=multiprocessing.cpu_count())
```

***

#### ⚠️ Gotchas

* Too many processes can **degrade performance** due to context-switching overhead.
* Memory usage increases with more processes (each has its own memory space).
* Threads would be worse for CPU-bound tasks in CPython due to the **GIL**.

***

#### ✅ Summary

* You **can** spawn more processes than CPU cores.
* It's **fine for IO-bound tasks**.
* It **hurts CPU-bound workloads** due to time-sharing.
* Always **profile and test** to find the sweet spot for your use case.
