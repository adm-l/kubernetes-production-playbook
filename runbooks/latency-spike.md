# Runbook: API latency spike

## Triage
- Establish time window, affected routes/regions/tenants and impact against SLO.
- Compare RPS, errors, P50/P95/P99, CPU, memory, throttling, restarts and replicas.
- Inspect traces for slow spans and logs with request/correlation IDs.
- Check DB pool waits, slow queries, locks, cache hit ratio, queue depth, downstream latency and rate limits.
- Compare with recent deployments/config changes and dependency incidents.

## Mitigate safely
Scale only if compute is the bottleneck and downstream capacity can absorb load. Consider rollback for a correlated bad release. Apply load shedding, concurrency limits or circuit breakers according to existing design. Avoid retry storms.

## Verify
Confirm latency and errors return within objectives, dependencies recover, and mitigation did not shift the bottleneck. Record timeline and follow-up actions.
