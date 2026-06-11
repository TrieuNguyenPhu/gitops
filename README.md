# GitOps Lab Course: From Zero to Production-Ready CI/CD with ArgoCD

Repository này chứa toàn bộ mã nguồn cấu hình Kubernetes, ArgoCD Application, và CI workflow phục vụ cho chuỗi bài thực hành (Lab 0 - Lab 7) về GitOps.

---

## 🗺️ Bản đồ các Lab đã thực hiện

### [Lab 0] Khởi tạo Cụm Local & Git Repository
* **Mục tiêu**: Chuẩn bị môi trường làm việc local và tạo kho chứa mã nguồn làm "Source of Truth".
* **Công việc đã làm**:
  - Khởi tạo cụm Kubernetes local bằng Minikube tên là `w9`.
  - Tạo cấu hình triển khai ứng dụng cơ bản (`web` deployment chạy image `nginx:1.27`).
  - Thiết lập Git repository local và đẩy lên GitHub làm kho chứa chính thức.

### [Lab 1] Cài đặt ArgoCD vào Cụm Kubernetes
* **Mục tiêu**: Đưa "người thợ" ArgoCD vào cụm để bắt đầu quản lý trạng thái tài nguyên.
* **Công việc đã làm**:
  - Tạo namespace chuyên biệt `argocd`.
  - Triển khai bộ tài nguyên ArgoCD chính thức bằng chế độ `--server-side` (để tránh lỗi vượt quá kích thước dữ liệu annotation cho phép do các tệp CRD của ArgoCD rất lớn).
  - Lấy thông tin tài khoản quản trị viên ban đầu và mở cổng truy cập dashboard local (port-forward).

### [Lab 2] Khởi tạo Application trong ArgoCD
* **Mục tiêu**: Kết nối ArgoCD với Git Repo và cấu hình tự động đồng bộ (Auto Sync) phiên bản đầu tiên của ứng dụng.
* **Công việc đã làm**:
  - Viết file tài nguyên `Application` trỏ tới đường dẫn thư mục `k8s/` chứa manifest ứng dụng.
  - Áp dụng cấu hình lên cụm, ArgoCD tự động phát hiện mã nguồn và triển khai 2 bản sao (replicas) ứng dụng Nginx vào namespace `demo`.

### [Lab 3] Cơ chế Tự động Đồng bộ (Auto Sync) & Tự phục hồi (Self-Heal)
* **Mục tiêu**: Thử nghiệm hai tính năng cốt lõi nhất của triết lý GitOps: Khớp trạng thái tự động và phục hồi lỗi sai khác.
* **Công việc đã làm**:
  - **Auto Sync**: Sửa số lượng replicas từ `2` thành `4` trong Git $\rightarrow$ ArgoCD tự động kéo (pull) thay đổi mới nhất và tăng số pod trên cụm mà không cần can thiệp thủ công.
  - **Self-Heal**: Cố tình can thiệp thủ công vào cụm bằng lệnh scale lên `9` replicas $\rightarrow$ ArgoCD ngay lập tức phát hiện sự sai lệch giữa trạng thái thực tế (cụm) với nguồn sự thật (Git) và tự động kéo giảm về đúng `4` replicas.

### [Lab 4] Rollback an toàn thông qua Git Revert
* **Mục tiêu**: Thực hiện khôi phục phiên bản cũ theo đúng chuẩn GitOps.
* **Công việc đã làm**:
  - Thay vì dùng lệnh `kubectl rollout undo` (lệnh này sẽ bị tính năng Self-heal của ArgoCD ghi đè lại vì Git vẫn giữ phiên bản mới), chúng ta thực hiện rollback thông qua việc tạo commit đảo ngược trạng thái trên Git (`git revert HEAD`).
  - Khi thay đổi được đẩy lên GitHub, cụm tự động được đưa về trạng thái trước đó (2 replicas) một cách tường minh và có lưu lại lịch sử.

### [Lab 5] Kiến trúc App-of-Apps
* **Mục tiêu**: Loại bỏ việc phải chạy lệnh `kubectl apply` thủ công mỗi khi có ứng dụng mới. Chỉ cần quản lý khai báo ứng dụng qua Git.
* **Công việc đã làm**:
  - Thiết lập cấu hình ứng dụng Root (`root.yaml`) quản lý toàn bộ các file khai báo ứng dụng con nằm trong thư mục `argocd/apps/`.
  - Đăng ký ứng dụng Root lên ArgoCD lần duy nhất. Kể từ đây, mỗi khi muốn thêm, bớt hoặc sửa cấu hình ứng dụng, chúng ta chỉ cần thả tệp YAML vào thư mục `argocd/apps/` và đẩy lên Git.

### [Lab 6] Quản lý thứ tự triển khai tài nguyên (Sync Waves)
* **Mục tiêu**: Giải quyết bài toán phụ thuộc tài nguyên (ví dụ: Deployment chỉ được phép chạy khi Namespace và ConfigMap chứa cấu hình của nó đã tồn tại sẵn sàng).
* **Công việc đã làm**:
  - Sử dụng annotation `argocd.argoproj.io/sync-wave` để đánh số thứ tự triển khai từ nhỏ đến lớn (số nhỏ chạy trước).
  - Phân chia tài nguyên cụ thể:
    - **Namespace** `demo`: Wave `-1` (Tạo trước tiên).
    - **ConfigMap** `web-config`: Wave `0` (Tạo cấu hình).
    - **Deployment** `web` (có đọc cấu hình từ ConfigMap): Wave `1` (Triển khai ứng dụng).
    - **Service** `web`: Wave `2` (Mở cổng kết nối dịch vụ).

### [Lab 7] Tích hợp CI (Plan-on-PR) & Branch Protection
* **Mục tiêu**: Ngăn chặn lỗi cú pháp hoặc cấu hình sai cấu trúc được merge vào nhánh chính làm hỏng hệ thống.
* **Công việc đã làm**:
  - Xây dựng luồng công việc CI bằng GitHub Actions sử dụng công cụ `kubeconform` để kiểm tra tính hợp lệ về cấu trúc (schema validation) của toàn bộ file YAML trong thư mục `k8s/` mỗi khi có Pull Request.
  - Cấu hình Branch Protection trên GitHub buộc các PR phải qua bước chạy thử CI thành công (xanh) mới được phép Merge vào nhánh `main`.

---

## 📂 Ý nghĩa các tệp tin cấu hình trong dự án

| Tên File | Đường dẫn | Ý nghĩa & Vai trò |
| :--- | :--- | :--- |
| **namespace.yaml** | [k8s/namespace.yaml](file:///d:/source%20code/gitops/k8s/namespace.yaml) | Khai báo namespace `demo`. Được đánh số wave `-1` để đảm bảo namespace luôn được tạo ra đầu tiên trước mọi tài nguyên khác. |
| **web.yaml** | [k8s/web.yaml](file:///d:/source%20code/gitops/k8s/web.yaml) | Chứa bộ ba tài nguyên cốt lõi của ứng dụng Web: ConfigMap (chứa thông tin cấu hình), Deployment (chạy ứng dụng Nginx liên kết với ConfigMap) và Service (điều phối traffic truy cập). |
| **web.yaml (App)** | [argocd/apps/web.yaml](file:///d:/source%20code/gitops/argocd/apps/web.yaml) | File cấu hình tài nguyên `Application` của ArgoCD cho ứng dụng `web`, định nghĩa nguồn tải mã nguồn (Git Repo) và đích đến để đồng bộ (cụm K8s local). |
| **root.yaml** | [argocd/root.yaml](file:///d:/source%20code/gitops/argocd/root.yaml) | File khai báo ứng dụng Root cho mô hình **App-of-Apps**, có nhiệm vụ theo dõi toàn bộ thư mục `argocd/apps/` để tự động kích hoạt các ứng dụng con. |
| **validate.yml** | [.github/workflows/validate.yml](file:///d:/source%20code/gitops/.github/workflows/validate.yml) | Kịch bản chạy tự động của GitHub Actions (CI) giúp thực hiện tải và quét lỗi schema của các file cấu hình bằng công cụ `kubeconform`. |

---

## ⚙️ Ý nghĩa các câu lệnh cốt lõi đã sử dụng

### 1. Quản lý cụm Kubernetes (Minikube & Kubectl)
* `minikube start -p w9 --driver=docker`
  - *Ý nghĩa*: Khởi chạy một cụm Kubernetes ảo local có tên là `w9`, chạy trực tiếp trên nền tảng Docker Container của máy vật lý.
* `kubectl create ns argocd`
  - *Ý nghĩa*: Khởi tạo một không gian tên (namespace) có tên `argocd` để gom nhóm tất cả các tài nguyên liên quan đến hệ thống ArgoCD.
* `kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
  - *Ý nghĩa*: Thực hiện tải và áp dụng toàn bộ cấu hình cài đặt của ArgoCD. Chế độ `--server-side` giúp gửi trực tiếp dữ liệu lớn lên API server xử lý, tránh lỗi vượt quá dung lượng metadata phía client.
* `kubectl -n argocd rollout status deploy/argocd-server`
  - *Ý nghĩa*: Theo dõi tiến trình khởi tạo và khởi động của server điều khiển ArgoCD, lệnh sẽ khóa màn hình cho đến khi server ở trạng thái sẵn sàng (`Running`).
* `kubectl -n argocd port-forward svc/argocd-server 8080:443`
  - *Ý nghĩa*: Ánh xạ cổng dịch vụ `443` (HTTPS) của ArgoCD Server trong cụm ra cổng `8080` ở máy vật lý local, cho phép người dùng mở trình duyệt truy cập dashboard.
* `kubectl -n demo scale deploy/web --replicas=9`
  - *Ý nghĩa*: Ép số lượng bản sao pod của deployment `web` chạy trên cụm thành 9. Lệnh này dùng để giả lập lỗi cấu hình thủ công nhằm kiểm chứng khả năng tự sửa lỗi (Self-heal) của ArgoCD.

### 2. Các lệnh bảo mật và lấy thông tin
* `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"`
  - *Ý nghĩa*: Truy vấn và trích xuất chuỗi mật khẩu mã hóa dạng Base64 của tài khoản `admin` được sinh ra tự động khi cài đặt ArgoCD.

### 3. Quy trình làm việc với Git
* `git commit -am "nội dung"`
  - *Ý nghĩa*: Thực hiện nhanh việc thêm các thay đổi của các file đã được tracking vào vùng staging và tạo một commit mới.
* `git revert HEAD --no-edit`
  - *Ý nghĩa*: Tạo tự động một commit mới có nội dung trái ngược hoàn toàn với commit hiện tại ở vị trí đầu (`HEAD`) để khôi phục mã nguồn về trạng thái cũ mà vẫn lưu lại dấu vết lịch sử phiên bản.
