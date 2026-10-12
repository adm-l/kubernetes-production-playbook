# Security baseline

This baseline does not replace your organization's threat model or policy.

- Run as non-root; drop Linux capabilities.
- Disable privilege escalation; use seccomp RuntimeDefault where supported.
- Use read-only root filesystem when supported; mount explicit writable paths when needed.
- Use dedicated ServiceAccounts and least-privilege RBAC. Disable token automount if the workload does not call the Kubernetes API.
- Never store credentials in Git. Use a secret manager integration and narrow access.
- Pin production images to digests, scan images, generate SBOMs and verify provenance according to policy.
- Test NetworkPolicies with the actual CNI and explicitly allow required DNS, telemetry and dependencies.
- Use quotas/limits and admission policies in multi-team clusters.
- Restrict API access and audit privileged operations.
- Patch Kubernetes, nodes, add-ons and dependencies through controlled processes.

Security settings can break apps that expect root or write to the image filesystem. Test and document justified exceptions.
