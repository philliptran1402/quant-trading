# Phân tích Oil Money project

## Mục tiêu

Oil Money khảo sát quan hệ giữa giá dầu và tỷ giá của một số nước sản xuất dầu, sau đó xây dựng chiến lược rolling regression cho tiền tệ. README phân tích CAD, COP, NOK và RUB; chiến lược chính được tách trong `Oil Money Trading backtest.py`.

Ý tưởng ban đầu: hồi quy tỷ giá petrocurrency lên dầu và một số yếu tố kinh tế/tiền tệ để ước lượng mức “cân bằng”; độ lệch giữa giá thực tế và forecast đóng vai trò spread. Dự án thử cả mean reversion và tiếp diễn xu hướng khi mô hình/quan hệ cũ không còn hợp lệ.

## Chiến lược chung trong `Oil Money Trading backtest.py`

### Dữ liệu và đầu vào

Chuỗi cần có ngày làm index cùng ít nhất hai cột giải thích/được dự báo (ví dụ `brent`, `nok`). Các bộ dữ liệu FX/oil theo quốc gia nằm trong `Oil Money project/data/`. Giá phải cùng đơn vị tiền tệ và cùng lịch ngày trước khi ước lượng.

### Tạo forecast và tín hiệu

- `oil_money(dataset)` khởi tạo các cột `signals`, dải ±1/±2 sigma và `forecast`.
- `signal_generation(dataset, x, y, method, holding_threshold=10, stop=0.5, rsquared_threshold=0.7, train_len=50)` fit hồi quy trên cửa sổ lăn. Chỉ coi mô hình đủ tin cậy khi R² vượt ngưỡng; tính residual và độ lệch chuẩn; dùng dải quanh dự báo để phát tín hiệu.
- Trong logic chiến lược, lệch khỏi dải được xem là cơ hội giao dịch; nếu lệch vượt hai sigma, code coi quan hệ/mô hình có thể đổi regime và có thể theo chiều chuyển động mới thay vì tiếp tục mean reversion.
- Khi có vị thế, đếm số kỳ nắm giữ; thoát nếu quá thời hạn hoặc giá cách điểm vào vượt ngưỡng `stop`. Sau khi thoát, cửa sổ được ước lượng lại.
- `portfolio()` chuyển lệnh thành lượng nắm giữ/cash; `profit()` và `plot()` tổng hợp lợi nhuận, giá trị tài khoản và vị trí giao dịch.

### Tham số

`train_len=50` là lookback hồi quy; `rsquared_threshold=0.7` là điều kiện chấp nhận mô hình; `holding_threshold=10` là số phiên tối đa; `stop=0.5` là mức giá tuyệt đối theo đơn vị cột y. Tham số thay đổi theo đồng tiền (COP dùng mức stop rất nhỏ trong script phân tích), nên không thể bê nguyên giữa các cặp.

## Các phân tích theo quốc gia

- **NOK — `Oil Money NOK.py`:** Brent quy đổi JPY và NOKJPY; đồ thị tương quan, lãi suất/GDP, hồi quy, chẩn đoán quan hệ và kiểm tra tham số giữ lệnh/stop. Cuối file gọi backtest chung và quét nhiều holding/stop. Đây là notebook chạy tuần tự ở cấp module.
- **COP — `Oil Money COP.py`:** Vasconia crude và COPAUD; dùng hồi quy và đánh giá dự báo, sau đó chạy chiến lược chung. Có phân tích momentum/clustering trong các đoạn khác.
- **CAD — `Oil Money CAD.py`:** WCS crude so với CAD/AUD và các nhóm/clustering; mục tiêu là kiểm tra liệu CAD có thể được coi là petrocurrency trong dữ liệu này.
- **RUB — `Oil Money RUB.py`:** Urals crude/RUB-AUD; so sánh mô hình theo giai đoạn và thể hiện thay đổi quan hệ qua thời gian.

README diễn giải Norway/Colombia có alpha cao hơn trong thử nghiệm, trong khi CAD không cho kết quả petrocurrency mong đợi. Đây là kết luận của bộ dữ liệu lịch sử cụ thể, không phải bảo đảm edge còn tồn tại.

## Visualization phụ trợ

- `oil production cost curve.py` đọc CSV cùng tên trong thư mục, lập đường cong chi phí hoạt động/vốn theo sản lượng và vẽ mốc percentile tổng chi phí.
- `oil production choropleth.py` ghép sản lượng với `worldmapshape.json` để lưu bản đồ HTML.

## Hạn chế và rủi ro

- Regression R² cao không chứng minh quan hệ nhân quả hay tính ổn định ngoài mẫu.
- Rolling model và ngưỡng được chọn sau khi quan sát kết quả có nguy cơ data snooping; cần walk-forward, holdout cuối cùng và kiểm tra parameter sensitivity.
- Cần đo spread, slippage, phí FX, thanh khoản và chi phí chuyển đổi đồng tiền; README gốc giả định frictionless.
- Dùng giá dầu và tỷ giá ở cùng ngày có thể tạo look-ahead nếu một thị trường đóng cửa trước thị trường kia.
- Nhiều script còn `os.chdir` ở lúc import và chạy cell ở cấp module; import như thư viện có tác dụng phụ.
- Các tệp NOK trước đây có lỗi cú pháp/import đã được sửa trong lượt review; tài liệu này mô tả code hiện tại.
