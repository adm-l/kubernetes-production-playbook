# Kubernetes Production Network Troubleshooting Runbook

A practical incident-response guide for Kubernetes networking, with examples for Go HTTP microservices and AWS EKS. Use this when requests fail, time out, become slow, or reach only some Pods.

> **Safety:** Start with read-only diagnostics. Run commands in the correct cluster and namespace. Use approved debug containers and packet-capture procedures. Do not delete Pods, restart CoreDNS, change NetworkPolicies, flush conntrack, or modify routes/security rules until you have evidence and an approved mitigation plan. Examples use placeholders such as `<pod>`; replace them before running.

## 1. Understand the request path

Typical external request path:

```text
Client
  -> DNS for public hostname
  -> Cloud load balancer / listener / target group
  -> Ingress controller or Gateway
  -> Kubernetes Service
  -> EndpointSlice (eligible Pod IPs and ports)
  -> Pod network namespace -> application listener
  -> downstream Service / database / external API
```

Kubernetes networking also depends on the cluster DNS service, CNI, node routes, firewall/security groups, network ACLs, and (where supported) NetworkPolicy enforcement.

**Key principle:** A Kubernetes Service object existing does not prove it has healthy endpoints, and a Pod being `Running` does not prove it is Ready or that the application is listening.

## 2. Incident triage: first 5–10 minutes

### Step 1 — Establish impact and timeline

Record:
- Start time and whether a deployment, node scale event, policy change, certificate rotation, or infrastructure change preceded it.
- Affected API routes, regions/availability zones, namespaces, services, Pods, and nodes.
- Error rate, request volume, P50/P95/P99 latency, timeout rate, and whether failures are intermittent.
- Whether the issue affects internal callers, external callers, or both.

Avoid starting with a cluster-wide restart. Determine the blast radius first.

### Step 2 — Snapshot cluster state

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get ingress -A
kubectl get events -A --sort-by=.lastTimestamp
```

For a focused namespace:

```bash
kubectl get pods,svc,endpointslice,networkpolicy -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --since=15m
kubectl logs <pod> -n <namespace> --previous
```

The `--previous` option is useful when a container restarted and the prior container's logs are available.

### Step 3 — Locate the failing hop

Test in this order:
1. Is the application process healthy and listening on the expected port?
2. Can a caller resolve the destination Service name?
3. Does the Service have eligible EndpointSlices?
4. Can a caller connect to a Pod IP and port (where your environment permits this test)?
5. Does the Service path work from inside the cluster?
6. Does the Ingress/Gateway path work?
7. Does the external load balancer path work?
8. For outbound traffic, can the Pod resolve and connect to the external destination?

Use the same source Pod, destination, port, protocol, and timestamp when comparing results.

## 3. Fast symptom-to-cause guide

| Symptom | Likely areas to inspect first |
|---|---|
| `no such host` | DNS name, namespace, CoreDNS, DNS egress/policy |
| `connection refused` | Wrong port, app not listening, wrong targetPort, destination actively rejecting |
| Connection timeout | NetworkPolicy, routes, firewall/security group, CNI, unhealthy or overloaded destination |
| HTTP 502 | Proxy cannot connect to upstream, upstream closed connection, invalid upstream response |
| HTTP 503 | No eligible/healthy backends, overloaded service, proxy has no upstream |
| HTTP 504 | Upstream response deadline exceeded; slow service or network path |
| Pod is Running but gets no traffic | Readiness, selectors, EndpointSlices, load-balancer health |
| Only one node/AZ is affected | Node/CNI/route/subnet/security configuration, conntrack or resource pressure |
| Internal traffic works, external traffic fails | Ingress, load balancer, public DNS, TLS, routing/firewall |
| Internal service works, external API fails | DNS, egress route/NAT, firewall, destination outage/rate limit |
| Small requests work; large requests hang | MTU/path-MTU, fragmentation, proxy/body-size limits |
| P95/P99 rises with intermittent failures | Packet loss/congestion, saturation, retries, connection churn, slow downstream |

These are hypotheses, not proofs. For example, an HTTP 504 can be caused by a slow database even when the network is healthy.

## 4. Runbooks by failure type

### 4.1 DNS resolution failure

**Symptoms**
- Errors such as `no such host`, `server misbehaving`, or long lookup delays.
- Multiple services fail to find cluster-local names.
- External names fail from Pods while resolving from a workstation.

**Checks**

```bash
kubectl get pods -n kube-system -o wide
kubectl get svc -n kube-system
kubectl get configmap coredns -n kube-system -o yaml
kubectl logs -n kube-system deploy/coredns --since=15m
kubectl exec -it <source-pod> -n <namespace> -- cat /etc/resolv.conf
kubectl exec -it <source-pod> -n <namespace> -- nslookup <service>.<namespace>.svc.cluster.local
```

If `nslookup` is unavailable in the application image, use an approved diagnostic Pod that has DNS utilities. Do not install tools into a production image during an incident unless your process permits it.

**Likely causes and actions**
- Typo or wrong namespace: use `service.namespace.svc.cluster.local` to remove ambiguity.
- CoreDNS unhealthy or saturated: inspect Pod readiness, restarts, logs, CPU/memory and DNS latency before scaling or restarting it.
- NetworkPolicy blocks DNS: verify the source Pod can reach the cluster DNS Service over UDP and TCP port 53 as required by your cluster.
- External DNS failure only: inspect upstream resolver configuration and egress path.
- Intermittent lookups: check CoreDNS metrics/logs, node-level packet loss, resolver timeouts and DNS query volume.

Do not hardcode Pod IPs as a fix; Pod IPs are ephemeral.

### 4.2 Service has no endpoints or routes to the wrong Pods

**Symptoms**
- Service exists, but traffic fails or returns 503.
- New deployment has no ready backends.
- Traffic is sent to an unexpected application version.

**Checks**

```bash
kubectl get svc <service> -n <namespace> -o yaml
kubectl get pods -n <namespace> --show-labels
kubectl get endpointslice -n <namespace> -l kubernetes.io/service-name=<service> -o yaml
kubectl describe svc <service> -n <namespace>
kubectl get pods -n <namespace> -o wide
```

**Likely causes and actions**
- Selector labels do not match Pod labels: correct the Service selector or workload labels.
- Pods are not Ready: inspect readiness probe events and application logs.
- Wrong port: verify Service `port`, `targetPort`, and the real listening port.
- Wrong namespace: Service discovery is namespace-scoped unless a qualified name is used.
- Unexpected endpoints: inspect selectors and rollout labels carefully.

EndpointSlices are the preferred API for examining Service backends. A legacy Endpoints object may not tell the full story in newer clusters.

### 4.3 Connection refused

A refusal usually means a connection attempt reached a destination that rejected it, or an intermediate component actively rejected it. It does not automatically prove the network is healthy.

**Checks**
- Confirm the process is alive and listening on the expected interface and port.
- Compare the Service's `targetPort` with the application's actual port.
- Inspect container logs and startup order.
- Where permitted, test the Pod IP:port and Service DNS:port separately.

For a Go HTTP server, listening on `127.0.0.1:8080` inside the container makes it unreachable through the Pod IP. A typical containerized service should listen on an appropriate interface such as `:8080`, with exposure controlled by Kubernetes networking.

**Fix:** Correct the listener, port mapping, or destination. Do not confuse `containerPort` metadata with opening a socket; the application itself must listen.

### 4.4 Ingress, Gateway, or load balancer failure

**Symptoms**
- Internal Service access works, but public traffic fails.
- HTTP 502/503/504, TLS errors, or cloud target health checks fail.

**Kubernetes checks**

```bash
kubectl get ingress -A
kubectl describe ingress <ingress> -n <namespace>
kubectl get svc,pods -n <ingress-namespace> -o wide
kubectl logs <ingress-controller-pod> -n <ingress-namespace> --since=15m
```

For Gateway API deployments, inspect the relevant Gateway, HTTPRoute, listener, and status conditions instead of assuming an Ingress resource is used.

**Check the whole path**
- Public DNS resolves to the intended load balancer.
- Listener and host/path rules match the request.
- TLS certificate, SNI, and backend protocol agree.
- Cloud target group health checks use the correct protocol, port, path, and expected response.
- Load balancer security groups/firewall rules allow the intended path.
- Ingress controller can connect to Service endpoints.
- Timeouts and body-size limits are consistent across proxy hops.

**Interpretation**
- 502 often indicates an upstream connection or response problem.
- 503 often indicates no eligible upstreams or overload.
- 504 means the proxy did not receive an upstream response before its deadline.

Confirm the specific controller/load balancer's behavior before concluding from the status code alone.

### 4.5 NetworkPolicy blocks traffic

**Checks**

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy> -n <namespace>
kubectl get pods -n <namespace> --show-labels
```

Review both source egress and destination ingress. If either side is isolated by policy, the flow must be permitted by the applicable policy rules. Confirm that the CNI supports and enforces NetworkPolicy.

For restrictive egress policies, explicitly review:
- DNS to the cluster resolver (UDP/TCP 53, as required).
- Required service-to-service ports.
- Database and message-broker ports.
- Telemetry exporters and approved external APIs.
- Health checks and control-plane integrations relevant to your platform.

**Fix:** Add the narrowest policy rule that permits the required flow. Avoid a blanket allow-all policy as a permanent production fix. Validate both the allowed path and that unrelated traffic remains blocked.

### 4.6 Pod IP exhaustion, CNI, or node networking failure

**Symptoms**
- Pods remain in `ContainerCreating`.
- Events mention sandbox creation, CNI setup, or IP allocation.
- Failures cluster around one node or availability zone.

**Checks**

```bash
kubectl get pods -A -o wide
kubectl describe pod <pod> -n <namespace>
kubectl describe node <node>
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A
```

Inspect CNI DaemonSet health and logs using the actual CNI namespace and component names in your cluster.

**For AWS EKS, also check**
- Free IP addresses in subnets used by worker nodes and Pods.
- VPC CNI/IPAM logs and configuration.
- ENI and per-instance Pod/IP limits for the selected instance type.
- Route tables, security groups, network ACLs, and AZ-specific differences.
- Whether prefix delegation or custom networking is enabled and configured as intended.

**Fix:** Restore CNI health or plan IP/subnet/node capacity. Adding nodes may not solve the problem if the underlying subnet is out of IP addresses.

### 4.7 Node-level connection tracking or connection churn

**Symptoms**
- Intermittent failures affect many Pods on a node.
- Failures increase with high connection volume or many short-lived connections.

**Checks**
- Compare affected Pods by node and AZ.
- Review node kernel/system logs, CNI metrics, conntrack usage, TCP retransmissions and connection counts through approved node diagnostics.
- Inspect client connection reuse and pool saturation.
- Compare new-connection rate with request rate.

**Fix:** Reuse HTTP/TCP connections, set reasonable pool limits, reduce unnecessary connection churn, and address node/CNI capacity. Treat conntrack exhaustion as a hypothesis until metrics or kernel evidence support it. Do not flush conntrack or change kernel limits blindly; either action can disrupt active traffic or consume additional resources.

### 4.8 Outbound traffic / NAT / firewall failure

**Symptoms**
- Cluster-local calls succeed, but an external payment provider or API is unreachable.
- Only private-subnet workloads or one AZ fail.

**Checks from an approved diagnostic container**

```bash
nslookup <external-hostname>
curl -v --connect-timeout 3 https://<external-hostname>/
```

A successful DNS lookup does not prove TCP connectivity or TLS is working. A failed HTTP request can also reflect application authentication, a remote outage, or rate limiting.

**Inspect**
- Route table and NAT gateway/egress gateway path for private subnets.
- Firewall, security group, network ACL, proxy and destination port.
- DNS resolver and TLS trust chain.
- Egress allowlists and remote API rate limits.
- AZ-specific NAT/routing design and flow logs where available.

**Fix:** Correct the specific missing route/rule or restore the failed egress component. For sensitive APIs, test only approved endpoints and avoid exposing tokens in shell history or logs.

### 4.9 TLS, certificates, and protocol mismatch

**Symptoms**
- Certificate expiry or hostname errors.
- TLS handshake failures.
- One hop speaks HTTPS while the next expects HTTP, or vice versa.

**Checks**
- Identify where TLS terminates: cloud load balancer, Ingress/Gateway, service mesh, or application.
- Verify certificate expiry, SAN/hostname, CA trust chain, SNI, and backend protocol.
- Inspect the relevant Kubernetes TLS Secret metadata and controller events without printing private key contents.

**Fix:** Rotate certificates safely, verify the full chain and hostname, and align protocols at every hop. Never disable certificate validation as a production workaround.

### 4.10 MTU, fragmentation, or large-request failures

**Symptoms**
- Small requests succeed, but larger payloads or some TLS sessions stall.
- Symptoms appear after enabling an overlay, VPN, tunnel, or service mesh.

**Checks**
- Compare failure rate by payload size and destination.
- Review MTU settings on the CNI, node interface, tunnel and underlying network.
- Inspect retransmissions and packet drops using approved packet-capture tools.
- Check proxy request-body limits separately; a 413 or explicit proxy rejection is not an MTU issue.

**Fix:** Align effective MTU across the actual path and ensure required path-MTU discovery behavior works. Change MTU only after confirming the path and validating the effect.

### 4.11 Latency, packet loss, and timeouts

**Symptoms**
- P95/P99 latency rises while average latency appears normal.
- Intermittent timeouts or retries amplify load.
- Only a subset of downstream calls are slow.

**Checks**
- Use distributed traces to identify the slow hop.
- Compare application duration with DNS, connection, TLS and response time where instrumentation allows.
- Inspect ingress latency, TCP retransmissions, node resource pressure, connection pools, and downstream service/database metrics.
- Compare retries and new connections with incoming request rate.

**Mitigation**
- Set explicit request deadlines and per-hop timeouts.
- Use bounded retries with exponential backoff and jitter for transient failures.
- Retry only when the operation is safe to retry; use idempotency keys or application-level deduplication for writes.
- Limit concurrency and connection pools to protect dependencies.
- Use circuit breakers or load shedding where appropriate.
- Do not increase timeouts or retry counts globally without understanding the bottleneck.

For Go services, propagate request contexts to outbound calls and database operations. Ensure HTTP clients are reused rather than created for every request.

### 4.12 Readiness and liveness probe mistakes

**Symptoms**
- Pods are Running but do not receive traffic.
- Rollouts stall or Pods restart during a temporary dependency slowdown.

**Checks**

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
kubectl logs <pod> -n <namespace> --previous
```

**Principles**
- Readiness decides whether a Pod should receive normal Service traffic.
- Liveness determines whether Kubernetes should restart a container.
- Startup probes can protect slow-starting applications from premature liveness failures.
- Probe paths and ports must match the application.
- Liveness should generally test whether the process can make progress, not whether every downstream dependency is healthy.

Configure realistic timeout, period, failure threshold, and startup behavior based on measured startup and recovery characteristics.

## 5. Safe diagnostic command reference

### Workload, Service, and events

```bash
kubectl get pods -n <namespace> -o wide
kubectl get svc -n <namespace> -o yaml
kubectl get endpointslice -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl describe node <node>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

### Logs and resources

```bash
kubectl logs <pod> -n <namespace> --since=15m
kubectl logs <pod> -n <namespace> --previous
kubectl top pods -n <namespace>
kubectl top nodes
```

Metrics commands require a working metrics pipeline. Missing metrics are not proof that CPU/memory usage is zero.

### Connectivity

```bash
# Run only if these tools exist in the image or approved debug container
nslookup <service>.<namespace>.svc.cluster.local
curl -v --connect-timeout 3 http://<service>.<namespace>.svc.cluster.local:<port>/healthz
curl -v --connect-timeout 3 https://<external-hostname>/
```

A health endpoint test proves only that this path returned that response at that time. It does not prove all application functions or dependencies are healthy.

## 6. Production incident decision tree

```text
Request fails
|
+-- Does the app work directly inside its Pod?
|   +-- No -> app listener, port, process, local resource issue
|   +-- Yes
|
+-- Can the caller resolve the Service name?
|   +-- No -> name/namespace, DNS, CoreDNS, DNS policy
|   +-- Yes
|
+-- Does the Service have ready EndpointSlices?
|   +-- No -> selectors, readiness, rollout, no available replicas
|   +-- Yes
|
+-- Can the caller connect to the Service?
|   +-- No -> port/targetPort, NetworkPolicy, CNI, routes/firewall
|   +-- Yes
|
+-- Does internal access work but external access fail?
|   +-- Yes -> public DNS, load balancer, Ingress/Gateway, TLS
|   +-- No
|
+-- Is only outbound access failing?
|   +-- Yes -> egress policy, route/NAT, firewall, external dependency
|   +-- No -> trace latency, packet loss, saturation, retries and dependencies
```

This is a guide for narrowing the fault domain, not a substitute for checking actual metrics and logs.

## 7. Observability to keep before the next incident

Monitor and alert on:
- Request rate, error rate, and P50/P95/P99 latency by route and dependency.
- DNS lookup latency/failures and CoreDNS health.
- Ingress/Gateway 4xx/5xx, upstream latency, active connections and target health.
- Ready endpoints, unavailable replicas, restart counts and rollout status.
- CNI errors, Pod IP/subnet capacity, node network drops and TCP retransmissions where available.
- NAT gateway errors/capacity signals, flow logs and firewall denies where supported.
- Connection-pool utilization, open connections, retries, timeout counts and circuit-breaker state.
- Distributed traces with request/correlation IDs; avoid logging credentials or sensitive payloads.

Define service-level objectives and alert on user-visible impact. A single CPU or network metric without context is rarely sufficient to diagnose a request failure.

## 8. Recovery and incident closure checklist

- [ ] Identify the failing hop and preserve useful timestamps/logs/metrics.
- [ ] Confirm blast radius and communicate impact to the incident owner.
- [ ] Apply the smallest approved mitigation; avoid unrelated changes.
- [ ] Verify recovery from the original caller path, not just from inside a Pod.
- [ ] Confirm error rate, P95/P99 latency, retries and downstream health return to expected levels.
- [ ] Check that the mitigation did not weaken security or break unrelated traffic.
- [ ] Record root cause, contributing factors, remediation owner and due date.
- [ ] Add a dashboard, alert, automated test, capacity plan or runbook improvement.

## 9. Local validation (no GitHub Actions)

These commands render manifests locally; they do not deploy anything:

```bash
kubectl kustomize examples/go-api/overlays/dev
kubectl apply --dry-run=client -k examples/go-api/overlays/dev
```

If a cluster is available, inspect the proposed changes before applying:

```bash
kubectl diff -k examples/go-api/overlays/dev
```

Use your organization's approved validation tools and change process. This repository intentionally does not add GitHub Actions or other workflow files.

## 10. Official references

- Kubernetes Services: https://kubernetes.io/docs/concepts/services-networking/service/
- DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- NetworkPolicy: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Debugging Services: https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/
- Debugging DNS resolution: https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
- Liveness, readiness, and startup probes: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
- Gateway API: https://gateway-api.sigs.k8s.io/
- Amazon EKS networking: https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html
- Amazon VPC CNI: https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html
