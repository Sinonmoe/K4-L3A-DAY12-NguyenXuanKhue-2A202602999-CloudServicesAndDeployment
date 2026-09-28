# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Xuân Khuê  Mã học viên: 2A202602999

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

**Tình huống cụ thể:**
Khi deploy ứng dụng lên môi trường production (hoặc staging), dev/DevOps quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard nhà cung cấp cloud (hoặc trong file `.env.production`).

- **Nếu để mặc định `"changeme"`:** App vẫn khởi động bình thường và báo trạng thái healthy. Lúc này, bất kỳ ai hoặc bot scan tự động trên Internet cũng có thể dùng header `Authorization: Bearer changeme` để gọi liên tục vào endpoint `/ask`. Ứng dụng liên tục gọi mô hình LLM bên dưới, tiêu tốn hàng nghìn USD tiền quota API của dự án. Lỗi này thường chỉ bị phát hiện khi hóa đơn cuối tháng gửi về hoặc khi tài khoản API LLM bị khóa vì cạn kiệt ngân sách.
- **Khi không có mặc định (Fail Fast):** Nhờ cơ chế validation của Pydantic, khi thiếu `AGENT_API_KEY`, ứng dụng ném ngoại lệ `ValidationError` và dừng ngay trong vài giây đầu tiên của quá trình khởi động (startup). Quá trình deploy fail lập tức, cloud orchestrator (Railway/Kubernetes/Docker) phát hiện container không khởi động được và gửi cảnh báo đỏ cho dev. Điều này ngăn chặn triệt để việc đưa một ứng dụng bảo mật kém lên môi trường production, bảo vệ an toàn cho ngân sách và dữ liệu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

- **Dòng log JSON thực tế thu được từ service:**
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:54:26.123456+00:00", "user_id": "test-user-01", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0000309, "duration_ms": 124.5}
```

- **Hai việc làm được với dòng log có cấu trúc (structured JSON) mà `print()` văn bản thuần túy không làm được:**
  1. **Lọc, tìm kiếm và thiết lập cảnh báo tự động trên hệ thống giám sát tập trung (Log Aggregator như Datadog, ELK Stack, Grafana Loki, CloudWatch):** Máy tính có thể parse JSON thành các trường dữ liệu riêng biệt. Ta có thể dễ dàng thiết lập rule cảnh báo tự động: `if cost_usd > 0.05` hoặc lọc tìm kiếm tức thì `user_id == "test-user-01" AND level == "error"`. Với `print()`, log là chuỗi text không cấu trúc, rất khó parse chính xác bằng regex và dễ sai sót khi format thay đổi.
  2. **Thống kê, vẽ biểu đồ và tổng hợp số liệu định lượng (Metrics & Analytics):** Hệ thống giám sát có thể tự động tính toán tổng chi phí (`SUM(cost_usd)`), trung bình token tiêu thụ (`AVG(tokens_in + tokens_out)`), hoặc latency p95/p99 (`duration_ms`) theo từng `user_id` hay theo từng khoảng thời gian trong ngày. Chuỗi `print("đã trả lời xong")` hoàn toàn không chứa dữ liệu định lượng để tổng hợp hay phục vụ việc thanh toán/audit.

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
| 1 stage (bản đầu, `python:3.11` full) | 1015 MB |
| Multi-stage (`python:3.11-slim` + wheels) | 271 MB |

**Giải thích phần dung lượng chênh lệch (~744 MB):**
Phần dung lượng chênh lệch khổng lồ này bao gồm:
1. **Base OS packages đầy đủ:** Base image `python:3.11` tiêu chuẩn dựa trên bản Debian đầy đủ, đi kèm hàng trăm tiện ích hệ thống, thư viện C, man pages, tài liệu hướng dẫn và công cụ dòng lệnh không dùng tới ở runtime. Trong khi đó, `python:3.11-slim` chỉ giữ lại các package tối thiểu để chạy Linux.
2. **Công cụ biên dịch và build tools:** Bản 1-stage chứa đầy đủ trình biên dịch `gcc`, `g++`, `make`, cùng các tệp C/C++ header files (`python3-dev`, `libc-dev`, v.v.) cần thiết khi build các thư viện Python có C-extension. Ở multi-stage build, toàn bộ công cụ này nằm lại ở stage `builder` và bị loại bỏ hoàn toàn khỏi image cuối.
3. **Cache của trình quản lý gói:** Bộ nhớ tạm của `apt` (`/var/lib/apt/lists/*`) và cache bánh xe wheel của `pip` (`~/.cache/pip`) trong quá trình cài đặt package đã được loại bỏ (nhờ dùng cờ `--no-cache-dir` và chỉ copy artifact đã cài đặt từ `/install` sang final stage).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Khi sửa một ký tự trong `app/main.py` với Dockerfile hiện tại:**
  - **Các layer được dùng lại từ cache (`CACHED`):**
    - `FROM python:3.11-slim as builder`
    - `WORKDIR /build`
    - `COPY requirements.txt .`
    - `RUN pip install --prefix=/install --no-cache-dir -r requirements.txt`
    - `FROM python:3.11-slim as final`
    - `RUN useradd -m -u 10001 -s /bin/bash appuser`
    - `COPY --from=builder /install /usr/local`
    - `WORKDIR /app`
  - **Các layer phải chạy lại:**
    - `COPY app ./app` (vì nội dung trong thư mục `app/` đã thay đổi, Docker invalidate cache từ bước này).
    - `USER appuser` và các chỉ thị tiếp theo (`EXPOSE`, `CMD`).
  - **Thời gian build:** Cực nhanh (dưới 1 giây) vì không phải tải và cài đặt lại bất kỳ package nào trong `requirements.txt`.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi khi sửa bất kỳ file mã nguồn nào (dù chỉ 1 ký tự trong `app/main.py`), cache của layer `COPY . .` bị vô hiệu hóa (bị bust).
  - Do Docker build tuần tự từ trên xuống dưới, việc cache bị vỡ tại `COPY . .` sẽ buộc Docker phải thực thi lại toàn bộ các bước phía sau nó, bao gồm cả `RUN pip install -r requirements.txt`.
  - Kết quả là lập trình viên phải chờ hàng phút để Docker tải và build lại các package Python mỗi khi sửa code, làm chậm quy trình CI/CD và vòng lặp phát triển cục bộ.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện leo thang đặc quyền từ code tới máy host:**
  1. **Khai thác lỗ hổng ứng dụng:** Code Python có lỗ hổng (ví dụ Remote Code Execution qua `eval()`, `pickle.loads()`, Command Injection qua `os.system()`, hoặc Path Traversal qua tải file).
  2. **Chiếm shell trong container:** Kẻ tấn công gửi payload kích hoạt lỗ hổng, mở được một reverse shell hoặc thực thi lệnh tùy ý bên trong container.
  3. **Thực thi dưới quyền root container:** Vì container mặc định chạy không khai báo `USER`, tiến trình Python chạy với quyền `root` (UID 0 bên trong container namespace).
  4. **Thoát container (Container Breakout) ra máy host:** Khi có quyền root trong container, kẻ tấn công dễ dàng khai thác các cấu hình hớ hênh hoặc lỗ hổng Linux kernel/container runtime. Ví dụ:
     - Nếu container mount Docker socket (`/var/run/docker.sock`), kẻ tấn công có UID 0 có thể ra lệnh cho Docker daemon trên host spawn một container mới mount toàn bộ thư mục `/` của host.
     - Nếu có mount volume từ host với quyền ghi (như `/etc` hay crontab), root container có thể ghi đè các file hệ thống của host.
     - Khai thác lỗ hổng kernel (như Dirty COW, runc CVE-2019-5736...). Vì UID 0 trong container thường ánh xạ trực tiếp tới UID 0 trên Linux host (nếu không bật user namespace remap).
  5. **Chiếm toàn quyền máy host:** Kẻ tấn công có quyền root trên host OS, kiểm soát toàn bộ server vật lý/máy ảo và các container khác.
- **Lệnh `USER appuser` cắt đứt chuỗi ở chỗ nào:**
  - Lệnh `USER appuser` chuyển quyền thực thi của tiến trình sang người dùng không có đặc quyền (unprivileged user, UID 10001).
  - Nó **cắt đứt chuỗi ngay tại Bước 3**: Dù kẻ tấn công khai thác thành công lỗ hổng trong code Python và thực thi lệnh, shell thu được chỉ có quyền hạn của `appuser` (UID 10001).
  - Kẻ tấn công không thể ghi vào các thư mục hệ thống của container (`/etc`, `/usr`), không thể can thiệp các thiết bị/socket gắn ngoài (như `docker.sock` đòi quyền root), và không đủ đặc quyền (capabilities như `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`) để thực hiện các cuộc tấn công container breakout.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số request tối đa trong 2 giây liên tiếp:** **20 request**.
- **Giải thích cách đạt được con số đó (Fixed Window Bursting):**
  - Giả sử hệ thống dùng cơ chế Fixed Window (cửa sổ cố định) reset bộ đếm vào đúng giây thứ 00 của mỗi phút (ví dụ từ 10:00:00 đến 10:00:59 là một khung, từ 10:01:00 đến 10:01:59 là khung tiếp theo).
  - **Giây thứ 1 (10:00:59):** Người dùng gửi dồn dập 10 request. Vì trong khung phút 10:00:xx trước đó chưa gửi request nào, toàn bộ 10 request này được chấp thuận (bộ đếm đạt 10/10).
  - **Ngay 1 giây sau (10:01:00):** Đồng hồ chuyển sang phút mới, bộ đếm bị reset về 0. Người dùng gửi tiếp 10 request nữa. Khung phút mới 10:01:xx ghi nhận 10/10 request hợp lệ.
  - Tổng cộng: Chỉ trong vòng 2 giây liên tiếp (từ 10:00:59 đến 10:01:00), hệ thống đã phải hứng chịu **20 request** (gấp đôi hạn mức 10 request/phút) mà cơ chế fixed window không hề phát hiện hay ngăn chặn.
  - *Ngược lại, Sliding Window (cửa sổ trượt)* lưu timestamp của từng request (dùng Redis Sorted Set). Khi kiểm tra ở thời điểm 10:01:00, nó sẽ đếm tất cả request trong khoảng `[10:00:00, 10:01:00]`. Do 10 request lúc 10:00:59 vẫn nằm trong khoảng 60s này, 10 request gửi ở 10:01:00 sẽ bị chặn ngay lập tức (HTTP 429).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau cốt lõi giữa hai cơ chế:**
  - **Rate Limit (Giới hạn tần suất):** Đo lường số lượng request trong một đơn vị thời gian (ví dụ: tối đa 10 req/phút). Mục tiêu là bảo vệ tài nguyên hạ tầng server, chống tấn công từ chối dịch vụ (DoS/DDoS), spam kết nối và tránh quá tải CPU/RAM.
  - **Cost Guard (Bảo vệ ngân sách/chi phí):** Đo lường số tiền (hoặc tổng token quy đổi thành USD) tích lũy trong một chu kỳ (ví dụ: tối đa $1.0/tháng/user). Mục tiêu là quản trị rủi ro tài chính, ngăn ngừa cạn kiệt ngân sách API do các mô hình LLM tính phí theo lượng token tiêu thụ.
- **Tình huống Rate Limit cho qua nhưng Cost Guard phải chặn:**
  - User gửi **chỉ 1 request duy nhất trong cả giờ** (tần suất cực thấp, hoàn toàn vượt qua Rate Limit 10 req/phút). Nhưng trong request đó, user gửi kèm một file tài liệu khổng lồ chứa 200.000 tokens và yêu cầu LLM tạo bài tóm tắt dài. Chi phí ước tính cho request này là **$3.00**, trong khi hạn mức ngân sách còn lại của user chỉ là **$0.50**. Cost Guard phát hiện vượt ngân sách nên từ chối phục vụ ngay với mã lỗi `402 Payment Required`.
- **Tình huống Cost Guard cho qua nhưng Rate Limit phải chặn:**
  - User mới tạo tài khoản và còn nguyên hạn mức ngân sách $10.00. User dùng script gửi **30 request liên tiếp trong vòng 3 giây**, mỗi request chỉ là một câu chào đơn giản "hi" tốn 5 tokens (chi phí siêu nhỏ, chỉ khoảng $0.00001/req, tổng 30 req tốn chưa đến $0.001, rất xa hạn mức $10). Cost Guard kiểm tra thấy ngân sách dồi dào nên cho phép, nhưng Rate Limit phát hiện số request vượt quá ngưỡng 10 req/phút nên lập tức chặn từ request thứ 11 với mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp làm một và dùng chung endpoint đó cho cả Liveness probe và Readiness probe:
1. **Sự kiện 1 - Redis gặp sự cố:** Redis gặp sự cố mạng hoặc khởi động lại, mất kết nối trong 30 giây.
2. **Sự kiện 2 - Health check đồng loạt fail:** Liveness probe của orchestrator (Docker/Kubernetes/Cloud Load Balancer) định kỳ gửi request tới endpoint kiểm tra trên cả 3 container agent. Vì endpoint có kiểm tra Redis, cả 3 container đều trả về HTTP 503 Service Unavailable (unhealthy).
3. **Sự kiện 3 - Orchestrator restart toàn bộ container:** Do Liveness probe báo lỗi (orchestrator cho rằng tiến trình Python đã chết hoặc treo hoàn toàn), orchestrator lập tức gửi tín hiệu kill và restart đồng thời cả 3 container.
4. **Sự kiện 4 - Rơi vào vòng lặp chết chóc (CrashLoopBackOff):** Các container mới được bật lên, tiến trình khởi động và lại kiểm tra Redis để qua liveness check. Do Redis vẫn đang trong thời gian chết 30 giây, health check lại tiếp tục trả về 503. Orchestrator lại tiếp tục kill và restart container nhiều lần liên tiếp.
5. **Sự kiện 5 - Sập toàn bộ dịch vụ (Cascading Failure):** Toàn bộ hệ thống sập hoàn toàn (100% downtime), ngay cả các request tĩnh hoặc các tác vụ không phụ thuộc vào Redis cũng không thể xử lý. Khi Redis phục hồi sau 30 giây, toàn bộ cụm container vẫn đang bị nghẽn trong chu kỳ reboot và dồn ứ kết nối khởi động.

*Nguyên tắc đúng chuẩn Cloud-Native:*
- `/health` (Liveness probe): Chỉ kiểm tra tiến trình app có đang sống và phản hồi HTTP không (không check Redis) để tránh việc restart container oan uổng.
- `/ready` (Readiness probe): Kiểm tra sự sẵn sàng của các dependency (Redis, DB). Khi Redis rớt, `/ready` trả về 503 -> Load Balancer chỉ tạm thời ngắt route traffic vào container đó, không restart container, chờ Redis phục hồi là tự động nhận lại traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi lưu lịch sử bằng Redis (Stateless - hệ thống hiện tại):**
  - Cả 3 container agent đều truy cập vào cùng một kho lưu trữ chung là Redis.
  - Khi gọi `/ask` liên tiếp với cùng một `X-User-Id`, dù request được Load Balancer phân bổ ngẫu nhiên (round-robin) đến container 1, container 2 hay container 3, `history_length` luôn tăng một cách nhất quán và tuần tự: `0 -> 2 -> 4 -> 6 -> 8...` (mỗi lượt gồm 1 câu hỏi user và 1 câu trả lời assistant).
- **Nếu lưu lịch sử trong một dict Python trong RAM (Stateful):**
  - Mỗi container sẽ sở hữu một vùng nhớ RAM và một dictionary Python hoàn toàn độc lập, tách biệt nhau.
  - Khi người dùng gửi các request liên tiếp, Load Balancer phân tán các request đến các container khác nhau:
    - Request 1 trúng Container A: RAM của A chưa có gì -> `history_length = 0` (sau đó lưu vào dict của A).
    - Request 2 trúng Container B: RAM của B chưa có gì -> `history_length` lại là `0` thay vì `2`!
    - Request 3 trúng Container C: RAM của C chưa có gì -> `history_length` lại là `0`!
    - Request 4 trúng Container A: RAM của A đã có 2 tin nhắn từ request 1 -> `history_length = 2`.
    - Request 5 trúng Container B: RAM của B đã có 2 tin nhắn từ request 2 -> `history_length = 2`.
  - **Kết quả:** Con số `history_length` sẽ nhảy lộn xộn, lúc tăng lúc giảm hoặc reset về 0 một cách ngẫu nhiên tùy thuộc vào container nào tiếp nhận request. Agent bị hiện tượng "mất trí nhớ từng phần", câu trả lời của LLM bị mất ngữ cảnh hội thoại trước đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:**
  Container bị crash liên tục ngay khi vừa khởi động (Exit code 1 hoặc CrashLoopBackOff), log trên service báo:
  `ERROR: Application startup failed. Exiting.`
  `NotImplementedError: TODO (CP4): cài đặt install`
- **Cách tìm ra nguyên nhân:**
  1. Kiểm tra log của container thông qua lệnh `docker compose logs agent` (hoặc vào tab Deploy Logs / Runtime Logs trên Railway).
  2. Quan sát thấy lỗi ném ra từ `app/lifecycle.py`, dòng gọi hàm `install()` trong FastAPI lifespan context manager (`lifespan` tại `app/main.py`).
  3. Mở file `app/lifecycle.py` và thấy hàm `install()` vẫn còn đoạn placeholder `raise NotImplementedError("TODO (CP4): cài đặt install")` chưa được triển khai mã nguồn xử lý graceful shutdown.
- **Cách sửa:**
  1. Mở `app/lifecycle.py` và hoàn thiện hàm `install()`:
     - Khởi tạo cờ toàn cục `is_shutting_down = False`.
     - Đăng ký hàm xử lý signal `request_shutdown` với `signal.SIGTERM` và `signal.SIGINT`.
  2. Hoàn thiện hàm `request_shutdown(signum, frame)`: đặt cờ `is_shutting_down = True`, ghi log có cấu trúc sự kiện bắt đầu quá trình shutdown và cho phép hoàn thành các request đang dở dang trước khi tiến trình dừng hoàn toàn.
  3. Rebuild lại Docker image và deploy lại, container khởi động thành công, vượt qua health check và phản hồi HTTP 200 OK.
