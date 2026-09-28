# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết dưới mỗi câu hỏi bằng lời của mình, dựa trên quan sát thật.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Nhân Sâm  Mã học viên: 2A202602672

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lần deploy Railway đầu tiên đã crash thật vì quên `AGENT_API_KEY`; log báo `agent_api_key Field required`. Đây là fail fast hữu ích: service không nhận request trong trạng thái ai cũng có thể dùng khóa mặc định `changeme`. Tôi mở Variables, thêm khóa riêng và deploy lại; Railway sau đó báo Online.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log đã quan sát: `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T08:06:06.085710+00:00","user_id":"http-demo-check","tokens_in":3,"tokens_out":41,"cost_usd":2.505e-05}`. Tôi có thể lọc theo `event` và `user_id` để tìm request liên quan; cũng có thể tổng hợp `cost_usd`, `tokens_in`, `tokens_out` theo thời gian để quan sát chi phí. Một câu `print("đã trả lời xong")` không có các trường để truy vấn hoặc cộng số liệu đó.

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
| 1 stage (bản đầu, `Dockerfile.single-stage`) | 1.73 GB |
| Multi-stage (`Dockerfile`) | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi lấy Dockerfile một stage gốc từ commit `306b897` thành `Dockerfile.single-stage`, build `day12-agent:single`; `docker images` trả `1.73GB`. Bản multi-stage build thành `day12-agent:cp2-test` trả `310MB`. Chênh lệch khoảng 1.42 GB chủ yếu do base `python:3.11` đầy đủ nặng hơn `python:3.11-slim`, và bản một stage giữ toàn bộ source, công cụ, dependency/cache cài đặt trong image. Bản mới chỉ chuyển virtualenv từ builder sang runtime slim và không mang builder sang image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile hiện `COPY requirements.txt` rồi cài thư viện ở stage builder trước khi `COPY app`. Khi chỉ sửa `app/main.py`, layer base image, requirements và `pip install` vẫn được cache; layer `COPY app` cùng các layer theo sau phải chạy lại. Log `docker compose up -d --build` vừa rồi hiển thị bước cài thư viện `CACHED`. Nếu `COPY . .` trước `RUN pip install`, mỗi lần sửa source sẽ làm layer cài thư viện mất cache và build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app có lỗ hổng cho chạy lệnh, kẻ tấn công có quyền của process trong container. Chạy bằng root khiến quyền đó cao hơn; nếu còn có cấu hình mount nhạy cảm hoặc lỗ hổng thoát container, thiệt hại trên host có thể lớn. `USER agent` (UID 10001) khiến process chỉ có quyền user thường trong container, giảm quyền ngay ở bước chạy lệnh trái phép. Nó giảm rủi ro chứ không tự bảo đảm không thể thoát container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request vào cuối phút thứ nhất và 10 request vào đầu phút kế tiếp, chỉ cách nhau khoảng 2 giây nhưng mỗi phút đồng hồ vẫn đúng hạn mức 10. Sliding window 60 giây nhìn toàn bộ 20 request trong cùng một cửa sổ nên sẽ chặn sau 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit chặn số lần gọi trong 60 giây; cost guard chặn tổng chi phí giả lập theo user trong tháng. Ví dụ một request rất dài đến khi ngân sách tháng chỉ còn rất ít: tần suất còn dưới 10/phút nhưng cost guard phải chặn. Ngược lại, 11 request rất ngắn trong một phút có tổng chi phí vẫn thấp hơn 10 USD: cost guard cho qua nhưng rate limit trả 429 ở lượt 11. Tôi đã quan sát trên Railway 10 lượt đầu trả 200 và 5 lượt sau trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp và dùng Redis trong liveness, Redis mất kết nối 30 giây làm probe của cả 3 container báo lỗi; nền tảng có thể restart cả 3 dù code vẫn chạy. Việc restart liên tục không sửa Redis và còn làm mất khả năng phục vụ ngay khi Redis hồi phục. Tách `/health` để báo process còn sống và `/ready` để báo dependency sẵn sàng giúp nền tảng tạm ngừng đưa request vào instance chưa sẵn sàng mà không restart vô ích.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy Compose agent + Redis, rồi khởi chạy thêm hai agent trong cùng network vì file Compose map cố định host port 8000. Cùng một `X-User-Id`, gọi lần lượt vào ba container cho `history_length` là `0`, `2`, `4`: lịch sử nằm ở Redis dùng chung. Nếu mỗi container dùng dict Python riêng thì khi chuyển sang container khác sẽ trở về `0` hoặc tăng theo chuỗi riêng từng container; restart cũng làm mất lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi cloud thực tế là deployment đầu tiên `Crashed`. Trong Deploy Logs, Pydantic báo `ValidationError: agent_api_key Field required` vì Railway Variables chưa có `AGENT_API_KEY`, trong khi `Settings` bắt buộc biến này. Tôi thêm khóa vào Variables trên Railway, nối `REDIS_URL` tới Redis service, đặt healthcheck `/ready`, rồi Deploy. Agent và Redis chuyển sang Online; `/health` và `/ready` đều trả 200, CP5 có 9 test pass.
