# On tap W10: kien thuc slide va vi tri trong project

File nay tom tat kien thuc trong 2 slide W10:

- Buoi sang: RBAC + Admission Policy.
- Buoi chieu: Secrets + Supply Chain + Platform Integration.

Project dang duoc phan tich nam trong thu muc `temp/`.

## 1. Buc tranh tong the

W10 tap trung vao cluster-level enforcement: khong chi deploy duoc app, ma phai co guardrail de chan loi va rui ro truoc khi vao cluster.

Trong project cua ban, cac nhom kien thuc nam o day:

| Kien thuc | Thu muc/file chinh |
|---|---|
| GitOps / App of Apps | `argocd/root.yaml`, `argocd/apps/*.yaml` |
| RBAC | `rbac/roles.yaml`, `rbac/rolebindings.yaml`, `argocd/apps/rbac.yaml` |
| Admission Policy / Gatekeeper | `gatekeeper/templates/*.yaml`, `gatekeeper/constraints/*.yaml`, `argocd/apps/gatekeeper*.yaml` |
| App canary / platform cu | `app-api/`, `app-analysis/`, `app-alert/`, `app-common/` |
| Secrets rotation / ESO | `eso/`, `argocd/apps/eso.yaml`, `argocd/apps/eso-config.yaml` |
| Supply chain security | `.github/workflows/build-push.yml`, `policies/cluster-image-policy.yaml`, `argocd/apps/policy-controller.yaml`, `argocd/apps/policies.yaml` |
| Evidence / runbook | `runbooks/*.md`, `policies/README.md`, `README_LAB_STEPS.md` |
| Multi-tenant payments | `tenants/payments/`, `apps/payments/`, `argocd/apps/payments*.yaml` |

## 2. GitOps va App of Apps

### Kien thuc can nho

GitOps nghia la cluster lay trang thai mong muon tu Git. Ban khong nen apply tung manifest bang tay, ma commit len Git, de ArgoCD sync vao cluster.

App of Apps la pattern trong ArgoCD: mot root Application quan ly nhieu child Application.

### Trong project cua ban

File `argocd/root.yaml` la root app:

- `source.repoURL`: tro ve repo `https://github.com/nguyenha0112/w10_day4_am_lab.git`.
- `source.path`: `argocd/apps`.
- `syncPolicy.automated.prune: true`: xoa resource khong con trong Git.
- `syncPolicy.automated.selfHeal: true`: cluster bi sua tay thi ArgoCD tu dua ve dung Git.

Thu muc `argocd/apps/` chua cac child app:

- `app-common.yaml`: tao namespace/demo resource chung.
- `app-api.yaml`: deploy API Rollout.
- `app-analysis.yaml`: deploy AnalysisTemplate.
- `app-alert.yaml`: deploy PrometheusRule.
- `rbac.yaml`: sync RBAC.
- `gatekeeper.yaml`: cai Gatekeeper controller.
- `gatekeeper-templates.yaml`: sync ConstraintTemplate.
- `gatekeeper-constraints.yaml`: sync Constraint.
- `eso.yaml`: cai External Secrets Operator.
- `eso-config.yaml`: sync SecretStore, ExternalSecret, secret-reader.
- `policy-controller.yaml`: cai Sigstore Policy Controller.
- `policies.yaml`: sync policy verify image signature.

### Diem quan trong

Project co dung `sync-wave` de sap xep thu tu:

- Namespace/common va controller can len truoc.
- Template phai len truoc Constraint.
- ESO controller phai len truoc ExternalSecret.
- Policy controller phai len truoc ClusterImagePolicy.

Neu sai thu tu, ArgoCD co the bao loi CRD chua ton tai.

## 3. RBAC: ai duoc lam gi

### Kien thuc can nho

RBAC tra loi cau hoi: user/service account nao duoc lam hanh dong nao tren resource nao.

Thanh phan chinh:

- `Role`: quyen trong mot namespace.
- `ClusterRole`: quyen toan cluster hoac quyen reusable.
- `RoleBinding`: gan Role/ClusterRole trong mot namespace.
- `ClusterRoleBinding`: gan ClusterRole tren toan cluster.

Nguyen tac quan trong la least privilege: chi cap dung quyen can thiet.

### Trong project cua ban

File `rbac/roles.yaml` tao 3 vai tro:

1. `Role developer` trong namespace `demo`

- Cho phep thao tac workload trong `demo`.
- Co quyen voi `pods`, `pods/log`, `services`, `configmaps`, `secrets`.
- Co quyen voi `deployments`, `replicasets`.
- Co quyen voi Argo Rollouts `rollouts`.

Luu y: slide yeu cau developer thao tac workload trong `demo`. Project cua ban hien dang cho developer CRUD ca `secrets`; neu cham theo least-privilege nghiem ngat, day la diem nen can nhac giam quyen.

2. `ClusterRole sre`

- Xem/toan tac pod tren nhieu namespace.
- Co `pods/exec`, `pods/log`.
- Co the patch/update deployments va rollouts.
- Chi doc `namespaces`, `nodes`, `services`, `events`.

3. `ClusterRole viewer`

- `apiGroups: ["*"]`, `resources: ["*"]`, verbs chi `get/list/watch`.
- Day la quyen doc toan cluster, khong co create/update/delete.

File `rbac/rolebindings.yaml` gan user:

- `alice` -> `developer` bang `RoleBinding` trong namespace `demo`.
- `bob` -> `sre` bang `ClusterRoleBinding`.
- `carol` -> `viewer` bang `ClusterRoleBinding`.

File `argocd/apps/rbac.yaml` dua thu muc `rbac/` vao cluster qua ArgoCD.

### Lenh tu kiem tra

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

## 4. Gatekeeper / Admission Policy

### Kien thuc can nho

Admission Policy tra loi cau hoi: manifest co hop le de vao cluster khong.

OPA Gatekeeper co 2 lop:

- `ConstraintTemplate`: dinh nghia logic bang Rego va tao ra loai constraint moi.
- `Constraint`: ap dung template do vao resource nao, voi tham so nao, enforcement la `deny` hay `warn`.

Flow:

1. User/ArgoCD gui manifest vao API Server.
2. Admission webhook cua Gatekeeper kiem tra.
3. Neu vi pham constraint `deny`, API Server reject.
4. Resource khong duoc tao trong cluster.

### Trong project cua ban

Thu muc `gatekeeper/templates/` co 5 ConstraintTemplate:

- `k8sdisallowlatesttag.yaml`: cam image `:latest` va image khong co tag ro rang.
- `k8srequiredcontainerlimits.yaml`: bat buoc container co `resources.limits.cpu` va `resources.limits.memory`.
- `k8sdisallowrootuser.yaml`: cam `runAsUser: 0` o pod hoac container.
- `k8sdisallowhostnetwork.yaml`: cam `hostNetwork: true`.
- `k8smaxreplicas.yaml`: custom policy, gioi han replicas.

Thu muc `gatekeeper/constraints/` ap dung cac template:

- `disallow-latest-tag.yaml`: enforce tren Pod.
- `require-container-limits.yaml`: enforce tren Pod.
- `disallow-root-user.yaml`: enforce tren Pod.
- `disallow-host-network.yaml`: enforce tren Pod.
- `max-replicas.yaml`: enforce tren Deployment va Rollout, `parameters.max: 5`.

Tat ca constraint deu dung:

```yaml
enforcementAction: deny
```

Nghia la vi pham se bi chan that, khong chi canh bao.

### Namespace duoc exclude

Constraint cua ban exclude cac namespace he thong:

- `argocd`
- `argo-rollouts`
- `cosign-system`
- `external-secrets`
- `gatekeeper-system`
- `kube-system`
- `monitoring`

Day la dung tinh than slide: tranh bat guardrail len controller/system pod roi lam sap platform.

### App API cua ban co dap ung policy khong?

File `app-api/rollout.yaml`:

- Image da pin tag: `ghcr.io/nguyenha0112/w10-api:0.0.1`, khong phai `:latest`.
- Co `resources.limits.cpu: 200m`.
- Co `resources.limits.memory: 128Mi`.
- `replicas: 4`, nho hon gioi han custom `max: 5`.
- Khong set `hostNetwork: true`.
- Khong set `runAsUser: 0`.

Vi vay app API hien tai phu hop voi cac guardrail Gatekeeper trong project.

### Lenh tu kiem tra

```powershell
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get k8sdisallowlatesttag
kubectl get k8srequiredcontainerlimits
kubectl get k8sdisallowrootuser
kubectl get k8sdisallowhostnetwork
kubectl get k8smaxreplicas
```

## 5. Argo Rollouts va monitoring cua platform cu

### Kien thuc can nho

Day la nen tang tu cac buoi truoc nhung van lien quan W10, vi guardrail phai khong lam sap platform dang chay.

Argo Rollouts giup progressive delivery:

- Chia traffic canary theo buoc.
- Chay analysis dua vao metric Prometheus.
- Co the rollback neu metric xau.

### Trong project cua ban

File `app-api/rollout.yaml`:

- `kind: Rollout`.
- `replicas: 4`.
- Canary steps: `10% -> pause 2m -> 50% -> pause 2m -> 100%`.
- Dung `AnalysisTemplate` ten `success-rate`.

File `app-analysis/analysis-template.yaml`:

- Dinh nghia metric dung cho rollout analysis.
- Thuong query Prometheus de do success rate.

File `app-alert/prometheus-rules.yaml`:

- Dinh nghia alert cho SLO/runtime issue.

File `app-api/servicemonitor.yaml`:

- Cho Prometheus scrape metric cua API.

## 6. Secrets Rotation voi External Secrets Operator

### Kien thuc can nho

Van de trong slide:

- Secret hard-code trong Git la rui ro lon.
- Kubernetes Secret chi la base64, khong phai encryption.
- Rotate secret bang cach sua Secret thu cong thuong can restart pod neu app doc env var.

Giai phap:

- Luu secret that o external secret manager, vi du AWS Secrets Manager.
- ESO sync secret tu external store vao Kubernetes Secret.
- Pod mount Secret qua volume de kubelet cap nhat file khi Secret thay doi.
- App doc file moi thi khong can restart pod.

### Trong project cua ban

File `argocd/apps/eso.yaml`:

- Cai chart `external-secrets`.
- `installCRDs: true`.
- Namespace dich: `external-secrets`.

File `argocd/apps/eso-config.yaml`:

- Sync thu muc `eso/` vao namespace `demo`.

File `eso/secret-store.yaml`:

- Tao `SecretStore` ten `aws-store`.
- Dung provider `aws` de doc AWS Secrets Manager.
- Region hien tai: `ap-southeast-1`.
- AWS secret key: `/w10/demo/db-password`.
- AWS credential khong commit vao Git, tao bang Kubernetes Secret `aws-credentials`.

File `eso/external-secret.yaml`:

- Tao `ExternalSecret` ten `db-secret`.
- `refreshInterval: 30s`, tot hon target slide `< 60s`.
- Doc tu `aws-store`.
- Tao Kubernetes Secret ten `db-secret`.
- Map remote key `/w10/demo/db-password` vao key `password`.

File `eso/secret-reader.yaml`:

- Deployment `secret-reader`.
- Mount Secret `db-secret` vao `/etc/db`.
- Container lap lai `cat /etc/db/password` moi 10 giay.
- Co `resources.limits`, nen khong bi Gatekeeper chan.

### Diem can hieu ro

Trong slide, production case la AWS Secrets Manager + IRSA. Trong project cua ban, ESO dang dung AWS Secrets Manager voi credential luu trong Kubernetes Secret `aws-credentials`.

```text
AWS Secrets Manager -> ESO controller -> Kubernetes Secret -> Pod mount volume
```

Neu dung EKS that, co the thay credential secret bang IRSA de khong can access key dai han.

### Lenh tu kiem tra

```powershell
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl get pod -n demo -l app=secret-reader
kubectl logs -n demo deploy/secret-reader
```

## 7. Supply Chain Security: Trivy, Cosign, Policy Controller

### Kien thuc can nho

Supply chain security tra loi cac cau hoi:

- Image co loi CVE nghiem trong khong?
- Image co dung nguon build tin cay khong?
- Cluster co chan image chua ky/chua tin cay khong?

Ba lop trong slide:

1. Trivy scan image trong CI.
2. Cosign ky image sau khi build/pass scan.
3. Admission verify chu ky truoc khi cho image chay.

### Trong project cua ban: CI build + scan + sign

File `.github/workflows/build-push.yml`:

- Trigger khi push vao `main` va thay doi `src/api/**`, `app-api/rollout.yaml`, workflow.
- Build image tu `src/api`.
- Image name: `ghcr.io/${{ github.repository_owner }}/w10-api`.
- Tinh semantic version bang `paulhatch/semantic-version`.
- Login GHCR bang `GITHUB_TOKEN`.
- Trivy scan image voi:
  - `severity: HIGH,CRITICAL`
  - `ignore-unfixed: true`
  - `exit-code: "1"`
- Neu scan fail thi pipeline do, image khong nen release.
- Push image len GHCR.
- Cai Cosign.
- Ky image bang Cosign key pair, private key lay tu GitHub Secrets:

```bash
cosign sign --key env://COSIGN_PRIVATE_KEY "$tag"
```

- Cap nhat `app-api/rollout.yaml` sang version moi.
- Commit version update va tao git tag.

### Trong project cua ban: admission verify

File `argocd/apps/policy-controller.yaml`:

- Cai Sigstore Policy Controller bang Helm chart.
- Namespace `cosign-system`.

File `argocd/apps/policies.yaml`:

- Sync thu muc `policies/`.

File `policies/cluster-image-policy.yaml`:

- Tao `ClusterImagePolicy` ten `require-signed-w10-api`.
- Chi ap dung image match:

```text
ghcr.io/nguyenha0112/w10-api**
```

- Tin image duoc ky bang private key tuong ung voi public key:

```text
signing/cosign.pub
```

Private key khong commit vao Git. Private key can duoc luu trong GitHub Secrets `COSIGN_PRIVATE_KEY`, va password trong `COSIGN_PASSWORD`.

### Diem can nho

Policy Controller chi enforce namespace co label:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

Chi nen gan label sau khi image da duoc build va ky thanh cong. Neu gan som, rollout hien tai co the bi reject.

### Lenh tu kiem tra

```powershell
kubectl get pods -n cosign-system
kubectl get clusterimagepolicy
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
kubectl run unsigned-test -n demo --image=nginx:1.27 --dry-run=server
```

Ket qua mong doi: image khong dung policy/khong co chu ky tin cay bi admission reject.

## 8. Runbook va ADR

### Kien thuc can nho

Production-ready khong chi co manifest. Can co tai lieu van hanh:

- Khi rotate secret thi lam gi?
- Khi CI fail vi CVE thi xu ly sao?
- Khi can exception thi ai duyet, het han luc nao?

### Trong project cua ban

File `runbooks/secret-rotation.md`:

- Huong dan rotate secret trong lab local.
- Sua `eso/secret-store.yaml` sang value/version moi.
- Sua `eso/external-secret.yaml` sang version moi.
- Commit/push de ArgoCD sync.
- Kiem tra Secret doi va pod khong restart.

File `runbooks/supply-chain.md`:

- Huong dan kiem tra GitHub Actions.
- Kiem tra Trivy, Cosign, admission verify.
- Huong dan bat label `policy.sigstore.dev/include=true`.

File `runbooks/adr-cve-exception.md`:

- Mau exception khi CVE HIGH/CRITICAL chua co ban va.
- Khong tat Trivy toan cuc.
- Exception phai co CVE ID, image/tag, ly do, mitigation, ngay het han.
- Mac dinh het han sau 7 ngay.

File `policies/README.md`:

- Nhac lai namespace label de policy-controller enforce.
- Nhac chi gan label sau khi image da signed.

## 9. Checklist doi chieu slide voi project

### Buoi sang: RBAC + Gatekeeper

| Yeu cau slide | Project cua ban | Trang thai |
|---|---|---|
| 3 role: developer, sre, viewer | `rbac/roles.yaml` | Co |
| Binding cho alice, bob, carol | `rbac/rolebindings.yaml` | Co |
| ArgoCD app cho RBAC | `argocd/apps/rbac.yaml` | Co |
| Cai Gatekeeper qua GitOps | `argocd/apps/gatekeeper.yaml` | Co |
| ConstraintTemplate | `gatekeeper/templates/*.yaml` | Co |
| 4 constraint bat buoc | `gatekeeper/constraints/disallow-*`, `require-container-limits.yaml` | Co |
| Custom policy | `k8smaxreplicas`, `max-replicas.yaml` | Co |
| App khong bi guardrail chan | `app-api/rollout.yaml` hop le | Co |

### Buoi chieu: ESO + Supply Chain

| Yeu cau slide | Project cua ban | Trang thai |
|---|---|---|
| ESO controller | `argocd/apps/eso.yaml` | Co |
| SecretStore | `eso/secret-store.yaml` | Co, dung AWS Secrets Manager |
| ExternalSecret | `eso/external-secret.yaml` | Co |
| Rotate < 60s | `refreshInterval: 30s` | Co |
| Pod mount Secret qua volume | `eso/secret-reader.yaml` | Co |
| CI Trivy HIGH/CRITICAL | `.github/workflows/build-push.yml` | Co |
| Cosign sign image | `.github/workflows/build-push.yml` | Co, key pair |
| Admission verify signed image | `policy-controller.yaml`, `cluster-image-policy.yaml` | Co |
| Runbook/ADR | `runbooks/*.md` | Co |

### Challenge: payments tenant

| Yeu cau slide | Project cua ban | Trang thai |
|---|---|---|
| Namespace `payments` | `tenants/payments/namespace.yaml` | Co |
| RBAC least-privilege | `tenants/payments/rbac.yaml` | Co |
| Khong doc secret / sua rolebinding | `tenants/payments/rbac.yaml` khong cap quyen nay | Co |
| ResourceQuota + LimitRange | `tenants/payments/quota-limitrange.yaml` | Co |
| NetworkPolicy co lap | `tenants/payments/networkpolicies.yaml` | Co |
| App team B qua GitOps | `apps/payments/`, `argocd/apps/payments-app.yaml` | Co |
| Guardrail cu tu ap dung | Gatekeeper ap dung moi ns; Sigstore ap dung nho label namespace | Co |

## 10. Cac diem nen chu y neu bi hoi van dap

### Vi sao RBAC khong du?

RBAC chi noi ai duoc lam gi. Neu user co quyen create Pod, RBAC khong tu biet Pod do co `:latest`, thieu limits, hay hostNetwork. Gatekeeper bo sung lop kiem tra noi dung manifest.

### Vi sao Gatekeeper can exclude namespace he thong?

Neu policy qua chat ap vao controller nhu ArgoCD, Gatekeeper, monitoring, external-secrets, ban co the tu chan chinh he thong van hanh. Project da exclude cac namespace nay trong moi constraint.

### Vi sao ESO tot hon commit Secret vao Git?

Secret that khong nam trong Git history. Git chi chua mapping/cau hinh sync. Khi rotate, ESO cap nhat Kubernetes Secret tu external store.

### Vi sao mount secret bang volume hay hon env var cho rotate?

Env var chi duoc inject khi container start. Volume secret co the duoc kubelet update file sau khi Secret thay doi. App doc file moi co the thay password moi ma khong restart.

### Vi sao can Trivy va Cosign ca hai?

Trivy tra loi "image co lo hong nghiem trong khong". Cosign tra loi "image nay co phai do pipeline tin cay build/ky khong". Mot image it CVE nhung khong ro nguon goc van la rui ro.

### Vi sao admission verify dat trong cluster?

CI co the ky image, nhung neu ai do sua manifest tro sang image khac, cluster van can lop admission de reject truoc khi pod chay.

## 11. Cac lenh on tap nhanh

```powershell
# ArgoCD apps
kubectl get applications -n argocd

# RBAC
kubectl auth can-i create deploy -n demo --as alice
kubectl auth can-i create deploy -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol

# Gatekeeper
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get k8sdisallowlatesttag
kubectl get k8srequiredcontainerlimits
kubectl get k8sdisallowrootuser
kubectl get k8sdisallowhostnetwork
kubectl get k8smaxreplicas

# API / Rollout
kubectl get rollout api -n demo
kubectl get pods -n demo -l app=api
kubectl get analysisrun -n demo

# ESO
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret db-secret -n demo
kubectl logs -n demo deploy/secret-reader

# Supply chain
kubectl get pods -n cosign-system
kubectl get clusterimagepolicy
kubectl run unsigned-test -n demo --image=nginx:1.27 --dry-run=server
```

## 12. Ket luan ngan gon

Project cua ban da gan khop kha day du voi kien thuc cua 2 slide W10:

- Buoi sang: co RBAC theo vai tro va Gatekeeper admission policy.
- Buoi chieu: co ESO secret sync/rotation, CI scan ky image, va admission verify chu ky.
- Nen tang GitOps da dung App of Apps de quan ly toan bo controller, policy va app.

Neu can chon 1 cau de giai thich project: "Day la mot platform Kubernetes duoc quan ly bang GitOps, trong do RBAC kiem soat ai duoc lam gi, Gatekeeper kiem soat manifest co hop le khong, ESO quan ly secret khong commit vao Git, va Sigstore/Trivy bao ve supply chain tu CI den admission."
