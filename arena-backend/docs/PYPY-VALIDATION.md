# Kiểm tra PyPy trong sandbox OTTv2

Image `sha256:fa371d5d81f7cd07f2e55a441e6086a8ca6fddb5201b9b5e8398319835ffcbbc`; PyPy trong worker, RLIMIT_AS và container 256 MiB; giữ nguyên seccomp. Không nới bộ nhớ, không tắt JIT để tạo số đo đẹp.

| Bot | Mong đợi | Thực tế | Đạt |
|---|---|---|---|
| starter | ok | ok | True |
| alphabeta | ok | ok | True |
| defensive | ok | ok | True |
| random | ok | ok | True |
| mcts | ok | ok | True |
| crash | crash | crash | True |
| timeout | timeout | timeout | True |
| spam | output_limit | output_limit | True |
| stale | stale_action | stale_action | True |
| illegal | illegal_action | illegal_action | True |
| memhog | crash | crash | True |
| sneaky | ok | ok | True |
| slow_start | timeout | timeout | True |

Mỗi bot đều được build/chạy riêng. JSON đầy đủ gồm startup, RAM cgroup, stderr và telemetry nếu bot chạy được.

Kết luận: Đạt bộ tương thích. PyPy 7.3.23 / Python 3.11 là runtime mặc định cho bot Python; tốc độ so với CPython và C++ xem [PYTHON-COMPARISON.md](PYTHON-COMPARISON.md). Mọi ngôn ngữ cùng quỹ giờ, không nhân hệ số.

Nếu PyPy không khởi động được trong hạn mức, nodes/s và độ sâu 100 ms trong điều kiện này là không đo được, không dùng số đo host ngoài sandbox thay thế.