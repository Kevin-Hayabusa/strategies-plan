# Trading Strategies — Plan Repository

> **Purpose**: Research and planning repository for quantitative trading ideas.
> Each strategy folder contains a source paper (PDF/Word) and a detailed implementation plan (`.md`).
> Selected strategies are implemented in a separate coding repository.

---

## Strategy Index

| # | Strategy | Type | Instrument | Frequency | Source | Status |
|---|----------|------|------------|-----------|--------|--------|
| 1 | [Dispersion Momentum](#1-dispersion-momentum) | Trend-following | CSI 300/500/1000 ETF | Weekly | Guosen Securities, 2017 | Ready |
| 2 | [Constituent Consistency](#2-constituent-consistency) | Trend-following | CSI 300 Futures (IF) | Intraday | GF Securities, 2016 | Ready |
| 3 | [Index Money Flow](#3-index-money-flow) | Cross-country L/S | HSI/CSI/KOSPI/NKY/TAIEX Futures | Weekly | Changjiang Securities, 2020 | Ready |

---

## Strategies

### 1. Dispersion Momentum

**Folder**: [dispersion/](dispersion/)
**Plan**: [dispersion/backtest_plan.md](dispersion/backtest_plan.md)
**Source**: [dispersion/dispersion.pdf](dispersion/dispersion.pdf)

**Core Idea**: 4-week price momentum on index ETFs, filtered by constituent return dispersion. When a large index move is accompanied by a *small* change in dispersion (concentrated, non-broad-based move), the strategy exits — detecting potential trend exhaustion.

| Parameter | Value |
|-----------|-------|
| Signal | 4-week return > 0 → Long, else Cash |
| Filter | `Δindex_return > N × Δdispersion` → exit |
| Rebalance | Weekly (Friday close, Monday open execution) |
| Instruments | CSI 300, CSI 500, CSI 1000 |
| Data | Daily OHLCV + constituent returns (Bloomberg) |
| Costs | 0.2% round-trip |

**Expected Performance** (2008–2017, in-sample):
- CSI 300: ~15–20% annual return, -11% to -16% max drawdown
- Improvement over simple momentum: lower drawdown, higher Sharpe

---

### 2. Constituent Consistency

**Folder**: [consistency/](consistency/)
**Plan**: [consistency/backtest_plan.md](consistency/backtest_plan.md)
**Source**: [consistency/consistency.pdf](consistency/consistency.pdf)

**Core Idea**: Uses PCA on intraday constituent prices to measure how "in unison" stocks are moving. When consistency (first PC variance explained ratio R) is high, the market has a collective force likely to produce a trend — trade the direction of the first T minutes. When R is low, skip the day.

| Parameter | Value |
|-----------|-------|
| Signal | R > rolling 60-day mean → follow T-minute direction |
| Consistency (R) | λ₁ / Σλ (first PC variance explained) |
| Entry time T | 24 minutes after open (optimized) |
| Stop-loss | 0.6% |
| Instrument | IF (CSI 300 futures), expandable to IH, IC |
| Data | 1-min futures + 30-min constituent bars (Bloomberg) |

**Expected Performance** (out-of-sample 2014–2016):
- Annual return: 23.1%, Max drawdown: -11.7%, Win rate: 36.8%

**Related**: Combines well with Dispersion strategy — dispersion for weekly positioning, consistency for intraday entry timing.

---

## Workflow

```
PDF / Word idea
      │
      ▼
 _ideas/              ← rough notes, not yet a full plan
      │
      ▼
 strategy-name/
 ├── backtest_plan.md  ← structured implementation plan
 └── source.pdf        ← original research paper
      │
      ▼
 [implementation repo] ← coding begins here
```

**Status labels**: `Idea` → `Planning` → `Ready` → `Implemented`

---

## Adding a New Strategy

1. Create a folder: `strategy-name/`
2. Add source material: `strategy-name/source.pdf` (or `.docx`)
3. Write the plan: `strategy-name/backtest_plan.md`
4. Add a row to the [Strategy Index](#strategy-index) above

---

### 3. Index Money Flow

**Folder**: [index-money-flow/](index-money-flow/)
**Plan**: [index-money-flow/backtest_plan.md](index-money-flow/backtest_plan.md)
**Source**: [index-money-flow/changjiang_hf_factor_7.pdf](index-money-flow/changjiang_hf_factor_7.pdf)

**Core Idea**: Extends the A-share 博弈因子 (Game Factor / Active Trading Ratio) from stock selection
to a **cross-country index long/short strategy**. For each APAC country index, constituent stocks'
active buy ratios are aggregated (cap-weighted) to produce an index-level 博弈因子. Countries with
high aggregate active buying are longed; countries with dominant active selling are shorted.

| Parameter | Value |
|-----------|-------|
| Universe | HSI, CSI 300, KOSPI 200, NKY, TAIEX |
| Factor | Index-level active buy ratio = Σ(w_i × Stock_博弈因子_i) |
| Distribution | Confidence Normal (markets w/ price limits) / Standardized Normal (HK, Japan) |
| Signal | 4-week rolling mean → cross-sectional z-score |
| Position | Long top-2, short bottom-2 countries |
| Rebalance | Weekly (Friday signal, Monday execution) |
| Execution | Index futures (HI1, IFc1, KM1, NK1, TW1) |
| Data | 30-min constituent bars via Bloomberg |

**Novel vs. original paper**: Original uses factor for intra-China stock selection.
This plan aggregates to index level and uses the signal for cross-country allocation — a
pure country rotation strategy with no individual stock positions.

**Interaction with other strategies**:
- **Dispersion factor** (Strategy #1): can be combined — dispersion controls weekly positioning,
  game factor provides cross-country tilt
- **Consistency factor** (Strategy #2): consistency measures intraday collective movement;
  game factor measures buy/sell imbalance — complementary signals

---

*Last updated: 2026-03-15*
