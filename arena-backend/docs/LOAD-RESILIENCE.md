# Kiểm thử tải và sự cố local

Chạy trên Linux/WSL trong `arena-backend`, sau khi Docker và image worker sẵn sàng:

```sh
make load-test
# Hoặc tăng đợt tải duy trì lên tối đa 60 giây:
.venv/bin/python tests/load_resilience.py --soak-seconds 30
```

Script tự tạo database tạm trong `test-results`, 6 đội giả lập và một server trên cổng localhost ngẫu nhiên. Server dùng đúng CLI hiện có (một process) nhưng đặt `ARENA_WORKERS=1` để kịch bản giết container đang chạy luôn xác định. Không dùng tài khoản, database, cổng 41810 hoặc kết quả giải đang sử dụng. Tiêm sự cố chỉ nhắm PID controller do script tạo và container có mount nằm trong đúng thư mục test. Không dừng Docker Desktop, không xóa image/volume.

## Các phép đo tải

- Đọc hỗn hợp `/teams/me`, dashboard, source, job, giải và config: 240 request ở mỗi mức 1, 8 và 24 client đồng thời.
- Duy trì tải đọc 12 client trong 15 giây mặc định. Đây là bài thử ngắn, không thay thế soak test nhiều giờ.
- Đợt 48 client đồng thời: nếu vượt giới hạn kết nối (`ARENA_CONNECTION_LIMIT`, mặc định 1024) thì HTTP 503 được ghi riêng, không tính là thành công. Sau đợt này phải tiếp tục phục vụ bình thường.
- 24 lần nộp lại cùng ZIP/version/idempotency key phải tạo đúng một bản ghi. Cùng key nhưng input khác phải trả 409.
- 30 request benchmark đồng thời (6 đội × 5 lần cùng một bài) khi worker đang bận: hàng đợi nằm trong SQLite, không có giới hạn chung nên cả 30 đều được nhận (202); các yêu cầu giống nhau của một đội dùng chung một job đang chờ, nên chỉ tạo đúng 6 job. Mỗi đội tối đa 4 job đang chạy và 4 yêu cầu đang chờ.
- 12 kết nối SSE thật giữ mở đồng thời với tải đọc API. Đóng người xem không được hủy job. Thu hồi phiên phải đóng stream và từ chối phiên cũ.

Mỗi client SSE trong bài thử mở một stream; đây không phải phép đo 12 trang Arena đầy đủ hoặc 12 người dùng trình duyệt. Các stream cũng chiếm slot admission của Uvicorn.

Báo cáo ghi số request, thời gian, throughput, P50/P95/P99/max, HTTP status và lỗi transport cho từng pha. RSS/peak RSS/thread/file descriptor đọc từ `/proc` của controller test; không phải tổng RAM cả Docker hoặc RSS riêng bot. Latency bao gồm client HTTP, kết nối mới và server trên cùng host. Client đóng keep-alive để các pha trước không để lại kết nối nhàn rỗi chiếm admission slot.

## Các sự cố

1. Build lỗi, debug làm hỏng stdout, sai turn ID, nước sai luật, bot crash, vượt giới hạn output: job phải có đúng mã lỗi; bot tốt chạy sau đó vẫn đạt đủ 72 mẫu.
2. Kill container đang benchmark: đây là lỗi hạ tầng `worker_error`, không bao giờ bị coi là bot hết giờ hay thua. Benchmark chưa có bàn cờ công khai nên worker tự chạy lại đúng một lần (sự kiện `infrastructure_retry`) và lần chạy lại phải đạt; các job đang chờ phải tiếp tục hoàn tất.
3. Kill controller bằng SIGKILL khi một trận chính thức đang chơi và một job còn chờ. Khởi động lại cùng DB: phiên đăng nhập còn hợp lệ; trận đang chơi `aborted/server_restart`, job còn chờ được giữ và chạy xong; giải chuyển `paused`; fixture lỗi cần BTC xử lý; điểm vẫn bằng 0.
4. BTC chủ động retry fixture và gọi start để tiếp tục. Hai lượt phải hoàn tất; attempt cũ còn lịch sử, không được cộng điểm hai lần.
5. ZIP hỏng, JSON trùng key, thiếu CSRF, truy cập admin bằng tài khoản đội hoặc đọc job đội khác phải bị từ chối. Đăng nhập sai nhiều lần trả 429 và `Retry-After`.
6. Khởi động controller test với tên image không tồn tại: liveness vẫn hoạt động, readiness và enqueue trả 503, không tạo job giả đang chờ. Khôi phục cấu hình image rồi bot tốt phải đạt 72 mẫu. Image thật và Docker daemon không bị thay đổi.

Sau SIGKILL, harness kiểm tra và thu dọn container sót của riêng nó trước khi khởi động lại; phép thử này **không chứng nhận hệ thống tự thu hồi container sau crash**. Chi tiết số container được thu dọn nằm trong JSON.

Để kiểm tra thêm deadline cứng 15 giây + khoảng chờ tuyên bố 3 giây, off-turn freeze, chặn fork/network/signal và replay/SSE reconnect:

```sh
.venv/bin/python tests/integration.py
```

Event timeout_claimed có `grace_waited_ms` đo bằng monotonic clock của controller và `grace_clock=controller-monotonic`. Đây là bằng chứng khoảng chờ; không lấy hiệu hai server_unix_ms để kiểm 3 giây, vì UTC/NTP/WSL có thể dịch chuyển. Deadline action vẫn do clock monotonic của worker quyết định. Unit test còn mô phỏng UTC chạy lùi và shutdown giữa khoảng chờ để kiểm không xử kết quả sai.

## Đọc kết quả

Sau khi preflight Docker/image thành công, `test-results/load-resilience-latest.json` chứa kết quả lần chạy mới nhất, cả khi thất bại; bản dễ đọc là `test-results/load-resilience-report.md`, log controller nằm cạnh đó. Có thể dựng lại Markdown bằng `.venv/bin/python tests/load_report.py`. Exit code khác 0 khi bất kỳ điều kiện bắt buộc nào không đạt. Chạy `make test` để kiểm tra regression logic riêng.

HTTP 429 do quota và HTTP 503 ở pha overload là cơ chế bảo vệ có chủ đích, không phải lời hứa phục vụ được mọi tải. Throughput đo bằng client closed-loop trên WSL local, dữ liệu nhỏ và image đã build; không thể quy đổi thành số người dùng production. Bài này chưa bao gồm soak nhiều giờ, cạn ổ đĩa, mất điện, mất toàn bộ Docker daemon/host, proxy/TLS hay nhiều máy.

## UI khi server bận

Uvicorn có thể từ chối ở lớp admission trước middleware, nên 503 có body text/plain thay vì JSON và có thể thiếu request ID. Client đã nhận diện trường hợp này để báo máy chủ đang bận; tải starter/report cũng đọc đúng loại body lỗi. Không tự retry mutation. 503 JSON có mã worker_unavailable vẫn giữ chẩn đoán Docker riêng.

Regression UI client: `node --test uet-bot-arena/tests/client.test.mjs` từ thư mục workspace. Fixture xem thông báo: `python3 uet-bot-arena/tests/overload-preview.py` (dữ liệu giả, không dùng backend thật).
