# Cài và thử UET Bot Arena trên máy của bạn

Hướng dẫn này dành cho checkout mới, dữ liệu mới và chạy local. Web chính gồm **UI + FastAPI + worker Docker**. Không cần chạy `npm run dev` hoặc cấu hình domain hosted.

## 1. Chuẩn bị máy

Môi trường đã dùng để kiểm chứng: Linux x86_64/Ubuntu WSL2, Python 3.12, Docker Linux containers, cgroup v2. Backend yêu cầu Python **3.11+**. Nên dành ít nhất 4 GB RAM cho Docker để chạy các bài thử cơ bản; mỗi build có giới hạn 1 GiB, mỗi container bot 256 MiB. Đây không phải ước lượng sức chứa giải đông đội.

### Windows 10/11 + WSL2

Nếu chưa có WSL, mở PowerShell với quyền Administrator:

```powershell
wsl --install -d Ubuntu-24.04
```

Khởi động lại Windows nếu được yêu cầu, mở Ubuntu và tạo Linux username/password. Cài Docker Desktop từ nguồn chính thức, chọn WSL2 backend/Linux containers; vào **Settings → Resources → WSL integration**, bật integration cho distro bạn dùng rồi Apply/Restart. Mở Docker Desktop và đợi engine hoạt động.

Từ terminal **Ubuntu**:

```bash
sudo apt-get update
sudo apt-get install -y git python3 python3-venv python3-pip make ca-certificates
python3 --version
docker version
docker info --format '{{.OSType}} {{.CgroupVersion}}'
```

Dòng cuối phải có `linux 2`. Nếu Ubuntu hiện có dùng Python 3.10, dùng distro Ubuntu 24.04 hoặc cài Python 3.11+ trước khi tạo venv; không thay Python hệ thống bằng symlink.

### Linux trực tiếp

Cài Git, Python 3.11+ với venv, make và Docker Engine theo hướng dẫn chính thức của distro. Người chạy API phải có quyền gọi Docker; kiểm tra **`docker info` không cần sudo** trước khi tiếp tục. Quyền dùng Docker là quyền đặc biệt trên host; chỉ cấp cho người vận hành nền tảng.

Không hỗ trợ chạy worker bằng Python Windows native hoặc Docker Windows containers. macOS chưa nằm trong phạm vi kiểm chứng; nếu cần, dùng Linux VM phù hợp.

## 2. Clone và setup

Các lệnh sau chạy trong **Ubuntu/Linux**, không phải PowerShell:

```bash
cd ~
git clone https://github.com/khoinguyencode/platform.git
cd platform
bash scripts/setup.sh
```

Script kiểm tra Python, Docker/Linux/cgroup v2, tạo `arena-backend/.venv`, cài dependency đã pin, build một worker image với bốn tag và migrate DB mới:

```text
uet-arena-worker:v1
uet-arena-worker:clock-v2
uet-arena-worker:experimental-v1
uet-arena-worker:experimental-clock-v2
```

Build lần đầu cần Internet để tải base image và package. Không cần compiler C++ trên host; source bot được build trong Docker. Runtime bot/build source đội dùng network=none. `.env.example` chỉ là mẫu; ứng dụng không tự đọc `.env`.

Nếu muốn chạy từng bước để xem lỗi:

```bash
cd ~/platform/arena-backend
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
docker build -t uet-arena-worker:v1 -t uet-arena-worker:clock-v2 \
  -t uet-arena-worker:experimental-v1 -t uet-arena-worker:experimental-clock-v2 .
.venv/bin/python -m arena_api migrate
```

`setup.sh --skip-worker-build` chỉ dùng khi đã có đủ bốn image, chẳng hạn kiểm thử lại checkout bằng daemon có sẵn. Máy mới chạy setup bình thường. Không chạy setup/migrate/bootstrap trong lúc controller của cùng data directory đang hoạt động.

## 3. Tạo tài khoản BTC của riêng bạn

```bash
cd ~/platform/arena-backend
.venv/bin/python -m arena_api bootstrap-admin --username organizer
```

CLI hỏi mật khẩu ẩn và xác nhận, 8–128 ký tự. **Repo không có credential mặc định**. Ghi nhớ username và mật khẩu bạn vừa chọn. Không đưa mật khẩu vào lệnh shell hoặc Git.

Chỉ bootstrap một lần cho DB mới. Khi đã có BTC, dùng tài khoản đó và UI quản trị. Tài khoản/DB trên máy người phát triển không được chia sẻ trong repo.

## 4. Khởi động và kiểm tra

```bash
cd ~/platform
bash scripts/start.sh
```

Giữ terminal này mở. Trên trình duyệt máy bạn, mở:

- Web: **http://127.0.0.1:41810/#/login**
- Liveness: **http://127.0.0.1:41810/api/v1/health/live**
- Worker readiness: **http://127.0.0.1:41810/api/v1/health/ready**
- API reference: **http://127.0.0.1:41810/docs**

```bash
curl --fail http://127.0.0.1:41810/api/v1/health/ready
```

Kết quả cần có `ready: true`, `database: ready`, `competition_scheduler: ready` và `image_id: sha256:...`. Liveness/UI chạy được chưa có nghĩa Docker đã sẵn sàng. UI và API dùng cùng origin; dùng đúng **127.0.0.1** theo URL, không đổi sang localhost ở giữa phiên.

Đổi cổng:

```bash
bash scripts/start.sh 41811
```

Script tự đặt origin tương ứng, trừ khi bạn đã export `ARENA_API_ORIGIN`. Nếu tự cấu hình environment, origin và port phải khớp. API bind loopback; không mở ra LAN/Internet bằng hướng dẫn này.

## 5. Test một giải với hai đội

1. Đăng nhập BTC `organizer` bằng mật khẩu tự tạo. Trong **Đội & tài khoản**, dùng **Thêm đội & cấp tài khoản** để tạo hai đội, hoặc gửi hai hồ sơ qua **Đăng ký đội** rồi BTC duyệt. Điền thông tin thử, không dùng danh tính thật nếu chỉ test.
2. BTC cấp username và mật khẩu tạm cho mỗi đội. Credential chỉ hiện ở hộp thoại cấp/reset; lưu lại cho bài test. Chức năng này không gửi email.
3. Đăng xuất BTC, đăng nhập đội 1 và đổi mật khẩu tạm ở lần đầu. Muốn mở nhiều đội cùng lúc, dùng browser profile/phiên riêng; các tab cùng profile chia sẻ session.
4. Vào **Nộp bot**, chọn ngôn ngữ (C++17, C11 hoặc Python 3.11) rồi **Tải starter OTTv2**. Nộp nguyên ZIP với version `v1.0.0`, chọn **SDK v2**. ZIP có file chính (`bot.cpp`, `bot.c` hoặc `bot.py`) ở gốc, không có thư mục bọc ngoài, binary hoặc build script.
5. Vào **Test & benchmark**, chọn bản bot, **OTT v2 tiêu chuẩn**, **đấu thường 5 phút/đội**. Chờ hoàn tất **72/72 mẫu** và `passed`. Có thể thử **Luyện tập** bằng cùng một version ở cả hai màu.
6. Làm tương tự với đội 2. BTC vào **Giải đấu**, tạo giải tiêu chuẩn hai đội, chọn **5 phút/đội** và **Mở đăng ký**.
7. Mỗi đội mở giải, chọn benchmark đã đạt và xác nhận ủy quyền xử lý timeout. Nếu không có benchmark để chọn, kiểm tra đúng luật/hash, quỹ thời gian, image và phiên bản bot.
8. BTC **Chốt bot & sinh lịch**, rồi **Bắt đầu giải**. Hai đội sẽ đấu hai fixture đổi Đỏ/Xanh. Chỉ tạo lịch chưa làm giải tự chạy; cần bấm Bắt đầu.
9. Mở **Trực tiếp** để xem bàn cờ và chuyển trận; xem **Bảng xếp hạng**, lịch và replay. Một cửa sổ khách có thể xem giải công khai mà không đăng nhập.

Thắng 3 điểm, hòa 1 điểm, thua 0 điểm, bằng điểm đồng hạng. Luyện tập không tính hạng. Lỗi worker/hạ tầng đưa trận vào trạng thái BTC cần xử lý, không tự tính đội thua.

## 6. Test Ruleset add-on

1. BTC mở **Ruleset add-on**, bật công tắc. Chọn preset và lưu/kiểm tra/công bố, hoặc nhập `arena-backend/examples/ruleset-addons/energy-dash.addon.json` và chọn công bố ngay.
2. Đội mở **Chi tiết luật** để xem changelog, vùng/mẫu đi/chi phí/cooldown và thử legal_actions. Tải **Starter nộp bot**, không nộp nguyên **Gói luật & tài liệu** có các file JSON.
3. Nộp source theo **SDK v3**. Chọn đúng ruleset đã công bố khi benchmark và luyện tập. Bot chỉ v2 không đủ điều kiện chạy add-on.
4. BTC **Tạo giải với luật này**, dùng cùng thời gian mà hai đội đã benchmark. Đăng ký/chốt/bắt đầu như giải tiêu chuẩn.

SDK giao tiếp JSON Lines qua stdin/stdout, không gọi HTTP cho từng nước. `init` có definition/hash; observation có `legal_actions`; bot chọn action_id hiện tại và sao chép envelope. Xem [hướng dẫn add-on](arena-backend/docs/RULESET-ADDONS.md) và [protocol v3](arena-backend/docs/EXPERIMENTAL.md).

## 7. Dừng, chạy lại và giữ dữ liệu

Ctrl+C trong terminal chạy `scripts/start.sh` để dừng. Chờ shutdown xong, nhất là khi có job đang xử lý. Lần sau mở Docker rồi chạy lại `bash scripts/start.sh`; không cần bootstrap lại.

DB, account, source upload, artifact và event log nằm trong `arena-backend/.platform-state/` (ignored). Khởi động lại giữ account/kết quả; job đang dở không resume đồng hồ, giải đang chạy được tạm dừng để BTC kiểm tra. Backup khi controller đã dừng và giữ cả DB **lẫn** source/artifact; không chỉ sao chép riêng file SQLite đang ghi.

Muốn một bộ dữ liệu test riêng:

```bash
cd ~/platform/arena-backend
export ARENA_API_DATA="$PWD/.platform-state-test"
.venv/bin/python -m arena_api migrate
.venv/bin/python -m arena_api bootstrap-admin --username organizer
cd ..
bash scripts/start.sh 41811
```

Giữ cùng `ARENA_API_DATA` cho migrate/bootstrap/serve. Một controller cho mỗi data directory. Không chạy migrate/bootstrap trong khi server đang dùng directory đó. Thư mục `.platform-state-test` cũng được ignore.

Nếu clone trên ổ Windows và muốn chạy nền từ PowerShell, sau khi đã setup trong WSL:

```powershell
cd C:\path\to\platform
powershell.exe -File .\arena-backend\scripts\start-local.ps1 -WslDistro Ubuntu-24.04 -Port 41810
```

Thay WslDistro bằng tên trong `wsl --list --verbose`. Nếu clone dưới Linux home, dùng terminal WSL và `scripts/start.sh`; không dùng helper đường dẫn Windows. Helper ghi log vào `.platform-state/logs/` và không tạo tài khoản mới.

## 8. Kiểm thử tự động

```bash
cd ~/platform/arena-backend
make test
make integration
make experimental-test
```

Unit/API tests dùng DB tạm. Integration thực sự biên dịch/chạy C++, cần daemon và image; không giả lập hoặc bỏ qua Docker lỗi. Các bài integration/load lưu báo cáo riêng ở `test-results/`, không dùng DB của bạn. Bài tải không chứng nhận số trận đồng thời tối đa trên host khác.

Kiểm thử UI cần Node.js 22+ (không cần npm install):

```bash
cd ~/platform/uet-bot-arena
npm run check
node --test tests/*.test.mjs
```

Trên PowerShell dùng `npm.cmd run check` nếu policy chặn npm.ps1. Prototype tham chiếu: `python3 scripts/package-starters.py` rồi `npm run dev`; đây là demo static cũ, không dùng để kiểm thử API/đăng nhập/nộp bot thật.

## 9. Lỗi thường gặp

| Hiện tượng | Kiểm tra / xử lý |
| --- | --- |
| docker không có trong Ubuntu | Bật Docker Desktop và WSL integration đúng distro; mở lại terminal |
| Permission denied với Docker socket | Kiểm tra quyền dùng Docker trên Linux; chạy `docker info` bằng đúng user sẽ chạy API |
| Worker readiness 503 | Docker chưa lên, thiếu image, cgroup không phù hợp; chạy setup và đọc detail trong health response |
| Address already in use | Đã có server ở cổng đó; kiểm tra process hoặc chọn cổng khác bằng start.sh |
| Windows không truy cập được nhưng WSL curl được | Kiểm tra localhost forwarding/WSL, firewall và `netsh interface ipv4 show excludedportrange protocol=tcp`; đổi cổng cả origin/server nếu cần |
| Login/CSRF/Origin 403 | Dùng đúng 127.0.0.1/port; bỏ environment origin cũ rồi restart, đăng nhập lại |
| Another controller owns this data directory | Dừng đúng controller cũ hoặc dùng data directory khác; không xóa lock để vượt kiểm tra |
| Python thiếu venv hoặc dưới 3.11 | Cài python3-venv và Python phù hợp rồi tạo venv mới |
| Không có bot đủ điều kiện dự giải | Benchmark phải đạt 72 mẫu và khớp cấu hình đã ghim; nộp version mới nếu source thay đổi |
| Compile error / protocol_error / illegal_action | Xem log job/request ID và hướng dẫn bot; chỉ ghi protocol trên stdout, log ra stderr, chọn action từ observation hiện tại |
| Demo hiện tài khoản/số liệu mẫu | Bạn đang chạy npm dev/prototype; mở API UI ở cổng 41810 |
| Docker báo dockerInference / engine.sock không truy cập được | Lỗi runtime Docker Desktop trên Windows; đọc mục sự cố trong SECURITY.md. Không factory reset/xóa volume chỉ để thử lại |

Tài liệu kỹ thuật: [backend](arena-backend/docs/BACKEND.md), [quản trị](arena-backend/docs/ADMIN.md), [thể thức giải](arena-backend/docs/COMPETITIONS.md), [timing](arena-backend/docs/TIMING.md), [bảo mật](arena-backend/docs/SECURITY.md).

## Worker, ngôn ngữ và hàng đợi

Python của server cần 3.11+. Bot Python chạy bằng **PyPy 7.3.23 (Python 3.11)** trong image worker, chỉ thư viện chuẩn, không pip. Ba ngôn ngữ `cpp17`, `c11`, `python3` dùng cùng giao thức và cùng quỹ giờ; C/C++ thường nhanh hơn Python. Xem [hướng dẫn ngôn ngữ](arena-backend/docs/BOT-LANGUAGES.md) và [số đo Python/C++](arena-backend/docs/PYTHON-COMPARISON.md).

Một tiến trình Uvicorn giữ khóa cho mỗi thư mục dữ liệu; bên trong có N worker, mặc định bằng số CPU trừ 2. Cấu hình bằng biến môi trường trước khi chạy (file `.env` không được tự đọc):

```bash
export ARENA_WORKERS=4
export ARENA_BUILD_WORKERS=2
export ARENA_CONNECTION_LIMIT=1024
# Chỉ khi có reverse proxy đã biết:
# export ARENA_TRUSTED_PROXIES=127.0.0.1/32,::1/128
bash scripts/start.sh
```

Mỗi trận chạy trên một CPU riêng; hai bot dùng chung CPU đó và bot không tới lượt bị đóng băng. Hàng đợi nằm trong SQLite, ưu tiên trận chính thức → benchmark → luyện tập, không giới hạn tổng số job. Mỗi đội tối đa 4 tác vụ đang chạy, 4 yêu cầu đang chờ và 200 tác vụ lưu trữ. Yêu cầu mới **cùng mục đích** (benchmark cho cùng giải hoặc cùng cấu hình giờ/luật, hay trận luyện tập mới) thay yêu cầu cũ còn đang chờ; tác vụ đang chạy không bị hủy. API trả `queue_position`. Khởi động lại giữ các job đang chờ; job đang chạy dở bị hủy, không tính kết quả.

Khi khởi động, server dọn container sót của chính thư mục dữ liệu này rồi kiểm tra các image worker; trong vài giây đầu `/health/live` đã trả 200 nhưng `/health/ready` còn 503. Image đã dùng cho giải được giữ thêm tag `uet-arena-pinned:<digest>`; đừng xóa image/artifact mà giải còn dùng. Backup SQLite, `sources`, `artifacts` và `compiled-cache` khi server đã dừng.

BTC tạo giải Swiss (tới 4096 đội) hoặc vòng tròn hai lượt (tới 32 đội), có thể tùy chỉnh bàn 7–15 hàng/cột, căn cứ và điều kiện thắng; luật được giữ cố định theo giải. Đội benchmark từ trang giải để dùng đúng luật và quỹ giờ. Xem [COMPETITIONS.md](arena-backend/docs/COMPETITIONS.md).

## Thử tải nhiều đội

`arena-backend/tests/load_lab/` tạo nhiều đội ảo, mỗi đội một địa chỉ IP loopback, cùng đăng nhập → nộp bot (nhiều thuật toán và các bot lỗi cố ý) → benchmark → đấu giải:

```bash
cd arena-backend
.venv/bin/python tests/load_lab/e2e_submissions.py --teams 100 --tournament 8
.venv/bin/python tests/load_lab/algo_eval.py scaling --workers 1,2,4,8
```

Dữ liệu và báo cáo nằm ở `test-results/load-lab/`, tách khỏi dữ liệu thật.
