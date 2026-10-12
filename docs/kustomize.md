# Kustomize workflow

Kustomize composes a base with environment-specific patches. It is built into recent kubectl versions.

```bash
kubectl kustomize examples/go-api/overlays/dev
kubectl diff -k examples/go-api/overlays/dev
kubectl apply --dry-run=client -k examples/go-api/overlays/dev
```

Review rendered output before applying. Keep environment differences explicit: image digest, replicas, resource requests, ingress host and scaling range. Do not put secrets in overlays. Use reviewed CI/GitOps for shared environments where required by company policy.
