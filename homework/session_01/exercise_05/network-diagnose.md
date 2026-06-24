

## Tình huống 1: Giải phóng cổng 8080 đang bị chiếm dụng

### Mô tả vấn đề

Ứng dụng `user-service` khởi chạy thất bại với thông báo lỗi:

```
Error: Address already in use (port 8080)
```

Nguyên nhân: một tiến trình khác đang lắng nghe trên cổng `8080`, khiến `user-service` không thể bind vào cổng đó.

---

### Các lệnh thực hiện

#### Bước 1 – Xác định tiến trình đang chiếm cổng 8080

```bash
sudo ss -tulpn | grep :8080
```

**Giải thích từng tùy chọn:**

| Tùy chọn | Ý nghĩa |
|----------|---------|
| `-t` | Hiển thị socket TCP |
| `-u` | Hiển thị socket UDP |
| `-l` | Chỉ liệt kê socket đang ở trạng thái LISTEN (đang lắng nghe) |
| `-p` | Hiển thị tên tiến trình và PID đang sở hữu socket |
| `-n` | Không phân giải tên host/service (hiển thị số IP và số cổng thô) |

**Kết quả mong đợi:**

```
tcp   LISTEN  0   128   0.0.0.0:8080   0.0.0.0:*   users:(("java",pid=23847,fd=12))
```

→ Tiến trình `java` với **PID = 23847** đang chiếm cổng `8080`.

---

#### Bước 2 – Tắt tiến trình chiếm cổng

```bash
sudo kill -9 23847
```

**Giải thích:**

- `kill` — gửi tín hiệu đến tiến trình theo PID.  
- `-9` — tín hiệu `SIGKILL` (buộc tắt ngay lập tức, không cho tiến trình dọn dẹp). Dùng khi tiến trình không phản hồi tín hiệu `-15` (SIGTERM).  
- `23847` — PID của tiến trình cần tắt, lấy từ kết quả bước trên.  
- `sudo` — cần quyền root khi tiến trình thuộc sở hữu của user khác.

---

#### Bước 3 – Xác nhận cổng đã được giải phóng

```bash
sudo ss -tulpn | grep :8080
```

**Kết quả mong đợi sau khi kill:** Không có dòng nào được in ra → cổng `8080` đã trống.

---

#### Bước 4 – Khởi động lại ứng dụng

```bash
./user-service start
# hoặc tùy cách chạy ứng dụng:
java -jar user-service.jar
```

---

## Tình huống 2: Kiểm tra kết nối tới Database PostgreSQL tại 10.0.1.20:5432

### Mô tả vấn đề

Ứng dụng không kết nối được tới Database PostgreSQL với lỗi:

```
Connection refused / Unable to connect to 10.0.1.20:5432
```

Cần kiểm tra hai lớp: **đường truyền vật lý** (host có reachable không) và **trạng thái cổng** (port 5432 có đang mở không).

---

### Các lệnh thực hiện

#### Bước 1 – Kiểm tra kết nối vật lý (ping)

```bash
ping -c 4 10.0.1.20
```

**Giải thích:**

- `ping` — gửi gói ICMP Echo Request đến địa chỉ IP đích, kiểm tra xem máy chủ có phản hồi không.  
- `-c 4` — chỉ gửi 4 gói rồi dừng (thay vì chạy vô hạn).  
- `10.0.1.20` — địa chỉ IP của máy chủ Database.

**Diễn giải kết quả:**

| Kết quả | Ý nghĩa |
|---------|---------|
| Nhận được reply (0% packet loss) | Đường truyền vật lý thông, máy chủ DB đang bật |
| Request timeout / 100% packet loss | Máy chủ DB tắt, hoặc firewall chặn ICMP |

---

#### Bước 2 – Kiểm tra bằng `curl` 

```bash
curl -v telnet://10.0.1.20:5432
```

**Giải thích:**

- `curl` — công cụ truyền dữ liệu hỗ trợ nhiều giao thức.  
- `-v` — chế độ verbose, hiển thị chi tiết quá trình kết nối.  
- `telnet://10.0.1.20:5432` — yêu cầu curl thực hiện TCP handshake tới host:port mà không cần gửi HTTP request.

---

#### Bước 3 – Kiểm tra thêm bằng `ss` trên máy chủ DB *(nếu có quyền truy cập)*

```bash
# Chạy trên máy chủ 10.0.1.20
sudo ss -tulpn | grep :5432
```

Lệnh này xác nhận PostgreSQL có thực sự đang lắng nghe hay không, và bind vào interface nào (`0.0.0.0` hay `127.0.0.1`).

---

### Tổng hợp kiểm tra và hành động khắc phục

| Kết quả ping | Kết quả nc/curl | Nguyên nhân có thể | Hành động |
|---|---|---|---|
| OK | OK | Vấn đề ở application config | Kiểm tra credentials, DB name trong config |
| OK | Refused | PostgreSQL chưa chạy hoặc bind sai interface | `sudo systemctl start postgresql` |
| OK | Timeout | Firewall chặn cổng 5432 | Mở rule firewall: `ufw allow 5432` |
| Fail | N/A | Máy chủ DB tắt hoặc network issue | Khởi động máy chủ, kiểm tra routing |

---

## Tóm tắt các lệnh đã sử dụng

```bash
# === TÌNH HUỐNG 1: Giải phóng cổng 8080 ===
sudo ss -tulpn | grep :8080     # Tìm PID chiếm cổng 8080
sudo kill -9 <PID>              # Buộc tắt tiến trình theo PID
sudo ss -tulpn | grep :8080     # Xác nhận cổng đã trống

# === TÌNH HUỐNG 2: Kiểm tra kết nối DB tại 10.0.1.20:5432 ===
ping -c 4 10.0.1.20             # Kiểm tra kết nối vật lý (ICMP)
nc -zv 10.0.1.20 5432           # Kiểm tra cổng 5432 có mở không
curl -v telnet://10.0.1.20:5432 # Thay thế nc nếu không cài sẵn
sudo ss -tulpn | grep :5432     # Kiểm tra PostgreSQL trên máy chủ DB
```

---

