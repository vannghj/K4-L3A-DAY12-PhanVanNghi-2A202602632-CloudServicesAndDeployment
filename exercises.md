# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phan Văn Nghị  Mã học viên: 2A202602632

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: mình deploy lên Railway nhưng quên thêm `AGENT_API_KEY` trong
> dashboard.
>
> - **Có mặc định `"changeme"`:** app khởi động bình thường, `/health` trả 200,
>   dashboard báo xanh, nên mình tưởng deploy đã xong. Nhưng repo này public, ai
>   đọc `config.py` cũng biết khóa là `"changeme"`. Bot quét Internet tìm thấy
>   URL mới và gọi `/ask` bằng khóa đó; mỗi request là tiền LLM của mình. Mình
>   chỉ phát hiện khi nhìn hóa đơn.
> - **Không có mặc định:** `Settings()` ném lỗi ngay khi được tạo. Mình thử
>   tạo `Settings` khi không có biến môi trường và nhận
>   `ValidationError: agent_api_key Field required`. Lỗi chỉ thẳng ra biến nào
>   bị thiếu, nên mình biết phải thêm gì vào dashboard thay vì đoán mò.
>
> Ở `docker-compose.yml` mình dùng `${AGENT_API_KEY:?...}` để fail fast từ tầng
> compose. Chạy compose khi thiếu khóa thì nó từ chối khởi động với thông báo
> `required variable AGENT_API_KEY is missing a value`. Nếu chỉ viết
> `${AGENT_API_KEY}`, biến thiếu sẽ thành chuỗi rỗng, pydantic vẫn chấp nhận
> chuỗi rỗng là một `str` hợp lệ, và app chạy với khóa `""`.
>
> Một điều mình quan sát được khi thử: container chạy thiếu khóa **vẫn khởi
> động và báo healthy**, vì `get_settings()` chỉ được gọi lần đầu khi có
> request vào `/ask` hoặc `/ready`, còn `/health` không đọc cấu hình. Lỗi
> `ValidationError` chỉ lộ ra ở request đầu tiên. Muốn chết ngay lúc khởi động
> thì phải gọi `get_settings()` trong `lifespan`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng lấy từ `docker compose logs agent` sau khi gọi `/ask` nhiều lần:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:55:43.822499+00:00", "user_id": "sv-rl", "tokens_in": 392, "tokens_out": 43, "cost_usd": 8.46e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và tổng hợp theo trường.** Vì mỗi dòng là một JSON, công cụ log
>    (hoặc chỉ cần `jq`) trả lời được câu "user nào tiêu nhiều tiền nhất hôm
>    nay?" bằng cách lọc `event == "ask_completed"`, nhóm theo `user_id` và
>    cộng `cost_usd`. Với `print` thì không có user, không có số tiền, chẳng
>    có gì để nhóm hay cộng.
> 2. **Phát hiện bất thường và đặt cảnh báo.** Trong chính các dòng log mình
>    thu được, `tokens_in` của cùng một user tăng dần 302 → 347 → 392 vì
>    history dài ra sau mỗi lượt. Có `timestamp` và `tokens_in` là dạng số thì
>    đặt được cảnh báo kiểu "`tokens_in` > 5000" hoặc "tỷ lệ `level == error`
>    trong 5 phút > 5%". Một chuỗi text tự do thì máy không đọc được để so
>    sánh.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB; nén 435 MB) |
| Multi-stage | 297 MB (nén 64 MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh khoảng 1.43 GB, nhỏ hơn ~5.8 lần. Mở từng image ra kiểm tra thì thấy:
>
> - **Base image là phần lớn nhất (~1.4 GB).** `python:3.11` bản đầy đủ là
>   Debian kèm cả bộ công cụ build: trong `agent:single` có sẵn `gcc`, `make`,
>   `git`, `curl` cùng header và thư viện dev. `python:3.11-slim` chỉ 215 MB và
>   trong `day12-agent:prod` không có công cụ nào ở trên.
> - **Pip cache (17 MB)** nằm lại ở `/root/.cache/pip` vì bản 1 stage chạy
>   `pip install` không có `--no-cache-dir`.
> - Phần giống nhau ở cả hai là thư viện Python đã cài (~77 MB trong
>   `site-packages`): 215 MB slim + ~80 MB thư viện và code ≈ 297 MB.
>
> Điều mình rút ra: ở bài này phần giảm chủ yếu đến từ việc đổi sang base
> `slim`, không phải từ multi-stage, vì stage `builder` không phải cài compiler
> (mọi thư viện trong `requirements.txt` đều có wheel dựng sẵn). Multi-stage
> phát huy tác dụng thật khi cần biên dịch thư viện (ví dụ `psycopg2`): compiler
> và file tạm nằm lại ở stage builder bị vứt đi, không lọt vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình đổi `SERVICE_VERSION` trong `app/main.py` rồi build lại với
> `--progress=plain`:
>
> - **Dùng lại cache (`CACHED`):** toàn bộ stage `builder` (`WORKDIR`,
>   `COPY requirements.txt`, `RUN pip install`), và ở stage runtime là
>   `COPY --from=builder`, `RUN useradd`, `WORKDIR /app`.
> - **Chạy lại:** chỉ `COPY app ./app` và `COPY utils ./utils` (layer `utils`
>   chạy lại vì Docker huỷ cache từ layer đầu tiên thay đổi trở đi). Cả lần
>   build mất khoảng 0.1 giây.
>
> Thử thêm một Dockerfile đặt `COPY . .` trước `pip install` và sửa một ký tự
> tương tự: `COPY . .` đổi checksum nên cache bị huỷ ngay tại đó, và
> `RUN pip install` phải cài lại toàn bộ thư viện, mất **13.3 giây** thay vì
> dùng cache. Mỗi lần sửa một dấu phẩy là mất chừng đó thời gian, và trên CI
> hoặc cloud còn phải tải lại thư viện từ PyPI.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy root:
>
> 1. Code Python có lỗ hổng cho phép thực thi lệnh (ví dụ đưa input của user
>    vào `subprocess`/`eval`, hoặc một thư viện bị lỗi deserialize).
> 2. Kẻ tấn công chạy được lệnh bên trong container **với uid 0**, vì process
>    uvicorn đang chạy bằng root.
> 3. Uid 0 trong container cũng là uid 0 trên kernel của host (Docker mặc định
>    không đổi uid). Với quyền root, kẻ tấn công có thể cài thêm công cụ, đọc
>    và ghi mọi file trong container, và ghi vào bất kỳ volume nào mount từ host
>    với quyền root.
> 4. Chỉ cần thêm một điểm yếu để thoát ra: container chạy `--privileged`, có
>    mount `/var/run/docker.sock`, hoặc một lỗi kernel. Khi đó kẻ tấn công ra tới
>    host với đúng quyền root, tức là chiếm toàn bộ máy.
>
> `USER appuser` cắt chuỗi ở **bước 2**: lệnh do kẻ tấn công chạy chỉ có quyền
> của uid 10001 (mình đã kiểm tra bằng `id` trong container:
> `uid=10001(appuser)`). User này không cài được gói, không ghi được file hệ
> thống, và nếu có thoát ra host thì cũng chỉ là một user thường không có
> quyền gì, không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **Tối đa 20 request trong 2 giây**, gấp đôi hạn mức.
>
> Cách đạt được: gửi 10 request lúc 10:00:59, đúng cuối phút 10:00, nên bộ
> đếm của phút này lên 10/10 nhưng vẫn hợp lệ. Sang 10:01:00 bộ đếm reset về 0,
> và gửi tiếp 10 request lúc 10:01:01, hợp lệ với phút mới. Kết quả là 20
> request trong khoảng 2 giây mà không vi phạm luật "10 request mỗi phút đồng
> hồ".
>
> Mình mô phỏng đúng kịch bản này bằng code: bộ đếm theo phút đồng hồ cho qua
> **20/20** request, còn `RateLimiter` sliding window của bài chỉ cho qua
> **10/20**. Lý do là lúc 10:01:01, cửa sổ trượt nhìn lại 60 giây (từ
> 10:00:01) và vẫn thấy 10 request lúc 10:00:59, nên chặn cả 10 request sau.
> Cửa sổ trượt không có "ranh giới" nào để lách qua.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau:** rate limit đo **tốc độ** (số request trong 60 giây gần nhất,
> trả 429, tự hồi phục sau vài giây). Cost guard đo **tổng tiền** (USD cộng dồn
> cả tháng theo key `cost:<user>:<YYYY-MM>`, trả 402, chỉ reset khi sang tháng
> mới). Một cái chống spam, một cái chống cháy ngân sách.
>
> **Rate limit cho qua, cost guard phải chặn:** một user gửi đều 5 request mỗi
> phút, luôn dưới hạn mức 10/phút, nhưng liên tục cả ngày với câu hỏi rất dài.
> Mỗi lượt đều tốn nhiều token, và history dài ra làm `tokens_in` tăng dần
> (log của mình: 302 → 347 → 392 chỉ sau vài lượt). Không lúc nào vi phạm rate
> limit nhưng tổng tiền vẫn vượt ngân sách tháng. Mình thử bằng cách đặt chi
> tiêu của `sv-rich` lên 10.5 USD trong Redis: request đầu tiên trong phút đó
> vẫn bị trả **402 monthly budget exceeded**.
>
> **Ngược lại, cost guard cho qua nhưng rate limit chặn:** một script gửi 15
> request liên tiếp trong chưa tới 1 giây. Mình chạy thử với `sv-rl`: kết quả
> là `200 ×10` rồi `429 ×5`, trong khi cả 10 request thành công chỉ tốn
> 0.00054 USD, còn rất xa ngân sách 10 USD. Tiền thì chưa có vấn đề, nhưng tốc
> độ đó là spam hoặc bot, nên rate limit chặn để bảo vệ tài nguyên cho user
> khác.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu gộp làm một endpoint có kiểm tra Redis:
>
> 1. **Giây 0:** Redis mất kết nối. Cả 3 container cùng dùng chung Redis đó,
>    nên endpoint gộp của cả 3 **cùng lúc** trả 503.
> 2. **Khoảng giây 10–30:** healthcheck (liveness) của orchestrator fail liên
>    tiếp đủ số lần `retries` và đánh dấu cả 3 container là unhealthy.
> 3. **Orchestrator restart cả 3 container cùng lúc**, vì với nó liveness fail
>    nghĩa là "process hỏng, cần khởi động lại". Mọi request đang xử lý dở bị
>    cắt ngang, và lúc này không còn instance nào phục vụ.
> 4. **Giây 30:** Redis quay lại, nhưng cả 3 container còn đang khởi động lại.
>    Nếu lúc khởi động chúng lại kiểm tra Redis thì có thể fail tiếp và rơi
>    vào vòng restart. Sự cố Redis 30 giây trở thành sự cố sập toàn hệ thống,
>    kéo dài hơn chính sự cố gốc.
>
> Khi tách riêng, mình đã chạy thử: `--scale agent=3` sau nginx, rồi
> `docker compose stop redis` trong 35 giây (hơn 3 lượt healthcheck 10 giây).
> Kết quả là cả 3 container đều `/health=200`, `/ready=503`, và Docker vẫn báo
> **`Up 47 seconds (healthy)`**, không container nào bị restart. `/ready` qua
> nginx trả `503 {"status":"not ready","redis":false}`, tức là load balancer
> chỉ ngừng gửi traffic. Bật Redis lại thì sau vài giây `/ready` trở về 200 và
> cả 3 container phục vụ tiếp ngay, không mất thời gian khởi động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy 3 instance `agent` sau nginx (dùng `nginx/nginx.conf` có sẵn) và
> gọi `/ask` 6 lần với cùng `X-User-Id`. Log cho thấy nginx chia vòng tròn:
> agent-1 → agent-2 → agent-3 → agent-1 → agent-2 → agent-3.
>
> - **History trong Redis:** `history_length` = `0 2 4 6 8 10`. Mỗi lượt lưu 2
>   message (câu hỏi và câu trả lời), nên tăng đều 2 dù mỗi lượt rơi vào một
>   container khác, vì cả 3 cùng đọc một Redis.
> - **History trong RAM của từng process:** để thử đúng trường hợp này, mình
>   đặt `REDIS_URL=fake://` cho cả 3 container, tức mỗi container có một Redis
>   giả nằm trong RAM của chính nó, giống hệt một dict Python. Kết quả là
>   `history_length` = **`0 0 0 2 2 2`**. Ba lượt đầu vào 3 container khác
>   nhau, container nào cũng thấy history rỗng. Lượt 4 quay lại container đầu
>   tiên, và nó chỉ nhớ đúng 1 lượt (2 message) mà chính nó đã xử lý.
>
> Với user, agent "mất trí nhớ" ngẫu nhiên tùy request rơi vào đâu. Container
> restart hoặc deploy bản mới cũng làm mất sạch lịch sử. Đó là lý do state
> phải nằm ngoài process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu lên Railway chạy thành công ngay: build, `$PORT` (Railway gán
> 8080), Redis và health check đều ổn. Vì vậy mình **cố ý tái hiện** lỗi "sai
> `REDIS_URL`" trên chính service đang chạy, bằng cách đặt
> `REDIS_URL=redis://localhost:6379/0`, đúng giá trị trong `.env` ở máy. Đây
> là lỗi dễ mắc nhất khi copy cấu hình từ máy lên cloud.
>
> **Triệu chứng:**
> - Railway vẫn báo deployment **`SUCCESS`**, vì health check của Railway gọi
>   `/health`, mà `/health` không chạm Redis. Nhìn dashboard thì không thấy gì
>   sai.
> - `/health` → 200, nhưng `/ready` → `503 {"status":"not ready","redis":false}`.
> - `/ask` có key → `500 Internal Server Error`.
> - `pytest tests/test_cp5.py` fail 2 test, với thông báo
>   `/ready trả 503 — nhiều khả năng biến REDIS_URL trên cloud chưa đúng`.
>
> **Tìm nguyên nhân:** `/ready` trả `"redis": false` cho biết app không nói
> chuyện được với Redis. `railway logs` cho thấy traceback của request `/ask`:
> `redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379.
> Connection refused.` Lỗi xảy ra ở `zremrangebyscore`, tức bước rate limiter,
> bước đầu tiên chạm Redis. `localhost` bên trong container trên Railway là
> chính container đó, và trong container không có Redis nào, nên kết nối bị
> từ chối. Chạy `railway variables` thì thấy đúng `REDIS_URL` đang là
> `localhost`.
>
> **Sửa:** đặt lại biến thành tham chiếu tới Redis add-on
> `railway variables --set 'REDIS_URL=${{Redis.REDIS_URL}}'`. Railway tự điền
> địa chỉ nội bộ `redis.railway.internal:6379` và redeploy. Sau đó `/ready` trở
> về 200 `{"status":"ready","redis":true}` và CP5 lại pass 9/9.
>
> Bài học: deployment `SUCCESS` chỉ nghĩa là liveness ổn. Sau mỗi lần deploy
> phải kiểm tra `/ready` (hoặc gọi thử `/ask`) mới biết service có thật sự
> phục vụ được không.
