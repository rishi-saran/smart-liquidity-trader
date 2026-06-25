# Smart Liquidity Trader V1

## Project Overview

Smart Liquidity Trader V1 is a Pine Script v5 trading system designed for Gold
(XAUUSD) and NAS100. It implements a highly selective liquidity-sweep reversal
strategy: the system waits for price to sweep a key liquidity level (prior day
high/low, Asia session high/low, or volume profile extremes), then enters a
reversal trade only when a Candle Impulse Structure Displacement (CISD) and a
Fair Value Gap (FVG) confirm institutional intent. The goal is precision over
frequency — a small number of high-conviction setups per week, not a high-volume
system.

## Folder Structure

```
smart-liquidity-trader/
├── CLAUDE.md                        # AI assistant context and hard rules
├── docs/
│   ├── 01-philosophy.md             # Strategy philosophy and edge rationale
│   ├── 02-technical-spec.md         # Full technical architecture
│   └── 03-decision-log.md           # Locked parameters and decisions, in order
├── pine/
│   └── smart_liquidity_trader_v1.pine   # Single Pine Script v5 file (entire system)
└── analysis/
    ├── requirements.txt             # Python dependencies
    ├── scripts/
    │   ├── load_trades.py           # Parse TradingView Strategy Tester CSV exports
    │   ├── stats_report.py          # Win rate, expectancy, profit factor, drawdown, R-dist
    │   └── monte_carlo.py           # Equity curve confidence bands via resampling
    └── data/                        # Drop exported CSVs here (gitignored)
```

## Stack

- **Pine Script v5** — the entire strategy lives in one file: `pine/smart_liquidity_trader_v1.pine`.
  TradingView has no deploy API; paste the file contents manually into the Pine Editor.
- **Python** (pandas / numpy / matplotlib) — post-backtest statistical analysis layer,
  reading CSV exports from TradingView's Strategy Tester.
- **No backend, no database, no frontend in V1.** That scope belongs to V2+.

## Status

**Phase 1 — Architecture complete, implementation starting.**

The scaffold, conventions, decision log, and skill definitions are in place.
Implementation follows the pipeline module order defined in CLAUDE.md: Session
Engine → Level Engine → Sweep Detector → CISD Engine → FVG Engine → Trade
Builder → Trade Manager → Visualization. Each module is committed only after
the user verifies it on a live TradingView chart.
