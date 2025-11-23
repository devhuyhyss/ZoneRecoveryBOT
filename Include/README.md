# AccountPanel.mqh - Tài Liệu Hướng Dẫn

## Tổng Quan
**AccountPanel.mqh** là một class MQL5 dùng để hiển thị thông tin tài khoản và thống kê hiệu suất chiến lược trading theo thời gian thực trên biểu đồ MetaTrader 5.

## Mục Đích
- ✅ Hiển thị thông tin tài khoản (Account ID, Balance)
- ✅ Đếm số lệnh đang mở (BUY/SELL) theo symbol
- ✅ Thống kê lệnh đã đóng (Total, BUY, SELL)
- ✅ Tính toán volume giao dịch tổng và backcom
- ✅ Lọc lịch sử giao dịch theo thời điểm bắt đầu BOT

## Cấu Trúc Class

### Thuộc Tính Private (Riêng Tư)

| Thuộc Tính | Kiểu Dữ Liệu | Mô Tả |
|------------|--------------|-------|
| `m_objName` | string | Tên object label trên chart (kết hợp prefix + symbol) |
| `m_corner` | int | Góc hiển thị panel (0=trái trên, 1=phải trên, 2=trái dưới, 3=phải dưới) |
| `m_xdist` | int | Khoảng cách theo trục X (pixel) |
| `m_ydist` | int | Khoảng cách theo trục Y (pixel) |
| `m_textColor` | color | Màu chữ của panel |
| `m_fontSize` | int | Cỡ chữ của panel |
| `m_start_time` | datetime | Thời điểm bắt đầu BOT (dùng để lọc lịch sử) |
| `m_use_start_time` | bool | Cờ kích hoạt lọc lịch sử theo start time |

### Hằng Số
```cpp
#define PANEL_OBJ_PREFIX "AccountPanel_"
```
- Prefix cho tên object label, đảm bảo tên unique khi kết hợp với symbol

## Các Phương Thức Public

### 1. Init() - Khởi Tạo Panel

**Chức năng:** Tạo và cấu hình object label trên chart để hiển thị thông tin

**Cú pháp:**
```cpp
void Init(
    int corner = 0,              // Góc hiển thị
    int xdist = 10,              // Offset X
    int ydist = 10,              // Offset Y
    color textColor = clrWhite,  // Màu chữ
    int fontSize = 11            // Cỡ chữ
)
```

**Tham số:**
- `corner`: Góc đặt panel (0-3)
  - 0 = Góc trái trên (CORNER_LEFT_UPPER)
  - 1 = Góc phải trên (CORNER_RIGHT_UPPER)
  - 2 = Góc trái dưới (CORNER_LEFT_LOWER)
  - 3 = Góc phải dưới (CORNER_RIGHT_LOWER)
- `xdist`: Khoảng cách từ góc theo chiều ngang (pixel)
- `ydist`: Khoảng cách từ góc theo chiều dọc (pixel)
- `textColor`: Màu sắc văn bản (mặc định: trắng)
- `fontSize`: Kích thước font chữ (mặc định: 11)

**Quy trình hoạt động:**
1. Tạo tên object: `"AccountPanel_" + Symbol`
2. Xóa object cũ nếu đã tồn tại
3. Tạo object label mới với kiểu `OBJ_LABEL`
4. Thiết lập các thuộc tính:
   - Vị trí góc và offset
   - Màu sắc và font size
   - Không cho phép select (`OBJPROP_SELECTABLE = false`)
   - Ẩn trong danh sách object (`OBJPROP_HIDDEN = true`)

**Ví dụ:**
```cpp
CAccountPanel panel;
panel.Init(0, 10, 10, clrWhite, 11);  // Góc trái trên, cách 10px, chữ trắng, size 11
```

---

### 2. SetStartTime() - Thiết Lập Thời Điểm Bắt Đầu

**Chức năng:** Đặt mốc thời gian bắt đầu BOT để lọc lịch sử giao dịch

**Cú pháp:**
```cpp
void SetStartTime(datetime t)
```

**Tham số:**
- `t`: Thời điểm bắt đầu (kiểu datetime)

**Quy tắc:**
- Khi được gọi, `m_use_start_time` sẽ được set = `true`
- Chỉ các deal có thời gian >= `m_start_time` mới được tính vào thống kê
- In log thông báo thời gian đã set

**Ví dụ:**
```cpp
panel.SetStartTime(D'2025.11.22 00:00');  // Chỉ tính deal từ 22/11/2025
```

**Mục đích:**
- Loại bỏ lịch sử giao dịch cũ không liên quan
- Chỉ thống kê từ khi BOT bắt đầu chạy
- Tính toán chính xác hiệu suất của BOT

---

### 3. Deinit() - Hủy Panel

**Chức năng:** Xóa object label khỏi chart khi EA bị deinit

**Cú pháp:**
```cpp
void Deinit()
```

**Quy trình:**
1. Kiểm tra object có tồn tại không
2. Nếu có → Xóa object
3. Giải phóng tài nguyên

**Khi nào gọi:**
- Trong hàm `OnDeinit()` của EA
- Khi muốn ẩn panel

---

### 4. CountOpenPositions() - Đếm Lệnh Đang Mở

**Chức năng:** Đếm số lượng lệnh BUY/SELL đang mở (toàn bộ account và riêng symbol hiện tại)

**Cú pháp:**
```cpp
void CountOpenPositions(
    int &buy_total,   // Output: Tổng số BUY trên tất cả symbol
    int &sell_total,  // Output: Tổng số SELL trên tất cả symbol
    int &buy_sym,     // Output: Số BUY trên symbol hiện tại
    int &sell_sym     // Output: Số SELL trên symbol hiện tại
)
```

**Tham số (tất cả là output):**
- `buy_total`: Tổng số lệnh BUY đang mở trên toàn bộ account
- `sell_total`: Tổng số lệnh SELL đang mở trên toàn bộ account
- `buy_sym`: Số lệnh BUY đang mở trên symbol hiện tại (`_Symbol`)
- `sell_sym`: Số lệnh SELL đang mở trên symbol hiện tại (`_Symbol`)

**Quy trình:**
1. Khởi tạo tất cả biến đếm = 0
2. Duyệt qua tất cả position đang mở (`PositionsTotal()`)
3. Với mỗi position:
   - Lấy ticket và select position
   - Đọc type (BUY/SELL) và symbol
   - Tăng `buy_total` hoặc `sell_total`
   - Nếu symbol = `_Symbol` → tăng `buy_sym` hoặc `sell_sym`

**Ví dụ:**
```cpp
int buy_all, sell_all, buy_current, sell_current;
panel.CountOpenPositions(buy_all, sell_all, buy_current, sell_current);
Print("BUY hiện tại: ", buy_current, " | SELL hiện tại: ", sell_current);
```

---

### 5. GetPerformanceStats() - Lấy Thống Kê Hiệu Suất

**Chức năng:** Phân tích lịch sử giao dịch để tính toán các chỉ số hiệu suất

**Cú pháp:**
```cpp
void GetPerformanceStats(
    int    &total_closed,   // Output: Tổng số lệnh đã đóng
    int    &buy_closed,     // Output: Số lệnh BUY đã đóng
    int    &sell_closed,    // Output: Số lệnh SELL đã đóng
    int    &tp_hit,         // Output: Số lần hit TP
    int    &sl_hit,         // Output: Số lần hit SL
    int    &hoa_count,      // Output: Số lệnh hòa vốn
    double &total_volume,   // Output: Tổng volume đã giao dịch
    double &backcom,        // Output: Backcom tính được
    double  base_lot,       // Input: Lot cơ bản (để tính ngưỡng hòa)
    double  spread_value,   // Input: Spread value (để tính ngưỡng hòa)
    double  backcom_base    // Input: Tỷ lệ backcom % (ví dụ: 6.4)
)
```

**Tham số Input:**
- `base_lot`: Lot size cơ bản (thường là 0.01)
- `spread_value`: Giá trị spread hiện tại
- `backcom_base`: Tỷ lệ backcom % (ví dụ: 6.4 = 6.4%)

**Tham số Output:**
- `total_closed`: Tổng số deal đã đóng (DEAL_ENTRY_OUT)
- `buy_closed`: Số deal BUY đã đóng
- `sell_closed`: Số deal SELL đã đóng
- `tp_hit`: Số lần đóng do hit TP (`DEAL_REASON_TP`)
- `sl_hit`: Số lần đóng do hit SL (`DEAL_REASON_SL`)
- `hoa_count`: Số lần đóng hòa vốn (profit ≈ 0)
- `total_volume`: Tổng volume đã giao dịch
- `backcom`: Tổng backcom = `total_volume × backcom_base / 100`

**Quy tắc lọc lịch sử:**
```cpp
if (m_use_start_time)
    HistorySelect(m_start_time, TimeCurrent());  // Chỉ load từ start_time
else
    HistorySelect(0, TimeCurrent());             // Load toàn bộ lịch sử
```

**Quy tắc tính hòa vốn:**
```cpp
double eps = base_lot + spread_value * 3.14;  // Ngưỡng hòa
if (MathAbs(profit) <= eps)
    hoa_count++;  // Coi như hòa vốn
```

**Quy trình:**
1. Load lịch sử deal theo `m_start_time` (nếu có) hoặc toàn bộ
2. Duyệt qua tất cả deal trong lịch sử
3. Lọc chỉ deal của symbol hiện tại
4. Nếu có `m_use_start_time` → bỏ qua deal trước `m_start_time`
5. Chỉ xử lý deal có `DEAL_ENTRY = DEAL_ENTRY_OUT` (lệnh đóng)
6. Phân loại theo:
   - Type: BUY/SELL
   - Reason: TP/SL/Hòa vốn
7. Tính tổng volume và backcom

**Ví dụ:**
```cpp
int closed, buy, sell, tp, sl, hoa;
double vol, backcom;
panel.GetPerformanceStats(
    closed, buy, sell, tp, sl, hoa, vol, backcom,
    0.01,    // base_lot
    20.0,    // spread_value
    6.4      // backcom_base = 6.4%
);
Print("Tổng đã đóng: ", closed, " | Volume: ", vol, " | Backcom: ", backcom);
```

---

### 6. Update() - Cập Nhật Panel

**Chức năng:** Cập nhật và hiển thị thông tin lên panel mỗi tick

**Cú pháp:**
```cpp
void Update(
    double base_lot = 0.01,      // Lot cơ bản
    double spread_value = 0.0,   // Spread value
    double backcom_base = 5.0    // Tỷ lệ backcom %
)
```

**Tham số:**
- `base_lot`: Lot size cơ bản (mặc định: 0.01)
- `spread_value`: Giá trị spread hiện tại (points)
- `backcom_base`: Tỷ lệ backcom % (mặc định: 5.0%)

**Quy trình:**
1. Kiểm tra object panel có tồn tại không, nếu không → tạo lại
2. Lấy balance hiện tại từ account
3. Gọi `CountOpenPositions()` để đếm lệnh đang mở
4. Gọi `GetPerformanceStats()` để lấy thống kê lịch sử
5. Format chuỗi text hiển thị
6. Hiển thị lên chart bằng `Comment()`

**Format hiển thị:**
```
Account: [Account ID]
Balance: [Balance]
Open BUY([Symbol]): [số lượng] | SELL([Symbol]): [số lượng]
Closed Total: [tổng] | BUY: [số BUY] | SELL: [số SELL]
Total closed Volume: [volume] | Backcom: [backcom]
```

**Ví dụ output:**
```
Account: 12345678
Balance: 10000.00
Open BUY(BTCUSD): 2 | SELL(BTCUSD): 1
Closed Total: 45 | BUY: 23 | SELL: 22
Total closed Volume: 12.50 | Backcom: 800.00
```

**Khi nào gọi:**
- Trong hàm `OnTick()` của EA
- Mỗi khi có tick mới để cập nhật real-time

**Ví dụ:**
```cpp
void OnTick() {
    double spread = GetSpreadPoints();
    panel.Update(0.01, spread, 6.4);  // Cập nhật với backcom 6.4%
}
```

---

## Luồng Hoạt Động Tổng Thể

### 1. Khởi Tạo (OnInit)
```cpp
CAccountPanel panel;

int OnInit() {
    panel.Init(0, 10, 10, clrWhite, 11);      // Tạo panel
    panel.SetStartTime(TimeCurrent());         // Set thời điểm bắt đầu
    return INIT_SUCCEEDED;
}
```

### 2. Cập Nhật Mỗi Tick (OnTick)
```cpp
void OnTick() {
    double spread = GetSpreadPoints();
    panel.Update(BASE_LOT, spread, InpBackcomRate);
}
```

### 3. Dọn Dẹp (OnDeinit)
```cpp
void OnDeinit(const int reason) {
    panel.Deinit();  // Xóa panel khỏi chart
}
```

---

## Quy Tắc Tính Toán

### 1. Ngưỡng Hòa Vốn
```cpp
eps = base_lot + spread_value × 3.14
```
- Nếu `|profit| <= eps` → coi như hòa vốn
- Hệ số 3.14 là hằng số kinh nghiệm để tính ngưỡng

### 2. Backcom
```cpp
backcom = total_volume × backcom_base / 100
```
- `total_volume`: Tổng volume đã giao dịch (lot)
- `backcom_base`: Tỷ lệ % (ví dụ: 6.4 = 6.4%)
- Kết quả: Tổng backcom nhận được (đơn vị tiền tệ account)

### 3. Lọc Lịch Sử
- **Nếu có SetStartTime():**
  - Chỉ tính deal có `deal_time >= m_start_time`
  - Loại bỏ lịch sử cũ trước khi BOT chạy
  
- **Nếu không có SetStartTime():**
  - Tính toàn bộ lịch sử từ đầu đến giờ

---

## Ưu Điểm

✅ **Tự động cập nhật:** Panel tự động refresh mỗi tick  
✅ **Lọc lịch sử thông minh:** Chỉ tính từ khi BOT bắt đầu  
✅ **Thống kê đầy đủ:** Lệnh mở, lệnh đóng, TP/SL, volume, backcom  
✅ **Dễ tích hợp:** Chỉ cần include và gọi 3 hàm chính (Init, Update, Deinit)  
✅ **Hiển thị trực quan:** Thông tin rõ ràng, dễ đọc trên chart  

---

## Lưu Ý Quan Trọng

⚠️ **Object Name Unique:** Mỗi symbol có 1 panel riêng (tên = prefix + symbol)  
⚠️ **Start Time Filter:** Nhớ gọi `SetStartTime()` nếu muốn lọc lịch sử  
⚠️ **Performance:** Hàm `GetPerformanceStats()` duyệt toàn bộ lịch sử → có thể chậm nếu lịch sử quá dài  
⚠️ **Backcom Calculation:** Công thức backcom đơn giản (volume × %), có thể cần điều chỉnh theo broker  
⚠️ **Hòa Vốn Threshold:** Công thức `eps = base_lot + spread × 3.14` là kinh nghiệm, có thể cần tune  

---

## Ví Dụ Sử Dụng Đầy Đủ

```cpp
#include <AccountPanel.mqh>

CAccountPanel panel;
input double InpBackcomRate = 6.4;  // Tỷ lệ backcom %
input color  InpPanelColor  = clrWhite;

int OnInit() {
    // Khởi tạo panel ở góc trái trên
    panel.Init(0, 10, 10, InpPanelColor, 11);
    
    // Set thời điểm bắt đầu = thời điểm EA được attach
    panel.SetStartTime(TimeCurrent());
    
    Print("AccountPanel initialized successfully");
    return INIT_SUCCEEDED;
}

void OnDeinit(const int reason) {
    // Xóa panel khi EA bị remove
    panel.Deinit();
    Comment("");  // Clear comment
}

void OnTick() {
    // Lấy spread hiện tại
    MqlTick tick;
    if(!SymbolInfoTick(_Symbol, tick)) return;
    double spread_points = (tick.ask - tick.bid) / _Point;
    
    // Cập nhật panel với base_lot = 0.01
    panel.Update(0.01, spread_points, InpBackcomRate);
}
```

---

## Tóm Tắt Các Phương Thức

| Phương Thức | Mục Đích | Khi Nào Gọi |
|-------------|----------|-------------|
| `Init()` | Tạo panel trên chart | OnInit() |
| `SetStartTime()` | Đặt mốc thời gian lọc lịch sử | OnInit() (optional) |
| `Deinit()` | Xóa panel khỏi chart | OnDeinit() |
| `CountOpenPositions()` | Đếm lệnh đang mở | Được gọi bởi Update() |
| `GetPerformanceStats()` | Thống kê lịch sử giao dịch | Được gọi bởi Update() |
| `Update()` | Cập nhật hiển thị panel | OnTick() |

---

## Kết Luận

**AccountPanel.mqh** là một công cụ hữu ích để theo dõi hiệu suất trading real-time. Class được thiết kế đơn giản, dễ sử dụng, và cung cấp đầy đủ thông tin cần thiết cho trader và EA developer.

**Tác giả:** ZoneRecoveryBOT Team  
**Phiên bản:** 1.0  
**Nền tảng:** MetaTrader 5  
**Ngôn ngữ:** MQL5
