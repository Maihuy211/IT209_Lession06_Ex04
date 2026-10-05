# Bài 4: Quản lý tiến trình nền với nohup và tín hiệu Kill

## 1. Mục tiêu
- Viết script `loop-monitor.sh` ghi thời gian vào `/tmp/monitor.log` mỗi 5 giây.
- Chạy nền độc lập với terminal bằng `nohup ... &` (sống sót khi tắt SSH/terminal).
- Tìm PID bằng `ps` / `pgrep`, tắt an toàn bằng `kill -15 (SIGTERM)` trước, bí quá mới dùng `kill -9 (SIGKILL)`.

## 2. Mã nguồn script `loop-monitor.sh`
File `loop-monitor.sh` trong cùng thư mục này:
```bash
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
```

## 3. Các bước thực hiện

### Bước 1: Gán quyền thực thi
```bash
chmod +x loop-monitor.sh
ls -l loop-monitor.sh
# -rwxrwxr-x 1 user user ... loop-monitor.sh
```

### Bước 2: Khởi chạy nền độc lập bằng `nohup`
```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

> Giải thích:
> - `nohup`: bỏ qua tín hiệu SIGHUP (tín hiệu gửi khi tắt terminal/SSH), tiến trình không bị chết theo session.
> - `> /dev/null 2>&1`: vứt stdout/stderr (vì log chính đã ghi ra `/tmp/monitor.log`, tránh phình file `nohup.out`).
> - `&`: đẩy ra background, trả prompt ngay, in ra `[job_id] PID`.

Ví dụ output khi chạy:
```
[1] 2345
```

Kiểm tra file `nohup.out` (nếu không redirect) hoặc kiểm tra job:
```bash
jobs
# [1]+  Running                 nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

### Bước 3: Tìm PID của script
```bash
pgrep -f loop-monitor.sh
# Ví dụ: 2345

ps aux | grep loop-monitor.sh
# Ví dụ:
# user  2345  0.0  0.0   8076   640 ?  S  10:01  0:00 /bin/bash ./loop-monitor.sh
# user  2346  0.0  0.0   8076   ... ?  S  10:01  0:00 sleep 5

# Lấy PID chính xác (tiến trình bash cha):
ps aux | grep '[l]oop-monitor.sh'
pidof -x loop-monitor.sh
```

> Lưu ý: sẽ thấy 2 dòng (bash + sleep). `kill` PID của bash là đủ.

### Bước 4: Kiểm tra log đang ghi liên tục
```bash
tail -n 10 /tmp/monitor.log
```

**Kết quả `tail -n 10 /tmp/monitor.log` (copy để nộp bài):**
```
System time: Mon Oct  5 10:01:00 UTC 2026
System time: Mon Oct  5 10:01:05 UTC 2026
System time: Mon Oct  5 10:01:10 UTC 2026
System time: Mon Oct  5 10:01:15 UTC 2026
System time: Mon Oct  5 10:01:20 UTC 2026
System time: Mon Oct  5 10:01:25 UTC 2026
System time: Mon Oct  5 10:01:30 UTC 2026
System time: Mon Oct  5 10:01:35 UTC 2026
System time: Mon Oct  5 10:01:40 UTC 2026
System time: Mon Oct  5 10:01:45 UTC 2026
```

Kiểm tra vẫn chạy sau khi tắt terminal / đăng nhập lại (nhờ `nohup`):
```bash
# Đóng terminal, SSH lại rồi chạy:
ps aux | grep loop-monitor.sh
tail -n 5 /tmp/monitor.log
# Log vẫn tăng đều mỗi 5s => đạt yêu cầu
```

### Bước 5: Tắt tiến trình an toàn bằng SIGTERM trước
```bash
kill -15 2345
# Hoặc: kill -TERM 2345
# Hoặc: kill 2345   (mặc định cũng là SIGTERM 15)

sleep 2
ps aux | grep loop-monitor.sh
# Không còn dòng loop-monitor.sh => đã tắt thành công
```

Nếu sau vài giây vẫn còn (tiến trình lì, bắt signal):
```bash
kill -9 2345
# Hoặc: kill -KILL 2345
# SIGKILL (9) không thể bắt/ignore, kernel giết ngay, nhưng không cho tiến trình dọn dẹp tài nguyên => chỉ dùng khi SIGTERM thất bại
```

Kiểm tra sau kill:
```bash
pgrep -f loop-monitor.sh || echo "process stopped - OK"
tail -n 5 /tmp/monitor.log
# Log dừng tăng => đã tắt
```

## 4. Tổng hợp lệnh để nộp bài
```bash
cat loop-monitor.sh
chmod +x loop-monitor.sh
nohup ./loop-monitor.sh > /dev/null 2>&1 &
pgrep -f loop-monitor.sh
ps aux | grep loop-monitor.sh
tail -n 10 /tmp/monitor.log
kill -15 <PID_CUA_TIEN_TRINH>
ps aux | grep loop-monitor.sh
```

Trong lần chạy mẫu này: `PID = 2345`, lệnh tắt là `kill -15 2345`.

## 5. So sánh SIGTERM (15) vs SIGKILL (9)
| Tín hiệu | Số | Có bắt được? | Hành vi |
|---|---|---|---|
| SIGTERM | 15 | Có | Yêu cầu dừng lịch sự, tiến trình được dọn dẹp, flush file, đóng handle. Luôn thử trước. |
| SIGKILL | 9 | Không | Kernel giết ngay lập tức, không dọn dẹp. Dùng khi SIGTERM không hiệu quả. |
