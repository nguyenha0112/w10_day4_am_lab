# Runbook: Rotate Secret With ESO + AWS Secrets Manager

## Mục tiêu

Kiểm tra secret được đồng bộ từ AWS Secrets Manager vào Kubernetes trong dưới 60 giây và pod không restart.

## Kiến trúc

```text
AWS Secrets Manager
        ↓
ESO SecretStore aws-store
        ↓
ExternalSecret db-secret
        ↓
Kubernetes Secret db-secret
        ↓
Pod secret-reader mount /etc/db/password
```

## Chuẩn bị AWS secret

Tạo secret trong AWS Secrets Manager ở region:

```text
ap-southeast-1
```

Tên secret:

```text
/w10/demo/db-password
```

Giá trị secret nên là plaintext password, ví dụ:

```text
initial-db-password
```

## Chuẩn bị AWS credential trong Kubernetes

Không commit AWS credential vào Git.

Tạo Kubernetes Secret thủ công:

```powershell
kubectl create secret generic aws-credentials -n demo `
  --from-literal=access-key-id="YOUR_AWS_ACCESS_KEY_ID" `
  --from-literal=secret-access-key="YOUR_AWS_SECRET_ACCESS_KEY"
```

Secret này được `eso/secret-store.yaml` dùng để đăng nhập AWS Secrets Manager.

## Kiểm tra ESO sync

```powershell
kubectl get pods -n external-secrets
kubectl get secretstore aws-store -n demo
kubectl get externalsecret db-secret -n demo
kubectl get secret db-secret -n demo
```

Đọc giá trị đã sync:

```powershell
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Kiểm tra pod đọc secret:

```powershell
kubectl get pod -n demo -l app=secret-reader
kubectl logs -n demo deploy/secret-reader
```

## Rotate secret

Vào AWS Secrets Manager và cập nhật value của secret:

```text
/w10/demo/db-password
```

Ví dụ đổi thành:

```text
rotated-db-password
```

Đợi tối đa `refreshInterval: 30s`, sau đó kiểm tra lại:

```powershell
kubectl get secret db-secret -n demo -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
kubectl get pod -n demo -l app=secret-reader
kubectl logs -n demo deploy/secret-reader
```

## Kỳ vọng

- Kubernetes Secret `db-secret` đổi theo AWS Secrets Manager.
- Pod `secret-reader` giữ nguyên `AGE`, không restart.
- Repo không chứa AWS credential hoặc database password thật.
