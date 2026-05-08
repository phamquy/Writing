---
icon: arrow-rotate-left
---

# How Many Event Loops Does asyncio Maintain Internally?

#### ✅ Default Behavior

By **default**, Python’s `asyncio` maintains **one event loop per thread**.

* In **single-threaded programs** (which is most common), there is **only one event loop**.
* You can create additional loops manually in other threads.

***

#### 🔧 Main Concepts

| Concept             | Details                                                                    |
| ------------------- | -------------------------------------------------------------------------- |
| **Main Event Loop** | Created and run in the main thread (`asyncio.run()` or `get_event_loop()`) |
| **Thread-local**    | Each thread can have its own separate event loop                           |
| **Not Shared**      | Event loops are **not shared** across threads                              |

***

#### 🧪 Example: Single Event Loop

```python
import asyncio

async def hello():
    print("Hello from", asyncio.get_running_loop())

asyncio.run(hello())
```

✅ Output shows one event loop, created and run internally by `asyncio.run()`.

***

#### 🧪 Example: One Event Loop per Thread

```python
import asyncio
import threading

def thread_func():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(asyncio.sleep(1))
    loop.close()

t1 = threading.Thread(target=thread_func)
t2 = threading.Thread(target=thread_func)

t1.start()
t2.start()
t1.join()
t2.join()
```

✅ This creates **two threads**, each with **its own event loop**.

***

#### ⚠️ Rules to Remember

| Rule | Description                                                    |
| ---- | -------------------------------------------------------------- |
| 🔄 1 | Only one event loop can be active per thread                   |
| 🚫 2 | You cannot use the same event loop across threads              |
| ⚙️ 3 | Use `asyncio.run()` or `get_running_loop()` in the main thread |

***

#### 🧠 Summary

* **One event loop per thread** is the design.
* `asyncio` maintains **a single event loop internally in main thread**, unless you explicitly create more.
* For most applications, you only need **one event loop**.
