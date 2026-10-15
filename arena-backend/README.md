# UET Bot Arena — backend OTTv2

Cài trên máy mới: dùng [INSTALL.md ở root repo](../INSTALL.md). Hướng dẫn đó tạo DB/BTC mới, build đủ worker image và chạy UI/API tại cổng 41810.

Backend chạy thật cho luồng **đăng ký → admin cấp tài khoản → ZIP C++ → benchmark → chốt bot → giải OTTv2 hai lượt → live/replay → xếp hạng**. Web API v0.6 dùng FastAPI, tài khoản/session và lịch giải lưu DB, phân quyền đội/admin, scheduler và OpenAPI. Worker/lab cũ giữ riêng để kiểm thử. UI chính đã nối API cùng origin trên WSL; chưa triển khai Internet hoặc cập nhật domain hosted. Không dùng kết quả lab nối 5 cũ để chứng nhận source mới.

## Bắt đầu với backend web mới

Đã có [chế độ khách xem trận](docs/GUEST-SPECTATOR.md): xem giải công khai, lịch, BXH, live/replay không cần tài khoản. Kiểm thử tải riêng bằng `make guest-load-test` (hai luồng SSE trên mỗi trang Arena).

Kiểm thử tải/sự cố có thể chạy lại bằng `make load-test`; xem [LOAD-RESILIENCE.md](docs/LOAD-RESILIENCE.md) cho phạm vi, cách đọc P95/HTTP 429/503 và giới hạn của phép đo local.

Hướng dẫn dành cho đội: [Viết bot OTTv2 C++17](../uet-bot-arena/dist/live/bot-guide.md). Bản này cũng có trong **Tài liệu & thể lệ** của UI, gồm format JSON Lines hiện tại, mẫu C++ hoàn chỉnh, input 81 ô để thử local, deadline và cách đọc lỗi/benchmark. Bot giao tiếp qua stdin/stdout của worker.

**Ruleset add-on** (Experimental, mặc định tắt): [pipeline nhập/xuất và công bố](docs/RULESET-ADDONS.md), [SDK v3 và cơ chế](docs/EXPERIMENTAL.md). BTC ghép cấu hình mô-đun, nhập gói JSON và kiểm tra/công bố trong một thao tác. Đội có thư viện luật, changelog sinh từ definition, tài liệu/SDK theo hash và playground legal_actions. Luật đã công bố bất biến; giải/benchmark ghim hash luật, SDK và image riêng. SDK v2/giải tiêu chuẩn giữ nguyên; tắt add-on không dừng trận đã chốt. Build và kiểm tra riêng bằng `make experimental-test`.

Trong thư mục `arena-backend` trên Linux/Ubuntu WSL2:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
docker build -t uet-arena-worker:v1 -t uet-arena-worker:clock-v2 -t uet-arena-worker:experimental-v1 -t uet-arena-worker:experimental-clock-v2 .
.venv/bin/python -m arena_api migrate
.venv/bin/python -m arena_api bootstrap-admin --username organizer
.venv/bin/python -m arena_api serve
```

Mở **http://127.0.0.1:4181/** để dùng UI + API cùng nhau. Xem [UI.md](docs/UI.md) để thử trọn luồng hai đội. API reference tại **http://127.0.0.1:4181/docs**, schema `/openapi.json`. CLI hỏi mật khẩu admin ẩn; không có mật khẩu mặc định. Đăng ký đội không cần password; admin duyệt/cấp mật khẩu tạm, đội bắt buộc đổi lần đầu. Xem [BACKEND.md](docs/BACKEND.md) cho kiến trúc, auth/CSRF, API contract, migration, kiểm thử và triển khai. Adapter cho UI: `examples/client.mjs`.

Nếu WSL truy cập được nhưng Windows báo không thể kết nối, kiểm tra từ PowerShell bằng `netsh interface ipv4 show excludedportrange protocol=tcp`. Ngày 2026-10-11, máy local dành riêng dải 4164–4263, bao gồm 4181. Khi đó dừng backend cũ bằng Ctrl+C, rồi chạy trong thư mục `arena-backend` trên WSL:

```bash
ARENA_API_ORIGIN=http://127.0.0.1:41810 .venv/bin/python -m arena_api serve --port 41810
```

Mở **http://127.0.0.1:41810/#/login**. Phải đổi cả `ARENA_API_ORIGIN` và `--port` để xác thực Origin/CSRF khớp. Xác nhận từ Windows bằng `curl.exe http://127.0.0.1:41810/api/v1/health/ready`; chỉ kiểm tra bên trong WSL chưa đủ chứng minh trình duyệt Windows truy cập được.

Để chạy nền tách khỏi terminal/chat, mở Docker Desktop rồi chạy `powershell -File .\scripts\start-local.ps1` từ `arena-backend` trên Windows. Script dùng Ubuntu WSL, cổng 41810, giữ DB hiện tại và ghi log vào `.platform-state/logs/`; nếu server đã phục vụ thì không khởi động thêm. Dừng bằng SIGINT/SIGTERM cho đúng process Python `arena_api serve --port 41810` trong WSL, sau khi kiểm tra không có trận đang chạy. Máy/WSL tắt thì cần mở lại dịch vụ.

Giải chính thức: [COMPETITIONS.md](docs/COMPETITIONS.md). Vòng tròn hai lượt, thắng 3đ, hòa 1đ, thua 0đ; xếp theo tổng điểm và đồng hạng khi bằng điểm. Chốt đúng source/binary đã benchmark; lỗi hạ tầng không tính điểm, admin xem/retry attempt; restart tạm dừng giải để kiểm tra. Chạy `make integration` để kiểm cả worker, tài khoản/luyện tập và giải giữa hai đội.

Quản trị BTC: [ADMIN.md](docs/ADMIN.md) — dashboard metrics, thêm/sửa/xóa mềm/khôi phục đội và giải, quản lý tài khoản, job/report và nhật ký thao tác.

Các hướng dẫn port 4180 và team token bên dưới áp dụng cho **worker lab v1**, không phải lớp web API mới.

## Đã có

- Engine theo [luật OTTv2 bạn cung cấp](docs/OTTV2-RULES.md): đúng bố trí 18 quân, Đỏ trước, 8 hướng/1 ô, khắc chế, căn cứ, diệt loại, bí nước và đầu hàng. Tọa độ `a1 = row 8, col 0`; `i9 = row 0, col 8`.
- HTTP control API có scope theo đội, source/version bất biến, SHA-256, idempotency, giới hạn upload/queue và SQLite lưu trạng thái/event.
- Hỗ trợ C++17, C11 và Python 3.11 (PyPy) trong container, cùng quỹ giờ; không bao giờ chạy source của đội trực tiếp trên host.
- Compiler và bot chạy trong Docker Linux. Mỗi bot một container, process được dừng bằng SIGSTOP khi không tới lượt. Source không nhận command/build script của đội.
- JSON Lines protocol v2, init/ready, chống phản hồi cũ/nhầm trận, clock do trusted supervisor trong worker đo.
- 24 trạng thái OTTv2 phía Đỏ × 3 lần, 72 mẫu không warm-up; báo median/p95/max, CPU cgroup, peak memory toàn container, startup/build/log và provenance.
- Web API có giải giữa các đội: draft/open/locked/running/paused/completed, snapshot qualification, lịch hai lượt lưu bền vững, bảng thắng/thua và stream đã lọc dữ liệu riêng. Console lab port 4180 chỉ ghép hai phiên bản của **cùng đội** luyện tập, xem board cập nhật tại chỗ và tua event log; quản lý giải qua UI chính và API v0.5.

## Chạy nhanh (Linux hoặc Ubuntu WSL2)

Yêu cầu: Python 3.11+, Docker Engine Linux hoạt động, cgroup v2 với `cpu.stat`, `memory.peak`, `memory.events`. Worker lab không cần dependency pip; web API và bộ test đầy đủ dùng venv/requirements ở trên. Docker Desktop cần bật integration cho distro Ubuntu nếu dùng WSL.

```bash
cd /path/to/arena-backend
docker build -t uet-arena-worker:v1 .
.venv/bin/python -m unittest discover -s tests -v
python3 -m arena.server --dev --port 4180
```

Mở **http://127.0.0.1:4180**. `--dev` cố ý cấp một session local cho `local-practice` khi mở trang gốc; chỉ dùng máy cá nhân. Cookie HttpOnly, SameSite=Strict; token không lưu localStorage. Host/Origin lạ bị chặn. Không dùng `--dev` sau reverse proxy hoặc trên server công khai.

Tải starter trong console, ZIP các file ở thư mục gốc, nộp `v1.0.0`, chạy benchmark rồi tạo trận với hai phiên bản. Có thể cho cùng một artifact đấu chính nó; vẫn tạo hai container riêng.

Chế độ có team token (chưa có hệ thống phát tài khoản/mật khẩu production):

```bash
# Cấp qua secret manager/env; không commit token. Map token ngẫu nhiên -> team ID.
export ARENA_TEAM_TOKENS='{"<random-token-at-least-24-characters>":"team-a"}'
python3 -m arena.server --port 4180
```

Bearer token cho CLI, hoặc đăng nhập console để đổi sang session 1 giờ. API chỉ bind loopback. Trước khi đưa ra Internet cần HTTPS gateway, `ARENA_PUBLIC_ORIGIN=https://your-domain`, `ARENA_COOKIE_SECURE=1`, rate limits và hệ thống auth/role chính thức; xem [security](docs/SECURITY.md).

## Time constraints — đã chốt cho bản này

| Giới hạn | Giá trị / cách áp dụng |
|---|---|
| Tổng thời gian / mỗi bot | Bullet 60s hoặc đấu thường 300s (mặc định). Chỉ trừ khi đến lượt, không increment. Giải cũ đã chốt giữ thời gian mỗi lượt. |
| Mốc bắt đầu `t0` | Worker lấy `monotonic_ns` ngay trước serialize/gửi observation vào pipe của bot. Bao gồm dispatch, ghi pipe, bot parse/tính/serialize, đọc đủ action + LF. |
| Deadline cứng `D` | `t0 + remaining_ns` (quỹ còn lại, không reset). Chỉ chấp nhận frame được worker quan sát đầy đủ khi **t1 < D**. Đúng D cũng muộn. Không tin timestamp do bot tự khai. |
| Khoảng bù | **3.000 ms**, chỉ chờ trước khi server đại diện đối thủ tuyên bố hết giờ. Không nhận action muộn trong khoảng bù. Đây là lựa chọn bạn đã xác nhận. |
| Quyền tuyên bố | Tạo trận phải gửi `delegate_timeout_claims: true`, đại diện cả hai bot luyện tập. API v1 chưa có chế độ tuyên bố thủ công. |
| Startup | 2.000 ms init→ready, không trừ vào lượt. V2: lỗi do bot được supervisor xác nhận là thua kỹ thuật; hai bot lỗi thì cả hai nhận một trận thua. Lỗi transport/container không tính kết quả. |
| Build | 30.000 ms trong container; host có thêm 15s cho Docker/control transport. Tách khỏi thời gian suy nghĩ. |
| RAM | 256 MiB/container bot; 1.024 MiB/container build; swap = RAM, không thêm swap. |
| CPU | Tối đa 1 CPU/container; bot single-thread/single-process. Clock xử thua là wall time; CPU delta là metric riêng. Không có hệ số bù theo ngôn ngữ. |
| Output | JSON frame tối đa 65.536 byte kể cả LF; stderr tổng tối đa 256 KiB/process. Không dùng stdout để log. |
| Kết thúc trận v2 | Lặp cùng bàn/bên tới lượt 3 lần, hoặc 512 ply/20 phút → hòa. Kiểm tra giới hạn giữa các lượt; không cắt ngang lượt đang chạy. Shutdown/hỏng hạ tầng không phải hòa. Giải v1 giữ guard/startup ở trạng thái cần BTC xử lý. |

Lỗi protocol, action bất hợp lệ, crash hoặc output limit khi đang đấu là thua kỹ thuật. Lỗi controller/transport/worker là job lỗi, **không** quy thành bot hết giờ. Nếu restart controller, job dở dang thành `aborted`; không resume từ clock của phiên cũ.

Chi tiết phiên bản thể lệ, hòa/startup và hàng đợi ưu tiên: [MATCH-POLICY.md](docs/MATCH-POLICY.md). Queue SQLite ưu tiên official → benchmark → practice, FIFO trong nhóm; không còn cap 16 toàn hệ thống. N worker ghim CPU, cấu hình ARENA_WORKERS. Không cắt ngang job đang chạy.

Worker không phải hệ điều hành real-time. Nếu scheduler đọc frame muộn, mốc worker quan sát vẫn quyết định; nên giữ dư thời gian. Bộ test công khai không bảo đảm mọi trạng thái đều kịp. P95 dùng nearest rank, median dùng trung bình hai mẫu giữa khi số mẫu chẵn; timeout không được ghi thành latency thành công.

## API bot và clock sync

Bot **không gọi HTTP qua Internet để gửi từng nước đi**. Bot giao tiếp với trusted worker qua stdin/stdout JSON Lines; network bị tắt. Engine/worker xử deadline tại chỗ. HTTP chỉ nộp source, tạo job, đọc report/event và đồng bộ clock hiển thị.

Nhờ vậy, lệch UTC/NTP hoặc RTT của máy đội/trình duyệt không tham gia phán quyết. Không so sánh monotonic timestamp của hai worker: mỗi frame mang `clock_id` riêng. Đồng hồ trong browser nội suy bằng `performance.now()`, lấy offset/RTT từ `/api/v1/time`; worker/host có probe riêng để ánh xạ thời gian hiển thị. Sai số được hiển thị; countdown không có quyền xử thua.

Xem [PROTOCOL.md](docs/PROTOCOL.md) cho schema bot và [TIMING.md](docs/TIMING.md) cho thuật toán timing, clock sync, grace, pause và reconnect. [API.md](docs/API.md) mô tả đầy đủ control endpoints, request/response và lỗi.

## Kiểm thử

```bash
.venv/bin/python -m unittest discover -s tests -v
# Phải có image/daemon thật; lệnh này không bỏ qua Docker lỗi.
python3 tests/integration.py
```

Integration dựng HTTP server thật trên loopback, nộp/biên dịch starter, chạy 72 mẫu, chạy trận, đọc event/replay, kiểm tra quyền khác đội và các bot vi phạm deadline/protocol. Kết quả nằm trong `test-results/latest.json` (ignored). Không giả lập số đo nếu Docker không sẵn sàng.

Xem [kết quả kiểm thử local](docs/VALIDATION.md) cho vòng kiểm thử đã chạy, số đo và các giới hạn của kết luận.

## Dữ liệu và tái lập

`.state/arena.sqlite` lưu submissions/jobs/events/idempotency; `.state/sources/` chứa source chỉ đọc; `.state/artifacts/<build-key>/` lưu binary và manifest. Không đặt `.state` trong thư mục static. Build key chứa hash source, immutable Docker image ID, compiler flags và SDK hash. Binary phải đúng hash trước khi dùng lại; engine/suite hash nằm trong report. Đổi image không sửa artifact cũ.

`ARENA_WORKER_IMAGE` chỉ admin cấu hình; mỗi job resolve tag thành immutable image ID. Không pull khi chạy job. Image build lần đầu có tải dependency mạng; runtime/build source của đội đều `--network=none`.

## Phạm vi cần làm trước giải thật

Đã có UI nối API v0.5, auth/password/team-admin roles và quản lý giải hai lượt. Chưa triển khai hosting, retention/backup vận hành, scheduling giữa máy, chống worker nhiễu hoặc Codon v2. Docker là lớp cô lập của bản beta; cần review và chạy trên host/VM riêng trước khi nhận source không tin cậy ở Internet. Bộ luật được hiện thực từ văn bản đính kèm; file JS gốc và unit test tham chiếu trong văn bản chưa được cung cấp để đối chiếu.

Site đang hosted vẫn là demo frontend. UI chính nối API thật tại port 4181; console worker lab tại 4180 giữ riêng. Bước triển khai sau là đưa UI + control API + worker lên hạ tầng riêng qua HTTPS cùng origin. Cloudflare static/Workers không chạy native compiler/container của nền tảng này.

Tham khảo nền tảng: [Python monotonic clock](https://docs.python.org/3/library/time.html#time.monotonic_ns), [Docker resource constraints](https://docs.docker.com/engine/containers/resource_constraints/), [Docker security](https://docs.docker.com/engine/security/).

### Đồng hồ cả trận

Request tạo giải/trận/benchmark mới dùng `match_ms: 60000` hoặc `300000`; xem [TIMING](docs/TIMING.md). Benchmark dùng chung một quỹ cho 72 mẫu. Action JSON Lines v2/v3 giữ nguyên. Xem [hướng dẫn bot](../uet-bot-arena/dist/live/bot-guide.md) để chia thời gian search.

Build worker mới riêng để giữ image/binary của các giải cũ:
```sh
docker build -t uet-arena-worker:clock-v2 -t uet-arena-worker:experimental-clock-v2 .
.venv/bin/python tests/time_bank_integration.py
```


Hướng dẫn ba ngôn ngữ, luật cơ bản theo giải, bot mẫu và chính sách thời gian: [BOT-LANGUAGES.md](docs/BOT-LANGUAGES.md). Swiss 2–4096 đội, hai ván đổi màu, điểm/Buchholz: [COMPETITIONS.md](docs/COMPETITIONS.md). Thử tải nhiều đội: `tests/load_lab/` (xem INSTALL.md).
