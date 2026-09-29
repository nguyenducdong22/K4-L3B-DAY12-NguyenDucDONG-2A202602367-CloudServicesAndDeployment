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

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:19:18.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00012}
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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) bao gồm:
1. Base image đầy đủ (`python:3.11`) chứa toàn bộ trình biên dịch (gcc, g++), build headers, thư viện hệ thống và package quản lý không cần thiết lúc chạy, trong khi bản runtime dùng `python:3.11-slim` chỉ giữ runtime tối thiểu.
2. Cache pip và các file build trung gian trong quá trình `pip install` nằm ở stage builder và không bị copy sang stage runtime (nhờ chỉ copy thư mục cài đặt sạch `/install` sang `/usr/local`).
3. Loại bỏ các file rác, file mã nguồn và môi trường ảo thừa (`.git`, `.venv`, cache tests) nhờ `.dockerignore`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`: Các layer được dùng lại từ cache bao gồm: `FROM python:3.11-slim`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install ...` và `COPY utils ./utils`. Chỉ các layer từ `COPY app ./app` trở đi mới phải chạy lại.
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

- Lỗi: Health check timeout / Service unreachable khi deploy lên Cloud (Railway/Render).
- Triệu chứng: Dashboard báo container failed to become healthy hoặc crash sau start-period.
- Nguyên nhân: Trong lệnh khởi chạy ứng dụng bị bind cứng vào `127.0.0.1` hoặc cố định cổng `8000`, trong khi nền tảng cloud cấp một cổng động qua biến môi trường `$PORT` và yêu cầu lắng nghe trên tất cả các network interface (`0.0.0.0`).
- Cách sửa: Cập nhật lệnh chạy server thành `uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}` và cấu hình Dockerfile `HEALTHCHECK` đọc biến môi trường `$PORT`.
