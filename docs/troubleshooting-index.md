# Troubleshooting index

Start with evidence, not guesses. Check Events and recent changes before modifying resources.

| Symptom | First checks | Runbook |
|---|---|---|
| CrashLoopBackOff | describe Pod, previous logs, exit code, probes, config, OOMKilled | [CrashLoopBackOff](../runbooks/crashloopbackoff.md) |
| Pod Pending | scheduling events, requests, taints, affinity, PVC, quota | [Pod Pending](../runbooks/pod-pending.md) |
| High API latency | P95/P99, traces, DB pool, dependencies, CPU throttling | [Latency spike](../runbooks/latency-spike.md) |
| Service has no traffic | Service selector, EndpointSlices, readiness, NetworkPolicy | [kubectl commands](kubectl-cheatsheet.md) |
| HPA not scaling | HPA events, metrics API, resource requests, configured metrics | [Resources and scaling](resources-and-scaling.md) |
| ImagePullBackOff | image name/tag/digest, registry reachability, image pull credentials | [kubectl commands](kubectl-cheatsheet.md) |
| OOMKilled | container memory usage/limit, heap/profile, burst behavior | [Resources and scaling](resources-and-scaling.md) |

Do not delete Pods or restart workloads as the first action unless immediate mitigation is approved and evidence supports it. Preserve events and logs for diagnosis.
