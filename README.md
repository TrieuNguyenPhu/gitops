# 🚀 GitOps Progressive Delivery Pipeline

[![ArgoCD](https://img.shields.io/badge/ArgoCD-Autopilot-blue?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![Argo Rollouts](https://img.shields.io/badge/Argo_Rollouts-Canary-orange?logo=argo&logoColor=white)](https://argoproj.github.io/rollouts/)
[![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-red?logo=prometheus&logoColor=white)](https://prometheus.io/)

> **Mục tiêu**: Xây dựng pipeline triển khai ứng dụng API an toàn và tự bảo vệ — mọi thay đổi qua Git, đo lường tự động bằng SLO, và Canary Rollout tự abort khi phát hiện lỗi.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PROGRESSIVE DELIVERY PIPELINE                    │
│                                                                      │
│   git push ──→ ArgoCD Sync ──→ Canary 25% ──→ Analysis ──→ 100%    │
│                                     │              │                 │
│                                     │         FAIL (x3)              │
│                                     │              │                 │
│                                     │         Auto Abort             │
│                                     │              │                 │
│   git revert ←── Rollback ←─────────┘──────────────┘                │
│                                                                      │
│   SLO < 95% ──→ Alert Fire ──→ Email Notification                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Mục lục

- [Kiến trúc tổng quan](#-kiến-trúc-tổng-quan)
- [Cấu trúc Repository](#-cấu-trúc-repository)
- [Phase 1: Nền tảng GitOps & ArgoCD](#-phase-1-nền-tảng-gitops--argocd)
- [Phase 2: Observability, Metrics & Canary](#-phase-2-observability-metrics--canary-rollouts)
- [Giải thích Prometheus Query & Ngưỡng](#-giải-thích-prometheus-query--ngưỡng)
- [Kịch bản Demo](#-kịch-bản-demo-end-to-end)
- [Tổng hợp lệnh từ đầu đến cuối](#-tổng-hợp-toàn-bộ-lệnh-từ-đầu-đến-cuối)

---

## 🏗 Kiến trúc tổng quan

```
                    ┌─────────────────────────────────────────────┐
                    │              GitHub Repository               │
                    │                                              │
                    │  argocd/apps/     k8s/      k8s-api/        │
                    └──────────┬───────────────────────────────────┘
                               │ auto sync
                    ┌──────────▼──────────┐
                    │       ArgoCD        │
                    │   (App-of-Apps)     │
                    └──┬─────┬────────┬──┘
                       │     │        │
          ┌────────────▼┐ ┌──▼─────┐ ┌▼──────────────┐
          │ Prometheus  │ │  Argo  │ │   API App      │
          │   Stack     │ │Rollouts│ │ (Rollout +     │
          │ + Grafana   │ │        │ │ AnalysisTempl) │
          │ + Alertmgr  │ │        │ │ + SLO/Alert    │
          └──────┬──────┘ └────────┘ └───────┬────────┘
                 │    metrics scrape          │
                 └───────────────────────────-┘
```

**Công nghệ sử dụng:**

| Thành phần | Công nghệ | Vai trò |
|:---|:---|:---|
| Cluster | Minikube (`w9`) | Cụm Kubernetes local |
| GitOps | ArgoCD + App-of-Apps | Tự đồng bộ Git → Cluster |
| Monitoring | kube-prometheus-stack | Prometheus + Grafana + Alertmanager |
| Progressive Delivery | Argo Rollouts | Canary deployment + AnalysisTemplate |
| Application | Flask + prometheus-flask-exporter | API có xuất metrics tại `/metrics` |
| CI | GitHub Actions + kubeconform | Kiểm tra YAML schema trên mỗi PR |

---

## 📂 Cấu trúc Repository

```
gitops/
│
├── app/                              # Mã nguồn ứng dụng
│   ├── app.py                        #   Flask API + Prometheus Exporter
│   └── Dockerfile                    #   Đóng gói Docker Image
│
├── k8s/                              # Manifest ứng dụng Web (Nginx)
│   ├── namespace.yaml                #   Namespace demo (sync-wave: -1)
│   └── web.yaml                      #   Deployment + ConfigMap + Service
│
├── k8s-api/                          # Manifest ứng dụng API (Flask)
│   ├── api.yaml                      #   Rollout (Canary + Analysis) + Service
│   ├── analysis-template.yaml        #   AnalysisTemplate: đo success rate từ Prometheus
│   ├── slo-alert.yaml                #   PrometheusRule: SLO 95% + Alert email
│   └── servicemonitor.yaml           #   ServiceMonitor: Prometheus scrape /metrics
│
├── argocd/
│   ├── root.yaml                     # Root Application (App-of-Apps pattern)
│   └── apps/
│       ├── api.yaml                  #   ArgoCD App → k8s-api/
│       ├── web.yaml                  #   ArgoCD App → k8s/
│       ├── kube-prometheus-stack.yaml #   Helm Chart: Prometheus + Alertmanager
│       └── argo-rollouts.yaml        #   Helm Chart: Argo Rollouts Controller
│
├── .github/workflows/
│   └── validate.yml                  # CI: kubeconform validation trên PR
│
├── alertmanager-secret.yaml          # ⛔ GITIGNORED — chứa SMTP password
└── .gitignore
```

---

## 🌟 Phase 1: Nền tảng GitOps & ArgoCD

<details>
<summary><b>Lab 0 — Khởi tạo Cụm Local & Git Repository</b></summary>

- Khởi tạo cụm Kubernetes local bằng Minikube tên là `w9`.
- Tạo cấu hình triển khai ứng dụng cơ bản (`web` deployment chạy image `nginx:1.27`).
- Thiết lập Git repository local và đẩy lên GitHub làm kho chứa chính thức.
</details>

<details>
<summary><b>Lab 1 — Cài đặt ArgoCD vào Cụm Kubernetes</b></summary>

- Tạo namespace chuyên biệt `argocd`.
- Triển khai bộ tài nguyên ArgoCD bằng chế độ `--server-side` để tránh lỗi dung lượng CRD lớn.
- Lấy thông tin mật khẩu admin ban đầu và mở cổng truy cập dashboard local (port-forward).
</details>

<details>
<summary><b>Lab 2 — Khởi tạo Application trong ArgoCD</b></summary>

- Viết cấu hình `Application` trỏ tới thư mục `k8s/` chứa manifest ứng dụng.
- Áp dụng cấu hình lên cụm, ArgoCD tự động triển khai 2 bản sao ứng dụng Nginx vào namespace `demo`.
</details>

<details>
<summary><b>Lab 3 — Tự động Đồng bộ (Auto Sync) & Tự phục hồi (Self-Heal)</b></summary>

- **Auto Sync**: Sửa replicas `2` → `4` trong Git → ArgoCD tự động tăng số pod.
- **Self-Heal**: Can thiệp thủ công scale lên `9` replicas → ArgoCD ngay lập tức tự kéo giảm về `4` replicas đúng cấu hình Git.
</details>

<details>
<summary><b>Lab 4 — Rollback an toàn qua Git Revert</b></summary>

- Thực hiện rollback bằng commit đảo ngược (`git revert HEAD`).
- Khi push lên GitHub, cụm tự động được đưa về trạng thái trước đó mà vẫn lưu vết lịch sử.
</details>

<details>
<summary><b>Lab 5 — Kiến trúc App-of-Apps</b></summary>

- Thiết lập Root Application (`root.yaml`) quản lý toàn bộ ứng dụng con trong `argocd/apps/`.
- Chỉ cần thả file Application mới vào `argocd/apps/` và push → ArgoCD tự tạo tài nguyên.
</details>

<details>
<summary><b>Lab 6 — Quản lý thứ tự triển khai (Sync Waves)</b></summary>

Sử dụng annotation `argocd.argoproj.io/sync-wave`:
| Wave | Tài nguyên | Mục đích |
|:----:|:-----------|:---------|
| `-1` | Namespace `demo` | Tạo trước tiên |
| `0` | ConfigMap `web-config` | Chuẩn bị cấu hình |
| `1` | Deployment `web` | Triển khai ứng dụng |
| `2` | Service `web` | Mở cổng kết nối |
</details>

<details>
<summary><b>Lab 7 — Tích hợp CI (Plan-on-PR) & Branch Protection</b></summary>

- GitHub Actions chạy `kubeconform` kiểm tra schema YAML trên mỗi Pull Request.
- Branch Protection buộc PR phải qua CI thành công mới được merge vào `main`.
</details>

---

## 🚀 Phase 2: Observability, Metrics & Canary Rollouts

### Lab 1 — Cài đặt Prometheus Stack + Argo Rollouts (qua GitOps)

Tạo Application Helm Chart cho `kube-prometheus-stack` và `argo-rollouts` trong `argocd/apps/`, đẩy lên Git để Root App tự đồng bộ và cài đặt lên cụm.

### Lab 2 — Tự viết App Flask & Đóng gói Docker Image

Viết ứng dụng Flask Python tích hợp `prometheus-flask-exporter`, đóng gói bằng Docker, và nạp vào cụm Minikube:

```python
# app/app.py — Tích hợp sẵn biến ERROR_RATE để inject lỗi khi demo
ERR = float(os.getenv("ERROR_RATE", "0"))    # 0 = không lỗi, 0.5 = 50% lỗi
VER = os.getenv("VERSION", "v1")             # Phân biệt phiên bản canary
```

### Lab 3 — Triển khai Rollout Canary & Khớp Prometheus Metrics

Khai báo `Rollout` với chiến lược Canary + `ServiceMonitor` để Prometheus cào metric `/metrics`. Tạo pod `load` giả lập traffic liên tục.

### Lab 4 — Điều khiển Canary Rollout thủ công (Promote)

Nâng `VERSION` lên `v2`, hệ thống dừng ở 25%. Dùng CLI `promote` để tiếp tục lên 100%.

### Lab 5 — SLO Recording Rule & Alert gửi Email

> **File**: [`k8s-api/slo-alert.yaml`](k8s-api/slo-alert.yaml)

Thiết lập đo lường chất lượng dịch vụ (SLO) và cảnh báo tự động:

| Tài nguyên | Tên | Mô tả |
|:---|:---|:---|
| Recording Rule | `api:success_rate:5m` | Tỷ lệ thành công (non-5xx) trên cửa sổ 5 phút |
| Alert Rule | `ApiHighErrorRate` | Fire khi tỷ lệ < 95% liên tục 1 phút → email |

Cấu hình Alertmanager gửi email qua SMTP Gmail. Password được bảo mật trong Kubernetes Secret riêng (gitignored), tham chiếu qua `configSecret`.

### Lab 6 — AnalysisTemplate: Canary Tự động Đo lường & Tự Abort

> **File**: [`k8s-api/analysis-template.yaml`](k8s-api/analysis-template.yaml)

Thay thế bước `pause: {}` chờ thủ công bằng `AnalysisTemplate` tên `success-rate`:

```
Canary Flow:  25% ──→ Analysis (đo metric) ──→ 50% ──→ pause 30s ──→ 100%
                            │
                       FAIL x3 lần
                            │
                      Auto Abort ──→ Rollback về bản cũ
```

| Tham số | Giá trị | Ý nghĩa |
|:---|:---|:---|
| `interval` | `30s` | Đo lại mỗi 30 giây |
| `successCondition` | `result[0] >= 0.95` | ≥ 95% request thành công → PASS |
| `failureLimit` | `3` | Cho phép tối đa 3 lần FAIL trước khi abort |

### Lab 7 — Kịch bản Demo: Canary Auto-Abort & Git Revert Rollback

Chứng minh toàn bộ pipeline hoạt động end-to-end (xem mục [Kịch bản Demo](#-kịch-bản-demo-end-to-end)).

---

## 📊 Giải thích Prometheus Query & Ngưỡng

### SLO Query — Dùng trong `PrometheusRule` (slo-alert.yaml)

```promql
sum(rate(flask_http_request_total{status!~"5..",namespace="demo",service="api"}[5m]))
/
sum(rate(flask_http_request_total{namespace="demo",service="api"}[5m]))
```

| Thành phần | Giải thích |
|:---|:---|
| `flask_http_request_total` | Metric do Flask Prometheus Exporter tự sinh, đếm tổng HTTP request |
| `status!~"5.."` | Regex lọc bỏ các status 5xx (lỗi server) — chỉ giữ request thành công |
| `rate(...[5m])` | Tính tốc độ request trung bình trong cửa sổ 5 phút gần nhất |
| `sum(success) / sum(total)` | **Tỷ lệ thành công** — kết quả từ 0.0 → 1.0 |

> **Ngưỡng SLO**: ≥ **95%** (0.95). Dưới 95% liên tục 1 phút → fire alert `ApiHighErrorRate` → gửi email.

### AnalysisTemplate Query — Dùng trong `AnalysisTemplate` (analysis-template.yaml)

```promql
sum(rate(flask_http_request_total{status!~"5..",namespace="demo",service="api"}[2m]))
/
sum(rate(flask_http_request_total{namespace="demo",service="api"}[2m]))
```

> Cùng logic với SLO nhưng dùng cửa sổ **2 phút** (ngắn hơn) để phản ứng nhanh trong quá trình Canary.
>
> **Ngưỡng**: ≥ **95%** → PASS. 3 lần liên tiếp < 95% → **tự động abort canary**.

### Tại sao chọn 95%?

- **Quá cao** (99%): Nhạy quá, dễ false positive khi traffic thấp.
- **Quá thấp** (80%): Cho phép quá nhiều lỗi qua canary.
- **95%**: Cân bằng giữa độ nhạy và độ ổn định, phù hợp cho lab environment.

---

## 🧪 Kịch bản Demo End-to-End

### Kịch bản 1 — Bản tốt → tự động lên 100% ✅

```bash
# Sửa api.yaml: VERSION="v3", ERROR_RATE="0"
git add . && git commit -m "release v3 - good version" && git push
```

```
Rollout: 25% → Analysis đo metric → 95%+ → PASS → 50% → pause 30s → 100%
Kết quả: Toàn bộ pod chạy v3 ✅
```

### Kịch bản 2 — Bản lỗi → tự abort ❌ → rollback ✅

```bash
# Sửa api.yaml: VERSION="v4", ERROR_RATE="0.5"
git add . && git commit -m "release v4 - inject 50% errors" && git push
```

```
Rollout: 25% → Analysis đo metric → ~50% < 95% → FAIL
                                   → FAIL (lần 2)
                                   → FAIL (lần 3)
                                   → AUTO ABORT → Rollback về v3 ✅
Alert:   ApiHighErrorRate FIRING → Email gửi tới nguyentrieu080604@gmail.com ✅
```

### Kịch bản 3 — Git revert rollback < 5 phút ⏱️

```bash
git revert HEAD
git push
# ArgoCD sync → cụm trở về cấu hình trước (v3, ERROR_RATE=0) ✅
```

---

## ⚙️ Tổng hợp toàn bộ lệnh từ đầu đến cuối

### Phase 1 — Nền tảng GitOps

```bash
# [Lab 0] Tạo cụm Kubernetes local
minikube start -p w9

# [Lab 1] Cài ArgoCD
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side

# Lấy mật khẩu admin ArgoCD
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Mở ArgoCD Dashboard → https://localhost:8080
kubectl -n argocd port-forward svc/argocd-server 8080:443

# [Lab 2] Áp dụng Root App (App-of-Apps) — lệnh apply DUY NHẤT cần chạy
kubectl apply -f argocd/root.yaml
```

### Phase 2 — Observability & Canary

```bash
# [Lab 2] Build & load Docker image API
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9

# [Lab 3] Tạo pod traffic giả lập (chạy 1 lần)
kubectl -n demo run load --image=busybox --restart=Never \
  -- sh -c "while true; do wget -qO- api:8080/; done"

# [Lab 5] Apply Alertmanager Secret (1 lần duy nhất, file GITIGNORED)
kubectl apply -f alertmanager-secret.yaml
```

### Port-forward dịch vụ giám sát

```bash
# Prometheus UI → http://localhost:9090
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090

# Grafana UI → http://localhost:3000 (admin / prom-operator)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000

# Alertmanager UI → http://localhost:9093
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093
```

### Kiểm tra & Debug

```bash
# Xem trạng thái rollout (real-time)
.\kubectl-argo-rollouts.exe get rollout api -n demo -w

# Xem kết quả AnalysisRun
kubectl -n demo get analysisrun
kubectl -n demo describe analysisrun -l rollouts-pod-template-hash

# Xem SLO trên Prometheus → query: api:success_rate:5m
# Xem PrometheusRule đã load chưa
kubectl -n demo get prometheusrule

# Xem pods
kubectl -n demo get pods

# Xem log Argo Rollouts controller
kubectl -n argo-rollouts logs -l app.kubernetes.io/name=argo-rollouts

# Git revert rollback
git revert HEAD
git push
```
