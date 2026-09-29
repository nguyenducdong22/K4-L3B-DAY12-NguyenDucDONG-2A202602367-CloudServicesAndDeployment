# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Đức Đông |
| Mã học viên | 2A202602367 |
| Repo | https://github.com/nguyenducdong22/K4-L3B-DAY12-NguyenDucDONG-2A202602367-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3b-day12-nguyenducdong-2a202602367-cloudserv-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis database của Railway, tham chiếu `${{Redis.REDIS_URL}}` (không copy giá trị) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

**Trên cloud (https://k4-l3b-day12-nguyenducdong-2a202602367-cloudserv-production.up.railway.app), chạy ngày 2026-09-29:**

```
$ curl <URL>/health
{"status":"ok","service":"day12-agent","version":"1.0.0"} [200]

$ curl <URL>/ready
{"status":"ready","redis":true} [200]

$ curl -X POST <URL>/ask -d '{"question":"Hello"}'                    # không có key
{"detail":"invalid or missing API key"} [401]

$ curl -X POST <URL>/ask -H "X-API-Key: khoa-sai" -d '{"question":"Hello"}'
{"detail":"invalid or missing API key"} [401]
```

TODO(học viên): dán thêm output lệnh 4 (có key → 200) và lệnh 5 (rate limit) chạy trên cloud.

**Trên máy (`docker compose up -d`, http://localhost:8000), chạy ngày 2026-09-29:**

```
$ curl http://localhost:8000/health
{"status":"ok","service":"day12-agent","version":"1.0.0"}

$ curl http://localhost:8000/ready
{"status":"ready","redis":true}

$ curl -X POST http://localhost:8000/ask -d '{"question":"Hello"}'      # không có key
{"detail":"invalid or missing API key"} [401]

# Có key, cùng X-User-Id: sv01, hỏi 3 lần
history_length= 0 cost= 2.265e-05 tokens= {'in': 3, 'out': 37}
history_length= 2 cost= 3.345e-05 tokens= {'in': 43, 'out': 45}
history_length= 4 cost= 4.17e-05 tokens= {'in': 90, 'out': 47}

# Rate limit: 15 lần liên tiếp, user mới
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429

# Redis dừng: /health vẫn 200, /ready chuyển 503
{"status":"ok","service":"day12-agent","version":"1.0.0"} [200]
{"status":"not ready","redis":false} [503]
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
