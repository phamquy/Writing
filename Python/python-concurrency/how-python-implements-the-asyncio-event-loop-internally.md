---
icon: arrow-rotate-left
---

# How Python Implements the asyncio Event Loop Internally

### 🔍 How Python Implements the `asyncio` Event Loop Internally — Does It Use `epoll`?

Yes — **Python's `asyncio` event loop** is built on top of **OS-level I/O multiplexing mechanisms** like `epoll`, `kqueue`, or `Select`.

***

### ⚙️ Event Loop Core: Built on `selectors` Module

Internally, Python's `asyncio` uses the `selectors` module, which picks the most efficient mechanism available for your OS:

| OS      | Mechanism Used        | Description                        |
| ------- | --------------------- | ---------------------------------- |
| Linux   | `epoll`               | Scalable and efficient I/O polling |
| macOS   | `kqueue`              | BSD’s native I/O multiplexer       |
| Windows | `Select` / `Proactor` | Windows I/O strategies             |

#### Under the Hood:

```python
import selectors
selector = selectors.DefaultSelector()
print(type(selector))
```

On Linux, this prints:

```
<class 'selectors.EpollSelector'>
```

***

### 📦 asyncio Loop Architecture

Here's how the event loop works in `asyncio`:

1. **Create and register events**:
   * Sockets, file descriptors, etc., are registered with the OS-level selector.
2. **Poll for readiness**:
   * Uses `epoll.select(timeout)` or equivalent to wait for I/O readiness.
3. **Dispatch callbacks**:
   * When events are ready, the loop dispatches them via scheduled coroutines.

#### Diagram:

```
Coroutine --> Future/Task --> Event Loop --> Selector (epoll/kqueue/etc)
```

***

### 🧪 Example of Using `selectors` (Low-Level):

```python
import selectors
import socket

sel = selectors.DefaultSelector()

def accept(sock):
    conn, addr = sock.accept()
    print("Accepted", addr)
    conn.setblocking(False)
    sel.register(conn, selectors.EVENT_READ, read)

def read(conn):
    data = conn.recv(1000)
    if data:
        print("Received:", data.decode())
    else:
        sel.unregister(conn)
        conn.close()

sock = socket.socket()
sock.bind(('localhost', 12345))
sock.listen()
sock.setblocking(False)
sel.register(sock, selectors.EVENT_READ, accept)

while True:
    for key, _ in sel.select():
        callback = key.data
        callback(key.fileobj)
```

***

### 🔁 In `asyncio`: You Don’t See This

But behind the scenes, `asyncio` is doing this type of I/O multiplexing for you, handling:

* **Tasks scheduling**
* **Timer callbacks**
* **Socket readiness**
* **Future and coroutine state**

***

### ✅ Summary

| Feature               | asyncio Behavior                           |
| --------------------- | ------------------------------------------ |
| Uses `epoll` on Linux | ✅ Yes — via `selectors.EpollSelector`      |
| Manual I/O polling    | ❌ Not needed (unless using low-level APIs) |
| Multi-platform        | ✅ Uses best available mechanism per OS     |

***

### 🔄 Walkthrough: How a Coroutine Is Registered and Resumed in Python's `asyncio` Event Loop

Let’s dive deep into **how `asyncio` schedules and resumes coroutines** behind the scenes — from creation to completion.

***

### 🧱 Step-by-Step: Coroutine Lifecycle

We'll walk through this example:

```python
import asyncio

async def greet():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

asyncio.run(greet())
```

***

#### Step 1️⃣: `asyncio.run()` Creates the Event Loop

* `asyncio.run()` is a convenience function that:
  1. Creates a new event loop.
  2. Schedules the coroutine (`greet`) as a `Task`.
  3. Runs the loop until the task is complete.
  4. Closes the loop.

```python
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
loop.run_until_complete(greet())  # This schedules and runs the coroutine
```

***

#### Step 2️⃣: `greet()` Becomes a Task

* `greet()` is turned into a `Task` using `asyncio.ensure_future()` or `loop.create_task()`.
* A `Task` wraps a coroutine and allows it to be scheduled by the event loop.

```python
task = loop.create_task(greet())  # task is now managed by the loop
```

***

#### Step 3️⃣: The Task Is Scheduled

* The event loop now holds the task in its **ready queue**.
* The loop repeatedly:
  * Picks a task that’s ready to run.
  * Calls `send(None)` on its coroutine.
  * Waits if it yields an awaitable.

***

#### Step 4️⃣: Coroutine Hits an `await`

Inside `greet()`:

```python
await asyncio.sleep(1)
```

* `asyncio.sleep()` returns a `Future` that completes after 1 second.
* The coroutine yields control (`await`) and **pauses**.

Internally:

* `sleep()` schedules a **callback** (e.g., `loop.call_later(1, ...)`) to **resume the task** after 1 second.

***

#### Step 5️⃣: Event Loop Pauses

* The event loop uses a **selector** (`epoll`, `kqueue`, etc.) to wait for:
  * Timers to expire (like `sleep`)
  * Sockets to become readable/writable
* When the timer expires, the `Future` becomes done → the coroutine is resumed.

***

#### Step 6️⃣: Task Is Resumed

*   The callback wakes the `Task`:

    ```python
    task._step()  # calls coroutine.send(result)
    ```
* Execution continues at the point after `await asyncio.sleep(1)`.

***

#### Step 7️⃣: Coroutine Completes

* Once the coroutine finishes (no more `await`), it returns a result or raises.
* The task is marked **done**, and the event loop removes it.

***

### 🔄 Visual Summary

```
                [ Task Created ]
                      ↓
          [ Scheduled in Event Loop ]
                      ↓
             ┌───────────────────┐
             │ coroutine.send()  │
             └───────────────────┘
                      ↓
             [ await something ]
                      ↓
            ┌────────────────────┐
            │ Yielded Future     │
            │ (e.g., sleep)      │
            └────────────────────┘
                      ↓
        [ Event Loop waits for readiness ]
                      ↓
             [ Future marked done ]
                      ↓
             [ Task resumed via .send() ]
                      ↓
              [ Coroutine continues ]
                      ↓
             [ Coroutine completes ]
```

***

### ✅ Key Concepts Recap

| Concept        | Description                                        |
| -------------- | -------------------------------------------------- |
| `Task`         | Wraps coroutine and manages its lifecycle          |
| `await`        | Yields control to event loop until result is ready |
| `Future`       | A placeholder for a result that’s set later        |
| `selector`     | Uses `epoll`, `kqueue`, or similar under the hood  |
| `call_later()` | Schedules callbacks after a delay                  |

see the **source code paths inside `asyncio`** (like `Task._step`, `Future.set_result`, etc.)!
