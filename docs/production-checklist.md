# Production readiness checklist

## Workload
- [ ] Image is reviewed, scanned, pinned to a digest for releases, and stored in an approved registry.
- [ ] Deployment selector matches Pod-template labels.
- [ ] Replica count and rollout strategy fit availability targets and cluster capacity.
- [ ] CPU/memory requests and limits are based on load tests and observed metrics.
- [ ] Startup, readiness and liveness probes match application semantics.
- [ ] App handles SIGTERM, drains in-flight work and exits within termination grace period.
- [ ] Logs, metrics and traces are available.
- [ ] Non-root user, no privilege escalation, dropped capabilities, seccomp RuntimeDefault.
- [ ] Writable paths are mounted explicitly when root filesystem is read-only.
- [ ] No credentials are embedded in images, Git or logs.

## Networking and configuration
- [ ] Service selector and targetPort are verified.
- [ ] TLS, hostnames, routing, request sizes and timeouts are reviewed.
- [ ] NetworkPolicies are tested with the actual CNI.
- [ ] DNS, telemetry, secret provider and dependency egress are allowed where needed.
- [ ] Secrets use the approved secret manager and access policy.

## Availability and scaling
- [ ] HPA metrics API works; min/max replicas are reviewed.
- [ ] Node autoscaling and topology spread fit the workload.
- [ ] PDB is compatible with replica count and maintenance plan.
- [ ] Database pools, downstream quotas and queues are sized for maximum replicas.
- [ ] Backups and restore tests exist for stateful dependencies.

## Delivery and operations
- [ ] CI validates YAML, schemas, policy and image provenance.
- [ ] Alerts map to user impact and SLOs.
- [ ] Rollback procedure is documented and tested.
- [ ] Runbooks identify owner, diagnosis, mitigation and verification.
- [ ] Changes use approved CI/GitOps for shared environments.
