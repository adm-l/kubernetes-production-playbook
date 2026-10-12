# Kubernetes learning path

1. Core objects: Namespace, Pod, labels/selectors, Deployment, ReplicaSet, Service.
2. Deployments: image tags/digests, rolling updates, rollout status/history/undo.
3. Configuration: ConfigMaps, external secrets, environment variables.
4. Reliability: probes, graceful shutdown, resource requests/limits, PDB.
5. Networking: Service DNS, Ingress/Gateway API, TLS, NetworkPolicy.
6. Scaling: HPA, metrics API, load tests, node autoscaling, dependency bottlenecks.
7. Security: ServiceAccounts, RBAC, non-root, seccomp, image scanning and admission policy.
8. Operations: logs, events, metrics, traces, alerts and incident runbooks.
9. Delivery: Kustomize/Helm, CI validation, GitOps/Argo CD.
10. EKS: IAM roles for workloads, VPC CNI, load balancer controller, cluster upgrades and node lifecycle.

Practice in a disposable kind or minikube cluster before shared environments. Never test failure scenarios in production without an approved exercise plan.
