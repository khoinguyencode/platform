# Timing và clock authority

## Quỹ thời gian cả trận trên worker

Trusted Python supervisor chạy trong cùng container với một bot nhưng có UID khác. Supervisor dùng `time.monotonic_ns()` cho cả start, writes, đọc frame và so deadline. Không dùng `Date.now()`, UTC, clock của đội hay timestamp client để xử thua. Hai bot có hai clock_id; không lấy timestamp worker A trừ timestamp worker B.

1. Engine/control plane tính state và legal moves ngoài ngân sách.
2. Supervisor kiểm tra không có stdout unsolicited/response dư, đọc metrics trước lượt.
3. Mỗi bot khởi tạo quỹ R=60.000.000.000ns (Bullet) hoặc 300.000.000.000ns (đấu thường). Lấy t0; đặt D=t0+R; serialize observation có timing. Không cấp lại R khi đổi lượt.
4. Gửi sự kiện turn_started, SIGCONT bot, ghi toàn frame bằng nonblocking pipe. Cùng **D tuyệt đối** dùng cho mọi lần ghi/đọc, không reset timeout sau khi gửi xong hoặc mỗi chunk.
5. Khi đọc được LF, lấy `t1` ngay tại worker, rồi SIGSTOP và đợi process dừng. Chỉ nhận nếu `t1 < D`.
6. Trừ R=max(0,R-(t1-t0)), giữ nguyên quỹ đối thủ; không increment. Parse/schema/legality/ghi SQLite/broadcast UI nằm sau mốc t1. Parse sai hoặc move sai vẫn thua kỹ thuật.

Dispatch/scheduling/pipe overhead phía worker nằm trong wall metric; Docker CLI/HTTP/SSE ngoài ngân sách bot. Container/supervisor startup được control plane kiểm riêng, init→ready giới hạn 2s. CPU cgroup và RAM là metric chẩn đoán của cả container, gồm supervisor, không giả thành CPU/memory riêng của bot. Host transport timeout hoặc supervisor chết không được đổi thành bot timeout.

Polling select dùng deadline tuyệt đối và tick tối đa 10ms để kiểm stderr. Độ phân giải timestamp ns không đồng nghĩa độ chính xác hay khả năng scheduling 1ns. Nếu kernel/worker bị deschedule, đọc muộn có thể làm mất lượt; đo trên host riêng, chừa dư và hiệu chuẩn trước khi thi. Chưa có phát hiện host nhiễu tự động.

## Deadline cứng và 3 giây bù

Sau D bot bị freeze; response đến muộn không bao giờ được chấp nhận. Controller nhận timeout rồi chờ monotonic tối thiểu 3.000ms, sau đó ghi `timeout_claimed` của đối thủ với `by=server_delegate` và xử thua. Vì chờ tính từ lúc controller biết timeout, claim có thể muộn hơn D+3s do transport; luôn bảo thủ, không claim sớm. Mỗi trận luyện tập phải đồng ý delegation trong request tạo trận.

Event riêng của job `timeout_claimed` còn ghi `grace_waited_ms` và `grace_clock=controller-monotonic` để kiểm chứng khoảng chờ. Không lấy hiệu hai `server_unix_ms` làm bằng chứng: UTC có thể slew/jump do NTP hoặc WSL sync. Deadline action vẫn chỉ do monotonic clock của worker quyết định; số đo grace không cho phép nhận action trễ. Nếu controller shutdown trong khoảng chờ thì không ghi claim hay trao chiến thắng do timeout.

Cấu hình mới dùng `match_ms: 60000 | 300000`, mặc định 300000, riêng cho mỗi bot trong cả trận. Chỉ tính từ bắt đầu serialize observation đến khi worker nhận đầy đủ action; init/ready, build, engine, controller và chờ đối thủ không trừ quỹ. Benchmark dùng một quỹ xuyên 72 mẫu. `turn_ms` chỉ dành cho request/giải đã ghim theo chế độ cũ; không gửi đồng thời cả hai trường. Profile cũ vẫn giữ byte/cấu hình tương đương, dùng image cũ. Worker clock-v2 có handshake capability để không vô tình chạy quỹ mới trên supervisor cũ. Không giảm ngân sách vì người xem mạng chậm. Đầu hàng chỉ có thể qua một action đúng hạn; action gửi muộn kể cả resign bị coi là timeout. Không có external resign endpoint tạo tranh chấp đồng thời trong v1.

Nước đi đúng hạn: căn cứ → hết toàn quân → hết một loại → bí nước. Theo quy tắc vận hành v2, lặp cùng thế cờ/bên tới lượt 3 lần hoặc đạt guard 512 ply/20 phút là hòa; một bot lỗi startup do supervisor xác nhận bị thua kỹ thuật, cả hai bot lỗi thì không có đội thắng và cả hai nhận một trận thua. Lỗi Docker/transport/supervisor, artifact mất và shutdown không tính kết quả. Giải cũ thiếu policy vẫn giữ cách xử lý v1 (startup/guard cần BTC xử lý). Chi tiết và thứ tự ưu tiên: [MATCH-POLICY.md](MATCH-POLICY.md).

## Đồng bộ clock hiển thị

`GET /api/v1/time` trả server_received_unix_ms (t1), server_sent_unix_ms (t2), clock_id controller. Browser ghi p0,p3 bằng performance.now, lấy ít nhất 3 probe và chọn RTT thấp nhất. Mốc server xấp xỉ `(t1+t2)/2` tương ứng mốc performance `(p0+p3)/2`; độ bất định đường truyền ước lượng `max(0,RTT-(t2-t1))/2`. Nội suy server-now bằng performance.now, không phụ thuộc thay đổi Date.now sau sync.

Worker được probe qua RPC: host ghi monotonic h0,h3, worker trả w; offset hiển thị `(h0+h3)/2-w`, uncertainty `RTT/2`. Turn_started chứa worker t0/D dạng string, estimated_started_unix_ms và clock_uncertainty_ms. Viewer chỉ suy ra countdown ước lượng từ mốc đó và ngân sách; cộng các uncertainty khi hiển thị. Các số RTT này không phải cam kết đồng bộ vật lý: có asymmetry, scheduling, clock drift và tuổi probe. Sync browser mỗi 30s và khi quay lại tab; worker mapping lấy lúc tạo container. Không dùng ước lượng này để duyệt action.

Nếu cần nhiều máy: engine+supervisor giữ quyền phán quyết tại worker, lease job gắn worker epoch, worker trả receipt/error có turn/version. NTP/PTP của host phục vụ vận hành/log, không thay thế monotonic deadline và không được hồi tố kết quả. Worker restart tạo clock_id khác, trận cũ aborted. V1 chưa có distributed scheduler/lease recovery.

## Live/replay/reconnect

Event sequence đơn điệu theo job, lưu SQLite trước khi gửi SSE. Reconnect mang Last-Event-ID hoặc after cursor; server trả sự kiện mới, client bỏ duplicate. Client chỉ thay nội dung ô cờ/count/clock/panel; không reload trang. Disconnect viewer không ảnh hưởng worker deadline. State/turn_started/timeout_pending chứa `clocks: {red,blue}` (ms); replay giữ số dư tại từng state, kể cả khi kết thúc. Replay đọc state events bất biến; không lấy clock hiện tại chạy lại bản ghi.
