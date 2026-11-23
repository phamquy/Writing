# Adaptive retry budget

Two clients call the same kind of downstream. One retries up to three times no matter what. The other spends from a retry budget that refills more slowly as the failure rate rises. Start an outage and watch how each one treats a struggling service.

## Fixed retries

Up to 3 attempts per request, always

–requests succeed –attempts per request –attempts/s sent

## Adaptive budget

Token bucket, refill ratio r driven by failure rate

–requests succeed –attempts per request –attempts/s sent

## Load on the downstream

FixedAdaptive

Attempts per second sent to the downstream by each client over the last 60 seconds, against capacity

## Requests that succeed

FixedAdaptive

Percentage of caller requests that succeeded over the last 60 seconds

## How r follows the failure rate

Refill ratio r as a function of observed failure rate, with the current operating point marked

## Adaptive client, right now

–observed failure rate f –refill ratio r –tokens in bucket Retry bucket

**Things to try.** Start an outage with the defaults: the fixed client triples its traffic, pushes the downstream past capacity, and may stay broken after the outage ends, while the adaptive client drops to a trickle of retries and recovers. Then lower requests per second to give the fixed client room to recover, or raise r\_max to see how a generous budget behaves.

**How the model works.** Each client has its own simulated downstream. Failures come from the baseline rate plus overload: when attempts per second exceed capacity, the failure probability climbs steeply. Both clients are measured with the same traffic, and both cap out at 3 attempts per request. Backoff and jitter are left out so the effect of the budget stays visible.
