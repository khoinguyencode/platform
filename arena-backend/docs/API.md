# Control API v1

Base local: `http://127.0.0.1:4180/api/v1`. JSON UTF-8. Header `Authorization: Bearer <team-token>` hoặc session cookie HttpOnly qua `/session`. Host phải khớp configured origin; Origin khác bị từ chối; không CORS wildcard. Token không đưa vào query URL, source, report hoặc container. Public health/time không trả source hoặc credential.

## Endpoints

| Method / path | Mục đích |
|---|---|
| GET `/health` | Docker/image sẵn sàng, runtime C++17; ready=false khi không chạy được. |
| GET `/time` | Server receive/send Unix ms + clock_id; chỉ sync UI. |
| POST `/session` | `{"token":"..."}` → session 1h, team_id; credentials không persisted. |
| POST `/logout` | Thu hồi session hiện tại. |
| GET `/config` | Team, protocol/profile/turn options/runtime allowlist. |
| GET `/starter.zip` | ZIP C++ v2: source, JSON header/license và README. |
| GET `/submissions` | Phiên bản của đội hiện tại. |
| POST `/submissions` | Body ZIP raw, không multipart. Headers bên dưới. |
| POST `/benchmarks` | Tạo job build + preflight + 72 mẫu. |
| POST `/matches` | Tạo trận giữa hai submission thuộc đội hiện tại. |
| GET `/jobs` | Lịch sử job của đội. |
| GET `/jobs/{id}` | Status, profile, image/engine provenance, error, report nullable. |
| GET `/jobs/{id}/report` | JSON thật, kể cả báo cáo một phần khi job lỗi. |
| GET `/jobs/{id}/events?after=0` | Event JSON, tối đa 1000 mục; phân trang bằng seq cuối. |
| GET `/jobs/{id}/events` + Accept SSE | Stream event `arena`, id=seq; Last-Event-ID hỗ trợ reconnect. |

Các `{id}` là opaque ID do server trả, không suy diễn từ tên đội/version. Truy cập record đội khác trả 404. Khác-team pairing cần hệ thống giải/admin riêng, chưa mở trong v1.

## Nộp source

```http
POST /api/v1/submissions
Authorization: Bearer <team-token>
Content-Type: application/zip
Content-Length: ...
X-Submission-Version: v1.0.0
X-Arena-Runtime: cpp17
Idempotency-Key: <unique-client-key>

<raw ZIP bytes>
```

201 trả `id, team_id, version, runtime, protocol_version, source{zip_sha256, files, uncompressed_bytes}, artifact:null`. Source path nội bộ không trả qua submission API. Header version là semver, không được ghi đè. Idempotency-Key 8..128 ký tự: cùng key/cùng input trả record cũ; khác input trả 409. Queue jobs cũng áp dụng idempotency và scope theo đội/operation.

ZIP tối đa 10 MiB, giải nén tối đa 20 MiB, 64 entries, thư mục phẳng, entry bot.cpp / bot.c / bot.py theo runtime cpp17 / c11 / python3. Chỉ source/header cùng ngôn ngữ, README.md, LICENSE.MIT; không directory, symlink, special file, encrypted ZIP, duplicate name, path traversal, binary, script. Giới hạn beta 20 submissions, 200 jobs/đội, 4 job đang chạy/đội; queue SQLite không cap 16 chung, mỗi loại/đội giữ bài chờ mới nhất, API trả queue_position. Các limit này là profile local, không phải hạn mức giải đã công bố.

## Tạo benchmark / match

```json
{"submission_id":"server-returned-id","match_ms":60000}
```

```json
{"red_submission_id":"id-a","blue_submission_id":"id-b","match_ms":60000,"delegate_timeout_claims":true}
```

POST tương ứng `/benchmarks` hoặc `/matches`, Content-Type application/json và Idempotency-Key. 202 trả job đầy đủ có `id,status:"queued",submission_ids,profile,image_id,engine_id,engine_sha256,report:null`. Client không gửi command, compiler flags, image, state ban đầu, budget tùy ý hoặc kết quả.

Trạng thái: queued → building → validating → benchmarking/playing → completed|failed|aborted. N worker ghim CPU riêng, ưu tiên official → benchmark → practice; ARENA_WORKERS mặc định CPU khả dụng trừ 2. Trong match, hai bot frozen khi off-turn. Build được cache theo source+SDK+image+flags; báo cáo vẫn chứa manifest của đúng artifact.

`completed` nghĩa job đã chạy xong, không phải được chấp thuận dự giải. API worker lab/luyện tập ở đây trả `eligible_for_tournament:false`. Web API v0.3 có official scope riêng do scheduler tạo sau khi chốt giải; xem [COMPETITIONS.md](COMPETITIONS.md). Trận dừng do guard/restart là aborted, không gán hòa hay điểm.

## Event và replay

```text
id: 18
event: arena
data: {"seq":18,"type":"state","server_unix_ms":1790000000000,"data":{"state":{...},"counts":{...}}}

```

Kinds: queued, phase, build, progress, turn_started, state, timeout_pending, timeout_claimed, completed, failed, aborted. Persist trước broadcast. State event chứa snapshot đầy đủ để reconnect/replay độc lập với DOM. Không reload cả trang. SSE connection đóng sau 60s; EventSource reconnect, gửi cursor. SSE/JSON chỉ phục vụ event của đội đã xác thực.

## Report

Artifact/source hashes, compiler version, flags, immutable image ID, engine/suite hashes, profile, startup samples, build log/time; per-turn samples có worker clock_id, start/receive/deadline ns dưới dạng **string**, wall_ms, cgroup_cpu_ms, memory_peak_bytes. `turn_wall_ms` là summary của mẫu thành công; lỗi không được biến thành một mẫu thời gian.

`sandbox_metrics` lấy cgroup CPU total, memory peak và memory.events sau kết thúc bot. Memory gồm supervisor; không gọi đó là peak RSS riêng của source. CPU delta mỗi turn cũng gồm supervisor. Có thể tải report partial khi validating/benchmarking lỗi; build lỗi xem event build và job.error, report có thể null.

## Lỗi

Envelope: `{"error":{"code":"...","detail":"..."}}`. 400 request/ZIP/runtime/protocol input sai; 401 auth; 403 Host/Origin; 404 record không thuộc đội/không tồn tại; 409 version/idempotency conflict; 429 quota; 503 worker unavailable. Bot fault nằm trong job/report, không phải status HTTP của lệnh tạo job. Không tự retry POST với key mới khi chưa biết kết quả; retry cùng key. Record và idempotency key commit cùng transaction SQLite. Sau restart, job queued được giữ, job đã claim/chạy dở bị abort; retry cùng key vẫn trả job cũ, không tự chạy trận lần hai.

Đồng hồ cả trận: Bullet 60.000ms hoặc đấu thường 300.000ms **mỗi bot**, không cộng giây. Observation thêm `payload.clocks` (red/blue, ms); `timing.time_control="match"`, `remaining_ms`, `initial_ms`, `increment_ms=0`. `budget_ms` là quỹ còn lại làm tròn xuống; dùng deadline tuyệt đối do worker cấp. Response/action giữ nguyên v2/v3. Benchmark chia chung quỹ cho 72 mẫu. Request `turn_ms` chỉ hỗ trợ cấu hình cũ; không gửi cùng `match_ms`. Xem [TIMING](TIMING.md).


Web API chính tại /api/v1: benchmark dự giải nhận {submission_id, competition_id} để ghim luật/quỹ; practice nhận sample_bot_id thay blue_submission_id để đấu bot BTC. Danh mục đội GET /sample-bots; admin GET/POST /admin/sample-bots và POST /admin/sample-bots/{id}/enabled. Upload là raw ZIP, headers X-Bot-Name (URI encoded), X-Bot-Level, X-Arena-Runtime, X-Arena-Protocol, Idempotency-Key; toggle JSON {enabled:true|false}. Không trả source/log bot BTC. Chi tiết schema hiện hành ở [openapi.json](openapi.json) và [BOT-LANGUAGES.md](BOT-LANGUAGES.md).
