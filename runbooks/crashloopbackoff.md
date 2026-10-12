# Runbook: CrashLoopBackOff

## Diagnose
1. `kubectl get pods -n <namespace> -o wide`
2. `kubectl describe pod <pod> -n <namespace>`
3. `kubectl logs <pod> -n <namespace> --previous`
4. Inspect events, exit code, image, command/args, config, Secret references, probes and dependency availability.
5. For OOMKilled, inspect memory use/limit and profile. For probe failures, verify endpoint, startup time and thresholds.

## Mitigate
- Roll back a known bad release if evidence links the incident to it.
- Correct invalid config/image reference through the normal reviewed deployment path.
- Do not just increase limits or disable probes without evidence.
- Do not print secret values into tickets or shared logs.

## Verify
Check rollout status, Pod readiness, request success rate, latency and downstream health. Record cause, mitigation and follow-up.
