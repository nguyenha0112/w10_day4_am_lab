# Hướng Dẫn Làm Lab W10 Buổi Sáng

Lab này dùng GitOps với ArgoCD để triển khai RBAC và Gatekeeper policy vào Kubernetes cluster.

Mục tiêu:

- Đổi repo GitOps sang repo của mình.
- Đổi image container sang image của mình trên GHCR.
- Tạo RBAC cho 3 user: `alice`, `bob`, `carol`.
- Cài Gatekeeper và thêm policy admission.
- Kiểm tra ArgoCD, pod API, RBAC và rollout.

## Bước 1. Vào đúng thư mục lab

Chạy:

```powershell
cd C:\Users\Admin\Desktop\W10-lab\temp
```

Tác dụng:

Đưa terminal vào đúng thư mục repo lab để các lệnh `git`, `kubectl`, `minikube` chạy trên đúng source code.

## Bước 2. Kiểm tra remote Git hiện tại

Chạy:

```powershell
git remote -v
```

Tác dụng:

Kiểm tra repo local đang nối với GitHub repo nào. Nếu vẫn trỏ về repo của người khác thì ArgoCD sẽ đọc sai source.

## Bước 3. Đổi remote sang repo của mình

Chạy:

```powershell
git remote set-url origin https://github.com/nguyenha0112/w10_day4_am_lab.git
git remote -v
```

Tác dụng:

Đổi remote `origin` sang repo GitHub của mình. Từ bước này trở đi, `git push` sẽ đẩy code lên repo:

```text
https://github.com/nguyenha0112/w10_day4_am_lab.git
```

## Bước 4. Sửa repoURL trong ArgoCD

Các file ArgoCD Application phải dùng repo của mình:

```text
https://github.com/nguyenha0112/w10_day4_am_lab.git
```

Các file cần kiểm tra:

```text
argocd/root.yaml
argocd/apps/app-common.yaml
argocd/apps/app-api.yaml
argocd/apps/app-analysis.yaml
argocd/apps/app-alert.yaml
argocd/apps/rbac.yaml
argocd/apps/gatekeeper-templates.yaml
argocd/apps/gatekeeper-constraints.yaml
```

Tác dụng:

ArgoCD hoạt động theo GitOps, nghĩa là ArgoCD đọc manifest từ GitHub. Nếu `repoURL` sai, ArgoCD sẽ kéo code từ repo cũ và không thấy bài làm của mình.

## Bước 5. Sửa image container sang image của mình

Mở file:

```text
app-api/rollout.yaml
```

Đảm bảo phần container dùng image của mình:

```yaml
containers:
- name: api
  image: ghcr.io/nguyenha0112/w10-api:0.0.1
  imagePullPolicy: IfNotPresent
```

Tác dụng:

Kubernetes sẽ pull image API từ GitHub Container Registry của mình, không dùng image của người khác.

Cấu trúc image:

```text
ghcr.io/nguyenha0112/w10-api:0.0.1
```

Ý nghĩa:

- `ghcr.io`: GitHub Container Registry.
- `nguyenha0112`: GitHub username của mình.
- `w10-api`: tên image.
- `0.0.1`: tag version.

## Bước 6. Kiểm tra workflow build image

File workflow:

```text
.github/workflows/build-push.yml
```

Trong file này có:

```yaml
IMAGE_NAME: ${{ github.repository_owner }}/w10-api
```

Tác dụng:

Khi push code lên GitHub, GitHub Actions sẽ build image và push lên GHCR theo owner của repo. Với repo của mình, image sẽ là:

```text
ghcr.io/nguyenha0112/w10-api:<tag>
```

## Bước 7. Tạo RBAC

Các file đã tạo:

```text
rbac/roles.yaml
rbac/rolebindings.yaml
argocd/apps/rbac.yaml
```

Tác dụng:

Tạo phân quyền cho 3 user:

- `alice`: developer, chỉ thao tác workload trong namespace `demo`.
- `bob`: sre, xem và thao tác pod toàn cluster.
- `carol`: viewer, chỉ được đọc tài nguyên toàn cluster.

## Bước 8. Kiểm tra RBAC

Chạy:

```powershell
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Kết quả mong đợi:

```text
yes
no
yes
no
```

Tác dụng:

Kiểm tra quyền của từng user có đúng đề bài không.

## Bước 9. Cài Gatekeeper bằng GitOps

Các file ArgoCD app:

```text
argocd/apps/gatekeeper.yaml
argocd/apps/gatekeeper-templates.yaml
argocd/apps/gatekeeper-constraints.yaml
```

Tác dụng:

- `gatekeeper.yaml`: cài Gatekeeper controller.
- `gatekeeper-templates.yaml`: cài các ConstraintTemplate.
- `gatekeeper-constraints.yaml`: cài các Constraint để enforce policy.

## Bước 10. Thêm 4 policy admission

Các constraint đã tạo:

```text
gatekeeper/constraints/disallow-latest-tag.yaml
gatekeeper/constraints/require-container-limits.yaml
gatekeeper/constraints/disallow-root-user.yaml
gatekeeper/constraints/disallow-host-network.yaml
```

Tác dụng:

Chặn các manifest xấu trước khi vào cluster:

- Cấm image tag `:latest`.
- Bắt buộc container có `resources.limits.cpu` và `resources.limits.memory`.
- Cấm chạy container bằng root user `runAsUser: 0`.
- Cấm pod dùng `hostNetwork: true`.

## Bước 11. Thêm custom policy

Các file custom policy:

```text
gatekeeper/templates/k8smaxreplicas.yaml
gatekeeper/constraints/max-replicas.yaml
```

Tác dụng:

Chặn `Deployment` hoặc `Rollout` nếu khai báo:

```yaml
replicas: 6
```

Policy chỉ cho phép tối đa:

```yaml
replicas: 5
```

## Bước 12. Commit và push code lên GitHub

Chạy:

```powershell
git status
git add .
git commit -m "feat: add rbac and gatekeeper policies"
git push -u origin main
```

Tác dụng:

Đưa toàn bộ manifest lên GitHub để ArgoCD có thể kéo về và sync vào cluster.

## Bước 13. Start minikube

Chạy:

```powershell
minikube start
minikube update-context
kubectl get nodes
```

Kết quả mong đợi:

```text
NAME       STATUS   ROLES           VERSION
minikube   Ready    control-plane   ...
```

Tác dụng:

Khởi động Kubernetes cluster local và đảm bảo `kubectl` đang trỏ đúng vào minikube.

Nếu gặp lỗi stale context, chạy lại:

```powershell
minikube update-context
```

## Bước 14. Apply root app của ArgoCD

Chạy:

```powershell
kubectl apply -f argocd/root.yaml
```

Tác dụng:

Tạo ArgoCD root application. Root app sẽ sinh ra các child app như:

- `common`
- `analysis`
- `api`
- `rbac`
- `gatekeeper`
- `gatekeeper-templates`
- `gatekeeper-constraints`
- `kube-prometheus-stack`
- `argo-rollouts`

## Bước 15. Kiểm tra trạng thái ArgoCD Applications

Chạy:

```powershell
kubectl get applications -n argocd
```

Tác dụng:

Kiểm tra app nào đã `Synced`, app nào còn `OutOfSync`, app nào đang `Progressing` hoặc `Missing`.

Trạng thái tốt:

```text
Synced   Healthy
```

Nếu app đang `Progressing`, thường là pod đang chạy hoặc đang pull image.

Nếu app `OutOfSync`, có thể bấm `SYNC` trong UI ArgoCD hoặc đợi auto-sync.

## Bước 16. Kiểm tra pod API

Chạy:

```powershell
kubectl get pods -n demo -l app=api
```

Tác dụng:

Xem pod API đã chạy chưa.

Kết quả tốt:

```text
NAME                  READY   STATUS    RESTARTS
api-xxxxx             1/1     Running   0
```

Nếu thấy `ContainerCreating`, pod đang pull image.

Nếu thấy `ImagePullBackOff`, image sai tên, sai tag, private, hoặc chưa được push lên GHCR.

## Bước 17. Xem lỗi chi tiết của pod API

Chạy:

```powershell
kubectl describe pod -n demo <ten-pod-api>
```

Ví dụ:

```powershell
kubectl describe pod -n demo api-55f66bfc4-76vhs
```

Tác dụng:

Xem event cuối pod, thường sẽ biết lỗi là:

- Đang pull image.
- Pull image thất bại.
- Probe fail.
- Admission policy reject.
- Thiếu resource.

## Bước 18. Kiểm tra rollout API

Chạy:

```powershell
kubectl get rollout api -n demo
kubectl describe rollout api -n demo
```

Tác dụng:

Kiểm tra Argo Rollouts đã rollout API chưa, đang ở bước canary nào, có bị pause hay fail không.

## Bước 19. Kiểm tra AnalysisRun

Chạy:

```powershell
kubectl get analysisrun -n demo
```

Tác dụng:

Xem các lần analysis do Argo Rollouts tạo ra.

Lưu ý quan trọng:

Lần deploy đầu tiên thường chưa có `AnalysisRun`. Argo Rollouts coi đó là `Initial deploy`, nên có thể trả:

```text
No resources found in demo namespace.
```

Muốn tạo `AnalysisRun`, cần có lần update tiếp theo, ví dụ đổi version hoặc image tag trong `app-api/rollout.yaml`, rồi commit và push.

Ví dụ đổi:

```yaml
- name: VERSION
  value: "v0.0.2"
```

Sau đó:

```powershell
git add app-api/rollout.yaml
git commit -m "test: trigger rollout analysis"
git push
```

Đợi ArgoCD sync xong rồi chạy lại:

```powershell
kubectl get analysisrun -n demo
```

## Bước 20. Kiểm tra Service và ServiceMonitor

Chạy:

```powershell
kubectl get svc -n demo
kubectl get servicemonitor -n demo
```

Tác dụng:

Kiểm tra API đã có Service để expose pod và Prometheus đã có ServiceMonitor để scrape metrics.

## Bước 21. Kiểm tra Gatekeeper

Chạy:

```powershell
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get k8smaxreplicas
kubectl get k8sdisallowlatesttag
kubectl get k8srequiredcontainerlimits
kubectl get k8sdisallowrootuser
kubectl get k8sdisallowhostnetwork
```

Tác dụng:

Kiểm tra Gatekeeper controller, templates và constraints đã được cài chưa.

## Bước 22. Mở ArgoCD UI

Chạy:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Mở trình duyệt:

```text
https://localhost:8080
```

Tác dụng:

Mở giao diện ArgoCD để xem các application, bấm sync, refresh và xem lỗi trực quan.

## Bước 23. Lấy password ArgoCD nếu cần

Chạy:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Tài khoản:

```text
admin
```

Tác dụng:

Lấy mật khẩu đăng nhập ArgoCD UI.

## Bước 24. Các lỗi thường gặp

### Pod API bị ContainerCreating lâu

Nguyên nhân thường gặp:

- Minikube đang pull image từ GHCR.
- Mạng chậm.
- Docker Desktop đang chậm.

Kiểm tra:

```powershell
kubectl describe pod -n demo <ten-pod-api>
```

Nếu event là:

```text
Pulling image "ghcr.io/nguyenha0112/w10-api:0.0.1"
```

thì chờ image pull xong.

### Pod API bị ImagePullBackOff

Nguyên nhân thường gặp:

- Sai tên image.
- Sai tag.
- Image chưa được push lên GHCR.
- Package GHCR đang private.

Kiểm tra image:

```powershell
docker manifest inspect ghcr.io/nguyenha0112/w10-api:0.0.1
```

### `kubectl get analysisrun -n demo` không có gì

Nếu kết quả là:

```text
No resources found in demo namespace.
```

thì chưa chắc là lỗi. AnalysisRun thường chỉ xuất hiện sau lần rollout update, không nhất thiết có ở lần deploy đầu.

### ArgoCD app OutOfSync

Có thể bấm `SYNC` trong ArgoCD UI hoặc chạy:

```powershell
kubectl annotate application api -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

Tác dụng:

Bắt ArgoCD refresh lại trạng thái từ Git và cluster.

## Bước 25. Các lệnh kiểm tra nhanh

```powershell
kubectl get applications -n argocd
kubectl get pods -n demo
kubectl get rollout -n demo
kubectl get analysisrun -n demo
kubectl get pods -n gatekeeper-system
kubectl get pods -n monitoring
```

Tác dụng:

Đây là nhóm lệnh dùng để kiểm tra nhanh toàn bộ lab: ArgoCD, API, rollout, analysis, Gatekeeper và monitoring.

## Bước 26. Lab buổi chiều: ESO

Các file đã thêm:

```text
eso/secret-store.yaml
eso/external-secret.yaml
eso/secret-reader.yaml
argocd/apps/eso.yaml
argocd/apps/eso-config.yaml
```

Tác dụng:

- `eso.yaml`: cài External Secrets Operator.
- `eso-config.yaml`: sync cấu hình secret trong thư mục `eso/`.
- `secret-store.yaml`: dùng provider `fake` để mô phỏng nguồn secret bên ngoài khi chạy minikube.
- `external-secret.yaml`: tạo Kubernetes Secret tên `db-secret`.
- `secret-reader.yaml`: pod mount secret qua volume để chứng minh secret đổi mà pod không restart.

Kiểm tra:

```powershell
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl get pod -n demo -l app=secret-reader
```

## Bước 27. Lab buổi chiều: Trivy + Cosign

Các file đã thêm:

```text
.github/workflows/build-push.yml
argocd/apps/policy-controller.yaml
argocd/apps/policies.yaml
policies/cluster-image-policy.yaml
policies/README.md
signing/cosign.pub
```

Tác dụng:

- Workflow build image, scan bằng Trivy, push lên GHCR, rồi ký image bằng Cosign keyless.
- `policy-controller.yaml`: cài Sigstore Policy Controller.
- `policies.yaml`: sync policy verify chữ ký.
- `cluster-image-policy.yaml`: chỉ tin image `ghcr.io/nguyenha0112/w10-api` được ký bởi GitHub Actions của repo này.

Lưu ý:

Policy Controller chỉ enforce namespace có label:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

Chỉ gắn label này sau khi GitHub Actions đã build và ký image thành công.

## Bước 28. Runbook và ADR

Các file đã thêm:

```text
runbooks/secret-rotation.md
runbooks/supply-chain.md
runbooks/adr-cve-exception.md
```

Tác dụng:

- `secret-rotation.md`: hướng dẫn rotate secret bằng ESO.
- `supply-chain.md`: hướng dẫn kiểm tra Trivy, Cosign và admission verify.
- `adr-cve-exception.md`: mẫu ngoại lệ CVE có thời hạn, không tắt scan toàn cục.
