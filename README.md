# GitOps Training Course: From Zero to Production-Ready Progressive Delivery

Repository này chứa toàn bộ mã nguồn cấu hình Kubernetes, ArgoCD Application, CI workflow và cấu hình Observability phục vụ cho chuỗi bài thực hành GitOps (Phase 1 & Phase 2).

---

## 🗺️ Bản đồ các Lab đã thực hiện

### 🌟 Phase 1: Nền tảng GitOps & ArgoCD

#### [Lab 0] Khởi tạo Cụm Local & Git Repository
* Khởi tạo cụm Kubernetes local bằng Minikube tên là `w9`.
* Tạo cấu hình triển khai ứng dụng cơ bản (`web` deployment chạy image `nginx:1.27`).
* Thiết lập Git repository local và đẩy lên GitHub làm kho chứa chính thức.

#### [Lab 1] Cài đặt ArgoCD vào Cụm Kubernetes
* Tạo namespace chuyên biệt `argocd`.
* Triển khai bộ tài nguyên ArgoCD bằng chế độ `--server-side` để tránh lỗi dung lượng CRD lớn.
* Lấy thông tin mật khẩu admin ban đầu và mở cổng truy cập dashboard local (port-forward).

#### [Lab 2] Khởi tạo Application trong ArgoCD
* Viết cấu hình `Application` trỏ tới thư mục `k8s/` chứa manifest ứng dụng.
* Áp dụng cấu hình lên cụm, ArgoCD tự động triển khai 2 bản sao (replicas) ứng dụng Nginx vào namespace `demo`.

#### [Lab 3] Cơ chế Tự động Đồng bộ (Auto Sync) & Tự phục hồi (Self-Heal)
* **Auto Sync**: Sửa số lượng replicas từ `2` thành `4` trong Git $\rightarrow$ ArgoCD tự động phát hiện và tăng số pod.
* **Self-Heal**: Cố tình can thiệp thủ công bằng lệnh scale lên `9` replicas $\rightarrow$ ArgoCD ngay lập tức tự động kéo giảm về đúng `4` replicas theo đúng cấu hình Git.

#### [Lab 4] Rollback an toàn thông qua Git Revert
* Thực hiện rollback thông qua việc tạo commit đảo ngược trạng thái trên Git (`git revert HEAD`).
* Khi thay đổi được đẩy lên GitHub, cụm tự động được đưa về trạng thái trước đó (2 replicas) mà vẫn lưu vết lịch sử phiên bản.

#### [Lab 5] Kiến trúc App-of-Apps
* Thiết lập cấu hình ứng dụng Root (`root.yaml`) quản lý toàn bộ các khai báo ứng dụng con nằm trong thư mục `argocd/apps/`.
* Giờ đây, chỉ cần khai báo file cấu hình ứng dụng mới thả vào `argocd/apps/` và push lên Git $\rightarrow$ ArgoCD tự động tạo tài nguyên mà không cần chạy lệnh apply thủ công.

#### [Lab 6] Quản lý thứ tự triển khai tài nguyên (Sync Waves)
* Sử dụng annotation `argocd.argoproj.io/sync-wave` để đánh số thứ tự triển khai tài nguyên từ nhỏ đến lớn:
  * **Namespace** `demo`: Wave `-1` (Tạo trước tiên).
  * **ConfigMap** `web-config`: Wave `0` (Tạo cấu hình).
  * **Deployment** `web`: Wave `1` (Triển khai ứng dụng).
  * **Service** `web`: Wave `2` (Mở cổng kết nối dịch vụ).

#### [Lab 7] Tích hợp CI (Plan-on-PR) & Branch Protection
* Xây dựng luồng công việc CI bằng GitHub Actions sử dụng công cụ `kubeconform` để kiểm tra lỗi schema của toàn bộ file YAML trong thư mục `k8s/` mỗi khi có Pull Request.
* Cấu hình Branch Protection trên GitHub buộc các PR phải qua bước chạy thử CI thành công mới được phép Merge vào nhánh `main`.

---

### 🚀 Phase 2: Observability, Metrics & Canary Rollouts

#### [Lab 1] Cài đặt Prometheus Stack + Argo Rollouts (qua GitOps)
* **Mục tiêu**: Thiết lập hệ thống giám sát metric và bộ điều khiển Rollout nâng cao theo đúng tinh thần GitOps.
* **Công việc đã làm**:
  - Tạo cấu hình Application Helm Chart cho `kube-prometheus-stack` (định nghĩa metric scraping) và `argo-rollouts` trong thư mục `argocd/apps/`.
  - Đẩy lên Git để Root App-of-Apps tự động đồng bộ và cài đặt tài nguyên lên cụm.

#### [Lab 2] Tự viết App Flask & Đóng gói Docker Image
* **Mục tiêu**: Tạo ứng dụng kiểm thử thực tế có xuất thống kê metrics của Prometheus.
* **Công việc đã làm**:
  - Viết code Flask Python (`app/app.py`) tích hợp thư viện `prometheus-flask-exporter` xuất metric tại endpoint `/metrics`.
  - Viết `app/Dockerfile` đóng gói ứng dụng.
  - Chạy `docker build -t w9-api:1 app/` và nạp trực tiếp vào cụm qua lệnh `minikube image load`.

#### [Lab 3] Triển khai Rollout Canary & Khớp Prometheus Metrics
* **Mục tiêu**: Khởi chạy ứng dụng với chiến lược Canary và liên kết dữ liệu với Prometheus.
* **Công việc đã làm**:
  - Khai báo tài nguyên `Rollout` tại [api.yaml](k8s-api/api.yaml) với các bước chia tỷ lệ: 25% → Tạm dừng vô hạn → 50% (30 giây) → 100%.
  - Tạo [servicemonitor.yaml](k8s-api/servicemonitor.yaml) để chỉ định Prometheus cào metric `/metrics` từ cổng `http` của Service.
  - Tạo pod `load` giả lập vòng lặp gọi API liên tục.
  - Xác nhận Prometheus nhận diện được Target `api` và thu thập dữ liệu thành công.

#### [Lab 4] Điều khiển Canary Rollout thủ công (Promote)
* **Mục tiêu**: Điều khiển quy trình chuyển đổi phiên bản an toàn.
* **Công việc đã làm**:
  - Nâng cấp `VERSION` lên `"v2"` trong `api.yaml` và đẩy lên Git.
  - Hệ thống tự động bắt đầu triển khai Canary và dừng lại ở 25% (1 pod v2, 3 pod v1).
  - Sử dụng CLI `kubectl-argo-rollouts` chạy lệnh `promote` để đưa tiến trình tiếp tục nâng cấp lên 50% trong 30s và kết thúc ở 100% bản v2 an toàn.

#### [Lab 5] SLO Recording Rule & Alert gửi Email
* **Mục tiêu**: Thiết lập đo lường chất lượng dịch vụ (SLO) và cảnh báo tự động khi chất lượng tụt giảm.
* **Công việc đã làm**:
  - Tạo [slo-alert.yaml](k8s-api/slo-alert.yaml) chứa `PrometheusRule` với:
    - **SLO Recording Rule** `api:success_rate:5m`: Tính tỷ lệ thành công (non-5xx) trên cửa sổ 5 phút.
    - **Alert Rule** `ApiHighErrorRate`: Kích hoạt khi tỷ lệ thành công < 95% liên tục trong 1 phút.
  - Cấu hình Alertmanager trong [kube-prometheus-stack.yaml](argocd/apps/kube-prometheus-stack.yaml) gửi email tới `nguyentrieu080604@gmail.com` qua SMTP Gmail.
  - Thêm `ruleSelector: {}` vào Prometheus để nhận diện Rule từ mọi namespace (không chỉ Helm-managed).

* **Giải thích Query SLO:**
  ```promql
  sum(rate(flask_http_request_total{status!~"5..",namespace="demo",service="api"}[5m]))
  /
  sum(rate(flask_http_request_total{namespace="demo",service="api"}[5m]))
  ```
  - `flask_http_request_total` — metric Flask Exporter tự sinh, đếm tổng HTTP request.
  - `status!~"5.."` — lọc request **không phải** lỗi 5xx (tức thành công).
  - `rate(...[5m])` — tốc độ request trung bình 5 phút gần nhất.
  - **Kết quả** = `tổng_thành_công / tổng_request` = **tỷ lệ thành công**.
  - **Ngưỡng SLO**: ≥ 95%. Dưới 95% liên tục 1 phút → fire alert `ApiHighErrorRate`.

#### [Lab 6] AnalysisTemplate — Canary Tự động Đo lường & Tự Abort
* **Mục tiêu**: Thay thế bước `pause: {}` chờ thủ công bằng phân tích metric tự động.
* **Công việc đã làm**:
  - Tạo [analysis-template.yaml](k8s-api/analysis-template.yaml) định nghĩa `AnalysisTemplate` tên `success-rate`:
    - Truy vấn Prometheus mỗi 30 giây, đo tỷ lệ thành công trong cửa sổ 2 phút.
    - `successCondition: result[0] >= 0.95` — ≥ 95% thì PASS.
    - `failureLimit: 3` — cho phép tối đa 3 lần thất bại trước khi **tự động abort**.
  - Cập nhật [api.yaml](k8s-api/api.yaml) — bước Canary 25% giờ sử dụng `analysis` thay vì `pause`:
    ```
    25% → analysis (đo metric) → 50% → pause 30s → 100%
    ```
  - **Bản tốt** (ERROR_RATE=0): Analysis PASS → tự động lên 100%.
  - **Bản lỗi** (ERROR_RATE > 0.05): Analysis FAIL → tự động abort về bản cũ.

* **Giải thích Query AnalysisTemplate:**
  ```promql
  sum(rate(flask_http_request_total{status!~"5..",namespace="demo",service="api"}[2m]))
  /
  sum(rate(flask_http_request_total{namespace="demo",service="api"}[2m]))
  ```
  - Tương tự SLO nhưng dùng cửa sổ **2 phút** (ngắn hơn) để phản ứng nhanh trong quá trình Canary.
  - **Ngưỡng**: ≥ 95% thì PASS. Nếu 3 lần liên tiếp < 95% → **abort canary**.

#### [Lab 7] Kịch bản Demo: Canary Auto-Abort & Git Revert Rollback
* **Mục tiêu**: Chứng minh toàn bộ pipeline hoạt động end-to-end.
* **Kịch bản 1 — Bản tốt lên 100%:**
  1. Đổi `VERSION: "v3"`, giữ `ERROR_RATE: "0"` trong `api.yaml`.
  2. `git add . && git commit -m "release v3" && git push`
  3. ArgoCD sync → Rollout: 25% → Analysis PASS → 50% → 100%. ✅
* **Kịch bản 2 — Bản lỗi tự abort:**
  1. Đổi `VERSION: "v4"`, `ERROR_RATE: "0.5"` (50% lỗi) trong `api.yaml`.
  2. `git add . && git commit -m "release v4 bad" && git push`
  3. ArgoCD sync → Rollout: 25% → Analysis đo metric → tỷ lệ thành công ~50% < 95% → **FAIL x3 → tự động abort** về v3. ✅
  4. Alert `ApiHighErrorRate` fire → email gửi tới `nguyentrieu080604@gmail.com`. ✅
* **Kịch bản 3 — Git revert rollback < 5 phút:**
  1. `git revert HEAD && git push`
  2. ArgoCD sync → cụm trở về cấu hình trước (v3, ERROR_RATE=0). ✅

---

## 📂 Ý nghĩa các tệp tin cấu hình trong dự án

| Tên File | Đường dẫn | Ý nghĩa & Vai trò |
| :--- | :--- | :--- |
| **kube-prometheus-stack.yaml** | `argocd/apps/` | Khai báo ứng dụng Prometheus Operator Helm Chart + cấu hình Alertmanager gửi email. |
| **argo-rollouts.yaml** | `argocd/apps/` | Khai báo ứng dụng Argo Rollouts Controller điều khiển chiến lược Canary. |
| **api.yaml** (ArgoCD App) | `argocd/apps/` | Khai báo Application của ArgoCD cho ứng dụng Flask API. |
| **app.py** | `app/` | Mã nguồn ứng dụng API Flask tích hợp Prometheus Exporter xuất metrics. |
| **Dockerfile** | `app/` | Cấu hình đóng gói ứng dụng API Flask thành Docker Image. |
| **api.yaml** (Rollout) | `k8s-api/` | Rollout Canary của API với AnalysisTemplate tự động + Service mở cổng 8080. |
| **analysis-template.yaml** | `k8s-api/` | AnalysisTemplate `success-rate` — đo tỷ lệ thành công từ Prometheus, quyết định promote/abort. |
| **slo-alert.yaml** | `k8s-api/` | PrometheusRule chứa SLO recording rule (95% success) + Alert gửi email khi vi phạm. |
| **servicemonitor.yaml** | `k8s-api/` | Quy tắc cho Prometheus tự động cào metric `/metrics` từ Service `api`. |

---

## ⚙️ Tổng hợp toàn bộ lệnh từ đầu đến cuối

### 🔧 Phase 1: Nền tảng

```bash
# ── [Lab 0] Tạo cụm Kubernetes local ──
minikube start -p w9

# ── [Lab 1] Cài ArgoCD ──
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side

# Lấy mật khẩu admin ArgoCD
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Mở ArgoCD Dashboard (truy cập https://localhost:8080)
kubectl -n argocd port-forward svc/argocd-server 8080:443

# ── [Lab 2] Áp dụng Root App (App-of-Apps) ──
kubectl apply -f argocd/root.yaml
```

### 🚀 Phase 2: Observability & Canary

```bash
# ── [Lab 2] Build & load Docker image API ──
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9

# ── [Lab 3] Tạo pod traffic giả lập (chạy 1 lần) ──
kubectl -n demo run load --image=busybox --restart=Never -- sh -c "while true; do wget -qO- api:8080/; done"

# ── Port-forward các dịch vụ giám sát ──
# Prometheus UI (http://localhost:9090)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090

# Grafana UI (http://localhost:3000, admin/prom-operator)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000

# Alertmanager UI (http://localhost:9093)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093

# ── Kiểm tra trạng thái Rollout (real-time) ──
.\kubectl-argo-rollouts.exe get rollout api -n demo -w

# ── Kiểm tra SLO query trên Prometheus ──
# Truy cập http://localhost:9090 → nhập query:
#   api:success_rate:5m
# Phải trả về giá trị gần 1.0 (100% thành công)

# ── Kiểm tra Alert trên Alertmanager ──
# Truy cập http://localhost:9093 → xem alert ApiHighErrorRate
```

### 🧪 Demo Kịch bản (Lab 7)

```bash
# ── Kịch bản 1: Deploy bản tốt (v3) — tự động lên 100% ──
# Sửa api.yaml: VERSION="v3", ERROR_RATE="0"
git add . && git commit -m "release v3 - good version" && git push
# Chờ ArgoCD sync → Xem rollout: 25% → Analysis PASS → 50% → 100% ✅

# ── Kịch bản 2: Deploy bản lỗi (v4) — tự động abort ──
# Sửa api.yaml: VERSION="v4", ERROR_RATE="0.5"
git add . && git commit -m "release v4 - inject 50% errors" && git push
# Chờ ArgoCD sync → Xem rollout: 25% → Analysis FAIL → AUTO ABORT ✅
# Kiểm tra email → nhận alert ApiHighErrorRate ✅

# ── Kịch bản 3: Git revert rollback < 5 phút ──
git revert HEAD
git push
# ArgoCD sync → cụm trở về bản v3 an toàn ✅
```

### 🔍 Các lệnh kiểm tra hữu ích

```bash
# Xem trạng thái tất cả pods
kubectl -n demo get pods

# Xem chi tiết rollout
.\kubectl-argo-rollouts.exe get rollout api -n demo

# Xem AnalysisRun (kết quả phân tích canary)
kubectl -n demo get analysisrun

# Xem chi tiết AnalysisRun mới nhất
kubectl -n demo describe analysisrun -l rollouts-pod-template-hash

# Xem PrometheusRule đã được nhận chưa
kubectl -n demo get prometheusrule

# Xem trạng thái alert
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093
# Truy cập http://localhost:9093

# Xem log Argo Rollouts controller
kubectl -n argo-rollouts logs -l app.kubernetes.io/name=argo-rollouts
```
