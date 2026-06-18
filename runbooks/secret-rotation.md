# Runbook: Rotate Secret With ESO

## Mục tiêu

Kiểm tra secret được đồng bộ vào Kubernetes trong dưới 60 giây và pod không restart.

## Kiểm tra secret hiện tại

```powershell
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
kubectl get pod -n demo -l app=secret-reader
```

## Rotate trong lab local

Lab minikube dùng ESO `fake` provider. Để mô phỏng rotate, sửa `eso/secret-store.yaml`:

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
git add eso
git commit -m "test: rotate db secret"
git push
```

Đợi ArgoCD sync, rồi kiểm tra:

```powershell
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
kubectl get pod -n demo -l app=secret-reader
```

## Kỳ vọng

- Secret đổi trong khoảng `refreshInterval`.
- Pod `secret-reader` giữ nguyên `AGE`, không restart.
- Repo không chứa AWS credential hoặc password thật.
