# Backend web — v0.6

Lớp FastAPI/Uvicorn mới nằm ở `arena_api/`, dùng lại `arena/engine.py`, `arena/service.py` và Docker worker đã kiểm thử. UI chính tại `/` đã nối API cùng origin; xem [UI.md](UI.md) cho luồng test trên WSL. UI hosted cũ vẫn dùng dữ liệu demo. Adapter `examples/client.mjs` giữ làm ví dụ cho tích hợp khác.

v0.3 thêm [giải vòng tròn hai lượt](COMPETITIONS.md): chốt bot/binary đã benchmark, lịch bền vững, scheduler, retry attempt lỗi và live feed dùng chung đã lọc dữ liệu riêng. Bảng xếp hạng hiện dùng thắng 3đ, hòa 1đ, thua 0đ, đồng hạng khi bằng tổng điểm.

v0.5 thêm [giao diện và thao tác BTC](ADMIN.md): dashboard metrics, CRUD có khôi phục, revision để tránh ghi đè, danh sách job/source và audit.

v0.6 thêm [quy tắc xử lý kết quả v2](MATCH-POLICY.md): hòa lặp thế/guard, lỗi startup do bot và hàng đợi ưu tiên giải chính thức. Giữ thể lệ v1 cho giải đã tồn tại; bảng xếp hạng/CSV/live/replay hiển thị hòa và cả hai bot lỗi startup.

## Chạy local trên Ubuntu / WSL2

[Guest spectator](GUEST-SPECTATOR.md) mở các GET giải chính thức cho khách; vẫn bảo vệ source, job riêng, mutation và trang quản trị. Giải nháp/đã ẩn chỉ BTC xem được, kể cả truy cập qua URL đã biết.

Trong thư mục `arena-backend`:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
docker build -t uet-arena-worker:clock-v2 -t uet-arena-worker:experimental-clock-v2 .
.venv/bin/python -m arena_api migrate
.venv/bin/python -m arena_api bootstrap-admin --username organizer
# CLI hỏi mật khẩu bằng getpass, không in ra hoặc đưa vào command history.
.venv/bin/python -m arena_api serve
```

UI: **http://127.0.0.1:4181/**. API reference: **http://127.0.0.1:4181/docs**. Schema: `/openapi.json`. Không có tài khoản hoặc mật khẩu mặc định. Không chạy `bootstrap-admin` khi controller đang giữ data directory; CLI sẽ từ chối thay vì chiếm queue đang chạy.

Các biến môi trường có trong `.env.example`; ứng dụng đọc environment của process, không tự đọc file `.env`. Đổi port phải đổi `ARENA_API_ORIGIN` tương ứng. `ARENA_API_DATA` mặc định `.platform-state` theo working directory. Lab cũ dùng `.state` trên port 4180; hai nơi không tự nhập tài khoản/source của nhau.

## Cấu trúc

```text
arena_api/
  app.py                 app factory, lifespan, HTTP boundary, error envelope
  settings.py            config + single-controller lease
  schemas.py             Pydantic input schemas; không nhận field role/team_id tùy ý
  dependencies.py        session, CSRF, role và first-password guards
  auth_routes.py         đăng ký, login, me, đổi mật khẩu, logout, recovery
  admin_routes.py        duyệt/từ chối, tài khoản, khóa/reset/delivered, audit
  arena_routes.py        đội, dashboard, source, benchmark, match, report, SSE
  ui_routes.py           UI assets, platform/rules công khai, admin dashboard
  competition_routes.py  giải, entry, chốt lịch, điều khiển, standings, live đã lọc
  competitions.py        SQL state machine, qualification, scheduler, reconcile
  accounts.py            Argon2, session repository, approval transactions
  migrations.py          chạy SQL migration theo version
  migrations/001_accounts.sql
  migrations/002_competitions.sql
arena/                   engine + queue + worker; clock authority giữ nguyên
examples/client.mjs      adapter cùng origin cho UI chính
deploy/                  systemd/nginx templates, chưa áp dụng lên server thật
```

SQLite WAL dùng chung connection có RLock, migrations có transaction và schema version; một process/controller cho mỗi data directory. Các bảng tài khoản/đội/session/registration/audit là relational; documents/events của worker giữ định dạng cũ. Chưa có PostgreSQL adapter hoặc distributed queue; không chạy nhiều Uvicorn workers trên cùng SQLite. Lease được giữ đến khi worker dừng và DB đóng.

## Luồng tài khoản

1. `POST /api/v1/registrations` gửi `team`, `email`, `phone` (tùy chọn), `members` (3–4 người: `name`, `student_id`), `confirmed: true`. Không gửi mật khẩu. Trả ID hồ sơ + `pending`.
2. Admin đăng nhập `/auth/login`, đọc `/admin/registrations`, gửi `/admin/registrations/{id}/approve` với `username` và `temporary_password` tùy chọn. Nếu bỏ mật khẩu, server sinh bằng CSPRNG. Approval tạo đội + account + cập nhật hồ sơ + audit trong **một transaction**. Trùng username sẽ rollback toàn bộ, hồ sơ vẫn pending.
3. Response cấp/reset chỉ lần đó có `temporary_password`. Admin phát riêng cho đội bằng kênh đã thống nhất; hệ thống không tự gửi email. `POST /admin/accounts/{id}/delivered` chỉ đánh dấu đã phát. Nếu làm mất response, cấp lại mật khẩu; không có API đọc lại mật khẩu cũ.
4. Đội đăng nhập bằng mật khẩu tạm. Phiên chỉ đọc `/auth/me`, đổi mật khẩu hoặc logout; các API đội đều trả `password_change_required` cho đến khi đổi thành công.
5. `POST /auth/password` nhận `current_password`, `new_password`. Mật khẩu mới phải khác mật khẩu hiện tại, 8–128 ký tự. Tất cả session cũ bị thu hồi; response đặt cookie và CSRF mới.
6. Khóa/mở khóa/cấp lại mật khẩu đều thu hồi phiên. Reset giữ nguyên trạng thái khóa, đưa account về bắt buộc đổi mật khẩu và đánh dấu chưa phát. Admin API không cấp/reset tài khoản admin; bootstrap admin là thao tác vận hành offline.
7. `/auth/recovery` luôn trả cùng thông báo dù tài khoản/email có khớp hay không. Chỉ yêu cầu khớp mới xuất hiện cho admin. Admin kiểm tra người yêu cầu ngoài hệ thống trước khi reset.

Password hash dùng Argon2id (64 MiB, time cost 2, parallelism 1, salt tự sinh). Session token 256 bit ngẫu nhiên, DB chỉ giữ SHA-256; không dùng JWT hoặc token ở localStorage. CSRF token là giá trị riêng trong session, không thay thế cookie đăng nhập. Session TTL mặc định 1 giờ, được kiểm ở mỗi request và khi tiếp tục SSE. Restart API giữ các session còn hạn; clock ID đổi, job dở thành aborted.

## Contract frontend

v0.4 phục vụ UI từ `ARENA_UI_ROOT` (mặc định checkout `uet-bot-arena/dist` cạnh backend), dùng entry `live/index.html`, không mount dữ liệu/source. `/api/v1/platform` và `/api/v1/rules` công khai cho trang chủ/đăng ký/tài liệu. `/api/v1/admin/dashboard` chỉ admin đọc counts thật. Job list của đội có `benchmark_summary` (passed, samples, suite_id, turn_wall_ms); entry vẫn được server xác minh provenance đầy đủ, summary không phải credential qualification.


Login trả:

```json
{
  "user": {"id": "…", "username": "uet-team-001", "role": "team", "team_id": "…", "must_change_password": true, "locked": false, "delivered": false, "created_ms": 0},
  "csrf_token": "…",
  "expires_ms": 0
}
```

Cookie `arena_web_session`: HttpOnly, SameSite=Strict, Path=/api, Secure trên HTTPS public. Browser gửi `credentials: 'same-origin'`. Sau reload gọi `/auth/me` để khôi phục user + CSRF. Mọi mutation có session phải gửi `X-CSRF-Token`; frontend giữ token này trong memory. Logout/password rotation phải cập nhật trạng thái UI tương ứng.

`GET /teams/me`, `/dashboard`, `/submissions`, `/jobs` chỉ dùng team ID từ session server. Admin không được giả làm đội bằng query/body. Jobs/source của đội khác trả 404. `/dashboard` có counts thật và 10 job gần nhất; không dựng ELO/bảng xếp hạng từ trận luyện tập.

Nộp source dùng raw `application/zip`, headers `X-Submission-Version`, `X-Arena-Runtime: cpp17|c11|python3`, `Idempotency-Key`. Benchmark/match vẫn là JSON theo [API worker](API.md). Giữ nguyên idempotency key khi retry cùng một thao tác; tạo key mới cho thao tác khác. Queue lưu SQLite, không có cap 16 toàn hệ thống; giữ job chờ mới nhất mỗi loại/đội, tối đa 4 job đang chạy và 200 job lịch sử/đội. Không nhận `team_id` tùy ý.

SSE tại `/jobs/{id}/events`: cookie cùng origin, event `arena`, sequence ID tăng dần. Reconnect dùng `Last-Event-ID` hoặc `after`; chỉ patch board, không reload cả trang. Nếu phiên hết hạn/thu hồi giữa stream, server gửi `auth_expired` rồi đóng; UI cần đóng EventSource và về đăng nhập. Server đóng stream sau tối đa 60s; browser nối lại từ cursor.

Lỗi dạng:

```json
{"error":{"code":"password_change_required","detail":"Change the temporary password before using team APIs"},"request_id":"…"}
```

401 chưa đăng nhập/phiên bị thu hồi; 403 CSRF/role/first-password; 404 bản ghi không thuộc đội hoặc không tồn tại; 409 conflict/report chưa có; 422 schema; 429 quota/throttle; 503 worker không sẵn sàng. Validation không echo giá trị password. `X-Request-ID` giúp đối chiếu. Endpoint JSON phải gửi Content-Length; chunked upload hiện chưa hỗ trợ.

Danh sách `/admin/*`, `/submissions`, `/jobs` có `limit` 1–100 và `offset`. Event log tối đa 1.000 event/page; tiếp tục bằng seq cuối. OpenAPI tự sinh từ code, bản xuất trong `docs/openapi.json`; `/docs` là reference tự host, không tải JavaScript CDN.

## Timing và giới hạn

Worker protocol/timing không đổi: [PROTOCOL](PROTOCOL.md), [TIMING](TIMING.md). Hỗ trợ C++17, C11 và Python 3.11 (PyPy), cùng quỹ 5 phút/đội/cả trận; tùy chọn Bullet 1 phút, không increment. Worker monotonic clock quyết định, action phải đến trước deadline cứng; server đại diện đối thủ sau tối thiểu 3s. Clock API chỉ đồng bộ hiển thị. Init/ready 2s; build 30s; run container 256 MiB/1 CPU; build container 1.024 MiB/1 CPU. Codon chưa bật ở API mới.

Login throttle lưu DB: 10 lần/username/15 phút, 30 lần/IP/15 phút (tính cả thành công). Đăng ký và recovery mỗi loại 5 lần/IP/giờ. IP ở đây là peer của ASGI, không tin X-Forwarded-For; khi đặt gateway cần giới hạn IP tại gateway riêng. Không ghi Authorization/cookie/password/source vào access log. Audit chỉ chứa actor ID/action/target ID/time.

## Kiểm thử và triển khai

Nếu WSL báo `Fatal Python error: error evaluating path` / `failed to make path absolute` ngay sau một bộ test đã pass, tiến trình Python kế tiếp chưa khởi động được, chưa phải kết quả bộ test đó. Makefile đã thêm `cd` theo đường dẫn tuyệt đối trước từng lệnh để xác lập lại working directory. Trong terminal đang lỗi, chạy lại `cd` tới thư mục `arena-backend` bằng đường dẫn tuyệt đối rồi `make integration`. Không cần tạo lại venv hoặc chạy với quyền Administrator chỉ vì lỗi này.

```bash
.venv/bin/python -m unittest discover -s tests -v
# Docker integration phải có daemon/image thật, không skip hoặc giả số đo.
.venv/bin/python tests/integration.py
.venv/bin/python tests/web_integration.py
.venv/bin/python tests/competition_integration.py
make load-test
make schema
```

`test_web_api.py` kiểm luồng tài khoản, first-password/CSRF/RBAC, khóa/reset, session expiry/persistence, atomic rollback, schema/body limits, ownership và SSE cursor. `web_integration.py` mở Uvicorn HTTP thật, đăng ký → admin duyệt → đổi mật khẩu → upload starter → benchmark 72 mẫu → trận OTTv2 → replay/SSE → admin reset thu hồi phiên. Report không chứa credential tại `test-results/web-api-latest.json`.

`test_competitions.py` kiểm qualification, state machine, lịch 3/32 đội, rollback chốt lịch, queue đầy, retry/restart, provenance và privacy. `competition_integration.py` chạy giải hai đội/hai lượt bằng C++ và Docker thật; report `test-results/competition-latest.json`. Xem [COMPETITIONS.md](COMPETITIONS.md) cho API và cách xử lý attempt `attention`.

`make load-test` chạy CLI thật trên database/cổng test riêng: tải đọc 1/8/24 client, overload 48 client, quota job, idempotency đồng thời, 12 stream SSE, revoke phiên, bot lỗi, kill worker/controller, restart/retry giải và thiếu image. Xem [LOAD-RESILIENCE.md](LOAD-RESILIENCE.md) và `test-results/load-resilience-latest.json`. Đây là phép đo ngắn trên WSL, không phải chứng nhận năng lực production.

Production cần host Linux riêng có Docker, reverse proxy HTTPS cùng origin với UI và backup DB/artifacts. Templates trong `deploy/` chưa được chạy. Config public phải có `ARENA_API_ORIGIN=https://arena.example.edu`, `ARENA_API_COOKIE_SECURE=1`, data directory riêng. API bind loopback; gateway giữ Host đúng public origin và tắt buffering cho SSE. Chạy bằng user dịch vụ có quyền điều khiển Docker trên host worker dành riêng; Docker access là quyền nhạy cảm của controller, không mount vào bot.

Shutdown chờ thao tác worker hiện tại kết thúc có giới hạn trước khi nhả lease; một lượt 300s có thể kéo dài shutdown. systemd template cho 400s, tránh kill rồi khởi động controller mới đua job cũ. Backup SQLite bằng SQLite backup API hoặc lúc dịch vụ đã dừng, không chỉ copy file main bỏ qua WAL. Lưu cùng source/artifacts tương ứng; thử restore trước giải thật.

Phạm vi chưa làm: publish UI/API chính lên server, Codon v2, PostgreSQL/distributed scheduler, nhiều worker đồng thời, audit bảo mật độc lập. UI local đã nối API. Chưa public API hoặc tạo tài khoản thật cho người dùng. Beta giải chỉ hỗ trợ vòng tròn hai lượt; chưa có nhánh đấu loại, xử khiếu nại hoặc disqualification sau khi chốt.

Tài liệu nền tảng: [FastAPI lifespan](https://fastapi.tiangolo.com/advanced/events/), [FastAPI security/password hashing](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/).


Worker thread cấu hình ARENA_WORKERS (mặc định CPU khả dụng trừ 2), mỗi trận ghim CPU; build semaphore ARENA_BUILD_WORKERS=2. Giữ một Uvicorn process/data directory. Queue có ưu tiên official → benchmark → practice, heartbeat/lease; lỗi hạ tầng tự thử lại một lần khi trận chưa bắt đầu, bot fault không tự retry. Startup dọn container đúng owner label, giữ queued và abort job chạy dở. SSE chia sẻ snapshot/JSON đã lọc, ARENA_CONNECTION_LIMIT=1024. Xem [BOT-LANGUAGES.md](BOT-LANGUAGES.md), [COMPETITIONS.md](COMPETITIONS.md).
