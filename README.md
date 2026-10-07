# Báo Cáo Bài Tập 4: Quản Lý Tiến Trình Nền Với Nohup Và Tín Hiệu Kill

- **Môn học**: DevOps Fundamentals (IT-209)
- **Session**: 06 - Linux Administration, Permissions & Process Management
- **Bài tập**: Bài 4 - Quản lý tiến trình nền với nohup và tín hiệu Kill
- **Đường dẫn nộp bài trên GitHub**: `homework/session_06/ex4/`
- **Học viên**: HoQuocKhai
- **Email**: khaiho6040@gmail.com

---

## 1. Mục tiêu bài thực hành
1. **Khởi chạy tiến trình nền độc lập bằng `nohup` và `&`**: Đảm bảo kịch bản giám sát không bị ảnh hưởng hay kết thúc khi ngắt kết nối phiên làm việc Terminal (SIGHUP).
2. **Theo dõi và quản lý tiến trình bằng `ps` và `grep`**: Xác định chính xác mã định danh tiến trình (PID) đang chạy trong hệ điều hành.
3. **Theo dõi tệp nhật ký theo thời gian thực**: Sử dụng `tail` để kiểm tra dữ liệu nhật ký được ghi liên tục từ tiến trình ngầm.
4. **Hiểu và vận dụng cơ chế tín hiệu (Signals) của lệnh `kill`**: Sử dụng tín hiệu `SIGTERM (15)` để dừng tiến trình an toàn, hiểu rõ lý do ưu tiên `SIGTERM` so với `SIGKILL (9)`.

---

## 2. Cơ sở lý thuyết: Quản lý tiến trình & Tín hiệu Linux

### 2.1. Giải phẫu câu lệnh `nohup ./loop-monitor.sh > /dev/null 2>&1 &`
- **`nohup` (No Hang Up)**: Ngăn chặn hệ điều hành gửi tín hiệu ngắt `SIGHUP` tới tiến trình khi phiên SSH bị ngắt.
- **`> /dev/null`**: Chuyển hướng luồng đầu ra chuẩn (Standard Output - `stdout` hay luồng `1`) vào thiết bị giả lập rỗng của hệ thống (hố đen dữ liệu), loại bỏ thông báo thừa.
- **`2>&1`**: Chuyển hướng luồng thông báo lỗi (Standard Error - `stderr` hay luồng `2`) vào cùng đích với luồng `1`.
- **`&`**: Đẩy tiến trình xuống chạy ngầm ở chế độ Background, giải phóng cửa sổ dòng lệnh cho người dùng tiếp tục thao tác.

### 2.2. So sánh tín hiệu `SIGTERM (15)` và `SIGKILL (9)`
| Tiêu chí | `SIGTERM (15)` | `SIGKILL (9)` |
| :--- | :--- | :--- |
| **Bản chất** | Yêu cầu kết thúc tiến trình an toàn (Graceful Termination). | Ép buộc tiến trình dừng ngay lập tức (Forced Kill). |
| **Khả năng xử lý** | Tiến trình có thể bắt (catch/trap) tín hiệu để giải phóng bộ nhớ, đóng file, hoàn tất giao dịch ghi đĩa. | Tiến trình **không thể** can thiệp hay chặn lại; do Linux Kernel trực tiếp tiêu diệt. |
| **Khuyến nghị sử dụng** | **Ưu tiên sử dụng đầu tiên** trong mọi trường hợp quản trị. | Chỉ sử dụng khi tiến trình bị treo cứng (hung/frozen) không phản hồi `SIGTERM`. |

---

## 3. Quá trình thực hiện chi tiết

### Bước 1: Khởi tạo mã nguồn [loop-monitor.sh](file:///d:/%5BIT-209%5D%20DevOps%20Fundamentals/Ss06/4/loop-monitor.sh)
Tạo file kịch bản thực hiện vòng lặp vô tận ghi thời gian hệ thống vào `/tmp/monitor.log` cứ mỗi 5 giây:
```bash
cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF
```

Cấp quyền thực thi cho kịch bản:
```bash
chmod +x loop-monitor.sh
```

---

### Bước 2: Khởi chạy kịch bản dưới chế độ nền với `nohup`
Thực thi lệnh chạy ngầm:
```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```
**Kết quả thực tế từ máy chủ:**
```text
devops@1161186107523368:~$ nohup ./loop-monitor.sh > /dev/null 2>&1 &
[1] 127092
```
> **Nhận xét:** Hệ điều hành khởi chạy tiến trình thành công với mã định danh **PID = 127092** tại Job `[1]`.

---

### Bước 3: Kiểm tra tiến trình hoạt động và xác nhận PID
Sử dụng `ps aux` kết hợp lọc `grep`:
```bash
ps aux | grep loop-monitor.sh
```
**Kết quả thực tế:**
```text
devops@1161186107523368:~$ ps aux | grep loop-monitor.sh
devops    127092  0.0  0.0   7344  3712 pts/1    S    13:14   0:00 /bin/bash ./loop-monitor.sh
devops    127098  0.0  0.0   6684  2304 pts/1    S+   13:14   0:00 grep --color=auto loop-monitor.sh
```
> **Nhận xét:** Tiến trình PID `127092` thuộc sở hữu của user `devops` đang ở trạng thái `S` (Interruptible Sleep), tiêu thụ tài nguyên tối thiểu.

---

### Bước 4: Kiểm tra tệp nhật ký ghi liên tục
Kiểm tra dữ liệu ghi ra tệp `/tmp/monitor.log`:
```bash
tail -n 10 /tmp/monitor.log
```
**Kết quả thực tế:**
```text
devops@1161186107523368:~$ tail -n 10 /tmp/monitor.log
System time: Wed Oct  7 01:14:37 PM BST 2026
System time: Wed Oct  7 01:14:42 PM BST 2026
System time: Wed Oct  7 01:14:47 PM BST 2026
System time: Wed Oct  7 01:14:52 PM BST 2026
```
> **Nhận xét:** Mỗi dòng log cách nhau đúng 5 giây theo cấu hình `sleep 5`.

---

### Bước 5: Dừng tiến trình an toàn bằng tín hiệu `SIGTERM (15)`
Gửi tín hiệu yêu cầu dừng tiến trình tới PID `127092`:
```bash
kill -15 127092
```

Kiểm tra lại danh sách tiến trình:
```bash
ps aux | grep loop-monitor.sh
```
**Kết quả thực tế:**
```text
devops@1161186107523368:~$ kill -15 127092
devops@1161186107523368:~$ ps aux | grep loop-monitor.sh
devops    127169  0.0  0.0   6684  2304 pts/1    S+   13:17   0:00 grep --color=auto loop-monitor.sh
[1]+  Terminated              nohup ./loop-monitor.sh > /dev/null 2>&1
```
> **Nhận xét:** Shell thông báo `[1]+ Terminated nohup ./loop-monitor.sh ...`. Tiến trình đã kết thúc sạch sẽ và biến mất khỏi danh sách tiến trình đang chạy.

---

## 4. Kết luận
- Kịch bản nền `loop-monitor.sh` được chạy thành công độc lập với Terminal qua `nohup`.
- Quản lý và dừng tiến trình an toàn bằng tín hiệu `SIGTERM (15)` tuân thủ đúng nguyên tắc vận hành hệ thống Linux.