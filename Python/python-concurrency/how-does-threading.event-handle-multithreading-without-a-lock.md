---
icon: lock-hashtag
---

# How Does threading.Event Handle Multithreading Without a Lock?

`threading.Event` **doesn't need an explicit lock** like `threading.Condition`, yet it is fully **thread-safe**. Here’s how and why:

***

#### 🔐 Internal Locking

Even though you don’t use a `Lock` directly, `Event` **internally** uses synchronization primitives (like a `Condition` or low-level atomic flags) to manage safe access across threads.

> ✅ So while you don't need to manage locks, the thread safety is **built-in**.

***

#### 🔁 How It Works

* `event.set()` sets a flag to `True`, and **unblocks all threads** waiting on `event.wait()`.
* `event.clear()` resets the flag to `False`, so future `.wait()` calls will **block** again.
* `event.wait()` checks the flag:
  * If set, it returns immediately.
  * If not, it blocks until another thread calls `set()`.

This works across **multiple threads** because of the thread-safe internal state management.

***

#### 🧪 Example: Multiple Waiters

```python
import threading
import time

event = threading.Event()

def waiter(id):
    print(f"Thread-{id} waiting for event...")
    event.wait()
    print(f"✅ Thread-{id} proceeding after event is set.")

# Launch 3 waiting threads
for i in range(3):
    threading.Thread(target=waiter, args=(i,)).start()

time.sleep(2)
print("🚀 Main thread setting the event")
event.set()  # All 3 threads will resume execution
```

Even with **no locks used by the programmer**, all threads are properly synchronized and resume after the `set()`.

***

#### ✅ When to Use `Event`

Use `Event` when:

* You want to **broadcast a simple signal** ("start", "stop", "done")
* Threads don't need to check complex shared state
* You want to avoid the extra complexity of `Lock` or `Condition`

***

#### 🚫 When NOT to Use `Event`

Don't use `Event` when:

* Threads need to wait for **specific shared state changes** (like a queue having data)
* You need to control which thread proceeds (`Event` resumes **all** waiters)

***

#### 🧠 Summary

| Feature                 | `threading.Event`                       |
| ----------------------- | --------------------------------------- |
| Explicit lock required? | ❌ No                                    |
| Thread-safe?            | ✅ Yes (handled internally)              |
| Waits on?               | Simple flag (`True` or `False`)         |
| Best for                | Broadcast signaling (e.g. "go", "stop") |

***
