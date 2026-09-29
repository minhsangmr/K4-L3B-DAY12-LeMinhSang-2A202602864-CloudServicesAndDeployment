# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời trực tiếp bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Minh Sang  Mã học viên: 2A202602864

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường cloud (như Railway hoặc Render) hay môi trường staging/production, lập trình viên quên khai báo biến môi trường AGENT_API_KEY trong dashboard quản lý.
- Nếu để giá trị mặc định là "changeme": Service vẫn khởi động bình thường, health check vẫn báo healthy/200, nhưng app đang hoạt động với API key công khai và cực kỳ yếu. Bất kỳ kẻ tấn công hoặc bot quét tự động nào thử các key phổ biến như "changeme" đều có thể truy cập trái phép vào endpoint /ask, gửi hàng loạt request tiêu tốn chi phí LLM, làm cạn kiệt ngân sách hoặc đánh cắp dữ liệu trước khi đội ngũ phát hiện ra.
- Lợi ích của Fail Fast: Do agent_api_key là trường bắt buộc không có giá trị mặc định, Pydantic sẽ ném ngoại lệ ValidationError ngay trong quá trình khởi tạo cấu hình lúc startup và dừng process ngay lập tức. Orchestrator / CI/CD sẽ nhận diện container crashed/unhealthy, ngăn chặn việc đẩy traffic vào một instance không an toàn và cảnh báo ngay cho kỹ sư devops sửa lỗi trước khi dịch vụ được public ra Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:06:34.120000+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`

Hai việc làm được với dòng log có cấu trúc (Structured Logging) mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn, lọc và phân tích tự động theo trường dữ liệu (Structured Querying & Aggregation)**: Các hệ thống thu thập log tập trung (Datadog, Grafana Loki, CloudWatch, ELK) có thể tự động parse chuỗi JSON thành các thuộc tính độc lập để truy vấn chính xác: ví dụ lọc tất cả request của riêng `user_id="sv-test"`, tính tổng chi phí `cost_usd` trong tháng, hoặc thống kê phân phối số lượng token (`tokens_in`, `tokens_out`) mà không cần viết các regex phức tạp và dễ gãy.
2. **Thiết lập cảnh báo tự động (Alerting & Anomaly Detection)**: Có thể cấu hình cảnh báo tự động kích hoạt khi có request tiêu tốn `cost_usd` vượt ngưỡng cho phép, hoặc phát hiện các event có `level="error"` tăng đột biến theo thời gian thực để kích hoạt quy trình ứng cứu sự cố bảo mật và tài chính ngay lập tức.

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
| 1 stage (bản đầu) | ~1040 MB |
| Multi-stage | 272 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~768 MB) bao gồm:
1. Base image đầy đủ (`python:3.11`) chứa toàn bộ bộ công cụ biên dịch C/C++ (`build-essential`, `gcc`, `g++`, `make`), các thư viện phát triển header (development headers), tài liệu hướng dẫn (man pages), và các gói tiện ích hệ thống Debian đầy đủ.
2. Trong mô hình multi-stage build, stage `builder` được sử dụng để cài đặt và biên dịch thư viện vào thư mục tạm `/install`, sau đó stage `runtime` (dùng base image `python:3.11-slim` chỉ gồm kernel và runtime tối giản) chỉ copy các file wheel/package đã được biên dịch xong sang `/usr/local`. Toàn bộ trình biên dịch, cache apt, build tools và các file trung gian không bị mang sang image cuối, giúp giảm hơn 70% dung lượng image, tăng tốc độ pull/deploy trên cloud và thu hẹp tối đa diện tích bề mặt tấn công (attack surface).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`:
  - Các layer được dùng lại từ cache (CACHED): Các bước khởi tạo base image, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...` ở stage builder; và `WORKDIR /app`, `COPY --from=builder ...`, `RUN useradd ...` ở stage runtime.
  - Layer phải chạy lại: Chỉ có layer `COPY . .` và các lệnh sau nó (`USER appuser`, `HEALTHCHECK`, `CMD`) bị invalidate cache và phải chạy lại (thời gian build chỉ mất dưới 1 giây).
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi lần sửa bất kỳ một ký tự nào trong mã nguồn ứng dụng, layer `COPY . .` sẽ bị thay đổi mã checksum, làm mất hiệu lực toàn bộ cache của các layer phía sau.
  - Kết quả là lệnh `RUN pip install` sẽ bị ép chạy lại từ đầu: tải lại toàn bộ dependencies từ PyPI, giải nén và biên dịch lại. Việc này khiến thời gian build tăng vọt từ 1 giây lên vài phút trong mỗi lần thay đổi code, làm giảm đáng kể hiệu suất của quy trình CI/CD.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện leo thang quyền:
  1. Kẻ tấn công phát hiện và khai thác một lỗ hổng trong mã nguồn Python (ví dụ Remote Code Execution qua deserialization không an toàn, command injection qua `os.system`, hoặc lỗ hổng upload file).
  2. Mã độc được thực thi dưới danh tính process của container. Do container mặc định chạy bằng `root` (UID 0), kẻ tấn công ngay lập tức có toàn quyền quản trị cao nhất bên trong container filesystem.
  3. Kẻ tấn công khai thác các lỗ hổng nhân Linux (kernel vulnerabilities như Dirty COW, Dirty Pipe), truy cập các socket mount nhạy cảm (như `/var/run/docker.sock` hoặc thư mục `/proc`, `/sys`) để thực hiện container breakout. Do UID 0 trong container mặc định ánh xạ trực tiếp tới UID 0 (root) trên máy host, kẻ tấn công chiếm toàn quyền kiểm soát máy chủ host.
- Lệnh `USER` cắt đứt chuỗi ở đâu:
  Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi ngay tại Bước 2. Khi mã khai thác được kích hoạt, process chỉ có quyền hạn của một người dùng thông thường không có đặc quyền. Kẻ tấn công không thể ghi vào các thư mục hệ thống (`/etc`, `/bin`, `/usr`), không thể can thiệp các socket kernel hay docker daemon, và bị chặn hoàn toàn các capability cần thiết để thực hiện container breakout ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa có thể gửi trong 2 giây liên tiếp: **20 requests**.
- Giải thích cách đạt được:
  - Với cơ chế đếm theo phút đồng hồ (Fixed Window Counter), bộ đếm tự động reset về 0 tại giây thứ 00 của mỗi phút (ví dụ 10:00:00, 10:01:00,...).
  - Người dùng có thể gửi dồn dập 10 requests ở giây cuối cùng của phút thứ nhất (10:00:59). Do trong phút 10:00 người này chưa gửi request nào, hệ thống chấp nhận cả 10 requests.
  - Ngay 1 giây sau đó (10:01:00), hệ thống bước sang phút mới và reset bộ đếm về 0. Người dùng lập tức gửi tiếp 10 requests nữa tại giây 10:01:00. Hệ thống kiểm tra thấy phút 10:01 chỉ có 10 requests nên tiếp tục cho qua.
  - Kết quả: Trong khoảng thời gian chỉ vỏn vẹn 2 giây (10:00:59 - 10:01:00), người dùng đã gửi thành công 20 requests, gấp đôi hạn mức 10 req/phút. Thuật toán Sliding Window với Redis Sorted Set khắc phục triệt để vấn đề này nhờ luôn tính chính xác số request trong khoảng thời gian trôi liên tục 60 giây gần nhất (`now - 60s`).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau:
  - **Rate Limit**: Giới hạn về **tần suất / số lượng requests** trong một cửa sổ thời gian ngắn (ví dụ 10 requests/phút) nhằm chống tấn công từ chối dịch vụ (DoS/Spam), nghẽn mạng và bảo vệ tính sẵn sàng của hạ tầng.
  - **Cost Guard**: Giới hạn về **chi phí tài chính tích lũy** theo chu kỳ ngân sách (ví dụ $10.0/tháng) nhằm bảo vệ rủi ro thâm hụt tài chính do các request tiêu tốn quá nhiều token của LLM.
- Tình huống Rate limit cho qua nhưng Cost guard phải chặn:
  - Người dùng mới chỉ gửi 1 request trong phút (thỏa mãn hạn mức 10 req/phút). Tuy nhiên, người dùng này đã tiêu $9.99 trên tổng ngân sách tháng $10.00. Request tiếp theo có chi phí ước tính là $0.05. Khi đó, tổng chi phí vượt quá ngân sách tháng -> Cost Guard chặn lại và trả về lỗi HTTP 402 Payment Required.
- Tình huống Cost guard cho qua nhưng Rate limit phải chặn:
  - Người dùng mới tiêu $0.10 trong tháng (ngân sách còn dư $9.90). Tuy nhiên, người dùng chạy một script tự động gửi liên tiếp 15 requests siêu ngắn (mỗi request chỉ vài token) trong vòng 2 giây. Chi phí phát sinh là không đáng kể nên Cost Guard cho qua, nhưng Rate Limiter phát hiện số lượng request đã vượt quá 10 req/phút -> Rate Limiter chặn lại và trả về HTTP 429 Too Many Requests kèm header Retry-After.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện diễn ra khi gộp chung endpoint và kiểm tra Redis:
1. **Redis gặp sự cố**: Kết nối mạng tới Redis bị ngắt hoặc Redis server khởi động lại trong 30 giây.
2. **Health check đồng loạt thất bại**: Orchestrator (Docker/Kubernetes/Platform) thực hiện kiểm tra định kỳ endpoint `/health` trên cả 3 container agent. Do `/health` gọi tới Redis đang mất kết nối, cả 3 container đều phản hồi lỗi 503 hoặc timeout.
3. **Orchestrator kích hoạt cơ chế tự phục hồi sai lầm**: Vì liveness probe thất bại, orchestrator kết luận rằng cả 3 container process đã bị treo/hỏng và gửi tín hiệu SIGKILL để khởi động lại (restart) toàn bộ 3 container.
4. **Sập hệ thống theo chuỗi (Cascading Failure / CrashLoopBackOff)**: Các container mới khởi động lại tiếp tục kiểm tra Redis trong khi Redis chưa kịp hồi phục. Cả 3 container lại fail health check và bị restart liên tục theo vòng lặp. Toàn bộ cụm dịch vụ sập hoàn toàn, không thể xử lý bất kỳ request nào.
5. **Ưu điểm khi tách biệt**: Nếu tách biệt, `/health` chỉ kiểm tra process nội bộ (trả 200 giúp container không bị restart oan), trong khi `/ready` kiểm tra Redis (trả 503 để load balancer tạm thời ngắt traffic vào container cho đến khi Redis kết nối lại thành công sau 30 giây).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis (Stateless):
  Mọi container agent đều đọc và ghi dữ liệu vào Redis List tập trung, do đó `history_length` tăng đều đặn và chính xác qua từng lượt hội thoại: 0, 2, 4, 6, 8,... bất kể request được xử lý bởi container nào.
- Nếu lịch sử được lưu trong dict Python (Stateful trong RAM của process):
  - Do load balancer phân phối các request tuần tự tới các container khác nhau (agent 1, agent 2, agent 3):
    - Request 1 đến Agent 1: `history_length` = 0 (Agent 1 ghi vào RAM của nó).
    - Request 2 đến Agent 2: `history_length` = 0 (vì RAM của Agent 2 chưa hề có lịch sử của user này).
    - Request 3 đến Agent 3: `history_length` = 0 (RAM Agent 3 hoàn toàn rỗng).
    - Request 4 quay lại Agent 1: `history_length` = 2.
    - Request 5 đến Agent 2: `history_length` = 2.
  - Người dùng sẽ thấy `history_length` nhảy giật cục, không đồng nhất, và mô hình LLM có phản hồi như bị "mất trí nhớ", không nắm bắt được ngữ cảnh các câu hỏi trước đó do trạng thái bị phân mảnh giữa các vùng nhớ độc lập của từng process container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**:
  `AssertionError: badge chưa báo passing. Vào tab Actions trên GitHub xem log lần chạy gần nhất (badge hiện đang là: no status).` xảy ra trong quá trình chạy GitHub Actions CI workflow lần đầu tiên.
- **Cách tìm ra nguyên nhân**:
  - Dùng GitHub CLI `gh run list --repo minhsangmr/K4-L3B-DAY12-LeMinhSang-2A202602864-CloudServicesAndDeployment` phát hiện commit đầu tiên đẩy lên thì run bị fail (`conclusion: failure`).
  - Chạy `gh run view <run-id> --log-failed` để phân tích log chi tiết của bước `Run pytest`.
  - Phân tích nguyên nhân: Trong workflow CI, lệnh `pytest --ignore=tests/test_cp5.py -v` đã chạy cả file `tests/test_bonus_cicd.py`. File này chứa test `test_badge_bao_passing`, vốn gửi HTTP request lên GitHub để kiểm tra xem badge trạng thái của workflow đã báo `passing` chưa. Vì đây là lần đầu tiên workflow được kích hoạt, workflow đang chạy dở và chưa từng có run nào thành công trước đó, nên GitHub trả về SVG badge với nội dung `<title>ci/cd pipeline - no status</title>`. Test `assert "passing" in content` bị rớt, tạo nên nghịch lý phụ thuộc vòng lặp (CI kiểm tra xem CI đã từng pass hay chưa).
- **Cách sửa**:
  - Cập nhật lệnh chạy test trong file `.github/workflows/ci.yml` thành:
    `pytest --ignore=tests/test_cp5.py -k "not test_badge_bao_passing" -v`
  - Nhờ đó, trong môi trường GitHub Actions CI, các bài test chức năng logic (CP1, CP2, CP3, CP4, build Docker) vẫn chạy đầy đủ 100%, nhưng loại trừ bước tự kiểm tra badge của chính nó. Sau khi push bản sửa này lên main, GitHub Actions hoàn thành màu xanh (`✓`), và badge chuyển sang trạng thái `passing`. Khi chạy test local `pytest tests/test_bonus_cicd.py -v`, test badge gọi lên GitHub thấy badge `passing` và vượt qua 100%.
