---
description: Demystifying `await`
icon: gears
---

# Understanding Context and Waker in Rust Futures

#### 1. The role of `cx` in `Future::poll`

"The `cx` parameter and its `Context` type are the key to how a runtime actually knows when to check any given future while still being lazy.

unpacks as follows:

**`Future::poll` signature**

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
```

* `self: Pin<&mut Self>` – gives the runtime access to the future without moving it.
* `cx: &mut Context<'_>` – provides runtime-specific information, especially how to wake this future when it can make progress.
* Returns `Poll::Ready(val)` or `Poll::Pending`.

**Why futures are lazy**

* Creating a future does nothing; it only runs when polled.
* The runtime decides when to call `poll`.

**`Context` (`cx`) and `Waker`**

* `Context` contains a `Waker`:

```rust
struct Context<'a> {
    waker: &'a Waker,
}
```

* The future uses `waker()` to register **how it should be woken up later**.
* When `poll` returns `Poll::Pending`, the future can store the `Waker` somewhere (e.g., I/O driver) to be called later.
* Later, when the future can make progress, the `Waker` is called, signaling the runtime to poll the future again.

**Lazy and responsive execution**

1. **Lazy**: Future does nothing until polled.
2. **Responsive**: Future notifies runtime via `cx.waker()` when ready.
3. No busy loops are needed; the runtime sleeps until a `Waker` triggers polling.

**Illustration:**

```
Future::poll(cx) -> Poll::Pending
   |
   |-- registers cx.waker somewhere (e.g., OS socket)
   v
IO event occurs -> Waker wakes runtime
   |
Runtime polls future again -> Poll::Ready(val) or Poll::Pending
```

**Bottom line:** `cx` is the bridge between the future and the runtime scheduler, enabling lazy yet event-driven execution.

***

#### 2. Who calls `waker()`?

The `Waker` is just a handle to tell the runtime:

> “Hey, this future is ready to be polled again.”

It **does not have to be called by the runtime’s main thread**. Usually, it’s called by:

1. **OS or external events:**
   * Example: TCP read becomes ready; OS signals I/O driver.
   * I/O driver calls `waker.wake()`.
2. **Other threads or tasks:**
   * Shared resource becomes available.
   * Background thread signals the future via `Waker`.
3. **Timer events:**
   * Timer expires; timer thread calls `waker.wake()`.

**Important:**

* The runtime thread may be busy polling other futures.
* Wake-up happens **from outside**, then the runtime thread polls the future again.

**Illustration:**

```
+-----------------+        +-----------------+
| Runtime thread  |        | IO/Timer thread |
| Polls future F  |<-------| Calls waker()   |
|                 |        |                 |
+-----------------+        +-----------------+
```

* F is initially pending.
* External event triggers wake.
* Runtime thread polls F again.

**Key insight:** `waker.wake()` can be called from any thread; the executor thread just responds by polling ready futures.
