# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thị Phương  Mã học viên: 2A202601315

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống em gặp thật hôm nay: deploy lên Railway và quên set `AGENT_API_KEY`
> trong dashboard. Vì `.dockerignore` loại `.env` ra khỏi image nên trong
> container hoàn toàn không có biến đó.
>
> Nếu `agent_api_key` có mặc định `"changeme"`: app khởi động bình thường,
> `/health` trả 200, Railway báo deploy thành công, mọi thứ trông như đã xong.
> Nhưng repo của em là public — ai đọc code cũng thấy chuỗi `"changeme"`, gửi
> header `X-API-Key: changeme` là gọi được `/ask`. Em chỉ phát hiện ra khi
> `MONTHLY_BUDGET_USD` cạn hoặc khi nhìn hóa đơn, tức là sau khi tiền đã mất.
> Cái nguy hiểm là hệ thống *im lặng* chạy sai chứ không báo lỗi.
>
> Không có mặc định thì pydantic ném `ValidationError` ngay lúc đọc config,
> deploy đỏ, health check trượt và Railway rollback về bản cũ. Em biết mình sai
> trong vòng 30 giây thay vì một tháng sau.
>
> Một điều em quan sát thêm ở code của mình: `get_settings()` được gọi *lười*
> (bên trong `verify_api_key`, lúc có request) chứ không phải lúc import. Nên
> thực tế app vẫn khởi động được và `/health` vẫn 200, chỉ `/ask` mới nổ 500.
> Muốn fail thật sự sớm thì phải gọi `get_settings()` ngay trong `lifespan`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật em thu được:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T07:16:52.782685+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
>
> **Việc 1 — cộng dồn và cảnh báo theo số liệu.** Vì `cost_usd` là một trường
> riêng có kiểu số, em có thể cho máy tính tổng chi phí theo từng `user_id`
> trong ngày, ví dụ lọc `event == "ask_completed"` rồi `sum(cost_usd)` group by
> `user_id`, và đặt cảnh báo khi một user vượt ngưỡng. Với `print("đã trả lời
> xong")` thì không có con số nào để cộng — muốn biết tốn bao nhiêu tiền phải
> đi hỏi từng người.
>
> **Việc 2 — truy vết một request cụ thể giữa hàng nghìn dòng.** `timestamp`
> theo chuẩn ISO-8601 kèm múi giờ UTC nên khi user báo "lúc 14h05 em bị lỗi",
> em lọc đúng khoảng thời gian đó và đúng `user_id` đó, kể cả khi 3 container
> cùng ghi log lẫn vào nhau trên Railway. Log dạng câu tiếng Việt thì phải viết
> regex để bóc thông tin, mà chỉ cần đổi câu chữ một lần là regex hỏng.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh khoảng 1.46 GB, gấp hơn 6 lần. Phần chênh gồm ba nhóm:
>
> **1. Base image.** Bản 1 stage dùng `python:3.11` đầy đủ (~1 GB), bản
> multi-stage dùng `python:3.11-slim` (~130 MB). Bản đầy đủ mang theo cả bộ
> công cụ biên dịch (`gcc`, `build-essential`), header của thư viện C, tài liệu,
> man page, và nhiều gói hệ thống mà một service FastAPI đã build xong không
> bao giờ dùng tới.
>
> **2. Rác của quá trình cài đặt.** Bản 1 stage chạy `pip install` thẳng vào
> image cuối nên cache của pip, các file `.whl` tải về và mọi thứ sinh ra lúc
> biên dịch đều nằm lại trong layer. Bản multi-stage cài bằng
> `--no-cache-dir --prefix=/install` ở stage `builder`, rồi stage `runtime`
> chỉ `COPY --from=builder /install /usr/local` — tức là chỉ bê **kết quả**
> sang, còn toàn bộ stage builder bị vứt đi và không xuất hiện trong image cuối.
>
> **3. Thứ không cần thiết bị copy vào.** Bản 1 stage dùng `COPY . .` nên nuốt
> cả những file chỉ phục vụ việc học (`LAB_GUIDE.md`, `tests/`, `screenshots/`).
> Bản multi-stage chỉ `COPY app ./app` và `COPY utils ./utils`.
>
> Ý nghĩa thực tế: 1.46 GB đó phải được push lên registry và pull về mỗi lần
> deploy. Đó là khác biệt giữa deploy 30 giây và deploy 5 phút.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Em thêm một ký tự vào `app/main.py` rồi `docker build` lại, output như sau:
>
> ```
> #6  COPY requirements.txt .                       CACHED
> #9  RUN pip install --prefix=/install ...         CACHED
> #7  COPY --from=builder /install /usr/local       CACHED
> #10 WORKDIR /app                                  CACHED
> #11 COPY app ./app                                chạy lại
> #12 COPY utils ./utils                            chạy lại
> #13 RUN useradd --create-home --uid 10001 appuser chạy lại
> ```
>
> Layer được dùng lại là toàn bộ phần cài thư viện; layer phải chạy lại bắt đầu
> từ `COPY app ./app` — đúng layer đầu tiên có nội dung thay đổi — và mọi layer
> đứng sau nó, kể cả `RUN useradd` dù lệnh đó chẳng liên quan gì tới code. Vì
> Docker cache theo dây chuyền: một layer hỏng thì tất cả layer phía sau cũng
> hỏng theo, do chúng được xây trên nền layer đó.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`: mỗi lần sửa một dấu phẩy trong
> code, layer `COPY` đổi checksum, kéo theo `pip install` mất cache và phải tải
> lại toàn bộ thư viện từ đầu — thêm 1–2 phút cho mỗi lần build, trong khi
> `requirements.txt` không đổi một chữ. Đây chính là lý do quy tắc "copy file
> ít thay đổi trước, file hay thay đổi sau" tồn tại: requirements.txt vài tuần
> mới sửa một lần, còn code thì sửa vài chục lần một ngày.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
>
> 1. Code Python có một lỗ hổng cho phép thực thi lệnh — ví dụ một thư viện
>    trong `requirements.txt` dính CVE deserialization, hoặc một endpoint nhận
>    đường dẫn từ user mà không kiểm tra.
> 2. Kẻ tấn công khai thác lỗ hổng đó và chạy được lệnh tuỳ ý. Lệnh này chạy
>    **với quyền của tiến trình uvicorn** — không hơn, không kém.
> 3. Nếu container chạy bằng root, tiến trình đó là uid 0. Kẻ tấn công lập tức
>    ghi được vào mọi thư mục trong container, cài thêm công cụ, sửa cả thư viện
>    Python trong `/usr/local` để cắm backdoor sống sót qua restart.
> 4. Từ root-trong-container sang root-trên-host: uid 0 trong container **cũng
>    là uid 0 với kernel của host** (trừ khi bật user namespace). Nên nếu host
>    có mount `/var/run/docker.sock` vào container, kẻ tấn công gọi Docker API
>    tạo một container mới mount nguyên ổ đĩa host — xong. Hoặc nếu có volume
>    mount thư mục host, hắn ghi thẳng vào đó. Hoặc khai thác một lỗ hổng thoát
>    container ở kernel, thứ hầu như luôn đòi quyền root mới dùng được.
>
> `USER appuser` cắt đứt ở **bước 3 sang bước 4**. Tiến trình chạy bằng uid
> 10001, nên khi kẻ tấn công chạy được lệnh thì hắn cũng chỉ là uid 10001: không
> ghi được vào `/usr/local`, không cài được gói, không đọc được file của root,
> và gần như mọi kỹ thuật thoát container đều không dùng được. Lỗ hổng vẫn còn
> đó và vẫn nghiêm trọng, nhưng thiệt hại bị nhốt lại trong phạm vi một user
> thường của một container — thay vì lan ra cả máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút?

> **20 request trong 2 giây.**
>
> Cách đạt được: gửi 10 request lúc 11:00:59 và 10 request nữa lúc 11:01:00.
> Cả 20 request đều hợp lệ dưới góc nhìn của cửa sổ cố định — 10 cái đầu thuộc
> phút 11:00, 10 cái sau thuộc phút 11:01, mỗi phút đều đúng hạn mức. Nhưng xét
> theo thời gian thực, server nhận 20 request trong khoảng 2 giây, tức gấp đôi
> hạn mức và dồn vào một khoảnh khắc. Đây gọi là hiệu ứng "boundary burst".
>
> Sliding window không có kẽ hở này vì nó không hỏi "phút này bạn gọi mấy lần"
> mà hỏi "60 giây vừa qua tính từ **bây giờ** bạn gọi mấy lần". Tại thời điểm
> 11:01:00, cửa sổ trải từ 11:00:00 đến 11:01:00 và đã thấy đủ 10 request
> trước đó, nên request thứ 11 bị chặn ngay.
>
> Em kiểm chứng trên bản deploy: gọi 15 lần liên tiếp với cùng `X-User-Id`,
> kết quả là `200 200 200 200 200 200 200 200 200 200 429 429 429 429 429` —
> đúng 10 lần đầu qua, 5 lần sau bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau ở **đơn vị đo và thứ chúng bảo vệ**. Rate limit đếm *số request
> trên một đơn vị thời gian ngắn* (10 req/phút) và bảo vệ khả năng phục vụ của
> hệ thống — chống spam, chống một user chiếm hết tài nguyên của người khác.
> Cost guard cộng dồn *số tiền trong một chu kỳ dài* (10 USD/tháng) và bảo vệ
> ví tiền. Một cái nhìn tốc độ, một cái nhìn tổng tích luỹ.
>
> **Rate limit cho qua nhưng cost guard phải chặn:** một user gửi mỗi 10 phút
> một request, nhưng mỗi câu hỏi kèm 50.000 token ngữ cảnh. Tốc độ 0.1 req/phút,
> rate limit không thấy gì bất thường cả. Nhưng sau vài chục request thì chi phí
> đã vượt ngân sách tháng. Rate limit hoàn toàn mù với chuyện này vì nó đếm
> *số lần*, không đếm *độ nặng của mỗi lần*.
>
> **Cost guard cho qua nhưng rate limit phải chặn:** đầu tháng, ngân sách còn
> nguyên 10 USD, một script lỗi gọi `/ask` 500 lần trong 10 giây với câu hỏi
> ngắn. Tổng tiền có thể chỉ vài xu nên cost guard thấy hoàn toàn ổn, nhưng
> 500 request đồng thời sẽ làm nghẽn service và những user khác không được phục
> vụ. Rate limit chặn từ request thứ 11.
>
> Vì vậy trong `/ask` em gọi `limiter.check()` rồi `guard.check()` **trước** khi
> gọi `ask_llm` — chặn sau khi đã gọi LLM thì vừa mất tiền vừa trả lỗi.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
>
> 1. **Giây 0:** Redis mất kết nối. Cả 3 container vẫn sống khoẻ, tiến trình
>    Python vẫn chạy, vẫn xử lý được mọi thứ không đụng tới Redis.
> 2. **Giây 0–10:** orchestrator gọi endpoint gộp trên từng container. Endpoint
>    này ping Redis, Redis không trả lời, nên nó trả 503 — **đồng loạt trên cả
>    3 container**, vì cả 3 cùng phụ thuộc một Redis.
> 3. **Giây ~10–30:** đủ số lần thất bại liên tiếp (`retries`), orchestrator kết
>    luận cả 3 container đã chết và **kill rồi restart cả 3 cùng lúc**.
> 4. **Trong lúc restart:** không còn instance nào phục vụ. Kể cả những request
>    không cần Redis cũng nhận lỗi 502. Đây là lúc downtime thật sự bắt đầu —
>    và nó do health check gây ra, không phải do Redis.
> 5. **Container khởi động lại:** chúng lại ping Redis, Redis vẫn chưa hồi, lại
>    503, lại bị kill. Vòng lặp crash-loop.
> 6. **Giây 30:** Redis hồi phục. Nhưng cụm container còn phải khởi động lại từ
>    đầu, nên tổng thời gian chết dài hơn 30 giây khá nhiều.
>
> Nghịch lý là: sự cố 30 giây của một dependency bị health check khuếch đại
> thành sự cố dài hơn của toàn hệ thống. Vì vậy `/health` của em chỉ trả lời
> "tiến trình còn sống không" và tuyệt đối không chạm vào Redis, còn `/ready`
> mới kiểm tra dependency. Load balancer dùng `/ready` để ngừng đẩy traffic vào
> instance chưa sẵn sàng — ngừng gửi traffic thì có thể phục hồi ngay khi Redis
> quay lại, còn restart container thì không.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Em gọi `/ask` hai lần liên tiếp trên bản deploy Railway với cùng
> `X-User-Id: sv-test`, `history_length` trả về lần lượt là **0 rồi 2** — mỗi
> lượt hỏi ghi thêm 2 message (một của user, một của assistant) và lượt sau
> đọc lại được đầy đủ những gì lượt trước ghi.
>
> Nếu lịch sử nằm trong một dict Python toàn cục thay vì Redis, mỗi container có
> vùng RAM riêng nên `history_length` sẽ **nhảy loạn theo container nào tình cờ
> nhận request**. Với 3 instance và load balancer chia vòng tròn, chuỗi quan sát
> sẽ đại loại là `0, 0, 0, 2, 2, 2, 4...` — ba lần đầu ra 0 vì mỗi container mới
> gặp user này lần đầu, và con số chỉ tăng khi request quay lại đúng container
> đã từng phục vụ. Người dùng thấy agent "mất trí nhớ" một cách ngẫu nhiên,
> giống hệt lỗi chập chờn khó tái hiện. Tệ hơn nữa, container bị restart lúc
> deploy là toàn bộ lịch sử biến mất, vì RAM không sống sót qua restart.
>
> Ghi chú khi làm thực tế: `docker compose up --scale agent=3` với
> `ports: "8000:8000"` sẽ lỗi trùng cổng, vì 3 container không thể cùng chiếm
> cổng 8000 trên host. Muốn scale thật phải bỏ `ports` của service `agent` và
> đặt nginx phía trước làm load balancer.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Thông báo lỗi.** Trên dashboard Railway, các bước Initialization → Build →
> Deploy đều xanh, nhưng bước cuối đỏ: `Network > Healthcheck — Healthcheck
> failure`, kèm `1/1 replicas never became healthy!`. Mở Deploy Logs thì thấy
> dòng này lặp lại 4 lần:
>
> ```
> Error: Invalid value for '--port': '${PORT:-8000}' is not a valid integer.
> ```
>
> **Cách tìm ra nguyên nhân.** Điều làm em chú ý là Build đã xanh — nghĩa là
> image không có vấn đề, lỗi nằm ở lúc *chạy*. Deploy Logs chỉ thẳng ra rằng
> uvicorn nhận được chuỗi ký tự `${PORT:-8000}` nguyên văn thay vì một con số.
> Đối chiếu lại thì trong `Dockerfile` em viết đúng cú pháp đó và nó chạy tốt ở
> máy, vì `CMD` của em bọc trong `sh -c` nên có shell nở biến. Nhưng
> `railway.toml` lại khai báo `startCommand`, và Railway chạy lệnh này **không
> qua shell** — không có shell thì `${PORT:-8000}` chỉ là một chuỗi bình thường.
> `startCommand` đã ghi đè `CMD` đúng của Dockerfile bằng một phiên bản hỏng.
> Việc container chết đi chết lại 4 lần cũng khớp với `restartPolicyMaxRetries = 3`.
>
> **Cách sửa.** Xoá hẳn dòng `startCommand` trong `railway.toml` để Railway dùng
> `CMD` của Dockerfile — cái đã có `sh -c` nên xử lý biến `PORT` đúng cách. Em
> nâng luôn `healthcheckTimeout` từ 30 lên 100 giây cho lần khởi động nguội.
> Sau khi push, log hiện `Uvicorn running on http://0.0.0.0:8080` (Railway tự
> gán `PORT=8080`) và `GET /health 200 OK`.
>
> Bài học em rút ra: cùng một chuỗi `${PORT:-8000}` đúng ở chỗ này và sai ở chỗ
> kia, tuỳ vào việc nó có đi qua shell hay không. Và Deploy Logs là nơi đầu tiên
> phải mở — bảng trạng thái trên dashboard chỉ nói bước nào hỏng, còn log mới
> nói vì sao.
