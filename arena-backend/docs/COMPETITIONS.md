# Giải chính thức

Thể thức: **vòng tròn hai lượt hoặc Swiss**. Với vòng tròn, mỗi cặp đội cầm Đỏ và Xanh một lần; **thắng 3đ, hòa 1đ, thua 0đ**. Tổng điểm `points = 3 × wins + draws` quyết định thứ hạng. Bằng tổng điểm thì đồng hạng, ví dụ 1, 1, 3; không dùng số thắng làm hệ số phụ. Ví dụ một trận thắng và ba trận hòa đều được 3đ.

API standings trả `ranking: "points_shared_rank"`, `scoring: {"win":3,"draw":1,"loss":0}` và mỗi đội có `played`, `wins`, `draws`, `losses`, `points`, `point_breakdown`, `rank`. `point_breakdown` chứa số điểm từ thắng/hòa/thua, còn độ dài các đoạn biểu đồ thể hiện số trận. Trước khi tất cả fixture hoàn tất, `provisional: true`. Trận hòa theo [MATCH-POLICY.md](MATCH-POLICY.md) được 1đ mỗi đội; cả hai bot lỗi startup tính thua kỹ thuật cho cả hai, mỗi đội 0đ. Lỗi hạ tầng ở trạng thái `attention` chưa tính vào bảng.

Migration `004_points_ranking.sql` chuyển cấu hình giải cũ từ `wins_shared_rank` sang công thức này, tăng revision và ghi audit một lần. Điểm được tính lại từ kết quả đã lưu; không sửa kết quả, lịch, binary hay cấu hình timing/adjudication đã chốt, không chạy lại trận. Sao lưu SQLite trước khi nâng cấp máy chủ.

## Một giải từ đầu đến cuối

Mọi endpoint dưới đây bắt đầu bằng `/api/v1`. Mutation cần session, `X-CSRF-Token` và **`Idempotency-Key` 8–128 ký tự**. Giữ key khi retry đúng thao tác; một key mới cho thao tác tiếp theo. Replay trả response gốc, không phải trạng thái mới nhất; đọc GET để lấy hiện trạng.

1. Admin `POST /admin/competitions`:

   ```json
   {"title":"UET OTTv2 Cup","match_ms":60000,"max_teams":16}
   ```

   Trả `draft`. Mặc định 16 đội; vòng tròn giới hạn 2–32, Swiss 2–4096; tối đa 100 giải trong beta local. Server ghim immutable worker image ID, hash engine/SDK/suite, compiler flags, profile và adjudication v2. Docker phải sẵn sàng để tạo giải mới; retry cùng key đã thành công vẫn trả record khi Docker offline. Sửa tên/sức chứa khi draft/open; chỉ đổi thời gian khi draft chưa có entry. Sau khi chốt không đổi cấu hình.

2. Admin `POST /admin/competitions/{id}/open`. Đội đọc `GET /competitions/{id}` để biết profile. Các đội có tài khoản hoạt động, đã đổi mật khẩu lần đầu có thể tham gia.
3. Đội nộp ZIP và chạy `POST /benchmarks` với `{submission_id, competition_id}` để ghim đúng luật và thời gian giải. Đợi job `completed`, `passed: true`, đủ 72 mẫu của suite `ottv2-public-smoke-v1`.
4. Đội `POST /competitions/{id}/entry`:

   ```json
   {"submission_id":"<32 hex>","benchmark_id":"<32 hex>","delegate_timeout_claims":true}
   ```

   Benchmark phải thuộc chính đội và version đó; cùng image, SDK, flags, source, suite, engine và profile. `qualification_required` trả 409 nếu không đạt. Đây là kiểm tra smoke và provenance, không bảo đảm bot sẽ hợp lệ hoặc kịp giờ trong mọi trạng thái.

   Trước khi chốt, đội được thay entry bằng POST mới, hoặc `POST /competitions/{id}/withdraw`. Admin có thể `POST /admin/competitions/{id}/entries/{team_id}/remove` để loại entry trước khi chốt, có audit. Không sửa/xóa source version đã nộp.

5. Admin `POST /admin/competitions/{id}/lock`. Cần ít nhất hai đội; các account dự giải phải đang hoạt động, không chờ đổi mật khẩu. Server kiểm lại qualification, lưu snapshot và sinh lịch **trong cùng một transaction**. Đội lẻ có lượt nghỉ. Có `N × (N−1)` trận; 32 đội tạo 992 fixture, mỗi đội Đỏ 31/Xanh 31. Không sửa entry, thay binary hoặc ghép thêm đội sau bước này.
6. Admin `POST /admin/competitions/{id}/start`. Scheduler đưa nhiều fixture vào các worker khả dụng; đọc `/fixtures`, `/standings` và `/fixtures/{fixture_id}/events`. Khi tất cả fixture hoàn tất hợp lệ, giải tự chuyển `completed`.

```text
draft → open → locked → running → completed
                         ↕
                       paused
```

`POST /admin/competitions/{id}/pause` ngăn dispatch mới. Attempt đã vào queue/đang chạy tiếp tục kết thúc; đây không phải nút kill bot giữa lượt. `start` tiếp tục từ lịch còn lại, không chạy lại các trận đã có kết quả.

## Binary, thời gian và kết quả

Entry chốt giữ submission ID, source manifest, benchmark ID/report hash, binary SHA-256. Official worker kiểm lại source, engine, SDK, flags, profile và binary trước khi chạy. Phải dùng **đúng binary đã benchmark**. V2: nếu mất artifact/manifest, dừng để BTC khôi phục artifact đã kiểm chứng rồi retry; không tự rebuild. V1 giữ cơ chế rebuild chỉ khi ra cùng SHA-256; code dùng `__TIME__` hoặc build không tái lập có thể không đáp ứng. Giữ backup artifact đã chốt.

Deadline giữ nguyên [TIMING.md](TIMING.md): worker `monotonic_ns` quyết định; chỉ nhận action khi `t1 < D`; sau hết giờ cứng chờ ít nhất 3.000 ms rồi server đại diện đối thủ tuyên bố. Cả hai đội ủy quyền riêng khi nhập entry. HTTP, trình duyệt, UTC/NTP và RTT không quyết định thắng/thua. Không thêm thời gian theo ngôn ngữ.

15/30/60/300s mỗi lượt; startup 2s, build 30s, bot 256 MiB/1 CPU, build 1.024 MiB/1 CPU. Profile thực thi dùng chung với luyện tập (ID hiện vẫn có tiền tố `ottv2-practice-*`); official scope nằm riêng trong fixture/job, không suy ra từ tên profile.

Một trận chỉ được tính khi attempt hiện hành `completed`, report có result hợp lệ khớp final state, đúng provenance/policy của giải và được worker xác nhận là official. Protocol/illegal action/crash/output limit trong lượt là thua kỹ thuật. Giải v2 hòa khi lặp thế 3 lần hoặc chạm 512 ply/20 phút; bot lỗi startup được supervisor xác nhận bị thua kỹ thuật, cả hai lỗi thì cả hai nhận một trận thua và không có đội thắng. Build/worker/control failure, artifact mất hoặc shutdown không có kết quả, fixture chuyển `attention`. Giải v1 giữ startup/guard ở `attention`. Xem [MATCH-POLICY.md](MATCH-POLICY.md). Trận luyện tập không vào bảng này.

Admin xem `/admin/competitions/{id}/fixtures/{fixture_id}/attempts` và `/attempts/{number}` để đọc lỗi/report riêng. Sau khi xử lý nguyên nhân, `POST /admin/competitions/{id}/fixtures/{fixture_id}/retry` tạo attempt mới cho fixture `attention`; scheduler chỉ chạy khi giải `running`. Worker thử lại một lần khi lỗi hạ tầng worker_error/worker_unavailable, không thử lại lỗi do bot, và chỉ thử lại tại chỗ khi trận chưa phát bàn cờ đầu tiên; trận đã bắt đầu thì fixture chuyển `attention` để BTC retry thành attempt mới, nên replay không bị trộn hai lần chạy. Nếu vẫn lỗi, BTC xử lý rồi retry fixture attention; không cho retry fixture đã có kết quả. Các attempt cũ được giữ, không tính điểm; một completion cũ đến sau retry không thể ghi đè kết quả hiện tại. Bảng xếp hạng tính từ fixture chính thức, không cộng dồn theo event hoặc mỗi lần GET.

## Scheduler và phục hồi

SQL giữ fixture, snapshot, attempts, result, command idempotency và hàng đợi. Có tối đa N attempt official outstanding với N worker; ưu tiên official → benchmark → practice, FIFO cùng ưu tiên. Job đã chạy không bị preempt. N mặc định số CPU khả dụng trừ 2, cấu hình ARENA_WORKERS; build giới hạn riêng ARENA_BUILD_WORKERS=2. Mỗi trận ghim một logical CPU. Không chạy nhiều controller dùng cùng DB.

Khởi động lại giữ job queued, hủy job đã claim/running và chuyển fixture tương ứng sang attention. Giải đang running chuyển paused; admin xem attempt, retry nếu cần rồi start. Không resume clock hoặc tự replay trận dở. Khởi động dọn riêng container mang label hash đúng data directory, không đụng container của controller khác. Heartbeat/lease được ghi mỗi 5 giây; khi một thread chết, readiness báo lỗi và cần restart controller giữ lease, chưa có distributed failover.

## Swiss và luật cơ bản theo giải

Tạo giải với `format: "swiss"`, `max_teams` tối đa 4096 và `rounds` tùy chọn. Khi chốt, số vòng mặc định là ceil(log2(số đội thực tế)) + 2, giới hạn bởi số đối thủ có thể gặp: N−1 vòng nếu N chẵn, N vòng nếu N lẻ. Số vòng nhập rõ vượt mức này bị từ chối. `rounds` trong API là 1–64. Vòng tròn giữ `format: "double_round_robin"` và tối đa 32 đội.

Mỗi cặp đánh hai ván đổi màu nên tổng Đỏ/Xanh cân bằng. Ghép cặp ưu tiên cùng điểm, cấm gặp lại đối thủ cũ. Với số đội lẻ, một đội chưa từng nghỉ và có điểm thấp được bye, tính như thắng cả vòng: **6 điểm** (hai ván × 3), ghi riêng ở `bye_points`, không tính vào số trận đã đấu; không đội nào nghỉ quá một lần. Xếp điểm giảm dần rồi Buchholz, là tổng điểm hiện tại của các đối thủ khác nhau đã gặp, không đếm đôi vì hai ván; bye không có đối thủ giả. Cùng điểm và Buchholz thì đồng hạng.

Chỉ ghép vòng kế sau khi mọi fixture vòng trước completed; attention chưa giải quyết sẽ chặn ghép vòng. Nếu không còn cách ghép nào tránh được tái đấu (thường gặp ở giải 5–6 đội đánh gần hết số vòng), hệ thống cho gặp lại với số cặp tái đấu ít nhất thay vì dừng giải. Một lỗi ghép bất thường chỉ tạm dừng đúng giải đó và lưu `pairing_error` cho BTC; bấm Bắt đầu để ghép lại. Các giải khác vẫn chạy bình thường. Swiss không phải chuẩn FIDE đầy đủ; đây là thể thức theo quy định dự án.

`basic_rules` cho phép `rows`, `cols` 7–15, `bases`, `eliminate_types` 1/2, `base_capture` và bố trí quân auto/manual. Xem schema OpenAPI và [BOT-LANGUAGES.md](BOT-LANGUAGES.md). Luật chuẩn hóa có SHA-256, ghim vào config/suite/artifact qualification và gửi trong init/observation. Quỹ giờ giữ nguyên Bullet 60 giây / thường 300 giây trên mọi kích thước bàn; chỉ giới hạn số nước (ply) tăng theo số ô (512 nước ở 9×9). Xếp quân tự động là hình răng cưa của OTTv2 tổng quát cho mọi kích thước, 9 quân mỗi bên, đối xứng tâm; bàn 9×9 luôn giống hệt OTTv2. Server từ chối luật cho phép một bên thắng ngay ở nước đầu. Luật mặc định giữ kết quả cũ. Add-on năng lượng/khiên vẫn là SDK v3 riêng, không kết hợp luật cơ bản tùy chỉnh.

Ba runtime C++17, C11 và Python/PyPy dùng chung giao thức, cùng quỹ giờ. Giải ghim image ID, toolchains và runtime flags thực tế; Python chạy bằng PyPy 7.3.23 (Python 3.11) sau bộ kiểm chứng [PYPY-VALIDATION.md](PYPY-VALIDATION.md).
## Live/replay và quyền truy cập

Các GET giải chính thức cho phép [khách xem không cần tài khoản](GUEST-SPECTATOR.md). Admin thấy giải nháp/đã ẩn; khách và đội chỉ thấy giải đã mở chưa bị ẩn. Chia sẻ tên đội, version, màu, board, countdown và kết quả. Không chia sẻ email, thành viên, source, compiler stderr, binary manifest, benchmark report hoặc metrics của đối thủ. Raw job official thuộc namespace nội bộ `competition:<id>`; kể cả hai đội dự trận cũng không đọc được qua `/jobs/{id}`. Admin đọc diagnostic qua API attempts có RBAC. Chọn/rút bot và các mutation vẫn cần tài khoản, CSRF như trước.

`GET /competitions/{id}/fixtures/{fixture_id}/events`:

- Không có `Accept: text/event-stream`: trả `attempt_id`, `items`, `next_after`. Giới hạn 1.000 raw event/page; một số event bị lọc nên `next_after` có thể lớn hơn seq cuối trong `items`. Tiếp tục bằng cursor này, không tự suy ra từ số event.
- SSE: `event: attempt` báo attempt đang theo dõi; `event: arena` chứa board/timing/result đã lọc; **ID là `<attempt_id>:<seq>`**. Browser reconnect gửi `Last-Event-ID`. Khi retry đổi attempt, cursor cũ reset về 0. Khi đổi giữa một stream, server gửi `attempt_changed`, đóng stream để browser nối lại. Không cộng state của hai attempt vào cùng một replay.
- JSON cũng nhận header `Last-Event-ID: <attempt_id>:<seq>`, nên dùng cách này để tiếp tục an toàn qua retry. Query `after` chỉ dùng khi đã biết vẫn cùng attempt.
- Session được kiểm lại khi stream đang mở; hết hạn/thu hồi gửi `auth_expired` rồi đóng. Mỗi stream tối đa 60s, heartbeat mỗi giây; cấu hình gateway không buffer SSE. Frontend đóng EventSource khi rời trang hoặc nhận `auth_expired`.

Frontend patch board và clock theo event, không full reload. Dùng `estimated_started_unix_ms`, `budget_ms`, `clock_uncertainty_ms` và clock sync API để hiển thị; không tự xử thua khi countdown về 0. Đóng EventSource khi nhận `completed` để tránh reconnect một trận đã xong. Adapter ví dụ: `ArenaAPI.liveFixture` trong `examples/client.mjs`. UI hosted hiện chưa nối các endpoint mới.

## Kiểm thử

```bash
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/python tests/competition_integration.py
make schema
```

`test_competitions.py` dùng benchmark record tổng hợp để kiểm state machine, qualification, SQL rollback, lịch 3/32 đội, queue đầy, retry/restart, chốt binary, privacy và idempotency. `competition_integration.py` dùng Uvicorn HTTP và Docker thật: hai đội nộp C++, benchmark đủ 72 mẫu/đội, chốt, đấu hai lượt, đọc SSE/replay, kiểm version/ownership và bảng đồng hạng. Report local tại `test-results/competition-latest.json` (ignored), không chứa credentials.

## Viewer theo dõi liên tục

`GET /api/v1/competitions/{id}/watch` trả snapshot JSON hoặc SSE `competition` khi Accept là text/event-stream. Snapshot chỉ gồm competition_id, status và fixtures đã whitelist (id, ordinal, màu/đội, status, result); không chứa job ID, source, báo cáo hay liên hệ. Có đầy đủ fixture, không phụ thuộc phân trang bảng. SSE kiểm phiên mỗi giây, heartbeat khi không đổi, kết thúc sau tối đa 60 giây để EventSource nối lại; luôn gửi snapshot hiện tại khi nối lại nên không cần event cursor. Giải hoàn tất gửi snapshot cuối rồi viewer đóng feed. Màn live dùng snapshot này để chuyển trận tại chỗ; Replay giữ cố định.

Đồng hồ cả trận: Bullet 60.000ms hoặc đấu thường 300.000ms **mỗi bot**, không cộng giây. Observation thêm `payload.clocks` (red/blue, ms); `timing.time_control="match"`, `remaining_ms`, `initial_ms`, `increment_ms=0`. `budget_ms` là quỹ còn lại làm tròn xuống; dùng deadline tuyệt đối do worker cấp. Response/action giữ nguyên v2/v3. Benchmark chia chung quỹ cho 72 mẫu. Request `turn_ms` chỉ hỗ trợ cấu hình cũ; không gửi cùng `match_ms`. Xem [TIMING](TIMING.md).
