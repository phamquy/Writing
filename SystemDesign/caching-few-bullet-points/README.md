---
icon: pancakes
---

# Caching - few bullet points

### Concept

Store copies of expensive-to-compute or slow-to-fetch data closer to consumers to cut latency and infra load.

Being closer here meaning that accessing cache will faster than accessing the data source.

* Reading CPU L1/2/3 cache is faster than reading from RAM
* Reading from RAM faster than reading from harddisk
* Reading from local hard disk faster than pulling data from remote server or data base
* Readind date from near CDN is faster than original server located in different continents
* ...

### How it works

Everytime data need to be read, server check the (closest) cache first, if the cache has the data (cache hit) return it. if cache does not have it (cache miss), data will be pulled from further source (furher cache, remote source...) to return data as fast as possible.

There are two somewhat competing goals when working with cache: (1) maximize the cache hit (or minimize cache miss) vs. (2) prevent data staleness (or maintain data freshness)

#### Maximize cache hit

More cache hit meaning faster data retrieval, less access to slower source. It reduce the traffic to source beyond cache layer, and improve overall performance and service availability.

Higher hit ratios also keep tail latencies predictable, since most requests short-circuit before touching overloaded origins. Fewer origin reads mean lower infra spend, more headroom for genuine misses, and less cascading failure risk when downstream systems degrade. In globally distributed systems, local hits prevent cross-region chatter, easing replication costs and helping teams defer expensive capacity upgrades while keeping SLAs tight.

**Cause of cache miss**

* **Cold start/compulsory**: key never cached yet, so the first few requests must fetch from origin.
* **Capacity**: working set exceeds cache size, so eviction pushes out data that is still popular.
* **Conflict**: keys contend for the same slot/shard (e.g., poor hash spread or multi-tenant hotspots) causing premature eviction.
* **Coherence/consistency**: data gets invalidated or bypassed intentionally (manual purge, TTL expiry) before the next request.
* **Contention stampedes**: simultaneous misses on a hot key evict intermediate data and keep the hit ratio low until stabilization.

**Strategies to maximize cache hit:**

* **Normalize keys and requests** so equivalent lookups map to the same entry (e.g., lowercase IDs, strip query noise) and avoid duplicate misses.
* **Tier caches (L1 in-process, L2 shared, CDN edge)** to serve most requests from the closest layer while letting colder keys fall through gracefully.
* **Tune TTLs by data volatility**—longer lifetimes for slow-changing resources and short TTLs (or background refresh) for hot-but-stable blobs keeps them resident without going stale.
* **Warm or prefetch critical keys** during deploys, traffic shifts, or after cache flushes so that expensive reads never happen under user load.
* **Segment by locality or tenant** (per-region, per-customer pools) to keep hot subsets from being evicted by unrelated traffic and to leverage data affinity.
* **Request coalescing/locking** prevents a thundering herd on cold keys; the first miss populates the cache while others wait, turning many misses into one.

#### Maintain data freshness (prevent data staleness)

Data mutates constantly, and any lag between the source of truth and the cached copy shows up as stale responses. Serving outdated data erodes user trust (wrong balances, expired entitlements), causes automation to act on bad signals, and can even create regulatory or financial exposure. Worse, downstream retries against incorrect data amplify traffic, masking the real root cause and forcing teams to debug symptoms created entirely by the cache layer.

**Why cache entries go stale:** Here are top commons causes:

* **TTL mismatch**: the item changes faster than its expiration policy, so the cache outlives the truth.
* **Missed invalidations**: application code forgets to delete or publish an update event after a write.
* **Lagging async pipelines**: write-back queues, CDC streams, or replication links fall behind and delay refreshes.
* **Regional skew**: only part of a multi-region deployment updates successfully, leaving other regions with old values.
* **Out-of-band writes**: hotfix SQL scripts or maintenance tooling update the origin without touching the cache.

**Strategies to keep caches fresh:**

* **Couple writes with cache actions**: write-through or transactional outbox patterns ensure every persisted change also updates or invalidates the relevant key.
* **Multi-channel invalidation**: combine TTLs with pub/sub, Kafka topics, or webhooks so critical keys refresh immediately rather than waiting for expiry.
* **Background refresh/read-repair**: cron jobs or async workers refresh near-expiry keys, and readers compare versions/ETags to repopulate when they detect drift.
* **Versioning and checksum guards**: include version numbers in cache keys so schema or business rule changes automatically bypass stale entries.
* **Localized caches with gossip/CRDT sync**: replicate freshness signals across regions to keep distant edge caches aligned without heavy centralized coordination.

#### Finding the balance

* **Set service-level targets first**: define acceptable staleness windows (e.g., 30s for product inventory, <1s for auth) alongside latency/error budgets; tune cache TTLs and refresh cadence to honor the stricter requirement.
* **Use adaptive policies**: mix short TTL + background refresh for mutable data, but allow relaxed TTLs with periodic validation for reference data so overall hit ratio stays high without sacrificing correctness on fast-changing keys.
* **Separate read/write paths**: keep hot read-mostly aggregates in cache, while writes flow through authoritative systems that emit invalidation events; this keeps freshness bounded without forcing every request to bypass cache.
* **Observability loops**: track hit ratio, origin load, staleness incidents, and replication lag in the same dashboard; when one metric drifts (e.g., stale responses spike), automatically tighten TTLs or boost refreshers temporarily.

**Example trade-offs:**

* **Product catalog**: base metadata (names, images) can live with 30–60 min TTL, but price and availability layer uses write-through + regional invalidation topics so shoppers rarely see stale counts yet benefit from cached descriptions.
* **Ad auction budgets**: per-campaign spend counters refresh every few seconds via Redis + stream ingest to keep bids accurate, while larger targeting configs update hourly; the split allows 99% cache hits on configs while preserving strict budget enforcement.
* **API rate limits**: counters are cached in-memory for fast reads but also replicated via pub/sub to edge nodes; when a central node deems a user blocked, it pushes an event so edges can invalidate immediately, combining high hit rates with consistency for abusive traffic.

#### Cache update/invalidate

* **Write-through**: every write goes to the backing store and cache simultaneously—hit rate stays high because reads usually find populated entries, and freshness is excellent since cache and source update atomically. The trade-off is slower writes and extra load on the database, so it fits workloads where correctness beats write throughput (e.g., account balances, entitlement toggles, fraud decisioning).
* **Write-back**: writes land in cache and flush asynchronously; this maximizes perceived write throughput and keeps hot entries resident (great hit rate), but freshness temporarily lags until flush completes. Works best when you can tolerate short-lived inconsistency and have durable queues/checkpointing to replay after crashes, such as analytics counters, social feed likes, or game leaderboards.
* **Write-around**: writes skip cache to avoid thrashing; the cache only fills on read miss, so freshness is perfect once data is cached, but hit rates dip right after writes because new values are absent until requested. Ideal for write-heavy datasets with low read reuse (bulk imports, telemetry ingestion, nightly ETL loads) where preserving cache space for true hot keys matters more than immediate hits.
* **Invalidation-first**: TTLs, explicit deletes, version tags, or pub/sub notifications keep cached data fresh by evicting or updating entries in response to changes; hit rate depends on how aggressive the invalidation is—short TTLs reduce hits but guarantee correctness, while event-driven invalidation allows longer TTLs without staleness. Choose a mix based on how fast data changes and how costly misses are; common in product catalogs, configuration stores, and feature-flag services.

### Troubleshooting

#### I see high cache miss

* **Metrics to collect/alert on:**
  * `overall_hit_ratio` and per-layer hit ratios (L1, CDN, Redis) to see where misses originate.
  * `miss_reason` counters (cold, expired, evicted, backend error) to immediately classify the population.
  * `evictions/sec`, `memory_utilization`, and `item_age_percentiles` to confirm if capacity pressure or TTLs are the culprit.
  * `key_space_cardinality` and top-N key distributions to catch skew or newly hot tenants flushing others out.
  * `origin_latency`/`origin_qps` to correlate whether the backend is now handling the load (confirming misses are real) or if a reporting bug misleads you.
* **Reasoning workflow:**
  1. **Validate telemetry**: if hit ratios drop but origin load is flat, suspect instrumentation drift; otherwise move on.
  2. **Break down by reason**: spikes in cold misses imply deploy/warmup issues, evictions point to capacity/sizing, expirations indicate TTL too short.
  3. **Inspect key patterns**: high churn/new tenants? If the working set outgrew cache, consider sharding, tiering, or bigger nodes; if a handful of keys dominate, enable request coalescing or hot-key pinning.
  4. **Check write behavior**: write-through disabled? write-back queue backed up? If invalidations surged (e.g., new feature pushes deletes), coordinate with the team to tune TTL or batch invalidations.
  5. **Simulate misses**: replay a few representative requests to ensure keys are normalized and not splintering due to case, locale, or header differences.
  6. **Act and verify**: once a fix (warm cache, add tier, extend TTL) is applied, watch hit ratio plus tail latency to confirm improvement and avoid regressions on freshness.

#### Downstream or customer report data staleness spike

* **Metrics to collect/alert on:**
  * `stale_response_rate` (requests serving data older than SLA) broken down by endpoint/region.
  * `cache_entry_age` histograms vs `origin_last_update` to see if TTLs or invalidations lag.
  * `write_throughput`, `write_error_rate`, and `write_back_queue_depth` to catch ingestion stalls.
  * `invalidation_lag` (difference between origin change and cache purge message arrival) plus pub/sub backlog metrics.
  * `cross-region version skew` or `checksum mismatch` gauges when a subset of caches fail to update.
* **Reasoning workflow:**
  1. **Verify reproduction**: fetch the offending key from both cache and origin, compare versions/timestamps to confirm staleness and scope (single region vs global).
  2. **Check invalidation path**: inspect logs/metrics for dropped pub/sub events, paused consumers, or muted webhook endpoints; if lag spikes, drain backlog before taking other action.
  3. **Assess TTL and refresh settings**: did a config change extend TTLs or disable background refresh? If yes, revert or tighten TTLs for hot data.
  4. **Inspect write pipelines**: for write-back, ensure flush workers run and queues aren’t stuck; for write-through, look for DB write failures that skipped cache updates.
  5. **Regional health**: confirm all cache clusters and replication links are healthy; a single unhealthy region can serve stale data even if others are fresh.
  6. **Mitigate and monitor**: temporarily force-refresh critical keys (warmers, batch invalidation) while fixing root cause; set short-term alerts on stale rate and invalidation lag to ensure the fix sticks.
