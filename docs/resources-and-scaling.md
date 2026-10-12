# Resources and scaling

## Requests and limits
- Requests influence scheduling and resource-based autoscaling calculations.
- CPU limits can throttle; memory limits can cause OOM termination.
- Choose values from representative load tests and observed behavior, not guesses.
- Profile high memory use before simply increasing limits.

## HPA
The example HPA targets CPU utilization relative to CPU requests. It needs working metrics (often metrics-server). CPU is not always the best demand signal; queue depth or custom metrics may be more appropriate.

```bash
kubectl get hpa -A
kubectl describe hpa payment-api -n production
kubectl top pods -n production
kubectl top nodes
```

## Three capacity layers
1. Pod autoscaling: HPA changes replica count.
2. Node autoscaling: Cluster Autoscaler or Karpenter provides node capacity.
3. Dependency capacity: DB connections, downstream quotas, queues and worker pools must handle more concurrency.

If 10 Pods each open 50 database connections, the database could see 500 connections. Model this before raising maxReplicas.

## Load-test questions
- Target RPS and peak/burst profile?
- P95/P99 latency and error-rate objectives?
- Which resource or dependency saturates first?
- Does latency rise while CPU stays low due to pool waits, locks, network or downstream latency?
- Is scale-up fast enough?
