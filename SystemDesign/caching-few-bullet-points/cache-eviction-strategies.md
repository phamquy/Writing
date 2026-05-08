---
icon: pancakes
---

# Cache Eviction Strategies

Caching is a fundamental technique in system design to improve performance by storing frequently accessed data in fast storage. When the cache reaches its capacity, an **eviction policy** determines which items to remove to make room for new entries.

***

### 1. LRU (Least Recently Used)

**How it works:** Evicts the item that hasn't been accessed for the longest time.

**Implementation:**

* Typically uses a **HashMap + Doubly Linked List**
* HashMap provides O(1) lookup
* Doubly linked list maintains access order

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # Mark as recently used
        return self.cache[key]
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # Remove least recently used
```

**Pros:**

* Simple to understand and implement
* Works well when recent data is more likely to be accessed again
* O(1) time complexity for both get and put operations

**Cons:**

* Doesn't consider frequency of access
* Can evict frequently used items if they haven't been accessed recently
* Susceptible to scan pollution (one-time sequential scans can flush useful cache)

**When to use:**

* _Web browser caches_ because it mirrors natural browsing behavior where users frequently revisit recently accessed websites, pages, and resources. When users navigate between tabs, return to previously visited sites, or reload pages, the recently cached content (HTML, CSS, JavaScript, images) is likely to be needed again soon, making LRU's recency-based eviction highly effective. Browser usage patterns exhibit strong temporal locality - users tend to work within a set of active websites during a session, repeatedly accessing the same resources as they navigate between related pages or refresh content. LRU efficiently maintains this "working set" of active web resources in the limited cache space while automatically evicting older, unused content that's less likely to be accessed again. The algorithm also handles the unpredictable nature of web browsing well, adapting to changing user interests by promoting newly accessed sites while gradually phasing out abandoned ones, making it a natural fit for the dynamic and user-driven access patterns typical in web browsing scenarios.
* _Database buffer pools_: LRU is an excellent strategy for database buffer pools because it leverages temporal locality - the principle that recently accessed data is likely to be accessed again soon. Database workloads naturally exhibit this pattern through repeated queries on active records, index traversals, and transaction processing where the same pages are referenced multiple times within short time windows. LRU effectively maintains the "hot" working set of frequently accessed pages in memory while evicting cold pages that haven't been used recently, which aligns well with both OLTP workloads (where recent user activity and active transactions dominate) and OLAP workloads (where sequential scans benefit from keeping recently read blocks cached). The algorithm is simple to implement with low overhead, making it a practical choice that works well for most real-world database access patterns, though some systems enhance it with variations like LRU-K to handle edge cases like large sequential scans that might otherwise pollute the buffer pool.
* General-purpose caching where temporal locality is strong

***

### 2. LFU (Least Frequently Used)

**How it works:** Evicts the item with the lowest access frequency. Ties are broken by LRU.

**Implementation:**

* Uses **HashMap + Frequency Map + Min Heap or Doubly Linked Lists**

```python
from collections import defaultdict

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val = {}
        self.key_to_freq = {}
        self.freq_to_keys = defaultdict(OrderedDict)
    
    def get(self, key: int) -> int:
        if key not in self.key_to_val:
            return -1
        self._update_freq(key)
        return self.key_to_val[key]
    
    def put(self, key: int, value: int) -> None:
        if self.capacity <= 0:
            return
        if key in self.key_to_val:
            self.key_to_val[key] = value
            self._update_freq(key)
            return
        if len(self.key_to_val) >= self.capacity:
            # Evict LFU (and LRU among same frequency)
            evict_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            del self.key_to_val[evict_key]
            del self.key_to_freq[evict_key]
        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1
    
    def _update_freq(self, key: int):
        freq = self.key_to_freq[key]
        self.key_to_freq[key] += 1
        del self.freq_to_keys[freq][key]
        if not self.freq_to_keys[freq] and self.min_freq == freq:
            self.min_freq += 1
        self.freq_to_keys[freq + 1][key] = None
```

**Pros:**

* Keeps frequently accessed items in cache
* Better for workloads with varying popularity

**Cons:**

* More complex to implement
* Higher memory overhead (tracking frequencies)
* Slow to adapt to changing access patterns
* New items are vulnerable to eviction before building up frequency

**When to use:**

* _CDN caching (popular content stays cached)_: because content popularity follows a _power-law distribution_ — a small percentage of content (viral videos, trending articles, popular images) accounts for the vast majority of requests. Unlike LRU, which might evict a consistently popular asset just because it wasn't accessed in the last few seconds, LFU keeps high-frequency items cached regardless of short gaps between requests. A viral video that gets 10,000 hits per hour will maintain a high frequency count and stay cached even during brief quiet periods, while one-time accessed content naturally gets evicted first. This aligns perfectly with CDN economics: serving popular content from edge cache dramatically reduces origin server load and bandwidth costs. The "slow to adapt" weakness of LFU is actually acceptable here since content popularity typically changes gradually (hours/days), not suddenly.
* _Recommendation systems_: user preferences and item popularity tend to be stable over time rather than volatile. Items that users frequently interact with (watched movies, purchased products, clicked articles) represent genuine preferences that should remain cached for future recommendations. Unlike browsing behavior where recency matters, recommendation engines need to remember that a user has watched a particular genre 50 times over the past month — even if their last watch was a week ago. LFU naturally preserves these high-frequency interaction patterns. Additionally, recommendation systems often cache item embeddings and feature vectors for popular items; LFU ensures the most frequently requested items (trending products, popular movies) stay in cache, reducing expensive model inference or database lookups. The "slow to adapt" characteristic of LFU is actually beneficial here — you don't want a user's long-term preferences flushed just because they briefly explored something new.
* _Scenarios where some items are consistently more popular_: LFU provides more consistent cache performance compared to LRU because frequency is inherently more stable than recency. A single access pattern spike won't drastically change what's cached — items with high historical access counts remain protected even during brief quiet periods, whereas LRU can flip-flop rapidly based on just the last few accesses. LFU is also resistant to one-time bursts: if a popular item has been accessed 1,000 times and a new item is accessed once, LRU might evict the popular item simply because the new one was accessed more recently, while LFU correctly keeps the high-value item cached. The eviction order in LFU is predictable (always the lowest frequency items), making it easier to reason about cache behavior compared to LRU where eviction depends on exact timing. This leads to a smoother performance curve where cache hit rates don't swing wildly based on recent access patterns — crucial for SLA guarantees where consistent latency matters. The trade-off is that LFU is slow to adapt when access patterns genuinely change; a previously popular item that's no longer relevant will stubbornly stay cached until enough new items accumulate higher frequencies. This is why LFU works best when popularity is inherently stable (CDNs, recommendation systems) rather than rapidly changing (user browsing sessions).

***

### 3. FIFO (First In, First Out)

**How it works:** Evicts the oldest item in the cache (first inserted).

**Implementation:**

* Simple **Queue (deque)** + HashMap

```python
from collections import deque

class FIFOCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.queue = deque()
    
    def get(self, key: int) -> int:
        return self.cache.get(key, -1)
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache[key] = value
            return
        if len(self.cache) >= self.capacity:
            oldest = self.queue.popleft()
            del self.cache[oldest]
        self.cache[key] = value
        self.queue.append(key)
```

**Pros:**

* Very simple to implement
* Low overhead
* Predictable behavior

**Cons:**

* Doesn't consider access patterns at all
* May evict frequently used items

**When to use:**

* _Simple embedded systems_: FIFO is ideal for embedded systems with limited computational resources (microcontrollers, IoT devices, sensors) because it requires minimal logic — just a queue pointer with no access tracking or frequency counting. The overhead of maintaining LRU's doubly linked list or LFU's frequency maps would consume precious memory and CPU cycles that embedded systems can't spare. FIFO's predictable, constant-time eviction with zero bookkeeping makes it suitable for devices where simplicity and reliability outweigh cache optimization. When memory is measured in kilobytes and every instruction counts, FIFO's "good enough" eviction is often the only practical choice.
* _When items have natural expiration based on age_: FIFO naturally aligns with data that becomes stale over time regardless of how often it's accessed. For example, log entries, audit trails, or streaming sensor data are inherently time-ordered — older entries lose relevance not because they're unpopular, but simply because newer data supersedes them. In these scenarios, FIFO's "oldest out first" behavior is actually the _**correct semantic**_, not just a simplification. Using LRU or LFU would be counterproductive since accessing old data shouldn't extend its lifetime when freshness is what matters.
* _CPU instruction caches (in some architectures)_: Some CPU instruction caches use FIFO because instruction fetch patterns often exhibit sequential locality — programs execute instructions in order, making recently _inserted_ instructions (not recently _accessed_) good predictors of future needs. FIFO is also trivial to implement in hardware with a simple circular buffer and head pointer, requiring no tag comparisons or access tracking logic. While modern high-performance CPUs use pseudo-LRU for better hit rates, simpler processors and embedded CPUs still use FIFO where gate count and power consumption are critical constraints.

***

### 5. Random Replacement

**How it works:** Randomly selects an item to evict.

```python
import random

class RandomCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.keys = []
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache[key] = value
            return
        if len(self.cache) >= self.capacity:
            evict_key = random.choice(self.keys)
            self.keys.remove(evict_key)
            del self.cache[evict_key]
        self.cache[key] = value
        self.keys.append(key)
```

**Pros:**

* Extremely simple
* No tracking overhead
* Works surprisingly well in some scenarios

**Cons:**

* Unpredictable performance
* May evict important items

**When to use:**

* When simplicity is paramount
* Hardware caches with limited logic

***

### 6. TTL (Time-To-Live)

**How it works:** Items expire after a fixed time period regardless of access patterns.

```python
import time

class TTLCache:
    def __init__(self, capacity: int, ttl: int):
        self.capacity = capacity
        self.ttl = ttl
        self.cache = {}  # key -> (value, expiry_time)
    
    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        value, expiry = self.cache[key]
        if time.time() > expiry:
            del self.cache[key]
            return -1
        return value
    
    def put(self, key: int, value: int) -> None:
        self._cleanup()
        self.cache[key] = (value, time.time() + self.ttl)
    
    def _cleanup(self):
        now = time.time()
        expired = [k for k, (v, exp) in self.cache.items() if now > exp]
        for k in expired:
            del self.cache[k]
```

**Pros:**

* Ensures data freshness
* Simple mental model
* Good for time-sensitive data

**Cons:**

* Popular items may expire unnecessarily
* Requires clock synchronization in distributed systems

**When to use:**

* Session caching
* DNS caching
* API response caching
* Any data that becomes stale over time

***

### 7. ARC (Adaptive Replacement Cache)

**How it works:** Combines LRU and LFU by maintaining four lists:

* **T1**: Recently accessed items (LRU)
* **T2**: Frequently accessed items (LFU)
* **B1**: Ghost entries evicted from T1
* **B2**: Ghost entries evicted from T2

ARC maintains two pairs of lists: T1/T2 for actual cached data and B1/B2 as "ghost" lists that only track keys of recently evicted items. T1 holds items accessed only once (recency-focused), while T2 holds items accessed multiple times (frequency-focused). The key innovation is the adaptive parameter p, which determines how much of the cache is dedicated to T1 vs T2. When a ghost hit occurs in B1 (recently evicted items are being re-requested), ARC increases p to allocate more space to recency. Conversely, a ghost hit in B2 causes ARC to decrease p, favoring frequency. This self-tuning mechanism allows ARC to automatically adapt to workload patterns—performing like LRU for recency-heavy workloads and like LFU for frequency-heavy ones—without manual configuration.

**Implementation:**

```python
from collections import OrderedDict

class ARCCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.p = 0  # Target size for T1 (adaptive parameter)
        self.t1 = OrderedDict()  # Recent items (accessed once)
        self.t2 = OrderedDict()  # Frequent items (accessed more than once)
        self.b1 = OrderedDict()  # Ghost entries evicted from T1
        self.b2 = OrderedDict()  # Ghost entries evicted from T2
    
    def _replace(self, key):
        """Move item from T1 or T2 to corresponding ghost list."""
        if self.t1 and (
            (key in self.b2 and len(self.t1) == self.p) or
            (len(self.t1) > self.p)
        ):
            # Evict from T1 to B1
            old_key, old_val = self.t1.popitem(last=False)
            self.b1[old_key] = None
        elif self.t2:
            # Evict from T2 to B2
            old_key, old_val = self.t2.popitem(last=False)
            self.b2[old_key] = None
    
    def get(self, key: int) -> int:
        # Case 1: Key in T1 (recent) -> promote to T2 (frequent)
        if key in self.t1:
            value = self.t1.pop(key)
            self.t2[key] = value
            self.t2.move_to_end(key)
            return value
        
        # Case 2: Key in T2 (frequent) -> move to end (most recent)
        if key in self.t2:
            self.t2.move_to_end(key)
            return self.t2[key]
        
        return -1
    
    def put(self, key: int, value: int) -> None:
        # Case 1: Key in T1 -> promote to T2
        if key in self.t1:
            self.t1.pop(key)
            self.t2[key] = value
            self.t2.move_to_end(key)
            return
        
        # Case 2: Key in T2 -> update and move to end
        if key in self.t2:
            self.t2[key] = value
            self.t2.move_to_end(key)
            return
        
        # Case 3: Key in B1 (ghost of recent) -> adapt toward recency
        if key in self.b1:
            # Increase p: favor keeping recent items
            delta = max(1, len(self.b2) // max(1, len(self.b1)))
            self.p = min(self.p + delta, self.capacity)
            self._replace(key)
            self.b1.pop(key)
            self.t2[key] = value  # Insert to T2 (now frequent)
            return
        
        # Case 4: Key in B2 (ghost of frequent) -> adapt toward frequency
        if key in self.b2:
            # Decrease p: favor keeping frequent items
            delta = max(1, len(self.b1) // max(1, len(self.b2)))
            self.p = max(self.p - delta, 0)
            self._replace(key)
            self.b2.pop(key)
            self.t2[key] = value  # Insert to T2 (now frequent)
            return
        
        # Case 5: Cache miss, not in any list
        total_t = len(self.t1) + len(self.t2)
        total_b1 = len(self.b1)
        
        if total_t == self.capacity:
            # Cache is full
            if len(self.t1) < self.capacity:
                # B1 has room, just replace
                if total_b1 + total_t == 2 * self.capacity and self.b1:
                    self.b1.popitem(last=False)  # Remove oldest ghost
                self._replace(key)
            else:
                # T1 is full, evict from T1
                self.t1.popitem(last=False)
        elif total_t < self.capacity and total_b1 + total_t >= self.capacity:
            if total_b1 + total_t == 2 * self.capacity and self.b1:
                self.b1.popitem(last=False)
            elif len(self.b2) > 0 and total_b1 + len(self.b2) + total_t >= 2 * self.capacity:
                self.b2.popitem(last=False)
        
        # Insert new item into T1
        self.t1[key] = value
```

**Pros:**

* Self-tuning based on workload
* Better hit rate than LRU/LFU alone
* Scan-resistant

**Cons:**

* Complex implementation
* Higher memory overhead (ghost lists)
* Patented (IBM)

**When to use:**

* File system caches (ZFS uses ARC)
* Database systems
* High-performance storage systems

***

### 8. 2Q (Two Queue)

**How it works:** Uses two queues:

* **A1in**: FIFO queue for new items
* **Am**: LRU queue for items accessed more than once

2Q protects the cache from scan pollution (one-time sequential reads) by making items "prove" they deserve long-term caching. New items enter A1in, a FIFO queue (typically 25% of capacity). If an item is accessed only once and then evicted from A1in, its key goes to the A1out ghost list. If that same key is requested again (ghost hit), it's promoted to Am, the main LRU queue (75% of capacity). Items that never get re-accessed simply flow through A1in and disappear — they never pollute Am. This means a one-time table scan won't flush your frequently-accessed hot data from the cache. Unlike ARC, 2Q uses a fixed ratio between queues (no adaptive tuning), making it simpler but less flexible to varying workloads.

**Implementation:**

```python
from collections import OrderedDict, deque

class TwoQueueCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        # A1in: FIFO queue for new items (25% of capacity is typical)
        self.a1in_capacity = max(1, capacity // 4)
        self.am_capacity = capacity - self.a1in_capacity
        
        self.a1in = deque()  # FIFO queue for first-time accessed items
        self.a1in_set = set()  # For O(1) lookup in A1in
        self.a1out = OrderedDict()  # Ghost list for evicted A1in items (keys only)
        self.am = OrderedDict()  # LRU queue for frequently accessed items
        self.values = {}  # Actual key-value storage
    
    def get(self, key: int) -> int:
        if key not in self.values:
            return -1
        
        # If in Am, move to end (most recently used)
        if key in self.am:
            self.am.move_to_end(key)
        # If in A1in, don't change position (FIFO behavior)
        # Promotion happens on second access after eviction from A1in
        
        return self.values[key]
    
    def put(self, key: int, value: int) -> None:
        # Case 1: Key already exists
        if key in self.values:
            self.values[key] = value
            if key in self.am:
                self.am.move_to_end(key)
            return
        
        # Case 2: Key in A1out (ghost list) -> promote to Am
        if key in self.a1out:
            del self.a1out[key]
            self._add_to_am(key, value)
            return
        
        # Case 3: New key -> add to A1in
        self._add_to_a1in(key, value)
    
    def _add_to_a1in(self, key: int, value: int):
        """Add new item to A1in (FIFO queue)."""
        # Evict from A1in if full
        while len(self.a1in) >= self.a1in_capacity:
            evicted = self.a1in.popleft()
            self.a1in_set.remove(evicted)
            del self.values[evicted]
            # Add to ghost list A1out
            self.a1out[evicted] = None
            # Limit ghost list size
            if len(self.a1out) > self.capacity:
                self.a1out.popitem(last=False)
        
        self.a1in.append(key)
        self.a1in_set.add(key)
        self.values[key] = value
    
    def _add_to_am(self, key: int, value: int):
        """Add item to Am (LRU queue) - for promoted items."""
        # Evict from Am if full
        while len(self.am) >= self.am_capacity:
            evicted, _ = self.am.popitem(last=False)
            del self.values[evicted]
        
        self.am[key] = None
        self.values[key] = value
```

**Pros:**

* Scan-resistant
* Simple than ARC
* Better than pure LRU

**Cons:**

* Fixed ratio between queues
* Doesn't adapt to workload

**When to use:**

* Database buffer pools
* File system caches

***

### Comparison Summary

| Strategy | Time Complexity | Space Overhead | Adaptability | Best For            |
| -------- | --------------- | -------------- | ------------ | ------------------- |
| LRU      | O(1)            | Low            | Medium       | General purpose     |
| LFU      | O(1)\*          | Medium         | Low          | Popular content     |
| FIFO     | O(1)            | Very Low       | None         | Simple systems      |
| Random   | O(1)            | Very Low       | None         | Hardware caches     |
| TTL      | O(1)            | Low            | None         | Time-sensitive data |
| ARC      | O(1)            | High           | High         | File systems        |
| 2Q       | O(1)            | Medium         | Medium       | Databases           |

\*O(log n) with naive implementation, O(1) with optimized structures

***

### Real-World Usage

| System            | Eviction Strategy                    |
| ----------------- | ------------------------------------ |
| Redis             | LRU, LFU, Random, TTL (configurable) |
| Memcached         | LRU                                  |
| CPU Cache (Intel) | Pseudo-LRU                           |
| ZFS               | ARC                                  |
| Linux Page Cache  | Two-list approximation               |
| CDNs (CloudFlare) | LRU + TTL                            |
| Browser Cache     | LRU + TTL                            |
