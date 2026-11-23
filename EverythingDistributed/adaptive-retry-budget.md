---
title: Adaptive retry budget — the mechanism
status: draft
created: 2026-11-23
last_updated: 2026-11-23
audience: Jack
related:
  - ../mock-interview-script.md
  - ../screen-prep.md
tags: [distributed-systems, retries, overload-protection, interview-prep]
---
# Adaptive retry budget — the mechanism

An adaptive retry budget is a limit on how much extra load a client may create by retrying, where the limit itself shrinks as the downstream service gets less healthy. It exists because retries are helpful when failures are rare and harmful when failures are widespread, and a client needs a rule that tells the two situations apart without being told which one it is in. This note describes the algorithm, explains what makes it adaptive, and shows how it stops a retry storm. It describes the general technique, the same family as **Finagle's RetryBudget**, gRPC retry throttling, and the AWS SDK's adaptive retry mode, and is not a record of any one production system.

## The problem it solves

**A retry storm is a feedback loop**. When a service becomes overloaded it starts failing requests, and every client that sees a failure sends the request again. Those retries arrive at exactly the moment the service has the least spare capacity, so the offered load goes up as the service's ability to handle it goes down, which produces more failures and therefore more retries. With a common policy of three attempts per request, a service that is failing everything receives three times its normal traffic, and the multiplication compounds across layers: three attempts at each of three layers is twenty-seven requests at the bottom for one at the top. The result is a system that stays down after the original trigger has gone, held there by its own clients.

The reason plain retries fail here is that they assume failures are independent. A retry works when the first attempt hit a transient problem, such as one bad host or one dropped packet, because the second attempt will probably land somewhere healthy. When the whole service is overloaded the failures are correlated, so the second attempt meets the same condition as the first and succeeds no more often, while adding load that makes the condition worse.

## The algorithm

The mechanism has two parts: *a token bucket that caps retries as a fraction of real traffic*, and *a rule that lowers that fraction as the observed failure rate rises.*

The token bucket is kept by each client instance, separately for each downstream service it calls. Every first attempt deposits a small amount, `r` tokens, into the bucket. Every retry must withdraw *one whole token*, and if the bucket holds less than one the request fails fast instead of retrying. The bucket has a maximum size so that a long quiet period cannot save up a large allowance to be spent in one burst. Because deposits come only from first attempts and each retry costs one token, retries can never exceed `r` times the first-attempt traffic over any sustained period. With `r` at `0.2`, retries add at most `20` percent.

The second part makes `r `a *function of the failure rate*, `f`. The client measures `f` over a short sliding window or as a moving average, counting *only failures that are worth retrying*, which means timeouts, server errors, and throttles. When `f` is low, `r` sits at its maximum. As `f` rises between a lower and an upper threshold, `r` falls in a straight line. Once `f` passes the upper threshold, `r` stays at a small minimum. With the suggested starting values, retries may add up to `20` percent extra load while fewer than `2` percent of calls fail, and only `1` percent once `30` percent of calls fail. The exact formula, the starting values, and the pseudocode are in Appendix A, which also notes a small correction to the formula as usually written.

## Why it is adaptive

It adapts in two independent ways, and both matter. The first is that the budget is a ratio and not a number. Nobody configures "at most 500 retries per second"; the allowance is earned by first attempts, so it scales by itself when traffic doubles at peak or halves overnight, and stays correct when the fleet of clients grows or shrinks. A fixed absolute limit is a guess about traffic that is wrong as soon as traffic changes, which is the same weakness a static throttle has.

The second way is that the ratio follows the health of the downstream. The failure rate is the client's evidence about which kind of failure it is seeing. A low rate suggests isolated, transient faults, where a retry is cheap and likely to succeed, so the budget is generous. A high rate suggests a correlated failure, where a retry is unlikely to succeed and certain to add load, so the budget closes down. The client therefore changes its own behavior from the responses it observes, with no operator, no central coordinator, and no signal from the downstream other than the failures themselves.

## How it protects against a retry storm

The protection comes from putting a hard ceiling on the gain of the feedback loop. Without a budget, the load a failing service receives can reach the attempt limit times its normal traffic. With the budget, it can reach at most one plus r times normal traffic, and the adaptive rule makes r smallest exactly when the service is weakest. A service at a 40 percent failure rate that would have received about one and a half times its traffic under three-attempt retries, and three times if it were failing everything, instead receives about 101 percent, so a rejection actually removes load instead of generating more. That is what lets an overloaded service shed its way back to health: the loop that held it down has been cut. The numbers behind this comparison are in Appendix B.

Two further properties make the protection practical. The first is that the budget never falls to zero. The small minimum lets a trickle of retries through even in a severe failure, which acts as a probe, so recovery is noticed and the budget reopens without anyone intervening. The second is that it needs no coordination. Each client instance enforces its own bucket, and because every instance obeys the same ratio, the total across the fleet obeys it too, which means the protection adds no new dependency that could itself fail during an incident.

The cost is deliberate and worth stating. During a brief blip, some retries that would have succeeded are refused because the bucket is empty, so the success rate in mild events is slightly worse than unlimited retries would give. The design accepts that in exchange for ruling out collapse in severe events.

## Try it

The simulator below runs the naive and adaptive policies side by side so the difference in load on the downstream can be seen directly. If it does not appear, the viewer does not allow embedded pages

<iframe src="res/Adaptive%20retry%20budget%20simulator.html" title="Adaptive retry budget simulator" width="100%" height="760" style="border:1px solid #CBD3D9;border-radius:8px;"></iframe>

## What it needs around it

The budget handles the gradual region between healthy and down, and it depends on a few companions to work well. Retries should happen at one layer only, or the per-layer budgets still multiply. The rate r should not move until the window holds enough samples, and it should fall quickly and recover slowly, or the control loop will oscillate. Explicit signals from the downstream, such as a throttle response or a retry-after hint, should push the budget to its minimum at once. A request should never be retried if its remaining deadline is too short for the retry to finish, and only idempotent calls should be retried at all. A circuit breaker belongs on top for the case where the downstream is fully down, since the budget is designed to slow retries, not to stop first attempts. These guardrails, a simpler variant that needs no thresholds, and the metrics to emit are in Appendices C to E.

---

# Appendix A — Formula, parameters, and pseudocode

**Refill ratio as a function of failure rate.**

```text
r = r_max                                                  if f <= f_lo
r = r_min + (r_max - r_min) * (f_hi - f) / (f_hi - f_lo)   if f_lo < f < f_hi
r = r_min                                                  if f >= f_hi
```

The middle line is often written as `r_max * (f_hi - f) / (f_hi - f_lo)`. That version falls to zero just below `f_hi` and then jumps up to `r_min` at `f_hi`, so it should either be written as above or clamped with `max(r_min, ...)`. With the suggested values the difference is small, but the form above is continuous and never goes below the minimum.

**Suggested starting values.**

| Parameter        | Meaning                                               | Starting value |
| ---------------- | ----------------------------------------------------- | -------------- |
| `r_max`        | Retry allowance per first attempt when healthy        | 0.2            |
| `r_min`        | Allowance under severe failure; the recovery probe    | 0.01           |
| `f_lo`         | Failure rate below which the budget is fully open     | 2%             |
| `f_hi`         | Failure rate above which the budget is at its minimum | 30%            |
| `max_tokens`   | Bucket cap, limiting burst after idle periods         | 10 to 100      |
| Window for `f` | Sliding window or EWMA; retryable failures only       | 10 to 30 s     |
| Minimum samples  | Requests needed before `r` is adjusted              | about 50       |
| `max_attempts` | Total attempts per request                            | 3              |

**Pseudocode.** State is per downstream, per client instance.

```python
def on_request_start():
    r = refill_ratio(failure_rate())              # recomputed from f
    bucket = min(max_tokens, bucket + r)

def may_retry(attempt, err, deadline):
    if attempt >= max_attempts:                 return False   # e.g. 3 total
    if not retryable(err):                      return False   # no 4xx, no non-idempotent calls
    if now() + backoff(attempt) > deadline:     return False
    if bucket < 1:                              return False   # budget spent: fail fast
    bucket -= 1
    return True

def backoff(attempt):
    return random(0, min(cap, base * 2**attempt))              # full jitter
```

# Appendix B — Worked numbers

The table shows the load a downstream receives, as a multiple of first-attempt traffic, at different failure rates. "No budget" assumes three attempts and that each retry fails at the same rate f, giving 1 + f + f². "Fixed budget" caps retries at 20 percent. "Adaptive" uses the formula and starting values in Appendix A. It backs the claim in the body that the adaptive rule is tightest when the service is weakest.

| Failure rate f | No budget, 3 attempts | Fixed budget, r = 0.2 | Adaptive r | Adaptive load |
| -------------- | --------------------- | --------------------- | ---------- | ------------- |
| 1%             | 1.01x                 | 1.01x                 | 0.20       | 1.01x         |
| 10%            | 1.11x                 | 1.11x                 | 0.146      | 1.11x         |
| 16%            | 1.19x                 | 1.19x                 | 0.105      | 1.11x         |
| 30%            | 1.39x                 | 1.20x                 | 0.01       | 1.01x         |
| 40%            | 1.56x                 | 1.20x                 | 0.01       | 1.01x         |
| 100%           | 3.00x                 | 1.20x                 | 0.01       | 1.01x         |

At low failure rates the budget is not binding and all three behave alike, which is the intended result: retries are left alone when they help. The columns diverge as failures become widespread. Across three layers that each retry, the "no budget" worst case is 3 × 3 × 3 = 27x at the bottom layer.

# Appendix C — Guardrails

| Guardrail                     | Rule                                                                                                        | Why                                                                |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Minimum samples and smoothing | Do not adjust `r` until the window has enough requests; cut fast, restore slowly                          | Prevents oscillation on thin or noisy data                         |
| Retry at one layer            | One layer retries; others pass errors through, or a retry-attempt header tells lower layers not to retry    | Stops multiplication across layers                                 |
| Honor server signals          | On a throttle response, a retry-after hint, or a load-shed signal, raise `f` or set `r = r_min` at once | The downstream knows its state better than the client's estimate   |
| Deadline propagation          | Never retry if the remaining deadline is shorter than the downstream's median latency                       | A retry that cannot finish is pure load                            |
| Circuit breaker on top        | If `f` stays above roughly 50% for a sustained window, open the breaker                                   | The budget covers the gradual region; the breaker covers hard-down |
| Idempotency                   | Retry only safe or idempotent calls, or use idempotency keys                                                | A retry must not apply an effect twice                             |

# Appendix D — Variant: AIMD on the budget

If choosing `f_lo` and `f_hi` is unwelcome, the ratio can be driven the way TCP drives its congestion window. On each success, `r` increases by a small constant up to `r_max`. On each retryable failure or throttle, `r` is multiplied by a factor of about 0.5, down to `r_min`. This is essentially what the AWS SDK's adaptive retry mode does. It is simpler and tunes itself, at the cost of a more jagged response than the threshold version.

# Appendix E — Metrics and alerts

Emit the current `r`, the bucket level, retries attempted against retries denied by the budget, and the ratio of retry traffic to first-attempt traffic. Alert if retry traffic stays above roughly `r_max` of the total, because that means the budget is not binding, which points to retries that bypass it or to a misconfiguration.

A per-instance budget is sufficient when clients are many small instances and avoids any coordination. The aggregate bound still holds: total retries are at most `r` times total first-attempt traffic, plus at most `max_tokens` of burst per instance.
