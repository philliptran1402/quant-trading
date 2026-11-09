# Chiến lược quyền chọn và tính volatility

Hai script xử lý options trong repo giải quyết hai bài toán khác nhau: chọn thời điểm mua straddle trong dữ liệu lịch sử và tính chỉ số volatility từ dải quyền chọn.

## 1. Long Straddle — `Options Straddle backtest.py`

### Mục tiêu và cấu trúc dữ liệu

Long straddle mua một call và một put có cùng strike và ngày đáo hạn. Người mua trả premium của cả hai chân; kết quả tốt khi giá tài sản cơ sở đi đủ xa khỏi strike để bù tổng premium. File dùng `data/stoxx50.xlsx`, trong đó `spot` là giá chỉ số và các sheet như `aug` chứa chuỗi quyền chọn. Cả bảng được index theo ngày.

### Hàm và quy trình

1. `find_strike_price(df)` quét tên cột, lấy nhóm 4 chữ số làm strike và giữ các strike xuất hiện nhiều lần, giả định tương ứng call/put.
2. `straddle(options, spot, contractsize, strikeprice)` chọn các cột options có chứa strike, ghép với spot theo index, đổi nhãn cột thành call/put/spot. Cột spot được nhân `contractsize` để dùng cùng đơn vị với strike.
3. `signal_generation(df, threshold)` bật tín hiệu khi chênh lệch tuyệt đối giữa giá call và put nhỏ hơn ngưỡng. Đây là quy tắc chọn cặp có premium gần nhau; không phải tín hiệu dự báo hướng thị trường.
4. `plot()` dùng ngày đầu tiên có tín hiệu làm giá mua hai options, tính payoff tại nhiều mức spot cuối kỳ và vẽ vùng lãi/lỗ. `main()` nạp Excel, chạy qua các strike và kỳ hạn.

### Diễn giải payoff

Tại đáo hạn, payoff long straddle xấp xỉ `abs(spot - strike) - call_premium - put_premium` (được nhân contract size theo quy ước trong code). Điểm hòa vốn nằm hai phía strike, cách strike bằng tổng premium trên mỗi đơn vị hợp đồng.

### Giới hạn cần kiểm tra

- Code chỉ dựa vào giá settlement lịch sử, không có bid/ask; premium có thể không phải giá có thể khớp.
- Quy tắc chọn `strike` dựa vào pattern tên cột và `re.search('\d{4}')`, dễ nhầm nếu schema/strike thay đổi hoặc các mã có số khác định dạng.
- Ngày hai chuỗi không giao nhau sẽ bị mất khi merge theo index.
- Payoff minh họa dùng premium tại lần tín hiệu đầu và spot cuối chuỗi; đây không phải backtest mark-to-market có đóng/mở vị thế đầy đủ.
- Cần xác minh đơn vị `contractsize`, strike và spot. Nhân spot nhưng không nhân strike/premium nhất quán có thể làm payoff sai đơn vị; xem đúng cách code xử lý trước khi dùng số kết quả.

## 2. VIX-style calculator — `VIX Calculator.py`

### Luồng tính

Script triển khai biến thể phương pháp variance từ options trên hai kỳ hạn. Đầu vào là chuỗi option Henry Hub, đường cong lãi suất Treasury và lịch nghỉ CME.

1. `cmt_rate_fill_date()` tạo lịch ngày liên tục và lấp lãi suất còn thiếu.
2. `get_settlement_day()` / `get_time_to_expiration()` quy đổi thời gian còn lại tới settlement theo phút, có tính giờ đáo hạn và ngày nghỉ.
3. `get_forward_strike()` chọn strike có chênh call-put nhỏ nhất, suy forward theo lãi suất và chọn strike lớn nhất không vượt forward.
4. `get_options_call_inclusion()` / `get_options_put_inclusion()` lấy options OTM phù hợp; vì dữ liệu không có bid/ask, prior settlement thay thế quote. Chuỗi options zero bị lọc khi gặp các strike zero liên tiếp.
5. `compute_sigma()` cộng đóng góp option theo khoảng cách strike, chiết khấu lãi suất, rồi trừ hiệu chỉnh forward/strike.
6. `compute_vix()` nội suy variance hai kỳ hạn về `num_of_mins_timeframe`, năm hóa bằng phút trong năm và nhân 100.
7. `vix_calculator()` ghép toàn bộ cho từng ngày giao dịch; `main()` đọc ba CSV trong `data/`.

### Cách hiểu kết quả

Đây là volatility suy ra từ giá quyền chọn, không phải độ lệch chuẩn thực nghiệm của giá spot. Chỉ số phụ thuộc mạnh vào chất lượng quote, strike ladder, lãi suất, thời điểm settlement và chính xác lịch nghỉ.

### Rủi ro triển khai

- Dữ liệu dùng `priorSettle` thay bid/ask midpoint, nên có thể stale hoặc không đại diện giá hiện tại.
- Hàm chọn options xử lý index vị trí và strike liền kề; thiếu strike, dữ liệu rỗng hoặc zero bids không liên tiếp có thể tạo biên chọn khác dự định.
- Phải so sánh đơn vị thời gian của `time_to_expiration`, số phút mục tiêu và số phút/năm. Sai đơn vị làm VIX bị scale sai.
- Tên VIX trong tài liệu chỉ mô tả phương pháp; dữ liệu là Henry Hub options nên chỉ số được tính không mặc nhiên là Cboe VIX của S&P 500.
- Lịch nghỉ và định nghĩa phiên giao dịch phải khớp với hợp đồng đầu vào.
