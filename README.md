# B4 - Nginx Reverse Proxy cho Spring Boot

## 1. Thông tin sinh viên

* Họ tên: **La Thế Phong**
* Mã lớp: **HN-KS24-CNTT1**
* Bài: **Session07 - B4**
* Nội dung: **Cấu hình Nginx Reverse Proxy cho ứng dụng Spring Boot**

---

## 2. Mục tiêu

Thực hiện cấu hình Nginx làm Reverse Proxy cho ứng dụng Spring Boot.

Mô hình:

```text
Client
   |
   | HTTP :80
   v
 Nginx
   |
   | /              -> Static HTML
   |
   | /api/          -> Spring Boot :8082
   v
Spring Boot
   |
   | /actuator/health
   v
  Health UP
```

Các yêu cầu chính:

1. Nginx phục vụ trang HTML tĩnh.
2. Trang HTML hiển thị thông tin sinh viên.
3. Spring Boot chạy trên port `8082`.
4. Nginx reverse proxy request `/api/` tới Spring Boot.
5. Kiểm tra Nginx bằng `nginx -t`.
6. Kiểm tra static page.
7. Kiểm tra API health thông qua Nginx.

---

## 3. Môi trường

```text
OS: Ubuntu WSL2
Nginx: 1.28.3
Java: OpenJDK 17
Spring Boot: Project bt
Nginx Port: 80
Spring Boot Port: 8082
```

Thư mục bài:

```text
/mnt/d/IT209_Devops_Funmentals/Session07/b4
```

Thư mục Spring Boot backend:

```text
/mnt/d/IT209_Devops_Funmentals/bt
```

---

## 4. Cấu hình Spring Boot

File cấu hình:

```text
bt/src/main/resources/application.yaml
```

Spring Boot được cấu hình chạy trên port `8082`:

```yaml
server:
  port: 8082
```

Actuator expose các endpoint:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

Health endpoint của Spring Boot:

```text
http://127.0.0.1:8082/actuator/health
```

---

## 5. Build Spring Boot

Di chuyển tới thư mục backend:

```bash
cd /mnt/d/IT209_Devops_Funmentals/bt
```

Build project:

```bash
./gradlew clean build -x test
```

Build thành công:

```text
BUILD SUCCESSFUL
```

File JAR:

```text
build/libs/bt-0.0.1-SNAPSHOT.jar
```

---

## 6. Chạy Spring Boot

Chạy:

```bash
java -jar build/libs/bt-0.0.1-SNAPSHOT.jar
```

Spring Boot chạy trên:

```text
http://127.0.0.1:8082
```

Kiểm tra port:

```bash
ss -lntp | grep 8082
```

Kết quả:

```text
LISTEN 0 100 *:8082 *:* users:(("java",pid=...,fd=...))
```

Điều này xác nhận Spring Boot đang LISTEN trên port `8082`.

---

## 7. Kiểm tra Spring Boot Actuator

Chạy:

```bash
curl -i http://127.0.0.1:8082/actuator/health
```

Kết quả:

```text
HTTP/1.1 200
Content-Type: application/vnd.spring-boot.actuator.v3+json
```

Response:

```json
{
  "groups": [
    "liveness",
    "readiness"
  ],
  "status": "UP"
}
```

Như vậy Spring Boot hoạt động bình thường.

---

## 8. Tạo trang HTML tĩnh

Nginx phục vụ file:

```text
/var/www/html/index.html
```

Nội dung trang:

```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>DevOps - Reverse Proxy</title>
</head>
<body>
    <h1>Thông tin học viên</h1>

    <p><strong>Họ tên:</strong> La Thế Phong</p>
    <p><strong>Mã lớp:</strong> HN-KS24-CNTT1</p>

    <h2>Nginx Reverse Proxy</h2>
    <p>Trang web tĩnh được phục vụ trực tiếp bởi Nginx.</p>
</body>
</html>
```

---

## 9. Cấu hình Nginx Reverse Proxy

File cấu hình:

```text
/etc/nginx/sites-available/spring-proxy.conf
```

Nội dung:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8082/actuator/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Giải thích

Request:

```text
/api/health
```

được Nginx chuyển tới:

```text
http://127.0.0.1:8082/actuator/health
```

Mapping:

```text
Client
   |
   | GET /api/health
   v
Nginx :80
   |
   | proxy_pass
   v
Spring Boot :8082
   |
   | /actuator/health
   v
HTTP 200
```

---

## 10. Enable Nginx site

Tạo symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/spring-proxy.conf
```

Kiểm tra:

```bash
ls -l /etc/nginx/sites-enabled/
```

Kết quả:

```text
spring-proxy.conf -> /etc/nginx/sites-available/spring-proxy.conf
```

Không còn site `devops-hackathon` gây conflict.

---

## 11. Kiểm tra cấu hình Nginx

Chạy:

```bash
sudo nginx -t
```

Kết quả:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Sau khi kiểm tra thành công, reload Nginx:

```bash
sudo systemctl reload nginx
```

---

## 12. Kiểm tra Static Website

Chạy:

```bash
curl http://localhost/
```

Kết quả:

```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>DevOps - Reverse Proxy</title>
</head>
<body>
    <h1>Thông tin học viên</h1>

    <p><strong>Họ tên:</strong> La Thế Phong</p>
    <p><strong>Mã lớp:</strong> HN-KS24-CNTT1</p>

    <h2>Nginx Reverse Proxy</h2>
    <p>Trang web tĩnh được phục vụ trực tiếp bởi Nginx.</p>
</body>
</html>
```

Kiểm tra HTTP status:

```bash
curl -I http://localhost/
```

Kết quả mong muốn:

```text
HTTP/1.1 200 OK
```

---

## 13. Kiểm tra Reverse Proxy

### Kiểm tra trực tiếp Spring Boot

```bash
curl -i http://127.0.0.1:8082/actuator/health
```

Kết quả:

```text
HTTP/1.1 200
```

và:

```json
{
  "groups": [
    "liveness",
    "readiness"
  ],
  "status": "UP"
}
```

### Kiểm tra thông qua Nginx

```bash
curl -i http://localhost/api/health
```

Kết quả:

```text
HTTP/1.1 200
Server: nginx/1.28.3 (Ubuntu)
```

Response:

```json
{
  "groups": [
    "liveness",
    "readiness"
  ],
  "status": "UP"
}
```

Điều này chứng minh Nginx đã reverse proxy request thành công tới Spring Boot.

---

## 14. Các lỗi gặp phải và cách xử lý

### Lỗi 1: Spring Boot không chạy trên port 8082

Kiểm tra:

```bash
ss -lntp | grep 8082
```

Không có kết quả.

Nguyên nhân là Spring Boot chưa khởi động thành công do cấu hình database.

Sau khi kiểm tra MySQL, sử dụng:

```text
Database: springboot_db
Username: spring-admin
```

và cấu hình Spring Boot để kết nối tới database này.

---

### Lỗi 2: `/api/health` trả 502 Bad Gateway

Ban đầu:

```bash
curl -I http://localhost/api/health
```

trả:

```text
HTTP/1.1 502 Bad Gateway
```

Nguyên nhân:

```text
Nginx → 127.0.0.1:8082
```

nhưng Spring Boot chưa chạy.

Sau khi Spring Boot chạy trên port `8082`, lỗi 502 được giải quyết.

---

### Lỗi 3: `/api/health` trả 404

Sau khi Spring Boot chạy, request:

```text
/api/health
```

trả:

```text
HTTP/1.1 404
```

Response cho biết:

```text
"path": "/health"
```

Nguyên nhân là Nginx đang chuyển:

```text
/api/health
```

thành:

```text
/health
```

trong khi Spring Boot Actuator sử dụng:

```text
/actuator/health
```

Đã sửa `proxy_pass` thành:

```nginx
proxy_pass http://127.0.0.1:8082/actuator/;
```

Sau khi reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

kiểm tra lại:

```bash
curl -i http://localhost/api/health
```

và nhận:

```text
HTTP/1.1 200
```

---

## 15. Cấu trúc thư mục bài nộp

Thư mục:

```text
Session07/
└── b4/
    ├── spring-proxy.conf
    └── README.md
```

File cấu hình đã được copy vào thư mục bài:

```bash
sudo cp /etc/nginx/sites-available/spring-proxy.conf \
/mnt/d/IT209_Devops_Funmentals/Session07/b4/spring-proxy.conf
```

Kiểm tra:

```bash
cd /mnt/d/IT209_Devops_Funmentals/Session07/b4
ls -l
```

Kết quả:

```text
spring-proxy.conf
```

---

## 16. Các lệnh kiểm tra cuối cùng

### Kiểm tra Spring Boot

```bash
ss -lntp | grep 8082
```

### Kiểm tra Spring Boot Health

```bash
curl -i http://127.0.0.1:8082/actuator/health
```

### Kiểm tra Nginx

```bash
sudo nginx -t
```

### Kiểm tra Nginx đang chạy

```bash
systemctl status nginx --no-pager
```

### Kiểm tra trang tĩnh

```bash
curl -I http://localhost/
```

### Kiểm tra Reverse Proxy

```bash
curl -i http://localhost/api/health
```

### Kiểm tra site đã enable

```bash
ls -l /etc/nginx/sites-enabled/
```

---

## 17. Kết quả cuối cùng

| Kiểm tra                             | Kết quả  |
| ------------------------------------ | -------- |
| Spring Boot port `8082`              | PASS     |
| Spring Boot `/actuator/health`       | `200 OK` |
| Nginx configuration                  | PASS     |
| Nginx reload                         | PASS     |
| Static `/`                           | `200 OK` |
| `/api/health`                        | `200 OK` |
| Reverse Proxy                        | PASS     |
| Họ tên sinh viên                     | PASS     |
| Mã lớp                               | PASS     |
| `spring-proxy.conf` trong thư mục B4 | PASS     |

---

## 18. Kết luận

Đã hoàn thành cấu hình Nginx Reverse Proxy cho ứng dụng Spring Boot.

Nginx đảm nhiệm:

```text
GET /
    ↓
/var/www/html/index.html
```

và:

```text
GET /api/health
    ↓
Nginx
    ↓
Spring Boot :8082
    ↓
/actuator/health
    ↓
HTTP 200
```

Hệ thống hoạt động thành công và đáp ứng các yêu cầu của bài B4.
