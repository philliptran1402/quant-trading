# Phân tích Smart Farmers project

## Mục tiêu và chuỗi xử lý

Smart Farmers dùng dữ liệu nông nghiệp và kinh tế để chọn cây trồng, ước lượng quan hệ giá–sản lượng–GDP/dân số, rồi tối ưu phân bổ sản xuất/đất theo năm. Luồng dự kiến:

`FAO raw data → cleanse data → country/crop selection → demand estimation → constrained production forecast → price/production charts`

CSV kết quả đã có trong `Smart Farmers project/data/` gồm bảng dân số, GDP, đất, giá, sản lượng, mapping và các bảng tổng hợp. Một số script ingest cần các bộ FAO raw CSV không được đưa vào repo.

## 1. Làm sạch dữ liệu — `cleanse data.py`

`prepare(target_land, target_prod, target_prix)` chọn và đổi cấu trúc bảng: đất; giá theo crop/năm; diện tích thu hoạch; sản lượng. Script lấy Malaysia và một nhóm quốc gia so sánh, bổ sung giá cho một số nhóm cây bằng dữ liệu thay thế, ghép `mapping.csv`, lọc cây có đủ chuỗi sản lượng/diện tích/giá, rồi xuất các CSV gọn.

**Giả định quan trọng:** quy đổi đơn vị, mapping crop và cách bù thiếu ảnh hưởng mọi bước sau. Phần bù đất năm 2018 và thay giá cereals/oilseeds là quy tắc nghiệp vụ được mã hóa thủ công, cần xem lại khi cập nhật dữ liệu.

## 2. Kiểm tra tính nhất quán — `check consistency.py`

Đọc raw production, prices, land use và mapping; lọc năm/quốc gia; so khớp danh sách crop và các trường. Mục tiêu là phát hiện chênh lệch coverage trước ETL. Script cần raw FAO CSV ngoài các file hiện có trong repo.

## 3. Chọn quốc gia/cây — `country selection.py`

Đọc raw FAO production, price và trade; lọc các năm và quốc gia, so sánh độ phủ cây/chuỗi dữ liệu, rồi tạo tập dữ liệu phục vụ mô hình. Đây là bước chuẩn bị mẫu; thay đổi tập quốc gia có thể làm thay đổi hệ số hồi quy và tối ưu.

## 4. Ước lượng cầu — `estimate demand.py`

- `create_xy()` đặt giá crop làm y; các biến giải thích gồm GDP Malaysia, dân số và sản lượng crop. Hệ số sản lượng được đổi dấu trong thiết kế biến.
- `lin_reg()` chạy OLS bằng statsmodels cho từng cây và có thể vẽ giá fitted/actual.
- `constrained_ols()` giải quadratic program bằng CVXOPT để áp ràng buộc lên hệ số, thay vì OLS không ràng buộc.
- `get_params()` lặp qua cây, lưu hệ số; các đoạn cuối dùng dữ liệu futures/forecast để nối chuỗi giá dầu cọ và vẽ đối chiếu.

Code phụ thuộc mạnh vào thứ tự/căn chỉnh năm giữa GDP, dân số, sản lượng và giá. Hệ số hồi quy không tự chứng minh quan hệ nhân quả; cần kiểm tra đa cộng tuyến, đơn vị, stationarity và validation ngoài mẫu.

## 5. Dự báo và phân bổ — `forecast.py`

### Chuẩn bị và tối ưu

`prepare(grand)` chia bảng theo năm vào dictionary toàn cục `D`. `get_ans()` gọi `cvxopt.solvers.qp`. Bản tuyến tính được lưu dưới `get_production_linear()`; `get_production()` dùng phiên bản quadratic với hệ số giá/elasticity (`alpha`, `beta`, `gamma`) và các ràng buộc.

Mục tiêu tối ưu được tạo từ giá quá khứ, sản lượng, thay đổi dân số/GDP và guess ban đầu. Ràng buột đất tổng thể dùng yield × production bằng lượng đất khả dụng; ràng buộc bất đẳng thức mô tả giới hạn sản lượng/diện tích và chu kỳ cây. `costfunction()` so sánh sản lượng dự báo với thực tế theo sai số tương đối bình phương; `ls_estimate()` tìm bộ tham số ban đầu bằng Nelder–Mead; `find_init()` thử nhiều guess ngẫu nhiên.

### Dự báo chuỗi

Các đoạn cuối bổ sung GDP/đất dự phóng, tạo sản lượng qua từng năm, rồi tính lại giá bằng `compute_price()`. Kết quả được biểu diễn qua chart tổng sản lượng, giá và crop.

### Rủi ro mô hình

- Solver QP cần hệ số/ma trận đúng chiều, convexity và ràng buộc khả thi. Không phải mọi guess đều đảm bảo nghiệm.
- Tối ưu theo lỗi tương đối có thể quá nhạy với giá trị thực tế gần 0.
- Guess ngẫu nhiên/Nelder–Mead phụ thuộc seed và điểm bắt đầu; nên lưu seed, nghiệm và trạng thái solver.
- Mở rộng GDP/đất bằng các giả định ngoại suy (ví dụ tăng đất 1% mỗi năm) làm kết quả phụ thuộc giả định.
- Biến toàn cục `D`, `currentyear`, `malay_*`, `upperbound` làm hàm khó gọi độc lập; notebook thực thi nhiều lệnh ngay khi nạp module.
- Một số dữ liệu raw FAO không có trong repo nên ETL từ đầu chưa thể chạy chỉ với checkout này.

## Thứ tự đọc và chạy

Đọc `README.md` dự án, xác nhận schema và đơn vị các CSV trong `data/`; kiểm tra consistency; chạy cleanse nếu có raw FAO inputs; sau đó chạy demand estimation và forecast. Không dùng kết quả tối ưu trước khi xác nhận đơn vị đất/sản lượng, mã crop, ràng buộc và validation theo năm.
