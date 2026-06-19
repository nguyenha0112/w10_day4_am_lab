# Runbook: Supply Chain Verification

## Mục tiêu

Image phải được scan bằng Trivy, ký bằng Cosign, và chỉ image đã ký mới được admission cho chạy.

## Kiểm tra CI

Vào GitHub Actions của repo:

```text
https://github.com/nguyenha0112/w10_day4_am_lab/actions
```

Workflow cần có các bước:

- Build local image.
- Trivy scan image.
- Push image lên GHCR.
- Cosign sign image with private key from GitHub Secrets.
- Commit tag mới vào `app-api/rollout.yaml`.

## Bật admission verify

Chỉ bật sau khi image đã ký:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

## Kiểm tra reject image chưa ký

```powershell
kubectl run unsigned-test -n demo --image=nginx:1.27 --dry-run=server
```

Kỳ vọng: admission reject vì image không match chữ ký tin cậy.

## Kiểm tra image đã ký

```powershell
kubectl get rollout api -n demo
kubectl get pods -n demo -l app=api
```

## Cosign key setup

Repo commits only the public key:

```text
signing/cosign.pub
```

Do not commit the private key. Add these GitHub repository secrets:

- `COSIGN_PRIVATE_KEY`: full content of local `signing/cosign.key`
- `COSIGN_PASSWORD`: password used when generating the key pair

Kỳ vọng: image được build từ workflow `build-push.yml` pass admission.
