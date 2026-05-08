---
icon: xmarks-lines
---

# What To Do When a threading.Barrier Is Broken

A **broken barrier** means that one or more threads failed to reach the barrier, and the synchronization cannot continue normally. You need to detect and handle this to avoid deadlocks or inconsistent state.

***

### 🚨 Why Does a Barrier Break?

A `Barrier` breaks in these situations:

| Scenario                                              | Outcome                                 |
| ----------------------------------------------------- | --------------------------------------- |
| A thread raises an exception before calling `.wait()` | Barrier can't be satisfied              |
| A thread times out while waiting                      | `BrokenBarrierError` is raised          |
| Someone explicitly calls `.abort()`                   | Barrier is broken manually              |
| `.reset()` is called while waiting                    | Barrier resets and breaks current state |

***

### 🧪 How to Handle a Broken Barrier

#### ✅ Always Wrap `.wait()` in Try/Except

```python
try:
    barrier.wait()
except threading.BrokenBarrierError:
    print(f"⚠️ Thread-{threading.get_ident()} detected broken barrier!")
    # Optionally: clean up or exit
```

***

### ✅ Example: Graceful Fallback When Barrier Breaks

```python
import threading
import time

barrier = threading.Barrier(3)

def worker(i):
    try:
        if i == 1:
            raise Exception("Simulated failure")  # Thread 1 crashes
        print(f"Thread-{i} is ready")
        time.sleep(0.5)
        barrier.wait()
        print(f"✅ Thread-{i} passed the barrier!")
    except threading.BrokenBarrierError:
        print(f"❌ Thread-{i} could not pass — barrier broken.")
    except Exception as e:
        print(f"💥 Thread-{i} failed: {e}")
        barrier.abort()  # Explicitly abort the barrier

threads = [threading.Thread(target=worker, args=(i,)) for i in range(3)]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

***

### 🧹 What You Should Do When Barrier Is Broken

| Action                          | Why It's Important                           |
| ------------------------------- | -------------------------------------------- |
| Catch `BrokenBarrierError`      | Avoid crashing or hanging the program        |
| Log or report it                | Helps diagnose why synchronization failed    |
| Use `barrier.reset()` if needed | Try again after cleanup or reinit            |
| Fallback gracefully             | Continue work with degraded behavior if safe |

***

### 🧠 Summary

* **Always catch `BrokenBarrierError`** to prevent crashes
* Use `.abort()` if a thread fails and the barrier must break
* Use `.reset()` if you want to **reuse** the barrier after fixing the problem

***
