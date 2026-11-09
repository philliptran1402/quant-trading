# Phân tích Monte Carlo project

**Mã nguồn:** [`Monte Carlo project/Monte Carlo backtest.py`](../../Monte%20Carlo%20project/Monte%20Carlo%20backtest.py)  
**Bài toán:** kiểm tra liệu mô phỏng ngẫu nhiên được fit trên lịch sử có dự báo được hướng giá hoặc các đợt giảm cực mạnh ngoài mẫu hay không.

## Ý tưởng thực nghiệm

README nêu lập luận phản biện việc chọn mô phỏng khớp tốt nhất trên train rồi coi đó là dự báo tương lai. Code dùng GE và NVDA ở các giai đoạn có biến động lớn; chia lịch sử thành train/test; tạo nhiều đường đi giá; chọn đường có sai số fit thấp; rồi so hướng và mức giá ở test.

## Luồng code

- `main()` tải giá ticker qua Yahoo Finance và gọi phần đánh giá.
- Dữ liệu giá được chuyển thành log return/đặc trưng tăng trưởng để ước lượng tham số của quá trình giá. Trong mô phỏng, bước tiếp theo được lấy từ phần lợi suất ngẫu nhiên và tái lập thành chuỗi giá.
- `monte_carlo(data, testsize, simulation)` tách khoảng kiểm định, sinh số lượng đường mô phỏng yêu cầu và lưu đường dự báo/metric để xếp hạng.
- `plot(df, forecast_horizon, d, pick, ticker)` vẽ đường được chọn cùng dữ liệu quan sát.
- `test(df, ticker, simu_start, simu_end, simu_delta)` lặp qua nhiều số lượng mô phỏng để xem kết luận có đổi theo iterations không.

## Đầu ra được kiểm tra

README so sánh dự báo hướng cuối kỳ (tăng/giảm) và liệu các đường mô phỏng có bao phủ đáy thực tế của GE/NVDA hay không. Kết luận của dự án là tăng số lần mô phỏng không tự khắc phục được giả định phân phối sai hoặc cú sốc chưa có trong dữ liệu train.

## Giả định và rủi ro

- Lợi suất tương lai được lấy từ phân phối ước lượng bằng quá khứ; regime change/fat tail chưa quan sát không tự xuất hiện trong mô phỏng.
- Chọn đường có fit tốt nhất trên cùng giai đoạn train dễ overfit. Metric tốt trong train không bảo đảm xác suất dự báo ngoài mẫu tốt.
- Cần cố định random seed, cách chia train/test và lịch sử dữ liệu để tái lập kết quả; phần sinh số ngẫu nhiên dùng `random` và NumPy nên seed hai nguồn nếu cần reproducibility.
- Lựa chọn mô hình và tham số trên toàn sample có thể đưa thông tin test vào quyết định. Tách rõ train, validation và test nếu dùng để nghiên cứu định lượng.
- Phân tích này là minh họa giới hạn forecast, không phải hệ thống định giá quyền chọn hay mô hình quản trị rủi ro hoàn chỉnh.

## Cách đọc code

Đọc `main()` để xác định ticker, khoảng ngày và cách lấy dữ liệu. Sau đó theo `monte_carlo()` để xem biến ngẫu nhiên và metric xếp hạng; cuối cùng đọc `test()` để xác định tiêu chí “đúng” và cách tổng hợp các iterations. README có ảnh kết quả GE/NVDA trong `preview/`.
