# Bot Python, C và C++

Ba ngôn ngữ dùng **cùng giao thức JSON Lines v2** qua stdin/stdout. Server không dịch code sang ngôn ngữ khác: mỗi bot là một process trong container riêng, server làm trọng tài và chuyển nước đi giữa hai process. Vì vậy bot Python đấu được với bot C++.

| `X-Arena-Runtime` | ZIP phẳng | Build / chạy |
|---|---|---|
| `cpp17` | bot.cpp, các .cpp/.hpp/.h | `g++ -std=c++17 -O2`, chạy file nhị phân |
| `c11` | bot.c, các .c/.h | `gcc -std=c11 -O2 -lm`, chạy file nhị phân |
| `python3` | bot.py, các module .py | kiểm cú pháp, đóng gói zipapp, chạy bằng PyPy 7.3.23 (Python 3.11), chỉ thư viện chuẩn |

Trên trang Nộp bot, chọn ngôn ngữ trước khi tải starter. API: `GET /api/v1/starter.zip?runtime=python3` (hoặc `c11`, `cpp17`). Starter nào cũng đã lo phần giao thức; đội chỉ sửa một hàm chọn nước (`choose_move` trong C/C++, `choose` trong Python). Starter C dùng jsmn (giấy phép MIT giữ trong `sdk/jsmn.h`, upstream commit `25647e692c7906b96ffd2b05ca54c097948e879c`).

Không nhận binary, build script, pip package, thư viện native hoặc ZIP bọc thư mục. Python không có pip/mạng. Seccomp chặn fork/thread/socket; RLIMIT_AS và RAM container 256 MiB. Mỗi bot sống suốt trận và bị đóng băng khi không tới lượt. Source của đội không bao giờ chạy trực tiếp trên host.

## Thời gian

**Cùng quỹ giờ cho mọi ngôn ngữ, không nhân hệ số:** Bullet 60 giây hoặc thường 300 giây mỗi bot cho cả trận, không cộng giây, trên mọi kích thước bàn. C/C++ thường tính được nhiều nước hơn Python trong cùng thời gian; chọn ngôn ngữ là quyết định của đội. Số đo trên OTTv2 nằm ở [PYTHON-COMPARISON.md](PYTHON-COMPARISON.md), kiểm tra tương thích PyPy ở [PYPY-VALIDATION.md](PYPY-VALIDATION.md).

Khởi động (init → ready) có 2 giây riêng. Benchmark 72 mẫu dùng chung một quỹ. Mỗi observation có `timing.budget_ms` là quỹ còn lại; bot cần chia cho các lượt sau và chừa dư cho JSON/pipe. Đồng hồ monotonic của worker quyết định, không dùng timestamp của bot. Worker image ID, compiler/interpreter thực tế được ghi trong artifact/report và ghim vào giải.

## Luật theo giải

BTC có thể chọn bàn 7–15 hàng/cột, vị trí căn cứ, bật/tắt thắng bằng chiếm căn cứ, thắng khi diệt sạch 1 hoặc 2 loại quân, và xếp quân tự động (hình răng cưa của OTTv2, 9 quân mỗi bên, đối xứng tâm) hoặc nhập tay. Server từ chối luật cho phép một bên thắng ngay ở nước đầu.

`init.payload.basic_rules` và `observation.payload.basic_rules` chứa `{definition, hash}`; `state.basic_rules` chứa trực tiếp definition đã chuẩn hóa. Tọa độ là chữ cột + số hàng: `a1`, `j10`, `o15`; hàng 0 của mảng board là hàng mang số lớn nhất. Đọc kích thước từ mảng board, đừng viết cứng 9×9. Bot vẫn chỉ trả `{from,to}` hoặc `{resign:true}`; `legal_moves` đã được server tính đúng luật. Add-on năng lượng/khiên là SDK v3 riêng, mặc định tắt.

Để dự giải, gửi benchmark `{submission_id, competition_id}`: server dùng nguyên luật và quỹ giờ đã ghim của giải.

## Hàng đợi của đội

Mỗi đội tối đa 4 tác vụ đang chạy, 4 yêu cầu đang chờ và 200 tác vụ lưu trữ. Các yêu cầu đang chờ **cùng mục đích** dùng chung một chỗ: benchmark cho cùng một giải (hoặc cùng cấu hình giờ/luật khi không gắn giải), và trận luyện tập. Gửi yêu cầu mới vào chỗ đó thì yêu cầu cũ chưa chạy được hủy với mã `superseded`, vì kết quả cũ không còn cần nữa; gửi lại y hệt yêu cầu đang chờ thì giữ nguyên vị trí cũ. Benchmark cho giải khác không bị ảnh hưởng, và tác vụ đang chạy không bao giờ bị hủy.

## Bot mẫu cho luyện tập

BTC vào Vận hành bot để tải ZIP, đặt tên, mức Dễ / Trung bình / Khó / Rất khó và bật/tắt. Bot mẫu **chỉ dùng cho luyện tập**: không dự giải, không tính điểm. Đội chọn bot mẫu làm đối thủ trong trang Luyện tập và chọn bot của mình cầm Đỏ hay Xanh. Trận luyện tập có ưu tiên thấp nhất. Đội không thấy source, build log, stderr hay chi tiết lỗi của bot mẫu, nhưng vẫn thấy đầy đủ chẩn đoán của bot mình.

`examples/reference-bots/bot.cpp` và `bot.py` có các chế độ alpha-beta, phòng thủ, ngẫu nhiên, MCTS/UCT; sửa hằng `MODE` rồi tải lên làm bot mẫu. Các mức alpha-beta chỉnh bằng `MAX_DEPTH`/`THINK_MS`. Đây là bot tham khảo, mức Dễ/Khó do BTC tự đặt.

Tham khảo thuật toán: [Kocsis & Szepesvári, Bandit Based Monte-Carlo Planning](https://www.cs.utexas.edu/~shivaram/readings/b2hd-KocsisSzepesvari2006.html) (UCT); alpha-beta với bảng chuyển vị theo giáo trình tìm kiếm trò chơi đối kháng.
