# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lê Thị Hoài Thương |
| Mã học viên | 2A202602898|
| Repo | https://github.com/thuongle06122004/K4-L3A-DAY12-LeThiHoaiThuong-2A202602898-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-1lr6.onrender.com |
| Platform | Render (Blueprint từ `render.yaml`) |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Render tự gán |
| `AGENT_API_KEY` | Cần set trong Render dashboard | Secret của service; chỉ ghi tên biến, không ghi giá trị |
| `REDIS_URL` | ✅ | Render tự gắn từ Redis add-on `day12-chat-redis` (thông qua `fromService`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra
```powershell
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl.exe -i https://day12-chat-1lr6.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready","redis":true} (đã nối được Redis)
curl.exe -i https://day12-chat-1lr6.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl.exe -i -X POST https://day12-chat-1lr6.onrender.com/ask -H "Content-Type: application/json" -d '{\"question\":\"Hello\"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
# Lấy giá trị từ biến môi trường local; không dán secret vào tài liệu.
$TOKEN = $env:DEPLOY_API_KEY
'{"question":"Deploy la gi"}' | Set-Content body.json -Encoding utf8
curl.exe -i -X POST https://day12-chat-1lr6.onrender.com/ask -H "Content-Type: application/json; charset=utf-8" -H "X-API-Key: $TOKEN" -H "X-User-Id: sv-test" --data-binary "@body.json"

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for ($i=1; $i -le 15; $i++) {
  (curl.exe -s -o $null -w "%{http_code} " -X POST https://day12-chat-1lr6.onrender.com/ask -H "Content-Type: application/json" -H "X-API-Key: $TOKEN" -H "X-User-Id: sv-test" --data-binary "@body.json")
}; Write-Host ""
```

## Kết Quả Chạy Thật

Sau khi Render redeploy commit này và biến `AGENT_API_KEY` được đặt trong
dashboard, chạy các lệnh phía trên và ghi lại kết quả thực tế tại đây. Không
ghi kết quả của API cũ (`/healthz`, `/readyz`, `/chat`) vì chúng không phải
endpoint của service trong repository này.

## Ảnh Chụp Màn Hình

Đặt trong `screenshots/`:
- `dashboard.png` — Render dashboard của service `day12-chat`
- `health.png` — kết quả gọi `/health`

---
