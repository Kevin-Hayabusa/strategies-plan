# Constituent Consistency-Based Index Trend Strategy - Backtest Implementation Plan

## Executive Summary

This document outlines the implementation plan for backtesting the **constituent stock consistency (成分股一致性)** strategy for measuring market trends and trading index futures, as described in the research paper "从成份股一致性来衡量市场的趋势 - 另类交易策略系列之三十二" (GF Securities, September 2016).

**Primary Target**: CSI 300 Index Futures (IF) — expandable to IH (SSE 50), IC (CSI 500/1000)  
**Data Source**: Bloomberg Terminal (xbbg Python library)  
**Test Period**: 2010-2013 (in-sample), 2014-2016 (out-of-sample), extended to present  
**Strategy Type**: Intraday trend-following with PCA-based consistency filter  
**Benchmark**: Simple random range breakout strategy

---

## 1. Strategy Theory

### 1.1 Constituent Consistency vs. Return Dispersion

This strategy is conceptually related to but mathematically distinct from the return dispersion strategy:

| Aspect | Return Dispersion | Constituent Consistency |
|--------|------------------|------------------------|
| **Measure** | Standard deviation of returns | First PC variance explained ratio |
| **Signal** | High dispersion → trend risk | Low consistency → avoid trading |
| **Frequency** | Weekly rebalancing | Intraday (T minutes after open) |
| **Instrument** | ETFs / Index | Index Futures (intraday) |
| **Math basis** | Cross-sectional std | PCA eigenvalue decomposition |

**Key Insight**: When constituent stocks move in unison (high consistency), market "collective force" creates larger trends. When stocks move randomly (low consistency), no clear trend forms.

### 1.2 Consistency Indicator Derivation

#### Step 1: Price Matrix Construction

Given constituent prices `p_i(t)` for i = 1, 2, ..., N constituents:

```
P = [ p_1(t-L+1)  ...  p_N(t-L+1) ]
    [     ⋮       ⋱         ⋮      ]
    [ p_1(t)      ...  p_N(t)      ]
```

Where:
- L = window length (e.g., 20 days × 8 intervals = 160 data points for 30-min bars)
- Prices are standardized: divide each series by its first value

#### Step 2: Covariance Matrix

```
Σ = covariance matrix of P  (N × N matrix)
σ(i,j) = cov(p_i, p_j)
```

#### Step 3: PCA - Finding the Dominant Direction

**Objective**: Find transformation vector u₁ that maximizes variance of the transformed series:
```
max  var(P·u₁) = u₁ᵀ Σ u₁
 u₁
subject to: u₁ᵀ u₁ = 1
```

**Solution**: u₁ is the eigenvector corresponding to the **largest eigenvalue λ₁** of Σ.

The maximum achievable variance is `var(q₁) = λ₁`.

#### Step 4: Consistency Indicator R

```
R = λ₁ / (λ₁ + λ₂ + ... + λ_N) × 100%
```

Where:
- λ₁ ≥ λ₂ ≥ ... ≥ λ_N ≥ 0 are all eigenvalues of Σ
- R = first principal component's **variance explained ratio**
- R ∈ [0, 100%]

**Interpretation**:
- R → 100%: All stocks move together (perfect consistency) → strong trend likely
- R → 0%: Stocks move independently (low consistency) → choppy market

**Typical values** in CSI 300: R ranges from ~50% to ~80% normally; spikes to 90%+ in extreme markets (e.g., 2015 crash).

### 1.3 Trading Strategy

#### Strategy 1: Pure Consistency Strategy (Primary)

```python
# At time T minutes after market open (T = 24 minutes optimal):
R = calculate_consistency(constituent_prices, window=160)  # 20 days × 8 bars

if R > threshold:
    # High consistency → market likely to trend
    if futures_price[T] > futures_price[open]:
        position = LONG   # Upward momentum
    else:
        position = SHORT  # Downward momentum
    
    # Hold until close or stop-loss
    stop_loss = entry_price * (1 - 0.006)  # 0.6% stop
else:
    # Low consistency → skip trading today
    position = FLAT
```

**Threshold Calculation**:
```python
# Rolling mean of past 60 trading days at same time T
threshold = mean(R_history[-60:])
```

**Key Parameters**:
- `T`: Entry time after open (optimize over 21-60 minutes)
- `stop_loss_ratio`: 0.6% (fixed)
- `lookback_days`: 60 (for threshold calculation)
- `window_bars`: 160 (20 days × 8 thirty-minute bars)

#### Strategy 2: Consistency-Controlled Range Breakout (Enhancement)

Applies the consistency filter to an existing **random range breakout strategy**:

```python
# Original range breakout logic
Range ~ Uniform(0.3%, 1.0%)  # Random threshold each day
upper_band = open_price * (1 + Range)
lower_band = open_price * (1 - Range)

# Enhanced: filter with consistency
if price > upper_band or price < lower_band:
    # Breakout signal triggered
    R = calculate_consistency(constituent_prices)
    
    if R > threshold:
        # High consistency → execute breakout trade
        execute_trade(direction=breakout_direction)
    else:
        # Low consistency → ignore breakout signal
        pass  # Skip this trade
```

**Monte Carlo Simulation**:
- Run 100 simulations per day with different random Range values
- Average the returns across simulations
- Report daily average as strategy return

---

## 2. Data Requirements

### 2.1 Bloomberg Data Fields

#### Index Futures Data
```python
# Bloomberg futures tickers
IF = "IFc1 Index"   # CSI 300 futures front-month continuous
IH = "IHc1 Index"   # SSE 50 futures (extension)
IC = "ICc1 Index"   # CSI 500 futures (extension)

# Required fields (intraday)
fields = [
    "PX_OPEN",    # Open price
    "PX_HIGH",    # High price
    "PX_LOW",     # Low price
    "PX_LAST",    # Close price
    "PX_VOLUME",  # Volume
]

# Download 1-minute bars
data = blp.bdib(
    ticker="IFc1 Index",
    dt="2016-07-15",
    session="day"
)
```

#### CSI 300 Constituent Data (Intraday)
```python
# Get current constituents
constituents = blp.bds("SHSZ300 Index", "INDX_MEMBERS")

# Download 1-minute or 30-minute bars for each constituent
for stock in constituents:
    data = blp.bdib(
        ticker=stock,
        dt="2016-07-15",
        session="day",
        bar_size=30  # 30-minute bars
    )
```

### 2.2 Data Frequency Requirements

| Data Type | Frequency | Purpose |
|-----------|-----------|---------|
| Index futures | 1-minute | Strategy execution signal |
| Constituent stocks | 30-minute | Consistency calculation |
| Constituent stocks | 1-minute | For T-1 minute calculation extension |

### 2.3 Calculation Window

**Daily consistency calculation at time T**:
- Use last 20 trading days of 30-minute bars: 20 × 8 = 160 data points per stock
- Include bars from 9:30 to T on current day
- Standardize each price series

**Threshold calculation**:
- Collect R values at same time T for past 60 trading days
- Threshold = rolling mean of these 60 values

### 2.4 Historical Data Considerations

- **Lookback period**: Need at least 60 trading days of history before first trade
- **Constituent changes**: Update constituent list quarterly (same as dispersion strategy)
- **Suspended stocks**: Exclude from consistency calculation on suspension days
- **Market hours**: A-shares trade 9:30-11:30, 13:00-15:00
- **After 2015-09-07**: Increased transaction costs for index futures (avoid same-day close)

---

## 3. Implementation Roadmap

### Phase 1: Data Infrastructure (Week 1-2)

#### 3.1 xbbg Intraday Data Wrapper
**File**: `src/index_momentum/data_xbbg.py`

```python
class XbbgDataWrapper:
    """Bloomberg data wrapper using xbbg library."""
    
    def download_intraday_data(self, ticker, start_date, end_date, 
                                bar_size=1, session="day"):
        """Download intraday bar data (1-min, 30-min etc.)."""
        from xbbg import blp
        
        all_data = []
        for date in trading_calendar(start_date, end_date):
            try:
                data = blp.bdib(
                    ticker=ticker,
                    dt=date,
                    session=session,
                    bar_size=bar_size
                )
                all_data.append(data)
            except Exception as e:
                print(f"Failed {date}: {e}")
        
        return pd.concat(all_data)
    
    def download_constituent_intraday(self, index_ticker, start_date, 
                                       end_date, bar_size=30):
        """Download intraday data for all index constituents."""
        constituents = self.download_constituents(index_ticker)
        
        data = {}
        for stock in constituents:
            print(f"Downloading {stock}...")
            data[stock] = self.download_intraday_data(
                stock, start_date, end_date, bar_size=bar_size
            )
        return data
```

#### 3.2 PCA Consistency Calculator
**File**: `src/index_momentum/consistency.py`

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import StandardScaler


class ConsistencyCalculator:
    """
    Calculate constituent stock consistency indicator using PCA.
    
    Based on the paper "从成份股一致性来衡量市场的趋势"
    (GF Securities, September 2016)
    """
    
    def build_price_matrix(self, constituent_data, current_time, 
                            window_bars=160):
        """
        Build price matrix P (L × N) from constituent data.
        
        Args:
            constituent_data: Dict of {ticker: DataFrame with price series}
            current_time: Current timestamp (use data up to here)
            window_bars: Number of bars to include (default 160 = 20 days × 8 bars)
        
        Returns:
            P: numpy array of shape (window_bars, N_constituents)
        """
        price_series = []
        
        for ticker, df in constituent_data.items():
            # Get price data up to current_time
            prices = df.loc[:current_time, 'close'].tail(window_bars)
            
            if len(prices) < window_bars:
                continue  # Skip stocks with insufficient data
            
            # Standardize: divide by first value
            standardized = prices / prices.iloc[0]
            price_series.append(standardized.values)
        
        if not price_series:
            return None
        
        P = np.column_stack(price_series)  # Shape: (window_bars, N)
        return P
    
    def calculate_covariance_matrix(self, P):
        """
        Calculate covariance matrix Σ of price matrix P.
        
        Args:
            P: Price matrix of shape (L, N)
        
        Returns:
            Σ: Covariance matrix of shape (N, N)
        """
        return np.cov(P.T)  # Transpose: each row is a variable
    
    def calculate_consistency(self, constituent_data, current_time, 
                               window_bars=160):
        """
        Calculate the consistency indicator R using PCA.
        
        R = λ₁ / (λ₁ + λ₂ + ... + λ_N) × 100%
        
        Args:
            constituent_data: Dict of {ticker: DataFrame}
            current_time: Current timestamp
            window_bars: Lookback window (default 160)
        
        Returns:
            R: Consistency indicator in [0, 1]
        """
        P = self.build_price_matrix(constituent_data, current_time, window_bars)
        
        if P is None or P.shape[1] < 2:
            return np.nan
        
        # Calculate covariance matrix
        sigma = self.calculate_covariance_matrix(P)
        
        # Calculate eigenvalues (sorted descending)
        eigenvalues = np.linalg.eigvalsh(sigma)
        eigenvalues = np.sort(eigenvalues)[::-1]  # Descending order
        
        # Remove negative eigenvalues (numerical artifacts)
        eigenvalues = np.maximum(eigenvalues, 0)
        
        # Consistency indicator: variance explained by first PC
        total_variance = np.sum(eigenvalues)
        
        if total_variance == 0:
            return np.nan
        
        R = eigenvalues[0] / total_variance
        return R
    
    def calculate_rolling_consistency(self, constituent_data, timestamps, 
                                       window_bars=160):
        """Calculate consistency for multiple timestamps."""
        results = {}
        for t in timestamps:
            results[t] = self.calculate_consistency(
                constituent_data, t, window_bars
            )
        return pd.Series(results)
    
    def calculate_threshold(self, consistency_history, lookback_days=60):
        """
        Calculate rolling threshold as mean of past 60 days.
        
        Args:
            consistency_history: Series of R values indexed by date
            lookback_days: Number of days for rolling mean
        
        Returns:
            threshold: Rolling mean threshold
        """
        return consistency_history.rolling(
            window=lookback_days, min_periods=20
        ).mean()
    
    def detect_consistency_signal(self, R, threshold):
        """
        Determine if consistency is high enough for trading.
        
        Args:
            R: Current consistency indicator
            threshold: Threshold value (rolling mean)
        
        Returns:
            bool: True if consistency is strong (R > threshold)
        """
        if np.isnan(R) or np.isnan(threshold):
            return False
        return R > threshold
```

### Phase 2: Strategy Implementation (Week 2-3)

#### 3.3 Intraday Strategy Classes
**File**: `src/index_momentum/strategies_intraday.py`

```python
class ConsistencyMomentumStrategy:
    """
    Pure consistency-based intraday momentum strategy.
    
    At time T after open:
    1. Calculate constituent consistency R
    2. If R > threshold: follow momentum direction
    3. Hold until close or stop-loss
    """
    
    def __init__(self, entry_time_minutes=24, stop_loss=0.006, 
                 lookback_days=60, window_bars=160):
        self.T = entry_time_minutes        # Minutes after 9:30
        self.stop_loss = stop_loss         # 0.6% stop loss
        self.lookback_days = lookback_days # Threshold calculation window
        self.window_bars = window_bars     # Price matrix window
        
        self.calc = ConsistencyCalculator()
    
    def generate_signals(self, futures_data, constituent_data):
        """
        Generate daily trading signals.
        
        Returns:
            signals: DataFrame with columns [date, direction, entry_price, 
                      exit_price, return, R, threshold, traded]
        """
        results = []
        
        for date in futures_data.index.get_level_values(0).unique():
            # Get T-minute timestamp
            entry_time = pd.Timestamp(f"{date} 09:{30+self.T}:00")
            open_time = pd.Timestamp(f"{date} 09:30:00")
            
            # Calculate consistency at time T
            R = self.calc.calculate_consistency(
                constituent_data, entry_time, self.window_bars
            )
            
            # Calculate threshold from history
            threshold = self._get_threshold(date)
            
            # Decision
            if self.calc.detect_consistency_signal(R, threshold):
                # High consistency → trade
                open_price = futures_data.loc[(date, open_time), 'open']
                entry_price = futures_data.loc[(date, entry_time), 'close']
                
                direction = 1 if entry_price > open_price else -1  # Long/Short
                
                # Simulate intraday PnL with stop-loss
                pnl, exit_price, exit_reason = self._simulate_trade(
                    futures_data, date, entry_time, entry_price, 
                    direction, self.stop_loss
                )
                
                results.append({
                    'date': date,
                    'R': R,
                    'threshold': threshold,
                    'traded': True,
                    'direction': direction,
                    'entry_price': entry_price,
                    'exit_price': exit_price,
                    'return': pnl,
                    'exit_reason': exit_reason,
                })
            else:
                # Low consistency → skip
                results.append({
                    'date': date,
                    'R': R,
                    'threshold': threshold,
                    'traded': False,
                    'direction': 0,
                    'return': 0,
                })
        
        return pd.DataFrame(results)
    
    def _simulate_trade(self, futures_data, date, entry_time, 
                         entry_price, direction, stop_loss_pct):
        """Simulate intraday trade with stop-loss."""
        # Get all bars from entry to close
        day_data = futures_data.loc[date]
        trade_bars = day_data.loc[entry_time:]
        
        stop_price = entry_price * (1 - direction * stop_loss_pct)
        
        for timestamp, bar in trade_bars.iterrows():
            # Check stop-loss
            if direction == 1 and bar['low'] <= stop_price:
                return -stop_loss_pct, stop_price, 'stop_loss'
            elif direction == -1 and bar['high'] >= stop_price:
                return -stop_loss_pct, stop_price, 'stop_loss'
        
        # Close at end of day
        close_price = trade_bars.iloc[-1]['close']
        pnl = direction * (close_price - entry_price) / entry_price
        return pnl, close_price, 'close_of_day'


class ConsistencyRangeBreakoutStrategy:
    """
    Consistency-controlled random range breakout strategy.
    
    Applies consistency filter to standard range breakout:
    - Only execute breakout trades when consistency is high
    - Uses Monte Carlo simulation (100 runs per day)
    """
    
    def __init__(self, range_min=0.003, range_max=0.01, 
                 n_simulations=100, lookback_days=60):
        self.range_min = range_min      # 0.3% minimum range
        self.range_max = range_max      # 1.0% maximum range
        self.n_sims = n_simulations     # 100 MC runs
        self.lookback_days = lookback_days
        
        self.calc = ConsistencyCalculator()
    
    def run_monte_carlo_day(self, futures_data, constituent_data, date,
                             use_consistency_filter=True):
        """Run 100 MC simulations for a single day."""
        daily_returns = []
        
        for _ in range(self.n_sims):
            # Random Range ~ Uniform(0.3%, 1.0%)
            rand_range = np.random.uniform(self.range_min, self.range_max)
            
            open_price = futures_data.loc[date].iloc[0]['open']
            upper_band = open_price * (1 + rand_range)
            lower_band = open_price * (1 - rand_range)
            
            # Simulate breakout
            ret = self._simulate_breakout(
                futures_data.loc[date], open_price, 
                upper_band, lower_band, constituent_data, date,
                use_consistency_filter
            )
            daily_returns.append(ret)
        
        return np.mean(daily_returns)
```

### Phase 3: Backtesting Framework (Week 3)

#### 3.4 Intraday Backtester
**File**: `src/index_momentum/backtester_intraday.py`

```python
class IntradayBacktester:
    """
    Backtester for intraday strategies.
    Handles tick/bar-level simulation with transaction costs.
    """
    
    def __init__(self, initial_capital=1_000_000):
        self.capital = initial_capital
        self.commission_rate = 0.0002    # 0.02% per side (pre-2015-09)
        self.leverage = 1.0              # No leverage (value-based returns)
    
    def run(self, strategy, futures_data, constituent_data):
        """Run full backtest."""
        signals = strategy.generate_signals(futures_data, constituent_data)
        
        # Apply transaction costs
        signals['net_return'] = signals['return'].apply(
            lambda r: r - 2 * self.commission_rate if r != 0 else 0
        )
        
        # Build equity curve
        signals['cumulative_return'] = (1 + signals['net_return']).cumprod()
        
        return signals
    
    def calculate_performance(self, results):
        """Calculate comprehensive performance metrics."""
        returns = results['net_return']
        traded = results[results['traded'] == True]['net_return']
        
        return {
            # Return metrics
            'total_return': (1 + returns).prod() - 1,
            'annual_return': self._annualize(returns),
            
            # Risk metrics
            'max_drawdown': self._max_drawdown(returns),
            'sharpe_ratio': self._sharpe(returns),
            'calmar_ratio': self._annualize(returns) / abs(self._max_drawdown(returns)),
            
            # Trade metrics
            'n_trades': len(traded),
            'win_rate': (traded > 0).mean(),
            'avg_return_per_trade': traded.mean(),
            'profit_loss_ratio': traded[traded > 0].mean() / abs(traded[traded < 0].mean()),
            
            # Consistency metrics
            'avg_R': results['R'].mean(),
            'trade_frequency': results['traded'].mean(),
        }
```

#### 3.5 Performance Analytics
**File**: `src/index_momentum/analytics.py`

**Key Metrics** (matching paper's Table 2):
- Annual return (年化收益率)
- Maximum drawdown (最大回撤率)
- Return/drawdown ratio (收益回撤比)
- Win rate (胜率)
- Average return per trade (单次交易平均收益率)
- Profit/loss ratio (盈亏比)

### Phase 4: Main Scripts (Week 4)

#### 3.6 Parameter Optimization Script
**File**: `optimize_consistency_params.py`

```python
"""
Parameter optimization for consistency strategy.
Optimizes T (entry time) using in-sample data.
"""

# In-sample: 2010-04-16 to 2013-12-31
# Test T from 21 to 60 minutes after open

results_grid = {}
for T in range(21, 61):
    strategy = ConsistencyMomentumStrategy(entry_time_minutes=T)
    backtester = IntradayBacktester()
    results = backtester.run(strategy, futures_data, constituent_data)
    metrics = backtester.calculate_performance(results)
    results_grid[T] = metrics

# Find optimal T (max return/drawdown ratio)
optimal_T = max(results_grid, key=lambda t: results_grid[t]['calmar_ratio'])
print(f"Optimal T = {optimal_T} minutes (expected: 24 min)")

# Plot sensitivity (replicate Figure 8 from paper)
plot_parameter_sensitivity(results_grid)
```

#### 3.7 Main Backtest Script
**File**: `backtest_consistency_strategy.py`

```python
"""
Main backtest script for constituent consistency strategy.
Tests on CSI 300 (IF) futures - expandable to IH, IC.
"""

def main():
    # ===== DATA LOADING =====
    data_wrapper = XbbgDataWrapper()
    
    print("Downloading IF futures data (1-minute bars)...")
    futures_data = data_wrapper.download_intraday_data(
        ticker="IFc1 Index",
        start_date="2010-04-16",
        end_date="2016-07-15",
        bar_size=1
    )
    
    print("Downloading CSI 300 constituent data (30-minute bars)...")
    constituent_data = data_wrapper.download_constituent_intraday(
        index_ticker="SHSZ300 Index",
        start_date="2010-04-16",
        end_date="2016-07-15",
        bar_size=30
    )
    
    # ===== PARAMETER OPTIMIZATION (IN-SAMPLE) =====
    print("\nRunning parameter optimization (2010-2013)...")
    in_sample_futures = futures_data.loc[:"2013-12-31"]
    in_sample_constituents = filter_by_date(constituent_data, end="2013-12-31")
    
    best_T = optimize_entry_time(in_sample_futures, in_sample_constituents)
    print(f"Optimal entry time T = {best_T} minutes")
    
    # ===== OUT-OF-SAMPLE BACKTEST =====
    print("\nRunning out-of-sample backtest (2014-2016)...")
    oos_futures = futures_data.loc["2014-01-01":"2016-07-15"]
    oos_constituents = filter_by_date(constituent_data, 
                                        start="2014-01-01", end="2016-07-15")
    
    # Strategy 1: Pure consistency strategy
    strategy1 = ConsistencyMomentumStrategy(entry_time_minutes=best_T)
    backtester = IntradayBacktester()
    results1 = backtester.run(strategy1, oos_futures, oos_constituents)
    metrics1 = backtester.calculate_performance(results1)
    
    # Strategy 2: Consistency range breakout
    strategy2 = ConsistencyRangeBreakoutStrategy()
    results2 = backtester.run(strategy2, oos_futures, oos_constituents)
    metrics2 = backtester.calculate_performance(results2)
    
    # Benchmark: Simple range breakout (no consistency filter)
    strategy_bench = ConsistencyRangeBreakoutStrategy(use_filter=False)
    results_bench = backtester.run(strategy_bench, oos_futures, oos_constituents)
    metrics_bench = backtester.calculate_performance(results_bench)
    
    # ===== RESULTS =====
    print("\n" + "="*60)
    print("OUT-OF-SAMPLE RESULTS (2014-2016)")
    print("="*60)
    
    comparison_table = pd.DataFrame({
        'Consistency Momentum': metrics1,
        'Consistency + Breakout': metrics2,
        'Simple Breakout': metrics_bench,
    }).T
    
    print(comparison_table[['annual_return', 'max_drawdown', 
                              'calmar_ratio', 'win_rate', 'avg_return_per_trade']])
    
    # Save results and plots
    save_results(results1, results2, results_bench, comparison_table)

if __name__ == "__main__":
    main()
```

#### 3.8 Configuration File
**File**: `config/consistency_config.yaml`

```yaml
# Constituent Consistency Strategy Configuration

instruments:
  primary:
    futures_ticker: "IFc1 Index"
    index_ticker: "SHSZ300 Index"
    name: "CSI 300 (IF)"
  
  extensions:
    - futures_ticker: "IHc1 Index"
      index_ticker: "SHY000016 Index"
      name: "SSE 50 (IH)"
    - futures_ticker: "ICc1 Index"
      index_ticker: "CSI500 Index"
      name: "CSI 500 (IC)"

backtest:
  in_sample_start: "2010-04-16"
  in_sample_end: "2013-12-31"
  out_of_sample_start: "2014-01-01"
  out_of_sample_end: "2016-07-15"
  extended_end: "2024-12-31"

strategy_params:
  consistency_momentum:
    entry_time_range: [21, 60]    # Minutes after 9:30 (optimize)
    optimal_T: 24                  # Optimal from in-sample (paper result)
    stop_loss_pct: 0.006           # 0.6% stop loss
    lookback_days: 60              # Threshold calculation window
    window_bars: 160               # 20 days × 8 thirty-min bars
  
  range_breakout:
    range_min: 0.003               # 0.3% minimum Range
    range_max: 0.010               # 1.0% maximum Range
    n_simulations: 100             # Monte Carlo runs per day
    stop_loss_pct: 0.006           # 0.6% stop loss

costs:
  commission_pre_sep2015: 0.0002   # 0.02% per side (before 2015-09-07)
  commission_post_sep2015: 0.0023  # Increased fees after 2015-09-07
  use_bottom_position: true        # Keep overnight position to avoid 平今 fee

bloomberg:
  bar_size_futures: 1              # 1-minute bars for futures
  bar_size_constituents: 30        # 30-minute bars for consistency calc
  cache_dir: "data/cache"
  use_cache: true

output:
  results_dir: "results/consistency"
  plots_dir: "results/consistency/plots"
```

---

## 4. Testing Methodology

### 4.1 Replicating Paper Results

**Validation targets** (from paper):

| Metric | In-Sample (2010-2013) | Out-of-Sample (2014-2016) |
|--------|----------------------|--------------------------|
| Cumulative return | 141.1% | 72.6% |
| Annual return | 26.8% | 23.1% |
| Max drawdown | -6.3% | -11.7% |
| Return/DD ratio | 4.24 | 1.97 |
| Win rate | 47.5% | 36.8% |
| Avg trade return | 0.20% | 0.19% |

### 4.2 Parameter Sensitivity

**Replicate Figure 8**: Plot return/drawdown ratio vs T (21-60 minutes)
- Expected: peak around T=24, generally robust across range
- All parameters should yield positive returns

**Replicate Table 1**: Near-optimal parameter performance
```
T=21: +24.2% annual, -8.7% DD, 2.79 ratio
T=24: +26.8% annual, -6.3% DD, 4.24 ratio (optimal)
T=29: +23.7% annual, -7.0% DD, 3.37 ratio
```

### 4.3 Control Experiments

**Experiment 1**: Trade only when consistency is LOW (inverse strategy)
- Expected: negative or near-zero returns (confirms filter works)

**Experiment 2**: T-1 minute calculation (pre-compute R)
- Expected: similar performance to T-minute calculation
- Confirms strategy doesn't require exact real-time computation

**Experiment 3**: Extended out-of-sample (2017-2024)
- Tests strategy robustness in recent market conditions

### 4.4 Breakout Strategy Comparison

**Replicate Table 4**:
```
                          Random Breakout   Controlled Breakout
2014 return:              -3.16%            -4.78%
2015 return:              +58.63%           +69.55%  (+10.92%)
2016 return:              +1.38%            +4.24%
Avg daily return:         0.08%             0.10%
```

---

## 5. Expected Deliverables

### 5.1 Code Structure

```
index_momentum/
├── src/
│   └── index_momentum/
│       ├── data_xbbg.py              # Bloomberg data (OHLCV + intraday)
│       ├── consistency.py            # PCA consistency calculator
│       ├── strategies_intraday.py    # Consistency momentum strategies
│       ├── backtester_intraday.py    # Intraday backtesting engine
│       └── analytics.py             # Performance metrics
├── config/
│   └── consistency_config.yaml
├── optimize_consistency_params.py    # Parameter optimization script
├── backtest_consistency_strategy.py  # Main backtest script
├── notebooks/
│   ├── 01_pca_consistency_analysis.ipynb    # Verify indicator
│   ├── 02_parameter_optimization.ipynb      # Reproduce Figure 8
│   ├── 03_consistency_strategy_results.ipynb # Reproduce Figures 9-10
│   └── 04_range_breakout_comparison.ipynb   # Reproduce Figure 17
└── results/consistency/
    ├── in_sample_results.csv
    ├── out_of_sample_results.csv
    ├── parameter_sensitivity.csv
    └── plots/
```

### 5.2 Visualization Outputs

1. **Consistency Time Series**: R over time with CSI 300 price overlay (Figure 5)
2. **Parameter Sensitivity**: Return/drawdown ratio vs T (Figure 8)
3. **In-sample Equity Curve**: 2010-2013 cumulative returns (Figure 9)
4. **Out-of-sample Equity Curve**: 2014-2016 cumulative returns (Figure 10)
5. **Daily R Values**: Consistency indicator across trading days (Figure 11)
6. **Intraday R Profile**: Single day consistency dynamics (Figure 12)
7. **Control Experiment**: Inverse strategy performance (Figures 14-15)
8. **Breakout Comparison**: Standard vs controlled breakout (Figure 17)

---

## 6. Comparison with Dispersion Strategy

This strategy should be compared with the dispersion strategy from the companion paper:

| Feature | Dispersion Strategy | Consistency Strategy |
|---------|--------------------|--------------------|
| Frequency | Weekly | Intraday |
| Signal | Momentum (4-week) | Intraday momentum (T-min) |
| Filter | Dispersion ratio | PCA R indicator |
| Instrument | ETFs/Index | Futures |
| Holding period | 1 week | Intraday only |
| Cost sensitivity | Lower | Higher (intraday) |
| Leverage | None | None (value-based) |

**Combined Strategy Potential**: 
- Dispersion strategy for weekly positioning
- Consistency strategy for intraday timing of entries/exits

---

## 7. Risk Considerations

### 7.1 Market Structure Risks

**2015-09-07 Rule Change**: CFFEX increased transaction fees significantly
- Strategy must account for post-restriction trading costs
- Holding overnight position ("底仓") strategy to avoid 平今仓 fees

**Circuit Breakers**: January 2016 circuit breaker episode
- Intraday strategies face early market halt risk
- Need to handle incomplete trading days

### 7.2 Computational Complexity

**Challenge**: PCA on 300×160 matrix per bar in real-time
- Eigenvalue decomposition: O(N³) = O(300³) ≈ 27 million operations
- Mitigation: Pre-compute at T-1 minute (shown to work equally well)
- Alternative: Incremental PCA (sklearn.decomposition.IncrementalPCA)

**Optimization**:
```python
# Use efficient eigenvalue solver
from scipy.linalg import eigh  # More efficient for symmetric matrices

eigenvalues = eigh(sigma, eigvals_only=True)  # Sorted ascending
R = eigenvalues[-1] / eigenvalues.sum()       # Last = largest
```

### 7.3 Data Quality

**1-minute constituent data**: Large volume (~300 stocks × 240 bars/day)
- Requires efficient caching (use parquet format)
- Missing bars for suspended stocks (exclude from calculation)
- Price adjustments for dividends/splits

---

## 8. Timeline

### Week 1: Data Infrastructure
- [ ] Extend xbbg wrapper for intraday bar data
- [ ] Download IF futures historical 1-minute data (2010-2016)
- [ ] Download CSI 300 constituent 30-minute data
- [ ] Validate data quality and completeness

### Week 2: Consistency Calculation
- [ ] Implement PCA consistency calculator
- [ ] Validate R values against paper's charts (Figure 11)
- [ ] Profile performance and optimize (scipy.linalg.eigh)
- [ ] Test T-1 minute calculation approach

### Week 3: Strategy Implementation
- [ ] Implement ConsistencyMomentumStrategy
- [ ] Implement ConsistencyRangeBreakoutStrategy
- [ ] Implement IntradayBacktester with cost modeling
- [ ] Unit test all components

### Week 4: Backtesting & Validation
- [ ] Run parameter optimization (replicate Figure 8)
- [ ] Run in-sample backtest (replicate Figures 9, Table 2)
- [ ] Run out-of-sample backtest (replicate Figure 10)
- [ ] Run control experiments (replicate Figures 14-15)
- [ ] Run breakout comparison (replicate Figure 17, Table 4)

### Week 5: Analysis & Extension
- [ ] Extended out-of-sample test (2017-2024)
- [ ] Compare with dispersion strategy
- [ ] Generate notebooks and visualization
- [ ] Write summary report

---

## 9. Future Extensions

### 9.1 Real-Time Implementation
- Stream constituent prices via Bloomberg API
- Compute PCA at T-1 minute (proven equivalent)
- Auto-execute via broker API when signal triggers

### 9.2 Multi-Instrument Extension
- Apply to IH (SSE 50) and IC (CSI 500/1000) futures
- Portfolio approach: trade all three simultaneously
- Risk-parity position sizing across contracts

### 9.3 Adaptive Threshold
- Dynamic lookback window based on market regime
- ML-based threshold optimization
- Regime detection to adjust lookback_days

### 9.4 Hybrid Strategy
- Combine weekly dispersion signal (from companion paper)
- Use consistency indicator for intraday entry timing
- Multi-timeframe framework

### 9.5 Expanded Consistency Measures
- Test number of principal components (top-k PCs, not just top-1)
- Test alternative: correlation-based consistency
- Test rolling window sensitivity (L = 80, 120, 160, 200 bars)

---

## 10. References

1. **Original Paper**: "从成份股一致性来衡量市场的趋势 - 另类交易策略系列之三十二" (GF Securities, September 2016)
2. **Related Papers**:
   - Series #14: EMDT策略 (Empirical Mode Decomposition Trend, 2014)
   - Series #15: MESA策略 (Maximum Entropy Spectral Analysis, 2014)
   - Series #18: Random Range Breakout (Thick-tail, 2014)
   - Series #21: Market Sentiment Smoothness Strategy (2015)
3. **Companion Paper**: "基于股票分化度的指数趋势策略" (Guosen Securities, 2017)
4. **PCA Reference**: Jolliffe, I.T. (2002). Principal Component Analysis. Springer.
5. **xbbg Documentation**: https://github.com/alpha-xone/xbbg
6. **Data Source**: Bloomberg Terminal + CFFEX (China Financial Futures Exchange)

---

## Appendix A: PCA Mathematical Background

### Eigenvalue Decomposition
For symmetric covariance matrix Σ (N×N):
```
Σ = U Λ Uᵀ
```
Where:
- U = eigenvector matrix (columns are eigenvectors u₁, u₂, ..., uN)
- Λ = diagonal matrix of eigenvalues λ₁ ≥ λ₂ ≥ ... ≥ λN ≥ 0

### Variance Explained
The i-th principal component explains fraction `λᵢ / Σλⱼ` of total variance.

### Consistency Indicator
```python
R = λ₁ / sum(λ) × 100%
```
- R = 100%: Perfect consistency (all stocks move identically)
- R = 1/N × 100%: Perfect independence (all eigenvalues equal)

### Example Values (from paper)
- High consistency example: R = 86.4%
- Low consistency example: R = 45.2%
- Typical CSI 300 range: 50% - 80%
- Extreme market (2015): R > 90%

---

## Appendix B: Transaction Cost Analysis

### Pre-2015-09-07
- Exchange commission: ~0.023% per side (front-month IF)
- Strategy uses: 0.02% per side = 0.04% round-trip

### Post-2015-09-07 (CFFEX restrictions)
- 平今仓 (same-day close): 23% per side
- Overnight close (平昨仓): 0.023% per side
- Strategy: Must maintain bottom position to avoid 平今 fees
- Effective cost with bottom position: ~0.046% round-trip

### Impact on Strategy
- Single trade avg return: 0.20% >> transaction cost 0.04%
- Strategy is relatively cost-insensitive
- Works even with post-2015 restrictions

---

**Document Version**: 1.0  
**Last Updated**: March 15, 2026  
**Status**: Ready for Implementation  
**Related Document**: `../dispersion/backtest_plan.md`
