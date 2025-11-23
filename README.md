# ZoneRecoveryBOT

## Overview
**ZoneRecovery_v4** is a Martingale Hedge Expert Advisor (EA) for MetaTrader 5 that uses ATR-based dynamic gap and take profit calculations. The bot implements a zone recovery strategy with risk management features including early exit mechanisms and margin protection.

## Core Strategy

### Initial Entry
- **Step 1**: Always starts with a BUY order of **0.01 lot** for each new sequence
- This initial order establishes the starting point for the martingale sequence

### GAP Calculation (Fixed per Sequence)
The GAP is calculated **once** at the beginning of each sequence when the initial BUY 0.01 order is opened:

```
GAP_points = max(ATR_start × K_Gap, MinGapPoints)
```

**Parameters:**
- `ATR_start`: ATR value at the time of opening the first BUY 0.01 order
- `K_Gap` (InpGapK): Multiplier for ATR to create GAP (default: 1.0)
- `MinGapPoints` (InpMinGapPoints): Minimum GAP in points (default: 12345)

**Key Points:**
- GAP is **fixed** for the entire sequence
- GAP is reset and recalculated only when a new sequence starts (after all positions are closed)
- Uses `g_gap_initialized` flag to track if GAP has been set for current sequence

### Take Profit Calculation (Dynamic)
TP is calculated **dynamically** on every tick using current ATR:

```
TP_points = max(
    GAP + 3×spread,
    ATR_now × K_TP + 3×spread,
    MinTPPoints
)
```

**Parameters:**
- `ATR_now`: Current ATR value (updates every tick)
- `K_TP` (InpTPK): Multiplier for ATR to calculate TP (default: 1.28)
- `MinTPPoints` (InpMinTPPoints): Minimum TP in points (default: 16868)
- `spread`: Current spread in points

**TP Logic:**
- For **BUY** positions: `TP_level = entry_price + TP_points × Point`
  - Closes when `BID >= TP_level`
- For **SELL** positions: `TP_level = entry_price - TP_points × Point`
  - Closes when `ASK <= TP_level`

### Mid-Price Trigger System
The bot uses **mid-price** to trigger new orders, reducing grid distortion caused by spread:

```
mid_price = (BID + ASK) / 2
```

**Trigger Logic:**
- If last order is **BUY**:
  - Trigger SELL when: `mid_price <= last_price - GAP_points × Point`
- If last order is **SELL**:
  - Trigger BUY when: `mid_price >= last_price + GAP_points × Point`

### Martingale Lot Sizing
Each new order doubles the previous lot size:

```
next_lot = last_volume × 2.0
```

**Sequence Example:**
1. BUY 0.01
2. SELL 0.02
3. BUY 0.04
4. SELL 0.08
5. BUY 0.16
... and so on

## ATR Configuration

### Symbol Profile Auto-Detection
The bot automatically selects appropriate ATR timeframes based on the symbol:

| Symbol Type | ATR Timeframe | Detection |
|-------------|---------------|-----------|
| **BTC** | M15 | Symbol name contains "BTC" |
| **XAU** (Gold) | M5 | Symbol name contains "XAU" |
| **Other** (FX) | User Input | Default from InpATRTimeframe |

**Parameters:**
- `InpAutoSymbolProfile`: Enable/disable auto-detection (default: true)
- `InpATRPeriod`: ATR period (default: 14)
- `InpATRTimeframe`: Default ATR timeframe if auto-detection is disabled (default: M5)

## Risk Management

### 1. Early Exit by Lot Size
Activates when the largest position reaches a threshold lot size:

**Trigger Condition:**
```
max_volume >= EarlyExitLotThreshold
```

**Exit Condition:**
```
total_PNL > 0 AND total_PNL > (spread_cost × EarlyExitPNLFactor)
```

**Parameters:**
- `InpEarlyExitLotThreshold`: Lot size threshold (default: 1.28)
- `InpEarlyExitPNLFactor`: PNL factor for early exit (default: 0.68)

**Calculation:**
```
spread_cost = total_volume × spread_points × Point / tick_size × tick_value
early_exit_threshold = spread_cost × EarlyExitPNLFactor
```

**Purpose:**
- Exits risky positions early when in profit
- Prevents excessive drawdown from large lot sizes
- Reduces PNL requirement when lot size is dangerous

### 2. Margin Protection
The `CheckMarginAndCloseIfSafe()` function provides a second layer of protection:
- Called on every tick
- Called after every new order is opened
- Monitors margin levels
- Works in conjunction with early exit logic

## Input Parameters

### Core Strategy Parameters
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `InpGapK` | double | 1.0 | K_Gap: Multiplier for ATR_start to create GAP |
| `InpTPK` | double | 1.28 | K_TP: Multiplier for ATR_now to calculate TP |
| `InpMinGapPoints` | double | 12345 | Minimum GAP in points |
| `InpMinTPPoints` | double | 16868 | Minimum TP in points |
| `InpMagicNumber` | ulong | 9999 | Magic number for EA identification |

### ATR & Symbol Profile
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `InpATRPeriod` | int | 14 | ATR period |
| `InpATRTimeframe` | ENUM_TIMEFRAMES | PERIOD_M5 | Default ATR timeframe |
| `InpAutoSymbolProfile` | bool | true | Auto-select profile based on symbol |

### Early Exit Parameters
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `InpEarlyExitLotThreshold` | double | 1.28 | Lot size threshold for early exit |
| `InpEarlyExitPNLFactor` | double | 0.68 | PNL factor for early exit calculation |

### Display & Accounting
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `InpPanelColor` | color | clrWhite | Panel text color |
| `InpBackcomRate` | double | 6.4 | Backcom rate % (for panel display) |

### Bot Start Time
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `InpUseCustomStartTime` | bool | false | Use custom start time |
| `InpCustomStartTime` | datetime | 2025.11.22 00:00 | Custom start time if enabled |

## Trading Flow

### Sequence Start
1. No open positions → Open BUY 0.01
2. Calculate GAP from current ATR: `GAP = max(ATR × K_Gap, MinGap)`
3. Set `g_gap_initialized = true`

### During Sequence
1. **Every Tick:**
   - Calculate dynamic TP from current ATR
   - Check if TP hit → Close all positions if true
   - Calculate mid-price
   - Check if mid-price crossed GAP threshold
   - If crossed → Open opposite direction with doubled lot size
   - Run margin check and early exit logic

2. **After Opening New Order:**
   - Call `CheckMarginAndCloseIfSafe()`
   - Monitor for early exit conditions

### Sequence End
1. TP is hit → Close all positions
2. Reset `g_gap_initialized = false`
3. Wait for next sequence to start

## Key Features

### ✅ Advantages
- **Dynamic TP**: Adapts to market volatility using current ATR
- **Fixed GAP**: Consistent grid spacing per sequence
- **Mid-Price Trigger**: Reduces spread impact on grid accuracy
- **Symbol-Aware**: Auto-adjusts ATR timeframe for different instruments
- **Early Exit**: Protects against excessive drawdown
- **Margin Protection**: Multiple layers of risk management
- **Visual Panel**: Real-time account statistics display

### ⚠️ Risk Warnings
- **Martingale Strategy**: Lot sizes grow exponentially
- **No Stop Loss**: Relies on TP and early exit only
- **Requires Adequate Margin**: Can use significant margin in deep sequences
- **Spread Sensitive**: High spread can affect profitability
- **Not Suitable for All Market Conditions**: Works best in ranging/oscillating markets

## Technical Implementation

### Global Variables
- `g_atr_handle`: ATR indicator handle
- `g_atr_tf`: Selected ATR timeframe
- `g_gap_points`: Fixed GAP for current sequence
- `g_gap_initialized`: Flag indicating if GAP is set
- `g_last_atr_points`: Last ATR value for logging
- `g_bot_start_time`: Bot start timestamp

### Main Functions
- `OnInit()`: Initialize EA, ATR indicator, and panel
- `OnTick()`: Main tick handler - calls ProcessLogic() and CheckMarginAndCloseIfSafe()
- `OnDeinit()`: Cleanup resources
- `ProcessLogic()`: Core trading logic (TP check, grid management, order opening)
- `CheckMarginAndCloseIfSafe()`: Risk management and early exit
- `SetupSymbolProfile()`: Auto-detect symbol type and set ATR timeframe
- `InitGapFromAtrOnFirstOrder()`: Initialize GAP for new sequence
- `CalcDynamicTPPoints()`: Calculate dynamic TP
- `OpenInitialBuy()`: Open first BUY 0.01 order
- `GetLastOrder()`: Get most recent order information
- `CloseAllPositions()`: Close all EA positions
- `CountOpenPositions()`: Count EA's open positions
- `GetAtrPoints()`: Get current ATR in points
- `GetSpreadPoints()`: Get current spread in points

## Version History

### Version 4 (Current)
- ✨ Fixed GAP calculation from ATR_start at sequence beginning
- ✨ Dynamic TP using current ATR_now
- ✨ Mid-price trigger system to reduce spread impact
- ✨ Early exit based on lot size threshold
- ✨ Symbol profile auto-detection (BTC/XAU/FX)
- ✨ Enhanced margin protection
- ✨ Integrated account statistics panel

## Usage Recommendations

### Optimal Settings
1. **For BTC**: Auto-profile will use M15 ATR
2. **For XAU (Gold)**: Auto-profile will use M5 ATR
3. **For FX Pairs**: Manually set appropriate ATR timeframe

### Risk Management
- Start with adequate account balance (recommended: enough for 8-10 martingale steps)
- Monitor the early exit lot threshold based on your risk tolerance
- Adjust K_Gap and K_TP based on backtesting results
- Consider spread costs in your profitability calculations

### Monitoring
- Watch the account panel for real-time statistics
- Monitor ATR values to understand current market volatility
- Track the largest lot size in active sequences
- Be aware of margin usage during deep sequences

## License & Disclaimer

This EA is provided for educational and trading purposes. Trading involves substantial risk of loss. Past performance does not guarantee future results. Use at your own risk.

---

**Author**: ZoneRecovery_v1
**Platform**: MetaTrader 5  
**Strategy**: Martingale Hedge with ATR-based GAP and Dynamic TP