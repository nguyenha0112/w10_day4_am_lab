# Runbook: Rotate Secret With ESO

## Mục tiêu

Kiểm tra secret được đồng bộ vào Kubernetes trong dưới 60 giây và pod không restart.

## Kiến trúc lab local

Lab dùng ESO `fake` provider để chạy nhanh trên minikube, không cần AWS account thật.

```text
fake SecretStore
        ↓
ExternalSecret db-secret
        ↓
Kubernetes Secret db-secret
        ↓
Pod secret-reader mount /etc/db/password
```

## Kiểm tra secret hiện tại

```powershell
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
kubectl get pod -n demo -l app=secret-reader
kubectl logs -n demo deploy/secret-reader
```

## Rotate trong lab local

Sửa `eso/secret-store.yaml`:

```yaml
value: rotated-db-password
version: v2
```

Sau đó sửa `eso/external-secret.yaml`:

```yaml
version: v2
```

Commit và push:

```powershell
git add eso runbooks/secret-rotation.md
git commit -m "test: rotate db secret"
git push
```

Đợi ArgoCD sync, rồi kiểm tra:

```powershell
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
kubectl get pod -n demo -l app=secret-reader
```

## Kỳ vọng

- Secret đổi trong khoảng `refreshInterval: 30s`.
- Pod `secret-reader` giữ nguyên `AGE`, không restart.
- Repo không chứa AWS credential hoặc database password thật.
