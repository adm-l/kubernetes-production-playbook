# Reference architecture

## Request path
Client -> DNS/TLS edge or cloud load balancer -> Ingress/Gateway -> Kubernetes Service -> ready Pod -> Go handler -> dependencies.

On AWS EKS, a common option is Route 53 plus AWS Load Balancer Controller and an ALB. Follow your organization's network and security standards.

## Core components
- API server validates and serves Kubernetes API requests.
- etcd stores cluster state.
- Scheduler selects nodes for unscheduled Pods.
- Controller manager runs reconciliation controllers.
- Worker nodes run kubelet, container runtime and networking.
- Deployment manages ReplicaSets and rollout.
- ReplicaSet attempts to maintain desired Pod count.
- Service provides stable virtual networking and service discovery.
- HPA changes workload replicas based on configured metrics.

## Desired-state reconciliation
A manifest describes desired state. Controllers compare observed state with desired state and attempt to reconcile differences. Three replicas does not guarantee three healthy Pods at all times.

## Stateless API baseline
- Multiple replicas sized to SLO and capacity.
- Readiness gates traffic; liveness detects unrecoverable process health; startup protects slow starts.
- Requests support scheduling; limits constrain consumption.
- HPA scales Pods; node autoscaling must separately provide capacity.
- PDB helps with supported voluntary disruptions but does not prevent all failures.
- Use topology spread when failure-domain diversity is required.
- Keep durable state outside disposable Pods unless using appropriate stateful patterns.

More replicas can increase database connections and downstream load. HPA does not fix database saturation. A containerPort does not expose an app externally.
