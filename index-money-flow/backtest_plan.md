# Index Money Flow — APAC Cross-Country Long/Short Strategy — Backtest Implementation Plan

## Executive Summary

This strategy aggregates the **博弈因子 (Game Factor)** — originally from Changjiang Securities
*"高频因子（七）：分布估计下的主动成交占比"* (2020-08-10) — to the **index level** and uses it
to **long/short APAC equity indices** against each other.

With **tick-level trade data available**, we use the **original, exact 博弈因子 method**: classify
each trade as buyer-initiated or seller-initiated by comparing the trade price to the prevailing
bid/ask quotes, then sum up active buy and sell volumes directly. No distribution estimation needed.

**Universe**: HSI (Hong Kong), CSI 300 (China), KOSPI 200 (South Korea), NKY (Japan), TAIEX (Taiwan)
**Execution**: Index futures or liquid ETFs
**Data Source**: Tick trade data (price, volume, direction) for all index constituents
**Signal Frequency**: Daily factor, weekly rebalancing
**Strategy Type**: Cross-sectional long/short on country indices
**Status**: Ready for Implementation

---

## 1. Strategy Theory

### 1.1 博弈因子 — Direct Tick Classification

The **博弈因子** captures the balance of buyer-initiated vs seller-initiated volume for a stock.
With tick data available, we compute it directly:

```
博弈因子 (ratio)      = Vol_Buy / Vol_Sell
博弈因子 (proportion) = Vol_Buy / (Vol_Buy + Vol_Sell)   ← used here: bounded in [0, 1]
```

Where:
- `Vol_Buy` = sum of all buyer-initiated trade volumes over the measurement period
- `Vol_Sell` = sum of all seller-initiated trade volumes
- Unclassified ticks (at midpoint) are excluded from both

**Economic intuition**:
- `博弈因子 > 0.5`: Buyers are more aggressive than sellers → bullish pressure
- `博弈因子 < 0.5`: Sellers are more aggressive → bearish pressure
- At index level: country with high aggregate 博弈因子 has broad-based buying pressure

### 1.2 Tick Classification: Quote-Based (Lee-Ready)

With L2 bid/ask tick data, each trade is classified by comparing its price to the **prevailing
best bid and best ask** at the time of the trade:

```
Rule 1 — Quote Rule (primary):
  trade_price > best_ask  →  active BUY   (buyer lifted the ask)
  trade_price < best_bid  →  active SELL  (seller hit the bid)
  best_bid ≤ trade_price ≤ best_ask  →  midpoint: use tick rule

Rule 2 — Tick Rule (fallback for midpoint trades):
  trade_price > prev_trade_price  →  BUY   (uptick)
  trade_price < prev_trade_price  →  SELL  (downtick)
  trade_price = prev_trade_price  →  same direction as last classified trade (zero-tick)
```

This is the standard **Lee-Ready (1991)** algorithm, and exactly the logic described in the
original paper for the first 博弈因子 construction.

### 1.3 Tick Classification: Trade-Only (Tick Test)

If only trade records are available (no concurrent bid/ask), fall back to the **pure tick test**:

```
current_price > prev_trade_price  →  BUY  (uptick)
current_price < prev_trade_price  →  SELL (downtick)
current_price = prev_trade_price  →  inherit direction of most recent non-zero-tick (zero-tick rule)
```

The tick test is less accurate than the quote rule (misclassification rate ~30% vs ~15%) but
is computed from trade data alone.

**Priority**: Use quote rule wherever bid/ask data is available; fall back to tick test otherwise.

### 1.4 Auction Treatment

Opening and closing auctions use batch pricing — a single clearing price with no meaningful
prior bid/ask context. These ticks must be handled separately:

```
Opening auction volume:  EXCLUDE from 博弈因子 (direction ambiguous at batch price)
Closing auction volume:  EXCLUDE from 博弈因子 (same reason)
Continuous session only: 09:30–11:30, 13:00–15:00 (A-shares); market-specific for others
```

Rationale: auction volume adds noise, not information, to the buy/sell imbalance signal.

### 1.5 From Stock to Index Level

Aggregate constituent-level 博弈因子 to a single **country-level score**:

```
Index_博弈因子_j = Σ_i  w_i × Stock_博弈因子_i

  where:
    i  = constituent stock of index j
    w_i = market-cap weight (re-normalised after excluding missing stocks)
```

**Alternative aggregation** (avoids weight sensitivity):
```
Index_博弈因子_j = Σ_i Vol_Buy_i  /  (Σ_i Vol_Buy_i + Σ_i Vol_Sell_i)
```
This pools all constituent volume directly before computing the ratio — equivalent to
volume-weighting each stock's buy/sell contribution implicitly.

Both approaches should be tested.

### 1.6 Cross-Country Signal Construction

At close of each week, for each country `j`:

```python
# Rolling N-week average of daily index 博弈因子
factor_j = mean(Index_博弈因子_j[-N_days:])   # e.g. 4 weeks = 20 trading days

# Cross-sectional z-score across all countries
z_j = (factor_j - mean(factor_all)) / std(factor_all)

# Signal: long top-k countries, short bottom-k countries
position_j = +1  if rank(z_j) is top-k      # high buying pressure → long
position_j = -1  if rank(z_j) is bottom-k   # high selling pressure → short
position_j =  0  otherwise
```

**Alternative signals to test**:
1. Raw factor level vs own 60-day rolling mean (is this country's buying above its own norm?)
2. Factor change (week-over-week delta — captures turning points)
3. Factor momentum (4-week avg vs prior 4-week avg)
4. Percentile rank vs own 252-day history

---

## 2. Universe & Market Details

### 2.1 Index Universe

| Index | Country | Bloomberg | Futures | ETF | Constituents | Tick Data Source |
|-------|---------|-----------|---------|-----|--------------|-----------------|
| HSI | Hong Kong | HSI Index | HI1 Index | EWH US / 2800 HK | ~80 | HKEX ASTS / Bloomberg |
| CSI 300 | China A | SHSZ300 Index | IFc1 Index | 510300 CH | 300 | Wind / local exchange |
| KOSPI 200 | S. Korea | KOSPI2 Index | KM1 Index | EWY US | 200 | KRX / Bloomberg |
| NKY | Japan | NKY Index | NK1 Index | EWJ US / 1570 JP | 225 | TSE FLEX / Bloomberg |
| TAIEX | Taiwan | TWSE Index | TW1 Index | EWT US | ~900 | TWSE / Bloomberg |

**Optional additions** (if tick data available):
- ASX 200 (Australia): AS51 Index
- STI (Singapore): STI Index

### 2.2 Tick Data Schema per Market

Each market's tick data feed has a slightly different structure. The target schema after
normalisation is identical across all markets:

**Normalised tick record**:
```
timestamp   : datetime (nanosecond precision where available)
ticker      : str
trade_price : float
trade_volume: int (shares/lots)
trade_amount: float (value in local currency)
best_bid    : float  (prevailing at trade time; NaN if not available)
best_ask    : float  (prevailing at trade time; NaN if not available)
session     : str    ('open_auction', 'continuous', 'close_auction')
```

**Per-market notes**:

| Market | Direction pre-labeled? | Bid/Ask in tick feed? | Lunch break | Auction handling |
|--------|----------------------|----------------------|-------------|-----------------|
| China A (SSE/SZSE) | Sometimes (BS flag) | Sometimes (L2) | 11:30–13:00 | Separate open/close call auction files |
| Hong Kong (HKEX) | No | Yes (ASTS has bid/ask) | No | Pre-opening session 09:00–09:30 |
| Korea (KRX) | Yes (buy/sell flag) | Yes | No | Opening/closing call auction flagged |
| Japan (TSE) | No | Yes (FLEX Full) | 11:30–12:30 | Itayose auction at open/close |
| Taiwan (TWSE) | No | Yes (5-level book) | No | Opening call, continuous from 09:00 |

**If direction is pre-labeled** (Korea, some A-share feeds): use label directly, skip classification.
**If not pre-labeled**: apply quote rule (with bid/ask) or tick test (without).

### 2.3 Trading Hour Alignment

Compute the daily factor using **each market's own full continuous session**:

| Market | Continuous Session (local) | UTC | Factor computed at |
|--------|--------------------------|-----|-------------------|
| China A | 09:30–11:30, 13:00–15:00 CST | UTC+8 | 15:00 CST |
| Hong Kong | 09:30–16:00 HKT | UTC+8 | 16:00 HKT |
| Taiwan | 09:00–13:30 CST | UTC+8 | 13:30 CST |
| South Korea | 09:00–15:30 KST | UTC+9 | 15:30 KST |
| Japan | 09:00–11:30, 12:30–15:30 JST | UTC+9 | 15:30 JST |

Signal is finalized **after each market's own close**. Execution at **next trading day's open**.

---

## 3. Data Requirements

### 3.1 Tick Data (Core Input)

The primary data input is **trade-by-trade tick records** for all constituents of each index.

```python
# Expected data format (normalised):
#   market / ticker / date  →  DataFrame of tick records

tick_schema = {
    "timestamp":    "datetime64[ns]",
    "trade_price":  "float64",
    "trade_volume": "int64",          # shares / units
    "trade_amount": "float64",        # price × volume in local ccy
    "best_bid":     "float64",        # NaN if unavailable
    "best_ask":     "float64",        # NaN if unavailable
    "direction":    "Int8",           # +1=buy, -1=sell, 0=unknown (if pre-labeled)
    "session":      "category",       # 'continuous', 'open_auction', 'close_auction'
}

# Recommended storage: Parquet, partitioned by market / date
# path: data/ticks/{market}/{YYYY-MM}/{ticker}.parquet
```

### 3.2 Supporting Data (Bloomberg)

```python
from xbbg import blp

# Historical constituent lists (quarterly)
constituents = blp.bds(
    tickers="HSI Index",
    flds="INDX_MEMBERS",
    INDX_MWEIGHT_HIST_DT="20230101"
)

# Market-cap weights for cap-weighted aggregation
weights = blp.bdh(
    tickers=constituent_list,
    flds=["CUR_MKT_CAP"],
    start_date=..., end_date=...
)

# Index futures daily OHLCV (for backtest execution / returns)
futures_map = {
    "HSI":    "HI1 Index",
    "CSI300": "IFc1 Index",
    "KOSPI":  "KM1 Index",
    "NKY":    "NK1 Index",
    "TAIEX":  "TW1 Index",
}
index_data = blp.bdh(
    tickers=list(futures_map.values()),
    flds=["PX_OPEN", "PX_HIGH", "PX_LOW", "PX_LAST", "PX_VOLUME"],
    start_date="20100101", end_date="20241231"
)
```

### 3.3 Data Volume Estimate

| Index | Constituents | Avg ticks/stock/day | Total ticks × 10yr |
|-------|-------------|--------------------|--------------------|
| HSI | ~80 | ~5,000 | ~1.0B |
| CSI 300 | 300 | ~8,000 | ~6.0B |
| KOSPI 200 | 200 | ~4,000 | ~2.0B |
| NKY | 225 | ~3,000 | ~1.7B |
| TAIEX | ~900 | ~2,000 | ~4.5B |

→ **Total**: ~15B rows over 10 years. Compress with **Parquet + Snappy**.
→ For factor computation, **pre-aggregate to 1-minute classified bins** to reduce runtime:

```python
# Pre-aggregation step: tick → 1-min buy/sell volume bins
# Reduces 15B rows → ~200M rows; factor calculation becomes trivial
# buy_vol_1min[t] = sum of active buy volume in minute t
# sell_vol_1min[t] = sum of active sell volume in minute t
```

### 3.4 Data Handling Challenges

| Challenge | Solution |
|-----------|----------|
| No bid/ask in feed | Fall back to tick test; flag these stocks separately |
| Pre-labeled direction available | Use directly; skip classification entirely |
| Suspended stocks | Exclude on suspension day; do not forward-fill factor |
| Lunch break (China, Japan) | Treat as two sub-sessions; exclude first bar after re-open |
| Opening/closing auction | Filter to `session == 'continuous'` only |
| Block trades / off-exchange | Exclude if flagged as non-exchange trades |
| Tick size constraints | Trades at bid/ask exact price: assign to quote side, not tick rule |
| Survivorship bias | Include delisted constituents in historical factor |
| FX conversion | Report in USD for cross-country comparison |

---

## 4. Factor Construction

### 4.1 Tick Classification

```python
import numpy as np
import pandas as pd


def classify_ticks(ticks: pd.DataFrame) -> pd.DataFrame:
    """
    Classify each trade tick as buyer-initiated (+1), seller-initiated (-1),
    or unclassified (0).

    Args:
        ticks: DataFrame with columns:
               [timestamp, trade_price, trade_volume, best_bid, best_ask,
                direction, session]

    Returns:
        ticks with 'side' column added: +1 (buy), -1 (sell), 0 (unclassified)
    """
    ticks = ticks.copy()

    # Skip auction ticks entirely
    ticks = ticks[ticks['session'] == 'continuous'].copy()

    # If direction is pre-labeled, use it directly
    if ticks['direction'].notna().all() and (ticks['direction'] != 0).any():
        ticks['side'] = ticks['direction']
        return ticks

    # Initialise side
    ticks['side'] = 0

    has_quotes = ticks['best_bid'].notna() & ticks['best_ask'].notna()

    # --- Quote Rule (primary: use where bid/ask is available) ---
    quote_mask = has_quotes
    ticks.loc[quote_mask & (ticks['trade_price'] > ticks['best_ask']), 'side'] = 1
    ticks.loc[quote_mask & (ticks['trade_price'] < ticks['best_bid']), 'side'] = -1
    # Midpoint trades fall through to tick rule below

    # --- Tick Rule (fallback: price change direction) ---
    price_change = ticks['trade_price'].diff()

    # Apply tick rule where quote rule left side=0 (midpoint or no quote)
    needs_tick_rule = ticks['side'] == 0
    ticks.loc[needs_tick_rule & (price_change > 0), 'side'] = 1    # uptick → buy
    ticks.loc[needs_tick_rule & (price_change < 0), 'side'] = -1   # downtick → sell

    # Zero-tick rule: propagate last non-zero direction
    zero_tick = needs_tick_rule & (price_change == 0)
    if zero_tick.any():
        # Forward-fill last classified direction into zero-tick positions
        last_direction = ticks['side'].replace(0, np.nan).ffill()
        ticks.loc[zero_tick, 'side'] = last_direction[zero_tick].fillna(0).astype(int)

    return ticks


def calc_stock_game_factor(ticks: pd.DataFrame) -> float:
    """
    Calculate 博弈因子 (active buy proportion) for a single stock on a single day.

    Args:
        ticks: classified tick DataFrame (output of classify_ticks)

    Returns:
        float: Vol_Buy / (Vol_Buy + Vol_Sell), in [0, 1]
               NaN if insufficient classified volume
    """
    buy_vol  = ticks.loc[ticks['side'] == +1, 'trade_volume'].sum()
    sell_vol = ticks.loc[ticks['side'] == -1, 'trade_volume'].sum()

    total_classified = buy_vol + sell_vol
    if total_classified == 0:
        return np.nan

    return buy_vol / total_classified
```

### 4.2 Index-Level Aggregation

```python
def calc_index_game_factor(
    constituent_ticks: dict,     # {ticker: classified_ticks_df}
    weights: pd.Series,          # cap weights indexed by ticker
    method: str = "cap_weighted" # 'cap_weighted', 'equal_weighted', 'volume_pooled'
) -> float:
    """
    Aggregate constituent 博弈因子 to index level.

    Three aggregation methods:
      cap_weighted   : weighted average of per-stock factors
      equal_weighted : simple average of per-stock factors
      volume_pooled  : pool all buy/sell volume across constituents before ratio
                       (implicitly volume-weights each constituent)
    """
    if method == "volume_pooled":
        total_buy  = sum(
            ticks.loc[ticks['side'] == +1, 'trade_volume'].sum()
            for ticks in constituent_ticks.values()
        )
        total_sell = sum(
            ticks.loc[ticks['side'] == -1, 'trade_volume'].sum()
            for ticks in constituent_ticks.values()
        )
        classified = total_buy + total_sell
        return total_buy / classified if classified > 0 else np.nan

    # Per-stock factors
    stock_factors = {}
    for ticker, ticks in constituent_ticks.items():
        f = calc_stock_game_factor(ticks)
        if not np.isnan(f):
            stock_factors[ticker] = f

    if not stock_factors:
        return np.nan

    factors = pd.Series(stock_factors)

    if method == "equal_weighted":
        return factors.mean()

    # cap_weighted (default)
    w = weights.reindex(factors.index).dropna()
    factors = factors.reindex(w.index)
    w = w / w.sum()
    return (factors * w).sum()
```

### 4.3 Daily Pipeline

```python
def compute_daily_index_factor(
    date: str,
    market: str,
    tick_store: TickDataStore,        # abstraction over parquet files
    constituent_registry: dict,       # date → list of (ticker, weight)
    aggregation: str = "cap_weighted"
) -> float:
    """Full pipeline for one market on one date."""

    constituents = constituent_registry.get_constituents(market, date)
    weights = pd.Series({t: w for t, w in constituents})

    classified_ticks = {}
    for ticker, _ in constituents:
        raw = tick_store.load(market, ticker, date)
        if raw is None or len(raw) == 0:
            continue
        classified = classify_ticks(raw)
        classified_ticks[ticker] = classified

    # Require at least 80% of constituents by weight to have data
    available_weight = weights[list(classified_ticks.keys())].sum()
    if available_weight / weights.sum() < 0.80:
        return np.nan

    return calc_index_game_factor(classified_ticks, weights, method=aggregation)
```

### 4.4 Weekly Signal Construction

```python
def build_weekly_signal(
    daily_factors: pd.DataFrame,    # columns = markets, index = dates
    lookback_weeks: int = 4,
    signal_type: str = "level"
) -> pd.DataFrame:
    """Build cross-sectional weekly signal from daily index 博弈因子."""

    weekly = daily_factors.resample('W-FRI').mean()

    if signal_type == "level":
        signal = weekly.rolling(lookback_weeks).mean()

    elif signal_type == "change":
        signal = weekly.diff(1)

    elif signal_type == "momentum":
        curr = weekly.rolling(lookback_weeks).mean()
        prev = curr.shift(lookback_weeks)
        signal = curr - prev

    elif signal_type == "percentile":
        signal = weekly.rolling(52).rank(pct=True)

    # Cross-sectional z-score
    signal_z = signal.sub(signal.mean(axis=1), axis=0)
    signal_z = signal_z.div(signal_z.std(axis=1), axis=0)
    return signal_z


def generate_positions(
    signal: pd.DataFrame,
    long_k: int = 2,
    short_k: int = 2,
) -> pd.DataFrame:
    """Long top-k, short bottom-k, equal dollar weight."""
    positions = pd.DataFrame(0.0, index=signal.index, columns=signal.columns)

    for date, row in signal.iterrows():
        valid = row.dropna()
        if len(valid) < long_k + short_k:
            continue
        ranked = valid.rank(ascending=False)
        long_idx  = ranked[ranked <= long_k].index
        short_idx = ranked[ranked > len(ranked) - short_k].index
        positions.loc[date, long_idx]  =  1.0 / long_k
        positions.loc[date, short_idx] = -1.0 / short_k

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
  - Primary: front-month index futures (roll 5 days before expiry)
  - Alternative: liquid index ETF (lower cost, no roll risk)
```

### 5.2 Transaction Costs

| Market | Futures | Est. Round-Trip | ETF Alternative |
|--------|---------|----------------|-----------------|
| HSI | HI1 Index | 0.10% | EWH US |
| CSI 300 | IFc1 Index | 0.08% (pre-2015) / 0.20% (post-2015) | 510300 CH |
| KOSPI 200 | KM1 Index | 0.10% | EWY US |
| NKY | NK1 Index | 0.08% | EWJ US / 1570 JP |
| TAIEX | TW1 Index | 0.10% | EWT US |

### 5.3 Currency Handling

Test in both currencies to distinguish equity signal from FX effect:
1. **Local-currency returns**: pure equity signal, FX-agnostic
2. **USD-denominated returns**: `return_usd ≈ return_local + return_fx`

If IC is significant in both, the signal is genuine equity signal.

FX pairs: USDCNH (China), USDHKD (HK, quasi-fixed), USDKRW (Korea), USDJPY (Japan), USDTWD (Taiwan)

### 5.4 Test Periods

| Period | Label | Notes |
|--------|-------|-------|
| 2010–2019 | In-sample | Parameter optimisation |
| 2020–2024 | Out-of-sample | COVID, rate cycle, Taiwan risk |
| 2010–2024 | Full period | Overall assessment |
| 2005–2009 | Stress test | GFC; run if tick data available |

### 5.5 Performance Metrics

| Metric | Target |
|--------|--------|
| Annual return (L/S, net of costs) | > 10% |
| Max drawdown | < −20% |
| Sharpe ratio | > 0.8 |
| IC (factor vs next-week index return) | < −0.03 (negative: high buying → positive return) |
| ICIR | < −0.4 |
| Win rate (weekly) | > 50% |
| % weeks with signal (at least 1 trade) | > 80% |

---

## 6. Implementation Roadmap

### Phase 1: Tick Data Infrastructure (Week 1–2)

#### 6.1 Tick Data Loader & Normaliser
**File**: `src/index_money_flow/tick_store.py`

```python
MARKET_CONFIG = {
    "CSI300": {
        "index_ticker":      "SHSZ300 Index",
        "futures_ticker":    "IFc1 Index",
        "session_start":     "09:30",
        "session_end":       "15:00",
        "lunch_break":       ("11:30", "13:00"),
        "auction_flags":     True,           # separate auction files exist
        "direction_labeled": False,          # classify ourselves
        "bid_ask_available": True,           # L2 data available
        "rebalance_months":  [3, 6, 9, 12],
    },
    "HSI": {
        "index_ticker":      "HSI Index",
        "futures_ticker":    "HI1 Index",
        "session_start":     "09:30",
        "session_end":       "16:00",
        "lunch_break":       None,
        "auction_flags":     True,
        "direction_labeled": False,
        "bid_ask_available": True,
        "rebalance_months":  [3, 6, 9, 12],
    },
    "KOSPI200": {
        "index_ticker":      "KOSPI2 Index",
        "futures_ticker":    "KM1 Index",
        "session_start":     "09:00",
        "session_end":       "15:30",
        "lunch_break":       None,
        "auction_flags":     True,
        "direction_labeled": True,           # KRX pre-labels buy/sell side
        "bid_ask_available": True,
        "rebalance_months":  [3, 6, 9, 12],
    },
    "NKY": {
        "index_ticker":      "NKY Index",
        "futures_ticker":    "NK1 Index",
        "session_start":     "09:00",
        "session_end":       "15:30",
        "lunch_break":       ("11:30", "12:30"),
        "auction_flags":     True,
        "direction_labeled": False,
        "bid_ask_available": True,           # TSE FLEX Full data
        "rebalance_months":  [3, 6, 9, 12],
    },
    "TAIEX": {
        "index_ticker":      "TWSE Index",
        "futures_ticker":    "TW1 Index",
        "session_start":     "09:00",
        "session_end":       "13:30",
        "lunch_break":       None,
        "auction_flags":     True,
        "direction_labeled": False,
        "bid_ask_available": True,
        "rebalance_months":  [3, 6, 9, 12],
    },
}
```

### Phase 2: Classification & Factor Engine (Week 2–3)

#### 6.2 Factor Pipeline
**File**: `src/index_money_flow/factor_engine.py`

```
For each trading day:
  For each market in universe:
    1. Load historical constituent list for that date
    2. For each constituent:
       a. Load raw tick records (continuous session only)
       b. If direction pre-labeled → use directly
          Else if bid/ask available → quote rule + tick rule fallback
          Else → pure tick test
       c. Compute stock_博弈因子 = buy_vol / (buy_vol + sell_vol)
    3. Aggregate to index_博弈因子 (cap-weighted, equal-weighted, volume-pooled)
    4. Store daily factor series

Output: panel DataFrame (trading_days × 5_markets)
Intermediate cache: per-stock daily factors (parquet)
```

### Phase 3: Signal & Backtest (Week 3–4)

#### 6.3 Signal Construction
**File**: `src/index_money_flow/signal.py`

Parameters to optimise (in-sample 2010–2019):
- `lookback_weeks`: [1, 2, 4, 6, 8]
- `signal_type`: ['level', 'change', 'momentum', 'percentile']
- `aggregation`: ['cap_weighted', 'equal_weighted', 'volume_pooled']
- `long_k`: [1, 2]
- `short_k`: [1, 2]

#### 6.4 Backtester
**File**: `src/index_money_flow/backtester.py`

```python
class IndexMoneyFlowBacktester:
    def __init__(self, initial_capital=10_000_000):  # USD 10M
        self.capital = initial_capital

    def run(self, positions, returns, costs):
        """Weekly rebalancing backtest with transaction costs."""
        ...

    def calculate_performance(self, results):
        return {
            "annual_return":     ...,
            "max_drawdown":      ...,
            "sharpe_ratio":      ...,
            "ic":                ...,   # cross-sectional IC
            "icir":              ...,
            "win_rate":          ...,
            "avg_weekly_return": ...,
            "classification_rate": ..., # % of volume successfully classified
        }
```

### Phase 4: Analysis (Week 4–5)

1. **Tick classification audit**: What % of volume is classified per market? How does misclassification affect factor?
2. **Factor validation**: IC(index_博弈因子_t, return_{t+1w}) by country and year
3. **Country profiles**: Does retail-heavy Taiwan/Korea show stronger predictability than HFT-heavy Japan?
4. **Aggregation comparison**: cap-weighted vs equal-weighted vs volume-pooled
5. **Style attribution**: Correlation with country momentum, carry, value factors
6. **Out-of-sample**: Performance on 2020–2024 holdout

---

## 7. Hypotheses to Test

### H1: Factor Validity
**Does index-level 博弈因子 predict next-week index returns cross-sectionally?**
- Expected: IC < 0 (high buying pressure → positive return next week; ranked low factor = net selling → negative)
- Wait — check sign: HIGH factor = more buying = BULLISH → expect POSITIVE forward return → IC > 0

### H2: Aggregation Method
**Does the aggregation method matter?**
- `volume_pooled`: naturally volume-weights each stock; large-cap stocks dominate (good for index futures)
- `cap_weighted`: weighted by market cap; may differ from volume-pooled when large stocks are inactive
- `equal_weighted`: all stocks equal; may pick up small-cap sentiment not driving the index
- Expected: `volume_pooled` ≈ `cap_weighted` > `equal_weighted` for index-level prediction

### H3: Quote Rule vs Tick Rule
**Does bid/ask-based classification improve signal quality vs tick test alone?**
- Lee-Ready quote rule: ~85% accuracy (Ellis, Michaely, O'Hara 2000)
- Pure tick test: ~68% accuracy
- Expected: markets where we have good bid/ask data (KRX pre-labeled, TSE FLEX) show stronger IC
- Test: compute factor with and without bid/ask; compare IC

### H4: Signal Type
**Is factor level or factor change more informative?**
- `level`: captures persistent bull/bear markets within a country
- `change`: captures turning points, higher turnover/cost
- Expected: `level` with 4-week lookback gives best risk-adjusted signal; `change` shows shorter-horizon IC

### H5: Market Hours Effect
**Does overlapping market hours contaminate the signal?**
- China and HK have overlapping hours with each other
- Japan/Korea are in same timezone — signals may be correlated
- Test: lag factor by 1 day to confirm no look-ahead; check pairwise factor correlation

### H6: Factor Decay
**At which horizon is the signal strongest?**
- Test IC at: 1-day, 1-week, 2-week, 4-week forward returns
- Expected: strongest at 1-week, decays by 4-week (consistent with paper's findings for stocks)

---

## 8. Risk Considerations

### 8.1 Structural Risks
- **China A-shares restrictions (post-2015-09-07)**: IF futures severely restricted; use ETF 510300
- **Market closures**: HK (2019 protests), COVID (2020 volatility), Taiwan geopolitical events
- **Capital controls (China)**: limits cross-market arbitrage; may sustain the signal longer

### 8.2 Data Risks
- **Tick data completeness**: Missing ticks for some stocks/dates; track fill rate per market per year
- **Classification rate**: % of volume successfully classified; aim >85% per stock
- **Survivorship bias**: Must include delisted constituents in historical factor computation
- **Auction contamination**: Misclassified auction ticks would distort the factor; filter carefully
- **Data format changes**: Exchange tick feed format may change over history; normalise carefully
- **Japan lunch break**: ~1hr gap; first tick after re-open has no valid prior context → skip

### 8.3 Factor Degradation
- Original paper notes signal weakened post-2017 in A-shares (more sophisticated participants)
- For APAC cross-country application: factor may degrade if arbitrageurs exploit the signal
- Taiwan and Korea (retail-heavy) likely most persistent; Japan (HFT-heavy) may degrade faster

### 8.4 Execution Risks
- **Futures roll**: Monthly contract roll; model roll cost explicitly
- **Basis risk**: Futures/spot divergence during stress; may gap on Monday open
- **TAIEX futures liquidity**: Thinner than HSI/NKY; slippage higher than estimated

---

## 9. Expected Deliverables

### 9.1 Code Structure

```
strategies-impl/
└── index_money_flow/
    ├── src/
    │   └── index_money_flow/
    │       ├── tick_store.py          # Tick data loading & normalisation
    │       ├── classifier.py          # Quote rule + tick rule classification
    │       ├── factor_engine.py       # Stock → index 博弈因子 pipeline
    │       ├── signal.py              # Weekly signal construction
    │       ├── backtester.py          # Backtest engine
    │       └── analytics.py          # Performance metrics + plots
    ├── config/
    │   └── market_config.yaml         # Per-market session/auction parameters
    ├── notebooks/
    │   ├── 01_classification_audit.ipynb  # Verify tick classification quality
    │   ├── 02_factor_validation.ipynb     # IC analysis per country
    │   ├── 03_signal_optimisation.ipynb   # Parameter sweep
    │   ├── 04_backtest_results.ipynb      # Full L/S backtest
    │   └── 05_risk_analysis.ipynb         # Style attribution, drawdowns
    ├── backtest_index_money_flow.py    # Main script
    └── results/
        ├── factor_panel.parquet
        ├── backtest_results.csv
        └── plots/
```

### 9.2 Key Outputs

1. **Classification audit table**: % classified buy / sell / unclassified per market per year
2. **Factor IC heatmap**: IC by country × year
3. **Country factor time series**: index 博弈因子 over time, overlaid with index return
4. **L/S equity curve**: USD-denominated cumulative return, net of costs
5. **Drawdown chart**: strategy vs equal-weight buy-and-hold APAC basket
6. **Parameter sensitivity heatmap**: Sharpe ratio across lookback × signal_type
7. **Style attribution**: correlation with country momentum, value, FX carry

---

## 10. Comparison with Original Paper

| Dimension | Original (Paper) | This Strategy |
|-----------|-----------------|---------------|
| Universe | A-share individual stocks | 5 APAC country indices |
| Signal use | Cross-sectional stock rank | Cross-country index rank |
| Factor method | Tick data OR distribution estimate | **Tick data only (direct)** |
| Holding period | Daily (long IC) | Weekly |
| Execution | Stock portfolio | Index futures / ETFs |
| Markets | China A-shares only | China, HK, Korea, Japan, Taiwan |
| Factor type | Absolute stock factor | Aggregated index factor |
| Return source | Stock alpha | Country allocation |
| FX exposure | None | Multi-currency |

---

## 11. Future Extensions

### 11.1 Sector-Level Application
Aggregate 博弈因子 within sectors (e.g., financials vs tech) → sector rotation within each country

### 11.2 Combination with Other Signals
- **Dispersion factor**: low cross-stock dispersion + high index 博弈因子 = broad-based buying = strong long
- **Consistency factor**: high PCA consistency (all stocks move together) + high 博弈因子 = high-conviction long

### 11.3 Intraday Timing
Use real-time rolling 博弈因子 (computed every N minutes) as an intraday entry signal for futures:
- Enter long when intraday 博弈因子 crosses above its N-day rolling mean
- Similar architecture to the consistency strategy (Strategy #2)

### 11.4 Extended Universe
- India (NSE NIFTY 50), Australia (ASX 200), Singapore (STI)
- Requires obtaining constituent tick data for those exchanges

---

## 12. References

1. **Primary Paper**: "高频因子（七）：分布估计下的主动成交占比" — Changjiang Securities (长江证券), 覃川桃 & 郑起, 2020-08-10
2. **Related prior work** (cited in paper):
   - "高频因子（五）：高频因子和交易行为" — original 博弈因子 with tick data
   - "高频因子（二）：结构化反转因子" — structural reversal factor
3. **Classification methodology**: Lee, C. and Ready, M. (1991). *Inferring trade direction from intraday data.* Journal of Finance, 46(2), 733–746.
4. **Accuracy benchmark**: Ellis, K., Michaely, R. and O'Hara, M. (2000). *The accuracy of trade classification rules.* Journal of Financial and Quantitative Analysis, 35(4), 529–551.
5. **Companion Plans** in this repo:
   - `../dispersion/backtest_plan.md` — weekly dispersion momentum (Guosen, 2017)
   - `../consistency/backtest_plan.md` — intraday PCA consistency (GF Securities, 2016)
6. **Execution reference**: HKEX, KRX, TSE, TAIFEX, CFFEX futures specifications

---

## Appendix A: Market-Specific Notes

### A-shares (CSI 300)
- T+1 settlement; no intraday short selling at stock level
- Post-2015-09-07: IF futures restricted → use ETF 510300 as execution vehicle
- STAR board stocks: ±20% price limit (not ±10%); separate handling if in constituent universe
- Tick data: Wind or exchange Level-2 data; buy/sell direction sometimes pre-labeled in BS field
- Lunch break 11:30–13:00: classify sub-sessions independently; first bar after re-open uses tick rule

### Hong Kong (HSI)
- ~40 stocks account for >80% of index weight — concentrated; factor sensitive to Tencent/HSBC
- Pre-opening session (09:00–09:30): auction prices — exclude
- Tick data: HKEX ASTS provides best bid/ask with each trade → use quote rule
- Short selling allowed; 博弈因子 signal has two-sided interpretation

### South Korea (KOSPI 200)
- KRX tick data **pre-labels** buy (매수) and sell (매도) side on each trade → use directly, skip classification
- ±30% price limit (since 2015; was ±15% before) — affects normalisation if mixing with bar-based data
- Opening/closing batch auctions clearly flagged in KRX feed — easy to filter

### Japan (Nikkei 225)
- **Price-weighted index** (unusual): stock price, not market cap, determines index weight
- For 博弈因子 aggregation: use **equal weight or volume-pooled** (price-weighting is an artifact of index construction, not economic relevance)
- TSE FLEX Full data provides full order book → excellent bid/ask availability → high classification accuracy
- Lunch break 11:30–12:30: first tick after re-open — use tick rule (no prior context)
- High HFT presence: tick data volume is large; may pre-aggregate to 1-second bins before classification

### Taiwan (TAIEX)
- ~900 constituents; focus on top 100–150 stocks by weight (cover >80% of index weight) to manage data volume
- TWSE provides 5-level order book with each trade → use quote rule
- Retail-heavy market: 博弈因子 signal likely stronger here than Japan
- Half-day trading on some pre-holiday dates: handle incomplete sessions by excluding or normalising

---

## Appendix B: Factor Validation Framework

Before running full cross-country backtest, validate at each stage:

```
Stage 1: Classification quality (single market, single stock)
  → Verify classification rate > 85% for liquid stocks
  → Cross-check against pre-labeled KRX data as ground truth

Stage 2: Stock-level factor validity
  → Replicate paper's A-share IC results using tick data (not distribution estimate)
  → Confirm factor IC < -5% for A-shares (paper benchmark)

Stage 3: Index aggregation
  → Does aggregated index 博弈因子 correlate with same-day index direction?
  → Plot index 博弈因子 vs next-day index return scatter for each market

Stage 4: Cross-country IC
  → For each country: IC(factor_t, return_{t+1w})
  → Expected: |IC| > 0.03 in at least 3 of 5 markets

Stage 5: Full cross-sectional backtest
  → Only proceed after stages 1-4 confirm valid signal
```

---

**Document Version**: 2.0 (updated to tick data)
**Last Updated**: 2026-03-15
**Status**: Ready for Implementation
**Source Paper**: `changjiang_hf_factor_7.pdf` (Changjiang Securities, 2020-08-10)
**Related Plans**: `../dispersion/backtest_plan.md`, `../consistency/backtest_plan.md`
