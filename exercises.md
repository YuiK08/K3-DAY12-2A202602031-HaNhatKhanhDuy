# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> [Câu trả lời của bạn]` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hà Nhật Khánh Duy  Mã học viên: 2A202602031

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định là "changeme", hệ thống vẫn khởi động bình thường trên production dù ta quên cấu hình biến môi trường thật. Điều này tạo ra lỗ hổng bảo mật nghiêm trọng: bất kỳ ai biết được chuỗi "changeme" cũng có thể gọi API hợp lệ và làm tiêu tốn toàn bộ ngân sách LLM của ta. Việc "chết sớm" (crash) giúp ta phát hiện ngay lỗi cấu hình trước khi ứng dụng kịp phơi mình ra internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON: `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T12:00:00Z", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 10, "cost_usd": 0.0000078}`. 
> Hai việc làm được: 1) Có thể dễ dàng nhập vào các hệ thống như Elasticsearch/Datadog để vẽ biểu đồ tổng chi phí (sum cost_usd) theo ngày mà không cần viết Regex phức tạp. 2) Dùng các tool như `jq` để lọc nhanh tất cả các request của một `user_id` cụ thể phục vụ mục đích debug.

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
| 1 stage (bản đầu) | ~1000 MB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch chính là toàn bộ toolchain dùng để biên dịch (gcc, make, trình biên dịch C/C++), mã nguồn dư thừa, cùng với các file cache cài đặt thư viện của `pip`. Trong Multi-stage, chúng ta chỉ giữ lại đúng phần thư viện Python đã biên dịch hoàn chỉnh để đưa vào bản runtime, bỏ lại toàn bộ rác thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Nếu để `pip install` trước `COPY . .`, khi sửa code ở `app/main.py`, Docker chỉ chạy lại từ lệnh `COPY . .` trở đi, việc cài đặt thư viện (rất tốn thời gian) được lấy từ cache nên build lại rất nhanh. Ngược lại, nếu đặt `COPY . .` lên trước, mỗi khi đổi code, layer `COPY` bị thay đổi kéo theo MỌI layer bên dưới (bao gồm `RUN pip install`) bị invalidate (hủy cache) và phải chạy lại cài đặt toàn bộ dependencies từ đầu, làm tăng đáng kể thời gian build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: Nếu app có lỗ hổng (như RCE), hacker chạy được lệnh bên trong container với quyền `root` (do mặc định). Từ quyền `root` đó, hacker có thể lợi dụng các kẽ hở hoặc lỗi cấu hình (như bind mount nhạy cảm) để thực hiện "container escape", nhảy ra ngoài chiếm luôn quyền của máy host vật lý. Lệnh `USER appuser` cắt đứt chuỗi này ngay từ đầu: nó ép ứng dụng chỉ chạy dưới tư cách một người dùng thường (UID 10001), khi hacker RCE được thì cũng chỉ có đặc quyền hạn hẹp, không thể phá vỡ lớp sandbox của Docker.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người có thể gửi tối đa 20 request trong 2 giây. Ví dụ: họ gửi 10 request lúc 10:00:59, hệ thống cho qua. Sang 10:01:00 (vừa nhảy sang phút mới), bộ đếm bị reset về 0, và họ lập tức gửi tiếp 10 request ở 10:01:01. Hệ thống đếm bằng phút đồng hồ vẫn coi đây là hợp lệ, tạo ra lỗ hổng traffic gấp đôi (burst) trong khoảng thời gian rất ngắn. Cửa sổ trượt (sliding window) sẽ khắc phục hoàn toàn điểm yếu này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate Limit chặn theo TẦN SUẤT (tốc độ gửi request), còn Cost Guard chặn theo SỐ TIỀN THỰC TẾ (tích lũy cả tháng).
> - Tình huống Rate Limit cho qua nhưng Cost Guard chặn: User chỉ gửi 1 request (rất chậm), nhưng câu hỏi dài cả trăm trang tài liệu tốn tới 11 USD (vượt hạn mức 10 USD/tháng). Cost Guard sẽ kích hoạt.
> - Tình huống Cost Guard cho qua nhưng Rate limit chặn: Đầu tháng, user vừa được cấp ngân sách nên tiền chi tiêu bằng 0. Nhưng họ spam 50 câu hỏi liên tục chỉ trong 1 giây. Tiền chưa hết, nhưng để tránh hệ thống bị quá tải, Rate Limit sẽ chặn từ request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1) Redis bị gián đoạn kết nối.
> 2) `ping()` thất bại, `/health` trả về 503 cho CẢ 3 container.
> 3) Orchestrator (K8s/Docker) tưởng ứng dụng bị treo, tiến hành SIGTERM (hoặc SIGKILL) và khởi động lại ĐỒNG LOẠT 3 container đó.
> 4) Khi Redis có kết nối lại ở giây 30, toàn bộ ứng dụng đang trong quá trình restart boot-up, không có bất kỳ container nào sẵn sàng để gánh traffic, khiến hệ thống sập cục bộ (Cascading failure).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Con số `history_length` sẽ nhảy lộn xộn hoặc ngẫu nhiên (ví dụ: 1, 1, 2, 1, 3...). Do Load Balancer (Nginx/Traefik) rải đều các request của ta ngẫu nhiên vào 3 container khác nhau, mỗi container chỉ giữ 1 phần mảnh ghép lịch sử hội thoại trong bộ nhớ RAM (dict) cục bộ của nó. Điều này dẫn tới hiện tượng bot "mất trí nhớ" tạm thời giữa các luồng câu hỏi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi: API `/ready` trả về `503 Service Unavailable` và response `{"status": "not ready", "redis": false}` sau khi ứng dụng báo deploy thành công trên Render.
> Nguyên nhân: Thông báo log chỉ điểm rõ `ping()` của Redis thất bại. Kiểm tra tab Environment trên Render thì phát hiện chưa thêm biến môi trường `REDIS_URL`.
> Sửa chữa: Thêm Add-on Redis vào nền tảng, copy link URL cung cấp và gán giá trị cho biến `REDIS_URL` trên dashboard Render, sau đó trigger lại quá trình deploy.
