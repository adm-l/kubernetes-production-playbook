# AWS EKS notes

This is a planning checklist, not an apply-ready AWS configuration.

## Architecture decisions
- Choose VPC/subnet layout, IP capacity, availability zones, private/public endpoint access and egress model.
- Select managed node groups or a compatible autoscaling approach such as Karpenter; define instance families, disruption and upgrade policy.
- Use IAM roles for workloads through EKS Pod Identity or IRSA according to your platform standard. Avoid static AWS credentials in Kubernetes Secrets.
- Decide how Services/Ingress map to AWS load balancers. AWS Load Balancer Controller and its IAM permissions are separate prerequisites.
- Confirm VPC CNI, CoreDNS, kube-proxy or replacements, EBS CSI and metrics-server requirements.
- Define cluster and workload logging, audit logs, metrics, tracing and alert ownership.
- Plan Kubernetes version upgrades, add-on compatibility, node replacement and rollback.
- Apply least privilege to cluster access, namespace RBAC and AWS permissions.

## Before deploying an app
- Verify cluster context and namespace; never assume current kubectl context.
- Check node and Pod IP capacity, quotas, storage classes and metrics API.
- Confirm DNS, image registry access, TLS certificates and dependency egress.
- Confirm load balancer controller, ingress class and security group policy.
- Review resource requests against node capacity and autoscaling behavior.
- Validate manifests in CI and inspect `kubectl diff` before applying.

## Useful commands
```bash
aws eks describe-cluster --name <cluster-name> --region <region>
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass
kubectl top nodes
kubectl get events -A --sort-by=.lastTimestamp
```

Never paste AWS access keys, kubeconfig tokens or secret values into this repository. EKS architecture should be implemented with reviewed IaC (for example Terraform or CloudFormation) and your organization's approval process.
