# Phân tích chiến lược kỹ thuật và backtest

Tài liệu này phân tích các script chiến lược độc lập ở thư mục gốc. Những file này là ví dụ nghiên cứu, không phải một engine backtest dùng chung: cách tạo tín hiệu, quản lý vốn và đóng vị thế khác nhau giữa các script.

## Mẫu luồng xử lý chung

Phần lớn script thực hiện: nạp OHLC/giá → tạo chỉ báo hoặc đặc trưng → tạo cột `positions` (trạng thái nắm giữ) hoặc `signals` (lệnh phát sinh) → vẽ tín hiệu. Một số có mô phỏng danh mục, số khác chỉ vẽ lệnh. Các con số không thể so sánh trực tiếp nếu vốn, số lượng cổ phiếu, thời điểm khớp và chi phí giao dịch khác nhau.

README gốc nói các backtest bỏ qua phí, trượt giá và thanh khoản. Đây là giả định lớn: đặc biệt với chiến lược intraday, kết quả lý tưởng có thể lệch nhiều so với khớp lệnh thực tế.

## 1. MACD — `MACD Oscillator backtest.py`

**Ý tưởng.** MACD ở đây là so sánh hai SMA của `Close`. `ma1` là cửa sổ ngắn, `ma2` là cửa sổ dài. Chênh lệch hai đường được lưu ở `oscillator`.

**Luồng code.** `main()` hỏi chu kỳ MA, thời gian, ticker và số hàng cần cắt bớt; lấy dữ liệu Yahoo Finance; `macd()` thêm hai cột rolling mean; `signal_generation()` xác lập vị thế 1 khi MA ngắn ≥ MA dài (sau đoạn khởi động `ma1`), 0 khi thấp hơn; sai phân `positions` thành lệnh `signals`. Do đó script này chỉ long hoặc đứng ngoài, không mở vị thế short. `plot()` vẽ giá, điểm đổi trạng thái, hai MA và oscillator.

**Diễn giải.** `signals=1` là mở long, `signals=-1` là đóng long; giá trị 0 nghĩa là không có thay đổi trạng thái. Đây là chiến lược bám xu hướng trễ: tín hiệu xác nhận sau khi MA đã giao nhau.

**Tham số và hạn chế.** `ma1`/`ma2` do người chạy nhập, script không tự tối ưu. Dữ liệu được tải online qua `fix_yahoo_finance` (package cũ). Tín hiệu tính từ giá đóng cửa nhưng code không mô phỏng rõ khớp ở giá nào sau khi tín hiệu được biết, vì thế cần tránh dùng PnL biểu đồ như kết quả khả thi.

## 2. Awesome Oscillator — `Awesome Oscillator backtest.py`

**Ý tưởng.** AO là SMA 5 trừ SMA 34 của trung điểm `(High+Low)/2`. File đồng thời tính nhánh MACD EWMA để so sánh.

**Logic.** `awesome_ma()` thêm MA ngắn/dài và oscillator. `awesome_signal_generation()` xét mô hình saucer 3 nến khi oscillator ở cùng phía zero, đồng thời tạo tín hiệu theo quan hệ MA ngắn/dài. `cumsum` được dùng để tránh cộng dồn lệnh vượt trạng thái nắm giữ mà tác giả mong muốn. Nhánh MACD trong `signal_generation()` tạo vị thế từ EMA và tính sai phân lệnh. `portfolio()` đặt vốn ban đầu 5.000 và số lượng 100, sau đó so sánh giá trị danh mục AO/MACD; `stats()` tính các số liệu tổng hợp.

**Cách hiểu vị thế.** Tín hiệu AO có thể là lệnh phát sinh; `cumsum` chuyển các lệnh này thành lượng đang giữ. Cần đọc cột `awesome signals` cùng `cumsum`, không hiểu mọi số 1/−1 là trạng thái vị thế nếu đang xem plot riêng.

**Rủi ro.** Quy tắc saucer được mã hóa trực tiếp trên Open/Close và oscillator; tên nến màu xanh/đỏ có thể khác quy ước của dữ liệu. So sánh AO với MACD chỉ công bằng khi dùng cùng ticker, thời gian, số lượng, giá khớp và chi phí.

## 3. Heikin-Ashi — `Heikin-Ashi backtest.py`

**Biến đổi nến.** `heikin_ashi()` tính HA Close bằng trung bình OHLC; HA Open bắt đầu từ Open ban đầu, sau đó bằng trung bình HA Open/Close kỳ trước. HA High/Low là cực trị giữa HA Open, HA Close và High/Low gốc.

**Tín hiệu.** `signal_generation()` tạo tín hiệu long khi nến HA giảm có Open bằng High và thân lớn hơn thân nến giảm trước đó; đóng các lệnh long khi xuất hiện nến tăng thỏa điều kiện Open bằng Low và nến trước cũng tăng. `stls` giới hạn số lệnh long tích lũy theo logic `cumsum`.

**Danh mục.** `portfolio()` mặc định vốn 10.000, 100 cổ phiếu mỗi đơn vị tín hiệu; tính holdings, cash, total asset và return. `stats()` trình bày giao dịch, thời gian, lợi nhuận và thước đo như drawdown, Sortino/Omega.

**Hạn chế.** Giá HA được làm mượt nhưng danh mục định giá theo Close gốc. Điều kiện nến rất cụ thể, phụ thuộc thứ tự hàng và chất lượng OHLC. Mô phỏng thêm vị thế phải được kiểm tra kỹ vì các tín hiệu dồn lệnh không đồng nghĩa với một giao dịch đơn.

## 4. Parabolic SAR — `Parabolic SAR backtest.py`

**Ý tưởng và tính toán.** `parabolic_sar()` giữ trạng thái trend, SAR, điểm cực trị EP và hệ số AF. AF khởi đầu 0,02, tăng 0,02 mỗi khi lập cực trị mới, tối đa 0,2. SAR tiến về EP; khi giá vượt điểm đảo chiều, xu hướng đổi dấu và SAR được khởi tạo lại.

**Tín hiệu.** `signal_generation()` đặt vị thế long khi `real sar < Close`, ngược lại đứng ngoài; sai phân vị thế tạo tín hiệu. Script không mở short. `main()` tải ticker EA trong khoảng 2016–2018 và cắt dữ liệu để vẽ.

**Hạn chế.** Công thức tự cài đặt cần đối chiếu với chuẩn SAR, đặc biệt xử lý đảo chiều và giới hạn SAR theo hai nến trước. Hàm dùng truy cập theo chỉ số nguyên; nên truyền DataFrame có index phù hợp hoặc chuẩn hóa index trước khi tái sử dụng.

## 5. RSI và nhận dạng mẫu — `RSI Pattern Recognition backtest.py`

**RSI.** `smma()` cập nhật trung bình làm mượt; `rsi()` lấy thay đổi Close, tách mức tăng/giảm, tính RS và RSI. `signal_generation()` long khi RSI < 30, đứng ngoài vùng giữa và đặt trạng thái -1 khi RSI > 70; sai phân thành tín hiệu. Hàm trả dữ liệu từ sau giai đoạn khởi động.

**Nhận dạng vai–đầu–vai trên giá.** `pattern_recognition()` tìm các điểm đáy gần nhau và một đỉnh đầu/đỉnh vai trong cửa sổ 25 kỳ, dùng `delta=0.2`, hệ số đầu/vai 1,1. Khi tìm được mẫu, phát tín hiệu -1; thoát bằng tín hiệu ngược khi RSI tăng hơn 4 điểm từ lúc vào hoặc quá 5 kỳ. `coordinates` lưu chỉ số các nút để vẽ.

**Giới hạn.** Phần tên file nói RSI pattern nhưng mẫu hình chính được tìm trên Close; RSI chủ yếu tham gia quy tắc thoát. Các ngưỡng 30/70, 0,2 đơn vị giá và 25 phiên không chuẩn hóa theo biến động hoặc mức giá nên khó chuyển ticker. Rà soát gần đây sửa việc gán RSI bằng chained indexing; vẫn cần kiểm tra kết quả mẫu trên chuỗi dữ liệu đại diện.

## 6. Shooting Star — `Shooting Star backtest.py`

**Điều kiện mẫu.** `shooting_star()` kiểm tra tám điều kiện: nến đỏ; bóng dưới nhỏ; thân nhỏ so với thân trung bình; bóng trên dài; hai phiên trước thể hiện xu hướng tăng; và nến kế tiếp có High/Close không vượt nến mẫu. Điều kiện tương lai nghĩa là mẫu chỉ được xác nhận sau khi nến kế tiếp đã đóng.

**Giao dịch.** `signal_generation()` biến mẫu thành tín hiệu short (-1); đi tiếp tối đa 7 phiên hoặc thoát khi giá Close biến động hơn 5% so với điểm vào, phát tín hiệu ngược (+1). `positions` là tổng tích lũy của tín hiệu. Dữ liệu chính được tải cho VOD.L, 2000–2021.

**Hạn chế quan trọng.** Vì dùng `shift(-1)`, tín hiệu được đặt ở hàng nến shooting star sau khi biết nến kế tiếp; nếu backtest khớp ở giá Close của hàng đó thì cần xác nhận timing để tránh look-ahead. Vòng đóng lệnh tăng chỉ số có thể vượt cuối DataFrame nếu tín hiệu cuối chuỗi không có đủ phiên tiếp theo. Tham số stop dùng Close, không phải High/Low trong phiên.

## 7. Bollinger W / đáy đôi — `Bollinger Bands Pattern Recognition backtest.py`

**Chỉ báo.** `bollinger_bands()` dùng cửa sổ 20 kỳ, độ lệch chuẩn mẫu và dải ±2 độ lệch chuẩn quanh SMA giá.

**Quy tắc W.** `signal_generation()` tìm trong 75 phiên khi chưa có vị thế: giá vượt dải trên; trước đó có đáy gần dải dưới; giá hồi lên vùng giữa; đáy sau gần dải dưới và cao hơn đáy đầu. Khi điều kiện được xác nhận, phát tín hiệu long và lưu tọa độ nút mẫu. Khi đang giữ vị thế, code thoát lúc độ lệch chuẩn giảm dưới `beta`.

**Tham số.** `alpha=0.0001` điều khiển độ gần mức giá/dải; `beta=0.0001` điều khiển co hẹp biến động. Hai ngưỡng tuyệt đối này được áp trên giá và độ lệch chuẩn, nên kết quả phụ thuộc đơn vị giá và gần như không portable giữa các cặp/ticker. Script đọc `gbpusd.csv` và vẽ dữ liệu ngày.

**Giới hạn.** Tìm mẫu bằng các vòng lặp lồng nhau và chỉ số hàng; chi phí tăng theo cửa sổ. Hàm `plot()` dựa vào có tín hiệu và tọa độ mẫu; trường hợp không có tín hiệu có thể gây lỗi khi lấy phần tử đầu tiên.

## 8. London Breakout — `London Breakout backtest.py`

**Luồng phiên.** Dữ liệu dự kiến là nến FX 1 phút theo giờ Eastern Standard Time. Script gom các mức giá ở 02:00 để lập biên Tokyo; lúc 03:00, lấy max/min làm ngưỡng. Trong 30 phút đầu London, vượt biên trên tạo long, thủng biên dưới tạo short. Biên độ vượt ngưỡng quá `risky_stop=0.01` bị bỏ qua; đây là đơn vị giá của cặp, không phải 1% tương đối.

**Quản lý vị thế.** `cumsum` tránh mở lặp cùng hướng. Lệnh thoát khi giá đi ±0,005 so với giá vào (một nửa `risky_stop`) hoặc tại 12:00 EST, theo giả định London đóng lúc đó. Các ngưỡng được giữ trong cột `upper`/`lower`.

**Hạn chế.** Quy tắc giờ cố định không xử lý daylight saving time hoặc ngày nghỉ. Danh sách Tokyo chỉ tích lũy những hàng có giờ đúng 02; nếu thiếu phút hoặc nguồn dùng timezone khác, biên bị sai/không có. Chỉ báo giá vào được giả định từ giá hàng tín hiệu, không mô hình bid/ask và spread.

## 9. Dual Thrust — `Dual Thrust backtest.py`

**Đặc trưng.** `min2day()` gom dữ liệu 1 phút thành OHLC theo ngày/tháng và tính range theo các phiên lùi; tham số `rg` là số phiên nhìn lại. `signal_generation()` bắt đầu sau ngày `rg` để có dữ liệu lịch sử.

**Ngưỡng.** Mỗi ngày vào 03:00 EST, giá tham chiếu cộng `param × range` cho biên trên và trừ `(1-param) × range` cho biên dưới. Vượt biên trên long, dưới biên short. Khi giá đi xuyên qua phía đối diện trong cùng ngày, lệnh được đảo chiều bằng tín hiệu ±2. Vị thế còn mở được đóng lúc 12:00 EST.

**Tham số/đầu vào.** `param` nên trong (0,1), `rg` xác định cửa sổ range; dùng cùng file phút `gbpusd.csv` như London Breakout. `column` xác định chuỗi giá dùng làm trigger.

**Hạn chế.** Mốc giờ và index date phải khớp chính xác; ngày thiếu dữ liệu/holiday, timezone, daylight saving và các thao tác quy đổi phút→ngày có ảnh hưởng lớn. Backtest bỏ qua spread và chi phí đảo chiều intraday.

## 10. Pair Trading — `Pair trading backtest.py`

**Kiểm định cặp.** `EG_method(X,Y)` chạy OLS `Y ~ const + X`, lấy residual rồi kiểm định ADF; khi residual stationary theo ngưỡng p-value trong hàm, cặp được xem là cointegrated. `main()` minh họa NVDA và AMD trong 2013–2014.

**Tín hiệu rolling.** `signal_generation()` dùng cửa sổ `bandwidth=250`; mỗi ngày kiểm tra lại đồng liên kết trên dữ liệu trước đó, tính fitted value và residual, chuẩn hóa residual thành z-score, rồi phát trạng thái ngược chiều spread: residual cao → short tài sản Y/long X theo quy ước vị thế; residual thấp → chiều ngược lại. Khi đồng liên kết mất, code xóa trạng thái đang mở. `portfolio()` đặt vốn 20.000 cho mỗi chân và quy đổi thành số cổ phiếu theo giá cực đại trong mẫu.

**Sửa trong lượt review.** Threshold dùng ±1 trên z-score đã chuẩn hóa. Trước đây code cộng/trừ độ lệch chuẩn residual theo đơn vị giá vào z-score, khiến threshold sai đơn vị và điều kiện không thể kích hoạt tại hàng đầu tiên.

**Rủi ro.** Cặp có thể mất đồng liên kết; ADF có thể nhạy với cửa sổ và lựa chọn cặp. Portfolio dùng số cổ phiếu cố định, không hedge theo hệ số hồi quy; vốn và market neutrality không được bảo đảm.
