# Phân tích Ore Money project

## Phạm vi code hiện có

README mô tả ý tưởng nghiên cứu quan hệ quặng sắt–FX của nước xuất khẩu, lấy cảm hứng từ Oil Money. Mã Python hiện tại chỉ trực quan hóa sản lượng theo quốc gia; chưa có mô hình tỷ giá, backtest hay ước lượng alpha.

## Bubble map — `iron ore production/iron ore production bubble map.py`

### Dữ liệu

Script đọc `iron ore production bubble map.csv` cạnh file Python. Bảng cung cấp tên quốc gia, sản lượng, kinh độ và vĩ độ (tọa độ thường là thủ đô, không phải trọng tâm quốc gia).

### Các bước vẽ

1. Tạo figure Matplotlib kích thước lớn.
2. Dùng Basemap vẽ biên giới/quốc gia và nền.
3. Đổi tọa độ địa lý sang tọa độ projection bằng `m(longitude, latitude)`.
4. Tính kích thước bubble tỷ lệ với sản lượng; tô màu theo cùng đại lượng bằng colormap.
5. Vẽ scatter và chú giải để so sánh quy mô sản xuất trực quan.

### Cách hiểu và giới hạn

Bubble lớn biểu thị sản lượng lớn hơn; màu và kích cỡ cùng mã hóa sản lượng nên cần xem chú giải để không diễn giải sai. Vị trí dùng tọa độ thủ đô và có điều chỉnh thủ công theo README, vì vậy không dùng như bản đồ địa lý chính xác. CSV là snapshot 2015 theo mô tả dự án; không thể suy luận thay đổi hiện tại.

## Bước tiếp theo nếu mở rộng thành chiến lược

Để biến ý tưởng Ore Money thành nghiên cứu định lượng, cần dữ liệu theo thời gian về giá quặng (benchmark, chất lượng/hàm lượng, cước vận tải), tỷ giá, chính sách/kiểm soát vốn, lịch ngày và giá có thể giao dịch. Sau đó phải xác định rõ giả thuyết nhân quả, thời điểm công bố dữ liệu, mô hình train/test và chi phí giao dịch. Những phần này chưa được cài trong repository hiện tại.
