# Kubernetes Production Playbook

A practical reference for deploying, scaling, securing, observing, and troubleshooting Kubernetes workloads. Examples focus on stateless Go HTTP microservices and AWS EKS while keeping generic Kubernetes manifests portable. This repository intentionally does not use GitHub Actions workflows; validation is done locally to avoid workflow charges.

## Start here
1. Read [the learning path](docs/learning-path.md).
2. Review the [production readiness checklist](docs/production-checklist.md).
3. Explore the [reference architecture](docs/architecture.md).
4. Inspect the [sample Go API manifests](examples/go-api/base/).
5. Use the [kubectl quick reference](docs/kubectl-cheatsheet.md) and [troubleshooting runbooks](runbooks/).

## Repository map
- `examples/go-api/base/`: Deployment, ServiceAccount, Service, HPA, PDB and NetworkPolicy.
- `examples/go-api/overlays/`: Kustomize development and production examples.
- `docs/`: Architecture, rollout strategy, security, resources/scaling, Kustomize and operations.
- `runbooks/`: CrashLoopBackOff, Pending Pods and latency spikes.

## Validate locally
Prerequisites: kubectl with Kustomize support; optionally use kind/minikube and a schema validator such as kubeconform.

```bash
# Render the development overlay; does not change a cluster.
kubectl kustomize examples/go-api/overlays/dev

# Client-side dry run; may require cluster/API discovery depending on kubectl version.
kubectl apply --dry-run=client -k examples/go-api/overlays/dev

# With a cluster available, inspect changes before applying.
kubectl diff -k examples/go-api/overlays/dev
kubectl apply -k examples/go-api/overlays/dev
```

## Safety notes
- These are reference templates, not universal production defaults. Review all values for your cluster, traffic, SLOs and compliance needs.
- `ghcr.io/adm-l/sample-go-api:1.0.0` is a placeholder image. Publish and verify your own image before deploying.
- The app must implement `/healthz` and `/readyz`, run as UID 10001, and support a read-only root filesystem for the provided Deployment to work unchanged.
- No real credentials belong in Git. Integrate External Secrets Operator, a cloud secret manager, or your organization's approved solution.
- The sample NetworkPolicy assumes a CNI that enforces NetworkPolicy. It restricts ingress and does not define egress; review DNS, telemetry and dependency flows before applying stricter egress policies.
- HPA requires working resource metrics (commonly metrics-server). Pod autoscaling and node autoscaling are separate.
- `containerPort` documents the listening port; it does not expose the app externally. A Service gives stable in-cluster access.
- Resource sizing, probes, replicas, PDBs and security controls must be tested and reviewed before production use.
- Apply shared-environment changes through approved change-management processes and your organization's deployment controls.

## Core model
Deployment manages rollout and ReplicaSets. ReplicaSets attempt to maintain the desired Pod count. Pods run containers. Services provide stable networking to selected Pods. HPA adjusts replica count based on configured metrics.

## License
Choose and add a license appropriate for your intended use.
