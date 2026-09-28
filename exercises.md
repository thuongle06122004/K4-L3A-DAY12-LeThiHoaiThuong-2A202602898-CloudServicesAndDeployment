# Phiếu Phản Ánh --- K4 Level 3A, Ngày 12

Họ và tên: Lê Thị Hoài Thương 
Mã học viên: 2A202602898

------------------------------------------------------------------------

### Câu 1 --- Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết
ngay khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống
cụ thể mà việc "chết sớm" này cứu bạn, so với việc để mặc định
`"changeme"`.

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên ứng dụng
sẽ phát hiện lỗi ngay khi khởi động nếu biến môi trường bị thiếu.

Ví dụ, khi deploy lên Render nhưng quên cấu hình biến môi trường
`API_TOKEN`, ứng dụng có thể báo
`ValidationError: api_token Field required`, khiến Uvicorn không khởi
động và health check thất bại. Nhờ vậy, lỗi được phát hiện ngay trên
Render thay vì để ứng dụng chạy với cấu hình sai.

Nếu đặt mặc định như `"changeme"`, ứng dụng vẫn có thể khởi động và
`/healthz` trả về 200, khiến bản deploy trông như đã thành công. Chỉ khi
gọi `/chat` mới phát hiện token không hợp lệ và phải mất thêm thời gian
debug. Ngoài ra, một token mặc định dễ đoán cũng tạo ra rủi ro bảo mật
nếu bị bỏ quên trên môi trường production.

------------------------------------------------------------------------

### Câu 2 --- Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được,
rồi nêu **hai** việc bạn làm được với dòng log đó mà
`print("đã trả lời xong")` không làm được.

Sau khi gọi `/chat`, log JSON thu được có dạng:

``` json
{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:37:03.414344+00:00", "client_id": "sv-log", "prompt_tokens": 2, "completion_tokens": 34, "usd_cost": 2.07e-05}
```

Hai việc có thể làm với log có cấu trúc mà `print("đã trả lời xong")`
khó thực hiện:

1.  **Lọc và cảnh báo theo mức độ:** có thể lọc các log có
    `severity=ERROR` để nhanh chóng tìm các request gặp lỗi thay vì phải
    đọc toàn bộ log.

2.  **Phân tích theo từng trường dữ liệu:** có thể thống kê `usd_cost`
    theo `client_id`, đếm số request theo thời gian hoặc theo dõi lượng
    token sử dụng. Với `print()` thông thường, dữ liệu chỉ là một chuỗi
    văn bản nên khó truy vấn và phân tích tự động.

------------------------------------------------------------------------

### Câu 3 --- Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

``` bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

  Bản                 Dung lượng
  ------------------- ------------
  1 stage (bản đầu)   ... MB
  Multi-stage         ... MB

Giải thích: phần dung lượng chênh lệch đó là những gì?

  Bản                 Dung lượng
  ------------------- ------------
  1 stage (bản đầu)   \~1.8 GB
  Multi-stage         **270 MB**

Multi-stage giúp giảm khoảng **1.5 GB** nhờ loại bỏ các thành phần chỉ
cần trong quá trình build nhưng không cần khi chạy ứng dụng.

Ví dụ:

-   Compiler và các development headers như `gcc`, `build-essential`,
    `libpython-dev`.
-   Pip cache.
-   Các công cụ phục vụ quá trình build như một số thành phần của
    `setuptools`, `wheel`.
-   Các layer và dependency chỉ tồn tại trong `builder stage`.

Ở runtime stage, Docker chỉ copy những thành phần cần thiết từ builder
sang image cuối cùng, nên image nhỏ hơn đáng kể.

------------------------------------------------------------------------

### Câu 4 --- Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn,
những layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn
đặt `COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Ở lần build thứ hai, khi image đã tồn tại, nhiều layer được Docker sử
dụng lại từ cache:

``` text
=> CACHED [builder 3/4] COPY requirements.txt .
=> CACHED [builder 4/4] RUN pip install --no-cache-dir --prefix=/insta...
=> CACHED [runtime 4/6] COPY app ./app
=> CACHED [runtime 5/6] COPY utils ./utils
=> CACHED [runtime 6/6] RUN useradd --create-home --uid 10001 appuser
```

Nếu chỉ sửa một ký tự trong `app/main.py`, layer `COPY app ./app` sẽ bị
invalid cache và phải chạy lại. Các layer trước đó vẫn có thể sử dụng
cache vì input của chúng không thay đổi.

Điều này cho thấy thứ tự các lệnh trong Dockerfile ảnh hưởng trực tiếp
đến thời gian build.

Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần thay đổi một file
bất kỳ trong project, chẳng hạn `README.md` hoặc test file, Docker có
thể phải chạy lại các layer phía sau, bao gồm cả `pip install`. Điều này
khiến quá trình build mất thêm thời gian dù dependency không hề thay
đổi.

------------------------------------------------------------------------

### Câu 5 --- Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ
hổng trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy
host", và lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Container không nên chạy ứng dụng bằng `root` vì nếu ứng dụng tồn tại lỗ
hổng và attacker thực thi được code tùy ý, quyền của process trong
container sẽ chính là quyền mà ứng dụng đang chạy.

Nếu chạy bằng root, mức độ ảnh hưởng của một lỗ hổng có thể lớn hơn đáng
kể, đặc biệt nếu kết hợp với một lỗi cấu hình container hoặc
vulnerability ở tầng kernel/runtime.

Việc sử dụng:

``` dockerfile
USER appuser
```

với UID không đặc quyền, chẳng hạn `10001`, giúp giới hạn quyền của
process. Khi ứng dụng bị khai thác, attacker chỉ có quyền tương ứng với
`appuser` thay vì toàn quyền root trong container.

Điều này không loại bỏ hoàn toàn nguy cơ container escape, nhưng giúp
**giảm đáng kể quyền và phạm vi thiệt hại** nếu ứng dụng bị compromise.

------------------------------------------------------------------------

### Câu 6 --- Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm
theo phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa
bao nhiêu request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải
thích cách đạt được con số đó.

Nếu rate limit là **10 request/phút** nhưng sử dụng cách đếm theo phút
trên đồng hồ và reset tại giây `00`, một người dùng có thể gửi tối đa
**20 request trong khoảng 2 giây**.

Ví dụ:

-   Từ `12:00:59` đến `12:00:59.xxx`: gửi 10 request.
-   Khi đồng hồ chuyển sang `12:01:00`, bộ đếm được reset.
-   Ngay sau đó gửi thêm 10 request.

Như vậy có thể gửi:

**10 request ở cuối phút trước + 10 request ở đầu phút sau = 20 request
trong khoảng 1--2 giây.**

Sliding window khắc phục hiện tượng này bằng cách xét **60 giây gần nhất
tính từ thời điểm request**, thay vì phụ thuộc vào ranh giới `:00` của
đồng hồ.

------------------------------------------------------------------------

### Câu 7 --- Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit
cho qua nhưng cost guard phải chặn, và một tình huống ngược lại.

Hai cơ chế đều giúp bảo vệ hệ thống nhưng kiểm soát **hai loại giới hạn
khác nhau**.

-   **Rate limit:** giới hạn số lượng request trong một khoảng thời
    gian.
-   **Cost guard:** giới hạn chi phí hoặc lượng tài nguyên được phép sử
    dụng, chẳng hạn số token hoặc số tiền gọi LLM.

**Trường hợp rate limit cho qua nhưng cost guard phải chặn:**

Một user chỉ gửi vài request nên chưa vượt giới hạn request/phút, nhưng
mỗi request chứa prompt rất dài và tạo ra rất nhiều token. Tổng chi phí
vượt ngân sách → cost guard phải chặn.

**Trường hợp ngược lại:**

Một user gửi rất nhiều request nhỏ và rẻ trong thời gian ngắn. Tổng chi
phí vẫn nằm dưới ngân sách nên cost guard chưa cần chặn, nhưng số
request vượt giới hạn/phút → rate limit phải chặn.

Vì vậy, hai cơ chế bổ sung cho nhau: **rate limit bảo vệ theo tần suất,
còn cost guard bảo vệ theo chi phí/tài nguyên.**

------------------------------------------------------------------------

### Câu 8 --- /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra
với cụm 3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ
tự sự kiện.

Nếu dùng chung một endpoint và endpoint đó kiểm tra cả Redis, khi Redis
mất kết nối 30 giây, chuỗi sự kiện có thể xảy ra như sau:

1.  Redis mất kết nối trong khoảng 30 giây.
2.  Cả 3 container gọi kiểm tra Redis và đều thất bại.
3.  Endpoint health của cả 3 container trả `503`.
4.  Orchestrator thấy cả 3 instance đều unhealthy và có thể restart
    chúng.
5.  Các container mới khởi động trong lúc Redis vẫn đang phục hồi tiếp
    tục trả `503`.
6.  Orchestrator có thể tiếp tục restart container, tạo thành
    restart/crash loop.
7.  Khi Redis phục hồi, các container mới có thể phải khởi động lại và
    service mới trở lại trạng thái bình thường.

Như vậy, một sự cố Redis ngắn có thể gây ra thời gian gián đoạn dài hơn
nếu health check và readiness check bị gộp làm một.

Do đó nên tách `/healthz` để kiểm tra process còn sống và `/readyz` để
kiểm tra ứng dụng đã sẵn sàng nhận traffic, bao gồm các dependency cần
thiết như Redis.

------------------------------------------------------------------------

### Câu 9 --- Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với
cùng một `X-User-Id`. Quan sát `history_length` trong response. Nếu lịch
sử được lưu trong một dict Python thay vì Redis, bạn sẽ thấy con số đó
thay đổi thế nào?

Nếu history được lưu trong một `dict` Python thay vì Redis, mỗi
container sẽ có **một bản dict riêng trong memory**.

Khi gửi nhiều request với cùng `X-User-Id`, request có thể được phân
phối đến các container khác nhau nên `history_length` sẽ không tăng liên
tục theo một lịch sử chung.

Ví dụ:

``` text
Request 1 → agent-1 → history_length = 1
Request 2 → agent-2 → history_length = 1
Request 3 → agent-1 → history_length = 2
Request 4 → agent-3 → history_length = 1
```

Kết quả phụ thuộc vào request được gửi đến instance nào.

Nếu lưu history trong Redis, cả 3 container cùng truy cập một nơi lưu
trữ chung nên lịch sử của cùng một `X-User-Id` được duy trì nhất quán
hơn.

Điều này cho thấy khi scale nhiều replica, state quan trọng không nên
chỉ được lưu trong memory của từng container.

------------------------------------------------------------------------

### Câu 10 --- Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health
check timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi
là gì, bạn tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Một lỗi gặp khi kiểm tra service bằng PowerShell là request tới `/chat`
trả về lỗi `422 Unprocessable Entity` với thông báo `JSON decode error`.

Nguyên nhân là PowerShell xử lý lệnh `curl` và JSON body khác với môi
trường Bash. Trong PowerShell, `curl` có thể được ánh xạ tới
`Invoke-WebRequest`, đồng thời việc truyền JSON trực tiếp trên command
line có thể gây vấn đề về quotation và encoding.

Cách xử lý là dùng `curl.exe` để gọi curl thực sự và đưa JSON body vào
một file riêng:

``` powershell
'{"message":"Render la gi"}' | Set-Content body.json -Encoding utf8

curl.exe -i -X POST https://day12-chat-1lr6.onrender.com/chat `
  -H "Content-Type: application/json; charset=utf-8" `
  -H "Authorization: Bearer $TOKEN" `
  -H "X-Client-Id: sv-test" `
  --data-binary "@body.json"
```

Sau khi sửa, request trả về `HTTP/1.1 200 OK` cùng response JSON chứa
nội dung phản hồi và `client_id`.

Qua lỗi này, mình nhận ra khi kiểm thử API trên cloud cần chú ý không
chỉ đến code của application mà còn đến shell đang sử dụng, cách truyền
request body và encoding.
