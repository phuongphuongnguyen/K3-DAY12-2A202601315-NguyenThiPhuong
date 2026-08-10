# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Thị Phương |
| Mã học viên | 2A202601315 |
| Repo | https://github.com/phuongphuongnguyen/K3-DAY12-2A202601315-NguyenThiPhuong |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-4dea.up.railway.app |
| Platform | Railway |
| Ngày deploy | 10/08/2026|

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway, nối bằng tham chiếu `${{Redis.REDIS_URL}}` |
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
# 1. Liveness
$ curl -i $URL/health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
$ curl -i $URL/ready
HTTP/1.1 200 OK
{"status":"ready","redis":true}

# 3. Không có API key
$ curl -i -X POST $URL/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

# 4. Có API key
$ curl -i -X POST $URL/ask -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test" \
    -d '{"question":"Deploy la gi?"}'
HTTP/1.1 200 OK
{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi
trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.
(Mình đang nhớ 2 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":2,
"cost_usd":3.465e-05,"tokens":{"in":43,"out":47}}

# 5. Rate limit — gọi 15 lần liên tiếp, RATE_LIMIT_PER_MINUTE=10
$ for i in $(seq 1 15); do curl -s -o /dev/null -w "%{http_code} " ... ; done
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Nhận xét: 10 request đầu qua được, 5 request cuối bị chặn bằng 429 — đúng hạn mức
`RATE_LIMIT_PER_MINUTE=10`. `history_length` tăng dần qua các lượt hỏi cùng
`X-User-Id`, chứng minh lịch sử nằm ở Redis chứ không nằm trong RAM của process.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

*Deploy thành công lên Railway nên không dùng phương án dự phòng.*
