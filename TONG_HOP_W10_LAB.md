# Tổng hợp W10 Lab: RBAC, Admission, Secrets, Supply Chain, Multi-tenant

File này dùng để ôn tập và giải thích repo W10. Nội dung bám theo 2 slide:

- Buổi sáng: RBAC + Admission Policy.
- Buổi chiều: Secrets + Supply Chain + Platform Integration.
- Challenge: onboard team `payments` theo hướng multi-tenant an toàn.

Project nằm trong thư mục `temp/`.

## 1. Kết luận trạng thái

Repo hiện đã đủ các phần chính:

| Nhóm yêu cầu | Trạng thái | Tác dụng | File/thư mục |
|---|---:|---|---|
| GitOps App of Apps | Đủ | Quản lý toàn bộ platform từ Git, ArgoCD tự sync các app con vào cluster | `argocd/root.yaml`, `argocd/apps/*.yaml` |
| RBAC 3 user | Đủ | Phân quyền ai được làm gì; giới hạn `alice`, `bob`, `carol` theo vai trò | `rbac/roles.yaml`, `rbac/rolebindings.yaml` |
| Gatekeeper Admission | Đủ | Chặn manifest xấu trước khi vào cluster, ví dụ `:latest`, thiếu limits, chạy root | `gatekeeper/templates/`, `gatekeeper/constraints/` |
| ESO secret rotation | Đủ, dùng AWS Secrets Manager | Sync secret từ AWS vào Kubernetes Secret, chứng minh rotate không cần restart pod | `eso/`, `argocd/apps/eso*.yaml` |
| Trivy scan | Đủ | Scan CVE trong image ở CI, fail pipeline nếu có lỗi HIGH/CRITICAL | `.github/workflows/build-push.yml` |
| Cosign signing | Đủ, dùng key-pair | Ký image sau khi build/scan để chứng minh image đến từ pipeline tin cậy | `.github/workflows/build-push.yml`, `signing/cosign.pub` |
| Policy Controller verify image | Đủ | Admission kiểm tra chữ ký image, reject image chưa ký hoặc không đúng public key | `argocd/apps/policy-controller.yaml`, `policies/cluster-image-policy.yaml` |
| Runbook/ADR | Đủ | Ghi cách vận hành, rotate secret, kiểm tra supply chain, xử lý ngoại lệ CVE có thời hạn | `runbooks/*.md`, `policies/README.md` |
| Challenge `payments` | Đủ | Onboard team mới vào namespace riêng, cô lập bằng RBAC, quota, limits, network policy và GitOps app riêng | `tenants/payments/`, `apps/payments/`, `argocd/apps/payments*.yaml` |

## 2. Phương án thay thế cần báo rõ

Có 1 điểm cần báo rõ khi trình bày:

1. Cosign dùng key-pair, không dùng keyless

Project hiện dùng Cosign key-pair:

- Public key commit trong `signing/cosign.pub`.
- Private key nằm local ở `signing/cosign.key` và bị `.gitignore`.
- GitHub Actions cần secrets:

```text
COSIGN_PRIVATE_KEY = nội dung file signing/cosign.key
COSIGN_PASSWORD    = password lúc generate key pair
```

Cluster verify image bằng public key trong `policies/cluster-image-policy.yaml`.

## 3. Kiểm tra tĩnh đã chạy

Đã chạy dry-run các manifest chính:

```powershell
kubectl apply --dry-run=client --validate=false `
  -f rbac `
  -f gatekeeper/templates `
  -f gatekeeper/constraints `
  -f eso `
  -f policies `
  -f tenants/payments `
  -f apps/payments `
  -f argocd/apps/payments.yaml `
  -f argocd/apps/payments-app.yaml
```

Kết quả: các resource parse được và dry-run thành công, gồm RBAC, Gatekeeper, ESO, ClusterImagePolicy, namespace/app `payments`.

Ngoài ra đã kiểm tra các cấu hình Cosign chính hiện dùng key-pair và không còn cấu hình OIDC/keyless cũ trong workflow hoặc policy.

## 4. GitOps và ArgoCD

GitOps nghĩa là Git là nguồn sự thật. Bạn sửa manifest, commit, push; ArgoCD đọc Git và sync vào cluster.

File chính:

- `argocd/root.yaml`: root app theo pattern App of Apps.
- `argocd/apps/*.yaml`: các child app.

Các child app quan trọng:

| App | Tác dụng |
|---|---|
| `app-common.yaml` | Tạo namespace `demo` |
| `app-api.yaml` | Deploy API Rollout/Service/ServiceMonitor |
| `app-analysis.yaml` | Deploy AnalysisTemplate |
| `app-alert.yaml` | Deploy PrometheusRule |
| `rbac.yaml` | Sync RBAC |
| `gatekeeper.yaml` | Cài Gatekeeper |
| `gatekeeper-templates.yaml` | Sync ConstraintTemplate |
| `gatekeeper-constraints.yaml` | Sync Constraint |
| `eso.yaml` | Cài External Secrets Operator |
| `eso-config.yaml` | Sync SecretStore/ExternalSecret/secret-reader |
| `policy-controller.yaml` | Cài Sigstore Policy Controller |
| `policies.yaml` | Sync ClusterImagePolicy |
| `payments.yaml` | Sync tenant `payments` |
| `payments-app.yaml` | Sync app mẫu của team `payments` |

## 5. RBAC

RBAC trả lời câu hỏi:

```text
Ai được làm gì?
```

File:

- `rbac/roles.yaml`
- `rbac/rolebindings.yaml`
- `argocd/apps/rbac.yaml`

### Vai trò

| User | Role | Phạm vi | Ý nghĩa |
|---|---|---|---|
| `alice` | `developer` | Namespace `demo` | Dev thao tác workload |
| `bob` | `sre` | Toàn cluster | SRE vận hành pod/deployment/rollout |
| `carol` | `viewer` | Toàn cluster | Chỉ đọc |

Điểm đã chỉnh: `developer` không còn quyền CRUD `secrets`. Điều này đúng hơn với least privilege.

Kiểm tra:

```powershell
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Kỳ vọng:

```text
yes
no
yes
no
```

## 6. Gatekeeper Admission Policy

Gatekeeper trả lời câu hỏi:

```text
Manifest này có hợp lệ không?
```

RBAC kiểm tra "ai được làm", Gatekeeper kiểm tra "thứ được tạo có an toàn không".

File:

- `gatekeeper/templates/*.yaml`
- `gatekeeper/constraints/*.yaml`
- `argocd/apps/gatekeeper.yaml`
- `argocd/apps/gatekeeper-templates.yaml`
- `argocd/apps/gatekeeper-constraints.yaml`

### Luật đang có

| Policy | Tác dụng |
|---|---|
| `K8sDisallowLatestTag` | Cấm image `:latest` hoặc image không có tag rõ ràng |
| `K8sRequiredContainerLimits` | Bắt buộc `resources.limits.cpu` và `resources.limits.memory` |
| `K8sDisallowRootUser` | Cấm `runAsUser: 0` |
| `K8sDisallowHostNetwork` | Cấm `hostNetwork: true` |
| `K8sMaxReplicas` | Chặn Deployment/Rollout nếu `replicas > 5` |

Các constraint đều đang dùng:

```yaml
enforcementAction: deny
```

Nghĩa là vi phạm sẽ bị reject thật.

Các namespace hệ thống được exclude:

```text
argocd
argo-rollouts
cosign-system
external-secrets
gatekeeper-system
kube-system
monitoring
```

Lý do: không để policy làm hỏng controller/hạ tầng.

## 7. API, Rollout, Monitoring

File chính:

- `src/api/app.py`
- `src/api/Dockerfile`
- `app-api/rollout.yaml`
- `app-api/service.yaml`
- `app-api/servicemonitor.yaml`
- `app-analysis/analysis-template.yaml`
- `app-alert/prometheus-rules.yaml`

API được deploy bằng Argo Rollouts:

- Canary: `10% -> 50% -> 100%`.
- Có AnalysisTemplate để query Prometheus.
- Có ServiceMonitor để Prometheus scrape `/metrics`.

App API hiện hợp lệ với Gatekeeper:

- Image có tag cụ thể: `ghcr.io/nguyenha0112/w10-api:0.0.1`.
- Có CPU/memory limits.
- Không chạy root.
- Không bật hostNetwork.
- `replicas: 4`, nhỏ hơn max `5`.

## 8. ESO Secret Rotation

ESO dùng để không commit secret thật vào Git. Project hiện dùng AWS Secrets Manager thật.

File:

- `argocd/apps/eso.yaml`
- `argocd/apps/eso-config.yaml`
- `eso/secret-store.yaml`
- `eso/external-secret.yaml`
- `eso/secret-reader.yaml`

Flow:

```text
AWS Secrets Manager -> SecretStore aws-store -> ExternalSecret -> Kubernetes Secret db-secret -> pod secret-reader mount volume
```

Chi tiết:

- `SecretStore` tên `aws-store`.
- AWS region: `ap-southeast-1`.
- AWS Secrets Manager key: `/w10/demo/db-password`.
- `ExternalSecret` tạo Kubernetes Secret tên `db-secret`.
- `refreshInterval: 30s`, đạt yêu cầu rotate dưới 60 giây.
- `secret-reader` mount Secret vào `/etc/db` và đọc file mỗi 10 giây.

Trước khi ESO sync được, cần tạo Kubernetes Secret chứa AWS credential:

```powershell
kubectl create secret generic aws-credentials -n demo `
  --from-literal=access-key-id="YOUR_AWS_ACCESS_KEY_ID" `
  --from-literal=secret-access-key="YOUR_AWS_SECRET_ACCESS_KEY"
```

Không commit AWS credential vào Git.

Kiểm tra:

```powershell
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl logs -n demo deploy/secret-reader
```

## 9. Supply Chain: Trivy, Cosign, Policy Controller

Mục tiêu:

```text
Trivy scan -> Cosign sign -> Admission verify
```

File:

- `.github/workflows/build-push.yml`
- `signing/cosign.pub`
- `policies/cluster-image-policy.yaml`
- `argocd/apps/policy-controller.yaml`
- `argocd/apps/policies.yaml`
- `policies/README.md`

### GitHub Actions

Workflow làm các việc chính:

1. Build image từ `src/api`.
2. Scan image bằng Trivy.
3. Fail nếu có CVE `HIGH` hoặc `CRITICAL`.
4. Push image lên GHCR.
5. Ký image bằng Cosign key-pair.
6. Update `app-api/rollout.yaml`.
7. Commit version mới và tạo Git tag.

Đoạn ký image:

```bash
cosign sign --key env://COSIGN_PRIVATE_KEY "$tag"
```

### ClusterImagePolicy

`policies/cluster-image-policy.yaml` chỉ tin image:

```text
ghcr.io/nguyenha0112/w10-api**
```

và verify bằng public key:

```text
signing/cosign.pub
```

Policy Controller chỉ enforce namespace có label:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

Lưu ý: chỉ bật label sau khi image đã được ký, nếu không rollout có thể bị reject.

## 10. Challenge payments tenant

Phần này đáp ứng yêu cầu onboard team `payments` theo hướng multi-tenant.

File:

- `tenants/payments/namespace.yaml`
- `tenants/payments/rbac.yaml`
- `tenants/payments/quota-limitrange.yaml`
- `tenants/payments/networkpolicies.yaml`
- `tenants/payments/README.md`
- `apps/payments/deployment.yaml`
- `apps/payments/service.yaml`
- `apps/payments/README.md`
- `argocd/apps/payments.yaml`
- `argocd/apps/payments-app.yaml`

### Namespace

Namespace `payments` có label:

```yaml
policy.sigstore.dev/include: "true"
```

Nghĩa là image signature policy cũ tự áp dụng cho team mới.

### RBAC payments

User:

```text
payments-dev
```

Được phép thao tác workload trong namespace `payments`, nhưng không được:

- tạo workload trong `demo`
- đọc `secrets`
- sửa `rolebindings`

Kiểm tra:

```powershell
kubectl auth can-i create deploy -n payments --as payments-dev
kubectl auth can-i create deploy -n demo --as payments-dev
kubectl auth can-i get secrets -n payments --as payments-dev
kubectl auth can-i create rolebindings -n payments --as payments-dev
```

Kỳ vọng:

```text
yes
no
no
no
```

### ResourceQuota và LimitRange

File:

```text
tenants/payments/quota-limitrange.yaml
```

Tác dụng:

- `ResourceQuota`: đặt ngân sách tổng CPU/RAM/pod/service.
- `LimitRange`: cấp default request/limit cho container thiếu khai báo.

### NetworkPolicy

File:

```text
tenants/payments/networkpolicies.yaml
```

Có 2 policy:

- `default-deny-ingress`: chặn traffic vào payments nếu không được allow.
- `restrict-egress`: chỉ cho egress cùng namespace và DNS.

Lưu ý: NetworkPolicy chỉ enforce nếu CNI hỗ trợ, ví dụ Calico.

### App payments

App mẫu:

```text
apps/payments/
```

Deployment `payments-api` dùng image:

```text
ghcr.io/nguyenha0112/w10-api:0.0.1
```

Image này phải được ký bằng workflow Cosign key-pair hiện tại trước khi namespace `payments` enforce signature policy.

## 11. Runbooks và ADR

File:

- `runbooks/secret-rotation.md`
- `runbooks/supply-chain.md`
- `runbooks/adr-cve-exception.md`

Tác dụng:

- `secret-rotation.md`: hướng dẫn rotate secret bằng ESO.
- `supply-chain.md`: hướng dẫn kiểm tra Trivy/Cosign/Policy Controller.
- `adr-cve-exception.md`: mẫu ngoại lệ CVE có thời hạn, không tắt scan toàn cục.

## 12. Checklist kiểm tra khi có cluster

ArgoCD:

```powershell
kubectl get applications -n argocd
```

RBAC:

```powershell
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Gatekeeper:

```powershell
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get k8sdisallowlatesttag
kubectl get k8srequiredcontainerlimits
kubectl get k8sdisallowrootuser
kubectl get k8sdisallowhostnetwork
kubectl get k8smaxreplicas
```

ESO:

```powershell
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl logs -n demo deploy/secret-reader
```

Supply chain:

```powershell
kubectl get pods -n cosign-system
kubectl get clusterimagepolicy
kubectl run unsigned-test -n demo --image=nginx:1.27 --dry-run=server
```

Payments:

```powershell
kubectl get ns payments --show-labels
kubectl get resourcequota,limitrange,networkpolicy -n payments
kubectl get deploy,svc -n payments
kubectl auth can-i create deploy -n payments --as payments-dev
kubectl auth can-i create deploy -n demo --as payments-dev
kubectl auth can-i get secrets -n payments --as payments-dev
kubectl auth can-i create rolebindings -n payments --as payments-dev
```

## 13. Checklist trước khi nộp

Trước khi nộp, cần nhớ:

- Không commit `signing/cosign.key`.
- GitHub repo phải có secrets `COSIGN_PRIVATE_KEY` và `COSIGN_PASSWORD`.
- ESO đang dùng AWS Secrets Manager thật; cần tạo secret `/w10/demo/db-password` trên AWS và tạo Kubernetes Secret `aws-credentials` thủ công.
- NetworkPolicy cần CNI hỗ trợ như Calico để test chặn traffic thật.
- Namespace `payments` đã bật `policy.sigstore.dev/include=true`, nên image phải signed trước khi app sync xanh.

## 14. Một câu giải thích toàn bộ project

Project này là một platform Kubernetes quản lý bằng GitOps: RBAC kiểm soát ai được làm gì, Gatekeeper chặn manifest xấu, ESO quản lý secret không commit vào Git, Trivy/Cosign/Policy Controller bảo vệ image supply chain, và tenant `payments` được cô lập bằng namespace, RBAC, quota, limit và network policy.
