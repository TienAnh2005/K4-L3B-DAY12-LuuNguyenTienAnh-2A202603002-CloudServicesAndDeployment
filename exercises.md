# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lưu Nguyễn Tiến Anh  Mã học viên: 2A202603002

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường cloud (Railway/Render) hoặc staging mà quên cấu hình biến môi trường `AGENT_API_KEY`.
- Nếu để giá trị mặc định `"changeme"`: Service vẫn khởi động bình thường và mở cổng ra Internet. Kẻ tấn công hoặc bot quét API có thể dễ dàng đoán được khóa `"changeme"` để gọi API miễn phí, tiêu tốn hạn mức và ngân sách LLM của hệ thống mà ta không hề hay biết cho tới khi nhận hóa đơn.
- Ngược lại, khi không có giá trị mặc định (fail fast): Pydantic sẽ ném `ValidationError` ngay lúc khởi động container. Service sẽ dừng ngay lập tức tại thời điểm deploy khi ta còn đang theo dõi log trên dashboard, buộc ta phải cấu hình đúng secret trước khi hệ thống phục vụ người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T12:40:35.836058+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`

Hai việc làm được với dòng log JSON:
1. Phân tích & truy vấn theo trường (Structured Querying): Cho phép các hệ thống gom log (như Datadog, CloudWatch, Loki) tự động parse các trường có kiểu dữ liệu rõ ràng để lọc theo `user_id`, tính tổng chi phí `cost_usd` theo khung giờ, hoặc xác định user nào tiêu tốn nhiều token nhất.
2. Thiết lập cảnh báo tự động (Alerting & Metrics): Dễ dàng tạo dashboard theo dõi thời gian thực (ví dụ đếm số lượng event lỗi `level="error"` trong 5 phút qua) và kích hoạt cảnh báo khi chi phí hoặc lượng token vượt ngưỡng bất thường.

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
| 1 stage (bản đầu) | ~1.05 GB |
| Multi-stage | ~271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~780 MB) bao gồm:
- Bản đầu dùng base image `python:3.11` đầy đủ vốn chứa rất nhiều công cụ hệ thống, gói thư viện hệ điều hành (build-essential, compilers, debuggers, manpages...). Bản multi-stage chuyển sang dùng base image `python:3.11-slim` cho runtime chỉ giữ lại những gì tối thiểu.
- Multi-stage tách riêng stage `builder` để cài đặt các package và chỉ `COPY --from=builder /install /usr/local` sang stage runtime; loại bỏ toàn bộ cache của pip, file tạm build và các công cụ biên dịch không cần thiết khi chạy thực tế.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`: Các layer cài đặt thư viện (`COPY requirements.txt .` và `RUN pip install ...`) nằm phía trước đều được Docker tái sử dụng lại từ cache (`CACHED`). Chỉ từ layer `COPY app ./app` trở đi mới bị invalidate cache và phải chạy lại, quá trình build chỉ mất chưa đầy vài giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ file nào trong source code (dù chỉ là 1 ký tự), layer copy toàn bộ mã nguồn sẽ bị đổi checksum, làm mất hiệu lực toàn bộ cache từ bước đó trở đi. Khi đó Docker buộc phải tải và cài đặt lại toàn bộ các thư viện trong `requirements.txt` từ đầu ở mỗi lần build, gây lãng phí thời gian và băng thông.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: Giả sử ứng dụng Python có lỗ hổng bảo mật (ví dụ: Remote Code Execution - RCE qua deserialization hoặc command injection). Khi container chạy bằng `root`, tiến trình Python có UID 0 bên trong container. Nếu kẻ tấn công khai thác được lỗi này và tận dụng một lỗ hổng container breakout (như qua kernel exploit hoặc mount socket docker/host file), tiến trình thoát ra ngoài máy host vẫn sẽ giữ nguyên quyền UID 0 (root). Kẻ tấn công sẽ chiếm toàn quyền điều khiển máy chủ host, đánh cắp dữ liệu và can thiệp vào các container khác.
- Lệnh `USER appuser` cắt đứt chuỗi đó ngay tại bước đầu tiên: Container chạy dưới quyền của một user không đặc quyền (non-root, UID 10001). Ngay cả khi chiếm được shell trong app Python, kẻ tấn công chỉ có quyền hạn chế của `appuser`, không thể ghi đè các file hệ thống quan trọng trong container và không thể tận dụng quyền root để tấn công leo thang ra host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Giải thích: Với cách đếm theo phút đồng hồ (reset ở giây 00), người dùng gửi 10 request vào giây cuối cùng của phút trước (10:00:59) — lúc này vẫn đúng hạn mức 10/phút. Ngay 1 giây sau, khi đồng hồ chuyển sang phút mới (10:01:00), bộ đếm bị reset về 0, người dùng gửi tiếp 10 request nữa. Kết quả là trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã phải chịu 20 request mà không hề chặn, gây nguy cơ spike traffic làm nghẽn server. Thuật toán Sliding Window (cửa sổ trượt 60s liên tục) khắc phục triệt để lỗ hổng này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác nhau:
  - Rate limit giới hạn tần suất/số lượng request trong một đơn vị thời gian ngắn (ví dụ: 10 request/phút) để bảo vệ server khỏi quá tải hoặc tấn công DoS.
  - Cost guard giới hạn ngân sách/chi phí tiền tệ ($) tích lũy trong chu kỳ dài (ví dụ: $10.0/tháng) để kiểm soát tổng chi phí sử dụng API của LLM.
- Tình huống Rate limit cho qua nhưng Cost guard chặn: Người dùng mới gọi 1 request trong phút (rate limit chưa vượt), nhưng request đó kèm prompt cực dài (50.000 token) hoặc tài khoản đã tiêu hết $10.0 của tháng trước đó → Cost guard phát hiện vượt budget và chặn (HTTP 402).
- Tình huống Cost guard cho qua nhưng Rate limit chặn: Người dùng còn nguyên ngân sách $10.0 chưa tiêu, nhưng gửi dồn dập 15 request ngắn (chỉ vài token mỗi câu) trong vòng 5 giây → Cost guard thấy số tiền rất nhỏ nên cho qua, nhưng Rate limiter phát hiện tần suất vượt quá 10 req/phút nên chặn (HTTP 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis mất kết nối → endpoint health/ready trả về mã lỗi 503.
2. Orchestrator (Docker/Kubernetes/Cloud) coi endpoint này là liveness check: thấy trả 503 nên kết luận cả 3 container agent đều đã hỏng.
3. Orchestrator tiến hành kill và restart liên tục cả 3 container agent cùng lúc để cố gắng tự phục hồi.
4. Sau 30s khi Redis hoạt động trở lại, các container agent vẫn đang trong vòng lặp restart (crash loop) hoặc chưa kịp khởi động xong, khiến toàn bộ hệ thống bị gián đoạn hoàn toàn (downtime). Sự cố tạm thời của một dependency (Redis) đã bị biến thành sự cố sập toàn bộ cụm ứng dụng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Nếu lưu trong dict Python của process: Khi scale lên 3 container agent (A, B, C), mỗi container có một vùng nhớ RAM hoàn toàn độc lập. Load balancer sẽ phân phối các request kế tiếp của cùng một user luân phiên vào các container khác nhau.
- Kết quả thấy được: `history_length` sẽ nhảy lộn xộn thay vì tăng đều đặn. Ví dụ: request 1 vào A (history = 0), request 2 vào B (history = 0 vì B chưa lưu gì), request 3 vào A (history = 2), request 4 vào C (history = 0). Agent sẽ bị hiện tượng "mất trí nhớ ngẫu nhiên". Khi dùng Redis chung, mọi container đều đọc/ghi cùng một nguồn state nên `history_length` luôn tăng đều đặn (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Lỗi gặp phải: Khi deploy service Agent lên Railway, ban đầu gặp thông báo lỗi "There was an error deploying from source" và không sinh được Public Networking domain.
- Cách tìm ra nguyên nhân: Kiểm tra tab Deployments và mục Variables của service trên Railway dashboard, phát hiện service thiếu các biến môi trường cấu hình (đặc biệt là `AGENT_API_KEY` khiến Pydantic Settings fail-fast ngay khi khởi động) và biến `REDIS_URL` chưa được liên kết với instance Redis.
- Cách sửa: Tạo service Redis trên Railway, sau đó trong tab Variables của Agent service cấu hình đầy đủ: `AGENT_API_KEY`, gán `REDIS_URL=${{Redis.REDIS_URL}}`, `RATE_LIMIT_PER_MINUTE=10`, `MONTHLY_BUDGET_USD=10.0`, `LOG_LEVEL=INFO`. Sau khi cấu hình xong, Railway tự động build và deploy thành công service, sau đó sinh public domain HTTPS hoạt động ổn định.
