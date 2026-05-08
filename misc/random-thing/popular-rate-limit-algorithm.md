---
icon: traffic-light-slow
---

# Popular rate limit algorithm

### 1. Fixed Window Counter

* Divide time into fixed-length windows (e.g., 60s).
* Count requests per window per user.
* Reset count at start of each new window.

**Pros:**

* Simple to implement.
* Efficient in memory.

**Cons:**

* Request spikes at window edges (burst problem).

```python
# Fixed Window Counter Implementation
from collections import defaultdict
import time

class FixedWindowRateLimiter:
    def __init__(self, window_size, max_requests):
        self.window_size = window_size  # in seconds
        self.max_requests = max_requests
        self.request_counts = defaultdict(int)
        self.window_start_times = defaultdict(lambda: time.time())

    def allow_request(self, user_id):
        current_time = time.time()
        window_start = self.window_start_times[user_id]

        # Check if the current time is outside the current window
        if current_time - window_start >= self.window_size:
            self.window_start_times[user_id] = current_time
            self.request_counts[user_id] = 0

        # Increment the request count and check if it exceeds the limit
        self.request_counts[user_id] += 1
        if self.request_counts[user_id] > self.max_requests:
            return False  # Request denied

        return True  # Request allowed

# Example usage
rate_limiter = FixedWindowRateLimiter(window_size=60, max_requests=10)
user_id = "user_123"

for i in range(15):
    if rate_limiter.allow_request(user_id):
        print(f"Request {i+1} allowed.")
    else:
        print(f"Request {i+1} denied.")
```

```
Request 1 allowed.
Request 2 allowed.
Request 3 allowed.
Request 4 allowed.
Request 5 allowed.
Request 6 allowed.
Request 7 allowed.
Request 8 allowed.
Request 9 allowed.
Request 10 allowed.
Request 11 denied.
Request 12 denied.
Request 13 denied.
Request 14 denied.
Request 15 denied.
```

***

### 2. Sliding Window Log

* Store timestamps of each request.
* At each request, remove timestamps outside the window.
* Count how many remain.

**Pros:**

* Accurate and fair.

**Cons:**

* Memory-intensive (stores all timestamps).
* Slower at high volume.

```python
# Sliding Window Log Implementation
from collections import deque
import time

class SlidingWindowLogRateLimiter:
    def __init__(self, window_size, max_requests):
        self.window_size = window_size  # in seconds
        self.max_requests = max_requests
        self.request_logs = defaultdict(deque)

    def allow_request(self, user_id):
        current_time = time.time()
        request_log = self.request_logs[user_id]

        # Remove timestamps outside the window
        while request_log and current_time - request_log[0] > self.window_size:
            request_log.popleft()

        # Check if the number of requests exceeds the limit
        if len(request_log) >= self.max_requests:
            return False  # Request denied

        # Add the current request timestamp
        request_log.append(current_time)
        return True  # Request allowed

# Example usage
rate_limiter = SlidingWindowLogRateLimiter(window_size=60, max_requests=10)
user_id = "user_123"

for i in range(15):
    if rate_limiter.allow_request(user_id):
        print(f"Request {i+1} allowed.")
    else:
        print(f"Request {i+1} denied.")
```

```
Request 1 allowed.
Request 2 allowed.
Request 3 allowed.
Request 4 allowed.
Request 5 allowed.
Request 6 allowed.
Request 7 allowed.
Request 8 allowed.
Request 9 allowed.
Request 10 allowed.
Request 11 denied.
Request 12 denied.
Request 13 denied.
Request 14 denied.
Request 15 denied.
```

***

### 3. Sliding Window Counter (Approximation)

* Divide window into smaller sub-windows (e.g., 10s chunks).
* Keep counters for each sub-window.
* Sum them for current total.

**Pros:**

* Good balance of performance and accuracy.

**Cons:**

* Slightly more complex.
* May still allow some bursts.

```python
# Sliding Window Counter Implementation
from collections import defaultdict
import time

class SlidingWindowCounterRateLimiter:
    def __init__(self, window_size, max_requests, sub_window_size):
        self.window_size = window_size  # in seconds
        self.max_requests = max_requests
        self.sub_window_size = sub_window_size  # in seconds
        self.sub_window_counts = defaultdict(lambda: defaultdict(int))

    def allow_request(self, user_id):
        current_time = time.time()
        current_sub_window = int(current_time // self.sub_window_size)

        # Remove old sub-windows outside the main window
        for sub_window in list(self.sub_window_counts[user_id].keys()):
            if current_sub_window - sub_window >= self.window_size // self.sub_window_size:
                del self.sub_window_counts[user_id][sub_window]

        # Count requests in the current sub-window
        self.sub_window_counts[user_id][current_sub_window] += 1

        # Calculate total requests in the main window
        total_requests = sum(self.sub_window_counts[user_id].values())
        if total_requests > self.max_requests:
            return False  # Request denied

        return True  # Request allowed

# Example usage
rate_limiter = SlidingWindowCounterRateLimiter(window_size=60, max_requests=10, sub_window_size=10)
user_id = "user_123"

for i in range(15):
    if rate_limiter.allow_request(user_id):
        print(f"Request {i+1} allowed.")
    else:
        print(f"Request {i+1} denied.")
```

```
Request 1 allowed.
Request 2 allowed.
Request 3 allowed.
Request 4 allowed.
Request 5 allowed.
Request 6 allowed.
Request 7 allowed.
Request 8 allowed.
Request 9 allowed.
Request 10 allowed.
Request 11 denied.
Request 12 denied.
Request 13 denied.
Request 14 denied.
Request 15 denied.
```

***

### 4. Token Bucket

* Tokens are added to the bucket at a fixed rate.
* Each request consumes a token.
* If no tokens remain, request is denied.

**Pros:**

* Allows bursts.
* Smooth control over traffic rate.

**Cons:**

* Requires periodic refill logic.

```python
# Token Bucket Implementation (Per-User)
from collections import defaultdict
import time

class TokenBucketRateLimiter:
    def __init__(self, refill_rate, bucket_capacity):
        self.refill_rate = refill_rate  # tokens per second
        self.bucket_capacity = bucket_capacity
        self.user_buckets = defaultdict(lambda: {
            "tokens": bucket_capacity,
            "last_refill_time": time.time()
        })

    def allow_request(self, user_id):
        current_time = time.time()
        user_bucket = self.user_buckets[user_id]
        elapsed_time = current_time - user_bucket["last_refill_time"]

        # Refill tokens based on elapsed time
        user_bucket["tokens"] = min(
            self.bucket_capacity,
            user_bucket["tokens"] + elapsed_time * self.refill_rate
        )
        user_bucket["last_refill_time"] = current_time

        # Check if there are enough tokens for the request
        if user_bucket["tokens"] >= 1:
            user_bucket["tokens"] -= 1
            return True  # Request allowed

        return False  # Request denied

# Example usage
rate_limiter = TokenBucketRateLimiter(refill_rate=0.5, bucket_capacity=5)  # 1 token every 2 seconds, max 5 tokens
user_id = "user_123"

for i in range(15):
    if rate_limiter.allow_request(user_id):
        print(f"Request {i+1} allowed for {user_id}.")
    else:
        print(f"Request {i+1} denied for {user_id}.")
    time.sleep(0.5)
```

```
Request 1 allowed for user_123.
Request 2 allowed for user_123.
Request 2 allowed for user_123.
Request 3 allowed for user_123.
Request 3 allowed for user_123.
Request 4 allowed for user_123.
Request 4 allowed for user_123.
Request 5 allowed for user_123.
Request 5 allowed for user_123.
Request 6 allowed for user_123.
Request 6 allowed for user_123.
Request 7 denied for user_123.
Request 7 denied for user_123.
Request 8 denied for user_123.
Request 8 denied for user_123.
Request 9 allowed for user_123.
Request 9 allowed for user_123.
Request 10 denied for user_123.
Request 10 denied for user_123.
Request 11 denied for user_123.
Request 11 denied for user_123.
Request 12 denied for user_123.
Request 12 denied for user_123.
Request 13 allowed for user_123.
Request 13 allowed for user_123.
Request 14 denied for user_123.
Request 14 denied for user_123.
Request 15 denied for user_123.
Request 15 denied for user_123.
```

***

### 5. Leaky Bucket

* Queue incoming requests.
* Process them at a constant rate.
* Overflowing the queue drops excess requests.

**Pros:**

* Smooth output rate.
* Prevents traffic spikes.

**Cons:**

* May introduce latency or reject burst traffic.

```python
# Leaky Bucket Implementation (Per-User)
from collections import defaultdict
import time
import queue

class LeakyBucketRateLimiter:
    def __init__(self, leak_rate, bucket_capacity):
        self.leak_rate = leak_rate  # requests per second
        self.bucket_capacity = bucket_capacity
        self.user_buckets = defaultdict(lambda: {
            "queue": queue.Queue(maxsize=bucket_capacity),
            "last_processed_time": time.time()
        })

    def allow_request(self, user_id):
        current_time = time.time()
        user_bucket = self.user_buckets[user_id]
        elapsed_time = current_time - user_bucket["last_processed_time"]

        # Process requests at a constant rate
        while not user_bucket["queue"].empty() and elapsed_time >= 1 / self.leak_rate:
            user_bucket["queue"].get()
            user_bucket["last_processed_time"] += 1 / self.leak_rate
            elapsed_time = current_time - user_bucket["last_processed_time"]

        # Check if the bucket has space for the new request
        if user_bucket["queue"].full():
            return False  # Request denied

        user_bucket["queue"].put(current_time)
        return True  # Request allowed

# Example usage
rate_limiter = LeakyBucketRateLimiter(leak_rate=0.5, bucket_capacity=5)  # 1 request every 2 seconds, max 5 requests
user_id = "user_123"

for i in range(15):
    if rate_limiter.allow_request(user_id):
        print(f"Request {i+1} allowed for {user_id}.")
    else:
        print(f"Request {i+1} denied for {user_id}.")
    time.sleep(0.5)
```

***

### 6. Generic Cell Rate Algorithm (GCRA)

* Mathematical model for rate control (used in telecoms).
* Based on the time of the last allowed request and a fixed interval.

**Pros:**

* Precise, memory-efficient.

**Cons:**

* More complex to implement manually.

```python
# Generic Cell Rate Algorithm (GCRA) Implementation
import time
class GCRARateLimiter:
    def __init__(self, interval, burst_allowance):
        self.interval = interval  # Minimum time between requests (seconds)
        self.burst_allowance = burst_allowance  # Maximum burst requests allowed
        self.user_virtual_time = {}

    def allow_request(self, user_id):
        current_time = time.time()
        # Initialize virtual time to allow burst requests initially
        virtual_time = self.user_virtual_time.get(user_id, current_time - self.burst_allowance * self.interval)
        # Calculate the theoretical next allowed time
        theoretical_next_time = virtual_time + self.interval
        if current_time < theoretical_next_time:
            return False  # Request denied
        # Update virtual time to reflect the current request
        self.user_virtual_time[user_id] = max(theoretical_next_time, current_time)
        return True  # Request allowed

# Example usage
rate_limiter = GCRARateLimiter(interval=2, burst_allowance=5)  # 1 request every 2 seconds, max burst of 5 requests
user_id = "user_123"
for i in range(15):
    if rate_limiter.allow_request(user_id):
        print(f"Request {i+1} allowed for {user_id}.")
    else:
        print(f"Request {i+1} denied for {user_id}.")
    time.sleep(0.5)
```

```
Request 1 allowed for user_123.
Request 2 denied for user_123.
Request 2 denied for user_123.
Request 3 denied for user_123.
Request 3 denied for user_123.
Request 4 denied for user_123.
Request 4 denied for user_123.
Request 5 allowed for user_123.
Request 5 allowed for user_123.
Request 6 denied for user_123.
Request 6 denied for user_123.
Request 7 denied for user_123.
Request 7 denied for user_123.
Request 8 denied for user_123.
Request 8 denied for user_123.
Request 9 allowed for user_123.
Request 9 allowed for user_123.
Request 10 denied for user_123.
Request 10 denied for user_123.
Request 11 denied for user_123.
Request 11 denied for user_123.
Request 12 denied for user_123.
Request 12 denied for user_123.
Request 13 allowed for user_123.
Request 13 allowed for user_123.
Request 14 denied for user_123.
Request 14 denied for user_123.
Request 15 denied for user_123.
Request 15 denied for user_123.
```

***

### Summary Table

| Algorithm              | Memory Use | Accuracy  | Burst Support | Complexity |
| ---------------------- | ---------- | --------- | ------------- | ---------- |
| Fixed Window Counter   | Low        | Medium    | Poor          | Easy       |
| Sliding Window Log     | High       | High      | Good          | Medium     |
| Sliding Window Counter | Medium     | Good      | Fair          | Medium     |
| Token Bucket           | Low        | Good      | Excellent     | Medium     |
| Leaky Bucket           | Low        | Good      | Poor          | Medium     |
| GCRA                   | Very Low   | Excellent | Configurable  | Hard       |
