# Đánh giá khả năng tích hợp Quant Trading với Trading Desk

**Mục tiêu:** đánh giá code hiện có trong `quant-trading`, đối chiếu với kiến trúc thật tại `/Users/phitran/fublab/trading-desk`, và đề xuất chức năng cùng lộ trình tích hợp theo phase. Tài liệu này là phân tích và thiết kế sơ bộ; chưa triển khai service hoặc thay đổi Trading Desk.

## Kết luận

Quant Trading có thể được tận dụng làm **research/backtest engine** cho crypto perpetual futures. Phần có thể tái sử dụng tốt nhất là ý tưởng và công thức chiến lược; phần lớn script hiện tại chưa thể chạy như service ngay vì chúng là chương trình nghiên cứu độc lập, dùng dữ liệu và schema khác nhau, phụ thuộc thư viện chưa được quản lý tập trung, và nhiều backtest bỏ qua chi phí giao dịch.

Trading Desk đã có Quant Lab, API backtest, market-data endpoints, Gateway/NATS, và paper trading. Do đó hướng phù hợp là tách tính toán chiến lược khỏi API thành service nội bộ Python, nhưng để Trading Desk API giữ xác thực, quyền người dùng, lấy dữ liệu và điều phối kết quả. Giai đoạn đầu chỉ dùng backtest; về sau mới nối paper trading. Không cho Quant service giữ API key sàn hoặc gọi Gateway đặt lệnh trực tiếp.

## Code Quant hiện có và mức độ phù hợp

Repo này có 26 script Python, chủ yếu là notebook đã xuất thành `.py`. Các file tạo tín hiệu gồm MACD, Awesome Oscillator, Heikin-Ashi, Parabolic SAR, RSI, Shooting Star, Bollinger pattern, London Breakout, Dual Thrust và Pair Trading. Repo còn có mô phỏng Monte Carlo, nghiên cứu Oil Money, định giá options/VIX, phân tích quặng sắt và dự báo nông nghiệp.

| Nhóm code | Khả năng dùng với crypto futures | Ghi chú |
|---|---|---|
| MACD, RSI, Bollinger, Parabolic SAR, Dual Thrust | Cao nhất để làm nguyên mẫu | Chuyển thành hàm nhận OHLCV chuẩn, khai báo timeframe và quy tắc vào/ra rõ ràng. |
| Pair Trading | Có điều kiện | Cần đồng bộ timestamp, kiểm tra cointegration theo rolling/out-of-sample, quản lý hedge ratio và rủi ro hai chân lệnh. |
| Monte Carlo | Hỗ trợ nghiên cứu/risk analysis | Không phải strategy executor; kết quả phụ thuộc giả định phân phối và chất lượng chuỗi đầu vào. |
| Options Straddle, VIX Calculator | Thấp cho perp | Dựa trên dữ liệu options/volatility và không chuyển trực tiếp sang perpetual contract. |
| Oil Money, Ore Money, Smart Farmers | Không phải logic crypto futures | Có thể tham khảo quy trình phân tích, nhưng dữ liệu/đơn vị/mô hình khác sản phẩm cần tích hợp. |

Trong [REPO_CODE_GUIDE](REPO_CODE_GUIDE.md) và tài liệu chiến lược, repo đã được mô tả chi tiết hơn. Code gốc thể hiện các giới hạn quan trọng: mỗi script tự nạp dữ liệu; không có package/API chung; không có manifest phụ thuộc cho toàn repo; nhiều lệnh xử lý và vẽ chạy ở cấp module; README gốc mô tả backtest bỏ qua phí, slippage và thanh khoản. Vì vậy không nên expose trực tiếp các script cho Trading Desk hoặc coi số liệu cũ là hiệu quả futures đã được xác nhận.

## Trading Desk hiện có gì liên quan?

Phân tích dưới đây dựa trên checkout đúng anh cung cấp: `/Users/phitran/fublab/trading-desk`, cùng các thành phần sibling `/Users/phitran/fublab/fts`, `/Users/phitran/fublab/gateway` và `/Users/phitran/fublab/protos`.

### Quant Lab và API

- Frontend Quant Lab nằm tại `trading-desk/app/src/components/panels/AdvancedWorkspace.tsx`; UI gọi `POST /api/v1/quant/backtests`.
- Route nằm ở `trading-desk/api/src/advanced-tools/advanced-tools.controller.ts`; tính toán hiện ở `advanced-tools.service.ts`.
- API hiện fetch Binance USDⓈ-M daily klines trực tiếp, tính bốn lựa chọn đơn giản (`Trend + momentum`, `Mean reversion`, `Breakout filter`, `Funding neutral`) ngay trong NestJS và dùng nến sinh mẫu nếu fetch lỗi. Kết quả có equity curve, return, drawdown, profit factor, win rate, Sharpe ước tính và diagnostics.
- Phần này là prototype phục vụ Quant Lab; không phải engine thực thi strategy. Fallback synthetic data có thể làm kết quả trông hợp lệ dù dữ liệu sàn không có, nên cần được thay bằng trạng thái lỗi rõ ràng trước khi dùng kết quả để ra quyết định.
- API cũng có `GET /market-data/futures/klines` làm proxy nến Binance, có interval, time cursor và giới hạn trang. Đây là điểm lấy dữ liệu lịch sử có sẵn để API điều phối backtest.

### Market data, Gateway và NATS

- `GatewayMarketDataService` trong API giao tiếp Gateway qua NATS request/reply và stream; code có order book stream, funding stream/health, commission query và instrument query.
- Rust Gateway có Binance USDⓈ-M adapter và market-data service. Code hiện tại có stream realtime order book/funding và persistent subscription state qua Redis.
- Kline endpoint đang ở API gọi Binance trực tiếp; trong các file Gateway/protos đã rà soát chưa thấy contract historical OHLCV tương ứng. Không nên giả định NATS/Gateway hiện đã cung cấp lịch sử candles cho backtest.
- Funding stream realtime không thay thế lịch sử funding rate. Backtest funding cần chuỗi rate và timestamp quá khứ phù hợp; user funding income history cũng không đồng nghĩa với funding-rate history cho mọi symbol.

### Paper execution và FTS

- Trading Desk có paper Futures orders/positions, dùng mark price Binance, ghi fills, phí mô phỏng, leverage/margin và trạng thái vị thế. Đây có thể là đích cho paper signal ở phase sau.
- FTS là service Rust riêng cho quote/workflow chuyển tiền. FTS không phải nơi chạy strategy và không nên dùng làm adapter cho quant calculations.
- Gateway là ranh giới exchange I/O. Quant service không nên nhận credentials, ký request, hoặc bypass API/OMS/risk để nói trực tiếp với sàn.

## Kiến trúc đích đề xuất

```text
Quant Lab UI
    │ session/auth hiện có
    ▼
Trading Desk API (NestJS)
    ├── xác thực user, quyền và input
    ├── lấy OHLCV qua market-data API hiện có
    └── gọi private service contract
             │ HTTP nội bộ ở MVP; timeout + service identity
             ▼
Quant Research Service (Python)
    ├── registry strategy/version
    ├── backtest deterministic, không gọi sàn
    └── trả metrics, equity curve, provenance, limitations

Phase paper về sau:
Quant result/signal → API risk checks → existing Paper Trading API

Chỉ sau một quyết định riêng về live execution:
Strategy signal → OMS/Risk → versioned Gateway contract → exchange
```

Python là lựa chọn thực dụng vì strategy source đang là Python. Nên đóng gói service/container riêng, tách khỏi process NestJS nhưng đặt trong hệ sinh thái `/Users/phitran/fublab` theo cách tương tự FTS. MVP backtest có thể stateless, không cần database riêng: API gửi request đã xác thực và dữ liệu candles; service trả kết quả. Chỉ thêm database/job queue khi khối lượng backtest dài, cần lưu lịch sử/replay, hoặc cần chạy bất đồng bộ.

Trading Desk API vẫn là entry point của browser. Dùng HTTP private service-to-service cho backtest đồng bộ là đủ ở MVP; nếu chạy NATS thì cần versioned message contract, timeout/reply correlation và payload limits. Không gửi bearer session của người dùng xuống service; dùng service identity riêng. Service chỉ được nhận dữ liệu và tham số đã validate.

## Phân chia chức năng

### Quant service nên sở hữu

- Chuẩn hóa và kiểm tra dữ liệu nhận vào; không tự tải dữ liệu từ exchange ở MVP.
- Strategy registry gồm tên, version, timeframe, tham số được phép và mô tả logic.
- Tính tín hiệu, vị thế giả lập, PnL, equity curve, drawdown, trade count và các chỉ số được định nghĩa rõ.
- Tính kết quả tái lập được từ cùng data snapshot + strategy version + config.
- Trả diagnostics: candle count, khoảng thời gian, data source/version, fees/funding/slippage assumptions và cảnh báo dữ liệu thiếu.

### Trading Desk API nên sở hữu

- User/session auth, workspace/team/account authorization, rate limit và validate request.
- Chọn venue/instrument, lấy lịch sử nến và sau này lấy funding/mark-price history.
- Chuyển request sang private service; chuẩn hóa timeout/unavailable/validation errors.
- Lưu/hiển thị kết quả nếu sản phẩm cần persistence; quản lý paper/live mode và quyền thao tác.

### Gateway nên sở hữu

- Giao tiếp exchange, private credentials, exchange-specific semantics, market stream và execution.
- Mở rộng historical data contract nếu cần nhiều venue hoặc không muốn API gọi thẳng Binance.
- Thực thi order chỉ theo contract được OMS/Risk cho phép; không nhận raw strategy code.

## Lộ trình tích hợp theo phase

| Phase | Mục tiêu | Phạm vi dự kiến | Điều kiện hoàn tất |
|---|---|---|---|
| **0. Chốt phạm vi** | Chọn một use case nhỏ và dữ liệu chuẩn | Chọn 1 strategy, ưu tiên MACD/RSI/Dual Thrust; xác định Binance USDⓈ-M, symbol, interval, timezone, candles, fee/funding/slippage semantics, benchmark và thời gian lookback. | Có contract request/response, glossary metrics và danh sách giả định được duyệt. |
| **1. Chuẩn hóa strategy** | Tách logic nghiên cứu khỏi script | Chuyển một strategy thành pure function; bỏ side effects `chdir`, đọc file, vẽ chart và lệnh chạy ở import; version tham số; candle schema thống nhất; chống look-ahead và xác định cách xử lý missing/gap. | Cùng snapshot/config cho kết quả deterministic; có unit và regression tests; kết quả độc lập khỏi dữ liệu mẫu âm thầm. |
| **2. Backtest service nội bộ** | Nối một strategy vào Quant Lab | Container Python riêng; private HTTP API; health/readiness; service identity; payload/timeouts/rate limit; API lấy klines qua endpoint hiện có rồi gọi service; trả provenance và lỗi upstream rõ. Chưa có database/job queue nếu backtest ngắn. | Compose local chạy được; service không có exchange secrets/order routes; Quant Lab hiển thị kết quả với nguồn dữ liệu/giả định. |
| **3. Futures realism và dữ liệu** | Giảm sai lệch giữa backtest và futures | Bổ sung historical funding rate, mark price, fee tiers, slippage/spread, tick/step filters, leverage/margin, liquidation/maintenance margin và quy tắc fill. Chốt data source (API direct cho Binance MVP hoặc Gateway contracts cho multi-venue); lưu snapshot/cache nếu cần replay. | Báo cáo nêu đầy đủ nguồn, coverage, thiếu dữ liệu và từng cost component; tính năng không giả vờ có funding/liquidation khi thiếu. |
| **4. Paper forward test** | Đối chiếu tín hiệu với paper execution | Strategy phát signal/intention có strategy version và timestamp; API áp risk limits; người dùng chủ động gửi lệnh paper qua paper endpoints hiện có. Theo dõi signal-to-order/fill, mark price, phí và divergence. | Đủ giai đoạn/số mẫu đã định trước; restart, stale data, duplicate signal, kill switch và giới hạn account được kiểm tra; chưa có live credentials. |
| **5. Execution orchestration (quyết định riêng)** | Chỉ xem xét tự động hóa sau paper evidence | Signal đi qua OMS/Risk và versioned Gateway contract; idempotency, order reconciliation, position limits, stale-market-data gate, kill switch, audit, canary/allowlist và rollout từng account/instrument. | Review vận hành/bảo mật/risk riêng và explicit approval trước bất kỳ mainnet order path nào. Không phải điều kiện để hoàn thành backtest service. |

Không cần triển khai tất cả phase ngay. Phase 0–2 tạo được Quant service dùng trong Quant Lab; Phase 3 cần thiết trước khi gọi kết quả là backtest futures có độ trung thực cao; Phase 4 mới nối paper trading. Phase 5 là quyết định sản phẩm và rủi ro độc lập.

## Contract MVP nên chốt trước Phase 2

Request gợi ý:

```json
{
  "schemaVersion": "v1",
  "symbol": "BTCUSDT",
  "venue": "BINANCE",
  "marketType": "USD_M_PERPETUAL",
  "interval": "1d",
  "strategyId": "macd-trend",
  "strategyVersion": "1.0.0",
  "parameters": {"fastPeriod": 12, "slowPeriod": 26, "signalPeriod": 9},
  "initialEquity": "10000",
  "feeModel": {"makerBps": "...", "takerBps": "..."},
  "candles": [{"openTime": 0, "open": "...", "high": "...", "low": "...", "close": "...", "volume": "..."}]
}
```

Response nên có `runId`/`requestHash`, strategy/version, data source + from/to + candle count, summary metrics, equity curve, closed-trade ledger và diagnostics. Dùng decimal strings cho giá/khối lượng/tiền trong contract để tránh hiểu khác nhau về độ chính xác; có thể trả metrics display dạng number khi API xác định rõ rounding.

`riskPercent` hiện trên UI không đủ rõ để coi là risk-at-stop: code hiện tại lấy một tỷ lệ equity làm allocation/exposure, chưa thấy stop-distance sizing trong backtest prototype. Phase 0 cần đổi tên thành `exposurePercent` hoặc định nghĩa công thức risk budget thật (vốn chịu lỗ / khoảng cách stop / contract value) trước khi tái sử dụng.

## Rủi ro và quyết định cần tránh

1. **Không dùng fallback synthetic data cho kết quả người dùng có thể hiểu là lịch sử thật.** API nên trả lỗi hoặc đánh dấu rõ demo fixture.
2. **Không coi fee 4 bps cố định là đủ.** Maker/taker, venue, VIP tier, funding và execution style làm thay đổi kết quả.
3. **Daily close backtest không mô phỏng intraday liquidation.** Cần mark-price path và margin model trước khi mô tả liquidation risk.
4. **Không biến scripts thành execution code bằng cách gọi trực tiếp `placeOrder`.** Bước paper phải đi qua API/risk controls; live cần phase và approval riêng.
5. **Không lấy user funding payment history làm dữ liệu funding forecast.** Đây là hai loại dữ liệu khác nhau.
6. **Không xây dựng một service cho mỗi file Python.** Registry/service nên có contract chung; mỗi strategy là module/version có metadata và acceptance evidence.
7. **Không chọn microservice vì tên “service”.** Nếu chỉ có một backtest ngắn và một người dùng, có thể tạm giữ logic sau API boundary; tách container khi cần cô lập Python dependencies, mở rộng tài nguyên độc lập hoặc sở hữu vòng đời model riêng.

## Những việc chưa được thực hiện

Chưa tích hợp Quant với `/Users/phitran/fublab/trading-desk`; chưa thay đổi API, UI, Compose, Gateway, protos, FTS hay database của checkout đó. Chưa chuyển strategy cũ thành package/service production. Tài liệu này chỉ mô tả hiện trạng và đề xuất các phase để anh quyết định phạm vi tiếp theo.
