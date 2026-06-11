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
  - Khai báo tài nguyên `Rollout` tại [api.yaml](file:///d:/source%20code/gitops/k8s-api/api.yaml) với các bước chia tỷ lệ: 25% $\rightarrow$ Tạm dừng vô hạn $\rightarrow$ 50% (30 giây) $\rightarrow$ 100%.
  - Tạo [servicemonitor.yaml](file:///d:/source%20code/gitops/k8s-api/servicemonitor.yaml) để chỉ định Prometheus cào metric `/metrics` từ cổng `http` của Service.
  - Tạo pod `load` giả lập vòng lặp gọi API liên tục.
  - Xác nhận Prometheus nhận diện được Target `api` và thu thập dữ liệu thành công.

#### [Lab 4] Điều khiển Canary Rollout thủ công (Promote)
* **Mục tiêu**: Điều khiển quy trình chuyển đổi phiên bản an toàn.
* **Công việc đã làm**:
  - Nâng cấp `VERSION` lên `"v2"` trong `api.yaml` và đẩy lên Git.
  - Hệ thống tự động bắt đầu triển khai Canary và dừng lại ở 25% (1 pod v2, 3 pod v1).
  - Sử dụng CLI `kubectl-argo-rollouts` chạy lệnh `promote` để đưa tiến trình tiếp tục nâng cấp lên 50% trong 30s và kết thúc ở 100% bản v2 an toàn.

---

## 📂 Ý nghĩa các tệp tin cấu hình trong dự án

| Tên File | Đường dẫn | Ý nghĩa & Vai trò |
| :--- | :--- | :--- |
| **kube-prometheus-stack.yaml** | [kube-prometheus-stack.yaml](file:///d:/source%20code/gitops/argocd/apps/kube-prometheus-stack.yaml) | Khai báo ứng dụng Prometheus Operator Helm Chart phục vụ giám sát và lưu trữ metric. |
| **argo-rollouts.yaml** | [argo-rollouts.yaml](file:///d:/source%20code/gitops/argocd/apps/argo-rollouts.yaml) | Khai báo ứng dụng Argo Rollouts Controller điều khiển các chiến lược deploy Canary/Blue-Green. |
| **app.py** | [app.py](file:///d:/source%20code/gitops/app/app.py) | Mã nguồn ứng dụng API Flask viết bằng Python tích hợp sẵn Prometheus Exporter xuất metrics. |
| **Dockerfile** | [Dockerfile](file:///d:/source%20code/gitops/app/Dockerfile) | Cấu hình đóng gói ứng dụng API Flask thành Docker Image. |
| **api.yaml** | [api.yaml](file:///d:/source%20code/gitops/k8s-api/api.yaml) | Chứa khai báo Rollout Canary của API ( replicas: 4, chiến lược dừng ở 25%) và Service `api` mở cổng `http` 8080. |
| **servicemonitor.yaml** | [servicemonitor.yaml](file:///d:/source%20code/gitops/k8s-api/servicemonitor.yaml) | Tài nguyên định nghĩa quy tắc cho Prometheus tự động cào metric từ Service `api`. |
| **api.yaml (App)** | [api.yaml](file:///d:/source%20code/gitops/argocd/apps/api.yaml) | Khai báo Application của ArgoCD cho ứng dụng Flask API. |

---

## ⚙️ Các lệnh quan trọng nhất ở Phase 2

* `docker build -t w9-api:1 app/`
  - *Ý nghĩa*: Đóng gói ứng dụng Flask API thành Docker image local với tag `w9-api:1`.
* `minikube image load w9-api:1 -p w9`
  - *Ý nghĩa*: Nạp trực tiếp Docker image local vừa build vào bên trong cụm ảo của Minikube `w9` mà không cần đẩy lên Docker Hub/Registry trung gian.
* `kubectl -n demo run load --image=busybox --restart=Never -- sh -c "while true; do wget -qO- api:8080/; done"`
  - *Ý nghĩa*: Chạy một pod tạm tên `load` liên tục thực hiện gọi API tới Service `api:8080` để giả lập lưu lượng truy cập thực tế (traffic) nhằm sinh ra các chỉ số metrics.
* `kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090`
  - *Ý nghĩa*: Ánh xạ cổng dịch vụ của Prometheus Server ra cổng 9090 máy local để xem đồ thị truy vấn metric.
* `.\kubectl-argo-rollouts.exe promote api -n demo`
  - *Ý nghĩa*: Phát lệnh tiếp tục quy trình chuyển giao (promotion) cho ứng dụng `api` khi đang bị tạm dừng (Paused) ở bước Canary, đưa phiên bản mới triển khai hoàn tất 100%.
