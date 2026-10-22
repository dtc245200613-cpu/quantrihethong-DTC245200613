# Quản trị hệ thống - Đề 3 (DTC245200613)

Hệ thống WordPress chạy bằng Docker Compose, gồm Nginx (HTTPS), WordPress, MySQL và phpMyAdmin.

## Thông tin
- MSSV: DTC245200613
- Ubuntu: 26.04.1 LTS (ghi chú: đề yêu cầu 24.04, cần xác nhận với thầy)
- Tên miền: wp.dtc245200613.lab và pma.dtc245200613.lab

## Cấu trúc
- docker-compose.yml: khai báo 4 dịch vụ db, wordpress, phpmyadmin, nginx
- nginx/conf.d/lab.conf: chuyển HTTP sang HTTPS, proxy tới WordPress và phpMyAdmin
- .env.example: mẫu biến môi trường (không chứa mật khẩu thật)
- .env: chứa mật khẩu thật, KHÔNG commit

## Cách chạy
1. cp .env.example .env, sau đó điền mật khẩu thật vào .env
2. Tạo chứng chỉ tự ký trong nginx/certs/ (lab.crt, lab.key)
3. Thêm hai dòng vào file hosts của Windows, trỏ tới IP của VM
4. docker compose up -d

## Mạng
- frontend: nginx
- backend (internal): db, wordpress, phpmyadmin, không publish cổng ra ngoài
- Chỉ nginx publish cổng 80 và 443

## Trạng thái
- Mốc 1: Nginx + WordPress + phpMyAdmin + MySQL (hoàn thành)
- Mốc 2: Prometheus + Grafana + exporter (hoàn thành)
- Mốc 3: Loki + Promtail + LogQL + hardening (hoàn thành)

## Giám sát (Mốc 2)
- Các file: docker-compose.monitor.yml, monitoring/prometheus.yml, monitoring/grafana/provisioning
- Thành phần: Prometheus, Grafana, node-exporter (máy ảo), cAdvisor (container), mysqld-exporter (MySQL)
- Grafana và Prometheus chỉ mở trên 127.0.0.1 (truy cập bằng localhost trong VM)
- Dashboard nhập sẵn: Node Exporter Full (1860), cAdvisor (14282), MySQL Overview (7362)
- Tài khoản exporter MySQL chỉ có quyền đọc số liệu. File monitoring/my.cnf được tạo từ .env và không được commit.

## Hạn chế đã biết (Mốc 2)
- Dashboard cAdvisor không có số liệu theo container. Nguyên nhân: Docker dùng storage driver overlayfs trên máy ảo, khiến cAdvisor không lấy được tên container. Kiểm tra: docker exec cadvisor wget -qO- http://127.0.0.1:8080/metrics | grep -c 'name="' trả về 0.
- Cách khắc phục là đổi sang overlay2 và tái tạo dữ liệu Docker. Chưa thực hiện để giữ dữ liệu WordPress.
- Node Exporter và MySQL Overview hoạt động bình thường.

## Log tập trung và LogQL (Mốc 3)
- Loki lưu log. Promtail đọc log qua Docker socket, gắn nhãn container và muc (mức độ lỗi).
- Truy vấn mẫu:
  - {container="wp-dtc245200613-wordpress-1"}
  - {container="wp-dtc245200613-nginx-1"} |= "GET"
  - {container=~"wp-dtc245200613-.*"} |= "error"
  - sum by (container) (count_over_time({container=~"wp-dtc245200613-.*", muc=~"error|warn|fatal|critical"}[5m]))

## Hardening (Mốc 3)
- Nginx: server_tokens off, X-Frame-Options, X-Content-Type-Options, Referrer-Policy (nginx/conf.d/security.conf).
- Container giám sát: security_opt no-new-privileges.
- Mật khẩu ngẫu nhiên cho MySQL, exporter và Grafana, lưu trong .env (không commit).
- Grafana và Prometheus chỉ publish trên 127.0.0.1.
- Tài khoản exporter MySQL chỉ có quyền PROCESS, REPLICATION CLIENT, SELECT.

## Hạn chế đã biết
- Promtail đọc Docker socket (quyền cao, chỉ đọc). Cần thay bằng cách khác trong môi trường thật.
- monitoring/my.cnf có quyền 644 để exporter đọc được (chứa mật khẩu exporter). Cần chuyển sang secret riêng.
- cAdvisor không có số liệu theo container do overlayfs (xem Mốc 2).
- Ubuntu 26.04.1 LTS thay vì 24.04 như đề yêu cầu.
