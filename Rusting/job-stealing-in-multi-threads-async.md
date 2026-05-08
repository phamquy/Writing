---
icon: user-ninja
---

# Job stealing in multi-threads async

In a single-threaded asynchronous model, only one thread can work on the task queue (where your asynchronous tasks are pushed). However, the Rust asynchronous runtime can use multiple threads.

## Why Use Multi-Threads in Async Rust?

Even though Node.js works fine with a single thread, multi-threaded async runtimes (like Tokio) provide advantages in Rust:

1. **CPU-bound Tasks**
   * Node.js’s single thread is great for I/O-bound work, but heavy computations block the event loop.
   * Multi-threaded runtimes can run CPU-heavy tasks in parallel without blocking other async tasks.
2. **High Concurrency on Multi-Core CPUs**
   * Modern machines have many cores. Single-threaded async cannot fully utilize them.
   * Multi-threaded runtimes can schedule futures across multiple cores to maximize throughput.
3. **Blocking Fallback**
   * Sometimes a task needs to do short blocking work (disk I/O, computation).
   * Multi-threading allows these tasks to run on other threads without stalling the entire async loop.
4. **Load Balancing with Work-Stealing**
   * Multiple threads + work-stealing ensures tasks are dynamically balanced, reducing idle cores.

> In short: single-thread async (like Node.js) works great for mostly I/O-bound workloads, but multi-threaded async in Rust allows **full CPU utilization and better handling of mixed workloads**.

## Work-Stealing Async Runtimes

**Work-stealing** is a scheduling strategy used in high-performance async runtimes like Tokio to efficiently distribute tasks across multiple threads, maximizing CPU utilization while minimizing idle time.

***

### 1. Concept Overview

* Each worker thread has its **own deque (double-ended queue)** of tasks to execute.
* Threads primarily **pop tasks from the front** of their own deque (local work).
* If a thread runs out of tasks, it becomes a **thief**:
  * It tries to **steal tasks from the back** of another thread's deque.
* This ensures **load balancing** across threads without central contention.

***

### 2. Why Work-Stealing is Useful

* Async runtimes often have **millions of small futures** that need to be executed.
* Simply using a single global queue leads to:
  * High contention between threads
  * CPU cores idle if one queue is empty
* Work-stealing allows:
  * Each thread to efficiently execute its own tasks
  * Idle threads to pull tasks from busier threads
  * Minimal locking overhead

***

### 3. How Tokio Implements It

1. **Thread Pool**
   * Tokio uses a **multi-threaded scheduler** (`Runtime::Builder::new_multi_thread()`).
   * Each thread runs its **local task queue**.
2. **Task Execution**
   * A thread pops tasks from its **own local queue**.
   * Tasks can spawn additional tasks, which are pushed to the same local queue.
3. **Stealing Tasks**
   * If a thread's local queue is empty, it tries to **steal a task from the back** of another thread’s queue.
   * Back of the deque is chosen because other threads usually push new tasks to the **front**, reducing cache contention.
4. **Reactor Integration**
   * I/O events triggered by the reactor (e.g., epoll) can be enqueued into any worker’s queue.
   * Workers wake up and poll futures as needed, stealing work if necessary.

***

### 4. Diagram (Conceptual)
