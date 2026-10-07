# BÀI TẬP 4: QUẢN LÝ TIẾN TRÌNH NỀN VỚI NOHUP VÀ TÍN HIỆU KILL

**Mã bài tập:** SESSION 06 - EXERCISE 4 (`homework/session_06/ex4/`)  
**Môn học:** IT215 - Cloud Infrastructure & DevOps Fundamentals  
**Học viên:** Trần Đức Anh (`TranDucAnh3103`)  
**Email:** `TranDucAnh3103@users.noreply.github.com`  
**Hệ điều hành / Môi trường:** Ubuntu Linux (Server ID: `0310263186`) / Bash Shell / User: `stephen-devops`  
**Đường dẫn thư mục dự án:** `homework/session_06/ex4/`  

---

## 1. MỤC TIÊU BÀI TẬP

- Khởi chạy tiến trình chạy nền độc lập với phiên làm việc Terminal bằng lệnh `nohup` và ký tự `&`.
- Giám sát tiến trình đang hoạt động trong hệ thống bằng các công cụ `ps`, `pgrep`.
- Theo dõi tệp tin nhật ký được tiến trình ghi liên tục bằng `tail -f` / `tail -n`.
- Quản lý vòng đời tiến trình và dừng tiến trình an toàn bằng tín hiệu hệ thống (`SIGTERM - 15` và `SIGKILL - 9`).

---

## 2. MÃ NGUỒN KỊCH BẢN GIÁM SÁT (`loop-monitor.sh`)

Kịch bản thực hiện ghi nhận nhãn thời gian hệ thống vào tệp `/tmp/monitor.log` theo chu kỳ 5 giây:

```bash
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
```

---

## 3. TIẾN TRÌNH THỰC HIỆN VÀ NHẬT KÝ LỆNH

### Bước 1: Tạo kịch bản và cấp quyền thực thi

```bash
# Tạo file kịch bản
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF

# Cấp quyền thực thi cho script
chmod +x loop-monitor.sh
```

---

### Bước 2: Khởi chạy tiến trình chạy ngầm độc lập với `nohup`

Khởi chạy kịch bản dưới chế độ nền (`&`) và miễn nhiễm với tín hiệu ngắt kết nối SSH/SIGHUP (`nohup`):

```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

> **Giải thích cú pháp:**
> - `nohup`: Bỏ qua tín hiệu `SIGHUP` (Signal Hang Up) khi đóng terminal hoặc ngắt kết nối SSH.
> - `> /dev/null 2>&1`: Chuyển hướng toàn bộ luồng chuẩn (`stdout`) và luồng lỗi (`stderr`) vào `/dev/null`.
> - `&`: Đẩy tiến trình xuống chạy ngầm (Background Job).

---

### Bước 3: Xác định số định danh tiến trình (PID)

Sử dụng lệnh `ps aux` hoặc `pgrep` để tìm mã PID của tiến trình:

```bash
ps aux | grep loop-monitor.sh
```

**Đầu ra ghi nhận:**
```text
stephen-devops  18452  0.0  0.1  13248  3240 pts/0    S    19:25   0:00 /bin/bash ./loop-monitor.sh
stephen-devops  18485  0.0  0.0  12136  2176 pts/0    S+   19:25   0:00 grep --color=auto loop-monitor.sh
```

- **Mã PID của tiến trình `loop-monitor.sh`:** `18452`

---

### Bước 4: Kiểm tra tệp tin log được ghi liên tục

Kiểm tra 10 dòng nhật ký mới nhất trong tệp `/tmp/monitor.log`:

```bash
tail -n 10 /tmp/monitor.log
```

**Đầu ra ghi nhận:**
```text
System time: Wed Oct  7 19:25:05 UTC 2026
System time: Wed Oct  7 19:25:10 UTC 2026
System time: Wed Oct  7 19:25:15 UTC 2026
System time: Wed Oct  7 19:25:20 UTC 2026
System time: Wed Oct  7 19:25:25 UTC 2026
System time: Wed Oct  7 19:25:30 UTC 2026
System time: Wed Oct  7 19:25:35 UTC 2026
System time: Wed Oct  7 19:25:40 UTC 2026
System time: Wed Oct  7 19:25:45 UTC 2026
System time: Wed Oct  7 19:25:50 UTC 2026
```

Dữ liệu thời gian được ghi nhận đều đặn mỗi 5 giây, xác nhận tiến trình hoạt động chính xác.

---

### Bước 5: Dừng tiến trình an toàn bằng tín hiệu SIGTERM (15)

Gửi tín hiệu yêu cầu kết thúc an toàn tới tiến trình thông qua PID đã xác định:

```bash
kill -15 18452
```

Kiểm tra lại trạng thái để xác nhận tiến trình đã dừng hoàn toàn:

```bash
ps aux | grep loop-monitor.sh
```

**Đầu ra ghi nhận:**
```text
stephen-devops  18512  0.0  0.0  12136  2176 pts/0    S+   19:26   0:00 grep --color=auto loop-monitor.sh
```

Tiến trình PID `18452` đã biến mất hoàn toàn khỏi danh sách tiến trình đang hoạt động.

---

## 4. TỔNG KẾT CÁC TÍN HIỆU QUẢN LÝ TIẾN TRÌNH

| Tín hiệu | Mã số | Tên chuẩn | Cơ chế hoạt động & Ứng dụng |
| :--- | :---: | :---: | :--- |
| **SIGTERM** | `15` | Termination | Gửi yêu cầu dừng an toàn (Graceful shutdown). Tiến trình có thể bắt lấy tín hiệu (trap) để lưu trạng thái, đóng file descriptor, giải phóng tài nguyên trước khi thoát. |
| **SIGKILL** | `9` | Kill | Buộc kernel tiêu diệt tiến trình ngay lập tức. Tiến trình không thể chặn, bỏ qua hay xử lý tín hiệu này. Chỉ sử dụng khi tiến trình bị treo cứng không phản hồi `SIGTERM`. |

---

## 5. KẾT LUẬN

- Hoàn thành thiết lập kịch bản nền `loop-monitor.sh` độc lập với phiên SSH bằng `nohup`.
- Theo dõi thành công tệp log `/tmp/monitor.log` ghi đều đặn mỗi 5 giây.
- Quản lý và kết thúc tiến trình an toàn bằng tín hiệu `SIGTERM (15)` theo đúng nguyên tắc vận hành hệ thống Linux.
