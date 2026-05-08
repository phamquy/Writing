---
icon: reel
---

# Threading.Event vs threading.Condition in Python

Both `Event` and `Condition` are synchronization primitives used in multithreading — but they serve different purposes and usage patterns.

***

### 🧠 TL;DR

| Feature            | `threading.Event`                     | `threading.Condition`                               |
| ------------------ | ------------------------------------- | --------------------------------------------------- |
| Type               | Simple flag/signal                    | General wait-notify with locking                    |
| Wait on condition? | Waits for flag to be set              | Waits for arbitrary condition inside a `with` block |
| Notifies           | All waiters (like broadcast)          | One or all waiters (selective)                      |
| Lock associated?   | ❌ No lock                             | ✅ Requires associated lock                          |
| Typical use case   | Signal or trigger (start, stop, done) | Wait until a shared resource state changes          |

***

### 🔔 `threading.Event`

* Has a single internal flag (`True` or `False`)
* Threads can wait until the flag is set
* Calling `set()` unblocks **all** waiting threads
* No need for a lock or shared state

#### ✅ Example

```python
event = threading.Event()

# In worker
event.wait()  # Blocks until event is set
# Continue execution

# In controller
event.set()   # Unblocks all waiting threads
```

Use when: you want to **signal** between threads (like a "go" or "stop" flag).

***

### 🔁 `threading.Condition`

* Used with a **Lock** (`with condition:`)
* Threads wait for a **specific condition** to be true (not just a flag)
* You must manually `notify()` or `notify_all()` when the condition changes
* Ideal when the condition is based on **shared state**

#### ✅ Example

```python
condition = threading.Condition()
queue = []

# Producer
with condition:
    queue.append("item")
    condition.notify()

# Consumer
with condition:
    while not queue:
        condition.wait()
    item = queue.pop()
```

Use when: you want threads to wait based on **complex state changes**, like a queue being non-empty.

***

### 🧪 When to Use What?

| Scenario                                    | Use                                 |
| ------------------------------------------- | ----------------------------------- |
| Wait for "start signal" from another thread | `Event`                             |
| Wait until a shared list is not empty       | `Condition`                         |
| Broadcast a state change to all workers     | `Event` or `Condition.notify_all()` |
| Only one thread should proceed              | `Condition.notify()`                |

***

#### 🧠 Summary Analogy

| Concept     | `Event`                | `Condition`                         |
| ----------- | ---------------------- | ----------------------------------- |
| Like a      | Light switch           | Conversation with conditions        |
| Signal type | Binary flag            | Arbitrary condition-based signaling |
| Wait style  | Wait until flag is set | Wait until condition becomes true   |

***
