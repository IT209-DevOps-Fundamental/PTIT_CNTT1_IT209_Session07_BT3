# Bài 3: Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Khởi tạo cơ sở dữ liệu MySQL và cấu hình phân quyền
Đăng nhập vào MySQL với quyền root:
```bash
sudo mysql -u root -p
```

Thực thi các lệnh SQL sau để tạo database và user:
```sql
-- Tạo database
CREATE DATABASE springboot_db;

-- Tạo user và thiết lập mật khẩu
CREATE USER 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';

-- Cấp toàn quyền trên database springboot_db cho user vừa tạo
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';

-- Áp dụng các thay đổi
FLUSH PRIVILEGES;
```
> **Lưu ý**: Tùy thuộc vào cấu hình, nếu Spring Boot chạy trên môi trường/container khác, có thể cần đổi `'localhost'` thành `'%'`.

## 2. Tạo user hệ thống Linux
Tạo user `spring-runner` không có shell login để tăng tính bảo mật (non-root):
```bash
sudo useradd -r -M -s /usr/sbin/nologin spring-runner
```
*(Giải thích: `-r` tạo system user, `-M` không tạo thư mục home, `-s /usr/sbin/nologin` ngăn user này login qua SSH hoặc Terminal)*

Đảm bảo user này có quyền thực thi ứng dụng:
```bash
sudo chown spring-runner:spring-runner /opt/spring-app/app.jar
sudo chmod 500 /opt/spring-app/app.jar
```

## 3. Cấu hình dịch vụ Systemd
Tạo file cấu hình tại `/etc/systemd/system/spring-app.service` với nội dung như trong file `spring-app.service` đính kèm:
```ini
[Unit]
Description=Spring Boot Application Service
After=syslog.target network.target mysql.service

[Service]
User=spring-runner
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Sau khi tạo xong file, thực hiện các lệnh sau để nạp lại daemon và khởi động dịch vụ:
```bash
# Nạp lại cấu hình systemd
sudo systemctl daemon-reload

# Thiết lập cho dịch vụ chạy cùng hệ thống khi khởi động
sudo systemctl enable spring-app.service

# Khởi động dịch vụ
sudo systemctl start spring-app.service
```

## 4. Kết quả kiểm tra (Báo cáo)

### Kiểm tra trạng thái dịch vụ
**Lệnh:**
```bash
sudo systemctl status spring-app.service
```
**Kết quả hiển thị (Mẫu):**
Dịch vụ hiển thị trạng thái `active (running)`.
```text
● spring-app.service - Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-10-08 07:30:00 +07; 5min ago
   Main PID: 12345 (java)
      Tasks: 35 (limit: 4613)
     Memory: 250.5M
     CGroup: /system.slice/spring-app.service
             └─12345 /usr/bin/java -jar /opt/spring-app/app.jar
```
*(Bạn có thể thay thế khối trên bằng ảnh chụp màn hình thực tế trên máy ảo Linux của bạn)*

### Kiểm tra cổng lắng nghe
**Lệnh:**
```bash
ss -tlnp | grep 8082
```
**Kết quả hiển thị (Mẫu):**
Cổng 8082 đang được lắng nghe bởi tiến trình Java chạy dưới quyền user `spring-runner`.
```text
LISTEN  0       100              *:8082             *:*      users:(("java",pid=12345,fd=14))
```
*(Bạn có thể thay thế khối trên bằng ảnh chụp màn hình thực tế trên máy ảo Linux của bạn)*
