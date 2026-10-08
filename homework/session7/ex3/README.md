## 1. Thông tin cấu hình
| Thành phần      | Giá trị                   |
| --------------- | ------------------------- |
| Database        | `springboot_db`           |
| MySQL User      | `spring-admin`            |
| MySQL Password  | `SpringSecure@123`        |
| Linux User      | `spring-runner`           |
| Shell           | `/usr/sbin/nologin`       |
| Application     | `/opt/spring-app/app.jar` |
| Java            | Java 21                   |
| Spring Boot     | 4.1.1                     |
| Port            | 8082                      |
| Systemd Service | `spring-app.service`      |

## 3. 2ạo user Linux

User `spring-runner` được tạo để chạy ứng dụng Spring Boot và không cho phép đăng nhập shell.

Kiểm tra:

```bash
getent passwd spring-runner
```

Kết quả:

```text
spring-runner:x:998:998::/home/spring-runner:/usr/sbin/nologin
```

Điều này cho thấy user sử dụng shell:

```text
/usr/sbin/nologin
```

nên không thể đăng nhập trực tiếp vào hệ thống bằng shell.

---

## 3. Chuẩn bị ứng dụng

Ứng dụng Spring Boot được đặt tại:

```text
/opt/spring-app/app.jar
```

Kiểm tra:

```bash
ls -lh /opt/spring-app/app.jar
```

Kết quả thực tế:

```text
-rw-r--r-- 1 spring-runner spring-runner 19M Oct  8 03:38 /opt/spring-app/app.jar
```

JAR thuộc quyền sở hữu của:

```text
spring-runner:spring-runner
```

---

## 4. Cấu hình MySQL

Tạo database:

```sql
CREATE DATABASE IF NOT EXISTS springboot_db;
```

Tạo user:

```sql
CREATE USER IF NOT EXISTS 'spring-admin'@'localhost'
IDENTIFIED BY 'SpringSecure@123';
```

Cấp toàn quyền trên database:

```sql
GRANT ALL PRIVILEGES ON springboot_db.* 
TO 'spring-admin'@'localhost';
```

Áp dụng quyền:

```sql
FLUSH PRIVILEGES;
```

---

## 5. Cấu hình Systemd

Tệp service:

```text
/etc/systemd/system/spring-app.service
```

Nội dung:

```ini
[Unit]
Description=Spring Boot Application
After=network.target mysql.service

[Service]
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar --server.port=8082

Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Giải thích

* `User=spring-runner`: ứng dụng không chạy bằng root.
* `Group=spring-runner`: sử dụng group của user chạy ứng dụng.
* `WorkingDirectory=/opt/spring-app`: thư mục làm việc của ứng dụng.
* `ExecStart`: lệnh khởi chạy Spring Boot.
* `--server.port=8082`: yêu cầu ứng dụng lắng nghe port `8082`.
* `Restart=on-failure`: tự khởi động lại khi ứng dụng bị lỗi.
* `RestartSec=10`: chờ 10 giây trước khi restart.
* `WantedBy=multi-user.target`: service được khởi động cùng hệ thống.

---

## 6. Nạp và khởi động Systemd

Sau khi tạo service:

```bash
sudo systemctl daemon-reload
```

Cho phép service tự khởi động cùng hệ thống:

```bash
sudo systemctl enable spring-app.service
```

Khởi động service:

```bash
sudo systemctl start spring-app.service
```

---

## 7. Kiểm tra trạng thái Systemd

Lệnh:

```bash
sudo systemctl status spring-app.service
```

Kết quả thực tế:

```text
● spring-app.service - Spring Boot Application
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; vendor preset: enabled)
     Active: active (running)
     Main PID: 29826 (java)
     CGroup: /system.slice/spring-app.service
             └─29826 /usr/bin/java -jar /opt/spring-app/app.jar --server.port=8082
```

Kết quả:

```text
Active: active (running)
```

cho thấy service đang hoạt động bình thường.

---

## 8. Kiểm tra Spring Boot

Log của Systemd cho thấy:

```text
Spring Boot :: (v4.1.1)
```

Ứng dụng sử dụng:

```text
Java 21.0.12.1
```

Tomcat được khởi động:

```text
Tomcat started on port 8082 (http) with context path '/'
```

Ứng dụng khởi động thành công:

```text
Started DemoApplication
```

---

## 9. Kiểm tra port 8082

Lệnh:

```bash
ss -tlnp | grep 8082
```

Kết quả thực tế:

```text
LISTEN 0      100                *:8082             *:*    users:(("java",pid=29826,fd=9))
```

Kết quả cho thấy tiến trình Java đang lắng nghe trên:

```text
*:8082
```

PID của ứng dụng:

```text
29826
```

---

## 10. Kiểm tra user chạy ứng dụng

Kiểm tra tiến trình:

```bash
ps -o user,group,pid,cmd -p 29826
```

Kết quả mong đợi:

```text
USER           GROUP          PID CMD
spring-runner  spring-runner  29826 /usr/bin/java -jar /opt/spring-app/app.jar --server.port=8082
```

Qua đó xác nhận ứng dụng chạy bằng user:

```text
spring-runner
```

thay vì `root`.