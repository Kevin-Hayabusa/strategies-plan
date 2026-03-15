# Index Money Flow — APAC Cross-Country Long/Short Strategy — Backtest Implementation Plan

## Executive Summary

This strategy extends the **主动成交占比 (Active Trading Ratio)** factor from Changjiang Securities
*"高频因子（七）：分布估计下的主动成交占比"* (2020-08-10) — which was designed for individual A-share
stock selection — to a **cross-country index long/short strategy across APAC**.

**Core Innovation**: Instead of ranking stocks, we aggregate constituent-level 博弈因子 to the
index level, then use the resulting country-level signal to long/short APAC equity indices.

**Universe**: HSI (Hong Kong), CSI 300 (China), KOSPI 200 (South Korea), NKY (Japan), TAIEX (Taiwan)
**Execution**: Index futures or liquid ETFs
**Data Source**: Bloomberg Terminal (xbbg) — intraday bar data
**Signal Frequency**: Daily factor, weekly rebalancing
**Strategy Type**: Cross-sectional long/short on country indices
**Status**: Ready for Implementation

---

## 1. Strategy Theory

### 1.1 Original 博弈因子 (recap from paper)

The original **博弈因子** captures the balance of buy-side vs sell-side driving force in a stock:

```
博弈因子 = ΣVol_Buy / ΣVol_Sell
```

Where `Vol_Buy` is identified using tick data (current price > previous bid-1). This requires
L2 tick data, which is hard to obtain across APAC. The paper's improved approach — the
**分布估计主动占比因子** — replaces tick classification with a continuous distribution-based
mapping using only bar-level OHLCV data, making it **cross-market compatible**.

### 1.2 Distribution-Based Active Ratio (Key Innovation from Paper)

For each intraday bar `i`, the **active buy amount** is estimated by mapping the return to a
probability using a CDF:

```
Active_Buy_Amount_i = Amount_i × F(ret_i)
Active_Sell_Amount_i = Amount_i - Active_Buy_Amount_i

Stock_博弈因子 = Σ Active_Buy_Amount_i / Σ Amount_i
```

Where `F()` is a monotone CDF satisfying:
1. F(x) ∈ [0, 1] — buy ratio bounded
2. F'(x) > 0 — larger return → higher buy ratio
3. F(x) + F(-x) = 1 — symmetric: buy/sell characterization consistent

**Best performer from paper**: 置信正态分布 (Confidence Normal) with uniform distribution:

```python
# Confidence Normal (for markets with ±10% price limits, e.g. A-shares, Taiwan):
Active_Buy_i = Amount_i × N(ret_i / 0.1 × 1.96)

# Standardized Normal (for markets without hard limits, e.g. Japan, HK):
Active_Buy_i = Amount_i × N(ret_i / σ_ret)

# Uniform (for markets with ±X% price limits):
Active_Buy_i = Amount_i × (ret_i - (-limit)) / (2 × limit)
```

### 1.3 From Stock to Index Level

**Key extension**: Rather than using the factor to rank stocks, we aggregate constituent-level
signals to produce a single **country-level 博弈因子**:

```
Index_博弈因子_j = Σ_i w_i × Stock_博弈因子_i
```

Where:
- `i` = constituent stock of index `j`
- `w_i` = market-cap weight (or equal weight)
- The aggregation is done daily at a fixed observation time (market close)

**Economic intuition**:
- **High index 博弈因子** (>0.5): Active buying > selling on aggregate → bullish sentiment,
  prices may continue upward (or already overextended — check for reversion)
- **Low index 博弈因子** (<0.5): Active selling dominates → bearish sentiment → downward pressure

### 1.4 Cross-Country Signal Construction

At end of each week, for each country `j`:

```python
# Rolling N-week average of daily index 博弈因子
factor_j = mean(Index_博弈因子_j[-N_days:])  # e.g. 4 weeks = 20 trading days

# Cross-sectional z-score across all countries
z_j = (factor_j - mean(factor_all)) / std(factor_all)

# Signal: long top-k countries, short bottom-k countries
position_j = +1 if rank(z_j) >= top_k
position_j = -1 if rank(z_j) <= bottom_k
position_j =  0 otherwise
```

**Alternative signals to test**:
1. Raw factor level (is country above/below its historical mean?)
2. Factor momentum (is the factor rising or falling?)
3. Factor change (weekly delta of index 博弈因子)
4. Percentile rank vs own 60-day history

---

## 2. Universe & Market Details

### 2.1 Index Universe

| Index | Country | Ticker (Bloomberg) | Futures | ETF | Constituents | Price Limit |
|-------|---------|-------------------|---------|-----|--------------|-------------|
| HSI | Hong Kong | HSI Index | HI1 Index (front) | EWH US / 2800 HK | ~80 | None |
| CSI 300 | China A | SHSZ300 Index | IFc1 Index | 510300 CH | 300 | ±10% |
| KOSPI 200 | South Korea | KOSPI2 Index | KM1 Index | EWY US | 200 | ±30% |
| NKY | Japan | NKY Index | NK1 Index | EWJ US / 1570 JP | 225 | None* |
| TAIEX | Taiwan | TWSE Index | TW1 Index | EWT US | ~900 | ±10% |

*Japan has dynamic circuit breakers but no fixed daily price limits

**Optional additions** (if data available):
- ASX 200 (Australia): AS51 Index, ±None
- STI (Singapore): STI Index, ±None

### 2.2 Market-Specific Distribution Parameters

The CDF normalization must adapt to each market's return distribution:

| Market | Method | Formula | Rationale |
|--------|--------|---------|-----------|
| China A (CSI 300) | Confidence Normal | `N(ret / 0.1 × 1.96)` | ±10% hard price limit |
| Taiwan (TAIEX) | Confidence Normal | `N(ret / 0.1 × 1.96)` | ±10% hard price limit |
| South Korea (KOSPI) | Confidence Normal | `N(ret / 0.3 × 1.96)` | ±30% hard price limit |
| Hong Kong (HSI) | Standardized Normal | `N(ret / σ_ret)` | No price limit |
| Japan (NKY) | Standardized Normal | `N(ret / σ_ret)` | No fixed limit |

**Uniform distribution alternative** (simpler, comparable performance):

| Market | Formula |
|--------|---------|
| China A, Taiwan | `(ret + 0.10) / 0.20` capped to [0,1] |
| Korea | `(ret + 0.30) / 0.60` capped to [0,1] |
| HK, Japan | `(ret + 3σ) / 6σ` capped to [0,1] |

### 2.3 Trading Hour Alignment

A key challenge: markets trade at different UTC times. To ensure comparability,
**use the full regular session** for each market:

| Market | Session (local) | UTC offset | Data collection point |
|--------|----------------|------------|----------------------|
| China A | 09:30–15:00 CST | UTC+8 | 15:00 CST |
| Hong Kong | 09:30–16:00 HKT | UTC+8 | 16:00 HKT |
| Taiwan | 09:00–13:30 CST | UTC+8 | 13:30 CST |
| South Korea | 09:00–15:30 KST | UTC+9 | 15:30 KST |
| Japan | 09:00–15:30 JST | UTC+9 | 15:30 JST |

→ Signal is computed **after each market's own close**, using that day's full session data.
→ Execution is at **next trading day's open** for each respective market.

---

## 3. Data Requirements

### 3.1 Bloomberg Data Fields

#### Intraday Constituent Bar Data (core input)

```python
# 5-minute or 10-minute bars for each constituent
# (1-min bars preferable but data volume is large)

from xbbg import blp

# Example: download intraday bars for a constituent
data = blp.bdib(
    ticker="700 HK Equity",    # Tencent (HSI constituent)
    dt="2023-01-15",
    session="day",
    bar_size=5                 # 5-minute bars
)
# Returns: open, high, low, close, volume, num_ticks
```

#### Constituent Lists (historical)

```python
# Get historical constituent list at a specific date
constituents = blp.bds(
    tickers="HSI Index",
    flds="INDX_MEMBERS",
    INDX_MWEIGHT_HIST_DT="20230101"
)
# Returns: member tickers + weights
```

#### Index Futures / ETF Data (for backtest execution)

```python
# Front-month continuous futures
futures_map = {
    "HSI":    "HI1 Index",
    "CSI300": "IFc1 Index",
    "KOSPI":  "KM1 Index",
    "NKY":    "NK1 Index",
    "TAIEX":  "TW1 Index",
}

# Daily OHLCV for backtesting
index_data = blp.bdh(
    tickers=list(futures_map.values()),
    flds=["PX_OPEN", "PX_HIGH", "PX_LOW", "PX_LAST", "PX_VOLUME"],
    start_date="20100101",
    end_date="20241231"
)
```

### 3.2 Data Volume Estimate

| Index | Constituents | Bars/day (5-min) | Stocks × Days × 10yr |
|-------|-------------|------------------|----------------------|
| HSI | ~80 | ~78 | ~15M rows |
| CSI 300 | 300 | ~48 | ~36M rows |
| KOSPI 200 | 200 | ~78 | ~31M rows |
| NKY | 225 | ~78 | ~35M rows |
| TAIEX | ~900 | ~54 | ~98M rows |

→ **Total**: ~215M rows. Use **parquet format with date partitioning**.
→ Consider using **30-minute bars** to reduce volume 6× with minimal signal loss.

### 3.3 Data Handling Challenges

| Challenge | Solution |
|-----------|----------|
| Suspended stocks | Exclude from factor calculation on suspension day |
| Halted trading sessions | Skip entire day's factor; use previous day's value |
| Constituent changes | Use historical constituent lists (quarterly updates) |
| Dividends / splits | Use adjusted close prices for return calculation |
| Missing bars | Forward-fill within session; zero-volume bars excluded |
| FX conversion | Report in USD-equivalent for cross-country comparison |

---

## 4. Factor Construction

### 4.1 Step-by-Step: Daily Index 博弈因子

```python
import numpy as np
import pandas as pd
from scipy.stats import norm

def calc_active_buy_ratio(ret: float, method: str, params: dict) -> float:
    """Map intraday bar return to active buy proportion."""
    if method == "confidence_normal":
        limit = params["price_limit"]       # e.g. 0.10 for A-shares
        return norm.cdf(ret / limit * 1.96)
    elif method == "standardized_normal":
        sigma = params["ret_std"]           # rolling std of returns
        return norm.cdf(ret / sigma) if sigma > 0 else 0.5
    elif method == "uniform":
        limit = params["price_limit"]
        return np.clip((ret + limit) / (2 * limit), 0, 1)
    else:
        raise ValueError(f"Unknown method: {method}")


def calc_stock_game_factor(bars: pd.DataFrame, method: str, params: dict) -> float:
    """
    Calculate 博弈因子 for a single stock on a single day.

    Args:
        bars: DataFrame with columns [open, high, low, close, volume, amount]
              indexed by intraday timestamp
        method: 'confidence_normal', 'standardized_normal', or 'uniform'
        params: method-specific parameters

    Returns:
        float: Active buy ratio in [0, 1]
    """
    if len(bars) < 2:
        return np.nan

    # Calculate bar returns
    bars = bars.copy()
    bars['ret'] = bars['close'].pct_change().fillna(0)

    # For standardized methods: compute sigma over the day's bars
    if method == "standardized_normal":
        params = params.copy()
        params['ret_std'] = bars['ret'].std()

    # Active buy amount for each bar
    bars['active_buy_ratio'] = bars['ret'].apply(
        lambda r: calc_active_buy_ratio(r, method, params)
    )

    total_amount = bars['amount'].sum()
    if total_amount == 0:
        return np.nan

    active_buy_amount = (bars['active_buy_ratio'] * bars['amount']).sum()
    return active_buy_amount / total_amount


def calc_index_game_factor(
    constituent_bars: dict,       # {ticker: daily_bars_df}
    weights: pd.Series,           # cap weights, indexed by ticker
    method: str,
    params: dict,
    weighting: str = "cap"        # 'cap' or 'equal'
) -> float:
    """
    Aggregate constituent 博弈因子 to index level.

    Args:
        constituent_bars: dict of {ticker: DataFrame of intraday bars for today}
        weights: market cap weights
        method: distribution method
        params: method params
        weighting: 'cap' (market-cap weighted) or 'equal'

    Returns:
        float: Index-level active buy ratio in [0, 1]
    """
    stock_factors = {}
    for ticker, bars in constituent_bars.items():
        factor = calc_stock_game_factor(bars, method, params)
        if not np.isnan(factor):
            stock_factors[ticker] = factor

    if not stock_factors:
        return np.nan

    factors = pd.Series(stock_factors)

    if weighting == "equal":
        return factors.mean()
    else:  # cap-weighted
        # Align weights with available stocks
        w = weights[factors.index]
        w = w / w.sum()  # Renormalize after excluding missing
        return (factors * w).sum()
```

### 4.2 Weekly Signal Construction

```python
def build_weekly_signal(
    daily_factors: pd.DataFrame,   # columns = country indices, index = dates
    lookback_weeks: int = 4,
    signal_type: str = "level"     # 'level', 'momentum', 'change'
) -> pd.DataFrame:
    """
    Build weekly cross-sectional signal from daily index 博弈因子.

    Returns:
        DataFrame of weekly signals (z-scores), index = Fridays
    """
    # Resample to weekly (Friday)
    weekly_factors = daily_factors.resample('W-FRI').mean()

    lookback = lookback_weeks

    if signal_type == "level":
        # Raw rolling mean
        signal = weekly_factors.rolling(lookback).mean()

    elif signal_type == "momentum":
        # Factor momentum: current 4-week avg vs previous 4-week avg
        current = weekly_factors.rolling(lookback).mean()
        previous = weekly_factors.rolling(lookback).mean().shift(lookback)
        signal = current - previous

    elif signal_type == "change":
        # Week-over-week change
        signal = weekly_factors.diff(1)

    elif signal_type == "percentile":
        # Rolling percentile rank vs own history
        signal = weekly_factors.rolling(52).rank(pct=True)

    # Cross-sectional z-score at each date
    signal_z = signal.sub(signal.mean(axis=1), axis=0)
    signal_z = signal_z.div(signal_z.std(axis=1), axis=0)

    return signal_z


def generate_positions(
    signal: pd.DataFrame,
    long_k: int = 2,
    short_k: int = 2,
    dollar_neutral: bool = True
) -> pd.DataFrame:
    """
    Generate long/short positions from cross-sectional signal.

    long top-k, short bottom-k, equal dollar weight
    """
    positions = pd.DataFrame(0.0, index=signal.index, columns=signal.columns)

    for date, row in signal.iterrows():
        valid = row.dropna()
        if len(valid) < long_k + short_k:
            continue

        ranked = valid.rank(ascending=False)

        # Long top-k (lowest 博弈因子 rank = most active selling = short)
        # Note: LOW factor = more selling → bearish → SHORT
        # HIGH factor = more buying → bullish → LONG
        long_idx = ranked[ranked <= long_k].index
        short_idx = ranked[ranked > len(ranked) - short_k].index

        # Equal dollar weight
        long_weight = 1.0 / long_k
        short_weight = -1.0 / short_k

        if dollar_neutral:
            positions.loc[date, long_idx] = long_weight
            positions.loc[date, short_idx] = short_weight
        else:
            positions.loc[date, long_idx] = long_weight

    return positions
```

---

## 5. Backtest Design

### 5.1 Execution Model

```
Signal date:   Friday close (after each market's own close)
Trade entry:   Following Monday open
Trade exit:    Following Friday close (weekly hold)
Rebalance:     Every Friday → execute Monday

For each country:
  - Execute via front-month index futures (roll 5 days before expiry)
  - OR via liquid ETF (lower cost, no roll, slight tracking error)
```

### 5.2 Transaction Costs

| Market | Vehicle | Estimated Round-Trip Cost |
|--------|---------|--------------------------|
| HSI | HSI Futures (HHI) | 0.10% |
| CSI 300 | IF Futures | 0.08% (pre-2015) / 0.20% (post-2015) |
| KOSPI 200 | KOSPI Futures | 0.10% |
| NKY | Nikkei Futures (OSE) | 0.08% |
| TAIEX | TAIEX Futures | 0.10% |
| (ETF alt.) | All ETFs (EWH/EWY/EWJ/EWT/FXI) | 0.15% + spread |

### 5.3 Currency Handling

Two approaches:
1. **USD-denominated returns**: Convert daily index returns to USD using spot FX (accounts for FX impact)
2. **Local-currency returns**: Ignore FX risk (measures pure equity effect)

→ Test both. If signal survives in both, it's genuine equity signal not FX-contaminated.

```python
# USD conversion
index_returns_usd = index_returns_local + fx_returns  # additive approximation
# fx_returns: USDCNH, USDHKD (fixed), USDKRW, USDJPY, USDTWD
```

### 5.4 Test Periods

| Period | Label | Notes |
|--------|-------|-------|
| 2005–2009 | Pre-GFC + GFC | Stress test |
| 2010–2019 | In-sample | Parameter optimization |
| 2020–2024 | Out-of-sample | COVID, rate cycle |
| 2010–2024 | Full | Overall assessment |

### 5.5 Performance Metrics

Matching style of original paper:

| Metric | Target |
|--------|--------|
| Annual return (L/S) | > 10% |
| Max drawdown | < -20% |
| Sharpe ratio | > 0.8 |
| IC (factor vs next-week return) | Negative (lower factor → better future return) |
| ICIR | < -0.4 |
| Win rate (weekly) | > 50% |

---

## 6. Implementation Roadmap

### Phase 1: Data Infrastructure (Week 1–2)

#### 6.1 Multi-Market Data Downloader
**File**: `src/apac_game_factor/data_downloader.py`

```python
MARKET_CONFIG = {
    "CSI300": {
        "index_ticker":   "SHSZ300 Index",
        "futures_ticker": "IFc1 Index",
        "method":         "confidence_normal",
        "params":         {"price_limit": 0.10},
        "bar_size":       30,          # 30-min bars
        "session":        "day",
        "rebalance_months": [3, 6, 9, 12],
    },
    "HSI": {
        "index_ticker":   "HSI Index",
        "futures_ticker": "HI1 Index",
        "method":         "standardized_normal",
        "params":         {},          # sigma computed from data
        "bar_size":       30,
        "session":        "day",
        "rebalance_months": [3, 6, 9, 12],
    },
    "KOSPI200": {
        "index_ticker":   "KOSPI2 Index",
        "futures_ticker": "KM1 Index",
        "method":         "confidence_normal",
        "params":         {"price_limit": 0.30},
        "bar_size":       30,
        "session":        "day",
        "rebalance_months": [3, 6, 9, 12],
    },
    "NKY": {
        "index_ticker":   "NKY Index",
        "futures_ticker": "NK1 Index",
        "method":         "standardized_normal",
        "params":         {},
        "bar_size":       30,
        "session":        "day",
        "rebalance_months": [3, 6, 9, 12],
    },
    "TAIEX": {
        "index_ticker":   "TWSE Index",
        "futures_ticker": "TW1 Index",
        "method":         "confidence_normal",
        "params":         {"price_limit": 0.10},
        "bar_size":       30,
        "session":        "day",
        "rebalance_months": [3, 6, 9, 12],
    },
}
```

### Phase 2: Factor Calculation Engine (Week 2–3)

#### 6.2 Factor Pipeline
**File**: `src/apac_game_factor/factor_engine.py`

```
For each trading day:
  For each index in universe:
    1. Load constituent list for that date (historical)
    2. Load intraday bars for each constituent
    3. Compute Stock_博弈因子 for each constituent
    4. Aggregate to Index_博弈因子 (cap-weighted)
    5. Store daily factor series

Output: panel of shape (trading_days × 5_countries)
```

### Phase 3: Signal & Backtest (Week 3–4)

#### 6.3 Signal Construction
**File**: `src/apac_game_factor/signal.py`

Parameters to optimize (in-sample 2010–2019):
- `lookback_weeks`: [2, 4, 6, 8]
- `signal_type`: ['level', 'momentum', 'change', 'percentile']
- `long_k`: [1, 2]
- `short_k`: [1, 2]
- `weighting`: ['cap', 'equal']
- `method`: ['confidence_normal', 'uniform', 'standardized_normal']

#### 6.4 Backtester
**File**: `src/apac_game_factor/backtester.py`

```python
class APACGameFactorBacktester:
    def __init__(self, initial_capital=10_000_000):  # USD 10M
        self.capital = initial_capital

    def run(self, positions, returns, costs):
        """Weekly rebalancing backtest with transaction costs."""
        # positions: DataFrame (Friday dates × countries)
        # returns: DataFrame (weekly returns by country)
        # costs: dict of round-trip costs by country
        ...

    def calculate_performance(self, results):
        return {
            "annual_return":   ...,
            "max_drawdown":    ...,
            "sharpe_ratio":    ...,
            "ic":              ...,
            "icir":            ...,
            "win_rate":        ...,
            "avg_weekly_return": ...,
        }
```

### Phase 4: Analysis (Week 4–5)

1. **Factor validation**: Does index-level 博弈因子 correlate with next-week index returns?
2. **Country profiles**: Which countries show strongest factor predictability?
3. **Drawdown analysis**: Does strategy protect during crashes (GFC 2008, COVID 2020)?
4. **Style neutrality**: Correlation with value/momentum/size factors at country level
5. **Out-of-sample**: Performance on 2020–2024 holdout

---

## 7. Hypotheses to Test

### H1: Factor Validity
**Does index-level 博弈因子 predict next-week index returns cross-sectionally?**
- Expected: IC < 0 (low active buying → downward pressure → negative return)
- Null: IC ≈ 0 (no predictability)

### H2: Aggregation Method
**Does cap-weighted aggregation outperform equal-weighted?**
- Large-cap stocks should drive index movement more
- Expected: cap-weighted performs better

### H3: Distribution Method
**Which CDF estimator (normal/uniform/t) works best cross-market?**
- Paper finds: confidence normal ≈ uniform > standardized normal > t-distribution
- For cross-market application: standardized normal may work better for HK/Japan

### H4: Signal Type
**Is factor level (absolute) or factor change (relative) more informative?**
- Level: captures persistent bull/bear markets
- Change: captures turning points → potentially higher turnover/cost

### H5: Market Hours Effect
**Does using same-day vs next-day signal matter?**
- Because Asian markets overlap partially, there may be information leakage
- Test: compute factor using T-1 data to avoid any look-ahead

### H6: Factor Decay
**How quickly does the signal decay?**
- Test IC at horizons: 1-day, 1-week, 2-week, 4-week
- Expected: signal strongest at 1-week, decays by 4-week

---

## 8. Risk Considerations

### 8.1 Structural Risks
- **China A-shares policy changes**: Futures restrictions post-2015-09 materially impact CSI 300
- **Market closures**: HK protest disruptions (2019), COVID (2020), Taiwan strait tensions
- **Capital controls**: China A-shares have inflow/outflow restrictions affecting cross-market arb

### 8.2 Data Risks
- **Constituent data completeness**: Historical constituent lists may be incomplete for some markets
- **Intraday data gaps**: Bloomberg may have missing bars, especially for smaller constituents
- **Survivorship bias**: Must include delisted stocks in historical factor calculations
- **Bar size**: Different optimal bar sizes across markets (A-shares 30-min vs HK 5-min)

### 8.3 Factor Degradation
The original paper (2020) already notes that performance weakened post-2017 in A-shares.
For cross-market application:
- Factor may be less powerful in markets with higher HFT presence (Japan)
- Factor likely stronger in less efficient markets with more retail participation (Taiwan, Korea)

### 8.4 Execution Risks
- **Futures roll**: Monthly futures contract roll needs to be modeled carefully
- **Basis risk**: Futures vs spot divergence during stress periods
- **Low liquidity**: TAIEX futures thinner than HSI/NKY

---

## 9. Expected Deliverables

### 9.1 Code Structure

```
strategies-impl/
└── apac_game_factor/
    ├── src/
    │   └── apac_game_factor/
    │       ├── data_downloader.py       # Multi-market Bloomberg data
    │       ├── factor_engine.py         # Stock + index 博弈因子 calculation
    │       ├── signal.py                # Weekly signal construction
    │       ├── backtester.py            # Backtest engine
    │       └── analytics.py            # Performance metrics + plots
    ├── config/
    │   └── market_config.yaml           # Per-market parameters
    ├── notebooks/
    │   ├── 01_factor_validation.ipynb   # IC analysis per country
    │   ├── 02_signal_optimization.ipynb # Parameter sweep
    │   ├── 03_backtest_results.ipynb    # Full backtest
    │   └── 04_risk_analysis.ipynb       # Style attribution, drawdowns
    ├── backtest_apac_game_factor.py     # Main script
    └── results/
        ├── factor_panel.parquet
        ├── backtest_results.csv
        └── plots/
```

### 9.2 Key Outputs

1. **Factor IC heatmap**: IC by country × year (validate factor universality)
2. **Country factor time series**: Plot of index 博弈因子 over time, overlaid with index return
3. **L/S equity curve**: USD-denominated cumulative return
4. **Drawdown chart**: Strategy vs buy-and-hold basket
5. **Parameter sensitivity**: Heatmap of Sharpe ratio across lookback × signal_type
6. **Style attribution**: Correlation with country momentum, value, carry factors

---

## 10. Comparison with Original Paper

| Dimension | Original (Paper) | This Strategy |
|-----------|-----------------|---------------|
| Universe | A-share individual stocks | 5 APAC country indices |
| Signal use | Cross-sectional stock rank | Cross-country index rank |
| Holding period | Daily (long IC period) | Weekly |
| Execution | Stock portfolio | Index futures / ETFs |
| Markets | China A-shares only | China, HK, Korea, Japan, Taiwan |
| Factor type | Absolute stock factor | Aggregated index factor |
| Return source | Stock alpha | Country allocation |
| FX exposure | None | Multi-currency (hedge or not) |

---

## 11. Future Extensions

### 11.1 Sector-Level Application
Aggregate 博弈因子 within sectors (not just indices) → sector rotation strategy within each country

### 11.2 Combination with Other Signals
- **Dispersion factor** (from companion plan): low dispersion + high 博弈因子 = strong trend signal
- **Consistency factor**: PCA-based consistency as a secondary filter

### 11.3 Intraday Timing
Rather than weekly rebalancing, use real-time index 博弈因子 for intraday entry timing:
- Similar to the consistency strategy, trade index futures when intraday 博弈因子 crosses threshold

### 11.4 Extended Universe
- Add India (SENSEX / Nifty), Australia (ASX 200), Singapore (STI)
- Potentially include emerging markets: Indonesia, Thailand

---

## 12. References

1. **Primary Paper**: "高频因子（七）：分布估计下的主动成交占比" — Changjiang Securities (长江证券), 覃川桃 & 郑起, 2020-08-10
2. **Related (prior work in paper)**:
   - "高频因子（五）：高频因子和交易行为" — original 博弈因子
   - "高频因子（二）：结构化反转因子" — structural reversal factor
   - "如何利用负面因子做指数增强？——高频因子篇" — 资金流向因子
3. **Companion Plans** in this repo:
   - `../dispersion/backtest_plan.md` — weekly dispersion momentum (Guosen, 2017)
   - `../consistency/backtest_plan.md` — intraday PCA consistency (GF Securities, 2016)
4. **Data**: Bloomberg Terminal (xbbg library)
5. **Execution reference**: Eurex, HKEX, KRX, OSE, TAIFEX futures specifications

---

## Appendix A: Market-Specific Notes

### A-shares (CSI 300)
- T+1 settlement, no intraday short selling on stocks
- Post-2015-09-07: IF futures restricted (use ETF 510300 instead)
- Science & Tech board stocks (STAR): ±20% limit
- Constituent data: SHSZ300 Index `INDX_MEMBERS` with `INDX_MWEIGHT_HIST_DT`

### Hong Kong (HSI)
- Stocks may be halted mid-day (results announcements)
- ~40 stocks account for >80% of weight — concentrated index
- Short selling allowed, futures liquid

### South Korea (KOSPI 200)
- Foreign ownership limits may affect some constituents' data
- ±30% daily limit since June 2015 (was ±15% before)
- Futures require margin in KRW — FX risk on margin

### Japan (Nikkei 225)
- Price-weighted index (unusual) — Toyota has more weight by stock price, not market cap
- For **constituent-level factor**: use equal weight (price weighting distorts signal)
- Morning session (09:00–11:30) + afternoon session (12:30–15:30) — lunch break gap
- Large intraday gaps at lunch: treat as two sub-sessions or ignore lunch bar

### Taiwan (TAIEX)
- ~900 constituents but top 50 = 70% weight → focus on top 100 for efficiency
- Very retail-heavy market — 博弈因子 effect may be stronger
- Half-day trading on some days — handle incomplete sessions

---

## Appendix B: Factor Validation Framework

Before running full backtest, validate factor at each stage:

```
Stage 1: Stock-level validation (single market)
  → Replicate paper's IC results for A-shares ✓
  → Confirm factor works in HK, KR, JP, TW individually

Stage 2: Index aggregation validation
  → Does aggregated factor track index direction?
  → Plot index 博弈因子 vs next-day index return scatter

Stage 3: Cross-country IC
  → For each country: IC(factor_t, return_{t+1w})
  → Expected: IC < -0.03 in most markets

Stage 4: Full cross-sectional backtest
  → Only after stages 1-3 show valid signal
```

---

**Document Version**: 1.0
**Last Updated**: 2026-03-15
**Status**: Ready for Implementation
**Source Paper**: `changjiang_hf_factor_7.pdf` (Changjiang Securities, 2020-08-10)
**Related Plans**: `../dispersion/backtest_plan.md`, `../consistency/backtest_plan.md`
