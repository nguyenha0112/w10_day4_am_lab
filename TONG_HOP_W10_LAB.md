# Tổng Hợp W10 Lab: RBAC, Admission, Secrets, Supply Chain

File này giải thích toàn bộ repo lab W10: cấu trúc thư mục, tác dụng từng file, tuần này đã làm thêm gì, và kiến thức trong hai slide buổi sáng/buổi chiều.

## 1. Repo này dùng để làm gì?

Repo này là một repo GitOps cho Kubernetes.

Ý tưởng chính:

```text
Code/YAML nằm trên GitHub
        ↓
ArgoCD đọc repo
        ↓
ArgoCD apply YAML vào Kubernetes
        ↓
Cluster tạo resource thật: Pod, Service, RBAC, Gatekeeper policy, ESO, monitoring...
```

GitHub là nơi lưu "bản thiết kế". Kubernetes là nơi chạy "đồ thật".

## 2. Trạng thái yêu cầu hiện tại

Về mặt file trong repo, các yêu cầu chính của hai slide đã có:

- Buổi sáng:
  - RBAC 3 vai trò: `alice`, `bob`, `carol`.
  - Gatekeeper templates và constraints.
  - 4 admission policy: cấm `latest`, bắt buộc limits, cấm root, cấm hostNetwork.
  - Custom policy: chặn replicas lớn hơn 5.

- Buổi chiều:
  - ESO operator app.
  - ESO config dùng fake provider để chạy được trên minikube không cần AWS thật.
  - Secret reader pod để chứng minh mount secret bằng volume.
  - Trivy scan trong GitHub Actions.
  - Cosign keyless signing trong GitHub Actions.
  - Sigstore Policy Controller app.
  - ClusterImagePolicy verify image đã ký.
  - Runbooks và ADR cho vận hành.

Lưu ý runtime:

Ở lần kiểm tra hiện tại, Docker Desktop chưa chạy nên `kubectl` không kết nối được minikube. Lỗi đang thấy là:

```text
dial tcp 127.0.0.1:<port>: connectex: No connection could be made
dockerDesktopLinuxEngine: The system cannot find the file specified
```

Nghĩa là file repo đủ, nhưng muốn kiểm tra ArgoCD/Kubernetes thật thì cần bật Docker Desktop rồi chạy:

```powershell
minikube start
minikube update-context
kubectl get nodes
kubectl get applications -n argocd
```

## 3. Cấu trúc thư mục tổng quan

```text
.
├── .github/workflows/        # CI/CD GitHub Actions
├── app-alert/                # Alert rule và hướng dẫn email alert
├── app-analysis/             # AnalysisTemplate cho Argo Rollouts
├── app-api/                  # Rollout, Service, ServiceMonitor của API
├── app-common/               # Namespace dùng chung
├── argocd/                   # Root app và các child app ArgoCD
├── eso/                      # SecretStore, ExternalSecret, pod đọc secret
├── gatekeeper/               # ConstraintTemplate và Constraint admission policy
├── policies/                 # Sigstore ClusterImagePolicy
├── rbac/                     # Role, ClusterRole, RoleBinding
├── runbooks/                 # Hướng dẫn vận hành và ADR
├── signing/                  # Ghi chú/public key signing
├── src/api/                  # Source Flask API và Dockerfile
├── README.md                 # README gốc
├── README_LAB_STEPS.md       # Các bước làm lab
└── TONG_HOP_W10_LAB.md       # File tổng hợp này
```

## 4. Nhóm ArgoCD

### `argocd/root.yaml`

Đây là app mẹ theo mô hình App of Apps.

Nó đọc:

```text
argocd/apps
```

Tác dụng:

- Tạo các app con.
- Mỗi app con quản lý một phần platform.
- Khi push GitHub, ArgoCD tự sync lại.

Ví dụ root app tạo ra các app:

```text
common
api
analysis
alert
rbac
gatekeeper
external-secrets
eso-config
policy-controller
image-policies
```

### `argocd/apps/app-common.yaml`

Trỏ ArgoCD tới thư mục:

```text
app-common/
```

Tác dụng: tạo namespace `demo`.

### `argocd/apps/app-api.yaml`

Trỏ ArgoCD tới:

```text
app-api/
```

Tác dụng: deploy API bằng Rollout, Service, ServiceMonitor.

### `argocd/apps/app-analysis.yaml`

Trỏ ArgoCD tới:

```text
app-analysis/
```

Tác dụng: deploy AnalysisTemplate để Argo Rollouts dùng khi canary.

### `argocd/apps/app-alert.yaml`

Trỏ ArgoCD tới:

```text
app-alert/
```

Tác dụng: deploy PrometheusRule cho cảnh báo.

### `argocd/apps/k8s-prometheus.yaml`

Cài kube-prometheus-stack bằng Helm.

Tác dụng:

- Prometheus: thu metrics.
- Alertmanager: gửi alert qua email.
- Grafana: dashboard.
- CRD `ServiceMonitor` và `PrometheusRule`.

Trong file này email đã đổi sang:

```text
nguyen229396@gmail.com
```

Secret email thật không commit vào Git, mà tạo bằng:

```powershell
kubectl create secret generic alertmanager-email -n monitoring --from-literal=password="APP_PASSWORD"
```

### `argocd/apps/k8s-rollout.yaml`

Cài Argo Rollouts controller.

Tác dụng:

- Kubernetes mặc định chỉ có Deployment.
- Argo Rollouts thêm resource `Rollout`, `AnalysisRun`, `AnalysisTemplate`.
- Dùng để canary deploy API.

### `argocd/apps/rbac.yaml`

Trỏ ArgoCD tới:

```text
rbac/
```

Tác dụng: sync Role, ClusterRole, RoleBinding.

### `argocd/apps/gatekeeper.yaml`

Cài OPA Gatekeeper controller bằng Helm.

Tác dụng: bật admission webhook để chặn manifest xấu trước khi vào cluster.

### `argocd/apps/gatekeeper-templates.yaml`

Trỏ ArgoCD tới:

```text
gatekeeper/templates/
```

Tác dụng: deploy ConstraintTemplate, tức là khuôn logic policy.

### `argocd/apps/gatekeeper-constraints.yaml`

Trỏ ArgoCD tới:

```text
gatekeeper/constraints/
```

Tác dụng: bật policy thật từ các template.

### `argocd/apps/eso.yaml`

Cài External Secrets Operator bằng Helm.

Tác dụng:

- Tạo CRD `SecretStore`.
- Tạo CRD `ExternalSecret`.
- Chạy controller để sync secret từ nguồn ngoài vào Kubernetes Secret.

### `argocd/apps/eso-config.yaml`

Trỏ ArgoCD tới:

```text
eso/
```

Tác dụng: sync cấu hình SecretStore, ExternalSecret và pod đọc secret.

### `argocd/apps/policy-controller.yaml`

Cài Sigstore Policy Controller bằng Helm.

Tác dụng:

- Tạo CRD `ClusterImagePolicy`.
- Tạo admission controller verify chữ ký image.
- Chỉ enforce namespace có label `policy.sigstore.dev/include=true`.

### `argocd/apps/policies.yaml`

Trỏ ArgoCD tới:

```text
policies/
```

Tác dụng: sync ClusterImagePolicy.

## 5. Nhóm app API

### `src/api/app.py`

Source code Flask API.

Endpoint chính:

```text
/        # trả JSON ok/version, có thể inject lỗi bằng ERROR_RATE
/healthz # health check
/metrics # Prometheus metrics do prometheus-flask-exporter tạo
```

Tác dụng trong lab:

- Có app thật để deploy.
- Có metrics để Prometheus scrape.
- Có `ERROR_RATE` để test canary/alert.

### `src/api/Dockerfile`

Mô tả cách build image API.

Tác dụng:

- GitHub Actions dùng Dockerfile này để build image.
- Image được push lên GHCR.

### `app-api/rollout.yaml`

Deploy API bằng Argo Rollouts.

Phần image:

```yaml
image: ghcr.io/nguyenha0112/w10-api:0.0.1
```

Tác dụng:

- Chạy API bằng custom resource `Rollout`.
- Có canary strategy: 10%, 50%, 100%.
- Dùng `AnalysisTemplate` để kiểm tra metric trong lúc rollout.

Lưu ý:

Nếu người khác bị `InvalidImageName`, kiểm tra dòng `image:` trong file này. Format đúng là:

```text
ghcr.io/<github-user>/<image-name>:<tag>
```

### `app-api/service.yaml`

Tạo Service tên `api`.

Tác dụng:

- Expose pod API trong cluster.
- Service port `80` trỏ tới container port `8080`.
- Prometheus cần Service này để scrape metrics.

### `app-api/servicemonitor.yaml`

Tạo ServiceMonitor cho Prometheus.

Tác dụng:

- Nói cho Prometheus scrape `/metrics` của Service `api`.
- Nếu thiếu CRD ServiceMonitor, app API có thể sync fail cho tới khi kube-prometheus-stack cài xong.

## 6. Nhóm analysis và alert

### `app-analysis/analysis-template.yaml`

Định nghĩa AnalysisTemplate.

Tác dụng:

- Argo Rollouts dùng nó để query Prometheus.
- Nếu metric tốt thì rollout tiếp.
- Nếu metric xấu thì rollout fail/rollback.

Lưu ý:

`kubectl get analysisrun -n demo` có thể chưa có gì ở lần deploy đầu. AnalysisRun thường sinh ra khi có lần update/canary tiếp theo, không nhất thiết sinh ở initial deploy.

### `app-alert/prometheus-rules.yaml`

Tạo PrometheusRule.

Tác dụng:

- Định nghĩa alert khi API error rate cao hoặc SLO thấp.
- Alert sẽ đi qua Alertmanager.

### `app-alert/email-secret.yaml.example`

File mẫu secret email.

Tác dụng:

- Cho biết Secret thật cần key `password`.
- File thật không commit lên Git.

## 7. Nhóm RBAC

### `rbac/roles.yaml`

Định nghĩa quyền.

Có 3 vai trò:

```text
developer
sre
viewer
```

Tác dụng:

- `developer`: thao tác workload trong namespace `demo`.
- `sre`: xem và thao tác pod ở toàn cluster.
- `viewer`: chỉ đọc tài nguyên.

### `rbac/rolebindings.yaml`

Gán user vào quyền.

Mapping:

```text
alice -> developer
bob   -> sre
carol -> viewer
```

Test theo slide:

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

Ý nghĩa kiến thức:

- RBAC trả lời câu hỏi: "Ai được làm gì?"
- `Role` bó trong namespace.
- `ClusterRole` áp dụng toàn cluster.
- `RoleBinding`/`ClusterRoleBinding` gán quyền cho user.

## 8. Nhóm Gatekeeper Admission Policy

Gatekeeper trả lời câu hỏi:

```text
Manifest có hợp lệ không?
```

Khác với RBAC:

- RBAC kiểm "ai".
- Admission policy kiểm "cái gì".

### `gatekeeper/templates/k8sdisallowlatesttag.yaml`

Template logic cấm image tag `:latest`.

Tác dụng: tránh deploy image không cố định version.

### `gatekeeper/templates/k8srequiredcontainerlimits.yaml`

Template logic bắt container có:

```text
resources.limits.cpu
resources.limits.memory
```

Tác dụng: tránh pod ăn hết tài nguyên node.

### `gatekeeper/templates/k8sdisallowrootuser.yaml`

Template logic cấm:

```yaml
runAsUser: 0
```

Tác dụng: tránh container chạy quyền root.

### `gatekeeper/templates/k8sdisallowhostnetwork.yaml`

Template logic cấm:

```yaml
hostNetwork: true
```

Tác dụng: tránh pod dùng network namespace của node.

### `gatekeeper/templates/k8smaxreplicas.yaml`

Custom template.

Tác dụng: chặn Deployment/Rollout nếu `replicas > 5`.

### `gatekeeper/constraints/*.yaml`

Các file này bật template thành policy thật.

Ví dụ:

```text
require-container-limits.yaml
```

dùng template `K8sRequiredContainerLimits` để enforce bắt buộc limits.

Các constraints có exclude namespace hạ tầng:

```text
argocd
argo-rollouts
cosign-system
external-secrets
gatekeeper-system
kube-system
monitoring
```

Lý do:

Policy nên chặn workload app như `demo`, nhưng không nên làm chết controller hệ thống như ArgoCD, Prometheus, ESO, Policy Controller.

## 9. Nhóm ESO

ESO là External Secrets Operator.

Kiến thức slide:

- Không commit secret thật vào Git.
- Secret lưu ở nguồn ngoài như AWS Secrets Manager.
- ESO sync secret từ nguồn ngoài vào Kubernetes Secret.
- App mount Secret qua volume thì nội dung file có thể update mà pod không restart.

Trong lab minikube, repo dùng provider `fake` để không cần AWS thật.

### `eso/secret-store.yaml`

Tạo `SecretStore` tên `fake-store`.

Tác dụng:

- Định nghĩa nguồn secret giả lập.
- Có key `/w10/demo/db-password`.
- Value ban đầu là `initial-db-password`.

### `eso/external-secret.yaml`

Tạo `ExternalSecret` tên `db-secret`.

Tác dụng:

- Nói ESO lấy key `/w10/demo/db-password`.
- Tạo Kubernetes Secret tên `db-secret`.
- Refresh mỗi `30s`.

### `eso/secret-reader.yaml`

Tạo Deployment `secret-reader`.

Tác dụng:

- Mount Secret `db-secret` vào `/etc/db/password`.
- In nội dung secret mỗi 10 giây.
- Dùng để chứng minh secret đổi mà pod không restart.

Test ESO:

```powershell
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl get pod -n demo -l app=secret-reader
```

## 10. Nhóm Supply Chain: Trivy, Cosign, Policy Controller

Kiến thức slide:

- Image không scan có thể chứa CVE.
- Image không ký thì cluster không biết image do ai build.
- CI nên scan image trước khi push.
- Image pass scan thì ký bằng Cosign.
- Admission controller verify chữ ký trước khi cho chạy.

### `.github/workflows/build-push.yml`

Workflow CI/CD.

Các việc chính:

1. Checkout repo.
2. Tính semantic version.
3. Login GHCR.
4. Build image local.
5. Trivy scan image.
6. Upload SARIF report.
7. Build và push image lên GHCR.
8. Cài Cosign.
9. Ký image bằng Cosign keyless.
10. Update `app-api/rollout.yaml` sang tag mới.
11. Commit/push tag mới.
12. Tạo Git tag.

Quyền quan trọng:

```yaml
id-token: write
```

Tác dụng:

- Cho phép Cosign keyless dùng GitHub OIDC để ký image.
- Không cần lưu private key dài hạn.

### `policies/cluster-image-policy.yaml`

Tạo ClusterImagePolicy.

Tác dụng:

- Chỉ áp dụng image:

```text
ghcr.io/nguyenha0112/w10-api**
```

- Chỉ tin chữ ký đến từ GitHub Actions workflow:

```text
build-push.yml@refs/heads/main
```

### `policies/README.md`

Ghi chú cách bật enforcement:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

Quan trọng:

Chỉ gắn label này sau khi image đã được CI ký. Nếu gắn trước, app `api` có thể bị reject khi rollout lại.

### `signing/cosign.pub`

Ghi chú về signing.

Repo hiện dùng Cosign keyless nên không có public/private key dài hạn.

Nếu giảng viên yêu cầu key-based signing, cần:

```powershell
cosign generate-key-pair
```

Sau đó:

- Commit `cosign.pub`.
- Lưu `cosign.key` và `COSIGN_PASSWORD` trong GitHub Secrets.

## 11. Nhóm runbooks

### `runbooks/secret-rotation.md`

Hướng dẫn rotate secret bằng ESO.

Tác dụng:

- Cách đổi value trong fake SecretStore.
- Cách kiểm tra Kubernetes Secret đổi.
- Cách kiểm tra pod không restart.

### `runbooks/supply-chain.md`

Hướng dẫn kiểm tra supply chain.

Tác dụng:

- Kiểm tra GitHub Actions.
- Bật namespace label cho policy-controller.
- Test image unsigned bị reject.
- Test image signed được chạy.

### `runbooks/adr-cve-exception.md`

Mẫu ADR cho ngoại lệ CVE.

Tác dụng:

- Không tắt Trivy scan toàn cục.
- Nếu CVE chưa fix được thì tạo exception có thời hạn.
- Ghi rõ CVE ID, image, lý do, mitigation và expiry.

## 12. Tuần này đã làm thêm gì so với repo ban đầu?

Repo ban đầu chủ yếu có:

```text
app-api/
app-analysis/
app-alert/
app-common/
argocd/
src/api/
```

Tức là nền tảng W9/W10 cũ: rollout, monitoring, alert, API.

Tuần này thêm các nhóm sau:

### Thêm RBAC

```text
rbac/roles.yaml
rbac/rolebindings.yaml
argocd/apps/rbac.yaml
```

Tác dụng:

- Phân quyền 3 user.
- Không còn "ai cũng admin".

### Thêm Gatekeeper Admission Policy

```text
gatekeeper/templates/
gatekeeper/constraints/
argocd/apps/gatekeeper.yaml
argocd/apps/gatekeeper-templates.yaml
argocd/apps/gatekeeper-constraints.yaml
```

Tác dụng:

- Chặn manifest xấu ở API server.
- Ép workload có tiêu chuẩn bảo mật.

### Thêm ESO

```text
eso/
argocd/apps/eso.yaml
argocd/apps/eso-config.yaml
```

Tác dụng:

- Không để secret thật trong Git.
- Secret tự sync vào Kubernetes.
- Chứng minh rotate không restart pod.

### Thêm Supply Chain Security

```text
policies/
signing/
argocd/apps/policy-controller.yaml
argocd/apps/policies.yaml
.github/workflows/build-push.yml
```

Tác dụng:

- CI scan CVE bằng Trivy.
- CI ký image bằng Cosign.
- Admission verify image đã ký.

### Thêm runbooks

```text
runbooks/
README_LAB_STEPS.md
TONG_HOP_W10_LAB.md
```

Tác dụng:

- Có tài liệu để vận hành và giải thích bài.
- Có ADR cho ngoại lệ CVE.

## 13. Cách hiểu toàn bộ flow từ đầu đến cuối

### Flow deploy app

```text
src/api/app.py
        ↓ Dockerfile
GitHub Actions build image
        ↓
Trivy scan
        ↓
Cosign sign
        ↓
Push image lên GHCR
        ↓
Update app-api/rollout.yaml
        ↓
ArgoCD sync app-api
        ↓
Argo Rollouts deploy API
        ↓
Prometheus scrape /metrics
        ↓
AnalysisTemplate kiểm tra canary
```

### Flow RBAC

```text
rbac/roles.yaml
rbac/rolebindings.yaml
        ↓
ArgoCD sync
        ↓
Kubernetes tạo Role/Binding
        ↓
kubectl auth can-i kiểm tra quyền
```

### Flow admission policy

```text
gatekeeper/templates/
        ↓
Tạo loại policy mới
        ↓
gatekeeper/constraints/
        ↓
Bật policy thật
        ↓
kubectl apply manifest
        ↓
API server gọi Gatekeeper webhook
        ↓
Pass hoặc reject
```

### Flow secret

```text
SecretStore fake-store
        ↓
ExternalSecret db-secret
        ↓
ESO tạo Kubernetes Secret db-secret
        ↓
secret-reader mount Secret bằng volume
        ↓
Secret đổi thì file trong pod đổi, pod không restart
```

### Flow image signature

```text
GitHub Actions
        ↓
Cosign keyless sign image
        ↓
Signature lưu ở registry
        ↓
Policy Controller kiểm tra khi deploy
        ↓
Image signed thì pass, unsigned thì reject
```

## 14. Checklist tự kiểm

Khi Docker/minikube chạy, dùng:

```powershell
kubectl get applications -n argocd
kubectl get pods -n demo
kubectl get rollout -n demo
kubectl get analysisrun -n demo
kubectl get pods -n monitoring
kubectl get pods -n gatekeeper-system
kubectl get pods -n external-secrets
kubectl get pods -n cosign-system
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
kubectl get constrainttemplates
kubectl get k8sdisallowlatesttag,k8srequiredcontainerlimits,k8sdisallowrootuser,k8sdisallowhostnetwork,k8smaxreplicas
```

ESO:

```powershell
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl logs -n demo -l app=secret-reader
```

Supply chain:

```powershell
kubectl get clusterimagepolicy
kubectl get pods -n cosign-system
```

ArgoCD UI:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Mở:

```text
https://localhost:8080
```

## 15. Những điểm dễ nhầm

### `can-i` không phải lệnh riêng

Sai:

```powershell
can-i create deploy -n demo --as alice
```

Đúng:

```powershell
kubectl auth can-i create deploy -n demo --as alice
```

### `AnalysisRun` không phải lúc nào cũng có

`kubectl get analysisrun -n demo` có thể trả:

```text
No resources found
```

Điều này bình thường nếu mới initial deploy. AnalysisRun thường sinh khi rollout có update/canary mới.

### Không bật signature policy quá sớm

Không label namespace `demo` trước khi image đã được ký:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

Nếu bật quá sớm, app có thể bị reject khi rollout lại.

### Không commit secret thật

Không commit:

```text
AWS access key
Gmail app password
cosign private key
database password thật
```

Chỉ commit:

```text
manifest
template
example
public key hoặc keyless policy
runbook
```

## 16. Kết luận ngắn gọn

Buổi sáng xây hàng rào trong cluster:

```text
RBAC = ai được làm gì
Gatekeeper = manifest có hợp lệ không
```

Buổi chiều bảo vệ secret và image:

```text
ESO = secret không nằm trong Git, tự sync vào cluster
Trivy = scan CVE trước khi push/deploy
Cosign = ký image để biết image đến từ pipeline tin cậy
Policy Controller = admission verify chữ ký image
```

Toàn bộ repo biến thành một platform GitOps nhỏ: push Git là thay đổi cluster, nhưng cluster vẫn có các lớp kiểm soát quyền, policy, secret và supply chain.
