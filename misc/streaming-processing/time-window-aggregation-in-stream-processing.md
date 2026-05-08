---
description: >-
  A deep dive into the design decisions that make or break your streaming
  aggregation pipeline
icon: window-frame-open
---

# Time Window Aggregation in Stream Processing

### Introduction

If you've built stream processing systems, you've likely faced this challenge: aggregate events over a sliding time window while handling the messy reality of distributed systems—network delays, clock skew, and out-of-order arrivals.

In this article, we'll explore the key design decisions for window aggregation using a concrete example: a telemetry service ingesting events from many clients.

```
Event Schema:
  eventId: string      // unique identifier
  timestamp: long      // epoch millis, client clock
  value: double        // metric to aggregate
```

**Our requirements example:**

* Maintain a **sliding 5-minute window** for sum/count aggregation
* Support **out-of-order events** (up to 60 seconds late)
* Handle high throughput with predictable resource usage

Let's dive into the three critical decisions you'll need to make.

***

### 1. Wall-Clock Time vs. Event Time: Which Defines Your Window?

This is the most fundamental decision in stream processing, and getting it wrong can lead to data loss, incorrect results, or unbounded memory growth.

#### Option A: Wall-Clock Time (Processing Time)

**How it works:** The window boundaries are determined by your server's clock at the moment of processing.

```
Window = [server_now - 5 minutes, server_now]
```

```
Timeline:
Server clock: 10:00    10:01    10:02    10:03    10:04    10:05    10:06
                |________|________|________|________|________|
                              Active Window
                              
Event arrives at 10:05:30 with timestamp 10:02:00
→ Window has moved to [10:00:30, 10:05:30]
→ Event timestamp 10:02:00 is IN window ✓

Event arrives at 10:06:00 with timestamp 10:00:30  
→ Window has moved to [10:01:00, 10:06:00]
→ Event timestamp 10:00:30 is OUT of window ✗ (dropped)
```

**Implementation:**

```python
def get_window_boundary(self) -> int:
    now = int(time.time() * 1000)
    return now - WINDOW_SIZE_MS

def is_in_window(self, event_timestamp: int) -> bool:
    return event_timestamp >= self.get_window_boundary()
```

**Characteristics:**

| Aspect          | Impact                                                                                                                      |
| --------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Memory**      | ✅ Bounded and predictable. Old data always evicted as wall-clock advances. Max buckets = window\_size / bucket\_granularity |
| **Latency**     | ✅ Low. No waiting for watermarks or late data                                                                               |
| **Throughput**  | ✅ High. Simple boundary check per event                                                                                     |
| **Correctness** | ⚠️ Results depend on when data arrives, not when it happened                                                                |
| **Replay**      | ❌ Non-deterministic. Same data replayed = different results                                                                 |

**When to use:**

* Real-time dashboards where "good enough" is acceptable
* Monitoring/alerting systems where freshness > accuracy
* High-throughput systems where simplicity is crucial
* When you control the producers and network latency is low

***

#### Option B: Event Time (Watermark-Based)

**How it works:** The window boundaries are determined by the timestamps embedded in the events themselves.

```
Window = [max_observed_event_time - 5 minutes, max_observed_event_time]
```

```
Timeline of event arrivals (processing time on x-axis):
Arrival:  10:00    10:01     10:02    10:03     10:04     10:05
Events:   [A@9:58] [C@10:01] [B@9:59] [D@10:02] [E@10:00] ...
                      ↑
              Arrived out of order!
              
Max event time seen: 10:02
Window covers: [9:57, 10:02]
Event B@9:59 → ACCEPTED (still in window based on event time)
```

**The Watermark Concept:**

A watermark is a system's assertion: _"I believe I have received all events with timestamp ≤ W"_

```
                    Events in transit
                    ↓ ↓ ↓
Producer → [Network Buffer] → Processor
                              
Watermark calculation:
  W = max(event_timestamps) - expected_delay
  
If W = 10:02 and expected_delay = 30s:
  Watermark = 10:01:30
  "Safe to finalize windows ending at 10:01:30 or earlier"
```

**Implementation (simplified):**

```python
class EventTimeAggregator:
    def __init__(self):
        self.max_event_time = 0
        self.allowed_lateness = 60_000  # 60 seconds
        self.windows: dict[int, Bucket] = {}
    
    def add_event(self, timestamp: int, value: float):
        self.max_event_time = max(self.max_event_time, timestamp)
        watermark = self.max_event_time - self.allowed_lateness
        
        if timestamp < watermark:
            return False  # Too late, window already closed
        
        window_key = self.get_window_key(timestamp)
        self.windows[window_key].add(value)
        
        # Emit and evict windows below watermark
        self.process_watermark(watermark)
        return True
```

**Characteristics:**

| Aspect          | Impact                                                                              |
| --------------- | ----------------------------------------------------------------------------------- |
| **Memory**      | ⚠️ Potentially unbounded. If events stop, windows never close. Need idle detection. |
| **Latency**     | ⚠️ Higher. Must wait for watermark to advance before emitting results               |
| **Throughput**  | ⚠️ Overhead from watermark tracking and out-of-order buffering                      |
| **Correctness** | ✅ Results reflect when events actually occurred                                     |
| **Replay**      | ✅ Deterministic. Same data = same results                                           |

**When to use:**

* Analytics and reporting where accuracy matters
* Billing systems (every event must be counted correctly)
* Audit logs and compliance
* When producers have unreliable networks (mobile, IoT)

***

#### Option C: Hybrid Approach (Recommended for Most Cases)

**How it works:** Use wall-clock time for window advancement, but allow a **grace period** for late events.

```
  ◄─────────── 5 min window ───────────►◄── 60s grace ──►
  
  [Evicted]      [Active Window]         [Late but OK]    [Too Late]
       ↑                ↑                      ↑               ↑
  wall_clock      wall_clock              wall_clock      event with
  - 6 min          - 5 min                  now           timestamp
                                                          < (now - 6 min)
```

**Implementation:**

```python
class HybridAggregator:
    WINDOW_MS = 5 * 60 * 1000      # 5 minutes
    LATE_TOLERANCE_MS = 60 * 1000  # 60 seconds grace
    
    def get_acceptance_boundary(self) -> int:
        """Events older than this are rejected"""
        now = int(time.time() * 1000)
        return now - self.WINDOW_MS - self.LATE_TOLERANCE_MS
    
    def add_event(self, timestamp: int, value: float) -> bool:
        if timestamp < self.get_acceptance_boundary():
            return False  # Too late
        
        # Event is accepted into appropriate bucket
        bucket_key = timestamp // BUCKET_SIZE_MS
        self.buckets[bucket_key].sum += value
        self.buckets[bucket_key].count += 1
        return True
    
    def get_stats(self) -> tuple[float, int]:
        """Query the current 5-minute window"""
        now = int(time.time() * 1000)
        window_start = now - self.WINDOW_MS
        # Aggregate buckets within [window_start, now]
        ...
```

**Why this is often the best choice:**

| Pure Wall-Clock        | Pure Event-Time                  | Hybrid                  |
| ---------------------- | -------------------------------- | ----------------------- |
| ❌ Drops delayed events | ❌ Memory unbounded               | ✅ Accepts late events   |
| ❌ Non-deterministic    | ❌ Complex watermarks             | ✅ Bounded memory        |
| ✅ Simple               | ❌ Idle source problem            | ✅ Simple implementation |
| ✅ Predictable          | ⚠️ Latency waiting for watermark | ✅ Low latency           |

***

#### Decision Matrix

| Your Situation                        | Recommended Approach                   |
| ------------------------------------- | -------------------------------------- |
| Real-time dashboard, best-effort      | Wall-clock time                        |
| Financial/billing accuracy required   | Event time with watermarks             |
| High throughput + reasonable accuracy | Hybrid with grace period               |
| IoT/mobile with unreliable networks   | Event time or hybrid with longer grace |
| Log aggregation for debugging         | Wall-clock (simplicity wins)           |
| Replay/backfill capability needed     | Event time (determinism required)      |

***

### 2. Exact vs. Approximate Aggregation

Once you've chosen your time semantics, the next question is: **do you need exact counts, or are estimates acceptable?**

#### Exact Aggregation

**How it works:** Store every event (or pre-aggregated buckets) and compute precise results.

```python
# Exact: Store actual values
buckets = {
    1706745600: Bucket(sum=1523.45, count=47),
    1706745601: Bucket(sum=892.10, count=31),
    # ... one bucket per second = 300 buckets for 5-min window
}

def get_exact_sum():
    return sum(b.sum for b in buckets.values())
```

**Characteristics:**

| Metric               | Impact                                                               |
| -------------------- | -------------------------------------------------------------------- |
| **Memory**           | O(window\_size / bucket\_granularity) → predictable but can be large |
| **Accuracy**         | 100% correct                                                         |
| **Query time**       | O(number of buckets)                                                 |
| **Merge complexity** | Simple addition                                                      |

**When to use:**

* Billing and financial calculations
* SLA compliance metrics
* When window size and throughput are bounded
* Audit requirements demand exact numbers

***

#### Approximate Aggregation

For high-cardinality data or extreme throughput, exact counting becomes impractical. Probabilistic data structures offer bounded memory with statistical guarantees.

> **⚠️ Important Clarification:** The data structures below solve a **different problem** than simple window aggregation.

| Problem                                           | Solution                             |
| ------------------------------------------------- | ------------------------------------ |
| "Sum/count of all values in 5-min window"         | **Bucketed HashMap** (exact, simple) |
| "Count per user\_id in 5-min window" (100M users) | **Count-Min Sketch**                 |
| "Unique users in 5-min window"                    | **HyperLogLog**                      |
| "P99 latency in 5-min window"                     | **T-Digest**                         |

> Use these when you need **per-key aggregation with high cardinality**, not for simple global aggregates.

**Count-Min Sketch (Frequency Estimation)**

**Problem it solves:** "How many times did each of 100M users appear in the last 5 minutes?"

Storing `{user_id: count}` for 100M users = \~4GB. Count-Min Sketch does it in \~200KB with slight over-counting.

***

**The Core Intuition: Multiple Witnesses**

Imagine you're counting votes, but you only have limited ballot boxes. Multiple candidates' votes go into each box (hash collision). The problem: how do you know a candidate's true count?

> 🎯 **Key Insight:** If you use **multiple independent ballot box systems**, a candidate's votes will collide with **different** other candidates in each system. The **minimum** count across systems is closest to truth.

**Visual Example:**

```
Counting votes for: Alice, Bob, Carol, Dave (actual: 5, 3, 2, 1)

System 1 (hash function h1):              System 2 (hash function h2):
┌─────┬─────┬─────┬─────┐                ┌─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │                │  0  │  1  │  2  │  3  │
├─────┼─────┼─────┼─────┤                ├─────┼─────┼─────┼─────┤
│  3  │  5  │  2  │  1  │                │  5  │  1  │  6  │  2  │
│ Bob │Alice│Carol│Dave │                │Alice│Dave │B+C  │     │
└─────┴─────┴─────┴─────┘                └─────┴─────┴─────┴─────┘

Query "Bob": min(h1[0]=3, h2[2]=6) = 3 ✅ Correct!
  - System 1: Bob alone → 3 (true count)
  - System 2: Bob+Carol collision → 6 (inflated)
  - Minimum gives us the "cleanest" witness
```

**Why does "minimum" work?**

| Scenario              | What happens                                   |
| --------------------- | ---------------------------------------------- |
| No collision          | All rows show true count → min = true          |
| Collision in 1 row    | That row inflated, others correct → min = true |
| Collision in k rows   | Remaining (d-k) rows correct → min = true      |
| Collision in ALL rows | Over-estimate (rare with enough rows)          |

The probability of collision in ALL rows decreases **exponentially** with depth!

***

**How it works:**

```
Event "user_123" →  h1("user_123") = 3  →  counters[0][3]++
                    h2("user_123") = 7  →  counters[1][7]++
                    h3("user_123") = 1  →  counters[2][1]++

Query: min(counters[0][3], counters[1][7], counters[2][1])
       ↑ Takes the row with fewest collisions
```

**Step-by-step visualization:**

```
Initial state (width=5, depth=3):

Row 0: [0, 0, 0, 0, 0]   ← h1 maps here
Row 1: [0, 0, 0, 0, 0]   ← h2 maps here  
Row 2: [0, 0, 0, 0, 0]   ← h3 maps here

After add("alice"), add("alice"), add("bob"):

              h1(alice)=2  h1(bob)=2 (collision!)
                    ↓         ↓
Row 0: [0, 0, 3, 0, 0]   ← alice(2) + bob(1) = 3

              h2(alice)=4  h2(bob)=1 (no collision)
                    ↓         ↓
Row 1: [0, 1, 0, 0, 2]   ← bob=1, alice=2

              h3(alice)=0  h3(bob)=3 (no collision)
                    ↓         ↓
Row 2: [2, 0, 0, 1, 0]   ← alice=2, bob=1

Query("alice"): min(3, 2, 2) = 2 ✅
Query("bob"):   min(3, 1, 1) = 1 ✅
```

***

**Implementation with Comments:**

```python
import random

class CountMinSketch:
    """
    Probabilistic frequency counter with bounded memory.
    
    Trade-off: May over-count (never under-counts)
    Memory: O(width × depth) regardless of unique items
    """
    
    def __init__(self, width: int = 1000, depth: int = 5):
        """
        Args:
            width: Number of counters per row. More = less collision = more accurate.
                   Error rate ε ≈ e/width (e.g., width=2718 → ~100% max error)
            depth: Number of rows (hash functions). More = higher confidence.
                   Confidence δ = e^(-depth) (e.g., depth=5 → 99.3% confidence)
        """
        self.width = width
        self.depth = depth
        self.table = [[0] * width for _ in range(depth)]
        # Each row uses a different hash function (via different seed)
        self.hash_seeds = [random.randint(0, 2**32) for _ in range(depth)]
    
    def add(self, item: str, count: int = 1) -> None:
        """Increment counters for this item in all rows."""
        for row in range(self.depth):
            col = self._hash(item, row)
            self.table[row][col] += count
    
    def estimate(self, item: str) -> int:
        """
        Return estimated count (may be inflated due to collisions).
        
        Takes minimum across all rows because:
        - True count is added to every row
        - Collisions only ADD to the count
        - Row with fewest collisions = closest to truth
        """
        return min(
            self.table[row][self._hash(item, row)]
            for row in range(self.depth)
        )
    
    def _hash(self, item: str, row: int) -> int:
        """Hash item to a column index for the given row."""
        return hash((item, self.hash_seeds[row])) % self.width

    def merge(self, other: 'CountMinSketch') -> 'CountMinSketch':
        """Merge two sketches (e.g., from different time windows)."""
        assert self.width == other.width and self.depth == other.depth
        result = CountMinSketch(self.width, self.depth)
        result.hash_seeds = self.hash_seeds  # Must use same hash functions!
        for row in range(self.depth):
            for col in range(self.width):
                result.table[row][col] = self.table[row][col] + other.table[row][col]
        return result
```

***

**Memory Sizing: How to Choose Width and Depth**

| Parameter | Formula                 | Intuition                                         |
| --------- | ----------------------- | ------------------------------------------------- |
| **Width** | `width = ceil(e / ε)`   | `ε` = acceptable error as fraction of total count |
| **Depth** | `depth = ceil(ln(1/δ))` | `δ` = probability of exceeding error bound        |

**Example sizing:**

```
Requirements:
- Error ≤ 0.1% of total events (ε = 0.001)
- 99.9% confidence (δ = 0.001)

Calculations:
- width = e / 0.001 = 2,718
- depth = ln(1/0.001) = ln(1000) ≈ 7

Memory = 2,718 × 7 × 4 bytes = 76 KB
  ↑ Compare to HashMap with 100M users = 4+ GB!
```

> ⚠️ **Key Point:** Width is based on **error tolerance**, NOT number of items!
>
> This is why CMS uses constant memory regardless of how many unique items you track.

***

**Guarantees:**

| Property                 | Value                                                     |
| ------------------------ | --------------------------------------------------------- |
| **Never underestimates** | Can only over-count due to collisions                     |
| **Error bound**          | True count ≤ estimate ≤ true + ε×N (with probability 1-δ) |
| **Memory**               | O(width × depth) = O(1/ε × log(1/δ))                      |
| **Insert time**          | O(depth)                                                  |
| **Query time**           | O(depth)                                                  |
| **Mergeable**            | Yes! Sum corresponding cells                              |

***

**Combining with Time Windows:**

```python
from collections import defaultdict

class SlidingWindowFrequency:
    """
    Track per-user event frequency over a sliding 5-minute window.
    
    Architecture:
    - One CMS per minute bucket
    - Sum estimates across all active buckets
    - Evict old buckets as time advances
    """
    
    WINDOW_MINUTES = 5
    
    def __init__(self, sketch_width: int = 10000, sketch_depth: int = 5):
        self.sketch_width = sketch_width
        self.sketch_depth = sketch_depth
        self.sketches: dict[int, CountMinSketch] = {}
    
    def add_event(self, user_id: str, timestamp_ms: int) -> None:
        """Record an event for a user at the given timestamp."""
        bucket = timestamp_ms // 60_000  # 1-minute buckets
        
        if bucket not in self.sketches:
            self.sketches[bucket] = CountMinSketch(
                width=self.sketch_width, 
                depth=self.sketch_depth
            )
        
        self.sketches[bucket].add(user_id)
        self._evict_old(timestamp_ms)
    
    def estimate_user_count(self, user_id: str) -> int:
        """
        Estimate how many events this user had in the last 5 minutes.
        
        Sums estimates across all time buckets.
        Note: Error can accumulate across buckets!
        """
        return sum(
            sketch.estimate(user_id) 
            for sketch in self.sketches.values()
        )
    
    def _evict_old(self, current_timestamp_ms: int) -> None:
        """Remove buckets older than the window."""
        current_bucket = current_timestamp_ms // 60_000
        cutoff = current_bucket - self.WINDOW_MINUTES
        
        old_buckets = [b for b in self.sketches if b < cutoff]
        for bucket in old_buckets:
            del self.sketches[bucket]
    
    def get_memory_usage(self) -> str:
        """Report current memory usage."""
        num_sketches = len(self.sketches)
        bytes_per_sketch = self.sketch_width * self.sketch_depth * 4
        total_bytes = num_sketches * bytes_per_sketch
        return f"{num_sketches} sketches × {bytes_per_sketch/1024:.1f}KB = {total_bytes/1024:.1f}KB"
```

**Usage Example:**

```python
tracker = SlidingWindowFrequency()

# Simulate events
tracker.add_event("user_alice", 1000)
tracker.add_event("user_alice", 2000)
tracker.add_event("user_bob", 3000)
tracker.add_event("user_alice", 4000)

print(tracker.estimate_user_count("user_alice"))  # ~3
print(tracker.estimate_user_count("user_bob"))    # ~1
print(tracker.estimate_user_count("user_unknown")) # 0

print(tracker.get_memory_usage())  # "1 sketches × 195.3KB = 195.3KB"
```

***

**When to Use Count-Min Sketch vs. Alternatives:**

| Scenario                         | Best Choice              | Why                                            |
| -------------------------------- | ------------------------ | ---------------------------------------------- |
| Track frequency of 100M user IDs | **Count-Min Sketch**     | Bounded memory, slight over-count OK           |
| Need exact counts                | **HashMap**              | CMS only approximates                          |
| Find top-K frequent items        | **CMS + Min-Heap**       | CMS for frequency, heap for ranking            |
| Count unique items               | **HyperLogLog**          | Different problem (cardinality, not frequency) |
| Need to delete items             | **Count-Min-Log Sketch** | Standard CMS can't decrement                   |

**HyperLogLog (Cardinality Estimation)**

**Problem it solves:** "How many unique users visited in the last 5 minutes?"

Storing a `Set` of 100M user IDs = \~4GB. HyperLogLog does it in **12KB** with \~2% error!

**The Core Intuition: Consecutive Coin Flips**

Imagine flipping a fair coin repeatedly. The question is: **how many flips until you see k consecutive heads in a row?**

> ⚠️ **Important:** This is about **consecutive runs**, not total heads count!
>
> * Total heads ≈ flips / 2 (not useful here)
> * Consecutive heads = a rare pattern that tells us about sample size

| Consecutive Run  | Probability | Expected Flips to See It |
| ---------------- | ----------- | ------------------------ |
| 1 head in a row  | 1/2 = 50%   | \~2 flips                |
| 2 heads in a row | 1/4 = 25%   | \~4 flips                |
| 3 heads in a row | 1/8 = 12.5% | \~8 flips                |
| k heads in a row | (1/2)^k     | \~2^k flips              |

**The logic:** If you observe a run of 5 consecutive heads, that's a 1/32 probability event. You probably flipped about 32 times to see something that rare.

**Key insight:** If someone says "I saw 5 heads in a row," you'd guess \~32 flips (2^5).

**From Coins to Hashes:**

A good hash function produces uniformly random bits. Leading zeros = "consecutive heads":

```
hash("alice") = 0b 1 0 1 1 0 0 1 0    → 0 leading zeros
hash("bob")   = 0b 0 0 1 0 1 1 0 1    → 2 leading zeros
hash("carol") = 0b 0 0 0 0 1 0 0 1    → 4 leading zeros ← rare!

Max leading zeros = 4 → Estimate ≈ 2^4 = 16 unique items
```

**The Problem: High Variance**

One unlucky hash with 15 leading zeros would estimate 32,768 users. Solution: **multiple buckets**.

**How HyperLogLog Works:**

```
1. Hash each item
2. Use first few bits to pick a bucket (e.g., 2048 buckets)
3. Count leading zeros in remaining bits
4. Store MAX leading zeros per bucket
5. Combine buckets using harmonic mean (reduces outlier impact)
```

```python
class HyperLogLog:
    def __init__(self, num_buckets=2048):
        self.num_buckets = num_buckets
        self.bucket_bits = 11  # log2(2048)
        self.registers = [0] * num_buckets
    
    def add(self, item: str):
        h = hash(item)
        bucket = h & (self.num_buckets - 1)          # First 11 bits → bucket
        remaining = h >> self.bucket_bits
        leading_zeros = self._count_leading_zeros(remaining)
        self.registers[bucket] = max(self.registers[bucket], leading_zeros)
    
    def count(self) -> int:
        # Harmonic mean reduces impact of outliers
        harmonic_mean = self.num_buckets / sum(2**(-r) for r in self.registers)
        alpha = 0.7213 / (1 + 1.079 / self.num_buckets)  # Correction factor
        return int(alpha * self.num_buckets * harmonic_mean)
    
    def _count_leading_zeros(self, value: int) -> int:
        if value == 0: return 32
        zeros = 0
        while (value & 1) == 0:
            zeros += 1
            value >>= 1
        return zeros + 1
```

**Why Harmonic Mean?**

```
Bucket 0:  max=3  → 2^3 = 8
Bucket 1:  max=4  → 2^4 = 16
Bucket 2:  max=15 → 2^15 = 32,768  ← Outlier!

Arithmetic mean: (8 + 16 + 32768) / 3 = 10,930 ❌ Skewed!
Harmonic mean: 3 / (1/8 + 1/16 + 1/32768) ≈ 12 ✅ Outlier dampened
```

<details>

<summary><strong>📐 Deep Dive: Harmonic Mean vs Arithmetic Mean</strong></summary>

**Definitions:**

| Mean Type      | Formula               | Intuition                                                 |
| -------------- | --------------------- | --------------------------------------------------------- |
| **Arithmetic** | (a + b + c) / n       | "Average of values"                                       |
| **Harmonic**   | n / (1/a + 1/b + 1/c) | "Average of rates" or "reciprocal of average reciprocals" |

**Key Property:** Harmonic mean is always ≤ arithmetic mean, and is **much more sensitive to small values**.

**Why This Matters for HyperLogLog:**

Each bucket's estimate is 2^max\_zeros. Outliers produce **huge** overestimates:

```
True unique count: ~20 items
Bucket estimates: [8, 16, 16, 32768]  ← one bucket saw rare pattern

Arithmetic mean: (8 + 16 + 16 + 32768) / 4 = 8,202 ❌ Wildly wrong!
Harmonic mean:   4 / (1/8 + 1/16 + 1/16 + 1/32768) = 4 / 0.1875 ≈ 21 ✅
```

**Intuition: Why harmonic mean ignores outliers**

Think of it as "averaging speeds" instead of distances:

```
Scenario: Drive 100 miles at 50 mph, then 100 miles at 100 mph
  
Wrong (arithmetic): (50 + 100) / 2 = 75 mph average
  
Right (harmonic):   2 / (1/50 + 1/100) = 2 / 0.03 = 66.7 mph
                    ↑ Time at slow speed dominates!
```

For HyperLogLog:

* Small estimates (low leading zeros) = "slow" = common, reliable
* Huge estimates (high leading zeros) = "fast" = rare, unreliable
* Harmonic mean weights toward the **common, reliable** estimates

**Mathematical intuition:**

```
Harmonic mean formula: n / Σ(1/xᵢ)

When xᵢ is huge (outlier):
  1/xᵢ ≈ 0  →  barely affects the sum!
  
When xᵢ is small (normal):
  1/xᵢ is large → dominates the sum
```

**Visual comparison:**

```
Values: [10, 10, 10, 1000]

Arithmetic:  (10+10+10+1000)/4 = 257.5
             ├─────────────────────────────────┤
             Outlier pulls average way up

Harmonic:    4/(0.1+0.1+0.1+0.001) = 4/0.301 = 13.3
             ├───┤
             Outlier contributes almost nothing (0.001 vs 0.3)
```

**In code:**

```python
def arithmetic_mean(values):
    return sum(values) / len(values)

def harmonic_mean(values):
    return len(values) / sum(1/v for v in values)

# Example with outlier
estimates = [8, 16, 16, 32768]
print(f"Arithmetic: {arithmetic_mean(estimates)}")  # 8202.0
print(f"Harmonic:   {harmonic_mean(estimates)}")    # 21.3
```

</details>

**Memory vs. Accuracy:**

| Buckets | Memory    | Error Rate |
| ------- | --------- | ---------- |
| 256     | 256 bytes | \~6.5%     |
| 2,048   | 2 KB      | \~2.3%     |
| 16,384  | 12 KB     | \~0.8%     |

**Combining with Time Windows:**

```python
class SlidingWindowCardinality:
    """Count unique users in last 5 minutes."""
    
    def __init__(self):
        self.hlls: dict[int, HyperLogLog] = {}  # minute → HLL
    
    def add_user(self, user_id: str, timestamp_ms: int):
        bucket = timestamp_ms // 60_000
        if bucket not in self.hlls:
            self.hlls[bucket] = HyperLogLog()
        self.hlls[bucket].add(user_id)
        self._evict_old(timestamp_ms)
    
    def count_unique(self) -> int:
        """Merge all active HLLs and count."""
        merged = HyperLogLog()
        for hll in self.hlls.values():
            for i in range(merged.num_buckets):
                merged.registers[i] = max(merged.registers[i], hll.registers[i])
        return merged.count()
```

**Guarantees:**

* \~2% error with 12KB memory (can count billions of uniques)
* Mergeable: `merged[i] = max(hll1[i], hll2[i])` — perfect for distributed systems
* Add-only: can't remove items (use HLL++ for that)

**T-Digest (Quantile Estimation)**

**Problem it solves:** "What's the P99 latency in the last 5 minutes?"

Storing all latency values for 1M requests = \~8MB. T-Digest does it in **\~10KB** with <1% error at extremes!

***

**The Core Intuition: Compress the Middle, Preserve the Tails**

Imagine you're summarizing 1 million exam scores. For most purposes, you don't need every score—you need:

* The extremes (top/bottom 1%) — for outlier detection
* A rough picture of the middle — for general distribution

> 🎯 **Key Insight:** T-Digest uses **variable-resolution compression**:
>
> * **Tails (P1, P99)**: Store many small, precise clusters
> * **Middle (P50)**: Merge into fewer, larger clusters
>
> This matches how percentile queries work—errors at P50 matter less than errors at P99!

**Visual: Why tails need more precision**

```
Distribution of 1M latency values:

     ▂▃▅██████████████▅▃▂▁
    ─────────────────────────────►
    1ms                      500ms
    
    ◄─── P1-P10 ───►◄─── P10-P90 ───►◄─── P90-P99 ───►
         ↑                  ↑                ↑
    Many small          Few large       Many small
     centroids          centroids        centroids
    (high precision)   (compressed)   (high precision)
```

**Why?** A 1% error at P99 could mean reporting 450ms vs 500ms (huge difference for SLAs). A 1% error at P50 means 100ms vs 101ms (nobody cares).

***

**How It Works: Centroids and Scale Functions**

A T-Digest maintains a sorted list of **centroids**, each representing a cluster of values:

```
Centroid = (mean, weight)
  - mean: average value of points in this cluster
  - weight: count of points in this cluster

Example: 1M latency values compressed to ~100 centroids:

Index  Mean    Weight   Represents
─────  ─────   ──────   ────────────────────
0      1.2ms   50       Very fast requests (P0-P0.005)
1      2.1ms   100      Still fast (P0.005-P0.015)
2      3.5ms   200      ...
...
50     105ms   50000    ← Middle: big clusters!
51     110ms   48000    
...
98     485ms   150      Slow requests
99     512ms   50       Slowest 0.005%
```

**The Scale Function:** Controls how much compression is allowed at each quantile.

```
k(q) = δ/2 × sin⁻¹(2q - 1)

Where:
  q = quantile (0 to 1)
  δ = compression parameter (typically 100-300)

Effect:
  q=0.01 (P1):  k grows slowly → small clusters allowed
  q=0.50 (P50): k grows fastest → large clusters allowed  
  q=0.99 (P99): k grows slowly → small clusters allowed
```

***

**Step-by-Step: Adding a Value**

```
Add latency = 487ms to T-Digest:

Step 1: Find neighboring centroids
  Centroid 97: (mean=472ms, weight=180)
  Centroid 98: (mean=485ms, weight=150)  ← closest!
  Centroid 99: (mean=512ms, weight=50)

Step 2: Check if we can merge into centroid 98
  Current weight: 150
  Max allowed weight at this quantile: k(q=0.985) ≈ 200
  150 + 1 = 151 ≤ 200 ✓ Can merge!

Step 3: Update centroid 98
  new_mean = (485 × 150 + 487 × 1) / 151 = 485.01ms
  new_weight = 151
```

**What if we can't merge?** Create a new centroid (rare at tails, common in middle initially).

***

**Implementation with Comments:**

```python
import bisect
from dataclasses import dataclass
from typing import List

@dataclass
class Centroid:
    mean: float
    weight: int
    
    def merge_with(self, value: float, count: int = 1) -> None:
        """Merge a new value into this centroid."""
        total_weight = self.weight + count
        self.mean = (self.mean * self.weight + value * count) / total_weight
        self.weight = total_weight


class TDigest:
    """
    Approximate quantile estimation with bounded memory.
    
    Key properties:
    - High accuracy at tails (P1, P99, P99.9)
    - Bounded memory (~100 centroids regardless of data size)
    - Mergeable across distributed systems
    """
    
    def __init__(self, compression: float = 100):
        """
        Args:
            compression: Controls accuracy vs memory trade-off.
                        Higher = more centroids = more accurate.
                        Typical values: 100 (default), 200 (high accuracy)
        """
        self.compression = compression
        self.centroids: List[Centroid] = []
        self.total_weight = 0
        self._buffer: List[float] = []  # Buffer for batch processing
        self._buffer_size = 500
    
    def add(self, value: float) -> None:
        """Add a single value to the digest."""
        self._buffer.append(value)
        if len(self._buffer) >= self._buffer_size:
            self._flush_buffer()
    
    def _flush_buffer(self) -> None:
        """Process buffered values and merge into centroids."""
        if not self._buffer:
            return
        
        # Sort buffer and merge with existing centroids
        self._buffer.sort()
        for value in self._buffer:
            self._add_single(value)
        self._buffer.clear()
        
        # Compress if we have too many centroids
        if len(self.centroids) > self.compression * 3:
            self._compress()
    
    def _add_single(self, value: float) -> None:
        """Add a single value, merging into existing centroid if possible."""
        self.total_weight += 1
        
        if not self.centroids:
            self.centroids.append(Centroid(value, 1))
            return
        
        # Find insertion point
        idx = bisect.bisect_left(
            [c.mean for c in self.centroids], value
        )
        
        # Try to merge with nearest centroid
        candidates = []
        if idx > 0:
            candidates.append((idx - 1, abs(self.centroids[idx-1].mean - value)))
        if idx < len(self.centroids):
            candidates.append((idx, abs(self.centroids[idx].mean - value)))
        
        if candidates:
            best_idx, _ = min(candidates, key=lambda x: x[1])
            centroid = self.centroids[best_idx]
            
            # Check weight limit using scale function
            q = self._quantile_at(best_idx)
            max_weight = self._max_weight(q)
            
            if centroid.weight + 1 <= max_weight:
                centroid.merge_with(value)
                return
        
        # Can't merge—create new centroid
        new_centroid = Centroid(value, 1)
        bisect.insort(self.centroids, new_centroid, key=lambda c: c.mean)
    
    def _quantile_at(self, idx: int) -> float:
        """Estimate quantile position of centroid at given index."""
        weight_before = sum(c.weight for c in self.centroids[:idx])
        weight_at = self.centroids[idx].weight
        return (weight_before + weight_at / 2) / self.total_weight
    
    def _max_weight(self, q: float) -> float:
        """
        Scale function: determines max cluster size at quantile q.
        
        Uses arcsin to give more resolution at tails (q near 0 or 1).
        """
        import math
        # k(q) = δ/2π × arcsin(2q - 1) + 0.5
        # Derivative gives allowed weight
        k = self.compression / (2 * math.pi) * math.asin(2 * q - 1) + 0.5
        return max(1, self.compression * self._scale_derivative(q))
    
    def _scale_derivative(self, q: float) -> float:
        """Derivative of scale function—controls compression ratio."""
        import math
        q = max(0.001, min(0.999, q))  # Avoid edge singularities
        return 1 / (2 * math.pi * math.sqrt(q * (1 - q)))
    
    def _compress(self) -> None:
        """Merge adjacent centroids to reduce count."""
        if len(self.centroids) <= 1:
            return
        
        new_centroids = [self.centroids[0]]
        for c in self.centroids[1:]:
            last = new_centroids[-1]
            q = self._quantile_at(len(new_centroids) - 1)
            max_weight = self._max_weight(q)
            
            if last.weight + c.weight <= max_weight:
                # Merge
                total = last.weight + c.weight
                last.mean = (last.mean * last.weight + c.mean * c.weight) / total
                last.weight = total
            else:
                new_centroids.append(c)
        
        self.centroids = new_centroids
    
    def quantile(self, q: float) -> float:
        """
        Estimate the value at quantile q (0 to 1).
        
        Examples:
            quantile(0.50) → median
            quantile(0.99) → P99
            quantile(0.999) → P99.9
        """
        self._flush_buffer()
        
        if not self.centroids:
            raise ValueError("No data")
        
        target_weight = q * self.total_weight
        cumulative = 0
        
        for i, centroid in enumerate(self.centroids):
            if cumulative + centroid.weight >= target_weight:
                # Interpolate within this centroid
                if i == 0:
                    return centroid.mean
                
                prev = self.centroids[i - 1]
                # Linear interpolation between centroids
                weight_in_centroid = target_weight - cumulative
                fraction = weight_in_centroid / centroid.weight
                return prev.mean + fraction * (centroid.mean - prev.mean)
            
            cumulative += centroid.weight
        
        return self.centroids[-1].mean
    
    def percentile(self, p: float) -> float:
        """Convenience method: percentile(99) = quantile(0.99)"""
        return self.quantile(p / 100)
    
    def merge(self, other: 'TDigest') -> 'TDigest':
        """Merge two T-Digests (e.g., from different shards)."""
        result = TDigest(self.compression)
        
        # Combine all centroids
        all_centroids = self.centroids + other.centroids
        all_centroids.sort(key=lambda c: c.mean)
        
        result.centroids = all_centroids
        result.total_weight = self.total_weight + other.total_weight
        result._compress()
        
        return result
    
    def memory_usage(self) -> str:
        """Report memory usage."""
        # Each centroid: ~16 bytes (mean: 8, weight: 8)
        bytes_used = len(self.centroids) * 16 + len(self._buffer) * 8
        return f"{len(self.centroids)} centroids = {bytes_used / 1024:.1f} KB"
```

***

**Why the Scale Function Works:**

```
The magic is in the arcsin-based scale function:

q (quantile)  |  Allowed weight  |  Why
──────────────┼──────────────────┼─────────────────────────────
0.01 (P1)     |  ~50             |  Tail: need precision
0.10 (P10)    |  ~200            |  Still important
0.50 (P50)    |  ~2000           |  Middle: can be approximate
0.90 (P90)    |  ~200            |  Getting important again
0.99 (P99)    |  ~50             |  Tail: need precision

Visual of weight limits across quantiles:

Max Weight
    │
2000┤           ╭───────╮
    │          ╱         ╲
 500┤        ╱             ╲
    │      ╱                 ╲
  50┼─────╱                   ╲─────
    └────┴────┴────┴────┴────┴────→ Quantile
       0.01  0.25  0.50  0.75  0.99
       
       ↑                       ↑
    Tight                   Tight
    (many small clusters)   (many small clusters)
```

***

**Combining with Time Windows:**

```python
from collections import defaultdict

class SlidingWindowPercentiles:
    """
    Track P50, P99, P99.9 latencies over a sliding 5-minute window.
    
    Architecture:
    - One T-Digest per minute bucket
    - Merge active buckets for queries
    - Evict old buckets as time advances
    """
    
    WINDOW_MINUTES = 5
    
    def __init__(self, compression: float = 100):
        self.compression = compression
        self.digests: dict[int, TDigest] = {}
    
    def add_latency(self, latency_ms: float, timestamp_ms: int) -> None:
        """Record a latency measurement."""
        bucket = timestamp_ms // 60_000  # 1-minute buckets
        
        if bucket not in self.digests:
            self.digests[bucket] = TDigest(self.compression)
        
        self.digests[bucket].add(latency_ms)
        self._evict_old(timestamp_ms)
    
    def get_percentile(self, p: float) -> float:
        """
        Get the Pth percentile across the entire window.
        
        Examples:
            get_percentile(50)  → median latency
            get_percentile(99)  → P99 latency
            get_percentile(99.9) → P99.9 latency
        """
        if not self.digests:
            raise ValueError("No data in window")
        
        # Merge all active digests
        merged = None
        for digest in self.digests.values():
            if merged is None:
                merged = digest
            else:
                merged = merged.merge(digest)
        
        return merged.percentile(p)
    
    def get_stats(self) -> dict:
        """Get common percentiles in one call."""
        return {
            "p50": self.get_percentile(50),
            "p90": self.get_percentile(90),
            "p95": self.get_percentile(95),
            "p99": self.get_percentile(99),
            "p99.9": self.get_percentile(99.9),
        }
    
    def _evict_old(self, current_timestamp_ms: int) -> None:
        """Remove buckets older than the window."""
        current_bucket = current_timestamp_ms // 60_000
        cutoff = current_bucket - self.WINDOW_MINUTES
        
        old_buckets = [b for b in self.digests if b < cutoff]
        for bucket in old_buckets:
            del self.digests[bucket]
    
    def memory_usage(self) -> str:
        """Report memory usage across all buckets."""
        total_centroids = sum(len(d.centroids) for d in self.digests.values())
        bytes_used = total_centroids * 16
        return f"{len(self.digests)} buckets, {total_centroids} centroids = {bytes_used / 1024:.1f} KB"
```

**Usage Example:**

```python
import random
import time

tracker = SlidingWindowPercentiles()

# Simulate latency data (mostly fast, some slow)
for i in range(10000):
    # 95% requests: 10-100ms, 5% requests: 200-500ms (slow)
    if random.random() < 0.95:
        latency = random.uniform(10, 100)
    else:
        latency = random.uniform(200, 500)
    
    tracker.add_latency(latency, int(time.time() * 1000))

# Query percentiles
stats = tracker.get_stats()
print(f"P50:   {stats['p50']:.1f}ms")    # ~55ms (middle of 10-100)
print(f"P95:   {stats['p95']:.1f}ms")    # ~100ms (edge of fast)
print(f"P99:   {stats['p99']:.1f}ms")    # ~350ms (middle of slow)
print(f"P99.9: {stats['p99.9']:.1f}ms")  # ~480ms (extreme slow)

print(tracker.memory_usage())  # "1 buckets, 87 centroids = 1.4 KB"
```

***

**Guarantees and Trade-offs:**

| Property            | Value                                       |
| ------------------- | ------------------------------------------- |
| **Accuracy at P99** | < 1% error                                  |
| **Accuracy at P50** | < 5% error (by design)                      |
| **Memory**          | O(compression) = \~100 centroids = \~1.6 KB |
| **Insert time**     | O(log n) amortized                          |
| **Query time**      | O(centroids)                                |
| **Mergeable**       | ✅ Perfect for distributed systems           |

**When to Use T-Digest vs Alternatives:**

| Scenario                          | Best Choice            | Why                         |
| --------------------------------- | ---------------------- | --------------------------- |
| P99, P99.9 latency monitoring     | **T-Digest**           | Optimized for tail accuracy |
| Exact median needed               | **Reservoir Sampling** | T-Digest is approximate     |
| Streaming median only             | **Two-Heap**           | Simpler, exact for median   |
| All percentiles equally important | **DDSketch**           | Uniform relative error      |
| Memory extremely limited          | **Q-Digest**           | Even more compact           |

***

#### Performance Comparison

| Method            | Memory (1M events)   | Query Time | Accuracy         | Use Case                    |
| ----------------- | -------------------- | ---------- | ---------------- | --------------------------- |
| Exact (bucketed)  | \~10KB (300 buckets) | O(buckets) | 100%             | Sum/count in bounded window |
| Exact (per-event) | \~100MB              | O(n)       | 100%             | Need raw events             |
| Count-Min Sketch  | \~40KB               | O(depth)   | Over-estimate ≤ε | Top-K, frequency            |
| HyperLogLog       | \~12KB               | O(1)       | ±2%              | Unique counts               |
| T-Digest          | \~10KB               | O(log n)   | ±1% at tails     | Percentiles                 |

***

#### Decision Matrix

| Your Aggregation Need       | Recommended Approach    |
| --------------------------- | ----------------------- |
| Sum, count, average         | Exact with time buckets |
| Unique visitors             | HyperLogLog             |
| Top-K items                 | Count-Min Sketch + Heap |
| P50, P99 latency            | T-Digest                |
| "Approximately 1M requests" | HyperLogLog             |
| "Exactly 1,523.47 billed"   | Exact                   |

***

### 3. Handling Late Events: Drop, Log, or Reprocess?

Events that arrive after the window has closed present a policy decision with real business implications.

#### Option A: Drop Silently

```python
def add_event(self, timestamp: int, value: float) -> bool:
    if timestamp < self.get_acceptance_boundary():
        return False  # Silently dropped
    # ... process event
```

**Characteristics:**

* ✅ Simplest implementation
* ✅ Predictable resource usage
* ❌ Data loss without visibility
* ❌ Hard to debug missing data issues

**When to use:**

* High-volume, low-value events (debug logs)
* When late data is truly worthless
* Ephemeral metrics (real-time dashboards)

***

#### Option B: Drop and Log

```python
def add_event(self, timestamp: int, value: float) -> bool:
    if timestamp < self.get_acceptance_boundary():
        self.late_event_counter.increment()
        logger.warning(f"Late event dropped: ts={timestamp}, delay={now - timestamp}ms")
        return False
    # ... process event
```

**Metrics to expose:**

```python
# Essential late-event metrics
metrics = {
    "late_events_total": Counter(),           # Total dropped
    "late_events_by_source": Counter(labels=["source"]),  # Per-producer
    "late_event_delay_seconds": Histogram(),  # How late they are
    "late_event_rate": Gauge(),               # Events/sec being dropped
}
```

**Alerting thresholds:**

| Metric                         | Warning             | Critical       |
| ------------------------------ | ------------------- | -------------- |
| Late event rate                | > 1% of total       | > 5% of total  |
| Max delay observed             | > 2x tolerance      | > 5x tolerance |
| Late events from single source | > 10% of its events | > 25%          |

**When to use:**

* Production systems where visibility matters
* Need to tune late tolerance over time
* Debugging producer clock issues

***

#### Option C: Side-Output for Reprocessing

```python
def add_event(self, event: Event) -> ProcessingResult:
    if event.timestamp < self.get_acceptance_boundary():
        # Write to dead-letter queue for later analysis/reprocessing
        self.late_event_queue.put(event)
        return ProcessingResult.LATE_QUEUED
    
    self.aggregate(event)
    return ProcessingResult.PROCESSED
```

```
Main Pipeline:
Events → [Aggregator] → Results
              ↓
         Late events
              ↓
      [Dead Letter Queue]
              ↓
      [Batch Reprocessor] → Corrections/Adjustments
```

**When to use:**

* Financial systems (every transaction matters)
* Compliance/audit requirements
* Analytics where historical accuracy is corrected later

***

#### Option D: Correction/Amendment Flow

For systems that can issue corrections:

```python
class AggregatorWithCorrections:
    def __init__(self):
        self.live_aggregates = {}      # Real-time results
        self.correction_buffer = {}     # Pending corrections
    
    def add_event(self, event: Event):
        if self.is_in_current_window(event.timestamp):
            self.live_aggregates.update(event)
        elif self.is_correctable(event.timestamp):
            # Within correction window (e.g., past 24 hours)
            self.correction_buffer.append(event)
            self.schedule_correction_job()
        else:
            self.log_and_drop(event)
    
    def emit_hourly_results(self):
        # Emit with "preliminary" flag
        yield Result(data=self.live_aggregates, status="PRELIMINARY")
    
    def emit_corrections(self):
        # Emit corrections for affected time periods
        for period, corrections in self.correction_buffer.items():
            yield Correction(period=period, delta=corrections)
```

**When to use:**

* Business intelligence / data warehouse feeds
* When downstream can handle amendments
* Regulatory reporting with correction windows

***

#### Decision Matrix

| Your Situation         | Late Event Strategy                  |
| ---------------------- | ------------------------------------ |
| Debug/development logs | Drop silently                        |
| Production monitoring  | Drop and log + alerts                |
| E-commerce analytics   | Side-output + daily corrections      |
| Payment processing     | Side-output + mandatory reprocessing |
| Real-time bidding      | Drop (latency > accuracy)            |
| IoT sensor data        | Configurable per-sensor policy       |

***

### Putting It All Together: A Production-Ready Design

Based on the patterns discussed, here's a production architecture:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Telemetry Ingestion Service                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Events ──→ [Validation] ──→ [Router by timestamp]                      │
│                                     │                                   │
│                    ┌────────────────┼────────────────┐                  │
│                    ↓                ↓                ↓                  │
│              [In-Window]     [Late but OK]    [Too Late]                │
│                    │                │                │                  │
│                    ↓                ↓                ↓                  │
│              ┌─────────────────────────┐      [DLQ + Metrics]           │
│              │   Window Aggregator     │                                │
│              │   - Bucketed by second  │                                │
│              │   - RWLock for reads    │                                │
│              │   - Lazy eviction       │                                │
│              └───────────┬─────────────┘                                │
│                          │                                              │
│                          ↓                                              │
│              [Query API: getStats()]                                    │
│                                                                         │
│  Metrics exported:                                                      │
│  - events_processed_total                                               │
│  - events_late_total (by delay bucket)                                  │
│  - window_bucket_count                                                  │
│  - aggregation_latency_seconds                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Key Configuration Parameters

```yaml
window_aggregator:
  # Time semantics
  window_type: "hybrid"              # wall-clock | event-time | hybrid
  window_size: "5m"
  late_tolerance: "60s"
  
  # Aggregation
  aggregation_type: "exact"          # exact | approximate
  bucket_granularity: "1s"           # Trades memory for precision
  
  # Late event handling
  late_event_policy: "log_and_drop"  # silent | log_and_drop | side_output
  late_event_side_output: "kafka://late-events"
  
  # Operational
  eviction_strategy: "lazy"          # lazy | background_thread
  metrics_prefix: "telemetry_aggregator"
```

***

### 4. Advanced Topics and Trade-offs

Beyond the core design, production systems face several nuanced challenges. Let's explore each in depth.

***

#### 4.1 Accuracy vs. Memory: Bucket Granularity Trade-offs

**The core tension:** Smaller time buckets give more accurate sliding windows, but consume more memory.

```
5-minute window with different granularities:

1-second buckets:  [B1][B2][B3]...[B300]  → 300 buckets, ~10KB
10-second buckets: [B1][B2][B3]...[B30]   → 30 buckets, ~1KB  
1-minute buckets:  [B1][B2][B3][B4][B5]   → 5 buckets, ~200B
```

**Why granularity matters for accuracy:**

```
Query at T=305 seconds: "Give me sum of last 5 minutes"
True window: [T-300, T] = [5, 305]

With 1-second buckets:
  Include: buckets 5, 6, 7, ..., 305
  Accuracy: Perfect ✅

With 1-minute buckets (60s each):
  Include: buckets covering [0-60], [60-120], [120-180], [180-240], [240-300]
  Problem: Bucket [0-60] includes data from T=0-4 (outside window!)
  Accuracy: Up to 60 seconds of extra/missing data ❌
```

**Visual: Bucket boundary misalignment**

```
True window:     |◄────────── 5 minutes ──────────►|
                 5s                               305s

1-min buckets:   [0────60][60───120][120──180][180──240][240──300]
                 ↑
                 Includes 5 seconds of stale data!
```

**Implementation: Partial bucket handling**

```python
class AccurateSlidingWindow:
    """Handle partial buckets at window boundaries."""
    
    def __init__(self, window_ms: int, bucket_ms: int):
        self.window_ms = window_ms
        self.bucket_ms = bucket_ms
        self.buckets: dict[int, Bucket] = {}
    
    def get_stats(self, now: int) -> Stats:
        window_start = now - self.window_ms
        result = Stats(sum=0, count=0)
        
        for bucket_start, bucket in self.buckets.items():
            bucket_end = bucket_start + self.bucket_ms
            
            if bucket_end <= window_start:
                continue  # Entirely outside window
            
            if bucket_start >= window_start:
                # Fully inside window
                result.sum += bucket.sum
                result.count += bucket.count
            else:
                # Partially inside - interpolate
                overlap = bucket_end - window_start
                fraction = overlap / self.bucket_ms
                result.sum += bucket.sum * fraction
                result.count += int(bucket.count * fraction)
        
        return result
```

> ⚠️ **Caveat:** Interpolation assumes uniform distribution within buckets. For bursty traffic, this can be inaccurate.

**Decision matrix:**

| Granularity | Memory (5-min window) | Accuracy    | Use Case                |
| ----------- | --------------------- | ----------- | ----------------------- |
| 100ms       | \~3000 buckets, 100KB | Excellent   | High-frequency trading  |
| 1 second    | \~300 buckets, 10KB   | Very good   | Real-time dashboards    |
| 10 seconds  | \~30 buckets, 1KB     | Good        | General monitoring      |
| 1 minute    | \~5 buckets, 200B     | Approximate | Low-cardinality metrics |

***

#### 4.2 Lock Granularity: Concurrency Strategies

**The problem:** Multiple threads inserting events and querying stats simultaneously.

```
Thread 1: add_event(ts=1000, val=50)  ──┐
Thread 2: add_event(ts=1001, val=30)  ──┼──► Concurrent writes
Thread 3: add_event(ts=1000, val=20)  ──┘
Thread 4: get_stats()                 ──────► Concurrent read
```

**Strategy A: Global Lock (Simple but Slow)**

```python
class GlobalLockAggregator:
    def __init__(self):
        self.lock = threading.Lock()
        self.buckets = {}
    
    def add_event(self, ts: int, value: float):
        with self.lock:  # All threads serialize here
            bucket = ts // 1000
            if bucket not in self.buckets:
                self.buckets[bucket] = Bucket()
            self.buckets[bucket].add(value)
    
    def get_stats(self):
        with self.lock:  # Readers also wait
            return self._compute_stats()
```

**Performance:** \~100K ops/sec (lock contention is the bottleneck)

***

**Strategy B: Read-Write Lock (Better for Read-Heavy)**

```python
class RWLockAggregator:
    def __init__(self):
        self.rw_lock = threading.RWLock()  # Or use readerwriterlock package
        self.buckets = {}
    
    def add_event(self, ts: int, value: float):
        with self.rw_lock.write():  # Exclusive
            # ... update bucket
            pass
    
    def get_stats(self):
        with self.rw_lock.read():  # Shared - multiple readers OK
            return self._compute_stats()
```

**Performance:** \~500K reads/sec, \~50K writes/sec (good when reads >> writes)

***

**Strategy C: Per-Bucket Locks (Fine-Grained)**

```python
from collections import defaultdict

class PerBucketLockAggregator:
    def __init__(self):
        self.buckets = {}
        self.bucket_locks = defaultdict(threading.Lock)
        self.global_lock = threading.Lock()  # For bucket creation only
    
    def add_event(self, ts: int, value: float):
        bucket_key = ts // 1000
        
        # Fast path: bucket exists
        if bucket_key in self.buckets:
            with self.bucket_locks[bucket_key]:
                self.buckets[bucket_key].add(value)
            return
        
        # Slow path: create bucket
        with self.global_lock:
            if bucket_key not in self.buckets:
                self.buckets[bucket_key] = Bucket()
        
        with self.bucket_locks[bucket_key]:
            self.buckets[bucket_key].add(value)
    
    def get_stats(self):
        # Snapshot bucket keys first
        with self.global_lock:
            keys = list(self.buckets.keys())
        
        # Aggregate without holding global lock
        result = Stats()
        for key in keys:
            with self.bucket_locks[key]:
                result += self.buckets[key].stats()
        return result
```

**Performance:** \~1M ops/sec (writes to different buckets don't block each other)

***

**Strategy D: Lock-Free with Atomic Operations (Best Throughput)**

```python
from atomics import AtomicInt64, AtomicFloat64
from dataclasses import dataclass

@dataclass
class AtomicBucket:
    sum: AtomicFloat64
    count: AtomicInt64
    
    def add(self, value: float):
        self.sum.fetch_add(value)
        self.count.fetch_add(1)

class LockFreeAggregator:
    """Uses atomic operations for lock-free updates."""
    
    def __init__(self, max_buckets: int = 300):
        # Pre-allocate buckets to avoid dynamic allocation
        self.buckets = [AtomicBucket() for _ in range(max_buckets)]
        self.bucket_count = max_buckets
    
    def add_event(self, ts: int, value: float):
        bucket_idx = (ts // 1000) % self.bucket_count
        self.buckets[bucket_idx].add(value)
    
    def get_stats(self):
        # Reading atomics is lock-free
        total_sum = sum(b.sum.load() for b in self.buckets)
        total_count = sum(b.count.load() for b in self.buckets)
        return Stats(sum=total_sum, count=total_count)
```

**Performance:** \~5M ops/sec (limited only by memory bandwidth)

> ⚠️ **Trade-off:** Pre-allocated array means fixed bucket count. Good for known window sizes.

***

**Strategy E: ConcurrentHashMap (Java) / Sharded Dict (Python)**

```python
class ShardedAggregator:
    """Shard by bucket key to reduce contention."""
    
    NUM_SHARDS = 16
    
    def __init__(self):
        self.shards = [
            {"lock": threading.Lock(), "buckets": {}}
            for _ in range(self.NUM_SHARDS)
        ]
    
    def _get_shard(self, bucket_key: int) -> dict:
        return self.shards[bucket_key % self.NUM_SHARDS]
    
    def add_event(self, ts: int, value: float):
        bucket_key = ts // 1000
        shard = self._get_shard(bucket_key)
        
        with shard["lock"]:
            if bucket_key not in shard["buckets"]:
                shard["buckets"][bucket_key] = Bucket()
            shard["buckets"][bucket_key].add(value)
    
    def get_stats(self):
        result = Stats()
        for shard in self.shards:
            with shard["lock"]:
                for bucket in shard["buckets"].values():
                    result += bucket.stats()
        return result
```

**Performance:** \~2M ops/sec (16x reduction in contention)

***

**Comparison:**

| Strategy        | Throughput     | Complexity | Best For             |
| --------------- | -------------- | ---------- | -------------------- |
| Global Lock     | \~100K/s       | Simple     | Prototyping          |
| RW Lock         | \~500K reads/s | Moderate   | Read-heavy workloads |
| Per-Bucket Lock | \~1M/s         | Complex    | General purpose      |
| Lock-Free       | \~5M/s         | Complex    | Extreme throughput   |
| Sharded         | \~2M/s         | Moderate   | Good balance         |

***

#### 4.3 Eviction Strategy: Background Thread vs. Lazy Eviction

**The problem:** Old buckets must be removed to bound memory. When and how?

**Strategy A: Lazy Eviction (On-Access)**

```python
class LazyEvictionAggregator:
    def add_event(self, ts: int, value: float):
        # Evict on every Nth write
        if self.write_count % 100 == 0:
            self._evict_stale(ts)
        
        # ... add to bucket
    
    def get_stats(self, now: int):
        self._evict_stale(now)  # Evict before reading
        # ... compute stats
    
    def _evict_stale(self, now: int):
        cutoff = now - self.window_ms - self.tolerance_ms
        stale = [k for k in self.buckets if k < cutoff]
        for key in stale:
            del self.buckets[key]
```

**Pros:**

* ✅ No background threads to manage
* ✅ No coordination overhead
* ✅ Eviction happens naturally with traffic

**Cons:**

* ❌ Latency spikes during eviction (unpredictable)
* ❌ If traffic stops, memory isn't reclaimed
* ❌ Eviction work proportional to staleness

**GC Pressure Analysis:**

```
Lazy eviction with Python dict:

Before eviction: 1000 buckets, ~300KB
Evict 100 buckets: 
  - 100 dict key deletions
  - 100 Bucket objects become garbage
  - Next GC cycle: ~1ms pause

Impact: Occasional latency spikes during GC
```

***

**Strategy B: Background Thread Eviction**

```python
import threading
import time

class BackgroundEvictionAggregator:
    def __init__(self):
        self.buckets = {}
        self.lock = threading.RLock()
        self._start_evictor()
    
    def _start_evictor(self):
        def evict_loop():
            while True:
                time.sleep(1)  # Run every second
                self._evict_stale(int(time.time() * 1000))
        
        thread = threading.Thread(target=evict_loop, daemon=True)
        thread.start()
    
    def _evict_stale(self, now: int):
        with self.lock:
            cutoff = now - self.window_ms - self.tolerance_ms
            stale = [k for k in self.buckets if k < cutoff]
            for key in stale:
                del self.buckets[key]
    
    def add_event(self, ts: int, value: float):
        with self.lock:
            # No eviction logic here - just add
            pass
```

**Pros:**

* ✅ Predictable eviction timing
* ✅ Memory reclaimed even without traffic
* ✅ Write path is simpler/faster

**Cons:**

* ❌ Thread coordination complexity
* ❌ Must handle shutdown gracefully
* ❌ Lock contention during eviction

***

**Strategy C: Generational/Epoch-Based Eviction**

```python
class EpochEvictionAggregator:
    """
    Use epochs instead of per-bucket timestamps.
    When epoch advances, swap entire bucket maps.
    """
    
    def __init__(self, epoch_duration_ms: int = 60_000):
        self.epoch_duration_ms = epoch_duration_ms
        self.epochs: dict[int, dict] = {}  # epoch_num → buckets
        self.max_epochs = 5  # Keep last 5 epochs
    
    def add_event(self, ts: int, value: float):
        epoch = ts // self.epoch_duration_ms
        
        if epoch not in self.epochs:
            self.epochs[epoch] = {}
            self._evict_old_epochs(epoch)
        
        bucket = ts // 1000
        if bucket not in self.epochs[epoch]:
            self.epochs[epoch][bucket] = Bucket()
        self.epochs[epoch][bucket].add(value)
    
    def _evict_old_epochs(self, current_epoch: int):
        # O(1) eviction - just drop entire epoch dict
        old = [e for e in self.epochs if e < current_epoch - self.max_epochs]
        for epoch in old:
            del self.epochs[epoch]  # One deletion, many buckets freed
```

**Pros:**

* ✅ O(1) eviction regardless of bucket count
* ✅ Amortized GC pressure (large batch frees)
* ✅ Natural alignment with time

**Cons:**

* ❌ Less precise window boundaries
* ❌ Memory usage varies with epoch size

***

**GC-Aware Design (Python Specific):**

```python
import gc

class GCAwareAggregator:
    def __init__(self):
        self.buckets = {}
        self.pending_deletion = []  # Batch deletions
        self.deletion_threshold = 100
    
    def _evict_stale(self, now: int):
        cutoff = now - self.window_ms
        stale = [k for k in self.buckets if k < cutoff]
        
        for key in stale:
            bucket = self.buckets.pop(key)
            self.pending_deletion.append(bucket)
        
        # Batch GC when enough garbage accumulated
        if len(self.pending_deletion) > self.deletion_threshold:
            self.pending_deletion.clear()
            gc.collect(generation=0)  # Minor collection only
```

***

**Decision matrix:**

| Strategy    | Latency           | Memory          | Complexity | Best For            |
| ----------- | ----------------- | --------------- | ---------- | ------------------- |
| Lazy        | Variable spikes   | Good            | Simple     | Low-traffic systems |
| Background  | Consistent        | Good            | Moderate   | Production systems  |
| Epoch-based | Consistent        | Slightly higher | Moderate   | High-throughput     |
| GC-Aware    | Controlled spikes | Good            | Complex    | Latency-sensitive   |

***

#### 4.4 Clock Skew: Handling Client Time Drift

**The problem:** Events come with client-side timestamps, but clients have drifted clocks.

```
Server time: 2024-01-15 10:00:00.000 UTC

Client A (accurate):     sends event with ts = 10:00:00.000 ✅
Client B (+5 minutes):   sends event with ts = 10:05:00.000 ❌ (future!)
Client C (-10 minutes):  sends event with ts = 09:50:00.000 ❌ (too old!)
```

**Impact:**

* Future events: May create buckets that shouldn't exist yet
* Past events: May be rejected as "late" even when sent on time
* Aggregation: Windows become meaningless if client times are arbitrary

***

**Strategy A: Reject Extreme Skew**

```python
class SkewAwareAggregator:
    MAX_FUTURE_SKEW_MS = 30_000    # 30 seconds into future
    MAX_PAST_SKEW_MS = 300_000     # 5 minutes into past
    
    def add_event(self, event_ts: int, value: float) -> Result:
        server_now = int(time.time() * 1000)
        skew = event_ts - server_now
        
        if skew > self.MAX_FUTURE_SKEW_MS:
            self.metrics.future_skew_rejected.inc()
            return Result.REJECTED_FUTURE
        
        if skew < -self.MAX_PAST_SKEW_MS:
            self.metrics.past_skew_rejected.inc()
            return Result.REJECTED_PAST
        
        # Proceed with normal processing
        return self._add_to_bucket(event_ts, value)
```

***

**Strategy B: Server-Side Timestamp Override**

```python
class ServerTimestampAggregator:
    """Ignore client timestamp, use server receive time."""
    
    def add_event(self, event: Event) -> Result:
        # Ignore event.timestamp, use current server time
        server_ts = int(time.time() * 1000)
        return self._add_to_bucket(server_ts, event.value)
```

**Pros:** Immune to client clock issues **Cons:** Loses "when it actually happened" semantics

***

**Strategy C: Hybrid with Skew Correction**

```python
class SkewCorrectedAggregator:
    """Track per-client skew and correct timestamps."""
    
    def __init__(self):
        self.client_skew: dict[str, float] = {}  # client_id → avg_skew_ms
        self.skew_window = 100  # Rolling average over 100 samples
    
    def add_event(self, event: Event) -> Result:
        server_now = int(time.time() * 1000)
        observed_skew = event.timestamp - server_now
        
        # Update rolling average skew for this client
        client_id = event.client_id
        if client_id not in self.client_skew:
            self.client_skew[client_id] = observed_skew
        else:
            alpha = 1 / self.skew_window
            self.client_skew[client_id] = (
                (1 - alpha) * self.client_skew[client_id] + 
                alpha * observed_skew
            )
        
        # Correct timestamp using estimated skew
        corrected_ts = event.timestamp - int(self.client_skew[client_id])
        
        return self._add_to_bucket(corrected_ts, event.value)
```

**Pros:** Adapts to each client's clock drift **Cons:** Complex, needs per-client state

***

**Strategy D: NTP/Time Sync Requirements**

```
Architecture-level solution:

┌─────────┐     NTP Sync      ┌─────────────┐
│ Client  │ ◄───────────────► │ NTP Server  │
└────┬────┘                   └─────────────┘
     │                              ▲
     │ Events                       │ NTP Sync
     ▼                              │
┌─────────┐                   ┌─────┴─────┐
│ Server  │ ◄────────────────►│ NTP Server│
└─────────┘                   └───────────┘

With NTP: Client and server within ~10ms of each other
```

**Best practice:** Require NTP sync and still handle edge cases:

```python
class NTPAwareAggregator:
    # Assume NTP keeps everyone within 100ms
    EXPECTED_MAX_SKEW_MS = 100
    
    # But allow for NTP failures
    HARD_LIMIT_SKEW_MS = 60_000
    
    def add_event(self, event: Event) -> Result:
        skew = abs(event.timestamp - server_now)
        
        if skew > self.HARD_LIMIT_SKEW_MS:
            return Result.REJECTED  # Clock is broken
        
        if skew > self.EXPECTED_MAX_SKEW_MS:
            self.metrics.skew_warning.inc(
                labels={"client": event.client_id}
            )
            # Alert: client may have NTP issues
        
        # Process normally - small skew is OK
        return self._add_to_bucket(event.timestamp, event.value)
```

***

**Monitoring clock skew:**

```python
# Essential metrics for clock skew visibility
metrics = {
    # Per-client skew tracking
    "client_clock_skew_seconds": Histogram(
        buckets=[0.01, 0.1, 1, 5, 30, 60, 300]
    ),
    
    # Anomaly counters
    "events_rejected_future_skew": Counter(),
    "events_rejected_past_skew": Counter(),
    
    # Client health
    "clients_with_skew_gt_1s": Gauge(),
}
```

***

**Decision matrix:**

| Strategy         | Accuracy    | Complexity | Best For                       |
| ---------------- | ----------- | ---------- | ------------------------------ |
| Reject extreme   | Good        | Simple     | Controlled environments        |
| Server timestamp | Approximate | Simple     | When event time doesn't matter |
| Skew correction  | Good        | Complex    | IoT, mobile (unreliable NTP)   |
| Require NTP      | Excellent   | Low        | Enterprise/datacenter          |

***

**Best Practice Recommendation:**

```yaml
clock_skew_handling:
  # First line of defense: require NTP
  require_ntp: true
  expected_max_skew_ms: 100
  
  # Second line: soft handling
  soft_future_limit_ms: 5000    # 5s future = warn
  soft_past_limit_ms: 60000     # 1m past = warn
  
  # Hard limits
  hard_future_limit_ms: 60000   # 1m future = reject
  hard_past_limit_ms: 300000    # 5m past = reject
  
  # Metrics
  emit_skew_histogram: true
  alert_threshold_clients_with_skew: 10  # Alert if 10+ clients drifting
```

***

### Conclusion

Designing window aggregation for stream processing requires balancing multiple trade-offs:

1. **Time semantics**: Wall-clock for simplicity, event-time for accuracy, hybrid for the best of both
2. **Aggregation precision**: Exact for correctness-critical paths, approximate for scale
3. **Late event handling**: Policy should match business requirements, always with observability

The "right" answer depends on your specific requirements. Start with the simplest approach (hybrid time + exact aggregation + log-and-drop) and evolve based on observed late event rates and accuracy needs.

**Key takeaways:**

* Always expose metrics on late events—you can't fix what you can't see
* The 60-second tolerance in our example isn't arbitrary; tune it based on observed P99 network delays
* Approximate algorithms are underused; HyperLogLog and T-Digest should be in every streaming engineer's toolkit
* Design for replay from day one if you might need historical accuracy

***

_Have questions or war stories about streaming aggregation? I'd love to hear about edge cases you've encountered._

```
```
