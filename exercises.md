# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng gợi ý (in nghiêng) phía dưới mỗi câu hỏi bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Quốc Khánh  Mã học viên: 2A202603013

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên Railway lần đầu, tôi quên set biến AGENT_API_KEY trước khi chạy `railway up`. Nếu app có mặc định "changeme", nó sẽ khởi động bình thường và endpoint /ask vẫn chạy—bất kỳ ai gõ đúng chuỗi "changeme" đều có thể gọi LLM mà không xác thực thực sự. Tôi chỉ phát hiện ra khi nhìn hóa đơn hoặc log bất thường, lúc đó đã bị lạm dụng. Vì không có mặc định, Pydantic ValidationError được ném ngay lúc khởi động, container không lên được, dashboard Railway báo lỗi ngay lập tức (crash loop), buộc tôi phải xem logs và fix ngay lập tức trước khi ai có cơ hội lạm dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:18:46.552274+00:00", "user_id": "user-test", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
```

Với dòng log JSON này, tôi có thể: (1) Dùng jq hoặc Python parse để lọc theo field (ví dụ `jq 'select(.cost_usd > 0.001)' logs.json` để tìm request tốn hơn 0.001$) và tính tổng cost_usd theo user trong ngày mà không cần regex đoán mò; (2) Đẩy log này vào hệ thống tập trung như Datadog hoặc CloudWatch, dựng alert tự động khi field `cost_usd` vượt ngưỡng hay `tokens_out` quá lớn, mà dạng print() thường không có field tách bạch để máy xử lý được.

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
| 1 stage (bản đầu, `python:3.11` full) | 1.73 GB |
| Multi-stage (`python:3.11-slim`, 2 stage) | 271 MB |

Đo bằng `docker images | grep agent` sau khi build cả hai từ cùng source code. Chênh lệch khoảng 1.46GB đó chủ yếu là: (1) base image `python:3.11` đầy đủ đã nặng ~1GB hơn `python:3.11-slim` vì mang theo nhiều package hệ thống, compiler (gcc), header files, dev tools không cần lúc chạy; (2) ở bản multi-stage, những công cụ build đó chỉ tồn tại trong stage `builder` tạm thời — stage `runtime` chỉ `COPY --from=builder /install /usr/local` lấy đúng phần thư viện Python đã cài xong, không mang theo compiler hay cache của pip. Stage builder sau khi build xong bị Docker vứt bỏ hoàn toàn, không đóng góp gì vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa `app/main.py` rồi build lại, layer `COPY requirements.txt` + `RUN pip install` vẫn "Using cache" (không thay đổi), nhưng layer `COPY app/` trở về sau phải chạy lại (vì nội dung app/ đã thay), khiến các layer sau cũng phải rebuild. Nếu tôi đặt `COPY . .` trước `RUN pip install`, Docker sẽ hash toàn bộ nội dung gốc—mỗi lần sửa 1 dòng code (dù không đụng requirements.txt) sẽ invalidate cache của lệnh COPY, dẫn tới cache của pip install cũng bị vô hiệu, phải cài lại toàn bộ thư viện mỗi lần build, rất chậm. Cách hiện tại (COPY requirements.txt trước) tối ưu hóa bằng cách tách nó thành layer riêng, chỉ rebuild khi thực sự cần cài gói mới.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: (1) Code Python có lỗ hổng RCE (deserialization, command injection...), (2) kẻ tấn công khai thác được để chạy lệnh tuỳ ý TRONG container, (3) vì process đang chạy bằng root, lệnh đó chạy với quyền root TRONG container, (4) nếu kết hợp một lỗ hổng thoát container (container escape, misconfiguration Docker daemon, hoặc mount nhầm socket Docker), root TRONG container có thể trở thành root TRÊN HOST—tức là người tấn công kiểm soát toàn bộ máy chủ. Lệnh `USER appuser` (uid 10001) cắt đứt chuỗi ở bước (3): dù code có lỗ hổng được khai thác, lệnh chỉ chạy được với quyền user thường, giới hạn thiệt hại kể cả khi bước (4) xảy ra.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Có thể gửi tối đa **20 request** trong 2 giây liên tiếp. Cách đạt được: user gửi 10 request lúc phút này giây 59 (lúc 10:00:59, còn 1 giây trước hết phút), các request này được đếm vào bucket của phút 00:xx. Ngay sau đó lúc phút kế tiếp giây 00 (10:01:00), bucket reset về 0, user có thể gửi thêm 10 request nữa—tổng 20 request trong vòng 2 giây (59s của phút này + 01s của phút kế) mà vẫn tuân theo rule (mỗi cửa sổ 60s đơn lẻ chỉ 10 request). Sliding window tránh được kẽ hở này vì nó đếm 60 giây GẦN NHẤT tính từ THỜI ĐIỂM request tới, không neo theo mốc phút cố định.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **SỐ LƯỢNG** request/phút (ví dụ 10 request/phút), cost guard giới hạn **TỔNG TIỀN**/tháng (ví dụ 10$). Chúng là hai trục độc lập. Tình huống rate limit cho qua nhưng cost guard chặn: user gửi đúng 5 request/phút (dưới 10) nhưng mỗi câu hỏi rất dài (50k token input) → vẫn dưới rate limit nhưng ngân sách tháng cạn nhanh vì mỗi request tốn ~0.05$, cost guard trả 402 khi dư dự toán không đủ. Tình huống ngược: user còn dư ngân sách rất nhiều (mới dùng 1$/10$ hạn) nhưng gửi request dồn dập 50 lần/giây với câu hỏi ngắn → cost guard không chặn (còn tiền) nhưng rate limit chặn ở request thứ 11 trong cửa sổ 60s, trả 429 Retry-After.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện: (1) Redis mất kết nối, (2) CẢ 3 container đồng loạt gọi endpoint gộp (coi là liveness probe) để check Redis, (3) CẢ 3 đều thấy Redis chết, trả 503 service unavailable, (4) vì 503 được coi là liveness probe failure, orchestrator (Kubernetes, Docker Swarm) hiểu là "các process này chết", RESTART CẢ 3 container CỰ LỮC CÙNG LÚC, (5) trong lúc cả 3 đang restart (vài giây để pull image, khởi động, chạy health check lại), không còn container nào sẵn sàng phục vụ → **toàn bộ service downtime hoàn toàn**, dù bản chất chỉ là Redis chập chờn 30 giây, root cause nhỏ bị khuếch đại thành sự cố toàn hệ thống. Tách /health (nhẹ, không check Redis) và /ready (check Redis) sẽ tránh được: liveness luôn 200, readiness 503 khi Redis chết, orchestrator chỉ evict traffic khỏi các container không sẵn sàng, không restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis (hiện tại), gọi `/ask` 5 lần liên tiếp với cùng `X-User-Id` qua Nginx phân phối vào 3 container khác nhau, `history_length` tăng dần đều: 0, 2, 4, 6, 8—vì lịch sử nằm trong Redis duy nhất (mọi container cùng đọc/ghi một nơi). Nếu lịch sử nằm trong một dict Python nội bộ của từng container: `history_length` sẽ KHÔNG tăng dần đều, mà nhảy lộn xộn hoặc thấp hơn thực tế—mỗi lần request rơi vào một container khác, container đó không có lịch sử cũ (dict rỗng riêng của nó) nên trả về `history_length` thấp hơn hẳn hoặc quay lại 0-2, tạo cảm giác agent "mất trí nhớ" ngẫu nhiên sau mỗi request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi: `railway up` báo "Deploy crashed", `railway logs` cho thấy `/bin/sh: 1: exec: docker-entrypoint.sh: not found` lặp lại liên tục (crash loop), domain generated là "redis-production-...", gọi vào bị 502. Nguyên nhân: tôi tạo `railway add --database redis` lần đầu, project chỉ có 1 service Redis. Khi chạy `railway up` để deploy app, CLI tưởng build Dockerfile là để cập nhật service Redis đó (vì nó là service duy nhất), nên deploy image Python/uvicorn ĐÈ lên service Redis → image mới không có script `docker-entrypoint.sh` của Redis, crash liên tục. Cách sửa: tạo service Railway MỚI tên "agent" riêng biệt (`railway add`/settings), link CLI vào đúng service agent, set biến REDIS_URL=`${{Redis.REDIS_URL}}` (tham chiếu nội bộ), rồi `railway up` lại. Đồng thời restore lại service Redis về image `redis:latest` gốc.
