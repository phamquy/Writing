---
icon: face-thinking
---

# Lock's Condition — What actually happene ?

### 1. What is a Condition?

A `Condition` is a **per-lock wait queue**. It replaces `Object.wait()` / `notify()` with named queues attached to a specific lock, so you can wake up exactly the right threads.

#### Core 3 methods

```java
condition.await();      // release lock + sleep until signalled  (like Object.wait)
condition.signal();     // wake ONE thread waiting on this condition (like Object.notify)
condition.signalAll();  // wake ALL threads waiting on this condition
```

`await()` **atomically** releases the lock and suspends. When it wakes up, it re-acquires the lock before returning.

***

### 2. Classic Example — Bounded Buffer (Producer / Consumer)

Two conditions let you wake _only_ producers or _only_ consumers:

```java
public class BoundedBuffer<T> {

    private final Lock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition(); // producers wait here
    private final Condition notEmpty = lock.newCondition(); // consumers wait here

    private final Queue<T> queue;
    private final int capacity;

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();       // buffer full → wait
            }
            queue.add(item);
            notEmpty.signal();         // tell a consumer: data is ready
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();      // buffer empty → wait
            }
            T item = queue.poll();
            notFull.signal();          // tell a producer: space is available
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

#### Why `while` instead of `if`?

Always re-check the condition in a loop due to **spurious wakeups** — a thread can wake up without being signalled (JVM / OS artifact).

```java
// WRONG
if (queue.isEmpty()) condition.await();

// CORRECT
while (queue.isEmpty()) condition.await();
```

***

### 3. `await` variants

```java
condition.await();                        // wait indefinitely
condition.await(2, TimeUnit.SECONDS);     // wait with timeout → returns false if timed out
condition.awaitUninterruptibly();         // ignore Thread.interrupt() while waiting
condition.awaitUntil(new Date(...));      // wait until a specific wall-clock time
```

***

### 4. Why Condition must come from a lock

A `Condition` **cannot be created independently** — it is permanently bound to one specific lock. When `await()` is called, it needs to:

1. Release the lock you currently hold
2. Suspend the thread
3. Re-acquire the same lock before returning

Without knowing which lock to release and re-acquire, `await()` cannot perform the atomic handoff that makes the pattern safe.

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();  // bound here, forever

lock.lock();
condition.await();  // internally: releases lock, sleeps, re-acquires lock
```

Calling `await()` without holding the lock throws `IllegalMonitorStateException` immediately.

`Condition` is an interface — the real implementation is `ConditionObject`, a private inner class inside `AbstractQueuedSynchronizer` (AQS). It is intentionally package-private and not directly accessible.

***

### 5. How timed `await(2, TimeUnit.SECONDS)` works

The sleeping thread does **zero work** during the wait. The flow is:

```
Thread calls await(2s)
      │
      ▼
Calls LockSupport.parkNanos(this, 2_000_000_000L)
      │
      ▼
OS kernel places thread in timer queue with deadline = now + 2s
      │
      ▼
Other threads run freely for 2 seconds
      │
      ▼
Hardware timer interrupt fires
      │
      ▼
Kernel moves thread back to run queue
      │
      ▼
await() returns false (timed out) or true (signalled before timeout)
```

The OS kernel — driven by a hardware timer interrupt — is what guarantees the wakeup. The sleeping thread never wakes itself.

***

### 6. `awaitUninterruptibly()` vs Ctrl-C

These are **two completely separate mechanisms**:

|                          | `Thread.interrupt()`              | Ctrl-C / SIGINT         |
| ------------------------ | --------------------------------- | ----------------------- |
| Level                    | Java thread                       | OS process              |
| What it does             | sets a flag on one thread         | kills the entire JVM    |
| `await()`                | throws `InterruptedException`     | thread dies immediately |
| `awaitUninterruptibly()` | **ignored**, keeps waiting        | thread dies immediately |
| Can be caught?           | yes, catch `InterruptedException` | only via shutdown hook  |
| Affects                  | one thread                        | all threads             |

`awaitUninterruptibly()` only blocks `Thread.interrupt()` — programmatic Java-level interrupts. It has no effect on Ctrl-C or `SIGKILL`.

Internally it defers the interrupt rather than discarding it:

```java
// simplified behaviour
boolean interrupted = false;
while (not signalled) {
    try {
        LockSupport.park(this);
    } catch (InterruptedException e) {
        interrupted = true;   // remember it, but keep waiting
    }
}
if (interrupted) {
    Thread.currentThread().interrupt();  // restore flag after done
}
```

***

### 7. What is CAS?

**Compare-And-Swap** — an atomic CPU instruction used by AQS to change lock state without `synchronized`.

```java
// conceptually
if (current value == expected) {
    set it to newValue;
    return true;
} else {
    return false;
}
```

All three steps happen as one unbreakable CPU instruction. On x86 it compiles to `LOCK CMPXCHG`.

AQS uses it to acquire the lock:

```java
compareAndSet(0, 1)  // "if state is 0 (free), set it to 1 (taken)"
```

***

### 8. Full stack — from Condition down to the OS

```
Condition.await()                  ← Java interface
        │
        ▼
AQS (AbstractQueuedSynchronizer)   ← pure Java queue + CAS logic
        │
        ▼
LockSupport.park()                 ← Java, thin wrapper
        │
        ▼
Unsafe.park()                      ← JVM intrinsic, platform-specific
        │
        ▼
OS primitive                       ← actual thread suspension
```

On each platform the OS primitive differs:

| OS      | Primitive                                          |
| ------- | -------------------------------------------------- |
| Linux   | `pthread_mutex` + `pthread_cond_wait`              |
| macOS   | `pthread_cond_wait`                                |
| Windows | `WaitForSingleObject` / `SleepConditionVariableCS` |

***

### 9. How multiple Java Conditions map to one OS primitive

The OS primitive is **per thread, not per condition**. Every Java thread has exactly one OS-level parker.

```
Java Condition notFull  ─┐
Java Condition notEmpty ─┼──► LockSupport.park(thread) → one OS parker per THREAD
Java Condition notPaused─┘
```

AQS maintains **separate wait queues in the Java heap** — one linked list per `Condition`:

```
ReentrantLock
├── Condition notFull   → queue: [Thread2] → [Thread5]   (Java heap)
├── Condition notEmpty  → queue: [Thread3]               (Java heap)
└── Condition notPaused → queue: [Thread1] → [Thread4]   (Java heap)
```

When `signal()` is called, AQS picks a thread off the Java queue, then calls `LockSupport.unpark(thread)` which pokes that thread's own OS parker. The OS never sees "Condition notEmpty" — it only ever sees "wake up thread 3".

**Java owns: which thread should wake up (routing)** **OS owns: how to suspend and resume threads (scheduling)**

***

### 10. Why the OS needs a condvar at all (not just a mutex)

A mutex alone cannot efficiently express "wait until something becomes true" — you'd have to busy-spin:

```c
// without condvar — broken busy spin
pthread_mutex_lock(&mutex);
while (queue.isEmpty()) {
    pthread_mutex_unlock(&mutex);
    // ← signal could fire HERE and be missed
    pthread_mutex_lock(&mutex);
}
```

The condvar provides one critical operation — **atomic unlock + sleep**:

```c
pthread_cond_wait(&cond, &mutex);
// atomically:
//   1. release mutex
//   2. sleep
// as ONE unbreakable step — no race window
```

Without atomicity, a signal fired between unlock and sleep is permanently missed and the thread sleeps forever.

| Primitive | Solves                                                               |
| --------- | -------------------------------------------------------------------- |
| Mutex     | mutual exclusion — one thread in critical section at a time          |
| Condvar   | atomic unlock + sleep — wait for a condition without missing signals |

***

### 11. Two queues, two levels

The OS and AQS each maintain their own queue for different reasons:

```
Java level (AQS — Java heap)
──────────────────────────────────────────────────────
Condition notEmpty → [Thread2] → [Thread5]
Condition notFull  → [Thread3]

OS level (kernel memory)
──────────────────────────────────────────────────────
pthread_cond_t  →  [OS thread 2] → [OS thread 3]
```

|            | AQS queue (Java)                       | Kernel queue (OS)            |
| ---------- | -------------------------------------- | ---------------------------- |
| Lives in   | Java heap                              | Kernel memory                |
| Tracks     | which Condition a thread is waiting on | which threads are suspended  |
| Decides    | **who** to wake (routing)              | **how** to wake (scheduling) |
| Managed by | AQS / JVM                              | OS scheduler                 |

Java conditions are numerous and cheap because they are just heap linked lists. The expensive part — actual thread suspension — is handled by the kernel's single per-thread parker.
