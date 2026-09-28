# Hướng dẫn chạy và trình bày Day 12

## 1. Ý tưởng của bài

Service FastAPI nhận câu hỏi tại `POST /ask`. Mỗi request đi qua xác thực API key, rate limit theo user, kiểm tra ngân sách tháng, đọc lịch sử Redis, gọi mock LLM, sau đó ghi lịch sử, chi phí và log JSON. Mock LLM trả lời tất định và tính **chi phí giả lập**, không gọi OpenAI.

```
Browser / curl → FastAPI → API key → rate limit → cost guard
                                    → Redis history → mock LLM
                                    → Redis history + cost → JSON log → response
```

`GET /health` chỉ kiểm tra process; `GET /ready` còn kiểm tra Redis. Trang `/demo` dùng cùng API trên cùng origin. Nó không chứa API key trong HTML; người demo nhập key tại chỗ và trang chỉ giữ key trong bộ nhớ tab.

## 2. Chạy tại máy Windows

Mở PowerShell tại thư mục repo. File `.env` phải có `AGENT_API_KEY` riêng, không commit. Docker Desktop cần báo Engine running.

```powershell
docker info
docker compose up -d --build
docker compose ps
Invoke-RestMethod http://localhost:8000/health
Invoke-RestMethod http://localhost:8000/ready
```

Mở `http://localhost:8000/demo`. Nhập giá trị `AGENT_API_KEY` từ `.env` trên máy bạn vào ô API key, nhập `demo-user` và gửi 2 câu liên tiếp. Response thứ hai phải có `history_length = 2` vì một lượt gồm tin nhắn user và assistant. Không chụp ảnh màn hình có key đang hiển thị.

Kiểm tra 401 mà không cần đưa key vào lệnh:

```powershell
try {
    Invoke-RestMethod -Method Post -Uri http://localhost:8000/ask -ContentType 'application/json' -Body '{"question":"Xin chao"}'
} catch {
    [int]$_.Exception.Response.StatusCode
}
```

Lệnh trên cần in `401`. Muốn xem log JSON:

```powershell
docker compose logs --tail 30 agent
```

Chạy test và điểm:

```powershell
.\.venv\Scripts\python.exe -m pytest tests/test_cp1.py tests/test_cp2.py tests/test_cp3.py tests/test_cp4.py -v
.\.venv\Scripts\python.exe -m pytest tests/test_cp5.py -v
.\.venv\Scripts\python.exe grade.py --no-bonus
```

`test_cp5.py` chỉ có ý nghĩa khi đã deploy công khai hoặc bật phương án local fallback và có stack Docker thật. Đừng điền URL mẫu hoặc bịa output vào `DEPLOYMENT.md`.

## 3. Nói gì khi demo (khoảng 3 phút)

1. **30 giây:** mở `/demo`, giải thích đây là cùng FastAPI service; `/health` xanh nghĩa process sống, `/ready` xanh nghĩa Redis kết nối được.
2. **45 giây:** gửi một câu với API key đúng. Chỉ vào answer, token, chi phí giả lập. Gửi tiếp câu thứ hai với cùng User ID; `history_length` tăng từ 0 lên 2.
3. **30 giây:** bỏ hoặc nhập sai key để thấy 401. Chỉ giải thích `secrets.compare_digest` và lý do secret lấy từ environment.
4. **45 giây:** giới thiệu Redis lưu ba loại key: `history:`, `ratelimit:`, `cost:`. Rate limit trả 429 khi gọi quá 10 lần trong 60 giây; cost guard trả 402 khi vượt ngân sách tháng.
5. **30 giây:** chỉ Dockerfile multi-stage, non-root, healthcheck, `$PORT`, rồi mở URL Railway nếu đã deploy.

## 4. Hiểu từng checkpoint

| CP | Vấn đề | Cách xử lý trong repo |
|---|---|---|
| CP1 | Cấu hình khác nhau giữa máy và cloud; log khó đọc | `Settings` đọc environment, key bắt buộc; log một dòng JSON; `/health` không phụ thuộc Redis |
| CP2 | Container to, quyền cao, cố định cổng | Builder cài dependency, runtime dùng slim và user thường; `.dockerignore` loại secret; Compose nối agent với Redis theo hostname `redis` |
| CP3 | Public URL có thể bị gọi bừa và tiêu tiền | API key trả 401, sorted set Redis giữ request 60 giây và trả 429, chi phí theo user/tháng trả 402 |
| CP4 | Nhiều instance mất lịch sử hoặc rớt request khi deploy | History ở Redis, giới hạn 20 tin và TTL 7 ngày; `/ready` ping Redis; SIGTERM chuyển sang trạng thái shutdown rồi gọi handler cũ |
| CP5 | Người khác không gọi được service trên localhost | Railway build từ Dockerfile, đặt biến môi trường trong dashboard, gắn Redis và cấp HTTPS URL; ghi bằng chứng thật trong `DEPLOYMENT.md` |

### Các câu hỏi dễ bị hỏi

- **Vì sao `/health` không ping Redis?** Redis mất tạm thời không có nghĩa process hỏng; restart mọi instance làm sự cố lớn hơn. `/ready` mới kiểm tra dependency để ngừng nhận traffic.
- **Vì sao state phải nằm ở Redis?** Request của một user có thể vào các container khác nhau; RAM từng container không chia sẻ và mất khi restart.
- **Vì sao rate limit và ngân sách đều cần?** Một bên giới hạn tần suất trong 60 giây; bên kia giới hạn tổng chi phí trong tháng. Ít request nhưng dài vẫn có thể tốn tiền.
- **Vì sao kiểm tra trước khi gọi LLM?** Nếu gọi rồi mới chặn, chi phí đã phát sinh. Bài này dùng mock LLM nên chỉ mô phỏng chi phí.
- **Vì sao API key không nằm trong image/HTML?** Repo và image có thể được chia sẻ; secret phải cấp lúc chạy qua environment. Người dùng demo tự nhập key vào browser của mình.
- **Giới hạn của bản lab:** `X-User-Id` là header do client cung cấp. API key bảo vệ toàn service, nhưng chưa xác thực danh tính từng user. Không dùng thiết kế này để tính phí user thật khi chưa có định danh đáng tin cậy. Rate limit và kiểm tra ngân sách hiện là các thao tác Redis riêng, nên trong tải đồng thời rất cao cần giao dịch hoặc Lua script để tránh race condition.

## 5. Đưa lên Railway

1. Đổi tên repo GitHub theo đúng mẫu trong `SUBMISSION.md`, push các file đã hoàn thiện; không push `.env`.
2. Trong Railway, tạo project từ repo GitHub này, dùng Dockerfile ở gốc repo.
3. Thêm Redis service và gắn `REDIS_URL` của nó vào agent. Đặt `AGENT_API_KEY` trong Variables của agent. Có thể để mặc định `RATE_LIMIT_PER_MINUTE=10`, `MONTHLY_BUDGET_USD=10.0`, `LOG_LEVEL=INFO`. Railway tự cấp `PORT`.
4. Tạo public domain HTTPS. Kiểm tra `/health` và `/ready` đều 200, `/ask` không key trả 401, `/ask` có key trả 200. Mở `/demo` để trình bày.
5. Điền **URL và output thật** vào `DEPLOYMENT.md`, chụp `screenshots/dashboard.png` và `screenshots/health.png`, sau đó chạy CP5. `DEPLOYMENT.md` chỉ ghi tên biến môi trường, không ghi giá trị API key.

Nếu Railway chưa khả dụng, làm phương án `LOCAL_FALLBACK=true` theo `DEPLOYMENT.md`; mức điểm CP5 tối đa là 9/15. Đừng mô tả local fallback thành deploy công khai.

## 6. Bài viết cá nhân và bằng chứng

`exercises.md` gồm 10 câu phản ánh. Điền bằng lời của chính bạn sau khi quan sát. Câu 2 cần log thật; câu 3 cần số đo image thật; câu 10 cần lỗi deploy thật. Không điền số liệu hay lỗi giả để làm file trông đủ.

## 7. Kết quả kiểm tra đã quan sát trong phiên này

- `pytest tests/test_cp1.py tests/test_cp2.py tests/test_cp3.py tests/test_cp4.py -v -m "not docker"`: **68 passed, 2 deselected, 1 warning**. Hai test bị bỏ qua trong lệnh này là build và đo kích thước Docker image.
- Chạy FastAPI trên `127.0.0.1:8000` với `REDIS_URL=fake://`: `/demo`, `/health`, `/ready` đều 200; `/ask` không key trả 401; hai lượt hỏi có key trả 200 và `history_length` lần lượt là 0, 2.
- Log thật của lượt đầu (key không xuất hiện trong log):

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:06:06.085710+00:00", "user_id": "http-demo-check", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
```

- Docker Desktop đã được thử khởi động nhưng Engine chưa xuất hiện; `docker info` báo không kết nối được `dockerDesktopLinuxEngine`. Chưa có bằng chứng build image, kích thước image, hoặc Compose stack thật.
- Railway chưa deploy trong phiên này. `DEPLOYMENT.md` vẫn là mẫu; chưa có public URL, screenshot hoặc kết quả CP5. Các phần đó phải điền sau khi chạy và kiểm tra thật.
