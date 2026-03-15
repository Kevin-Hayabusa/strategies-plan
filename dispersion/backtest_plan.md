# Dispersion-Based Index Momentum Strategy - Backtest Implementation Plan

## Executive Summary

This document outlines the implementation plan for backtesting the **stock dispersion-enhanced momentum strategy** as described in the research paper "基于股票分化度的指数趋势策略" (Guosen Securities, November 2017).

**Target Indices**: CSI 300, CSI 500, CSI 1000  
**Data Source**: Bloomberg Terminal (xbbg Python library)  
**Test Period**: 2008-2017 (in-sample), 2018-present (out-of-sample)  
**Strategy Core**: 4-week momentum with dispersion risk filter

---

## 1. Strategy Theory

### 1.1 Stock Dispersion (Return Dispersion)

**Definition**: Stock dispersion measures the degree of divergence in returns among index constituents at a given time.

**Mathematical Formula**:
```
RD_t = std(R_i,t)
```
Where:
- `RD_t` = Return dispersion at time t
- `R_i,t` = Return of constituent stock i at time t
- `std()` = Standard deviation across all constituents

**Interpretation**:
- **High dispersion**: Individual stocks moving in different directions → stock selection opportunities
- **Low dispersion**: Stocks moving together → market-driven movements
- **Dispersion changes**: Can signal trend exhaustion or continuation

### 1.2 Simple Momentum Strategy (Baseline)

**Signal Generation**:
```python
if return_past_N_weeks > 0:
    position = LONG
else:
    position = CASH
```

**Parameters**:
- Lookback period: N = 3, 4, 6, 8, 10 weeks
- Rebalance frequency: Weekly (Friday close)
- Execution: Monday open price

**Characteristics**:
- Right-side trading (reactive)
- Trend-following approach
- Prone to whipsaws in choppy markets

### 1.3 Dispersion-Enhanced Momentum Strategy

**Core Innovation**: Use dispersion changes as a risk filter to detect trend exhaustion.

**Risk Signal Logic**:
```python
# Calculate week-over-week changes
delta_index_return = index_return[t] - index_return[t-1]
delta_dispersion = dispersion[t] - dispersion[t-1]

# Risk threshold
if abs(delta_index_return) > N * abs(delta_dispersion):
    # Small dispersion change causing large index move
    # → Potential trend exhaustion
    position = CASH  # Exit or stay out
else:
    # Normal market condition
    # → Follow momentum signal
    position = momentum_signal
```

**Key Insight**: When a small change in constituent dispersion leads to a large index movement, it suggests:
- Concentrated moves in few stocks (not broad-based)
- Potential reversal risk
- Time to reduce exposure

**Parameters**:
- `N` (dispersion multiplier): Typically 1-3
- Higher N = less sensitive filter (fewer exits)
- Lower N = more conservative (more frequent exits)

---

## 2. Data Requirements

### 2.1 Bloomberg Data Fields

#### Index Level Data
```python
# Bloomberg tickers
CSI_300 = "SHSZ300 Index"
CSI_500 = "CSI500 Index"  
CSI_1000 = "CSI1000 Index"

# Required fields
fields = [
    "PX_OPEN",      # Open price
    "PX_HIGH",      # High price
    "PX_LOW",       # Low price
    "PX_LAST",      # Close price
    "PX_VOLUME",    # Volume
]
```

#### Constituent Data
```python
# Get constituent list at each rebalance date
constituents = blp.bds(
    tickers="SHSZ300 Index",
    flds="INDX_MEMBERS",
    INDX_MWEIGHT_HIST_DT="20200101"  # Historical date
)

# For each constituent, download OHLCV
for stock in constituents:
    data = blp.bdh(
        tickers=stock,
        flds=["PX_OPEN", "PX_HIGH", "PX_LOW", "PX_LAST", "PX_VOLUME"],
        start_date="20080101",
        end_date="20171231"
    )
```

### 2.2 Data Frequency and Alignment

- **Primary frequency**: Daily data
- **Resampling**: Aggregate to weekly (Friday close)
- **Alignment**: Ensure constituent data matches index rebalance dates
- **Handling**: 
  - Missing data: Forward fill (max 5 days)
  - Suspended stocks: Exclude from dispersion calculation
  - New listings: Include only after 60 trading days

### 2.3 Historical Constituent Lists

**Challenge**: Index constituents change over time (quarterly rebalancing)

**Solution**:
```python
# Download historical constituent lists
rebalance_dates = get_index_rebalance_dates("SHSZ300 Index", "2008-2017")

for date in rebalance_dates:
    constituents[date] = blp.bds(
        "SHSZ300 Index",
        "INDX_MEMBERS",
        INDX_MWEIGHT_HIST_DT=date
    )
```

**Survivorship Bias**: Using historical constituent lists avoids look-ahead bias

---

## 3. Implementation Roadmap

### Phase 1: Data Infrastructure (Week 1-2)

#### 3.1 xbbg Data Wrapper
**File**: `src/index_momentum/data_xbbg.py`

**Functions**:
```python
class XbbgDataWrapper:
    """Bloomberg data wrapper using xbbg library."""
    
    def download_index_data(self, ticker, start_date, end_date, fields):
        """Download index OHLCV data."""
        pass
    
    def download_constituents(self, index_ticker, date):
        """Get constituent list at specific date."""
        pass
    
    def download_constituent_data(self, tickers, start_date, end_date):
        """Download OHLCV for multiple stocks."""
        pass
    
    def resample_to_weekly(self, daily_data):
        """Convert daily to weekly frequency (Friday close)."""
        pass
    
    def cache_data(self, data, cache_key):
        """Cache downloaded data to avoid repeated API calls."""
        pass
```

**Key Features**:
- Batch downloading with progress tracking
- Automatic retry on API failures
- Local caching (pickle/parquet format)
- Data validation and cleaning

#### 3.2 Dispersion Calculator
**File**: `src/index_momentum/dispersion.py`

```python
class DispersionCalculator:
    """Calculate stock return dispersion for indices."""
    
    def calculate_weekly_returns(self, constituent_data):
        """Calculate weekly returns for each constituent."""
        pass
    
    def calculate_dispersion(self, weekly_returns):
        """Calculate cross-sectional standard deviation."""
        # RD_t = std(R_i,t) across all constituents
        pass
    
    def calculate_dispersion_change(self, dispersion_series):
        """Calculate week-over-week dispersion changes."""
        pass
    
    def detect_risk_signal(self, index_returns, dispersion, multiplier=2):
        """Detect trend exhaustion risk using dispersion filter."""
        pass
```

**Validation**:
- Compare calculated dispersion with paper's charts (Figures 1-4)
- Verify dispersion spikes during 2009, 2012, 2014-2015 bull markets

### Phase 2: Strategy Implementation (Week 2-3)

#### 3.3 Backtrader Strategy Classes
**File**: `src/index_momentum/strategies.py`

**Strategy 1: Simple Momentum (Baseline)**
```python
class SimpleMomentumStrategy(bt.Strategy):
    """4-week momentum strategy without dispersion filter."""
    
    params = (
        ('lookback_weeks', 4),
        ('rebalance_day', 4),  # Friday = 4
    )
    
    def __init__(self):
        self.momentum = WeeklyMomentum(
            self.data.close, 
            period=self.params.lookback_weeks
        )
    
    def next(self):
        if self.is_rebalance_day():
            if self.momentum[0] > 0:
                self.order_target_percent(target=1.0)  # 100% long
            else:
                self.order_target_percent(target=0.0)  # Cash
```

**Strategy 2: Dispersion-Enhanced Momentum**
```python
class DispersionMomentumStrategy(bt.Strategy):
    """4-week momentum with dispersion risk filter."""
    
    params = (
        ('lookback_weeks', 4),
        ('dispersion_multiplier', 2),
        ('rebalance_day', 4),
    )
    
    def __init__(self):
        self.momentum = WeeklyMomentum(self.data.close, period=self.p.lookback_weeks)
        self.dispersion = DispersionIndicator(self.data)  # Custom indicator
        self.index_return = bt.indicators.ROC(self.data.close, period=1)
        
    def next(self):
        if self.is_rebalance_day():
            # Calculate risk signal
            delta_return = abs(self.index_return[0] - self.index_return[-1])
            delta_dispersion = abs(self.dispersion[0] - self.dispersion[-1])
            
            risk_signal = delta_return > self.p.dispersion_multiplier * delta_dispersion
            
            if risk_signal:
                # Trend exhaustion risk → exit
                self.order_target_percent(target=0.0)
            else:
                # Normal condition → follow momentum
                if self.momentum[0] > 0:
                    self.order_target_percent(target=1.0)
                else:
                    self.order_target_percent(target=0.0)
```

#### 3.4 Custom Indicators
**File**: `src/index_momentum/indicators.py`

```python
class DispersionIndicator(bt.Indicator):
    """Backtrader indicator for stock dispersion."""
    
    lines = ('dispersion',)
    params = (('constituent_data', None),)
    
    def __init__(self):
        # Load pre-calculated dispersion data
        self.dispersion_data = self.params.constituent_data
    
    def next(self):
        # Map current date to dispersion value
        current_date = self.data.datetime.date(0)
        self.lines.dispersion[0] = self.dispersion_data.loc[current_date]


class WeeklyMomentum(bt.Indicator):
    """Weekly momentum indicator."""
    
    lines = ('momentum',)
    params = (('period', 4),)
    
    def __init__(self):
        self.momentum = self.data.close / self.data.close(-self.p.period) - 1
```

### Phase 3: Backtesting Framework (Week 3-4)

#### 3.5 Enhanced Backtester
**File**: `src/index_momentum/backtester.py`

**Enhancements**:
```python
class DispersionBacktester(BacktestEngine):
    """Enhanced backtester for dispersion strategies."""
    
    def add_dispersion_data(self, dispersion_series):
        """Add dispersion data as auxiliary feed."""
        pass
    
    def set_weekly_rebalancing(self):
        """Configure weekly rebalancing schedule."""
        pass
    
    def set_transaction_costs(self, commission=0.001, slippage=0.001):
        """Set realistic transaction costs (0.2% round-trip)."""
        self.cerebro.broker.setcommission(commission=commission)
        # Add slippage model
        pass
    
    def run_parameter_sweep(self, param_grid):
        """Run strategy with multiple parameter combinations."""
        pass
```

#### 3.6 Analytics Module
**File**: `src/index_momentum/analytics.py`

```python
class PerformanceAnalyzer:
    """Comprehensive performance analysis."""
    
    def calculate_metrics(self, results):
        """Calculate key performance metrics."""
        return {
            'total_return': self._total_return(results),
            'annual_return': self._annual_return(results),
            'sharpe_ratio': self._sharpe_ratio(results),
            'max_drawdown': self._max_drawdown(results),
            'win_rate': self._win_rate(results),
            'calmar_ratio': self._calmar_ratio(results),
        }
    
    def compare_strategies(self, results_dict):
        """Compare multiple strategies side-by-side."""
        pass
    
    def plot_equity_curves(self, results_dict):
        """Plot equity curves for comparison."""
        pass
    
    def plot_drawdowns(self, results_dict):
        """Plot drawdown analysis."""
        pass
    
    def generate_report(self, results_dict, output_path):
        """Generate comprehensive PDF/HTML report."""
        pass
```

**Key Metrics** (matching paper's Table 2 & 3):
- Annual return (年化收益)
- Maximum drawdown (最大回撤)
- Win rate (周胜率)
- Sharpe ratio
- Calmar ratio (return/max_drawdown)

### Phase 4: Execution & Analysis (Week 4-5)

#### 3.7 Main Backtest Script
**File**: `backtest_dispersion_strategy.py`

```python
"""
Main script to backtest dispersion-enhanced momentum strategy
on CSI 300, CSI 500, and CSI 1000 indices.
"""

import pandas as pd
from src.index_momentum.data_xbbg import XbbgDataWrapper
from src.index_momentum.dispersion import DispersionCalculator
from src.index_momentum.backtester import DispersionBacktester
from src.index_momentum.strategies import SimpleMomentumStrategy, DispersionMomentumStrategy
from src.index_momentum.analytics import PerformanceAnalyzer

def main():
    # Configuration
    indices = {
        'CSI300': 'SHSZ300 Index',
        'CSI500': 'CSI500 Index',
        'CSI1000': 'CSI1000 Index',
    }
    
    start_date = '2008-01-01'
    end_date = '2017-12-31'
    
    # Step 1: Download data
    print("Downloading data from Bloomberg...")
    data_wrapper = XbbgDataWrapper()
    
    for name, ticker in indices.items():
        print(f"\nProcessing {name}...")
        
        # Download index data
        index_data = data_wrapper.download_index_data(
            ticker, start_date, end_date
        )
        
        # Download constituent data
        constituents = data_wrapper.download_constituents_history(
            ticker, start_date, end_date
        )
        
        constituent_data = data_wrapper.download_constituent_data(
            constituents, start_date, end_date
        )
        
        # Step 2: Calculate dispersion
        print(f"Calculating dispersion for {name}...")
        calc = DispersionCalculator()
        dispersion = calc.calculate_dispersion(constituent_data)
        
        # Step 3: Run backtests
        print(f"Running backtests for {name}...")
        
        # Baseline: Simple momentum
        bt_simple = DispersionBacktester()
        bt_simple.add_data(index_data)
        bt_simple.add_strategy(SimpleMomentumStrategy, lookback_weeks=4)
        bt_simple.set_transaction_costs(commission=0.001)
        results_simple = bt_simple.run()
        
        # Enhanced: Dispersion momentum
        bt_dispersion = DispersionBacktester()
        bt_dispersion.add_data(index_data)
        bt_dispersion.add_dispersion_data(dispersion)
        bt_dispersion.add_strategy(
            DispersionMomentumStrategy, 
            lookback_weeks=4,
            dispersion_multiplier=2
        )
        bt_dispersion.set_transaction_costs(commission=0.001)
        results_dispersion = bt_dispersion.run()
        
        # Step 4: Analyze results
        print(f"Analyzing results for {name}...")
        analyzer = PerformanceAnalyzer()
        
        comparison = analyzer.compare_strategies({
            'Buy & Hold': index_data,
            'Simple Momentum': results_simple,
            'Dispersion Momentum': results_dispersion,
        })
        
        print(comparison)
        
        # Generate plots
        analyzer.plot_equity_curves({
            'Buy & Hold': index_data,
            'Simple Momentum': results_simple,
            'Dispersion Momentum': results_dispersion,
        }, save_path=f'results/{name}_equity_curve.png')
        
        # Save results
        comparison.to_csv(f'results/{name}_performance.csv')
    
    print("\nBacktest complete! Results saved to 'results/' folder.")

if __name__ == "__main__":
    main()
```

#### 3.8 Configuration File
**File**: `config/dispersion_config.yaml`

```yaml
# Dispersion Strategy Configuration

indices:
  CSI300:
    ticker: "SHSZ300 Index"
    name: "沪深300"
  CSI500:
    ticker: "CSI500 Index"
    name: "中证500"
  CSI1000:
    ticker: "CSI1000 Index"
    name: "中证1000"

backtest:
  start_date: "2008-01-01"
  end_date: "2017-12-31"
  initial_cash: 100000
  rebalance_frequency: "weekly"  # Friday close
  execution_delay: 1  # Execute at Monday open

costs:
  commission: 0.001  # 0.1% per side
  slippage: 0.001    # 0.1% slippage
  # Total round-trip cost: 0.2%

strategy_params:
  simple_momentum:
    lookback_weeks: [3, 4, 6, 8, 10]
  
  dispersion_momentum:
    lookback_weeks: [3, 4, 6, 8, 10]
    dispersion_multiplier: [1, 2, 3]

bloomberg:
  fields:
    - PX_OPEN
    - PX_HIGH
    - PX_LOW
    - PX_LAST
    - PX_VOLUME
  
  cache_dir: "data/cache"
  use_cache: true

output:
  results_dir: "results"
  plots_dir: "results/plots"
  reports_dir: "results/reports"
```

---

## 4. Testing Methodology

### 4.1 Validation Steps

#### Step 1: Data Validation
- [ ] Verify index data matches Bloomberg terminal
- [ ] Check constituent lists against official index provider
- [ ] Validate dispersion calculations against paper's charts

#### Step 2: Strategy Logic Validation
- [ ] Test simple momentum on known periods (2015 bull market)
- [ ] Verify dispersion filter triggers during 2015 crash
- [ ] Compare signals with paper's strategy description

#### Step 3: Performance Validation
- [ ] Compare results with paper's Tables 2 & 3
- [ ] Verify annual returns within reasonable range
- [ ] Check max drawdown matches expected values

### 4.2 Parameter Sensitivity Analysis

**Test Grid**:
```python
param_grid = {
    'lookback_weeks': [3, 4, 6, 8, 10],
    'dispersion_multiplier': [1, 1.5, 2, 2.5, 3],
}
```

**Expected Results** (from paper):
- 4-week lookback performs best
- Dispersion multiplier ~2 optimal
- Strategy robust across parameter ranges

### 4.3 Out-of-Sample Testing

**In-Sample**: 2008-2017 (strategy development)  
**Out-of-Sample**: 2018-2024 (validation)

**Purpose**: Verify strategy doesn't overfit to historical data

---

## 5. Expected Deliverables

### 5.1 Code Deliverables

```
index_momentum/
├── src/
│   └── index_momentum/
│       ├── data_xbbg.py          # Bloomberg data wrapper
│       ├── dispersion.py         # Dispersion calculator
│       ├── indicators.py         # Custom backtrader indicators
│       ├── strategies.py         # Strategy implementations
│       ├── backtester.py         # Enhanced backtesting engine
│       └── analytics.py          # Performance analysis
├── config/
│   └── dispersion_config.yaml    # Configuration file
├── backtest_dispersion_strategy.py  # Main execution script
├── notebooks/
│   ├── 01_data_exploration.ipynb    # Data analysis
│   ├── 02_dispersion_analysis.ipynb # Dispersion characteristics
│   └── 03_strategy_results.ipynb    # Results visualization
└── results/
    ├── CSI300_performance.csv
    ├── CSI500_performance.csv
    ├── CSI1000_performance.csv
    └── plots/
```

### 5.2 Analysis Deliverables

#### Performance Summary Table
| Index | Strategy | Annual Return | Max Drawdown | Sharpe | Win Rate |
|-------|----------|---------------|--------------|--------|----------|
| CSI300 | Buy & Hold | X% | -Y% | Z | N/A |
| CSI300 | Simple Momentum | X% | -Y% | Z | W% |
| CSI300 | Dispersion Momentum | X% | -Y% | Z | W% |
| ... | ... | ... | ... | ... | ... |

#### Visualization Outputs
1. **Equity Curves**: Compare all strategies
2. **Drawdown Analysis**: Underwater plots
3. **Dispersion Time Series**: Overlay with index
4. **Signal Analysis**: Entry/exit points
5. **Parameter Heatmaps**: Sensitivity analysis

#### Research Report
- Executive summary
- Strategy description
- Data methodology
- Backtest results
- Parameter sensitivity
- Conclusions and recommendations

---

## 6. Risk Considerations

### 6.1 Data Quality Risks

**Issue**: Missing or incorrect constituent data  
**Mitigation**: 
- Cross-validate with multiple sources
- Implement data quality checks
- Handle missing data conservatively

**Issue**: Survivorship bias  
**Mitigation**: 
- Use historical constituent lists
- Include delisted stocks in calculations

### 6.2 Implementation Risks

**Issue**: Look-ahead bias  
**Mitigation**: 
- Ensure all signals use only past data
- Execution delay (signal Friday → execute Monday)

**Issue**: Transaction cost sensitivity  
**Mitigation**: 
- Test with multiple cost assumptions (0.1%, 0.2%, 0.3%)
- Include slippage modeling

### 6.3 Strategy Risks

**Issue**: Overfitting to historical data  
**Mitigation**: 
- Out-of-sample testing (2018+)
- Parameter robustness checks
- Walk-forward analysis

**Issue**: Regime changes  
**Mitigation**: 
- Test across multiple market regimes
- Monitor strategy performance degradation

---

## 7. Timeline

### Week 1: Data Infrastructure
- [ ] Set up xbbg connection
- [ ] Implement data download functions
- [ ] Download and cache all required data
- [ ] Validate data quality

### Week 2: Dispersion Calculation
- [ ] Implement dispersion calculator
- [ ] Validate against paper's charts
- [ ] Create dispersion analysis notebook

### Week 3: Strategy Implementation
- [ ] Implement simple momentum strategy
- [ ] Implement dispersion momentum strategy
- [ ] Create custom backtrader indicators
- [ ] Unit test strategy logic

### Week 4: Backtesting
- [ ] Run backtests on all three indices
- [ ] Parameter sensitivity analysis
- [ ] Generate performance reports

### Week 5: Analysis & Documentation
- [ ] Create visualization notebooks
- [ ] Write research report
- [ ] Document code
- [ ] Prepare presentation

---

## 8. Future Extensions

### 8.1 ETF Implementation
- Apply strategy to liquid ETFs (510300, 510500)
- Real-time signal generation
- Transaction cost optimization

### 8.2 Futures Implementation
- Adapt for IH, IF, IC contracts
- Handle contract rollover
- Leverage management

### 8.3 Multi-Strategy Portfolio
- Combine multiple momentum lookbacks
- Dynamic parameter selection
- Risk parity allocation

### 8.4 Machine Learning Enhancement
- Predict dispersion changes with ML
- Optimize dispersion multiplier dynamically
- Regime detection algorithms

---

## 9. References

1. **Original Paper**: "基于股票分化度的指数趋势策略" (Guosen Securities, 2017)
2. **xbbg Documentation**: https://github.com/alpha-xone/xbbg
3. **Backtrader Documentation**: https://www.backtrader.com/
4. **Index Providers**:
   - CSI Index Company: http://www.csindex.com.cn/
   - China Securities Index: Historical constituent data

---

## 10. Contact & Support

**Project Lead**: Kevin  
**Repository**: https://github.com/Kevin-Hayabusa/strategies-plan  
**Questions**: Open GitHub issues or contact via email

---

## Appendix A: Bloomberg Field Codes

```python
# Index data fields
INDEX_FIELDS = {
    'PX_OPEN': 'Open price',
    'PX_HIGH': 'High price',
    'PX_LOW': 'Low price',
    'PX_LAST': 'Close price',
    'PX_VOLUME': 'Trading volume',
    'PX_LAST_EOD': 'End of day close',
}

# Constituent fields
CONSTITUENT_FIELDS = {
    'INDX_MEMBERS': 'Index members',
    'INDX_MWEIGHT': 'Member weights',
    'INDX_MWEIGHT_HIST': 'Historical weights',
}

# Corporate actions
CORPORATE_ACTIONS = {
    'DVD_HIST_ALL': 'Dividend history',
    'SPLIT_HIST': 'Split history',
}
```

## Appendix B: Expected Performance Benchmarks

Based on the paper's results (2008-2017):

### CSI 300
- **Simple 4-week momentum**: ~15-20% annual return, -16% max drawdown
- **Dispersion momentum**: ~16-20% annual return, -11-16% max drawdown
- **Improvement**: Lower drawdown, higher Sharpe ratio

### CSI 500
- **Simple 4-week momentum**: ~20-25% annual return, -23% max drawdown
- **Dispersion momentum**: ~22-25% annual return, -20-23% max drawdown
- **Improvement**: Smoother equity curve, better risk-adjusted returns

---

**Document Version**: 1.0  
**Last Updated**: March 15, 2026  
**Status**: Ready for Implementation
