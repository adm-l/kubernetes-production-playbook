# Deployment strategies

## RollingUpdate
A common choice for stateless APIs. Deployment creates a new ReplicaSet and scales new/old ReplicaSets according to maxSurge and maxUnavailable.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Ensure capacity for surge Pods. A strict zero-unavailable setting can stall if new Pods never become Ready. Alert on stalled rollouts.

## Recreate
Stops old Pods before starting new Pods. Use only when versions cannot safely coexist and downtime is accepted.

## Blue/green and canary
Usually require a traffic-routing layer or progressive-delivery controller. Define metrics-based promotion and rollback.

## Commands
```bash
kubectl rollout status deployment/payment-api -n production
kubectl rollout history deployment/payment-api -n production
kubectl rollout undo deployment/payment-api -n production
kubectl get rs,pods -n production -l app.kubernetes.io/name=payment-api
```

Rollback application image/configuration, but separately consider database schema compatibility and irreversible side effects. Prefer expand-and-contract migrations.
