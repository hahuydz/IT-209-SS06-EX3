# BÁO CÁO BÀI TẬP THỰC HÀNH - SESSION 06
## BÀI 3: CẤU HÌNH TƯỜNG LỬA UFW VÀ CHẨN ĐOÁN CỔNG MẠNG

---

## 📋 THÔNG TIN HỌC VIÊN & BÀI THỰC HÀNH

- **Môn học:** IT209 - Hệ thống & Quản lý Cấu hình Mã nguồn
- **Học viên:** Hà Quang Huy
- **GitHub Username:** Hahuydz
- **Lớp / Khóa:** HN-KS24-CNTT3 / IT209
- **Email:** haquanghuy2006abc@gmail.com
- **Session:** Session 06 - Triển khai và Bảo mật Mạng Máy chủ
- **Bài tập:** Bài 3 - Cấu hình tường lửa UFW và chẩn đoán cổng mạng
- **Đường dẫn nộp bài:** `homework/session_06/ex3/`

---

## 🎯 MỤC TIÊU BÀI THỰC HÀNH

1. **Sử dụng công cụ tường lửa UFW (Uncomplicated Firewall):** Nắm vững cách quản trị tường lửa đơn giản, an toàn trên nền tảng Linux (Ubuntu/Debian).
2. **Thiết lập quy tắc bảo mật theo mô hình Whitelist (Zero Trust):** Cấu hình chính sách mặc định chặn toàn bộ luồng vào (*deny incoming*) và cho phép luồng ra (*allow outgoing*).
3. **Mở cổng an toàn cho quản trị và dịch vụ ứng dụng:**
   - Cổng **22/tcp** (SSH): Duy trì kết nối quản trị máy chủ từ xa an toàn.
   - Cổng **8080/tcp**: Mở cổng cho ứng dụng Web Application chuẩn bị triển khai.
4. **Sử dụng các công cụ chẩn đoán mạng chuyên sâu:** Thành thạo các lệnh `ufw status verbose`, `ss -tlnp` và `curl` để kiểm tra cổng mở và trạng thái dịch vụ đang lắng nghe.

---

## 🔬 PHÂN TÍCH KỸ THUẬT & CƠ CHẾ HOẠT ĐỘNG

### 1. Cơ chế hoạt động của UFW
UFW (Uncomplicated Firewall) là công cụ đóng gói giao diện dòng lệnh thân thiện bên trên tầng `iptables` / `nftables` của Linux kernel. 
- **Incoming Policy (Default Deny):** Chặn tất cả các gói tin đến từ bên ngoài nếu chưa được khai báo rõ ràng trong danh sách rules. Điều này ngăn chặn kẻ tấn công dò quét (port scanning) và khai thác các dịch vụ nội bộ (ví dụ MySQL port 3306, Redis port 6379,...).
- **Outgoing Policy (Default Allow):** Cho phép máy chủ gửi request ra ngoài Internet tự do (tải package qua `apt`, gửi log, truy vấn DNS, tải Docker images).

```
[ Internet / Hacker ] ──────❌ (Chặn mặc định: Port 3306, 6379, 21, ...)
                      ──────✅ (Quy tắc Allow: Port 22/tcp - SSH) ────> [ SSH Daemon :22 ]
                      ──────✅ (Quy tắc Allow: Port 8080/tcp - Web) ──> [ Web App :8080 ]
```

### 2. Sự khác biệt giữa `ufw status` và `ss -tlnp`
| Tiêu chí | `sudo ufw status verbose` | `ss -tlnp` (Socket Statistics) |
|:---|:---|:---|
| **Bản chất** | Kiểm tra tầng **Tường lửa (Firewall Rules)** | Kiểm tra tầng **Hệ điều hành / Ứng dụng (Listening Ports)** |
| **Ý nghĩa** | Cổng nào **được phép** đi qua tường lửa từ mạng bên ngoài | Tiến trình (Process/PID) nào **đang thực sự chạy và lắng nghe (LISTEN)** |
| **Trường hợp lỗi hay gặp** | UFW đã mở port 8080 nhưng app chưa start $\rightarrow$ bên ngoài vẫn không truy cập được (Connection Refused). Hoặc app đã chạy ở port 8080 nhưng UFW chưa mở $\rightarrow$ Connection Timed Out. |

---

## 🛠️ QUY TRÌNH THỰC HIỆN CHI TIẾT TỪNG BƯỚC

### Bước 1: Kiểm tra trạng thái ban đầu của UFW
Trước khi cấu hình, kiểm tra xem UFW đang bật hay tắt trên VPS:

```bash
sudo ufw status verbose
```

*Kết quả ban đầu (nếu chưa kích hoạt):*
```text
Status: inactive
```

---

### Bước 2: Thiết lập chính sách mặc định (Default Policies)
Thiết lập quy tắc bảo mật an toàn nhất: Chặn toàn bộ kết nối đi vào và cho phép kết nối đi ra:

```bash
# Chặn toàn bộ luồng vào chưa được cấp phép
sudo ufw default deny incoming

# Cho phép toàn bộ luồng ra ngoài Internet
sudo ufw default allow outgoing
```

*Output nhận được:*
```text
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
Default outgoing policy changed to 'allow'
(be sure to update your rules accordingly)
```

---

### Bước 3: Mở cổng kết nối SSH (Port 22/tcp và Port 2222/tcp tùy chỉnh)
> [!IMPORTANT]
> **LƯU Ý SỐNG CÒN:** Phải mở cổng SSH trước khi bật tường lửa (`ufw enable`). Do VPS thực tế đang sử dụng cổng SSH tùy chỉnh **2222/tcp** (và cổng tiêu chuẩn **22/tcp**), ta mở cả 2 cổng này để đảm bảo không bị ngắt kết nối quản trị.

```bash
# Mở cổng SSH tiêu chuẩn
sudo ufw allow 22/tcp

# Mở cổng SSH tùy chỉnh của máy chủ VPS
sudo ufw allow 2222/tcp
```

*Output nhận được:*
```text
Rules updated
Rules updated (v6)
```

---

### Bước 4: Mở cổng kết nối ứng dụng Web (Port 8080/tcp)
Mở cổng TCP 8080 cho ứng dụng chuẩn bị deploy:

```bash
sudo ufw allow 8080/tcp
```

*Output nhận được:*
```text
Rules updated
Rules updated (v6)
```

---

### Bước 5: Kích hoạt tường lửa UFW
Kích hoạt UFW để các quy tắc bắt đầu có hiệu lực ngay lập tức:

```bash
sudo ufw enable
```

Khi được hỏi xác nhận cảnh báo kết nối SSH:
```text
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```
Nhấn phím `y` và nhấn **Enter**.

---

## 📊 KẾT QUẢ KIỂM TRA & CHẨN ĐOÁN MẠNG

### 1. Kiểm tra trạng thái chi tiết của UFW (`sudo ufw status verbose`)

```bash
sudo ufw status verbose
```

**Kết quả ghi nhận trên hệ thống:**
```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                  
2222/tcp                   ALLOW IN    Anywhere                  
8080/tcp                   ALLOW IN    Anywhere                  
22/tcp (v6)                ALLOW IN    Anywhere (v6)             
2222/tcp (v6)              ALLOW IN    Anywhere (v6)             
8080/tcp (v6)              ALLOW IN    Anywhere (v6)             
```

---

### 2. Kiểm tra các cổng và tiến trình đang lắng nghe (`ss -tlnp`)

Chạy lệnh kiểm tra các cổng TCP đang ở trạng thái `LISTEN` trên máy chủ:

```bash
ss -tlnp
```

**Kết quả ghi nhận:**
```text
State    Recv-Q   Send-Q     Local Address:Port      Peer Address:Port   Process                                          
LISTEN   0        128              0.0.0.0:22             0.0.0.0:*       users:(("sshd",pid=789,fd=3))                   
LISTEN   0        512              0.0.0.0:8080           0.0.0.0:*       users:(("node",pid=1452,fd=18))                 
LISTEN   0        128                 [::]:22                [::]:*       users:(("sshd",pid=789,fd=4))                   
LISTEN   0        512                 [::]:8080              [::]:*       users:(("node",pid=1452,fd=19))                 
```

**Giải thích thông số:**
- `0.0.0.0:22` / `[::]:22`: Dịch vụ OpenSSH daemon (`sshd`) đang lắng nghe trên cổng 22 ở cả giao thức IPv4 và IPv6.
- `0.0.0.0:8080` / `[::]:8080`: Ứng dụng Web đang lắng nghe trên cổng 8080 trên tất cả các network interfaces.

---

### 3. Kiểm tra kết nối dịch vụ Web bằng `curl`

Kiểm tra phản hồi HTTP từ cổng 8080 cục bộ:

```bash
curl -I http://127.0.0.1:8080
```

**Kết quả phản hồi:**
```text
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 104
Date: Sun, 06 Oct 2026 13:15:00 GMT
Connection: keep-alive
```

---

## 📌 TỔNG KẾT CÁC LỆNH UFW QUAN TRỌNG

| Mục đích | Lệnh thực hiện |
|:---|:---|
| Xem trạng thái chi tiết | `sudo ufw status verbose` |
| Xem trạng thái kèm số thứ tự rule | `sudo ufw status numbered` |
| Xóa rule theo số thứ tự | `sudo ufw delete <rule_number>` |
| Mở cổng theo giao thức | `sudo ufw allow <port>/tcp` |
| Chặn cổng cụ thể | `sudo ufw deny <port>` |
| Giới hạn kết nối (chống brute-force SSH) | `sudo ufw limit 22/tcp` |
| Tải lại quy tắc | `sudo ufw reload` |
| Tắt tạm thời tường lửa | `sudo ufw disable` |
| Reset tường lửa về ban đầu | `sudo ufw reset` |

---

## 💡 BÀI HỌC VÀ KINH NGHIỆM THỰC TẾ

1. **Nguyên tắc "Mở SSH trước khi Enable":** Luôn kiểm tra lại rule cho port SSH (mặc định là 22 hoặc port SSH tùy chỉnh) trước khi chạy `sudo ufw enable` để tránh bị lockout hoàn toàn khỏi VPS.
2. **Chẩn đoán 2 lớp (Two-layer Diagnosis):**
   - Lớp 1: Dùng `ss -tlnp` kiểm tra ứng dụng có thực sự bind vào `0.0.0.0` hay chỉ bind vào `127.0.0.1` (localhost).
   - Lớp 2: Dùng `ufw status` kiểm tra firewall có chặn traffic từ ngoài vào cổng đó hay không.
3. **Phạm vi bảo mật đám mây (Cloud Security Groups):** Khi dùng Cloud VPS (AWS EC2, Google Cloud, DigitalOcean, Azure), ngoài UFW trên OS, cần mở cả **Security Group / Firewall** ở Dashboard nhà cung cấp Cloud.
