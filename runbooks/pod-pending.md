# Runbook: Pod stuck Pending

1. Run `kubectl describe pod <pod> -n <namespace>` and inspect Events.
2. Check CPU/memory requests against available node capacity.
3. Review node selectors, affinity, topology spread, taints and tolerations.
4. If storage is used, inspect PVC/PV and StorageClass events.
5. Check namespace ResourceQuota and LimitRange.
6. Check node readiness and autoscaler events if nodes should be added.
7. Fix the scheduling constraint or capacity issue; do not remove security controls blindly.

Verify the Pod is scheduled, becomes Ready and serves healthy traffic.
