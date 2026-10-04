# Bài 4: Chạy song song nhiều cổng dịch vụ (Nginx Virtual Hosts) - IT209 DevOps

## 1. Mục tiêu bài tập
- Cấu hình dịch vụ Web Server Nginx chạy song song hai trang web tĩnh độc lập trên cùng một địa chỉ IP máy chủ ảo thông qua cơ chế phân chia cổng dịch vụ (Port-based Virtual Hosting).
- Ưu tiên cổng cá nhân **2906** (và **2907**) để không bị xung đột khi nhiều người dùng chung 1 VPS, đồng thời hỗ trợ song song các cổng theo đề bài (**8080** và **8090**).
- Thiết lập tường lửa UFW mở đầy đủ các cổng dịch vụ (2906, 2907, 8080, 8090).

---

## 2. Cấu trúc tệp tin trong bài nộp
Đường dẫn trên GitHub: `homework/session_03/ex4/`
- `beta-app/index.html`: Giao diện trang kiểm thử ứng dụng (Beta App) phục vụ trên cổng `2906` và `8080`.
- `internal-app/index.html`: Giao diện trang thông tin nội bộ (Internal App) phục vụ trên cổng `2907` và `8090`.
- `multi-port.conf`: File cấu hình Nginx Server Block cấu hình cổng 2906 trước, sau đó đến cổng 8080 và 8090.
- `README.md`: Hướng dẫn chi tiết quy trình triển khai và kết quả kiểm thử thực tế.

---

## 3. Quy trình thực hiện chi tiết

### Bước 1: Chuyển tệp tin từ máy tính cá nhân lên VPS (`scp`)
Trên cửa sổ Git Bash ở máy tính cá nhân (tại thư mục bài làm `Session03/Ex4`):
```bash
scp -r beta-app internal-app multi-port.conf root@<IP_VPS>:~
```

---

### Bước 2: Tạo thư mục Web Root và phân quyền trên VPS
Đăng nhập vào VPS qua SSH và thực thi các lệnh:
```bash
# 1. Tạo 2 thư mục web root riêng biệt
sudo mkdir -p /var/www/beta-app/html
sudo mkdir -p /var/www/internal-app/html

# 2. Chuyển các file index.html vào đúng thư mục tương ứng
sudo mv ~/beta-app/index.html /var/www/beta-app/html/
sudo mv ~/internal-app/index.html /var/www/internal-app/html/

# 3. Phân quyền sở hữu cho Nginx (người dùng www-data)
sudo chown -R www-data:www-data /var/www/beta-app/html
sudo chown -R www-data:www-data /var/www/internal-app/html
sudo chmod 644 /var/www/beta-app/html/index.html
sudo chmod 644 /var/www/internal-app/html/index.html
```

---

### Bước 3: Cấu hình Nginx Virtual Hosts (`multi-port.conf`)
Chuyển file cấu hình vào thư mục `sites-available` của Nginx và kích hoạt:
```bash
sudo mv ~/multi-port.conf /etc/nginx/sites-available/multi-port.conf
sudo ln -sf /etc/nginx/sites-available/multi-port.conf /etc/nginx/sites-enabled/
```

---

### Bước 4: Cấu hình tường lửa UFW (Mở cổng 2906 trước)
```bash
# 1. Mở cổng cá nhân 2906 và 2907
sudo ufw allow 2906/tcp
sudo ufw allow 2907/tcp

# 2. Mở cổng theo đề bài 8080 và 8090
sudo ufw allow 8080/tcp
sudo ufw allow 8090/tcp

# Kiểm tra trạng thái
sudo ufw status
```

---

### Bước 5: Kiểm tra cú pháp và tải lại Nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```
> Kết quả `nginx -t` thành công:
> ```text
> nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
> nginx: configuration file /etc/nginx/nginx.conf test is successful
> ```

---

## 4. Kết quả kiểm thử thực tế trên Droplet / VPS (Proof of Work)

### Kiểm tra 1: Truy cập cổng cá nhân 2906 (Beta App - Ưu tiên hàng đầu)
```bash
curl -I http://160.250.246.219:2906
```
**Kết quả thực tế từ máy chủ:**
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 04 Oct 2026 21:41:18 GMT
Content-Type: text/html
Content-Length: 6567
Last-Modified: Sun, 04 Oct 2026 21:37:09 GMT
Connection: keep-alive
ETag: "6ac2c705-19a7"
Accept-Ranges: bytes
```
Tiêu đề: `<title>Beta App - Port 8080</title>`

---

### Kiểm tra 2: Truy cập cổng cá nhân 2907 (Internal App)
```bash
curl -I http://160.250.246.219:2907
```
**Kết quả thực tế từ máy chủ:**
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 04 Oct 2026 21:41:18 GMT
Content-Type: text/html
Content-Length: 6579
Last-Modified: Sun, 04 Oct 2026 21:37:09 GMT
Connection: keep-alive
ETag: "6ac2c705-19b3"
Accept-Ranges: bytes
```
Tiêu đề: `<title>Internal App - Port 8090</title>`

---

### Kiểm tra 3: Truy cập cổng theo đề bài 8080 (Beta App)
```bash
curl -I http://160.250.246.219:8080
```
**Kết quả:** `HTTP/1.1 200 OK`

---

### Kiểm tra 4: Truy cập cổng theo đề bài 8090 (Internal App)
```bash
curl -I http://160.250.246.219:8090
```
**Kết quả:** `HTTP/1.1 200 OK`

---

## 5. Giải thích cơ chế kỹ thuật
- **Ưu tiên cổng 2906**: Bằng cách khai báo `listen 2906;` trước `listen 8080;`, dịch vụ đảm bảo môi trường độc lập cho từng học viên trên cùng một máy chủ chia sẻ.
- **Port-based Virtual Hosting**: Nginx sử dụng chỉ thị `listen <port>` trong từng khối `server` để phân biệt các ứng dụng.
- **Tường lửa (Firewall)**: Đảm bảo cả các cổng cá nhân lẫn cổng đề bài đều được phép kết nối qua TCP.
