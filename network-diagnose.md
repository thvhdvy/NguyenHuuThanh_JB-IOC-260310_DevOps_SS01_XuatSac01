# Báo cáo chẩn đoán mạng

## Tình huống 1: Trùng cổng 8080 (`Address already in use`)

### Bối cảnh

Khi `user-service` khởi động, Spring Boot cố `bind` cổng 8080. Mỗi cổng TCP chỉ một tiến trình được giữ tại một địa chỉ, nên nếu đã có tiến trình khác giữ thì ứng dụng báo `Address already in use` rồi thoát.

Để mô phỏng, dùng một web server giả chiếm cổng 8080.

### Các lệnh đã dùng

```bash
python3 -m http.server 8080
```
**Mục đích:** dựng tiến trình giả chiếm cổng 8080 (chạy ở cửa sổ A, giữ nguyên không đóng). Chạy lần hai ở cửa sổ khác sẽ báo `OSError: [Errno 98] Address already in use`, đúng lỗi của sự cố.

```bash
sudo ss -tulpn | grep :8080
```
**Mục đích:** tìm tiến trình đang giữ cổng 8080.

**Kết quả trước khi tắt** (ảnh `port_fix.png`):

```
tcp   LISTEN 0   5   0.0.0.0:8080   0.0.0.0:*   users:(("python3",pid=5276,fd=3))
```

Tiến trình `python3` có **PID 5276** đang giữ cổng 8080.

```bash
kill 5276
```
**Mục đích:** gửi SIGTERM, yêu cầu tiến trình tự tắt, cho nó cơ hội đóng kết nối và dọn dẹp. 
**Kết quả:** cửa sổ chạy server in `Terminated`, tiến trình đã tắt.

```bash
sudo ss -tulpn | grep :8080
```
**Mục đích:** kiểm tra lại cổng đã được giải phóng.
 **Kết quả:** không in ra dòng nào, nghĩa là không còn tiến trình nào giữ 8080.


## Tình huống 2: Không kết nối được PostgreSQL tại `10.0.1.20`

### Bối cảnh

Lỗi "không kết nối được DB" có thể hỏng ở nhiều tầng. Cần kiểm tra từ dưới lên:

1. Tầng IP: máy `10.0.1.20` có tới được không? (`ping`)
2. Tầng TCP: cổng 5432 của PostgreSQL có mở không? (`nc`, `curl`)
3. Tầng ứng dụng: tài khoản, mật khẩu, cấu hình. Chỉ xét khi hai tầng trên đã thông.

### Các lệnh đã dùng

```bash
ping -c 4 10.0.1.20
```
**Mục đích:** kiểm tra đường truyền tới máy DB. `-c 4` gửi đúng 4 gói rồi dừng.

**Kết quả:**
```
4 packets transmitted, 0 received, 100% packet loss, time 3054ms
```

```bash
nc -zv -w 3 10.0.1.20 5432
```
**Mục đích:** thăm dò cổng 5432. `-z` chỉ dò, không gửi dữ liệu. `-v` in kết quả chi tiết. `-w 3` chờ tối đa 3 giây.

**Kết quả:**
```
nc: connect to 10.0.1.20 port 5432 (tcp) timed out: Operation now in progress
```

### Kết luận chẩn đoán

| Quan sát | Ý nghĩa |
|---|---|
| `ping`: 100% mất gói | Không có phản hồi ICMP |
| `nc`: `timed out` | Không có phản hồi TCP ở cổng 5432 |

Cả hai tầng IP và TCP đều không phản hồi, nên lỗi nằm ở **tầng mạng**, chưa phải sai tài khoản hay mật khẩu PostgreSQL.

Kết quả ở đây là `timed out`. Vì `ping` thất bại chưa chắc máy chết (firewall có thể chặn ICMP), nên đã thử thêm cổng TCP 5432, và cổng đó cũng không phản hồi.