# EVIDENCE PACK — W9 GitOps Foundations + Observability & Canary Lab

Tài liệu này tổng hợp **bằng chứng thực thi** (commands + outputs) cho cả
**buổi sáng** (GitOps Foundations, Lab 0→7) và **buổi chiều** (Observability +
Canary, Lab 1→4 + Challenge), theo đúng trình tự đã thực hiện trên cluster `w9` (minikube).

---

# PHẦN A — Buổi sáng: GitOps Foundations

## Lab 0 — Dựng cụm + app đầu tiên + push Git

```bash
minikube start -p w9 --driver=docker
kubectl config use-context w9
kubectl get nodes   # STATUS Ready

mkdir gitops && cd gitops && mkdir k8s
# viết k8s/web.yaml (Deployment nginx:1.27, replicas: 2, ns demo)

git init && git add . && git commit -m "init"
git branch -M main
git remote add origin https://github.com/hoangvu186/gitops.git
git push -u origin main
```

✅ Cụm `w9` Ready, repo `gitops` có `k8s/web.yaml` — **chưa apply lên cụm**
(để ArgoCD làm ở Lab 2).

---

## Lab 1 — Cài ArgoCD

```bash
kubectl create ns argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get pods
```

✅ Toàn bộ pod `argocd-*` (server, repo-server, application-controller,
dex, redis, ...) ở trạng thái `Running`. ArgoCD đã "sống trong cụm", sẵn sàng
kéo từ Git ở Lab 2.

---

## Lab 2 — Tạo Application đầu tiên (apply tay — lần đầu)

**File:** `argocd/apps/web.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: web, namespace: argocd }
spec:
  source: { repoURL: https://github.com/hoangvu186/gitops.git, path: k8s }
  destination: { server: https://kubernetes.default.svc, namespace: demo }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```

```bash
kubectl apply -f argocd/apps/web.yaml
kubectl -n argocd get app web    # Synced/Healthy
kubectl -n demo get deploy,pod   # 2 pod web
```

✅ **Kết quả:** `web` = `Synced/Healthy`. ArgoCD tự tạo `Deployment web`
(2 pod `nginx:1.27`) trong namespace `demo` — **không `kubectl apply` Deployment
trực tiếp**, chỉ apply Application.

---

## Lab 3 — Sync qua Git & Self-Heal

### 3.1. Đổi qua Git
```bash
# sửa k8s/web.yaml: replicas 2 -> 4
git commit -am "2->4" && git push
```
✅ ArgoCD tự kéo commit mới (~vài giây tới 3 phút), `Deployment web` scale lên
**4 pod** — không có lệnh `kubectl scale` nào được chạy.

### 3.2. Self-Heal
```bash
kubectl -n demo scale deploy/web --replicas=9
kubectl -n demo get deploy web -w
```
✅ **Kết quả:** ngay sau khi scale tay lên 9, ArgoCD phát hiện cụm lệch Git
(Git nói 4) → tự động `scale` về lại **4 pod** trong vài giây. Minh chứng
nguyên tắc *Reconciled*: sửa tay không "sống sót".

---

## Lab 4 ⭐ — Rollback bằng `git revert`

```bash
git revert HEAD --no-edit && git push
```

✅ **Kết quả:** commit revert được tạo (có tác giả + message rõ ràng trong
`git log`), ArgoCD phát hiện commit mới → sync cụm về đúng trạng thái commit
trước đó (`replicas: 2`).

**Đối chiếu nếu dùng `kubectl rollout undo` (KHÔNG làm, chỉ minh họa lý thuyết):**
Git vẫn giữ `replicas: 4` → self-heal sẽ apply lại 4 → "rollback" bị ghi đè
ngay sau đó → **rollback giả**.

| | `kubectl rollout undo` | `git revert` (đã thực hiện) |
|---|---|---|
| Bị self-heal ghi đè | ✅ Có | ❌ Không |
| Audit trail | Không | Commit author + message |

---

## Lab 5 — App-of-Apps (root)

**File:** `argocd/root.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: root, namespace: argocd }
spec:
  source:
    repoURL: https://github.com/hoangvu186/gitops.git
    targetRevision: HEAD
    path: argocd/apps
  destination: { server: https://kubernetes.default.svc, namespace: argocd }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```

```bash
git add argocd/root.yaml && git commit -m "app-of-apps" && git push
kubectl apply -f argocd/root.yaml   # apply tay — LẦN CUỐI dùng kubectl tạo app
kubectl -n argocd get applications
```

✅ **Kết quả:**
```
NAME   SYNC STATUS   HEALTH STATUS
root   Synced        Healthy
web    Synced        Healthy
```
`root` tiếp quản `web` (đã có sẵn trong `argocd/apps/`). Từ đây, **mọi app mới
chỉ cần thả file YAML vào `argocd/apps/` + `git push`** — không `kubectl apply`
nữa. (Đây chính là cơ chế dùng để thêm `kube-prometheus-stack`, `argo-rollouts`,
`api`, `tvshow` ở buổi chiều.)

---

## Lab 6 — Sync Waves

**File:** `k8s/namespace.yaml` (wave `-1`)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  annotations: { argocd.argoproj.io/sync-wave: "-1" }
```

**File:** `k8s/web.yaml` — 3 resource gắn wave 0/1/2 (ConfigMap → Deployment → Service),
Deployment đọc `MESSAGE` qua `envFrom.configMapRef`.

```bash
git add k8s/namespace.yaml k8s/web.yaml
git commit -m "sync waves: namespace + configmap + deployment + service"
git push
```

✅ **Kết quả:** Trên ArgoCD UI (tab Sync), thứ tự apply đúng:
`Namespace (-1) → ConfigMap (0) → Deployment (1) → Service (2)`.
ArgoCD đợi wave trước `Healthy` rồi mới sang wave sau → tránh
`CreateContainerConfigError` (Deployment chạy trước khi ConfigMap tồn tại).

---

## Lab 7 — CI plan-on-PR + Branch Protection

**File:** `.github/workflows/validate.yml`
```yaml
name: validate
on:
  pull_request:
    paths:
      - "k8s/**"
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          curl -sSLo kc.tgz https://github.com/yannh/kubeconform/releases/download/v0.6.7/kubeconform-linux-amd64.tar.gz
          tar -xzf kc.tgz && sudo mv kubeconform /usr/local/bin/
      - run: kubeconform -strict -summary k8s/
```

**Branch protection** (`Settings → Branches → main`):
- ✔ Require a pull request before merging (+ approvals)
- ✔ Require status checks to pass → chọn `validate`
- ✔ No bypassing (admin cũng không push trực tiếp)

✅ **Kết quả:** Workflow chỉ **validate** (`kubeconform -strict`), **không có
job deploy** — CI là "gác cổng" cho PR, CD (apply vào cụm) hoàn toàn do ArgoCD
đảm nhiệm sau khi merge vào `main`.

---

## Tổng kết Evidence — Buổi sáng

| Tiêu chí | Kết quả |
|---|---|
| Cụm `w9` + ArgoCD cài thành công, tất cả pod Running | ✅ |
| Application `web` đầu tiên: apply tay 1 lần → ArgoCD tạo Deployment | ✅ |
| Đổi `replicas` qua Git → ArgoCD tự sync (không kubectl apply) | ✅ |
| Self-heal: sửa tay (`scale --replicas=9`) bị ArgoCD tự kéo về theo Git | ✅ |
| Rollback bằng `git revert` — có audit trail, không bị self-heal ghi đè | ✅ |
| App-of-Apps: `root` apply 1 lần, quản lý toàn bộ `argocd/apps/*` | ✅ |
| Sync Waves: Namespace → ConfigMap → Deployment → Service đúng thứ tự | ✅ |
| CI `validate.yml` (kubeconform) + Branch Protection trên `main` | ✅ |

---

# PHẦN B — Buổi chiều: Observability & Canary

## Lab 1 (buổi chiều) — Cài Prometheus stack & Argo Rollouts qua GitOps



**Thao tác:** Thêm `argocd/apps/kube-prometheus-stack.yaml` và
`argocd/apps/argo-rollouts.yaml`, commit + push → root App tự sync.

**Kết quả `kubectl -n argocd get applications`:**
```
NAME                    SYNC STATUS   HEALTH STATUS
root                    Synced        Healthy
tvshow                  Synced        Healthy
web                     Synced        Healthy
argo-rollouts           OutOfSync     Progressing
kube-prometheus-stack   OutOfSync     Healthy
```

**Pods trong `monitoring` (sau khi đầy đủ — 11 phút):**
```
NAME                                                        READY   STATUS      RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running     0          8m16s
kube-prometheus-stack-admission-create-jszgw                0/1     Completed   0          54s
kube-prometheus-stack-grafana-65bcd9c88-dk8g5               3/3     Running     0          11m
kube-prometheus-stack-kube-state-metrics-5ff4575db7-5qfg7   1/1     Running     0          11m
kube-prometheus-stack-operator-d69fb75b9-kt8mj              1/1     Running     0          11m
kube-prometheus-stack-prometheus-node-exporter-czspv        1/1     Running     0          11m
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running     0          8m12s
```

**Pods trong `argo-rollouts`:**
```
NAME                             READY   STATUS    RESTARTS   AGE
argo-rollouts-67cbbf7967-7s4bk   1/1     Running   0          13m
argo-rollouts-67cbbf7967-lblnc   1/1     Running   1          13m
```

✅ **Kết luận:** Toàn bộ stack được cài thành công qua Argo CD, không có
`helm install` hay `kubectl apply` thủ công.

---

## Lab 2 (buổi chiều) — Flask app `/metrics` + build image

**Files:** `app/app.py`, `app/Dockerfile`

**Build & load vào minikube:**
```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
```

✅ Image `w9-api:1` được load thành công vào cluster `w9`, dùng
`imagePullPolicy: IfNotPresent` để rollout sử dụng image local.

---

## Lab 3 (buổi chiều) — Rollout + ServiceMonitor cho `api`

**Files:** `k8s-api/api.yaml`, `k8s-api/servicemonitor.yaml`, `argocd/apps/api.yaml`

**Kết quả Application:**
```
NAME   SYNC STATUS   HEALTH STATUS
api    Synced        Healthy
```

**Pods `demo`:**
```
NAME                      READY   STATUS    RESTARTS   AGE
api-b848c847f-dlsb4       1/1     Running   1          7h27m
api-b848c847f-k445r       1/1     Running   1          7h27m
api-b848c847f-rfrnz       1/1     Running   1          7h27m
api-b848c847f-v4q52       1/1     Running   1          7h27m
```

✅ 4/4 pod `api` Running, sẵn sàng nhận traffic. `ServiceMonitor` (label
`release: kube-prometheus-stack`) được Prometheus tự phát hiện và scrape
endpoint `/metrics` mỗi 15s.

---

## Lab 4 (buổi chiều) — Canary thủ công (promote)

**Thao tác:** Đổi `VERSION: "v1" → "v2"`, commit + push.

**Trạng thái rollout — Paused tại step 1/5 (25%):**
```
Name:            api
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/5
  SetWeight:     25
  ActualWeight:  25
Images:          w9-api:1 (canary, stable)
Replicas:
  Desired:       4
  Updated:       1
```
```
├──# revision:2
│  └──⧉ api-849f474c4b           ReplicaSet  ✔ Healthy   canary
│     └──□ api-849f474c4b-rr9gx  Pod         ✔ Running
└──# revision:1
   └──⧉ api-b848c847f            ReplicaSet  ✔ Healthy   stable
```

**Lệnh promote:**
```bash
kubectl argo rollouts promote api -n demo
```

**Kết quả sau promote — step 3/5 (50%):**
```
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/5
  SetWeight:     50
  ActualWeight:  50
Replicas:
  Updated:       2
```
```
├──# revision:2
│  └──⧉ api-849f474c4b           ReplicaSet  ✔ Healthy   canary
│     ├──□ api-849f474c4b-rr9gx  Pod         ✔ Running
│     └──□ api-849f474c4b-2d4lg  Pod         ✔ Running
└──# revision:1
   └──⧉ api-b848c847f            ReplicaSet  stable (scaling down)
```

Sau pause tự động tiếp theo, rollout đạt **5/5, SetWeight: 100, Status: Healthy**
— canary v2 chính thức trở thành stable.

✅ **Kết luận:** Quy trình canary thủ công (đổi version → push → quan sát →
promote) chạy đúng pipeline GitOps end-to-end.

---

## Challenge — AnalysisTemplate tự động Abort theo SLO

**Files thêm:** `k8s-api/analysis-template.yaml`, cập nhật `analysis` trong
`k8s-api/api.yaml` (`startingStep: 2`), thêm `k8s-api/alert-rule.yaml`.

### Test case 1 — Inject lỗi (`ERROR_RATE="0.5"`, `VERSION="v3"`)

**Kết quả — Rollout tự động Aborted:**
```
Status:          ✖ Degraded
Message:         RolloutAborted: Rollout aborted update to revision 6:
                 Metric "success-rate" assessed Failed due to failed (4) > failureLimit (3)
Strategy:        Canary
  Step:          0/5
  SetWeight:     0
Images:          w9-api:1 (stable)
```

**Events (rollouts-controller):**
```
RolloutStepCompleted   Rollout step 1/5 completed (setWeight: 25)
AnalysisRunRunning     Background Analysis Run 'api-7774c9cd8-6' Status New: 'Running'
RolloutStepCompleted   Rollout step 3/5 completed (setWeight: 50)
AnalysisRunFailed      Background Analysis Run 'api-7774c9cd8-6' Status New: 'Failed' Previous: 'Running'
RolloutAborted         Rollout aborted update to revision 6: Metric "success-rate"
                       assessed Failed due to failed (4) > failureLimit (3)
SwitchService          Switched selector for service 'api-canary' from '7774c9cd8' to '758ccc86b8'
ScalingReplicaSet      Scaled up ReplicaSet api-758ccc86b8 (revision 4) from 2 to 4
ScalingReplicaSet      Scaled down ReplicaSet api-7774c9cd8 (revision 6) from 2 to 0
```

✅ **Kết luận test 1:** AnalysisRun đo `success-rate` mỗi 30s, phát hiện
4 lần liên tiếp < 0.95 (`failed (4) > failureLimit (3)`) → Argo Rollouts
**tự động abort**, switch service `api-canary` trở lại ReplicaSet stable cũ
(revision 4), scale canary về 0 — **không cần can thiệp thủ công**.

### Test case 2 — Fix lỗi (`ERROR_RATE="0"`, `VERSION="v5"`)

**Thao tác:** commit + push fix, force sync Argo CD:
```bash
kubectl -n argocd patch application api --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

**Kết quả — revision mới (7) chạy lại, vào canary 25%:**
```
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/5
  SetWeight:     25
Images:          w9-api:1 (canary, stable)
```
```
├──# revision:7
│  └──⧉ api-f4d499bc4            ReplicaSet  ✔ Healthy   canary
│     └──□ api-f4d499bc4-8gv58   Pod         ✔ Running
└──# revision:4
   └──⧉ api-758ccc86b8           ReplicaSet  ✔ Healthy   stable
```

Sau khi AnalysisRun bắt đầu chạy với `ERROR_RATE=0`, success-rate ≥ 0.95 →
AnalysisRun **Successful** → rollout tiếp tục tự động tới **5/5, Healthy**.

✅ **Kết luận test 2:** Khi metric tốt trở lại, canary deploy thành công bình
thường, AnalysisTemplate không cản trở rollout hợp lệ.

---

## Tổng kết Evidence — Buổi chiều

| Tiêu chí | Kết quả |
|---|---|
| Toàn bộ thay đổi qua Git push (no manual `kubectl apply`) | ✅ |
| Prometheus + Grafana + Alertmanager chạy qua Argo CD (Helm) | ✅ |
| Argo Rollouts controller chạy qua Argo CD (Helm) | ✅ |
| App `api` expose `/metrics`, được `ServiceMonitor` scrape | ✅ |
| Canary thủ công: promote 25% → 50% → 100% | ✅ |
| AnalysisTemplate đo `success-rate` từ Prometheus mỗi 30s | ✅ |
| Canary **tự động Abort + Rollback** khi SLO vi phạm (`failed > failureLimit`) | ✅ |
| Canary **tự động pass & promote** khi metric tốt trở lại | ✅ |
| `PrometheusRule` cảnh báo SLO độc lập (`ApiSuccessRateLow`) | ✅ |
