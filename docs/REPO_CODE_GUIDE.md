# Hướng dẫn đọc và giải thích mã nguồn Quant Trading

Tài liệu này mô tả mã nguồn hiện có trong repository, dựa trên các script Python, README, tên cột dữ liệu được dùng và luồng xử lý thể hiện trong code. Đây là bản đồ để đọc và chạy từng phần, không phải xác nhận rằng các kết quả backtest có thể tái lập trên mọi máy.

Tài liệu phân tích chi tiết theo nhóm nằm trong [docs/README.md](README.md); dùng các file đó để xem từng dự án và từng chiến lược.

## 1. Repo này làm gì?

Repo gồm các chương trình nghiên cứu định lượng viết theo kiểu notebook đã xuất thành `.py`. Hầu hết không phải thư viện có cấu trúc package: mỗi file tự nạp dữ liệu, định nghĩa hàm tính toán/tín hiệu, vẽ biểu đồ và thường có `main()` ở cuối.

Các nhóm chính:

| Nhóm | Mục đích | Thư mục / file |
|---|---|---|
| Chiến lược chỉ báo kỹ thuật | Tạo tín hiệu mua/bán từ giá và mô phỏng vị thế | Các file `*backtest.py` ở thư mục gốc |
| Định giá quyền chọn | Xây dựng long straddle và tính chỉ số VIX từ chuỗi quyền chọn | `Options Straddle backtest.py`, `VIX Calculator.py` |
| Mô phỏng | Dùng mô phỏng Monte Carlo để kiểm tra dự báo giá | `Monte Carlo project/` |
| Phân tích dầu và tiền tệ | Phân tích quan hệ dầu–FX, mô hình và backtest | `Oil Money project/` |
| Phân tích quặng sắt | Vẽ sản lượng quặng sắt theo quốc gia | `Ore Money project/` |
| Nông nghiệp Malaysia | Làm sạch dữ liệu, chọn quốc gia, ước lượng cầu và dự báo | `Smart Farmers project/` |

README gốc mô tả các chiến lược là backtest lịch sử và giả định giao dịch không có ma sát (không slippage, phí, hoặc giới hạn thanh khoản). Vì thế số liệu đầu ra không phản ánh đầy đủ kết quả giao dịch thực tế.

## 2. Kiến trúc và luồng dữ liệu phổ biến

Nhiều backtest tuân theo chuỗi xử lý sau:

1. Nạp OHLC hoặc dữ liệu kinh tế từ CSV/Excel hay Yahoo Finance.
2. Chuẩn hóa bảng theo ngày và tính chỉ báo/phân tích thống kê.
3. Gán tín hiệu vào từng hàng dữ liệu (thường `1` mua, `-1` bán/short, `0` đứng ngoài; ý nghĩa chính xác tùy file).
4. Chuyển tín hiệu thành vị thế và giá trị tài khoản.
5. Tạo biểu đồ giá, vị thế, oscillator hoặc kết quả mô phỏng.

Các file không dùng chung một giao diện dữ liệu chuẩn. Tên cột như `Open`, `High`, `Low`, `Close`, `Adj Close`, `Volume`, `Date`, `Year`, `Value`, `Element`, `Item` được giả định trực tiếp trong từng script. Cần kiểm tra cách script đọc dữ liệu trước khi thay nguồn dữ liệu.

## 3. Chiến lược ở thư mục gốc

### MACD Oscillator — `MACD Oscillator backtest.py`

- **Ý tưởng:** tính hai đường trung bình động trên `Close`; chênh lệch ngắn hạn/dài hạn tạo tín hiệu xu hướng.
- **Hàm:** `macd(signals, ma1, ma2)` thêm các cột MA; `signal_generation(df, method, ma1, ma2)` dựng tín hiệu; `plot(new, ticker)` vẽ; `main()` nạp dữ liệu và chạy quy trình.
- **Cách đọc:** bắt đầu từ `main()` để xem ticker, khung thời gian và cách lấy dữ liệu, sau đó xem `signal_generation` để xác định thời điểm đổi trạng thái.
- **Lưu ý:** chu kỳ MA được truyền rõ qua tham số, không còn phụ thuộc biến global. Script import `fix_yahoo_finance`, một package cũ.

### Pair Trading — `Pair trading backtest.py`

- **Ý tưởng:** chọn hai tài sản có quan hệ đồng liên kết; dùng phần dư của hồi quy Engle–Granger làm thước đo độ lệch. Khi spread lệch khỏi vùng ngưỡng, mua tài sản rẻ tương đối và bán tài sản đắt tương đối.
- **Hàm:** `EG_method(X, Y, show_summary)` kiểm tra/hồi quy Engle–Granger; `signal_generation(asset1, asset2, method, bandwidth)` tạo spread và vị thế; `plot(...)` trực quan hóa; `portfolio(data)` tính chuỗi tài sản; `main()` nạp và ghép dữ liệu.
- **Rủi ro mô hình:** đồng liên kết trong quá khứ có thể biến mất; kết quả nhạy với cặp tài sản, khoảng thời gian, tần suất và ngưỡng. Cần xác minh căn chỉnh ngày của cả hai chuỗi.

### Heikin-Ashi — `Heikin-Ashi backtest.py`

- **Ý tưởng:** biến đổi OHLC thành nến Heikin-Ashi để làm mượt nhiễu, rồi dùng màu/hình thái nến cho tín hiệu theo xu hướng.
- **Hàm:** `heikin_ashi(data)` tính OHLC mới; `signal_generation(df, method, stls)` quyết định tín hiệu và stop loss; `candlestick(...)` vẽ nến; `portfolio`, `profit`, `stats` tính hiệu quả; `omega`, `sortino`, `mdd` tính các thước đo rủi ro; `main()` điều phối.
- **Đầu vào:** dữ liệu OHLC và vốn ban đầu; code dùng `yfinance`, SciPy, pandas, NumPy, matplotlib.

### London Breakout — `London Breakout backtest.py`

- **Ý tưởng:** dùng vùng giá của giờ trước phiên London làm ngưỡng breakout; kiểm tra biến động đầu phiên, vào lệnh theo hướng vượt ngưỡng và đóng lệnh theo stop/target hoặc cuối phiên.
- **Hàm:** `london_breakout(df)` chuẩn bị đặc trưng/khung thời gian; `signal_generation(df, method)` áp dụng quy tắc; `plot(new)` vẽ kết quả; `main()` nạp dữ liệu phút và chạy backtest.
- **Đầu vào:** dữ liệu FX tần suất phút, cần timestamp và OHLC phù hợp. Quy tắc phiên phụ thuộc múi giờ và giờ mùa hè; cần xác nhận timezone dữ liệu.

### Awesome Oscillator — `Awesome Oscillator backtest.py`

- **Ý tưởng:** so sánh MA ngắn và dài của trung điểm giá `(High + Low)/2`; đồng thời giữ phần thử nghiệm MACD dạng EWMA để đối chiếu.
- **Hàm:** `ewmacd`, `signal_generation`, `awesome_ma`, `awesome_signal_generation` tính chỉ báo/tín hiệu; `plot` vẽ; `portfolio`, `profit`, `mdd`, `stats` tổng hợp kết quả; `main()` chạy.
- **Lưu ý:** dùng `fix_yahoo_finance`; phần MACD và AO có quy tắc riêng nên không nên coi cùng một cột tín hiệu là tương đương.

### Bollinger Bands Pattern Recognition — `Bollinger Bands Pattern Recognition backtest.py`

- **Ý tưởng:** tính dải Bollinger và nhận dạng mẫu đáy W bằng quy tắc số học (đáy thứ nhất/thứ hai và vị trí so với dải), thay vì mô hình học máy.
- **Hàm:** `bollinger_bands(df)` tính dải; `signal_generation(data, method)` phân loại/tạo tín hiệu; `plot(new)` hiển thị mẫu; `main()` nạp dữ liệu.
- **Lưu ý:** file đổi thư mục làm việc sang `data/` cạnh repo; thao tác này vẫn tác động global đến tiến trình khi import. Cần xem cả đoạn chuẩn bị dữ liệu trong `main()` trước khi dùng.

### Dual Thrust — `Dual Thrust backtest.py`

- **Ý tưởng:** chiến lược breakout trong ngày; biên trên/dưới lấy từ phạm vi OHLC của các ngày trước, sau đó vị thế được đóng vào cuối ngày.
- **Hàm:** `min2day(df, column, year, month, rg)` chuyển dữ liệu phút thành dữ liệu/ngưỡng theo ngày; `signal_generation(df, intraday, param, column, rg)` dựng ngưỡng và lệnh; `plot(...)` vẽ; `main()` chạy.
- **Lưu ý:** file đổi thư mục làm việc sang `data/` cạnh repo; thao tác này vẫn tác động global đến tiến trình khi import. Tần suất, cột thời gian và quy tắc gom phiên quyết định kết quả.

### Parabolic SAR — `Parabolic SAR backtest.py`

- **Ý tưởng:** tự tính Parabolic Stop and Reverse với hệ số gia tốc, điểm cực trị và trạng thái xu hướng; chuyển SAR qua/ra khỏi giá để đảo chiều.
- **Hàm:** `parabolic_sar(new)` tính chuỗi SAR; `signal_generation(df, method)` ánh xạ kết quả sang giao dịch; `plot(new, ticker)` vẽ; `main()` lấy dữ liệu.
- **Lưu ý:** code có một số tham số AF quy ước trong thân hàm; đối chiếu chúng với thị trường/khung dữ liệu mục tiêu trước khi diễn giải hiệu năng.

### RSI Pattern Recognition — `RSI Pattern Recognition backtest.py`

- **Ý tưởng:** tính RSI bằng smoothed moving average; ngoài ngưỡng quá mua/quá bán, code còn thử nhận diện các hình thái RSI như vai–đầu–vai và các mẫu liên quan.
- **Hàm:** `smma`, `rsi` tính oscillator; `signal_generation` tạo tín hiệu đơn giản; `pattern_recognition(df, method, lag)` tìm mẫu; `plot` và `pattern_plot` trực quan hóa; `main()` điều phối.
- **Lưu ý:** quy tắc mẫu phụ thuộc số phiên `lag`; đây là bộ quy tắc được mã hóa thủ công, không phải mô hình học đã huấn luyện.

### Shooting Star — `Shooting Star backtest.py`

- **Ý tưởng:** phát hiện nến shooting star qua màu nến, thân, bóng trên/dưới và ngưỡng kích thước; sau đó mô phỏng lệnh đảo chiều với stop và giới hạn thời gian nắm giữ.
- **Hàm:** `shooting_star(data, lower_bound, body_size)` phân loại nến; `signal_generation(...)` áp dụng vào lệnh; `candlestick(...)` và `plot(...)` vẽ; `main()` nạp dữ liệu.
- **Đầu vào:** OHLC từ `yfinance`; điều kiện hình nến và ngưỡng stop/holding period nằm trong tham số.

## 4. Quyền chọn và độ biến động

### Options Straddle — `Options Straddle backtest.py`

- **Ý tưởng:** mô phỏng long straddle, tức mua call và put cùng strike/đáo hạn để hưởng lợi khi biến động đủ lớn.
- **Hàm:** `find_strike_price(df)` chọn strike gần spot; `straddle(options, spot, contractsize, strikeprice)` ghép payoff/giá trị hai chân; `signal_generation(df, threshold)` xác định giao dịch; `plot(...)` vẽ payoff; `main()` đọc dữ liệu và chạy.
- **Dữ liệu:** repo kèm `data/stoxx50.xlsx`; README của script mô tả các bảng spot và chuỗi options. Cần kiểm tra sheet, mã quyền chọn, đơn vị hợp đồng và cách ghép ngày trước khi tính.
- **Lưu ý:** payoff tại đáo hạn và lợi nhuận khi mark-to-market trong thời gian nắm giữ là hai đại lượng khác nhau; xem chính xác cách code định nghĩa giá trị vị thế.

### VIX Calculator — `VIX Calculator.py`

- **Ý tưởng:** triển khai các bước tính variance từ quyền chọn OTM ở hai kỳ hạn, nội suy về kỳ hạn mục tiêu rồi quy đổi thành chỉ số biến động năm hóa.
- **Hàm:** `cmt_rate_fill_date` điền lịch lãi suất; `get_settlement_day` và `get_time_to_expiration` xử lý thời điểm đáo hạn; `get_forward_strike` suy ra forward và strike; `get_options_call_inclusion` / `get_options_put_inclusion` chọn các options được đưa vào tổng; `compute_sigma` tính variance kỳ hạn; `compute_vix` nội suy; `vix_calculator` ghép quy trình; `main()` nạp dữ liệu.
- **Dữ liệu:** cần chuỗi option có ngày giao dịch, ngày đáo hạn, strike, loại call/put, giá/quote và lãi suất kỳ hạn. Lịch nghỉ ảnh hưởng số phút còn lại.
- **Lưu ý:** file đổi thư mục làm việc sang `data/` cạnh repo; thao tác này vẫn tác động global đến tiến trình khi import. Công thức nhạy với quote rỗng/zero bid và quy tắc lọc strike.

## 5. Các dự án trong thư mục con

### Monte Carlo project

`Monte Carlo project/Monte Carlo backtest.py` ước lượng các tham số từ chuỗi lợi suất, sinh đường giá ngẫu nhiên bằng giả định kiểu geometric Brownian motion, xếp hạng các mô phỏng theo sai số trong giai đoạn huấn luyện rồi đánh giá hướng/độ bao phủ trên giai đoạn kiểm tra.

- `monte_carlo(data, testsize, simulation)` tạo tập huấn luyện/kiểm tra và môỏng các đường đi.
- `plot(df, forecast_horizon, d, pick, ticker)` trình bày dự báo được chọn.
- `test(df, ticker, simu_start, simu_end, simu_delta)` so sánh nhiều số lượng mô phỏng.
- `main()` lấy chuỗi giá và khởi chạy thử nghiệm.

README dự án dùng GE và NVDA để minh họa rằng mô phỏng khớp lịch sử tốt không đồng nghĩa dự báo hướng hay biến cố cực đoan tốt. Đây là kiểm tra giới hạn mô hình, không phải công cụ dự báo chắc chắn. Tính ngẫu nhiên và cách chia train/test ảnh hưởng kết quả; cần giữ seed nếu muốn tái lập.

### Oil Money project

Mục tiêu dự án là khảo sát mối liên hệ dầu thô–tiền tệ của các nước xuất khẩu dầu và thử chiến lược dự báo/động lượng. README phân tích CAD, COP, NOK, RUB; các file `Oil Money CAD.py`, `Oil Money COP.py`, `Oil Money NOK.py`, `Oil Money RUB.py` chứa các notebook phân tích quốc gia. Chúng không cùng một cấu trúc hoàn toàn: có hồi quy/OLS, kiểm tra dự báo, clustering/KMeans và biểu đồ tùy theo thị trường.

`Oil Money Trading backtest.py` là phần chiến lược chung:

- `oil_money(dataset)` chuẩn bị cột tín hiệu và các biên độ sigma/forecast.
- `signal_generation(dataset, x, y, method, holding_threshold, stop, rsquared_threshold, train_len)` ước lượng quan hệ trên cửa sổ huấn luyện, tạo dự báo và tín hiệu khi độ lệch đạt ngưỡng; tham số kiểm soát thời gian giữ, stop, chất lượng hồi quy và độ dài huấn luyện.
- `portfolio(signals, close_price, capital0)` chuyển lệnh thành giá trị danh mục; `plot`, `profit` hiển thị/tổng hợp kết quả; `main()` khởi chạy.

Thư mục `oil production/` có `oil production cost curve.py` để lập đường cong chi phí và `oil production choropleth.py` để tô bản đồ theo quốc gia bằng Folium. Các CSV cùng thư mục là đầu vào bản đồ/đường cong; `worldmapshape.json` ở đây là dữ liệu hình học địa lý.

**Lỗi đã sửa trong lượt review:** file NOK có hai lỗi cú pháp (chuỗi nhãn lãi suất bị ngắt và thiếu dấu phẩy trong lời gọi `signal_generation`) và dùng import tên module không khớp tên file có dấu cách. Các lỗi này đã được sửa; module backtest được nạp theo đường dẫn file.

### Ore Money project

README giới thiệu ý tưởng mở rộng Oil Money sang quan hệ quặng sắt và tiền tệ của các nước xuất khẩu. Mã nguồn hiện có là `iron ore production/iron ore production bubble map.py`: đọc bảng quốc gia/sản lượng, ánh xạ vị trí và kích thước điểm lên bản đồ thế giới bằng Basemap/Matplotlib. CSV cùng thư mục là dữ liệu bản đồ; PNG trong `preview/` là ảnh kết quả được lưu sẵn. Đây là trực quan hóa dữ liệu, chưa phải backtest chiến lược quặng sắt.

### Smart Farmers project

Dự án dùng dữ liệu cây trồng, sản lượng, đất, giá, GDP và dân số để ước lượng cầu và phân bổ sản xuất.

- `cleanse data.py`: hàm `prepare(target_land, target_prod, target_prix)` lọc cột, tách `Area harvested` và `Production`, ghép dữ liệu đất/giá/sản lượng và chuẩn hóa bảng cho bước kế tiếp.
- `check consistency.py`: kiểm tra/so sánh tính nhất quán của các file dữ liệu đầu vào/đã làm sạch.
- `country selection.py`: chọn tập quốc gia/crop phù hợp từ các bảng đầu vào, chuẩn bị dữ liệu liên quốc gia cho ước lượng.
- `estimate demand.py`: `create_xy(...)` dựng biến mục tiêu giá và biến giải thích như GDP, dân số, sản lượng; `lin_reg(...)` chạy hồi quy; `constrained_ols(x, y)` áp ràng buộc hệ số bằng tối ưu hóa; `get_params(...)` lặp việc ước lượng cho crop.
- `forecast.py`: `prepare(grand)` chia bảng theo năm; `get_ans(...)` giải bài toán tối ưu bậc hai với ràng buộc; các hàm `get_production(...)` xây dựng nghiệm/dự báo sản xuất, `costfunction`, `ls_estimate`, `find_init`, `write_file`, `compute_price` hỗ trợ tối ưu và xuất kết quả.

**Lưu ý đọc code:** bản đầu của `get_production` đã được đổi tên `get_production_linear` để không bị định nghĩa tối ưu bậc hai ở dưới ghi đè. `get_production` hiện trỏ rõ ràng tới phiên bản quadratic. Các phép đổi ma trận đã chuyển từ `np.mat` sang mảng NumPy và nối bảng dùng `pd.concat` để tương thích pandas/NumPy mới.

## 6. Dữ liệu và tài sản đi kèm

- `data/`: `bitcoin.csv`, `gbpusd.csv`, `henry hub european options.csv`, `treasury yield curve rates.csv`, `cme holidays.csv` và `stoxx50.xlsx` cung cấp chuỗi giá, options, lãi suất và lịch nghỉ cho các ví dụ.
- `Smart Farmers project/data/`: các bảng dân số, GDP, đất, giá, sản lượng, dự báo và mapping quốc gia/crop.
- `Oil Money project/data/`: các chuỗi dầu và tỷ giá phục vụ phân tích từng đồng tiền.
- `Oil Money project/oil production/`, `Ore Money project/iron ore production/`: dữ liệu bản đồ và sản lượng.
- `preview/` và các thư mục `preview/` trong dự án: biểu đồ kết quả đã xuất, không phải mã nguồn thực thi.

Tên file không có metadata schema đầy đủ. Khi thay CSV/XLSX, hãy đối chiếu header, loại dữ liệu ngày, đơn vị tiền tệ, tần suất và thứ tự cột với thao tác `read_csv`/`read_excel` trong script tương ứng.

## 7. Cách tiếp cận để chạy và kiểm tra

1. Chọn một script đơn lẻ và đọc `main()` trước để tìm dữ liệu đầu vào, khoảng thời gian, ticker và tham số.
2. Kiểm tra phần import và các lệnh `os.chdir` ở cấp module. Các đường dẫn Windows tuyệt đối đã được thay bằng đường dẫn tương đối với vị trí script, nhưng việc đổi working directory vẫn có tác động global.
3. Cài đủ thư viện được import trong chính script: thường gặp pandas, NumPy, matplotlib, SciPy, statsmodels, scikit-learn, yfinance/fix_yahoo_finance, Folium, Basemap, cvxopt. Repo không có manifest phụ thuộc chung (ví dụ `requirements.txt`/`pyproject.toml`).
4. Xác nhận dữ liệu cột và thời gian khớp với giả định script; đặc biệt FX intraday, options và dữ liệu kinh tế.
5. Đọc cách tính vị thế, phí, vốn, stop và giá khớp. README gốc nói các backtest bỏ qua phí/slippage/thanh khoản, và từng script có thể dùng quy tắc khác nhau.

## 8. Các vấn đề kỹ thuật nổi bật khi rà soát

- Các script đọc dữ liệu còn đổi working directory ở cấp module, có thể gây tác dụng phụ khi được import như thư viện.
- Một số script Smart Farmers cần bộ CSV FAO thô mà repo không kèm; các luồng này cần dữ liệu ngoài trước khi chạy trọn vẹn.
- Repo không có cấu hình môi trường/phụ thuộc tập trung, nên phiên bản package cũ như `fix_yahoo_finance` có thể khó cài hoặc không tương thích.
- Một vài hàm còn phụ thuộc biến global và notebook chạy lệnh ở cấp module; điều này làm khó tái sử dụng hàm ngoài luồng chạy dự kiến.
- Nhiều script tạo biểu đồ và kết quả theo trạng thái máy cục bộ; chưa có bộ kiểm thử tự động hoặc quy trình chạy thống nhất được ghi nhận trong repo.

## 9. Bản đồ file Python

Đây là danh sách toàn bộ 26 file Python được rà soát:

```text
Awesome Oscillator backtest.py
Bollinger Bands Pattern Recognition backtest.py
Dual Thrust backtest.py
Heikin-Ashi backtest.py
London Breakout backtest.py
MACD Oscillator backtest.py
Monte Carlo project/Monte Carlo backtest.py
Oil Money project/Oil Money CAD.py
Oil Money project/Oil Money COP.py
Oil Money project/Oil Money NOK.py
Oil Money project/Oil Money RUB.py
Oil Money project/Oil Money Trading backtest.py
Oil Money project/oil production/oil production choropleth.py
Oil Money project/oil production/oil production cost curve.py
Options Straddle backtest.py
Ore Money project/iron ore production/iron ore production bubble map.py
Pair trading backtest.py
Parabolic SAR backtest.py
RSI Pattern Recognition backtest.py
Shooting Star backtest.py
Smart Farmers project/check consistency.py
Smart Farmers project/cleanse data.py
Smart Farmers project/country selection.py
Smart Farmers project/estimate demand.py
Smart Farmers project/forecast.py
VIX Calculator.py
```
