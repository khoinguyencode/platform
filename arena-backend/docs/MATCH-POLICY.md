# Quy tắc xử lý kết quả OTTv2 · v2

Áp dụng từ ngày 2026-10-11. Đây là quy tắc vận hành bổ sung cho văn bản luật gốc, không thay đổi nước đi hoặc khắc chế của OTTv2.

## Phạm vi và phiên bản

Giải mới ghim `config.adjudication` với ID `ottv2-match-v2`; job chính thức và report lưu cùng giá trị. Luyện tập mới cũng dùng v2. Benchmark vẫn kiểm 72 mẫu hợp lệ, không tự chứng nhận mọi trạng thái hoặc hành vi startup ở cả hai màu.

Giải/job cũ thiếu trường này được hiểu là `ottv2-match-v1`: startup lỗi hoặc chạm guard vẫn không có kết quả và cần BTC xử lý. Không tự nâng thể lệ giải đã tồn tại, không tính lại lịch sử. Muốn dùng v2 hãy tạo giải mới; benchmark cũ vẫn dùng được nếu source, binary, engine, image, SDK và timing profile khớp.

## Hòa

- Cùng toàn bộ bàn cờ **và bên tới lượt** xuất hiện lần thứ ba: `{"winner":null,"reason":"threefold_repetition"}`. Tính cả vị trí ban đầu; số ply và timestamp không thuộc khóa thế cờ.
- Sau 512 ply mà chưa có kết quả: `{"winner":null,"reason":"ply_limit"}`.
- Đạt 20 phút thời gian trận: `{"winner":null,"reason":"wall_limit"}`. Kiểm tra giữa các lượt; không cắt ngang một lượt hợp lệ đang dùng ngân sách. Build/startup không nằm trong đồng hồ 20 phút này.
- Thắng theo luật trên nước vừa đi được xét trước lặp thế và giới hạn ply. Invalid action hoặc timeout vẫn xử thua; không được biến thành hòa.
- Shutdown hoặc hỏng worker/controller không phải hòa. Attempt không có kết quả chuyển `attention`; server không tiếp tục clock cũ.

Mỗi đội tăng `played` và `draws` một, không tăng `wins` hoặc `losses`, và nhận 1 điểm. Bảng xếp hạng dùng tổng điểm: thắng 3đ, hòa 1đ, thua 0đ. Bằng tổng điểm thì đồng hạng, không dùng số thắng làm hệ số phụ.

## Lỗi startup và hạ tầng

| Trường hợp | Kết quả |
|---|---|
| Supervisor xác nhận bot quá 2 giây init→ready, crash, sai protocol/stale ready hoặc output quá giới hạn | Kiểm tra startup cả hai bên; một bên lỗi thì đối thủ thắng với `startup_failure` |
| Cả hai bot lỗi startup | `winner:null`, `reason:double_startup_failure`; mỗi đội thêm một trận đã chơi và một trận thua, không tính hòa |
| Docker/container/transport/supervisor lỗi, không có phản hồi trusted | Job lỗi, không trao thắng/thua; fixture `attention` |
| Binary/manifest đã chốt bị mất hoặc hash sai | `artifact_missing` / `artifact_corrupt`; BTC khôi phục artifact đã kiểm chứng rồi retry, không tự build thay binary |
| Engine/SDK/profile/policy không khớp cấu hình ghim | `provenance_mismatch`; không tính điểm |
| Lỗi compile hoặc benchmark startup | Benchmark không đạt; không có trận đấu để tính thắng/thua |

Nếu một bot đã lỗi startup nhưng chưa kiểm tra được bên kia vì lỗi hạ tầng, **không** trao chiến thắng. Report admin giữ `startup_faults` theo phe; log riêng không phát qua SSE công khai. Lỗi trong lượt có thêm stage, turn ID, state version và action đã nhận trong report của người có quyền.

Deadline trong lượt giữ nguyên: worker monotonic clock, nhận đủ LF trước deadline cứng; server tuyên bố timeout thay đối thủ sau tối thiểu 3 giây. Khoảng chờ này không áp dụng cho handshake startup.

## Hàng đợi

N worker, mỗi trận ghim một logical CPU; mặc định số CPU khả dụng trừ 2, cấu hình ARENA_WORKERS. Queue lưu SQLite, không cap 16 toàn hệ thống. Ưu tiên official → benchmark → practice; cùng nhóm FIFO. Mỗi loại/đội giữ bài chờ mới nhất; tối đa 4 job đang chạy và 200 job lịch sử/đội. API trả queue_position. Scheduler giữ tối đa N official outstanding.

Job đang chạy được hoàn tất, không bị cắt ngang để nhường giải. Khi giải đang chạy liên tục, luyện tập có thể phải chờ hết giải. Pause chỉ ngừng dispatch fixture mới; attempt đã nhận vẫn hoàn tất. Chưa bảo đảm fairness giữa nhiều giải. Worker thử lại lỗi hạ tầng một lần, không thử lại lỗi do bot. Restart giữ queued, abort job đã claim/chạy dở và không trao điểm; fixture lỗi cần BTC xử lý.

## Kiểm tra

`make test` kiểm lặp thế/side-to-move, thắng ở ply cuối, giới hạn thời gian, shutdown, lỗi startup cả hai bên, lỗi hạ tầng không tính điểm, binary mất, legacy policy, bảng xếp hạng và reserved queue. `tests/competition_integration.py` dùng HTTP, ZIP C++ và Docker thật trên DB tạm để chạy cả hai lượt thường, hòa lặp thế, một bot timeout startup và hai bot crash startup.
