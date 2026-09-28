# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Dương Quốc Khánh |
| Mã học viên | 2A202603013 |
| Repo | https://github.com/jaimesHub/K4-L3A-DAY12-DuongQuocKhanh-2A202603013-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-292b.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis service riêng trong cùng project Railway, qua `${{Redis.REDIS_URL}}` (không phải Upstash) |
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

```
=== 1. /health ===
HTTP/2 200
content-type: application/json
server: railway-hikari

{"status":"ok","service":"day12-agent","version":"1.0.0"}

=== 2. /ready ===
HTTP/2 200
content-type: application/json
server: railway-hikari

{"status":"ready","redis":true}

=== 3. /ask không có API key ===
HTTP/2 401
content-type: application/json
server: railway-hikari

{"detail":"invalid or missing API key"}

=== 4. /ask có API key ===
HTTP/2 200
content-type: application/json
server: railway-hikari

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

=== 5. Rate limit — 15 lần gọi liên tiếp (RATE_LIMIT_PER_MINUTE=10) ===
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

9 request đầu trả 200, từ request thứ 10 trở đi bị chặn với 429 — đúng như kỳ vọng với `RATE_LIMIT_PER_MINUTE=10`.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không áp dụng — học viên đã deploy thành công lên Railway tại
https://agent-production-292b.up.railway.app, không cần dùng phương án dự
phòng (không set `LOCAL_FALLBACK`).
