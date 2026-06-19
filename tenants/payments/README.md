# Payments tenant

This folder onboards the `payments` team as a separate Kubernetes tenant.

## What is included

- `namespace.yaml`: creates namespace `payments` and labels it for Sigstore image policy enforcement.
- `rbac.yaml`: grants user `payments-dev` workload permissions only inside `payments`.
- `quota-limitrange.yaml`: sets tenant budget and default container requests/limits.
- `networkpolicies.yaml`: denies ingress by default and restricts egress to same namespace plus DNS.

## Verification

```powershell
kubectl auth can-i create deploy -n payments --as payments-dev
kubectl auth can-i create deploy -n demo --as payments-dev
kubectl auth can-i get secrets -n payments --as payments-dev
kubectl auth can-i create rolebindings -n payments --as payments-dev
kubectl get resourcequota,limitrange -n payments
kubectl get networkpolicy -n payments
kubectl get deploy,svc -n payments
```

Expected:

- `payments-dev` can create workloads in `payments`.
- `payments-dev` cannot create workloads in `demo`.
- `payments-dev` cannot read secrets or modify rolebindings.
- Pods without explicit limits receive defaults from `LimitRange`.
- Pods exceeding `ResourceQuota` are rejected.

NetworkPolicy enforcement requires a CNI that supports NetworkPolicy, such as Calico.
