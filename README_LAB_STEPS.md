# W10 Morning Lab - RBAC + Gatekeeper Steps

## 1. Kiem tra repo va remote

```powershell
cd C:\Users\Admin\Desktop\W10-lab\temp
git remote -v
```

Doi remote sang repo moi:

```powershell
git remote set-url origin https://github.com/nguyenha0112/w10_day4_am_lab.git
git remote -v
```

## 2. Sua GitOps repoURL

Tat ca ArgoCD Application dung repo moi:

```text
https://github.com/nguyenha0112/w10_day4_am_lab.git
```

Nhung file can kiem tra:

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

## 3. Sua image cua minh

Trong `app-api/rollout.yaml`, image phai tro ve GHCR cua user minh:

```yaml
containers:
- name: api
  image: ghcr.io/nguyenha0112/w10-api:0.0.1
  imagePullPolicy: IfNotPresent
```

Workflow `.github/workflows/build-push.yml` build image theo:

```yaml
IMAGE_NAME: ${{ github.repository_owner }}/w10-api
```

Vi vay image dung la:

```text
ghcr.io/nguyenha0112/w10-api:<tag>
```

## 4. RBAC

Da tao:

```text
rbac/roles.yaml
rbac/rolebindings.yaml
argocd/apps/rbac.yaml
```

Kiem tra quyen:

```powershell
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Ket qua mong doi:

```text
yes
no
yes
no
```

## 5. Gatekeeper

Da tao app cai Gatekeeper:

```text
argocd/apps/gatekeeper.yaml
argocd/apps/gatekeeper-templates.yaml
argocd/apps/gatekeeper-constraints.yaml
```

Da tao 4 constraint bat buoc:

```text
gatekeeper/constraints/disallow-latest-tag.yaml
gatekeeper/constraints/require-container-limits.yaml
gatekeeper/constraints/disallow-root-user.yaml
gatekeeper/constraints/disallow-host-network.yaml
```

Da tao 1 custom policy:

```text
gatekeeper/templates/k8smaxreplicas.yaml
gatekeeper/constraints/max-replicas.yaml
```

Custom policy nay chan `Deployment` hoac `Rollout` neu `replicas > 5`.

## 6. Commit va push

```powershell
git status
git add .
git commit -m "feat: add rbac and gatekeeper policies"
git push -u origin main
```

## 7. Start minikube va sua context neu can

```powershell
minikube start
minikube update-context
kubectl get nodes
```

Node phai o trang thai `Ready`.

## 8. Apply root app

```powershell
kubectl apply -f argocd/root.yaml
```

Kiem tra ArgoCD Applications:

```powershell
kubectl get applications -n argocd
```

## 9. Kiem tra pod API

```powershell
kubectl get pods -n demo -l app=api
kubectl get all -n demo -l app=api
```

## 10. Mo ArgoCD UI

Chay port-forward:

```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Mo trinh duyet:

```text
https://localhost:8080
```

Lay password admin neu can:

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

## 11. Luu y loi hay gap

Neu `kubectl` bao stale context:

```powershell
minikube update-context
```

Neu Gatekeeper hoac Prometheus chua len:

```powershell
kubectl get pods -n gatekeeper-system
kubectl get pods -n monitoring
kubectl get applications -n argocd
```

Neu bi ket `ContainerCreating` o buoc pull image, thu lai sau khi Docker/network on dinh.
