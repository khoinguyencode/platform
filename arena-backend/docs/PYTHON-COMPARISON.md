# Đo Python / C++ trên OTTv2

Image: `sha256:fa371d5d81f7cd07f2e55a441e6086a8ca6fddb5201b9b5e8398319835ffcbbc`. Sáu thế từ suite OTTv2 công khai, lặp 12 lần (72 mẫu/runtime); cùng thuật toán/evaluation/thứ tự nước; fixed-depth C++/CPython kiểm tra bằng nhau cả nước, số nút và độ sâu.

| Ngôn ngữ | Nút/s median (100 ms) | Độ sâu median | Search ms median |
|---|---:|---:|---:|
| C++17 | 705017 | 4 | 100.03 |
| PyPy (Python mặc định) | 224362 | 4 | 100.07 |
| CPython (đối chứng) | 35138 | 3 | 100.25 |

| Runtime | Startup handshake median ms | Peak RAM container MiB |
|---|---:|---:|
| C++17 | 1.43 | 10.52 |
| PyPy (Python mặc định) | 86.83 | 78.93 |
| CPython (đối chứng) | 32.99 | 14.09 |

12 ván C++/CPython, đổi màu trên 6 thế mở đầu: C++ thắng 7, CPython thắng 2, hòa 3. Giới hạn thí nghiệm 128 ply, lặp 3 lần hòa. Đây là mẫu nhỏ, không đại diện mọi bot; chưa đo tỉ lệ thắng C++/PyPy. JSON từng ván nằm trong test-results/language-benchmark.json.

| Build cùng starter | Median ms |
|---|---:|
| without_pch | 5693.1 |
| with_pch | 4374.8 |

PCH đối chứng: cùng image và source; chỉ thêm newline vào bản json.hpp để buộc nhánh không dùng PCH. Không gọi đây là đo nguyên commit cũ.

## Chính sách thời gian

**Cùng quỹ giờ cho mọi ngôn ngữ; không nhân hệ số Python.** Bot Python chạy bằng PyPy 7.3.23 / Python 3.11; trong phép đo này PyPy tìm được khoảng 6.4 lần số nút/giây của CPython. C/C++ vẫn thường nhanh hơn Python. Xem [PYPY-VALIDATION.md](PYPY-VALIDATION.md).

Các lựa chọn đã cân nhắc:

1. Cùng thời gian: đơn giản, đo đúng bot nhanh trên máy thi; Python thường tìm nông hơn.
2. Nhân thời gian Python: tăng thời lượng giải; hệ số phụ thuộc thuật toán/thế cờ, không suy ra công bằng từ một median.
3. PyPy hoặc Codon: phải thêm image/toolchain riêng và kiểm sandbox, bộ thư viện/giao thức; PyPy là JIT, Codon có thể biên dịch một tập con Python. Không bảo đảm bot CPython tùy ý sẽ chạy được.

Phép đo dùng sandbox thật. Nhãn python3 trong JSON đo chỉ CPython đối chứng; API python3 trong sản phẩm chạy PyPy. Không suy hệ số công bằng chung từ một thuật toán hay median.