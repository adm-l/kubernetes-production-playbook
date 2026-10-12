# kubectl quick reference

## Inspect
```bash
kubectl get nodes -o wide
kubectl get ns
kubectl get deploy,rs,pods -n production
kubectl get pods -o wide -n production
kubectl get svc,ingress -n production
kubectl get events -n production --sort-by=.lastTimestamp
kubectl describe pod <pod> -n production
```

## Logs and process
```bash
kubectl logs deploy/payment-api -n production
kubectl logs <pod> -n production --previous
kubectl logs <pod> -n production -c <container>
kubectl exec -it <pod> -n production -- sh
kubectl port-forward svc/payment-api 8080:80 -n production
```

## Render and deploy
```bash
kubectl kustomize examples/go-api/overlays/dev
kubectl diff -k examples/go-api/overlays/dev
kubectl apply --dry-run=client -k examples/go-api/overlays/dev
kubectl apply -k examples/go-api/overlays/dev
kubectl rollout status deploy/payment-api -n production
kubectl rollout history deploy/payment-api -n production
kubectl rollout undo deploy/payment-api -n production
```

## Scaling and networking
```bash
kubectl top pods -n production
kubectl describe hpa payment-api -n production
kubectl get endpointslices -n production
kubectl get networkpolicy -n production
```

Avoid dumping Secret values or sensitive environment variables into terminals, logs, or tickets. Use production commands under approved access and change-management procedures.
