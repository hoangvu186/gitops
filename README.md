# W9 — GitOps Foundations + Observability & Progressive Delivery (Canary)

Repo này là kết quả của **2 buổi học W9**:
- **Buổi sáng:** Xây dựng nền tảng GitOps (ArgoCD, app-of-apps, self-heal, rollback bằng `git revert`, sync waves, CI validate)
- **Buổi chiều:** Observability (Prometheus/Grafana) + Progressive Delivery (Argo Rollouts Canary + Automated Analysis)


Repo này triển khai một pipeline **GitOps** hoàn chỉnh trên Kubernetes (minikube),
sử dụng **Argo CD** (App-of-Apps) để quản lý toàn bộ infrastructure và application,
**kube-prometheus-stack** cho observability, và **Argo Rollouts** để thực hiện
**Canary Deployment có Automated Analysis** dựa trên metric Prometheus.

---

## 0. Nguyên lý GitOps & ArgoCD (buổi sáng)

### 0.1. Tại sao cần GitOps?
Deploy tay (`kubectl apply` từ laptop) gặp 4 vấn đề lớn:
1. **Không ghi lại state** — không ai biết cụm đang chạy version nào
2. **Nhầm context** — gõ đúng lệnh nhưng vào sai cluster (vd: xóa nhầm prod)
3. **Không rollback được** — `kubectl rollout undo` chỉ lưu 10 revisions, không có message/audit
4. **Credential cũ còn tồn** — người nghỉ việc vẫn giữ kubeconfig có quyền apply vào cụm

→ Gốc rễ: không có **single source of truth**.

### 0.2. 4 nguyên tắc OpenGitOps
| Nguyên tắc | Ý nghĩa |
|---|---|
| **Declarative** | Khai báo "muốn gì" (`replicas: 3`), không viết "làm thế nào" |
| **Versioned** | Mọi thay đổi nằm trong Git → có lịch sử, audit trail đầy đủ |
| **Pulled** | Agent (ArgoCD) trong cụm tự kéo từ Git — không ai push từ ngoài vào |
| **Reconciled** | ArgoCD liên tục so Git vs cụm thật; lệch → tự sửa về theo Git |

**Model:** Git = bản thiết kế · ArgoCD = thợ sửa cụm cho khớp thiết kế.

### 0.3. ArgoCD core concepts
- **Application** (CRD, namespace `argocd`) = "chỉ dẫn" cho ArgoCD: `source` (repo+branch+path),
  `destination` (namespace+cluster), `syncPolicy` (`automated.prune` + `automated.selfHeal`)
- **3 hoạt động:**
  - *Sync*: đưa cụm khớp Git
  - *Self-Heal*: ai sửa tay làm lệch Git → ArgoCD tự kéo về theo Git
  - *Prune*: xóa resource khỏi Git → ArgoCD xóa khỏi cụm (mặc định tắt, phải bật `prune: true`)
- **Synced ≠ Healthy**: `Synced` = cụm khớp YAML trong Git; `Healthy` = Pod chạy tốt.
  Một image lỗi vẫn có thể `Synced` ✅ nhưng `Degraded` ❌.

### 0.4. App-of-Apps pattern
- Không dùng app-of-apps → mỗi app mới phải `kubectl apply` Application thủ công, không GitOps 100%.
- Dùng app-of-apps → apply `root` **1 lần duy nhất**, sau đó **thêm app = thêm file YAML vào `argocd/apps/` + git push**, root tự phát hiện và tạo Application con.

```yaml
# argocd/root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: root, namespace: argocd }
spec:
  source:
    repoURL: https://github.com/<ban>/gitops.git
    targetRevision: HEAD
    path: argocd/apps
  destination: { server: https://kubernetes.default.svc, namespace: argocd }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
```

### 0.5. Sync Waves — ép thứ tự apply
Dùng annotation `argocd.argoproj.io/sync-wave: "N"` (wave nhỏ chạy trước, ArgoCD đợi wave trước Healthy mới sang wave sau):

```
Namespace (wave -1) → ConfigMap (wave 0) → Deployment (wave 1) → Service (wave 2)
```
Thiếu wave: Deployment chạy trước khi ConfigMap tồn tại → `CreateContainerConfigError`.

### 0.6. Rollback đúng cách: `git revert`, KHÔNG `kubectl rollout undo`
Vì nguyên tắc *Reconciled* — cụm phải khớp Git, kể cả khi rollback:

```bash
# ❌ SAI — self-heal sẽ ghi đè lại version mới sau vài giây/phút
kubectl rollout undo deployment/backend

# ✅ ĐÚNG — sửa nguồn sự thật, có audit trail (commit author + message)
git revert HEAD --no-edit && git push
```

| | `kubectl rollout undo` | `git revert` |
|---|---|---|
| Tác động | Chỉ cụm | Git + Cụm |
| Lịch sử | Không (mất sau 10 revisions) | Có (Git history vĩnh viễn) |
| Bị self-heal ghi đè | ✅ Có (rollback giả) | ❌ Không |
| Audit trail | Không biết ai rollback | Commit author + message |

### 0.7. CI/CD: plan-on-PR / apply-on-merge
- **CI** (`.github/workflows/validate.yml`) chạy `on: pull_request`, dùng `kubeconform -strict` để validate YAML — **KHÔNG có job deploy**.
- **CD** = ArgoCD tự poll/webhook sau khi PR merge vào `main` → apply tự động.
- **Branch protection** trên `main`: require PR review + require status check `validate` xanh + no bypass — không ai push thẳng `main`.

---

## 1. Kiến trúc tổng quan

```
argocd/
├── root.yaml                     # App-of-Apps: quản lý tất cả Application con
└── apps/
    ├── web.yaml                    # App demo "web" (sync waves: Namespace/ConfigMap/Deploy/Svc)
    ├── kube-prometheus-stack.yaml # Prometheus + Grafana + Alertmanager (Helm)
    ├── argo-rollouts.yaml         # Argo Rollouts controller (Helm)
    ├── tvshow.yaml                # App demo "tvshow"
    └── api.yaml                   # App "api" (Flask, có canary + analysis)

k8s/
├── namespace.yaml           # Namespace "demo" (sync-wave: -1)
└── web.yaml                 # ConfigMap (wave 0) + Deployment (wave 1) + Service (wave 2)

.github/workflows/
└── validate.yml             # CI plan-on-PR: kubeconform -strict k8s/

k8s-tvshow/
└── deployment.yaml

k8s-api/
├── api.yaml                # Rollout (canary) + Service + Service canary cho "api"
├── servicemonitor.yaml     # ServiceMonitor scrape /metrics cho Prometheus
├── analysis-template.yaml  # AnalysisTemplate "success-rate" cho canary
└── alert-rule.yaml          # PrometheusRule cảnh báo SLO success-rate < 95%

app/
├── app.py        # Flask app expose /, /healthz, /metrics
└── Dockerfile
```

Toàn bộ cluster state được đồng bộ qua **Argo CD** từ repo Git này (`targetRevision: HEAD`),
với `syncPolicy.automated` (prune + selfHeal) — mọi thay đổi muốn áp dụng vào cluster
**phải đi qua `git push`**, không `kubectl apply` thủ công.

---

## 2. Các thành phần chính

### 2.1. Observability — kube-prometheus-stack
- Cài qua Helm chart `kube-prometheus-stack` (namespace `monitoring`)
- Bao gồm: Prometheus, Grafana (admin/admin123), Alertmanager, kube-state-metrics, node-exporter
- `serviceMonitorSelectorNilUsesHelmValues: false` để Prometheus tự nhận mọi `ServiceMonitor`
  có label `release: kube-prometheus-stack`

### 2.2. Progressive Delivery — Argo Rollouts
- Cài qua Helm chart `argo-rollouts` (namespace `argo-rollouts`)
- Plugin CLI `kubectl argo rollouts` dùng để quan sát & điều khiển canary

### 2.3. Ứng dụng demo — `api`
Flask app đơn giản (`app/app.py`):

| Endpoint   | Mô tả |
|------------|-------|
| `GET /`        | Trả `200 {ok:true, version:VERSION}` hoặc `500 {error:"injected"}` theo xác suất `ERROR_RATE` |
| `GET /healthz` | Health check cho readiness probe |
| `GET /metrics` | Expose bằng `prometheus_flask_exporter` — Prometheus scrape mỗi 15s |

Biến môi trường:
- `VERSION` — version hiện tại của app (hiển thị trong response)
- `ERROR_RATE` — tỉ lệ lỗi 500 nhân tạo (dùng để test SLO / canary abort)

### 2.4. Canary Strategy (`k8s-api/api.yaml`)
```yaml
strategy:
  canary:
    canaryService: api-canary
    stableService: api
    analysis:
      templates:
      - templateName: success-rate
      startingStep: 2
    steps:
    - setWeight: 25
    - pause:
        duration: 1m
    - setWeight: 50
    - pause:
        duration: 2m
    - setWeight: 100
```

- Step 1: 25% traffic sang canary, pause 1 phút
- Từ step 2: **AnalysisRun** bắt đầu chạy song song, kiểm tra `success-rate` mỗi 30s
- Step 3: 50% traffic, pause 2 phút
- Step 5: 100% — canary trở thành stable

### 2.5. AnalysisTemplate — `success-rate`
```yaml
successCondition: result[0] >= 0.95
failureLimit: 3
query: |
  sum(rate(flask_http_request_total{namespace="demo",status!~"5.."}[2m]))
  /
  sum(rate(flask_http_request_total{namespace="demo"}[2m]))
```

Nếu tỉ lệ request **không lỗi 5xx** trong 2 phút gần nhất < 95%, và điều này lặp
quá `failureLimit: 3` lần check liên tiếp → **Rollout tự động Abort**, rollback
service về ReplicaSet stable cũ.

### 2.6. SLO Alert — `alert-rule.yaml`
`PrometheusRule` bắn cảnh báo `ApiSuccessRateLow` (severity: critical) khi
success-rate < 95% trong ≥ 30 giây — độc lập với cơ chế canary abort, dùng cho
Alertmanager/Slack/Grafana alerting.

---

## 3. Quy trình thực hiện

### Buổi sáng — GitOps Foundations (Lab 0 → 7)

| Lab | Nội dung | Cách thực hiện |
|-----|----------|-----------------|
| **Lab 0** | Dựng cụm + viết app đầu tiên + push Git | `minikube start -p w9 --driver=docker`, tạo repo `gitops`, viết `k8s/web.yaml` (Deployment nginx), push lên GitHub — **chưa apply** |
| **Lab 1** | Cài ArgoCD | `kubectl create ns argocd` + `kubectl apply --server-side -f .../install.yaml`, đợi `argocd-*` pods Running |
| **Lab 2** | Tạo Application đầu tiên | Viết `argocd/apps/web.yaml`, `kubectl apply -f argocd/apps/web.yaml` (apply tay — **lần cuối** trước khi có root) → ArgoCD tự tạo Deployment `web` trong `demo` |
| **Lab 3** | Sync qua Git & Self-Heal | Đổi `replicas: 2→4` trong `k8s/web.yaml`, push → ArgoCD tự sync. Test self-heal: `kubectl scale deploy/web --replicas=9` → ArgoCD tự kéo về theo Git |
| **Lab 4** ⭐ | Rollback bằng `git revert` | `git revert HEAD --no-edit && git push` — rollback thật có audit trail, không bị self-heal ghi đè (khác `kubectl rollout undo`) |
| **Lab 5** | App-of-Apps | Viết `argocd/root.yaml`, `kubectl apply -f argocd/root.yaml` (apply tay **lần cuối cùng**) → root tiếp quản `web`, từ đây thêm app = thả file vào `argocd/apps/` + push |
| **Lab 6** | Sync Waves | Thêm `k8s/namespace.yaml` (wave -1) + cập nhật `k8s/web.yaml` thành ConfigMap (wave 0) → Deployment (wave 1, dùng `envFrom`) → Service (wave 2) |
| **Lab 7** | CI plan-on-PR | `.github/workflows/validate.yml` chạy `kubeconform -strict` trên PR; cấu hình Branch Protection cho `main` (require PR review + status check `validate`) |

### Buổi chiều — Observability & Canary (Lab 1 → 4 + Challenge)

| Lab | Nội dung | Cách thực hiện |
|-----|----------|-----------------|
| **Lab 1** | Cài Prometheus stack + Argo Rollouts | Thêm 2 `Application` (Helm) vào `argocd/apps/`, push → root App tự sync |
| **Lab 2** | Viết Flask app có `/metrics`, build image | `app/app.py` + `app/Dockerfile`, build `w9-api:1`, `minikube image load` |
| **Lab 3** | Deploy `Rollout` + `ServiceMonitor` cho `api` | `k8s-api/api.yaml`, `k8s-api/servicemonitor.yaml`, `argocd/apps/api.yaml`, verify target `UP` trên Prometheus |
| **Lab 4** | Canary thủ công | Đổi `VERSION` → commit/push → `kubectl argo rollouts get rollout api -n demo --watch` → `promote` / `abort` bằng tay |
| **Challenge** | Canary tự động abort theo SLO | Thêm `analysis-template.yaml` + cấu hình `analysis` trong `api.yaml`; test với `ERROR_RATE="0.5"` → AnalysisRun fail → Rollout tự abort & rollback |

---

## 4. Các lệnh quan trọng

### Buổi sáng (GitOps Foundations)
```bash
# Khởi tạo cụm
minikube start -p w9 --driver=docker
kubectl config use-context w9

# Cài ArgoCD (server-side vì CRD rất lớn)
kubectl create ns argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server

# Mở ArgoCD UI
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Rollback đúng cách (GitOps)
git revert HEAD --no-edit && git push
```

### Buổi chiều (Observability & Canary)
```bash
# Khởi động lại cluster (sau khi tắt máy / restart Docker)
minikube start -p w9

# Xem toàn bộ Argo CD Applications
kubectl -n argocd get applications

# Theo dõi canary rollout
kubectl argo rollouts get rollout api -n demo --watch

# Promote canary sang step tiếp theo
kubectl argo rollouts promote api -n demo

# Abort canary, rollback về stable
kubectl argo rollouts abort api -n demo

# Retry rollout sau khi đã abort (deploy lại sau khi fix)
kubectl argo rollouts retry rollout api -n demo

# Port-forward Prometheus để xem targets / query
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090
# → mở http://localhost:9090
```

---

## 5. Kết quả đạt được

### Buổi sáng — GitOps Foundations
- ✅ Cụm `w9` (minikube, driver docker) chạy ArgoCD, namespace `argocd` Running
- ✅ Application `web` (apply tay 1 lần) → ArgoCD tự tạo Deployment 2 pod `nginx:1.27` trong `demo`
- ✅ Đổi `replicas: 2→4` qua Git → ArgoCD tự sync lên 4 pod, không `kubectl apply`
- ✅ Self-heal: `kubectl scale --replicas=9` bị ArgoCD tự kéo về 4 (theo Git) sau vài giây
- ✅ Rollback bằng `git revert HEAD --no-edit` — có audit trail, không bị self-heal ghi đè lại version lỗi
- ✅ App-of-Apps: `argocd/root.yaml` apply 1 lần → root quản lý toàn bộ `argocd/apps/*`, không cần `kubectl apply` cho app mới
- ✅ Sync Waves: `Namespace (-1) → ConfigMap (0) → Deployment (1) → Service (2)` apply đúng thứ tự
- ✅ CI `validate.yml` (`kubeconform -strict`) chạy trên PR, branch protection chặn merge khi CI đỏ

### Buổi chiều — Observability & Canary
- ✅ Toàn bộ infra (Prometheus, Grafana, Alertmanager, Argo Rollouts) được cài
  **100% qua GitOps** (Argo CD App-of-Apps, không `helm install` thủ công)
- ✅ App `api` expose metric Prometheus, `ServiceMonitor` được Prometheus scrape thành công
- ✅ Canary deployment chạy theo từng bước (25% → 50% → 100%) qua Argo Rollouts
- ✅ Canary có thể **promote / abort thủ công** qua `kubectl argo rollouts`
- ✅ Canary **tự động abort & rollback** khi inject lỗi (`ERROR_RATE=0.5`) làm
  success-rate < 95%, AnalysisRun fail quá `failureLimit`
- ✅ Sau khi fix (`ERROR_RATE=0`), rollout mới chạy lại và pass AnalysisRun → Healthy

Chi tiết log/evidence của từng bước: xem [`EVIDENCE.md`](./EVIDENCE.md).
