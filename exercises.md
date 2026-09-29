# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Đông  Mã học viên: 2A202602367

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `agent_api_key="changeme"`, khi deploy lên production mà ta quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động bình thường. Bất kỳ ai trên Internet đều có thể đoán được key mặc định `"changeme"` hoặc dùng script quét tự động để gọi API thoải mái, đốt sạch quota/hóa đơn LLM của ta mà ta chỉ phát hiện khi nhận thông báo hết tiền hoặc hóa đơn thẻ tín dụng vào cuối tháng. Việc không đặt mặc định buộc Pydantic ném `ValidationError` ngay lúc khởi động (Fail Fast), container lập tức dừng và báo lỗi đỏ trong log deploy, giúp ta phát hiện và bổ sung biến bí mật ngay lập tức trước khi lộ ra ngoài.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được (lấy từ `docker compose logs agent` sau khi gọi `/ask`
với `X-User-Id: sv01`):
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:26:08.172255+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Hai việc làm được với dòng log JSON này:
1. Các công cụ giám sát và thu thập log tập trung (Datadog, Grafana Loki, CloudWatch) có thể tự động parse các trường có cấu trúc để thống kê, vẽ biểu đồ chi phí LLM (`cost_usd`) và số lượng token theo từng người dùng (`user_id`).
2. Thiết lập bộ lọc và cảnh báo tự động (Alerting) khi có bất thường, ví dụ: kích hoạt cảnh báo nếu tổng `cost_usd` của một user vượt ngưỡng cho phép hoặc tỉ lệ `level == "error"` tăng đột biến trong 5 phút.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73 GB (content size 446 MB) |
| Multi-stage | 271 MB (content size 63.9 MB) |

Số đo từ `docker images agent` trên Docker Desktop (Windows). Cột "content
size" là dung lượng nén khi đẩy lên registry.

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chênh lệch khoảng 1.46 GB, gần như toàn bộ đến từ base image:
1. `python:3.11` bản đầy đủ dựa trên Debian đầy đủ, mang theo gcc/g++, header
   phát triển, git, và rất nhiều thư viện hệ thống chỉ cần khi biên dịch.
   Stage runtime dùng `python:3.11-slim` chỉ giữ phần tối thiểu để chạy Python.
2. Stage `builder` cài thư viện vào `/install` rồi bị bỏ đi; runtime chỉ
   `COPY --from=builder /install` nên không mang theo gì của quá trình build.
   `--no-cache-dir` giúp không lưu cache của pip.
3. Bản multi-stage chỉ copy `app/` và `utils/`, không `COPY . .` toàn bộ repo.
   (Hai bản build cùng dùng `.dockerignore` hiện tại nên `.env`, `.venv`, `.git`
   đều không lọt vào image nào.)

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi thêm một dòng comment vào `app/main.py` rồi build lại
  (`docker build --progress=plain`), output cho thấy:
  - **CACHED**: `WORKDIR /build`, `COPY requirements.txt .`,
    `RUN pip install ...`, `WORKDIR /app`, `COPY --from=builder /install /usr/local`.
  - **Chạy lại**: `COPY app ./app`, `COPY utils ./utils`, `RUN useradd ...`.
  `COPY utils` và `useradd` không đổi gì nhưng vẫn chạy lại vì chúng đứng
  **sau** layer bị thay đổi — Docker hủy cache từ layer đầu tiên thay đổi trở
  đi. Bước tốn thời gian nhất (`pip install`) vẫn được giữ nguyên.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi thay đổi bất kỳ ký tự nào trong mã nguồn, Docker sẽ làm mất hiệu lực (cache bust) toàn bộ các layer phía sau nó, buộc Docker phải chạy lại `pip install` từ đầu, làm tăng thời gian build từ vài giây lên vài phút mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Container mặc định chạy bằng root (UID 0), trùng với UID root trên máy host Linux. Chuỗi sự kiện:
1. Kẻ tấn công phát hiện lỗ hổng RCE (Remote Code Execution) hoặc lỗ hổng thư viện trong ứng dụng Python để thực thi lệnh shell.
2. Khi chiếm được shell trong container, kẻ tấn công lập tức có quyền root trong container.
3. Từ quyền root này, kẻ tấn công có thể khai thác các lỗ hổng nhân hệ điều hành (kernel privilege escalation), hoặc truy cập các volume mount nhạy cảm / docker socket (`/var/run/docker.sock`) để thoát (container escape) ra máy host với toàn quyền root.
Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay từ bước 2: tiến trình ứng dụng chỉ chạy với UID 10001 (user thông thường không có quyền sudo), không thể chỉnh sửa file hệ thống của container, chặn đứng bước leo thang đặc quyền để escape ra host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20** request trong 2 giây liên tiếp.
Cách đạt được:
- Lúc 10:00:59, người dùng gửi 10 request (hợp lệ trong phút 10:00).
- Ngay 1 giây sau, đồng hồ chuyển sang 10:01:00, bộ đếm bị reset về 0, người dùng gửi tiếp 10 request nữa trong giây 10:01:00 - 10:01:01.
Như vậy trong khoảng thời gian 2 giây (từ 10:00:59 đến 10:01:01), hệ thống phải chịu 20 request (gấp đôi hạn mức). Thuật toán Sliding Window (cửa sổ trượt) bằng Redis ZSET giải quyết triệt để lỗi này bằng cách luôn xét đúng 60 giây trôi qua tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau: Rate limit giới hạn **tần suất/số lượng request** trong một đơn vị thời gian (bảo vệ tài nguyên server/API khỏi bị nghẽn mạng). Cost guard giới hạn **tổng chi phí/tiền bạc** phát sinh trong kỳ (bảo vệ ví tiền khỏi hóa đơn LLM).
- Tình huống Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 3 request/phút (dưới hạn mức 10 req/phút), nhưng mỗi request có prompt và tài liệu đính kèm khổng lồ (hàng chục ngàn tokens), khiến chi phí vượt quá ngân sách tháng (`monthly_budget_usd`), Cost guard sẽ ném lỗi 402 chặn lại.
- Tình huống Cost guard cho qua nhưng Rate limit chặn: User gửi 30 request trong 10 giây, mỗi request chỉ hỏi câu ngắn "hi", "ok" (chi phí cực kỳ rẻ, chỉ 0.00001 USD, ngân sách còn rất nhiều), nhưng vì gửi quá dồn dập nên Rate limit ném lỗi 429 chặn lại để tránh quá tải dịch vụ.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện khi gộp và Redis mất kết nối 30 giây:
1. Redis bị mất kết nối hoặc khởi động lại.
2. Liveness check (`/health`) của cả 3 container đều gọi vào Redis và bị fail (trả về 503 hoặc timeout).
3. Container orchestrator (Docker/Kubernetes) nhận thấy endpoint liveness báo fail nên kết luận cả 3 container đã "chết", lập tức gửi lệnh restart hoặc kill cả 3 container cùng lúc.
4. Hệ thống rơi vào trạng thái không còn bất kỳ container nào sống để nhận traffic, toàn bộ người dùng gặp lỗi 502 Bad Gateway / 503 Service Unavailable.
5. Khi Redis phục hồi xong, cả 3 container vẫn đang trong quá trình restart/khởi tạo lại. Một sự cố mạng tạm thời nhỏ của Redis đã bị phóng đại thành downtime hoàn toàn của toàn bộ hệ thống.
Do đó, `/health` (liveness) chỉ kiểm tra process app có sống không (không phụ thuộc Redis), còn `/ready` (readiness) mới kiểm tra kết nối Redis để điều hướng traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Quan sát thật: chạy 3 instance agent sau Nginx
(`docker compose -f docker-compose.yml -f docker-compose.lb.yml up -d --scale agent=3`),
gọi `/ask` 6 lần qua cổng 8080 với `X-User-Id: sv-scale`. Log cho thấy
agent-1, agent-2, agent-3 mỗi container xử lý 2 request (round-robin), nhưng
`history_length` vẫn tăng đều: 0, 2, 4, 6, 8, 10 — vì cả 3 container cùng đọc
và ghi lịch sử ở một Redis.

Nếu lịch sử hội thoại được lưu trong một `dict` trong RAM Python của từng container:
Khi scale lên 3 container (A, B, C), mỗi container có một vùng nhớ RAM tách biệt hoàn toàn. Khi người dùng gửi liên tiếp các câu hỏi với cùng một `X-User-Id`, các request sẽ được load balancer chia tải ngẫu nhiên vào các container khác nhau:
- Câu hỏi 1 rơi vào container A: `history_length = 0`, A lưu vào RAM của A.
- Câu hỏi 2 rơi vào container B: B chưa từng gặp user này nên `history_length = 0` (thay vì 2), người dùng thấy agent bị "mất trí nhớ".
- Câu hỏi 3 rơi vào container A: `history_length = 2`.
- Câu hỏi 4 rơi vào container C: `history_length = 0`.
Con số `history_length` sẽ nhảy lung tung hoặc tụt về 0. Khi lưu trên Redis tập trung, `history_length` tăng đều đặn 0 -> 2 -> 4 -> 6... bất kể request đi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp khi chạy container ở máy (trước khi đẩy lên cloud — cùng image sẽ
được Railway build và chạy):

- **Triệu chứng:** `docker compose stop agent` xong, container thoát với mã
  **137** (`docker inspect --format '{{.State.ExitCode}}'`), và trong
  `docker compose logs agent` không có dòng `service_stopped` hay
  `Shutting down` — tức là app bị SIGKILL, graceful shutdown ở CP4 không hề chạy.
- **Tìm nguyên nhân:** mã 137 = 128 + 9 (SIGKILL). Xem lại Dockerfile:
  `CMD ["sh", "-c", "uvicorn ..."]` → tiến trình PID 1 trong container là `sh`,
  uvicorn chỉ là tiến trình con. Docker gửi SIGTERM cho PID 1, nhưng `sh` không
  chuyển tiếp tín hiệu cho con, nên uvicorn không biết phải tắt; hết thời gian
  chờ Docker SIGKILL cả container. Test pytest không bắt được lỗi này vì test
  gọi thẳng `request_shutdown()` chứ không đi qua Docker.
- **Cách sửa:** đổi thành `CMD ["sh", "-c", "exec uvicorn ... --port ${PORT:-8000}"]`.
  `exec` làm uvicorn thay thế `sh` và trở thành PID 1 (vẫn giữ được việc đọc
  `$PORT`). Build lại và thử: container thoát mã **0**, log có
  `INFO: Shutting down` → `{"event": "service_stopped", ...}` →
  `Finished server process [1]`.

> TODO(học viên): nếu khi deploy lên Railway gặp thêm lỗi khác, có thể ghi lỗi đó ở đây.
