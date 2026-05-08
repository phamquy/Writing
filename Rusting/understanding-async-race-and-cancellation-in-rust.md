---
icon: person-running-fast
---

# Understanding Async Race and Cancellation in Rust

### Introduction

When working with async Rust, one of the most powerful patterns is "racing" futures against each other - running multiple async operations concurrently and taking the result of whichever completes first. This article explores how `trpl::race` works internally and how it efficiently cancels slower tasks.

### The Question: Does Racing Waste CPU Cycles?

A common concern when using `trpl::race` is whether the "losing" future continues to consume CPU cycles after the race completes. Let's examine what actually happens:

#### ✅ What Really Happens with `trpl::race`

When you use `trpl::race(future_to_try, trpl::sleep(max_time))`, here's the execution flow:

**If the timeout wins:**

1. `trpl::sleep(max_time)` completes first
2. `trpl::race` returns `Either::Right(_)`
3. **The `future_to_try` is DROPPED** - it stops executing completely
4. **No more CPU cycles are wasted**

Let's test this with a practical example:

```rust
:dep trpl = "*"

use std::future::Future;
use std::time::Duration;
use trpl::Either;

async fn timeout<F: Future>(
    future_to_try: F,
    max_time: Duration,
) -> Result<F::Output, Duration> {
    match trpl::race(future_to_try, trpl::sleep(max_time)).await {
        Either::Left(output) => Ok(output),
        Either::Right(_) => Err(max_time),
    }
}

trpl::run(async {
    let slow = async {
        println!("Task started!");
        
        for i in 1..=10 {
            trpl::sleep(Duration::from_millis(500)).await;
            println!("Task working... {}/10", i);
        }
        
        println!("Task completed!"); // This WON'T print if timeout occurs
        "Task finished successfully"
    };

    println!("Starting timeout test...");
    
    match timeout(slow, Duration::from_secs(2)).await {
        Ok(message) => println!("Success: {}", message),
        Err(duration) => {
            println!("Timeout after {} seconds", duration.as_secs());
        }
    }
    
    println!("Waiting a bit more to see if task continues...");
    trpl::sleep(Duration::from_secs(3)).await;
    println!("Program finished - no more task output should appear");
});
```

**Output Analysis:**

```
Starting timeout test...
Task started!
Task working... 1/10
Task working... 2/10
Task working... 3/10
Task working... 4/10
Timeout after 2 seconds
Waiting a bit more to see if task continues...
Program finished - no more task output should appear
```

**Notice:** No more "Task working..." messages appear after the timeout!

### How Does `poll` Drive Execution Without `await`?

A key insight about async Rust is understanding that **`poll` IS the execution mechanism** - there's no separate "start" statement needed.

#### The Polling Model

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

// Let's demonstrate with a simple future that shows when it's being executed
struct CountingFuture {
    count: u32,
    target: u32,
}

impl Future for CountingFuture {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        println!("🔄 CountingFuture::poll called! count={}, target={}", self.count, self.target);
        
        self.count += 1;
        
        if self.count >= self.target {
            println!("✅ CountingFuture completed with {}", self.count);
            Poll::Ready(self.count)
        } else {
            println!("⏳ CountingFuture not ready yet, waking for next poll");
            cx.waker().wake_by_ref(); // Ask to be polled again
            Poll::Pending
        }
    }
}

// Demo showing when poll gets called
trpl::run(async {
    println!("=== Creating futures ===");
    
    let future_a = CountingFuture { count: 0, target: 3 };
    let future_b = CountingFuture { count: 0, target: 5 };
    
    println!("\n=== Starting race (this will call poll repeatedly) ===");
    
    match trpl::race(future_a, future_b).await {
        trpl::Either::Left(result) => println!("🏆 Future A won with: {}", result),
        trpl::Either::Right(result) => println!("🏆 Future B won with: {}", result),
    }
});
```

#### Key Concepts:

1. **Future Creation ≠ Execution**

```rust
let future_a = async { /* code */ };  // ← Creates future, DOESN'T run it
let future_b = async { /* code */ };  // ← Creates future, DOESN'T run it
```

2. **Polling Drives Execution**

```rust
impl Future for Race<A, B> {
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // THIS LINE executes future A's code:
        if let Poll::Ready(result_a) = Pin::new(&mut self.future_a).poll(cx) {
            //                                                    ^^^^^^^^
            //                                            This RUNS future A!
            return Poll::Ready(Either::Left(result_a));
        }

        // THIS LINE executes future B's code:
        if let Poll::Ready(result_b) = Pin::new(&mut self.future_b).poll(cx) {
            //                                                    ^^^^^^^^
            //                                            This RUNS future B!
            return Poll::Ready(Either::Right(result_b));
        }

        Poll::Pending
    }
}
```

### Conceptual Implementation of Race

Here's how `trpl::race` works internally to achieve efficient cancellation:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use trpl::Either;

// This is conceptually how `race` works internally
struct Race<A, B> {
    future_a: Option<A>,
    future_b: Option<B>,
}

impl<A: Future, B: Future> Future for Race<A, B> {
    type Output = Either<A::Output, B::Output>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // Try polling future A first
        if let Some(ref mut fut_a) = self.future_a {
            if let Poll::Ready(result_a) = Pin::new(fut_a).poll(cx) {
                // A finished first! 
                // CRITICAL: Drop future B to cancel it
                self.future_b = None;  // 🔥 This drops B and cancels it!
                return Poll::Ready(Either::Left(result_a));
            }
        }

        // Try polling future B
        if let Some(ref mut fut_b) = self.future_b {
            if let Poll::Ready(result_b) = Pin::new(fut_b).poll(cx) {
                // B finished first!
                // CRITICAL: Drop future A to cancel it
                self.future_a = None;  // 🔥 This drops A and cancels it!
                return Poll::Ready(Either::Right(result_b));
            }
        }

        // Neither is ready yet
        Poll::Pending
    }
}
```

#### Memory Management Timeline

```
Time    Race Structure                What Happens
────    ──────────────                ────────────
T1      Race {                        Both futures alive
          future_a: Some(FastTask),   
          future_b: Some(SlowTask),   
        }

T2      future_a.poll() → Pending     Execute some of A's code
        future_b.poll() → Pending     Execute some of B's code
        return Pending

T3      future_a.poll() → Ready(x)    Future A completes!
        future_b = None               Future B gets DROPPED
        return Ready(Left(x))
```

### Advanced Cancellation Patterns

#### 1. Enhanced Cancellation with Spawn Tasks

For better control over cancellation, you can use `spawn_task` with handles:

```rust
use std::future::Future;
use std::time::Duration;
use trpl::Either;

async fn timeout_with_cancellation<F, T>(
    future_to_try: F,
    max_time: Duration,
) -> Result<T, Duration> 
where
    F: Future<Output = T> + 'static,
    T: 'static,
{
    // Spawn the task so we can cancel it
    let task_handle = trpl::spawn_task(future_to_try);
    
    match trpl::race(task_handle, trpl::sleep(max_time)).await {
        Either::Left(result) => Ok(result.unwrap()),
        Either::Right(_) => {
            // The task handle is dropped here, which cancels the spawned task
            Err(max_time)
        }
    }
}
```

#### 2. Manual Cancellation with Channels

For cooperative cancellation where tasks can clean up:

```rust
async fn cancellable_task(cancel_rx: &mut trpl::Receiver<()>) -> Result<&'static str, ()> {
    for i in 1..=10 {
        // Check for cancellation signal
        match trpl::race(
            trpl::sleep(Duration::from_millis(500)),
            cancel_rx.recv()
        ).await {
            Either::Left(_) => {
                println!("Working... {}/10", i);
            }
            Either::Right(_) => {
                println!("Task received cancellation signal!");
                return Err(());
            }
        }
    }
    println!("Task completed normally!");
    Ok("Task finished successfully")
}
```

### Key Differences Between Cancellation Methods

| Method                     | Future Behavior After Timeout | CPU Usage            | Use Case                 |
| -------------------------- | ----------------------------- | -------------------- | ------------------------ |
| `trpl::race`               | **Dropped & Stopped**         | ✅ **No waste**       | Simple timeout scenarios |
| `spawn_task` (not awaited) | **Continues running**         | ❌ **Still uses CPU** | Background tasks         |
| Regular async block        | **Dropped & Stopped**         | ✅ **No waste**       | Most common case         |
| Channel-based cancellation | **Cooperative cleanup**       | ✅ **Clean shutdown** | Complex cleanup needed   |

### Mental Model

Think of async execution like this:

1. **Creating a future** = Writing a recipe 📝
2. **Polling a future** = Following the recipe step by step 👨‍🍳
3. **Race polling both** = Having two chefs work on different recipes simultaneously
4. **Winner finishes first** = One chef completes their dish 🏆
5. **Loser gets cancelled** = The other chef stops cooking and cleans up 🗑️

### Timeline Visualization

```
Time    Race::poll()              What Happens
────    ────────────              ────────────
T1      fut_a.poll() → Pending    Execute some of future A's code
        fut_b.poll() → Pending    Execute some of future B's code
        return Pending

T2      fut_a.poll() → Pending    Continue future A's execution  
        fut_b.poll() → Pending    Continue future B's execution
        return Pending

T3      fut_a.poll() → Ready(x)   Future A completes!
        drop(fut_b)               Future B gets cancelled
        return Ready(Left(x))
```

### Summary

The `trpl::race` function provides efficient cancellation through:

1. **Automatic cancellation** - As soon as one future completes, the other is immediately dropped
2. **No CPU waste** - Cancelled futures stop consuming resources instantly
3. **Memory safety** - Rust's ownership system ensures proper cleanup
4. **Zero overhead** - The cancellation mechanism has no runtime cost

The key insight is that **polling IS execution** in async Rust. When a future is dropped (cancelled), its `poll` method will never be called again, effectively stopping all execution and resource consumption.
