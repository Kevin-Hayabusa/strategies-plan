# Futures Roll Direction Prediction via Basis Level

**Source**: CITIC Futures Research — 《如何利用基差捕捉跨期套利空间》
**Strategy Type**: Calendar Spread / Roll Arbitrage
**Instruments**: IF00 vs IF02 (CSI 300), IC00 vs IC02 (CSI 500)
**Frequency**: Monthly (one trade per expiry cycle)
**Status**: Ready

---

## 1. Core Idea

Around futures expiry, large open interest forces institutional holders to roll their positions
from the front-month contract to the next active contract. The direction of this roll pressure
creates a predictable pattern in the **calendar spread** (跨期价差 = front-month − back-month):

- **Deep discount** (贴水, basis ≪ 0): Short holders dominate → they must **buy near / sell far**
  to close → near contract demand rises, far contract demand falls →
  **spread widens** (front rises relative to back) → **long the spread**

- **Deep premium** (升水, basis ≫ 0): Long holders dominate → they must **sell near / buy far**
  to roll → near contract supply rises, far contract demand rises →
  **spread converges** (front falls relative to back) → **short the spread**

The signal is measured at **T-10 trading days before front-month expiry** (the roll window
T-10 → T-2 is when institutional roll activity is most concentrated).

---

## 2. Signal Construction

### 2.1 Annualized Basis (年化折溢价率)

```
basis_annualized = (futures_price - spot_price + accrued_dividends) / days_to_expiry × 365
```

- `futures_price`: front-month (IF00 or IC00) settlement/close price on signal day (T-10)
- `spot_price`: underlying index close price on the same day
- `accrued_dividends`: cumulative index dividend yield from today to expiry (annualized from index
  dividend yield; can use rolling 12-month yield scaled to remaining days / 365)
- `days_to_expiry`: calendar days from signal day to expiry date

> **Sign convention**: Negative = discount (贴水), Positive = premium (升水)

### 2.2 Adaptive Thresholds

To avoid hard-coded static thresholds, use rolling quantiles of the basis history:

```python
# Computed on T-10 day of each expiry cycle, using all available prior cycles
lower_threshold = basis_series.rolling(500_trading_days).quantile(1/3)  # deep discount line
upper_threshold = basis_series.rolling(500_trading_days).quantile(2/3)  # deep premium line
```

Signal logic:
```
basis < lower_threshold  →  long spread  (buy near IF00/IC00, sell far IF02/IC02)
basis > upper_threshold  →  short spread (sell near IF00/IC00, buy far IF02/IC02)
lower_threshold ≤ basis ≤ upper_threshold  →  no trade (neutral zone)
```

---

## 3. Trade Mechanics

### 3.1 Instruments

| Instrument | Index | Near Contract | Far Contract |
|------------|-------|---------------|--------------|
| CSI 300    | 沪深300 | IF00 (current front-month) | IF02 (second deferred) |
| CSI 500    | 中证500 | IC00 (current front-month) | IC02 (second deferred) |

> Use IF02 / IC02 (not IF01 / IC01) as the far leg: the second-deferred contract is less
> sensitive to the same roll pressure and provides a cleaner spread.

### 3.2 Entry and Exit

| Event | Action |
|-------|--------|
| T-10 (signal day) | Compute basis; if signal fires, enter spread at close (or next-day open) |
| T-2 (exit day) | Close both legs at close; 8-trading-day hold maximum |
| Sharp adverse move | Optional stop: if spread moves > 1.5× expected range, exit early |

> **T** = front-month expiry date. Standard A-share futures expiry: third Friday of the expiry month.

### 3.3 Position Sizing

- Trade one spread unit (1 near-contract, 1 far-contract, opposite sides) per signal
- Scale by portfolio notional if desired; the spread P&L is in index points × contract multiplier
  - IF multiplier: ¥300 per index point
  - IC multiplier: ¥200 per index point
- Can run IF and IC simultaneously (diversified across two indices)

---

## 4. Data Requirements

### 4.1 Daily Data (primary)

| Field | Description | Source |
|-------|-------------|--------|
| IF00, IF02 close/settlement | Front and second-deferred CSI 300 futures daily close | Wind / Bloomberg |
| IC00, IC02 close/settlement | Front and second-deferred CSI 500 futures daily close | Wind / Bloomberg |
| CSI 300 index close | Underlying for IF basis | Wind / Bloomberg |
| CSI 500 index close | Underlying for IC basis | Wind / Bloomberg |
| Index dividend yield | For accrued dividend adjustment | Wind |
| Futures expiry calendar | Third Friday of each month | Exchange calendar |

### 4.2 Derived Fields

```python
def compute_basis(futures_close, spot_close, annual_dividend_yield, days_to_expiry):
    """Annualized basis rate (年化折溢价率)."""
    accrued_div = spot_close * annual_dividend_yield * days_to_expiry / 365
    raw_basis = futures_close - spot_close + accrued_div
    return raw_basis / spot_close / days_to_expiry * 365

def get_signal_day(expiry_date, trading_calendar, offset=-10):
    """T-10 trading days before expiry."""
    return trading_calendar[trading_calendar < expiry_date].iloc[offset]

def get_exit_day(expiry_date, trading_calendar, offset=-2):
    """T-2 trading days before expiry."""
    return trading_calendar[trading_calendar < expiry_date].iloc[offset]
```

---

## 5. Backtest Setup

### 5.1 Universe and Period

| Parameter | Value |
|-----------|-------|
| Instruments | IF calendar spread, IC calendar spread |
| Backtest period | 2015–present (IC listed April 2015) |
| Signal frequency | Monthly (one evaluation per expiry cycle) |
| Hold period | ~8 trading days (T-10 to T-2) |
| Transaction costs | 0.23 per lot per side (exchange + broker), roughly 1–2 index points round-trip |

### 5.2 Return Calculation

```python
# Spread P&L for a single trade
spread_entry  = near_price_entry  - far_price_entry    # long spread: buy near, sell far
spread_exit   = near_price_exit   - far_price_exit
pnl_points    = spread_exit - spread_entry              # positive = profit for long spread

# For short spread: flip sign
# In currency terms:
pnl_if = pnl_points * 300   # IF multiplier
pnl_ic = pnl_points * 200   # IC multiplier
```

### 5.3 Performance Benchmarks (from source paper)

| Metric | IF Spread | IC Spread |
|--------|-----------|-----------|
| Win rate | 65.71% | 75.86% |
| Annualized return | 0.74% | 1.37% |
| Trades per year | ~12 (monthly) | ~12 (monthly) |
| Hold period | 8 trading days | 8 trading days |

> Note: Returns are low in absolute terms because calendar spreads have low nominal exposure.
> Capital efficiency improves significantly when combined with a directional overlay or when
> margin is recycled. The key value is the **high win rate** and **low correlation** with
> outright index positions.

---

## 6. Special Considerations

### 6.1 Snowball Effect on IC (雪球结构)

CSI 500 (IC) futures are heavily influenced by **snowball structured products** (雪球期权):

- Snowball issuers (dealers) are long knock-in puts on CSI 500 → they hedge by **shorting IC futures**
- When IC is at deep discount, it partially reflects snowball delta hedging, not just fundamental roll pressure
- This can **amplify the discount signal** (more short pressure → deeper discount → stronger long-spread signal)
- However, it also creates **jump risk**: a large CSI 500 rally can trigger mass snowball knock-out,
  forcing dealers to unwind their shorts rapidly → sudden spread compression

**Risk management response**:
- Monitor the aggregate notional of outstanding snowball products (公开数据或中国期货协会披露)
- Reduce IC spread size when estimated snowball notional is abnormally high (> 2σ above rolling mean)
- Consider tighter stop-loss on IC long-spread positions during elevated snowball periods

### 6.2 Position Limit and Liquidity

- Front-month contracts (IF00, IC00) have the highest liquidity but volume drops sharply in the
  final 2 days before expiry → exit by T-2 to avoid wide spreads
- IF02 and IC02 have lower open interest; use limit orders at entry; do not chase
- Maximum position: limited by the thinner far-month contract's daily volume

### 6.3 Dividend Season (分红季)

- May–August: A-share index constituents pay dividends → accrued dividend adjustment is material
- Failing to adjust for dividends will **understate futures discount** during this period
- Use rolling 12-month realized dividend yield (or a forward estimate) scaled to remaining days

### 6.4 Holiday Effects

- Chinese public holidays (Spring Festival, National Day) create multi-day gaps
- If T-10 falls over a holiday cluster, signal timing may shift; always use trading-day calendar,
  not calendar days, for the T-10 offset

---

## 7. Risk Filters

Apply the following filters before entering a trade:

```python
def apply_risk_filters(signal, basis_value, spread_std_30d, snowball_notional_zscore):
    """Return False to skip trade if any filter triggers."""

    # 1. Basis not extreme enough — stay in neutral zone
    if signal == 0:
        return False

    # 2. Excess volatility: if recent spread std is > 2x long-run average, skip
    if spread_std_30d > 2 * LONG_RUN_SPREAD_STD:
        return False

    # 3. Snowball risk: if snowball notional z-score > 2, halve IC position or skip
    if snowball_notional_zscore > 2.0 and signal == LONG_SPREAD_IC:
        return False  # or reduce size

    return True
```

---

## 8. Extensions and Research Questions

1. **Multi-leg**: Can we trade IF *and* IC spreads simultaneously with a combined signal?
   - Test correlation between IF and IC spread returns; if low, a combined portfolio may
     improve Sharpe
2. **Signal refinement**: Does adding **open interest imbalance** (near/far OI ratio) improve
   the basis signal?
3. **Timing within T-10 to T-2**: Is there an intra-window pattern? (e.g., most of the P&L
   comes in the last 3 days of the window)
4. **Interaction with Dispersion strategy** (Strategy #1): When IF is at deep discount *and*
   CSI 300 momentum is positive, the long-spread trade is directionally consistent with being
   long futures — could serve as a lower-risk substitute
5. **Cross-market basis**: Apply the same framework to IH (CSI 50) if data permits

---

## 9. Implementation Checklist

- [ ] Build futures expiry calendar (2015–present) with T-10 and T-2 dates
- [ ] Collect IF00, IF02, IC00, IC02 daily close + settlement prices
- [ ] Collect CSI 300 and CSI 500 index closes
- [ ] Implement `compute_basis()` with dividend adjustment
- [ ] Implement rolling 500-day quantile thresholds
- [ ] Build trade log: entry date, exit date, legs, entry spread, exit spread, P&L
- [ ] Validate signal on known historical episodes (e.g., 2015 crash — deep IC discount)
- [ ] Run full backtest 2015–present; compute win rate, annual return, max drawdown, Sharpe
- [ ] Stress-test snowball period (2021–2022) separately
- [ ] Paper trade for 3 months before live deployment

---

*Last updated: 2026-03-15*
